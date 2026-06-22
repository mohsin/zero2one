Shift the schedule of a Notion workspace forward when a phase start date has been missed. Cascades the change to all downstream phases, projects, and sprints.

$ARGUMENTS

## Instructions

Run this when a phase didn't start on time and you need to push all downstream dates forward without rebuilding the entire workspace.

---

### Step 1 — Locate the workspace

Check `product-notes.md` for stored Notion IDs:
```
<!-- notion-workspace-id: ... -->
<!-- notion-goals-db-id: ... -->
<!-- notion-epics-db-id: ... -->
<!-- notion-sprints-db-id: ... -->
```

If not found, ask for the Goals database URL or ID.

---

### Step 2 — Fetch current schedule

Query Goals & OKRs for all phases with their current Start Date, End Date, and Status:

```python
r = requests.post(f"https://api.notion.com/v1/databases/{goals_db_id}/query",
                  headers=headers, json={"sorts": [{"property": "Start Date", "direction": "ascending"}]})
goals = [{
    "id": p["id"],
    "name": p["properties"]["Goal"]["title"][0]["plain_text"],
    "start": p["properties"].get("Start Date", {}).get("date", {}).get("start"),
    "end": p["properties"].get("End Date", {}).get("date", {}).get("start"),
    "status": p["properties"].get("Status", {}).get("select", {}).get("name")
} for p in r.json()["results"]]
```

---

### Step 3 — Ask what changed

> "Which phase are you rescheduling, and what is the new start date?"

Example response: "Phase 1, starting August 4"

Identify:
- `affected_phase` — the phase being moved
- `new_start` — the new start date
- `slip_days` — difference between old start and new start (in days)

Only phases that haven't started yet (Status = Planning or On Track, not Active/Achieved) are eligible to reschedule. If the user tries to move an Active phase, warn them:
> "Phase 1 is marked Active — rescheduling it will also shift all its downstream phases. Are you sure?"

---

### Step 4 — Compute cascaded new dates

All phases at or after the affected phase get shifted by `slip_days`:

```python
from datetime import datetime, timedelta

def shift(date_str, days):
    if not date_str:
        return None
    d = datetime.strptime(date_str, "%Y-%m-%d")
    return (d + timedelta(days=days)).strftime("%Y-%m-%d")

# Find index of affected phase
idx = next(i for i, g in enumerate(goals) if affected_phase in g["name"])

# Shift all phases from idx onwards
for goal in goals[idx:]:
    goal["new_start"] = shift(goal["start"], slip_days)
    goal["new_end"] = shift(goal["end"], slip_days)
```

Phases before the affected one are unchanged.

Also shift Sprint dates: any sprint whose Start Date falls within the shifted phase window gets moved by the same `slip_days`.

---

### Step 5 — Present the new schedule for confirmation

Show before/after for all affected items:

```
RESCHEDULE PLAN  (slip: +18 days)
==================================
                    BEFORE          AFTER
Phase 1 Start:      Jul 14    →     Aug 01
Phase 1 End:        Oct 06    →     Oct 24
Phase 2 Start:      Oct 06    →     Oct 24
Phase 2 End:        Dec 15    →     Jan 02
Phase 3 Start:      Dec 15    →     Jan 02
Phase 3 End:        Feb 23    →     Mar 13

Sprint 1 Start:     Jul 14    →     Aug 01
Sprint 1 End:       Jul 28    →     Aug 15

Confirm? (yes / adjust / cancel)
```

Do not write until the user confirms.

---

### Step 6 — Write updated dates

```python
for goal in goals[idx:]:
    requests.patch(f"https://api.notion.com/v1/pages/{goal['id']}",
                   headers=headers, json={
                       "properties": {
                           "Start Date": {"date": {"start": goal["new_start"]}},
                           "End Date": {"date": {"start": goal["new_end"]}}
                       }
                   })
```

Then update project due dates within each shifted phase:

```python
for goal in goals[idx:]:
    r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query",
                      headers=headers, json={
                          "filter": {"property": "Goal", "relation": {"contains": goal["id"]}}
                      })
    for epic in r.json()["results"]:
        old_start = epic["properties"].get("Start Date", {}).get("date", {}).get("start")
        old_due = epic["properties"].get("Due Date", {}).get("date", {}).get("start")
        requests.patch(f"https://api.notion.com/v1/pages/{epic['id']}",
                       headers=headers, json={
                           "properties": {
                               "Start Date": {"date": {"start": shift(old_start, slip_days)}},
                               "Due Date": {"date": {"start": shift(old_due, slip_days)}}
                           }
                       })
```

Then update sprints:

```python
r = requests.post(f"https://api.notion.com/v1/databases/{sprints_db_id}/query",
                  headers=headers, json={})
for sprint in r.json()["results"]:
    sprint_start = sprint["properties"].get("Start Date", {}).get("date", {}).get("start")
    # Only shift sprints that start on/after the affected phase's old start
    if sprint_start and sprint_start >= goals[idx]["start"]:
        old_end = sprint["properties"].get("End Date", {}).get("date", {}).get("start")
        requests.patch(f"https://api.notion.com/v1/pages/{sprint['id']}",
                       headers=headers, json={
                           "properties": {
                               "Start Date": {"date": {"start": shift(sprint_start, slip_days)}},
                               "End Date": {"date": {"start": shift(old_end, slip_days)}}
                           }
                       })
```

---

### Step 7 — Report

```
RESCHEDULE COMPLETE  (+18 days)
================================
✓ Phase 1:  Jul 14 → Aug 01  (end: Oct 24)
✓ Phase 2:  Oct 06 → Oct 24  (end: Jan 02)
✓ Phase 3:  Dec 15 → Jan 02  (end: Mar 13)
✓ 20 projects shifted
✓ 1 sprint shifted

Unchanged: 0 phases (all were downstream of the affected phase)
```
