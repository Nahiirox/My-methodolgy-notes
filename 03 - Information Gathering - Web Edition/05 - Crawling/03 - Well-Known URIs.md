# Well-Known URIs

Standardized paths under `/.well-known/` — passive recon, single GET per URI, no auth required, often overlooked.

Check them all on any new target:

```bash
curl -s https://<TARGET>/.well-known/security.txt
curl -s https://<TARGET>/.well-known/openid-configuration
curl -s https://<TARGET>/.well-known/assetlinks.json
curl -s https://<TARGET>/.well-known/mta-sts.txt
```

---

### High-Value URIs

`security.txt` → security contact email, disclosure policy — confirms org uses a bug bounty program, leaks contact names/emails.

`openid-configuration` → full OAuth2/OIDC config dump: authorization endpoint, token endpoint, userinfo endpoint, JWKS URI, supported scopes, signing algorithms. Each endpoint is a testable attack surface.

From `openid-configuration` output: `authorization_endpoint` → start OAuth flow testing / `token_endpoint` → token endpoint attacks / `jwks_uri` → pull public keys, check for weak algorithms (RS256 vs HS256 confusion attacks) / `scopes_supported` → map what data is accessible via OIDC.

`assetlinks.json` → digital asset ownership links (Android apps etc.) — reveals associated app IDs and package names.

> [!tip] If `openid-configuration` exists, the site uses OAuth2/OIDC. Pull the full JSON and map every endpoint before testing auth. The `jwks_uri` is particularly useful — fetch it and check key algorithms. HS256 in an RS256-signed system = JWT algorithm confusion attack potential.