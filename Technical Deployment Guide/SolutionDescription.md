# Predictive Analytics for Customer Churn in Retail

1. Approach
===============
A supervised machine learning model is used in the solution. The response and predictor variables are defined as follows. The response variable is binary with 1 as churn and 0 as non-churn. We say a customer churned when that customer spent no money at the store during the last 21 days. This definition can be customized by two factors: the number of days from today and the amount of money spent. For example, some businesses might define a churned customer as someone who has made less than $10 in purchases over the last 30 days. 

Through feature engineering we developed around 20 features to be used in the model. Some example features include:
- number of days between 2 consecutive transactions (average)
- number of cumulative store visits 
- money spent per visit (average)
- variations in money spent per visit
2. Data Sources
===============
The solution demo uses the public [Tafeng](http://recsyswiki.com/wiki/Grocery_shopping_datasets) dataset which can be found in this GitHub repository's [resource folder](https://github.com/Azure/cortana-intelligence-churn-prediction-solution/tree/master/Technical%20Deployment%20Guide/resource). The data contain 4 months transaction information. Out of these, the first 2 months data are used as historical data and the last 2 months data are used to emulate live data stream over time. 
3. ML Implementation
===============
The machine learning model of the solution is implemented using [Azure ML Studio](https://studio.azureml.net/). The training experiments use a two class boosted decision tree, selecting variables using the Azure ML Studio "Filter Based Feature Selection" module.

The corresponding predictive experiment is [published](https://gallery.cortanaintelligence.com/Experiment/Retail-Churn-Predictive-Exp-1) with this solution. 

When adapting this solution to other datasets, the ML section can be expanded as needed:  
- More preprocessing [Azure ML Studio](https://studio.azureml.net/) modules for [feature engineering](https://msdn.microsoft.com/en-us/library/azure/dn905834.aspx) can be added.  
- Different machine learning [algorithms](https://msdn.microsoft.com/en-us/library/azure/dn905812.aspx) for Classification or Regression can be added and compared.  

The model used in this solution is based on an Azure ML collection of experiments is published [here](https://gallery.cortanaintelligence.com/Collection/Retail-Customer-Churn-Prediction-Template-1). The collection of experiments include details on how to prepare data sources, create features, set up machine learning models, and evaluating models.


