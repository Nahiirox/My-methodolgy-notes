# Intro to Databases

Background-only section. No commands, no attack surface yet.

Architecture that matters: client (Tier I) → app server/middleware (Tier II) → DBMS (Tier III). SQLi targets the Tier II → Tier III boundary — malicious input from Tier I passes through Tier II and reaches the DBMS as a valid query.

App server and DBMS can be co-hosted or split. Co-hosted = compromise the app → direct local DB access. Separate = need credentials or a second hop.