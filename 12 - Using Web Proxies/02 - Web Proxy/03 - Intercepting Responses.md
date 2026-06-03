# Intercepting Responses

Intercept responses to modify HTML before the browser renders it — removes client-side restrictions without touching the request.

Burp → `Proxy > Proxy settings > Response interception rules` → enable "Intercept responses". Then forward the request, edit the response that comes back.

ZAP → click `Step` on an intercepted request — it auto-intercepts the response.

Common edits to make in the response before forwarding:

Change `type="number"` to `type="text"` and bump `maxlength` to remove input restrictions:

```html
<input type="text" id="ip" name="ip" maxlength="100" ...>
```

Faster alternatives — no response interception needed:

ZAP HUD → third left-pane button (lightbulb) → instantly enables disabled fields and reveals hidden fields without refresh. Also: Burp `Proxy > Proxy settings > Response modification rules` → "Unhide hidden form fields".

ZAP HUD Comments button → shows HTML comment positions inline in the browser — faster than reading source for recon.

> [!tip] Disabled input field → try ZAP HUD lightbulb first, takes one click. Only drop to response interception if you need custom edits beyond enable/unhide.