### Solution Overview
I propose a tiered storage solution that leverages Azure Blob Storage for cold data while maintaining hot data in Cosmos DB, with an intelligent data access layer to seamlessly handle requests for both hot and cold data.

#### Key Components:
Hot Storage: Azure Cosmos DB for active records (last 3 months)

Cold Storage: Azure Blob Storage for archived records (older than 3 months)

Data Movement Service: Azure Function to handle archival process

Query Router: API layer that transparently routes requests to appropriate storage

## Architecture Diagram
![image1](https://github.com/user-attachments/assets/2566419b-4ad7-4b75-8d18-c802e4d364ce)
![image2](https://github.com/user-attachments/assets/16b6d74d-2abc-468e-9cb7-128f631a4632)

### Implementation Details
## 1. Data Archival Process (Azure Function)

# PowerShell script to deploy the archival function
az functionapp create `
  --resource-group YourResourceGroup `
  --name YourArchivalFunction `
  --storage-account YourStorageAccount `
  --runtime dotnet `
  --functions-version 4 `
  --consumption-plan-location eastus

 ## // C# Azure Function for archiving data
[FunctionName("ArchiveOldRecords")]
public static async Task Run(
    [TimerTrigger("0 0 3 * * *")] TimerInfo myTimer, // Runs daily at 3 AM
    [CosmosDB(databaseName: "billing", collectionName: "records", 
        ConnectionStringSetting = "CosmosDBConnection")] DocumentClient client,
    [Blob("archived-records/{datetime}", FileAccess.Write, 
        Connection = "BlobStorageConnection")] CloudBlobContainer blobContainer,
    ILogger log)
{
    var threeMonthsAgo = DateTime.UtcNow.AddMonths(-3);
    
    var query = client.CreateDocumentQuery<BillingRecord>(
        UriFactory.CreateDocumentCollectionUri("billing", "records"),
        new FeedOptions { EnableCrossPartitionQuery = true })
        .Where(r => r.CreatedDate < threeMonthsAgo)
        .AsDocumentQuery();

    while (query.HasMoreResults)
    {
        var batch = await query.ExecuteNextAsync<BillingRecord>();
        
        foreach (var record in batch)
        {
            // Compress and store in blob
            var blobName = $"{record.CreatedDate:yyyy-MM}/{record.Id}.json.gz";
            var blob = blobContainer.GetBlockBlobReference(blobName);
            
            using (var ms = new MemoryStream())
            {
                using (var gzip = new GZipStream(ms, CompressionMode.Compress, true))
                using (var writer = new StreamWriter(gzip))
                {
                    await writer.WriteAsync(JsonConvert.SerializeObject(record));
                }
                
                ms.Position = 0;
                await blob.UploadFromStreamAsync(ms);
            }
            
            // Create reference document in Cosmos
            var referenceDoc = new {
                id = record.Id,
                archived = true,
                blobPath = blobName,
                originalData = null as object // Remove actual data
            };
            
            await client.ReplaceDocumentAsync(
                UriFactory.CreateDocumentUri("billing", "records", record.Id),
                referenceDoc);
        }
    }
}

## 2. Query Router Implementation
// Middleware component to handle routing
public class BillingRecordRouter
{
    private readonly DocumentClient _cosmosClient;
    private readonly CloudBlobContainer _blobContainer;
    
    public BillingRecordRouter(DocumentClient cosmosClient, CloudBlobContainer blobContainer)
    {
        _cosmosClient = cosmosClient;
        _blobContainer = blobContainer;
    }
    
    public async Task<BillingRecord> GetRecord(string recordId)
    {
        // First try to get from Cosmos
        try
        {
            var document = await _cosmosClient.ReadDocumentAsync<Document>(
                UriFactory.CreateDocumentUri("billing", "records", recordId));
            
            if (!document.Document.GetPropertyValue<bool>("archived"))
            {
                return document.Document.ToObject<BillingRecord>();
            }
            
            // If archived, fetch from blob
            var blobPath = document.Document.GetPropertyValue<string>("blobPath");
            var blob = _blobContainer.GetBlockBlobReference(blobPath);
            
            using (var ms = new MemoryStream())
            {
                await blob.DownloadToStreamAsync(ms);
                ms.Position = 0;
                
                using (var gzip = new GZipStream(ms, CompressionMode.Decompress))
                using (var reader = new StreamReader(gzip))
                {
                    return JsonConvert.DeserializeObject<BillingRecord>(
                        await reader.ReadToEndAsync());
                }
            }
        }
        catch (DocumentClientException e) when (e.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }
    }
}

### 3. Cost Optimization Analysis
### Before Optimization:

Cosmos DB storage cost for 2M records × 300KB ≈ 600GB

Estimated cost: $150/month for storage + RU costs

### After Optimization:

Assume 20% active data (400,000 records) in Cosmos DB: 120GB

80% (1.6M records) in Blob Storage (compressed to ~150KB each): 240GB

Estimated costs:

Cosmos DB: $30 storage + reduced RU costs

Blob Storage (Cool tier): ~$10/month

Total: ~$40/month (73% reduction)


### Implementation Steps
Setup Blob Storage Container

powershell
az storage container create --name archived-records --account-name YourStorageAccount
### Deploy Archival Function

Create function with timer trigger

Configure Cosmos DB and Blob Storage connections

### Update Data Access Layer

Implement query router middleware

No API contract changes needed

### Initial Data Migration

Run archival function manually first time to migrate existing old data

Verify data integrity

### Monitoring Setup

Add logging for archival process

Monitor access patterns to cold data
