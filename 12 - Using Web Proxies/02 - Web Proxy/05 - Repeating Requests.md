# Repeating Requests

Pull any request from history → modify → resend without re-intercepting.

Burp → `Proxy > HTTP History` → select request → `Ctrl+R` (send to Repeater) → `Ctrl+Shift+R` (jump to Repeater tab) → edit → `Send`. Response appears in the same pane.

ZAP → `History` tab (bottom) → right-click request → `Open/Resend with Request Editor` → edit → `Send`. Or in HUD → click request in History pane → `Replay in Console` (response in HUD) / `Replay in Browser` (rendered).

Both allow switching HTTP method inline — Burp: right-click → `Change Request Method`. ZAP: Method dropdown in the editor.

> [!tip] Burp history shows both the original and the edited request — click the pane header to toggle. Useful for diffing what you changed vs what was sent.