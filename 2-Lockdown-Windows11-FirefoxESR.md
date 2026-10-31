# Dokument 2: Lockdown Windows 11 Pro mit Firefox ESR, Cold Turkey Pro und Kate

Stand: 31. Oktober 2026. Geprüfte Versionen: Windows 11 Pro, Firefox ESR 153.6.0, Cold Turkey Blocker 4.9(2), Kate 26.08.2 (Microsoft Store), Python 3.15.2, Eclipse Temurin JDK 25.0.5, Eclipse JDT Language Server 1.63.0, python-lsp-server 1.15.0, Git for Windows 2.56.0(4). Quellen mit Abrufdatum stehen in Abschnitt 13; Verweise in eckigen Klammern beziehen sich darauf.

## 1. Für wen, wofür, Voraussetzungen

Diese Anleitung richtet dir einen Windows-11-Pro-Rechner so ein, dass du darauf mit Firefox ESR nur freigegebene Webseiten erreichst, mit Kate, Python und Java arbeitest und lernst, und dass du den Lockdown nicht selbst aufheben kannst. Die Sperren tragen eine Vertrauensperson, die das Adminkennwort und das Cold-Turkey-Passwort hält; du arbeitest in einem Standardkonto. Jede Sperre nennt den Weg, den sie schließt; was offen bleibt, steht als Restrisiko in Abschnitt 10.

Voraussetzungen:

- Zwei Personen: du (Arbeitskonto, im Folgenden „du“) und die Vertrauensperson (Adminkonto, im Folgenden „der Admin“). Die Vertrauensperson ist bei der Einrichtung anwesend oder erledigt die Admin-Schritte selbst.
- Windows 11 Pro, Version 22H2 oder neuer (für `CiTool`), mit TPM 2.0 (für BitLocker mit PIN), auf dem aktuellen Updatestand.
- Zugang zum Internet während der Einrichtung; die Netzsperren werden erst in Abschnitt 8 scharf.
- Cold Turkey Pro (Produktschlüssel), Lizenzschlüssel verwahrt der Admin.
- Zeit: ein Tag für die Einrichtung ohne Audit-Phasen, danach eine Arbeitswoche Audit (App Control und Firewall im Protokollmodus), dann ein halber Tag für Ausbruchsprüfung und Scharfschalten.
- Backup: vollständige Sicherung deiner Daten auf ein externes Laufwerk vor Abschnitt 3; der Rechner wird nicht neu aufgesetzt, aber Eclipse, Edge und alle Firefox-Profile werden entfernt.
- Ein Blatt Papier und ein Umschlag für Kennwörter, die der Admin verwahrt: Adminkennwort, BitLocker-Wiederherstellungsschlüssel, BitLocker-PIN, Cold-Turkey-Passwort, Cold-Turkey-Einstellungspasswort, Lizenzschlüssel.

Prüfschritte tragen die Kennung „P“ mit Nummer (etwa P13). Die Nummern entsprechen der Liste in Dokument 1, Abschnitt 12; jeder Prüfschritt ist aber an seiner Stelle in dieser Anleitung vollständig beschrieben, mit erwartetem Ergebnis. Ein Prüfschritt steht dort, wo eine Angabe nicht aus Dokumentation belegt werden konnte und am Gerät geprüft werden muss.

Platzhalter in Befehlen: `<DEIN-KONTO>` ist der Anmeldename deines Standardkontos, `<ADMIN>` der des Adminkontos, `<STARTSEITE>` eine freigegebene Seite (zum Beispiel `https://docs.python.org/3/`). Befehle mit der Angabe „als Admin“ laufen in einer PowerShell, die über Rechtsklick > **Als Administrator ausführen** gestartet wurde.

## 2. Schutzschichten

| Schicht | Was sie schließt | Was dadurch frei bleiben kann |
|---|---|---|
| Standardkonto und Adminkonto beim Admin (Abschnitt 7.1) | jede systemweite Änderung: Richtlinien, Dienste, Firewall, Anwendungssteuerung, Deinstallation | Registry-Editor, Task-Manager, Einstellungen-App bleiben nutzbar; sie können in `HKLM` und an Diensten nichts ändern |
| BitLocker mit TPM und PIN (7.2) | Wiederherstellungsumgebung und Fremdstart als Weg an die Systemdateien | keine Einschränkung im Alltag außer der PIN beim Start |
| App Control for Business (8.3) | Programme außerhalb von `C:\Windows`, `C:\Program Files`, `C:\Program Files (x86)` und außerhalb signierter Store-Apps: portable Browser, heruntergeladene Installer, umbenannte Kopien, `winget`-Benutzerinstallationen, `mshta.exe`, `wscript.exe` | Shells, Skripte in Python und Java, alle installierten Werkzeuge |
| AppLocker je Benutzer (8.3) | Edge, Microsoft Store, fremde Store-Apps für dein Konto | Kate aus dem Store, Windows Terminal |
| Firewall, Standardaktion ausgehend Block (8.1) | Internetzugang für alles außer Firefox, Cold Turkey und Windows-Diensten: Python, Java, `curl`, PowerShell, WebView2, Kate, Git | `localhost` für eigene Entwicklung |
| Firefox-Richtlinien in `HKLM` (4) | Profile, `about:config`, Erweiterungsverwaltung, private Fenster, Entwicklerwerkzeuge, Proxy, Fehlerbehebungsmodus; `WebsiteFilter` als zweite Freigabeliste | alle freigegebenen Seiten, Downloads, lokale Dateien, PDF |
| Cold Turkey mit Sperrtyp Password (6) | nicht freigegebene Webseiten, Edge, Store, Apps mit eingebettetem Browser, Ablenkungs-Apps; schließt Firefox, wenn die Erweiterung fehlt | alles, was nicht auf der Sperrliste steht |
| Dateirechte (7.3) | Änderungen an Programmordnern, Richtlinien, Wheel-Ordner | eigene Dateien, Firefox-Profil, Kate-Konfiguration |

Profil-Dateien von Firefox (`user.js`, `userChrome.css`) werden nicht genutzt; sie gelten nur je Profil und sind für dich löschbar. Alles Sicherheitsrelevante liegt systemweit.

## 3. Vorbereitung

### 3.1 Sichern

1. Melde dich mit deinem heutigen Konto an. Kopiere `C:\Users\<DEIN-KONTO>` (mindestens Dokumente, Desktop, Downloads, `eclipse-workspace`, Projekte) auf ein externes Laufwerk.
2. Notiere, welche Programme installiert sind: **Einstellungen** > **Apps** > **Installierte Apps**. Die Liste brauchst du in 3.4.

### 3.2 Eclipse vollständig entfernen

1. **Einstellungen** > **Apps** > **Installierte Apps**: Suche „Eclipse“. Falls ein Eintrag existiert (Installation über den Eclipse-Installer), **Deinstallieren**. Bei einer ZIP-Installation gibt es keinen Eintrag.
2. Lösche den Eclipse-Programmordner (häufig `C:\Users\<DEIN-KONTO>\eclipse\...` oder `C:\eclipse`) und den Ordner des Eclipse-Installers.
3. Lösche `C:\Users\<DEIN-KONTO>\.p2` (Bundle-Pool des Installers; der Oomph-Entwickler nennt das Löschen des Installationsordners als Weg und den Pool als optional zu entfernen [Q72]) und `C:\Users\<DEIN-KONTO>\.eclipse`.
4. Sichere `C:\Users\<DEIN-KONTO>\eclipse-workspace` (eigene Projekte) und lösche ihn danach oder behalte ihn als normalen Projektordner.
5. Entferne Verknüpfungen auf dem Desktop, in der Taskleiste und im Startmenü (`%APPDATA%\Microsoft\Windows\Start Menu\Programs`).
6. Das für Eclipse installierte JDK bleibt nur, wenn es Temurin 25 ist (Abschnitt 9.4); sonst **Installierte Apps** > JDK-Eintrag > **Deinstallieren**. Oracle weist darauf hin, dass die JDK-Deinstallation Registry-Einträge zurücklassen kann [Q73]; das stört den Lockdown nicht.

### 3.3 Edge entfernen (EWR)

Windows 11 erlaubt im Europäischen Wirtschaftsraum das Deinstallieren von Edge über die Einstellungen (Microsoft-Ankündigung zur DMA-Umsetzung) [Q38].

1. **Einstellungen** > **Apps** > **Installierte Apps** > **Microsoft Edge** > Menü (drei Punkte) > **Deinstallieren**. Prüfschritt P13: Ist die Schaltfläche ausgegraut, bleibt Edge installiert; die Sperre in Abschnitt 6 und 8.3 schließt den Weg trotzdem.
2. WebView2 bleibt installiert; das ist richtig so (Abschnitt 7.5).

### 3.4 Firefox ESR installieren

1. Lade von `https://www.firefox.com/en-US/download/all/desktop-esr/` die Datei **Windows 64-bit MSI** für Deutsch [Q97]. Prüfschritt P33: Lade aus dem Versionsverzeichnis `https://releases.mozilla.org/pub/firefox/releases/153.6.0esr/` die Dateien `SHA256SUMS` und `KEY` (laut Community-Antworten dort abgelegt [Q98][Q99]) und vergleiche den Hash:
   ```powershell
   Get-FileHash .\Firefox*.msi -Algorithm SHA256
   ```
2. Installation als Admin, Standardziel `C:\Program Files\Mozilla Firefox`, Mozilla Maintenance Service mitinstallieren (Standard; er aktualisiert Firefox ohne Adminabfrage [Q59]):
   ```powershell
   msiexec /i .\Firefox*.msi /quiet
   ```
   Das MSI ist ein Wrapper um den Vollinstaller und erlaubt `/quiet` [Q58].
3. Starte Firefox einmal und schließe ihn. Prüfe über **Hilfe** > **Über Firefox**, dass „153.6.0esr“ steht.

### 3.5 Bestehende Firefox-Profile bereinigen

Profile enthalten nichts Sicherheitsrelevantes, aber Altlasten (Zweitprofile aus dem früheren Versuch) sollen weg:

1. Firefox vollständig beenden.
2. `%APPDATA%\Mozilla\Firefox\Profiles` öffnen. Behalte das Hauptprofil (Ordner mit `.default-esr`), lösche alle anderen Ordner.
3. `%APPDATA%\Mozilla\Firefox\profiles.ini` mit dem Editor öffnen; nur die Abschnitte für das Hauptprofil und `[Install...]` bleiben, alle `[ProfileN]`-Abschnitte anderer Profile löschen.
4. Im Hauptprofil die Ordner `chrome` (falls aus dem früheren Versuch vorhanden) und die Datei `user.js` löschen.

### 3.6 Reihenfolge der weiteren Abschnitte

Abschnitt 7.1 (Konten) kommt vor allen Sperren, weil die Sperren an dein Standardkonto gebunden werden. Danach: 9 (Kate, Python, Java, Git installieren, solange das Netz noch offen ist), 4 (Firefox-Richtlinien), 6 (Cold Turkey, noch ungesperrt), 8 (Firewall und Anwendungssteuerung im Protokollmodus, Audit-Woche), 10 (Ausbruchsprüfung), dann Scharfschalten (6.7 und 8.3). Abschnitt 5 ist leer, weil keine Profilmaßnahmen nötig sind.

## 4. Firefox-Richtlinien

Die Richtlinien liegen in `HKLM\SOFTWARE\Policies\Mozilla\Firefox`. Firefox liest diesen Schlüssel wie eine Gruppenrichtlinie; Maschinenrichtlinien haben Vorrang vor Benutzerrichtlinien [Q86]. Der Schlüssel ist für ein Standardkonto nicht beschreibbar. `DisableSafeMode` wirkt beim Start nur aus der Registry, nicht aus einer `policies.json` [Q6][Q88]; deshalb gehen alle Richtlinien in die Registry und es gibt keine `policies.json`.

### 4.1 Freigabeliste

Diese Liste gilt zweimal: hier für `WebsiteFilter` und in Abschnitt 6 für Cold Turkey. Trage deine freigegebenen Seiten aus Cold Turkey ein; die Dokumentationsseiten stehen schon da. Muster für `WebsiteFilter` sind Match-Patterns [Q8]; Muster für Cold Turkey sind Domains mit `*` als Platzhalter [Q16].

| Zweck | WebsiteFilter-Muster | Cold-Turkey-Eintrag |
|---|---|---|
| Python-Dokumentation | `https://docs.python.org/*` | `docs.python.org` |
| Java-Dokumentation | `https://docs.oracle.com/en/java/*`, `https://dev.java/*` | `docs.oracle.com/en/java`, `dev.java` |
| Kate-Dokumentation | `https://docs.kde.org/*`, `https://kate-editor.org/*` | `docs.kde.org`, `kate-editor.org` |
| deine Seiten aus Cold Turkey | `https://<domain>/*` je Eintrag | wie in Cold Turkey hinterlegt |

Hinweis: Ohne freigegebene Suchmaschine landet jede Suche aus der Adressleiste auf der Sperrseite. Das ist gewollt; wer eine Suchmaschine freigeben will, nimmt sie in beide Listen auf.

### 4.2 Richtlinienskript

Speichere den Block als `Firefox-Richtlinien.ps1` und führe ihn als Admin aus. Vorher die Platzhalter ersetzen: `<STARTSEITE>`, die Freigabeliste in `$exceptions`, und die Cold-Turkey-Erweiterungs-ID in `$coldTurkeyId` (Prüfschritt P1, siehe 6.2).

```powershell
# Firefox-Richtlinien.ps1
# Setzt die Firefox-Enterprise-Richtlinien in HKLM. Als Administrator ausführen.
$root = "HKLM:\SOFTWARE\Policies\Mozilla\Firefox"
New-Item -Path $root -Force | Out-Null

function Set-Dword($name, $value) {
  New-ItemProperty -Path $root -Name $name -PropertyType DWord -Value $value -Force | Out-Null
}
function Set-String($name, $value) {
  New-ItemProperty -Path $root -Name $name -PropertyType String -Value $value -Force | Out-Null
}

# Interne Seiten und Profile
Set-Dword "BlockAboutConfig"   1
Set-Dword "BlockAboutSupport"  1
Set-Dword "BlockAboutAddons"   1
Set-Dword "BlockAboutProfiles" 1
Set-Dword "DisableProfileRefresh" 1
Set-Dword "DisableProfileImport"  1
Set-Dword "DisableSafeMode"       1
Set-Dword "DisableDeveloperTools" 1
Set-Dword "PrivateBrowsingModeAvailability" 1
Set-Dword "DisableFirefoxAccounts" 1
Set-Dword "DisableAccounts"        1
Set-Dword "DisableTelemetry"       1
Set-Dword "DisableFirefoxStudies"  1
Set-Dword "DisableFeedbackCommands" 1
Set-Dword "DontCheckDefaultBrowser" 1
Set-Dword "SkipTermsOfUse"         1
Set-Dword "ExtensionUpdate"        1
Set-Dword "AppAutoUpdate"          1

# Start- und Update-Seiten
Set-Dword  "NewTabPage" 0
Set-String "OverrideFirstRunPage" ""
Set-String "OverridePostUpdatePage" ""

New-Item -Path "$root\Homepage" -Force | Out-Null
New-ItemProperty -Path "$root\Homepage" -Name "URL" -PropertyType String -Value "<STARTSEITE>" -Force | Out-Null
New-ItemProperty -Path "$root\Homepage" -Name "Locked" -PropertyType DWord -Value 1 -Force | Out-Null
New-ItemProperty -Path "$root\Homepage" -Name "StartPage" -PropertyType String -Value "homepage" -Force | Out-Null

# Proxy aus und gesperrt
New-Item -Path "$root\Proxy" -Force | Out-Null
New-ItemProperty -Path "$root\Proxy" -Name "Mode" -PropertyType String -Value "none" -Force | Out-Null
New-ItemProperty -Path "$root\Proxy" -Name "Locked" -PropertyType DWord -Value 1 -Force | Out-Null

# Downloads: fester Ordner, kein Dialog
Set-String "DownloadDirectory" '${home}\Downloads'   # einfache Anführungszeichen: ${home} ist ein Firefox-Platzhalter, keine PowerShell-Variable
Set-Dword  "PromptForDownloadLocation" 0

# Hinweise und Empfehlungen aus
New-Item -Path "$root\UserMessaging" -Force | Out-Null
foreach ($n in "ExtensionRecommendations","FeatureRecommendations","UrlbarInterventions","MoreFromMozilla","FirefoxLabs") {
  New-ItemProperty -Path "$root\UserMessaging" -Name $n -PropertyType DWord -Value 0 -Force | Out-Null
}
New-ItemProperty -Path "$root\UserMessaging" -Name "SkipOnboarding" -PropertyType DWord -Value 1 -Force | Out-Null
New-ItemProperty -Path "$root\UserMessaging" -Name "Locked" -PropertyType DWord -Value 1 -Force | Out-Null

# Erweiterungen: alles gesperrt außer den drei erzwungenen
$coldTurkeyId = "<COLD-TURKEY-ID>"
$extensionSettings = @"
{
  "*": { "installation_mode": "blocked", "blocked_install_message": "Erweiterungen werden vom Admin verwaltet." },
  "uBlock0@raymondhill.net": { "installation_mode": "force_installed",
    "install_url": "https://addons.mozilla.org/firefox/downloads/latest/uBlock0@raymondhill.net/latest.xpi" },
  "{ef87d84c-2127-493f-b952-5b4e744245bc}": { "installation_mode": "force_installed",
    "install_url": "https://addons.mozilla.org/firefox/downloads/latest/{ef87d84c-2127-493f-b952-5b4e744245bc}/latest.xpi" },
  "$coldTurkeyId": { "installation_mode": "force_installed",
    "install_url": "https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi",
    "private_browsing": true }
}
"@
New-ItemProperty -Path $root -Name "ExtensionSettings" -PropertyType MultiString -Value @($extensionSettings) -Force | Out-Null

# Präferenzen: neues Profil-Feature aus
$preferences = @"
{
  "browser.profiles.enabled": { "Value": false, "Status": "locked" }
}
"@
New-ItemProperty -Path $root -Name "Preferences" -PropertyType MultiString -Value @($preferences) -Force | Out-Null

# WebsiteFilter: alles sperren, Freigabeliste als Ausnahmen
New-Item -Path "$root\WebsiteFilter\Block" -Force | Out-Null
New-Item -Path "$root\WebsiteFilter\Exceptions" -Force | Out-Null
New-ItemProperty -Path "$root\WebsiteFilter\Block" -Name "1" -PropertyType String -Value "<all_urls>" -Force | Out-Null
$exceptions = @(
  "https://docs.python.org/*",
  "https://docs.oracle.com/en/java/*",
  "https://dev.java/*",
  "https://docs.kde.org/*",
  "https://kate-editor.org/*"
  # weitere Einträge aus der Freigabeliste
)
$i = 1
foreach ($e in $exceptions) {
  New-ItemProperty -Path "$root\WebsiteFilter\Exceptions" -Name "$i" -PropertyType String -Value $e -Force | Out-Null
  $i++
}
Write-Host "Fertig. Firefox starten und about:policies prüfen."
```

### 4.3 Wirkung je Richtlinie

| Richtlinie | Wirkung | Beleg |
|---|---|---|
| `BlockAboutConfig` | sperrt `about:config` und damit jede Präferenzänderung von Hand | Q4 |
| `BlockAboutSupport` | sperrt `about:support` (Profilordner öffnen, Firefox bereinigen) | Q4 |
| `BlockAboutAddons` | sperrt die Erweiterungsverwaltung | Q4 |
| `BlockAboutProfiles` | sperrt `about:profiles`, `about:profilemanager`, `about:editprofile`, `about:newprofile`, `about:deleteprofile` und schaltet das Profil-Flyout ab | Q10, Q84 |
| `DisableProfileRefresh` | entfernt „Firefox bereinigen“ | Q4 |
| `DisableProfileImport` | entfernt den Import aus anderen Browsern | Q4 |
| `DisableSafeMode` | sperrt den Fehlerbehebungsmodus, aus der Registry auch per Kommandozeile und Umschalttaste | Q6, Q88 |
| `DisableDeveloperTools` | entfernt alle Entwicklerwerkzeuge samt Tastenkürzeln, `about:debugging`, Remote-Agent | Q4, Q137 |
| `PrivateBrowsingModeAvailability` = 1 | private Fenster nicht verfügbar | Q7 |
| `DisableFirefoxAccounts`, `DisableAccounts` | Konto und Sync aus; `DisableAccounts` hat Vorrang (ESR ab 128) | Q4 |
| `DisableTelemetry`, `DisableFirefoxStudies`, `DisableFeedbackCommands` | keine Datensendungen, Studien, Meldeformulare | Q4 |
| `DontCheckDefaultBrowser` | kein Standardbrowser-Dialog | Q4 |
| `SkipTermsOfUse` | keine Nutzungsbedingungen beim Start | Q4 |
| `ExtensionUpdate` = 1 | erzwungene Erweiterungen bleiben aktuell | Q4 |
| `AppAutoUpdate` = 1 | ESR-Updates werden ohne Nachfrage installiert | Q4 |
| `NewTabPage` = 0, `OverrideFirstRunPage` = "", `OverridePostUpdatePage` = "" | leerer neuer Tab, keine Mozilla-Seiten nach Start oder Update | Q4 |
| `Homepage` (URL, Locked, StartPage) | Start immer auf der Startseite | Q4 |
| `Proxy` Mode none, Locked | kein Proxy, nicht änderbar | Q11 |
| `DownloadDirectory`, `PromptForDownloadLocation` = 0 | Downloads immer nach `Downloads`, kein Dateidialog | Q4 |
| `UserMessaging` | keine Empfehlungen, kein Onboarding | Q4 |
| `ExtensionSettings` | sperrt alle Erweiterungen und erzwingt uBlock Origin, ActivityWatch und Cold Turkey; `install_url` ist für AMO-Erweiterungen seit 153 optional, für die Cold-Turkey-XPI nötig; `private_browsing` erlaubt die Cold-Turkey-Erweiterung in privaten Fenstern | Q9 |
| `Preferences` `browser.profiles.enabled` locked false | zweites Netz gegen das Profil-Feature | Q5, Q13 |
| `WebsiteFilter` | zweite Freigabeliste im Browser; wirkt auch im Fehlerbehebungsmodus, wenn Erweiterungen aus sind; bis 1000 Einträge je Liste | Q8 |

Nicht gesetzt, mit Grund: `DisableAppUpdate` (Sicherheitsupdates bleiben an), `PDFjs` (Dokumentation lesen), `LocalFileLinks` und `file:`-Sperre (lokale Dateien bleiben erreichbar, Entscheidung E15), `SanitizeOnShutdown` (kein Weg), `DisableFormHistory` (kein Weg), `Handlers` (Downloads werden ohnehin nur ausgeführt, wenn App Control es erlaubt).

### 4.4 Verifikation

1. Firefox starten, `about:policies` öffnen: Reiter **Aktiv** listet alle Richtlinien; Reiter **Fehler** muss leer sein. Ein Fehler bei `ExtensionSettings` deutet auf ein JSON-Problem im Skript (Anführungszeichen, Kommas).
2. Nach dem ersten Start sind uBlock Origin, ActivityWatch und Cold Turkey im Erweiterungsmenü sichtbar, ohne Schaltfläche **Entfernen**.
3. Verhalten: `about:config` zeigt die Sperrseite; **Strg+Umschalt+P** öffnet kein privates Fenster; **F12** tut nichts; eine nicht freigegebene Seite zeigt die Sperrseite von Firefox; eine freigegebene Seite lädt.

## 5. Profil

Keine Maßnahmen. Es gibt keine `user.js` und keine `userChrome.css`; sie gelten nur je Profil, und ein neues Profil (über `-P` jederzeit möglich) hätte sie nicht [Q85][Q86]. Alle Sperren aus Abschnitt 4 gelten in jedem Profil dieser Installation. Die Avatar-Schaltfläche (Profil-Flyout) verschwindet mit `BlockAboutProfiles`; bleibt ein Rest sichtbar, ist er ohne Funktion (Prüfschritt in 10.1, Zeile F1).

## 6. Cold Turkey

Alle Bezeichnungen folgen dem User Guide des Herstellers [Q16]. Cold Turkey ist die sichtbare Schicht: Es sperrt Webseiten im Browser und beendet gesperrte Programme. Was Cold Turkey nicht leistet (Administratoren aufhalten, Programme unter fremdem Namen erkennen, Netzverkehr filtern), leisten Abschnitt 7 und 8.

### 6.1 Installation

1. Download als Admin von `https://getcoldturkey.com/download/win/` (Datei `Cold_Turkey_Installer.exe`); Version laut Release Notes 4.9(2) [Q30]. Installation verlangt Adminrechte [Q18]. Zielordner `C:\Program Files\Cold Turkey`.
2. Nach der Installation zeigt Cold Turkey die gefundenen Browser und bietet je Browser **Install** an [Q16]. Für Firefox ESR: **Install** klicken. Erkennt Cold Turkey Firefox ESR nicht (die Systemanforderungen nennen Firefox, Beta, Dev und Nightly, nicht ESR [Q18]), bleibt die Erweiterung über die Richtlinie aus Abschnitt 4 erzwungen; Prüfschritt in 6.2.
3. Pro freischalten: Produktschlüssel in der App eingeben (**Settings**, Bereich für den Schlüssel; der Admin trägt ihn ein) [Q32].
4. Diese Schritte laufen in deinem Standardkonto, nachdem Abschnitt 7.1 erledigt ist, damit die Blöcke an dein Konto gebunden werden; die Installation selbst fragt nach dem Adminkennwort.

### 6.2 Firefox-Erweiterung und Erweiterungs-ID (Prüfschritt P1)

Die Erweiterung liegt nicht auf addons.mozilla.org, sondern unter `https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi` [Q34]. Für `ExtensionSettings` brauchst du ihre ID:

1. XPI als Admin herunterladen (**Rechtsklick** > **Link speichern unter**, Dateiendung `.xpi` belassen).
2. In der PowerShell:
   ```powershell
   Copy-Item .\Cold_Turkey_Firefox_Addon.xpi .\ct.zip
   Expand-Archive .\ct.zip -DestinationPath .\ct
   Select-String -Path .\ct\manifest.json -Pattern '"id"'
   ```
   Die Zeile unter `browser_specific_settings` > `gecko` > `id` ist die ID (Form `name@domain` oder `{uuid}`). Trage sie in `Firefox-Richtlinien.ps1` ein und führe das Skript erneut aus.
3. Erwartung nach Firefox-Neustart: Die Erweiterung ist installiert, in `about:policies` ohne Fehler. Bleibt sie aus, war die XPI nicht von Mozilla signiert; dann bleibt der Weg über Cold Turkeys eigenen Installer (6.1 Schritt 2), und die Zeile wird aus `$extensionSettings` entfernt. Cold Turkey schließt Firefox ohnehin, wenn die Erweiterung in einem Profil fehlt (Grace Period, 6.5).
4. Die Berechtigung **In privaten Fenstern ausführen** wird über `private_browsing: true` erzwungen; private Fenster sind ohnehin abgeschaltet.

### 6.3 Blocklisten

Zwei fortlaufende Blöcke („continuous“), keine Zeitpläne, damit die Systemzeit keine Rolle spielt.

| | Block „Lockdown“ | Block „Ablenkung“ |
|---|---|---|
| Zweck | Freigabeliste im Browser, Browser und Apps mit Webinhalt sperren | Apps ohne Webinhalt sperren, die nur ablenken (Entscheidung E11) |
| Typ | continuous | continuous |
| Websites | `*.*` (alles) | keine |
| Website Exceptions | `file://*` (Standard, lokale Dateien), `localhost` (eigene Entwicklung), dazu jeder Eintrag der Freigabeliste aus 4.1 in Cold-Turkey-Schreibweise | keine |
| Applications | siehe 6.4, Liste A | siehe 6.4, Liste B |
| Breaks | No breaks | No breaks |
| Sperrtyp (Lock) | Password (Pro), gesetzt erst nach 10 | Password |
| Sperrdauer | unbefristet (continuous, Toggle bleibt an) | unbefristet |

Begründung je Ausnahme: `file://*` ist der Standardeintrag und lässt lokale PDF- und HTML-Dateien laden [Q16]; `localhost` ist laut Release Notes als Ausnahme zulässig und wird für eigene Webanwendungen gebraucht [Q30]; die Dokumentationsseiten sind Arbeitsmaterial. Keine Ausnahmen für `about:`- oder `moz-extension://`-Schemata (Dokument 1, Abschnitt 4.2); Prüfschritt P2 in 10.1.

### 6.4 Anwendungssperren

Hinzufügen über **Applications** > **Add exe file** (Datei wählen) bzw. **Add Windows 10 app** (Store-Apps) [Q16]. Eine gesperrte exe-Datei kann während des Blocks nicht verschoben oder umbenannt werden [Q16].

Liste A, Block „Lockdown“ (Weg: zweiter Browser oder Webinhalt außerhalb Firefox):

| Programm | Pfad oder Store-App | Weg, den die Sperre schließt |
|---|---|---|
| Microsoft Edge | `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` (nur wenn Edge nicht entfernt wurde oder zurückkehrt) | zweiter Browser |
| Microsoft Store | Store-App „Microsoft Store“ (`WinStore.App.exe`, `StoreExperienceHost.exe`) | Installation fremder Apps je Benutzer |
| `mshta.exe`, `hh.exe` | `C:\Windows\System32\mshta.exe`, `C:\Windows\hh.exe` (Prüfschritt P11, ob vorhanden) | HTML-Anzeige über die alte Internet-Explorer-Engine |
| `wsl.exe`, `wslInstaller.exe` | `C:\Windows\System32\wsl.exe` und Store-App, bis P10 zeigt, dass WSL für dich nicht installierbar ist (Microsofts Anleitung verlangt für `wsl --install` eine PowerShell als Administrator [Q206]) | Linux mit eigenem Browser |
| WebView2-Hosts: Widgets, Microsoft Teams, Outlook (new), Microsoft 365 Copilot, Copilot, Hilfe anfordern, Feedback-Hub, Dev Home, Phone Link (Smartphone-Link), Cross-Device-Dienste, News, Wetter, Microsoft Start, Clipchamp, Xbox-Apps, Game Bar, ClickToDo, AI-Context, People | als Store-App (**Add Windows 10 app**) bzw. exe gemäß Edge-Dokumentation: `Widgets.exe`, `WidgetService.exe`, `WidgetBoard.exe`, `WebExperienceHostApp.exe`, `MicrosoftStartFeed...`, `Microsoft.Msn.News.exe`, `Microsoft.Msn.Weather.exe`, `ms-teams.exe`, `ms-teams_autostarter.exe`, `ms-teamsupdate.exe`, `olk.exe`, `olkBg.exe`, `olkExthost.exe`, `olkFullTrustExecutor.exe`, `olkMcpServer.exe`, `olkPushNotification...`, `CoPilotLibraryBroker.exe`, `m365copilot_autostarter.exe`, `m365copilot_widget.exe`, `m365copilotwindows...`, `aicontext.exe`, `AppActions.exe`, `ClickToDo.exe`, `DevHome.exe`, `DiscoveryHubApp.exe`, `FeedbackHub.exe`, `FeedbackHubServices.exe`, `GetHelp.exe`, `PhoneExperienceHost.exe`, `PhoneAgentMcpServer.exe`, `YourPhoneAppProxy.exe`, `YourPhoneAppProxyHost.exe`, `CrossDeviceFilesHost.exe`, `CrossDeviceResume.exe`, `CrossDeviceService.exe`, `CrossDeviceSettings...`, `CrossDeviceShareHost.exe`, `CrossDeviceStreaming...`, `EdgeGameAssist.exe`, `GameBar.exe`, `GameBarElevatedFT.exe`, `GameBarFTServer.exe`, `XboxGameBarWidgets.exe`, `Xbox.TCUI.exe`, `XboxIdp.exe`, `XboxPartyChatHost.exe`, `XboxPcApp.exe`, `XboxPcAppAdminServer.exe`, `XboxPcAppCE.exe`, `XboxPcAppFT.exe`, `XboxPcTray.exe`, `Clipchamp.exe`, `Clipchamp.Widgets.exe`, `PeopleExperienceHost.exe` | Webinhalt in eingebettetem Browser; die Firewall nimmt WebView2 zusätzlich das Netz, Cold Turkey macht die Sperre sichtbar |

Liste B, Block „Ablenkung“ (kein Ausbruchsweg; Zweck Schutz vor Ablenkung):

`Solitaire.exe`, `Microsoft.Solitaire...`, `ClockWidgets.exe`, `Todo.exe`, `StickyNotesStub.exe`, `DesktopStickerEditor...`, `PAD.Console.Host.exe` (Power Automate). Store-Apps über **Add Windows 10 app**.

Nicht sperren (Freigabeliste; Grund in Klammern): `firefox.exe` (Browser), `Cold Turkey Blocker.exe` (Blocker), `kate.exe` (Editor, Store-Paket), `kioworker.exe` (Kate-Dateizugriff), `python.exe`, `pythonw.exe`, `py.exe`, `pylsp.exe` (Interpreter und Language Server), `java.exe`, `javaw.exe` (JDK und JDT LS), `cmd.exe`, `powershell.exe`, `pwsh.exe`, `wt.exe`, `conhost.exe` (Shells für Kate und Entwicklung), `git.exe`, `bash.exe`, `sh.exe`, `git-bash.exe` (Git), `explorer.exe`, `Taskmgr.exe`, `regedit.exe`, `SystemSettings.exe` (Systemwerkzeuge; im Standardkonto ohne Ausbruchswirkung), `msedgewebview2.exe` (Laufzeit für Windows-Dialoge; ohne Netz durch die Firewall), `mspaint.exe`, `SnippingTool.exe`, `ScreenClippingHost.exe`, `SoundRec.exe`, `CalculatorApp.exe`, `Notepad.exe`, `Photos.exe` (Werkzeuge ohne Webinhalt), die Prozesse der Edge-Dokumentation Abschnitt 5 (`AccountsControlHost.exe`, `AppInstaller.exe`, `AppResolverUX.exe`, `AuthenticationManager.exe`, `BioEnrollmentHost.exe`, `CredDialogHost.exe`, `FileExplorer.exe`, `FilePicker.exe`, `IGCC.exe`, `LockApp.exe`, `PrintDialog.exe`, `SearchHost.exe`, `SecHealthUI.exe`, `ShellExperienceHost.exe`, `StartMenuExperienceHost.exe`, `TextInputHost.exe`, `Time.exe`, `WindowsPackageManagerServer.exe`), `winget.exe`, `WindowsBackupClient.exe`, `AssignedAccessLockApp.exe`, `CHXSmartScreen.exe`, `WpcUapApp.exe`, `CameraBarcodeScanner...`, `AddSuggestedFolders...`, `SpeechToTextOverlay64...`, `QuickAssist.exe`, `DeploymentAgent.exe` (Systemkomponenten ohne Weg oder durch App Control abgedeckt).

Hinweiskasten: Sperre nie `explorer.exe`, `svchost.exe`, `SystemSettings.exe` oder `Cold Turkey Blocker.exe` selbst. Windows wird sonst unbedienbar, und mit gesperrtem Block kann nur der Admin die Liste wieder ändern.

### 6.5 Einstellungsseite

| Reiter | Option | Wert | Grund |
|---|---|---|---|
| Preferences | Open on startup | an | Oberfläche beim Anmelden sichtbar; Blocken läuft unabhängig davon |
| Preferences | Move to system tray | an | Fenster stört nicht |
| Preferences | Show extension install dialog | an | zeigt, wenn ein Browser ohne Erweiterung erkannt wird |
| Notifications | alle Benachrichtigungen | an | du siehst, warum etwas geschlossen wurde |
| Blocking | Block unsupported browsers | an | Weg: Browser, den Cold Turkey nicht kennt |
| Blocking | Block browsers if extension crashes | an | Weg: Erweiterung über den Browser-Task-Manager beenden |
| Blocking | Block time changes | an | Weg: Systemzeit (zusätzlich hat ein Standardkonto das Recht nicht, 7.1) |
| Blocking | Block task manager | aus | kein Weg im Standardkonto; der Task-Manager bleibt zur Fehlersuche |
| Blocking | Block inactive browser tabs | an | sonst bleiben gesperrte Seiten in Hintergrund-Tabs offen |
| Blocking | Block embedded content | an | eingebettete Videos von gesperrten Seiten auf freigegebenen Seiten |
| Blocking | Block Pause for a Cause | an | sonst 10 Minuten Pause gegen Spende möglich |
| Blocking | Block the installer | an | Weg: Neuinstallation oder Update ohne Admin; Updates nur im Wartungsfenster (11.2) |
| Blocking | Force allow access to file URLs | aus | lokale Dateien bleiben über die Standardausnahme offen |
| Blocking | Grace period | 60 Sekunden (Standard) | Zeit bis zum Schließen eines Browsers ohne Erweiterung; Prüfung alle 10 Sekunden |
| Blocking | Frozen Turkey | nicht genutzt | kein Zweck |
| Statistics | Enable statistics | an | zeigt dir, was gesperrt wurde |
| Statistics | Allowance Strictness | ohne Bedeutung (keine Allowance) | |
| Password | Password Settings | an, Passwort vom Admin (6.7) | Weg: Oberfläche und Overview ohne Passwort |

Die Features-Seite nennt den Reiter auch „Block Strictness“ [Q31]; gemeint ist derselbe Reiter.

### 6.6 Browserschutz

Browser ohne Erweiterung werden während eines gesperrten Blocks nach der Grace Period geschlossen; Cold Turkey zeigt einen roten Hinweis unten links und öffnet den Installationsdialog [Q16]. Private Fenster sind per Richtlinie aus; die Erweiterung hat über `private_browsing: true` trotzdem die Berechtigung. Die Erweiterung lässt sich nicht entfernen (`force_installed`, `about:addons` gesperrt); wird sie über den Browser-Task-Manager beendet, greift „Block browsers if extension crashes“.

### 6.7 Passwort und Scharfschalten

1. Der Admin erzeugt zwei Passwörter mit 24 oder mehr zufälligen Zeichen (Passwort-Generator des eigenen Passwort-Managers oder Würfelverfahren): eines für die Blöcke (Sperrtyp Password), eines für **Password Settings**.
2. Reihenfolge: Beide Blöcke mit eingeschaltetem Toggle, aber ohne Sperre betreiben, bis die Ausbruchsprüfung (10) bestanden ist. Dann je Block **Lock** > **Password** mit dem Blockpasswort; danach **Settings** > **Password** > **Password Settings** mit dem Einstellungspasswort.
3. Verwahrung: beide Passwörter auf dem Umschlag-Blatt beim Admin, nicht auf dem Rechner, nicht bei dir.
4. Notfall: Nur der Admin kann einen Block entsperren (Passwort eingeben, Block ausschalten). Ohne Admin gibt es keinen Weg; das ist Absicht. Für Reparaturen hält der Admin das Wartungsfenster (11).

### 6.8 Ist-Soll-Abgleich zur Edge-Dokumentation

| Einstellung | Ist (Edge-Dokumentation) | Soll | Grund |
|---|---|---|---|
| Websperren | nur Eclipse-Seiten gesperrt (sonst nicht dokumentiert) | `*.*` mit Freigabeliste | Zweck: nur freigegebene Seiten |
| Python-Prozesse | 11 Einträge gesperrt | freigegeben | Firewall und App Control (8) |
| `cmd.exe`, `powershell.exe`, `wt.exe` | gesperrt | freigegeben | nötig für Kate und Entwicklung; Weg durch 8 geschlossen |
| `regedit.exe`, `Taskmgr.exe`, `QuickAssist.exe` | gesperrt | freigegeben | Standardkonto |
| `winget.exe`, `DeploymentAgent.exe` | gesperrt | freigegeben | App Control und AppLocker-Packaged-App-Regel |
| Store-Prozesse | gesperrt | gesperrt | zweite Schicht neben AppLocker |
| WebView2-Hosts, Ablenkungs-Apps | gesperrt | gesperrt, in zwei Blöcken | Weg eingebetteter Browser; Zweck |
| `mspaint.exe`, `SnippingTool.exe`, `ScreenClippingHost.exe`, `SoundRec.exe`, `WindowsBackupClient.exe`, Systemkomponenten | gesperrt | freigegeben | kein Weg |
| `eclipse.exe`, Eclipse-Websperren | freigegeben bzw. gesperrt | entfernt | Eclipse entfernt |
| `msedge.exe`, `mshta.exe`, `hh.exe` | nicht enthalten | gesperrt | zweiter Browser, IE-Engine |
| „Strict Mode“, „Block uninstalling Cold Turkey“, „Block changing system time“ | so benannt | Sperrtyp Password, „Block the installer“, „Block time changes“ | Namen der aktuellen Oberfläche |
| Zeitpläne | nicht dokumentiert | keine | Systemzeit ohne Bedeutung |

## 7. Betriebssystem

### 7.1 Konten

Ziel: dein Konto ist Standardbenutzer, der Admin hat ein eigenes lokales Adminkonto, das eingebaute Konto „Administrator“ ist deaktiviert (Microsoft empfiehlt das [Q55]). Standardbenutzer bekommen bei jeder Verwaltungsaufgabe die Anmeldeaufforderung und brauchen Admin-Anmeldedaten [Q54].

Als Admin (noch in deinem heutigen Konto, das derzeit Adminrechte hat) in PowerShell:

```powershell
# 1. Adminkonto der Vertrauensperson anlegen; Kennwort tippt die Vertrauensperson selbst ein
$pw = Read-Host -AsSecureString "Kennwort fuer <ADMIN>"
New-LocalUser -Name "<ADMIN>" -Password $pw -FullName "Vertrauensperson" -PasswordNeverExpires
Add-LocalGroupMember -SID "S-1-5-32-544" -Member "<ADMIN>"      # S-1-5-32-544 = Gruppe Administratoren

# 2. Eingebautes Konto Administrator deaktivieren
Disable-LocalUser -Name "Administrator"
```

Syntax von `New-LocalUser` nach Microsoft [Q185]; `Add-LocalGroupMember` nimmt die Gruppe als SID und Mitglieder als Namen [Q205]; S-1-5-32-544 ist die bekannte SID der Gruppe Administratoren [Q204]. Danach abmelden, als `<ADMIN>` anmelden, prüfen, dass **Einstellungen** > **Konten** > **Andere Benutzer** das Konto als Administrator zeigt, und erst dann dein Konto herabstufen:

```powershell
# 3. Dein Konto wird Standardbenutzer (als <ADMIN> ausführen)
Remove-LocalGroupMember -SID "S-1-5-32-544" -Member "<DEIN-KONTO>"
```

Prüfschritt: Melde dich als `<DEIN-KONTO>` an, öffne eine PowerShell als Administrator; die Anmeldeaufforderung muss nach dem Kennwort von `<ADMIN>` fragen.

Rechte, die damit geschlossen sind: Schreiben in `HKLM`, Dienste, Firewall, Anwendungssteuerung, Installation nach `C:\Program Files`, Zeitänderung (das Recht „Ändern der Systemzeit“ haben nur Administratoren und der lokale Dienst [Q78]), Deinstallation von Cold Turkey.

Hinweiskasten: Das Adminkennwort darf nur der Admin kennen. Ein Kennwort, das du „für Notfälle“ kennst, hebt jede Maßnahme dieser Anleitung auf.

### 7.2 BitLocker mit TPM und PIN

Weg, den das schließt: Wiederherstellungsumgebung (WinRE über **Umschalt+Neustart** oder nach zwei Fehlstarts) und Start von fremden Medien. Ohne PIN oder Wiederherstellungsschlüssel ist das Laufwerk nicht lesbar [Q57]. Als Admin:

1. `gpedit.msc` öffnen: **Computerkonfiguration** > **Administrative Vorlagen** > **Windows-Komponenten** > **BitLocker-Laufwerkverschlüsselung** > **Betriebssystemlaufwerke** > **Zusätzliche Authentifizierung beim Start anfordern**: **Aktiviert**, darin **TPM-Systemstart-PIN konfigurieren** auf **Systemstart-PIN bei TPM erforderlich**. Prüfschritt: Diese Richtlinie ist Voraussetzung für einen PIN-Schutz; die Bezeichnungen der deutschen Oberfläche sind aus Microsofts englischer Doku abgeleitet und am Bildschirm zu prüfen.
2. PowerShell als Admin; die PIN tippt der Admin ein, nicht du:
   ```powershell
   $pin = Read-Host -AsSecureString "BitLocker-PIN (mindestens 6 Ziffern)"
   Add-BitLockerKeyProtector -MountPoint "C:" -RecoveryPasswordProtector
   Enable-BitLocker -MountPoint "C:" -EncryptionMethod Aes256 -UsedSpaceOnly -Pin $pin -TpmAndPinProtector
   (Get-BitLockerVolume -MountPoint "C:").KeyProtector
   ```
   `Enable-BitLocker` nimmt nur einen Schutz je Aufruf; der Wiederherstellungsschlüssel kommt deshalb vorher über `Add-BitLockerKeyProtector` [Q186].
3. Den Wiederherstellungsschlüssel (48 Ziffern, Ausgabe des letzten Befehls) und die PIN schreibt der Admin auf das Umschlag-Blatt. Nichts davon bleibt auf dem Rechner.
4. Neustart; die PIN-Abfrage erscheint vor Windows.

Unter Variante B (du behältst Adminrechte) schließt BitLocker nichts, weil ein Admin die Verschlüsselung aufheben kann (Dokument 1, Abschnitt 10).

### 7.3 Dateirechte

Standardbenutzer können in `C:\Program Files`, `C:\Program Files (x86)`, `C:\Windows` und `HKLM` nicht schreiben; das ist der Zustand nach der Installation, nichts muss geändert werden. Prüfschritt in deinem Konto:

```powershell
New-Item "C:\Program Files\Mozilla Firefox\test.txt"   # erwartet: Zugriff verweigert
```

Zusätzlich wird in 8.2 ein Ordner `C:\Program Files\Wheels` angelegt, in den nur der Admin schreibt.

### 7.4 Updates

Sicherheitsupdates bleiben an. Microsoft warnt, dass abgeschaltete automatische Updates auch Sicherheitsupdates stoppen [Q48]; Defender-Signaturen kommen über Windows Update [Q50]. Als Admin in `gpedit.msc`:

1. **Computerkonfiguration** > **Administrative Vorlagen** > **Windows-Komponenten** > **Windows Update** > **Automatische Updates konfigurieren**: **Aktiviert**, Option **4, Automatisch herunterladen und Installation planen** [Q49].
2. Aktive Stunden in **Einstellungen** > **Windows Update** > **Erweiterte Optionen** auf deine Arbeitszeit setzen (Neustarts außerhalb).
3. Optional: **Zugriff auf alle Windows Update-Funktionen entfernen** (nur die Schaltfläche „Nach Updates suchen“ verschwindet, Hintergrund-Scans laufen weiter [Q49]). Kein Ausbruchsweg, deshalb nicht nötig.

Firefox ESR aktualisiert sich über den Maintenance Service ohne Adminabfrage [Q59] und bleibt innerhalb der Linie 153 (Richtlinien kommen hinzu, keine fällt weg). Kate aus dem Store aktualisiert der Store im Hintergrund (Prüfschritt P16). Cold Turkey, Python, JDK, JDT LS, Git: nur im Wartungsfenster (11).

Was die Edge-Dokumentation stattdessen tat (`wuauserv`, `UsoSvc`, `TrustedInstaller` deaktiviert) wird rückgängig gemacht, falls der Rechner diesen Stand hat:

```powershell
sc.exe config wuauserv start= demand
sc.exe config UsoSvc start= auto
sc.exe config TrustedInstaller start= demand
```

Diese Dienste schließen keinen Ausbruchsweg (Dokument 1, Abschnitt 3.2); die Starttypen sind die Windows-Standardwerte (Prüfschritt: `Get-Service wuauserv,UsoSvc,TrustedInstaller | Select Name,StartType` auf einem anderen Windows 11 vergleichen).

Ebenso entfernen, falls gesetzt: `NoControlPanel` und `DisableTaskMgr`:

```powershell
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" -Name NoControlPanel -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" -Name NoControlPanel -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Policies\System" -Name DisableTaskMgr -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System" -Name DisableTaskMgr -ErrorAction SilentlyContinue
```

Registry-Pfade laut Microsoft [Q51][Q52]. Beide Richtlinien schließen im Standardkonto keinen Weg und nehmen dir Anzeige-, Ton- und Druckereinstellungen.

### 7.5 Edge, WebView2, Protokoll-Handler

Edge ist entfernt (3.3) oder gesperrt (6.4 und 8.3). WebView2 bleibt: Es ist Teil von Windows 11, wird von Widgets, Teams, Office, Outlook und vielen weiteren Apps genutzt und lässt sich nach Installation durch Edge Update nicht deinstallieren [Q40][Q79]. Edge-Browserrichtlinien gelten nicht für WebView2, und WebView2 hat keine URL-Sperrliste [Q39][Q41]. Deshalb: `msedgewebview2.exe` wird nicht gesperrt, bekommt aber durch die Firewall-Standardaktion (8.1) keinen Internetzugang, und die Apps, die es als Fenster nutzen, stehen in Cold Turkey (6.4). Links mit `microsoft-edge:` laufen ins Leere, weil kein Edge startet; Firefox ist Standardbrowser (**Einstellungen** > **Apps** > **Standard-Apps** > **Firefox** > **Als Standard festlegen**, in deinem Konto).

### 7.6 App-Ausführungsaliase

Der Store-Installationsmanager für Python registriert Aliase `python`, `python3`, `py`, `pymanager`, `pyw`, `pythonw`, die ohne Installation den Store öffnen [Q63]. In deinem Konto: **Einstellungen** > **Apps** > **Erweiterte App-Einstellungen** > **App-Ausführungsaliase** (Microsoft nennt die Seite so; der vollständige Pfad ist Prüfschritt P12 [Q74]): alle Einträge für Python und den Installationsmanager **Aus**. Erwartung: `python` in einer neuen PowerShell startet `C:\Program Files\Python 3.15\python.exe` (`Get-Command python`).

### 7.7 Neu bewertete Windows-Sperren der Edge-Dokumentation

| Edge-Dokumentation | Hier | Grund |
|---|---|---|
| `NoControlPanel`, `DisableTaskMgr` | entfernt | kein Weg im Standardkonto |
| Dienste `wuauserv`, `UsoSvc`, `TrustedInstaller` deaktiviert | Standardwerte | Sicherheitsupdates |
| `regedit.exe`, `cmd.exe`, `powershell.exe`, `Taskmgr.exe` in Cold Turkey | frei | Standardkonto, App Control, Firewall |
| Edge-Richtlinien (`BrowserAddProfileEnabled` usw.) | entfallen | Edge entfernt oder gesperrt |
| Registry-Härtung „unumgehbar“ | Standardkonto und BitLocker | nur so hält Registry-Schutz gegen den Nutzer selbst |

## 8. Ausgleichsmaßnahmen

### 8.1 Netz: Firewall, Standardaktion ausgehend Block

Weg, den das schließt: Internetzugang für Python, Java, die Language Server, `curl.exe`, PowerShell, `certutil`, `bitsadmin`, WebView2, Kate samt KIO-Worker, Git und jedes andere Programm außer Firefox, Cold Turkey und Windows-Diensten. Windows-Firewall: Standard ausgehend ist Allow; Block-Regeln haben Vorrang vor Allow-Regeln; die Standardaktion lässt sich auf Block setzen, was Microsoft für Umgebungen mit strengen Kontrollen vorsieht [Q67][Q68]. Programmregeln brauchen den vollständigen Pfad ohne Wildcards [Q24]. Ein Standardkonto kann keine Regeln anlegen; die Werkzeuge verlangen Adminrechte [Q67].

Phase 1, Protokollmodus (eine Arbeitswoche): Standardaktion bleibt Allow, die Firewall protokolliert nichts Blockiertes; stattdessen wird in Phase 2 geschaltet und das Protokoll der verworfenen Pakete ausgewertet. Als Admin, Datei `Firewall.ps1`:

```powershell
# Firewall.ps1: Standardaktion ausgehend Block mit Freigabeliste. Als Administrator ausführen.
$sys = "C:\Windows\System32"

# Protokoll verworfener Pakete (Auswertung in der Audit-Woche)
Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True `
  -LogFileName "%SystemRoot%\System32\LogFiles\Firewall\pfirewall.log" -LogMaxSizeKilobytes 32767

# Freigaben: Programme
New-NetFirewallRule -DisplayName "Lockdown Allow Firefox" -Direction Outbound -Action Allow `
  -Program "C:\Program Files\Mozilla Firefox\firefox.exe"
Get-ChildItem "C:\Program Files\Cold Turkey" -Filter *.exe -Recurse | ForEach-Object {
  New-NetFirewallRule -DisplayName "Lockdown Allow Cold Turkey $($_.Name)" -Direction Outbound -Action Allow -Program $_.FullName
}

# Freigaben: Windows-Dienste (laufen in svchost.exe): Update, Übermittlungsoptimierung, Store-Updates, DNS, DHCP, Zeit, Lizenz, Netzwerkstatus, Zertifikate
foreach ($svc in "wuauserv","UsoSvc","BITS","DoSvc","InstallService","Dnscache","Dhcp","W32Time","ClipSVC","LicenseManager","NlaSvc","CryptSvc","WinDefend") {
  New-NetFirewallRule -DisplayName "Lockdown Allow Dienst $svc" -Direction Outbound -Action Allow -Service $svc
}

# Wartungsregeln (deaktiviert; der Admin schaltet sie im Wartungsfenster ein, 11.2)
New-NetFirewallRule -DisplayName "Wartung Python" -Direction Outbound -Action Allow `
  -Program "C:\Program Files\Python 3.15\python.exe" -Enabled False
New-NetFirewallRule -DisplayName "Wartung Git" -Direction Outbound -Action Allow `
  -Program "C:\Program Files\Git\mingw64\bin\git-remote-https.exe" -Enabled False
New-NetFirewallRule -DisplayName "Wartung PowerShell" -Direction Outbound -Action Allow `
  -Program "$sys\WindowsPowerShell\v1.0\powershell.exe" -Enabled False

Write-Host "Regeln angelegt. Standardaktion wird in Phase 2 gesetzt."
```

Phase 2, scharf (nach der Audit-Woche, als Admin):

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Block
```

Audit-Prozedur in der Woche davor: Setze die Standardaktion einen Arbeitstag lang auf Block, arbeite normal, prüfe dann `C:\Windows\System32\LogFiles\Firewall\pfirewall.log` auf Zeilen mit `DROP` und `SEND`; für jeden Windows-Dienst, der darin steht und gebraucht wird (Name über die Prozess-ID und `Get-Process -Id <PID> -IncludeUserName` bzw. `tasklist /svc`), ergänze eine Dienstregel. Setze die Standardaktion danach bis zur Scharfschaltung zurück auf Allow. Die Dienstliste oben ist ein Startpunkt, kein Beleg; welche Dienste dein Windows braucht, zeigt nur das Protokoll (Prüfschritt P7). Die vordefinierten Regeln „Kernnetzwerk“ (DHCP, DNS, ICMP, IPv6) bleiben unverändert.

Loopback (Prüfschritt P4): In deinem Konto `python -m http.server 8000 --bind 127.0.0.1` starten, in Firefox `http://localhost:8000/` öffnen. Erwartet: die Verzeichnisliste. Falls nicht, als Admin:

```powershell
New-NetFirewallRule -DisplayName "Lockdown Allow Loopback v4" -Direction Outbound -Action Allow -RemoteAddress 127.0.0.1
New-NetFirewallRule -DisplayName "Lockdown Allow Loopback v6" -Direction Outbound -Action Allow -RemoteAddress ::1
```

Was offen bleibt: ein lokaler Dienst, den du selbst schreibst, erreicht Firefox über `localhost`, hat aber selbst keinen Internetzugang; er kann nur ausliefern, was schon auf dem Rechner liegt. Firefox-Proxy ist gesperrt (4.3).

Prüfschritte nach dem Scharfschalten (in deinem Konto):

```powershell
python -c "import urllib.request; urllib.request.urlopen('https://example.org')"   # erwartet: URLError
curl.exe -I https://example.org                                                     # erwartet: Fehler
Invoke-WebRequest https://example.org                                               # erwartet: Fehler
```

Firefox lädt eine freigegebene Seite; **Einstellungen** > **Windows Update** > **Nach Updates suchen** findet Updates (im Adminkonto).

### 8.2 Paketinstallation

Vor dem Scharfschalten installiert der Admin Python, `pip`, `python-lsp-server`, `debugpy`, das JDK, JDT LS, Kate und Git (Abschnitt 9). Danach gilt: `pip` hat kein Netz und schlägt mit einem Verbindungsfehler fehl. Pakete kommen über den Wheel-Ordner:

1. Als Admin anlegen:
   ```powershell
   New-Item -ItemType Directory "C:\Program Files\Wheels"
   New-Item -ItemType File "C:\Program Files\Wheels\Wunschliste.txt"
   icacls "C:\Program Files\Wheels\Wunschliste.txt" /grant "<DEIN-KONTO>:M"
   ```
   Der Ordner gehört zu `C:\Program Files` (nur Admin schreibt); nur die Wunschliste ist für dich beschreibbar.
2. Du schreibst Paketnamen in die Wunschliste. Im Wartungsfenster lädt der Admin (Wartungsregel „Wartung Python“ eingeschaltet, 11.2):
   ```powershell
   & "C:\Program Files\Python 3.15\python.exe" -m pip download -d "C:\Program Files\Wheels" <paket>
   ```
3. Du installierst in deine venv offline:
   ```powershell
   .\venv\Scripts\python.exe -m pip install --no-index --find-links "C:\Program Files\Wheels" <paket>
   ```
   Prüfschritt P21: `pip download` und `--no-index --find-links` gegen die pip-Dokumentation (`https://pip.pypa.io/en/stable/cli/pip_download/`, `.../pip_install/`) prüfen; die Seiten waren bei der Erstellung dieser Anleitung nicht abrufbar.

Java: Für Ordner mit `.java`-Dateien ohne Build-Werkzeug braucht JDT LS kein Netz („unsichtbares Projekt“, Jars in `lib/` werden erkannt) [Q71]. Maven- oder Gradle-Projekte brauchen Abhängigkeiten aus dem Internet; der Gradle-Wrapper lädt zudem Gradle selbst [Q165]. Wenn du ein Maven-Projekt brauchst: Der Admin führt im Wartungsfenster in deinem Projektordner `mvn dependency:go-offline` aus (Prüfschritt P35), danach arbeitet JDT LS mit `~/.m2`. Standard laut Kontext: keine Abhängigkeiten.

### 8.3 Ausführung: App Control for Business und AppLocker

Weg, den das schließt: Start von Programmen außerhalb der Systemordner (portable Browser, Installer aus `Downloads`, umbenannte Kopien, Benutzerinstallationen, `mshta.exe`, `wscript.exe`); je Benutzer zusätzlich Edge, Store und fremde Store-Apps. App Control ist auf Windows 11 Pro verfügbar [Q42]; AppLocker wird seit KB 5024351 auf allen Windows-11-Editionen durchgesetzt [Q20]. Microsoft empfiehlt App Control als Basis und AppLocker zur Feinsteuerung [Q43].

#### 8.3.1 App-Control-Richtlinie erstellen (als Admin)

Grundlage ist Microsofts Beispielrichtlinie „Default Windows“ (Windows-Komponenten, Store-Apps, WHQL-Treiber), ergänzt um Pfadregeln für die Programmordner. Pfadregeln gelten zur Laufzeit nur, wenn der Pfad ausschließlich für Admin-SIDs beschreibbar ist [Q45]; damit ist alles in Benutzerordnern ausgeschlossen, ohne dass es aufgezählt werden muss.

```powershell
# AppControl-Erstellen.ps1: als Administrator ausführen
$work = "C:\ProgramData\Lockdown\AppControl"
New-Item -ItemType Directory -Force $work | Out-Null
Copy-Item "C:\Windows\schemas\CodeIntegrity\ExamplePolicies\DefaultWindows_Audit.xml" "$work\Basis.xml"

# Pfadregeln für die Programmordner
$rules = @()
$rules += New-CIPolicyRule -FilePathRule 'C:\Program Files\*'
$rules += New-CIPolicyRule -FilePathRule 'C:\Program Files (x86)\*'

# venv-Starter von Python (Hash, unabhängig von Name und Ort; nach jedem Python-Update erneuern, 11.3)
$py = "C:\Program Files\Python 3.15"
foreach ($f in (Get-ChildItem $py -Recurse -Include venvlauncher.exe,venvwlauncher.exe)) {
  $rules += New-CIPolicyRule -DriverFilePath $f.FullName -Level Hash
}

Merge-CIPolicy -OutputFilePath "$work\Lockdown.xml" -PolicyPaths "$work\Basis.xml" -Rules $rules

# Mehrfach-Richtlinienformat, Name, Optionen
Set-CIPolicyIdInfo -FilePath "$work\Lockdown.xml" -ResetPolicyID -PolicyName "Lockdown"
Set-RuleOption -FilePath "$work\Lockdown.xml" -Option 0     # Enabled:UMCI (Benutzermodus prüfen)
Set-RuleOption -FilePath "$work\Lockdown.xml" -Option 3     # Enabled:Audit Mode (erst Protokoll)
Set-RuleOption -FilePath "$work\Lockdown.xml" -Option 6     # Enabled:Unsigned System Integrity Policy
Set-RuleOption -FilePath "$work\Lockdown.xml" -Option 16    # Enabled:Update Policy No Reboot

# Binärdatei nach Policy-GUID benennen und aktivieren
[xml]$x = Get-Content "$work\Lockdown.xml"
$guid = $x.SiPolicy.PolicyID
ConvertFrom-CIPolicy -XmlFilePath "$work\Lockdown.xml" -BinaryFilePath "$work\$guid.cip"
CiTool --update-policy "$work\$guid.cip"
CiTool --list-policies
```

Belege: Beispielrichtlinien und Pfad [Q187]; `New-CIPolicyRule -FilePathRule` und `-DriverFilePath -Level Hash` [Q188]; `Merge-CIPolicy -Rules` [Q189]; `Set-CIPolicyIdInfo -ResetPolicyID` [Q190]; `Set-RuleOption` und `Set-RuleOption -Help` für die Optionsliste [Q191]; Optionsbedeutungen 0, 3, 6, 16 [Q45]; `ConvertFrom-CIPolicy` [Q192]; Dateiname `{GUID}.cip` und `CiTool --update-policy` [Q47]. Prüfschritt P5: `Get-AuthenticodeSignature "$py\venvlauncher.exe"`; ist der Starter signiert, kann eine Herausgeberregel (`-Level Publisher`) die Hash-Regeln ersetzen und überlebt Python-Updates.

Audit-Woche: Arbeite in deinem Konto normal (Kate, Python, Java, Git, Firefox, Cold Turkey). Der Admin prüft täglich das Protokoll **Ereignisanzeige** > **Anwendungs- und Dienstprotokolle** > **Microsoft** > **Windows** > **CodeIntegrity** > **Operational** auf Einträge, die etwas Erlaubtes betreffen [Q193]; Skriptereignisse stehen unter **AppLocker** > **MSI and Script**. Jede nötige Datei bekommt eine Regel (`New-CIPolicyRule -DriverFilePath <Datei> -Level Publisher -Fallback Hash`, dann `Merge-CIPolicy` und erneut `ConvertFrom-CIPolicy` und `CiTool --update-policy`).

Erzwingen (Prüfschritt P8 bestanden, als Admin):

```powershell
$work = "C:\ProgramData\Lockdown\AppControl"
Set-RuleOption -FilePath "$work\Lockdown.xml" -Option 3 -Delete
[xml]$x = Get-Content "$work\Lockdown.xml"; $guid = $x.SiPolicy.PolicyID
ConvertFrom-CIPolicy -XmlFilePath "$work\Lockdown.xml" -BinaryFilePath "$work\$guid.cip"
CiTool --update-policy "$work\$guid.cip"
```

Hinweiskasten: Erzwinge die Richtlinie nie ohne Audit-Woche. Eine Pfadregel, die zur Laufzeit nicht greift (Ordner für Benutzer beschreibbar), blockiert alles darin, auch Firefox und Cold Turkey. Rückweg: `CiTool --remove-policy "{GUID}"` als Admin [Q47].

Skriptdurchsetzung bleibt an: PowerShell läuft für nicht erlaubte Skripte im Constrained Language Mode, `mshta.exe` und MSXML sind gesperrt, `wscript`/`cscript` ohne Regel ebenfalls; Python- und Java-Code wird nicht kontrolliert („unenlightened hosts“) [Q44]. Für deine eigenen `.ps1`-Skripte heißt das: Sie laufen im eingeschränkten Sprachmodus; wer PowerShell-Skripte in vollem Umfang braucht, legt sie in einen Admin-Ordner unter `C:\Program Files\Skripte` (Pfadregel gilt).

Installer im Wartungsfenster: Ein Installer in `Downloads` des Admins ist nicht erlaubt. Der Admin kopiert ihn nach `C:\ProgramData\Lockdown\Install` (Ordner vorher mit `icacls` nur für Administratoren beschreibbar machen; Prüfschritt: Pfadregel für diesen Ordner mit `New-CIPolicyRule -FilePathRule 'C:\ProgramData\Lockdown\Install\*'` ergänzen) oder schaltet für die Dauer der Installation mit Option 3 in den Audit-Modus und danach zurück.

#### 8.3.2 AppLocker je Benutzer (als Admin)

1. `sc.exe config appidsvc start=auto` ausführen; die Dienste-Konsole kann den Starttyp des Anwendungsidentitätsdiensts nicht mehr setzen [Q194].
2. `secpol.msc` > **Sicherheitseinstellungen** > **Anwendungssteuerungsrichtlinien** > **AppLocker**.
3. **Ausführbare Regeln** > Rechtsklick > **Standardregeln erstellen** (erlaubt Programme in `%windir%` und `%programfiles%` für Jeder und alles für Administratoren [Q21]).
4. **Ausführbare Regeln** > **Neue Regel erstellen**: **Verweigern**, Benutzer `<DEIN-KONTO>`, Bedingung **Herausgeber**, **Durchsuchen** auf `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` (falls Edge noch da ist; sonst Regel später anlegen, wenn Edge zurückkehrt), Schieberegler auf **Produktname**, Name „Verweigern Edge“ [Q195].
5. **Paket-App-Regeln** > **Standardregeln erstellen**, dann die Standardregel „Alle signierten Paket-Apps“ öffnen und über **Herausgeber** auf `CN=Microsoft Windows, O=Microsoft Corporation, L=Redmond, S=Washington, C=US` einschränken (Beispiel eines installierten Microsoft-Pakets wählen, Schieberegler auf **Herausgeber**); zweite Erlauben-Regel für das Kate-Paket (Herausgeber aus `Get-AppxPackage *Kate*`, Prüfschritt P17); Verweigern-Regel für Benutzer `<DEIN-KONTO>`, Herausgeber, Referenz „Microsoft Store“, Schieberegler auf **Paketname** [Q22].
6. Rechtsklick **AppLocker** > **Eigenschaften** > Reiter **Erzwingung**: bei **Ausführbare Regeln** und **Paket-App-Regeln** **Konfiguriert** mit **Regeln erzwingen** [Q196].
7. Prüfschritt in deinem Konto: Edge-Start (falls vorhanden) zeigt die AppLocker-Meldung; **Microsoft Store** startet nicht; Kate startet; Windows Terminal startet.

### 8.4 Virtuelle Umgebungen

`venv` legt unter Windows seit 3.7.2 keine Kopie von `python.exe` an, sondern ein Umleitungsprogramm (`venvlauncher.exe` als `python.exe`), das den Basisinterpreter aus `home` in `pyvenv.cfg` startet [Q25][Q64]. Deshalb: Netz wird dem Basisinterpreter verweigert (8.1, gilt), und der Starter im Benutzerordner braucht die Hash-Regel aus 8.3.1. Prüfschritt P5 und P6 (Windows-Teil):

```powershell
python -m venv C:\Users\<DEIN-KONTO>\Projekte\venv
C:\Users\<DEIN-KONTO>\Projekte\venv\Scripts\python.exe -c "import sys; print(sys.prefix, sys.base_prefix)"
C:\Users\<DEIN-KONTO>\Projekte\venv\Scripts\python.exe -c "import urllib.request; urllib.request.urlopen('https://example.org')"
```

Erwartet: erste Zeile zeigt venv und Basis; zweiter Aufruf scheitert mit `URLError`. Nach jedem Python-Update: Hash-Regeln erneuern (11.3).

### 8.5 Eingebettete Web-Engines

Pakete wie `pywebview`, `PyQt6-WebEngine`, `PySide6`, JavaFX WebView sind ohne Netz nicht installierbar (8.2). Kommen sie über den Wheel-Ordner, laufen ihre Renderer als Kindprozesse ohne Allow-Regel: Standardaktion Block. `pywebview` nutzt unter Windows WebView2 [Q40], das ebenfalls ohne Netz ist. Sie zeigen nur lokale Inhalte; das ist kein Weg ins Internet. Prüfschritt (nur wenn ein solches Paket installiert wird): Testfenster mit `https://example.org` bleibt leer.

### 8.6 Benutzerprofil

Programme aus deinem Konto dürfen das Firefox-Profil, die Kate-Konfiguration (`%LOCALAPPDATA%\katerc`, `%LOCALAPPDATA%\kate\lspclient\settings.json`) und Autostart-Einträge (Aufgabenplanung im eigenen Kontext [Q53], Startmenü-Autostart) ändern. Nichts davon ist sicherheitsrelevant: Firefox-Sperren liegen in `HKLM`, Erweiterungen sind erzwungen, Anwendungssteuerung und Firewall sind systemweit, Cold Turkey ist mit Passwort gesperrt. Ein Autostart-Eintrag startet nur, was erlaubt ist, und bekommt kein Netz.

### 8.7 Neubewertung der Sperrliste

Durch 8.1 und 8.3 entfallen aus Cold Turkey: alle Python-Prozesse, `cmd.exe`, `powershell.exe`, `wt.exe`, `winget.exe`, `DeploymentAgent.exe`, `regedit.exe`, `Taskmgr.exe`, `QuickAssist.exe`, Werkzeuge ohne Webinhalt und Systemkomponenten (6.8). Bleiben: Store, WSL (bis P10), WebView2-Hosts, Ablenkungs-Apps. Neu: Edge, `mshta.exe`, `hh.exe`.

## 9. Editor: Kate mit Python und Java

### 9.1 Rolle im Lockdown

Kate ist der Editor; Python und Java führen Code aus. Kate darf Befehle starten (Build, External Tools, Terminal) und Dateien über Netzprotokolle öffnen; das ist gewollt und abgesichert: Programme starten nur aus den Systemordnern (8.3), und kein Prozess außer Firefox, Cold Turkey und Windows-Diensten bekommt Netz (8.1). Was bleibt: Du kannst mit Python jede Datei auf dem Rechner lesen und schreiben, lokale Webserver betreiben und Programme schreiben, die lokale Inhalte anzeigen. Nichts davon holt neue Inhalte aus dem Internet. Das Debugger-Plugin von Kate ist nicht nach Windows portiert [Q156]; Python wird im Terminal mit `pdb` debuggt, Java mit `jdb`.

### 9.2 Kate installieren

Quelle: Microsoft Store, Paket „Kate“, Herausgeber KDE e.V., Store-ID 9NWMW7BB59HW [Q65][Q143]. Grund: signiertes Paket, automatische Updates, Installationsordner `C:\Program Files\WindowsApps` ist schreibgeschützt und vom System gesperrt [Q66]. Store-Apps werden je Benutzer installiert, deshalb in deinem Konto, bevor der Store für dich gesperrt wird (8.3.2 Schritt 5):

1. In deinem Konto: **Microsoft Store** öffnen, „Kate“ suchen, Herausgeber „KDE e.V.“ prüfen, **Installieren**. Aktuelle stabile Version: 26.08.2 vom 8. Oktober 2026 [Q141]; die Store-Version steht im Store nicht sichtbar, nach der Installation unter **Hilfe** > **Über Kate**.
2. Prüfschritt P17 als Admin: `Get-AppxPackage -AllUsers *Kate*` zeigt `Publisher` und `PackageFullName`; den Herausgeberstring brauchst du für die AppLocker-Paket-App-Regel (8.3.2).
3. Rückfall, falls der Store nicht genutzt werden soll: EXE-Installer `kate-release_26.08-12761-windows-cl-msvc2022-x86_64.exe` aus `https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/windows/` mit `.sha256` [Q144]; Installation als Admin mit Auswahl „alle Benutzer“ (Installer bietet die Auswahl über NSIS MultiUser [Q146][Q147]); Signatur nicht belegt, daher vor dem Start `Get-AuthenticodeSignature` prüfen.

### 9.3 Python installieren

Entscheidung E13: klassischer Installer für 3.15, systemweit. Der Installer ist seit 3.14 als veraltet markiert und wird für 3.16 nicht mehr erzeugt; die Laufzeiten des neuen Installationsmanagers sind nur je Benutzer installierbar [Q63]. Beim Wechsel auf 3.16 (11.3): `py install --target="C:\Program Files\Python 3.16"` als Admin, wie in der Python-Doku beschrieben [Q63].

1. Als Admin `python-3.15.2-amd64.exe` von `https://www.python.org/downloads/release/python-3152/` laden [Q150].
2. Stille Installation für alle Benutzer nach `C:\Program Files\Python 3.15` (Muster aus der Python-Doku [Q63]):
   ```powershell
   .\python-3.15.2-amd64.exe /quiet InstallAllUsers=1 PrependPath=1 Include_test=0
   ```
3. Language Server und Debugger systemweit (Netz noch offen oder Wartungsregel an):
   ```powershell
   & "C:\Program Files\Python 3.15\python.exe" -m pip install "python-lsp-server[all]" debugpy
   ```
   `python-lsp-server` 1.15.0 ist reines Python ohne Node.js [Q140][Q151]; `debugpy` ebenfalls [Q155]. Nicht gewählt: `pyright` (lädt Node.js und das npm-Paket zur Laufzeit nach [Q152]), `basedpyright` (bringt Node.js als Wheel mit, ein zweiter Interpreter [Q153]), `jedi-language-server` (ohne Node, aber weniger Plugins [Q154]).
4. App-Ausführungsaliase in deinem Konto abschalten (7.6). Prüfschritt: `Get-Command python,pylsp` zeigt Pfade unter `C:\Program Files\Python 3.15`.
5. Virtuelle Umgebungen nach 8.4; der Language Server läuft aus der Systeminstallation, die venv braucht ihn nicht.

### 9.4 Java installieren

JDT Language Server 1.63.0 verlangt Java 21 oder neuer; die Entwicklungsversion verlangt seit dem 30. Oktober 2026 Java 25 [Q69][Q158]. Gewählt: Eclipse Temurin JDK 25.0.5+8 LTS.

1. Als Admin `OpenJDK25U-jdk_x64_windows_hotspot_25.0.5_8.msi` von `https://adoptium.net/temurin/releases/` laden [Q159]; Prüfsumme auf der Downloadseite.
2. Installation je Maschine nach `C:\Program Files\Eclipse Adoptium\jdk-25.0.5.8-hotspot` mit PATH und `JAVA_HOME` [Q160]:
   ```powershell
   msiexec /i .\OpenJDK25U-jdk_x64_windows_hotspot_25.0.5_8.msi ADDLOCAL=FeatureMain,FeatureEnvironment,FeatureJavaHome /quiet
   ```
3. JDT LS: `jdt-language-server-1.63.0-202610191322.tar.gz` und `.sha256` von `https://download.eclipse.org/jdtls/milestones/1.63.0/` [Q157]; Hash vergleichen; als Admin entpacken nach `C:\Program Files\jdtls` (Windows 11 bringt `tar.exe` mit):
   ```powershell
   New-Item -ItemType Directory "C:\Program Files\jdtls"
   tar -xzf .\jdt-language-server-1.63.0-202610191322.tar.gz -C "C:\Program Files\jdtls"
   Get-ChildItem "C:\Program Files\jdtls\plugins\org.eclipse.equinox.launcher_*.jar"
   ```
   Den Dateinamen des Launchers notieren (Version im Namen); er steht in der LSP-Konfiguration (9.5).
4. Konfigurationskopie je Benutzer: JDT LS schreibt in den Ordner, der mit `-configuration` übergeben wird; der mitgelieferte `config_win` bleibt unverändert, wenn `-configuration` auf einen Benutzerordner zeigt [Q69][Q163]. Kate übergibt ihn im Startbefehl (9.5).
5. Netz: Für Ordner mit `.java`-Dateien ohne Build-Werkzeug braucht der Server kein Netz [Q71]; Maven und Gradle nur über das Wartungsfenster (8.2).

### 9.5 Code-Vervollständigung

Die deutschen Bezeichnungen in diesem Abschnitt folgen der deutschen Übersetzung von Kate; belegt sind die englischen Namen („Settings“ > „Configure Kate…“ > „Application“ > „Plugins“, Seite „LSP Client“, Feld „User Server Settings“, Optionen „Enable semantic highlighting“, „Show program diagnostics“, „Incrementally synchronize documents with the LSP server“) [Q166][Q178]. Weicht die deutsche Oberfläche ab, gilt der englische Name.

1. In Kate **Einstellungen** > **Kate einrichten…** > **Anwendung** > **Erweiterungen**: **LSP-Client** einschalten [Q166]. Die Seite **LSP-Client** erscheint im Dialog; dort **Semantische Hervorhebung aktivieren**, **Programmdiagnosen anzeigen**, **Dokumente inkrementell mit dem LSP-Server synchronisieren** eingeschaltet lassen (Standard) [Q178]. Keine KI-Vervollständigung: Kate 26.08 enthält kein Plugin mit KI-, Copilot- oder LLM-Funktion [Q174].
2. **Benutzereinstellungen des Servers** (Feld auf der LSP-Client-Seite; Datei `%LOCALAPPDATA%\kate\lspclient\settings.json` [Q167]). Kates Standardliste nennt `pylsp --check-parent-process` und das Kommando `jdtls` [Q164]; `jdtls` ist ein Python-Startskript [Q163], deshalb steht hier der `java`-Aufruf direkt. `<LAUNCHER>` durch den Dateinamen aus 9.4 Schritt 3 ersetzen, `<DEIN-KONTO>` einsetzen:

```json
{
    "servers": {
        "python": {
            "command": ["pylsp", "--check-parent-process"],
            "url": "https://github.com/python-lsp/python-lsp-server",
            "highlightingModeRegex": "^Python$"
        },
        "java": {
            "command": [
                "C:\\Program Files\\Eclipse Adoptium\\jdk-25.0.5.8-hotspot\\bin\\java.exe",
                "-Declipse.application=org.eclipse.jdt.ls.core.id1",
                "-Dosgi.bundles.defaultStartLevel=4",
                "-Declipse.product=org.eclipse.jdt.ls.core.product",
                "-Dlog.level=ALL",
                "-Xmx1G",
                "--add-modules=ALL-SYSTEM",
                "--add-opens", "java.base/java.util=ALL-UNNAMED",
                "--add-opens", "java.base/java.lang=ALL-UNNAMED",
                "-jar", "C:\\Program Files\\jdtls\\plugins\\<LAUNCHER>",
                "-configuration", "C:\\Users\\<DEIN-KONTO>\\AppData\\Local\\jdtls\\config_win",
                "-data", "C:\\Users\\<DEIN-KONTO>\\AppData\\Local\\jdtls\\workspace"
            ],
            "url": "https://github.com/eclipse-jdtls/eclipse.jdt.ls",
            "highlightingModeRegex": "^Java$"
        }
    }
}
```

   Der `java`-Aufruf entspricht dem Startbefehl der JDT-LS-README [Q69]; `-configuration` zeigt auf eine Kopie von `config_win` im Benutzerordner. Vorher einmal in deinem Konto:
   ```powershell
   New-Item -ItemType Directory "$env:LOCALAPPDATA\jdtls" -Force
   Copy-Item "C:\Program Files\jdtls\config_win" "$env:LOCALAPPDATA\jdtls\config_win" -Recurse
   ```
3. Beim ersten Öffnen einer Python- oder Java-Datei fragt Kate, ob der Server gestartet werden darf („LSP server start requested“ mit der Kommandozeile) [Q167]; **Ja**. Die Entscheidung landet in `katerc`, Gruppe `[lspclient]`, `AllowedServerCommandLines`.
4. Prüfschritt P20 (Nutzbarkeitstest, 10.2): Python-Datei mit `import os` und `os.pa` zeigt Vervollständigung; eine Zeile mit Syntaxfehler zeigt eine Diagnose; **Strg+Klick** auf einen Funktionsnamen springt zur Definition. Java-Datei im Ordner `Projekte\java\Hallo\Hallo.java` mit `System.out.pr` zeigt Vervollständigung; fehlendes Semikolon zeigt eine Diagnose; Sprung zur Definition einer eigenen Methode funktioniert. Der erste Java-Start dauert, bis der Server den Ordner als Projekt angelegt hat.

### 9.6 Plugins

Freigegeben (alle unter **Erweiterungen** einschaltbar): LSP-Client, Erstellen (Build), Externe Werkzeuge, Terminal (Konsole-Teil; im Windows-Paket enthalten, Lauffähigkeit Prüfschritt P19 [Q169]; sonst Windows Terminal nebenbei), Projekte (Git), Git-Blame, Dokumentvorschau (nutzt `markdownpart` ohne Web-Engine [Q172][Q173]), Suchen und Ersetzen, Dateisystem-Browser, Symbolansicht, Snippets, Textfilter, SQL. Keines öffnet einen Weg, den 8.1 und 8.3 nicht schließen (Dokument 1, Abschnitt 7.5). Links aus Hilfetexten und Language-Server-Hinweisen öffnen im Standardbrowser, also in Firefox ESR (7.5), und landen auf der Sperrseite, wenn die Seite nicht freigegeben ist.

**Datei** > **Öffnen** nimmt auch Adressen (`http://`) entgegen; der KIO-Worker hat kein Netz (8.1). Prüfschritt P18: Während eines Öffnungsversuchs einer `http://`-Adresse zeigt der Task-Manager den Prozess `kioworker.exe`; der Versuch endet mit einem Fehler.

### 9.7 Einstellungen sperren

Nicht nötig: Kein Kate-Plugin schließt einen Weg, der nicht systemweit geschlossen ist. Deshalb werden weder Kiosk-Einträge gesetzt noch Konfigurationsdateien schreibgeschützt (das bräche Sitzungen und Fensterlayout). Wer trotzdem Terminal, Externe Werkzeuge und Projekt-Terminal sperren will: KConfig liest systemweite Dateien über Qt-Standardpfade, unter Windows `C:\ProgramData\kdeglobals`, und wertet `[$i]` plattformneutral aus (Qt-Pfadtabelle und KConfig-Quelltext [Q168][Q175][Q176]); ein Beleg aus der KDE-Doku für Windows fehlt. Option (Prüfschritt): Datei `C:\ProgramData\kdeglobals` als Admin mit

```ini
[KDE Action Restrictions][$i]
shell_access=false
```

anlegen; Terminal-, External-Tools- und Project-Plugin beachten `shell_access`, Build-, Textfilter- und LSP-Plugin nicht [Q177]. Erwartung: Das Terminal-Plugin zeigt keine Ansicht mehr.

### 9.8 Cold Turkey

Freigegeben (nicht sperren): `kate.exe`, `kioworker.exe`, `python.exe`, `pythonw.exe`, `py.exe`, `pylsp.exe`, `java.exe`, `javaw.exe`, `cmd.exe`, `powershell.exe`, `pwsh.exe`, `wt.exe`, `conhost.exe`, `git.exe`, `bash.exe`, `sh.exe`, `git-bash.exe` (6.4).

### 9.9 Git

Git for Windows 2.56.0(4): `Git-2.56.0.4-64-bit.exe` von `https://git-scm.com/install/windows` (Release-Notes mit SHA-256 [Q180][Q181]); als Admin installieren, Standardziel `C:\Program Files\Git` [Q182]. Das Paket enthält Bash, cURL 8.22.0 und OpenSSH [Q181]; alle ohne Netz (8.1). Entscheidung E16: Git lokal; `git push` zu einem Remote scheitert. Kate-Projekt-Plugin nutzt `git` aus dem Pfad [Q184].

### 9.10 Prüfschritte

Ausbruchsprüfung (10.1, Zeilen K1 bis K9) und Nutzbarkeitstest (10.2).

## 10. Ausbruchsprüfung und Nutzbarkeitstest

Vor dem Scharfschalten: Cold-Turkey-Blöcke eingeschaltet, aber nicht gesperrt; Firewall-Standardaktion Block; App Control erzwungen; AppLocker erzwungen. Jede Zeile wird in deinem Konto ausgeführt, das tatsächliche Ergebnis eingetragen. Scharfgeschaltet wird nur, wenn jede Zeile „bestanden“ oder „Restrisiko benannt“ trägt.

### 10.1 Protokoll

| Nr. | Weg | Prüfschritt | Erwartet | Tatsächlich |
|---|---|---|---|---|
| F1 | Profil-Flyout, `about:profiles`, `about:profilemanager` | Adressen eingeben; Avatar-Schaltfläche suchen | Sperrseite; keine Profilverwaltung | |
| F2 | `firefox.exe -P` | Ausführen > `"C:\Program Files\Mozilla Firefox\firefox.exe" -P` | Profilmanager erscheint; ein neues Profil hat dieselben Sperren (F3 bis F7 im neuen Profil wiederholen); Cold-Turkey-Erweiterung ist da | |
| F3 | `about:config` | Adresse eingeben | Sperrseite | |
| F4 | `about:support` | Adresse eingeben | Sperrseite | |
| F5 | `about:addons`, Erweiterungsmenü | Adresse eingeben; Erweiterungsmenü öffnen | Sperrseite; keine Schaltfläche **Entfernen** oder **Deaktivieren** | |
| F6 | Fehlerbehebungsmodus | **Hilfe** > **Fehlerbehebungsmodus**; `firefox.exe -safe-mode`; Umschalttaste beim Start | kein Menüpunkt; normaler Start | |
| F7 | Privates Fenster | **Strg+Umschalt+P**; Menü | kein privates Fenster | |
| F8 | Entwicklerwerkzeuge | **F12**, **Strg+Umschalt+I**, `about:debugging`, Rechtsklick **Untersuchen** | nichts; Sperrseite; kein Menüpunkt | |
| F9 | `view-source:` | `view-source:https://docs.python.org/3/` und `view-source:https://example.org/` | Quelltext der freigegebenen Seite; Sperrseite für die andere | |
| F10 | Reader-Ansicht | auf freigegebener Seite Reader-Ansicht öffnen | nur Inhalt der freigegebenen Seite | |
| F11 | `file://` | `file:///C:/Windows/win.ini` | Datei wird angezeigt (gewollt, E15) | |
| F12 | `data:`-URL | `data:text/html,<a href="https://example.org">x</a>` eingeben, Link klicken | Seite zeigt den Link; Klick führt zur Sperrseite | |
| F13 | PDF | PDF von docs.python.org öffnen | wird angezeigt | |
| F14 | Download starten | ZIP von einer freigegebenen Seite laden | landet in `Downloads`, kein Dialog | |
| F16 | Erweiterung aus Datei | XPI per Drag-and-drop ins Fenster | Meldung „vom Administrator blockiert“ | |
| F17 | Proxy | `about:preferences` > Netzwerk-Einstellungen | **Einstellungen…** ausgegraut | |
| F18 | Firefox-Konto | Menü, `about:preferences#sync` | kein Konto-Eintrag | |
| F20 | Firefox-Update | **Hilfe** > **Über Firefox** | Update läuft ohne Adminabfrage | |
| F21 | Zweite Firefox-Installation | Firefox-Installer (EXE) laden und starten; portable Firefox-ZIP entpacken und `firefox.exe` starten | Installer: App Control blockiert den Start (Ereignis im CodeIntegrity-Protokoll); portable: blockiert | |
| F22 | `about:preferences`, `about:downloads`, `about:policies` | Adressen eingeben | erreichbar, nichts Sicherheitsrelevantes änderbar | |
| F23 | Eigene Richtlinie in `HKCU` | `reg add HKCU\SOFTWARE\Policies\Mozilla\Firefox /v BlockAboutConfig /t REG_DWORD /d 0 /f`, Firefox neu starten | `about:config` bleibt gesperrt (HKLM hat Vorrang) | |
| B1 | Adminrechte | PowerShell **Als Administrator ausführen** | Anmeldeaufforderung verlangt `<ADMIN>` | |
| B2 | Edge | Startmenü „Edge“; `microsoft-edge:https://example.org` in Ausführen | nicht vorhanden oder AppLocker-Meldung bzw. Cold Turkey beendet | |
| B3 | Portable Browser | ein portabler Chromium-Browser aus `Downloads` | App Control blockiert | |
| B4 | WebView2 | Widgets-Taste **Win+W**; Hilfe anfordern | Cold Turkey beendet die App; Firewall-Protokoll zeigt DROP für `msedgewebview2.exe` | |
| B4a | `mshta.exe`, `hh.exe` | `mshta https://example.org`, `hh https://example.org` | App Control blockiert `mshta`; `hh` zeigt leeres Fenster, Firewall DROP | |
| B5 | Programm vom USB-Stick | `notepad++-portable.exe` o. ä. vom Stick starten | App Control blockiert | |
| B7 | Systemzeit | **Einstellungen** > **Zeit und Sprache** > **Datum und Uhrzeit** > manuell setzen; `Set-Date` in PowerShell | Schalter inaktiv oder Adminabfrage; `Set-Date` verweigert; Cold Turkey schließt die Einstellungen-App | |
| B8 | Neues Konto | **Einstellungen** > **Konten** > **Andere Benutzer** > **Konto hinzufügen** | Adminabfrage | |
| B9 | Aufgabenplanung | Aufgabe anlegen, die `cmd.exe /c start https://example.org` startet | Aufgabe läuft, Firefox zeigt die Sperrseite | |
| B10 | winget, Store | `winget install Mozilla.Firefox`; **Microsoft Store** öffnen | Installation nach Benutzerordner gelingt, Start wird blockiert; Store: AppLocker-Meldung | |
| B11 | Downloads per Shell | `curl.exe -O https://example.org/index.html`; `Invoke-WebRequest`; `certutil -urlcache -split -f https://example.org x.txt`; `bitsadmin /transfer t https://example.org/index.html %TEMP%\x.html` | alle scheitern | |
| B12 | Registry | `reg add HKLM\SOFTWARE\Policies\Mozilla\Firefox /v BlockAboutConfig /t REG_DWORD /d 0 /f` | Zugriff verweigert | |
| B13 | Abgesicherter Modus | **Umschalt+Neustart** > **Problembehandlung** > **Erweiterte Optionen** > **Starteinstellungen** > **Neu starten** > **4** (Prüfschritt P15; mit BitLocker-PIN des Admins) | im abgesicherten Modus: Cold Turkey läuft (Task-Manager, Prozess `Cold Turkey Blocker.exe`) oder nicht; Firefox-Sperren gelten in jedem Fall; Ergebnis dokumentieren | |
| B14 | WinRE | **Umschalt+Neustart** > **Problembehandlung** > **Eingabeaufforderung** (Prüfschritt P14) | ohne PIN oder Wiederherstellungsschlüssel kein Zugriff auf `C:` | |
| B16 | WSL | `wsl --install` (Prüfschritt P10) | Cold Turkey beendet `wsl.exe`; ohne Sperre: Adminabfrage | |
| I1 | Python mit Netz | `python -c "import urllib.request; urllib.request.urlopen('https://example.org')"`; Java: `jshell` mit `new java.net.URL("https://example.org").openStream()` | beide scheitern | |
| I2 | Lokaler Webserver | `python -m http.server 8000 --bind 127.0.0.1`; `http://localhost:8000/` in Firefox (P4) | erreichbar; Inhalt nur aus dem lokalen Ordner | |
| I3 | pip | `python -m pip install requests` | Verbindungsfehler; offline aus Wheel-Ordner gelingt | |
| I4 | venv | 8.4 | Starter läuft, kein Netz | |
| I5 | Kopie eines gesperrten Programms | `copy "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" %USERPROFILE%\x.exe` und starten (falls Edge noch da) | App Control blockiert | |
| I6 | Kindprozess | `python -c "import subprocess; subprocess.run([r'C:\Users\<DEIN-KONTO>\Downloads\firefox-portable\firefox.exe'])"` | blockiert | |
| I9 | Autostart | `.cmd` in `shell:startup`, das `python -c "..."` mit Netz startet; neu anmelden | läuft, scheitert am Netz | |
| I10 | Aliase | `python` in neuer PowerShell | `C:\Program Files\Python 3.15\python.exe`, kein Store | |
| T1 | Cold-Turkey-Block ausschalten | Toggle des Blocks (vor der Sperre: geht; nach der Sperre wiederholen) | nach Sperre: Passwortabfrage | |
| T2 | Cold Turkey deinstallieren | **Installierte Apps** > Cold Turkey > **Deinstallieren** | Adminabfrage; mit gesperrtem Block scheitert der Uninstaller | |
| T3 | Dienst beenden | Task-Manager > **Dienste** > Cold-Turkey-Dienst > **Beenden** | Zugriff verweigert | |
| T4 | Erweiterung beenden | Firefox **Weitere Werkzeuge** > **Task-Manager** > Cold Turkey beenden | Firefox wird nach kurzer Zeit geschlossen („Block browsers if extension crashes“) | |
| T5 | Browser ohne Erweiterung | neues Profil (`-P`) ohne Erweiterung erzwingen ist nicht möglich; stattdessen Firefox-Beta portable (F21) | blockiert | |
| T6 | Ausnahmen, neuer Tab (P2) | **Strg+T** | Tab bleibt offen; sonst `about:blank` als Ausnahme ergänzen | |
| T7 | Umbenennen | `ren msedge.exe x.exe` im Edge-Ordner | Zugriff verweigert | |
| T10 | Zweites Konto | wie B8 | Adminabfrage | |
| K1 | Kate Build | **Erstellen**-Plugin: Ziel `curl.exe https://example.org` | Befehl läuft, Verbindung scheitert | |
| K2 | Kate Terminal | Terminal-Ansicht: `curl.exe https://example.org` | scheitert | |
| K3 | Kate öffnet Adresse | **Datei** > **Öffnen**, `https://example.org/index.html` eingeben (P18) | Fehler; `kioworker.exe` im Task-Manager | |
| K4 | Vorschau | Markdown mit `<iframe src="https://example.org">` | kein Inhalt aus dem Netz | |
| K5 | Plugin einschalten | External Tools: Werkzeug `cmd /c start https://example.org` | Firefox zeigt die Sperrseite | |
| K7 | Portable Kate | Kate-7z aus KDE-CDN in `Downloads` entpacken, starten | App Control blockiert | |
| K8 | LSP fremder Server | `settings.json`: `"command": ["C:\\Users\\<DEIN-KONTO>\\x.exe"]` | Kate fragt; Start blockiert | |

### 10.2 Nutzbarkeitstest

| Aufgabe | Schritte | Erwartet | Tatsächlich |
|---|---|---|---|
| Python schreiben, ausführen, debuggen | `Projekte\hallo.py` mit einer Funktion und `breakpoint()`; in Kate Vervollständigung, Diagnose, Sprung zur Definition (9.5); im Terminal `python hallo.py`, `pdb` bedienen (`n`, `p`, `c`) | alles funktioniert | |
| Python mit venv und Wheel | venv anlegen, Paket aus dem Wheel-Ordner installieren, importieren | funktioniert | |
| Java schreiben, ausführen, debuggen | `Projekte\java\Hallo\Hallo.java`; Vervollständigung, Diagnose, Sprung; `javac Hallo.java`, `java Hallo`; `jdb Hallo` mit `stop in Hallo.main`, `run`, `next` | alles funktioniert | |
| Freigegebene Seiten | jede Zeile der Freigabeliste öffnen | lädt | |
| Dokumentation lesen | Python-Doku als PDF öffnen, Java-Doku, Kate-Handbuch | lesbar | |
| Git | `git init`, `git add`, `git commit` im Projektordner; Kate Projekt-Plugin zeigt Status | funktioniert | |
| Alltag | Datei drucken, Bluetooth-Gerät koppeln, Lautstärke, Bildschirmhelligkeit, Taschenrechner, Notepad, Paint, Snipping Tool | funktioniert | |
| Windows Update | als Admin **Nach Updates suchen** | findet Updates | |

### 10.3 Scharfschalten

Nach bestandener Prüfung: 6.7 (Cold-Turkey-Passwörter), 8.3.1 (App Control erzwungen, falls noch Audit), 8.1 Phase 2 (Firewall Block), 7.2 (BitLocker aktiv). Danach die Zeilen T1 und T2 wiederholen.

## 11. Betrieb

### 11.1 Updates

| Was | Wie | Wer | Wann |
|---|---|---|---|
| Windows | automatisch (7.4) | Windows | laufend |
| Firefox ESR | Maintenance Service, `AppAutoUpdate` | Firefox | laufend; bei Wechsel der ESR-Linie (nächste Linie laut Mozilla-Kalender ESR 175 ab Mitte 2027 [Q1]) prüft der Admin `about:policies` auf Fehler und diese Anleitung gegen die Richtlinienreferenz |
| Kate | Store-Hintergrundupdate (P16) | Store | laufend |
| Cold Turkey | **Block the installer** ist an; im Wartungsfenster Block entsperren, Installer von `getcoldturkey.com` ausführen, Block wieder sperren | Admin | bei neuer Version |
| Python | neuer Installer „Für alle Benutzer“; danach `pip install --upgrade "python-lsp-server[all]" debugpy` und Hash-Regeln (11.3) | Admin | bei neuer Version |
| JDK, JDT LS | neues MSI; neues Tarball nach `C:\Program Files\jdtls`, `<LAUNCHER>`-Name in `settings.json` prüfen | Admin | halbjährlich oder bei Bedarf |
| Git | neuer Installer | Admin | bei Bedarf |

### 11.2 Wartungsfenster

Termin, nicht Zustand. Der Admin meldet sich an, schaltet die nötigen Wartungsregeln ein, erledigt die Liste, schaltet sie aus, meldet sich ab:

```powershell
Enable-NetFirewallRule -DisplayName "Wartung Python","Wartung PowerShell"
# Arbeiten: pip download, Installer, Updates
Disable-NetFirewallRule -DisplayName "Wartung Python","Wartung PowerShell"
Get-NetFirewallRule -DisplayName "Wartung*" | Select DisplayName,Enabled
```

Pakete: Wunschliste aus `C:\Program Files\Wheels\Wunschliste.txt` abarbeiten (8.2).

### 11.3 Nach einem Python-Update

Hash-Regeln für `venvlauncher.exe` und `venvwlauncher.exe` erneuern: Block „venv-Starter“ aus `AppControl-Erstellen.ps1` ausführen (Regeln erzeugen, `Merge-CIPolicy` mit der bestehenden `Lockdown.xml` als Basis, `ConvertFrom-CIPolicy`, `CiTool --update-policy`). Firewall-Regeln mit dem Pfad `C:\Program Files\Python 3.15\...` auf die neue Version anpassen.

### 11.4 Nach einem Windows-Funktionsupdate

Prüfen: `CiTool --list-policies` zeigt die Richtlinie als erzwungen; `Get-NetFirewallProfile | Select Name,DefaultOutboundAction` zeigt Block; Edge nicht zurückgekehrt (**Installierte Apps**); App-Ausführungsaliase noch aus; `Get-Service appidsvc` läuft; `about:policies` ohne Fehler; Zeilen B2, B4, B10, I1 aus 10.1.

## 12. Rückbau

Nur durch den Admin, in dieser Reihenfolge:

1. Cold Turkey: Blöcke mit Passwort entsperren und ausschalten; **Password Settings** aus; optional deinstallieren.
2. Firewall: `Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Allow`; `Get-NetFirewallRule -DisplayName "Lockdown*","Wartung*" | Remove-NetFirewallRule`.
3. App Control: `CiTool --remove-policy "{GUID}"` (GUID aus `CiTool --list-policies`); Neustart.
4. AppLocker: `secpol.msc` > AppLocker > **Eigenschaften** > Erzwingung bei beiden Sammlungen auf **Nicht konfiguriert**; Regeln löschen; `sc.exe config appidsvc start=demand`.
5. Firefox: `Remove-Item -Recurse "HKLM:\SOFTWARE\Policies\Mozilla\Firefox"`; Firefox neu starten, `about:policies` leer.
6. Konten: `Add-LocalGroupMember -SID "S-1-5-32-544" -Member "<DEIN-KONTO>"`, falls du wieder Admin sein sollst.
7. BitLocker bleibt (Schutz ohne Nachteil); wer ihn entfernen will: `Disable-BitLocker -MountPoint "C:"`.
8. Windows-Update-Richtlinie in `gpedit.msc` auf **Nicht konfiguriert**, falls gewünscht.

Nötig: Adminkennwort, Cold-Turkey-Passwörter, BitLocker-PIN. Ohne den Umschlag beim Admin gibt es keinen Rückbau; das ist der Sinn der Anordnung.

## 13. Quellen

Alle Quellen am 31.10.2026 abgerufen. Quelltextangaben beziehen sich auf den GitHub-Spiegel mozilla-firefox/firefox (Branch esr153) und die GitHub-Spiegel der KDE-Repositories (master und release/26.08). Seiten unter firefox-admin-docs.mozilla.org leiten auf mozilla.github.io/enterprise-admin-reference um.

- Q1 https://whattrainisitnow.com/release/?version=esr
- Q4 https://mozilla.github.io/enterprise-admin-reference/reference/policies/
- Q5 https://mozilla.github.io/enterprise-admin-reference/reference/policies/preferences/
- Q6 https://mozilla.github.io/enterprise-admin-reference/reference/policies/disablesafemode/
- Q7 https://mozilla.github.io/enterprise-admin-reference/reference/policies/privatebrowsingmodeavailability/
- Q8 https://mozilla.github.io/enterprise-admin-reference/reference/policies/websitefilter/
- Q9 https://mozilla.github.io/enterprise-admin-reference/reference/policies/extensionsettings/
- Q10 https://mozilla.github.io/enterprise-admin-reference/reference/policies/blockaboutprofiles/
- Q11 https://mozilla.github.io/enterprise-admin-reference/reference/policies/proxy/
- Q13 https://bugzilla.mozilla.org/show_bug.cgi?id=2021356
- Q16 https://getcoldturkey.com/support/user-guide/
- Q18 https://getcoldturkey.com/support/system-requirements/
- Q20 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/requirements-to-use-applocker
- Q21 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/executable-rules-in-applocker
- Q22 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/packaged-apps-and-packaged-app-installer-rules-in-applocker
- Q24 https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule
- Q25 https://docs.python.org/3/library/venv.html
- Q30 https://getcoldturkey.com/news/ (Release Notes)
- Q31 https://getcoldturkey.com/features/
- Q32 https://getcoldturkey.com/pricing/
- Q34 https://getcoldturkey.com/support/extensions/firefox/
- Q38 https://blogs.windows.com/windows-insider/2023/11/16/previewing-changes-in-windows-to-comply-with-the-digital-markets-act-in-the-european-economic-area/
- Q39 https://learn.microsoft.com/en-us/deployedge/webview2-enterprise
- Q40 https://learn.microsoft.com/microsoft-edge/webview2/concepts/end-user-faq
- Q41 https://learn.microsoft.com/en-us/deployedge/microsoft-edge-webview-policies
- Q42 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol
- Q43 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview
- Q44 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/script-enforcement
- Q45 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/select-types-of-rules-to-create
- Q47 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/deploy-appcontrol-policies-with-script und https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/operations/citool-commands
- Q48 https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-update
- Q49 https://learn.microsoft.com/en-us/windows/deployment/update/waas-wu-settings
- Q50 https://learn.microsoft.com/en-us/defender-endpoint/manage-protection-updates-microsoft-defender-antivirus
- Q51 https://learn.microsoft.com/windows/client-management/mdm/policy-csp-admx-controlpanel
- Q52 https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-ctrlaltdel
- Q53 https://learn.microsoft.com/en-us/windows/win32/TaskSchd/security-contexts-for-running-tasks
- Q54 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/how-it-works
- Q55 https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/local-accounts
- Q57 https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/countermeasures
- Q58 https://support.mozilla.org/en-US/kb/deploy-firefox-msi-installers
- Q59 https://support.mozilla.org/en-US/kb/what-mozilla-maintenance-service
- Q63 https://docs.python.org/3/using/windows.html
- Q64 https://mail.python.org/pipermail/python-checkins/2018-December/158634.html
- Q65 https://kate-editor.org/get-it/
- Q66 https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-behind-the-scenes
- Q67 https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules
- Q68 https://learn.microsoft.com/en-us/powershell/module/netsecurity/set-netfirewallprofile
- Q69 https://github.com/eclipse-jdtls/eclipse.jdt.ls (README; Stand Tag v1.63.0: https://raw.githubusercontent.com/eclipse-jdtls/eclipse.jdt.ls/v1.63.0/README.md)
- Q71 https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/master/CHANGELOG.md
- Q72 https://www.eclipse.org/forums/index.php/t/1066859/
- Q73 https://docs.oracle.com/en/java/javase/21/install/installation-jdk-microsoft-windows-platforms.html
- Q74 https://learn.microsoft.com/de-de/windows/apps/desktop/modernize/desktop-to-uwp-extensions
- Q78 https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/change-the-system-time
- Q79 https://learn.microsoft.com/en-my/troubleshoot/microsoft-edge/development/webview2-unexpected-install-windows-server
- Q84 Firefox-Quelltext esr153: browser/components/profiles/SelectableProfileService.sys.mjs
- Q85 Firefox-Quelltext esr153: browser/components/enterprisepolicies/Policies.sys.mjs
- Q86 Firefox-Quelltext esr153: toolkit/components/enterprisepolicies/EnterprisePoliciesParent.sys.mjs
- Q88 Firefox-Quelltext esr153: toolkit/xre/SafeMode.h und toolkit/xre/PolicyChecks.h
- Q97 https://www.firefox.com/en-US/download/all/desktop-esr/
- Q98 https://support.mozilla.org/en-US/questions/1197118 (Community)
- Q99 https://support.mozilla.org/en-US/questions/1415779 (Community)
- Q137 Firefox-Quelltext esr153: devtools/startup/DevToolsStartup.sys.mjs
- Q140 https://github.com/python-lsp/python-lsp-server
- Q141 https://kde.org/announcements/gear/26.08.2/
- Q143 https://kate-editor.org/post/2025/2025-06-03-kate-and-co-in-the-microsoft-store/
- Q144 https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/windows/
- Q146 Craft-Quelltext: bin/Packager/Nsis/NullsoftInstaller.nsi (Repository packaging/craft)
- Q147 https://nsis.sourceforge.io/Docs/MultiUser/Readme.html
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
- Q160 https://adoptium.net/installation/windows/
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
- Q180 https://git-scm.com/install/windows und https://github.com/git-for-windows/git/releases/latest
- Q181 https://github.com/git-for-windows/git/releases/tag/v2.56.0.windows.1
- Q182 https://github.com/git-for-windows/build-extra/blob/main/installer/install.iss
- Q184 Kate-Quelltext: apps/lib/gitprocess.cpp
- Q185 https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser
- Q186 https://learn.microsoft.com/en-us/powershell/module/bitlocker/enable-bitlocker
- Q187 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/example-appcontrol-base-policies
- Q188 https://learn.microsoft.com/en-us/powershell/module/configci/new-cipolicyrule
- Q189 https://learn.microsoft.com/en-us/powershell/module/configci/merge-cipolicy
- Q190 https://learn.microsoft.com/en-us/powershell/module/configci/set-cipolicyidinfo
- Q191 https://learn.microsoft.com/en-us/powershell/module/configci/set-ruleoption
- Q192 https://learn.microsoft.com/en-us/powershell/module/configci/convertfrom-cipolicy
- Q193 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/audit-appcontrol-policies
- Q194 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/configure-the-application-identity-service
- Q195 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/create-a-rule-that-uses-a-publisher-condition
- Q196 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/configure-an-applocker-policy-for-enforce-rules
- Q204 https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers
- Q205 https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/add-localgroupmember
- Q206 https://learn.microsoft.com/en-us/windows/wsl/install
