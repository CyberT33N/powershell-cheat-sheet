# powershell-cheat-sheet



# $PROFILE#
- Same like ~/.bashrc in linux
- 
```powershell
notepad $PROFILE
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

