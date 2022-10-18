---
layout: single
title:  "Configure Azure Functions Service Bus Trigger to use a Proxy"
date:   2022-02-09 00:30:00 +0000
permalink: /functions-service-bus-trigger-proxy
---

If you are developing Azure Functions in an enterprise environment, you may be in a situation where you have no direct internet access and must send all your traffic to a web proxy. When you create the [`ServiceBusClient`](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Azure.Messaging.ServiceBus/src/Client/ServiceBusClient.cs) yourself, you have the option of specifying the proxy using the [`ServiceBusClientOptions`](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Azure.Messaging.ServiceBus/src/Client/ServiceBusClientOptions.cs) class as demonstrated below.

```csharp
string connectionString = "";
string webProxyAddress = "http://127.0.0.1:3128";

ServiceBusClientOptions clientOptions = new ServiceBusClientOptions()
{
    TransportType = ServiceBusTransportType.AmqpWebSockets,
    WebProxy = new WebProxy(webProxyAddress)
};
ServiceBusClient serviceBusClient = new ServiceBusClient(connectionString, clientOptions);
```

However, when using an Azure Function, triggered by the Service Bus you do not have the option to provide `ServiceBusClientOptions`, and the [`ServiceBusTrigger`](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Microsoft.Azure.WebJobs.Extensions.ServiceBus/src/ServiceBusTriggerAttribute.cs) does not allow you to provide the proxy configuration either. 

```csharp
[FunctionName("ServiceBusTriggeredFunction")]
public void Run(
    [ServiceBusTrigger("myqueue", Connection = "ServiceBusConnection")]string myQueueItem,
    ILogger log)
{
    log.LogInformation($"Processed message: {myQueueItem}");
}
```

Luckily, the [Microsoft.Azure.WebJobs.Extensions.ServiceBus](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/servicebus/Microsoft.Azure.WebJobs.Extensions.ServiceBus) library, which implements the Service Bus trigger will obtain the `AzureFunctionsJobHost:extensions:ServiceBus` section of your Function's configuration and use it to configure the proxy (amongst other settings) of the underlying `ServiceBusClient` used by the trigger. The configuration binding is handled by [`ServiceBusHostBuilderExtensions`](https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/servicebus/Microsoft.Azure.WebJobs.Extensions.ServiceBus/src/Config/ServiceBusHostBuilderExtensions.cs#L44).

Therefore, we can make the `ServiceBusTrigger` use the proxy by setting the `AzureFunctionsJobHost:extensions:ServiceBus:WebProxy` property to the address and the port of your proxy in your `local.settings.json` file:

#### `local.settings.json`
```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet",
        "ServiceBusConnection": "<NAMESPACE CONNECTION STRING>",
        // PROXY
        "AzureFunctionsJobHost:extensions:ServiceBus:WebProxy": "http://127.0.0.1:3128",
        // USE PORT 443 INSTEAD OF 5671
        "AzureFunctionsJobHost:extensions:ServiceBus:TransportType": "AmqpWebSockets"
    }
}
```

Additionally, if your proxy does not allow you to use the AMPQS port (5671), you can configure the Service Bus client to use AMQP over Web Sockets (which uses port 443) by setting the `AzureFunctionsJobHost:extensions:ServiceBus:TransportType` property to `AmqpWebSockets`.