This blueprint is used to add a script that will announce when the next alarm is set on the specified Sonos speaker. If the alarm is less than two hours away it will indicate how long until it rings. Otherwise the alarm will indicate the time when the alarm is set using a 12 hour format (e.g. 1 AM vs 1 PM). If it cannot find an alarm it will indicate that as well.

It will gracefully handle when speakers are in groups, ensure the message doesn't repeat regardless of the state of the speeaker, and restore playing music after it says the message.

The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker (that the alarms are attached to)
* _[required]_ Text of the message to say
* _[optional]_ Volume to used to play the announcement. This only impacts the indicated speaker, not the group.

The following blueprint inputs can be given when creating the script:
* _[optional]_ Text-to-Speech Service to use to say the message. Default is `google_translate_say`

This script is particularly convenient when:
* You want to announce something on a Sonos speaker but want the speakers to continue what they were doing
  beforehand (e.g. playing music)

## Additional notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fsafe_sonos_alarm.yaml)

````yaml

````
&nbsp;
# Revisions #
* _2022-05-22_: Initial release

&nbsp;
# Available Blueprints #
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)