--- 
layout: post
title: "Device Offline detection with Durable Entities"
author: "Kees Schollaart" 
backgroundUrl: /img/2019/back5.jpg
pageThemeColor: "#2e0000"
comments: true 
---  

How to detect the absense of device messages in a scalable, distributed and cost effictive way? Durable Functions to the rescue!

<!--more-->

## The Desire

If you maintain the backend of an IoT device it's very likely that you would like to have a 'Offline Detection' capability. Most devices send messages to the cloud with a known frequency, for example a heartbeat message. If you don't get any messages for more than x minutes, you know... It's offline! 

This post describes one way of building this Offline Detection capability. In a very scalable and cost-effective way using cloud native technologies on the Microsoft Azure cloud.
 
##  The Challenges

Altough it might sound quite trivial, but if you continue to add zero's to the requirements it becomes less and less trivial. This blogpost assumes 1.000.000 connected devices that ingest 1 message every 10 minutes (±1.000 p/s) via Azure EventGrid, Azure EventHub, IoT-Hub or an Azure Storage Queue. With these numbers, you start to run into challenges, for example...

### No message is no trigger

Some time after the last received message, we want to run code that triggers the offine-event. A trigger needs to start code after some time, we don't want to manage all the timers/threads for all these devices. 

### Distributed state

To be scalable we would like to be stateless, Device Offline detection however is a stateful operation. How can we make this work on a scalable infrastructure where different messages can end up on different nodes. A distributed state is quite expensive in terms of both software complexity and IO. IO is also limiting scalability because of locks and at least 1 write and 1 read operation per message to deal with the state (last message received at). 

### Disaster recovery

What if suddenly all devices disconnect and/or reconnect? This will cause an enormous peak in the offline detection logic. Because of this, both the backing-storage and the compute host need to be able to deal with sudden load. When the detection logic get's overwealmed, by a peak of messages or because it was offline during an update, it needs to deal with this 'delayed-processing' as well.

### Devicetype specific behaviour

Not every device is the same, especially if the backend has to work for devices from different manufacturers. The timeout has to be different per device. 

## Durable Entities to the rescue!

A solution to this challenge is to use Azure Durable Functions. Durable Functions are an extension of Azure Functions that lets you write stateful functions in a serverless environment. The extension manages state, checkpoints, and restarts for you. The rest of this post assumes basic understanding of [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview).

[Durable Entities](https://docs.microsoft.com/nl-nl/azure/azure-functions/durable/durable-functions-preview#entity-functions) is a recent addition to the Durable Functions framework (2.0 and later) and enables you to work with small pieces of state. This feature is heavily inspired by the Actor Model which you might know from [Akka.net](https://getakka.net/articles/intro/what-problems-does-actor-model-solve.html), [project Orleans](https://www.microsoft.com/en-us/research/project/orleans-virtual-actors/) or [Service Fabric Reliable Actors](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-introduction). 

This implementation for our device offline detection can be visualized in a sequence diagram like this:

<a id="single_image" href="/img/2019/sequence.png" class="fancybox" rel="seq"><img src="/img/2019/sequence-thumb.png" style="border:1px solid black;"/></a> 

### The Function

For every incoming message we check if there is already an orchestrator running/waiting. This lookup is by ID and in our example we used the device ID as the ID for the orchestrator. 

~~~ cs
[FunctionName(nameof(HttpTrigger))]
public static async Task<IActionResult> HttpTrigger(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpTriggerArgs args,
    [OrchestrationClient] IDurableOrchestrationClient durableOrchestrationClient,
    ILogger log)
{
    var status = await durableOrchestrationClient.GetStatusAsync(args.DeviceId);

    switch (status?.RuntimeStatus)
    {
        case OrchestrationRuntimeStatus.Running:
        case OrchestrationRuntimeStatus.Pending:
        case OrchestrationRuntimeStatus.ContinuedAsNew:
            await durableOrchestrationClient.RaiseEventAsync(args.DeviceId, "MessageReceived", null);
            break;
        default:
            await durableOrchestrationClient.StartNewAsync(nameof(WaitingOrchestrator), args.DeviceId, new OrchestratorArgs { DeviceId = args.DeviceId });
            break;
    }
    return new OkResult();
}
~~~

If there is already a running orchestrator, this running orchestrator is notified that there is a new message. The framework will (in a asynchronous manner) wakeup the this orchestrator.

### The Orchestrator

The first time an orchestrator is started, it will create the Durable Entity, then it will fetch the properties of this particular device.

Then the orchestrator will do a `WaitForExternalEvent` for the given time(out). While it's waiting, the Durable Functions framework will 'kill' the thread. The orchestrator thread will be revived when this event is triggered or the timeout period has elapsed.  

~~~ cs
[FunctionName(nameof(WaitingOrchestrator))]
public static async Task WaitingOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext ctx,
    ILogger log)
{
    var orchestratorArgs = ctx.GetInput<OrchestratorArgs>();

    var entity = new EntityId(nameof(DeviceEntity), orchestratorArgs.DeviceId);

    var offlineAfter = await ctx.CallEntityAsync<TimeSpan>(entity, "GetOfflineAfter");
    var lastActivity = await ctx.CallEntityAsync<DateTime?>(entity, "GetLastMessageReceived");
    try
    { 
        await ctx.WaitForExternalEvent("MessageReceived", offlineAfter); 
        log.LogInformation($"Message received, resetting timeout!"); 
        ctx.ContinueAsNew(orchestratorArgs);
        return;
    }
    catch (TimeoutException)
    {
        await ctx.CallActivityAsync(nameof(SendStatusUpdate), new StatusUpdateArgs(orchestratorArgs.DeviceId, false));
        log.LogInformation($"No message received, orchestrator will terminate"); 
        return;
    }
}
~~~

When the offline detection timeout has been reached, a `TimeoutException` will be thrown, then we call the `SendStatusUpdate` activity function. When the `MessageReceived` event is raised, `ctx.ContinueAsNew` is called whichs will 'bring back' the orchestrator to it's next iteration/state. As long as the device is online, this orchestrator is considered to live forever.

### The Entity

The entity is the stateful object we work with. The instance has an unique ID and can represent anything, from a user to a building and in our ase a device. In our Device Entity we manage the `OfflineAfter` and `LastCommunicationDateTime` properties/state. With Durable Entities you implement an Entity as if it is a normal Azure Function:

~~~ cs
[FunctionName(nameof(DeviceEntity))]
public static async Task DeviceEntity([EntityTrigger] IDurableEntityContext ctx)
{
    var device = ctx.GetState<Device>(); // deserialize state for this device 
    if (device == null)
    {
        device = new Device(); 
        device.OfflineAfter = TimeSpan.FromSeconds(30);
        ctx.SetState(device); // from now on, we have a state for this new device
    }

    switch (ctx.OperationName)
    {
        case "UpdateLastCommunicationDateTime":
            device.LastCommunicationDateTime = DateTime.UtcNow;
            ctx.SetState(device);
            break;
        case "GetLastMessageReceived":
            ctx.Return(device.LastCommunicationDateTime);
            break;
        case "GetOfflineAfter":
            ctx.Return(device.OfflineAfter);
            break;
    }
} 
~~~

For now, all interactions with the Entity are implemented via the `switch` on `ctx.OperationName`. This will change [later](https://medium.com/@cgillum/azure-functions-durable-entities-67db648d2f74) so that properties/methods can be used.

### What does this enable?

So... We have offline detection and the LastCommunication in the Azure Functions Durable Entity state, now what?

- We can push out a message on state changes (no polling required)
- We can expose an HTTP Trigger based Function to return the Last Communication DateTime
- We only depend on Azure Storage, 20.000 request/sec, low cost
- No reserved capacity for VM's, containers, CosmosDb... No messages == No cost!
- No plumbing-code for triggers, distributed state, scaling and resiliency thanks to Azure (Durable) Functions!

## Yes, but have you tought about...

### CosmosDb

Using CosmosDb (Triggers) to work with the state was definitely on my radar. CosmosDb still requires you to provision/assume some load. In typical scenario's, this fits well since the load on the DB correlates very much with the (fairly stable) number of connected devices. In exceptional cases where all devices get disconnected or when the Function App stopped for some time, it's very difficult to recover. For example, Azure Functions Consumption plan keeps scaling up when the queue is full even though CosmosDb is already giving 429 exceptions. Next to that, CosmosDb is quite expensive compared to plain Azure Storage.

In most cases, CosmosDb is a natural fit you the device registry/metadata. The backing-storage for the online/offline state however, is better off with Azure Storage. 

### Stream Analytics

Stream Analytics enables you to make conclusions on streaming data. For this challenge, Steam Analytics seems to be a potential solution. I see 2 problems:
- Is complicated to work with a device-specific timeout. Stream Analytics can use reference data for lookups like this but a device registry of 1.000.000 is just to much
- No message means no trigger point. It's very hard to create a 'there is no message' conclusion with Stream Analytics since it only allows you to make conclusions over stream data.

### Durable Functions 1.x

This blogpost really focusses on solving this with Durable Entities which is part of the Durable Functions 2.0 release. We also can solve this challenge with Durable Function 1.0, it's not as nice but...

~~~ cs
[FunctionName(nameof(WaitingOrchestrator))]
public static async Task WaitingOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext ctx,
    ILogger log)
{
    var orchestratorArgs = ctx.GetInput<OrchestratorArgs>();

    if (!orchestratorArgs.OfflineAfter.HasValue){
        orchestratorArgs.OfflineAfter = await ctx.CallActivityAsync<int>(nameof(GetOfflineAfter), orchestratorArgs.DeviceId);
    }
        
    try
    {
        await ctx.WaitForExternalEvent("MessageReceived", offlineAfter); 
        ctx.ContinueAsNew(orchestratorArgs);
        return;
    }
    catch (TimeoutException)
    { 
        await ctx.CallActivityAsync(nameof(SendStatusUpdate), new StatusUpdateArgs(orchestratorArgs.DeviceId, false));
        return;
    }
}
~~~

So the first run of the Orchestrator, we use an Activity Function called `GetOfflineAfter` to get the `OfflineAfter` timespance, then it's passed through. The downside of this is, there is no state to call. So we cannot 'ask' anyone what the current state is or when there was a last message.

## Performance

Durable Entities is at this time still in preview and there is an explicit note about performance. Still I wanted to get some sense of the current state and I ran a short loadtest via loader.io. It only allowes for a 60 seconds load test:

<a id="single_image" href="/img/2019/loadtest1.png" class="fancybox" rel="loadtest"><img src="/img/2019/loadtest1-thumb.png"/></a> 
<a id="single_image" href="/img/2019/loadtest2.png" class="fancybox" rel="loadtest"><img src="/img/2019/loadtest2-thumb.png"/></a> 

In these 60 seconds I was already able to scale to more than 1.000 requests per second! I'm sure that with some tweaking on the Durable Functions configuration options this will scale much further! 
 
## Conclusion

I think Durable Entities is quite a powerful construct and enables a lot of advanced distributed statefull scenario's in a very scalable and const effective way. 

The code for this PoC can be found on [GitHub](https://github.com/keesschollaart81/ServerlessDeviceOfflineDetection/).