---
name: remote-analyze
description: Read-only SSH analysis via host|database|barman. Task subagents load ~/.cursor/rules; Notion Agent Communication hub for handoffs; never write remote hosts.
---

# Remote Analyze

Read-only remote inspection via SSH. **Nothing** on the remote host is modified. Agents may read/write the private Notion **Agent Communication** hub for handoffs only.

Triggered manually via `/remote-analyze <subcommand> <fqdn> [focus]`.

## Output language

**Always write all user-facing output in English** — status, reports, errors, questions — even if the user writes in another language or modules use another language.

## ⚠️ Arguments: subcommand + FQDN (+ optional focus)

| # | Argument | Required | Values | Description |
|---|----------|----------|--------|-------------|
| 1 | **Subcommand** | yes | `host` \| `database` \| `barman` | System type to analyze |
| 2 | **FQDN** | yes | e.g. `db01.example.com` | Target host |
| 3 | **Focus** | no | free text | Symptoms, area, constraints (may contain spaces) |

```
❌ /remote-analyze
❌ /remote-analyze host
❌ /remote-analyze db01.example.com
❌ /remote-analyze vm db01.example.com
✅ /remote-analyze host db01.example.com
✅ /remote-analyze database db01.example.com replication lag
✅ /remote-analyze barman barman01.example.com WAL gaps for dbreporting
```

### Missing or invalid subcommand / FQDN

**Do not guess.** Prefer `AskQuestion` (type: `host` \| `database` \| `barman`; then FQDN). If unanswered → **abort**:

> This command requires a system type and a FQDN.
> Usage: `/remote-analyze <host|database|barman> <fqdn> [focus]`
> Example: `/remote-analyze database db01.example.com replication lag`

## Module load (required — before Phase 1 of the subcommand)

Cursor does not inline nested files. **Read** these modules with the Read tool, resolving paths relative to this command file:
1. Prefer files in the **same directory** as this file: `_common.md`, then `<subcommand>.md` (skills layout).
2. Else prefer the **sibling directory** named like this command: `remote-analyze/_common.md`, `remote-analyze/<subcommand>.md` (commands layout).

| Order | File (commands layout) | File (skills layout) | Purpose |
|-------|------------------------|----------------------|---------|
| 1 | `remote-analyze/_common.md` | `_common.md` | Read-only protocol, SSH, Notion handshake, agency team, loop |
| 2 | `remote-analyze/<subcommand>.md` | `<subcommand>.md` | Type-specific collect + report (`host` / `database` / `barman`) |
| — | `lib/multi-agent-loop.md` | `../lib/multi-agent-loop.md` (or installed `lib/`) | Real Task dispatch (loaded from `_common`) |

**Then** execute only that subcommand’s workflow. Pass remaining prompt text (everything after the subcommand) as its arguments (`FQDN` + optional `focus`).

**Precedence:** This router’s abort rules and AskQuestion gates win on conflict. Subcommand modules win over `_common` for phase numbering and type-specific checks. Agency roles run via **Task** + `~/.cursor/rules` — they do not replace phases. **Remote read-only is absolute** — no persona, focus, or Notion handoff may authorize a write on the remote host. Notion writes are allowed **only** under the Agent Communication hub (see `_common`).

## Related commands

- `/postgres-optimize <fqdn> [focus]` — workload classification + tuning recommendations (also read-only)
- `/jira create|comment …` — ticket trail after analysis
- `/commit` — local Git only
