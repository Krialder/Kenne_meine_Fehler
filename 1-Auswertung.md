# Dokument 1: Auswertung

Lockdown mit Firefox ESR, Cold Turkey Pro und Kate auf Windows 11 Pro sowie macOS 14 und 15. Stand der Prüfung: 31. Oktober 2026. Alle Belege tragen dieses Abrufdatum; die Quellenliste steht in Abschnitt 13.

## 0. Was geprüft wurde und unter welchen Annahmen

Geprüft wurden drei Quellen:

1. Der Chatverlauf (Gemini), in dem ein Firefox-ESR-Lockdown mit Cold Turkey Pro Stück für Stück versucht wurde.
2. Die Dokumentation „System Lockdown & Security Protocol, Windows 11, Microsoft Edge & Cold Turkey Pro“ (drei Seiten, im Folgenden Edge-Dokumentation).
3. Die Cold-Turkey-Konfiguration. Screenshots lagen nicht bei; als Ist-Stand gilt deshalb die Edge-Dokumentation. Alles, was dort nicht steht (Blocklisten-Namen, Website-Liste, Ausnahmen, Zeitpläne, Sperrart, Einstellungsseite), steht in Dokument 2 und 3 als Prüfschritt.

Annahmen aus dem Kontext, soweit du Standardwerte freigegeben hast:

| Punkt | Angenommener Wert |
|---|---|
| Firefox-ESR-Linie | 153 ESR, erste Version 153.0esr vom 21.07.2026, aktuelle Version 153.6.0esr vom 27.10.2026, nächste 153.7 am 10.11.2026. Die Linie 140 ESR endete mit 140.17.0 am 29.09.2026 [Q1][Q2][Q80][Q81][Q82]. Firefox selbst erscheint seit Version 155 alle zwei Wochen, ESR-Punktversionen folgen diesem Takt [Q29]. |
| Python | aktuelle stabile 3.x aus python.org, nur Standardbibliothek, weitere Pakete im Wartungsfenster, virtuelle Umgebungen ja, kein Internetzugang eigener Programme |
| Java | JDK in der Version, die der Eclipse JDT Language Server verlangt, keine weiteren Abhängigkeiten |
| Kate | aktuelle stabile Version, Git lokal |
| Freigegebene Webseiten | wie in Cold Turkey hinterlegt, dazu die offizielle Dokumentation von Python (docs.python.org), Java (docs.oracle.com/en/java, dev.java) und Kate (docs.kde.org, kate-editor.org) |
| macOS-Hardware | unbekannt; keine der geprüften Maßnahmen unterscheidet nach Intel oder Apple Silicon, mit einer Ausnahme (Wiederherstellung, siehe Matrix Zeile B14). Dort steht ein Prüfschritt. |
| Adminkonto | Variante A (Standardkonto, Adminkennwort bei der Vertrauensperson) ist die Empfehlung; Variante B (du behältst Adminrechte) wird in Abschnitt 10 je Maßnahme mit Restrisiko bewertet |

Was in dieser Auswertung nicht belegt werden konnte, steht als solches markiert und wird in den Anleitungen als Prüfschritt für dich geführt, nicht als Anweisung.

## 1. Der Kernirrtum des Chatverlaufs: Installation gegen Profil

Der Chatverlauf dreht sich fast vollständig um das Firefox-Profil-Menü. Er vermischt dabei zwei Ebenen, die Firefox strikt trennt:

**Je Installation** gelten Enterprise-Richtlinien. Unter Windows liest Firefox sie aus dem Registry-Zweig `HKLM\SOFTWARE\Policies\Mozilla\Firefox` (Gruppenrichtlinie oder importierte Registry-Datei) oder aus `policies.json` im Ordner `distribution` neben `firefox.exe`; unter macOS aus einem Konfigurationsprofil oder `/Library/Preferences/org.mozilla.firefox.plist` (nur mit `EnterprisePoliciesEnabled`) oder aus `policies.json` in `Firefox.app/Contents/Resources/distribution` [Q14][Q15]. Die ADMX-Vorlagen dafür liefert Mozilla mit den Richtlinienvorlagen [Q60]. Richtlinien wirken in jedem Profil, das diese Installation öffnet: erzwungene Erweiterungen (`ExtensionSettings` mit `force_installed`), gesperrte `about:`-Seiten, `WebsiteFilter`, `Proxy`, gesperrte `Preferences` [Q4][Q5][Q9]. Ein neues Profil ändert daran nichts.

**Je Profil** gelten nur Dateien im Profilordner: `user.js`, `prefs.js` und `chrome/userChrome.css`. Die Präferenz `toolkit.legacyUserProfileCustomizations.stylesheets` wird je Profil gelesen, und `userChrome.css` liegt im Profilordner [Q5]. Alles, was der Chatverlauf über `userChrome.css` und `user.js` erreichen wollte, verschwindet mit jedem neuen Profil und mit jedem Löschen der Datei durch ein Programm, das der Benutzer startet.

Daraus folgt:

- Ein Zweitprofil umgeht keine Richtlinie und keine erzwungene Erweiterung. Wer im Chatverlauf „mit Profilen umgehen“ konnte, hat damit `userChrome.css`/`user.js` umgangen oder ein Profil ohne die Cold-Turkey-Erweiterung geöffnet. Letzteres schließt Cold Turkey selbst: während eines gesperrten Blocks muss die Erweiterung aktiv sein, sonst wird der Browser nach der Grace Period (Standard 60 Sekunden) geschlossen [Q16]. Wird die Cold-Turkey-Erweiterung zusätzlich per `ExtensionSettings` erzwungen, ist sie in jedem Profil sofort da.
- Das Profil-Flyout (Avatar-Schaltfläche, Firefox 138 und neuer, Release Notes [Q83]) hängt an der Präferenz `browser.profiles.enabled` und an der Richtlinienfunktion `profileManagement`. Im Quellcode der ESR-153-Linie meldet der `SelectableProfileService` das Feature als abgeschaltet, sobald `BlockAboutProfiles` gesetzt ist, weil diese Richtlinie die Funktion `profileManagement` sperrt [Q84][Q85]. Die Richtlinienreferenz nennt für `BlockAboutProfiles` zusätzlich `about:profilemanager`, `about:editprofile`, `about:newprofile` und `about:deleteprofile` als gesperrt [Q10]. Mozilla dokumentiert in Bug 2021356 außerdem, dass `browser.profiles.enabled=false` den neuen Profil-Manager samt diesen Seiten abschaltet, `about:profiles` aber nicht [Q13]. Die Präferenz fällt unter das erlaubte Präfix `browser.` und kann mit Status `locked` gesetzt werden [Q5]. Dokument 2 und 3 setzen beides: `BlockAboutProfiles` als tragende Maßnahme, die Präferenz als zweites Netz.
- Die Startparameter `-P`, `-ProfileManager` und `--profile` sperrt keine Richtlinie; weder die Referenz noch der Richtlinien-Code enthalten eine Sperre der Profilwahl auf der Kommandozeile [Q85]. Sie sind aber harmlos, solange nichts Sicherheitsrelevantes im Profil liegt: Das neue Profil startet mit denselben Richtlinien und denselben erzwungenen Erweiterungen, denn Richtlinien werden aus Installationsordner, Registry oder macOS-Defaults gelesen, nie aus dem Profil [Q86]. Deshalb enthalten Dokument 2 und 3 keine `userChrome.css` und keine `user.js` für Sperrzwecke. Siehe Entscheidung E5.
- `about:config` lässt sich per `BlockAboutConfig` sperren, `about:support` per `BlockAboutSupport`, `about:addons` per `BlockAboutAddons` [Q3][Q4]. Der Chatverlauf nennt keine davon.
- Der Fehlerbehebungsmodus (`-safe-mode`, Umschalttaste beim Start) schaltet Erweiterungen ab, also auch uBlock, ActivityWatch und die Cold-Turkey-Erweiterung [Q87]. `DisableSafeMode` sperrt unter Windows auch den Start über die Kommandozeile, aber nur, wenn die Richtlinie in der Registry steht; der Startcode liest ausschließlich die Registry, `policies.json` entfernt nur den Menüpunkt [Q6][Q88]. Unter macOS gibt es diesen Startcheck nicht; `-safe-mode` und die Wahltaste bleiben möglich [Q88]. Zwei Dinge halten in diesem Modus trotzdem: Cold Turkey schließt den Browser nach der Grace Period, weil die Erweiterung fehlt [Q16], und die Richtlinie `WebsiteFilter` wirkt weiter, weil Richtlinien nicht im Profil liegen. Das ist der Grund, warum Dokument 2 und 3 `WebsiteFilter` als zweite Freigabeliste neben Cold Turkey setzen.

Der im Chatverlauf vermutete Unterschied zwischen `"key": false` und `{"Value": false, "Status": "locked"}` ist kein Unterschied in der Wirksamkeit, sondern in der Syntax: Die `Preferences`-Richtlinie verlangt die Objektform mit `Value` und `Status`; die Kurzform `"key": false` ist in der aktuellen Referenz nicht dokumentiert [Q5]. Die ersten vier Fassungen der `policies.json` im Chatverlauf waren an dieser Stelle also unwirksam, nicht „manchmal nicht geschützt“.

## 2. Prüftabelle Chatverlauf

Spalte Entscheidung bezieht sich auf <grundsatz>: eine Sperre nur, wenn sie einen konkreten Ausbruchsweg schließt; mildere Maßnahme vor härterer; systemweit vor Profil.

| Nr. | Aussage im Chatverlauf | Prüfergebnis | Beleg | Entscheidung |
|---|---|---|---|---|
| C1 | Cold Turkey kann `msedge.exe` „absolut unmöglich“ zu öffnen machen; „Computer-Neustart-Schutz“ | überzogen. Cold Turkey blockiert exe-Dateien, Ordner und Store-Apps und verhindert während eines Blocks das Verschieben oder Umbenennen der Datei; eine Option „Computer-Neustart-Schutz“ existiert nicht, es gibt den Sperrtyp „Restart“ (Entsperren erst nach Neustart) [Q16]. | Q16 | ändern: `msedge.exe` wird gesperrt, zusätzlich AppLocker-Verweigerungsregel (Abschnitt 8.3), Protokoll-Handler und Edge-Entfernung als Entscheidung E3 |
| C2 | Cold-Turkey-Erweiterung aus AMO installieren; „Strict Mode“ friert die Erweiterung ein | teils falsch. Die Erweiterung liegt nicht auf addons.mozilla.org; der Cold-Turkey-Installer bietet sie je gefundenem Browser an, sonst wird sie als XPI von der Cold-Turkey-Seite installiert [Q16][Q34]. Einen „Strict Mode“ gibt es in der aktuellen Oberfläche nicht; es gibt Sperrtypen (Timer, Random Text, Time Range, Password, Restart, Schedule) und unter Settings > Blocking „Block browsers if extension crashes“ sowie die Grace Period [Q16]. Die Erweiterung selbst wird nicht „eingefroren“; sie wird vorausgesetzt, sonst wird der Browser geschlossen. | Q16 | ersetzen: Erweiterung per `ExtensionSettings` mit `install_url` auf die Hersteller-XPI erzwingen (Prüfschritt P1: Erweiterungs-ID aus der XPI, siehe Abschnitt 4.3), Cold Turkey mit Sperrtyp „Password“ und „Block browsers if extension crashes“ |
| C3 | Neuer Tab wird geschlossen, weil `about:blank`/`about:newtab` nicht in den Ausnahmen steht; Lösung `about:blank` und `moz-extension://*` als Ausnahme | unbelegt. Die Herstellerdoku kennt als Standardausnahme nur `file://*`; über `about:`-Seiten sagt sie nichts [Q16]. Ob das Symptom überhaupt von den Ausnahmen kommt, ist nicht belegt. | Q16 | ändern: zuerst `NewTabPage` false und `Homepage` auf eine freigegebene Seite; dann Prüfschritt in Dokument 2, ob ein Eintrag `about:blank` nötig ist. `moz-extension://*` wird nicht eingetragen, siehe C4. |
| C4 | `moz-extension://*` ist eine Hintertür; stattdessen exakte Erweiterungs-IDs | teils richtig in der Richtung, falsch in der Form. Cold-Turkey-Ausnahmen gelten für Webseiten-Muster; was ein Muster `moz-extension://*` in Cold Turkey öffnet, ist nicht dokumentiert. Die Seiten der erzwungenen Erweiterungen (uBlock-Dashboard, ActivityWatch) lassen sich ohnehin nicht zum Surfen nutzen. | Q16 | streichen: keine Ausnahme für Erweiterungs-Schemata; Prüfschritt, ob uBlock und ActivityWatch unter der Freigabeliste funktionieren |
| C5 | `ExtensionSettings` mit `force_installed` für uBlock Origin und ActivityWatch | richtig in der Syntax. `force_installed` installiert automatisch und verhindert das Entfernen; `install_url` ist seit Firefox 153 für AMO-Erweiterungen optional [Q9]. Erweiterungs-IDs: siehe Abschnitt 4 (Beleg je ID). | Q9 | übernehmen, ergänzt um `"*": {"installation_mode": "blocked"}` (schließt Erweiterungen aus Dateien und AMO) und um die Cold-Turkey-Erweiterung |
| C6 | Puzzle-Symbol durch „Symbolleiste anpassen“ entfernen | unwirksam als Sperre: Die Anpassung liegt im Profil (`browser.uiCustomization.state`) und ist für den Benutzer jederzeit rückgängig zu machen. Mit `BlockAboutAddons` und `force_installed` ist die Schaltfläche harmlos. | Q4, Q9 | streichen (kein Ausbruchsweg) |
| C7 | Adressleiste per `userChrome.css` (`#nav-bar {display:none}`) ausblenden | überzogen und unwirksam als Sperre: je Profil, vom Benutzer löschbar, und `about:config` nötig. Ohne Adressleiste ist Firefox für Arbeit und Lernen unbrauchbar. | Q5 | streichen |
| C8 | `DisableProfileCreation: true` | nicht existent. Der Name steht weder in der alten noch in der aktuellen Richtlinienreferenz [Q3][Q4]. | Q3, Q4 | ersetzen durch `BlockAboutProfiles` und `Preferences` `browser.profiles.enabled` locked false |
| C9 | `ProfileManager: false` | nicht existent [Q3][Q4]. | Q3, Q4 | ersetzen wie C8 |
| C10 | `DisableDeveloperTools: true` | richtig: entfernt den Zugriff auf alle Entwicklerwerkzeuge [Q3]. | Q3 | übernehmen |
| C11 | `DisableFormHistory: true` | existiert; schließt keinen Ausbruchsweg (Formularverlauf). | Q4 | streichen |
| C12 | `SanitizeOnShutdown` (Cache, Downloads, FormData, Locked) | existiert; schließt keinen Ausbruchsweg; „Downloads: true“ leert nur die Liste, nicht die Dateien. | Q4 | streichen |
| C13 | `DisableSetDesktopBackground` | existiert; kein Ausbruchsweg. | Q4 | streichen |
| C14 | `DisableFirefoxAccounts: true` | existiert; schließt Sync als Weg, fremde Einstellungen und Erweiterungen aus einem anderen Gerät zu beziehen. In der aktuellen Referenz zusätzlich `DisableAccounts` [Q4]. | Q4 | übernehmen (Prüfschritt: `DisableAccounts` in ESR 153 verfügbar?) |
| C15 | `DontCheckDefaultBrowser` | existiert; Komfort, kein Ausbruchsweg. | Q4 | übernehmen (verhindert Dialoge, kein Sicherheitsbezug) |
| C16 | `AppAutoUpdate: false` | existiert; führt dazu, dass Sicherheitsupdates vom Benutzer angestoßen werden müssen. Update-Verhalten ist Entscheidung E2. | Q3 | ändern: Entscheidung E2 |
| C17 | `DisableMasterPasswordCreation: false` | existiert; `false` ist der Standard, Eintrag ohne Wirkung. | Q4 | streichen |
| C18 | `NewTabPage: false`, `OverrideFirstRunPage: ""`, `OverridePostUpdatePage: ""` | existieren [Q4]. `NewTabPage: false` verhindert die Firefox-Startseite mit Empfehlungen; die beiden Override-Richtlinien verhindern Mozilla-Seiten nach Erststart und Update. Alle drei vermeiden Verbindungen zu nicht freigegebenen Seiten. | Q4 | übernehmen |
| C19 | `NewTabPage: {"Locked": true}` | falsch: Die Richtlinie ist ein Boolean, kein Objekt [Q3]. | Q3 | streichen (durch C18 ersetzt) |
| C20 | `Preferences` in Kurzform `"browser.profiles.enabled": false` | falsch in der Syntax (siehe Abschnitt 1) [Q5]. | Q5 | ersetzen durch Objektform |
| C21 | `Preferences` `browser.profiles.enabled` Objektform locked false | richtig; Präfix `browser.` erlaubt [Q5]; Wirkung belegt [Q13]. | Q5, Q13 | übernehmen |
| C22 | `browser.profile-reset.enabled` locked false | Präfix erlaubt [Q5]; Wirkung auf „Firefox bereinigen“ nicht belegt. Dafür gibt es `DisableProfileRefresh` [Q4]. | Q4, Q5 | ersetzen durch `DisableProfileRefresh: true` |
| C23 | `signon.rememberSignons` locked false | Präfix erlaubt; kein Ausbruchsweg. Dafür gibt es `PasswordManagerEnabled`/`OfferToSaveLogins` [Q4]. | Q4 | streichen |
| C24 | `browser.shell.checkDefaultBrowser` locked false | durch `DontCheckDefaultBrowser` abgedeckt. | Q4 | streichen (doppelt) |
| C25 | Startparameter `-no-remote -P "default-release"` in der Verknüpfung erzwingen | unwirksam: Verknüpfungen sind Benutzerdateien; ein Start über `firefox.exe` direkt ignoriert sie. Der Profilname `default-release` war zudem falsch, das ESR-Profil heißt `default-esr` (Screenshot im Chatverlauf). | Chatverlauf | streichen |
| C26 | Cold Turkey soll den Windows-Explorer oder den Pfad `AppData\Roaming\Mozilla\Firefox\Profiles` sperren | unbelegt und überzogen. Cold Turkey sperrt Programme und Webseiten, keine Ordnerzugriffe für andere Programme; eine Sperre des Explorers macht Windows unbrauchbar. Dateizugriff regeln Dateirechte und die Tatsache, dass sicherheitsrelevante Einstellungen nicht im Profil liegen. | Q16 | streichen |
| C27 | Schreibrechte auf `distribution` für das eigene Konto entziehen, „auch nicht mit Administratorrechten“ | teils richtig, teils falsch. Standardbenutzer haben in `C:\Program Files` ohnehin keine Schreibrechte; ein Administrator kann jedes Recht zurücknehmen. Deshalb Variante A (Abschnitt 10). Unter Windows werden Richtlinien außerdem in die Registry (HKLM) gesetzt, die ein Standardbenutzer nicht ändern kann; `DisableSafeMode` sperrt den Umschalt-Start laut Referenz nur über Gruppenrichtlinie, nicht über `policies.json` [Q6]. | Q6 | ersetzen: Richtlinien als Registry/Gruppenrichtlinie (Dokument 2), Standardkonto |
| C28 | `userChrome.css` mit `#fxa-menu-button-container, #profiler-button, #identity-box {display:none}` | überzogen und teils schädlich: `#identity-box` ist die Sicherheitsanzeige links in der Adressleiste (Schloss, Berechtigungen), kein Profilelement; das Ausblenden nimmt dem Benutzer die Zertifikatsanzeige. `#profiler-button` ist der Leistungs-Profiler, nicht das Benutzerprofil. Je Profil, vom Benutzer löschbar. | Q5 (je Profil) | streichen |
| C29 | `user.js` mit `browser.profiles.enabled` false „überschreibt jeden internen Schalter hart“ | falsch: `user.js` setzt Benutzerwerte je Profil; eine per Richtlinie gesperrte Präferenz hat Vorrang und braucht keine `user.js`. | Q5 | streichen |
| C30 | Profilordner `jsbqnmfa.default-esr` ist das ESR-Profil | richtig für die Benennung. Für die Anleitung ohne Bedeutung, da nichts mehr im Profil abgelegt wird; bestehende Profile werden vor dem Scharfschalten bereinigt (Dokument 2, Vorbereitung). | Chatverlauf | übernehmen als Hinweis zur Bereinigung |
| C31 | `profiles.ini` bereinigen und Fremdprofile löschen | richtig als Bereinigung vor dem Scharfschalten; keine Sperrwirkung, da ein Benutzer Profile neu anlegen kann (siehe Abschnitt 1). | Chatverlauf | übernehmen als Vorbereitungsschritt |
| C32 | Richtlinien gelten laut Chatverlauf nicht, weil „Firefox im Standard-Modus zu viele Freiheiten lässt“ | falsch. Die Richtlinien griffen nicht, weil zwei Namen nicht existieren (C8, C9) und die `Preferences`-Syntax falsch war (C20). | Q3, Q4, Q5 | ersetzen |

Was im Chatverlauf fehlt und in Dokument 2 und 3 ergänzt wird: `BlockAboutConfig`, `BlockAboutSupport`, `BlockAboutAddons`, `BlockAboutProfiles`, `DisableSafeMode`, `PrivateBrowsingModeAvailability` (Wert 1; ersetzt `DisablePrivateBrowsing`) [Q7], `DisableProfileRefresh`, `DisableProfileImport`, `Proxy` mit `Mode: none, Locked: true` [Q11], `WebsiteFilter` als zweite Freigabeliste im Browser (`<all_urls>` blockieren, Ausnahmen für die freigegebenen Seiten; bis 1000 Einträge je Liste, `file:` wird unterstützt) [Q8], `ExtensionSettings` mit `"*": blocked`, `DisableFeedbackCommands`, `DisableTelemetry`, `DisableFirefoxStudies`, `UserMessaging`, `Handlers` und `PDFjs` nach Matrix, `DisableAppUpdate` oder `ManualAppUpdateOnly` nach Entscheidung E2.
## 3. Prüftabelle Edge-Dokumentation

Die Edge-Dokumentation beschreibt einen funktionierenden Zustand, aber unter drei Bedingungen, die jetzt wegfallen: Edge als Hauptbrowser, der Nutzer als Administrator ohne Anwendungssteuerung, Eclipse als IDE. Jede Maßnahme wird deshalb danach bewertet, welchen Ausbruchsweg sie unter den neuen Bedingungen (Standardkonto, AppLocker, Firewall-Regeln) noch schließt.

### 3.1 Edge-Richtlinien (Abschnitt 1 der Edge-Doku)

| Nr. | Maßnahme | Prüfergebnis | Entscheidung |
|---|---|---|---|
| E1 | `BrowserAddProfileEnabled`, `BrowserGuestModeEnabled`, `HubsSidebarEnabled`, `ShowDeveloperTools`, `EdgeShoppingAssistantEnabled` = 0 unter `HKLM\...\Policies\Microsoft\Edge` | Edge-Richtlinien; sie betreffen nur Edge. | streichen für den Browser, denn Edge wird gesperrt oder entfernt (Entscheidung E3). Offen bleibt, ob sie für WebView2 nützlich sind (Entscheidung E4): WebView2 liest einen Teil der Edge-Richtlinien, siehe Abschnitt 6. |
| E2 | „New Tab Redirect“ auf `about:blank`, Ausnahme `chrome-extension://...` | Edge-spezifisch; Ausnahme nicht belegt (Abschnitt 4.1). | ersetzen durch `NewTabPage: false` in Firefox |
| E3 | `edge://extensions` „ohne Internetverbindung harmlos“ | Für Firefox entspricht das `about:addons`; dort lassen sich Erweiterungen deaktivieren, was einen gesperrten Block auslöst (Browser wird geschlossen). | ersetzen durch `BlockAboutAddons` und erzwungene Erweiterungen |

### 3.2 Windows-Sperren und Dienste (Abschnitt 3 der Edge-Doku)

| Nr. | Maßnahme | Welchen Weg schließt sie unter Variante A? | Nebenwirkung | Entscheidung |
|---|---|---|---|---|
| E4 | `NoControlPanel = 1` (Explorer-Richtlinie „Zugriff auf Systemsteuerung und PC-Einstellungen nicht zulassen“, Benutzerbereich) [Q51] | Keinen, der nicht bereits durch das Standardkonto geschlossen ist: Zeitänderung, Richtlinien, Dienste, Firewall, Konten verlangen Adminrechte. | Der Nutzer kann Anzeige, Ton, Bluetooth, Bedienungshilfen und Drucker nicht einstellen; Windows-Update-Seite nicht erreichbar. | streichen (Variante A). Unter Variante B eine schwache Hürde ohne Schutz vor einem Administrator, dort ebenfalls streichen. |
| E5 | `DisableTaskMgr = 1` (Richtlinie „Task-Manager entfernen“, Benutzerbereich) [Q52] | Keinen: Ein Standardbenutzer kann den Cold-Turkey-Dienst nicht beenden; eigene Prozesse zu beenden ist harmlos. Cold Turkey bietet dafür „Block task manager“ [Q16]. | Fehlersuche (hängendes Programm) ohne Task-Manager. | streichen; falls gewünscht die Cold-Turkey-Option statt der Richtlinie (mildere Maßnahme, rückbaubar durch die Vertrauensperson) |
| E6 | `sc config wuauserv start= disabled` | Keinen Ausbruchsweg. Ein Standardbenutzer kann über Windows Update keine Anwendung installieren. | Keine Sicherheitsupdates für Windows; Defender-Updates kommen zwar über eigene Kanäle, Qualitätsupdates aber nicht. | streichen, ersetzen durch Update-Steuerung nach Entscheidung E2 |
| E7 | `sc config UsoSvc start= disabled` | Keinen. | Wie E6. | streichen |
| E8 | `sc config TrustedInstaller start= disabled` | Keinen. Der Dienst (Windows Modules Installer) installiert Updates und optionale Komponenten und arbeitet nicht auf Benutzerwunsch. | Updates und Reparaturen (DISM, sfc) schlagen fehl; Windows kann in einen nicht wartbaren Zustand geraten. | streichen |

Die Begründung der Edge-Doku („kombinierter Schutz“: weil `regedit.exe`, `cmd.exe`, `powershell.exe` und `Taskmgr.exe` gesperrt sind, können Registry-Schlüssel nicht zurückgesetzt werden) ist unter einem Adminkonto falsch: Ein Administrator startet jedes dieser Programme unter anderem Namen, nutzt `reg.exe`, `pwsh.exe`, die PowerShell ISE oder den Registrierungs-Editor über `mmc`. Unter einem Standardkonto ist die Sperre überflüssig, weil `HKLM` nicht beschreibbar ist. Die Registry-Sperre der Edge-Doku hatte also nur Bestand, solange niemand ernsthaft suchte.

### 3.3 Cold-Turkey-Prozessliste (Abschnitt 4 der Edge-Doku), neu bewertet

Die 110 Einträge sind nach Zweck gruppiert. „Weg“ ist der Ausbruchs- oder Ablenkungsweg, den die Sperre schließt. „A“ ist die mildere Maßnahme, die den Weg ebenfalls schließt. Entscheidung nach <grundsatz>; Einträge, die nur der Ablenkung dienen und keinen Ausbruchsweg öffnen, sind gekennzeichnet, weil der Zweck des Lockdowns ausdrücklich Schutz vor Ablenkung einschließt. Dort entscheidest du (E11).

| Gruppe | Prozesse der Edge-Doku | Weg | A: mildere Maßnahme | Entscheidung |
|---|---|---|---|---|
| Python | `py.exe`, `pymanager.exe`, `py-manager.exe`, `pyshellext.exe`, `python.exe`, `python3.exe`, `pythonw.exe`, `pythonw3.exe`, `pyw.exe`, `pywmanager.exe`, `pyw-manager.exe` | Code mit Netzzugriff, Paketinstallation, Start anderer Programme | Firewall-Regel ausgehend (Abschnitt 8.1), AppLocker (8.3), Paketregel (8.2) | freigeben; alle elf Einträge entfernen. `pymanager`/`py-manager` werden nicht installiert (klassischer Installer, Abschnitt 7.3). |
| Shells, die Kate und die Interpreter brauchen | `cmd.exe`, `powershell.exe`, `wt.exe` | Start beliebiger Programme; Downloads per `curl.exe` (in Windows enthalten [Q61]), `Invoke-WebRequest`, `bitsadmin`, `certutil`, sowie `bash.exe` und `curl.exe` aus Git for Windows [Q62] | AppLocker schließt den Start von Programmen außerhalb der Systemordner; die Firewall-Regel ausgehend schließt Downloads durch `powershell.exe`, `pwsh.exe`, `curl.exe`, `certutil.exe`, `bitsadmin.exe` (Abschnitt 8.1) | freigeben: `cmd.exe` und `powershell.exe` sind für Kates Build-Plugin und für das Arbeiten mit Python und Java nötig; `wt.exe` (Windows Terminal) ebenfalls. Die Sperre war nur unter einem Adminkonto sinnvoll und dort unvollständig (`pwsh.exe`, `powershell_ise.exe`, `conhost.exe` fehlten). |
| Registry, Task-Manager, Fernwartung | `regedit.exe`, `Taskmgr.exe`, `QuickAssist.exe` | Richtlinien zurücksetzen; Dienste beenden; Fernsteuerung durch Dritte | Standardkonto (HKLM nicht beschreibbar, Dienste nicht beendbar); QuickAssist erlaubt nur, dass eine andere Person den Bildschirm steuert, mit denselben Rechten wie der Nutzer | `regedit.exe`, `Taskmgr.exe`: freigeben unter Variante A. `QuickAssist.exe`: freigeben (kein Ausbruchsweg); unter Variante B alle drei sperren, mit benanntem Restrisiko (Adminkonto). |
| WSL, Paketmanager, Store | `wsl.exe`, `wslInstaller.exe`, `winget.exe`, `store.exe`, `WinStore.App.exe`, `StoreExperienceHost.exe`, `StoreDesktopExtension...`, `DeploymentAgent.exe` | Browser aus dem Store oder per `winget` ohne Adminrechte installieren; Linux-Umgebung mit eigenem Browser und ungefiltertem Netz | AppLocker: Packaged-App-Regel nur für Microsoft-Herausgeber, Verweigerung für `Microsoft.WindowsStore`; Programme aus Benutzerordnern nicht startbar; WSL-Installation braucht Adminrechte (Prüfschritt in Dokument 2) | `winget.exe`, `DeploymentAgent.exe`: streichen (durch AppLocker geschlossen). Store-Prozesse: übernehmen als zweite Schicht neben der AppLocker-Regel, weil Store-Apps per Benutzer installiert werden und die Packaged-App-Regeln bei jedem Store-Update neu zu prüfen sind. `wsl.exe`, `wslInstaller.exe`: übernehmen, bis der Prüfschritt zeigt, dass WSL für Standardbenutzer nicht installierbar ist. |
| Eingebettete Browser (WebView2) und Microsoft-Dienste mit Webinhalt | `Widgets.exe`, `WidgetService.exe`, `WidgetBoard.exe`, `WebExperienceHostApp.exe`, `MicrosoftStartFeed...`, `Microsoft.Msn.News.exe`, `Microsoft.Msn.Weather.exe`, `ms-teams.exe`, `ms-teams_autostarter.exe`, `ms-teamsupdate.exe`, `olk.exe`, `olkBg.exe`, `olkExthost.exe`, `olkFullTrustExecutor.exe`, `olkMcpServer.exe`, `olkPushNotification...`, `CoPilotLibraryBroker.exe`, `m365copilot_autostarter.exe`, `m365copilot_widget.exe`, `m365copilotwindows...`, `aicontext.exe`, `AppActions.exe`, `ClickToDo.exe`, `DevHome.exe`, `DiscoveryHubApp.exe`, `FeedbackHub.exe`, `FeedbackHubServices.exe`, `GetHelp.exe`, `PhoneExperienceHost.exe`, `PhoneAgentMcpServer.exe`, `YourPhoneAppProxy.exe`, `YourPhoneAppProxyHost.exe`, `CrossDevice*.exe` (sechs Einträge), `EdgeGameAssist.exe`, `GameBar.exe`, `GameBarElevatedFT.exe`, `GameBarFTServer.exe`, `XboxGameBarWidgets.exe`, `Xbox.TCUI.exe`, `XboxIdp.exe`, `XboxPartyChatHost.exe`, `XboxPcApp*.exe` (fünf Einträge), `Clipchamp.exe`, `Clipchamp.Widgets.exe`, `ClockWidgets.exe`, `PeopleExperienceHost.exe`, `Todo.exe`, `StickyNotesStub.exe`, `WindowsBackupClient.exe`, `WpcUapApp.exe`, `PAD.Console.Host.exe`, `AssignedAccessLockApp.exe`, `CHXSmartScreen.exe`, `CameraBarcodeScanner...`, `AddSuggestedFolders...`, `SpeechToTextOverlay64...`, `DesktopStickerEditor...`, `ScreenClippingHost.exe`, `SnippingTool.exe`, `SoundRec.exe`, `mspaint.exe`, `Solitaire.exe`, `Microsoft.Solitaire...` | Zwei Wege: (1) Webinhalte außerhalb Firefox anzeigen (Widgets, News, Teams, Outlook, Copilot, Hilfe, Feedback-Hub, Dev Home, Phone Link, Xbox-Apps nutzen WebView2 oder eigene Web-Ansichten); (2) Ablenkung ohne Web (Solitaire, Game Bar, Clipchamp-Editor, Sticker). | Für (1): Firewall-Regel ausgehend für `msedgewebview2.exe` (Entscheidung E4) schließt den Internetzugang aller WebView2-Hosts auf einmal. Für (2): keine mildere Maßnahme. | Übernehmen, aber in zwei Listen getrennt: Liste „Web außerhalb Firefox“ (alle WebView2-Hosts) bleibt gesperrt, zusätzlich zur Firewall-Regel, weil Cold Turkey sichtbar meldet, was die Firewall still verwirft. Liste „Ablenkung“ (Solitaire, Game Bar, Xbox, Clipchamp, Sticker, Clock-Widgets): Entscheidung E11; Empfehlung sperren, weil der Zweck Schutz vor Ablenkung ist. Freigeben: `SnippingTool.exe`, `ScreenClippingHost.exe`, `SoundRec.exe`, `mspaint.exe` (Werkzeuge ohne Webinhalt, Paints Copilot-Funktionen laufen über die gesperrte WebView2/Netzverbindung), `WindowsBackupClient.exe` (nur bei Microsoft-Konto relevant; kein Weg), `AssignedAccessLockApp.exe`, `CHXSmartScreen.exe`, `WpcUapApp.exe` (Systemkomponenten, Sperre bringt nichts und kann Dialoge stören), `CameraBarcodeScanner...`, `AddSuggestedFolders...`, `SpeechToTextOverlay64...` (Hilfsprozesse ohne Weg). |
| Freigabeliste der Edge-Doku (Abschnitt 5) | `AccountsControlHost.exe` bis `WindowsPackageManagerServer.exe` | keine Sperre | | übernehmen; ergänzt um `kate.exe`, `python.exe`, `pythonw.exe`, `py.exe`, `java.exe`, `javaw.exe`, `cmd.exe`, `powershell.exe`, `wt.exe`, `conhost.exe`, `git.exe` und die Hilfsprogramme von Git, `firefox.exe`, `Cold Turkey Blocker.exe`, `msedgewebview2.exe` (nicht sperren, siehe E4), `explorer.exe`, `Taskmgr.exe`, `regedit.exe` (Variante A). Hinweis: `AppInstaller.exe` und `WindowsPackageManagerServer.exe` stehen in der Edge-Doku auf der Freigabeliste, `winget.exe` auf der Sperrliste; das war widersprüchlich. Künftig wird nichts davon in Cold Turkey gesperrt, AppLocker übernimmt. |

Neu hinzu kommen in Cold Turkey: `msedge.exe` (Weg: zweiter Browser), `mshta.exe` und `hh.exe` (Weg: HTML-Anzeige über die alte Internet-Explorer-Engine; Prüfschritt, ob beide unter Windows 11 noch vorhanden sind), sowie unter Entscheidung E4 ggf. `msedgewebview2.exe` (nicht empfohlen, siehe Abschnitt 6).

### 3.4 Eclipse-Härtung (Abschnitt 6 der Edge-Doku)

| Maßnahme | Entscheidung |
|---|---|
| `eclipse.p2.unsignedPolicy=prompt` in `config.ini` | entfällt mit Eclipse |
| Websperren `*.eclipse.org/downloads*`, `marketplace.eclipse.org*` | entfällt; unter einer Freigabeliste sind sie ohnehin gesperrt |
| Web Browser auf „External Browser“ | entfällt; in Kate öffnen Links im Standardbrowser (Firefox ESR) |
| Freigabe `eclipse.exe`, `java.exe`, `javaw.exe` | `eclipse.exe` entfernen; `java.exe`, `javaw.exe` bleiben freigegeben (Abschnitt 7) |

Eclipse-Reste nach der Deinstallation (Dokument 2, Vorbereitung): der Eclipse-Programmordner (bei ZIP-Installation wird er nicht per Deinstallation entfernt), `%USERPROFILE%\.p2` (Bundle-Pool des Installers, laut Oomph-Entwickler nach dem Löschen des Installationsordners optional zu entfernen [Q72]), `%USERPROFILE%\.eclipse`, `%USERPROFILE%\eclipse-workspace` (enthält eigene Projekte, sichern), Startmenü- und Desktop-Verknüpfungen, der Eclipse-Installer-Ordner, Einträge in Cold Turkey (Freigabe `eclipse.exe`, Websperren). Das für Eclipse installierte JDK bleibt, wenn seine Version zum Java Language Server passt (Abschnitt 7.4), sonst wird es über „Programme und Features“ entfernt; Oracle weist darauf hin, dass die Deinstallation Registry-Einträge zurücklassen kann [Q73].

### 3.5 Strict Mode und Passwort (Abschnitt 7 der Edge-Doku)

Die drei genannten Optionen gibt es unter diesem Namen nicht (Abschnitt 4.1). Der beschriebene Zustand entspricht: Sperrtyp „Password“ je Block, Option „Block the installer“, Option „Block time changes“, dazu das Settings-Passwort („Password Settings“) für den Zugang zur Oberfläche. Der Satz „ein langes, zufälliges Passwort“ wird übernommen (Abschnitt 11, Entscheidung E10). Die Behauptung „unumgehbar“ wird nicht übernommen; was Cold Turkey allein nicht leistet, steht in Abschnitt 4.6.
## 4. Cold Turkey: Ist-Stand gegen Herstellerdokumentation

Geprüft wurde die aktuelle Version Cold Turkey Blocker 4.9(2) vom 1. März 2026 (Release Notes; der Installer trägt das Datum 19. März 2026, ein dokumentierter Hotfix dazu fehlt) [Q30][Q31]. Windows und Mac teilen Version und Release Notes. Admin-Rechte sind für die Installation auf beiden Plattformen zwingend [Q18].

### 4.1 Begriffe der Edge-Dokumentation und des Chatverlaufs

| Begriff in Edge-Doku oder Chatverlauf | Existiert in Cold Turkey 4.9(2)? | Richtiger Name und Wirkung | Beleg |
|---|---|---|---|
| „Strict Mode“ | nein | Es gibt Sperrtypen je Block: Timer, Random Text, Time Range, Password (Pro), Restart (nur bei fortlaufenden Blöcken), Schedule (nur bei geplanten Blöcken); die Marketingseite nennt zusätzlich „Delay“. Ein gesperrter Block lässt den Uninstaller fehlschlagen, verlangt die aktive Browser-Erweiterung, deaktiviert einige Optionen unter Settings > Blocking und verhindert Brave-Tor-Fenster und Gastbrowsing. | Q16, Q31 |
| „Block uninstalling Cold Turkey“ | nein | Deinstallationsschutz ist Folge jedes gesperrten Blocks; zusätzlich die Option „Block the installer“ unter Settings > Blocking, die auch Updates verhindert. | Q16 |
| „Block changing system time“ | nicht wörtlich | „Block time changes“ (User Guide) bzw. „Block time and date settings“ (Features-Seite). Unter Windows blockiert sie die Einstellungen-App beim Ändern der Zeit; der Hersteller räumt in der FAQ Umgehungen ein und empfiehlt, dem Konto das Recht zur Zeitänderung zu entziehen und `mmc.exe` zu sperren. Unter macOS werden die Systemeinstellungen blockiert. | Q16, Q17 |
| „Web-Exceptions“ | nicht wörtlich | Tab „Website Exceptions“ je Block. Ausnahmen gelten nur für den Block, in dem sie stehen. Standardeintrag `file://*`. `localhost` ist als Ausnahme zulässig (Release Notes 3.10). | Q16, Q30 |
| Ausnahme `chrome-extension://...` (Edge-Doku) | nicht belegt | Kein Herstellertext nennt dieses Schema. Browser-interne Seiten werden laut Hersteller-Anleitung mit `chrome://`, `edge://` und `about:` angesprochen. | Q33 |
| Ausnahme `moz-extension://*` (Chatverlauf) | nicht belegt | Wie oben. Technisch ist `moz-extension://` das Schema aller Erweiterungsseiten; eine solche Ausnahme nähme jede Erweiterungsseite vom Block aus. | Q33 |
| „Computer-Neustart-Schutz“ (Chatverlauf) | nein | Sperrtyp „Restart“: Entsperren erst nach Neustart. Mit Sperrtyp „Password“ ist er überflüssig. | Q16 |
| „Whitelist-Modus“ (Chatverlauf) | nein als Schalter | Eine Freigabeliste entsteht durch `*.*` im Tab „Websites“ (oder Import > The entire internet) plus Einträge im Tab „Website Exceptions“. | Q16, Q31 |
| „Allowed/blocked browsers“, „Start at login“ | nein | Verwandt: „Block unsupported browsers“, „Block browsers if extension crashes“, Autostart „Sign In“. | Q16, Q31 |

### 4.2 Was die Herstellerdokumentation zum Symptom „neuer Tab wird geschlossen“ hergibt

Belegt ist nur: Browser-interne Seiten werden nicht mit der Block-Seite beantwortet, sondern der Tab wird geschlossen und eine Benachrichtigung erscheint; ein `*` im Tab „Websites“ trifft Browser-URLs nicht; `about:*` trifft die Neuer-Tab-, Erweiterungs- und Willkommensseiten nicht, die müssten mit voller Adresse eingetragen werden [Q33]. Nicht belegt ist, dass `*.*` die Firefox-Neuer-Tab-Seite trifft oder dass `about:blank` als Ausnahme nötig ist. Entscheidung: `NewTabPage: false` per Richtlinie, keine `about:`-Einträge im Tab „Websites“, und ein Prüfschritt in Dokument 2 und 3, ob ein neuer Tab offen bleibt; nur wenn nicht, wird `about:blank` als Ausnahme eingetragen (öffnet eine leere Seite, keinen Inhalt).

### 4.3 Browser-Erweiterung für Firefox

Die Cold-Turkey-Erweiterung für Firefox liegt nicht auf addons.mozilla.org. Der Hersteller verteilt sie selbst als `https://getcoldturkey.com/files/Cold_Turkey_Firefox_Addon.xpi` über die Seite `getcoldturkey.com/support/extensions/firefox` [Q34]. Die Erweiterung muss in privaten Fenstern laufen dürfen („Run in Private Windows“ in `about:addons`), der Hersteller begründet das mit einem sonst einfachen Umgehungsweg [Q35]. Die Cold-Turkey-App prüft alle 10 Sekunden, ob die Erweiterung aktiv ist; während eines gesperrten Blocks wird der Browser nach der Grace Period (Standard 60 Sekunden) geschlossen [Q16].

Folgen für die Anleitungen:

- Die Erweiterung lässt sich per `ExtensionSettings` erzwingen, denn `install_url` darf auf eine beliebige URL zeigen [Q9]. Die Erweiterungs-ID (`browser_specific_settings.gecko.id` in `manifest.json` der XPI) ist nicht belegt; sie steht in Dokument 2 und 3 als Prüfschritt (XPI entpacken, ID ablesen). Mit `private_browsing: true` in `ExtensionSettings` ist die Freigabe für private Fenster erzwungen [Q9]; da private Fenster ohnehin per `PrivateBrowsingModeAvailability` abgeschaltet werden, ist das ein zweites Netz.
- Mit erzwungener Erweiterung ist jedes neue Firefox-Profil sofort abgedeckt. Ohne Erzwingung schützt nur die Grace Period.
- `about:addons` wird per `BlockAboutAddons` gesperrt. Cold Turkey rät bei Problemen, die Erweiterung über `about:addons` zu prüfen; für die Vertrauensperson bleibt dafür das Wartungsfenster mit temporär entfernter Richtlinie, siehe Dokument 2 und 3, Betrieb.

### 4.4 Blöcke und Benutzer

Blöcke gelten für ausgewählte Windows- oder macOS-Benutzer („Edit affected users“, Pro-Funktion „Custom User Selection“); der Standardwert ist nicht dokumentiert, und die Fehlersuche des Herstellers beginnt mit „Make sure your user is included“ [Q16][Q31][Q32]. Seit 4.2 wirken Blöcke unter macOS auch auf neu angelegte Benutzer [Q30]. Folge: In beiden Anleitungen steht ein Prüfschritt, dass jeder Block für das Arbeitskonto aktiv ist; nach Anlegen des Standardkontos (Variante A) muss das geprüft werden, bevor der Block gesperrt wird.

### 4.5 Anwendungssperren je Plattform

Windows: „Add exe file“, „Add exe's in folder“, „Add Windows 10 app“ (Store-Apps), „Add window title“ (Pro, nur Windows). Eine gesperrte exe-Datei kann während des Blocks nicht verschoben oder umbenannt werden [Q16]. macOS: „Add application“, „Add folder“, „Add binary“ (für Programme ohne App-Bundle, der Hersteller nennt Java als Beispiel). Unter macOS bleibt eine App auch nach Umbenennen oder Verschieben gesperrt [Q16]. Beides deckt nur Programme ab, die Cold Turkey kennt; Kopien gesperrter Programme unter anderem Namen (Windows) und portable Programme in Benutzerordnern erfasst nur die Anwendungssteuerung (Abschnitt 8.3).

### 4.6 Was der Hersteller nicht sagt

Nicht dokumentiert sind: Verhalten im abgesicherten Modus von Windows und im sicheren Modus von macOS; ob ein zweites Adminkonto gesperrte Blöcke ändern oder die App entfernen kann; Verhalten bei VPN, Proxy oder DNS; portable oder umbenannte Browser (nur „Block unsupported browsers“ als Auffangnetz); die unter macOS nötigen Berechtigungen außer Vollzugriff auf die Festplatte für die Safari-Prüfung [Q36]. Jeder dieser Punkte steht in der Matrix (Abschnitt 9) mit der Maßnahme, die ihn unabhängig von Cold Turkey schließt, oder als Restrisiko.

Der Hersteller selbst veröffentlicht eine Anleitung zur Neuinstallation, die hinter einer Wartezeit liegt und die er zu sperren empfiehlt [Q76]. Drittquellen beschreiben Deinstallationswege während eines Blocks (Dienst über `msconfig` deaktivieren, abgesicherter Modus). Sie sind nicht geprüft; sie bestätigen aber, dass Cold Turkey allein keinen Administrator aufhält. Das ist das Hauptargument für Variante A (Abschnitt 10).
## 5. Windows 11 Pro: Nebenwirkungen der Edge-Doku und Grundschutz der neuen Anleitung

### 5.1 Updates und TrustedInstaller

Die Edge-Dokumentation schaltet `wuauserv`, `UsoSvc` und `TrustedInstaller` ab. Microsoft warnt bei der Richtlinie, die automatische Updates abschaltet (Wert 5 von `AllowAutoUpdate`), ausdrücklich, dass dann auch keine Sicherheitsupdates mehr ankommen [Q48]. Defender-Definitionen und Plattform-Updates werden über Windows Update ausgeliefert; die Microsoft-Fallback-Quelle greift standardmäßig erst nach sieben Tagen ohne Update [Q50]. Der Windows Modules Installer (TrustedInstaller) wird für Updates und Komponentenänderungen gebraucht; dauerhaftes Deaktivieren führt zu Update-Fehlern (Community-Antwort auf Microsoft Q&A, keine Produktdokumentation gefunden) [Q77]. Eine Microsoft-Aussage, dass Reparaturen mit DISM und sfc ohne den Dienst scheitern, konnte nicht belegt werden; sie steht als Prüfschritt nicht in der Anleitung, die Deaktivierung wird unabhängig davon gestrichen, weil sie keinen Ausbruchsweg schließt (Abschnitt 3.2).

Ersatz in Dokument 2 (Entscheidung E2): automatische Updates bleiben an; Richtlinie „Automatische Updates konfigurieren“ mit Option 4 (Download und geplante Installation) und Aktive Stunden gegen Neustarts während der Arbeit [Q49]; die Richtlinie „Zugriff auf alle Windows Update-Funktionen entfernen“ nimmt dem Standardbenutzer nur die Schaltfläche „Nach Updates suchen“, Hintergrund-Scans laufen weiter [Q49]. Updates installieren keine Browser und ändern keine Richtlinien; nach einem Funktionsupdate steht in Dokument 2, Betrieb, was erneut zu prüfen ist (App-Control-Richtlinie aktiv, Firewall-Profil, Aliase, Edge nicht zurückgekehrt).

### 5.2 Standardkonto gegen Adminkonto

Standardbenutzer bekommen bei Verwaltungsaufgaben die UAC-Anmeldeaufforderung und brauchen gültige Admin-Anmeldedaten [Q54]. Was sie trotzdem können, und was deshalb eine eigene Maßnahme braucht:

| Ein Standardbenutzer kann | Beleg | Maßnahme |
|---|---|---|
| Programme in den eigenen Benutzerordner installieren (Python „Install Now“, Firefox ohne Adminrechte ins Home-Verzeichnis) | Q63, Q58 | Anwendungssteuerung (8.3) |
| Store-Apps installieren (Packaged Apps brauchen keine Adminrechte) | Q22 | AppLocker-Packaged-App-Regel und Cold-Turkey-Sperre des Store (3.3) |
| Eigene Aufgaben in der Aufgabenplanung anlegen, nur im eigenen Kontext ohne erhöhte Rechte | Q53 | harmlos, solange die Anwendungssteuerung steht; Aufgaben starten nur erlaubte Programme |
| „Nach Updates suchen“ auslösen | Q49 | optional entfernen (5.1); kein Ausbruchsweg |
| Firefox-Profile anlegen, `profiles.ini` ändern, `user.js` löschen | Abschnitt 1 | nichts Sicherheitsrelevantes im Profil (Grundsatz) |
| Eigene Firewall-Regeln: nein. Werkzeuge verlangen Adminrechte; eine Abfrage an Nicht-Admins erzeugt immer Block-Regeln | Q67 | keine |
| `HKLM` schreiben, Dienste konfigurieren: nein (Microsofts Anleitungen verlangen dafür „als Administrator ausführen“) | Q63 (per-user-services), Standard-ACL | keine |

Das eingebaute Administratorkonto lässt sich nicht löschen, aber deaktivieren; Microsoft empfiehlt das [Q55]. Dokument 2 sieht vor: ein lokales Adminkonto für die Vertrauensperson, das eingebaute Konto deaktiviert, dein Konto als Standardbenutzer.

### 5.3 Anwendungssteuerung: App Control for Business oder AppLocker

Beide stehen auf Windows 11 Pro zur Verfügung und werden durchgesetzt: App Control for Business ist laut Editionstabelle auf Pro verfügbar und wirkt laut Verfügbarkeitsübersicht auf allen Editionen, nur die PowerShell-Cmdlets fehlen auf Home [Q42][Q19]; AppLocker-Richtlinien werden seit KB 5024351 auf allen Editionen von Windows 11 durchgesetzt, eine bestimmte Edition ist nicht mehr nötig [Q20]. Microsoft stuft App Control als Sicherheitsfeature nach MSRC-Kriterien ein, AppLocker nicht, und empfiehlt, App Control zu nutzen, wo möglich, und AppLocker zur Feinsteuerung je Benutzer darüberzulegen [Q43].

| Kriterium | App Control for Business | AppLocker |
|---|---|---|
| Gilt für | alle Benutzer einschließlich Admin | wählbar je Benutzer oder Gruppe |
| Pfadregeln | nur für Pfade, die zur Laufzeit ausschließlich für Admin-SIDs beschreibbar sind; sonst gilt die Regel nicht (Option 18 würde das abschalten) [Q45] | Pfadregeln ohne Schreibbarkeitsprüfung; `%windir%\*` schließt beschreibbare Unterordner wie `Windows\Temp` ein, Ausnahmen sind von Hand zu pflegen [Q21] |
| Skripte und LOLBins | mit Skriptdurchsetzung: `mshta` und MSXML vollständig gesperrt, `wscript`/`cscript` nur mit Regel, PowerShell im Constrained Language Mode für nicht erlaubte Skripte [Q44] | `.ps1`, `.vbs`, `.js`, `.cmd`, `.bat` über Skriptregeln; `mshta.exe`, `rundll32.exe` usw. nur über eigene Verweigerungsregeln |
| Python- und Java-Code | nicht kontrolliert („unenlightened hosts“); wer den Interpreter erlaubt, erlaubt alles darin [Q44] | gleich |
| Wartung | Installer aus dem Download-Ordner des Admins werden ebenfalls blockiert; Installer vorher in einen nur für Admins beschreibbaren Ordner kopieren oder im Wartungsfenster in den Audit-Modus wechseln [Q45][Q47] | Admin ist ausgenommen (Standardregel „Alle Dateien“ für Administratoren) [Q21] |
| Werkzeuge | App Control Policy Wizard (Open Source, MSIX) oder ConfigCI-Cmdlets; Bereitstellung lokal mit `CiTool --update-policy` [Q47] | `secpol.msc`, Dienst „Anwendungsidentität“, PowerShell AppLocker-Cmdlets |

Entscheidung E14 mit Empfehlung: App Control for Business als Basis (Vorlage „Default Windows Mode“ plus Pfadregeln für `C:\Program Files*` und `C:\Program Files (x86)*`, erst im Audit-Modus, dann erzwungen) und AppLocker nur für Verweigerungen je Benutzer (`msedge.exe`, Store-App, Packaged-App-Regel). Begründung: Die Laufzeitprüfung der Pfadregeln schließt die bekannten beschreibbaren Unterordner von `C:\Windows` ohne Pflegeliste; die Skriptdurchsetzung schließt `mshta.exe` als eingebetteten Browser; und du selbst hast in den früheren Sitzungen App Control als tragende Schicht festgehalten. Preis: Im Wartungsfenster muss die Vertrauensperson Installer in einen Admin-Ordner kopieren, und Python-Umgebungen brauchen eine eigene Regel (8.4).

Alternative, falls dir App Control zu aufwendig ist: nur AppLocker mit den Standardregeln, Ausnahmen für die beschreibbaren Unterordner unter `C:\Windows` (Liste als Prüfschritt in Dokument 2) und Verweigerungsregeln für `mshta.exe`, `hh.exe`, `wscript.exe`, `cscript.exe`. Restrisiko dann: dokumentierte AppLocker-Umgehungen über signierte Windows-Programme, die Code laden (Microsoft nennt AppLocker deshalb kein Sicherheitsfeature [Q43]).

### 5.4 Firewall ausgehend

Standard ausgehend ist Allow; Block-Regeln haben Vorrang vor Allow-Regeln; Wildcards in Programmpfaden gibt es nicht; die Standardaktion lässt sich auf Block setzen, was Microsoft für Umgebungen mit strengen Kontrollen als Option nennt [Q67][Q68][Q24]. Regeln können mit `-LocalUser` auf Benutzer-SIDs eingeschränkt werden [Q24].

Zwei Wege, beide in Dokument 2 beschrieben, Entscheidung E6:

| | Blockliste je Programm | Standard Block mit Freigabeliste |
|---|---|---|
| Regeln | Block ausgehend für `python.exe`, `pythonw.exe`, `java.exe`, `javaw.exe`, `powershell.exe`, `pwsh.exe`, `curl.exe`, `certutil.exe`, `bitsadmin.exe`, `mshta.exe`, `wscript.exe`, `cscript.exe`, `kate.exe` und den KIO-Worker, `msedgewebview2.exe` (Pfad enthält die Versionsnummer, Regel bricht bei jedem Update), Git-Hilfsprogramme (`curl.exe`, `ssh.exe` unter `C:\Program Files\Git`) | Standardaktion ausgehend Block; Allow für `firefox.exe`, Cold Turkey, `svchost.exe` je Dienst (Windows Update, BITS, DoSvc, Dnscache, w32time, Lizenzierung), `MsMpEng.exe`, weitere nach Audit |
| Schließt | nur die aufgezählten Programme; jedes vergessene Windows-Programm mit Netzfunktion bleibt offen (Microsoft liefert allein in `System32` mehrere Dutzend) | alles, was nicht freigegeben ist, auch WebView2 unabhängig vom Pfad, auch unbekannte Programme |
| Risiko | Lücke durch Unvollständigkeit | Windows-Funktionen ohne Allow-Regel schlagen still fehl; Einrichtung braucht einen Audit-Lauf mit Firewall-Protokoll (verworfene Pakete) |
| Admin-Wartung | Block-Regeln mit `-LocalUser` auf dein Konto beschränken, dann bleibt `pip` für die Vertrauensperson nutzbar | Allow-Regeln für `python.exe` mit `-LocalUser` Admin |

Empfehlung: Standard Block mit Freigabeliste. Nicht belegt ist, ob die Windows-Firewall Loopback-Verkehr filtert; eine Microsoft-Aussage dazu wurde nicht gefunden. Deshalb steht in Dokument 2 ein Prüfschritt: lokaler Python-Webserver auf `127.0.0.1`, Aufruf in Firefox, erwartet erreichbar. Falls Loopback gefiltert wird, kommt eine Allow-Regel für `127.0.0.1` und `::1` hinzu. Der Weg „lokaler Dienst holt Inhalte aus dem Internet und reicht sie an Firefox durch“ ist in beiden Varianten geschlossen, weil der lokale Dienst selbst keinen Internetzugang hat; `localhost` bleibt in Cold Turkey als Ausnahme erlaubt (Abschnitt 4.1).

### 5.5 Edge und WebView2

Edge lässt sich im EWR über Einstellungen > Apps > Installierte Apps deinstallieren (Microsoft-Ankündigung zur DMA-Umsetzung, Windows 11 23H2 Build 22631.3235) [Q38]; Nutzer berichteten 2024 von einer schrittweisen Verteilung [Q26]. Was mit dem Protokoll `microsoft-edge:` nach der Deinstallation geschieht, ist nicht belegt; mit gesperrtem `msedge.exe` (Cold Turkey, AppLocker-Verweigerung) ist der Handler ohnehin tot. WebView2 ist Teil von Windows 11, wird von Widgets, Teams, Office, Outlook, Wetter und beliebigen weiteren Apps genutzt, bleibt bei Edge-Deinstallation erhalten und lässt sich nach Installation durch Edge Update nicht deinstallieren [Q40][Q79]. Edge-Browserrichtlinien gelten nicht für WebView2, und WebView2 hat keine eigene URL-Sperrliste [Q39][Q41]. Folgerung (Entscheidung E4): `msedgewebview2.exe` wird nicht gesperrt (Prozesse anderer Apps würden sterben, Einstellungen-Dialoge betroffene Apps nicht belegt); der Internetzugang wird ihm über die Firewall genommen (bei Standard Block automatisch), und die Apps, die ihn als Fenster nutzen, bleiben in Cold Turkey gesperrt (3.3).

### 5.6 Wiederherstellungsumgebung, abgesicherter Modus, Systemzeit

WinRE ist über Umschalt+Neustart, Einstellungen > Wiederherstellung und nach zwei Fehlstarts erreichbar; für Werkzeuge aus Windows heraus verlangt Microsoft Name und Kennwort eines lokalen Adminkontos, schreibt aber an anderer Stelle, die meisten Werkzeuge liefen ohne Adminkonto [Q56]. Nicht belegt ist damit, ob ein Standardbenutzer in WinRE eine Eingabeaufforderung mit Vollzugriff erhält. Geschlossen wird der Weg unabhängig davon durch BitLocker mit TPM und PIN: ohne PIN oder Wiederherstellungsschlüssel ist das Laufwerk in WinRE nicht lesbar [Q57]; der Wiederherstellungsschlüssel liegt bei der Vertrauensperson. Der abgesicherte Modus ist in der Cold-Turkey-Doku nicht erwähnt; eine Community-Meldung auf Microsoft Q&A beschreibt, dass dort der Dienst beendet wurde, mit Adminrechten (keine Herstellerquelle; Abschnitt 4.6). Unter Variante A braucht der abgesicherte Modus zum Deaktivieren von Diensten Adminrechte; Restrisiko siehe Matrix B13. Systemzeit: Cold Turkey „Block time changes“ plus FAQ-Empfehlung, dem Konto das Recht zur Zeitänderung zu entziehen [Q17]; unter Variante A hat ein Standardbenutzer das Recht nicht.
## 6. macOS 14 und 15: Grundschutz und die Stellen, an denen sich die Versionen unterscheiden

Für macOS gibt es keine Vorlage; die Edge-Dokumentation deckt nur Windows ab. Dieser Abschnitt legt fest, welche Schichten Dokument 3 aufbaut, und markiert, wo 14 und 15 auseinandergehen.

### 6.1 Konten

Administratoren verwalten Benutzer, installieren Apps und ändern Einstellungen; Standardbenutzer können Apps installieren und eigene Einstellungen ändern, aber keine Benutzer anlegen und keine fremden Einstellungen ändern (Apple-Handbuch 14 und 15, identischer Text) [Q89]. „Apps installieren“ heißt für einen Standardbenutzer: in den eigenen Benutzerordner, etwa `~/Applications`; dass `/Applications` nur für die Gruppe `admin` beschreibbar ist, konnte auf einer Apple-Supportseite nicht belegt werden und steht in Dokument 3 als Prüfschritt (`ls -ld /Applications`). Folge: Die Anwendungssteuerung muss den Benutzerordner abdecken (6.3). Gastkonto: unter „Benutzer:innen & Gruppen“ abschalten, Admin nötig [Q90]. Manuell installierte Konfigurationsprofile kann ein Benutzer nur entfernen, wenn er Name und Kennwort eines lokalen Admins kennt [Q91]; die Handbuchseiten 14 und 15 sagen gleichlautend, einige Profile könnten nur von einem Administrator entfernt werden [Q92].

### 6.2 Firefox-Richtlinien unter macOS

Firefox liest Richtlinien auf macOS aus der Defaults-Domain `org.mozilla.firefox`, nur wenn dort `EnterprisePoliciesEnabled` gesetzt ist; der Key wird in `policies.json` nicht ausgewertet und erzeugt dort einen Fehler in `about:policies` [Q14][Q93]. Admin-Quellen (`/Library/Preferences/org.mozilla.firefox.plist`, Geräteprofile, `policies.json` im Bundle) haben Vorrang vor Benutzerquellen (`~/Library/Preferences/org.mozilla.firefox.plist`, Benutzerprofile); ein Benutzer kann aber Richtlinien setzen, die kein Admin gesetzt hat [Q12]. Die Richtlinie, die Benutzerquellen abschaltet (`DisableMacOSUserPolicies`), gibt es erst ab Firefox 160 und nicht in ESR [Q12]. Restrisiko (Matrix F23): Ein Benutzer kann mit eigener Plist zusätzliche Richtlinien setzen, keine gesetzten überschreiben; eine Prüfung aller Richtlinien, die ein Benutzer gewinnbringend hinzufügen könnte, ergibt keine mit Ausbruchswirkung, weil jede sicherheitsrelevante Richtlinie in Dokument 3 vom Admin gesetzt wird und dann Vorrang hat.

Wahl des Mechanismus für Dokument 3: Konfigurationsprofil (`.mobileconfig`, Geräte-Scope, Einstellungen unter `mcx_preference_settings` in der Domain `org.mozilla.firefox`) [Q14]. Begründung: Ein Profil überlebt Firefox-Updates, weil es außerhalb des App-Bundles liegt; es ist nur mit Admin-Kennwort entfernbar [Q91]; `about:policies` zeigt die Herkunft. `policies.json` im Bundle wird nicht genutzt: Mozilla dokumentiert nicht, was mit der Datei bei einem Update des Bundles passiert (Bug 1792595 blieb unaufgeklärt), und das Einfügen in ein signiertes Bundle hat in einem Community-Fall eine „beschädigt“-Warnung ausgelöst [Q94][Q95]. `defaults write` in `/Library/Preferences` bleibt als Verifikationsweg (`sudo defaults read`), nicht als Hauptweg.

Unterschied 14 gegen 15 beim Profil installieren: macOS 14: „Systemeinstellungen“ > „Datenschutz & Sicherheit“ > „Profile“; macOS 15: „Systemeinstellungen“ > „Allgemein“ > „Geräteverwaltung“ [Q92][Q28]. Die Schritte danach sind gleich: Profil öffnen, unter „Geladen“ doppelklicken, „Fortfahren“, „Installieren“.

Firefox-Updates: Die App in `/Applications` gehört nach der Installation durch den Admin dem Admin; ein Standardbenutzer kann sie nicht überschreiben, und das Update verlangt Admin-Anmeldedaten (Aussage eines Mozilla-Mitarbeiters in Bug 1893686, keine Supportseite) [Q96]. Folge: Firefox-Updates im Wartungsfenster (E2); `ManualAppUpdateOnly` unterdrückt die Hinweise an den Benutzer [Q4].

Installationsmedium: Mozilla bietet für ESR auf macOS nur das DMG an, kein PKG [Q97]. Prüfsummen: Mozilla hält `SHA256SUMS` und `KEY` im Versionsverzeichnis unter `releases.mozilla.org/pub/firefox/releases/<Version>/` bereit; belegt nur durch Community-Antworten, das Verzeichnis für 153.6.0esr war aus dieser Umgebung nicht abrufbar [Q98][Q99]. Prüfschritt in Dokument 2 und 3.

Zweite Firefox-Installation: Developer Edition und Nightly haben eigene Bundle-IDs (`org.mozilla.firefoxdeveloperedition`, `org.mozilla.nightly`); Richtlinien in der Domain `org.mozilla.firefox` gelten dort nicht [Q100]. Geschlossen wird der Weg nicht durch Firefox, sondern durch die Anwendungssteuerung (6.3) und dadurch, dass Cold Turkey Browser ohne aktive Erweiterung schließt und seine Freigabeliste auch in einem Zweitbrowser mit Erweiterung durchsetzt [Q16]. Unter Windows lesen Developer Edition und Nightly denselben Registry-Schlüssel `SOFTWARE\Policies\Mozilla\Firefox` (Code-Ableitung über den gemeinsamen Anwendungsnamen) [Q101].

### 6.3 Anwendungssteuerung: Gatekeeper, Quarantäne, Santa

Gatekeeper prüft Software, die außerhalb des App Store heruntergeladen und geöffnet wird, beim ersten Öffnen; Benutzer können die Prüfung übergehen, sofern kein Gerätemanagement das verhindert [Q27]. Unterschied 14 gegen 15: In macOS 15 entfällt das Übergehen per Ctrl-Klick, die Freigabe geht nur noch über „Systemeinstellungen“ > „Datenschutz & Sicherheit“ > „Dennoch öffnen“ [Q37][Q102]. Die Einstellung „Apps erlauben, die geladen wurden von“ („App Store“ oder „App Store und verifizierte Entwickler“) lässt sich per Konfigurationsprofil (`com.apple.systempolicy.control`, Keys `AllowIdentifiedDevelopers`, `EnableAssessment`) festsetzen; Apple führt die Payload mit „Allow manual install: macOS“, also ohne MDM installierbar [Q103].

Die Lücke, die der Auftrag benennt: Gatekeeper prüft nur Dateien mit Quarantäne-Attribut; das Attribut setzen Apps, die Downloads als solche kennzeichnen (Browser, sandboxed Apps), nicht aber Unix-Werkzeuge wie `curl` oder `scp` und nicht Programme, die ein Skript selbst schreibt. Apple-Beleg nur indirekt (Property-List-Key `LSFileQuarantineEnabled`, Aussagen von Apple-DTS im Entwicklerforum) [Q104][Q105]. Ob ein Standardbenutzer das Attribut mit `xattr -d` entfernen darf, ist nicht belegt; Apples eigene Anleitung nutzt den Befehl ohne `sudo` [Q105]. Folge: Gatekeeper allein hält keinen Benutzer auf, der ein Programm ohne Quarantäne auf die Platte bringt. Mit den anderen Schichten sieht es so aus:

| Weg zu einem fremden Programm im Benutzerordner | Schließt |
|---|---|
| Download mit Firefox | Cold Turkey und `WebsiteFilter` lassen nur freigegebene Seiten zu; was von dort kommt, trägt das Quarantäne-Attribut und scheitert an Gatekeeper „App Store“ (Profil) |
| Download mit `curl`, Python, Java, Git | Firewall ausgehend (6.4): diese Programme haben keinen Internetzugang |
| App Store | Prüfschritt: Restriktion „Require administrator password to install or update apps“ aus Apples Restriktionsliste (OS X 10.9, ohne Betreuung) als Profil-Key setzen; zusätzlich Cold Turkey „Add application“ für App Store [Q106] |
| Mitgebrachter Datenträger, AirDrop, E-Mail-Anhang | Quarantäne-Attribut wird von Mail und Finder-Kopien aus Downloads gesetzt; für Datenträger nicht belegt. Restrisiko, siehe Matrix B5 (macOS) |
| Programm aus eigenem Code geschrieben (Python schreibt eine Mach-O-Datei) | ohne Netz kann Python nur schreiben, was es schon hat; ein vollständiger Browser entsteht so nicht. Restrisiko benannt (Matrix I8) |

Santa (northpolesec) im Lockdown-Modus würde diese Lücke schließen, denn er erlaubt nur Binaries nach Regel (CDHASH, BINARY, SIGNINGID, CERTIFICATE, TEAMID) und blockiert alles Unbekannte; er prüft aber keine Skripte, braucht Admin für Paket, Systemerweiterung und Vollzugriff auf die Festplatte, und seine Konfiguration ist als MDM-Profil dokumentiert, eine manuelle Profilinstallation ohne MDM nicht [Q107][Q108][Q109][Q110]. Apples eigene App-Allowlist gibt es erst für betreute Geräte mit macOS 27 [Q111]. Bewertung (Entscheidung E9): Santa ist die einzige echte Allowlist für macOS 14 und 15, aber mit zwei ungeprüften Voraussetzungen (Profil ohne MDM, `santactl`-Regeln lokal mit `StaticRules`). Empfehlung: zunächst ohne Santa, mit Gatekeeper-Profil, Firewall und Cold-Turkey-Ordnersperren für `~/Applications` und `~/Downloads` („Add folder“ sperrt alle Apps in einem Ordner [Q16]; Prüfschritt, ob neu angelegte Apps darunter erfasst werden). Santa als Prüfschritt mit eigener Anleitung in Dokument 3, Anhang, falls die Ausbruchsprüfung über `~/Applications` scheitert.

### 6.4 Firewall ausgehend

Die eingebaute Firewall regelt eingehende Verbindungen; ausgehende Verbindungen erwähnt Apple nicht [Q112][Q113]. Menüpfad 14 und 15 identisch: „Systemeinstellungen“ > „Netzwerk“ > „Firewall“ [Q112]. Für ausgehende Regeln je Programm:

| | LuLu 4.5.3 (Objective-See, Open Source, kostenlos) | Little Snitch 6 (Objective Development, kostenpflichtig) |
|---|---|---|
| Ausgehend je Programm | ja, nur ausgehend; Erkennung signierter Programme über die Code-Signatur [Q114] | ja; Regeln hängen an der Code-Identität (Team-ID, Identifier), nicht am Pfad, gelten also nach Verschieben und Update weiter; unsignierte Programme über Hash [Q115] |
| Schutz vor dem Benutzer | Beenden und Deinstallation verlangen Authentifizierung; zu Regeländerungen durch Standardbenutzer sagt die Doku nichts [Q114] | Sicherheitseinstellungen hinter Schloss mit Administratorkennwort: „Allow Rule Editing“, „Allow Profile Switching“, „Allow Settings Editing“, „Allow Global Rule Editing“, „Allow access via Terminal“ lassen sich abschalten [Q116] |
| Installation | Systemerweiterung und Netzwerkfilter brauchen Admin-Freigabe [Q117] | gleich; App nach `/Applications` [Q118] |
| Deinstallation | Authentifizierung [Q114] | App in den Papierkorb; ob dabei ein Kennwort verlangt wird, steht nicht in der Doku [Q119] |
| venv-Symlink auf den Framework-Interpreter | nicht belegt | nicht belegt; aus der Code-Identitäts-Logik folgt, dass ein Symlink auf denselben signierten Interpreter dieselbe Identität hat (Ableitung) |

Empfehlung: Little Snitch 6 mit „Allow Rule Editing“ aus und stillem Modus „Verbindungen verweigern“, sodass kein Dialog dem Benutzer eine Freigabe anbietet (Prüfschritt: Name des stillen Modus in Version 6 und Verhalten der temporären Regeln aus dem Alarm). LuLu nur, wenn Kosten entscheidend sind; dann steht der fehlende Beleg zum Regelschutz als Restrisiko. Beide Werkzeuge werden vom Admin installiert; ein Standardbenutzer kann eine Systemerweiterung nicht freigeben [Q117]. Ausgehend erlaubt: Firefox, Cold Turkey, Apple-Systemdienste; verweigert: alles andere, insbesondere `python3`, `java`, `curl`, `git`, Kate samt KIO-Worker, Safari.

### 6.5 Bildschirmzeit

Menüpfad 14: „Systemeinstellungen“ > „Bildschirmzeit“; Webinhalte unter „Inhalt & Datenschutz“ > „Store-, Web-, Siri- und Game Center-Inhalte“. Menüpfad 15: gleich bis „Inhalt & Datenschutz“, dann „App Store, Medien, Web & Spiele“; die Zeile ist dort mit „Safari“ beschriftet [Q120][Q121]. Optionen auf beiden: „Unbeschränkter Zugriff“, „Nicht jugendfreie Inhalte beschränken“, „Nur erlaubte Websites“. Ob der Filter andere Browser als Safari betrifft, sagt Apple nicht. App-Beschränkungen gibt es auf dem Mac nur für Kamera, Book Store, Siri & Diktat und SharePlay; Safari lässt sich darüber nicht abschalten [Q122].

Schwachstelle: Der Bildschirmzeit-Code ist vierstellig, und die Apple-Seiten 14 und 15 beschreiben, dass Ändern oder Entfernen des Codes mit dem Code oder mit Passwort bzw. Touch ID möglich ist [Q120][Q121]. Ob damit das Anmeldekennwort des Benutzerkontos gemeint ist, muss ein Prüfschritt klären; falls ja, hält die Bildschirmzeit im eigenen Konto den Benutzer nicht auf. Dokument 3 nutzt Bildschirmzeit deshalb nur als zweite Schicht für Safari („Nur erlaubte Websites“ mit leerer Liste), tragend bleiben Cold Turkey („Add application“ Safari) und die Firewall (Safari ohne Netz). Cold Turkey bietet auf dem Mac eine Option, die Bildschirmzeit-Einstellungen zu sperren (nur auf der Features-Seite genannt) [Q31].

### 6.6 Safari und Standardbrowser

Safari ist durch SIP geschützt und nicht deinstallierbar [Q123]. Maßnahmen: Cold Turkey „Add application“ Safari; Firewall verweigert Safari ausgehend; Bildschirmzeit als zweite Schicht; Firefox als Standardbrowser über „Systemeinstellungen“ > „Schreibtisch & Dock“ > „Standard-Webbrowser“ [Q124]. Cold Turkey unterstützt Safari ohne Block-Seite und braucht ab Safari 17 Vollzugriff auf die Festplatte, um die Safari-Erweiterung zu prüfen [Q18][Q36]; da Safari ohnehin gesperrt wird, wird die Safari-Erweiterung nicht konfiguriert [Q75]. Eine verwaltete Safari-Konfiguration gibt es erst ab macOS 26 [Q125].

### 6.7 Wiederherstellung und sicherer Modus

Apple Silicon: Einschalttaste halten, „Optionen“; Intel: Befehl-R; danach einen Benutzer wählen, dessen Kennwort bekannt ist [Q126]. Apple Silicon unterstützt kein Firmware-Passwort; ein Wiederherstellungs-Sperrkennwort gibt es nur per MDM; Änderungen an der Startsicherheitsrichtlinie verlangen einen Benutzer, der Admin und Volume-Besitzer ist [Q127][Q128]. Intel: Firmware-Passwort durch den Admin sperrt Recovery und sicheren Modus [Q129]. Nicht belegt ist, ob ein Standardbenutzer mit Secure Token in der Wiederherstellung ein Terminal mit Zugriff auf das entsperrte Datenvolume erhält und damit `/Library/Preferences` oder die LaunchDaemons von Cold Turkey ändern könnte. Das ist auf Apple Silicon das größte Restrisiko dieses Lockdowns (Matrix B14) und wird in Dokument 3 als erster Prüfschritt der Ausbruchsprüfung geführt. Deshalb Entscheidung E17 (Hardware).

Sicherer Modus: verhindert Anmeldeobjekte und nicht benötigte Erweiterungen [Q129]; ob LaunchDaemons wie der Cold-Turkey-Dienst oder Netzwerk-Systemerweiterungen dort laufen, ist nicht belegt. Firefox-Richtlinien und `WebsiteFilter` gelten im sicheren Modus weiter (Dateien außerhalb der App); Safari wäre ohne Cold Turkey und ohne Firewall offen. Prüfschritt in Dokument 3; bis zum Ergebnis Restrisiko B13 (macOS).

### 6.8 Homebrew, Python, Java, Kate unter macOS

Homebrew lässt sich laut Doku auch für ein Konto ohne Adminrechte installieren, mit eigenem Präfix; der Installer bricht ab, wenn das Präfix nicht beschreibbar ist und kein `sudo` vorliegt [Q130][Q131]. Ohne Netz (6.4) kann Homebrew nichts laden; der Weg ist über die Firewall geschlossen, nicht über ein Verbot. Der python.org-Installer legt `/Library/Frameworks/Python.framework` und Symlinks in `/usr/local/bin` an und verlangt Adminrechte [Q132]; `venv` legt auf macOS Symlinks auf den Interpreter an (CPython-Quelle: Symlinks überall außer Windows) [Q25][Q133]. Apples `/usr/bin/python3` ist ein Stub für die Command Line Tools und wird nicht genutzt [Q132]. Kate für macOS: kate-editor.org bietet nur Nightly-DMGs von der KDE-CI an, „nur zum Testen empfohlen“ (aktuell `kate-master-12820-macos-clang-arm64.dmg`, 28.10.2026) [Q134][Q135]; ob das DMG signiert und notarisiert ist, ist für Kate nicht belegt, KDEs CI kann seit Ende 2024 notarisieren [Q136]. Das ist eine offene Stelle für Dokument 3 (Abschnitt 7.2).
## 7. Editorwechsel und Interpreter: Eclipse zu Kate, Python und Java freigegeben

### 7.1 Was von Eclipse entfällt und was bleibt

Alle vier Eclipse-Maßnahmen der Edge-Dokumentation entfallen (3.4). Der Eclipse JDT Language Server bringt seine Eclipse-Plattform als eigenständiges Paket mit (`plugins/`, `config_win`, `config_mac`, `bin/`) und hängt nicht an einer Eclipse-IDE [Q69]. Eclipse-Reste nach der Deinstallation: siehe 3.4; Dokument 2 enthält die Liste als Prüfschritt im Explorer.

### 7.2 Kate: Version und Bezug

Aktuelle stabile Version: Kate aus KDE Gear 26.08.2 vom 8. Oktober 2026 [Q141][Q142].

| | Windows | macOS |
|---|---|---|
| Quellen laut kate-editor.org | Microsoft Store (Herausgeber KDE e.V., Store aktualisiert automatisch) und Nightly-Installer von der KDE-CI [Q65][Q143] | nur Nightly-DMGs (ARM und Intel) von der KDE-CI, „nur zum Testen“ [Q65] |
| Release-Builds auf dem KDE-CDN | Ordner `release-26.08/windows/`: `kate-release_26.08-12761-windows-cl-msvc2022-x86_64.exe` mit `.sha256`, dazu `.7z` und `-sideload.appx` (19.10.2026) [Q144] | Ordner `release-26.08/macos-arm64/`: `kate-release_26.08-12761-macos-clang-arm64.dmg`, ohne Prüfsummendatei [Q145] |
| Installationsumfang | Store: je Benutzer nach `C:\Program Files\WindowsApps`, schreibgeschützt und vom System gesperrt [Q66]; EXE-Installer: NSIS MultiUser mit Auswahlseite „alle Benutzer“ (Program Files) oder „nur aktueller Benutzer“ (lokales AppData) (Craft-Installervorlage und NSIS-Doku) [Q146][Q147] | DMG mit Ziehen nach `/Applications` durch den Admin [Q148] |
| Signatur | nicht belegt für EXE und APPX; Store-Pakete sind durch den Store signiert | Notarisierung für Kate nicht belegt; Wunschfehler 470738 offen, letzte Meldung 04.01.2026 [Q149]; KDEs Infrastruktur kann notarisieren [Q136] |
| Empfehlung | Microsoft Store. Grund: signiert, automatische Updates, Installationsordner für den Benutzer unbeschreibbar [Q66]. Die Installation erfolgt in deiner Sitzung, bevor der Store für dich gesperrt wird; Store-Hintergrundupdates laufen ohne die Store-App (Prüfschritt in Dokument 2, Betrieb). AppLocker-Packaged-App-Regel erlaubt den Herausgeber des Kate-Pakets (Prüfschritt: Herausgeberstring aus `Get-AppxPackage`). Rückfall: EXE-Installer aus dem Release-Ordner mit Prüfsumme, „alle Benutzer“, Signaturprüfung als Prüfschritt. | Release-DMG aus `release-26.08/macos-arm64/` (bzw. `macos-x86_64/`) durch den Admin nach `/Applications`; Gatekeeper-Freigabe unter macOS 15 über „Datenschutz & Sicherheit“ > „Dennoch öffnen“, falls nicht notarisiert (Prüfschritt); danach ist die App ohne Quarantäne-Attribut und für dich nicht ersetzbar, weil `/Applications` dem Admin gehört (Prüfschritt 6.1). Prüfsumme: keine vom Projekt; Dokument 3 lässt den Admin den SHA-256 nach dem Download notieren, damit spätere Vergleiche möglich sind. |

Hinweis für macOS: Eine notarisierte, stabile Kate-Version gibt es derzeit nicht. Das ist kein Sicherheitsproblem für den Lockdown (die App wird vom Admin installiert und liegt in einem Ordner, den du nicht beschreibst), aber ein Stabilitätsrisiko: Die Release-Branch-Builds sind CI-Builds. Alternative, die Dokument 3 als Option nennt: MacPorts oder Homebrew-Installation durch den Admin (nicht geprüft, Prüfschritt).

### 7.3 Python

Aktuelle stabile Version: 3.15.2 vom 25. Oktober 2026 [Q150]. Windows: klassischer Installer `python-3.15.2-amd64.exe`, „Customize installation“, „Install for all users“ nach `C:\Program Files\Python 3.15`, Launcher für alle Benutzer; der Installer ist seit 3.14 als veraltet markiert und wird für 3.16 nicht mehr erzeugt (Entscheidung E13) [Q63]. App-Ausführungsaliase: Der Store-Installationsmanager registriert `python`, `python3`, `py`, `pymanager`, `pyw`, `pythonw` als Aliase; sie werden in den Einstellungen auf der Seite „App-Ausführungsaliase“ abgeschaltet (Microsoft nennt die Seite so, der vollständige deutsche Menüpfad ist nicht belegt; Prüfschritt) [Q74][Q63]. macOS: `python-3.15.2-macos11.pkg`, signiert und notarisiert mit PSF-Developer-ID, installiert nach `/Library/Frameworks/Python.framework` mit Symlinks in `/usr/local/bin`, Adminrechte nötig [Q132][Q150].

Language Server: `python-lsp-server` 1.15.0 (27.07.2026), reine Python-Implementierung auf Jedi-Basis, Installation `pip install "python-lsp-server[all]"`, keine Node.js-Abhängigkeit [Q140][Q151]. Alternativen und Gründe gegen sie: `pyright` lädt zur Laufzeit Node.js und das npm-Paket nach (scheitert ohne Netz, und Node wäre ein weiterer Interpreter) [Q152]; `basedpyright` bringt Node.js als Wheel mit (zweiter Interpreter ohne Firewall-Regel nach Name) [Q153]; `jedi-language-server` ist eine schlankere Alternative ohne Node, aber mit weniger Plugins [Q154]. Debugging: `debugpy` 1.8.22, reines Python; Kates Debugger-Plugin bringt ein `debugpy`-Profil mit, ist aber laut Quelltext nicht nach Windows portiert [Q155][Q156]. Folge für den Nutzbarkeitstest: Unter Windows wird Python im Terminal mit `python -m pdb` oder `breakpoint()` debuggt, unter macOS zusätzlich über das Debugger-Plugin (Prüfschritt). Das steht ehrlich in Dokument 2, Abschnitt 9.

Zweck-Regel für Kate: `pylsp` wird mit dem Basisinterpreter installiert (im Wartungsfenster, systemweit mit `pip` des Adminkontos nach `C:\Program Files\Python 3.15\Scripts` bzw. `/Library/Frameworks/.../bin`); die venv des Benutzers braucht den Server nicht, Kate startet `pylsp` aus dem Systempfad.

### 7.4 Java

Eclipse JDT Language Server 1.63.0 vom 19. Oktober 2026 (`jdt-language-server-1.63.0-202610191322.tar.gz` mit `.sha256`) [Q70][Q157]. Voraussetzung laut README des Tags 1.63.0: Java 21 oder neuer; die README des Hauptzweigs verlangt seit dem 30. Oktober 2026 Java 25 für kommende Versionen [Q69][Q158]. Entscheidung: Eclipse Temurin JDK 25 LTS (25.0.5+8): Windows-MSI installiert je Maschine nach `C:\Program Files\Eclipse Adoptium\`, macOS-PKG nach `/Library/Java/JavaVirtualMachines/`, beides durch den Admin [Q159][Q160][Q161]. Temurin-Binaries sind mit dem Code-Signing-Zertifikat der Eclipse Foundation signiert [Q162].

Start: Das mitgelieferte `bin/jdtls` ist ein Python-Skript (Python 3.9 oder neuer), das `java` aus `JAVA_HOME` sucht, `config_win`/`config_mac` wählt und `-data` auf einen Benutzerordner legt [Q69][Q163]. Kates Standardkonfiguration ruft `jdtls` als nacktes Kommando auf [Q164]. Dokument 2 und 3 legen den vollständigen `java`-Aufruf (Equinox-Launcher, `-configuration` auf eine Kopie im Benutzerordner, `-data` je Projekt) direkt in Kates LSP-Einstellungen ab, damit kein Python-Skript dazwischen liegt und der Pfad eindeutig ist.

Netz: Für Ordner mit `.java`-Dateien ohne Build-Werkzeug nutzt JDT LS ein „unsichtbares Projekt“ und erkennt Jars in `lib/` [Q71]. Maven- und Gradle-Projekte lösen Abhängigkeiten über M2Eclipse und Buildship; der Gradle-Wrapper lädt die Gradle-Distribution herunter, wenn sie nicht im Cache liegt [Q165]; es gibt Offline-Schalter (`java.import.maven.offline.enabled`, `java.import.gradle.offline.enabled`) [Q71]. Ohne Netz (8.1) bleibt es bei Projekten ohne Build-Werkzeug oder mit Abhängigkeiten, die der Admin im Wartungsfenster in `~/.m2` bzw. den Gradle-Cache legt (8.2). Eine Aussage „läuft vollständig offline“ gibt es in der Doku nicht; Prüfschritt.

### 7.5 Kate: Plugins und Ausbruchswege

Plugins werden unter „Einstellungen“ > „Kate einrichten…“ > „Anwendung“ > „Erweiterungen“ eingeschaltet (Handbuch) [Q166]; die Schalter liegen in `katerc`, Gruppe `[Kate Plugins]`, Schlüssel = Bibliotheksname (`lspclientplugin`, `katebuildplugin`, `externaltoolsplugin`, `katekonsoleplugin`, `kateprojectplugin`, `ktexteditorpreviewplugin`, `kategdbplugin`, `textfilterplugin`, `katesqlplugin`) [Q167]. Ort von `katerc`: Windows `%LOCALAPPDATA%\katerc`, macOS `~/Library/Preferences/katerc` (Qt-Standardpfade) [Q168].

| Weg über Kate | Was ihn schließt | Entscheidung |
|---|---|---|
| External Tools, Build & Run, Text Filter: beliebige Befehle | Anwendungssteuerung erlaubt nur Programme aus den Systemordnern; Firewall nimmt ihnen das Netz (8.1, 8.3) | freigeben |
| Terminal-Plugin (Konsole-Part; Windows-Backend ConPTY, im Windows- und macOS-Paket gebündelt, Lauffähigkeit im ausgelieferten Paket nicht belegt) [Q169][Q170] | gleich | freigeben; Prüfschritt, ob das Plugin im Store-Paket funktioniert; sonst Windows Terminal (`wt.exe`) |
| „Datei“ > „Öffnen“ mit `http://`-Adresse (KIO-Worker für `file`, `ftp`, `http` werden ohne Plattformbedingung gebaut und sind im Paket) [Q171] | Firewall: Kate und der KIO-Worker ohne Netz (Prüfschritt: Prozessname des Workers, erwartet `kioworker.exe` bzw. `kioworker`) | freigeben; kein Menü „Adresse öffnen“, die Funktion liegt im normalen Öffnen-Dialog |
| Vorschau-Plugin | nutzt den KPart für den MIME-Typ; im Paket ist `markdownpart`, ein Wrapper um `QTextBrowser` ohne Web-Engine [Q172][Q173] | freigeben |
| Abgeschaltete Plugins wieder einschalten, `katerc` bearbeiten | kein Plugin öffnet einen Weg, den 8.1 und 8.3 nicht schließen | kein Sperren nötig; deshalb keine Kiosk-Einträge im Hauptweg |
| Sitzungen, portable Kate in Benutzerordnern | Sitzungen sind Benutzerdaten ohne Sicherheitsbezug; eine portable Kate läuft unter Windows nicht (8.3), unter macOS ohne Netz | kein Sperren nötig |
| SQL-Plugin (Verbindung zu Datenbankservern) | Firewall | freigeben |
| LSP-Client startet fremde Kommandozeile | Kate fragt vor dem ersten Start eines Servers und speichert die Entscheidung in `[lspclient]` `AllowedServerCommandLines`/`BlockedServerCommandLines` [Q167]; gestartet wird nur, was die Anwendungssteuerung erlaubt | freigeben; Dokument 2 und 3 belegen die Listen mit `pylsp` und dem `java`-Aufruf vor |
| KI-Vervollständigung | im Addon-Baum von 26.08 gibt es kein Plugin mit KI-, Copilot- oder LLM-Bezug; Vervollständigung kommt aus KTextEditor, LSP, Project/CTags, Snippets [Q174] | nichts abzuschalten |

Kiosk-Mechanismus: KConfig wertet `[$i]`-Einträge plattformneutral aus und liest systemweite Dateien über die Qt-Standardpfade, unter Windows `C:\ProgramData\katerc` und `C:\ProgramData\kdeglobals`, unter macOS `/Library/Preferences/katerc` und `/Library/Preferences/kdeglobals` (Qt-Pfadtabelle und KConfig-Quelltext; keine Doku-Aussage für Windows oder macOS) [Q168][Q175][Q176]. Die Einschränkung `[KDE Action Restrictions][$i]` mit `shell_access=false` wird vom Terminal-, External-Tools- und Project-Plugin beachtet, nicht vom Build-, Text-Filter-, LSP- oder Debugger-Plugin [Q177]. Da nach <grundsatz> kein Kate-Plugin gesperrt werden muss, kommt Kiosk nur als Option in die Anleitungen (Anhang, mit Prüfschritt), nicht in den Hauptweg; Dateirechte auf die Benutzerkonfiguration werden nicht gesetzt, weil sie Sitzungen und Fensterlayout bräche und nichts schlösse.

### 7.6 LSP-Konfiguration

Kates Standardserverliste in 26.08 enthält für Python `pylsp --check-parent-process` und für Java `jdtls` [Q164]; die Benutzerdatei liegt unter Windows in `%LOCALAPPDATA%\kate\lspclient\settings.json`, unter macOS in `~/Library/Preferences/kate/lspclient/settings.json` [Q167]. Die Einstellungsseite bietet unter anderem „Enable semantic highlighting“, „Show program diagnostics“, „Incrementally synchronize documents with the LSP server“ und den Reiter „Allowed && Blocked Servers“ [Q178]. Die Handbuchseite zum LSP-Client beschreibt `command`, `root`, `rootIndicationFileNames`, `highlightingModeRegex`, `initializationOptions`, `settings` [Q179]. Der vollständige Codeblock mit beiden Servern steht in Dokument 2 und 3, Abschnitt 9.5; der Prüfschritt dort: je eine Python- und eine Java-Datei mit Vervollständigung, Fehleranzeige und Sprung zur Definition.

### 7.7 Git

Git for Windows 2.56.0(4) vom 26. Oktober 2026, Installer `Git-2.56.0.4-64-bit.exe`, Standardziel `C:\Program Files\Git`, Installation als Admin für alle Benutzer; das Paket enthält Bash, cURL 8.22.0 und OpenSSH [Q180][Q181][Q182]. Die mitgelieferten Netzwerkprogramme (`curl.exe`, `ssh.exe`, `git-remote-https.exe`) fallen unter die Firewall-Standardaktion Block (8.1); Git arbeitet lokal (E16). macOS: Git über die Command Line Tools (`xcode-select --install`) durch den Admin [Q183]. Kates Project-Plugin ruft `git` aus dem Pfad auf [Q184].

### 7.8 Cold-Turkey-Freigaben für Editor, Interpreter und Language Server

Nicht sperren (Windows): `kate.exe` (Store-Paket, Prüfschritt zum Prozessnamen), `kioworker.exe`, `python.exe`, `pythonw.exe`, `py.exe`, `pylsp.exe` (pip-Startprogramm in `Scripts`), `java.exe`, `javaw.exe`, `cmd.exe`, `powershell.exe`, `wt.exe`, `conhost.exe`, `git.exe`, `bash.exe`, `sh.exe`, `git-bash.exe`. Nicht sperren (macOS): `Kate.app` und ihre Hilfsprozesse, `python3` (Framework), `pylsp`, `java`, `Terminal.app`, `git`. Cold Turkey Mac „Add binary“ wird für nichts davon genutzt.
## 8. Ausgleichsmaßnahmen für freigegebene Interpreter und alles, was Code ausführt

Jede Maßnahme mit Wirkung, Beleg und Prüfschritt; Windows und macOS getrennt, wo sie sich unterscheiden.

### 8.1 Netz

| | Windows | macOS |
|---|---|---|
| Maßnahme | Firewall-Standardaktion ausgehend Block für alle Profile; Allow-Regeln für `firefox.exe`, `Cold Turkey Blocker.exe` und den Cold-Turkey-Dienst, `svchost.exe` je Dienst (Windows Update, BITS, Übermittlungsoptimierung, DNS-Client, Zeitdienst, Lizenzierung, Netzwerklistendienst), `MsMpEng.exe`; Allow-Regeln für `python.exe` und `pip` mit `-LocalUser` auf das Adminkonto (Wartungsfenster). Lokale Regelzusammenführung bleibt an, weil ein Standardbenutzer keine Regeln anlegen kann. | Little Snitch 6: Regeleditierung gesperrt, stiller Modus „verweigern“, Regeln: Firefox erlaubt, Cold Turkey erlaubt, Apple-Systemdienste erlaubt (vordefinierte Regelgruppe), alles andere verweigert |
| Wirkung | `python.exe`, `pythonw.exe`, `java.exe`, `javaw.exe`, die Language Server, `curl.exe`, `powershell.exe`, `pwsh.exe`, `certutil.exe`, `bitsadmin.exe`, `msedgewebview2.exe`, `mshta.exe`, `kate.exe`, der KIO-Worker, Git-Hilfsprogramme: kein Internet. Keine Pfadpflege nötig. | gleich, nach Code-Identität statt Pfad |
| Beleg | Q67 (Standardaktion Block als Option, Vorrang von Block), Q68 (`-DefaultOutboundAction`), Q24 (`-LocalUser`), Q44 (`mshta` zusätzlich per Skriptdurchsetzung) | Q115, Q116 |
| `localhost` | nicht belegt, ob Loopback gefiltert wird; Prüfschritt mit `python -m http.server --bind 127.0.0.1` und Aufruf in Firefox; falls nötig Allow-Regeln für `127.0.0.1/32` und `::1/128` | Little Snitch filtert lokale Verbindungen nach Regel; Prüfschritt gleich |
| Firefox-Proxy | `Proxy` `Mode: none`, `Locked: true` (D2/D3 §4) | gleich |
| Prüfschritt | `python -c "import urllib.request; urllib.request.urlopen('https://example.org')"` schlägt fehl; `curl.exe https://example.org` schlägt fehl; Firefox lädt eine freigegebene Seite; Windows Update findet Updates (Protokoll der verworfenen Pakete prüfen) | gleich mit `python3`, `curl`; Little-Snitch-Protokoll zeigt verweigerte Verbindungen |

Restrisiko: Ein Dienst, der im Audit vergessen wurde, fällt still aus (Windows); die Anleitung enthält die Audit-Prozedur (Firewall-Protokoll für verworfene Pakete eine Woche lang auswerten, dann Allow-Regeln ergänzen).

### 8.2 Paketinstallation

Vor dem Scharfschalten installiert der Admin: Python mit `pip`, `python-lsp-server` samt Plugins, Java-JDK, Eclipse JDT Language Server, Kate, Git. Danach gilt Entscheidung E8: Wartungsfenster plus lokaler Wheel-Ordner. Der Ordner liegt unter Windows in `C:\Program Files\Wheels` (nur Admin schreibt), unter macOS in `/Library/Wheels` (Besitzer root); `pip install --no-index --find-links <Ordner> <Paket>` installiert daraus offline, in die venv des Benutzers. Ablauf: Du führst eine Wunschliste (Textdatei im Ordner, für dich beschreibbar); die Vertrauensperson lädt im Wartungsfenster mit `pip download` in den Ordner. Das Wartungsfenster ist ein Termin, kein Zustand: Die Vertrauensperson meldet sich an, arbeitet die Liste ab, meldet sich ab. Beleg für `pip download` und `--no-index --find-links`: pip-Dokumentation (Prüfschritt in Dokument 2 und 3 mit Link; aus dieser Umgebung nicht gelesen).

Java: Für Projekte ohne Build-Werkzeug braucht JDT LS kein Netz (Prüfschritt mit einer `.java`-Datei). Maven und Gradle laden Abhängigkeiten aus dem Internet; ohne Netz schlägt der Import fehl. Lösung: Maven-Projekte mit Abhängigkeiten nur, wenn der Admin im Wartungsfenster das lokale Repository `~/.m2` befüllt (`mvn dependency:go-offline` im Wartungsfenster, Prüfschritt). Standardannahme laut Kontext: keine Abhängigkeiten.

### 8.3 Ausführung

Windows: App Control for Business (E14), Vorlage „Default Windows Mode“ (Windows-Komponenten, Store-Apps, WHQL-Treiber) plus Pfadregeln `C:\Program Files*` und `C:\Program Files (x86)*`; Option „Enabled:UMCI“ (Benutzermodus prüfen), Audit-Modus zuerst, dann Option 3 entfernen [Q45][Q46][Q47]. Pfadregeln gelten nur, wenn der Pfad zur Laufzeit nur für Admins beschreibbar ist, also nie für `C:\Users\...`, `C:\Windows\Temp` oder `C:\ProgramData`-Unterordner mit Benutzerrechten [Q45]. Das schließt portable Browser, heruntergeladene Programme, umbenannte Kopien, winget-Benutzerinstallationen, Python „Install Now“ im Benutzerordner. Skriptdurchsetzung bleibt an: PowerShell läuft für nicht erlaubte Skripte im Constrained Language Mode, `mshta` ist gesperrt [Q44]; `.py`-Dateien sind nicht betroffen, weil Python ein nicht erleuchteter Host ist [Q44]. AppLocker ergänzt je Benutzer: Verweigerung `msedge.exe` (Herausgeber Microsoft, Produkt Microsoft Edge), Packaged-App-Regel: Verweigerung `Microsoft.WindowsStore` und Standardregel für signierte Packaged Apps durch eine Regel nur für Microsoft-Herausgeber plus Kate (KDE e.V., Prüfschritt zum Herausgebernamen) ersetzen [Q22]. Prüfschritte: portable Firefox aus `Downloads` starten (erwartet: blockiert, Ereignis 3077 im CodeIntegrity-Protokoll, Ereignis-ID als Prüfschritt), Store-App „Microsoft Store“ starten (erwartet: AppLocker-Verweigerung), Kate aus dem Store starten (erwartet: läuft).

macOS: Lücke benannt (6.3): Programme ohne Quarantäne-Attribut laufen. Maßnahmen ohne Santa: Gatekeeper-Profil „App Store“ für quarantänierte Downloads; Firewall nimmt den Werkzeugen das Netz; Cold-Turkey-Ordnersperren `~/Applications`, `~/Downloads`, `~/Desktop` (Prüfschritt: Cold Turkey Mac „Add folder“ erfasst alle Apps im Ordner, auch neu angelegte?). Santa im Lockdown-Modus als geprüfte Option: Paket signiert und notarisiert, Installation per `sudo installer`, Systemerweiterung und Vollzugriff auf die Festplatte freigeben, Konfiguration als Profil (`ClientMode` 2, `StaticRules` mit Regeln für Apple, Firefox (Team-ID Mozilla), Cold Turkey, Kate, Python, Java, Little Snitch), `santactl rule` lokal durch `StaticRules` gesperrt [Q107][Q108][Q109][Q110]. Nicht belegt: ob das Profil ohne MDM wirkt. Bewertung: hoher Nutzen (einzige Allowlist), mittlerer Aufwand (Team-IDs ermitteln, Monitor-Modus eine Woche), eine ungeprüfte Voraussetzung. Entscheidung E9; Dokument 3 enthält Santa als Anhang mit Prüfschritt, nicht im Hauptweg.

### 8.4 Virtuelle Umgebungen

Windows: Seit Python 3.7.2 legt `venv` keine Kopie von `python.exe` an, sondern ein Umleitungsprogramm (`venvlauncher.exe` als `python.exe`, `venvwlauncher.exe` als `pythonw.exe`), das den Basisinterpreter aus `home` in `pyvenv.cfg` startet [Q25][Q64]. Folgen: (1) Die Firewall sieht den Basisinterpreter `C:\Program Files\Python 3.x\python.exe`; unter Standard Block ist das ohnehin gleichgültig. (2) App Control sieht den Starter im Benutzerordner und blockiert ihn. Lösung: Hash-Regeln für `venvlauncher.exe` und `venvwlauncher.exe` aus dem Python-Installationsordner (Dateihash unabhängig von Name und Ort); nach jedem Python-Update erneuern (Betrieb). Alternative Herausgeber-Regel (Python Software Foundation): Subject und Aussteller des Zertifikats sind nicht belegt, Prüfschritt mit `Get-AuthenticodeSignature`; wenn der Starter signiert ist, ersetzt eine Herausgeber-Regel mit Dateiname `venvlauncher.exe` die Hash-Regeln [Q23]. Prüfschritt: `python -m venv C:\Users\<du>\Projekte\venv`, `venv\Scripts\python.exe -c "import sys; print(sys.prefix)"` läuft; `venv\Scripts\python.exe -c "urlopen"` scheitert.

macOS: `venv` legt Symlinks an [Q25][Q133]; Prozess und Code-Identität sind die des Framework-Interpreters; Little-Snitch-Regel nach Code-Identität gilt weiter (Ableitung, Prüfschritt). Gatekeeper ist nicht betroffen (kein Download).

### 8.5 Eingebettete Web-Engines

Pakete, die eine Engine mitbringen (`pywebview`, `PyQt6-WebEngine`, `PySide6`, `cefpython`, JavaFX WebView, Electron über `electron`-npm), sind ohne Netz nicht installierbar (8.2). Werden sie im Wartungsfenster installiert, laufen ihre Renderer als Kindprozesse des Interpreters oder als eigene Prozesse ohne Allow-Regel: Standard Block (Windows) bzw. Verweigerung nach Code-Identität (macOS) nimmt ihnen das Netz. `pywebview` nutzt unter Windows WebView2 [Q40] (ohne Netz durch Standard Block), unter macOS WebKit (ohne Netz durch Little Snitch, Prüfschritt mit dem Prozessnamen des WebKit-Helpers). Beleg für die Netzsperre: 8.1. Restrisiko: lokale HTML-Inhalte sind anzeigbar; das ist kein Weg ins Internet.

### 8.6 Benutzerprofil

Programme des Benutzers können im Firefox-Profil alles ändern (`prefs.js`, `user.js`, Erweiterungsdaten, `profiles.ini`), die Kate-Konfiguration (`katerc`, Sitzungen, LSP-Benutzereinstellungen) und Autostart-Einträge (Aufgabenplanung, `~/Library/LaunchAgents`, Startmenü-Autostart). Deshalb liegt alles Sicherheitsrelevante außerhalb: Firefox-Richtlinien in `HKLM` oder im Geräteprofil, Erweiterungen erzwungen, Anwendungssteuerung und Firewall systemweit, Cold Turkey mit Passwort. Für Kate gilt (Abschnitt 7): Plugins, die der Benutzer wieder einschalten könnte, dürfen keinen Ausbruchsweg öffnen; die Kate-Konfiguration ist deshalb nicht sicherheitsrelevant.

### 8.7 Neubewertung der Sperrliste

Durch 8.1 und 8.3 werden überflüssig und freigegeben: alle Python-Prozesse, `cmd.exe`, `powershell.exe`, `wt.exe`, `winget.exe`, `DeploymentAgent.exe`, `regedit.exe`, `Taskmgr.exe`, `QuickAssist.exe`, `mspaint.exe`, `SnippingTool.exe`, `ScreenClippingHost.exe`, `SoundRec.exe`, `WindowsBackupClient.exe`, `AssignedAccessLockApp.exe`, `CHXSmartScreen.exe`, `WpcUapApp.exe`, `CameraBarcodeScanner...`, `AddSuggestedFolders...`, `SpeechToTextOverlay64...`, die Eclipse-Einträge, `NoControlPanel`, `DisableTaskMgr`, die drei Dienstabschaltungen. Bleiben: Store-Prozesse, WSL (bis Prüfschritt), WebView2-Hosts, Ablenkungs-Apps nach E11. Neu: `msedge.exe`, `mshta.exe`, `hh.exe`, Safari, Cold-Turkey-Ordnersperren unter macOS.
## 9. Ausbruchsmatrix

Spalten: Weg; Gegenmaßnahme Windows mit Verweis auf den Abschnitt in Dokument 2 (D2); Gegenmaßnahme macOS mit Verweis auf Dokument 3 (D3); Beleg; Status und Restrisiko. Abschnittsnummern folgen <aufbau_anleitung>: §4 Firefox-Richtlinien, §5 Profil, §6 Cold Turkey, §7 Betriebssystem, §8 Ausgleichsmaßnahmen, §9 Editor, §10 Ausbruchsprüfung. „Prüfschritt“ heißt: Wirkung ist nicht belegt, die Anleitung enthält einen Test mit erwartetem Ergebnis.

### 9.1 Firefox intern

| Nr. | Weg | Windows (D2) | macOS (D3) | Beleg | Status |
|---|---|---|---|---|---|
| F1 | Profilverwaltung, Profil-Flyout, `about:profiles`, `about:profilemanager` | `BlockAboutProfiles`; `Preferences` `browser.profiles.enabled` locked false (§4) | gleich (§4) | Q10, Q13, Q84, Q85 | geschlossen |
| F2 | Startparameter `-P`, `-ProfileManager`, `--profile`, `-CreateProfile` | keine Richtlinie; harmlos, weil Richtlinien und erzwungene Erweiterungen je Installation gelten (§4, §5) | gleich | Q85, Q86 | offen, ohne Wirkung: neues Profil hat dieselben Sperren. Restrisiko: Verlust von Einstellungen und ActivityWatch-Daten des Hauptprofils, keine Ausbruchswirkung |
| F3 | `about:config` | `BlockAboutConfig` (§4) | gleich | Q3, Q85 | geschlossen |
| F4 | `about:support`, Profilordner öffnen, Firefox bereinigen | `BlockAboutSupport`, `DisableProfileRefresh` (§4) | gleich | Q4 | geschlossen; der Profilordner ist ohnehin über den Explorer/Finder erreichbar, enthält aber nichts Sicherheitsrelevantes |
| F5 | `about:addons` | `BlockAboutAddons`; Erweiterungen `force_installed`; `"*": blocked` (§4) | gleich | Q9 | geschlossen |
| F6 | Fehlerbehebungsmodus (`-safe-mode`, Umschalttaste) | `DisableSafeMode` in der Registry (sperrt Menü, Kommandozeile und Umschalttaste nur über Gruppenrichtlinie/Registry) (§4); `WebsiteFilter` gilt weiter; Cold Turkey schließt Browser ohne Erweiterung (§6) | kein Startcheck auf macOS; `WebsiteFilter` und Cold-Turkey-Grace-Period halten (§4, §6) | Q6, Q88, Q16, Q8 | Windows geschlossen; macOS offen mit Ausgleich: Erweiterungen fehlen bis zu 60 s, Freigabeliste hält über `WebsiteFilter`. Restrisiko: bis zu 60 s ohne uBlock, aber nur freigegebene Seiten |
| F7 | Privates Fenster | `PrivateBrowsingModeAvailability` 1 (§4); Cold-Turkey-Erweiterung mit `private_browsing: true` (§4) | gleich | Q7, Q9, Q35 | geschlossen |
| F8 | Entwicklerwerkzeuge, Tastenkürzel, `about:debugging`, Remote-Agent | `DisableDeveloperTools` (§4); Tastenkürzel werden im Code nicht registriert | gleich | Q3, Q137 | geschlossen |
| F9 | `view-source:` | keine Richtlinie; `WebsiteFilter` bildet `view-source:`-URLs auf die Original-URL ab, gefilterte Seiten bleiben gefiltert | gleich | Q138 | offen, ohne Wirkung: zeigt nur Quelltext freigegebener Seiten |
| F10 | Reader-Ansicht | keine Richtlinie, `reader.`-Präfix nicht erlaubt; `WebsiteFilter` filtert `about:reader?url=` auf die Original-URL | gleich | Q138, Q5 | offen, ohne Wirkung |
| F11 | `file://` | `WebsiteFilter` kann `file:` sperren; Entscheidung E15: zugelassen; Cold-Turkey-Standardausnahme `file://*` | gleich | Q8, Q16 | offen nach Entscheidung: lokale Dateien sind vorhandener Inhalt |
| F12 | `data:` und `blob:` | `blob:` nicht per `WebsiteFilter` filterbar (Schema nicht zugelassen); `data:` nicht geprüft | gleich | Q139 | offen, ohne Wirkung: zeigt nur Inhalt, den der Benutzer selbst eingibt oder eine freigegebene Seite erzeugt; kein Netzzugang |
| F13 | PDF-Betrachter | `PDFjs` bleibt an (Dokumentation lesen); PDFs von freigegebenen Seiten sind freigegebener Inhalt | gleich | Q4 | offen, ohne Wirkung |
| F14 | Downloads öffnen, heruntergeladene Programme | Downloads nur von freigegebenen Seiten; ausgeführt wird nur, was die Anwendungssteuerung erlaubt (§8.3); `DownloadDirectory` gesperrt auf `${home}\Downloads` | Gatekeeper-Profil „App Store“ für quarantänierte Downloads (§7); Ordnersperre `~/Downloads` in Cold Turkey (§6) | Q45, Q21, Q103 | Windows geschlossen; macOS Prüfschritt (Ordnersperre), sonst Restrisiko B5 |
| F15 | Lesezeichen-Import | `DisableProfileImport` (§4) | gleich | Q4 | geschlossen (kein Ausbruch, schließt nur den Import fremder Profile) |
| F16 | Erweiterungen aus Dateien, AMO, `about:debugging` temporär | `ExtensionSettings` `"*": blocked` entfernt und sperrt alles ohne eigenen Eintrag; `DisableDeveloperTools` sperrt `about:debugging`; `InstallAddonsPermission` Default false (§4) | gleich | Q9, Q137 | geschlossen |
| F17 | Proxy-Einstellungen, lokaler Proxy | `Proxy` `Mode: none`, `Locked: true` (§4); lokaler Proxy hat ohnehin kein Netz (§8.1) | gleich | Q11 | geschlossen |
| F18 | Firefox-Konto, Sync | `DisableFirefoxAccounts`, zusätzlich `DisableAccounts` (ESR ab 128) (§4) | gleich | Q4 | geschlossen |
| F19 | `userChrome.css`, `user.js` löschen oder ändern | nicht genutzt (§5) | gleich | Q5 | ohne Wirkung |
| F20 | Firefox-Update | Mozilla Maintenance Service aktualisiert ESR innerhalb der Linie; Richtlinien bleiben (§7, §11) | Update nur durch Admin im Wartungsfenster (§11) | Q58, Q59, Q96 | geschlossen; Restrisiko: Richtlinienänderung bei Wechsel der ESR-Linie (Prüfliste §11) |
| F21 | Deinstallation, Neuinstallation, zweite Installation (portable, Developer Edition, Nightly, Tor) | Deinstallation verlangt Adminrechte; Installation in Benutzerordner möglich, Start aber durch Anwendungssteuerung gesperrt (§8.3); Developer Edition/Nightly lesen denselben Registry-Schlüssel | Installation in `~/Applications` möglich; Gatekeeper-Profil blockiert quarantänierte Downloads; Cold-Turkey-Ordnersperre `~/Applications` (Prüfschritt); Zweitbrowser ohne Erweiterung wird nach Grace Period geschlossen, mit Erweiterung gilt die Cold-Turkey-Freigabeliste; Richtlinien gelten dort nicht (eigene Bundle-ID) | Q45, Q101, Q100, Q16 | Windows geschlossen; macOS Prüfschritt. Restrisiko macOS: Zweitbrowser mit Cold-Turkey-Erweiterung umgeht `WebsiteFilter` und ActivityWatch, nicht die Cold-Turkey-Freigabeliste |
| F22 | `about:preferences`, `about:downloads`, `about:policies` | keine Richtlinie sperrt sie; alles Sicherheitsrelevante in `about:preferences` ist gesperrt (Proxy, Startseite, Downloads, Erweiterungen) | gleich | Q85 | offen, ohne Wirkung |
| F23 | Benutzer setzt eigene Richtlinien (`HKCU`, `~/Library/Preferences/org.mozilla.firefox.plist`) | Maschinenrichtlinien haben Vorrang vor Benutzerrichtlinien; Benutzer kann nur zusätzliche setzen | gleich; `DisableMacOSUserPolicies` nicht in ESR | Q86, Q12 | offen, ohne Wirkung: Zusatzrichtlinien ohne Ausbruchswirkung, da alle sicherheitsrelevanten vom Admin gesetzt sind |

### 9.2 Betriebssystem

| Nr. | Weg | Windows (D2) | macOS (D3) | Beleg | Status |
|---|---|---|---|---|---|
| B1 | Adminrechte des Nutzers | Variante A: Standardkonto, eingebautes Adminkonto deaktiviert (§7) | Variante A: Standardkonto (§7) | Q54, Q55, Q89 | geschlossen unter A; unter B offen für jede Maßnahme (Abschnitt 10) |
| B2 | Vorinstallierte andere Browser | Edge entfernt (EWR) und gesperrt: Cold Turkey `msedge.exe`, AppLocker-Verweigerung (§6, §7) | Safari: Cold Turkey, Firewall, Bildschirmzeit (§6, §7) | Q38, Q16, Q123 | geschlossen |
| B3 | Browser ohne Adminrechte in Benutzerordner installieren oder portabel ausführen | App Control: Pfadregeln nur für admin-beschreibbare Pfade, Benutzerordner nie erlaubt (§8.3) | wie F21 | Q45 | Windows geschlossen; macOS Prüfschritt |
| B4 | Eingebettete Browser: WebView2, WebKit-Ansichten, Electron-Apps, Office, Hilfefenster, `mshta.exe`, `hh.exe` | WebView2 ohne Netz (Firewall Standard Block, §8.1); sichtbare WebView2-Hosts in Cold Turkey (§6); `mshta`/MSXML durch App-Control-Skriptdurchsetzung gesperrt; `hh.exe` Firewall und Cold Turkey | jede App außer Firefox ohne Netz (Little Snitch, §8.1); Hilfe-Fenster (WebKit) laden damit nur lokale Inhalte | Q39, Q40, Q44 | geschlossen; Restrisiko: Apps zeigen lokalen HTML-Inhalt an |
| B5 | Programme über Datenträger, AirDrop, E-Mail | Anwendungssteuerung (§8.3) | Gatekeeper nur bei Quarantäne-Attribut; Datenträger nicht belegt | Q45, Q104 | Windows geschlossen; macOS Restrisiko: Programm ohne Quarantäne von USB-Stick läuft, außer Santa ist aktiv (E9) |
| B6 | Protokoll-Handler (`microsoft-edge:`, Standardbrowser umstellen) | Edge entfernt und gesperrt; `DontCheckDefaultBrowser`; Standardbrowser kann der Benutzer nur auf installierte, erlaubte Browser setzen | Standardbrowser kann der Benutzer setzen, aber nur auf Firefox oder gesperrtes Safari | Q38, Q124 | geschlossen |
| B7 | Systemzeit | Recht „Ändern der Systemzeit“ nur Administratoren (§7); Cold Turkey „Block time changes“ (§6) | Cold Turkey „Block time changes“; ob Standardbenutzer die Zeit ändern dürfen, nicht belegt (Prüfschritt) | Q78, Q16 | Windows geschlossen; macOS Prüfschritt. Hinweis: Zeitplan-Sperren werden in Dokument 2 und 3 nicht genutzt (Sperrtyp Password statt Zeitfenster), damit die Zeit keine Rolle spielt |
| B8 | Benutzerkontenwechsel, Gastkonto, neues Konto | Standardbenutzer kann keine Konten anlegen; Cold-Turkey-Blöcke für alle Benutzer (Prüfschritt „Edit affected users“) (§6, §7) | gleich; Gastkonto aus; Blöcke gelten auch für neue Benutzer (4.2) | Q54, Q16, Q30 | geschlossen |
| B9 | Dienste, Aufgabenplanung, LaunchAgents | Standardbenutzer: nur eigene Aufgaben im eigenen Kontext (§7); gestartet wird nur Erlaubtes | eigene LaunchAgents in `~/Library/LaunchAgents` möglich; starten nur, was erlaubt ist und ohne Netz | Q53 | geschlossen |
| B10 | Paketmanager (winget, Store, Homebrew, pip) | winget-Benutzerinstallationen laufen nicht (App Control); Store: AppLocker-Packaged-App-Regel und Cold Turkey (§6, §8.3) | Homebrew ohne Netz; App Store: Prüfschritt Restriktion plus Cold Turkey (§7) | Q22, Q130 | Windows geschlossen; macOS Prüfschritt |
| B11 | Terminal, Shells, Skripte | freigegeben; Downloads scheitern an der Firewall (Standard Block), Programme außerhalb der Systemordner an App Control; `wscript`/`cscript` per Skriptdurchsetzung gesperrt (§8) | Terminal freigegeben; Netz nur für Firefox und Systemdienste (§8.1) | Q44, Q67 | geschlossen für Ausbruch; offen für Arbeit (gewollt) |
| B12 | Registry (`HKLM`) bzw. `defaults` in `/Library` | nur Admin | nur Admin (`sudo`) | Q63, Q89 | geschlossen unter A |
| B13 | Abgesicherter Modus / sicherer Modus | Cold-Turkey-Dienst im abgesicherten Modus nicht belegt; Deaktivieren braucht Adminrechte; `WebsiteFilter` und Edge-Verweigerung in der Registry gelten weiter; Prüfschritt §10 | LaunchDaemons im sicheren Modus nicht belegt; Firefox-Richtlinien gelten; Safari ohne Cold Turkey und Firewall möglicherweise offen; Intel: Firmware-Passwort sperrt sicheren Modus; Prüfschritt §10 | Q129, Q17 | Windows Prüfschritt mit geringem Restrisiko (Firefox bleibt beschränkt); macOS Apple Silicon Restrisiko, bis der Prüfschritt zeigt, dass Cold Turkey im sicheren Modus läuft oder Safari anders gesperrt ist |
| B14 | Wiederherstellungsumgebung | BitLocker mit TPM und PIN, Wiederherstellungsschlüssel bei der Vertrauensperson; Standardkonto kann die Verschlüsselung nicht aufheben (§7) | Intel: Firmware-Passwort; Apple Silicon: kein Firmware-Passwort, Recovery-Sperre nur per MDM; ob ein Standardbenutzer in der Wiederherstellung ein Terminal auf das entsperrte Volume bekommt, ist nicht belegt (Prüfschritt §10) | Q57, Q127, Q128, Q126 | Windows geschlossen; macOS Intel geschlossen; macOS Apple Silicon Restrisiko |
| B15 | Bildschirmzeit (macOS) umgehen | entfällt | Code möglicherweise mit Kontokennwort entfernbar (Prüfschritt); deshalb nicht tragend | Q120, Q121 | macOS: Schicht ohne Verlass, Hauptschutz liegt bei Cold Turkey und Firewall |
| B16 | WSL, Hyper-V, Windows Sandbox, virtuelle Maschinen | Aktivierung optionaler Features braucht Adminrechte; `wsl.exe`/`wslInstaller.exe` in Cold Turkey gesperrt, bis Prüfschritt bestätigt | VM-Software nicht installiert; Installation scheitert an Gatekeeper-Profil und Firewall | Q54 | Windows Prüfschritt mit geringem Restrisiko; macOS geschlossen |

### 9.3 Interpreter und Language Server

| Nr. | Weg | Windows (D2) | macOS (D3) | Beleg | Status |
|---|---|---|---|---|---|
| I1 | Python- oder Java-Code mit Netzzugriff | Firewall Standard Block: `python.exe`, `pythonw.exe`, `java.exe`, `javaw.exe` ohne Allow-Regel (§8.1) | Little Snitch verweigert `python3`, `java` (§8.1) | Q67, Q115 | geschlossen; Prüfschritt: `urllib.request.urlopen` schlägt fehl |
| I2 | Lokaler Proxy oder Webdienst, den Firefox über `localhost` aufruft | lokaler Dienst hat kein Netz; `localhost` bleibt erreichbar (Prüfschritt Loopback) | gleich | 5.4 | geschlossen; Loopback Prüfschritt |
| I3 | `pip install`, Maven, Gradle, andere Paketquellen | kein Netz; Pakete nur im Wartungsfenster oder aus dem lokalen Wheel-Ordner (§8.2) | gleich | E8 | geschlossen |
| I4 | Virtuelle Umgebungen | venv-Starter ist ein Umleitungsprogramm, das den Basisinterpreter startet; App-Control-Hash- oder Herausgeber-Regel für `venvlauncher.exe`/`venvwlauncher.exe` (§8.4); Netz: Basisinterpreter ohne Allow-Regel | Symlink auf den Framework-Interpreter; Code-Identität bleibt (§8.4, Prüfschritt) | Q25, Q64, Q133, Q115 | Prüfschritt beide; sonst Restrisiko: venv unbenutzbar, nicht Ausbruch |
| I5 | Programme schreiben und starten; umbenannte Kopien gesperrter Programme | App Control: nicht erlaubt außerhalb der Systemordner; Kopie von `msedge.exe` in Benutzerordner läuft nicht; Kopie innerhalb von Program Files unmöglich (keine Schreibrechte) (§8.3) | Gatekeeper prüft Kopien ohne Quarantäne nicht; Cold Turkey Mac sperrt Apps auch nach Umbenennen (4.5); Safari-Kopie in `~/Applications`: Cold Turkey folgt der App (Prüfschritt), Firewall: Code-Identität gleich, also ohne Netz | Q45, Q16, Q115 | Windows geschlossen; macOS Prüfschritt, Restrisiko wie B5 |
| I6 | Start anderer Programme als Kindprozess | nur erlaubte Programme starten (App Control gilt für Kindprozesse) | Kindprozesse erben nichts; jeder Prozess hat eigene Code-Identität in der Firewall | Q45 | geschlossen |
| I7 | Pakete mit eingebetteter Web-Engine (pywebview, PyQt WebEngine, JavaFX WebView, Electron in Python) | nicht installiert (kein Netz für pip); falls im Wartungsfenster installiert: Engine läuft im Interpreterprozess oder als Kindprozess ohne Netz; WebView2-Hosts in Cold Turkey | gleich mit WebKit | Q40 | geschlossen; Restrisiko: zeigt lokale Inhalte |
| I8 | Python schreibt ein Programm (Mach-O, PE) | kann es nicht starten (App Control) | Gatekeeper prüft es nicht; es könnte nur Code enthalten, der ohne Netz nichts Neues holt | Q45, Q104 | Windows geschlossen; macOS Restrisiko benannt: eigener Browser ohne Netz ist kein Ausbruch |
| I9 | Änderungen an Profil- und Konfigurationsdateien, Autostart-Einträge | Firefox-Profil: nichts Sicherheitsrelevantes; Kate-Konfiguration: Abschnitt 7; Autostart: startet nur Erlaubtes | gleich | Grundsatz | geschlossen |
| I10 | App-Ausführungsaliase für `python.exe` (Store) | Aliase in den Einstellungen abschalten; Store für den Benutzer gesperrt (§9.3) | entfällt | Q74, Q63 | geschlossen |
| I11 | Node.js aus einem Language Server | `python-lsp-server` und JDT LS brauchen kein Node (Abschnitt 7) | gleich | Q140 | geschlossen |

### 9.4 Cold Turkey selbst

| Nr. | Weg | Windows (D2) | macOS (D3) | Beleg | Status |
|---|---|---|---|---|---|
| T1 | Block ohne Sperre ändern | Sperrtyp „Password“ je Block, Settings-Passwort (§6) | gleich | Q16 | geschlossen |
| T2 | Deinstallation, Installer, Update | gesperrter Block blockiert Uninstaller; „Block the installer“; Variante A: Deinstallation braucht Adminrechte | gleich | Q16, Q17 | geschlossen unter A |
| T3 | Dienst beenden, Task-Manager | Standardbenutzer kann Systemdienste nicht beenden; „Block task manager“ optional | „Block task manager“ (Aktivitätsanzeige); Prozesse des Admins nicht beendbar | Q16 | geschlossen unter A |
| T4 | Erweiterung deaktivieren oder entfernen | `about:addons` gesperrt; `force_installed`; Grace Period 60 s, dann Browser zu; „Block browsers if extension crashes“ | gleich | Q9, Q16 | geschlossen |
| T5 | Browser ohne Erweiterung | „Block unsupported browsers“; Grace Period; App Control | „Block unsupported browsers“; Grace Period; Gatekeeper-Profil, Ordnersperren | Q16 | Windows geschlossen; macOS Prüfschritt |
| T6 | Ausnahmen und was sie öffnen | nur `file://*` (Standard), `localhost`, freigegebene Seiten; keine `about:`- oder Erweiterungs-Schemata | gleich | Q16, Q33 | geschlossen; Prüfschritt `about:blank` |
| T7 | Prozessnamen umbenennen | Windows: gesperrte exe kann während des Blocks nicht umbenannt werden; Kopien laufen nicht (App Control) | Mac: Block folgt der App | Q16 | geschlossen |
| T8 | Zeitplanlücken | keine Zeitpläne, fortlaufende Blöcke mit Password | gleich | E10 | geschlossen |
| T9 | Neustart in den abgesicherten Modus | siehe B13 | siehe B13 | | Prüfschritt |
| T10 | Zweites Benutzerkonto ohne Block | „Edit affected users“ auf alle Benutzer prüfen; Standardbenutzer kann keine Konten anlegen | Blöcke wirken auf neue Benutzer (4.2) | Q16, Q30 | geschlossen |
| T11 | Mac-Version gegenüber Windows | „Add window title“ nur Windows; sonst gleich (4.5) | | Q16 | kein Weg, Hinweis |
### 9.5 Kate

| Nr. | Weg | Windows (D2) | macOS (D3) | Beleg | Status |
|---|---|---|---|---|---|
| K1 | External Tools, Build & Run, Text Filter starten beliebige Befehle | nur Programme aus Systemordnern laufen (§8.3); Netz nur für Firefox und Dienste (§8.1) | Netz nur nach Little-Snitch-Regel; gestartete Programme ohne Netz | Q45, Q116 | geschlossen, Plugins bleiben frei |
| K2 | Terminal-Plugin | wie K1 | wie K1 | Q169 | geschlossen; Prüfschritt Lauffähigkeit im Store-Paket |
| K3 | „Datei“ > „Öffnen“ mit `http://`, `ftp://` (KIO) | Kate und KIO-Worker ohne Netz (§8.1, Prüfschritt Prozessname) | gleich | Q171 | geschlossen nach Prüfschritt |
| K4 | Vorschau-Plugin | `markdownpart` ohne Web-Engine | gleich | Q172, Q173 | kein Weg |
| K5 | Abgeschaltete Plugins über Einstellungen oder `katerc` einschalten | kein Plugin öffnet einen Weg (K1 bis K4) | gleich | 7.5 | kein Weg; Kiosk-Einträge nur als Option |
| K6 | Sitzungen | Benutzerdaten | gleich | | kein Weg |
| K7 | Portable Kate in Benutzerordnern | läuft nicht (App Control) | läuft, aber ohne Netz (Code-Identität unbekannt, also verweigert) | Q45, Q115 | Windows geschlossen; macOS geschlossen für Netz |
| K8 | LSP-Client startet fremde Server | Freigabeliste in `[lspclient]`; nur erlaubte Programme startbar | gleich | Q167 | geschlossen |
| K9 | Debugger-Plugin (DAP) | unter Windows nicht verfügbar; `debugpy` ohne Netz | `debugpy` lauscht auf `localhost`; ohne Netz | Q156 | kein Weg |
## 10. Adminrechte: Variante A und Variante B

Du hast derzeit Adminrechte auf beiden Systemen. Die Matrix (Abschnitt 9) führt das als Ausbruchsweg B1. Hier die beiden Varianten, die du am Checkpoint entscheidest.

### Variante A: Standardkonto, Adminkennwort bei der Vertrauensperson

Du arbeitest im Standardkonto. Die Vertrauensperson hält das Adminkennwort, das Cold-Turkey-Passwort, den BitLocker-Wiederherstellungsschlüssel (Windows) bzw. das FileVault-Wiederherstellungskennwort und bei Bedarf den Bildschirmzeit-Code (macOS). Installationen, Paketinstallationen, Firefox-Updates unter macOS und Richtlinienänderungen laufen in einem vereinbarten Wartungsfenster, in dem die Vertrauensperson anwesend ist oder sich anmeldet.

Folgen: Jede systemweite Maßnahme (Richtlinien in `HKLM`, Konfigurationsprofil, Anwendungssteuerung, Firewall, Dateirechte) hält, weil nur der Admin sie ändern kann. Cold Turkey ist dann die sichtbare Schicht, nicht die tragende. Der Preis ist das Wartungsfenster: `pip install` eines neuen Pakets wartet bis dahin (8.2).

### Variante B: du behältst Adminrechte

Jede Maßnahme der Anleitungen ist dann von dir rücknehmbar. Was jeweils bleibt:

| Maßnahme | Unter Variante B rücknehmbar durch | Was bleibt |
|---|---|---|
| Firefox-Richtlinien (Registry, Profil) | Registry-Eintrag löschen, Profil entfernen | eine Hürde von Minuten; Cold Turkey meldet eine fehlende Erweiterung erst, wenn ein Block läuft |
| App Control / AppLocker | Richtlinie entfernen (`CiTool --remove-policy`, `secpol.msc`) | nichts |
| Firewall-Regeln | Regel löschen | nichts |
| Cold Turkey Block mit Passwort | Deinstallation während eines gesperrten Blocks ist vom Hersteller blockiert; Drittanleitungen beschreiben das Deaktivieren des Dienstes über `msconfig` oder den abgesicherten Modus, beides mit Adminrechten [Abschnitt 4.6] | eine Hürde; hält gegen einen Moment, nicht gegen einen Abend |
| Cold Turkey „Block time changes“ | Adminkonto hat das Recht „Ändern der Systemzeit“ [Q78] | Cold Turkey blockiert nur die Einstellungen-App [Q17] |
| BitLocker/FileVault gegen Wiederherstellungsumgebung | Admin kann Verschlüsselung aufheben oder Schlüssel auslesen | nichts |
| macOS Konfigurationsprofil, Bildschirmzeit | Admin entfernt Profil, ändert Code | nichts |

Empfehlung: Variante A. Die Edge-Dokumentation ist in Wahrheit an diesem Punkt gescheitert: Sie sperrte `regedit.exe` und `cmd.exe`, weil der Nutzer Administrator war, und ließ `reg.exe`, `pwsh.exe` und den abgesicherten Modus offen. Ein Lockdown, der auch gegen dich selbst halten soll, braucht eine Person, die das Adminkennwort hat, und ein Konto, das es nicht hat. Unter Variante B verbleiben alle Maßnahmen als Restrisiko „rücknehmbar durch den Nutzer selbst“; die Anleitungen weisen das dann an jeder Stelle aus, und Cold Turkey mit Sperrtyp „Password“ wird zur einzigen Schicht mit Verzögerungswirkung.

## 11. Entscheidungen am Checkpoint

Jede Entscheidung mit Empfehlung und Folge. Antwortest du nicht zu einem Punkt, gilt die Empfehlung; die Annahme wird in Dokument 1, Abschnitt 12, nachgetragen.

| Nr. | Entscheidung | Optionen | Empfehlung | Folge der Empfehlung |
|---|---|---|---|---|
| E1 | Adminkonto | A: Standardkonto, Adminkennwort bei der Vertrauensperson. B: du behältst Adminrechte. | A | Wartungsfenster nötig; alle systemweiten Maßnahmen halten. Unter B werden sie in den Anleitungen als rücknehmbar markiert (Abschnitt 10). |
| E2 | Updates | Windows: automatische Updates an mit Aktiven Stunden; oder nur im Wartungsfenster. Firefox Windows: automatisch über den Mozilla Maintenance Service; oder `ManualAppUpdateOnly`. Firefox macOS: im Wartungsfenster (ein Standardbenutzer kann eine App in `/Applications`, die dem Admin gehört, nicht aktualisieren). Cold Turkey: „Block the installer“ an, Update im Wartungsfenster. Kate: Store-Updates (Windows) bzw. Wartungsfenster (macOS). | Windows und Firefox Windows automatisch; macOS Firefox im Wartungsfenster alle vier Wochen (ESR-Punktversionen erscheinen im Vier-Wochen-Takt, Q1). | Sicherheitsupdates bleiben; nach Funktionsupdates Prüfliste aus Dokument 2, Betrieb. Risiko: Ein Firefox-Update könnte eine Richtlinie ändern; in der ESR-Linie passiert das innerhalb der Linie nicht, Richtlinien werden ergänzt, nicht entfernt (Prüfschritt bei Wechsel der ESR-Linie). |
| E3 | Edge und Safari | Edge: entfernen (EWR-Funktion in Einstellungen > Apps, Q38), sperren (Cold Turkey und AppLocker-Verweigerung), oder nur Protokoll-Handler umleiten. Safari: lässt sich nicht entfernen; sperren über Cold Turkey „Add application“ und Bildschirmzeit. | Edge entfernen und zusätzlich sperren (falls Edge durch ein Update zurückkommt). Safari sperren über Cold Turkey und als zweite Schicht Bildschirmzeit „Nur erlaubte Websites“. Firefox ESR als Standardbrowser auf beiden Systemen. | `microsoft-edge:`-Links laufen ins Leere; WebView2 bleibt (E4). Unter macOS öffnen Systemlinks aus Apps in Firefox; Safari-Start wird von Cold Turkey beendet. |
| E4 | Edge WebView2 | Prozess sperren (bricht Widgets, Outlook, Teams, Office-Dialoge, weitere unbekannte Apps); nicht sperren, Internetzugang per Firewall nehmen und die sichtbaren WebView2-Hosts in Cold Turkey sperren. | nicht sperren, Firewall und Cold-Turkey-Hosts | Offline-Dialoge in Apps funktionieren; Webinhalt in WebView2 lädt nicht. Restrisiko: ein WebView2-Host, der lokale HTML aus einem Python-Programm anzeigt; das ist Inhalt, der schon auf dem Rechner ist, kein Internet. |
| E5 | `userChrome.css` | brauchen oder nicht | nicht | Keine Profilmaßnahmen, Zweitprofile sind harmlos (Abschnitt 1). Die Avatar-Schaltfläche verschwindet mit `browser.profiles.enabled=false`; falls ein Rest sichtbar bleibt (Prüfschritt), ist er ohne Funktion. |
| E6 | Firewall ausgehend | Blockliste je Programm oder Standard Block mit Freigabeliste (5.4) | Standard Block mit Freigabeliste | Einrichtung mit Audit-Lauf; danach ist jeder Internetzugang außer Firefox, Cold Turkey und Windows-Diensten geschlossen. |
| E7 | Internetzugang für eigene Programme | nein; ja für bestimmte Ziele (Allow-Regel je Ziel-IP/-Port für `python.exe`, so eng wie möglich) | nein | Eigene Programme, die APIs aufrufen, warten auf das Wartungsfenster oder laufen auf einem anderen Gerät. Ein Zugang „für bestimmte Ziele“ ist per Firewall nur nach IP/Port möglich, nicht nach Domain, und öffnet mit jedem Ziel einen Weg zu Inhalten dieses Ziels. |
| E8 | Paketinstallation | nur im Wartungsfenster durch den Admin; oder feste Paketquelle (lokaler Wheel-Ordner `pip install --no-index --find-links`, den der Admin befüllt) | Wartungsfenster, plus lokaler Wheel-Ordner als Komfort | `pip` ohne Netz schlägt sofort fehl, das ist gewollt. Der Admin lädt im Wartungsfenster gewünschte Pakete als Wheels in einen nur für Admins beschreibbaren Ordner; du installierst daraus offline in deine venv. Ablauf: wer wann prüft, steht in Dokument 2 und 3, Betrieb. |
| E9 | Anwendungssteuerung unter macOS | Santa im Lockdown-Modus (Allowlist); oder Gatekeeper plus Bildschirmzeit plus Cold Turkey ohne Allowlist | siehe Abschnitt 6.3 nach Prüfung; Empfehlung dort | |
| E10 | Sperrart und Passwort | Sperrtyp „Password“ je Block (Pro) plus Settings-Passwort; Erzeugung durch die Vertrauensperson, 24 oder mehr zufällige Zeichen aus einem Passwort-Generator, Verwahrung außerhalb des Rechners (Papier im Umschlag oder Passwort-Manager der Vertrauensperson) | wie beschrieben | Notfall: Die Vertrauensperson kann jeden Block entsperren; ohne sie gibt es keinen Weg, das ist Absicht. Scharfschalten erst nach bestandener Ausbruchsprüfung (Dokument 2 und 3, Abschnitt 10). |
| E11 | Ablenkungs-Apps ohne Web (Solitaire, Game Bar, Xbox, Clipchamp, Sticker, Clock-Widgets) | sperren oder freigeben | sperren | entspricht dem Zweck „Schutz vor Ablenkung“; kein Ausbruchsweg, deshalb deine Entscheidung |
| E12 | Virtuelle Umgebungen | ja mit Regel für den venv-Starter (8.4); nein (nur Basisinterpreter, Pakete systemweit im Wartungsfenster) | ja | Windows: Hash-Regel für `venvlauncher.exe`/`venvwlauncher.exe` (nach jedem Python-Update erneuern) oder Herausgeber-Regel, Prüfschritt zur Signatur; macOS: Symlink-Verhalten, Prüfschritt mit Firewall. |
| E13 | Python-Installationsweg unter Windows | klassischer Installer „Für alle Benutzer“ nach `C:\Program Files\Python 3.x` (ab 3.14 als veraltet markiert, für 3.16 nicht mehr geplant, Q63); oder Python-Installationsmanager mit `py install --target` in einen Admin-Ordner (Laufzeiten sonst nur je Benutzer, Q63) | klassischer Installer für die aktuelle 3.15-Linie; Wechsel auf den Installationsmanager beim Umstieg auf 3.16 im Wartungsfenster | Pfade und Firewall-Regeln bleiben stabil; `py`-Launcher der klassischen Installation; Store-Aliase abgeschaltet. |
| E14 | Anwendungssteuerung unter Windows | App Control for Business als Basis plus AppLocker je Benutzer; oder nur AppLocker (5.3) | App Control plus AppLocker | Audit-Phase von mindestens einer Arbeitswoche vor dem Erzwingen; Installer im Wartungsfenster aus einem Admin-Ordner starten. |
| E15 | `file://` in Firefox | zulassen (lokale Dokumentation, eigene HTML-Ausgaben) oder per `WebsiteFilter` sperren | zulassen | Lokale Dateien sind Inhalt, der schon auf dem Rechner ist; die Interpreter können ohne Netz keine neuen holen. `LocalFileLinks` bleibt unangetastet. |
| E16 | Git-Remote | nur lokal; oder ein Remote-Ziel (GitHub) per Firewall-Allow für `git.exe` | nur lokal | `git push`/`pull` zum Internet schlägt fehl; ein Remote-Ziel würde über `git`’s eigenes `curl` einen Downloadweg öffnen. |
| E17 | macOS-Hardware | Intel oder Apple Silicon je Gerät | Angabe am Checkpoint | nur für die Wiederherstellung relevant (Matrix B14): bei Intel Firmware-Passwort, bei Apple Silicon Admin-Kennwort mit Secure Token |
## 12. Annahmen und Prüfschritte, die in Dokument 2 und 3 übergehen

Annahmen gelten bis zu deiner Antwort am Checkpoint (Abschnitt 11). Prüfschritte sind Stellen, an denen eine Quelle fehlt oder widersprüchlich ist; die Anleitungen enthalten dort einen Test mit erwartetem Ergebnis statt einer Behauptung.

| Nr. | Prüfschritt | Erwartetes Ergebnis | Dokument |
|---|---|---|---|
| P1 | Erweiterungs-ID der Cold-Turkey-Firefox-Erweiterung aus `manifest.json` der XPI (`browser_specific_settings.gecko.id`) | ID für `ExtensionSettings` | D2 §4, D3 §4 |
| P2 | Neuer Tab bleibt mit `NewTabPage: false` und ohne `about:`-Ausnahmen offen | offen; sonst `about:blank` als Ausnahme | D2 §6, D3 §6 |
| P3 | Cold-Turkey-Block gilt für dein Standardkonto („Edit affected users“) | ja | D2 §6, D3 §6 |
| P4 | Loopback unter Firewall-Standardaktion Block: lokaler Python-Webserver in Firefox erreichbar | erreichbar; sonst Allow-Regeln für `127.0.0.1` und `::1` | D2 §8.1, D3 §8.1 |
| P5 | Signatur von `venvlauncher.exe` (`Get-AuthenticodeSignature`) | signiert mit Herausgeber Python Software Foundation; sonst Hash-Regeln | D2 §8.4 |
| P6 | venv auf macOS unter Little Snitch: Interpreter aus der venv hat kein Netz, `pylsp` läuft | wie erwartet | D3 §8.4 |
| P7 | Windows-Firewall-Audit: eine Woche Protokoll der verworfenen Pakete, Windows Update und Defender funktionieren | keine blockierten Systemdienste | D2 §8.1 |
| P8 | App Control eine Arbeitswoche im Audit-Modus: Ereignisprotokoll CodeIntegrity ohne Einträge für Kate, Python, Java, Git, Firefox, Cold Turkey | keine Einträge; dann erzwingen | D2 §8.3 |
| P9 | Portable Firefox aus `Downloads` startet nicht; Kate aus dem Store startet | wie erwartet | D2 §10 |
| P10 | WSL für Standardbenutzer nicht installierbar | Fehler ohne Adminrechte | D2 §7 |
| P11 | `mshta.exe` und `hh.exe` vorhanden? Start unter App Control | gesperrt bzw. nicht vorhanden | D2 §10 |
| P12 | Deutscher Menüpfad zur Seite „App-Ausführungsaliase“ in Windows 11 | Einstellungen > Apps > Erweiterte App-Einstellungen > App-Ausführungsaliase (Community-Angabe) | D2 §9.3 |
| P13 | Edge-Deinstallation im EWR über Einstellungen > Apps verfügbar | Schaltfläche „Deinstallieren“ vorhanden | D2 §7 |
| P14 | WinRE mit Standardkonto: Erweiterter Start, Eingabeaufforderung, Zugriff auf `C:` | ohne BitLocker-PIN kein Zugriff | D2 §10 |
| P15 | Cold-Turkey-Dienst im abgesicherten Modus (Windows) aktiv | aktiv; sonst Restrisiko B13 bestätigt (Firefox bleibt durch Richtlinien beschränkt) | D2 §10 |
| P16 | Store-Hintergrundupdates für Kate ohne Store-App | Updates kommen an | D2 §11 |
| P17 | Herausgeberstring des Kate-Store-Pakets für die AppLocker-Regel | aus `Get-AppxPackage` | D2 §8.3 |
| P18 | Prozessname des KIO-Workers unter Windows und macOS | `kioworker.exe` bzw. `kioworker` | D2 §9, D3 §9 |
| P19 | Terminal-Plugin im Kate-Store-Paket lauffähig | ja; sonst `wt.exe` | D2 §9 |
| P20 | JDT LS 1.63.0 unter Temurin 25 mit einer `.java`-Datei ohne Netz: Vervollständigung, Fehleranzeige, Sprung zur Definition | funktioniert | D2 §9, D3 §9 |
| P21 | `pip download` und `pip install --no-index --find-links` (pip-Doku lesen, aus dieser Umgebung nicht abrufbar) | Befehle wie beschrieben | D2 §8.2, D3 §8.2 |
| P22 | macOS: `ls -ld /Applications` zeigt Schreibrecht nur für `admin` | ja | D3 §7 |
| P23 | macOS: Standardbenutzer kann in der Wiederherstellung ein Terminal mit Zugriff auf das entsperrte Volume öffnen (Apple Silicon) | nein; falls ja, Restrisiko B14 bestätigt, Entscheidung über Intel-Gerät oder MDM-Recovery-Sperre | D3 §10 |
| P24 | macOS sicherer Modus: Cold-Turkey-Dienst und Little Snitch aktiv, Safari gesperrt | aktiv; sonst Restrisiko B13 | D3 §10 |
| P25 | Bildschirmzeit-Code: lässt er sich mit dem Anmeldekennwort des Benutzerkontos entfernen? | Ergebnis entscheidet, ob Bildschirmzeit als Schicht zählt | D3 §7 |
| P26 | Cold Turkey Mac „Add folder“ für `~/Applications`, `~/Downloads`: neu angelegte App im Ordner wird beendet | ja; sonst Santa (Anhang) | D3 §6 |
| P27 | Gatekeeper-Profil `com.apple.systempolicy.control` manuell installiert: Einstellung „Apps erlauben, die geladen wurden von“ ist auf „App Store“ fixiert | ja | D3 §7 |
| P28 | App-Store-Restriktion „Require administrator password to install or update apps“: Key-Name in der Payload `com.apple.applicationaccess` | Key gefunden und wirksam | D3 §7 |
| P29 | Little Snitch 6: stiller Modus „Verbindungen verweigern“ und Sperre der Regelbearbeitung verhindern jede Freigabe durch den Benutzer | ja | D3 §8.1 |
| P30 | Standardbenutzer kann unter macOS Datum und Uhrzeit nicht ändern | Schloss verlangt Adminkennwort | D3 §7 |
| P31 | Kate-DMG aus `release-26.08`: Gatekeeper-Meldung unter macOS 15, Freigabe über „Dennoch öffnen“ | App startet danach | D3 §9.2 |
| P32 | Santa: manuell installiertes Profil `com.northpolesec.santa` wird gelesen (nur falls Santa gewählt) | `santactl status` zeigt Lockdown | D3 Anhang |
| P33 | Firefox-ESR-Prüfsummen: `SHA256SUMS` und `KEY` im Versionsverzeichnis auf `releases.mozilla.org` | vorhanden | D2 §3, D3 §3 |
| P34 | Debugger-Plugin unter macOS mit `debugpy`: Haltepunkt in einer Python-Datei | hält an | D3 §9 |
| P35 | Maven-Projekt ohne Netz nach `mvn dependency:go-offline` im Wartungsfenster | JDT LS importiert ohne Fehler | D2 §8.2, D3 §8.2 |
| P36 | `DisableAccounts` in ESR 153 wirksam (`about:policies`) | aktiv | D2 §4, D3 §4 |

## 13. Quellen

Alle Quellen am 31.10.2026 abgerufen. Quelltextangaben beziehen sich auf die genannten Repositories und Branches; die Firefox-Quellen wurden aus dem GitHub-Spiegel `mozilla-firefox/firefox`, Branch `esr153`, gelesen, die KDE-Quellen aus den GitHub-Spiegeln der `invent.kde.org`-Repositories (Branch `master` und `release/26.08`, Stand 31.10.2026). Seiten unter `firefox-admin-docs.mozilla.org` leiten auf `mozilla.github.io/enterprise-admin-reference` um; beide Adressen sind dieselbe Dokumentation.

- Q1 https://whattrainisitnow.com/release/?version=esr
- Q2 https://www.linuxtoday.com/blog/mozilla-firefox-153-esr-is-now-available-for-download-heres-whats-new/ (Sekundärquelle)
- Q3 https://mozilla.github.io/policy-templates/ (alte Referenz, laut eigener Kopfzeile nicht mehr aktuell)
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
- Q15 https://mozilla.github.io/enterprise-admin-reference/guides/policies-configuration/
- Q16 https://getcoldturkey.com/support/user-guide/
- Q17 https://getcoldturkey.com/support/
- Q18 https://getcoldturkey.com/support/system-requirements/
- Q19 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/feature-availability
- Q20 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/requirements-to-use-applocker
- Q21 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/executable-rules-in-applocker
- Q22 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/packaged-apps-and-packaged-app-installer-rules-in-applocker
- Q23 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/understanding-the-publisher-rule-condition-in-applocker
- Q24 https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule
- Q25 https://docs.python.org/3/library/venv.html
- Q26 https://learn.microsoft.com/en-us/answers/a/2597224 (Microsoft Q&A, Community)
- Q27 https://support.apple.com/guide/security/gatekeeper-and-runtime-protection-sec5599b66df/web
- Q28 https://support.apple.com/guide/mac-help/configuration-profiles-standardize-settings-mh35561/mac
- Q29 https://mozilla.github.io/enterprise-admin-reference/guides/firefox-channels/
- Q30 https://getcoldturkey.com/news/ (Release Notes)
- Q31 https://getcoldturkey.com/features/
- Q32 https://getcoldturkey.com/pricing/
- Q33 https://getcoldturkey.com/support/how-to/block-chrome-urls/
- Q34 https://getcoldturkey.com/support/extensions/firefox/
- Q35 https://getcoldturkey.com/support/extensions/firefox/privatemode/
- Q36 https://getcoldturkey.com/support/extensions/mac/safari/full-disk-access
- Q37 https://developer.apple.com/news/?id=saqachfa
- Q38 https://blogs.windows.com/windows-insider/2023/11/16/previewing-changes-in-windows-to-comply-with-the-digital-markets-act-in-the-european-economic-area/
- Q39 https://learn.microsoft.com/en-us/deployedge/webview2-enterprise
- Q40 https://learn.microsoft.com/microsoft-edge/webview2/concepts/end-user-faq
- Q41 https://learn.microsoft.com/en-us/deployedge/microsoft-edge-webview-policies
- Q42 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol
- Q43 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview
- Q44 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/script-enforcement
- Q45 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/select-types-of-rules-to-create
- Q46 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/create-appcontrol-policy-for-lightly-managed-devices
- Q47 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/deploy-appcontrol-policies-with-script und https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/operations/citool-commands
- Q48 https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-update
- Q49 https://learn.microsoft.com/en-us/windows/deployment/update/waas-wu-settings
- Q50 https://learn.microsoft.com/en-us/defender-endpoint/manage-protection-updates-microsoft-defender-antivirus
- Q51 https://learn.microsoft.com/windows/client-management/mdm/policy-csp-admx-controlpanel
- Q52 https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-ctrlaltdel
- Q53 https://learn.microsoft.com/en-us/windows/win32/TaskSchd/security-contexts-for-running-tasks
- Q54 https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/how-it-works
- Q55 https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/local-accounts
- Q56 https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-recovery-environment--windows-re--technical-reference
- Q57 https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/countermeasures
- Q58 https://support.mozilla.org/en-US/kb/deploy-firefox-msi-installers
- Q59 https://support.mozilla.org/en-US/kb/what-mozilla-maintenance-service
- Q60 https://support.mozilla.org/en-US/kb/customizing-firefox-using-group-policy-windows
- Q61 https://learn.microsoft.com/en-us/windows/curl/
- Q62 https://gitforwindows.org/faq
- Q63 https://docs.python.org/3/using/windows.html
- Q64 https://mail.python.org/pipermail/python-checkins/2018-December/158634.html
- Q65 https://kate-editor.org/get-it/
- Q66 https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-behind-the-scenes
- Q67 https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules
- Q68 https://learn.microsoft.com/en-us/powershell/module/netsecurity/set-netfirewallprofile
- Q69 https://github.com/eclipse-jdtls/eclipse.jdt.ls (README; Stand Tag v1.63.0: https://raw.githubusercontent.com/eclipse-jdtls/eclipse.jdt.ls/v1.63.0/README.md)
- Q70 https://download.eclipse.org/jdtls/milestones/
- Q71 https://github.com/eclipse-jdtls/eclipse.jdt.ls/blob/master/CHANGELOG.md
- Q72 https://www.eclipse.org/forums/index.php/t/1066859/
- Q73 https://docs.oracle.com/en/java/javase/21/install/installation-jdk-microsoft-windows-platforms.html
- Q74 https://learn.microsoft.com/de-de/windows/apps/desktop/modernize/desktop-to-uwp-extensions
- Q75 https://getcoldturkey.com/support/extensions/mac/safari/
- Q76 https://getcoldturkey.com/support/reinstall-instructions/
- Q77 https://learn.microsoft.com/en-us/answers/a/2015184 (Microsoft Q&A, Community)
- Q78 https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/change-the-system-time
- Q79 https://learn.microsoft.com/en-my/troubleshoot/microsoft-edge/development/webview2-unexpected-install-windows-server
- Q80 https://product-details.mozilla.org/1.0/firefox_versions.json
- Q81 https://www.firefox.com/en-US/firefox/153.0esr/releasenotes/
- Q82 https://www.firefox.com/en-US/firefox/140.17.0/releasenotes/
- Q83 https://www.firefox.com/en-US/firefox/138.0/releasenotes/
- Q84 Firefox-Quelltext esr153: browser/components/profiles/SelectableProfileService.sys.mjs
- Q85 Firefox-Quelltext esr153: browser/components/enterprisepolicies/Policies.sys.mjs
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
- Q101 Firefox-Quelltext esr153: build/moz.configure/init.configure
- Q102 https://support.apple.com/de-de/102445
- Q103 https://developer.apple.com/documentation/devicemanagement/systempolicycontrol.md
- Q104 https://developer.apple.com/documentation/bundleresources/information-property-list/lsfilequarantineenabled.md
- Q105 https://developer.apple.com/forums/thread/666452 und https://developer.apple.com/forums/thread/765139 (Apple-Entwicklerforum, Antworten von Apple DTS)
- Q106 https://support.apple.com/guide/deployment/depba790e53/web
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
- Q125 https://developer.apple.com/documentation/devicemanagement/safarisettings.md und https://support.apple.com/guide/deployment/depb514ccd65/web
- Q126 https://support.apple.com/en-us/102518
- Q127 https://support.apple.com/guide/deployment/dep5810e849c/web
- Q128 https://support.apple.com/guide/security/startup-disk-security-policy-control-sec7d92dc49f/web und https://support.apple.com/guide/deployment/use-secure-and-bootstrap-tokens-dep24dbdcf9e/web
- Q129 https://support.apple.com/en-kg/HT201262 und https://support.apple.com/de-de/guide/mac-help/mh21245/14.0/mac/14.0
- Q130 https://docs.brew.sh/Installation
- Q131 https://raw.githubusercontent.com/Homebrew/install/master/install.sh
- Q132 https://docs.python.org/3/using/mac.html
- Q133 https://github.com/python/cpython/blob/main/Lib/venv/__init__.py
- Q134 https://origin.cdn.kde.org/ci-builds/utilities/kate/master/macos-arm64/
- Q135 https://kate-editor.org/de/get-it/
- Q136 https://bugs.kde.org/show_bug.cgi?id=428062 und https://mail.kde.org/pipermail/kde-mac/2024-August/010160.html
- Q137 Firefox-Quelltext esr153: devtools/startup/DevToolsStartup.sys.mjs
- Q138 Firefox-Quelltext esr153: browser/components/enterprisepolicies/helpers/WebsiteFilter.sys.mjs
- Q139 Firefox-Quelltext esr153: toolkit/components/extensions/MatchPattern.cpp und dom/chrome-webidl/MatchPattern.webidl
- Q140 https://github.com/python-lsp/python-lsp-server
- Q141 https://kde.org/announcements/gear/26.08.2/
- Q142 https://apps.kde.org/kate/
- Q143 https://kate-editor.org/post/2025/2025-06-03-kate-and-co-in-the-microsoft-store/
- Q144 https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/windows/
- Q145 https://origin.cdn.kde.org/ci-builds/utilities/kate/release-26.08/macos-arm64/
- Q146 Craft-Quelltext: bin/Packager/Nsis/NullsoftInstaller.nsi (Repository packaging/craft)
- Q147 https://nsis.sourceforge.io/Docs/MultiUser/Readme.html
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
- Q160 https://adoptium.net/installation/windows/
- Q161 https://adoptium.net/installation/macOS/
- Q162 https://adoptium.net/docs/secure-software/
- Q163 JDT-LS-Quelltext: org.eclipse.jdt.ls.product/scripts/jdtls und jdtls.py
- Q164 Kate-Quelltext: addons/lspclient/settings.json (Branch release/26.08)
- Q165 https://docs.gradle.org/current/userguide/gradle_wrapper.html
- Q166 https://docs.kde.org/stable5/en/kate/kate/plugins.html
- Q167 Kate-Quelltext: apps/lib/katepluginmanager.cpp, addons/lspclient/lspclientplugin.cpp, apps/kate/main.cpp
- Q168 https://doc.qt.io/qt-6/qstandardpaths.html
- Q169 Kate-Quelltext: addons/konsole/kateconsole.cpp; Konsole-Quelltext: src/CMakeLists.txt
- Q170 https://bugs.kde.org/show_bug.cgi?id=182078
- Q171 KIO-Quelltext: src/kioworkers/CMakeLists.txt; Craft-Blueprint: kde/applications/kate/kate.py (Repository packaging/craft-blueprints-kde)
- Q172 Kate-Quelltext: addons/preview/previewwidget.cpp
- Q173 https://github.com/KDE/markdownpart
- Q174 Kate-Quelltext: addons/CMakeLists.txt (master und release/26.08)
- Q175 https://github.com/KDE/kconfig/blob/master/docs/options.md
- Q176 KConfig-Quelltext: src/core/kconfig.cpp; https://develop.kde.org/docs/administration/kiosk/introduction/ und https://develop.kde.org/docs/administration/kiosk/keys/
- Q177 Kate-Quelltext: addons/externaltools/externaltoolsplugin.cpp, addons/project/kateprojecttreeviewcontextmenu.cpp
- Q178 Kate-Quelltext: addons/lspclient/lspconfigwidget.ui
- Q179 https://docs.kde.org/stable5/en/kate/kate/kate-application-plugin-lspclient.html
- Q180 https://git-scm.com/install/windows und https://github.com/git-for-windows/git/releases/latest
- Q181 https://github.com/git-for-windows/git/releases/tag/v2.56.0.windows.1
- Q182 https://github.com/git-for-windows/build-extra/blob/main/installer/install.iss
- Q183 https://git-scm.com/install/mac
- Q184 Kate-Quelltext: apps/lib/gitprocess.cpp
