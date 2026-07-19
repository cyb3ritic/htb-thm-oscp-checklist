# 🌍 Web Enumeration

> **If HTTP/HTTPS is open, this section is your playbook.** Web services are the #1 attack surface on CTF machines.

---

## Table of Contents
- [Manual Browser Recon](#manual-browser-recon)
- [Tech Stack Identification](#tech-stack-identification)
- [Directory & File Fuzzing](#directory--file-fuzzing)
- [Virtual Host Discovery](#virtual-host-discovery)
- [Parameter & Input Discovery](#parameter--input-discovery)
- [API Enumeration](#api-enumeration)
- [Web Scanner Tools](#web-scanner-tools)
- [HTTPS / Certificate Analysis](#https--certificate-analysis)
- [JavaScript Analysis](#javascript-analysis)

---

## Manual Browser Recon

Do this **before** running any scanners:

- [ ] Browse the entire site manually — every link, form, and button
- [ ] **Check these paths manually first**:
  ```
  robots.txt          ← often reveals hidden paths
  sitemap.xml         ← full site structure
  .git/               ← exposed git repo (massive win)
  .svn/               ← SVN repo
  .env                ← environment variables / credentials
  /backup             ← backups
  /admin, /administrator, /manage, /panel
  /login, /signin, /dashboard
  /api, /api/v1, /api/v2
  /config.php, /config.yml, /settings.php
  /phpinfo.php        ← PHP configuration dump
  /.well-known/       ← may reveal internal info
  ```
- [ ] **Read page source** — look for comments, credentials, hidden fields, JS file paths
- [ ] **Check cookies** — look at cookie names/values; JWT tokens? Base64 blobs?
- [ ] **Check forms** — every input field is a potential injection point
- [ ] **HTTP headers** — what server/framework is running? Hidden info?
  ```bash
  curl -I http://$IP
  curl -I https://$IP
  ```

---

## Tech Stack Identification

- [ ] **WhatWeb** (fast fingerprinting):
  ```bash
  whatweb http://$IP
  whatweb -v http://$IP   # verbose
  ```
- [ ] **Wappalyzer** — browser extension for visual stack detection
- [ ] **Manual checks**:
  ```bash
  # Headers
  curl -sI http://$IP | grep -i "server\|x-powered-by\|x-generator\|via"
  
  # Check response body for tech hints
  curl -s http://$IP | grep -i "wordpress\|joomla\|drupal\|django\|rails\|laravel"
  ```
- [ ] **WAF Detection**:
  ```bash
  wafw00f http://$IP
  ```
- [ ] **SSL/TLS info** (version, cipher, cert details):
  ```bash
  nmap --script ssl-enum-ciphers -p 443 $IP
  ```

---

## Directory & File Fuzzing

> **💡 Tip:** Always add file extensions relevant to the detected tech stack (PHP site → `.php`, ASP.NET → `.aspx`, Java → `.jsp`)

- [ ] **Gobuster — Common wordlist** (fast first pass):
  ```bash
  gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -x php,txt,html,bak,old,zip -o web/gobuster_common.txt
  ```
- [ ] **Gobuster — Medium wordlist** (more thorough):
  ```bash
  gobuster dir -u http://$IP \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,txt,html,asp,aspx,jsp -o web/gobuster_medium.txt
  ```
- [ ] **FFuF** (fastest fuzzer, great for filtering):
  ```bash
  # Basic dir fuzzing
  ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -u http://$IP/FUZZ -e .php,.txt,.html,.bak -mc 200,301,302,403 \
    -o web/ffuf_results.json

  # Filter by response size (remove false positives)
  ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -u http://$IP/FUZZ -fs <false_positive_size>
  ```
- [ ] **Feroxbuster** (recursive, great for deep directories):
  ```bash
  feroxbuster -u http://$IP \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -x php,txt,html -r -o web/feroxbuster.txt
  ```
- [ ] **Dirsearch**:
  ```bash
  dirsearch -u http://$IP -e php,txt,html,js,bak -x 404,403
  ```

**Extension cheatsheet by tech:**

| Tech | Extensions to add |
|---|---|
| PHP | `.php, .php3, .php5, .phtml, .inc` |
| ASP.NET | `.aspx, .asp, .ashx, .asmx, .config` |
| Java | `.jsp, .jspx, .do, .action` |
| Python | `.py, .pyc` |
| General | `.bak, .old, .swp, .tar.gz, .zip, .txt, .log` |

---

## Virtual Host Discovery

> **💡 If you find a domain name**, add it to `/etc/hosts` and enumerate for virtual hosts.

- [ ] **Add to /etc/hosts**:
  ```bash
  echo "$IP  <domain.tld>" | sudo tee -a /etc/hosts
  ```
- [ ] **FFuF vhost fuzzing**:
  ```bash
  ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -H "Host: FUZZ.<domain.tld>" -u http://$IP -fs <default_size>
  ```
  > Filter by `-fs` (the default response size for non-existent vhosts)
- [ ] **Gobuster vhost**:
  ```bash
  gobuster vhost -u http://<domain.tld> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    --append-domain
  ```
- [ ] **Add any found subdomains to /etc/hosts** and enumerate them too

---

## Parameter & Input Discovery

- [ ] **FFuF parameter fuzzing** (GET parameters):
  ```bash
  ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -u "http://$IP/page?FUZZ=test" -mc 200,302
  ```
- [ ] **Arjun** (smart parameter discovery):
  ```bash
  python3 arjun.py -u http://$IP/page
  ```
- [ ] **POST parameter fuzzing**:
  ```bash
  ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -X POST -d "FUZZ=test" -u http://$IP/page -mc 200,302
  ```

---

## API Enumeration

- [ ] **API endpoint discovery**:
  ```bash
  gobuster dir -u http://$IP/api \
    -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
    -o web/api_endpoints.txt
  ```
- [ ] **Common API patterns**:
  ```
  /api/v1/users
  /api/v1/admin
  /api/users/1          ← IDOR — try changing ID
  /api/users/me
  /graphql              ← GraphQL introspection
  /swagger.json         ← API docs exposed
  /openapi.json
  /.well-known/openid-configuration   ← OAuth discovery
  ```
- [ ] **GraphQL introspection**:
  ```bash
  curl -s -X POST http://$IP/graphql \
    -H "Content-Type: application/json" \
    -d '{"query":"{__schema{types{name}}}"}'
  ```

---

## Web Scanner Tools

- [ ] **Nikto** (web vulnerability scanner):
  ```bash
  nikto -h http://$IP -o web/nikto.txt
  ```
- [ ] **Nuclei** (template-based scanner — fast, accurate):
  ```bash
  nuclei -u http://$IP -t /path/to/nuclei-templates/ -o web/nuclei.txt
  
  # Or with specific tags
  nuclei -u http://$IP -tags cve,xss,sqli
  ```

> **⚠️ OSCP Note:** Check the current OSCP ruleset before using automated scanners. Nikto/Nuclei may be restricted in exam conditions.

---

## HTTPS / Certificate Analysis

Certificates often leak hostnames and internal domains:

- [ ] **View certificate**:
  ```bash
  openssl s_client -connect $IP:443 2>/dev/null | openssl x509 -noout -text
  ```
- [ ] **Check Subject Alternative Names (SANs)**:
  ```bash
  openssl s_client -connect $IP:443 2>/dev/null | openssl x509 -noout -text | grep "DNS:"
  ```
- [ ] **In browser**: Click the padlock → Certificate → Details → SANs

> **💡 Tip:** Internal hostnames in certificates (e.g., `internal.company.htb`) are goldmines — add them to `/etc/hosts` and enumerate.

---

## JavaScript Analysis

JS files often contain API keys, endpoints, credentials, and hints:

- [ ] **Dump all JS files from a page**:
  ```bash
  # Use browser DevTools (F12) → Sources tab to browse JS
  # Or extract links from HTML:
  curl -s http://$IP | grep -Eo 'src="[^"]*\.js"' | sed 's/src="//g' | sed 's/"//g'
  ```
- [ ] **Search JS for secrets**:
  ```bash
  # After downloading JS files:
  grep -i "api_key\|apikey\|password\|secret\|token\|auth\|key" *.js
  ```
- [ ] **LinkFinder** (extract endpoints from JS):
  ```bash
  python3 linkfinder.py -i http://$IP -d -o cli
  ```
- [ ] **gau** (fetch known URLs from Wayback Machine / AlienVault):
  ```bash
  gau $IP
  echo "$IP" | gau | tee web/gau_urls.txt
  ```

---

## 🔗 References

- [PayloadsAllTheThings — Web](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks — Web Security](https://book.hacktricks.xyz/pentesting-web)
- [SecLists Discovery](https://github.com/danielmiessler/SecLists/tree/master/Discovery)
- [FFuF GitHub](https://github.com/ffuf/ffuf)

---

*[← Network Scanning](./01_network_scanning.md) · [Recon Hub](./README.md) · [DNS Enumeration →](./03_dns_enum.md)*
