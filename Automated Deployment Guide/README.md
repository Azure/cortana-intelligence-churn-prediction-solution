# [Retail Customer Churn Prediction](https://gallery.cortanaintelligence.com/Solution/c2920246ecae45d28db7adc970d67c9b)

This folder contains the deployment instructions for the Retail Customer Churn Prediction solution in the Cortana Intelligence Gallery. To start a new solution deployment, visit the gallery page [here](https://gallery.cortanaintelligence.com/Solution/c2920246ecae45d28db7adc970d67c9b).

<Guide type="PostDeploymentGuidance" url="https://github.com/Azure/cortana-intelligence-churn-prediction-solution/blob/master/Automated%20Deployment%20Guide/Post%20Deployment%20Instructions.md"/>


## Summary
<Guide type="Summary">
Retail Customer Churn Prediction uses Cortana Intelligence Suite components to predict churn probability and helps find patterns in existing data associated with the predicted churn rate.
</Guide>


## Prerequisites
<Guide type="Prerequisites">
This solution requires the following prerequisites:

1.  An active [Azure subscription](https://azure.microsoft.com/en-us/) with login credentials
2.  A free [Azure Machine Learning Studio](https://azure.microsoft.com/en-us/services/machine-learning/) subscription
3.  A [Microsoft Power BI](https://powerbi.microsoft.com/en-us/) account
4.  A local installation of [Power BI Desktop](https://powerbi.microsoft.com/en-us/desktop/?gated=0&number=0)
5.  A local installation of [Visual Studio with SQL Server Data Tools (SSDT)](https://azure.microsoft.com/en-us/documentation/articles/sql-data-warehouse-install-visual-studio/)
</Guide>

#### Estimated Provisioning Time: <Guide type="EstimatedTime">30 Minutes</Guide>


## Description
<Guide type="Description">

Keeping existing customers is five times cheaper than the cost of acquiring new ones. For this reason, for marketing executives it is important to estimate the likelihood of customer churn and to take the necessary actions to minimize the churn rate.

Customer Churn Prediction uses Azure Machine Learning to predict churn probability and helps find patterns in existing data associated with the predicted churn rate. This information empowers businesses with actionable intelligence to improve customer retention and profit margins.

The objective of this guide is to demonstrate predictive data pipelines for retailers to predict customer churn. Retailers can utilize these predictions to prevent customer churn by applying their domain knowledge combined with proper marketing strategies to address customers at risk. 

The guide also shows how customer churn models can be retrained leveraging newer datasets as and when they become available.

## What's Under the Hood
The end-to-end solution is implemented in the cloud with Microsoft Azure and uses several Azure services to ingest, store and process the customer data. Advanced analytics and visualization provides retailers with actionable insights.  
The advanced analytics implemented in Azure Machine Learning Studio allows users to use Python or R language to build data science models (or reuse existing in-house or third-party libraries).  

## Solution Diagram
![Solution Diagram](https://user-images.githubusercontent.com/18489406/27402331-4c0e7520-5694-11e7-911b-a6ed2b51eabe.png)


## Technical details and workflow

1.  Historical sample data is loaded from **Azure Blob Storage** into **SQL Data Warehouse** using Polybase.

2.  Real-time event data is ingested through **Azure Event Hub** into **Azure Stream Analytics** and finally into **SQL Data Warehouse**.

3.  The predictions from **Azure Machine Learning** web service are performed in batches using the **Azure Data Factory**. Azure ML web service takes the data from **SQL Data Warehouse** as input and outputs the prediction results to **Azure Blob Storage**.

4. **Azure Data Factory** loads the prediction results from **Azure Blob Storage** back into **SQL Data Warehouse**.

5.  Finally, **Power BI** is used to visualize the results from **SQL Data Warehouse**.

</Guide>

#### Disclaimer

©2017 Microsoft Corporation. All rights reserved.  This information is provided "as-is" and may change without notice. Microsoft makes no warranties, express or implied, with respect to the information provided here.  Third party data was used to generate the solution.  You are responsible for respecting the rights of others, including procuring and complying with relevant licenses in order to create similar datasets.
