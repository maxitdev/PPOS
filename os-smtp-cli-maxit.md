# os-smtp-cli-maxit – Generischer SMTP-Relay-Client für OPNsense

`os-smtp-cli-maxit` ist ein Plugin der m.a.x. Informationstechnologie AG,
das einen ausgehenden SMTP-Relay (Smarthost, STARTTLS/TLS, Zugangsdaten)
einmalig auf der Firewall einrichtet – ohne einen vollständigen lokalen
Mailserver wie Postfix betreiben zu müssen. Es ist bewusst als
eigenständiger Baustein angelegt: Andere Plugins (z. B. `os-nmap`) können
darüber E-Mails/Reports verschicken, ohne selbst SMTP-Zugangsdaten
verwalten zu müssen.

## Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Einrichtung](#einrichtung)
4. [Felder der Konfigurationsseite](#felder-der-konfigurationsseite)
5. [Testmail versenden](#testmail-versenden)
6. [Für Entwickler: Wiederverwendung durch andere Plugins](#für-entwickler-wiederverwendung-durch-andere-plugins)
7. [Berechtigungen](#berechtigungen)
8. [Support / Fehlerbehebung](#support--fehlerbehebung)

---

## Voraussetzungen

- Eine OPNsense-Installation mit Zugriff auf das Firmware-Repository, über
  das `os-smtp-cli-maxit` bezogen wird (das Paket `smtp-cli` wird als
  Abhängigkeit automatisch mitinstalliert).
- Zugangsdaten eines bestehenden SMTP-Relays/Smarthosts (Host, Port,
  optional Benutzername/Passwort).

## Installation

1. Unter **System → Firmware → Plugins** nach `os-smtp-cli-maxit` suchen
   und installieren.
2. In der linken Navigation erscheint unter **Services** der neue
   Menüpunkt **SMTP-Send → General**.

## Einrichtung

1. Zu **Services → SMTP-Send → General** wechseln.
2. **Enable** aktivieren.
3. **SMTP host** und **Port** eintragen (gängige Werte: `587` für
   STARTTLS, `465` für implizites TLS/SSL, `25` für unverschlüsselt).
4. **Encryption** passend zum Relay wählen (siehe
   [Felder](#felder-der-konfigurationsseite)).
5. Falls das Relay eine Anmeldung verlangt: **Use authentication**
   aktivieren und **Username**/**Password** eintragen.
6. **From address** eintragen (Absenderadresse für alle über dieses
   Relay verschickten Mails).
7. **Save** klicken. Es gibt keinen separaten "Apply"-Schritt – das
   Plugin hat keinen Dienst und keine Konfigurationsdatei, die zusätzlich
   neu geladen werden müsste; die gespeicherten Werte werden bei jedem
   Mailversand direkt aus der Konfiguration gelesen.
8. Zur Kontrolle **Test recipient** eintragen, erneut **Save** klicken
   und anschließend [eine Testmail versenden](#testmail-versenden).

## Felder der Konfigurationsseite

| Feld | Beschreibung |
|---|---|
| Enable | Schaltet den Relay aktiv/inaktiv. Andere Plugins können den Mailversand nur nutzen, wenn diese Option aktiviert ist. |
| SMTP host | Hostname oder IP des ausgehenden Mail-Relays/Smarthosts. |
| Port | Üblich: `587` (STARTTLS), `465` (implizites TLS/SSL), `25` (unverschlüsselt). |
| Encryption | `None`, `STARTTLS` (z. B. Port 587) oder `TLS/SSL, implicit` (z. B. Port 465). |
| Use authentication | Aktiviert SMTP-AUTH mit den Feldern Username/Password. |
| Username / Password | Zugangsdaten für die Anmeldung am Relay (nur relevant, wenn "Use authentication" aktiviert ist). |
| From address | Absenderadresse, die für alle über dieses Relay verschickten Mails verwendet wird. |
| Test recipient | Empfängeradresse für den Button [Send test email](#testmail-versenden). |

## Testmail versenden

Der Button **Send test email** unterhalb des Formulars verschickt sofort
eine kurze Testmail an die unter **Test recipient** hinterlegte Adresse
und zeigt die Rückmeldung des SMTP-Versands (Erfolg oder Fehlermeldung)
direkt unterhalb des Buttons an. Damit lässt sich die Konfiguration prüfen,
ohne auf ein anderes Plugin angewiesen zu sein.

## Für Entwickler: Wiederverwendung durch andere Plugins

Der eigentliche Zweck dieses Plugins ist es, anderen Plugins eine
schlanke, generische "eine E-Mail verschicken"-Funktion bereitzustellen,
ohne dass diese eigene SMTP-Zugangsdaten verwalten müssen. Es gibt dafür
bewusst keine PHP-API, sondern lediglich eine configd-Aktion, die per
`configdRun()` (oder von einem eigenständigen Skript aus per
`/usr/local/sbin/configctl`) aufgerufen wird:

```php
$backend->configdRun(sprintf(
    'smtpcli send %s %s %s %s',
    escapeshellarg($recipient),
    base64_encode($subject),
    base64_encode($body),
    $attachmentPath !== '' ? escapeshellarg($attachmentPath) : '-'
));
```

- `$recipient` – Empfängeradresse (Klartext, ohne Base64).
- `$subject`, `$body` – Base64-kodiert, da configd Parameter anhand der
  Leerzeichen-Anzahl zählt und Freitext Leerzeichen/Zeilenumbrüche
  enthalten kann.
- `$attachmentPath` – absoluter Pfad zu einer bereits existierenden Datei,
  die als Anhang verschickt werden soll, oder `"-"` für keinen Anhang.
  Größere Reports sollten hier statt im Mailtext übergeben werden: manche
  Mailclients (insbesondere Outlook im Dark Mode) stellen sehr lange
  Klartext-Mailbodies fehlerhaft dar, und quoted-printable-Zeilenumbrüche
  vertragen sich schlechter mit ungewöhnlich langen Einzelzeilen als ein
  Anhang mit eigener Kodierung.

Ist dieses Plugin nicht installiert, aktiviert oder konfiguriert, schlägt
der Aufruf einfach fehl (Exit-Code ungleich 0, Fehlertext auf
stdout/stderr) – ein aufrufendes Plugin sollte diesen Fehlschlag als
"E-Mail konnte nicht verschickt werden" behandeln, nicht als harten
Fehler seiner eigenen Kernfunktion. `os-nmap` ist ein Beispiel für genau
diese Art der (optionalen) Einbindung.

## Berechtigungen

Alle Seiten und die zugehörige API liegen unter den Mustern
`ui/smtpcli/*` bzw. `api/smtpcli/*`. Damit weitere Benutzer/Benutzergruppen
(neben `root`) darauf zugreifen dürfen, muss ihnen unter **System →
Zugriff → Gruppen** die Berechtigung **"Services: smtp-cli"** zugewiesen
werden.

## Support / Fehlerbehebung

- Liefert der Button "Send test email" eine Fehlermeldung, prüft zuerst
  Host/Port/Encryption sowie – falls "Use authentication" aktiviert ist –
  Username/Password.
- Der aktuelle Änderungsverlauf des Plugins ist in der Datei `pkg-descr`
  dieses Verzeichnisses dokumentiert.
