# 💼 Business Logic (Tricks #30–#34)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Business logic bugs are the most underrated class in bug bounty — scanners
never find them, most hunters skip them, and they regularly pay **Critical**
because they cause direct financial loss or account takeover. They require
understanding how the app is *supposed* to work, then doing exactly what it
shouldn't allow.

---

## Table of Contents

1. [Price Manipulation via Demand-Based Logic](#trick-30)
2. [Negative Quantity / Amount Injection in Carts](#trick-31)
3. [Bypass Purchase Limits via Parameter Tampering](#trick-32)
4. [Free Trial Abuse via Email Alias Tricks](#trick-33)
5. [Currency Confusion in Multi-Currency Checkouts](#trick-34)

---

<a name="trick-30"></a>
## 🧩 Trick #30 — Price Manipulation via Demand-Based Logic

**Category:** Business Logic
**Impact:** Critical (financial)
**Tools:** Burp, custom scripting

### The Idea
Some apps raise prices based on demand — bookings, limited inventory,
dynamic pricing. If the demand counter can be manipulated **without paying**,
you can force prices up or down on a large scale.

### Attack Patterns

**1. Inflate demand (denial of revenue):**
- Initiate many bookings but never complete payment.
- Inventory locks in "reserved" state.
- Real users see inflated prices.
- Business loses sales.

**2. Deflate demand (cheap purchase):**
- Race to fill demand slots, then cancel.
- Price resets to base.
- Buy at the low price.

**3. Race the price lock:**
- Add item to cart at price A.
- Trigger a price change.
- Checkout uses stale price A vs. new price B.

### Steps

1. **Identify dynamic pricing:**
   - Search for `price`, `demand`, `dynamic`, `surge` in JS.
   - Watch prices change as you browse.

2. **Test demand manipulation:**
   ```bash
   for i in {1..50}; do
     curl -X POST https://target.com/api/book -d '{"slot":"2026-11-01"}' &
   done
   wait
   ```

3. **Watch price after flooding the endpoint.**

4. **Confirm:**
   - Screenshot before/after prices.
   - Document the business impact.

### Why It Works
- Inventory reserves are non-atomic.
- Demand counters aren't tied to payment.
- Price recalculation happens server-side but based on user actions.

### Fix
- Tie inventory reservation to a payment intent with TTL.
- Rate limit booking creation per user/IP.
- Recompute price at checkout (never trust cart price).
- Detect anomalous booking patterns.

---

<a name="trick-31"></a>
## 🧩 Trick #31 — Negative Quantity / Amount Injection in Carts

**Category:** Business Logic
**Impact:** Critical (financial)
**Tools:** Burp

### The Idea
If the cart total is calculated as `sum(price × quantity)`, a negative
quantity can subtract from the total — sometimes making the order free or
even generating a "refund" to the attacker.

### Payloads

```json
{"item_id": 123, "quantity": -1}
{"item_id": 123, "quantity": -100}
{"item_id": 123, "price": -50.00}
{"item_id": 123, "amount": "-10.00"}
{"item_id": 123, "quantity": 1e10}
{"item_id": 123, "quantity": 0.0001}
{"item_id": 123, "quantity": "1; DROP"}
{"item_id": 123, "quantity": true}
{"item_id": 123, "quantity": []}
{"item_id": 123, "quantity": null}
```

**Add a negative-price item, then a normal item:**
```
cart = [
  {item: expensive_thing, qty: 1},
  {item: cheap_thing,    qty: -1}
]
```

**Split across multiple line items:**
```json
{"items":[
  {"id":"A","qty":1,"price":100},
  {"id":"A","qty":-1,"price":-100}
]}
```

### Steps

1. **Add an item to cart.**

2. **Intercept the update request:**
   ```
   POST /cart/update
   {"item_id":123,"quantity":1}
   ```

3. **Change quantity to negative.**

4. **Check the cart total.**

5. **Try to check out** — if total is ≤ 0, does it succeed?

6. **Escalate:**
   - Stack negatives with positives
   - Try decimal quantities (0.0001)
   - Try exponential (1e10)
   - Try type confusion (`"1"`, `true`, `[]`)

### Why It Works
- Backend trusts client-supplied `quantity`.
- No validation for `> 0`.
- Total calculation is naive summation.
- Type coercion bugs (`true` → 1, `[]` → 0).

### Fix
- Validate `quantity > 0` and `< max` server-side.
- Reject non-integer quantities (or enforce decimal precision).
- Recompute line items from the product catalog — never trust client prices.
- Reject unknown types (strict JSON schema).

---

<a name="trick-32"></a>
## 🧩 Trick #32 — Bypass Purchase Limits via Parameter Tampering

**Category:** Business Logic
**Impact:** High
**Tools:** Burp

### The Idea
Products "limited to 1 per user" often enforce the limit client-side or via a
parameter the attacker controls (`userId`, `email`, `deviceId`). Tamper with
that parameter and the limit resets.

### Attack Patterns

**1. User ID rotation:**
```
POST /cart/add
{"product_id":123,"quantity":1,"user_id":"victim"}
POST /cart/add
{"product_id":123,"quantity":1,"user_id":"victim2"}
```

**2. Email alias rotation:**
```
victim+1@example.com
victim+2@example.com
v.i.c.t.i.m@example.com
```

**3. Header rotation:**
```
X-Forwarded-For: 1.2.3.4
X-Real-IP: 1.2.3.5
X-Device-Id: abc123
```

**4. Cookie rotation:**
- Clear session cookie between adds
- Use incognito for each add

**5. Array parameter duplication:**
```
POST /cart/add
product_id=123&product_id=123&product_id=123
```

**6. JSON array:**
```json
{"items":[{"id":123,"qty":1},{"id":123,"qty":1}]}
```

**7. Multiple checkout attempts:**
- Add to cart once
- Add again immediately after checkout (before limit resets)
- Race the limit reset

### Steps

1. **Find a limited product** (limited edition, one-per-customer, sale item).

2. **Attempt to add twice** — observe the error.

3. **Tamper with parameters** listed above.

4. **Confirm bypass** by checking quantity in cart / order.

### Why It Works
- Limit is tracked by a client-controlled identity.
- Race between "check limit" and "add to cart".
- Duplicate parameters are concatenated server-side.

### Fix
- Enforce limits server-side per authenticated user.
- Use database unique constraint on `(user_id, product_id)`.
- Ignore client-supplied `user_id`.
- Detect duplicate params (reject or process only the first).
- Rate limit cart operations per session.

---

<a name="trick-33"></a>
## 🧩 Trick #33 — Free Trial Abuse via Email Alias Tricks

**Category:** Business Logic
**Impact:** Medium
**Tools:** Burp, multiple mailboxes

### The Idea
Free trials are limited by email. But email providers treat some aliases as
equivalent — and apps often forget to normalize them. You can register the
same person multiple times and get unlimited free trials.

### Alias Tricks

**Gmail:**
```
user@gmail.com
user+1@gmail.com
user+2@gmail.com
u.s.e.r@gmail.com       (dots ignored)
us.er@gmail.com
```

**Outlook / Hotmail:**
```
user@outlook.com
user+1@outlook.com
u.s.e.r@outlook.com     (dots NOT ignored — but some apps assume they are)
```

**Custom domains (catch-all):**
```
anything@yourdomain.com
```

**Disposable email services:**
- `mail.tm`, `temp-mail.org`, `guerrillamail.com`
- Rotate addresses automatically

**Gmail dots hack:**
- `u.s.e.r@gmail.com` and `user@gmail.com` are **the same inbox**.
- Many apps treat them as different users.

### Steps

1. **Sign up for a free trial** with `user@gmail.com`.

2. **Try again** with `u.s.e.r@gmail.com` → same inbox.

3. **Try again** with `user+1@gmail.com`.

4. **Try again** with a catch-all domain you control.

5. **Document:** N trials for the same person → revenue loss.

### Why It Works
- Apps don't normalize emails.
- Apps trust the email local-part as unique.
- Catch-all domains give unlimited aliases.

### Fix
- **Normalize emails on input:**
  - Lowercase
  - Strip Gmail dots
  - Strip `+tag`
- Block known disposable email providers.
- Require verified payment method (card fingerprint) for trial.
- Device fingerprinting for trial eligibility.

---

<a name="trick-34"></a>
## 🧩 Trick #34 — Currency Confusion in Multi-Currency Checkouts

**Category:** Business Logic
**Impact:** High
**Tools:** Burp

### The Idea
Multi-currency apps sometimes update the currency **display** but not the
**price** server-side — or vice versa. An attacker can buy in a weak currency
at a strong currency's numeric value.

### Attack Patterns

**1. Currency switch mid-checkout:**
- Add item to cart (USD $100)
- Change currency to JPY
- Checkout — if server uses `100` as JPY → you pay ~$0.67

**2. Parameter tampering:**
```
POST /checkout
amount=100&currency=USD
```
→
```
POST /checkout
amount=100&currency=JPY
```

**3. Currency code swapping:**
```
POST /cart
{"price":100,"currency":"USD"}
```
→
```
POST /cart
{"price":100,"currency":"JPY"}
```

**4. Rounding abuse:**
- Currency with 0 decimal places (JPY) vs. 2 (USD).
- Buy many cheap items; rounding errors accumulate.

**5. Case sensitivity:**
```
currency=usd
currency=UsD
currency=USD
```
→ server might treat these as different currencies.

**6. Symbol vs. code:**
```
currency=$
currency=USD
```

### Steps

1. **Add item in a strong currency** (USD, EUR, GBP).

2. **Intercept the checkout request.**

3. **Try changing:**
   - `currency` param
   - `amount` numeric value
   - Case of currency code

4. **Observe the charged amount** (via payment provider sandbox or bank).

5. **Escalate:**
   - Stack with coupon
   - Combine with negative quantity

### Why It Works
- Currency is a client-supplied parameter.
- Server stores `amount` independently of `currency`.
- Display currency and charge currency are decoupled.

### Fix
- Store product price in a single base currency.
- Convert on the server at checkout time using current rate.
- Lock currency at cart creation.
- Reject client-supplied `amount` — always compute from products.

---

## 📌 Quick Reference Summary

| #  | Trick                        | Impact   | Effort |
|----|------------------------------|----------|--------|
| 30 | Demand-based price abuse     | Critical | Medium |
| 31 | Negative quantity / amount   | Critical | Low    |
| 32 | Purchase limit bypass        | High     | Low    |
| 33 | Free trial abuse via aliases | Medium   | Low    |
| 34 | Currency confusion           | High     | Low    |

---

> **Prev:** [Auth & Session ←](05-auth-session.md)
> **Next:** [Cache & Infrastructure →](07-cache-infra.md)
