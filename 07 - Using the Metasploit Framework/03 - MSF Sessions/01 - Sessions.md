# MSF — Sessions & Jobs

Background a session without killing it — CTRL+Z or from Meterpreter:

```
meterpreter > background
```

List all active sessions:

```
msf6 > sessions
```

Interact with a specific session by ID:

```
msf6 > sessions -i <ID>
```

Run a post module against a backgrounded session — set SESSION option after selecting the module:

```
msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(...)> set SESSION <ID>
msf6 post(...)> run
```

Run an exploit as a background job — frees the console immediately, keeps the listener alive:

```
msf6 exploit(multi/handler) > exploit -j
```

Manage jobs — killing a session with CTRL+C leaves the port bound, killing the job frees it:

```
msf6 > jobs -l
msf6 > jobs -k <ID>
msf6 > jobs -K
```

> [!warning] CTRL+C on an active exploit does NOT free the port — the process lingers. Use `jobs -k <ID>` to actually release it before running another module on the same port.

> [!tip] `exploit -j` on multi/handler lets you catch shells passively while doing other enum in the same msfconsole — standard practice when waiting on a payload to execute.