# os-nmap – Nmap-Netzwerkscanner für OPNsense

`os-nmap` ist ein Plugin der m.a.x. Informationstechnologie AG, mit dem sich
Nmap-Scans (Ping/SYN/UDP sowie NSE-Script-basierte Schwachstellenscans)
direkt aus der OPNsense-WebGUI heraus starten lassen – ohne SSH-Zugriff auf
die Firewall. Der Scan läuft im Hintergrund, die Ausgabe wird live in der
GUI nachgeladen und kann optional per E-Mail als Bericht verschickt werden.

> **Wichtiger Hinweis:** Portscans (insbesondere mit NSE-Scripts der
> Kategorie `vuln`) sollten nur gegen Netze/Hosts durchgeführt werden, für
> die eine ausdrückliche Erlaubnis besteht. Ungefragtes Scannen fremder
> Netze kann rechtliche Konsequenzen haben.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Bedienung](#bedienung)
4. [Felder der Konfigurationsseite](#felder-der-konfigurationsseite)
5. [Scan starten, stoppen, Ausgabe leeren](#scan-starten-stoppen-ausgabe-leeren)
6. [E-Mail-Report](#e-mail-report)
7. [Berechtigungen](#berechtigungen)
8. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über
  das `os-nmap` bezogen wird (das Paket `nmap` wird als Abhängigkeit
  automatisch mitinstalliert).
- Für den optionalen [E-Mail-Report](#e-mail-report): das Plugin
  `os-smtp-cli-maxit` installiert, aktiviert und mit einem funktionierenden
  SMTP-Relay konfiguriert (siehe dessen eigenes README).

## Installation

1. Unter **System → Firmware → Plugins** nach `os-nmap` suchen und
   installieren.
2. In der linken Navigation erscheint unter **Interfaces → Diagnostics**
   der neue Menüpunkt **Nmap**.

## Bedienung

Die Seite **Interfaces → Diagnostics → Nmap** enthält eine einzige
Ansicht mit dem Konfigurationsformular, den Aktions-Buttons (Scan / Stop /
Clear) und der Ausgabebox darunter. Es gibt keinen separaten
"Speichern"-Schritt: Ein Klick auf **Scan** speichert die aktuelle
Formularbelegung und startet anschließend den Scan.

## Felder der Konfigurationsseite

| Feld | Beschreibung |
|---|---|
| IP | Ziel des Scans: einzelne IP-Adresse oder Netz in CIDR-Schreibweise (z. B. `10.0.0.0/24`). |
| ScanType | `Pingscan` (`-sn`, nur Host-Discovery ohne Portscan), `SYNScan` (`-sS`, Standard) oder `UDPscan` (`-sU`). |
| Show only open ports | Entspricht `--open`: blendet Ports ohne "open"-Status aus der Ausgabe aus. |
| Verbose | Entspricht `-v`. |
| Port range | Optional, nmap-Syntax für Portbereiche (z. B. `1-1024` oder `22,80,443`). Leer lassen, um den nmap-Standardportsatz zu scannen. |
| NSE script categories | Auswahl mehrerer NSE-Scriptkategorien (`--script`): `vuln` (Schwachstellenscans), `safe` (unkritische Scripts), `discovery`, `auth`. Es sind bewusst nur unkritische Kategorien wählbar – destruktive Kategorien wie `dos`, `brute` oder `exploit` stehen hier nicht zur Verfügung. |
| NSE script arguments | Optionaler `--script-args`-Wert (z. B. `vulns.showall`). Erlaubt sind nur Buchstaben, Ziffern sowie `, = : _ . -` und Leerzeichen. |
| Email report when finished | Siehe [E-Mail-Report](#e-mail-report). |
| Report recipient | Empfängeradresse für den E-Mail-Report. |

## Scan starten, stoppen, Ausgabe leeren

- **Scan** – speichert das Formular und startet den Scan asynchron im
  Hintergrund. Die Ausgabebox pollt alle 3 Sekunden den aktuellen Stand
  und zeigt die nmap-Ausgabe live an, sobald Zeilen geschrieben werden.
  Während ein Scan läuft, ist der Button deaktiviert (erkennbar am
  Spinner-Icon); es kann immer nur ein Scan gleichzeitig laufen.
- **Stop** – bricht einen laufenden Scan ab (beendet den nmap-Prozess).
  Der Button dient gleichzeitig als Reset: Sollte ein vorheriger Scan
  hängen geblieben oder sein Prozess anderweitig verschwunden sein, setzt
  ein Klick auf Stop den internen Status trotzdem zurück, sodass ein neuer
  Scan gestartet werden kann.
- **Clear** – leert nur die Anzeige in der Ausgabebox (rein clientseitig,
  ohne Einfluss auf einen eventuell noch laufenden Scan).

Ein Neuladen der Seite während ein Scan läuft ist unproblematisch: Beim
Laden wird der aktuelle Stand einmalig abgefragt, ein laufender Scan wird
dabei automatisch wieder erkannt und das Live-Polling fortgesetzt.

## E-Mail-Report

Ist **Email report when finished** aktiviert und ein **Report recipient**
eingetragen, wird nach Abschluss des Scans automatisch eine E-Mail
verschickt:

- Die vollständige Scan-Ausgabe wird als `.txt`-Anhang beigefügt (nicht in
  den Mailtext eingebettet – manche Mailclients, insbesondere Outlook im
  Dark Mode, stellen sehr lange Klartext-Mailbodies fehlerhaft dar). Der
  Mailtext selbst enthält nur eine kurze Zusammenfassung.
- Der Versand erfolgt über die generische `send`-Aktion des Plugins
  `os-smtp-cli-maxit`. Ist dieses Plugin nicht installiert oder nicht
  konfiguriert, schlägt nur der Mailversand fehl – der Scan selbst wird
  davon nicht beeinträchtigt; ein entsprechender Fehlschlag wird als
  zusätzliche Zeile am Ende der Ausgabebox vermerkt.

## Berechtigungen

Alle Nmap-Seiten und die zugehörige API liegen unter den Mustern
`ui/nmap/*` bzw. `api/nmap/*`. Damit weitere Benutzer/Benutzergruppen
(neben `root`) darauf zugreifen dürfen, muss ihnen unter **System →
Zugriff → Gruppen** die Berechtigung **"Services: Nmap"** zugewiesen
werden.

## Support / Fehlerbehebung

- Ein hängender oder nicht mehr reagierender Scan lässt sich über den
  [Stop-Button](#scan-starten-stoppen-ausgabe-leeren) beenden bzw.
  zurücksetzen.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr`
  dieses Verzeichnisses dokumentiert.
