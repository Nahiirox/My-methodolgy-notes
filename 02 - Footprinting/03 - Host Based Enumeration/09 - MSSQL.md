# MSSQL — Footprinting

Port TCP/1433 (default), TCP/1434 (DAC). Runs as `NT SERVICE\MSSQLSERVER` by default. Windows Authentication = domain/local SAM auth → compromised domain account = DB access.

Named pipes enabled = additional attack surface beyond TCP. Note the pipe path from Nmap output for later.

---

### Nmap

Full MSSQL script battery — hostname, instance name, version, named pipe path, NTLM info, empty `sa` password check:

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p1433 <TARGET>
```

MSF ping scan — confirms instance name, version, clustering, named pipe:

```
use auxiliary/scanner/mssql/mssql_ping
set rhosts <TARGET>
run
```

---

### Connect

Windows auth (domain or local account):

```bash
mssqlclient.py <USER>@<TARGET> -windows-auth
```

SQL auth:

```bash
mssqlclient.py <USER>:<PASS>@<TARGET>
```

Once in:

```sql
select name from sys.databases
```

> [!tip] SSMS saves credentials client-side — if you find a developer or admin workstation with SSMS installed, check for saved connections. Those credentials connect directly to the DB instance and are often reused elsewhere.

> [!warning] Default `sa` account is sometimes left enabled with a weak or empty password. Always test `sa` with empty password before anything else. Nmap `ms-sql-empty-password` catches this automatically.

Dangerous config to flag: unencrypted client connections + self-signed cert + named pipes enabled + `sa` not disabled = standard misconfigured MSSQL. Any one of these is a finding; all four together is critical.