# m.a.x. it Professional Plugins für OPNsense

Die **Professional Plugins** der
[m.a.x. Informationstechnologie AG](https://www.max-it.de) erweitern OPNsense
um Funktionen, die in der freien Distribution nicht enthalten sind: zentrale
Verwaltung mehrerer Firewalls, Anwendungsblockierung inklusive Auswertung,
Apache als Reverse Proxy mit Web Application Firewall, DNS-Filterung,
Blocklisten für den Webproxy, TLS-Aufbruch, Netzwerkscans, Mailversand für
Reports und einiges mehr.

Verteilt werden die Plugins über ein eigenes Firmware-Repository
(`os-maxit`), das sich in jeder OPNsense-Installation hinterlegen lässt.
Installation, Updates und Deinstallation laufen danach vollständig über die
gewohnten OPNsense-Bordmittel unter **System → Firmware**. Informationen zum
Service, zum Bezug der Zugangsdaten sowie Video-Anleitungen finden sich unter
**[opnsense.max-it.de](https://opnsense.max-it.de)**.

> **Zu diesem Repository:** Hier liegt ausschließlich die
> Endanwender-Dokumentation der Plugins – kein Quellcode. Die Handbücher sind
> exakte Kopien der READMEs, die mit den jeweiligen Plugins ausgeliefert
> werden, damit sie sich vor einer Installation lesen und verlinken lassen.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [os-maxit Repository einrichten](#os-maxit-repository-einrichten)
3. [Plugins installieren](#plugins-installieren)
4. [Plugins und Handbücher](#plugins-und-handbücher)
5. [Aktualisierung und Deinstallation](#aktualisierung-und-deinstallation)
6. [Support und Kontakt](#support-und-kontakt)

---

## Voraussetzungen

- Eine laufende OPNsense-Installation mit Internetzugang für den
  Paketabruf (HTTPS).
- Administrativer Zugriff auf die OPNsense-WebGUI.
- Zugangsdaten (Benutzername/Passwort) für das `os-maxit`-Repository. Diese
  gehören zum Professional-Angebot und werden von m.a.x. it bereitgestellt –
  siehe [Support und Kontakt](#support-und-kontakt).

Einzelne Plugins haben zusätzliche Anforderungen (z. B. eine dedizierte
Instanz oder ein bereits eingerichtetes Intrusion-Detection-System). Diese
stehen jeweils im Abschnitt *Voraussetzungen* des betreffenden Handbuchs.

## os-maxit Repository einrichten

Dieser Schritt ist einmalig pro Firewall nötig. Sind bereits Plugins von
m.a.x. it installiert, ist das Repository in der Regel schon eingerichtet.

1. Zugangsdaten von m.a.x. it anfragen bzw. aus dem Kunden-/Support-Bereich
   entnehmen.
2. Eine ausführliche Video-Anleitung zur Einrichtung stellt m.a.x. it hier
   bereit:
   [opnsense.max-it.de/professional-content/freie-videos](https://opnsense.max-it.de/professional-content/freie-videos/)
3. In der WebGUI unter **System → Firmware → Einstellungen** das
   `os-maxit`-Repository mit den erhaltenen Zugangsdaten hinterlegen.
4. Anschließend unter **System → Firmware → Aktualisierungen** auf
   **Nach Updates suchen** klicken, damit OPNsense die Paketlisten des neuen
   Repositories abruft.

Danach erscheinen die Professional Plugins unter **System → Firmware →
Plugins** in der Liste der verfügbaren Pakete.

## Plugins installieren

1. **System → Firmware → Plugins** öffnen.
2. Das gewünschte Paket suchen (die Paketnamen stehen in der Tabelle unten,
   z. B. `os-apache-maxit`) und über das **+** installieren.
3. Nach der Installation die WebGUI-Seite neu laden – die Menüeinträge der
   Plugins erscheinen erst dann.

Abhängigkeiten (FreeBSD-Pakete wie `nmap`, `smtp-cli`, `apache24`, `sslproxy`
oder das Plugin `os-squid`) zieht OPNsense automatisch mit.

## Plugins und Handbücher

Für jedes der folgenden Plugins existiert ein vollständiges deutsches Handbuch
(Voraussetzungen, Einrichtung, Feldreferenz, Fehlersuche):

| Plugin | Paket | Beschreibung | Handbuch |
| --- | --- | --- | --- |
| Central Management | `os-cm-ansible` | Zentrale Verwaltung mehrerer OPNsense-Firewalls: Regeln, Aliase, Captive Portal, Zertifikate, Benutzer, Updates und Backups an einer Stelle – per Ansible/API verteilt. Läuft auf einer dedizierten Management-Instanz. | [os-cm-ansible.md](os-cm-ansible.md) |
| App Blocking | `os-appblocking-maxit` | Fertige Suricata-Regelsätze für bekannte Anwendungen (ChatGPT, Netflix, TikTok, Steam, öffentliche DoH/DoT-Resolver u. v. m.), kategorieweise auswählbar, plus die Auswertungsseite **Analyze Threats**. | [os-appblocking-maxit.md](os-appblocking-maxit.md) |
| Apache | `os-apache-maxit` | Apache als HTTP-Server, Reverse Proxy und Lastverteiler mit TLS-Terminierung, Security-Headern, Basic Auth, IP-ACLs und mod_security2-WAF (OWASP CRS). | [os-apache-maxit.md](os-apache-maxit.md) · [WAF im Detail](os-apache-maxit-waf.md) |
| AdGuard Home | `os-adguardhome-maxit` | AdGuard Home als netzwerkweiter DNS-Filter (Werbung, Tracker, Malware-Domains, Parental Control) mit eigener Weboberfläche, als OPNsense-Dienst eingebunden. | [os-adguardhome-maxit.md](os-adguardhome-maxit.md) |
| Squid Blocklist | `os-proxybl-maxit` | Kategorieweise Blocklisten (Malware, Phishing, Ransomware, Werbung, Betrug, Piraterie) für den Squid-Webproxy, zugeordnet über Netzgruppen und Policies inkl. Whitelists. | [os-proxybl-maxit.md](os-proxybl-maxit.md) |
| SSLproxy | `os-sslproxy-maxit` | Transparenter TLS-Aufbruch (Nachfolger von SSLsplit) mit vollständiger Optionsabdeckung: Protokoll-Listener, Zertifikatsbehandlung, Benutzerauthentifizierung, Spiegelung auf ein IDS und Mitschnitt (Content-/PCAP-Log). | [os-sslproxy-maxit.md](os-sslproxy-maxit.md) |
| Nmap | `os-nmap` | Nmap-Scans (Ping/SYN/UDP sowie NSE-Schwachstellenscans) direkt aus der WebGUI, asynchron mit Live-Ausgabe und optionalem E-Mail-Bericht. | [os-nmap.md](os-nmap.md) |
| SMTP-Send | `os-smtp-cli-maxit` | Ausgehender SMTP-Relay (Smarthost, STARTTLS/TLS, Auth) ohne lokalen Mailserver. Andere Plugins versenden ihre Reports darüber. | [os-smtp-cli-maxit.md](os-smtp-cli-maxit.md) |

Das Apache-Handbuch liegt zusätzlich auf Englisch vor:
[os-apache-maxit.en.md](os-apache-maxit.en.md) und
[os-apache-maxit-waf.en.md](os-apache-maxit-waf.en.md).

Über das `os-maxit`-Repository werden darüber hinaus weitere Plugins
ausgeliefert. Bei Bedarf an Dokumentation dazu genügt eine kurze Nachricht –
siehe [Support und Kontakt](#support-und-kontakt).

## Aktualisierung und Deinstallation

- **Updates:** Neue Plugin-Versionen kommen über **System → Firmware →
  Aktualisierungen** wie jedes andere OPNsense-Update. Ein separater
  Update-Weg pro Plugin existiert nicht.
- **Deinstallation:** Unter **System → Firmware → Plugins** über das
  Papierkorb-Symbol. Die Konfiguration eines Plugins bleibt in der
  `config.xml` erhalten, sodass eine Neuinstallation die Einstellungen
  wiederfindet.

## Support und Kontakt

m.a.x. Informationstechnologie AG

- Website: [www.max-it.de](https://www.max-it.de)
- Professional Plugins, Zugangsdaten und Videos:
  [opnsense.max-it.de](https://opnsense.max-it.de)

Für Supportanfragen zu einem Plugin sind folgende Angaben hilfreich: Name und
Version des Plugins (**System → Firmware → Plugins**), die OPNsense-Version
sowie die relevanten Log-Auszüge (**System → Protokolldateien** bzw. der
Log-Viewer des jeweiligen Plugins).
