Sync a local product notes `.md` file with its matching Notion page — pushing new sections from the file into Notion, and optionally pulling Notion-only content back to the file.

$ARGUMENTS

## Instructions

### Step 1 — Locate the product notes file

If `$ARGUMENTS` is a file path, use that. Otherwise scan the working directory for a file matching `*product-notes*`, `*notes*`, or `*brief*` with a `.md` extension. If multiple matches, prefer the one whose name most closely matches the project name.

### Step 2 — Get the Notion page ID

Check whether the file contains a stored page ID at the top:
```
<!-- notion-page-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

If found, use it. If not found, ask the user:
> "No Notion page ID found in this file. Provide a Notion page ID or URL to sync with, or type 'new' to create a new Notion page."

If the user provides a URL, extract the 32-character ID from the end (last path segment, strip hyphens if needed). If the user says 'new', jump to Step 6 (Create new page) before continuing.

### Step 3 — Get Notion credentials

Check for `NOTION_API_KEY` (or `NOTION_TOKEN`) in environment variables. If missing, ask the user for the integration token before proceeding.

```python
headers = {
    "Authorization": f"Bearer {os.environ.get('NOTION_API_KEY', '')}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json"
}
```

### Step 4 — Fetch existing Notion blocks

Retrieve all blocks from the Notion page, handling pagination:

```python
import requests

def get_all_blocks(page_id, headers):
    blocks, cursor = [], None
    while True:
        params = {"page_size": 100}
        if cursor:
            params["start_cursor"] = cursor
        r = requests.get(
            f"https://api.notion.com/v1/blocks/{page_id}/children",
            headers=headers, params=params
        )
        data = r.json()
        blocks.extend(data.get("results", []))
        if not data.get("has_more"):
            break
        cursor = data.get("next_cursor")
    return blocks
```

Extract section headings from the block list. A "section" is a `heading_2` or `heading_3` block plus all non-heading blocks that follow it until the next heading.

### Step 5 — Compare sections

Parse the local `.md` file into sections (a `##` or `###` heading plus its content). Compare against the Notion sections by heading text.

**Sections in `.md` but not in Notion** → push to Notion
**Sections in Notion but not in `.md`** → offer to pull into `.md`
**Sections in both** → no action (content may have diverged intentionally)

Present the diff before making any changes:

```
SYNC PLAN for "[product name]" ↔ Notion
=========================================
→ PUSH to Notion (in .md, missing from page):
  + ### Notifications  (3 bullets)
  + ### Audit Log  (2 bullets)

← PULL to .md (in Notion, missing from file):
  + ### Dark Mode  (1 bullet)

= No changes (present in both):
  Overview, User Roles, Billing, ...

Proceed? (yes / skip push / skip pull / cancel)
```

Wait for user confirmation before writing anything.

### Step 6 — Push new sections to Notion

Convert each new markdown section to Notion block objects:

| Markdown | Notion block type |
|----------|------------------|
| `## Heading` | `heading_2` |
| `### Heading` | `heading_3` |
| `- bullet` | `bulleted_list_item` |
| `1. item` | `numbered_list_item` |
| Prose paragraph | `paragraph` |
| blank line | _(skip)_ |
| `**bold**` inline | rich_text with `"bold": true` annotation |

```python
def make_block(block_type, text):
    return {
        "object": "block",
        "type": block_type,
        block_type: {
            "rich_text": [{"type": "text", "text": {"content": text}}]
        }
    }
```

Append blocks in batches of up to 100:
```python
def append_blocks(page_id, blocks, headers):
    for i in range(0, len(blocks), 100):
        requests.patch(
            f"https://api.notion.com/v1/blocks/{page_id}/children",
            headers=headers,
            json={"children": blocks[i:i+100]}
        )
```

If the user chose **skip push**, skip this step.

### Step 7 — Pull Notion-only sections to `.md`

For each section to pull, convert Notion blocks to markdown:

| Notion block type | Markdown |
|------------------|---------|
| `heading_1` | `# text` |
| `heading_2` | `## text` |
| `heading_3` | `### text` |
| `bulleted_list_item` | `- text` |
| `numbered_list_item` | `1. text` |
| `paragraph` | `text` (prose) |
| empty `paragraph` | blank line |
| `bold` annotation | `**text**` |

Insert the pulled section at the correct position in the `.md` file, immediately before the section that follows it. If the position cannot be determined, append to end of file.

If the user chose **skip pull**, skip this step.

### Step 8 — Store page ID (if not already stored)

If the page ID was not in the file at the start, prepend it now:
```
<!-- notion-page-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

This allows future syncs to run without prompting.

### Step 9 — Report

```
SYNC COMPLETE for "[product name]" ↔ Notion
=============================================
→ Pushed to Notion:  2 sections (+8 bullets)
← Pulled to .md:     1 section (+2 bullets)
= Unchanged:         12 sections
```

If nothing changed: *"Already in sync — no changes made."*
