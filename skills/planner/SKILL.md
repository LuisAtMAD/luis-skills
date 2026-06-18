---
name: planner
description: >
  BLOCKING: Always invoke this skill — NEVER use `gh issue create` directly. This skill is
  the only correct way to create manager tasks because it also sets the task category label,
  adds the issue to the "Luis Planner" project, and sets the Owner field.
  Raw `gh` commands skip all three. Trigger on: "new task", "create task",
  "add a task", "remind me to", "log a task", "make a note", "follow up on", "I need to",
  "don't let me forget", "track this", "add to my list", or any clear intent to log a
  personal to-do or action item — even without the word "task".
  Task categories: Follow-Up, Action Item, Decision, Admin, Strategic.
---

# Manager Task Creator

Turn freeform input into a published GitHub issue in `LuisAtMAD/Luis`, added to the "Luis Planner" project.

> **IMPORTANT — do not bypass this skill.**
> Using `gh issue create` directly skips three required steps: setting the category label, adding the issue to the Luis Planner project board, and setting the Owner field. Always invoke this skill instead.

---

## Step 1 — Prereq Check

Check if `gh` is installed at the default path:

```bash
"/c/Program Files/GitHub CLI/gh.exe" auth status
```

- If the file is not found at that path: tell the user *"GitHub CLI was not found. Download and install it from https://cli.github.com, then run `gh auth login` in a terminal."* Stop.
- If found but not authenticated: tell the user to open a terminal and run `gh auth login`, and stop.
- If found and authenticated: proceed.

---

## Step 2 — Draft the Task

Classify the input using the fields below. Ask only if something is genuinely ambiguous.

### Category

| Category | When to use |
|----------|-------------|
| **Follow-Up** | Checking back on something delegated or sent out |
| **Action Item** | Task arising from a meeting or conversation |
| **Decision** | Something Luis needs to make a call on |
| **Admin** | Paperwork, approvals, scheduling |
| **Strategic** | Planning, goal-setting, bigger-picture work |

### Priority

Infer from urgency cues in the description if possible. If not mentioned and not inferable, default to **Medium** and note that in the draft.

| Priority | When to use |
|----------|-------------|
| **High** | Time-sensitive, blocks others, or has near-term consequences if missed |
| **Medium** | Important but not urgent |
| **Low** | Nice to have, no immediate deadline pressure |

### Due Date

Extract any date or deadline mentioned. Accept natural language ("Friday", "end of month", "June 30") — convert to `YYYY-MM-DD` for the body. If no date is mentioned, leave it as `Not set`.

### Owner

Infer from the notes (e.g., who owns the outcome or who this is about). If not mentioned and cannot be inferred, ask: *"Who is the owner of this task?"*

### Title

Imperative tone, ≤10 words (e.g., "Follow up with vendor on quote", "Approve budget request for Q3").

---

Produce a draft:

```
Category:   Follow-Up | Action Item | Decision | Admin | Strategic
Priority:   High | Medium | Low
Due Date:   YYYY-MM-DD  (or "Not set")
Owner:      <name>

<full description — preserve all context from the user's input. Use [clarify: ...] for anything genuinely unclear.>
```

Rules:
- **Title**: imperative tone (e.g., "Follow up with vendor on quote", "Decide on org chart changes")
- **Description**: preserve all meaningful context.
- **Assignee** is not shown in the draft — set automatically to `@me` when created.
- The draft header is for review only — these become structured fields and labels, not body text.

---

## Step 3 — Confirm and Edit

Show the draft. Ask exactly: *"Anything to adjust before I create this?"*

- Apply any changes, re-show the updated draft if changes were substantial, and ask again.
- When the user confirms, proceed.

---

## Step 4 — Create the Issue

The label is the category (slugs: `follow-up`, `action-item`, `decision`, `admin`, `strategic`).

Format the body:

```
**Priority:** High | Medium | Low
**Due Date:** YYYY-MM-DD (or "Not set")

<description text>
```

Write only the body content to a uniquely named temp file (use a timestamp to avoid collisions), then run:

```bash
GH="/c/Program Files/GitHub CLI/gh.exe"
BODY_FILE="/tmp/gh_task_body_$(date +%s).md"
# write body content to $BODY_FILE here
"$GH" issue create \
  --repo LuisAtMAD/Luis \
  --title "<title>" \
  --body-file "$BODY_FILE" \
  --label "<category-slug>" \
  --assignee @me
rm -f "$BODY_FILE"
```

Capture the issue URL printed by gh (last line of output). Store it as `ISSUE_URL`.

---

## Step 5 — Add to Project

Use PowerShell (not bash) for all JSON parsing since `jq` is not available.

```powershell
$GH = "C:\Program Files\GitHub CLI\gh.exe"

$PROJECT_NUMBER = (& $GH project list --owner LuisAtMAD --format json | ConvertFrom-Json).projects |
  Where-Object { $_.title -eq "Luis Planner" } | Select-Object -ExpandProperty number

$ITEM_JSON = & $GH project item-add $PROJECT_NUMBER --owner LuisAtMAD --url "$ISSUE_URL" --format json | ConvertFrom-Json
$ITEM_ID = $ITEM_JSON.id
```

---

## Step 6 — Set the Owner Custom Field

Resolve the project node ID and field ID using PowerShell:

```powershell
$GH = "C:\Program Files\GitHub CLI\gh.exe"

$PROJECT_ID = (& $GH project list --owner LuisAtMAD --format json | ConvertFrom-Json).projects |
  Where-Object { $_.title -eq "Luis Planner" } | Select-Object -ExpandProperty id

$FIELD_ID = (& $GH project field-list $PROJECT_NUMBER --owner LuisAtMAD --format json | ConvertFrom-Json).fields |
  Where-Object { $_.name -eq "Owner" } | Select-Object -ExpandProperty id
```

If `$FIELD_ID` is empty or null: tell the user *"An 'Owner' text field doesn't exist in the project yet. Add it at https://github.com/users/LuisAtMAD/projects then re-run."* Stop.

Otherwise, set the field:

```powershell
& $GH project item-edit `
  --project-id $PROJECT_ID `
  --id $ITEM_ID `
  --field-id $FIELD_ID `
  --text "<owner-name>"
```

---

## Step 7 — Report

Tell the user:
- The issue URL (as a clickable link)
- Confirmation it was added to the Luis Planner project with Owner set

One line each. Done.
