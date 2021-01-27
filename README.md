
This is a minimalistic, step-by-step tutorial on how to bring up a full-featured alarm system based on the Konnected Alarm Pro board in Home Assistant.

Motivation: I wrote this because I found existing documentation lacking on what specific functions are offered by the HA alarm panel, and what functions you must add yourself in order to bring up a functioning system.

Audience: this tutorial has many limitations and shows things by example rather than cover concepts in depth. That's by design: it's intended for audiences who are ok understand things as they bring them up, as opposed to audiences who want to master Home Assistant concepts in detail before writing any configuration code. I "learn with my hands". If you are like me, this tutorial might be a more useful initial starting point than HA documentation.

Choices: All my design decisions are arbitrary and tuned to my very personal needs. Change them according to your needs. All feedback is welcome, but I'll make corrections only workload and family permitting. I have a full time job, two kids and other hobbies.

# Step 1 - configure the boards

Start configuring the Konnected boards from the Konnected mobile phone app.This step is pretty straightforward and already documented plenty elsewhere.
Once you are done, take a screenshot of your configuration page, as you will need it in the next step.
(If you chose to use SmartThings rather than Home Assistant, you'd be already almost done at this step.)

# Step 2 - integration

In this step, you configure the Konnected integration.

Start this process in the Home Assistant UI by selecting "Configuration", then "Integrations", then "Konnected.io". After you provide your Konnected username and password, HomeAssistant must create one device for each board you have.

At this point you need to configure AGAIN your zones inside Home Assistant, replicating the same options you selected in the app in the previous step.

Do it by clicking "Options" on the card associated with each board.
It is a laborious process: you'll be presented with two pages to configure zones 1-6 and 7-12 plus outputs, then as many additional pages as the zones you enabled. This is when the screenshot you took in the previous step comes in handy.
Even if it takes extra time, choose good, descriptive names for the zones now, as opposed to choosing poor names that you'll come back to revise later. Home Assistant will create entity names based on your descriptive names. Doing things right the first time will save you time in the end.
Good examples of descriptive names are "Bedroom window sensor", "Living room motion sensor" and "Boiler room CO detector".

# Step 3 - create the alarm automaton

In this step you create the alarm "control panel" in Home Assistant terminology.
Contrary to expectations, this panel performs only a few of the functions you would expect.
Think of it not as a control panel but as a simple finite state machine, i.e., an automaton (spelled like I did, not to be confused with an automation).
The control panel automaton has:
1. a defined collection of states: disarmed, arming, armed_home, armed_away, pending, triggered;
2. customizable delays between one state and the other, and
3. UI code that displays the panel in the dashboard and takes your input.
It is important to realize what the control panel DOES NOT do: it does not include triggers and responses. You'll need to write those yourself.Specifically:
1. you will define what trigger cause each transition from one state to the other, except for the arm/disarm UI inputs. The most important among them will be what triggers the alarm;
2. you will define the response: i.e., how  you want Home Assistant to react when the alarm triggers, is armed, is disarmed, become pending, etc.
I cover triggers and responses in the next steps. In this step, you only configure the automaton.
You do this by editing your configuration.yaml file.
You might be surprised that I create two panels, one for intrusion and for fire/CO. I chose to do that because I want to be able to arm/disarm the intrusion alarm according to presence in the house, whereas I want fire/CO detection to be always active except for exceptional circumstances (e.g., when manually disarmed for maintenance or incident investigation).
Here are the lines I added in my configuration.yaml
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
    
Configuration options are as follows:
- "Arming time" is the time you have to exit the house once you gave the order to arm the alarm
- "Delay time" is the time you have to disarm the alarm once you come back in the house before the alarm triggers
- "Trigger time" is how long your siren will sound (or equivalent trigger action will last) once the alarm triggers
Contrary to most examples you find on the internet, I chose NOT to set up a code. Definitely do that if you plan to leave wall-mounted tablets in the house, else a burglar could disarm the system with a simple tap. Otherwise, do this later and rather fortify security at the app/website login point.
Can come back and tweak these setting later, including setting a code. At that point, you'll know enough the process that you can come back and choose options yourself from the HA "manual" documentation.

# Step 4 - configuring sensor groups

In this step you configure the sensor groups, creating as many groups as necessary.

In my example, I group motion sensors together, door sensors together, and window sensors together.

The only purpose of this step is to simplify trigger rules and sensor testing when you have many sensors and zones.

If you don't care about separating sensors by type, it's still useful to at least put them all in a single group, to simplify trigger automation.

You do this by adding the following contents to your groups.yaml file.You might have to remove the original [] contents, if that's what you have in that file.

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

# Step 5

In this step you add the alarm UI cards to the Lovelace dashboards.
You need to create at least the control panel card.In the example figure below, it's the one called "Home Alarm" in the left column.
In addition to that, I recommend you create a history card that tracks the alarm state in the last 24 hours against your presence. In the example below, I chart alarm status against me and my wife's presence at home. Presence here is detected via Zone and mobile phone sensors. If you don't have your Home Assistant phone app configured yet, I recommend you go configure it now, and definitely before you get to Step 7, where you will create "smart" automations.
You should also create entity cards depicting the state of the individual sensors, and sensor groups. You'll be using them at least once during the walkaround, but I find it useful to see what's going on in the house.
Here is my security dashboard at the and of my Step 4:
Security dashboard example

# Step 6 - walkaround

Do a walkaround of the house, with your phone in your hand, explicitly triggering all sensors one by one and verifying that each of them behaves as desired.

Finally, trigger the buzzers manually and trigger the siren manually, verifying they behave as desired.

# Step 6 - core automations

In this step you will create the core automations (alarm triggers and responses) that perform those functions you would expect a traditional, keypad based, "dumb" intrusion alarm system to perform: arm, disarm, trigger, sound the siren, send you notifications.

I recommend the automations described below. They are quite a few.

Of course my setup is arbitrary and could be done in a gazillion alternate, but this is a relatively comprehensive example and it works well with my mental representation of the alarm state machine. Modify it as you please.

Consider that the when your triggers fire, the alarm does NOT go into "triggered". It goes instead into the "pending" status. This is a the "oh, fuck the alarm is on! let me disarm it!" 30-second grace period. I sound the buzzer during that period.

Similarly, when press the "arm home"  or "arm away" buttons, the alarm transitions into the "arming" state before it transitions into the respective armed status. That's designed to give you time to exit the house. I also sound the buzzer during that period.

I list the automations in order of importance, with the names that I suggest:
- Intrusion: Response - Siren.
 - the trigger is based on the State of alarm_control_panel.home_alarm, specifically if it changes to "triggered".
 - Leave Conditions empty.
 - Add two actions:
  - add an action of type "Call Service" specifying service notify.notify with the following Service Data:title: Intrusion in progress!message: 'Alarm triggered at {{ states(''sensor.date_time'') }}'or a message of your preference.
  - add an action of type Device, for device "Konnected Alarm Panel Pro" (or whatever name you chose for the integration), and Action "Turn on Siren" (assuming that you named "Siren" the output terminal connected to your siren).
  - if you have smart lighting controlled by Home Assistant, consider adding here actions to turn on the lights inside and outside the house as appropriate.
- Intrusion: Response - Buzzer.
 - consider making a duplicate of the previous action, from which you will replace the siren with the buzzer. The rationale is to create a rule that is identical to the full trigger response, but does not use the siren.
 - You'll use this automation to test that the alarm is triggering during those hours in which you can't operate a siren.
 - For the purpose of those tests and ONLY during those tests, you will leave this automation enabled, and the previous one disabled.
- Intrusion: Trigger list - Armed Away**:**
 - in this rule, you add three triggers:
  - if the state of group.door_sensors changes to on, or
  - if the state of group.window_sensors changes to on, or
  - if the state of group.motion_sensors changes to on;
 - under Conditions, select that State of alarm_control_panel.home_alarm is armed_away;
 - under Actions
  - add an action of type "Call Service", specifying service alarm_control_panel.alarm_trigger for entity alarm_control_panel.home_alarm (leave Service Data empty);then add a second action of type "Call Service" specifying service notify.notify with Service Data "message: Intrusion Alarm is triggering now" or a message of your preference.
 - Pay attention that this rule only causes the alarm panel state machine to go from "armed" to "pending". The actual response to a trigger is defined in another rule below.
- Intrusion: trigger list - Armed Home:
 - Create this rule by duplicating the previous one. Then, in this rule, you remove one of the triggers, specifically you remove the motion sensor group.
 - The rationale is that when you are home and arm the alarm (e.g., for the night), you still want the alarm to trigger if a door or window opens, but not if people move around inside the house.
 - Under Conditions, select that State of alarm_control_panel.home_alarm is armed_home;
- Intrusion: buzz when arming
 - this is a very simple rule
 - when the State of alarm_control_panel.home_alarm changes to arming
 - leave Conditions empty,
 - under Actions, add one action
  - of type Device,
  - for device "Konnected Alarm Panel Pro xxx" (or whatever name you chose for the integration),
  - and Action "Turn on Buzzer" (assuming that you named "Buzzer" the output terminal connected to your buzzer).
- Intrusion: stop buzz when armed
 - This is a mirror image of the previous rule. Create it by duplicating the previous one.
 - when the State of alarm_control_panel.home_alarm changes from arming (leave "to" empty),
 - action is "Turn off the buzzer".
- Intrusion: pre-trigger warning:
 - when the State of alarm_control_panel.home_alarm becomes "pending",
 - Actions:
  - send a notification
  - title: INTRUSION DETECTED
  - message: Disarm the system NOW if you want to prevent the siren from activating.
 - turn on the buzzer
- Intrusion: disarm response
 - When the State of alarm_control_panel.home_alarm becomes disarmed
 - Actions:
  - turn off the siren,
  - send an appropriate notification
  - turn on buzzer
  - add a 1-2 seconds delay, then
  - turn off the buzzer.
 - The reason why you want all these actions is to handle incoming transitions from all states (armed, pending, and triggered). The buzzer and the siren states are different depending on them.
 - Actions turn off the Siren so that this disarm response also acts as a "clear-all". Then, have a 1-2 second buzz to give auditory feedback when the user does a "clean" disarm. Finally, turn off the buzzer, which also stop the buzzer that started in the pending status.

# Step 8 - Smart automations
Consider creating additional "smart" automations, i.e., functions that traditional 1990s alarm systems did not have, and that take advantage of Home Assistant app features, most prominently mobile-phone-based presence detection.
I don't describe them in length because you'll be an automation expert by now:
- Intrusion: auto-arm at a given time in the evening if you are home
- Intrusion: auto-disarm the system at 6.30am (or your wake-up time) if it was in the armed_home state and you are home;
- Intrusion: disarm reminder - returning home. Sends you a notification reminding you to disarm the system if you are coming home and the system is armed.
- Intrusion: reminder to arm when you leave. Sends you a notification reminding you to arm the system if you are leaving home and the system is not armed. You might decide not to, if there are occupants home. If you know you never leave guests alone, you can turn this reminder into an auto-arm.
- Intrusion: suggest arming (away) at a given time in the evening if not home. Sends you a notification reminding you to arm the system (armed_away) if you are not home at a given time in the evening.
- ...

# Step 9

Repeat Steps 4...8 for the fire and carbon monoxide alarm system.
You may want a separate dashboard, depending on the complexity of your system.
You need a few automations. I implemented the following:
- Fire/CO: trigger response - Siren - LEAVE ON EXCEPT WHEN TESTING
- Fire/CO: trigger response - Buzzer
- Fire/CO: disarm response
- Fire/CO: auto-arm at noon
All these rules involve the alarm_control_panel.fire_and_co_alarm panel instead of the home alarm panel used so far. I won't describe them in detail because they are a simplified version of those I already described for the intrusion alarm.
Fire/CO automations are fewer and simpler than those of the intrusion alarm because you basically want fire/CO protection to be always armed, regardless of where you are.
