>**Note:** SharePoint webhooks is currently in preview and is subject to change. SharePoint webhooks are not currently supported for use in production environments.

# Overview
SharePoint [webhooks](http://en.wikipedia.org/wiki/Webhook) allow developers to build service integrations which subscribe to receive notifications on specific events that occur in SharePoint. When one of those events are triggered, SharePoint will send a HTTP POST payload to the subscriber. Webhooks are easier to develop and consume since they're regular HTTP services (web API) in comparison to WCF services used by SharePoint add-in remote event receivers.

The first iteration will deliver webhooks for [SharePoint list items](./lists/overview-sharepoint-list-webhooks). The SharePoint list item webhooks cover the events corresponding to list item changes for a given SharePoint list or a document library. SharePoint webhooks provide a simple notification pipeline so your application can be aware of changes to a SharePoint list without polling the service. 

## Creating webhooks
To create a new SharePoint webhook, you add a new subscription to the specific SharePoint resource, such as a SharePoint list. 

The following information is required for creating a new subscription:

- Resource
  - The resource endpoint URL you are creating the subscription for. For example, SharePoint List API URL.
- Server Notification URL
  - This is your service endpoint URL. SharePoint will send a HTTP POST to this endpoint when events in the specified resource.
- Expiration Date Time
  - Expiration date time for your subscription. The expiration date time should not be more than 6 months. By default, subscriptions are set to expire 6 months from when they are created. 

You can also include the following additional information if needed:

- Client State  
  - Opaque string passed back to the client on all notifications. You can choose to use this for various reasons, such as, but not limited to:
     - validate notifications
     - tag different subscriptions

## Handling webhook validation requests

When a new subscription is created, SharePoint will validate whether the notification URL supports receiving webhook notifications. This validation is performed during the subscription creation request. The subscription will only be created if your service responds in a timely manner back with the validation token.

When a new subscription is created, SharePoint will POST a request to the registered URL in the following format:

### Example validation request

```http
POST https://contoso.azurewebsites.net/your/webhook/service?validationToken={randomString}
Content-Length: 0
```

### Response

For the subscription to be created successfully, your service must respond to this request by returning the value of the **validationToken** query string parameter as a plain-text response.

```http
HTTP/1.1 200 OK
Content-Type: text/plain

{randomString}
```

If your application returns a status code other than `200` or fails to respond with the value of the validationToken parameter, the request to create a new subscription will fail.

## Receiving notifications
The notification payload will inform the service that an event occurred in a given resource for a given subscription. Multiple notifications to your service may be batched together into a single request, if multiple changes occurred in the resource within the same time period.

The notification payload body will also contain your client state if you used it when creating the subscription. 

### Webhook notification resource

The notification resource defines the shape of the data provided to your service when a SharePoint webhook notification request is submitted to your registered notification URL.

#### JSON representation

Each notification generated by the service is serialized into a **webhookNotifiation** instance:

```json
{
    "subscriptionId":"91779246-afe9-4525-b122-6c199ae89211",
    "clientState":"00000000-0000-0000-0000-000000000000",
    "expirationDateTime":"2016-04-30T17:27:00.0000000Z",
    "resource":"b9f6f714-9df8-470b-b22e-653855e1c181",
    "tenantId":"00000000-0000-0000-0000-000000000000",
    "siteUrl":"/",
    "webId":"dbc5a806-e4d4-46e5-951c-6344d70b62fa"
}
```

Since multiple notifications may be submitted to your service in a single request, these are combined together in an object with a single array **value**:

```json
{
   "value":[
      {
         "subscriptionId":"91779246-afe9-4525-b122-6c199ae89211",
         "clientState":"00000000-0000-0000-0000-000000000000",
         "expirationDateTime":"2016-04-30T17:27:00.0000000Z",
         "resource":"b9f6f714-9df8-470b-b22e-653855e1c181",
         "tenantId":"00000000-0000-0000-0000-000000000000",
         "siteUrl":"/",
         "webId":"dbc5a806-e4d4-46e5-951c-6344d70b62fa"
      }
   ]
}
```

#### Properties

| Property Name          | Type              | description                                                                                                                         |
|:-----------------------|:------------------|:------------------------------------------------------------------------------------------------------------------------------------|
| **resource**           | String            | Unique identifier of the list where the subscription is registered.                                                                 |
| **subscriptionId**     | String            | The unique identifier for the subscription resource                                                                                 |
| **clientState**        | String - optional | An optional string value that is passed back in the notification message for this subscription.                                     |
| **expirationDateTime** | DateTime          | The date and time when the subscription will expire if not updated or renewed.                                                      |
| **tenantId**           | String            | Unique identifier for the tenant which generated this notification.                                                                 |
| **siteUrl**            | String            | Server relative URL of the site where the subscription is registered.                                                               |
| **webId**              | String            | Unique identifier of the web where the subscription is registered.                                                                  |

#### Example notification
Below is an example payload notification:

The body of the HTTP request to your service notification URL will contain a [Webhook Notification](./lists/webhook-notification) similar to the following:

```json
{
   "value":[
      {
         "subscriptionId":"91779246-afe9-4525-b122-6c199ae89211",
         "clientState":"00000000-0000-0000-0000-000000000000",
         "expirationDateTime":"2016-04-30T17:27:00.0000000Z",
         "resource":"b9f6f714-9df8-470b-b22e-653855e1c181",
         "tenantId":"00000000-0000-0000-0000-000000000000",
         "siteUrl":"/",
         "webId":"dbc5a806-e4d4-46e5-951c-6344d70b62fa"
      }
   ]
}
```

You'll notice that the notification doesn't include any information about the changes that triggered it. Your application is expected to use the [GetChanges API](https://msdn.microsoft.com/EN-US/library/office/dn531433.aspx#bk_ListGetChanges) on the list to query the collection of changes from the change log and store the change token value for the next subsequent call when you are notified.

## Event types
SharePoint webhooks support only asynchronous events. This means, webhooks are only fired after a change happened (so similar to **-ed** events), and thus synchronous (**-ing** events) are not possible.

## Error handling
If an error occurs while sending the notification to your service, SharePoint will follow exponential back-off logic. Any response with an HTTP status code outside of the 200-299 range, or that times out, will be attempted again over the next several minutes. If the request is not successful after 15 minutes, the notification is dropped.

Future notifications will still be attempted to your service, although the service reserves the right to remove the subscription if a sufficient number of failures are detected.

## Expiration
Webhook subscriptions are set to expire 6 months by default or at the specified date time from when they are created. 

You need to set an expiration date time when creating the subscription. The expiration date time should not be more than 6 months. Your application is expected to handle the expiration date time according to your application's needs by updating the subscription periodically. 

## Retry mechanism

When a given event occurs in SharePoint and when your service endpoint is not reachable (e.g. due to maintenance) SharePoint will retry sending the notification. Today SharePoint performs a retry of **5 times with a 5 minute wait time** between the attempts. If for some reason your service is down for a longer time, the next time when it's up and gets called by SharePoint the call to `GetChanges()` will get you the changes that were missed when your service was not available.