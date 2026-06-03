# Detection & Prevention — Shells & Payloads

Useful for knowing what defenders watch and what to avoid on evasive engagements.

---

## What Gets Flagged

File uploads to web apps — app logs capture these. File type, path, and timing all visible.

User commands that don't fit the account's role — `whoami`, `net user`, `net localgroup` from a non-admin process. PowerShell logging captures cmdlet usage and script blocks.

Port 4444 in NetFlow — default Meterpreter port is a known IOC. Any heartbeat on non-standard ports is suspicious. Change LPORT to 443, 80, or 8080 to blend in.

Cleartext shell traffic — unencrypted nc sessions expose every command in Wireshark/DPI. `net user hacker /add` visible in plain TCP stream. Use encrypted channels (HTTPS Meterpreter, SSH tunnels) on evasive engagements.

Anomalous SMB lateral movement — end host → end host SMB is unusual. Normal pattern is client → infra server.

---

## MITRE Tags

`T1190` Exploit Public-Facing Application (initial access via web shell) / `T1059` Command and Scripting Interpreter (execution) / `T1102`/`T1071` C2 over standard protocols

---

## Mitigations Worth Noting for Reports

App sandboxing, least privilege, host segmentation (DMZ for internet-facing hosts), firewall rules blocking non-established inbound connections and non-standard outbound ports. These are the controls that break bind shells and limit reverse shell egress paths.

> [!tip] If Meterpreter HTTPS is an option, always use it over raw TCP — encrypted traffic can't be read by DPI, and port 443 outbound is almost universally allowed.