# Add here the list of triggers that you want to activate the intrusion alarm when you are away. 
#
# This list will typically include all the door/window sensors and all the motion sensors.

alias: 'Intrusion: Trigger list -  Armed Away'
description: 'Defines the triggers that cause the alarm to activate when in Armed Away state.'
trigger:
  - platform: state
    entity_id: group.door_sensors
    to: 'on'
  - platform: state
    entity_id: group.motion_sensors
    to: 'on'
  - platform: state
    entity_id: group.window_sensors
    to: 'on'
condition:
  - condition: state
    entity_id: alarm_control_panel.home_alarm
    state: armed_away
action:
  - service: alarm_control_panel.alarm_trigger
    entity_id: alarm_control_panel.home_alarm
  - service: notify.notify
    data:
      message: Alarm should trigger
mode: single
