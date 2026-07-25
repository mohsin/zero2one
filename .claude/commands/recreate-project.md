Reverse-engineer the canonical project documents (structured PRD, product notes, bootstrap JSON) from an EXISTING codebase, then optionally sync to Apple Notes / Notion. The mirror image of `/create-project`: that skill goes brief → docs → code; this one goes code → docs.

$ARGUMENTS

## Instructions

Use this when a product already exists (shipped or mid-build) but was never bootstrapped through `/create-project`, so it has no PRD or product notes. The skill analyzes the repository and any surrounding client folder, infers everything the create-project questionnaire would have asked, confirms only what cannot be inferred from code, and emits the same canonical artefacts (`product-plan.md`, `product-notes.md`, `project-bootstrap.json`) so all downstream zero2one skills (`/add-feature`, `/notes-sync`, `/notion-sync`, `/gen-db-schema`) work exactly as if the project had been created from zero.

The single biggest difference from `/create-project`: the PRD must distinguish **what is shipped from what is planned**. Every epic and user story carries a status.

---

### Step 1 — Detect environment capabilities

Same as `/create-project`:

```python
import os, platform, shutil

CAPS = {
    "filesystem":  os.access(os.path.expanduser("~"), os.W_OK),
    "macos_notes": platform.system() == "Darwin" and shutil.which("osascript") is not None,
    "notion":      bool(os.environ.get("NOTION_API_KEY") or os.environ.get("NOTION_TOKEN")),
    "git":         shutil.which("git") is not None,
}
```

---

### Step 2 — Locate the project

If `$ARGUMENTS` contains a path, use it as the repo root. Otherwise use the current working directory.

Also identify the **client folder** (the parent that holds `Projects/`, `Design/`, `Documents/`, etc. per the canonical layout). If the repo lives at `<client>/Projects/<repo>`, the client folder is two levels up. Documents are written to `<client>/Requirements/`; if the client folder has no `Requirements/` directory, create just that one directory (do not reorganize the rest — that is `/project-reorganize`'s job).

If the repo cannot be found, ask for the path and stop until provided.

---

### Step 3 — Scan the codebase (the reverse questionnaire)

This step replaces the create-project questionnaire. Fill the same slots, but from evidence. Scan in this order, cheapest first:

1. **Manifests**: `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` — name, dependencies (framework, DB clients, payment SDKs, analytics), scripts (deploy targets, test tooling).
2. **Config**: framework config files (`nuxt.config.*`, `next.config.*`, etc.), CI configs, Dockerfiles, `serverless.yml`, `netlify.toml`/`vercel.json` — hosting, services, build pipeline.
3. **Schema**: migrations / ORM models / `*.sql` — tables become module candidates; enums reveal domain vocabulary (transaction types, roles, statuses); RLS policies reveal the tenancy/auth model.
4. **Routes & pages**: pages/screens directory and API routes — the app's real feature surface. Each route cluster is a module candidate.
5. **Env var NAMES only** (`.env.example`, or `grep -oE '^[A-Z_]+' .env`) — never read values. Names reveal integrations (payment gateways, AI providers, storage, email).
6. **Existing docs**: `README`, `docs/`, any `*STRATEGY*`, `*MODELS*`, `TODO`, legal docs (`privacy`, `tos`) — monetization intent, compliance posture, roadmap fragments.
7. **Git history** (if available): `git log --oneline` — development narrative, recently active areas, abandoned directions (ported-away-from tech), and a start date (first commit).
8. **Sibling services**: worker/processor/lambda folders, cron configs, admin tools.

From the scan, fill every create-project slot:

| Slot | Evidence |
|---|---|
| Project shape | Route structure + auth model + tenancy (same taxonomy as `/create-project` Step 3) |
| Name / tagline | Manifest name, README, meta tags, landing page copy |
| Roles | Auth tables, role enums, RLS policies, admin surfaces |
| Modules | Route clusters + schema table clusters, merged |
| Foundation features | Evidence of theming, i18n/currency tables, realtime, feature flags, audit logs, push, offline |
| Tech stack | Dependencies + config, per layer |
| Monetization | Payment SDKs/env keys, pricing/credits/subscription tables, tier configs |
| Compliance | Legal docs present, consent flows, deletion/export endpoints (or their absence) |
| Ops | Rate limiting, admin dashboards/CLIs, analytics, error tracking, backups |
| **Status per feature** | Implemented route/table/handler = `shipped`; stubs, TODOs, `coming soon` strings, empty handlers = `pending`; docs-only mentions = `planned` |

Record an **evidence note** for each non-obvious inference (one line: "credits system → `credit_transactions` table + `add_credits` SQL function"). These go into the confirmation summary so the user can spot wrong guesses cheaply.

---

### Step 4 — Ask only what code cannot tell you

Batch with AskUserQuestion (max 4 per call). Typical non-inferable slots:

1. **Target market & launch geography** — code rarely says who it's for.
2. **Tagline / positioning** — confirm the inferred one-liner or supply the real one.
3. **Roadmap intent** — which `pending` items are actually planned vs abandoned; what phases (if any) the user wants the plan organized into. For a live product, prefer a two-part plan: "Shipped (vN)" as a factual record, then forward phases only for pending work.
4. **Team size, monetization details not visible in code** (pricing philosophy, tier intent).
5. **Integrations** — offer only capability-true options: write docs to `<client>/Requirements/`? Apple Note (ask for the target folder path, supporting nesting like `Projects > Current`)? Notion page? Chain `/gen-db-schema` (usually unnecessary — the schema already exists; offer only if the project has no schema doc)?

Skip every question whose answer is already unambiguous from the scan or from the user's instructions in `$ARGUMENTS`.

---

### Step 5 — Confirm before writing

Show a compact summary: inferred identity, shape, roles, module list with shipped/pending counts, stack one-liners, monetization, compliance gaps found, files about to be written, and integrations chosen — plus the evidence notes for anything non-obvious. Wait for yes / edit / cancel, unless the user already pre-approved execution in their request.

---

### Step 6 — Write `product-plan.md` (the PRD)

Use the same maximal template and trimming rules as `/create-project` Step 7, with these deltas for existing products:

- Title block gains: `**Status:** Reverse-engineered from the codebase on [date]` and the version/commit analyzed.
- **Every epic heading gets a status tag**: `[SHIPPED]`, `[PARTIAL]`, or `[PENDING]`.
- **Every user story bullet ends with a status**: `— ✅ shipped`, `— 🚧 partial`, or `— ⏳ pending`. A shipped story is still written in user-story form; it documents the behaviour the code guarantees.
- `## Decisions Made` becomes `## Decisions Made (as built)` — record what the code actually does, including notable pivots visible in git history ("background removal: client MediaPipe → AWS Lambda → Cloud Run BiRefNet").
- `## MVP Phase Breakdown` becomes `## Roadmap`: Phase 0 is "Shipped to date" (factual), subsequent phases contain only pending/planned work per the user's Step 4 answers.
- Keep Success Metrics, Risk Mitigation, and Glossary — pre-seed Risk with risks actually observed in the code (single-provider dependencies, model deprecations, quota ceilings).

User story format is identical to `/create-project` (bold role, one behaviour per bullet, `so that` clause, epic letter codes).

---

### Step 7 — Write `product-notes.md`

Same stakeholder-readable template as `/create-project` Step 8 (What Is X / Rollout / Tech Stack / User Roles / Core Platform Features / per-domain features / Monetization / Compliance & Operations / Phases / Documentation), with the same trimming rules, plus:

- Feature bullets that are not yet built get a `(planned)` suffix; everything unmarked reads as live behaviour.
- Include a short `## Current Status` section near the top: one paragraph of where the product stands (live/beta, what works end to end, what is pending).
- This file is the source of truth for Apple Notes / Notion sync — same as create-project.

---

### Step 8 — Save `project-bootstrap.json`

Identical schema to `/create-project` Step 9 (same optional-key rules), with two additions:

```json
{
  "recreated": { "from": "codebase", "repo": "<path>", "commit": "<short-sha>", "date": "YYYY-MM-DD" },
  "modules": [ { "name": "Wardrobe", "status": "shipped" }, { "name": "Payments", "status": "pending" } ]
}
```

(`modules` entries may be objects with `status` instead of bare strings; consumers must accept both.)

---

### Step 9 — Optional: create an Apple Note

As `/create-project` Step 10 (existence check, markdown → Apple Notes HTML table, title `"<Name> - Product Notes"`), with one extension: **nested folder support**. The user may specify a folder path like `Projects > Current`; resolve each segment inside the previous folder:

```applescript
tell application "Notes"
  set parentFolder to folder "Projects" of default account
  set targetFolder to folder "Current" of parentFolder
  make new note at targetFolder with properties {name:"TITLE", body:"HTML"}
end tell
```

If any segment of the path does not exist, create it (`make new folder at parentFolder with properties {name:"Current"}`) rather than silently falling back to the top level.

---

### Step 10 — Optional: create a Notion notes page

Identical to `/create-project` Step 11 (markdown → blocks, 100-block batching, embed `notion-page-id` comment into `product-notes.md`).

---

### Step 11 — Report

Same shape as `/create-project` Step 13, with the shipped/pending split up front:

```
PROJECT RECREATED: [Name]
==========================
Analyzed          <repo path> @ <commit>, N files scanned
Shape             [shape]
Modules           N shipped · M partial · K pending
Stack             [one line]
Monetization      [model — status]

Files written
  ✓ <client>/Requirements/product-plan.md    (N stories: X shipped / Y pending)
  ✓ <client>/Requirements/product-notes.md
  ✓ <client>/Requirements/project-bootstrap.json

Integrations
  ✓ Apple Note "<title>" in <folder path>
  ⊘ Notion — declined

Gaps surfaced by the analysis
  - [promised-but-missing feature, legal gap, dead code, etc.]

Next steps
  - /add-feature for new features (keeps PRD + notes in sync)
  - /notes-sync after future edits to product-notes.md
```

The "Gaps surfaced" section is the recreate-specific payoff: anything the code promises but doesn't deliver (e.g. a privacy policy promising data export with no endpoint), stub handlers, and dead config — list them even if the user didn't ask.
