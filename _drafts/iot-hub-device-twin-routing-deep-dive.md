---
layout: post
title: Azure IoT Hub - Device Twin routing deep dive
tags: [azure, azure-iot]
img_dir: /assets/img/device-twin-routing
---

Azure IoT Hub message routing is a great feature that let you send messages to different endpoints based on rules, directly in the IoT Hub service. It's mostly used for routing telemetry messages, but can also send other kind of events, such as lifecyle or Device Twin events.  

The first time I used it for Device Twin change events, I could not find answers to the following questions:  
- Does it trigger an event for change on any section of the twin (reported, desired and tags) ?
- What is the content of the events payloads ?
- Is the condition on a route executed before or after the change ?

In this post I will do a deep dive into IoT Hub message routing on Device Twin change events to answer these questions.  
I am using az cli from bash prompt in WSL throughout this article, you can also use the Azure portal or Powershell cmdlets if you will.

## Creating the demo environment

### Configuring the IoT Hub
For this demo we will need an IoT Hub in the Free or Standard tier, we can't use the Basic tier as it does not support the Device Twin feature.  
Once the IoT Hub is created, in the *Message routing* pane we add a route to the built-in endpoint with data source set to *Device Twin Change Events*, like this in the portal:
![Route creation in portal]({{ page.img_dir}}/01-route-portal.png){: width="600" }  
Or with the following command in az cli:
```console
$ hub_name="<ENTER-HUB-NAME>"
$ az iot hub route create --hub-name $hub_name -n route-device-twin-changes --source twinchangeevents --endpoint-name events
```
It's a pretty simple setup: no custom endoint, no routing query (for the moment), a single route to send all twin changes to the same endpoint already present at the creation of the IoT Hub.  

### Creating a sample device
We need a device, so we create one with Sas authentication as it's always simpler to use for demos (we also store its connection string in a variable for later use): 
```console
$ hub_name="<ENTER-HUB-NAME>"
$ device_id="<ENTER-DEVICE-ID>"
$ az iot hub device-identity create --hub-name $hub_name -d $device_id
$ device_connection_string=$(az iot hub device-identity show-connection-string --hub-name $hub_name -d $device_id -o tsv)
```
To send device twin change events we will need two things:
1. For reported property changes, we must simulate the device so I have prepared a simple .NET console app available [here](https://github.com/xaviermignot/device-twin-routing-demo){: target="_blank" }. From the project folder, we run the following commands to store the device connection string and run the tool to initialize reported properties:
```console
$ dotnet user-secrets set "DeviceConnectionString" $device_connection_string
$ dotnet run
DeviceClient created ! Getting current twin...
First time with this device, let's initialize it !
Done ! Exiting now...
```
The first run of the tool creates two reported properties:
- `propertyUsedOnlyOnce` which will never be used again 😔
- `color` set to `Red`, next runs of the tool will change it to `Blue`, then back to `Red`, and so on...
2. For desired property and tag changes, we can use az cli with the `az iot hub device-twin update` command. Let's do this a first time to finish the initialization of the device:
```console
$ az iot hub device-twin update --hub-name $hub_name -d $device_id --set tags='{"tagProperty":"tagValue"}'
$   
```

Set-up is done, later to monitor the events flowing to the IoT Hub we will also use az cli with the `az iot hub monitor-events` command.

The following diagrams sums up the whole set-up:


## Testing changes on properties (desired/reported) and tags



### Test on reported properties


For each section, show if a change generates an event and what's in the payload


## Adding a condition on the route

Demonstrate that the condition is applied *after* the change


## What about message enrichments ?

Add a message enrichment and test if it's propagated in message properties