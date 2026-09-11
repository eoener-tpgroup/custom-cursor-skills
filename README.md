# Custom Cursor Skills & Commands

Reusable agent workflows for day-to-day engineering work: Git commits and merge requests, Jira tickets, and read-only remote host / PostgreSQL analysis.

Each `*.md` file in this repo is one workflow (for example `commit.md` → `/commit` on the CLI, or a `commit` skill in the IDE). Place them as **CLI commands**, **IDE skills**, or both.

---

## What’s in this repo

| Name | File | Purpose |
|------|------|---------|
| `commit` | [`commit.md`](commit.md) | Conventional Commits + optional feature branch; ansible: never commit on main/master/env branches |
| `merge-request` | [`merge-request.md`](merge-request.md) | Push branch and create GitHub PR or GitLab MR |
| `merge-request-review` | [`merge-request-review.md`](merge-request-review.md) | Intensive MR/PR code review |
| `merge-request-fix` | [`merge-request-fix.md`](merge-request-fix.md) | Apply actionable review feedback locally |
| `jira-read` | [`jira-read.md`](jira-read.md) | Load and summarize a Jira ticket |
| `jira-create` | [`jira-create.md`](jira-create.md) | Create a Jira Service Desk ticket (PITOPS) |
| `jira-comment` | [`jira-comment.md`](jira-comment.md) | Post a comment on a Jira ticket |
| `remote-analyze` | [`remote-analyze.md`](remote-analyze.md) | Read-only SSH analysis (host / PostgreSQL / Barman) |
| `postgres-optimize` | [`postgres-optimize.md`](postgres-optimize.md) | Read-only PostgreSQL performance tuning advice |

---

## Place as Cursor CLI commands

CLI slash-commands live as flat Markdown files:

```text
~/.cursor/commands/<name>.md
```

Invoked as `/<name> …` (for example `/commit`, `/jira-read PITOPS-123`).

### Clone, then copy

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/commands
cp ~/src/custom-cursor-skills/*.md ~/.cursor/commands/
# Optional: keep README out of the commands folder
rm -f ~/.cursor/commands/README.md
```

### Clone, then symlink (easy updates via `git pull`)

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/commands
for f in ~/src/custom-cursor-skills/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  ln -sfn "$f" ~/.cursor/commands/"$(basename "$f")"
done
```

### Use the clone as `~/.cursor/commands`

Only if you want this repo itself to *be* your commands directory:

```bash
# Backup any existing commands first, then:
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/.cursor/commands
```

Reload Cursor / the CLI after installing so new commands appear.

---

## Place as Cursor IDE skills

IDE skills live as a directory per skill with a required `SKILL.md`:

| Scope | Path |
|-------|------|
| Personal (all projects) | `~/.cursor/skills/<name>/SKILL.md` |
| Project (shared in a repo) | `<project>/.cursor/skills/<name>/SKILL.md` |

Do **not** install into `~/.cursor/skills-cursor/` — that path is reserved for Cursor’s built-in skills.

### Personal skills from this repo

Each repo file becomes one skill folder; the file content is `SKILL.md`:

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/skills

for f in ~/src/custom-cursor-skills/*.md; do
  name="$(basename "$f" .md)"
  [ "$name" = "README" ] && continue
  mkdir -p ~/.cursor/skills/"$name"
  cp "$f" ~/.cursor/skills/"$name"/SKILL.md
  # Or symlink instead of copy:
  # ln -sfn "$f" ~/.cursor/skills/"$name"/SKILL.md
done
```

### Project skills (team / repo-local)

From a project root:

```bash
REPO=~/src/custom-cursor-skills   # path to your clone of this repo
mkdir -p .cursor/skills

for f in "$REPO"/*.md; do
  name="$(basename "$f" .md)"
  [ "$name" = "README" ] && continue
  mkdir -p .cursor/skills/"$name"
  cp "$f" .cursor/skills/"$name"/SKILL.md
done
```

Commit `.cursor/skills/` with the project if the whole team should get them.

### Using skills in the IDE

After placement, invoke by name in Agent chat, or with the same slash form as the CLI:

```text
/commit
/merge-request-review !42 focus on auth
/remote-analyze database db01.example.com replication lag

Use the commit skill for these changes
Follow merge-request-review on MR !42, focus on auth
Run remote-analyze as a skill: database db01.example.com replication lag
```

---

## CLI vs IDE (same content, different location)

| | Cursor CLI **commands** | Cursor IDE **skills** |
|--|-------------------------|------------------------|
| Location | `~/.cursor/commands/<name>.md` | `~/.cursor/skills/<name>/SKILL.md` or `.cursor/skills/<name>/SKILL.md` |
| Layout | One flat `.md` file | One folder + `SKILL.md` |
| Invoke | `/commit`, `/jira-read …` | Slash commands in Agent chat (`/commit …`) or ask by skill name |
| Same file? | Yes — copy or symlink the repo `.md` as the command file or as `SKILL.md` | Yes |

You can install **both**: keep CLI copies under `~/.cursor/commands/` and skill copies under `~/.cursor/skills/`.

---

## Prerequisites (by family)

| Area | Needs |
|------|--------|
| Git / MR workflows | Local git repo; `gh` (GitHub) and/or `glab` / GitLab MCP (GitLab) |
| Jira workflows | Jira MCP **or** `JIRA_URL` + `JIRA_TOKEN` in `~/.cursor/.env` |
| Remote / Postgres workflows | Working SSH access to the target host (keys / agent) |

---

## Shared guidelines (all commands)

These rules appear across the command definitions and should be treated as hard constraints:

1. **User-facing output is English** — status, reports, drafts, and questions — even if the user writes in another language.
2. **Ask before mutating** — commits, pushes, MR/PR creation, Jira writes, and applying review fixes require an explicit confirmation (`AskQuestion`) with a visible draft/plan.
3. **Do not guess required IDs** — missing MR IDs, ticket keys, FQDNs, or system types → ask or abort with the documented usage message.
4. **No secrets** — never print tokens, commit credentials, or put passwords/API keys into tickets or MR comments.
5. **Platform-correct markup** — GitHub GFM vs GitLab Markdown for MR/PR text; Jira Markdown / Wiki / ADF for Jira — never mix dialects or paste the wrong platform’s markup.
6. **Git safety** — no `git config` changes, no force-push to `main`/`master`, no skipping hooks, no destructive git without an explicit user request.
7. **Remote analysis is read-only** — `/remote-analyze` and `/postgres-optimize` must never write, restart services, or run DDL/DML on the remote host.

---

## Command reference

### `/commit`

Creates a Conventional Commits commit. On the default branch it can create a feature branch first. For `pitops/ansible` it never commits on `main` / `master` / `development` / `integration` / `acceptance` / `production` — feature branches are created from `development`.

| | |
|--|--|
| **Can do** | Inspect status/diff/log; plan files to stage; draft Conventional Commit message; create `type/short-description` branch; stage and commit after confirmation |
| **Cannot do** | Commit without confirmation; amend unless user-rule conditions are met; skip hooks; commit secrets (`.env`, credentials, tokens); invent a message when diff + context are insufficient; commit on protected ansible branches (even if the user asks) |
| **Guidelines** | Prefer one logical change per commit; subject imperative, ≤72 chars; body explains *why*; issue refs in footer; ask when instructions conflict with the diff; ansible: AskQuestion when on main/master/integration/acceptance/production |

**Usage examples**

```text
/commit
/commit fix(api): handle null response in webhook retry
/commit only stage auth files, type fix scope auth
/commit refs PROJ-789 — branch fix/webhook-race
```

---

### `/merge-request`

Pushes the current feature branch and creates a GitHub Pull Request or GitLab Merge Request against the target/default branch.

| | |
|--|--|
| **Can do** | Detect GitHub vs GitLab; draft title + Summary / Changes / Test plan; push with `-u`; create PR/MR via `gh`, `glab`, or GitLab MCP; return the MR/PR URL |
| **Cannot do** | Run from the default branch; push/create without confirmation; force-push; invent fake test steps; use Jira markup in the MR/PR body |
| **Guidelines** | Title = plain text (no Markdown); description = platform Markdown only; confirm plan before push; if MR already exists, return existing URL |

**Usage examples**

```text
/merge-request
/merge-request feat(auth): add JWT refresh — target develop
/merge-request draft, focus on migration rollback test plan
/merge-request breaking change: remove v1 API endpoints
```

---

### `/merge-request-review`

Runs an intensive, structured code review of an MR/PR, including existing discussion threads.

| | |
|--|--|
| **Can do** | Resolve MR ID from args, chat, or current branch; load diffs, commits, CI, approvals, and **all** comments; multi-pass review (architecture, quality, security/perf/tests); severity classification; English report in chat |
| **Cannot do** | Auto-post the review as an MR comment (only on explicit request); implement fixes (use `/merge-request-fix`); proceed without a resolvable MR ID |
| **Guidelines** | Cite `file:line`; avoid duplicating open discussion; Critical first; constructive positives; chat report ≠ implementation |

**Usage examples**

```text
/merge-request-review 42
/merge-request-review !42 focus on SQL injection and auth bypass
/merge-request-review security review for webhook handler
/merge-request-review
```

---

### `/merge-request-fix`

Reads open review comments, plans minimal fixes, applies them after confirmation, and can optionally commit / push / reply on the MR.

| | |
|--|--|
| **Can do** | Load all review threads; triage actionable vs skip/invalid; show fix plan; apply focused code changes; produce requested→fixed summary; optional commit/push/MR reply after a **second** confirmation |
| **Cannot do** | Apply fixes or commit/push/post without confirmation; blindly follow wrong or conflicting feedback; change CI only to force green; force-push |
| **Guidelines** | Review comments are source of truth; minimal scope; validate before applying; every comment appears in the summary with status |

**Usage examples**

```text
/merge-request-fix 42
/merge-request-fix !42 only fix null-check comments, skip architecture feedback
/merge-request-fix address @reviewer's SQL injection comment
/merge-request-fix
```

---

### `/jira-read`

Loads a Jira issue and returns a structured English summary (fields, description, comments, recent changelog).

| | |
|--|--|
| **Can do** | Validate `PROJECT-123` keys; fetch via Jira MCP or REST; summarize overview, custom fields, all comments, recent activity; quote ticket text as-is |
| **Cannot do** | Run without a ticket key; invent a key from chat; download attachments unless asked; print tokens |
| **Guidelines** | Abort immediately on missing/invalid key; do not translate ticket body unless asked; truncate very long fields with a clear note |

**Usage examples**

```text
/jira-read PITOPS-144486
/jira-read pitops-144486
```

---

### `/jira-create`

Creates a new ticket in Jira Service Desk project **PITOPS** (default request type: Service Request Platform).

| | |
|--|--|
| **Can do** | Build summary + description from prompt or chat; collect required fields (request type, environment, platform, component); show draft; create via MCP or Service Desk / REST after confirmation |
| **Cannot do** | Create without confirmation; fill required fields by guessing; put secrets in the ticket; use GitHub/GitLab-only Markdown that Jira won’t render |
| **Guidelines** | Ticket content always English; summary plain text ≤~255 chars; description in one Jira dialect (Markdown / Wiki / ADF); professional tone without agent/meta chatter |

**Usage examples**

```text
/jira-create
/jira-create db01.pd.example.com filesystem usage above 90% on /srv — productive Transporeon storage
/jira-create Platform incident: API gateway 5xx spike since 14:00 UTC
```

---

### `/jira-comment`

Posts a comment on an existing Jira ticket.

| | |
|--|--|
| **Can do** | Require and normalize ticket key; draft comment from prompt or chat; verify ticket exists; post via MCP or REST after confirmation |
| **Cannot do** | Run without a valid `PROJECT-NUMBER` key; post without confirmation; include secrets or Cursor/agent meta commentary |
| **Guidelines** | Comment body always English; Jira-compatible markup only; prefer investigation/status/fix templates |

**Usage examples**

```text
/jira-comment PITOPS-144486
/jira-comment PITOPS-144486 Investigation complete, FS check passed
/jira-comment PITOPS-144486 Root cause identified and fix deployed.
```

---

### `/remote-analyze`

Read-only SSH analysis of a remote system.

| | |
|--|--|
| **Can do** | Analyze `host`, `database` (PostgreSQL), or `barman`; optional focus text; collect metrics/logs/catalogs; structured health report with next steps |
| **Cannot do** | Write files, restart services, install packages, run DDL/DML, trigger Barman backups/`cron`, change `~/.ssh/config` |
| **Guidelines** | Strict allow/deny command lists; evidence-based findings; summarize don’t dump; redact secrets; partial analysis OK if some checks are skipped |

**Usage examples**

```text
/remote-analyze host db01.example.com
/remote-analyze database db01.example.com
/remote-analyze barman barman01.example.com
/remote-analyze database db01.example.com replication lag and long-running queries
/remote-analyze barman barman01.example.com WAL gaps for dbreporting since yesterday
/remote-analyze host app01.example.com high load and OOM in logs since 06:00
```

---

### `/postgres-optimize`

Read-only PostgreSQL performance analysis: workload classification and prioritized tuning recommendations (advisory only).

| | |
|--|--|
| **Can do** | SSH + SELECT-only / file reads; classify read/write/mixed/analytics/standby; recommend `shared_buffers`, WAL, autovacuum, etc. with risk and reload vs restart |
| **Cannot do** | Apply config changes, VACUUM/ANALYZE/REINDEX, terminate backends, reload Postgres, run write benchmarks |
| **Guidelines** | Recommendations must cite metrics; version-aware; host resources + `pg_settings` together; never mutate remote state |

**Usage examples**

```text
/postgres-optimize db01.example.com
/postgres-optimize db01.example.com OLTP write-heavy, WAL disk pressure
/postgres-optimize db01.example.com focus on shared_buffers and checkpoint tuning
```

---

## Typical workflows

```text
# Local change → review → ship
/commit feat(api): add retry jitter to webhook client
/merge-request target develop
/merge-request-review
/merge-request-fix only fix critical and major inline comments

# Ops investigation → ticket trail
/remote-analyze database db01.example.com replication lag
/postgres-optimize db01.example.com checkpoint pressure
/jira-create disk alert investigation on db01 — productive, storage
/jira-comment PITOPS-123456 Posted analysis summary and recommended next steps
/jira-read PITOPS-123456
```

---

## Contributing

1. Keep workflow bodies actionable and concise; prefer tables and checklists.
2. Preserve confirmation gates for any write (git, Jira, MR comments, code fixes).
3. Keep remote workflows strictly read-only.
4. Document new args with usage ✅/❌ examples in the workflow file **and** update this README.
5. Do not commit `~/.cursor/.env`, tokens, or host-specific secrets.
