# Das apache-Plugin

*[English version](os-apache-maxit.en.md)*

Eine auf dem Apache HTTP Server basierende Alternative zum `www/nginx`-Plugin:
HTTP-Server (vhosts), Locations, Reverse Proxy / Lastverteilung,
Security-Header, Basic Auth, IP-ACLs, benutzerdefinierte Fehlerseiten, eine
mod_security2-WAF (OWASP Core Rule Set) sowie ein Log-Viewer.

Dieses README besteht aus zwei Teilen: einem **Benutzerhandbuch** (Referenz
für jede Seite und jedes Feld) und einer **Schritt-für-Schritt-Anleitung**
(ein durchgängiges Beispiel). Design-Hintergründe und Implementierungs-
hinweise für Mitwirkende finden sich weiter unten unter
[Hinweise für Entwickler](#hinweise-für-entwickler). Für eine ausführliche
Beschreibung der WAF im Speziellen (Implementierung, Fehlersuche, eigene
Regeln schreiben, Falsch-Positive), siehe **[WAF.de.md](os-apache-maxit-waf.md)**.

- [Benutzerhandbuch](#benutzerhandbuch)
  - [Voraussetzungen](#voraussetzungen)
  - [Menüübersicht](#menüübersicht)
  - [General Settings](#general-settings)
  - [Reverse Proxy](#reverse-proxy)
  - [Access](#access)
  - [WAF](#waf) -- siehe auch [WAF.de.md](os-apache-maxit-waf.md) für die ausführliche Anleitung
  - [Logs](#logs)
  - [Änderungen übernehmen](#änderungen-übernehmen)
  - [Tipps und Fallstricke](#tipps-und-fallstricke)
- [Schritt-für-Schritt-Anleitung: eine interne Anwendung über HTTPS veröffentlichen](#schritt-für-schritt-anleitung-eine-interne-anwendung-über-https-veröffentlichen)
- [Hinweise für Entwickler](#hinweise-für-entwickler)

## Benutzerhandbuch

### Voraussetzungen

Bei der Installation des Plugins (**System &rarr; Firmware &rarr; Plugins
&rarr; os-apache**) werden das Apache-HTTP-Server-Paket sowie seine
WAF-/Proxy-Module automatisch mitinstalliert (`apache24`,
`ap24-mod_security`, `ap24-mod_proxy_msrpc`). Eine
manuelle Paketinstallation ist nicht nötig.

Wer HTTPS-vhosts betreiben möchte, sollte das entsprechende Zertifikat
vorher unter **System &rarr; Trust &rarr; Certificates** importieren oder
erzeugen -- das Apache-Plugin erlaubt nur die *Auswahl* eines bestehenden
Zertifikats, es stellt selbst keines aus (das `os-acme-client`-Plugin lässt
sich problemlos daneben betreiben; siehe das Feld ACME Passthrough weiter
unten).

### Menüübersicht

Das Plugin fügt ein Menü **Services &rarr; Apache** mit folgenden Seiten
hinzu:

| Seite | Wofür |
|---|---|
| General Settings | Apache ein-/ausschalten, globales Logging/Kompression, WAF ein/aus, Worker-Tuning |
| Reverse Proxy | HTTP Servers (vhosts), Locations, Upstreams (Lastverteiler-Pools) |
| Access | Basic-Auth-Benutzer/Benutzerlisten, IP-ACLs, Security-Header-Sätze, benutzerdefinierte Fehlerseiten |
| WAF | mod_security2-Regelsätze und eigene Regeln |
| Logs / Access, Logs / Error | Log-Viewer für Zugriffs-/Fehlerprotokolle je vhost |
| Logs / WAF Audit | Viewer für `modsec_audit.log` (global, nicht je vhost -- eine mod_security2-Engine für die gesamte Instanz) |
| Log File | Rohe Syslog-Ansicht (Diagnostics &rarr; Log Files &rarr; apache) |

Jede Seite besitzt eine **Apply**-Schaltfläche, die das Formular speichert
und den laufenden Dienst neu konfiguriert (regeneriert `httpd.conf` und
lädt Apache bei Bedarf neu bzw. startet es) -- siehe
[Änderungen übernehmen](#änderungen-übernehmen).

### General Settings

Drei Tabs, die alle auf demselben Formular basieren; jeder Tab hat seine
eigene Apply-Schaltfläche.

**General Settings tab**

| Feld | Beschreibung |
|---|---|
| Enable Apache | Hauptschalter für den gesamten Dienst. |
| Server Tokens | Wie viele Versionsdetails Apache im `Server`-Header und auf generierten Seiten preisgibt (`Minimal` vs. `Full`). |
| Error Log Level | Minimale Schwere, die ins globale Fehlerprotokoll geschrieben wird (`emerg` … `debug`). Kann je HTTP Server überschrieben werden. |
| Enable Compression | Schaltet die `mod_deflate`-Ausgabekompression für komprimierbare Inhaltstypen ein. |

**WAF tab**

| Feld | Beschreibung |
|---|---|
| Enable WAF | Globaler Schalter für mod_security2. Dies lädt nur die Engine und die OWASP-CRS-Basiskonfiguration -- es schützt für sich genommen **nichts**. Zusätzlich muss jedem zu schützenden HTTP Server oder jeder Location ein WAF-Regelsatz zugewiesen werden (siehe [Reverse Proxy](#reverse-proxy) und [WAF](#waf)). |

**Worker Settings tab**

| Feld | Beschreibung |
|---|---|
| Threads per Child | `ThreadsPerChild` -- Worker-Threads je Kindprozess. |
| Max Clients | `MaxRequestWorkers` -- maximale Anzahl gleichzeitiger Verbindungen. |
| Keepalive Timeout | `KeepAliveTimeout` -- Sekunden, die eine persistente Verbindung zwischen Anfragen offengehalten wird. |

### Reverse Proxy

Drei Tabs: **HTTP Servers**, **Locations**, **Upstreams**. Ein typischer
vhost wird von unten nach oben aufgebaut: Upstream Server(s) &rarr;
Upstream &rarr; Location &rarr; HTTP Server (die konkrete Reihenfolge
zeigt die [Schritt-für-Schritt-Anleitung](#schritt-für-schritt-anleitung-eine-interne-anwendung-über-https-veröffentlichen)).

**HTTP Servers** (eine Zeile je vhost)

| Feld | Beschreibung |
|---|---|
| Server Name(s) | Kommagetrennte Hostnamen. Der erste wird zu `ServerName`, die übrigen zu `ServerAlias`. |
| Default Server | Diesen vhost bedienen, wenn kein `ServerName`/`ServerAlias` auf derselben Listen-Adresse passt. Je Listen-Adresse ist nur ein Default Server erlaubt -- das Formular weist einen widersprüchlichen zweiten zurück. |
| Listen Address (HTTP) | Kommagetrennte Liste von Adressen/Ports für einfaches HTTP, z. B. `*:80`, `127.0.0.1:8080`. |
| Listen Address (HTTPS) | Dasselbe, für TLS. Wirkt erst, sobald zusätzlich unten ein Zertifikat ausgewählt ist. |
| Certificate | Server-Zertifikat für den/die HTTPS-Listener, ausgewählt aus dem Trust Store. |
| Client Auth CA *(advanced)* | CA(s) zur Prüfung von Client-Zertifikaten, für mutual TLS. |
| Client Certificate Verification *(advanced)* | `off` / `optional` / `require` -- ob ein Client-Zertifikat angefragt oder zwingend erforderlich ist. |
| TLS Protocols *(advanced)* | Erlaubte TLS-Versionen. |
| TLS Cipher Suite *(advanced)* | Eigene Cipher-Liste; leer lassen für die Systemvorgabe. |
| Enable HTTP/2 | HTTP/2 für diesen vhost anbieten und akzeptieren. |
| Real Client IP Header | Header, der die echte Client-IP trägt, wenn dieser vhost hinter einem weiteren Proxy steht. |
| Trusted Proxies | Netzwerke, die diesen Header setzen dürfen. |
| Access Log Format | `combined` / `common` / `disabled`. |
| Error Log Level | Vhost-spezifische Überschreibung des globalen Error-Log-Levels. |
| Document Root *(advanced)* | Ausweich-`DocumentRoot`, wenn eine passende Location keine eigene besitzt. |
| Max Body Size *(advanced)* | `LimitRequestBody`; leer bedeutet Apache-Standard. |
| Locations | Welche Location-Einträge dieser vhost bedient. |
| ACME Passthrough *(advanced)* | Leitet `/.well-known/acme-challenge/` an den eingebauten HTTP-01-Listener von `os-acme-client` weiter, damit dieser für diesen vhost Zertifikate ausstellen kann. |
| Enable Outlook Anywhere Passthrough | Erkennt klassischen Outlook-Anywhere-Verkehr (RPC-over-HTTP/MAPI) und proxyt ihn über gewöhnliche `ProxyPass`-Locations korrekt, mittels `mod_proxy_msrpc`. Für reine MAPI/HTTP-Exchange-Umgebungen (2016+) nicht nötig. |
| Outlook Anywhere User-Agents *(advanced)* | Zusätzliche User-Agent-Zeichenketten, die als Outlook-Anywhere-Verkehr behandelt werden; leer nutzt die eingebaute Vorgabe von `mod_proxy_msrpc` (`MSRPC`). |
| Security Headers | Ein Security-Header-Satz, der auf den gesamten vhost angewendet wird. |
| Error Pages | Zu servierende benutzerdefinierte Fehlerseiten. |
| IP ACL | Zugriff auf diesen vhost nach Client-IP einschränken. |
| WAF Ruleset | mod_security2/OWASP-CRS-Regelsatz für den gesamten vhost (erfordert global aktivierte WAF). |
| WAF Custom Rules *(advanced)* | Zusätzliche eigene mod_security2-Regeln für diesen vhost. |

**Locations** (eine Zeile je URL-Pfadregel, einem oder mehreren HTTP
Servers zugeordnet)

| Feld | Beschreibung |
|---|---|
| Description | Freitext-Bezeichnung. |
| URL Pattern | Der Pfad, auf den diese Location passt, z. B. `/` oder `/app/`. |
| Match Type *(advanced)* | Präfix-Treffer (`Location`), exakter Treffer oder ein regulärer Ausdruck (`LocationMatch`). |
| Upstream | Anfragen hierhin an diesen Upstream-Pool weiterleiten. Leer lassen, um stattdessen statische Dateien oder PHP lokal zu servieren, oder um stattdessen ein Redirect auszulösen. |
| Path Prefix *(advanced)* | Optionaler Pfad, der an die Balancer-URL angehängt wird. |
| Preserve Host Header *(advanced)* | Den ursprünglichen `Host`-Header an das Backend weiterleiten. |
| WebSocket Support | Passende Verbindungen zu einem WebSocket-Tunnel Richtung Upstream hochstufen. |
| Proxy Timeout *(advanced)* | Netzwerk-Timeout zum Backend, in Sekunden. Für Long-Polling-Clients erhöhen, z. B. Exchange ActiveSync/MAPI-Push-Benachrichtigungen oder Nextclouds `notify_push`. |
| Keep Backend Connection Alive *(advanced)* | Nutzt die TCP-Verbindung zum Backend über mehrere Anfragen hinweg (`ProxyPass keepalive=On`), statt für jede Anfrage eine neue aufzubauen. Standardmäßig aktiviert; erforderlich, damit NTLM/Negotiate-(Kerberos-)Authentifizierung den Proxy übersteht -- Exchange Outlook Anywhere/MAPI over HTTP, EWS und ActiveSync hängen alle davon ab. |
| Pass Request URI Unmodified *(advanced)* | Leitet die Request-URI unverändert an das Backend weiter (`ProxyPass nocanon`), ohne dass Apache sie zuvor normalisiert/dekodiert. Für einige Exchange-Endpunkte nötig (EWS, Autodiscover, ActiveSync), deren URLs kodierte Zeichen enthalten, die unverändert beim Backend ankommen müssen. |
| Redirect Target URL / Redirect Status Code | Löst statt Proxying oder lokalem Servieren einen HTTP-Redirect aus; wird nur genutzt, wenn kein Upstream ausgewählt ist. Praktisch z. B. um `/` auf einem Exchange-vhost nach `/owa` zu schicken, oder um Nextclouds `/.well-known/carddav`- und `/.well-known/caldav`-Komfort-Redirects nach `/remote.php/dav` einzurichten. |
| Extra Request Headers *(advanced)* | Ein `Name: Wert`-Paar pro Zeile, wird der an das Backend gesendeten Anfrage hinzugefügt. |
| Extra Response Headers *(advanced)* | Ein `Name: Wert`-Paar pro Zeile, wird der an den Client gesendeten Antwort hinzugefügt -- z. B. CORS (`Access-Control-Allow-Origin: ...`) für CalDAV/CardDAV-Clients. |
| Document Root *(advanced)* | Dateisystempfad, der serviert wird, wenn kein Upstream ausgewählt ist. |
| Enable PHP | `*.php`-Anfragen an den lokalen php-fpm-Socket weiterreichen. |
| Index Files *(advanced)* | Kommagetrennte `DirectoryIndex`-Dateinamen. |
| Directory Listing *(advanced)* | Eine Verzeichnisauflistung zeigen, wenn keine Index-Datei vorhanden ist. |
| Custom Rewrite Rule *(advanced)* | Eine rohe `RewriteRule`-Zeile, wortwörtlich. |
| Basic Auth Realm | Realm, der im Login-Dialog des Browsers angezeigt wird; leer lassen, um Basic Auth hier zu deaktivieren. Bei Locations, die zu einer Anwendung mit eigener Authentifizierung proxyen (Exchange, Nextcloud, ...), dieses Feld leer lassen -- sonst würde Apaches Basic Auth den `Authorization`-Header abfangen, bevor NTLM/Negotiate oder das Login der Anwendung ihn zu sehen bekommen. |
| Basic Auth User List | Welche Benutzerliste sich hier authentifizieren darf. |
| IP ACL | Zugriff auf diese Location nach Client-IP einschränken. |
| WAF Ruleset | Überschreibt den WAF-Regelsatz des vhosts nur für diese Location. |
| WAF Custom Rules *(advanced)* | Zusätzliche eigene Regeln für diese Location. |

Eine Location wirkt erst, sobald sie im **Locations**-Feld eines HTTP
Servers eingetragen ist.

**Upstreams** (Lastverteiler-Pools, aufgeteilt in zwei Tabellen)

*Upstream Servers* -- einzelne Backends:

| Feld | Beschreibung |
|---|---|
| Description | Freitext-Bezeichnung. |
| Server | Backend-Hostname oder -IP. |
| Port | Backend-TCP-Port. |
| Load Factor *(advanced)* | Relatives Gewicht im Lastverteiler. |
| Status | `Active`, `Disabled`, oder `Hot Standby` (wird nur genutzt, wenn alle aktiven Server ausgefallen sind). |

*Upstreams* -- Pools der obigen Server:

| Feld | Beschreibung |
|---|---|
| Description | Freitext-Bezeichnung, wird bei der Auswahl dieses Upstreams in einer Location angezeigt. |
| Servers | Welche Upstream Servers zu diesem Pool gehören. |
| Load Balancing Method | `By Requests` (Round Robin, gewichtet), `By Traffic`, oder `By Busyness`. |
| Enable TLS | Mit den Backends über HTTPS statt einfachem HTTP sprechen. |
| Verify Certificate *(advanced)* | Das Backend-Zertifikat prüfen: Vertrauenskette gegen die unten angegebene CA-Liste, Hostname-/CN-Abgleich sowie Ablaufdatum. Ausschalten, wenn das Backend ein selbstsigniertes oder abgelaufenes Zertifikat verwendet, das sich nicht ersetzen lässt (z. B. das Standard-Webinterface-Zertifikat einer anderen OPNsense-Box). |
| Client Certificate *(advanced)* | Optionales Client-Zertifikat, das dem Backend präsentiert wird. |
| Trusted CA *(advanced)* | CA(s) zur Prüfung des Backends; leer nutzt den System-Trust-Store. |
| TLS Protocols *(advanced)* | Richtung Backend erlaubte Protokollversionen. |

### Access

Vier Tabs.

**Users & Auth** -- zwei Tabellen: **Users** (`Username`/`Password`, beim
Speichern gehasht) und **User Lists** (`Name` + welche Users dazugehören).
Eine Benutzerliste im *Basic Auth User List*-Feld einer Location
referenzieren, um sie mit HTTP Basic Auth zu schützen.

**IP ACLs** -- `Description`, `Default Action` (erlauben/verweigern, wenn
nichts unten passt), `Allow Networks`, `Deny Networks` (kommagetrennte
IPs/CIDRs, haben immer Vorrang vor der Default Action). Eine ACL im
*IP ACL*-Feld eines HTTP Servers oder einer Location zuweisen.

**Security Headers** -- ein benannter Satz von Response-Headern:
Referrer-Policy, X-XSS-Protection, X-Content-Type-Options, HSTS (max-age /
include-subdomains / preload), sowie ein vollständiger
Content-Security-Policy-Baukasten (`default-src`, `script-src`,
`style-src`, `img-src`, `font-src`, `connect-src`, `media-src`,
`frame-src`, `frame-ancestors`, `form-action`, `worker-src`, plus ein
Report-Only-Schalter). Einen Satz im *Security Headers*-Feld eines HTTP
Servers zuweisen.

**Error Pages** -- `Name`, `Status Codes` (kommagetrennt, z. B. `404` oder
`500,502,503,504`), eine optionale `Redirect URL`, oder rohes HTML als
`Page Content`. Eine oder mehrere Seiten im *Error Pages*-Feld eines HTTP
Servers referenzieren.

### WAF

Zwei Tabellen, beide bewusst global (nicht je vhost), damit dasselbe
Tuning über viele HTTP Servers/Locations hinweg wiederverwendet werden
kann. Dieser Abschnitt beschreibt nur die Felder -- wie die WAF tatsächlich
implementiert ist, der tägliche Betrieb, Fehlersuche, das Schreiben
eigener Regeln und der Umgang mit Falsch-Positiven stehen in
**[WAF.de.md](os-apache-maxit-waf.md)**.

**Rulesets**

| Feld | Beschreibung |
|---|---|
| Description | Freitext-Bezeichnung, wird bei der Zuweisung dieses Regelsatzes an anderer Stelle angezeigt. |
| Learning Mode | Nur Erkennung: Verstöße werden protokolliert, aber nichts wird blockiert. Beim Einrichten eines neuen Regelsatzes hiermit beginnen. |
| Paranoia Level | 1 (Basis, Standard) bis 4 (Maximum) -- höhere Stufen aktivieren strengere OWASP-CRS-Regeln, auf Kosten von mehr Falsch-Positiven. |
| Inbound / Outbound Anomaly Threshold *(advanced)* | Anfrage/Antwort wird blockiert, sobald ihr kumulativer Anomalie-Score diesen Wert erreicht. |
| Application Exclusion Packages *(advanced)* | Bekannte Falsch-Positiv-Ausnahmen für bestimmte Anwendungen (WordPress, Nextcloud, …), sofern das passende OWASP-CRS-Ausnahmepaket installiert ist. |
| Whitelisted Source IPs *(advanced)* | IPs/CIDRs, die die WAF vollständig umgehen. |

**Custom Rules** -- `Description`, `Phase` (1 Request Headers, 2 Request
Body -- Standard, 3 Response Headers, 4 Response Body, 5 Logging), sowie
der rohe `SecRule`-/`SecAction`-Text.

Nichts davon wirkt, solange nicht **Enable WAF** unter General Settings
eingeschaltet *und* der Regelsatz/die eigenen Regeln einem HTTP Server oder
einer Location auf der Reverse-Proxy-Seite zugewiesen sind.

### Logs

Unter **Logs / Access** und **Logs / Error** lässt sich ein vhost und eine
Logdatei (aktuell oder eine rotierte `.gz`-Datei) auswählen, darin blättern
und per Textfilter durchsuchen: das Filterfeld sucht in der Request-Zeile
(Access) bzw. der Fehlermeldung (Error). **Logs / WAF Audit** zeigt
`modsec_audit.log` auf dieselbe Weise, nur ohne vhost-Auswahl -- das
Audit-Log von mod_security2 ist eine einzige globale Datei für alle
vhosts, nicht eine je vhost -- mit Spalten für die getroffenen Regel-IDs,
Meldungen und ob die Anfrage blockiert wurde; das Filterfeld sucht in den
zusammengefügten Regelmeldungen. **Log File** (unter Diagnostics &rarr;
Log Files) zeigt den rohen Syslog-Strom für alles, was das Setup-Skript
selbst protokolliert (Probleme beim Zertifikatsexport, fehlgeschlagene
Konfigurationstests usw.) -- das ist nicht dasselbe wie die
Zugriffs-/Fehlerprotokolle je vhost oder das WAF-Audit-Log.

### Änderungen übernehmen

Die **Apply**-Schaltfläche jeder Seite speichert deren Felder und
konfiguriert Apache anschließend neu: `httpd.conf` (samt der Blöcke je
vhost) wird aus der aktuellen Konfiguration neu erzeugt, TLS-Schlüssel/
-Zertifikate und Basic-Auth-Passwortdateien werden (erneut) auf die
Festplatte exportiert, und Apache wird neu geladen (oder gestartet, falls
es nicht lief). Ein kleines Status-Widget neben der Apply-Schaltfläche
zeigt, ob der Dienst gerade läuft, und erlaubt, ihn direkt zu
starten/stoppen/neu zu starten.

### Tipps und Fallstricke

* Mit **(advanced)** markierte Felder oben sind standardmäßig
  ausgeblendet -- auf den kleinen "advanced mode"-Schalter oben in einem
  Tab klicken, um sie einzublenden.
* Kommagetrennte Felder (Server Name(s), Listen Address, Trusted
  Proxies, Index Files, Netzwerke/CIDRs, Status Codes, …) erwarten eine
  einfache, direkt ins Feld eingetippte kommagetrennte Zeichenkette, kein
  Tag-/Token-Auswahlfeld.
* Eine Location bedient erst Datenverkehr, sobald sie im *Locations*-Feld
  eines HTTP Servers eingetragen ist, und ein HTTP Server akzeptiert
  HTTPS erst, wenn sowohl eine *Listen Address (HTTPS)* als auch ein
  *Certificate* gesetzt sind.
* Die WAF wird in zwei Schritten aktiviert: zunächst global (General
  Settings), dann einen Regelsatz je HTTP Server/Location zuweisen
  (Reverse-Proxy-Seite). Allein den globalen Schalter umzulegen schützt
  nichts.
* Je Listen-Adresse (HTTP und HTTPS zusammen betrachtet) ist nur ein
  *Default Server* erlaubt -- das Formular weist einen zweiten ab, bis der
  Konflikt aufgelöst ist.

### Reverse Proxy für reale Anwendungen (Exchange, Nextcloud, ...)

Die obigen Location-Felder decken das ab, was die meisten produktiven
Web-Apps hinter einem Reverse Proxy tatsächlich brauchen. Zwei Backends,
die fast alle davon fordern, sind Microsoft Exchange und Nextcloud:

* **Exchange (OWA, ECP, EWS, Outlook Anywhere/MAPI, ActiveSync,
  Autodiscover)** -- *Basic Auth Realm* bei diesen Locations leer lassen,
  damit Apache den `Authorization`-Header nicht selbst abfängt; Exchange
  braucht NTLM/Negotiate unverändert. *Keep Backend Connection Alive*
  aktiviert lassen (Standard), damit dieser Handshake über mehrere
  Anfragen hinweg bestehen bleibt. *Pass Request URI Unmodified* bei den
  Locations für EWS, Autodiscover und ActiveSync
  (Microsoft-Server-ActiveSync) aktivieren -- deren URLs enthalten
  kodierte Zeichen, die Apache sonst verändern würde. *Proxy Timeout* bei
  der ActiveSync-Location erhöhen, damit Long-Polling-("Push"-)
  Verbindungen nicht abgebrochen werden. Ein *Redirect* von `/` nach
  `/owa` ist auf dem Mail-vhost eine übliche Komfortfunktion. Falls auch
  klassisches Outlook Anywhere (RPC-over-HTTP/MAPI, im Gegensatz zu
  reinem MAPI/HTTP ab Exchange 2016) unterstützt werden soll,
  *Enable Outlook Anywhere Passthrough* am HTTP Server aktivieren und
  proxyende Locations für `/rpc/rpcproxy.dll` und
  `/rpcwithcert/rpcproxy.dll` Richtung Exchange-CAS anlegen.
* **Nextcloud** -- am HTTP Server eine großzügige *Max Body Size* setzen
  (oder `0` für unbegrenzt), damit große Datei-Uploads nicht abgelehnt
  werden, und *Proxy Timeout* bei der Location für `/` erhöhen, damit
  große Uploads/Downloads sowie die Long-Polling-Verbindung der
  `notify_push`-App nicht abbrechen (`notify_push` braucht außerdem
  aktiviertes *WebSocket Support*). Der automatisch gesetzte
  `X-Forwarded-Host`-Header (siehe unten) ermöglicht es Nextclouds
  `overwritehost`, den öffentlichen Hostnamen hinter diesem Proxy korrekt
  zu erkennen. Zwei kleine *Redirect*-Locations, `/.well-known/carddav`
  und `/.well-known/caldav` nach `/remote.php/dav`, bedienen die
  CalDAV/CardDAV-Auto-Erkennung der meisten Clients.
* Jede proxyende Location sendet automatisch `X-Forwarded-Proto`,
  `X-Forwarded-Port`, `X-Forwarded-Host` und `X-Real-IP` an das Backend --
  die meisten Anwendungen (einschließlich der beiden oben) nutzen diese,
  um hinter einem Reverse Proxy korrekte, auf sich selbst verweisende URLs
  zu bilden, ganz ohne zusätzliche Konfiguration. *Extra Request/Response
  Headers* decken alles ab, was eine Anwendung darüber hinaus braucht
  (ein individueller Trusted-Header-Name, CORS für CalDAV/CardDAV-Clients,
  Cache-Control-Anpassungen, ...).

## Schritt-für-Schritt-Anleitung: eine interne Anwendung über HTTPS veröffentlichen

Dies führt durch eine realistische Ersteinrichtung: eine einzelne interne
Webanwendung (`app.internal.example`, erreichbar unter `10.0.0.20:8080`)
wird über Apache mit TLS-Terminierung reverse-proxied, anschließend werden
Basic Auth und ein WAF-Regelsatz ergänzt.

### 1. Apache aktivieren

**Services &rarr; Apache &rarr; General Settings &rarr; General Settings
tab** &rarr; **Enable Apache** anhaken &rarr; **Apply**.

Server Tokens/Error Log Level/Compression zunächst auf den Standardwerten
belassen; diese lassen sich später anpassen.

### 2. Sicherstellen, dass ein Zertifikat vorhanden ist

**System &rarr; Trust &rarr; Certificates**: ein Zertifikat für
`app.example.com` (den öffentlichen Namen, den Besucher verwenden werden)
importieren oder erzeugen. Falls noch keines vorhanden ist, funktionieren
sowohl ein intern signiertes als auch ein per `os-acme-client`
ausgestelltes Zertifikat -- diesen Schritt einfach nachholen, sobald eines
vorhanden ist.

### 3. Das Backend definieren (Upstream Server + Upstream)

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; Upstreams tab**:

1. Unter **Upstream Servers** einen anlegen:
   - Description: `app backend`
   - Server: `10.0.0.20`
   - Port: `8080`
   - Status: `Active`
2. Unter **Upstreams** einen Pool anlegen:
   - Description: `app pool`
   - Servers: `app backend` auswählen
   - Load Balancing Method: `By Requests`
   - *Enable TLS* ausgeschaltet lassen (das Backend spricht hier einfaches HTTP)

Selbst ein einzelnes Backend benötigt ein Upstream-Server-/Upstream-Paar --
Apache proxied stets über einen benannten Balancer, es gibt keine direkte
"proxy zu genau diesem einen Host"-Abkürzung.

### 4. Die Location definieren

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; Locations tab** &rarr;
eine anlegen:

- Description: `app root`
- URL Pattern: `/`
- Upstream: `app pool`
- Preserve Host Header: an (Standard)
- WebSocket Support ausgeschaltet lassen, außer die Anwendung braucht es

### 5. Den HTTP Server (vhost) definieren

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; HTTP Servers tab**
&rarr; einen anlegen:

- Server Name(s): `app.example.com`
- Listen Address (HTTP): `*:80` (beibehalten, damit reine HTTP-Anfragen
  Apache weiterhin erreichen; später bei Bedarf mit einer
  Redirect-auf-HTTPS-Location kombinieren, oder einfach darauf setzen,
  dass Besucher die HTTPS-URL verwenden)
- Listen Address (HTTPS): `*:443`
- Certificate: das in Schritt 2 importierte Zertifikat
- Enable HTTP/2: an
- Locations: `app root` auswählen

**Apply**. Im Browser `https://app.example.com/` aufrufen -- die interne
Anwendung sollte nun über Apache ausgeliefert werden.

### 6. Prüfen, ob es funktioniert hat

- **Services &rarr; Apache &rarr; General Settings**: das Status-Widget
  neben einer beliebigen Apply-Schaltfläche sollte den Dienst als
  laufend anzeigen.
- **Services &rarr; Apache &rarr; Logs / Access**: den vhost auswählen
  und prüfen, ob Anfragen erscheinen.
- Auf der Firewall-Kommandozeile validiert `configctl apache test_config`
  die erzeugte `httpd.conf` erneut, ohne etwas neu zu starten.

### 7. (Optional) Einen Pfad mit Basic Auth schützen

**Services &rarr; Apache &rarr; Access &rarr; Users & Auth tab**:

1. Einen **User** anlegen: Benutzername/Passwort für die Person, die
   Zugriff benötigt.
2. Eine **User List** anlegen (z. B. `app-admins`), die diesen User enthält.

Zurück auf **Reverse Proxy &rarr; Locations**: entweder `app root`
bearbeiten oder eine neue Location für einen bestimmten Pfad anlegen
(z. B. `/admin/`):

- Basic Auth Realm: `Admin area`
- Basic Auth User List: `app-admins`

**Apply**. Dieser Pfad (oder der gesamte vhost, falls `app root` bearbeitet
wurde) fragt nun nach Zugangsdaten.

### 8. (Optional) Nach Quell-IP einschränken

**Services &rarr; Apache &rarr; Access &rarr; IP ACLs tab** &rarr; eine
anlegen:

- Description: `office only`
- Default Action: `Deny Access`
- Allow Networks: `203.0.113.0/24`

Über das **IP ACL**-Feld des HTTP Servers oder der Location zuweisen,
dann **Apply**.

### 9. (Optional) Die WAF einschalten

1. **Services &rarr; Apache &rarr; WAF &rarr; Rulesets tab** &rarr; einen
   anlegen: Description `default`, Learning Mode zunächst **an** (damit
   nichts blockiert wird, während auf Falsch-Positive geachtet wird),
   Paranoia Level `1`.
2. **General Settings &rarr; WAF tab** &rarr; **Enable WAF** &rarr; Apply.
3. Zurück auf **Reverse Proxy &rarr; HTTP Servers**: den vhost bearbeiten
   und **WAF Ruleset** auf `default` setzen. **Apply**.
4. Einige Tage **Logs / WAF Audit** (oder
   `/var/log/apache/modsec_audit.log`) beobachten, dann **Learning Mode**
   ausschalten, sobald Sicherheit besteht, dass kein legitimer Verkehr
   blockiert wird.

Damit steht ein vollständiger, TLS-terminierter, authentifizierter,
WAF-geschützter Reverse Proxy vor einer internen Anwendung -- alles
Weitere auf den Seiten Reverse Proxy/Access/WAF (mehrere Locations,
mehrere Backends je Upstream, Security Headers, benutzerdefinierte
Fehlerseiten, eigene mod_security2-Regeln) baut auf genau dieser
Grundstruktur auf.

## Hinweise für Entwickler

Die folgenden Abschnitte sind Implementierungshinweise für alle, die
dieses Plugin erweitern oder überprüfen -- für die reine Nutzung nicht
erforderlich.

### Verhältnis zu www/nginx

Dieses Plugin versucht bewusst **nicht**, ein funktionsgleicher Port von
`www/nginx` zu sein. Wo nginx keine Apache-Entsprechung hat, entfällt das
jeweilige Feature, statt es vorzutäuschen:

* TCP/UDP-Stream-Proxying (nginx `stream {}`-Blöcke)
* VTS-Traffic-Statistik-Modul
* TLS-Client-Fingerprint-MITM-Erkennung
* HTTP/3 / QUIC, 0-RTT
* die `resolver`-Direktive und SNI-basiertes Upstream-Routing-Mapping
* NAXSIs Punkte-basierter Regel-Score -- ersetzt durch mod_security2s/
  OWASP CRS' eigenes Tuning-Modell (Paranoia Level, Anomaly Score
  Threshold, Rule Exclusions, Custom-Rule-Schnipsel), da mod_security2 in
  der Praxis genauso konfiguriert wird

Wo ein Feature doch übernommen wird, geschieht dies mit dem nächstgelegenen
Apache-Modul statt der Nachahmung von nginx-Direktivennamen:
`mod_proxy_balancer` für Upstreams, `mod_authz_core`s `Require`-Familie
für IP-ACLs, `mod_headers` für Security Headers, `mod_security2` für die
WAF usw.

### Frontend

Anders als das mitgelieferte Backbone/Webpack-Frontend von `www/nginx`
nutzt dieses Plugin OPNsenses Standard-Grid-/Dialog-Framework
(`UIBootgrid`, `base_bootgrid_table`, `base_dialog`,
`base_tabs_header/content`) -- dasselbe Muster, das `www/caddy` und
`devel/grid_example` verwenden. Es gibt keinen separaten JS-Build-Schritt.

### Das apache-Plugin als Infrastruktur

In Anlehnung an die Include-Hook-Konvention des nginx-Plugins: jede Datei,
die unter `/usr/local/etc/apache24/opnsense_vhost_plugins/*.conf` abgelegt
wird, wird automatisch eingebunden -- für Plugins, die etwas über Apache
ausliefern müssen, ohne selbst einen vhost zu besitzen.

### Einen vhost anhaken

Ein Verzeichnis namens `UUID_pre` oder `UUID_post` (UUID ist die UUID des
`http_server`) unter `/usr/local/etc/apache24/` anlegen und eine
`.conf`-Datei hineinlegen; sie wird am Anfang bzw. Ende des Bodys dieses
vhosts eingebunden.

### Hinweise zur Verifikation

Dieses Plugin wurde ursprünglich ohne Zugriff auf ein FreeBSD-/
OPNsense-System verfasst und konnte daher nicht Ende-zu-Ende funktional
getestet werden (`apachectl configtest`, ein tatsächlicher
Browser-Roundtrip durch einen reverse-proxied vhost, das Blockieren einer
Anfrage durch mod_security2 usw.). Alles wurde auf XML-Wohlgeformtheit und
PHP-Syntaxbalance geprüft, und Template-Direktiven wurden gegen die
Apache-/mod_security2-Dokumentation abgeglichen -- eine Verifikation in der
Praxis auf einer echten OPNsense-Box oder -VM war jedoch weiterhin nötig
vor dem Produktiveinsatz, insbesondere:

* das genaue Layout von `etc/apache24/modules.d/*.conf` / `Modules/*.conf`,
  in das der `www/mod_security`-Port seine Dateien ablegt
* ob `mod_http2` in einem Standard-`apache24`-(Apache-2.4-)Paket-Build
  verfügbar ist

Auf einer echten Box bestätigt: `etc/apache24/modules.d/*.conf` ist bei
OPNsenses `apache24`-Paket leer bzw. minimal (kein MPM, kein `mod_unixd`,
…), anders als bei einem reinen FreeBSD-Ports-Build. `httpd.conf` setzt
inzwischen nicht mehr voraus, dass irgendein Basismodul von dort kommt --
jedes von der erzeugten Konfiguration tatsächlich genutzte Modul (MPM,
Auth, Logging, mime/dir/alias, gzip, Headers, Rewrite, der
Proxy-/Balancer-/TLS-Stack) wird nun explizit geladen, jeweils mit
`<IfModule !x>` abgesichert, damit nichts doppelt geladen wird, falls eine
Installation es doch bereitstellt. `mod_http2` und `mod_security2` sind
bewusst nicht in dieser Liste enthalten, da sie in optionalen
Zusatzpaketen/-Ports stecken und nur über `<IfModule>`-Absicherungen
genutzt werden -- eine Installation ohne diese funktioniert weiterhin,
nur ohne HTTP/2 bzw. WAF.

Ebenfalls seither auf einer echten Box bestätigt und behoben: fehlende
`Listen`-Direktiven (Apache band tatsächlich nie an die konfigurierten
vhost-Ports), mehrere Jinja2-Template-Include-Newline-Bugs, die
Direktiven auf einer Zeile zusammenklebten, der TLS-Schlüssel-/
Zertifikatsexport, der außerhalb der eigenen GUI-Aktionen des Plugins nie
lief (behoben über den `apache24_setup`-rc.conf.d-Hook, dieselbe
Konvention, die `www/nginx`/`mail/postfix`/`net/freeradius` verwenden),
die `scripts/apache/*.php`-Hilfsskripte, denen das Ausführungsbit fehlte,
ein fehlendes `SecDataDir`-Verzeichnis, wodurch mod_security2 nicht
geladen werden konnte, sobald die WAF aktiviert wurde, `<type>textarea</type>`
beim "Rule"-Feld der Custom Rules und beim "Page Content"-Feld der Error
Pages (kein gültiger `form_input_tr.volt`-Typ -- der Core rendert für
einen unbekannten Typ still und leise gar kein Eingabeelement, wodurch
das Feld nur leer aussah; der korrekte Typ ist `textbox`), die
Abschnitte "Client Address" und "Logging" im HTTP-Servers-Dialog, bei
denen jedes Feld als `advanced` markiert war, wodurch diese Abschnitte
standardmäßig komplett leer aussahen, sowie ein Upstream-Schalter
*Verify Certificate*, der `SSLProxyCheckPeerExpire` nicht anfasste (eine
von `SSLProxyVerify` unabhängige Direktive, die standardmäßig `on` ist),
wodurch ein Backend mit abgelaufenem Zertifikat den TLS-Handshake selbst
bei vollständig deaktiviertem *Verify Certificate* weiterhin scheitern
ließ, sowie die Logs-Seiten, die nie Inhalte anzeigten: Seite,
Zeilen-pro-Seite und Filter wurden als Query-String übertragen
(`$.get(url, {page, ...})`), aber der Router von OPNsense bindet
Action-Parameter ausschließlich aus URL-Pfadsegmenten, sodass
`accessesAction`/`errorsAction` immer ihre Standardwerte erhielten (Seite
0, 0 Zeilen pro Seite -- was "jede Zeile unbegrenzt zurückgeben"
bedeutet -- und einen leeren Filter), unabhängig davon, was in der
Oberfläche ausgewählt war; behoben, indem diese drei Werte stattdessen
als zusätzliche Pfadsegmente gesendet werden, wie es der Controller die
ganze Zeit erwartet hatte (sein `urldecode()`-Aufruf auf den
Filter-Parameter ergibt nur bei einem Pfadsegment Sinn, da das Framework
solche Segmente nicht automatisch dekodiert). Dabei auch gleich das
Filterfeld selbst repariert: es sendete reinen Text, aber die
Log-Parser erwarten ein JSON-Objekt, das Spaltennamen auf Filtertext
abbildet, wodurch der Text stillschweigend ignoriert wurde -- jetzt
serverseitig je Log-Typ verpackt (`request_line` für Access,
`message` für Error). Die Pfadsegment-Korrektur allein reichte unter
PHP 8.2+ aber noch nicht aus: `LogParserBase` setzte
`$this->file_name`/`page`/`per_page`/`query`/`page_count`/`total_lines`/
`query_lines`, ohne diese jemals als Klasseneigenschaften zu
deklarieren, sodass jede Anfrage einen Schwung "Deprecated: Creation of
dynamic property"-Warnungen auf stdout ausgab, noch bevor das
eigentliche JSON folgte. Da `LogsController::sendConfigdToClient()` die
rohe Ausgabe von configd direkt in den HTTP-Antwortkörper schreibt,
landeten diese Warnungen vor dem JSON, sodass der Browser eine
ungültige, nicht parsbare Antwort erhielt und die Tabelle stillschweigend
leer blieb -- obwohl `configctl apache log ...` auf der Kommandozeile
eindeutig gute Daten zurücklieferte. Behoben, indem alle Eigenschaften
von `LogParserBase` von vornherein deklariert werden.
