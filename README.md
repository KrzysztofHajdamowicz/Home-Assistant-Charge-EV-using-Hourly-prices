# Home-Assistant: Charge your EV using Hourly prices
## (aka "Dynamic Tariffs")
More and more grid operators offer dynamic pricing. Use Home Assistant to charge your car using cheapest hours!
This Repository contains recipe to show how I do it.


## Pstryk

In case you are a client of Pstryk.pl as your electricity seller, you can use their Home Assistant integration:  
https://github.com/Bankilo/pstryk-home-assistant  
Their pricing sensor factors-in all factors and provide true cost of electricity you will pay.  
It's also already compatible with "EV Smart Charging" integration and "cheapest-energy-hours" jinja template.

If you are not their client, next two steps are for you:

### Get dynamic pricing info in Poland
In Poland, single point of truth in terms of electricity price for dynamic tariffs is TGE (Towarowa Giełda Energii).  
Price table is published every day as "Rynek Dnia Następnego" (day-ahead market): https://tge.pl/energia-elektryczna-rdn  
Unfortunately, as of 15th of December 2024, no electricity seller offers an API to provide pricing table.  

Happily, @PiotrMachowski created Home Assistant Custom Integration that downloads those data in format compatible with other tools in this area:  
https://github.com/PiotrMachowski/Home-Assistant-custom-components-TGE  
Thank You so much, @PiotrMachowski, your contribution is invaluable!

### Extend pricing by distribution price
Next, when You buy electricity from your seller, you need to pay your distributor to deliver those electrons to your utility meter.  
In Poland, when you switch to dynamic tariffs, you are moved into G12W distribution tariff (Peak and Off-Peak during the day, Off-peak weekends)
Consult your utility bill for prices of distribution.  

To factor-in price of distribution to your price calculation, use "Price Modifier template" available in TGE custom integration.  
I don't know how, but @PiotrMachowski created such improvement in TGE Integration, Thank You again!
Use this template as a starting point, also check other discussions in original repo to find template suitable for your needs.  
In case something is missing, you can write your own template using Jinja2 templating engine, same as Home Assistant `Template` entities.  
https://github.com/PiotrMachowski/Home-Assistant-custom-components-TGE/discussions/8

## Control entities using pricing data
That's the fundamental of this post.  
So far, I have found two (Powerful!) integrations that help in controlling entities when prices are low or high.  
1. First is "EV Smart Charging" by Jonas B Karlsson: https://github.com/jonasbkarlsson/ev_smart_charging/  
It's quite easy to configure and you can use it to charge your car in cheapest timeperiod between _NOW_ and _Departure Time_

2. Second is "Cheapest Energy Hours" written by TheFes  
https://github.com/TheFes/cheapest-energy-hours/
Incredibly powerful tool, capable of finding cheapest and more expensive hours, in various size blocks, in continous and non-continous form.
Read documentation of this Jinja2 template macro. It's not *super* hard, but be prepared and job will be easier.

## Visualize

![ApexCharts card showing graph of electricity price](energy_prices.png?raw=true "Title")

I recommend "ApexCharts Card" for this job.  
https://github.com/RomRider/apexcharts-card

Example configuration do show electricity prices with most expensive and cheapest, consecutive and non-consecutive blocks looks like this:
```YAML
type: custom:apexcharts-card
experimental:
  color_threshold: true
graph_span: 2day
update_interval: 30s
show:
  loading: true
now:
  show: true
  label: Now
  color: gray
span:
  offset: +0day
  start: day
yaxis:
  - id: primary
    decimals: 1
    min: 0
  - id: periods
    min: 0
    max: 1
    decimals: 0
    show: false
header:
  show: true
  title: Electricity Price - Today and tomorrow
  show_states: true
  colorize_states: true
series:
  - entity: sensor.tge_fixing_1_rate
    type: line
    curve: stepline
    yaxis_id: primary
    name: Now
    color_threshold:
      - value: 0
        color: green
      - value: 75
        color: orange
      - value: 150
        color: red
    stroke_width: 3
    show:
      extremas: true
      in_header: before_now
      legend_value: false
      header_color_threshold: true
    data_generator: |
      return entity.attributes.prices.map(d => {
      return [new Date(d.time).getTime(), d.price];
      });
  - entity: binary_sensor.4_cheapest_consecutive_hours
    type: area
    curve: stepline
    name: 4 cheapest consecutive
    yaxis_id: periods
    stroke_width: 0
    opacity: 0.3
    color: "#03a9f4"
    show:
      legend_value: false
      in_header: false
      datalabels: false
      extremas: false
    data_generator: |
      let debugData = entity.attributes.debug;
      let dataPoints = [];

      try {
        // Parse the JSON string into an object/array
        let parsedData = JSON.parse(debugData);

        // Check if parsedData is an array and process it accordingly
        if (Array.isArray(parsedData)) {
          parsedData.forEach(item => {
            if (item.start && item.end) {
              let startTime = Date.parse(item.start);
              let endTime = Date.parse(item.end);
              dataPoints.push([startTime, 1]);
              dataPoints.push([endTime, 0]);
            }
          });
        } else if (parsedData && parsedData.start && parsedData.end) {
          // Handle the case where parsedData is a single object
          let startTime = Date.parse(parsedData.start);
          let endTime = Date.parse(parsedData.end);
          dataPoints.push([startTime, 1]);
          dataPoints.push([endTime, 0]);
        } else {
          console.error('Parsed debugData structure is not handled:', parsedData);
        }
      } catch (e) {
        console.error('Failed to parse debugData as JSON:', e);
      }
      return dataPoints;
  - entity: binary_sensor.4_cheapest_non_consecutive_hours
    type: area
    curve: stepline
    name: 4 cheapest non-consecutive
    yaxis_id: periods
    stroke_width: 0
    opacity: 0.3
    color: lightblue
    show:
      legend_value: false
      in_header: false
      datalabels: false
      extremas: false
    data_generator: |
      let debugData = entity.attributes.debug;
      let dataPoints = [];

      try {
        // Parse the JSON string into an object/array
        let parsedData = JSON.parse(debugData);

        // Check if parsedData is an array and process it accordingly
        if (Array.isArray(parsedData)) {
          parsedData.forEach(item => {
            if (item.start && item.end) {
              let startTime = Date.parse(item.start);
              let endTime = Date.parse(item.end);
              dataPoints.push([startTime, 1]);
              dataPoints.push([endTime, 0]);
            }
          });
        } else if (parsedData && parsedData.start && parsedData.end) {
          // Handle the case where parsedData is a single object
          let startTime = Date.parse(parsedData.start);
          let endTime = Date.parse(parsedData.end);
          dataPoints.push([startTime, 1]);
          dataPoints.push([endTime, 0]);
        } else {
          console.error('Parsed debugData structure is not handled:', parsedData);
        }
      } catch (e) {
        console.error('Failed to parse debugData as JSON:', e);
      }

      return dataPoints;
  - entity: binary_sensor.4_most_expensive_consecutive_hours
    type: area
    curve: stepline
    name: 4 most expensive consecutive
    yaxis_id: periods
    stroke_width: 0
    opacity: 0.3
    color: red
    show:
      legend_value: false
      in_header: false
      datalabels: false
      extremas: false
    data_generator: |
      let debugData = entity.attributes.debug;
      let dataPoints = [];

      try {
        // Parse the JSON string into an object/array
        let parsedData = JSON.parse(debugData);

        // Check if parsedData is an array and process it accordingly
        if (Array.isArray(parsedData)) {
          parsedData.forEach(item => {
            if (item.start && item.end) {
              let startTime = Date.parse(item.start);
              let endTime = Date.parse(item.end);
              dataPoints.push([startTime, 1]);
              dataPoints.push([endTime, 0]);
            }
          });
        } else if (parsedData && parsedData.start && parsedData.end) {
          // Handle the case where parsedData is a single object
          let startTime = Date.parse(parsedData.start);
          let endTime = Date.parse(parsedData.end);
          dataPoints.push([startTime, 1]);
          dataPoints.push([endTime, 0]);
        } else {
          console.error('Parsed debugData structure is not handled:', parsedData);
        }
      } catch (e) {
        console.error('Failed to parse debugData as JSON:', e);
      }

      return dataPoints;
  - entity: binary_sensor.4_most_expensive_non_consecutive_hours
    type: area
    curve: stepline
    name: 4 most expensive non-consecutive
    yaxis_id: periods
    stroke_width: 0
    opacity: 0.3
    color: purple
    show:
      legend_value: false
      in_header: false
      datalabels: false
      extremas: false
    data_generator: |
      let debugData = entity.attributes.debug;
      let dataPoints = [];

      try {
        // Parse the JSON string into an object/array
        let parsedData = JSON.parse(debugData);

        // Check if parsedData is an array and process it accordingly
        if (Array.isArray(parsedData)) {
          parsedData.forEach(item => {
            if (item.start && item.end) {
              let startTime = Date.parse(item.start);
              let endTime = Date.parse(item.end);
              dataPoints.push([startTime, 1]);
              dataPoints.push([endTime, 0]);
            }
          });
        } else if (parsedData && parsedData.start && parsedData.end) {
          // Handle the case where parsedData is a single object
          let startTime = Date.parse(parsedData.start);
          let endTime = Date.parse(parsedData.end);
          dataPoints.push([startTime, 1]);
          dataPoints.push([endTime, 0]);
        } else {
          console.error('Parsed debugData structure is not handled:', parsedData);
        }
      } catch (e) {
        console.error('Failed to parse debugData as JSON:', e);
      }

      return dataPoints;
```
Binary sensors to calculate time ranges are:
(I used "Cheapest Energy Hours" Jinja2 macro)
```YAML
template:
  - binary_sensor:
    - unique_id: phahw3ra8aiShohphaibiegee9ahf6Cu
      name: 4 cheapest consecutive hours
      device_class: battery_charging
      state: >
        {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
        {% set sensor = 'sensor.tge_fixing_1_rate' -%}
        {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
        {% set end = states('input_datetime.find_cheapest_hours_until') -%}
        {% set hours = 4 -%}
        {{
          cheapest_energy_hours(
            sensor=sensor,
            attr_today='prices_today',
            attr_tomorrow='prices_tomorrow',
            time_key='time',
            value_key='price',
            start=start,
            end=end,
            include_tomorrow=true,
            hours=4,
            split=false,
            mode='is_now'
          )
        }}
      availability: >
        {{
          (states('sensor.tge_fixing_1_rate') | is_number)
        }}
      attributes:
        start: "{{ states('input_datetime.find_cheapest_most_expensive_hours_since') }}"
        end: "{{ states('input_datetime.find_cheapest_hours_until') }}"
        hours: "4"
        debug: >
          {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
          {% set sensor = 'sensor.tge_fixing_1_rate' -%}
          {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
          {% set end = states('input_datetime.find_cheapest_hours_until') -%}
          {% set hours = 4 -%}
          {{
            cheapest_energy_hours(
              sensor=sensor,
              attr_today='prices_today',
              attr_tomorrow='prices_tomorrow',
              time_key='time',
              value_key='price',
              start=start,
              end=end,
              include_tomorrow=true,
              hours=hours,
              split=false,
              mode='all'
            )
          }}
    - unique_id: Aesouthaico3ic7oowa4agahiJeeghoo
      name: 4 cheapest non-consecutive hours
      device_class: battery_charging
      state: >
        {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
        {% set sensor = 'sensor.tge_fixing_1_rate' -%}
        {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
        {% set end = states('input_datetime.find_cheapest_hours_until') -%}
        {% set hours = 4 -%}
        {{
          cheapest_energy_hours(
            sensor=sensor,
            attr_today='prices_today',
            attr_tomorrow='prices_tomorrow',
            time_key='time',
            value_key='price',
            start=start,
            end=end,
            include_tomorrow=true,
            hours=4,
            split=true,
            mode='is_now'
          )
        }}
      availability: >
        {{
          (states('sensor.tge_fixing_1_rate') | is_number)
        }}
      attributes:
        start: "{{ states('input_datetime.find_cheapest_most_expensive_hours_since') }}"
        end: "{{ states('input_datetime.find_cheapest_hours_until') }}"
        hours: "4"
        debug: >
          {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
          {% set sensor = 'sensor.tge_fixing_1_rate' -%}
          {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
          {% set end = states('input_datetime.find_cheapest_hours_until') -%}
          {% set hours = 4 -%}
          {{
            cheapest_energy_hours(
              sensor=sensor,
              attr_today='prices_today',
              attr_tomorrow='prices_tomorrow',
              time_key='time',
              value_key='price',
              start=start,
              end=end,
              include_tomorrow=true,
              hours=hours,
              split=true,
              mode='all'
            )
          }}
    - unique_id: yaesat9Phai3wu4eishuThofietoo8Sh
      name: 4 most expensive consecutive hours
      device_class: battery_charging
      state: >
        {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
        {% set sensor = 'sensor.tge_fixing_1_rate' -%}
        {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
        {% set end = states('input_datetime.find_cheapest_hours_until') -%}
        {% set hours = 4 -%}
        {{
          cheapest_energy_hours(
            sensor=sensor,
            attr_today='prices_today',
            attr_tomorrow='prices_tomorrow',
            time_key='time',
            value_key='price',
            start=start,
            end=end,
            include_tomorrow=true,
            hours=4,
            split=false,
            lowest=false,
            mode='is_now'
          )
        }}
      availability: >
        {{
          (states('sensor.tge_fixing_1_rate') | is_number)
        }}
      attributes:
        start: "{{ states('input_datetime.find_cheapest_most_expensive_hours_since') }}"
        end: "{{ states('input_datetime.find_cheapest_hours_until') }}"
        hours: "4"
        debug: >
          {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
          {% set sensor = 'sensor.tge_fixing_1_rate' -%}
          {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
          {% set end = states('input_datetime.find_cheapest_hours_until') -%}
          {% set hours = 4 -%}
          {{
            cheapest_energy_hours(
              sensor=sensor,
              attr_today='prices_today',
              attr_tomorrow='prices_tomorrow',
              time_key='time',
              value_key='price',
              start=start,
              end=end,
              include_tomorrow=true,
              hours=hours,
              split=false,
              lowest=false,
              mode='all'
            )
          }}
    - unique_id: fuecooPhaizavoh0ShooSeequu8echee
      name: 4 most expensive non-consecutive hours
      device_class: battery_charging
      state: >
        {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
        {% set sensor = 'sensor.tge_fixing_1_rate' -%}
        {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
        {% set end = states('input_datetime.find_cheapest_hours_until') -%}
        {% set hours = 4 -%}
        {{
          cheapest_energy_hours(
            sensor=sensor,
            attr_today='prices_today',
            attr_tomorrow='prices_tomorrow',
            time_key='time',
            value_key='price',
            start=start,
            end=end,
            include_tomorrow=true,
            hours=4,
            split=true,
            lowest=false,
            mode='is_now'
          )
        }}
      availability: >
        {{
          (states('sensor.tge_fixing_1_rate') | is_number)
        }}
      attributes:
        start: "{{ states('input_datetime.find_cheapest_most_expensive_hours_since') }}"
        end: "{{ states('input_datetime.find_cheapest_hours_until') }}"
        hours: "4"
        debug: >
          {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
          {% set sensor = 'sensor.tge_fixing_1_rate' -%}
          {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
          {% set end = states('input_datetime.find_cheapest_hours_until') -%}
          {% set hours = 4 -%}
          {{
            cheapest_energy_hours(
              sensor=sensor,
              attr_today='prices_today',
              attr_tomorrow='prices_tomorrow',
              time_key='time',
              value_key='price',
              start=start,
              end=end,
              include_tomorrow=true,
              hours=hours,
              split=true,
              lowest=false,
              mode='all'
            )
          }}
    - unique_id: pa1ueCh3neiTahkad0Ohch8eipet4sew
      name: 2 most expensive consecutive hours
      device_class: battery_charging
      state: >
        {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
        {% set sensor = 'sensor.tge_fixing_1_rate' -%}
        {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
        {% set end = states('input_datetime.find_cheapest_hours_until') -%}
        {% set hours = 2 -%}
        {{
          cheapest_energy_hours(
            sensor=sensor,
            attr_today='prices_today',
            attr_tomorrow='prices_tomorrow',
            time_key='time',
            value_key='price',
            start=start,
            end=end,
            include_tomorrow=true,
            hours=hours,
            split=false,
            lowest=false,
            mode='is_now'
          )
        }}
      availability: >
        {{
          (states('sensor.tge_fixing_1_rate') | is_number)
        }}
      attributes:
        start: "{{ states('input_datetime.find_cheapest_most_expensive_hours_since') }}"
        end: "{{ states('input_datetime.find_cheapest_hours_until') }}"
        hours: "2"
        debug: >
          {% from 'cheapest_energy_hours.jinja' import cheapest_energy_hours -%}
          {% set sensor = 'sensor.tge_fixing_1_rate' -%}
          {% set start = states('input_datetime.find_cheapest_most_expensive_hours_since') -%}
          {% set end = states('input_datetime.find_cheapest_hours_until') -%}
          {% set hours = 2 -%}
          {{
            cheapest_energy_hours(
              sensor=sensor,
              attr_today='prices_today',
              attr_tomorrow='prices_tomorrow',
              time_key='time',
              value_key='price',
              start=start,
              end=end,
              include_tomorrow=true,
              hours=hours,
              split=false,
              lowest=false,
              mode='all'
            )
          }}

```
