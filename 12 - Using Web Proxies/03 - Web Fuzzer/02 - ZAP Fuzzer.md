# ZAP Fuzzer

No throttle on free version — use this over Burp Intruder for anything larger than a short list.

Right-click request in Proxy History → `Attack > Fuzz` → Fuzzer window opens.

**Locations** → select the word to fuzz in the request pane → `Add`. Green marker confirms placement.

**Payloads** → `Add` → payload types worth knowing: `File` (your own wordlist), `File Fuzzers` (built-in dirbuster/SecLists databases, no external file needed), `Numberzz` (sequential int ranges).

**Processors** → add `URL Encode` to avoid server errors on special chars. Other useful ones: `Prefix/Postfix String` (append/prepend to every payload), `Base64 Encode`, `Script` (custom logic). Click `Generate Preview` to confirm output looks right before running.

**Options** → set concurrent threads to 20 for speed. `Depth first` = exhaust all payloads on one position before moving on (good for password lists per user). `Breadth first` = one payload across all positions before next (good for spraying).

Sort results by Response code after run — 200 = hit. Also watch `Size Resp. Body` for size outliers and `RTT` for time-based blind issues.

> [!tip] ZAP has built-in dirbuster wordlists via File Fuzzers — no need to point to an external file for basic dir fuzzing. More wordlist databases installable from ZAP Marketplace.