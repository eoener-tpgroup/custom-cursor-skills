# Jira — subcommand: read

Loaded by `/jira read …` after `jira/_common.md`. Do not run as a standalone command.

Loads a Jira ticket by key and returns a **structured, complete summary** — including status, fields, description, and comments.

**Invocation:** `/jira read <ticket-key>`

Ticket content (summary, description, comments) is quoted **as-is** from Jira — do not translate unless the user explicitly asks.

---

## ⚠️ Required argument: ticket key

**Abort immediately without a ticket key.** Do not load a ticket, do not guess, do not offer alternatives.

```
❌ /jira read                    → ABORT
❌ /jira read read               → ABORT (not a valid key)
❌ /jira read 123456             → ABORT (no project prefix)
✅ /jira read PITOPS-144486      → Ticket PITOPS-144486
✅ /jira read pitops-144486      → Normalize to PITOPS-144486
```

**Abort message** (report exactly as follows):

> This command requires a Jira ticket key as an argument.
> Usage: `/jira read <ticket-key>`
> Example: `/jira read PITOPS-144486`

The ticket key is in the user prompt **after the subcommand** (e.g. `/jira read PITOPS-144486`).

---

## Workflow

### Phase 1: Validate argument

1. Extract the ticket key from the user prompt (text after `/jira read`)
2. Normalize and validate format — see `jira/_common.md` (Ticket key rules)
3. If invalid → **abort immediately** (abort message)

### Phase 2: Check Jira access

See `jira/_common.md` (Jira access).

### Phase 3: Load ticket

**REST API (fallback):**

```bash
source ~/.cursor/.env
curl -s -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>?expand=renderedFields,names,changelog"
```

Also load comments if not fully present in `fields.comment`:

```bash
curl -s -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>/comment?orderBy=created&maxResults=100"
```

**Data to load (required):**

| Data | Purpose |
|------|---------|
| Key, Summary, Description | Core content |
| Status, Resolution | Current state |
| Issue Type, Priority | Classification |
| Reporter, Assignee, Created, Updated | Metadata |
| Components, Labels | Assignment |
| Custom fields (when present) | Environment, Platform, Request Type, etc. |
| Comments (all) | Full comment history |
| Changelog (last 20 entries) | Status/field changes |

On HTTP 404: ticket not found — have the user check the key.
On other errors: report HTTP status and message (without token).

### Phase 4: Summarize ticket

Output a structured report:

```markdown
# Jira Ticket: <KEY> — <summary>

## Overview
| Field | Value |
|-------|-------|
| Key | <KEY> |
| Summary | ... |
| Type | ... |
| Status | ... |
| Priority | ... |
| Reporter | ... |
| Assignee | ... |
| Created | ... |
| Updated | ... |
| Components | ... |
| Labels | ... |
| URL | <JIRA_URL>/browse/<KEY> |

## Description
<description — rendered/plain, preserve formatting>

## Custom Fields
| Field | Value |
|-------|-------|
| Environment | ... |
| Platform | ... |
| ... | ... |

## Comments (<count>)
| # | Author | Date | Content |
|---|--------|------|---------|
| 1 | ... | ... | ... |

## Recent Activity (Changelog)
| Date | Author | Change |
|------|--------|--------|
| ... | ... | status: Open → In Progress |

## Links
- Browse: <JIRA_URL>/browse/<KEY>
```

**Summary rules:**

1. **Complete**: Include all comments, not only the latest
2. **Chronological**: Sort comments by creation date
3. **No interpretation**: Quote description and comments verbatim
4. **Long content**: For very long descriptions/comments (>2000 characters), truncate with “(truncated — full content in Jira)”
5. **Attachments**: List only (filename, author, date) — do not download unless the user explicitly asks

### Phase 5: Report result

- Ticket URL (`<JIRA_URL>/browse/<KEY>`)
- One-sentence summary of the ticket content
- Related subcommands:
  - `/jira comment <key>` — add a comment
  - `/jira create` — create a new ticket

---

## Error handling

| Problem | Action |
|---------|--------|
| No ticket key | Abort immediately (abort message) |
| Invalid key format | Abort message; give an example |
| Ticket not found (404) | Have the user check key and project |
| Auth failed (401/403) | Check credentials in `~/.cursor/.env` |
| MCP not authenticated | Guide auth; try REST fallback |
| Network error | Report error; offer retry |
| Ticket with empty description/comments | Mark fields as “(empty)”; do not abort |
