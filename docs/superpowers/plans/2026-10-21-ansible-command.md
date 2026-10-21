# `/ansible` Command Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Cursor command `/ansible` (`ansible.md`) plus README documentation so agents can create, extend, improve, or review Ansible artefacts with a fixed multi-agent team and confirm-before-write gates.

**Architecture:** One flat Markdown command file (same pattern as `merge-request-review.md` / `commit.md`) that encodes modes, agency roster, phases, fix loops (max 4), safety, and report skeleton. README gains a table row, agency mapping, prerequisites, and usage. No new `.mdc` rule in v1.

**Tech Stack:** Cursor commands (Markdown + YAML frontmatter); agency personas under `~/.cursor/rules/*.mdc`; optional local `ansible-lint` / `yamllint` / `ansible-playbook` for verify phase.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-10-21-ansible-command-design.md` — implement exactly; no scope creep
- Modes: only `create` | `extend` | `improve` | `review`
- Always load fixed core agency team; optional rules by focus signal
- Max fix loops: **4**
- Mutating modes: Plan → `AskQuestion` → write; `review` is report-only
- User-facing output from the command when run: **English**
- Repo scope: generic Ansible; detect `pitops/ansible` like `/commit`
- No remote deploy / AWX / Molecule-against-hosts / new Ansible `.mdc` in v1
- Commits: only when the human explicitly asks (user rule); plan commit steps are optional gates
- Do not invent secrets, tokens, or host-specific credentials in examples

## File map

| File | Responsibility |
|------|----------------|
| `ansible.md` | Full `/ansible` workflow (create) |
| `README.md` | Catalog, agency table, prerequisites, usage, typical workflow (modify) |
| Spec (read-only) | `docs/superpowers/specs/2026-10-21-ansible-command-design.md` |

---

### Task 1: Create `ansible.md`

**Files:**
- Create: `ansible.md`
- Test: structural checklist via `rg` (no unit test framework in this repo)

**Interfaces:**
- Consumes: Spec decisions (modes, roster, phases, max loops=4, safety, report skeleton); `commit.md` pitops/ansible remote detection rules
- Produces: Slash command `/ansible` with frontmatter `name: ansible`

- [ ] **Step 1: Write failing structural check script**

Create a temporary expected-section list and run it against missing `ansible.md` to confirm red:

```bash
bash -c '
REQUIRED=(
  "^name: ansible$"
  "^# Ansible"
  "## Agency agent"
  "engineering-multi-agent-systems-architect.mdc"
  "engineering-devops-automator.mdc"
  "engineering-code-reviewer.mdc"
  "security-cloud-security-architect.mdc"
  "engineering-minimal-change-engineer.mdc"
  "Max fix loops"
  "Fix loops.*4|≤ 4|max.*4"
  "AskQuestion"
  "pitops/ansible"
  "## Output language"
  "Phase 1"
  "Executive Summary"
  "create"
  "extend"
  "improve"
  "review"
)
fail=0
for p in "${REQUIRED[@]}"; do
  if ! rg -q "$p" ansible.md 2>/dev/null; then
    echo "MISSING: $p"
    fail=1
  fi
done
exit $fail
'
```

Expected: FAIL (file missing or patterns missing) with `MISSING:` lines.

- [ ] **Step 2: Create `ansible.md` with full command body**

Write `ansible.md` with **exactly** this content (UTF-8, no trailing junk):

```markdown
---
name: ansible
description: Create, extend, improve, or review Ansible roles/playbooks with a fixed multi-agent team from ~/.cursor/rules. Modes create|extend|improve|review; plan confirmation before writes; max 4 fix loops. Optional focus after target.
---

# Ansible

Creates, extends, improves, or reviews Ansible roles, playbooks, and related inventory/config artefacts using a **fixed multi-agent team** (personas from `~/.cursor/rules`).

Triggered manually via `/ansible <mode> [target] [focus]`.

## Output language

**Always write all user-facing output in English** — status messages, plans, reports, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

## ⚠️ Arguments: mode (+ optional target + focus)

| # | Argument | Required | Values | Description |
|---|----------|----------|--------|-------------|
| 1 | **Mode** | yes | `create` \| `extend` \| `improve` \| `review` | Workflow mode |
| 2 | **Target** | conditional | role/playbook/path/name | Artefact to create or operate on |
| 3 | **Focus** | no | free text | Requirements, constraints, OS, env, security emphasis |

**Target rules:**

- `extend` / `improve` / `review`: target required (path or name). If missing → `AskQuestion` or abort.
- `create`: target optional when focus fully specifies what to create; otherwise ask for name/type (role vs playbook vs other).

**Focus** is everything after target (or after mode when target omitted for `create`) and may contain spaces.

```
❌ /ansible
❌ /ansible refactor nginx
✅ /ansible create role nginx — Rocky 9, TLS behind HAProxy
✅ /ansible extend roles/postgresql add WAL archiving to Barman
✅ /ansible improve playbooks/site.yml idempotency and handlers
✅ /ansible review roles/tp_db focus secrets and privilege escalation
```

### Missing or invalid mode

**Do not guess.** If mode is missing or not in the allowed set:

1. Prefer `AskQuestion` with options: `create`, `extend`, `improve`, `review`
2. If unanswered → **abort** with:

> This command requires a mode.
> Usage: `/ansible <create|extend|improve|review> [target] [focus]`
> Example: `/ansible review roles/nginx focus idempotency`

## Agency agent

**Before Phase 1 discovery work**, read and adopt judgment from the roster below. Use Task/subagents for role passes when helpful; fold results back into this workflow.

### Core (always)

| Role | Path | When |
|------|------|------|
| Orchestrator | `~/.cursor/rules/engineering-multi-agent-systems-architect.mdc` | Always |
| Implementer | `~/.cursor/rules/engineering-devops-automator.mdc` | Always |
| Reviewer | `~/.cursor/rules/engineering-code-reviewer.mdc` | Always |
| Security | `~/.cursor/rules/security-cloud-security-architect.mdc` | Always |
| Diff-Guard | `~/.cursor/rules/engineering-minimal-change-engineer.mdc` | Mutating modes only (`create` / `extend` / `improve`) |

### Optional (when focus/target matches)

| Signal | Path |
|--------|------|
| Secrets, Vault, credentials | `~/.cursor/rules/security-secrets-credential-engineer.mdc` |
| Network, firewall, LB | `~/.cursor/rules/engineering-network-engineer.mdc` |
| Reliability, ops, SLO | `~/.cursor/rules/engineering-sre.mdc` |
| Docs, meta, README | `~/.cursor/rules/engineering-technical-writer.mdc` |
| Explicit appsec / injection / auth focus | `~/.cursor/rules/security-appsec-engineer.mdc` |

**Precedence:** This command’s phases, safety rules, confirm gates, max fix loops, and abort conditions win on conflict. Personas guide quality; they do not replace phases, auto-deploy, commit, or invent findings without reading the repo.

If a listed agency-agent file is missing, continue the command workflow and note that the agency agent could not be loaded.

## Team loop (always)

Fixed pipeline for every run:

```text
Discover → Plan → [AskQuestion if mutating] → Implement
       → Review + Security → Fix (≤ 4 loops) → Verify → Report
```

| Exit | Behavior |
|------|----------|
| No Critical/Major open | Stop fix loops; finish Verify + Report |
| Max fix loops = **4** | Stop; list remaining open findings in Report |
| User cancels confirm / stop | Abort; do not apply further writes after cancel |

**Severity for loop exit:** Critical / Major block exit; Minor / Suggestion / Question do not force another loop (may still fix within remaining budget).

## Safety protocol (strict)

- **No secrets** in files or chat — Vault/ref names OK; never plaintext passwords, keys, tokens
- **No remote deploy** / live inventory apply unless the user **explicitly** requests it after the report in the same session
- **No commit / push / MR** — hand off to `/commit`, `/merge-request`
- **No** `git config` changes, force-push, or skipping hooks
- Destructive modules / broad host patterns → ask or keep local-only verification
- Prefer idempotent tasks, handlers for restarts, and least-privilege become/sudo

## Workflow

### Phase 0: Validate arguments

1. Parse tokens after `/ansible`:
   - **Mode** — first token, lowercase
   - **Target** — second token when present and mode needs it (path-like or name)
   - **Focus** — remaining text
2. Validate mode ∈ `{create, extend, improve, review}`
3. If mode invalid/missing → AskQuestion or abort
4. If target missing for `extend`/`improve`/`review` → AskQuestion or abort
5. For `create` without target: derive intended artefact type/name from focus or ask

### Phase 1: Load agency team

1. Read all **core** rule files (Diff-Guard only for mutating modes)
2. Read **optional** rules matching focus/target signals
3. Record loaded vs skipped (missing) for the final report roster

### Phase 2: Discover repository

Inspect the current workspace (read-only):

| Area | What to find |
|------|----------------|
| Layout | `roles/`, `playbooks/`, `inventories/`, `group_vars/`, `host_vars/`, `collections/`, `ansible.cfg` |
| Quality | Molecule, `ansible-lint` config, CI lint jobs, `.yamllint*` |
| Conventions | Existing role structure, naming, `meta/main.yml`, tags, handler patterns |
| Git | `git remote -v`, current branch |

**`pitops/ansible` detection** (same idea as `/commit`):

After `git remote -v`, treat as Ansible-mode when **any** remote URL matches:

- SSH: `git@gitlab.office.transporeon.com:pitops/ansible.git` (with or without `.git`)
- HTTPS: host `gitlab.office.transporeon.com` and path **ends** with `/pitops/ansible` or `/pitops/ansible.git` (do not match e.g. `pitops/ansible-roles`)

If detected:

- Warn when current branch is protected: `main`, `master`, `development`, `integration`, `acceptance`, `production`
- Note that follow-up `/commit` must use a feature branch from `development` — this command does **not** commit
- Prefer in-repo layout/naming over inventing new top-level trees

### Phase 3: Plan

Produce an English plan covering:

1. Mode + target + focus interpretation
2. Files to add/change (or review scope)
3. Team roster (core + optional)
4. Risks (secrets, handlers, privilege, blast radius)
5. Verification commands available locally
6. Out of scope / non-goals for this run

### Phase 4: Confirm (mutating modes only)

For `create` / `extend` / `improve`:

1. Show the plan (and intended file list)
2. **`AskQuestion`** — proceed / revise / cancel
3. On cancel → abort with no writes
4. On revise → update plan and re-confirm

For `review`: skip confirm; no writes in later phases.

### Phase 5: Execute

#### `create` / `extend` / `improve`

1. **Implementer** applies changes following repo conventions and Diff-Guard (minimal scope)
2. **Reviewer** + **Security** (and optionals) assess the result
3. **Fix loop** (max **4**): address Critical/Major findings, re-review
4. Stop per Team loop exit table

#### `review`

1. No file writes
2. Multi-pass review: architecture/correctness → Ansible quality (idempotency, handlers, vars, tags) → security
3. Classify findings: Critical / Major / Minor / Suggestion / Question
4. Skip fix loops (report findings only) unless the user explicitly asks to switch mode afterward

### Phase 6: Verify

Run when tools exist locally; never invent success:

| Check | Command (examples) |
|-------|---------------------|
| YAML lint | `yamllint <paths>` if configured/available |
| Ansible lint | `ansible-lint <paths>` if available |
| Syntax | `ansible-playbook --syntax-check <playbook>` when a playbook is in scope |

If a tool is missing → note under Verification as skipped. **Do not** SSH to targets or run plays against inventory unless explicitly requested after the report.

### Phase 7: Report

Always emit:

```markdown
# Ansible <mode>: <target>

## Executive Summary
<2-4 sentences: what was done or found; open Critical/Major count>

## Team roster (loaded / skipped)
| Role | Rule | Status |
|------|------|--------|
| ... | ... | loaded / skipped (missing) |

## Plan vs delivered
< Bullets: planned vs actual; note confirm outcome >

## Changes (files)
| Path | Action | Notes |
|------|--------|-------|
| ... | added/modified | ... |

*(For review mode: state N/A — no writes)*

## Findings
### Critical
| # | Location | Problem | Recommendation | Status |
|---|----------|---------|----------------|--------|
| 1 | ... | ... | ... | fixed / open |

### Major
...

### Minor / Suggestion / Question
...

## Verification
| Check | Result |
|-------|--------|
| ansible-lint | pass / fail / skipped (reason) |
| yamllint | ... |
| syntax-check | ... |

## Risks & follow-ups
- ...

## Related next steps
1. `/commit …` (feature branch from `development` if pitops/ansible)
2. `/merge-request …` when ready
3. ...
```

## Mode deliverables

| Mode | Deliverable |
|------|-------------|
| `create` | New artefact matching repo conventions (tasks/handlers/defaults/vars/meta as appropriate) |
| `extend` | Minimal diffs adding the requested capability |
| `improve` | Targeted quality diffs (idempotency, tags, handlers, naming, lint) — no unrelated refactors |
| `review` | Findings-only report |

## Principles

1. **Confirm before mutate** — no writes without AskQuestion approval in mutating modes
2. **Team always** — load core roster every run; optionals by signal
3. **Fix budget 4** — then report remaining opens honestly
4. **Evidence** — cite paths/tasks; no invented lint results
5. **Secrets hygiene** — never embed credentials
6. **Local verify only** by default
7. **English** user-facing output
8. **Command phases win** over persona disagreements

## Error handling

| Problem | Action |
|---------|--------|
| Missing/invalid mode | AskQuestion or abort with usage |
| Missing target (`extend`/`improve`/`review`) | AskQuestion or abort |
| Not an Ansible-looking repo | Ask whether to continue; if yes, use sensible defaults and state assumptions |
| Agency rule file missing | Note skipped; continue |
| User cancels confirm | Abort; no further writes |
| Lint tools missing | Skip verify checks; document |
| Protected branch (pitops/ansible) | Warn; do not commit; suggest feature branch via `/commit` |
| Secrets in proposed content | Block write; ask for Vault/ref approach |
| Ambiguous create scope | Ask artefact type (role/playbook/other) before planning |

## Related commands

- `/commit` — Conventional commit (+ pitops/ansible protected-branch rules)
- `/merge-request` — Push and open MR/PR
- `/merge-request-review` — Forge MR/PR review (complementary to local `/ansible review`)
```

- [ ] **Step 3: Re-run structural check — expect PASS**

Run the same `bash -c` block from Step 1.

Expected: exit code `0`, no `MISSING:` lines.

Also confirm frontmatter:

```bash
bash -c 'head -n 5 ansible.md'
```

Expected: YAML frontmatter with `name: ansible` and a non-empty `description:`.

- [ ] **Step 4: Commit (only if user explicitly requested a commit)**

```bash
git add ansible.md
git commit -m "$(cat <<'EOF'
feat: add /ansible command for role and playbook workflows

EOF
)"
```

If the user has not asked to commit, skip this step and leave changes unstaged/uncommitted.

---

### Task 2: Update `README.md`

**Files:**
- Modify: `README.md`
- Test: `rg` checklist for ansible rows/sections

**Interfaces:**
- Consumes: `/ansible` behavior from Task 1 / spec
- Produces: Documented command in catalog + agency table + prerequisites + command reference + typical workflow

- [ ] **Step 1: Confirm current README anchors**

```bash
bash -c 'rg -n "What’s in this repo|Agency agents|Prerequisites|Command reference|Typical workflows" README.md'
```

Expected: headings exist so edits land in the right sections.

- [ ] **Step 2: Apply README updates**

Make these concrete edits:

**A. Intro paragraph (line ~3)** — extend the summary list to mention Ansible workflows, e.g. append `and Ansible role/playbook create–review loops`.

**B. What’s in this repo table** — add row after `postgres-optimize` (or in logical place):

```markdown
| `ansible` | [`ansible.md`](ansible.md) | Create, extend, improve, or review Ansible roles/playbooks with a multi-agent team |
```

**C. Prerequisites table** — add:

```markdown
| Ansible workflows | Local Ansible content in the workspace; agency rules under `~/.cursor/rules/`; optional `ansible-lint` / `yamllint` / `ansible-playbook` |
```

**D. Agency agents table** — add:

```markdown
| `/ansible` | `engineering-multi-agent-systems-architect.mdc` + `engineering-devops-automator.mdc` + `engineering-code-reviewer.mdc` + `security-cloud-security-architect.mdc` (+ `engineering-minimal-change-engineer.mdc` for writes) | Optional: secrets / network / SRE / technical-writer / appsec rules by focus |
```

**E. Shared guidelines** — ensure guideline #8 (agency agents) still applies; no change required unless text claims only Jira/MR commands use agency agents — if so, broaden to include `/ansible`.

**F. Command reference** — add a full `### `/ansible`` section (English, same style as other commands) covering Can do / Cannot do / Guidelines / Usage examples matching Task 1.

**G. Typical workflows** — add an example block:

```text
# Ansible role work
/ansible create role nginx — Rocky 9, TLS behind HAProxy
/ansible improve roles/nginx idempotency and handlers
/ansible review roles/nginx focus secrets
/commit feat(nginx): add TLS termination role
/merge-request
```

- [ ] **Step 3: Verify README mentions**

```bash
bash -c '
rg -n "`ansible`|ansible.md|/ansible" README.md
test -f ansible.md
'
```

Expected: multiple hits including table row, agency mapping, command reference heading, and `ansible.md` exists.

- [ ] **Step 4: Commit (only if user explicitly requested a commit)**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
docs: document /ansible command in README

EOF
)"
```

If the user has not asked to commit, skip this step.

---

## Spec coverage checklist (self-review)

| Spec requirement | Task |
|------------------|------|
| Modes create/extend/improve/review | Task 1 |
| Generic + pitops/ansible detection | Task 1 Phase 2 |
| Always-on fixed team | Task 1 Agency + Team loop |
| Optional domain agents | Task 1 Optional table |
| Plan → AskQuestion → write | Task 1 Phase 4 |
| Max fix loops 4 | Task 1 Team loop |
| Review report-only | Task 1 Phase 5 `review` |
| English output | Task 1 Output language |
| Report skeleton | Task 1 Phase 7 |
| Safety / no deploy / no commit | Task 1 Safety |
| README integration | Task 2 |
| No new `.mdc` in v1 | Honored (no task creates one) |

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-10-21-ansible-command.md`. Two execution options:

**1. Subagent-Driven (recommended)** — fresh subagent per task, review between tasks, fast iteration  

**2. Inline Execution** — execute tasks in this session using executing-plans, batch execution with checkpoints  

Which approach?
