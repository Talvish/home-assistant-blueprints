This script blueprint will say a message on Sonos speakers. The script
handles oddities I've observed when attempting to do so including saving/restore state,
handling speaker groups, pausing music, disabling repeat, etc.
* Saving the state of the Sonos and restoring when done (so music will stop and continue)
* Handle when speakers are in groups (in old and newer versions of HA)
* Pausing music so volume adjustments don't impact current music
* Disable repeat so that the announcement doesn't repeat (and adding delays to handle setting correctly)
* Adaptive delays to minimize chance of cut-offs


The following field parameters can be given when the script is called:
* _[required]_ Sonos speaker (that the alarms are attached to)
* _[required]_ Text of the message to say
* _[optional]_ Volume to used to play the message. This only impacts the indicated speaker, not the group.
* _[optional]_ Maximum number of seconds that the system will wait for the nessage to play

The following blueprint inputs can be given when creating the script:
* _[optional]_ Text-to-Speech Service engine to use to say the specified message. Default is `google_translate_say`.
* _[optional]_ Language to say the message in. Default is `en`.

This script is particularly convenient when:
* You want to announce something on a Sonos speaker but want the speakers to continue what they were doing
  beforehand.

## Additional notes ##

* I recommend setting the mode to `parallel` when using the same script for multiple automations
&nbsp;

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fscript%2Fsafe_sonos_alarm.yaml)

````yaml

````
&nbsp;
# Revisions #
* _2022-05-25_: Initial release

&nbsp;
# Available Blueprints #
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)