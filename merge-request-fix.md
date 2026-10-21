---
name: merge-request-fix
description: Applies fixes for open code review comments on a merge request. MR ID optional. Confirmation before fixes and before commit/push via AskQuestion.
---

# Merge Request Fix

Reads **code review comments** for a merge/pull request by MR ID, implements **valid, actionable change requests** in the local codebase, and produces a **summary** of what was requested vs. what was fixed.

Triggered manually via `/merge-request-fix [id] [instructions]`.

## ⚙️ Arguments & context

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **MR ID** | conditional | Numeric IID/PR number (`42`, `!42`, `#42`) |
| 2 | **Instructions** | no | Free text — which comments to fix, how to proceed, what to ignore |

Everything after the MR ID (if present) is optional **Instructions** text and may contain spaces.

**Resolve MR ID** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit ID** in the user prompt | First valid token after `/merge-request-fix` (`42`, `!42`, `#42`) |
| 2 | **Chat context** | MR URL, `!42`, `MR 42`, `PR #42` in the conversation |
| 3 | **Current branch** | Find open MR/PR for the local branch via `gh`/`glab`/GitLab MCP |

**Instructions** (explicit or from context):

- Which comments/threads to address (`only inline comments on api.ts`, `fix critical and major only`)
- What **not** to implement (`skip the config restructure suggestion`)
- Approach (`add tests for edge case X`, `reply to @reviewer in MR comment`)
- Reference to prior review discussion in chat

**Derive from context — rules:**

1. Take instructions from chat when the prompt has no free text after the ID
2. **Do not guess** — if no ID can be resolved and no unambiguous MR in context → abort or ask the user
3. If multiple MRs are possible → **ask the user**

```
✅ /merge-request-fix 42
✅ /merge-request-fix !42 only fix null-check comments, skip architecture feedback
✅ /merge-request-fix address @reviewer's SQL injection comment
✅ /merge-request-fix                    → ID from branch or chat context
❌ /merge-request-fix fix                → no valid ID token, no context → abort
```

### Missing or invalid MR ID

When neither argument, context, nor branch search yields a unique MR ID:

1. **Prefer `AskQuestion`**: ask for MR ID or branch/URL
2. If unanswered → **abort** with:

> This command requires a merge request ID (or enough context to identify one).
> Usage: `/merge-request-fix [id] [instructions]`
> Example: `/merge-request-fix 42 only fix critical inline comments`

The MR ID appears in the user prompt **after the command name** (e.g. `/merge-request-fix 42`). Optional instructions follow after that.

---

## Output language

The command definition is in **English**. **Always write all user-facing output in English** — status messages, fix summaries, error messages, questions to the user, and MR/PR comments. This applies even if the user writes in another language.

## Agency agent

**Before Phase 1**, read and adopt judgment/quality guidance from:

| Role | Path |
|------|------|
| Primary | `~/.cursor/rules/engineering-minimal-change-engineer.mdc` |

**Precedence:** This command’s phases, triage categories, dual-platform detection, Content formatting, Git Safety Protocol, and AskQuestion gates win on conflict. The agency agent enforces minimal diffs and refusal of scope creep — it does not expand fixes beyond actionable review items, replace phases, or override abort/confirmation rules. When project conventions under `.cursor/rules/` or `CONTRIBUTING.md` conflict with a review comment, note the conflict in the summary rather than silently over-refactoring.

If a listed agency-agent file is missing, continue the command workflow and note that the agency agent could not be loaded.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 2), format MR/PR reply comments for **that platform only** — whether posting via **`gh`**, **`glab`**, **GitLab MCP**, or **GitHub/GitLab REST API**.

| Platform | Comment dialect | Do not use |
|----------|-----------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only `!MR` shorthand as primary link style, Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax, Jira Wiki/ADF |

Never post Jira markup into MR/PR comments. Never use raw HTML for structure.

### Comment body rules

| Field | Rules |
|-------|-------|
| **Comment body** | Platform Flavored Markdown: `## Review feedback addressed`, one bullet per fix, `@mention` reviewers when useful |
| **Structure** | Short intro → addressed items → still-open items (e.g. **Still open (non-blocking):**) |
| **File references** | `` `path/to/file.ts` `` or `file.ts:42` — match existing review comment style on that platform |
| **Status** | Per item: fixed / skipped / why |
| **Cross-refs** | GitHub: `#123`, commit SHA · GitLab: `#123`, `!42` when linking MRs |

Prefer the **common Markdown subset** (headings, bullets, code fences, links, `@mentions`) so rendering is reliable on both platforms.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq`, HEREDOC, `--body-file`, or MCP native JSON args
2. **Preserve newlines** — lists/paragraphs need real newlines (`\n` in JSON)
3. **Special characters** — code snippets and paths via proper JSON encoding
4. **No HTML** — Markdown instead of `<br>` / `<ul>`
5. **Preview** — Phase 5 and Phase 8 show the MR comment as platform Markdown, not escaped JSON
6. **MCP schema first** — put the Markdown string in the tool’s `body` / `note` field as required by the schema

**GitHub (`gh` or REST) — GFM:**

```bash
gh pr comment <id> --body-file /tmp/pr-reply.md
```

**GitLab (`glab`, MCP, or REST) — GitLab Flavored Markdown:**

```bash
glab mr note <id> -m "$(cat <<'EOF'
## Review feedback addressed
...
EOF
)"
```

- MCP / REST: `body` = platform Markdown string, JSON-encoded via tool args or `json.dumps`

### Pre-post checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Each addressed review point is its own bullet
- [ ] Open / not-implemented items called out explicitly
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Workflow

### Phase 1: Validate arguments & resolve MR ID

1. Parse user prompt after `/merge-request-fix`
2. Check first token: numeric IID/PR number (`42`, `!42`, `#42`) → **MR ID**
3. Remaining text → **Instructions** (scope, priorities, exclusions)
4. If no valid ID token: check **chat context** and **current branch** (see above)
5. Supplement instructions from context when the prompt has no free text after the ID
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
| **Unknown** | None of the rules match | **Do not guess** — `AskQuestion` (GitHub vs GitLab) or abort. No probing with `gh`/`glab`/MCP for detection. No “push + manual URL” path |

**Important:** Availability of `gh`, `glab`, or GitLab MCP **must not override platform detection**. URL → platform first, then client.

**GitLab client choice:** Whatever is available and authenticated. If **both** work → **prefer `glab`**; otherwise GitLab MCP. Both are not required.

**After detection:** Fix plan, summary, commit footer, and reply drafts use the detected platform only — GitHub → `PR #<id>`, `#n`, GFM; GitLab → `MR !<id>`, `!n`, GitLab Flavored Markdown (Phases 5, 7, 8). See **Content formatting — GitLab or GitHub**.

Then load data via the matching subsection below (**GitLab** or **GitHub**).

#### GitLab (via `glab` or MCP)

**Client:** Prefer **`glab`** when both `glab` and GitLab MCP are available and authenticated; otherwise GitLab MCP. Tool discovery for MCP: `GetDynamicTools`, Namespace `GitLab`. On `needsAuth`: ask the user to authenticate; **do not** proceed without auth.

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
2. Call the appropriate MCP tools (same data types as below)

| Data | Purpose |
|------|---------|
| MR details | Title, description, author, status, source/target branch |
| MR diffs | Context for inline comments and affected files |
| MR commits | Commit history |
| MR discussions / notes | **All** review comments, threads, inline notes |
| MR approvals | Review states (`CHANGES_REQUESTED`, etc.) |

**GitLab — load comments (required):**

Via **`glab`** (when chosen as client):

1. `glab mr view <id> --comments` — quick overview of notes
2. Resolve/encode project path, then `glab api "projects/${PROJECT}/merge_requests/<id>/discussions"` — full threads including resolved/unresolved and inline positions (`PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)`)

Via **GitLab MCP** (when chosen as client):

1. Check tool schema via `GetDynamicTools` (e.g. `list_merge_request_discussions`, `get_merge_request_notes`, `get_merge_request`)
2. Load **all** discussions and notes — including:
   - General MR comments (discussion notes)
   - Inline comments on code lines (diff notes)
   - Thread replies (full conversation)
   - Review notes with `CHANGES_REQUESTED` status
3. For each comment capture: author, date, file/line (if inline), content, resolved/unresolved, thread ID

**GitLab — post comment (Phase 8, after confirmation):**

- `glab mr note <id> -m "..."` or MCP (`create_workitem_note` or equivalent — schema first)
- Body = **GitLab Flavored Markdown** (see **Content formatting — GitLab or GitHub**)

#### GitHub (via `gh`)

```bash
gh pr view <id> --json title,body,state,author,baseRefName,headRefName,files,reviews,reviewDecision
gh pr diff <id>
gh api repos/{owner}/{repo}/pulls/<id>/comments
gh api repos/{owner}/{repo}/issues/<id>/comments
gh api graphql -f query='...'  # for unresolved review threads, if needed
```

**GitHub — load comments (required):**

- Issue comments (general PR discussion)
- Review comments (`pulls/{id}/comments` — inline on code lines)
- Review submissions (`CHANGES_REQUESTED` + body)
- Include threads and replies in full
- Consider resolved/unresolved status

**GitHub — post comment (Phase 8, after confirmation):**

- `gh pr comment <id> --body-file /tmp/pr-reply.md` or `--body` with HEREDOC
- Body = **GitHub GFM** (see **Content formatting — GitLab or GitHub**)
- **Resolve implemented review threads (required when applicable):** After posting the reply (or as part of the same Option-4 path), mark every **unresolved** review thread whose finding was **✅ Fixed** in this run as resolved. Do **not** resolve threads that remain open / skipped / not implemented.

```bash
# List unresolved review threads on the PR
gh api graphql -f query='
query($owner:String!,$name:String!,$number:Int!) {
  repository(owner:$owner, name:$name) {
    pullRequest(number:$number) {
      reviewThreads(first: 100) {
        nodes { id isResolved path comments(first: 1) { nodes { body } } }
      }
    }
  }
}' -F owner=OWNER -F name=REPO -F number=<id>

# Resolve one thread (repeat for each fixed finding)
gh api graphql -f query='
mutation($id:ID!) {
  resolveReviewThread(input:{threadId:$id}) { thread { id isResolved } }
}' -f id='PRRT_...'
```

### Phase 3: Prepare local repository

1. Verify the current directory is a Git repository
2. Align MR source branch with local branch:

```bash
git status
git branch -vv
git fetch origin
```

3. **Switch branch** if needed:

   - If local branch ≠ MR source branch → switch to MR source branch (`git checkout <source-branch>`)
   - Branch missing locally → `git fetch origin <source-branch>:<source-branch>` or `git checkout -b <source-branch> origin/<source-branch>`

4. Check uncommitted changes:
   - If uncommitted changes exist: inform the user and **do not** overwrite — have them stash or commit
5. Optional: `git pull --rebase origin <source-branch>` to stay up to date (no force)

### Phase 4: Triage review comments

**Honor Phase 1 instructions:** Only process comments mentioned in instructions or within scope. Mark explicitly excluded items in the summary as “not implemented (per instruction)”.

Process **only actionable, open review items**. Classify comments:

| Category | Action |
|----------|--------|
| ✅ **Actionable** | Concrete change request, bug, missing test, naming fix | → Plan fix |
| ⏭️ **Resolved** | Already marked resolved or author confirmed | → Skip |
| 💬 **Question** | Follow-up without a clear fix | → Skip, note as “open” in summary |
| ❌ **Invalid / Out of scope** | Feedback contradicts requirements or is incorrect | → Do not fix, explain in summary |
| 🤖 **Bot (Bugbot etc.)** | Automated review comment | → Review content, fix only on **valid** finding |
| 📝 **Review report only** | General review without a concrete fix (e.g. `/merge-request-review` output) | → Extract individual findings and treat as actionable |

**Priority:**

0. **Instructions/context** — user-named comments, files, or severities first
1. Unresolved inline comments on code lines
2. `CHANGES_REQUESTED` reviews
3. General MR comments with a concrete change request
4. Findings from automated review comments (Critical/Major first)

**Merge duplicates:** Multiple comments on the same issue → one fix, multiple references in the summary.

**No code changes yet** — only finalize the fix plan for Phase 5.

### Phase 5: Show fix plan and get confirmation (required)

**Before any code change**, show the full fix plan and all generated text and wait for confirmation via **`AskQuestion`**.

**Do not apply fixes automatically** — only after explicit confirmation.

**Labels (strict — match detected platform):**

- **GitHub:** `PR #<id>` in the fix plan table and summary; commit footer `address PR #<id> review comments`
- **GitLab:** `MR !<id>`; commit footer `address MR !<id> review comments`

Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
## Fix plan

| Field | Value |
|-------|-------|
| PR | #<id> — <title> |
| Branch | `<source>` → `<target>` |
| Actionable comments | <n> |
| To fix | <n> |
| To skip | <n> |
| Open / manual | <n> |

### Planned fixes
| # | Source | File | Requirement | Planned change |
|---|--------|------|-------------|----------------|
| 1 | @reviewer, `file.ts:42` | `file.ts` | Null-check missing | Add guard in `handleRequest()` |
| 2 | ... | ... | ... | ... |

### Skipped / not implemented
| # | Source | Reason |
|---|--------|--------|
| 3 | @reviewer | Already resolved |
| 4 | @reviewer | Architecture decision — needs discussion |

### Draft commit message
```
fix(review): address PR #<id> review comments

- <change 1>
- <change 2>
```

### Draft PR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro if replying to a specific thread>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Actions
1. Apply listed code changes locally
2. Produce fix summary report
3. *(Later, separate confirmation)* Commit / push / post PR comment
```

```markdown
<!-- GitLab skeleton (MR / !) -->
## Fix plan

| Field | Value |
|-------|-------|
| MR | !<id> — <title> |
| Branch | `<source>` → `<target>` |
| Actionable comments | <n> |
| To fix | <n> |
| To skip | <n> |
| Open / manual | <n> |

### Planned fixes
| # | Source | File | Requirement | Planned change |
|---|--------|------|-------------|----------------|
| 1 | @reviewer, `file.ts:42` | `file.ts` | Null-check missing | Add guard in `handleRequest()` |
| 2 | ... | ... | ... | ... |

### Skipped / not implemented
| # | Source | Reason |
|---|--------|--------|
| 3 | @reviewer | Already resolved |
| 4 | @reviewer | Architecture decision — needs discussion |

### Draft commit message
```
fix(review): address MR !<id> review comments

- <change 1>
- <change 2>
```

### Draft MR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro if replying to a specific thread>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Actions
1. Apply listed code changes locally
2. Produce fix summary report
3. *(Later, separate confirmation)* Commit / push / post MR comment
```

**AskQuestion** (required — no fallback to implicit continuation):

> Proceed with applying these fixes?

Options:

1. **Yes, apply fixes** — Implement planned changes in code
2. **Edit plan first** — User adjusts scope, individual fixes, or text; update plan and confirm again
3. **Cancel** — Make no code changes

On **Cancel**: Briefly confirm that no fixes were applied.

### Phase 6: Apply fixes (only after confirmation)

For each **actionable** item:

1. **Read affected file(s)** — not only the diff hunk, but surrounding context
2. **Follow project conventions** — `.cursor/rules/`, `CONTRIBUTING.md`, existing style
3. **Minimal, focused fix** — change only what is needed, no drive-by refactoring
4. **Stay in scope** — do not change CI workflows just to force green checks; no unrelated changes
5. After each fix, briefly verify (syntax, obvious logic)

**Do not auto-fix without a clear basis:**

- Architecture decisions that need discussion
- Contradictory feedback between reviewers
- Breaking changes without explicit requirement in the comment
- Security-related changes requiring design decisions → mark as “needs manual clarification”

**Tests:** When a comment requests missing tests and the fix is clear → add tests.

### Phase 7: Produce summary

At the end **always** output the following report (even if nothing was fixed):

**Chat title (strict — match detected platform):**

- **GitHub:** `# Fix Summary: PR #<id> — <title>`
- **GitLab:** `# Fix Summary: MR !<id> — <title>`

Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
# Fix Summary: PR #<id> — <title>

## Overview
| Metric | Value |
|--------|-------|
| PR | #<id> — <title> |
| Branch | `<source>` → `<target>` |
| Comments total | <n> |
| Actionable | <n> |
| Fixed | <n> |
| Skipped | <n> |
| Open / manual | <n> |

## Requested → Fixed

| # | Source | File | Requirement | Status | Implementation |
|---|--------|------|-------------|--------|----------------|
| 1 | @reviewer, inline `file.ts:42` | `file.ts` | Missing null-check | ✅ Fixed | Guard added in `handleRequest()` |
| 2 | ... | ... | ... | ... | ... |

**Status legend:** ✅ Fixed · ⏭️ Skipped (resolved/duplicate) · ❌ Not implemented (with reason) · 💬 Open (needs clarification)

## Changed files
- `path/to/file.ts` — <short description>
- ...

## Not implemented (reason)
1. ...

## Next steps
1. Review changes (`git diff`)
2. Commit with `/commit` or manually
3. Push and update PR (`/merge-request` or `git push`)
4. Optional: Reply on PR; on GitHub, resolve review threads for ✅ Fixed findings
```

```markdown
<!-- GitLab skeleton (MR / !) -->
# Fix Summary: MR !<id> — <title>

## Overview
| Metric | Value |
|--------|-------|
| MR | !<id> — <title> |
| Branch | `<source>` → `<target>` |
| Comments total | <n> |
| Actionable | <n> |
| Fixed | <n> |
| Skipped | <n> |
| Open / manual | <n> |

## Requested → Fixed

| # | Source | File | Requirement | Status | Implementation |
|---|--------|------|-------------|--------|----------------|
| 1 | @reviewer, inline `file.ts:42` | `file.ts` | Missing null-check | ✅ Fixed | Guard added in `handleRequest()` |
| 2 | ... | ... | ... | ... | ... |

**Status legend:** ✅ Fixed · ⏭️ Skipped (resolved/duplicate) · ❌ Not implemented (with reason) · 💬 Open (needs clarification)

## Changed files
- `path/to/file.ts` — <short description>
- ...

## Not implemented (reason)
1. ...

## Next steps
1. Review changes (`git diff`)
2. Commit with `/commit` or manually
3. Push and update MR (`/merge-request` or `git push`)
4. Optional: Reply on MR / resolve discussion threads
```

### Phase 8: Next steps — show text and get confirmation (required)

**After the summary**, show the planned **commit message** and **MR/PR comment** in full again and wait for confirmation via **`AskQuestion`**.

**Do not commit, push, or post MR/PR comments automatically** — only after explicit confirmation.

Use **PR #<id>** / PR wording on GitHub; **MR !<id>** / MR wording on GitLab. Output **exactly one** skeleton below.

```markdown
<!-- GitHub skeleton (PR / #) -->
## Next steps plan

### Commit message
```
fix(review): address PR #<id> review comments

- <change 1>
- <change 2>
```

### PR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Changed files
- `path/to/file.ts` — <description>
```

```markdown
<!-- GitLab skeleton (MR / !) -->
## Next steps plan

### Commit message
```
fix(review): address MR !<id> review comments

- <change 1>
- <change 2>
```

### MR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Changed files
- `path/to/file.ts` — <description>
```

**AskQuestion** (required — no fallback to implicit continuation):

> How should the fixes be handled next?

Options:

1. **Summary only** — Fixes remain local and uncommitted; post nothing
2. **Commit** — Create commit with the message above
3. **Commit + push** — Commit and `git push` to the PR/MR branch
4. **Commit + push + PR/MR comment** — Also post the PR/MR comment above; **on GitHub**, then resolve implemented review threads (see below)
5. **Edit texts first** — User adjusts commit message or PR/MR comment; confirm again

On **Commit** / **Commit + push** / **Commit + push + PR/MR comment**:

- Follow Git Safety Protocol (see `/commit`)
- Commit message references MR ID and briefly lists main fixes
- Do not commit secrets

When **posting MR/PR comment** (Option 4 — text from draft above, English, no automation markers; follow **Content formatting — GitLab or GitHub** — MCP and REST/CLI):
- **GitLab**: `glab mr note <id> -m "..."` or MCP note tool — body = **GitLab Flavored Markdown**
- **GitHub**: `gh pr comment <id> --body-file /tmp/pr-reply.md` or `--body` with HEREDOC — body = **GitHub GFM**
- **GitHub — resolve threads:** If review threads for **✅ Fixed** findings are still unresolved → resolve via GraphQL `resolveReviewThread` (see **GitHub — post comment**). Skip threads for skipped / open / not-implemented items. Report how many threads were resolved.

On success: return the **comment URL** (and briefly: `Resolved <n> review thread(s)` on GitHub when applicable).

On **Summary only** / **Cancel**: Briefly confirm that nothing was committed, pushed, or posted.

---

## Fix principles

1. **Review comments are the source of truth** — fix only what is explicitly or clearly implicitly requested
2. **Minimal scope** — smallest correct fix per item
3. **Validate, don’t blindly follow** — do not implement incorrect or contradictory feedback; explain instead
4. **No duplicates** — do not re-address resolved/already-addressed comments
5. **Read context** — consider thread history and author replies
6. **Transparency** — summary must reflect every review item with status
7. **GitHub: resolve when fixed** — After Option 4 (PR comment), resolve review threads for findings that were implemented; leave others open

## Git Safety Protocol (strict)

- **Never** change `git config`
- **Never** run destructive Git commands (`push --force`, `reset --hard`, etc.) without explicit user request
- **Never** skip hooks (`--no-verify`, `--no-gpg-sign`, etc.)
- **Never** force-push to `main`/`master`
- **Do not** commit secrets

## Error handling

| Problem | Action |
|---------|--------|
| MR ID cannot be resolved | AskQuestion or abort (see above) |
| Multiple MRs in context | Ask user which MR/PR is meant |
| MR/PR not found | Verify ID and project |
| Unknown remote host | AskQuestion or abort — do not guess. No “push + manual URL” path |
| GitHub without `gh` auth | Abort, guide `gh auth login` |
| GitLab without `glab` and without MCP | Abort, guide CLI or MCP setup |
| MCP not authenticated | Guide auth, abort |
| No local repo / wrong branch | Inform user, offer branch checkout |
| No open review comments | Report and summary with 0 actionable |
| Contradictory feedback | Do not fix; mark in summary as “needs manual clarification” |
| Fix would change CI workflow | Do not auto-apply — inform user |
| User cancels confirmation | Do not apply fixes and/or do not commit/push/post |

## Related commands

- `/merge-request-review [id] [focus]` — In-depth code review (produce findings)
- `/commit` — Commit changes
- `/merge-request` — Push branch and create/update MR
