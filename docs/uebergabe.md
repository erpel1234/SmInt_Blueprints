# Übergabe

Dieses Dokument richtet sich an die Person, die das Repo übernimmt. Es beantwortet
drei Fragen: Was gehört dazu? Was gehört *nicht* dazu, wird aber gebraucht? Und was
ist noch offen?

Stand: **31. August 2026**.

## Was dieses Repo ist

Drei Home-Assistant-Blueprints plus deren Dokumentation. Kein Programm, kein Add-on,
keine Integration — nur YAML, das Home Assistant beim Anlegen einer Automation als
Vorlage benutzt.

```
SmInt_Blueprints/
├── tagreader_inpput_boolean_1_1.yaml      Schicht 1: Hardware-Layer
├── tagreader_reader_session_handler.yaml  Schicht 2: pro Station
├── tagreader_mc_session_controller.yaml   Schicht 3: Session-Lebenszyklus
├── README.md                              Einstieg
├── LICENSE                                MIT
└── docs/                                  diese Dokumentation
```

Es gibt keinen Build, keine Tests, keine CI und keine Abhängigkeiten im
Software-Sinn. Eine Änderung ist eine Änderung an einer YAML-Datei; die einzige
maschinelle Prüfung, die sich anbietet, ist ein YAML-Parse:

```bash
python -c "import yaml,sys; \
  yaml.SafeLoader.add_constructor('!input', lambda l,n: None); \
  [yaml.safe_load(open(f, encoding='utf-8')) for f in sys.argv[1:]]" tagreader_*.yaml
```

Stand heute parsen alle drei Dateien sauber (120 / 77 / 94 Inputs).

## Externe Abhängigkeiten

Das Repo ist **nicht** die vollständige Anlage. Vier Dinge liegen außerhalb und
werden gebraucht, damit das System läuft. Für jedes davon braucht der neue Owner
Zugang.

### 1. Die Home-Assistant-Instanz

Enthält alles, was die Blueprints voraussetzen, aber nicht mitbringen:

- die Helfer-Definitionen (`input_boolean`, `input_text`, `input_number`)
- die Utility Meter
- die `final_*`-Template-Sensoren inklusive **CO₂-Faktor** — dieser Faktor ist eine
  fachliche Festlegung und steht nirgends in diesem Repo
- die Flush-Scripts
- die aus den Blueprints erzeugten Automationen
- die Dashboards, über die Sessions gestartet und Experimente angesehen werden

Rezepte für all das stehen in [Entities und Helfer](entities.md) — als Vorlage,
nicht als Abbild der real laufenden Konfiguration. **Ein Backup der
HA-Konfiguration gehört zwingend mit übergeben.**

### 2. Der MQTT-Broker

Zugangsdaten, Adresse und die retained Topics unter `homeassistant/sensor/#`. Dort
liegen die Messdaten aller je gelaufenen Experimente.

### 3. Die ESPHome-Firmware der Reader

Die Reader-Konfigurationen (RFID-Leser, Display, IN-/OUT-Aktionen) liegen nicht in
diesem Repo. Relevant ist vor allem das **UID-Format**, das die Firmware in den
Sensor-State schreibt — es unterscheidet sich zwischen den Installationen
(`04A224B91C2A80` gegenüber `13-CF-91-2A`) und der Vergleich im Hardware-Layer ist
exakt.

### 4. Der Konfigurations-Generator

Für das Labor Z108/Z115 (5 Karten, 1 MC-Reader, 7 Stations-Reader ⇒ rund
28 Helfer je Karte) werden die Helfer und Automationen nicht von Hand angelegt,
sondern von einem Generator erzeugt: **`code_generator/app_v3.js`**.

**Dieser Generator liegt in einem anderen Repository.** Er ist kein optionales
Komfortwerkzeug — ohne ihn ist eine Anlage dieser Größe praktisch nicht
einzurichten.

> **Zu klären bei der Übergabe:** In welchem Repo liegt er, und bekommt der neue
> Owner Zugriff darauf?

## Übergabe-Checkliste

### Vom bisherigen Owner zu erledigen

- [ ] **GitHub-Repository übertragen** (Settings → Danger Zone → Transfer
      ownership). Ein Transfer legt eine Weiterleitung von der alten URL an —
      bestehende Blueprint-Importe in Home Assistant funktionieren dadurch
      zunächst weiter.
- [ ] **Zugriff auf das Generator-Repo** (`code_generator/app_v3.js`) klären und
      übertragen.
- [ ] **Backup der Home-Assistant-Konfiguration** übergeben (Helfer,
      Template-Sensoren mit CO₂-Faktor, Scripts, Automationen, Dashboards).
- [ ] **MQTT-Broker**: Adresse und Zugangsdaten übergeben, Zugangsdaten danach
      rotieren.
- [ ] **ESPHome-Konfigurationen** der Reader übergeben, inklusive der Info, welche
      Firmware welches UID-Format liefert.
- [ ] **Lizenz klären**: `LICENSE` trägt „Copyright (c) 2026 Leonard Hermanns".
      Bei einem Eigentümerwechsel entscheiden, ob der Copyright-Vermerk bestehen
      bleibt (üblich und rechtlich unproblematisch) oder erweitert wird.
- [ ] **Weitere Zugänge**: Home-Assistant-Benutzerkonten, VPN oder Netzwerkzugang
      zur Anlage, physischer Zugang zu Readern und Karten.

### Vom neuen Owner zu erledigen

- [ ] **Git-Remote umstellen**, falls lokal geklont:
      `git remote set-url origin <neue-url>`
- [ ] **Dokumentation lesen**, in dieser Reihenfolge: [README](../README.md) →
      [Architektur](architektur.md) → [Blueprint-Referenz](blueprint-referenz.md).
- [ ] **Testlauf durchführen** nach
      [Installation → Erste Inbetriebnahme prüfen](installation.md#schritt-4--erste-inbetriebnahme-prüfen).
      Besonders Punkt 6 — der direkte MC-Checkout ohne vorherigen Station-Checkout.
- [ ] **Reader-Listen prüfen**: für jede Karte im MC Session Controller
      `tagN_reader_booleans` und `tagN_reader_numbers` mit der tatsächlichen
      Stationsliste abgleichen, und dasselbe für die `final_*`-Template-Sensoren.
      Das ist die einzige Fehlerquelle, die still falsche Zahlen liefert.
- [ ] **Retained MQTT-Topics** ansehen (`homeassistant/sensor/#`) und entscheiden,
      ob alte Experimente aufgeräumt werden sollen.

## Offene Punkte

Nach absteigender Wichtigkeit.

### Nicht an echter Hardware getestet

Die 10-Karten-Blueprints sind **statisch verifiziert** — YAML-Parsing und
Template-Logik wurden geprüft —, aber seit dem Umbau auf 10 Karten **nicht mit
echter Hardware durchgespielt**. Vor einem Rollout: mit ein bis zwei Karten
testen, insbesondere den MC-OFF-Zweig (Reihenfolge Cleanup → kWh/CO₂).

Das ist der wichtigste offene Punkt der Übergabe.

### Race Condition beim Station-zuerst-Check-in

Checkt eine Karte an einer Station ein, ohne dass vorher am MC-Reader gescannt
wurde, laufen Namensgenerierung und Discovery-Publikation parallel. Die
MQTT-Discovery-Config kann dann beim vorherigen Experiment landen. Analytisch
hergeleitet, nicht an Hardware reproduziert. Beschreibung und Lösungsvorschlag:
[Betrieb und Fehlersuche](betrieb.md#station-zuerst-check-in-ohne-vorherigen-mc-scan).

Umgehung im Betrieb: Sessions immer am MC-Reader starten.

### `source_url` fehlt

Keines der drei Blueprints hat ein `source_url`-Feld. Home Assistant kann sie
deshalb nicht per „Blueprint aktualisieren" nachziehen; nach jeder Änderung muss
über die URL neu importiert werden.

Sinnvoll **nach** dem Repo-Transfer nachzurüsten, wenn die endgültige URL feststeht:

```yaml
blueprint:
  name: "..."
  source_url: https://github.com/<neuer-owner>/SmInt_Blueprints/blob/main/<datei>.yaml
```

### Platzhalter im MQTT-Discovery-Payload

Im Reader Session Handler stehen im Device-Block feste Platzhalterwerte:

```yaml
"model_id": "test123",
"serial_number": "12345",
"configuration_url": "https://example.com/experiment_config"
```

Sie sind rein kosmetische Gerätemetadaten in Home Assistant und ohne funktionale
Wirkung. Bewusst unverändert gelassen, um bei der Übergabe keine Code-Änderung
einzuführen. Wer sie aufräumt, kann sie ersatzlos streichen oder zu optionalen
Blueprint-Inputs machen.
Fundstelle: [tagreader_reader_session_handler.yaml:382-386](../tagreader_reader_session_handler.yaml#L382-L386).

### Zehnter Stations-Reader

Der Hardware-Layer deckt MC + 9 Stationen in einer Automation ab. Für Z108/Z115
(MC + 7) reicht das. Ein zehnter Stations-Reader erfordert eine Erweiterung des
Blueprints: ein Sensor-Input, ein Trigger und eine Spalte in der Karten-Matrix.

### Mehr als 10 Karten

Bewusst nicht umgesetzt, Begründung in
[Entscheidungen](entscheidungen.md#warum-genau-10-karten-slots). Die Engine ist
bereits listengetrieben; ein zusätzlicher Slot ist ein Input-Block, ein Trigger und
eine Zeile in `tag_defs` je Blueprint.

### Delays bündeln

Der Reader Session Handler wartet beim Auschecken `1 s` pro Zusatzsensor. Bei
vielen Zusatzsensoren summiert sich das. Ließe sich durch ein gebündeltes
Publish-Muster verkürzen — bisher nicht angefasst, weil die Delays das Zusammenspiel
von Discovery-Config und State absichern.

### Veraltete Trigger-Syntax

Die Blueprints nutzen die ältere Schreibweise `trigger:` / `platform:` statt
`triggers:` / `trigger:`. Home Assistant unterstützt beides; die alte Form ist
deprecated, aber funktionsfähig. Eine Modernisierung ist rein mechanisch, würde
aber einen Re-Import aller Blueprints erfordern und sollte deshalb mit einem
ohnehin anstehenden Umbau zusammengelegt werden.

## Bekannter Installationsstand

| Installation | Karten | Reader | Status |
|---|---|---|---|
| Ursprüngliche Anlage | 001–003 (Modell bis 010) | MC-K-01, TR-K-01, TR-K-02 | produktiv |
| Labor Z108/Z115 | 301–305 | 1× MC, 7× Station (TR-Z108-01…04, TR-Z115-01…03) | seit 6. August 2026 im Aufbau, Instanziierung über den Generator |

## Wo was steht

| Frage | Dokument |
|---|---|
| Wie funktioniert das System? | [Architektur](architektur.md) |
| Wie richte ich es ein? | [Installation](installation.md) |
| Welche Helfer brauche ich? | [Entities und Helfer](entities.md) |
| Was bedeutet dieses Eingabefeld? | [Blueprint-Referenz](blueprint-referenz.md) |
| Es funktioniert nicht — warum? | [Betrieb und Fehlersuche](betrieb.md) |
| Warum ist das so gebaut? | [Entscheidungen und Historie](entscheidungen.md) |
