# Directory Fuzzing (ffuf)

Key flags: `-w` (wordlist, append `:KEYWORD` to name it), `-u` (target URL, place `FUZZ` where you want to fuzz), `-t` (threads, default 40), `-mc` (match status codes), `-ms` (match response size), `-fc` (filter status codes), `-fs` (filter response size), `-recursion` + `-recursion-depth`, `-o` (output file).

Basic dir fuzz:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/FUZZ
```

Default matcher catches `200,204,301,302,307,401,403` — a 301 with no content still confirms the dir exists and is accessible.

Filter noise by response size when everything comes back with the same code:

```bash
ffuf -w <WORDLIST>:FUZZ -u http://<TARGET>:<PORT>/FUZZ -fs <SIZE>
```

> [!warning] `-t 200` can DoS the target and kill your own connection — default 40 threads is fine, don't crank it on remote hosts.