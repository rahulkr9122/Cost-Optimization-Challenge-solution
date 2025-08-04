Name : Rahul Kumar Prasad
Exp : 3 year
phone : 6203027876



# Cost-Optimization-Challenge-solution
Cost Optimization Challenge: Managing Billing Records in Azure Serverless Architecture

Prompt:

We use Azure serverless architecture and store billing records in Cosmos DB. It's read-heavy, with records older than 3 months rarely accessed. Data size and costs are growing. I need a cost optimization solution that:

Has no API changes

Has no data loss or downtime

Is simple to implement

Keeps old data accessible within a few seconds
Please suggest a detailed, practical architecture with Azure-native tools.







solution : 

Proposed Architecture: Tiered Storage with Archival Strategy
1. Active Tier (Hot Data) – Azure Cosmos DB
Keep only the last 3 months of billing records in Cosmos DB.

This will drastically reduce RU/s usage and storage costs.

Cosmos DB continues to serve as the backend for all API operations.

2. Cold Tier – Azure Blob Storage (Hot/Cool Tier)
Migrate older records (older than 3 months) to Azure Blob Storage as compressed JSON or Avro files.

Use a structured folder hierarchy: e.g., billing-records/yyyy/MM/.

--------------------------------x---------------------------------------



How It Works
✅ Read Path (Unified Access)
Implement a middleware logic layer (e.g., Azure Function or API Management policy):

Check Cosmos DB first (hot tier).

If the record is not found (404), fallback to Blob Storage.

Blob access latency is in the order of seconds—acceptable per your requirements.

✅ Write Path
Continue writing all new records to Cosmos DB.

Use Time-to-Live (TTL) to automatically expire documents older than 3 months after archival.


-------------------------------x-------------------------------------------------

Background Archival Process :
Implement an Azure Data Factory (ADF) pipeline or Azure Durable Function that:
Runs daily or weekly.

Scans Cosmos DB for records older than 3 months.

Copies those records to Azure Blob Storage in batches.

Verifies successful copy before deleting them from Cosmos DB.

💡 For large record size (300 KB), store in compressed format (e.g., GZIP) to minimize storage cost in Blob.

----------------------------------x------------------------------------------------


📦 Optional Enhancements
Blob Index Tags or Metadata

Allow fast lookup using record IDs or timestamps.

Blob Cache

-> Cache recently accessed cold records in-memory or Redis for quicker second access.

Partitioned Cosmos DB

-> Ensure partitioning by date (e.g., month or timestamp) to optimize range queries and TTL.

Monitor with Azure Monitor / Log Analytics

-> Track archival success, errors, and cold-record access latency.

---------------------------------x---------------------------------------------------


Tools Stack Summary
Service	Purpose
Azure Cosmos DB	Hot storage for recent records
Azure Blob Storage	Cost-efficient cold storage
Azure Functions / APIM	Unified access routing logic
Azure Data Factory	Archival pipeline
Azure Monitor	Observability and alerting


-----------------------------------X-------------------------------------------------

Final Result
You get a cost-optimized, tiered storage system that:
Meets your SLAs for latency.
Is fully backward-compatible.
Avoids vendor lock-in.
Leverages serverless capabilities and Azure-native tools.

----------------------------------------------x---------------------------------------------



HANDSON CODE IN C#


C# Azure Function acting as a middleware API that:
First attempts to read from Cosmos DB (hot tier).
If not found (404), falls back to Blob Storage (cold tier).
Assumes records are stored in Blob as individual JSON files using record ID.




using System.IO;
using System.Net;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Cosmos;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Specialized;
using Newtonsoft.Json;

public static class BillingRecordMiddleware
{
    private static readonly string cosmosConnectionString = Environment.GetEnvironmentVariable("CosmosDBConnection");
    private static readonly string databaseId = "BillingDB";
    private static readonly string containerId = "Records";
    private static readonly string blobConnectionString = Environment.GetEnvironmentVariable("BlobStorageConnection");
    private static readonly string blobContainerName = "billing-archive";

    [FunctionName("GetBillingRecord")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "billing/{recordId}")] HttpRequest req,
        string recordId,
        ILogger log)
    {
        var cosmosClient = new CosmosClient(cosmosConnectionString);
        var container = cosmosClient.GetContainer(databaseId, containerId);

        try
        {
            // Try from Cosmos DB first
            ItemResponse<dynamic> response = await container.ReadItemAsync<dynamic>(recordId, new PartitionKey(recordId));
            return new OkObjectResult(response.Resource);
        }
        catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            // Fallback to Blob Storage
            var blobClient = new BlobClient(blobConnectionString, blobContainerName, $"{recordId}.json");

            if (await blobClient.ExistsAsync())
            {
                var download = await blobClient.DownloadAsync();
                using var reader = new StreamReader(download.Value.Content);
                var content = await reader.ReadToEndAsync();
                var obj = JsonConvert.DeserializeObject(content);
                return new OkObjectResult(obj);
            }
            else
            {
                return new NotFoundObjectResult($"Record {recordId} not found in both Cosmos DB and Blob Storage.");
            }
        }
    }
}




