````markdown
I will provide you with the **complete content of one CPTS module**.

Transform it into a **concise, practical, methodology-focused penetration-testing reference** in clean Obsidian Markdown.

The notes are for:
- CPTS exam preparation and exam labs
- HTB machines / boxes
- Practical penetration-testing work
- Fast lookup when I already understand the fundamentals

These notes are **not a replacement for the module**. If I need an explanation, background, or deeper understanding, I will return to the original module or research it separately.

The output must answer:

> **What should I look for? What should I run? What should I check? What findings matter? What attack/enumeration workflow does the module provide?**

Do not turn the notes into a textbook.

---

# 1. SOURCE FIDELITY — ABSOLUTE RULE

The provided CPTS module is the **ONLY source**.

Extract, condense, and restructure the module. Do not supplement it with outside knowledge.

NEVER invent:
- Commands
- Flags
- Tools
- Payloads
- Exploits
- Techniques
- Methodology steps
- Attack paths
- Wordlists
- Files
- URLs
- Results
- Credentials
- Explanations not supported by the module

If the module says:

> Enumerate SMB shares.

Do NOT create an `smbclient` command unless the module itself provides one.

If the module provides a command, preserve it.

If the module provides useful methodology only as prose, preserve that methodology in concise form.

### Important distinction

**Explicitly present does NOT mean automatically include.**

The module may contain educational material that is technically correct but useless during quick practical lookup. Remove that material when it is merely:
- introductory
- theoretical
- conceptual
- basic syntax
- protocol background
- generic definitions
- study-oriented discussion

Your job:

> **EXTRACT → FILTER → CONDENSE → STRUCTURE → QUICK REFERENCE**

Not:

> **TEACH → EXPAND → GENERALIZE → ADD KNOWLEDGE**

---

# 2. TARGET OUTPUT STYLE

Write like a **professional pentester's field reference**.

Priorities:

1. Practical attack/enumeration workflow
2. Exact commands provided by the module
3. Important flags/options
4. Dangerous/misconfigured settings
5. Payloads/scripts explicitly provided
6. Enumeration findings and what they reveal
7. Important files, paths, extensions, shares, directories
8. Credentials/usernames/hashes explicitly provided
9. Short context necessary to understand a technique
10. Theory only when it directly affects practical testing

Use **high information density**.

Prefer:

```text
### Enumeration
1. Identify exposed services.
2. Enumerate the service.
3. Check configuration.
4. Test the relevant weakness.
````

over long explanatory paragraphs.

Do not turn every paragraph from the module into a bullet.

---

# 3. PRESERVE THE MODULE STRUCTURE

Preserve the module's **original section names and order**.

Do not:

* Rename sections
* Merge sections
* Reorder sections
* Invent new major sections
* Create summaries at the end

Use:

```markdown
# Module Name
## Section
### Subsection
#### Subsection
```

You may reorganize information **inside an existing section** when it improves lookup speed.

### Important

Do NOT flatten the module into a generic tool cheat sheet.

If the module contains:

```text
# FTP
## Default Configuration
## Dangerous Settings
## FTP Interaction
## Footprinting the Service
```

preserve those sections.

If a section contains only pure theory and no practical value, it may be removed.

---

# 4. REMOVE BASIC / TEXTBOOK MATERIAL

I already understand basic IT, networking, common protocols, and basic tool syntax.

Remove basic explanations such as:

* "FTP is a file transfer protocol."
* "SMB is used for file sharing."
* "Port 445 is SMB."
* "FTP `get` downloads a file."
* "FTP `put` uploads a file."
* "DNS A records contain IP addresses."
* "SMTP is used for email."
* Generic explanations of common commands.

Do NOT include standard port mappings unless the module uses the port in a command or the port itself is operationally relevant.

Do NOT reproduce complete command reference lists merely because the module contains them.

For example, if the module lists:

```text
get = download
put = upload
ls = list
```

remove the definitions.

However, if a command is used as part of an actual attack/enumeration workflow, preserve the command.

---

# 5. DO NOT DELETE PRACTICAL METHODOLOGY

This is critical.

Do NOT assume:

> "No command = useless."

If the module explains a practical methodology, decision point, observation, or attack logic entirely in prose, preserve it.

Examples worth keeping:

* What to investigate first
* What a finding can reveal
* What to investigate after discovering something
* How one finding leads to another
* What configuration indicates a weakness
* What a particular result means operationally
* What attack path the module demonstrates
* What information should be collected before proceeding

Condense it aggressively, but preserve the practical meaning.

Example:

```text
- Enumerate subdomains → resolve hosts → identify directly hosted infrastructure → inspect exposed services.
```

Do not add steps that the module never states.

---

# 6. DO NOT CREATE AI-GENERATED "PRINCIPLES"

Do NOT invent sections such as:

* Core Principles
* Key Takeaways
* Best Practices
* Pro Tips
* Exam Tips
* Common Mistakes
* Why This Matters
* Summary
* Important Concepts
* Methodology
* Attack Strategy
* Lessons Learned

unless the **module itself contains that section or information**.

Do not transform ordinary content into artificial "AI methodology."

If the module says:

> Enumeration is iterative.

You may preserve:

```text
- Enumeration is iterative: findings guide subsequent enumeration.
```

But do not create a new "Core Enumeration Principles" section unless the module itself has one.

---

# 7. COMMANDS — PRESERVE, DO NOT INVENT

Include relevant commands explicitly present in the module.

Preserve:

* Tool
* Syntax
* Flags
* Arguments
* Options
* Script names
* Payloads
* Query syntax
* Important examples

Do NOT generate commands from prose.

If the module says:

> Enumerate DNS records.

and provides no command, do not invent `dig`.

If it provides:

```bash
dig any <TARGET_DOMAIN>
```

preserve it.

### Command grouping

Group related commands into logical blocks when useful.

Avoid splitting every single command into a separate code block unless it improves lookup.

---

# 8. PLACEHOLDERS

Replace lab-specific values with:

* IP → `<TARGET_IP>`
* Domain → `<TARGET_DOMAIN>`
* URL → `<TARGET_URL>`
* Username → `<USERNAME>`
* Password → `<PASSWORD>`
* Share → `<SHARE>`
* SID → `<SID>`
* Database → `<DATABASE>`
* Table → `<TABLE>`

Only use placeholders when the original value is actually lab-specific.

Do NOT replace generic examples that are technically meaningful.

Do NOT create separate "Targets", "Credentials", or "Variables" sections.

Placeholders belong inside commands/examples or directly relevant technical content.

---

# 9. PATH / WORDLIST NORMALIZATION

This rule is extremely important.

Remove environment-specific installation paths from commands **without destroying technically meaningful paths**.

### Wordlists

Always strip filesystem paths from wordlists.

Convert:

```bash
wpscan --passwords /usr/share/wordlists/rockyou.txt
```

to:

```bash
wpscan --passwords rockyou.txt
```

Convert:

```bash
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt
```

to:

```bash
onesixtyone -c snmp.txt
```

Preserve:

* filename
* extension
* capitalization

### Installed tools/scripts

Remove unnecessary filesystem prefixes.

Convert:

```bash
/usr/share/doc/impacket/examples/GetNPUsers.py
```

to:

```bash
GetNPUsers.py
```

Convert:

```bash
./odat.py all -s <TARGET_IP>
```

to:

```bash
odat.py all -s <TARGET_IP>
```

Convert:

```bash
./ssh-audit.py <TARGET_IP>
```

to:

```bash
ssh-audit.py <TARGET_IP>
```

### Technically meaningful paths

DO NOT strip paths such as:

```text
/etc/exports
/etc/passwd
/etc/ssh/sshd_config
/etc/bind/db.domain.com
/etc/vsftpd.conf
```

These are meaningful configuration/system paths and must remain.

### Critical rule

Never blindly remove `/` or filesystem paths.

Ask:

> Is this path merely showing where a tool/wordlist is installed, or is the path itself technically relevant?

Strip the former.

Preserve the latter.

---

# 10. CODE BLOCKS ARE NOT IMMUTABLE

Actively normalize commands **inside code blocks**.

Do not preserve an original command unchanged merely because it is inside a fenced block.

Apply:

* Placeholder replacement
* Wordlist path stripping
* Tool/script path stripping
* Lab-specific value normalization

while preserving the command's actual syntax.

Do not otherwise modify or "improve" commands.

---

# 11. PAYLOADS / SCRIPTS / CODE

Preserve every technically relevant payload, script, query, or code example explicitly provided by the module.

Do not:

* Optimize
* Correct
* Improve
* Replace
* Add alternatives
* Rewrite the technique

Only apply:

* Required placeholders
* Required path normalization

Preserve meaningful differences between examples.

---

# 12. CONFIGURATION / DANGEROUS SETTINGS

Keep security-relevant configuration settings when the module presents them as:

* dangerous
* weak
* insecure
* exploitable
* useful for enumeration
* relevant to attack paths

Do NOT add your own interpretation.

Example:

```text
### Dangerous Settings

- `anonymous_enable=YES`
- `anon_upload_enable=YES`
- `no_root_squash`
```

Do not turn these into long explanations.

If the module explicitly explains the practical consequence, preserve that consequence concisely.

---

# 13. ENUMERATION / ATTACK WORKFLOWS

When the module gives a workflow, preserve it as a concise actionable sequence.

Example:

```markdown
### Enumeration

1. Identify exposed services.
2. Fingerprint the service.
3. Enumerate configuration.
4. Check for dangerous settings.
5. Enumerate users/shares/resources.
6. Follow relevant findings.
```

Only use steps actually supported by the module.

Do not add generic pentesting steps.

Do not turn a workflow into a command list if the module expresses important logic in prose.

---

# 14. FINDINGS / RESULT INTERPRETATION

Keep information about **what a finding reveals** when it is explicitly stated in the module.

Examples:

```text
- TXT record → may reveal third-party providers.
- `allow-transfer` → controls hosts permitted to receive zone transfers.
- `no_root_squash` → root-created files retain UID/GID 0.
```

Keep the practical implication.

Remove lengthy background explanations.

Do NOT invent implications from your own knowledge.

---

# 15. WORDLISTS

Include wordlists when they are used by the module.

Use **filename only**.

Example:

```text
subdomains-top1million-110000.txt
snmp.txt
```

Never preserve:

```text
/usr/share/wordlists/...
/opt/useful/seclists/...
```

Do not replace the filename with a generic name.

---

# 16. URLS / EXTERNAL RESOURCES

Keep URLs only when they are technically relevant to:

* exploitation
* enumeration
* reproduction
* a technique
* a useful resource explicitly used by the module

Do not copy every external link from the module.

Never fabricate URLs.

Apply target placeholders where appropriate.

---

# 17. CREDENTIALS / SECRETS

Preserve credentials explicitly provided by the module when they are technically relevant.

Examples:

* Default credentials
* Default passwords
* Usernames
* Community strings
* Example credentials used by the technique

Do not invent credentials.

Do not create a separate credentials section unless the module itself has one.

---

# 18. FILES / PATHS / EXTENSIONS

Extract technically useful:

* Configuration files
* Interesting files
* Registry paths
* Shares
* Directories
* Extensions
* Web paths
* Service files
* Zone files

Preserve exact technically meaningful paths.

Do not invent paths.

---

# 19. DEFAULT CONFIGURATION SECTIONS

If the module explicitly provides default configuration files or paths, preserve them.

However:

Do NOT invent commands such as:

```bash
cat /etc/example.conf
```

unless the module itself provides that command.

A configuration file mentioned in prose should remain a file/path reference, not become a generated command.

---

# 20. PORTS

Do not create generic port-reference material.

Keep ports only when:

* used directly in a command
* required to understand a specific service interaction
* explicitly relevant to the technique
* the module itself emphasizes the port

Do not add:

```text
SMB → 445
FTP → 21
SSH → 22
```

unless the module uses that information operationally.

---

# 21. TOOL NAMES

Preserve tools explicitly used in the module.

Do not add alternatives.

If the module uses:

```text
rpcclient
smbclient
smbmap
CrackMapExec
enum4linux-ng
```

keep them.

Do not add tools such as `netexec`, `nxc`, `rustscan`, `ffuf`, etc. unless they appear in the source module.

---

# 22. DUPLICATION

Remove **unnecessary duplication** while preserving meaningful differences.

If the same command appears repeatedly with no meaningful difference, consolidate it.

If later examples introduce:

* different flags
* different targets
* different payloads
* different options
* different attack contexts

preserve them.

Do not aggressively deduplicate the point where useful examples disappear.

---

# 23. FORMATTING

Output must be clean Obsidian Markdown.

Use:

* `#`, `##`, `###`, `####`
* Bullets
* Numbered workflows
* Inline code
* Fenced code blocks

Use appropriate code block languages:

```text
bash
powershell
python
sql
php
text
```

when applicable.

Do NOT use:

* Tables
* Emojis
* Decorative symbols
* Generated TOCs
* Long introductions
* Repetitive prose
* Artificial summaries
* Excessive bolding

Commands must never be placed inside tables.

---

# 24. IMPORTANT: DO NOT OVER-CONDENSE

The goal is **quick reference**, not maximum deletion.

Keep something if it helps me during an actual CPTS exam/HTB box to:

* know what to enumerate
* know what command to run
* recognize an interesting configuration
* recognize an interesting result
* know what to investigate next
* understand an attack path
* reproduce a technique from the module
* identify a useful file/resource/credential
* understand a practical decision point

Remove something if it only teaches:

* basic terminology
* protocol theory
* generic syntax
* obvious command meanings
* introductory background
* repeated information
* study-oriented filler

When uncertain between **keeping a practically useful detail** and deleting it for brevity, keep it.

---

# 25. SOURCE-ORDER RULE

Preserve the source module's information flow.

Do not reorganize everything according to your own preferred pentesting methodology.

The notes should remain recognizable as a compressed version of the original module.

You are allowed to improve **local organization inside a section**, but not rewrite the module into a different curriculum.

---

# 26. INTERNAL VALIDATION

Before outputting, silently verify:

### Source fidelity

* Every technical fact comes from the module.
* No outside cybersecurity knowledge was added.
* No commands were invented from prose.
* No tools were invented.
* No payloads were invented.
* No flags were invented.
* No attack paths were invented.

### Practicality

* Practical methodology was preserved.
* Decision points were preserved.
* Useful findings and interpretations were preserved.
* Relevant commands were preserved.
* Relevant configuration weaknesses were preserved.
* Relevant payloads/scripts were preserved.

### Filtering

* Basic definitions removed.
* Generic protocol theory removed.
* Basic command explanations removed.
* Standard port mappings removed unless operationally relevant.
* Artificial AI-generated principles removed.
* Study-oriented filler removed.

### Normalization

* Lab IPs/domains/credentials normalized.
* Wordlist filesystem paths removed.
* Tool installation paths removed.
* Meaningful system/configuration paths preserved.
* Filenames and extensions preserved.

### Formatting

* Original section names/order preserved.
* No invented major sections.
* No tables.
* No TOC.
* No summary.
* No fluff.
* Commands are easy to locate.
* Output is valid Obsidian Markdown.

---

# FINAL INSTRUCTION

Output **ONLY the finished Obsidian Markdown notes**.

Do not explain your decisions.

Do not describe what you removed.

Do not provide a summary.

Do not mention this prompt.

Do not add information from outside the module.

Do not make the notes artificially shorter just to reduce length.

The final result should feel like:

> **The CPTS module compressed into a professional, practical field reference — retaining the module's real methodology, attack logic, commands, findings, and useful technical details while removing textbook material and AI-generated filler.**

**Read → filter → preserve practical methodology → preserve exact technical content → normalize → condense → structure → output.**

```
```
