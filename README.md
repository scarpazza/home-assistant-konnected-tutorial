# Creating a smart alarm system in Home Assistant with Konnected Alarm Pro boards

This is a minimalistic, step-by-step tutorial on how to bring up a full-featured alarm system based on the Konnected Alarm Pro board in Home Assistant.

## Motivation
I wrote this tutorial because I found existing documentation lacking in their coverage of the distinction between the functions built into Home Assistant's `manual` alarm panel and those you must add yourself to bring up a functional system. New users of Konnected.io with no prior Home Assistant experience seem especially puzzled by `manual`'s documentation. 

## Audience
My ideal reader is someone who just unboxed their Konnected boards and wants to integrate them into an existing Home Assistant setup, but has no prior experience with HA's alarm panels. This tutorial has many limitations and teaches by example rather than covering concepts in depth. That's by design: it's intended for audiences who prefer understanding a design as they bring it up, as opposed to mastering all concepts before writing any configuration code. 

I *learn with my hands*. If you are like me, this tutorial might be a more useful initial starting point than HA documentation.

My design decisions are somewhat arbitrary, and tuned to my very personal needs. Change them according to your needs. 

All feedback is welcome, but I'll make corrections only workload and family permitting. I have a full time job, two kids and other hobbies.

## Acknowledgments 
Many thanks to @johnny-co for his help in testing and fixing my typos.

## Step 1 - configure the boards

Start configuring the Konnected boards from the Konnected mobile phone app including inclusion into your home wi-fi network. While this step is pretty straightforward and already documented plenty elsewhere, there is one important caveat that has to do with your phone's OS.

At one point during the setup of a new board, the Konnected app will require you to join the konnected-<device id> wi-fi, and will open up your phone's wi-fi settings page for you to do so.

On Android phones, pay special attention to your phone's notifications, including one saying
```
Wi-fi has no internet access
Touch for options
```
Do touch that notification, which leads to a dialog box informing you that
```
This network has no internet access. 
Stay connected? [No] [Yes]
```
You must select the checkbox "Don't ask again for this network" and tap "Yes". 
Failing to do so might well cause your Android phone to revert automatically to your home network (without warning), thus preventing communication with the Konnected board.

You now have the options to configure Zones and Sensors in the Konnected app. You only need to do so if you plan to register your board with the Konnected Cloud, which is only necessary if you plan to integrate it into Samsung's SmartThings.

The decision is up to you: the device can be operated in the SmartThings and Home Assistant ecosystems concurrently. If you choose to do so, consider taking a screenshot of your configuration page, in order to ensure it is consistent with the Home Assistant configuration you will specify in the next step.


## Step 2 - integration

A. Configure the Konnected integration.

Start this process in the Home Assistant UI by selecting "Configuration", then "Integrations", then "Konnected.io". 
After you provide your Konnected username and password, HomeAssistant must create one device for each board you have.

At this point you need to configure *once again* your zones inside Home Assistant, replicating the same options you selected in the app in the previous step.
(For the avoidance of doubt, I use the word *zone* as a synonym for sensor, as Konnected.io documentation does. This is a disjoint concept from Home Assistant's Zones, that are basically geofencing areas designed to track people's presence.)  

Do it by clicking "Options" on the card associated with each board. This is a rather laborious process: you'll be presented with two pages to configure zones 1-6 and 7-12 (plus outputs), followed by as many additional pages as the zones you enabled. It is at this moment that the screenshot you took in the previous step comes in handy.

Even if it takes extra time, choose good, descriptive names for the zones now, as opposed to choosing poor names that you'll come back to revise later. Home Assistant will create entity names based on your descriptive names. Doing things right the first time will save you time in the end.

Good examples of descriptive names are "Bedroom window sensor", "Living room motion sensor" and "Boiler room CO detector".

B. Configure additional (optional) support systems
Studio Code Server - Tool to easily modify configuration.yaml, scripts.yaml, groups.yaml, automations.yaml, etc.
	- Install via Configuration / Add-ons
	
Sonos - Use Sonos TTS to give verbal warnings, eg. 'Disarm before alarm sounds', 'House Armed', etc.

## Step 3 - create the alarm automaton

In this step you create what Home Assistant calls a "manual alarm control panel".

Contrary to reasonable expectations, this panel only performs a fraction of an alarm panel's typical functions.
You should think of it not as a complete control panel, but rather as a finite state machine or *automaton*. 
(Do not confuse *automaton* with *automation* - we will have instances of both.) 

Specifically, the `manual` control panel automaton has:
  1. a defined collection of states: `disarmed`, `arming`, `armed_home`, `armed_away`, `pending`, `triggered`;
  2. customizable delays between one state and the other, and
  3. user interface code that displays the panel in the dashboard and takes your input.


It is important to realize what the control panel does *not*: it does not include **triggers** and **responses**. 
You'll need to write those yourself:
  1. you will define what triggers will cause a transition from a state to another, except for the arm/disarm UI inputs. The most important among them will be what triggers the alarm;
  2. you will define the responses you want when the alarm enters or leaves certain states. They determines how Home Assistant reacts when the alarm triggers, is armed, is disarmed, become pending, etc.
  
This tutorial covers triggers and responses in the next steps. In this step, you merely create the automaton.
You do so by editing your `configuration.yaml` file. 

You might be surprised by my example creating not one but two panels: one for intrusion and for fire/CO (CO: carbon monoxide detection).

I recommend two distinct panels because you want them to behave differently. Specifically, you may want to arm/disarm the home intrusion alarm automatically according to presence in the house, whereas fire/CO detection should always active (except for exceptional circumstances like maintenance or incident investigation).

Here are the lines I added in my `configuration.yaml`:

```
alarm_control_panel:
  - platform: manual
    name: Home Alarm
    arming_time: 15
    delay_time: 30
    trigger_time: 180
  - platform: manual
    name: Fire and CO Alarm
    arming_time: 0
    delay_time: 0
    trigger_time: 180
````

Relevant configuration options are as follows:
* `arming_time` is the time you have to exit the house once you gave the order to arm the alarm
* `delay_time` is the time you have to disarm the alarm once you come back in the house and trip a sensor, before the alarm triggers
* `trigger_time` is how long your siren will sound (or equivalent trigger action will last) once the alarm triggers

Contrary to most examples you find on the internet, I chose *not* to set up an arming/disarming code. 
Consider requiring a code if you have wall-mounted tablets in the house, to prevent burglars from disarming the system with a simple tap. 
Otherwise, rather focus on fortifying security at the app/website login point, e.g., enabling encryption (https).

You can always come back and tweak these setting later, including setting a code. At that point, you'll know the process enough to naviage options yourself from the HA `manual` documentation.

If you choose to set up a code, the alarm card in the UI will present you with a numeric keyboard to type that code. If you do not, the UI will hide that keyboard.


## Step 4 - define sensor groups

In this step you aggregate your sensors in meaningful groups, creating as many as necessary, with special attention to groups that you don't want to trigger the alarm when you are home at night, and sensors you always want to trigger a response if the alarm is armed.

In my example, I group motion sensors together, door sensors together, and window sensors together.

The main purpose of this step is to simplify trigger rules and sensor testing. This is especially useful when you have many sensors and zones.

Even if you don't care about grouping sensors by type, it's still useful to create at least one group where they all belong: this makes it easier for you write trigger automations.

You do this by adding to your `groups.yaml` file the contents in [sensor-groups.yaml](sensor-groups.yaml).
(Remove the original `[]` contents, if that's all you have in that file.)


## Step 5 - user interface

In this step you add the alarm user interface cards to HA's Lovelace dashboards.

You need to create at least the control panel card. In the example figure below, it's the one called "Home Alarm" in the left column.

In addition to that, I recommend you create a history card that tracks the alarm state in the last 24 hours, displaying it against your presence at home. In the example below, I chart alarm status against me and my wife's presence at home. Presence here is detected via Zone and mobile phone sensors. 

If you don't have your Home Assistant phone app configured yet, I recommend you go configure it now, and definitely before you get to Step 8, during which you will create "smart" automations.

You should also create entity cards depicting the state of the individual sensors, and sensor groups. You'll be using them at least once during the walkaround, but I find it useful to see what's going on in the house.

For illustration purposes, here is my security dashboard at the end of this step: 
![Sample security dashboard](https://i.imgur.com/DAL00GR.png)



## Step 6 - walkaround

Do a walkaround of the house, with your phone in your hand, explicitly triggering all sensors one by one and verifying that each of them behaves as desired.

Finally, trigger the buzzers manually and trigger the siren manually, verifying they behave as desired.

## Step 7 - core automations

In this step you will create automations (alarm triggers and responses) that perform those functions you would expect a traditional, keypad based, 1990s, "dumb" intrusion alarm system to perform: arm, disarm, trigger, sound the siren, send you notifications.

I recommend the automations described below. They are quite a few. Of course my setup is arbitrary and could be done in a gazillion alternate, but this is a relatively comprehensive example and it works well with my mental representation of the alarm state machine. Modify it as you please.

Consider that the when your triggers fire, the alarm does NOT transition to state `triggered`. It goes instead into the `pending` state. This is a the 30 seconds grace period in which you go "oh, f'ck the alarm is on! let me disarm it!" and scramble to reach for your Home Assistant app as you drop your groceries. I sound the buzzer during that period.

Similarly, when press the "arm home"  or "arm away" buttons, the alarm transitions into the `arming` state before it transitions into the respective armed status. That's designed to give you time to exit the house. I also sound the buzzer during that period.

To support automations that rely on time and date, add the following entry under your `sensor:` section in your `configuration.yaml` file:

```
sensor:
  # ...
  - platform: time_date                
    display_options:
      - 'time'
      - 'date'     
      - 'date_time'    
      - 'date_time_utc'
      - 'date_time_iso'
      - 'time_date'
      - 'time_utc'
      - 'beat'             
```
  
I list the automations in order of importance, with the names that I suggest:

* Intrusion: [Trigger response - siren](trigger-response-siren.yaml)
   * in this automation, list what actions you want the system to carry out when the alarm triggers, i.e., when it believes that an actual intrusion is in progress
   * the trigger for this automation is a state transition of `alarm_control_panel.home_alarm` changing to `triggered`
   * do not specify any conditions
   * add two actions are:
     * "Call Service" for service `notify.notify` with a notification message. See the YAML code for the details and feel free to customize as desired
     * a "Device"-type action for device "Konnected Alarm Panel Pro" (or the id of the panel) with action "Turn on Siren" 
       (assuming that you named "Siren" the output terminal connected to your siren)
   * if you have smart lighting controlled by Home Assistant, consider adding here actions to turn on those indoor and outdoor lights you'd want on if a burglary was in progress.

* Intrusion: Response - Buzzer
  * consider making a duplicate of the previous action, from which you will replace the siren with the buzzer. The rationale is to create a rule that is identical to the full trigger response, but does not use the siren.
  * You'll use this automation to test that the alarm is triggering during those hours in which you can't operate a siren.
  * For the purpose of those tests and ONLY during those tests, you will leave this automation enabled, and the previous one disabled.

* Intrusion: [Trigger list - Armed Away](trigger-list-away.yaml)
  * in this rule, you add Three triggers:
    * if the State of `group.door_sensors` changes to `on`, or
    * if the State of `group.window_sensors` changes to `on`, or
    * if the State of `group.motion_sensors` changes to `on`;
  * under Conditions, select that State of `alarm_control_panel.home_alarm` is `armed_away`;
  * under Actions
    * add an action of type "Call Service", specifying service `alarm_control_panel.alarm_trigger` for entity `alarm_control_panel.home_alarm` 
      * leave Service Data empty; 
    * add an action of type "Call Service" specifying service `notify.notify` with Service Data `message: Intrusion Alarm is triggering now` or a message of your choice;
  * pay attention to the fact that that this automation only causes the alarm state machine to transition from `armed` to `pending`. This automation defines no response. I leave that to a separate rule, below.

* Intrusion: [Trigger list - Armed Home](trigger-list-home.yaml)
  * Create this rule by duplicating the previous one
  * in this rule, you will remove all triggers associated with indoor movement
, specifically you remove the motion sensor group.
  * The rationale is that when you are home and arm the alarm (e.g., for the night), you still want the alarm to trigger if a door or window opens, but not if people move around inside the house.
  * Under Conditions, select that State of `alarm_control_panel.home_alarm` is `armed_home`;
  
* Intrusion: buzz when arming
  * this is a very simple rule
  * when the State of `alarm_control_panel.home_alarm` changes to `arming`
  * leave Conditions empty,
  * under Actions, add one action
    * of type Device,
    * for device "Konnected Alarm Panel Pro" (or the name you chose for the integration),
    * and Action "Turn on Buzzer" (assuming that you named "Buzzer" the output terminal connected to your buzzer).
* Intrusion: stop buzz when armed
  * this is a mirror image of the previous rule. Create it by duplicating the previous one
  * the Trigger is a change in the State of `alarm_control_panel.home_alarm` from `arming` (leave the "to" field empty),
  * the Action is "Turn off the buzzer".

* Intrusion: [Pre-trigger warning](pre-trigger-warning.yaml)
  * This automation controls how the system behaves during the grace period between the time it detects and intrusion and the time it triggers
    * This grace period is controlled by property `delay_time` in your `configuration.yaml`. 
    * A typical value is 30 seconds.
    * The grace period is intented for legitimate occupants to disarm the system once they realized they entered the home without first disarming the system.
  * Triggers: when the State of `alarm_control_panel.home_alarm` becomes `pending`,
  * Actions:
    * send an appropriate notification (see the YAML file for details)
    * turn on the buzzer
    * announce that the alarm is about to go off via smart speakers using text to speech features

* Intrusion: [Disarm response](disarm-response.yaml)
  * Add to this automation all the actions you want the system to carry out when the system is disarmed.
  * Triggers: when the State of `alarm_control_panel.home_alarm` becomes `disarmed`
  * No conditions
  * Actions:
    * turn off the siren,
    * send an appropriate notification
    * announce that the system is disarmed via smart speakers using text-to-speech features;
    * turn on the buzzer
    * add a 1-2 seconds delay, then
    * turn off the buzzer.
  * Make sure that you define a set of actions that handle cleanly a transition from any possible state (from `armed`, `pending`, or `triggered`), because buzzer and siren states differ depending on the previous system state. 
  * The actions I chose will first turn off the siren so that this disarm response also acts as a "clear-all". 
  * I also have a notification, a smart speaker text-to-speech announcement and a buzzer feedback. 
  * The final "turn off buzzer" action finally catches any buzzer activity started in the pending state.

### Text to speech
In the previous sections I mentioned text-to-speech announcements associated with alarm status transitions.
At home I have Sonos smart speakers, and I use those for the announcements.
I play those announcements via actions relying on a [Sonos TTS script](sonos-tts-script.yaml) that is designed to pause the music being currently played on the speaker, get the announcement wavefile as rendered by a Google TTS call, play the announcement, and restore the smart speaker state (which will resume playing any music it was playing before the announcement, if any).
  
You will need the following entries in your `configuration.yaml` file to use Google's TTS interface:
 
```
tts:
  - platform: google_translate               
    cache: true
    cache_dir: /tmp/tts-cache    
    time_memory: 36000
    base_url: https://your_domain_name.duckdns.org:8123   
```
  
Caveats:
1. While the rest of the intrusion alarm is entirely self sufficient and able to work locally 
   (including during network loss intentionally cause by burglars), this TTS function relies on 
   access to Google. There are alternative TTS engines you can configure locally, but I haven't explored them yet.
2. I manually created `/tmp/tts-cache` on my linux box where I run Home Assistant Core. 
   If you run on a different platform, adapt the cache directory accordingly.
3. Provide your external URL as a parameter to `base_url`.
   You can set up external HTTPS access following [this tutorial](https://github.com/scarpazza/home-assistant-cookbook/blob/main/https.md).
  
  
  
## Step 8 - geofencing automations

You should now consider smarter automations, primarily alarm geofencing, that Home Assistant can perform thanks to its app running on a GPS-enabled mobile phone. Geofencing is the most prominent feature that the traditional 1990s systems would not offer. 
  
My philosophy is that I prefer to keep humans in the loop, so my automations do not always arm or disarm the system when your presence
in proximity of the house changes. It rather prompts you with suggestions to arm or disarm.
Of course, the automations I present are just examples, and you should configure yours to suit your unique needs.
  
### Set up people and mobile apps
  
Configuring your people:
  - create one user for each person that you want to be able to control the alarm system
  - set the respective passwords
  - install the Home Assistant mobile app on each phone
  - log in on each phone with the respective user-password pairs you set up
  - in *People* configuration, populate the "Track Device" fields, associating to each person the respective device.
  
Note that device tracker entities become available in Home Assistant only after you have configured each mobile app.
  
### Define a family group
  
You now want to define a family group for presence-tracking purposes, by adding the following to your `groups.yaml` file.
```
family:
  name: Family
  entities:
  - device_tracker.person1
  - device_tracker.person2
  - device_tracker.person3  
```  
  
This will act as your family group for the purpose of determining if someone is at home.
Ideally, the system should prompt you to arm as soon as nobody is home, and remind you to unarm as soon as someone is coming home.
The `family` group works well for that purpose:
- each device tracker is either in a `home` or `not_home` state
- [Groups](https://www.home-assistant.io/integrations/group) in Home Assistant by default 
  have a state that is the "logical OR" of the states of the member entities;
  (you can change it to a logical AND, but you shouldn't in this context);
- this means that the group will be in state `home` when at least one of the trackers are home;
- and the group will be in state `not_home` when all of the trackers are not at home.
  
### Write your geo-fenced automations
  
Once your family group is established, I recommend the following automations:

* Intrusion: [auto-arm at bed time if someone is home](auto-arm.yaml)

* Intrusion: [auto-disarm the system at your usual wake-up time](auto-disarm.yaml)
  
* Intrusion: disarm reminder - returning home
  * sends you a notification reminding you to disarm the system 
  * if you are coming home 
  * and the system is armed (either `armed_home` or `armed_away`).
  
* Intrusion: [Suggest arming the system when everybody has left the home](suggest-arm-everybody-left.yaml)
  * whenever all registered users leave the home, it sends you an [Actionable Notification](https://companion.home-assistant.io/docs/notifications/actionable-notifications/) prompting you to arm the system 
  * an Actionable Notification is a message that your phone's OS displays together with buttons or links to perform associated actions
  * we give users the option to do nothing, in case they knew that there are untracked  legitimate occupants staying at home (e.g., relatives, babysitters, etc.). 
  * if you know with certainty that you will never leave the home to non-tracked occupants, you can change short-circuit this automation so that it arms the system.
  
* Intrusion: [Suggest arming the system when nobody is home at night](suggest-arm-late-nobody-home.yaml)
  Suggest arming (away) at a given time in the evening if no member of the family is home

* Intrusion: [Notification action responses](notification-action-responses.yaml)
  * This is just internal machinery designed to process the user requests coming via
    action prompts presented via *Actionable notifications*, described above. 
  * You need to separate handlers, one for `ARM_HOME` and one for `ARM_AWAY`.
    See the YAML source file for details.
  
## Step 9 - repeat for the fire/CO alarm

Repeat Steps 4...8 for the fire and carbon monoxide alarm system.

You may want a separate dashboard, depending on the complexity of your system.

You need a few automations. 
I recommend the following:
* Fire/CO: trigger response - Siren - LEAVE ON EXCEPT WHEN TESTING
* Fire/CO: trigger response - Buzzer
* Fire/CO: disarm response
* Fire/CO: unconditional auto-arm at noon

Ensure that all these rules operate on `alarm_control_panel.fire_and_co_alarm` rather than the home alarm panel we used so far. 

I won't describe them in detail because they are a simplified version of those I already described for the intrusion alarm.

Fire/CO automations are fewer and simpler than those of the intrusion alarm because you basically want fire/CO protection to be always armed, regardless of where you are.
