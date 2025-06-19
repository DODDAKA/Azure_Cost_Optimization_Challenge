### Solution Overview
I propose a tiered storage solution that leverages Azure Blob Storage for cold data while maintaining hot data in Cosmos DB, with an intelligent data access layer to seamlessly handle requests for both hot and cold data.

#### Key Components:
Hot Storage: Azure Cosmos DB for active records (last 3 months)

Cold Storage: Azure Blob Storage for archived records (older than 3 months)

Data Movement Service: Azure Function to handle archival process

Query Router: API layer that transparently routes requests to appropriate storage

┌─────────────────────────────────────────────────────────────────────────────┐
│                                Client Applications                           │
└───────────────────────┬───────────────────────────────────┬─────────────────┘
                        │                                   │
                        ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Layer (Existing)                           │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                            Query Router                              │  │
│  │                                                                      │  │
│  │  • Intercepts all read requests                                      │  │
│  │  • Checks record timestamp                                           │  │
│  │  • Routes to Cosmos DB (hot) or Blob Storage (cold)                  │  │
│  │  • Returns unified response format                                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┬─────────────────┘
                        │                                   │
                        ▼                                   ▼
┌───────────────────────┴───────┐               ┌───────────┴───────────────────┐
│       Hot Storage             │               │        Cold Storage           │
│  Azure Cosmos DB              │               │  Azure Blob Storage          │
│  • Current + 3 months data    │               │  • Compressed JSON files     │
│  • Optimized for frequent     │               │  • Organized by date ranges  │
│    access                     │               │  • Lower cost storage        │
└───────────────────────────────┘               └──────────────────────────────┘
                        ▲                                   ▲
                        │                                   │
                        └───────────┬───────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────────┐
                    │        Data Archival Service      │
                    │                                   │
                    │  • Timer-triggered Azure Function│
                    │  • Moves data >3 months to blob  │
                    │  • Maintains index in Cosmos DB   │
                    │  • Compresses data before storage │
                    └───────────────────────────────────┘
