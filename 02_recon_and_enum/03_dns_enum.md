# 🔖 DNS Enumeration

> **DNS often reveals the full picture of the network** — subdomains, internal hostnames, mail servers, and more.

---

## Table of Contents
- [Basic DNS Queries](#basic-dns-queries)
- [Zone Transfer](#zone-transfer)
- [DNS Brute-Force](#dns-brute-force)
- [Subdomain Enumeration](#subdomain-enumeration)
- [Reverse DNS Lookup](#reverse-dns-lookup)
- [DNS Recon Tools](#dns-recon-tools)

---

## Basic DNS Queries

Always start with these to understand the DNS landscape:

- [ ] **Basic DNS lookup**:
  ```bash
  nslookup $IP
  nslookup <domain.tld> $IP   # Query specific nameserver
  ```
- [ ] **Dig — all record types**:
  ```bash
  dig any <domain.tld> @$IP     # All records
  dig a <domain.tld>            # A record (IPv4)
  dig aaaa <domain.tld>         # AAAA record (IPv6)
  dig mx <domain.tld>           # Mail servers
  dig ns <domain.tld>           # Nameservers
  dig txt <domain.tld>          # TXT records (often contain secrets/SPF)
  dig cname <domain.tld>        # CNAME
  dig soa <domain.tld>          # Start of Authority (admin email!)
  ```
- [ ] **Host command**:
  ```bash
  host -t any <domain.tld>
  host -t ns <domain.tld>
  ```

> **💡 Tip:** The SOA record's admin email (`admin@domain.tld` → `admin.domain.tld`) is often a real username.

---

## Zone Transfer

A misconfigured DNS server will dump the entire zone — all hostnames at once:

- [ ] **Zone transfer with dig**:
  ```bash
  dig axfr @$IP <domain.tld>
  dig axfr @<nameserver> <domain.tld>
  ```
- [ ] **Zone transfer with host**:
  ```bash
  host -l <domain.tld> $IP
  ```
- [ ] **Zone transfer with dnsrecon**:
  ```bash
  dnsrecon -d <domain.tld> -t axfr
  ```
- [ ] **Zone transfer with nmap**:
  ```bash
  nmap --script dns-zone-transfer -p 53 $IP
  ```

> **💡 Tip:** If zone transfer works, add ALL discovered hostnames to `/etc/hosts` and enumerate each one.

---

## DNS Brute-Force

When zone transfer fails, brute-force subdomains:

- [ ] **dnsrecon**:
  ```bash
  dnsrecon -d <domain.tld> \
    -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t brt
  ```
- [ ] **dnsenum**:
  ```bash
  dnsenum <domain.tld> --dnsserver $IP \
    -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
  ```
- [ ] **Gobuster DNS mode**:
  ```bash
  gobuster dns -d <domain.tld> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -r $IP
  ```

---

## Subdomain Enumeration

- [ ] **FFuF — vhost-based subdomain discovery**:
  ```bash
  ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -H "Host: FUZZ.<domain.tld>" -u http://$IP \
    -fs <size_of_404_response>
  ```
- [ ] **Amass** (passive + active):
  ```bash
  amass enum -d <domain.tld>
  amass enum -d <domain.tld> -active -brute
  ```
- [ ] **Subfinder** (passive):
  ```bash
  subfinder -d <domain.tld>
  ```
- [ ] **Assetfinder**:
  ```bash
  assetfinder --subs-only <domain.tld>
  ```

> **💡 Tip:** Combine passive tools (subfinder, amass passive) with active brute-forcing for best coverage.

---

## Reverse DNS Lookup

If you have an IP range, look for hostnames:

- [ ] **Reverse lookup**:
  ```bash
  dig -x $IP
  nslookup $IP
  ```
- [ ] **Reverse lookup with dnsrecon**:
  ```bash
  dnsrecon -r <subnet/24> -n $IP
  ```

---

## DNS Recon Tools

- [ ] **dnsrecon — comprehensive scan**:
  ```bash
  dnsrecon -d <domain.tld> -t std       # Standard recon
  dnsrecon -d <domain.tld> -t brt       # Brute force
  dnsrecon -d <domain.tld> -t axfr      # Zone transfer
  dnsrecon -d <domain.tld> -t rvl -r <subnet/24>  # Reverse lookup
  ```
- [ ] **Nmap DNS scripts**:
  ```bash
  nmap -p 53 --script dns-brute,dns-recursion,dns-cache-snoop $IP
  ```

---

## 🔗 References

- [HackTricks — DNS](https://book.hacktricks.xyz/network-services-pentesting/pentesting-dns)
- [dnsrecon GitHub](https://github.com/darkoperator/dnsrecon)

---

*[← Web Enumeration](./02_web_enum.md) · [Recon Hub](./README.md) · [SMB Enumeration →](./04_smb_enum.md)*
