SMACC – SMA Charge Control
SMACC ist ein Home-Assistant-Package zur nachvollziehbaren Ladefreigabe von SMA-Batteriesystemen.
Es bewertet PV-Erzeugung, Hausverbrauch, Batteriestand, Ziel-SOC und Zeitfenster und entscheidet daraus, ob der Akku jetzt laden, später laden oder gesperrt bleiben soll. Die Entscheidung wird nicht nur angezeigt, sondern kann über Modbus auch aktiv an den Wechselrichter bzw. die Batterie-Steuerung geschrieben werden.
Warum SMACC?
Viele Setups zeigen zwar Energieflüsse und Forecasts an, haben aber keine saubere Lade-Logik darüber. Typische Probleme sind:
der Akku lädt zu früh, obwohl später genug PV zu erwarten ist
der Akku startet zu spät und verfehlt das Tagesziel
reine Kurzfrist-Forecasts reagieren zu nervös
es ist schwer zu erkennen, warum gerade geladen oder gesperrt wird
SMACC löst das, indem es Live-Daten, Forecasts und historische Mittelwerte zusammenführt und daraus lesbare Entscheidungs-Sensoren und eine steuernde Automatik baut.
Was SMACC macht
SMACC berechnet zunächst den aktuellen energetischen Zustand des Hauses aus PV-Leistung, Netzbezug, Einspeisung und Batterie-Leistung. Darauf aufbauend ermittelt es:
ein wirksames Ladeziel
eine Zielzeit
die fehlende Energie bis zum Ziel
die erwartete Ladeleistung
den sinnvollen Ladebeginn
Der aktuelle Stand nutzt für den Ladebeginn des Tagesziels nicht mehr primär nur die nächsten zwei Forecast-Stunden, sondern zusätzlich einen 30-Minuten-Zeitslot-Schnitt der letzten 3 Tage. Dadurch wird die Startentscheidung meist ruhiger und plausibler.
Zusätzlich erzeugt SMACC Diagnose- und Statussensoren wie z. B.:
aktuelle Entscheidung
aktuelles Ladeziel
Ladebeginn aktuelles Ziel
Ladebeginn Tagesziel
benötigte Ladezeit bis Ziel
Ziel erreicht / Laden gesperrt
Voraussetzungen
Für SMACC werden in Home Assistant mindestens folgende Bausteine benötigt:
YAML-Packages
Recorder
SQL-Integration
Modbus-Integration
Template-, Script-, Automation-, Timer- und Helper-Funktionen
Packages können z. B. so eingebunden werden:
```yaml
homeassistant:
  packages: !include_dir_named packages
```
Benötigte Integrationen
Recorder
Recorder ist Pflicht, weil SMACC historische Statistikdaten nutzt. Die 3-Tage-Logik greift auf die Kurzzeitstatistik von Home Assistant zu. Ohne Recorder fehlen diese Grundlagen.
SQL
Die SQL-Integration wird benötigt, um aus den Recorder-Daten Zeitfenster-Mittelwerte zu bilden, z. B. den PV-Mittelwert oder Hausverbrauch im gleichen 30-Minuten-Slot der letzten drei Tage.
Modbus
Modbus ist erforderlich, wenn SMACC nicht nur anzeigen, sondern tatsächlich steuern soll. Das Package schreibt Register für Lade- und Entladefreigaben sowie Betriebsmodi.
PV-Forecast
SMACC braucht stündliche Forecast-Daten. Welche Integration diese liefert, ist egal – entscheidend ist, dass passende Forecast-Sensoren vorhanden sind.
Benötigte Entitätsrollen
Die konkreten Entity-IDs sind installationsabhängig. Für andere Nutzer müssen diese Werte auf die jeweils vorhandenen Sensoren gemappt werden. Wichtig ist also nicht der Name, sondern die Bedeutung der Entität.
Pflichtrollen für die Kernlogik
Folgende Informationen müssen in Home Assistant vorhanden sein:
Rolle	Einheit	Beschreibung
Batterie-SOC	%	aktueller Ladezustand der Batterie
Batterie-Ladeleistung	W	aktuelle Ladeleistung
Batterie-Entladeleistung	W	aktuelle Entladeleistung
PV-Gesamtleistung	W	aktuelle gesamte PV-Erzeugung
Netzbezug	W	aktuelle Leistung aus dem Netz
Netzeinspeisung	W	aktuelle Leistung ins Netz
Pflichtrollen für Forecast und Zielzeit
Zusätzlich braucht SMACC Forecast-Werte, mindestens für:
Rolle	Einheit	Beschreibung
Forecast aktuelle Stunde	kWh	erwartete PV-Energie in der laufenden Stunde
Forecast nächste Stunde	kWh	erwartete PV-Energie in der nächsten Stunde
Forecast Resttag	kWh	verbleibende PV-Energie heute
Forecast heute gesamt	kWh	gesamte erwartete PV-Energie heute
Forecast morgen gesamt	kWh	gesamte erwartete PV-Energie morgen
Das Beispiel-Package rechnet mit mehreren PV-Flächen. Wenn ein anderes Setup nur eine oder zwei Forecast-Quellen hat, müssen die Templates entsprechend angepasst werden.
Optionale Rollen
Optional unterstützt SMACC zusätzlich EV-/Wallbox-Kontext, z. B.:
EV-Lademodus
EV-Ladeleistung
Hausverbrauch ohne Auto
EV-bedingtes PV-Defizit
Zielwert für Batterie-Entladeleistung im EV-Kontext
Diese Rollen sind nicht zwingend für die Grundfunktion, aber im aktuellen Package bereits berücksichtigt.
Modbus-Anbindung
SMACC ist als aktive Steuerung gedacht. Dafür muss das Zielsystem per Modbus beschreibbar sein.
Im aktuellen Beispiel werden unter anderem folgende Register verwendet:
minimale Ladeleistung
maximale Ladeleistung
minimale Entladeleistung
maximale Entladeleistung
Grid-Setpoint
Betriebsmodus
Die konkreten Register und Werte sind hardware- und setupabhängig. Wer SMACC auf einem anderen System einsetzt, muss die Modbus-Definition unbedingt an das eigene SMA-/BMS-Setup anpassen.
Einrichtung
Recorder, SQL und Modbus in Home Assistant verfügbar machen.
Das Package in das eigene Packages-Verzeichnis kopieren.
Alle lokalen Entity-IDs im Package auf die eigenen Sensoren und Aktoren anpassen.
Falls nötig, die Forecast-Logik auf die eigene Anzahl von PV-Flächen umbauen.
Modbus-Hub, Slave und Register auf das eigene Gerät anpassen.
Home Assistant neu starten.
Danach prüfen, ob die zentralen SMACC-Entitäten sauber erzeugt wurden und plausible Werte liefern.
Nach der Installation prüfen
Direkt nach dem Start sollten insbesondere diese Ergebnisse plausibel sein:
Entscheidungs-Text
wirksames Ladeziel
Ladebeginn aktuelles Ziel
Ladebeginn Tagesziel
benötigte Ladezeit bis Ziel
erwartete Ladeleistung
Ziel erreicht / Laden gesperrt
Wichtig: Die 3-Tage-Zeitschnitt-Logik braucht Historie. Direkt nach frischer Installation sind diese Werte noch nicht voll aussagekräftig. Nach einigen Stunden wird es besser, nach rund drei Tagen ist die Logik deutlich belastbarer.
Hinweise
SMACC ist nicht Plug-and-Play über feste Entity-Namen.
Für andere Installationen müssen die Entitäten nach Rolle/Bedeutung gemappt werden.
Die Forecast-Struktur im Beispiel ist nur eine mögliche Ausprägung.
Die Modbus-Register müssen vor produktivem Einsatz zum echten Zielsystem passen.
Persistente Helper wie eine aktive Ladephase sollten bei Neustarts sauber mitgedacht werden, damit es nicht zu ungewolltem Weiterladen kommt.
Ziel des Projekts
SMACC soll aus einzelnen Energie-Sensoren eine verständliche und steuernde Ladelogik machen. Nicht nur „Daten anzeigen“, sondern eine belastbare Ladeentscheidung treffen und erklären – so, dass man sie in Home Assistant nachvollziehen, anpassen und bei Bedarf aktiv auf das SMA-System anwenden kann.
