# Access Control & IDOR — Key Learnings

**Status:** 13/13 labs completed ✅

---

## What is it?

Access control vulnerabilities occur when an application fails to properly verify whether the user making a request is actually allowed to perform that action or access that resource. This covers everything from unprotected admin panels to a normal user escalating themselves to admin just by tampering with a request.

## Why it exists in real apps?

In most cases, access control is implemented at the UI level only — a button is hidden, a menu item doesn't render — but the underlying endpoint has no server-side check. Developers assume "if the user can't see it, they can't reach it." That assumption breaks the moment someone sends a raw request directly to the endpoint, bypassing the UI entirely.

## Most interesting lab + what I learned

**Referer-based access control** was the most interesting one for me. I hadn't thought about the `Referer` header as a security mechanism before — it tells the server which page the user navigated from. Some apps use this to decide whether a request "came from" an authorized internal page (like an admin panel link) before allowing access to a sensitive action.

The problem: the `Referer` header is fully attacker-controlled. The browser sends whatever the client sets it to. So if access control logic is "allow this action only if Referer contains `/admin`", you can just set that header yourself and walk straight through.

**Key takeaway:** any security decision based on a header the client controls (Referer, X-Forwarded-For, custom headers) is not a security decision — it's a suggestion.

## Where I had to think the most

**Method-based access control** and **Multi-step process with no access control on one step** felt similar in spirit — both weren't hard once I saw the pattern, but both revolved around the same core idea:

The classic example is the user role escalation flow — a normal user (`wiener`) finds a way to convert themselves into an admin by exploiting a gap in how the app checks permissions across a multi-step action or across different HTTP methods. If step 1 of a process is protected but step 2 isn't, or if `GET /admin/users` is blocked but `POST /admin/users` with the same parameters isn't — the access control is effectively broken, because attackers don't have to follow the "intended" path.

## The common pattern across all 13 labs

If I had to summarize all 13 labs in one sentence:

**Access control is almost never broken at the place where the data lives — it's broken at the edges: the parameter that decides "which user", the header that decides "where you came from", the HTTP method that decides "what action", or the step in a sequence that "nobody thought to check again."**

Specific patterns I kept seeing:
- **Trusting client-supplied identifiers** — `user_id`, `role`, or similar values sent in the request and trusted blindly by the server (horizontal → vertical escalation).
- **Checking the wrong thing** — validating that a user is *logged in*, but not validating that they're *allowed to do this specific thing*.
- **Inconsistent enforcement** — one entry point to a resource is protected, but an alternate path (different method, different URL casing, different step in a flow) isn't.
- **Security through obscurity** — hiding an admin panel behind an unguessable URL instead of actually checking permissions. Works until someone finds the URL (via `robots.txt`, JS files, etc.)

## Real-world bug bounty angle

Access control issues are consistently among the **highest-frequency findings** on bug bounty programs because:
- They require no special tools — just Burp Suite and careful observation of how requests change between user roles.
- Every application has multiple roles (user/admin, free/paid, owner/non-owner of a resource) — every role boundary is a potential test point.
- IDOR specifically is everywhere: any endpoint that takes an ID (`/api/orders/1234`, `/invoice/5678`) is worth testing by changing the ID to something that belongs to another user.

**My approach going forward:** for every endpoint I find during recon, I'll ask three questions — *(1) does this check who I am, (2) does this check what I'm allowed to do, and (3) is there another way to reach the same functionality that skips check #1 or #2?*

## Tools used

- Burp Suite (Repeater for request tampering, Proxy for intercepting requests)
- Manual testing — most of this category is observation-driven, not tool-driven
