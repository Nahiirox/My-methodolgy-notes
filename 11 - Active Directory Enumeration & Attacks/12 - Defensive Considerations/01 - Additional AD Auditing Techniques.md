# Additional AD Auditing Techniques

These are reporting and evidence-gathering tools — run them when you want to give clients a comprehensive picture or make sure your own enum didn't miss anything.

---

## AD Explorer (Sysinternals)

Snapshot AD at a point in time for offline analysis and before/after comparison. Any valid domain user can log in and browse all objects without elevated privileges.

File → Create Snapshot → name it → pull it offline for the reporting phase. Useful for documenting the state of AD at time of assessment.

---

## PingCastle

Domain security risk scoring based on CMMI maturity model. Generates HTML report with misconfig findings, trust maps, delegation issues, vulnerability susceptibility — much faster than manually enumerating everything.

Run interactively:

```cmd
PingCastle.exe
```

Option 1 (healthcheck) → full domain risk report. Option 3 (carto) → trust relationship map across all interconnected domains.

Scanner submenu worth running: `nullsession`, `spooler`, `zerologon`, `smb`, `laps_bitlocker`, `aclcheck`.

Good client deliverable — gives them a numeric risk score they can act on and benchmark against over time.

---

## Group3r

GPO-specific vulnerability finder. Needs a domain-joined host or `runas /netonly` context. Finds misconfigured policy settings, interesting permissions, writable paths — things other tools miss:

```cmd
group3r.exe -f <OUTFILE>.log
```

Read output by indentation level: GPO → policy setting → finding. Each finding box explains the issue and why it matters.

---

## ADRecon

Broad sweep collector — dumps users, groups, SPNs, GPOs, trusts, OUs, DNS, computers, LAPS, BitLocker keys (privileged), printers into CSV + HTML report. Run it when you want confidence nothing was missed:

```powershell
.\ADRecon.ps1
```

Output drops to a timestamped folder. Requires Excel for auto-generated report; otherwise use `-GenExcel` from another host against the CSV output folder. Requires GroupPolicy PS module for GPO report.

> [!tip] PingCastle is the best single-tool summary to hand a client. ADRecon is the "make sure I haven't missed anything" sweep. Group3r fills the GPO gap that BloodHound doesn't cover well. Run all three on assessments where stealth isn't a concern.