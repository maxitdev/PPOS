# os-sslproxy-maxit – SSLproxy (TLS-Aufbruch) für OPNsense

`os-sslproxy-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG, das
[SSLproxy](https://github.com/sonertari/SSLproxy) in die OPNsense-WebGUI
integriert. SSLproxy ist ein transparenter TLS-Proxy (Weiterentwicklung von
SSLsplit): Er terminiert umgeleitete TLS-Verbindungen, stellt für die
angefragte Gegenstelle im Vorbeigehen ein Zertifikat aus einer eigenen CA aus
und baut zum eigentlichen Ziel eine neue TLS-Verbindung auf. Der entschlüsselte
Datenstrom kann protokolliert (Connect-Log, Content-Log, PCAP), auf ein
Interface gespiegelt (IDS/IPS-Auswertung) oder an ein lauschendes Programm
übergeben werden.

Das Plugin deckt den Optionsumfang von SSLproxy vollständig in acht Reitern ab
und generiert daraus `/usr/local/etc/sslproxy.conf`.

> **Rechtlicher Hinweis:** Der Aufbruch von TLS-Verbindungen greift in die
> vertrauliche Kommunikation der Nutzer ein. Vor dem produktiven Einsatz sind
> Datenschutz, Mitbestimmung (z. B. Betriebsrat) und die Zulässigkeit im
> jeweiligen Umfeld zu klären. Content- und PCAP-Logs enthalten Klartextdaten
> inklusive Zugangsdaten und sind entsprechend zu schützen.

> **Technischer Hinweis:** TLS-Aufbruch funktioniert nicht mit Certificate
> Pinning (viele Apps, Update-Dienste, Banking) und nicht mit Diensten, die
> Client-Zertifikate erwarten. Solche Ziele müssen vom Aufbruch ausgenommen
> werden – entweder über die NAT-Regeln, die den Verkehr umleiten, oder über
> `PassSite`-Direktiven unter
> [Additional Configuration](#reiter-advanced).

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Funktionsprinzip](#funktionsprinzip)
3. [Installation](#installation)
4. [Einrichtung](#einrichtung)
5. [Split- und Divert-Betrieb](#split--und-divert-betrieb)
6. [Reiter "General"](#reiter-general)
7. [Reiter "Listeners"](#reiter-listeners)
8. [Reiter "TLS / Certificates"](#reiter-tls--certificates)
9. [Reiter "HTTP"](#reiter-http)
10. [Reiter "Authentication"](#reiter-authentication)
11. [Reiter "Mirroring"](#reiter-mirroring)
12. [Reiter "Logging"](#reiter-logging)
13. [Reiter "Advanced"](#reiter-advanced)
14. [Reiter "Content" und Reset](#reiter-content-und-reset)
15. [Dateien und Pfade](#dateien-und-pfade)
16. [Berechtigungen](#berechtigungen)
17. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über das
  `os-sslproxy-maxit` bezogen wird. Das FreeBSD-Paket `sslproxy` wird als
  Abhängigkeit automatisch mitinstalliert.
- Eine eigene Zertifizierungsstelle unter **System → Vertrauen → Autoritäten**
  (Typ: interne CA, inklusive privatem Schlüssel). Diese CA signiert die im
  Betrieb erzeugten Zertifikate.
- Das CA-Zertifikat muss auf allen betroffenen Clients als vertrauenswürdige
  Stammzertifizierungsstelle installiert sein, sonst erhalten die Nutzer bei
  jeder Verbindung Zertifikatswarnungen.
- NAT-Regeln, die den zu untersuchenden Verkehr auf die Listener von SSLproxy
  umleiten (siehe [Funktionsprinzip](#funktionsprinzip)).
- Ausreichend Plattenplatz unter `/var/db`, falls Content- oder PCAP-Logging
  verwendet wird.

## Funktionsprinzip

SSLproxy leitet selbst keinen Verkehr um – das übernimmt die
Paketfilter-Ebene:

1. Eine Port-Weiterleitung unter **Firewall → NAT → Port-Weiterleitung** lenkt
   den gewünschten Verkehr (z. B. LAN → beliebiges Ziel, TCP/443) auf die
   Firewall selbst, Ziel `127.0.0.1` und den unter
   [Listeners](#reiter-listeners) konfigurierten Port (z. B. `8443`).
2. SSLproxy fragt über die konfigurierte **NAT Engine** (`pf`) das
   ursprüngliche Verbindungsziel ab und baut dorthin die ausgehende Verbindung
   auf.
3. Für die Client-Seite wird ein Zertifikat mit dem angefragten Hostnamen aus
   der ausgewählten **Root CA** erzeugt.

Die vom Plugin generierten Listener binden bewusst nur auf `127.0.0.1` und
`::1`. Ohne passende NAT-Regel erreicht SSLproxy daher kein Paket – ein
gestarteter Dienst ohne Einträge unter [Content](#reiter-content-und-reset) ist
in der Regel genau darauf zurückzuführen.

## Installation

1. Unter **System → Firmware → Plugins** nach `os-sslproxy-maxit` suchen und
   installieren.
2. In der linken Navigation erscheint unter **Services** der neue Menüpunkt
   **SSLproxy → General**.

## Einrichtung

1. Unter **System → Vertrauen → Autoritäten** eine interne CA anlegen (falls
   noch nicht vorhanden) und deren Zertifikat auf den Clients verteilen.
2. **Services → SSLproxy → General** öffnen, **Enable** aktivieren und unter
   **Root CA** die eben angelegte CA auswählen. Die Auswahl einer CA ist
   zwingend – ohne sie kann SSLproxy keine Zertifikate ausstellen und startet
   nicht.
3. Im Reiter **Listeners** die benötigten Protokoll-Ports setzen und alle
   übrigen Felder leer lassen (Standard: HTTP `8080`, HTTPS `8443`).
4. Im Reiter **Logging** festlegen, was protokolliert werden soll. Für einen
   ersten Test genügt das **Connection Log**.
5. **Save** klicken. Der Button speichert alle Reiter, schreibt
   `/usr/local/etc/sslproxy.conf`, exportiert die Zertifikate und startet den
   Dienst neu.
6. Unter **Firewall → NAT → Port-Weiterleitung** die Umleitung des zu
   untersuchenden Verkehrs auf `127.0.0.1` und den jeweiligen Listener-Port
   anlegen. Für Ausnahmen (Pinning, Banking, Update-Dienste) an dieser Stelle
   vorgelagerte Regeln ohne Umleitung verwenden.
7. Von einem Client eine Testverbindung aufbauen und im Reiter **Content**
   prüfen, ob Dateien in `/var/db/sslproxy` entstehen.

## Split- und Divert-Betrieb

SSLproxy kennt zwei Betriebsarten:

- **Split-Betrieb:** Der entschlüsselte Datenstrom wird nur protokolliert bzw.
  gespiegelt und direkt weitergeleitet. Das entspricht dem Verhalten von
  SSLsplit.
- **Divert-Betrieb:** Der entschlüsselte Datenstrom wird an ein lokal
  lauschendes Programm (IDS, HTTP-Proxy o. ä.) übergeben, das die Daten
  auswertet und an SSLproxy zurückgibt. Dafür muss in der jeweiligen
  `ProxySpec` ein Divert-Port (`up:<port>`) angegeben sein, und das
  Zielprogramm muss die von SSLproxy in das erste Paket eingefügte
  Rücksprungadresse verstehen.

Die vom Plugin aus dem Reiter [Listeners](#reiter-listeners) generierten
`ProxySpec`-Zeilen enthalten **keinen** Divert-Port. Für Divert-Betrieb genügt
der Schalter **Divert Mode** allein also nicht: Die betreffende `ProxySpec` ist
im Reiter [Advanced](#reiter-advanced) unter **Additional Configuration**
selbst zu definieren – einzeilig mit `up:<port>` oder als strukturierter
`ProxySpec`-Block, in dem sich zusätzlich `DivertPort`, `ReturnAddr` und
`FilterRule`-Blöcke setzen lassen. Für reine Mitschnitt-/IDS-Szenarien über die
Reiter *Logging* und *Mirroring* ist **Divert Mode** zu deaktivieren, damit
Konfiguration und tatsächliches Verhalten übereinstimmen.

## Reiter "General"

| Feld | Beschreibung |
|---|---|
| Enable | Aktiviert den Dienst (Autostart und Start/Stop beim Speichern). |
| Root CA | CA aus **System → Vertrauen → Autoritäten**, mit der die im Betrieb erzeugten Zertifikate signiert werden (`CACert`/`CAKey`). Pflichtangabe für den Betrieb. |
| Default Leaf Certificate | Optionales festes Zertifikat/Schlüsselpaar (`DefaultLeafCert`/`LeafKey`), das statt bzw. als Rückfallebene für dynamisch erzeugte Zertifikate verwendet wird. |
| Client Certificate | Optionales Client-Zertifikat (`ClientCert`/`ClientKey`), das gegenüber Zielservern präsentiert wird, die eine Client-Authentifizierung verlangen. |
| NAT Engine | Mechanismus, über den das ursprüngliche Verbindungsziel ermittelt wird (`NATEngine`): `pf` (Standard für OPNsense) oder `ipfw`. |
| Divert Mode | Schaltet Divert- statt Split-Betrieb ein (`Divert`). Siehe [Split- und Divert-Betrieb](#split--und-divert-betrieb). |
| Run As User / Run As Group | Benutzer/Gruppe, auf die nach dem Öffnen der Sockets abgesenkt wird (`User`/`Group`, Standard `nobody`). |
| Chroot Directory | Verzeichnis, in das nach dem Start gewechselt wird (`Chroot`). Leer lassen, sofern nicht ausdrücklich benötigt. |
| Open Files Limit | Maximale Anzahl offener Dateideskriptoren, 50–10000 (`OpenFilesLimit`, Standard 1024). Bei vielen parallelen Verbindungen erhöhen. |
| Connection Idle Timeout | Sekunden ohne Aktivität, nach denen eine Verbindung geschlossen wird (`ConnIdleTimeout`, Standard 120). |
| Expired Connection Check Period | Intervall in Sekunden, in dem nach abgelaufenen Verbindungen gesucht wird (`ExpiredConnCheckPeriod`, Standard 10). |

## Reiter "Listeners"

Jedes Feld erzeugt – sofern gefüllt – eine `ProxySpec`-Zeile für `127.0.0.1`
und `::1` mit dem angegebenen Port. Leere Felder erzeugen keinen Listener.

| Feld | Protokoll / Zweck | Standard |
|---|---|---|
| TCP Port | Unverschlüsselte TCP-Verbindungen (Durchleitung ohne TLS-Behandlung). | – |
| SSL Port | Generische SSL/TLS-Verbindungen. | – |
| HTTP Port | HTTP-Verbindungen (inkl. Header-Bearbeitung, siehe [HTTP](#reiter-http)). | `8080` |
| HTTPS Port | HTTPS-Verbindungen – der übliche Listener für Web-Verkehr. | `8443` |
| POP3 Port / SMTP Port | POP3 bzw. SMTP mit STARTTLS-Unterstützung. | – |
| POP3S Port / SMTPS Port | POP3S bzw. SMTPS mit implizitem TLS. | – |
| AutoSSL Port | Erkennt anhand der ersten Bytes selbstständig, ob TLS gesprochen wird, und schaltet entsprechend um. | – |

## Reiter "TLS / Certificates"

| Feld | Beschreibung |
|---|---|
| Minimum Protocol / Maximum Protocol | Zulässiger Bereich der SSL/TLS-Versionen (`MinSSLProto`/`MaxSSLProto`), Standard TLS 1.0 bis TLS 1.3. Für aktuelle Umgebungen empfiehlt sich TLS 1.2 als Minimum. |
| Cipher List | OpenSSL-Cipherliste für TLS ≤ 1.2 (`Ciphers`), Standard `ALL:-aNULL`. |
| TLS 1.3 Ciphersuites | Ciphersuites für TLS 1.3 (`CipherSuites`). |
| ECDH Curve | Kurve für Ephemeral ECDH (`ECDHCurve`), Standard `prime256v1`. |
| DH Parameters File | Pfad zu einer PEM-Datei mit DH-Gruppenparametern (`DHGroupParams`). |
| Leaf Key Size | RSA-Schlüssellänge der im Betrieb erzeugten Zertifikate (`LeafKeyRSABits`), Standard 2048. Größere Schlüssel kosten deutlich mehr CPU-Zeit pro Verbindung. |
| Leaf CRL URL | CRL-Verteilpunkt, der in die erzeugten Zertifikate eingetragen wird (`LeafCRLURL`). |
| Leaf Certificate Directory | Verzeichnis mit vorbereiteten Zertifikaten, die anhand des Common Name verwendet werden, statt neue zu erzeugen (`LeafCertDir`). |
| Write Generated Certificates To | Ablage für Kopien der dynamisch erzeugten Zertifikate (`WriteGenCertsDir`). |
| Write All Certificates To | Ablage für Kopien aller gesehenen Zertifikate (`WriteAllCertsDir`). |
| Allow SSL Compression | Erlaubt TLS-Kompression (`SSLCompression`). Standardmäßig aus – Kompression ermöglicht CRIME-artige Angriffe. |
| Deny OCSP Requests | Beantwortet OCSP-Anfragen mit `tryLater` statt sie weiterzuleiten (`DenyOCSP`). |
| Passthrough On Failure | Lässt Verbindungen, die nicht aufgebrochen werden können, unverändert durch, statt sie zu verwerfen (`Passthrough`). Erhöht die Verträglichkeit mit Pinning-Zielen, verringert aber die Abdeckung. |
| Verify Peer Certificates | Prüft die Zertifikate der Zielserver gegen den System-CA-Speicher (`VerifyPeer`, Standard aktiv). |
| Allow SNI/Host Mismatch | Trägt die angefragte SNI auch dann in das erzeugte Zertifikat ein, wenn sie nicht zum Zertifikat des Ziels passt (`AllowWrongHost`). |
| Validate Protocol | Prüft, ob das tatsächlich gesprochene Protokoll zur ProxySpec passt (`ValidateProto`). |
| Max HTTP Header Size | Maximale Größe der verarbeiteten HTTP-Header in Byte (`MaxHTTPHeaderSize`, Standard 8192). |

## Reiter "HTTP"

| Feld | Beschreibung |
|---|---|
| Remove Accept-Encoding Header | Entfernt den `Accept-Encoding`-Header, damit HTTP-Inhalte unkomprimiert und damit lesbar/auswertbar bleiben (`RemoveHTTPAcceptEncoding`, Standard aktiv). |
| Remove Referer Header | Entfernt den `Referer`-Header (`RemoveHTTPReferer`, Standard aktiv). |

## Reiter "Authentication"

| Feld | Beschreibung |
|---|---|
| Require User Authentication | Verlangt eine Anmeldung, bevor Verbindungen behandelt werden (`UserAuth`). |
| User Database Path | Pfad zur SQLite-Datenbank mit Benutzern/Sitzungen (`UserDBPath`). |
| User Idle Timeout | Sekunden Inaktivität, nach denen eine Sitzung abläuft (`UserTimeout`, Standard 300). |
| Login Redirect URL | URL, auf die nicht angemeldete Nutzer umgeleitet werden (`UserAuthURL`). |
| Divert Users | Kommaliste von bis zu 50 Benutzernamen, deren Verbindungen immer aufgebrochen werden (`DivertUsers`). |
| Pass Users | Kommaliste von bis zu 50 Benutzernamen, deren Verbindungen immer unverändert durchgelassen werden (`PassUsers`). |

Die Felder unterhalb von *Require User Authentication* werden nur in die
Konfiguration geschrieben, wenn dieser Schalter aktiv ist. *Divert Users* und
*Pass Users* werden unabhängig davon geschrieben.

## Reiter "Mirroring"

| Feld | Beschreibung |
|---|---|
| Mirror Interface | Interface, auf das der entschlüsselte Datenstrom gespiegelt wird (`MirrorIf`) – z. B. für eine Auswertung durch ein separates IDS/IPS. |
| Mirror Target Address | Ziel-MAC-/IP-Adresse für die Spiegelung (`MirrorTarget`). Bleibt leer, wenn auf ein Dummy-Interface gespiegelt wird. |

## Reiter "Logging"

| Feld | Beschreibung |
|---|---|
| Connection Log | Eine Zusammenfassungszeile pro Verbindung in `/var/db/sslproxy/connect.log` (`ConnectLog`, Standard aktiv). |
| Content Log | Vollständiger Verbindungsinhalt in Einzeldateien unter `/var/db/sslproxy` (`ContentLogPathSpec`). |
| PCAP Log | Vollständiger Verbindungsinhalt als PCAP-Dateien unter `/var/db/sslproxy` (`PcapLogPathSpec`) – direkt in Wireshark auswertbar. |
| Log Local Process Info | Ermittelt und protokolliert den lokalen Prozess einer Verbindung, soweit unterstützt (`LogProcInfo`). |
| Log TLS Master Keys | Schreibt die Session-Master-Keys im `SSLKEYLOGFILE`-Format nach `/var/db/sslproxy/masterkeys.log` (`MasterKeyLog`), für die nachträgliche Entschlüsselung in Wireshark. |
| Log Statistics | Protokolliert regelmäßig Verbindungsstatistiken nach Syslog (`LogStats`, Standard aktiv). |
| Statistics Period | Intervall der Statistikausgabe in Sekunden (`StatsPeriod`, Standard 1). |

> **Hinweis:** Content-, PCAP- und Masterkey-Logs wachsen schnell und enthalten
> Klartextdaten. Sie sollten nur zeitlich begrenzt für konkrete Analysen
> aktiviert und danach über den [Reset-Button](#reiter-content-und-reset)
> geleert werden. In den Hilfetexten einiger Felder ist noch `/var/log/sslproxy`
> genannt – tatsächlich schreibt die generierte Konfiguration alle Logs nach
> `/var/db/sslproxy`.

## Reiter "Advanced"

| Feld | Beschreibung |
|---|---|
| OpenSSL Engine | Name einer OpenSSL-Engine, die aktiviert und standardmäßig verwendet wird (`OpenSSLEngine`). |
| Additional Configuration | Rohe `sslproxy.conf`-Direktiven, die unverändert an das Ende der generierten Konfiguration angehängt werden. |

**Additional Configuration** ist die Erweiterungsschnittstelle für alles, was
die Reiter nicht abbilden – etwa `Include`, `Define`, `PassSite` (Ausnahmen vom
Aufbruch) sowie strukturierte `ProxySpec`- und `FilterRule`-Blöcke, mit denen
sich Divert-Ports, abweichende Lauschadressen und feingranulare Regeln pro
Listener definieren lassen. Die Direktiven werden ohne Prüfung übernommen: Ein
Syntaxfehler verhindert den Start des Dienstes.

## Reiter "Content" und Reset

Der Reiter **Content** zeigt alle fünf Sekunden aktualisiert den Inhalt von
`/var/db/sslproxy` (Verzeichnislisting). So lässt sich prüfen, ob überhaupt
Verbindungen ankommen und wie stark die Logs wachsen. Die Dateien selbst werden
zur weiteren Analyse per SCP von der Firewall geholt.

Der Button **Reset** unten rechts löscht nach einer Rückfrage alle `*.log`- und
`*.pcap`-Dateien in `/var/db/sslproxy` und lädt den Dienst anschließend neu.
Der Button **Save** speichert alle Reiter gemeinsam und startet den Dienst neu.

## Dateien und Pfade

| Pfad | Inhalt |
|---|---|
| `/usr/local/etc/sslproxy.conf` | Generierte Konfiguration (aus allen Reitern). Nicht manuell bearbeiten – wird bei jedem Speichern überschrieben. |
| `/usr/local/etc/sslproxy-ca.crt` / `-ca.key` | Aus der gewählten Root CA exportiertes Zertifikat und Schlüssel (Modus 0600). Werden bei jedem Start/Neustart neu geschrieben. |
| `/usr/local/etc/sslproxy-leaf-cert.pem` / `-leaf-key.pem` | Exportiertes festes Leaf-Zertifikat, falls konfiguriert. |
| `/usr/local/etc/sslproxy-client-cert.pem` / `-client-key.pem` | Exportiertes Client-Zertifikat, falls konfiguriert. |
| `/var/db/sslproxy/` | Connect-Log, Content-Logs, PCAPs, Masterkey-Log. Wird beim Start automatisch angelegt. |
| `/etc/rc.conf.d/sslproxy` | Generiert, enthält nur `sslproxy_enable`. |
| `/var/run/sslproxy.pid` | PID-Datei, über die OPNsense den Dienststatus ermittelt. |

Auf der Kommandozeile stehen die configd-Aktionen
`configctl sslproxy start|stop|restart|status|content|resetdb` bereit. `start`
und `restart` exportieren zuvor automatisch die Zertifikate.

## Berechtigungen

Alle Seiten und die zugehörige API liegen unter den Mustern `ui/sslproxy/*`
bzw. `api/sslproxy/*`. Damit weitere Benutzer/Benutzergruppen (neben `root`)
darauf zugreifen dürfen, muss ihnen unter **System → Zugriff → Gruppen** die
Berechtigung **"Service: SSLproxy"** zugewiesen werden.

## Support / Fehlerbehebung

- **Der Dienst startet nicht:** Häufigste Ursachen sind eine fehlende Auswahl
  unter **Root CA** (die exportierten CA-Dateien bleiben dann leer), eine
  fehlerhafte Eingabe unter **Additional Configuration** oder ein bereits
  belegter Listener-Port. Zur Diagnose die Konfiguration im Vordergrund
  starten:

  ```
  /usr/local/bin/sslproxy -f /usr/local/etc/sslproxy.conf -D
  ```

  SSLproxy gibt dabei die eingelesene Konfiguration sowie Fehler direkt aus;
  Abbruch mit `Ctrl-C`. Zusätzliche Meldungen finden sich unter **System →
  Protokolldateien → Allgemein**.
- **Keine Dateien unter `/var/db/sslproxy` bzw. im Reiter Content:** Es kommt
  kein Verkehr an. Die Listener binden nur auf `127.0.0.1`/`::1`; ohne
  NAT-Port-Weiterleitung auf diese Adresse und den konfigurierten Port sieht
  SSLproxy keine Verbindungen.
- **Zertifikatswarnungen auf den Clients:** Das CA-Zertifikat ist auf dem
  Client nicht als vertrauenswürdig installiert, oder es wurde eine andere CA
  ausgewählt als verteilt. Manche Anwendungen (Browser mit eigenem
  Zertifikatsspeicher, Java, Python-Tools) pflegen einen separaten Trust Store,
  der zusätzlich versorgt werden muss.
- **Einzelne Anwendungen funktionieren nicht mehr:** Typisch für Certificate
  Pinning. Betroffene Ziele über die NAT-Regeln vom Aufbruch ausnehmen oder
  `PassSite`-Direktiven unter *Additional Configuration* verwenden; alternativ
  **Passthrough On Failure** aktivieren.
- **Hohe CPU-Last:** TLS-Aufbruch ist rechenintensiv. **Leaf Key Size**
  reduzieren (z. B. 2048 statt 4096), Content-/PCAP-Logging deaktivieren und
  den umgeleiteten Verkehr über die NAT-Regeln auf das Nötige begrenzen.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr` dieses
  Verzeichnisses dokumentiert.
