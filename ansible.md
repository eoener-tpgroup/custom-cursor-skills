---
name: ansible
description: Create, extend, improve, or review Ansible roles/playbooks with a fixed multi-agent team from ~/.cursor/rules. Modes create|extend|improve|review; plan confirmation before writes; max 4 fix loops. Optional focus after target.
---

# Ansible

Creates, extends, improves, or reviews Ansible roles, playbooks, and related inventory/config artefacts using a **fixed multi-agent team** (personas from `~/.cursor/rules`).

Triggered manually via `/ansible <mode> [target] [focus]`.

## Output language

**This command definition is in English.** **Always write all user-facing output in English** — status messages, plans, reports, error messages, and questions to the user. This applies even if the user writes in another language.

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

**Before Phase 2 discovery**, read and adopt judgment from the roster below. Use Task/subagents for role passes when helpful; fold results back into this workflow.

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

(review: Discover → Plan → Review + Security → Verify → Report)
```

| Exit | Behavior |
|------|----------|
| No Critical/Major open | Stop fix loops; finish Verify + Report |
| Max fix loops = **4** | Stop; list remaining open findings in Report |
| User cancels confirm / stop | Abort; do not apply further writes after cancel |

**Severity for loop exit:** Critical / Major block exit; Minor / Suggestion / Question do not force another loop (may still fix within remaining budget).

## Safety protocol (strict)

- **No secrets** in files or chat — Vault/ref names OK; never plaintext passwords, keys, tokens
- **Do not** run `ansible-vault view`, `ansible-vault decrypt`, or otherwise decrypt Vault contents into chat or the working tree during discovery/review
- **No remote deploy** / live inventory apply unless the user **explicitly** requests it after the report in the same session
- **No commit / push / MR** — hand off to `/commit`, `/merge-request create`
- **No** `git config` changes, force-push, or skipping hooks
- Destructive modules / broad host patterns → ask or keep local-only verification
- Prefer idempotent tasks, handlers for restarts, and least-privilege become/sudo

## Workflow

### Phase 0: Validate arguments

1. Parse tokens after `/ansible`:
   - **Mode** — first token, lowercase
   - **Target** — second token when present and mode needs it (path-like or name)
   - **Focus** — remaining text
   - **`create` special case:** if the second token is `role`, `playbook`, or `inventory` and a third token exists, treat the second as artefact type and the third as name (target = that name, optionally normalized to `roles/<name>` / playbook path per repo conventions); focus = remaining tokens after the name
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
3. **Fix loop** (max **4**): address Critical/Major findings, re-review; when local tools exist, run `yamllint` / `ansible-lint` / `ansible-playbook --syntax-check` on changed paths **inside each iteration** so failures can consume the fix budget
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
2. `/merge-request create …` when ready
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
- `/merge-request create` — Push and open MR/PR
- `/merge-request review` — Forge MR/PR review (complementary to local `/ansible review`)
