# ESP8266 HA-Water-Meter

Dieses Projekt dokumentiert die Integration eines ESP8266-basierten Wasserzählers in [Home Assistant](https://www.home-assistant.io/). Es ermöglicht die Erfassung und Auswertung des Wasserverbrauchs auf stündlicher, täglicher, monatlicher und jährlicher Basis.

## Home Assistant Konfiguration

Zur Nutzung der Verbrauchswerte wird die [`utility_meter`](https://www.home-assistant.io/integrations/utility_meter/) Integration verwendet. Füge den folgenden Konfigurationsblock zu deiner `configuration.yaml` hinzu:

```yaml
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
