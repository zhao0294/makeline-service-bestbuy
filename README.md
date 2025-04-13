
# Makeline Service

**Makeline Service** is a RESTful API built in Go using the [Gin framework](https://github.com/gin-gonic/gin). It processes orders from Azure Service Bus and stores them in MongoDB. Designed to complement the store-admin application, it provides efficient order management.

----------

## 📌 Overview

- **Purpose**: Manage and process orders via a RESTful API.
- **Tech Stack**: Go, Gin, Azure Service Bus (AMQP 1.0), MongoDB.
- **Dependencies**: Requires a message queue (Azure Service Bus) and a database (MongoDB).

----------

## ✅ Prerequisites

Ensure the following tools are installed before proceeding:

- [Go](https://golang.org/doc/install) (v1.18 or later)
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- [MongoSH](https://www.mongodb.com/docs/mongodb-shell/install/) (for MongoDB interaction)

----------

## ⚙️ Setup Instructions

### 1. Configure Azure Service Bus

1. **Create a Resource Group**
    - Navigate to the [Azure Portal](https://portal.azure.com).
    - Search for **Resource Groups** and click **+ Create**.
    - Enter a name, select a region, and click **Review + Create**, then **Create**.
2. **Create a Service Bus Namespace**
    - Search for **Service Bus** and click **+ Create**.
    - Provide:
      - **Namespace Name** (globally unique).
      - **Pricing Tier**: Basic or Standard.
      - **Region**: Match the resource group.
      - **Resource Group**: Select the created group.
    - Click **Review + Create**, then **Create**.
3. **Create a Queue**
    - Open the Service Bus namespace.
    - Go to **Queues** > **+ Queue**.
    - Name it orders and accept default settings.
    - Click **Create**.
4. **Set Up Shared Access Policy**
    - In the namespace, go to **Shared Access Policies** > **+ Add**.
    - Name it listener and enable **Listen**.
    - Copy the **Primary Connection String** and **Primary Key**.
5. **Set Environment Variables**  
    Configure the following in your terminal:

```bash
    export ORDER_QUEUE_URI=amqps://<your-namespace>.servicebus.windows.net 
    export ORDER_QUEUE_LISTENER_USERNAME=listener 
    export ORDER_QUEUE_LISTENER_PASSWORD=<YourPrimaryKeyHere> 
    export ORDER_QUEUE_NAME=orders export ORDER_QUEUE_TRANSPORT=tls
```

**Note**: If order-service publishes to the queue, ensure it has matching environment variables, including ORDER_QUEUE_TRANSPORT=tls.

### 2. Configure MongoDB

For a local MongoDB instance (e.g., via Docker), set these environment variables:

```bash

    export ORDER_DB_URI=mongodb://localhost:27017 
    export ORDER_DB_NAME=orderdb 
    export ORDER_DB_COLLECTION_NAME=orders

```

### 3. Run the Application Locally

Ensure MongoDB and Azure Service Bus are running or configured.

1. **Clone the Repository**

```bash
    git clone <repository-url> cd makeline-service
```

2. **Start Dependencies**
    Launch required services in the background:

``` bash
    docker compose up -d
```

3. **Install Go Dependencies**

``` bash
    go get .
```

4. **Run the Application**

```bash
    go run .
```

5. **Verify Startup**

    On successful startup, expect output like:

```text
    [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached. 
    [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production. 
    - using env: export GIN_MODE=release 
    - - using code: gin.SetMode(gin.ReleaseMode) 
    [GIN-debug] GET /order/fetch --> main.fetchOrders (4 handlers) [GIN-debug] GET /order/:id --> main.getOrder (4 handlers) 
    [GIN-debug] PUT /order --> main.updateOrder (4 handlers) 
    [GIN-debug] GET /health --> main.main.func1 (4 handlers) [GIN-debug] [WARNING] 
    - You trusted all proxies, this is NOT safe. We recommend you to set a value. 
    - Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details. 
    [GIN-debug] Listening and serving HTTP on :3001
    
    The service is now running on port 3001.
```

----------

## 🔍 Testing the API

Use the provided test-makeline-service.http file to test endpoints:

1. Install [Visual Studio Code](https://code.visualstudio.com/).
2. Add the [REST Client Extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client).
3. Open test-makeline-service.http in VS Code.
4. Send HTTP requests to interact with the API.

----------

## 🗃️ Querying MongoDB

Verify orders in MongoDB using these steps:

1. **Connect to MongoDB**

```bash
    mongosh
```

2. **Switch to Database**

```bash
    show dbs use orderdb
```

3. **Query Orders**

``` bash
    show collections db.orders.find() # List all orders  
    db.orders.findOne({status: 1}) # View completed orders
```

----------

## 📎 Additional Notes

- **Environment Variables**: Verify all variables are set correctly before starting.
- **Troubleshooting**: Confirm connections to Azure Service Bus and MongoDB if issues arise.
