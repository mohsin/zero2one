# zero2one

A library of Claude Code slash commands for taking a product from idea to implementation. Covers the full 0→1 journey: capture ideas, structure them into a PRD, generate a database schema, and keep everything in sync as the product evolves.

---

## How it works

Each skill is a `.md` file in `.claude/commands/`. When you type `/skill-name` in Claude Code, it loads that file as the system prompt for the command and runs it against your project directory.

Skills are designed to be **stateless and project-agnostic** — they read from and write to well-known file names (`product-plan.md`, `product-notes.md`, `database-schema.md`) so they chain together naturally without configuration.

```
/new-prd → product-plan.md + product-notes.md
               ↓
/add-feature → updates both files, syncs Apple Note + Notion
               ↓
/gen-db-schema → database-schema.md
               ↓
/gen-dbml → database-schema.dbml
               ↓
/feature-to-schema → updates schema + DBML in sync
               ↓
/gen-notion-workspace → full Notion PM workspace (Goals → Epics → Tasks → Sprints → Platforms)
               ↓
/notion-schedule → set phase dates, cascade to epics and sprints
               ↓
/notion-design-sync → import screens, match to epics, detect PRD gaps (run after each design round)
               ↓
/notion-start-sprint → create sprint, set goal, close previous (run every 2 weeks)
               ↓  (calls automatically if no tasks exist for targeted epic)
/notion-add-tasks → break chosen epic into typed, sized tasks
```

---

## Skills

| Command | What it does |
|---------|-------------|
| `/new-prd` | Bootstraps a full structured PRD and product notes file from a product brief; offers to create a matching Apple Note and/or Notion page |
| `/add-feature` | Takes a feature description and adds it to the PRD (user stories) and product notes (bullets); syncs Apple Note and Notion page if they exist |
| `/notes-sync` | Bidirectional sync between a local product notes `.md` and its Apple Note — pushes file additions to Notes, pulls Note additions back to the file |
| `/notion-sync` | Bidirectional sync between a local product notes `.md` and its Notion page — pushes new sections to Notion, optionally pulls Notion-only content back to the file |
| `/gen-db-schema` | Generates a full PostgreSQL DDL schema document from the PRD |
| `/gen-dbml` | Converts the schema doc to DBML format for dbdiagram.io / dbdocs.io |
| `/feature-to-schema` | Extends the existing schema (and DBML) to support a newly described feature |
| `/gen-notion-workspace` | Generates a complete Notion PM workspace from the PRD: Goals, Epics, Tasks, Sprints, and Platforms databases — all linked — seeded with phases, epics, and platform records from the PRD; branches for solo vs team projects; auto-detects platforms from the tech stack; adds Design Status property to Epics |
| `/notion-schedule` | Sets start and end dates on all phases in an existing Notion workspace: asks for a start date, accepts fixed deadlines or estimates based on epic count, confirms before writing, cascades dates to projects and sprints |
| `/notion-reschedule` | Shifts all downstream phase, project, and sprint dates forward when a phase start is missed: computes the slip in days and applies it to every affected record after confirmation |
| `/notion-start-sprint` | Creates a new sprint in Notion: determines the next sprint number, sets 2-week dates, auto-generates a goal from targeted epics, closes the previous sprint, and calls `/notion-add-tasks` for any epic that has no tasks yet |
| `/notion-add-tasks` | Breaks down a chosen epic into individual tasks using the PRD: generates typed, sized, and prioritised tasks, asks only where the spec is ambiguous, and writes to Notion after confirmation — calls `/notion-start-sprint` if no active sprint exists |
| `/notion-design-sync` | Imports UI/UX screens from Claude Design, matches each screen to its epic, embeds screen links in epic page bodies, detects features visible in the design but missing from the PRD, and updates Design Status on each epic |

---

## Setup

### Install globally (recommended)

Makes all skills available in every Claude Code session on this machine:

```bash
cp ~/projects/zero2one/.claude/commands/*.md ~/.claude/commands/
```

### Install per-project

Copy only the skills you need into a project:

```bash
cp ~/projects/zero2one/.claude/commands/new-prd.md /your/project/.claude/commands/
```

### Verify

Open Claude Code in any project and type `/` — the skills will appear in the autocomplete list.

---

## Usage

### Start a new product from a brief

```
/new-prd A task management tool for small teams. Members can create and assign tasks, set deadlines, and track progress. Managers get an overview dashboard. Launching as a web app with a mobile companion.
```

Creates `product-plan.md` (user stories by role, phased MVP) and `product-notes.md` (readable stakeholder summary). Optionally creates a matching Apple Note and/or Notion page.

---

### Add a feature to an existing PRD

```
/add-feature Users should be able to leave comments on tasks. Anyone assigned to the task or mentioned in a comment gets notified. The task owner can delete any comment.
```

Finds the right epic in the PRD, writes user stories in the correct format, updates the product notes, and syncs Apple Notes and Notion if they've been set up.

---

### Sync product notes with Apple Notes

```
/notes-sync
```

Compares `product-notes.md` against its matching Apple Note. Shows a diff, then — on confirmation — pushes additions to Notes and pulls additions back to the file.

The Apple Notes integration is useful because ideas don't only happen at your desk. Jot a feature idea into Notes on your phone, come back to your Mac, run `/notes-sync` to pull it into `product-notes.md`, then run `/add-feature` to expand it into proper user stories in the PRD. Nothing gets lost between the shower and the keyboard.

---

### Sync product notes with Notion

```
/notion-sync
```

Compares `product-notes.md` against its matching Notion page (identified by a `<!-- notion-page-id: ... -->` comment stored at the top of the file after first sync). Shows a diff, pushes new sections to Notion, optionally pulls Notion-only sections back to the file.

Requires `NOTION_API_KEY` in your environment, or the skill will ask for it.

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

Converts `database-schema.md` to `database-schema.dbml`. Import at [dbdiagram.io](https://dbdiagram.io) or [dbdocs.io](https://dbdocs.io) for a visual ER diagram.

---

### Extend the schema for a new feature

```
/feature-to-schema Users can react to comments with emoji. Each reaction records who made it so duplicates from the same user are prevented.
```

Reads the existing schema, adds new tables/columns/enums, updates the DBML in sync, and flags migration warnings if any change could break existing data.

---

### Generate a Notion PM workspace from the PRD

```
/gen-notion-workspace
```

Reads the PRD, creates Goals, Epics, Tasks, Sprints, and Platforms databases in Notion, wires all relations and rollups, seeds records from the PRD, assigns platforms per epic, and adds Design Status to all epics. Asks solo vs team upfront and skips Owner/Team DB for solo projects.

---

### Set schedule on all phases

```
/notion-schedule
```

Asks for a start date and whether you have fixed deadlines per phase. Estimates end dates based on epic count where flexible. Shows the full proposed schedule before writing anything, then cascades dates down to epics and Sprint 1.

---

### Import design screens into the workspace

```
/notion-design-sync https://claude.ai/design/p/...
```

Or with a local handoff bundle:
```
/notion-design-sync /path/to/Super\ App-handoff.zip
```

Matches all screens to their epics, appends a Screens section to each epic page body, sets Design Status, detects features in the design that aren't in the PRD, and proposes new epics for unmatched screens.

---

### Start a sprint and break down an epic

```
/notion-start-sprint
```

Determines the next sprint number, sets 2-week dates, asks which epics you're targeting, auto-generates a sprint goal, closes the previous sprint, then calls `/notion-add-tasks` for any epic that has no tasks yet.

```
/notion-add-tasks E1.1
```

Reads the PRD and epic page body (including design screens if synced), generates a typed and sized task list, asks only where scope is ambiguous, and writes tasks linked to the active sprint.

---

### Reschedule when a phase start is missed

```
/notion-reschedule
```

Asks which phase was missed and what the new start date is. Computes the slip in days and shifts all downstream phase, epic, and sprint dates forward, then shows a before/after table before writing.

---

## Document conventions

Skills enforce consistent formats so they can read each other's output.

### PRD (`product-plan.md`)

User stories follow this exact format:
```
- As a **[role]**, I want [action], so that [outcome].
```

Rules:
- Role is always bolded
- One testable behaviour per story — split compound ideas
- `so that` clause is never omitted
- Epics use letter codes: P = platform, then first letter of each role (A = Admin, U = User, M = Manager, etc.)

### Product notes (`product-notes.md`)

```
### Feature Name
- Concise capability description
- Another capability
```

Plain imperative bullets. No "As a..." framing. Readable as a stakeholder summary. Used as the source of truth for Apple Notes and Notion sync.

Notion page ID is stored as a comment on line 1 after first sync:
```
<!-- notion-page-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

---

## Adding or improving a skill

1. Write or edit the skill file in `.claude/commands/<name>.md`
2. Copy it to `~/.claude/commands/<name>.md` to make it live globally
3. If it's new, add a row to the table above
4. Keep examples generic — no client names, project-specific domain terms, or hardcoded paths

A skill is worth adding if the answer to this is yes: *"Could someone on a completely different project run this same sequence of steps to get the same kind of output?"*

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI or desktop app
- `NOTION_API_KEY` environment variable — required for all `/notion-*` skills and Notion steps in `/new-prd` and `/add-feature`. Set it in your shell profile or pass inline. The Bash tool does not inherit `export` from your terminal, so the key must be available in the environment at skill run time.
- Notion integration set up once per workspace: go to [notion.so/profile/integrations](https://www.notion.so/profile/integrations), create an Internal integration, copy the secret, then share the workspace root page with the integration via the `...` → Connections menu.
- No other dependencies — skills run entirely through Claude
