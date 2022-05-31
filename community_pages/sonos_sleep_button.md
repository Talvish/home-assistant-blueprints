This blueprint creates an automation that listens to a button press on a **Lutron Caseta Pico Remote** and depending on the time of day performs different actions on a **Sonos Speaker**:

If pressed during the `Wake Phase` the automation will:
* see if an alarm is playing and if so, stop it, otherwise temporarily disable an upcoming alarm 
* optionally run an additional action (e.g. open blinds and/or play music)

If pressed during the `Sleep Phase`, the automation will:
* optionally run an additional action (e.g. turn off lights and/or turn on security system)
* announce when the next alarm is set to run
* optionally play music and set a sleep timer

## Script Inputs ##

General inputs:
* Sonos speaker id to use for the alarms and music
* Pico remote id and button

`Wake Phase` inputs:
* Time when the phase starts
* Action to run
* Time period to look for an alarm to disable

`Sleep Phase` inputs:
* Time when the phase starts
* Action to run
* Time period to look for an alarm to announce, and volume level for the announcement
* Sleep music to play, sleep music volume, and sleep timer


## How to Install ##
This automation blueprint relies on three script blueprints I previously created. Since Home Assistant blueprints don't support dependencies (more on that in a separate post), the dependencies have to be installed manually. I recommend the following order:
1. Install the `Announce Sonos Alarm` script blueprint (details [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)) and create a script with an `Entity ID` of `announce_sonos_alarm` and optionally change the text-to-speech provider<br /> [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fannounce_sonos_alarm.yaml)

2. Install the `Stop Sonos Alarm` script blueprint (details [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)) and create a script with an `Entity ID` of `stop_sonos_alarm`<br/> [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fstop_sonos_alarm.yaml) 

3. Install the `Play Random Media` script blueprint (details [here](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)) and create a script with an `Entity ID` of `play_random_media` <br /> [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fplay_random_media.yaml) <br />... and finally ... <br/>

4. Install the actual `Sonos Sleep Button` automation blueprint itself<br />
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_button.yaml)

## Additional Notes ##
* This blueprint is a bit of an experiment, both technically (configurable triggers and actions) and experientially (see what it is like having dependencies on other blueprints); I'll post my thoughts on that later, but installing requires more tinkering/knowledge than is ideal. If the instructions below are insufficent, I will update them. 
* This is one of my **favourite automations**. It simplifies sleep alarm handling down to one contextual button press. In my configuration, my `Sleep Phase` action includes turning off all downstairs lights and checking to see if the master bathroom is occupied and has lights on, and if so, dims the lights to the lowest setting to minimize light bleed into the master bedroom.
* This blueprint requires `2022.5.0` because it runs the provided actions in parallel to the script actions
* I recommend setting the mode to `single` since most 

&nbsp;

````yaml

````

# Revisions #
* _2022-05-30_: Initial release

# Available Blueprints #
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Random Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)
