# SmInt Blueprints

Home-Assistant-Blueprints für ein RFID-basiertes Energie-Mess-System.

Nutzer:innen checken sich mit einer RFID-Karte an Messstationen ein und aus. Pro
Karte und Session werden Energieverbrauch, Wasser und CO₂ erfasst und per MQTT als
eigenes Gerät veröffentlicht — ein Gerät je Experiment.

```
Karte scannen  →  input_boolean  →  Utility Meter  →  MQTT-Experiment-Gerät
```

## Dokumentation

| Dokument | Wofür |
|---|---|
| **[Architektur](docs/architektur.md)** | Wie das System funktioniert: Schichten, Datenfluss, MQTT-Struktur, Ablaufdiagramm |
| **[Installation](docs/installation.md)** | Einrichtung von null auf laufende Anlage, plus Karten und Stationen ergänzen |
| **[Entities und Helfer](docs/entities.md)** | Welche Helfer es braucht, wie sie heißen, fertige Definitionen |
| **[Blueprint-Referenz](docs/blueprint-referenz.md)** | Jedes Eingabefeld der drei Blueprints |
| **[Betrieb und Fehlersuche](docs/betrieb.md)** | Symptome, Ursachen, bekannte Einschränkungen |
| **[Entscheidungen und Historie](docs/entscheidungen.md)** | Warum es so gebaut ist, gefixte Bugs, entfernte Dateien |
| **[Übergabe](docs/uebergabe.md)** | Was außerhalb dieses Repos liegt, Checkliste, offene Punkte |

**Neu hier?** [Architektur](docs/architektur.md) lesen, dann
[Installation](docs/installation.md).
**Repo gerade übernommen?** Mit [Übergabe](docs/uebergabe.md) anfangen.

## Aufbau

| Gerät | Rolle |
|---|---|
| **MC-Reader** (Master Checkout) | Startet und beendet eine komplette Experiment-Session. Einer pro Anlage. |
| **Stations-Reader** | Messstationen, an denen der eigentliche Verbrauch gemessen wird. Beliebig viele. |
| **RFID-Karten** | Bis zu 10 Karten, jede kann an jedem Reader gescannt werden. |

Die Reader sind ESPHome-Geräte mit RFID-Leser und Display. Die UID der gescannten
Karte landet als State in einem Home-Assistant-Sensor — das ist die einzige
Schnittstelle, die dieses Repo zur Hardware voraussetzt.

## Wie es funktioniert

**Session-Start.** Eine Karte wird am MC-Reader gescannt (oder eine Session über
das Dashboard gestartet). Das System publiziert zuerst den alten Zählerstand
(„Flush"), setzt dann die Zähler auf 0, generiert einen zufälligen Experimentnamen
— dieser ist zugleich die MQTT-Geräte-ID — und veröffentlicht die Startzeit.

**Messen.** Wird dieselbe Karte an einem Stations-Reader gescannt, checkt sie dort
ein: der Utility Meter der Station wird auf 0 kalibriert, per MQTT Discovery wird
ein Sensor für das laufende Experiment angelegt, das Reader-Display zeigt die
eingeloggten Karten. Beim Auschecken wird der gemessene Verbrauch auf den
Karten-Zähler aufaddiert und per MQTT veröffentlicht.

**Session-Ende.** Karte erneut am MC-Reader scannen. Das System checkt die Karte
zuerst an allen Stations-Readern aus — damit der letzte Messwert noch erfasst wird
—, wartet kurz und veröffentlicht dann Endzeit, Gesamt-kWh, optional Wasser und
CO₂. Das MC-Display meldet „abgeschlossen".

## Die drei Blueprints

Die Logik ist in drei Schichten geschnitten, jede ist ein eigenes Blueprint.
Verbunden sind sie ausschließlich über `input_boolean`-Helfer, wodurch sich jede
Schicht einzeln testen lässt.

| Datei | Instanzen | Zweck |
|---|---|---|
| [tagreader_inpput_boolean_1_1.yaml](tagreader_inpput_boolean_1_1.yaml) | 1× pro Anlage | **Hardware-Layer.** UID + Reader → `input_boolean.toggle`. Deckt MC + bis zu 9 Stations-Reader in einer Automation ab. |
| [tagreader_reader_session_handler.yaml](tagreader_reader_session_handler.yaml) | 1× pro Station | **Reader Session Handler.** Check-in und Check-out an der Station: Meter kalibrieren, aufaddieren, MQTT publizieren, Display aktualisieren. Inklusive frei definierbarer Zusatzsensoren. |
| [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml) | 1× global | **MC Session Controller.** Session-Lebenszyklus in garantierter Reihenfolge — genau dafür existiert diese Schicht. |

Alle drei arbeiten intern **listengetrieben**: sie bauen zu Beginn eine Liste aller
Karten-Definitionen auf, suchen daraus per `trigger.entity_id` den passenden
Eintrag und laufen dann einmal generisch durch, statt eines `choose`-Zweigs pro
Karte. Eine zusätzliche Karte kostet dadurch keine Logik-Duplikation.

## Schnellstart

1. **Helfer anlegen** — `input_boolean`, `input_text`, `input_number`, Utility
   Meter, Template-Sensoren. Vorlagen in
   [Entities und Helfer](docs/entities.md).
2. **Blueprints importieren** — Einstellungen → Automationen & Szenen →
   Blueprints → Blueprint importieren, mit der Raw-GitHub-URL der YAML-Datei.
3. **Automationen erstellen** — 1× Hardware-Layer, 1× Reader Session Handler pro
   Station, 1× MC Session Controller. In allen dieselben Karten-Slots in derselben
   Reihenfolge belegen.
4. **Testlauf** — die sechs Schritte unter
   [Erste Inbetriebnahme prüfen](docs/installation.md#schritt-4--erste-inbetriebnahme-prüfen).

Ausführlich: [Installation](docs/installation.md).

**Voraussetzungen:** Home Assistant mit konfigurierter MQTT-Integration,
ESPHome-Tagreader, Utility-Meter- und Template-Sensor-Helfer.

## Karten und Stationen

**Karten:** Jedes Blueprint bietet 10 Slots. Karte 001 ist Pflicht, 002–010 sind
optional — nicht benötigte Slots bleiben leer. Warum die Obergrenze bei 10 liegt
und nicht über ein freies Listenfeld beliebig wächst, steht in
[Entscheidungen](docs/entscheidungen.md#warum-genau-10-karten-slots).

**Stationen:** Nach oben offen. Der MC Session Controller hinterlegt pro Karte
alle Stations-Booleans und kWh-Speicher als Mehrfachauswahl, der Reader Session
Handler wird ohnehin pro Station instanziiert. Einzige Grenze ist der
Hardware-Layer mit 9 Stations-Slots.

> **Die eine Stelle, an der es still schiefgehen kann:** Beide Mehrfachauswahlen im
> MC Session Controller und die `final_*`-Template-Sensoren müssen **jede** Station
> kennen. Fehlt eine, bleiben Booleans hängen oder Messwerte fehlen — ohne
> Fehlermeldung. Siehe
> [Betrieb und Fehlersuche](docs/betrieb.md#unvollständige-reader-listen).

## Status

Die Blueprints sind produktiv im Einsatz. Der Umbau auf 10 Karten ist statisch
verifiziert, aber **noch nicht mit echter Hardware durchgespielt** — siehe
[offene Punkte](docs/uebergabe.md#offene-punkte).

## Lizenz

[MIT](LICENSE) © 2026 Leonard Hermanns
