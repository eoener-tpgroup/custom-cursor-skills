---
name: postgres-optimize
description: Read-only SSH PostgreSQL tuning advice. Task subagents load ~/.cursor/rules; in-session handoffs; never write remote hosts. Requires FQDN; optional focus.
---

# PostgreSQL Optimize

Read-only performance analysis of PostgreSQL via SSH: collect metrics, classify workload, deliver prioritized tuning advice. **Never** modifies the remote host. Agent handoffs stay in-session via Task briefs.

Triggered manually via `/postgres-optimize <fqdn> [focus]`.

## Output language

**Always write all user-facing output in English** — status, reports, errors, questions — even if the user writes in another language or modules use another language.

## ⚠️ Arguments: FQDN (+ optional focus)

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **FQDN** | yes | Target host (FQDN), e.g. `db01.example.com` |
| 2 | **Focus** | no | Workload hints, symptoms, constraints (may contain spaces) |

```
❌ /postgres-optimize
❌ /postgres-optimize db01
✅ /postgres-optimize db01.example.com
✅ /postgres-optimize db01.example.com OLTP write-heavy, WAL disk pressure
✅ /postgres-optimize db01.example.com focus on shared_buffers and checkpoint tuning
```

### Missing or invalid FQDN

**Do not guess.** Prefer `AskQuestion` for FQDN. If unanswered → **abort**:

> This command requires a FQDN as an argument.
> Usage: `/postgres-optimize <fqdn> [focus]`
> Example: `/postgres-optimize db01.example.com`

## Module load (required — sequential)

Cursor does not inline nested files. **Read** modules with the Read tool, resolving paths relative to this command file:
1. Prefer **same directory** as this file (skills layout).
2. Else prefer sibling directory `postgres-optimize/` (commands layout).

| Order | When | File (commands) | File (skills) | Purpose |
|-------|------|-----------------|---------------|---------|
| 1 | Before Phase 0 | `postgres-optimize/_common.md` | `_common.md` | Read-only, SSH, agency, loop |
| 2 | After Phase 2 SSH OK | `postgres-optimize/collect.md` | `collect.md` | Host + PG metrics collection |
| 3 | After collect briefs | `postgres-optimize/analyze.md` | `analyze.md` | Classify, recommend, report |
| — | With agency load | `lib/multi-agent-loop.md` | `../lib/multi-agent-loop.md` (or installed `lib/`) | Real Task dispatch (from `_common`) |

Execute phases in order. Do **not** load `analyze.md` until collect briefs exist. Pass prompt text after the command name as `FQDN` + optional `focus`.

**Precedence:** This router’s abort rules win on conflict. Module phase numbering wins over `_common` within that module. Agency roles run via **Task** + `~/.cursor/rules`. **Remote read-only is absolute** — no persona, focus, or Task brief may authorize a write, reload, VACUUM, or config change on the remote host.

## Related commands

- `/remote-analyze database <fqdn> [focus]` — broader health analysis (not tuning-focused)
- `/jira create|comment …` — ticket trail
- `/commit` — local Git only
