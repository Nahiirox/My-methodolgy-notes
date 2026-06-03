# Pivoting, Tunneling & Port Forwarding — Intro & Networking Concepts

Pivoting = moving to other networks through a compromised host to reach previously isolated segments. Tunneling = encapsulating traffic inside another protocol to evade detection or bypass controls. Lateral movement = spreading wide within the same network segment. Different goals, often confused.

First thing on any new host: `ifconfig` / `ipconfig` — look for multiple NICs. A second NIC = potential pivot into another subnet.

---

## Key Checks on a Compromised Host

Look for additional network interfaces immediately:

```bash
ifconfig
```

```cmd
ipconfig /all
```

Check the routing table — reveals what networks the host can already reach:

```bash
netstat -r
# or
ip route
```

```cmd
route print
```

Any network in the routing table is potentially reachable through this host. Any route that was administratively added (not just directly connected) is worth noting — it was put there for a reason.

---

## What to Document

For every host you land on, capture: all IP addresses + subnet masks per NIC, default gateway, full routing table, any VPN/tunnel interfaces present. This data drives every pivot decision downstream.

`tun0` interface present → VPN is active on the host → may have routes to otherwise unreachable segments.

Dual-homed host (two real NICs with IPs in different subnets) = pivot candidate. This is the beachhead.

> [!tip] Document network topology as you discover it — Draw.io or even ASCII. Under exam pressure, a visual of hop1 → hop2 → target saves serious time when you need to chain tunnels.

Ports matter for pivoting — use protocols and ports that are permitted through the firewall. HTTP/80, HTTPS/443, SSH/22 are common egress paths. When crafting reverse shells through pivots, match the listener port to what's allowed outbound from the pivot host.