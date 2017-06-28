# [Retail Customer Churn Prediction](https://gallery.cortanaintelligence.com/Solution/c2920246ecae45d28db7adc970d67c9b)

This document is focusing on the post deployment instructions for the [Automated Deployment](https://gallery.cortanaintelligence.com/Solution/c2920246ecae45d28db7adc970d67c9b) through the Cortana Intelligence Gallery. The source code of the solution as well as manual deployment instructions can be found [here](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide).

## Architecture
The architecture diagram shows various Azure services that are deployed by [Retail Customer Churn Prediction Solution](https://gallery.cortanaintelligence.com/Solution/c2920246ecae45d28db7adc970d67c9b) using [Cortana Intelligence Solutions](https://gallery.cortanaintelligence.com/solutions), and how they are connected to each other in the end to end solution.

![Solution Diagram](https://user-images.githubusercontent.com/18489406/27402331-4c0e7520-5694-11e7-911b-a6ed2b51eabe.png)

## Technical details and workflow

1.  Historical sample data is loaded from **Azure Blob Storage** into **SQL Data Warehouse** using Polybase.

2.  Real-time event data will be ingested through **Azure Event Hub** into **Azure Stream Analytics** and finally into **SQL Data Warehouse**.

3.  The predictions from **Azure Machine Learning** web service are performed in batches using the **Azure Data Factory**. Azure ML web service takes the data from **SQL Data Warehouse** as input and outputs the prediction results to **Azure Blob Storage**.

4. **Azure Data Factory** loads the prediction results from **Azure Blob Storage** back into **SQL Data Warehouse**.

5.  Finally, **Power BI** is used for results visualization from **SQL Data Warehouse**.

All the resources listed above besides Power BI are already deployed in your subscription. The following instructions will guide you on how to monitor things that you have deployed and create visualizations in Power BI.

## Post Deployment Instructions
### Monitor progress
Once the solution is deployed to the subscription, you can see the services deployed by clicking the resource group name on the final deployment screen in the Cortana Intelligence Solutions page. This will show all the resources under this resource groups on [Azure management portal](https://portal.azure.com/). The entire solution should start automatically on cloud. You can monitor the progress of the following resources.

#### Azure Storage
Azure Storage account is used by the Azure Functions to maintain some required information and is used by the solution to store the historical user data and the prediction of the machine learning model.

#### Azure Event Hub
Azure Event Hub is used to ingest the data from data generator.

#### Azure Stream Analytics
Azure Stream Analytics is used to process the data from event hub and redirect the data to SQL data warehouse.

#### Azure Functions
Azure Functions provide the primary mechanism for deploying the solution and simulating the user activity. The functions interact with Azure Storage, Azure Machine Learning, Azure Event Hub, Azure Stream Analytics, Azure Data Factory, and SQL DW to perform all these functions. It also hosts and runs the data generator application. Below is a summary of the functions.

Functions used to assist in deployment:
* GalleryToMlWebSvc: This is used to copy the Scoring Experiment from the Gallery to the workspace and create the webservice endpoints.

* CiqsHelpers: These are helper functions for the automated deployment.

* executeSqlQuery: Function to execute SQL Query that uploads the data to SQL DW from a Blob Storage account using Polybase. The SQL Query takes the Storage account Name, Account key and Connection String as parameter.

* PopulateStorageContainer: Populates the storage account with raw csv files.

* StartADFPipeline: Sets the Start and End time of Azure Data Factory to current time.

* startStreamingJob: Starts Azure Stream Analytics Pipeline.

* UpdateDatabaseInfoInExperiment: The Machine learning experiment that’s copied from gallery reads from a SQL Database, this functions updates the database information in the experiment to the provided values.

Web job used for data simulation:
* eventhub_15min: The data generator that emits one day's transaction data every 15 minutes to reduce the wait time for viewing results in this solution.

#### Azure Machine Learning
Azure Machine Learning is used to help retail companies predict customer churns. This solution focuses on binary churn prediction, i.e. classifying the users as churners or non-churners, and employed a two-class boosted decision tree model. Example features used in the model include: days between most recent activity date and current date, total purchases by each customer, average number of days between 2 consecutive purchases, etc. The data used by the solution include 4 months' activities information. It generates historical data from the first 2 months data and streams the remaining 2 months data.

#### SQL Data Warehouse
SQL Data Warehouse is used to store the historical data and forecast the future predictions.

#### Azure Data Factory
Azure Data Factory is used to apply the machine learning model to the user and activity data and predict churn results. It is also used to load the prediction results from Azure Blob Storage into SQL Data Warehouse.

### Visualization
Power BI is used to create visualizations for monitoring sales and predictions. It can also be used to help detect trends in important factors for predicting churn. The instructions that follow describe how you can use Power BI to visualize your data.

The detailed instructions can be found [here](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide#powerbi-dashboard). **Note:** the _unique string_ mentioned in the instructions is your resource group name.

At this point, you will have a working solution that runs the customer churn prediction. **Note:** In order to visualize the results, you might need to slide the bar in each dashboard to change the range of the date for predictions accordingly.

### Retrain the Predictive Model (Optional)
Customer behavior patterns may change over time; prediction accuracy can then be improved by retraining the model. Two approaches for retraining the web services are described below: updating manually, and updating through a pipeline every 1 hour.

* Retrain the Predictive Model Manually

  The detailed instructions can be found [here](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide#retrainmanually). You could follow the step-by-step instructions to retrain the predictive model manually.

* Retrain the Predictive Model with ADF

  The detailed instructions can be found [here](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide#retrain). You could follow the step-by-step instructions to retrain the predictive model with ADF.

## Customization
You can reuse the source code in the [Manual Deployment Guide](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide) to customize the solution for your data and business needs.

## Stopping the Solution
To entirely remove the solution:
1. Go to the _resource group_ you created for this solution.
2. Click **Delete** at the top of the screen.
