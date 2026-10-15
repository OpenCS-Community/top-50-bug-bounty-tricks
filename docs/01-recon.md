# 🔍 Recon & Attack Surface (Tricks #1–#6)

> ⚠️ Disclaimer: Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Recon is where bug bounties are won or lost. The tricks below focus on finding
attack surface that scanners miss — origin IPs, forgotten extensions,
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

Category: Recon
Impact: High (WAF bypass)
Tools: crt.sh, SecurityTrails, Shodan, dig

### The Idea
Targets sitting behind Cloudflare, Akamai, or Fastly still have an **origin
server** somewhere. Historical DNS records, Certificate Transparency logs, and
misconfigured services often leak that origin IP — giving you a direct line
that bypasses the WAF entirely.

### Steps

1. Query Certificate Transparency logs
  
   curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u
   
2. Pull historical A records
   - [SecurityTrails](https://securitytrails.com/) → DNS History
   - [ViewDNS](https://viewdns.info/iphistory/) → IP History
   - [DNSDumpster](https://dnsdumpster.com/)

3. Cross-reference with Shodan
  
   ssl:"target.com"
   ssl.cert.subject.CN:"target.com"
   http.title:"target.com"
   
4. Confirm the origin
  
   curl -H "Host: target.com" http://CANDIDATE_IP/
   
   If the response matches the real site (or looks like a default page for the
   target), you've found the origin.

5. Test WAF bypass
   Send a known-blocked payload (e.g., <script>alert(1)</script>) to the
   origin IP with the correct Host header. If it executes, you bypassed the
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

Category: Recon
Impact: High (leaked secrets, internal APIs)
Tools: crxextractor, unzip, grep, trufflehog

### The Idea
Companies often publish internal Chrome/Firefox extensions to the public
Web Store so employees can install them easily. These extensions frequently
contain hardcoded API keys, internal hostnames, and staging endpoints.

### Steps

1. Find candidate extensions
  
   site:chrome.google.com/webstore "target"
   site:chromewebstore.google.com "target"
   site:addons.mozilla.org "target"
   site:microsoftedge.microsoft.com/addons "target"
   
2. Download and unpack
   - Chrome Web Store ID → https://clients2.google.com/service/update2/crx?response=redirect&prodversion=100&acceptformat=crx3&x=id%3D<ID>%26uc
   - Rename to .zip, unzip, or use crxextractor.com.

3. Grep for secrets
  
   grep -rEi "(api[_-]?key|token|secret|password|bearer)" ./extracted/
   grep -rEo "https?://[a-zA-Z0-9._-]+" ./extracted/ | sort -u
   
4. Run a secret scanner
  
   trufflehog filesystem ./extracted/ --only-verified
   
5. Test internal endpoints
   - If you find https://internal-staging.target.com/api/..., try it
     unauthenticated.
   - Check for auth bypass or default creds.

### Why It Works
Extension source is fully client-side and readable. Developers assume only
employees will inspect it.