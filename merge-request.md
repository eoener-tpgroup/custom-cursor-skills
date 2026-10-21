---
name: merge-request
description: Pushes the current branch and creates a Merge Request (GitLab) or Pull Request (GitHub). Optional instructions or chat context. Confirmation before push/MR via AskQuestion.
---

# Merge Request

Pushes the current feature branch and creates a Merge/Pull Request against the default branch.
Triggered manually via `/merge-request [instructions]`.

## ⚙️ Optional arguments & context

Everything **after** `/merge-request` in the user prompt is optional free text (`[instructions]`). It may contain spaces.

**Two sources** (priority):

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit text** after `/merge-request` | User provided instructions directly |
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
✅ /merge-request
✅ /merge-request feat(auth): add JWT refresh — target develop
✅ /merge-request draft, focus on migration rollback test plan
✅ /merge-request breaking change: remove v1 API endpoints
```

## Output language

**Always write all user-facing output in English** — status messages, summaries, MR/PR descriptions, error messages, and questions to the user. This command definition is also in English. The user may still write in another language; respond in English regardless.

## Agency agent

**Before Phase 0**, read and adopt judgment/quality guidance from:

| Role | Path | When |
|------|------|------|
| Primary | `~/.cursor/rules/engineering-git-workflow-master.mdc` | Always |
| Optional | `~/.cursor/rules/project-management-jira-workflow-steward.mdc` | When a Jira ticket key appears in branch name, commits, instructions, or chat context |

**Precedence:** This command’s phases, dual-platform detection (GitHub/GitLab), Content formatting, Git Safety Protocol, and AskQuestion gates win on conflict. Prefer this command’s Conventional Commits–style titles and platform Markdown over Gitmoji or agent-specific branch templates unless the **repository** already uses those conventions. The agency agent guides clean history, review-ready summaries, and Jira traceability — it does not replace phases, invent platform clients, or override abort/confirmation rules.

If a listed agency-agent file is missing, continue the command workflow and note that the agency agent could not be loaded.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 3), format title and description for **that platform only** — whether creating via **`gh`**, **`glab`**, **GitLab MCP**, **GitHub/GitLab REST API**, or any other client.

| Platform | Description dialect | Do not use |
|----------|---------------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only refs (`!42` as MR shorthand in body when linking PRs), Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax / PR conventions that GitLab ignores, Jira Wiki/ADF |

Never post Jira markup into an MR/PR. Never mix HTML into the body.

### Shared field rules

| Field | Rules |
|-------|-------|
| **Title** | Plain text — **no** Markdown (`#`, `**`, lists). Short, imperative, max. ~72 chars when possible. No newlines |
| **Description / Body** | Platform Flavored Markdown: `##` headings, `-` lists, `- [ ]` checkboxes in Test plan, `**bold**`, `` `inline` ``, fenced code with language tag |
| **Structure** | Blank line between sections; fixed order: Summary → Changes → Test plan |
| **Links** | Full URLs (`https://…`); platform-native issue/MR links when appropriate |
| **Tables** | Only if needed; simple GFM/GitLab Markdown tables |

### Platform-specific Markdown

| Feature | GitHub (GFM) | GitLab Flavored Markdown |
|---------|--------------|--------------------------|
| Headings / lists / code | Standard GFM | Same common subset — prefer this |
| Task lists | `- [ ]` / `- [x]` | `- [ ]` / `- [x]` |
| Mentions | `@user` | `@user` / `@group` |
| Cross-refs | `#123` (issue/PR), commit SHAs | `#123` (issue), `!42` (MR), `$pipeline_id` where useful |
| Collapsed detail | `<details>` sparingly if needed | Same; prefer open sections for reviewability |
| Alerts / admonitions | Avoid GitHub alert blockquotes unless sure | Prefer plain `##` sections |

When unsure, stick to the **common subset** (headings, bullets, checkboxes, fenced code, links, `@mentions`) — it renders on both.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq -n --arg`, HEREDOC, or `--body-file` / MCP native JSON args
2. **Preserve newlines** — `\n` in JSON strings; real newlines in HEREDOC (do not squash the body onto one line)
3. **Special characters** — `"`, `\`, backticks via proper encoding
4. **No HTML** as structure (`<br>`, `<p>`, `<ul>`) — Markdown only
5. **Preview** — Phase 5 shows title + full description as rendered Markdown for the detected platform, not escaped JSON
6. **MCP schema first** — map draft fields to the tool’s parameters (`title`/`description` or equivalents); body remains platform Markdown

**GitHub (`gh` or REST):**

```bash
gh pr create --base <target> --title "<plain title>" --body-file /tmp/pr-body.md
# or --body "$(cat <<'EOF'
## Summary
...
EOF
)"
```

**GitLab (`glab`, MCP, or REST):**

```bash
glab mr create --target-branch <target> --title "<plain title>" --description "$(cat <<'EOF'
## Summary
...
EOF
)"
```

- MCP / REST: `title` = plain text (JSON-encoded); `description` = **GitLab Markdown** string (JSON-encoded)

### Pre-create checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Title plain text, no markup, no newlines
- [ ] Description uses `##` sections and blank lines
- [ ] Test plan is a checkbox list (`- [ ]`)
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Prerequisites

- Git repository with configured remote (`origin` or recognizable primary remote)
- Current branch is **not** the default branch
- At least one commit on the branch that is not yet on the default branch
- Branch was committed beforehand (→ `/commit` if needed)

## Git Safety Protocol (strict compliance)

- **Never** change `git config`
- **Never** force-push to `main`/`master` — warn the user
- **Never** use `--force` without explicit user request
- **Never** skip hooks

---

## Workflow

### Phase 0: Evaluate instructions & context

1. Extract text after `/merge-request` — if present, treat as **explicit instructions**
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

**Determine default branch** (same as `/commit`):

1. `git symbolic-ref refs/remotes/origin/HEAD`
2. Fallback: `main` → `master` → `develop`

**Target branch**: From instructions/context (`target <branch>`, `base <branch>`). If not specified → default branch.

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

### Phase 3: Detect remote platform (origin URL first)

**Order is mandatory:** resolve the remote URL **before** choosing a client or drafting body text.

1. Read `git remote get-url origin` (fallback: recognizable primary remote from `git remote -v`).
2. Classify from the URL only:

| Platform | Detection (URL only) | Client afterward |
|----------|----------------------|------------------|
| **GitHub** | URL contains `github.com`, or hostname contains `github` (e.g. `github.mycompany.com`) | `gh` CLI — then check `gh auth status` |
| **GitLab** | Hostname contains `gitlab` (incl. self-hosted, e.g. `gitlab.office.transporeon.com`) | `glab` **or** GitLab MCP (see client choice below) |
| **Unknown** | Neither rule matches | **Do not guess** — `AskQuestion` (GitHub vs GitLab) or abort. No probing with `gh`/`glab`/MCP for detection. No "push + manual URL" path |

**Important:** Availability of `gh`, `glab`, or GitLab MCP **must not override the platform**. URL → platform first, then client.

**GitLab client choice:** Whatever is available and authenticated. If **both** work → **prefer `glab`**; otherwise GitLab MCP. Both are not required.

**After detection:** All title/description text and chat labels exclusively in the dialect of the detected platform (see **Content formatting**). GitHub → "Pull Request", `#n`, GFM. GitLab → "Merge Request", `!n`, GitLab Flavored Markdown.

### Phase 4: Create MR/PR draft

Analyze commits and diff, design MR/PR content — **no push yet, no MR/PR creation yet**.

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

**Before `git push` and MR/PR creation** show the full plan and wait for confirmation with **`AskQuestion`**.

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

**AskQuestion** (required — no fallback to implicit continuation) — wording matches platform:

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

**Formatting:** See **Content formatting — GitLab or GitHub** — Title plain text; Description as **GitHub GFM** or **GitLab Flavored Markdown** depending on detected platform; applies to `gh` / `glab` / **MCP** / REST.

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
3. Use available MCP tools for MR creation — `title` plain text, `description` as **GitLab Flavored Markdown** (see Content formatting)

### Phase 8: Report result

- **URL** of the created MR/PR
- Branch name and target branch
- Number of commits and changed files
- Pipeline/CI status, if retrievable
- Note: `/merge-request-review [id] [focus]` for code review, `/merge-request-fix [id] [instructions]` for review fixes — ID as `#<n>` on GitHub, `!<n>` on GitLab

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

## Related commands

- `/commit [instructions]` — Commit changes
- `/merge-request-review [id] [focus]` — Code review
- `/merge-request-fix [id] [instructions]` — Address review comments
