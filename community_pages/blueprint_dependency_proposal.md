
# The Problem #
Blueprints are a great way to share scripts and automations. While they can use user created scripts, there is no way to describe dependencies and have them automatically downloaded and installed. As the developer of a blueprint you have two options ...
1. **Document an installation process outlining the dependencies to manually install**  (which increases potential for errors and decreases chances the blueprint will be used)

*or*

2. **Cut and paste code between automations and scripts** (which means cutting and pasting bug fixes and enhancements and republishing impacted blueprints, while inadvertently encouraging others to cut and paste code and never receiving fixes)

Neither are great options for re-usability both for developers trying to manage their own work, and for the community looking to promote sharing, and using existing scripts as building blocks for more complex solutions.

## An Example of the Problem
I wrote this proposal because I came across this dependency problem while creating blueprints. In particular, I created an alarm button handler automation that allows a physical button (e.g. Lutron Pico remote) to do different alarm and sleep music operations during the day on my Sonos speakers (see [here](https://community.home-assistant.io/t/automation-for-sonos-lutron-picos-to-magically-manage-sleep-alarms-and-music/426477)). The intention was for the automation to use three script blueprints I previously created. Each script blueprint solved different problems and each were intended to be used independently:
* Announce an alarm (see [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-announce-upcoming-alarm/419700))
* Stop an alarm from going off (see [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-stop-current-alarm-or-temporarily-disable-upcoming-alarm/417610))
* Play random music (see [here](https://community.home-assistant.io/t/play-random-media-script-w-shuffle-repeat-multi-room-volume-support/426445))

In addition, the announce alarm blueprint would ideally use another script blueprint I created that solved some oddities when trying to use text-to-speech on Sonos speakers (see [here](https://community.home-assistant.io/t/script-for-sonos-speakers-to-do-text-to-speech-and-handle-typical-oddities/424842)).

When I released these blueprints I had to decided which of the two options were 'optimal':
* For the alarm button handler blueprint I decided to keep the alarm announcement, stopping alarm and music playing as separate script blueprints and described how to install each of them. It was too much code to cut and paste into the alarm button handler
* For the announcing of text, I decided to cut and paste the code into the announce alarm script blueprint because it wasn't a huge amount of code, and I belived it would otherwise diminish the chance of someone installing the announce alarm blueprint

# The Simple Proposal #

The V1 of the proposal is simple from the blueprint creator's perspective ... add a section to any blueprint (script or automation) that outlines script blueprints that this blueprint is dependent upon. More capabilities can be added later, but this solves the primary issue. 

A simple example:

````yaml
blueprint:
  name: "Sample Automation"
  description: "Sample Description"
  domain: automation

# the new dependencies section

dependencies: # the following are script blueprints this blueprint requires
  example_script_reference_name:                                        # name used in the actions section
    source_url: https://github.com/sample/example_entity_source_url/    # where to find the script blueprint

# showing the actions using dependencies (no change from today)

action:
  - service: script.example_script_reference_name # the script name referred to in the dependencies 
````
Note: The script reference name (e.g. `example_script_reference_name` above) does not represent the actual name of a script instance in `scripts.yaml`, but is a symbol that maps the blueprint's `dependencies` to the script name used in the `actions` section (as shown above). See "*When a user creates an automation or script from a blueprint ...*" below for how the names map to script instances.

For the blueprint creator, that's all there is to it!

## Mini-FAQ ##
**What kind of blueprints can be dependencies?**

Only `scripts`; they are designed to be re-used. An error should be thrown for any other `domain` types

**So dependencies can have dependencies?**

Yes. They are treated the exact same way.

**Can automation and script blueprints have more than one dependency**

Yes. It is simply another entry in your `dependencies` section. For example:
````yaml
blueprint:
  name: "Sample Automation"
  description: "Sample Description"
  domain: automation

# the new dependencies section

dependencies: # the following are script blueprints this blueprint requires
  example_script_reference_name:                                        # name used in the actions section
    source_url: https://github.com/sample/example_entity_source_url/    # where to find the script blueprint
  another_script_reference_name:                                        # name used in the actions section
    source_url: https://github.com/sample/another_entity_source_url/    # where to find the script blueprint
````


## Future considerations ##
- Allow specifying inputs in the blueprint's `dependencies` sections as defaults 
- Allow forcing "headless" / non-interactive installation (will requires inputs)
- Consider versioning (beyond the URL)
- Allow re-usable Jinja as a blueprint type

# Illustrative Solution for the Example Problem #
Using the proposal, here is how I would implement my alarm button handler. The automation blueprint would depend on three script blueprints `announce_sonos_alarm`, `stop_sonos_alarm` and `play_random_media`:
````yaml
blueprint:
  name: Sonos Sleep / Alarm Handler
  domain: automation


#### for simplicity, removed inputs ###

dependencies: 
  announce_sonos_alarm: 
    source_url: https://github.com/Talvish/home-assistant-blueprints/blob/main/script/announce_sonos_alarm.yaml

  stop_sonos_alarm: 
    source_url: https://github.com/Talvish/home-assistant-blueprints/blob/main/script/stop_sonos_alarm.yaml

  play_random_media: 
    source_url: https://github.com/Talvish/home-assistant-blueprints/blob/main/script/play_random_media.yaml

#### for simplicity, removed variables and triggers

action:
  - choose:
      # 'Wake Up' phase of the automation
      - conditions:
          - condition: time
            after: !input wake_start_time
            before: !input sleep_start_time
        sequence:
            - service: script.stop_sonos_alarm #### the script name referred to in the dependencies ####
            data:
                entity_id: !input speaker_id
                time_window: !input disable_alarm_time_window

    default:
      # 'Sleep' phase of the automation
        - sequence:
            - service: script.announce_sonos_alarm #### the script name referred to in the dependencies ####
            data:
                entity_id: !input speaker_id
                time_window: !input announce_alarm_time_window
                volume_level: !input announce_volume_level

            - service: script.play_random_media #### the script name referred to in the dependencies ####
            data:
                media_content_type: music 
                entity_id: !input speaker_id
                repeat: "all" 
                volume_level: !input music_volume_level
                shuffle: true 
                media_content_ids: >
                {{ music_content_ids }}
````

In addition, the `announce_sonos_alarm` script could now depend on the `sonos_say` script blueprint:

````yaml
blueprint:
  name: Announce Sonos Alarm
  domain: script

#### for simplicity, removed inputs ###

dependencies: 
  sonos_say: 
    source_url: https://github.com/Talvish/home-assistant-blueprints/blob/main/script/sonos_say.yaml

#### for simplicity, removed variables and fields
our
sequence:

#### for simplicity, removed other actions

  - service: script.sonos_say #### the script name referred to in the dependencies ####
  data:
      entity_id: "{{ entity_group_leader }}"
      message: >
        "{{ message }}""

````

# A Potential Implementation #
While the above targets blueprint creators, the following outlines a potential high-level solution for Home Assistant developers; I wanted to think through some of the potential issues and solutions.

## When a user imports a blueprint... ##
1. System downloads the blueprint into a temporary location
2. System looks for dependencies in the downloaded blueprint
3. For each dependency, the system loops back to step 1 to ensure all nested dependencies are downloaded, while confirming the dependency blueprint's `domain` is a `script` (and likely validating against dependency loops)
4. System shows the user the entire set of dependency blueprints being imported, indicating which already exist (using logic that exists today) and allows them to preview any blueprint
5. When the user clicks 'Import Blueprint', the system moves the downloaded blueprints from the temporary locations into the appropriate directories in HA (using logic that exists today)
6. System cleans-up temporary downloads

At this point every blueprint that is needed in the entire dependency chain will be downloaded and should be available in the [http://homeassistant.local:8123/config/blueprint/dashboard](http://homeassistant.local:8123/config/blueprint/dashboard) screen.

## When a user creates an automation or script from a blueprint ... ##
1. When the user clicks to create an automation or script, the system analyzes and stores in memory the entire dependency chain found in all dependent blueprint files
1. For each blueprint dependency, the system adds a visual section to the create script/automation screen
2. In each of those visual sections, the system gives the user the option to either a) use any existing script instance that was previously created (allowing re-use of inputs), or b) create a new script instance and asks for the required inputs. Note: if re-using a script instance, and the script had dependencies, it implies re-using the dependency scripts instances
3. When the user saves the script / automation, the system saves an entry in the `scripts.yaml` / `automations.yaml` file like today BUT adds a new `dependencies` section for that script / automation
4. Into this new `dependencies` section, the systems stores an entry that maps each dependency's script reference name to the name of the actual script instance (as found in the `scripts.yaml`) that will be used. Here is an example entry in the `automations.yaml` showing a blueprint with a dependency :
````yaml
- id: '100000'
  alias: Sample Automation
  use_blueprint:
    path: Source/automation_blueprint.yaml
    dependencies:
      example_script_reference_name: example_script_instance_name # `example_script_instance_name` must exist in scripts.yaml
````
Again, this means the `scripts.yaml` must have an entry for `example_script_instance_name`. For example:
````yaml
example_script_instance_name: # matches the name in the above automation's dependencies list
  alias: Example Script
  use_blueprint:
    path: Source/script_blueprint.yaml
  mode: parallel
  max: 10
````

## When the system restarts or reloads scripts / automations into memory for execution ... ##
1. When the system loads the `scripts.yaml` and `automations.yaml` files, it stores a symbol dependency map in memory for each automation and script to know how to map to the correct instance
2. The system ensures the script name can be found in the `scripts.yaml` 
3. The system verifies that the blueprint found at the `path` URL contains a `source_url` that matches the `dependencies`'s `source_url`, to ensure we have the right script. Continuing the example from above, here is the automation blueprint's file (showing a dependency with a `source_url`):

````yaml
blueprint:
  name: "Sample Automation"
  domain: automation

dependencies: # the following are script blueprints this blueprint requires
  example_script_reference_name:                                        # name used in the actions section
    source_url: https://github.com/sample/example_entity_source_url/    # where to find the script blueprint
````
And here is the associated `Source/script_blueprint.yaml` file:
````yaml
blueprint:
  name: Example Script 
  domain: script
  source_url: https://github.com/sample/example_entity_source_url/ 
````
Notice that the `source_url`s are the same.

And that's the overview. Happy to discuss and provide more clarity.
