# Entities und Helfer

Die Blueprints erzeugen **keine** Helfer. Sie erwarten, dass alle benötigten
`input_boolean`, `input_text`, `input_number`, Utility Meter, Template-Sensoren
und Flush-Scripts bereits existieren, und bekommen sie über die Blueprint-Inputs
zugewiesen.

Dieses Dokument ist die Referenz dafür: welche Helfer es pro Karte und pro Station
braucht, wie sie heißen und wie die nicht-trivialen aussehen.

## Namenskonventionen

Die Blueprints erzwingen **keine** Namen — jede Entity wird per Picker zugewiesen.
Das folgende Schema ist trotzdem verbindlich zu empfehlen, weil die Fehlersuche
ohne erkennbares Muster bei 10 Karten × N Stationen sehr schnell unangenehm wird.

| Entity | Bedeutung |
|---|---|
| `input_boolean.mc_k_01_switch_tagXXX` | MC-seitiger Status „Karte XXX ist gerade eingecheckt". Eine Quelle der Wahrheit für die ganze Session. |
| `input_boolean.tr_k_0N_switch_tagXXX` | Reader-seitiger Status pro Station (N = 01, 02, …). |
| `input_text.experiment_tagXXX` | Aktueller, zufällig generierter Experimentname. Wird als MQTT-Geräte-Identifier verwendet. |
| `input_text.input_experiment_tagXXX` | Basis-Name, vom Dashboard befüllt. Ausgangspunkt der Zufallsgenerierung. |
| `input_number.tr_k_0N_kwh_um_added_tagXXX` | Aufaddierter kWh-Wert pro Station+Karte seit dem letzten MC-Reset. |
| `sensor.tr_k_0N_kwh_um_tagXXX` | Utility Meter, Delta seit letzter Kalibrierung. Quelle für den Wert darüber. |
| `sensor.final_kwh_tagXXX` | Template-Sensor: Summe über **alle** Stations-Reader. |
| `sensor.final_co2_tagXXX` | Template-Sensor: CO₂-Äquivalent der Session. |
| `sensor.final_l_tagXXX` | Template-Sensor: Wasserverbrauch der Session (optional). |
| `script.send_tagXXX_updates` | Flush-Script: publiziert vor dem MC-Reset den noch nicht abgeholten kWh-Wert. |

`XXX` ist dreistellig: `001` … `010`. Im Labor Z108/Z115 laufen die Karten unter
`301` … `305` — das Schema bleibt gleich, nur die Nummer ändert sich.

## Was pro Karte gebraucht wird

Für **jede** zusätzliche Karte, bei *S* Stations-Readern:

| Anzahl | Helfer |
|---|---|
| 1× | `input_boolean` MC-Status |
| S× | `input_boolean` Reader-Status |
| S× | `input_number` kWh-Speicher |
| S× | Utility Meter |
| 2× | `input_text` (Experimentname + Basis-Name) |
| 2–3× | Template-Sensoren (`final_kwh`, `final_co2`, optional `final_l`) |
| 0–1× | Flush-Script (optional, aber empfohlen) |

Bei 2 Stationen sind das **12–13 Helfer pro Karte**. Bei 7 Stationen (Z108/Z115)
schon 27–28. Das ist der Grund, warum die Instanziierung dort über einen
Generator läuft — siehe [Übergabe](uebergabe.md#externe-abhängigkeiten).

## Was pro Station gebraucht wird

Beim Hinzufügen eines **neuen Stations-Readers** ist an vier Stellen nachzuziehen:

1. Pro Karte: `input_boolean`, `input_number` und Utility Meter für die neue
   Station anlegen.
2. `input_text` für das Display dieses Readers anlegen.
3. Die `final_*`-Template-Sensoren **jeder Karte** um die neue Station erweitern.
4. Im MC Session Controller die neue Station in `tagN_reader_booleans` **und**
   `tagN_reader_numbers` jeder Karte ergänzen.

Schritt 3 und 4 sind die Stellen, an denen eine unvollständige Konfiguration
stillschweigend falsche Messwerte erzeugt — siehe
[Betrieb und Fehlersuche](betrieb.md#unvollständige-reader-listen).

## Rezepte

Die folgenden Definitionen liegen außerhalb der Blueprints und gehören in die
Home-Assistant-Konfiguration. Sie sind **Beispiele** und an die eigene Anlage
anzupassen — insbesondere die Anzahl der summierten Stationen und der
CO₂-Faktor.

### Utility Meter

Pro Station und Karte einer. Wichtig ist nur, dass er per
`utility_meter.calibrate` auf 0 gesetzt werden kann; ein Zyklus ist nicht nötig,
weil die Blueprints die Kalibrierung selbst steuern.

```yaml
utility_meter:
  tr_k_01_kwh_um_tag001:
    name: "TR-K-01 kWh UM Tag001"
    source: sensor.tr_k_01_energie   # der physische Zähler der Station
    cycle: none
```

### kWh-Speicher

```yaml
input_number:
  tr_k_01_kwh_um_added_tag001:
    name: "TR-K-01 kWh added Tag001"
    min: 0
    max: 1000000
    step: 0.001
    mode: box
    unit_of_measurement: kWh
```

> `max` großzügig wählen. `input_number` klemmt still am Maximum ab — ein zu
> niedriger Wert deckelt lange Sessions, ohne dass ein Fehler im Log erscheint.

### Template-Sensoren: die Session-Summen

Diese Sensoren sind der Übergabepunkt an den MC Session Controller. Sie müssen
über **alle** Stationen summieren.

```yaml
template:
  - sensor:
      - name: "final kwh tag001"
        unique_id: final_kwh_tag001
        unit_of_measurement: kWh
        state_class: total
        device_class: energy
        state: >
          {{ (states('input_number.tr_k_01_kwh_um_added_tag001') | float(0)
            + states('input_number.tr_k_02_kwh_um_added_tag001') | float(0))
            | round(3) }}

      - name: "final co2 tag001"
        unique_id: final_co2_tag001
        unit_of_measurement: "kg"
        state_class: total
        state: >
          {# CO2-Faktor der eigenen Anlage einsetzen (kg CO2e je kWh) #}
          {% set faktor = 0.380 %}
          {{ (states('sensor.final_kwh_tag001') | float(0) * faktor) | round(3) }}

      - name: "final l tag001"
        unique_id: final_l_tag001
        unit_of_measurement: "L"
        state_class: total
        device_class: water
        state: >
          {{ states('input_number.tr_k_01_l_added_tag001') | float(0) | round(2) }}
```

Der Wasser-Sensor ist optional: bleibt das Feld `tagN_l_sensor` im MC Session
Controller leer, wird der ganze Wasser-Block beim Checkout übersprungen und gar
kein Wasser-Sensor im Experiment-Gerät angelegt. Strom und CO₂ werden immer
publiziert.

### Flush-Script

Läuft als **erster** Schritt beim Session-Start und rettet einen kWh-Wert, der
noch aus der vorherigen Session hängt (z. B. weil sie nie sauber ausgecheckt
wurde). Publiziert nur, wenn tatsächlich etwas da ist.

```yaml
script:
  send_tag001_updates:
    alias: "Flush Tag001"
    mode: single
    sequence:
      - condition: template
        value_template: "{{ states('sensor.final_kwh_tag001') | float(0) > 0 }}"
      - action: mqtt.publish
        data:
          topic: >-
            homeassistant/sensor/{{ states('input_text.experiment_tag001') }}/kwh_001/state
          payload: "{{ states('sensor.final_kwh_tag001') }}"
          retain: true
```

Das Script ist optional. Ohne Flush-Script geht ein nicht abgeholter Restwert
beim Reset verloren — das ist der einzige Effekt, die laufende Session
funktioniert trotzdem.

> **Wichtig:** Die `condition` gehört in ein eigenes **Script-Entity**, das im
> Blueprint dann per Aktion aufgerufen wird — nicht als `condition` direkt in das
> Aktionsfeld des Blueprints. Eine fehlschlagende Bedingung in einer eingebetteten
> Sequenz beendet den gesamten Automationslauf, und Reset, Namensgenerierung sowie
> Startzeit würden ausfallen. Details in
> [Betrieb und Fehlersuche](betrieb.md#zusammenspiel-mit-dem-flush-script).

### Display-Texte

Pro Reader einer, plus einer für den MC-Reader. Werden von den Blueprints
beschrieben; die ESPHome-Firmware liest sie und stellt sie dar.

```yaml
input_text:
  tr_k_01_display:
    name: "TR-K-01 Display"
    max: 255
  mc_k_01_display:
    name: "MC-K-01 Display"
    max: 255
```

> `max: 255` ist das Maximum von `input_text`. Bei 10 gleichzeitig eingeloggten
> Karten bleibt der generierte Text deutlich darunter, aber ein zu kleines `max`
> lässt `input_text.set_value` fehlschlagen.

### Experimentname und Basis-Name

```yaml
input_text:
  input_experiment_tag001:      # vom Dashboard befüllt, z. B. "Trocknung"
    name: "Basis-Name Tag001"
    max: 100
  experiment_tag001:            # vom MC Controller gesetzt, z. B. "20260831_Trocknung_A7K2"
    name: "Experiment Tag001"
    max: 255
```

`input_experiment_tag001` ist das Feld, in das Nutzer:innen vor dem Session-Start
den sprechenden Teil des Namens eintragen. `experiment_tag001` wird
ausschließlich vom MC Session Controller geschrieben und ist die MQTT-Identität
der laufenden Session — von Hand ändern heißt, die laufende Session von ihren
Messwerten zu trennen.
