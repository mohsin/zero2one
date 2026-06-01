Generate a full PostgreSQL database schema document from this project's PRD.

$ARGUMENTS

## Instructions

### Step 1 — Locate the PRD

Find the product plan file in the working directory (likely `product-plan.md`, `*prd*.md`, or `*plan*.md`). Read it fully before writing any schema.

If `$ARGUMENTS` names a specific file, use that instead.

### Step 2 — Identify all entities and relationships

From the PRD, extract:
- Every **noun that persists** (user, account, order, product, message, review, etc.)
- Every **relationship** between those nouns (one-to-many, many-to-many, optional vs. required)
- Every **enumerated value** mentioned (status types, role types, policy options, etc.)
- Every **time-series or geospatial** concern (check-ins over time, location queries, sound levels)
- Every **flexible / extensible** field that warrants JSONB

### Step 3 — Write the schema document

Create a file named `database-schema.md`.

#### Header block (required)

```markdown
# [Product Name] — Database Schema

## Tech Stack
- PostgreSQL 15+
- pgcrypto (`gen_random_uuid()`)
[add extensions only if the PRD requires them, e.g.:]
[- PostGIS — if the PRD has geospatial/location features]
[- TimescaleDB — if the PRD has time-series data (sensor readings, activity logs)]
[- pg_trgm — if the PRD has fuzzy text search]

## Conventions
- All PKs are `UUID DEFAULT gen_random_uuid()`
- All timestamps are `TIMESTAMPTZ`
- Soft deletes via `deleted_at TIMESTAMPTZ`
- JSONB for extensible/flexible fields
- `updated_at` maintained by trigger
```

#### ENUM types section

Define all enums before any tables:
```sql
CREATE TYPE [name] AS ENUM ('[value1]', '[value2]', ...);
```

Group related enums together with a comment header.

#### Tables — organised by domain

Group tables under domain comment headers:
```sql
-- =============================================
-- DOMAIN: [Domain Name]
-- =============================================
```

Each table:
```sql
CREATE TABLE [name] (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ
);
```

Rules:
- UUID PKs everywhere
- Foreign keys with `ON DELETE CASCADE` or `ON DELETE SET NULL` — choose deliberately
- NOT NULL on all required fields
- JSONB for fields that are extensible or vary by context
- Only use specialised types (GEOMETRY, BIT, arrays) when the PRD explicitly warrants them — do not add extensions speculatively

#### Indexes section

After all tables:
```sql
-- INDEXES
CREATE INDEX ... ON ...;
CREATE UNIQUE INDEX ... ON ...;
```

Include:
- Indexes on all FK columns
- Partial indexes for active/non-deleted rows where queries will commonly filter on `deleted_at IS NULL`
- GIN indexes for any full-text or fuzzy-searchable columns (if pg_trgm is in use)
- GIST indexes for geometry columns (if PostGIS is in use)
- Composite indexes for common query patterns derived from the PRD's access patterns

#### Relationships summary

Add a markdown table at the end:
```markdown
## Relationships Summary
| Table | Relates To | Type |
|-------|-----------|------|
| ... | ... | one-to-many |
```

#### Ephemeral / cache data section (if applicable)

If the PRD describes real-time, ephemeral, or high-frequency data (live queues, sessions, presence indicators, rate limiting), add a final section noting what is deliberately NOT in PostgreSQL and why:
```markdown
## Ephemeral Data (Not in PostgreSQL)
- **[Key pattern]** — [what it stores] — [why ephemeral / which store: Redis, Memcached, etc.]
```

Omit this section entirely if the PRD has no such requirements.

### Step 4 — Review for completeness

Before finishing, check:
- Every role from the PRD has a corresponding user/profile table
- Every many-to-many relationship has a junction table
- Every enum referenced in a table is defined
- No table references an undefined table
- Any specialised extension types used (GEOMETRY, hypertables, etc.) are only present if the PRD warrants them

### Step 5 — Report

List the domains created, total table count, enum count, and any schema decisions that were non-obvious (e.g. why a junction table was used over an array column).
