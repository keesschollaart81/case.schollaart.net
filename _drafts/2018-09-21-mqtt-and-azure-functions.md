--- 
layout: post
title: "MQTT and Azure Functions"
author: "Kees Schollaart" 
backgroundUrl: /img/2018/back6.jpg
comments: true 
---  

I've build an Azure Functions extension which enables you to subscribe an publish to MQTT topics.

<!--more-->


<center>
<img src="/img/2018/mqtt_loves_azure_functions.png"/>
</center>

##  Why

For multiple reasons MQTT [gained a lot of popularity](https://trends.google.nl/trends/explore?date=today%205-y&q=%2Fm%2F0h5619c,%2Fm%2F09rsvzj,%2Fm%2F0dyn96) in the last couple of years. If you are interested in what MQTT is, I think [this blogpost explains it very well](https://devopedia.org/mqtt). MQTT is used in the IoT domain and [I started to use it as part of my Home Automation](/2018/05/26/serverless-ai-in-my-backyard.html).

The MQTT messaging patterns fit very well with the serverless patterns. So I wanted to run some serverless code in the cloud when something get published to a MQTT topic. It would also be cool to publish messages. 

## Azure Functions

Azure Functions has the notion of _input_ and _output_ bindings, you could see them as _triggers_ and _results_. There are a lot of [implementations](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings) available out of the box developed by Microsoft. There are (at this time) two major releases of the Azure Functions host and SDK being referred to as v1 and v2. In v2 it's possible to develop your own bindings.

So I build the MQTT trigger and output bindings for Azure Functions V2. It's open source and free to use, the code is available in my GitHub project [CaseOnline.Azure.WebJobs.Extensions.Mqtt](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/).

## How to use

In your Azure Functions project reference the [CaseOnline.Azure.WebJobs.Extensions.Mqtt NuGet package](https://www.nuget.org/packages/CaseOnline.Azure.WebJobs.Extensions.Mqtt/).

This is a simple example, receicing messages for topic ```my/topic/in``` and publishing messages on topic ```testtopic/out```.

``` csharp
public static class ExampleFunctions
{
    [FunctionName("SimpleFunction")]
    public static void SimpleFunction(
        [MqttTrigger("my/topic/in")] IMqttMessage message,
        [Mqtt] out IMqttMessage outMessage,
        ILogger logger)
    {
        var body = message.GetMessage();
        var bodyString = Encoding.UTF8.GetString(body);
        logger.LogInformation($"{DateTime.Now:g} Message for topic {message.Topic}: {bodyString}");
        outMessage = new MqttMessage("testtopic/out", new byte[] { }, MqttQualityOfServiceLevel.AtLeastOnce, true);
    }
}
```

But there is much more to say, checkout these pages in the Wiki:

* [Getting Started](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/wiki/Getting-started)
* [Publish via output](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/wiki/Publish-via-output)
* [Subscribe via trigger](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/wiki/Subscribe-via-trigger)
* [Integrate with Azure IoT Hub](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/wiki/Azure-IoT-Hub)
* [And more in the Wiki](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/wiki)