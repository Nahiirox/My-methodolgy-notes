# Vhost Fuzzing (ffuf)

Fuzz the `Host:` header against the known IP — hits both public and non-public vhosts without touching `/etc/hosts`:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<DOMAIN>:<PORT>/ -H 'Host: FUZZ.<DOMAIN>'
```

Everything comes back 200 with identical size — that's expected, the server accepts any Host header. A real vhost returns a **different response size** because it serves different content. Filter on the baseline size to cut the noise (get the baseline size from the uniform results, then `-fs <SIZE>`).

> [!warning] All-200 output with identical sizes is not results — it's noise. Don't stop here, filter before reading anything into it.