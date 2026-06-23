# Business Logic Vulnerabilities — Key Learnings

**Status:** 11/11 labs completed ✅

---

## What is it?

Business logic vulnerabilities are flaws in the **design and implementation of an application's intended workflow** — not in the code syntax or missing sanitization, but in the assumptions developers make about how users will interact with the app. The app works exactly as coded; the problem is the code doesn't account for edge cases, unexpected sequences, or values that fall outside what the developer imagined a "normal" user would do.

## Why it exists in real apps?

Developers build features thinking about the happy path — a user adds a product, pays a positive amount, checks out. They rarely think: what if quantity is negative? What if someone applies the same coupon twice in rapid succession? What if you skip step 3 of a 4-step checkout? These gaps aren't caught by standard security scanners because there's no "injection" happening — the app is doing exactly what it was told, just with inputs the developer never expected.

## Most interesting lab + what I learned

**Low-level logic flaw** was the trickiest one mechanically. The idea: keep adding products in large quantities until the total price overflows past the maximum integer value and wraps around to a negative number. Now your cart total is negative — you're owed money. The challenge is getting it back into a range your account balance can cover — which means adding another cheap product in just the right quantity to bring the negative total back up to something small and positive that fits within your available funds.

What made this genuinely interesting: it required Burp Intruder running many iterations to find the exact quantity that tips the price over, then careful math to land in the right range. It's not a one-shot exploit — it's iterative, and you're essentially doing arithmetic with overflow behavior.

**Key insight:** developers validate that quantity is a positive number, but they don't think about what happens when the *total* crosses an integer boundary. The check happens at the wrong level.

**Infinite money logic flaw** was the most eye-opening in terms of real-world applicability. The vulnerability: a discount coupon can be applied, a product purchased, and then refunded — but the coupon isn't invalidated after the refund. So the cycle repeats: apply coupon → buy → refund → repeat. Each loop nets you a small profit. Run it enough times and you have enough balance to buy anything.

The technical tool here was **Burp Macros** — first time I actually used it properly. The concept: record a sequence of requests (apply coupon → add to cart → checkout → refund), then set up Burp to loop that sequence automatically. What would take hours manually runs in seconds. This is directly applicable to race condition testing, multi-step flow abuse, and any scenario where you need to repeat a request sequence rapidly.

## The common pattern across all 11 labs

Business logic bugs all come from the same root: **the application trusts that users will follow the intended flow and provide expected values.**

Patterns I kept seeing:

- **Client-side trust** — price, quantity, discount value sent from the browser and accepted by the server without server-side recalculation. Change the value in Burp, server accepts it.
- **Missing bounds checking** — no validation on minimum values (negative quantity), no overflow protection on totals.
- **State not tracked properly** — coupons that can be reused, workflow steps that can be skipped, actions that can be repeated because the server doesn't remember what already happened.
- **Dual-use endpoints** — one endpoint handles both user and admin actions based on a parameter. Remove the parameter restriction and a normal user triggers an admin action.
- **Trusting step sequence** — multi-step flows where step 2 doesn't verify step 1 actually completed, or where skipping directly to step 3 works fine.

## Real-world bug bounty angle

Business logic bugs are **underreported and undervalued by beginners** because they're not flashy — no XSS payload, no SQLi syntax. But they're consistently found in real programs because:

- Automated scanners don't find them. At all. This is purely manual testing territory.
- Every e-commerce app, subscription service, or financial platform has complex business rules — and complex rules have gaps.
- High impact: price manipulation, unauthorized access to paid features, infinite balance exploits — these are direct financial impact bugs, which programs take seriously.

**Hunting approach:** for any app with transactions, coupons, subscriptions, or multi-step flows — map the entire workflow first, then ask: what if I repeat this step? What if I skip that step? What if this value is negative, zero, or extremely large? What if I do two things simultaneously that the app expects to happen sequentially?

## Tools used

- Burp Suite Intruder — iterating quantity values to trigger integer overflow in low-level logic flaw
- Burp Suite Macros — automating request sequences for infinite money exploit (apply coupon → buy → refund loop)
- Burp Suite Repeater — manual flow manipulation across all labs
