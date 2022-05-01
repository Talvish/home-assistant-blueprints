# Home Assistant Blueprints
In early 2022 I started playing with Home Assistant to increase my control over my home's smart devices. I've created a number of automations and scripts and I've started converting relevent ones into blueprints I can share. 

Here is what I've converted so far.

## Scripts
| Name / Code | HA Integration | Details |
| --- | --- | --- |
| [Play Media](https://github.com/Talvish/home-assistant-blueprints/blob/main/script/play_media.yaml) | [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fplay_media.yaml) | Plays the specified media with shuffle, repeat and volume options. If you have a multi-room setup (e.g. Sonos) additional media players can be specified. The script will place all players into a group and if a volume is specified it will be applied to all players. There are no inputs for this blueprint. |
| [Play Random Media](https://github.com/Talvish/home-assistant-blueprints/blob/main/script/play_random_media.yaml) | [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fplay_random_media.yaml) | This is the same as `Play Media` except you can specify a list of media content ids that the script will randomly choose from whenever the script is invoked. There are no inputs for this blueprint. |
| [Stop Sonos Alarm](https://github.com/Talvish/home-assistant-blueprints/blob/main/script/stop_sonos_alarm.yaml) | [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fstop_sonos_alarm.yaml) | Stops current audio on the specified Sonos speaker but if nothing is playing the script checks if an alarm is coming soon and if so, prevents it from going off by disabling the alarm and then re-enabling after it would have gone off. Only one alarm is impacted. This is perfect for when you wake up earlier than your alarm and don't want the alarm to wake others. There are no inputs for this blueprint. |

&nbsp;
## Automations

| Name / Code | HA Integration | Details |
| --- | --- | --- |
| [Automatic Timed Turn Off](https://github.com/Talvish/home-assistant-blueprints/blob/main/automation/timed_turn_off.yaml) | [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Ftimed_turn_off.yaml) | Automatically turns off the specified switch after a given time period whenever the specified switch is turned on. This was my equivalent of `hello world` to figure out how to create blueprints. |
