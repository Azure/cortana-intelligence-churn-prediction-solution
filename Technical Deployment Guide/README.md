#Cortana Intelligence Suite Retail Customer Churn Solution

## Introduction

The objective of this tutorial is to demonstrate predictive data pipelines for retailer to predict customer churn.  By combining domain knowledge and predictive analytics, retailers can prevent customer churn by using the predictive result and proper marketing strategies.   

The end-to-end solution is implemented in the cloud, using Microsoft Azure. The solution is composed of several Azure components, including data ingest, data storage, data movement, advanced analytics and visualization. The advanced analytics is featured with Azure Machine Learning where you can use Python or R language to  build data science model and also reuse existing in-house or third-party libraries.  With data ingest, the solution can make prediction based on the data that moves in to azure from on-premises environment.

This deployment guide will guide you through the process of creating such a customer churn prediction solution. The tutorial will include:

- The generation and ingestion of purchase transaction data using an Azure Event Hub and Azure Streaming Analytics.
- The creation of an Azure SQL Data Warehouse (DW) to store large amount transaction record in real time
- Use of PolyBase to load on-premises data to Azure SQL DW
- Using Azure Machine Learning to deploy prediction model as web services and run periodic prediction in Azure Data Factory (ADF)
- Dashboard to display sales and customer churn data

#Prerequisites

The steps described later in this guide  requires the
following prerequisites:

1)  Azure subscription with login credentials
    (https://azure.microsoft.com/en-us/)

2)  Azure Machine learning Studio subscription
    (https://azure.microsoft.com/en-us/services/machine-learning/)

3)  A Microsoft Power BI account
    (https://powerbi.microsoft.com/en-us/)

4)  Power BI Desktop installation
    (https://powerbi.microsoft.com/en-us/desktop/?gated=0&number=0)

5) Microsoft Azure Storage Explorer
    (http://storageexplorer.com/)

6)  A local installation of <a href="https://azure.microsoft.com/en-us/documentation/articles/sql-data-warehouse-install-visual-studio/">Visual Studio with SQL Server Data Tools (SSDT)</a>


#Architecture
============

Figure 1 illustrates the Azure architecture developed in this sample.

![](media/architecture.png)
Figure 1: Architecture

The above figure is the implemented architecture. The historical data as text format will be loaded from Azure Blob Storage into Azure SQL Data Warehouse (DW) though Polybase.  The real-time event data will be ingested through Event Hub into Azure.  Azure Stream Analytics will store the data in Azure SQL DW. The prediction through Azure Machine Learning (AML) runs in batch mode and is invoked by Azure Data Factory. AML will import data from Azure SQL DW and output the prediction to Azure Blob Storage. Through PolyBase, the prediction result can be loaded into Azure SQL DW efficiently and fast. Azure SQL DW serves the queries to populate the PowerBI dashboard. We use Azure Data Factory to orchestrate 1) AML prediction 2) Copy the prediction from Azure Blob Storage to  Azure SQL Data Warehouse. The machine learning model here is used as an example experiment and it shows the general techniques of data science that can be used in customer churn prediction. You can use domain knowledge and combine the available datasets to build more advanced model to meet your business requirements.

 For the nature of the problem, customer behavior changes slowly and doesn’t require real-time or near real-time prediction in minute scale. Therefore, we use Azure Machine Learning in batch mode.  We use Azure SQL DW for its scalability to query large amount of data, and its elasticity of starting small and scaling up as needed easily.  We chose to use PolyBase to load data into Azure SQL DW because of its high efficiency.


 #Deploy
 =====================

 Below are the steps to deploy the use case into your Azure subscription. Note that to condense the steps somewhat, **>** is used between repeated actions. For example:

 1. Click: **Button A**
 1. Click: **Button B**

 is written as

 1. Click: **Button A** > **Button B**  


## Unique String
 You will need a unique string to identify your deployment because some Azure services, e.g. Azure Storage requires a unique name across a region. We suggest you use only letters and numbers in this string and the length should not be greater than 9.
 We suggest you use "[UI]churn[N]"  where [UI] is the user's initials,  N is a random integer that you choose and characters must be entered in in lowercase. Please open your memo file and write down "unique:[unique]" with "[unique]" replaced with your actual unique string.

##Create an Azure Resource Group for the solution
1. Log into the Azure Management Portal https://ms.portal.azure.com
1. Click  **Resource groups** button on upper left, and then click **+** to add a resource group.
1. Enter your **unique string** for the resource group and choose your subscription.
1. For Resource Group Location you should choose one of the following as they are the locations that support all the Azure services used in this guide:
  - South Central US
  - West Europe
  - Southeast Asia

Please open your memo file and write down "resource group:[unique]" with "[unique]" replaced with your actual unique string.

In the following steps, if any entry or item is not mentioned in the instruction, please leave it as the default value.

## Create Azure Storage Account
1. Go to Azure Portal https://ms.portal.azure.com and choose the resource group you just deployed
2. In "Overview" panel, click **+** and enter **storage account** and hit "Enter" key to search
3. Click ** Storage account** offered by Microsoft in "Storage" category
4. Click ** Create** at the bottom of the description panel
5. Enter your **unique string** for "Name". Please write down in your memo: storage account: [Unique string]
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Leave everything else to use the default value
8. Click **Create** at the bottom. The portal may lead you back to the storage account description panel. DO NOT click "Create".
9. Go back to your resource group overview and wait until the storage account is created. To check if the resource is create or not, refresh the page or the list of the resources in the resource group as needed.

These are the steps to get the access key that will be used in the SQL script to load the data into Azure SQL DW:
10. Click the created storage account and in the new panel click **Access keys**
11. In the new panel, click the icon of "Click to copy" and paste the key in your memo

| **Azure Storage Account** |                     |
|------------------------|---------------------|
| Storage Account        |[unique string]|
| Connection String      |             |
| Primary access key     |             ||

These are the steps for creating containers and uploading the data to Azure blob storage:
1. Click **Containers** and on the new panel click **+** to add a containers
1. Enter **data** for "Name" and click **Create** at the bottom
1. Click the **data** container -> click "Upload" button on the top of the new panel
1. Choose Users.csv file that you can download from "resource" folder
1. Click **Upload**
1. Repeat the last two steps for Actvities.csv, age.csv, region.csv



## Create Azure SQL Data Warehouse
1. Go to Azure Portal https://ms.portal.azure.com and choose the resource group you just deployed
2. In "Overview" panel, click **+** and enter **SQL Data Warehouse** and hit "Enter" key to search
3. Click **SQL Data Warehouse** offered by Microsoft in "Databases" category
4. Click ** Create** at the bottom of the description panel
5. Enter your **unique string** for "Database Name"
6. Make sure the selected resource group is the one you just created. If not, choose the resource group you created for this solution.
7. Leave **Select source** as the default value "Blank database"
7. Click **Server** > **Create a new server**
8. In the new panel for "New Server"
    1. Enter **unique string** for "Server name"
    2. Use **azureadmin** or any of your preferred admin name. Please write it down in your memo: [server admin login]:[the value you entered]
    3. Use **pass@word1**  or any of your preferred password. Please write it down in your memo: [Password]:[the value you entered]
    4. Click **Create**
8. Adjust Performance to **100** by dragging the sliding bar to the left
9. Click **Create** at the bottom. The portal may lead you back to the SQL Data Warehouse description panel. DO NOT click "Create".
9. Go back to your resource group overview and wait until the resouce is create  To check if the resource is create or not, refresh the page or the list of the resources in the resource group as needed.
10. In the list of resources, click on the SQL Server that was just created.
11. Under ***Settings*** for the new server, click ***Firewall*** and create a rule called ***open*** with the IP range of 0.0.0.0 to 255.255.255.255. This will allow you to access the database from your desktop. Click ***Save*** at the top of the panel. 

    **Note**: This firewall rule is not recommended for production level systems but for this demo is acceptable. You will want to set this rule to the IP range of your secure system.

    | **Azure SQL Data Warehouse** |                     |
|------------------------|---------------------|
| Server Name            |[unique string]|
| Database               |[unique string]|
| server admin login     |                     |
| Password               |                     ||



## Load Historical Data into Azure SQL Data Warehouse


## Create an Azure Event Hub
