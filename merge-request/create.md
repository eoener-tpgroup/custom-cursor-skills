# Merge Request — subcommand: create

Loaded by `/merge-request create …` after `merge-request/_common.md`. Do not run as a standalone command.

Pushes the current feature branch and creates a Merge/Pull Request against the default branch.

**Invocation:** `/merge-request create [instructions]`

Platform detection, Content formatting, and Git Safety: see `merge-request/_common.md`.

## Agency agent (Task dispatch — required)

**Before Phase 0:**

1. **Read** `lib/multi-agent-loop.md` (commands: `lib/multi-agent-loop.md`; skills: installed `lib/` copy).
2. Dispatch **Task** (`generalPurpose`) for Primary (and Optional when matched). Each prompt requires `Read` of the rule path. Bus = **in-session**.
3. Parent owns platform detection, AskQuestion, push/create — Task briefs guide quality only.

| Role | Path | When |
|------|------|------|
| Primary | `~/.cursor/rules/engineering-git-workflow-master.mdc` | Always |
| Optional | `~/.cursor/rules/project-management-jira-workflow-steward.mdc` | When a Jira ticket key appears in branch name, commits, instructions, or chat context |

**Precedence:** This module’s phases, dual-platform rules from `_common`, and AskQuestion gates win on conflict. Prefer Conventional Commits–style titles and platform Markdown over Gitmoji unless the **repository** already uses those conventions. Task briefs guide history/summaries/Jira traceability — they do not invent platform clients or override abort/confirmation.

Missing rule → note skipped. Task unavailable → in-parent fallback per lib.

## ⚙️ Optional arguments & context

Everything **after** `/merge-request create` in the user prompt is optional free text (`[instructions]`). It may contain spaces.

**Two sources** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit text** after `/merge-request create` | User provided instructions directly |
| 2 | **Chat context** | No explicit text — infer from the conversation so far |

**Typical instructions** (explicit or from context):

- MR/PR title or summary hints
- Target branch (`target develop`, `base main`)
- Test plan focus or breaking-change notes
- Labels, reviewers, draft status (when platform/CLI supports it)
- "draft MR" / "ready for review"

**Infer from context — rules:**

1. Derive title and summary from commits, diff, and chat — include discussed changes and ticket keys
2. Do not invent fictional test steps — derive from diff and context or keep generic
3. If target branch is unclear and not the default → **ask the user**

```
✅ /merge-request create
✅ /merge-request create feat(auth): add JWT refresh — target develop
✅ /merge-request create draft, focus on migration rollback test plan
✅ /merge-request create breaking change: remove v1 API endpoints
```

## Prerequisites

- Git repository with configured remote (`origin` or recognizable primary remote)
- Current branch is **not** the default branch
- At least one commit on the branch that is not yet on the default branch
- Branch was committed beforehand (→ `/commit` if needed)

---

## Workflow

### Phase 0: Evaluate instructions & context

1. Extract text after `/merge-request create` — if present, treat as **explicit instructions**
2. If empty: scan **chat context** for MR-relevant hints (feature description, ticket keys, test notes, breaking changes)
3. Translate instructions into concrete decisions:
   - MR/PR title
   - Summary / Changes / Test plan content
   - Target branch (if different from default)
   - Draft vs. ready
4. Unclear or conflicting instructions → **ask the user** before creating

### Phase 1: Check repository status

Run **in parallel**:

```bash
git status
git branch -vv
git remote -v
git log --oneline -10
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null || true
```

**Determine default branch** and **target branch**: see `merge-request/_common.md`. Target from instructions/context (`target <branch>`, `base <branch>`); if not specified → default branch.

### Phase 2: Validate preconditions

| Check | On failure |
|-------|------------|
| On default branch? | Abort: "Create a feature branch first with `/commit`" |
| Uncommitted changes? | Ask user: commit, stash, or abort |
| No remote? | Abort; have user configure remote |
| Branch has no commits vs. default? | Abort: "No changes to push" |

Optional:

```bash
git fetch origin
git log origin/<default-branch>..HEAD --oneline
```

### Phase 3: Detect remote platform

See `merge-request/_common.md` (Detect remote platform).

### Phase 4: Create MR/PR draft

Analyze commits and diff — **no push yet, no MR/PR creation yet**.

```bash
git log origin/<target-branch>..HEAD --oneline
git diff origin/<target-branch>..HEAD --stat
```

**Capture draft:**

- Title (Conventional Commits style; instructions/context take precedence)
- Full description (Summary, Changes, Test plan, Breaking changes if relevant)
- Target branch, source branch, platform, draft yes/no

**MR/PR description** — template:

```markdown
## Summary
<1-3 bullet points — what and why>

## Changes
<Short list of the most important changes>

## Test plan
- [ ] <Test step 1>
- [ ] <Test step 2>
```

### Phase 5: Show draft and obtain confirmation (required)

**Before `git push` and MR/PR creation** show the full plan and wait for confirmation with **`AskQuestion`** (or the same numbered options in chat if AskQuestion is unavailable).

**Do not automatically push or create MR/PR** — only after explicit confirmation.

Output **exactly one** skeleton below — match Phase 3 (do not keep a neutral `PR/MR` heading after detection).

```markdown
GitHub skeleton:
## Pull request plan

| Field | Value |
|-------|-------|
| Platform | GitHub |
| Source branch | `<current-branch>` |
| Target branch | `<target-branch>` |
| Commits | `<n>` |
| Changed files | `<n>` (+/- lines) |
| Draft | yes / no |

### Title
`<title>`

### Description
<full PR body — Summary, Changes, Test plan>

### Actions
1. `git push -u origin HEAD`
2. Create PR via `gh`
```

```markdown
GitLab skeleton:
## Merge request plan

| Field | Value |
|-------|-------|
| Platform | GitLab |
| Source branch | `<current-branch>` |
| Target branch | `<target-branch>` |
| Commits | `<n>` |
| Changed files | `<n>` (+/- lines) |
| Draft | yes / no |

### Title
`<title>`

### Description
<full MR body — Summary, Changes, Test plan>

### Actions
1. `git push -u origin HEAD`
2. Create MR via `glab` or GitLab MCP
```

**AskQuestion** (required — no implicit continuation) — wording matches platform. Prefer `AskQuestion` with the options below. If that tool is unavailable, present the **same numbered options in chat** and wait for an explicit choice. Do **not** proceed without confirmation.

> Proceed with push and pull request creation?  
> *(GitLab: “Proceed with push and merge request creation?”)*

Options:

1. **Yes, push and create PR** *(GitLab: create MR)* — Push and create PR/MR with the title and description above
2. **Edit first** — User provides corrections; adjust draft, confirm again
3. **Cancel** — Do not push, do not create PR/MR

On **Cancel**: Briefly confirm that nothing was pushed or created.

### Phase 6: Push branch (only after confirmation)

```bash
git push -u origin HEAD
```

On "upstream rejected" or diverged history: **do not** force-push — inform the user.

### Phase 7: Create Merge/Pull Request (only after confirmation)

**Formatting:** See `merge-request/_common.md`. Title plain text; Description as **GitHub GFM** or **GitLab Flavored Markdown**.

#### GitHub (via `gh`)

```bash
gh pr create --base <target-branch> --head <current-branch> --title "<title>" --body "$(cat <<'EOF'
<description from Phase 4 draft>
EOF
)"
```

#### GitLab (via `glab`)

```bash
glab mr create --target-branch <target-branch> --title "<title>" --description "$(cat <<'EOF'
<description from Phase 4 draft>
EOF
)"
```

#### GitLab (via MCP, when `glab` is not available)

1. Check GitLab MCP (`GetDynamicTools`, namespace `GitLab`)
2. If `needsAuth`: ask user to authenticate
3. Use available MCP tools for MR creation — `title` plain text, `description` as **GitLab Flavored Markdown**

### Phase 8: Report result

- **URL** of the created MR/PR
- Branch name and target branch
- Number of commits and changed files
- Pipeline/CI status, if retrievable
- Note: `/merge-request review [id] [focus]` · `/merge-request fix [id] [instructions]` — ID as `#<n>` on GitHub, `!<n>` on GitLab

---

## PR/MR description — best practices

1. **Summary**: What changed and why
2. **Test plan**: Concrete, checkable steps
3. **Breaking changes**: Call out explicitly
4. **Scope**: Prefer small, focused MRs

## Error handling

| Problem | Action |
|---------|--------|
| On default branch | Abort, recommend `/commit` |
| Push rejected | Show error, no force-push |
| Unknown remote host | AskQuestion or abort — do not guess |
| GitHub without `gh` auth | Abort, guide `gh auth login` |
| GitLab without `glab` and without MCP | Abort, guide CLI or MCP setup |
| Auth failed | Inform user |
| MR/PR already exists | Return existing URL |
| User cancels confirmation | Do not push, do not create MR/PR |
