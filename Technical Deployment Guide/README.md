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



 You will need a unique string to identify your deployment. We suggest you use only letters and numbers in this string and the length should not be greater than 9. Please open your memo file and write down "unique:[unique]" with "[unique]" replaced with your actual unique string.

##Create an Azure Resource Group for the solution
1. Log into the Azure Management Portal
1. Click "Resource groups" button on upper left, and then click +Add.
1. Enter in a name for the resource group and choose your subscription.
1. For Resource Group Location you should choose one of the following as they are the locations that support machine learning workspaces:
- South Central US
- West Europe
- Southeast Asia
