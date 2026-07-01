# Server-Side Request Forgery (SSRF) — Key Learnings

**Status:** 7/7 labs completed ✅

---

## What is it?

SSRF is a vulnerability where an attacker can make the **server** send HTTP requests to unintended locations — internal systems, localhost, cloud metadata endpoints, or other back-end services that are otherwise not accessible from the internet. The core idea: you're not attacking the server directly, you're making the server attack itself or its internal network on your behalf.

## Why it exists in real apps?

Modern applications frequently fetch external resources — loading a URL preview, fetching a webhook, importing data from a link, checking stock from a supplier's API. Developers often don't think about what happens when that URL isn't "external" at all, but points inward. The server trusts itself, so when it receives a request from `127.0.0.1`, internal admin panels and APIs that are firewalled from the internet respond freely.

## Most interesting lab + what I learned

**SSRF with whitelist-based input filter** was the most interesting one. The difference between blacklist and whitelist-based filters is fundamental:

- **Blacklist:** Block known bad inputs (`127.0.0.1`, `localhost`, `169.254.169.254`). Problem: there are too many ways to represent the same thing — decimal IP, octal, hex, URL encoding, IPv6 `[::1]`, DNS that resolves to localhost, etc. Blacklists always lose this game.
- **Whitelist:** Only allow specific approved values (e.g. `stock.weliketoshop.net`). Stronger in theory — but it relies entirely on how the server **parses** the URL.

The bypass here exploited URL parsing confusion. Most URL parsers allow a username/password in the URL before the `@` symbol — `https://allowed-host.com@evil.com` — and different parsers may disagree on which part is the "host." You can embed a whitelisted value into a position the filter checks, while the actual request goes somewhere else entirely. That parser ambiguity is the core of the bypass.

**SSRF with filter bypass via open redirection** was also sharp — the trick here was that the server's URL validator only checked the initial URL, not where it ended up after a redirect. So if the application had an open redirect (e.g. `/next?url=...`), you could point SSRF at that redirect endpoint, pass the filter with a legitimate-looking URL, and let the server follow the redirect to an internal address. Two bugs chained into one.

## Practical impact — what you can actually do with SSRF

In these labs the end goal was clear: reach `http://localhost/admin`, then delete a user account. This maps directly to real-world impact — SSRF can expose:
- Internal admin panels (no auth required from localhost)
- Cloud metadata endpoints (`169.254.169.254`) — AWS, GCP, Azure all expose credentials and config here
- Internal APIs and databases not exposed to the internet
- Other back-end services (Redis, Elasticsearch, internal microservices)

## Burp Collaborator — what I actually learned to use here

Blind SSRF labs were where Burp Collaborator clicked for me. When SSRF doesn't return a visible response (blind), you can't confirm it by reading output — you need the server to make an outbound connection to a host you control. Burp Collaborator gives you a unique DNS/HTTP endpoint. If the target server hits it, you get a ping — proof the SSRF exists even with no visible response.

Key insight: **out-of-band interaction is the only way to confirm blind SSRF.** No Collaborator hit = no confirmation. This is also how blind command injection and blind XXE are confirmed — same principle.

## The common pattern across all 7 labs

Every SSRF lab came down to one of these:
- **Trust in user-supplied URLs** — server fetches whatever you give it without validating destination
- **Filter bypass** — blacklists are incomplete, whitelist parsers can be confused
- **Redirect following** — server validates the initial URL but blindly follows redirects
- **Blind interaction** — no output, but server still makes the request — OOB detection required

The real-world hunting angle: look for any parameter that takes a URL, domain, IP, or path as input — `url=`, `uri=`, `src=`, `dest=`, `webhook=`, `fetch=`, `endpoint=`, `callback=`. These are your SSRF candidates.

## Real-world bug bounty angle

SSRF is consistently a **Critical/High** finding in bug bounty programs because internal network access and cloud metadata exposure are direct paths to infrastructure compromise. On AWS, a successful SSRF to `169.254.169.254/latest/meta-data/iam/security-credentials/` can hand you IAM credentials for the entire account.

Hunting approach going forward: during recon, any endpoint that takes a URL as input gets SSRF tested. First try localhost and common internal ranges, then Burp Collaborator for blind confirmation, then filter bypass if initial attempts are blocked.

## Tools used

- Burp Suite Repeater — request tampering and filter bypass testing
- Burp Collaborator — out-of-band detection for blind SSRF
- Manual testing — filter bypass requires understanding how the server parses URLs, not just automated scanning
