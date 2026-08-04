# os-appblocking-maxit – Anwendungsblockierung und Auswertung für OPNsense

`os-appblocking-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG,
das die Intrusion-Detection-Funktion von OPNsense (Suricata) um fertige
Regelsätze für bekannte Anwendungen erweitert: ChatGPT, Netflix, TikTok,
Steam, WeTransfer, öffentliche DoH/DoT-Resolver und einige Hundert weitere,
gruppiert in Kategorien. Ausgewählt wird kategorieweise in der GUI, die
Suricata-Regeln erzeugt das Plugin daraus selbst.

Die zweite Seite des Plugins, **Analyze Threats**, wertet aus, was diese
Regeln tatsächlich gesehen haben: welche Anwendungen im Netz auftauchen, von
welchen Clients, wie oft, mit welchem Datenvolumen und ob die Treffer nur
protokolliert oder wirklich verworfen wurden.

> **Wichtiger Hinweis:** Die Erkennung arbeitet auf den im Klartext
> sichtbaren Namen (DNS-Abfrage, TLS-SNI, HTTP-URL). Wer verschlüsseltes DNS
> oder Encrypted Client Hello nutzt, ist damit nicht mehr erfassbar – deshalb
> die Kategorie **DNS Privacy**. Ebenso gilt: die Regeln vergleichen den
> Domainnamen als Teilstring ohne Anker, siehe
> [Bekannte Grenzen](#bekannte-grenzen).

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Anwendungen auswählen](#anwendungen-auswählen)
4. [Wie die Regeln entstehen](#wie-die-regeln-entstehen)
5. [Analyze Threats – Aufbau der Seite](#analyze-threats--aufbau-der-seite)
6. [Zeitraum und Geltungsbereich](#zeitraum-und-geltungsbereich)
7. [Woher die Zahlen kommen](#woher-die-zahlen-kommen)
8. [Laufzeit bei großen Protokolldateien](#laufzeit-bei-großen-protokolldateien)
9. [Hinweise auf der Seite](#hinweise-auf-der-seite)
10. [Diagramme ohne mitgeliefertes Javascript](#diagramme-ohne-mitgeliefertes-javascript)
11. [API und Kommandozeile](#api-und-kommandozeile)
12. [Berechtigungen](#berechtigungen)
13. [Womit das Plugin verzahnt ist](#womit-das-plugin-verzahnt-ist)
14. [Bekannte Grenzen](#bekannte-grenzen)
15. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über
  das `os-appblocking-maxit` bezogen wird. Entwickelt und geprüft gegen
  **OPNsense 26.7**.
- Die Intrusion-Detection-Funktion unter **Services → Intrusion Detection →
  Administration** muss aktiviert und auf den zu überwachenden Interfaces
  aktiv sein. Ohne laufendes Suricata entstehen weder Regeln noch Alarme.
- Für die Auswertung muss Suricata seine Alarme im EVE-/JSON-Format nach
  `/var/log/suricata/eve.json` schreiben. Das ist die Datei, aus der auch die
  Alarmansicht von OPNsense selbst liest. Ob sie gefüllt wird, lässt sich mit
  `tail -n 1 /var/log/suricata/eve.json` prüfen.
- Kein zusätzliches Paket, kein Dienst, keine Datenbank. Die Auswertung liest
  die Protokolldateien bei jedem Seitenaufruf neu.

## Installation

1. Unter **System → Firmware → Plugins** nach `os-appblocking-maxit` suchen
   und installieren.
2. In der linken Navigation erscheinen unter **Services → Intrusion
   Detection** zwei neue Menüpunkte:
   - **Application Blocking** – Auswahl der zu blockierenden Anwendungen
   - **Analyze Threats** – Auswertung der Treffer

## Anwendungen auswählen

Auf der Seite **Application Blocking** wird das Plugin zunächst über
**Enable** eingeschaltet. Darunter steht je Kategorie ein Mehrfachauswahlfeld
(AI, Crypto, DNS Privacy, Dating, File Hosting, Gambling, Gaming, Shopping,
Streaming, Social Media). Ausgewählt wird pro Anwendung, nicht pro Domain.

**Save & Apply** speichert die Auswahl, erzeugt die Regeldateien neu und lässt
Suricata die Regeln nachladen. Ein Neustart der Firewall oder des Dienstes ist
nicht nötig.

Ob ein Treffer anschließend nur protokolliert oder tatsächlich verworfen wird,
entscheidet **nicht** dieses Plugin, sondern die IDS-Konfiguration: Alle
erzeugten Regeln sind `drop`-Regeln, verworfen wird aber nur, wenn für das
betreffende Interface der IPS-Modus aktiv ist. Ist er es nicht, meldet die
Auswertungsseite das als Warnung, und die Kennzahl **Dropped** steht bei 0 %.

## Wie die Regeln entstehen

| Datei | Rolle |
| --- | --- |
| `src/opnsense/mvc/app/models/OPNsense/Appblocking/Metadata/categories.json` | Pflegedatei: Kategorien, Anwendungen, Domains, Resolver-IPs |
| `tools/generate_category_include.py` | erzeugt daraus das Template-Include |
| `src/opnsense/service/templates/OPNsense/Appblocking/categories.inc` | generiert – **nicht** von Hand bearbeiten |
| `src/opnsense/service/templates/OPNsense/Appblocking/appblocking.*.rules` | ein Regeltemplate je Kategorie |
| `/usr/local/etc/suricata/rules/appblocking.*.rules` | Ergebnis auf dem Zielsystem |

Pro Domain entstehen drei Regeln (DNS-Abfrage, HTTP-URL, TLS-SNI), pro
Resolver-IP der Kategorie DNS Privacy vier weitere (DoH über TCP/443 und
UDP/443, DoT über TCP/853, DoQ über UDP/853); ist für eine Anwendung
zusätzlich der pauschale DoT-Port-Block gesetzt, kommen zwei weitere hinzu.
Die SIDs werden ab `69000000` fortlaufend vergeben, der `classtype` einer
Regel entspricht ihrer Kategorie.

Neue Anwendungen werden in `categories.json` ergänzt, danach wird
`generate_category_include.py` **aus dem Verzeichnis `tools/` heraus**
ausgeführt – die Pfade im Skript sind relativ – und die geänderte
`categories.inc` mit eingecheckt.

## Analyze Threats – Aufbau der Seite

Die Seite besteht von oben nach unten aus:

**Filterzeile** – Zeitraum, Geltungsbereich, **Refresh**. Rechts steht, welches
Zeitfenster ausgewertet wurde und wann der letzte Treffer registriert wurde.
Die Filter gelten für die gesamte Seite, es gibt keine Filter pro Diagramm.

**Sechs Kennzahlen** – Treffer (mit Anzahl Signaturen und Flows), erkannte
Anwendungen (mit Anzahl Kategorien), Clients (mit dem aktivsten), angefragte
Hostnamen (mit dem häufigsten), Flow-Volumen (getrennt nach Richtung) sowie
**Dropped** als Anteil der tatsächlich verworfenen Treffer, darunter ein
Balken verworfen/nur protokolliert.

**Drei Ringdiagramme** – Treffer je Kategorie, Flow-Volumen je Kategorie und
die Art der Erkennung (DNS-Abfrage, TLS-SNI, HTTP-URL, DoH, DoT, DoQ).
Ringdiagramme zeigen höchstens sechs Segmente; alles darunter wird zu
**Other** zusammengefasst, damit die Anteile ablesbar bleiben.

**Zwei Balkenlisten** – die zehn häufigsten Anwendungen (in der Farbe ihrer
Kategorie) und die zehn aktivsten Clients. Ranglisten mit dicht
beieinanderliegenden Werten sind als Balken lesbar, als Kreissegmente nicht.

**Zeitverlauf** – Treffer je Zeitfenster als Balkendiagramm.

**Optionale Ringdiagramme** – Treffer je Interface (nur bei mehr als einem
Interface), TLS-Version der getroffenen Verbindungen (nur wenn TLS-Daten
vorliegen) und Alarm-Schweregrad (nur wenn mehr als ein Schweregrad vorkommt,
praktisch also im Geltungsbereich *All Suricata alerts* – die Regeln dieses
Plugins haben alle denselben).

**Drei Tabellen** – die häufigsten tatsächlich angefragten Hostnamen (also
`global.telemetry.insights.video.a2z.com`, nicht nur die Regeldomain
`a2z.com`), die häufigsten Signaturen mit SID und die eingelesenen
Protokolldateien mit Größe, Trefferzahl und Zustand.

Jede Grafik hat rechts oben eine Schaltfläche, die sie gegen die zugehörige
Tabelle tauscht – dieselben Zahlen, ohne Farbe als Bedeutungsträger, auch für
Copy & Paste in einen Bericht.

## Zeitraum und Geltungsbereich

**Time range** – letzte Stunde, 6 Stunden, 24 Stunden, 7 Tage, 30 Tage oder
*Everything on disk*. Der Zeitraum bestimmt auch, welche Protokolldateien
überhaupt geöffnet werden und wie breit die Balken im Zeitverlauf sind
(5 Minuten bis 1 Woche, ausgerichtet an der lokalen Zeitzone).

**Scope**

- *Application blocking rules* – nur Alarme dieses Plugins. Kategorie,
  Anwendung und Erkennungsart sind dann bekannt.
- *All Suricata alerts* – alle Alarme, die Suricata protokolliert hat, also
  auch ET-Open-Regeln und andere Regelsätze. Als Kategorie dient dann der
  `classtype` des Alarms, als Anwendung der Signaturtext; die Erkennungsart
  entfällt und ihr Diagramm wird ausgeblendet.

## Woher die Zahlen kommen

Ausgewertet wird `/var/log/suricata/eve.json` samt der rotierten Dateien
(`eve.json.0`, `eve.json.1`, … auch gepackt als `.gz`, `.bz2` oder `.xz`).
Das übernimmt das Skript
`/usr/local/opnsense/scripts/OPNsense/Appblocking/eve_stats.py`, das die
Dateien zeilenweise streamt und ein einzelnes JSON-Dokument zurückgibt.

Ein paar Details, die die Zahlen erklären:

- **Dateiauswahl** – Dateien, deren letzte Änderung vor dem gewählten
  Zeitfenster liegt, werden nicht geöffnet. Sie erscheinen in der Tabelle der
  Protokolldateien als *skipped, older than the time range*. Innerhalb einer
  Datei sucht das Skript den Anfang des Zeitfensters per Binärsuche, siehe
  [Laufzeit bei großen Protokolldateien](#laufzeit-bei-großen-protokolldateien).
- **Kategorie** – Suricata kann sie nicht liefern: die `classtype`-Werte des
  Plugins stehen nicht in `classification.config`, im Protokoll steht deshalb
  `"category":"Unknown Classtype"`. Das Skript liest die Kategorie stattdessen
  aus den erzeugten Regeldateien (SID → `classtype`) und greift auf die
  Zuordnung Anwendungsname → Kategorie aus `categories.json` zurück, falls die
  SID dort nicht mehr vorkommt.
- **Erkennungsart und Anwendung** – werden aus dem Signaturtext gelesen, der
  dem Muster `Appblocking - <Anwendung> - <Erkennung>` folgt.
- **Angefragter Hostname** – `dns.queries[].rrname`, `tls.sni` oder
  `http.hostname` des Alarms, sonst die Ziel-IP.
- **Flow-Volumen** – jeder Alarm enthält die Zähler des Flows zum Zeitpunkt
  des Treffers. Da ein Flow oft mehrere Regeln auslöst (erst DNS, dann TLS),
  würde einfaches Addieren doppelt zählen; gezählt wird deshalb je Flow nur
  der Zuwachs seit dem vorherigen Alarm desselben Flows.
- **Grenzen** – nach 5.000.000 gelesenen Zeilen bricht die Auswertung ab, pro
  Dimension werden höchstens 20.000 verschiedene Werte geführt und die
  Flow-Entdopplung greift für 500.000 Flows. Wird eine dieser Grenzen
  erreicht, steht das als Warnung auf der Seite – dann den Zeitraum
  verkleinern.

Unter den Tabellen steht, wie viele Zeilen in wie vielen Millisekunden
gelesen wurden. Das ist der ehrlichste Indikator dafür, ob der gewählte
Zeitraum für die Größe der Protokolldateien noch sinnvoll ist.

## Laufzeit bei großen Protokolldateien

Auf einer Firewall mit viel Verkehr wächst `eve.json` schnell auf 100 MB und
mehr. Damit die Seite trotzdem in Sekunden antwortet, tut das Skript
Folgendes:

- **Es sucht den Anfang des Zeitfensters, statt die Datei von vorn zu lesen.**
  Suricata schreibt zeitlich sortiert, also lässt sich der erste relevante
  Byteoffset per Binärsuche finden – gut zwei Dutzend `seek`-Vorgänge statt
  100 MB. Für „letzte Stunde“ auf einer 100-MB-Datei werden dadurch nur noch
  wenige MB gelesen. In der Zustandsspalte der Dateitabelle steht, wie viel
  vom Dateianfang übersprungen wurde.
- **Sicherheitsabstand von 5 Minuten.** Die Arbeitsthreads von Suricata
  schreiben nicht streng monoton. Die Binärsuche setzt deshalb bewusst früher
  an; jeder Satz wird zusätzlich einzeln gegen das Zeitfenster geprüft. Zu
  früh anzusetzen kostet nur Lesezeit, zu spät würde Datensätze verlieren.
- **Es liest Bytes, nicht Text.** Die ganze Datei nach UTF-8 zu dekodieren,
  um 99 % davon zu verwerfen, wäre der teuerste Einzelposten. Gesucht wird
  deshalb als Bytefolge (`Appblocking - `), und nur die Treffer gehen durch den
  JSON-Parser.
- **Zeitstempel ohne JSON-Parser.** Der Zeitstempel ist das erste Feld jeder
  Zeile und wird als Ausschnitt gelesen. Sätze außerhalb des Fensters kosten
  damit keinen vollständigen Parse – das ist vor allem im Geltungsbereich
  *All Suricata alerts* der Unterschied.
- **Gepackte Dateien werden gestreamt.** In einer `.gz`-Datei zu springen
  bedeutet, sie ohnehin von vorn zu entpacken; hier hilft nur der
  mtime-Filter, der die rotierten Dateien bei kurzen Zeiträumen komplett
  auslässt. Im Standardfall betrifft das niemanden, siehe den nächsten
  Abschnitt: OPNsense packt die rotierten Dateien nicht.

### Wie eve.json rotiert wird

Suricata selbst rotiert nicht (`filetype: regular`, keine Größengrenze in
`suricata.yaml`). Zuständig ist `newsyslog`, und zwar mit einer Datei, die der
Core aus einem Template erzeugt: `OPNsense/IDS/newsyslog.conf` →
`/etc/newsyslog.conf.d/suricata`. Der Eintrag lautet im Kern:

```
/var/log/suricata/eve.json  root:wheel  640  <Save logs>  500000  $<Rotate log>  B  /var/run/suricata.pid  1
```

Daraus folgt für die Praxis:

- **Nicht von Hand editieren.** Die Datei wird beim nächsten Anwenden der
  IDS-Einstellungen aus dem Template neu geschrieben. Die Stellschrauben
  stehen in der GUI unter **Services → Intrusion Detection → Administration**:
  *Rotate log* (Modellfeld `AlertLogrotate`, `W0D23` = weekly oder `D0` =
  daily) und *Save logs* (`AlertSaveLogs`, Anzahl der Generationen, Vorgabe 4).
- **Voreingestellt ist weekly.** Das ist der häufigste Grund für eine
  eve.json von mehreren Hundert MB. Auf *Daily* umzustellen ist die
  wirksamste einzelne Maßnahme.
- **Die Größenschwelle greift praktisch nie.** Die `500000` in der
  Größenspalte sind Kilobyte – rund 488 MiB. Sie ist im Template des Core
  fest verdrahtet, es zählt also der Zeitplan.
- **Rotierte Dateien sind nicht gepackt.** Es ist kein `J`/`Z`-Flag gesetzt,
  sie heißen `eve.json.0`, `eve.json.1` und so weiter. Die Binärsuche
  funktioniert deshalb auf ihnen genauso wie auf der aktiven Datei; die
  Unterstützung für `.gz`, `.bz2` und `.xz` in diesem Plugin ist reine
  Vorsorge für abweichende Setups.
- **Rotation ist ein Rename**, die Änderungszeit der Datei bleibt also
  erhalten. Genau das macht den mtime-Filter zulässig, mit dem zu alte
  Dateien gar nicht geöffnet werden. Anschließend schickt `newsyslog` per
  `/var/run/suricata.pid` ein SIGHUP, damit Suricata die Datei neu öffnet.

Was sich zusätzlich lohnt, wenn es weiterhin zu lange dauert:

1. Kleineren Zeitraum wählen – die Kennzahlen darüber sagen, was der Scan
   gekostet hat.
2. *Rotate log* auf **Daily** stellen und *Save logs* so wählen, dass der
   gewünschte Auswertungszeitraum abgedeckt ist (bei Daily und 30 gespeicherten
   Generationen also 30 Tage). Je feiner rotiert wird, desto mehr Dateien kann
   der mtime-Filter bei kurzen Zeiträumen komplett überspringen.
3. Geltungsbereich *Application blocking rules* statt *All Suricata alerts*:
   der Vorfilter greift dann viel härter.
4. Das eigentliche Protokollvolumen senken, indem im IDS unnötig laute
   Regelsätze abgeschaltet werden.

Reicht das eines Tages nicht mehr – etwa bei mehreren GB pro Tag –, wäre der
nächste Schritt eine inkrementelle Vorverdichtung per Cron: Zwischenstand und
letzter Byteoffset werden gespeichert, jeder Lauf verarbeitet nur die neuen
Bytes. Das ist bewusst noch nicht gebaut, weil es Zustand auf der Firewall
bedeutet und für die üblichen Größen nicht nötig ist.

## Hinweise auf der Seite

Über den Kennzahlen erscheinen Hinweise, die aus den Daten selbst folgen:

| Hinweis | Bedeutung |
| --- | --- |
| Alle Treffer nur protokolliert | Die Regeln greifen, aber der IPS-Modus ist für die überwachten Interfaces aus. Es wird nichts blockiert. |
| Ein Teil der Treffer nur protokolliert | Typisch, wenn IPS nur auf einem von mehreren Interfaces aktiv ist. |
| Keine Suricata-Protokolldatei gefunden | IDS ist nicht aktiv oder schreibt kein EVE-Protokoll. |
| Keine erzeugten Regeln gefunden | Es wurde noch kein **Save & Apply** ausgeführt; Kategorien werden dann nur über den Anwendungsnamen bestimmt. |
| Abbruch nach n Zeilen / langer Rest verworfen | Eine der oben genannten Grenzen wurde erreicht. |

## Diagramme ohne mitgeliefertes Javascript

Die Seite bringt **keine** eigene Diagrammbibliothek mit. Sie nutzt das
Chart.js, das OPNsense für die Dashboard-Widgets ohnehin ausliefert
(`/ui/js/chart.umd.min.js`, siehe `OPNsense/Core/dashboard.volt` im Core).
Damit gibt es nichts zu bauen, nichts nachzuladen und keinen zweiten
Bibliotheksstand, der gepflegt werden müsste.

Was Chart.js nicht selbst kann, ergänzt die Seite: die feste Farbe je
Kategorie (ein geänderter Zeitraum färbt keine Kategorie um), die Zahl in der
Mitte des Rings, die Prozentwerte an den maßgeblichen Segmenten und die
Legende mit den absoluten Werten.

Sollte eine künftige OPNsense-Version diese Datei umbenennen, erkennt die
Seite das, weist darauf hin und zeigt alle Zahlen als Tabellen. Sie bleibt
also benutzbar.

## API und Kommandozeile

Endpunkt der Auswertung:

```
POST /api/appblocking/stats/detections
{"since": "24h", "scope": "appblocking", "top": 25}
```

- `since` – `all` oder eine Zahl mit Einheit: `60m`, `24h`, `7d` (Vorgabe `24h`)
- `scope` – `appblocking` oder `all` (Vorgabe `appblocking`)
- `top` – Länge der Ranglisten, 5 bis 100 (Vorgabe 25)

Ungültige Werte werden auf die Vorgaben zurückgesetzt, nicht abgewiesen. Die
Antwort enthält `totals`, `categories`, `apps`, `clients`, `hosts`, `rules`,
`types`, `interfaces`, `tls_versions`, `severities`, `timeline`, `scan` und
`notices`.

Dasselbe direkt auf der Firewall:

```
configctl appblocking stats 24h appblocking 25
/usr/local/opnsense/scripts/OPNsense/Appblocking/eve_stats.py --since 7d --scope all
```

Der bestehende Endpunkt `POST /api/appblocking/service/reconfigure` erzeugt die
Regeldateien neu, wie es die Schaltfläche **Save & Apply** tut.

## Berechtigungen

Zwei ACLs, zuweisbar unter **System → Zugriff → Gruppen**:

| Berechtigung | Muster | Zweck |
| --- | --- | --- |
| Services: Appblocking | `ui/appblocking/*`, `api/appblocking/*` | vollständiger Zugriff einschließlich Konfiguration |
| Services: Appblocking: Analyze Threats | `ui/appblocking/analyze*`, `api/appblocking/stats/*` | nur die Auswertung, ohne die Anwendungsauswahl ändern zu können |

Die zweite ist für Auswerter gedacht, die berichten, aber nicht konfigurieren
sollen.

## Womit das Plugin verzahnt ist

Diese Schnittstellen der GUI und des Systems werden genutzt und sind gegen
**26.7** geprüft. Wer das Plugin auf eine neuere Version hebt, sollte hier
hinschauen:

| Schnittstelle | Verwendung |
| --- | --- |
| `/var/log/suricata/eve.json` samt Rotation | Datenquelle der Auswertung; rotiert wird über `/etc/newsyslog.conf.d/suricata` aus dem Core-Template `OPNsense/IDS/newsyslog.conf` |
| `/usr/local/etc/suricata/rules/appblocking.*.rules` | Zuordnung SID → Kategorie |
| Suricata-EVE-Felder `alert`, `flow`, `dns`, `tls`, `http`, `in_iface` | alle Kennzahlen der Auswertungsseite |
| `/ui/js/chart.umd.min.js` | Chart.js aus dem Core, siehe [oben](#diagramme-ohne-mitgeliefertes-javascript) |
| `ApiControllerBase` mit JSON-Rumpf | `$this->request->getPost()` liest den JSON-Body der Anfrage |
| configd-Aktion `type:script_output` mit `timeout:180` | Aufruf von `eve_stats.py` als root |
| `IndexController` und `$this->view->pick()` | Einstieg der Seite `analyze.volt` |
| `layout_partials/base_form` | Formular der Seite Application Blocking |
| `POST /api/ids/service/reload_rules` | Nachladen der Regeln nach **Save & Apply** |
| Menü `Services → IDS` | beide Menüpunkte des Plugins |
| Jinja-Templates mit `namespace()` | fortlaufende SID-Vergabe in den Regeltemplates |

## Bekannte Grenzen

- **Teilstringvergleich der Domains.** Die Regeln prüfen den Domainnamen als
  Teilstring ohne Anker. Kurze Domains treffen deshalb auch dort, wo sie nur
  Teil eines anderen Namens sind – `t.co`, `x.com`, `ea.com`, `ps.com` und
  ähnliche Kandidaten blockieren im IPS-Modus deutlich mehr als beabsichtigt.
  Solche Anwendungen mit Bedacht auswählen und die Wirkung anschließend auf
  der Auswertungsseite prüfen.
- **Verschlüsselte Namen sind unsichtbar.** DNS over HTTPS/TLS/QUIC sowie
  Encrypted Client Hello entziehen der Erkennung ihre Grundlage. Die Kategorie
  DNS Privacy blockiert die bekannten Resolver, aber es bleibt ein Wettlauf.
- **Client ist die Quell-IP.** Es gibt keine Zuordnung zu Benutzern; hinter
  NAT oder auf Terminalservern sieht die Auswertung nur eine Adresse.
- **Volumen ist eine Näherung.** Gezählt werden die Flow-Zähler zum Zeitpunkt
  der Alarme, nicht der vollständige Verkehr einer Anwendung. Flows, die vor
  dem Zeitfenster begonnen haben, gehen mit ihrem Stand zum ersten Alarm im
  Fenster ein.
- **Kategorie historischer Alarme.** SIDs werden bei jeder Änderung der
  Auswahl neu vergeben. Sehr alte Alarme können deshalb einer Kategorie
  zugeordnet werden, die zu ihrer Entstehungszeit eine andere war; der
  Rückfall über den Anwendungsnamen fängt die meisten Fälle ab.
- **Keine automatische Aktualisierung.** Die Seite lädt beim Öffnen und bei
  **Refresh**; sie pollt nicht.
- **Kein Zeitraum vor der Rotation.** Was `newsyslog` bereits gelöscht hat,
  kann die Auswertung nicht zeigen. *Everything on disk* heißt genau das:
  aktive `eve.json` plus die unter *Save logs* aufbewahrten Generationen. Wer
  30 Tage auswerten will, muss sie dort auch vorhalten – siehe
  [Wie eve.json rotiert wird](#wie-evejson-rotiert-wird).

## Support / Fehlerbehebung

- **Die Seite bleibt leer, alle Kennzahlen auf 0.** Prüfen, ob überhaupt
  Alarme geschrieben werden:
  `grep -c 'Appblocking - ' /var/log/suricata/eve.json`. Kommt dabei 0 heraus,
  liegt es an der IDS-Konfiguration (aktiv? richtiges Interface? Regeln
  geladen?), nicht an der Auswertung.
- **Fehlermeldung beim Laden.** Das Skript einmal direkt aufrufen, seine
  Ausgabe ist reines JSON und enthält im Fehlerfall `status` und `message`:
  `/usr/local/opnsense/scripts/OPNsense/Appblocking/eve_stats.py --since 24h`
- **Aktion nicht bekannt.** Nach dem Update der configd-Aktionen einmal
  `service configd restart` ausführen.
- **Regeln fehlen unter `/usr/local/etc/suricata/rules/`.** Auf der Seite
  Application Blocking **Save & Apply** ausführen und die Ausgabe beachten;
  Vorlagenfehler melden sich dort im Klartext.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr`
  dieses Verzeichnisses dokumentiert.
