# Jira — subcommand: create

Loaded by `/jira create …` after `jira/_common.md`. Do not run as a standalone command.

Creates a **new Jira ticket** in Transporeon Service Desk, project **PITOPS**. Ticket content comes from the user prompt or is derived from **chat context**.

**Invocation:** `/jira create [description]`

**Jira ticket content (summary, description) must always be written in English** — regardless of the language used in chat.

Wiki Markup, security, and auth: see `jira/_common.md`.

---

## Workflow

### Phase 1: Determine ticket content

**Two sources** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit text** in the user prompt after `/jira create` | User provided a description directly |
| 2 | **Chat context** | No explicit text — derive from the conversation so far |

**Derive from context — rules:**

1. Clearly identify the **problem/request** (What? Why? Which system?)
2. Include **relevant details**: hostnames, error messages, time ranges, affected services
3. **No internal meta-references** (tools, agent, automation)
4. Professional tone — as if the user wrote the ticket personally
5. If context has **no clear ticket topic** → **do not guess**, ask the user

**When unclear → ask the user (required):**

If required information is missing or ambiguous, prefer **`AskQuestion`** (ask individually or bundled as needed). If AskQuestion is unavailable, present the **same questions/options in chat** and wait — do not guess, and do not continue without answers.

**Required questions (when not derivable from prompt/context):**

| Field | Question | Options (example) |
|-------|----------|-------------------|
| **Summary** | What should the ticket title (summary) be? | Free text or suggestions from context |
| **Request Type** | What type of request is this? | Service Request Platform / Platform Incident / Change Request / Update Request / Other |
| **Environment** | Which environment is affected? | Productive / Acceptance / Integration / Development / HQ / Enterprise / Other |
| **Platform** | Which platform? | Transporeon / Ticontract / Mercareon / Tim / ControlPay / All / Other |
| **Component** | Which component area? | Database / Network / Storage / Kubernetes / Monitoring / VMware / Other / … |
| **Description** | What details should the ticket description include? | Free text |

**AskQuestion example** (when several fields are unclear):

> I need a few details to create the Jira ticket:

Options for Request Type, Environment, Platform, Component — multiple-choice where possible, plus “Other (I'll specify)”.

**Do not create** until at least Summary, Request Type, Environment, Platform, and Component are clarified.

### Phase 2: Check Jira access

See `jira/_common.md` (Jira access).

### Phase 3: Determine request type and required fields

**Default project:** `PITOPS` (Platforms and IT Operation)
**Service Desk ID:** `5`

**Request types** (most common — ask the user when unclear):

| Request Type | ID | When to use |
|--------------|-----|-------------|
| Service Request Platform | `657` | Default for platform requests, investigation, support |
| Platform Incident | `153` | Production incident / outage |
| Change Request | `155` | Planned change |
| Update Request | `154` | Update/patch request |
| Service Request | `149` | General service request |
| Internal | — (via REST API 2) | Internal ticket, not Service Desk |

**Default:** `657` (Service Request Platform) when no other category is clear from context.

**Required fields for request type 657** (Service Request Platform):

| Field | Field ID | Format | Example |
|-------|----------|--------|---------|
| Summary | `summary` | string, max ~255 chars | `dbamq08.pd.tp.nil - FS corruption alert` |
| Description | `description` | string (**Jira Wiki Markup**) | Detailed description |
| Environment | `customfield_12929` | multiselect (array) | `[{"id": "13339"}]` = Productive |
| Platform | `customfield_12930` | select | `{"id": "13345"}` = Transporeon |
| Component/s | `components` | array | `[{"id": "20471"}]` = Storage |

**Environment values:**

| Label | ID |
|-------|-----|
| Productive | `13339` |
| Acceptance | `13340` |
| Integration | `13341` |
| Development | `13342` |
| HQ / Enterprise | `13343` |
| Other | `13344` |

**Platform values:**

| Label | ID |
|-------|-----|
| Transporeon | `13345` |
| Ticontract | `13346` |
| Mercareon | `13347` |
| Tim | `13348` |
| ControlPay | `20970` |
| All | `13359` |
| Other | `13349` |

**Component values** (selection — ask the user when unclear, or derive from context):

| Label | ID |
|-------|-----|
| Database | `14972` |
| Network | `14895` |
| Storage | `20471` |
| VMware | `20470` |
| Kubernetes | `22870` |
| Monitoring | `23874` |
| Server Hardware | `14896` |
| Other | `14894` |
| … | more via API |

**Derive field values from context:**

- Error on a production server → Environment: Productive
- Hostname contains `.pd.` → usually Productive
- DB/PostgreSQL/MySQL topic → Component: Database
- Filesystem/disk → Component: Storage
- When unsure → **ask the user**, do not guess

### Phase 4: Show ticket draft and obtain confirmation (required)

**Always before creating**, show the full draft and wait for confirmation via **`AskQuestion`** (or the same numbered options in chat if AskQuestion is unavailable).

**Do not create automatically** — only after explicit confirmation.

```markdown
## Ticket draft

| Field | Value |
|-------|-------|
| Project | PITOPS |
| Request Type | Service Request Platform |
| Summary | ... |
| Environment | Productive |
| Platform | Transporeon |
| Component | Storage |

### Description
<full description>

---
Create this ticket in Jira?
```

**AskQuestion** (required — no implicit continuation): Prefer `AskQuestion` with the options below. If that tool is unavailable, present the **same numbered options in chat** and wait for an explicit choice. Do **not** proceed without confirmation.

> Create this Jira ticket?

Options:

1. **Yes, create ticket** — Create the ticket
2. **Edit first** — User provides corrections; adjust draft and confirm again
3. **Cancel** — Create nothing

On **Cancel**: Briefly confirm that no ticket was created.

### Phase 5: Create ticket (only after confirmation)

**Formatting:** See `jira/_common.md` (Content formatting — Jira). Summary plain text; Description **always Jira Wiki Markup**; payload via `json.dumps` or MCP JSON args.

**Service Desk API** (preferred for request types):

```bash
source ~/.cursor/.env
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/servicedeskapi/request" \
  -d "$(python3 <<'PYEOF'
import json
payload = {
    "serviceDeskId": "5",
    "requestTypeId": "<REQUEST_TYPE_ID>",
    "requestFieldValues": {
        "summary": "<summary>",
        "description": "<description>",
        "customfield_12929": [{"id": "<environment_id>"}],
        "customfield_12930": {"id": "<platform_id>"},
        "components": [{"id": "<component_id>"}]
    }
}
print(json.dumps(payload))
PYEOF
)"
```

**Fallback — REST API 2** (for issue types without Service Desk, e.g. Task, Internal):

```bash
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue" \
  -d "$(python3 <<'PYEOF'
import json
payload = {
    "fields": {
        "project": {"key": "PITOPS"},
        "issuetype": {"id": "<ISSUE_TYPE_ID>"},
        "summary": "<summary>",
        "description": "<description>"
    }
}
print(json.dumps(payload))
PYEOF
)"
```

**Issue type IDs** (REST API 2 fallback):

| Type | ID |
|------|-----|
| Service Request | `10902` |
| Task | `3` |
| Internal | `11000` |

On HTTP 201: success — extract `issueKey` from the response.
On error: report HTTP status and message (without token). Identify missing required fields and ask the user again.

### Phase 6: Report result

On success:

```markdown
## Ticket created

| Field | Value |
|-------|-------|
| Key | <KEY> |
| Summary | ... |
| URL | <JIRA_URL>/browse/<KEY> |
```

- Ticket key and browse URL
- Brief confirmation
- Note: `/jira read <key>` · `/jira comment <key>`

On cancel:

- Briefly confirm that no ticket was created

---

## Description — best practices

1. **Problem statement**: What is wrong / what is requested?
2. **Context**: Affected systems, hostnames, environment
3. **Details**: Error messages, logs, timestamps (quote verbatim)
4. **Impact**: Effect on users/services (if known)
5. **Steps taken**: What was already tried (if known from context)
6. **Expected outcome**: What the ticket should achieve

**Template (Wiki Markup):**

```text
h2. Problem
<what is wrong or what is requested>

h2. Affected Systems
* Host: <hostname>
* Environment: <env>
* Service: <service>

h2. Details
{code}
<error messages, logs, timestamps>
{code}

h2. Impact
<business/operational impact>

h2. Steps Already Taken
* <step 1>
* <step 2>

h2. Expected Outcome
<what resolution is needed>
```

---

## Error handling

| Problem | Action |
|---------|--------|
| Unclear ticket content | AskQuestion (or chat options) — do not guess |
| Required fields missing | Ask the user; do not fill with defaults |
| User cancels | Confirm; create nothing |
| Auth failed | Check credentials |
| Validation error (400) | Identify missing/invalid fields; ask the user |
| Secrets in draft | Warn, remove, confirm again |
| Suspected duplicate | Warn if a similar ticket was mentioned in context |
