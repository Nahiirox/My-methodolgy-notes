# MSF — Database

Start postgres and init before first use:

```bash
sudo systemctl start postgresql
sudo msfdb init
sudo msfdb run
```

If init errors out, reinit:

```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

Confirm connected inside msfconsole:

```
msf6 > db_status
```

Workspaces — one per target/engagement, keeps results isolated:

```
msf6 > workspace -a <NAME>
msf6 > workspace <NAME>
msf6 > workspace              # list all, * = active
```

Run Nmap and auto-populate DB in one shot:

```
msf6 > db_nmap -sV -sS <TARGET>
```

Import existing Nmap XML (preferred over .nmap plaintext):

```
msf6 > db_import Target.xml
```

Query what's been found:

```
msf6 > hosts
msf6 > services
msf6 > creds
msf6 > loot
```

Feed DB results directly into RHOSTS — no copy-pasting IPs:

```
msf6 > hosts -R
msf6 > services -p 445 -R
```

Export before ending session:

```
msf6 > db_export -f xml <OUTFILE>.xml
```

> [!tip] `services -p 445 -R` sets RHOSTS to every host with 445 open — fast setup for SMB module runs across a subnet.