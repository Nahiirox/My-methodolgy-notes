# Search Engine Discovery

Passive recon — no packets sent, all public data. Use before touching anything.

## Operators

`site:` → restrict to domain / `inurl:` → term in URL / `filetype:` or `ext:` → file type / `intitle:` → term in page title / `intext:` → term in body / `cache:` → cached version of page

Combine with `AND`, `OR`, `-` (exclude), `" "` (exact phrase), `*` (wildcard), `..` (range).

## Dork Patterns

Login/admin panels:

```
site:<TARGET> inurl:login
site:<TARGET> (inurl:login OR inurl:admin)
```

Exposed files — docs, spreadsheets:

```
site:<TARGET> (filetype:pdf OR filetype:xls OR filetype:docx)
```

Config files — often contain creds or stack details:

```
site:<TARGET> inurl:config.php
site:<TARGET> (ext:conf OR ext:cnf OR ext:env)
```

DB backups:

```
site:<TARGET> (inurl:backup OR filetype:sql)
```

Creds / sensitive strings in body:

```
site:<TARGET> intext:"password"
site:<TARGET> intext:"BEGIN PRIVATE KEY"
```

> [!tip] More dorks → Google Hacking Database (GHDB). Search by category before building your own.