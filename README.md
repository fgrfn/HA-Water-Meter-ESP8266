# ESP8266 HA-Water-Meter

Dieses Projekt zeigt, wie ein ESP8266-basierter Wasserzähler mithilfe von [ESPHome](https://esphome.io/) in [Home Assistant](https://www.home-assistant.io/) integriert wird.

## Grundlage: ESPHome

Der ESP8266 wird mit ESPHome geflasht und ist über WLAN direkt in Home Assistant integrierbar.

- **Flow Rate** (L/min)
- **Total Water Consumption** (m³, `total_increasing`)
- **Device Online** (online/offline via `binary_sensor`)

👉 Die vollständige ESPHome-Konfiguration findest du in [`ha-water-meter.yaml`](https://github.com/fgrfn/HA-Water-Meter-ESP8266/blob/main/ha-water-meter.yaml).

---

## Home Assistant: Verfügbarkeit + Last Seen

### 1) Device Online/Offline

Der Sensor `binary_sensor.flow_device_online` (Name je nach Friendly Name) wird direkt vom ESPHome-`status` Binary Sensor geliefert.

### 2) Last Seen (optional)

Die Variante über `last_changed` vom Verbrauchssensor ist bei Nullverbrauch unzuverlässig.
Robuster ist ein trigger-basierter Last-Seen-Sensor, der auf Online-Status **oder** einen Heartbeat (z. B. Uptime) reagiert:

```yaml
template:
  - trigger:
      - platform: state
        entity_id:
          - binary_sensor.flow_device_online
          - sensor.flow_uptime
    sensor:
      - name: "Water Meter Last Seen"
        unique_id: water_meter_last_seen
        device_class: timestamp
        state: "{{ now().isoformat() }}"
```

Damit aktualisiert sich „Last Seen" auch dann regelmäßig, wenn gerade kein Wasser fließt.

---

## Home Assistant: Langzeitverbrauch (Tag/Woche/Monat/Jahr)

```yaml
utility_meter:
  util_water_usage_daily:
    source: sensor.flow_total_water_consumption
    cycle: daily
  util_water_usage_weekly:
    source: sensor.flow_total_water_consumption
    cycle: weekly
  util_water_usage_monthly:
    source: sensor.flow_total_water_consumption
    cycle: monthly
  util_water_usage_yearly:
    source: sensor.flow_total_water_consumption
    cycle: yearly
```

---

## Kosten-Sensor (mit Wasserpreis)

### 1) Wasserpreis als Helper (`input_number`)

```yaml
input_number:
  water_price_per_m3:
    name: Wasserpreis pro m³
    min: 0
    max: 20
    step: 0.01
    unit_of_measurement: "€/m³"
    mode: box
```

### 2) Template-Sensoren für Kosten

```yaml
template:
  - sensor:
      - name: "Water Cost Today"
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {{ (states('sensor.util_water_usage_daily')|float(0) * states('input_number.water_price_per_m3')|float(0)) | round(2) }}

      - name: "Water Cost This Month"
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {{ (states('sensor.util_water_usage_monthly')|float(0) * states('input_number.water_price_per_m3')|float(0)) | round(2) }}

      - name: "Water Cost This Year"
        unit_of_measurement: "€"
        device_class: monetary
        state_class: total
        state: >
          {{ (states('sensor.util_water_usage_yearly')|float(0) * states('input_number.water_price_per_m3')|float(0)) | round(2) }}
```

---

## Troubleshooting (typische Fälle)

Für Diagnose sind in der ESPHome-Konfiguration zusätzliche Entitäten sinnvoll:
- `sensor.flow_uptime`
- `text_sensor.flow_reset_reason`
- `button.flow_restart_device`

### 1) Flow-Wert kommt verzögert (z. B. erst nach 20–30s)
Das ist bei kleinen Durchflüssen normal, weil erst Pulse eintreffen müssen.

Bereits empfohlen im YAML:
- `timeout: 10s` → setzt den Flow schneller auf 0 bei ausbleibenden Pulsen
- `internal_filter: 50ms` → Entprellung gegen Störimpulse

### 2) „Unerwartete Geräteklasse“ in HA
Für Durchfluss muss der Sensor auf `device_class: volume_flow_rate` stehen (oder ohne Geräteklasse laufen).

### 3) Sprunghafte Werte
- Sensor mechanisch stabil montieren
- Kabellänge kurz halten / sauber schirmen
- Masseführung prüfen
- Falls nötig Entprellung (`internal_filter`) erhöhen

### 4) Verbrauch steigt nicht
- Pulseingang / Pin prüfen (`pulse_pin`)
- Pullup/Pulldown am Sensor prüfen
- In ESPHome-Logs nach Pulseingang schauen

---

## Hinweis
Die Langzeit-Auswertung gehört in HA (Recorder/Utility Meter), nicht auf den ESP. So bleibt der ESP schlank und stabil, während HA die Historie übernimmt.
