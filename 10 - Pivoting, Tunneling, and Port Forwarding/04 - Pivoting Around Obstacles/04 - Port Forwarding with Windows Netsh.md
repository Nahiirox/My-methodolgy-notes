# Port Forwarding with Windows Netsh

Built-in Windows tool — no binaries to transfer, no AV risk. Requires admin privileges.

Forward traffic arriving on a Windows host's port to an internal target:

```cmd
netsh.exe interface portproxy add v4tov4 listenport=<LPORT> listenaddress=<WINDOWS_HOST_IP> connectport=<TARGET_PORT> connectaddress=<TARGET_IP>
```

Verify the rule was created:

```cmd
netsh.exe interface portproxy show v4tov4
```

Connect from your attack host to the Windows pivot's listening port — netsh handles the redirect internally:

```bash
xfreerdp /v:<WINDOWS_HOST_IP>:8080 /u:<USER> /p:<PASS>
```

Clean up after — rules persist across reboots:

```cmd
netsh.exe interface portproxy delete v4tov4 listenport=<LPORT> listenaddress=<WINDOWS_HOST_IP>
```

> [!tip] netsh portproxy is LOTL — pre-installed on all Windows, no file drops, blends into admin activity. Combine with `netsh advfirewall` if the port isn't already allowed through the Windows firewall: `netsh advfirewall firewall add rule name="pivot" protocol=TCP dir=in localport=8080 action=allow`

> [!warning] Rules created by `portproxy` persist after reboot. Always clean up — delete the rule and the firewall exception before ending the assessment.