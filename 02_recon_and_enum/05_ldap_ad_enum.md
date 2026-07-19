# 🏛️ LDAP & Active Directory Enumeration

> **If you see ports 88 (Kerberos), 389 (LDAP), or 3268 (Global Catalog) — you're on a domain controller.** This section covers initial AD enumeration before moving to [Phase 8 — Active Directory Attacks](../08_active_directory/README.md).

---

## Table of Contents
- [Identifying AD Environment](#identifying-ad-environment)
- [LDAP Enumeration](#ldap-enumeration)
- [Kerbrute — User Enumeration](#kerbrute--user-enumeration)
- [BloodHound & SharpHound](#bloodhound--sharphound)
- [PowerView Enumeration](#powerview-enumeration)
- [Automated AD Enum Tools](#automated-ad-enum-tools)

---

## Identifying AD Environment

Signs you're dealing with Active Directory:

| Indicator | What to look for |
|---|---|
| Port 88 open | Kerberos — almost certainly a DC |
| Port 389 / 636 | LDAP / LDAPS |
| Port 3268 / 3269 | Global Catalog |
| SMB returns domain name | `enum4linux-ng` shows domain |
| DNS records like `_kerberos._tcp` | AD integrated DNS |
| Machine name ends in `.htb` or has FQDN | Domain-joined |

```bash
# Quick check — AD or not?
nmap -p 88,389,445,3268 $IP
crackmapexec smb $IP    # shows domain, hostname, OS
```

---

## LDAP Enumeration

- [ ] **Check if LDAP is accessible (anonymous)**:
  ```bash
  ldapsearch -x -H ldap://$IP -s base namingcontexts
  # Look for the base DN, e.g., DC=domain,DC=local
  ```
- [ ] **Enumerate with anonymous bind**:
  ```bash
  ldapsearch -x -H ldap://$IP -b "DC=<domain>,DC=<tld>" "(objectClass=*)"
  ```
- [ ] **Get users**:
  ```bash
  ldapsearch -x -H ldap://$IP -b "DC=<domain>,DC=<tld>" "(objectClass=user)" sAMAccountName
  ```
- [ ] **Get all objects (dump everything)**:
  ```bash
  ldapsearch -x -H ldap://$IP -b "DC=<domain>,DC=<tld>" \
    "(objectClass=*)" > ldap_dump.txt
  ```
- [ ] **Authenticated LDAP search**:
  ```bash
  ldapsearch -x -H ldap://$IP -D "<user>@<domain>" -w "<pass>" \
    -b "DC=<domain>,DC=<tld>" "(objectClass=user)"
  ```
- [ ] **ldapdomaindump** (dumps AD info into readable HTML/JSON):
  ```bash
  ldapdomaindump -u '<domain>\<user>' -p '<pass>' $IP -o ldap_dump/
  # Open domain_users.html in browser for readable view
  ```
- [ ] **windapsearch** (Go-based, fast):
  ```bash
  windapsearch -d <domain> --dc $IP -u <user> -p <pass> --users
  windapsearch -d <domain> --dc $IP -u <user> -p <pass> --computers
  windapsearch -d <domain> --dc $IP -u <user> -p <pass> --groups
  windapsearch -d <domain> --dc $IP -u <user> -p <pass> --da  # domain admins
  ```
- [ ] **Check for password in LDAP description fields**:
  ```bash
  ldapsearch -x -H ldap://$IP -b "DC=<domain>,DC=<tld>" \
    "(objectClass=user)" description | grep -i "pass\|pwd\|cred"
  ```

---

## Kerbrute — User Enumeration

Kerbrute can enumerate valid usernames **without credentials** by testing Kerberos pre-authentication:

- [ ] **Install Kerbrute**:
  ```bash
  # Download from: https://github.com/ropnop/kerbrute/releases
  chmod +x kerbrute_linux_amd64
  ```
- [ ] **Username enumeration**:
  ```bash
  ./kerbrute userenum -d <domain> --dc $IP \
    /usr/share/seclists/Usernames/Names/names.txt
  ```
- [ ] **Password spray** (after finding valid users):
  ```bash
  ./kerbrute passwordspray -d <domain> --dc $IP users.txt '<password>'
  ```
- [ ] **Brute force specific user**:
  ```bash
  ./kerbrute bruteuser -d <domain> --dc $IP /usr/share/wordlists/rockyou.txt <user>
  ```

> **⚠️ OSCP Note:** Password spraying can lock accounts. Spray carefully — max 1-2 attempts per user per window.

---

## BloodHound & SharpHound

BloodHound maps AD attack paths visually. **Requires credentials** (or domain access).

- [ ] **Start BloodHound**:
  ```bash
  sudo neo4j start       # Start Neo4j database
  bloodhound             # Launch BloodHound GUI
  # Default creds: neo4j / neo4j (change on first login)
  ```
- [ ] **Collect data with SharpHound** (on Windows target):
  ```powershell
  # Upload SharpHound.ps1 or SharpHound.exe to target
  Import-Module SharpHound.ps1
  Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Temp\
  ```
- [ ] **Collect data with BloodHound-python** (from attacker Linux machine):
  ```bash
  pip3 install bloodhound
  bloodhound-python -u <user> -p <pass> -d <domain> -ns $IP -c All
  # Creates JSON files to import into BloodHound
  ```
- [ ] **Import data**: BloodHound GUI → Upload Data → Select JSON files
- [ ] **Key BloodHound queries to run**:
  ```
  Find All Domain Admins
  Find Shortest Paths to Domain Admins
  Find Principals with DCSync Rights
  Find Computers where Domain Users are Local Admin
  Shortest Paths from Kerberoastable Users
  ```

---

## PowerView Enumeration

Run on a Windows target once you have a foothold:

```powershell
# Load PowerView
Import-Module PowerView.ps1
# Or bypass execution policy:
powershell -ep bypass -c "Import-Module .\PowerView.ps1"
```

- [ ] **Domain info**:
  ```powershell
  Get-NetDomain              # Domain details
  Get-DomainController       # List DCs
  Get-DomainPolicy           # Password policy
  ```
- [ ] **User enumeration**:
  ```powershell
  Get-NetUser                        # All users
  Get-NetUser | select samaccountname, description  # Descriptions (may contain passwords!)
  Get-NetUser -SPN                   # Kerberoastable users (have SPNs)
  Get-NetUser -PreauthNotRequired    # AS-REP roastable users
  ```
- [ ] **Group enumeration**:
  ```powershell
  Get-NetGroup                       # All groups
  Get-NetGroup "Domain Admins"       # Members of group
  Get-NetGroupMember "Domain Admins" # Recursive membership
  ```
- [ ] **Computer enumeration**:
  ```powershell
  Get-NetComputer                    # All computers
  Get-NetComputer -OperatingSystem "*Server*"
  ```
- [ ] **Share enumeration**:
  ```powershell
  Find-DomainShare                   # All accessible shares
  Find-DomainShare -CheckShareAccess # Only accessible ones
  ```
- [ ] **ACL / permission analysis**:
  ```powershell
  Get-ObjectAcl -Identity "<user>" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "Write"}
  Find-InterestingDomainAcl          # Interesting ACEs (GenericAll, WriteDACL, etc.)
  ```
- [ ] **Trust enumeration**:
  ```powershell
  Get-NetDomainTrust                 # Domain trusts
  Get-NetForestTrust                 # Forest trusts
  ```

---

## Automated AD Enum Tools

- [ ] **ADRecon** (comprehensive AD report):
  ```powershell
  # On Windows target
  .\ADRecon.ps1
  ```
- [ ] **PingCastle** (AD health / attack surface report):
  ```
  # Run PingCastle.exe on the domain
  ```
- [ ] **enum4linux-ng on AD-joined machines**:
  ```bash
  enum4linux-ng -A -C $IP
  ```

---

## 🔗 References

- [HackTricks — Active Directory](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [BloodHound GitHub](https://github.com/BloodHoundAD/BloodHound)
- [PowerView Cheatsheet](https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993)
- [Kerbrute GitHub](https://github.com/ropnop/kerbrute)
- [ldapdomaindump GitHub](https://github.com/dirkjanm/ldapdomaindump)

---

*[← SMB Enumeration](./04_smb_enum.md) · [Recon Hub](./README.md) · [Other Services →](./06_other_services.md)*
