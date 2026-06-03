# Automatic Modification

Set-and-forget rules that fire on every request/response — no manual interception needed per cycle.

Burp → `Proxy > Proxy settings > HTTP match and replace rules > Add`. ZAP → `Ctrl+R` (Replacer) → Add.

Auto-replace User-Agent on every request (Burp):

Type: `Request header` / Match: `^User-Agent.*$` (regex on) / Replace: `User-Agent: HackTheBox Agent 1.0`

Auto-fix input field restrictions on every response so edits persist across refreshes (Burp):

Type: `Response body` / Match: `type="number"` / Replace: `type="text"` (add second rule: `maxlength="3"` → `maxlength="100"`)

ZAP Replacer equivalent fields: Match Type → `Request Header (will add if not present)`, Match String → header name, Replacement String → new value. Initiators defaults to all HTTP(S) — leave it.

> [!tip] Use auto response modification instead of per-request interception when you need the same HTML change every page load — otherwise the fix disappears on refresh.