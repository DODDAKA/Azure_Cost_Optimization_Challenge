### Solution Overview
I propose a tiered storage solution that leverages Azure Blob Storage for cold data while maintaining hot data in Cosmos DB, with an intelligent data access layer to seamlessly handle requests for both hot and cold data.

#### Key Components:
Hot Storage: Azure Cosmos DB for active records (last 3 months)

Cold Storage: Azure Blob Storage for archived records (older than 3 months)

Data Movement Service: Azure Function to handle archival process

Query Router: API layer that transparently routes requests to appropriate storage
