 # Cortana Intelligence Suite Retail Customer Churn Solution



## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup Steps](#setup-steps)
   - [General](#setup-steps)
   - [Azure Storage](#storage)
   - [Azure SQL Data Warehouse](#dw)
   - [Azure Machine Learning](#aml)
   - [Azure Event Hubs](#eventhub)
   - [Azure Stream Analytics](#asa)
   - [Azure Web App](#webapp)
   - [Azure Data Factory](#adf)
- [PowerBI Dashboard](#powerbi-dashboard)
   - [Local setup](#pbilocal)
   - [Publishing](#pbipublish)
- [Retrain the Predictive Model Manually](#retrainmanually)
- [Retrain the Predictive Model with ADF](#retrain)
   - [Azure Machine Learning](#amlupdates)
   - [Azure Data Factory](#adfmodifications)

## Introduction

The objective of this Guide is to demonstrate predictive data pipelines for retailers to predict customer churn.  Retailers can use these predictions to prevent customer churn by applying their use-case knowhow and proper marketing strategies to address at-risk customers. This tutorial also shows how customer churn models can be retrained to leverage additional data as it becomes available.

The end-to-end solution is implemented in the cloud, using Microsoft Azure. The solution is composed of several Azure components, including data ingest, data storage, data movement, advanced analytics and visualization. The advanced analytics are implemented in Azure Machine Learning Studio, where one can use Python or R language to build data science models (or reuse existing in-house or third-party libraries).  With data ingest, the solution can make predictions based on data that is being transferred to Azure from an on-premises environment.

This deployment guide will walk you through the steps of creating a customer churn prediction solution, including:

- Generation and ingestion of purchase transaction data using Azure Event Hub and Azure Streaming Analytics
- Creation of an Azure SQL Data Warehouse (DW) to store large volumes of transaction records in real time
- Use of PolyBase to transfer on-premises data to Azure SQL DW
- Use of Azure Machine Learning (AML) to deploy predictive models as web services and schedule periodic predictions via Azure Data Factory (ADF)
- Creation of a dashboard to display sales and customer churn data
- Retraining of the machine learning model with newer data

## Prerequisites

The steps described later in this guide require the following prerequisites:

1.  An active [Azure subscription](https://azure.microsoft.com/en-us/) with login credentials
2.  A free [Azure Machine Learning Studio](https://azure.microsoft.com/en-us/services/machine-learning/) subscription
3.  A [Microsoft Power BI](https://powerbi.microsoft.com/en-us/) account
4.  A local installation of [Power BI Desktop](https://powerbi.microsoft.com/en-us/desktop/?gated=0&number=0)
5.  A local installation of <a href="https://azure.microsoft.com/en-us/documentation/articles/sql-data-warehouse-install-visual-studio/">Visual Studio with SQL Server Data Tools (SSDT)</a>

## Architecture

Figure 1 illustrates the solution architecture using azure services.

![Figure 1: Architecture](media/architecture.png)
Figure 1: Customer Churn Solution Architecture for Retail

The historical data (in text format) will be loaded from Azure Blob Storage into Azure SQL DW though PolyBase.  The real-time event data will be ingested through Event Hub into Azure SQL DW; Azure Stream Analytics will then store the data in Azure SQL DW. Predictions from Azure Machine Learning models are performed in batches invoked by Azure Data Factory. AML will import data from Azure SQL DW and output the predictions to Azure Blob Storage. Through PolyBase, the prediction result can be loaded into Azure SQL DW quickly and efficiently. Power BI will source data from Azure SQL DW to generate reports. We will use Azure Data Factory to orchestrate:

1. Generating the AML predictions, 
1. Copying the predictions from Azure Blob Storage to Azure SQL DW,
1. Retraining model, and 
1. Updating web service with the retrained model

The machine learning model used here shows the general techniques of data science that can be used in customer churn prediction. You can use domain knowledge and combine the available datasets to build more advanced models to meet your business requirements.

Customer behavior changes slowly and doesn’t require real-time or near real-time prediction. Therefore, we use AML in batch mode.  We use Azure SQL DW for its scalability to query large databases, and its elasticity for easy scaling as needed.  We chose to use PolyBase to load data into Azure SQL DW because of its high efficiency.

## Setup Steps

The following are the steps to deploy the end-to-end solution for the predictive pipelines.

### Accessing Files in the Git Repository

This tutorial will refer to files available in the Technical Deployment Guide section of the [Cortana Intelligence Churn Prediction git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution). You can download all of these files at once by clicking the "Clone or download" button on the repository.

You can download or view individual files by navigating through the repository folders. If you choose this option, be sure to download the "raw" version of each file by clicking the filename to view it, then cliking Download.

### Choose a Unique String

You will need a unique string to identify your deployment because some Azure services, e.g. Azure Storage requires a unique name for each instance across the service. We suggest you use only letters and numbers in this string and the length should not be greater than 9.
 
We suggest you use "[UI]churn[N]"  where [UI] is the user's initials,  N is a random integer that you choose and characters must be entered in in lowercase. Please open your memo file and write down "unique:[unique]" with "[unique]" replaced with your actual unique string.

### Create an Azure Resource Group

1. Log into the [Azure Management Portal](https://ms.portal.azure.com).
1. Click **Resource groups** button on upper left, and then click **+** button to add a resource group.
1. Enter your **unique string** for the resource group and choose your subscription.
1. For **Resource Group Location**, you should choose one of the following as they are the locations that offer all of the Azure services used in this guide (with the exception of Azure Data Factory, which need not be located in the same location):
  - South Central US
  - West Europe
  - Southeast Asia

Please open your memo file and save the information in the form of the following table. Please replace the content in [] with its actual value.  

| **Azure Resource Group** |                     |
|------------------------|---------------------|
| resource group name    |[unique]|
| region              |[region]||

### Instruction for Finding Your Resource Group Overview

In this tutorial, all resources will be generated in the resource group you just created. You can easily access these resources from the resource group overview page, which can be accessed as follows:

1. Log into the [Azure Management Portal](https://ms.portal.azure.com).
1. Click the **Resource groups** button on the upper-left of the screen.
1. Choose the subscription your resource group resides in.
1. Search for (or directly select) your resource group in the list of resource groups.
 
Note that you may need to close the resource description page to add new resources.

In the following steps, if any entry or item is not mentioned in the instruction, please leave it as the default value.

<a name="storage"></a>
### Create an Azure Storage Account

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just created.
2. In ***Overview*** panel, click **+** to add a new resource. Enter **Storage Account** and hit "Enter" key to search.
3. Click on **Storage Account** offered by Microsoft (in the "Storage" category).
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "Name".
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Leaving the default values for all other fields, click the **Create** button at the bottom.
8. Go back to your resource group overview and wait until the storage account is deployed. To check the deployment status, refresh the page or the list of the resources in the resource group as needed.

#### Get the Primary Key for the Azure Storage Account
These are the steps to get the access key that will be used in the SQL script to load the data into Azure SQL DW:

1. Click the created storage account. In the new panel, click on **Access keys**.
1. In the new panel, click the "Click to copy" icon next to `key1`, and paste the key into your memo.

| **Azure Storage Account** |                     |
|------------------------|---------------------|
| Storage Account        |[unique string]|
| Access Key     |[key]             ||

#### Create Containers and Upload Data to Azure Storage Account
These are the steps for creating containers and uploading the data to Azure Blob Storage:

1. Click on **Containers** in the **BLOB SERVICE** group in the left-hand sidebar. In the new panel, click **+ Container** to add a container.
1. Enter **data** for "Name" and click **Create** at the bottom.
1. Click on the name of the container **data**, then click the "Upload" button on the top of the new panel.
1. Choose the [Users.csv](resource/Users.csv) file that you obtained from the `resource` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide). Click **Upload**.
1. Repeat the last two steps for each of the remaining files:
    - [Activities.csv](resource/Activities.csv)
    - [age.csv](resource/age.csv)
    - [region.csv](resource/region.csv)

<a name="dw"></a>
### Create Azure SQL Data Warehouse
1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you just deployed.
2. In the ***Overview*** panel, click **+** to add a new resource. Enter **SQL Data Warehouse** and hit "Enter" key to search.
3. Click on the **SQL Data Warehouse** option offered by Microsoft in the "Databases" category.
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** for "Database Name".
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Leave **Select source** as the default value, "Blank database".
7. Click **Server**. In the new panel, click **Create a new server**.
8. In the "New Server" panel:
    1. Enter **unique string** for "Server name".
    2. Use **azureadmin** (or your own preferred username) as the admin username. Record the username in the memo table given below.
    3. Use **pass@word1** (or your own preferred password) as the password. Record the password in the memo table given below.
    4. Click **Create**.
8. Adjust Performance to **300** DWU by dragging the sliding bar to the left. See  [Manage compute power in Azure SQL Data Warehouse (Overview)](https://docs.microsoft.com/en-us/azure/sql-data-warehouse/sql-data-warehouse-manage-compute-overview) for more information about DWU.
9. Click **Create** at the bottom.
9. Navigate to your resource group overview and wait until the resource is deployed. To check whether the resource is ready, refresh the page or the list of the resources in the resource group as needed until the resource name appears.
10. In the list of resources, click on the SQL Server that was just created.
11. Under ***Settings*** for the new server, click ***Firewall***.
12. Create a rule called ***open*** with the IP range of **0.0.0.0** to **255.255.255.255**. This will allow you to access the database from your desktop. Click ***Save*** at the top of the panel.

    **[Note]: This firewall rule is not recommended for production-level systems, but for this demo, it is acceptable. You may change this rule to only allow connections from IP addresses that you trust.**

| **Azure SQL Data Warehouse** |                     |
|------------------------|---------------------|
| Server Name            |[unique string].database.windows.net|
| Database               |[unique string]|
| User     |                     |
| Password               |                     ||


### Load Historical Data into Azure SQL Data Warehouse
1. Open the [createTableAndLoadData.dsql](resource/createTableAndLoadData.dsql) (available in the resource folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)) in Visual Studio 2015 with SQL Server Data Tools (SSDT).
2. Modify the query by inserting your Azure Storage account name and access key (recorded earlier in your memo table) for the IDENTITY and SECRET values on lines 48 and 49. On line 58, type your Azure Storage account name in place of "[unique]" in the LOCATION value.
2. Click the green button on the top-left corner of the file window to execute the query.
3. You will be prompted for the server name, database name, and login credentials: use the values stored in your SQL DW memo table. Make sure to use the full path for the server name by including the part ".database.windows.net" as well. Choose **SQL Server Authentication** for **Authentication**.
4. Wait until all the queries finish executing.

<a name="aml"></a>
### Set up Azure Machine Learning

#### The Azure ML Model
The model used in this Guide is based on the [Retail Customer Churn Prediction Template](https://gallery.cortanaintelligence.com/Collection/Retail-Customer-Churn-Prediction-Template-1). Specifically, the model (two-class boosted decision tree) and features are the same as the template. Example features include: days between most recent activity date and current date, total purchases by each customer, average number of days between 2 consecutive purchases, etc.  

The data used by the Template include 4 months' activities information. This Guide generates history data from the first 2 months data and streams the remaining 2 months data. 

#### Create Azure Machine Learning Workspace

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to the resource group you created.
2. In ***Overview*** panel, click **+** to add a new resource. Enter **Machine Learning Workspace** and hit "Enter" key to search.
3. Click on **Machine Learning Workspace** offered by Microsoft in the "Intelligence + analytics" category.
4. Click the **Create** button at the bottom of the description panel.
5. In the Machine Learning workspace panel:
    1. Enter your **unique string** for "Workspace name".
    2. Leave "Subscription", "Resource group", and "Location" as the default.
    3. Choose **Use existing** for "Storage account" and select the storage account you created earlier.  
    4. Choose **Standard** for "Workspace pricing tier".
    5. Choose **Create new** for "Web service plan".
    6. Click on **Web service plan tier**, choose **S1 Standard** and click **Select** at the bottom.
    7. Click **Create** at the bottom.

#### Deploy Azure Machine Learning Predictive Web Service
1. Go to the [Retail Churn Predictive Experiment](https://gallery.cortanaintelligence.com/Experiment/Retail-Churn-Predictive-Exp-1) web page in the Cortana Intelligence Gallery.
2. Click the **Open in Studio** button on the right. Log in if needed.
3. Choose the region and workspace. For region, you should choose the region that your resource group resides. You can get the information from table "Azure Resource Group".  For workspace, choose the workspace you just created.
4. Wait until the experiment is copied.
5. Input database information in the two **Import Data** modules at the top of the experiment. Select the module to change its parameters. You only need to change **Database server name**,**Database name**, **User name** and **Password**. Use the information you collected in the table "Azure SQL Data Warehouse". Leave the query as it is.
6. Click **Run** at the bottom of the page. It takes around three minutes to run the experiment.
7. Click **Deploy Web Service** at the bottom of the page, choose **classic** web service, and click "Yes" to publish the web service. (You may receive a warning message that the web service input has not been selected: you can ignore this message.) This will lead you to the web service page.  The web service home page can also be found by clicking the ***WEB SERVICES*** button on the left-hand menu bar in your workspace.
8. Copy the ***API key*** from the web service home page and save it in the memo table given below.
9. Click the link ***BATCH EXECUTION*** under the ***API HELP PAGE*** section. On the BATCH EXECUTION help page, copy the
***Request URI*** under the ***Request*** section and add it to the table below as you will need this information later
. Copy only the URI part (`https:… /jobs`), ignoring the URI parameters starting with "?". Here is an example of what it should look like: `https://ussouthcentral.services.azureml.net/workspaces/xxxx/services/xxxx/jobs`

|**Machine learning Web Service** |      |
| --------------------------- |--------------------------:|
| apiKey                     | [API key from API help page]|
| mlEndpoint              |        [Batch Request URI]                   ||

<a name="eventhub"></a>
### Create an Azure Event Hub
1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to your resource group.
2. In "Overview" panel, click **+ Add** to add a new resource. Type **Event Hubs** and hit "Enter" key to search.
3. Click on **Event Hubs** offered by Microsoft in the "Internet of Things" category.
4. Click **Create** at the bottom of the description panel.
5. In the new panel for creating a namespace, enter your **unique string** for "Name".
6. Leaving the default values for all other fields, click the **Create** button at the bottom of the panel.
7. Return to your resource group's overview page. When it has finished deploying, click on the resource of type "Event hubs".
9. Click the **+ Event Hub** button to add an event hub.
10. In the new panel:
    1. Enter **churn** for "Name".
    2. Enter **4** for "Partition Count".
    3. Enter **2** for "Message Retention".
    4. Click **Create** at the bottom.
11. Click on the ***Event Hubs*** option in the menu bar at left (under the "Entities" heading).
12. Click on the event hub named **churn** created through the previous steps. In the new panel:
    1. Click ***Shared access policies*** in the left-hand menu bar (under the ***SETTINGS*** heading).
    2. In the new panel,  click **+ Add** to add a new policy.
    3. In the new panel:
        1. Enter **sendreceive** for the "Policy name".
        2. Check **Send** and **Listen**.
        3. Click **Create** at the bottom of the panel.
        4. Wait until the new policy is created and shown in the listing of "Shared access policies".
        5. Click the policy **sendreceive**, and save the "PRIMARY KEY" to the memo table given below.

| **Azure Event Hub** |                        |
|---------------------|------------------------|
| EventHubServiceNamespace | [unique string]  |
| Event Hub           |  churn           |
| EventHubServicePolicy  |     sendreceive                 |
| EventHubServiceKey       |  [PrimaryKey]     ||

<a name="asa"></a>
### Create an Azure Stream Analytics Job
1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to your resource group.
2. In ***Overview*** panel, click **+ Add** to add a new resource. Enter **Stream Analytics job** and hit "Enter" key to search.
3. Click on **Stream Analytics job** offered by Microsoft in the "Internet of Things" category.
4. Click **Create** at the bottom of the description panel.
5. Enter your **unique string** in the "Job name" field.
6. Click **Create** at the bottom.
7. Return to your resource group and refresh the listing until your Stream Analytics job appears, indicating that deployment has finished. Click on the Stream Analytics job's name.
9. In the Stream Analytics job overview panel, click **Inputs**.
10. In the new panel, click **+ Add** to add an input. In the "New input" panel:
    1. Enter  **datagen** for the "Input alias".
    2. Choose **Data Stream** for the "Source type".
    3. Choose **Event hub** for the "Source".
    5. Choose your **unique string** for the "Service bus namespace".
    6. Choose **churn** for the "Event hub name".
    7. Choose **sendreceive** as the "Event hub policy name".
    8. Leaving other fields on their default values, click the **Create** button at the bottom.
10. Return to the Stream Analytics job overview panel and click **Outputs**.
10. In the new panel, click **+ Add** to add an output. In the "New output" panel:
    1. Enter **sqldw** for the "Output alias".
    2. Choose **SQL database** for the "Sink".
    4. Choose your **unique string** for the "Database".
    5. Enter your username and password (recorded in the Azure SQL Data Warehouse memo table) for the database.
    6. Enter **Activities** for the "Table".
    7. Click the **Create** button at the bottom.
11. Return to the Stream Analytics job overview panel and click **Query**.
11. In the new panel, click **Query** and click **+** in the new panel. Remove the default query content and enter:

    ```
    SELECT
    System.Timestamp systime,
    TransactionId,
    "Timestamp",
    UserId,
    ItemId,
    Quantity,
    Value,
    ProductCategory,
    Location
    INTO
        sqldw
    FROM datagen;
    ```
    
    Click the **Save** icon to save the query.
    **[Note]: The input and output aliases are used in the query, and the selected column names must exactly match those in the Activities table.**
12. Return to the overview of the Stream Analytics job and click the "Start" button.
13. In the new panel, choose "Now" for the "Job output start time".
14. Click the **Start** button at the bottom.  

<a name="webapp"></a>
### Set up Azure Web Job/Data Generator

The data generator emits one day's transaction data every 15 minutes to reduce the wait time for viewing results in this demo.

1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to your resource group.
2. In ***Overview*** panel, click **+ Add** to add a new resource. Enter **Web App** and hit "Enter" key to search.
3. In the result list, click on **Web App** offered by Microsoft in the "Web + Mobile" category.
4. Click the **Create** button at the bottom of the description panel.
5. In the new panel:
    1. Enter your **unique string** for the "App name".
    3. Click **App Service Plan/Location**. In the new **App Service Plan** panel:
        1. Click **Create New** and enter your unique string for the **App Service Plan**.
        2. Choose the region where your resource group resides.
        3. Click **OK** at the bottom of the panel.
6. Click the **Create** button at the bottom.
7. Return to the resource group overview. Refresh the resource listing until the app service deployment completes (usually takes around two minutes).
8. Click on the App Service resource whose type is "App Service" (not "App Service Plan") in your resource group to load its overview panel.
8. In the left-side panel, search for "Application Settings" and click on the **Application Settings** result. In the new panel:
    1. Choose **2.7** for the Python version.
    2. Choose **64-bit** for the Platform.
    3. Toggle **On** for the "Always on" setting.
    4. In the **App settings** section, add the following key-value pairs (using values recorded in your Azure Event Hub memo table) and leave the default entry as it is:

      | **Azure App Service Settings** |        | 
      |------------------------|---------------------|
      | EventHubServiceNamespace |[unique string]          |
      | EventHub              |churn         |
      | EventHubServicePolicy              |sendreceive         |
      |    EventHubServiceKey           |[unique string]            ||

  Note that the value for "EventHubServiceNamespace" is the **unique string** only, without the extension "servicebus.windows.net." Click **Save** and return to the App Service overview panel.

9. On the side panel, search for "WebJobs" and click on the **WebJobs** result.
10. Click on the **+ Add** button to add a WebJob. In the new panel:
    1. Enter **eventhub15min** for the "Name".
    2. Select the [eventhub_15min.zip](resource/eventhub_15min.zip) file (available in the `resource` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)) for "File Upload".
    3. Choose **Triggered** for Type.
    4. Choose the **Manual** setting from the "Triggers" drop-down menu.  (Note: the uploaded zip file contains a scheduled settings file, so we do not need to specify the schedule settings when submitting the WebJob.)
    5. Click the **OK** button at the bottom.
11. If necessary, refresh the listing until the new WebJob appears.
12. Select the job "eventhub15min" and click the **Run** button. Wait until the job finishes. To check the job's status, click the **Refresh** button until the status changes to "Completed".


### Check Data Ingest

You can check whether the data is being ingested into your SQL Data Warehouse by using Visual Studio 2015 with SSDT to run testing queries.

1. Open Visual Studio 2015 with SSDT.
2. In the top menu bar, click "View" and select "SQL Server Object Explorer" in the drop-down menu.
4. Click the "Add SQL Server" icon (green plus sign over a gray, rectangular server) to add the SQL server that you created earlier.
5. Fill in the server name, credentials, and database name recorded in your SQL Data Warehouse memo table. Choose "SQL Server Authentication" for the **Authentication** type.
6. After connecting to the server, click the arrow beside its entry in the left-hand menu to expand it.  In the "Databases" section, choose the database you create in this solution, which has the same name as the unique string.
6. Right-click on the database and choose "New query".
7. Run the following query:

    ```
    select top 1 * from Activities order by timestamp desc
    ```
    
8. Compare the value in the "SysTime" with the current UTC time.  The difference should be no more than 15 minutes. Two columns in the activities dataset ("ProductCategory" and "Location") has a single "Unknown" value and this is the way data are provided in the [Template](https://gallery.cortanaintelligence.com/Collection/Retail-Customer-Churn-Prediction-Template-1). 

<a name="adf"></a>
### Set up Azure Data Factory
1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to your resource group.
2. In ***Overview*** panel, click **+ Add** to add a new resource. Type **Data Factory** and hit "Enter" key to search.
3. Click on **Data Factory** offered by Microsoft in the "VM Extension" category.
4. Click the **Create** button at the bottom of the description panel.
5. In the "New data factory" panel:
    1. Enter your **unique string** for "Name".
    2. Click the **Create** button at the bottom.
6. Navigate to your resource group's overview page and refresh until the Data Factory's deployment concludes. Click on the Data Factory in the resource listing.
7. Click on ***Author and deploy*** in the new panel.
8. Create Linked services:
    1. Create the Azure Storage Linked service:
         1. Click **New Data Store** and choose **Azure Storage**.
         2. Replace the content in the editor with the content in [AzureStorageLinkedService.json](resource/AzureDataFactory/AzureStorageLinkedService.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Replace "[unique]" with your unique string and "[Key]" with your storage key.
         4. Click the up arrow button to deploy this linked service.
    2. Create the Azure SQL DW Linked service:
         1. Click **New Data Store** and choose **Azure SQL Data Warehouse**.
         2. Replace the content in the editor with the content in [AzureSqlDWLinkedService.json](resource/AzureDataFactory/AzureSqlDWLinkedService.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Replace "[unique]" with your unique string and "[User]" and "[password]" with the values you chose earlier (recorded in the SQL Data Warehouse memo table). Note that there are two instances of "[unique]".
         4. Click the up arrow button to deploy this linked service.
    3.  Create the Azure ML Linked Service:
         1. Click **...More** -> **New Compute** and choose **Azure ML**.
         2. Replace the content in the editor with the content in [AzureMLLinkedService.json](resource/AzureDataFactory/AzureMLLinkedService.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Replace  the content in "mlEndpoint" and "apikey" with the values recorded in the Azure ML memo table.
         4. Click the up arrow button to deploy this linked service.
9. Create the datasets:
    1. Create the Azure Blob Storage dataset:
         1. Click **New Dataset** and choose ***Azure Blob Storage***.
         2. Replace the content in the editor with the content in [AzureBlobDataset.json](resource/AzureDataFactory/AzureBlobDataset.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Click the up arrow button to deploy the dataset.
    1. Create the user dataset:
         1. Click **New Dataset** and choose ***Azure SQL Data Warehouse***.
         2. Replace the content in the editor with the content in [AzureSqlDWInputUser.json](resource/AzureDataFactory/AzureSqlDWInputUser.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Click the up arrow button to deploy the dataset.
    1. Create the activity dataset:
        1. Click **New Dataset** and choose ***Azure SQL Data Warehouse***.
        2. Replace the content in the editor with the content in [AzureSqlDWInputActivity.json](resource/AzureDataFactory/AzureSqlDWInputActivity.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        3. Click the up arrow button to deploy the dataset.
    1. Create the prediction dataset:
        1. Click **New Dataset** and choose ***Azure SQL Data Warehouse***.
        2. Replace the content in the editor with the content in [AzureSqlDWOutputPrediction.json](resource/AzureDataFactory/AzureSqlDWOutputPrediction.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        3. Click the up arrow button to deploy the solution.
10. Create the pipelines:
    1. Create the pipeline from SQL Data Warehouse to Azure ML:
        1. Right-click **Drafts** and choose ***New pipeline***.
        2. Copy the content in [MLPipeline.json](resource/AzureDataFactory/MLPipeline.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)) to the editor.
        3. Replace "[unique]" with your unique string and  "[User]" and "[password]" with the values chosen earlier (recorded in the Azure SQL Data Warehouse memo table).
        4. Specify an active period for the pipeline. You should use the current UTC time as the start time. Our data-generating web job will create data every 15 minutes for up to 15 hours, so it is not necessary to choose a duration longer than fifteen hours. As an example, if the current UTC time were midnight on December 1, 2016, one would enter:
	
            ```
            "start": "2016-12-01T00:00:00Z",
            "end": "2016-12-01T15:00:00Z",
            ```
	    
        5.  Set the value "isPaused" to "false":
	
            ```
            "isPaused": false
            ```
	    
        6.  Click the up arrow button to deploy the pipeline.

    2. Create the pipeline from Blob Storage to SQL Data Warehouse:
        1. Right-click **Drafts** and choose "New pipeline",
        2. Copy the content in [BlobToSqlDW.json](resource/AzureDataFactory/BlobToSqlDW.json) (available in the `resource/AzureDataFactory` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)) to the editor.
        3. Specify an active period that you want the pipeline to run. The active period should be the same as what you chose above.
        4. Set the value "isPaused" to "false".
        5. Click the up arrow button to deploy the pipeline.

<a name="pbilocal"></a>
## PowerBI Dashboard

Power BI is used to create visualizations for monitoring sales and predictions. It can also be used to help detect trends in important factors for predicting churn. The instructions that follow describe how you can use the provided Power BI desktop file (Customer-Churn-Report.pbix) to visualize your data. 

1. If you have not already done so, download and install the [Power BI Desktop application](https://powerbi.microsoft.com/en-us/desktop).
1. Download the Power BI template file `Customer-Churn-Report.pbix` (available in the `Power BI` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)) by left-clicking on the file and clicking on "Download" on the page that follows.
1. Double click the downloaded ".pbix" file to open it in Power BI Desktop.
1. The template file connects to a database used in development. You'll need to change some parameters so that it links to your own database. To do this, follow these steps:
    1. Click on "Edit Queries" as shown in the following figure.
       
        [![Figure 1][pic 1]][pic 1] 

    1. Select a Query from the Queries panel (e.g., Age) and click on "Advanced Editors" as shown in the following figure.

        [![Figure 2][pic 2]][pic 2] 

    1. In the pop-up window for Advanced Editor, replace all "dbchurn" values with your "unique string" (database and server name). This process is shown in the following two figures, which assumes that the unique string is "mydb". (You should use the name of your own database.) Click "Done" after making the changes.

        [![Figure 3][pic 3]][pic 3] 
        [![Figure 4][pic 4]][pic 4] 

    1. With the same Query (e.g., Age) selected, click on "Edit Credentials" and enter your your credentials for accessing your database (recorded in the SQL Data Warehouse memo table). Click on "Connect" as shown in the following figure.

        [![Figure 5][pic 5]][pic 5] 

    1. The data for your table should be displayed if the connection information was correct, as in the following figure.

        [![Figure 6][pic 6]][pic 6]

    1. Update the other Queries by replacing "dbchurn" with the name of your database. 
    1. Click on the "Close & Apply" ribbon after all Queries have been updated.
    
You should now see multiple tabs in Power BI Desktop's report page. The "MyDashboard" tab combines the content from the "Activities" and "Predictions" tabs. The "Features" tab displays the important variables for predicting churn: days between transactions, region, and number of transactions.

<a name="pbipublish"></a>
Now we can publish the report into Power BI online to easily share with others: 
 
1. Click on "Publish" as shown below. Sign in with your Power BI credentials and choose a destination (e.g., My Workspace). 
[![Figure 7][pic 7]][pic 7]

1. After the report is successfully published, you should see a window like the following. Click on "Got it."
[![Figure 8][pic 8]][pic 8]

1. Sign into [Power BI](www.powerbi.microsoft.com) and click on the "Customer-Churn-Report" report (under Reports) to open it. 
1. We'll share the Churn Rate Overview tab from the report to create a dashboard. To do this, click on the "MyDashboard" tab and select "Pin Live Page" as shown in the following figure. 
[![Figure 9][pic 9]][pic 9]

1. Pin the page to a new dashboard named "Customer Churn Dashboard" as shown in the following figure.
[![Figure 10][pic 10]][pic 10]

1. Pin the remaining 3 tabs using the same approach: Churn Rate Drill Down, Sales Overview, and Machine Learning. 
1. Locate the newly created "Customer Churn Dashboard" under Dashboards group. You can share it with others by clicking on the Share button, as shown in the following figure.
[![Figure 11][pic 11]][pic 11]

Below is an overview of the different reports:

1. The Churn Rate Overview report shows the number of customers in 3 churn risk groups: low risk (no more than 30% churn rate), moderate risk (30% 50% churn rate) and high risk (more than 30% churn rate) on each prediction date. 

1. The Churn Rate Drill Down report shows that two regions Northeast, and Mountain-Prairie have a high percentage of high risk users. Marketing managers in these regions can thus be notified to take churn prevention actions. 

1. The Sales Overview report provides sales information by different dimensions: date, region, and age.

1. The Machine Learning report demonstrates how Azure Machine Learning can help users identify important variables. 

In this dashboard, we used cutoff points 0.3 and 0.5 to generate 3 churn risk groups: low risk, moderate risk, and high risk. These cutoff points can be modified when necessary. The following screen shot shows how this can be done. 

1. Click on the bar chart.
 
1. Click on the Churn Risk variable in the Prediction table.

1. Change the cutoff points and labels.

1. Click on the check mark to confirm the changes.

[![Figure a][pic a]][pic a]

At this point, you have a working solution that runs the customer churn prediction. Customer behavior patterns may change over time; prediction accuracy can then be improved by retraining the model. Two approaches for retraining the web services are described below: updating manually, and updating through a pipeline every 1 hour.

<a name="retrainmanually"></a>
## Retrain the Predictive Model Manually
1. Go to https://gallery.cortanaintelligence.com/Experiment/Retail-Churn-Train-1
2. Click ***Open in Studio*** on the right. Log in if needed.
3. Choose the region and workspace where the experiment should be copied. Choose the region where your resource group resides and the Azure ML workspace you created earlier. Wait until the experiment is copied.
5. Add your database information in the two **Import Data** modules. You only need to change "Database server name", "Database name", "User name", and "Password". Use the information you collected in the "Azure SQL Data Warehouse" memo table. Leave the query as it is.
6. Click **Run** at the bottom of the page. It takes around three minutes to run the experiment.
7. Click **SET UP WEB SERVICE** then **Predictive Web Service [Recommended]** at the bottom of the page to publish the web service. A **Predictive experiment** will be generated automatically. 
8. In the **Predictive experiment** select the trained model (as in the following figure) and press CTRL+C to copy it. 
[![Figure 12][pic 12]][pic 12]
9. On the left sidebar, navigate to the **EXPERIMENTS** page
10. Click on "Retain Churn [Predictive Exp.]" to open the experiment used previously.
11. Press CTRL+P to paste the newly trained model to this experiment. 
12. Delete the existing trained model and link the newly trained model to the "Score Model" module as shown in the following figure. 
[![Figure 13][pic 13]][pic 13]
13. Click **Run** at the bottom of the page. Wait until it finishes in about 3 minutes.
14. Click **DEPLOY WEB SERVICE** at the bottom of the page, choose **classic** web service, click **Yes** for the warning message saying that web service input or output is not selected and **Yes** for overwriting service. Now the web service has been updated. The API key for the web service remains the same.

[pic 1]: https://cloud.githubusercontent.com/assets/9322661/21234603/5c05440c-c2c1-11e6-9f2d-c93f2add350b.PNG
[pic 2]: https://cloud.githubusercontent.com/assets/9322661/21234604/5c09ee62-c2c1-11e6-8f8a-2b17090ffe9a.PNG
[pic 3]: https://cloud.githubusercontent.com/assets/9322661/21234609/5c0d783e-c2c1-11e6-8812-f69a85a909e5.PNG
[pic 4]: https://cloud.githubusercontent.com/assets/9322661/21234605/5c0a1a0e-c2c1-11e6-809f-4b37ad48895c.PNG
[pic 5]: https://cloud.githubusercontent.com/assets/9322661/21234606/5c0ad1d8-c2c1-11e6-90f7-ad1b7427971c.PNG
[pic 6]: https://cloud.githubusercontent.com/assets/9322661/21234608/5c0b73d6-c2c1-11e6-90bf-15306fd84c60.PNG
[pic 7]: https://cloud.githubusercontent.com/assets/9322661/21239433/e4142f76-c2d4-11e6-9b95-7a94136c5d01.PNG
[pic 8]: https://cloud.githubusercontent.com/assets/9322661/21234610/5c108682-c2c1-11e6-9c04-6bc9e72ca182.PNG
[pic 9]: https://cloud.githubusercontent.com/assets/9322661/22650292/8d59b22c-ec4c-11e6-9f22-c3e3182cd41a.PNG
[pic 10]: https://cloud.githubusercontent.com/assets/9322661/22650290/8d582f9c-ec4c-11e6-95fd-1943da0a2de9.PNG
[pic 11]: https://cloud.githubusercontent.com/assets/9322661/22650291/8d58e91e-ec4c-11e6-9c8f-494929ddf0be.PNG
[pic 12]: https://cloud.githubusercontent.com/assets/9322661/21438148/a84b86f2-c855-11e6-9168-f9f674715f48.PNG
[pic 13]: https://cloud.githubusercontent.com/assets/9322661/21438147/a83f0166-c855-11e6-9aa1-3882cfeb25de.PNG
[pic a]: https://cloud.githubusercontent.com/assets/9322661/22650293/8d59f804-ec4c-11e6-9caf-35a212e0556c.PNG

<a name="retrain"></a>
## Retrain the Predictive Model with ADF

<a name="amlupdates"></a>
<a name="trainingwebservice"></a>
### Deploy the Training Web Service
1. Go to https://gallery.cortanaintelligence.com/Experiment/Retail-Churn-Train-1
2. Click ***Open in Studio*** on the right. Log in if needed.
3. Choose the region and workspace where the experiment should be copied. Choose the region where your resource group resides and the Azure ML workspace you created earlier. Wait until the experiment is copied.
5. Add your database information in the two **Import Data** modules. You only need to change "Database server name", "Database name", "User name", and "Password". Use the information you collected in the "Azure SQL Data Warehouse" memo table. Leave the query as it is.
6. Click **Run** at the bottom of the page. It takes around three minutes to run the experiment.
7. Click **Deploy Web Service** at the bottom of the page, choose a classic web service, and click "Yes" to publish the web service. This will lead you to the web service page. The web service page can also be found by clicking the WEB SERVICES button on the left menu once logged into your workspace.
8. Copy the API key from the web service home page and save it to the memo table given below.
9. Click the ***BATCH EXECUTION*** link under the ***API HELP PAGE section***. On the BATCH EXECUTION help page, copy the Request URI under the Request section and add it to the table below as you will need this information later. Copy only the URI part (`https:… /jobs`), ignoring the URI parameters starting with a question mark (`?`).

  | **Train Machine Learning Web Service** |                           |
  | --------------------------- |--------------------------|
  | apiKey                     | [API key from API help page]|
  | mlEndpint              |        [Batch Request URI]                   ||


<a name="updatablewebservice"></a>
### Modify the Predictive Web Service to allow updates
The default web service endpoint we deployed in the section of "Deploy Azure Machine Learning Predictive Web Service" is associated with the experiment itself. In order to have a updatable endpoint, we need to create an additional service endpoint.

1. Go to https://studio.azureml.net, and choose workspace with the name of your unique string. You can change the workspace by clicking drop-down list on the top right of the web page.
2. On the left sidebar, navigate to the **Web Services** page by clicking the globe icon.
3. Click on "Retail Churn [Predictive Exp.]" (the first web service that you deployed in this tutorial).
3. Click ***Manage endpoints*** at the bottom of the page in the "Additional endpoints" section.
4. On the newly loaded page, click **+New**.
    1. Enter **update** for "Name".
    2. Leave the default values for everything else.
    3. Click **Save**.
5. After the endpoint is created, click the "update" service endpoint.
    1. Click ***Use endpoint*** under the **BASICS** picture. Record the "**Primary key**", "**Batch requests**" (only need to copy up to "jobs") and "**Patch**" in the memo table given below.
    2. Click on the ***API help*** link under the ***Patch*** URI. Record the **Resource Name** in the ***Updatable Resources*** section to the memo table below. (The resource name starts with "Retail Churn Template".) This name will be used in an Azure Data Factory pipeline to update the model.

    | **Updatable Predictive Machine Learning Web Service** |                           |
    | --------------------------- |--------------------------:|
    | apiKey                     | [Primary Key]|
    | mlEndpint              |        [Batch Request URI]                   |
    | updateResourceEndpoint |   [Patch URI]|
    | trainedModelName | [Resource Name]|

<a name="adfmodifications"></a>
### Modify the Azure Data Factory For Retraining and Updating
1. Go to the [Azure Portal](https://ms.portal.azure.com) and navigate to your resource group.
2. Click on the data factory you created earlier in this solution.
3. Click ***Author and deploy***.
3. Stop MLPipeline:
    1.  Click **MLPipeline** in the "Pipelines" list.
    2.  Stop the pipeline by setting the value "isPaused" to "true":
        ```
        "isPaused": true
        ```
    3.  Click the up arrow button to deploy the modified (paused) pipeline.
4. Create/Update Linked services:
    1. Create the Azure ML Training Linked Service:
         1. Click **New Compute** and choose ***Azure ML***.
         2. Replace the default content in the editor with the content in [AzureMLLinkedServiceTraining.json](resource/AzureDataFactoryRetrain/AzureMLLinkedServiceTraining.json)  (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Replace the content in "mlEndpoint" and "apikey" with the values from the "Train Machine learning Web Service" memo table.
         4. Click the up arrow button to deploy the linked service.
    2. Make the Azure ML Linked Service updatable:
         1. Click on the existing **AzureMLLinkedService**.
         2. Remove the contents of the editor and paste in the contents of [AzureMLLinkedService.json](resource/AzureDataFactoryRetrain/AzureMLLinkedService.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
         3. Fill in the "mlEndpoint" (Batch Requests), "apikey" and "updateResourceEndpoint" (Patch) values from the "Updatable Predictive Machine learning Web Service" memo table.
         4. Click the up arrow button to deploy the modified linked service.
5. Create datasets:
    1.  Create the Trained Model Blob dataset:
        1. Click **New Dataset** and choose ***Azure Blob Storage***.
        2. Replace the default content in the editor with the content in [TrainedModelBlob.json](resource/AzureDataFactoryRetrain/TrainedModelBlob.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        3. Click the up arrow button to deploy the dataset.
    1.  Create the User Retrain dataset:
        1. Click **New Dataset** and choose ***Azure SQL Data Warehouse***.
        1. Replace the default content in the editor with the content in [AzureSqlDWInputUserRetrain.json](resource/AzureDataFactoryRetrain/AzureSqlDWInputUserRetrain.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        1. Click the up arrow button to deploy the dataset.
    1. Create the Activity Retrain dataset:
        1. Click **New Dataset** and choose ***Azure SQL Data Warehouse***.
        1. Replace the default content in the editor with the content in [AzureSqlDWInputActivityRetain.json](resource/AzureDataFactoryRetrain/AzureSqlDWInputActivityRetrain.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        1. Click the up arrow button to deploy the dataset.
    1. Create the Placeholder Retrain dataset:
        1. Click **New Dataset** and choose ***Azure Blob Storage***.
        1. Replace the default content in the editor with the content in [PlaceHolderRetrain.json](resource/AzureDataFactoryRetrain/PlaceHolderRetrain.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        1. Click the up arrow button to deploy the dataset.
6. Create pipelines
    1. Create the retraining pipeline:
        1. Right-click **Drafts** and choose "New pipeline".
        1. Replace the default content in the editor with the content in [RetrainPipeline.json](resource/AzureDataFactoryRetrain/RetrainPipeline.json) (available in the `resource/AzureDataFactoryRetrain` folder of the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        2. Replace both instances of "[unique]" with your unique string. Insert the SQL DW username and password in place of "[User]" and "[password]", respectively.
        3. Specify an active period for the pipeline.  You should use the current UTC time as the start time. Our data-generating web job will create data every 15 minutes for up to 15 hours, so it is not necessary to choose a duration longer than fifteen hours. As an example, if the current UTC time were midnight on December 1, 2016, one would enter:
        ```
        "start": "2016-12-01T00:00:00Z",
        "end": "2016-12-01T15:00:00Z",
        ```
	
        4.  Set the value of "isPaused" to "false":
            ```
            "isPaused": false
            ```
	    
        4.  Click the up arrow button to deploy the pipeline.

    2. Create the update pipeline:
        1. Right=click **Drafts** and choose "New pipeline".
        1. Replace the default content in the editor with the content in [UpdatePipeline.json](resource/AzureDataFactoryRetrain/UpdatePipeline.json) (available in the `resource/AzureDataFactoryRetrain` folder of the the [git repository](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide)).
        2. Replace the value for "trainedModelName" with the value from the memo table. 
        3. Specify an active period that you want the pipeline to run. It should be the same as for the retrain pipeline just created.
        3. Set the value of "isPaused" to "false".
        4. Click the up arrow button to deploy the pipeline.
7. Restart MLPipeline
    1.  Click **MLPipeline** in the "Pipelines" list.
    2.  Set the value of "isPaused" to "false".
    3.  Click the up arrow button to deploy the modified pipeline.

To get more information about retraining, please go to [Updating models using Update Resource Activity](https://docs.microsoft.com/en-us/azure/data-factory/data-factory-azure-ml-batch-execution-activity#updating-models-using-update-resource-activity).
