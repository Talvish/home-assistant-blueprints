This script blueprint provides an easy to way play media. The following field parameters can be given when the script is called:
* _[required]_ Primary media player
* _[required]_ ID for the content to play (typically a URL)
* _[required]_ Content type of the media to play (defaults to `music`)
* _[required]_ Primary media player
* _[optional]_ List of additional media players (to group together for a multi-room setup, e.g. Sonos)
* _[optional]_ Volume (it is applied to all grouped players)
* _[optional]_ Shuffle
* _[optional]_ Repeat


The script also has a few fail safes based on observed behaviour including:
* Double checking that shuffle and repeat actually get set and if not, set again
* If shuffle didn't get set the script also forces skipping to the next song so you don't always hear the first song in an album or playlist

## Additional notes ##
* I recommend setting the mode to `parallel` when using the same script for multiple automations
* This blueprint has no inputs, but is meant to be a script to call from automations  
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fplay_media.yaml)

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

# Plays the specified media on the specified devices with various options for shuffle, repeat, etc.

blueprint:
  name: "Play Media"
  description:
    "This blueprint is used to add a script for playing media. The script plays the media
    specified and provides shuffle, repeat and volume options. \
    If you have a multi-room setup (e.g. Sonos) additional media players can be specified. The players \
    will be grouped together with the primary media player."
  domain: script

fields:
  media_content_id:
    name: Content ID
    description: The ID of the content to play. Platform dependent.
    example: https://open.spotify.com/playlist/37i9dQZF1DX4fQhfyVRsHW?si=9e78114227474022
    required: true
    selector:
      text:
  media_content_type:
    name: "Content Type"
    description: "The type of content that will play. Platform dependent."
    example: "music"
    required: true
    selector:
      text:
  entity_id:
    description: "The entity id that will be used to play and control the specified media."
    name: "Entity"
    required: true
    selector:
      entity:
        domain: media_player
  group_members:
    description: "The entity ids of the additional entities to group together in a multi-room system. This ignores all types but entity ids."
    name: "Group Members"
    required: false
    selector:
      target:
        entity:
          domain: media_player
  shuffle:
    name: "Shuffle"
    description: "True/false for enabling/disabling shuffle. If not set, current setting is used."
    required: false
    selector:
      boolean: null
  repeat:
    name: "Repeat Mode"
    description: "Repeat mode set to off, all, one. If not set, current mode is used."
    required: false
    selector:
      select:
        options:
          - "off"
          - "all"
          - "one"
  volume_level:
    name: "Volume Level"
    description: "Float for volume level. Range 0..1. If not set, current value is used. If you specified Group Members the volume will be applied to all members."
    required: false
    selector:
      number:
        min: 0
        max: 1
        step: 0.01
        mode: slider

sequence:
  # if group members are defined then we do a grouping
  - choose:
      - conditions: >
          {{ group_members is defined }}
        sequence:
          - service: media_player.join
            data:
              group_members: >-
                {# can't modify a list or reset a variable, so using a replaceable array in a mutable container as work around#}
                {# we only uses entity_id, and there will only be one given it is a dictionary #}
                {# but in the future we can look at converting areas and devices into entities #}
                {%- set workaround = namespace(members = []) -%}
                {%- for key in group_members if key == 'entity_id'-%} 
                  {%- set workaround.members = group_members[key] -%}
                {%- endfor -%} 
                {{ workaround.members }}
              entity_id: "{{ entity_id }}"

  # we only set repeat if the repeat value was given
  - choose:
      - conditions: >
          {{ repeat is defined }}
        sequence:
          - service: media_player.repeat_set
            data:
              repeat: "{{ repeat }}"
              entity_id: "{{ entity_id }}"
    default: []

  # it appears that if you set shuffle and repeat
  # that both do not happen all the time (at least
  # on Sonos), so delay is here to help
  - choose:
      - conditions: >
          {{ repeat is defined and shuffle is defined }}
        sequence:
          - delay:
              hours: 0
              minutes: 0
              seconds: 1
              milliseconds: 0
    default: []

  # we only set shuffle if the shuffle value was given
  - choose:
      - conditions: >
          {{ shuffle is defined }}
        sequence:
          - service: media_player.shuffle_set
            data:
              shuffle: "{{ shuffle }}"
              entity_id: "{{ entity_id }}"
    default: []

  # we only set the volume if the volume value was given
  - choose:
      - conditions: >
          {{ volume_level is defined }}
        sequence:
          - service: media_player.volume_set
            data:
              volume_level: "{{ volume_level }}"
              entity_id: "{{ entity_id }}"
          - choose:
              # if we have additional members, we set all their volumes
              - conditions: >
                  {{ group_members is defined }}
                sequence:
                  # need to make the right target definition
                  - service: media_player.volume_set
                    data:
                      volume_level: "{{ volume_level }}"
                    target: >-
                      {{ group_members }}
            default: []
    default: []

  # no matter what, we get the media playing
  # assuming we have the right media type
  - service: media_player.play_media
    data:
      media_content_id: "{{media_content_id}}" 
      media_content_type: "{{ media_content_type }}"
      entity_id: "{{ entity_id }}"

  # now we double check shuffle state and set
  # again because sometimes it doesn't take
  - choose:
      - conditions: >
          {{ shuffle is defined and is_state_attr(entity_id, 'shuffle', shuffle) == false }}
        sequence:
          - service: media_player.shuffle_set
            data:
              shuffle: "{{ shuffle }}"
              entity_id: "{{ entity_id }}"
          # in this case we wil also force the next
          # track just to be sure
          - choose:
              - conditions: >
                  {{ shuffle == true }}
                sequence:
                  - service: media_player.media_next_track
                    data:
                      entity_id: "{{ entity_id }}"
            default: []
    default: []

  # now we double check repeat state and set
  # again because sometimes it doesn't take
  - choose:
      - conditions: >
          {{ repeat is defined and is_state_attr(entity_id, 'repeat', repeat) == false }}
        sequence:
          - service: media_player.repeat_set
            data:
              repeat: "{{ repeat }}"
              entity_id: "{{ entity_id }}"
    default: []

mode: parallel
icon: mdi:music-box-outline
````
As a note, I have a very similar blueprint that takes a list of media content ids (instead of one) and will randomly choose which to play whenever the script is invoked. It can be found [here](https://github.com/Talvish/home-assistant-blueprints).
&nbsp;
# Revisions #
* _2022-04-24_: Initial release

&nbsp;
# List of Bluepints #
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)