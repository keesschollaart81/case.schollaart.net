--- 
layout: post
title: ".NET Core 2.0 (ARM) on your Raspberry Pi with Azure IoT Hub"
author: "Kees Schollaart" 
backgroundUrl: /img/back4.jpg
pageThemeColor: "#030059"  
---  

In this post I'll describe how you connect you're Raspberry Pi 3 to Azure IoT Hub using .NET Core.

<!--more-->

## Equip your Pi with Ubuntu

First we have to obtain an OS image to flash the SD card with. I used Ubuntu Classic Server 16.04 for Raspberry Pi 3 downloaded from [https://ubuntu-pi-flavour-maker.org/](https://ubuntu-pi-flavour-maker.org/download/):

 <a id="single_image" href="/img/2017/ubuntupi.png" class="fancybox"><img src="/img/2017/ubuntupi_thumb.png"/></a>

To flash the image file to the SD Card I used [Etcher](https://etcher.io/), available for OSx, Linux and Windows.

 <a id="single_image" href="/img/2017/etcher.png" class="fancybox"><img src="/img/2017/etcher_thumb.png"/></a>

First we have to update Ubuntu and then we have to install the dependecies of the .NET Core framework. Now SSH into your Raspberry Pi 3 and execute te following commands. 

~~~ bash
sudo apt-get -y update

sudo apt-get install libunwind8 libunwind8-dev gettext libicu-dev liblttng-ust-dev libcurl4-openssl-dev libssl-dev uuid-dev unzip
~~~  

## Install .NET Core

To install .NET Core, we have to get the right version of the framework, we're looking for a Linux and ARM compatible one. At the time I'm writing this, the 2.x version of .NET core is in the Master branch of the 'core-setup' repository:

[https://github.com/dotnet/core-setup/tree/master](https://github.com/dotnet/core-setup/tree/master)

Go to the CoreFx github repository and search for the .tar.gz file under 'Linux (armhf) (for glibc based OS)'. Copy the URL of this file.

 <a id="single_image" href="/img/2017/targz.png" class="fancybox"><img src="/img/2017/targz_thumb.png"/></a>

There's also an Ubuntu 16.04 build but that one is targetted for x64 architecture (instead of ARM). 

Now into our SSH session, 'wget' this package using the URL we just copied:

~~~ bash 
wget https://dotnetcli.blob.core.windows.net/dotnet/Runtime/master/dotnet-runtime-latest-linux-arm.tar.gz

mkdir /home/ubuntu/dotnet

tar -xvf dotnet-runtime-latest-linux-arm.tar.gz -C /home/ubuntu/dotnet 
~~~ 

After this, you should be able to run the dotnet command:

<a id="single_image" href="/img/2017/ssh.png" class="fancybox"><img src="/img/2017/ssh_thumb.png"/></a>

The CLI is not (yet) available on Linux/ARM, follow [this thread](https://github.com/dotnet/cli/issues/2556) for more info.

.NET Core on ARM is not fully supported and working, for more info, follow [this thread](https://github.com/dotnet/coreclr/issues/3977).

## Our test app

In order to test if this works I've created a test app a little bit more complex than just <i>Hello World</i>. This app uses the Azure IoT Hub Device SDK and listens to messages which involves serialization, AMQP, security etc.

The test app can be found in this GitHub repository:
[https://github.com/keesschollaart81/DotNetCoreIotHubDeviceClientExample](https://github.com/keesschollaart81/DotNetCoreIotHubDeviceClientExample)

Let highlight some parts which make this work:

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType> 
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <RuntimeFrameworkVersion>2.0.0-*</RuntimeFrameworkVersion>
    <RuntimeIdentifiers>win8-arm;ubuntu.14.04-arm;ubuntu.16.04-arm</RuntimeIdentifiers>
</PropertyGroup> 
 * snip *
</Project> 
```

First thing to notice is the target framework, we're specifically targetting netcoreapp2.0, this version of the framework is needed on the development machine and also on the Linux distro (which I just showed).

Also notice the RuntimeIdentifiers. This element needs to be added and tells the compiler to target a specific runtime. Because this is targetting specifically ARM Based runtimes, it's not possible to run this build on X86 runtimes.

## Deploy and Run

In order to run the application it needs to be published using ```dotnet publish  -r ubuntu.16.04-arm```. After that go to the ```/DotNetCoreIotHubDeviceClientExample/bin/Debug/netcoreapp2.0/ubuntu.16.04-arm/publish/``` folder.

From within this folder run ```scp -pr . ubuntu@192.168.2.22:/home/ubuntu/testapp/``` to copy all files in  this publish folder to a 'testapp' folder on Ubuntu. 

Now SSH into Ubuntu and navigate to the ```/home/ubuntu/testapp/``` folder. Inside this folder run ```dotnet ./CoreIotHubClient.dll```

<a id="single_image" href="/img/2017/run.png" class="fancybox"><img src="/img/2017/run_thumb.png"/></a>
