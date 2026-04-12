# System Topologies & Infrastructure

An API does not exist in isolation. Its performance, security, and scalability are entirely dictated by the infrastructure gateways and traffic managers situated immediately upstream.

## The Two-Tier Gateway Pattern

Treating an API Gateway as a monolithic "traffic cop" leads to choke points and security failures. Modern architectures strictly separate concerns:

1. **The Edge Gateway (North-South Traffic):** Positioned at the literal perimeter. Its primary responsibility is coarse-grained security, volumetric protection, and routing external Internet traffic into the cluster.
   - *Tasks:* OAuth 2.1 token validation, DDoS protection edge caching, WAF inspection, global rate limiting.
2. **The Internal Gateway / Service Mesh (East-West Traffic):** Positioned within the cluster Kubernetes perimeter. It manages how internal services communicate with each other side-by-side.
   - *Tasks:* Implementing Mutual TLS (mTLS) for "Zero Trust" internal auth, intelligent content-based traffic splitting (canary releases), and dynamic retries/circuit breakers.

*Note on Kubernetes:* Focus architectural output towards the **Kubernetes Gateway API** standard, moving away from legacy Ingress controllers.

## Multi-Tier Caching Architectures

To survive extreme latency constraints and database saturation, implement layered caching:

1. **Edge/CDN Cache:** Return entirely static/idempotent `GET` responses directly from a Point of Presence (PoP) close to the user before they ever hit your Edge Gateway.
2. **Distributed Application Cache (Redis/Memcached):** Placed behind the API service logic. Ensures that scaled microservices hit a centralized cache that avoids redundant database queries.
3. **Local In-Memory Cache:** Used specifically for ultra-high frequency lookups (e.g., config switches) within a specific pod. Must be managed carefully to avoid node skew.

### Write-Through vs. Cache-Aside
* **Cache-Aside:** The most common API read pattern. The application searches the cache first, on miss it searches the DB, and then writes into the cache. Best for heavy reads.
* **Write-Behind:** API writes to cache immediately for fast return, and DB syncs asynchronously. High throughput, but risks data loss on cache-node failure.
