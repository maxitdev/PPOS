# os-reports-maxit – Executive Report für OPNsense

`os-reports-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG.
Es fasst auf einer Seite unter **Reporting → Executive Report** zusammen,
was die Firewall in einem Zeitraum getan hat: blockierte Pakete, IPS-Alarme,
Datenverkehr, Fehler und Drops der Netzwerkkarten, erkannte Anwendungen, Webproxy, DNS, VPN-Verbindungen,
Anmeldungen, Konfigurationsänderungen und den Zustand des Systems. Derselbe
Bericht geht auf Wunsch täglich, wöchentlich oder monatlich als HTML-Mail
hinaus, nach dem Vorbild des täglichen Executive Reports der Sophos UTM.

> **Wichtiger Hinweis:** Top-Listen von Benutzern und internen Hosts nach
> Dauer oder Datenvolumen sind eine Leistungs- und Verhaltenskontrolle im
> Sinne des Betriebsverfassungsgesetzes und personenbezogene Daten im Sinne
> der DSGVO. Vor dem Einsatz mit Betriebsrat bzw. Datenschutz klären und bei
> Bedarf **Pseudonymise users and internal hosts** einschalten oder die
> betroffenen Abschnitte abwählen, siehe
> [Pseudonymisierung](#pseudonymisierung).

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Aufbau der Seite](#aufbau-der-seite)
4. [OpenVPN-Sitzungen](#openvpn-sitzungen)
5. [Woher die Zahlen kommen](#woher-die-zahlen-kommen)
6. [Zeiträume](#zeiträume)
7. [Der Auswertungs-Cache](#der-auswertungs-cache)
8. [Einstellungen](#einstellungen)
9. [Bericht per E-Mail](#bericht-per-e-mail)
10. [Pseudonymisierung](#pseudonymisierung)
11. [API und Kommandozeile](#api-und-kommandozeile)
12. [Berechtigungen](#berechtigungen)
13. [Womit das Plugin verzahnt ist](#womit-das-plugin-verzahnt-ist)
14. [Bekannte Grenzen](#bekannte-grenzen)
15. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über
  das `os-reports-maxit` bezogen wird. Entwickelt und geprüft gegen
  **OPNsense 26.7**.
- Kein Dienst und keine Datenbank. Das Plugin liest die Logs, die OPNsense
  ohnehin schreibt, und hält unter `/var/cache/reports` einen Cache
  vorverdichteter Zwischenstände.
- `os-smtp-cli-maxit` ab Version 1.2 für den Mailversand. Es wird als
  Abhängigkeit automatisch mitinstalliert; eingerichtet wird es unter
  **Services → SMTP-Send**, sobald der Bericht per Mail gehen soll.
- Jeder Abschnitt hängt von seiner Datenquelle ab. Fehlt sie, zeigt der
  Bericht den Abschnitt mit einem Hinweis statt mit Nullen:

  | Abschnitt | braucht |
  | --- | --- |
  | Firewall | Regeln mit aktivierter Protokollierung; die Default-Deny-Regel protokolliert ab Werk |
  | Netzwerknutzung | **Reporting → NetFlow** mit mindestens einem erfassten Interface |
  | Interfaces | nichts, die Zähler des Kernels gibt es immer |
  | Intrusion Prevention | **Services → Intrusion Detection** aktiv, Alarme im EVE-Format |
  | Anwendungskontrolle | `os-appblocking-maxit` mit ausgewählten Anwendungen |
  | Webproxy | `os-squid` mit eingeschaltetem Zugriffsprotokoll, gleich welches Logging-Ziel (siehe [Webproxy](#woher-die-zahlen-kommen)) |
  | DNS | Unbound mit eingeschaltetem **Reporting → Unbound DNS** |
  | VPN | OpenVPN-, IPsec- bzw. WireGuard-Instanzen |
  | Administrativer Zugriff, Konfiguration, System | nichts, das liefert jede OPNsense |


## Installation

1. Unter **System → Firmware → Plugins** nach `os-reports-maxit` suchen und
   installieren. `os-smtp-cli-maxit` wird dabei mitinstalliert.
2. Unter **Reporting** erscheint der Menüpunkt **Executive Report**.
3. Die Statistik füllt sich ab dem ersten Cron-Lauf (spätestens nach einer
   Viertelstunde). Beim ersten Lauf werden die vorhandenen Logs innerhalb
   der Aufbewahrungsdauer nachgelesen, siehe
   [Der Auswertungs-Cache](#der-auswertungs-cache).

## Aufbau der Seite

Die Seite hat drei Reiter: **Report**, **VPN sessions** (siehe
[OpenVPN-Sitzungen](#openvpn-sitzungen)) und **Settings**.

Oben im Reiter **Report** stehen die Auswahl des Zeitraums, **Refresh** und
**Mail preview**. Die Vorschau öffnet den Bericht in einem neuen Fenster so,
wie er per Mail verschickt würde. Rechts stehen Hostname, Firmware und das
genaue Zeitfenster.

Darunter folgen:

- **Summary** – Kacheln mit den wichtigsten Zahlen: Datenverkehr,
  blockierte Pakete, IPS-Alarme (davon blockiert), Anwendungserkennungen,
  Web-Anfragen (davon blockiert), VPN-Verbindungen, fehlgeschlagene
  Anmeldungen, Konfigurationsänderungen. Kacheln abgewählter oder nicht
  verfügbarer Abschnitte entfallen.
- **Activity over time** – blockierte Pakete als Balken und IPS-Alarme als
  Linie auf eigener Achse, je Stunde bis zu zwei Tagen Zeitraum, darüber je
  Tag.
- Je Abschnitt eine Karte mit Kennzahlen und Top-Listen. Jede Liste zeigt
  den Wert, einen Balken relativ zum Spitzenwert und, wo es eine Gesamtsumme
  gibt, den Anteil daran.

| Abschnitt | Kennzahlen | Top-Listen |
| --- | --- | --- |
| Firewall | blockierte Pakete, ein- und ausgehend, Anzahl Quellen | Quellen, Ziele, Dienste, Interfaces, Regeln |
| Netzwerknutzung | Datenverkehr, Pakete, aktive Hosts | Verkehr je Interface, interne Hosts, externe Hosts, Dienste |
| Interfaces | Interface-Fehler, -Drops, Kollisionen, empfangene und verworfene IP-Fragmente, TCP-Wiederholungen, fehlerhafte Prüfsummen, abgewiesene mbuf-Anforderungen | Datenverkehr, Pakete, Fehler, Drops und Kollisionen je Interface; die gewachsenen Fehler- und Drop-Zähler des Kernels |
| Intrusion Prevention | Alarme, blockierte Angriffe, Signaturen, Angreifer | Angreifer, Angriffsziele, Signaturen, Klassifizierungen |
| Anwendungskontrolle | Erkennungen, verworfen, Anwendungen, Clients | Anwendungen, Clients |
| Webproxy | Anfragen, Datenverkehr, blockierte Anfragen, Clients | Websites nach Verkehr und Anfragen, Clients, blockierte Websites und Clients |
| DNS | Anfragen, durch Blockliste blockiert, aus dem Cache | Domains, blockierte Domains |
| VPN | OpenVPN-Verbindungen, -Benutzer, jetzt verbunden, Verbindungsdauer, OpenVPN-Datenvolumen, fehlgeschlagene Anmeldungen, IPsec-Tunnel, aktive WireGuard-Peers | OpenVPN-Benutzer (Verbindungen, Dauer, Upload, Download, zuletzt verbunden, letzte Adresse), die letzten 30 OpenVPN-Verbindungen, fehlgeschlagene Anmeldungen, IPsec-Verbindungen und -Gegenstellen, WireGuard-Peers |
| Administrativer Zugriff | WebGUI-, SSH- und Konsolenanmeldungen, fehlgeschlagene | Anmeldungen je Benutzer, Fehlversuche je Quelle und Benutzername |
| Konfigurationsänderungen | Änderungen, Anzahl Benutzer | Änderungen je Benutzer, letzte Änderungen mit Beschreibung |
| System | durchschnittliche und Spitzenlast, durchschnittliche und höchste CPU-Auslastung, CPU-Kerne, Firmware, verfügbare Updates, Laufzeit, Neustarts, Gateway-Alarme, CARP-Umschaltungen, aktualisierte Pakete, Speicherplatz, aktuelle Systemlast | Diagramm Last und CPU-Auslastung je Stunde bzw. Tag; Gateway-Alarme, Ereignisse |

## OpenVPN-Sitzungen

Für jede OpenVPN-Verbindung hält das Plugin einen Datensatz fest:

| Feld | Bedeutung |
| --- | --- |
| User | Benutzername; ohne Benutzer-Authentifizierung der Common Name des Zertifikats. Weicht der Common Name vom Benutzernamen ab, steht er in Klammern dahinter. |
| Remote address | öffentliche Adresse und Port, von der sich der Client verbunden hat |
| Tunnel address | die zugewiesene VPN-Adresse, IPv4 und, falls vergeben, IPv6 |
| Connected | Beginn der Verbindung |
| Disconnected | Ende der Verbindung oder **still connected** |
| Duration | Dauer, bei offenen Verbindungen bis jetzt |
| Upload / Download | übertragene Daten aus Sicht des Benutzers: Upload ist, was der Server vom Client empfangen hat |
| Ended by | Grund des Endes, siehe unten; dahinter die Serverinstanz |

**Woher die Daten kommen.** Zwei Quellen ergänzen sich:

- Das **OpenVPN-Log** (`/var/log/openvpn/`) liefert Beginn
  (`Peer Connection Initiated`), Ende (`client-instance exiting`), die vom
  Pool vergebene Tunneladresse (`MULTI_sva: pool returned` bzw.
  `MULTI: Learn`) und den Grund des Endes.
- Die **Management-Schnittstelle** jeder Serverinstanz
  (`/var/etc/openvpn/*.sock`, Befehl `status 3`) liefert für alle gerade
  verbundenen Clients Adresse, Tunneladresse, Bytezähler und „verbunden
  seit“. Das sind dieselben Daten wie unter **VPN → OpenVPN → Connection
  Status**. Das Plugin fragt sie bei jedem Lauf ab, also alle 15 Minuten
  und beim Öffnen der Seite.

Die Datenmenge einer Verbindung ist deshalb der Stand der letzten Abfrage
vor ihrem Ende, im Mittel also bis zu 15 Minuten zu niedrig. In die
Kennzahlen eines Zeitraums geht jeweils der Zuwachs zwischen zwei Abfragen
ein, so verteilt sich das Volumen langer Verbindungen richtig auf die Tage.
Verbindungen, die schon vor der Installation oder vor dem ältesten Log
bestanden, werden über die Statusabfrage gefunden und mit ihrem echten
Beginn erfasst.

> **Warum nicht Interfaces → Diagnostics → Netstat?** Netstat zeigt Sockets,
> Routen und Zähler je Interface. Alle Clients einer OpenVPN-Instanz teilen
> sich aber ein Interface (`ovpns1` usw.); eine Aufteilung nach Benutzer
> gibt es dort nicht. Die gibt nur OpenVPN selbst über seine
> Management-Schnittstelle her.

**Gründe für das Ende:**

| Anzeige | Bedeutung |
| --- | --- |
| client disconnected | der Client hat sich abgemeldet (`remote-exit`) |
| timeout | keine Keepalive-Antwort mehr (`ping-restart`) |
| server restart | die Serverinstanz wurde neu gestartet |
| new session from the same address | dieselbe Adresse und derselbe Port haben eine neue Verbindung aufgebaut |
| gone from the server status | das Log enthielt kein Ende, die Statusabfrage listet den Client aber nicht mehr; als Ende gilt die letzte Abfrage, in der er noch verbunden war |
| no sign of life for 14 days | weder Log noch Status haben die Verbindung 14 Tage lang bestätigt |

Andere Signale aus dem Log (z. B. `tls-error`) erscheinen unverändert.

**Reiter VPN sessions.** Oben stehen Zeitraum, das Auswahlfeld **User**
(mit Suche), ein Freitextfilter, **Only connected**, **Refresh** und
**CSV**.

- **Alle Benutzer** (Auswahl *All users*): Die Tabelle **Users** zeigt je
  Benutzer Verbindungen, Verbindungszeit, Upload, Download, fehlgeschlagene
  Anmeldungen, zuletzt verbunden und die letzte Herkunftsadresse; ein grüner
  Punkt markiert, wer gerade verbunden ist. Ein Klick auf den Namen wählt
  den Benutzer aus.
- **Ein Benutzer**: Statt der Übersicht erscheint seine Statistik:
  - Kennzahlen: Verbindungen, Verbindungszeit, durchschnittliche und längste
    Verbindung, Upload, Download, fehlgeschlagene Anmeldungen, Anzahl
    verschiedener Herkunftsadressen; daneben, ob er gerade verbunden ist
    (seit wann) oder wann er sich zuletzt getrennt hat;
  - die Diagramme **Connected time** und **Data** (Upload und Download)
    je Stunde, bei Zeiträumen über zwei Tage je Tag;
  - die Tabellen **Remote addresses** und **Tunnel addresses** (je Adresse
    Verbindungen, Dauer, zuletzt verbunden) und **How the sessions ended**.

  **All users** neben dem Namen führt zurück zur Übersicht.

Verbindungszeit, Upload, Download und Fehlversuche in Übersicht,
Kennzahlen und Diagrammen stammen aus den Stundenständen und gelten genau
für den Zeitraum: Eine Verbindung, die vor dem Zeitraum begann, zählt nur
mit ihrem Anteil darin. Die Sitzungsliste darunter zeigt dagegen jede
Verbindung mit ihrer vollen Dauer und Datenmenge.

Die Liste **Sessions** enthält alle Verbindungen, die in den Zeitraum
fallen, auch solche, die davor begonnen haben oder noch bestehen, bis zu
5000 Stück, die neuesten zuerst, eingeschränkt auf den gewählten Benutzer.
Das Suchfeld filtert zusätzlich nach Benutzer, Common Name, Adressen und
Serverinstanz, **Only connected** zeigt nur die offenen Verbindungen. Ein
Klick auf einen Spaltenkopf sortiert. Oben rechts stehen die Summen von
Upload und Download der angezeigten Zeilen. **CSV** lädt genau die
angezeigten Zeilen herunter, bei gewähltem Benutzer mit dessen Namen im
Dateinamen: Semikolon als Trennzeichen, UTF-8 mit BOM, Zeiten in UTC nach
ISO 8601, Dauer in Sekunden, Daten in Bytes. So öffnet ein deutsches Excel
die Datei direkt.

Im Bericht und in der Mail steht im Abschnitt VPN die Tabelle **OpenVPN
users** (je Benutzer Verbindungen, Dauer, Upload, Download, zuletzt
verbunden und von welcher Adresse) und die Tabelle **Latest OpenVPN
sessions** mit den 30 neuesten Verbindungen des Zeitraums.

## Woher die Zahlen kommen

| Abschnitt | Quelle | Art |
| --- | --- | --- |
| Firewall | `/var/log/filter/filter_*.log` (filterlog), Regelbeschreibungen aus `/tmp/rules.debug` | Cache |
| Netzwerknutzung | NetFlow-Aggregate des Core, abgefragt über `/usr/local/opnsense/scripts/netflow/get_top_usage.py` | live |
| Interfaces | `netstat -i -b -n -d -W`, `netstat -s`, `netstat -m`, dieselben Zähler wie unter **Interfaces → Diagnostics → Netstat** | Cache, Probe bei jedem Lauf |
| Intrusion Prevention, Anwendungskontrolle | `/var/log/suricata/eve.json*`, Ereignistyp `alert`; Signaturen `Appblocking - …` zählen als Anwendung, alle anderen als IPS-Alarm | Cache |
| Webproxy | `/var/log/squid/access.log*` bzw. bei Syslog `/var/log/squid/access/access_*.log`, jedes Format, das `os-squid` schreibt | Cache |
| DNS | configd-Aktion `unbound qstats totals` des Core | live |
| VPN | `/var/log/openvpn/openvpn_*.log`, OpenVPN-Management-Sockets `/var/etc/openvpn/*.sock`, `/var/log/ipsec/ipsec_*.log`, `wg show all dump` | Cache, WireGuard live |
| Administrativer Zugriff, System-Ereignisse | `/var/log/audit/audit_*.log`, `/var/log/system/system_*.log`, `/var/log/gateways/gateways_*.log` | Cache |
| Konfigurationsänderungen | `<revision>` aus `/conf/backup/config-*.xml` und `/conf/config.xml` | Cache |
| Systemzustand | `opnsense-version`, `kern.boottime`, `/tmp/pkg_upgrade.json`, Belegung von `/`, Load Average | live |
| Last und CPU-Auslastung | Load Average (`getloadavg`), `kern.cp_time`, `hw.ncpu` | Cache, Probe bei jedem Lauf |

Einige Zählweisen im Einzelnen:

- **Firewall.** Eine protokollierte Zeile mit Aktion `block` ist ein Paket.
  Pakete von Regeln ohne Protokollierung tauchen nicht auf; erlaubte Pakete
  werden nicht gezählt.
- **Datenverkehr.** Summe der eingehenden Bytes aller von NetFlow erfassten
  Interfaces. Geroutete Pakete betreten die Firewall genau einmal, so wird
  nichts doppelt gezählt. Die Host-Listen zählen je Adresse, was sie gesendet
  und empfangen hat, und trennen private bzw. link-lokale Adressen (intern)
  von allen anderen (extern). Dienste ab Port 49152 werden ausgeblendet, weil
  dort fast nur die Antwortflüsse zu den dynamischen Ports der Clients
  landen.
- **Webproxy.** `os-squid` bietet unter **Services → Squid Web Proxy →
  General** im Feld **Access log target** (erweiterte Ansicht) fünf Ziele
  für das Zugriffsprotokoll. Alle werden gelesen:

  | Ziel | Datei | Format |
  | --- | --- | --- |
  | File | `/var/log/squid/access.log` | natives Squid-Format |
  | File (Extended) | `/var/log/squid/access.log` | Client, MAC, Benutzer, `[Zeit]`, Anfrage, Status, Bytes, Referer, User-Agent, Squid-Status |
  | File (Json) | `/var/log/squid/access.log` | ein JSON-Objekt je Zeile (ECS) |
  | Syslog | `/var/log/squid/access/access_*.log` | natives Format in der Syslog-Zeile |
  | Syslog (Json) | `/var/log/squid/access/access_*.log` | JSON in der Syslog-Zeile |

  Das Format wird für jede Zeile einzeln erkannt; wer das Ziel mitten im
  Zeitraum umstellt, verliert nichts. Als Client zählt der angemeldete
  Benutzer, ohne Anmeldung die Adresse. Blockiert ist eine Anfrage mit
  Squid-Status `TCP_DENIED`.
- **Last und CPU-Auslastung.** Bei jedem Lauf (alle 15 Minuten) wird der
  Load Average über 1, 5 und 15 Minuten festgehalten. **Durchschnittliche
  Last** und die Balken im Diagramm sind der Mittelwert des 1-Minuten-Werts
  aller Proben im Zeitraum bzw. in der Stunde oder am Tag, **Spitzenlast**
  und die gestrichelte Linie der höchste gemessene Wert. Die CPU-Auslastung
  stammt aus den Tick-Zählern des Kernels (`kern.cp_time`, alle Kerne
  zusammen): Der Anteil der belegten Ticks zwischen zwei Proben ist die
  tatsächliche durchschnittliche Auslastung dieser Viertelstunde, kein
  Momentwert. **Höchste CPU-Auslastung** ist die stärkste Viertelstunde.
  Nach einem Neustart fehlt die Viertelstunde mit dem Neustart. Die Kachel
  **Systemlast** zeigt weiterhin den aktuellen Wert beim Erstellen des
  Berichts. Kurze Spitzen von wenigen Sekunden sieht diese Abtastung nicht.
- **Interfaces.** Der Kernel zählt seit dem Booten. Das Plugin nimmt bei
  jedem Lauf (alle 15 Minuten) eine Probe und bucht den Zuwachs seit der
  vorigen in den aktuellen Stundenabschnitt. Ist ein Zähler kleiner als
  zuvor, wurde neu gestartet, und er zählt ab null. Die erste Probe nach der
  Installation ist nur der Ausgangswert; Zahlen gibt es ab dem zweiten Lauf.
  Gezählt wird je Interface die `<Link#n>`-Zeile; `lo0`, `pflog` und
  `pfsync` bleiben außen vor. Die Tabelle **Error and drop counters**
  zeigt alle Zähler aus `netstat -s` und `netstat -m`, die im Zeitraum
  gewachsen sind und deren Beschreibung nach Fehler, Drop, Verwerfen,
  Prüfsumme, Fragment, Wiederholung, Timeout oder Ablehnung klingt, mit der
  Beschreibung, die netstat selbst ausgibt (z. B. `tcp: data packets
  retransmitted`). Die Kennzahlen darüber werden aus denselben Zählern über
  Teile ihrer Beschreibung gebildet.
- **OpenVPN.** Siehe [OpenVPN-Sitzungen](#openvpn-sitzungen). Die Dauer
  einer Verbindung wird stundengenau auf die Zeitabschnitte verteilt; noch
  offene Verbindungen werden bei jedem Lauf bis zum aktuellen Zeitpunkt
  mitgezählt.
- **Fehlgeschlagene Anmeldungen** (Kachel) ist die Summe aus WebGUI, SSH und
  OpenVPN.
- **Konfigurationsänderungen.** Jede gespeicherte Revision ist eine
  Änderung. Rückwirkend reicht das nur so weit, wie OPNsense Sicherungen
  aufhebt (**System → Configuration → History**, Standard 60); ab der
  Installation des Plugins wird jede Revision im Cache festgehalten.
  Nicht gezählt werden Revisionen, die das System selbst schreibt und
  keine Änderung eines Administrators sind: die Modell-Migrationen nach
  einem Firmware-Update oder einer Paketinstallation (Beschreibung
  `/usr/local/opnsense/mvc/script/run_migrations.php made changes`).
  Schon gezählte Migrationen nimmt das Plugin beim ersten Lauf nach dem
  Update einmalig wieder aus dem Cache.

## Zeiträume

| Auswahl | Fenster |
| --- | --- |
| Last 24 hours | jetzt minus 24 Stunden bis jetzt |
| Yesterday | gestern 00:00 bis heute 00:00 (Ortszeit) |
| Last 7 days | jetzt minus 7 Tage bis jetzt |
| Last 7 full days | die 7 Kalendertage vor heute |
| Last 30 days | jetzt minus 30 Tage bis jetzt |
| Last month | der vorige Kalendermonat |

Die Cache-Daten liegen für die letzten drei Tage stundengenau, davor
tageweise vor. Ein Fenster, das mitten an einem älteren Tag beginnt, nimmt
diesen Tag ganz, wenn seine Mitte im Fenster liegt, sonst gar nicht.

## Der Auswertungs-Cache

Jede Logzeile wird genau einmal gelesen und in den Stundenabschnitt
einsortiert, in dem sie geschrieben wurde. Ein Abschnitt hält je Quelle
Summen, Zähler je Adresse, Benutzer, Dienst usw. und eine kurze Liste von
Ereignissen. Stundenabschnitte, die älter als drei Tage sind, werden zu
Tagesabschnitten zusammengefasst; ein 30-Tage-Bericht liest also rund 30
kleine Dateien statt 30 Tage Logs.

| Pfad | Inhalt |
| --- | --- |
| `/var/cache/reports/hours/<epoch>.json` | Stundenabschnitte |
| `/var/cache/reports/days/<epoch>.json` | Tagesabschnitte |
| `/var/cache/reports/sessions/<epoch>.json` | beendete OpenVPN-Verbindungen, eine Datei je Tag des Verbindungsbeginns |
| `/var/cache/reports/state.json` | Lesestand je Logdatei, offene OpenVPN-Verbindungen mit letztem Zählerstand, letzte netstat- und CPU-Probe, gesehene Konfigurationsrevisionen |
| `/var/cache/reports/mail.json` | Stand des Mailversands |
| `/var/cache/reports/secret` | Schlüssel der Pseudonymisierung, nur für root lesbar |
| `/var/cache/reports/mail/` | zuletzt verschickter Bericht als HTML und Text |

- **Aktualisierung.** Der Cron-Eintrag `/usr/local/etc/cron.d/reports`
  liest alle 15 Minuten nach. Öffnet jemand die Seite, holt sie bis zu
  32 MB neuer Logdaten selbst nach; bei mehr übernimmt der Cron-Lauf, und
  die Seite weist darauf hin.
- **Logdateien** werden über Gerät und Inode wiedererkannt, nicht über den
  Namen. Eine rotierte Datei (`eve.json` → `eve.json.0`) wird an der Stelle
  fortgesetzt, an der sie verlassen wurde. Komprimiert newsyslog eine Datei
  gleich beim Rotieren (wie das Squid-`access.log` um Mitternacht), wird
  die komprimierte Generation am Inhalt ihres Anfangs wiedererkannt und ab
  der gemerkten Stelle weitergelesen, so fehlen auch die letzten Minuten
  vor der Rotation nicht. Andere komprimierte Generationen werden nicht
  gelesen; gzip, bzip2 und xz werden erkannt.
- **Erster Lauf.** Dateien, die seit Beginn der Aufbewahrungsdauer nicht mehr
  geschrieben wurden, werden übersprungen; alles Jüngere wird nachgelesen.
  Bei großen Logs kann das einige Minuten dauern.
- **Obergrenzen.** Ein Zähler behält je Stunde die 2000, je Tag die 5000
  größten Einträge. Die Top-Listen bleiben dadurch exakt, die Kennzahl
  „Anzahl Quellen“ kann bei sehr vielen Quellen nach oben begrenzt sein.
- **Löschen.** Der Cache darf jederzeit vollständig gelöscht werden
  (`rm -rf /var/cache/reports`). Beim nächsten Lauf wird er aus den noch
  vorhandenen Logs neu aufgebaut. Verloren sind dann die Datenmengen
  beendeter OpenVPN-Verbindungen, denn die stehen in keinem Log. Bei der Deinstallation bleibt er liegen,
  weil pkg ein Update nicht zuverlässig von einer Deinstallation
  unterscheidet; danach kann er von Hand entfernt werden.

## Einstellungen

Reiter **Settings**, gespeichert mit **Save**. Ein eigenes Apply gibt es
nicht, das Backend liest die Einstellungen bei jedem Lauf aus der
Konfiguration.

| Feld | Bedeutung |
| --- | --- |
| Collect statistics | Logs alle 15 Minuten in den Cache lesen und die Interface-Zähler abfragen. Ausgeschaltet zeigt der Bericht nur noch Live-Werte (NetFlow, DNS, WireGuard, Systemzustand). |
| Keep statistics (days) | Aufbewahrung des Caches, 7 bis 400 Tage, Standard 90. Die Logs selbst behalten ihre eigene Aufbewahrung unter **System → Settings → Logging**. |
| Report language | Sprache des Berichts und der Mail, Englisch oder Deutsch. Die Bedienelemente der Seite folgen der Sprache der GUI. |
| Rows per top list | 5, 10 oder 20 Zeilen je Liste |
| Sections | Abschnitte, die Seite und Mail enthalten |
| Pseudonymise users and internal hosts | siehe [Pseudonymisierung](#pseudonymisierung) |
| Send report by email | Mailversand nach Zeitplan einschalten |
| Recipients | eine oder mehrere Adressen, durch Kommas getrennt |
| Frequency | Daily (Vortag), Weekly (die sieben Tage vor dem gewählten Wochentag), Monthly (Vormonat) |
| Send at hour | Stunde (0–23, Ortszeit), ab der der Bericht verschickt wird |
| Weekday | Wochentag des Wochenberichts |

## Bericht per E-Mail

Der Cron-Eintrag ruft jede Stunde zur Minute 7 `configctl reports mail`
auf. Ein Bericht ist fällig, wenn sein Zeitraum abgeschlossen ist, die
eingestellte Stunde erreicht ist und er für genau diesen Zeitraum noch nicht
erfolgreich verschickt wurde. Schlägt der Versand fehl, wird es stündlich
erneut versucht, bis zu drei Tage nach Ende des Zeitraums. War die Firewall
zur fälligen Stunde aus, kommt der Bericht also nach.

Vor dem Versand liest das Plugin die Logs auf den neuesten Stand, damit der
Vortag vollständig ist. Der Bericht geht als HTML mit einer Textfassung über
die Aktion `smtpcli sendhtml` von `os-smtp-cli-maxit` an jeden Empfänger
einzeln. Das HTML verwendet nur Tabellen und Inline-Styles; es sieht in
Outlook, Thunderbird und gängigen Webmailern gleich aus, Diagramme sind als
Tabellenbalken gezeichnet.

**Send report now** auf dem Reiter Settings speichert zuerst das Formular und
verschickt dann sofort den Bericht des zuletzt abgeschlossenen Zeitraums der
eingestellten Frequenz. Unter dem Knopf steht, wann und mit welchem Ergebnis
zuletzt versucht wurde zu senden.

## Pseudonymisierung

Mit **Pseudonymise users and internal hosts** ersetzt der Bericht
Benutzernamen (OpenVPN, Webproxy, WireGuard-Peers) und interne Adressen
(privat und link-lokal, also auch die Tunneladressen) in allen Listen und
im Reiter VPN sessions durch ein Pseudonym wie
`host-3f9a12c0` bzw. `user-…`. Das Pseudonym ist ein HMAC mit einem
Schlüssel, der nur auf dieser Firewall liegt: Derselbe Benutzer behält
dasselbe Pseudonym, Ranglisten bleiben lesbar, ohne Namen zu nennen, und
ohne den Schlüssel lässt sich das Pseudonym nicht zurückrechnen.

Nicht pseudonymisiert werden externe Adressen (Angreifer, Gegenstellen,
Websites) und Administratorkonten in den Abschnitten Administrativer Zugriff
und Konfigurationsänderungen, weil dort gerade die Zuordnung der Zweck ist.

Wer bestimmte Auswertungen gar nicht erheben will, wählt die Abschnitte
unter **Sections** ab. Der Cache sammelt dann weiterhin; nur Seite und Mail
zeigen die Abschnitte nicht. Soll gar nicht gesammelt werden, **Collect
statistics** ausschalten und den Cache löschen.

## API und Kommandozeile

| Endpunkt | Methode | Zweck |
| --- | --- | --- |
| `/api/reports/stats/report` | POST `{"range": "24h"}` | Bericht als JSON |
| `/api/reports/stats/preview` | POST `{"range": "24h"}` | Bericht als HTML-Mail, `{"status": "ok", "html": "…"}` |
| `/api/reports/stats/sessions` | POST `{"range": "24h"}` | alle OpenVPN-Verbindungen des Zeitraums als JSON, dazu je Benutzer Verbindungszeit, Upload, Download und Fehlversuche je Stunde bzw. Tag (`series`) |
| `/api/reports/settings/get`, `/set` | GET / POST | Einstellungen |
| `/api/reports/service/sendnow` | POST | Bericht jetzt verschicken |
| `/api/reports/service/mailstatus` | GET | Stand des Mailversands |

Auf der Kommandozeile:

```sh
configctl reports ingest          # Cache aktualisieren
configctl reports stats 7d        # Bericht als JSON
configctl reports preview yesterday > /tmp/report.html
configctl reports sessions 7d     # OpenVPN-Verbindungen als JSON
configctl reports mailnow         # Bericht jetzt verschicken
configctl reports mailstatus

# direkt, ohne configd
/usr/local/opnsense/scripts/OPNsense/Reports/report.py --stats 24h
```

`range` ist einer der Werte `24h`, `yesterday`, `7d`, `lastweek`, `30d`,
`lastmonth`.

## Berechtigungen

Zwei ACLs, zuweisbar unter **System → Zugriff → Gruppen**:

| Berechtigung | Muster | Zweck |
| --- | --- | --- |
| Reporting: Executive Report | `ui/reports/*`, `api/reports/stats/*` | Bericht, OpenVPN-Verbindungen und Mailvorschau ansehen |
| Reporting: Executive Report: Settings | `api/reports/settings/*`, `api/reports/service/*` | Einstellungen ändern, Bericht verschicken |

Wer nur die erste hat, sieht den Reiter Settings, kann aber weder laden noch
speichern.

## Womit das Plugin verzahnt ist

Diese Schnittstellen der GUI und des Systems werden genutzt und sind gegen
**26.7** geprüft. Wer das Plugin auf eine neuere Version hebt, sollte hier
hinschauen, denn der Bericht liest Logformate und Ausgaben, die sich mit
einer neuen OPNsense- oder FreeBSD-Version ändern können:

| Schnittstelle | Verwendung |
| --- | --- |
| filterlog-CSV wie in `scripts/filter/read_log.py` des Core | Firewall-Abschnitt |
| `/tmp/rules.debug`, `label "<rid>" … # <Beschreibung>` | Namen der blockierenden Regeln |
| `scripts/netflow/get_top_usage.py` mit den Providern `FlowInterfaceTotals`, `FlowSourceAddrTotals`, `FlowDstPortTotals` | Netzwerknutzung |
| Suricata-EVE-Felder `alert.signature`, `alert.action`, `alert.category`, `src_ip`, `dest_ip` | IPS und Anwendungskontrolle |
| Meldungstexte von OpenVPN 2.6 (`Peer Connection Initiated`, `client-instance exiting`, `MULTI_sva: pool returned`, `MULTI: Learn`) und strongSwan (`IKE_SA … established`) | VPN |
| OpenVPN-Management-Sockets `/var/etc/openvpn/*.sock`, `status 3` mit `HEADER`-Zeile | OpenVPN-Sitzungen: Datenmengen, Tunneladressen, laufende Verbindungen |
| Textausgabe von FreeBSDs `netstat -i -b -n -d -W` (Spalten `Ipkts Ierrs Idrop Ibytes Opkts Oerrs Obytes Coll Drop`), `netstat -s`, `netstat -m` | Abschnitt Interfaces |
| Audit-Meldungen `Successful login for user`, `authentication error for`, sshd, login(1) | Administrativer Zugriff |
| `kernel: ---<<BOOT>>---`, `carp: … -> MASTER`, `GATEWAY ALARM`, pkg `upgraded:` | System-Ereignisse |
| `<revision>` in `/conf/backup/config-*.xml` | Konfigurationsänderungen |
| configd-Aktion `unbound qstats totals` | DNS |
| `/tmp/pkg_upgrade.json` der Firmware-Prüfung | verfügbare Updates |
| configd-Aktion `smtpcli sendhtml` aus `os-smtp-cli-maxit` | Mailversand |
| `/ui/js/chart.umd.min.js` | Chart.js aus dem Core für den Verlauf |
| `ApiMutableModelControllerBase`, `layout_partials/base_form`, `saveFormToEndpoint`, `mapDataToFormUI` | Reiter Settings |
| Menü `Reporting` | Menüpunkt Executive Report |

## Bekannte Grenzen

- **Keine DPI.** Anwendungen erkennt nur Appblocking anhand der
  ausgewählten Anwendungen; eine vollständige Anwendungserkennung wie bei
  Sophos gibt es auf OPNsense nicht.
- **Nur protokollierte Pakete.** Die Firewall-Zahlen hängen davon ab,
  welche Regeln protokollieren.
- **NetFlow-Auflösung.** Der Core hält NetFlow-Daten nur einen Tag
  stündlich, danach tageweise. Für ältere Zeiträume sind die Grenzen des
  Fensters auf Tage gerundet.
- **DNS-Zeitraum.** Die Unbound-Zahlen beziehen sich auf den Zeitraum, den
  das Unbound-Reporting selbst vorhält, nicht auf das gewählte Fenster. Der
  Abschnitt weist darauf hin.
- **Webproxy ohne Kategorien.** Blockiert zählt, was Squid mit `TCP_DENIED`
  ablehnt (z. B. durch `os-proxybl-maxit`). Umleitungen eines URL-Filters wie
  ufdbGuard erscheinen als normale Anfrage.
- **WireGuard ohne Verlauf.** WireGuard protokolliert nicht. Gezählt werden
  Peers, deren letzter Handshake im Zeitraum liegt; das Volumen ist der
  Zähler seit Start des Tunnels.
- **OpenVPN-Datenmenge.** Sie stammt aus der Statusabfrage alle 15 Minuten;
  was eine Verbindung in den letzten Minuten vor ihrem Ende überträgt, fehlt.
  Antwortet kein Management-Socket (anderer Pfad, Instanz ohne
  Management-Schnittstelle), bleiben Upload und Download bei 0, Beginn, Ende
  und Dauer stammen dann allein aus dem Log.
- **Interface-Zähler grob getaktet.** Die Zuwächse werden alle 15 Minuten
  gebucht; ein Fehlerschub lässt sich damit auf die Viertelstunde, im
  Bericht auf die Stunde eingrenzen, nicht genauer. Die Beschreibungen in
  `netstat -s` ändern sich gelegentlich zwischen FreeBSD-Versionen; ein
  Zähler mit neuer Beschreibung beginnt dann als neuer Eintrag.
- **Verwaltungs-Konten.** WebGUI-Anmeldungen über API-Schlüssel erscheinen
  nicht als Anmeldung.

## Support / Fehlerbehebung

- **Ein Abschnitt meldet „nicht verfügbar“.** Die Meldung nennt die
  fehlende Datenquelle; siehe die Tabelle unter
  [Voraussetzungen](#voraussetzungen).
- **Alle Zahlen bleiben bei 0.** Prüfen, ob der Cache wächst:
  `ls -l /var/cache/reports/hours/`. Von Hand nachlesen mit
  `configctl reports ingest`. Läuft der Cron-Eintrag nicht, liegt das meist an
  einem nicht neu gestarteten configd: `service configd restart`.
- **Fehlermeldung beim Laden.** Das Skript direkt aufrufen, die Ausgabe ist
  reines JSON:
  `/usr/local/opnsense/scripts/OPNsense/Reports/report.py --stats 24h`
- **Die Mail kommt nicht an.** Unter dem Knopf **Send report now** steht das
  Ergebnis des letzten Versuchs. Zuerst die Testmail unter **Services →
  SMTP-Send** prüfen. Meldet der Bericht „os-smtp-cli-maxit is too old“, ist
  dort eine Version vor 1.2 installiert, der die Aktion `sendhtml` fehlt;
  das Plugin aktualisieren.
- **OpenVPN-Verbindungen ohne Datenmenge.** Prüfen, ob die
  Management-Sockets da sind und antworten:
  `ls /var/etc/openvpn/*.sock` und
  `printf 'status 3\nquit\n' | nc -U /var/etc/openvpn/<instanz>.sock`.
  Die Ausgabe muss `CLIENT_LIST`-Zeilen enthalten.
- **Zahlen wirken falsch.** Den Cache löschen und neu aufbauen lassen:
  `rm -rf /var/cache/reports && configctl reports ingest`.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr`
  dieses Verzeichnisses dokumentiert.
