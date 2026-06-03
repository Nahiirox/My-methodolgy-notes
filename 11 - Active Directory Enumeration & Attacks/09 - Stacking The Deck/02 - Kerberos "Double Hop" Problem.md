# Kerberos Double Hop Problem

WinRM authenticates with Kerberos TGS (service ticket) — not password, not TGT. The TGS grants access to hop 1 only. When you try to reach hop 2 (DC, file share, another host), no TGT is in the session to prove identity → access denied. Password-based auth (PSExec, SMB) stores NTLM hash in memory, which travels forward — Kerberos doesn't.

Diagnose it: `klist` inside a WinRM session shows only one ticket for the current host's HTTP service. RDP session shows TGT + multiple TGS tickets → no double hop problem there.

---

## Workaround 1 — PSCredential Object (evil-winrm / any WinRM)

Pass `-credential $Cred` explicitly to every PowerView call that needs DC access:

```powershell
$SecPassword = ConvertTo-SecureString '<PASS>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $SecPassword)
Get-DomainUser -spn -credential $Cred | select samaccountname
```

Works anywhere. Tedious but reliable. Must specify `-credential` on every AD query.

---

## Workaround 2 — Register PSSession Configuration (Windows only, requires GUI/RDP)

Register a named session that impersonates the user at the local machine level — TGT gets cached properly, no per-command creds needed:

```powershell
Register-PSSessionConfiguration -Name <SESSNAME> -RunAsCredential <DOMAIN>\<USER>
Restart-Service WinRM
```

Then connect using the named config:

```powershell
Enter-PSSession -ComputerName <TARGET> -Credential <DOMAIN>\<USER> -ConfigurationName <SESSNAME>
```

Verify fix — `klist` should now show `krbtgt` TGT, not just the HTTP service ticket.

> [!warning] `Register-PSSessionConfiguration` does NOT work from evil-winrm — requires elevated PowerShell with GUI credential prompt. Only use this from RDP or a domain-joined Windows attack host. Also does not work reliably from PS on Linux.

> [!tip] If on RDP already, no workaround needed — password is cached, TGT is present, double hop doesn't apply. Use RDP as your jump host whenever possible to avoid this entirely.