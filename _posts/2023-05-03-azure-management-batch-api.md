---
layout: single
title:  "Use the Azure Management Batch API to Send Multiple Requests to management.azure.com"
date:   2023-05-04 20:00:00 +0000
permalink: /azure-management-batch-api
---

This endpoint is not documented by Microsoft, and therefore, likely subject to unannounced breaking changes, or withdrawal. It is not recommended to rely on this endpoint for any Production implementations.
{: .notice--warning}

If you would like to send several requests to the Azure Management API (management.azure.com), rather than managing queueing these requests yourself you can utilise the batch endpoint https://management.azure.com/batch (utilised by the Azure Portal) to send up to 500 requests (20 for requests with a body) in a single HTTP request and let Azure handle the requests for you.

# Request

```
POST https://management.azure.com/batch?api-version=2022-12-01
```

## Headers

| Header        | Value            | Description                                   |
| ------------- | ---------------- | --------------------------------------------- |
| Content-Type  | application/json |                                               |
| Authorization | Bearer ey...     | Bearer token scoped to `management.azure.com` |

## Body

| Name     | Type                                 | Description                                    |
| -------- | ------------------------------------ | ---------------------------------------------- |
| requests | [Request Object []](#request-object) | List of requests, allowing up to 500 requests (no body) or 20 requests (with body). |

### Request Object

| Name                 | Type                      | Description                                                                                                                                                                 |
| -------------------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| httpMethod           | string                    | HTTP request method                                                                                                                                                         |
| name                 | guid                      | GUID uniquely idenfiying the request. This value will be returned in the response to allow you to match the requests to responses. Required if submitting over 20 requests. |
| relativeUrl          | string                    | URL path of the request. Either the `relativeUrl` or `url` property must be provided for each request.                                                                      |
| url                  | string                    | Full url of the request. Either the `relativeUrl` or `url` property must be provided for each request.                                                                      |
| requestHeaderDetails | Dictionary<string,string> | HTTP headers for the request. Optional.                                                                                                                                     |
| content              | object                    | HTTP request body for the request. Optional.                                                                                                                                |

```json
{
    "requests": [
        {
            "httpMethod": "GET",
            "name": "0001b6e6-c00b-4e69-85b2-7a4fe86e9f31",
            "relativeUrl": "/subscriptions?api-version=2022-01-01",
            "url": "",
            "requestHeaderDetails": {},
            "content": {}
        },
        {
            ...
        }
    ]
}
```

# Response

- If over 20 requests are included, the endpoint will respond with `202 Accepted` status and a `Retry-After` and `Location` headers to allow you to poll for the results.
- If over 100 requests are included, the response will include a `nextLink` property with the URL which includes a skip token to fetch the next page of results.

## 200 OK

```json
{
    "value": [
        {
            "name": "0001b6e6-c00b-4e69-85b2-7a4fe86e9f31",
            "httpStatusCode": 200,
            "headers": {},
            "content": {},
            "contentLength": 2168
        }
    ],
    "nextLink": "https://management.azure.com/batch/ey...?api-version=2022-12-01&%24skiptoken=ey..."
}
```

## 202 Accepted

If the request contains over 20 requests, the `202 Accepted` response will be returned immediately, with a `Location` URL header. The URL can be polled at any time and the response body will include partial responses, for requests that have finished processing. As long as some requests are still pending the results endpoint will continue to return the status of `202 Accepted`. Once all requests have finished processing, the endpoint will start responding with `200 OK` instead.

| Header      | Value                                                           |
| ----------- | --------------------------------------------------------------- |
| Retry-After | 5                                                               |
| Location    | https://management.azure.com/batch/ey...?api-version=2022-12-01 |
