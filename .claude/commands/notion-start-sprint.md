Create a new sprint in the Notion Sprints database, set its dates and goal, and optionally kick off task assignment for the sprint's chosen epic(s).

$ARGUMENTS

## Instructions

Run this at the start of each 2-week cycle. It checks what sprints already exist, creates the next one with correct dates and a goal, and hands off to `/notion-add-tasks` if tasks haven't been created yet for the targeted epic.

---

### Step 1 — Locate the workspace

Check `product-notes.md` for stored IDs:
```
<!-- notion-sprints-db-id: ... -->
<!-- notion-epics-db-id: ... -->
<!-- notion-tasks-db-id: ... -->
```

---

### Step 2 — Check existing sprints

```python
r = requests.post(f"https://api.notion.com/v1/databases/{sprints_db_id}/query",
                  headers=headers,
                  json={"sorts": [{"property": "Start Date", "direction": "descending"}]})
sprints = r.json()["results"]
```

From the results, determine:
- **Next sprint number**: highest existing number + 1 (e.g. if Sprint 3 exists, next is Sprint 4)
- **Last sprint end date**: use as the new sprint's start date if it's in the future; otherwise use today
- **Active sprint**: any sprint whose Start Date ≤ today ≤ End Date and Status = "Active"

If an active sprint already exists, warn:
> "Sprint {N} is still active (ends {date}). Do you want to close it and start Sprint {N+1}, or add tasks to the existing sprint?"

---

### Step 3 — Determine sprint dates

Default: 2-week sprint.

```python
from datetime import date, timedelta

start = max(last_sprint_end or date.today(), date.today())
end   = start + timedelta(days=13)  # 13 = end of day 14
```

If `$ARGUMENTS` contains a start date, use that instead. Otherwise confirm with the user:
> "Sprint {N}: {start} → {end} (2 weeks). Adjust? (press Enter to confirm)"

---

### Step 4 — Set the sprint goal

Query the Epics DB for epics that are Planning or Active and have no tasks yet (or few tasks):

```python
r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query",
                  headers=headers, json={"page_size": 100})
candidate_epics = [
    p for p in r.json()["results"]
    if p["properties"].get("Status", {}).get("select", {}).get("name") in ("Planning", "Active")
    and p["properties"].get("Task Count", {}).get("rollup", {}).get("number", 0) == 0
]
```

Present the candidates and ask:
> "Which epic(s) are you targeting this sprint? (choose one or more)"

Once selected, auto-generate a sprint goal from the epic names:
- 1 epic:  "Ship {epic name}"
- 2 epics: "Ship {epic 1} and {epic 2}"
- 3+:      "Complete Phase {X} epics: {comma list}"

Let the user override the goal if they want.

---

### Step 5 — Create the sprint

```python
r = requests.post("https://api.notion.com/v1/pages", headers=headers, json={
    "parent": {"database_id": sprints_db_id},
    "properties": {
        "Sprint":     {"title": [{"text": {"content": f"Sprint {next_num}"}}]},
        "Status":     {"select": {"name": "Active"}},
        "Start Date": {"date": {"start": start.isoformat()}},
        "End Date":   {"date": {"start": end.isoformat()}},
        "Goal":       {"rich_text": [{"text": {"content": sprint_goal}}]}
    }
})
sprint_id   = r.json()["id"]
sprint_name = f"Sprint {next_num}"
```

Also close the previous sprint if it was still Active:
```python
if prev_active_sprint:
    requests.patch(f"https://api.notion.com/v1/pages/{prev_active_sprint['id']}",
                   headers=headers, json={
                       "properties": {"Status": {"select": {"name": "Complete"}}}
                   })
```

---

### Step 6 — Check if targeted epics have tasks

For each epic chosen in Step 4:

```python
task_count = epic["properties"].get("Task Count", {}).get("rollup", {}).get("number", 0)
```

If `task_count == 0`:
> "E1.1: Authentication & Onboarding has no tasks yet. Running task breakdown now..."
> → Call `/notion-add-tasks` for that epic, passing `sprint_name` so tasks are auto-assigned to this sprint.

If tasks already exist but aren't assigned to this sprint, ask:
> "E1.2 has {N} existing tasks not yet assigned to {sprint_name}. Assign them? (yes / no)"

If yes:
```python
for task in unassigned_tasks:
    requests.patch(f"https://api.notion.com/v1/pages/{task['id']}",
                   headers=headers, json={
                       "properties": {"Sprint": {"relation": [{"id": sprint_id}]}}
                   })
```

---

### Step 7 — Report

```
SPRINT STARTED
==============
✓ Sprint 2 created
  Dates:  2026-07-28 → 2026-08-10  (2 weeks)
  Goal:   Ship E1.1 Authentication & Onboarding
  Status: Active

✓ Sprint 1 closed

Tasks:
  ✓ 9 new tasks created for E1.1 and assigned to Sprint 2
    (via /notion-add-tasks)

  OR

  ✓ 6 existing tasks assigned to Sprint 2

Next: Open Notion → Tasks DB → "Current Sprint" view to see your board.
```
