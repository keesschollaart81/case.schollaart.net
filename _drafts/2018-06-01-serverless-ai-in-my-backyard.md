--- 
layout: post
title: "Serverless AI in my backyard"
author: "Kees Schollaart" 
backgroundUrl: /img/back4.jpg
comments: true 
---  

Recently I added some cool stuff to my home automation platform using computer vision, serverless code and MQTT.

<!--more-->

## The 'problem'

When you have kids you know, they leave doors open, also the gate in the fence. This does not have to be a big problem, except when there are bike's unlocked in the backyard, we dont want them to be stolen! 

I have a Home Automation platform running but I dont want to equip my bikes with sensors. I also dont want to put sensors on this wooden gate in my fence. 

How can I monitor this situation and being alerted when this occurs?

<img src="/img/2018/the-problem.png"/>

## Solution overview

The solution consists of four major pieces:

- Home Assistant
 
  [Home Assistant](https://www.home-assistant.io/) is an open-source home automation platform running on Python 3. You can track and control all devices at home and automate control. 

- CustomVision.ai

  With [CustomVision.ai](https://customvision.ai) you can easily customize your own state-of-the-art computer vision models that fit perfectly with your unique use case. Just bring a few examples of labeled images and let Custom Vision do the hard work.

- Azure Functions

  Accelerate your development with an event-driven, serverless compute experience. Scale on demand and pay only for the resources you consume.

- Dafang Hacks

  This is the [custom firmware for the Xiaomi Dafang camera](https://github.com/EliasKotlyar/Xiaomi-Dafang-Hacks) which enables motion tracking, full HD camera and MQTT publishing.

The camera publishes messages to a MQTT topic when motion is detected. These 'motion-detected' messages are published on the MQTT Broker inside of Home Assistant. An Azure Function is subscribed to this topic, gets the current picture of the camera an pushes this iamge to the Custom Vision api. This Custom Vision is trained by me and knows what an open gate and a bike looks like. The result is parsed in the Azure Function and then published back to the MQTT broker. In Home Assistant a sensor is configured to listen to this result and with that, I can do all sort of cool stuff with these values inside of Home Assistant.

## The hardware

### The Home Assistant host

I want my Home Automation to work, also without Internet. Therefor I run Home Assistant using [Hassio](https://www.home-assistant.io/hassio/) on a [Raspberry Pi 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/).

### Xiaomi Dafang Camera

This camera is both cheap and feature rich. It's only ±30 euros at [GearBest](https://www.gearbest.com/ip-cameras/pp_693217.html) and [AliExpress](https://www.aliexpress.com/item/Xiaomi-Mijia-Xiaofang-Dafang-Smart-IP-Camera-110-Degree-1080p-FHD-Intelligent-Security-WIFI-IP-Cam/32839736818.html)

I've updated the camera's firmware to be able to use it in a more advanced manner. With the [Dafang Hacks Custom Firmware](https://github.com/EliasKotlyar/Xiaomi-Dafang-Hacks) there are lots of possibilities.

<!-- <img src="/img/2018/xiaomi-dafang.jpg"/> -->
<img src="/img/2018/xiaomi-dafang-motion-settings.png"/>

As you can see in the image above, its easy to setup a zone for motion detection. I dont want motion-events when a car drives by or when the wind blows the plants too much.

## Home Assistant

My home is automated using Home Assistant, it is configured using YAML files, my configuration is open, [check it out on GitHub](https://github.com/keesschollaart81/Home-Assistant-Configuration).

I use it all kind of stuff, for example, I turn on/off my [Yeelight Smart Led Bulb's](https://www.yeelight.com/en_US/product/wifi-led-w) using [Xiaomis wall switches](https://xiaomi-mi.com/sockets-and-sensors/aqara-smart-light-wall-switch-zigbee-version-double-key/) and Alexa. I measure all sorts of sensors, for example Eneco Toon, motion, humidity, etc. (mostly using [Xiaomi Home](https://xiaomi-mi.com/sockets-and-sensors/xiaomi-mi-smart-home-kit/)). We get a push notification when the washing machine is ready, etc. etc.

For this case, Home Assistant does four things:

- Acts as the MQTT broker
- Processes/persist the gate/bike status messages as sensor-value ([MQTT Sensor config](https://github.com/keesschollaart81/Home-Assistant-Configuration/blob/master/configuration.yaml#L71-L82)
- The UI for the camera feed and door/bike status ([Camera Dashboard Config](https://github.com/keesschollaart81/Home-Assistant-Configuration/blob/master/groups.yaml#L38-L58))
- Run the automations ([Example Automation](https://github.com/keesschollaart81/Home-Assistant-Configuration/blob/master/automations.yaml#L156-L185))

## MQTT & Azure Functions

When the camera detects motion, it publishes a message on a MQTT topic. The MQTT broker is hosted with my Home Assisant because I want the core features of my home automation platform to be offline-capable. For this feature however, this is not a requirement and thats why doing the detection in the cloud is fine, at this point in time.

I need of course something to subscribe to these messages. As a passionate Azure developer I immediatly thought of Azure Functions. Azure Function did not have a MQTT Trigger binding, so I wrote one. This will be another blogpost in the near future bu feel free to check it out in the GitHub reposity: ['Mqtt Bindings for Azure Functions'](https://github.com/keesschollaart81/CaseOnline.Azure.WebJobs.Extensions.Mqtt/).

This Azure Function now does five things:

- Receives the message when motion is detected
- Goes back to my camera to get the image
- Persists this image to Azure Storage for later reference
- Executes the trained Object Detection model (more on this in the next chapter)
- Publishes MQTT 'return' messages for the results (is the gate open or not, is there a bike visible?) 

All of this, I call my 'backend'. The code of the cloud backend for my home automation can be found on GitHub: '[Home-Assistant-Backend](https://github.com/keesschollaart81/Home-Assistant-Backend)'.

I did some extra stuff in this codebase but if you want, this could easily be written in less than 20 lines of code!

## CustomVision.ai

CustomVision.ai makes it really easy to train an AI model to detect objects in an image. First you give the engine some example images where you set what the objects look like. In my case I trained the service to detect the following 'tags':

 - Gate open
 - Gate closed
 - Barn door open
 - Barn door closed
 - Bike Jasmijn (kids bike)
 - Bike Marleen (wifes bike)

Training looks like this:

<img src="/img/2018/customvision-index.png"/>
<img src="/img/2018/customvision-train.png"/>

When each tag has 15 or more training images the model can be trained, the result of this training is a usable product.

<img src="/img/2018/customvision-performance.png"/>

Now we can use this trained model to detect objects in (new) images using their API. Because I've implemented the backend in C# and therfor I interact with this API using their SDK available as a NuGet package: ```[Microsoft.Azure.CognitiveServices.Vision.CustomVision.Prediction](https://www.nuget.org/packages/Microsoft.Azure.CognitiveServices.Vision.CustomVision.Prediction/)```. I used [step 6 of this HowTo](https://docs.microsoft.com/en-us/azure/cognitive-services/custom-vision-service/csharp-tutorial-od#step-6-get-and-use-the-default-prediction-endpoint) to get this working. The code calling this CustomVision API is as simple as [this](https://github.com/keesschollaart81/Home-Assistant-Backend/blob/master/src/Function/MotionService.cs#L98-L106):

``` csharp
var endpoint = new PredictionEndpoint() { ApiKey = _config.PredictionKey };
var predictionResult = await endpoint.PredictImageAsync(new Guid(_config.ProjectId), stream);

foreach (var c in predictionResult.Predictions)
{
    _log.LogDebug($"\t{c.TagName}: {c.Probability:P1} [ {c.BoundingBox.Left}, {c.BoundingBox.Top}, {c.BoundingBox.Width}, {c.BoundingBox.Height} ]");
}

var meaningfulPredictions = predictionResult.Predictions.Where(x => x.Probability > 0.15);
```

After images are being send to CustomVision.ai and processed this can be monitored. Each result can be turned into a new test sample in order to train the model even more.

At this point I use CustomVision.ai but I'll expect to move to the Azure Based computer vision PaaS service. CustomVision.ai is a good kickstarter because it has this easy to use UI to play with.

<img src="/img/2018/customvision-predictions.png"/>



## The result
 
WIP

## There is more

ARM
VSTS