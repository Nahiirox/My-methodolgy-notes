# Burp Intruder

Free version = 1 req/sec, throttled hard. Use for short targeted fuzzes only — for large wordlists use ffuf/ZAP instead. Pro = unlimited speed.

Send request to Intruder: `Ctrl+I` from anywhere, or right-click in Proxy History → `Send to Intruder`. Jump to Intruder tab: `Ctrl+Shift+I`.

**Positions** → wrap the fuzz target with `§` or select text → `Add §`. Attack type `Sniper` = one payload position, one wordlist iterating through it.

**Payloads** — four things to set:

Payload type `Simple List` → `Load` → select wordlist. Use `Runtime file` instead if wordlist is huge (avoids memory bloat).

Payload Processing → `Add` → `Skip if matches regex` → `^\..*$` skips dot-prefixed lines before they're sent.

Payload Encoding → URL-encode on by default, leave it.

**Settings** → `Grep - Match` → clear defaults → add `200 OK` → disable "Exclude HTTP Headers" → sorts hits to top of results. Sort results column by `200 OK` or `Length` to find real hits fast.

> [!warning] Community Intruder is too slow for real dir/param brute — use it for small targeted attacks (password spray against OWA/VPN/RDS/Citrix, short param lists). Anything large → ffuf or ZAP fuzzer.