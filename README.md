# Udacity Cloud Developer using Microsoft Azure Nanodegree Program - Project: Deploy the Neighborly App with Azure Functions

- [Introduction](#introduction)
- [Getting Started](#getting-started)
  - [Dependencies](#dependencies)
- [Instructions](#instructions)
  - [Creating Azure Function app](#creating-azure-function-app)
    - [Login with Azure CLI](#login-with-azure-cli)
    - [Create a Resource Group](#create-a-resource-group)
    - [Create a storage account](#create-a-storage-account)
    - [Create an Azure Function App](#create-an-azure-function-app)
    - [Set up a Cosmos DB Account](#set-up-a-cosmos-db-account)
    - [Create MongoDB database and collections](#create-mongodb-database-and-collections)
    - [Get connection string](#get-connection-string)
  - [Import sample data](#import-sample-data)
    - [Add connection string to API files](#add-connection-string-to-api-files)
    - [Deploy your Azure Functions](#deploy-your-azure-functions)
  - [Deploying the client-side Flask web application](#deploying-the-client-side-flask-web-application)
  - [CI/CD deployment](#cicd-deployment)
    - [Deploy your client app](#deploy-your-client-app)
    - [TBD](#tbd)
  - [Event Hubs and Logic App](#event-hubs-and-logic-app)
- [Clean-up](#clean-up)
- [Screenshots](#screenshots)
- [References](#references)
- [Requirements](#requirements)
- [License](#license)

## Introduction

Neighborly is a Python Flask-powered web application that allows neighbors to post advertisements for services and products they can offer.

The Neighborly project is comprised of a front-end application that is built with the Python Flask micro framework. The application allows the user to view, create, edit, and delete the community advertisements.

The application makes direct requests to the back-end API endpoints. These are endpoints that we will also build for the server-side of the application.

You can see an example of the deployed app below:

![Deployed App](images/final-app.png)

## Getting Started

1. Clone this repository
2. Ensure you have all the dependencies
3. Follow the instructions below

### Dependencies

You will need to install the following locally:

- [Pipenv](https://pypi.org/project/pipenv/)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Azure Function tools V3](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack)
- [MongoDB](https://docs.mongodb.com/manual/installation/)

## Instructions

### Creating Azure Function app

#### Login with Azure CLI

This project uses your Azure user and the Azure CLI to login and execute commands:

```bash
az login
```

Check [Create an Azure service principal with the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest) if you prefer using a service principal instead.

#### Create a Resource Group

Run the code below to create a new Resource Group:

```bash
az group create \
    --name neighborly-app-rg \
    --location eastus \
    --tags "dept=Engineering" \
    --tags "environment=Production" \
    --tags "project=Udacity Neighborly App" \
    --tags "createdby=CLI"
```

#### Create a storage account

Create a storage account (within the previously created resource group and region):

```bash
az storage account create \
    --name neighborlyst \
    --resource-group neighborly-app-rg \
    --location eastus
```

Then we create our container: 

TBD: Is it required?

```bash
az storage container create \
    --account-name neighborlyst \
    --name images \
    --auth-mode login \
    --public-access container
```

#### Create an Azure Function App

Create an Azure Function App within the resource group, region and storage account:

```bash
az functionapp create \
    --name neighborly-api-v1 \
    --resource-group neighborly-app-rg \
    --storage-account neighborlyst \
    --functions-version 3 \
    --os-type Linux \
    --runtime python \
    --runtime-version 3.8 \
    --consumption-plan-location eastus
```

- Note that app names need to be unique across all of Azure
- Make sure it is a Linux app, with a Python runtime

#### Set up a Cosmos DB Account

You will need to use the same resource group, region and storage account, but can name the Cosmos DB account as you prefer:

```bash
COSMOS_ACCOUNT="neighborly-app-cosmos"
RESOURCE_GROUP="neighborly-app-rg"
REGION="eastus"

az cosmosdb create \
    --name $COSMOS_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --kind MongoDB \
    --locations regionName=$REGION failoverPriority=0 isZoneRedundant=False 
```

This step may take a little while to complete (15-20 minutes in some cases).

#### Create MongoDB database and collections

Create a MongoDB Database in CosmosDB Azure and two collections, one for `advertisements` and one for `posts`:

```bash
DATABASE_NAME="neighborly-app-cosmos-db"
CREATE_LEASE_COLLECTION=0 # yes,no=(1,0)

COLLECTION_ADS="advertisements"
COLLECTION_POSTS="posts"

az cosmosdb mongodb database create \
    --account-name $COSMOS_ACCOUNT \
    --name $DATABASE_NAME \
    --resource-group $RESOURCE_GROUP

az cosmosdb mongodb collection create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_ACCOUNT \
    --database-name $DATABASE_NAME \
    --name $COLLECTION_ADS \
    --throughput 400

az cosmosdb mongodb collection create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_ACCOUNT \
    --database-name $DATABASE_NAME \
    --name $COLLECTION_POSTS \
    --throughput 400
```

#### Get connection string

Copy/paste the primary connection string, it will be used later in the application:

```bash
az cosmosdb keys list \
    --name $COSMOS_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --type connection-strings
```

### Import sample data

Import the data from the `sample_data` directory for ads and posts to initially fill your app:

```bash
MONGODB_HOST="$COSMOS_ACCOUNT.mongo.cosmos.azure.com"
MONGODB_PORT="10255"

# Copy/past the primary password here
PRIMARY_PW="MJFwUScqM9g1MoFeoSb5IbTOfIoXkMLvMCfeQSkeLqiiLVHHiGbpqm2oysFQa5oueRQifbyv1QT7JBuayDIkrQ=="

mongoimport -h $MONGODB_HOST:$MONGODB_PORT \
    --writeConcern="{w:0}" \
    -d $DATABASE_NAME \
    -c $COLLECTION_ADS \
    -u $COSMOS_ACCOUNT \
    -p $PRIMARY_PW \
    --ssl --jsonArray --file sample_data/sampleAds.json

mongoimport -h $MONGODB_HOST:$MONGODB_PORT \
    --writeConcern="{w:0}" \
    -d $DATABASE_NAME \
    -c $COLLECTION_POSTS \
    -u $COSMOS_ACCOUNT \
    -p $PRIMARY_PW \
    --ssl --jsonArray --file sample_data/samplePosts.json
```

You can get the primary password from the Azure Portal on the Azure Cosmos DB settings ("Connection String").

#### Add connection string to API files

You need to hook up your connection string into the NeighborlyAPI server folder. You will need to replace the *url* variable with your own connection string you copy-and-pasted in the last step, along with some additional information. Check out [this post](https://docs.microsoft.com/en-us/azure/cosmos-db/connect-mongodb-account) if you need help with what information is needed.

Go to each of the `__init__.py` files in `getPosts`, `getPost`, `getAdvertisements`, `getAdvertisement`, `deleteAdvertisement`, `updateAdvertisement`, `createAdvertisements` and replace your connection string. You will also need to set the related `database` and `collection` appropriately.

#### Deploy your Azure Functions

Test it out locally first. To do so, you have to install the dependencies:

```bash
pipenv install
pipenv shell
```

Switch to the API folder, fetch the app settings and run the functions locally:

```bash
cd NeighborlyAPI
```

Fetch app settings:

```bash
func azure functionapp fetch-app-settings neighborly-api-v1
```

Add connection string to ```local.settings.json```:

```bash
{
  "IsEncrypted": true,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "<from fetch-app-settings>".
    "FUNCTIONS_EXTENSION_VERSION": "<from fetch-app-settings>".
    "AzureWebJobsStorage": "<from fetch-app-settings>".
    "APPINSIGHTS_INSTRUMENTATIONKEY": "<from fetch-app-settings>".
    "MongoDBConnectionString": "", # TODO: set your MongoDB connection string
  },
  "ConnectionStrings": {}
}
```

Run functions locally:

```bash
func start
```

You may need to change `"IsEncrypted"` to `false` in `local.settings.json` if this fails.

At this point, Azure functions are hosted in [localhost:7071](http://localhost:7071). For example:

- [http://localhost:7071/api/getAdvertisements](http://localhost:7071/api/getAdvertisements)
- [http://localhost:7071/api/getPosts](http://localhost:7071/api/getPosts)

If everything works, you can deploy the functions to Azure by publishing your function app:

```bash
func azure functionapp publish neighborly-api-v1
```

Add the MongoDB connection string as a new application setting in your function app:

![Application setting](images/application-setting.png)

Save the function app url [https://neighborly-api-v1.azurewebsites.net/api/](https://neighborly-api-v1.azurewebsites.net/api/) since you will need to update that in the client-side of the application.

### Deploying the client-side Flask web application

We are going to update the Client-side `settings.py` with published API endpoints. First navigate to the `settings.py` file in the `NeighborlyFrontEnd/` directory.

Use a text editor to update the API_URL to your published url from the last step:

```bash
# Inside file settings.py

# ------- For Local Testing -------
#API_URL = "http://localhost:7071/api"

# ------- For production -------
# where APP_NAME is your Azure Function App name 
API_URL="https://neighborly-api-v1.azurewebsites.net/api"
```

### CI/CD deployment

#### Deploy your client app

Use a different app name here to deploy the front-end, or else you will erase your API. From within the `NeighborlyFrontEnd` directory:

- Install dependencies with `pipenv install`
- Go into the pip env shell with `pipenv shell`
- Deploy your application to the app service:

    ```bash
    az webapp up --sku F1 -n neighborly-app --resource-group neighborly-app-rg
    ```

- If you want to update your app, make changes to your code and then run:

    ```bash
    az webapp up \
        --name neighborly-app \
        --verbose
    ```

Make sure to also provide any necessary information in `settings.py` to move from localhost to your deployment (see [above](#deploying-the-client-side-flask-web-application)).

#### TBD

1. Create an Azure Registry and dockerize your Azure Functions. Then, push the container to the Azure Container Registry.
2. Create a Kubernetes cluster, and verify your connection to it with `kubectl get nodes`.
3. Deploy app to Kubernetes, and check your deployment with `kubectl config get-contexts`.

### Event Hubs and Logic App

1. Create a Logic App that watches for an HTTP trigger. When the HTTP request is triggered, send yourself an email notification.
2. Create a namespace for event hub in the portal. You should be able to obtain the namespace URL.
3. Add the connection string of the event hub to the Azure Function.

## Clean-up

TBD

Clean up and remove all services, or else you will incur charges:

```bash
# replace with your resource group
RESOURCE_GROUP="<YOUR-RESOURCE-GROUP>"
# run this command
az group delete --name $RESOURCE_GROUP
```

## Screenshots

1. TBD

![]()

## References

- [Develop, test, and deploy an Azure Function with Visual Studio](https://docs.microsoft.com/en-us/learn/modules/develop-test-deploy-azure-functions-with-visual-studio)
- [Work with NoSQL data in Azure Cosmos DB](https://docs.microsoft.com/en-us/learn/paths/work-with-nosql-data-in-azure-cosmos-db)

## Requirements

Graded according to the [Project Rubric](https://review.udacity.com/#!/rubrics/2825/view).

## License

- **[MIT license](http://opensource.org/licenses/mit-license.php)**
- Copyright 2021 © [Thomas Weibel](https://github.com/thom).
