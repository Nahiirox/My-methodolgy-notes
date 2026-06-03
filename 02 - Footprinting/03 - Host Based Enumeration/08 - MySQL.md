# MySQL — Footprinting

Port TCP/3306. Nmap `mysql*` scripts fire automatically — check for empty root password, valid usernames, version info:

```bash
sudo nmap <TARGET> -sV -sC -p3306 --script mysql*
```

> [!warning] Nmap `mysql-brute` and `mysql-empty-password` results are noisy — false positives are common. Always manually confirm before reporting empty root password as a finding.

---

### Connect

No password:

```bash
mysql -u root -h <TARGET>
```

With password (no space between `-p` and the password):

```bash
mysql -u <USER> -p<PASS> -h <TARGET>
```

---

### Enumeration Queries

```sql
show databases;
use <DB>;
show tables;
show columns from <TABLE>;
select * from <TABLE>;
select * from <TABLE> where <COLUMN> = "<STRING>";
select version();
```

High-value tables to check: `mysql.user` (hashes, privileges), `information_schema.tables` (full DB map), any `users`/`accounts`/`config`/`settings` table in app DBs.

Check who's connecting and from where:

```sql
use sys;
select host, unique_users from host_summary;
```

> [!tip] `/etc/mysql/mysql.conf.d/mysqld.cnf` contains `user`, `password`, and `admin_address` in plaintext. If you have LFI or file read → grab this file immediately. Also check `secure_file_priv` value — empty string means `LOAD_FILE()` and `INTO OUTFILE` work anywhere → file read/write on the host OS.

`debug=ON` or `sql_warnings=ON` in config → verbose error messages leak schema info and query structure → feeds SQLi enumeration.