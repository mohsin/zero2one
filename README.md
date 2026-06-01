# zero2one

A set of Claude Code slash commands to take a product from idea to implementation. Covers the full 0→1 journey: capture ideas, structure them into a PRD, generate a database schema, and keep everything in sync as the product evolves.

---

## What's included

| Command | What it does |
|---------|-------------|
| `/add-feature` | Takes a feature description and adds it to the PRD (user stories) and product notes (bullets); syncs the matching Apple Note if one exists, or offers to create it |
| `/new-prd` | Bootstraps a full structured PRD and product notes file from a product brief; offers to create a matching Apple Note in the Projects folder |
| `/notes-sync` | Bidirectional sync between a local product notes `.md` and its Apple Note — pushes additions from the file into Notes, pulls anything Notes has that the file doesn't |
| `/gen-db-schema` | Generates a full PostgreSQL DDL schema document from the PRD |
| `/gen-dbml` | Converts the schema doc to DBML format for dbdiagram.io / dbdocs.io |
| `/feature-to-schema` | Extends the existing schema (and DBML) to support a newly described feature |

---

## Setup

### Option A — Project-level (recommended)

Copy the `.claude/` directory into the root of your project:

```bash
cp -r /path/to/zero2one/.claude /path/to/your-project/
```

The commands will appear as slash commands when you run Claude Code from that project directory.

### Option B — Global (available in all projects)

Copy the commands into your global Claude Code commands directory:

```bash
cp /path/to/zero2one/.claude/commands/*.md ~/.claude/commands/
```

### Verify

Open Claude Code in your project and type `/` — you should see `add-feature`, `new-prd`, `gen-db-schema`, `gen-dbml`, and `feature-to-schema` in the list.

---

## Usage

### Start a new product from a brief

```
/new-prd A task management tool for small teams. Members can create and assign tasks, set deadlines, and track progress. Managers get an overview dashboard. Launching as a web app with a mobile companion.
```

Creates `product-plan.md` (user stories by role, MVP phases) and `product-notes.md` (readable stakeholder summary).

---

### Add a feature to an existing PRD

```
/add-feature Users should be able to leave comments on tasks. Anyone assigned to the task or mentioned in a comment gets notified. The task owner can delete any comment.
```

Finds the right section in your PRD, writes user stories in the correct format, and updates the product notes. If an Apple Note with a matching title exists in your Projects folder, it is updated in sync. If no Note exists yet, you'll be asked whether to create one.

---

### Sync product notes with Apple Notes

```
/notes-sync
```

Compares the local `product-notes.md` against its matching Apple Note (matched by title, searched in your Notes "Projects" folder). Shows a diff of what's in each but not the other, then — on your confirmation — pushes additions to Notes and pulls additions back to the file. Useful after editing a Note directly on iPhone/iPad and wanting the file to catch up, or vice versa.

---

### Generate a database schema from the PRD

```
/gen-db-schema
```

Reads `product-plan.md`, extracts every entity and relationship, and creates `database-schema.md` with full PostgreSQL DDL — enums, tables, indexes, and an ephemeral data section for anything that doesn't belong in the primary database.

---

### Export the schema to a diagram tool

```
/gen-dbml
```

Converts `database-schema.md` to `database-schema.dbml`. Import the output file at [dbdiagram.io](https://dbdiagram.io) or [dbdocs.io](https://dbdocs.io) to get a visual ER diagram.

---

### Extend the schema for a new feature

```
/feature-to-schema Users can react to comments with emoji. Each reaction records who made it so duplicates from the same user are prevented.
```

Reads the existing schema, identifies what's new, adds the tables/columns/enums, and updates the DBML file in sync. Flags any migration warnings if a change could break existing data.

---

## Document conventions

These commands enforce a consistent format across projects.

### PRD user stories

```
- As a **[role]**, I want [action], so that [outcome].
```

Examples:
```
- As an **admin**, I want to invite team members by email, so that they can access the workspace without manual setup.
- As a **subscriber**, I want to pause my subscription, so that I'm not charged during periods I'm not using the service.
- As the **platform**, I want to lock an account after five failed login attempts, so that brute-force access is prevented.
```

- Role is always bolded
- One testable behaviour per story — split compound ideas
- `so that` clause is never omitted
- Epics use letter codes: P (platform), then first letter of each role (e.g. A = Admin, U = User, M = Manager)

### Product notes

```
### Feature Name
- Concise capability description
- Another capability
```

Examples:
```
### Notifications
- Users choose which notification types they receive (email, push, in-app)
- Digest mode batches low-priority alerts into a daily summary
- Critical alerts are always delivered regardless of preference settings
```

Plain imperative bullets. No user-story framing. Readable as a stakeholder summary.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI or desktop app
- No other dependencies — the commands run entirely through Claude
