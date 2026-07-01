# Authentication Vulnerabilities — Key Learnings

**Status:** 14/14 labs completed ✅

---

## What is it?

Authentication vulnerabilities are flaws in how an application verifies **who you are**. Unlike access control (which is about what you're allowed to do once verified), authentication bugs let you either bypass the verification entirely or impersonate someone else. The impact is direct: you get into accounts that aren't yours.

## Why it exists in real apps?

Authentication is deceptively hard to implement correctly. Developers handle the "happy path" (correct username + correct password = login) but make assumptions everywhere else — they assume brute force isn't feasible, assume 2FA codes can't be predicted, assume the password reset flow can't be manipulated. Each assumption is a potential bypass.

## Most interesting labs + what I learned

**Broken brute-force protection, multiple credentials per request** (Expert) was the sharpest lab. The protection being bypassed: rate limiting or lockout after N failed attempts per request. The bypass: instead of sending one password per request, convert the password list into a JSON array and send all of them in a single request. The server processes each credential in the array sequentially but counts it as one attempt from the rate limiter's perspective. One request, hundreds of passwords tested — lockout never triggers.

This works because the rate limiting logic operates at the **request level**, not at the **credential-check level**. The developer thought about "too many requests" but not "too many passwords inside one request."

**Broken brute-force protection, IP block** taught me something I'll use constantly — `X-Forwarded-For` header manipulation. When a server blocks an IP after failed attempts, it's often reading the client IP from this header rather than the actual TCP connection IP. Since `X-Forwarded-For` is a header the client controls, you can rotate it with each request and the server thinks every attempt is coming from a different IP. Never actually blocked.

**Payload processing in Burp Intruder** — this came up across multiple labs where the server wasn't accepting plain text passwords but hashed/encoded credentials. The flow: raw password list → Intruder applies MD5 hash → then Base64 encodes → sends the processed payload. This matters because if you just throw a wordlist at a login endpoint without matching the server's expected format, every attempt fails silently — not because the password is wrong but because the format is wrong. Matching the server's processing pipeline is a prerequisite to effective brute forcing.

**2FA broken logic** — the 2FA code was being validated against a `verifyUser` cookie that could be set to any username. So you could log in with your own credentials, get to the 2FA step, swap the cookie to the victim's username, and brute force the 4-digit OTP (only 10,000 combinations). The 2FA was protecting the wrong thing — it checked the cookie, not the session that was actually authenticated in step 1.

## The common pattern across all 14 labs

Authentication failures almost always come from one of these:

- **Rate limiting at the wrong layer** — limiting requests but not credential attempts per request, or limiting by IP which is attacker-controlled via headers
- **Trusting client-controlled values** — cookies, headers, parameters that identify "who is being authenticated" can be tampered with
- **Decoupled multi-step flows** — step 1 (password) and step 2 (2FA) don't share state properly, so step 2 can be manipulated independently of step 1
- **Predictable/short tokens** — 4-6 digit OTPs, weak password reset tokens, stay-logged-in cookies based on username+timestamp hashes with known secrets

## Real-world bug bounty angle

Authentication bugs are high-value findings because account takeover is direct, demonstrable impact. Programs take them seriously.

Hunting approach: for any login form — test username enumeration first (response differences, timing differences, lockout behavior). For any 2FA — check if the session from step 1 is actually validated in step 2, or if step 2 can be reached independently. For any password reset — check if the token is tied to a specific user session or just floating. For any rate limiting — test `X-Forwarded-For` rotation and JSON array credential stuffing.

## Tools used

- Burp Intruder — primary tool across all brute force labs
- Burp Intruder Payload Processing — MD5 hash + Base64 encode applied to wordlists before sending
- Burp Repeater — manual flow manipulation for 2FA logic and cookie tampering
- Manual observation — username enumeration via response differences and timing
