# SMACC – SMA Charge Control

> Home-Assistant-Package zur nachvollziehbaren und aktiven Ladefreigabe von SMA-/BYD-Batteriesystemen.

⚠️ **Disclaimer:** Dieses Projekt wird nicht von SMA, BYD oder einem Hersteller begleitet oder supportet. Nutzung auf eigene Gefahr. Wer Modbus-Schreibzugriffe aktiviert, greift aktiv in das Ladeverhalten des Systems ein. Vor produktivem Einsatz müssen Register, Werte, Entitäten und Sicherheitsgrenzen zum eigenen Wechselrichter-/Batterie-Setup passen.

## Wichtiger Hinweis zur Git-Version

Die im Repository veröffentlichten Dateien sind bewusst **anonymisiert**.

Das betrifft insbesondere:

- `packages/smacc.yaml`
- `dashboards/smacc-dashboard.json`
- alle Beispiel-Entity-IDs
- Modbus-Hub, Slave-ID und Register
- Forecast-Entitäten
- SMA-/BYD-/Zähler-Entitäten
- optionale EV-/Wallbox-Entitäten

Die veröffentlichten Packages und Dashboards enthalten **keine echten Entitäten aus der Referenzanlage**.  
Sie müssen vor der Nutzung zwingend auf die eigene Home-Assistant-Installation angepasst werden.

SMACC ist damit **kein Plug-and-Play-Package**, sondern eine Vorlage mit fertiger Logik, die auf die eigenen Sensoren gemappt werden muss.

## Was SMACC macht

SMACC entscheidet, ob ein SMA-/BYD-Akku laden darf, gesperrt bleiben soll oder gezielt bis zu einem definierten Ziel-SOC geladen werden muss.

Dazu kombiniert SMACC:

- aktuellen Batterie-SOC
- aktuelle PV-Erzeugung
- Netzbezug und Netzeinspeisung
- aktuelle Batterie-Lade- und Entladeleistung
- PV-Forecast für heute, Resttag, aktuelle Stunde, nächste Stunde und morgen
- historischen PV- und Verbrauchsverlauf aus Home Assistant
- Ziel-SOC für Mittag, Tagesende, Nachtreserve und Balancing
- Ladezeit bis Ziel
- ein Ladefenster mit frühestem und spätestem Start
- optionalen EV-/Wallbox-Kontext
- optionales Schonladen anhand der maximalen BYD-Zellspannung
- aktive Modbus-Schreibzugriffe für CmpBMS-Freigabe und Ladeleistungsgrenzen

Das Ziel ist nicht nur Anzeige, sondern eine nachvollziehbare, steuernde Ladeentscheidung.

## Funktionsumfang des aktuellen Packages

### Grundlogik

- Aktivierbare globale Ladesteuerung
- Berechnung des Hausverbrauchs aus:
  - PV-Gesamtleistung
  - Netzbezug
  - Netzeinspeisung
  - Batterie-Entladung
  - Batterie-Ladung
- Berechnung einer geglätteten Hausverbrauchs-Prognose
- Berechnung der fehlenden Energie bis zum Ziel-SOC
- Berechnung der benötigten Ladezeit bis Ziel
- Berechnung der erwarteten Ladeleistung
- Entscheidungssensor mit lesbarem Status
- Diagnose-Script mit zentralen Werten als Home-Assistant-Benachrichtigung
- Persistente Helper ohne `initial`, damit Werte Home-Assistant-Neustarts überstehen

### Betriebsmodi

SMACC kennt aktuell zwei Betriebsmodi:

- `Dynamik`
- `Schonladen`

#### Dynamik

Der Modus `Dynamik` ist der normale Automatikbetrieb.

Er berücksichtigt:

- dynamisches Tagesziel
- Zielzeit
- Forecast
- historischen 3-Tage-Zeitslot
- Ladefenster
- Ladebeginn-Fixierung
- Ziel-Hysterese
- Nachtziel
- Morgen-Forecast-Anhebung
- Balancing

#### Schonladen

Der Modus `Schonladen` reduziert die Ladeleistung abhängig von der maximalen BYD-Zellspannung.

Dabei werden verwendet:

- Ziel-SOC für Schonladen
- maximale Ladeleistung für Schonladen
- Start-Spannung für die Spannungsbremse
- Reset-Spannung zum Aufheben der Bremse
- minimaler C-Faktor
- aktuell gespeicherte Ladegrenze
- maximale Zellspannung des BYD-Towers

Die Logik senkt die Ladegrenze stufenweise ab, wenn die Zellspannung hochläuft. Dadurch kann der obere SOC-Bereich schonender und kontrollierter geladen werden.

### Jetzt laden

Über `Jetzt laden` kann man eine manuelle Ladephase starten.

Dabei wird:

- `input_boolean.sma_akku_jetzt_laden_aktiv` aktiviert
- ein Ziel-SOC gespeichert
- die normale Automatik temporär übersteuert
- bis zum Ziel geladen
- nach Zielerreichung wieder beendet

Zusätzlich gibt es einen Stop-Button, der die manuelle Ladephase abbricht und die Ladefreigabe wieder sperrt.

### Dynamisches Ziel-SOC

Das wirksame Ziel-SOC kann aus mehreren Quellen entstehen:

- Tagesziel
- Nachtziel
- Morgen-Forecast-Anhebung
- Schonladen-Ziel
- manuelle Jetzt-laden-Phase
- aktives 100-%-Balancing

Die Logik wählt daraus das aktuell passende Ziel.

Beispiele:

- normaler Tag: Tagesziel, z. B. 80 %
- schlechte PV-Prognose für morgen: Zielanhebung, z. B. 90–95 %
- aktiviertes Nachtziel: Ziel anhand erwartetem Nachtverbrauch
- aktives Balancing: 100 %

### Nachtziel

Das Nachtziel berechnet einen sinnvollen SOC für den Abend beziehungsweise die Nacht.

Es berücksichtigt:

- letzten gemessenen Nacht-SOC-Verlust
- 7-Tage-Mittel des Nacht-SOC-Verlusts
- Fallback-Wert
- gewünschten Rest-SOC bei stabilem PV-Start
- zusätzliche Nachtreserve
- Minimum und Maximum für das Nachtziel

Dadurch kann SMACC das Tagesziel automatisch höher setzen, wenn die Nacht voraussichtlich mehr Energie benötigt.

### Morgen-Forecast-Anhebung

SMACC kann den Forecast für den nächsten Tag auswerten.

Wenn der Forecast für morgen unter definierte Schwellen fällt, kann das Ziel-SOC automatisch angehoben werden.

Typische Logik:

- Forecast morgen gut: normales Tagesziel
- Forecast morgen mittel: Zielanhebung auf einen höheren SOC
- Forecast morgen schlecht: stärkere Zielanhebung

### PV-Forecast

Das aktuelle Package rechnet im Referenz-Setup mit drei PV-Flächen beziehungsweise drei Forecast-Quellen.

Verarbeitet werden:

- aktuelle Forecast-Stunde
- nächste Forecast-Stunde
- Forecast der nächsten zwei Stunden
- Resttag-Forecast
- Forecast heute gesamt
- Forecast morgen gesamt
- nutzbarer Forecast für den Akku nach erwarteten Hausverbräuchen

Wenn ein anderes Setup nur eine oder zwei PV-Flächen hat, müssen die Templates entsprechend angepasst werden.

### Historischer 3-Tage-Zeitslot

SMACC nutzt zusätzlich zur Kurzfrist-Prognose einen SQL-basierten 30-Minuten-Zeitslot-Schnitt der letzten drei Tage.

Berechnet werden:

- PV-Mittelwert im aktuellen 30-Minuten-Zeitslot der letzten drei Tage
- Hausverbrauch-Mittelwert im aktuellen 30-Minuten-Zeitslot der letzten drei Tage
- daraus eine erwartete Ladeleistung für den aktuellen Tagesabschnitt

Das macht die Startentscheidung ruhiger als eine reine Kurzfrist-Forecast-Logik.

Wichtig:

- Diese Logik benötigt Recorder-Daten.
- Direkt nach Installation sind die Werte noch nicht belastbar.
- Nach einigen Stunden wird es besser.
- Nach etwa drei Tagen ist die Zeitslot-Logik sinnvoll nutzbar.

### Zielzeit und Ladebeginn

SMACC berechnet den Ladebeginn so, dass der Ziel-SOC rechtzeitig erreicht wird.

Berücksichtigt werden:

- Zielzeit
- dynamische Zielzeit nach PV-Ende
- Puffer vor PV-Ende
- Sicherheitsreserve
- früheste Ladephase
- späteste Ladephase
- benötigte Ladezeit
- Ladezeitreserve
- erwartete Ladeleistung
- aktueller SOC
- Ziel-SOC

Der berechnete Ladebeginn wird zusätzlich fixiert, damit die Automatik nicht durch kurzfristige Schwankungen ständig hin- und herspringt.

Die Fixierung wird nachts zurückgesetzt.

### PV-Ende und stabiler PV-Start

SMACC erkennt:

- PV-Ende am Nachmittag/Abend
- stabilen PV-Start am Morgen

Diese Werte werden genutzt für:

- dynamische Zielzeit
- Nachtverbrauchs-Berechnung
- Nacht-SOC-Verlust
- Nachtziel-Prognose

### Ziel-erreicht-Sperre

Wenn das aktuelle Ziel erreicht ist, sperrt SMACC die weitere Ladung aktiv.

Ausnahmen:

- aktives 100-%-Balancing
- bestimmte Balancing-Phasen
- Ziel 100 %, bei dem kein sofortiges Sperren gewünscht ist

Zusätzlich erkennt SMACC, wenn der Akku trotz Sperre weiter lädt, und triggert dann erneut die Modbus-Schreiblogik.

### PV-Standby

Bei sehr niedriger PV-Leistung kann SMACC in einen Standby-Zustand gehen.

Dann wird:

- keine Ladephase aktiv gehalten
- der Modbus-Refresh gestoppt
- der Status auf Standby gesetzt

### 100-%-Balancing

SMACC unterstützt manuelles und wöchentliches 100-%-Balancing.

Konfigurierbar sind:

- Balancing aktiv
- wöchentliches Balancing aktiv
- Wochentag
- Startzeit
- Mindest-Restforecast

Die Automatik kann das Balancing starten und nach Zielerreichung beziehungsweise am Folgetag wieder beenden.

### Modbus-/CmpBMS-Steuerung

SMACC kann aktiv auf den SMA-Wechselrichter schreiben.

Im Referenz-Package werden unter anderem folgende CmpBMS-bezogene Werte geschrieben:

- minimale Ladeleistung
- maximale Ladeleistung
- minimale Entladeleistung
- maximale Entladeleistung
- Grid-Setpoint
- Betriebsmodus beziehungsweise Freigabe-/Sperrcode

Beispielhafte Register aus dem Referenzstand:

| Register | Bedeutung im Package |
|---:|---|
| `40793` | minimale Ladeleistung |
| `40795` | maximale Ladeleistung |
| `40797` | minimale Entladeleistung |
| `40799` | maximale Entladeleistung |
| `40801` | Grid-Setpoint |
| `40236` | CmpBMS-Freigabe-/Sperrcode |

Im Referenzstand werden beispielhaft diese Codes genutzt:

| Code | Bedeutung im Package |
|---:|---|
| `1438` | Laden erlaubt |
| `2290` | Laden gesperrt |

⚠️ Diese Werte sind nicht blind zu übernehmen. Hub, Slave-ID, Register und Codes müssen zum eigenen SMA-/BMS-Setup passen.

### Optionaler EV-/Wallbox-Kontext

Das Package kann mit einer separaten EV-/Akku-Verriegelung kombiniert werden.

Optional eingebunden sind Rollen wie:

- EV-Lademodus
- EV-Ladeleistung
- Hausverbrauch ohne Auto
- PV-Defizit durch Auto
- Batterie-Entladeleistungsziel im EV-Kontext
- EV-Akku-Verriegelung

Wer keine EV-Logik nutzt, muss die optionalen Dashboard-Karten entfernen oder die entsprechenden Sensoren durch eigene Dummy-/Template-Sensoren ersetzen.

## Dashboard

Zum Projekt gehört ein Lovelace-Dashboard als JSON-Datei.

Das Dashboard enthält zwei Views:

- `Kompakt`
- `Ausführlich`

### View: Kompakt

Die Kompaktansicht zeigt die wichtigsten Bedien- und Statuswerte:

- Betriebsmodus
- Entscheidung
- aktuelles Ziel
- SOC
- Ziel-SOC
- wirksame Ladeleistung
- Modbus-Status
- Dynamik-Einstellungen
- Schonladen-Einstellungen
- Jetzt-laden-Aktionen

### View: Ausführlich

Die ausführliche Ansicht zeigt zusätzlich:

- Steuerung und Aktionen
- aktives Zielbild
- Ladeleistung und Modbus
- Dynamik-Details
- Schonladen-Details
- Schutz- und Freigabesensoren
- Nachtziel und Balancing
- optionalen EV-Kontext
- Prognosewerte
- PV- und Verbrauchswerte

Wichtig: Auch das Dashboard ist in der Git-Version anonymisiert beziehungsweise generisch gehalten. Alle externen Sensoren müssen auf die eigenen Entitäten angepasst werden.

## Voraussetzungen

In Home Assistant werden benötigt:

- YAML-Packages
- Recorder
- SQL-Integration
- Modbus-Integration, falls aktiv gesteuert werden soll
- Template-Integration
- Statistics-Sensoren
- Automationen
- Scripts
- Helper:
  - `input_boolean`
  - `input_button`
  - `input_select`
  - `input_datetime`
  - `input_number`
  - `input_text`
  - `timer`

Packages können zum Beispiel so eingebunden werden:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

## Benötigte Entitätsrollen

Die konkreten Entity-IDs sind installationsabhängig. Entscheidend ist die Rolle der Entität.

### Pflichtrollen für die Kernlogik

| Rolle | Einheit | Beschreibung |
|---|---:|---|
| Batterie-SOC | `%` | aktueller Ladezustand der Batterie |
| Batterie-Ladeleistung | `W` | aktuelle Ladeleistung der Batterie |
| Batterie-Entladeleistung | `W` | aktuelle Entladeleistung der Batterie |
| PV-Gesamtleistung | `W` | aktuelle gesamte PV-Erzeugung |
| Netzbezug | `W` | aktuelle Leistung aus dem Netz |
| Netzeinspeisung | `W` | aktuelle Leistung ins Netz |
| BYD maximale Zellspannung | `mV` | höchste Zellspannung des BYD-Towers für Schonladen |

### Pflichtrollen für Forecast

| Rolle | Einheit | Beschreibung |
|---|---:|---|
| Forecast aktuelle Stunde | `kWh` | erwartete PV-Energie in der laufenden Stunde |
| Forecast nächste Stunde | `kWh` | erwartete PV-Energie in der nächsten Stunde |
| Forecast Resttag | `kWh` | verbleibende PV-Energie für heute |
| Forecast heute gesamt | `kWh` | erwartete PV-Energie für den gesamten Tag |
| Forecast morgen gesamt | `kWh` | erwartete PV-Energie für morgen |

Das Beispiel geht von drei Forecast-Quellen aus. Bei weniger oder mehr PV-Flächen müssen die Summen-Templates angepasst werden.

### Optionale Rollen für EV-/Wallbox

| Rolle | Einheit | Beschreibung |
|---|---:|---|
| EV-Lademodus | Text/Zahl | aktueller Wallbox-/EV-Modus |
| EV-Ladeleistung | `W` | aktuelle Ladeleistung des Autos |
| Hausverbrauch ohne Auto | `W` | bereinigter Hausverbrauch |
| EV-PV-Defizit | `W` | PV-Defizit durch Autoladung |
| Akku-Entladeleistungsziel | `W` | Zielwert für Batterieentladung im EV-Kontext |
| EV-Akku-Verriegelung | boolean | ob Entladung wegen EV-Kontext begrenzt werden soll |

## Platzhalter-Mapping der Git-Version

Die Git-Version sollte keine echten Seriennummern, Gerätenamen oder privaten Entity-IDs enthalten.

Empfohlene Platzhalter:

| Platzhalter | Rolle |
|---|---|
| `sensor.smacc_battery_soc` | Batterie-SOC |
| `sensor.smacc_battery_charge_power` | Batterie-Ladeleistung |
| `sensor.smacc_battery_discharge_power` | Batterie-Entladeleistung |
| `sensor.smacc_grid_import_power` | Netzbezug |
| `sensor.smacc_grid_export_power` | Netzeinspeisung |
| `sensor.smacc_pv_power_total` | PV-Gesamtleistung |
| `sensor.smacc_byd_max_cell_voltage_mv` | BYD maximale Zellspannung in mV |
| `sensor.smacc_pv1_forecast_current_hour` | PV-Fläche 1, Forecast aktuelle Stunde |
| `sensor.smacc_pv1_forecast_next_hour` | PV-Fläche 1, Forecast nächste Stunde |
| `sensor.smacc_pv1_forecast_today_remaining` | PV-Fläche 1, Forecast Resttag |
| `sensor.smacc_pv1_forecast_today` | PV-Fläche 1, Forecast heute |
| `sensor.smacc_pv1_forecast_tomorrow` | PV-Fläche 1, Forecast morgen |
| `sensor.smacc_pv2_forecast_*` | Forecast-Werte der PV-Fläche 2 |
| `sensor.smacc_pv3_forecast_*` | Forecast-Werte der PV-Fläche 3 |

Optionale EV-Platzhalter:

| Platzhalter | Rolle |
|---|---|
| `sensor.smacc_ev_charge_power` | EV-Ladeleistung |
| `sensor.smacc_ev_mode` | EV-Lademodus |
| `sensor.smacc_ev_mode_text` | EV-Lademodus als Text |
| `sensor.smacc_ev_house_consumption_without_car` | Hausverbrauch ohne Auto |
| `sensor.smacc_ev_pv_deficit` | PV-Defizit durch Auto |
| `sensor.smacc_ev_battery_discharge_target` | Akku-Entladeleistungsziel |
| `binary_sensor.smacc_ev_battery_lock_should_be_active` | EV-Akku-Verriegelung Soll |
| `input_boolean.smacc_ev_battery_lock_active` | EV-Akku-Verriegelung aktiv |

## Anpassung vor der Nutzung

Vor der Nutzung müssen mindestens diese Punkte geprüft und angepasst werden:

- Package-Datei in das eigene Home-Assistant-`packages`-Verzeichnis kopieren
- alle Platzhalter-Entitäten im Package auf eigene Sensoren mappen
- alle Platzhalter-Entitäten im Dashboard auf eigene Sensoren mappen
- Anzahl der PV-Forecast-Flächen prüfen
- Forecast-Sensoren auf eigene Forecast-Integration anpassen
- BYD-Zellspannungssensor für Schonladen anpassen oder Schonladen entfernen/deaktivieren
- optionale EV-/Wallbox-Entitäten anpassen oder entfernen
- Modbus-Hub anpassen
- Modbus-Slave-ID anpassen
- Modbus-Register prüfen
- Modbus-Freigabe-/Sperrcodes prüfen
- maximale Ladeleistung passend zum System setzen
- Batteriekapazität passend zum eigenen Akku setzen
- Ladefenster und Zielzeiten einstellen
- Recorder- und SQL-Funktion prüfen
- Dashboard importieren und fehlende Entitäten bereinigen

## Installation

1. Repository herunterladen oder Dateien kopieren.
2. Package nach Home Assistant kopieren, z. B.:
   - `config/packages/smacc.yaml`
3. Falls noch nicht vorhanden, Packages in `configuration.yaml` aktivieren:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
4. Platzhalter im Package ersetzen.
5. Modbus-Konfiguration prüfen.
6. Forecast-Quellen prüfen.
7. Home Assistant Konfiguration prüfen.
8. Home Assistant neu starten.
9. Dashboard JSON importieren.
10. Dashboard-Entitäten prüfen und anpassen.
11. Erst ohne aktive Modbus-Steuerung plausibilisieren.
12. Danach Modbus-Schreibzugriffe bewusst aktivieren.

## Prüfung nach der Installation

Nach dem Neustart sollten diese Werte plausibel sein:

- `sensor.sma_akku_hausverbrauch_aktuell`
- `sensor.sma_akku_hausverbrauch_prognoseleistung`
- `sensor.sma_akku_pv_forecast_resttag_gesamt`
- `sensor.sma_akku_pv_forecast_morgen_gesamt`
- `sensor.sma_akku_dynamisches_ziel_soc`
- `sensor.sma_akku_modus_basis_ziel_soc`
- `sensor.sma_akku_aktuelles_ladeziel_soc`
- `sensor.sma_akku_benoetigte_ladezeit_bis_ziel`
- `sensor.sma_akku_ladebeginn_prognose_text`
- `sensor.sma_akku_entscheidung_text`
- `binary_sensor.sma_akku_laden_soll_erlaubt_sein`
- `input_text.sma_akku_ladestatus`

Wenn diese Sensoren `unknown`, `unavailable` oder offensichtlich falsche Werte liefern, sind fast immer Entity-Mapping, Recorder/SQL oder Forecast-Quellen die Ursache.

## Betriebslogik in Kurzform

SMACC arbeitet grob nach dieser Reihenfolge:

1. Prüfen, ob die Ladesteuerung aktiv ist.
2. Aktuelle PV-, Netz-, Akku- und Verbrauchsdaten berechnen.
3. Forecast und historische Werte auswerten.
4. Wirksames Ziel-SOC bestimmen.
5. Fehlende Energie und Ladezeit berechnen.
6. Zielzeit und Ladebeginn bestimmen.
7. Ladebeginn stabilisieren/fixieren.
8. Prüfen, ob Laden erlaubt sein soll.
9. Bei Freigabe Modbus-Payload schreiben.
10. Bei Sperre Ladeleistung auf 0 setzen und Sperrcode schreiben.
11. Status und Diagnosewerte aktualisieren.

## Sicherheitshinweise

Vor produktiver Nutzung sollte man zwingend prüfen:

- Stimmen alle Sensorwerte und Einheiten?
- Ist Batterie-SOC wirklich in Prozent?
- Sind Leistungswerte wirklich in Watt?
- Sind Forecast-Werte wirklich in kWh?
- Ist die BYD-Zellspannung wirklich in mV?
- Stimmen Modbus-Hub und Slave-ID?
- Stimmen Registeradressen und Codes?
- Ist die maximale Ladeleistung passend zum Wechselrichter und Akku?
- Wird bei Zielerreichung sauber gesperrt?
- Wird bei Standby nicht unnötig geschrieben?
- Verhält sich die Anlage nach Home-Assistant-Neustart korrekt?
- Ist klar, was bei fehlenden Forecast-/SQL-Werten passiert?

## Bekannte Grenzen

- Die Logik ist auf das Referenzsetup mit SMA-/BYD-Batterie ausgelegt.
- Andere Wechselrichter oder Batterie-Setups benötigen Anpassungen.
- Die Forecast-Struktur ist beispielhaft auf drei PV-Flächen ausgelegt.
- SQL-Abfragen greifen auf Home-Assistant-Recorder-Strukturen zu.
- Ohne ausreichende Historie ist die 3-Tage-Zeitslot-Logik nur eingeschränkt brauchbar.
- EV-/Wallbox-Funktionen sind optional und hängen von separaten Entitäten ab.
- Modbus-Schreibzugriffe sind hardware- und firmwareabhängig.
- Falsche Register oder Werte können unerwünschtes Ladeverhalten auslösen.

## Empfehlung für die Veröffentlichung im Repository

Für Git sollten die Dateien so abgelegt werden:

```text
packages/
  smacc.yaml

dashboards/
  smacc-dashboard.json

README.md
```

Zusätzlich sinnvoll:

```text
examples/
  entity-mapping.example.yaml

docs/
  screenshots/
```

Die Beispiel-Dateien sollten nur Platzhalter enthalten und keine echten privaten Entity-IDs, Seriennummern, IP-Adressen oder Gerätenamen.

## Ziel des Projekts

SMACC soll aus einzelnen Energie-, Forecast- und Batterie-Sensoren eine verständliche, robuste und aktiv steuernde Ladelogik machen.

Der Mehrwert liegt darin, dass Home Assistant nicht nur Energieflüsse anzeigt, sondern eine begründete Ladeentscheidung trifft:

- Warum lädt der Akku jetzt?
- Warum bleibt er gesperrt?
- Welches Ziel gilt gerade?
- Reicht die Ladezeit noch?
- Wann muss spätestens gestartet werden?
- Warum wurde die Ladeleistung begrenzt?
- Was wurde per Modbus geschrieben?

Genau diese Nachvollziehbarkeit ist der Kern von SMACC.
