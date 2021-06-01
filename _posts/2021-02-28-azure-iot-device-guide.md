---
title: My guide for Azure IoT devices
tags: [azure-iot]
img_dir: /assets/img/azure-iot-device-guide
---

If you want to start prototyping a project, or just start learning Azure IoT with a physical device, you have several options. Obviously you can simulate one on your machine, but it's much funnier to use a physical one, right ?  
Over the years I have accumulated several boards for various reasons (maybe I have a problem with it).  
![Raspberry Pi 1]({{ page.img_dir }}/rpi-alone.jpg){: width="350" .normal }
![All my devices]({{ page.img_dir }}/all-devices.jpg){: width="350" .normal }
_How it started / how it's going_

In this post I will share my experience with all these devices and hopefully help you choose one. I will go through the devices I have used by chronological order, based on my experience.  
As a reminder/disclaimer I don't have any background in embedded development, I'm a backend developer who has developed an interest for IoT and therefore for coding on devices 🤗  


## Raspberry Pi boards

There is nothing I could write about Raspberry Pi that hasn't been written yet. I have been using them for many years for various purposes: media center, home automation box, network ad blocker, retrogaming, etc.  
On my first meetup as a speaker in 2016, I have presented with a colleague a little project based on a traffic light powered by a Raspberry Pi. It was running a .NET App to control the 3 lights using the pins of the Pi and electronic relays.  
![Raspberry Pi and relays]({{ page.img_dir }}/rpi-relay-small.jpg){: width="300" .normal }
![Raspberry Pi and relays]({{ page.img_dir }}/traffic-light.jpg){: width="400" .normal }
_An open view with the Pi and the relays / the traffic light once closed and light up_  
There was also a Slack bot to interact with the device, and a integration with Twitter so that the audience could tweet and depending on the hashtag the light was changing. It was a fun project to build.  
I can't recommend enough the Raspberry Pi to start with Azure IoT, or for physical computing in general, for the following reasons:
- you can choose you preferred language: Python, Javascript, .NET (except on Pi Zero), ...
- you'll find tons of documentation and inspiration on the web
- the devices have built-in support for bluetooth and wifi


## Tiny computers compared to microcontrollers

At this point I want to make a pause to answer a question I was often asked: *How can I choose between a Raspberry Pi and an Arduino board ?*  
Well, they are from different kind of devices, as Raspberry Pis are *tiny computers* (except the shiny new Pico) and Arduino is a *microcontroller* platform. Let's go through the specific aspects of both kinds. 

### Tiny computers
If we look at the features of a Raspberry Pi, we realize that it can handle most of the things people do with their computers:
- it powers up a full blown operating system with a graphical interface, desktop, file explorer and so on
- it comes with a preinstalled software like an internet browser, a text editor, a video player, etc.
- it can perform many tasks in parallel: host a website, act like a DNS server, a media-center, ...

It also require a minimum of power: even if it uses the same micro USB port as a smartphone, it requires more current so a dedicated, more powerful power supply than your smartphone's must be used for the Pi. If you need portability, using a usb power supply for instance, you should consider a Raspberry Pi Zero over a *full-size* model.   
For storage you have to provide a separated micro-SD card, as the Pi doesn't have onboard storage. So you can have plenty of space but it can also be corrupted, that's why you should always shutdown you Pi gracefully, and not unplug it like a maniac !

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
A Pi can also be used to perform AI on the edge, like running a custom vision model as shown in [this demo](https://docs.microsoft.com/en-us/samples/azure-samples/custom-vision-service-iot-edge-raspberry-pi/custom-vision-azure-iot/).

Let's finish this long pause and move on to the first microcontroller board I have used.


## Azure MXChip IoT DevKit

Around 2017 Microsoft has released the [Azure IoT DevKit](https://microsoft.github.io/azure-iot-developer-kit/) through a partnership with MXChip. Designed to help people to prototype projects with Azure IoT, it is packed with a lot of nice features:
- onboard sensors (temperature, humidity, magnetic, accelerometer, gyroscope, ...)
- other features on the board: small oled screen, headphone jack, microphone, leds ...
- wifi connectivity !
- extensibility through GPIO pins like on Microbit boards

As hardware is not everything, it comes with an SDK to interact with the sensors and the cloud, and extensions for Visual Studio Code for a well integrated development experience.  

Me and a colleague have used two of them for a talk in 2018, with another super useful demo: shaking one device triggered a call to an Azure Function who called a direct method on the other device to power up a PC fan to move polystyrene balls in a ball. Yep we did that, code is [here](https://github.com/neotechsolutions/dx2018-iot) if you're interested. 

![DevKit Blender]({{ page.img_dir }}/blender.gif)
_The blender project in action_

I have also built another traffic light program, as I was running out of stupid project ideas, and I needed a pet project to continue to use this DevKit. The code is available on my [GitHub](https://github.com/xaviermignot/traffic-light-mxchip-client), I enjoy working on it from time to time, but I have to admit that coding C/C++ can be hard sometime, especially when working with strings, JSON serialization, and sometimes the compiler throws an error out of nowhere that seem impossible to tackle.  

But overall I think this board is a great choice if you are focused on Azure IoT, except IoT Edge of course. Even if there is few updates on the dedicated website, over the last few months the board has support for IoT Plug and Play, and Azure RTOS, and is almost always mentioned in the tutorials in Azure IoT docs. You can do tons of things using this board, including TinyML as seen on [Benjamin Cabe's blog](https://blog.benjamin-cabe.com/2020/04/27/how-to-run-tensorflow-lite-on-the-mxchip-az3166-iot-devkit).  
Another option for microcontrollers with Azure IoT would be the Azure Sphere development board that I don't cover it as I have never used it.


## Wilderness Lab Meadow board

In 2019 I've bought a Meadow board from [Wilderness Labs](https://www.wildernesslabs.co/), after missing their Kickstarter campaign in 2018. The promise is to run a full .NET Standard on a microcontroller board, which was appealing to me as C# is my preferred language, and I had some issues with C on the MXChip.  
![Meadow upside]({{ page.img_dir }}/meadow-up.jpg){: width="350" .normal }
![Meadow bottom side]({{ page.img_dir }}/meadow-bottom.jpg){: width="350" .normal }
_The Meadow kit from above / and from below (look how sharp this wooden board looks 🤩)_

However I have barely used this board for the moment, I only went through the getting started guides and that's it. The development experience with Visual Studio "full" (not Code) was not that great, I had to reset/reflash my board all the time, it was pretty clunky so I put it aside for the moment.  
The project is still in beta, it requires lots of work so I don't mind, I will probably get back to it once VS Code support will be out (it's on the roadmap).  
The fact that I finally haven't used yet is totally on me, I do want to mention it though as I think it's great to have options to code in C# on microcontrollers.  
If you are really into .NET and don't want to use any other language than C# (I know some folks like that 😜), you should consider getting a Meadow. There are other options to explore like the .NET [nanoframework](https://docs.nanoframework.net/).  
These options seem targeted to professional users, they're more like "prototype you product before industrialization" than "build something for fun with your kids". 


## Adafruit boards

Recently I have discovered Adafruit boards, a huge family of microcontroller boards, including some of them running CircuitPython, an implementation of the Python language running on microcontrollers.  
I really like Python, although I need to practice with it more often, and I already knew Adafruit as a very reliable component manufacturer so I decided to give CircuitPython a shot.  
![Adafruit boards]({{ page.img_dir }}/adafruit.jpg){: width="350" }
_Trinket m0 on the breaboard (it's so tiny !), Feather m0 Express on the right (all on the homemade board inspired by Meadow's_

It turns out it's pretty great, I really like the development flow: plugging the device on my laptop USB port, editing a `code.py` file using VS Code, save it and *BAM !* the code is already running on the device.  
I can also use Python's *REPL* to enter individual lines of code and see what happens on the device, it's really helpful to figure out of things work.  
So after a few experiments on my Trinket M0 and Feather M0 Express (blinking a led, turning a servo, ...), I was like *"Hey, let's connect that thing to an IoT Hub !"*. And that's how silly me learned that not all development boards come with Wifi connectivity 🤦‍♂️  
Wifi connectivity is possible using an add-on like the [AirLift FeatherWing](https://www.adafruit.com/product/4264), or using another board like the [Metro M4 Express](https://learn.adafruit.com/adafruit-metro-m4-express-airlift-wifi).  
Anyway event if I don't do IoT with my Adafruit boards, I really like using them, it helps me to practice Python, and to improve my electronic knowledge. That's also what I like with Adafruit, their website is full of great tutorial to teach electronics, with content suitable for kids or for grown up noobs like me.


## A few words on CPX and Microbit

Speaking of learning for kids, I want to mention two last devices: Adafruit Circuit Playground Express and BBC Microbit. It's slightly off the post's topic as they are not communicating devices (Microbit has bluetooth though), but their great for introducing people to programming, especially youngsters.  
![Microbit and CPX]({{ page.img_dir }}/microbit-cpx.jpg){: width="350" }
_BBC Micro:Bit on the left, Adafruit Ciruit Playground Express (CPX) on the right_

Each platform website provides great content and project inspiration, and there is also a nice integration on Microsoft MakeCode to let people program the devices without coding, using a blocky interface. Here are the links:
- [BBC Microbit project page](https://microbit.org/projects/)
- [Adafruit CPX learn page](https://learn.adafruit.com/category/express)
- [Microsoft MakeCode](https://www.microsoft.com/en-us/makecode)


## So how to choose ?

Wrapping up what we have seen in this post, let's summarize everything in this table to help you choose:

| Device | Reasons to choose | Things to consider | Approx price |
|--------|-------------- ----|--------------------|--------------|
| Raspberry Pi model A/B | The best choice for starting out<br /> Great learning platform as well | Nothing to say | $35 |
| Raspberry Pi Zero W | Best choice for starting with portable projects | No support for C# unlike A/B models | $8 |
| Azure DevKit/MXChip | Take this one if you are really into Azure IoT <br /> Azure IoT certified, PnP ready, runs Azure RTOS | Don't be afraid of coding in C | $40 |
| Wilderness Labs Meadow | For C# enthusiasts <br /> For pro developers | No VS Code support yet <br /> Project in beta, some features & improvements to come  | $50 |
| Adafruit boards | Great electronics & code learning platform <br /> Code in CircuitPython or C (Arduino) <br /> For beginners and pro hackers | Few boards provide internet connectivity, <br />choose carefully for an IoT project | from $9 to $35 |
| Microbit & CPX | Great learning platform for kids, using code or blocks | Mostly for kids (of any age 😉) <br /> No connectivity beside BLE on the Microbit | from $15 to $25 |


## What's next ?

There are lots of devices to try out there (Wio Terminal from Seeed Studio, Pico, ...) but I have more than enough devices at the moment. I could make something with CircuitPython on a connected device (like the Metro), or use a BLE CircuitPython board as a child device and a Pi as a gateway for instance.  
I should also do some ML at the edge, whether it's on a Raspberry Pi with Azure IoT Edge, or on the MXChip with TinyML.  
Projects first, buying things next !