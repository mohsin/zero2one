Generate a complete Notion project management workspace from a product PRD or product notes file, following the 4-database best-practices structure.

$ARGUMENTS

## Instructions

### Step 0 — One-time Notion setup (ask the user to complete before proceeding)

Before any API calls can be made, the user must complete these three steps in Notion. Check whether they've already done this (e.g. `NOTION_API_KEY` is in environment and a parent page ID is known). If not, prompt them with the following:

---

**You'll need to do three things in Notion before I can build the workspace:**

**1. Create a Notion integration** (one-time, takes 2 minutes):
- Go to [https://www.notion.so/profile/integrations](https://www.notion.so/profile/integrations)
- Click **"New integration"**
- Name it (e.g. `My CLI` or `[Product Name] CLI`)
- Set type to **Internal**
- Click **Save**, then copy the **Internal Integration Secret** (starts with `ntn_...`)
- Run in your terminal: `export NOTION_API_KEY=ntn_your_key_here`

**2. Create a parent page in Notion** (where the workspace will live):
- Open Notion and create a new page (name it e.g. `[Product Name] Workspace`)
- This will be the container for all databases

**3. Share that page with your integration:**
- Open the page you just created
- Click **`...`** (top right) → **Connections** → search for your integration name → click to add
- Copy the page URL — it looks like: `https://app.notion.com/p/Page-Name-xxxxxxxxxx`

Once done, share the page URL with me and I'll build everything automatically.

---

Wait for the user to confirm before continuing to Step 1.

### Step 1 — Locate source files

If `$ARGUMENTS` is a file path, use that as the PRD. Otherwise scan the working directory for:
- A PRD/plan file: `*product-plan*`, `*prd*`, `*PRD*`, `*.md` with "Product Plan" or "PRD" in content
- A product notes file: `*product-notes*`, `*notes*`

Read both files. Extract:
- **Product name** (from the `# [Name]` heading)
- **User roles** (any section mentioning "Role", "User", "Admin", "Manager" etc.)
- **Phases** (Phase 1, Phase 2, etc. with descriptions and dates if present)
- **Epics / Feature areas** (major capability groups — headings under "User Stories by Role" or equivalent)
- **Open questions or decisions** from the PRD

### Step 2 — Get credentials and parent page

Check for `NOTION_API_KEY` (or `NOTION_TOKEN`) in environment. If missing, ask the user.

Ask the user:
> "Which Notion page should the workspace be created under? Provide the page URL or ID. (You can also type 'root' to create at workspace root level, but note that root-level databases can't be deleted via API.)"

Extract the 32-character page ID from the URL if needed. Format it as hyphenated `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.

Also ask:
> "Is this a solo project or a team project?"

- **Solo**: skip the Team database entirely; skip Assignee/Capacity on Tasks; skip Owner on Goals and Epics.
- **Team**: create the Team database (Name, Role, Email, Capacity, Notes); include Assignee, Capacity on Tasks; include Owner (people) on Goals and Epics.

```python
headers = {
    "Authorization": f"Bearer {os.environ.get('NOTION_API_KEY', '')}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json"
}
```

### Step 3 — Create the workspace root page

Create a root page to contain the entire workspace:

```python
import requests

payload = {
    "parent": {"type": "page_id", "page_id": parent_page_id},
    "properties": {
        "title": {"title": [{"type": "text", "text": {"content": f"{product_name} Workspace"}}]}
    },
    "children": [
        {"object": "block", "type": "heading_1", "heading_1": {
            "rich_text": [{"type": "text", "text": {"content": f"{product_name} — Project Management"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "Central workspace. All databases are linked. Start with the Home Dashboard."}}]
        }},
        {"object": "block", "type": "divider", "divider": {}},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "Databases"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "Goals → Epics → Tasks. All views set up manually in Notion UI after creation."}}]
        }}
    ]
}
r = requests.post("https://api.notion.com/v1/pages", headers=headers, json=payload)
workspace_page_id = r.json()["id"]
```

Save `workspace_page_id` — all databases are children of this page.

### Step 4 — Create Goals / OKRs database

Create the first database. Seed it with phases extracted from the PRD.

**Create database:**
```python
goals_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "title": [{"type": "text", "text": {"content": "Goals & OKRs"}}],
    "properties": {
        "Goal": {"title": {}},
        "Quarter": {"select": {"options": [
            {"name": "Q1", "color": "blue"}, {"name": "Q2", "color": "green"},
            {"name": "Q3", "color": "yellow"}, {"name": "Q4", "color": "orange"}
        ]}},
        "Status": {"select": {"options": [
            {"name": "On Track", "color": "green"}, {"name": "At Risk", "color": "yellow"},
            {"name": "Achieved", "color": "blue"}, {"name": "Missed", "color": "red"}
        ]}},
        "Start Date":  {"date": {}},
        "End Date":    {"date": {}},
        "Description": {"rich_text": {}},
        "Progress":    {"number": {"format": "percent"}}
        # Owner intentionally omitted — add only for team projects:
        # **( {"Owner": {"people": {}}} if is_team_project else {} )
    }
}).json()
goals_db_id = goals_db["id"]
```

**Seed from PRD phases** — each phase becomes a goal. For each record, generate a rich page body from the PRD content (see Step 4a below):
```python
for phase in phases:  # extracted from PRD
    page = requests.post("https://api.notion.com/v1/pages", headers=headers, json={
        "parent": {"database_id": goals_db_id},
        "properties": {
            "Goal": {"title": [{"text": {"content": phase["name"]}}]},
            "Status": {"select": {"name": "On Track"}},
            "Start Date": {"date": {"start": phase.get("start_date", "")}},
            "Description": {"rich_text": [{"text": {"content": phase.get("description", "")}}]}
        }
    }).json()
    goal_page_ids[phase["name"]] = page["id"]
    # Fill page body immediately after creation (see Step 4a)
    append_blocks(page["id"], generate_goal_body(phase, epics))
```

### Step 4a — Generate page body content from PRD

Every record created in Goals, Projects, and Sprints must have a rich page body — not just a title. Generate the body using content extracted from the PRD and product notes. Do not leave pages empty.

**Goal page body structure:**
```
[Callout: date range + status]
---
## Objective
[2-3 sentence summary of what this phase delivers]
---
## What's In Scope
- [bullet per major feature/capability]
---
## Success Metrics
- [bullet per measurable KPI from the PRD]
---
## Epics
- [bullet per epic in this phase]
---
## Tech Stack (if relevant to this phase)
- [bullets]
```

**Epic page body structure:**
```
[Callout: phase + priority + one-line purpose]
---
## Overview
[2-3 sentences on what this epic does and why it exists]
---
## [Main feature sections — varies by epic]
[Bullet lists covering: user flows, what each role sees/does, key rules]
---
## Technical Requirements
- [bullet per implementation constraint or architecture note from the PRD]
---
## Success Metric
[Quoted from PRD if present]
```

Use `PATCH /v1/blocks/{page_id}/children` to append blocks after creating each record:
```python
def append_blocks(page_id, blocks):
    for i in range(0, len(blocks), 100):
        requests.patch(f"{BASE}/blocks/{page_id}/children",
                       headers=headers, json={"children": blocks[i:i+100]})
```

Block helper functions:
```python
def h2(text): return {"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": text}}]}}
def p(text):  return {"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": text}}]}}
def bullet(text): return {"object": "block", "type": "bulleted_list_item", "bulleted_list_item": {"rich_text": [{"type": "text", "text": {"content": text}}]}}
def divider(): return {"object": "block", "type": "divider", "divider": {}}
def callout(text, emoji="💡"): return {"object": "block", "type": "callout", "callout": {"rich_text": [{"type": "text", "text": {"content": text}}], "icon": {"type": "emoji", "emoji": emoji}}}
```

Generate the body content by reading the PRD/notes deeply — every section, user story, technical note, and success metric is a candidate for inclusion. The richer the page body, the more useful Notion is as a working document.

### Step 5 — Create Epics database

```python
epics_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "title": [{"type": "text", "text": {"content": "Epics"}}],
    "properties": {
        "Epic": {"title": {}},
        "Status": {"select": {"options": [
            {"name": "Planning", "color": "gray"}, {"name": "Active", "color": "blue"},
            {"name": "On Hold", "color": "yellow"}, {"name": "Done", "color": "green"}
        ]}},
        "Priority": {"select": {"options": [
            {"name": "P0", "color": "red"}, {"name": "P1", "color": "orange"},
            {"name": "P2", "color": "yellow"}, {"name": "P3", "color": "gray"}
        ]}},
        "Start Date":   {"date": {}},
        "Due Date":     {"date": {}},
        "Description":  {"rich_text": {}}
    }
}).json()
epics_db_id = epics_db["id"]
```

**Add relation to Goals** (after both databases exist):
```python
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Goal": {
            "relation": {
                "database_id": goals_db_id,
                "single_property": {}
            }
        }
    }
})
```

**Seed from PRD epics** — each epic becomes a record. If Platforms DB was created (Step 8), also assign the Platform relation based on the epic's nature:

Platform assignment rules:
- If the epic is primarily screen/UI work → Mobile
- If the epic is primarily API, billing, data, or backend logic → Backend
- If the epic is testing, deployment, or CI/CD → Infrastructure
- Most feature epics need both Mobile + Backend — assign both

```python
def infer_platforms(epic_name, platform_ids):
    name = epic_name.lower()
    assigned = []
    # Infrastructure signals
    if any(w in name for w in ["launch", "testing", "qa", "deploy", "ci", "infra"]):
        assigned.append(platform_ids.get("Infrastructure (DevOps)"))
    # Backend-only signals
    if any(w in name for w in ["billing", "calculation", "payment", "tracking", "report", "dashboard", "api"]):
        assigned.append(platform_ids.get("Backend (OctoberCMS/Laravel)"))
    # Mobile-only signals (pure UI)
    if any(w in name for w in ["home screen", "coming soon", "geofencing"]):
        assigned.append(platform_ids.get("Mobile (KMM)"))
    # Default: both Mobile + Backend for feature epics
    if not assigned:
        assigned = [platform_ids.get("Mobile (KMM)"), platform_ids.get("Backend (OctoberCMS/Laravel)")]
    return [{"id": pid} for pid in assigned if pid]

for epic in epics:  # extracted from PRD
    # Description: derive from PRD if not explicitly set.
    # A good description is one sentence summarising what the epic delivers and its key scope.
    # Never leave it blank — scan the PRD's feature section for that epic and write a tight summary.
    description = epic.get("description") or derive_description_from_prd(epic["name"])

    props = {
        "Epic":        {"title": [{"text": {"content": epic["name"]}}]},
        "Status":      {"select": {"name": "Planning"}},
        "Priority":    {"select": {"name": epic.get("priority", "P2")}},
        "Goal":        {"relation": [{"id": epic["goal_id"]}]},
        "Description": {"rich_text": [{"text": {"content": description}}]}
    }
    if platforms_db_id:
        props["Platform"] = {"relation": infer_platforms(epic["name"], platform_ids)}

    requests.post("https://api.notion.com/v1/pages", headers=headers, json={
        "parent": {"database_id": epics_db_id},
        "properties": props
    })
```

Also add `Design Status` property to Epics DB at creation time (it will be populated later by `/notion-design-sync`):

```python
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Design Status": {
            "select": {"options": [
                {"name": "Not Started", "color": "gray"},
                {"name": "In Design",   "color": "yellow"},
                {"name": "In Review",   "color": "orange"},
                {"name": "Dev Ready",   "color": "blue"},
                {"name": "Shipped",     "color": "green"}
            ]}
        }
    }
})
```

After seeding, audit for blanks and fill any missed descriptions:
```python
r = requests.post(f"https://api.notion.com/v1/databases/{epics_db_id}/query", headers=headers, json={"page_size": 100})
for page in r.json()["results"]:
    name = page["properties"]["Epic"]["title"][0]["plain_text"]
    desc = page["properties"].get("Description", {}).get("rich_text", [])
    if not desc:
        filled = derive_description_from_prd(name)
        requests.patch(f"https://api.notion.com/v1/pages/{page['id']}", headers=headers, json={
            "properties": {"Description": {"rich_text": [{"text": {"content": filled}}]}}
        })
        print(f"  Filled description: {name}")
```

### Step 6 — Create Tasks database

```python
tasks_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "title": [{"type": "text", "text": {"content": "Tasks"}}],
    "properties": {
        "Task": {"title": {}},
        "Status": {"select": {"options": [
            {"name": "Backlog", "color": "gray"}, {"name": "To Do", "color": "default"},
            {"name": "In Progress", "color": "blue"}, {"name": "In Review", "color": "yellow"},
            {"name": "Done", "color": "green"}, {"name": "Blocked", "color": "red"}
        ]}},
        "Priority": {"select": {"options": [
            {"name": "Urgent", "color": "red"}, {"name": "High", "color": "orange"},
            {"name": "Normal", "color": "yellow"}, {"name": "Low", "color": "gray"}
        ]}},
        "Type": {"select": {"options": [
            {"name": "Feature", "color": "blue"}, {"name": "Bug", "color": "red"},
            {"name": "Chore", "color": "gray"}, {"name": "Research", "color": "purple"}
        ]}},
        "Effort": {"select": {"options": [
            {"name": "XS", "color": "gray"}, {"name": "S", "color": "blue"},
            {"name": "M", "color": "green"}, {"name": "L", "color": "yellow"},
            {"name": "XL", "color": "red"}
        ]}},
        "Assignee": {"people": {}},
        "Due Date": {"date": {}},
        "Summary": {"rich_text": {}},
        "Acceptance Criteria": {"rich_text": {}}
    }
}).json()
tasks_db_id = tasks_db["id"]
```

**Add relation to Epics:**
```python
requests.patch(f"https://api.notion.com/v1/databases/{tasks_db_id}", headers=headers, json={
    "properties": {
        "Epic": {
            "relation": {
                "database_id": epics_db_id,
                "single_property": {}
            }
        }
    }
})
```

**Add an explicit Tasks relation on Epics first** (rollups require a relation property on the same database):

`single_property` relations are one-directional — the target database does NOT get an automatic backlink property. So to add a rollup on Epics counting its Tasks, you must first add a separate relation on Epics pointing to Tasks:

```python
# Add Tasks relation on Epics side
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Tasks": {
            "relation": {
                "database_id": tasks_db_id,
                "single_property": {}
            }
        }
    }
})

# Then add the rollup using that relation
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Task Count": {
            "rollup": {
                "relation_property_name": "Tasks",
                "rollup_property_name": "Task",
                "function": "count"
            }
        }
    }
})
```

### Step 7 — Create Sprints database

```python
sprints_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "title": [{"type": "text", "text": {"content": "Sprints"}}],
    "properties": {
        "Sprint": {"title": {}},
        "Status": {"select": {"options": [
            {"name": "Planning", "color": "gray"}, {"name": "Active", "color": "blue"},
            {"name": "Complete", "color": "green"}
        ]}},
        "Goal": {"rich_text": {}},
        "Start Date": {"date": {}},
        "End Date": {"date": {}}
    }
}).json()
sprints_db_id = sprints_db["id"]
```

**Link Sprints to Tasks:**
```python
requests.patch(f"https://api.notion.com/v1/databases/{tasks_db_id}", headers=headers, json={
    "properties": {
        "Sprint": {
            "relation": {
                "database_id": sprints_db_id,
                "single_property": {}
            }
        }
    }
})
```

### Step 8 — Create Platforms database (auto-detected from PRD)

Scan the PRD and product notes for a tech stack section. Look for distinct runtime platforms — things like Backend, Mobile, Infrastructure, Web, Admin Panel. A platform is a separate codebase or deployment target, not a library or tool.

Signals to look for:
- Named sections: "Tech Stack", "Architecture", "Platforms", "Frontend / Backend / Mobile"
- Distinct framework groupings: e.g. "Laravel backend", "KMM mobile app", "DigitalOcean infrastructure"
- Separate teams or owners per platform

If platforms are found, extract each one with its name, owner, tech stack, and key deliverables from the PRD. **Do not ask the user** — auto-detect and proceed. Only skip this step if the PRD has no discernible platform structure (e.g. a simple single-codebase tool).

Create the Platforms database and add a Platform relation on the Epics database:

```python
platforms_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "title": [{"type": "text", "text": {"content": "Platforms"}}],
    "properties": {
        "Platform":         {"title": {}},
        "Status":           {"select": {"options": [
            {"name": "Planning", "color": "gray"}, {"name": "Active", "color": "blue"},
            {"name": "On Hold",  "color": "yellow"}, {"name": "Complete", "color": "green"}
        ]}},
        "Owner":            {"rich_text": {}},
        "Tech Stack":       {"rich_text": {}},
        "Key Deliverables": {"rich_text": {}},
        "Notes":            {"rich_text": {}}
    }
}).json()
platforms_db_id = platforms_db["id"]

# Add Platform relation on Epics so epics can be tagged by platform
requests.patch(f"https://api.notion.com/v1/databases/{epics_db_id}", headers=headers, json={
    "properties": {
        "Platform": {
            "relation": {"database_id": platforms_db_id, "single_property": {}}
        }
    }
})
```

Seed one record per platform from the PRD tech stack section. Add a rich page body to each: Overview, Tech Stack (bullets), Key Deliverables (bullets), Dependencies (bullets). Skip this step if the project has no distinct platforms.

### Step 9 — Create Team Members database (team projects only)

Skip this step entirely if the user said **solo**. Only run if **team**.

```python
if is_team_project:
    team_db = requests.post("https://api.notion.com/v1/databases", headers=headers, json={
        "parent": {"type": "page_id", "page_id": workspace_page_id},
        "title": [{"type": "text", "text": {"content": "Team"}}],
        "properties": {
            "Name": {"title": {}},
            "Role": {"select": {"options": [
                {"name": "Engineering", "color": "blue"}, {"name": "Design", "color": "purple"},
                {"name": "Product", "color": "green"}, {"name": "QA", "color": "yellow"},
                {"name": "Marketing", "color": "orange"}
            ]}},
            "Email": {"email": {}},
            "Capacity (pts/sprint)": {"number": {"format": "number"}},
            "Notes": {"rich_text": {}}
        }
    }).json()
    team_db_id = team_db["id"]
```

### Step 9 — Create Home Dashboard page

Create a dashboard page inside the workspace that links to all databases via embedded views. Note: linked database views must be wired up manually in the Notion UI — the page scaffolding is created here.

```python
requests.post("https://api.notion.com/v1/pages", headers=headers, json={
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "properties": {
        "title": {"title": [{"type": "text", "text": {"content": "🏠 Home Dashboard"}}]}
    },
    "children": [
        {"object": "block", "type": "callout", "callout": {
            "rich_text": [{"type": "text", "text": {"content": "This is your daily entry point. Wire up linked database views for each section in the Notion UI."}}],
            "icon": {"type": "emoji", "emoji": "💡"}
        }},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "🔴 Overdue Tasks"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "[Add linked view: Tasks — filter: Assignee = Me, Status ≠ Done, Due < Today]"}}]
        }},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "📅 Due This Week"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "[Add linked view: Tasks — filter: Assignee = Me, Due = Next 7 days]"}}]
        }},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "🏃 Active Epics"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "[Add linked view: Epics — filter: Status = Active]"}}]
        }},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "🎯 Goals This Quarter"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "[Add linked view: Goals — filter: Status = On Track or At Risk]"}}]
        }},
        {"object": "block", "type": "heading_2", "heading_2": {
            "rich_text": [{"type": "text", "text": {"content": "🏷️ Current Sprint"}}]
        }},
        {"object": "block", "type": "paragraph", "paragraph": {
            "rich_text": [{"type": "text", "text": {"content": "[Add linked view: Sprints — filter: Status = Active]"}}]
        }}
    ]
})
```

### Step 10 — Schedule dates (hand off to `/notion-schedule`)

After the workspace is created and all IDs are stored in `product-notes.md`, tell the user:

```
WORKSPACE CREATED: {product_name}
=========================================
✓ Root page: {product_name} Workspace
✓ Goals & OKRs database ({N} goals seeded from PRD phases)
✓ Epics database ({N} epics seeded from PRD feature areas)
✓ Tasks database (empty — ready for backlog grooming)
✓ Sprints database (Sprint 1 created)
✓ Platforms database ({N} platforms seeded from PRD tech stack)  ← if applicable
✓ Home Dashboard page

Relations wired:
  Epics → Goals (via "Goal" property)
  Epics → Platforms (via "Platform" property)
  Tasks → Epics (via "Epic" property)
  Tasks → Sprints (via "Sprint" property)
  Epics.Task Count (rollup of linked Tasks)

TO COMPLETE SETUP IN NOTION UI (can't do via API):
  1. Open Goals db → Add views: Table (default), Board by Status, Timeline
  2. Open Epics db → Add views: Table, Kanban by Status, Timeline (Roadmap), filtered "Active"
  3. Open Tasks db → Add views: Kanban by Status, filtered "My Tasks", "Today", "Overdue", "Backlog"
  4. Open Platforms db → Add views: Board by Status, Gallery
  5. Open Home Dashboard → add linked database views for each section

Stored IDs (saved to product-notes.md):
<!-- notion-workspace-id: {workspace_page_id} -->
<!-- notion-goals-db-id: {goals_db_id} -->
<!-- notion-epics-db-id: {epics_db_id} -->
<!-- notion-tasks-db-id: {tasks_db_id} -->
<!-- notion-sprints-db-id: {sprints_db_id} -->
<!-- notion-platforms-db-id: {platforms_db_id} -->

NEXT: Run /notion-schedule to set start and end dates on all phases.
When designs are ready, run /notion-design-sync to attach screens to epics and detect PRD gaps.
Then during sprint planning, run /notion-add-tasks to break down an epic into tasks.
```

Then immediately invoke `/notion-schedule` to handle date planning — it will ask the user for a start date and any fixed deadlines, estimate the rest, confirm the full schedule, and write all dates in one pass.

Store the IDs in `product-notes.md` as comments:
```
<!-- notion-workspace-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- notion-goals-db-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- notion-epics-db-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- notion-tasks-db-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- notion-sprints-db-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- notion-platforms-db-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

### Key API patterns (reference)

**Creating a database inside a page:**
```python
requests.post("https://api.notion.com/v1/databases", headers=headers, json={
    "parent": {"type": "page_id", "page_id": parent_id},
    "title": [{"type": "text", "text": {"content": "DB Name"}}],
    "properties": { ... }
})
```

**Adding a relation (CORRECT format — no "type" key):**
```python
requests.patch(f"https://api.notion.com/v1/databases/{db_id}", headers=headers, json={
    "properties": {
        "RelationName": {
            "relation": {
                "database_id": target_db_id,
                "single_property": {}
            }
        }
    }
})
```

**Adding a rollup:**
```python
{
    "rollup": {
        "relation_property_name": "RelationPropertyName",
        "rollup_property_name": "PropertyNameInTargetDB",
        "function": "count"
    }
}
```

**Creating a record in a database:**
```python
requests.post("https://api.notion.com/v1/pages", headers=headers, json={
    "parent": {"database_id": db_id},
    "properties": {
        "Name": {"title": [{"text": {"content": "Record title"}}]},
        "Status": {"select": {"name": "In Progress"}},
        "Due Date": {"date": {"start": "2026-07-01"}},
        "Notes": {"rich_text": [{"text": {"content": "Some text"}}]}
    }
})
```

**Deleting a page:**
```python
requests.patch(f"https://api.notion.com/v1/pages/{page_id}", headers=headers, json={"in_trash": True})
# Note: "in_trash" not "archived". Workspace-root pages and teamspace pages cannot be deleted via API.
```
