# ⚡ Race Conditions (Tricks #14–#17)

> ⚠️ **Disclaimer:** Educational use only. Only test in-scope assets you are
> authorized to test. Never touch systems without explicit written permission.

Race conditions are one of the most underrated bug classes. They pay well
because they often lead to **financial loss** or **privilege escalation**, and
most programs have never tested for them. The tricks below cover the modern
single-packet technique and the highest-impact race patterns.

---

## Table of Contents

1. [Single-Packet Attack for Reliable Race Conditions](#trick-14)
2. [Coupon Reuse via Parallel Redemption](#trick-15)
3. [Multiple Withdrawals via TOCTOU on Balance Checks](#trick-16)
4. [Bypass Rate Limits with HTTP/2 Multiplexing](#trick-17)

---

<a name="trick-14"></a>
## 🧩 Trick #14 — Single-Packet Attack for Reliable Race Conditions

**Category:** Race Condition
**Impact:** Critical
**Tools:** Burp Suite (Repeater), Turbo Intruder

### The Idea
Traditional race conditions are unreliable because network jitter spreads
your requests across milliseconds. The **single-packet attack** bundles 20–30
HTTP/2 requests into **one TCP packet**, so the server receives them
simultaneously — removing jitter entirely.

### Steps (Burp Suite)

1. **Capture the target request** (e.g., coupon apply, withdraw, vote).

2. **Send to Repeater.**

3. **Duplicate the tab 20–30 times:**
   - Right-click the tab → **Duplicate tab**
   - Or select multiple tabs → **Duplicate**

4. **Group the tabs:**
   - Select all duplicated tabs
   - Right-click → **Add to new group** (or drag into a group)

5. **Send in parallel:**
   - Right-click the group → **Send group in parallel (single-packet attack)**

6. **Analyze responses:**
   - Count how many succeeded
   - If N > 1, you have a race condition

### Steps (Turbo Intruder — more control)

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    for i in range(30):
        engine.queue(target.req, gate='race1')
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

Key points:
- `concurrentConnections=1` → one TCP connection
- `engine=Engine.BURP2` → HTTP/2 with single-packet support
- `openGate('race1')` → release all requests at once

### Why It Works
HTTP/2 multiplexing lets many requests share one connection. By holding them
and releasing simultaneously, the server processes them within microseconds.

### Fix
- Use database transactions with proper isolation.
- Atomic check-and-act (e.g., `UPDATE ... WHERE balance >= X`).
- Row-level locks (`SELECT ... FOR UPDATE`).
- Idempotency keys per user action.

### References
- [PortSwigger: Single-packet attack](https://portswigger.net/research/smashing-the-state-machine)

---

<a name="trick-15"></a>
## 🧩 Trick #15 — Coupon Reuse via Parallel Redemption

**Category:** Race Condition
**Impact:** High (financial loss)
**Tools:** Burp Repeater (single-packet), Turbo Intruder

### The Idea
Coupons are usually redeemed with a check-then-write pattern:
```python
if not coupon.used:
    coupon.used = True
    apply_discount()
```
Fire 20 parallel requests and the check passes for all of them **before** any
write completes. Result: multi-applied discount.

### Steps

1. **Capture the "apply coupon" request:**
   ```
   POST /cart/coupon HTTP/2
   Host: target.com

   code=SAVE20
   ```

2. **Send to Repeater and duplicate 20–30×.**

3. **Group them and use single-packet attack.**

4. **Check the cart total:**
   - If discount applied N times → race confirmed.

5. **Escalate:**
   - Try `amount` parameter with negative values in parallel
   - Try stacking different coupons

### Why It Works
- Application-level checks aren't atomic.
- Database writes happen after the check.
- No unique constraint on `(user_id, coupon_id)`.

### Fix
- Atomic apply: `UPDATE coupons SET used=1 WHERE code=? AND used=0` — check
  `rowcount == 1`.
- Unique constraint on `(user_id, coupon_id)`.
- Idempotency key per cart.

---

<a name="trick-16"></a>
## 🧩 Trick #16 — Multiple Withdrawals via TOCTOU on Balance Checks

**Category:** Race Condition (TOCTOU)
**Impact:** Critical (direct financial loss)
**Tools:** Burp Repeater, Turbo Intruder

### The Idea
**TOCTOU** = Time-of-Check to Time-of-Use. If the app:
1. Reads balance
2. Checks `balance >= amount`
3. Sends withdrawal
4. Updates balance

...then firing N parallel withdrawals of the full balance can drain N × balance.

### Steps

1. **Find the withdrawal endpoint:**
   ```
   POST /api/withdraw HTTP/2
   Host: target.com

   amount=1000&to_account=attacker
   ```

2. **Send to Repeater, duplicate 10×.**

3. **Send with single-packet attack.**

4. **Check balance and transaction history:**
   - Multiple withdrawals may be pending simultaneously.

5. **Escalate:**
   - Try the max amount multiple times
   - Try interleaving with a transfer out

### Why It Works
- Balance check and balance deduction are separate operations.
- No pessimistic lock on the account row.
- Message queue may process withdrawals asynchronously.

### Fix
- Pessimistic lock: `SELECT balance FROM accounts WHERE id=? FOR UPDATE`
- Atomic deduction: `UPDATE accounts SET balance = balance - ? WHERE id=? AND balance >= ?`
- Serializable isolation level.
- Idempotency key per withdrawal request.

---

<a name="trick-17"></a>
## 🧩 Trick #17 — Bypass Rate Limits with HTTP/2 Multiplexing

**Category:** Race Condition / Rate Limit Bypass
**Impact:** High
**Tools:** Burp Repeater, `h2c`, custom scripts

### The Idea
Many rate limiters count **per connection**, not per request. HTTP/2
multiplexing sends many requests over one connection — and the limiter
doesn't count them separately.

### Steps

1. **Identify the rate limit:**
   - Fire 10 requests → observe 429 after N.
   - Note the threshold (e.g., 5/min).

2. **Send 100 requests over one HTTP/2 connection:**
   - Burp Repeater: duplicate tab → group → send sequentially on one conn.
   - Or use Turbo Intruder with `concurrentConnections=1`.

3. **Observe:**
   - If all 100 succeed → limiter is connection-based.
   - If blocked → limiter may be per-request or per-IP.

4. **Combine with header tricks:**
   - Rotate `X-Forwarded-For`
   - Rotate `X-Real-IP`
   - Rotate query param `?_=timestamp`

### Why It Works
Rate limiters often live at the edge (nginx, Cloudflare) and count TCP
connections or apply per-connection buckets. HTTP/2 breaks that model.

### Fix
- Count **every request** atomically (Redis `INCR` per user+action).
- Don't trust `X-Forwarded-For` from untrusted sources.
- Use distributed rate limiting (e.g., Redis, Envoy rate-limit filter).
- Separate limit buckets per endpoint, not per connection.

---

## 📌 Quick Reference Summary

| #  | Trick                         | Impact   | Effort |
|----|-------------------------------|----------|--------|
| 14 | Single-packet attack          | Critical | Low    |
| 15 | Coupon reuse                  | High     | Low    |
| 16 | Withdrawal TOCTOU             | Critical | Medium |
| 17 | HTTP/2 rate limit bypass      | High     | Low    |

---

> **Prev:** [SSRF & Cloud ←](02-ssrf-cloud.md)
> **Next:** [Client-Side Attacks →](04-client-side.md)
