Bootstrap a full Product Requirements Document (PRD) and product notes file for this project.

$ARGUMENTS

## Instructions

### Step 1 — Gather context

If `$ARGUMENTS` is provided, treat it as the product brief (a description of what the product is, who it's for, and what it does). If no arguments are provided, look for any existing brief, notes, or README files in the working directory and read them before proceeding. If nothing is found, ask the user for the product brief before continuing.

### Step 2 — Identify roles, domains, and scope

From the brief, extract:
- **User roles** — every type of person who interacts with the product (e.g. end user, admin, moderator, external partner)
- **Core domains** — the major feature areas (e.g. auth, discovery, events, payments, notifications)
- **Platform-level concerns** — things the system enforces, not tied to one user (e.g. feature flags, internationalisation, rate limiting)
- **Out of scope for MVP** — note anything explicitly deferred to post-launch

### Step 3 — Create the PRD file

Create a file named `product-plan.md` (or append `-prd` if that name already exists).

Structure:
```
# [Product Name] — Product Plan

## Context
[2-3 sentence product description]

## Decisions Made
[Key tech, architectural, or scope decisions already locked in]

## User Stories by Role

### EPIC: Platform Foundation
#### P-01: [Capability]
- As the **platform**, I want ...

### ROLE: [Role Name]
#### [R]-01: [Capability Name]
- As a **[role]**, I want ..., so that ...
...

## MVP Phase Breakdown

### Phase 1: Foundation (Month 1-2)
...

### Phase 2: [Name] (Month 3-4)
...

## Tech Stack Recommendation
...

## Open Questions
...
```

Rules for user stories:
- Format: `As a **[role]**, I want [action], so that [outcome].`
- Bold the role name
- One testable behaviour per bullet
- Always include the `so that` clause
- Cover the primary actor, secondary actors, and platform enforcement
- Use epic codes: P (platform), then first letter of each role (A=Artist, V=Venue, U=User/Fan, etc.)

Phase breakdown rules:
- 4 phases is standard for a 6-8 month MVP; adjust count to match the actual scope
- Phase 1 is always foundation: auth, data model, core profiles
- Each phase title includes the month range
- List the concrete deliverables per phase as bullets

### Step 4 — Create the product notes file

Create a file named `product-notes.md`.

Structure:
```
# [Product Name] — Product Notes

## What Is [Product Name]?
[2-3 sentence plain-English description]

## Rollout Strategy
...

## Tech Stack
...

## User Roles
### [Role]
...

## [Domain] Features
### [Feature]
- [Bullet]
...

## MVP Phases
...
```

Rules:
- Prose bullets, no user-story framing
- Each section maps to a role or feature domain
- Keep it readable as a stakeholder summary, not a technical spec

### Step 5 — Offer to create a matching Apple Note

After creating `product-notes.md`, check whether a Note already exists with a matching title:
- Derive the title: filename without path and `.md`, hyphens/underscores → spaces (e.g. `product-notes.md` → `product notes`, or `MyApp-Product-Notes.md` → `MyApp - Product Notes`)

Check via:
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
```

- If a Note **already exists** with that title — skip creation, report it exists.
- If no Note exists — ask the user: *"Would you like me to also create a matching Apple Note titled '[title]' in your Projects folder so the product notes stay in sync?"* and wait for their reply before creating it.

If the user says yes, convert `product-notes.md` to Apple Notes HTML format (see conversion table in `/add-feature`) and create the Note in the "Projects" folder:
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

### Step 6 — Report

After creating all files, list:
- The roles identified
- The epics created and total user story count
- The MVP phase breakdown summary (one line per phase)
- Any open questions you surfaced
- Whether an Apple Note was created or already existed
