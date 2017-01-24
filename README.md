# Retail Customer Churn Prediction - A Cortana Intelligence Solution How-to Guide

Keeping existing customers is five times cheaper than the cost of attaining new ones.[1] For this reason, marketing executives often find themselves trying to estimate the likelihood of customer
churn and finding the necessary actions to minimize the churn rate. 

Customer Churn Prediction uses Azure Machine Learning to predict churn probability and helps find patterns in existing data associated with the
predicted churn rate. This information empowers businesses with actionable intelligence to improve customer retention and profit margins.

The objective of this guide is to demonstrate predictive data pipelines for retailers to predict customer churn.  Retailers can use these predictions to prevent customer churn by using their domain knowledge and proper marketing strategies to address at-risk customers. The guide also shows how customer churn models can be retrained to leverage additional data as it becomes available.

## What's Under the Hood
The end-to-end solution is implemented in the cloud, using Microsoft Azure. The solution is composed of several Azure components, including data ingest, data storage, data movement, advanced analytics and visualization. The advanced analytics are implemented in Azure Machine Learning Studio, where one can use Python or R language to build data science models (or reuse existing in-house or third-party libraries).  With data ingest, the solution can make predictions based on data that being transferred to Azure from an on-premises environment.

## Solution Dashboard
The snapshot below shows an example PowerBI dashboard that gives insights into the the predicted churn rates across the customer base.
![Insights](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/blob/master/Technical%20Deployment%20Guide/media/customer-churn-dashboard-1.png)

## Solution Architecture
![Solution Diagram Picture](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/blob/master/Technical%20Deployment%20Guide/media/architecture.png)
## Getting Started

This solution package contains materials to help both technical and business audiences understand our  Customer Churn Prediction Solution built on the [Cortana Intelligence Suite](https://www.microsoft.com/en-us/server-cloud/cortana-intelligence-suite/Overview.aspx).

## Business Audiences

In this repository you will find a folder labeled [*Solution Overview for Business Audiences*](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Solution%20Overview%20for%20Business%20Audiences) which contains a  presentation covering this solution and benefits of using the Cortana Intelligence Suite

For more information on how to tailor Cortana Intelligence to your needs [connect with one of our partners](http://aka.ms/CISFindPartner).

## Technical Audiences

See the [*Technical Deployment Guide*](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide) folder for a full set of instructions on how to put together and deploy a Customer Churn Prediction Solution using the Cortana Intelligence Suite. For technical problems or questions about deploying this solution, please post in the issues tab of the repository.

## Related Resources
- [Customer Churn Machine Learning Template](https://gallery.cortanaintelligence.com/Collection/Retail-Customer-Churn-Prediction-Template-1) - This template walks through the steps required to build a machine learning model to predict customer churn using Azure Machine Learning.
- [Customer Churn Template with SQL Server R Services](https://gallery.cortanaintelligence.com/Tutorial/Customer-Churn-Prediction-Template-with-SQL-Server-R-Services-1) - This template also walks through the steps to build a machine learning model for customer churn, but is implemented within SQL Server for data stored on-premise.
