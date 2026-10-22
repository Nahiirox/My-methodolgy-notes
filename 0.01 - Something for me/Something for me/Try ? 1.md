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

The provided CPTS module is the **ONLY source**. Extract, condense, and restructure it. Do not supplement it with outside knowledge.

NEVER invent:
- Commands, flags, tools
- Payloads, exploits, techniques
- Methodology steps, attack paths
- Wordlists, files, URLs
- Results, credentials
- Explanations not supported by the module

If the module says:

> Enumerate SMB shares.

do NOT create an `smbclient` command unless the module itself provides one. If the module provides a command, preserve it. If the module provides useful methodology only as prose, preserve that methodology in concise form.

### Explicitly present ≠ automatically include

The module may contain correct but practically useless material. Remove it when it is merely introductory, theoretical, conceptual, basic syntax, protocol background, generic definitions, or study-oriented discussion.

> **EXTRACT → FILTER → CONDENSE → STRUCTURE → QUICK REFERENCE**
> Not: **TEACH → EXPAND → GENERALIZE → ADD KNOWLEDGE**

---

# 2. WHAT TO KEEP, WHAT TO CUT

Keep anything that helps during an actual CPTS exam / HTB box:
- knowing what to enumerate and what command to run
- recognizing an interesting configuration or result
- knowing what to investigate next; how one finding leads to another
- attack paths and practical decision points
- reproducing a technique from the module
- useful files, resources, credentials

Remove anything that only teaches:
- basic terminology, protocol theory, generic syntax
- obvious command meanings (`get` downloads, `put` uploads)
- introductory background, repeated information, study filler

Specific cuts:
- **Port mappings** (`SMB → 445`, `FTP → 21`) unless the module uses the port operationally.
- **Complete command-reference lists** — but keep any command that appears inside an actual attack/enumeration workflow.
- **Module-end Summary sections** and **Skills Assessment / exercise / question sections** — drop them unless they contain commands or facts that appear nowhere else.
- **Screenshots, diagrams, figures** — drop. If a figure's command or output matters and exists as text, keep the text.
- **Module tables** — convert to plain bullets or lines (tables are banned in output, see Section 9).

**When uncertain between keeping a practically useful detail and deleting it for brevity — keep it.**

No fixed length target. Good notes usually land around 10–20% of the module length; go longer whenever commands, payloads, or configuration justify it. Never cut practical content to hit a length.

### Prose methodology is not useless

Do NOT assume "no command = useless". If the module explains methodology, decision points, observations, or attack logic entirely in prose, preserve it — condensed aggressively, practical meaning intact:

```text
- Enumerate subdomains → resolve hosts → identify directly hosted infrastructure → inspect exposed services.
```

When the module gives a workflow, preserve it as a concise numbered sequence. Only steps the module actually supports — no generic pentesting steps, no steps invented to fill gaps. Don't convert a workflow into a bare command list when the module expresses its logic in prose.

---

# 3. STRUCTURE — PRESERVE THE MODULE

Preserve the module's original section names and order. The notes must remain recognizable as a compressed version of the original module.

Do not:
- rename sections
- merge sections
- reorder sections
- invent new major sections
- create summaries at the end

```markdown
# Module Name
## Section
### Subsection
#### Subsection
```

You may reorganize information freely **inside** an existing section when it improves lookup speed — but don't rewrite the module into a different curriculum.

Do NOT flatten the module into a generic tool cheat sheet. If the module contains:

```text
# FTP
## Default Configuration
## Dangerous Settings
## FTP Interaction
## Footprinting the Service
```

preserve those sections. A section containing only pure theory with no practical value may be removed entirely.

---

# 4. NO TEMPLATE-GENERATED FILLER SECTIONS

Do NOT invent sections such as:

*Core Principles, Key Takeaways, Best Practices, Pro Tips, Exam Tips, Common Mistakes, Why This Matters, Summary, Important Concepts, Methodology, Attack Strategy, Lessons Learned*

unless the **module itself** contains that section or information.

If the module says "Enumeration is iterative", you may keep:

```text
- Enumeration is iterative: findings guide subsequent enumeration.
```

but you do not create a "Core Enumeration Principles" section.

---

# 5. COMMANDS, PAYLOADS, NORMALIZATION

### Preserve, do not invent

Include commands explicitly present in the module. Preserve tool, syntax, flags, arguments, options, script names, payloads, query syntax, and important examples.

Do NOT generate commands from prose:
- "Enumerate DNS records" with no command → no `dig`.
- `dig any <TARGET_DOMAIN>` provided → keep it exactly.

Group related commands into logical blocks. Don't split every command into its own code block unless that improves lookup.

Short inline `#` comments for context are fine (own-notes style). Never alter the command syntax itself.

### Payloads / scripts / code

Preserve every technically relevant payload, script, query, or code example the module provides. Do not optimize, correct, improve, replace, rewrite, or add alternatives. Only apply placeholders and path normalization. Preserve meaningful differences between examples.

### Code blocks are not immutable

Actively normalize commands **inside** code blocks — placeholders, wordlist/tool path stripping, lab-value normalization. A command isn't sacred just because it's fenced. Beyond that, don't modify or "improve" it.

### Placeholders

Replace lab-specific values:
- IP → `<TARGET_IP>`
- Domain → `<TARGET_DOMAIN>`
- URL → `<TARGET_URL>`
- Username / password → `<USERNAME>` / `<PASSWORD>`
- Share → `<SHARE>`
- SID → `<SID>`
- Database / table → `<DATABASE>` / `<TABLE>`

Only when the original value is actually lab-specific; don't replace generic examples that are technically meaningful. Don't create separate "Targets" / "Credentials" / "Variables" sections — placeholders live inside the content they belong to.

### Path / wordlist normalization — critical

Strip environment-specific install paths without destroying technically meaningful paths.

**Wordlists — filename only:**

```bash
wpscan --passwords rockyou.txt        # was: /usr/share/wordlists/rockyou.txt
onesixtyone -c snmp.txt               # was: /opt/useful/seclists/Discovery/SNMP/snmp.txt
```

Preserve filename, extension, capitalization. Never replace the filename with a generic name.

**Installed tools/scripts — drop filesystem prefixes and `./`:**

```bash
GetNPUsers.py                          # was: /usr/share/doc/impacket/examples/GetNPUsers.py
odat.py all -s <TARGET_IP>
ssh-audit.py <TARGET_IP>
```

**System/config paths — NEVER strip:**

```text
/etc/exports  /etc/passwd  /etc/ssh/sshd_config  /etc/bind/db.domain.com  /etc/vsftpd.conf
```

The test: *is this path merely where a tool/wordlist is installed, or is the path itself technically relevant?* Strip the former, keep the latter.

### Tools

Keep the tools the module uses (`rpcclient`, `smbclient`, `smbmap`, CrackMapExec, `enum4linux-ng`...). Never add alternatives (`netexec`, `nxc`, `rustscan`, `ffuf`...) unless they appear in the module.

### Duplication

Consolidate exact repeats. Keep examples that differ in flags, targets, payloads, options, or attack context. Don't dedupe so aggressively that useful examples disappear.

---

# 6. TECHNICAL CONTENT TYPES

### Configuration / dangerous settings

Keep security-relevant settings the module presents as dangerous, weak, insecure, exploitable, or enumeration-useful:

```text
### Dangerous Settings

- `anonymous_enable=YES`
- `anon_upload_enable=YES`
```

If the module states the practical consequence, keep it in one short line (`no_root_squash` → root-created files keep UID/GID 0). Don't add your own interpretation, don't write long explanations.

A config file mentioned in prose stays a path reference — don't generate `cat /etc/example.conf` unless the module provides it.

### Findings / result interpretation

Keep what a finding reveals when the module states it:

```text
- TXT record → may reveal third-party providers.
- `allow-transfer` → hosts permitted to receive zone transfers.
```

Keep the practical implication, drop lengthy background. Never invent implications from your own knowledge.

### Credentials / secrets

Preserve default credentials, default passwords, usernames, community strings, and example credentials the module explicitly provides. Never invent credentials. No separate credentials section unless the module itself has one.

### Files / paths / extensions

Extract useful config files, interesting files, registry paths, shares, directories, extensions, web paths, service files, zone files — with exact technically meaningful paths. Never invent paths.

### URLs

Keep only URLs relevant to exploitation, enumeration, reproduction, a technique, or a resource the module explicitly uses. Never fabricate URLs, don't copy every external link. Apply placeholders where appropriate.

---

# 7. VOICE — MY FIELD NOTES, NOT TEMPLATE OUTPUT

As important as source fidelity. Uniform, symmetrical, template-perfect notes read as template-generated even when the content is correct. Actively avoid that.

**Scope:** variation applies to how content is arranged *inside* each section — prose vs bullets vs code blocks, fragments vs sentences. The module's own section headings and their order are always preserved exactly (Section 3).

- Vary the internal arrangement per section. A short paragraph followed by one command block is fine; not everything must be bullets.
- Break up bullet monocultures — where the logic reads naturally as a sentence or two, write the sentence.
- Occasional `→` shorthand is fine (`null session → full user list`). Don't make it the default connector on every line; mix in plain phrasing ("this leaks the OS version", "means root can write here").
- Don't bold every keyword. Reserve bold for settings/flags that actually need to jump out.
- Terse, clipped phrasing is welcome ("check reuse", "same as above but SSH"). Not every line needs to be a complete, grammatically tidy sentence.
- Don't restate a section's heading inside its content.
- Vary sentence rhythm — short fragments next to longer lines, not a uniform uniform cadence of similarly-sized bullets.
- No filler connective tissue ("Also...", "..."). State the fact.

The test: it should look like something *I* typed while going through the module and taking my own shortcuts — not a document generated to a template.

---

# 8. WORKED EXAMPLE

*Illustrative only — never copy content from this example into the output; always use the actual module.*

Module excerpt (input):

```text
FTP is one of the oldest protocols still in use, operating on a
client-server model over TCP with separate command and data channels.
[... more theory ...]

## FTP Interaction

  ls      - list directory contents
  get     - download a file
  put     - upload a file
  status  - show session status

## Dangerous Settings

  anonymous_enable     allow anonymous logins
  anon_upload_enable   allow anonymous uploads

If anonymous_enable is set to YES and anon_upload_enable to YES, and the
anonymous user has write access to a directory that is also served by a
web server, we can upload a malicious file there and reach code execution.

## Footprinting the Service

nmap -sV -p21 -sC 10.129.14.128

Anonymous login can be tested against a non-standard port as well:
ftp anonymous@10.129.14.128 -p 2121

Brute-forcing:
hydra -l ftp -P /usr/share/wordlists/rockyou.txt ftp://10.129.14.128
```

Desired output:

````markdown
# FTP

## Dangerous Settings

- `anonymous_enable=YES`
- `anon_upload_enable=YES`

anon write access + a directory also served by a web server → upload a payload there, code execution.

## Footprinting the Service

```bash
nmap -sV -p21 -sC <TARGET_IP>

# anon login test — module example ran on non-standard port 2121
ftp anonymous@<TARGET_IP> -p 2121

hydra -l ftp -P rockyou.txt ftp://<TARGET_IP>
```
````

What this demonstrates: theory dropped; "FTP Interaction" dropped entirely (pure command reference); settings table converted to bullets; prose consequence condensed to one line; lab IPs → placeholders; wordlist path stripped, filename kept; port kept because operationally used; one short comment preserves the module's context; a single `→`, not arrow-patterned lines throughout.

---

# 9. FORMATTING

Clean Obsidian Markdown.

Use: `#` / `##` / `###` / `####` headings, bullets, numbered workflows, inline code, fenced code blocks with language tags (`bash`, `powershell`, `python`, `sql`, `php`, `text`) when applicable.

Do NOT use:
- Tables — commands must never appear inside tables
- Emojis or decorative symbols
- Generated TOCs
- Long introductions, repetitive prose, artificial summaries
- Excessive bolding

Formatting varies naturally between sections per Section 7 — no forced identical structure, bolding pattern, or bullet density everywhere.

---

# FINAL CHECK

Before outputting, verify:

- **Fidelity** — every fact, command, tool, flag, payload, URL, and credential comes from the module; nothing invented, nothing supplemented.
- **Practicality** — methodology, decision points, findings/interpretations, commands, config weaknesses, payloads all preserved.
- **Filtering** — basic definitions, protocol theory, generic syntax, port mappings, filler all removed.
- **Normalization** — lab values replaced; wordlist/tool install paths stripped; system/config paths intact; filenames and extensions preserved.
- **Structure** — original section names/order kept; no invented sections, tables, TOC, or summary.
- **Voice** — internal arrangement varies naturally between sections; reads like field notes, not a template.

---

# FINAL INSTRUCTION

Output **ONLY the finished Obsidian Markdown notes**.

Do not explain your decisions. Do not describe what you removed. Do not provide a summary. Do not mention this prompt. Do not add information from outside the module. Do not make the notes artificially shorter just to reduce length.

The final result should feel like:

> **My own compressed field notes from working through the CPTS module — retaining the real methodology, attack logic, commands, findings, and useful technical details while removing textbook material, template-generated filler, and template-uniform formatting.**

**Read → filter → preserve practical methodology → preserve exact technical content → normalize → condense → structure with natural variation → output.**