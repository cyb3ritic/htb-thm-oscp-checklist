# 🚨 Troubleshooting Guide

> **Stuck? Here's how to diagnose and fix the most common problems.**

---

## Table of Contents
- [Shell Issues](#shell-issues)
- [Network & Connectivity Issues](#network--connectivity-issues)
- [Scan Issues](#scan-issues)
- [Privilege Escalation Failures](#privilege-escalation-failures)
- [When You're Completely Stuck](#when-youre-completely-stuck)
- [Common Gotchas](#common-gotchas)
- [Pro Tips](#pro-tips)

---

## Shell Issues

### Shell Dies Immediately

```bash
# Problem: nc shell dies after ~1 second
# Fix 1: Start listener with rlwrap
rlwrap nc -nvlp 4444

# Fix 2: Use a stabler payload (Python, not bash)
python3 -c 'import socket,subprocess,os;...'

# Fix 3: Check for shell timeout in the application config
# Fix 4: Use port 443 (HTTPS) — firewalls may reset others
```

### Special Characters Not Working in Shell

```
Problem: Can't use arrow keys, backspace shows ^H, Ctrl+C kills shell
Fix: Upgrade to full TTY immediately after getting shell

python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
Enter Enter
export TERM=xterm-256color
```

### Shell Lost After Ctrl+C

```
Problem: Accidentally pressed Ctrl+C and lost shell
Fix: 
  - Upgrade to full TTY (above) to prevent this
  - Re-trigger the exploit
  - Always have a backup listener on a different port
```

### Shell Hangs on Connection

```bash
# Problem: nc shows connection but no prompt
# Try: Send a newline
press Enter

# Or try a different payload type (bash → python → nc)
# Check if it's a bind shell (connect to target instead):
nc $IP 4444
```

---

## Network & Connectivity Issues

### Reverse Shell Not Connecting Back

```bash
# Check 1: Is your listener running?
nc -nvlp 4444    # Is this running before you trigger the shell?

# Check 2: Are you using the right IP?
ip a show tun0   # Make sure you're using VPN IP, not local LAN IP

# Check 3: Is the port blocked?
# Try: 80, 443, 53 instead of 4444

# Check 4: Is the firewall blocking outbound?
# Try: curl http://attacker:80/ from target shell (test if outbound HTTP works)

# Check 5: Target may not be able to reach you
# Alternative: Use a bind shell instead
```

### VPN Connectivity Issues

```bash
# Check VPN is connected
ip a | grep tun0
ping -c 3 $IP

# Reconnect VPN
sudo pkill openvpn
sudo openvpn ~/vpn/yourfile.ovpn

# Reset machine on HTB/THM if it seems corrupted
```

### Can't Reach Internal Host After Pivot

```bash
# Check your route was added
ip route | grep ligolo
ip route | grep 10.10.x

# With ligolo-ng — add route manually
sudo ip route add 10.10.10.0/24 dev ligolo

# With chisel/proxychains — check proxychains config
cat /etc/proxychains4.conf | grep socks
# Make sure it matches your chisel port (default 1080)

# With nmap through proxychains — must use -sT (TCP connect), not -sS
proxychains nmap -sT -Pn -p 80,443,22 <internal_ip>
```

---

## Scan Issues

### Nmap Returns No Results

```bash
# Problem: Target seems dead but you can ping it
# Fix: Target may be blocking SYN scans
nmap -sT $IP    # TCP connect scan (slower, bypasses some firewalls)
nmap -Pn $IP    # Skip host discovery (assume host is up)
nmap -Pn -sT $IP  # Both combined

# Or use a slower scan
nmap -T2 --scan-delay 2s $IP
```

### Getting Filtered Instead of Open/Closed

```bash
# Problem: Ports show as "filtered" — firewall is dropping packets
# Fix 1: Firewall evasion
nmap -f $IP              # Fragment packets
nmap --source-port 53 $IP  # Spoof source port

# Fix 2: TCP connect instead of SYN
nmap -sT $IP

# Fix 3: Try from a different angle
# If you have a shell on the target, scan from there:
for i in $(seq 1 65535); do (echo >/dev/tcp/127.0.0.1/$i) 2>/dev/null && echo "$i open"; done
```

### Gobuster/FFuF Returns Everything (Too Many Results)

```bash
# Problem: Every path returns 200 or there are thousands of results
# Fix: Filter by size
ffuf -w wordlist.txt -u http://$IP/FUZZ -fs <size_of_homepage>

# Find the default response size first:
curl -s http://$IP/definitelynotarealpage12345 | wc -c

# Filter 403s (if site shows 403 for everything)
gobuster dir -u http://$IP -w wordlist.txt --exclude-length 403
```

---

## Privilege Escalation Failures

### sudo -l Says Nothing Useful

```bash
# Move on to: SUID → capabilities → cron → kernel → writable files
# Run linpeas and read ALL output carefully
./linpeas.sh | tee /tmp/lp.txt 2>&1
# Look for: yellow/red highlights especially
```

### SUID Binary Listed but GTFOBins Says Not Exploitable

```bash
# Check the exact binary version — sometimes only specific versions work
/usr/bin/binary --version

# Check if nosuid is set on the filesystem
mount | grep nosuid
# If the partition has nosuid: SUID binaries won't work from there
```

### Linpeas Shows Nothing Obvious

```bash
# This is common — do manual checks:
# 1. Read /etc/crontab very carefully
# 2. Run pspy64 and wait 5 minutes (cron may run every minute)
./pspy64

# 3. Check running processes for odd commands
ps aux --forest | grep -v "\["

# 4. Look for writable directories in PATH
ls -la /usr/local/bin/ /usr/local/sbin/ /opt/

# 5. Check internal services (only accessible on localhost)
ss -tlnp

# 6. Look for credentials in config files
grep -r "password" /var/www/ /etc/ /opt/ 2>/dev/null | grep -v "#"
```

### WinPEAS Crashes or Doesn't Run

```cmd
REM Try different versions
winPEAS.bat          REM Batch version
winPEAS.exe          REM .exe version

REM AV may be blocking it — try manual checks:
whoami /priv
whoami /groups
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\\windows\\"
```

---

## When You're Completely Stuck

### Structured Checklist (Go Through This Order)

```
1. Did you find ALL ports? (full port scan -p- done?)
2. Did you enumerate every single open port?
3. Did you run directory brute-force on every web port?
4. Did you check for vhosts/subdomains?
5. Did you check robots.txt, /backup, /admin, /.git?
6. Did you read ALL JS files for endpoints/secrets?
7. Did you try ALL default credential combinations?
8. Did you Google the exact version of every service?
9. Did you look at the page source fully?
10. Did you check cookies, headers, and SSL cert info?
```

### Re-Read Everything

```bash
# Re-read your nmap output from the beginning
cat nmap/targeted.txt

# Look for services/versions you ignored the first time
# Version numbers you didn't search for
# Services on unusual ports
```

### Try a Different Angle

- If you've been attacking the web service → try the other services
- If SUID didn't work → try cron
- If web app login bypass failed → try SQLi
- If you can't get RCE → maybe you just need to read a file (LFI/file disclosure)

### Fresh Eyes Technique

```bash
# Write down EVERYTHING you know:
# - Open ports: 22, 80, 8080, 445
# - Services: Apache 2.4.49 on 80, OpenSSH 7.6 on 22
# - Directories: /backup, /admin (403), /api/v1
# - Users: admin, john (from enum)
# - Credentials: None found yet

# Now look for what you HAVEN'T tried:
# - Apache 2.4.49 → searchsploit → path traversal CVE-2021-41773!
```

---

## Common Gotchas

| Gotcha | Fix |
|---|---|
| Shell reverse connects but dies | Use rlwrap; upgrade to TTY immediately |
| SUID binary not working | Check `nosuid` on mount; check for AppArmor |
| SQLMap slow / timing out | Add `--timeout=30 --retries=3`; reduce threads |
| Gobuster false positives | Use `-fs` to filter common response sizes |
| curl vs wget encoding | Always URL-encode special chars in shell payloads |
| Windows firewall blocking | Try port 80/443 instead of 4444 |
| base64 with spaces (Windows) | Use `certutil -encode` instead |
| cron job runs but shell doesn't connect | Check if cron has access to network; use `/bin/sh` not `bash` |
| LFI can't read /etc/shadow | Need root OR the file to be world-readable |
| linpeas output too long to read | `./linpeas.sh > /tmp/lp.txt` then `grep -A 2 "95%" /tmp/lp.txt` |
| AD password spray lockout | Check policy first; spray max 1-2 attempts per lockout window |
| Proxychains nmap missing results | Use `-sT` not `-sS`; add `-Pn` |
| Evil-WinRM SSL error | Add `-S` flag for HTTPS or `-c`/`-k` for certificates |

---

## Pro Tips

1. **Always save all output to files** — you'll search it 20 minutes later
2. **Take screenshot of every finding** — you won't remember everything
3. **Start scans immediately in background** — don't wait for one scan to finish before starting another
4. **Google version numbers** — `"apache 2.4.49 exploit"` is faster than manual testing
5. **Read error messages carefully** — they often reveal internal paths, usernames, or software versions
6. **When stuck, zoom out** — re-read all enumeration from the start
7. **Version specificity matters** — search for exact version e.g. `OpenSSH_7.6p1 Ubuntu` not just `openssh`
8. **In CTFs, everything is intentional** — unusual service on odd port, odd error message, odd cookie value

---

*[← Reverse Shells](./reverse_shells.md) · [Quick Reference Hub](./README.md) · [Back to Main](../README.md)*
