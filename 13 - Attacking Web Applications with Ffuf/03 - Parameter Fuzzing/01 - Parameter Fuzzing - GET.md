# Parameter Fuzzing — GET

Fuzz the parameter name, use a dummy value, filter baseline size:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<DOMAIN>:<PORT>/<path>?FUZZ=key -fs <BASELINE_SIZE>
```

Hit with different size = valid parameter. Visit it and check the response — "deprecated" or similar means the param exists but is dead, keep enumerating.

> [!tip] Run unfiltered first for one second to grab the baseline size, then rerun with `-fs`. Same drill as vhost fuzzing.