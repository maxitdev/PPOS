# WAF (mod_security2 / OWASP CRS) im Detail

*[English version](os-apache-maxit-waf.en.md)*

Dieses Dokument beschreibt die Web-Application-Firewall-Integration im
Apache-Plugin: wie sie unter der Haube zusammenhängt, wie sie im
Tagesbetrieb bedient wird, wie man sie bei Problemen untersucht, wie man
eigene Regeln schreibt und wie man mit Falsch-Positiven umgeht. Für die
grundlegende "was macht diese Checkbox"-Referenz siehe den
[WAF-Abschnitt](os-apache-maxit.md#waf) des Haupt-READMEs.

- [Wie es implementiert ist](#wie-es-implementiert-ist)
  - [Paket und Dateien auf der Festplatte](#paket-und-dateien-auf-der-festplatte)
  - [Was wann generiert wird](#was-wann-generiert-wird)
  - [Vergabe von Rule-IDs](#vergabe-von-rule-ids)
  - [Rangfolge: global vs. vhost vs. Location](#rangfolge-global-vs-vhost-vs-location)
- [Verwendung](#verwendung)
  - [Aktivieren](#aktivieren)
  - [Rulesets: die Stellschrauben](#rulesets-die-stellschrauben)
  - [Custom Rules](#custom-rules)
- [Fehlersuche](#fehlersuche)
  - [Die WAF zeigt überhaupt keine Wirkung](#die-waf-zeigt-überhaupt-keine-wirkung)
  - [Apache startet/lädt nicht neu, wenn die WAF aktiviert ist](#apache-startetlädt-nicht-neu-wenn-die-waf-aktiviert-ist)
  - [Das Audit-Log lesen](#das-audit-log-lesen)
  - [Herausfinden, welche Regel eine Anfrage blockiert hat](#herausfinden-welche-regel-eine-anfrage-blockiert-hat)
- [Eigene Regeln schreiben](#eigene-regeln-schreiben)
  - [Aufbau von SecRule](#aufbau-von-secrule)
  - [Phasen](#phasen)
  - [Nützliche Variablen](#nützliche-variablen)
  - [Nützliche Operatoren](#nützliche-operatoren)
  - [Häufige Aktionen](#häufige-aktionen)
  - [Beispiele](#beispiele)
  - [Eine Rule-ID für Custom Rules wählen](#eine-rule-id-für-custom-rules-wählen)
- [Umgang mit Falsch-Positiven](#umgang-mit-falsch-positiven)
  - [Der Eskalationspfad](#der-eskalationspfad)
  - [Eine bestimmte Regel für eine bestimmte Anfrage ausschließen](#eine-bestimmte-regel-für-eine-bestimmte-anfrage-ausschließen)
  - [Application Exclusion Packages](#application-exclusion-packages)
  - [Whitelisting nach Quell-IP](#whitelisting-nach-quell-ip)
  - [Paranoia Level und Anomaly Thresholds einstellen](#paranoia-level-und-anomaly-thresholds-einstellen)

## Wie es implementiert ist

### Paket und Dateien auf der Festplatte

Die WAF-Engine selbst (`mod_security2`) und das OWASP Core Rule Set kommen
aus dem FreeBSD-Paket `ap24-mod_security`, das automatisch als
Plugin-Abhängigkeit mitinstalliert wird -- nichts muss separat installiert
werden. Das Plugin bündelt oder lädt selbst keine Regeln herunter (anders
als der NAXSI-Regel-Downloader des `www/nginx`-Plugins); es verweist
Apache lediglich auf das, was das Paket unter
`/usr/local/etc/apache24/modsecurity.d/owasp-crs/` ablegt.

Vom Plugin verwaltete Laufzeit-Dateien:

| Pfad | Zweck |
|---|---|
| `/usr/local/etc/apache24/modsecurity.d/owasp-crs/crs-setup.conf` | CRS-Standardeinstellungen, vom Paket bereitgestellt |
| `/usr/local/etc/apache24/modsecurity.d/owasp-crs/rules/*.conf` | Die eigentlichen OWASP-CRS-Regeldateien, vom Paket bereitgestellt |
| `/var/db/apache/modsecurity/` | `SecDataDir` -- der persistente Zustand von mod_security (IP-Sammlungen, die von manchen CRS-Regeln genutzt werden) |
| `/var/log/apache/modsec_audit.log` | `SecAuditLog` -- das Audit-Log, siehe [Das Audit-Log lesen](#das-audit-log-lesen) |

### Was wann generiert wird

Drei Templates arbeiten zusammen und werden bei jedem Apply frisch in
`httpd.conf` erzeugt (siehe [Änderungen übernehmen](os-apache-maxit.md#änderungen-übernehmen)
im Haupt-README):

- **`waf.conf`** -- wird einmal, global, von `httpd.conf` eingebunden.
  Erzeugt nur dann überhaupt etwas, wenn **Enable WAF** eingeschaltet
  ist. Richtet `SecDataDir`, `SecAuditLog` und Ähnliches ein und bindet
  anschließend die CRS-Basiskonfiguration sowie die Regeldateien ein.
  Dies ist die einzige Stelle, an der `SecRuleEngine` **nicht** gesetzt
  wird -- ob die Engine an oder aus ist, wird stets je vhost/Location
  entschieden (siehe unten), niemals global.
- **`waf_server.conf`** -- wird einmal je HTTP Server (vhost) innerhalb
  seines `<VirtualHost>`-Blocks eingebunden. Ist diesem vhost ein **WAF
  Ruleset** zugewiesen, schaltet es `SecRuleEngine` ein (oder
  `DetectionOnly`, falls beim Ruleset Learning Mode aktiv ist) und
  erzeugt Paranoia Level/Thresholds/Exclusions/Whitelist als
  `SecAction`-Tuning-Direktiven, gefolgt vom rohen Text zugewiesener
  **WAF Custom Rules**. Ist kein Ruleset zugewiesen, wird für diesen
  vhost explizit `SecRuleEngine Off` gesetzt.
- **`waf_location.conf`** -- wird einmal je Location eingebunden, die ein
  eigenes **WAF Ruleset** zugewiesen hat, innerhalb des jeweiligen
  `<Location>`-/`<LocationMatch>`-Blocks. Gleicher Aufbau wie
  `waf_server.conf`, wird aber nur erzeugt, wenn die Location das
  Ruleset tatsächlich überschreibt (ein leeres *WAF Ruleset*-Feld an
  einer Location bedeutet "übernimmt, was der vhost entschieden hat",
  siehe [Rangfolge](#rangfolge-global-vs-vhost-vs-location)).

Alle drei erzeugen nichts (No-Op), solange **Enable WAF** aus ist, und
alle drei sind in `<IfModule security2_module>` eingebettet, damit ein
Build ohne mod_security weiterhin funktioniert (nur eben ohne WAF).

### Vergabe von Rule-IDs

Jede CRS-/mod_security-Regel benötigt eine numerische ID, und IDs müssen
über die gesamte Konfiguration hinweg eindeutig sein. Dieses Plugin
leitet sie deterministisch aus der UUID des jeweiligen Objekts ab, statt
einen laufenden Zähler zu führen, damit sich IDs nicht verschieben, wenn
unbeteiligte vhosts hinzugefügt/entfernt werden:

```
waf_server.conf:   15.000.000 + (erste 6 Hex-Zeichen der vhost-UUID)     -> Bereich 15.000.000-31.777.215
waf_location.conf: 40.000.000 + (erste 6 Hex-Zeichen der Location-UUID)  -> Bereich 40.000.000-56.777.215
```

Innerhalb dieser Basis werden feste Offsets verwendet:

| Offset | Zweck |
|---|---|
| `+0` | Paranoia Level |
| `+1` | Inbound Anomaly Threshold |
| `+2` | Outbound Anomaly Threshold |
| `+5` | Whitelist-Regel für Quell-IPs |
| `+11` .. `+17` | eine je ausgewähltem Application Exclusion Package |

Kollisionen sind astronomisch unwahrscheinlich (6 Hex-Zeichen = 16,7 Mio.
mögliche Werte je vhost/Location), aber nicht mathematisch ausgeschlossen;
`setup.php` führt nach jeder Neuerzeugung `apachectl configtest` aus, um
genau dies (und alles andere) abzufangen, bevor Apache neu geladen wird --
siehe
[Apache startet/lädt nicht neu, wenn die WAF aktiviert ist](#apache-startetlädt-nicht-neu-wenn-die-waf-aktiviert-ist).
Eigene [Custom Rules](#eine-rule-id-für-custom-rules-wählen) benötigen
eine ID außerhalb beider Bereiche.

### Rangfolge: global vs. vhost vs. Location

1. **Enable WAF** (General Settings) ist der Hauptschalter. Ist er aus,
   erzeugen `waf.conf`/`waf_server.conf`/`waf_location.conf` gar nichts --
   mod_security kann trotzdem geladen sein (falls eine andere
   Konfiguration darauf verweist), aber dieses Plugin gibt dann
   überhaupt keine `SecRuleEngine`-/Tuning-Direktiven aus.
2. Ist die WAF aktiviert, entscheidet jeder **HTTP Server** seine eigene
   Grundeinstellung: ein zugewiesenes Ruleset bedeutet
   `SecRuleEngine On` (oder `DetectionOnly`) mit dessen Tuning; kein
   Ruleset bedeutet für diesen ganzen vhost explizit
   `SecRuleEngine Off`.
3. Jede **Location** *übernimmt* die Entscheidung des vhosts, sofern sie
   nicht ein eigenes **WAF Ruleset** setzt -- in dem Fall gibt
   `waf_location.conf` `SecRuleEngine`/Tuning innerhalb dieses konkreten
   `<Location>`-Blocks erneut aus und überschreibt die Einstellung des
   vhosts nur für diesen Pfad. Es gibt keine Möglichkeit, eine Location
   auf "Off" zurückzusetzen, wenn der vhost auf "On" steht, außer ihr ein
   eigenes Ruleset mit als unbedenklich eingestuften Regeln zu geben,
   oder ein Ruleset mit aktivem Learning Mode.
4. **Custom Rules** wirken additiv auf der Ebene, auf der sie zugewiesen
   sind (vhost und/oder Location) -- sie ersetzen nicht die CRS-Regeln
   des Rulesets, sondern laufen zusätzlich dazu.

## Verwendung

### Aktivieren

1. Mindestens ein **WAF Ruleset** anlegen (Services &rarr; Apache &rarr;
   WAF &rarr; Rulesets tab) -- schon `Description: default` mit allen
   übrigen Feldern auf Standardwerten reicht zum Start.
2. **General Settings &rarr; WAF tab** &rarr; **Enable WAF** &rarr; Apply.
3. Dieses Ruleset einem **HTTP Server** (oder einer bestimmten
   **Location**) auf der Reverse-Proxy-Seite zuweisen &rarr; Apply.

Es wird nichts geprüft oder blockiert, solange nicht alle drei Punkte
erfüllt sind. Das ist beabsichtigt -- siehe
[Die WAF zeigt überhaupt keine Wirkung](#die-waf-zeigt-überhaupt-keine-wirkung),
falls trotz aller drei Schritte nichts blockiert wird.

### Rulesets: die Stellschrauben

Ein Ruleset ist ein benanntes Bündel von CRS-Tuning, wiederverwendbar über
viele vhosts/Locations hinweg (die vollständige Feldliste steht im
[Benutzerhandbuch](os-apache-maxit.md#waf)). Die zwei meistgenutzten:

- **Learning Mode** -- erzwingt `SecRuleEngine DetectionOnly`: jede Regel
  läuft weiterhin und jeder Treffer wird weiterhin im Audit-Log
  protokolliert, aber es wird nie etwas blockiert. Bei einem neuen
  Ruleset stets hiermit beginnen.
- **Paranoia Level** -- CRS liefert vier Regelstufen; Stufe 1 (Standard)
  ist die Basis mit den wenigsten Falsch-Positiven, Stufe 4 ist maximal
  streng und markiert bei den meisten Anwendungen viel legitimen
  Verkehr. Schrittweise erhöhen und dabei jedes Mal das Audit-Log
  beobachten (siehe
  [Paranoia Level einstellen](#paranoia-level-und-anomaly-thresholds-einstellen)).

### Custom Rules

Eine Custom Rule ist eine (oder mehrere) rohe `SecRule`-/`SecAction`-
Zeile(n), genau so eingetippt, wie sie in einer `.conf`-Datei stehen
würde, direkt einem vhost oder einer Location zugewiesen (nicht Teil
eines Rulesets). Damit lässt sich alles abdecken, was CRS nicht
mitbringt: ein bestimmtes bekanntes Angriffsmuster blockieren, einen
Pfad freigeben, den das eigene CRS-Tuning nicht sauber ausdrücken kann,
oder eine Schwachstelle virtuell patchen, während auf einen echten Fix
in der Anwendung gewartet wird. Siehe
[Eigene Regeln schreiben](#eigene-regeln-schreiben).

## Fehlersuche

### Die WAF zeigt überhaupt keine Wirkung

Diese Checkliste der Reihe nach durchgehen -- jeder Schritt setzt den
vorherigen voraus:

1. Ist **Enable WAF** wirklich eingeschaltet (General Settings &rarr; WAF
   tab)? Das wird leicht übersehen, da das Zuweisen eines Rulesets zu
   einem vhost ein separater Schritt ist und nicht warnt, falls der
   globale Schalter noch aus ist.
2. Ist dem vhost (oder der konkreten Location) tatsächlich ein **WAF
   Ruleset** zugewiesen? Ein HTTP Server ohne Ruleset bekommt explizit
   `SecRuleEngine Off` -- es gibt kein implizites "standardmäßig an".
3. Ist bei diesem Ruleset versehentlich **Learning Mode** aktiv
   geblieben? Alles wird als Treffer im Audit-Log protokolliert, aber
   nichts blockiert -- das sieht von außen identisch aus wie "die WAF
   läuft nicht".
4. Wurde nach der letzten Änderung **Apply** geklickt? In
   `/usr/local/etc/apache24/httpd.conf` (oder mit `configctl apache
   test_config`) prüfen, ob die erwarteten `SecRuleEngine`-/
   `SecAction`-Zeilen für diesen vhost tatsächlich vorhanden sind.
5. Ist `mod_security2` tatsächlich geladen? `apachectl -M | grep
   security2` in der Konsole/SSH-Sitzung der Firewall. Fehlt es, ist
   entweder das Paket `ap24-mod_security` nicht installiert, oder es
   legt seinen Modul-Lade-Schnipsel nicht unter
   `/usr/local/etc/apache24/modules.d/` ab -- jeder
   `<IfModule security2_module>`-Block in der von diesem Plugin
   erzeugten Konfiguration wird dann stillschweigend zum No-Op, weshalb
   Schritt 5 eine direkte Prüfung braucht, statt es aus der
   Konfigurationsdatei zu erschließen.

### Apache startet/lädt nicht neu, wenn die WAF aktiviert ist

- **`SecDataDir ... is not a directory` / Berechtigungsfehler bei
  `/var/db/apache/modsecurity`** -- `setup.php` legt dieses Verzeichnis
  bei jedem Lauf an; fehlt es, ist `setup.php` selbst nicht gelaufen.
  Siehe den Hinweis im Haupt-README zum `apache24_setup`-rc.conf.d-Hook --
  ein rohes `/usr/local/etc/rc.d/apache24 <cmd>` von der Kommandozeile
  löst ihn nur für `start`/`restart`/`reload` aus, nie für `configtest`.
- **Doppelte Rule-ID** -- extrem unwahrscheinlich (siehe
  [Vergabe von Rule-IDs](#vergabe-von-rule-ids)), aber falls `apachectl
  configtest` eine doppelte `id:` meldet, handelt es sich fast sicher um
  eine [Custom Rule](#eine-rule-id-für-custom-rules-wählen) mit einer ID,
  die mit den abgeleiteten Bereichen kollidiert. Sie außerhalb von
  `15.000.000-56.777.215` ansiedeln.
- **`Unable to open ... crs-setup.conf` / CRS-Regeln nicht gefunden** --
  das Paket `ap24-mod_security` hat CRS nicht dort abgelegt, wo dieses
  Plugin es erwartet (`etc/apache24/modsecurity.d/owasp-crs/`); beide
  Includes sind `IncludeOptional`, das scheitert also still statt den
  Start zu blockieren -- es entsteht lediglich eine Engine ohne
  geladene CRS-Regeln. Mit
  `ls /usr/local/etc/apache24/modsecurity.d/owasp-crs/` prüfen.

### Das Audit-Log lesen

`/var/log/apache/modsec_audit.log`, `SecAuditEngine RelevantOnly` (nur
Anfragen, die eine Regel ausgelöst oder einen Fehler zurückgegeben haben,
werden protokolliert, nicht jede Anfrage), `SecAuditLogParts ABIJDEFHZ`
(Request-/Response-Header und -Bodys plus der Trailer mit den
getroffenen Regeln). Jeder Eintrag ist ein mehrteiliger, durch einen
Grenzmarker getrennter Datensatz; die im Alltag wichtigsten Teile:

- **Part A** -- Zeitstempel, Verbindungsinformationen, eine eindeutige
  Transaktions-ID.
- **Part B** -- die Request-Zeile und Header, wie sie der Client
  gesendet hat.
- **Part F** -- die Response-Header, die Apache gesendet hätte.
- **Part H** -- der Trailer: welche Regel(n) getroffen haben, die
  Meldung, der Anomaly Score, und (entscheidend) ob die Transaktion
  `Intercepted` wurde oder (Learning Mode) nur protokolliert wurde.

Live mitverfolgen, während das Problem reproduziert wird:

```
tail -f /var/log/apache/modsec_audit.log
```

oder nach einer bestimmten Transaktions-ID filtern, sobald eine aus einer
403-Antwort/den Browser-Entwicklertools vorliegt.

### Herausfinden, welche Regel eine Anfrage blockiert hat

1. Die Anfrage reproduzieren, die ungefähre Uhrzeit notieren.
2. `grep -B5 'Intercepted' /var/log/apache/modsec_audit.log` (oder das
   Log live mitverfolgen), um den passenden Part-H-Trailer zu finden --
   er listet jede getroffene Rule-ID und die endgültige Entscheidung.
3. Die Meldung zu jedem Treffer nennt meist die CRS-Regel (z. B.
   `[id "942100"] [msg "SQL Injection Attack Detected via
   libinjection"]`) -- diese ID wird im nächsten Abschnitt ausgeschlossen,
   nicht die von `waf_rule_base` abgeleiteten Tuning-IDs (die blockieren
   selbst nie etwas, sie setzen nur Variablen).

## Eigene Regeln schreiben

Dieser Abschnitt setzt eine gewisse Vertrautheit mit der `SecRule`-Sprache
von mod_security voraus; es ist ein großes Thema, und das
[ModSecurity Reference Manual](https://github.com/owasp-modsecurity/ModSecurity/wiki/Reference-Manual-(v2.x))
ist die maßgebliche Quelle. Im Folgenden die Teilmenge, die die meisten
Custom-Rule-Bedürfnisse in diesem Plugin abdeckt.

### Aufbau von SecRule

```
SecRule VARIABLE "OPERATOR" "ACTIONS"
```

Beispiel:

```
SecRule REQUEST_HEADERS:User-Agent "@contains BadBot" \
    "id:60000001,phase:1,deny,status:403,log,msg:'Blocked known bad crawler'"
```

- `VARIABLE` -- was untersucht wird (`ARGS`, `REQUEST_URI`, ein
  bestimmter Header, `REQUEST_BODY`, …).
- `OPERATOR` -- wie geprüft wird, mit `@` vorangestellt (`@contains`,
  `@rx` für reguläre Ausdrücke, `@eq`, `@ipMatch`, …). Ohne
  `@`-Präfix bedeutet ein einfacher String-Vergleich.
- `ACTIONS` -- kommagetrennt, immer mit einer eindeutigen `id`.

### Phasen

mod_security verarbeitet eine Anfrage in fünf Phasen; die
`phase`-Aktion einer Custom Rule bestimmt, wann sie läuft (das
entspricht dem **Phase**-Dropdown im Custom-Rules-Grid):

| Phase | Läuft bei | Typische Verwendung |
|---|---|---|
| 1 | Request Headers | Schnelle Prüfungen, bevor der Body überhaupt gelesen wird (Header, URL, Methode) |
| 2 | Request Body | Formularfelder, JSON-/XML-Bodys, Datei-Uploads -- hier liegen die meisten CRS-Regeln |
| 3 | Response Headers | Prüft, was das Backend gleich zurücksenden wird |
| 4 | Response Body | Prüfungen auf Datenlecks, Inspektion des Antwortinhalts |
| 5 | Logging | Läuft unabhängig vom Ausgang der vorherigen Phasen; kann nicht blockieren, nur protokollieren/markieren |

### Nützliche Variablen

| Variable | Erfasst |
|---|---|
| `ARGS` | Alle Request-Parameter (Query-String + Body), nach Wert |
| `ARGS_NAMES` | Alle Namen der Request-Parameter |
| `REQUEST_URI` | Der Request-Pfad + Query-String |
| `REQUEST_HEADERS:Name` | Ein bestimmter Request-Header |
| `REQUEST_COOKIES:name` | Ein bestimmtes Cookie |
| `REQUEST_BODY` | Der rohe Request-Body |
| `RESPONSE_HEADERS:Name` | Ein bestimmter Response-Header (ab Phase 3) |
| `REMOTE_ADDR` | Client-IP |
| `TX.varname` | Eine von einer früheren Regel gesetzte Transaktionsvariable (`setvar`) |

### Nützliche Operatoren

| Operator | Bedeutung |
|---|---|
| `@contains` | Teilstring-Treffer |
| `@rx` | regulärer Ausdruck (PCRE) |
| `@eq` / `@gt` / `@lt` | numerischer Vergleich |
| `@streq` | exakter String-Vergleich |
| `@ipMatch` | IP-/CIDR-Abgleich, kommagetrennte Liste |
| `@pm` | schneller Parallelabgleich gegen eine Liste von Ausdrücken |
| `@detectSQLi` / `@detectXSS` | die eingebauten, libinjection-basierten Detektoren von mod_security -- dieselben, die CRS selbst nutzt |

### Häufige Aktionen

| Aktion | Wirkung |
|---|---|
| `id:NNN` | erforderlich, eindeutige numerische ID (siehe [Eine Rule-ID wählen](#eine-rule-id-für-custom-rules-wählen)) |
| `phase:N` | in welcher Phase diese Regel läuft |
| `deny` | die Anfrage blockieren |
| `status:403` | HTTP-Status, der beim Blockieren gesendet wird |
| `pass` | die Anfrage weiterlaufen lassen (genutzt bei reinen `setvar`-Regeln, die nie blockieren) |
| `log` / `nolog` | ob dieser Treffer ins Audit-Log geschrieben wird |
| `msg:'...'` | für Menschen lesbare Meldung, die im Audit-Log erscheint |
| `t:none` / `t:lowercase` / `t:urlDecode` | Transformation, die vor dem Operator auf die Variable angewendet wird |
| `chain` | der Treffer dieser Regel ist nur zusammen mit der *nächsten* Regel sinnvoll (logisches UND) |
| `ctl:ruleEngine=Off` | die Engine für den Rest dieser Transaktion abschalten (für Whitelisting genutzt, siehe unten) |
| `setvar:tx.varname=value` | eine Transaktionsvariable setzen, die andere Regeln lesen können |

### Beispiele

Einen bestimmten Pfad grundsätzlich blockieren:

```
SecRule REQUEST_URI "@streq /phpmyadmin" \
    "id:60000010,phase:1,deny,status:404,log,msg:'Blocked disallowed path'"
```

Einen großen Upload nur an einem bestimmten Endpunkt erlauben (das
Body-Limit nur für diesen Pfad anheben, überall sonst bleibt der
Standard des vhosts bestehen):

```
SecRule REQUEST_URI "@streq /api/upload" \
    "id:60000011,phase:1,pass,nolog,ctl:requestBodyLimit=52428800"
```

Ein von Rate/Anomaly unabhängiger virtueller Patch für einen bekannten
verwundbaren Parameter in einer bestimmten Anwendung, während auf einen
echten Fix gewartet wird:

```
SecRule ARGS_NAMES "@streq vulnerable_param" \
    "id:60000012,phase:2,chain,deny,status:403,log,msg:'Virtual patch for CVE-XXXX-YYYY'"
    SecRule ARGS:vulnerable_param "@rx \.\./" "t:none"
```

### Eine Rule-ID für Custom Rules wählen

Rule-IDs für Custom Rules müssen zwei bereits von diesem Plugin genutzte
Bereiche meiden: `15.000.000-31.777.215` (vhost-Tuning) und
`40.000.000-56.777.215` (Location-Tuning) -- siehe
[Vergabe von Rule-IDs](#vergabe-von-rule-ids). CRS selbst nutzt IDs
unterhalb von 1.000.000. Alles im Bereich ab `60.000.000` (wie in den
obigen Beispielen) ist ein sicherer, unbenutzter Block; einen eigenen
Startpunkt wählen und für jede neue Custom Rule hochzählen, um
Kollisionen mit den eigenen weiteren Regeln zu vermeiden.

## Umgang mit Falsch-Positiven

### Der Eskalationspfad

Wenn etwas Legitimes blockiert wird, nicht einfach die WAF abschalten.
Stattdessen diese Schritte vom am wenigsten bis zum am stärksten
störenden durchgehen:

1. **Bestätigen, dass es wirklich CRS ist, nicht eine eigene Custom
   Rule** -- zuerst
   [die passende Rule-ID im Audit-Log finden](#herausfinden-welche-regel-eine-anfrage-blockiert-hat).
2. **Nach einem Application Exclusion Package schauen** -- läuft
   WordPress, Nextcloud oder eine andere Anwendung mit passendem Eintrag
   unter **Application Exclusion Packages**, kann dessen Aktivierung
   eine ganze Klasse von Falsch-Positiven in einem Schritt beheben
   (siehe unten).
3. **Nur diese eine Regel für nur diese eine Anfrage ausschließen** --
   der chirurgisch präziseste Fix, siehe unten.
4. **Das Paranoia Level senken** -- nur, wenn viele Falsch-Positive über
   mehrere Paranoia-Level-spezifische Regeln hinweg auftreten, nicht bei
   einem einzelnen Einzelfall.
5. **Learning Mode** bei diesem Ruleset vorübergehend setzen -- nur als
   letztes Mittel, während die obigen Schritte bearbeitet werden, da es
   in der Zwischenzeit auch das Blockieren echter Angriffe stoppt.

### Eine bestimmte Regel für eine bestimmte Anfrage ausschließen

Der von CRS empfohlene, präzise Weg, eine Regel für eine URL zu
unterdrücken (statt global), ist ein `SecRuleRemoveById` (oder
`SecRuleUpdateTargetById`, um einzugrenzen, was eine Regel untersucht,
falls nur ein Parameter das Problem ist), begrenzt auf ein
`<LocationMatch>`, hinzugefügt als [Custom Rule](#custom-rules) an
dieser Location:

```
SecRuleRemoveById 942100
```

Als Custom Rule an der konkreten Location zugewiesen, an der das
Falsch-Positiv auftritt, statt am vhost, hält der Ausschluss so eng wie
das Problem selbst. Tritt das Falsch-Positiv nur bei einem Parameter auf,
statt bei der gesamten Anfrage:

```
SecRuleUpdateTargetById 942100 "!ARGS:comment_body"
```

### Application Exclusion Packages

**Application Exclusion Packages** an einem Ruleset setzt
`tx.<name>-rule-exclusions-enabled=1` für jede ausgewählte Anwendung
(WordPress, Drupal, Nextcloud, cPanel, DokuWiki, phpBB, XenForo). Das
wirkt nur, wenn das passende OWASP-CRS-Exclusion-Plugin-Paket für diese
Anwendung zusätzlich zu den Basis-CRS-Regeln installiert ist -- die
Checkbox allein bewirkt nichts, wenn die entsprechende Regeldatei nicht
vorhanden ist. Mit
`/usr/local/etc/apache24/modsecurity.d/owasp-crs/` auf eine Datei
`plugins/<name>-*` prüfen, bevor man sich darauf verlässt.

### Whitelisting nach Quell-IP

**Whitelisted Source IPs** an einem Ruleset nimmt die WAF für passende
Client-IPs vollständig aus der Transaktion heraus
(`ctl:ruleEngine=Off`) -- dies für vertrauenswürdige interne Scanner,
Monitoring-Systeme oder Admin-Netzwerke nutzen, die die Prüfung umgehen
müssen, nicht als allgemeinen Falsch-Positiv-Fix (es schaltet *jeglichen*
Schutz für diese Quelle ab, nicht nur die fehlauslösende Regel).

### Paranoia Level und Anomaly Thresholds einstellen

CRS bewertet jede Anfrage mit einem Score, statt beim ersten Treffer zu
blockieren; eine Anfrage wird erst tatsächlich blockiert, sobald ihr
kumulativer Anomaly Score den **Inbound/Outbound Anomaly Threshold**
erreicht. Zwei unabhängige Stellschrauben, die oft verwechselt werden:

- **Paranoia Level** bestimmt, *welche Regeln überhaupt laufen* -- Stufe
  1 führt nur die Regeln mit den wenigsten Falsch-Positiven aus, höhere
  Stufen fügen zunehmend strengere (und geräuschvollere) hinzu.
- **Anomaly Threshold** bestimmt, *wie viele/wie schwere Treffer nötig
  sind, bevor blockiert wird*, beim jeweils gewählten Paranoia Level.
  Den Threshold zu erhöhen (Standard ist 5 inbound / 4 outbound, passend
  zu CRS' eigenen Standardwerten) macht denselben Regelsatz toleranter,
  ohne zu ändern, welche Regeln laufen -- nützlich, wenn mehrere
  geringfügige Treffer bei legitimem Verkehr auftreten, die einzeln
  betrachtet keinen Regelausschluss rechtfertigen.

Treten nach Erhöhung des Paranoia Levels viele unterschiedliche
Falsch-Positive auf, ist das ein Zeichen, dass das Level für diese
Anwendung zu hoch ist, statt dass man sich einzeln herausschließen
sollte -- zunächst wieder herabsetzen.
