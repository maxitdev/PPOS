# os-cm-ansible – Central Management für OPNsense

`os-cm-ansible` ist ein Plugin der m.a.x. Informationstechnologie AG, mit dem sich
mehrere OPNsense-Firewalls zentral verwalten lassen: Firmenweite Firewall-Regeln,
Aliase, Captive-Portal-Einstellungen, Updates/Upgrades und Backups werden an
einer Stelle gepflegt und per Ansible/API auf die einzelnen Firewalls verteilt.

> **Wichtiger Hinweis:** `os-cm-ansible` ist **kein Plugin für produktive
> Firewalls**. Es ist ausdrücklich für den Einsatz auf einer dedizierten
> OPNsense-Instanz gedacht, die ausschließlich als "Verwaltungs-Firewall"
> (Central Management) dient. Auf den zu verwaltenden Produktiv-Firewalls wird
> dieses Plugin **nicht** installiert.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Netzwerk-Architektur](#netzwerk-architektur)
3. [Schritt 1: os-maxit Repository einrichten](#schritt-1-os-maxit-repository-einrichten)
4. [Schritt 2: os-cm-ansible installieren](#schritt-2-os-cm-ansible-installieren)
5. [Schritt 3: Zu verwaltende Firewalls vorbereiten](#schritt-3-zu-verwaltende-firewalls-vorbereiten)
6. [Schritt 4: Firewall im Inventar anlegen](#schritt-4-firewall-im-inventar-anlegen)
7. [Dashboard](#dashboard)
8. [Inventar (Inventory)](#inventar-inventory)
9. [Gruppen (Groups)](#gruppen-groups)
10. [Firewall-Regeln (Firewall)](#firewall-regeln-firewall)
11. [Aliase (Alias)](#aliase-alias)
12. [Zertifikate & CA-Verteilung](#zertifikate--ca-verteilung)
13. [Benutzerverwaltung (Users)](#benutzerverwaltung-users)
14. [Backups](#backups)
15. [Erweitert: AliasSet, RuleSet, Captive Portal](#erweitert-aliasset-ruleset-captive-portal)
16. [CM Tools: Config Builder [BETA]](#cm-tools-config-builder-beta)
17. [Berechtigungen](#berechtigungen)
18. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine gültige Subscription/Zugangsdaten (Benutzername/Passwort) von m.a.x. it
  für das Firmware-Repository `os-maxit`.
- Eine dedizierte OPNsense-Instanz (z. B. eine kleine VM), die als zentrale
  Management-Firewall dient.
- Netzwerkverbindung von dieser Management-Instanz zu allen zu verwaltenden
  Firewalls (siehe [Netzwerk-Architektur](#netzwerk-architektur)).
- Auf jeder zu verwaltenden Firewall: Zugriff auf die WebGUI/API sowie ein API-Key
  des `root`-Benutzers (siehe [Schritt 3](#schritt-3-zu-verwaltende-firewalls-vorbereiten)).

## Netzwerk-Architektur

`os-cm-ansible` steuert alle Firewalls über deren WebGUI-REST-API (HTTPS) und
optional per SSH (z. B. für den Ansible-Backup-/Config-Builder-Teil). Die
Management-Firewall benötigt daher eine Route/Erreichbarkeit zu jeder verwalteten
Firewall.

Empfohlener Aufbau:

- Die Management-Instanz wird als eigene, kleine OPNsense-VM in einem
  abgeschotteten **Management-Netz** betrieben (z. B. in eurer Virtualisierungs-
  umgebung neben oder hinter den zu verwaltenden Firewalls).
- Von diesem Management-Netz aus muss mindestens der WebGUI-Port (Standard
  `443/tcp`, siehe Inventar) jeder verwalteten Firewall erreichbar sein.
- Zugriff auf die Management-Instanz selbst sollte auf Administratoren
  beschränkt sein (eigene Firewall-Regeln, ggf. VPN-Zugang).
- Die zu verwaltenden Firewalls benötigen im Gegenzug **keine** ausgehende
  Verbindung zur Management-Instanz – die Kommunikation läuft immer von der
  Management-Firewall aus (Ansible/API-Client) zu den Ziel-Firewalls.

```
                +-----------------------------+
                |   Management-Netz (VM)      |
                |  OPNsense + os-cm-ansible    |
                +--------------+---------------+
                               | HTTPS (API) / optional SSH
             ------------------------------------------------
             |                |                |            |
      +------v-----+   +------v-----+   +------v-----+  +----v-------+
      | Firewall A |   | Firewall B |   | Firewall C |  | Firewall … |
      +------------+   +------------+   +------------+  +------------+
```

## Schritt 1: os-maxit Repository einrichten

Damit die OPNsense-Firmware das Plugin `os-cm-ansible` überhaupt anbieten kann,
muss zunächst das m.a.x. it Firmware-Repository (`os-maxit`) auf der
Management-Instanz eingerichtet werden.

1. Zugangsdaten (Benutzername/Passwort) von m.a.x. it anfragen bzw. aus dem
   Kunden-/Support-Bereich entnehmen.
2. Eine ausführliche Video-Anleitung zur Einrichtung des Repositories stellt
   m.a.x. it hier zur Verfügung:
   [opnsense.max-it.de/professional-content/freie-videos](https://opnsense.max-it.de/professional-content/freie-videos/)
3. In der OPNsense-WebGUI der Management-Instanz unter **System → Firmware →
   Einstellungen** das `os-maxit`-Repository mit den erhaltenen Zugangsdaten
   (Benutzername/Passwort) hinterlegen.
4. Anschließend unter **System → Firmware → Aktualisierungen** auf
   **Nach Updates suchen** klicken, damit OPNsense die Paketlisten des neuen
   Repositories abruft und die m.a.x. it Plugins (u. a. `os-cm-ansible`) in der
   Pluginliste erscheinen.

> Solltet ihr bereits andere m.a.x. it Plugins (z. B. `os-apache-maxit`)
> installiert haben, ist das Repository in der Regel bereits eingerichtet und
> dieser Schritt kann übersprungen werden.

## Schritt 2: os-cm-ansible installieren

1. Unter **System → Firmware → Plugins** nach `os-cm-ansible` suchen.
2. Plugin installieren.
3. Nach der Installation erscheinen in der linken Navigation zwei neue Menüs:
   - **Central Management** – Dashboard, Inventory, Groups, Firewall, Alias,
     Backups sowie unter "Advanced" AliasSet, RuleSet und Captive Portal.
   - **CM Tools [BETA]** – Config Builder.

## Schritt 3: Zu verwaltende Firewalls vorbereiten

Damit die Management-Instanz eine Firewall steuern kann, muss diese Firewall
zwei Dinge bereitstellen: Erreichbarkeit der WebGUI und einen API-Schlüssel.

1. **Firewall-Regel für WebGUI-Zugriff:** Auf der zu verwaltenden Firewall unter
   **Firewall → Regeln** eine Regel anlegen (bzw. prüfen, ob bereits vorhanden),
   die Zugriff vom Management-Netz/der Management-IP auf den WebGUI-Port
   (Standard `443/tcp`, ggf. abweichend konfiguriert) erlaubt. Ohne diese Regel
   kann die Management-Instanz die API der Firewall nicht erreichen.
2. **API-Key für den root-Benutzer anlegen:** Unter **System → Zugriff →
   Benutzer** den Benutzer `root` öffnen und im Abschnitt "API-Schlüssel" über
   das `+` einen neuen Schlüssel generieren. OPNsense erzeugt daraufhin einen
   **Key** und ein **Secret** und bietet die Zugangsdaten als Download-Datei an.
   Das Secret wird nur einmal angezeigt – Datei sicher aufbewahren.
3. Diese Vorbereitung muss für **jede** Firewall wiederholt werden, die über
   das Central Management verwaltet werden soll.

## Schritt 4: Firewall im Inventar anlegen

Mit Key/Secret aus Schritt 3 wird die Firewall nun im Central Management
eingetragen (siehe [Inventar](#inventar-inventory) für alle Felder):

1. Auf der Management-Instanz zu **Central Management → Inventory** wechseln.
2. Über `+` einen neuen Eintrag anlegen.
3. **Firewall Name** vergeben, **API Key** und **API Secret** aus Schritt 3
   eintragen, **Firewall IP** (Host) und ggf. abweichenden **Port** angeben.
4. Optional Tags vergeben, SSH-Port/-Key hinterlegen (falls Ansible-Funktionen
   wie Backup oder Config Builder per SSH benötigt werden) und Backup
   aktivieren.
5. Speichern und anschließend oben auf **Save** klicken, damit die
   Konfiguration übernommen wird (`reconfigure`).
6. Die Firewall erscheint nun im [Dashboard](#dashboard) und kann dort
   angesprochen bzw. zu einer [Gruppe](#gruppen-groups) hinzugefügt werden.

---

## Dashboard

Das Dashboard (**Central Management → Dashboard**) zeigt alle im Inventar
hinterlegten Firewalls in einer Übersichtstabelle:

- **Name / Tags** – Bezeichnung und Tags aus dem Inventar.
- **WebAdmin** – Button, der die WebGUI der jeweiligen Firewall in einem neuen
  Tab öffnet.
- **Version** – aktuell installierte OPNsense-Version, wird per Live-Abfrage
  (alle 30 Sekunden) über die API der Firewall ermittelt.
- **Status** – farbiger Indikator: grün = Firewall erreichbar/API-Key gültig,
  rot = Firewall nicht erreichbar (Timeout) oder API-Key falsch/fehlt.
- **Update / Upgrade** – Buttons, um auf der jeweiligen Firewall einzeln ein
  Update (Patch-Level) oder ein Upgrade (Major-Version) anzustoßen. Während der
  Ausführung wird der Status laufend abgefragt.

Zusätzliche Funktionen:

- **Gruppenfilter** (oberhalb der Tabelle) – blendet die Liste auf eine
  bestimmte [Gruppe](#gruppen-groups) ein.
- **Mehrfachauswahl + "WebAdmin"** – öffnet die WebGUI aller ausgewählten
  Firewalls gleichzeitig in neuen Tabs.
- **Mehrfachauswahl + "Update"** – stößt für alle ausgewählten Firewalls ein
  Update an (bricht ab, falls eine der ausgewählten Firewalls gerade nicht
  erreichbar ist).
- **Tab "Support" → Clear Cache** – leert zwischengespeicherte Cache-Dateien des
  Plugins, falls Anzeigen (z. B. Versionsstände) nicht mehr aktuell wirken.

## Inventar (Inventory)

Unter **Central Management → Inventory** wird die Liste aller verwalteten
Firewalls gepflegt. Felder pro Eintrag:

| Feld | Beschreibung |
|---|---|
| Enabled | Firewall aktiv/inaktiv schalten, ohne den Eintrag zu löschen. |
| Firewall Name | Anzeigename, muss eindeutig sein. |
| API Key / API Secret | Zugangsdaten aus [Schritt 3](#schritt-3-zu-verwaltende-firewalls-vorbereiten). |
| Firewall IP | IP-Adresse/Host, auf der die WebGUI der Firewall lauscht. |
| Port | WebGUI-Port, falls abweichend von 443. |
| Tags | Frei definierbare Schlagworte, u. a. für Filterung im Dashboard. |
| SSH Port / SSH Key | Optional, für Ansible-Funktionen, die SSH statt/zusätzlich zur API nutzen. |
| Enable Backup | Nimmt die Firewall in die zentrale Backup-Routine auf (siehe [Backups](#backups)). |
| Delete Backup after (days) | Aufbewahrungsdauer der Backups dieser Firewall; `0` = unbegrenzt aufbewahren. |

Nach Änderungen im Grid immer auf **Save** klicken, damit die Konfiguration neu
geschrieben wird.

## Gruppen (Groups)

Gruppen (**Central Management → Groups**) sind der zentrale Baustein, um
Regeln, Aliase und Captive-Portal-Konfigurationen gebündelt auf mehrere
Firewalls auszurollen.

- Eine Gruppe ist eine **Host-Gruppe**: Eine Firewall (Mitglied) kann jeweils
  nur **einer** Gruppe angehören, aber jede Gruppe kann mehrere Firewall-Regeln
  enthalten. Gemeinsame Regeln für alle Firewalls sollten daher in jede
  bestehende Gruppe übernommen werden, statt eine zusätzliche "Alle"-Gruppe
  anzulegen.
- Voraussetzung für die zentrale Regel-/Alias-Verteilung: Alle Mitglieds-
  Firewalls benötigen mindestens OPNsense 24.1 oder das Plugin `os-firewall`.

Felder je Gruppe:

- **Members** – die Firewalls (aus dem Inventar) dieser Gruppe.
- **Rules** bzw. **RuleSet** – einzelne Firewall-Regeln oder ein zusammen-
  gefasstes [RuleSet](#erweitert-aliasset-ruleset-captive-portal), die auf die
  Mitglieder verteilt werden (nur eines von beidem gleichzeitig wählbar).
- **Alias** bzw. **AliasSet** – einzelne Aliase oder ein
  [AliasSet](#erweitert-aliasset-ruleset-captive-portal) für die Verteilung.
- **Captive Portal** – optionale Zuordnung eines Captive-Portal-Eintrags.
- **Certificate Authorities** / **Certificates** – CAs bzw. Zertifikate (von
  der Management-Instanz), die auf die Mitglieder verteilt werden sollen,
  siehe [Zertifikate & CA-Verteilung](#zertifikate--ca-verteilung).
- **Don't send CA keys** – wenn aktiviert, werden von den zugeordneten CAs nur
  die öffentlichen Zertifikate verteilt, keine privaten Schlüssel. Private
  Schlüssel von Zertifikaten werden davon unabhängig immer mit verteilt.
- **Group Prefix** (Advanced) – Präfix, mit dem zentral verteilte Aliase/Regeln
  auf der Ziel-Firewall gekennzeichnet werden. Dadurch bleiben lokal auf der
  Firewall angelegte Aliase/Regeln unangetastet und werden bei einer erneuten
  Verteilung nicht überschrieben oder gelöscht.

Buttons unterhalb des Grids stoßen die Verteilung für alle ausgewählten
Gruppen (Mehrfachauswahl per Checkbox) nacheinander an – eine Gruppe wird
erst bearbeitet, sobald die vorherige abgeschlossen ist:

- **Alias** – deployt die zugeordneten Aliase.
- **Rules** – deployt die zugeordneten Firewall-Regeln (vorher sollten
  geänderte Aliase deployt werden, damit die Regeln auf aktuelle Aliase
  verweisen).
- **Captive Portal** – deployt die Captive-Portal-Konfiguration.
- **Deploy Certs/CAs** – deployt die zugeordneten Zertifikate/CAs, siehe
  [Zertifikate & CA-Verteilung](#zertifikate--ca-verteilung).

### Befehle in der Grid-Spalte "Commands"

Jede Zeile des Gruppen-Grids bietet neben den Standardaktionen **Bearbeiten**
(Stift) und **Löschen** (Papierkorb) folgende Befehle, die direkt auf die
Mitglieder der jeweiligen Gruppe wirken:

| Befehl | Beschreibung |
|---|---|
| Deploy rules | Deployt die zugeordneten Firewall-Regeln (Rules/RuleSet) auf alle Mitglieder dieser Gruppe. Vor der Ausführung erscheint der Hinweis, geänderte Aliase vorher zu deployen. |
| Deploy aliases | Deployt die zugeordneten Aliase (Alias/AliasSet) auf alle Mitglieder dieser Gruppe. |
| Deploy plugins | Installiert ein frei wählbares OPNsense-Plugin auf allen Mitgliedern dieser Gruppe. Nach Bestätigung wird die komplette, der Management-Instanz bekannte Firmware-/Plugin-Liste in einer durchsuchbaren Auswahlliste angezeigt; nach Auswahl eines Plugins wird dieses per Ansible auf jeder Firewall der Gruppe installiert. So lässt sich z. B. ein neues Plugin auf einer ganzen Gruppe von Firewalls ausrollen, ohne jede Firewall einzeln aufzurufen. |
| Deploy CP | Deployt die zugeordnete Captive-Portal-Konfiguration auf alle Mitglieder dieser Gruppe. |
| Flush aliases | Entfernt zuvor zentral verteilte, aber der Gruppe nicht mehr zugeordnete bzw. nicht mehr verwendete Aliase wieder von den Mitgliedern dieser Gruppe. Nur ungenutzte Aliase werden entfernt. |
| Deploy Certs/CAs | Deployt die zugeordneten Zertifikate/CAs auf alle Mitglieder dieser Gruppe, siehe [Zertifikate & CA-Verteilung](#zertifikate--ca-verteilung). |

Jeder dieser Befehle steht zusätzlich als Sammelaktion zur Verfügung: Werden
über die Checkboxen mehrere Gruppen markiert, erscheint eine
"(bulk)"-Variante des Befehls, die alle ausgewählten Gruppen in **einem**
Aufruf bedient (im Unterschied zu den Buttons unterhalb des Grids, die
mehrere Gruppen nacheinander abarbeiten). Vor der Ausführung wird jeweils
eine Bestätigung mit betroffener(n) Gruppe(n) und Befehlsname eingeblendet;
nach Bestätigung wechselt die Ansicht automatisch in den Tab "Status" und
zeigt den Ausführungsfortschritt live an.

### Tab "Updates" – Zeitpläne

Im Tab **Updates** lassen sich wiederkehrende oder einmalige Zeitpläne für
**Update** oder **Upgrade** je Gruppe anlegen:

- Wiederkehrend (z. B. jeden "ersten Montag" um eine feste Uhrzeit) oder
- einmalig zu einem festen Datum (Tag/Monat/Jahr).

Die nächste geplante Ausführung wird je Zeitplan angezeigt.

### Tab "Status" – Ausführungsprotokoll

Der Tab **Status** zeigt die Live-Log-Ausgabe der zuletzt über die
[Grid-Befehle](#befehle-in-der-grid-spalte-commands) oder die Buttons
unterhalb des Grids angestoßenen Aktion an. Die Ansicht öffnet sich beim
Auslösen eines Befehls automatisch; der Inhalt lässt sich per Klick auf
**Click here to copy to clipboard** in die Zwischenablage kopieren.

## Firewall-Regeln (Firewall)

Unter **Central Management → Firewall** werden die zentral verwaltbaren
Firewall-Regeln gepflegt (unabhängig von einer Gruppe – die Zuordnung zu
Firewalls erfolgt erst über die [Gruppen](#gruppen-groups), Feld **Rules**
bzw. **RuleSet**). Felder pro Regel:

| Feld | Beschreibung |
|---|---|
| Enabled | Regel aktiv/inaktiv schalten, ohne den Eintrag zu löschen. |
| Sequence | Numerischer Sortierwert (1–99999); bestimmt die Reihenfolge der Regel innerhalb der auf der Ziel-Firewall erzeugten Regelliste. |
| Action | Pass, Block oder Reject. |
| Quick | Entspricht pf's "Quick"-Verhalten: bei aktivierter Option (Standard) beendet ein Treffer die Regelauswertung sofort; ist die Option deaktiviert, wird wie bei pf üblich die zuletzt zutreffende Regel angewendet. |
| Direction | In oder Out. |
| TCP/IP Version | IPv4 oder IPv6. |
| Protocol | Beliebig, TCP/UDP oder ein einzelnes Protokoll (TCP, UDP, ICMP, …). |
| Source (Alias) / Source | Quelle entweder als bestehender Alias (Host-/Netzwerk-Typ) **oder** als frei eingegebenes IP/Subnetz/Interface – nur eines von beidem gleichzeitig wählbar. |
| Source / Invert | Kehrt den Sinn des Quell-Matches um ("not"). |
| Source port (Alias) / Source port | Nur bei TCP/UDP relevant. Quell-Port als bestehender Alias (Port-Typ) **oder** als Freitext/Portbereich. |
| Destination (Alias) / Destination | Wie Source, jedoch für das Ziel. |
| Destination / Invert | Kehrt den Sinn des Ziel-Matches um. |
| Destination port (Alias) / Destination port | Wie Source port, jedoch für das Ziel. Freitext akzeptiert auch bekannte Portnamen (http, https, imap, imaps, …) und Bereiche mit Bindestrich. |
| Log | Aktiviert die Protokollierung der Regel auf der Ziel-Firewall. |
| Description | Freitextbeschreibung; dient gleichzeitig als eindeutiger Bezeichner der Regel (muss eindeutig sein). |

## Aliase (Alias)

Unter **Central Management → Alias** werden zentrale Aliase gepflegt, die in
Regeln, Captive-Portal-Konfigurationen und AliasSets verwendet werden können.
Felder pro Alias:

| Feld | Beschreibung |
|---|---|
| Enabled | Alias aktiv/inaktiv schalten, ohne den Eintrag zu löschen. |
| Name | Eindeutiger Name, bis zu 32 Zeichen abzüglich der Länge eines eventuellen Group-Prefix. Kann nach dem Anlegen nicht mehr geändert werden (das Namensfeld wird beim Bearbeiten gesperrt). |
| Type | Host, Network, Port oder MAC. |
| Nested Alias | Aktivieren, wenn der Content dieses Alias aus den Namen anderer Aliase besteht (verschachtelter Alias) statt aus direkten Werten. |
| Category | Optionale Gruppierung/Kategorisierung (frei wählbar, max. 32 Zeichen). |
| Content | Liste der eigentlichen Werte (IP-Adressen, Netze, Ports bzw. MAC-Adressen, oder Namen anderer Aliase bei aktiviertem "Nested Alias"). |
| Description | Freitextbeschreibung (max. 255 Zeichen). |

## Zertifikate & CA-Verteilung

Central Management kann Zertifikate und Zertifizierungsstellen (CAs), die auf
der Management-Instanz selbst hinterlegt sind, gebündelt auf die Mitglieder
einer [Gruppe](#gruppen-groups) verteilen:

1. Auf der Management-Instanz unter **System → Trust → Authorities** bzw.
   **System → Trust → Certificates** die gewünschte(n) CA(s)/Zertifikat(e)
   anlegen oder importieren.
2. In der jeweiligen Gruppe unter **Certificate Authorities** bzw.
   **Certificates** die zu verteilenden Einträge auswählen und optional
   **Don't send CA keys** aktivieren.
3. Über den Button **Deploy Certs/CAs** unterhalb des Gruppen-Grids oder über
   das gleichnamige Kommando in der [Grid-Spalte "Commands"](#befehle-in-der-grid-spalte-commands)
   die Verteilung anstoßen.

Existiert auf einer Ziel-Firewall bereits eine CA/ein Zertifikat mit
identischer **Description**, wird der bestehende Eintrag ersetzt (die dortige
Referenz-ID bleibt erhalten), statt einen zweiten Eintrag anzulegen. Ein
erneutes Deployment ersetzt damit automatisch ein zwischenzeitlich erneuertes
bzw. bald ablaufendes Zertifikat oder eine CA mit derselben Description – ist
die Description noch nicht vorhanden, wird stattdessen ein neuer Eintrag
angelegt.

## Benutzerverwaltung (Users)

Im Tab **Users** der [Gruppen-Seite](#gruppen-groups) lassen sich lokale
System-Benutzer einer einzelnen Firewall oder aller Mitglieder einer Gruppe
zentral einsehen und verwalten:

1. Im Dropdown **Firewall / Group** eine Firewall oder Gruppe auswählen. Die
   Tabelle zeigt anschließend alle lokalen Benutzer der ausgewählten
   Firewall(s), inklusive der Angabe, auf welcher/welchen Firewall(s) ein
   Benutzer existiert und ob er dort Admin-Rechte besitzt. Ergebnisse werden
   30 Minuten gecacht; über **Refresh** lässt sich eine sofortige
   Neuabfrage erzwingen.
2. Über `+` lässt sich ein neuer Benutzer anlegen, der auf allen Firewalls im
   gewählten Scope (einzelne Firewall oder alle Gruppenmitglieder) angelegt
   wird.
3. Über den Stift bei einem bestehenden Benutzer lässt sich dieser bearbeiten;
   die Änderung wird auf allen Firewalls im gewählten Scope übernommen – auf
   Firewalls, auf denen der Benutzer dort noch nicht existiert, wird er dabei
   neu angelegt. Damit lässt sich z. B. ein bislang nur auf einer Firewall
   vorhandener Benutzer auf die restliche Gruppe ausrollen.
4. Über den Papierkorb lässt sich ein Benutzer von allen Firewalls im
   gewählten Scope löschen.

Felder im Dialog "Add/Modify User":

- **Username** – bei bestehenden Benutzern nur änderbar, sofern es sich nicht
  um `root` handelt; der Benutzername `root` kann nie geändert werden.
- **Full name**
- **Password** – beim Bearbeiten leer lassen, um das bestehende Passwort
  unverändert zu lassen. Wird beim Neuanlegen kein Passwort angegeben, vergibt
  die Ziel-Firewall ein zufälliges Passwort (anschließend ist ein
  Passwort-Reset auf der Firewall nötig).
- **Deploy with Admin privileges** – nimmt den Benutzer auf der Ziel-Firewall
  in die Gruppe `admins` auf bzw. entfernt ihn daraus; andere
  Gruppenmitgliedschaften des Benutzers bleiben dabei unverändert.
  Voraussetzung: Auf der Ziel-Firewall muss eine Gruppe mit dem Namen `admins`
  existieren (bei einer Standard-OPNsense-Installation bereits vorhanden).

> Der `root`-Benutzer kann weder umbenannt noch gelöscht werden.

Rückmeldungen der Verteilung (z. B. nicht erreichbare Firewalls oder von der
API zurückgemeldete Fehler) werden nach Abschluss je Firewall in einem Dialog
aufgelistet.

## Backups

Unter **Central Management → Backups** lässt sich für jede im Inventar mit
**Enable Backup** markierte Firewall die zentrale Konfigurationssicherung
verfolgen:

- Die Tabelle zeigt je Firewall den Zeitpunkt des letzten Backups.
- Über den Download-Button lässt sich immer nur das **aktuellste** Backup
  herunterladen. Ältere Sicherungen liegen auf der Management-Instanz unter
  `/usr/local/etc/ansible/opn-backup` und können bei Bedarf von dort
  entnommen werden.
- Der Link **Schedule** führt direkt zu dem Cron-Job "Run central config
  backups" unter **System → Zeitgesteuerte Aufgaben**, über den sich der
  Backup-Rhythmus einstellen lässt.
- Wie lange Backups je Firewall aufbewahrt werden, wird über **Delete Backup
  after (days)** im [Inventar](#inventar-inventory) gesteuert.

## Erweitert: AliasSet, RuleSet, Captive Portal

Unter **Central Management → Advanced** stehen weitere Bausteine zur
Verfügung, die primär dazu dienen, wiederkehrende Kombinationen einmal zu
definieren und dann in Gruppen zu referenzieren:

- **AliasSet** – fasst mehrere bestehende Aliase zu einem benannten Satz
  zusammen, der als Ganzes einer Gruppe zugewiesen werden kann.
- **RuleSet** – fasst mehrere bestehende Firewall-Regeln zu einem benannten
  Satz zusammen, der als Ganzes einer Gruppe zugewiesen werden kann.
- **Captive Portal** – definiert Captive-Portal-Vorlagen (Interface, erlaubte
  Aliase/AliasSets vom Typ MAC, erlaubte Adressen, Template), die über eine
  Gruppe auf die Mitglieds-Firewalls verteilt werden.

Eine Gruppe kann entweder mehrere einzelne Regeln/Aliase **oder** mehrere
RuleSets/AliasSets zugewiesen bekommen, aber nicht beides gleichzeitig
gemischt.

## CM Tools: Config Builder [BETA]

Das Menü **CM Tools → Config Builder** (Beta-Status) erlaubt es, komplette
Grundkonfigurationen für neue oder neu aufzusetzende Firewalls vorzubereiten,
u. a.:

- Netzwerkschnittstellen (physische Zuordnung, VLAN, DHCP/feste IP) und
  Gateways.
- Dnsmasq/DHCP-Bereiche und -Optionen je Interface.
- Benutzer/Gruppen und Authentifizierungsserver.
- Zertifikate (CA/Zertifikat), OpenVPN-Instanzen und Client-Specific-Overrides.
- Verknüpfung mit den zentralen Central-Management-Aliasen und -Regeln.

Da sich diese Funktion noch in der Beta-Phase befindet, sollten damit erzeugte
Konfigurationen vor dem produktiven Einsatz geprüft werden.

## Berechtigungen

Alle Central-Management-Seiten und die zugehörige API liegen unter den
Mustern `ui/cm/*` bzw. `api/cm/*`. Damit weitere Benutzer/Benutzergruppen
(neben `root`) auf das Central Management zugreifen dürfen, muss ihnen unter
**System → Zugriff → Gruppen** die Berechtigung **"CentralManagement"**
zugewiesen werden.

## Support / Fehlerbehebung

- Sollte eine Funktion nicht wie erwartet arbeiten (z. B. veraltete Anzeigen
  im Dashboard), kann im **Dashboard → Tab "Support"** über den Button
  **Clear Cache** der Cache des Plugins geleert werden. Das behebt die
  meisten typischen Anzeigeprobleme.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr`
  dieses Verzeichnisses dokumentiert.
