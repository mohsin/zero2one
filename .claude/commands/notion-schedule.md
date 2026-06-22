Set the start and end dates on all Goals (phases) in a Notion workspace, then cascade due dates down to linked Projects.

$ARGUMENTS

## Instructions

This skill is called after `/gen-notion-workspace` has created the workspace structure. It handles one thing: dates. It asks the right questions, confirms before writing, then fills in dates via the Notion API.

---

### Step 1 — Locate the workspace

Check whether `product-notes.md` contains stored Notion IDs:
```
<!-- notion-workspace-id: ... -->
<!-- notion-goals-db-id: ... -->
<!-- notion-epics-db-id: ... -->
<!-- notion-sprints-db-id: ... -->
```

If found, use them. If not, ask the user:
> "What is your Notion workspace page URL or Goals database URL?"

---

### Step 2 — Fetch existing goals (phases)

Query the Goals & OKRs database to get all goal names, current statuses, and any existing dates:

```python
import requests, os

headers = {
    "Authorization": f"Bearer {os.environ['NOTION_API_KEY']}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json"
}

r = requests.post(f"https://api.notion.com/v1/databases/{goals_db_id}/query",
                  headers=headers, json={})
goals = [{
    "id": p["id"],
    "name": p["properties"]["Goal"]["title"][0]["plain_text"],
    "start": p["properties"].get("Start Date", {}).get("date", {}).get("start"),
    "end": p["properties"].get("End Date", {}).get("date", {}).get("start"),
    "status": p["properties"].get("Status", {}).get("select", {}).get("name")
} for p in r.json()["results"]]
```

---

### Step 3 — Ask the user about scheduling intent

Present the goals list and ask two questions:

**Question A — Start date:**
> "When do you want to start Phase 1 / the first goal?
> - Type a date (e.g. 2026-08-01)
> - Type 'today' or 'now'
> - Type '3 weeks' or '2 months' for a relative date
> - Press Enter to skip (keep existing or leave blank)"

**Question B — Do you have target end dates?**
> "Do you have fixed deadlines for any of the phases?
> Options:
> a) Yes — I'll give you a target end date for each phase
> b) No — estimate based on scope (I'll review before you write anything)
> c) Mix — some phases have deadlines, others are flexible"

If they choose (a) or (c), ask per-phase:
> "Phase 2: Financial Operations — target end date? (or 'estimate')"

---

### Step 4 — Estimate end dates for flexible phases

For phases where the user said "estimate" or chose option (b), generate a guesstimate based on:
- Number of epics in that phase (query Projects DB, filter by linked Goal)
- Complexity signal from epic names (Testing/Polish epics = faster; Auth/Billing = longer)
- Default rhythm: 1.5–2 weeks per epic, minimum 6 weeks per phase, maximum 16 weeks

```python
def estimate_weeks(epic_count, has_complex_epics):
    base = epic_count * 1.5
    if has_complex_epics:
        base *= 1.2
    return max(6, min(16, round(base)))
```

Phases cascade: Phase 2 starts the day Phase 1 ends. Phase 3 starts the day Phase 2 ends.

---

### Step 5 — Present the proposed schedule for confirmation

Show the full schedule before writing anything:

```
PROPOSED SCHEDULE
=================
Phase 1: Resident Experience & Dues
  Start:  2026-07-14
  End:    2026-10-06   (~12 weeks, 10 epics — estimated)

Phase 2: Financial Operations & Work Orders
  Start:  2026-10-06
  End:    2026-12-15   (~10 weeks, 6 epics — estimated)

Phase 3: Community & Facilities
  Start:  2026-12-15
  End:    2027-02-23   (~10 weeks, 4 epics — estimated)

Sprint 1
  Start:  2026-07-14
  End:    2026-07-28   (2-week default)

Confirm? (yes / adjust / cancel)
```

If the user wants to adjust, prompt per-phase. Do not write until confirmed.

---

### Step 6 — Write dates to Notion

For each goal, update Start Date and End Date. If End Date property doesn't exist on the Goals database, add it first:

```python
# Add End Date property if missing
requests.patch(f"https://api.notion.com/v1/databases/{goals_db_id}",
               headers=headers, json={"properties": {"End Date": {"date": {}}}})

# Update each goal
for goal in goals:
    requests.patch(f"https://api.notion.com/v1/pages/{goal['id']}",
                   headers=headers, json={
                       "properties": {
                           "Start Date": {"date": {"start": goal["new_start"]}},
                           "End Date": {"date": {"start": goal["new_end"]}}
                       }
                   })
```

For each epic linked to a goal, set its Start Date and Due Date proportionally within the phase window:
- Divide the phase duration evenly across epics in order
- P0 epics get earlier slots; P1/P2 get later slots

```python
# Get epics for this phase
r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query",
                  headers=headers, json={
                      "filter": {"property": "Goal", "relation": {"contains": goal["id"]}}
                  })
epics = r.json()["results"]
# Sort by priority (P0 first)
# Assign date windows proportionally
```

For Sprint 1, set Start Date = Phase 1 start, End Date = start + 14 days.

---

### Step 7 — Report

```
SCHEDULE SET
============
✓ Phase 1: Jul 14 → Oct 6  (12 weeks)
✓ Phase 2: Oct 6 → Dec 15  (10 weeks)
✓ Phase 3: Dec 15 → Feb 23 (10 weeks)
✓ Sprint 1: Jul 14 → Jul 28
✓ 20 projects updated with proportional date windows

To reschedule if you miss a start date, run /notion-reschedule.
```
