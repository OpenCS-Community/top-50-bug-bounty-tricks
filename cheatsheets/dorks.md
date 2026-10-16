# 🔎 Google Dorks Cheatsheet

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

A curated list of Google dorks for bug bounty reconnaissance — cloud storage,
config files, admin panels, exposed secrets, and more.

---

## Table of Contents

1. [Cloud Storage](#cloud-storage)
2. [Config & Secrets](#config--secrets)
3. [Admin Panels](#admin-panels)
4. [API Keys & Credentials](#api-keys--credentials)
5. [Subdomains & Hosts](#subdomains--hosts)
6. [Browser Extensions](#browser-extensions)
7. [Backups & Dumps](#backups--dumps)
8. [Directory Listings](#directory-listings)
9. [Error Pages & Debug Info](#error-pages--debug-info)
10. [Paste Sites & Code Repos](#paste-sites--code-repos)
11. [Documents & Files](#documents--files)
12. [Login & Auth Pages](#login--auth-pages)
13. [IoT & Devices](#iot--devices)
14. [WordPress Specific](#wordpress-specific)
15. [Jira / Jenkins / GitLab](#jira--jenkins--gitlab)
16. [Combining Dorks](#combining-dorks)

---

## Cloud Storage

### Amazon S3

```
site:s3.amazonaws.com "target"
site:s3.amazonaws.com "target" ext:sql
site:s3.amazonaws.com "target" ext:env
site:s3.amazonaws.com "target" ext:bak
site:s3.amazonaws.com "target" ext:log
site:s3.amazonaws.com "target" ext:json
site:s3.amazonaws.com "target" "index of"
site:s3.amazonaws.com "target" "private"
```

### Google Cloud Storage

```
site:storage.googleapis.com "target"
site:storage.googleapis.com "target" ext:sql
site:storage.googleapis.com "target" ext:json
```

### Azure Blob Storage

```
site:blob.core.windows.net "target"
site:blob.core.windows.net "target" ext:sql
```

### DigitalOcean Spaces

```
site:digitaloceanspaces.com "target"
site:nyc3.digitaloceanspaces.com "target"
site:sgp1.digitaloceanspaces.com "target"
```

### Other CDNs

```
site:rackcdn.com "target"
site:cloudfront.net "target"
site:azureedge.net "target"
site:fastly.net "target"
```

### Public Bucket Listings

```
intitle:"index of" "s3.amazonaws.com"
intitle:"index of" "storage.googleapis.com"
intitle:"index of" "blob.core.windows.net"
```

---

## Config & Secrets

```
site:target.com ext:env
site:target.com ext:config
site:target.com ext:cfg
site:target.com ext:ini
site:target.com ext:yaml
site:target.com ext:yml
site:target.com ext:xml
site:target.com ext:toml
site:target.com inurl:.env
site:target.com inurl:config
site:target.com inurl:settings
site:target.com inurl:.git
site:target.com inurl:.svn
site:target.com inurl:web.config
site:target.com inurl:appsettings.json
site:target.com "DB_PASSWORD"
site:target.com "API_KEY"
site:target.com "SECRET_KEY"
site:target.com "AWS_ACCESS_KEY_ID"
site:target.com "BEGIN RSA PRIVATE KEY"
site:target.com "BEGIN OPENSSH PRIVATE KEY"
```

---

## Admin Panels

```
site:target.com inurl:admin
site:target.com inurl:administrator
site:target.com inurl:adminpanel
site:target.com inurl:admin/login
site:target.com inurl:dashboard
site:target.com inurl:manage
site:target.com inurl:management
site:target.com inurl:panel
site:target.com inurl:console
site:target.com inurl:controlpanel
site:target.com inurl:phpmyadmin
site:target.com inurl:wp-admin
site:target.com inurl:cpanel
site:target.com inurl:webmail
site:target.com intitle:"admin panel"
site:target.com intitle:"login"
site:target.com intitle:"dashboard"
```

---

## API Keys & Credentials

```
site:pastebin.com "target.com"
site:pastebin.com "target.com" password
site:pastebin.com "target.com" api_key
site:github.com "target.com" "api_key"
site:github.com "target.com" "password"
site:github.com "target.com" "secret"
site:github.com "target.com" "BEGIN RSA PRIVATE KEY"
site:gitlab.com "target.com" "token"
site:bitbucket.org "target.com" "secret"
site:trello.com "target.com"
site:target.com "authorization: bearer"
site:target.com "x-api-key"
site:target.com "AKIA"
```

---

## Subdomains & Hosts

```
site:*.target.com -www
site:*.target.com -www -mail
site:target.com -site:www.target.com
site:*.target.com inurl:dev
site:*.target.com inurl:staging
site:*.target.com inurl:test
site:*.target.com inurl:beta
site:*.target.com inurl:internal
site:*.target.com inurl:api
site:*.target.com inurl:portal
site:*.target.com inurl:jenkins
site:*.target.com inurl:gitlab
site:*.target.com inurl:jira
```

---

## Browser Extensions

```
site:chrome.google.com/webstore "target"
site:chromewebstore.google.com "target"
site:addons.mozilla.org "target"
site:microsoftedge.microsoft.com/addons "target"
site:chrome.google.com/webstore "internal"
site:chrome.google.com/webstore "admin"
```

---

## Backups & Dumps

```
site:target.com ext:bak
site:target.com ext:backup
site:target.com ext:old
site:target.com ext:orig
site:target.com ext:save
site:target.com ext:swp
site:target.com ext:swo
site:target.com ext:tmp
site:target.com ext:temp
site:target.com ext:copy
site:target.com ext:1
site:target.com inurl:backup
site:target.com inurl:.git
site:target.com inurl:.svn
site:target.com inurl:dump.sql
site:target.com inurl:database.sql
site:target.com inurl:backup.zip
site:target.com inurl:backup.tar.gz
```

---

## Directory Listings

```
site:target.com intitle:"index of"
site:target.com intitle:"index of /"
site:target.com intitle:"index of" "parent directory"
site:target.com intitle:"index of" ".git"
site:target.com intitle:"index of" ".env"
site:target.com intitle:"index of" "backup"
site:target.com intitle:"index of" "config"
site:target.com intitle:"index of" "password"
site:target.com intitle:"index of" "private"
site:target.com intitle:"index of" "logs"
```

---

## Error Pages & Debug Info

```
site:target.com intext:"SQL syntax"
site:target.com intext:"mysql_fetch"
site:target.com intext:"Warning: mysql"
site:target.com intext:"Fatal error"
site:target.com intext:"stack trace"
site:target.com intext:"PHP Notice"
site:target.com intext:"Notice: Undefined"
site:target.com intext:"Traceback (most recent call last)"
site:target.com intext:"Whitelabel Error Page"
site:target.com intitle:"error"
site:target.com intitle:"exception"
site:target.com inurl:debug
site:target.com inurl:phpinfo.php
site:target.com inurl:server-status
site:target.com inurl:server-info
```

---

## Paste Sites & Code Repos

```
site:pastebin.com "target.com"
site:pastebin.com "target" "password"
site:pastebin.com "target" "api_key"
site:pastebin.com "target" "@"
site:ghostbin.com "target"
site:hastebin.com "target"
site:dpaste.com "target"
site:gitlab.com "target.com"
site:bitbucket.org "target.com"
site:gist.github.com "target.com"
site:replit.com "target"
site:codepen.io "target"
```

---

## Documents & Files

```
site:target.com filetype:pdf
site:target.com filetype:doc
site:target.com filetype:docx
site:target.com filetype:xls
site:target.com filetype:xlsx
site:target.com filetype:ppt
site:target.com filetype:pptx
site:target.com filetype:csv
site:target.com filetype:txt
site:target.com filetype:log
site:target.com filetype:sql
site:target.com filetype:xml
site:target.com filetype:json
site:target.com filetype:conf
site:target.com filetype:cnf
site:target.com filetype:ini
site:target.com filetype:sh
site:target.com filetype:bash
site:target.com filetype:ps1
site:target.com filetype:bat
```

### Sensitive Filenames

```
site:target.com ext:pdf "confidential"
site:target.com ext:pdf "internal use only"
site:target.com ext:xlsx "credentials"
site:target.com ext:xlsx "salary"
site:target.com ext:csv "password"
site:target.com ext:sql "INSERT INTO"
site:target.com ext:log "Authorization"
```

---

## Login & Auth Pages

```
site:target.com inurl:login
site:target.com inurl:signin
site:target.com inurl:signup
site:target.com inurl:register
site:target.com inurl:auth
site:target.com inurl:oauth
site:target.com inurl:sso
site:target.com inurl:saml
site:target.com inurl:password
site:target.com inurl:reset
site:target.com intitle:"login"
site:target.com intitle:"sign in"
```

---

## IoT & Devices

```
site:target.com inurl:cam
site:target.com inurl:camera
site:target.com inurl:webcam
site:target.com intitle:"webcamXP"
site:target.com intitle:"Live View / - AXIS"
site:target.com intitle:"DVR"
site:target.com intitle:"NVR"
site:target.com intitle:"Router"
site:target.com intitle:"printer"
site:target.com inurl:8080
site:target.com inurl:8443
```

---

## WordPress Specific

```
site:target.com inurl:wp-admin
site:target.com inurl:wp-content
site:target.com inurl:wp-includes
site:target.com inurl:wp-json
site:target.com inurl:xmlrpc.php
site:target.com inurl:wp-config.php
site:target.com inurl:wp-config.php.bak
site:target.com inurl:wp-cron.php
site:target.com inurl:readme.html
site:target.com ext:sql inurl:wp-
```

---

## Jira / Jenkins / GitLab

```
site:target.com inurl:jira
site:target.com inurl:secure/Dashboard.jspa
site:target.com inurl:secure/Signup!default.jspa
site:target.com inurl:jenkins
site:target.com intitle:"Dashboard [Jenkins]"
site:target.com inurl:gitlab
site:target.com inurl:users/sign_up
site:target.com inurl:confluence
site:target.com inurl:servicenow
site:target.com inurl:navpage.do
```

---

## Combining Dorks

You can stack dorks for precision:

```
site:target.com inurl:admin ext:php
site:target.com inurl:api ext:json
site:target.com inurl:backup ext:sql
site:target.com intitle:"index of" ext:env
site:target.com filetype:pdf "internal"
site:*.target.com -www -mail -blog
site:target.com inurl:dev OR inurl:staging OR inurl:test
```

### Operators Cheatsheet

| Operator    | Meaning                          | Example                  |
|-------------|----------------------------------|--------------------------|
| `site:`     | Limit to a domain                | `site:target.com`        |
| `inurl:`    | String in URL                    | `inurl:admin`            |
| `intitle:`  | String in page title             | `intitle:"index of"`     |
| `intext:`   | String in page body              | `intext:"password"`      |
| `filetype:` | File extension (`ext:` also)     | `filetype:pdf`           |
| `ext:`      | File extension                   | `ext:env`                |
| `-`         | Exclude                          | `-www`                   |
| `OR` / `\|` | OR operator                      | `inurl:dev OR inurl:test`|
| `"..."`     | Exact match                      | `"api key"`              |
| `*`         | Wildcard                         | `site:*.target.com`      |
| `..`        | Numeric range                    | `2019..2024`             |

---

## Automation Tips

### Google Dorking Tools

- **`pagodo`** — automated Google dorking for bug bounty
- **`dorky`** — bulk Google dorking
- **`fast-recon`** — combining dorks with other tools

### Rate Limit Awareness

Google blocks automated queries quickly. Add delays:

- **1 query per 5–10 seconds**
- Rotate user agents
- Use different IPs (proxies) if needed
- Prefer **Bing** or **DuckDuckGo** for automation — less strict

### Combine With Other Sources

- **Shodan** — `shodan search "target.com"`
- **Censys** — `censys search "target.com"`
- **FOFA** — `fofa.info`
- **ZoomEye** — `zoomeye.org`
- **BinaryEdge** — `binaryedge.io`

---

## Legal Reminder

Google dorking is **legal** when used against your own systems or authorized
targets. Using these dorks against systems you don't own or have permission to
test is **illegal** in most jurisdictions.

Always verify:

- You're within the bug bounty program's scope.
- The target is on the authorized list.
- You have written permission for private engagements.

---

> **Back to:** [README](../README.md)
> **Prev:** [Payloads ←](payloads.md)
> **Next:** [Tools →](tools.md)
