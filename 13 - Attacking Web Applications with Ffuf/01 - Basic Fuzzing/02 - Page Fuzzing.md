# Page Fuzzing (ffuf)

Unknown extension → fuzz it on a known file (`index` exists on almost everything). The extensions wordlist includes the dot, don't add one:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<TARGET>:<PORT>/blog/indexFUZZ
```

403 on `.phps` means it exists but is blocked — note it. 200 on `.php` confirms the stack, now fuzz pages with that extension:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/blog/FUZZ.php
```

Size 0 on a 200 = empty page, skip it. Non-zero size = has content, visit it.

> [!tip] Server header → guess stack first (Apache → `.php`, IIS → `.asp`/`.aspx`), then confirm with extension fuzz before page fuzzing — cuts unnecessary runs