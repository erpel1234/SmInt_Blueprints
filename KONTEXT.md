# SmInt Blueprints — Kontext & Übersicht

Dieses Dokument fasst zusammen, wie die Blueprints in diesem Ordner zusammenspielen, welche Entity-Namenskonventionen verwendet werden, und welche Bugs im Laufe der Zeit gefunden und gefixt wurden. Gedacht als Gedächtnisstütze, falls man nach einer Pause wieder einsteigt.

Ablaufdiagramm: siehe [ablaufdiagramm.md](ablaufdiagramm.md).

## Hardware-Aufbau

- **1 MC-Reader** (MC-K-01): "Master Checkout" — startet/beendet eine komplette Experiment-Session.
- **2 Stations-Reader** (TR-K-01, TR-K-02): wo die eigentliche Nutzung (z.B. Mikrowelle) gemessen wird.
- **3 Karten-Slots** (Tag 001/002/003): jede der 3 physischen RFID-Karten kann an jedem der 3 Reader gescannt werden.

## Die Blueprints

### Aktiv / produktiv genutzt

| Datei | Zweck |
|---|---|
| [tagreader_inpput_boolean_1_1.yaml](tagreader_inpput_boolean_1_1.yaml) | Hardware-Layer. Liest UID vom jeweiligen Reader-Sensor, togglet das passende `input_boolean` (z.B. `r1_tag1_boolean`). 1 Automation für alle 3 Reader. |
| [tagreader_reader_session_handler.yaml](tagreader_reader_session_handler.yaml) | Pro-Reader-Logik (1x für TR-K-01, 1x für TR-K-02 instanziiert). Bei Karten-Boolean ON: MC-Boolean einschalten falls aus, Utility Meter kalibrieren, MQTT-Discovery-Sensor publizieren, ESPHome IN. Bei OFF: Meter-Wert aufaddieren, MQTT-State publizieren, ESPHome OUT. Danach Display dieses Readers aktualisieren. |
| [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml) | Globale MC-Logik (1x instanziiert). Bei MC-Boolean ON: alten kWh-Wert flushen → R1/R2-Zähler resetten → neuen Zufallsnamen generieren → MQTT Startzeit → ESPHome IN. Bei OFF: Reader-Booleans aufräumen → MQTT Endzeit/kWh/CO2 → ESPHome OUT → Display "abgeschlossen". |

### In Testing, noch nicht produktiv

| Datei | Zweck |
|---|---|
| [tagreader_reader_session_handler_v2.yaml](tagreader_reader_session_handler_v2.yaml) | Erweiterte Version von `tagreader_reader_session_handler.yaml`: bis zu 10 Karten-Slots statt 3, plus pro Karte ein freies Listenfeld für beliebig viele Zusatzsensoren (Passthrough oder Meter-basiert). Soll TR-K-01/02 perspektivisch ersetzen, nachdem es ausreichend getestet wurde. |

### Abgelöst, nur als Rollback-Referenz behalten

Diese 5 Blueprints wurden durch die zwei aktiven oben ersetzt. Die zugehörigen Automationen sollten deaktiviert sein.

| Datei | Wurde ersetzt durch |
|---|---|
| [tagreader_smint.yaml](tagreader_smint.yaml) | tagreader_reader_session_handler.yaml |
| [tagreader_device_creation.yaml](tagreader_device_creation.yaml) | tagreader_reader_session_handler.yaml |
| [tagreader_ergaenzungunddasandere.yaml](tagreader_ergaenzungunddasandere.yaml) | tagreader_mc_session_controller.yaml |
| [tagreader_mc_device_erzeugen.yaml](tagreader_mc_device_erzeugen.yaml) | tagreader_mc_session_controller.yaml |
| [tagreader_generate_random_experiment_name.yaml](tagreader_generate_random_experiment_name.yaml) | Logik ist inline in tagreader_mc_session_controller.yaml |

## Wichtige Entity-Namenskonventionen

- `input_boolean.mc_k_01_switch_tagXXX` — MC-seitiger "ist Karte XXX gerade eingecheckt"-Status (global, eine Quelle der Wahrheit für die ganze Session).
- `input_boolean.tr_k_0N_switch_tagXXX` — Reader-seitiger Status pro Station (N = 01/02).
- `input_text.experiment_tagXXX` — aktueller, zufällig generierter Experimentname für Karte XXX (wird als MQTT-Geräte-Identifier verwendet).
- `input_text.input_experiment_tagXXX` — Basis-Name, vom Dashboard befüllt, Ausgangspunkt für die Zufallsgenerierung.
- `input_number.tr_k_0N_kwh_um_added_tagXXX` — aufaddierter kWh-Wert pro Reader+Karte seit letztem MC-Reset.
- `sensor.tr_k_0N_kwh_um_tagXXX` — Utility Meter (Delta seit letzter Kalibrierung), Quelle für obigen Wert.
- `sensor.final_kwh_tagXXX` / `sensor.final_co2_tagXXX` — Template-Sensoren, Summe aus TR-K-01 + TR-K-02 für die finale MC-Veröffentlichung.
- `script.send_tagXXX_updates` — Flush-Script, publiziert vor dem MC-Reset den noch nicht abgeholten kWh-Wert per MQTT (nur wenn > 0).

## Bekannte, bereits gefixte Bugs

1. **Tag002/003 Auto-Checkout-Bug**: In der alten `tagreader_generate_random_experiment_name.yaml` wurde `mc_tagX_boolean` für Tag 2/3 direkt nach der Namensvergabe wieder ausgeschaltet → sofortiges, ungewolltes Checkout. Tag 1 war korrekt (turn_off war dort schon auskommentiert). Fix: turn_off für alle 3 Tags auskommentiert, Logik jetzt ohnehin inline im MC Session Controller.
2. **Flush-vs-Reset-Race**: Das Flush-Script und der Zähler-Reset liefen als zwei separate, parallele Automationen ohne garantierte Reihenfolge. Fix: beides läuft jetzt sequenziell innerhalb von `tagreader_mc_session_controller.yaml` (Flush garantiert vor Reset).
3. **Doppeltes ESPHome-Signal beim Checkout**: Die alte `tagreader_mc_device_erzeugen.yaml` rief `esphome_in_action` sowohl bei ON als auch bei OFF auf, zusätzlich zum separaten `esphome_out` aus `tagreader_ergaenzungunddasandere.yaml`. Fix: `esphome_in_action`/`esphome_out_action` sind jetzt getrennte Inputs im MC Session Controller.
4. **Direkter MC-Checkout ohne vorherigen Reader-Checkout**: Die finalen kWh/CO2-Werte wurden gelesen, bevor die Reader-Booleans (und damit der zugrundeliegende Zähler) aufgeräumt waren — Ergebnis: 0 kWh, wenn man nicht vorher manuell am Reader ausgecheckt hat. Fix: Reihenfolge in der OFF-Sequenz umgedreht — Reader-Booleans zuerst ausschalten (löst Reader Session Handler aus), 1s warten, erst dann kWh/CO2 publizieren.
5. **`manufacturer` war global statt pro Karte**: Sollte die jeweilige physische Karte repräsentieren ("Karte 001/002/003"), war aber ein einzelner globaler Wert für alle 3 Tags. Fix: `tag1_manufacturer`/`tag2_manufacturer`/`tag3_manufacturer` als eigene Inputs.

## Offene Punkte / Future Work

- `tagreader_mc_session_controller.yaml` unterstützt weiterhin nur 3 Karten. Falls `tagreader_reader_session_handler_v2.yaml` (10 Karten) produktiv geht, bräuchten Karten 4-10 ebenfalls einen vollständigen MC-Checkout-Lebenszyklus (Namensgenerierung, Reset, finale Aggregation) — bisher nicht geplant/umgesetzt.
- TR-K-02 auf den neuen `tagreader_reader_session_handler.yaml` (bzw. später v2) umzustellen wurde vorgeschlagen, Status der tatsächlichen Umstellung in Home Assistant ist hier nicht verifiziert — im Zweifel in HA nachsehen, welche Automationen aktiv sind.
- `tagreader_reader_session_handler_v2.yaml` ist nur in einer Test-Automation ausprobiert, noch nicht für TR-K-01/02 produktiv im Einsatz.
