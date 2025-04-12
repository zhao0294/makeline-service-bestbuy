# Table of Contents

- [Table of Contents](#table-of-contents)
- [Makeline Service](#makeline-service)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Setup Instructions](#setup-instructions)
    - [1. Configure Azure Service Bus (Message Queue)](#1-configure-azure-service-bus-message-queue)
      - [Option 1: Azure CLI](#option-1-azure-cli)
      - [Option 2: Azure Portal](#option-2-azure-portal)
    - [2. Configure MongoDB (Database)](#2-configure-mongodb-database)
    - [3. Run the Application Locally](#3-run-the-application-locally)
  - [Testing the API](#testing-the-api)
  - [Querying MongoDB](#querying-mongodb)
  - [Additional Notes](#additional-notes)
  - [Summary: Makeline Service Environment Variables](#summary-makeline-service-environment-variables)
    - [Azure Service Bus Related Environment Variables](#azure-service-bus-related-environment-variables)
    - [MongoDB Related Environment Variables](#mongodb-related-environment-variables)
    - [GIN Mode Setting](#gin-mode-setting)

---
# Makeline Service

The **Makeline Service** is a Go-based REST API built with the [Gin framework](https://github.com/gin-gonic/gin). It processes orders from a message queue (Azure Service Bus) and stores them in a MongoDB database. This service is designed to work alongside the store-admin application.

## Overview

- **Purpose**: Process and manage orders via a REST API.
- **Tech Stack**: Go, Gin framework, Azure Service Bus (AMQP 1.0), MongoDB.
- **Dependencies**: Requires a message queue and database to function.

## Prerequisites

Before running the application, ensure you have the following installed:

- [Go](https://golang.org/doc/install) (version 1.18 or later recommended)
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- [MongoSH](https://www.mongodb.com/docs/mongodb-shell/install/) (for MongoDB interaction)

## Setup Instructions

### 1. Configure Azure Service Bus (Message Queue)

The application uses Azure Service Bus with AMQP 1.0 for message queueing. You must configure a Service Bus namespace and a queue named `orders`. Below are two methods to set this up:

#### Option 1: Azure CLI

1. **Create a Resource Group and Service Bus Namespace**:
  
  ```bash
  RGNAME=<resource-group-name>
  LOCNAME=<location>
  
  az group create --name $RGNAME --location $LOCNAME
  az servicebus namespace create --name <namespace-name> --resource-group $RGNAME
  az servicebus queue create --name orders --namespace-name <namespace-name> --resource-group $RGNAME
  ```
  
2. **Choose Authentication Method**:
  
  - **Microsoft Entra Workload Identity (Recommended)**:
    
    Assign the Azure Service Bus Data Receiver role to your account:
    
    ```bash
    PRINCIPALID=$(az ad signed-in-user show --query objectId -o tsv)
    SERVICEBUSBID=$(az servicebus namespace show --name <namespace-name> --resource-group $RGNAME --query id -o tsv)
    az role assignment create --role "Azure Service Bus Data Receiver" --assignee $PRINCIPALID --scope $SERVICEBUSBID
    ```
    
    Set environment variables:
    
    ```bash
    HOSTNAME=$(az servicebus namespace show --name <namespace-name> --resource-group $RGNAME --query serviceBusEndpoint -o tsv | sed 's/https:\/\///;s/:443\///')
    export ORDER_QUEUE_HOSTNAME=$HOSTNAME
    export ORDER_QUEUE_NAME=orders
    export USE_WORKLOAD_IDENTITY_AUTH=true
    ```
    
  - **Shared Access Policy**:
    
    Create a policy with Listen permissions:
    
    ```bash
    az servicebus namespace authorization-rule create --name listener --namespace-name <namespace-name> --resource-group $RGNAME --rights Listen
    ```
    
    Retrieve connection details and set environment variables:
    
    ```bash
    HOSTNAME=$(az servicebus namespace show --name <namespace-name> --resource-group $RGNAME --query serviceBusEndpoint -o tsv | sed 's/https:\/\///;s/:443\///')
    PASSWORD=$(az servicebus namespace authorization-rule keys list --namespace-name <namespace-name> --resource-group $RGNAME --name listener --query primaryKey -o tsv)
    export ORDER_QUEUE_URI=amqps://$HOSTNAME
    export ORDER_QUEUE_LISTENER_USERNAME=listener
    export ORDER_QUEUE_LISTENER_PASSWORD=$PASSWORD
    export ORDER_QUEUE_NAME=orders
    export ORDER_QUEUE_TRANSPORT=tls
    ```
    

#### Option 2: Azure Portal

1. **Create a Resource Group**:
  
  - Navigate to the [Azure Portal](https://portal.azure.com).
  - Search for **Resource Groups** and click **+ Create**.
  - Enter a **Resource Group Name** and select a **Region**.
  - Click **Review + Create**, then **Create**.
2. **Create a Service Bus Namespace**:
  
  - Search for **Service Bus** and click **+ Create**.
  - Provide:
    - **Namespace Name** (globally unique)
    - **Pricing Tier**: Basic (or Standard if needed)
    - **Region**: Same as your resource group
    - **Resource Group**: Select the created group
  - Click **Review + Create**, then **Create**.
3. **Create a Queue**:
  
  - Go to your Service Bus namespace.
  - In the left panel, select **Queues** and click **+ Queue**.
  - Name the queue `orders` and keep defaults.
  - Click **Create**.
4. **Set Up Shared Access Policy**:
  
  - In the namespace, go to **Shared Access Policies** and click **+ Add**.
  - Name the policy `listener` and enable **Listen** permission.
  - Click **Create**.
  - Open the policy to copy:
    - **Primary Connection String**
    - **Primary Key**
5. **Set Environment Variables**:
  
  ```bash
  export ORDER_QUEUE_URI=amqps://your-namespace.servicebus.windows.net
  export ORDER_QUEUE_LISTENER_USERNAME=listener
  export ORDER_QUEUE_LISTENER_PASSWORD=YourPrimaryKeyHere
  export ORDER_QUEUE_NAME=orders
  export ORDER_QUEUE_TRANSPORT=tls
  ```
  

> **Note**: If the `order-service` writes to Azure Service Bus, update the `docker-compose.yml` file with the correct environment variables for `order-service`, including `ORDER_QUEUE_TRANSPORT=tls` for a secure connection.

### 2. Configure MongoDB (Database)

For a local MongoDB instance (e.g., via Docker), set the following environment variables:

```bash
export ORDER_DB_URI=mongodb://localhost:27017
export ORDER_DB_NAME=orderdb
export ORDER_DB_COLLECTION_NAME=orders
```

### 3. Run the Application Locally

The application depends on Azure Service Bus and MongoDB. To simulate order generation, you can run the order-service and virtual-customer applications alongside makeline-service. A docker-compose.yml file is provided for convenience.

1. **Clone the Repository**:
  
  ```bash
  git clone <repository-url> 
  cd makeline-service
  ```
  
2. **Start Dependencies**:
  
  ```bash
  docker compose up -d
  ```
  
3. **Run the Application**:
  
  ```bash
  go get . 
  go run .
  ```
  
  Upon successful startup, you should see output like this:
  
  ```text
  [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached. 
  [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production. 
  - using env: export GIN_MODE=release 
  - using code: gin.SetMode(gin.ReleaseMode) 
  [GIN-debug] GET /order/fetch --> main.fetchOrders (4 handlers) 
  [GIN-debug] GET /order/:id --> main.getOrder (4 handlers) 
  [GIN-debug] PUT /order --> main.updateOrder (4 handlers) 
  [GIN-debug] GET /health --> main.main.func1 (4 handlers) 
  [GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value. 
  Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details. 
  [GIN-debug] Listening and serving HTTP on :3001
  ```
  

## Testing the API

To test the API endpoints, use the provided test-makeline-service.http file. This requires:

- [Visual Studio Code](https://code.visualstudio.com/)
- [REST Client Extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)

Open test-makeline-service.http in VS Code and send requests using the REST Client.

## Querying MongoDB

To verify orders in the MongoDB database:

1. Connect to MongoDB:
  
  ```bash
  mongosh
  ```
  
2. List databases and switch to orderdb:
  
  ```bash
  show dbs
  use orderdb
  ```
  
3. Check collections and query orders:
  
  ```bash
  show collections 
  db.orders.find() # View all orders
  db.orders.findOne({status: 1}) # View completed orders
  ```
  

## Additional Notes

- Ensure all environment variables are set correctly before running the application.
- For production, set GIN_MODE=release to disable debug mode (export GIN_MODE=release).
- If you encounter issues, verify connectivity to Azure Service Bus and MongoDB.

## Summary: Makeline Service Environment Variables

### Azure Service Bus Related Environment Variables

| Environment Variable            | Description                                           | Example Value                                   |
| ------------------------------- | ----------------------------------------------------- | ----------------------------------------------- |
| `ORDER_QUEUE_URI`               | Azure Service Bus URI                                 | `amqps://your-namespace.servicebus.windows.net` |
| `ORDER_QUEUE_LISTENER_USERNAME` | Listener username for Service Bus                     | `listener`                                      |
| `ORDER_QUEUE_LISTENER_PASSWORD` | Listener password for Service Bus                     | `YourPrimaryKeyHere`                            |
| `ORDER_QUEUE_NAME`              | Queue name in Service Bus                             | `orders`                                        |
| `ORDER_QUEUE_TRANSPORT`         | Connection transport type                             | `tls`                                           |
| `USE_WORKLOAD_IDENTITY_AUTH`    | Whether to use Azure Workload Identity authentication | `true`                                          |

### MongoDB Related Environment Variables

| Environment Variable       | Description             | Example Value               |
| -------------------------- | ----------------------- | --------------------------- |
| `ORDER_DB_URI`             | MongoDB connection URI  | `mongodb://localhost:27017` |
| `ORDER_DB_NAME`            | MongoDB database name   | `orderdb`                   |
| `ORDER_DB_COLLECTION_NAME` | MongoDB collection name | `orders`                    |

### GIN Mode Setting

| Environment Variable | Description                | Example Value                              |
| -------------------- | -------------------------- | ------------------------------------------ |
| `GIN_MODE`           | Gin framework running mode | `release` (set for production environment) |