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

