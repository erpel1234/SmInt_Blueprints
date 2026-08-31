# Betrieb und Fehlersuche

Symptome, Ursachen und bekannte Einschränkungen im laufenden Betrieb.

## Werkzeugkasten

Bevor es ans Raten geht — die drei Stellen, an denen praktisch jede Frage
beantwortet wird:

1. **Entwicklerwerkzeuge → Zustände.** Zeigt den echten State jedes Readers,
   Booleans, Meters und Template-Sensors. Der erste Blick bei „das System reagiert
   nicht".
2. **Automation → Traces.** Jede Automation speichert die letzten Läufe mit allen
   Zwischenvariablen. Besonders `this_tag` und `sensor_defs` sind hier sichtbar —
   ist `this_tag` leer, hat der Karten-Lookup nicht gegriffen.
3. **Entwicklerwerkzeuge → MQTT → Themen abhören** auf
   `homeassistant/sensor/#`. Zeigt, was die Blueprints tatsächlich publizieren.

---

## Nichts passiert beim Scannen

**Häufigste Ursache: die UID stimmt nicht überein.** Der Vergleich im
Hardware-Layer ist exakt und case-sensitiv, und ein Fehlschlag ist geräuschlos —
keine Fehlermeldung, kein Logeintrag.

Prüfen:

1. Karte scannen, dann in *Entwicklerwerkzeuge → Zustände* den State des
   Reader-Sensors ansehen.
2. Diesen Wert **wörtlich** mit dem UID-Feld im Hardware-Layer vergleichen.
   `13-cf-91-2a` und `13-CF-91-2A` sind verschiedene Werte.
3. Bei Abweichung: den State-Wert kopieren und ins Blueprint einsetzen.

Weitere mögliche Ursachen:

- Der Reader-Sensor ist gar nicht als Trigger konfiguriert (falscher oder leerer
  `reader_N`-Slot).
- Das Boolean-Feld dieser Karte für **diesen** Reader ist leer — dann ist der Scan
  per Definition folgenlos.
- Der Reader-Sensor ändert seinen State nicht, weil zweimal dieselbe UID
  hintereinander gescannt wurde. Ein `state`-Trigger feuert nicht bei einem
  gleichbleibenden Wert. Die Firmware muss den Sensor zwischendurch zurücksetzen.

## Karte scheint verkehrt herum eingeloggt

Der Hardware-Layer schaltet per `input_boolean.toggle`. Damit ist **der Boolean die
Quelle der Wahrheit, nicht die Karte**. Ein verpasster Scan (Karte nicht erkannt)
oder ein doppelter Scan (Karte zu lange am Reader) invertiert den Zustand
dauerhaft: der nächste Scan checkt aus, obwohl der Nutzer einchecken wollte.

**Behebung:** Das betroffene `input_boolean` in *Entwicklerwerkzeuge → Zustände*
oder über das Dashboard von Hand auf den richtigen Wert setzen. Beim Umschalten
laufen die Blueprints ganz normal an, es geht also kein Messwert verloren.

Das ist eine bewusste Design-Eigenschaft, kein Fehler — Toggle erlaubt es, mit
*einem* Scan-Ereignis sowohl ein- als auch auszuchecken.

## Session meldet 0 kWh

Die Reihenfolge im OFF-Zweig des MC Session Controllers deckt den klassischen Fall
ab (Checkout am MC ohne vorherigen Checkout an der Station — früher Bug 4). Bleibt
der Wert trotzdem 0, in dieser Reihenfolge prüfen:

1. **`sensor.final_kwh_tagXXX`** in den Zuständen ansehen. Steht dort schon 0, liegt
   es nicht am Blueprint, sondern am Template-Sensor oder den `input_number`.
2. **`input_number.tr_k_0N_kwh_um_added_tagXXX`** aller Stationen ansehen. Stehen
   dort Werte, aber der Template-Sensor summiert sie nicht → der Template-Sensor
   kennt die Station nicht (siehe [Unvollständige Reader-Listen](#unvollständige-reader-listen)).
3. **Utility Meter** ansehen. Zählt er überhaupt hoch? Ist seine Quelle
   (`source`) verfügbar, oder steht sie auf `unavailable`?
4. **`input_number`-Maximum.** `input_number` klemmt still am `max`-Wert ab. Ein zu
   niedriges Maximum deckelt lange Sessions ohne jede Fehlermeldung.

## Unvollständige Reader-Listen

Die einzige Stelle im System, an der eine unvollständige Konfiguration
**stillschweigend** falsche Messwerte erzeugt. Drei Listen müssen jede Station
kennen:

| Ort | Wenn eine Station fehlt |
|---|---|
| `tagN_reader_booleans` (MC Controller) | Das Boolean bleibt nach Session-Ende auf `on` hängen, sein letzter Messwert fehlt in der Session. |
| `tagN_reader_numbers` (MC Controller) | Der alte kWh-Wert dieser Station wird beim Session-Start nicht zurückgesetzt und läuft in die nächste Session weiter. |
| `sensor.final_*_tagXXX` (Template-Sensor, außerhalb der Blueprints) | Der Verbrauch dieser Station fehlt in der Gesamtsumme. |

**Alle drei müssen für jede Karte vollständig sein.** Nach dem Hinzufügen einer
Station ist das der Punkt zum Durchgehen — siehe
[Installation → Station hinzufügen](installation.md#station-hinzufügen).

Ein hängengebliebenes Boolean fällt im Betrieb dadurch auf, dass das Reader-Display
eine Karte als eingeloggt anzeigt, die längst weg ist.

## Werte laufen in die nächste Session weiter

Fast immer eine unvollständige `tagN_reader_numbers`-Liste (siehe oben). Seltener:
das Flush-Script hat den alten Wert publiziert, aber der Reset wurde übersprungen,
weil die Liste leer ist — bei komplett leerer Liste überspringt der ON-Zweig den
Reset-Schritt bewusst.

## Station-zuerst: Check-in ohne vorherigen MC-Scan

**Bekannte Einschränkung.** Der Reader Session Handler schaltet beim Einchecken
das MC-Boolean ein, falls es noch `off` ist — ein Komfortweg, der eine Session
implizit startet, ohne dass vorher am MC-Reader gescannt wurde.

Dabei laufen zwei Automationen gleichzeitig los:

- der **Reader Session Handler** kalibriert das Meter und publiziert seine
  MQTT-Discovery-Config unter `states(experiment_text)` — dem **aktuellen**,
  also noch alten Experimentnamen;
- der **MC Session Controller** erzeugt im selben Moment einen **neuen**
  Experimentnamen und schreibt ihn in dieselbe `input_text`-Entity.

Welche der beiden zuerst fertig ist, ist nicht garantiert. Im ungünstigen Fall
landet die Discovery-Config der Station beim vorherigen Experiment, während der
Messwert beim Auschecken unter dem neuen Namen publiziert wird — der Sensor
erscheint dann am neuen Experiment-Gerät nicht.

> Diese Einschränkung ist aus dem Code hergeleitet und **nicht** an echter Hardware
> reproduziert worden. Sie betrifft ausschließlich den Station-zuerst-Weg.

**Umgehung im Betrieb:** Sessions immer am MC-Reader starten. Dann ist das
MC-Boolean beim Check-in an der Station bereits `on`, der Zweig läuft gar nicht an,
und die Race Condition existiert nicht.

**Mögliche Lösung im Code:** Im ON-Zweig des Reader Session Handlers nach dem
`turn_on` mit einem `wait_template` warten, bis sich der Experimentname geändert
hat, bevor die Discovery-Config publiziert wird.

## Checkout dauert lange

Das ist Absicht, nicht Trägheit: die Blueprints setzen `delay: 1s` zwischen
MQTT-Publikationen, damit Discovery-Config und State nicht in derselben
Millisekunde eintreffen und Home Assistant die Config sicher zuerst verarbeitet.

Grob:

| Vorgang | Wartezeit |
|---|---|
| Session-Start (MC ON) | ca. 2 s |
| Station-Checkout | 1 s + 1 s je Zusatzsensor |
| Session-Ende (MC OFF) | ca. 6 s, mit Wasser-Sensor ca. 8 s |

Bei vielen Zusatzsensoren summiert sich das spürbar. Wenn das stört, ließe sich
das Publish-Muster bündeln (Discovery-Configs gesammelt vorab, States am Stück) —
bisher nicht umgesetzt.

## Experiment-Gerät erscheint nicht in Home Assistant

1. **MQTT Discovery aktiv?** Das Discovery-Präfix muss `homeassistant` sein, sonst
   passen die Topics nicht.
2. **Wurde die Config publiziert?** Auf `homeassistant/sensor/<experimentname>/#`
   lauschen und einen Scan auslösen.
3. **Experimentname leer oder `unknown`?** Dann geht das Topic ins Leere. Prüfen,
   ob `input_text.experiment_tagXXX` einen Wert hat und `input_text.input_experiment_tagXXX`
   befüllt ist.
4. **Name mittendrin geändert?** Wird `experiment_tagXXX` von Hand geändert,
   während eine Session läuft, trennt das die Session von ihren bereits
   publizierten Messwerten.

## Alte Experimente sammeln sich an

Alle MQTT-Publikationen laufen mit `retain: true`. Jedes je gestartete Experiment
bleibt damit als Gerät in Home Assistant sichtbar, bis sein Topic aktiv geleert
wird.

Das ist gewollt — die Messwerte sollen einen Broker- oder HA-Neustart überleben.
Zum Aufräumen muss auf das jeweilige `.../config`-Topic eine **leere Nachricht mit
`retain: true`** publiziert werden; Home Assistant entfernt das Gerät daraufhin.

## Verhalten bei Neustarts

- `input_boolean`, `input_text` und `input_number` stellen ihren Zustand nach einem
  HA-Neustart wieder her. Eine laufende Session übersteht einen Neustart also.
- Automationen, die im Moment des Neustarts mitten in einer Sequenz stehen
  (z. B. in einem der `delay: 1s`), werden abgebrochen und **nicht** fortgesetzt.
  Ein Neustart genau während eines Checkouts kann daher eine unvollständige
  MQTT-Publikation hinterlassen.
- Utility Meter behalten ihren Stand.

## Zwei Stationen mit derselben `reader_id`

Die `reader_id` bildet das MQTT-Topic-Suffix des Pro-Station-Sensors. Zwei Reader
Session Handler mit derselben ID publizieren auf dasselbe Topic und überschreiben
sich gegenseitig. Beim Anlegen einer neuen Station immer prüfen, dass die ID
eindeutig ist (`tr_k_01_kwh`, `tr_k_02_kwh`, …). Der Default lautet
`tr_k_01_kwh` — der zweite Reader erbt ihn also, wenn man das Feld nicht anfasst.

## Traces zeigen Läufe ohne sichtbare Wirkung

Die Trigger des Reader Session Handlers sind einfache `state`-Trigger ohne
`from`/`to`. Sie feuern damit auch bei reinen Attributänderungen des Booleans. Die
`choose`-Zweige prüfen anschließend auf `'on'` bzw. `'off'`, sodass in diesem Fall
nur die Display-Aktualisierung am Ende läuft. Solche Läufe in den Traces sind
normal und harmlos.

Der MC Session Controller hat dieses Verhalten nicht — seine Trigger sind mit
`from`/`to` explizit auf die Übergänge eingeschränkt.

## Zusammenspiel mit dem Flush-Script

Das Flush-Script wird über einen `action`-Selektor eingebunden und läuft als
**eingebettete Sequenz** im ON-Zweig des MC Session Controllers.

**Deshalb: die „nur wenn > 0"-Bedingung gehört in ein eigenes Script-Entity**, das
dann per Aktion aufgerufen wird — nicht als `condition` direkt in das Aktionsfeld
des Blueprints. Eine fehlschlagende `condition` in einer eingebetteten Sequenz
beendet den gesamten Automationslauf; Reset, Namensgenerierung und Startzeit
würden dann ausfallen. Als eigenständiges Script gekapselt bleibt der Abbruch
lokal, und der ON-Zweig läuft normal weiter.

Ein Beispiel-Script steht in [Entities und Helfer](entities.md#flush-script).
