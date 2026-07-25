Add the following feature to this project's PRD and product notes:

$ARGUMENTS

## Instructions

### Step 1 — Locate the docs

Scan the working directory for:
- A **product plan / PRD file** (likely named `*product-plan*`, `*prd*`, or `*plan*`, with `.md` extension). This file contains user stories organised by role/epic.
- A **product notes file** (likely named `*product-notes*`, `*notes*`, or `*brief*`, with `.md` extension). This file contains prose bullets describing features by domain.

If either file is missing, create it with the appropriate scaffolding before proceeding.

### Step 2 — Determine where the feature belongs

Read both files to understand:
- What roles exist (e.g. Artist, Venue, Fan, Manager, Promoter)
- What epics / sections exist (e.g. Guestlist System, Karaoke Queue, Artist Features)
- Whether this feature belongs to an existing epic or needs a new one

### Step 3 — Add to the PRD (user stories format)

Add user stories under the correct role/epic section. If this is a new epic, add a new `####` heading with the next available number.

User story format — each bullet must follow this pattern exactly:
```
- As a **[role]**, I want [action], so that [outcome].
```

Rules:
- Use specific role names drawn from the PRD (`admin`, `user`, `manager`, `platform`, etc.) — read the roles from the file, do not invent them
- Bold the role with `**role**`
- Keep each story to a single, testable behaviour — split compound ideas into multiple stories
- Cover at least: the primary actor's core need, any secondary actor's perspective, and any platform-side enforcement logic
- End every story with a `so that [outcome]` clause — never omit it

### Step 4 — Add to the product notes (bullets format)

Add a new subsection (or extend an existing one) under the relevant domain in the product notes file.

Format:
```
### [Feature Name]
- [Concise capability description]
- [Another capability]
- ...
```

Rules:
- Use plain imperative bullets, no "As a..." framing
- Each bullet describes one capability or rule
- Group related behaviour under one heading
- Keep phrasing generic enough to apply to any project of this type — avoid hardcoding specific names unless they are product-specific terms

### Step 5 — Sync to Apple Notes (if a matching Note exists)

After updating the product notes `.md` file, check whether a corresponding Apple Note exists.

**Derive the expected Note title** from the product notes filename:
- Strip the directory path and `.md` extension
- Replace hyphens and underscores with spaces, keeping capitalisation (e.g. `my-app-notes.md` → `my app notes`, `MyApp-Product-Notes.md` → `MyApp - Product Notes`)

**Check for the Note** (search "Projects" folder first, then all notes as fallback):
```python
import subprocess
script = '''
tell application "Notes"
  -- check Projects folder first
  repeat with f in folders of default account
    if name of f is "Projects" then
      repeat with n in notes of f
        if name of n is "TITLE" then return "found"
      end repeat
    end if
  end repeat
  -- fallback: search all notes
  repeat with n in notes of default account
    if name of n is "TITLE" then return "found"
  end repeat
  return "not found"
end tell
'''
result = subprocess.run(['osascript', '-e', script.replace('TITLE', note_title)], capture_output=True, text=True)
```

If the Note is **not found**, ask the user: *"No Apple Note found titled '[title]'. Would you like me to create one in your Projects folder?"* If yes, create it (see `/create-project` Step 10 for the creation pattern). If no, skip silently.

If the Note **is found**, read its current body, build the HTML for the new section, and insert it at the correct position.

**Convert the new markdown section to Apple Notes HTML:**

| Markdown | Notes HTML |
|----------|-----------|
| `### Section Name` | `<div><b><h2>Section Name</h2></b></div>` |
| `- bullet` (first in group) | `<ul>\n<li>bullet</li>` |
| `- bullet` (subsequent) | `<li>bullet</li>` |
| end of bullet list | `</ul>` |
| blank line / section gap | `<div><br></div>` |
| `**bold**` | `<b>bold</b>` |
| `&` | `&amp` (no semicolon — Notes style) |
| `>` | `&gt` (no semicolon — Notes style) |
| `"` / `"` curly quotes | straight `"` |
| `—` em dash | `-` hyphen |

**Find the anchor** to insert after. The new section should appear immediately before the section that follows it in the `.md` file. Use exact string matching against the Note HTML:
```python
# anchor = last line of the preceding section's </ul> or </div>
patched = body.replace(anchor, anchor + new_html)
```

If the anchor is not found, append to the end of the body (before the final `</div>` if any, or just concatenate).

**Write back via osascript:**
```python
script = f'''
tell application "Notes"
  repeat with n in notes of default account
    if name of n is "{note_title}" then
      set body of n to "{escaped_body}"
      return "ok"
    end if
  end repeat
end tell
'''
subprocess.run(['osascript', '-e', script], ...)
```
Escape the body for AppleScript: replace `\` with `\\`, replace `"` with `\"`.

### Step 6 — Sync to Notion (if a page ID is stored)

After updating local files and Apple Notes, check whether `product-notes.md` contains a Notion page ID comment at the top:
```
<!-- notion-page-id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
```

If **no page ID is found**, ask: *"Would you like me to also update a Notion page with this feature? If yes, provide the Notion page ID."* If the user declines or doesn't provide one, skip this step.

If a page ID **is found** (or the user provides one):

**1. Get the Notion API key**

Check for `NOTION_API_KEY` (or `NOTION_TOKEN`) in environment. If missing, ask the user.

**2. Find the insertion point in the Notion page**

Retrieve existing page blocks:
```python
import requests, os

headers = {
    "Authorization": f"Bearer {os.environ['NOTION_API_KEY']}",
    "Notion-Version": "2022-06-28"
}
r = requests.get(f"https://api.notion.com/v1/blocks/{page_id}/children?page_size=100", headers=headers)
existing_blocks = r.json().get("results", [])
```

Scan existing blocks to find the section that should precede the new one (match by heading text). If found, the new blocks go after that block's `id` using an `after` parameter. If not found, append to end.

**3. Build and append the new blocks**

Convert the new markdown section to Notion blocks (see block format in `/create-project` Step 11). Append using:
```python
requests.patch(
    f"https://api.notion.com/v1/blocks/{page_id}/children",
    headers=headers,
    json={"children": new_blocks}
)
```

### Step 7 — Report what changed

After editing all files, report in one short paragraph:
- Which PRD section was updated (new epic or appended to existing)
- How many user stories were added
- Which product notes section was updated or created
- Whether a matching Apple Note was found and updated (or skipped if not found)
- Whether a Notion page was updated (or skipped)
