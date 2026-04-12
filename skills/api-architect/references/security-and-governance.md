# Security & Distributed Governance 

API perimeters are obsolete. Designing for highly distributed integrations requires embracing the Zero Trust philosophy. Every service must validate requests based on contextual policies, not subnet locations.

## Zero Trust Verification
* **Public/Delegated Access:** Rely strictly on **OAuth 2.1**. Deprecate outdated "Implicit" and "Password" flows. Enforce the **Authorization Code Flow with PKCE** for all frontends, utilizing short-lived access tokens and rotated refresh tokens.
* **Internal Service-to-Service:** Utilize **Mutual TLS (mTLS)** via your Service Mesh. Both the calling microservice and the receiving service validate their distinct X.509 certificates to cryptographically seal communications.

## Standardized Error Definitions (RFC 9457)

Stop returning arbitrary JSON keys for errors (`{ "err": "broken" }`). The API architecture must natively implement the **RFC 9457: Problem Details for HTTP APIs** specification. 

* The `content-type` response must be `application/problem+json`.
* Critical fields include: `type` (a documented URI), `title`, `status`, `detail`, and `instance` (the request tracker). 
* Use the extended `errors` array immediately to pass out exact payload validation flaws.

## Distributed Rate Limiting (IETF)

API gateways mitigate volumetric disaster using algorithmic rate limiters (Token Bucket, GCRA).

To empower developers to write safe, proactive programmatic clients, ensure APIs export the **IETF RateLimit Header** standards:
* `RateLimit-Limit`: Maximum requests per window.
* `RateLimit-Remaining`: The client’s current quota.
* `RateLimit-Reset`: The exact Unix timestamp defining when the quota replenishes.

Instruct clients returning a `429 Too Many Requests` code to observe the `Retry-After` header strictly and utilize local Exponential Backoff with Jitter for their internal retry queues.
