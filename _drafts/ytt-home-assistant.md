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
The main problem in my setup is that each room has the same devices but a proper configuration: the temperature schedule is not the same in my kid's room and in my home office. So some settings are shared, others are specific.  
Let's see how ytt will help to deal with this situation.

### DRY: Writing required YAML only
The only YAML I need to edit to adjust my home automation settings is a file that lists all the rooms in my house:
```yaml
rooms:
- name: living_room
  display_name: Living room
  weekend_confort_start: "08:45:00"
  weekend_confort_end: "22:00:00"
- name: home_office
  display_name: Home office
  eco_temp: 16
  comfort_temp: 18.5
  weekday_confort_am_start: "07:45:00"
  weekday_confort_pm_end: "18:00:00"
- name: bathroom
  display_name: Bathroom
  comfort_temp: 20
  weekday_confort_pm_end: "07:30:00"
- name: bedroom
  display_name: Bedroom
  eco_temp: 15
  comfort_temp: 18
```
{: file="values/rooms.yaml" }

It uses the ytt concept of [data values](https://carvel.dev/ytt/docs/latest/how-to-use-data-values/), ie variables that can be referenced in templates.

### Defining a schema for the rooms
In the previous files, you'll notice that some properties are not declared for each room. There is indeed a [schema](https://carvel.dev/ytt/docs/latest/how-to-write-schema/) document that defines the properties, sets default values, and if properties are mandatory or not:
```yaml
#@data/values-schema
---
rooms:
#@schema/validation min_len=1
- name: ""
  #@schema/validation min_len=1
  display_name: ""
  min_temp: 14
  max_temp: 21
  away_temp: 15
  comfort_temp: 19.0
  eco_temp: 17
  keep_alive: "00:00:30"
  weekday_confort_am_start: "06:30:00"
  #@schema/nullable
  weekday_confort_am_end: "09:00:00"
  #@schema/nullable
  weekday_confort_pm_start: "17:00:00"
  weekday_confort_pm_end: "22:00:00"
  #@schema/nullable
  weekend_confort_start: "08:45:00"
  #@schema/nullable
  weekend_confort_end: "22:00:00"
```
{: file = "config/schema.yaml" }

This schema uses just a few features of ytt's schema validations: mark some properties as required using `schema/validation min_len` and others as nullable using `schema/nullable`. [This page](https://carvel.dev/ytt/docs/latest/schema-validations-cheat-sheet/) in the docs is a great reference for the other schema validation possibilities.

### Using the data and schema to generate Home Assistant's objects
This is where the fun begins, let's put the schema and values in action, starting with the generation of thermostats:
```yaml
#@ load("@ytt:data", "data")
---
#@ for room in data.values.rooms:
- platform: generic_thermostat
  unique_id: #@ "thermostat_" +  room.name
  name: #@ "Thermostat in " + room.display_name)
  heater: #@ "switch.heater_" + room.name
  target_sensor: #@ "sensor.temp_" + room.name
  min_temp: #@ room.min_temp 
  max_temp: #@ room.max_temp
  away_temp: #@ room.away_temp
  comfort_temp: #@ room.comfort_temp
  eco_temp: #@ room.eco_temp
  keep_alive: #@ room.keep_alive
#@ end
```
{: file="config/climate.yaml" }
This files uses a `for` loop to iterate through the rooms and declare for each room an object following the YAML schema of the `generic_thermostat` [integration](https://www.home-assistant.io/integrations/generic_thermostat/#yaml-configuration) of Home Assistant.  

Moving on to the schedules, the fun continues with `for` loops for week days and weekends inside a loop over the rooms:
```yaml
#@ load("@ytt:data", "data")
---
#@ for room in data.values.rooms:
#@yaml/text-templated-strings
'schedule_thermostat_(@= room.name @)':
  name: #@ "Schedule for thermostat in " + room.display_name
  #@ for day in ['monday', 'tuesday', 'wednesday', 'thursday', 'friday']:
  (@= day @):
    #@ if room.weekday_confort_am_end != None:
    - from: #@ room.weekday_confort_am_start
      to: #@ room.weekday_confort_am_end
    - from: #@ room.weekday_confort_pm_start
      to: #@ room.weekday_confort_pm_end
    #@ else:
    - from: #@ room.weekday_confort_am_start
      to: #@ room.weekday_confort_pm_end
    #@ end
  #@ end
  #@ if room.weekend_confort_start != None:
  #@ for day in ['saturday', 'sunday']:
  (@= day @):
    - from: #@ room.weekend_confort_start
      to: #@ room.weekend_confort_end
  #@ end
  #@ end
#@ end
```
{: file="config/schedules.yaml" }

A few things to mention in the above file:
- The `#@yaml/text-templated-strings` line followed by `'schedule_thermostat_(@= room.name @)':` brings the use of ytt's [text templating](https://carvel.dev/ytt/docs/latest/ytt-text-templating/) feature. It allows templating for property names, which is not possible using the comment-based (`#@`) syntax. The feature is also used on lines 8 and 21 (`(@= day @)`) to use the days names as properties.
- The `#@ if <condition> != None:` lines shows how nullable values are handled in templates. Note that nullable properties are different from properties with default values

Lastly this the template file for automations:
```yaml
#@ load("@ytt:data", "data")
---
#@ for room in data.values.rooms:
- id: #@ "heating_{}_on".format(room.name)
  alias: #@ "Heating on in room {}".format(room.display_name)
  description: #@ "Switch to comfort preset in room {}".format(room.display_name)
  triggers:
      - trigger: state
        entity_id:
          - #@ "schedule.schedule_thermostat_{}".format(room.name)
        to: "on"
  actions:
    - action: climate.set_preset_mode
      target:
        entity_id: #@ "climate.thermostat_{}".format(room.name)
      data:
        preset_mode: comfort
- id: #@ "heating_{}_off".format(room.name)
  alias: #@ "Heating off in room {}".format(room.display_name)
  description: #@ "Switch to eco preset in room {}".format(room.display_name)
  triggers:
      - trigger: state
        entity_id:
          - #@ "schedule.schedule_thermostat_{}".format(room.name)
        to: "off"
  actions:
    - action: climate.set_preset_mode
      target:
        entity_id: #@ "climate.thermostat_{}".format(room.name)
      data:
        preset_mode: eco
#@ end
```
{: file="config/automations.yaml" }

Nothing new in this file, each room will have two automations: one for setting its thermostat to the comfort preset when an event starts in its schedule, and one for setting it to the eco preset once the event ends.

## Deploying to Home Assistant
Let's see how to move this YAML files to Home Assistant. First I use the following script to generate the _actual_ YAML files using ytt:
```shell
#!/bin/sh

script_dir=$(dirname "$0")
ytt -f "$script_dir/config" --data-values-file "$script_dir/values/rooms.yaml" --output-files "$script_dir/out"
```
{: file="generate.sh" }

The generated YAML files are output in an `out` directory, there is one file for automations, one for thermostats, etc.  
Deploying these files to Home Assistant is done with the GUI using the [File Editor](https://www.home-assistant.io/common-tasks/supervised/#installing-and-using-the-file-editor-add-on) add-on.  
First, theses lines must be added to the main configuration file to reference the new files:
```
automation: !include automations.yaml
climate: !include climate.yaml
schedule: !include schedules.yaml
```
{: file="homeassistant/configuration.yaml" }

Then, each file can be created and edited using File Editor in the GUI. Once the file are up-to-date in Home Assistant, I run a [configuration check](https://www.home-assistant.io/common-tasks/supervised/#configuration-check) and if everything is ok, I [reload](https://www.home-assistant.io/docs/tools/dev-tools/#reloading-the-yaml-configuration) the configuration to apply the changes.

As a disclaimer, my solution is not fully automated:
- I add physical devices to Home Assistant using the GUI as it requires physical interactions (pushing a button) on each device, so it can't be done with a _zero-touch_ approach
- For the moment I also deploy the generated files to Home Assistant using the GUI

## Adding new features

## Wrapping-up