# Flex Table

[![Version](https://img.shields.io/badge/version-0.4.0-green.svg?style=plastic)](#)
[![stability-stable-1.0-release-pending](https://img.shields.io/badge/stability-stable_1.0_release_incoming-green.svg?style=plastic)](#)
[![maintained](https://img.shields.io/maintenance/yes/2019.svg?style=plastic)](#)


## Installation (quick & "dirty")

* Find your homeassistent directory containing your configuration (let's say `~/.homeassistant/`)
* Change into `~/.homeassistant/www` (create the `www` directory, if it is not existing, you then might have to restart HA)
* `$ wget https://raw.githubusercontent.com/custom-cards/flex-table-card/master/flex-table-card.js` downloads the `.js` file directly where it should reside
* Finally, add the following on top of your UI Lovelace configuration (means either via Config UI or .yaml)
``` yaml
resources:
  - type: js
    url: /local/flex-table-card.js
```
* Verify that it works with one of the examples below

## Configuration
The `flex-table-card` aims for more flexibility for tabular-ish visuallization needs, by realizing:

- unlimited columns / rows 
- various different data-sources may be used in a single table
- lots of possibilities for configuration: entity selection (include, exclude), (hidden-)column-sorting, js-based content manipulation, row limiting and more...

Flex Table gives you the possibility to visualize any tabular data within Lovelace. Especially overviews with high data densities can be easily realized. Some screenshots:

<img src="https://github.com/daringer/image_dump/raw/master/tbl1.png" width=20% /><img src="https://github.com/daringer/image_dump/raw/master/tbl2.png" width=20% /><img src="https://github.com/daringer/image_dump/raw/master/tbl3.png" width=20% /><span><img src="https://github.com/daringer/image_dump/raw/master/trash_tbl.png" width=20% /><img src="https://github.com/daringer/image_dump/raw/master/tbl4.png" width=20% /></span>


**Configuration Options**


***Top-level options***

| Name            | Type     | Required?     | Description
| ----            | ----     | ------------- | -----------
| type            | string   | **required**  | `custom:flex-table-card`
| title           | string   |   optional    | A title for the card
| strict          | bool     |   optional    | If `true`, each cell must have a match, or row will be hidden
| sort_by         | col-id   |   optional    | Sort by column (see &lt;content&gt;), append '+' (ascending) or '-' (descending)
| max_rows        | int      |   optional    | Restrict the number of (shown) rows to this maximum number
| clickable       | bool     |   optional    | Activates the entities' on-click popup dialog
| entities        | section  | **required**  | Section defining the entity *data sources* (see below)
| columns         | section  | **required**  | Section defining the column(s) (see below)


***2nd-level options: entity selection / querying / filtering***

| `entities`    | Type     | Required?     | Description
| ----          | ----     | ------------- | -----------
| include       | regexp   | **required**  | Defines the initial entity data source(s)
| exclude       | regexp   |   optional    | Reduces the *included* data sources(s) 


***2nd-level options: columns definition, each list-item defines a column***

| `columns`     | Type     | Required?     | Description
| ----          | ----     | ------------- | -----------
| name          | string   |   optional    | Column header (if not set, &lt;content&gt; is used)
| hidden        | bool     |   optional    | `true` to avoid showing the column (e.g., for sorting)
| modify        | string   |   optional    | apply java-script code, `x` is data, i.e., `(x) => eval(<modfiy>)`
|&nbsp;&lt;content&gt; |        | **required**  | see in `column contents` below, one of those must exist!


***3rd-level options: column (cell) content definition, one required and mutually exclusive***

| `column contents` | Type     | Description
| ---------------   | ----     | -----------
| attr              | regexp   | matches to the first attribute matching this regexp
| prop              | string   | matches the entity's state members, e.g. **state** (any from [here](https://www.home-assistant.io/docs/configuration/state_object/) )
| attr_as_list      | string   | matched attribute is expected to contain a list to be expanded down the table (see table 1, 2 and 3)
 

**Examples**

- configuration of the contents is done by a two step approach
 
  - first the candidate **rows** have to be *queried* using 
    wildcarding/regular expressions, leading to a set of 
    entities (candidates)
  - eventually for each column a rule has to be given how 
    the matching should happen, matched e.g. attributes will 
    then be exposed as the contents of the row (cells)

``` yaml
type: custom:flex-table-card 
title: may be omitted, to be hidden

# 1st the **canidate** entities will be selected
entities:
  include: zwave.*
  exclude: zwave.unknown_node*

# 2nd, the *column contents* are defined, there are
# different ways to match contents:
columns:
  # example: match entity's attribute(s)
  - name: Column Header
    attr: receivedTS
    # extract only date from string
    modify: x.split("T")[0]
  - name: More Header
    attr: sendTS
  # example: match and show the given entity-property 
  #          e.g., the state (incl. any non-attr members)
  - name: Next Head
    prop: state
```

An extremly useful `flex-table-card` config: **List and sort all battery powered devices** and list them ascending by their battery level, so the next batteries to be replaced are always visible within the first row(s):

``` yaml
type: 'custom:flex-table-card'
sort_by: battery_level+
strict: true
title: Battery Levels
entities:
  exclude:
    - unknown_device
  include: zwave.*
columns:
  - attr: node_id
    name: NodeID
  - name: Name
    prop: name
  - attr: battery_level
    name: Reported Battery Level (%)
```

**Monitoring and identifying nodes, which are not communicating anymore** with the Z-Wave controller, can be extracted and also sorted according to their last sent/received message with the following config:

``` yaml
type: 'custom:flex-table-card'
max_rows: 25
#sort_by: sentTS- # <--- for last sent msg-based sorting
sort_by: receivedTS-
title: Durations Since Last Message (recv. & sent by node)
clickable: true   # <--- allows to click on row to show entity-popup for more information
columns:
  - attr: node_id
    name: NodeID
  - name: Name
    prop: name
  - attr: receivedTS
    modify: Math.round((Date.now() - Date.parse(x)) / 36000.) / 100.
    name: Recv. Age (h)
  - attr: sentTS
    modify: Math.round((Date.now() - Date.parse(x)) / 36000.) / 100.
    name: Sent Age (h)
entities:
  exclude:
    - zwave.unknown_device_*
    - zstick_gen5
  include: zwave.*
```

Further I also regulary have the need to 
transport, store and visualize non-trivial data
within the frontend, provided by appdaemon.

I use the **variable component**, which does 
simply nothing, despite providing an entity to be written
into from *appdaemon's* side via the set_variable() service.

To keep data ordered, passing a full (py-)list to the
service, leads to proper jsonification. But now we've done 
a reshape for the input data but luckily no problem. Nothing
more to adjust than to add a single keyword, with an obvious
naming:
  
``` yaml
type: custom:flex-table-card 
title: Fancy tabular data
# this matches then just the *variable component* entitiy
entities:
  include: variable.muell_tracker

# the columns are now similar, just with one match leading
# to severall lines being filled, check the screenshots:
columns:
  - name: Date
    attr_as_list: due_dates
  - name: Description
    attr_as_list: descriptions
```

**Current Issues / Drawbacks / Plans**

* cell alignment to be configured for each column
* allow "prefix" and "suffix" for each column to add units or similar stuff,
  means a simplifyed version of "modify"
* additional colunm *selector* for a service call maybe
* history / recorder access realization to match for historical data ...
* (click)-able sorting of columns   
* generally 'functions' might be a thing, a sum/avg/min/max ? but is the frontend the right spot for a micro-excel?
