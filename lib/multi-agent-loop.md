# Multi-agent Task loop (shared)

**Not a slash-command.** Loaded by agency-using workflows (`/ansible`, `/remote-analyze`, `/postgres-optimize`, `/jira`, `/merge-request`, `/commit`). Do not invent a `/lib` or `/multi-agent-loop` command.

## Purpose

Turn roster personas from `~/.cursor/rules/*.mdc` into a **real** multi-agent loop:

- Parent chat agent = **Orchestrator** (coordinates, confirms with user, synthesizes final report)
- Each roster role = a **separate Task subagent** that reads its rule file and returns a structured brief
- Parent **must not** role-play every persona alone when Task is available

## When to load

After the command resolves its agency roster (core + optional), **Read** this file once per run:

| Layout | Path |
|--------|------|
| Commands | `lib/multi-agent-loop.md` (sibling of top-level `*.md`) |
| Skills | `../lib/multi-agent-loop.md` or copy of `lib/` next to the skill tree |

If this file is missing → fall back to in-parent Read of rule files; note `Multi-agent loop: lib missing (in-parent fallback)` in the report.

## Hard rules

1. **Command phases win** — Task results never override abort rules, AskQuestion gates, remote read-only, Wiki/GFM dialects, or max loop counts.
2. **One role → one Task** — do not merge unrelated roster roles into a single subagent unless the command’s roster marks them as a joint pass (e.g. Reviewer+Security in one review pass is OK when the command says so).
3. **Rule file is mandatory context** — every Task prompt must tell the subagent to `Read` its `~/.cursor/rules/<file>.mdc` **before** doing work and to adopt that persona’s judgment.
4. **Missing rule file** — skip that role; note in roster; continue with remaining roles.
5. **Task unavailable** — if the Task tool cannot be invoked, fall back to in-parent Read + persona adoption for that role; note `Task: unavailable (in-parent fallback)`.
6. **No secrets** in Task prompts, Notion handoffs, or returned briefs.
7. **User-facing English** — parent reports/questions in English; subagent briefs may be English structured text for the parent.

## Cursor Task dispatch

Use the **Task** tool (dynamic `cursor` namespace / native Task):

| Field | Value |
|-------|--------|
| `description` | Short label: `<role> <command-phase>` (3–5 words) |
| `subagent_type` | Default `generalPurpose`. Use `explore` only for pure codebase discovery. Use `security-review` / `bugbot` only when the command explicitly maps a role to those types (otherwise keep `generalPurpose` + rule file). |
| `model` | Omit or `inherit` unless the user named an allowed model |
| `prompt` | Full autonomous brief (see template) — subagents do **not** see prior chat |
| `run_in_background` | `false` for sequential challenge/fix loops; parallel collect passes may use concurrent Task calls in one parent turn |

### Parallelism

- **Allowed:** independent collect / discovery passes with no shared mutable state
- **Forbidden:** parallel writers on the same files without orchestrator merge; parallel Notion writes that race the same `run_id` row without coordination
- Challenge / fix loops are **sequential** (read prior briefs → Task → fold → next)

## Prompt template (every role Task)

Copy and fill:

```text
You are the <ROLE_NAME> agent for command /<COMMAND> (run_id=<RUN_ID>).

## Persona (required)
1. Read the full rule file at: <RULES_PATH>
2. Adopt that agent's identity, critical rules, and quality bar.
3. If the file is missing, return status=skipped_missing_rule and stop.

## Authority
- Follow the command constraints below over the persona if they conflict.
- Do NOT replace command phases, AskQuestion gates, or safety forbid lists.
- Do NOT commit/push/open MR/PR unless the command phase explicitly assigns that to you (normally never).

## Command constraints
<PASTE: remote read-only / Notion hub rules / Wiki Markup / Git safety / max scope — as applicable>

## Inputs
- run_id: <RUN_ID>
- command: <COMMAND>
- phase: <PHASE>
- args: <FQDN / mode / target / MR id / ticket / focus>
- prior_briefs: <summaries or Notion handoff pointers for this run_id>
- workspace / paths in scope: <…>

## Your job this pass
<CONCRETE TASK — collect X, review Y, challenge Z, draft ticket body, etc.>

## Deliverable (return to parent — structured)
Return English structured text with:
1. role: <ROLE_NAME>
2. status: ok | partial | skipped_missing_rule | blocked
3. evidence_refs: commands/SQL/paths/URLs used (no secrets)
4. findings_or_recommendations: list with severity when applicable
5. skipped: what you could not do and why
6. handoff_to: orchestrator | <next_role> | all
7. notes_for_orchestrator: risks, open questions, suggested next collect
```

## Orchestrator (parent) duties

1. Validate args / load command modules
2. Build roster (core + optional signals)
3. Read this lib file
4. Bus handshake (Notion hub **or** in-session — per command)
5. Dispatch Task(s) per team-loop phase
6. Fold briefs → challenge/fix loops per command limits
7. Own all **AskQuestion** / user confirmations (subagents do not confirm with the user)
8. Emit the command’s final report (include Task vs fallback roster notes)

## Bus modes

| Mode | Used by | Behavior |
|------|---------|----------|
| **Notion hub** | `/remote-analyze`, `/postgres-optimize` | READ/WRITE only under Agent Communication hub; each Task pass should WRITE a Handoff (or parent writes the handoff from the Task return); next roles READ same `run_id` |
| **In-session** | `/ansible`, `/jira`, `/merge-request`, `/commit` | Parent keeps briefs in chat/state; pass `prior_briefs` into the next Task prompt; no Notion required |

If Notion is unavailable in Notion-hub mode → in-session briefs + note in report; **never** compensate with remote host writes.

## Team-loop shapes (reference)

Commands define exact loops; this lib only mandates Task dispatch.

```text
# ansible (mutating)
Discover(Task) → Plan(parent) → AskQuestion → Implement(Task) → Review+Security(Task)
  → Fix ≤4 (Task pair) → Verify(parent/tools) → Report(parent)

# ansible (review)
Discover → Review+Security(Task) → Report

# remote-analyze / postgres-optimize
Validate → Roster+Notion → SSH(parent) → Collect(Task, parallel OK)
  → Challenge ≤N (Task Reality Checker [+ Benchmarker]) → Report(parent)

# jira / merge-request (light)
Validate → Primary role Task (optional second Task) → Parent executes phases with brief guidance

# commit (light — single primary Task)
Status/diff(parent) → Branch plan(parent) → Git-workflow Task (optional Jira Task)
  → Parent normalizes message → AskQuestion → stage/commit(parent only)
```

**Light vs full loop:** `/commit` (and similarly light jira/MR create) must not spawn Reviewer/Security/Orchestrator teams — one Primary Task (+ optional) is enough.

## Roster status table (for reports)

| Role | Rule path | Dispatch | Status |
|------|-----------|----------|--------|
| … | `~/.cursor/rules/….mdc` | Task \| in-parent fallback | loaded / skipped (missing) / unavailable |

## Anti-patterns (forbidden)

- “I adopted all personas myself” when Task worked
- Task prompt without the absolute `~/.cursor/rules/…` path
- Subagent inventing lint/SSH/MR results without running tools
- Subagent posting to Jira/MR/Git without parent confirmation gates
- Using Task to bypass remote read-only or to write Notion outside the hub
