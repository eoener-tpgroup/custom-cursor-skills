# Jira — shared module

Loaded by `/jira` before every subcommand. Do not run as a standalone command.

## Agency agent (Task dispatch — required)

**Before Phase 1 of the subcommand:**

1. **Read** `lib/multi-agent-loop.md` (commands: `lib/multi-agent-loop.md`; skills: installed `lib/` copy).
2. Dispatch a **Task** (`generalPurpose`) for the Primary role. Prompt must require `Read` of the rule path below. Bus = **in-session**.
3. Parent remains orchestrator: owns auth, AskQuestion, and MCP/REST writes after confirmation. Do not skip Task when it is available.

| Role | Path |
|------|------|
| Primary | `~/.cursor/rules/project-management-jira-workflow-steward.mdc` |

**Precedence:** Command phases, PITOPS/Service Desk field IDs, Wiki Markup formatting, security rules, and AskQuestion gates win on conflict. The Task brief guides ticket hygiene and tone — it does not replace phases, invent field IDs/ticket content, or override abort/confirmation rules.

Missing rule file → continue; note skipped. Task unavailable → in-parent Read fallback per lib.

## Prerequisites

- Jira access via MCP **or** credentials in `~/.cursor/.env` (`JIRA_URL`, `JIRA_TOKEN`)
- Subcommand-specific args (ticket key, fields, etc.) — see the loaded subcommand module

## Security (strict)

- **Never** print, log, or copy token values into code/chat
- **Never** commit credentials or write them into files
- **No secrets** in tickets or comments (passwords, tokens, API keys, internal credentials)
- Load credentials only from `~/.cursor/.env` — **do not** ask the user for a token when that file exists

## Jira access (priority)

Same order for every subcommand (see also `service-credentials.mdc` if present):

1. Check Jira MCP (`GetDynamicTools`, pattern `jira`)
2. If available and authenticated: use MCP tools (inspect schema before each call)
3. **Fallback**: REST / Service Desk API via `~/.cursor/.env`

```bash
source ~/.cursor/.env
# JIRA_URL and JIRA_TOKEN must be set
```

On missing credentials or auth failure (401/403): inform the user; **do not** continue without auth.

## Content formatting — Jira (strict, MCP and REST)

**Mandatory for every write path** (`create`, `comment`). Whether via **Jira MCP**, **Service Desk API**, or **REST API 2/3**, bodies MUST use **Jira Wiki Markup** — never GitHub/GitLab GFM, never Jira Markdown (`##`, `` `code` ``, fenced ```` ``` ````), never raw HTML.

### Dialect (fixed)

**Always author and submit as Jira Wiki Markup** (string `description` / `body`). Do **not** detect or switch dialects. Do **not** use Jira Markdown or hand-written ADF as the source format.

| Preference | Transport | Action |
|------------|-----------|--------|
| 1 | REST / Service Desk with string field | Pass Wiki Markup string as-is |
| 2 | MCP tool that only accepts ADF / `document` | Prefer REST with Wiki string; if MCP is required, convert the **Wiki Markup draft** to ADF per schema — never invent Markdown first |

### Field rules

| Field | Rules |
|-------|-------|
| **Summary** | Plain text only — **no** Wiki/Markdown. Max. ~255 chars. Short, specific, single line |
| **Description / comment body** | **Jira Wiki Markup only**. Structured sections; blank line between sections |
| **Links** | `[label\|https://example.com]` or full URLs; no relative paths |
| **Error messages / logs** | Verbatim inside `{code}…{code}` or `{noformat}…{noformat}` |

### Jira Wiki Markup cheat sheet

| Element | Syntax | Do not use |
|---------|--------|------------|
| Headings | `h2. Problem`, `h3. …` | `## Problem`, `#` |
| Bold / italic | `*bold*`, `_italic_` | `**bold**`, HTML |
| Bullets | `* item` / `# item` (ordered) | `- item` GFM lists |
| Inline / blocks | `{{hostname}}`, `{code}…{code}`, `{noformat}…{noformat}` | Fenced ```` ``` ````, `` `inline` `` |
| Links | `[label\|https://example.com]` | Markdown `[text](url)` |

### Technical encoding (MCP and REST)

1. **Never hand-escape JSON** — always `json.dumps()` for the full payload (or native MCP JSON args)
2. **Newlines** — in string fields as `\n`; blank line between sections
3. **Special characters** — quotes in logs/paths via proper JSON encoding only
4. **No HTML** in body fields
5. **Preview** — show drafts as Wiki Markup (as Jira would render), not escaped JSON
6. **Wiki first** — draft in Wiki Markup; only convert to ADF if an MCP tool schema forces it

```bash
# Correct: full payload via json.dumps (REST / shell fallback)
python3 <<'PYEOF'
import json
payload = {
    "serviceDeskId": "5",
    "requestTypeId": "657",
    "requestFieldValues": {
        "summary": "db01.example.com - disk alert investigation",
        "description": "h2. Problem\n\nFilesystem usage above threshold.\n\nh2. Details\n\n{code}\n<log excerpt>\n{code}",
    }
}
print(json.dumps(payload))
PYEOF
```

### Pre-write checklist

- [ ] Summary (if any) plain text, under 255 chars, no markup
- [ ] Body is **Wiki Markup only** (`h2.`, `*`, `{code}`) — no Markdown/ADF as source
- [ ] `h2.` / `h3.` sections with blank lines
- [ ] Logs/commands in `{code}` / `{noformat}`
- [ ] Payload valid for MCP schema or REST JSON
- [ ] No secrets, no agent/Cursor meta references

## Ticket key rules (for `read` and `comment`)

- Format: `<PROJECT>-<NUMBER>` — regex `^[A-Z][A-Z0-9]+-\d+$`
- Normalize: trim; uppercase project prefix (`pitops-123` → `PITOPS-123`)
- **Never invent** a key from chat; missing/invalid → abort with the subcommand’s abort message
