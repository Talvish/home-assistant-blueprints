This blueprint creates an automation that listens for a button press on a **Lutron Caseta Pico Remote** and depending on the time of day performs different sleep / alarm actions on a **Sonos Speaker**:

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

4. Install the actual `Sonos Sleep Handler` automation blueprint itself and create the automation<br />
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_handler.yaml)

## Additional Notes ##
* This blueprint is an experiment, both technically (configurable triggers and actions) and experientially (see what it is like having dependencies on other blueprints); I'll post my thoughts on that later, but installing requires more tinkering/knowledge than is ideal. If the instructions below are insufficent, I will update them. 
* This is one of my **favourite automations**. It simplifies sleep alarm handling down to one contextual button press. In my configuration, my `Sleep Phase` action includes turning off all downstairs lights and checking to see if the master bathroom is occupied and has lights on, and if so, dims the lights to the lowest setting to minimize light bleed into the master bedroom.
* This blueprint requires Home Assistant `2022.5.0` because it runs the provided actions in parallel with the script actions
* I recommend setting the mode to `single` since general safer to let it run

&nbsp;

````yaml
# ***************************************************************************
# *  Copyright 2022 Joseph Molnar
# *
# *  Licensed under the Apache License, Version 2.0 (the "License");
# *  you may not use this file except in compliance with the License.
# *  You may obtain a copy of the License at
# *
# *      http://www.apache.org/licenses/LICENSE-2.0
# *
# *  Unless required by applicable law or agreed to in writing, software
# *  distributed under the License is distributed on an "AS IS" BASIS,
# *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# *  See the License for the specific language governing permissions and
# *  limitations under the License.
# ***************************************************************************

# Automation blueprint that makes sleeping and waking time convenient on Sonos.

blueprint:
  name: Sonos Sleep / Alarm Handler
  description:
    This blueprint creates an automation that listens for a button press on a Lutron Caseta Pico remote
    and depending on the time of day does different things. If pressed during the `Wake Phase` it will
    disable the current or upcoming alarm on a Sonos speaker disable, and execute an optional action.
    If pressed during the `Sleep Phase`, it executes an optional action, annouce when the next alarm
    is set to run, and ends with some optional sleep music.
  domain: automation
  homeassistant:
    min_version: 2022.5.0 # this is needed because we are using 'parallel' below
  input:
    # general inputs

    speaker_id:
      name: Sonos Speaker
      description: The entity id of the Sonos speaker.
      selector:
        entity:
          domain: media_player
          integration: sonos
    remote_id:
      name: Pico Remote
      description: The entity id of the pico remote to enable.
      selector:
        device:
          integration: lutron_caseta

    button:
      name: Pico Remote Button
      description: The button on the pico remote. Common values are 'raise', 'lower', 'on', 'off' and 'stop'.
      selector:
        text:

    # wake phase inputs

    wake_start_time:
      name: Wake Phase | Start Time
      description:
        The time when someone may wake and need to disable alarm. It should be set a good amount of
        time prior to when an alarm would go off but after you go to sleep. The `Wake Phase` ends
        at `Sleep Phase Start Time`.
      default: "04:00:00"
      selector:
        time:
    wake_action:
      name: Wake Phase | Action
      description: An action to run while disabling the alarm. Examples could include opening blinds, playing music, etc.
      default: []
      selector:
        action:
    disable_alarm_time_window:
      name: Wake Phase | Period of Time to Find Alarm to Disable
      description:
        The number of minutes (up to 6 hours) the script will look forward to find an active alarm to disable
        if music isn't playing. The alarm is re-enabled about 5 minutes after the time it would have gone off.
      default: 60
      selector:
        number:
          min: 0
          max: 360
          unit_of_measurement: minutes
          mode: slider

    # sleep phase inputs

    sleep_start_time:
      name: Sleep Phase | Start Time
      description: A time close to when, though before, you go to sleep. The `Sleep Phase` ends at the `Wake Phase Start Time`.
      default: "18:00:00"
      selector:
        time:
    sleep_action:
      name: Sleep Phase | Action
      description:
        An action to run while announcing the alarm time and playing sleep music. Examples include turning
        off lights, turning down a thermostat, etc.
      default: []
      selector:
        action:
    announce_alarm_time_window:
      name: Sleep Phase | Period of Time to Find Alarm to Announce
      description:
        The amount of time, in minutes, to look forward for an alarm to announce. Maximum time is 1439 minutes
        (23 hours and 59 minutes), minimum is 30 minutes and the default is 720 minutes (12 hours).
      default: 720
      selector:
        number:
          min: 30
          max: 1439
          unit_of_measurement: minutes
          mode: slider
    announce_volume_level:
      name: "Sleep Phase | Alarm Announce Volume"
      description: "Float for volume level. Range 0..1. If value isn't given, volume isn't changed. Recommend the value be more than the music volume."
      default: 0.15
      selector:
        number:
          min: 0
          max: 1
          step: 0.01
          mode: slider
    music_content_ids:
      name: "Sleep Phase | Music Content IDs"
      description: "The list of music IDs and one will be randomly chosen to play. Examples include Spotify playlists or albums, Tune-In radio stations, etc."
      selector:
        object:
    music_volume_level:
      name: "Sleep Phase | Music Volume Level"
      description: "Float for volume level. Range 0..1. If value isn't given, volume isn't changed."
      default: 0.02
      selector:
        number:
          min: 0
          max: 1
          step: 0.01
          mode: slider
    music_sleep_timer:
      name: Sleep Phase | Music Timer
      description: How long music will play before stopping. Default is 15 minutes. Maximum is 600 (10 hours).
      default: 15
      selector:
        number:
          min: 0
          max: 600
          unit_of_measurement: minutes
          mode: slider

variables:
  # oddly you can't get blueprint inputs directly into jina so putting the
  # input into a variable so I can verify after
  music_content_ids: !input music_content_ids
  music_sleep_timer_hack: !input music_sleep_timer
  # we multiple by 60 since the Sonos sleep timer is in seconds but we asked for
  # minutes and yes I could multiplied directly for the constants but thought
  # better to show the why
  music_sleep_timer: >
    {{ ( 25 * 60 ) if ( music_sleep_timer_hack is undefined or music_sleep_timer_hack < 0 ) else ( 600 * 60 ) if music_sleep_timer_hack > 600 else ( music_sleep_timer_hack * 60 ) }}

trigger:
  # I would love to make this trigger more generic but not really sure
  # how to do that yet, certainly can't make it an input as of yet
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: !input button

action:
  - choose:
      # 'Wake Up' phase of the automation
      - conditions:
          - condition: time
            after: !input wake_start_time
            before: !input sleep_start_time
        sequence:
          # we run these in parallel since the stop alarm may have a long
          # delay no it to re-enable the alarm
          - parallel:
              # first thing here is to turn off the alrm, which uses and
              # existing script I built, which needs to be manually added
              - service: script.stop_sonos_alarm
                data:
                  entity_id: !input speaker_id
                  time_window: !input disable_alarm_time_window
              # we now run the wake action
              # doing this weird choose thing to make so it will create the automation
              # since I can't put call !input directly after the hyphen
              - choose:
                  - conditions: >
                      {{ false }}
                    sequence: []
                default: !input wake_action

    default:
      # we run the actions given to us, and the others in parallel
      # since we don't know what it will do or take
      - parallel:
          # we start by running the sleep action
          # doing this weird choose thing to make so it will create the automation
          # since I can't put call !input directly after the hyphen
          - choose:
              - conditions: >
                  {{ false }}
                sequence: []
            default: !input sleep_action
          # but these actions, since they involve the sonos speaker
          # we want to run in sequence
          - sequence:
              # we want to announce the alarm
              - service: script.announce_sonos_alarm
                data:
                  entity_id: !input speaker_id
                  time_window: !input announce_alarm_time_window
                  volume_level: !input announce_volume_level

              # next we start to play the music that was requested, assuming
              # it was given, this also uses an existing script I built which
              # needs to be manually added
              - choose:
                  - conditions: >
                      {{ music_content_ids != None and music_content_ids != "" }}
                    sequence:
                      - service: script.play_random_media
                        data:
                          media_content_type: music # forcing music
                          entity_id: !input speaker_id
                          repeat: "all" # forcing repeat
                          volume_level: !input music_volume_level
                          shuffle: true # forcing shuffle
                          media_content_ids: >
                            {{ music_content_ids }}
                      # we clear out any existing sleep timer, just in case
                      - service: sonos.clear_sleep_timer
                        data:
                          entity_id: !input speaker_id
                      # and now set the sleep timer
                      - service: sonos.set_sleep_timer
                        data:
                          sleep_time: >
                            {{ music_sleep_timer }}
                          entity_id: !input speaker_id
                default: []
mode: single
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
