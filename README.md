# makeline-service

This is a Golang app that provides an API for processing orders. It is meant to be used in conjunction with the store-admin app.

It is a simple REST API written with the Gin framework that allows you to process orders from a RabbitMQ queue and send them to a MongoDB database.

## Prerequisites

- [Go](https://golang.org/doc/install)
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- [MongoSH](https://docs.mongodb.com/mongodb-shell/install/)

## Message queue options

This app connect to Azure Service Bus using AMQP 1.0, and you need to provide appropriate environment variables for connecting to the message queue.


### Get environment variables to connect to Azure Service Bus

To run this against Azure Service Bus, you will need to create a Service Bus namespace and a queue. You can do this using the Azure CLI or Azure Portal.

#### Option 1: Azure CLI

```bash
RGNAME=<resource-group-name>
LOCNAME=<location>

az group create --name $RGNAME --location $LOCNAME
az servicebus namespace create --name <namespace-name> --resource-group $RGNAME
az servicebus queue create --name orders --namespace-name <namespace-name> --resource-group $RGNAME
```

Once you have created the Service Bus namespace and queue, you will need to decide on the authentication method. You can create a shared access policy with the **Send** permission for the queue or use Microsoft Entra Workload Identity for a passwordless experience (this is the recommended approach).

If you choose to use Workload Identity, you will need to assign the `Azure Service Bus Data Receiver` role to the identity that is running the app, which in this case will be your account. You can do this using the Azure CLI.

```bash
PRINCIPALID=$(az ad signed-in-user show --query objectId -o tsv)
SERVICEBUSBID=$(az servicebus namespace show --name <namespace-name> --resource-group <resource-group-name> --query id -o tsv)

az role assignment create --role "Azure Service Bus Data Receiver" --assignee $PRINCIPALID --scope $SERVICEBUSBID
```

Next, get the hostname for the Azure Service Bus.

```bash
HOSTNAME=$(az servicebus namespace show --name <namespace-name> --resource-group <resource-group-name> --query serviceBusEndpoint -o tsv | sed 's/https:\/\///;s/:443\///')
```

Finally, set the environment variables.

```bash
export ORDER_QUEUE_HOSTNAME=$HOSTNAME
export ORDER_QUEUE_NAME=orders
export USE_WORKLOAD_IDENTITY_AUTH=true
```

If you choose to use a shared access policy, you can create one using the Azure CLI. Otherwise, you can skip this step and proceed to [provision a database](#database-options).

```bash
az servicebus namespace authorization-rule create --name listener --namespace-name <namespace-name> --resource-group $RGNAME --rights Listen
```

Next, get the connection information for the Azure Service Bus.

```bash
HOSTNAME=$(az servicebus namespace show --name <namespace-name> --resource-group $RGNAME --query serviceBusEndpoint -o tsv | sed 's/https:\/\///;s/:443\///')
PASSWORD=$(az servicebus namespace authorization-rule keys list --namespace-name <namespace-name> --resource-group $RGNAME --name listener --query primaryKey -o tsv)
```

Finally, set the environment variables.

```bash
export ORDER_QUEUE_URI=amqps://$HOSTNAME
export ORDER_QUEUE_LISTENER_USERNAME=listener
export ORDER_QUEUE_LISTENER_PASSWORD=$PASSWORD
export ORDER_QUEUE_NAME=orders
```

#### Option 2: Azure Portal

##### Step 1: Create a Resource Group

1. Go to [Azure Portal](https://portal.azure.com)
2. In the top search bar, search for **"Resource groups"**
3. Click **“+ Create”**
4. Fill in:
   - **Resource group name**
   - **Region**
5. Click **“Review + Create”** and then **“Create”**

---

##### Step 2: Create a Service Bus Namespace

1. In the top search bar, search for **"Service Bus"**
2. Click **“+ Create”**
3. Fill in:
   - **Namespace name** (must be globally unique)
   - **Pricing tier**: Choose **Basic** (or **Standard** if needed)
   - **Region**: Same as your resource group
   - **Resource group**: Select the one you created
4. Click **“Review + Create”**, then **“Create”**

---

##### Step 3: Create a Queue

1. Navigate to your **Service Bus namespace**
2. In the left panel, click **“Queues”**
3. Click **“+ Queue”**
4. Set a name ( `orders`)
5. Leave other settings as default
6. Click **“Create”**

---

##### Step 4: Create a Shared Access Policy (SAS Authentication)

1. In your Service Bus namespace, click **“Shared access policies”** in the left panel
2. Click **“+ Add”**
3. Fill in:
   - **Name** (`listener`)
   - **Permissions**: Select **Listen** 
4. Click **“Create”**
5. After creation, click the policy name to view:
   - **Primary Connection String**
   - **Primary Key**
   - **Policy Name**

---

##### Step 5: Get Hostname & Environment Variable Values

You can find connection details in two ways:

###### Option 1: From the Overview Tab

- Go to your **Service Bus namespace**
- Copy the **Service Bus Endpoint**, e.g.:sb://your-namespace.servicebus.windows.net/
- - Remove the prefix (`sb://`) and trailing slash to get:
- ORDER_QUEUE_HOSTNAME=your-namespace.servicebus.windows.net
###### Option 2: From the SAS Connection String

Example connection string:Endpoint=sb://your-namespace.servicebus.windows.net/;SharedAccessKeyName=listener;SharedAccessKey=YourPrimaryKeyHere
From this, extract:

```env
ORDER_QUEUE_HOSTNAME=your-namespace.servicebus.windows.net
ORDER_QUEUE_LISTENER_USERNAME=listener
ORDER_QUEUE_LISTENER_PASSWORD=YourPrimaryKeyHere
```
##### Step 6:set the environment variables.

```bash
export ORDER_QUEUE_URI=amqps://your-namespace.servicebus.windows.net
export ORDER_QUEUE_LISTENER_USERNAME=listener
export ORDER_QUEUE_LISTENER_PASSWORD=YourPrimaryKeyHere
export ORDER_QUEUE_NAME=orders
export ORDER_QUEUE_TRANSPORT=tls
```


> NOTE: If you are using Azure Service Bus, you will want your `order-service` to write orders to it correspondingly. If that is the case, then you'll need to update the [`docker-compose.yml`](./docker-compose.yml) and modify the environment variables for the `order-service` to include the proper connection info to connect to Azure Service Bus. Also you will need to add the `ORDER_QUEUE_TRANSPORT=tls` configuration to connect over TLS.

## Use MongoDB for Database 

If you are using a local MongoDB container, run the following commands:

```bash
export ORDER_DB_URI=mongodb://localhost:27017
export ORDER_DB_NAME=orderdb
export ORDER_DB_COLLECTION_NAME=orders
```

## Running the app locally

The app relies on Azure Service Bus and MongoDB. Additionally, to simulate orders, you will need to run the [order-service](../order-service) with the [virtual-customer](../virtual-customer) app. A docker-compose file is provided to make this easy.

To run the necessary services, clone the repo, open a terminal, and navigate to the `makeline-service` directory. Then run the following command:

```bash
docker compose up -d
```

Now you can run the following commands to start the application:

```bash
go get .
go run .
```

When the app is running, you should see output similar to the following:

```text
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /order/fetch              --> main.fetchOrders (4 handlers)
[GIN-debug] GET    /order/:id                --> main.getOrder (4 handlers)
[GIN-debug] PUT    /order                    --> main.updateOrder (4 handlers)
[GIN-debug] GET    /health                   --> main.main.func1 (4 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on :3001
```

Using the [`test-makeline-service.http`](./test-makeline-service.http) file in the root of the repo, you can test the API. However, you will need to use VS Code and have the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension installed.

To view the orders in MongoDB, open a terminal and run the following command:

```bash
# connect to mongodb
mongosh

# show databases and confirm orderdb exists
show dbs

# use orderdb
use orderdb

# show collections and confirm orders exists
show collections

# get the orders
db.orders.find()

# get completed orders
db.orders.findOne({status: 1})
```

To view the orders in Azure CosmosDB using `mongosh`, open a terminal an run the following command:

```bash
# connect to cosmosdb
mongosh -u $USERNAME -p $PASSWORD --tls --retryWrites=false mongodb://$COSMOSDBNAME.mongo.cosmos.azure.com:10255/orderdb

# show collections and confirm orders exists
show collections

# get the orders
db.orders.find()

# get completed orders
db.orders.findOne({status: 1})
```
