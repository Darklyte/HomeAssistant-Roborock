# HomeAssistant-Roborock
Improved automation for roborock vacuums via home assistant and node-red

## Requirements
* Home Assistant
* Node-red
* Roborock Vacuum associated with the Mi Home app
* Home Assistant Community Store (Optional, for mapping)

## Initial Setup
Follow the instructions here to get your Roborock Vacuum integrated with Home Assistant: https://www.home-assistant.io/integrations/xiaomi_miio/

Next you will need to set up several entities in home assistant. You can do this through Configuration -> Helpers, but since there are so many I recommend adding them to your configuration.yaml manually instead. There will be three entities for each room and five entities for each vacuum.

### Rooms 

All of my entities in home assistant follow the same format: [\[Type\]] [Room] [Purpose]. Since these are all designed to track how dirty/clean a room becomes, I do [Debris] Room: Purpose.  Create all three of these entities for each finite area you want to track.

```
input_number:
  # Measures how dirty a room becomes. Is adjusted automatically by node-red flows, but can be manually adjusted in large steps by the user. Room is cleaned in one cycle for each level of debris.
  debris_kitchen_bar_level:
    name: "[Debris] Kitchen Bar: Level"
    icon: mdi:broom
    min: 0
    max: 3
    step: 0.5
  # Tracks how often a room should be vacuumed per week.
  debris_kitchen_bar_frequency:
    name: "[Debris] Kitchen Bar: Frequency"
    icon: mdi:sine-wave
    min: 0
    max: 14
    step: 1
    unit_of_measurement: times per week
  # Tracks the area of the room. Used to determine how effective a cleaning cycle was. Maximum size can be adjusted. I chose 20m² as my upper limit since my largest measured area is 19m² This value will be set by the vacuum the first time it cleans the room.
  debris_kitchen_bar_area:
    name: "[Debris] Kitchen Bar: Area"
    icon: mdi:move-resize
    min: 0
    max: 20
    step: 1
    unit_of_measurement: m²
```



#### Vacuums 
It is important that these counters contain your robot vacuums name. Specifically, what is the vacuum entity named? Mine are named vacuum.roborock_maxine. The beginning of these entities should include everything after "vacuum." and be followed by _bin_time, _tank_time, and _mop_time. Essentially I consider these as part of the vacuum device and so they are named the same way.

I use counters because it gives integer values. You can use input_numbers, but you always end up with a float and none of these are recorded in decimals.

```
counter:
  roborock_maxine_bin_time:
    name: "[Vacuum] Roborock Maxine: Bin Time"
    icon: mdi:delete-circle
    minimum: 0
    maximum: 300
    step: 1
  roborock_maxine_tank_time:
    name: "[Vacuum] Roborock Maxine: Tank Time"
    icon: mdi:cup-water
    minimum: 0
    maximum: 300
    step: 1
  roborock_maxine_mop_time:
    name: "[Vacuum] Roborock Maxine: Mop Pad Time"
    icon: mdi:food-croissant
    minimum: 0
    maximum: 300
    step: 1
  roborock_maxine_cleaning_queue
	name: "[Vacuum] Roborock Maxine: Cleaning Queue
	icon: mdi:tray-alert
	minimum: 0
	maximum: 10
```

This min_max entities helps determine which room is the dirtiest and should be cleaned next. Include all of the debris levels that the vacuum could potentially clean. Adjust it based on your rooms.

```
sensor:
  - platform: min_max
    name: debris_roborock_maxine_next_room
    type: max
    entity_ids:
      - input_number.debris_kitchen_prep_level
      - input_number.debris_kitchen_bar_level
      - input_number.debris_foyer_level
      - input_number.debris_living_room_level
      - input_number.debris_dining_room_level
```

Once you've added all the entities, go into lovelace dashboard and set the desired frequency for each room. All of the other entities will update themselves automatically with the node-red flows.

### Helper Entities 
These are entities that are likely unnecessary, but I use regularly and are referenced in the node-red flows. You can just as easily remove them from the flows.

```
input_boolean:
  # Toggled by presence sensors. True if anyone is home. False if everyone is away.
  location_home:
    name: [Location] Home
  # Controlled by motion sensors. Set to true when it is night (after 10pm and no motion on main floor). Set to false at 6:45 AM
  location_night:
    name: [Location] Night
```  

## Node-Red Integration

If you want to jump right in, import `All-necessary-flows.json` into your node-red. Pretty much every single node has comments written, and there are some comment nodes to describe the purpose of each group. For a more detailed setup, continue reading and you can important each group separately.

### Setting up Room Segments 


This is the hardest part of the setup. Assuming you have the Xaiomi integration working and your rooms divided as you want in the Mi Home app, you now need to determine the values for each room segment. The best way I've found to do this is to have a short flow in node red that attempts to clean a room. The Mi Home app will highlight the room is it cleaning in color while the rest of the map is greyed out. If the map is entirely greyed out, the room segment is not used.

Room segments start from 1 and I have not seen any above 30. Each time you split or merge a room, the new room(s) take the next available segment number.

To help find the rooms, you can import the code from `0-segment-finder.json` into node-red. This flow will increment through every room segment, commanding the vacuum to clean it. Keep your Mi Home app open on your phone and watch for which room it is cleaning. label each segment with your room number (save this information for later).

The `0-segment-finder.json` is not included in `All-necessary-flows.json`

### Auto-Incrementing Debris

Note: If you imported `All-necessary-flows.json`, this flow should appear near the very bottom.

Import `Auto-Increase-Debris.json` into node-red. This flow automatically runs every 60 minutes and pulls the debris_level of each room, then adds a value to it based on the room_frequency. For example, if you set `input_number.debris_kitchen_frequency` to 7.0 (times per week), `input_number.debris_kitchen_level` will reach 1.0 seven times per week, i.e. every day.

The `input_number.debris_xxx_level` are designed to keep track of how dirty a room is expected to get over the course of the week. 

Assuming you used my naming convention, you should not have to modify this flow at all.


### Universal Triggers

Import `1-vacuum-triggers.json` into node-red. This group controls the universal initial triggers for the vacuums. You will need to edit the function and the switch to match your vacuum names (see comment nodes and comments in the function's code). 

The vacuum is triggered every 5 minutes if you are home (assuming you choose to keep `input_boolean.location_home`), Whenever the cleaning queue is increased from 0, when the vacuum is returning to its based, and when the location mode is changed to away. When one of these conditions is met, the flow determines which vacuum was triggered then checks if the vacuum is cleaning or errored. If it is not, it continues to check if it is appropriate to vacuum, sending the payload along a flow determined by the vacuum.

### Vacuum-specific Restrictions

There is no file for vacuum specific restrictions. Each of my vacuums has its own specific restrictions and you might want those for your vacuum as well.

Attached the `link-in` node to the `link-out` node from the previous flows based for  each vacuum. I mostly use `current state` nodes to see if it is appropriate for the vacuum to run, such as checking if motion has been inactive for 20 minutes, if people are home, or if it is night. If you want to skip this, you can link everything directly into the next flow.

### Determine Room to Clean

Import `3-determine-room-to-clean.json` into node-red. Attached the `link-in` node to the `link-out` node from the previous flows. You will need to edit the function node here to add your room segments. Use my room segments as examples for the naming format: The segments here should be all lowercase with underscores in place of spaces. The number is the appropriate segment as determined from the Mi Home app.

This flow checks the `sensor.debris_{{vacuum_name}}_next_room` entity to determine which room is due next for cleaning. If the vacuum's cleaning queue is greater than 0, it cleans that room. Otherwise the room must have at least 1.00 debris level to be cleaned. The flow then pulls the room segment and continues to the next section.

### Choose Fan Speed and Begin Cleaning

Import `4-select-fan-speed-and-start.json` into node-red. Attached the `link-in` node to the `link-out` node from the previous flows. No customization should be necessary for this group, unless you wish to remove the optional `input_boolean`s

The flow uses both optional `input_boolean`s to determine the fan speed. The value will be sent to the vacuum (as a word) and is stored in flow context (as a number). The fan speed is set, the vacuum's cleaning queue is decreased (to a minimum of 0), and the vacuum is told to clean the room. If no segment is called, it will clean the entire floor.

### Log Completed Cleans

Import `5-Log-Completed-Cleans.json` into node-red. No customization is necessary in this group.

When the vacuum reports that it is returning to the dock, this flow pull context variables to determine which room was being cleaned, what the fan speed it was using, how much of the room was cleaned, if the vacuum was mopping, and if the vacuum was mopping. The flow then calculates how much of the target room was cleaned and reduces the debris level based on that and the fan speed. 

If the area cleaned is greater than `input_number.debris_{{room}}_area`, the flow will adjust the room size automatically if the size is within 25% of the previous size. Otherwise, it will prompt if you want to increase the size.

If the area cleaned is less than 25%, it considers the room a failure and sets the debris level to 0 so the vacuum does not repeatedly attempt to clean the same room. The error handler will reset the debris level later, after it believes you have had a chance to fix the problem.

The bin time, mop time, and tank time are logged if approriate, and a cleaning report is built. You will receive this report when one of the optional input_booleans change. I like receiving my reports when I wake up and when I get home.

### Error Handler 

Import `6-Error-Handler.json` into node-red. Connect the `Error-Handler link-out` from the previous section into the `link-in` here. Add your `person.` entity to the `Arriving or Leaving Home` node near the top. 

This node handles the edge cases of vacuuming. Particularly, if the area cleaned was greater than 100% or less than 25%. If the cleaned area was less than 25%, the previous flow set the debris level to 0.00 to avoid repeated cleaning attempts. The error handler then waits for you to arrive/leave home, at which point this value is reset as it believes you have had a chance to fix the issue. If the area cleaned was greater than 100% but less than 125%, it automatically increases the known size of the room. If it is greater than 125%, it will ask you if you want to increase the size of the room.

The notification subflows are designed for android. They are based on [This subflow](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/cookbook/actionable-notifications-subflow-for-android.html#demo-flow]), slightly modified. 

### Maintenance Logging

Import `7-Reset-Counters.json`. No modifications are necessary unless you use an iPhone (sorry, you're on your own with that)

