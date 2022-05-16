This blueprint is used to add a script that will announce when the next alarm is set on the specified Sonos speaker. If the alarm is less than two hours away it will indicate how long until it rings. Otherwise the alarm will indicate the time when the alarm is set using a 12 hour format (e.g. 1 AM vs 1 PM). If it cannot find an alarm it will indicate that as well.

It will gracefully handle when speakers are in groups, and restore playing music after it announces the alarm.

The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker (that the alarms are attached to)
* _[optional]_ Time window to look for an alarm to announce. Maximum time is 1439 minutes (23 hours and 59 minutes), minimum is 30 minutes and the default is 720 minutes (12 hours)
* _[optional]_ Volume to used to play the announcement. This only impacts the indicated speaker, not the group.

The following blueprint inputs can be given when creating the script:
* _[optional]_ Text-to-Speech Service to use to make the announcements. Default is `google_translate_say`

This script is particularly convenient when:
* You don't want to check phone apps when going to bed to see if/when an alarm is set
* You have have a phyiscal button near your bed (e.g. a Lutron Caseta Pico remote)

## Additional notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fannounce_sonos_alarm.yaml)

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
# - script-based optional playback vs alarm speaker
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
  input:
    tts_service_name:
      name: Text-To-Speech Service Name
      description:
        The text-to-speech service to use to announce the alarm. This must match your Home
        Assistant configuration.
      default: "google_translate_say"
      selector:
        text:

fields:
  entity_id:
    description: The entity id of the Sonos speaker to find the alarm and then announce the alarm on.
    name: Entity
    required: true
    selector:
      entity:
        domain: media_player
        integration: sonos
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
  # oddly you can't get blueprint inputs directly into jina so putting the
  # input into a variable so then in the engine variable I can verify values
  tts_hack: !input tts_service_name
  tts_engine: >-
    {%- if tts_hack is undefined or tts_hack== None or tts_hack == "" -%}
      tts.google_translate_say
    {%- else -%}
      tts.{{ tts_hack }}
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

  # play the appropriate message, whether we found an alarm or we not
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ alarm.found == false }}"
        sequence:
          # didn't find alarm so indicate the period we checked over
          - service: "{{ tts_engine }}"
            data:
              entity_id: "{{ entity_group_leader }}"
              message: >
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
                  "Could not find an alarm"
                {%- else -%}
                  "Could not find an alarm over the next {{ delta_string }}"
                {%- endif -%}

    default:
      # found an alarm so we we indicate when the alarm will go over BUT we
      # say different things if within two hours, same day, or tomorrow
      - service: "{{ tts_engine }}"
        data:
          entity_id: "{{ entity_group_leader }}"
          message: >
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
      seconds: 4
  # then we wait for it to finish announcing before we continue
  - wait_template: "{{ states( entity_id ) != 'playing' }}"
    timeout:
      seconds: 20 # timeout so doesn't sit forever, but longer to ensure words are said

  # and now we restore where we were which should cover repeat, what's playing, etc.
  - service: sonos.restore
    data:
      entity_id: "{{ entity_group_leader }}"
      with_group: true

mode: parallel
icon: mdi:account-voice
````
&nbsp;
# Revisions #
* _2022-05-13_: Added setting volume and fixed bug when Sonos reports `recurrence` as `WEEKDAYS` and `WEEKENDS` 
* _2022-05-07_: Initial release

&nbsp;
# Available Blueprints #
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)