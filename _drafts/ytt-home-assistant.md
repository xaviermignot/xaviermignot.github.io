---
title: Simplifying Home Assistant's configuration with ytt
description: How you can use templating for your Home Assistant configuration with ytt
---

I have been a light user of Home Assistant to automate my home for a few years, and have recently lost all my configuration (I could blame Raspberry Pi's SD card usage for that, but I should have anticipated that with backups...).  
This was a good opportunity to reconfigure everything "as code" (using YAML files instead of the GUI), and in this post I will show how I have simplified my configuration using ytt.

## What is ytt ?
Ytt stands for "YAML Templating Tool", it adds templating syntax to YAML files and can be used in many "YAML-oriented" scenarios: K8S configuration, CI/CD pipelines authoring, etc.  
It's part of the Carvel set of tools, which is a CNCF Sandbox project.  
The best way to start with ytt is the official documentation [here](https://carvel.dev/ytt/), but in a nutshell it uses a specific comment syntax starting with `#@` to turn any YAML file into a template:
```yaml
#@ for room in ['Bedroom', 'Living room', 'Bathroom']:
- platform: generic_thermostat
  name: #@ "{} thermostat".format(room)
#@ end
```
This (uncompleted) snippet will for instance create 3 thermostats with distincts names: `Bedroom thermostat`, `Living room thermostat` and `Bathroom thermostat`.  
Ytt has many features and can be used in many ways. If you are new to ytt, the best way to start learning is the [interactive playground](https://carvel.dev/ytt/#example:example-plain-yaml) to gradually go through its key concepts.

## My Home Assistant setup

> I assume that if you are still reading this you are familiar with Home Assistant but if it's not the case, you should start [here](https://www.home-assistant.io/) or [here](https://www.home-assistant.io/installation/).
{: .prompt-tip }

The only automated things in my home are electric heaters with "pilot wires", each room contains the following physical devices:
- A heater with a pilot wire
- A on/off module connected to the pilot wire
- A temperature/humidity sensor

The sensor and the on/off module communicate with Home Assistant using the RFXCOM integration. I'm currently planning to move to another platform but that's not important here.  

In Home Assistant, I just need to regulate the temperature according to a different schedule for each room, so I have:
- A [generic thermostat](https://www.home-assistant.io/integrations/generic_thermostat/) with presets (eco, comfort, away, ...) for each room
- A [schedule](https://www.home-assistant.io/integrations/schedule/) to define when each thermostat should use the comfort preset
- [Automations](https://www.home-assistant.io/docs/automation/) to actually change the thermostat's presets according to the schedules

Your installation is probably different but that doesn't matter, what's interesting here is how we can use ytt to simplifying editing Home Assistant's YAML configuration files.

> The YAML snippets in this post are just samples based on my usage, not the real files I am using for my installation (those are stored in a private repo).
{: .prompt-info }

## How ytt can help

The main problem in my setup is that each room has the same devices but a proper configuration: the temperature schedule is not the same for my kid's room or my home office. So some settings are shared, others are specific.  
