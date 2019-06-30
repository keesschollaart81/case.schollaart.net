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




## Yes, but have you tought about...
 
### Stream Analytics


### Durable Functions 1.x

