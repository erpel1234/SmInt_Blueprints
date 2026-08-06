# SmInt Blueprints — Kontext & Übersicht

Dieses Dokument fasst zusammen, wie die Blueprints in diesem Ordner zusammenspielen, welche Entity-Namenskonventionen verwendet werden, und welche Bugs im Laufe der Zeit gefunden und gefixt wurden. Gedacht als Gedächtnisstütze, falls man nach einer Pause wieder einsteigt.

Ablaufdiagramm: siehe [ablaufdiagramm.md](ablaufdiagramm.md).

## Hardware-Aufbau

- **1 MC-Reader** (MC-K-01): "Master Checkout" — startet/beendet eine komplette Experiment-Session.
- **2 Stations-Reader** (TR-K-01, TR-K-02): wo die eigentliche Nutzung (z.B. Mikrowelle) gemessen wird.
- **Karten-Slots**: 10 Karten, jede kann an jedem Reader gescannt werden. Karte 001 ist Pflicht, 002-010 optional.

Der Aufbau oben beschreibt die ursprüngliche Installation. Die Blueprints sind nicht darauf festgelegt — das seit dem 6. August 2026 im Aufbau befindliche **Labor Z108/Z115** hat 5 Karten (301-305), einen MC-Reader und **sieben** Stations-Reader (TR-Z108-01…04, TR-Z115-01…03). Die Instanziierung dafür erzeugt der Generator (`code_generator/app_v3.js`).

## Die Blueprints

Alle drei sind aktiv und produktiv. Es gibt seit dem 10-Karten-Umbau keine "in Testing"-Variante mehr.

| Datei | Zweck |
|---|---|
| [tagreader_inpput_boolean_1_1.yaml](tagreader_inpput_boolean_1_1.yaml) | Hardware-Layer. Liest UID vom jeweiligen Reader-Sensor, togglet das passende `input_boolean` (z.B. `r1_tag1_boolean`). 1 Automation für die gesamte Anlage: MC + bis zu 9 Stations-Reader × 10 Karten. |
| [tagreader_reader_session_handler.yaml](tagreader_reader_session_handler.yaml) | Pro-Reader-Logik (1x für TR-K-01, 1x für TR-K-02 instanziiert). Bei Karten-Boolean ON: MC-Boolean einschalten falls aus, Utility Meter kalibrieren, MQTT-Discovery-Sensor publizieren, ESPHome IN. Bei OFF: Meter-Wert aufaddieren, MQTT-State publizieren, ESPHome OUT. Danach Display dieses Readers aktualisieren. Zusätzlich pro Karte ein freies Listenfeld für beliebig viele Zusatzsensoren (Passthrough oder Meter-basiert). |
| [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml) | Globale MC-Logik (1x instanziiert). Bei MC-Boolean ON: alten kWh-Wert flushen → R1/R2-Zähler resetten → neuen Zufallsnamen generieren → MQTT Startzeit → ESPHome IN. Bei OFF: Reader-Booleans aufräumen → MQTT Endzeit/kWh/CO2 → ESPHome OUT → Display "abgeschlossen". |

## Karten-Modell: 10 feste Slots

Alle drei Blueprints arbeiten intern **listengetrieben**: die Aktionen kennen keine festen Karten-Nummern mehr, sondern bauen zu Beginn eine Liste `tag_defs` aus allen 10 Karten-Definitionen, suchen daraus per `trigger.entity_id` den passenden Eintrag (`this_tag`) und laufen dann genau einmal generisch durch — statt eines `choose`-Zweigs pro Karte. Die 10 GUI-Slots sind die Eingabemasken für diese Liste.

Karte 001 ist Pflicht, 002-010 optional — nicht benötigte Slots einfach leer lassen. Alle optionalen Felder haben `default: []`; ein leerer Slot liefert damit `[]` statt einer Entity-ID und fällt bei der Auswertung durch `select('string')` heraus.

**Warum ausgerechnet 10 und nicht beliebig viele?** Eine unbegrenzte Karten-Liste wurde geprüft und bewusst verworfen (Entscheidung vom 6. August 2026, siehe [Offene Punkte](#offene-punkte--future-work)). Technisch ginge es nur über ein Freitext-`object`-Feld, in das man Entity-IDs von Hand einträgt, plus eine separate Mehrfachauswahl der Trigger-Booleans — denn HA-Blueprint-Trigger brauchen statische Entity-IDs und `!input`-Werte können nicht aus einem Template kommen. Das kostet alle Entity-Picker, Domain-Filter und Plausibilitätsprüfungen der Oberfläche und ist beim Einrichten deutlich fehleranfälliger. Falls doch mehr als 10 Karten gebraucht werden: die Engine ist bereits listengetrieben, ein weiterer Slot ist nur ein zusätzlicher Input-Block, ein Trigger und eine Zeile in `tag_defs`.

**Warum ein einzelnes Flush-Script pro Slot im MC Controller?** `action`-Selektoren (Scripts als eingebettete Aktionssequenz) sind nicht template-fähig — man kann sie nicht über `this_tag` auswählen. Deshalb steht im ON-Zweig ein `choose` mit einem Zweig je Slot, der nur das jeweilige Flush-Script aufruft. Das ist die einzige Stelle, die pro Karte noch ausgeschrieben ist.

### Reader-Anzahl

Der **MC Session Controller** kennt pro Karte zwei Mehrfachauswahl-Felder: `tagN_reader_booleans` (alle Stations-Booleans dieser Karte, werden beim Checkout ausgeschaltet) und `tagN_reader_numbers` (alle kWh-Speicher, werden beim Session-Start auf 0 gesetzt). Damit sind beliebig viele Stationen möglich, ohne das Blueprint anzufassen.

Beide Listen müssen **vollständig** sein — das ist die einzige Stelle, an der eine unvollständige Konfiguration stillschweigend falsche Messwerte erzeugt:

- fehlt ein Reader in `tagN_reader_booleans`, bleibt sein Boolean nach Session-Ende auf `on` hängen und sein letzter Messwert fehlt in der Session (Bug 4 aus der Liste unten, nur über die Reader-Achse)
- fehlt einer in `tagN_reader_numbers`, läuft sein alter kWh-Wert in die nächste Session weiter

Die Template-Sensoren `sensor.final_kwh_tagXXX` / `sensor.final_co2_tagXXX` liegen außerhalb der Blueprints und müssen ebenfalls über alle Stationen summieren.

Der **Reader Session Handler** skaliert ohnehin, weil er pro Reader einmal instanziiert wird.

Der **Hardware-Layer** hat 10 Reader-Slots: `reader_mc` (Checkout) und `reader_1` … `reader_9` (Stationen). Jeder Slot braucht einen eigenen `state`-Trigger mit eigener Trigger-ID, weil Trigger nicht aus einer Liste entstehen können — deshalb feste Slots statt einer Mehrfachauswahl wie im MC Controller.

Die Zuordnung ist eine Matrix: pro Karte je ein `input_boolean` pro Reader (`mc_tagN_boolean`, `r1_tagN_boolean` … `r9_tagN_boolean`), also 100 Felder plus 10 Sensoren plus 10 UIDs = 120 Inputs. Ausgefüllt wird nur, was existiert; ein leeres Feld heißt „diese Karte wird an diesem Reader nicht verwendet".

Der Trick, der die Aktionslogik trotzdem kurz hält: die **Trigger-ID ist zugleich der Schlüssel im Karten-Dict**. `trigger.id` ist `mc`/`r1`…`r9`, und der Lookup ist ein einziges `hit[0].get(trigger.id, '')` — unabhängig davon, wie viele Reader-Slots es gibt.

Vor dem 6. August 2026 hatte dieses Blueprint nur 3 Reader-Slots, was bei mehr Readern mehrere Instanzen mit jeweils identischer UID-Liste erzwang. Das war die einzige Stelle im System, an der dieselbe Information mehrfach zu pflegen war, und ist mit der Erweiterung auf 10 Slots weggefallen.

## Wichtige Entity-Namenskonventionen

- `input_boolean.mc_k_01_switch_tagXXX` — MC-seitiger "ist Karte XXX gerade eingecheckt"-Status (global, eine Quelle der Wahrheit für die ganze Session).
- `input_boolean.tr_k_0N_switch_tagXXX` — Reader-seitiger Status pro Station (N = 01/02).
- `input_text.experiment_tagXXX` — aktueller, zufällig generierter Experimentname für Karte XXX (wird als MQTT-Geräte-Identifier verwendet).
- `input_text.input_experiment_tagXXX` — Basis-Name, vom Dashboard befüllt, Ausgangspunkt für die Zufallsgenerierung.
- `input_number.tr_k_0N_kwh_um_added_tagXXX` — aufaddierter kWh-Wert pro Reader+Karte seit letztem MC-Reset.
- `sensor.tr_k_0N_kwh_um_tagXXX` — Utility Meter (Delta seit letzter Kalibrierung), Quelle für obigen Wert.
- `sensor.final_kwh_tagXXX` / `sensor.final_co2_tagXXX` — Template-Sensoren, Summe aus TR-K-01 + TR-K-02 für die finale MC-Veröffentlichung.
- `script.send_tagXXX_updates` — Flush-Script, publiziert vor dem MC-Reset den noch nicht abgeholten kWh-Wert per MQTT (nur wenn > 0).

Für Karten 004-010 gilt dasselbe Schema mit `tag004` … `tag010`. Beim Anlegen der Helfer für eine neue Karte werden pro Karte gebraucht: 1× MC-Boolean, 1× Boolean je Stations-Reader, 1× `input_number` je Stations-Reader, 1× Utility Meter je Stations-Reader, 2× `input_text` (Experimentname + Basis-Name), 2× Template-Sensor (final kWh + CO2) und optional 1× Flush-Script.

## Entfernte Blueprints (Stand: 6. August 2026)

Die folgenden Dateien wurden aus dem Arbeitsverzeichnis **gelöscht**, weil sie durch die drei aktiven Blueprints vollständig ersetzt sind. Sie sind nicht verloren — der letzte Commit, in dem sie enthalten sind, ist **`e3e26b8`**; wiederherstellbar mit z.B. `git show e3e26b8:tagreader_smint.yaml` oder `git checkout e3e26b8 -- <datei>`.

| Entfernte Datei | Wurde ersetzt durch |
|---|---|
| `tagreader_smint.yaml` | tagreader_reader_session_handler.yaml |
| `tagreader_device_creation.yaml` | tagreader_reader_session_handler.yaml |
| `tagreader_ergaenzungunddasandere.yaml` | tagreader_mc_session_controller.yaml |
| `tagreader_mc_device_erzeugen.yaml` | tagreader_mc_session_controller.yaml |
| `tagreader_generate_random_experiment_name.yaml` | Logik ist inline in tagreader_mc_session_controller.yaml |
| `tagreader_reader_session_handler_v2.yaml` | in tagreader_reader_session_handler.yaml aufgegangen (siehe unten) |

**Zu v2:** Der frühere Testkandidat `tagreader_reader_session_handler_v2.yaml` (10 Karten + Zusatzsensor-Listen) ist jetzt der reguläre Reader Session Handler. Die Input-Schlüssel sind identisch geblieben, eine bestehende Test-Automation muss also nur in ihrem YAML auf den neuen Blueprint-Pfad zeigen (`use_blueprint: path:`), die konfigurierten Werte bleiben gültig.

Bevor die alten Blueprints ganz verschwinden: **die zugehörigen Automationen in Home Assistant müssen deaktiviert bzw. gelöscht sein**, sonst laufen sie ins Leere.

## Bekannte, bereits gefixte Bugs

1. **Tag002/003 Auto-Checkout-Bug**: In der alten `tagreader_generate_random_experiment_name.yaml` wurde `mc_tagX_boolean` für Tag 2/3 direkt nach der Namensvergabe wieder ausgeschaltet → sofortiges, ungewolltes Checkout. Tag 1 war korrekt (turn_off war dort schon auskommentiert). Fix: turn_off für alle Tags entfernt, Logik jetzt ohnehin inline im MC Session Controller.
2. **Flush-vs-Reset-Race**: Das Flush-Script und der Zähler-Reset liefen als zwei separate, parallele Automationen ohne garantierte Reihenfolge. Fix: beides läuft jetzt sequenziell innerhalb von `tagreader_mc_session_controller.yaml` (Flush garantiert vor Reset).
3. **Doppeltes ESPHome-Signal beim Checkout**: Die alte `tagreader_mc_device_erzeugen.yaml` rief `esphome_in_action` sowohl bei ON als auch bei OFF auf, zusätzlich zum separaten `esphome_out` aus `tagreader_ergaenzungunddasandere.yaml`. Fix: `esphome_in_action`/`esphome_out_action` sind jetzt getrennte Inputs im MC Session Controller.
4. **Direkter MC-Checkout ohne vorherigen Reader-Checkout**: Die finalen kWh/CO2-Werte wurden gelesen, bevor die Reader-Booleans (und damit der zugrundeliegende Zähler) aufgeräumt waren — Ergebnis: 0 kWh, wenn man nicht vorher manuell am Reader ausgecheckt hat. Fix: Reihenfolge in der OFF-Sequenz umgedreht — Reader-Booleans zuerst ausschalten (löst Reader Session Handler aus), 1s warten, erst dann kWh/CO2 publizieren.
5. **`manufacturer` war global statt pro Karte**: Sollte die jeweilige physische Karte repräsentieren ("Karte 001/002/003"), war aber ein einzelner globaler Wert für alle Tags. Fix: `tagN_manufacturer` als eigener Input je Karte.
6. **Leerer Reader-State konnte einen leeren UID-Slot treffen**: Im Hardware-Layer wurde die gescannte UID direkt mit den konfigurierten UIDs verglichen. Bei 10 Slots sind die meisten UID-Felder leer, und ein Reader-Sensor, der auf `unknown`/`unavailable`/leer wechselt, hätte auf so einen leeren Slot gematcht. Fix: expliziter Guard, der leere und ungültige Reader-States vor dem Abgleich verwirft.

## Offene Punkte / Future Work

- Die 10-Karten-Blueprints sind statisch (YAML-Parsing, Template-Logik) verifiziert, aber **noch nicht in Home Assistant mit echter Hardware getestet**. Vor dem Rollout im neuen Labor: mit 1-2 Karten durchspielen, insbesondere den MC-OFF-Zweig (Reihenfolge Cleanup → kWh/CO2).
- **Unbegrenzt viele Karten wurden bewusst nicht umgesetzt** (Entscheidung vom 6. August 2026). Eine Variante mit Freitext-Kartenliste + separater Trigger-Auswahl war gebaut und funktionsfähig, wurde aber zugunsten der einfacheren 10-Slot-Lösung wieder entfernt — Begründung siehe Abschnitt "Karten-Modell". Wiederherstellbar aus der Git-Historie, falls der Bedarf doch kommt.
- Der Hardware-Layer deckt MC + 9 Stations-Reader in **einer** Automation ab (Stand 6. August 2026, vorher 3). Für Z108/Z115 (MC + 7) reicht das; bei einem zehnten Stations-Reader müsste ein weiterer Slot ergänzt werden — das sind ein Sensor-Input, ein Trigger und eine Spalte in der Karten-Matrix.
- **UID-Format prüfen**: der Vergleich im Hardware-Layer ist exakt und case-sensitiv. Die Karten des neuen Labors haben das Format `13-CF-91-2A` (Bindestriche, Großbuchstaben), die ursprüngliche Installation `04A224B91C2A80` — also unterschiedliche Reader-Firmware. Bei Abweichung passiert schlicht nichts, ohne Fehlermeldung. Nach dem ersten Scan den State des Reader-Sensors ansehen und die Schreibweise angleichen.
- `mode: parallel` läuft in beiden Session-Blueprints mit `max: 25` (vorher Default 10) — bei 10 Karten × 2 Readern reicht das.
- Der Reader Session Handler ruft beim Auschecken pro Zusatzsensor ein `delay: 1s` auf. Bei vielen Zusatzsensoren summiert sich das; falls das stört, ließe sich das Publish-Muster bündeln.
