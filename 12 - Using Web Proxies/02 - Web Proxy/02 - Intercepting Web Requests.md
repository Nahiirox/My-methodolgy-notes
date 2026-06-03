# Intercepting & Manipulating Requests

Burp → `Proxy > Intercept` → intercept is on by default, click the button to toggle. Forward request → `Forward`. Drop it → `Drop`.

ZAP → interception off by default (green button = passing). Toggle with the button or `Ctrl+B`. Forward with the Step button (next to red break button). HUD alternative: left pane, second button from top.

Intercept is the bypass for any client-side validation — JS blocking non-numeric input, length limits, restricted fields. The back-end may not validate at all. Intercept → modify → forward.

Caught a POST, want to test command injection in a param — change the value directly in the request pane before forwarding:

```
ip=1  →  ip=;ls;
```

Common things to test by manipulating intercepted requests: SQLi, CMDi, upload bypass, auth bypass, XSS, XXE, deserialization, error handling.

> [!tip] If ZAP HUD behaves oddly, toggle it off and use the main UI instead — browser version compatibility issues are common.