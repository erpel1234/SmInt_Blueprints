# Entscheidungen und Historie

Warum das System so aussieht, wie es aussieht — und was auf dem Weg dorthin
schiefging. Gedacht für den Fall, dass jemand eine dieser Entscheidungen
rückgängig machen will und wissen sollte, warum sie so getroffen wurde.

## Warum drei Blueprints statt einem

Die Schichten trennen drei Dinge, die sich unabhängig voneinander ändern:

- **Hardware** (welcher Reader liefert welches UID-Format) ändert sich beim
  Firmware-Wechsel,
- **Stationslogik** (was passiert an einer Messstelle) ändert sich beim
  Hinzufügen einer Station,
- **Session-Lebenszyklus** (was heißt „Experiment") ändert sich fast nie.

Die Kopplung über `input_boolean`-Helfer statt über direkte Aufrufe hat einen
praktischen Nebeneffekt: jede Schicht lässt sich einzeln testen, indem man das
entsprechende Boolean von Hand umschaltet. Kein Reader, kein MQTT, keine Karte nötig.

## Warum genau 10 Karten-Slots

Alle drei Blueprints arbeiten intern listengetrieben (siehe
[Architektur](architektur.md#warum-listengetrieben)). Die Engine hätte mit einer
unbegrenzten Kartenliste kein Problem — die zehn GUI-Slots sind nur die
Eingabemaske für diese Liste.

Eine unbegrenzte Variante **wurde gebaut, war funktionsfähig und wurde wieder
verworfen** (Entscheidung vom 6. August 2026).

Der Grund liegt in zwei Einschränkungen von Home-Assistant-Blueprints:

1. Trigger brauchen **statische** Entity-IDs. Sie können nicht aus einer Liste
   entstehen.
2. `!input`-Werte können nicht aus einem Template kommen.

Unbegrenzt viele Karten gingen deshalb nur über ein Freitext-`object`-Feld, in das
man Entity-IDs von Hand einträgt, plus eine separate Mehrfachauswahl der
Trigger-Booleans. Das kostet sämtliche Entity-Picker, Domain-Filter und
Plausibilitätsprüfungen der Oberfläche und ist beim Einrichten deutlich
fehleranfälliger — bei einer Konfiguration mit über hundert Feldern der falsche
Tausch.

**Falls doch mehr als 10 Karten gebraucht werden:** Die Engine ist bereits
listengetrieben. Ein zusätzlicher Slot ist ein Input-Block, ein Trigger und eine
Zeile in `tag_defs`, in jedem der drei Blueprints. Die verworfene Freitext-Variante
liegt in der Git-Historie.

## Warum das Flush-Script pro Slot ausgeschrieben ist

Der ON-Zweig des MC Session Controllers enthält als **einzige** Stelle im Repo
noch ein `choose` mit einem Zweig je Karten-Slot:

```yaml
- choose:
    - conditions: "{{ this_tag.label == '001' }}"
      sequence:
        - sequence: !input tag1_flush_script
    # ... × 10
```

Grund: `action`-Selektoren — also eingebettete Aktionssequenzen — sind **nicht
template-fähig**. Anders als eine Entity-ID lässt sich eine Aktionssequenz nicht
über `this_tag` auswählen. Alles andere im Blueprint läuft generisch; nur hier
musste die Auswahl ausgeschrieben werden.

## Warum Trigger-ID gleich Dict-Schlüssel (Hardware-Layer)

Der Hardware-Layer hat 10 Reader-Slots (`reader_mc`, `reader_1` … `reader_9`).
Jeder braucht einen eigenen `state`-Trigger mit eigener Trigger-ID, weil Trigger
nicht aus einer Liste entstehen können — deshalb feste Slots statt einer
Mehrfachauswahl wie im MC Controller.

Die Zuordnung ist eine Matrix: 10 Karten × 10 Reader = 100 Boolean-Felder, plus
10 Sensoren und 10 UIDs = 120 Inputs.

Damit die Aktionslogik trotzdem kurz bleibt, ist die **Trigger-ID zugleich der
Schlüssel im Karten-Dict**: `trigger.id` ist `mc` oder `r1` … `r9`, und der
komplette Lookup ist ein einziges `hit[0].get(trigger.id, '')` — unabhängig davon,
wie viele Reader-Slots existieren.

Vor dem 6. August 2026 hatte dieses Blueprint nur 3 Reader-Slots. Bei mehr Readern
erzwang das mehrere Instanzen mit jeweils identischer UID-Liste — die einzige
Stelle im System, an der dieselbe Information mehrfach zu pflegen war. Mit der
Erweiterung auf 10 Slots ist das weggefallen.

## Warum Mehrfachauswahl im MC Controller, aber Slots im Hardware-Layer

Der MC Session Controller **triggert** nur auf MC-Booleans (10 feste Trigger), muss
aber auf **beliebig viele** Stations-Entities *wirken*. Wirken ist template-fähig,
Triggern nicht — deshalb dort zwei Mehrfachauswahl-Felder pro Karte
(`tagN_reader_booleans`, `tagN_reader_numbers`) statt fester Reader-Slots. Die
Stationszahl ist damit nach oben offen, ohne das Blueprint anzufassen.

Der Preis: beide Listen müssen von Hand vollständig gehalten werden. Das ist die
einzige Stelle, an der eine unvollständige Konfiguration still falsche Messwerte
erzeugt — siehe [Betrieb und Fehlersuche](betrieb.md#unvollständige-reader-listen).

## Warum `retain: true` überall

Die Messwerte eines Experiments sollen einen Broker- oder Home-Assistant-Neustart
überleben. Der Preis ist, dass jedes je gestartete Experiment als Gerät sichtbar
bleibt, bis sein Topic aktiv geleert wird.

## Warum die Zahlenpräfixe in den MQTT-Namen

`01_Startzeitpunkt`, `02_Endzeitpunkt`, `05_Stromverbrauch (Reader)`,
`11_Stromverbrauch Gesamt`, `12_Wasserverbrauch Gesamt` — die Präfixe steuern
ausschließlich die **Sortierung im Dashboard**.

- `11`/`12` sind der historische Stand. `05`/`06` waren eine zwischenzeitliche
  Abweichung und wurden zurückgedreht.
- `05` ist im Reader Session Handler für den Pro-Station-Wert vergeben. Die
  Gesamtwerte dürfen damit nicht kollidieren.
- `CO2 Äquivalent Gesamt` hat bewusst **kein** Präfix und sortiert dadurch hinter
  den nummerierten Einträgen.

## Bereits gefixte Bugs

Die folgenden Fehler sind behoben. Sie stehen hier, weil einige davon
Reihenfolge-Garantien erzwungen haben, die man beim Umbauen leicht wieder
zerstört.

1. **Tag002/003 Auto-Checkout.**
   In der alten `tagreader_generate_random_experiment_name.yaml` wurde
   `mc_tagX_boolean` für Tag 2 und 3 direkt nach der Namensvergabe wieder
   ausgeschaltet — sofortiges, ungewolltes Checkout. Tag 1 war korrekt, dort war
   das `turn_off` bereits auskommentiert.
   *Fix:* `turn_off` für alle Tags entfernt; die Logik liegt ohnehin inline im MC
   Session Controller.

2. **Flush-vs-Reset-Race.**
   Flush-Script und Zähler-Reset liefen als zwei separate, parallele Automationen
   ohne garantierte Reihenfolge. Bei ungünstiger Reihenfolge wurde eine 0 geflusht.
   *Fix:* beides läuft jetzt sequenziell im MC Session Controller, Flush garantiert
   vor Reset.

3. **Doppeltes ESPHome-Signal beim Checkout.**
   Die alte `tagreader_mc_device_erzeugen.yaml` rief `esphome_in_action` sowohl bei
   ON als auch bei OFF auf, zusätzlich zum separaten `esphome_out` aus
   `tagreader_ergaenzungunddasandere.yaml`.
   *Fix:* `esphome_in_action` und `esphome_out_action` sind getrennte Inputs im MC
   Session Controller.

4. **Direkter MC-Checkout ohne vorherigen Reader-Checkout.**
   Die finalen kWh/CO₂-Werte wurden gelesen, **bevor** die Reader-Booleans und
   damit der zugrundeliegende Zähler aufgeräumt waren. Ergebnis: 0 kWh, wenn man
   nicht vorher manuell an der Station ausgecheckt hatte.
   *Fix:* Reihenfolge im OFF-Zweig umgedreht — Reader-Booleans zuerst ausschalten
   (löst den Reader Session Handler aus), 1 s warten, erst dann publizieren.
   **Diese Reihenfolge darf nicht wieder vertauscht werden.**

5. **`manufacturer` war global statt pro Karte.**
   Sollte die jeweilige physische Karte repräsentieren, war aber ein einzelner
   globaler Wert für alle Tags.
   *Fix:* `tagN_manufacturer` als eigener Input je Karte.

6. **Leerer Reader-State traf einen leeren UID-Slot.**
   Im Hardware-Layer wurde die gescannte UID direkt mit den konfigurierten UIDs
   verglichen. Bei 10 Slots sind die meisten UID-Felder leer, und ein
   Reader-Sensor, der auf `unknown`, `unavailable` oder leer wechselte, matchte
   auf so einen leeren Slot.
   *Fix:* expliziter Guard, der leere und ungültige Reader-States vor dem Abgleich
   verwirft.

## Entfernte Blueprints

Diese Dateien wurden am 6. August 2026 aus dem Arbeitsverzeichnis gelöscht, weil
sie in den drei aktiven Blueprints aufgegangen sind. Sie sind nicht verloren: der
letzte Commit, der sie enthält, ist **`e3e26b8`**.

```bash
git show e3e26b8:tagreader_smint.yaml          # ansehen
git checkout e3e26b8 -- tagreader_smint.yaml   # zurückholen
```

| Entfernte Datei | Ersetzt durch |
|---|---|
| `tagreader_smint.yaml` | `tagreader_reader_session_handler.yaml` |
| `tagreader_device_creation.yaml` | `tagreader_reader_session_handler.yaml` |
| `tagreader_ergaenzungunddasandere.yaml` | `tagreader_mc_session_controller.yaml` |
| `tagreader_mc_device_erzeugen.yaml` | `tagreader_mc_session_controller.yaml` |
| `tagreader_generate_random_experiment_name.yaml` | inline im `tagreader_mc_session_controller.yaml` |
| `tagreader_reader_session_handler_v2.yaml` | in `tagreader_reader_session_handler.yaml` aufgegangen |

**Zu v2:** Der frühere Testkandidat (10 Karten + Zusatzsensor-Listen) ist heute der
reguläre Reader Session Handler. Die Input-Schlüssel sind identisch geblieben; eine
bestehende Test-Automation muss in ihrem YAML nur auf den neuen Blueprint-Pfad
zeigen (`use_blueprint: path:`), die konfigurierten Werte bleiben gültig.

Vor dem endgültigen Entfernen alter Blueprints: **die zugehörigen Automationen in
Home Assistant müssen deaktiviert bzw. gelöscht sein**, sonst laufen sie ins Leere.

## Dokumentation umstrukturiert (31. August 2026)

Im Zuge der Übergabe wurden `KONTEXT.md` und `ablaufdiagramm.md` aufgelöst und ihr
Inhalt auf `docs/` verteilt:

| Alt | Neu |
|---|---|
| `KONTEXT.md`, Abschnitt Hardware/Blueprints | [architektur.md](architektur.md) |
| `KONTEXT.md`, Abschnitt Entity-Namenskonventionen | [entities.md](entities.md) |
| `KONTEXT.md`, Abschnitt Karten-Modell/Reader-Anzahl | dieses Dokument |
| `KONTEXT.md`, Bugs und entfernte Blueprints | dieses Dokument |
| `KONTEXT.md`, offene Punkte | [uebergabe.md](uebergabe.md#offene-punkte) |
| `ablaufdiagramm.md` | [architektur.md](architektur.md#ablaufdiagramm) |

Beide Dateien sind über die Git-Historie weiterhin abrufbar. An den YAML-Dateien
wurde dabei **nichts** geändert.
