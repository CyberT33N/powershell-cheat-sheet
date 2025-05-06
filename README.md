# powershell-cheat-sheet



# $PROFILE
- Same like ~/.bashrc in linux
  
```powershell
notepad $PROFILE
```












<br><br>
________
________
<br><br>


# Execution Policy



<details><summary>Click to expand..</summary>

# Ausführen von PowerShell-Skript deaktivieren

```
Set-ExecutionPolicy Restricted -Scope CurrentUser
Set-ExecutionPolicy Restricted -Scope LocalMachine
```

# Ausführen von PowerShell-Skript aktivieren
```
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
Set-ExecutionPolicy Restricted -Scope LocalMachine
```


<br><br>

## 🧨 Der Trick: CMD → PowerShell → Bypass für eine Session → Skript ausführen

<details><summary>Click to expand..</summary>

### 🔧 Befehl (in `cmd.exe`):

```cmd
powershell -ExecutionPolicy Bypass -File "C:\git\privadent\scripts\install\Main.ps1"
```

---

## 🔍 Was passiert hier?

- `-ExecutionPolicy Bypass` → ignoriert alle Policies **nur in dieser Instanz**
- `-File ...` → führt dein Skript direkt aus
- Kein Einfluss auf globale oder persistente ExecutionPolicy – **clean und temporär**

---

## 🧠 Optionaler Bonus: In `.bat` verpacken

```bat
@echo off
powershell -ExecutionPolicy Bypass -File "%~dp0Main.ps1"
```

Dann kannst du `install.bat` doppelklicken – Skript wird ausgeführt trotz PS-Sperre.

</details>


</details>

























<br><br>
________
________
<br><br>

# Testing
- https://github.com/pester/Pester










<br><br>
________
________
<br><br>

# Building


<details><summary>Click to expand..</summary>

- **Nich testest bisher was unten steht**

Okay, verstanden. Du möchtest also sehen, wie der Build-Prozess mit `Invoke-Build` und `psake` aussehen würde, wenn du eine zentrale `core.ps1` hast, die (während der Entwicklung) die Funktionen aus anderen Dateien "importiert" (via Dot-Sourcing).

Das Grundprinzip bleibt gleich: Die Build-Tools orchestrieren das Zusammenfügen der Inhalte. Im finalen, gebündelten Skript sind die Dot-Sourcing-Aufrufe aus `core.ps1` überflüssig, da der *Inhalt* der anderen Dateien bereits im selben Skript-Scope vorhanden ist. Der Build-Prozess muss sicherstellen, dass die Dateien mit den Funktionsdefinitionen *vor* dem Code aus `core.ps1` (der diese Funktionen aufruft) in die Zieldatei geschrieben werden.

**Annahme für die Beispiele:**

*   Projektstruktur wie zuvor:
    ```
    MeinPowerShellProjekt/
    ├── src/
    │   ├── modules/
    │   │   ├── 01-Check-Network.ps1
    │   │   ├── 02-Install-App.ps1
    │   │   └── ... (weitere Funktionsdateien)
    │   ├── lib/
    │   │   └── Helpers.ps1  (evtl. Hilfsfunktionen)
    │   └── core.ps1        # Deine Hauptdatei, die Funktionen aus modules/lib aufruft
    │
    ├── build/              # Ort für Build-Skripte (oder im Root)
    │
    └── dist/               # Ausgabeordner
        └── Deploy-Skript.ps1
    ```
*   Deine `src/core.ps1` enthält die Hauptlogik und *könnte* während der Entwicklung Zeilen wie `. "$PSScriptRoot\modules\01-Check-Network.ps1"` enthalten (die aber im Build-Ergebnis irrelevant sind, da die Funktion `Test-NetworkConnectivity` dann direkt verfügbar ist).
*   Die Funktionsdateien (`modules/*.ps1`, `lib/*.ps1`) enthalten hauptsächlich `function ... { ... }` Blöcke.

---

**Beispiel 1: Mit `Invoke-Build`**

<details><summary>Click to expand..</summary>

1.  **Installation:** `Install-Module InvokeBuild -Scope CurrentUser`
2.  **Build-Skript erstellen:** Erstelle eine Datei im Root deines Projekts oder im `build`-Ordner, z.B. `MeinProjekt.build.ps1`.

```powershell
# MeinProjekt.build.ps1 (Beispiel für Invoke-Build)
#Requires -Modules InvokeBuild

# --- Konfiguration ---
$ProjectRoot = $PSScriptRoot
$SourcePath = Join-Path $ProjectRoot "src"
$DistPath = Join-Path $ProjectRoot "dist"
$OutputFileName = "Deploy-Skript.ps1"
$ZielDatei = Join-Path $DistPath $OutputFileName
$CoreFile = Join-Path $SourcePath "core.ps1" # Deine Haupt-Skriptdatei

# --- Tasks ---

# Task zum Aufräumen des Ausgabeordners
task Clean {
    Write-Host "Aufräumen des Ausgabeordners: $DistPath"
    if (Test-Path $DistPath) {
        Remove-Item -Path $DistPath -Recurse -Force
    }
    New-Item -Path $DistPath -ItemType Directory -Force | Out-Null
}

# Task zum Bündeln der Skripte
task Bundle -Depends Clean { # Stellt sicher, dass Clean vorher läuft
    Write-Host "Starte Bündelungsprozess..."

    # 1. Finde alle Funktionsdateien (alles außer core.ps1)
    #    Sortiere sie, um eine konsistente Reihenfolge sicherzustellen (z.B. nach Name)
    $FunktionsDateien = Get-ChildItem -Path $SourcePath -Include *.ps1 -Recurse | Where-Object { $_.FullName -ne $CoreFile } | Sort-Object Name

    # 2. Definiere die gesamte Reihenfolge: Erst Funktionen, dann Core-Logik
    $DateienZumZusammenfuehren = $FunktionsDateien + (Get-Item $CoreFile)

    Write-Host "Folgende Dateien werden in '$ZielDatei' zusammengeführt:"
    $DateienZumZusammenfuehren | ForEach-Object { Write-Host "- $($_.FullName)" }

    # 3. Header für die Zieldatei erstellen
    $Header = @"
# --- Automatisch generiertes Skript (Invoke-Build) ---
# Quelle: $SourcePath
# Erstellt am: $(Get-Date)
# NICHT DIREKT BEARBEITEN! Änderungen in 'src' vornehmen und neu bauen.
# ---

"@
    Set-Content -Path $ZielDatei -Value $Header -Encoding UTF8 # UTF8 empfohlen

    # 4. Inhalte zusammenfügen
    foreach ($Datei in $DateienZumZusammenfuehren) {
        Write-Host "Füge Inhalt hinzu: $($Datei.Name)"
        $Inhalt = Get-Content -Path $Datei.FullName -Raw
        # Optional: Kommentar hinzufügen, woher der Code stammt
        $InhaltMitMarker = @"

# --- Beginn Inhalt von: $($Datei.Name) ---
$Inhalt
# --- Ende Inhalt von: $($Datei.Name) ---

"@
        Add-Content -Path $ZielDatei -Value $InhaltMitMarker -Encoding UTF8
    }

    Write-Host "Bündelung erfolgreich! '$ZielDatei' erstellt."
}

# Standard-Task definieren (wird ausgeführt, wenn nur 'Invoke-Build' aufgerufen wird)
task . -Depends Bundle

# --- Ausführung ---
# Navigiere im Terminal zum Projektordner (wo die .build.ps1 liegt) und führe aus:
# Invoke-Build                  (Führt den Standard-Task '.', also Bundle, aus)
# Invoke-Build Bundle           (Führt explizit den Bundle-Task aus, inkl. Clean-Dependency)
# Invoke-Build Clean            (Führt nur den Clean-Task aus)
```

**Ausführung:**
Öffne PowerShell im Projektverzeichnis (`MeinPowerShellProjekt/`) und führe `Invoke-Build` aus.


</details>









<br><br><br><br>






**Beispiel 2: Mit `psake`**

<details><summary>Click to expand..</summary>

1.  **Installation:** `Install-Module psake -Scope CurrentUser`
2.  **Build-Skript erstellen:** Erstelle eine Datei namens `psake.ps1` (Standardname) oder einen anderen Namen (z.B. `build.ps1`) im Root deines Projekts.

```powershell
# psake.ps1 (Beispiel für psake)
# Requires -Modules psake # Informell, psake prüft das nicht streng

# --- Eigenschaften (Variablen) ---
Properties {
    $ProjectRoot = $PSScriptRoot
    $SourcePath = Join-Path $ProjectRoot "src"
    $DistPath = Join-Path $ProjectRoot "dist"
    $OutputFileName = "Deploy-Skript.ps1"
    $ZielDatei = Join-Path $DistPath $OutputFileName
    $CoreFile = Join-Path $SourcePath "core.ps1"
}

# --- Tasks ---

Task Clean {
    Assert (Test-Path $DistPath -PathType Container) "Ausgabeordner $DistPath existiert bereits, wird gelöscht." -ContinueOnError # Psake's Assert
    Write-Host "Aufräumen des Ausgabeordners: $DistPath"
    if (Test-Path $DistPath) {
        Remove-Item -Path $DistPath -Recurse -Force
    }
    New-Item -Path $DistPath -ItemType Directory -Force | Out-Null
}

Task Bundle -depends Clean { # Definiert Abhängigkeit zu Clean
    Write-Host "Starte Bündelungsprozess..."

    # 1. Finde Funktionsdateien (alles außer core.ps1), sortiert
    $FunktionsDateien = Get-ChildItem -Path $SourcePath -Include *.ps1 -Recurse | Where-Object { $_.FullName -ne $CoreFile } | Sort-Object Name

    # 2. Definiere gesamte Reihenfolge
    $DateienZumZusammenfuehren = $FunktionsDateien + (Get-Item $CoreFile)

    Write-Host "Folgende Dateien werden in '$ZielDatei' zusammengeführt:"
    $DateienZumZusammenfuehren | ForEach-Object { Write-Host "- $($_.FullName)" }

    # 3. Header für Zieldatei
    $Header = @"
# --- Automatisch generiertes Skript (psake) ---
# Quelle: $SourcePath
# Erstellt am: $(Get-Date)
# NICHT DIREKT BEARBEITEN! Änderungen in 'src' vornehmen und neu bauen.
# ---

"@
    Set-Content -Path $ZielDatei -Value $Header -Encoding UTF8

    # 4. Inhalte zusammenfügen
    foreach ($Datei in $DateienZumZusammenfuehren) {
        Write-Host "Füge Inhalt hinzu: $($Datei.Name)"
        $Inhalt = Get-Content -Path $Datei.FullName -Raw
        $InhaltMitMarker = @"

# --- Beginn Inhalt von: $($Datei.Name) ---
$Inhalt
# --- Ende Inhalt von: $($Datei.Name) ---

"@
        Add-Content -Path $ZielDatei -Value $InhaltMitMarker -Encoding UTF8
    }

    Write-Host "Bündelung erfolgreich! '$ZielDatei' erstellt."
}

# Standard-Task definieren
Task Default -depends Bundle {
    Write-Host "psake Build erfolgreich abgeschlossen."
}

# --- Ausführung ---
# Navigiere im Terminal zum Projektordner (wo die psake.ps1 liegt) und führe aus:
# Invoke-Psake                  (Führt den Task 'Default' aus)
# Invoke-Psake Bundle           (Führt explizit den Bundle-Task aus, inkl. Clean-Dependency)
# Invoke-Psake -BuildFile .\build.ps1 Bundle (Wenn die Datei nicht psake.ps1 heißt)
```

**Ausführung:**
Öffne PowerShell im Projektverzeichnis (`MeinPowerShellProjekt/`) und führe `Invoke-Psake` aus (wenn die Datei `psake.ps1` heißt) oder `Invoke-Psake -BuildFile DEIN_DATEINAME.ps1`.

---

**Vergleich und Fazit:**

*   **Ähnlichkeit:** Beide Tools verfolgen einen ähnlichen Ansatz mit Tasks und Abhängigkeiten. Die Kernlogik des Dateisammelns und -zusammenfügens ist in beiden Fällen fast identisch.
*   **Syntax:** `Invoke-Build` verwendet `task NAME { Skriptblock }` und eine spezielle Syntax für den Default-Task (`task .`). `psake` verwendet `Task NAME -depends TASK { Skriptblock }` und einen `Properties { }` Block für Variablen.
*   **Verbreitung/Community:** Beide sind etabliert. `Invoke-Build` ist etwas neuer und vielleicht "PowerShell-iger" in seiner Syntax. `psake` orientiert sich stärker an klassischen Build-Tools wie Rake/Make.
*   **Funktionen:** Beide bieten mehr als nur Task-Ausführung (z.B. Parameterübergabe an Tasks, komplexere Abhängigkeitsketten, Frameworks für Tests etc.).
*   **Wahl:** Für dein Szenario sind beide hervorragend geeignet. Es ist oft eine Frage der persönlichen Präferenz, welche Syntax oder welches Ökosystem einem besser gefällt.

Mit beiden Ansätzen erreichst du dein Ziel:
1.  **Lokal modular arbeiten:** In `src/` mit `core.ps1` und vielen kleinen Funktionsdateien.
2.  **Build-Prozess:** Ein klar definierter Schritt (z.B. `Invoke-Build` oder `Invoke-Psake`), der die Dateien zu `dist/Deploy-Skript.ps1` zusammenführt.
3.  **Clean Deployment:** Nur die eine `Deploy-Skript.ps1` wird verteilt.
 


</details>









<br><br><br><br>





## ✅ **Variante 3: PS2EXE – der PowerShell-Compiler**


<details><summary>Click to expand..</summary>
  
Du entwickelst modular in mehreren `.ps1`-Dateien. Deine `Main.ps1` ruft sie zusammen. Dann kompiliert `ps2exe` **alles** zu einer `.exe`.



---



### 📁 Beispielstruktur



```
MyPowerTool/
├── modules/
│   ├── 01-Check-Network.ps1
│   ├── 02-Install-App.ps1
│   ├── 03-Write-Env.ps1
│   ├── 04-Set-Registry.ps1
│   ├── 05-Cleanup.ps1
├── Main.ps1
```



### 🧠 Inhalt: `Main.ps1`



```powershell
# Main.ps1 – Einstiegspunkt
. "$PSScriptRoot\modules\01-Check-Network.ps1"
. "$PSScriptRoot\modules\02-Install-App.ps1"
. "$PSScriptRoot\modules\03-Write-Env.ps1"
. "$PSScriptRoot\modules\04-Set-Registry.ps1"
. "$PSScriptRoot\modules\05-Cleanup.ps1"



Write-Host "Alles erledigt ✅"
```



### ⚙️ Installation & Kompilierung



```powershell
Install-Module -Name ps2exe -Scope CurrentUser
```



Dann:



```powershell
Invoke-ps2exe .\Main.ps1 .\installer.exe
```



Oder wenn du kein Terminal willst:  
> Es gibt auch ein GUI-Tool: `ps2exe-GUI.ps1` → [GitHub-Link](https://github.com/MScholtes/PS2EXE)



---



### ✅ Vorteile



- Eine **einzige ausführbare `.exe`**
- **Keine PowerShell notwendig** auf der Zielmaschine
- Kann **silent** laufen (`-noConsole`)
- Digitale Signatur möglich
- Keine Module, keine Abhängigkeiten, keine Fragen


 

</details>









</details>



















<br><br>
________
________
<br><br>

# Debug

## VS Code
- https://devblogs.microsoft.com/scripting/debugging-powershell-script-in-visual-studio-code-part-1/

<details><summary>Click to expand..</summary>

Ja, du kannst auch PowerShell-Skripte in **VSCode** debuggen, aber der Prozess ist ein wenig anders als bei Node.js. Hier sind die Schritte, um ein PowerShell-Skript zu debuggen:

1. **PowerShell-Erweiterung installieren**:
   - Stelle sicher, dass du die **PowerShell**-Erweiterung in VSCode installiert hast. Diese ist notwendig, um die PowerShell-Skripte korrekt zu interpretieren.

2. **Launch-Konfiguration einrichten**:
   - Öffne die **Run and Debug**-Ansicht (Strg+Shift+D) und klicke auf **Add Configuration**.
   - Wähle die Option für **PowerShell** aus, um eine neue Konfiguration zu erstellen. Dadurch wird eine `launch.json`-Datei erstellt, die du anpassen kannst.

3. **Breakpoint setzen**:
   - Setze Breakpoints in deinem Skript, indem du auf den linken Rand der Zeile klickst, wo du anhalten möchtest.

4. **Debuggen starten**:
   - Starte das Debugging durch Klicken auf den grünen Play-Button oder drücke F5. Das PowerShell-Skript wird ausgeführt, und die Ausführung wird an den Breakpoints gestoppt, wo du dann variablen und den Stack untersuchen kannst.

Mit dieser Methode kannst du PowerShell-Skripte ähnlich wie Node.js-Dateien debuggen.


</details>






























<br><br>
________
________
<br><br>

# Taskbar




## Pin icons to taskbar
- https://learn.microsoft.com/en-us/windows/configuration/taskbar/pinned-apps?tabs=gpo&pivots=windows-11
- Not tested
- **This setting is under development and only applicable to Windows Insider Preview builds. T**




GGf. vorher:
```
Set-ExecutionPolicy RemoteSigned -Scope Process -Force
```


Apply-TaskbarLayout.ps1 (Powershell als Admin)
```
# Simple PowerShell script to apply Taskbar Layout

# Pfad zur XML-Datei (im selben Verzeichnis wie dieses Skript)
$scriptDirectory = Split-Path -Parent $MyInvocation.MyCommand.Path
$xmlFileName = 'TaskbarLayout.xml'
$xmlFilePath = Join-Path -Path $scriptDirectory -ChildPath $xmlFileName

Write-Host "Reading layout from: $xmlFilePath"

if (-not (Test-Path $xmlFilePath)) {
    Write-Error "ERROR: File '$xmlFileName' not found in '$scriptDirectory'."
    exit 1
}

# Lese den gesamten XML-Inhalt als einzelne Zeichenkette
try {
    $xmlLayoutContent = Get-Content -Path $xmlFilePath -Raw -Encoding UTF8 -ErrorAction Stop
    Write-Host "XML content read successfully."
} catch {
    Write-Error "ERROR reading XML file '$xmlFilePath': $($_.Exception.Message)"
    exit 1
}

# Definiere den Registry-Pfad und den Wertnamen
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Explorer"
$regValueName = "StartLayoutFile"

# Überprüfe, ob der Schlüssel existiert, und erstelle ihn bei Bedarf
if (-not (Test-Path $regPath)) {
    Write-Host "Creating registry key: $regPath"
    try {
        New-Item -Path $regPath -Force -ErrorAction Stop | Out-Null
    } catch {
        Write-Error "ERROR creating registry key '$regPath'. Run as Admin? ($($_.Exception.Message))"
        exit 1
    }
} else {
    Write-Host "Registry key already exists: $regPath"
}

# Setze den Registry-Wert mit dem XML-Inhalt
Write-Host "Setting registry value '$regValueName' in '$regPath'..."
try {
    Set-ItemProperty -Path $regPath -Name $regValueName -Value $xmlLayoutContent -Type String -Force -ErrorAction Stop
    Write-Host "Registry value set successfully."
    Write-Host "Restart Explorer or log off/on for changes to take effect."
} catch {
    Write-Error "ERROR setting registry value '$regValueName': $($_.Exception.Message)"
    Write-Error "Ensure the script is run as Administrator."
    exit 1
}

exit 0 
```





TaskbarLayout.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LayoutModificationTemplate
    xmlns="http://schemas.microsoft.com/Start/2014/LayoutModification"
    xmlns:defaultlayout="http://schemas.microsoft.com/Start/2014/FullDefaultLayout"
    xmlns:taskbar="http://schemas.microsoft.com/Start/2014/TaskbarLayout"
    Version="1">
  <CustomTaskbarLayoutCollection PinListPlacement="Replace">
    <defaultlayout:TaskbarLayout>
      <taskbar:TaskbarPinList>
        <taskbar:DesktopApp DesktopApplicationLinkPath="%APPDATA%\Microsoft\Windows\Start Menu\Programs\yourApp.lnk" />
        <taskbar:DesktopApp DesktopApplicationID="Microsoft.Windows.Explorer"/>
        <taskbar:DesktopApp DesktopApplicationID="MSEdge"/> 
      </taskbar:TaskbarPinList>
    </defaultlayout:TaskbarLayout>
  </CustomTaskbarLayoutCollection>
</LayoutModificationTemplate> 
```



Explorer Neustart:
```
Stop-Process -Name explorer -Force; Start-Process explorer
```






























<br><br>
________
________
<br><br>

# Registry


## Get Registry-Wert 

```
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Explorer" -Name "StartLayoutFile" | Select-Object -ExpandProperty StartLayoutFile
```




















<br><br>
________
________
<br><br>

# Kill

# Kill spezific process (all)



<details><summary>Click to expand..</summary>

```powershell
# example for node.js
Get-Process node | ForEach-Object { $_.Kill() }
```
  
</details>

