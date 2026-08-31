# Blueprint-Referenz

Jedes Eingabefeld der drei Blueprints, seine Bedeutung und was passiert, wenn es
leer bleibt.

Der Aufbau ist in allen drei Dateien gleich: ein Abschnitt für allgemeine
Einstellungen, danach zehn identisch aufgebaute Karten-Abschnitte. Beschrieben ist
jeweils **Karte 001**; die Slots 002 bis 010 sind Wort für Wort identisch, nur mit
anderer Nummer im Schlüssel (`tag1_boolean` → `tag2_boolean` usw.) und
durchgehend `default: []`.

| Blueprint | Inputs gesamt | Abschnitte | Modus | Trigger |
|---|---|---|---|---|
| Hardware-Layer | 120 | 1 + 10 | `queued`, max 25 | 10 |
| Reader Session Handler | 77 | 1 + 10 | `parallel`, max 25 | 10 |
| MC Session Controller | 94 | 1 + 10 | `parallel`, max 25 | 20 |

`max: 25` (statt des Defaults 10) deckt 10 Karten an mehreren Readern gleichzeitig
ab.

---

## 1. Hardware-Layer

Datei: [tagreader_inpput_boolean_1_1.yaml](../tagreader_inpput_boolean_1_1.yaml)
· Instanzen: **1 pro Anlage**

Besonderheit: **kein einziges Feld ist technisch Pflicht.** Alle haben einen
Default. Praktisch braucht es mindestens den MC-Reader-Sensor, eine UID und ein
Boolean, sonst tut die Automation nichts.

### Abschnitt „Reader-Sensoren"

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `reader_mc` | `sensor` | leer | Sensor des MC-Readers. Sein State muss beim Scannen die UID sein. Trigger-ID `mc`. |
| `reader_1` … `reader_9` | `sensor` | leer | Sensoren der Stations-Reader. Trigger-IDs `r1` … `r9`. Nicht benutzte Slots leer lassen. |

### Abschnitt „Karte 001" (× 10)

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `tag_1_uid` | Text | leer | **Exakte** UID, genau wie der Reader-Sensor sie liefert. Case-sensitiv, keine Leerzeichen. Beispiele: `04A224B91C2A80`, `13-CF-91-2A`. |
| `mc_tag1_boolean` | `input_boolean` | leer | Wird getoggelt, wenn diese Karte am **MC**-Reader gescannt wird. |
| `r1_tag1_boolean` … `r9_tag1_boolean` | `input_boolean` | leer | Wird getoggelt, wenn diese Karte am jeweiligen **Stations**-Reader gescannt wird. |

Ein leeres Boolean-Feld bedeutet „diese Karte wird an diesem Reader nicht
verwendet" — ein Scan dort bleibt dann folgenlos.

Die Zuordnung ist eine Matrix aus 10 Karten × 10 Readern, also 100 Boolean-Felder
plus 10 Sensoren plus 10 UIDs = 120 Inputs. Die Aktionslogik bleibt trotzdem kurz,
weil die Trigger-ID zugleich der Schlüssel im Karten-Dict ist.

---

## 2. Reader Session Handler

Datei: [tagreader_reader_session_handler.yaml](../tagreader_reader_session_handler.yaml)
· Instanzen: **1 pro Stations-Reader**

### Abschnitt „Allgemein / primärer Sensor"

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `display_text` | `input_text` | **Pflicht** | Display dieses Readers. Wird nach jedem Ein-/Auschecken neu beschrieben. |
| `esphome_in` | Aktion | leer | Wird beim Einchecken ausgeführt. Leer lassen, wenn der Reader nichts schalten soll. |
| `esphome_out` | Aktion | leer | Wird beim Auschecken ausgeführt. |
| `reader_id` | Text | `tr_k_01_kwh` | Suffix im MQTT-Topic des primären Sensors. **Muss je Station eindeutig sein** — zwei Stationen mit gleicher ID überschreiben sich gegenseitig. |
| `sensor_name` | Text | `05_Stromverbrauch (Reader)` | Anzeigename in Home Assistant. Das Zahlenpräfix steuert die Sortierung. |
| `unit_of_measurement` | Text | `kWh` | Einheit des primären Sensors. |
| `device_class` | Text | `energy` | Geräteklasse. Leerer String lässt das Feld im Discovery-Payload weg. |

### Abschnitt „Karte 001" (× 10)

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `tag1_boolean` | `input_boolean` | **Pflicht** | Der Trigger: das Boolean **dieser Station** für diese Karte. |
| `mc_tag1_boolean` | `input_boolean` | **Pflicht** | Das globale MC-Boolean. Wird beim Einchecken eingeschaltet, falls es noch `off` ist. |
| `tag1_number` | `input_number` | **Pflicht** | kWh-Speicher dieser Station. Beim Auschecken wird das Meter-Delta aufaddiert. |
| `tag1_meter` | `sensor` | **Pflicht** | Utility Meter dieser Station. Beim Einchecken auf 0 kalibriert. |
| `tag1_text` | `input_text` | **Pflicht** | Experimentname. Bestimmt, an welches MQTT-Gerät die Messwerte gehen. |
| `tag1_manufacturer` | Text | `Karte 001` | Erscheint als „Hersteller" am MQTT-Gerät. Repräsentiert die physische Karte. |
| `tag1_sensors` | Objekt (Liste) | leer | Zusätzliche Messgrößen, siehe unten. |

> Pflicht heißt hier: **für Karte 001**. In den Slots 002 bis 010 sind dieselben
> Felder optional. Wer einen Slot nutzt, sollte ihn allerdings vollständig
> ausfüllen — ein halb belegter Slot führt zu Templates, die auf `[]` statt einer
> Entity-ID zugreifen.

### Zusatzsensoren

`tagN_sensors` ist ein freies Listenfeld ohne festes Limit. Jeder Eintrag:

| Schlüssel | Pflicht | Bedeutung |
|---|---|---|
| `value_entity` | ja | Die Entity, deren Wert publiziert wird. Bei Meter-Betrieb muss es ein `input_number` sein. |
| `name` | ja | Anzeigename in Home Assistant. Zahlenpräfix steuert die Sortierung. |
| `unit` | ja | Einheit. |
| `topic_id` | ja | Suffix im MQTT-Topic. Muss innerhalb eines Experiments eindeutig sein. |
| `meter_entity` | nein | Wenn gesetzt: Meter-Betrieb (kalibrieren beim Einchecken, aufaddieren beim Auschecken). |
| `device_class` | nein | Geräteklasse; wird weggelassen, wenn nicht gesetzt. |

Einträge ohne `topic_id` werden stillschweigend verworfen (`selectattr('topic_id', 'defined')`).

**Zwei Betriebsarten:**

```yaml
# Passthrough: aktueller Wert wird beim Auschecken 1:1 übernommen
- value_entity: sensor.argon_verbrauch_tag001
  name: "07_Argon Verbrauch"
  unit: "ml"
  topic_id: "argon"

# Meter-basiert: verhält sich wie der primäre kWh-Sensor
- value_entity: input_number.stickstoff_summe_tag001
  meter_entity: sensor.stickstoff_meter_tag001
  name: "08_Stickstoff Verbrauch"
  unit: "ml"
  topic_id: "stickstoff"
```

> Beim Auschecken läuft pro Zusatzsensor ein `delay: 1s`. Bei vielen
> Zusatzsensoren summiert sich das spürbar — siehe
> [Betrieb und Fehlersuche](betrieb.md#checkout-dauert-lange).

---

## 3. MC Session Controller

Datei: [tagreader_mc_session_controller.yaml](../tagreader_mc_session_controller.yaml)
· Instanzen: **1 global**

### Abschnitt „Allgemein"

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `station_id` | Text | `001` | Suffix der Gesamtwert-Topics: `kwh_<id>`, `l_<id>`, `co2_<id>`. |
| `display_text` | `input_text` | **Pflicht** | MC-Display. Bekommt beim Checkout `<Experimentname> abgeschlossen!`. |
| `esphome_in_action` | Aktion | leer | Beim Session-Start. |
| `esphome_out_action` | Aktion | leer | Beim Checkout. Getrennt von IN — früher rief eine Automation dieselbe Aktion für beides auf (Bug 3). |

### Abschnitt „Karte 001" (× 10)

| Input | Typ | Default | Bedeutung |
|---|---|---|---|
| `mc_tag1_boolean` | `input_boolean` | **Pflicht** | Der Trigger. `off`→`on` startet die Session, `on`→`off` beendet sie. |
| `tag1_reader_booleans` | `input_boolean`, mehrfach | leer | **Alle** Stations-Booleans dieser Karte. Werden beim Checkout ausgeschaltet. |
| `tag1_reader_numbers` | `input_number`, mehrfach | leer | **Alle** kWh-Speicher dieser Karte. Werden beim Session-Start auf 0 gesetzt. |
| `tag1_text_storage` | `input_text` | **Pflicht** | Wird mit dem neu generierten Experimentnamen beschrieben. |
| `base_tag1` | `input_text` | **Pflicht** | Basis-Name aus dem Dashboard, geht in den generierten Namen ein. |
| `tag1_kwh_sensor` | `sensor` | **Pflicht** | Finaler kWh-Sensor, Summe über alle Stationen. |
| `tag1_co2_sensor` | `sensor` | **Pflicht** | Finaler CO₂-Sensor. |
| `tag1_l_sensor` | `sensor` | leer | Finaler Wasser-Sensor. **Optional** — bleibt das Feld leer, wird gar kein Wasser-Sensor publiziert. |
| `tag1_flush_script` | Aktion | leer | Publiziert den alten kWh-Wert vor dem Reset. Optional. |

### Die beiden Mehrfachauswahlen

`tagN_reader_booleans` und `tagN_reader_numbers` sind der Grund, warum dieses
Blueprint mit **beliebig vielen** Stationen umgehen kann, ohne es anzufassen.
Sie sind zugleich die einzige Stelle im ganzen System, an der eine unvollständige
Konfiguration still falsche Messwerte erzeugt:

- Fehlt ein Reader in `tagN_reader_booleans`, bleibt sein Boolean nach Session-Ende
  auf `on` hängen, und sein letzter Messwert fehlt in der Session.
- Fehlt einer in `tagN_reader_numbers`, läuft sein alter kWh-Wert in die nächste
  Session weiter.

Und: die `final_*`-Template-Sensoren liegen außerhalb der Blueprints und müssen
**ebenfalls** über alle Stationen summieren.

### Der Experimentname

```
{{ now().strftime('%Y%m%d') }}_{{ states(base) }}_{{ 4 Zufallszeichen }}
```

Zeichenvorrat `A-Z0-9`. Beispiel: `20260831_Trocknung_A7K2`.

Der Name ist die MQTT-Geräte-Identität der Session. Er wird bei jedem
Session-Start neu erzeugt, wodurch jede Session ein eigenes Gerät in Home
Assistant wird.

### Warum das Flush-Script pro Slot ausgeschrieben ist

Der ON-Zweig enthält als einzige Stelle im ganzen Repo ein `choose` mit einem
Zweig je Karten-Slot. Grund: `action`-Selektoren (eingebettete Aktionssequenzen)
sind **nicht template-fähig** — sie lassen sich nicht über `this_tag` auswählen wie
eine Entity-ID. Ausführlicher in
[Entscheidungen](entscheidungen.md#warum-das-flush-script-pro-slot-ausgeschrieben-ist).
