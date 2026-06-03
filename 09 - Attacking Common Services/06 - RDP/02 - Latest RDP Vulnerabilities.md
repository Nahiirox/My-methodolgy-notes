# BlueKeep CVE-2019-0708

Unauthenticated pre-auth RCE on RDP. Use-After-Free in the virtual channel creation during initial connection handshake — no credentials required, no user interaction. Runs as LocalSystem.

Affected: Windows 7, Server 2008/2008 R2, XP, Server 2003. Server 2019/2016 and Win10 are not affected. ~25% of originally vulnerable hosts still unpatched.

Attack flow: manipulated init request → UAF in virtual channel function (Privileges: LocalSystem) → kernel memory freed and overwritten with attacker instructions → RCE callback.

> [!warning] BlueKeep is highly unstable — can BSOD the target. Do not run the exploit without explicit written client approval. Confirm the target is actually vulnerable and confirm the risk tolerance before touching it. In most engagements, document it as a finding and move on unless the client specifically wants it tested.

Check for vulnerability with Metasploit scanner before deciding whether to attempt exploitation:

```
use auxiliary/scanner/rdp/cve_2019_0708_bluekeep
set RHOSTS <TARGET>
run
```