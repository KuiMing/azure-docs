---
title: Diagnose and troubleshoot issues when using Azure Cosmos DB .NET SDK
description: Use features like client-side logging and other third-party tools to identify, diagnose, and troubleshoot Azure Cosmos DB issues when using .NET SDK.
author: rothja
ms.service: cosmos-db
ms.date: 03/05/2021
ms.author: jroth
ms.subservice: cosmosdb-sql
ms.topic: troubleshooting
ms.reviewer: sngun
ms.custom: devx-track-dotnet
---
# Diagnose and troubleshoot issues when using Azure Cosmos DB .NET SDK
[!INCLUDE[appliesto-sql-api](../includes/appliesto-sql-api.md)]

> [!div class="op_single_selector"]
> * [Java SDK v4](troubleshoot-java-sdk-v4-sql.md)
> * [Async Java SDK v2](troubleshoot-java-async-sdk.md)
> * [.NET](troubleshoot-dot-net-sdk.md)
> 

This article covers common issues, workarounds, diagnostic steps, and tools when you use the [.NET SDK](sql-api-sdk-dotnet.md) with Azure Cosmos DB SQL API accounts.
The .NET SDK provides client-side logical representation to access the Azure Cosmos DB SQL API. This article describes tools and approaches to help you if you run into any issues.

## Checklist for troubleshooting issues

Consider the following checklist before you move your application to production. Using the checklist will prevent several common issues you might see. You can also quickly diagnose when an issue occurs:

*    Use the latest [SDK](sql-api-sdk-dotnet-standard.md). Preview SDKs should not be used for production. This will prevent hitting known issues that are already fixed.
*    Review the [performance tips](performance-tips.md), and follow the suggested practices. This will help prevent scaling, latency, and other performance issues.
*    Enable the SDK logging to help you troubleshoot an issue. Enabling the logging may affect performance so it's best to enable it only when troubleshooting issues. You can enable the following logs:
*    [Log metrics](../monitor-cosmos-db.md) by using the Azure portal. Portal metrics show the Azure Cosmos DB telemetry, which is helpful to determine if the issue corresponds to Azure Cosmos DB or if it's from the client side.
*    Log the [diagnostics string](/dotnet/api/microsoft.azure.documents.client.resourceresponsebase.requestdiagnosticsstring) in the V2 SDK or [diagnostics](/dotnet/api/microsoft.azure.cosmos.responsemessage.diagnostics) in V3 SDK from the point operation responses.
*    Log the [SQL Query Metrics](sql-api-query-metrics.md) from all the query responses 
*    Follow the setup for [SDK logging]( https://github.com/Azure/azure-cosmos-dotnet-v2/blob/master/docs/documentdb-sdk_capture_etl.md)

Take a look at the [Common issues and workarounds](#common-issues-workarounds) section in this article.

Check the [GitHub issues section](https://github.com/Azure/azure-cosmos-dotnet-v2/issues) that's actively monitored. Check to see if any similar issue with a workaround is already filed. If you didn't find a solution, then file a GitHub issue. You can open a support tick for urgent issues.


## <a name="common-issues-workarounds"></a>Common issues and workarounds

### General suggestions
* Run your app in the same Azure region as your Azure Cosmos DB account, whenever possible. 
* You may run into connectivity/availability issues due to lack of resources on your client machine. We recommend monitoring your CPU utilization on nodes running the Azure Cosmos DB client, and scaling up/out if they're running at high load.

### Check the portal metrics
Checking the [portal metrics](../monitor-cosmos-db.md) will help determine if it's a client-side issue or if there is an issue with the service. For example, if the metrics contain a high rate of rate-limited requests (HTTP status code 429) which means the request is getting throttled then check the [Request rate too large](troubleshoot-request-rate-too-large.md) section. 

## Retry Logic <a id="retry-logics"></a>
Cosmos DB SDK on any IO failure will attempt to retry the failed operation if retry in the SDK is feasible. Having a retry in place for any failure is a good practice but specifically handling/retrying write failures is a must. It's recommended to use the latest SDK as retry logic is continuously being improved.

1. Read and query IO failures will get retried by the SDK without surfacing them to the end user.
2. Writes (Create, Upsert, Replace, Delete) are "not" idempotent and hence SDK cannot always blindly retry the failed write operations. It is required that user's application logic to handle the failure and retry.
3. [Trouble shooting sdk availability](troubleshoot-sdk-availability.md) explains retries for multi-region Cosmos DB accounts.

### Retry design

The application should be designed to retry on any exception unless it is a known issue where retrying will not help. For example, the application should retry on 408 request timeouts, this timeout is possibly transient so a retry may result in success. The application should not retry on 400s, this typically means that there is an issue with the request that must first be resolved. Retrying on the 400 will not fix the issue and will result in the same failure if retried again. The table below shows known failures and which ones to retry on.

## Common error status codes <a id="error-codes"></a>

| Status Code | Retryable | Description | 
|----------|-------------|-------------|
| 400 | No | Bad request (i.e. invalid json, incorrect headers, incorrect partition key in header)| 
| 401 | No | [Not authorized](troubleshoot-unauthorized.md) | 
| 403 | No | [Forbidden](troubleshoot-forbidden.md) |
| 404 | No | [Resource is not found](troubleshoot-not-found.md) |
| 408 | Yes | [Request timed out](troubleshoot-dot-net-sdk-request-timeout.md) |
| 409 | No | Conflict failure is when the ID provided for a resource on a write operation has been taken by an existing resource. Use another ID for the resource to resolve this issue as ID must be unique within all documents with the same partition key value. |
| 410 | Yes | Gone exceptions (transient failure that should not violate SLA) |
| 412 | No | Precondition failure is where the operation specified an eTag that is different from the version available at the server. It's an optimistic concurrency error. Retry the request after reading the latest version of the resource and updating the eTag on the request.
| 413 | No | [Request Entity Too Large](../concepts-limits.md#per-item-limits) |
| 429 | Yes | It is safe to retry on a 429. This can be avoided by following the link for [too many requests](troubleshoot-request-rate-too-large.md).|
| 449 | Yes | Transient error that only occurs on write operations, and is safe to retry. This can point to a design issue where too many concurrent operations are trying to update the same object in Cosmos DB. |
| 500 | Yes | The operation failed due to an unexpected service error. Contact support by filing an [Azure support issue](https://aka.ms/azure-support). |
| 503 | Yes | [Service unavailable](troubleshoot-service-unavailable.md) | 

### <a name="snat"></a>Azure SNAT (PAT) port exhaustion

If your app is deployed on [Azure Virtual Machines without a public IP address](../../load-balancer/load-balancer-outbound-connections.md), by default [Azure SNAT ports](../../load-balancer/load-balancer-outbound-connections.md#preallocatedports) establish connections to any endpoint outside of your VM. The number of connections allowed from the VM to the Azure Cosmos DB endpoint is limited by the [Azure SNAT configuration](../../load-balancer/load-balancer-outbound-connections.md#preallocatedports). This situation can lead to connection throttling, connection closure, or the above mentioned [Request timeouts](troubleshoot-dot-net-sdk-request-timeout.md).

 Azure SNAT ports are used only when your VM has a private IP address is connecting to a public IP address. There are two workarounds to avoid Azure SNAT limitation (provided you already are using a single client instance across the entire application):

* Add your Azure Cosmos DB service endpoint to the subnet of your Azure Virtual Machines virtual network. For more information, see [Azure Virtual Network service endpoints](../../virtual-network/virtual-network-service-endpoints-overview.md). 

    When the service endpoint is enabled, the requests are no longer sent from a public IP to Azure Cosmos DB. Instead, the virtual network and subnet identity are sent. This change might result in firewall drops if only public IPs are allowed. If you use a firewall, when you enable the service endpoint, add a subnet to the firewall by using [Virtual Network ACLs](/previous-versions/azure/virtual-network/virtual-networks-acl).
* Assign a [public IP to your Azure VM](../../load-balancer/troubleshoot-outbound-connection.md#assignilpip).

### <a name="high-network-latency"></a>High network latency
High network latency can be identified by using the [diagnostics string](/dotnet/api/microsoft.azure.documents.client.resourceresponsebase.requestdiagnosticsstring) in the V2 SDK or [diagnostics](/dotnet/api/microsoft.azure.cosmos.responsemessage.diagnostics#Microsoft_Azure_Cosmos_ResponseMessage_Diagnostics) in V3 SDK.

If no [timeouts](troubleshoot-dot-net-sdk-request-timeout.md) are present and the diagnostics show single requests where the high latency is evident.

# [V3 SDK](#tab/diagnostics-v3)

Diagnostics can be obtained from any `ResponseMessage`, `ItemResponse`, `FeedResponse`, or `CosmosException` by the `Diagnostics` property:

```csharp
ItemResponse<MyItem> response = await container.CreateItemAsync<MyItem>(item);
Console.WriteLine(response.Diagnostics.ToString());
```

Network interactions in the diagnostics will be for example:

```json
{
    "name": "Microsoft.Azure.Documents.ServerStoreModel Transport Request",
    "id": "0e026cca-15d3-4cf6-bb07-48be02e1e82e",
    "component": "Transport",
    "start time": "12: 58: 20: 032",
    "duration in milliseconds": 1638.5957
}
```

Where the `duration in milliseconds` would show the latency.

# [V2 SDK](#tab/diagnostics-v2)

The diagnostics are available when the client is configured in [direct mode](sql-sdk-connection-modes.md), through the `RequestDiagnosticsString` property:

```csharp
ResourceResponse<Document> response = await client.ReadDocumentAsync(documentLink, new RequestOptions() { PartitionKey = new PartitionKey(partitionKey) });
Console.WriteLine(response.RequestDiagnosticsString);
```

And the latency would be on the difference between `ResponseTime` and `RequestStartTime`:

```bash
RequestStartTime: 2020-03-09T22:44:49.5373624Z, RequestEndTime: 2020-03-09T22:44:49.9279906Z,  Number of regions attempted:1
ResponseTime: 2020-03-09T22:44:49.9279906Z, StoreResult: StorePhysicalAddress: rntbd://..., ...
```
--- 

This latency can have multiple causes:

* Your application is not running in the same region as your Azure Cosmos DB account.
* Your [PreferredLocations](/dotnet/api/microsoft.azure.documents.client.connectionpolicy.preferredlocations) or [ApplicationRegion](/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions.applicationregion) configuration is incorrect and is trying to connect to a different region to where your application is currently running on.
* There might be a bottleneck on the Network interface because of high traffic. If the application is running on Azure Virtual Machines, there are possible workarounds:
    * Consider using a [Virtual Machine with Accelerated Networking enabled](../../virtual-network/create-vm-accelerated-networking-powershell.md).
    * Enable [Accelerated Networking on an existing Virtual Machine](../../virtual-network/create-vm-accelerated-networking-powershell.md#enable-accelerated-networking-on-existing-vms).
    * Consider using a [higher end Virtual Machine](../../virtual-machines/sizes.md).

### Common query issues

The [query metrics](sql-api-query-metrics.md) will help determine where the query is spending most of the time. From the query metrics, you can see how much of it is being spent on the back-end vs the client. Learn more about [troubleshooting query performance](troubleshoot-query-performance.md).

* If the back-end query returns quickly, and spends a large time on the client check the load on the machine. It's likely that there are not enough resource and the SDK is waiting for resources to be available to handle the response.
* If the back-end query is slow, try [optimizing the query](troubleshoot-query-performance.md) and looking at the current [indexing policy](../index-overview.md)

    > [!NOTE]
    > For improved performance, we recommend Windows 64-bit host processing. The SQL SDK includes a native ServiceInterop.dll to parse and optimize queries locally. ServiceInterop.dll is supported only on the Windows x64 platform. For Linux and other unsupported platforms where ServiceInterop.dll isn't available, an additional network call will be made to the gateway to get the optimized query.

If you encounter the following error: `Unable to load DLL 'Microsoft.Azure.Cosmos.ServiceInterop.dll' or one of its dependencies:` and are using Windows, you should upgrade to the latest Windows version.

## Next steps

* Learn about Performance guidelines for [.NET V3](performance-tips-dotnet-sdk-v3-sql.md) and [.NET V2](performance-tips.md)
* Learn about the [Reactor-based Java SDKs](https://github.com/Azure-Samples/azure-cosmos-java-sql-api-samples/blob/main/reactor-pattern-guide.md)

 <!--Anchors-->
[Common issues and workarounds]: #common-issues-workarounds
[Enable client SDK logging]: #logging
[Azure SNAT (PAT) port exhaustion]: #snat
[Production check list]: #production-check-list
