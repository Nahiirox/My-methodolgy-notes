# Detection & Prevention — Pivoting & Tunneling

Useful for reporting recommendations and understanding what defenders watch for.

---

## What Gets Flagged

Protocol tunneling (T1572) — SSH-over-SSH, DNS tunneling, ICMP tunneling, HTTP C2 all have behavioral signatures. DNS tunneling: unusual TXT record volume, external DNS queries from internal hosts that should use internal DNS only. ICMP tunneling: ICMP traffic volume, payload size anomalies. HTTP/HTTPS C2: beaconing patterns (regular interval requests even if encrypted).

Non-standard ports (T1571) — common protocol on unexpected port (HTTP on 444, SSH on 8080) is a detection signal. Baseline normal port usage → deviations stand out.

LOTL — hard to detect without behavioral baseline. Watching for cmd/PS spawning from unusual parent processes, EDR/AV telemetry, network flow analysis.

Proxy use (T1090) — detect via unusual outbound connections, non-standard egress IPs, traffic to unknown external hosts. Allow/block list enforcement at egress helps most.

---

## Mitigations Worth Recommending

Network segmentation — users in HR/Finance shouldn't reach infrastructure management. Separate production, management, and user networks.

Egress filtering — restrict what protocols/ports can leave each segment. Disallow internal hosts making external DNS queries (only DNS servers should).

MFA on all remote access (RDP, SSH, VPN) — even with creds compromised, attacker needs second factor.

OOB management for network infrastructure — switches/routers should only be reachable from a management network, never from general user segments.

Dual-homed host inventory — document and audit all hosts with multiple NICs. These are the pivot candidates. Defenders should know every one of them.

SIEM + baseline — without normal traffic baseline, anomalies can't be spotted. Logging + correlation is the foundation.

---

## MITRE Tags for the Report

`T1133` External Remote Services / `T1021` Remote Services / `T1571` Non-Standard Ports / `T1572` Protocol Tunneling / `T1090` Proxy Use