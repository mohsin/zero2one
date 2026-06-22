Break down a Notion Epic into individual tasks and create them in the Tasks database, linked to the epic and optionally to an active sprint.

$ARGUMENTS

## Instructions

Run this during sprint planning when you're ready to start work on an epic. It reads the PRD, generates a task breakdown, and writes to Notion after confirmation.

---

### Step 1 — Locate the workspace

Check `product-notes.md` for stored Notion IDs:
```
<!-- notion-epics-db-id: ... -->
<!-- notion-tasks-db-id: ... -->
<!-- notion-sprints-db-id: ... -->
```

If not found, ask for the Epics database URL or ID.

---

### Step 2 — Identify the target epic

If `$ARGUMENTS` contains an epic name or ID, use that.

Otherwise, query the Epics DB and present a numbered list:

```python
r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query",
                  headers=headers, json={"sorts": [{"property": "Epic", "direction": "ascending"}]})
epics = [{
    "id": p["id"],
    "name": p["properties"]["Epic"]["title"][0]["plain_text"],
    "status": p["properties"].get("Status", {}).get("select", {}).get("name", ""),
    "goal": p["properties"].get("Goal", {}).get("relation", [{}])[0].get("id", "")
} for p in r.json()["results"]]
```

Show the list and ask:
> "Which epic do you want to break into tasks?"

---

### Step 3 — Check for design screens

Before reading the PRD, check if this epic has design screens attached (from a previous `/notion-design-sync` run):

```python
r = requests.get(f"https://api.notion.com/v1/pages/{epic_id}", headers=headers)
design_url    = r.json()["properties"].get("Design URL", {}).get("url", "")
design_status = r.json()["properties"].get("Design Status", {}).get("select", {}).get("name", "")
```

Also fetch the epic page body — if `/notion-design-sync` has run, it will contain a **Screens** section and possibly a **Design Additions** section with features found in the design but not yet in the PRD.

If design screens are present:
- Include them as context when generating tasks (a screen implies frontend work)
- Include any "Design Additions" items as task candidates
- Note which tasks are specifically for implementing a named screen

If `design_status` is "Not Started" or "In Design" (design not ready), warn the user:
> "This epic's design status is '{status}'. Tasks generated now may be incomplete. Consider running /notion-design-sync first, or confirm you want to proceed without design context."

### Step 4 — Read the PRD for this epic

Read the PRD/product notes file. Find all content related to the chosen epic:
- User stories for the features in this epic
- Technical requirements and constraints
- Acceptance criteria or success metrics
- Any explicit task lists or checklists already in the PRD

Also fetch the epic's page body from Notion for additional context:
```python
r = requests.get(f"https://api.notion.com/v1/blocks/{epic_id}/children?page_size=100", headers=headers)
blocks = r.json().get("results", [])
```

---

### Step 4 — Generate the task breakdown

Using the PRD content, generate a complete task list for this epic. Follow these rules:

**Task types and when to use them:**
| Type | Use for |
|---|---|
| Feature | User-facing functionality — screens, flows, interactions |
| Chore | Dev infrastructure — migrations, API endpoints, config, tests, documentation |
| Research | Unknowns that need a spike or investigation before implementation |
| Bug | Only if the epic is specifically a bug-fix epic |

**Effort sizing:**
| Size | Meaning |
|---|---|
| XS | < 2 hours — a single change, trivial integration |
| S | Half a day — one small screen or one API endpoint |
| M | 1-2 days — a complete feature with backend + frontend |
| L | 3-4 days — a complex feature with multiple moving parts |
| XL | 1 week+ — architecture-level work, only use if truly unavoidable |

**Priority:**
- **Urgent**: blocks other tasks or is on the critical path
- **High**: core to the epic's success metric
- **Normal**: standard implementation work
- **Low**: polish, nice-to-have, or post-launch refinement

**Generation rules:**
- One task = one deployable/testable unit of work. Never bundle two distinct things into one task.
- Always include at least one Chore task for the API endpoint(s) and one for tests/QA.
- If the epic involves a mobile app, split Android and iOS into separate tasks only if they diverge significantly; otherwise one task covers both (KMM handles it).
- If the PRD mentions specific acceptance criteria, turn each into a task or acceptance note.
- If a requirement is ambiguous or missing detail, flag it as a Research task rather than guessing.

**Ask the user only if:**
- A feature's scope is genuinely unclear from the PRD (e.g. "basic reporting" with no spec)
- There's a design/product decision embedded in the task (e.g. "should this be a modal or a separate screen?")
- Two valid implementation approaches exist with meaningfully different effort

For each ambiguity, ask one targeted question — do not ask about things you can reasonably infer.

---

### Step 5 — Present the task list for confirmation

Show the full breakdown before writing anything:

```
TASK BREAKDOWN: E1.1 Authentication & Onboarding
=================================================
  #  Task                                          Type     Effort  Priority
  1  OTP login screen (phone/email input + verify) Feature  M       High
  2  Society invite link flow                      Feature  M       High
  3  Path B: approval gate UI (waiting pool)       Feature  L       High
  4  Path C: create society onboarding flow        Feature  M       Normal
  5  Auth API endpoints (login, verify, session)   Chore    M       Urgent
  6  UserSocietyRole DB migration                  Chore    S       Urgent
  7  Role-based access middleware                  Chore    M       High
  8  Unit tests — auth flows                       Chore    S       Normal
  9  QA sign-off: all 3 onboarding paths           Chore    S       Normal

Total: 9 tasks  (Urgent: 2 · High: 4 · Normal: 3)
Estimated effort: ~14 days

Assign to sprint? (type sprint name, 'none', or press Enter for active sprint)
Confirm? (yes / edit / cancel)
```

If the user wants to edit, allow per-task changes before writing.

---

### Step 5b — Ensure a sprint exists

Before writing tasks, check for an active sprint:

```python
r = requests.post(f"https://api.notion.com/v1/databases/{sprints_db_id}/query",
                  headers=headers, json={"page_size": 100})
active_sprint = next(
    (p for p in r.json()["results"]
     if p["properties"].get("Status", {}).get("select", {}).get("name") == "Active"),
    None
)
```

If no active sprint exists:
> "No active sprint found. Creating one now..."
> → Invoke `/notion-start-sprint` to create the sprint first, then continue with task creation, passing the new sprint ID so tasks are assigned immediately.

If an active sprint exists, confirm:
> "Assign these tasks to {sprint_name} ({start} → {end})? (yes / different sprint / none)"

### Step 6 — Write tasks to Notion

```python
for task in confirmed_tasks:
    requests.post("https://api.notion.com/v1/pages", headers=headers, json={
        "parent": {"database_id": tasks_db_id},
        "properties": {
            "Task":     {"title": [{"text": {"content": task["name"]}}]},
            "Epic":     {"relation": [{"id": epic_id}]},
            "Status":   {"select": {"name": "To Do"}},
            "Priority": {"select": {"name": task["priority"]}},
            "Type":     {"select": {"name": task["type"]}},
            "Effort":   {"select": {"name": task["effort"]}},
            **( {"Sprint": {"relation": [{"id": sprint_id}]}} if sprint_id else {} )
        }
    })
```

If the user chose a sprint, find it by name:
```python
r = requests.post(f"https://api.notion.com/v1/databases/{sprints_db_id}/query", headers=headers, json={})
sprint = next((p for p in r.json()["results"]
               if p["properties"]["Sprint"]["title"][0]["plain_text"] == sprint_name), None)
sprint_id = sprint["id"] if sprint else None
```

If no sprint is specified but one is Active, auto-assign to it and note this in the report.

---

### Step 7 — Report

```
TASKS CREATED: E1.1 Authentication & Onboarding
================================================
✓ OTP login screen                    Feature  M   High
✓ Society invite link flow            Feature  M   High
✓ Path B: approval gate UI            Feature  L   High
✓ Path C: create society flow         Feature  M   Normal
✓ Auth API endpoints                  Chore    M   Urgent
✓ UserSocietyRole DB migration        Chore    S   Urgent
✓ Role-based access middleware        Chore    M   High
✓ Unit tests — auth flows             Chore    S   Normal
✓ QA sign-off: all 3 onboarding paths Chore   S   Normal

9 tasks created · Sprint 1 · Epic: E1.1
```
