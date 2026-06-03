# Web Proxy Setup — Browser & Certs

Both Burp and ZAP default to `127.0.0.1:8080`. Quickest start — use the built-in pre-configured browser: Burp → `Proxy > Intercept > Open Browser`, ZAP → Firefox icon in top bar.

For real Firefox, use FoxyProxy: add proxy → `127.0.0.1`, port `8080`, save, select it from the icon. Change port if 8080 is taken: Burp → `Proxy > Proxy settings > Proxy listeners`, ZAP → `Tools > Options > Network > Local Servers/Proxies`.

Install CA cert or HTTPS traffic breaks silently:

Burp → with Burp selected in FoxyProxy, browse to `http://burp` → download CA Certificate → Firefox `about:preferences#privacy` → View Certificates → Authorities → Import → trust both checkboxes.

ZAP → `Tools > Options > Network > Server Certificates` → Save → same Firefox import steps.

> [!warning] Skipping CA cert install = HTTPS requests either fail or pop a cert warning every time — do this before touching any HTTPS target.