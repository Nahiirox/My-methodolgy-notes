# Parameter Fuzzing — POST

POST params go in `-d`, not the URL. PHP endpoints need the Content-Type header or the server won't parse the body:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<DOMAIN>:<PORT>/<path> -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <BASELINE_SIZE>
```

Confirm a hit with curl before chasing it further:

```bash
curl http://<DOMAIN>:<PORT>/<path> -X POST -d '<PARAM>=key' -H 'Content-Type: application/x-www-form-urlencoded'
```

"Invalid id!" = param accepted, wrong value → fuzz the value next. "Not found" / same baseline response = param not real.

> [!tip] Run GET and POST param fuzzing on the same endpoint — different params can exist on each method, as seen here with `id` only appearing on POST.