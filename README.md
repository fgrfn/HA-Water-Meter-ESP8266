# ESP8266 HA-Water-Meter

Dieses Projekt zeigt, wie ein ESP8266-basierter Wasserzähler mithilfe von [ESPHome](https://esphome.io/) in [Home Assistant](https://www.home-assistant.io/) integriert wird.

## Grundlage: ESPHome

Der ESP8266 wird mit ESPHome geflasht und ist über WLAN direkt in Home Assistant integrierbar.

- **Flow Rate** (L/min)
- **Total Water Consumption** (m³, `total_increasing`, übersteht Neustarts und Stromausfälle)
- **Set Meter Reading** (Zählerstand ans physische Zählwerk angleichen)
- **Device Online** (online/offline via `binary_sensor`)
- Diagnose: Uptime, WiFi-Signal, Reset Reason, IP, SSID, ESPHome-Version, Restart-Button

👉 Die vollständige ESPHome-Konfiguration findest du in [`ha-water-meter.yaml`](ha-water-meter.yaml).

---

## Hardware

- **Board:** ESP8266 (NodeMCU v2, `board: nodemcuv2` – bei anderem Board anpassen)
- **Zähler:** „dummer" Wasserzähler mit Impulsausgang (z. B. Reed-Kontakt), **1 Puls = 1 Liter**
- **Verdrahtung:** Impulsausgang zwischen `GPIO12` und `GND`

Liefert dein Zähler einen anderen Impulswert (z. B. 1 Puls = 10 L), passe den
`multiply`-Filter am Total-Sensor an (`0.001` = 1 L/Puls → m³).
Falls kein externer Pull-up-Widerstand verbaut ist, aktiviere im YAML den
internen Pull-up (`mode: INPUT_PULLUP`, im YAML als Kommentar vorbereitet).

---

## Installation & Secrets

Alle Zugangsdaten liegen in einer `secrets.yaml` (nicht im Repo, siehe `.gitignore`):

1. [`secrets.yaml.example`](secrets.yaml.example) nach `secrets.yaml` kopieren
   (im ESPHome-Dashboard: die globale `secrets.yaml` um die Einträge ergänzen).
2. Werte eintragen. Einen API-Verschlüsselungskey erzeugst du mit:

   ```bash
   openssl rand -base64 32
   ```

> ⚠️ Ist das Gerät bereits in Home Assistant eingebunden, behalte den
> vorhandenen API-Key bei — sonst verliert HA die Verbindung zum Gerät.

---

## Entitäten & Entity-IDs

ESPHome stellt den Gerätenamen (`ha-water-meter`) den Entity-IDs voran. Die
Standard-IDs lauten damit z. B.:

| Entität | Entity-ID (Standard) |
|---|---|
| Durchfluss | `sensor.ha_water_meter_flow_flow_rate` |
| Gesamtverbrauch | `sensor.ha_water_meter_flow_total_water_consumption` |
| Zählerstand setzen | `number.ha_water_meter_flow_set_meter_reading` |
| Online-Status | `binary_sensor.ha_water_meter_flow_device_online` |
| Uptime | `sensor.ha_water_meter_flow_uptime` |
| Reset Reason | `text_sensor.ha_water_meter_flow_reset_reason` |

**Wichtig:** Wurden Entitäten in Home Assistant umbenannt (z. B. zu
`sensor.wasserverbrauch_total` / `sensor.wasserdurchfluss`), müssen die IDs in
allen folgenden Beispielen entsprechend ersetzt werden.

---

## Zählerstand: Persistenz & Abgleich

Der Gesamtverbrauch wird als Pulszähler in einem `global` mit
`restore_value` gespeichert und regelmäßig ins Flash geschrieben
(`restore_from_flash: true`, max. alle 5 Minuten gegen Flash-Verschleiß).
Damit übersteht der Zählerstand Neustarts **und** Stromausfälle — es können
höchstens die letzten ~5 Minuten Verbrauch verloren gehen.

Über die Entität **„Set Meter Reading"** (m³) lässt sich der Zählerstand
jederzeit ans physische Zählwerk angleichen — z. B. nach dem ersten Flashen
oder nach einem Zählertausch. Einfach den aktuellen Stand des Wasserzählers
eintragen, der Total-Sensor springt sofort auf diesen Wert.

---

## Home Assistant: Verfügbarkeit + Last Seen

### 1) Device Online/Offline

`binary_sensor.ha_water_meter_flow_device_online` wird direkt vom
ESPHome-`status` Binary Sensor geliefert.

### 2) Last Seen (optional)

Die Variante über `last_changed` vom Verbrauchssensor ist bei Nullverbrauch unzuverlässig.
Robuster ist ein trigger-basierter Last-Seen-Sensor, der auf Online-Status **oder** einen Heartbeat (z. B. Uptime) reagiert:

```yaml
template:
  - trigger:
      - platform: state
        entity_id:
          - binary_sensor.ha_water_meter_flow_device_online
          - sensor.ha_water_meter_flow_uptime
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
    source: sensor.ha_water_meter_flow_total_water_consumption
    cycle: daily
  util_water_usage_weekly:
    source: sensor.ha_water_meter_flow_total_water_consumption
    cycle: weekly
  util_water_usage_monthly:
    source: sensor.ha_water_meter_flow_total_water_consumption
    cycle: monthly
  util_water_usage_yearly:
    source: sensor.ha_water_meter_flow_total_water_consumption
    cycle: yearly
```

Dank `state_class: total_increasing` verkraften die Utility Meter auch einen
Reset des Quellsensors. Der Sensor eignet sich außerdem direkt für das
**Energie-Dashboard** (Abschnitt „Wasser").

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

Für Diagnose stehen zusätzliche Entitäten bereit:
- `sensor.ha_water_meter_flow_uptime`
- `text_sensor.ha_water_meter_flow_reset_reason`
- `button.ha_water_meter_flow_restart_device`

### 1) Flow-Wert kommt verzögert (z. B. erst nach 20–30s)
Das ist bei kleinen Durchflüssen normal, weil erst Pulse eintreffen müssen.

Bereits empfohlen im YAML:
- `timeout: 10s` → setzt den Flow schneller auf 0 bei ausbleibenden Pulsen
- `internal_filter: 50ms` → Entprellung gegen Störimpulse

**Hinweis:** Bei 1 Puls/Liter und `timeout: 10s` pendeln Durchflüsse unter
6 L/min zwischen Messwert und 0 — das ist prinzipbedingt. Wer kleine
Durchflüsse stabiler sehen will, erhöht das `timeout` (auf Kosten der
Reaktionszeit beim Zudrehen).

### 2) „Unerwartete Geräteklasse" in HA
Für Durchfluss muss der Sensor auf `device_class: volume_flow_rate` stehen (oder ohne Geräteklasse laufen).

### 3) Sprunghafte Werte
- Sensor mechanisch stabil montieren
- Kabellänge kurz halten / sauber schirmen
- Masseführung prüfen
- Falls nötig Entprellung (`internal_filter`) erhöhen

### 4) Verbrauch steigt nicht
- Pulseingang / Pin prüfen (`pulse_pin`)
- Pull-up/Pull-down am Sensor prüfen (ggf. `mode: INPUT_PULLUP` im YAML aktivieren)
- In ESPHome-Logs nach Pulseingang schauen

### 5) Zählerstand nach Neustart falsch
Der Zählerstand wird alle 5 Minuten ins Flash geschrieben — nach einem
Stromausfall können also die letzten Minuten Verbrauch fehlen. Über
**„Set Meter Reading"** lässt er sich jederzeit wieder ans physische
Zählwerk angleichen. `text_sensor.ha_water_meter_flow_reset_reason` zeigt,
warum das Gerät neu gestartet ist.

---

## CI & Releases

Bei jedem Push/PR, der `ha-water-meter.yaml` ändert, validiert eine GitHub
Action die Konfiguration mit `esphome config` (gegen Dummy-Secrets). Nach
einem Merge auf `main` wird automatisch ein Release mit angehängter YAML
erstellt.

---

## Hinweis
Die Langzeit-Auswertung gehört in HA (Recorder/Utility Meter), nicht auf den ESP. So bleibt der ESP schlank und stabil, während HA die Historie übernimmt.

## Lizenz

[MIT](LICENSE)
