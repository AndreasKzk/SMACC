# SMACC – SMA Charge Control

SMACC ist ein Home-Assistant-Package zur **nachvollziehbaren Ladefreigabe von SMA-Batteriesystemen**.

Es bewertet PV-Erzeugung, Hausverbrauch, Batteriestand, Ziel-SOC und Zeitfenster und entscheidet daraus, ob der Akku **jetzt laden**, **später laden** oder **gesperrt bleiben** soll. Die Entscheidung wird nicht nur angezeigt, sondern kann über Modbus auch aktiv an den Wechselrichter bzw. die Batterie-Steuerung geschrieben werden.

## Warum SMACC?

Viele Setups zeigen zwar Energieflüsse und Forecasts an, haben aber keine saubere Lade-Logik darüber. Typische Probleme sind:

- der Akku lädt zu früh, obwohl später genug PV zu erwarten ist
- der Akku startet zu spät und verfehlt das Tagesziel
- reine Kurzfrist-Forecasts reagieren zu nervös
- es ist schwer zu erkennen, **warum** gerade geladen oder gesperrt wird

SMACC löst das, indem es Live-Daten, Forecasts und historische Mittelwerte zusammenführt und daraus lesbare Entscheidungs-Sensoren und eine steuernde Automatik baut.

## Was SMACC macht

SMACC berechnet zunächst den aktuellen energetischen Zustand des Hauses aus PV-Leistung, Netzbezug, Einspeisung und Batterie-Leistung. Darauf aufbauend ermittelt es:

- ein wirksames Ladeziel
- eine Zielzeit
- die fehlende Energie bis zum Ziel
- die erwartete Ladeleistung
- den sinnvollen Ladebeginn

Der aktuelle Stand nutzt für den **Ladebeginn des Tagesziels** nicht mehr primär nur die nächsten zwei Forecast-Stunden, sondern zusätzlich einen **30-Minuten-Zeitslot-Schnitt der letzten 3 Tage**. Dadurch wird die Startentscheidung meist ruhiger und plausibler.

Zusätzlich erzeugt SMACC Diagnose- und Statussensoren wie z. B.:

- aktuelle Entscheidung
- aktuelles Ladeziel
- Ladebeginn aktuelles Ziel
- Ladebeginn Tagesziel
- benötigte Ladezeit bis Ziel
- Ziel erreicht / Laden gesperrt

## Voraussetzungen

Für SMACC werden in Home Assistant mindestens folgende Bausteine benötigt:

- YAML-Packages
- Recorder
- SQL-Integration
- Modbus-Integration
- Template-, Script-, Automation-, Timer- und Helper-Funktionen

Packages können z. B. so eingebunden werden:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

## Benötigte Integrationen

Die ursprüngliche Referenz-Umgebung ließ sich auf diese Integrationen zurückführen. In der **anonymisierten Share-Version** sind die konkreten Entity-IDs daraus aber bewusst durch Platzhalter ersetzt:

- **`forecast_solar`** für die Forecast-Werte der PV-Flächen im Referenz-Setup
- **`sma`** für Batterie-SOC sowie Lade-/Entladeleistung im Referenz-Setup
- **`pysmaplus`** für Netzbezug und Netzeinspeisung im Referenz-Setup
- **`template`** für zusammengeführte Summensensoren im Referenz-Setup
- **`recorder`** als Datenbasis für historische Werte
- **`sql`** für den 3-Tage-Zeitschnitt
- **`modbus`** für die eigentliche Ladefreigabe / Sperre

Für andere Nutzer ist nicht entscheidend, ob exakt dieselben Integrationen verwendet werden. Entscheidend ist, dass die benötigten **Entitätsrollen** vorhanden sind und auf die Platzhalter im Package gemappt werden.

### Recorder

Recorder ist Pflicht, weil SMACC historische Statistikdaten nutzt. Die 3-Tage-Logik greift auf die Kurzzeitstatistik von Home Assistant zu. Ohne Recorder fehlen diese Grundlagen.

### SQL

Die SQL-Integration wird benötigt, um aus den Recorder-Daten Zeitfenster-Mittelwerte zu bilden, z. B. den PV-Mittelwert oder Hausverbrauch im gleichen 30-Minuten-Slot der letzten drei Tage.

### Modbus

Modbus ist erforderlich, wenn SMACC nicht nur anzeigen, sondern tatsächlich steuern soll. Das Package schreibt Register für Lade- und Entladefreigaben sowie Betriebsmodi.

### Forecast.Solar

Im Referenz-Setup stammen die Forecast-Werte aus **Forecast.Solar** (`forecast_solar`). Die Share-Version nutzt dafür bewusst anonyme Platzhalter wie `sensor.smacc_pv1_forecast_current_hour`.

Wenn jemand statt Forecast.Solar eine andere Forecast-Integration nutzt, müssen diese Platzhalter einfach auf die passenden eigenen Forecast-Entitäten gemappt werden.

### SMA Integration

Im Referenz-Setup stammen Batterie-SOC sowie Lade-/Entladeleistung aus der **Home-Assistant-SMA-Integration** (`sma`). In der Share-Version wurden seriennummernhaltige Entity-IDs bewusst entfernt und durch generische Platzhalter ersetzt.

### PySMAPlus

Im Referenz-Setup stammen Netzbezug und Netzeinspeisung aus **PySMAPlus** (`pysmaplus`). Auch diese Entitäten sind in der Share-Version anonymisiert.

### Template

Die gesamte PV-Leistung kann z. B. aus einer **Template-Entität** (`template`) stammen. In der Share-Version wird dafür der generische Platzhalter `sensor.smacc_pv_power_total` verwendet.

## Benötigte Entitätsrollen

Die **konkreten Entity-IDs sind installationsabhängig**. Für andere Nutzer müssen diese Werte auf die jeweils vorhandenen Sensoren gemappt werden. Wichtig ist also nicht der Name, sondern die Bedeutung der Entität.

### Pflichtrollen für die Kernlogik

Folgende Informationen müssen in Home Assistant vorhanden sein. In Klammern steht jeweils, **woher sie im aktuellen Package konkret kommen**:

| Rolle | Einheit | Beschreibung |
|---|---:|---|
| Batterie-SOC | % | aktueller Ladezustand der Batterie (`sma`) |
| Batterie-Ladeleistung | W | aktuelle Ladeleistung (`sma`) |
| Batterie-Entladeleistung | W | aktuelle Entladeleistung (`sma`) |
| PV-Gesamtleistung | W | aktuelle gesamte PV-Erzeugung (`template` im Beispiel) |
| Netzbezug | W | aktuelle Leistung aus dem Netz (`pysmaplus`) |
| Netzeinspeisung | W | aktuelle Leistung ins Netz (`pysmaplus`) |

### Pflichtrollen für Forecast und Zielzeit

Zusätzlich braucht SMACC Forecast-Werte. Im aktuellen Package kommen diese konkret aus **Forecast.Solar** und liegen pro PV-Fläche getrennt vor:

| Rolle | Einheit | Beschreibung |
|---|---:|---|
| Forecast aktuelle Stunde | kWh | erwartete PV-Energie in der laufenden Stunde |
| Forecast nächste Stunde | kWh | erwartete PV-Energie in der nächsten Stunde |
| Forecast Resttag | kWh | verbleibende PV-Energie heute |
| Forecast heute gesamt | kWh | gesamte erwartete PV-Energie heute |
| Forecast morgen gesamt | kWh | gesamte erwartete PV-Energie morgen |

Das Beispiel-Package rechnet konkret mit **drei** Forecast-Quellen bzw. PV-Flächen. In der anonymisierten Version heißen diese Platzhalter:

- `sensor.smacc_pv1_forecast_*`
- `sensor.smacc_pv2_forecast_*`
- `sensor.smacc_pv3_forecast_*`

Wenn ein anderes Setup nur eine oder zwei Forecast-Quellen hat oder anders benannte Forecast-Entitäten liefert, müssen die Templates entsprechend angepasst werden.

### Optionale Rollen

Optional unterstützt SMACC zusätzlich EV-/Wallbox-Kontext, z. B.:

- EV-Lademodus
- EV-Ladeleistung
- Hausverbrauch ohne Auto
- EV-bedingtes PV-Defizit
- Zielwert für Batterie-Entladeleistung im EV-Kontext

Diese Rollen sind nicht zwingend für die Grundfunktion, aber im aktuellen Package bereits berücksichtigt.

## Platzhalter-Entitäten der Share-Version

Die Datei `smacc.yaml` ist absichtlich anonymisiert. Statt realer Geräte-, Flächen- oder Seriennummern-Namen verwendet sie generische Platzhalter:

| Platzhalter | Bedeutung |
|---|---|
| `sensor.smacc_battery_soc` | Batterie-SOC |
| `sensor.smacc_battery_charge_power` | aktuelle Batterieladeleistung |
| `sensor.smacc_battery_discharge_power` | aktuelle Batterieentladeleistung |
| `sensor.smacc_grid_import_power` | aktueller Netzbezug |
| `sensor.smacc_grid_export_power` | aktuelle Netzeinspeisung |
| `sensor.smacc_pv_power_total` | aktuelle gesamte PV-Leistung |
| `sensor.smacc_pv1_forecast_*` | Forecast-Werte der PV-Fläche 1 |
| `sensor.smacc_pv2_forecast_*` | Forecast-Werte der PV-Fläche 2 |
| `sensor.smacc_pv3_forecast_*` | Forecast-Werte der PV-Fläche 3 |

Diese Namen sind **Beispiel-/Mapping-Namen für die Veröffentlichung**. Sie sollen vor allem zeigen, **welche Rolle** eine Entität hat – nicht, wie sie beim ursprünglichen Nutzer hieß.

## Modbus-Anbindung

SMACC ist als **aktive Steuerung** gedacht. Dafür muss das Zielsystem per Modbus beschreibbar sein.

Im aktuellen Beispiel werden unter anderem folgende Register verwendet:

- minimale Ladeleistung
- maximale Ladeleistung
- minimale Entladeleistung
- maximale Entladeleistung
- Grid-Setpoint
- Betriebsmodus

Die konkreten Register und Werte sind **hardware- und setupabhängig**. Wer SMACC auf einem anderen System einsetzt, muss die Modbus-Definition unbedingt an das eigene SMA-/BMS-Setup anpassen.

## Einrichtung

1. Recorder, SQL und Modbus in Home Assistant verfügbar machen.
2. Die Datei `smacc.yaml` in das eigene Packages-Verzeichnis kopieren.
3. Alle Platzhalter-Entitäten im Package auf die eigenen Sensoren und Aktoren mappen.
4. Falls nötig, die Forecast-Logik auf die eigene Anzahl von PV-Flächen umbauen.
5. Modbus-Hub, Slave und Register auf das eigene Gerät anpassen.
6. Home Assistant neu starten.
7. Danach prüfen, ob die zentralen SMACC-Entitäten sauber erzeugt wurden und plausible Werte liefern.

## Nach der Installation prüfen

Direkt nach dem Start sollten insbesondere diese Ergebnisse plausibel sein:

- Entscheidungs-Text
- wirksames Ladeziel
- Ladebeginn aktuelles Ziel
- Ladebeginn Tagesziel
- benötigte Ladezeit bis Ziel
- erwartete Ladeleistung
- Ziel erreicht / Laden gesperrt

Wichtig: Die 3-Tage-Zeitschnitt-Logik braucht Historie. Direkt nach frischer Installation sind diese Werte noch nicht voll aussagekräftig. Nach einigen Stunden wird es besser, nach rund drei Tagen ist die Logik deutlich belastbarer.

## Hinweise

- SMACC ist **nicht Plug-and-Play über feste Entity-Namen**.
- Die Share-Version ist absichtlich anonymisiert und nutzt generische Platzhalter.
- Für andere Installationen müssen die Entitäten nach **Rolle/Bedeutung** gemappt werden.
- Die Forecast-Struktur im Beispiel ist nur eine mögliche Ausprägung.
- Die Modbus-Register, der Hub-Name und die Slave-ID müssen vor produktivem Einsatz zum echten Zielsystem passen.
- Persistente Helper wie eine aktive Ladephase sollten bei Neustarts sauber mitgedacht werden, damit es nicht zu ungewolltem Weiterladen kommt.

## Ziel des Projekts

SMACC soll aus einzelnen Energie-Sensoren eine verständliche und steuernde Ladelogik machen. Nicht nur „Daten anzeigen“, sondern **eine belastbare Ladeentscheidung treffen und erklären** – so, dass man sie in Home Assistant nachvollziehen, anpassen und bei Bedarf aktiv auf das SMA-System anwenden kann.
