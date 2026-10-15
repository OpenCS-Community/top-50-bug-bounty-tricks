# 📝 Reporting & Mindset (Tricks #48–#50)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Bug bounty isn't just about finding bugs — it's about **choosing the right
targets**, **chaining weak bugs into critical ones**, and **writing reports
that triagers actually reward**. These three tricks are the meta-skills that
separate top hunters from everyone else.

---

## Table of Contents

1. [Think in Chains, Not Single Bugs](#trick-48)
2. [Chase Depth, Not Volume](#trick-49)
3. [Write Reports That Triagers Love](#trick-50)

---

<a name="trick-48"></a>
## 🧩 Trick #48 — Think in Chains, Not Single Bugs

**Category:** Mindset
**Impact:** Turns low-severity bugs into Critical payouts
**Tools:** Personal notes, gadget notebook

### The Idea
Most hunters report a single bug and move on. Top hunters keep a **"gadget
notebook"** — a running list of weak bugs they've found but not reported.
Later, when they find another weak bug, they check the notebook for chains.

A low-severity bug alone might pay $50. Chained with another low-severity bug,
it pays $5,000.

### Classic Chains

| Bug A (Low)         | + Bug B (Low)         | = Chain (Critical)       |
|---------------------|-----------------------|--------------------------|
| Open redirect       | OAuth flow            | Account takeover         |
| Self-XSS            | CSRF                  | Stored XSS               |
| CSPT                | JSON endpoint         | Account takeover         |
| Prototype pollution | DOM gadget            | XSS                      |
| Cache deception     | Authenticated page    | PII disclosure           |
| Session fixation    | Login page            | Account takeover         |
| PostMessage `*`     | Message handler       | Privilege escalation     |
| SSRF (blind)        | Internal service      | RCE / data exfil         |
| Host header inject  | Password reset        | Account takeover         |
| Deep link hijack    | OAuth flow            | Account takeover         |

### How to Build a Gadget Notebook

**Structure:**
```markdown
## Target: target.com

### Gadgets (unreported)

#### Open Redirect
- URL: /redirect?url=
- Payload: /redirect?url=https://evil.com
- Filter: blocks "http" but allows "//evil.com"

#### CSPT
- Endpoint: /api/user/{id}/avatar
- Payload: ../../../admin
- No filter on ../

#### PostMessage
- Handler: window.addEventListener('message')
- No origin check
- Actions: setEmail, setRole

#### Possible Chains
- Open redirect + OAuth → token theft
- CSPT + open redirect → ATO
- PostMessage + setEmail → ATO
```

### The Chain-Building Workflow

1. **Find a weak bug → add to notebook.**
2. **Keep hunting the same target.**
3. **When you find another weak bug → check notebook.**
4. **If they chain → test the chain.**
5. **If the chain works → report it as one Critical.**

### Example — Real Chain

**Setup:**
- Bug A: `/login?next=` doesn't validate → open redirect.
- Bug B: `/api/user/{id}/avatar` allows `../` → CSPT.

**Chain:**
1. `/api/user/../../../redirect?url=https://evil.com/payload.json/avatar`
2. Browser follows redirect to attacker JSON.
3. App fetches JSON as user's profile.
4. Attacker controls victim's profile email.
5. Password reset goes to attacker.
6. **ATO.**

**Reported as:** "Account takeover via CSPT + open redirect chain"
**Severity:** Critical
**Payout:** Top tier

### Why It Works
- Triagers evaluate **impact**, not just bug class.
- A chain has higher impact than either bug alone.
- Programs pay more for Critical → maximize your chains.

### Tips

1. **Never report a weak bug immediately** — save it as a gadget.
2. **Document every weak bug** — even self-XSS, even open redirect.
3. **Re-check your notebook** every time you find a new bug.
4. **Learn classic chains** (list above).
5. **Test chains in a sandbox** before reporting.
6. **Report the chain, not the individual bugs** — it's one Critical, not
   three Low.

### References
- [PortSwigger Web Security Academy — Exploiting chains](https://portswigger.net/web-security)

---

<a name="trick-49"></a>
## 🧩 Trick #49 — Chase Depth, Not Volume

**Category:** Mindset
**Impact:** Higher payouts, less burnout
**Tools:** Focus, patience

### The Idea
Most hunters spray tools across 100 targets and find nothing. Top hunters
pick **one target** and go **deep** — reading docs, understanding business
logic, and finding bugs nobody else can see.

Scanners find the same 20 bugs on every target. Those bugs are usually
duplicates. **Depth finds the uniques.**

### The Contrast

| Volume Approach                 | Depth Approach                    |
|---------------------------------|-----------------------------------|
| 100 subdomains, `nuclei` scan   | 1 target, read every endpoint     |
| Report 50 Low bugs              | Find 1 Critical bug                |
| Duplicates everywhere           | Unique findings                    |
| $50–$500 total                  | $2,000–$20,000 total               |
| Burnout in 3 months             | Sustainable for years              |
| Relies on tools                 | Relies on understanding            |

### The Depth Playbook

**Phase 1 — Recon (1–2 days):**
- Enumerate all subdomains.
- Map every endpoint (Burp + `katana`).
- Read every JS file.
- Identify tech stack (frameworks, versions, services).

**Phase 2 — Understanding (2–5 days):**
- Create an account.
- Use every feature.
- Read documentation (help pages, API docs, changelogs).
- Understand what the app is **supposed** to do.

**Phase 3 — Hunting (ongoing):**
- Test business logic (Tricks #30–#34).
- Test auth flows (Tricks #24–#29).
- Test API authorization (Tricks #40–#45).
- Chain weak bugs (Trick #48).

**Phase 4 — Persistence:**
- Re-test after every release.
- Watch for new features (changelogs, blogs, tweets).
- New features = new bugs.

### What to Focus On

**High-value areas (fewer duplicates):**
- Business logic (payments, refunds, coupons)
- Multi-tenant isolation (BOLA across orgs)
- Complex auth (SSO, OAuth, SAML)
- Internal APIs (`/api/internal/`, `/admin/`)
- Race conditions (Tricks #14–#17)
- Custom file parsers (upload, import)
- Mobile app ↔ web ↔ API interactions

**Low-value areas (duplicate-heavy):**
- Reflected XSS on public pages
- Missing security headers
- Open redirects on `/logout`
- Cookie flags
- Directory listing on `/static/`

### Signs You're Going Deep

- You can explain the app's **business model** to someone.
- You know **all its features** — even ones you can't use.
- You've read its **changelog** for the last 6 months.
- You have a **gadget notebook** with 5+ weak bugs.
- You've tested **every parameter** of critical endpoints.

### Signs You're Going Wide (and should stop)

- You're running `nuclei` on 50 subdomains and hoping.
- You're reporting the same bug on 10 different hosts.
- You can't remember what the target does.
- You haven't created an account.
- You're chasing the "next shiny target" every week.

### Why It Works
- Duplicates come from shallow scanning.
- Uniques come from understanding.
- Deep targets have **hidden endpoints** that scanners miss.
- Business logic bugs require understanding → no scanner finds them.

### How to Pick a Deep Target

**Good signs:**
- Complex business logic (payments, multi-role)
- Recently launched features
- Public API
- Mobile app
- Rich user-generated content
- Scope includes `*.target.com`

**Bad signs:**
- Simple static site
- Tiny scope (`www.target.com` only)
- Public program with thousands of hunters
- No recent updates
- No user accounts

### References
- [Bug Bounty Reports Explained (YouTube)](https://www.youtube.com/@BugBountyReportsExplained)

---

<a name="trick-50"></a>
## 🧩 Trick #50 — Write Reports That Triagers Love

**Category:** Mindset
**Impact:** Faster triage, higher payouts, repeat relationships
**Tools:** Markdown, screen recording tools

### The Idea
A great bug with a bad report gets closed as Informative. A modest bug with a
great report gets rewarded. **The report is your product.** Treat it that way.

Triagers read hundreds of reports a week. If yours is clear, concise, and
reproducible, they'll love you — and they'll reward your future work faster.

### The Perfect Report Template

```markdown
# [Severity] — [Vulnerability] in [Endpoint/Feature] leads to [Impact]

## Summary
One-paragraph description. What the bug is, where it lives, and what an
attacker can do.

## Severity
CVSS 3.1: [Score] ([Vector string])
Rationale: [why this severity]

## Affected Component
- URL: https://target.com/endpoint
- Parameter: userId
- Method: GET/POST

## Steps to Reproduce
1. Log in as attacker@test.com / password.
2. Navigate to https://target.com/endpoint?id=1.
3. Intercept the request in Burp.
4. Change id to 2 (victim's ID).
5. Observe the response contains victim's email/address/etc.

## Proof of Concept
[Request]
```
GET /api/users/2 HTTP/1.1
Host: target.com
Cookie: session=ATTACKER_SESSION
```

[Response]
```
HTTP/1.1 200 OK
{"id":2,"email":"victim@target.com","address":"..."}
```

[Video / Screenshot]
See attached `poc.mp4`.

## Impact
An authenticated attacker can read any user's PII (name, email, address,
phone number) by enumerating the `id` parameter. This affects ~N users.

## Remediation
- Enforce ownership check on `/api/users/{id}`:
  ```python
  if user.id != current_user.id:
      return 403
  ```
- Use opaque, per-user IDs.
- Add rate limiting per user.

## References
- [OWASP BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
```

### What Triagers Actually Want

| Do                             | Don't                          |
|--------------------------------|--------------------------------|
| One-sentence title             | Vague title ("XSS bug")        |
| Clear impact statement         | "Could be dangerous"           |
| Exact steps to reproduce       | "Just do XSS"                  |
| Raw HTTP requests/responses    | Screenshots of Burp only       |
| Video PoC for complex chains   | No PoC                         |
| Suggested fix                  | "Fix it please"                |
| CVSS score + vector            | "High severity"                |
| Proof of victim test account   | "Trust me, it works"           |
| Concise (under 1 page)         | Walls of text                  |

### The Anatomy of Impact

**Bad impact:** "This is a serious vulnerability."
**Good impact:** "An authenticated attacker can read any user's PII."

**Bad impact:** "Could lead to XSS."
**Good impact:** "The stored XSS executes in the admin panel, allowing an
attacker to create new admin users."

**Bad impact:** "Leads to account takeover."
**Good impact:** "An attacker can take over any user's account in 3 requests
by exploiting the reset token leak via Referer."

### Proof of Concept Best Practices

**1. Raw HTTP requests:**
Always include the exact request/response. Triagers replay them.

**2. Video PoC:**
For multi-step chains, record a 30–60 second screen video (Loom, OBS).

**3. Test accounts:**
Provide credentials for any test accounts you created, so triagers can
reproduce quickly.

**4. Burp project file:**
For complex bugs, attach a `.burp` project file with the requests.

**5. Clean up sensitive data:**
Redact real victim PII (replace with `REDACTED`). Use your own test accounts.

### Common Report Mistakes

**1. Wrong scope:**
- Reporting bugs on out-of-scope assets.
- Reporting bugs found via out-of-scope chains.

**2. No impact:**
- "XSS on self-only page" → Informative.
- Always demonstrate the attacker's gain.

**3. No repro steps:**
- "XSS in search parameter" without a payload.
- Triagers can't reproduce → closed.

**4. Duplicate:**
- Same bug already reported.
- Before reporting, search the program's disclosed reports.

**5. Testing in production with real victims:**
- Never test on real user data.
- Always use your own test accounts.

**6. Aggressive communication:**
- Pinging every day.
- Demanding higher payout.
- Arguing with triagers.

### The Triager Relationship

**Good behavior:**
- Report clearly and concisely.
- Respond to triage questions promptly.
- Accept triage decisions gracefully.
- Report further bugs on the same program.

**Bad behavior:**
- Ping for updates.
- Dispute severity without evidence.
- Report low-quality spam.
- Threaten disclosure.

**Why it matters:** Triagers remember good hunters. A reputation for quality
gets you:
- Faster triage.
- Higher payouts (discretionary bonuses).
- Early access to private programs.
- Direct contact with security teams.

### Post-Report Checklist

- [ ] Title clearly states severity, bug, and impact.
- [ ] Summary is one paragraph.
- [ ] Severity has CVSS + rationale.
- [ ] Steps to reproduce are numbered and exact.
- [ ] Raw HTTP requests/responses included.
- [ ] Video PoC attached (for chains).
- [ ] Impact is concrete (what attacker gains).
- [ ] Remediation is specific.
- [ ] No real victim data (redacted).
- [ ] Test accounts provided.
- [ ] Spell-checked.
- [ ] Under 1 page (unless complex).

### Template Repositories

- [bugcrowd/report-templates](https://github.com/bugcrowd)
- [OWASP Vulnerability Disclosure Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html)

---

## 📌 Quick Reference Summary

| #  | Trick                    | Impact                        |
|----|--------------------------|-------------------------------|
| 48 | Think in chains          | Turns Low → Critical payouts  |
| 49 | Chase depth, not volume  | Unique bugs, sustainable      |
| 50 | Reports triagers love    | Faster triage, higher payouts |

---

## 🏁 You've Reached the End

You now have the **Top 50 Bug Bounty Hunting Tricks** — from recon and SSRF
to race conditions, business logic, and reporting.

**Next steps:**
1. Pick one trick and try it on a target **today**.
2. Build your own **gadget notebook**.
3. Pick a target and go **deep** for a month.
4. Write **one great report**.

**Remember:**
- Depth beats volume.
- Chains beat single bugs.
- Reports beat tools.

Good hunting. 🎯

---

> **Prev:** [Mobile ←](09-mobile.md)
> **Back to:** [README](../README.md)
