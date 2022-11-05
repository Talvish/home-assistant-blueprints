This blueprint creates an automation allowing a **Lutron Caseta** 5 button Pico remote to contextually 
control music or an **Xbox**. It assumes **Sonos** speakers are being used, and the same speakers are used 
for both music and the Xbox. In my configuration I have an Xbox attached to a TV that has Sonos 
speakers attached via an HDMI Arc/eArc channel.

When the Xbox is on, the remote's `on` (top) and `off` (bottom) buttons map to the Xbox `A` and `B` 
buttons respectively. This is great, particularly in media apps, since `A` is used to select, play or
pause while `B` is used to stop or go back.

When the Xbox is not powered, `on` (top) is mapped to playing / pausing music and `off` (bottom) is mapped 
to going to the next music track.

For both Xbox and music playback the `raise` (up arrow) button increases speaker volume and `lower` 
(down arrow) button decreases speaker volume. While `favourite` / `stop` (round middle) button is 
configurable.

The blueprint tries to handle oddities and conveniences for an optimal experience, including:
* If the Xbox is on but the source isn't `TV` then it ungroups speakers and sets the source to `TV`
* Re-groups speakers when re-starting music playback (groups are commonly broken after TV playback)
* When re-grouping speakers, sets the volume on all speakers to the lead speaker's volume 
* When re-setting the volume during re-grouping the order of operations changes so you don't get odd volume spikes
* When you press the `raise` / `lower` buttons the volume changes for all speakers

## Script Inputs ##

* _[required]_ Pico remote
* _[required]_ Xbox console
* _[required]_ Primary Sonos speaker
* _[optional]_ List of additional Sonos speakers (to group together for a multi-room setup)
* _[optional]_ List of music content IDs (typically URLs) where one will randomly be chosen to play
* _[required]_ Volume (for music playback)
* _[required]_ Shuffle (for music playback)
* _[required]_ Repeat (for music playback)
* _[optional]_ Action to run when you press the `favourite` button

Note: the volume, shuffle and repeat values are only used when music is set to play the first time.


## How to Install ##
This automation blueprint relies on another blueprint I previously created. Since Home Assistant blueprints don't support dependencies the dependencies have to be installed manually. I recommend the following order:

1. Install the `Play Random Media` script blueprint (details [here](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)) and create a script with an `Entity ID` of `play_random_media` <br /> [![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fplay_random_media.yaml)

2. Install the actual `Contextual Pico Media Remote` automation blueprint  and create the automation<br />
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fcontextual_pico_media_remote.yaml)

## Additional Notes ##
* This is another of my **favourite automations** since I commonly switch between music and Xbox. While you still need an Xbox controller or media remote to browse content, I find the infrared on the Xbox can be finicky, media remotes aren't physically stable for quick button presses so you have to pick them up (unlike a Pico remote on a stand), and powering up Xbox controllers takes time. So this script is very convenient for quickly selecting, playing, pausing and going back.
* This only requires Sonos speakers because I'm uncertain if other speakers can change their `source` to `TV` and I only have Sonos speakers to test with
* I recommend setting the mode to `single` since this creates a script bound to fixed speakers, Xbox and remote

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

# Automation blueprint that contextual controls music on speakers or context on an Xbox.

blueprint:
  name: Contextual Pico Media Remote
  description:
    This blueprint creates an automation allowing a **Lutron Caseta** 5 button  Pico remote to contextually 
    control music or an **Xbox**. It assumes **Sonos** speakers are being used, and the same speakers are used 
    for both music and the Xbox. In my configuration I have an Xbox attached to a TV that has Sonos 
    speakers attached via an HDMI Arc/eArc channel.
    <br>
    <br>
    When the Xbox is on, the remote's `on` (top) and `off` (bottom) buttons map to the Xbox `A` and `B` 
    buttons respectively. This is great, particularly in media apps, since `A` is used to select, play or
    pause while `B` is used to stop or go back.
    <br>
    <br>
    When the Xbox is not powered, `on` (top) is mapped to playing / pausing music and `off` (bottom) is mapped 
    to going to the next music track.
    <br>
    <br>
    For both Xbox and music playback the `raise` (up arrow) button increases speaker volume and `lower` 
    (down arrow) button decreases speaker volume. While `favourite` / `stop` (round middle) button is 
    configurable.

  domain: automation
  input:
    # general inputs
    remote_id:
      name: Pico Remote
      description: The entity id of the pico remote to enable.
      selector:
        device:
          model: PJ2-3BRL-GXX-X01 (Pico3ButtonRaiseLower)
    xbox_id:
      name: Xbox Console
      description: The entity id of the Xbox console.
      selector:
        entity:
          domain: media_player
          integration: xbox
    speaker_id:
      name: Speaker
      description: The entity id of the primary speaker.
      selector:
        entity:
          domain: media_player
          integration: sonos
    speaker_group_members:
      name: "Speaker Group Members"
      description:
        The entity ids of the additional entities to group together in a multi-room system. This ignores
        all types but entity ids. These speakers are only used when playing music, not when playing on
        the Xbox. Defaults to  blank.
      default: {}
      selector:
        target:
          entity:
            domain: media_player
            integration: sonos

    # music related inputs
    music_content_ids:
      name: "Music Content IDs"
      description:
        The list of music IDs and one will be randomly chosen to play. Examples include Spotify playlists or albums,
        Tune-In radio stations, etc. Defaults to blank.
      default: {}
      selector:
        object:
    music_volume_level:
      name: "Music Volume Level"
      description: Float for volume level. Range 0..1. Defaults to 0.2.
      default: 0.2
      selector:
        number:
          min: 0
          max: 1
          step: 0.01
          mode: slider
    music_shuffle:
      name: "Music Shuffle"
      description: True/false for enabling/disabling shuffle. Defaults to `true`.
      default: true
      selector:
        boolean: null
    music_repeat:
      name: "Music Repeat Mode"
      description: Repeat mode set to off, all, one. Defaults to `all`.
      default: "all"
      selector:
        select:
          options:
            - "off"
            - "all"
            - "one"

    # favourite as an input
    favourite_action:
      name: Favourite Action
      description: An action to run when the Pico remote's favourite button is pressed.
      default: []
      selector:
        action:

variables:
  # oddly you can't get blueprint inputs directly into jina so putting the
  # input into a variable so I can verify after
  speaker_entity_id: !input speaker_id
  speaker_group_members_hack: !input speaker_group_members
  speaker_group_members: >-
    {# can't modify a list or reset a variable, so using a replaceable array in a mutable container as work around#}
    {# we only uses entity_id, and there will only be one given it is a dictionary #}
    {# but in the future we can look at converting areas and devices into entities #}
    {%- set workaround = namespace(members = []) -%}
    {%- for key in speaker_group_members_hack if key == 'entity_id'-%}
      {%- set workaround.members = speaker_group_members_hack[key] -%}
    {%- endfor -%}
    {{ workaround.members }}
  current_group_members: >-
    {{ state_attr( speaker_entity_id, 'group_members') }}
  xbox_entity_id: !input xbox_id
  xbox_device_id: >-
    {{ device_id( xbox_entity_id ) }}
  music_content_ids: !input music_content_ids
  music_shuffle: !input music_shuffle
  music_repeat: !input music_repeat
  music_volume_level: !input music_volume_level
  speaker_volume_level: >-
    {%- set volume = state_attr(speaker_entity_id, 'volume_level') -%}
    {%- if volume == None or volume <= 0 -%}
      {{ music_volume_level }}
    {%- else -%}
      {{ volume }}
    {%- endif -%}

  # to make it easier and have less jinja code, and repeated state
  # checks we create some variables
  is_speaker_source_tv: >-
    {{ is_state_attr( speaker_entity_id, 'source', 'TV') }}
  is_speaker_media_set: >-
    {%- set duration = state_attr(speaker_entity_id, 'media_duration') -%}
    {%- if duration == None or duration <= -1 -%}
      {{ false }}
    {%- else -%}
      {{ true }}
    {%- endif -%}
  is_speaker_playing: >-
    {{ is_state(speaker_entity_id, 'playing') }}
  is_speaker_lonely: >-
    {{ current_group_members == None or current_group_members|length == 0 or ( current_group_members|length == 1 and speaker_entity_id in current_group_members ) }}
  is_xbox_on: >-
    {{ is_state(xbox_entity_id, 'on') }}

trigger:
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: "on"
    id: on_pressed
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: "off"
    id: off_pressed
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: raise
    id: raise_pressed
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: lower
    id: lower_pressed
  - platform: device
    device_id: !input remote_id
    domain: lutron_caseta
    type: press
    subtype: stop
    id: stop_pressed

condition: []
action:
  - choose:
      - conditions:
          - condition: trigger
            id: on_pressed
        sequence:
          - choose:
              - conditions: >
                  {{ not is_xbox_on }}
                sequence:
                  - choose:
                      - conditions: >
                          {{ is_speaker_media_set }}
                        sequence:
                          # if xbox is not on and the speaker is set to play music
                          # we see if we should group and if so, we set the volume
                          # for all speakers to be the same as the leader and then
                          # play/pause, otherwise we just play/pause
                          - choose:
                              - conditions: >
                                  {{ is_speaker_lonely and speaker_group_members|length > 0 }}
                                sequence:
                                  - choose:
                                      - conditions: >
                                          {{ is_speaker_playing }}
                                        sequence:
                                          # we set volume second so you don't hear a
                                          # change in volume before it pauses
                                          - service: media_player.media_pause
                                            target:
                                              entity_id: >-
                                                {{ speaker_entity_id }}
                                            data: {}
                                          - service: media_player.volume_set
                                            target:
                                              entity_id:
                                                - "{{ speaker_entity_id }}"
                                            data:
                                              volume_level: "{{ speaker_volume_level }}"
                                          - service: media_player.volume_set
                                            target:
                                              entity_id: >-
                                                {{ speaker_group_members }}
                                            data:
                                              volume_level: "{{ speaker_volume_level }}"
                                    default:
                                      # we set volume first so you don't hear a
                                      # change in volume after it starts to play
                                      - service: media_player.volume_set
                                        target:
                                          entity_id:
                                            - "{{ speaker_entity_id }}"
                                        data:
                                          volume_level: "{{ speaker_volume_level }}"
                                      - service: media_player.volume_set
                                        target:
                                          entity_id: >-
                                            {{ speaker_group_members }}
                                        data:
                                          volume_level: "{{ speaker_volume_level }}"
                                      # we aren't playing so we play after volume
                                      # changes so it doesn't sound odd
                                      - service: media_player.media_play
                                        target:
                                          entity_id: >-
                                            {{ speaker_entity_id }}
                                        data: {}
                                  # we do the join after the above play/pause
                                  # and volume changes, since there can be a
                                  # delay when calling join, but we want people
                                  # to know the button was pressed
                                  - service: media_player.join
                                    data:
                                      group_members: >-
                                        {{ speaker_group_members }}
                                    target:
                                      entity_id: >-
                                        {{ speaker_entity_id }}
                            default:
                              - service: media_player.media_play_pause
                                target:
                                  entity_id: >-
                                    {{ speaker_entity_id }}
                                data: {}
                    default:
                      # if xbox is not on and the speaker is not set to play
                      # music then we start playing what was passed in
                      - service: script.play_random_media
                        data:
                          media_content_type: music
                          entity_id: >-
                            {{ speaker_entity_id }}
                          group_members:
                            entity_id: >-
                              {{ speaker_group_members }}
                          repeat: >-
                            {{ music_repeat }}
                          volume_level: >-
                            {{ music_volume_level }}
                          shuffle: >-
                            {{ music_shuffle }}
                          media_content_ids: >
                            {{ music_content_ids }}
            default:
              # if the xbox is on, we make sure the source is set to the TV
              - choose:
                  - conditions: >
                      {{ not is_speaker_source_tv }}
                    sequence:
                      - service: media_player.media_pause
                        data: {}
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                      - service: media_player.unjoin
                        data: {}
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                      - service: media_player.select_source
                        data:
                          source: TV
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                default: []
              # and then send the remote command
              - service: remote.send_command
                data:
                  command: A
                  device_id: >-
                    {{ xbox_device_id }}

      - conditions:
          - condition: trigger
            id: off_pressed
        sequence:
          - choose:
              - conditions: >
                  {{ not is_xbox_on }}
                sequence:
                  # if the xbox isn't on, then we to go the next track
                  - service: media_player.media_next_track
                    target:
                      entity_id: >-
                        {{ speaker_entity_id }}
                    data: {}
            default:
              # if the xbox is on then we make sure the source is set to the TV
              - choose:
                  - conditions: >
                      {{ not is_speaker_source_tv }}
                    sequence:
                      - service: media_player.media_pause
                        data: {}
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                      - service: media_player.unjoin
                        data: {}
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                      - service: media_player.select_source
                        data:
                          source: TV
                        target:
                          entity_id: >-
                            {{ speaker_entity_id }}
                default: []
              # and then send the command
              - service: remote.send_command
                data:
                  command: B
                  device_id: >-
                    {{ xbox_device_id }}

      - conditions:
          - condition: trigger
            id: raise_pressed
        sequence:
          # raises volume, including grouped speakers as appropriate
          - choose:
              - conditions: >
                  {{ is_speaker_lonely }}
                sequence:
                  - service: media_player.volume_up
                    target:
                      entity_id:
                        - "{{ speaker_entity_id }}"
                    data: {}
            default:
              - service: media_player.volume_up
                target:
                  entity_id: >-
                    {{ current_group_members }}
                data: {}

      - conditions:
          - condition: trigger
            id: lower_pressed
        sequence:
          # lowers volume, including grouped speakers as appropriate
          - choose:
              - conditions: >
                  {{ is_speaker_lonely }}
                sequence:
                  - service: media_player.volume_down
                    target:
                      entity_id:
                        - "{{ speaker_entity_id }}"
                    data: {}
            default:
              - service: media_player.volume_down
                target:
                  entity_id: >-
                    {{ current_group_members }}
                data: {}

      - conditions:
          - condition: trigger
            id: stop_pressed
        sequence:
          - choose:
              - conditions: >
                  {{ false }}
                sequence: []
            default: !input favourite_action

    default: []
mode: single
max_exceeded: silent
````

# Revisions #
* _2022-11-05_: Initial release

# Available Blueprints #
* Automation: [Lutron Pico Media Remote for Contextually Handling Music or Xbox on Sonos Speakers](https://community.home-assistant.io/t/lutron-pico-media-remote-for-contextually-handling-music-or-xbox-on-sonos-speakers/484748)
* Automation: [Automation for Sonos + Lutron Picos to Magically Manage Sleep Alarms and Music](https://community.home-assistant.io/t/automation-for-sonos-lutron-picos-to-magically-manage-sleep-alarms-and-music/426477)
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Random Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)
