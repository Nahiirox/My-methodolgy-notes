# Brute Force — Attack Selection & Default Creds

Attack type decision logic:

Simple brute → no info on password, small keyspace only. Dictionary → likely weak/common password. Hybrid → likely modified common password (append char/year). Credential stuffing → have breach dump, suspect reuse. Password spraying → lockout policy in place, spread attempts across many accounts. Reverse brute → know a password, many unknown usernames.

> [!warning] Password spraying ≠ brute force per account — one or two attempts per user, move on. Lockout policies will burn you if you hammer a single account.

Default creds worth trying before any wordlist attack — many never get changed:

|Device|User|Pass|
|---|---|---|
|Linksys/D-Link/TP-Link/Asus/Cisco-adjacent|`admin`|`admin`|
|Netgear / Belkin|`admin`|`password`|
|Cisco|`cisco`|`cisco`|
|Zyxel|`admin`|`1234`|
|Ubiquiti UniFi|`ubnt`|`ubnt`|
|Hikvision / Panasonic DVR|`admin`|`12345`|
|Axis camera|`root`|`pass`|
|Samsung SmartCam|`admin`|`4321`|

Full list: `Default-Credentials/default-passwords.txt` in SecLists.

> [!tip] Default username often stays even after password change — `admin`/`root`/`ubnt` as fixed `-l`, wordlist on `-P` only cuts the keyspace in half immediately.