# Dokument 3: Lockdown macOS 14 und 15 mit Firefox ESR, Cold Turkey Pro und Kate

Stand: 31. Oktober 2026. Geprüfte Versionen: macOS 14 Sonoma und macOS 15 Sequoia (jeweils aktuelle Unterversion), Firefox ESR 153.6.0, Cold Turkey Blocker 4.9(2), Kate 26.08 (KDE-Release-Build), Python 3.15.2, Eclipse Temurin JDK 25.0.5, Eclipse JDT Language Server 1.63.0, python-lsp-server 1.15.0, Little Snitch 6. Quellen mit Abrufdatum stehen in Abschnitt 13. Wo sich macOS 14 und 15 unterscheiden, steht es an der Stelle mit „macOS 14:“ und „macOS 15:“.

## 1. Für wen, wofür, Voraussetzungen

Diese Anleitung richtet dir einen Mac so ein, dass du mit Firefox ESR nur freigegebene Webseiten erreichst, mit Kate, Python und Java arbeitest und lernst, und den Lockdown nicht selbst aufheben kannst. Die Sperren trägt eine Vertrauensperson mit eigenem Adminkonto, dem Cold-Turkey-Passwort und dem FileVault-Wiederherstellungsschlüssel; du arbeitest in einem Standardkonto. Jede Sperre nennt den Weg, den sie schließt; was offen bleibt, steht als Restrisiko in Abschnitt 10, und auf einem Mac mit Apple-Chip bleibt mehr offen als auf einem Intel-Mac (7.6).

Voraussetzungen:

- Zwei Personen: du (Standardkonto, „du“) und die Vertrauensperson (Adminkonto, „der Admin“).
- macOS 14 oder 15 auf dem aktuellen Stand; Angabe, ob Intel oder Apple-Chip (Apple-Menü > **Über diesen Mac**).
- Internet während der Einrichtung; die Netzsperre wird in Abschnitt 8 scharf.
- Cold Turkey Pro (Produktschlüssel), Little Snitch 6 (Lizenz, kostenpflichtig), beides verwahrt der Admin.
- Zeit: ein Tag Einrichtung, eine Woche Little Snitch im stillen Erlaubnismodus zum Protokollieren, ein halber Tag Ausbruchsprüfung und Scharfschalten.
- Backup: Time Machine oder Kopie deines Benutzerordners auf ein externes Laufwerk.
- Umschlag beim Admin: Adminkennwort, FileVault-Wiederherstellungsschlüssel, Firmware-Passwort (nur Intel), Cold-Turkey-Passwörter, Bildschirmzeit-Code, Lizenzschlüssel.

Prüfschritte tragen die Kennung „P“ mit Nummer (etwa P13). Die Nummern entsprechen der Liste in Dokument 1, Abschnitt 12; jeder Prüfschritt ist aber an seiner Stelle in dieser Anleitung vollständig beschrieben, mit erwartetem Ergebnis. Ein Prüfschritt steht dort, wo eine Angabe nicht aus Dokumentation belegt werden konnte und am Gerät geprüft werden muss.

Platzhalter: `<DEIN-KONTO>` (Kurzname deines Kontos, Ordner unter `/Users`), `<ADMIN>`, `<STARTSEITE>`. Befehle „als Admin“ laufen im Terminal des Adminkontos; `sudo` fragt nach dem Adminkennwort.

## 2. Schutzschichten

| Schicht | Was sie schließt | Was frei bleibt |
|---|---|---|
| Standardkonto, Adminkonto beim Admin (7.1) | `sudo`, `/Library`, `/Applications`, Systemeinstellungen mit Schloss, Profile entfernen, Cold Turkey und Little Snitch deinstallieren | eigene Einstellungen, `~/Applications`, Terminal |
| FileVault und (Intel) Firmware-Passwort (7.2, 7.6) | Zugriff auf die Platte von außen; Intel: Wiederherstellung und sicherer Modus | Alltag unverändert |
| Konfigurationsprofil Gatekeeper „App Store“ (7.4) | Start heruntergeladener Programme mit Quarantäne-Attribut aus anderen Quellen | alle vom Admin installierten Programme |
| Konfigurationsprofil App Store „Admin zum Installieren nötig“ (7.4) | Installation aus dem Mac App Store durch dich | Updates vom Admin |
| Little Snitch, stiller Verweigerungsmodus (8.1) | Internet für alles außer Firefox, Cold Turkey und Apple-Diensten: Python, Java, `curl`, Homebrew, Git, Kate, Safari, jede fremde App | `localhost` |
| Firefox-Richtlinien in `/Library/Preferences` (4) | Profile, `about:config`, Erweiterungen, private Fenster, Entwicklerwerkzeuge, Proxy; `WebsiteFilter` als zweite Freigabeliste | freigegebene Seiten, lokale Dateien, PDF |
| Cold Turkey mit Sperrtyp Password (6) | nicht freigegebene Webseiten; Safari, App Store, Apps in `~/Applications` und `~/Downloads`; Browser ohne Erweiterung | alles andere |
| Bildschirmzeit (7.5) | zweite Schicht für Safari | |

Profil-Dateien von Firefox (`user.js`, `userChrome.css`) werden nicht genutzt; alles Sicherheitsrelevante liegt außerhalb deines Benutzerordners.

## 3. Vorbereitung

1. Sichern: Time Machine oder Kopie von `/Users/<DEIN-KONTO>`.
2. Installierte Programme notieren: Finder > **Programme**, dazu `~/Applications`, falls vorhanden.
3. Alte Firefox-Profile bereinigen (nichts davon ist sicherheitsrelevant, aber Altlasten sollen weg): Firefox beenden; in `~/Library/Application Support/Firefox/Profiles` alle Ordner außer dem Hauptprofil (`.default-esr`) löschen; in `~/Library/Application Support/Firefox/profiles.ini` nur das Hauptprofil behalten; im Hauptprofil `chrome` und `user.js` löschen, falls vorhanden.
4. Firefox ESR installieren (als Admin): von `https://www.firefox.com/en-US/download/all/desktop-esr/` die Datei **macOS** (DMG, Deutsch) laden [Q97]. Prüfschritt P33: `SHA256SUMS` und `KEY` aus `https://releases.mozilla.org/pub/firefox/releases/153.6.0esr/` laden (laut Community-Antworten dort abgelegt [Q98][Q99]) und vergleichen: `shasum -a 256 Firefox*.dmg`. DMG öffnen, **Firefox** nach **Programme** ziehen. Die App gehört danach dem Admin; du kannst sie nicht ersetzen oder aktualisieren (ein Mozilla-Mitarbeiter beschreibt das in Bug 1893686 [Q96]).
5. Firefox einmal starten, **Firefox** > **Über Firefox** zeigt „153.6.0esr“; beenden.
6. Firefox als Standardbrowser: **Systemeinstellungen** > **Schreibtisch & Dock** > **Standard-Webbrowser** > **Firefox** (macOS 14 und 15 gleich; ohne Adminrechte möglich) [Q124].

Reihenfolge der weiteren Abschnitte: 7.1 (Konten) vor allen Sperren; dann 9 (Kate, Python, Java, Git installieren, solange das Netz offen ist), 4 (Firefox-Richtlinien), 6 (Cold Turkey ungesperrt), 7.2 bis 7.5, 8 (Little Snitch, erst stiller Erlaubnismodus), 10 (Ausbruchsprüfung), dann Scharfschalten. Abschnitt 5 ist leer.

## 4. Firefox-Richtlinien

Firefox liest auf macOS Richtlinien aus der Defaults-Domain `org.mozilla.firefox`, sobald dort `EnterprisePoliciesEnabled` gesetzt ist [Q14][Q93]. Gewählt wird die Admin-Plist `/Library/Preferences/org.mozilla.firefox.plist`: Sie gehört `root`, du kannst sie nicht ändern, sie überlebt Firefox-Updates, und Admin-Quellen haben Vorrang vor allem, was du in `~/Library/Preferences` setzen könntest [Q12]. Nicht gewählt: `policies.json` im App-Bundle (Verhalten bei Updates von Mozilla nicht dokumentiert, Bug 1792595 [Q94]; Einfügen ins signierte Bundle löste in einem Community-Fall eine „beschädigt“-Warnung aus [Q95]). Ein Konfigurationsprofil mit `mcx_preference_settings` wäre gleichwertig [Q14]; die Plist ist der von Mozilla selbst beschriebene Weg und einfacher zu prüfen.

Einschränkung: `DisableSafeMode` wirkt auf macOS nur im Menü; der Start mit `-safe-mode` oder gehaltener Wahltaste bleibt möglich, weil der Startcode keine Richtlinie prüft [Q6][Q88]. Im Fehlerbehebungsmodus sind Erweiterungen aus [Q87]; dann schließt Cold Turkey den Browser nach der Grace Period (6.5), und `WebsiteFilter` wirkt weiter, weil er in der Plist liegt. Deshalb ist `WebsiteFilter` hier Pflicht.

### 4.1 Freigabeliste

Gilt zweimal: hier für `WebsiteFilter` (Match-Patterns [Q8]) und in Abschnitt 6 für Cold Turkey (Domains mit `*` [Q16]).

| Zweck | WebsiteFilter-Muster | Cold-Turkey-Eintrag |
|---|---|---|
| Python-Dokumentation | `https://docs.python.org/*` | `docs.python.org` |
| Java-Dokumentation | `https://docs.oracle.com/en/java/*`, `https://dev.java/*` | `docs.oracle.com/en/java`, `dev.java` |
| Kate-Dokumentation | `https://docs.kde.org/*`, `https://kate-editor.org/*` | `docs.kde.org`, `kate-editor.org` |
| deine Seiten aus Cold Turkey | `https://<domain>/*` je Eintrag | wie hinterlegt |

Ohne freigegebene Suchmaschine landet jede Suche aus der Adressleiste auf der Sperrseite; das ist gewollt.

### 4.2 Richtliniendatei

Speichere den Block als `firefox-policies.plist`. Vorher ersetzen: `<STARTSEITE>`, die Ausnahmen in `WebsiteFilter`, die Cold-Turkey-Erweiterungs-ID `<COLD-TURKEY-ID>` (6.2). Syntax je Richtlinie nach der Referenz [Q4]; das Plist-Format der Beispiele steht dort je Richtlinie (zum Beispiel `ExtensionSettings` [Q9], `UserMessaging` [Q4], `Preferences` [Q5]).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>EnterprisePoliciesEnabled</key><true/>

  <key>BlockAboutConfig</key><true/>
  <key>BlockAboutSupport</key><true/>
  <key>BlockAboutAddons</key><true/>
  <key>BlockAboutProfiles</key><true/>
  <key>DisableProfileRefresh</key><true/>
  <key>DisableProfileImport</key><true/>
  <key>DisableSafeMode</key><true/>
  <key>DisableDeveloperTools</key><true/>
  <key>PrivateBrowsingModeAvailability</key><integer>1</integer>
  <key>DisableFirefoxAccounts</key><true/>
  <key>DisableAccounts</key><true/>
  <key>DisableTelemetry</key><true/>
  <key>DisableFirefoxStudies</key><true/>
  <key>DisableFeedbackCommands</key><true/>
  <key>DontCheckDefaultBrowser</key><true/>
  <key>SkipTermsOfUse</key><true/>
  <key>ExtensionUpdate</key><true/>
  <key>ManualAppUpdateOnly</key><true/>

  <key>NewTabPage</key><false/>
  <key>OverrideFirstRunPage</key><string></string>
  <key>OverridePostUpdatePage</key><string></string>
  <key>Homepage</key>
  <dict>
    <key>URL</key><string>&lt;STARTSEITE&gt;</string>
    <key>Locked</key><true/>
    <key>StartPage</key><string>homepage</string>
  </dict>

  <key>Proxy</key>
  <dict>
    <key>Mode</key><string>none</string>
    <key>Locked</key><true/>
  </dict>

  <key>DownloadDirectory</key><string>${home}/Downloads</string>
  <key>PromptForDownloadLocation</key><false/>

  <key>UserMessaging</key>
  <dict>
    <key>ExtensionRecommendations</key><false/>
    <key>FeatureRecommendations</key><false/>
    <key>UrlbarInterventions</key><false/>
    <key>SkipOnboarding</key><true/>
    <key>MoreFromMozilla</key><false/>
    <key>FirefoxLabs</key><false/>
    <key>Locked</key><true/>
  </dict>

  <key>ExtensionSettings</key>
  <dict>
    <key>*</key>
    <dict>
      <key>installation_mode</key><string>blocked</string>
      <key>blocked_install_message</key><string>Erweiterungen werden vom Admin verwaltet.</string>
    </dict>
    <key>uBlock0@raymondhill.net</key>
    <dict>
      <key>installation_mode</key><string>force_installed</string>
      <key>install_url</key><string>https://addons.mozilla.org/firefox/downloads/latest/uBlock0@raymondhill.net/latest.xpi</string>
    </dict>
    <key>{ef87d84c-2127-493f-b952-5b4e744245bc}</key>
    <dict>
      <key>installation_mode</key><string>force_installed</string>
      <key>install_url</key><string>https://addons.mozilla.org/firefox/downloads/latest/{ef87d84c-2127-493f-b952-5b4e744245bc}/latest.xpi</string>
    </dict>
    <key>&lt;COLD-TURKEY-ID&gt;</key>
    <dict>
      <key>installation_mode</key><string>force_installed</string>
      <key>install_url</key><string>https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi</string>
      <key>private_browsing</key><true/>
    </dict>
  </dict>

  <key>Preferences</key>
  <dict>
    <key>browser.profiles.enabled</key>
    <dict>
      <key>Value</key><false/>
      <key>Status</key><string>locked</string>
    </dict>
  </dict>

  <key>WebsiteFilter</key>
  <dict>
    <key>Block</key>
    <array>
      <string>&lt;all_urls&gt;</string>
    </array>
    <key>Exceptions</key>
    <array>
      <string>https://docs.python.org/*</string>
      <string>https://docs.oracle.com/en/java/*</string>
      <string>https://dev.java/*</string>
      <string>https://docs.kde.org/*</string>
      <string>https://kate-editor.org/*</string>
    </array>
  </dict>
</dict>
</plist>
```

Hinweis zur Schreibweise: In XML müssen `<STARTSEITE>` und `<COLD-TURKEY-ID>` beim Ersetzen ohne spitze Klammern eingetragen werden; `&lt;all_urls&gt;` bleibt so, das ist die XML-Schreibweise für `<all_urls>`.

Einspielen als Admin:

```bash
sudo defaults import /Library/Preferences/org.mozilla.firefox ./firefox-policies.plist
sudo defaults read /Library/Preferences/org.mozilla.firefox
sudo chown root:wheel /Library/Preferences/org.mozilla.firefox.plist
sudo chmod 644 /Library/Preferences/org.mozilla.firefox.plist
```

Prüfschritt: `man defaults` auf diesem Mac zeigt `import` (die ältere Manpage kennt nur `write`; Community-Skripte nutzen `defaults import domain path` [Q201]). Fehlt `import`, stattdessen `sudo defaults write /Library/Preferences/org.mozilla.firefox "$(cat ./firefox-policies.plist)"` (die Manpage erlaubt `write domain 'plist'`, also ein vollständiges Wörterbuch als Wert [Q201]). Mozilla selbst beschreibt `sudo defaults write /Library/Preferences/org.mozilla.firefox EnterprisePoliciesEnabled -bool TRUE` als Grundform [Q14].

### 4.3 Wirkung je Richtlinie

| Richtlinie | Wirkung | Beleg |
|---|---|---|
| `EnterprisePoliciesEnabled` | schaltet das Lesen der Plist ein; ohne diesen Key ignoriert Firefox alles andere | Q93 |
| `BlockAboutConfig`, `BlockAboutSupport`, `BlockAboutAddons` | sperren `about:config`, `about:support`, `about:addons` | Q4 |
| `BlockAboutProfiles` | sperrt `about:profiles` und die Seiten des neuen Profil-Managers, schaltet das Profil-Flyout ab | Q10, Q84 |
| `DisableProfileRefresh`, `DisableProfileImport` | kein „Firefox bereinigen“, kein Import | Q4 |
| `DisableSafeMode` | entfernt den Menüpunkt; Start mit `-safe-mode` bleibt (siehe oben) | Q6, Q88 |
| `DisableDeveloperTools` | keine Entwicklerwerkzeuge, keine Tastenkürzel, kein `about:debugging` | Q4, Q137 |
| `PrivateBrowsingModeAvailability` 1 | keine privaten Fenster | Q7 |
| `DisableFirefoxAccounts`, `DisableAccounts` | kein Konto, kein Sync | Q4 |
| `DisableTelemetry`, `DisableFirefoxStudies`, `DisableFeedbackCommands`, `DontCheckDefaultBrowser`, `SkipTermsOfUse` | keine Datensendungen, Studien, Meldungen, Dialoge | Q4 |
| `ExtensionUpdate` | erzwungene Erweiterungen bleiben aktuell | Q4 |
| `ManualAppUpdateOnly` | keine Update-Hinweise an dich; der Admin aktualisiert im Wartungsfenster (11) | Q4 |
| `NewTabPage` false, Override-Seiten leer, `Homepage` gesperrt | leerer neuer Tab, Start auf der Startseite | Q4 |
| `Proxy` none, Locked | kein Proxy | Q11 |
| `DownloadDirectory`, `PromptForDownloadLocation` | Downloads nach `~/Downloads`, kein Dialog | Q4 |
| `UserMessaging` | keine Empfehlungen | Q4 |
| `ExtensionSettings` | alle Erweiterungen gesperrt außer uBlock Origin, ActivityWatch, Cold Turkey (erzwungen); `install_url` für die Cold-Turkey-XPI nötig | Q9 |
| `Preferences` `browser.profiles.enabled` | zweites Netz gegen das Profil-Feature | Q5, Q13 |
| `WebsiteFilter` | zweite Freigabeliste; wirkt auch ohne Erweiterungen | Q8 |

### 4.4 Verifikation

1. Firefox starten, `about:policies`: Reiter **Aktiv** zeigt alle Richtlinien, **Fehler** leer.
2. uBlock Origin, ActivityWatch und Cold Turkey sind installiert, nicht entfernbar.
3. `about:config` zeigt die Sperrseite; **Befehl+Umschalt+P** öffnet kein privates Fenster; eine nicht freigegebene Seite zeigt die Sperrseite.
4. Eigene Benutzerrichtlinien können Admin-Richtlinien nicht überschreiben; Prüfschritt in 10.1, Zeile F23.

## 5. Profil

Keine Maßnahmen; Begründung wie in Abschnitt 2. Ein neues Profil (`/Applications/Firefox.app/Contents/MacOS/firefox -P`) hat dieselben Sperren, weil sie in `/Library/Preferences` liegen [Q86].

## 6. Cold Turkey

Bezeichnungen nach dem User Guide [Q16]. Die Mac-Version unterscheidet sich von Windows: Anwendungen werden über **Add application**, **Add folder** und **Add binary** hinzugefügt, bleiben auch nach Umbenennen oder Verschieben gesperrt, und es gibt kein **Add window title** [Q16]. Safari wird unterstützt, ohne Block-Seite [Q18].

### 6.1 Installation

1. Als Admin `Cold_Turkey_Mac_Installer.pkg` von `https://getcoldturkey.com/download/mac/` laden (Version 4.9(2) [Q30]) und installieren; Adminrechte sind Pflicht [Q18]. Ziel `/Applications/Cold Turkey Blocker.app`.
2. In deinem Konto (nach 7.1) Cold Turkey starten; es zeigt gefundene Browser und bietet **Install** je Browser [Q16]. Für Firefox **Install**; für Safari nichts (Safari wird gesperrt, die Safari-Erweiterung bräuchte zudem Vollzugriff auf die Festplatte für Cold Turkey [Q36]).
3. Pro freischalten: Produktschlüssel in **Settings** eintragen (Admin) [Q32].
4. Berechtigungen: Der Hersteller nennt nur Adminrechte für die Installation und Vollzugriff auf die Festplatte für die Safari-Prüfung [Q18][Q36]. Prüfschritt: Fragt macOS beim ersten Start nach Berechtigungen (Bedienungshilfen, Bildschirmaufnahme), gewähren und in 10.1 Zeile T0 eintragen.

### 6.2 Firefox-Erweiterung und Erweiterungs-ID (Prüfschritt P1)

Die Erweiterung liegt beim Hersteller unter `https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi` [Q34]. ID auslesen (als Admin im Terminal):

```bash
cd ~/Downloads
curl -O https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi
mkdir ct && cd ct && unzip -q ../Cold_Turkey_Firefox_Addon.xpi
grep -A3 '"gecko"' manifest.json
```

Die Zeile `"id"` ist die ID; in `firefox-policies.plist` eintragen und 4.2 erneut einspielen. Erwartung: Erweiterung nach Firefox-Neustart vorhanden, `about:policies` ohne Fehler. Bleibt sie aus (XPI nicht von Mozilla signiert), Zeile aus der Plist entfernen und die Erweiterung über Cold Turkeys **Install** in 6.1 einrichten; Browser ohne Erweiterung schließt Cold Turkey ohnehin (6.5).

### 6.3 Blocklisten

| | Block „Lockdown“ | Block „Ablenkung“ |
|---|---|---|
| Zweck | Freigabeliste; Safari, App Store, Zweitbrowser, Benutzerordner-Apps sperren | Ablenkungs-Apps ohne Web (E11) |
| Typ | continuous | continuous |
| Websites | `*.*` | keine |
| Website Exceptions | `file://*` (Standard, lokale Dateien [Q16]), `localhost` (laut Release Notes zulässig [Q30]), Einträge der Freigabeliste 4.1 | keine |
| Applications | siehe 6.4 | Spiele, Apple Arcade, Musik, TV, Podcasts, News, Aktien (nach deiner Wahl) |
| Breaks | No breaks | No breaks |
| Lock | Password, erst nach Abschnitt 10 | Password |

Keine Zeitpläne, damit die Systemzeit keine Rolle spielt. Keine `about:`- oder Erweiterungs-Schemata in den Ausnahmen; Prüfschritt P2 (neuer Tab) in 10.1.

### 6.4 Anwendungssperren (Block „Lockdown“)

| Eintrag | Methode | Weg, den die Sperre schließt |
|---|---|---|
| Safari | **Add application** > `/Applications/Safari.app` | zweiter Browser (nicht deinstallierbar, SIP [Q123]) |
| App Store | **Add application** > `/System/Applications/App Store.app` | Installation fremder Apps (zweite Schicht neben 7.4) |
| Microsoft Edge, Google Chrome und andere Browser, falls installiert | **Add application** | zweiter Browser; besser: als Admin aus `/Applications` löschen |
| `~/Applications` | **Add folder** > `/Users/<DEIN-KONTO>/Applications` (Ordner vorher anlegen) | Apps, die du ohne Adminrechte installierst |
| `~/Downloads`, `~/Desktop`, `~/Documents` | **Add folder** | Apps, die du dorthin entpackst oder schreibst; Prüfschritt P26: Cold Turkey beendet auch eine neu erzeugte App in diesen Ordnern |
| Terminal | nicht sperren | Entwicklung; Netz ist über 8.1 geschlossen |

Nicht sperren: Finder, Systemeinstellungen, Terminal, Aktivitätsanzeige, Firefox, Cold Turkey Blocker, Kate, Python Launcher, `python3`, `java`, Little Snitch, Textedit, Vorschau, Rechner, Notizen, Fotos. **Add binary** wird nicht genutzt.

Hinweiskasten: Sperre nie Finder, Systemeinstellungen oder Cold Turkey selbst; mit gesperrtem Block kann nur der Admin die Liste ändern.

### 6.5 Einstellungsseite

| Reiter | Option | Wert | Grund |
|---|---|---|---|
| Preferences | Open on startup, Move to system tray, Show extension install dialog | an | Oberfläche sichtbar; Hinweis bei Browser ohne Erweiterung |
| Notifications | alle | an | du siehst, was gesperrt wurde |
| Blocking | Block unsupported browsers | an | unbekannter Browser |
| Blocking | Block browsers if extension crashes | an | Erweiterung beendet |
| Blocking | Block time changes | an | Systemzeit (blockiert die Systemeinstellungen beim Ändern [Q16]); Prüfschritt P30, ob ein Standardkonto die Zeit überhaupt ändern darf |
| Blocking | Block task manager | aus | Aktivitätsanzeige bleibt zur Fehlersuche; Prozesse des Admins kannst du nicht beenden |
| Blocking | Block inactive browser tabs, Block embedded content, Block Pause for a Cause | an | wie Windows: Hintergrund-Tabs, eingebettete Inhalte, Spendenpause |
| Blocking | Block the installer | an | keine Neuinstallation oder Updates ohne Admin |
| Blocking | Force allow access to file URLs | aus | lokale Dateien bleiben über die Standardausnahme offen |
| Blocking | Grace period | 60 Sekunden | Standard |
| Blocking | Option für Bildschirmzeit-Einstellungen („Block screen time settings“, nur auf der Features-Seite genannt [Q31]) | an, falls vorhanden (Prüfschritt) | Bildschirmzeit-Schicht gegen Änderung schützen |
| Statistics | Enable statistics | an | Übersicht |
| Password | Password Settings | an, Admin-Passwort (6.7) | Oberfläche |

### 6.6 Browserschutz

Browser ohne aktive Erweiterung werden während eines gesperrten Blocks nach der Grace Period geschlossen [Q16]; Firefox Developer Edition oder Nightly in `~/Applications` liegen zusätzlich in einem gesperrten Ordner (6.4) und haben keine Richtlinien (eigene Bundle-ID [Q100]); mit Cold-Turkey-Erweiterung würde dort die Cold-Turkey-Freigabeliste gelten, nicht `WebsiteFilter` und nicht uBlock (Restrisiko F21). Private Fenster sind per Richtlinie aus.

### 6.7 Passwort und Scharfschalten

Wie der Umschlag: Der Admin erzeugt zwei Passwörter mit 24 oder mehr zufälligen Zeichen. Reihenfolge: Blöcke eingeschaltet und ungesperrt bis zur bestandenen Ausbruchsprüfung (10); dann je Block **Lock** > **Password**; dann **Settings** > **Password** > **Password Settings**. Verwahrung beim Admin. Notfall: nur der Admin entsperrt; Reparaturen im Wartungsfenster (11).

### 6.8 Ist-Soll-Abgleich

Für den Mac gab es keine Vorlage. Gegenüber der Windows-Dokumentation mit Edge gilt: Websperren als Freigabeliste statt Einzelsperren; keine Prozess-Sperrliste nach Windows-Muster, sondern Apps und Ordner; Safari statt Edge; Shells, Python, Java frei; Zeitpläne entfallen; die Begriffe „Strict Mode“, „Block uninstalling Cold Turkey“, „Block changing system time“ gibt es nicht, sie heißen Sperrtyp Password, „Block the installer“, „Block time changes“ [Q16].

## 7. Betriebssystem

### 7.1 Konten

Administratoren verwalten Benutzer, installieren Apps und ändern Einstellungen; Standardbenutzer können Apps installieren und eigene Einstellungen ändern, aber keine Benutzer anlegen [Q89]. Ziel: der Admin hat ein eigenes Adminkonto, dein Konto ist Standard, das Gastkonto ist aus.

1. Als heutiger Admin: **Systemeinstellungen** > **Benutzer:innen & Gruppen** (macOS 14 und 15 gleich) > **Account hinzufügen…** > Kontotyp **Administrator:in**, Name `<ADMIN>`, Kennwort tippt die Vertrauensperson [Q90].
2. Abmelden, als `<ADMIN>` anmelden, prüfen, dass das Konto in der Liste als Admin steht.
3. Dein Konto herabstufen: **Benutzer:innen & Gruppen** > Info-Taste neben deinem Konto > Schalter **Benutzer:in darf diesen Computer verwalten** aus. Prüfschritt: Die genaue Beschriftung des Schalters in 14 und 15 am Bildschirm prüfen; Apple nennt auf der Hilfeseite nur den Weg über die Info-Taste [Q90].
4. **Gastbenutzer:in** > **Gast:innen erlauben, sich an diesem Computer anzumelden** aus [Q90].
5. Prüfschritt in deinem Konto: `sudo -v` im Terminal verlangt ein Kennwort und lehnt deines ab; **Systemeinstellungen** > **Datenschutz & Sicherheit** zeigt beim Schloss eine Admin-Abfrage.

Damit geschlossen: `sudo`, Schreiben in `/Library` und `/Applications` (Prüfschritt P22: `ls -ld /Applications` zeigt `root:admin` mit `drwxrwxr-x`), Profile entfernen (nur mit Admin-Anmeldedaten [Q91]), Little Snitch und Cold Turkey deinstallieren, Systemerweiterungen freigeben [Q117].

Hinweiskasten: Das Adminkennwort darf nur der Admin kennen. Mit ihm fällt jede Maßnahme dieser Anleitung.

### 7.2 FileVault

Weg: Platte ausbauen oder von außen lesen. Als Admin: **Systemeinstellungen** > **Datenschutz & Sicherheit** > **FileVault** > **Einschalten**; Option **Wiederherstellungsschlüssel erstellen und meinen iCloud-Account nicht verwenden**; Schlüssel notieren (Umschlag); bei weiteren Benutzern **Benutzer:innen aktivieren** und dein Kennwort eingeben, damit du den Mac starten kannst. Pfad in macOS 14 und 15 gleich [Q197]. FileVault erfordert einen Administrator; ohne Anmeldedaten oder Schlüssel bleiben die Daten verschlüsselt [Q197].

Folge für die Wiederherstellung: siehe 7.6; FileVault allein schließt die Wiederherstellung für dich nicht, weil dein Kontokennwort das Volume entsperrt.

### 7.3 Dateirechte

Vom Admin installierte Apps in `/Applications` gehören dem Admin; `/Library/Preferences/org.mozilla.firefox.plist` gehört `root` (4.2); `/Library/Wheels` (8.2) gehört `root`. Prüfschritt in deinem Konto: `touch /Applications/x` und `touch /Library/Preferences/x` verweigern.

### 7.4 Konfigurationsprofil: Gatekeeper und App Store

Gatekeeper prüft heruntergeladene Software mit Quarantäne-Attribut beim ersten Öffnen; Benutzer können die Prüfung übergehen, wenn kein Gerätemanagement das verhindert [Q27]. macOS 14: Übergehen per Ctrl-Klick > **Öffnen** möglich. macOS 15: Ctrl-Klick entfällt; Freigabe nur über **Systemeinstellungen** > **Datenschutz & Sicherheit** > **Dennoch öffnen** [Q37][Q102]. Das Profil setzt die Einstellung **Apps erlauben, die geladen wurden von** auf **App Store** und verhindert damit beides für dich; zusätzlich verlangt der App Store einen Admin für Installationen.

Payloads: `com.apple.systempolicy.control` mit `EnableAssessment` und `AllowIdentifiedDevelopers`, manuell installierbar [Q103]; `com.apple.appstore` mit `restrict-store-require-admin-to-install` (Apple-Payload; der Schlüsselname stammt aus der MacAdmins-Dokumentation der Apple-Profilreferenz, Prüfschritt P28) [Q198]. Profilstruktur nach Apples Beispiel [Q198][Q203]. Datei `Lockdown.mobileconfig`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadType</key><string>com.apple.systempolicy.control</string>
      <key>PayloadIdentifier</key><string>lockdown.gatekeeper</string>
      <key>PayloadUUID</key><string>5B2C1F6E-2F3A-4C7B-9D21-0A1B2C3D4E5F</string>
      <key>PayloadVersion</key><integer>1</integer>
      <key>PayloadDisplayName</key><string>Gatekeeper nur App Store</string>
      <key>EnableAssessment</key><true/>
      <key>AllowIdentifiedDevelopers</key><false/>
    </dict>
    <dict>
      <key>PayloadType</key><string>com.apple.appstore</string>
      <key>PayloadIdentifier</key><string>lockdown.appstore</string>
      <key>PayloadUUID</key><string>7D4E2A90-6B1C-4F3D-8E52-1B2C3D4E5F60</string>
      <key>PayloadVersion</key><integer>1</integer>
      <key>PayloadDisplayName</key><string>App Store nur Admin</string>
      <key>restrict-store-require-admin-to-install</key><true/>
    </dict>
  </array>
  <key>PayloadDisplayName</key><string>Lockdown</string>
  <key>PayloadIdentifier</key><string>lockdown.profile</string>
  <key>PayloadType</key><string>Configuration</string>
  <key>PayloadUUID</key><string>9A8B7C6D-5E4F-4A3B-8C2D-1E0F9A8B7C6D</string>
  <key>PayloadVersion</key><integer>1</integer>
  <key>PayloadScope</key><string>System</string>
</dict>
</plist>
```

Installation als Admin: Datei doppelklicken, dann
macOS 14: **Systemeinstellungen** > **Datenschutz & Sicherheit** > **Profile** > unter **Geladen** das Profil doppelklicken > **Installieren** [Q92].
macOS 15: **Systemeinstellungen** > **Allgemein** > **Geräteverwaltung** > unter **Geladen** doppelklicken > **Installieren** [Q92][Q28].
Entfernen kann ein manuell installiertes Profil nur, wer Name und Kennwort eines lokalen Admins kennt [Q91].

Prüfschritt P27: In deinem Konto zeigt **Datenschutz & Sicherheit** die Auswahl auf **App Store** ausgegraut; ein mit Firefox geladenes Programm eines verifizierten Entwicklers startet nicht und bietet kein **Dennoch öffnen**. Prüfschritt P28: **App Store** > Installieren einer kostenlosen App verlangt Admin-Anmeldedaten.

Was das Profil nicht schließt: Programme ohne Quarantäne-Attribut (von `curl`, Python oder einem USB-Stick), denn Gatekeeper prüft nur quarantänierte Dateien (Aussagen von Apple-DTS; Apple-Doku nur indirekt über `LSFileQuarantineEnabled` [Q104][Q105]). Dafür: Netzsperre (8.1), Cold-Turkey-Ordnersperren (6.4), Restrisiko B5 für Datenträger; Santa als Option im Anhang.

### 7.5 Bildschirmzeit

Zweite Schicht für Safari, nicht tragend: Die Apple-Seiten beschreiben, dass Ändern oder Entfernen des Codes mit dem Code oder mit Passwort bzw. Touch ID möglich ist [Q120][Q121]; Prüfschritt P25, ob dein Anmeldekennwort reicht. Als Admin in deinem Konto:

macOS 14: **Systemeinstellungen** > **Bildschirmzeit** > **Inhalt & Datenschutz** > einschalten > **Store-, Web-, Siri- und Game Center-Inhalte** > Webinhalte **Nur erlaubte Websites**, Liste leeren; zurück, **Bildschirmzeit sperren** mit vierstelligem Code (Admin) [Q120].
macOS 15: **Systemeinstellungen** > **Bildschirmzeit** > **Inhalt & Datenschutz** > einschalten > **App Store, Medien, Web & Spiele** > Zeile **Safari** auf **Nur erlaubte Websites**, Liste leeren; **Einstellungen für Bildschirmzeit sperren** mit Code [Q121].
App-Beschränkungen gibt es auf dem Mac nur für Kamera, Book Store, Siri & Diktat, SharePlay; Safari lässt sich dort nicht abschalten [Q122]. Ob der Webfilter Firefox betrifft, sagt Apple nicht; falls Firefox freigegebene Seiten nicht lädt, Webinhalte auf **Unbeschränkter Zugriff** stellen (Prüfschritt).

### 7.6 Wiederherstellung und sicherer Modus

Intel: Firmware-Passwort setzen (Admin): macOS-Wiederherstellung starten (**Befehl-R**), **Dienstprogramme** > **Startsicherheitsdienstprogramm**, Adminkennwort eingeben, Firmwarepasswort aktivieren, Passwort auf den Umschlag [Q202]. Ohne Passwort startet der Mac nur vom Startvolume, nicht in die Wiederherstellung oder von anderen Volumes [Q202]; der sichere Modus wird auf dieser Seite nicht genannt (Prüfschritt P24).

Apple-Chip: kein Firmware-Passwort; Apple verweist auf FileVault und auf eine Fernsperre per MDM [Q202][Q127]. Die Wiederherstellung öffnet sich nach Auswahl eines Benutzers, dessen Kennwort bekannt ist [Q126]; ob dein Standardkonto dort zugelassen ist und das Datenvolume entsperren kann, ist nicht belegt (Prüfschritt P23). Falls ja, bleibt der Weg offen: Im Terminal der Wiederherstellung ließen sich `/Library/Preferences/org.mozilla.firefox.plist`, Little Snitch und Cold Turkey entfernen. Restrisiko B14; Optionen: MDM mit Wiederherstellungs-Sperrkennwort (nur per MDM, macOS 11.5 und neuer [Q127]) oder Intel-Gerät. Diese Entscheidung steht in Dokument 1, E17, und wird nach P23 getroffen.

Sicherer Modus (Umschalttaste beim Start, Apple-Chip: Startoptionen > Volume > Umschalt > **Im gesicherten Modus fortfahren**) verhindert Anmeldeobjekte und nicht benötigte Erweiterungen [Q129]; ob der Cold-Turkey-Dienst und die Little-Snitch-Netzwerkerweiterung dort laufen, ist nicht belegt. Firefox-Richtlinien gelten dort weiter. Prüfschritt P24 in 10.1.

### 7.7 Homebrew, Terminal, Shells

Nicht gesperrt. Homebrew ließe sich auch ohne Adminrechte installieren [Q130][Q131], scheitert aber ohne Netz beim Herunterladen (8.1). Terminal, `zsh`, `bash`, `python3`, `java`, `git` laufen; nichts davon erreicht das Internet. Apples `/usr/bin/python3` (Stub der Command Line Tools) wird nicht genutzt [Q132].

## 8. Ausgleichsmaßnahmen

### 8.1 Netz: Little Snitch 6 im stillen Verweigerungsmodus

Die eingebaute macOS-Firewall steuert nur eingehende Verbindungen; ausgehende erwähnt Apple nicht [Q112][Q113]. Little Snitch 6 filtert ausgehende Verbindungen je Programm nach Code-Identität (Team-ID, Identifier), unabhängig vom Pfad, auch nach Verschieben und Update; unsignierte Programme nach Hash [Q115]. Im Modus **Silent Deny** werden Verbindungen ohne passende Regel ohne Rückfrage verweigert [Q199]. Die Sicherheitseinstellungen liegen hinter einem Schloss mit Administratorkennwort: **Allow Rule Editing**, **Allow Profile Switching**, **Allow Settings Editing**, **Allow Global Rule Editing**, **Allow access via Terminal** lassen sich abschalten [Q116]. Beenden und Deinstallation durch dich: Die Doku nennt für die Deinstallation keinen Kennwortzwang [Q119]; die Netzwerkerweiterung kann aber nur ein Admin freigeben oder entfernen [Q117], und `/Applications` ist für dich nicht beschreibbar (7.3).

Gewählt statt LuLu, weil Little Snitch den Regelschutz dokumentiert; LuLu dokumentiert Authentifizierung nur für Beenden und Deinstallation, nicht für Regeländerungen [Q114].

Installation als Admin: Little Snitch nach **Programme** [Q118]; beim ersten Start Netzwerkfilter zustimmen und die Systemerweiterung unter **Datenschutz & Sicherheit** erlauben (macOS 14 und 15; die Freigabe verlangt einen Administrator [Q117]).

Regeln (als Admin, Programm **Little Snitch Rules**, **Create and edit rules** [Q200]), alle mit Besitzer **Any user** (Regeln für einen bestimmten Besitzer haben Vorrang; „Any user“ deckt dein Konto ab [Q200]):

| Programm | Richtung | Aktion | Grund |
|---|---|---|---|
| Firefox (`/Applications/Firefox.app`) | ausgehend, alle Ziele | Allow | Browser; Inhalte filtern Cold Turkey und `WebsiteFilter` |
| Cold Turkey Blocker (App und Hilfsprozesse) | ausgehend | Allow | Lizenz, Updates im Wartungsfenster |
| Apple-Systemdienste (vordefinierte Regelgruppe „macOS Services“ oder gleichnamig, Prüfschritt) | ausgehend | Allow | Softwareupdate, Zeit, Apple-Dienste |
| Little Snitch selbst | ausgehend | Allow | Lizenz, Updates |
| alles andere | ausgehend | keine Regel | im Silent-Deny-Modus verweigert: `python3`, `java`, `curl`, `git`, Kate, `kioworker`, Safari, App Store, Homebrew |
| Wartungsregeln: `python3` (`/Library/Frameworks/Python.framework/Versions/3.15/Resources/Python.app`), `git`, `curl` | ausgehend | Allow, **deaktiviert** | der Admin aktiviert sie im Wartungsfenster (11.2) |

Protokollwoche: zuerst **Silent Allow** (Verbindungen erlaubt, Network Monitor merkt Regelbedarf) [Q199], eine Woche normal arbeiten, dann im Network Monitor die Apple-Dienste prüfen, die Regeln brauchen, und Allow-Regeln anlegen. Danach **Silent Deny** (Prüfschritt P29: Modus und Name in Little Snitch 6 am Bildschirm; in den Sicherheitseinstellungen **Allow Rule Editing**, **Allow Profile Switching**, **Allow Settings Editing** aus, Schloss schließen).

Hinweiskasten: Schalte Silent Deny erst nach der Protokollwoche ein. Ohne Allow-Regel für Apples Softwareupdate fallen Sicherheitsupdates still aus; der Network Monitor zeigt verweigerte Verbindungen.

Loopback (Prüfschritt P4): `python3 -m http.server 8000 --bind 127.0.0.1`, in Firefox `http://localhost:8000/`; erwartet: Verzeichnisliste. Falls verweigert, Allow-Regel für Ziel `localhost` für **Any process**.

Prüfschritte nach dem Scharfschalten (dein Konto):

```bash
python3 -c "import urllib.request; urllib.request.urlopen('https://example.org')"   # erwartet: URLError
curl -I https://example.org                                                          # erwartet: Fehler
```

venv-Interpreter: `venv` legt auf macOS Symlinks auf den Framework-Interpreter an [Q25][Q133]; die Code-Identität bleibt die des Frameworks (Ableitung aus [Q115], Prüfschritt P6: aus der venv heraus `urlopen` scheitert, `pylsp` läuft).

Restrisiko: ein Apple-Dienst ohne Regel fällt still aus (Network Monitor zeigt es); Wartungsregeln bleiben nur im Wartungsfenster aktiv.

### 8.2 Paketinstallation

Vor dem Scharfschalten installiert der Admin Python, `pip`, `python-lsp-server`, `debugpy`, JDK, JDT LS, Kate, Git (9). Danach: `pip` ohne Netz. Wheel-Ordner als Admin:

```bash
sudo mkdir -p /Library/Wheels
sudo touch /Library/Wheels/Wunschliste.txt
sudo chown root:wheel /Library/Wheels /Library/Wheels/Wunschliste.txt
sudo chmod 755 /Library/Wheels
sudo chmod 666 /Library/Wheels/Wunschliste.txt
```

Im Wartungsfenster (Wartungsregel `python3` aktiv): `sudo /usr/local/bin/python3 -m pip download -d /Library/Wheels <paket>`. Du installierst offline: `./venv/bin/python -m pip install --no-index --find-links /Library/Wheels <paket>` (Prüfschritt P21 gegen die pip-Doku). Java: Ordner mit `.java`-Dateien brauchen kein Netz [Q71]; Maven oder Gradle nur nach `mvn dependency:go-offline` im Wartungsfenster (P35) [Q165].

### 8.3 Ausführung

Lücke benannt: Gatekeeper prüft nur quarantänierte Dateien (7.4); ein Programm, das Python schreibt, `curl` lädt (ohne Netz nicht möglich) oder ein USB-Stick bringt, läuft ohne Prüfung. Maßnahmen ohne Allowlist: Netzsperre (8.1), Gatekeeper-Profil (7.4), Cold-Turkey-Ordnersperren für `~/Applications`, `~/Downloads`, `~/Desktop`, `~/Documents` (6.4, Prüfschritt P26). Was bleibt: ein Programm von einem Datenträger in einem anderen Benutzerordner (Restrisiko B5); ein selbstgeschriebenes Programm ohne Netz (Restrisiko I8, kein Weg ins Internet).

Santa (Allowlist, Lockdown-Modus) als geprüfte Option: Anhang A. Apples eigene App-Allowlist gibt es erst ab macOS 27 auf betreuten Geräten [Q111].

### 8.4 Virtuelle Umgebungen

`python3 -m venv ~/Projekte/venv` erzeugt Symlinks auf den Framework-Interpreter [Q133]. Firewall nach Code-Identität: gilt weiter (8.1). Gatekeeper: nicht betroffen. Prüfschritt P6:

```bash
~/Projekte/venv/bin/python -c "import sys; print(sys.prefix, sys.base_prefix)"
~/Projekte/venv/bin/python -c "import urllib.request; urllib.request.urlopen('https://example.org')"   # erwartet: URLError
```

### 8.5 Eingebettete Web-Engines

`pywebview` nutzt unter macOS WebKit; `PyQt6-WebEngine`, JavaFX WebView bringen eigene Engines. Ohne Netz nicht installierbar (8.2); kommen sie über den Wheel-Ordner, verweigert Little Snitch ihren Prozessen das Netz (keine Regel). Sie zeigen nur lokale Inhalte. Prüfschritt bei Installation: Testfenster mit `https://example.org` bleibt leer; Prozessname des WebKit-Helfers im Network Monitor notieren.

### 8.6 Benutzerprofil

Du kannst Firefox-Profil, Kate-Konfiguration (`~/Library/Preferences/katerc`, `~/Library/Preferences/kate/lspclient/settings.json`) und `~/Library/LaunchAgents` ändern. Nichts davon ist sicherheitsrelevant: Richtlinien in `/Library/Preferences`, Erweiterungen erzwungen, Little Snitch und Cold Turkey mit Admin- bzw. Passwortschutz, Profil nur mit Admin entfernbar. Ein LaunchAgent startet nur Programme ohne Netz.

### 8.7 Neubewertung

Nicht gesperrt werden Terminal, Shells, Python, Java, Git, Homebrew-Installation, Aktivitätsanzeige, Systemeinstellungen: alles ohne Ausbruchswirkung unter 7.1, 7.4 und 8.1. Gesperrt: Safari, App Store, Benutzerordner für Apps, Ablenkungs-Apps nach Wahl.

## 9. Editor: Kate mit Python und Java

### 9.1 Rolle im Lockdown

Kate ist der Editor; Python und Java führen Code aus. Kate darf Befehle starten und Dateien über Netzprotokolle öffnen; das ist abgesichert, weil kein Prozess außer Firefox, Cold Turkey und Apple-Diensten Netz bekommt (8.1) und heruntergeladene Programme am Gatekeeper-Profil scheitern (7.4). Was bleibt: Du kannst lokal alles lesen, schreiben, bauen und ausführen; nichts holt neue Inhalte aus dem Internet. Das Debugger-Plugin von Kate ist auf macOS verfügbar und bringt ein Profil für `debugpy` mit [Q156]; Java wird mit `jdb` debuggt.

### 9.2 Kate installieren

kate-editor.org nennt für macOS nur Nightly-Installer von der KDE-CI, „nur zum Testen“ [Q65]. Auf dem KDE-CDN gibt es Release-Branch-Builds: `kate-release_26.08-12761-macos-clang-arm64.dmg` (Apple-Chip) unter `https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/macos-arm64/`, ohne Prüfsummendatei [Q145]; für Intel der Ordner `macos-x86_64` (Prüfschritt: vorhanden?). Ob das DMG signiert und notarisiert ist, ist für Kate nicht belegt; der Notarisierungswunsch ist offen [Q149], KDEs CI kann notarisieren [Q136]. Eine stabile Release-Version aus dem App Store gibt es nicht.

1. Als Admin DMG laden, `shasum -a 256` notieren (für spätere Vergleiche; es gibt keinen Sollwert), DMG öffnen, **Kate** nach **Programme** ziehen [Q148].
2. Erster Start als Admin. macOS 14: Bei Gatekeeper-Meldung Ctrl-Klick > **Öffnen**. macOS 15: Meldung schließen, **Systemeinstellungen** > **Datenschutz & Sicherheit** > **Dennoch öffnen** > **Öffnen** [Q102] (Prüfschritt P31). Danach ist die App freigegeben und ohne Quarantäne-Attribut; dein Konto kann sie nicht ersetzen.
3. Prüfschritt: **Kate** > **Über Kate** zeigt 26.08.x.

Alternative mit Stabilitätsvorteil (nicht geprüft, Prüfschritt): Installation über MacPorts oder Homebrew durch den Admin im Wartungsfenster.

### 9.3 Python installieren

1. Als Admin `python-3.15.2-macos11.pkg` von `https://www.python.org/downloads/release/python-3152/` laden (signiert und notarisiert mit PSF-Developer-ID, universal2 [Q132][Q150]); Installation verlangt Adminrechte; Ziel `/Library/Frameworks/Python.framework/Versions/3.15`, Symlinks `/usr/local/bin/python3` [Q132].
2. Language Server und Debugger systemweit (Netz noch offen oder Wartungsregel an):
   ```bash
   sudo /usr/local/bin/python3 -m pip install "python-lsp-server[all]" debugpy
   ```
   `python-lsp-server` 1.15.0 ohne Node.js [Q140][Q151]; `debugpy` ohne Abhängigkeiten [Q155]. Nicht gewählt: `pyright` (lädt Node.js nach [Q152]), `basedpyright` (Node.js als Wheel [Q153]), `jedi-language-server` (weniger Plugins [Q154]).
3. Prüfschritt: `which python3 pylsp` zeigt `/usr/local/bin/python3` und `/Library/Frameworks/Python.framework/Versions/3.15/bin/pylsp`; falls `pylsp` nicht im Pfad liegt, in Kate den vollen Pfad eintragen (9.5).

### 9.4 Java installieren

JDT LS 1.63.0 verlangt Java 21 oder neuer, die Entwicklungsversion seit 30. Oktober 2026 Java 25 [Q69][Q158]. Gewählt: Temurin JDK 25.0.5+8.

1. Als Admin `OpenJDK25U-jdk_aarch64_mac_hotspot_25.0.5_8.pkg` (Apple-Chip) oder `OpenJDK25U-jdk_x64_mac_hotspot_25.0.5_8.pkg` (Intel) von `https://adoptium.net/temurin/releases/` laden [Q159]; Installation nach `/Library/Java/JavaVirtualMachines/temurin-25.jdk` [Q161]:
   ```bash
   sudo installer -pkg OpenJDK25U-jdk_*_mac_hotspot_25.0.5_8.pkg -target /
   /usr/libexec/java_home -v 25
   ```
2. JDT LS: Tarball und `.sha256` von `https://download.eclipse.org/jdtls/milestones/1.63.0/` [Q157]; prüfen und entpacken:
   ```bash
   shasum -a 256 -c jdt-language-server-1.63.0-202610191322.tar.gz.sha256
   sudo mkdir -p /Library/jdtls
   sudo tar -xzf jdt-language-server-1.63.0-202610191322.tar.gz -C /Library/jdtls
   ls /Library/jdtls/plugins/org.eclipse.equinox.launcher_*.jar
   ```
   Launcher-Dateinamen notieren (`<LAUNCHER>`).
3. Konfigurationskopie in deinem Konto: `mkdir -p ~/Library/Caches/jdtls && cp -R /Library/jdtls/config_mac ~/Library/Caches/jdtls/config_mac` (JDT LS schreibt in den `-configuration`-Ordner; der mitgelieferte bleibt unverändert [Q69][Q163]).

### 9.5 Code-Vervollständigung

Belegt sind die englischen Bezeichnungen („Settings“ > „Configure Kate…“ > „Application“ > „Plugins“, Seite „LSP Client“, „User Server Settings“) [Q166][Q178]; die deutsche Übersetzung ist am Bildschirm zu prüfen.

1. **Einstellungen** > **Kate einrichten…** > **Anwendung** > **Erweiterungen**: **LSP-Client** einschalten. Standardoptionen lassen (**Semantische Hervorhebung**, **Programmdiagnosen anzeigen**, **inkrementell synchronisieren**). Keine KI-Vervollständigung in Kate 26.08 [Q174].
2. **Benutzereinstellungen des Servers** (Datei `~/Library/Preferences/kate/lspclient/settings.json` [Q167]); Kates Standardliste nennt `pylsp` und `jdtls` [Q164]; `jdtls` ist ein Python-Skript [Q163], deshalb der `java`-Aufruf direkt. `<LAUNCHER>` und `<DEIN-KONTO>` ersetzen; `<JDK>` ist die Ausgabe von `/usr/libexec/java_home -v 25`:

```json
{
    "servers": {
        "python": {
            "command": ["/Library/Frameworks/Python.framework/Versions/3.15/bin/pylsp", "--check-parent-process"],
            "url": "https://github.com/python-lsp/python-lsp-server",
            "highlightingModeRegex": "^Python$"
        },
        "java": {
            "command": [
                "<JDK>/bin/java",
                "-Declipse.application=org.eclipse.jdt.ls.core.id1",
                "-Dosgi.bundles.defaultStartLevel=4",
                "-Declipse.product=org.eclipse.jdt.ls.core.product",
                "-Dlog.level=ALL",
                "-Xmx1G",
                "--add-modules=ALL-SYSTEM",
                "--add-opens", "java.base/java.util=ALL-UNNAMED",
                "--add-opens", "java.base/java.lang=ALL-UNNAMED",
                "-jar", "/Library/jdtls/plugins/<LAUNCHER>",
                "-configuration", "/Users/<DEIN-KONTO>/Library/Caches/jdtls/config_mac",
                "-data", "/Users/<DEIN-KONTO>/Library/Caches/jdtls/workspace"
            ],
            "url": "https://github.com/eclipse-jdtls/eclipse.jdt.ls",
            "highlightingModeRegex": "^Java$"
        }
    }
}
```

   Startbefehl nach der JDT-LS-README [Q69].
3. Beim ersten Öffnen einer Python- oder Java-Datei fragt Kate, ob der Server starten darf; **Ja** [Q167].
4. Prüfschritt P20: wie in 10.2.

### 9.6 Plugins

Freigegeben: LSP-Client, Erstellen (Build), Externe Werkzeuge, Terminal (Konsole-Teil; im macOS-Paket gebündelt, Lauffähigkeit Prüfschritt P19 [Q169]), Projekte (Git), Git-Blame, Dokumentvorschau (`markdownpart` ohne Web-Engine [Q172][Q173]), Debugger (DAP, `debugpy`-Profil [Q156]), Suchen und Ersetzen, Dateisystem-Browser, Symbolansicht, Snippets, Textfilter, SQL. Kein Plugin öffnet einen Weg, den 7.4 und 8.1 nicht schließen. Links aus Hilfetexten öffnen in Firefox ESR (Standardbrowser, 3.6). **Datei** > **Öffnen** mit `http://`-Adresse: der KIO-Worker hat kein Netz (Prüfschritt P18: Prozess `kioworker` in der Aktivitätsanzeige).

### 9.7 Einstellungen sperren

Nicht nötig (Begründung wie 8.6). Option mit Prüfschritt: KConfig liest systemweite Dateien unter `/Library/Preferences` (Qt-Standardpfade [Q168]) und wertet `[$i]` plattformneutral aus (KConfig-Quelltext [Q175][Q176]); `/Library/Preferences/kdeglobals` mit `[KDE Action Restrictions][$i]` und `shell_access=false` würde Terminal-, External-Tools- und Projekt-Terminal sperren, nicht Build und Textfilter [Q177]. Beleg aus der KDE-Doku für macOS fehlt.

### 9.8 Cold Turkey

Nicht sperren: Kate, Terminal, `python3`, `pylsp`, `java`, `git` (6.4).

### 9.9 Git

Als Admin `xcode-select --install` (Command Line Tools, enthält `git`) [Q183]. Ohne Netz: `git` lokal (E16); `git clone` von Remotes scheitert. Kate-Projekt-Plugin nutzt `git` aus dem Pfad [Q184].

### 9.10 Prüfschritte

10.1 Zeilen K1 bis K9; 10.2.

## 10. Ausbruchsprüfung und Nutzbarkeitstest

Vor dem Scharfschalten: Cold-Turkey-Blöcke an, ungesperrt; Little Snitch in Silent Deny; Profil installiert; FileVault an; Intel: Firmware-Passwort gesetzt. Alle Zeilen in deinem Konto. Scharfschalten nur, wenn jede Zeile „bestanden“ oder „Restrisiko benannt“ trägt.

### 10.1 Protokoll

| Nr. | Weg | Prüfschritt | Erwartet | Tatsächlich |
|---|---|---|---|---|
| F1 | Profil-Flyout, `about:profiles` | Adressen eingeben; Avatar-Schaltfläche | Sperrseite; keine Profilverwaltung | |
| F2 | `firefox -P` | Terminal: `/Applications/Firefox.app/Contents/MacOS/firefox -P` | Profilmanager; neues Profil hat dieselben Sperren (F3 bis F7 wiederholen) | |
| F3 | `about:config` | eingeben | Sperrseite | |
| F4 | `about:support` | eingeben | Sperrseite | |
| F5 | `about:addons`, Erweiterungsmenü | eingeben; Menü | Sperrseite; kein Entfernen | |
| F6 | Fehlerbehebungsmodus | **Hilfe**-Menü; Terminal: `.../firefox -safe-mode`; Wahltaste beim Start | kein Menüpunkt; Start mit `-safe-mode` gelingt, Erweiterungen aus, Cold Turkey schließt Firefox nach bis zu 60 Sekunden, bis dahin zeigt `WebsiteFilter` die Sperrseite für nicht freigegebene Seiten (Restrisiko F6: bis zu 60 Sekunden ohne uBlock) | |
| F7 | Privates Fenster | **Befehl+Umschalt+P** | nichts | |
| F8 | Entwicklerwerkzeuge | **Wahl+Befehl+I**, `about:debugging` | nichts; Sperrseite | |
| F9 | `view-source:` | freigegebene und gesperrte Seite | Quelltext nur der freigegebenen | |
| F11 | `file://` | `file:///etc/hosts` | angezeigt (E15) | |
| F12 | `data:` | `data:text/html,<a href="https://example.org">x</a>`, klicken | Sperrseite | |
| F14 | Download | ZIP von freigegebener Seite | in `~/Downloads`, kein Dialog | |
| F16 | Erweiterung aus Datei | XPI ins Fenster ziehen | „vom Administrator blockiert“ | |
| F17 | Proxy | `about:preferences` > Netzwerk | ausgegraut | |
| F20 | Firefox-Update | **Firefox** > **Über Firefox** | kein Hinweis (`ManualAppUpdateOnly`); Update im Wartungsfenster | |
| F21 | Zweiter Firefox | Firefox Nightly DMG mit Firefox laden, nach `~/Applications` ziehen, starten | Gatekeeper-Profil verweigert (quarantäniert, nicht App Store); falls doch gestartet: Cold Turkey beendet die App (Ordnersperre), Restrisiko wie 6.6 | |
| F22 | `about:preferences`, `about:policies` | eingeben | erreichbar, nichts Sicherheitsrelevantes änderbar | |
| F23 | Benutzerrichtlinie | `defaults write ~/Library/Preferences/org.mozilla.firefox BlockAboutConfig -bool FALSE`, Firefox neu starten | `about:config` bleibt gesperrt; danach `defaults delete ~/Library/Preferences/org.mozilla.firefox BlockAboutConfig` | |
| B1 | Adminrechte | `sudo -v`; Schloss in **Datenschutz & Sicherheit** | abgelehnt; Admin-Abfrage | |
| B2 | Safari | Safari starten; Link aus Mail öffnen | Cold Turkey beendet Safari; Links öffnen in Firefox | |
| B3 | Browser in `~/Applications` | wie F21 mit einem Chromium-Browser | verweigert oder beendet | |
| B4 | WebKit-Ansichten | **Hilfe** > **macOS-Hilfe** > Link anklicken; Mail mit HTML; Notizen mit Link | Hilfe zeigt nur lokale Seiten; Netz verweigert (Network Monitor) | |
| B5 | Programm vom USB-Stick | portable App vom Stick nach `~/Documents` kopieren, starten | läuft ohne Netz oder Cold Turkey beendet sie (Ordnersperre); Restrisiko B5 benannt | |
| B7 | Systemzeit | **Allgemein** > **Datum & Uhrzeit** (14 und 15 gleich), Automatik aus | Schloss verlangt Admin (P30); Cold Turkey schließt die Einstellungen | |
| B8 | Neues Konto, Gast | **Benutzer:innen & Gruppen** | Schloss verlangt Admin; Gast aus | |
| B9 | LaunchAgent | `~/Library/LaunchAgents/test.plist`, das `curl https://example.org` startet | läuft, scheitert am Netz | |
| B10 | Homebrew, App Store, pip | Homebrew-Installer; App Store-Installation; `pip install requests` | Download scheitert; Admin-Abfrage; Verbindungsfehler | |
| B11 | Terminal-Downloads | `curl -O https://example.org/index.html` | scheitert | |
| B12 | `defaults` in `/Library` | `sudo defaults write /Library/Preferences/org.mozilla.firefox BlockAboutConfig -bool FALSE` | `sudo` verweigert | |
| B13 | Sicherer Modus (P24) | Neustart im sicheren Modus (Admin dabei) | Cold Turkey läuft (Aktivitätsanzeige) und Little Snitch filtert; sonst Restrisiko B13: Safari dort offen, Intel: Firmware-Passwort verhindert den Start | |
| B14 | Wiederherstellung (P23) | Apple-Chip: Einschalttaste halten > **Optionen**; Intel: **Befehl-R** | Intel: Firmware-Passwort; Apple-Chip: Ergebnis dokumentieren (zugelassener Benutzer? Terminal? Volume entsperrbar?) | |
| B15 | Bildschirmzeit (P25) | Code-Einstellung ändern mit deinem Kennwort | Ergebnis dokumentieren | |
| I1 | Python, Java mit Netz | `python3 -c "... urlopen ..."`; `jshell` mit `new java.net.URL("https://example.org").openStream()` | scheitern | |
| I2 | Lokaler Webserver | `python3 -m http.server 8000 --bind 127.0.0.1`; Firefox `http://localhost:8000/` (P4) | erreichbar | |
| I4 | venv | 8.4 | kein Netz | |
| I5 | Kopie von Safari | `cp -R /Applications/Safari.app ~/Applications/S.app`, starten | SIP verhindert das Kopieren nicht; Cold Turkey beendet die Kopie (folgt der App) oder Ordnersperre; Netz verweigert (gleiche Code-Identität) | |
| I6 | Kindprozess | `python3 -c "import subprocess; subprocess.run(['open','-a','Safari'])"` | Safari wird beendet | |
| I8 | Programm mit Python schreiben | kleines C- oder Go-Binary ohne Netz ist nicht erzeugbar ohne Compiler; stattdessen `cp /bin/ls ~/Documents/x && ~/Documents/x` | läuft (kein Quarantäne-Attribut); Restrisiko I8 benannt: läuft ohne Netz | |
| I9 | Autostart | **Anmeldeobjekte** in **Allgemein** > Skript mit `curl` | läuft, scheitert am Netz | |
| T1 | Block ausschalten | Toggle | nach Sperre Passwortabfrage | |
| T2 | Cold Turkey deinstallieren | App in den Papierkorb | verweigert (`/Applications` nicht beschreibbar) | |
| T3 | Prozesse beenden | Aktivitätsanzeige > Cold Turkey beenden | nicht möglich oder Block läuft weiter (Hersteller: Block wirkt auch nach Prozessende [Q31]) | |
| T4 | Erweiterung beenden | Firefox **Weitere Werkzeuge** > **Task-Manager** | Firefox wird geschlossen | |
| T6 | Neuer Tab (P2) | **Befehl+T** | bleibt offen | |
| T7 | Umbenennen | `mv ~/Applications/S.app ~/Applications/T.app` | Block folgt der App | |
| K1 | Kate Build | Ziel `curl https://example.org` | scheitert | |
| K2 | Kate Terminal | `curl https://example.org` | scheitert | |
| K3 | Kate öffnet Adresse (P18) | **Datei** > **Öffnen**, `https://example.org/index.html` | Fehler; `kioworker` in der Aktivitätsanzeige | |
| K4 | Vorschau | Markdown mit `<iframe>` | kein Netzinhalt | |
| K5 | External Tools | `open https://example.org` | Firefox Sperrseite | |
| K7 | Portable Kate | zweites Kate-DMG nach `~/Applications` | Gatekeeper-Profil oder Ordnersperre | |
| K8 | LSP fremder Server | `"command": ["/Users/<DEIN-KONTO>/x"]` | Kate fragt; Programm ohne Netz | |
| T0 | Cold-Turkey-Berechtigungen | nach 6.1 | notiert | |

### 10.2 Nutzbarkeitstest

| Aufgabe | Schritte | Erwartet | Tatsächlich |
|---|---|---|---|
| Python schreiben, ausführen, debuggen | `~/Projekte/hallo.py`; Vervollständigung, Diagnose, Sprung (9.5); Terminal `python3 hallo.py`; Debugger-Plugin mit Profil `debugpy`, Haltepunkt setzen (P34) oder `breakpoint()` mit `pdb` | funktioniert | |
| venv und Wheel | venv anlegen, Paket aus `/Library/Wheels` installieren | funktioniert | |
| Java schreiben, ausführen, debuggen | `~/Projekte/java/Hallo/Hallo.java`; Vervollständigung, Diagnose, Sprung; `javac`, `java`, `jdb` | funktioniert | |
| Freigegebene Seiten, Dokumentation | jede Zeile aus 4.1; PDF | lädt | |
| Git | `git init`, `commit`; Projekt-Plugin | funktioniert | |
| Alltag | Drucken, Bluetooth, Lautstärke, Helligkeit, Vorschau, Textedit | funktioniert | |
| Softwareupdate | als Admin **Allgemein** > **Softwareupdate** | findet Updates | |

### 10.3 Scharfschalten

6.7 (Passwörter), 8.1 (Silent Deny, Sicherheitseinstellungen gesperrt), 7.4 (Profil), 7.2 (FileVault), 7.6 (Intel: Firmware-Passwort). Danach T1 und T2 wiederholen.

## 11. Betrieb

### 11.1 Updates

| Was | Wie | Wer | Wann |
|---|---|---|---|
| macOS | **Allgemein** > **Softwareupdate**, automatische Updates an | Admin | laufend; nach jedem Update 11.4 |
| Firefox ESR | **Firefox** > **Über Firefox** im Adminkonto (die App gehört dem Admin [Q96]); ESR-Punktversionen alle vier Wochen [Q1] | Admin | Wartungsfenster alle vier Wochen |
| Cold Turkey | Block entsperren, neues PKG, Block sperren | Admin | bei neuer Version |
| Little Snitch | eigene Allow-Regel; Update über die App | Admin | bei neuer Version |
| Kate | neues Release-DMG aus `release-26.xx` nach **Programme** (Gatekeeper-Freigabe wie 9.2) | Admin | bei neuer KDE-Gear-Version |
| Python, pylsp, debugpy | neues PKG; `sudo python3 -m pip install --upgrade ...` mit Wartungsregel | Admin | bei neuer Version; danach 9.5 Pfade prüfen |
| JDK, JDT LS | neues PKG; neues Tarball nach `/Library/jdtls`, `<LAUNCHER>` in `settings.json` | Admin | halbjährlich |

### 11.2 Wartungsfenster

Termin: Der Admin meldet sich an, aktiviert in Little Snitch die Wartungsregeln (`python3`, `git`, `curl`), arbeitet die Wunschliste ab (`pip download`), deaktiviert die Regeln, prüft **Silent Deny**, meldet sich ab.

### 11.3 Nach einem Python-Update

Pfade in `settings.json` (`Versions/3.15` → neue Version) und die Wartungsregel in Little Snitch (Code-Identität bleibt, Pfadregeln prüfen).

### 11.4 Nach einem macOS-Update

Prüfen: Profil noch installiert (14: **Datenschutz & Sicherheit** > **Profile**; 15: **Allgemein** > **Geräteverwaltung**); Little Snitch aktiv und im Silent-Deny-Modus; FileVault an; Firmware-Passwort (Intel); `about:policies` ohne Fehler; Zeilen B2, B3, I1 aus 10.1. Nach einem Wechsel auf macOS 16 oder neuer: Menüpfade dieser Anleitung erneut prüfen.

## 12. Rückbau

Nur der Admin:

1. Cold Turkey: Blöcke entsperren und ausschalten; **Password Settings** aus; App in den Papierkorb.
2. Little Snitch: Sicherheitseinstellungen entsperren, Alert Mode oder Deinstallation (App in den Papierkorb, Systemerweiterung wird entfernt [Q119]).
3. Profil entfernen: 14: **Datenschutz & Sicherheit** > **Profile** > Profil > **Entfernen**; 15: **Allgemein** > **Geräteverwaltung** > **Entfernen** [Q92].
4. Firefox: `sudo defaults delete /Library/Preferences/org.mozilla.firefox` und `sudo rm /Library/Preferences/org.mozilla.firefox.plist`.
5. Bildschirmzeit: Code entfernen, **Inhalt & Datenschutz** aus.
6. Konten: dein Konto wieder Admin (**Benutzer:in darf diesen Computer verwalten**), falls gewünscht.
7. FileVault bleibt; Firmware-Passwort (Intel) bei Bedarf im Startsicherheitsdienstprogramm ausschalten.

Nötig: Adminkennwort, Cold-Turkey-Passwörter, Bildschirmzeit-Code, Firmware-Passwort.

## Anhang A: Santa im Lockdown-Modus (Option, nach Prüfschritt P26)

Nur wenn die Ordnersperren in 6.4 ein neu angelegtes Programm nicht beenden oder du Datenträger-Programme (B5) schließen willst. Santa (northpolesec) autorisiert Mach-O-Binaries nach Regeln (CDHASH, BINARY, SIGNINGID, CERTIFICATE, TEAMID) und blockiert im Lockdown-Modus alles Unbekannte; Skripte prüft es nicht [Q107][Q108]. Konfiguration ist als MDM-Profil `com.northpolesec.santa` dokumentiert; ob ein manuell installiertes Profil ohne MDM wirkt, ist nicht belegt (Prüfschritt P32) [Q109].

1. Profil `com.northpolesec.santa` mit `ClientMode` 1 (Monitor), `StaticRules` für Apple, Firefox, Cold Turkey, Little Snitch, Kate, Python, Java, Git (Team-IDs über `codesign -dv --verbose=4 <App>` ermitteln), `PayloadScope` System, installieren wie 7.4 [Q109].
2. Paket (signiert und notarisiert) als Admin: `sudo installer -pkg santa-<version>.pkg -target /`; Systemerweiterung unter **Anmeldeobjekte & Erweiterungen** freigeben; Vollzugriff auf die Festplatte für `com.northpolesec.santa.daemon` [Q110].
3. Eine Woche Monitor-Modus; `santactl status` und das Protokoll auf Blockierungen prüfen, Regeln ergänzen.
4. `ClientMode` 2 (Lockdown) im Profil setzen, Profil neu installieren; `santactl rule` ist durch `StaticRules` gesperrt [Q109].
5. Prüfschritt: Zeilen B5, I8, K7 aus 10.1 wiederholen; erwartet: blockiert.

Was Santa nicht schließt: Python- und Java-Skripte (gewollt) und Interpreter-Argumente; Begründung des Herstellers [Q108].

## 13. Quellen

Alle Quellen am 31.10.2026 abgerufen. Quelltextangaben beziehen sich auf den GitHub-Spiegel mozilla-firefox/firefox (Branch esr153) und die GitHub-Spiegel der KDE-Repositories (master und release/26.08). Seiten unter firefox-admin-docs.mozilla.org leiten auf mozilla.github.io/enterprise-admin-reference um. Apple-Handbuchseiten mit /14.0/ und /15.0/ im Pfad sind die versionsgebundenen Seiten für macOS 14 und 15.

- Q1 https://whattrainisitnow.com/release/?version=esr
- Q4 https://mozilla.github.io/enterprise-admin-reference/reference/policies/
- Q5 https://mozilla.github.io/enterprise-admin-reference/reference/policies/preferences/
- Q6 https://mozilla.github.io/enterprise-admin-reference/reference/policies/disablesafemode/
- Q7 https://mozilla.github.io/enterprise-admin-reference/reference/policies/privatebrowsingmodeavailability/
- Q8 https://mozilla.github.io/enterprise-admin-reference/reference/policies/websitefilter/
- Q9 https://mozilla.github.io/enterprise-admin-reference/reference/policies/extensionsettings/
- Q10 https://mozilla.github.io/enterprise-admin-reference/reference/policies/blockaboutprofiles/
- Q11 https://mozilla.github.io/enterprise-admin-reference/reference/policies/proxy/
- Q12 https://mozilla.github.io/enterprise-admin-reference/reference/policies/disablemacosuserpolicies/
- Q13 https://bugzilla.mozilla.org/show_bug.cgi?id=2021356
- Q14 https://github.com/mozilla/policy-templates/blob/master/mac/README.md
- Q16 https://getcoldturkey.com/support/user-guide/
- Q18 https://getcoldturkey.com/support/system-requirements/
- Q25 https://docs.python.org/3/library/venv.html
- Q27 https://support.apple.com/guide/security/gatekeeper-and-runtime-protection-sec5599b66df/web
- Q28 https://support.apple.com/guide/mac-help/configuration-profiles-standardize-settings-mh35561/mac
- Q30 https://getcoldturkey.com/news/ (Release Notes)
- Q31 https://getcoldturkey.com/features/
- Q32 https://getcoldturkey.com/pricing/
- Q34 https://getcoldturkey.com/support/extensions/firefox/
- Q36 https://getcoldturkey.com/support/extensions/mac/safari/full-disk-access
- Q37 https://developer.apple.com/news/?id=saqachfa
- Q65 https://kate-editor.org/get-it/
- Q69 https://github.com/eclipse-jdtls/eclipse.jdt.ls (README; Stand Tag v1.63.0: https://raw.githubusercontent.com/eclipse-jdtls/eclipse.jdt.ls/v1.63.0/README.md)
- Q71 https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/master/CHANGELOG.md
- Q84 Firefox-Quelltext esr153: browser/components/profiles/SelectableProfileService.sys.mjs
- Q86 Firefox-Quelltext esr153: toolkit/components/enterprisepolicies/EnterprisePoliciesParent.sys.mjs
- Q87 https://support.mozilla.org/en-US/kb/diagnose-firefox-issues-using-troubleshoot-mode
- Q88 Firefox-Quelltext esr153: toolkit/xre/SafeMode.h und toolkit/xre/PolicyChecks.h
- Q89 https://support.apple.com/de-de/guide/mac-help/mtusr001/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mtusr001/15.0/mac/15.0
- Q90 https://support.apple.com/de-de/guide/mac-help/mchl3e281fc9/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mchl3e281fc9/15.0/mac/15.0
- Q91 https://support.apple.com/guide/deployment/profiles-and-payloads-depc0aadd3fe/web
- Q92 https://support.apple.com/de-de/guide/mac-help/mh35561/14.0/mac/14.0, https://support.apple.com/de-de/guide/mac-help/mh35474/14.0/mac/14.0, https://support.apple.com/de-de/guide/mac-help/mh35561/15.0/mac/15.0, https://support.apple.com/de-de/guide/mac-help/mh35474/15.0/mac/15.0
- Q93 https://mozilla.github.io/enterprise-admin-reference/reference/policies/enterprisepoliciesenabled/ und https://support.mozilla.org/en-US/kb/managing-policies-macos-desktops
- Q94 https://bugzilla.mozilla.org/show_bug.cgi?id=1792595
- Q95 https://support.mozilla.org/en-US/questions/1298717 (Community)
- Q96 https://bugzilla.mozilla.org/show_bug.cgi?id=1893686
- Q97 https://www.firefox.com/en-US/download/all/desktop-esr/
- Q98 https://support.mozilla.org/en-US/questions/1197118 (Community)
- Q99 https://support.mozilla.org/en-US/questions/1415779 (Community)
- Q100 Firefox-Quelltext esr153: xpcom/base/nsMacPreferencesReader.mm, browser/moz.configure, browser/branding/nightly/configure.sh
- Q102 https://support.apple.com/de-de/102445
- Q103 https://developer.apple.com/documentation/devicemanagement/systempolicycontrol.md
- Q104 https://developer.apple.com/documentation/bundleresources/information-property-list/lsfilequarantineenabled.md
- Q105 https://developer.apple.com/forums/thread/666452 und https://developer.apple.com/forums/thread/765139 (Apple-Entwicklerforum, Antworten von Apple DTS)
- Q107 https://github.com/northpolesec/santa
- Q108 https://northpole.dev/features/binary-authorization/ und https://northpole.dev/limitations/
- Q109 https://northpole.dev/deployment/profile-configuration/ und https://northpole.dev/configuration/keys/
- Q110 https://northpole.dev/deployment/install-package/, https://northpole.dev/deployment/profile-system-extension/, https://northpole.dev/deployment/profile-tcc/
- Q111 https://support.apple.com/guide/deployment/dep001044b08/web
- Q112 https://support.apple.com/de-de/guide/mac-help/mh34041/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mh34041/15.0/mac/15.0
- Q113 https://support.apple.com/guide/security/firewall-security-in-macos-seca0e83763f/web
- Q114 https://objective-see.org/products/lulu.html und https://github.com/objective-see/LuLu
- Q115 https://help.obdev.at/littlesnitch6/adv-code-identity
- Q116 https://help.obdev.at/littlesnitch6/pref-security
- Q117 https://support.apple.com/guide/deployment/system-and-kernel-extensions-in-macos-depa5fb8376f/web und https://developer.apple.com/documentation/devicemanagement/systemextensions.md
- Q118 https://help.obdev.at/littlesnitch6/intro-install
- Q119 https://help.obdev.at/littlesnitch6/intro-uninstall
- Q120 https://support.apple.com/de-de/guide/mac-help/mchl336af525/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mchlbcf0dfe2/14.0/mac/14.0
- Q121 https://support.apple.com/de-de/guide/mac-help/mchl336af525/15.0/mac/15.0 und https://support.apple.com/de-de/guide/mac-help/mchlbcf0dfe2/15.0/mac/15.0
- Q122 https://support.apple.com/de-de/guide/mac-help/mchl3a19a9e7/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mchl3a19a9e7/15.0/mac/15.0
- Q123 https://support.apple.com/de-de/102149
- Q124 https://support.apple.com/de-de/102362
- Q126 https://support.apple.com/en-us/102518
- Q127 https://support.apple.com/guide/deployment/dep5810e849c/web
- Q129 https://support.apple.com/en-kg/HT201262 und https://support.apple.com/de-de/guide/mac-help/mh21245/14.0/mac/14.0
- Q130 https://docs.brew.sh/Installation
- Q131 https://raw.githubusercontent.com/Homebrew/install/master/install.sh
- Q132 https://docs.python.org/3/using/mac.html
- Q133 https://github.com/python/cpython/blob/main/Lib/venv/__init__.py
- Q136 https://bugs.kde.org/show_bug.cgi?id=428062 und https://mail.kde.org/pipermail/kde-mac/2024-August/010160.html
- Q137 Firefox-Quelltext esr153: devtools/startup/DevToolsStartup.sys.mjs
- Q140 https://github.com/python-lsp/python-lsp-server
- Q145 https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/macos-arm64/
- Q148 Craft-Quelltext: bin/Packager/MacDMGPackager.py und bin/Utils/CodeSign.py
- Q149 https://bugs.kde.org/show_bug.cgi?id=470738
- Q150 https://www.python.org/downloads/ und https://www.python.org/downloads/release/python-3152/
- Q151 https://pypi.org/project/python-lsp-server/
- Q152 https://pypi.org/project/pyright/
- Q153 https://pypi.org/project/basedpyright/ und https://docs.basedpyright.com/latest/installation/command-line-and-language-server/
- Q154 https://pypi.org/project/jedi-language-server/
- Q155 https://pypi.org/project/debugpy/
- Q156 Kate-Quelltext: addons/gdbplugin/CMakeLists.txt und addons/gdbplugin/dap/servers.json
- Q157 https://download.eclipse.org/jdtls/milestones/1.63.0/
- Q158 https://raw.githubusercontent.com/eclipse-jdtls/eclipse.jdt.ls/master/README.md (Commit 217c210 vom 30.10.2026)
- Q159 https://api.adoptium.net/v3/info/available_releases und https://adoptium.net/temurin/releases/
- Q161 https://adoptium.net/installation/macOS/
- Q163 JDT-LS-Quelltext: org.eclipse.jdt.ls.product/scripts/jdtls und jdtls.py
- Q164 Kate-Quelltext: addons/lspclient/settings.json (Branch release/26.08)
- Q165 https://docs.gradle.org/current/userguide/gradle_wrapper.html
- Q166 https://docs.kde.org/stable5/en/kate/kate/plugins.html
- Q167 Kate-Quelltext: apps/lib/katepluginmanager.cpp, addons/lspclient/lspclientplugin.cpp, apps/kate/main.cpp
- Q168 https://doc.qt.io/qt-6/qstandardpaths.html
- Q169 Kate-Quelltext: addons/konsole/kateconsole.cpp; Konsole-Quelltext: src/CMakeLists.txt
- Q172 Kate-Quelltext: addons/preview/previewwidget.cpp
- Q173 https://github.com/KDE/markdownpart
- Q174 Kate-Quelltext: addons/CMakeLists.txt (master und release/26.08)
- Q175 https://github.com/KDE/kconfig/blob/master/docs/options.md
- Q176 KConfig-Quelltext: src/core/kconfig.cpp; https://develop.kde.org/docs/administration/kiosk/introduction/ und https://develop.kde.org/docs/administration/kiosk/keys/
- Q177 Kate-Quelltext: addons/externaltools/externaltoolsplugin.cpp, addons/project/kateprojecttreeviewcontextmenu.cpp
- Q178 Kate-Quelltext: addons/lspclient/lspconfigwidget.ui
- Q183 https://git-scm.com/install/mac
- Q184 Kate-Quelltext: apps/lib/gitprocess.cpp
- Q197 https://support.apple.com/de-de/guide/mac-help/mh11785/14.0/mac/14.0 und https://support.apple.com/de-de/guide/mac-help/mh11785/15.0/mac/15.0
- Q198 https://developer.apple.com/documentation/devicemanagement/appstore.md (Apple, Payload com.apple.appstore) und https://mosen.github.io/profiledocs/payloads/mac/appstore.html (MacAdmins-Community, Schlüssel restrict-store-require-admin-to-install)
- Q199 https://help.obdev.at/littlesnitch6/concepts-opmodes
- Q200 https://help.obdev.at/littlesnitch6/concepts-rules
- Q201 https://www.unix.com/man-page/osx/1/defaults (ältere Fassung der Manpage) und https://gist.github.com/inoperable/6041a6cc9426043bb830f4f4ed658449 (Nutzung von defaults import/export, Community)
- Q202 https://support.apple.com/de-de/102384
- Q203 https://developer.apple.com/documentation/devicemanagement/toplevel.md
