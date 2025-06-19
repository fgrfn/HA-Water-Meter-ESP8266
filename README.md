# ESP8266 HA-Water-Meter

Dieses Projekt zeigt, wie ein ESP8266-basierter Wasserzähler mithilfe von [ESPHome](https://esphome.io/) in [Home Assistant](https://www.home-assistant.io/) integriert werden kann. Damit lässt sich der Wasserverbrauch zuverlässig erfassen und in stündlichen, täglichen, monatlichen sowie jährlichen Intervallen auswerten.

## Grundlage: ESPHome

Der ESP8266 wird mit ESPHome geflasht und ist über WLAN direkt in Home Assistant integrierbar. Über einen geeigneten Sensor (z. B. Impulsgeber am Wasserzähler) misst das Gerät den Wasserverbrauch und stellt diesen als Sensor `sensor.flow_total_water_consumption` zur Verfügung.

👉 Die vollständige ESPHome-Konfiguration findest du in der Datei [`ha-water-meter.yaml`](https://github.com/fgrfn/HA-Water-Meter-ESP8266/blob/main/ha-water-meter.yaml) in diesem Repository.

## Home Assistant Konfiguration

Zur Auswertung der Verbrauchsdaten nutzt dieses Projekt die [`utility_meter`](https://www.home-assistant.io/integrations/utility_meter/) Integration von Home Assistant. Füge folgenden YAML-Block in deine `configuration.yaml` ein:

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
