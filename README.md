# SmInt Blueprints

Home-Assistant-Blueprints für ein RFID-basiertes Energie-Mess-System ("SmInt"):
Nutzer:innen checken sich mit einer RFID-Karte an Messstationen (z. B. einer Mikrowelle)
ein und aus. Pro Karte und Session werden Energieverbrauch (kWh) und CO₂ erfasst und
per MQTT als eigenes Gerät/Experiment veröffentlicht.

> Detaillierte Interna, Entity-Namenskonventionen und Bugfix-Historie: [KONTEXT.md](KONTEXT.md)
> Ablaufdiagramm (Mermaid): [ablaufdiagramm.md](ablaufdiagramm.md)

## Hardware-Aufbau

| Gerät | Rolle |
|---|---|
| **MC-K-01** (Master-Checkout-Reader) | Startet/beendet eine komplette Experiment-Session |
| **TR-K-01 / TR-K-02** (Stations-Reader) | Messstationen, an denen der eigentliche Verbrauch gemessen wird |
| **RFID-Karten** (Tag 001 … 010) | Bis zu 10 Karten, jede kann an jedem Reader gescannt werden |

Die Reader sind ESPHome-Geräte mit RFID-Leser und Display; der UID-Wert der gescannten
Karte landet als State in einem Home-Assistant-Sensor.

## Wie es funktioniert — inhaltlich

**Session-Start:** Eine Karte wird am MC-Reader gescannt (oder eine Session wird über
das Dashboard gestartet). Das System vergisst den alten Zählerstand nicht, sondern
publiziert ihn erst ("Flush"), setzt dann die Zähler auf 0, generiert einen zufälligen
Experimentnamen (dient als eindeutige MQTT-Geräte-ID) und veröffentlicht die Startzeit.

**Messen:** Wird dieselbe Karte an einem Stations-Reader gescannt, checkt sie dort ein:
Der Utility Meter der Station wird auf 0 kalibriert, per MQTT Discovery wird ein Sensor
für das laufende Experiment angelegt, das Reader-Display zeigt die eingeloggten Karten.
Beim Auschecken an der Station wird der gemessene Verbrauch auf den Karten-Zähler
aufaddiert und per MQTT veröffentlicht.

**Session-Ende:** Karte erneut am MC-Reader scannen. Das System checkt die Karte
zuerst an allen Stations-Readern aus (damit der letzte Messwert noch erfasst wird),
wartet kurz, und veröffentlicht dann Endzeit, Gesamt-kWh und CO₂ des Experiments.
Das MC-Display meldet "abgeschlossen".

## Wie es funktioniert — code-seitig

Die Logik ist in drei Schichten aufgeteilt, jede Schicht ist ein eigenes Blueprint:

1. **Hardware-Layer** — [tagreader_inpput_boolean_1_1.yaml](tagreader_inpput_boolean_1_1.yaml)
   Eine einzige Automation für alle 3 Reader. `state`-Trigger auf den drei
   Reader-Sensoren; die gescannte UID wird in der Kartenliste nachgeschlagen und
   das passende `input_boolean` getoggelt (z. B.
   `input_boolean.tr_k_01_switch_tag001`). Die Booleans sind die einzige
   Schnittstelle zu den nächsten Schichten — die Reader-Hardware ist damit
   vollständig entkoppelt. `mode: queued` verhindert verlorene Scans.

2. **Reader Session Handler** — [tagreader_reader_session_handler.yaml](tagreader_reader_session_handler.yaml)
   Pro Stations-Reader einmal instanziiert (TR-K-01, TR-K-02). Triggert auf die
   Reader-lokalen Tag-Booleans.
   *ON-Zweig:* MC-Boolean einschalten falls noch aus (startet implizit die Session),
   Utility Meter per `utility_meter.calibrate` auf 0 setzen, MQTT-Discovery-Config
   für den Experiment-Sensor publizieren, ESPHome-IN-Aktion ausführen.
   *OFF-Zweig:* Meter-Delta auf das `input_number` aufaddieren, MQTT-State
   publizieren, ESPHome-OUT-Aktion. Danach wird in beiden Fällen der Display-Text
   des Readers per Jinja-Template neu berechnet (`input_text.set_value`).
   Pro Karte lassen sich zusätzlich beliebig viele weitere Messgrößen als Liste
   hinterlegen (Passthrough oder Meter-basiert, z. B. Argon- oder Gasverbrauch).

3. **MC Session Controller** — [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml)
   Einmal global instanziiert. Triggert auf die MC-Booleans. Pro Karte werden
   die Booleans und kWh-Speicher **aller** Stations-Reader als Mehrfachauswahl
   hinterlegt — die Anzahl der Stationen ist damit beliebig. Kapselt die
   Session-Lebenszyklus-Logik in **garantierter Reihenfolge** (früher waren das
   mehrere parallel laufende Automationen mit Race Conditions, siehe KONTEXT.md):
   *ON:* Flush-Script → Zähler-Reset → Zufallsnamen-Generierung → MQTT-Startzeit →
   ESPHome IN. *OFF:* Reader-Booleans ausschalten (triggert Schicht 2, die den
   finalen Messwert befüllt) → 1 s warten → MQTT Endzeit/kWh/CO₂ → ESPHome OUT →
   Display-Meldung.

Alle drei Blueprints arbeiten intern **listengetrieben**: sie bauen zu Beginn eine
Liste aller Karten-Definitionen auf, suchen daraus per `trigger.entity_id` den
passenden Eintrag und laufen dann einmal generisch durch — statt eines
`choose`-Zweigs pro Karte. Deshalb kostet eine zusätzliche Karte keine
Logik-Duplikation mehr, sondern nur einen Listeneintrag.

Datenfluss der Messwerte: `sensor.tr_k_0N_kwh_um_tagXXX` (Utility Meter, Delta seit
Kalibrierung) → `input_number.tr_k_0N_kwh_um_added_tagXXX` (aufaddiert pro Session) →
`sensor.final_kwh_tagXXX` / `sensor.final_co2_tagXXX` (Template-Sensoren, Summe über
beide Stationen) → finale MQTT-Publikation beim MC-Checkout.

## Karten hinzufügen

Jedes Blueprint bietet **10 Karten-Slots**. Karte 001 ist Pflicht, 002–010 sind
optional — nicht benötigte Slots bleiben einfach leer.

Da die Blueprints intern listengetrieben arbeiten, ist ein weiterer Slot kein
Logik-Umbau mehr, sondern nur ein zusätzlicher Input-Block, ein Trigger und eine
Zeile in `tag_defs`. Warum die Obergrenze trotzdem fest bei 10 liegt und nicht
über ein freies Listenfeld beliebig wächst, steht in
[KONTEXT.md](KONTEXT.md#karten-modell-10-feste-slots).

## Dateiübersicht

| Datei | Zweck |
|---|---|
| [tagreader_inpput_boolean_1_1.yaml](tagreader_inpput_boolean_1_1.yaml) | Hardware-Layer: UID → `input_boolean.toggle` |
| [tagreader_reader_session_handler.yaml](tagreader_reader_session_handler.yaml) | Pro-Reader-Logik (Check-in/Check-out an der Station, inkl. Zusatzsensoren) |
| [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml) | Globale Session-Steuerung (Start/Ende, Reihenfolge-Garantien) |
| [beispiel_z108_z115_automations.yaml](beispiel_z108_z115_automations.yaml) | Beispiel-Instanziierung: Labor Z108/Z115 mit 5 Karten, 1 MC- und 7 Stations-Readern |

Fünf ältere Blueprints sowie der frühere Testkandidat
`tagreader_reader_session_handler_v2.yaml` wurden entfernt, weil sie in den drei
aktiven Dateien aufgegangen sind. Sie bleiben über die Git-Historie
wiederherstellbar — welche Datei wodurch ersetzt wurde und aus welchem Commit sie
sich zurückholen lässt, steht in
[KONTEXT.md](KONTEXT.md#entfernte-blueprints-stand-6-august-2026).

## Installation

1. Blueprint in Home Assistant importieren:
   **Einstellungen → Automationen & Szenen → Blueprints → Blueprint importieren**
   und die Raw-GitHub-URL der jeweiligen YAML-Datei einfügen.
2. Benötigte Helfer anlegen (`input_boolean`, `input_text`, `input_number`,
   Utility Meter, Template-Sensoren) — Namensschema siehe
   [KONTEXT.md](KONTEXT.md#wichtige-entity-namenskonventionen).
3. Automationen aus den Blueprints erstellen:
   - 1× Hardware-Layer (alle 3 Reader in einer Automation)
   - 1× Reader Session Handler **pro Stations-Reader**
   - 1× MC Session Controller

   In allen drei Automationen dieselben Karten-Slots in derselben Reihenfolge
   belegen (Karte 001 = Slot 1 usw.) — das erspart beim Debuggen viel Sucherei.
4. Nach Änderungen an einem Blueprint: in HA **Re-Import** ausführen, damit die
   Automationen die neue Version übernehmen.

Voraussetzungen: Home Assistant mit MQTT-Integration (Broker konfiguriert),
ESPHome-Tagreader, Utility-Meter- und Template-Sensor-Helfer.

## Lizenz

[MIT](LICENSE) © 2026 Leonard Hermanns
