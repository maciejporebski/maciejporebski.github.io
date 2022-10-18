---
layout: single
title:  "Common Azure AD First-Party Token Scopes"
date:   2022-04-20 17:26:00 +0000
permalink: /common-azure-ad-first-party-token-scopes
---

When obtaining a token from the v2 token endpoint you must provide a list of scopes rather than just the resource. Resources often expose a `/.default` scope (e.g. `https://graph.azure.com/.default`, however, a more specific scope is often a good idea if available.
{: .notice--warning}


| Resource                                                                                                                                  | Identifier URI                             |
| ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Azure Management                                                                                                                          | `https://management.azure.com`             |
| Azure DevOps                                                                                                                              | `499b84ac-1321-427f-aa17-267ca6975798`     |
| [Microsoft Graph](https://docs.microsoft.com/en-us/graph/permissions-reference)                                                           | `https://graph.azure.com`                  |
| Azure Key Vault                                                                                                                           | `https://vault.azure.net`                  |
| Azure Service Bus                                                                                                                         | `https://servicebus.azure.net`             |
| Azure Service Bus (resource specific)                                                                                                     | `https://<namespace>.servicebus.azure.net` |
| Azure SQL                                                                                                                                 | `https://database.windows.net`             |
| Azure Storage                                                                                                                             | `https://storage.azure.com`                |
| [Azure Storage (blob)](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad-app?tabs=dotnet#azure-storage-resource-id)  | `https://<account>.blob.core.windows.net`  |
| [Azure Storage (file)](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad-app?tabs=dotnet#azure-storage-resource-id)  | `https://<account>.file.core.windows.net`  |
| [Azure Storage (queue)](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad-app?tabs=dotnet#azure-storage-resource-id) | `https://<account>.queue.core.windows.net` |
| [Azure Storage (table)](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad-app?tabs=dotnet#azure-storage-resource-id) | `https://<account>.table.core.windows.net` |
| AD Graph (deprecated)                                                                                                                     | `https://graph.windows.net`                |
