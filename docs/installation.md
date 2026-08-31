# Installation

Vollständige Einrichtung von null auf eine laufende Anlage. Wer nur eine Karte
oder eine Station ergänzen will, springt direkt zu
[Anlage erweitern](#anlage-erweitern).

## Voraussetzungen

| | |
|---|---|
| **Home Assistant** | Aktuelle Version. Die Blueprints nutzen `utility_meter.calibrate`, `repeat/for_each`, Blueprint-Sections und `selector: action` — alles seit HA 2024.6 vorhanden. |
| **MQTT-Integration** | Broker konfiguriert und verbunden. MQTT Discovery muss aktiv sein (Standard-Präfix `homeassistant`). |
| **ESPHome-Reader** | Pro Reader ein Sensor, dessen **State die gescannte UID** ist, plus ein Weg, Display-Text aus einem `input_text` zu lesen. |
| **Helfer** | Utility Meter, Template-Sensoren, `input_boolean`/`input_text`/`input_number` — siehe [Entities und Helfer](entities.md). |

## Schritt 1 — Helfer anlegen

**Vor** dem Import der Blueprints. Die Blueprints haben nur Entity-Picker; was
nicht existiert, lässt sich nicht auswählen.

Vorgehen: [Entities und Helfer](entities.md) durchgehen und pro Karte und pro
Station alle dort gelisteten Helfer anlegen. Bei mehr als zwei bis drei Karten
lohnt es sich, das über YAML-Packages statt über die Oberfläche zu machen — die
Definitionen sind hochgradig repetitiv.

Ergebnis dieses Schritts: die vollständige Helfer-Matrix existiert und die
`final_*`-Template-Sensoren liefern plausible Werte (`0`, nicht `unavailable`).

## Schritt 2 — Blueprints importieren

In Home Assistant: **Einstellungen → Automationen & Szenen → Blueprints →
Blueprint importieren**, und die Raw-GitHub-URL der jeweiligen YAML-Datei
einfügen.

Alle drei importieren:

| Blueprint | Datei |
|---|---|
| Hardware-Layer | `tagreader_inpput_boolean_1_1.yaml` |
| Reader Session Handler | `tagreader_reader_session_handler.yaml` |
| MC Session Controller | `tagreader_mc_session_controller.yaml` |

> Die Blueprints haben **kein** `source_url`-Feld. Home Assistant kann sie
> deshalb nicht per „Blueprint aktualisieren" nachziehen — nach einer Änderung
> im Repo muss der Import erneut über die URL erfolgen. Siehe
> [Übergabe](uebergabe.md#offene-punkte).

## Schritt 3 — Automationen erstellen

**Reihenfolge einhalten**, sonst fehlen beim jeweils nächsten Schritt die
Referenzen:

1. **1× Hardware-Layer** für die gesamte Anlage.
2. **1× Reader Session Handler pro Stations-Reader** (bei zwei Stationen also zwei
   Automationen).
3. **1× MC Session Controller** global.

> **Die wichtigste Regel:** In allen drei Automationen dieselben Karten-Slots in
> derselben Reihenfolge belegen. Karte 001 gehört überall in Slot 1, Karte 002 in
> Slot 2 und so weiter. Die Blueprints erzwingen das nicht — sie finden ihre
> Karte über die Entity-ID, nicht über die Slot-Nummer. Aber jede spätere
> Fehlersuche wird ohne diese Disziplin zur Sucharbeit.

Die vollständige Beschreibung jedes einzelnen Eingabefelds steht in der
[Blueprint-Referenz](blueprint-referenz.md).

### 3a — Hardware-Layer

1. Abschnitt **Reader-Sensoren**: den MC-Reader-Sensor auswählen, dann die
   Stations-Reader in `reader_1` … `reader_9`. Nicht benutzte Slots leer lassen.
2. Pro Karte: die **UID exakt** eintragen (siehe
   [UID-Format](#uid-format-prüfen)) und die `input_boolean` je Reader zuordnen.
3. Ein leeres Boolean-Feld bedeutet „diese Karte wird an diesem Reader nicht
   verwendet" und ist völlig in Ordnung.

### 3b — Reader Session Handler (pro Station)

1. Abschnitt **Allgemein**: Display-`input_text` dieses Readers, ESPHome-IN- und
   -OUT-Aktion, sowie:
   - `reader_id` — Suffix im MQTT-Topic, muss **je Station eindeutig** sein
     (`tr_k_01_kwh`, `tr_k_02_kwh`, …). Zwei Stationen mit derselben `reader_id`
     überschreiben sich gegenseitig die Messwerte.
   - `sensor_name`, `unit_of_measurement`, `device_class` — Anzeige des primären
     Sensors.
2. Pro Karte: Tag-Boolean **dieser Station**, das MC-Boolean, den kWh-Speicher,
   den Utility Meter und den Experimentnamen-`input_text` zuweisen.
3. Optional: Zusatzsensoren als YAML-Liste eintragen, Format siehe
   [Blueprint-Referenz](blueprint-referenz.md#zusatzsensoren).

### 3c — MC Session Controller

1. Abschnitt **Allgemein**: `station_id` (Suffix der Gesamtwert-Topics,
   z. B. `001`), MC-Display-`input_text`, ESPHome-IN-/-OUT-Aktion.
2. Pro Karte:
   - MC-Boolean (der Trigger)
   - **`tagN_reader_booleans`** — Mehrfachauswahl **aller** Stations-Booleans
     dieser Karte
   - **`tagN_reader_numbers`** — Mehrfachauswahl **aller** kWh-Speicher dieser Karte
   - Experimentname-`input_text` und Basis-Namen-`input_text`
   - finaler kWh-Sensor, finaler CO₂-Sensor, optional finaler Wasser-Sensor
   - optional das Flush-Script

> Die beiden Mehrfachauswahlen sind die einzige Stelle im System, an der eine
> unvollständige Konfiguration **stillschweigend** falsche Messwerte erzeugt.
> Nach jeder Änderung an der Stationszahl beide Listen für **jede** Karte prüfen.

## Schritt 4 — Erste Inbetriebnahme prüfen

Mit **einer** Karte und **einer** Station durchspielen, bevor die Anlage in
Betrieb geht:

1. **UID-Erkennung.** Karte an einem Stations-Reader scannen. Erwartung: das
   zugehörige `input_boolean` wechselt auf `on`. Passiert nichts, siehe
   [UID-Format](#uid-format-prüfen).
2. **Session-Start.** Das MC-Boolean muss automatisch mitgegangen sein, der
   Experimentname in `input_text.experiment_tagXXX` neu generiert. In den
   MQTT-Geräten von Home Assistant taucht ein neues Gerät mit diesem Namen auf,
   mit `01_Startzeitpunkt`.
3. **Messung.** Am Gerät der Station etwas verbrauchen. Der Utility Meter zählt hoch.
4. **Station-Checkout.** Karte erneut an derselben Station scannen. Erwartung:
   `input_number.tr_k_0N_kwh_um_added_tagXXX` enthält jetzt den Verbrauch,
   `05_Stromverbrauch (Reader)` erscheint am MQTT-Gerät.
5. **Session-Ende.** Karte am MC-Reader scannen. Erwartung: `02_Endzeitpunkt`,
   `11_Stromverbrauch Gesamt` und `CO2 Äquivalent Gesamt` erscheinen, das
   MC-Display meldet `<Name> abgeschlossen!`.
6. **Der kritische Fall.** Neue Session starten, an der Station einchecken,
   Verbrauch erzeugen — und dann **direkt am MC auschecken, ohne vorher an der
   Station auszuchecken**. Erwartung: der Gesamtwert ist trotzdem korrekt und
   nicht 0. Das ist der Fall, den die OFF-Reihenfolge im MC Session Controller
   absichert (früher Bug 4).

Wenn alle sechs Punkte durchlaufen, ist die Anlage betriebsbereit.

## UID-Format prüfen

Der Vergleich im Hardware-Layer ist **exakt und case-sensitiv**. Passt die
Schreibweise nicht, passiert schlicht gar nichts — keine Fehlermeldung, kein
Logeintrag.

Beobachtete Formate, abhängig von der Reader-Firmware:

| Format | Beispiel |
|---|---|
| Mit Bindestrichen, Großbuchstaben | `13-CF-91-2A` |
| Durchgehend | `04A224B91C2A80` |

**Immer so vorgehen:** Karte einmal am Reader scannen, dann in den
Entwicklerwerkzeugen unter *Zustände* den State des Reader-Sensors ansehen und
diesen Wert **wörtlich** in das UID-Feld kopieren.

## Anlage erweitern

### Karte hinzufügen

1. Helfer für die neue Karte anlegen ([Entities und Helfer](entities.md)).
2. Im **Hardware-Layer** den nächsten freien Karten-Slot mit UID und Booleans belegen.
3. In **jedem** Reader Session Handler denselben Slot belegen.
4. Im **MC Session Controller** denselben Slot belegen, inklusive beider
   Mehrfachauswahlen.

Über 10 Karten hinaus geht es nicht ohne Änderung an den Blueprints. Warum, und
was dafür nötig wäre, steht in
[Entscheidungen](entscheidungen.md#warum-genau-10-karten-slots).

### Station hinzufügen

1. Helfer für die neue Station anlegen — pro Karte je ein `input_boolean`, ein
   `input_number` und ein Utility Meter, plus ein Display-`input_text`.
2. Die `final_*`-Template-Sensoren **jeder Karte** um die neue Station erweitern.
3. Im **Hardware-Layer** den nächsten freien Reader-Slot (`reader_1` … `reader_9`)
   mit dem Sensor belegen und in jeder Karten-Zeile das passende Boolean eintragen.
4. Eine **neue Automation** aus dem Reader Session Handler anlegen, mit eigener
   eindeutiger `reader_id`.
5. Im **MC Session Controller** bei **jeder** Karte die neue Station in
   `tagN_reader_booleans` und `tagN_reader_numbers` ergänzen.

Schritt 2 und 5 werden am ehesten vergessen und sind genau die, die still falsche
Ergebnisse produzieren.

Der Hardware-Layer deckt MC + 9 Stationen in einer Automation ab. Bei einem
zehnten Stations-Reader muss das Blueprint um einen Slot erweitert werden: ein
Sensor-Input, ein Trigger und eine Spalte in der Karten-Matrix.

## Nach Änderungen am Blueprint

Wird eine der YAML-Dateien im Repo geändert, übernehmen bestehende Automationen
die Änderung **nicht** automatisch. In Home Assistant muss das Blueprint erneut
importiert werden (gleiche URL, Überschreiben bestätigen). Danach die betroffenen
Automationen einmal öffnen und speichern, damit neue Inputs mit ihren Defaults
gesetzt werden.

Ändert sich der **Name eines Inputs**, verlieren bestehende Automationen den
zugehörigen Wert kommentarlos. Deshalb: Input-Schlüssel möglichst stabil halten.
