# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Contributors section in README
- Banner image in `/assets`
- Interactive web version of the Top 50
- Translations (ES, FR, DE, PT-BR, HI)
- Additional cheatsheets (per tech stack)
- Video walkthroughs for select tricks

---

## [1.0.0] — 2026-10-15

### Added

**Core Documentation**
- `README.md` — Main entry point with full Top 50 list, category tables, and
  quick links.
- `LICENSE` — MIT License (Copyright © 2026 Computer Science Projects).
- `CONTRIBUTING.md` — Contribution guide with style rules, PR process, and
  recognition policy.
- `CODE_OF_CONDUCT.md` — Contributor Covenant v2.1 with security-specific
  additions.
- `CHANGELOG.md` — This file.
- `.gitignore` — Ignore rules for OS, IDE, Python, Node, secrets, and notes.

**Deep-Dive Docs (`/docs`)**
- `01-recon.md` — Tricks #1–#6: origin IP discovery, browser extension
  enumeration, open registration on internal tools, job-posting
  fingerprinting, cloud storage dorking, SPF/DMARC subdomain enumeration.
- `02-ssrf-cloud.md` — Tricks #7–#13: DNS rebinding, backend auth-header
  theft, IMDSv2 redirector bypass, IP encoding bypasses, PDF/thumbnail blind
  SSRF, `file://` / `gopher://` / `dict://` schemes, XXE → SSRF.
- `03-race-conditions.md` — Tricks #14–#17: single-packet attack, coupon
  reuse, TOCTOU on balance checks, HTTP/2 rate-limit bypass.
- `04-client-side.md` — Tricks #18–#23: CSPT, open redirect + CSPT chain,
  silent CSTI payloads, DOM clobbering, postMessage `null` origin,
  prototype pollution → XSS.
- `05-auth-session.md` — Tricks #24–#29: OAuth `redirect_uri` bypass, JWT
  `alg:none` / key confusion, session fixation, password reset token via
  Referer, MFA response manipulation, email verification race.
- `06-business-logic.md` — Tricks #30–#34: demand-based price abuse,
  negative quantity/amount, purchase limit bypass, free trial abuse, currency
  confusion.
- `07-cache-infra.md` — Tricks #35–#39: web cache deception, cache poisoning
  via unkeyed headers, CPDoS, HTTP request smuggling (CL.TE / TE.CL), host
  header reset poisoning.
- `08-api-modern.md` — Tricks #40–#45: GraphQL introspection + batching,
  GraphQL field duplication for rate-limit bypass, mass assignment, BOLA /
  IDOR, WebSocket origin bypass (CSWSH), gRPC reflection.
- `09-mobile.md` — Tricks #46–#47: SSL pinning bypass with Frida, deep link
  hijacking for auth token theft.
- `10-reporting-mindset.md` — Tricks #48–#50: think in chains, chase depth,
  write reports triagers love.

**Cheatsheets (`/cheatsheets`)**
- `payloads.md` — SSRF, XSS, SSTI, path traversal, open redirect, JWT,
  GraphQL payloads.
- `dorks.md` — Google dorks for cloud storage, config files, admin panels,
  API keys, subdomains, extensions.
- `tools.md` — Recon, fuzzing, SSRF, race conditions, JWT, mobile, GraphQL,
  utility tools.

### Notes

- All docs include a security disclaimer at the top.
- All tricks include category, impact, tools, steps, why it works, fix, and
  references.
- No live unpatched bugs or victim data are included.

---

## How to Update This Changelog

When contributing, add an entry under `[Unreleased]` in the appropriate
section:

- **Added** — for new tricks, docs, or cheatsheets.
- **Changed** — for edits to existing content.
- **Deprecated** — for content being phased out.
- **Removed** — for deleted content.
- **Fixed** — for typos, broken links, or corrected payloads.
- **Security** — for security-relevant changes to the repo itself.

Format:

```markdown
### Added
- Trick #51 — Your new trick (#PR_NUMBER)

### Fixed
- Typo in Trick #31 (#PR_NUMBER)

### Changed
- Improved explanation in Trick #14 (#PR_NUMBER)
```

When a release is cut, move the unreleased items into a new version heading
with the release date.

---

## Versioning

This project uses **Semantic Versioning**:

- **MAJOR** — structural changes or removals (e.g., re-organizing docs).
- **MINOR** — new tricks, new cheatsheets, new categories.
- **PATCH** — typo fixes, small corrections, formatting.

---

## Links

- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
- [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
- [Conventional Commits](https://www.conventionalcommits.org/)

---

> **Maintainers:** Computer Science Projects
> **Last updated:** 2026-10-15
