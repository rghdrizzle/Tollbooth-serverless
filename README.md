# Tollbooth-serverless
A What-The-hack challenge

<p>The solution begins with vehicle photos being uploaded to an Azure Storage blob storage container, as they are captured. A blob storage trigger fires on each image upload, executing the photo processing Azure Function endpoint (on the side of the diagram), which in turn sends the photo to the Cognitive Services Computer Vision API OCR service to extract the license plate data.

If processing was successful and the license plate number was returned, the function submits a new Event Grid event, along with the data, to an Event Grid topic with an event type called "savePlateData". However, if the processing was unsuccessful, the function submits an Event Grid event to the topic with an event type called "queuePlateForManualCheckup".

Two separate functions are configured to trigger when new events are added to the Event Grid topic, each filtering on a specific event type, both saving the relevant data to the appropriate Azure Cosmos DB collection for the outcome, using the Cosmos DB output binding.

A Logic App that runs on a 15-minute interval executes an Azure Function via its HTTP trigger, which is responsible for obtaining new license plate data from Cosmos DB and exporting it to a new CSV file saved to Blob storage. If no new license plate records are found to export, the Logic App sends an email notification to the Customer Service department via their Office 365 subscription.

Application Insights is used to monitor all of the Azure Functions in real-time as data is being processed through the serverless architecture. This real-time monitoring allows you to observe dynamic scaling first-hand and configure alerts when certain events take place.<p>
<img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/infrastructure.png>
# Walkthrough
  ## Provisioning
To implement the solution , we have to provision the following resources:
  
  1.An Azure Cosmos DB account 

    API : Core (SQL)
    a container
        Database ID "LicensePlates"
        Uncheck Provision database throughput
        Container ID "Processed"
        Partition key : "/licensePlateText"
    a second container
        Database ID created above "LicensePlates"
        Container ID "NeedsManualReview"
        Partition key : "/fileName"

2. A storage account with a container "images" and another container "export"

3. A function app with .NET runtime stack

4. A function app (event) with Node.js runtime stack

5. An Event Grid Topic 
  
6. A Computer Vision API service (S1 pricing tier)(In azure cognitive service)
  
7. A Key Vault with the following secrets
  
  | Name| Value |
  | --- | --- |
  | computerVisionApiKey| computer Vision API key |
  | eventGridTopicKey 	Event| Grid Topic access key |
  | cosmosDBAuthorizationKey| Cosmos DB Primary Key |
  | blobStorageConnection|Blob storage connection string |
  
  The resources provisioned (11 of them):
  <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(96).png>
  <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(97).png>
  Secrets created in the resource KeyVault:
  <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(108).png>
 # Configuration
Next I configured application setting in the first function app with the following key:value pairs:
| Application key | value |
| --- | --- |
|computerVisionApiUrl |Computer Vision API endpoint by appending vision/v2.0/ocr to the end|
|computerVisionApiKey 	|computerVisionApiKey from Key Vault|
|eventGridTopicEndpoint |	Event Grid Topic endpoint|
|eventGridTopicKey 	|eventGridTopicKey from Key Vault|
|cosmosDBEndPointUrl |	Cosmos DB URI|
|cosmosDBAuthorizationKey |	cosmosDBAuthorizationKey from Key Vault|
|cosmosDBDatabaseId |	Cosmos DB database id (LicensePlates)|
|cosmosDBCollectionId |	Cosmos DB processed collection id (Processed)|
|exportCsvContainerName |	Blob storage CSV export container name (export)|
|blobStorageConnection |	blobStorageConnection from Key Vault|
<img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(109).png>

For referencing the secrets from the KeyVault I followed the steps from this <a href=https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references?tabs=azure-cli>doc.</a>
Then I published the tollbooth app to the function app through vscode. You can take a look at those functions in the repo. There are two functions , ProcessImage and ExportLicencePlate. After publishing the app to the function app , I then added the event grid subscription to the "Process Image" function. Similarly I published the function events (which u can take a look in the repo) to the function app and created event grid subscription to each of those event functions which you can see in the image below.
<img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(110).png>
Then I integrated these event functions with cosmosDb to process the output and store the data in the container.
Function integration chart:
<img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(104).png> 
Then I provisioned a new resource(Application insights) for monitoring the whole architecture. I also configured the function apps to connect to the application insight . The file UploadImages is a cli application which asks the user the blobstorage connection key to upload images to the blob. So first I chose to upload 10 images, you can see the metrics below:
  <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(98).png>
  
Then I chose to upload 1000 images to observe the function, you can view the screenshot of the metrics below:
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(100).png>
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(101).png>
If you notice , the number of servers when I upload only 10 images was only 4 but when I was uplaoding 1000 images , the servers increased to 7 and few of them were not even used. So basically the servers scaled up by itself to handle the required requests. This also shows that serverless is truly sever less since we dont have to really manage anything , the provider takes care of it all.
  
The images that get stored in the cosmosdb containers after executing the function:
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(107).png>
After observing the infrastructure, I created a logic app . A logic app is an automated workflow which can be run on triggers. For this application the trigger reoccurs every 15 mins. Then the workflow will call the function ExportLicensePlates and then I added a condition control where the status code has to be 200. If the function returns something else the workflow will send an email notifying me about it. If the status code is 200, then the function will export the licence plate numbers to a csv file in the export container which we created earlier. 
Logic design:
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(103).png>
 This is the final resources in the resource group after completion:
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(105).png>
 <img src=https://github.com/rghdrizzle/Tollbooth-serverless/blob/main/Screenshot%20(106).png>
 Well the solution is now complete and fully functional. This project taught me how to work with new services such as computervision , CosmosDB , EventGrids and logic apps and I had fun playing the platform and experimenting stuffs.
 # Thank you for going through this :)
 
  
  <img src=https://media.giphy.com/media/7J4P7cUur2DlErijp3/giphy.gif>
