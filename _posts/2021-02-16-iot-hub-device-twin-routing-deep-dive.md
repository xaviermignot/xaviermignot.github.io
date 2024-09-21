---
title: Azure IoT Hub - Routing device twin changes deep dive
tags: [azure, azure-iot]
media_subpath: /assets/img/device-twin-routing
---

Azure IoT Hub message routing is a great feature that let you send messages to different endpoints based on rules, directly in the IoT Hub service. It's mostly used for routing telemetry messages, but can also send other kind of events, such as lifecyle or device twin change events.  

The first time I used it for device twin change events, I could not find answers to the following questions in the docs:  
- Does it trigger an event for change on any section of the twin (reported, desired and tags) ?
- What is the content of the events payloads ?
- Is the condition on a route executed before or after the change ?

In this post I will do a deep dive into IoT Hub message routing on device twin change events to answer these questions.  
If you're not familiar with the concepts of message routing and device twin, I encourage you to read the following content first: 
- [Message routing for telemetry](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-d2c)
- [Device twin presentation](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins)

*TL;DR* If you're in a hurry and need the answers right now you can jump to the [summary](#summary) section of this post, otherwise I'll appreciate if you read the whole thing 🤗  


## Creating the demo environment

### Overview
For this demo I will use the following components:
- In azure, an IoT Hub as the single resource
- On my laptop, Windows Terminal with a few WSL tabs to:
  - Update the device twin using az cli commands
  - Run a .NET 5 console app to simulate a device
  - Monitor the events from the IoT Hub using other az cli commands

I have chosen to use only az cli from bash as I want to get more comfortable with it, but you can also use the Azure portal to configure the IoT Hub and update the desired properties and tags section of the twin, or Powershell cmdlets. The [Azure IoT Hub](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-toolkit) extension for VS Code is also great to update the twin and monitor events.

The following diagrams shows the whole set-up:
![Diagram](/diagram.webp)

### Configuring the IoT Hub
We need an IoT Hub in the Free or Standard tier, we can't use the Basic tier as it does not support the device twin feature.  
Once the IoT Hub is created, we add a route to the built-in endpoint with data source set to *Device Twin Change Events* with the following command in az cli:
```console
$ hub_name="<ENTER-HUB-NAME>"
$ az iot hub route create --hub-name $hub_name -n route-device-twin-changes --source twinchangeevents --endpoint-name events
```

### Creating a sample device
We need a device, so we create one with Sas authentication as it's always simpler to use for demos (we also store its connection string in a variable for later use): 
```console
$ device_id="<ENTER-DEVICE-ID>"
$ az iot hub device-identity create --hub-name $hub_name -d $device_id
$ device_connection_string=$(az iot hub device-identity show-connection-string --hub-name $hub_name -d $device_id -o tsv)
```
To send device twin change events we will need two things:
1. For reported property changes, we must simulate the device so I have prepared a simple .NET console app available [here](https://github.com/xaviermignot/device-twin-routing-demo). From the project folder, we run the following commands to securely store the device connection string and run the tool to initialize reported properties:
```console
$ dotnet user-secrets set "DeviceConnectionString" $device_connection_string
$ dotnet run
DeviceClient created ! Getting current twin...
First time with this device, let's initialize it !
Done ! Exiting now...
```
The first run of the tool creates two reported properties:
- `propertyUsedOnlyOnce` which will never be used again 😔
- `color` set to `Red`, next runs of the tool will change it to `Blue`, then back to `Red`, then back to `Blue`, and so on...
2. For desired property and tag changes, we can use az cli with the `az iot hub device-twin update` command. Let's do this a first time to finish the initialization of the device:
```console
$ az iot hub device-twin update --hub-name $hub_name -d $device_id --set tags='{"tagProperty":"tagValue"}'
$ az iot hub device-twin update --hub-name $hub_name -d $device_id --set properties.desired.desiredProperty='desiredValue'
```
This creates properties in the desired and tags section of the device twin that will not be changed by the upcoming tests. We will see if these properties will be embedded in event payloads or not.

Set-up is done, let's jump to the first test !


## Testing changes on properties (desired/reported) and tags

For each test I am using Windows Terminal with two panes open in WSL: one on the lef to make the twin change, one on the right to monitor the changes (using the `az iot hub monitor-events` command):  
![Reported test in Windows Terminal](/test-reported.webp)

### Test on reported properties
As seen in the image above, runing the device tool in the left pane changes the `color` reported property from `Red` to `Blue`. In the right pane, we see an event with the following payload:
```json
{
    "version": 10,
    "properties": {
        "reported": {
            "color": "Blue",
            "$metadata": {
                "$lastUpdated": "2021-02-13T17:56:23.2704925Z",
                "color": {
                    "$lastUpdated": "2021-02-13T17:56:23.2704925Z"
                }
            },
            "$version": 7
        }
    }
}
```
As a comparison, here is the whole reported section of the device twin:
```json
"reported": {
    "propertyUsedOnlyOnce": "This is set only once",
    "color": "Blue",
    "$metadata": {
        "$lastUpdated": "2021-02-13T17:56:23.2704925Z",
        "propertyUsedOnlyOnce": {
            "$lastUpdated": "2021-02-13T16:07:58.6609507Z"
        },
        "color": {
            "$lastUpdated": "2021-02-13T17:56:23.2704925Z"
        }
    },
    "$version": 7
}
```
What's interesting is that the `propertyUsedOnlyOnce` property is *not* present in the event payload: the payload contains only the *patch* sent by the device.  
To summarize, in the event payload we have:
- The `version` of the whole device twin
- The new values of the properties included in the change
- The `$metadata` of the properties in the change
- The `$version` of the reported property section

And we don't have:
- The reported properties not included in the change (and their `$metadata`)
- Any data from the other section of the device twin (tags, desired)

### Test on desired properties
Back in my Windows Terminal, on the left pane I enter the following command to add a new property in the desired section:
```console
az iot hub device-twin update --hub-name $hub_name -d $device_id --set properties.desired.desiredColor='Orange'
```
In the right pane I receive an event with the following payload:
```json
{
    "version": 11,
    "tags": {
        "tagProperty": "tagValue"
    },
    "properties": {
        "desired": {
            "desiredProperty": "desiredValue",
            "desiredColor": "Orange",
            "$metadata": {
                "$lastUpdated": "2021-02-14T18:05:38.5825801Z",
                "$lastUpdatedVersion": 3,
                "desiredProperty": {
                    "$lastUpdated": "2021-02-14T18:05:38.5825801Z",
                    "$lastUpdatedVersion": 3
                },
                "desiredColor": {
                    "$lastUpdated": "2021-02-14T18:05:38.5825801Z",
                    "$lastUpdatedVersion": 3
                }
            },
            "$version": 3
        }
    }
}
```
We have more content in this payload:
- The `version` of the whole device twin
- The tags section
- The whole desired properties section (not only the one that has changed), with the full `$metadata` and the `$version`
Only the reported section and the content of the twin managed by the IoT Hub (deviceId, connectionState, ...) are missing, otherwise the payload contains everything that can be modified from the cloud side.

### Test on tags
Let's do the same test but by adding a new property in the tags sections with the following command:
```console
az iot hub device-twin update --hub-name $hub_name -d $device_id --set tags.newTagProperty='newTagValue'
```
It generates an event with the following payload:
```json
{
    "version": 12,
    "tags": {
        "tagProperty": "tagValue",
        "newTagProperty": "newTagValue"
    },
    "properties": {
        "desired": {
            "desiredProperty": "desiredValue",
            "desiredColor": "Orange",
            "$metadata": {
                "$lastUpdated": "2021-02-14T19:53:14.4410421Z",
                "$lastUpdatedVersion": 4,
                "desiredProperty": {
                    "$lastUpdated": "2021-02-14T19:53:14.4410421Z",
                    "$lastUpdatedVersion": 4
                },
                "desiredColor": {
                    "$lastUpdated": "2021-02-14T19:53:14.4410421Z",
                    "$lastUpdatedVersion": 4
                }
            },
            "$version": 4
        }
    }
}
```
Basically we have the same event as for updating the desired properties: all the tags and all the desired properties with the `$metadata`.


## Adding a condition on the route

Now that we know that an event is triggered for each section of the twin, and what's in the payload, we can move to the third question: if we add a condition on the route, will this condition be evaluated *before* or *after* the change ?  
Let's update the route to figure this out:
```console
az iot hub route update --hub-name $hub_name -n route-device-twin-changes --condition '$twin.properties.reported.color = "Red"
'
```
With this condition, now only events from device with reported color set to `Red` will be sent to the endpoint.  
Using the dotnet tool again to simulate the device, whose reported color is `Blue`, the input says that the reported color has turned to `Red`:
```console
$ dotnet run
DeviceClient created ! Getting current twin...
Changing 'color' reported property from 'Blue' to 'Red'...
Done ! Exiting now...
```
In the monitoring pane, there is an event flowing:
```json
{
    "version": 13,
    "properties": {
        "reported": {
            "color": "Red",
            "$metadata": {
                "$lastUpdated": "2021-02-14T22:50:11.6725239Z",
                "color": {
                    "$lastUpdated": "2021-02-14T22:50:11.6725239Z"
                }
            },
            "$version": 8
        }
    }
}
```
Running the dotnet tool again, the reported color turns back to `Blue`:
```console
$ dotnet run
DeviceClient created ! Getting current twin...
Changing 'color' reported property from 'Red' to 'Blue'...
Done ! Exiting now...
```
In the monitoring pane, no event is flowing.  
So the answer is clear: first the change is made on the device twin, then the routing condition is evaluated, on the updated version of the device twin.


## Summary

To finish this post, let's sum up what we have learnt with this table who answers the two first questions from the intro:

Updated twin section | Event triggered ? | Payload of event
-------------------- | ----------------- | ---------------
Reported property    | yes | Part of the reported section updated (the *patch* sent by the device)
Desired property     | yes | All the tags and desired properties
Tags                 | yes | Same content as desired property updates

Lastly, we have seen that if there is a condition on the route, it will be evaluated *after* the application of the device twin change.  
I hope this post was helpful, I you have any question don't hesitate to reach out on [Twitter](https://twitter.com/_xavierm), or leave a comment below if you are reading this post from [DEV.to](https://dev.to/xaviermignot).