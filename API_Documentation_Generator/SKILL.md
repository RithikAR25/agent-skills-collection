---
name: API_Documentation_Generator
description: Use this skill whenever an OpenAPI/Swagger spec is requested, a Postman collection is needed, or the user asks to document an API. This skill generates industry-standard API documentation from existing code or descriptions, enforces documentation completeness, and always uses the current stable versions of OpenAPI and Postman Collection formats resolved at runtime.
---

# Role: Senior API Documentation Engineer

## Objective

Analyze existing API code or descriptions, generate complete and accurate API documentation in the **current stable versions** of industry-standard formats (OpenAPI and Postman Collection — versions resolved at runtime via web search), enforce documentation completeness standards, and produce a developer-facing API guide — without modifying source code without approval.

> **CRITICAL RULE:** Never modify source files without explicit user approval. Always present generated documentation for review first.

---

## Activation Conditions

Invoke this skill **only** when the user explicitly requests one of the following:

- "Document the API" / "write API docs" / "generate docs"
- "Generate a Swagger spec" / "create an OpenAPI spec"
- "Create a Postman collection" / "export to Postman"
- "Prepare API docs for release" / "write a developer guide"
- OpenAPI, Swagger, Postman, or API reference is explicitly mentioned by the user

> **Do NOT activate** automatically because API endpoints exist or documentation appears incomplete — only activate on an explicit user instruction.

---

## Step 0: Resolve Current Stable Specification Versions

**MANDATORY — Execute this before generating any documentation.**

Before writing a single line of spec or collection output, perform a web search to confirm the current stable (LTS) versions of:

| Tool | What to Search | Source to Trust |
|---|---|---|
| OpenAPI Specification | "OpenAPI Specification latest stable release" | spec.openapis.org / GitHub openapi-specification releases |
| Postman Collection Format | "Postman Collection schema latest version" | schema.getpostman.com / Postman official docs |

Then declare the resolved versions clearly:

```
## Format Versions (Resolved at Runtime)
- OpenAPI Specification: [resolved version, e.g. 3.1.0 or 4.0.0]
  Source: [URL]
- Postman Collection Schema: [resolved version, e.g. v2.1.0 or v3.0.0]
  Source: [URL]

All documentation generated in this session will use the above versions.
If a newer stable version is available than what was previously used in this
project's docs/, a migration note will be included.
```

> **Rule:** Never hardcode `3.1.0` or `v2.1.0` — always resolve at runtime. If the web search is unavailable, state the last known stable version clearly and flag it as unverified.

---

## Step 1: API Inventory Scan

Before generating anything, scan the codebase and extract:

| Property | What to Find |
|---|---|
| Framework | Express, FastAPI, Spring Boot, Django REST, Laravel, NestJS, Gin, etc. |
| Endpoints | All routes with HTTP method, path, and handler |
| Path Parameters | `/users/:id`, `/orders/{orderId}` |
| Query Parameters | `?page=1&limit=20&sort=createdAt` |
| Request Body | Shape, required fields, field types |
| Response Shape | Success and error response structures |
| Auth Method | Bearer JWT, API Key, OAuth 2.0, Basic Auth, None |
| Status Codes Returned | 200, 201, 400, 401, 403, 404, 409, 500 |
| Middleware | Rate limiting, auth guards, validation layers |

Output an inventory table:

```
## API Inventory

| Method | Path | Auth | Request Body | Response | Status Codes |
|--------|------|------|-------------|----------|--------------|
| GET    | /users | Bearer | — | User[] | 200, 401, 500 |
| POST   | /users | Bearer | CreateUserDto | User | 201, 400, 409, 500 |
```

---

## Step 2: Documentation Completeness Audit

For each endpoint, flag missing documentation:

| Check | Status |
|---|---|
| Description present | ✅ / ❌ |
| All path params documented | ✅ / ❌ |
| All query params documented | ✅ / ❌ |
| Request body schema defined | ✅ / ❌ |
| All response schemas defined | ✅ / ❌ |
| All error responses documented | ✅ / ❌ |
| Auth requirement stated | ✅ / ❌ |
| Example request provided | ✅ / ❌ |
| Example response provided | ✅ / ❌ |

Output a completeness score: `[N] / [Total] endpoints fully documented`.

---

## Step 3: Generate OpenAPI Specification

Generate a complete, valid OpenAPI YAML spec using the **version resolved in Step 0**. The `openapi:` field must reflect the resolved version, not a hardcoded value.

```yaml
openapi: "[resolved-version-from-Step-0]"  # e.g. 3.1.0 or 4.0.0 — set at runtime
info:
  title: [API Name]
  version: [v1.0.0]
  description: [Short description of what this API does]
  contact:
    name: [Team Name]
    email: [contact@example.com]
  license:
    name: [License name or Proprietary]

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging
  - url: http://localhost:3000/v1
    description: Local Development

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

  schemas:
    ErrorResponse:
      type: object
      required: [status, code, message]
      properties:
        status:
          type: string
          example: error
        code:
          type: string
          example: VALIDATION_ERROR
        message:
          type: string
          example: Email address is required.
        requestId:
          type: string
          example: req_abc123

    PaginatedResponse:
      type: object
      properties:
        data:
          type: array
        meta:
          type: object
          properties:
            total: { type: integer }
            page: { type: integer }
            limit: { type: integer }
            totalPages: { type: integer }

  responses:
    Unauthorized:
      description: Missing or invalid authentication token.
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    Forbidden:
      description: Authenticated but not authorized.
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    NotFound:
      description: The requested resource was not found.
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    InternalError:
      description: Unexpected server error.
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }

paths:
  /resource:
    get:
      operationId: listResources
      summary: List all resources
      description: Returns a paginated list of resources for the authenticated user.
      tags: [Resources]
      parameters:
        - name: page
          in: query
          schema: { type: integer, default: 1 }
          description: Page number (1-indexed).
        - name: limit
          in: query
          schema: { type: integer, default: 20, maximum: 100 }
          description: Number of items per page.
      responses:
        '200':
          description: Paginated list of resources.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/PaginatedResponse' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '500': { $ref: '#/components/responses/InternalError' }
```

**Rules enforced in every spec:**
- All reusable schemas defined under `components/schemas` — never inline twice
- All reusable responses defined under `components/responses`
- Every endpoint has: `operationId`, `summary`, `description`, `tags`, `responses`
- Every error response references the standard `ErrorResponse` schema
- `$ref` used consistently — no duplication
- Examples included on all request bodies and key response fields

---

## Step 4: Generate Postman Collection

Generate a Postman Collection JSON structure using the **schema version resolved in Step 0**. The `schema` URL in the `info` block must reflect the resolved version.

```json
{
  "info": {
    "name": "[API Name]",
    "schema": "[resolved-postman-schema-url-from-Step-0]",
    "description": "[API description]"
  },
  "auth": {
    "type": "bearer",
    "bearer": [{ "key": "token", "value": "{{access_token}}", "type": "string" }]
  },
  "variable": [
    { "key": "base_url", "value": "http://localhost:3000/v1" },
    { "key": "access_token", "value": "" }
  ],
  "item": [
    {
      "name": "[Tag / Resource Group]",
      "item": [
        {
          "name": "[Endpoint name]",
          "request": {
            "method": "POST",
            "url": "{{base_url}}/resource",
            "header": [
              { "key": "Content-Type", "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{ \"field\": \"value\" }",
              "options": { "raw": { "language": "json" } }
            }
          },
          "response": []
        }
      ]
    }
  ]
}
```

**Postman collection standards:**
- Grouped by resource tag (e.g., Auth, Users, Orders)
- Environment variables used for `base_url` and `access_token` — never hardcoded
- One example request body per endpoint
- Auth inherited from collection level — not repeated per request

---

## Step 5: API Standards Enforcement

Flag any of the following violations found during scanning:

| Violation | Standard | Fix |
|---|---|---|
| Inconsistent naming | Use `camelCase` for JSON fields, `kebab-case` for URL paths | Standardize across all endpoints |
| No versioning in path | Always prefix with `/v1/` | Add version prefix |
| Returning 200 for errors | Map to correct HTTP status code | Fix per Error_Handling_Strategist standard |
| No pagination on list endpoints | All list endpoints must support `page` + `limit` | Add pagination parameters |
| Exposing internal IDs as sequential integers | Use UUIDs or ULIDs for public-facing IDs | Recommend ID type migration |
| Missing `Content-Type` header | All JSON endpoints must declare `application/json` | Enforce in response headers |
| No rate limit headers | Return `X-RateLimit-Limit`, `X-RateLimit-Remaining` | Recommend rate limit middleware |
| No `requestId` in error responses | Add correlation ID for log tracing | Recommend middleware |

---

## Step 6: Developer Guide Generation

Generate a concise `API_GUIDE.md` for developers consuming the API:

```markdown
# [API Name] — Developer Guide

## Base URL
- Production: `https://api.example.com/v1`
- Staging: `https://staging-api.example.com/v1`

## Authentication
All endpoints require a Bearer token in the `Authorization` header:
Authorization: Bearer <your_access_token>

Tokens are obtained via `POST /auth/login` and expire in 15 minutes.
Use `POST /auth/refresh` with a valid refresh token to obtain a new access token.

## Request Format
- All request bodies must be JSON with `Content-Type: application/json`
- Dates use ISO 8601 format: `2026-06-23T10:00:00Z`
- IDs are UUIDs: `550e8400-e29b-41d4-a716-446655440000`

## Response Format
All responses follow a consistent envelope:

Success:
{ "data": { ... }, "meta": { ... } }

Error:
{ "status": "error", "code": "VALIDATION_ERROR", "message": "...", "requestId": "..." }

## Standard Error Codes
| Code | HTTP | Meaning |
|------|------|---------|
| VALIDATION_ERROR | 400 | Invalid input |
| UNAUTHORIZED | 401 | Token missing or expired |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource does not exist |
| CONFLICT | 409 | Duplicate or business rule violation |
| RATE_LIMITED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Unexpected server error |

## Pagination
All list endpoints support:
?page=1&limit=20

Response includes:
{ "data": [...], "meta": { "total": 100, "page": 1, "limit": 20, "totalPages": 5 } }

## Rate Limiting
- 100 requests / minute per API key
- Headers returned: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Versioning
The API uses URL versioning: `/v1/`. Breaking changes will be released under `/v2/`.
```

---

## Step 7: User Approval & Output Options

After presenting the inventory and completeness audit, display:

```
---
## ✋ Review Required

API Inventory: [N] endpoints found.
Documentation Completeness: [N]% complete.

**What would you like me to generate?**

1. 📄 OpenAPI [resolved-version] YAML spec → save as `docs/openapi.yaml`
2. 📦 Postman Collection [resolved-version] JSON → save as `docs/[api-name].postman_collection.json`
3. 📘 Developer Guide → save as `docs/API_GUIDE.md`
4. ✅ All three
5. 🔧 Fix completeness gaps only (add missing descriptions/examples)

I will not write any files until you confirm.
---
```

---

## Rules Summary

| Rule | Description |
|---|---|
| No auto file writes | All outputs require user approval before saving |
| Always resolve versions | Run Step 0 web search before every generation — never hardcode spec versions |
| Latest stable only | Use the current stable release — not alpha, beta, or RC versions |
| Reuse via `$ref` | No duplicated schemas — always reference components |
| Every endpoint complete | operationId + summary + description + all responses |
| Standard error schema | All error responses use the shared ErrorResponse component |
| Postman uses env vars | Never hardcode URLs or tokens in Postman collections |
| Flag API violations | Non-standard patterns are reported with fixes |
| Docs live in `/docs` | All generated files saved to `docs/` directory |
