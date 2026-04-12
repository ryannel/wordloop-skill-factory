# Resource Design & REST

When designing synchronous HTTP interactions, the Uniform Resource Identifier (URI) serves strictly to identity the component, while the requested action is governed completely by the HTTP Method.

## Resource Modeling & Nesting

1. **Plural Nouns:** All representations should ideally be plural nouns (`/customers` over `/customer`). 
2. **Kebab-Case:** Multi-word URIs must use kebab-case (`/user-profiles`). No snake_case or camelCase in URLs.
3. **The Flat Hierarchy Rule:** Avoid deep nesting. While `/customers/1/orders` identifies relationships well, pushing deeper (`/customers/1/orders/99/products`) creates inflexible code architectures. Switch to top level collections (`/orders/99/products`) after initializing context.

## HTTP Verbs and Representation

The shape of the JSON data sent in a `POST/PUT` payload should structurally mirror the data returned by the final `GET` representation.

* **GET (Safe & Idempotent):** Read resources.
* **POST (Unsafe & Non-Idempotent):** Create items or perform generic actions if REST rules fail to map cleanly. 
* **PUT (Idempotent Update):** Full replace of a resource.
* **PATCH (Non-Idempotent / Idempotent Update):** Partial replace. Modern standards prefer PATCH endpoints to be coded defensively to act idempotently under scale.
* **DELETE (Idempotent):** Execute a destruction event.

## High Performance Retrieval (Pagination & Filtering)

Offset-based pagination (`?offset=1000`) forces databases to scan over skipped rows, exponentially slowing down as offsets scale. It also causes data-drift when new items are created during browsing.

**Enforce Cursor-Based Pagination:**
Use a "keyset" or opaque encoded cursor string that points directly to the last retrieved record. 
Example: `GET /logs?after_cursor=2026-04-01T12:00:00Z&limit=50`.
Execution remains at constant time $O(1)$ regardless of depth. Ensure `pagination.next_cursor` metadata is attached to the output representation.

Use standard functional suffixes (`_gt`, `_lt`, `_gte`) instead of raw SQL-like injection strings for robust parameter querying.
