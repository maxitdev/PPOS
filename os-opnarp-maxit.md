# os-opnarp-maxit – ARP- und NDP-Monitor für OPNsense

`os-opnarp-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG.
Es beobachtet die ARP-Tabelle (IPv4) und die NDP-Tabelle (IPv6) der
Firewall und führt ein Verzeichnis aller Geräte, die im Netz aufgetaucht
sind. Neue Geräte, geänderte MAC-Adressen und Adressen, die zwischen zwei
Geräten hin- und herspringen, erscheinen unter **Services → OPN-Arp**,
werden in eine eigene Logdatei geschrieben und auf Wunsch per Mail über
`os-smtp-cli-maxit` gemeldet.

> **Wichtiger Hinweis:** Das Plugin sieht nur, was die Firewall selbst
> sieht. Ein Gerät taucht erst auf, wenn es mit der Firewall oder über sie
> spricht; Verkehr zwischen zwei Geräten im selben Segment bleibt
> unsichtbar. OPN-Arp ist eine Erkennung, kein Schutz: es blockiert nichts.
> Wer unbekannte Geräte aussperren will, braucht eine
> Netzwerkzugangskontrolle (802.1X) auf den Switches.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Umstieg von früheren Versionen](#umstieg-von-früheren-versionen)
4. [Bedienung](#bedienung)
5. [Die Ereignisse](#die-ereignisse)
6. [Einstellungen](#einstellungen)
7. [Meldung per E-Mail](#meldung-per-e-mail)
8. [Logdatei und Monit](#logdatei-und-monit)
9. [Wie das Plugin arbeitet](#wie-das-plugin-arbeitet)
10. [Berechtigungen](#berechtigungen)
11. [Womit das Plugin verzahnt ist](#womit-das-plugin-verzahnt-ist)
12. [Bekannte Grenzen](#bekannte-grenzen)
13. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über
  das `os-opnarp-maxit` bezogen wird. Entwickelt für **OPNsense 26.7**.
- Keine weiteren Pakete. Der Monitor ist ein Python-Dienst und nutzt nur,
  was OPNsense mitbringt: Python 3, SQLite und `py-netaddr` für die
  Herstellernamen.
- Optional `os-smtp-cli-maxit` für die Meldung per Mail. Es wird nicht
  automatisch mitinstalliert; ohne das Plugin bleibt es bei Anzeige und
  Logdatei.

## Installation

1. Unter **System → Firmware → Plugins** nach `os-opnarp-maxit` suchen und
   installieren.
2. Unter **Services → OPN-Arp → Monitor** auf den Reiter **Settings**
   wechseln, **Enable** setzen, bei Bedarf die Interfaces auswählen und
   speichern.
3. Der erste Durchlauf nimmt alle Geräte auf, die gerade in den Tabellen
   stehen, ohne sie zu melden, und markiert sie als bekannt. Gemeldet wird
   erst, was danach hinzukommt oder sich ändert. Im Log steht dazu eine
   Zeile `learned … stations with … addresses`.

## Umstieg von früheren Versionen

Versionen vor 2.0 bestanden aus einem Bash-Skript, das alle fünf Sekunden
`arp -an` und `ndp -an` abfragte und neue Paare in das Systemlog schrieb.
Beim Update auf 2.0 ändert sich:

- **Die bisherige Logzeile bleibt erhalten.** Beim Update wird
  **Compatible log line** eingeschaltet. Der Monitor schreibt dann für jedes
  zum ersten Mal gesehene Paar aus IP- und MAC-Adresse weiterhin
  `New IPv4/MAC pair seen: (<IP>)<MAC>` bzw.
  `New IPv6/MAC pair seen <IP><MAC>` in das Systemlog
  (`/var/log/system/latest.log`), genau wie das Bash-Skript. Bestehende
  Monit-Prüfungen und Logfilter laufen ohne Änderung weiter.
- **Zusätzlich** gibt es die neuen Meldungen in einer eigenen Logdatei unter
  **Services → OPN-Arp → Log File** (`/var/log/opnarp/`), siehe
  [Logdatei und Monit](#logdatei-und-monit). Wer seine Prüfungen auf die
  neuen Meldungen oder die eingebaute Mail umgestellt hat, kann
  **Compatible log line** ausschalten.
- **Die Interfaces werden aus der Liste der zugewiesenen Interfaces
  gewählt** statt als Gerätename (`igb0`) eingetippt. Bereits eingetragene
  Gerätenamen werden beim Update übernommen, sofern sie einem
  zugewiesenen Interface gehören; alle anderen fallen weg.
- **Die alten Daten aus `/tmp` werden nicht übernommen.** Der neue Monitor
  lernt beim ersten Start den aktuellen Bestand neu, ohne ihn zu melden.
- `bash` wird nicht mehr benötigt.

## Bedienung

Die Seite **Services → OPN-Arp → Monitor** hat drei Reiter.

### Stations

Ein Eintrag je MAC-Adresse:

| Spalte | Bedeutung |
| --- | --- |
| MAC address | die Hardware-Adresse des Geräts |
| Vendor | Hersteller laut IEEE-OUI-Verzeichnis; **(private address)** bei lokal verwalteten Adressen, wie sie Smartphones und Notebooks als private WLAN-Adresse verwenden |
| Addresses | alle IPv4- und IPv6-Adressen, unter denen das Gerät gesehen wurde, die zuletzt gesehene zuerst |
| Interface | das Interface, auf dem das Gerät zuletzt gesehen wurde |
| First seen / Last seen | erster und letzter Nachweis |
| State | **new** für Geräte, die noch niemand bestätigt hat, **known** für bestätigte |

Mit dem Haken-Knopf wird ein Gerät als bekannt markiert, mit dem
Pfeil-Knopf wieder als neu. **Mark all as known** bestätigt alle auf
einmal. **Only new stations** blendet die bestätigten aus, so bleibt
übrig, was noch angesehen werden muss.

Der Papierkorb vergisst ein Gerät mitsamt seinen Adressen. Taucht es
wieder auf, wird es erneut als neues Gerät gemeldet. Das ist der Weg für
Geräte, die das Netz endgültig verlassen haben.

### Events

Alle Ereignisse, das neueste zuerst, mit Zeit, Art, IP-Adresse, neuer und
vorheriger MAC-Adresse, Hersteller und Interface. Die Auswahl oben
beschränkt die Liste auf eine Art. Hier stehen alle Ereignisse, auch die
Arten, die unter **Report** abgewählt sind. Angezeigt werden höchstens die
letzten 20 000.

### Settings

Siehe [Einstellungen](#einstellungen). **Reset database** löscht alle
Geräte, Adressen und Ereignisse; der Monitor lernt den aktuellen Bestand
anschließend neu, ohne ihn zu melden.

## Die Ereignisse

Ein Gerät ist eine MAC-Adresse, eine Adresse ist eine IP-Adresse. Für jede
Adresse merkt sich der Monitor die beiden Geräte, die sie zuletzt hatten.

| Ereignis | Bedeutung | typische Ursache |
| --- | --- | --- |
| New station | eine MAC-Adresse, die noch nie gesehen wurde | neues Gerät, Besucher, private WLAN-Adresse eines Smartphones |
| New address | ein bekanntes Gerät mit einer Adresse, die noch keinem Gerät gehörte | neue DHCP-Adresse, neue temporäre IPv6-Adresse |
| Changed MAC address | eine Adresse antwortet jetzt mit einem anderen, bekannten Gerät | ausgetauschtes Gerät, DHCP-Adresse an ein anderes Gerät vergeben, **ARP-Spoofing** |
| Flip flop | eine Adresse springt zu dem Gerät zurück, das sie davor hatte | doppelt vergebene Adresse, zwei Geräte streiten um eine IP, **ARP-Spoofing** |

Ein neues Gerät auf einer Adresse, die vorher einem anderen gehörte, wird
als **New station** gemeldet; das vorherige Gerät steht in der Spalte
**Previous MAC address**.

Dieselbe Änderung derselben Adresse wird innerhalb von 15 Minuten nur
einmal gemeldet. Streiten zwei Geräte um eine Adresse, entsteht so nicht
bei jedem Durchlauf ein neues Ereignis.

Link-lokale IPv6-Adressen (`fe80::…`) werden mit ihrem Interface geführt
(`fe80::1%igb1`), weil auf jedem Segment dieselbe Adresse vorkommen darf.

## Einstellungen

| Feld | Bedeutung | Vorgabe |
| --- | --- | --- |
| Enable | Monitor einschalten | aus |
| Interfaces | zu beobachtende Interfaces; leer bedeutet alle | leer |
| IPv6 | auch die NDP-Tabelle lesen | an |
| Scan interval | Sekunden zwischen zwei vollständigen Durchläufen (erweiterte Ansicht); Änderungen, die der Kernel meldet, werden dazwischen aufgegriffen | 30 |
| Report | Ereignisarten, die in die Logdatei und in die Mail gehen | New station, Changed MAC address, Flip flop |
| Keep events (days) | Aufbewahrung der Ereignisse; 0 behält sie unbegrenzt | 180 |
| Compatible log line | zusätzlich die Logzeile früherer Versionen in das Systemlog schreiben, siehe [Umstieg](#umstieg-von-früheren-versionen) | aus, nach einem Update an |
| Send mail | Meldung per Mail einschalten | aus |
| Recipients | eine oder mehrere Adressen, durch Kommas getrennt | leer |
| Collect for (minutes) | Sammelzeit für die Mail, siehe unten | 15 |

**New address** ist ab Werk abgewählt: Geräte mit temporären
IPv6-Adressen (Privacy Extensions) bekommen alle paar Stunden eine neue
Adresse und würden laufend Meldungen erzeugen.

Geräte werden nie automatisch gelöscht, nur über den Papierkorb oder
**Reset database**.

## Meldung per E-Mail

Voraussetzung ist ein eingerichtetes `os-smtp-cli-maxit` unter **Services →
SMTP-Send**; zuerst dort die Testmail prüfen.

Das erste Ereignis geht sofort hinaus. Alle weiteren werden für die
eingestellte Sammelzeit gesammelt und in einer Mail verschickt, damit z. B.
ein ganzer Raum voller neuer Geräte nicht zu Dutzenden einzelner Mails
führt. Eine Mail enthält höchstens 200 Ereignisse, der Rest steht auf der
Seite. Der Betreff lautet `OPN-Arp on <Hostname>: <Anzahl> events`.

Ist Mail eingeschaltet, aber `os-smtp-cli-maxit` nicht installiert, steht
beim Start des Dienstes eine Warnung im Log. Fehlgeschlagene Zustellungen
werden ebenfalls dort vermerkt.

## Logdatei und Monit

Der Dienst schreibt unter dem Programmnamen `opnarp` nach syslog, die
Meldungen landen in `/var/log/opnarp/` und sind unter **Services → OPN-Arp
→ Log File** zu sehen. Eine Zeile je Ereignis:

```
new station: ip=192.168.1.20 mac=3c:a6:2f:01:02:03 interface=igb1 vendor="AVM GmbH"
changed MAC address: ip=192.168.1.1 mac=00:11:22:33:44:55 old_mac=00:0d:b9:4a:11:22 interface=igb1
flip flop: ip=192.168.1.1 mac=00:0d:b9:4a:11:22 old_mac=00:11:22:33:44:55 interface=igb1
new address: ip=192.168.1.57 mac=3c:a6:2f:01:02:03 interface=igb1
```

Mit **Compatible log line** kommt für jedes neue Paar aus IP- und
MAC-Adresse zusätzlich die Zeile früherer Versionen in das Systemlog,
unter dem Programmnamen `opn-arp`:

```
New IPv4/MAC pair seen: (192.168.1.20)3c:a6:2f:01:02:03
New IPv6/MAC pair seen fe80::1%igb13c:a6:2f:01:02:03
```

Sie kommt unabhängig von der Auswahl unter **Report**, denn das Bash-Skript
hat jedes neue Paar gemeldet. Ein Gerät, das zu einem früheren Paar
zurückkehrt (Flip flop), war schon einmal gesehen und erzeugt die Zeile
nicht, auch das wie früher. Der Programmname ist bewusst `opn-arp` mit
Bindestrich: Der Filter der eigenen Logdatei erfasst nur `opnarp`, so
bleibt die Zeile im Systemlog, wo bestehende Prüfungen sie erwarten.

Wer statt der eingebauten Mail mit Monit arbeiten will, legt unter
**Services → Monit** einen Dienst vom Typ **File** für
`/var/log/opnarp/latest.log` an und hängt einen Test vom Typ **Content**
an, z. B. mit der Bedingung `content = "changed MAC address|flip flop"`.

Über **System → Einstellungen → Protokollierung** lassen sich die
Meldungen außerdem an einen entfernten Syslog-Server oder ein SIEM
weiterleiten; als Anwendung `opnarp` auswählen.

## Wie das Plugin arbeitet

- Der Dienst `/usr/local/opnsense/scripts/OPNsense/Opnarp/opnarpd.py` wird
  von `daemon(8)` überwacht und nach einem Absturz nach zehn Sekunden neu
  gestartet. Ausgaben und Fehler gehen ebenfalls in die Logdatei.
- Er liest die Tabellen über `arp -an` und `ndp -an`. Eigene Adressen der
  Firewall (`permanent`), Proxy-ARP-Einträge (`published`) und nicht
  aufgelöste Einträge (`incomplete`) werden übergangen.
- Zwischen den vollständigen Durchläufen lauscht er am Routing-Socket des
  Kernels. Meldet der Kernel eine Änderung, folgt ein Durchlauf, frühestens
  fünf Sekunden nach dem letzten. Die meisten Ereignisse erscheinen damit
  binnen Sekunden.
- Das Verzeichnis liegt in SQLite unter `/var/db/opnarp/opnarp.db` und
  übersteht einen Neustart. Der Zeitpunkt **Last seen** eines unveränderten
  Geräts wird höchstens alle fünf Minuten geschrieben, um die Festplatte zu
  schonen.
- Die Seite liest und ändert das Verzeichnis über configd
  (`configctl opnarp list stations`, `… list events`,
  `… station ack|unack|delete <MAC>`, `… resetdb`). Der laufende Dienst
  bemerkt solche Änderungen beim nächsten Durchlauf von selbst.

## Berechtigungen

Zwei ACLs, zuweisbar unter **System → Zugriff → Gruppen**:

| Berechtigung | Muster | Zweck |
| --- | --- | --- |
| Services: OPN-Arp | `ui/opnarp/*`, `api/opnarp/*` | Geräte und Ereignisse ansehen und bestätigen, Einstellungen ändern |
| Services: OPN-Arp: Log File | `ui/diagnostics/log/core/opnarp`, `api/diagnostics/log/core/opnarp/*` | Logdatei ansehen |

## Womit das Plugin verzahnt ist

Diese Schnittstellen der GUI und des Systems werden genutzt. Sie sind für
**26.7** geschrieben; wer das Plugin auf eine neuere Version hebt, sollte
hier hinschauen:

| Schnittstelle | Verwendung |
| --- | --- |
| Textausgabe von FreeBSDs `arp -an` (`? (<IP>) at <MAC> on <Interface> …`) | IPv4-Nachbarn |
| Textausgabe von FreeBSDs `ndp -an` (Spalten Neighbor, Linklayer Address, Netif, Expire) | IPv6-Nachbarn |
| Routing-Socket (`AF_ROUTE`), Nachrichtentyp `RTM_ADD`, `RTM_DELETE`, `RTM_CHANGE` | Auslöser für einen Durchlauf |
| `py-netaddr`, `EUI(…).oui.registration().org` | Herstellernamen |
| `helpers.physical_interface()` im Template | Interface → Gerätename |
| `<plugin>_syslog()` und `templates/OPNsense/Syslog/local/opnarp.conf` | eigene Logdatei |
| `/ui/diagnostics/log/core/opnarp` | Menüpunkt Log File |
| configd-Aktion `smtpcli send` aus `os-smtp-cli-maxit` | Mailversand |
| `logger(1)` mit Tag `opn-arp`, Systemlog als Ziel für Programme ohne eigenen Filter | Compatible log line |
| `ApiControllerBase::searchRecordsetBase`, `UIBootgrid` mit `formatters` und `requestHandler` | Reiter Stations und Events |
| `ApiMutableModelControllerBase`, `ApiMutableServiceControllerBase`, `layout_partials/base_form` | Reiter Settings, Dienststeuerung |

## Bekannte Grenzen

- **Nur Nachbarn der Firewall.** Geräte, die nie mit der Firewall sprechen,
  erscheinen nicht. Hinter einem Router liegende Netze zeigen nur den
  Router.
- **ARP-Einträge verfallen.** Ein Gerät, das sich nur kurz meldet, steht so
  lange in der Tabelle, wie der Kernel den Eintrag hält (ab Werk 20
  Minuten), und wird in dieser Zeit sicher gesehen. Ob der Kernel jede
  einzelne ARP-Änderung auch auf dem Routing-Socket meldet, hängt von der
  FreeBSD-Version ab; im schlechtesten Fall erscheint ein Ereignis erst mit
  dem nächsten vollständigen Durchlauf.
- **DHCP erzeugt Changed MAC address.** Vergibt der DHCP-Server eine frei
  gewordene Adresse an ein anderes Gerät, ist das für den Monitor eine
  geänderte MAC-Adresse. In Netzen mit kurzen Leases und vielen
  wechselnden Geräten (Gäste-WLAN) ist das Ereignis deshalb wenig
  aussagekräftig; dort hilft die Auswahl der Interfaces.
- **Private WLAN-Adressen.** Smartphones und Notebooks wechseln ihre
  MAC-Adresse je Netz und teilweise regelmäßig. Jeder Wechsel ist ein neues
  Gerät.
- **CARP.** In einem Hochverfügbarkeits-Paar sieht nur der Knoten die
  Geräte, über den der Verkehr läuft. Die Einstellungen werden
  synchronisiert, das Verzeichnis nicht.
- **RAM-Dateisystem.** Liegt `/var/db` auf einem RAM-Dateisystem, geht das
  Verzeichnis bei jedem Neustart verloren und der Monitor lernt neu.

## Support / Fehlerbehebung

- **Der Dienst startet nicht.** Unter **Services → OPN-Arp → Log File**
  nachsehen; ein Python-Fehler steht dort mit Traceback. Von Hand prüfen
  mit `service opnarp status` bzw. `configctl opnarp restart`.
- **Es erscheinen keine Geräte.** Prüfen, ob die Tabellen überhaupt
  Einträge haben: `arp -an` und `ndp -an`. Sind Interfaces ausgewählt, muss
  das Gerät dort stehen; der Gerätename in der ausgegebenen Spalte `on …`
  muss zu einem ausgewählten Interface gehören. Die erzeugte Konfiguration
  steht in `/usr/local/etc/opnarp.conf`.
- **Meldungen fehlen in der Logdatei.** Nur die unter **Report**
  ausgewählten Arten werden geschrieben; der Reiter Events zeigt alle.
  Steht nach der Installation gar nichts unter **Log File**, einmal
  syslog neu starten (`configctl syslog restart`), damit die neue
  Logdatei angelegt wird.
- **Die Mail kommt nicht an.** Zuerst die Testmail unter **Services →
  SMTP-Send** prüfen, dann in der Logdatei nach `mail to … failed` suchen.
- **Das Verzeichnis direkt abfragen.** Die Ausgabe ist reines JSON:
  `/usr/local/opnsense/scripts/OPNsense/Opnarp/opnarp_ctl.py stations`
- **Zu viele Meldungen nach einem Umbau.** Nach dem Umzug in ein neues Netz
  oder dem Tausch vieler Geräte **Reset database** verwenden, statt jedes
  Gerät einzeln zu bestätigen.

Die Änderungen jeder Version stehen im Changelog in `pkg-descr`.
