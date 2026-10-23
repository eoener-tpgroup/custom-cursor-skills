# Merge Request — shared module

Loaded by `/merge-request` before every subcommand. Do not run as a standalone command.

## Git Safety Protocol (strict)

- **Never** change `git config`
- **Never** force-push to `main`/`master` — warn the user
- **Never** use `--force` / destructive git (`reset --hard`, etc.) without explicit user request
- **Never** skip hooks (`--no-verify`, `--no-gpg-sign`, etc.)
- **Do not** commit secrets

## Detect remote platform (origin URL first)

**Order is mandatory** for every subcommand: resolve the remote URL **before** choosing a client or drafting body text.

1. Read `git remote get-url origin` (fallback: recognizable primary remote from `git remote -v`).
2. Classify from the URL only:

| Platform | Detection (URL only) | Client afterward |
|----------|----------------------|------------------|
| **GitHub** | URL contains `github.com`, or hostname contains `github` (e.g. `github.mycompany.com`) | `gh` CLI — then check `gh auth status` |
| **GitLab** | Hostname contains `gitlab` (incl. self-hosted, e.g. `gitlab.office.transporeon.com`) | `glab` **or** GitLab MCP (see client choice below) |
| **Unknown** | Neither rule matches | **Do not guess** — `AskQuestion` (GitHub vs GitLab) or abort. No probing with `gh`/`glab`/MCP for detection. No "push + manual URL" path |

**Important:** Availability of `gh`, `glab`, or GitLab MCP **must not override the platform**. URL → platform first, then client.

**GitLab client choice:** Whatever is available and authenticated. If **both** work → **prefer `glab`**; otherwise GitLab MCP. Both are not required.

**After detection:** All title/description/comment text and chat labels exclusively in the dialect of the detected platform. GitHub → "Pull Request", `#n`, GFM. GitLab → "Merge Request", `!n`, GitLab Flavored Markdown.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection, format for **that platform only** — whether via **`gh`**, **`glab`**, **GitLab MCP**, or **GitHub/GitLab REST API**.

| Platform | Dialect | Do not use |
|----------|---------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only `!MR` shorthand as primary link style, Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax / PR conventions that GitLab ignores, Jira Wiki/ADF |

Never post Jira markup into an MR/PR. Never mix HTML into the body.

### Shared field rules

| Field | Rules |
|-------|-------|
| **Title** | Plain text — **no** Markdown. Short, imperative, max. ~72 chars when possible. No newlines |
| **Description / comment body** | Platform Flavored Markdown: `##` headings, `-` lists, `- [ ]` checkboxes, `**bold**`, `` `inline` ``, fenced code with language tag |
| **Structure** | Blank line between sections |
| **Links** | Full URLs (`https://…`); platform-native issue/MR links when appropriate |
| **Cross-refs** | GitHub: `#123`, commit SHA · GitLab: `#123`, `!42`, `$pipeline_id` where useful |

When unsure, stick to the **common subset** (headings, bullets, checkboxes, fenced code, links, `@mentions`) — it renders on both.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq -n --arg`, HEREDOC, or `--body-file` / MCP native JSON args
2. **Preserve newlines** — `\n` in JSON strings; real newlines in HEREDOC
3. **Special characters** — `"`, `\`, backticks via proper encoding
4. **No HTML** as structure (`<br>`, `<p>`, `<ul>`) — Markdown only
5. **Preview** — show drafts as rendered Markdown for the detected platform, not escaped JSON
6. **MCP schema first** — map draft fields to the tool’s parameters; body remains platform Markdown

### Pre-write checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Title (if any) plain text, no markup, no newlines
- [ ] Body uses `##` sections and blank lines where structured
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Resolve MR/PR ID (for `review` and `fix`)

| Priority | Source | When |
|----------|--------|------|
| 1 | **Explicit ID** in the user prompt | First valid token after subcommand (`42`, `!42`, `#42`) |
| 2 | **Chat context** | MR URL, `!42`, `MR 42`, `PR #42` in the conversation |
| 3 | **Current branch** | Find open MR/PR for the local branch via `gh`/`glab`/GitLab MCP |

**Do not guess.** If no ID can be resolved and no unambiguous MR in context → `AskQuestion` or abort. If multiple MRs are possible → **ask the user**.

## Default branch detection

1. `git symbolic-ref refs/remotes/origin/HEAD`
2. Fallback: `main` → `master` → `develop`
