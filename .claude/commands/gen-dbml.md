Convert this project's database schema document to DBML format for import into dbdiagram.io or dbdocs.io.

$ARGUMENTS

## Instructions

### Step 1 — Locate the schema document

Find `database-schema.md` in the working directory (or the file named in `$ARGUMENTS`). Read it fully before writing any DBML.

### Step 2 — Convert to DBML

Create a file named `database-schema.dbml` (same base name as the source file with `.dbml` extension).

#### DBML structure rules

**Enums:**
```dbml
Enum [name] {
  value1
  value2 [note: 'optional clarification']
}
```

**Tables:**
```dbml
Table [name] {
  id uuid [pk, default: `gen_random_uuid()`]
  column_name type [constraints]

  Note: 'Optional table-level description'
}
```

**Column constraints:**
- `[pk]` — primary key
- `[not null]` — required field
- `[unique]` — unique constraint
- `[default: value]` — default value (use backticks for expressions: `` `NOW()` ``)
- `[ref: > other_table.id]` — foreign key (`>` = many-to-one, `<` = one-to-many, `-` = one-to-one)
- `[note: 'text']` — inline note for non-obvious types or constraints

**Indexes:**
```dbml
Table [name] {
  ...

  indexes {
    (col1, col2) [name: 'idx_name']
    col3 [unique, name: 'uq_name']
  }
}
```

**PostgreSQL-specific types with no DBML native equivalent:**
Map as follows and add a `[note:]` explaining the real type:

| PostgreSQL type | DBML representation |
|----------------|---------------------|
| `UUID` | `uuid` |
| `TIMESTAMPTZ` | `timestamptz` |
| `GEOMETRY(...)` | `varchar` with `[note: 'PostGIS GEOMETRY(...)']` |
| `BIT(n)` | `varchar` with `[note: 'BIT(n) — proprietary token']` |
| `JSONB` | `jsonb` |
| `TEXT[]` | `varchar` with `[note: 'TEXT[] array']` |
| `INET` | `varchar` with `[note: 'INET']` |

**Composite primary keys** (junction tables) — use `indexes` block:
```dbml
Table junction_table {
  table_a_id uuid [ref: > table_a.id, not null]
  table_b_id uuid [ref: > table_b.id, not null]

  indexes {
    (table_a_id, table_b_id) [pk]
  }
}
```

**Inline refs vs. separate Ref blocks:**
- Prefer inline `[ref: > other_table.id]` on the FK column for clarity
- Use separate `Ref:` blocks only when the inline form would be ambiguous

**File header comment:**
```dbml
// [Product Name] — Database Schema (DBML)
// Generated from database-schema.md
// Import at: https://dbdiagram.io or https://dbdocs.io
//
// Notes:
// - PostgreSQL-specific types (GEOMETRY, BIT, arrays) are represented as varchar with [note:] annotations
// - Any extension-specific annotations (hypertables, etc.) are noted in table Note fields
// - Composite PKs are defined in indexes blocks
```

### Step 3 — Validate the DBML

**Always validate with the official parser** (catches errors manual review misses):

```bash
npx -y -p @dbml/cli dbml2sql database-schema.dbml -o /tmp/validate.sql
```

Note: the `-p` flag is required; `npx -y @dbml/cli dbml2sql` fails with "could not determine executable to run". On failure the CLI writes a `dbml-error.log` next to the input file listing every error with line numbers; delete it after fixing.

Also verify:
- Every `ref:` target table and column exists in the DBML
- Every enum referenced in a column is defined as an `Enum` block
- All composite PKs use `indexes { (...) [pk] }` syntax, not repeated `[pk]` columns
- No PostgreSQL-only syntax leaks into the DBML (no `CREATE TYPE`, no `::`, no `DEFAULT gen_random_uuid()` without backticks)

Known conversion pitfalls (from real runs):
- Index columns with `DESC`/`ASC` modifiers: DBML does not support ordering; strip the modifier and record it in the index `note:`
- Operator-class indexes (`title gin_trgm_ops`): keep only the column name; record `USING GIN` and the opclass in the index `note:`
- `INTERVAL`, `SMALLINT`, `TIME` columns: map to `varchar`/`int`/`time` with a `[note:]` naming the real type
- Column-level `CHECK (...)` constraints: move into the column `note:`; beware that column names starting with "check" (e.g. `checked_in_at`) are not constraints
- JSONB defaults containing commas (`DEFAULT '{"a": true, "b": false}'`): commas inside quoted defaults are part of the value, not column separators
- For large schemas (100+ tables), a mechanical parse of the SQL source is more reliable than hand-writing the DBML; validate the result with the CLI either way

### Step 4 — Report

State how many tables and enums were converted, and list any types that required special handling with notes.
