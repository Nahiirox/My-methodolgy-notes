# FUFF

Ffuf is a web fuzzer that sends wordlist-driven requests, reads response codes and sizes, and separates real pages from 404 noise. It covers directories, files/extensions, vhosts, parameters, and parameter values.

## Web Fuzzing

Fuzzing means sending varied input to an interface and studying the reaction:

* SQL injection: special characters
* Buffer overflow: growing strings
* Web applications: wordlists of common paths

Servers do not publish their page tree. Probe common names and let HTTP responses answer:

* `404`: no page
* `200`: real page

Use **SecLists** rather than building your own wordlists. On Pwnbox, it is available at `/opt/useful/SecLists`.

Standard directory wordlist:

```
/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
```

It starts with copyright comments that can pollute results. Use `-ic` to strip them.

## Directory Fuzzing

The core `ffuf` pattern uses:

* `-w`: wordlist, optionally followed by `:KEYWORD`
* `-u`: URL with the keyword in the fuzz target position

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ
```

The default 40 threads can send roughly 88k requests in under 10 seconds. Although `-t 200` can increase speed, it may DoS a remote target or choke your own connection.

Results show entries like:

```
[Status: 301, Size: 326, ...]
```

A `301` or `200` directory with an empty page still means access was successful. Fuzz pages beneath it.

Key flags:

| Flag          | Purpose     |
| ------------- | ----------- |
| `-H`          | Header      |
| `-X`          | HTTP method |
| `-b`          | Cookies     |
| `-d`          | POST data   |
| `-mc` / `-ms` | Matchers    |
| `-fc` / `-fs` | Filters     |
| `-o`          | Output      |

Default matching includes `200`, `204`, `301`, `302`, `307`, `401`, and `403`.

## Page Fuzzing

### Extension Fuzzing

Find the site's extension before fuzzing filenames. Guessing from server headers is unreliable, so fuzz it instead.

Every site has an `index.*` page. Fuzz its extension with the web-extensions wordlist, which already includes the dot:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ
```

If `.php` returns `200`, the site uses PHP.

When needed, use two wordlists and two keywords, such as `FUZZ_1.FUZZ_2`, to fuzz names and extensions together.

### Page Fuzzing With a Known Extension

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php
```

Check the `Size` column:

* `index` with size `0`: empty page
* Non-zero size: content worth visiting

## Recursive Fuzzing

Automate the directory, subdirectory, and file discovery loop:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v
```

| Option               | Purpose                                                        |
| -------------------- | -------------------------------------------------------------- |
| `-recursion`         | Re-scan every newly found directory                            |
| `-recursion-depth 1` | Stop at direct subdirectories                                  |
| `-e .php`            | Append the extension, effectively scanning with and without it |
| `-v`                 | Print full URLs                                                |

Deep directory trees can greatly increase scan time. Start with a limited depth, then re-run targeted scans against interesting branches.

Recursive fuzzing can cost roughly six times as many requests as a flat scan, but one command can map the whole tree.

## DNS Records

Browsers resolve names through `/etc/hosts` and public DNS. Internal or lab hostnames such as `academy.htb` exist in neither and may result in a connection failure.

Add the hostname manually:

```bash
sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'
```

If the named domain serves the same site as the raw IP, look for subdomains.

## Sub-domain Fuzzing

Sub-domain fuzzing finds **public** subdomains with DNS records pointing to a working server.

SecLists provides relevant wordlists under:

```
/opt/useful/seclists/Discovery/DNS/
```

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.inlanefreight.com/
```

Hits are resolvable, live subdomains such as `support`, `ns3`, `blog`, or `www`.

On a non-public domain such as `academy.htb`, the same scan can return only errors because the main domain resolves only through `/etc/hosts` and no other subdomain exists in public DNS.

That does not mean there are no subdomains. It means there are no public ones. Use vhost fuzzing instead.

## Vhost Fuzzing

A **VHost** is a subdomain served from the same server or IP address. One IP can host multiple sites.

VHosts may not have public DNS records, so subdomain fuzzing cannot find them. Instead, send requests to the known host and vary the `Host:` header:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb'
```

Every request may return `200` because the default page always responds. The key indicator is response **size**: a recognized VHost serves different content from the default page.

Filter out the default response size:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs 900
```

Verify a hit by adding it to `/etc/hosts`, such as `admin.academy.htb`, then browse to it. Different content and a `404` on paths that exist on the main host confirm a separate VHost.

## Filtering Results

Default filtering keeps:

* `200`
* `204`
* `301`
* `302`
* `307`
* `401`
* `403`

It drops `404`.

When everything returns `200`, such as during vhost fuzzing or with custom 404 pages, filter based on the response body fingerprint.

| Option        | Purpose                 |
| ------------- | ----------------------- |
| `-fs` / `-ms` | Response size           |
| `-fw` / `-mw` | Word count              |
| `-fl` / `-ml` | Line count              |
| `-fr` / `-mr` | Regular expression      |
| `-fc` / `-mc` | Status codes and ranges |

Use **matching** when you know what a hit looks like.

Use **filtering** when you know what misses look like. This is usually the better approach: identify the constant response size from noise and exclude it.

## Parameter Fuzzing: GET

Pages such as `admin.php` that refuse access without login may accept an undocumented parameter such as `?param1=key`.

GET parameters are placed after `?` in the URL:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx
```

Replace `xxx` with the default response size.

Hits can be dead ends. A deprecated parameter may still respond differently even though it is no longer in use.

Fuzzing parameters can expose unpublished parameters that are less tested and less secured.

## Parameter Fuzzing: POST

POST data lives in the request body rather than the URL.

* `-d` sets the data.
* `-X POST` sets the method.
* PHP accepts `application/x-www-form-urlencoded`.

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```

Verify hits directly:

```bash
curl http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=key' -H 'Content-Type: application/x-www-form-urlencoded'
```

A changed error such as `Invalid id!` confirms the parameter is live.

## Value Fuzzing

The last step is fuzzing an accepted parameter's value.

Pre-made lists work for guessable values such as usernames. Custom parameters require custom lists.

For a numeric `id`, generate a list:

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

Fuzz the value position:

```bash
ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```

For a hit, replay it with `curl`.

The workflow is:

1. Fuzz directories.
2. Fuzz extensions.
3. Fuzz pages.
4. Recurse into discovered directories.
5. Vhost-fuzz beyond the public DNS boundary.
6. Fuzz GET parameters.
7. Fuzz POST parameters.
8. Build a value list.
9. Fuzz parameter values.
