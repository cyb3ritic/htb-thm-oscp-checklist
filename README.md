# 🎯 HTB / THM / OSCP — Master Penetration Testing Checklist

> **A complete, modular methodology for Hack The Box, TryHackMe, and OSCP-level machines.**  
> Designed for beginners and intermediate practitioners. Follow it phase by phase, or jump to any section with the links below.

**Author:** [cyb3ritic](https://github.com/cyb3ritic) · **Version:** 3.0 · **Updated:** July 2025

---

## 🗺️ Quick Navigation

| Phase | Section | Focus |
|---|---|---|
| 0️⃣ | [Methodology & Mindset](./00_methodology/README.md) | Thinking like a pentester, OSCP rules |
| 1️⃣ | [Pre-Engagement Setup](./01_setup/README.md) | VPN, tools, workspace, wordlists |
| 2️⃣ | [Recon & Enumeration](./02_recon_and_enum/README.md) | Scanning, services, web, DNS, AD |
| 3️⃣ | [Vulnerability Assessment](./03_vuln_assessment/README.md) | Finding weaknesses to exploit |
| 4️⃣ | [Exploitation](./04_exploitation/README.md) | Web attacks, shells, Metasploit |
| 5️⃣ | [Password Attacks](./05_password_attacks/README.md) | Hash cracking, spraying, Responder |
| 6️⃣ | [Post-Exploitation](./06_post_exploitation/README.md) | Shell stabilization, looting, recon |
| 7️⃣ | [Privilege Escalation](./07_privilege_escalation/README.md) | Linux, Windows, Database |
| 8️⃣ | [Active Directory](./08_active_directory/README.md) | Kerberoasting, BloodHound, DCSync |
| 9️⃣ | [Lateral Movement](./09_lateral_movement/README.md) | Tunneling, pivoting, PtH/PtT |
| 🔟 | [Persistence](./10_persistence/README.md) | Backdoors, scheduled tasks |
| 1️⃣1️⃣ | [Objective Hunting](./11_objective_hunting/README.md) | Flags, data exfil, clean-up |

---

## ⚡ Quick Reference

| Resource | Description |
|---|---|
| [🗒️ Cheatsheets](./quick_reference/cheatsheets.md) | Top commands per phase on one page |
| [🔧 Tools Index](./quick_reference/tools_index.md) | Every tool: purpose, install, usage |
| [📂 File Transfers](./quick_reference/file_transfers.md) | Get files on/off target (Linux & Windows) |
| [💻 Reverse Shells](./quick_reference/reverse_shells.md) | All shell types + listeners + upgrades |
| [📖 Wordlists Guide](./quick_reference/wordlists.md) | Which SecList to use and when |
| [🚨 Troubleshooting](./quick_reference/troubleshooting.md) | Common problems and fixes |

---

## 🏁 Attack Workflow (TL;DR)

```
┌─────────────────────────────────────────────────────────────┐
│  START → Setup → Recon → Vuln Assessment → Exploit          │
│    ↓                                                         │
│  Get Shell → Stabilize → Enumerate Locally                  │
│    ↓                                                         │
│  Privilege Escalate → Loot → Lateral Move (if needed)       │
│    ↓                                                         │
│  Grab Flags → Document → Clean Up → DONE ✅                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Pre-Engagement Checklist (Quick)

Before starting **any** machine, verify:

- [ ] VPN connected (`ip a | grep tun0`)
- [ ] Target IP set as variable: `export IP=<target_ip>`
- [ ] Output directory created: `mkdir -p ~/htb/<machinename>/{nmap,web,exploits,loot}`
- [ ] Note-taking tool open (Obsidian / CherryTree / Notion)
- [ ] `tmux` or multiple terminal tabs ready
- [ ] Burp Suite proxy configured (if web machine)

> **💡 Tip:** Set `export IP=<target>` at the start. All commands in this guide use `$IP` as the target variable.

---

## 🧠 Mindset Reminders

- **Enumerate more than you exploit.** 80% of the time, the path forward is in the details of your enumeration.
- **Document everything.** Screenshots, tool output, credentials found — all of it.
- **If stuck for 30 minutes:** step back and re-read ALL your enumeration output from the beginning.
- **Try the obvious things first:** default creds, anonymous access, version-specific CVEs.
- **Google the exact version of everything you find.**

---

## ⚠️ OSCP Exam Rules (Know Before You Go)

- Only **1 Metasploit module** (including `exploit/`, `auxiliary/`, `post/`) allowed per exam
- **No automated exploitation tools** (SQLMap, etc.) — manual techniques only  
- **No AI assistance** during the exam
- Document every step with screenshots for your exam report
- Active Directory set is **all-or-nothing** (0 or full points per set)

> See [00_methodology/README.md](./00_methodology/README.md) for the full OSCP strategy guide.

---

## 📦 Repository Structure

```
htb-thm-oscp-checklist/
├── 00_methodology/         ← Mindset, decision trees, OSCP rules
├── 01_setup/               ← Environment prep, tools, workspace
├── 02_recon_and_enum/      ← Network, web, DNS, SMB, LDAP, services
├── 03_vuln_assessment/     ← Vuln scanning, web app testing, CMS
├── 04_exploitation/        ← Web exploits, network, shells
├── 05_password_attacks/    ← Cracking, spraying, Responder
├── 06_post_exploitation/   ← Shell upgrade, local recon, looting
├── 07_privilege_escalation/ ← Linux, Windows, Database privesc
├── 08_active_directory/    ← AD attacks, BloodHound, Kerberos
├── 09_lateral_movement/    ← Pivoting, tunneling, WinRM
├── 10_persistence/         ← Backdoors, cron jobs, reg keys
├── 11_objective_hunting/   ← Flags, exfil, cleanup
└── quick_reference/        ← Cheatsheets, tools, shells, wordlists
```

---

> *Maintained by [cyb3ritic](https://github.com/cyb3ritic). PRs and issues welcome!*  
**Keywords:** penetration-testing · oscp · hackthebox · tryhackme · ctf · ethical-hacking · kali-linux · methodology · checklist  
**Environment**: Kali Linux / HTB/THM Platforms
















