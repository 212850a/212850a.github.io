---
layout: post
title: "Energy statistics in Home Assistant"
date: 2022-12-23 09:00:00 +0200
tags: homeassistant shelly solar huawei energy
published: true
---
## Preface
With a help of [Shelly 3EM](https://shelly.cloud/products/shelly-3em-smart-home-automation-energy-meter/) devices it’s possible to gather energy statistics from electrical phases you have.
You have historical metrics like current (A), power (kW), voltage (V), energy (kWh), energy returned to grid (kWh).
If you know what phase is used for some specific device you can analyse how much energy that device is used in various scenarios - it can be very interesting. For instant if you knew how much energy refrigerator used in the past and now it’s not the case - may be it’s time to clean it ;)

## Using tariffs for day & night energy cost
After you have base metrics you introduce/create additional ones - for example to calculate how much energy you’ve used in day/month and how much it will cost. It’s also possible to calculate cost if you have different energy tariffs for days and nights.
Here is example how to add additional daily & monthly metrics for each phase and count them based on day & night tariffs:
```
utility_meter:
  daily_total_energy_a:
    source: sensor.shellyem3_channel_a_energy
    name: Daily Energy
    cycle: daily
    tariffs:
      - day
      - night
  monthly_total_energy_a:
    source: sensor.shellyem3_channel_a_energy
    name: Monthly Energy
    cycle: monthly
    tariffs:
      - day
      - night
  daily_total_energy_b:
    source: sensor.shellyem3_channel_b_energy
    name: Daily Energy
    cycle: daily
    tariffs:
      - day
      - night
  monthly_total_energy_b:
    source: sensor.shellyem3_channel_b_energy
    name: Monthly Energy
    cycle: monthly
    tariffs:
      - day
      - night
  daily_total_energy_c:
    source: sensor.shellyem3_channel_c_energy
    name: Daily Energy
    cycle: daily
    tariffs:
      - day
      - night
  monthly_total_energy_c:
    source: sensor.shellyem3_channel_c_energy
    name: Monthly Energy
    cycle: monthly
    tariffs:
      - day
      - night

automation energy:
  - alias: "Change energy tariff day"
    trigger:
      - platform: time
        at: "07:00:00"
    condition:
      - condition: time
        weekday:
          - mon
          - tue
          - wed
          - thu
          - fri
    action:
      - service: utility_meter.select_tariff
        target:
          entity_id:
            - utility_meter.daily_total_energy_a
            - utility_meter.monthly_total_energy_a
            - utility_meter.daily_total_energy_b
            - utility_meter.monthly_total_energy_b
            - utility_meter.daily_total_energy_c
            - utility_meter.monthly_total_energy_c
        data:
          tariff: day

  - alias: "Change energy tariff night"
    trigger:
      - platform: time
        at: "23:00:00"
    condition:
      - condition: time
        weekday:
          - mon
          - tue
          - wed
          - thu
          - fri
    action:
      - service: utility_meter.select_tariff
        target:
          entity_id:
            - utility_meter.daily_total_energy_a
            - utility_meter.monthly_total_energy_a
            - utility_meter.daily_total_energy_b
            - utility_meter.monthly_total_energy_b
            - utility_meter.daily_total_energy_c
            - utility_meter.monthly_total_energy_c
        data:
          tariff: night

sensor:
  - platform: template
    sensors:
      total_energy_daily_day:
        friendly_name: 'Total Daily Energy (day tariff)'
        value_template: "{{ states('sensor.daily_total_energy_a_day')|float + states('sensor.daily_total_energy_b_day')|float + states('sensor.daily_total_energy_c_day')|float }}"
        unit_of_measurement: "kWh"
      total_energy_daily_night:
        friendly_name: 'Total Daily Energy (night tariff)'
        value_template: "{{ states('sensor.daily_total_energy_a_night')|float + states('sensor.daily_total_energy_b_night')|float + states('sensor.daily_total_energy_c_night')|float}}"
        unit_of_measurement: "kWh"
      total_energy_month_day:
        friendly_name: 'Total Month Energy (day tariff)'
        value_template: "{{ states('sensor.monthly_total_energy_a_day')|float + states('sensor.monthly_total_energy_b_day')|float + states('sensor.monthly_total_energy_c_day')|float }}"
        unit_of_measurement: "kWh"
      total_energy_month_night:
        friendly_name: 'Total Month Energy (night tariff)'
        value_template: "{{ states('sensor.monthly_total_energy_a_night')|float + states('sensor.monthly_total_energy_b_night')|float + states('sensor.monthly_total_energy_c_night')|float}}"
        unit_of_measurement: "kWh"
```

## Solar production statistics
Additional energy statistics comes if you have solar production which you have access to.
For instance with a help of [huawei_solar](https://github.com/wlcrs/huawei_solar) it’s possible to connect integration to Huawei SUN2000 inverter and get statistics about how much energy is generated on each phase, what is power and current. 

Before connecting Home Assistant to Huawei inverter ensure it's connected to your network and ModBus-TCP setting is set to Connection: Enable (unrestricted) - it would allow to connect to inverter for devices of your network.
![modbus-tcp](/assets/modbus-tcp.png)

By default Home Asistant doesn't have huawer_solar intergation so it has to be intalled manually:
* Download [huawei_solar](https://github.com/wlcrs/huawei_solar) to custom_components/huawei_solar folder
* Restart HA
* Go to Configuration -> Integrations and click the + Add Integration. Select Huawei Solar from the list
* Specify details for your Huawei inverter:
```
connection type: Network
address: INVERTER_IP
port: 502
slave id: 1
```

It is worth to mention that ModBus connectivity has limitation - only one client can be connected to inverter, so it won't be possible to use [huawei_solar](https://github.com/wlcrs/huawei_solar) integration if you need multiple connections to inverter from different devices. Alternative is [fusion_solar](https://github.com/tijsverkoyen/HomeAssistant-FusionSolar) integration which connects to [Huawei FusionSolar](https://eu5.fusionsolar.huawei.com) servers instead. Before to start using this integrartion [Northbound API Account](https://forum.huawei.com/enterprise/en/smart-pv-encyclopedia-how-to-create-a-northbound-api-account-through-the-fusionsolar/thread/1025182-100027) has to be created via FusionSolar portal:

System -> Company Management -> Northbound Management -> Add.

After you have energy usage and solar generation stats you can populate Energy Dashboard which is great in Home Assistant. 

All you need to do is just to specify what metrics are used for energy usage, energy returned, solar production and you have beautiful dashboard where you can see run report for energy usage & generation for day/week/month/year.

![ha-energy-dashboard-settings](/assets/ha-energy-dashboard-settings.png)
*H_Electricity channel A/B/C Energy & Energy Returned are metrics from Shelly 3EM, Total Yield - from Huawei invertor.*

![ha-energy-dashboard](/assets/ha-energy-dashboard.png)

If you have battery system - it can be added there either.
