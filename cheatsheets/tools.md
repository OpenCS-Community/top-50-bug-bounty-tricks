# 🛠️ Tools Cheatsheet

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

A curated list of tools for every phase of bug bounty hunting — recon,
fuzzing, exploitation, and reporting.

---

## Table of Contents

1. [Recon & Subdomain Enumeration](#recon--subdomain-enumeration)
2. [HTTP Probing & Content Discovery](#http-probing--content-discovery)
3. [Port Scanning](#port-scanning)
4. [Fuzzing](#fuzzing)
5. [Vulnerability Scanning](#vulnerability-scanning)
6. [SSRF & OOB Interaction](#ssrf--oob-interaction)
7. [Race Conditions](#race-conditions)
8. [JWT Tools](#jwt-tools)
9. [GraphQL Tools](#graphql-tools)
10. [Mobile Tools](#mobile-tools)
11. [API Testing](#api-testing)
12. [Exploitation Frameworks](#exploitation-frameworks)
13. [Proxy & Interception](#proxy--interception)
14. [Encoding & Payload Generation](#encoding--payload-generation)
15. [Wordlists](#wordlists)
16. [Cloud & OSINT](#cloud--osint)
17. [Reporting & Documentation](#reporting--documentation)
18. [Installation & Setup](#installation--setup)

---

## Recon & Subdomain Enumeration

### Subdomain Enumeration

| Tool            | Description                                       |
|-----------------|---------------------------------------------------|
| **subfinder**   | Fast passive subdomain enumeration                |
| **amass**       | In-depth subdomain enumeration (active + passive) |
| **assetfinder** | Find domains and subdomains                       |
| **findomain**   | Fast cross-platform subdomain enumerator          |
| **sublist3r**   | Passive subdomain enumeration                     |
| **puredns**     | Fast domain resolver and subdomain bruteforcer    |
| **shuffledns**  | Wrapper around massdns                            |
| **dnsx**        | Fast DNS toolkit                                  |
| **massdns**     | High-performance DNS resolver                     |

### ASN & IP Enumeration

| Tool            | Description                          |
|-----------------|--------------------------------------|
| **asnmap**      | Map networks to ASN                  |
| **bgp.he.net**  | Web-based ASN lookup                 |
| **whois**       | WHOIS lookup                         |
| **amass intel** | ASN + reverse whois                  |

### Certificate Transparency

| Tool / Site      | Description                     |
|------------------|---------------------------------|
| **crt.sh**       | Web-based CT log search         |
| **certspotter**  | CT log monitoring               |
| **tlsx**         | TLS grabber for CT data         |

### Continuous Recon

| Tool         | Description                        |
|--------------|------------------------------------|
| **reNgine**  | Automated recon framework          |
| **reconftw** | Automated recon workflow           |
| **osmedeus** | Automated recon & vulnerability scan |
| **nuclei**   | Template-based scanner (also recon) |

---

## HTTP Probing & Content Discovery

### HTTP Probing

| Tool         | Description                                  |
|--------------|----------------------------------------------|
| **httpx**    | Fast HTTP toolkit (probing, tech detection)  |
| **httprobe** | Simple HTTP probe                            |
| **aquatone** | Visual inspection of hosts                   |
| **gowitness**| Web screenshot tool                          |

### Content Discovery

| Tool            | Description                                  |
|-----------------|----------------------------------------------|
| **ffuf**        | Fast web fuzzer (dirs, params, vhosts)       |
| **feroxbuster** | Recursive content discovery                  |
| **dirsearch**   | Web path scanner                             |
| **gobuster**    | Directory/file & DNS busting                 |
| **wfuzz**       | Web fuzzer                                   |
| **katana**      | Next-gen crawling framework                  |
| **gau**         | Get all URLs (AlienVault, Wayback, etc.)     |
| **waybackurls** | Fetch URLs from Wayback Machine              |
| **hakrawler**   | Fast web crawler for JS, forms, endpoints    |
| **paramspider** | Mine parameters from web archives            |
| **arjun**       | HTTP parameter discovery                     |
| **x8**          | Hidden parameter discovery                   |

### JavaScript Analysis

| Tool            | Description                          |
|-----------------|--------------------------------------|
| **LinkFinder**  | Extract endpoints from JS            |
| **SecretFinder**| Find secrets in JS                   |
| **subjs**       | Fetch JS files from domains          |
| **getJS**       | Extract JS file URLs                 |
| **jsluice**     | Extract URLs and secrets from JS     |
| **JSFinder**    | JS endpoint extractor                |

---

## Port Scanning

| Tool         | Description                            |
|--------------|----------------------------------------|
| **nmap**     | Industry-standard port scanner         |
| **masscan**  | Fastest port scanner                   |
| **naabu**    | Fast port scanner (ProjectDiscovery)   |
| **rustscan** | Fast port scanner with nmap integration|
| **zmap**     | Internet-wide scanner                  |

### Common Nmap Commands

```bash
# Fast scan
nmap -T4 -F target.com

# Full TCP scan
nmap -p- -T4 target.com

# Service + version detection
nmap -sV -sC target.com

# UDP top 100
nmap -sU --top-ports 100 target.com

# Vuln scripts
nmap --script vuln target.com

# All ports + version + default scripts
nmap -p- -sV -sC -T4 target.com
```

---

## Fuzzing

### Directory / File Fuzzing

| Tool            | Description                     |
|-----------------|---------------------------------|
| **ffuf**        | Fast web fuzzer                 |
| **feroxbuster** | Recursive content discovery     |
| **dirsearch**   | Directory scanner               |
| **gobuster**    | Directory/DNS busting           |
| **wfuzz**       | Web fuzzer                      |

### Parameter Fuzzing

| Tool            | Description                          |
|-----------------|--------------------------------------|
| **ffuf**        | Parameter fuzzing                    |
| **arjun**       | Parameter discovery                  |
| **x8**          | Hidden parameters                    |
| **param-miner** | Burp extension for hidden params     |

### VHost Fuzzing

```bash
ffuf -w subdomains.txt -u http://TARGET_IP -H "Host: FUZZ.target.com" -fs 0
```

### Common ffuf Commands

```bash
# Directory brute force
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -u https://target.com/FUZZ

# Extension fuzzing
ffuf -w wordlist.txt -u https://target.com/FUZZ \
  -e .php,.html,.js,.json,.bak

# Parameter fuzzing
ffuf -w params.txt -u "https://target.com/api?FUZZ=test"

# POST body fuzzing
ffuf -w params.txt -X POST -d "FUZZ=test" \
  -u https://target.com/api

# Header fuzzing
ffuf -w headers.txt -u https://target.com \
  -H "FUZZ: test"
```

---

## Vulnerability Scanning

| Tool           | Description                         |
|----------------|-------------------------------------|
| **nuclei**     | Template-based scanner (fastest)    |
| **nikto**      | Web server scanner                  |
| **wpscan**     | WordPress vulnerability scanner     |
| **joomscan**   | Joomla vulnerability scanner        |
| **droopescan** | Drupal scanner                      |
| **sqlmap**     | SQL injection automation            |
| **XSStrike**   | XSS detection suite                 |
| **dalfox**     | XSS scanner                         |
| **commix**     | Command injection exploiter         |
| **tplmap**     | SSTI detection and exploitation     |

### Nuclei Common Commands

```bash
# Update templates
nuclei -update-templates

# Scan with all templates
nuclei -u https://target.com

# Scan specific tags
nuclei -u https://target.com -tags cve,rce,sqli

# Scan with severity filter
nuclei -u https://target.com -severity critical,high

# Scan list of targets
nuclei -l targets.txt -o results.txt

# Rate limit
nuclei -u https://target.com -rate-limit 100

# Workflow with subdomains
subfinder -d target.com | httpx | nuclei
```

---

## SSRF & OOB Interaction

| Tool                   | Description                              |
|------------------------|------------------------------------------|
| **interactsh**         | OOB interaction server (ProjectDiscovery)|
| **Burp Collaborator**  | Built into Burp Suite Pro                |
| **ngrok**              | Expose local server to internet          |
| **webhook.site**       | Instant webhook URL                      |
| **requestbin**         | Request capture                          |
| **rebind.network**     | DNS rebinding service                    |
| **dnsFookup**          | DNS rebinding tool                       |
| **ssrfmap**            | SSRF exploitation framework              |
| **Gopherus**           | Gopher payload generator                 |

### Interactsh Usage

```bash
# Start listener
interactsh-client

# Generates a URL like: abc123.interactsh.com
# Use it as the SSRF target
```

---

## Race Conditions

| Tool                  | Description                          |
|-----------------------|--------------------------------------|
| **Burp Repeater**     | Single-packet attack via tab groups  |
| **Turbo Intruder**    | Custom race scripts (Burp extension) |
| **race-the-web**      | Race condition testing tool          |
| **h2spacex**          | HTTP/2 single-packet attack library  |

### Turbo Intruder — Single Packet Attack

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    for i in range(30):
        engine.queue(target.req, gate='race1')
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

---

## JWT Tools

| Tool                  | Description                          |
|-----------------------|--------------------------------------|
| **jwt_tool**          | Comprehensive JWT testing            |
| **jwt.io**            | Online JWT debugger                  |
| **hashcat**           | JWT secret cracking                  |
| **john**              | JWT secret cracking                  |
| **Burp JWT Editor**   | Burp extension for JWT manipulation  |
| **jwt-cracker**       | Quick secret brute force             |

### jwt_tool Commands

```bash
# Decode
python3 jwt_tool.py <token>

# Test alg:none
python3 jwt_tool.py <token> -X a

# Test key confusion
python3 jwt_tool.py <token> -X k -pk public.pem

# Crack secret
python3 jwt_tool.py <token> -C -d rockyou.txt

# Tamper claims
python3 jwt_tool.py <token> -T
```

### Hashcat JWT Cracking

```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

---

## GraphQL Tools

| Tool                  | Description                       |
|-----------------------|-----------------------------------|
| **graphql-cop**       | GraphQL security scanner          |
| **InQL**              | Burp extension for GraphQL        |
| **clairvoyance**      | Schema recovery when introspection off |
| **graphql-voyager**   | Visual schema explorer            |
| **Altair**            | GraphQL client                    |
| **GraphQLmap**        | GraphQL exploitation tool         |

### Common Commands

```bash
# Introspection
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__schema{types{name}}}"}'

# graphql-cop
graphql-cop -t https://target.com/graphql

# clairvoyance
clairvoyance https://target.com/graphql -o schema.json -w wordlist.txt
```

---

## Mobile Tools

| Tool           | Description                                |
|----------------|--------------------------------------------|
| **Frida**      | Dynamic instrumentation                    |
| **objection**  | Runtime mobile exploration                 |
| **MobSF**      | Mobile security framework (static + dynamic)|
| **jadx**       | Android decompiler                         |
| **apktool**    | APK decoder                                |
| **Ghidra**     | Reverse engineering (NSA)                  |
| **needle**     | iOS security testing                       |
| **drozer**     | Android attack surface analysis            |
| **mitmproxy**  | Intercept mobile traffic                   |

### Frida Common Commands

```bash
# List apps
frida-ps -U

# List running processes
frida-ps -Uai

# Attach to running app
frida -U com.target.app

# Spawn and inject script
frida -U -f com.target.app -l bypass.js --no-pause
```

### Objection Commands

```bash
objection -g com.target.app explore
# Inside:
android sslpinning disable
ios sslpinning disable
android root disable
ios jailbreak disable
env
```

---

## API Testing

| Tool           | Description                          |
|----------------|--------------------------------------|
| **Postman**    | API client                           |
| **Insomnia**   | API client (lighter)                 |
| **Burp Suite** | Proxy for API testing                |
| **mitmproxy**  | CLI proxy                            |
| **kiterunner** | API endpoint discovery               |
| **Arjun**      | Parameter discovery                  |
| **Autorize**   | Burp extension for BOLA/IDOR testing |
| **grpcurl**    | gRPC CLI client                      |
| **grpcui**     | gRPC web UI                          |
| **ghz**        | gRPC load testing                    |

### grpcurl Commands

```bash
# List services (if reflection enabled)
grpcurl -plaintext target.com:443 list

# Describe a service
grpcurl -plaintext target.com:443 describe target.UserService

# Call a method
grpcurl -plaintext -d '{"user_id":"123"}' \
  target.com:443 target.UserService.GetUser

# With auth header
grpcurl -plaintext -H 'authorization: Bearer X' \
  -d '{}' target.com:443 target.UserService.GetUser
```

---

## Exploitation Frameworks

| Tool              | Description                          |
|-------------------|--------------------------------------|
| **Metasploit**    | Exploitation framework               |
| **Cobalt Strike** | Commercial C2 (red team)             |
| **Sliver**        | Open-source C2                       |
| **Empire**        | Post-exploitation framework          |
| **sqlmap**        | SQL injection exploitation           |
| **BeEF**          | Browser exploitation framework       |
| **XSS Hunter**    | Blind XSS detection                  |
| **Responder**     | LLMNR/NBT-NS/MDNS poisoner           |
| **impacket**      | Python classes for network protocols |

---

## Proxy & Interception

| Tool                | Description                        |
|---------------------|------------------------------------|
| **Burp Suite Pro**  | Industry standard                  |
| **Burp Suite CE**   | Free version (limited)             |
| **Caido**           | Modern Burp alternative (free tier)|
| **mitmproxy**       | CLI + web interface                |
| **OWASP ZAP**       | Free, open-source proxy            |
| **Charles**         | macOS-friendly proxy               |
| **Fiddler**         | Windows-friendly proxy             |
| **Proxyman**        | macOS/iOS proxy                    |

### Burp Suite Essential Extensions

- **Autorize** — BOLA/IDOR detection
- **Param Miner** — Hidden parameter/header discovery
- **HTTP Request Smuggler** — CL.TE / TE.CL detection
- **JWT Editor** — JWT manipulation
- **InQL** — GraphQL testing
- **Turbo Intruder** — Advanced race conditions
- **Collaborator Everywhere** — Automatic OOB testing
- **Logger++** — Advanced logging
- **Retire.js** — JS library vulnerabilities
- **Nuclei** — Template scanning from Burp

---

## Encoding & Payload Generation

| Tool                        | Description                      |
|-----------------------------|----------------------------------|
| **CyberChef**               | Swiss army knife for encoding/decoding |
| **Burp Decoder**            | Built-in Burp encoder            |
| **PayloadsAllTheThings**    | Payload collection               |
| **SecLists**                | Wordlist collection              |
| **HackTricks**              | Technique reference              |
| **GTFOBins**                | Unix binary exploitation         |
| **LOLBAS**                  | Windows binary exploitation      |
| **revshells.com**           | Reverse shell generator          |
| **msfvenom**                | Payload generator                |

### Common Encoding Chains

```
URL encode:         %27
Double URL encode:  %2527
HTML entity:        &#39;
Unicode:            \u0027
Base64:             Jw==
Hex:                \x27
Octal:              \047
```

---

## Wordlists

### SecLists (The Essential Collection)

```bash
git clone https://github.com/danielmiessler/SecLists.git
```

**Key paths:**

```
Discovery/Web-Content/common.txt
Discovery/Web-Content/raft-large-directories.txt
Discovery/Web-Content/raft-large-files.txt
Discovery/DNS/subdomains-top1million-5000.txt
Discovery/DNS/subdomains-top1million-20000.txt
Passwords/Common-Credentials/10-million-password-list-top-1000.txt
Fuzzing/XSS/
Fuzzing/SQLi/
Fuzzing/LFI/
Usernames/
```

### Other Wordlists

| Wordlist                  | Description                     |
|---------------------------|---------------------------------|
| **Assetnote Wordlists**   | Curated by Assetnote            |
| **OneListForAll**         | Mega wordlist                   |
| **PayloadsAllTheThings**  | Payload lists                   |
| **Probable-Wordlists**    | Sorted by probability           |

---

## Cloud & OSINT

### Cloud Enumeration

| Tool                | Description                            |
|---------------------|----------------------------------------|
| **cloud_enum**      | Multi-cloud public resource enumeration|
| **s3scanner**       | S3 bucket scanner                      |
| **GCPBucketBrute**  | GCP bucket enumeration                 |
| **AzureHound**      | Azure AD enumeration                   |
| **Pacu**            | AWS exploitation framework             |
| **ScoutSuite**      | Multi-cloud security auditing          |
| **cloudfox**        | Cloud enumeration                      |
| **trufflehog**      | Secret scanning in git repos + buckets |

### OSINT

| Tool                | Description                       |
|---------------------|-----------------------------------|
| **theHarvester**    | Email, subdomain, host OSINT      |
| **sherlock**        | Username search across platforms  |
| **GHunt**           | Google account OSINT              |
| **mosint**          | Email OSINT                       |
| **holehe**          | Email registration checker        |
| **SpiderFoot**      | Automated OSINT                   |
| **maltego**         | Link analysis                     |
| **recon-ng**        | Recon framework                   |

### Search Engines

| Engine         | Description                            |
|----------------|----------------------------------------|
| **Shodan**     | Internet-connected devices             |
| **Censys**     | Internet-wide scan data                |
| **FOFA**       | Chinese internet intelligence          |
| **ZoomEye**    | Cyberspace search                      |
| **BinaryEdge** | Internet scanning                      |
| **GreyNoise**  | Internet noise (attacker IPs)          |
| **Netlas**     | Internet intelligence                  |
| **Onyphe**     | Cyber threat intelligence              |

---

## Reporting & Documentation

| Tool            | Description                     |
|-----------------|---------------------------------|
| **Markdown**    | Report format                   |
| **Loom**        | Video PoC                       |
| **OBS Studio**  | Screen recording                |
| **ShareX**      | Windows screenshot + GIF        |
| **Flameshot**   | Linux screenshot                |
| **Carbon**      | Beautiful code screenshots      |
| **Excalidraw**  | Diagrams                        |
| **draw.io**     | Diagrams                        |

### Report Templates

- [Bugcrowd Vulnerability Rating Taxonomy](https://bugcrowd.com/vulnerability-rating-taxonomy)
- [OWASP Vulnerability Disclosure Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html)
- [HackerOne Report Templates](https://docs.hackerone.com/programs/report-templates.html)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)

---

## Installation & Setup

### Kali Linux (Recommended)

Most tools above are pre-installed on Kali Linux.

```bash
sudo apt update && sudo apt install -y \
  nmap ffuf gobuster dirsearch sqlmap nuclei subfinder amass httpx
```

### Go Tools (ProjectDiscovery)

```bash
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
```

### Python Tools

```bash
pip install sqlmap arjun trufflehog
```

### Wordlists

```bash
# SecLists
git clone https://github.com/danielmiessler/SecLists.git

# PayloadsAllTheThings
git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git
```

### Docker

```bash
docker run -it --rm projectdiscovery/nuclei:latest -u https://target.com
docker run -it --rm projectdiscovery/subfinder:latest -d target.com
```

---

## Suggested Workflow

A typical bug bounty workflow using these tools:

```
1. subfinder -d target.com -o subs.txt
        ↓
2. httpx -l subs.txt -o live.txt
        ↓
3. katana -list live.txt -o urls.txt
        ↓
4. gau target.com >> urls.txt
        ↓
5. cat urls.txt | gf ssrf | sort -u > ssrf_urls.txt
        ↓
6. ffuf -w wordlist.txt -u https://target.com/FUZZ
        ↓
7. nuclei -l live.txt -severity critical,high -o nuclei.txt
        ↓
8. Manual testing in Burp on interesting endpoints
```

---

## Learning Resources

| Resource                                | Description              |
|-----------------------------------------|--------------------------|
| **PortSwigger Web Security Academy**    | Free, best-in-class labs |
| **HackTricks**                          | Technique reference      |
| **PayloadsAllTheThings**                | Payload collection       |
| **OWASP Cheat Sheets**                  | Best practices           |
| **YesWeHack Dojo**                      | Free labs                |
| **PentesterLab**                        | Labs (paid + free)       |
| **TryHackMe**                           | Beginner-friendly        |
| **HackTheBox**                          | Advanced labs            |
| **Bug Bounty Reports Explained**        | YouTube channel          |
| **STÖK / NahamSec / InsiderPhD**        | YouTube channels         |

---

## Legal Reminder

Use these tools only on systems you **own** or are **explicitly authorized**
to test. Many tools are powerful enough to cause damage if misused. Always:

- Read the program's scope.
- Respect rate limits.
- Never test on production with real user data.
- Follow responsible disclosure.

---

> **Back to:** [README](../README.md)
> **Prev:** [Google Dorks ←](dorks.md)
