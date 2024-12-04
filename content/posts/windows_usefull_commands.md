---
title: "Windows_usefull_commands"
date: 2024-09-25T16:58:15+02:00
draft: true
---

## Deactivate Windows Firewall

```bash
Set-NetFirewallProfile -Enabled False
```

## Make a user admin

```bash
net localgroup "Administrators" /add "UserAccountName"
```

## Deactivate Windows Defender

```bash
Get-MpComputerStatus | Select-Object -Property Antivirusenabled,AMServiceEnabled,AntispywareEnabled,BehaviorMonitorEnabled,IoavProtectionEnabled,NISEnabled,OnAccessProtectionEnabled,RealTimeProtectionEnabled,IsTamperProtected,AntivirusSignatureLastUpdatedCopied
```

Disable Defender

```bash
Set-MpPreference -DisableRealtimeMonitoring $true
```

Disable the cloud-delivered protection:
```bash
Set-MpPreference -MAPSReporting Disabled
```

## OpenSSH

Install OpenSSH

```bash
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
```

```bash
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Start SSH

```bash
Start-Service -Name sshd
```

Set Startup Type automatic

```bash
Set-Service sshd -StartupType Automatic
```

Make Powershell the default shell
```bash
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```
