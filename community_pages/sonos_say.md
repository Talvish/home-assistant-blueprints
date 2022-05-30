This script blueprint uses a Text-to-Speech service to play messages on Sonos speakers. The script
handles oddities to ensure a proper message playing experience. Examples include:
* Saving the state of the Sonos and restoring when done (so music will stop and continue)
* Handling  speakers in groups (for both old and newer versions of HA)
* Pausing music so volume adjustments don't impact current music
* Disabling repeat so that the announcement doesn't repeat 
* Adaptive delays to ensure proper handling of settings and removing chances of playback cut-offs

The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker 
* _[required]_ Text of the message to say
* _[optional]_ Volume for message playback. This only impacts the indicated speaker, not the group, and reverts when done.
* _[optional]_ Maximum number of seconds that the system will wait for the message to play

The following blueprint inputs can be given when creating the script:
* _[optional]_ Text-to-Speech Service engine to use to say the specified message. Default is `google_translate_say`.
* _[optional]_ Language to say messages in. Default is `en`.

This script is particularly convenient when:
* You want to announce something on a Sonos speaker but want the speakers to continue what they were doing
  when playback completes (e.g. opening a dishwasher and announcing the dishes are clean on a kitchen speaker)

## Additional Notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fsonos_say.yaml)

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

# Announces the provided message on the specified speaker.

blueprint:
  name: "Text-to-Speech on Sonos"
  description:
    This blueprint is used to add a script that will say messages on Sonos speakers. The script
    handles oddities to ensure a proper experience including saving/restore state, handling
    speaker groups, pausing music, disabling repeat, adding delays, etc.

    I recommend setting the mode to parallel if you will use this script on more than one speaker.
  domain: script
  input:
    tts_service_name:
      name: Text-To-Speech Service Name
      description:
        The text-to-speech service to use when saying the message. This must match your Home
        Assistant configuration.
      default: "google_translate_say"
      selector:
        text:
    tts_language:
      name: Language
      description: The language to use with the text-to-speech service.
      default: "en"
      selector:
        text:

fields:
  entity_id:
    description: The entity id of the Sonos speaker that will play the message.
    name: Entity
    required: true
    selector:
      entity:
        domain: media_player
        integration: sonos
  message:
    description: The text that will be played.
    name: Message
    required: true
    selector:
      text:
  volume_level:
    name: "Volume Level"
    description:
      Float for volume level. Range 0..1. If value isn't given, volume isn't changed.
      The volume will revert to the previous level after it plays the message.
    required: false
    selector:
      number:
        min: 0
        max: 1
        step: 0.01
        mode: slider
  max_wait:
    name: "Maximum Wait"
    description:
      The maximum number of seconds that the system will wait for the message to play
      before continuing with the script. If the value is too short, the message may
      get cut off. If this value isn't given, the system will calculate a value based on
      the length of the string. This is likely fine for languages such as English and
      French, but likely too short for languages such as Chinese.
    required: false
    selector:
      number:
        min: 1
        max: 60
        step: 0.25
        unit_of_measurement: seconds
        mode: slider

variables:
  entity_group_leader: >-
    {# we see if in a group since the repeat is typically controlled by it #}
    {# we use this for doing all the work since it is the primary speaker #}
    {# and everything will be shared across speakers anyhow #}
    {%- set group_members = state_attr( entity_id, "group_members" ) -%}
    {%- if group_members == None -%}
      {# we maybe on an older version of HA, so look for a different group name#}
      {%- set group_members = state_attr( entity_id, "sonos_group" ) -%}
      {%- if group_members == None -%}
        {{ entity_id }}
      {%- else -%}
        {{ group_members[0] }}
      {%- endif -%}
    {%- else -%}
      {# the first seems to be the control, at least on Sonos #}
      {{ group_members[0] }}
    {%- endif -%}
  entity_repeat_state: >-
    {# we grab the repeat state so that if in repeat mode we turn off #}
    {# and also sanity check that we got a value otherwise default to off #}
    {%- set repeat = state_attr( entity_group_leader, "repeat" ) -%}
    {%- if repeat == None -%}
      off
    {%- else -%}
      {{ repeat }}
    {%- endif -%}

  # oddly you can't get blueprint inputs directly into jina so . . .
  # 1) putting service into a variable to verify values and provide default
  tts_hack: !input tts_service_name
  tts_engine: >-
    {%- if tts_hack is undefined or tts_hack== None or tts_hack == "" -%}
      tts.google_translate_say
    {%- else -%}
      tts.{{ tts_hack }}
    {%- endif -%}
  # 2) putting language into a variable to verify values and provide default
  lang_hack: !input tts_language
  tts_language: >-
    {%- if lang_hack is undefined or lang_hack== None or lang_hack == "" -%}
      "en"
    {%- else -%}
      {{ lang_hack }}
    {%- endif -%}
  # try to approximate delays to use we do some simple checks based on text length,
  # this approximation is generally fine for western languages, though not great
  # for others BUT the range is 1 and 4 second, so not too long for hard delay
  fixed_delay: >-
    {# we use a rough estimate of speaking 16 characters per second, which is #}
    {# high, but we put min of 1 second and max of 4 regardless for the fixed delay #}
    {# this fixed delay seems to increase the chance of proper playback #}
    {% set delay = ( message | length ) / 16 %}
    {{ delay if ( delay >= 1 and delay <= 4 ) else 4 if delay >= 4 else 1 }}
  # the wait delay is used for monitoring when the speaker is done playing the
  # message. Must be long enough to not cut the message off, but being too long
  # is unlikely an issue since the state will change once the message is played
  wait_delay: >-
    {%- if max_wait is undefined or max_wait == None or not (max_wait is number) -%}
      {# if we have a bad or missing max_wait then we calculate something based  #}
      {# on the number of characters, which is great more for western languages #}
      {# others may get cut off #}
      {% set delay = ( ( message | length ) / 10 ) - fixed_delay %}
      {{ delay if delay >= 1 else 1 }}
    {%- elif max_wait < 1 -%}
      1
    {%- else -%}
      {{ max_wait }}
    {%- endif -%}

sequence:
  # save current state so we can restore to whatever was happening previously
  - service: sonos.snapshot
    data:
      entity_id: "{{ entity_group_leader }}"
      with_group: true
  # we set the volume if it is defined, and in this case we only set the
  # speaker volume not the group leader volume
  - choose:
      - conditions: >
          {{ volume_level is defined }}
        sequence:
          - choose:
              # see if the speaker is playing and pause it is so you don't
              # hear the volume adjust on the current playing audio
              - conditions:
                  - condition: template
                    value_template: >
                      {{ is_state(entity_group_leader, 'playing') }}
                sequence:
                  # so it is playing, so we want to pause
                  - service: media_player.media_pause
                    data:
                      entity_id: >
                        {{ entity_group_leader }}
                  # and do a quick to make sure it isnt' playing
                  - wait_template: "{{ states( entity_id ) != 'playing' }}"
                    timeout:
                      seconds: 2
            default: []
          # now we can set the set the volume
          - service: media_player.volume_set
            data:
              volume_level: "{{ volume_level }}"
              entity_id: "{{ entity_id }}"
  # we check to see if the player is in repeat state and turn off otherwise
  # the alarm announcement will repeat forever
  - choose:
      - conditions: >
          {{ entity_repeat_state != "off" }}
        sequence:
          - service: media_player.repeat_set
            data:
              repeat: "off"
              entity_id: "{{ entity_group_leader }}"
          - wait_template: "{{ state_attr( entity_group_leader, 'repeat' ) == 'off' }}"
            timeout:
              seconds: 4
    default: []

  # the actual call to say the message
  - service: "{{ tts_engine }}"
    data:
      entity_id: "{{ entity_group_leader }}"
      message: "{{ message }}"
      language: "{{ tts_language }}"

  # not sure why, but setting the repeat doesn't always properly take
  # I can see it changing in the state  when the TTY goes the repeat
  # turns back on ... calling to turn off again helps
  - service: media_player.repeat_set
    data:
      repeat: "off"
      entity_id: "{{ entity_group_leader }}"

  # first we wait for it to start to properly announce the time
  - wait_template: "{{ states( entity_id ) == 'playing' }}"
    timeout:
      seconds: 2 # timeout so doesn't sit forever
  # we then put in a slight delay since the announcement can get cut off
  - delay:
      seconds: >
        {{ fixed_delay | int }}
      milliseconds: >
        {{ ( ( fixed_delay - ( fixed_delay | int ) ) * 1000 ) | int }}

  # then we wait for it to finish announcing before we continue
  - wait_template: "{{ states( entity_id ) != 'playing' }}"
    timeout:
      seconds: > # timeout so doesn't sit forever
        {{ wait_delay | int }}
      milliseconds: >
        {{ ( ( wait_delay - ( wait_delay | int ) ) * 1000 ) | int }}

  # and now we restore where we were which should cover repeat, what's playing, etc.
  # NOTE: so far this works when driven by Sonos or HA, but when driven from Alexa
  #       it doesn't seem to work as well
  - service: sonos.restore
    data:
      entity_id: "{{ entity_group_leader }}"
      with_group: true

mode: parallel
icon: mdi:account-voice
````

# Revisions #
* _2022-05-25_: Initial release

# Available Blueprints #
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Random Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)
