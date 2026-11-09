# Attacking Web Applications with Ffuf

## Introduction

ffuf is a web fuzzer for directories, files and extensions, vhosts, PHP parameters, and parameter values.

ffuf: https://github.com/ffuf/ffuf

## Web Fuzzing

Finds hidden pages and directories with wordlists. Response 404 means missing, 200 means it exists.

### Fuzzing

Sending many inputs to an interface to study how it reacts.

### Wordlists

SecLists: https://github.com/danielmiessler/SecLists

```bash
locate directory-list-2.3-small.txt
```

`-ic` ignores the copyright comment lines at the top of the wordlist

## Directory Fuzzing

Fuzzes web directories by placing the keyword assigned to the wordlist where the directory would be in the URL.

### Ffuf

```bash
apt install ffuf -y
```

`-u` target URL | `-w` wordlist path, optional :KEYWORD | `-H` header | `-X` HTTP method, default GET | `-b` cookie data | `-d` POST data | `-o` write output to file | `-c` colored output | `-t` threads, 200 is faster but can DoS a remote site

### Directory Fuzzing

```bash
ffuf -w directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ
```

Empty page without 404 or 403 means the directory exists and is accessible.

## Page Fuzzing

Finds hidden pages in a directory. Identify the page extension first, then fuzz filenames with it.

### Extension Fuzzing

Fuzz the extension on index, since most sites have an index file. A Server header guess (Apache .php, IIS .asp or .aspx) is less practical than fuzzing.

```bash
ffuf -w web-extensions.txt:FUZZ -u http://<TARGET_IP>:<PORT>/<DIRECTORY>/indexFUZZ # fuzz extension on index
```

The wordlist already contains the dot, so do not add one after index.

A 200 response shows the extension the site uses. Two wordlists need a unique keyword each, such as FUZZ_1.FUZZ_2.

### Page Fuzzing

```bash
ffuf -w directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/<DIRECTORY>/FUZZ.php
```

Size 0 means an empty page, other 200 hits have content.

## Recursive Fuzzing

Starts a new scan under every directory found, automating directory and file fuzzing across the whole site.

### Recursive Flags

Always set a depth, deep trees take very long. Then rerun on the most interesting directories.

`-recursion` enable recursive scan | `-recursion-depth` max depth, 1 fuzzes main directories and direct sub-directories only | `-e` extension, such as .php, usually site-wide | `-v` full URLs

Recursion needs the URL to end in FUZZ and only supports the FUZZ keyword.

### Recursive Scanning

```bash
ffuf -w directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ -recursion -recursion-depth 1 -e .php -v
```

## DNS Records

Browsers resolve names through /etc/hosts, then public DNS. A name in neither fails to connect, so add it to /etc/hosts.

```bash
sudo sh -c 'echo "<TARGET_IP>  <TARGET_DOMAIN>" >> /etc/hosts'
```

Add the port to the URL when browsing.

## Sub-domain Fuzzing

Finds sub-domains (`*.<TARGET_DOMAIN>`) that have a public DNS record.

### Sub-domains

```bash
ffuf -w subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.<TARGET_DOMAIN>/
```

Finds only public DNS records. A main domain in /etc/hosts does not cover its sub-domains.

## Vhost Fuzzing

A vhost is a sub-domain served on the same server and IP. It may or may not have a public DNS record.

### Vhosts vs. Sub-domains

Sub-domain fuzzing finds only public sub-domains. Vhost fuzzing on a known IP finds public and non-public ones.

### Vhosts Fuzzing

Fuzzes the Host header, so the wordlist does not go into /etc/hosts.

```bash
ffuf -w subdomains-top1million-5000.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/ -H 'Host: FUZZ.<TARGET_DOMAIN>'
```

Every word returns 200. A real vhost returns a different response size.

## Filtering Results

Match or filter results by status code, size, words, lines, or regexp. Default matcher keeps 200,204,301,302,307,401,403.

`-mc` match status codes, `all` for everything | `-ml` match lines | `-mr` match regexp | `-ms` match size | `-mw` match words

`-fc` filter status codes | `-fl` filter lines | `-fr` filter regexp | `-fs` filter size | `-fw` filter words

`-fc` `-fl` `-fs` `-fw` take comma separated lists and ranges.

Use a filter when the wrong response size is known and the valid one is not.

```bash
ffuf -w subdomains-top1million-5000.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/ -H 'Host: FUZZ.<TARGET_DOMAIN>' -fs <SIZE> # filter default response size
```

Add the found vhost to /etc/hosts before browsing, and keep the current port.

## Parameter Fuzzing - GET

Finds unpublished GET parameters a page accepts. They tend to be less tested and less secured, so test them for web vulnerabilities.

### GET Request Fuzzing

```bash
ffuf -w burp-parameter-names.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/<DIRECTORY>/<TARGET_FILE>?FUZZ=key -fs <SIZE> # filter default response size
```

## Parameter Fuzzing - POST

POST data goes in the request data field, not in the URL.

```bash
ffuf -w burp-parameter-names.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/<DIRECTORY>/<TARGET_FILE> -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <SIZE> # fuzz POST parameters
curl http://<TARGET_DOMAIN>:<PORT>/<DIRECTORY>/<TARGET_FILE> -X POST -d '<PARAM>=key' -H 'Content-Type: application/x-www-form-urlencoded' # test found parameter
```

PHP accepts only application/x-www-form-urlencoded for POST data, set it with -H.

## Value Fuzzing

Fuzzes the value of a working parameter with a pre-made or custom wordlist matching the expected value type.

### Custom Wordlist

Look for a matching list in SecLists first, such as for usernames. For an id parameter, build a number list.

```bash
for i in $(seq 1 1000); do echo $i >> <WORDLIST>; done
```

### Value Fuzzing

```bash
ffuf -w <WORDLIST>:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/<DIRECTORY>/<TARGET_FILE> -X POST -d '<PARAM>=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <SIZE>
```

Send the hit value with the curl POST request from the previous section.