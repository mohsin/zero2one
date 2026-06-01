Extend this project's existing database schema (and DBML if present) to support a newly described feature.

$ARGUMENTS

## Instructions

### Step 1 — Understand the feature

Read `$ARGUMENTS` as a plain-language description of the feature to add. If no arguments are provided, ask the user to describe the feature before continuing.

### Step 2 — Read the existing schema

Find and read:
- `database-schema.md` — the primary PostgreSQL DDL schema
- `database-schema.dbml` — the DBML diagram file (if it exists)

Understand the current table structure, enums, and naming conventions before writing anything new.

### Step 3 — Determine what schema changes are needed

For the described feature, identify:
- **New tables** — entities that don't yet exist
- **New columns** — fields to add to existing tables
- **New enums** — new enumerated type values or entirely new enum types
- **New indexes** — query patterns the feature will introduce
- **Junction tables** — any new many-to-many relationships
- **No-op items** — things the feature needs that the schema already supports

State your analysis as a short bullet list before making any changes, so the user can verify your reading is correct.

### Step 4 — Update database-schema.md

Apply the changes:

**New enum** — add to the ENUM types section, grouped with related enums:
```sql
CREATE TYPE [name] AS ENUM ('[value1]', '[value2]', ...);
```

**New column on existing table** — add as `ALTER TABLE`:
```sql
ALTER TABLE [table] ADD COLUMN [name] [type] [constraints];
```
Place these ALTER statements in a clearly commented block at the end of the relevant domain section, not inline in the CREATE TABLE. This preserves the original table definition for readability.

**New table** — add a full `CREATE TABLE` block under the correct domain section.

**New indexes** — add to the INDEXES section.

Follow existing conventions:
- UUID PKs via `gen_random_uuid()`
- TIMESTAMPTZ for all timestamps
- NOT NULL on required fields
- JSONB for flexible/extensible fields
- FK ON DELETE policy must be deliberate (CASCADE vs SET NULL vs RESTRICT)

### Step 5 — Update database-schema.dbml (if it exists)

Apply matching changes to the DBML file:
- New `Enum` blocks for new enum types
- New `Table` blocks for new tables
- New column lines in existing `Table` blocks (for new columns)
- New `[ref:]` annotations on FK columns
- Update `indexes {}` blocks for new composite indexes

Use the same type-mapping rules from the `gen-dbml` skill (GEOMETRY → varchar + note, BIT(n) → varchar + note, etc.).

### Step 6 — Assess impact on existing data

If any changes are destructive or require a migration note (e.g. adding a NOT NULL column without a default to a table that may already have rows), flag it explicitly:

> ⚠️ Migration note: `[table].[column]` is NOT NULL with no default. A backfill is required before this migration can run safely on an existing dataset.

### Step 7 — Report

List:
- New enums added (name + values)
- New tables added (name + purpose in one line)
- Columns added to existing tables (table.column)
- Any migration warnings
- What the schema already covered that required no changes
