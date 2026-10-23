# Custom Cursor Skills & Commands

Reusable agent workflows for day-to-day engineering work: Git commits and merge requests, Jira tickets, read-only remote host / PostgreSQL analysis, and Ansible role/playbook create–review loops.

Each top-level `*.md` file (except `README.md`) is one slash-command (for example `commit.md` → `/commit`). Some commands are **routers** with a sibling directory of modules the agent **Reads** at runtime (for example `jira.md` + `jira/{_common,read,create,comment}.md`). Place them as **CLI commands**, **IDE skills**, or both.

---

## What’s in this repo

| Name | File / modules | Purpose |
|------|----------------|---------|
| `commit` | [`commit.md`](commit.md) | Conventional Commits (no emojis) + optional feature branch; ansible: never commit on main/master/env branches |
| `merge-request` | [`merge-request.md`](merge-request.md) + [`merge-request/`](merge-request/) | Subcommands `create` \| `review` \| `fix` (GitHub PR / GitLab MR) |
| `jira` | [`jira.md`](jira.md) + [`jira/`](jira/) | Subcommands `read` \| `create` \| `comment` (PITOPS Service Desk) |
| `remote-analyze` | [`remote-analyze.md`](remote-analyze.md) | Read-only SSH analysis (host / PostgreSQL / Barman) |
| `postgres-optimize` | [`postgres-optimize.md`](postgres-optimize.md) | Read-only PostgreSQL performance tuning advice |
| `ansible` | [`ansible.md`](ansible.md) | Create, extend, improve, or review Ansible roles/playbooks with a multi-agent team |

---

## Place as Cursor CLI commands

CLI slash-commands are discovered from **top-level** Markdown files only:

```text
~/.cursor/commands/<name>.md
~/.cursor/commands/<name>/          # optional modules Read by the router (not slash-commands themselves)
```

Invoked as `/<name> …` (for example `/commit`, `/jira read PITOPS-123`, `/merge-request review !42`).

**Important:** When installing by copy or symlink, always include sibling module directories (`jira/`, `merge-request/`). Routers tell the agent to Read those files; without them the subcommand workflows are incomplete.

### Clone, then copy

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/commands
# Top-level command files
cp ~/src/custom-cursor-skills/*.md ~/.cursor/commands/
# Module directories for routers
cp -a ~/src/custom-cursor-skills/jira ~/src/custom-cursor-skills/merge-request ~/.cursor/commands/
# Optional: keep README out of the commands folder
rm -f ~/.cursor/commands/README.md
# Migration: remove legacy flat commands superseded by routers
rm -f ~/.cursor/commands/jira-{read,create,comment}.md \
      ~/.cursor/commands/merge-request-{review,fix}.md
```

### Clone, then symlink (easy updates via `git pull`)

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/commands
for f in ~/src/custom-cursor-skills/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  ln -sfn "$f" ~/.cursor/commands/"$(basename "$f")"
done
# Module directories (routers Read paths under ~/.cursor/commands/<name>/)
ln -sfn ~/src/custom-cursor-skills/jira ~/.cursor/commands/jira
ln -sfn ~/src/custom-cursor-skills/merge-request ~/.cursor/commands/merge-request
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

Each top-level command file becomes one skill folder (`SKILL.md`). For routers, also copy the sibling module directory next to `SKILL.md` (or keep modules under `~/.cursor/commands/<name>/` and ensure the skill text still points the agent at those paths):

```bash
git clone git@github.com:eoener-tpgroup/custom-cursor-skills.git ~/src/custom-cursor-skills
mkdir -p ~/.cursor/skills

for f in ~/src/custom-cursor-skills/*.md; do
  name="$(basename "$f" .md)"
  [ "$name" = "README" ] && continue
  mkdir -p ~/.cursor/skills/"$name"
  cp "$f" ~/.cursor/skills/"$name"/SKILL.md
  # Router modules (if present): keep next to SKILL.md under the same name
  if [ -d ~/src/custom-cursor-skills/"$name" ]; then
    cp -a ~/src/custom-cursor-skills/"$name"/. ~/.cursor/skills/"$name"/
  fi
done
```

**Note:** Routers resolve modules relative to the command file: same-directory `_common.md` / `<subcommand>.md` (this skills layout), else sibling `jira/` or `merge-request/` (CLI commands layout). Both installs work after the copy above.

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
  if [ -d "$REPO/$name" ]; then
    cp -a "$REPO/$name"/. .cursor/skills/"$name"/
  fi
done
```

Commit `.cursor/skills/` with the project if the whole team should get them.

### Using skills in the IDE

After placement, invoke by name in Agent chat, or with the same slash form as the CLI:

```text
/commit
/merge-request review !42 focus on auth
/jira read PITOPS-144486
/remote-analyze database db01.example.com replication lag

Use the commit skill for these changes
Follow merge-request review on MR !42, focus on auth
Run remote-analyze as a skill: database db01.example.com replication lag
```

---

## CLI vs IDE (same content, different location)

| | Cursor CLI **commands** | Cursor IDE **skills** |
|--|-------------------------|------------------------|
| Location | `~/.cursor/commands/<name>.md` (+ optional `<name>/` modules) | `~/.cursor/skills/<name>/SKILL.md` or `.cursor/skills/<name>/SKILL.md` |
| Layout | One top-level `.md` (+ optional module dir) | One folder + `SKILL.md` (+ optional module files) |
| Invoke | `/commit`, `/jira read …`, `/merge-request create …` | Slash commands in Agent chat (`/commit …`) or ask by skill name |
| Same file? | Yes — copy or symlink the repo `.md` as the command file or as `SKILL.md` | Yes |

You can install **both**: keep CLI copies under `~/.cursor/commands/` and skill copies under `~/.cursor/skills/`.

---

## Prerequisites (by family)

| Area | Needs |
|------|--------|
| Git / MR workflows | Local git repo; `gh` (GitHub) and/or `glab` / GitLab MCP (GitLab) |
| Jira workflows | Jira MCP **or** `JIRA_URL` + `JIRA_TOKEN` in `~/.cursor/.env` |
| Remote / Postgres workflows | Working SSH access to the target host (keys / agent) |
| Ansible workflows | Local Ansible content in the workspace; agency rules under `~/.cursor/rules/`; optional `ansible-lint` / `yamllint` / `ansible-playbook` |

---

## Shared guidelines (all commands)

These rules appear across the command definitions and should be treated as hard constraints:

1. **User-facing output is English** — status, reports, drafts, and questions — even if the user writes in another language.
2. **Ask before mutating** — commits, pushes, MR/PR creation, Jira writes, and applying review fixes require an explicit confirmation (`AskQuestion`) with a visible draft/plan.
3. **Do not guess required IDs** — missing MR IDs, ticket keys, FQDNs, or system types → ask or abort with the documented usage message.
4. **No secrets** — never print tokens, commit credentials, or put passwords/API keys into tickets or MR comments.
5. **Platform-correct markup** — GitHub GFM vs GitLab Markdown for MR/PR text; **Jira Wiki Markup** for Jira create/comment — never Markdown/GFM/ADF as the authoring dialect for those commands. For `/merge-request create|review|fix`, the forge comes from the **origin URL** (not from which CLI happens to be installed).
6. **Git safety** — no `git config` changes, no force-push to `main`/`master`, no skipping hooks, no destructive git without an explicit user request.
7. **Remote analysis is read-only** — `/remote-analyze` and `/postgres-optimize` must never write, restart services, or run DDL/DML on the remote host.
8. **Agency agents** — Jira, MR, and `/ansible` commands instruct the agent to **read** matching personas from `~/.cursor/rules/*.mdc` before the workflow (see below). Command phases, safety, and AskQuestion gates always win on conflict.
9. **Router modules** — `/jira` and `/merge-request` require a subcommand; the agent must **Read** `_common.md` plus the matching subcommand module before executing phases. Do not invent missing modules.

---

## Agency agents (`~/.cursor/rules`)

These workflows expect agency-agent style rules installed under `~/.cursor/rules/` (agent-requestable `.mdc` files, typically `alwaysApply: false`). At the start of a run the command tells the agent to **Read** the primary (and optional) rule and adopt its judgment — not to replace the command procedure.

| Command | Primary rule | Optional |
|---------|--------------|----------|
| `/jira` (`read` \| `create` \| `comment`) | `project-management-jira-workflow-steward.mdc` | — |
| `/merge-request create` | `engineering-git-workflow-master.mdc` | `project-management-jira-workflow-steward.mdc` when a ticket key is in branch/commits/context |
| `/merge-request review` | `engineering-code-reviewer.mdc` | `security-appsec-engineer.mdc` when Focus is security-related |
| `/merge-request fix` | `engineering-minimal-change-engineer.mdc` | — |
| `/ansible` | `engineering-multi-agent-systems-architect.mdc` + `engineering-devops-automator.mdc` + `engineering-code-reviewer.mdc` + `security-cloud-security-architect.mdc` (+ `engineering-minimal-change-engineer.mdc` for writes) | Optional: secrets / network / SRE / technical-writer / appsec rules by focus |

If a listed file is missing, continue the command workflow and note that the agency agent could not be loaded.

---

## Command reference

### `/commit`

Creates a Conventional Commits commit. On the default branch it can create a feature branch first. For `pitops/ansible` it never commits on `main` / `master` / `development` / `integration` / `acceptance` / `production` — feature branches are created from `development`.

| | |
|--|--|
| **Can do** | Inspect status/diff/log; plan files to stage; draft Conventional Commit message; create `type/short-description` branch; stage and commit after confirmation |
| **Cannot do** | Commit without confirmation; amend unless user-rule conditions are met; skip hooks; commit secrets (`.env`, credentials, tokens); invent a message when diff + context are insufficient; put emojis/Gitmoji in commit messages; commit on protected ansible branches (even if the user asks) |
| **Guidelines** | Prefer one logical change per commit; subject imperative, ≤72 chars, no emojis; body explains *why*; issue refs in footer; Conventional Commits format is mandatory even if project history differs; ask when instructions conflict with the diff; ansible: AskQuestion when on main/master/integration/acceptance/production |

**Usage examples**

```text
/commit
/commit fix(api): handle null response in webhook retry
/commit only stage auth files, type fix scope auth
/commit refs PROJ-789 — branch fix/webhook-race
```

---

### `/merge-request`

Router with required subcommand. Modules: [`merge-request/_common.md`](merge-request/_common.md), [`create.md`](merge-request/create.md), [`review.md`](merge-request/review.md), [`fix.md`](merge-request/fix.md).

| Subcommand | Purpose |
|------------|---------|
| `create` | Push branch and create GitHub PR / GitLab MR |
| `review` | Intensive structured code review (incl. existing threads) |
| `fix` | Apply actionable review feedback locally |

| | |
|--|--|
| **Can do** | `create`: detect forge from origin URL; draft title + Summary / Changes / Test plan; push + create via `gh`/`glab`/MCP. `review`: resolve MR ID; multi-pass review; English report; optional inline comments + GitHub `suggestion` fences. `fix`: triage threads; apply minimal fixes after confirmation; optional commit/push/reply (+ resolve GitHub threads for ✅ Fixed) |
| **Cannot do** | Run without a valid subcommand; push/create/fix without confirmation; force-push; invent fake test steps; use Jira markup in MR/PR bodies; auto-post reviews (only on explicit request); invent an MR ID |
| **Guidelines** | Subcommand required (`AskQuestion` or abort). Always **Read** `_common` + subcommand module first. Origin URL decides the forge. Title plain text; body platform Markdown. Agency: git-workflow (`create`), code-reviewer (`review`), minimal-change (`fix`). |

**Usage examples**

```text
/merge-request create
/merge-request create feat(auth): add JWT refresh — target develop
/merge-request review !42 focus on SQL injection and auth bypass
/merge-request fix 42 only fix null-check comments, skip architecture feedback
```

---

### `/jira`

Router with required subcommand. Modules: [`jira/_common.md`](jira/_common.md), [`read.md`](jira/read.md), [`create.md`](jira/create.md), [`comment.md`](jira/comment.md).

| Subcommand | Purpose |
|------------|---------|
| `read` | Load and summarize a ticket |
| `create` | Create a PITOPS Service Desk ticket |
| `comment` | Post a comment on an existing ticket |

| | |
|--|--|
| **Can do** | `read`: validate key; fetch via MCP/REST; structured summary. `create`: draft from prompt/chat; collect required fields; create after confirmation. `comment`: draft from prompt/chat; verify ticket; post after confirmation |
| **Cannot do** | Run without a valid subcommand; invent ticket keys; create/comment without confirmation; put secrets in tickets; use Markdown/GFM instead of Wiki Markup for writes |
| **Guidelines** | Subcommand required. Always **Read** `_common` + subcommand module first. Ticket content English; writes always **Jira Wiki Markup**. Load `project-management-jira-workflow-steward` before Phase 1. |

**Usage examples**

```text
/jira read PITOPS-144486
/jira create db01.pd.example.com filesystem usage above 90% on /srv — productive Transporeon storage
/jira comment PITOPS-144486 Investigation complete, FS check passed
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

### `/ansible`

Creates, extends, improves, or reviews Ansible roles, playbooks, and related inventory/config using a **fixed multi-agent team** (personas from `~/.cursor/rules`). Modes: `create`, `extend`, `improve`, `review`.

| | |
|--|--|
| **Can do** | Discover repo layout, conventions, and optional `pitops/ansible` context; load core + optional agency personas; **all user-facing output in English** (status, plans, reports, errors, questions) including an English plan with **AskQuestion** before writes (mutating modes); implement or extend roles/playbooks with minimal scope; multi-pass review + security; fix loop (≤ **4**) for Critical/Major; local `yamllint` / `ansible-lint` / `ansible-playbook --syntax-check` when available; structured report with verification and follow-ups |
| **Cannot do** | Commit, push, or open MR/PR (use `/commit`, `/merge-request create`); deploy or run plays against live inventory without an explicit post-report request; embed secrets (Vault/ref names OK); apply writes without confirmation in `create` / `extend` / `improve`; invent lint or syntax-check results; replace command phases with persona-only improvisation |
| **Guidelines** | **English only** for all user-facing output (status, plans, reports, errors, questions), even if the user writes in another language; target required for `extend`, `improve`, and `review`; `review` is findings-only (no file writes, no fix loop unless user switches mode); max **4** fix loops then report open Critical/Major; warn on protected branches in `pitops/ansible` and defer commits to `/commit` on a feature branch from `development`. Load orchestrator, implementer, reviewer, and security before Phase 2 discovery; load minimal-change engineer for mutating modes; add optional rules (secrets, network, SRE, docs, appsec) when focus/target signals match. |

**Usage examples**

```text
/ansible create role nginx — Rocky 9, TLS behind HAProxy
/ansible extend roles/postgresql add WAL archiving to Barman
/ansible improve playbooks/site.yml idempotency and handlers
/ansible review roles/tp_db focus secrets and privilege escalation
```

---

## Typical workflows

```text
# Local change → review → ship
/commit feat(api): add retry jitter to webhook client
/merge-request create target develop
/merge-request review
/merge-request fix only fix critical and major inline comments

# Ops investigation → ticket trail
/remote-analyze database db01.example.com replication lag
/postgres-optimize db01.example.com checkpoint pressure
/jira create disk alert investigation on db01 — productive, storage
/jira comment PITOPS-123456 Posted analysis summary and recommended next steps
/jira read PITOPS-123456

# Ansible role work
/ansible create role nginx — Rocky 9, TLS behind HAProxy
/ansible improve roles/nginx idempotency and handlers
/ansible review roles/nginx focus secrets
/commit feat(nginx): add TLS termination role
/merge-request create
```

---

## Contributing

1. Keep workflow bodies actionable and concise; prefer tables and checklists.
2. Preserve confirmation gates for any write (git, Jira, MR comments, code fixes).
3. Keep remote workflows strictly read-only.
4. Document new args with usage ✅/❌ examples in the workflow file **and** update this README.
5. Do not commit `~/.cursor/.env`, tokens, or host-specific secrets.
6. For routers (`jira`, `merge-request`): keep the top-level file thin; put shared rules in `_common.md` and subcommand phases in named modules; never register module `.md` files as top-level slash-commands.
