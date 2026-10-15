# 🔍 Recon & Attack Surface (Tricks #1–#6)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Recon is where bug bounties are won or lost. The tricks below focus on finding
**attack surface that scanners miss** — origin IPs, forgotten extensions,
publicly exposed internal tools, and passive OSINT leaks.

---

## Table of Contents

1. [Find Origin IPs via Historical DNS + CT Logs](#trick-1)
2. [Enumerate Publicly Published Internal Browser Extensions](#trick-2)
3. [Check Open Registration on Jira / Jenkins / ServiceNow](#trick-3)
4. [Mine Job Postings for Tech Stack Fingerprinting](#trick-4)
5. [Google Dork for Misconfigured Cloud Storage](#trick-5)
6. [Enumerate Subdomains via SPF/DMARC Misconfigurations](#trick-6)

---

<a name="trick-1"></a>
## 🧩 Trick #1 — Find Origin IPs via Historical DNS + CT Logs

**Category:** Recon
**Impact:** High (WAF bypass)
**Tools:** `crt.sh`, SecurityTrails, Shodan, `dig`

### The Idea
Targets sitting behind Cloudflare, Akamai, or Fastly still have an **origin
server** somewhere. Historical DNS records, Certificate Transparency logs, and
misconfigured services often leak that origin IP — giving you a direct line
that bypasses the WAF entirely.

### Steps

1. **Query Certificate Transparency logs**
   ```bash
   curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u
   ```

2. **Pull historical A records**
   - [SecurityTrails](https://securitytrails.com/) → DNS History
   - [ViewDNS](https://viewdns.info/iphistory/) → IP History
   - [DNSDumpster](https://dnsdumpster.com/)

3. **Cross-reference with Shodan**
   ```
   ssl:"target.com"
   ssl.cert.subject.CN:"target.com"
   http.title:"target.com"
   ```

4. **Confirm the origin**
   ```bash
   curl -H "Host: target.com" http://CANDIDATE_IP/
   ```
   If the response matches the real site (or looks like a default page for the
   target), you've found the origin.

5. **Test WAF bypass**
   Send a known-blocked payload (e.g., `<script>alert(1)</script>`) to the
   origin IP with the correct `Host` header. If it executes, you bypassed the
   WAF.

### Why It Works
- Certificates are often issued to origin IPs *before* WAF onboarding.
- Old DNS records persist in historical databases.
- Many orgs forget to firewall the origin after moving to a CDN.

### Fix
- Restrict origin to WAF IP ranges (AWS SG / Cloudflare Tunnel / Akamai).
- Rotate origin IP after CDN migration.
- Enable mTLS between WAF and origin.
- Monitor CT logs for unintended cert issuance.

### References
- [Cloudflare: Protecting your origin](https://developers.cloudflare.com/fundamentals/concepts/cloudflare-ip-addresses/)

---

<a name="trick-2"></a>
## 🧩 Trick #2 — Enumerate Publicly Published Internal Browser Extensions

**Category:** Recon
**Impact:** High (leaked secrets, internal APIs)
**Tools:** `crxextractor`, `unzip`, `grep`, `trufflehog`

### The Idea
Companies often publish **internal** Chrome/Firefox extensions to the public
Web Store so employees can install them easily. These extensions frequently
contain hardcoded API keys, internal hostnames, and staging endpoints.

### Steps

1. **Find candidate extensions**
   ```
   site:chrome.google.com/webstore "target"
   site:chromewebstore.google.com "target"
   site:addons.mozilla.org "target"
   site:microsoftedge.microsoft.com/addons "target"
   ```

2. **Download and unpack**
   - Chrome Web Store ID → `https://clients2.google.com/service/update2/crx?response=redirect&prodversion=100&acceptformat=crx3&x=id%3D<ID>%26uc`
   - Rename to `.zip`, unzip, or use `crxextractor.com`.

3. **Grep for secrets**
   ```bash
   grep -rEi "(api[_-]?key|token|secret|password|bearer)" ./extracted/
   grep -rEo "https?://[a-zA-Z0-9._-]+" ./extracted/ | sort -u
   ```

4. **Run a secret scanner**
   ```bash
   trufflehog filesystem ./extracted/ --only-verified
   ```

5. **Test internal endpoints**
   - If you find `https://internal-staging.target.com/api/...`, try it
     unauthenticated.
   - Check for auth bypass or default creds.

### Why It Works
Extension source is fully client-side and readable. Developers assume only
employees will inspect it.

### Fix
- Never hardcode secrets in client-side code.
- Never publish internal extensions publicly.
- Rotate any leaked keys immediately.
- Use short-lived tokens via backend.

---

<a name="trick-3"></a>
## 🧩 Trick #3 — Check Open Registration on Jira / Jenkins / ServiceNow

**Category:** Recon
**Impact:** Critical (authenticated access to internal tooling)
**Tools:** Browser, `nuclei` (Jira/Jenkins templates)

### The Idea
Many orgs expose internal tools (Jira, Jenkins, GitLab, ServiceNow) to the
internet for contractor access — and forget to disable **public signup**.
Registration often grants you a **real user session**, exposing private
projects, issues, and sometimes build infrastructure.

### Common Signup Paths

| Tool       | Signup URL                                    |
|------------|-----------------------------------------------|
| Jira       | `/secure/Signup!default.jspa`                 |
| Confluence | `/signup.action`                              |
| Jenkins    | `/signup` or `/securityRealm/createAccount`   |
| GitLab     | `/users/sign_up`                              |
| ServiceNow | `/navpage.do` → "Register" (self-registration)|
| SonarQube  | `/account/create`                             |

### Steps

1. **Fingerprint the tool**
   ```bash
   curl -sI https://target.com/ | grep -i "x-*\|server"
   curl -s https://target.com/ | grep -i "jira\|jenkins\|gitlab"
   ```

2. **Visit the signup path** and attempt registration with a throwaway email.

3. **If registration succeeds:**
   - Enumerate projects/repos.
   - Look for exposed secrets in CI configs.
   - Check for default users (`admin:admin`, `test:test`).

4. **Nuclei quick check**
   ```bash
   nuclei -u https://target.com -tags jira,jenkins,gitlab
   ```

### Why It Works
- Contractors need accounts → admin enables signup → forgets to disable.
- Email verification is often misconfigured or skipped.
- Internal tools rarely have robust auth.

### Fix
- Disable public signup.
- Restrict by email domain (email domain allowlist).
- Put behind VPN / SSO.
- Monitor for unexpected account creation.

---

<a name="trick-4"></a>
## 🧩 Trick #4 — Mine Job Postings for Tech Stack Fingerprinting

**Category:** Recon
**Impact:** Medium
**Tools:** LinkedIn, Indeed, Glassdoor, Google

### The Idea
Job postings list **exact technologies, versions, and internal tool names**.
This is passive recon gold: you learn what to target before touching the app.

### What to Look For
- Frameworks: "Spring Boot", "Django", "Rails", "Laravel"
- Cloud: "AWS", "GCP", "Azure", "Kubernetes", "EKS"
- Gateways: "Kong", "Apigee", "Istio", "Envoy"
- Auth: "Okta", "Auth0", "Keycloak", "Azure AD"
- Databases: "Postgres", "MongoDB", "Cassandra"
- Monitoring: "Datadog", "Splunk", "New Relic"
- Internal tools: "Jira", "Confluence", "Backstage"

### Steps

1. **Search job boards**
   ```
   site:linkedin.com/jobs "target.com" "engineer"
   site:indeed.com "target" "backend engineer"
   site:glassdoor.com "target" "software engineer"
   ```

2. **Extract stack keywords** into a notes file.

3. **Cross-reference with the live app**
   - Check response headers (`X-Powered-By`, `Server`).
   - Check JS bundle for framework hints.
   - Look for `/api/`, `/graphql`, `/actuator` (Spring Boot), `/admin`.

4. **Build a targeted wordlist**
   ```
   /actuator/env
   /actuator/heapdump
   /graphql
   /api/v1/admin
   /keycloak/
   /okta/
   ```

### Why It Works
Recruiters overshare. Engineers list exact versions ("Spring Boot 2.7",
"Kubernetes 1.26"). This narrows your attack surface dramatically.

### Fix
- N/A (OSINT hygiene — you can't stop this, but you can patch the tech).

---

<a name="trick-5"></a>
## 🧩 Trick #5 — Google Dork for Misconfigured Cloud Storage

**Category:** Recon
**Impact:** Critical (customer data, backups, PII)
**Tools:** Google, `cloud_enum`, `s3scanner`

### The Idea
Misconfigured S3 / GCS / Azure Blob buckets regularly leak **customer data,
database backups, and source code**. Naming conventions are predictable, and
Google indexes public ones.

### Dorks

```
site:s3.amazonaws.com "target"
site:s3.amazonaws.com "target" ext:sql
site:s3.amazonaws.com "target" ext:env
site:storage.googleapis.com "target"
site:blob.core.windows.net "target"
site:digitaloceanspaces.com "target"
site:nyc3.digitaloceanspaces.com "target"
site:rackcdn.com "target"
```

### Common Bucket Naming Patterns

```
target-backup
target-backups
target-prod
target-dev
target-staging
target-assets
target-media
target-logs
target-db-backup
target-uploads
```

### Steps

1. **Run dorks** and note all hits.

2. **Test for public listing**
   ```bash
   curl -s https://BUCKET.s3.amazonaws.com/ | head
   ```
   If it returns XML `<ListBucketResult>`, listing is enabled.

3. **Enumerate with cloud_enum**
   ```bash
   cloud_enum -k target -k target-backup -k target-prod
   ```

4. **Check for writable buckets**
   ```bash
   aws s3 cp test.txt s3://BUCKET/ --no-sign-request
   ```

5. **Look for sensitive files**
   - `.env`, `config.json`, `backup.sql`, `dump.sql`
   - `credentials`, `id_rsa`, `.pem`
   - Customer exports (CSV, XLSX)

### Why It Works
Buckets are created quickly during sprints and forgotten. Public read is a
one-click mistake.

### Fix
- Enable S3 Block Public Access at the account level.
- Enable bucket logging + CloudTrail.
- Use IAM policies + bucket policies strictly.
- Regular audits with `s3scanner` / AWS Config rules.

### References
- [AWS: Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)

---

<a name="trick-6"></a>
## 🧩 Trick #6 — Enumerate Subdomains via SPF/DMARC Misconfigurations

**Category:** Recon
**Impact:** Medium
**Tools:** `dig`, `dnsx`, `amass`

### The Idea
SPF records list **mail servers** (often internal). DMARC `rua` / `ruf`
addresses leak internal domains. Together, they reveal subdomains that
subdomain brute-forcing might miss.

### Steps

1. **Pull SPF record**
   ```bash
   dig +short TXT target.com | grep spf
   ```
   Example output:
   ```
   "v=spf1 include:_spf.google.com include:mail.internal.target.com ip4:203.0.113.0/24 -all"
   ```

2. **Extract domains + IP ranges**
   - `include:` entries → new domains to enumerate
   - `ip4:` / `ip6:` → netblocks to scan
   - `a:` / `mx:` → hostnames

3. **Pull DMARC**
   ```bash
   dig +short TXT _dmarc.target.com
   ```
   Example:
   ```
   "v=DMARC1; p=reject; rua=mailto:dmarc@internal.target.com; ruf=mailto:forensics@target.com"
   ```
   → reveals `internal.target.com`

4. **Pull MX records**
   ```bash
   dig +short MX target.com
   ```

5. **Feed everything back into subdomain enumeration**
   ```bash
   amass enum -d target.com -include-domains internal.target.com -passive
   dnsx -d target.com -r resolvers.txt -silent
   ```

6. **Also check these records for leaks**
   ```bash
   dig +short TXT _dmarc.target.com
   dig +short TXT default._domainkey.target.com
   dig +short TXT _acme-challenge.target.com
   dig +short CAA target.com
   ```

### Why It Works
DNS is public by design. Orgs rarely sanitize SPF/DMARC strings, and internal
hostnames commonly appear there.

### Fix
- Minimize `include:` entries in SPF.
- Use neutral domains for `rua` / `ruf` mailboxes.
- Audit DNS records quarterly.
- Use CAA to prevent rogue cert issuance.

---

## 📌 Quick Reference Summary

| # | Trick                       | Impact   | Effort |
|---|-----------------------------|----------|--------|
| 1 | Origin IP discovery         | High     | Medium |
| 2 | Internal extension enum     | High     | Low    |
| 3 | Open signup on internal tools | Critical | Low  |
| 4 | Job posting fingerprinting  | Medium   | Low    |
| 5 | Cloud storage dorking       | Critical | Low    |
| 6 | SPF/DMARC subdomain enum    | Medium   | Low    |

---

> **Next:** [SSRF & Cloud →](02-ssrf-cloud.md)
