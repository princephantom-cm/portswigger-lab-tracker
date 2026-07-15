# Cross-Site Request Forgery (CSRF) — Key Learnings

**Status:** 8/8 labs completed ✅

---

## What is it?

CSRF tricks a victim's browser into making an authenticated request to a target application without the victim's knowledge. The browser automatically attaches cookies with every request — including session cookies — so if an attacker can get the victim to load a page with a crafted form or request, that request goes to the server with the victim's full authentication attached.

## Why it exists in real apps?

The web's cookie model is the root cause — browsers send cookies automatically, regardless of which site initiated the request. CSRF defenses (tokens, SameSite cookies) were added later as patches to this fundamental behavior. Implementations are inconsistent, partial, or bypassed because developers add a token check but don't think through all the ways that check can be weak.

## Most interesting labs + what I learned

**The core insight across all labs:** every CSRF defense is only as strong as what the server actually validates. Adding a CSRF token field to a form doesn't mean the server properly validates it — it might check that the token *exists* without checking it belongs to *this user's session*, or it might check that token and cookie *match each other* without verifying either is legitimate.

This is where the most interesting bypasses came from:

**Token not tied to user session** — the server had a pool of valid CSRF tokens. It checked that the submitted token was *a* valid token, not that it was *the* token issued to *this* session. So you could log in with your own account, grab your CSRF token, and use it in an attack against a different user. The token validates, attack succeeds.

**Token tied to non-session cookie (double submit pattern)** — the server checked that `csrf` parameter matched the `csrfKey` cookie value. Both had to match — but they didn't have to be tied to any real user session. So `csrf=anything` + `csrfKey=anything` worked, as long as both values were identical. Attacker sets both to the same fake value, server says valid.

**Burp's "Generate CSRF PoC"** — under Engagement Tools in Burp, this auto-generates an HTML page with a form that replicates the target request. You open it in a browser while logged out, the victim opens it while logged in — their browser submits the form with their session attached. This tool saves significant time when crafting the actual attack payload.

**SameSite and Referer bypasses** — some labs had no CSRF token at all but relied on SameSite cookie attributes or Referer header validation. SameSite=Lax has a gap: it allows cookies on top-level GET navigations, so a GET request that triggers a state change bypasses Lax protection. Referer validation can be bypassed by hosting the PoC on a URL that contains the target domain as a substring, or by stripping the Referer header entirely (some servers allow requests with no Referer).

## The common pattern across all 8 labs

CSRF defenses fail when:
- **Token exists but isn't validated** — server checks presence, not correctness
- **Token validated but not session-bound** — any valid token works for any user
- **Double submit without real binding** — matching values accepted regardless of origin
- **SameSite misconfigured** — Lax allows GET state changes, None without Secure is exploitable
- **Referer check bypassable** — substring match or missing header accepted

The strongest CSRF defense is SameSite=Strict + a session-bound token validated server-side. Most real apps have one or the other, not both, which leaves gaps.

## Real-world bug bounty angle

CSRF findings are common on:
- Account settings pages (email change, password change, notification preferences)
- Any state-changing GET request (logout, delete, follow/unsubscribe)
- Forms that use predictable or improperly validated tokens
- Apps that haven't fully migrated to SameSite=Strict

**Hunting approach:** for every state-changing request — remove the CSRF token entirely, try a fake value, try your own token on another user's action, try changing POST to GET. Use Burp's Generate CSRF PoC to quickly build test payloads. Check SameSite cookie attributes in response headers — if it's Lax or missing, GET-based state changes are worth testing.

## Tools used

- Burp Suite Repeater — token manipulation, method switching, header removal
- Burp Suite Engagement Tools → Generate CSRF PoC — auto-generating HTML attack pages
- Manual analysis — reading token validation logic from server responses
