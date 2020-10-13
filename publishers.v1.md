
# Building a Platform for Transactional Data Integrations

A distributed platform built around a set of micro-services that are loosely coupled. A loosely coupled architecture is composed of services that can stand independently and are resilient to changes in the behavior of components with which they collaborate. In other words, changes to one service should have no impact on other services. Communications between services are conducted using asynchronous message queues.

Use RabbitMQ to manage and operate these queues, with MassTransit running on top of that for distributed application development services. MassTransit enables a loose coupling between services, including publishers, using a message based approach and adds concurrency, connection management, exception handling and retries, serialization of .Net classes, message correlation, message consumer lifecycle management, message routing, saga and routing slip patterns, scheduling and more.

<https://masstransit-project.com/>

<https://github.com/MassTransit/MassTransit>


## Publishing Process

Jobs are accepted for publishing, publishers are selected at runtime according to configuration settings. A publisher can run from anywhere so long as it has communication with RabbitMQ servers.

Publishing of data is built using a [Routing Slip pattern](https://masstransit-project.com/MassTransit/advanced/courier/). This allows jobs to be dynamically routed to required components and for each component to provide data to the next component in the route. Many (most) publish jobs only need to do two steps:

1. Fetch any existing data from the target system
2. Compare any existing data and if there are any changes then write them to the the target system.

There can be other steps as well including synchronisation and snapshotting, deduplication or transactionally (ACID properties) publishing data to multiple target system.

The routing slip pattern enables dynamical configuration of how a job is to be processed, at run time per request, and also allows the entire job to be transactional.

Use MassTransit to manage this, so a new publisher needs to be configured to work with MassTransit. This document describes how to do that and how to use the Courier API with MassTransit.

Because MassTransit is intrinsically asynchronous at any one time there will be multiple instances of your publisher processing publish jobs concurrently. All code should be written as asynchronous code.

By default the number of publishing processes will be a maximum of 4 times the processor core count, but can be configured to be higher. Because of this care should be taken to ensure correct management of machine resources especially if using HttpClient. It can often be the case that target system SDKs assume a single process and do not effectively manage HttpClient for our scenario (i.e. you cannot inject HttpClient into the SDK). PowerOffice SDK is a good example of this, and if used it will create a new HttpClient for every publish job processed. When publishing hundreds or even thousands of PowerOffice objects and we create a new HttpClient each time for each object and given that sockets are held in the TIME_WAIT state for several minutes after a connection is closed we can quickly reach a situation where we are in danger of exhausting available sockets.

*HttpClients must be managed correctly so a single client is reused across all instances of the publisher*. The skeleton project provides an example of this.

## Publisher Rules and Development

Publishers should not do any transformation of the payload data. If the payload data is not in the correct form this is an issue for data transformation services. Publishers take what they are given and attempt to publish that. If there is an error of validation or similar the publisher will report that error and fail.

The publisher can assume that the data it is given in the payload is completely ready and correct for the target API to accept and if it that data is rejected report as an error.

Publishers should be completely agnostic of any integration product. Never introduce code into a publisher that solves a problem or rule for an integration product.

A publisher must never write/update data if there are no changes to data in the target system. This is to prevent continuous update loops between two seperate systems. So a query is always performed before a write and the result is compared to the job to see if any update will be required.

Publishers can be of two types:

1) for publishing batches to a batch API (e.g. a file upload, SQL Server BulkCopy, sending an email attachment)
2) publishing single objects via an API

Both types of publisher are implemented in a similar way, the main difference being that batches will not be sent in the Payload object but will have a storage reference for the publisher to retrieve the batch from external storage. This mechanism is handled for us by MassTransit. This document focuses on the 2nd type of publisher.

All services should be loosely coupled (see above) and this includes publishers.

* Publishers must contain all their dependencies and must not rely on any other publisher or other service for any code or dll.
* Neither is it permitted for a publisher share code with another publisher. If you want to reuse code that is fine, just take a copy of it and to your publisher and avoid making a cross publisher dependency. Changes to one publisher should never impact any other service or publisher.

If a library (Nuget, etc.) is used it must be open source and without commercial restriction (Apache/MIT license for example).

## Library Standards

There are not many:

* IoC is with Autofac
* .Net Core 3+
* Developed as Linux Containers
* Using Microsoft.Extensions.Hosting.BackgroundService
* MassTransit
* Polly for adding resilience and API fault handling (in fact there are two levels of retry, with Polly and with MassTransit)
* Logging to three sinks (console, syslog and Azure Analytics). Any logging can be used so long as these sinks are available.

### Publisher Naming

The name of the publisher is key. The name is used in the publish context of the job to identify which publisher the job should be routed to and the name is used for Azure Service Bus queues for your publisher. The name should be clear and obvious as possible, it has to be unique.

### Publisher Message Queues on Azure Service Bus

Message queues have a standard naming:

* {PublisherName}Get
* {PublisherName}Write
* {PublisherName}Compensate

The publisher name plus standardised queue names enables dynamic routing at run time using MassTransit Courier (routing slip).

## Publisher Functions

Each publisher has to be able to do three basic things.

1. make a query to see if that data exists in the target system, and return it, or a fault.
2. write data to the target system or return a fault.
3. compensate data from the target (undo a write).

Each operation will always be for a single object. If the target system has no query this function is still present but returns null values. Each write will always be for a single object and each compensation will be for a single object.

An object (properties only, no behavior) is provided to the publisher as JSON (JToken, [Newtonsoft JSON.net](https://www.newtonsoft.com/json)). This is used to perform the requested activity (Get, Write, Compensate).

Each activity has access to the following data:

* Guid StageId
* Guid DocumentId
* string JobId
* JToken Payload
* string ObjectType
* JToken PublishContext

The first three items are mostly used if errors need to be reported. **Payload** is the object to be processed. **ObjectType** is a string which will be a name that the publisher 'understands' that describes the type of data to be published (an order, a product, customer, etc). Typically this will match the API, or documentation, etc. of the target system (including case). Finally the PublishContext.

## Payload

This can be anything, but will always be in JSON represented as a JToken. An example from the PowerOffice publisher here is an [OutgoingInvoice](https://api.poweroffice.net/Web/docs/index.html#Reference/Rest/Type_OutgoingInvoice.md) object.

```json
{
  "importedOrderNo": 8265199,
  "externalImportReference": "shopname:8265199",
  "orderDate": "2019-06-06 07:40:00",
  "currencyCode": "NOK",
  "customerReference": "shopname_8265199",
  "outgoingInvoiceLines": [
    {
      "quantity": 1.0,
      "unitPrice": 1612.08,
      "description": "Rab Microlight Alpine Blå Dunjakke",
      "productCode": "535311"
    },
    {
      "quantity": 1.0,
      "unitPrice": 76.8,
      "description": "Posten Servicepakke ():",
      "productCode": "CF01"
    }
  ],
  "customerCode": 10000,
  "deliveryAddressId": 4102618
}
```

The payload does not include any extra data, just the object ready to be published in form the target API expects. So the publisher does not do any extra transformational work on the payload.

## Publish Context

This is data, options, etc. specifically to enable the publisher to communicate with the target system. The publish context is specifically for this publisher and the publisher can expect required properties to be present (or throw a fault). Connection strings, user credentials, server URIs, information about objects and options about how to handle publish requests (i.e demo or live environments) will all be presented in the publish context.

The publish context for PowerOffice is simple and looks like this:

``` JSON
{
    "powerOffice" : {
        "clientKey" : "55eb570d-9d4b-424d-ad41-ab21efbfd699",
        "applicationKey" : "c66c5b2d-5ebd-4602-a353-6e31dd8b7917",
        "productionEnv" : false
    }
}
```

Others can be more complex. You can design the publish context how it is needed by the publisher. For example, SQLServerBatch publish context looks like this, including some information about table names and nested arrays that occur as object properties:

```JSON
{
    "sqlServer" : {
        "server" : "11.11.0.20,1433",
        "database" : "247Office_Tests",
        "user" : "arcee",
        "password" : "66NTc_46ttXSPe4T",
        "connectionString" : "",
        "tableNameMappings" : {
            "Projects" : "Projects",
            "Products" : "Products",
            "Transactions" : "Transactions",
            "Companies" : "Companies"
        },
        "nestedObjectIds" : [
            {
                "Dimensions" : {
                    "idProperty" : "Id",
                    "childPrefix" : "transaction."
                }
            }
        ]
    }
}
```

## Visual Studio Solution

Will contain two projects:

1) Publisher Contracts
2) Publisher Worker Service

### Contracts Project

A simple class library containing three files to define the contracts as interfaces. They will always look the same for every publisher. The contract is used by the publishing system to send data to your publisher. Your publish service listens for messages sent to your service URI (an Azure Service Bus URI for your publishers message queue defined by your publisher name).

#### Get.Activity.Arguments.cs

```c#
using Newtonsoft.Json.Linq;
using System;

namespace PowerOffice.Worker.Contracts
{
    public interface IGetActivityArguments
    {
        string JobId { get; }
        Guid StageId { get; }
        Guid DocumentId { get; }
        string ObjectType { get; }
        JToken Payload { get; }
        JToken PublishContext { get; }
    }
}
```

Note the private setter.

#### Write.Activity.Arguments.cs

```c#
using Newtonsoft.Json.Linq;
using System;

namespace PowerOffice.Worker.Contracts
{
    public interface IWriteActivityArguments
    {
        string JobId { get; }
        Guid StageId { get; }
        Guid DocumentId { get; }
        string ObjectType { get; }
        JObject Payload { get; }
        JObject PublishContext { get; }
        JObject ExistingObject { get; }
    }
}
```

#### WriteObject.Log.cs

```c#
using Newtonsoft.Json.Linq;
using System;

namespace PowerOffice.Worker.Contracts
{
    public interface IWriteLog
    {
        Guid StageId { get;  }
        JToken PublishContext { get; }
        JObject ExistingObject { get; }
        JObject PublishedObject { get; }
        string ObjectType { get; }
    }
}
```


### Worker Project

While a publisher can in theory run anywhere we have some restrictions placed upon us by .Net, Azure and MassTransit.

MassTransit can run anywhere .Net/.Net Core can run. Because publishers can be long running services we are currently developing to deploy publishers as Linux Containers.

The Visual Studio Worker Service project template with VS2019 and later handles this very well.

There is a skeleton project provided as an example, which also uses Autofac for managing dependencies within the publisher and has examples of how to use MassTransit with Autofac. It also injects a singleton http client to be re-used across all publisher instances.

The worker project will have an Activities folder with two classes defined:

1) Get.Activity.cs
2) Write.Activity.cs

#### Get.Activity.cs

```c#
using System;
using System.Threading.Tasks;
using MassTransit.Courier;
using Newtonsoft.Json.Linq;
using PowerOffice.Worker.Contracts;
using Serilog;

namespace PowerOffice.Worker
{
    public enum POObjectType { customer, customerAddress, product, project, department, outgoingInvoice }
    //Get has no compensation - it is a read only activity
    public class GetActivty : ExecuteActivity<IGetActivityArguments>
    {
        private readonly ILogger _log;
        private ITokenCache _tokenStore;
        private IPowerOfficeRestClient _po;

        public GetActivty(ILogger log,  IPowerOfficeRestClient poClient, ITokenCache tokenStore)
        {
            _log = log.ForContext<GetActivty>();
            _po = poClient;
            _tokenStore = tokenStore;
        }

        public async Task<ExecutionResult> Execute(ExecuteContext<IGetActivityArguments> context)
        {

            //Authenticate using configuration from the publish context.
            JToken pubContext = context.Arguments.PublishContext;
            AuthCredentials credentials = null;

            credentials = new AuthCredentials
            {
                ApplicationKey = pubContext["powerOffice"]["applicationKey"].Value<string>(),
                ClientKey = pubContext["powerOffice"]["clientKey"].Value<string>(),
                ProductionEnv = pubContext["powerOffice"]["productionEnv"] == null ? false : pubContext["powerOffice"]["productionEnv"].Value<bool>()
            };

            //PowerOffice publisher uses an in memory bearer token cache which is shared across all instances of the publisher.
            //The cache takes care of token refresh as well as token storage, after 1 hour of non-use the token is removed.
            //It can be that dozens or hundreds of publish jobs for the same PowerOffice account are happening asynchronously.
            //By caching the token for that PO account we can save many calls and re-use the same token for all jobs.
            //Depending on the API this may not be necessary, but for PowerOffice it makes a lot of sense.
            Token token = null;
            try
            {
                token = await _tokenStore.GetToken(credentials);
            }
            catch (Exception ex)
            {
                _log.Error("Authentication failure, {ex}", ex.Message);
                throw;
            }


            //check expected object types
            if (!Enum.TryParse(context.Arguments.ObjectType, out POObjectType poType))
            {
                _log.Error("Unknown object {$type}", context.Arguments);
                throw new Exception($"Unknown object {context.Arguments.ObjectType}");
            }

            //The contract properties are available the data passed to Execute.
            string jobId = context.Arguments.JobId;
            Guid stageId = context.Arguments.StageId;
            Guid documentId = context.Arguments.DocumentId;
            string objectType = context.Arguments.ObjectType;
            JToken payload = context.Arguments.Payload;

            JObject result;
            //Use the injected typed HttpClient to query PowerOffice for the given payload object
            try
            {
                result = await _po.Get(pubContext, poType, payload, token);
            }
            catch(Exception ex)
            {
                //The typed HttpClient is configured using Polly for transient failures.
                _log.Error(ex, "Failed to get PowerOffice object {objType} after 3 attempts. {$obj}", context.Arguments.ObjectType.ToString(), context.Arguments);
                throw;
            }

            //This call informs MassTransit we have successfully completed and adds the fetched object to the arguments that will be presented to the next step in the publish job (write data, snapshot data, or synchronise data)
            return context.CompletedWithVariables(new
            {
                ExistingObject = result
            });
        }
    }
}
```

#### Write.Activity.cs

```c#
using MassTransit.Courier;
using Newtonsoft.Json.Linq;
using PowerOffice.Worker.Contracts;
using Serilog;
using System;
using System.Linq;
using System.Threading.Tasks;

namespace PowerOffice.Worker
{
    public class WriteActivity : Activity<IWriteActivityArguments, IWriteLog>
    {
        private ILogger _log;
        private ITokenCache _tokenStore;
        private IPowerOfficeRestClient _po;
        private bool PowerOfficeIsMaster;

        public WriteActivtiy(ILogger log, IPowerOfficeRestClient poClient, ITokenCache tokenStore)
        {
            _log = log.ForContext<WriteActivity>();
            _po = poClient;
            _tokenStore = tokenStore;
        }

        public async Task<ExecutionResult> Execute(ExecuteContext<IWriteActivityArguments> context)
        {
            _log.Debug("Received WriteObject command");
            JObject result = null;

            //The contract properties are available the data passed to Execute.
            string jobId = context.Arguments.JobId;
            Guid stageId = context.Arguments.StageId;
            Guid documentId = context.Arguments.DocumentId;
            string objectType = context.Arguments.ObjectType;
            JToken payload = context.Arguments.Payload;
            JObject existingObject = context.Arguments.ExistingObject; //Note this was provided by the GetActivity

            try
            {
                result = await PublishObject(payload, existingObject, objectType, publishContext, stageId);
            }
            catch(Exception ex)
            {
                _log.Error(ex,"Failed to publish PowerOffice object {objectType}, {$payload}", objectType, payload);
                //A faulted activity will cause MassTransit to do several things, but one of them is to send a message to execute the compensate activity.
                return context.Faulted(ex);
            }

            //Check PowerOffices response for succes or failure
            //We add the published object to the routing slip for the next step (synchronise, snapshot, or another publisher)
            if( result["success"] != null && result["success"].Value<bool>())
            {
                return context.CompletedWithVariables(new
                {
                    PublishedObject = result,
                    Message = "Published to PowerOffice",
                });
            }
            else if(result["success"] != null && !result["success"].Value<bool>())
            {
                return context.CompletedWithVariables(new
                {
                    PublishedObject = result,
                    Message = result.ToString(), //attach the PowerOffice response so this can be reported
                    CompletedWithErrors = true
                });
            }
            else
            {
                return context.Faulted(new Exception($"Publishing to PowerOffice returned unexpected result. The publish has most likely failed, see inner exception.", new Exception(result.ToString())));
            }
        }

        //***************************************************************************************
        //A write activity must always have a compensate activity
        //The compensation activity is passed the Activity log message (WriteObject.Log.cs)
        //The compensation rolls back any changes made by the write activity.
        //***************************************************************************************
        public async Task<CompensationResult> Compensate(CompensateContext<IWriteLog> context)
        {

            AuthCredentials credentials = new AuthCredentials
            {
                ApplicationKey = context.Log.PublishContext["powerOffice"]["applicationKey"].Value<string>(),
                ClientKey = context.Log.PublishContext["powerOffice"]["clientKey"].Value<string>(),
                ProductionEnv = context.Log.PublishContext["powerOffice"]["productionEnv"] == null ? false : context.Log.PublishContext["powerOffice"]["productionEnv"].Value<bool>()
            };

            Token token = null;
            try
            {
                token = await _tokenStore.GetToken(credentials);
            }
            catch (Exception ex)
            {
                _log.Error("Authentication failure during compensation {stageId}, {ex}",context.Log.StageId, ex.Message);
                return context.Failed();
            }


            if (Enum.TryParse(context.Log.ObjectType, out POObjectType type))
            {
                JObject result;
                if (context.Log.ExistingObject == null || !context.Log.ExistingObject.HasValues && (context.Log.PublishedObject != null && context.Log.PublishedObject.HasValues))
                {
                    //Created a new object and it must be deleted
                    result = await _po.Delete(type, context.Log.PublishedObject, token);
                    if (result["success"].Value<bool>())
                    {
                        return context.Compensated();
                    }
                    else
                    {
                        _log.Error($"PowerOffice reported a failure to delete an object during a compensation {context.Log.StageId.ToString()}");
                        return context.Failed();
                    }
                }
                else
                {
                    //an object was updated and we must put it back how it was

                    result = await publishSimplePOObject(context.Log.PublishContext, type, context.Log.ExistingObject, token);

                    if (result["success"].Value<bool>())
                    {
                        return context.Compensated();
                    }
                    else
                    {
                        _log.Error($"PowerOffice reported a failure to update during a compensation {context.Log.StageId.ToString()}");
                        return context.Failed();
                    }
                }
            }
            else
            {
                _log.Error($"Unknown PowerOffice object type in compensation (log). {context.Log.StageId.ToString()}");
                return context.Failed();
            }
        }
    }
}
```
