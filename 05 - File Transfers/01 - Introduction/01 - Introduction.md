# File Transfers — Intro

Transfer method depends entirely on what's allowed outbound. Always have fallbacks.

Decision chain from the example scenario — try in this order when each gets blocked:

PowerShell download → blocked by AppLocker/WDAC → certutil from GitHub → web filtering blocks it → FTP (port 21) → firewall blocks it → SMB (port 445) → allowed → use impacket smbserver → works.

> [!tip] When one transfer method fails, the question is always "what port/protocol does the firewall allow outbound?" — pick your transfer method to match that, not the other way around.

Host controls to account for: AppLocker / WDAC (blocks PS, specific binaries), AV/EDR (flags known tools), web content filtering (blocks external file hosts). Network controls: firewall egress rules per port, IDS/IPS signatures on common transfer patterns.

SeImpersonatePrivilege → PrintSpoofer/JuicyPotato/RoguePotato → SYSTEM. Transfer the binary first, then execute.