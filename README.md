# Real-time predictions on manufacturing quality using Azure Machine Learning

High level reference architecture:

![Architecture](https://github.com/jomit/real-time-predictions/blob/master/images/real-time-architecture.png)

### Business Case

Improve quality and reduce production scrap by predicting the quality of a production run near real time using the machine sensor data coming out of the PLC's / Historian.

### Prerequisites

- Active [Azure Subscription](https://azure.microsoft.com/en-us/free/)

- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)


### Steps

#### Training Pipeline

- Create [Azure Blob storage Account](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal) and upload `data\train.csv` in a container named `mldatasets`

- Create a new [Data Science Virtual Machine](https://docs.microsoft.com/en-us/azure/machine-learning/data-science-virtual-machine/dsvm-ubuntu-intro) and verify that the Jupyter notebook server is running on port 8000.

- Deploy the [Azure ML Workspace](https://docs.microsoft.com/en-us/azure/machine-learning/service/concept-workspace) using the `mlworkspace\template.json` template file

    - `az group deployment create --resource-group <resource-group-name> --template-file <path-to-template>`

- Upload all the notebooks from `mlnotebooks` folder to the Jupyter notebook server using the Web UI.

- Run all notebooks as per the sequence to build and deploy the initial model. 

    - Make sure to update the parameters in the notebook as per the instructions.

    - For production deployments use [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-deploy-and-where#aks) to deploy the scoring api

#### Inference Pipeline

- Create an [Event Hub Namespace](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-create) and an event hub inside it named `qualityprediction`

- Create a new [Cosmos DB Account](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-manage-database-account) and add a new container with following details:

    - Database id : `MLPrediction`
    - Container id: `qualitypredictions`
    - Partition key: `/batchid`

- Create an [Azure Function App](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-azure-function) with Runtime Stack as `Nodejs`

    - Add new Azure Event hub trigger function with following parameters:
        - Name: `PredictQuality`
        - Event Hub Connection: `<Use the event hub created above>`
        - Event Hub Name: `qualityprediction`
        - The event hub cardinality: `One`
        - Event parameter name: `eventHubMessage`
         
    - Click on `Integrate` in the function and add a New Output of type `Azure Cosmos DB` with following parameters:
        - Document parameter name: `predictionResultDocument`
        - Database name: `MLPrediction`
        - Collection name: `qualitypredictions`
        - Azure Cosmos DB account connection: `<Choose the Cosmos DB Account created above>`

    - Updated the function code with `function\index.js`. Make sure to replace the scoring api url.

- Setup Edge Gateway `\edgegateway` which is a nodejs app. (For more complex edge requirements please use [Azure IoT Edge](https://docs.microsoft.com/en-us/azure/iot-edge/iot-edge-as-gateway) along with [Azure IoT Hub](https://docs.microsoft.com/en-us/azure/iot-hub/about-iot-hub))

    - Rename the `sample-dot-env` file to `.env`

    - Update the `eventHubConnectionString` and `eventHubName` values in the .env file

    - (Optional) If integrating with local SQL DB 
        - Update the sql related values in the .env file
        - Update code in `app.js` and `sqlserverdb.js`

### Additional Resources

- [Scalability, Monitoring and Security considerations for using Azure Kubernetes Service to deploy ML Models](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/ai/realtime-scoring-python#scalability-considerations)

- [Scaling Azure Funcions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale)

- [Scaling Azure Event Hubs](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability)

