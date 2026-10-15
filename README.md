<div align="center">

# 🎯 Top 50 Bug Bounty Hunting Tricks

**A curated, practical collection of high-impact techniques from real hunters.**

By **Computer Science Projects** — your hub for ethical hacking, OSINT, bug
bounty, programming, and networking.

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Channel](https://img.shields.io/badge/WhatsApp-Channel-25D366?logo=whatsapp&logoColor=white)](https://whatsapp.com/channel/0029Vb6d5MtAInPhysn10r06)
[![Telegram](https://img.shields.io/badge/Telegram-Channel-26A5E4?logo=telegram&logoColor=white)](https://t.me/computerscienceprojects)
[![Group](https://img.shields.io/badge/Telegram-Group-26A5E4?logo=telegram&logoColor=white)](https://t.me/hk3knsu)

</div>

---

## 👋 Welcome to Computer Science Projects

We're a community for learners and practitioners of:

- 🔐 **Ethical Hacking**
- 🕵️ **OSINT**
- 🎯 **Bug Bounty Hunting**
- 💻 **Programming**
- 🌐 **Networking**

This repository is one of our **flagship resources** — a curated collection of
50 battle-tested bug bounty tricks, written to be **actionable, not
theoretical**.

> ⚠️ **Disclaimer:** For educational purposes only. Only test on programs you
> are authorized to test. Always follow scope and disclosure rules. Never
> publish live unpatched bugs or victim data.

---

## 📢 Join the Community

Stay updated with new tricks, write-ups, and projects:

| Platform            | Link                                                                 |
|---------------------|----------------------------------------------------------------------|
| 💬 **WhatsApp Channel** | [Join here](https://whatsapp.com/channel/0029Vb6d5MtAInPhysn10r06) |
| 💬 **WhatsApp Group**   | [Join here](https://chat.whatsapp.com/LWNI2iE9CgQKJzC5Bo1mUm)      |
| 📨 **Telegram Channel** | [@computerscienceprojects](https://t.me/computerscienceprojects)   |
| 👥 **Telegram Group**   | [Join here](https://t.me/hk3knsu)                                  |

> **Rule reminder:** Share only CS-related content. No spam. No off-topic
> chats. **If you're serious about studying, stay.** 🎓

---

## 🗂️ The Top 50

### 🔍 Recon & Attack Surface — [#1–#6](docs/01-recon.md)
| # | Trick | Impact |
|---|-------|--------|
| 1 | Find origin IPs via historical DNS + CT logs to bypass WAF | High |
| 2 | Enumerate publicly published internal browser extensions | High |
| 3 | Check open registration on Jira/Jenkins/ServiceNow | Critical |
| 4 | Mine job postings for tech stack fingerprinting | Medium |
| 5 | Google dork for misconfigured cloud storage (S3/GCP/Azure) | Critical |
| 6 | Enumerate subdomains via SPF/DMARC misconfigurations | Medium |

### 🌐 SSRF & Cloud — [#7–#13](docs/02-ssrf-cloud.md)
| # | Trick | Impact |
|---|-------|--------|
| 7 | DNS rebinding to bypass SSRF allowlists | Critical |
| 8 | Steal backend `Authorization` headers via unvalidated webhooks | Critical |
| 9 | IMDSv2 bypass using an external redirector | Critical |
| 10 | Bypass SSRF filters with IPv6, decimal, and octal IPs | High |
| 11 | Exploit PDF/thumbnail generators for blind SSRF | High |
| 12 | Access cloud metadata via `file://`, `gopher://`, `dict://` | Critical |
| 13 | Force SSRF through XML parsers (XXE → SSRF) | Critical |

### ⚡ Race Conditions — [#14–#17](docs/03-race-conditions.md)
| # | Trick | Impact |
|---|-------|--------|
| 14 | Single-packet attack for reliable race conditions | Critical |
| 15 | Coupon reuse via parallel redemption | High |
| 16 | Multiple withdrawals via TOCTOU on balance checks | Critical |
| 17 | Bypass rate limits with HTTP/2 multiplexing | High |

### 🖥️ Client-Side Attacks — [#18–#23](docs/04-client-side.md)
| # | Trick | Impact |
|---|-------|--------|
| 18 | Client-Side Path Traversal (CSPT) as a gadget | High |
| 19 | Chain open redirect + CSPT for account takeover | Critical |
| 20 | Silent CSTI payloads via `constructor` (no `alert`) | High |
| 21 | DOM clobbering to hijack script behavior | High |
| 22 | PostMessage origin bypass via `null` origin | High |
| 23 | Prototype pollution → XSS chain | Critical |

### 🔐 Auth & Session — [#24–#29](docs/05-auth-session.md)
| # | Trick | Impact |
|---|-------|--------|
| 24 | OAuth `redirect_uri` bypass via path traversal | Critical |
| 25 | JWT `alg:none` + key confusion attacks | Critical |
| 26 | Session fixation via pre-auth cookies | High |
| 27 | Password reset token leakage via Referer | Critical |
| 28 | Bypass MFA with response manipulation | Critical |
| 29 | Account takeover via email verification race | High |

### 💼 Business Logic — [#30–#34](docs/06-business-logic.md)
| # | Trick | Impact |
|---|-------|--------|
| 30 | Price manipulation via demand-based logic | Critical |
| 31 | Negative quantity / amount injection in carts | Critical |
| 32 | Bypass purchase limits via parameter tampering | High |
| 33 | Free trial abuse via email alias tricks | Medium |
| 34 | Currency confusion in multi-currency checkouts | High |

### 🗄️ Cache & Infrastructure — [#35–#39](docs/07-cache-infra.md)
| # | Trick | Impact |
|---|-------|--------|
| 35 | Web cache deception via path confusion | High |
| 36 | Cache poisoning via unkeyed headers | Critical |
| 37 | CPDoS (Cache Poisoned Denial of Service) | High |
| 38 | HTTP request smuggling via CL.TE / TE.CL | Critical |
| 39 | Host header injection for password reset poisoning | Critical |

### 🔌 API & Modern Stacks — [#40–#45](docs/08-api-modern.md)
| # | Trick | Impact |
|---|-------|--------|
| 40 | GraphQL introspection + batching attacks | High |
| 41 | GraphQL field duplication for rate-limit bypass | High |
| 42 | Mass assignment via hidden API fields | Critical |
| 43 | BOLA/IDOR in REST APIs via UUID enumeration | Critical |
| 44 | WebSocket origin bypass + CSWSH | High |
| 45 | gRPC reflection for API discovery | Medium |

### 📱 Mobile — [#46–#47](docs/09-mobile.md)
| # | Trick | Impact |
|---|-------|--------|
| 46 | Bypass SSL pinning with Frida scripts | High |
| 47 | Deep link hijacking for auth token theft | Critical |

### 📝 Reporting & Mindset — [#48–#50](docs/10-reporting-mindset.md)
| # | Trick | Impact |
|---|-------|--------|
| 48 | Think in chains, not single bugs | — |
| 49 | Chase depth, not volume | — |
| 50 | Write reports that triagers love | — |

---

## 📚 Cheatsheets

- [Payloads Cheatsheet](cheatsheets/payloads.md) — SSRF, XSS, SSTI, JWT, XXE, SQLi, and more
- [Google Dorks](cheatsheets/dorks.md)
- [Tools](cheatsheets/tools.md)

---

## 📁 Repository Structure

```
bug-bounty-tricks/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
├── /docs              # 10 deep-dive category files
├── /cheatsheets       # Payloads, dorks, tools
└── /assets            # Images and diagrams
```

---

## 🤝 Contributing

We welcome contributions from the community! See
[CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

- New tricks
- Improved payloads
- Better explanations
- New cheatsheets
- Translations

---

## 📜 License

MIT — see [LICENSE](LICENSE).

Copyright © 2026 **Computer Science Projects**

---

<div align="center">

### 🎓 Keep Learning. Keep Hacking. Stay Ethical.

**Made with ❤️ by Computer Science Projects**

[WhatsApp Channel](https://whatsapp.com/channel/0029Vb6d5MtAInPhysn10r06) •
[WhatsApp Group](https://chat.whatsapp.com/LWNI2iE9CgQKJzC5Bo1mUm) •
[Telegram Channel](https://t.me/computerscienceprojects) •
[Telegram Group](https://t.me/hk3knsu)

</div>
