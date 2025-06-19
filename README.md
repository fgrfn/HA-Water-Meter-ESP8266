# HA-Water-Meter-ESP8266


configuration.yaml in HomeAssistant

# ESP8266 HA-Water-Meter
utility_meter:
  util_water_usage_hourly:
    source: sensor.flow_total_water_consumption
    cycle: hourly
  util_water_usage_daily:
    source: sensor.flow_total_water_consumption
    cycle: daily
  util_water_usage_monthly:
    source: sensor.flow_total_water_consumption
    cycle: monthly
  util_water_usage_yearly:
    source: sensor.flow_total_water_consumption
    cycle: yearly
