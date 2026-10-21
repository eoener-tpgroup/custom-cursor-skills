---
name: merge-request-review
description: Intensive code review of a merge request including existing MR comments. MR ID optional (/merge-request-review [id] [focus]) — from argument, branch, or chat context.
---

# Merge Request Review

Analyzes a merge/pull request by MR ID and performs an **intensive, structured code review**.

Triggered manually via `/merge-request-review [id] [focus]`.

## ⚙️ Arguments & context

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **MR ID** | conditional | Numeric IID/PR number (`42`, `!42`, `#42`) |
| 2 | **Focus** | no | Free text — review emphasis, symptoms, files, questions |

Everything after the MR ID (if present) is optional **Focus** and may contain spaces.

**Resolve MR ID** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit ID** in user prompt | First valid token after `/merge-request-review` (`42`, `!42`, `#42`) |
| 2 | **Chat context** | MR URL, `!42`, `MR 42`, `PR #42` in the conversation |
| 3 | **Current branch** | Find open MR/PR for local branch via `gh`/`glab`/GitLab MCP |

**Focus** (explicit or from context):

- Review emphasis (`security`, `performance`, `only auth module`)
- Concrete questions (`is the retry logic safe under load?`)
- Areas to skip (`skip generated files`, `ignore docs changes`)
- Reference to prior discussion in chat

**Derive from context — rules:**

1. Take focus from chat when the prompt has no free text after the ID
2. **Do not guess** — if no ID can be resolved and no unambiguous MR in context → abort or ask the user
3. If multiple MRs are possible (e.g. several URLs in chat) → **ask the user**

```
✅ /merge-request-review 42
✅ /merge-request-review !42 focus on SQL injection and auth bypass
✅ /merge-request-review security review for webhook handler
✅ /merge-request-review                    → ID from branch or chat context
❌ /merge-request-review review             → no valid ID token, no context → abort
```

### Missing or invalid MR ID

When neither argument nor context nor branch search yields a unique MR ID:

1. **Prefer `AskQuestion`**: ask for MR ID or branch/URL
2. If unanswered → **abort** with:

> This command requires a merge request ID (or enough context to identify one).
> Usage: `/merge-request-review [id] [focus]`
> Example: `/merge-request-review 42 security and error handling`

The MR ID appears in the user prompt **after the command name** (e.g. `/merge-request-review 42`). Optional focus follows after that.

---

## Output language

**This command definition is in English.** **Always write all user-facing output in English** — status messages, review reports, error messages, questions to the user, and MR/PR comments. This applies even if the user writes in another language.

## Agency agent

**Before Phase 1**, read and adopt judgment/quality guidance from:

| Role | Path | When |
|------|------|------|
| Primary | `~/.cursor/rules/engineering-code-reviewer.mdc` | Always |
| Optional | `~/.cursor/rules/security-appsec-engineer.mdc` | When Focus (argument or chat) emphasizes security, auth, secrets, injection, XSS, or similar |

**Precedence:** This command’s phases, severity scale (Critical/Major/Minor/Suggestion/Question), dual-platform detection, Content formatting, inline-comment/suggestion rules, and “post only on explicit request” win on conflict. Map the primary agent’s blocker/suggestion/nit language onto this command’s severity levels. The agency agent guides review depth and constructive tone — it does not replace phases, auto-post to the MR/PR, or invent findings without reading the diff and existing discussion.

If a listed agency-agent file is missing, continue the command workflow and note that the agency agent could not be loaded.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 2), format MR/PR review comments for **that platform only** — whether posting via **`gh`**, **`glab`**, **GitLab MCP**, or **GitHub/GitLab REST API**.

| Platform | Comment dialect | Do not use |
|----------|-----------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only `!MR` shorthand as primary link style, Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax, Jira Wiki/ADF |

Never post Jira markup into MR/PR comments. Never use raw HTML for structure. Chat review reports may use the same Markdown; when posting to the platform, the body MUST match the detected dialect.

### Comment body rules (review posts)

| Field | Rules |
|-------|-------|
| **Comment body** | Platform Flavored Markdown: `##`-headings, numbered/bulleted lists, `**bold**` for severity/recommendation, `` `file.ts:42` `` for file refs |
| **Structure** | Executive Summary + recommendation first → Findings by severity → Positives → Test plan |
| **Severity** | Icons or **Bold** labels (`**Critical**`, `**Major**`) — consistent throughout |
| **Code quotes** | Fenced blocks with language tag or inline backticks; no huge dumps |
| **Links** | Full URLs to files/lines where the platform supports them |
| **Cross-refs** | GitHub: `#123`, commit SHA · GitLab: `#123`, `!42` when linking MRs |
| **Inline (required when possible)** | Post findings with a clear diff line as **line comment** at `path` + `line` — do not only list them in a general PR/MR comment |
| **GitHub Suggestions** | When a concrete fix for the commented lines can be stated → **`suggestion` block** in the inline comment (see below). Otherwise explanation + optional code suggestion without `suggestion` |

Prefer the **common Markdown subset** (headings, bullets, code fences, links, `@mentions`) so rendering is reliable on both platforms.

### GitHub line comments + Suggestions (GFM)

When posting on **GitHub**, prefer **inline review comments** on the changed lines over a single top-level issue comment for actionable findings.

**Suggestion fence** (must be a fenced block whose language tag is exactly `suggestion` — GitHub renders an “Apply suggestion” UI):

````markdown
**Major:** Guard against null before use.

```suggestion
if (value == null) {
  return;
}
doWork(value);
```
````

| Rule | Detail |
|------|--------|
| When suggestion | Concrete, applyable replacement for the exact commented line(s); same indentation/style as the file |
| When no suggestion | Architectural questions, “consider X”, multi-file refactors, or when the correct fix is ambiguous → prose + optional non-suggestion code fence |
| Multi-line | Use API `start_line` + `line` (and `start_side`/`side`) so the suggestion covers the full hunk being replaced |
| Side | Almost always `RIGHT` (the PR’s new file version) |
| Commit | Anchor comments to the PR head commit (`headRefOid`) |

Do **not** put a `suggestion` block in a general PR conversation comment — GitHub only applies suggestions on **pull request review comments** attached to a file/line.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq`, HEREDOC, `--body-file`, or MCP native JSON args
2. **Preserve newlines** — paragraphs and lists need real newlines (`\n` in JSON)
3. **Special characters** — backticks, `"`, `\` in code examples via proper JSON encoding
4. **No HTML** — Markdown instead of `<br>` / `<pre>`
5. **Preview** — before posting, show the comment draft as platform Markdown, not escaped JSON
6. **MCP schema first** — put the Markdown string in the tool’s `body` / `note` field as required by the schema

**GitHub (`gh` or REST) — GFM:**

General summary (optional, after inline comments):

```bash
gh pr comment <id> --body-file /tmp/review-summary.md
```

**Preferred — inline line comments** (with optional `suggestion`). Resolve `OWNER`/`REPO` and head SHA first:

```bash
OWNER=$(gh repo view --json owner -q .owner.login)
REPO=$(gh repo view --json name -q .name)
COMMIT=$(gh pr view <id> --json headRefOid -q .headRefOid)

# Single-line inline comment (+ suggestion when possible)
gh api "repos/${OWNER}/${REPO}/pulls/<id>/comments" \
  -f commit_id="$COMMIT" \
  -f path="src/example.ts" \
  -F line=42 \
  -f side=RIGHT \
  -f body="$(cat <<'EOF'
**Major:** Null check missing before use.

```suggestion
if (value == null) {
  return;
}
doWork(value);
```
EOF
)"
```

Multi-line replacement (set `start_line`/`line` to the range being replaced):

```bash
gh api "repos/${OWNER}/${REPO}/pulls/<id>/comments" \
  -f commit_id="$COMMIT" \
  -f path="src/example.ts" \
  -F start_line=40 \
  -F line=45 \
  -f start_side=RIGHT \
  -f side=RIGHT \
  -f body="$(cat <<'EOF'
**Critical:** Replace the whole block with a safe variant.

```suggestion
// replacement lines matching start_line..line
```
EOF
)"
```

Or submit several inline comments as one review (summary body + comments array via `gh api repos/.../pulls/<id>/reviews` with JSON/`--input`). Prefer that when posting Critical/Major batches so the recommendation stays in one review event.

**GitLab (`glab`, MCP, or REST) — GitLab Flavored Markdown:**

Prefer **inline / diff notes** on the relevant lines when the API/MCP supports positions. General note only for summary:

```bash
glab mr note <id> -m "$(cat <<'EOF'
## Code Review
...
EOF
)"
```

GitLab applyable suggestions (when posting an inline note on the exact lines):

````markdown
```suggestion:-0+0
replacement code
```
````

- MCP / REST: `body` = platform Markdown string, JSON-encoded via tool args or `json.dumps`; for inline notes include path/line (or position) per schema

### Pre-post checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Actionable findings posted as **inline line comments** when `path` + line are known from the diff
- [ ] **GitHub:** `suggestion` fence included whenever a concrete line-level fix is clear; anchored to head commit + `RIGHT` side
- [ ] Headings and blank lines between sections (summary comment)
- [ ] Findings with `file:line` where possible
- [ ] Review recommendation (Approve / Request changes) clear and near the top
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Workflow

### Phase 1: Validate arguments & resolve MR ID

1. Parse user prompt after `/merge-request-review`
2. Check first token: numeric IID/PR number (`42`, `!42`, `#42`) → **MR ID**
3. Remaining text → **Focus** (review emphasis, questions, constraints)
4. If no valid ID token: check **chat context** and **current branch** (see above)
5. Supplement focus from context when the prompt has no free text after the ID
6. If MR ID still unclear → `AskQuestion` or abort (see above)
7. Derive project path from remote URL or user context

### Phase 2: Detect platform and load MR data

**Order is mandatory:** resolve the remote URL **before** choosing a client or loading MR/PR data.

1. Read `git remote get-url origin` (fallback: recognizable primary remote from `git remote -v`).
2. Classify from the URL only:

| Platform | Detection (URL only) | Client next |
|----------|----------------------|-------------|
| **GitHub** | URL contains `github.com`, or hostname contains `github` (e.g. `github.mycompany.com`) | `gh` CLI — then check `gh auth status` |
| **GitLab** | Hostname contains `gitlab` (incl. self-hosted, e.g. `gitlab.office.transporeon.com`) | `glab` **or** GitLab MCP (see client choice below) |
| **Unknown** | Neither rule matches | **Do not guess** — `AskQuestion` (GitHub vs GitLab) or abort. No probing with `gh`/`glab`/MCP for detection. No “push + manual URL” path |

**Important:** Availability of `gh`, `glab`, or GitLab MCP **must not override the platform**. URL → platform first, then client.

**GitLab client choice:** Whatever is available and authenticated. If **both** work → **prefer `glab`**; otherwise GitLab MCP. Both are not required.

**After detection:** Chat review title and refs use the detected platform only — GitHub → `Code Review: PR #<id>`, `#n`; GitLab → `Code Review: MR !<id>`, `!n` (Phase 7). Optional posts (Phase 8) use that platform’s Markdown only (see **Content formatting — GitLab or GitHub**).

Then load data via the matching subsection below (**GitLab** or **GitHub**).

#### GitLab (via `glab` or MCP)

**Client:** Prefer **`glab`** when both `glab` and GitLab MCP are available and authenticated; otherwise GitLab MCP. Tool discovery for MCP: `GetDynamicTools`, namespace `GitLab`. On `needsAuth`: ask user to authenticate, **do not** proceed without auth.

**Via `glab` — load data:**

```bash
glab mr view <id>
glab mr diff <id>
glab mr view <id> --comments
# Resolve project path, then load discussions (URL-encode namespace/project)
PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)
glab api "projects/${PROJECT}/merge_requests/<id>/discussions"
```

**Via MCP — load data:**

1. Check tool schema via `GetDynamicTools` (e.g. `get_merge_request`, `get_merge_request_notes`, `list_merge_request_discussions`)
2. Call matching MCP tools (same data types as below)

| Data | Purpose |
|------|---------|
| MR details | Title, description, author, status, labels, approvals |
| MR diffs | All file changes |
| MR commits | Commit history, message quality |
| MR pipelines | CI/CD status |
| MR discussions / notes | **All** existing comments, threads, review notes |
| MR approvals | Approvals already granted and review states |

**GitLab — load comments (required):**

Via **`glab`** (when chosen as client):

1. `glab mr view <id> --comments` — quick overview of notes
2. Resolve/encode project path, then `glab api "projects/${PROJECT}/merge_requests/<id>/discussions"` — full threads incl. resolved/unresolved and inline positions (`PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)`)

Via **GitLab MCP** (when chosen as client):

1. Check tool schema via `GetDynamicTools` (e.g. `get_merge_request_notes`, `list_merge_request_discussions`, `get_merge_request`)
2. Load **all** discussions and notes — including:
   - General MR comments (discussion notes)
   - Inline comments on code lines (diff notes)
   - System notes only when relevant (e.g. “resolved”, “approved”)
   - Replies in threads (full conversation, not just top-level)
3. Per comment capture: author, date, file/line (if inline), content, resolved/unresolved

**GitLab — post comment (only on explicit user request, Phase 8):**

- `create_workitem_note` or equivalent MCP tool (check schema before call)
- Body = **GitLab Flavored Markdown** (see **Content formatting — GitLab or GitHub**)

#### GitHub (via `gh`)

```bash
gh pr view <id> --json title,body,state,author,baseRefName,headRefName,files,commits,statusCheckRollup,reviews,comments
gh pr diff <id>
gh api repos/{owner}/{repo}/pulls/<id>/comments
gh api repos/{owner}/{repo}/issues/<id>/comments
gh pr view <id> --comments
```

**GitHub — load comments (required):**

- Issue comments (general PR discussion)
- Review comments (`pulls/{id}/comments` — inline on code lines)
- Review submissions (`APPROVED`, `CHANGES_REQUESTED`, `COMMENTED` + body)
- Include threads and replies in full

### Phase 3: Analyze existing MR comments

**Before your own code review**, evaluate all loaded comments:

1. **Summarize**: What was already noted? By whom (reviewer, author, CI bot)?
2. **Check status**: Which points are open vs. resolved/answered?
3. **Avoid duplicates**: Do **not** report already discussed findings as new
4. **Identify gaps**: What was raised in comments but not yet fixed in code?
5. **Use context**: Include author replies and discussions in the assessment
6. **Contradictions**: If comments contradict each other or are stale → mark as 💬 Question

In the review report include a section **“Existing MR discussion”**:

```markdown
## Existing MR discussion
| # | Author | Type | Status | Summary | Relevance for review |
|---|--------|------|--------|---------|----------------------|
| 1 | @user | inline `file.ts:42` | open | ... | Still open — included in findings |
| 2 | @reviewer | general | resolved | ... | Already addressed — do not repeat |
```

### Phase 4: Gather context

1. **Read affected files in repo** — not just diff hunks
2. **Project conventions** — `.cursor/rules/`, `CONTRIBUTING.md`
3. **Tests** — Are changed areas covered by tests?
4. **Dependencies** — Are other modules/APIs affected?
5. **MR comments** — Consider open points from Phase 3 in the analysis
6. **Focus from Phase 1** — prioritize named areas; only skim explicitly excluded areas

### Phase 5: Intensive code review (three passes)

**Respect focus:** When the user set emphasis (e.g. security, specific files), prioritize these in all passes and address explicitly in the Executive Summary.

#### Pass 1: Architecture & correctness

- Does the MR meet its stated goal?
- Logic errors, race conditions, edge cases?
- Breaking changes documented?
- Fits existing architecture?

#### Pass 2: Code quality & maintainability

- Readability, function size, naming
- Error handling complete?
- Consistency with project style
- No unnecessary changes

#### Pass 3: Security, performance & tests

- Security: injection, XSS, auth bypass, secrets
- Performance: N+1, missing pagination
- Tests and CI status
- Observability on critical paths

### Phase 6: Severity classification

| Level | Meaning | Merge blocker? |
|-------|---------|----------------|
| 🔴 **Critical** | Bug, security, data loss | Yes |
| 🟠 **Major** | Incorrect behavior, missing critical tests | Yes (usually) |
| 🟡 **Minor** | Style, naming | No |
| 🔵 **Suggestion** | Alternative approaches | No |
| 💬 **Question** | Unclear | No |

### Phase 7: Review report

**Chat title (strict — match detected platform):**

- **GitHub:** `# Code Review: PR #<id> — <title>`
- **GitLab:** `# Code Review: MR !<id> — <title>`

Use `#id` / “Pull Request” wording in the GitHub report; `!id` / “Merge Request” on GitLab. Do not mix dialects in one report. Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
# Code Review: PR #<id> — <title>

## Executive Summary
<2-4 sentences + merge recommendation>

**Recommendation:** ✅ Approve / ⚠️ Approve with comments / ❌ Request changes

## PR metadata
| Field | Value |
|-------|-------|
| Author | ... |
| Branch | `<source>` → `<target>` |
| Commits | ... |
| Files | ... (+/- lines) |
| Checks / CI | ✅/❌/⏳ |
| Existing comments | <count> (<open> open, <resolved> resolved) |

## Existing PR discussion
<Summary + table from Phase 3>

## Findings
### 🔴 Critical
| # | File | Problem | Recommendation | Already commented? |
...

## Positives
- ...

## Commit message analysis
- Conventional Commits followed? Yes/No

## Test plan assessment
- Test plan present? Missing tests?

## Next steps
1. ...
```

```markdown
<!-- GitLab skeleton (MR / !) -->
# Code Review: MR !<id> — <title>

## Executive Summary
<2-4 sentences + merge recommendation>

**Recommendation:** ✅ Approve / ⚠️ Approve with comments / ❌ Request changes

## MR metadata
| Field | Value |
|-------|-------|
| Author | ... |
| Branch | `<source>` → `<target>` |
| Commits | ... |
| Files | ... (+/- lines) |
| Pipeline | ✅/❌/⏳ |
| Existing comments | <count> (<open> open, <resolved> resolved) |

## Existing MR discussion
<Summary + table from Phase 3>

## Findings
### 🔴 Critical
| # | File | Problem | Recommendation | Already commented? |
...

## Positives
- ...

## Commit message analysis
- Conventional Commits followed? Yes/No

## Test plan assessment
- Test plan present? Missing tests?

## Next steps
1. ...
```

### Phase 8: MR/PR comments (optional, on request only)

**Output review report in chat** (Phase 7). **Do not** post to the platform automatically.

Posting only when the user **explicitly** asks (e.g. “post as PR comment”, “post critical & major only”, “post inline with suggestions”). Then:

1. Per finding with known diff position: **inline line comment** at `path` + `line` (do not dump everything into one general thread)
2. **GitHub:** When a clear line-level fix is possible → body includes a **`suggestion` block** (see **GitHub line comments + Suggestions**). Otherwise severity + explanation + optional normal code fence
3. Reference existing comments instead of duplicating
4. Optionally **one** summary comment / review body with recommendation + overview (Critical/Major counts); details stay on the lines
5. Formatting — **Content formatting — GitLab or GitHub**
6. Post:
   - **GitHub**: `gh api .../pulls/<id>/comments` (inline + suggestion) and/or `.../pulls/<id>/reviews` with `comments[]`; summary optional via `gh pr comment` — **GitHub GFM**
   - **GitLab**: Inline/diff notes via `glab api .../discussions` or MCP when position is known; optional `suggestion:-0+0`; summary via `glab mr note` — **GitLab Flavored Markdown**
7. Confirm success and return **URLs** of posted inline comments (and summary if any)

**Before posting**, show planned inline bodies (incl. suggestions) in chat and send only after confirmation — same ask-before-mutating principle as other write paths.

**Inline comment template** (GitHub, English, no automation markers):

````markdown
**<Severity>:** <one-sentence problem>

<optional short rationale>

```suggestion
<exact replacement for the commented line(s)>
```
````

**Summary template** (optional top-level / review body — English):

```markdown
## Code Review

**Recommendation:** Approve / Approve with comments / Request changes

### Summary
<2-3 sentences>

### Inline findings
Posted <n> line comments (Critical: <n>, Major: <n>, …). See threads on the Files changed tab.

### Positives
<What works well>

### Test plan
<Assessment>
```

---

## Review principles

1. **Concrete**: file + line + problem + fix suggestion — findings without line only when no diff anchor is possible
2. **Inline first**: when posting, comment on lines; on **GitHub** use suggestions as soon as the fix for exactly those lines is clear
3. **Constructive**: highlight what works well
4. **Prioritized**: Critical first
5. **No fix without request**: review ≠ implementation (suggestion ≠ locally applied fix)
6. **No duplicates**: load existing MR/PR comments, summarize, and do not repeat
7. **Include discussion**: consider author replies and resolved/unresolved status

## Error handling

| Problem | Action |
|---------|--------|
| No MR ID can be resolved | AskQuestion or abort (see above) |
| Multiple MRs in context | Ask user which MR is meant |
| MR/PR not found | Check ID and project |
| Unknown remote host | AskQuestion or abort — do not guess. No “push + manual URL” path |
| GitHub without `gh` auth | Abort, guide `gh auth login` |
| GitLab without `glab` and without MCP | Abort, guide CLI or MCP setup |
| MCP not authenticated | Guide auth, abort |
| MR/PR >500 lines diff | Review by module, recommend split |

## Related commands

- `/merge-request-fix [id] [instructions]` — implement review comments and create fix summary
- `/commit` — commit changes
- `/merge-request` — create MR/PR
