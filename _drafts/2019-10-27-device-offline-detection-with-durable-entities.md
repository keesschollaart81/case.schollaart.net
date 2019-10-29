--- 
layout: post
title: "Device Offline detection with Durable Entities - Revisited"
author: "Kees Schollaart" 
backgroundUrl: /img/2019/back6.jpg
comments: true 
---  

How to detect the absense of device messages in a scalable, distributed and cost effictive way? Durable Functions to the rescue!

<!--more-->

## The Desire

If you maintain the backend of an IoT device it's very likely that you would like to have a 'Offline Detection' capability. Most devices send messages to the cloud with a known frequency, for example a heartbeat message. If you don't get any messages for more than x minutes... it's offline! 

This post describes one way of building this Offline Detection capability. In a very scalable and cost-effective way using cloud native technologies on the Microsoft Azure cloud.
 
##  The Challenges

Detecting offline devices might sound quite trivial, but if you continue to add zero's to the requirements it becomes more and more complex. This blogpost assumes 1.000.000 connected devices that ingest 1 message every 10 minutes (±1.000 p/s) via Azure EventGrid, Azure EventHub, IoT-Hub or an Azure Storage Queue. With these numbers, you start to run into challenges, for example...

### No message is no trigger

Some time, after the last received message, we want to run code that triggers the offine-event. A trigger needs to start code after some time, we don't want to manage all the timers/threads for all these devices. 

### Distributed state

To be scalable we would like to be stateless, Device Offline detection however is a stateful operation. How can we make this work on a scalable infrastructure where different messages can end up on different nodes. A distributed state is quite expensive in terms of both software complexity and IO. IO is also limiting scalability because of locks and at least 1 write and 1 read operation per message to deal with the state. 

### Disaster recovery

What if suddenly all devices disconnect and/or reconnect? This will cause an enormous peak in the offline detection logic. Both the backing-storage and the compute host need to be able to deal with this kind of peaks. When the detection logic get's overwealmed by a peak of messages or because it was offline during an update, it needs to deal with this 'delayed-processing' as well.

### Devicetype specific behaviour

Not every device is the same, especially if the backend has to work for devices from different manufacturers. The timeout has to be different per device. 

## Durable Entities to the rescue!

A solution to this challenge is to use Azure Durable Functions. Durable Functions are an extension of Azure Functions that lets you write stateful functions in a serverless environment. The extension manages state, checkpoints, and restarts for you. The rest of this post assumes basic understanding of [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview).

[Durable Entities](https://docs.microsoft.com/nl-nl/azure/azure-functions/durable/durable-functions-preview#entity-functions) is the newest addition to the Durable Functions framework (2.0 and later) and enables you to work with small pieces of state. This feature is heavily inspired by the Actor Model which you might know from [Akka.net](https://getakka.net/articles/intro/what-problems-does-actor-model-solve.html), [project Orleans](https://www.microsoft.com/en-us/research/project/orleans-virtual-actors/) or [Service Fabric Reliable Actors](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-introduction). 

This implementation for our device offline detection can be visualized in a sequence diagram like this:

<a id="single_image" href="/img/2019/sequenceUpdated.png" class="fancybox" rel="seq"><img src="/img/2019/sequenceUpdated-thumb.png" style="border:1px solid black;"/></a> 

### The Client Function

Similar to Orchestrators, Durable Entities cannot be reached directly via a normal trigger binding, to work with Entities we need a 'Client Function'. This 'Client Function' can be triggered by anything, in an IoT scenario it will probably be triggered by an IoT-Hub or Queue trigger. The Client Function takes a dependency on `IDurableEntityClient` which will be injected by the Durablable Framework. This `durableEntityClient` allows you to read the state of an Entity with `ReadEntityStateAsync()` and to call a method on an Entity with `SignalEntityAsync()`. When working with entities, an EntityId is always needed. This Id uniquely identifies the instance and state of an entity, in our example the DeviceId.  

In the example Client Function below we get the DeviceId from the HttpTrigger (used for simplicity) to construct the EntityId. Then the `SignalEntityAsync()` is called. The first argument being the DeviceId (and the name of the entity) and the second argument the reference to the method that we want to invoke.

Although there is an await statement, it is a 'fire and forget' operation as the actual operation on the entity will happen later. There is an await statement because of the IO it takes to persist the operation to an internal queue. So the Client Function completes very quickly and the Durable Framework will asynchronously instantiate the Entity and invoke the `MessageReceived` method.

~~~ cs
[FunctionName(nameof(HttpTrigger))]
public static async Task<IActionResult> HttpTrigger(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpTriggerArgs args,
    [DurableClient] IDurableEntityClient durableEntityClient,
    ILogger log)
{
    log.LogInformation($"Receiving message for device {args.DeviceId}");

    var entity = new EntityId(nameof(DeviceEntity), args.DeviceId);
    await durableEntityClient.SignalEntityAsync(entity, nameof(DeviceEntity.MessageReceived));

    return new OkResult();
}
~~~

### Device Entity

The entity is the stateful object we work with. The instance has an unique ID and can represent anything, from a user to a building and in our ase a device. In our Device Entity we manage the `LastCommunicationDateTime` properties/state. Entities can be implemented in two patterns: 'Function Based' and 'Class Based', in example I use the 'Class Based' pattern. So each device will get an instance of my 'DeviceEntity' class:

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

Entities need an entry point to construct the entity, this static method is decorated with the `[FunctionName()]` attribute and takes a `IDurableEntityContext` as an argument. With the `IDurableEntityContext` the entity is constructed via the `DispatchAsync()` method and an initiate state can be provisioned with `SetState()`.

The values of properties on the class will be automatically serialized to the state of the object after working with them. So if a Client Function calls the `MessageReceived()` method, the `DeviceEntity` is autmatically instantiated and in the body of `MessageReceived()` the properties of the class are recovered from (persistent) state. So properties like `this.LastCommunicationDateTime` can be updated and then, when `MessageReceived()` returns, Durable Functions will persist the state before it executes a new operation for this specific entity.

So far the basics of Durable Entities. To get Device Offline detection to work in a scalable way, we need some more infrastructure...

### Timeout Queue


~~~cs
public async Task MessageReceived()
{
    this.LastCommunicationDateTime = DateTime.UtcNow;
 
    if (this.TimeoutQueueMessageId == null)
    {  
        // put message on timeout queue with visibility of the 'OfflineAfter' of this device

        var message = new CloudQueueMessage(this.Id);
        await timeoutQueue.AddMessageAsync(message, null, this.OfflineAfter, null, null);

        // store the reference of this queue message in the state of this device
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


### Status Changes

### Dashboard & TestApp
  
### What does this enable?

So... we have offline detection and the LastCommunication in the Azure Functions Durable Entity state, now what?

- We can push out a message on state changes (no polling required)
- We can expose an HTTP Trigger based Function to return the Last Communication DateTime
- We only depend on Azure Storage, 20.000 request/sec, low cost
- No reserved capacity for VM's, containers, CosmosDb... No messages == No cost!
- No plumbing-code for triggers, distributed state, scaling and resiliency thanks to Azure (Durable) Functions!

## Performance

Durable Entities is, at this time, still in preview and there is an explicit note about performance. Still I wanted to get some sense of the current state and I ran a short loadtest via loader.io. It only allowes for a 60 seconds load test:

<a id="single_image" href="/img/2019/loadtest1.png" class="fancybox" rel="loadtest"><img src="/img/2019/loadtest1-thumb.png"/></a> 
<a id="single_image" href="/img/2019/loadtest2.png" class="fancybox" rel="loadtest"><img src="/img/2019/loadtest2-thumb.png"/></a> 

In these 60 seconds I was already able to scale to more than 1.000 requests per second! I'm sure that with some tweaking on the Durable Functions configuration options this will scale much further! 
 
## Conclusion

I think Durable Entities is quite a powerful construct and enables a lot of advanced distributed statefull scenario's in a very scalable and const effective way. 

The code for this PoC can be found on [GitHub](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/).



# removed
 - Compare with CosmosDb
 - Functions 1.0
 - Orchestrator