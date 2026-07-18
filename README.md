# Universal Self-Learning Irrigation

**Universal Self-Learning Irrigation** ist eine selbstlernende Bewässerungssteuerung für Home Assistant. Das System steuert eine gemeinsame Pumpe und bis zu vier Pflanzenventile, berechnet den Gießbedarf aus Bodenfeuchte und Bodentemperatur, teilt größere Güsse in einzelne Shots auf und verbessert seine Mengenempfehlung aus dem gemessenen Feuchteanstieg.

> [!NOTE]
> Die vorhandenen Dateinamen und Home-Assistant-Entitäts-IDs behalten aus Kompatibilitätsgründen den technischen Präfix `plantinator_bewasserung_`. Der öffentliche Projektname und der GitHub-Repositoryname verwenden „Plantinator“ nicht.

Zum Projekt gehören außerdem:

- manuelle und automatische Bewässerung,
- universelles Crop-Steering mit frei gemappten Quellen und Stage-Namen,
- optionale Tankumwälzung,
- ein beaufsichtigter Mess-Drain-Modus,
- Tages-, Wochen-, Monats- und Gesamtzähler,
- Sensoralter-, Wasserstands- und Alarmüberwachung,
- ein sicherer Stopp bei Fehlern und nach einem Home-Assistant-Neustart,
- ein umfangreiches Lovelace-Dashboard,
- ein optionaler Debug-Logger.

> [!CAUTION]
> Diese Konfiguration schaltet reale Pumpen und Ventile. Ein falsches Mapping, eine falsche Pumpenleistung oder ein ungeeigneter Grenzwert kann Pflanzen, Elektrik und Gebäude beschädigen. Verwende eine physische Überlaufbegrenzung, geeignete Netzteile, Sicherungen und wasserfeste Installation. Teste zunächst mit sehr kleinen Mengen in ein Messgefäß und lasse die Anlage dabei nicht unbeaufsichtigt. Das Projekt ersetzt keine zertifizierte Wasser- oder Elektrosicherung.

> [!IMPORTANT]
> Diese Bewässerungssteuerung wurde bisher ausschließlich mit
> **TRUEBNER-SMT100-Bodenfeuchtesensoren** praktisch getestet. Andere
> Bodenfeuchtesensoren können technisch über das Mapping angebunden werden,
> ihre Messwerte, Skalierung, Filterung und Reaktionszeit wurden mit diesem
> Projekt jedoch nicht validiert. Verwende die mitgelieferten Feuchte-,
> Dryback- und Lernwerte daher nicht ungeprüft mit einem anderen Sensortyp.
> Kalibriere zunächst jeden Sensor im verwendeten Substrat und teste die
> Automatik mit kleinen Wassermengen unter Aufsicht.

## Inhaltsverzeichnis

- [Funktionsweise](#funktionsweise)
- [Screenshots](#screenshots)
- [Voraussetzungen](#voraussetzungen)
- [Dateien im Projekt](#dateien-im-projekt)
- [Installation](#installation)
- [Ersteinrichtung in sicherer Reihenfolge](#ersteinrichtung-in-sicherer-reihenfolge)
- [Standardwerte und Kalibrierung](#standardwerte-und-kalibrierung)
- [Inbetriebnahme](#inbetriebnahme)
- [Bedienung](#bedienung)
- [Automatik im Detail](#automatik-im-detail)
- [Lernfunktion](#lernfunktion)
- [Monitoring und Alarmverriegelung](#monitoring-und-alarmverriegelung)
- [Wartung und Updates](#wartung-und-updates)
- [Fehlersuche](#fehlersuche)
- [Bekannte Grenzen](#bekannte-grenzen)
- [Veröffentlichung auf GitHub](#veröffentlichung-auf-github)

## Funktionsweise

```mermaid
flowchart LR
    A[Bodenfeuchte und Temperatur] --> B[Mapping]
    B --> C[Profil und Lernwert]
    D[Crop-Steering] --> E[Mengenempfehlung]
    F[Optionaler Umweltfaktor] --> E
    C --> E
    E --> G[Shot-Plan]
    G --> H[Shot EXEC]
    I[Manueller Normalguss] --> J[EXEC]
    H --> J
    J --> K[Pumpe und Ventile]
    L[Sensoralter, Wasserstand und Alarm] -. Sicherheitsfreigabe .-> J
    M[Sicher Stop / NOTAUS] -. Abbruch .-> J
    J --> N[Feedback und Verbrauchszähler]
```

Die abgegebene Wassermenge wird zeitbasiert berechnet:

```text
Pumpenlaufzeit in Sekunden = gewünschte Menge in ml / Pumpenleistung in ml/s
```

Ein Durchflussmesser ist nicht erforderlich, aber die Pumpenleistung muss sorgfältig kalibriert werden. Das System prüft nach einem Schaltbefehl, ob Pumpe und Ventile den erwarteten Zustand melden. Es kann jedoch keine verstopfte Leitung, gelöste Schlauchverbindung oder tatsächlich geflossene Wassermenge erkennen.

## Screenshots

Die Bilder zeigen die wichtigsten Bereiche des mitgelieferten Home-Assistant-Dashboards. Klicke auf ein Bild, um es in voller Größe zu öffnen. Die universelle Crop-Steering-Seite enthält Quellen-Mapping, Stage-Aliase sowie editierbare Phasenrezepte für Zeitfenster, Dryback und Shot-Größe.

### Übersicht

[![Übersicht mit Systemfreigaben, Pflanzenstatus und Bodenfeuchteverlauf](docs/images/overview.png)](docs/images/overview.png)

| Automatik | EXEC und manuelle Bewässerung |
|---|---|
| [![Automatik mit Freigaben, Verbrauchszählern und Pflanzenauswahl](docs/images/automation.png)](docs/images/automation.png) | [![EXEC mit Sicherheitsstatus, Shot-Ausführung und Hardwarezuständen](docs/images/exec.png)](docs/images/exec.png) |

| Tankumwälzung | Shot-Plan |
|---|---|
| [![Tankumwälzung mit zyklischem Lauf und Pre-Guss](docs/images/circulation.png)](docs/images/circulation.png) | [![Shot-Plan mit globalen Grenzwerten und Pflanzenplänen](docs/images/shot-plan.png)](docs/images/shot-plan.png) |

| Selbstlernendes Feedback | Bewässerungsprofil |
|---|---|
| [![Selbstlernendes Feedback mit individuellen Lernwerten](docs/images/self-learning-feedback.png)](docs/images/self-learning-feedback.png) | [![Bewässerungsprofil mit Feuchte-, Temperatur- und Mengenwerten](docs/images/profile.png)](docs/images/profile.png) |

| Hardware- und Sensormapping | Universelles Crop Steering |
|---|---|
| [![Mapping für Pumpe, Ventile und Pflanzensensoren](docs/images/mapping.png)](docs/images/mapping.png) | [![Universelles Crop Steering mit Quellen-Mapping, Stage-Aliasen und editierbaren Phasenrezepten](docs/images/crop-steering.png)](docs/images/crop-steering.png) |

| Monitoring und Dryback | Mess-Drain |
|---|---|
| [![Monitoring mit Alarmen, Umweltfaktoren und Dryback](docs/images/monitoring.png)](docs/images/monitoring.png) | [![Mess-Drain mit Pulssteuerung und manuellen EC- und pH-Werten](docs/images/mess-drain.png)](docs/images/mess-drain.png) |

### Debug und Live-Diagnose

[![Debug-Ansicht mit Auto-Check, Logger und Live-Diagnose](docs/images/debug.png)](docs/images/debug.png)

## Voraussetzungen

### Software

- eine aktuelle Home-Assistant-Installation,
- Zugriff auf `/config`,
- aktivierte YAML-Packages,
- ein YAML-Lovelace-Dashboard,
- keine Custom Cards; das Dashboard verwendet Home-Assistant-Kernkarten.

### Erforderliche Hardware-Entitäten

| Funktion | Erwartete Entität | Hinweise |
|---|---|---|
| Gemeinsame Pumpe | `switch.*` | Muss ihren tatsächlichen Ein-/Aus-Zustand zuverlässig melden |
| Ventil je aktiver Pflanze | `switch.*` | Ein eigenes Ventil für P1 bis P4 |
| Bodenfeuchte je aktiver Pflanze | numerischer `sensor.*` | Erwarteter Bereich 0 bis 100 % |
| Bodentemperatur je aktiver Pflanze | numerischer `sensor.*` | Wert in °C |

### Optionale Entitäten

| Funktion | Erwartete Entität |
|---|---|
| Hauptventil | `switch.*` |
| Wasserstand | `binary_sensor.*` oder `input_boolean.*` |
| Tankumwälzung | `switch.*` |
| Zulauf EC, pH und Temperatur | jeweils numerischer `sensor.*` |
| Lufttemperatur und Luftfeuchtigkeit | jeweils numerischer `sensor.*` |
| PPFD | numerischer `sensor.*` |

Crop-Steering funktioniert wahlweise vollständig manuell oder mit jeder externen Software, deren Werte in Home Assistant als Zustand oder Attribut einer Entität verfügbar sind:

| Crop-Quelle | Erwarteter Wert |
|---|---|
| Pflanzenphase | beliebiger kurzer Text, beispielsweise `Stretch`, `Flower Week 4` oder `Blüte 2` |
| Licht an | Uhrzeit als `HH:MM` oder `HH:MM:SS` |
| Licht aus | Uhrzeit als `HH:MM` oder `HH:MM:SS` |

Die Domain der Quell-Entität ist nicht festgelegt. Möglich sind unter anderem `sensor.*`, `select.*`, `input_select.*`, `text.*`, `time.*` und `input_datetime.*`. Liegt der benötigte Wert in einem Attribut, kann dessen Name separat gemappt werden. Ohne externe Quelle verwendet das System die manuellen Stage- und Lichtzeit-Helper.

## Dateien im Projekt

| Datei | Aufgabe |
|---|---|
| [`plantinator_bewasserung_mapping.yaml`](plantinator_bewasserung_mapping.yaml) | Zuordnung der realen Hardware und Pflanzensensoren |
| [`plantinator_bewasserung_profil.yaml`](plantinator_bewasserung_profil.yaml) | Pumpenleistung, Feuchte-, Temperatur- und Mengenlimits |
| [`plantinator_bewasserung_lernsensorik.yaml`](plantinator_bewasserung_lernsensorik.yaml) | Zustandsbewertung und Mengenempfehlung |
| [`plantinator_bewasserung_crop_steering.yaml`](plantinator_bewasserung_crop_steering.yaml) | Universelles Quellen-Mapping, Stage-Aliase, Lichtzeiten und Crop-Parameter |
| [`plantinator_shot_plan.yaml`](plantinator_shot_plan.yaml) | Aufteilung einer Empfehlung in einzelne Shots |
| [`plantinator_bewasserung_exec.yaml`](plantinator_bewasserung_exec.yaml) | Sichere Pumpen- und Ventilsequenz |
| [`plantinator_bewasserung_shot_exec.yaml`](plantinator_bewasserung_shot_exec.yaml) | Ausführung kompletter Shot-Pläne |
| [`plantinator_bewasserung_auto_controller.yaml`](plantinator_bewasserung_auto_controller.yaml) | Pflanzenauswahl, Pausen und automatische Starts |
| [`plantinator_bewasserung_guss_feedback.yaml`](plantinator_bewasserung_guss_feedback.yaml) | Lernen aus Feuchteanstieg und abgegebener Menge |
| [`plantinator_bewasserung_tank_umwalzung_v1.yaml`](plantinator_bewasserung_tank_umwalzung_v1.yaml) | Manuelle, zyklische und Pre-Guss-Umwälzung |
| [`plantinator_mess_drain.yaml`](plantinator_mess_drain.yaml) | Beaufsichtigter Pulsbetrieb bis Drain erkannt wird |
| [`plantinator_bewasserung_monitoring.yaml`](plantinator_bewasserung_monitoring.yaml) | Sensoralter, Umweltwerte, Dryback und Alarmverriegelung |
| [`plantinator_bewasserung_startup_safety.yaml`](plantinator_bewasserung_startup_safety.yaml) | Sicherer Zustand nach Home-Assistant-Start |
| [`plantinator_bewasserung_debug_logger.yaml`](plantinator_bewasserung_debug_logger.yaml) | Optionaler Diagnose-Logger |
| [`dashboard.yaml`](dashboard.yaml) | Vollständiges Lovelace-Dashboard |

Alle `plantinator*.yaml`-Dateien sind voneinander abhängig und sollten gemeinsam installiert werden. `dashboard.yaml` ist keine Package-Datei und darf nicht in den Package-Ordner kopiert werden.

## Installation

### 1. Backup erstellen

Erstelle vor der Installation ein vollständiges Home-Assistant-Backup. Die Home-Assistant-Dokumentation beschreibt Backups unter [Common tasks – Backups](https://www.home-assistant.io/common-tasks/general/).

### 2. Package-Dateien kopieren

Lege beispielsweise diesen Ordner an:

```text
/config/packages/universal_irrigation/
```

Kopiere alle Dateien, deren Name mit `plantinator` beginnt und auf `.yaml` endet, in diesen Ordner. Die Verzeichnisstruktur kann anschließend so aussehen:

```text
/config/
├── configuration.yaml
├── dashboard.yaml
└── packages/
    └── universal_irrigation/
        ├── plantinator_bewasserung_auto_controller.yaml
        ├── plantinator_bewasserung_debug_logger.yaml
        ├── plantinator_bewasserung_exec.yaml
        ├── plantinator_bewasserung_guss_feedback.yaml
        ├── plantinator_bewasserung_lernsensorik.yaml
        ├── plantinator_bewasserung_mapping.yaml
        ├── plantinator_bewasserung_monitoring.yaml
        ├── plantinator_bewasserung_crop_steering.yaml
        ├── plantinator_bewasserung_profil.yaml
        ├── plantinator_bewasserung_shot_exec.yaml
        ├── plantinator_bewasserung_startup_safety.yaml
        ├── plantinator_bewasserung_tank_umwalzung_v1.yaml
        ├── plantinator_mess_drain.yaml
        └── plantinator_shot_plan.yaml
```

Aktiviere Packages in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Existiert bereits ein `homeassistant:`-Block, ergänze darin nur die Zeile `packages:`. Lege keinen zweiten `homeassistant:`-Block an. Weitere Informationen stehen in der offiziellen Dokumentation zu [Home-Assistant-Packages](https://www.home-assistant.io/docs/configuration/packages/).

### 3. Dashboard installieren

Kopiere `dashboard.yaml` direkt nach:

```text
/config/dashboard.yaml
```

Ergänze `configuration.yaml`:

```yaml
lovelace:
  dashboards:
    universal-irrigation:
      mode: yaml
      filename: dashboard.yaml
      title: Universal Self-Learning Irrigation
      icon: mdi:water-pump
      show_in_sidebar: true
      require_admin: true
```

Der Schlüssel `universal-irrigation` muss einen Bindestrich enthalten. `require_admin: true` ist für ein Dashboard, das reale Aktoren schaltet, ausdrücklich empfohlen. Details enthält die Home-Assistant-Dokumentation zu [mehreren YAML-Dashboards](https://www.home-assistant.io/dashboards/dashboards/).

### 4. Optionalen Debug-Logger vorbereiten

Dieser Schritt ist nur nötig, wenn der Debug-Logger Dateien schreiben soll. Bleibt der Schalter **Debug Logger aktiv** aus, kann dieser Abschnitt übersprungen werden.

Ergänze den vorhandenen `homeassistant:`-Block:

```yaml
homeassistant:
  packages: !include_dir_named packages
  allowlist_external_dirs:
    - /config/plantinator_diag
```

Lege den Ordner an:

```text
/config/plantinator_diag
```

Füge anschließend in Home Assistant die Integration **File** hinzu:

1. Öffne **Einstellungen → Geräte & Dienste → Integration hinzufügen**.
2. Wähle **File**.
3. Verwende als Dateipfad `/config/plantinator_diag/plantinator_v3_snapshot.txt`.
4. Benenne die erzeugte Notify-Entität in `notify.plantinator_status` um.

Der Logger verwendet `notify.send_message` und erwartet exakt diese Entitäts-ID. Die offizielle Einrichtung ist unter [File integration](https://www.home-assistant.io/integrations/file/) beschrieben.

### 5. Konfiguration prüfen und neu starten

1. Öffne **Entwicklerwerkzeuge → YAML**.
2. Führe die Konfigurationsprüfung aus.
3. Behebe jeden gemeldeten Fehler.
4. Starte Home Assistant vollständig neu.

Viele Package-Bestandteile sind Helper, Skripte und Automationen. Ein vollständiger Neustart ist bei der Erstinstallation zuverlässiger als ein teilweises Neuladen. Die YAML-Werkzeuge sind in den [Home-Assistant-Entwicklerwerkzeugen](https://www.home-assistant.io/docs/tools/dev-tools/) beschrieben.

## Ersteinrichtung in sicherer Reihenfolge

### 1. Anlage zunächst sperren

Prüfe direkt nach dem Neustart auf der Seite **Übersicht**:

- **Hauptschalter:** aus
- **Automatik:** aus
- **Auto Controller:** aus
- **Sperre:** ein
- **EXEC läuft:** aus
- **Shot EXEC läuft:** aus
- **Umwälzung läuft:** aus

Nach einem Home-Assistant-Start wartet die Startup-Safety 20 Sekunden und schaltet danach Pumpe, Umwälzung und alle gemappten Ventile aus. Ein unterbrochener Guss wird bewusst nicht fortgesetzt. Sind alle drei Automatikschalter weiterhin aktiv, erfolgt erst nach zwei Minuten ein neuer Auto-Check.

### 2. Hardware-Mapping eintragen

Öffne **Mapping**.

#### Zentrale Zuordnung

- **Pumpe Entity:** `switch.*` der gemeinsamen Wasserpumpe
- **Hauptventil Entity optional:** leer lassen oder ein `switch.*` eintragen
- **Wasserstand Entity:** optionaler `binary_sensor.*` oder `input_boolean.*`
- **Wasserstand sicher bei:** `on` oder `off`, passend zur realen Sensorlogik
- **Tank-Umwälzung Entity:** optionaler `switch.*`
- **Wasserstand verwenden:** nur einschalten, wenn Entität und sicherer Zustand geprüft sind
- **Tank-Umwälzung verwenden:** nur bei tatsächlich vorhandener Umwälzpumpe einschalten

#### Zuordnung je Pflanze

Für P1 bis P4:

- **Aktiv:** nur für vorhandene Pflanzen einschalten
- **Name:** frei wählbare Bezeichnung
- **Ventil Entity:** `switch.*`
- **Bodenfeuchte Entity:** numerischer `sensor.*`
- **Bodentemperatur Entity:** numerischer `sensor.*`

Setze `input_number.plantinator_bewasserung_anzahl_pflanzen` passend zur Installation. Nur aktive Pflanzen innerhalb dieser Anzahl werden von Sensoralter und Automatik berücksichtigt.

Dieser Helper ist in der gelieferten Dashboard-Datei derzeit nicht als Eingabefeld eingeblendet. Stelle ihn einmalig unter **Entwicklerwerkzeuge → Aktionen** ein:

```yaml
action: input_number.set_value
target:
  entity_id: input_number.plantinator_bewasserung_anzahl_pflanzen
data:
  value: 4
```

Ersetze `4` durch die tatsächliche Anzahl von 1 bis 4. Das Skript **Mapping Standardwerte setzen** setzt diesen Wert ebenfalls auf 4.

#### Beobachtungssensoren

Zulauf-, Klima- und PPFD-Sensoren sind optional. Sie werden für Monitoring und den Umweltfaktor verwendet. Nicht konfigurierte Beobachtungssensoren blockieren die Kernbewässerung nicht.

> [!WARNING]
> Das Skript **Mapping Standardwerte setzen** trägt installationsspezifische Beispiel-Entitäten wie `switch.pumpe_wasser` und `switch.ventil_pflanze_1` ein. Verwende dieses Skript nicht blind. Trage dein Mapping besser manuell ein oder ersetze unmittelbar danach jede Beispiel-ID, während Hauptschalter aus und Sperre ein bleiben.

Der **Mapping Status** muss anschließend `ok` anzeigen.

### 3. Grundwerte initialisieren

Die YAML-Helper besitzen bewusst keine fest erzwungenen `initial:`-Werte, damit deine Einstellungen Neustarts überleben. Bei einer Neuinstallation solltest du die Grundwerte einmal setzen, während Hauptschalter aus und Sperre ein sind.

Führe über die jeweiligen Dashboard-Seiten diese Skripte aus:

1. **Profil Standardwerte setzen**
2. **Lernsensorik Standardwerte setzen**
3. **Shot Plan Standardwerte setzen**
4. **EXEC Standardwerte setzen**
5. **Feedback Standardwerte setzen**
6. **Auto Standardwerte setzen**
7. **Monitoring Standardwerte setzen**
8. **Umwälzung Standardwerte setzen**, falls das Modul verwendet wird
9. **Crop-Standardwerte setzen**

Das Auto-Skript schaltet den Auto Controller aus und setzt die Tageszähler zurück. Das Feedback-Skript verwirft vorhandene Lernwert-Gültigkeiten. Verwende diese Standardskripte später deshalb nicht unbedacht auf einer bereits eingelernten Anlage.

Das optionale Skript `script.plantinator_bewasserung_debug_logger_standardwerte_setzen` kann über **Entwicklerwerkzeuge → Aktionen** aufgerufen werden. Es schaltet den Logger aus und setzt sein Intervall auf 60 Sekunden.

**Mapping Standardwerte setzen** ist von dieser Liste absichtlich ausgenommen, weil es Beispiel-Entitäten einträgt.

### 4. Profil einrichten

Öffne **Profil** und stelle ein:

- reale Pumpenleistung in ml/s,
- Bodenfeuchte Minimum, Ziel und Maximum,
- maximale Menge pro Normalguss,
- maximale Tagesmenge je Pflanze,
- zulässige Bodentemperatur,
- Mess-Drain-Limits.

Gültig ist nur:

```text
Bodenfeuchte Minimum < Bodenfeuchte Ziel < Bodenfeuchte Maximum
Bodentemperatur Minimum < Bodentemperatur Maximum
Pumpenleistung > 0
```

Der **Profil Status** muss `ok` anzeigen.

### 5. Sensoren prüfen

Prüfe unter **Mapping**, **Übersicht** und **Monitoring**:

- alle aktiven Bodenfeuchten sind numerisch und plausibel,
- alle aktiven Bodentemperaturen sind numerisch und plausibel,
- die Sensorwerte aktualisieren sich,
- **Sensoralter Status** ist `ok`,
- **Kritischer Systemstatus** ist `ok`,
- **Alarm Status** ist `bereit`.

Der Standardwert für das maximal erlaubte Sensoralter ist 30 Minuten.

### 6. Crop-Steering einrichten

Öffne **Crop Steering** und führe zuerst **Crop-Standardwerte setzen** aus. Dadurch werden konservative Phasenrezepte, übliche Stage-Aliase, ein Substratvolumen von 10 Litern und manuelle Lichtzeiten von 06:00 bis 18:00 eingetragen. Anschließend können alle Rezeptwerte im Dashboard verändert werden.

Für die Anbindung einer externen Software:

1. Trage unter **Universelle Crop-Quellen** die Entity-ID für die Pflanzenphase ein.
2. Trage optional den Attributnamen ein, falls die Phase nicht der Zustand der Entität ist.
3. Wiederhole das für **Licht an** und **Licht aus**.
4. Lies den angezeigten **Stage Rohwert** ab.
5. Ergänze diesen Namen unter **Stage-Aliase** bei genau einer internen Phase.
6. Prüfe, dass **Quellen Status** `ok` und **Einstellungen Status** `ok` anzeigt.
7. Trage unter **Crop-Steering Einstellungen** das tatsächliche Substratvolumen ein.
8. Prüfe aktive Rezeptur, Gießschwelle, Shot-Größe, Crop Tagesphase und Normalguss Erlaubnis.

Ohne externe Software lässt du die drei Entity-Felder leer und stellst Stage sowie Lichtzeiten unter **Manueller Fallback** ein. Der Quellenstatus zeigt dann `manuell`; das ist ein gültiger Betriebszustand.

#### Unterschiedliche Stage-Namen zuordnen

Intern verwendet die Bewässerung immer dieselben acht Phasen:

```text
keimung
steckling
early_veg
mid_veg
late_veg
early_flower
mid_flower
late_flower
```

Die externe Bezeichnung wird nicht geraten, sondern über kommagetrennte Aliaslisten eindeutig übersetzt. Beispiele:

| Externer Rohwert | Eintrag in Aliasliste | Interne Phase |
|---|---|---|
| `Seedling` | Keimung | `keimung` |
| `Vegetation Week 2` | Mid Veg | `mid_veg` |
| `Stretch` | Early Flower | `early_flower` |
| `Flower Week 4` | Mid Flower | `mid_flower` |
| `Ripening` | Late Flower | `late_flower` |

Groß-/Kleinschreibung wird ignoriert. Leerzeichen, Bindestriche und Schrägstriche werden wie Unterstriche behandelt. `Flower Week 4`, `flower-week-4` und `flower_week_4` gelten daher als derselbe Name. Ein Alias darf nur in einer Liste vorkommen. Bei einem unbekannten oder mehrdeutigen Namen wird die interne Stage `unbekannt`; die Automatik wechselt sicher auf `nur_manuell`.

## Standardwerte und Kalibrierung

Die Schaltflächen **Standardwerte setzen** schreiben die folgenden Startwerte. Sie sind Beispiele und keine universelle Empfehlung.

### Grundprofil

| Parameter | Standardwert |
|---|---:|
| Pumpenleistung | 50 ml/s |
| Bodenfeuchte Minimum | 32 % |
| Bodenfeuchte Ziel | 42 % |
| Bodenfeuchte Maximum | 55 % |
| Maximalmenge pro Normalguss | 500 ml |
| Tageslimit je Pflanze | 1200 ml |
| Maximalmenge Mess-Drain | 1200 ml |
| Mess-Drain-Puls | 200 ml |
| Mess-Drain-Pulspause | 180 s |
| Bodentemperatur Minimum | 17 °C |
| Bodentemperatur Maximum | 26 °C |

### Lernsensorik und Feedback

| Parameter | Standardwert |
|---|---:|
| Startwert | 35 ml/% |
| Kleinster empfohlener Guss | 50 ml |
| Notguss-Schwelle unter Minimum | 5 Prozentpunkte |
| Feedback-Wartezeit | 4 min |
| Lernrate | 30 % |
| Mindest-Feuchteanstieg | 2 % |
| Untergrenze Lernwert | 20 ml/% |
| Obergrenze Lernwert | 250 ml/% |

### Shot-Plan

| Parameter | Standardwert |
|---|---:|
| Shot Minimum | 50 ml |
| Shot Maximum | 200 ml |
| Mindest-Restmenge | 100 ml |
| Pause Normalguss | 180 s |
| Pause Notguss | 90 s |
| Maximale Shot-Anzahl | 10 |

### Crop-Steering-Phasenrezepte

Die folgenden Werte sind konservative Ausgangspunkte für sensorgestützte Bewässerung in Coco oder Steinwolle. Sie sind keine universelle Garantie. Topfgröße, Substrat, Tropfer, Klima, Sensorposition und Genetik müssen bei der Kalibrierung berücksichtigt werden.

| Interne Phase | Mengenfaktor | Start nach Licht an | Stop vor Licht aus | Dryback-Ziel | Shot vom Substratvolumen |
|---|---:|---:|---:|---:|---:|
| `keimung` | 0,45 | 0 min | 120 min | 5 % | 1,0 % |
| `steckling` | 0,70 | 0 min | 120 min | 5 % | 1,5 % |
| `early_veg` | 0,90 | 30 min | 60 min | 8 % | 2,0 % |
| `mid_veg` | 1,00 | 45 min | 90 min | 8 % | 3,0 % |
| `late_veg` | 1,00 | 90 min | 150 min | 12 % | 3,5 % |
| `early_flower` | 1,00 | 150 min | 210 min | 18 % | 5,0 % |
| `mid_flower` | 1,00 | 90 min | 150 min | 12 % | 3,5 % |
| `late_flower` | 0,90 | 120 min | 180 min | 18 % | 4,0 % |

Weitere Crop-Standardwerte:

| Parameter | Standardwert |
|---|---:|
| Substratprofil | `coco` |
| Substratvolumen | 10 l |
| Ramp-up-Dauer nach Bewässerungsstart | 120 min |
| Ramp-up Shot-Faktor | 60 % |

Während `ramp_up` wird die durch Stage, Substratvolumen und globale
Shot-Grenzen bestimmte Basis-Shot-Größe mit dem Ramp-up-Faktor reduziert.
Aus einem auf 300 ml begrenzten Basis-Shot werden bei 60 % beispielsweise
180 ml. Das globale Shot-Minimum bleibt als Untergrenze aktiv. In
`maintenance` und allen anderen Tagesphasen beträgt der effektive Faktor
automatisch 100 %. Der Faktor verändert nur die Aufteilung in kleinere
Shots; die berechnete Gesamt-Empfehlung und die Sicherheitslimits bleiben
unverändert.

Die Größenordnungen orientieren sich an publizierten Praxisleitfäden: Grodan nennt je nach Strategie ungefähr 1–6 % Shot-Größe relativ zum Substratvolumen sowie spätere Start- und frühere Stopzeiten für generatives Steering. AROYA beschreibt kleinere Drybacks für vegetatives und größere Drybacks für generatives Steering. Das konkrete Substrat bleibt entscheidend; ein Vergleich von Botanicare zeigte unterschiedliche Ergebnisse von Einzel- und Mehrfachpulsen in Coco und Steinwolle.

- [Grodan – Precision Irrigation in Cannabis (PDF)](https://www.grodan.com/siteassets/downloads/downloads-na-101/grow-guide-2023/precision-irrigation.pdf)
- [AROYA – Drybacks 101](https://aroya.io/education-guides/drybacks-101)
- [Botanicare – Irrigation Strategies for Coco Pro and Rockwool](https://www.botanicare.com/hydro-101/irrigation-strategies-cocopro-rockwool/)

### EXEC, Automatik und Umwälzung

| Parameter | Standardwert |
|---|---:|
| Manueller Testguss | 100 ml |
| Ventil-Vorlauf | 1 s |
| Ventil-Nachlauf | 1 s |
| Auto-Check-Intervall | 5 min |
| Mindestpause je Pflanze | 45 min |
| Umwälzintervall | 180 min |
| Zyklische Umwälzdauer | 120 s |
| Pre-Guss-Umwälzdauer | 60 s |
| Mindestpause Pre-Guss | 30 min |

### Monitoring und Umweltfaktor

| Parameter | Standardwert |
|---|---:|
| Maximales Sensoralter | 30 min |
| Klimaeinfluss | 0 % |
| Lichteinfluss | 0 % |
| Zulaufeinfluss | 0 % |
| Maximale gesamte Umweltkorrektur | 10 % |
| VPD-Referenz | 1,2 kPa |
| PPFD-Referenz | 600 µmol/m²/s |
| DLI-Referenz | 35 mol/m² |
| Zulauf-EC-Referenz | 2,0 |
| Zulauf-pH-Referenz | 5,8 |
| Zulauftemperatur-Referenz | 20 °C |

Die drei Umwelteinflüsse beginnen absichtlich bei 0 %. Beobachte die Rohfaktoren zunächst über mehrere Tage. Erhöhe Einflüsse nur in kleinen Schritten und behalte Maximalguss und Tageslimit als harte Grenzen bei.

### Pumpenleistung kalibrieren

1. Führe den Pumpenausgang in einen Messbecher.
2. Sorge dafür, dass eine nicht selbstansaugende Pumpe gefüllt ist. Lass eine dafür ungeeignete Pumpe niemals trocken laufen.
3. Schalte die Pumpe für eine exakt gemessene Zeit ein.
4. Miss die geförderte Menge.
5. Berechne:

```text
Pumpenleistung = gemessene Menge / Laufzeit
```

Beispiel:

```text
500 ml / 10 s = 50 ml/s
```

Wiederhole die Messung mindestens dreimal unter realen Schlauch-, Höhen- und Druckbedingungen und verwende den Mittelwert. Prüfe die Kalibrierung regelmäßig, weil Filter, Leitungen und Pumpen altern.

## Inbetriebnahme

Führe diese Schritte in Anwesenheit und in der angegebenen Reihenfolge aus:

1. Hauptschalter aus, Automatik aus, Auto Controller aus, Sperre ein.
2. Prüfen, dass Pumpe und alle Ventile physisch aus beziehungsweise geschlossen sind.
3. Mapping, Sensorwerte, Wasserstand und Profil kontrollieren.
4. Auslass der ersten Pflanze in ein Messgefäß führen.
5. Sperre ausschalten und Hauptschalter einschalten.
6. Unter **EXEC → Manueller Normalguss** eine kleine, sichere Testmenge einstellen.
7. Nur P1 starten und die reale Schaltfolge beobachten:
   - Pumpe zunächst aus,
   - andere Ventile zu,
   - optionales Hauptventil auf,
   - Pflanzenventil auf,
   - Pumpe an,
   - Pumpe aus,
   - Nachlauf,
   - Ventile zu.
8. Abgegebene Menge nachmessen und Pumpenleistung korrigieren.
9. Den Test für jede aktive Pflanze wiederholen.
10. Während eines kleinen Tests **NOTAUS** beziehungsweise **Sicher Stop** prüfen.
11. Wasserstandssensor während eines beaufsichtigten Tests in den unsicheren Zustand bringen. Der Lauf muss stoppen und der Alarm verriegeln.
12. Ursache beheben und erst danach den Alarm quittieren.
13. Anlage mehrere Tage manuell beobachten.
14. Erst dann Automatik und Auto Controller einschalten.

Nach jedem Test müssen Pumpe, Hauptventil und Pflanzenventile wieder aus sein.

## Bedienung

### Übersicht

Die Übersicht ist die tägliche Betriebsseite.

#### Systemfreigaben

| Element | Wirkung |
|---|---|
| Hauptschalter | Globale Freigabe aller normalen Abläufe |
| Automatik | Erlaubt automatische Bewässerungsentscheidungen |
| Auto Controller | Führt die regelmäßigen Auto-Checks aus |
| Sperre | Blockiert Starts und löst bei Aktivierung einen sicheren Stopp aus |
| EXEC läuft | Kern-Pumpensequenz ist aktiv |
| Shot EXEC läuft | Ein kompletter Shot-Plan wird ausgeführt |
| Shot EXEC reserviert | Ein Shot-Lauf hat die Anlage reserviert, gegebenenfalls schon vor der Pre-Guss-Umwälzung |
| Umwälzung läuft | Der Tankaktor ist aktiv |
| Alarm quittieren | Entriegelt nur, wenn der kritische Systemstatus wieder `ok` ist |
| NOTAUS | Stoppt Pumpe, Umwälzung und Ventile und setzt Abbruchanforderungen |

#### Auto-Kurzstatus

**Shot-EXEC Wartezeit** zeigt den gemeinsamen Timer für:

- Pausen zwischen zwei Shots,
- die anschließende Feedback-Wartezeit.

Der Timer ist daher nicht ausschließlich die Restlaufzeit der Pumpe. Im Leerlauf ist er beendet beziehungsweise inaktiv.

Die Pflanzenübersicht zeigt Feuchte, Feuchtezustand, Gießbedarf, Auto-Freigabe, empfohlene Menge, heutigen Verbrauch und die berechnete Shot-Sequenz.

### Automatik

Für einen automatischen Start müssen gleichzeitig eingeschaltet sein:

- Hauptschalter,
- Automatik,
- Auto Controller.

Zusätzlich müssen Sperre und Alarm aus sein und alle Sicherheitsfreigaben passen. Der Auto Controller prüft im eingestellten Intervall und auch bei relevanten Zustandsänderungen.

Unter **Pflanzenfreigaben** ist der konkrete Ablehnungsgrund jeder Pflanze sichtbar. Der automatische Start verwendet die berechnete Empfehlung und führt sie über den Shot-Plan aus.

Die Tageszähler werden täglich um 00:05 Uhr zurückgesetzt. Wochen- und Monatsverbrauch werden über Utility Meter geführt.

### EXEC

#### Manueller Normalguss

Der manuelle Normalguss verwendet den Wert **Manueller Testguss ml**. Er verwendet nicht automatisch die berechnete Pflanzenempfehlung.

1. Gewünschte Menge einstellen.
2. Sicherheitsstatus kontrollieren.
3. Starttaste der Pflanze drücken.
4. Lauf beobachten.
5. Feedback-Wartezeit abwarten.

Auch ein manueller Guss benötigt eine gültige EXEC-Freigabe. Ein bereits ausstehendes Feedback der Pflanze kann einen weiteren Lauf blockieren.

Ein manueller Normalguss wird auf **Max ml pro Normalguss** begrenzt. Er durchläuft jedoch nicht die automatische Crop-, Mindestpausen- und Tageslimit-Freigabe. Prüfe diese Werte als Bediener selbst.

#### Shot-Plan manuell starten

Die vier Shot-Plan-Starttasten starten die aktuell berechnete Sequenz für die gewählte Pflanze. Vorher prüfen:

- Gießbedarf ist vorhanden,
- Empfehlung und Shot-Sequenz sind plausibel,
- heutiger Verbrauch plus Empfehlung überschreitet das Tageslimit nicht,
- Crop-Freigabe und aktuelle Tagesphase erlauben den Lauf,
- kein anderer Prozess reserviert die Anlage.

Der manuelle Shot-Start prüft Gießbedarf, Empfehlung, Shot-Grenzen, Pflanzensensoren und die allgemeinen Sicherheitsfreigaben. Er verwendet aber nicht die vollständige Auto-Freigabe mit Crop-Regel, Mindestpause und Tageslimit. Diese drei Punkte müssen bei einem manuellen Start vom Bediener geprüft werden.

Mit **Shot-Plan Stop** oder **Shot EXEC Sicher Stop** wird der Lauf abgebrochen.

#### Sicherheitsaktionen

- **Shot EXEC Sicher Stop:** beendet Shot-Timer und Shot-Ausführung.
- **Sicher Stop:** globaler sicherer Stopp.
- **Pumpe aus:** schaltet nur die gemappte Pumpe aus.
- **Alle Ventile zu:** schließt Haupt- und Pflanzenventile.

Im Zweifel immer **Sicher Stop** verwenden.

### Shot Plan

Die Seite zeigt für jede Pflanze:

- Gießbedarf und Empfehlung,
- Anzahl der Shots,
- Sequenz in ml,
- errechnete Pumpenlaufzeiten,
- geplante Gesamtmenge,
- Pause zwischen den Shots,
- gesamte Pumpenlaufzeit.

Größere Mengen werden anhand von **Shot Maximum** geteilt. Eine kleine Restmenge unter **Shot Minimum Rest** wird dem vorherigen letzten Shot zugeschlagen. Ein Notguss verwendet die kürzere Notguss-Pause.

### Feedback

Nach einem Normal- oder Shot-Guss:

1. wird die Bodenfeuchte vor dem Guss gespeichert,
2. bleibt **Feedback ausstehend** aktiv,
3. wartet das System die konfigurierte Feedback-Zeit,
4. liest die Bodenfeuchte erneut,
5. berechnet den Feuchteanstieg und gegebenenfalls einen neuen Lernwert.

Während dieser Transaktion ist ein weiterer Guss derselben Pflanze blockiert. Ändere Feuchtesensor, Lernwerte oder Feedback-Flags nicht während eines laufenden Feedbacks.

### Umwälzung

Die Tankumwälzung ist optional und benötigt einen gemappten `switch.*`.

- **Umwälzung verwenden:** globale Hardwarefreigabe.
- **Zyklisch aktiv:** startet nach Ablauf des Intervalls einen zeitbegrenzten Lauf.
- **Pre-Guss aktiv:** führt vor einem Guss eine Umwälzung aus, wenn die Mindestpause überschritten ist.
- **Manuell starten/stoppen:** direkte Bedienung.
- **Sofort aus:** schaltet den Umwälzaktor aus.

Eine Bewässerung hat Vorrang. Beginnt ein EXEC- oder Shot-Lauf, wird die Umwälzung beendet beziehungsweise nicht gestartet.

### Mess-Drain

> [!WARNING]
> Mess-Drain kann mit den Standardwerten bis zu 1200 ml je Lauf abgeben. Dieser Modus muss durchgehend beaufsichtigt werden.

Der Modus dient dazu, Wasser in Pulsen zuzuführen, bis am Substrat Drain sichtbar wird.

1. Unter **Mess-Drain** den Modus `manuell` wählen.
2. Maximalmenge, Pulsmenge und Pulspause kontrollieren.
3. Messgefäß für den Drain bereitstellen.
4. Starttaste der Pflanze drücken.
5. Den Ablauf ständig beobachten.
6. Sobald Drain sichtbar ist, bei derselben Pflanze **Drain bestätigt / Stop** drücken.
7. Die Anzeige **Erfolgreich abgegeben** ablesen.
8. Drain-EC und Drain-pH bei Bedarf als Handwerte eintragen.
9. Drain-Modus anschließend wieder auf `aus` setzen.

Ohne Bestätigung endet der Lauf spätestens an der Maximalmenge. Die eingetragenen EC- und pH-Handwerte werden protokolliert, aber nicht automatisch zur Bewässerungsregelung verwendet.

### Profil und Mapping

Diese Seiten sind Konfigurationsseiten, keine tägliche Bedienung. Änderungen an Mapping, Pumpenleistung, Grenzen und Maximalmengen können sofort auf den nächsten Lauf wirken. Nimm solche Änderungen nur mit eingeschalteter Sperre und ausgeschalteter Automatik vor.

### Crop Steering

Für jede interne Phase gibt es ein vollständig editierbares Rezept:

- Mengenfaktor für die berechnete Gesamtmenge,
- frühester Bewässerungsstart in Minuten nach Licht an,
- letzter Bewässerungszeitpunkt in Minuten vor Licht aus,
- gewünschter Dryback in Feuchte-Prozentpunkten,
- Shot-Größe als Prozent des Substratvolumens.

Der Schalter **Individuelle Parameter** bestimmt, ob die Dashboardwerte verwendet werden. Ist er aus oder meldet **Einstellungen Status** einen Fehler, werden die dokumentierten Standardwerte verwendet. Ein unbekannter oder nicht eindeutig gemappter Pflanzenstatus blockiert weiterhin die Automatik.

#### Dryback und Gießschwelle

Das Dryback-Ziel wird direkt in die normale Gießentscheidung einbezogen:

```text
Gießschwelle =
  max(Bodenfeuchte Minimum, Bodenfeuchte Ziel - Dryback-Ziel)
```

Beispiel: Ziel 31 %, Minimum 19 %, Dryback 8 % ergibt eine normale Gießschwelle von 23 %. Ein Dryback von 18 % würde rechnerisch 13 % ergeben, wird aber durch das Sicherheitsminimum auf 19 % begrenzt.

Der globale Wert **Bodenfeuchte Minimum** ist damit eine harte Untergrenze für das normale Crop-Steering. `kritisch_trocken` wird weiterhin erst unter **Minimum minus Notguss-Schwelle** ausgelöst. Das Dryback-Rezept darf diese Notfalllogik nicht absenken.

> [!IMPORTANT]
> Die Prozentwerte kapazitiver Feuchtesensoren sind keine genormten Wassergehalte. Kalibriere Ziel, Minimum und Dryback anhand des tatsächlich verwendeten Sensors und Substrats. Verändere Dryback-Ziele schrittweise und beobachte mindestens mehrere vollständige Lichtzyklen.

#### Dynamische Shot-Größe

Die Zielgröße eines Shots wird so berechnet:

```text
Shot ml =
  Substratvolumen in Liter
  × 1000
  × Shot-Prozent
  / 100
```

Bei 10 Litern Substrat entsprechen 3 % beispielsweise 300 ml. Das Ergebnis wird zusätzlich durch **Shot Minimum** und **Shot Maximum** begrenzt. Die berechnete Gesamtmenge, der Maximalguss, die maximale Shot-Anzahl und das Tageslimit bleiben weitere Grenzen.

Das Feld **Substratprofil** kennzeichnet die verwendete Grundlage (`coco`, `steinwolle`, `erde` oder `benutzerdefiniert`). Es verändert keine Werte heimlich; die tatsächlichen Rezeptwerte bleiben im Dashboard sichtbar und einzeln editierbar.

#### Tagesphasen

Aus aktivem Rezept und Lichtzeiten entstehen:

- `morgen_dryback`: vor dem phasenspezifischen Start,
- `ramp_up`: vom Start bis zum Ende der einstellbaren Ramp-up-Dauer,
- `maintenance`: reguläres Bewässerungsfenster,
- `overnight_dryback`: innerhalb der phasenspezifischen Stopzeit vor Licht aus,
- `nacht`: Licht aus.

In Nacht- und Dryback-Phasen wird normaler automatischer Guss eingeschränkt oder gesperrt. Kritische Notgüsse bleiben abhängig von den übrigen Sicherheitsprüfungen möglich. Wenn Start plus Stop mindestens so lang wie die Lichtdauer ist, meldet der Einstellungsstatus `fehler_keine_bewaesserungszeit` und automatische Normalgüsse werden blockiert.

### Monitoring

Die Monitoring-Seite enthält:

- Alarmzustand und letzten Alarmgrund,
- Alter der aktiven Pflanzensensoren,
- Klima-, Licht- und Zulaufsensorstatus,
- Roh- und Effektivwerte des Umweltfaktors,
- VPD, PPFD und DLI,
- Zulauf-EC, pH und Temperatur,
- 1-Stunden- und Nacht-Dryback,
- Verlaufsdiagramme.

Die Monitoring-Drybacks zeigen die gemessene Entwicklung. Das phasenspezifische Dryback-Ziel steuert zusätzlich die normale Gießschwelle; es verändert nicht die harte Notfallgrenze.

### Debug

Die Debug-Seite zeigt die Auto-Entscheidung, Freigaben, Flags und gemappten Aktorzustände. Mit eingerichtetem File-Logger können Testmeldungen, einzelne Snapshots oder periodische Snapshots geschrieben werden.

Der periodische Logger ist standardmäßig aus. Beim Einschalten wird die bestehende Zieldatei geleert und anschließend neu beschrieben.

## Automatik im Detail

### Gießbedarf

Die Bodenfeuchte wird ungefähr so eingeordnet:

| Zustand | Bedingung |
|---|---|
| `kritisch_trocken` | Unter Sicherheitsminimum minus Notguss-Schwelle |
| `trocken` | Unter aktiver Crop-Gießschwelle |
| `unter_ziel` | Zwischen aktiver Crop-Gießschwelle und Ziel |
| `zielbereich` | Zwischen Ziel und Maximum |
| `zu_nass` | Über Maximum |
| Sensorfehler | Kein gültiger Zahlenwert zwischen 0 und 100 |

Ein kritischer Temperaturzustand blockiert einen normalen Guss. Bei kritisch trockener Pflanze kann die Notguss-Logik Vorrang erhalten.

### Mengenempfehlung

Vereinfacht gilt:

```text
Feuchtedefizit = max(0, Ziel - aktuelle Bodenfeuchte)

Empfehlung =
  Feuchtedefizit
  × ml pro Prozent
  × Crop-Faktor
  × Umweltfaktor
```

Anschließend begrenzen Mindestguss und Maximalmenge pro Normalguss die Empfehlung. Das Tageslimit wird zusätzlich bei der automatischen Pflanzenfreigabe geprüft. Manuelle Starts umgehen diese Auto-Prüfung. Solange noch kein gültiger Lernwert vorhanden ist, wird der konfigurierte Startwert in ml/% verwendet.

### Pflanzenauswahl

Eine Pflanze ist nur automatisch freigegeben, wenn unter anderem:

- sie aktiv und innerhalb der Pflanzenanzahl liegt,
- ein Gießbedarf besteht,
- kein Feedback aussteht,
- Crop-Steering den Lauf erlaubt,
- die Mindestpause seit dem letzten Auto-Start abgelaufen ist,
- Empfehlung und Tageslimit den Lauf erlauben.

Die Auswahl erfolgt in dieser Reihenfolge:

1. Notguss P1 bis P4,
2. normaler Bedarf P1 bis P4.

Damit ist die Priorität fest und nicht nach Trockenheitsgrad sortiert. Nach einem Lauf wird erneut geprüft.

### Parallelität

Pumpe, Normalguss, Shot-Plan, Mess-Drain und Umwälzung sind gegenseitig verriegelt. Die Shot-Ausführung setzt bereits vor einer möglichen Pre-Guss-Umwälzung eine Reservierung, damit kein zweiter Ablauf dazwischen startet.

## Lernfunktion

Der gemessene Wert eines Gusses lautet:

```text
berechnete ml/% = abgegebene Menge / Feuchteanstieg
```

Gelernt wird nur, wenn:

- ein gültiges Feedback aussteht,
- die Bodenfeuchte vor und nach dem Guss gültig ist,
- die Menge größer als 0 ist,
- der Feuchteanstieg mindestens dem konfigurierten Mindestanstieg entspricht.

Der Messwert wird zunächst auf die eingestellte Unter- und Obergrenze begrenzt. Danach wird er geglättet:

```text
neuer Lernwert =
  alter Wert × (1 - Lernrate)
  + begrenzter Messwert × Lernrate
```

Bei 30 % Lernrate wirken 30 % des neuen Messwerts und 70 % des bisherigen Werts. Mess-Drain-Läufe werden nicht zum Lernen verwendet.

Beachte, dass kapazitive Feuchtesensoren häufig verzögert, positionsabhängig und nicht linear reagieren. Ein Lernwert ist deshalb eine praktische Regelgröße und keine exakte physikalische Substratkennzahl.

## Monitoring und Alarmverriegelung

### Kritischer Systemstatus

Der Status ist nur `ok`, wenn:

- Mapping Status `ok` ist,
- Profil Status `ok` ist,
- alle aktiven Pflanzen gültige und ausreichend aktuelle Feuchte- und Temperatursensoren haben,
- ein aktivierter Wasserstandssensor gültig ist,
- der Wasserstand dem ausgewählten sicheren Zustand entspricht.

Mögliche Statusgruppen:

| Status | Bedeutung |
|---|---|
| `mapping_fehler` | Mindestens eine erforderliche Zuordnung ist ungültig |
| `profil_fehler` | Pumpen-, Feuchte- oder Temperaturprofil ist unplausibel |
| `sensoralter_pX_bodenfeuchte_ungueltig` | Feuchtesensor fehlt oder ist nicht numerisch |
| `sensoralter_pX_bodentemperatur_ungueltig` | Temperatursensor fehlt oder ist nicht numerisch |
| `sensoralter_pX_bodenfeuchte_veraltet` | Feuchtesensor ist älter als erlaubt |
| `sensoralter_pX_bodentemperatur_veraltet` | Temperatursensor ist älter als erlaubt |
| `wasserstand_ungueltig` | Wasserstandsentität fehlt oder liefert keinen gültigen Zustand |
| `wasserstand_nicht_sicher` | Wasserstand entspricht nicht dem ausgewählten sicheren Zustand |

Bleibt ein kritischer Fehler zwei Minuten bestehen, wird der Alarm verriegelt. Während eines laufenden Wasserprozesses führt ein unsicherer Wasserstand bereits nach etwa zwei Sekunden zum Alarm und sicheren Stopp.

### Alarmstatus

| Alarm Status | Bedeutung |
|---|---|
| `bereit` | Kein aktueller kritischer Fehler und keine gespeicherte Verriegelung |
| `verriegelt` | Alarm ist gespeichert und alle Pumpenabläufe bleiben blockiert; Entriegeln ist erst nach behobener Ursache möglich |
| `fehler_nicht_quittierbar` | Die Ursache besteht noch; Quittierung wird abgelehnt |

### Alarm richtig quittieren

1. Hauptschalter aus oder Sperre ein.
2. Letzten Alarmgrund lesen.
3. Physische Anlage auf Leck, Trockenlauf, offenen Schlauch und Ventilzustand prüfen.
4. Mapping, Sensor und Wasserstand korrigieren.
5. Warten, bis **Kritischer Systemstatus** `ok` zeigt.
6. **Alarm quittieren** drücken.
7. Erst danach kontrolliert wieder freigeben.

Der Quittierknopf umgeht keine Sicherheitsbedingung.

### Sicher Stop

Der globale sichere Stopp:

- erhöht die Stop-Generation, damit ältere Abläufe ungültig werden,
- setzt Abbruch- und Stop-Anforderungen,
- beendet den Shot-Timer,
- schaltet Pumpe und Umwälzung aus,
- schließt Haupt- und Pflanzenventile,
- setzt Lauf- und Reservierungsflags zurück.

Hauptschalter aus oder Sperre ein löst ebenfalls einen sicheren Stopp aus.

## Wartung und Updates

### Regelmäßig kontrollieren

- täglich auf Leckagen und ungewöhnliche Verbrauchsmengen prüfen,
- Feuchtewerte mit dem realen Substrat vergleichen,
- Filter, Tropfer und Ventile auf Verstopfung prüfen,
- Tank und Wasserstandssensor prüfen,
- Pumpenkalibrierung wiederholen,
- Alarmgrund und Hardwarefehlerzähler beobachten,
- Tages-, Wochen- und Monatsverbrauch auf Ausreißer prüfen,
- Backup vor größeren Parameteränderungen erstellen.

### Projekt aktualisieren

1. Automatik und Auto Controller ausschalten.
2. Sperre einschalten.
3. Sicher Stop ausführen.
4. Home-Assistant-Backup erstellen.
5. Package-Dateien und `dashboard.yaml` ersetzen.
6. YAML-Konfiguration prüfen.
7. Home Assistant neu starten, damit neue Helper angelegt werden.
8. Die Versionshinweise prüfen und nur dort ausdrücklich verlangte Standardwerte neu setzen.
9. Mapping, Crop-Status, Profil und Sicherheitsstatus erneut kontrollieren.
10. Einen kleinen manuellen Test durchführen.

### Projekt deaktivieren

Für eine vorübergehende Deaktivierung genügen:

- Automatik aus,
- Auto Controller aus,
- Hauptschalter aus,
- Sperre ein,
- Sicher Stop ausführen.

Entferne Package-Dateien erst, wenn die Aktoren physisch sicher aus sind. Entferne beim vollständigen Ausbau außerdem den Dashboard-Eintrag und gegebenenfalls die File-Integration.

## Fehlersuche

| Problem oder Status | Wahrscheinliche Ursache | Vorgehen |
|---|---|---|
| Dashboard zeigt viele nicht vorhandene Entitäten | Packages nicht geladen oder Neustart fehlt | Package-Pfad und `configuration.yaml` prüfen, Konfiguration validieren, neu starten |
| Mapping Status ist `fehler` | Falsche Domain, leere ID oder inaktive Entität | Mapping-Seite prüfen; Pumpe/Ventile müssen `switch.*`, Pflanzensensoren numerische `sensor.*` sein |
| Profil Status ist `fehler` | Grenzen in falscher Reihenfolge oder Pumpenleistung 0 | Minimum, Ziel, Maximum, Temperaturgrenzen und ml/s korrigieren |
| Sensoralter meldet `ungueltig` | Sensor fehlt, `unknown` oder nicht numerisch | Sensorintegration und Mapping prüfen |
| Sensoralter meldet `veraltet` | Sensor aktualisiert sich nicht innerhalb der Maximalzeit | Funkverbindung, Batterie und Updateintervall prüfen |
| Wasserstand ist nicht sicher | Falscher sicherer Zustand oder Tank leer | Tatsächlichen Zustand in Entwicklerwerkzeuge prüfen und `on`/`off` korrekt wählen |
| Alarm lässt sich nicht quittieren | Kritischer Systemstatus ist noch nicht `ok` | Ursache zuerst beheben |
| EXEC Freigabe nicht `frei` | Hauptschalter, Sperre, Alarm, Mapping, Profil, Wasserstand oder anderer Lauf blockiert | Den angezeigten Freigabegrund schrittweise beheben |
| Shot EXEC bleibt reserviert | Vorlauf/Umwälzung läuft oder ein abgebrochener Ablauf ist nicht bereinigt | Status prüfen, bei Unsicherheit Shot EXEC Sicher Stop und Sicher Stop ausführen |
| Shot-EXEC Wartezeit läuft trotz ausgeschalteter Pumpe | Pause zwischen Shots oder Feedback-Wartezeit | Shot-Status und Feedback ausstehend prüfen |
| Pflanze wird automatisch übersprungen | Kein Bedarf, Crop-Sperre, Mindestpause, Feedback oder Tageslimit | Auto-Freigabe der Pflanze lesen |
| Empfehlung ist 0 ml | Kein Defizit, ungültiger Sensor, Temperatur-/Crop-Sperre oder Tageslimit | Feuchtezustand, Gießbedarf und Freigaben prüfen |
| Automatik läuft nachts nicht | Crop-Tagesphase sperrt normale Güsse | Erwartetes Sicherheitsverhalten; nur Notguss kann erlaubt sein |
| Crop Quellen Status meldet unbekannte oder mehrdeutige Stage | Der externe Rohwert fehlt in den Aliaslisten oder steht in mehreren Listen | Rohwert ablesen und exakt einer Aliasliste zuordnen |
| Crop Einstellungen Status meldet einen Fehler | Aktives Rezept ist ungültig oder Start plus Stop lässt kein Bewässerungsfenster | Crop-Standardwerte setzen, Lichtdauer und aktives Phasenrezept prüfen |
| Tatsächliche Menge stimmt nicht | Pumpenleistung falsch, Druck/Höhe verändert oder Leitung verstopft | Neu kalibrieren und Hardware prüfen |
| Feedback lernt nicht | Anstieg unter Mindestwert, ungültiger Sensor oder keine Feedback-Transaktion | Feedback-Status und Vor-/Nachwerte prüfen |
| Debug-Test meldet unbekannte Notify-Aktion | File-Integration fehlt oder Entity-ID stimmt nicht | `notify.plantinator_status` einrichten oder Debug Logger auslassen |
| Nach Neustart steht `startup_sicher_stop` im Status | Startup-Safety hat einen möglichen Alt-Lauf beendet | Normal; Sicherheitsstatus prüfen und neuen Lauf bewusst starten |

Zusätzliche Diagnosemöglichkeiten:

- **Übersicht → Status**
- **EXEC → Freigabe / Status**
- **Debug → Auto-Check Kernlogik**
- **Debug → Live Diagnose**
- Home-Assistant-Protokoll unter **Einstellungen → System → Protokolle**

## Bekannte Grenzen

- Das System unterstützt höchstens vier Pflanzen.
- Praktisch getestet wurde die Feuchteerfassung bisher nur mit dem TRUEBNER SMT100; andere Sensortypen sind nicht validiert.
- Es verwendet eine gemeinsame Pumpenleistung für alle Pflanzen.
- Die Wassermenge wird aus der Laufzeit berechnet; es gibt keine Rückmeldung eines Durchflussmessers.
- Pumpen und Ventile müssen als `switch.*` vorliegen.
- Nur ein Wasserprozess kann gleichzeitig laufen.
- Die Priorität der Automatik ist fest: Notguss P1 bis P4, danach Normalbedarf P1 bis P4.
- Crop-Regel, Mindestpause und Tageslimit gelten als Auto-Freigaben. Manuelle Normal- und Shot-Starts müssen vom Bediener entsprechend geprüft werden.
- Die Pflanzenanzahl ist als Helper vorhanden, aber in der gelieferten Dashboard-Datei nicht als Eingabefeld eingeblendet.
- Externe Software muss ihre Stage und Lichtzeiten als Home-Assistant-Zustand oder -Attribut bereitstellen; direkte Netzwerk-APIs werden nicht selbst abgefragt.
- Das Dryback-Ziel wird aus der Differenz zwischen Ziel und normaler Gießschwelle abgeleitet. Es ist nur so genau wie der Feuchtesensor und wird durch das globale Sicherheitsminimum begrenzt.
- Manuelle Drain-EC- und pH-Werte werden nicht automatisch geregelt.
- Der Umweltfaktor ist standardmäßig wirkungslos, bis seine Einflüsse bewusst größer als 0 % gesetzt werden.
- Die phasenspezifische Shot-Zielgröße wird durch `Shot Minimum` und `Shot Maximum` begrenzt. Eine kleine Restmenge kann durch die bestehende Mindest-Restlogik dem letzten Shot zugeschlagen werden.
- Nach einem Home-Assistant-Neustart wird ein unterbrochener Guss nicht fortgesetzt.
- Software kann eine physische Leckage-, Überlauf- oder Trockenlaufsicherung nicht ersetzen.

## Veröffentlichung auf GitHub

Vor dem ersten Push:

1. Kontrolliere, dass keine Passwörter, Tokens, IP-Adressen oder sonstige Secrets enthalten sind.
2. Entscheide, ob die Beispiel-Entitäts-IDs im Mapping-Skript bleiben sollen. Kennzeichne sie weiterhin deutlich als Beispiele.
3. Prüfe, ob die mitgelieferte MIT-Lizenz für deine Veröffentlichung passt und ob der angegebene Rechteinhaber korrekt ist.
4. Ergänze bei Bedarf Screenshots in einem Ordner wie `docs/images/`.
5. Beschreibe im Repository, mit welcher Home-Assistant-Version die Konfiguration praktisch getestet wurde.
6. Verwende GitHub-Releases oder Tags für funktionierende Stände.
7. Weise in Issues immer darauf hin, dass Logs vor dem Hochladen auf private Entitätsnamen und andere persönliche Daten geprüft werden müssen.

Beispiel für einen ersten Commit:

```bash
git init
git add README.md dashboard.yaml plantinator*.yaml
git commit -m "Initial release: Universal Self-Learning Irrigation"
```

Danach kann ein zuvor auf GitHub angelegtes Repository als Remote verbunden und gepusht werden.

---

Dieses Projekt ist eine Home-Assistant-Konfiguration für eigenverantwortlichen Einsatz. Beginne mit kleinen Mengen, prüfe jeden Sicherheitsweg und aktiviere die Automatik erst nach erfolgreicher manueller Inbetriebnahme.
