# Automating Recon

Other frameworks worth knowing exist: Recon-ng, theHarvester, SpiderFoot, OSINT Framework — module covers them by name only, no commands.

FinalRecon covers headers, whois, SSL, crawl, DNS, subdomain enum, dir brute, wayback, port scan in one tool.

Install:

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git && cd FinalRecon && pip3 install -r requirements.txt && chmod +x ./finalrecon.py
```

Quick targeted run — headers + whois:

```bash
./finalrecon.py --headers --whois --url http://<TARGET>
```

Full sweep:

```bash
./finalrecon.py --full --url http://<TARGET>
```

Key flags: `--headers` / `--sslinfo` / `--whois` / `--crawl` / `--dns` / `--sub` / `--dir` / `--wayback` / `--ps`

Useful tweaks: `-w <WORDLIST>` for dir enum / `-e txt,php,xml` for extensions / `-d <DNS_SERVER>` custom resolver / `-o json` output format / `-k shodan@<KEY>` for API-backed subdomain sources

Output lands in `~/.local/share/finalrecon/dumps/` by default. Change with `-cd <PATH>`.