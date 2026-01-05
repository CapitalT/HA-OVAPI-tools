# Home Assistant OVAPI tools for Dutch GTFS API

This project provides a set of YAML code to use **OVAPI** data in Home Assistant without the need of custom components. The new sensors can be displayed as pre-formatted enitities or the attributes can be used for custom formatting as is done in the example **DRIS-style departure board**.

![DRIS departure board](doc/Example_DRIS.png) or ![Example_attributes](doc/Example_attributes.png)


This project might also work for other API's that provide GTFS data.

## Data flow



```mermaid

flowchart LR

    rest("REST sensor") --> Helper("Helper template sensor\n(data.yaml)") --> Departures("Departure template sensor\n(vertrek.yaml)") --> DRIS("picture-elements card") & Entities("Entities card") & Badge(Badge)

```

The REST integration used the OVAPI Timing Point Code (tpc) endpoint to get data and create a sensor for each tpc. This data is preprocessed by one or more helper templates. Once filtered and sorted, the actual OVAPI 'vertrek' sensor comes with a state with the departure information of the next bus/tram/ferry and attributes for the next three options.

This data can then be used in cards, badges or automations.



## Installation

### Prerequisites

- The REST integration needs to be installed
- You need to have access to Home Assistant configuration files

### Blueprints

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FCapitalT%2FHA-OVAPI-tools%2Fblob%2Fmain%2Fblueprints%2Ftemplates%2Fdata.yaml)

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FCapitalT%2FHA-OVAPI-tools%2Fblob%2Fmain%2Fblueprints%2Ftemplates%2Fvertrek.yaml)

Put the blueprints `data.yaml` and `vertrek.yaml` in ```homeassistant/blueprints/ovapi``` using the buttons above or manually.


### Package
Edit the file `ovapi.yaml` in `package` and save it under a suitable name. You can either save this file under `homeassistant/packages` or put the sensors in `configuration.yaml` directly. This is the only data you will have to change.

In the package file you find a rest sensor. The example file is set to request three 'Timing Point Codes' (tpc) but you can also use more or just a single one. Home Assistant can't handle too much data in its attributes very well. The tpcs you can find in https://ovzoeker.nl/. A signle bustop usually has two tpcs, one for each side of the road.

```yaml
rest:
  - resource: "http://v0.ovapi.nl/tpc/30005031,30005032,30005065"

```

Create a feed sensor for each tpc. This is your *ovapi feed sensor*. It contains the raw feed data.

```yaml
    scan_interval: 60
    sensor:
      - name: "OVAPI Feed 30005031"
        unique_id: ovapi_feed_30005031
        json_attributes_path: $['30005031']
        json_attributes:
            - Stop
            - GeneralMessages
            - Passes
        value_template: >-
            {{ value_json['30005031'].Stop.TimingPointName }}

```

Next create a helper sernsor that will sort and filter the data. This helper sensor will use the `data.yaml` blueprint. This is where you decide what to display.
A helper can use multiple `source_sensor`s, for instance if you want to make a table for both sides of a busstop or multiple quays of a busstation.

The `lijnen` attribute is optional and can be fully ommitted. I you want to filter on a line number (string). Make it a list, also if it is a signle line you want to filter on! If you use filter, don't forget that nightbusses use a different line number.


```yaml
template:
  - use_blueprint:
      path: ovapi/data.yaml
      input:
        source_sensor: 
          - 'sensor.ovapi_feed_30005031'
        lijnen:
          - "14"
          - "4"
    name: "OVAPI helper sensor 30005031"
    unique_id: ovapi_data_30005031
```

Repeat this step for as many *helper sensors* you want. This way you can make sensors per stop, per line or even per bus station.

`source_sensor` should contain the entity_id of the OVAPI feed sensor above. If you changed the name and unique id of the feed sensor correctly in the first step, changing the tpcs should have been enough. If the sensor is not working, check the entity id of the rest sensor.

Also predefined with blueprint templates are the final information sensors. Here the name is less important and may be more descriptive. The Source sensor of the _vertrek sensor_  is the *OVAPI helper sensor*. Unless something strange happened the entity id of the data sensor is the slug of its name. If no data comes through, check that the entity id of the feed and helper sensors and make sure all sensors have a unique unique_id.^["The entity_id is not the same as the unique_id, they are unrelated."]

```yaml
  - use_blueprint:
      path: ovapi/vertrek.yaml
      input:
        source_sensor: 'sensor.ovapi_helper_sensor_30005031'
    name: "OVAPI vertrek Damrak"
    unique_id: ovapi_vertrek_30005031

```

You should now have three sensors per tpc. A feed, a helper and a vertrek (departure) sensor in [your helpers](https://my.home-assistant.io/redirect/helpers/).


### Cards

The `vertrek` sensor contains a formatted value showing the time to the next bus/ferry/metro/tram and attributes for the next three upcoming rides. Not that not all cards will allow you to display the attributes but with the sensor the possibilities are endless.

In the repo you find an example lovelace card of a [DRIS sign](https://www.spie-nl.com/oplossing/dynamische-reizigers-informatie-systemen). The yaml from this card you can copy-paste to a custom card. It will automatically fetch the background image from github but if you want to host the image locally you can upload the png image also via the UI and it will be locally stored.

To make adjustments easier the lovelace card in the repo makes extensive use of YAML-anchors. The anchors are converted to plain YAML and removed when you save the card with the Home Assistant code editor. After reopening the code editor you may remove the tags used for anchors from the top of you yaml file to enable the use of the UI editor again.

![Example DRIS departure board oneway and combined](doc/example_DRIS_oneway_and_combined.png)

Alternatively you could use a vertical-stack card with hidden entity names and just the attributes

![Example_attributes](doc/Example_attributes.png)

```yaml

type: vertical-stack
title: Vertrektijden Dam
cards:
  - type: tile
    entity: sensor.ovapi_vertrek_dam
    name: " "
    state_content:
      - lijn_1
      - bestemming_1
      - vertrektijd_1
    vertical: false
    features_position: bottom
  - type: tile
    entity: sensor.ovapi_vertrek_dam
    name: " "
    state_content:
      - lijn_2
      - bestemming_2
      - vertrektijd_2
    vertical: false
    features_position: bottom
  - type: tile
    entity: sensor.ovapi_vertrek_dam
    name: " "
    state_content:
      - lijn_3
      - bestemming_3
      - vertrektijd_3
    vertical: false
    features_position: bottom

```

Or simply use the values of one or more sensors in a entities card:

![Example_entities](doc/Example_entities.png)

Or even more simple, make a badge out of the `vertrek` sensor:

![Example_badges](doc/Example_badges.png)

```yaml

type: entity
show_name: false
show_state: true
show_icon: true
entity: sensor.ovapi_vertrek_dam

```
