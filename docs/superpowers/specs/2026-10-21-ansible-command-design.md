# Design: `/ansible` Cursor Command

**Date:** 2026-10-21  
**Status:** Approved / Implemented  
**Repo:** `custom-cursor-skills` / `~/.cursor/commands`

## Goal

Add a Cursor command `/ansible` that creates, extends, improves, or reviews Ansible roles, playbooks, and related inventory/config artefacts. Every run uses a fixed multi-agent team (personas from `~/.cursor/rules/*.mdc`) in an implement → review → fix loop. Mutating modes require an explicit plan confirmation before writes.

## Decisions (locked)

| Topic | Choice |
|-------|--------|
| Invocation | Single command with explicit mode: `/ansible <mode> …` |
| Repo scope | Generic Ansible repos; detect `pitops/ansible` for extra conventions |
| Agent team | Always on — fixed pipeline (Approach 1) |
| Writes | Plan → `AskQuestion` → write (`create` / `extend` / `improve`); `review` report-only |
| Max fix loops | **4** |
| User-facing language | English (repo convention) |
| Out of scope (v1) | Remote deploy, AWX/Tower triggers, Molecule against real hosts, new dedicated Ansible `.mdc` |

## Command surface

**File:** `ansible.md` → slash command `/ansible`

```text
/ansible <mode> [target] [focus…]
```

| Argument | Required | Values / meaning |
|----------|----------|------------------|
| **mode** | yes | `create` \| `extend` \| `improve` \| `review` |
| **target** | conditional | Role / playbook / path / name. For `create`, optional if focus is enough; otherwise `AskQuestion` |
| **focus** | no | Free text: requirements, constraints, OS, env, security emphasis |

### Modes

| Mode | Purpose | Writes? |
|------|---------|---------|
| `create` | New role, playbook, inventory fragment, or similar | Yes (after confirm) |
| `extend` | Add capabilities to existing artefacts | Yes (after confirm) |
| `improve` | Refactor for quality (idempotency, structure, lint) without scope creep | Yes (after confirm) |
| `review` | Structured review report | No |

### Usage examples

```text
✅ /ansible create role nginx — Rocky 9, TLS termination behind HAProxy
✅ /ansible extend roles/postgresql add WAL archiving to Barman
✅ /ansible improve playbooks/site.yml idempotency and handler notifications
✅ /ansible review roles/tp_db focus secrets and privilege escalation
❌ /ansible                              → missing mode
❌ /ansible refactor nginx               → invalid mode
```

### Missing / invalid mode

1. Prefer `AskQuestion` with options `create`, `extend`, `improve`, `review`
2. If unanswered → abort with usage message

## Agency team (always)

**Before discovery**, Read and adopt judgment from core rules. Missing file → continue and note in report.

### Core (always)

| Role | Path | Duty |
|------|------|------|
| Orchestrator | `~/.cursor/rules/engineering-multi-agent-systems-architect.mdc` | Pipeline, contracts, exit criteria |
| Implementer | `~/.cursor/rules/engineering-devops-automator.mdc` | Ansible design & implementation |
| Reviewer | `~/.cursor/rules/engineering-code-reviewer.mdc` | Correctness, maintainability, Ansible practice |
| Security | `~/.cursor/rules/security-cloud-security-architect.mdc` | Secrets, privilege, network, hardening |
| Diff-Guard | `~/.cursor/rules/engineering-minimal-change-engineer.mdc` | Writes only — keep diffs minimal |

### Optional (when focus/target matches)

| Signal | Path |
|--------|------|
| Secrets / Vault / credentials | `~/.cursor/rules/security-secrets-credential-engineer.mdc` |
| Network / firewall / LB | `~/.cursor/rules/engineering-network-engineer.mdc` |
| Reliability / ops | `~/.cursor/rules/engineering-sre.mdc` |
| Docs / meta / README | `~/.cursor/rules/engineering-technical-writer.mdc` |
| Explicit appsec review focus | `~/.cursor/rules/security-appsec-engineer.mdc` |

### Precedence

Command phases, safety rules, and `AskQuestion` gates **win** over persona guidance. Personas guide quality; they do not replace phases or invent remote deploys.

### Loop topology (fixed)

```text
Discover → Plan → [AskQuestion if mutating] → Implement
       → Review + Security → Fix (≤ 4) → Verify → Report
```

| Exit condition | Behavior |
|----------------|----------|
| No Critical/Major open | Stop loops; report |
| Max fix loops = 4 | Stop; list remaining open findings |
| User cancels confirm / stop | Abort cleanly; no partial surprise writes after cancel |

Orchestrator may use Task/subagents for role passes; results fold back into the main workflow.

## Workflow phases

1. **Validate** — Parse mode / target / focus; validate mode ∈ {create, extend, improve, review}
2. **Agency load** — Read core + matching optional rules
3. **Discover** — Inspect repo: `roles/`, `playbooks/`, `group_vars/`, `host_vars/`, `ansible.cfg`, collections, Molecule, CI lint config; detect `pitops/ansible` via remotes (same URL rules as `/commit`)
4. **Plan** — English plan: scope, files, risks, roster, verification steps
5. **Confirm** — Mutating modes: `AskQuestion` with visible plan; `review` skips
6. **Execute** — Implementer applies changes; Reviewer + Security pass; fix loop ≤ 4
7. **Verify** — If available locally: `ansible-lint`, `yamllint`, `ansible-playbook --syntax-check` (no remote apply without explicit user request)
8. **Report** — Structured English report (skeleton below)

### `pitops/ansible` extras

When Ansible mode is detected (remote matches `pitops/ansible` as in `/commit`):

- Warn if worktree is on protected branch: `main`, `master`, `development`, `integration`, `acceptance`, `production`
- Prefer feature-branch workflow from `development` for follow-up `/commit` (this command does **not** commit)
- Prefer existing role/playbook layout and naming conventions found in-repo over inventing new top-level structures

## Safety

- Never print or commit secrets; Vault variable references OK; no plaintext passwords/keys in files or chat
- No remote deploy / production run unless the user explicitly requests it in the same session after the report
- No git commit / push / MR — hand off to `/commit`, `/merge-request`
- Do not skip hooks or force-push
- When in doubt on a destructive Ansible module or host targeting → ask or keep dry-run / local-only verification

## Report skeleton

```markdown
# Ansible <mode>: <target>

## Executive Summary
## Team roster (loaded / skipped)
## Plan vs delivered
## Changes (files)          # N/A for review
## Findings                 # Critical → Suggestion; note open after loops
## Verification             # lint / syntax / skipped + why
## Risks & follow-ups
## Related next steps       # /commit, /merge-request, …
```

## Deliverables by mode

| Mode | Deliverable |
|------|-------------|
| `create` | New artefact structure matching repo conventions (tasks, handlers, defaults, vars, meta as appropriate) |
| `extend` | Minimal diffs adding requested capability |
| `improve` | Targeted quality diffs (idempotency, tags, handlers, naming, lint) |
| `review` | Findings-only report |

## Repo integration

| Change | Detail |
|--------|--------|
| Add | `ansible.md` — full command workflow |
| Update | `README.md` — table row, agency mapping, prerequisites, usage, typical workflow |

**Prerequisites (soft):** Local git repo with Ansible content; optional `ansible-lint` / `yamllint` / `ansible-playbook` for verification. Agency rules under `~/.cursor/rules/`.

## Related commands

- `/commit` — Conventional commit (+ ansible protected-branch rules)
- `/merge-request` — Push and open MR/PR
- `/merge-request-review` — Forge MR/PR review (complementary to `/ansible review` of local artefacts)

## Implementation notes (for plan)

- Mirror style of `merge-request-review.md` / `commit.md`: YAML frontmatter (`name`, `description`), phases, tables, English output rule, agency section, error handling table, related commands
- Keep body actionable; prefer checklists over prose
- README agency table must include `/ansible` core + optional mapping
- Do not invent a new Ansible-specific `.mdc` in v1

## Non-goals (explicit)

- Replacing Molecule CI or production AWX workflows
- Auto-committing Ansible changes
- Running playbooks against live inventory by default
