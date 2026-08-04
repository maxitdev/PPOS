# os-proxybl-maxit – Blocklisten für den Squid-Webproxy

`os-proxybl-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG, das
den Squid-Webproxy von OPNsense um gepflegte, kategorieweise auswählbare
Domain-Blocklisten erweitert (Malware, Phishing, Ransomware, Werbung, Betrug,
Piraterie u. a.). Die Listen werden von der Firewall selbst heruntergeladen und
über frei definierbare **Access Lists** (Netzgruppen) und **Policies**
(Kategorien + Ausnahmen) den Clients zugeordnet.

Anders als eine reine "alles oder nichts"-Lösung erlaubt das Plugin damit
unterschiedliche Filterstufen pro Netz: z. B. nur Malware-Kategorien für
Serversegmente, zusätzlich Werbung und Piraterie für Gäste-WLANs, und ein
komplett gesperrtes Netz mit reiner Positivliste.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Funktionsweise](#funktionsweise)
4. [Einrichtung](#einrichtung)
5. [Reiter "General"](#reiter-general)
6. [Reiter "Access Lists"](#reiter-access-lists)
7. [Reiter "Policies"](#reiter-policies)
8. [Verfügbare Kategorien](#verfügbare-kategorien)
9. [Reihenfolge der erzeugten Regeln](#reihenfolge-der-erzeugten-regeln)
10. [Namensregeln](#namensregeln)
11. [Aktualisierung der Listen](#aktualisierung-der-listen)
12. [Hochverfügbarkeit (HA-Sync)](#hochverfügbarkeit-ha-sync)
13. [Dateien und Pfade](#dateien-und-pfade)
14. [Berechtigungen](#berechtigungen)
15. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über das
  `os-proxybl-maxit` bezogen wird. Das Plugin `os-squid` wird als Abhängigkeit
  automatisch mitinstalliert.
- Ein eingerichteter und aktivierter Squid-Webproxy (**Services → Squid Web
  Proxy → Administration**), den die Clients tatsächlich nutzen – entweder
  explizit (Proxy-Einstellung/WPAD) oder transparent per NAT-Umleitung.
- Internetzugang der Firewall (HTTPS) für den Download der Blocklisten.

> **Hinweis zu HTTPS:** Squid sieht bei HTTPS-Verbindungen ohne aktiviertes
> SSL-Bumping nur den `CONNECT`-Hostnamen. Die Domain-Blocklisten greifen
> dadurch auch ohne TLS-Aufbruch, jedoch nur auf Hostname-Ebene; über die
> IP-Adresse direkt angesprochene Ziele werden nur von den IP-basierten
> Kategorien erfasst.

## Installation

1. Unter **System → Firmware → Plugins** nach `os-proxybl-maxit` suchen und
   installieren.
2. In der linken Navigation erscheint unter **Services → Squid Web Proxy** der
   neue Menüpunkt **Blocklist**.

## Funktionsweise

1. Die configd-Aktion `proxybl fetch` lädt die Blocklisten herunter und legt sie
   als Squid-ACL-Dateien unter `/usr/local/etc/squid/acl/<kategorie>-maxit` ab.
2. Aus den Angaben der Reiter *General*, *Access Lists* und *Policies* wird die
   Datei `/usr/local/etc/squid/pre-auth/proxybl.conf` generiert. Sie enthält die
   `acl`-Definitionen (Listen und Netzgruppen) sowie die daraus abgeleiteten
   `http_access`-Regeln.
3. Squid liest diese Datei über `include /usr/local/etc/squid/pre-auth/*.conf`
   ein – und zwar **vor** seinem eigenen Regelwerk und vor der
   Proxy-Authentifizierung.
4. Ein Klick auf **Save** führt alle drei Schritte aus: Listen herunterladen,
   Konfiguration schreiben, Squid neu konfigurieren.

Aus Schritt 3 folgt ein wichtiger Punkt: Da Squid `http_access`-Regeln von oben
nach unten auswertet und bei der ersten passenden Regel abbricht, entscheiden
die Regeln dieses Plugins **vor** allen Squid-eigenen Regeln. Eine Policy ohne
*Default Deny* erzeugt am Ende ein `http_access allow` für ihre Netzgruppe –
damit ist der Zugriff für dieses Netz bereits erlaubt, bevor Squid seine eigenen
ACLs oder eine Proxy-Authentifizierung auswerten kann. In Umgebungen mit
Proxy-Anmeldung oder weiteren Squid-Regeln ist das bei der Planung der Policies
zu berücksichtigen.

## Einrichtung

1. **Services → Squid Web Proxy → Blocklist** öffnen.
2. Im Reiter **General** die Option **Enable** aktivieren und **Save** klicken.
   Der erste Download der Listen kann einige Sekunden dauern.
3. In den Reiter **Access Lists** wechseln und über **+** eine Netzgruppe
   anlegen: Name (z. B. `Gaeste`) und die zugehörigen Netze in
   CIDR-Schreibweise. Speichern.
4. In den Reiter **Policies** wechseln und über **+** eine Policy anlegen:
   Beschreibung, gewünschte Kategorien, optional Ausnahmen (Whitelist) und die
   eben erstellte Access List. Speichern.
5. Auf einem Client eine gesperrte Domain aufrufen – Squid muss die Anfrage mit
   einer Fehlerseite ("Access Denied") ablehnen.
6. Unter **System → Einstellungen → Cron** einen täglichen Job für die Aktion
   **"Download Proxybl Blocklist"** anlegen (siehe
   [Aktualisierung der Listen](#aktualisierung-der-listen)).

## Reiter "General"

| Feld | Beschreibung |
|---|---|
| Enable | Aktiviert bzw. deaktiviert das gesamte Regelwerk. Bei deaktivierter Option wird eine leere `proxybl.conf` erzeugt – Access Lists und Policies bleiben gespeichert, greifen aber nicht. |

Der Button **Save** in diesem Reiter (wie auch in den beiden anderen) löst
jeweils den Listendownload, die Neuerzeugung der Konfiguration und eine
Neukonfiguration des Squid-Dienstes aus.

## Reiter "Access Lists"

Eine Access List ist eine benannte Gruppe von Quellnetzen. Sie wird zu einer
Squid-ACL vom Typ `src` und in den Policies referenziert.

| Feld | Beschreibung |
|---|---|
| Enabled | Schaltet die Access List aktiv/inaktiv. Inaktive Listen werden nicht in die Konfiguration geschrieben – Policies, die darauf verweisen, verlieren damit ihre Wirkung. |
| Description | Name der Access List. Wird unverändert als Squid-ACL-Name verwendet, daher sind nur Buchstaben, Ziffern, Punkt, Bindestrich und Unterstrich erlaubt (max. 64 Zeichen). |
| Networks | Ein oder mehrere Netze/Adressen in CIDR-Schreibweise (z. B. `192.168.1.0/24`, `10.0.5.17/32`). Mehrere Einträge werden im Feld einzeln hinzugefügt. |
| Disable Logging | Schreibt `access_log none` für diese Netzgruppe – Zugriffe dieser Clients erscheinen dann nicht im Squid-Zugriffsprotokoll. Nützlich für Netze, die aus Datenschutz- oder Volumengründen nicht protokolliert werden sollen. |

## Reiter "Policies"

Eine Policy verknüpft eine Access List mit den zu sperrenden Kategorien und den
Ausnahmen.

| Feld | Beschreibung |
|---|---|
| Enabled | Schaltet die Policy aktiv/inaktiv. |
| Description | Name der Policy. Wird zur Bildung des Whitelist-ACL-Namens verwendet (`<Description>wl`) und daher ohne Leerzeichen, Umlaute oder Sonderzeichen angeben – siehe [Namensregeln](#namensregeln). |
| Default Deny | Sperrt **alle** Domains für die zugeordnete Access List; erlaubt bleiben nur die Einträge der Whitelist. Damit lässt sich ein Netz mit reiner Positivliste betreiben. |
| Category | Auswahl der zu sperrenden Kategorien (Mehrfachauswahl). Siehe [Verfügbare Kategorien](#verfügbare-kategorien). Der Eintrag `None` bedeutet: keine Kategoriesperre (sinnvoll in Kombination mit *Default Deny*). |
| Whitelist | Domains, die für diese Policy immer erlaubt sind – auch wenn sie in einer gesperrten Kategorie stehen. Es gilt die Squid-Syntax für `dstdomain`: `.example.com` erfasst die Domain samt aller Unterdomains, `www.example.com` nur diesen Host. |
| Acl | Die Access List (Netzgruppe), für die diese Policy gilt. Pflichtangabe. |

Mehrere Policies dürfen dieselbe Access List verwenden; ihre Regeln werden dann
in der unten beschriebenen Reihenfolge zusammengeführt.

## Verfügbare Kategorien

| Kategorie | Quelle | Art der Einträge |
|---|---|---|
| URLHaus | `urlhaus.abuse.ch` | Hostnamen aus aktuell verteilten Malware-URLs |
| ThreatFox | `threatfox.abuse.ch` | Hostnamen aus Indicators of Compromise |
| FeodoTracker | `feodotracker.abuse.ch` | IP-Adressen von Botnet-C2-Servern |
| SSLBL | `sslbl.abuse.ch` | IP-Adressen von C2-Servern mit auffälligen TLS-Zertifikaten |
| Ads | `blocklistproject.github.io` | Werbe- und Trackingdomains |
| Fraud | `blocklistproject.github.io` | Betrugsdomains |
| Phishing | `blocklistproject.github.io` | Phishing-Domains |
| Piracy | `blocklistproject.github.io` | Domains von Piraterie-/Streaming-Portalen |
| Ransomware | `blocklistproject.github.io` | Ransomware-bezogene Domains |

> **Wichtig:** Die Auswahlliste im Policy-Dialog enthält darüber hinaus die
> Einträge **DoH**, **MalwareHosts**, **Phishtank**, **Openphish** und
> **Scam**. Für diese Kategorien wird derzeit keine Liste heruntergeladen; ihre
> Auswahl führt zu einer `http_access`-Regel, die auf eine nicht existierende
> ACL verweist, und damit zu einer Squid-Konfiguration, die nicht mehr geladen
> werden kann. Diese fünf Einträge sind daher nicht zu verwenden. Für das
> Sperren öffentlicher DoH-/DoT-Resolver steht die entsprechende Kategorie im
> Plugin `os-appblocking-maxit` zur Verfügung.

Die beiden IP-basierten Kategorien (*FeodoTracker*, *SSLBL*) greifen nur, wenn
ein Client die Zieladresse direkt als IP anspricht – für Domain-Aufrufe sind die
domainbasierten Kategorien zuständig.

## Reihenfolge der erzeugten Regeln

Die generierte `proxybl.conf` ist immer nach demselben Muster aufgebaut. Die
Reihenfolge ist entscheidend, weil Squid bei der ersten passenden Regel
abbricht:

1. **ACL-Definitionen** der Kategorielisten (`dstdomain`).
2. **ACL-Definitionen** der Access Lists (`src`) sowie `access_log none` für
   Netzgruppen mit deaktivierter Protokollierung.
3. **ACL-Definitionen** der Whitelists aller Policies (`dstdomain`).
4. **Alle Allow-Regeln der Whitelists** – `http_access allow <Netzgruppe>
   <Policy>wl`.
5. **Alle Deny-Regeln der Kategorien** – `http_access deny <Netzgruppe>
   remoteblacklist_<Kategorie>`.
6. **Alle Default-Deny-Regeln** – `http_access deny <Netzgruppe>` für Policies
   mit aktiviertem *Default Deny*.
7. **Alle Default-Allow-Regeln** – `http_access allow <Netzgruppe>` für Policies
   ohne *Default Deny*.

Daraus folgt: Whitelist-Einträge gewinnen immer gegen Kategoriesperren (Punkt 4
vor Punkt 5), und der abschließende Allow (Punkt 7) beendet die Auswertung für
das betreffende Netz – nachfolgende Squid-Regeln, einschließlich
Proxy-Authentifizierung, greifen für dieses Netz dann nicht mehr.

## Namensregeln

Beschreibungen werden unverändert zu Squid-ACL-Namen. Für Access Lists erzwingt
die Eingabemaske bereits zulässige Zeichen. Bei **Policies** findet keine solche
Prüfung statt, der Name wird aber ebenfalls verwendet (als `<Description>wl`).
Daher gilt für Policy-Namen:

- Keine Leerzeichen, keine Umlaute, keine Sonderzeichen – zulässig sind
  Buchstaben, Ziffern, Punkt, Bindestrich und Unterstrich.
- Eindeutige Namen verwenden; zwei Policies mit gleichem Namen führen zu
  vermischten Whitelist-ACLs.

Ein unzulässiger Name fällt erst beim Neuladen von Squid auf – der Dienst
verweigert dann den Start bzw. behält die alte Konfiguration.

## Aktualisierung der Listen

Der Download der Blocklisten erfolgt bei jedem Klick auf **Save** in einem der
drei Reiter. Für den laufenden Betrieb ist ein regelmäßiger Abgleich nötig:

1. **System → Einstellungen → Cron** öffnen und einen neuen Job anlegen.
2. Als Befehl **"Download Proxybl Blocklist"** wählen und einen täglichen
   Zeitpunkt außerhalb der Hauptlastzeit setzen.

> **Hinweis:** Das Plugin stellt zwei configd-Aktionen für Cron bereit –
> `proxybl fetch` (lädt die Listen herunter) und `proxybl start` (erzeugt nur die
> Konfigurationsdatei neu). Beide tragen derzeit dieselbe Beschreibung, sodass im
> Auswahlfeld zwei identisch benannte Einträge erscheinen. Ob der gewählte
> Eintrag wirklich herunterlädt, lässt sich nach dem ersten Lauf am Zeitstempel
> der Dateien unter `/usr/local/etc/squid/acl/` erkennen; andernfalls den
> jeweils anderen Eintrag verwenden.

Auf der Kommandozeile entsprechen dem:

```
configctl proxybl fetch
configctl proxybl start
```

## Hochverfügbarkeit (HA-Sync)

Das Plugin registriert sich für die Konfigurationssynchronisation eines
HA-Verbunds. Unter **System → Hochverfügbarkeit → Einstellungen** lässt sich in
der Liste der zu synchronisierenden Dienste der Eintrag **Proxybl** aktivieren –
Access Lists und Policies werden dann auf den Backup-Knoten übertragen. Die
Blocklisten selbst lädt jeder Knoten eigenständig herunter; der Cron-Job ist
daher auf beiden Knoten einzurichten.

## Dateien und Pfade

| Pfad | Inhalt |
|---|---|
| `/usr/local/etc/squid/acl/<kategorie>-maxit` | Die heruntergeladenen Blocklisten, eine Datei pro Kategorie. |
| `/usr/local/etc/squid/pre-auth/proxybl.conf` | Generierte Squid-Konfiguration (ACLs und `http_access`-Regeln). Nicht manuell bearbeiten – wird bei jedem Speichern überschrieben. |
| `/usr/local/opnsense/scripts/OPNsense/Proxybl/setup.sh` | Download-Skript der Listen. |
| `/tmp/blocklist/` | Temporäres Arbeitsverzeichnis während des Downloads. |

## Berechtigungen

Alle Seiten und die zugehörige API liegen unter den Mustern `ui/proxybl/*` bzw.
`api/proxybl/*`. Damit weitere Benutzer/Benutzergruppen (neben `root`) darauf
zugreifen dürfen, muss ihnen unter **System → Zugriff → Gruppen** die
Berechtigung **"Service: Proxy Blocklist"** zugewiesen werden. Für die Bedienung
wird zusätzlich Zugriff auf die Squid-Seiten benötigt, falls der Proxy selbst
verwaltet werden soll.

## Support / Fehlerbehebung

- **Squid startet nach dem Speichern nicht mehr:** In der Regel liegt ein
  Problem in der generierten `proxybl.conf` – etwa eine Kategorie ohne Liste
  (siehe [Verfügbare Kategorien](#verfügbare-kategorien)) oder ein Policy-Name
  mit unzulässigen Zeichen. Die Konfiguration lässt sich auf der Konsole prüfen:

  ```
  squid -k parse
  ```

  Die Fehlerausgabe nennt Datei und Zeile. Zur schnellen Wiederherstellung des
  Betriebs im Reiter *General* die Option **Enable** deaktivieren und speichern.
- **Es wird nichts geblockt:** Prüfen, ob (a) **Enable** aktiv ist, (b) die
  Quell-IP des Clients tatsächlich in einem der Netze der zugeordneten Access
  List liegt, (c) der Client wirklich über den Proxy geht, und (d) die
  Listendateien unter `/usr/local/etc/squid/acl/` vorhanden und nicht leer sind.
- **Eine benötigte Seite wird geblockt (False Positive):** Die Domain in die
  **Whitelist** der betreffenden Policy aufnehmen (mit führendem Punkt, um
  Unterdomains einzuschließen) und speichern.
- **Downloads schlagen fehl:** Das Download-Skript verwendet `fetch` mit einem
  Timeout von 5 Sekunden. Bei Firewalls hinter einem vorgeschalteten Proxy oder
  mit eingeschränktem Internetzugang müssen die Quellen (`abuse.ch`,
  `blocklistproject.github.io`) per HTTPS erreichbar sein.
- **Nach der Deinstallation blockt Squid weiter:** Die generierte Datei
  `/usr/local/etc/squid/pre-auth/proxybl.conf` und die Listen unter
  `/usr/local/etc/squid/acl/` bleiben erhalten. Beide gegebenenfalls manuell
  entfernen und Squid neu konfigurieren.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr` dieses
  Verzeichnisses dokumentiert.
