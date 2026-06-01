Sync a local product notes `.md` file with its matching Apple Note — pushing additions from the file into Notes, and pulling anything in Notes that is missing from the file.

$ARGUMENTS

## Instructions

### Step 1 — Locate the product notes file

If `$ARGUMENTS` is a file path, use that. Otherwise scan the working directory for a file matching `*product-notes*`, `*notes*`, or `*brief*` with a `.md` extension. If multiple matches, prefer the one whose name most closely matches the project name.

### Step 2 — Derive the Apple Note title

From the filename (no path, no `.md`), derive the expected Note title:
- Replace hyphens and underscores with spaces
- Preserve capitalisation as-is (e.g. `my-app-notes.md` → `my app notes`, `MyApp-Product-Notes.md` → `MyApp - Product Notes`)

### Step 3 — Find the Apple Note

Search the "Projects" folder first, then fall back to all notes in the default account:

```python
import subprocess

def find_note(title):
    script = f'''
tell application "Notes"
  repeat with f in folders of default account
    if name of f is "Projects" then
      repeat with n in notes of f
        if name of n is "{title}" then return body of n
      end repeat
    end if
  end repeat
  repeat with n in notes of default account
    if name of n is "{title}" then return body of n
  end repeat
  return "NOT_FOUND"
end tell
'''
    r = subprocess.run(['osascript', '-e', script], capture_output=True, text=True)
    return r.stdout.strip()
```

If the Note is **not found**, ask the user:
> "No Apple Note found titled '[title]'. Would you like me to create one in your Projects folder so this file stays in sync?"

Wait for their response before proceeding. If yes, convert the full `.md` to Notes HTML (see Step 5 conversion table) and create the Note:
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
  if targetFolder is missing value then set targetFolder to folder "Notes" of default account
  make new note at targetFolder with properties {{name:"{title}", body:"{escaped_body}"}}
end tell
'''
```

### Step 4 — Compare: identify differences

Parse both the `.md` file and the Note HTML into a list of sections. A **section** is a `###` heading plus its bullets (or a `##` heading for top-level domains).

**Sections in `.md` but not in Note** → push to Note  
**Sections in Note but not in `.md`** → pull into `.md`  
**Sections present in both** → no action (do not overwrite; content may have diverged intentionally)

To detect section presence, match by heading text only (strip markdown `###`/`##` vs HTML `<h2>`). Ignore minor punctuation differences.

Present the diff to the user before making any changes:

```
SYNC PLAN for "[title]"
=======================
→ PUSH to Apple Note (in .md, missing from Note):
  + ### Email Notifications  (4 bullets)
  + ### Two-Factor Authentication  (3 bullets)

← PULL to .md (in Note, missing from .md):
  + ### Dark Mode  (1 bullet)

= No changes (present in both):
  User Profile, Billing, Dashboard, ...

Proceed? (yes / skip push / skip pull / cancel)
```

Wait for user confirmation before writing anything.

### Step 5 — Push additions to Apple Note

For each section to push, convert markdown → Apple Notes HTML:

| Markdown | Notes HTML |
|----------|-----------|
| `### Section Name` | `<div><b><h2>Section Name</h2></b></div>` |
| `## Domain Name` | `<div><b><h2>Domain Name</h2></b></div>` |
| First `- bullet` in a list | `<ul>\n<li>bullet text</li>` |
| Subsequent `- bullet` | `<li>bullet text</li>` |
| End of bullet group | `</ul>` |
| Blank line between sections | `<div><br></div>` |
| Prose paragraph | `<div>paragraph text</div>` |
| `**bold**` | `<b>bold</b>` |
| `&` | `&amp` (no semicolon — Notes entity style) |
| `>` literal | `&gt` (no semicolon) |
| `"` / `"` curly quotes | straight `"` |
| `—` em dash | `-` hyphen |

**Find the insertion anchor** in the Note HTML. The new section should appear after the section that precedes it in the `.md`. Use exact string matching:
```python
# anchor = closing </ul> or </div> of the preceding section
patched_body = note_body.replace(anchor, anchor + "\n" + new_html)
```

If no anchor is found (the preceding section is also missing), append to end of body.

Apply all pushes in a single write:
```python
def write_note(title, new_body):
    escaped = new_body.replace('\\', '\\\\').replace('"', '\\"')
    script = f'''
tell application "Notes"
  repeat with f in folders of default account
    if name of f is "Projects" then
      repeat with n in notes of f
        if name of n is "{title}" then
          set body of n to "{escaped}"
          return "ok"
        end if
      end repeat
    end if
  end repeat
  repeat with n in notes of default account
    if name of n is "{title}" then
      set body of n to "{escaped}"
      return "ok"
    end if
  end repeat
  return "note not found"
end tell
'''
    return subprocess.run(['osascript', '-e', script], capture_output=True, text=True).stdout.strip()
```

### Step 6 — Pull additions into `.md`

For each section to pull, convert the Note HTML back to markdown:

| Notes HTML | Markdown |
|-----------|---------|
| `<div><b><h2>X</h2></b></div>` | `### X` |
| `<li>text</li>` | `- text` |
| `<b>text</b>` | `**text**` |
| `&amp` | `&` |
| `&gt` | `>` |
| `<div>text</div>` | `text` (prose paragraph) |
| `<div><br></div>` | blank line |

Insert the new markdown section at the correct position in the `.md` — immediately before the section that follows it (matched by heading text).

### Step 7 — Report

```
SYNC COMPLETE for "[title]"
============================
→ Pushed to Apple Note:  2 sections (+12 bullets)
← Pulled to .md:         1 section (+1 bullet)
= Unchanged:             18 sections
```

If nothing changed: *"Files are already in sync — no changes made."*
