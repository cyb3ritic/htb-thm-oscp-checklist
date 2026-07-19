# 🔍 Phase 2 — Recon & Enumeration

> **The most important phase.** Don't rush this. Every open port is a potential attack surface.  
> Start scans running in the background immediately, then manually work through each discovered service.

---

## Sub-Sections

| File | Contents |
|---|---|
| [01_network_scanning.md](./01_network_scanning.md) | Nmap, Masscan, RustScan — find open ports |
| [02_web_enum.md](./02_web_enum.md) | Directory fuzzing, vhosts, parameter discovery |
| [03_dns_enum.md](./03_dns_enum.md) | Zone transfers, subdomain brute-force |
| [04_smb_enum.md](./04_smb_enum.md) | SMB shares, users, vulns |
| [05_ldap_ad_enum.md](./05_ldap_ad_enum.md) | LDAP, Active Directory, BloodHound |
| [06_other_services.md](./06_other_services.md) | FTP, SSH, SNMP, NFS, Redis, MSSQL, MySQL, RDP, WinRM, and more |

---

## Quick Start Checklist

Copy-paste this at the start of every machine:

```bash
# 1. Full TCP port scan (run in background immediately)
nmap -p- --min-rate 5000 -oN nmap/alltcp.txt $IP &

# 2. Quick top-1000 scan for fast initial results
nmap -sC -sV --top-ports 1000 -oN nmap/initial.txt $IP

# 3. Once full scan finishes, targeted scan on all found ports
# nmap -sC -sV -p <ports> -oN nmap/targeted.txt $IP

# 4. UDP scan (top 20 common UDP ports)
sudo nmap -sU --top-ports 20 -oN nmap/udp.txt $IP
```

---

## Service-to-Section Map

When you find an open port, jump to the right section:

| Port(s) | Service | Section |
|---|---|---|
| 21 | FTP | [06_other_services.md](./06_other_services.md#ftp-21) |
| 22 | SSH | [06_other_services.md](./06_other_services.md#ssh-22) |
| 25, 465, 587 | SMTP | [06_other_services.md](./06_other_services.md#smtp-25) |
| 53 | DNS | [03_dns_enum.md](./03_dns_enum.md) |
| 80, 443, 8080, 8443 | HTTP/HTTPS | [02_web_enum.md](./02_web_enum.md) |
| 88 | Kerberos | [05_ldap_ad_enum.md](./05_ldap_ad_enum.md) |
| 110, 143 | POP3/IMAP | [06_other_services.md](./06_other_services.md#pop3--imap) |
| 111 | RPCBind | [06_other_services.md](./06_other_services.md#rpcbind-111) |
| 139, 445 | SMB | [04_smb_enum.md](./04_smb_enum.md) |
| 161 | SNMP | [06_other_services.md](./06_other_services.md#snmp-161-udp) |
| 389, 636 | LDAP | [05_ldap_ad_enum.md](./05_ldap_ad_enum.md) |
| 1433 | MSSQL | [06_other_services.md](./06_other_services.md#mssql-1433) |
| 2049 | NFS | [06_other_services.md](./06_other_services.md#nfs-2049) |
| 3306 | MySQL | [06_other_services.md](./06_other_services.md#mysql-3306) |
| 3389 | RDP | [06_other_services.md](./06_other_services.md#rdp-3389) |
| 5432 | PostgreSQL | [06_other_services.md](./06_other_services.md#postgresql-5432) |
| 5985, 5986 | WinRM | [06_other_services.md](./06_other_services.md#winrm-59855986) |
| 6379 | Redis | [06_other_services.md](./06_other_services.md#redis-6379) |
| 27017 | MongoDB | [06_other_services.md](./06_other_services.md#mongodb-27017) |

---

## Pro Tips

> **💡** Always save nmap output with `-oN`, `-oX`, or `-oA`. You'll want to search it later.

> **💡** If a web port is open, run gobuster **in parallel** while you explore manually in the browser.

> **💡** UDP is often overlooked. `SNMP (161)` and `TFTP (69)` live there and are common on CTF machines.

> **💡** If you see port 88 (Kerberos), 389 (LDAP), or 445 (SMB) — you're likely dealing with Active Directory. Jump straight to [05_ldap_ad_enum.md](./05_ldap_ad_enum.md).

---

*[← Setup](../01_setup/README.md) · [Back to Main](../README.md) · [Vuln Assessment →](../03_vuln_assessment/README.md)*
