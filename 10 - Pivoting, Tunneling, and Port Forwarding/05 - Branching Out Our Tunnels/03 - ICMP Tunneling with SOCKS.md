# ICMP Tunneling with ptunnel-ng

Traffic encapsulated in ICMP echo request/reply packets. Use when all TCP/UDP outbound is blocked but ping is allowed — rare but effective. Requires root on both ends.

---

## Setup

Clone and build:

```bash
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng && sudo ./autogen.sh
scp -r ptunnel-ng <USER>@<PIVOT_IP>:~/
```

Static binary build (if glibc version mismatch):

```bash
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

---

## Run

Start ptunnel-ng server on pivot — `-r` is the pivot's reachable IP, `-R` is the port to forward through the tunnel (SSH = 22):

```bash
sudo ./ptunnel-ng -r<PIVOT_IP> -R22
```

Start ptunnel-ng client on attack host — creates local port 2222, traffic exits at pivot:22:

```bash
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
```

SSH through the ICMP tunnel via local port 2222:

```bash
ssh -p2222 -l<USER> 127.0.0.1
```

Add SSH dynamic port forwarding on top for full proxychains access:

```bash
ssh -D 9050 -p2222 -l<USER> 127.0.0.1
```

Then scan/connect via proxychains as normal:

```bash
proxychains nmap -sV -sT <TARGET_IP> -p3389
```

> [!warning] Requires root on both ends (`sudo`). Also verify glibc versions match between your build host and the target — static build flag avoids this entirely.

> [!tip] Verify the tunnel is actually working with Wireshark: normal SSH shows TCP+SSHv2 traffic. ICMP tunnel shows only ICMP echo packets. If you still see TCP, the tunnel isn't being used.