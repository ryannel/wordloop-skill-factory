# OpenAPI 3.1 Specification Guide

## Table of Contents

- [Structure Overview](#structure-overview)
- [Info Object](#info-object)
- [Servers](#servers)
- [Paths and Operations](#paths-and-operations)
- [Components](#components)
- [Schemas and Data Types](#schemas-and-data-types)
- [Schema Composition](#schema-composition)
- [Validation Constraints](#validation-constraints)
- [Security Schemes](#security-schemes)
- [Examples](#examples)
- [Tags and Organization](#tags-and-organization)
- [JSON Schema Alignment](#json-schema-alignment)
- [Code Generation](#code-generation)
- [Validation and Linting](#validation-and-linting)
- [Best Practices](#best-practices)

---

## Structure Overview

OpenAPI 3.1 is fully aligned with JSON Schema, resolving historical schema discrepancies. A single spec serves as the source of truth for documentation, validation, SDK generation, and contract testing.

```yaml
openapi: 3.1.0
info: { ... }              # API metadata
servers: [ ... ]           # Base URLs
paths: { ... }             # Endpoints and operations
components:
  schemas: { ... }         # Reusable data models
  responses: { ... }       # Reusable response definitions
  parameters: { ... }      # Reusable parameter definitions
  examples: { ... }        # Reusable examples
  securitySchemes: { ... } # Authentication definitions
security: [ ... ]          # Global security requirements
tags: [ ... ]              # Endpoint grouping
```

---

## Info Object

```yaml
info:
  title: Users API
  version: 1.0.0
  description: |
    # Users API

    Manages user accounts and profiles.

    ## Features
    - User CRUD operations
    - OAuth 2.1 authentication
    - Role-based authorization
  termsOfService: https://example.com/terms
  contact:
    name: API Support Team
    email: api-support@example.com
    url: https://example.com/support
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT
```

The `description` field supports Markdown — use it for rich documentation that renders in API viewers like Scalar, Redoc, and Swagger UI.

---

## Servers

```yaml
servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging
  - url: http://localhost:3000/v1
    description: Local development

  # Server variables for dynamic URLs
  - url: https://{environment}.example.com/{version}
    variables:
      environment:
        default: api
        enum: [api, staging, dev]
      version:
        default: v1
        enum: [v1, v2]
```

---

## Paths and Operations

### Complete Endpoint Example

```yaml
paths:
  /users:
    get:
      summary: List users
      description: Retrieve a paginated list of users with optional filtering
      operationId: listUsers
      tags: [Users]
      parameters:
        - name: cursor
          in: query
          description: Opaque cursor for pagination
          schema: { type: string }
        - name: limit
          in: query
          description: Maximum items to return
          schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
        - name: status
          in: query
          description: Filter by status
          schema:
            type: string
            enum: [active, inactive, suspended]
      security:
        - BearerAuth: []
      responses:
        "200":
          description: Paginated list of users
          content:
            application/json:
              schema: { $ref: "#/components/schemas/UserListResponse" }
              examples:
                success: { $ref: "#/components/examples/UserListSuccess" }
        "401": { $ref: "#/components/responses/Unauthorized" }
        "429": { $ref: "#/components/responses/TooManyRequests" }

    post:
      summary: Create user
      description: Create a new user account
      operationId: createUser
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/CreateUserRequest" }
      responses:
        "201":
          description: User created successfully
          headers:
            Location:
              description: URL of the created user
              schema: { type: string, format: uri }
          content:
            application/json:
              schema: { $ref: "#/components/schemas/User" }
        "400": { $ref: "#/components/responses/BadRequest" }
        "409": { $ref: "#/components/responses/Conflict" }

  /users/{id}:
    parameters:
      - name: id
        in: path
        required: true
        description: User ID
        schema: { type: string, format: uuid }
    get:
      summary: Get user
      operationId: getUser
      tags: [Users]
      responses:
        "200":
          description: User found
          content:
            application/json:
              schema: { $ref: "#/components/schemas/User" }
        "404": { $ref: "#/components/responses/NotFound" }
    patch:
      summary: Update user
      operationId: updateUser
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/UpdateUserRequest" }
      responses:
        "200":
          description: User updated
          content:
            application/json:
              schema: { $ref: "#/components/schemas/User" }
        "404": { $ref: "#/components/responses/NotFound" }
    delete:
      summary: Delete user
      operationId: deleteUser
      tags: [Users]
      responses:
        "204": { description: User deleted }
        "404": { $ref: "#/components/responses/NotFound" }
```

### Operation ID

Every operation must have a unique `operationId`. This is used by code generators to name SDK methods and by AI agents to identify operations.

Use descriptive names: `listUsers`, `createUser`, `getUserById`, `updateUser`, `deleteUser`. Avoid generic names like `get` or `post`.

For AI-agent readability, prefer verbose, self-explanatory names: `get_user_by_email` over `getUser`.

---

## Components

### Reusable Schemas

```yaml
components:
  schemas:
    User:
      type: object
      required: [id, email, name, created_at]
      properties:
        id:         { type: string, format: uuid, readOnly: true }
        email:      { type: string, format: email }
        name:       { type: string, minLength: 1, maxLength: 100 }
        status:     { type: string, enum: [active, inactive, suspended], default: active }
        created_at: { type: string, format: date-time, readOnly: true }
        updated_at: { type: string, format: date-time, readOnly: true }
        metadata:   { type: object, additionalProperties: { type: string } }

    CreateUserRequest:
      type: object
      required: [email, name]
      properties:
        email:    { type: string, format: email }
        name:     { type: string, minLength: 1, maxLength: 100 }
        metadata: { type: object, additionalProperties: { type: string } }

    UpdateUserRequest:
      type: object
      properties:
        email: { type: string, format: email }
        name:  { type: string, minLength: 1, maxLength: 100 }
      minProperties: 1

    UserListResponse:
      type: object
      required: [data, pagination]
      properties:
        data:
          type: array
          items: { $ref: "#/components/schemas/User" }
        pagination:
          $ref: "#/components/schemas/CursorPage"

    CursorPage:
      type: object
      required: [has_more]
      properties:
        next_cursor: { type: string, nullable: true }
        has_more:    { type: boolean }
```

### Reusable Responses

```yaml
components:
  responses:
    BadRequest:
      description: Malformed request
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/ProblemDetail" }
    Unauthorized:
      description: Authentication required
      headers:
        WWW-Authenticate: { schema: { type: string } }
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/ProblemDetail" }
    NotFound:
      description: Resource not found
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/ProblemDetail" }
    Conflict:
      description: State conflict
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/ProblemDetail" }
    TooManyRequests:
      description: Rate limit exceeded
      headers:
        Retry-After: { schema: { type: integer } }
        RateLimit:   { schema: { type: string } }
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/ProblemDetail" }
```

### Reusable Parameters

```yaml
components:
  parameters:
    CursorParam:
      name: cursor
      in: query
      description: Opaque pagination cursor
      schema: { type: string }
    LimitParam:
      name: limit
      in: query
      description: Maximum items per page
      schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
    IdParam:
      name: id
      in: path
      required: true
      description: Resource ID
      schema: { type: string, format: uuid }
```

---

## Schemas and Data Types

### Primitive Types

```yaml
# String variants
type: string                              # Plain string
type: string, format: email               # Email address
type: string, format: date-time           # ISO 8601 datetime
type: string, format: date                # ISO 8601 date
type: string, format: uuid               # UUID
type: string, format: uri                # URI
type: string, format: uri-reference       # Relative URI

# Numeric types
type: integer, format: int32             # 32-bit integer
type: integer, format: int64             # 64-bit integer
type: number, format: float              # 32-bit float
type: number, format: double             # 64-bit double

# Boolean
type: boolean

# Null (OAS 3.1 supports native null via JSON Schema)
type: ["string", "null"]                 # Nullable string
```

### Arrays

```yaml
type: array
items: { $ref: "#/components/schemas/User" }
minItems: 0
maxItems: 100
uniqueItems: true
```

### Objects

```yaml
type: object
required: [name, email]
properties:
  name:  { type: string }
  email: { type: string, format: email }
additionalProperties: false        # Strict — reject unknown properties
```

### Enums

```yaml
type: string
enum: [active, inactive, suspended]
default: active
```

---

## Schema Composition

### allOf — Inheritance/Extension

```yaml
AdminUser:
  allOf:
    - $ref: "#/components/schemas/User"
    - type: object
      properties:
        admin_level: { type: integer }
        permissions: { type: array, items: { type: string } }
```

### oneOf — Exactly One Match

```yaml
PaymentMethod:
  oneOf:
    - $ref: "#/components/schemas/CreditCard"
    - $ref: "#/components/schemas/BankTransfer"
  discriminator:
    propertyName: method_type
```

### anyOf — One or More Match

```yaml
SearchResult:
  anyOf:
    - $ref: "#/components/schemas/User"
    - $ref: "#/components/schemas/Product"
```

Use `discriminator` with `oneOf` to help code generators produce proper union types.

---

## Validation Constraints

```yaml
# String validation
type: string
minLength: 1
maxLength: 255
pattern: "^[a-zA-Z0-9_-]+$"

# Numeric validation
type: integer
minimum: 0
maximum: 100
exclusiveMinimum: 0    # > 0 (not >= 0)
multipleOf: 5

# Array validation
type: array
minItems: 1
maxItems: 50
uniqueItems: true

# Object validation
type: object
minProperties: 1       # At least one property (useful for PATCH)
maxProperties: 20
```

---

## Security Schemes

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

    ApiKey:
      type: apiKey
      in: header
      name: X-API-Key

    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            users:read: Read user data
            users:write: Create and update users
            users:delete: Delete users

# Global security
security:
  - BearerAuth: []
```

---

## Examples

```yaml
components:
  examples:
    UserListSuccess:
      summary: Successful user list
      value:
        data:
          - id: "usr_001"
            email: john@example.com
            name: John Doe
            status: active
            created_at: "2024-01-15T10:30:00Z"
        pagination:
          next_cursor: "eyJpZCI6InVzcl8wMjAifQ"
          has_more: true

    CreateUserBasic:
      summary: Create user with minimal fields
      value:
        email: newuser@example.com
        name: New User
```

Include realistic examples for every schema. AI agents and documentation renderers rely on examples to understand data shapes.

---

## Tags and Organization

```yaml
tags:
  - name: Users
    description: User account management
  - name: Orders
    description: Order lifecycle management
  - name: Products
    description: Product catalog
```

Group related endpoints under the same tag. Documentation generators use tags to organize endpoints into sections.

---

## JSON Schema Alignment

OpenAPI 3.1 is fully compatible with JSON Schema draft 2020-12. This means:

- **`type` can be an array**: `type: ["string", "null"]` replaces the OAS 3.0 `nullable: true`
- **`$ref` can have siblings**: You can add `description` alongside `$ref`
- **`if/then/else`** conditional schemas are supported
- **`contentMediaType`** and `contentEncoding` for binary data validation
- Schemas can be shared between validation libraries and OpenAPI tooling without conversion

---

## Code Generation

Generate SDKs and server stubs from the spec:

```bash
# TypeScript client
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./sdk/typescript

# Go server stub
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g go-server \
  -o ./internal/api

# Python client
npx @openapitools/openapi-generator-cli generate \
  -i openapi.yaml \
  -g python \
  -o ./sdk/python
```

---

## Validation and Linting

```bash
# Redocly CLI — recommended lint tool
npx @redocly/cli lint openapi.yaml

# Spectral — advanced custom rules
npx @stoplight/spectral-cli lint openapi.yaml

# Prism — mock server for contract testing
npx @stoplight/prism-cli mock openapi.yaml
```

Integrate linting into CI/CD to catch spec errors before merge.

---

## Best Practices

1. **Use `$ref` for reuse** — Define schemas, responses, and parameters in `components`
2. **Add realistic examples** — Every schema and response should have examples
3. **Document every response** — Include all possible status codes per operation
4. **Unique `operationId`** — Descriptive names for code generation and AI agents
5. **Tag all endpoints** — Organize into logical groups
6. **Validate in CI** — Run `npx @redocly/cli lint` on every PR
7. **Version the spec** — Track spec changes alongside code changes
8. **Use `readOnly`/`writeOnly`** — Distinguish between read and write properties
9. **Provide security schemes** — Document auth for every operation
10. **Include `description`** — Describe every field, parameter, and response
