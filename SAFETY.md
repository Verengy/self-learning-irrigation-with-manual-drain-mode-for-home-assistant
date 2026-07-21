# Sicherheitslogik

[Zur Hauptanleitung](README.md) · [English](SAFETY_EN.md)

Diese Datei beschreibt, welche Zustände nur einen Start verhindern, welche
einen einzelnen Bewässerungskanal sperren und welche die gesamte Anlage
verriegeln. Die Logik ersetzt keine mechanische Überlaufbegrenzung,
Leckageüberwachung oder elektrische Schutzeinrichtung.

## Grundprinzip

- Ein Fehler an einem einzelnen Pflanzenventil sperrt möglichst nur diese
  Pflanze. P1 bis P4 werden identisch behandelt.
- Fehler an gemeinsam genutzten Aktoren verhindern alle neuen Starts.
- Kann ein eingeschalteter Aktor nicht nachweislich ausgeschaltet werden,
  wird die gesamte Anlage verriegelt.
- Eine globale Verriegelung kann erst quittiert werden, wenn der kritische
  Zustand behoben und `Alle Aktoren sicher AUS bestätigt` eingeschaltet ist.
- Warnungen und kritische Ereignisse werden im Home-Assistant-Logbuch und in
  der persistenten Benachrichtigung `plantinator_bewasserung_sicherheit`
  dokumentiert.

## Aktorbestätigung bei WLAN-Schaltern

Die Zeit **Schalt-Rückmeldung max. (keine Laufzeit)** ist ein Timeout für
die Zustandsrückmeldung eines WLAN-Schalters. Sie ist keine Bewässerungsdauer.
Das folgende `turn_on`-Verfahren gilt ausschließlich beim Öffnen des
Haupt- und Pflanzenventils. Mit den Standardwerten 8 Sekunden
Bestätigungszeit und 2 Sekunden Wiederholpause läuft es so ab:

1. `turn_on` senden und ab diesem Befehl höchstens 8 Sekunden auf eine neue
   `on`-Rückmeldung warten.
2. Fehlt sie, `turn_off` senden und ab diesem Befehl höchstens 8 Sekunden auf
   `off` warten. War das Ventil bereits nachweislich `off`, ist dieser Schritt
   sofort erfüllt.
3. Zwei Sekunden warten.
4. Einen zweiten `turn_on`-Befehl senden und dafür eine neue Frist von
   höchstens 8 Sekunden starten.
5. Schlägt auch dieser Versuch fehl, erneut ausschalten und den sicheren
   `off`-Zustand mit einer eigenen Frist von höchstens 8 Sekunden prüfen.

Die acht Sekunden beginnen daher **nicht** beim ersten Ausschaltversuch und
laufen nicht als gemeinsame Gesamtfrist. Jeder geprüfte Ventilbefehl erhält
seine eigene Bestätigungsfrist.

Für die Pumpe gilt bewusst etwas anderes: Ihre berechnete Laufzeit startet
unmittelbar mit dem Pumpen-`turn_on`. Gleichzeitig wird auf eine **neue**
`on`-Rückmeldung gewartet, jedoch höchstens bis zum früheren Ende aus
Bestätigungsfrist und berechneter Laufzeit. Ein 3-Sekunden-Shot erhält also
keine zusätzlichen 8 Sekunden. Wird `on` nach einer Sekunde bestätigt, bleiben
noch zwei Sekunden Soll-Laufzeit. Fehlt die Bestätigung bis zum Shot-Ende,
wird sofort ausgeschaltet, **0 ml** werden verbucht und die gemeinsame Pumpe
vorübergehend gesperrt. Ein automatischer zweiter Pumpenstart erfolgt nicht,
weil bei fehlender Rückmeldung sonst eine unbekannte Zusatzmenge fließen
könnte.

Nach dem berechneten Shot-Ende wird sofort `turn_off` gesendet. Erst **ab
diesem Ausschaltbefehl** stehen maximal 8 Sekunden zur Verfügung, um die
sichere `off`-Rückmeldung zu erhalten. Dieses Sicherheitsfenster verlängert
weder den Shot noch die eingeschaltete Sollzeit der Pumpe.

## Fehlerklassen und Reaktion

| Zustand | Reaktion | Freigabe | iPhone |
|---|---|---|---|
| Mapping, Profil, Sensoralter oder Wasserstand vor einem Lauf ungültig | Start wird verhindert | automatisch nach Behebung | keine kritische Meldung |
| Pflanzenventil nicht erreichbar | nur betroffene Pflanze gesperrt | nach mindestens 60 s automatisch, wenn das Ventil wieder erreichbar und `off` ist | zeitkritische Warnung |
| Zwei Öffnungsversuche eines Pflanzenventils fehlgeschlagen, `off` aber bestätigt | nur betroffene Pflanze gesperrt | nach mindestens 60 s automatisch bei zuverlässig gemeldetem `off` | zeitkritische Warnung |
| Hauptventil nicht erreichbar oder zweimal nicht geöffnet, `off` aber bestätigt | alle neuen Starts blockiert, keine globale Alarmverriegelung | nach mindestens 60 s automatisch bei zuverlässig gemeldetem `off` | zeitkritische Warnung |
| Pumpenstart nicht frisch als `on` bestätigt, anschließend `off` aber sicher bestätigt | 0 ml verbucht und alle neuen Starts über Pumpensperre blockiert | nach mindestens 60 s automatisch bei erreichbarer Pumpe mit sicherem `off` | zeitkritische Warnung |
| Wetback-Ziel zu niedrig, Wetback-Sensor ungültig oder unkritischer Nachgießfehler | Sequenz beendet bzw. Nachgießen unterlassen | automatisch beim nächsten gültigen Zyklus | zeitkritische Warnung |
| Pumpe, Pflanzenventil oder Hauptventil lässt sich nach einem Lauf nicht sicher `off` bestätigen | sicherer Stopp und globale Verriegelung | nur manuell nach behobener Ursache und bestätigtem sicheren AUS | kritische Meldung |
| Pumpe unerwartet an oder Pumpe ohne offenes Pflanzenventil | sicherer Stopp und globale Verriegelung | nur manuell | kritische Meldung |
| Pflanzenventil unerwartet offen oder mehrere Pflanzenventile gleichzeitig offen | sicherer Stopp und globale Verriegelung | nur manuell | kritische Meldung |
| Benötigter Aktor wird während des Laufs `unknown` oder `unavailable` | sicherer Stopp und globale Verriegelung | nur manuell | kritische Meldung |
| Wasserstand wird während eines Wasserprozesses unsicher | sicherer Stopp und globale Verriegelung | nur manuell | kritische Meldung |
| Wetback-Peak überschreitet die harte Obergrenze | sofortiger sicherer Stopp und globale Verriegelung | nur manuell | kritische Meldung |
| Sicherer Stopp oder Startprüfung kann nicht bestätigen, dass alle gemappten Aktoren aus sind | globale Verriegelung | nur manuell nach physischer Prüfung | kritische Meldung |

## Pflanzensperren

Im Dashboard unter **Monitoring → Aktorsperren und Laufzustand** stehen für
P1 bis P4 jeweils Sperre und Sperrgrund. Ein gesperrter Kanal wird von der
automatischen Pflanzenauswahl übersprungen. Die übrigen Pflanzen bleiben
verfügbar, solange Pumpe und gemeinsam genutztes Hauptventil sicher sind.

Ein gemeinsames Hauptventil und die gemeinsame Pumpe können nicht einer
einzelnen Pflanze zugeordnet werden. Ihre weichen Aktorsperren blockieren
deshalb alle neuen Starts, lösen aber keine globale Alarmverriegelung aus,
solange `off` sicher bestätigt ist.

## Globale Verriegelung quittieren

1. Hauptschalter ausschalten oder die manuelle Sperre einschalten.
2. Alarmgrund, letztes Sicherheitsereignis und Aktorzustände lesen.
3. Pumpe, Hauptventil, alle Pflanzenventile und Leitungen physisch prüfen.
4. Mapping oder Verbindung reparieren und warten, bis alle Aktoren wieder
   erreichbar sind.
5. **Sicher Stop** ausführen. `Alle Aktoren sicher AUS bestätigt` muss danach
   eingeschaltet sein und der kritische Systemstatus muss `ok` anzeigen.
6. **Alarm quittieren** drücken.
7. Erst anschließend mit einer kleinen, beaufsichtigten Menge neu testen.

Eine Quittierung wird abgelehnt, solange ein kritischer Zustand besteht oder
der sichere AUS-Zustand nicht bestätigt ist.

## Ereignisprotokoll

Jedes Sicherheitsereignis besitzt eine Stufe (`info`, `warnung` oder
`kritisch`), einen Code, Details und einen Zeitstempel. Diese Werte sind im
Monitoring-Dashboard sichtbar. Das Home-Assistant-Logbuch dient als
dauerhafte zeitliche Übersicht; die letzte persistente Benachrichtigung zeigt
den aktuellsten Eintrag.

Der in **Monitoring Einstellungen** eingetragene Notify-Dienst erhält:

- Warnungen mit iOS-Unterbrechungsstufe `time-sensitive`;
- kritische Verriegelungen mit `critical`, Ton und voller Lautstärke.
