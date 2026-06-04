# SMACC
SMACC - SMA Charge Control
SMA Akku Package
GitHub-taugliche Requirements- und Installationsanleitung für das Home-Assistant-Package zur SMA-Akku-Ladesteuerung.
---
Zweck
Dieses Package steuert die Ladefreigabe eines SMA-Akkus auf Basis von:
aktuellem SOC
PV-Leistung und PV-Forecast
Hausverbrauch
Ziel-SOC am Mittag und Tagesende
Zielzeit bis wann der Akku geladen sein soll
optionalen EV-/Wallbox-Signalen
Modbus-Schreibzugriff auf den SMA-Wechselrichter bzw. CmpBMS
> Der aktuelle Stand nutzt für den **Ladebeginn des Tagesziels** nicht mehr primär die 2h-Prognose, sondern einen **30-Minuten-Zeitslot-Schnitt der letzten 3 Tage**.
---
Voraussetzungen in Home Assistant
Pflicht
YAML-Packages aktiviert
Recorder aktiviert
SQL-Integration nutzbar
Template-/Automation-/Script-/Timer-/Helper-Funktionen verfügbar
Modbus-Integration aktiv
Empfohlen
Kurzzeit-Statistiken laufen bereits seit einigen Tagen
Dashboard `sma-akkusteuerung` vorhanden oder geplant
---
YAML-Packages aktivieren
In der `configuration.yaml` z. B.:
```yaml
homeassistant:
  packages: !include_dir_named packages
```
Danach kann das Package z. B. unter folgendem Pfad liegen:
```text
/config/packages/sma_akku_ladesteuerung.yaml
```
---
Benötigte Integrationen
1. Recorder
Pflicht für:
Statistiken
SQL-Abfragen auf Historienwerte
3-Tage-Zeitschnitt-Logik
Die neue Logik liest aus `statistics_short_term`.
2. SQL
Pflicht für die Sensoren:
`sensor.sma_akku_pv_zeitslot_mittel_3d`
`sensor.sma_akku_hausverbrauch_zeitslot_mittel_3d`
3. Modbus
Pflicht für die echte Steuerung.
Verwendeter Service:
```yaml
modbus.write_register
```
Erwartete Modbus-Definition:
Hub: `sma_stp10se`
Slave: `3`
---
Externe Pflicht-Entitäten
Diese Entitäten müssen bereits in Home Assistant vorhanden sein.
Live-Leistungs- und Batterie-Werte
`sensor.pv_produktion_gesamt`
`sensor.shm2_em_metering_power_absorbed`
`sensor.shm2_em_metering_power_supplied`
`sensor.sn_3012402082_battery_power_charge_total_3`
`sensor.sn_3012402082_battery_power_discharge_total_3`
`sensor.sn_3012402082_battery_soc_total_3`
Diese Werte werden verwendet für:
aktuellen Hausverbrauch
Lade-/Entladeprüfung
Zielerreichung
Sperrlogik
PV-Standby-Erkennung
PV-Forecast-Entitäten
`sensor.energy_current_hour`
`sensor.energy_current_hour_2`
`sensor.energy_current_hour_3`
`sensor.energy_next_hour`
`sensor.energy_next_hour_2`
`sensor.energy_next_hour_3`
`sensor.energy_production_today`
`sensor.energy_production_today_2`
`sensor.energy_production_today_3`
`sensor.energy_production_today_remaining`
`sensor.energy_production_today_remaining_2`
`sensor.energy_production_today_remaining_3`
`sensor.energy_production_tomorrow`
`sensor.energy_production_tomorrow_2`
`sensor.energy_production_tomorrow_3`
Diese liefern:
Resttag-Forecast
2h-Forecast
Morgen-Forecast
Grundlage für Zielzeit- und Ziel-SOC-Berechnung
---
Optionale externe Entitäten
Diese Entitäten sind nicht zwingend für die Kernfunktion, werden aber im aktuellen Package berücksichtigt.
EV-/Wallbox-/Verriegelungslogik
`input_boolean.sma_ev_akku_verriegelung_aktiv`
`binary_sensor.sma_ev_akku_verriegelung_soll_aktiv_sein`
`sensor.sma_ev_lademodus_aktuell`
`sensor.sma_ev_lademodus_text`
`sensor.sma_ev_ladeleistung_aktuell`
`sensor.sma_ev_hausverbrauch_ohne_auto`
`sensor.sma_ev_pv_defizit_auto`
`sensor.sma_ev_akku_entladeleistung_ziel`
Wichtig:
`sensor.sma_ev_akku_entladeleistung_ziel` wird direkt beim Modbus-Schreiben verwendet.
Wenn der Sensor fehlt, fällt das Package per Template auf `input_number.sma_akku_entladeleistung_max` zurück. Für einen sauberen Betrieb ist die EV-Logik mit vollständigen Entitäten trotzdem empfehlenswert.
---
Dashboard-Zusatzentitäten
Wenn zusätzlich das Dashboard `sma-akkusteuerung` genutzt werden soll, sind je nach Karten weitere Entitäten sinnvoll.
Optional für Visualisierung
`weather.wetter`
Optional für EV-Karten im Dashboard
`select.smaev_3023521409_operating_mode_of_charge_session`
`sensor.smaev_3023521409_charging_station_power`
`sensor.smaev_3023521409_charging_session_energy`
---
Vom Package angelegte Haupt-Entitäten
Das Package erzeugt selbst eine große Zahl von Helfern, Sensoren, Skripten und Automationen.
Betriebsflags
`input_boolean.sma_akku_ladesteuerung_aktiv`
`input_boolean.sma_akku_ladephase_bis_ziel_aktiv`
`input_boolean.sma_akku_zielzeit_dynamisch_aktiv`
`input_boolean.sma_akku_nachtziel_aktiv`
`input_boolean.sma_akku_morgen_forecast_aktiv`
`input_boolean.sma_akku_balancing_100`
`input_boolean.sma_akku_balancing_woechentlich`
Parameter / Sollwerte
`input_number.sma_akku_ziel_mittag`
`input_number.sma_akku_ziel_tagesende`
`input_number.sma_akku_ziel_hysterese`
`input_number.sma_akku_kapazitaet_kwh`
`input_number.sma_akku_ladewirkungsgrad`
`input_number.sma_akku_ladeleistung_max`
`input_number.sma_akku_entladeleistung_max`
`input_number.sma_akku_ladezeit_reserve_stunden`
`input_number.sma_akku_hausverbrauch_faktor`
`input_number.sma_akku_forecast_sicherheitsfaktor`
Zeitfenster / Zielzeiten
`input_datetime.sma_akku_ladephase_fruehestens_zeit`
`input_datetime.sma_akku_ladephase_spaetestens_zeit`
`input_datetime.sma_akku_ziel_erreicht_zeit`
`input_datetime.sma_akku_pv_ende_letzte`
`input_datetime.sma_akku_pv_start_stabil_letzte`
`input_datetime.sma_akku_balancing_startzeit`
Wichtige abgeleitete Sensoren
`sensor.sma_akku_hausverbrauch_aktuell`
`sensor.sma_akku_hausverbrauch_prognoseleistung`
`sensor.sma_akku_pv_forecast_resttag_gesamt`
`sensor.sma_akku_forecast_pv_leistung_naechste_2h`
`sensor.sma_akku_pv_zeitslot_mittel_3d`
`sensor.sma_akku_hausverbrauch_zeitslot_mittel_3d`
`sensor.sma_akku_erwartete_ladeleistung_zeitslot_3d`
`sensor.sma_akku_benoetigte_ladezeit_bis_ziel`
`sensor.sma_akku_ladebeginn_prognose_text`
`sensor.sma_akku_ladebeginn_wirksames_ziel_text`
`sensor.sma_akku_entscheidung_text`
Wichtige Binary-Sensoren
`binary_sensor.sma_akku_laden_soll_erlaubt_sein`
`binary_sensor.sma_akku_ladezeit_reicht_noch`
`binary_sensor.sma_akku_pv_standby_aktiv`
Skripte / Automationen
`script.sma_akku_cmpbms_laden_erlauben`
`script.sma_akku_cmpbms_laden_sperren`
`script.sma_akku_diagnose_anzeigen`
`automation.sma_akku_cmpbms_ladesteuerung_dynamisch`
`automation.sma_akku_ziel_erreicht_hart_sperren`
---
Modbus-Register
Das Package schreibt auf folgende Register:
Register	Bedeutung
`40793`	Min. Ladeleistung
`40795`	Max. Ladeleistung
`40797`	Min. Entladeleistung
`40799`	Max. Entladeleistung
`40801`	Grid Setpoint
`40236`	Betriebsmodus
Verwendete Modi:
`1438` = Laden erlaubt / Automatik
`2290` = Laden sperren / Entladen
---
Besonderheit der 3-Tage-Zeitschnitt-Logik
Die neue Logik berechnet den Ladebeginn des Tagesziels über:
PV-Mittelwert des aktuellen 30-Minuten-Zeitslots der letzten 3 Tage
Hausverbrauch-Mittelwert des gleichen 30-Minuten-Zeitslots der letzten 3 Tage
Dafür gilt:
`sensor.pv_produktion_gesamt` muss Statistikdaten erzeugen
`sensor.sma_akku_hausverbrauch_aktuell` muss erst Historie aufbauen
direkt nach der ersten Installation ist die 3-Tage-Logik noch nicht voll belastbar
Praktisch bedeutet das
nach Einbau funktioniert die Logik grundsätzlich schon
nach einigen Stunden werden erste Werte brauchbar
nach ca. 3 Tagen ist der Zeitschnitt vollständig aussagekräftig
Im aktuellen Stand gibt es Fallbacks, damit die Berechnung anfangs nicht komplett leer läuft.
---
Installation
1. Backup erstellen
Vorher sichern:
bestehendes Akku-Package
Dashboard `sma-akkusteuerung`
relevante Modbus-Konfiguration
ggf. Automationen oder EV-Helfer
2. Package-Datei kopieren
Beispiel:
```text
/config/packages/sma_akku_ladesteuerung.yaml
```
3. Voraussetzungen prüfen
Vor dem Neustart sicherstellen:
Recorder aktiv
SQL verfügbar
Modbus-Hub `sma_stp10se` vorhanden
Pflicht-Entitäten existieren
4. Home Assistant neu starten
Für Recorder-/SQL-/Package-Änderungen ist ein voller Neustart die sicherste Variante.
5. Nach dem Neustart prüfen
Direkt nach dem Start prüfen:
`automation.sma_akku_cmpbms_ladesteuerung_dynamisch`
`automation.sma_akku_ziel_erreicht_hart_sperren`
`sensor.sma_akku_entscheidung_text`
`sensor.sma_akku_ladebeginn_prognose_text`
`sensor.sma_akku_ladebeginn_wirksames_ziel_text`
`sensor.sma_akku_pv_zeitslot_mittel_3d`
`sensor.sma_akku_hausverbrauch_zeitslot_mittel_3d`
`sensor.sma_akku_erwartete_ladeleistung_zeitslot_3d`
`input_text.sma_akku_ladestatus`
6. Nach 1 bis 3 Tagen erneut prüfen
Dann kontrollieren, ob:
die 3-Tage-Sensoren plausible Werte liefern
der Ladebeginn fürs Tagesziel stabil reagiert
die alte 2h-Prognose nicht mehr die Hauptentscheidung dominiert
---
Bekannte Stolperfallen
Persistente Ladephase
`input_boolean.sma_akku_ladephase_bis_ziel_aktiv` ist persistent.
Das kann nach Reload/Neustart dazu führen, dass eine frühere Ladephase noch aktiv ist, obwohl der neu berechnete Ladebeginn eigentlich noch in der Zukunft liegt.
Fehlende Historie
Die 3-Tage-Logik braucht Historie. Direkt nach Erstinstallation sind die Werte noch begrenzt aussagekräftig.
Fehlende EV-Entitäten
Einige Teile haben Fallbacks, aber für sauberen Betrieb sollte die EV-Logik entweder vollständig vorhanden oder bewusst entfernt/angepasst sein.
---
Mindest-Checkliste
Home Assistant
[ ] YAML-Packages aktiv
[ ] Recorder aktiv
[ ] SQL nutzbar
[ ] Modbus aktiv
Pflicht-Entitäten
[ ] `sensor.pv_produktion_gesamt`
[ ] `sensor.shm2_em_metering_power_absorbed`
[ ] `sensor.shm2_em_metering_power_supplied`
[ ] `sensor.sn_3012402082_battery_power_charge_total_3`
[ ] `sensor.sn_3012402082_battery_power_discharge_total_3`
[ ] `sensor.sn_3012402082_battery_soc_total_3`
[ ] alle benötigten `sensor.energy_*`-Forecast-Entitäten
Modbus
[ ] Hub `sma_stp10se` vorhanden
[ ] Slave `3` korrekt
[ ] Schreibzugriff auf `40793`, `40795`, `40797`, `40799`, `40801`, `40236`
Optional / empfohlen
[ ] EV-/Wallbox-Entitäten vorhanden
[ ] Dashboard-Entitäten vorhanden
[ ] einige Tage Recorder-Historie vorhanden
---
Empfohlene nächste Ausbaustufe
Wenn das Package portabler oder veröffentlichungsreif werden soll, sind diese Punkte sinnvoll:
standortspezifische Entity-IDs parametrisierbar machen
EV-/Wallbox-Logik als optionales Modul trennen
Modbus-Hub, Slave und Register als klaren Konfigurationsblock dokumentieren
Reload-/Neustart-Schutz für `input_boolean.sma_akku_ladephase_bis_ziel_aktiv` ergänzen
---
Datei
Diese GitHub-taugliche Markdown-Datei wurde aus dem aktuell verwendeten Package-Stand abgeleitet.
