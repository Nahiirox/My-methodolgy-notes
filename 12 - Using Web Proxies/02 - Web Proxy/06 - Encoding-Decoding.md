# Encoding/Decoding

URL-encode before sending — spaces, `&`, and `#` break requests if left raw.

Burp Repeater → select text → `Ctrl+U` (URL-encode key chars). Or right-click → `Convert Selection > URL > URL-encode key characters`. Enable "encode as you type" for automatic. ZAP auto-encodes in the background.

Full encoder access: Burp → `Decoder` tab. ZAP → `Ctrl+E`.

Both support: URL, HTML, Unicode, Base64, ASCII hex — encode and decode in either direction.

Spot a base64 cookie → Burp Decoder → paste → `Decode as > Base64` → read plaintext → modify → re-encode → drop back into request in Repeater. Same flow in ZAP `Ctrl+E` Encode/Decode/Hash tab.

Burp Inspector (available inside Proxy and Repeater panes) does inline encode/decode without switching tabs.

> [!tip] Encoded cookie with `is_admin:false` or similar → decode → flip value → re-encode → replay. Classic priv check, takes 30 seconds with Decoder.