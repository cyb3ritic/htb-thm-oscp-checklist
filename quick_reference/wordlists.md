# 📖 Wordlists Guide

> **The right wordlist at the right time.** Using the wrong list wastes hours.

---

## Quick Selection Guide

| Situation | Use This |
|---|---|
| First web directory scan | `common.txt` |
| Thorough web directory scan | `directory-list-2.3-medium.txt` |
| PHP site specifically | `directory-list-2.3-medium.txt` + `-x php,html,txt` |
| API endpoints | `api/api-endpoints.txt` |
| Subdomain / vhost discovery | `subdomains-top1million-5000.txt` |
| DNS brute force | `dns-Jhaddix.txt` |
| Password cracking (general) | `rockyou.txt` |
| Password spraying (corporate) | `common-corporate-passwords.txt` |
| Username brute force | `names.txt` or `xato-net-10-million-usernames.txt` |
| Default credentials | `default-credentials/` folder |
| SNMP community strings | `Discovery/SNMP/snmp.txt` |
| Parameter discovery | `burp-parameter-names.txt` |

---

## SecLists — Full Path Reference

### Web Content Discovery

```bash
# Fast / common (1K-5K entries) — first pass
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/quickhits.txt

# Medium (200K entries) — thorough
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
/usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt

# Large (2M entries) — deep enumeration
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt

# API endpoints
/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
/usr/share/seclists/Discovery/Web-Content/api/objects.txt

# Parameter names (for fuzzing GET/POST params)
/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# Backup files
/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt

# PHP-specific
/usr/share/seclists/Discovery/Web-Content/PHP.fuzz.txt
```

### DNS / Subdomains

```bash
# Fast (5K) — most common subdomains
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Medium (20K)
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# Large (1M) — very thorough
/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt

# Brute force DNS
/usr/share/seclists/Discovery/DNS/dns-Jhaddix.txt

# Fierce wordlist
/usr/share/seclists/Discovery/DNS/fierce-hostlist.txt
```

### Passwords

```bash
# The classic — rockyou (14M passwords)
/usr/share/wordlists/rockyou.txt

# 500 worst passwords (for quick checks)
/usr/share/seclists/Passwords/Common-Credentials/500-worst-passwords.txt

# Top 10K passwords
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt

# Default credentials (organized by service)
/usr/share/seclists/Passwords/Default-Credentials/

# Corporate password patterns
/usr/share/seclists/Passwords/Common-Credentials/best110.txt

# Leaked databases
/usr/share/seclists/Passwords/Leaked-Databases/

# Fast spray list
/usr/share/seclists/Passwords/Common-Credentials/common-credentials.txt
```

### Usernames

```bash
# Names (for username generation)
/usr/share/seclists/Usernames/Names/names.txt

# Top 10K usernames
/usr/share/seclists/Usernames/top-usernames-shortlist.txt

# Large username list
/usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt

# Statistically likely usernames
/usr/share/seclists/Usernames/cirt-default-usernames.txt
```

### SNMP

```bash
/usr/share/seclists/Discovery/SNMP/snmp.txt
/usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt
```

---

## Wordlist Tips

### 1. Always Start Fast, Scale Up

```bash
# First — fast list (seconds)
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/common.txt

# If nothing found — try medium (minutes)
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

### 2. Always Add Extensions

```bash
# Match your tech stack:
# PHP site
-x php,php3,php5,phtml,html,txt,bak,zip

# ASP.NET
-x aspx,asp,html,txt,config

# Java
-x jsp,jspx,html,txt

# Generic (when unsure)
-x html,php,aspx,jsp,txt,bak,zip,tar.gz
```

### 3. Filter False Positives

```bash
# Run once, note the size of non-existent pages, filter it:
ffuf -w wordlist.txt -u http://$IP/FUZZ -fs 1234   # filter by size
ffuf -w wordlist.txt -u http://$IP/FUZZ -fc 403    # filter status 403
```

### 4. Create Custom Wordlists

```bash
# CeWL — scrape website for words
cewl http://$IP -m 5 -w custom_wordlist.txt

# Add company name + variations
echo "Company2024
Company123
Welcome1
CompanyPass
companyIT" > corporate_spray.txt

# Username generation
john --wordlist=names.txt --rules=NT --stdout | head -1000 > usernames.txt
```

### 5. Combine Wordlists

```bash
cat wordlist1.txt wordlist2.txt | sort -u > combined.txt
```

---

## Install SecLists (If Missing)

```bash
sudo apt install seclists

# Or from GitHub:
git clone --depth 1 https://github.com/danielmiessler/SecLists /usr/share/seclists

# Or just download the specific file you need:
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/common.txt
```

---

*[← Tools Index](./tools_index.md) · [Quick Reference Hub](./README.md) · [File Transfers →](./file_transfers.md)*
