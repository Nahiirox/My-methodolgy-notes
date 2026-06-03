# Extensions

**Burp BApp Store** — `Extensions` tab → `BApp Store` → sort by Popularity. Some extensions are Pro-only.

Worth installing:

`Active Scan++` `Autorize` `Backslash Powered Scanner` `JS Link Finder` `Java Deserialization Scanner` `PHP Object Injection Check` `CSRF Scanner` `Retire.JS` `CSP Auditor` `Software Vulnerability Scanner` `Headers Analyzer` `Error Message Checks`

`Decoder Improved` — drop-in replacement for Burp Decoder, adds MD5/more hash types and extra encoders.

> [!tip] Some extensions need Jython installed first — check requirements before wondering why install fails.

**ZAP Marketplace** — toolbar → `Manage Add-ons` → `Marketplace` tab. Stick to Release builds; Beta/Alpha are unstable.

Install `FuzzDB Files` + `FuzzDB Offensive` — adds FuzzDB wordlists to ZAP Fuzzer's File Fuzzers list. CMDi payloads at `fuzzdb > attack > os-cmd-execution` — useful against WAF-protected targets where generic wordlists get blocked.