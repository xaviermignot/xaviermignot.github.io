---
layout: post
title: My journey with Azure IoT devices
tags: [azure-iot]
img_dir: /assets/img/azure-iot-device-journey
image: /assets/img/azure-iot-device-journey/banner.jpg
---

If you want to start prototyping a project, or just start learning Azure IoT with a physical device, you have several options. Obviously you can simulate one on your machine, but it's much funnier to use a physical one, right ?  
Over the years I have accumulated several boards for various reasons (maybe I have a problem with it). In this post I will share my experience with all these devices and hopefully help you one.  
I will go through the devices I have used by chronological order, based on my experience.  
As a reminder/disclaimer I don't have any background in embedded development, I'm a backend developer who has developed an interest for IoT and therefore for coding on devices 🤗  


## Raspberry Pi boards

There is nothing I could write about Raspberry Pi that hasn't been written yet. I have been using them for many years for various purposes: media center, home automation box, network ad blocker, retrogaming, etc.  
On my first meetup as a speaker in 2016, I have presented with a colleague a little project based on a traffic light powered by a Raspberry Pi. It was running a .NET App to control the 3 lights using the pins of the Pi and electronic relays.  
![Raspberry Pi and relays]({{ page.img_dir }}/rpi-relay-small.jpg){: width="300" .normal }
![Raspberry Pi and relays]({{ page.img_dir }}/traffic-light.jpg){: width="400" .normal }  
There was also a Slack bot to interact with the device, and a integration with Twitter so that the audience could tweet and depending on the hashtag the light was changing. It was a fun project to build.  
I can't recommend enough the Raspberry Pi to start with Azure IoT, or for physical computing in general, for the following reasons:
- you can choose you preferred language: Python, Javascript, .NET (not on Zero board), ...
- you'll find tons of documentation and inspiration on the web
- built-in support for bluetooth and wifi


## Tiny computers compared to microcontrollers

At this point I want to make a pause to answer a question I was often asked: *How can I choose between a Raspberry Pi and an Arduino board ?*  
Well, they are from different kind of devices, as Raspberry Pis are *tiny computers* (except the shiny new Pico) and Arduino is a *microcontroller* platform.  

### Tiny computers
If we look at the features of a Raspberry Pi, we realize that it can handle most of the things people do with their computers:
- it powers up a full blown operating system with a graphical interface, desktop, file explorer and so on
- it comes with a preinstalled with software like an internet browser, text editor, video player, etc.
- it can perform many tasks in parallel: host a website, act like a DNS server, a media-center, ...

It also require a minimum of power: even if it uses the same micro USB port as a smartphone, it requires more current so a dedicated, more powerful power supply than your smartphone's must be used for the Pi. If you need portability, using a usb power supply for instance, you should consider a Raspberry Pi Zero over a *full-size* model.   
For storage you have to provide a separated micro-SD card, as the Pi doesn't have onboard storage. So you can have plenty of space but this can also be corrupted, that's why you should always shutdown you Pi gracefully, and not unplug it like a maniac !

### Microcontrollers
On the other hand we have microcontroller-based devices, like Arduino boards, who differ from tiny computers because:
- they *use* less power, you can plug them to a USB port of your laptop, use a battery pack and they'll boot up
- they also *have* less power, I mean less compute capability, less memory, less storage, etc.
- they use a simpler OS than computers, that will run your code very quickly once booted
- they're also more robust, you can unplug and plug them anytime without any issue

### Other differences
So we have seen that tiny computers and microcontrollers have a huge difference in power capability and usage, but that's not the only difference.  
In terms of programming language, tiny computers support almost any language: Python, Js, C#/.NET, etc.  
Microcontrollers are more restricted, C based languages (C, C++, Arduino C) are the kings of microcontrollers. But there are some implementation of Python that run on microcontrollers like MicroPython or CircuitPython (more on that later).

In terms of connectivity, almost all Raspberry Pis come with Wifi and Bluetooth built-in support, whereas lots of microcontroller-base boards need an additional component to connect to a network.

### Raspberry Pi, IoT device or not ? 
Overall, shall we consider tiny computers like Raspberry Pis as IoT devices or not ?  
I would say it depend on how you use them, a Raspberry Pi can be used as a desktop computer, or as a small server, in this case it will not be an IoT device.  
But a Pi can also be connected to sensors, and act as a gateway to push telemetry to the cloud, it can also run Azure IoT Edge, so of course in this case it would be an IoT device (a beefier IoT device than a microcontroller, so to speak).

Let's finish this long pause and move on to the first microcontroller board I have used.


## Azure MXChip IoT DevKit

Around 2017 Microsoft has released the Azure IoT DevKit through a partnership with MXChip. Designed to help people to prototype projects with Azure IoT, it is packed with a lot of nice features:
- onboard sensors (temperature, humidity, magnetic, accelerometer, gyroscope, ...)
- other features on the board: small oled screen, headphone jack, microphone, leds ...
- wifi connectivity !
- extensibility through GPIO pins like on Microbit boards
As hardware is not everything, it comes with an SDK to interact with the sensors and the cloud, and extensions for Visual Studio Code for a well integrated development experience.  

Me and a colleague have used two of them for a talk in 2018, with another super useful demo: shaking one device triggered a call to an Azure Function who called a direct method on the other device to power up a PC fan to move polystyrene balls in a ball. Yep we did that, code is [here](https://github.com/neotechsolutions/dx2018-iot) if you're interested. 

I have also built another traffic light program, as I was running out of stupid project ideas, and I needed a pet project to continue to use this DevKit. The code is available on my [GitHub](https://github.com/xaviermignot/traffic-light-mxchip-client), I enjoy working on it from time to time, but I have to admit that coding C/C++ can give hard times, especially when working with strings, JSON serialization, and sometimes the compiler throws an error out of nowhere that seem impossible to tackle.  
Anyway, I still have things to do with that board, I should try Azure RTOS for instance, but at one time I needed to move back to a language I know more, that's when I heard about a microcontroller board that run C#.


## Wilderness Lab Meadow board

Why ? I was having troubles with C, so C# on a MCU was more than appealing
Pros:
- Runs C#, on a MCU
Cons:
- VS Code support not ready yet
- Experience is still clunky: needed to reconnect, flash it again, ... (need to retry in .NET 5)


## Adafruit boards

Why ? I like Python so writing it on a device seemed great
Pros:
- Cheap
- The company is great, CircuitPython is very active, library support is amazing
- The documentation is great
- It's very straightforward, you plug the device, edit the code, and it's already deployed
- The REPL is very handy to troubleshoot, try things quickly, or understand things
Cons:
- Most of the boards are not ready for IoT: no WiFi or event BLE
- As the dev workflow is handy, if you want to host your code on a repo it's not the best


## So how to choose ?
It depends on what you want to do, and where your are in your learning journey:
- You are just starting and have no specific idea for the moment ? Use a Raspberry Pi model A/B
- You are starting and you want to build something wireless or portable ? Use Raspberry Pi Zero W


## What's next ?

There are lots of devices to try out there (Wio Terminal from Seeed Studio, Pico, ...) but I have enough devices at the moment. I could make something with CircuitPython on a connected device (like the Metro), or use a BLE CircuitPython board as a child device and a Pi as a gateway for instance.  
Projects first, buying things next !

