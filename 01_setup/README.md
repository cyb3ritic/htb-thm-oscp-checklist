# 🛠️ Phase 1 — Pre-Engagement Setup

> **Get your environment ready before touching the target.** A messy workspace = missed findings.

---

## Table of Contents
- [VPN & Connectivity](#vpn--connectivity)
- [Workspace Setup](#workspace-setup)
- [Essential Tools Check](#essential-tools-check)
- [Wordlists](#wordlists)
- [tmux Workflow](#tmux-workflow)
- [Burp Suite Setup](#burp-suite-setup)

---

## VPN & Connectivity

- [ ] **Connect to VPN** (HTB/THM):
  ```bash
  sudo openvpn ~/vpn/<yourfile>.ovpn
  ```
- [ ] **Verify VPN interface**:
  ```bash
  ip a | grep tun0
  # Should show tun0 with an IP address
  ```
- [ ] **Test connectivity to target**:
  ```bash
  ping -c 3 $IP
  ```
- [ ] **Set target IP variable** (do this at the start of every session):
  ```bash
  export IP=<target_ip>
  echo $IP   # verify
  ```

> **⚠️ OSCP Note:** On OSCP, use `tun0` as your LHOST for all reverse shells. Run `ip a show tun0` to confirm your IP.

---

## Workspace Setup

- [ ] **Create per-machine directory**:
  ```bash
  mkdir -p ~/htb/<machinename>/{nmap,web,exploits,loot,screenshots}
  cd ~/htb/<machinename>
  ```
- [ ] **Open note-taking tool** and create a new note for the machine
  - Recommended: **Obsidian** (markdown-based, offline, searchable)
  - Alternatives: CherryTree, Notion, Joplin, plain text
- [ ] **Screenshot tool configured**:
  ```bash
  # Flameshot (Linux) — configure a hotkey in system settings
  flameshot gui
  ```
- [ ] **Multiple terminals or tmux ready** (see [tmux Workflow](#tmux-workflow))

---

## Essential Tools Check

Run these to verify your toolkit is installed and up to date:

```bash
# Scanning
nmap --version
masscan --version
rustscan --version 2>/dev/null || echo "rustscan: not installed"

# Web
gobuster version
ffuf -V
nikto -Version
whatweb --version

# Exploitation
msfconsole --version
searchsploit --version

# Post-exploitation helpers
which linpeas.sh 2>/dev/null || echo "Download linpeas from github.com/carlospolop/PEASS-ng"
which winpeas.exe 2>/dev/null || echo "Download winpeas from github.com/carlospolop/PEASS-ng"

# Password attacks
hashcat --version
john --version
hydra --version
```

### Tool Download Quick-Reference

| Tool | Install / Source |
|---|---|
| LinPEAS / WinPEAS | `https://github.com/carlospolop/PEASS-ng/releases` |
| pspy | `https://github.com/DominicBreuker/pspy/releases` |
| Chisel | `https://github.com/jpillora/chisel/releases` |
| Ligolo-ng | `https://github.com/nicocha30/ligolo-ng/releases` |
| Evil-WinRM | `gem install evil-winrm` |
| Impacket | `pip3 install impacket` or `apt install python3-impacket` |
| CrackMapExec | `apt install crackmapexec` or `pipx install crackmapexec` |
| BloodHound | `apt install bloodhound` |

---

## Wordlists

> See [quick_reference/wordlists.md](../quick_reference/wordlists.md) for the full guide.

- [ ] **Verify SecLists installed**:
  ```bash
  ls /usr/share/seclists/
  # If missing:
  sudo apt install seclists
  # Or: git clone https://github.com/danielmiessler/SecLists /usr/share/seclists
  ```
- [ ] **Most-used wordlist paths** (memorize these):
  ```bash
  /usr/share/seclists/Discovery/Web-Content/common.txt          # Quick web fuzzing
  /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  # Thorough
  /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt        # DNS/vhosts
  /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz        # Hash cracking
  /usr/share/wordlists/rockyou.txt                               # (pre-extracted on Kali)
  /usr/share/seclists/Usernames/Names/names.txt                  # Username list
  ```

---

## tmux Workflow

Using tmux lets you have multiple panes in one terminal — essential for running scans in the background while you work manually.

```bash
# Start a new session
tmux new -s htb
```

**Key bindings (default prefix: Ctrl+B)**

```text
Split pane horizontally: Ctrl+B then "
Split pane vertically:   Ctrl+B then %
Navigate between panes:  Ctrl+B then ←/→/↑/↓
Detach session:          Ctrl+B then D
```
```bash
# Reattach session
tmux attach -t htb
```


**Suggested layout:**
- Pane 1: Main work (commands, manual enum)
- Pane 2: Running scans (nmap, gobuster — background scans)
- Pane 3: Netcat listener (ready to catch shell)
- Pane 4: Notes / reference

---

## Burp Suite Setup

For web-heavy machines:

- [ ] **Launch Burp Suite** and create a new project
- [ ] **Configure browser proxy**: `127.0.0.1:8080`
  ```bash
  # Firefox → Settings → Network Settings → Manual proxy
  # Or use FoxyProxy extension
  ```
- [ ] **Install Burp CA certificate** in browser (for HTTPS interception)
- [ ] **Enable intercept** for initial exploration, then disable for passive capture

> **💡 Tip:** Use Burp's "Target > Site Map" to build a visual map of all visited endpoints. Use "Intruder" for fuzzing and "Repeater" for manually testing requests.

> **⚠️ OSCP Note:** Burp Suite Community Edition is fine for OSCP. Intruder rate-limiting in Community Edition is slow — use FFuF or Gobuster for directory brute-forcing instead.

---

## Initial Information Gathering

Once setup is done, before your first scan:

- [ ] **Record machine details**: IP, OS (if hinted), difficulty, platform tags
- [ ] **Check platform hints/tags**: THM room description, HTB machine info box
- [ ] **Set up a timer** — track how long you spend on each phase
- [ ] **If not exam**: brief search for known machine writeup intros (don't read exploits, just orient)

---

## 🔗 References

- [tmux Cheatsheet](https://tmuxcheatsheet.com)
- [SecLists on GitHub](https://github.com/danielmiessler/SecLists)
- [Kali Linux Tools](https://www.kali.org/tools/)

---

*[← Back to Main README](../README.md) · [Next: Recon & Enumeration →](../02_recon_and_enum/README.md)*
