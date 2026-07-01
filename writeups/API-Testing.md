# API Testing — Key Learnings

**Status:** 5/5 labs completed ✅

---

## What is it?

API testing is about finding vulnerabilities in how an application's API endpoints are designed, exposed, and secured. Unlike traditional web app testing where you interact via a UI, API testing means working directly with the underlying requests — changing methods, manipulating parameters, probing undocumented endpoints, and reading error responses to map out attack surface.

## Why it exists in real apps?

APIs are built fast. Documentation gets exposed accidentally. Developers add a DELETE method "temporarily" and forget to remove it. A PATCH endpoint accepts fields it shouldn't. An admin API path isn't restricted because "it's internal anyway." These assumptions break the moment someone starts looking at traffic in Burp instead of clicking through the UI.

## Most interesting labs + what I learned

**Exploiting an API endpoint using documentation** — the entry point here was the API documentation itself being publicly accessible (`/api/docs`, `/openapi.json`, or similar). These docs list every endpoint, accepted method, and parameter — essentially handing you a map of the entire attack surface. In real apps, exposed API docs are more common than developers think, especially on staging environments that get promoted to production.

**HTTP method tampering + price manipulation** — the UI only lets you do certain things (view a product, add to cart), but the underlying API accepts methods the UI never uses. Switching from GET to PATCH on a product endpoint and sending a modified price or discount field worked — the server accepted it because there was no server-side validation that the price field should be read-only. 100% discount, zero cost purchase. This is **mass assignment** in practice — the API blindly accepts any field you send in the request body without checking which fields should actually be user-controlled.

**Path traversal in API endpoints** — reaching the admin panel without administrator credentials by manipulating the API path. The server was constructing internal paths based on user input, so by injecting `../../administrator` (and encoding it various ways — `%2f..%2f`, `%23`, `#`, URL double-encoding) you could traverse out of the expected directory and hit restricted endpoints. The key was trying multiple encoding forms because the server or WAF was decoding at different stages — what gets blocked in plain text sometimes passes encoded.

**Error-driven testing** — this was the biggest mindset shift. Errors aren't failures, they're information:
- `404` — path doesn't exist, try a different one
- `403` — path exists, you're not allowed, look for a bypass
- `401` — path exists, authentication needed, check if a different method skips it
- `400` — bad request format, server is telling you what it expects
- `500` — something broke server-side, you touched something sensitive

Reading errors and deciding the next step based on what they reveal is what separates methodical API testing from random fuzzing.

**Email + password reset parameter manipulation** — changing the email field in a password reset or account update request to point to an admin account, combined with path-based access, allowed taking over the administrator account without ever knowing their password.

## The common pattern across all 5 labs

Every API lab came down to: **the API trusts the client more than it should.**

- Trusts that the client will only send the fields it should (mass assignment)
- Trusts that the client will only use the HTTP methods the UI exposes (method tampering)
- Trusts that the path the client provides maps to something the client owns (path traversal)
- Trusts that the client's request is well-intentioned and doesn't validate against business rules server-side

## Real-world bug bounty angle

API vulnerabilities are extremely common in modern apps because:
- Every mobile app and SPA communicates via API — massive attack surface
- API docs get exposed accidentally all the time (`/swagger`, `/api-docs`, `/redoc`)
- Mass assignment is endemic in frameworks that auto-map request body to model fields
- HTTP method restrictions are often forgotten — DELETE and PATCH left open

**Hunting approach:** intercept all traffic in Burp while using the app normally. Note every API endpoint. Then for each one: try all HTTP methods, send unexpected fields in the body, check if numeric IDs can be swapped (IDOR), look for documentation endpoints, and read every error response carefully — it's telling you something about the server's internal logic.

## Tools used

- Burp Suite Proxy — capturing all API traffic while using the application
- Burp Suite Repeater — method switching, parameter manipulation, path traversal attempts
- Manual testing — error-driven, methodical probing of each endpoint
