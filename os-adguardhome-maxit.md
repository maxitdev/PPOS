# os-adguardhome-maxit – AdGuard Home für OPNsense

`os-adguardhome-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG,
das [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) als
netzwerkweiten DNS-Filter (Werbung, Tracker, Malware-Domains, Parental
Control) auf der Firewall bereitstellt. Das Plugin bringt das
AdGuard-Home-Binary mit, verankert es als OPNsense-Dienst (Start/Stop/Status,
Autostart, configd-Aktionen) und stellt eine kleine Konfigurationsseite in der
WebGUI bereit.

> **Wichtig:** Die eigentliche Konfiguration von AdGuard Home (Filterlisten,
> Upstream-Resolver, Clients, Benutzerkonten, Query-Log) findet **nicht** in
> der OPNsense-WebGUI statt, sondern in der eigenen Weboberfläche von AdGuard
> Home. Die OPNsense-Seite kennt nur zwei Schalter. Entsprechend liegt diese
> Konfiguration auch nicht in der `config.xml`, sondern in
> `/usr/local/AdGuardHome/AdGuardHome.yaml` – siehe
> [Backup und Deinstallation](#backup-und-deinstallation).

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Einrichtung](#einrichtung)
4. [Felder der Konfigurationsseite](#felder-der-konfigurationsseite)
5. [Betrieb neben Unbound und Dnsmasq](#betrieb-neben-unbound-und-dnsmasq)
6. [Dienststeuerung und Cron](#dienststeuerung-und-cron)
7. [Dateien und Pfade](#dateien-und-pfade)
8. [Updates](#updates)
9. [Backup und Deinstallation](#backup-und-deinstallation)
10. [Berechtigungen](#berechtigungen)
11. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über das
  `os-adguardhome-maxit` bezogen wird. Das AdGuard-Home-Binary ist im Paket
  enthalten, es werden keine weiteren FreeBSD-Pakete benötigt.
- Ein freier TCP-Port für die AdGuard-Home-Weboberfläche. Die Ports 80 und 443
  sind in der Regel von der OPNsense-WebGUI belegt – für AdGuard Home ist
  daher z. B. `3000` zu verwenden.
- Soll AdGuard Home DNS-Anfragen des Netzes direkt annehmen (UDP/TCP 53), muss
  Port 53 frei sein, d. h. **Unbound** bzw. **Dnsmasq** müssen deaktiviert oder
  auf einen anderen Port gelegt werden (siehe
  [Betrieb neben Unbound und Dnsmasq](#betrieb-neben-unbound-und-dnsmasq)).

## Installation

1. Unter **System → Firmware → Plugins** nach `os-adguardhome-maxit` suchen und
   installieren.
2. In der linken Navigation erscheint unter **Services** der neue Menüpunkt
   **Adguardhome → General**.

## Einrichtung

1. Zu **Services → Adguardhome → General** wechseln, **Enable** aktivieren und
   **Save** klicken. Damit wird der Dienst eingerichtet (Autostart) und
   gestartet; der Dienststatus ist an der Ampel rechts oben im Seitenkopf
   erkennbar.
2. Den Einrichtungsassistenten von AdGuard Home im Browser öffnen:
   `http://<IP-der-Firewall>:3000`. Beim ersten Start hört AdGuard Home
   ausschließlich auf diesem Port.
3. Im Assistenten:
   - **Admin Web Interface:** Der vorgeschlagene Port ist `80` – dieser ist von
     der OPNsense-WebGUI belegt und muss geändert werden, z. B. auf `3000`.
     Andernfalls startet AdGuard Home nach Abschluss des Assistenten nicht mehr.
   - **DNS server:** Port `53`, wenn AdGuard Home der primäre Resolver des
     Netzes werden soll (dann muss Port 53 vorher freigeräumt sein), sonst ein
     alternativer Port wie `5353`.
   - Administrator-Benutzer und Passwort festlegen.
4. Nach Abschluss des Assistenten in AdGuard Home die Upstream-Resolver und die
   gewünschten Filterlisten konfigurieren (**Settings → DNS settings** bzw.
   **Filters**).
5. Zurück in OPNsense: Läuft AdGuard Home auf Port 53, in **Services →
   Adguardhome → General** zusätzlich **Primary DNS** aktivieren und **Save**
   klicken.
6. Die Clients auf die Firewall als DNS-Server zeigen lassen (z. B. über die
   DHCP-Einstellungen der jeweiligen Schnittstelle).

> **Hinweis zu Firewall-Regeln:** Für den Zugriff aus dem LAN gilt die normale
> Regelbasis (die Standard-LAN-Regel erlaubt den Zugriff bereits). Der
> Admin-Port von AdGuard Home darf keinesfalls von der WAN-Seite erreichbar
> sein.

## Felder der Konfigurationsseite

| Feld | Beschreibung |
|---|---|
| Enable | Aktiviert bzw. deaktiviert den AdGuard-Home-Dienst (setzt `adguardhome_enable` in `/etc/rc.conf.d/adguardhome` und startet/stoppt den Dienst beim Speichern). Nur bei aktivierter Option erscheint AdGuard Home in der Dienstübersicht und startet nach einem Reboot automatisch. |
| Primary DNS | Nur zu aktivieren, wenn AdGuard Home selbst auf Port 53 lauscht. Die Option meldet OPNsense, dass der DNS-Port 53 durch AdGuard Home belegt ist, damit die Kernkomponenten diesen Umstand bei der Konfliktprüfung mit den eigenen Resolvern berücksichtigen. Der Lauschport von AdGuard Home wird dadurch **nicht** verändert – dieser wird ausschließlich in der Weboberfläche von AdGuard Home gesetzt. |

Beide Optionen werden beim Speichern sofort angewendet; einen separaten
"Apply"-Schritt gibt es nicht.

## Betrieb neben Unbound und Dnsmasq

Port 53 kann nur von einem Dienst belegt werden. Es haben sich zwei Varianten
etabliert:

**Variante A – AdGuard Home als primärer Resolver (empfohlen)**

1. **Services → Unbound DNS → General** öffnen und Unbound entweder
   deaktivieren oder auf einen anderen Port legen (z. B. `5353`); dasselbe
   gegebenenfalls für Dnsmasq.
2. In AdGuard Home den DNS-Listener auf Port 53 setzen.
3. Als Upstream in AdGuard Home entweder öffentliche Resolver (DoT/DoH) oder
   den auf 5353 verschobenen Unbound (`127.0.0.1:5353`) eintragen – letzteres,
   wenn die lokale Namensauflösung von Unbound (Host-Overrides, DHCP-Registry)
   weiter genutzt werden soll.
4. In OPNsense **Primary DNS** aktivieren.

**Variante B – AdGuard Home als Zwischenstufe**

AdGuard Home lauscht auf einem alternativen Port (z. B. `5353`), Unbound bleibt
auf Port 53 und trägt AdGuard Home als Query-Forwarding-Ziel ein. In diesem
Fall bleibt **Primary DNS** deaktiviert. Diese Variante ist etwas
umständlicher, erhält aber alle Unbound-Funktionen unverändert.

Damit Clients den Filter nicht umgehen, empfiehlt sich in beiden Varianten eine
NAT-Regel, die ausgehende DNS-Anfragen (Port 53) auf die Firewall umleitet, und
das Blockieren von DoH/DoT-Resolvern – letzteres deckt das Plugin
`os-appblocking-maxit` mit der Kategorie *DoH/DoT* ab.

## Dienststeuerung und Cron

- Start, Stop und Neustart sind über das Dienst-Widget im Seitenkopf sowie über
  **System → Diagnose → Dienste** möglich.
- Auf der Kommandozeile stehen die configd-Aktionen
  `configctl adguardhome start|stop|restart|status` bereit.
- Die Aktion **"Restart Adguardhome service"** lässt sich unter
  **System → Einstellungen → Cron** als geplanter Job hinterlegen, z. B. für
  einen nächtlichen Neustart.

## Dateien und Pfade

| Pfad | Inhalt |
|---|---|
| `/usr/local/AdGuardHome/AdGuardHome` | Das mitgelieferte AdGuard-Home-Binary. |
| `/usr/local/AdGuardHome/AdGuardHome.yaml` | Die komplette AdGuard-Home-Konfiguration (Filter, Clients, Benutzer, Upstreams). Wird beim ersten Start bzw. vom Einrichtungsassistenten angelegt. |
| `/usr/local/AdGuardHome/data/` | Laufzeitdaten: Statistiken, Query-Log, heruntergeladene Filterlisten. |
| `/usr/local/etc/rc.d/adguardhome` | rc-Skript (startet das Binary über `daemon(8)`). |
| `/etc/rc.conf.d/adguardhome` | Aus der OPNsense-Konfiguration generiert, enthält nur `adguardhome_enable`. |
| `/var/run/adguardhome.pid` | PID-Datei, über die OPNsense den Dienststatus ermittelt. |

## Updates

Das AdGuard-Home-Binary ist Bestandteil des Plugins und wird ausschließlich
über Plugin-Updates aktualisiert (**System → Firmware → Aktualisierungen**).
Die zum jeweiligen Plugin-Stand gehörende AdGuard-Home-Version ist im
Änderungsverlauf in der Datei `pkg-descr` dieses Verzeichnisses vermerkt.

Die in der AdGuard-Home-Oberfläche angebotene Selbstaktualisierung sollte
**nicht** verwendet werden: Ein so ersetztes Binary wird beim nächsten
Plugin-Update wieder überschrieben, und der Selbst-Update-Mechanismus ist auf
diese Paketierung nicht abgestimmt.

## Backup und Deinstallation

- Ein OPNsense-Konfigurationsbackup (`config.xml`) enthält von diesem Plugin
  nur die beiden Schalter **Enable** und **Primary DNS** – nicht die
  AdGuard-Home-Konfiguration selbst. Wer Filterlisten, Clients und Benutzer
  sichern will, muss `/usr/local/AdGuardHome/AdGuardHome.yaml` separat sichern
  (z. B. per SCP).
- Bei der Deinstallation des Plugins wird das Verzeichnis
  `/usr/local/AdGuardHome/` **vollständig entfernt**. Damit gehen Konfiguration,
  Statistiken und Query-Log verloren. Vor einer Deinstallation – auch vor einer
  "kurzen" Neuinstallation zur Fehlersuche – daher unbedingt vorher die
  `AdGuardHome.yaml` sichern.

## Berechtigungen

Alle Seiten und die zugehörige API liegen unter den Mustern `ui/adguardhome/*`
bzw. `api/adguardhome/*`. Damit weitere Benutzer/Benutzergruppen (neben `root`)
darauf zugreifen dürfen, muss ihnen unter **System → Zugriff → Gruppen** die
Berechtigung **"Services: Adguardhome"** zugewiesen werden. Die Anmeldung an
der AdGuard-Home-Weboberfläche selbst ist davon unabhängig und wird in AdGuard
Home verwaltet.

## Support / Fehlerbehebung

- **Der Dienst startet nicht bzw. fällt sofort wieder aus:** Meist ist ein Port
  belegt – entweder Port 53 (Unbound/Dnsmasq läuft noch) oder der Admin-Port
  (80/443 der WebGUI). Zur Diagnose lässt sich AdGuard Home im Vordergrund
  starten:

  ```
  /usr/local/AdGuardHome/AdGuardHome -s run
  ```

  Fehlermeldungen erscheinen dann direkt auf der Konsole; Abbruch mit `Ctrl-C`.
- **Keine Logausgaben in den OPNsense-Protokollen:** Das rc-Skript startet
  AdGuard Home über `daemon(8)` mit `-f`, wodurch Stdout/Stderr verworfen
  werden. Diagnoseinformationen liefern das Dashboard und das Query-Log von
  AdGuard Home selbst; alternativ lässt sich in der `AdGuardHome.yaml` über den
  Schlüssel `log_file` eine Logdatei setzen.
- **Admin-Oberfläche nicht erreichbar, Port vergessen:** Der konfigurierte Port
  steht in `/usr/local/AdGuardHome/AdGuardHome.yaml` (Abschnitt `http`).
- **Clients umgehen den Filter:** Prüfen, ob die Clients per DHCP tatsächlich
  die Firewall als DNS-Server erhalten, und ob DNS-Umleitung bzw. eine
  DoH/DoT-Blockade aktiv ist.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr` dieses
  Verzeichnisses dokumentiert.
