# HomeAssistant-Roborock
Improved automation for roborock vacuums via home assistant and node-red

## Initial Setup.
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
  # Tracks the area of the room. Used to determine how effective a cleaning cycle was. Maximum size can be adjusted. I chose 20m² as my upper limit since my largest measured area is 19m²
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

=== Helper Entities ===

These are entities that are likely unnecessary, but I use regularly.


input_boolean:
  # Toggled by presence sensors. True if anyone is home. False if everyone is away.
  location_home:
    name: [Location] Home
  # Controlled by motion sensors. Set to true when it is night (after 10pm and no motion on main floor). Set to false at 6:45 AM
  location_night:
    name: [Location] Night
  

	  
== Setting up Room Segments ==

This is the hardest part of the setup. Assuming you have the Xaiomi integration working and your rooms divided as you want in the Mi Home app, you now need to determine the values for each room segment. The best way I've found to do this is to have a short flow in node red that attempts to clean a room. The Mi Home app will highlight the room is it cleaning in color while the rest of the map is greyed out. If the map is entirely greyed out, the room segment is not used.

Room segments start from 1 and I have not seen any above 30. Each time you split or merge a room, the new room(s) take the next available segment number.


