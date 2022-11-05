This blueprint creates an automation allowing a **Lutron Caseta** 5 button  Pico remote to contextually 
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
[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTalvish%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Fautomation%2Fsonos_sleep_handler.yaml)

## Additional Notes ##
* This is another of my **favourite automations** since I commonly switch between music and Xbox. While you still need an Xbox controller or media remote to browse content, I find the infrared on the Xbox can be finicky, media remotes aren't physically stable for quick button presses so you have to pick them up (unlike a Pico remote on a stand), and powering up Xbox controllers takes time. So this script is very convenient for quickly selecting, playing, pausing and going back.
* This only requires Sonos speakers because I'm uncertain if other speakers can change their `source` to `TV` and I only have Sonos speakers to test with
* I recommend setting the mode to `single` since this creates a script bound to fixed speakers, Xbox and remote

&nbsp;

````yaml
````

# Revisions #
* _2022-11-05_: Initial release

# Available Blueprints #
* Script: [Automation for Sonos + Lutron Picos to Magically Manage Sleep Alarms and Music](https://community.home-assistant.io/t/automation-for-sonos-lutron-picos-to-magically-manage-sleep-alarms-and-music/426477)
* Script: [Script for Sonos Speakers to do Text-to-Speech and Handle Typical Oddities](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)
* Script: [Script for Sonos Speakers to Announce Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700)
* Script: [Script for Sonos Speakers to Stop Current Alarm or Temporarily Disable Upcoming Alarm](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610)
* Script: [Play Random Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445)
* Script: [Play Media w/ Shuffle, Repeat, Multi-room, Volume support](https://community.home-assistant.io/t/play-media-script-w-shuffle-repeat-multi-room-volume-support/415234)
* Others in my [GitHub repository](https://github.com/Talvish/home-assistant-blueprints)
