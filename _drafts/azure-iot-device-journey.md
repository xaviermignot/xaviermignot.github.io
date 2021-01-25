---
layout: post
title: My journey with Azure IoT devices
tags: [azure-iot]
img_dir: /assets/img/azure-iot-device-journey
image: /assets/img/azure-iot-device-journey/banner.jpg
---

If you want to start prototyping a project, or just start learning Azure IoT with a physical device, you have several options. Obviously you can simulate one on your machine, but it's much funnier to use a physical one, right ?  
Over the years I have accumulated several boards for various reasons (maybe I have a problem with it). In this post I will share my experience with all these devices and hopefully help you chose a device.  
Let's start with the most obvious choice, the mighty Raspberry Pi.


## Raspberry Pi boards
There is nothing I could write about Raspberry Pi that hasn't been written yet. I have been using them for many years for various purposes: media center, home automation box, network ad blocker, retrogaming, etc.  
On my first meetup as a speaker, I have presented a little project based on a traffic light made using a Raspberry Pi. It was running a .NET App to control the 3 lights using the pins of the Pi and electronic relays.  
![Raspberry Pi and relays]({{ page.img_dir }}/rpi-relay-small.jpg){: width="300" .normal }
![Raspberry Pi and relays]({{ page.img_dir }}/traffic-light.jpg){: width="400" .normal }  
There was also a Slack bot to interact with the device, and a integration with Twitter so that the audience could tweet and depending on the hashtag the light was changing.  
At first it was running an UWP app in Windows IoT Core, later I changed it as a .NET Core app running on Linux.  
Overall, using the Raspberry Pi as a starter is best choice to interact with Azure IoT, for the following reasons:
- you can choose you preferred language: Python, Javascript, .NET (not on Zero board), ...
- you'll find tons of documentation and inspiration on the web
- builtin support for bluetooth and wifi

Anyway, do I consider Raspberry Pis as IoT devices ? Well, not really, and people are often asking the difference between a Raspberry Pi and and an Arduino board. And the difference is huge !
As tiny as they are, Raspberry Pi boards are more computers that IoT devices:
- they're packed with a lot of power (for their size)
- they run a full operating system
- can perform many tasks in paralell: host a website, a DNS server, ...
- run using micro-SD card: lots of storage, but not very reliable

Microcontrollers (like Arduino boards) on the other end, are more focus on a single task:
- use less power (a battery pack or a USB connection to your laptop is enough)
- use a simpler OS that run *only* your code

Item|Raspberry Pis|Microcontrollers
--|--|--
Power supply|Require a 15W power supply|Can run from a battery pack or a USB port on your laptop
Storage|External microSD card with lots of space !|Onboard storage with less space but more reliable (you can unplug the device without worrying about corrupted storage issues)
OS|Run a full-featured Linux|Run RTOS that run *only* your code

## Azure MXChip IoT DevKit

## Wilderness Lab Meadow board

## Adafruit boards

## What's next ?

