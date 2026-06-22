Import UI/UX screens from Claude Design into the Notion workspace: match each screen to its epic, embed screen links in the epic page body, detect features visible in the design that are missing from the PRD, and update Design Status on each epic.

$ARGUMENTS

## Instructions

Run this after a design round and before `/notion-add-tasks`. It ensures every epic has its screens attached and that the PRD reflects what was actually designed.

---

### Step 0 — Determine design source

Two modes are supported — use whichever is available:

**Mode A: Claude Design MCP** (preferred)
Check if claude_design MCP tools are available. If not connected:
> "The claude_design MCP connector is not connected. To set it up:
> 1. Run: `claude mcp add claude-design --transport http https://api.anthropic.com/v1/design/mcp`
> 2. Restart Claude Code
> 3. If prompted for auth, run `/design-login`"

**Mode B: Local handoff bundle** (fallback when MCP unavailable)
If `$ARGUMENTS` contains a local `.zip` or `.html` file path, or if a `design-handoff/` folder exists in the project directory, use that instead:
- Unzip the bundle if needed: `unzip "*.zip" -d design-handoff/`
- Read the primary HTML file (usually `Super Mobile App.html` or the file named in the README)
- Extract all screen IDs and labels from the `const screens = [...]` array in the file
- All screen labels become the screen inventory for the rest of the skill

If neither MCP nor local bundle is available, ask:
> "Please share the Claude Design project URL or drop the handoff .zip file path."

---

### Step 1 — Locate the workspace

Check `product-notes.md` for stored Notion IDs:
```
<!-- notion-epics-db-id: ... -->
<!-- notion-workspace-id: ... -->
```

Also check for a stored design project ID:
```
<!-- claude-design-project-id: ... -->
```

If `$ARGUMENTS` contains a Claude Design URL, extract the project ID from it:
- URL format: `https://claude.ai/design/p/{project_id}?file={filename}`

If no design project is known, ask:
> "What is the Claude Design project URL?"

---

### Step 2 — Import the design project

Use the claude_design MCP to import the project and list all screens/frames:

```python
# Import project
project = claude_design.import_project(url=design_url)
project_id = project["id"]

# List all screens/artboards
screens = claude_design.list_screens(project_id=project_id)
# Each screen: { id, name, url, preview_url, frame_name, last_modified }
```

Print a summary of what was found:
```
Design project: Super Mobile App
Screens found: 42
  - Login / OTP Entry
  - Society Selection
  - Home Dashboard (Resident)
  - Monthly Bill View
  ...
```

---

### Step 3 — Fetch epics from Notion

```python
r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query",
                  headers=headers, json={"page_size": 100})
epics = [{
    "id": p["id"],
    "name": p["properties"]["Epic"]["title"][0]["plain_text"],
    "design_status": p["properties"].get("Design Status", {}).get("select", {}).get("name", "Not Started"),
    "existing_design_url": p["properties"].get("Design URL", {}).get("url", "")
} for p in r.json()["results"]]
```

---

### Step 4 — Match screens to epics

Match each screen to an epic by name similarity. Use these rules:

- Strip epic codes (e.g. "E1.3: " prefix) before matching
- A screen belongs to an epic if its name contains keywords from the epic name, or the epic name describes the feature shown in the screen
- One screen can only belong to one epic (assign to the best match)
- Unmatched screens go into an "Unmatched" bucket — surface these to the user at the end

Example matching:
```
"Monthly Bill View"         → E1.5: Monthly Bill Generation
"Upload Payment Proof"      → E1.3: Dues & Payment Proof System
"Manager Approval Screen"   → E1.3: Dues & Payment Proof System
"Home Dashboard (Resident)" → E1.7: Home Screen (Adaptive Dashboard)
"OTP Login Screen"          → E1.1: Authentication & Onboarding
```

If a screen could match multiple epics, pick the more specific one. If it's genuinely ambiguous, add it to the "Needs Review" list and ask the user after the full matching pass.

---

### Step 5 — Read the PRD and detect missing features

For each epic, compare:
1. The screens matched to it (screen names, visible UI elements from design)
2. The PRD content for that epic (user stories, requirements)

Look for things present in the design that are NOT in the PRD:
- UI states not mentioned (empty state, error state, loading skeleton)
- Controls or filters that imply backend behaviour (sort, filter, search)
- New flows or screens not described in any user story
- Edge cases visible in the design (e.g. "maximum 3 files" shown in UI)

Compile a list of gaps per epic:
```
E1.3: Dues & Payment Proof System
  Design has: "Proof rejected — reason" screen
  PRD has: no mention of rejection reason being shown to resident
  → ADD to PRD: "Resident sees rejection reason when proof is rejected by manager"

E1.7: Home Screen (Adaptive Dashboard)
  Design has: "Quick Actions" row with 4 shortcut buttons
  PRD has: no mention of quick actions
  → ADD to PRD: "Home screen includes quick action shortcuts (Pay Dues, Request Maintenance, Chat, Documents)"
```

---

### Step 6 — Present the plan for confirmation

Show a summary before writing anything:

```
DESIGN SYNC PLAN
================
Project: Super Mobile App  (42 screens)

SCREEN → EPIC ASSIGNMENTS
  E1.1 Authentication & Onboarding     3 screens  Design Status: Dev Ready
  E1.3 Dues & Payment Proof System     6 screens  Design Status: In Review
  E1.5 Monthly Bill Generation         4 screens  Design Status: Dev Ready
  E1.7 Home Screen                     5 screens  Design Status: Dev Ready
  ...
  Unmatched screens: 2  (review needed)

PRD GAPS DETECTED  (8 items across 5 epics)
  E1.3: Proof rejection reason screen — not in PRD
  E1.7: Quick Actions row — not in PRD
  E2.1: "Minimum charge" warning label — not in PRD
  ... (full list)

ACTIONS:
  ✓ Embed screen links in 18 epic page bodies
  ✓ Update Design Status property on 18 epics
  ✓ Add Design URL property to Epics DB (if missing)
  ✓ Add 8 missing features to PRD and epic page bodies
  ✓ List 2 unmatched screens for manual review

Proceed? (yes / skip-prd-gaps / cancel)
```

---

### Step 7 — Add Design Status and Design URL properties to Epics DB (if missing)

```python
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Design Status": {
            "select": {
                "options": [
                    {"name": "Not Started", "color": "gray"},
                    {"name": "In Design",   "color": "yellow"},
                    {"name": "In Review",   "color": "orange"},
                    {"name": "Dev Ready",   "color": "blue"},
                    {"name": "Shipped",     "color": "green"}
                ]
            }
        },
        "Design URL": {"url": {}}
    }
})
```

---

### Step 7b — Identify missing epics

After matching screens to epics, look at the unmatched screens bucket. For each unmatched screen:
- Check if it describes a feature that should exist in the PRD phases but has no epic yet
- If yes → propose a new epic (name, which phase it belongs to, platform assignment)
- If it belongs to a future phase not yet planned → note it but do not create the epic
- If it's clearly cross-cutting (e.g. "Empty & Error States", "Upgrade / Pricing") → assign to the most relevant existing epic rather than creating a new one

Present proposed new epics to the user before creating them.

When creating new epics, assign the Platform relation using the same inference rules as `gen-notion-workspace`:
- Screen-heavy / UI-first → Mobile (KMM) + Backend
- Logic/data/API-only → Backend only
- Infra/DevOps/testing → Infrastructure + others as needed

### Step 8 — Write to Notion

For each epic with matched screens:

```python
for epic in matched_epics:
    props = {
        "Design Status": {"select": {"name": epic["design_status"]}},
    }
    # Assign Platform if not already set and Platforms DB exists
    if platforms_db_id and not epic.get("existing_platform"):
        props["Platform"] = {"relation": infer_platforms(epic["name"], platform_ids)}

    # Update properties
    requests.patch(f"https://api.notion.com/v1/pages/{epic['id']}", headers=headers, json={
        "properties": props
    })

    # Append Screens section to page body
    blocks = [
        divider(),
        h2("Screens"),
    ]
    for screen in epic["screens"]:
        blocks.append(bullet(f"{screen['name']} — {screen['url']}"))

    # If PRD gaps exist for this epic, append a section
    if epic.get("prd_gaps"):
        blocks += [
            divider(),
            h2("Design Additions (not yet in PRD)"),
            callout("These features appear in the design but were not in the original PRD. Add them to the PRD and include in task generation.", "⚠️"),
        ]
        for gap in epic["prd_gaps"]:
            blocks.append(bullet(gap))

    append_blocks(epic["id"], blocks)
```

Also update `product-notes.md` to store the design project ID:
```
<!-- claude-design-project-id: {project_id} -->
```

---

### Step 9 — Update the PRD

For each PRD gap confirmed by the user, append the missing feature to the relevant epic section in the PRD file:

```python
# In the PRD file, find the epic's section and append the gap as a new user story or requirement
```

Use the format already established in the PRD (user stories or bullet requirements depending on what's already there for that epic).

---

### Step 10 — Report

```
DESIGN SYNC COMPLETE
====================
✓ 42 screens imported from Super Mobile App
✓ 18 epics updated with screen links and Design Status
✓ 8 PRD gaps added to epic pages and PRD file
✓ Design URL property added to Epics database

Unmatched screens (assign manually in Notion):
  - "Onboarding Splash v2" — possibly E1.1 or E1.8
  - "Admin Settings Panel" — no matching epic found

Next: Run /notion-add-tasks on any Dev Ready epic to generate tasks.
```
