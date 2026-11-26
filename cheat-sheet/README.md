# 📚 Cheat Sheet Index

All entries are standalone Markdown quick references. Use them only on systems you are explicitly authorized to test.

> Real flags, API keys and credentials are intentionally not stored in this repository. Examples use placeholders such as `<TARGET>`, `<USER>` and `<TOKEN>`.

---

## 🧱 Core vulnerabilities

| Cheat sheet | Focus |
|-------------|-------|
| [IDOR.md](./IDOR.md) | Object-level authorization, predictable IDs, ownership checks, Commnet case study |
| [XSS.md](./XSS.md) | Reflected, stored and DOM XSS; contexts, bypasses, CSP |
| [sql-injection.md](./sql-injection.md) | Union, error, boolean and time-based SQLi across DBMSes |
| [Auth-Bypass.md](./Auth-Bypass.md) | Login logic, token handling, verification flaws, default creds |
| [File-Upload.md](./File-Upload.md) | Extension/content-type checks, polyglots, path traversal, webshells |

## 🎯 Common and critical

| Cheat sheet | Focus |
|-------------|-------|
| [SSRF.md](./SSRF.md) | URL schemes, SSRF to internal services, cloud metadata, blind OOB |
| [OS-CMD-Injection.md](./OS-CMD-Injection.md) | Separators, time-based detection, encoding and filter bypasses |
| [NoSQLi.md](./NoSQLi.md) | MongoDB operators, auth bypass, blind extraction |
| [JWT.md](./JWT.md) | `alg:none`, algorithm confusion, key IDs, JWK injection |
| [Business-Logic.md](./Business-Logic.md) | Price manipulation, workflows, races, coupon/reuse abuse |

## 🧩 Advanced and API

| Cheat sheet | Focus |
|-------------|-------|
| [GraphQL.md](./GraphQL.md) | Introspection, aliases/batching, auth flaws, injections |
| [Prototype-Pollution.md](./Prototype-Pollution.md) | `__proto__`, `constructor.prototype`, gadgets and sinks |
| [Request-Smuggling.md](./Request-Smuggling.md) | CL.TE / TE.CL / TE.TE desync and front/back-end differences |
| [Cache-Poisoning.md](./Cache-Poisoning.md) | Unkeyed inputs, cache keys, web cache deception case study |
| [CORS.md](./CORS.md) | Origin reflection, credentials, null origin, preflight flaws |

## 🛠️ Additional

| Cheat sheet | Focus |
|-------------|-------|
| [Clickjacking.md](./Clickjacking.md) | Framing, UI redress, XFO/CSP frame ancestors |
| [XXE.md](./XXE.md) | XML entities, local file read, SSRF, blind OOB, DoS |
| [CSRF.md](./CSRF.md) | Token validation, SameSite, state-changing requests |
| [WebSockets.md](./WebSockets.md) | Handshake auth, message injection, CSWSH, rate limits |
| [Host-Header.md](./Host-Header.md) | Virtual hosts, reset poisoning, proxy header trust, cache keys |

---

## ✅ Status

- 20 / 20 vulnerability cheat sheets populated
- 2 / 20 were populated originally (SQL Injection, NoSQL Injection)
- 18 / 20 added in the completion pass

---

**🛡️ Happy authorized hacking!**
