# Creating a smart alarm system in Home Assistant with Konnected Alarm Pro boards

This is a minimalistic, step-by-step tutorial on how to bring up a full-featured alarm system based on the Konnected Alarm Pro board in Home Assistant.

## Motivation
I wrote this tutorial because I found existing documentation somewhat lacking on what exact functions are built into Home Assistant's `manual` alarm panel, and what functions you must add yourself to bring up a functional system. Inexperienced Konnected.io users seem to be especially puzzled by `manual`'s documentation, that does not seem to be designed for begineers. 

## Audience
My ideal reader is someone who just unboxed his Konnected boards and wants to integrate them into a Home Assistant setup, but has no prior experience doing so. This tutorial has many limitations and teaches by example rather than covering concepts in depth. That's by design: it's intended for audiences who prefer understanding a design as they bring it up, as opposed to audiences who prefer mastering Home Assistant concepts before writing any configuration code. 

I *learn with my hands*. If you are like me, this tutorial might be a more useful initial starting point than HA documentation.

My design decisions are somewhat arbitrary, and tuned to my very personal needs. Change them according to your needs. 

All feedback is welcome, but I'll make corrections only workload and family permitting. I have a full time job, two kids and other hobbies.


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

This step configures the Konnected integration.

Start this process in the Home Assistant UI by selecting "Configuration", then "Integrations", then "Konnected.io". 
After you provide your Konnected username and password, HomeAssistant must create one device for each board you have.

At this point you need to configure *once again* your zones inside Home Assistant, replicating the same options you selected in the app in the previous step.
(For the avoidance of doubt, I use the word *zone* as a synonym for sensor, as Konnected.io documentation does. This is a disjoint concept from Home Assistant's Zones, that are basically geofencing areas designed to track people's presence.)  

Do it by clicking "Options" on the card associated with each board. This is a rather laborious process: you'll be presented with two pages to configure zones 1-6 and 7-12 (plus outputs), followed by as many additional pages as the zones you enabled. It is at this moment that the screenshot you took in the previous step comes in handy.

Even if it takes extra time, choose good, descriptive names for the zones now, as opposed to choosing poor names that you'll come back to revise later. Home Assistant will create entity names based on your descriptive names. Doing things right the first time will save you time in the end.

Good examples of descriptive names are "Bedroom window sensor", "Living room motion sensor" and "Boiler room CO detector".


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

I recommend two distinct panels because you want them to behave differently. Specifically, you may want to arm/disarm the intrusion alarm automatically according to presence in the house, whereas fire/CO detection should always active (except for exceptional circumstances like maintenance or incident investigation).

Here are the lines I added in my `configuration.yaml`:

```
alarm_control_panel:
  - platform: manual
    name: Intrusion Alarm
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

You do this by adding the following contents to your `groups.yaml` file.
You might have to remove the original `[]` contents, if that's what you have in that file.

```
motion_sensors:                                                                                                          
    name: Motion Sensors                                                                                                 
    icon: hass:walk                                                                                                      
    entities:                                                                                                            
     - binary_sensor.upstairs_motion                                                                                     
     - binary_sensor.basement_motion                                                                                     
     - binary_sensor.dining_room_motion                                                                                                                                                                      
door_sensors:                                                                                                            
    name: Door Sensors                                                                                                   
    icon: hass:door                                                                                                      
    entities:                                                                                                            
     - binary_sensor.front_door                                                                                          
     - binary_sensor.garage_door                                                                                         
     - binary_sensor.breakfast_corner_door                                                                               
     - binary_sensor.living_room_slides   
     - binary_sensor.living_room_door                                                                                    
     - binary_sensor.basement_door                                                                                  
window_sensors:                                                                                                          
    name: Window Sensors                                                                                                 
    icon: hass:window-closed                                                                                             
    entities:                                                                                                            
     - binary_sensor.breakfast_corner_window                                                                             
     - binary_sensor.dining_room_windows                                                                                 
     - binary_sensor.laundry_room_window                                                                                 
     - binary_sensor.living_room_windows                                                                                 
     - binary_sensor.basement_windows                                                                                    
     - binary_sensor.tv_room_windows      
```


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

I list the automations in order of importance, with the names that I suggest:

* Intrusion: Response - Siren
   * the Trigger is based on the State of `alarm_control_panel.home_alarm` changing to `triggered`
   * leave Conditions empty
   * add two Actions:
     * add an action of type "Call Service" for service `notify.notify` with the following Service Data: `title: Intrusion in progress! message: 'Alarm triggered at {{ states(''sensor.date_time'') }}'`. Customize the message per your preferences
     * add an action of type "Device" for device "Konnected Alarm Panel Pro" (or the name you chose) and Action "Turn on Siren" (assuming that you named "Siren" the output terminal connected to your siren)
   * if you have smart lighting controlled by Home Assistant, consider adding here actions to turn on those indoor/outdoor lights you'd want on if a burglary was in progress.

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
* Intrusion: pre-trigger warning:
  * when the State of `alarm_control_panel.home_alarm` becomes `pending`,
  * Actions:
    * send a notification
      * title: INTRUSION DETECTED
      * message: Disarm the system NOW if you want to prevent the siren from activating.
    * turn on the buzzer
* Intrusion: disarm response
  * When the State of `alarm_control_panel.home_alarm` becomes `disarmed`
  * Actions:
    * turn off the siren,
    * send an appropriate notification
    * turn on buzzer
    * add a 1-2 seconds delay, then
    * turn off the buzzer.
  * The reason why you want these three actions is to handle the diverse incoming transitions (from `armed`, `pending`, and `triggered`). The buzzer and the siren states are different depending on them. Actions will first turn off the Siren, so that this disarm response also acts as a "clear-all". Then, have a 1-2 second buzz to give auditory feedback when the user does a clean disarm. Finally, turn off the buzzer, which also stop the buzzer that started in the pending status.

## Step 8 - smart automations

You should now consider creating "smart" automations, i.e., alarm-related functions that Home Assistant can perform thanks to its app, and that traditional 1990s alarm systems would not offer. Most prominently, this includes presence detection.  

I won't describe them in length because you'll be an automation expert by now. 
I found the following to be useful:

* Intrusion: [auto-arm at bed time if you are home](auto-arm.yaml)

* Intrusion: [auto-disarm the system at your typical wake-up time](auto-disarml.yaml)
  
* Intrusion: disarm reminder - returning home
  * sends you a notification reminding you to disarm the system 
  * if you are coming home 
  * and the system is armed (either `armed_home` or `armed_away`).
  
* Intrusion: reminder to arm when you leave
  * sends you a notification reminding you to arm the system 
  * if you are leaving home and 
  * the system is not `armed`;
  * the reminder gives you the option *not* to arm the system if you know there are non-app occupants home (e.g., relatives, babysitters, etc.). 
  * if you know with certainty that you will never leave the home to non-tracked occupants, you can change this automation so that it arms the system.
  
* Intrusion: [suggest arming the system when everybody has left the home](suggest-arm-everybody-left.yaml)
  
* Intrusion: [suggest arming the system when nobody is home at night](suggest-arm-late-nobody-home.yaml)
  Suggest arming (away) at a given time in the evening if no member of the family is home

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
