---
type: post
title: install openssh on windows
date: '2019-07-16T09:32:28-0500'
author: arges
tags:
- windows
modified_time: '2019-07-16T09:32:12-0500'
---

The following are instructions on how to setup OpenSSH on various Windows
server versions. This is great if you come from a Linux background and are very
used to SSH for accessing machines.

Windows Server 2019
-------------------
First run PowerShell as Administrator.

Next install SSH and SSHD for Windows:
```
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

Confirm installation by ensuring rules are correct:
```
Get-NetFirewallRule -Name *ssh*
```

Windows Server 2016/2012R2
---------------------
Download zip from github:
```
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest https://github.com/PowerShell/Win32-OpenSSH/releases/download/v8.0.0.0p1-Beta/OpenSSH-Win64.zip -OutFile openssh.zip
```

Extact it (for 2012R2):
```
[System.Reflection.Assembly]::LoadWithPartialName("System.IO.Compression.FileSystem")
[System.IO.Compression.ZipFile]::ExtractToDirectory('openssh.zip', 'C:\Program Files\')
```

Extract it (for 2016):
```
Expand-Archive .\openssh.zip 'C:\Program Files\'
```

Install it:
```
cd 'C:\Program Files\OpenSSH-Win64\'
.\install-sshd.ps1
mkdir C:\ProgramData\ssh
.\ssh-keygen.exe -A
.\FixHostFilePermissions.ps1
# select A to all questions
Start-Service -Name sshd
Set-Service -Name "sshd" -StartupType automatic
```

Confirm it is running:
```
Get-WMIObject win32_service -Filter "name = 'sshd'"
```

Open up firewall:
```
netsh advfirewall firewall add rule name="Open Port 22" dir=in action=allow protocol=TCP localport=22
```
