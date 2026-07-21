# STM32-Projekte mit CubeMX, CMake, GCC und VS Code — Anleitung für Windows

Gleicher Stil wie in der Linux-Anleitung dieses Standards. Die
Projekte sind vollständig plattformneutral: Ein unter Windows angelegtes
Projekt baut unverändert unter Fedora und umgekehrt, weil der gesamte
Build in versionierten Textdateien steckt (CMakeLists, Presets,
Toolchain-Datei) und nicht in IDE-Einstellungen.

---

## Glossar: Unsere Werkzeuge und warum

| Werkzeug | Was es ist | Warum wir es benutzen |
|---|---|---|
| **STM32CubeMX** | Grafischer Hardware-Konfigurator und Code-Generator von ST | Erzeugt aus Klicks (Pins, Takt, Peripherie) das komplette Projektgerüst — fehlerfrei und in Minuten statt Stunden Handarbeit. Kompiliert selbst NIE. |
| **STM32CubeCLT** | ST-Werkzeugkasten: Compiler (arm-none-eabi-gcc), Flasher (STM32_Programmer_CLI), Debug-Server (ST-LINK_gdbserver) — unter Windows inkl. CMake und Ninja | Eine feste, versionierte Installation → in fünf Jahren exakt reproduzierbare Builds. Kostenlos. |
| **arm-none-eabi-gcc** | Der eigentliche C-Compiler für ARM-Controller (Teil von CubeCLT) | Industriestandard, kostenlos, quelloffen — keine IAR-/Keil-Lizenzen nötig. |
| **CMake + Ninja** | Build-System: CMake beschreibt WAS gebaut wird, Ninja führt es schnell aus | Der Build steckt in versionierten Textdateien statt in IDE-Einstellungen → identisches Ergebnis auf jedem Rechner, egal ob Terminal oder VS Code. |
| **VS Code** | Editor mit Erweiterungen für STM32, Debugging und CMake | Kostenlos, plattformübergreifend; ersetzt CubeIDE/IAR als tägliche Arbeitsumgebung. |
| **Git** | Versionsverwaltung: protokolliert jede Änderung am Projekt lokal | Jeder Stand ist wiederherstellbar, jede Änderung nachvollziehbar ("wer/wann/warum"). |
| **GitHub** | Online-Ablage für Git-Projekte | Backup, Austausch im Team, zentrale "Wahrheit" — was nicht auf GitHub liegt, existiert offiziell nicht. |

Das Zusammenspiel in einem Satz: **CubeMX generiert das Projekt, CMake+GCC
bauen es, VS Code ist der Arbeitsplatz, Git/GitHub sind das Gedächtnis.**
CubeMX wird nur beim Anlegen und bei Hardware-Änderungen gebraucht — die
tägliche Arbeit (Code, Build, Flash, Debug, Commit) läuft ohne.

---

## Teil 1: Rechner einrichten (einmalig pro Rechner)

### 1.1 STM32CubeCLT installieren (Compiler, Flasher, Debugger)

Von st.com → Suche "STM32CubeCLT" → Get Software → Windows-Installer
(ST-Konto oder E-Mail nötig). Installer ausführen, Standardziel
`C:\ST\STM32CubeCLT_<version>\` übernehmen. Der Installer richtet den
ST-Link-USB-Treiber ein und trägt alle Werkzeuge in den PATH ein.
**CubeCLT für Windows bringt CMake und Ninja mit** — dafür ist keine
Extra-Installation nötig.

Danach ein **NEUES** PowerShell-Fenster öffnen (PATH-Änderungen gelten
nur in neu geöffneten Fenstern!) und kontrollieren — alle drei müssen
antworten:

```powershell
arm-none-eabi-gcc --version
STM32_Programmer_CLI --version
ST-LINK_gdbserver --version
```

Die installierte Version gehört in das README jedes Projekts — das ist
die Reproduzierbarkeits-Dokumentation. Mehrere Versionen können parallel
unter C:\ST\ liegen.

### 1.2 CubeMX installieren (Hardware-Konfigurator)

Von st.com → "STM32CubeMX" → Windows-Installer ausführen.
**Mindestens Version 6.11** — ältere können kein CMake generieren.

### 1.3 Git und VS Code

```powershell
winget install Git.Git
winget install GitHub.cli
winget install Microsoft.VisualStudioCode
```

(Alternativ die Installer von git-scm.com und code.visualstudio.com;
beim Git-Installer die Vorgaben übernehmen.) Danach in einem NEUEN
PowerShell-Fenster:

```powershell
git config --global core.autocrlf input
git config --global user.name  "Vorname Nachname"
git config --global user.email "mail@zur-github-adresse.de"
gh auth login        # GitHub.com -> HTTPS -> "Login with a web browser"
```

Zu `core.autocrlf input`: Windows beendet Textzeilen mit CRLF, Linux mit
LF. Diese Einstellung sorgt dafür, dass im Repository immer LF liegt —
sonst produziert jeder Plattformwechsel riesige Schein-Änderungen in
`git diff`. Die E-Mail sollte die des GitHub-Kontos sein. `gh auth login`
erledigt die Anmeldung einmalig über den Browser — danach funktioniert
`git push` dauerhaft. (Das normale GitHub-Passwort funktioniert beim
Push grundsätzlich NICHT — GitHub verlangt Token, die `gh` verwaltet.)

VS Code-Erweiterungen (Strg+Umschalt+X): **STM32Cube for Visual Studio
Code** (STMicroelectronics), **Cortex-Debug**, **C/C++**, **CMake Tools**.
Die ST-Erweiterung findet CubeCLT unter C:\ST\ selbständig; falls sie
fragt, `C:\ST\STM32CubeCLT_<version>` angeben. Projekte mit
`.vscode/extensions.json` schlagen die Erweiterungen beim Öffnen ohnehin
automatisch vor.

---

## Teil 2: Neues Projekt anlegen (pro Projekt)

### 2.1 In CubeMX erzeugen

1. CubeMX → *File → New Project* → im MCU-Selector den Controller suchen
   (z. B. STM32G071CBTx) → *Start Project*.
2. **Pinout & Configuration**: Pins, Peripherie, Middleware (z. B.
   FreeRTOS). **Clock Configuration**: Takt.
3. **Project Manager** — die drei entscheidenden Felder:
   - **Project Name:** kurz, OHNE Leerzeichen und Umlaute
     (z. B. `Projekt123`) — wird auch der Name des Build-Ziels.
   - **Project Location:** z. B. `C:\projekte\` — auch der Pfad ohne
     Leerzeichen/Umlaute (erspart Ärger mit Build-Werkzeugen).
   - **Toolchain / IDE: `CMake`** ← DAS ist der Kern des ganzen Stils.
     Steht hier CubeIDE oder EWARM, entsteht das falsche Projektformat.
4. **GENERATE CODE** klicken.

Ergebnis: `.ioc` (die Hardware-Konfiguration als Datei),
`CMakeLists.txt`, `CMakePresets.json`, `cmake/gcc-arm-none-eabi.cmake`,
`cmake/stm32cubemx/CMakeLists.txt`, `Core/` (dein Arbeitsbereich),
`Drivers/` (ST-Bibliotheken), Startup-Datei, Linkerscript.

### 2.2 Erster Build (PowerShell oder VS Code, identisches Ergebnis)

```powershell
cd C:\projekte\Projekt123
cmake --preset Debug          # einmalig: Build-Ordner konfigurieren
cmake --build --preset Debug  # bauen
```

Oder in VS Code: `code .`, Erweiterungs-Empfehlung annehmen, wenn nach
einem "Kit"/Compiler gefragt wird **"[Unspecified]"** wählen (das Preset
regelt den Compiler selbst!), Preset "Debug" wählen, F7.

Erfolgskriterium ist die Speichertabelle am Ende
(`FLASH: xxxx B / xxx KB`). Diese Zahlen ins README notieren — sie sind
der Referenzwert für jeden späteren Build. Das fertige Programm liegt
unter `build\Debug\<Name>.elf` (+ `.hex`/`.bin`).

### 2.3 Projekt-Hygiene ergänzen (unser Standard)

Vier Dateien gehören zusätzlich in jedes Projekt. Entweder aus einem
bestehenden Projekt kopieren — oder mit genau den folgenden Inhalten neu
anlegen. Als Beispiel-Projektname dient überall `Projekt123`.

> Hinweis Windows: Die Schrägstriche `/` in den JSON-Pfaden sind korrekt —
> VS Code versteht sie auch unter Windows, keine Backslashes eintragen.

#### `.gitignore` (im Projekt-Hauptordner)

Sagt Git, welche Dateien NICHT versioniert werden. Grundregel: Alles, was
aus den Quellen jederzeit neu erzeugt werden kann (Build-Ausgaben) oder
reine Werkzeug-Caches sind, gehört nicht ins Repository — sonst bläht es
auf und jeder Build erzeugt Schein-Änderungen.

```gitignore
# Build-Ausgaben - entstehen aus den Quellen jederzeit neu
build/
*.elf
*.hex
*.bin
*.map

# Editor- und Werkzeug-Caches
.cache/
.vscode/ipch

# CubeMX-Arbeitsdateien und Sicherungskopien
.mxproject
*.bak
```

#### `.vscode/extensions.json`

Die Erweiterungs-Empfehlungen des Projekts. Öffnet jemand das Projekt in
VS Code, erscheint automatisch "Do you want to install the recommended
extensions?" — ein Klick, und der Rechner ist arbeitsfähig. Die IDs
folgen dem Muster `herausgeber.erweiterungsname`.

```json
{
    "recommendations": [
        "stmicroelectronics.stm32-vscode-extension",
        "marus25.cortex-debug",
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools"
    ]
}
```

#### `.vscode/launch.json`

Die Debug-Konfiguration — sie macht aus F5 den kompletten Ablauf
"flashen, GDB-Server starten, an main() anhalten".

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug (ST-Link)",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "stlink",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/Debug/Projekt123.elf",
            "device": "STM32G071CB",
            "svdFile": "",
            "runToEntryPoint": "main"
        }
    ]
}
```

Die Felder im Einzelnen: `type: cortex-debug` wählt die
Debug-Erweiterung, `servertype: stlink` den GDB-Server aus CubeCLT.
**Anpassen pro Projekt:** `executable` (Projektname der .elf) und
`device` (exakter Controller — bestimmt Flash-Layout und
Registeransicht). Bei FreeRTOS-Projekten zusätzlich die Zeile
`"rtos": "FreeRTOS",` einfügen — dann zeigt der Debugger die Task-Liste
mit Zuständen und Stack-Auslastung an. `${workspaceFolder}` ist eine
VS-Code-Variable für den Projektordner und bleibt wörtlich so stehen.

#### `README.md`

Die Visitenkarte des Projekts — GitHub zeigt sie als Startseite an.
Regel: Ein neuer Kollege muss allein mit dem README bauen, flashen und
den Stand einordnen können. Bewährtes Gerüst zum Ausfüllen:

````markdown
# Projekt123 — <Einzeiler: was ist das Gerät/die Firmware?>

- MCU: <z. B. STM32G071CBT6 (Cortex-M0+, 128 KB Flash, 36 KB RAM)>
- Takt: <z. B. 16 MHz HSI>
- RTOS: <keins / FreeRTOS (CMSIS-RTOS v2), Tasks: ...>
- Besonderheiten: <EEPROM-Emulation, externer Watchdog, ...>

## Voraussetzungen

STM32CubeCLT <VERSION EINTRAGEN — Pflichtangabe!>, STM32CubeMX,
VS Code mit den Erweiterungen aus .vscode/extensions.json.

## Bauen

```bash
cmake --preset Debug
cmake --build --preset Debug
```

Referenzwerte (mit obiger Toolchain):
- Debug:   Flash <...> B, RAM <...> B
- Release: Flash <...> B

## Flashen und Debuggen

```bash
STM32_Programmer_CLI -c port=SWD -w build/Debug/Projekt123.hex -v -rst
```

Debuggen: F5 in VS Code ("Debug (ST-Link)").

## Workflow

Hardware-Konfiguration ausschließlich über Projekt123.ioc in CubeMX
ändern (Generate Code; eigener Code nur in USER-CODE-Blöcken).
Logik-Änderungen direkt in Core/Src/.

## Versionen

- v1.0 (<Datum>): <was ist drin> — gebaut mit CubeCLT <Version>

## Offene Punkte

- <bekannte Baustellen, ungeklärte Fragen, geplante Validierungen>
````

Warum die CubeCLT-Version Pflicht ist: Sie macht jeden Build in Jahren
exakt reproduzierbar. Warum Referenz-Buildgrößen: Ein Build auf einem
anderen Rechner, der (bis auf wenige Bytes bei anderer GCC-Generation)
dieselben Zahlen liefert, ist der schnellste Beweis, dass die Umgebung
korrekt eingerichtet ist.

### 2.4 Git und GitHub — was wir tun und warum

Kurz die Begriffe, dann die Befehle:

**Git** führt im versteckten Unterordner `.git\` ein Protokoll über jeden
festgeschriebenen Stand des Projekts. Ein festgeschriebener Stand heißt
**Commit** — wie ein beschriftetes Foto des kompletten Projektordners zu
einem Zeitpunkt. **GitHub** ist der Server, auf den diese Commits
hochgeladen werden (**push**). `main` ist der Name der Hauptlinie
(**Branch**) der Historie. Ein **Tag** ist ein Etikett an einem
bestimmten Commit ("das hier ist v1.0").

Erst auf github.com (eingeloggt, "New repository") ein LEERES Repository
anlegen — ohne Häkchen bei "Add README", sonst kollidiert es gleich mit
unserem lokalen Stand. Dann im Projektordner:

```powershell
git init                       # macht den Ordner zu einem Git-Projekt (legt .git\ an)
git add .                      # legt alle Dateien in den "Warenkorb" für den nächsten Commit
                               #   (alles, was .gitignore nicht ausschließt)
git commit -m "Projektgeruest: CubeMX-generiert (CMake), STM32CubeCLT <version>"
                               # schreibt den Warenkorb als Stand Nr. 1 fest -
                               #   die Meldung (-m) beschreibt, WAS und WARUM
git branch -M main             # nennt die Hauptlinie "main" (GitHub-Standard)
git remote add origin https://github.com/<benutzer>/<repo>.git
                               # verknüpft das lokale Projekt mit dem GitHub-Repo
                               #   ("origin" ist nur der übliche Spitzname dafür)
git push -u origin main        # lädt alle Commits zu GitHub hoch; -u merkt sich
                               #   die Verbindung, ab jetzt reicht "git push"
```

**Warum der frisch generierte Stand der ERSTE Commit sein soll, noch vor
eigenem Code:** Dann trennt die Historie für immer sauber zwischen "das
hat der Generator erzeugt" und "das haben wir geändert" — und nach jedem
späteren "Generate Code" zeigt `git diff` exakt, was CubeMX angefasst hat.

Die drei Kontrollbefehle für den Alltag (kosten nichts, ändern nichts):

```powershell
git status          # Welche Dateien sind geändert/neu? Wo stehe ich?
git diff            # Was GENAU wurde geändert (Zeile für Zeile)?
git log --oneline   # Die Historie: ein Commit pro Zeile
```

Angewöhnen: **vor jedem Commit einmal `git status` und `git diff`** —
committen, was man gesehen und verstanden hat, nicht blind.

Und für Releases:

```powershell
git tag -a v1.0 -m "v1.0 - gebaut mit STM32CubeCLT <version>"
git push origin v1.0
```

Warum taggen: Der Tag friert den exakten Quellstand einer Auslieferung
ein. "Welcher Code läuft auf den Geräten von Kunde X?" ist damit in
Sekunden beantwortbar: `git checkout v1.0` stellt genau diesen Stand
wieder her.

---

## Teil 3: Der tägliche Arbeitszyklus

```
Code ändern -> speichern -> bauen -> flashen/testen -> git status/diff -> committen -> pushen
```

- **Eigener Code AUSSCHLIESSLICH in die
  `/* USER CODE BEGIN/END */`-Blöcke** der generierten Dateien — oder in
  eigene .c/.h-Dateien, die in der Root-`CMakeLists.txt` unter
  `target_sources(...)` eingetragen werden. Code außerhalb der Blöcke
  wird beim nächsten "Generate Code" KOMMENTARLOS GELÖSCHT.
- Bauen: `cmake --build --preset Debug` oder F7 in VS Code.
  Garantiert frischer Komplettbau: `cmake --build --preset Debug --clean-first`
- Flashen: `STM32_Programmer_CLI -c port=SWD -w build\Debug\<name>.hex -v -rst`
  (`-v` = verifizieren, `-rst` = danach neu starten)
- Debuggen: F5 in VS Code — flasht, startet den GDB-Server, hält an
  `main()`. F10 = Zeile für Zeile, F5 = weiterlaufen.
- Committen: `git add -u` (alle geänderten, bereits versionierten
  Dateien) plus `git add <datei>` für neue, dann `git commit -m "..."`,
  `git push`.

**Hardware-Änderung nötig?** `.ioc` in CubeMX → ändern → Generate Code →
**`git diff` ansehen** → bauen → eigener Commit ("CubeMX: UART3 ergänzt").
Generator-Änderungen und Handänderungen nie im selben Commit mischen.

**Serienstand:** Release-Build aus dem Terminal
(`cmake --preset Release`, dann `cmake --build --preset Release`), auf
Hardware validieren, Version taggen, README nachführen.

---

## Teil 4: Checkliste für jedes neue Projekt

- [ ] CubeMX: Toolchain = CMake, Name UND Pfad ohne Leerzeichen/Umlaute
- [ ] Erster Build läuft (Terminal), Speichertabelle im README notiert
- [ ] .gitignore, .vscode/, README.md ergänzt (inkl. CubeCLT-Version)
- [ ] `git config core.autocrlf input` gesetzt (einmalig pro Rechner)
- [ ] Generierter Stand = erster Commit, Repo auf GitHub, Push erfolgreich
- [ ] Eigener Code nur in USER-CODE-Blöcken
- [ ] Keine leeren Zählschleifen für Timing — GCC optimiert sie ersatzlos
      weg! Immer HAL_Delay oder einen Hardware-Timer verwenden.
- [ ] Timing-Konstanten als benannte #define, nicht als nackte Zahlen
- [ ] Bei Flash-EEPROM-Emulation: Seiten im Linkerscript reservieren
      (FLASH-LENGTH kürzen)
- [ ] Release-Build getestet, auf Hardware validiert, Version getaggt

---

## Teil 5: Troubleshooting — typische Fehler und ihre Lösung

**`arm-none-eabi-gcc` / `cmake` / `code` / `git` "wird nicht erkannt"**
Fast immer: Das PowerShell-Fenster war schon offen, bevor das Werkzeug
installiert wurde. PATH-Änderungen gelten nur in NEU geöffneten Fenstern
— Fenster schließen, neues öffnen. Gilt genauso für das Terminal
INNERHALB von VS Code: nach Installationen VS Code neu starten.

**`unzip`/Pfad-Fehler: Datei nicht gefunden**
Fundort prüfen: `dir $env:USERPROFILE\Downloads` bzw. im Explorer
nachsehen, wohin der Browser die Datei gelegt hat, dann den Pfad im
Befehl anpassen. In PowerShell hilft die Tab-Taste beim Vervollständigen
von Datei- und Ordnernamen.

**`ninja: no work to do` — obwohl Dateien geändert/ersetzt wurden**
Ninja entscheidet nach Zeitstempel; entpackte/kopierte Dateien tragen
oft ein altes Datum. Lösung:
`cmake --build --preset Debug --clean-first`.
Merksatz: Nach jedem Entpacken über ein Projekt einmal sauber neu bauen.

**Build nutzt den falschen Compiler (x86-Fehler, Linkerfehler zu Windows-Bibliotheken)**
In VS Code wurde ein "Kit" gewählt (z. B. ein Visual-Studio- oder
MinGW-Compiler). Beheben: Strg+Umschalt+P → "CMake: Select a Kit" →
**[Unspecified]** → "CMake: Delete Cache and Reconfigure". Kontrolle:
Im Output muss `Found assembler: C:/ST/STM32CubeCLT_.../arm-none-eabi-gcc`
stehen.

**ST-Erweiterung: "Checking toolchain step failed" / Bundle-Fehler**
Bekannter Schluckauf der Erweiterung, betrifft das Bauen NICHT — Terminal
und CMake-Tools funktionieren unabhängig davon. Fenster neu laden
("Developer: Reload Window"); zur Not Erweiterung de- und neu
installieren. Löst sich oft nach dem ersten Terminal-Build von selbst.

**`STM32_Programmer_CLI -l` findet keinen ST-Link**
Anderes USB-Kabel probieren (reine Ladekabel haben keine Datenleitungen),
anderen USB-Port, Zielplatine mit Versorgung (der ST-Link speist das
Ziel i. d. R. nicht). Im Geräte-Manager muss unter USB-Geräte ein
"ST-Link" auftauchen — fehlt er, den CubeCLT-Installer erneut ausführen
(Treiber-Option).

**Virenscanner/SmartScreen blockiert Installer oder arm-none-eabi-gcc**
ST-Installer sind signiert — bei SmartScreen "Weitere Informationen →
Trotzdem ausführen". Bei Firmen-Virenscannern die Ordner C:\ST\ und den
Projektordner als Ausnahme eintragen lassen (Build-Werkzeuge, die viele
Dateien schnell erzeugen, werden gern fälschlich angemeckert und
bremsen den Build massiv).

**`git commit` → "Please tell me who you are"**
Einmalig `git config --global user.name/user.email` setzen (Schritt 1.3).

**`git push` → Authentifizierungsfehler / Passwort wird abgelehnt**
GitHub akzeptiert keine Kontopasswörter. `gh auth login` ausführen
(Schritt 1.3), danach klappt der Push dauerhaft.

**`git status`: "nichts zu committen" — obwohl doch etwas geändert wurde**
Fast immer: falscher Ordner, oder die Änderung ist nie angekommen.
`pwd` (bzw. Pfad in der Titelzeile) prüfen, dann `git diff` — ist der
leer, wurde real nichts geändert.

**`git diff` zeigt ALLE Zeilen als geändert (Zeilenenden-Rauschen)**
`core.autocrlf input` fehlte beim Klonen/Anlegen. Einstellung setzen
(Schritt 1.3); bei einem bereits betroffenen Repo im Zweifel frisch von
GitHub klonen.

**Eigener Code ist nach "Generate Code" verschwunden**
Er stand außerhalb der USER-CODE-Blöcke. Wiederherstellen mit Git:
`git diff` zeigt den Verlust, `git checkout -- <datei>` holt den letzten
committeten Stand zurück (deshalb: VOR jedem Generate committen!).
Danach den Code in einen USER-Block umziehen.

**Linkerfehler `region 'FLASH' overflowed by N bytes`**
Das Programm ist größer als der (ggf. für EEPROM-Emulation gekürzte)
Flash. Erst mit Release-Build (-Os) prüfen; dann Code verkleinern oder
Reservierung überdenken.

**Linker-Warnungen `_close/_read/_write is not implemented`**
Harmlos: Stubs der C-Bibliothek für Dateifunktionen, die es auf dem
Controller nicht gibt. Nur relevant, falls printf/scanf genutzt werden
sollen — dann eine echte Ausgabe (z. B. UART) hinterlegen.

**`HAL_Delay()` hängt oder Zeiten stimmen nicht**
Zeitbasis prüfen: Läuft der Tick-Timer/SysTick? Bei exotischen Takten
(sehr niedrige Frequenzen) die generierte Timebase-Berechnung
kontrollieren — Standardformeln setzen ≥ 1 MHz voraus
(Datei stm32g0xx_hal_timebase_tim.c).
