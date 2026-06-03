# Enumeration Methodology & Principles

Enumerate everything before you exploit anything. Brute-forcing SSH/RDP/WinRM on first contact is wrong — noisy, gets you blocked, and skips the actual goal. The goal is finding all paths in, not forcing the first one you see.

Enumeration is a loop, not a checklist. Every piece of info you pull feeds the next round. OSINT is separate — passive only, done before active enum starts.

---

### 6-Layer Model

Work outward → inward. Each layer feeds the next.

`1 — Internet Presence` — domains, subdomains, vhosts, ASN, netblocks, IPs, cloud instances, security measures

`2 — Gateway` — firewall, DMZ, IPS/IDS, EDR, proxies, NAC, VPN, Cloudflare — understand what's protecting the perimeter before touching it

`3 — Accessible Services` — service type, version, config, port, interface — this is where most of the module-level work lives

`4 — Processes` — what happens when the service runs, what data flows where, PID/source/destination

`5 — Privileges` — users, groups, permissions on each service — this is where misconfigs and over-provisioning hide

`6 — OS Setup` — OS type, patch level, network config, config files, sensitive files — internal access only

> [!tip] Layers 1–2 don't apply to internal AD-only engagements. Start at Layer 3 once you're inside the perimeter.

---

### The Decision Filter

Before moving on from any service or host, run through: what can I see → why is it there → what does it imply → what can't I see → why might it be hidden.

Gaps you find don't all lead forward. Not every filtered port is worth forcing. Map the whole surface first, then pick the path.

Three principles that don't change regardless of the target:

`1` — There is always more than the obvious. Check what's absent, not just what's present.

`2` — Distinguish what you see from what you cannot see.

`3` — There is always more info available. If you're stuck, you haven't understood the target yet — go back and enumerate deeper, don't escalate to force.