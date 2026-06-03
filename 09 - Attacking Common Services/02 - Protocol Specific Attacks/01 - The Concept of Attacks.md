# The Concept of Attacks — Framework

Mental model for analyzing any vulnerability: Source → Process → Privileges → Destination. Apply this to understand existing exploits, analyze source code, and debug your own payloads.

**Source** — where the malicious input enters (user input, HTTP header, API call, config, library)

**Process** — what the application does with it (misinterpretation, parsing, logging, execution)

**Privileges** — what context the process runs in (SYSTEM/root, service account, user) — this determines blast radius

**Destination** — where the result goes (local file write, local execution, network callback, RCE)

---

## Log4j (CVE-2021-44228) as the Reference Example

Source: attacker injects JNDI lookup into User-Agent header

Process: Log4j misinterprets the string as a command instead of logging it as text

Privileges: logging runs as admin → execution inherits admin context

Destination: JNDI lookup reaches attacker-controlled LDAP/RMI server → fetches malicious Java class → second cycle begins

Second cycle — Source: malicious Java class / Process: class executes / Privileges: still admin / Destination: reverse connection back to attacker = RCE

---

## Using This During Assessments

When researching a service exploit: map each step to Source/Process/Privileges/Destination before running it. Tells you what input to manipulate, what will happen, what rights you'll get, and where to point your listener.

When a payload fails: check which step broke — wrong input format (Source), parsing not triggering (Process), insufficient rights (Privileges), wrong callback address (Destination).

When reviewing source code for vulns: trace user-controlled input through the call chain and check what privileges each processing function runs under.