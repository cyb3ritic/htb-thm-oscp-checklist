# 🧠 Phase 0 — Methodology & Mindset

> **Read this before your first machine.** Understanding the *why* behind the methodology is what separates someone who gets stuck from someone who roots the box.

---

## Table of Contents
- [The Pentesting Loop](#the-pentesting-loop)
- [Decision Tree — What to Do When Stuck](#decision-tree)
- [Phase Time Budget](#phase-time-budget)
- [OSCP Exam Strategy](#oscp-exam-strategy)
- [Note-Taking Template](#note-taking-template)

---

## The Pentesting Loop

Every machine, no matter how complex, follows the same core loop:

```
ENUMERATE → ANALYZE → RESEARCH → EXPLOIT → REPEAT
     ↑                                         │
     └─────────── (you'll come back here) ─────┘
```

**Enumerate** everything visible. **Analyze** what's unusual or version-specific. **Research** that thing (Google, SearchSploit, HackTricks). **Exploit** it. Then enumerate again from your new position.

> **💡 The Golden Rule:** If you're stuck, you haven't enumerated enough. Go back and look harder.

---

## Decision Tree

Use this when you don't know what to do next.

### Got Nothing Yet (No Shell)

```
Did you run a full port scan (-p-)?  ──NO──► Run it now
        │YES
        ▼
Did you check ALL services, not just common ones?  ──NO──► Enumerate each port
        │YES
        ▼
Did you run directory brute-force on ALL web ports?  ──NO──► Run gobuster/ffuf
        │YES
        ▼
Did you check for virtual hosts / subdomains?  ──NO──► Fuzz vhosts
        │YES
        ▼
Did you try default/weak credentials on every login?  ──NO──► Try them
        │YES
        ▼
Did you Google the exact service versions you found?  ──NO──► Search for CVEs
        │YES
        ▼
Look at everything again. Read source code. Read JS. Check certificates.
```

### Got a Shell — Need Privesc

```
Did you run linpeas / winpeas?  ──NO──► Run it and read output fully
        │YES
        ▼
Checked sudo -l ?  ──NO──► Run it (Linux)
        │YES
        ▼
Checked SUID binaries + GTFOBins?  ──NO──► Run find and cross-reference
        │YES
        ▼
Checked cron jobs + writable scripts?  ──NO──► Check /etc/crontab and pspy
        │YES
        ▼
Checked capabilities (getcap)?  ──NO──► Run getcap -r / 2>/dev/null
        │YES
        ▼
Checked for internal services (ports only on localhost)?  ──NO──► ss -tlnp
        │YES
        ▼
Check file system for readable config files, backups, history files, SSH keys
```

---

## Phase Time Budget

Use this as a rough guide (for a 2-hour medium machine):

| Phase | Suggested Time | Notes |
|---|---|---|
| Setup | 5 min | VPN, dirs, tmux |
| Network Recon | 10–15 min | Run scans in background immediately |
| Service Enum | 20–30 min | Go deep on each open port |
| Web/App Enum | 20–30 min | If web ports found |
| Exploit Research | 10–15 min | CVE/version lookup |
| Exploitation | 10–20 min | Don't brute-force time here |
| Post-Exploit + PrivEsc | 20–30 min | Read linpeas output carefully |
| Flags + Notes | 5 min | Always document |

> **💡 Tip:** Start your full port scan (`nmap -p-`) in the background **immediately** and begin manual enumeration of obvious ports while it runs.

---

## OSCP Exam Strategy

### Rules Summary
| Rule | Detail |
|---|---|
| Duration | 23 hours 45 minutes active testing |
| Points needed | 70/100 to pass |
| Metasploit limit | **1 module only** (exploit, auxiliary, or post) — choose wisely |
| Banned tools | Automated exploitation (SQLMap, etc.), AI assistance |
| AD set | All 3 machines in the AD set are 40 points combined — high priority |
| Standalone machines | 3 standalone machines at 20 pts each (10 user + 10 root) |
| Reporting | Separate 24 hours to write report after testing ends |

### Recommended Exam Order
1. **Start with Active Directory (40 pts)** — It's all-or-nothing; get foothold + PE + DC ASAP
2. **Then tackle 20-pt standalone machines** — Balance difficulty vs. time
3. **Save your 1 Metasploit module** for the hardest machine or a specific exploit you know works
4. **Screenshot everything** — Every step, every command, every flag

### Exam Report Must-Haves
- Screenshot of `ipconfig`/`ifconfig` showing the IP address for every machine
- Screenshot of each flag (`type proof.txt` or `cat proof.txt` with IP visible)
- Step-by-step reproduction of every exploit used
- All command syntax you ran

---

## Note-Taking Template

Create a file per machine. Suggested format:

```markdown
# Machine: <name>
**IP:** 10.10.x.x  
**OS:** Linux / Windows  
**Difficulty:** Easy / Medium / Hard  
**Platform:** HTB / THM / OSCP  
**Date:** YYYY-MM-DD  

---

## Open Ports
| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 7.6 |

---

## Web Directories Found
- /admin → login page
- /backup → directory listing

---

## Credentials Found
| Username | Password / Hash | Service | Notes |
|---|---|---|---|
| admin | password123 | HTTP | found in /backup/db.bak |

---

## Exploit Path
1. Found LFI at /page.php?file=
2. Used LFI to read SSH private key at /home/user/.ssh/id_rsa
3. SSH login as user
4. sudo -l showed: (root) NOPASSWD: /usr/bin/vim
5. GTFOBins vim privesc → root

---

## Flags
- **user.txt:** `flag{...}`
- **root.txt:** `flag{...}`

---

## Lessons Learned
- Always check /home/<user>/.ssh/ for keys
```

---

## 🔗 References

- [HackTricks](https://book.hacktricks.xyz) — The ultimate pentest reference
- [GTFOBins](https://gtfobins.github.io) — Linux SUID/sudo/capability escape
- [LOLBAS](https://lolbas-project.github.io) — Windows living-off-the-land binaries
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — Payloads for every attack
- [Revshells.com](https://www.revshells.com) — Interactive reverse shell generator
- [CyberChef](https://gchq.github.io/CyberChef) — Encoding/decoding/analysis

---

*[← Back to Main README](../README.md)*
