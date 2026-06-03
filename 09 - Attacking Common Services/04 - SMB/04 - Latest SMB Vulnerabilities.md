# SMBGhost CVE-2020-0796

SMB v3.1.1 compression integer overflow. Windows 10 versions 1903/1909. Unauthenticated RCE as SYSTEM. Patched — but unpatched legacy hosts still appear in enterprise environments.

---

## What It Is

Integer overflow in the SMB driver's compression handling during session negotiation (`Negotiate Protocol Response`). Oversized compressed packet overflows a buffer → overwrites CPU instructions → attacker-controlled code executes in kernel context with SYSTEM privileges.

No auth required — only TCP/445 reachable.

---

## Attack Flow (Source → Process → Privileges → Destination)

Phase 1 — Buffer overflow: attacker sends malformed compressed packet (Source) → SMB driver processes it during negotiation (Process) → runs with SYSTEM/admin privileges (Privileges) → local process memory is the target (Destination)

Phase 2 — RCE: overwritten buffer now contains attacker instructions (Source) → CPU executes them instead of the original code path (Process) → still SYSTEM (Privileges) → callback to attacker (Destination)

---

## Detection / Mitigation Note

Check SMB version in Nmap output — `SMB v3.1.1` + Windows 10 1903/1909 + no patch = vulnerable. Mitigation: disable SMB compression (registry) or patch. SMB signing doesn't protect against this one — it's pre-auth.

> [!warning] PoCs for SMBGhost exist publicly but are highly unstable — can BSOD the target. Confirm with client before running against production systems. Verify the patch status with Nmap scripts or CME before attempting.