# 🌐 Network Scanning

> **Goal:** Discover all open ports and services on the target as quickly and thoroughly as possible.

---

## Table of Contents
- [Nmap — Quick Scans](#nmap--quick-scans)
- [Nmap — Full & Targeted](#nmap--full--targeted)
- [Nmap — Advanced Options](#nmap--advanced-options)
- [UDP Scanning](#udp-scanning)
- [RustScan (Fast)](#rustscan-fast)
- [Masscan (Fastest)](#masscan-fastest)
- [Recommended Scan Workflow](#recommended-scan-workflow)

---

## Nmap — Quick Scans

Start these immediately when you get a target IP:

- [ ] **Default scripts + service detection** (good starting point):
  ```bash
  nmap -sC -sV $IP -oN nmap/initial.txt
  ```
- [ ] **Top 1000 ports** (faster):
  ```bash
  nmap --top-ports 1000 -sV $IP -oN nmap/top1000.txt
  ```
- [ ] **Fast SYN scan**:
  ```bash
  nmap -sS -T4 -F $IP -oN nmap/fast.txt
  ```

---

## Nmap — Full & Targeted

Run these while working on initial results:

- [ ] **All 65535 TCP ports** (essential — don't skip):
  ```bash
  nmap -p- --min-rate 5000 -T4 $IP -oN nmap/alltcp.txt
  ```
- [ ] **Targeted scan** (once you have all open ports from above):
  ```bash
  # Replace with your actual ports
  nmap -sC -sV -p 22,80,443,8080 $IP -oN nmap/targeted.txt
  ```
- [ ] **Aggressive scan** (adds OS detection + traceroute):
  ```bash
  nmap -A -T4 $IP -oN nmap/aggressive.txt
  ```
- [ ] **OS detection** only:
  ```bash
  sudo nmap -O $IP -oN nmap/os.txt
  ```
- [ ] **Save all formats** (normal, XML, grepable):
  ```bash
  nmap -sC -sV -p- $IP -oA nmap/full
  ```

---

## Nmap — Advanced Options

- [ ] **Vulnerability scripts**:
  ```bash
  nmap --script vuln -p <ports> $IP -oN nmap/vuln.txt
  ```
- [ ] **SMB vulnerabilities specifically**:
  ```bash
  nmap --script smb-vuln* -p 445 $IP
  ```
- [ ] **Banner grabbing**:
  ```bash
  nmap -sV --version-intensity 9 -p <ports> $IP
  ```
- [ ] **Firewall evasion** (if scans are getting blocked):
  ```bash
  nmap -f $IP                    # Fragment packets
  nmap -D RND:10 $IP             # Add decoy IPs
  nmap --source-port 53 $IP      # Spoof source port
  nmap -T1 --scan-delay 2s $IP  # Slow scan to evade IDS
  nmap -sF $IP                   # FIN scan (bypasses some filters)
  nmap -sX $IP                   # Xmas scan
  ```
- [ ] **Script categories**:
  ```bash
  nmap --script "default and safe" $IP
  nmap --script "not intrusive" $IP
  ```

---

## UDP Scanning

> **⚠️ Important:** Many services live on UDP and are missed by TCP-only scans. Always do at least a top-100 UDP scan.

- [ ] **Top 20 common UDP ports** (fast):
  ```bash
  sudo nmap -sU --top-ports 20 $IP -oN nmap/udp_top20.txt
  ```
- [ ] **Top 100 UDP ports** (recommended):
  ```bash
  sudo nmap -sU --top-ports 100 $IP -oN nmap/udp.txt
  ```

**Common UDP services to look for:**

| Port | Service | Why it matters |
|---|---|---|
| 53 | DNS | Zone transfer, cache poisoning |
| 67/68 | DHCP | Rogue DHCP |
| 69 | TFTP | Unauthenticated file access |
| 111 | RPCBind | NFS disclosure |
| 123 | NTP | Amplification, info leak |
| 161 | SNMP | Community string → system info, creds |
| 500 | IKE/IPSec | VPN misconfig |

---

## RustScan (Fast)

RustScan scans all 65535 ports in seconds, then pipes to Nmap for service detection.

- [ ] **Install**:
  ```bash
  # Download from: https://github.com/RustScan/RustScan/releases
  # Or: cargo install rustscan
  ```
- [ ] **Basic scan** (pipe to nmap):
  ```bash
  rustscan -a $IP -- -sC -sV -oN nmap/rustscan.txt
  ```
- [ ] **Scan with custom rate**:
  ```bash
  rustscan -a $IP --ulimit 5000 -- -sC -sV
  ```

> **💡 Tip:** RustScan is significantly faster than nmap for initial port discovery. Use it to find ports, then run targeted nmap for service details.

---

## Masscan (Fastest)

Best for large /16 or /8 networks. Overkill for single targets but very fast.

- [ ] **All ports**:
  ```bash
  sudo masscan -p1-65535,U:1-65535 $IP --rate=10000 -oL masscan_output.txt
  ```
- [ ] **Specific rate** (lower if network is unstable):
  ```bash
  sudo masscan -p1-65535 $IP --rate=1000
  ```

---

## Recommended Scan Workflow

```bash
# STEP 1: Launch full port scan in background immediately
nmap -p- --min-rate 5000 $IP -oN nmap/alltcp.txt &

# STEP 2: Quick initial scan for fast results while waiting
nmap -sC -sV --top-ports 1000 $IP -oN nmap/initial.txt

# STEP 3: While step 1 & 2 run, start manual enumeration
# of anything obvious (web page, FTP anonymous login, etc.)

# STEP 4: Once step 1 completes, run targeted scan on ALL found ports
PORTS=$(grep "open" nmap/alltcp.txt | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -sC -sV -p $PORTS $IP -oN nmap/targeted.txt

# STEP 5: UDP scan
sudo nmap -sU --top-ports 100 $IP -oN nmap/udp.txt
```

---

## 🔗 References

- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [RustScan GitHub](https://github.com/RustScan/RustScan)
- [Nmap Script Database](https://nmap.org/nsedoc/)

---

*[← Recon Hub](./README.md) · [Next: Web Enumeration →](./02_web_enum.md)*
