# Race Conditions — Key Learnings

**Status:** 6/6 labs completed ✅

---

## What is it?

Race conditions occur when an application performs a sequence of operations that depend on a shared state, and an attacker sends multiple requests simultaneously to exploit the window between those operations. The server processes requests in parallel, and if it doesn't properly handle concurrent access, it can act on a state that's already been changed by another request — leading to unintended behavior.

## Why it exists in real apps?

Most developers think sequentially — request comes in, server processes it, responds. They don't account for what happens when 50 requests arrive at the exact same millisecond. Checks that work perfectly in sequence (verify balance → deduct balance → confirm purchase) break under concurrency because two requests can both pass the "verify balance" check before either one deducts anything. This is called a **time-of-check to time-of-use (TOCTOU)** flaw.

## Most interesting labs + what I learned

**Single packet attack** was the biggest technical insight from this entire topic. The problem with sending concurrent requests normally is network jitter — even if you fire 50 requests simultaneously from your machine, they arrive at the server milliseconds apart due to network variance. That gap is enough for the server to process them sequentially.

The single packet attack solves this: bundle multiple HTTP/2 requests into a single TCP packet. Since they're in one packet, they arrive at the server at the exact same time — truly simultaneous. Burp Suite's "Send group in parallel (single-packet attack)" feature does this. The result: the server receives all requests in the same instant and has to process them concurrently, triggering the race window.

**Multi-endpoint race condition** — the scenario: add an expensive product to cart, then simultaneously send the "add cheap product" request and the "checkout" request in parallel. The server checks cart contents and processes payment concurrently. Because the checkout request reads the cart state before the cheap-product swap completes (or after, depending on timing), you can end up paying the cheap product's price for the expensive one. The server returned 200 because from its perspective, both operations were valid — it just processed them in an order that wasn't intended.

**Bypassing rate limits via race conditions** — rate limiting typically works by: check attempt count → if under limit, allow → increment count. Under concurrent requests, multiple requests can all pass the "check attempt count" step before any of them increment the counter. So 50 login attempts can all be counted as "attempt #1." Rate limit never triggers.

**Limit overrun** — the classic: a single-use coupon that can be used multiple times by sending concurrent redemption requests. Each request checks "has this coupon been used?" simultaneously, all see "no," all apply it. The coupon gets applied 10-20 times before any request marks it as used.

## The common pattern across all 6 labs

Every race condition lab had the same structure: **a check and an action that should be atomic, but aren't.**

- Check happens → action happens → state updates
- If two requests both hit the check before either updates the state, both pass
- The fix is always atomicity — database transactions, locks, or atomic operations that make check+action happen as one indivisible unit

What makes race conditions genuinely hard to find: they're timing-dependent. The same request that exploits a race condition might fail 9 out of 10 times. You need to send enough concurrent requests to reliably hit the window, and the single-packet attack is what makes this consistent.

## Real-world bug bounty angle

Race conditions are **underreported** in bug bounty because most people don't test for them — they require thinking about concurrency, not just input manipulation. This means less competition on programs that have them.

High-value targets: any feature involving limits (coupons, referral bonuses, free trial activations, vote counts, rate-limited APIs), any multi-step transaction (add to cart → pay → confirm), any single-use token (password reset links, email verification codes).

**Hunting approach:** identify any endpoint with a "check then act" pattern. Use Burp's parallel send (single-packet attack for HTTP/2, last-byte sync for HTTP/1). Send 20-50 concurrent requests. Look for responses that indicate the check passed multiple times — duplicate transactions, negative balances, limit overruns.

## Tools used

- Burp Suite Repeater — "Send group in parallel (single-packet attack)" for true concurrent requests
- Burp Suite Intruder — sending high volumes of concurrent requests for limit overrun testing
- Manual analysis — understanding the check-then-act sequence in each endpoint before exploiting
