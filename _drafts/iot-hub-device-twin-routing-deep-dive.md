---
layout: post
title: Azure IoT Hub - Device Twin routing deep dive
tags: [azure, azure-iot]
---

Azure IoT Hub message routing is a great feature that let you send messages to different endpoints based on rules, directly in the IoT Hub service. It's mostly used for routing telemetry messages, but can also send other kind of events, such as lifecyle or Device Twin events.  

The first time I used it for Device Twin change events, I could not find answers to the following questions:  
- Does it trigger an event for change on any section of the twin (reported, desired and tags) ?
- What is the content of the events payloads ?
- Is the condition on a route executed before or after the change ?

In this post I will do a deep dive into IoT Hub message routing on Device Twin change events to answer these questions.


## The test protocol

Presentation of the solution to test:
- Link to repo for console app to simulate device
- Commands in az cli to monitor events
- IoT Hub configuration
- Small Visio diagram ?


## Testing changes on properties (desired/reported) and tags

For each section, show if a change generates an event and what's in the payload


## Adding a condition on the route

Demonstrate that the condition is applied *after* the change