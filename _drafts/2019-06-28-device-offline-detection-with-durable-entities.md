--- 
layout: post
title: "Device Offline detection with Durable Entities"
author: "Kees Schollaart" 
backgroundUrl: /img/2019/back5.jpg
comments: true 
---  

How to detect the absense of device messages in a scalable, distributed and cost effictive way? Durable Functions to the rescue!

<!--more-->

## The Desire

If you maintain the backend of an IoT device it's very likely that you would like to have a 'Offline Detection' capbility. Most devices send messages to the cloud with certain frequency, for example a heartbeat message. If you don't get any messages, at some point you know... It's offline! 

This post describes one way of building this Offline Detection capability. In a very scalable and cost effective way using cloud native technologies on the Microsoft Azure cloud.
 
##  The Challenges

Altough is might sound quite trivial, doing it at a scale of more than 100 messages per second (even up to 100.000 per second) brings quite some challenges. For example... This blogpost assumes 1.000.000 connected devices that ingest 1 message every minute (±17k p/s) via Azure EventHub, IoT-Hub or an Azure Storage Queue. 

### No entry point

Some time after the last received message, we want to run code that triggers the offine-event. We need to have a way to trigger this without a running thread per device. 

### Distributed state

To be scalable we would like to be stateless, Device Offline detection however is a stetful operation. How can we make this work on a scalable infrastructure where different messages can end up on different nodes. A distributed state is quite expensive in terms of both software complexity and IO. IO is also limiting scalability because of locks and at least 1 write and 1 read operation per message to deal with the state (last message received at). 

### Disaster recovery

What if suddenly all devices disconnect and/or reconnect? This will cause a enormous peak in the offline detection logic. Because of this, both the backing-storage and the compute host need to be able to deal with these sudded peaks. Wthen the detection logic get's overwealmed by a peak of messages or because it was offline for a but during an update, it needs to compensate for that 'processing-latency' as well.

### Devicetype specific behaviour

Not every device is the same, especially if the backend has to work for devices from different manufacturers. The timeout to work with will be different per device. 

## Durable Entities to the rescue!

A solution to this challenge is to use Azure Durable Functions. Durable Functions are an extension of Azure Functions that lets you write stateful functions in a serverless environment. The extension manages state, checkpoints, and restarts for you. The rest of this post assumes basic understanding of [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview).

[Durable Entities](https://docs.microsoft.com/nl-nl/azure/azure-functions/durable/durable-functions-preview#entity-functions) is a recent addition to the Durable Functions framework (2.0 and later) and enables work with small pieces of state. This feature is heavily inspired by the Actor Model which you might know from [Akka.net](https://getakka.net/articles/intro/what-problems-does-actor-model-solve.html), [project Orleans](https://www.microsoft.com/en-us/research/project/orleans-virtual-actors/) or [Service Fabric Reliable Actors](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-actors-introduction). 

This implementation for our device offline detection can be visualized in a sequence diagram like this:

<img src="https://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgT2ZmbGluZSBEZXRlY3Rpb24KCkV2ZW50U291cmNlLT4rRnVuABMFOiBGaXJzdCBvciBkZWxheWVkIG1lc3NhZ2UKABsILT5EdXJhYmxlRnJhbWV3b3JrOiBOZXcgT3JjaGVzdHJhdG9yIAAjCy0AXws6CgAsEC0-KisALgwARwUKAEAMLT4qRW50aXR5OiBDcmVhdGUADw8AFQhVcGRhdGUgTGFzdENvbW11bmljYXRpb25EYXRlVGltACcQU3RhdHVzTm90aWZpZXI6IE9ubGluAEwQLQCBWRJXYWl0IGZvciB4CiNub3RlIHJpZ2h0IG9mIEJvYjogQm9iIHRoaW5rcyBhYm91dCBpdCAKIAogCgphbHQgTm8AgkYJAIF3EgCBeQ9UaW1lb3V0RXhjZXAAgy4FAIFaHXN0YXRlIHRvIG8Ag2UGAIE5EACBXhEAIAdlbHNlIE5vcm1hbACDehkAGQYAhAEIAINQDACDeBJSYWlzZQCEVwVBc3luYyAAg14sAIN6D1dha2UgdXAgJiBjb250aW51ZSEgAIM2RgCESAhHZXQgQ3VycmVudCBTdACEShEAhBMSAIQqBXkAhCUHICh3aGVuIGMAOgcAgnwGaXMAgnoIKQCEKiJDAIFRByBhcwCGPgUgCmVuZCAgCg&s=napkin"/>

### The Function

For every incoming message we check if there is already an orchestrator running/waiting. This lookup is by ID and in our example we used the device ID as the ID for the orchestrator. 

```cs
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
```

If there is already a running orchestrator, this orchestrator is notified that there is a new message. The framework will (in a asynchronous manner) wakeup the this orchestrator.

### The Orchestrator

The first time an orchestrator is started, it will create the Durable Entity, then it will fetch the properties of this particular device.

```cs 
var entity = new EntityId(nameof(DeviceEntity), orchestratorArgs.DeviceId);

var offlineAfter = await ctx.CallEntityAsync<TimeSpan>(entity, "GetOfflineAfter");
var lastActivity = await ctx.CallEntityAsync<DateTime?>(entity, "GetLastMessageReceived");
```

Then the orchestrator will do a `WaitForExternalEvent` for a certain amount of time. The Durable Functions framework will 'kill' the thread. The orchestrator will be revived when this event is triggered or the timeout period has elapsed.  

```cs
try
{ 
    await ctx.WaitForExternalEvent("MessageReceived", offlineAfter); 
    log.LogInformation($"Message received for device {orchestratorArgs.DeviceId}, resetting timeout of {offlineAfter.TotalSeconds} seconds offline detection..."); 
    ctx.ContinueAsNew(orchestratorArgs);
    return;
}
catch (TimeoutException)
{
    await ctx.CallActivityAsync(nameof(SendStatusUpdate), new StatusUpdateArgs(orchestratorArgs.DeviceId, false));
    return;
}
```
When the offline detection timeout has been reached, a `TimeoutException` will be thrown, then we call the `SendStatusUpdate` activity function. When the `MessageReceived` event is raised, `ctx.ContinueAsNew` is called whichs will 'bring back' the orchestrator to it's next iteration/state. As long as the device is online, this orchestrator is considered to live forever.

### The Entity



## Yes, but have you tought about...

### CosmosDb


### Stream Analytics


### Durable Functions 1.x


## Performance

 
## Conclusion

