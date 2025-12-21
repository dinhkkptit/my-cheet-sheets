
# Windows WLAN Operations – Cheat Sheet

## 1. Adapter (NIC) Management
```powershell
Get-NetAdapter
Enable-NetAdapter -Name "Wi-Fi 3" -Confirm:$false
Disable-NetAdapter -Name "Wi-Fi 3" -Confirm:$false
```

## 2. Connection Status & Signal
```powershell
netsh wlan show interfaces
```

## 3. Connect / Disconnect
```powershell
netsh wlan connect name="dinhkk"
netsh wlan disconnect
```

## 4. Profile Management
```powershell
netsh wlan show profiles
netsh wlan show profiles name="dinhkk"
netsh wlan delete profile name="dinhkk"
```

### Bulk delete (keep one)
```powershell
$keep = "dinhkk"
(netsh wlan show profiles | Select-String "All User Profile").ForEach{
  $p = ($_ -split ":\s*",2)[1].Trim()
  if ($p -ne $keep) { netsh wlan delete profile name="$p" }
}
```

## 5. Auto-connect & Roaming Control
```powershell
netsh wlan set profileparameter name="dinhkk" connectionmode=auto
netsh wlan set profileparameter name="dinhkk" autoswitch=no
```

## 6. Export / Import Profiles
```powershell
netsh wlan export profile name="dinhkk" folder="C:\Temp" key=clear
netsh wlan add profile filename="C:\Temp\Wi-Fi-dinhkk.xml" user=all
```

## 7. Portable / Hardened Profile (XML Snippets)
### Hidden SSID
```xml
<nonBroadcast>true</nonBroadcast>
```

### Disable MAC Randomization
```xml
<MacRandomization xmlns="http://www.microsoft.com/networking/WLAN/profile/v3">
  <enableRandomization>false</enableRandomization>
</MacRandomization>
```

### Security
```xml
<authentication>WPA2PSK</authentication>
<encryption>AES</encryption>
```

## 8. Auto-Reconnect Strategy
Preferred: Use Windows auto-connect.
Fallback:
```powershell
netsh wlan connect name="dinhkk"
```

## 9. Security Checklist
- WPA2/WPA3 only
- Remove old profiles
- Disable autoswitch
- Disable MAC randomization on trusted networks
- Restrict XML permissions

## 10. Ansible Snippets
```yaml
- ansible.windows.win_shell: |
    if ((Get-NetAdapter -Name "Wi-Fi 3").Status -ne "Up") {
      Enable-NetAdapter -Name "Wi-Fi 3" -Confirm:$false
    }

- ansible.windows.win_shell: |
    netsh wlan connect name="dinhkk"
```
