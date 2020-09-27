# Sentiment Analysis

The purpose of this lab is to show how you can apply in real-time a machine learning
model on streaming data. This use case will apply sentiment analysis on an incoming
stream of Twitter tweets.

## Prerequisites

To execute this lab successfully, you need the following:

• An Azure subscription. You can create a free one over [here](https://azure.microsoft.com/en-us/free/).

• An Azure Machine Learning Studio workspace. you can create a free one
over [here](https://studio.azureml.net/)

• A Power BI pro subscription. You can create a 60-day trial over [here](https://signup.microsoft.com/signup?sku=a403ebcc-fae0-4ca2-8c8c-7a907fd6c235&email&ru=https%3A%2F%2Fapp.powerbi.com%3Fpbi_source%3Dweb%26redirectedFromSignup%3D1%26noSignUpCheck%3D1).

## Solution design

The high level solution design of this lab looks like this.

![](Images/Images/images:sentimentanalytics:*.png/image--000.png)

• Two Logic Apps are capturing tweets that contain #happy or #sad

• These tweets are ingested into Event Hubs

• Azure Stream Analytics performs the sentiment analysis against an Azure Mache
Learning web service

• Azure Stream Analytics also calculates the average value over a specific time
window

• The results are outputted to Power BI, where they can be easily visualized

## Ingest tweets

### Create an Event Hub

First of all, we need a messaging service that can handle huge amounts of streaming
data. Azure Event Hubs is a great service that offers all features to build a realtime data
ingestion pipeline.

• Sign in to the [Azure portal](https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize?client_id=c44b4083-3bb0-49c1-b47d-974e53cbdf3c&response_type=code%20id_token&scope=https%3A%2F%2Fmanagement.core.windows.net%2F%2Fuser_impersonation%20openid%20email%20profile&state=OpenIdConnect.AuthenticationProperties%3DG-CudUDi8KW-XgKNmi4ZG5wXoDitdjCzpCwqMsrQATL1077wWP4onUPNzPiXTi7Z4C3dQ2aOo16kAtxI7FftyrkiEopsXjs-_cNSJF3QlBF7Pj5xmh_Cb1ZltyN0dQGpZqgAr6_lp2cyeDrOfvRCfXuX75pQgQupLRNbeg4zxQyKDPNOynjvNv-PU8hpY2c0HhSnLsMgm_46exx4kIuSh-jGo0bDn2mdasCZpNmFug6f8xd0KDoTaTsEgW7zEVmZKAfeRk0A2beK4s2rupezfQPcR3y7FQo7RijXveovxESvSWvONQh6LjYlzIdY9bG6XwFzEGJLftP6I83RTow0o5XtABb3yZXmiPr5schm-I5kFEZTetKZrUkWni7y1TMS&response_mode=form_post&nonce=637368226324609634.ZDU3NzZkY2UtYWI0OC00MzQzLWI1MDQtYjVlYTUyMjI3NGEyNTcwYmU4OGMtNmU5ZS00YjlhLWJlY2MtMDdmYzk5NTVlODYy&redirect_uri=https%3A%2F%2Fportal.azure.com%2Fsignin%2Findex%2F&site_id=501430&client-request-id=779c199b-1cd0-4069-9e44-b5675cc5074e&x-client-SKU=ID_NET45&x-client-ver=5.3.0.0) by using the credentials for your Azure subscription.

• In the upper-left corner of Azure portal, select + Create a resource.

![](Images/Images/images:sentimentanalytics:*.png/image--001.png)

• Use the search bar to find Event Hubs

• Select Event Hubs

• Click Create

![](Images/Images/images:sentimentanalytics:*.png/image--002.png)

• Provide the following information to configure your new eventhub.

## Field : Description

**Resource group** : Use an existing resource group in your subscription or enter a name to create a new resource group. A resource group holds related resources for an Azure solution.

**Namespace** : Enter a unique name that identifies your event hub namespace. Names must be unique across the resource group. *{prefix}-sentiment-analysis-ingestion*

**Subscription** : Select the Azure subscription that you want to use.

**Location** : Select the location closest to your users and the data resources to create your workspace.

**Pricing Tier** : Basic

**Throughput Units** : 1

![](Images/Images/images:sentimentanalytics:*.png/image--003.png)

• Click Next:Features

• Leave Defaults

• Click Next:Tags

• Click Next:Review+Create

• If the validation is successful, click create

![](Images/Images/images:sentimentanalytics:*.png/image--004.png)

• To view the new workspace, select Go to resource.

![](Images/Images/images:sentimentanalytics:*.png/image--005.png)

• Click on +Event Hub

![](Images/Images/images:sentimentanalytics:*.png/image--006.png)

• Create a new EventHub ingestion-eventHubs. A partition count of 2 and 1 day of
message retention is sufficient. No need to enable the capture feature.

![](Images/Images/images:sentimentanalytics:*.png/image--007.png)

Click Create

## Create an access policy

Each client that reads from the Event Hub needs to be assigned to a particular consumer
group. It's a good practice to give each consumer group a separate access policy, so you
can revoke each one of them separately.

• Navigate to the previously created Event Hub by selecting the event hub.

![](Images/Images/images:sentimentanalytics:*.png/image--008.png)

• Click on Shared access policies (under settings)

![](Images/Images/images:sentimentanalytics:*.png/image--009.png)

• Add here a new policy, that gives read access to Azure Stream Analytics

![](Images/Images/images:sentimentanalytics:*.png/image--010.png)

• Click Create

• Click on the created access policy and copy the connection string with primary
key. You'll need this later in this lab.

![](Images/Images/images:sentimentanalytics:*.png/image--011.png)

## Create a Logic App

In order to provide a simplified way to ingest tweets, we will use Azure Logic Apps.

• Go to the resource group you created earlier (you can search the resource group
in search bar on the portal page)

• Click on +Add

![](Images/Images/images:sentimentanalytics:*.png/image--012.png)

• Search for Logic app.

![](Images/Images/images:sentimentanalytics:*.png/image--013.png)

• Click Create

• Create a Logic App, named *{prefix}-sentiment-analysis-ingestion-happy*, choose
the same region as previously.

![](Images/Images/images:sentimentanalytics:*.png/image--014.png)

• Click Review+Create

• Click Create.

• Once deployed, click on “Go to resource”

• Choose to start from *Blank Logic App*.

![](Images/Images/images:sentimentanalytics:*.png/image--015.png)

## Add a trigger that receives specific tweets

This Logic App must fire each time a tweet contains a certain key word.

• Search for twitter in the search connector and trigger window

![](Images/Images/images:sentimentanalytics:*.png/image--016.png)

• Select Twitter

• Select When a new tweet is posted and authenticate with your Twitter account, by
clicking Signin.

![](Images/Images/images:sentimentanalytics:*.png/image--017.png)

• Click Authorize app in popup window

![](Images/Images/images:sentimentanalytics:*.png/image--018.png)

• Provide *#happy* as the hashtag to search for and poll every second.

![](Images/Images/images:sentimentanalytics:*.png/image--019.png)

## Send the tweets to Event Hubs

This Logic App has to send the captured tweets to Event Hubs.

• Below the trigger, click on New step to add an action to send to Event Hubs via
the Send event action.

• Search for event hub in search connector and action window

![](Images/Images/images:sentimentanalytics:*.png/image--020.png)

• Select event hub -> Send event

![](Images/Images/images:sentimentanalytics:*.png/image--021.png)

• Connect the action to the previously created Event Hub namespace and provide
connection name

![](Images/Images/images:sentimentanalytics:*.png/image--022.png)

• Select the event hub policy and click create

![]9Images/Images/images:sentimentanalytics:*.png/image--023.png)

• In the parameter drop-down select content

![](Images/Images/images:sentimentanalytics:*.png/image--024.png)

• Select the eventhub and add the following JSON structure:

                 {
                   "text" : "@{triggerBody()['TweetText']}",
                    "hashtag" : "#happy",
                    "time" : "@{utcNow()}"
                 }
                 
• This should result in the following Logic App:

![](Images/Images/images:sentimentanalytics:*.png/image--025.png)

• Click Save

![](mages/Images/images:sentimentanalytics:*.png/image--026.png)

• Go to the Overview blade and click Refresh. After a while, you should see
successful Logic App runs. All tweets that contain #happy are from now on being
ingested into your Event Hub.

![](Images/Images/images:sentimentanalytics:*.png/image--027.png)

### Repeat the above steps to create another Logic App that ingests tweets that contain *#sad*.

## Create a web service that performs sentiment analysis

In this step, we will create an Azure Machine Learning (AML) web service that performs
the sentiment analysis.

• Navigate to the Azure AI Gallery [experiment for sentiment analysis](https://gallery.azure.ai/Experiment/Predictive-Mini-Twitter-sentiment-analysis-Experiment-1).

• Click on Open in studio.




























