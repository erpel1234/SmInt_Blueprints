# SmInt Tagreader — Ablaufdiagramm

Aktueller Stand nach allen Bugfixes (siehe [KONTEXT.md](KONTEXT.md) für Details zu jedem Schritt).

Der Ablauf ist derselbe für jede Karte — die Blueprints suchen zu Beginn den zur auslösenden Entity gehörenden Karten-Eintrag heraus (`this_tag`) und durchlaufen den Graphen dann genau einmal. Ob es sich um Karte 001 oder 010 handelt, ändert nur die verwendeten Entities, nicht den Weg.

```mermaid
flowchart TD
    A[Karte an Reader 1/2 oder MC scannen] --> B{"UID in der Kartenliste?<br/>(Slot 001-010)"}
    B -- nein --> Z[keine Aktion]
    B -- ja --> C["Hardware-Layer:<br/>input_boolean.toggle<br/>(r1/r2/mc_tagX_boolean)"]

    C -->|"r1/r2_tagX_boolean<br/>geändert"| D{Reader-lokaler<br/>Tag-Boolean}
    C -->|"mc_tagX_boolean<br/>direkt geändert"| M{MC-Boolean}
    DashB[Dashboard: neues<br/>Experiment anlegen] -->|"setzt mc_tagX_boolean"| M

    subgraph RSH["Reader Session Handler (1x pro Reader, z.B. TR-K-01)"]
        D -- ON --> E[MC-Boolean einschalten,<br/>falls noch aus]
        E --> F[Utility Meter auf 0<br/>kalibrieren]
        F --> G[MQTT Discovery-Config<br/>publizieren]
        G --> H[ESPHome IN]

        D -- OFF --> I["Meter-Wert zu<br/>tagX_number addieren"]
        I --> J["MQTT State publizieren<br/>(1s delay)"]
        J --> K[ESPHome OUT]

        H --> L[Reader-Display<br/>neu berechnen]
        K --> L
    end

    E -.->|"löst aus"| M

    subgraph MCC["MC Session Controller (1x global, MC-K-01)"]
        M -- ON --> N["① Flush-Script:<br/>alten kWh-Wert publizieren"]
        N --> O["② kWh-Speicher aller<br/>Stationen auf 0 resetten"]
        O --> P["③ neuen Zufallsnamen<br/>generieren"]
        P --> Q["④ MQTT start_time<br/>publizieren"]
        Q --> R[⑤ ESPHome IN]

        M -- OFF --> T["① Reader-Booleans aller<br/>Stationen ausschalten"]
        T --> T2["löst Reader Session Handler<br/>OFF-Zweig aus (s.o.)"]
        T2 --> Wait["② 1s warten"]
        Wait --> S["③ MQTT end_time/kwh/co2<br/>publizieren"]
        S --> U[④ ESPHome OUT]
        U --> V["⑤ MC-Display:<br/>'... abgeschlossen!'"]
    end
```

Die nummerierten Schritte markieren Reihenfolgen, die früher zwischen mehreren separaten, parallel laufenden Automationen zufällig waren und jetzt durch die Konsolidierung in [tagreader_mc_session_controller.yaml](tagreader_mc_session_controller.yaml) garantiert sind:

- **ON-Zweig**: Flush vor Reset vor Namensgenerierung — sonst würde der alte kWh-Wert nach dem Reset als 0 "geflusht".
- **OFF-Zweig**: Reader-Booleans zuerst ausschalten (das triggert den Reader Session Handler, der den finalen kWh-Wert erst befüllt), dann erst die finalen kWh/CO2-Werte publizieren — sonst wird beim direkten MC-Checkout ohne vorherigen Reader-Checkout fälschlich 0 veröffentlicht.
