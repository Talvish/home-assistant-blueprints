This blueprint creates an automation that listens to a button press on a Pico remote and depending on the time of day does different things. 

If pressed during the `Wake Phase` (time is configurable) it will:
* see if an alarm is playing and if so, stop it, otherwise temporarily disable an alarm that will play soon (how far in time it looks is configurable)
* optionally run an additional action (e.g. it could open blinds, play music, etc)

If pressed during the `Sleep Phase` (time is configurable), it will:
* optionally run an additional action (e.g. turn off lights)
* announce when the next alarm is set to run
* optionally play from a set of music you specify and set a sleep timer

## Additional notes ##
* This blueprint is a bit of an experiment because it uses a good number of the blueprint scripts I created previously, so was very curious how this feels in practice. I'll be sending my thoughts on that later.
* This is my **favourite automation**. It simplifies alarm handling down to one contextual button press. In my configuration, my `Sleep Phase` action includes turning off all downstairs lights and checking to see if the master bathroom is occupied and has lights on, and if so, dims the lights to the lowest setting to minimize light bleed into the master bedroom.


* I recommend setting the mode to `single`
&nbsp;
## Required Blueprints ##
The following is the main blueprint:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_button.yaml)


But it relies on three script blueprints I've created previously...

* Announcing an alarm: 

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_button.yaml)

* Stopping an alarm:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_button.yaml)

* Playing random media:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_button.yaml)


````yaml

````
&nbsp;
# Revisions #
* _2022-05-30_: Initial release

&nbsp;
# Available Blueprints #
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)