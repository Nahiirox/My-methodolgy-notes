# Staff — OSINT via LinkedIn & Social Media

Job postings and employee profiles are passive intel — no contact with the target.

---

### Job Postings → Stack Fingerprint

A single job post gives: languages (Java/Python/PHP/Ruby), databases (PostgreSQL/MySQL/Oracle), frameworks (Flask/Django/Spring/ASP.NET), tools (Git/SVN/Atlassian/Docker/K8s), auth patterns (JWT, REST API design). Map these to known vuln classes before touching the app.

Search LinkedIn job postings:

```
site:linkedin.com/jobs "<COMPANY>"
```

---

### Employee Profiles → Direct Intel

Technical employees (dev + security roles) reveal: current project stack, open source repos with hardcoded secrets, personal emails that may match corporate format, JWT tokens or API keys in public code.

Check every GitHub link in a profile — `grep -r "secret\|token\|password\|key"` across their public repos. Hardcoded JWT → decode at jwt.io immediately, check if it's valid against any exposed endpoint.

Security team profiles → tells you what defensive tooling the company uses (EDR, SIEM, WAF vendors). Dev team profiles → framework versions → map to CVEs.

---

### Username Schema

From any confirmed name + email format → build spray list with Username Anarchy or manually:

`flast` / `first.last` / `f.last` / `firstl` — confirm format via OSINT before spraying.

> [!tip] Filter LinkedIn search by "Security Engineer" or "DevOps" at the target company — security roles leak defensive stack, DevOps roles leak infra (cloud provider, CI/CD platform, container orchestration). Both are high-value for planning the approach.