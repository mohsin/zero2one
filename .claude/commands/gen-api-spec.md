Generate an API specification from this project's PRD.

$ARGUMENTS

## Instructions

### Step 1 — Determine format

If `$ARGUMENTS` specifies a format (`openapi` or `postman`), use it.

Otherwise, ask the user:

> Which format do you want?
> 1. **OpenAPI 3.0** — YAML spec (`api-spec.yaml`). Best for documentation, code generation, and tools like Swagger UI or Redoc.
> 2. **Postman Collection 2.1** — JSON collection (`api-collection.json`). Best for hands-on testing, sharing with teammates, and import into Postman or Insomnia.

Wait for the answer before continuing.

### Step 2 — Read the PRD

Find the product plan file (`product-plan.md`, `*prd*.md`, `*plan*.md`, or the path in `$ARGUMENTS`). Read it fully before deriving any endpoints.

If a `database-schema.md` exists, read it too — the table names and field names inform the request/response shapes.

### Step 3 — Extract domains and endpoints

From the PRD, derive:

- Every **resource** that an API consumer creates, reads, updates, or deletes (e.g. users, events, orders, reviews)
- Every **action** that is not pure CRUD but is described as an API-level interaction (e.g. approve, cancel, scan, invite, publish)
- Every **relationship** endpoint implied by the user stories (e.g. `GET /events/:id/guestlist`, `POST /artists/:id/follow`)
- Every **role boundary** — if some endpoints are admin-only, platform-only, or role-restricted, note it
- Every **query pattern** — filter, sort, and pagination params implied by list views described in the PRD

Group derived endpoints into **domain folders**, one folder per major subsystem (Auth, Events, Guestlist, etc.).

State your domain list before writing the spec, so the user can flag missing or wrong groupings.

### Step 4 — Generate the spec

#### If OpenAPI 3.0

Write `api-spec.yaml` with this structure:

```yaml
openapi: 3.0.3
info:
  title: [Product Name] API
  version: 0.1.0
  description: [One-line description from PRD]

servers:
  - url: https://api.[domain].com/v1
    description: Production

security:
  - bearerAuth: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    # Define reusable schemas for primary resources here
    Error:
      type: object
      properties:
        error: { type: string }
        message: { type: string }

paths:
  # Grouped by domain with comments
```

Rules:
- Use `$ref` for any schema used in more than one place
- Every path includes at least one response: `200` (or `201`), `400`, `401`, `404` where applicable
- List endpoints use `limit`/`offset` or `cursor`/`limit` pagination parameters
- Role-restricted endpoints carry an `x-roles` extension noting which roles can call them
- Path parameters use `{id}` style; query parameters are documented inline

#### If Postman Collection 2.1

Write `api-collection.json` with this structure:

```json
{
  "info": {
    "name": "[Product Name] API",
    "_postman_id": "[generate a random UUID]",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "baseUrl", "value": "https://api.[domain].com/v1" },
    { "key": "bearerToken", "value": "" }
  ],
  "auth": {
    "type": "bearer",
    "bearer": [{ "key": "token", "value": "{{bearerToken}}", "type": "string" }]
  },
  "item": [ /* domain folders */ ]
}
```

Rules:
- One top-level folder per domain
- Every request has a `name`, `method`, full `url` with path params as `:param` style, and appropriate `body` (raw JSON where applicable)
- Auth header is inherited from collection root — do not repeat per request
- Include a `Pre-request Script` on the collection root that sets `bearerToken` from the `pm.environment` variable `token` if present
- List requests include query params for `limit`, `offset`, and any filterable fields from the PRD
- Request bodies use realistic example values derived from the PRD domain (names, statuses, enums)
- Requests that require a specific role note it in the request description

### Step 5 — Review for completeness

Before finishing, verify:
- Every primary resource in the PRD has at minimum: list (GET), create (POST), get-by-id (GET), update (PATCH/PUT), delete/archive (DELETE)
- Every cross-resource relationship from the PRD has a corresponding nested route or filter param
- Every role from the PRD has at least the endpoints it needs to fulfil its user stories
- No endpoint references a resource not defined in the collection

### Step 6 — Report

State:
- Domains covered and endpoint count per domain
- Total endpoint count
- Any user stories that implied functionality but were ambiguous to translate into an endpoint (flag for the user to resolve)
- Any schema decisions that were non-obvious (e.g. why an action is `POST /resource/:id/action` rather than `PATCH /resource/:id`)
