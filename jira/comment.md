# Jira — subcommand: comment

Loaded by `/jira comment …` after `jira/_common.md`. Do not run as a standalone command.

Posts a **comment** on an existing Jira ticket. Comment content comes from the user prompt or is derived from **chat context**.

**Invocation:** `/jira comment <ticket-key> [comment text]`

**Jira comments must always be written in English** — regardless of the language used in chat.

Wiki Markup, security, and auth: see `jira/_common.md`.

**Comment-only nuances** (in addition to `_common`):
- Do **not** adopt the Jira agency agent’s commit/branch conventions when this command only posts a comment.
- Do **not** mirror Markdown from existing ticket comments — convert ideas into **Jira Wiki Markup**.

---

## ⚠️ Required argument: ticket key

**Abort immediately without a ticket key.** Do not post a comment, do not guess, do not offer alternatives.

```
❌ /jira comment                 → ABORT
❌ /jira comment fix this        → ABORT (not a valid key)
❌ /jira comment 123456          → ABORT (no project prefix)
✅ /jira comment PITOPS-144486  → Derive comment from context
✅ /jira comment PITOPS-144486 Investigation complete, FS check passed → Comment from prompt
```

**Abort message** (report exactly as follows):

> This command requires a Jira ticket key as the first argument.
> Usage: `/jira comment <ticket-key> [comment text]`
> Example: `/jira comment PITOPS-144486 Root cause identified and fix deployed.`

The ticket key is the **first argument** after the subcommand. Optional free text after that = explicit comment content.

---

## Workflow

### Phase 1: Validate argument

1. Extract the ticket key from the user prompt (first token after `/jira comment`)
2. Normalize and validate format — see `jira/_common.md` (Ticket key rules)
3. Treat remaining text after the key as the **explicit comment** (if present)
4. If the key is invalid → **abort immediately** (abort message)

### Phase 2: Determine comment content

**Two sources** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit text** in the user prompt after the ticket key | User provided the comment directly |
| 2 | **Chat context** | No explicit text — derive from the conversation so far |

**Derive from context — rules:**

1. Summarize the **core of the latest work/discussion** (investigation, fix, status update, etc.)
2. Include only information **relevant to Jira participants**
3. No internal tool/agent references, no meta-comments about automation
4. Professional, factual tone — as if the user wrote it personally
5. Format as **Jira Wiki Markup** (see `jira/_common.md`) — do not mirror existing ticket comments as Markdown
6. Do not adopt commit/branch conventions from the agency agent — this subcommand only posts a comment

**When unclear → ask the user (required):**

If the desired comment content is **not clear** (neither from prompt nor context), prefer **`AskQuestion`**. If AskQuestion is unavailable, present the **same numbered options in chat** and wait — do not guess, and do not post without confirmation.

Question:

> What should the Jira comment say?

Options:

1. **Post investigation/status summary from context** — Summary of work so far
2. **Post a short status update** — Short status update (user supplies text)
3. **Let me specify the comment** — User writes the exact comment
4. **Cancel** — Abort; post nothing

**Do not post** while comment content remains unconfirmed when there was ambiguity.

### Phase 3: Verify ticket

Before posting, verify the ticket exists:

```bash
source ~/.cursor/.env
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>"
```

| HTTP code | Action |
|-----------|--------|
| 200 | Continue |
| 404 | Abort: ticket not found |
| 401/403 | Abort: auth problem |

Optional: load short ticket info (Summary, Status) for the confirmation shown to the user.

### Phase 4: Show comment draft and obtain confirmation (required)

**Always before posting**, show the full comment draft and wait for confirmation via **`AskQuestion`** (or the same numbered options in chat if AskQuestion is unavailable).

**Do not post automatically** — only after explicit confirmation.

```markdown
## Comment draft for <TICKET-KEY>

<comment body>

---
Post this comment to <TICKET-KEY>?
```

**AskQuestion** (required — no implicit continuation): Prefer `AskQuestion` with the options below. If that tool is unavailable, present the **same numbered options in chat** and wait for an explicit choice. Do **not** proceed without confirmation.

> Post this comment to <TICKET-KEY>?

Options:

1. **Yes, post comment** — Submit the comment
2. **Edit first** — User provides corrections; adjust draft and confirm again
3. **Cancel** — Post nothing

On **Cancel**: Briefly confirm that nothing was posted.

### Phase 5: Post comment (only after confirmation)

**Formatting:** See `jira/_common.md` — **always Jira Wiki Markup**.

**Priority:** MCP first, then REST (see `_common`).

**REST API:**

```bash
source ~/.cursor/.env
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>/comment" \
  -d "$(python3 <<'PYEOF'
import json
body = """<comment text here>"""
print(json.dumps({"body": body}))
PYEOF
)"
```

On HTTP 201/200: success.
On error: report HTTP status and message (without token).

### Phase 6: Report result

On success:

- Confirmation: “Comment posted to `<TICKET-KEY>`”
- Ticket link: `<JIRA_URL>/browse/<TICKET-KEY>`
- Comment ID if available from the response
- Briefly show the posted comment again (first ~200 characters)

On cancel:

- Briefly confirm that nothing was posted

---

## Comment templates (Wiki Markup)

### Status update

```text
h2. Status Update

<1-2 sentences on current state>

*Next steps:*
* <step 1>
* <step 2>
```

### Investigation summary

```text
h2. Investigation Summary

*Scope:* <what was investigated>

*Findings:*
* <finding 1>
* <finding 2>

*Conclusion:* <conclusion>

*Actions taken:*
* <action 1>
```

### Fix deployed

```text
h2. Fix Applied

*Problem:* <brief problem description>
*Solution:* <what was done>
*Verification:* <how it was verified>

Related: <MR/PR link if applicable>
```

---

## Error handling

| Problem | Action |
|---------|--------|
| No ticket key | Abort immediately (abort message) |
| Invalid key format | Abort message |
| Ticket not found | Have the user check the key |
| Unclear comment content | AskQuestion (or chat options) — do not guess |
| User cancels | Confirm; post nothing |
| Auth failed | Check credentials |
| Secrets in draft | Warn, remove from comment, confirm again |
| Permission denied (403) | Inform the user — may lack comment permission |
