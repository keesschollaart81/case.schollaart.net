--- 
layout: post
title: ".NET Core 2.0 (ARM) on your Raspberry Pi"
author: "Kees Schollaart" 
backgroundUrl: /img/back4.jpg
comments: true 
---  

asdasdasd

asdasdasd

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

~~~ bash
# 'wget' the url you just copied
wget https://dotnetcli.blob.core.windows.net/dotnet/Runtime/master/dotnet-runtime-latest-linux-arm.tar.gz

mkdir /home/ubuntu/dotnet

tar -xvf dotnet-runtime-latest-linux-arm.tar.gz -C /home/ubuntu/dotnet 
~~~ 

After this, you should be able to run the dotnet command:

<a id="single_image" href="/img/2017/ssh.png" class="fancybox"><img src="/img/2017/ssh_thumb.png"/></a>

The CLI is not (yet) available on Linux/ARM, follow [this thread](https://github.com/dotnet/cli/issues/2556) for more info.

.NET Core on ARM is not fully supported and working, for more info, follow [this thread](https://github.com/dotnet/coreclr/issues/3977).

## Our test app

## Deploy and Run

