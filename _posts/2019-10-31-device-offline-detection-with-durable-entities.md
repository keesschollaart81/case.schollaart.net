--- 
layout: post
title: "Device Offline detection with Durable Entities - Revisited"
author: "Kees Schollaart" 
backgroundUrl: /img/2019/back5.jpg
comments: true  
pageThemeColor: "#5c0000"
description: How to detect the absence of device messages in a scalable, distributed and cost effective way? Durable Functions to the rescue!
---  

How to detect the absence of device messages in a scalable, distributed and cost effective way? Durable Functions to the rescue! Un update of the previous post on this idea, as part of the final release of Durable Functions 2.0.

<!--more-->

> This is a follow-up on a [post]({% post_url 2019-07-01-device-offline-detection-with-durable-entities %}) I did ~4 months ago. Since then, the technology I used (Durable Entities) has gone through some beta's and is now General Available! Next to that, I got a lot of feedback and ideas on how to improve it so I've completely rewritten everyting except from the problem statement.

## The Desire

If you maintain the backend of an IoT device it's very likely that you would like to have a 'Offline Detection' capability. Most devices send messages to the cloud with a known frequency, for example a heartbeat message. If you don't get any messages for more than x minutes... it's offline! 

This post describes one way of building this Offline Detection capability, in a very scalable and cost-effective way using cloud native technologies on the Microsoft Azure cloud.
 
##  The Challenges

Detecting offline devices might sound quite trivial, but if you continue to add zero's to the requirements it becomes more and more complex. This blogpost assumes 1.000.000 connected devices that emit 1 message every 10 minutes (±1.600 p/s) via Azure EventGrid, Azure EventHub, IoT-Hub or an Azure Storage Queue. With these numbers, you start to run into challenges, for example...

### No message is no trigger

Some time, after the last received message, we want to run code that triggers the offine-event. A trigger needs to start code after some time, we don't want to manage all the timers/threads for all these devices. 

### Distributed state

To be scalable we would like to be stateless, Device Offline detection however is a stateful operation. How can we make this work on a scalable infrastructure where different messages can end up on different nodes. A distributed state is quite expensive in terms of both software complexity and IO.  While every operation requires at least 1 write and 1 read operation per message to deal with the state, we would like to avoid IO and locks as much as possible. 

### Disaster recovery

What if suddenly all devices disconnect and/or reconnect? This will cause an enormous peak in the offline detection logic. Both the backing-storage and the compute host need to be able to deal with this kind of peaks. When the detection logic get's overwhelmed by a peak of messages or because it was offline during an update, it needs to deal with this 'delayed-processing' as well.

### Devicetype specific behavior

Not every device is the same, especially if the backend has to work for devices from different manufacturers. The timeout has to be different per device. 

### So, what do we need?

Let quickly summarize the requirements:

- Serverless infrastructure (scalable, no infra-burden, cost effective)
- High throughput (>1000 messages per second)
- As less IO as possible
- Push mechanism for device state changes
- Pull mechanism for current state of device

## Durable Entities to the rescue!

A solution to this challenge is to use Azure Durable Functions. Durable Functions are an extension of Azure Functions that lets you write stateful functions in a serverless environment. The extension manages state, checkpoints, and restarts for you. The rest of this post assumes basic understanding of [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview).

[Durable Entities](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities) is the newest addition to the Durable Functions framework (2.0 and upwards) and enables you to work with small pieces of state. This feature is heavily inspired by the Actor Model which you might know from [Akka.net](https://getakka.net/articles/intro/what-problems-does-actor-model-solve.html), [project Orleans](https://www.microsoft.com/en-us/research/project/orleans-virtual-actors/) or [Service Fabric Reliable Actors](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-introduction). 

This implementation for our device offline detection can be visualized in a sequence diagram like this:

<a id="single_image" href="/img/2019/sequenceUpdated.png" class="fancybox" rel="seq"><img src="/img/2019/sequenceUpdated-thumb.png" style="border:1px solid black;"/></a>
<!--
<a id="single_image" href="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/sequenceUpdated.png"><img src="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/sequenceUpdated-thumb.png" style="border:1px solid black;"/></a>
--> 

### The Client Function

Similar to Orchestrators, Durable Entities cannot be reached directly via a normal trigger binding, to work with Entities we need a 'Client Function'. A 'Client Function' is a normal Azure Functions that can be triggered by anything and can interact with Entities. Below an example Client Function, this Client Function will be triggered for every device message and interacts with our Durable Entity:

~~~ cs
[FunctionName(nameof(QueueTrigger))]
public async Task QueueTrigger(
    [QueueTrigger("device-messages")] CloudQueueMessage message,
    [DurableClient] IDurableEntityClient durableEntityClient,
    ILogger log)
{
    log.LogInformation($"Receiving message for deviceId: {message.AsString}");

    var entity = new EntityId(nameof(DeviceEntity), message.AsString);
    await durableEntityClient.SignalEntityAsync(entity, nameof(DeviceEntity.MessageReceived));
}
~~~

The Client Function takes a dependency on `IDurableEntityClient` which will be injected by the Durablable Framework. This `durableEntityClient` can be used both to read the state of an Entity with `ReadEntityStateAsync()` and to invoke a method on an Entity with `SignalEntityAsync()`. When working with entities, an EntityId is always needed. This Id uniquely identifies the instance and state of an entity by it's name and Id, in our example the 'DeviceEntity' and the Id of the device.  

In the 'Client Function' from the example below, the DeviceId is read from the queue message to construct the EntityId. Then the `SignalEntityAsync()` is called with 2 arguments, first the DeviceId (and the name of the entity) and the secondly the name of the method that we want to invoke (`MessageReceived`).

Although there is an await statement, `SignalEntityAsync()` is a 'fire and forget' operation as the actual method invocation on the entity will happen later. There is an await statement because of the IO it takes to persist the operation to an internal queue. So the Client Function completes very quickly and the Durable Framework will asynchronously instantiate the Entity and invoke the `MessageReceived` method.


### Device Entity

The entity is the stateful object we work with. An entity can represent anything, from a user to a building and in our scenario a device. In our `DeviceEntity` we keep track of the `LastCommunicationDateTime` properties/state. Entities can be implemented in two patterns: 'Function Based' and 'Class Based', in this example I use the 'Class Based' pattern. So each device will get an instance of the 'DeviceEntity' class.

The state of the Entity lives in the properties of an object, Durable Framework does the (de)serialization every time code starts/stops on an instance of an entity. In the example below, the `Id` and `LastCommunicationDateTime` will be set/managed by the Durable Framework.

~~~ cs
[JsonObject(MemberSerialization.OptIn)]
public class DeviceEntity
{
    [JsonProperty]
    public string Id { get; set; }

    [JsonProperty]
    public DateTime? LastCommunicationDateTime { get; set; }

    private readonly ILogger logger;

    public DeviceEntity(string id, ILogger logger)
    {
        this.Id = id;
        this.logger = logger;
    }

    [FunctionName(nameof(DeviceEntity))]
    public static async Task HandleEntityOperation(
        [EntityTrigger] IDurableEntityContext context,
        ILogger logger)
    {
        if (context.IsNewlyConstructed)
        {
            context.SetState(new DeviceEntity(context.EntityKey, logger));
        }

        await context.DispatchAsync<DeviceEntity>(context.EntityKey, logger);
    }

    public async Task MessageReceived()
    {
        this.LastCommunicationDateTime = DateTime.Now;

        // removed the rest of the implementation for brevity
    } 
}
~~~

Durable Frameworks needs an entry point to construct the entity, this static method is decorated with the `[FunctionName(...)]` attribute and takes a `IDurableEntityContext` as an argument. In this operation the class based entity needs to be instantiated via the `DispatchAsync()` method. An initial state can be provisioned with the `SetState()` operation.

The values of properties on the object will be automatically serialized to the state of the object after working with them. So if a Client Function calls the `MessageReceived()` method, the `DeviceEntity` is automatically instantiated and in the body of `MessageReceived()` the properties of the object are recovered (from the persistent state). So properties like `this.LastCommunicationDateTime` can be updated and then, when `MessageReceived()` returns, Durable Functions will persist the state before it executes a new operation for this specific entity.

### Entities and Dependencies

How do we do dependency injection and IO in a Durable Entity?

In this scenario I decided to publish the state of the Device to Azure SignalR Service. Later, I also need an Azure Storage Queue for timeout messages. Instance methods on Durable Entity classes cannot take input or output bindings. Input and Output bindings are only available on the entry point of the entity, in our example the static `HandleEntityOperation()` method. This method is responsible for the instantiation of the entity and can pass these services/dependencies to the constructor of the entity.

~~~cs
public DeviceEntity(string id, ILogger logger, CloudQueue timeoutQueue, IAsyncCollector<SignalRMessage> signalRMessages)
{
    this.Id = id;
    this.logger = logger;
    this.timeoutQueue = timeoutQueue;
    this.signalRMessages = signalRMessages;
}

[FunctionName(nameof(DeviceEntity))]
public static async Task HandleEntityOperation(
    [EntityTrigger] IDurableEntityContext context,
    [SignalR(HubName = "devicestatus")] IAsyncCollector<SignalRMessage> signalRMessages,
    [Queue("timeoutQueue")] CloudQueue timeoutQueue,
    ILogger logger)
{
    // inject the dependencies and input/output bindings to the constructor of the entity
    await context.DispatchAsync<DeviceEntity>(context.EntityKey, logger, timeoutQueue, signalRMessages);
}
~~~

The constructor takes all the dependencies as you're used to, in this case a reference to ILogger, a CloudQueue, etc. After constructing this Entity, the Durable Framework can invoke instance methods such as `MessageReceived`, then, the fields are available as if they were input/output bindings (but now as field on the object).

Now that we have a working entity, how can we keep track of the devices online/offline state? 

### Timeout Queue

The `MessageReceived` operation on the DeviceEntity will be invoked for every message coming from the device. Here we will use the 'Timeout Queue'. In the Timeout Queue we put 1 message per device. Every time we get a message from the device we check if there is already a message in the Timeout Queue, if not we add one. On this new message, the 'Visibility Timeout' is set equal to the 'Offline After' of a device. In our example 'Offline After' is fixed to 30 seconds but this can be a variable value per device.

With Azure Storage Queues it is possible to update a message that is currently in a queue by using a reference of this message (Id and PopReceipt). This reference, we store as state on the Device Entity. As long as messages come in within these 30 seconds and there is a reference to a message in the Timeout Queue, the 'Visibility Timeout' of this message is reset to 30 seconds from now.

~~~cs
public async Task MessageReceived()
{
    this.LastCommunicationDateTime = DateTime.UtcNow;
 
    if (this.TimeoutQueueMessageId == null)
    {  
        // no message in the TimeoutQueue yet
        // put message on timeout queue with visibility of the 'OfflineAfter' of this device

        var message = new CloudQueueMessage(this.Id);
        await timeoutQueue.AddMessageAsync(message, null, this.OfflineAfter, null, null);

        // store the reference to this queue message in the state of this device
        this.TimeoutQueueMessageId = message.Id;
        this.TimeoutQueueMessagePopReceipt = message.PopReceipt;

        // tell the world this device is now online
        await this.ReportState("online");
    }
    else
    {
        // there is already a messag in the timeout queue, create a reference to it
        var message = new CloudQueueMessage(this.TimeoutQueueMessageId, this.TimeoutQueueMessagePopReceipt);

        // update the message in the queue with a new/reset 'OfflineAfter' visibility timeout
        await this.timeoutQueue.UpdateMessageAsync(message, this.OfflineAfter.Value, MessageUpdateFields.Visibility);
    }
}
~~~

When a device turns offline, there will be no message in the 'OfflineAfter' time period causing the message to be released from the TimeoutQueue. This will trigger another normal client function (`HandleOfflineMessage`) which will invoke the `DeviceTimeout()` method on our DeviceEntity.

> In the next release of Durable Functions (2.1) it will be possible create 'entity reminders'. With [this feature](https://github.com/Azure/azure-functions-durable-extension/issues/716) it's possible to let an entity signal itself on a given schedule. This could potentially eliminate the need of this timeout queue and simplify the implementation of this offline detection even more.   

### Read the state

Now that we have ve seen how to track and push out status changes, let's look at how can we implement an endpoint that allows for systems to get the current state of a device.

For this I've build another Client Function based on a HTTP trigger. This `GetStatus` function will return the state for a specific device. 

To get the state of an entity we need the `IDurableEntityClient` again. In the previous Client Function we used the `SignalEntityAsync`, this time we will use  `ReadEntityStateAsync`. `ReadEntityStateAsync` can be used to read the state of an entity. Other than `SignalEntityAsync`, `ReadEntityStateAsync` will go to storage to get the data and directly return it. 

~~~cs
[FunctionName(nameof(GetStatus))]
public static async Task<IActionResult> GetStatus(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpTriggerArgs args,
    [DurableClient] IDurableEntityClient durableEntityClient)
{
    var entity = new EntityId(nameof(DeviceEntity), args.DeviceId);
    var device = await durableEntityClient.ReadEntityStateAsync<DeviceEntity>(entity);

    return new OkObjectResult(device);
} 
~~~

### Status Changes & Dashboard

The DeviceEntity is responsible to publish status changes, there are a dozen ways one can do that, for this demo I chose Azure SignalR Service. It's really easy to publish messages to SignalR using the output bindings. I also expose the negotiate endpoint that SignalR clients need in my Azure Functions app. This way, my entire app can run self contained within serverless infrastructure.

~~~cs
private async Task ReportState(string state)
{
    await this.signalRMessages.AddAsync(new SignalRMessage
    {
        Target = "statusChanged",
        Arguments = new[] { new { deviceId = this.Id, status = state } }
    });
}
~~~

To test this Device Offline Detection meganism, I've build a very simple dashboard. The dashboard uses the SignalR client side SDK to connect to the negotiate endpoint in Azure Functions which will 'redirect' it to Azure SignalR Service. Then with some javascript the device status changes are visualized...

<a href="/img/2019/dashboard.gif" class="fancybox" rel="dashboard" ><img src="/img/2019/dashboard_still.png"/></a> 
<!--
<a  href="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/dashboard.gif" ><img src="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/dashboard_still.png"/></a> 
-->
### What does this enable?

So... we have offline detection and the LastCommunication DateTime in the Azure Functions Durable Entity state, now what?

- We can push out a message on state changes (no polling required)
- We can expose an HTTP Trigger based Function to return the LastCommunication DateTime
- We only depend on Azure Storage which can take up to 20.000 request/sec and is low in cost
- No reserved capacity for VM's, containers, CosmosDb... No messages == No cost!
- No plumbing-code for triggers, distributed state, scaling and resiliency thanks to Azure (Durable) Functions!

## Performance

As this blogpost started with some requirements on performance I wanted to see how far we can stretch Durable Entities. To test this, I ran a simple loadtest using a [TestDevice](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/blob/dev/src/TestDevice/Program.cs) (just a Console App) that puts messages in a queue. 

I stopped the tests before everything melted. In the background, I monitored the internal queues of Durable Functions and I stopped the load test when I noticed that the workers were not able to keep the queues empty any more (>1000 messages in the queue). 

Below some screenshots I took from Azure Monitor showing the number of requests that Azure Functions processed. 

<a  href="/img/2019/loadtest3.png" class="fancybox" rel="loadtest" title="Load Test with normal Consumption plan. Purple offline messages at the end."><img src="/img/2019/loadtest3-thumb.png"/></a> 
<a  href="/img/2019/loadtest4.png" class="fancybox" rel="loadtest" title="Second load test on Premium Consumption plan"><img src="/img/2019/loadtest4-thumb.png"/></a> 
<a  href="/img/2019/loadtest5.png" class="fancybox" rel="loadtest" title="Second load test on Premium Consumption plan scaling to ~18 nodes"><img src="/img/2019/loadtest5-thumb.png"/></a> 

<!--
<a href="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest3.png" title="Load Test with normal Consumption plan. Purple offline messages at the end."><img src="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest3-thumb.png"/></a> 
<a href="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest4.png"  title="Second load test on Premium Consumption plan"><img src="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest4-thumb.png"/></a> 
<a  href="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest5.png"  title="Second load test on Premium Consumption plan scaling to ~18 nodes"><img src="https://raw.githubusercontent.com/keesschollaart81/case.schollaart.net/master/img/2019/loadtest5-thumb.png"/></a> 
-->

A normal Azure Functions Consumption plan was able to process 300 messages per second. I also did a testrun with an Azure Functions Premium Consumption plan with the mid-sized ES2 SKU. This run (screenshot 2 and 3) was able to process ~1250 messages per second. 

## Conclusion

I think Durable Entities is quite a powerful construct and enables a lot of advanced distributed stateful scenario's in a very scalable and cost effective way. 

The code for this PoC can be found on [GitHub](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/). The readme of this repository contains all the information needed to run this example yourself as it contains both the [Azure Pipelines YAML definition](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/blob/dev/azure-pipelines.yaml) as well the [ARM template](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/blob/dev/src/AzureResourceGroup/azuredeploy.json) to provision the Azure infrastructure. 
