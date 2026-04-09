# Semantic Hub — Frontend API Reference

> **Jira:** BE-163  
> **Audience:** Frontend Team (Semantic Hub Model View feature)  
> **Tested against:** Semantic Hub v1 running locally (Spring Boot 3.4.5, Fuseki 5.0.0)

---

## Table of Contents

1. [Base URL & Server Configuration](#1-base-url--server-configuration)
2. [Authentication & Authorization](#2-authentication--authorization)
3. [Common Conventions](#3-common-conventions)
4. [Endpoints](#4-endpoints)
   - [4.1 List Models — GET /models](#41-list-models)
   - [4.2 Create Model — POST /models](#42-create-model)
   - [4.3 Update Model — PUT /models](#43-update-model)
   - [4.4 Batch Lookup — POST /models/lookup](#44-batch-lookup)
   - [4.5 Get Model — GET /models/{urn}](#45-get-model)
   - [4.6 Update Model Status — PUT /models/{urn}](#46-update-model-status)
   - [4.7 Delete Model — DELETE /models/{urn}](#47-delete-model)
   - [4.8 Get Model File — GET /models/{urn}/file](#48-get-model-file)
   - [4.9 Get Model Diagram — GET /models/{urn}/diagram](#49-get-model-diagram)
   - [4.10 Get Model Documentation — GET /models/{urn}/documentation](#410-get-model-documentation)
   - [4.11 Get JSON Schema — GET /models/{urn}/json-schema](#411-get-json-schema)
   - [4.12 Get OpenAPI Spec — GET /models/{urn}/openapi](#412-get-openapi-spec)
   - [4.13 Get Example Payload — GET /models/{urn}/example-payload](#413-get-example-payload)
   - [4.14 Get AAS Submodel Template — GET /models/{urn}/aas](#414-get-aas-submodel-template)
5. [Model Lifecycle (State Machine)](#5-model-lifecycle-state-machine)
6. [Pagination](#6-pagination)
7. [Error Handling](#7-error-handling)
8. [Swagger UI](#8-swagger-ui)
9. [Known Issues & Caveats](#9-known-issues--caveats)

---

## 1. Base URL & Server Configuration

| Environment | Base URL |
|-------------|----------|
| Local dev   | `http://localhost:4242/api/v1` |
| Deployed (via ingress) | `https://<host>/semantics/hub/api/v1` |

The API version prefix `/api/v1` is part of all endpoint paths.

---

## 2. Authentication & Authorization

### OAuth2 / OpenID Connect

In **production**, the API is protected by OAuth2 JWT bearer tokens issued by Keycloak.

| Setting | Value |
|---------|-------|
| Token header | `Authorization: Bearer <JWT>` |
| IDP Issuer URI | Configured per deployment (`idpIssuerUri` in Helm values) |
| Client ID | Configured per deployment (`idpClientId`, default: `default-client`) |

When `authentication` is set to `false` in the Helm chart (or using the `local` Spring profile), authentication is **completely bypassed** — no token is required.

### RBAC Roles

Roles are extracted from the JWT claim path: `resource_access.<clientId>.roles`.

| Role | Grants access to |
|------|-----------------|
| `view_semantic_model` | All **GET** endpoints |
| `add_semantic_model` | **POST** endpoints (create model, batch lookup) |
| `update_semantic_model` | **PUT** endpoints (update model, change status) |
| `delete_semantic_model` | **DELETE** endpoints |

### Public Endpoints (no auth required)

The following paths are accessible without authentication even in production:

- `/` — Redirects to Swagger UI
- `/swagger-ui/**`
- `/swagger-resources/**`
- `/v3/api-docs/**`
- `/semantic-hub-openapi.yaml`
- `/actuator/**`

---

## 3. Common Conventions

### URN Encoding

Model URNs follow the pattern: `urn:samm:<namespace>:<version>#<AspectName>`

Example: `urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect`

**URNs must be URL-encoded** when used as path parameters:

```
urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect
```

### Content Types

| Operation | Request `Content-Type` | Response `Content-Type` |
|-----------|----------------------|------------------------|
| List / Get / Create / Update (metadata) | — | `application/json` |
| Create / Update model (body) | `text/plain` (TTL) | `application/json` |
| Batch lookup | `application/json` | `application/json` |
| Get file | — | `text/turtle` |
| Get diagram | — | `image/svg` |
| Get documentation | — | `text/html` |
| Get JSON schema | — | `application/json` |
| Get OpenAPI | — | `application/json` |
| Get example payload | — | `application/json` |
| Get AAS (XML, default) | — | `application/xml` |
| Get AAS (JSON) | — | `application/json` |
| Get AAS (FILE) | — | `application/octet-stream` |

### Model Types

| Value | Description |
|-------|-------------|
| `SAMM` | Semantic Aspect Meta Model (current) |
| `BAMM` | Legacy BAMM format |

---

## 4. Endpoints

### 4.1 List Models

```
GET /api/v1/models
```

Returns a paginated list of all semantic models, optionally filtered by namespace and status.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | yes | `0` | Zero-based page index |
| `pageSize` | integer | yes | `10` | Items per page (spec says `10`, `50`, or `100`; server accepts any positive integer) |
| `namespaceFilter` | string | no | `""` | Substring match on model namespace |
| `status` | string | no | — | Filter by status: `DRAFT`, `RELEASED`, `STANDARDIZED`, `DEPRECATED` |

**Response:** `200 OK` — `application/json`

```json
{
  "items": [
    {
      "urn": "urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect",
      "version": "1.0.0",
      "name": "TestAspect",
      "type": "SAMM",
      "status": "DRAFT"
    }
  ],
  "totalItems": 1,
  "currentPage": 0,
  "totalPages": 1,
  "itemCount": 1
}
```

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models?page=0&pageSize=10' | jq
```

**With filters:**

```bash
curl -s 'http://localhost:4242/api/v1/models?page=0&pageSize=10&namespaceFilter=tractusx&status=DRAFT' | jq
```

---

### 4.2 Create Model

```
POST /api/v1/models
```

Creates a new semantic model by uploading a TTL (Turtle) definition.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `type` | string | yes | — | `SAMM` or `BAMM` |
| `status` | string | no | `DRAFT` | Initial status: `DRAFT`, `RELEASED`, `STANDARDIZED`, `DEPRECATED` |

**Request Body:** `Content-Type: text/plain` — raw Turtle (TTL) model definition.

**Response:** `200 OK` — `application/json`

> **Note:** The OpenAPI spec declares `201 Created`, but the server actually returns `200 OK`.

```json
{
  "urn": "urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect",
  "version": "1.0.0",
  "name": "TestAspect",
  "type": "SAMM",
  "status": "DRAFT"
}
```

**cURL:**

```bash
curl -s -X POST 'http://localhost:4242/api/v1/models?type=SAMM&status=DRAFT' \
  -H 'Content-Type: text/plain' \
  --data-binary '@model.ttl' | jq
```

---

### 4.3 Update Model

```
PUT /api/v1/models
```

Updates an existing model's TTL definition. The model URN is extracted from the TTL content.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `type` | string | yes | — | `SAMM` or `BAMM` |
| `status` | string | no | — | New status (must follow state machine rules) |

**Request Body:** `Content-Type: text/plain` — updated Turtle (TTL) model definition.

**Response:** `200 OK` — `application/json` (SemanticModel object)

**cURL:**

```bash
curl -s -X PUT 'http://localhost:4242/api/v1/models?type=SAMM' \
  -H 'Content-Type: text/plain' \
  --data-binary '@model-updated.ttl' | jq
```

---

### 4.4 Batch Lookup

```
POST /api/v1/models/lookup
```

Looks up multiple models by their URNs in a single request. Returns only models that exist — missing URNs are silently omitted from the result.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | yes | `0` | Zero-based page index |
| `pageSize` | integer | yes | `10` | Items per page |

**Request Body:** `Content-Type: application/json` — JSON array of URN strings (max 10,000 items).

```json
[
  "urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect",
  "urn:samm:org.eclipse.tractusx.other:2.0.0#OtherAspect"
]
```

**Response:** `200 OK` — `application/json` (SemanticModelList, same schema as List Models)

**cURL:**

```bash
curl -s -X POST 'http://localhost:4242/api/v1/models/lookup?page=0&pageSize=10' \
  -H 'Content-Type: application/json' \
  -d '["urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect"]' | jq
```

---

### 4.5 Get Model

```
GET /api/v1/models/{urn}
```

Returns the metadata for a single model.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `application/json`

```json
{
  "urn": "urn:samm:org.eclipse.tractusx.test:1.0.0#TestAspect",
  "version": "1.0.0",
  "name": "TestAspect",
  "type": "SAMM",
  "status": "DRAFT"
}
```

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect' | jq
```

---

### 4.6 Update Model Status

```
PUT /api/v1/models/{urn}
```

Changes the status of an existing model. Status transitions must follow the [state machine rules](#5-model-lifecycle-state-machine).

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `status` | string | yes | Target status: `DRAFT`, `RELEASED`, `STANDARDIZED`, `DEPRECATED` |

**Response:** `200 OK` — `application/json` (SemanticModel with updated status)

**cURL:**

```bash
curl -s -X PUT 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect?status=RELEASED' | jq
```

---

### 4.7 Delete Model

```
DELETE /api/v1/models/{urn}
```

Deletes a model. Only models in `DRAFT` or `DEPRECATED` status can be deleted.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `204 No Content` on success.

> **⚠ Known Bug:** This endpoint currently returns `404 Not Found` even for existing models due to a `ModelPackageUrn` lookup issue. See [Known Issues](#9-known-issues--caveats).

**cURL:**

```bash
curl -s -X DELETE 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect' -w '%{http_code}'
```

---

### 4.8 Get Model File

```
GET /api/v1/models/{urn}/file
```

Returns the raw Turtle (TTL) definition of the model.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `text/turtle`

> **Note:** The returned TTL is reconstructed from the triple store and may differ in formatting from the originally uploaded file.

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/file'
```

---

### 4.9 Get Model Diagram

```
GET /api/v1/models/{urn}/diagram
```

Returns an SVG diagram visualising the model.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `image/svg` (can be ~200 KB for complex models)

**Frontend usage:** Render directly with `<img>` tag or embed inline SVG via `dangerouslySetInnerHTML`.

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/diagram' -o diagram.svg
```

---

### 4.10 Get Model Documentation

```
GET /api/v1/models/{urn}/documentation
```

Returns auto-generated HTML documentation for the model.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `text/html` (can be ~500 KB, includes inline CSS)

**Frontend usage:** Render in an `<iframe>` or via `dangerouslySetInnerHTML`.

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/documentation' -o doc.html
```

---

### 4.11 Get JSON Schema

```
GET /api/v1/models/{urn}/json-schema
```

Returns the JSON Schema generated from the model.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `application/json`

```json
{
  "$schema": "http://json-schema.org/draft-04/schema",
  "type": "object",
  "properties": {
    "testProperty": {
      "$ref": "#/components/schemas/urn_samm_org.eclipse.esmf.samm_characteristic_2.1.0_Text"
    }
  },
  "required": ["testProperty"]
}
```

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/json-schema' | jq
```

---

### 4.12 Get OpenAPI Spec

```
GET /api/v1/models/{urn}/openapi
```

Generates an OpenAPI specification for the model. Requires a `baseUrl` query parameter that will be used as the server URL in the generated spec.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `baseUrl` | string | yes | Server base URL for the generated OpenAPI spec |

**Response:** `200 OK` — `application/json`

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/openapi?baseUrl=https://api.example.com' | jq
```

---

### 4.13 Get Example Payload

```
GET /api/v1/models/{urn}/example-payload
```

Returns a JSON example payload generated from the model definition.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Response:** `200 OK` — `application/json`

```json
{
  "testProperty": "eOMtThyhVNLWUZNRcBaQKxI"
}
```

> **Note:** String values are randomly generated. Numeric/date values use random seeds.

**cURL:**

```bash
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/example-payload' | jq
```

---

### 4.14 Get AAS Submodel Template

```
GET /api/v1/models/{urn}/aas
```

Returns an Asset Administration Shell (AAS) submodel template in the specified format.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urn` | string | URL-encoded model URN |

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `aasFormat` | string | no | `XML` | Output format: `XML`, `JSON`, or `FILE` |

**Response varies by format:**

| `aasFormat` | Content-Type | Description |
|-------------|-------------|-------------|
| `XML` (default) | `application/xml` | AAS XML representation |
| `JSON` | `application/json` | AAS JSON representation |
| `FILE` | `application/octet-stream` | `.aasx` downloadable package |

**cURL examples:**

```bash
# XML (default)
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/aas'

# JSON
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/aas?aasFormat=JSON' | jq

# FILE (downloadable)
curl -s 'http://localhost:4242/api/v1/models/urn%3Asamm%3Aorg.eclipse.tractusx.test%3A1.0.0%23TestAspect/aas?aasFormat=FILE' -o template.aasx
```

---

## 5. Model Lifecycle (State Machine)

Models follow a strict lifecycle. Status transitions that violate these rules return `400 Bad Request`.

```
DRAFT ──────► RELEASED ──────► STANDARDIZED
                  │
                  └──────────► DEPRECATED
```

| From | Allowed transitions |
|------|-------------------|
| `DRAFT` | → `RELEASED` |
| `RELEASED` | → `STANDARDIZED`, → `DEPRECATED` |
| `STANDARDIZED` | _(terminal — no transitions)_ |
| `DEPRECATED` | _(terminal — no transitions)_ |

### Deletion Rules

| Status | Deletable? |
|--------|-----------|
| `DRAFT` | Yes |
| `RELEASED` | **No** |
| `STANDARDIZED` | **No** |
| `DEPRECATED` | Yes (by design, but see [known issues](#9-known-issues--caveats)) |

---

## 6. Pagination

All list endpoints use offset-based pagination.

**Response envelope:**

```json
{
  "items": [ ... ],
  "totalItems": 42,
  "currentPage": 0,
  "totalPages": 5,
  "itemCount": 10
}
```

| Field | Description |
|-------|-------------|
| `items` | Array of `SemanticModel` objects for the current page |
| `totalItems` | Total number of matching models |
| `currentPage` | Zero-based current page index |
| `totalPages` | Total number of pages |
| `itemCount` | Number of items in the current page |

---

## 7. Error Handling

### Error Response Schema

```json
{
  "error": {
    "message": "Human-readable error description",
    "path": "/api/v1/models/...",
    "details": {}
  }
}
```

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `204` | Deleted successfully (no body) |
| `400` | Bad request — invalid input, invalid status transition, malformed TTL |
| `401` | Unauthorized — missing or invalid JWT token |
| `404` | Resource not found |
| `500` | Internal server error |

---

## 8. Swagger UI

Interactive API documentation is available at:

| Environment | URL |
|-------------|-----|
| Local | `http://localhost:4242/swagger-ui/index.html` |
| Deployed | `https://<host>/semantics/hub/swagger-ui/index.html` |

Navigating to the root URL (`/`) redirects to Swagger UI automatically.

The static OpenAPI YAML spec is served at `/semantic-hub-openapi.yaml`.

---

## 9. Known Issues & Caveats

### DELETE Endpoint Bug

`DELETE /api/v1/models/{urn}` returns `404 Not Found` for models that exist and are retrievable via GET. This is a pre-existing backend bug caused by a `ModelPackageUrn` lookup mismatch in the triple store persistence layer.

**Impact:** Frontend should show the delete action but handle 404 gracefully with a retry or error message.

### POST Returns 200 Instead of 201

`POST /api/v1/models` returns HTTP `200 OK` on success. The OpenAPI spec declares `201 Created`. Frontend should accept both `200` and `201` as success indicators.

### `pageSize` Enum Not Enforced

The OpenAPI spec restricts `pageSize` to `[10, 50, 100]`, but the server accepts any positive integer. Frontend may use these three values for a dropdown or allow custom input.

### Model File Reconstruction

`GET /models/{urn}/file` returns TTL reconstructed from the triple store, not the original uploaded file. Formatting (whitespace, ordering, comments) will differ from the original.

### Large Response Payloads

- Diagrams (`/diagram`): ~200 KB SVG
- Documentation (`/documentation`): ~500 KB HTML
- JSON Schema, OpenAPI, AAS: vary by model complexity

Consider lazy-loading these resources in the frontend rather than fetching all on model view load.
