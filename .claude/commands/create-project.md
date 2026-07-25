Bootstrap a brand-new product from scratch: client folder structure, structured PRD, product notes, and optional integrations (Apple Notes, Notion workspace, DB schema, DBML).

$ARGUMENTS

## Instructions

This skill is the single entry point for starting a new project from zero. It first classifies the **project shape** (multi-role platform, multi-tenant SaaS, single-user app, content site, CLI tool, library, desktop app, game, internal tool, or other) and then runs a shape-conditional questionnaire — asking role and onboarding questions only when the shape actually has users, asking pricing only for paid products, and so on. It generates the canonical document set (`product-plan.md`, `product-notes.md`, `project-bootstrap.json`) with only the sections that apply, and chains the rest of the zero2one skills as opt-in steps after confirmation.

Every step that touches the host environment (filesystem, Apple Notes, Notion, `gh`, `git`) is gated on a capability check, so the same skill can run in a local Mac environment or, later, in a stripped-down cloud environment.

---

### Step 1 — Detect environment capabilities

Before asking anything, detect what the host can do:

```python
import os, platform, shutil

CAPS = {
    "filesystem":  os.access(os.path.expanduser("~"), os.W_OK),
    "macos_notes": platform.system() == "Darwin" and shutil.which("osascript") is not None,
    "notion":      bool(os.environ.get("NOTION_API_KEY") or os.environ.get("NOTION_TOKEN")),
    "github_cli":  shutil.which("gh") is not None,
    "git":         shutil.which("git") is not None,
}
```

Store the results — every optional step later checks the relevant capability and silently skips (and logs in the summary) if it's missing.

---

### Step 2 — Intake the brief

If `$ARGUMENTS` is provided, treat it as the product brief.

Otherwise, look for an existing brief in the working directory in this order:
1. `brief.md` / `BRIEF.md`
2. `README.md` (if it reads like a brief, not just install instructions)
3. Any `*-notes.md` or `*-prd.md` file
4. Any `.txt` file that looks like notes

If none is found, ask: *"What's the product? A short paragraph is fine — who it's for, what it does, the main outcome."*

The brief is the source of truth for inferred defaults in the next step.

---

### Step 3 — Infer defaults from the brief

Read the brief and infer these defaults. The **project shape** is the most important inference — it decides which subsequent sections apply. A CLI tool has no roles or onboarding. A single-user consumer app has one profile per user and no role taxonomy. A content site has authors and readers, not "account types" and "sub-roles". Every section below is gated on the shape.

**Project shape** (pick the closest fit):

| Shape | Typical signals in the brief |
|---|---|
| **Multi-role platform** | Multiple types of user (buyers/sellers, artists/venues, etc.), discovery, social features, marketplace dynamics |
| **Multi-tenant SaaS** | Organisations sign up; each org has its own users; per-tenant data isolation implied |
| **Single-user consumer app** | "personal", "your", "my"; one profile per user; no cross-user features |
| **Content site / blog / marketing** | Mostly read-only for visitors; small admin team; publishing workflow |
| **CLI tool / library / SDK** | No UI; developer audience; terminal; imported by other code |
| **Desktop app** | Runs locally; offline-first; filesystem-heavy |
| **Game** | Player-facing entertainment; engine references; single or multiplayer |
| **Internal tool** | Team-only; no public users; "our team", "internal" |
| **Other** | Anything else; user describes shape |

**Other slot-level inferences** (only apply where the shape supports them):

| Slot | What to look for |
|---|---|
| Project name | First proper noun used as a product name, or a capitalised phrase that recurs |
| Tagline | Any sentence that positions the product ("X is a Y that does Z") |
| Target market | Locations, demographics, vertical mentioned |
| Roles | Only if the shape has users — extract role names |
| Modules | Domain nouns repeated as feature areas |
| Tech hints | Explicit framework, language, or service mentioned |
| Monetization | freemium, subscription, ad, commission, marketplace |
| Compliance | GDPR, PCI-DSS, HIPAA, DPDP, KYC — or none |
| Rollout | Phased launch, single-region first, global-from-day-one |

These become pre-selected defaults. Show the shape choice first and get explicit confirmation before proceeding — every subsequent section is gated on it.

---

### Step 4 — Run the shape-first questionnaire

The questionnaire branches on the confirmed project shape. **Skip any section that doesn't apply.** Do not force a user through role taxonomy or onboarding questions when their product has no users, and do not ask about SSG frameworks for a mobile-first platform.

Use AskUserQuestion to batch related questions (max 4 per call). For every question with an inferred default, show it as the first option and mark it `(Recommended — inferred from brief)`.
For multi-select questions, use `multiSelect: true`.
For free-text answers (name, tagline, role lists, module lists), ask directly and accept the reply verbatim.

#### Section A0 — Confirm the project shape (always)

1. **Project shape** — confirm the inference from Step 3, or pick a different shape. Everything below adapts.

#### Section A — Identity (always)

2. **Project name**
3. **One-line tagline / positioning**
4. **Detailed description** (3-5 sentences; pre-fill from brief)
5. **Target market** — geography + audience
6. **Team size** — solo / small team (2-5) / larger team
7. **Brand direction** — logo description, colour palette, theming (light / dark / system), positioning word.
   *Skip for CLI / library / SDK shapes.*

#### Section B — Users & auth (skip if the shape has no users)

Skip entirely for CLI / library / SDK.
Ask a compressed version for internal tool, single-user app, content site.
Ask the full version for multi-role platform, multi-tenant SaaS, game with accounts.

8. **Do you have user accounts?** — no / just an admin login / yes, one role / yes, several roles

Branch on the answer:

- `no` → skip to Section C
- `just admin login` → record `roles = ["admin"]`; skip taxonomy questions
- `yes, one role` → ask the role's name; skip taxonomy questions
- `yes, several roles` → continue with 9-14:

9. **Distinct user roles** — free text list; inferred from brief
10. **Is there a two-tier account+sub-role model?** *(only ask if 3+ roles)* — some roles are publicly discoverable (account types), others are operational and invite-only (sub-roles) — yes / no
11. **Multi-context per user?** *(only ask if 2+ roles)* — can one account hold the same role across multiple entities, or different roles across different entities? yes / no
12. **Onboarding paths** (multi-select) — invited / approval gate / direct self-serve / create-new-entity
13. **Identity verification** (multi-select) — OTP (phone) / email / government ID / profile photo / age verification / KYC docs / password only
14. **Invite-only launch?** (manual approval before activation) — yes / no

#### Section C — Platform foundation (shape-preselected menu; user tweaks)

Present the full menu but pre-select the subset that fits the shape. Users tick or untick.

| Feature | Preselected for shapes |
|---|---|
| Theming (light / dark / system) | consumer, SaaS, game, content, desktop |
| Internationalisation (locale, currency, timezone) | consumer, SaaS, multi-role, content |
| Accessibility (screen reader, ARIA, keyboard nav) | consumer, SaaS, multi-role, content, internal |
| Feature flags | SaaS, multi-role |
| Multi-context / multi-tenant scoping | SaaS, multi-role |
| Location model (hierarchical, geo) | multi-role |
| Audit log (immutable change history) | SaaS, multi-role, internal |
| Fuzzy / full-text search | multi-role, SaaS, content |
| Real-time updates (WebSockets / pub-sub) | multi-role, game |
| Offline support | desktop, consumer, game |
| Coming-soon / geofencing | multi-role |
| Push notifications | consumer, multi-role, SaaS |
| In-app chat | multi-role, SaaS |
| In-app notification inbox | consumer, multi-role, SaaS |
| SEO structured data | content |
| RSS / feed export | content |
| SSO (SAML / OIDC) | internal, SaaS |

15. **Foundation features** (multi-select; presets above)

*Skip the entire section for CLI / library / SDK shapes.*

#### Section D — Core modules (always)

16. Extract candidate modules from the brief and present them. User confirms, adds, removes. Every project — regardless of shape — has feature areas; the label varies:
    - Platform / SaaS / consumer: **modules**
    - Content site: **content types** or **sections**
    - CLI / library: **sub-commands** or **packages**
    - Game: **systems** (combat, inventory, world, etc.)
    - Desktop: **modules** or **panels**

#### Section E — Monetization (skip if free / open source / internal)

17. **How is this funded?** — free / paid / freemium / open source / internal-only
    If `free` / `open source` / `internal` → skip the rest of this section.
18. **Monetization model** — subscription / one-time purchase / per-transaction fee / ad-supported / in-app currency / marketplace commission / hybrid
19. **Payment gateway** — Stripe / Razorpay / PhonePe / PayPal / App Store IAP / Play Store IAP / none / TBD
20. **If freemium**: tier names and limits (free / paid tier caps on records, storage, seats, or rate).

#### Section F — Tech stack (options depend on shape)

Ask only the layers relevant to the shape. Pre-fill defaults from brief hints; user overrides.

| Shape | Layers to ask |
|---|---|
| Multi-role platform | mobile, backend, database (+extensions), cache, storage, search, push, hosting |
| Multi-tenant SaaS | frontend, backend, database (+ tenancy strategy: shared / schema-per-tenant / db-per-tenant), cache, storage, hosting |
| Single-user consumer app | mobile OR web, backend (if any), local storage, sync strategy, hosting |
| Content site | SSG framework (Next / Astro / Hugo / Nuxt / SvelteKit), CMS, hosting |
| CLI / library | language + package manager, distribution channel (npm / crates.io / PyPI / brew / apt) |
| Desktop app | shell (Electron / Tauri / native), language, local database |
| Game | engine (Unity / Unreal / Godot / custom), platform targets, netcode (if multiplayer) |
| Internal tool | frontend, backend, database, hosting, SSO provider |

21. Ask the relevant layers in one batch. Skip layers that don't apply.

#### Section G — Compliance & operations (skip for CLI/library with no data)

Frame each as *"commonly missed — include?"*. Skip the whole section for CLI / library / SDK shapes with no data persistence. For every other shape, ask only the items whose subject matter actually applies (e.g. skip data-export questions if there are no user accounts).

22. **Compliance scope** (multi-select) — GDPR / India DPDP / PCI-DSS / HIPAA / SOC 2 / none *(skip if no user data collected)*
23. **Data export & account deletion** — yes / no / later *(only ask if there are user accounts)*
24. **Email & SMS provider** — SendGrid / Postmark / Mailgun / SES / Twilio / MSG91 / none *(skip if the product doesn't send anything)*
25. **Rate limiting & API abuse prevention** — yes / no *(only if there's a network-facing API)*
26. **A/B testing infrastructure** — yes / no / later *(skip for CLI/library/desktop)*
27. **Webhooks / partner API** — yes / no / later
28. **Backup & disaster recovery** — full DR plan / nightly backups only / TBD / not applicable
29. **Platform-operator admin dashboard** — yes / no *(only if the operator needs to manage tenants, users, or content)*
30. **Customer support flow** — in-app tickets / email-only / phone / chatbot / community forum / none

#### Section H — MVP plan (always, but scale to shape)

Default phase count adapts to shape:

| Shape | Default phases × months |
|---|---|
| Multi-role platform, multi-tenant SaaS | 4 × 2 months |
| Single-user consumer app | 2-3 × 1-2 months |
| Content site | 1-2 × 1 month |
| CLI / library / SDK | 1 phase (or version-based cadence, no phases) |
| Desktop app | 2-3 × 1-2 months |
| Game | ask user — project-scale-dependent |
| Internal tool | 1-2 × 1 month |

31. **Number of phases** — pre-fill per shape; user overrides
32. **Phase duration** in months — pre-fill per shape
33. **Rollout pattern** — city-by-city / national / global from day 1 / regional / release cadence (for CLI / library: semver / continuous / LTS) / N/A
34. **Start date** — today by default

#### Section I — Optional integrations (capability-gated)

Show only options whose capability is `True` from Step 1.

35. **Create folder structure** at chosen path (default `~/Work/Client/<Name>/`) — yes / no / custom path
36. **Create Apple Note** mirroring `product-notes.md` — yes / no *(needs macOS + Notes)*
37. **Create Notion notes page** — yes / no *(needs `NOTION_API_KEY`)*
38. **Initialise git repo** — yes / no
39. **Run `/gen-db-schema`** — yes / no *(skip prompt for CLI / library / content-site shapes with no relational DB)*
40. **Run `/gen-dbml`** — yes / no (only if 39 = yes)
41. **Run `/gen-notion-workspace`** — yes / no *(needs `NOTION_API_KEY`; skip prompt for CLI / library shapes where a full PM workspace isn't warranted)*

---

### Step 5 — Confirm before writing

Print a single summary table of every answer the user gave (or that was inferred and unchanged). Group by section. End with:

```
About to create:
  ✓ ~/Work/Client/<ProjectName>/  (with subfolders per global CLAUDE.md)
  ✓ product-plan.md                (N user stories across M epics)
  ✓ product-notes.md
  ✓ project-bootstrap.json
  ✓ Apple Note "<ProjectName> - Product Notes" in Projects folder
  ✓ Notion notes page under <parent>
  ✓ Will then run: /gen-db-schema, /gen-dbml, /gen-notion-workspace
  ✗ Skipped: git init (user declined)
  ⊘ Skipped: Apple Notes (capability missing — not on macOS)

Confirm? (yes / edit / cancel)
```

Wait for explicit yes. On "edit", let the user say what to change and re-show the summary. On cancel, exit without writing.

---

### Step 6 — Create the folder structure (if opted in)

If the user chose to scaffold folders, create the canonical client folder layout from the global CLAUDE.md:

```
<root>/<ProjectName>/
├── Projects/
├── Invoices/
├── Contracts/
├── Requirements/
├── Design/
├── Assets/
├── Originals/
├── Documents/
│   └── Emails/
├── Screenshots/
├── Recordings/
├── Backups/
└── Credentials/         # never read or commit from this folder
```

```python
import os
folders = ["Projects", "Invoices", "Contracts", "Requirements", "Design",
           "Assets", "Originals", "Documents/Emails", "Screenshots",
           "Recordings", "Backups", "Credentials"]
root = os.path.expanduser(answers["folder_root"])
project_dir = os.path.join(root, answers["identity"]["name"])
for f in folders:
    os.makedirs(os.path.join(project_dir, f), exist_ok=True)
```

The PRD and product notes are written into `<project_dir>/Requirements/`. Subsequent skills (`/gen-db-schema`, etc.) operate from that folder.

If the user is not on a Mac or said no, skip this step and write all files into the current working directory.

---

### Step 7 — Write `product-plan.md` (the PRD)

The template below is the maximal shape (multi-role platform). **Emit only the sections that apply to the chosen project shape.** Do not include role/onboarding/monetization/compliance sections when the answers show they don't apply. If a section would be empty, drop it entirely rather than leaving a placeholder.

Maximal structure — trim to what applies:

```
# [Project Name] — Product Plan

## Context
[Detailed description from Section A.3]

## Decisions Made
[Include only the lines whose answers exist. Skip any line whose value is empty or "n/a".]
- Project shape: [shape]
- Roles: [list, if any]
- Onboarding paths: [list, if any]
- Tech stack: [one-liner per layer that was chosen]
- Monetization: [model + gateway, if paid]
- Compliance: [list, if any]
- Rollout: [pattern, if applicable]

## User Stories by [Role | Module | Sub-command | System]
[Heading label depends on shape: "Role" for platforms/SaaS, "Module" for consumer apps,
 "Sub-command" for CLI/library, "System" for games.]

### EPIC: Platform Foundation
[Only emit this epic if Section C selected at least one foundation feature.
 One #### P-XX per selected feature.]
#### P-01: [Foundation capability]
- As the **platform**, I want ...

### EPIC: Onboarding & Authentication
[Only emit if Section B recorded onboarding paths or identity verification.
 One #### O-XX per path + one for verification.]
#### O-01: [Path name]
- As an **admin**, I want ...

### ROLE: [Role name from Section B]
[Only emit if the product has user roles. One ROLE section per role.
 For each role generate a Profile epic and one epic per module the role owns.]
#### [Code]-01: [Capability name]
- As a **[role]**, I want ...

### EPIC: [Module name from Section D]
[If shape has no roles (CLI/library/content/desktop-solo), organise stories by module
 instead of by role. Use the module label appropriate for the shape.]
#### [Code]-01: ...

### EPIC: Monetization
[Only emit if Section E returned anything other than free/open-source/internal.]
#### M-01: ...

### EPIC: Compliance & Operations
[Only emit sections whose Section G answer was "yes"/"include". Skip the whole epic if none apply.]
#### C-01: Data Export (if selected)
#### C-02: Account Deletion (if selected)
#### C-03: Rate Limiting (if selected)
#### C-04: Webhooks / Partner API (if selected)
#### C-05: Backup & DR (if selected)
#### C-06: Support & Disputes (if selected)
#### C-07: Platform-Operator Admin Dashboard (if selected)

## MVP Phase Breakdown
[Emit N phases per Section H answers. For CLI/library, use release cadence instead of phases:
 a bulleted list of v0.1, v0.2, v1.0 goals rather than time-boxed phases.]

### Phase 1: [Name] ([start] – [end])
- ...

### Phase 2: [Name] ([start] – [end])
...
```

## Tech Stack
- Mobile: ...
- Backend: ...
- Database: ...
- Cache/Real-time: ...
- Storage: ...
- Search: ...
- Push: ...
- Hosting: ...

## Rollout Strategy
[Pattern from Section I.39, with launch city from Section A.5]

## Success Metrics & KPIs
- Adoption: ...
- Financial: ...
- Operational: ...
(Generate generic placeholders per domain; user can refine later.)

## Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|

(Pre-seed with: adoption risk, payment gateway delay, regulatory change, key personnel risk, server scaling. User refines later.)

## Glossary

| Term | Definition |
|---|---|

(Pre-seed with project-specific proper nouns inferred from the brief — any branded term, internal codename, or coined word that needs a definition for a new reader.)
```

**User story format — every bullet must follow this exactly:**
```
- As a **[role]**, I want [action], so that [outcome].
```

Rules:
- Bold the role name with `**role**`
- One testable behaviour per bullet — split compound ideas
- Always include the `so that` clause
- Epic codes: `P` = platform foundation, `O` = onboarding, `M` = monetization, `C` = compliance/ops, then first letter of each role or module from the user's list
- If two roles or modules share a letter, use two-letter codes (e.g. `AD` for Admin and `AU` for Auditor)
- For shapes with no roles, use codes derived from the module names instead (e.g. `IN` for Inventory, `WO` for World)

---

### Step 8 — Write `product-notes.md`

Same content as the PRD but in stakeholder-readable form: prose bullets, no user-story framing, organised by domain rather than role. **Emit only the sections that have content** — a CLI tool's notes file should not contain empty "User Roles" or "Compliance & Operations" sections. Drop empty sections entirely.

Maximal structure — trim to what applies:

```
# [Project Name] — Product Notes

## What Is [Project Name]?
[Detailed description]

## Rollout Strategy
[Only if applicable. For CLI/library, replace with "Release Cadence".]

## Tech Stack
[Only layers chosen in Section F]

## User Roles
[Skip entirely if no roles. Emit one ### per role.]

### [Role]
- [Capability]

## Core Platform Features
[Skip if Section C selected none. One ### per selected feature.]

### [Foundation feature]
- [Bullet]

## [Domain] Features
[One ## per module from Section D. Emit "Modules", "Content Types", "Sub-commands",
 or "Systems" as the outer label depending on shape.]

### [Feature]
- [Bullet]

## Monetization Strategy
[Skip if free / open source / internal.]

## Compliance & Operations
[Skip if none of the Section G items were selected. Otherwise, emit bullets only for
 the items that were selected.]

## MVP Phases
[Emit only the phases the user chose. For CLI/library, emit "Release Milestones" with
 versioned goals instead.]

### Phase 1: [Name] ([dates])
- ...

## Documentation
- Where the team will keep architecture, API contracts, ADRs
```

Rules:
- Plain imperative bullets
- Each section maps to a role or feature domain
- Read it as a stakeholder summary, not a technical spec
- This file is the source of truth for Apple Notes and Notion sync

---

### Step 9 — Save the answers JSON

Write `project-bootstrap.json` next to the two `.md` files. Every top-level key except `version`, `shape`, `identity`, and `integrations` is **optional** and should be omitted entirely when the section didn't apply to the chosen shape. Do not emit empty objects or empty arrays for skipped sections — leave the key out.

Full schema (all optional keys shown; omit any that didn't apply):

```json
{
  "version": "1",
  "shape": "multi-role-platform | multi-tenant-saas | single-user-consumer | content-site | cli-library | desktop | game | internal-tool | other",
  "identity": {
    "name": "...",
    "tagline": "...",
    "description": "...",
    "target_market": "...",
    "team_size": "solo | small | large",
    "brand": { "logo": "...", "palette": "...", "themes": ["light", "dark", "system"], "positioning": "..." }
  },
  "users": {
    "accounts": "none | admin-only | single-role | multi-role",
    "roles": [...],
    "two_tier": true,
    "multi_context": true,
    "onboarding_paths": ["invited", "approval", "direct", "create_new"],
    "verification": ["otp", "profile_photo", "age"],
    "invite_only_launch": true
  },
  "foundation": [...],
  "modules": [...],
  "monetization": {
    "model": "subscription | freemium | one-time | ad-supported | ...",
    "gateway": "stripe | razorpay | phonepe | app-store-iap | ...",
    "tiers": [{"name": "Free", "limits": {}}]
  },
  "stack": {
    "mobile": "...", "frontend": "...", "backend": "...",
    "database": "...", "tenancy": "shared | schema-per-tenant | db-per-tenant",
    "cache": "...", "storage": "...", "search": "...",
    "push": "...", "hosting": "...",
    "ssg": "...", "cms": "...",
    "language": "...", "package_manager": "...", "distribution": "...",
    "shell": "...", "engine": "...", "netcode": "...", "sso": "..."
  },
  "compliance": [],
  "ops": {
    "data_export": true,
    "account_deletion": true,
    "email_provider": "...",
    "sms_provider": "...",
    "rate_limiting": true,
    "ab_testing": false,
    "webhooks": false,
    "backup_dr": "nightly | full-DR | TBD | N/A",
    "admin_dashboard": true,
    "support_flow": "in-app | email-only | ..."
  },
  "mvp": {
    "phases": 4,
    "phase_months": 2,
    "start_date": "YYYY-MM-DD",
    "rollout": "city-by-city | national | global-day-1 | regional | semver | continuous | LTS"
  },
  "integrations": {
    "folder_root": "~/Work/Client/",
    "apple_notes": true,
    "notion_page": true,
    "git": false,
    "db_schema": true,
    "dbml": true,
    "notion_workspace": true
  }
}
```

This JSON is the portable artefact — when the skill is later moved to a web app, a form fills the same object and the same generation logic runs against it. Consumers of the JSON must treat every key except `version`, `shape`, `identity`, and `integrations` as optional.

---

### Step 10 — Optional: create an Apple Note

Skip if `CAPS["macos_notes"]` is false or the user declined.

Derive the Note title from the project name + " - Product Notes" (e.g. for a project named `Plantly` the Note title would be `Plantly - Product Notes`).

Check whether a Note already exists:
```python
import subprocess
script = '''
tell application "Notes"
  repeat with n in notes of default account
    if name of n is "TITLE" then return "found"
  end repeat
  return "not found"
end tell
'''
result = subprocess.run(["osascript", "-e", script], capture_output=True, text=True)
```

If found, report it exists and skip creation. If not, convert `product-notes.md` to Apple Notes HTML and create the Note in the "Projects" folder of the default account:

| Markdown | Apple Notes HTML |
|---|---|
| `# H1` | `<h1>...</h1>` |
| `## H2` | `<h2>...</h2>` |
| `### H3` | `<h3>...</h3>` |
| `- bullet` | `<ul><li>...</li></ul>` (collapse consecutive bullets into one `<ul>`) |
| `1. item` | `<ol><li>...</li></ol>` |
| Blank line | (paragraph break) |

```python
script = f'''
tell application "Notes"
  set targetFolder to missing value
  repeat with f in folders of default account
    if name of f is "Projects" then
      set targetFolder to f
      exit repeat
    end if
  end repeat
  if targetFolder is missing value then
    set targetFolder to folder "Notes" of default account
  end if
  make new note at targetFolder with properties {{name:"{title}", body:"{escaped_body}"}}
end tell
'''
```

---

### Step 11 — Optional: create a Notion notes page

Skip if `CAPS["notion"]` is false or the user declined.

Get `NOTION_API_KEY` from environment. Ask for the parent page ID under which to create the notes page (the 32-character ID from the Notion URL).

Convert `product-notes.md` to Notion blocks:

| Markdown | Notion block type |
|---|---|
| `# H1` | `heading_1` |
| `## H2` | `heading_2` |
| `### H3` | `heading_3` |
| `- bullet` | `bulleted_list_item` |
| `1. item` | `numbered_list_item` |
| Prose | `paragraph` |
| Blank | _(skip)_ |

```python
def make_block(block_type, text):
    return {
        "object": "block",
        "type": block_type,
        block_type: {"rich_text": [{"type": "text", "text": {"content": text}}]}
    }
```

Strip leading `#`, `-`, `*`, or `N.` before using the content text.

```python
import os, requests
headers = {
    "Authorization": f"Bearer {os.environ['NOTION_API_KEY']}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json"
}
payload = {
    "parent": {"page_id": parent_page_id},
    "properties": {"title": {"title": [{"type": "text", "text": {"content": project_name}}]}},
    "children": blocks[:100]
}
r = requests.post("https://api.notion.com/v1/pages", headers=headers, json=payload)
page_id = r.json()["id"]
for batch_start in range(100, len(blocks), 100):
    requests.patch(
        f"https://api.notion.com/v1/blocks/{page_id}/children",
        headers=headers,
        json={"children": blocks[batch_start:batch_start+100]}
    )
```

Append the page ID to the top of `product-notes.md` so later syncs find it:
```
<!-- notion-page-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

---

### Step 12 — Optional: chain to other zero2one skills

In the order the user opted in:

1. **`/gen-db-schema`** — generates `database-schema.md` from `product-plan.md`. Run only after the PRD is written.
2. **`/gen-dbml`** — converts the schema to `database-schema.dbml`. Requires step 1.
3. **`/gen-notion-workspace`** — creates the full PM workspace (Goals → Epics → Tasks → Sprints → Platforms) from the PRD. Requires `NOTION_API_KEY`.

For each, invoke the skill and pass the generated artefacts as input. Report errors but continue with the remaining steps.

---

### Step 13 — Report

Print a final summary:

```
PROJECT BOOTSTRAPPED: [Project Name]
=====================================

Identity
  Name             [Name]
  Positioning      [tagline]
  Target market    [market]
  Launch           [geography]
  Team             [solo / team size]

Roles
  Account types    [list]
  Sub-roles        [list]

Onboarding         [paths]
Foundation         [N capabilities]
Modules            [N modules]
Monetization       [model + gateway]
Stack              [mobile / backend / db]
Compliance         [list]
MVP plan           [N phases × M months, starting <date>, ending <date>]

Files written
  ✓ <project_dir>/Requirements/product-plan.md     (N user stories, M epics)
  ✓ <project_dir>/Requirements/product-notes.md
  ✓ <project_dir>/Requirements/project-bootstrap.json

Folder structure
  ✓ <project_dir>/ with 12 subfolders

Integrations
  ✓ Apple Note created: "[title]" in Projects folder
  ✓ Notion notes page: <URL>
  ✓ /gen-db-schema → database-schema.md
  ✓ /gen-dbml → database-schema.dbml
  ✓ /gen-notion-workspace → workspace URL

Skipped
  ⊘ git init — capability missing
  ✗ Apple Notes — user declined

Open questions surfaced
  - [question]
  - [question]

Next steps
  - Run /add-feature to add features as the brief evolves
  - Run /notion-schedule to set phase dates
  - Run /notion-design-sync once you have UI designs
```

If any open questions were left during the questionnaire, list them at the end so the user can come back to them.
