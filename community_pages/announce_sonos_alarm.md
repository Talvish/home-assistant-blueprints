This blueprint is used to add a script that will announce when the next alarm is set on the specified Sonos speaker. If the alarm is less than two hours away it will indicate how long until it rings. Otherwise the alarm will indicate the time when the alarm is set using a 12 hour format (e.g. 1 AM vs 1 PM). If it cannot find an alarm it will indicate that as well.

It will gracefully handle when speakers are in groups, and restore playing music after it announces the alarm.

The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker to check for an alarm 
* _[optional]_ Sonos speaker to announce the time of alarm
* _[optional]_ Time window to look for an alarm to announce. Maximum time is 1439 minutes (23 hours and 59 minutes), minimum is 30 minutes and the default is 720 minutes (12 hours)
* _[optional]_ Volume to used to play the announcement. 
* _[optional]_ Minimum wait to ensure state changes in Sonos are correct (advanced setting). 
* _[optional]_ Maximum wait to help ensure messages finish playing (advanced setting).

This script is particularly convenient when:
* You don't want to check phone apps when going to bed to see if/when an alarm is set
* You have have a phyiscal button near your bed (e.g. a Lutron Caseta Pico remote)

## How to Install ##
This automation blueprint relies on another script blueprint I previously created. Since Home Assistant blueprints don't support dependencies, the dependency has to be installed manually. I recommend the following installation order:
1. Install the `Sonos Text-to-Speech` script blueprint (details [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)) and create a script with an `Entity ID` of `sonos_say` and optionally change the text-to-speech provider and language<br />[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fsonos_say.yaml)
2. Install this `Announce Sonos Alarm` automation blueprint <br />[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fannounce_sonos_alarm.yaml)

## Additional Notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
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

# Announces when the next alarm is set on a Sonos speaker.
# Future considerations
# - blueprint-based 12 vs 24 clock

blueprint:
  name: "Announce Sonos Alarm"
  description:
    This blueprint is used to add a script that will announce when the next alarm is set on the
    specified Sonos speaker. If the alarm is less than two hours away it will indicate how long
    until it rings. Otherwise the alarm will indicate the time when the alarm is set using a
    12 hour format (e.g. 1 AM vs 1 PM). If it cannot find an alarm it will indicate that as well.

    The script will gracefully handle when speakers are in groups, and restore playing music
    after it announces the alarm.

    This is helpful as part of an automation when going to bed so you don't need to use a phone
    app to see when/if an alarm is set.

    I recommend setting the mode to parallel if you will use this script on more than one speaker.
  domain: script

fields:
  entity_id:
    description:
      The entity id of the Sonos speaker to find the alarm and, if `announce_entity_id`
      isn't set, then the speaker to announce on.
    name: Alarm Entity
    required: true
    selector:
      entity:
        domain: media_player
        integration: sonos
  announce_entity_id:
    description:
      An optional entity id for a speaker, which doesn't have to be from Sonos,
      to announce the alarm found on `entity_id`. If this isn't specified then
      the alarm is announced on `entity_id`.
    name: Announce Entity
    required: false
    selector:
      entity:
        domain: media_player
  time_window:
    name: Time Window
    description:
      The amount of time, in minutes, to look forward for an alarm. Maximum time is 1439 minutes
      (23 hours and 59 minutes), minimum is 30 minutes and the default is 720 minutes (12 hours).
    required: false
    default: 720
    selector:
      number:
        min: 30
        max: 1439
        unit_of_measurement: minutes
        mode: slider
  volume_level:
    name: "Volume Level"
    description: "Float for volume level. Range 0..1. If value isn't given, volume isn't changed."
    required: false
    selector:
      number:
        min: 0
        max: 1
        step: 0.01
        mode: slider
  min_wait:
    name: "Minimum Wait"
    description:
      The minimum number fo seconds that the system will wait for state changes from
      Sonos. See the sonos_say script for details.
    required: false
    selector:
      number:
        min: 0
        max: 60
        step: 0.25
        unit_of_measurement: seconds
        mode: slider

  max_wait:
    name: "Maximum Wait"
    description:
      After the system starts playing the message the system waits for the message
      to finish play. See the sonos_say script for details.
    required: false
    selector:
      number:
        min: 1
        max: 60
        step: 0.25
        unit_of_measurement: seconds
        mode: slider

variables:
  valid_time_window: >-
    {# make sure we have a valid time window to use #}
    {{ 720 if ( time_window is undefined or time_window < 30 ) else 1439 if time_window > 1439 else time_window }}
  alarm: >-
    {# Note: I'm new to jinja so there may be more efficient ways to do this...if so, contact me #}
    {%- set entities = device_entities( device_id( entity_id ) ) -%}
    {# since the current day can flip while running this script and we want this as accurate as #}
    {# possible, we capture the current time right away and base all calculations on it so there #}
    {# are no oddities but does mean up front math and more variables to use later #}
    {%- set current_time = now() -%}
    {# set for current midnight to help with calculating alarms for today below #}
    {# again, doesn't use today_at() to ensure no cross-over in the day during execution #}
    {%- set current_midnight = current_time.replace(hour=0,minute=0,second=0,microsecond=0) %} 
    {# we track tomorrow to help with cross-overs at midnight #}
    {%- set tomorrow_midnight = current_midnight + timedelta(days=1) -%} 
    {# the Sonos alarms start the week on Sunday but start at 0 and the ISO week starts on Monday #}
    {# but starts at 1, so to correct the math we just modulo against 7 #}
    {%- set current_weekday = current_midnight.isoweekday() % 7 -%}
    {%- set tomorrow_weekday = ( current_weekday + 1 ) % 7 -%}
    {# we use this namespace to track the alarm time info we find #}
    {%- set alarm_timing = namespace( days = "", time={}) -%}
    {# we set initial delta to help processing later #}    
    {%- set found_alarm = namespace( found = false, entity_id = "", time = "", delta = timedelta(minutes=valid_time_window )) -%} 
    {# we iterate through all the entities on the specified entity looking for alarms #}
    {%- for entity_id in entities -%}
      {%- set entity_id_parts = entity_id.split('.') -%}
      {%- set entity = states[entity_id_parts[0]][entity_id_parts[1]] -%}
      {%- set alarm_id = state_attr( entity_id, "alarm_id") -%}
      {# we see if we found a switch that has an alarm #}
      {%- if entity.domain == "switch" and alarm_id != None and entity.state == "on" -%}
        {# we need to figure out which days the alarm is on #}
        {%- set recurrence = state_attr( entity_id, "recurrence") -%}
        {%- if recurrence == "DAILY" -%}
          {# if a daily alarm, we just indicate we will look across all the days #}
          {%- set alarm_timing.days = "0123456" -%}
        {%- elif recurrence == "WEEKDAYS" -%}
          {# if weekdays we just indicate we will look across Monday through Friday #}
          {%- set alarm_timing.days = "12345" -%}
        {%- elif recurrence == "WEEKENDS" -%}
          {# if a daily alarm, we just indicate we will look across Saturday and Sunday #}
          {%- set alarm_timing.days = "06" -%}
        {%- else -%}
          {# if not a daily alarm, the format is ON_#### (e.g. ON_1236), where ### are numbers of the weekdays #}
          {%- set alarm_timing.days = recurrence.split('_')[1] -%}
        {%- endif -%}
        {# we need the time as a delta so we can add later, yes this is hefty work #}
        {# but I don't know another way to take a string and turn into a timedelta #}
        {%- set alarm_time = state_attr( entity_id, "time") -%}
        {%- set alarm_time_parts = alarm_time.split(':') -%}
        {%- set alarm_timing.time = timedelta( hours=alarm_time_parts[0] | int, minutes=alarm_time_parts[1] | int ) -%}
        {# so now we loop through each day to see if we have a matching alarm #}
        {# ideally we'd shortcircuit once we find it or go too far #}
        {%- for day_string in alarm_timing.days %}
          {%- set day = day_string | int -%}
          {%- if day == current_weekday -%}
            {# if the alarm is for today then we use current_midnight as a basis for time comparison #}
            {%- set alarm_timestamp = current_midnight + alarm_timing.time-%}
            {%- set delta = alarm_timestamp - current_time -%}
            {# delta can't be negative, and replace alarm if delta is less than the last found (which includes never found) #}
            {%- if delta >= timedelta() and found_alarm.delta >= delta -%}
              {%- set found_alarm.found = true -%}
              {%- set found_alarm.entity_id = entity_id %}
              {%- set found_alarm.time = alarm_time %}
              {%- set found_alarm.call_timestamp = current_time %}
              {%- set found_alarm.delta = delta %}
            {%- endif -%}
          {%- elif day == tomorrow_weekday %}
            {# if the alarm is for tomorrow then we use tomorrow_midnight as a basis for time comparison #}
            {%- set alarm_timestamp = tomorrow_midnight + alarm_timing.time-%}
            {%- set delta = alarm_timestamp - current_time -%}
            {# delta can't be negative, and replace alarm if delta is less than the last found (which includes never found) #}
            {%- if delta >= timedelta() and found_alarm.delta >= delta -%}
              {%- set found_alarm.found = true -%}
              {%- set found_alarm.entity_id = entity_id %}
              {%- set found_alarm.time = alarm_time %}
              {%- set found_alarm.call_timestamp = current_time %}
              {%- set found_alarm.delta = delta %}
            {%- endif -%}
          {%- endif -%}
        {%- endfor %}
      {%- endif -%}
    {%- endfor -%}
    {# if we find the alarm then we can stop / pause it #}
    {%- if found_alarm.found == true %}
      {# return back the delta to help calculate when to re-enable the alarm, yes missing microseconds, but close enough #}
      {% set delta_hours = ( found_alarm.delta.seconds / ( 3600 ) ) | int -%}
      {% set remaining_seconds = found_alarm.delta.seconds - ( delta_hours * 3600 ) | int -%}
      {% set delta_minutes = ( remaining_seconds / 60 ) | int  %}
      {% set remaining_seconds = remaining_seconds - ( delta_minutes * 60 ) -%}
      {{ { "found": true, "entity_id": found_alarm.entity_id, "time": found_alarm.time, "call_timestamp" : found_alarm.call_timestamp | string, "delta": { "hours": delta_hours,"minutes": delta_minutes, "seconds" : remaining_seconds } } }}
    {%- else -%}
      {{ { "found": false } }}
    {%- endif -%}
  actual_announce_entity_id: >-
    {# for playback we see if we have an announce_entity_id and use it #}
    {# otherwise we use the entity_id #}
    {%- if announce_entity_id is undefined or announce_entity_id == None or announce_entity_id == "" -%}
      {# so we don't have an announce_entity_id so now we see if we #}
      {# are in a group since the repeat is typically controlled by it #}
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
    {%- else -%}
      {# we have the announce_entity_id, so we use it #}
      {{ announce_entity_id }}
    {%- endif -%}

  message: >-
    {%- if alarm.found == false -%}
      {# an alarm wasn't found so we indicate that #}

      {# calculate strings to indicate the hours / minutes we checked #} 
      {# but words change depending IF / HOW MANY hours / minutes we have#}
      {% set window_hours = ( valid_time_window / ( 60 ) ) | int -%}
      {% set window_minutes = valid_time_window - ( window_hours * 60 ) -%}
      {# so first the hours #}
      {%- set hour_string = "" -%}
      {%- if window_hours == 1 -%}
        {%- set hour_string = "1 hour" -%}
      {%- elif window_hours > 1 -%}
        {%- set hour_string = ( window_hours | string) + " hours" -%}
      {%- endif -%}
      {# then the minutes #}
      {%- set minute_string = "" -%}
      {%- if window_minutes == 1 -%}
        {%- set minute_string = "1 minute" -%}
      {%- elif window_minutes > 1 -%}
        {%- set minute_string = ( window_minutes | string) + " minutes" -%}
      {%- endif -%}
      {# then combine the hours and minutes #}
      {%- set delta_string = hour_string + ( " and " if ( hour_string != "" and minute_string != "" ) else "" ) + minute_string  -%}
      {%- if delta_string == "" -%}
        {# this should never happen, but will put something just in case #}
        "Couldn't find an alarm"
      {%- else -%}
        "Couldn't find an alarm over the next {{ delta_string }}"
      {%- endif -%}

    {%- else -%}
      {# an alarm was found so we indicate that #}

      {%- set window_timedelta = timedelta( hours = 2 ) -%}
      {%- set alarm_timedelta = timedelta(hours=alarm.delta.hours, minutes=alarm.delta.minutes, seconds=alarm.delta.seconds) -%}
      {%- if alarm_timedelta <= window_timedelta -%}
        {# we are less than the two hour delta #}
        {# calculate strings to indicate the hours / minutes until ringing #} 
        {# but words change depending IF / HOW MANY hours / minutes we have#}
        {# so first the hours #}
        {%- set hour_string = "" -%}
        {%- if alarm.delta.hours == 1 -%}
          {%- set hour_string = "1 hour" -%}
        {%- elif alarm.delta.hours > 1 -%}
          {%- set hour_string = ( alarm.delta.hours | string) + " hours" -%}
        {%- endif -%}
        {# then the minutes #}
        {%- set minute_string = "" -%}
        {%- if alarm.delta.minutes == 1 -%}
          {%- set minute_string = "1 minute" -%}
        {%- elif alarm.delta.minutes > 1 -%}
          {%- set minute_string = ( alarm.delta.minutes | string) + " minutes" -%}
        {%- endif -%}
        {# then combine the hours and minutes #}
        {%- set delta_string = hour_string + ( " and " if ( hour_string != "" and minute_string != "" ) else "" ) + minute_string  -%}
        {%- if delta_string == "" -%}
          {# if no hour or minute, it means it is about to go off #}
          "Alarm rings in less than a minute" 
        {%- else -%}
          "Alarm rings in {{ delta_string }}"
        {%- endif -%}

      {%- else -%}
        {# the time format on Sonos alarms is HH:MM:SS in 24-hour format #}
        {# so we extract the hours and minutes and make it 12-hour based #}
        {# and add the AM or PM #}
        {%- set alarm_time_parts = alarm.time.split(':') -%}
        {%- set alarm_time_hour = alarm_time_parts[0] | int -%}
        {%- set alarm_time = ( alarm_time_parts[0] if alarm_time_hour < 13 else ( alarm_time_hour - 12 ) | string ) + ":" + alarm_time_parts[1] + ( " AM" if alarm_time_hour < 13 else " PM" ) -%}
        
        {% if today_at(alarm.time) < strptime(alarm.call_timestamp, "%Y-%m-%d %H:%M:%S.%f%z") %}
          {# this means the alarm goes off tomorrow sometime, so add the tomorrow #}
          "Alarm set for tomorrow at {{ alarm_time }}"
        {%- else -%}            
          {# this means the alarm goes off today #}
          "Alarm set for {{ alarm_time }}"
        {%- endif -%}
      {%- endif -%}
    {%- endif -%}

sequence:
  - service: script.sonos_say
    data:
      entity_id: "{{ actual_announce_entity_id }}"
      message: "{{ message }}"
      volume_level: "{{ volume_level|default(None) }}"
      min_wait: "{{ min_wait|default(None) }}"
      max_wait: "{{ max_wait|default(None) }}"

mode: parallel
max_exceeded: silent
icon: mdi:account-voice
````

# Revisions #
* _2022-10-28_: Added optional playback speaker and added better handling of optional parameters for volume and waits
* _2022-10-16_: Simplified implementation and now depends on the [Sonos Text-to-Speech script](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842) script allowing it take advantage of improvements
* _2022-05-13_: Added setting volume and fixed bug when Sonos reports `recurrence` as `WEEKDAYS` and `WEEKENDS` 
* _2022-05-07_: Initial release

# Available Blueprints #
* Automation: [Lutron Pico Media Remote for Contextually Handling Music or Xbox on Sonos Speakers](https://community.home-assistant.io/t/lutron-pico-media-remote-for-contextually-handling-music-or-xbox-on-sonos-speakers/484748)
* Automation: [Automation for Sonos + Lutron Picos to Magically Manage Sleep Alarms and Music](https://community.home-assistant.io/t/automation-for-sonos-lutron-picos-to-magically-manage-sleep-alarms-and-music/426477)
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Random Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)