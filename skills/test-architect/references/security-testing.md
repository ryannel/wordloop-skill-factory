# Security Testing as a Continuous Attribute

This module defines how security testing is integrated as a continuous, automated discipline within the CI/CD pipeline — not a periodic audit or separate gating phase. Security validation is woven into every layer of the testing hierarchy.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Broken Access Control](#broken-access-control)
3. [Continuous DAST and Fuzzing](#continuous-dast-and-fuzzing)
4. [SAST and SCA in the PR Loop](#sast-and-sca-in-the-pr-loop)
5. [Security by Layer](#security-by-layer)
6. [Pipeline Integration](#pipeline-integration)

---

## Core Principle

Security is not a separate phase. It is a **continuous attribute** validated at every testing level. The cost of a security defect scales dramatically with how late it is discovered. Catching a broken access control bug in a Service Test costs minutes; catching it in production costs millions.

### Priority Order (by Impact)

1. **Broken Access Control** — The highest-impact vulnerability class. Validate at the service level in every Service Test.
2. **Injection Flaws** — SQL injection, command injection, XSS. Caught by DAST/fuzzing and validated via Service Tests.
3. **Vulnerable Dependencies** — Known CVEs in third-party libraries. Caught by SCA scanning in the PR loop.
4. **Misconfiguration** — Default credentials, exposed debug endpoints, overly permissive CORS. Caught by static analysis and Service Tests.

---

## Broken Access Control

### Why It's #1

Broken access control consistently ranks as the most critical web application vulnerability. It occurs when:

- A user can access another user's data by modifying an ID in the URL
- A regular user can perform admin actions by calling admin endpoints directly
- Service-to-service authentication is absent or bypassable
- Authorization checks are missing from new endpoints

### Validation Strategy

Test access control at the **Service Test** level — on every PR. Every endpoint must have explicit tests for:

| Test Category | What to Validate |
|---------------|-----------------|
| **Authentication Required** | Unauthenticated requests return 401 |
| **Authorization Enforced** | Users cannot access other users' resources |
| **Role-Based Access** | Regular users cannot access admin endpoints |
| **Service-to-Service Auth** | Inter-service calls require valid service tokens |
| **Resource Ownership** | Users can only modify resources they own |

### Example: Access Control Service Test

```go
func TestGetMeeting_ShouldReturn403_WhenUserDoesNotOwnMeeting(t *testing.T) {
    // Setup: Create meeting owned by Alice
    aliceToken := createUserToken(t, "alice")
    meeting := createMeeting(t, aliceToken, "Alice's Meeting")
    
    // Act: Bob tries to access Alice's meeting
    bobToken := createUserToken(t, "bob")
    resp := httpGet(t, fmt.Sprintf("/api/v1/meetings/%s", meeting.ID),
        withAuth(bobToken))
    
    // Assert: Access denied
    assert.Equal(t, 403, resp.StatusCode)
}

func TestDeleteMeeting_ShouldReturn401_WhenNoAuthToken(t *testing.T) {
    resp := httpDelete(t, "/api/v1/meetings/meeting_001")
    assert.Equal(t, 401, resp.StatusCode)
}

func TestAdminEndpoint_ShouldReturn403_WhenCalledByRegularUser(t *testing.T) {
    userToken := createUserToken(t, "regular-user", role: "user")
    resp := httpGet(t, "/api/v1/admin/users", withAuth(userToken))
    assert.Equal(t, 403, resp.StatusCode)
}
```

### Systematic Coverage Pattern

For every endpoint, create a test matrix:

```
Endpoint: POST /api/v1/meetings
  ├── No auth token        → 401
  ├── Expired token         → 401
  ├── Valid user token       → 201 (success)
  ├── Service token (impersonation) → 201 (success)
  └── Invalid service token  → 401

Endpoint: GET /api/v1/meetings/:id
  ├── No auth token        → 401
  ├── Owner's token         → 200 (success)
  ├── Non-owner's token     → 403
  └── Admin token           → 200 (success, if admin access allowed)
```

---

## Continuous DAST and Fuzzing

### Dynamic Application Security Testing (DAST)

DAST tests the running application from the outside by sending crafted, potentially malicious requests and analyzing responses. When driven by an OpenAPI specification, DAST tools can automatically explore the entire API surface.

### OpenAPI-Driven Fuzzing

Leverage existing OpenAPI specifications to generate intelligent, aggressive payloads that exercise:

- **Boundary values:** Maximum string lengths, zero-value integers, enormous arrays
- **Injection attempts:** SQL injection patterns, command injection, LDAP injection
- **Type confusion:** Strings where integers are expected, nested objects where strings are expected
- **Encoding attacks:** Unicode normalization exploits, null bytes, path traversal sequences
- **Format violations:** Invalid dates, malformed UUIDs, oversized payloads

### Implementation in Service Tests

```go
func TestFuzzEndpoints_ShouldNotPanic_WhenGivenMaliciousPayloads(t *testing.T) {
    payloads := []string{
        `{"title": "'; DROP TABLE meetings; --"}`,                  // SQL injection
        `{"title": "` + strings.Repeat("A", 100000) + `"}`,       // Buffer overflow
        `{"title": "<script>alert('xss')</script>"}`,               // XSS
        `{"title": "../../etc/passwd"}`,                            // Path traversal
        `{"title": "\u0000\u0001\u0002"}`,                          // Null bytes
        `not-json-at-all`,                                          // Invalid content type
    }
    
    for _, payload := range payloads {
        resp := httpPost(t, "/api/v1/meetings",
            withBody(payload),
            withAuth(validToken))
        
        // Must not panic (5xx) — should return 4xx with a clean error
        assert.True(t, resp.StatusCode >= 400 && resp.StatusCode < 500,
            "endpoint panicked or returned 5xx for payload: %s", payload[:50])
    }
}
```

### Go Native Fuzz Testing

Go's built-in fuzz testing provides coverage-guided fuzzing that explores code paths systematically:

```go
func FuzzCreateMeeting(f *testing.F) {
    // Seed corpus
    f.Add(`{"title": "Valid Meeting", "duration": 60}`)
    f.Add(`{"title": "", "duration": -1}`)
    f.Add(`{"title": null}`)
    
    f.Fuzz(func(t *testing.T, body string) {
        resp := httpPost(t, "/api/v1/meetings",
            withBody(body),
            withAuth(validToken))
        
        // The service must never panic
        if resp.StatusCode >= 500 {
            t.Errorf("server error for input: %s", body[:min(100, len(body))])
        }
    })
}
```

---

## SAST and SCA in the PR Loop

### Static Application Security Testing (SAST)

SAST analyzes source code for security vulnerabilities without executing it. Integrated into the PR pipeline, it catches issues before code is merged.

| Tool | Language | What It Catches |
|------|----------|----------------|
| **gosec** | Go | SQL injection, hardcoded credentials, weak crypto, unsafe pointer usage |
| **bandit** | Python | Shell injection, insecure deserialization, weak hashing |
| **semgrep** | Multi-language | Custom security rules, OWASP patterns, framework-specific issues |
| **ESLint security plugins** | TypeScript/JavaScript | XSS via dangerouslySetInnerHTML, eval usage, prototype pollution |

### Software Composition Analysis (SCA)

SCA scans third-party dependencies for known CVEs. This is critical because the majority of application code is third-party — and vulnerabilities in dependencies are the most common attack vector.

| Tool | Language | Integration |
|------|----------|------------|
| **govulncheck** | Go | Native Go vulnerability scanner using the Go vulnerability database |
| **pip-audit** | Python | Scans pip dependencies against the PyPI advisory database |
| **npm audit** | Node.js | Built-in npm security audit |
| **Snyk / Dependabot** | Multi-language | Automated PR creation for vulnerable dependency updates |

### Pipeline Example

```yaml
# .github/workflows/security.yml
name: Security Checks
on: pull_request

jobs:
  sast:
    steps:
      - name: Go Security Scan
        run: |
          go install github.com/securego/gosec/v2/cmd/gosec@latest
          gosec -fmt=json -out=gosec-report.json ./...
          
      - name: Python Security Scan
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-report.json
          
  sca:
    steps:
      - name: Go Vulnerability Check
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
          
      - name: Python Dependency Audit
        run: |
          pip install pip-audit
          pip-audit --desc --fix --dry-run
```

---

## Security by Test Type

Security validation is distributed across every testing layer:

| Test Type | Security Testing |
|-------|-----------------|
| **Small (Unit)** | Input validation functions, sanitization logic, crypto utilities |
| **Service Test** | Access control matrix (auth/authz per endpoint), injection resistance, error response safety |
| **System Test** | Cross-service auth propagation, token forwarding, impersonation validation |
| **Live System Test** | Real OAuth/OIDC flows, certificate validation, production credential verification |
| **PR Pipeline (Static)** | SAST (gosec, bandit), SCA (govulncheck, pip-audit), secret detection |
| **Shift-Right** | WAF log analysis, anomaly detection, penetration testing |

---

## Pipeline Integration

### Security as a Non-Negotiable Gate

Security scanning results must **block merging** for critical and high-severity findings. Medium and low findings are tracked but do not block.

```yaml
# Severity-based gating
security_gate:
  block_on:
    - critical
    - high
  warn_on:
    - medium
  ignore:
    - low
    - informational
```

### Secret Detection

Prevent credentials, API keys, and tokens from being committed to source control:

```yaml
# Pre-commit hook
- repo: https://github.com/Yelp/detect-secrets
  hooks:
    - id: detect-secrets
      args: ['--baseline', '.secrets.baseline']
```

### DevSecOps Checklist

For every new endpoint or service, ensure:

- [ ] Access control tests cover auth, authz, and resource ownership
- [ ] SAST scan passes with no critical/high findings
- [ ] SCA scan confirms no known CVEs in dependencies
- [ ] Fuzz testing confirms no panics/crashes on malformed input
- [ ] Error responses do not leak internal details (stack traces, SQL queries, file paths)
- [ ] Secrets are not hardcoded in source or configuration files
- [ ] CORS, CSP, and security headers are configured correctly

---

## See Also

- `validation-layers.md` — Access control testing by test type
- `pipeline-architecture.md` — Security gates in the PR pipeline
- `shift-right-testing.md` — Production security monitoring
