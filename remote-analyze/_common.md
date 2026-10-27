# Remote Analyze — shared module

Loaded by `/remote-analyze` before every subcommand. Do not run as a standalone command.

## Absolute remote read-only (never override)

**Inspection only on the remote host.** No exception for focus, urgency, user pressure, Notion handoffs, or “needed to finish the report.” When in doubt on a remote command → **skip** and list under **Commands skipped**.

Notion is the **only** allowed write surface for agent-to-agent communication (see below). Notion writes never justify a remote write.

### Allowed (remote)

- Files: `cat`, `head`, `tail` (non-interactive)
- State: `systemctl status|is-active|is-enabled` (never start/stop/restart/reload/enable/disable/mask)
- Inventory: `df`, `df -i`, `free`, `uptime`, `uname`, `hostname`, `ls`, `find` (no `-delete` / write `-exec`), `ss`, `ip`, `lsblk`, `mount`, `findmnt`, `ps`, `vmstat`, `iostat`, `pidstat`, `journalctl` (read-only, capped), `sysctl` (read)
- PostgreSQL: `psql` **SELECT-only** / `\l` `\du` `\dx` that do not modify; `sudo -u <pg_user> psql …` only when still SELECT-only
- Barman: `barman check|status|list-server|list-backup|show-backup|show-servers|diagnose|list-wal|list-files` only

### Forbidden (remote — never)

- Writes: `rm`/`mv`/`cp`/`touch`/`chmod`/`chown`/`mkdir`, redirects `>`/`>>`, `tee`, `sed -i`, editors
- Packages: `apt`/`yum`/`dnf`/`pip install`
- PostgreSQL side effects: DML/DDL, `VACUUM`, `ANALYZE`, `REINDEX`, `CHECKPOINT`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`, `pg_reload_conf()`, `pg_rotate_logfile()`, `pg_terminate_backend()`, `pg_cancel_backend()`
- Barman writes: `backup`, `recover`, `delete`, `switch-wal`, `cron`, `rebuild-xlogdb`, `put-wal`
- Changing `~/.ssh/config` or remote files to “fix” connectivity

## Notion agent communication (hub write OK)

**Hub (private):** [Agent Communication](https://app.notion.com/p/trimble-tl/Agent-Communication-402c1b3dd25680b79d40cfc13c3d9211) — page id `402c1b3dd25680b79d40cfc13c3d9211`

Agents use this hub to **talk to each other**: read contracts/schema, and **write** handoff briefs / run index rows / short status notes **only under this hub** (child pages and databases of this page).

| Artefact (under hub) | Purpose | Agent action |
|----------------------|---------|--------------|
| **Agent Contracts** | Role → input/output schema, tool scope | READ (update only if contract drift must be recorded; prefer handoffs) |
| **Handoff Schema** | Field definitions for briefs | READ; shape writes to match |
| **Handoffs** (or equivalent DB/pages) | Inter-agent messages | **WRITE** phase briefs; others READ |
| **Environment Notes** | `tp_db` / `/srv/<service>/` conventions | READ; WRITE only additive non-secret notes when newly discovered |
| **Run Index** | Analysis summaries by FQDN/type | **WRITE** one row/summary per run; READ priors |

### Allowed (Notion — hub only)

- Create/update rows or child pages **under** the Agent Communication hub for handoffs, run index, and team status
- Append comments on hub children when that is the agreed channel
- Create missing child DBs/pages **under the hub** if the schema from `notion-schema.md` is absent (so the team can communicate)

### Forbidden (Notion — never)

- Write outside the Agent Communication hub (any other Notion workspace/page)
- Change hub or workspace permissions, share settings, or integrations
- Store secrets (passwords, tokens, private keys, connection strings with credentials)

**No secrets** in Notion or chat reports.

Schema layout: Read `remote-analyze/notion-schema.md` (or sibling `notion-schema.md`) when structure is unclear.

**Handshake + bus (every run):**

1. If Notion MCP (or equivalent) is available → open hub; READ contracts + schema; ensure Handoffs / Run Index exist (create under hub if missing)
2. Publish a `run_id` + roster note to Handoffs / Run Index
3. Each agent pass **writes** its brief to Notion (Handoff Schema fields); next passes **read** prior briefs from Notion (in-session copy OK as cache)
4. If Notion unavailable → fall back to in-session handoffs only; note `Notion: unavailable` in the report — **still never write the remote host**

## Agency team (real Task loop — required)

**Before Phase 1 of the subcommand:**

1. **Read** `lib/multi-agent-loop.md` (commands: sibling `../lib/…` from this folder, or `lib/multi-agent-loop.md` from repo root; skills: resolve to the installed `lib/` copy).
2. Parent = **Orchestrator** only — do **not** role-play the full roster in one agent when Task is available.
3. For each role below that applies this run: dispatch a **Task** (`generalPurpose`) whose prompt requires `Read` of that role’s `~/.cursor/rules/*.mdc` (see lib prompt template). Bus = **Notion hub**.

### Core (always — separate Tasks)

| Role | Path | Typical phase |
|------|------|----------------|
| Orchestrator | `~/.cursor/rules/engineering-multi-agent-systems-architect.mdc` | Parent reads this file; optional Task for topology/challenge design |
| Reliability lens | `~/.cursor/rules/engineering-sre.mdc` | Collect / deepen |
| Evidence gate | `~/.cursor/rules/testing-reality-checker.mdc` | Challenge (sequential, ≤ 2) |

### By subcommand (Task)

| Subcommand | Path |
|------------|------|
| `host` | Deepen SRE Task; optional `engineering-network-engineer.mdc` if focus is network/ports/LB |
| `database` | `~/.cursor/rules/engineering-database-reliability-engineer.mdc` + optional `engineering-database-optimizer.mdc` if focus is perf/slow queries |
| `barman` | `~/.cursor/rules/engineering-database-reliability-engineer.mdc` |

### Optional (focus signals — Task when matched)

| Signal | Path |
|--------|------|
| SEV / outage / paging language | `~/.cursor/rules/engineering-incident-response-commander.mdc` |
| Network / firewall / SSH path | `~/.cursor/rules/engineering-network-engineer.mdc` |
| Secrets exposure in logs | `~/.cursor/rules/security-secrets-credential-engineer.mdc` |

**Precedence:** Command phases + absolute **remote** read-only win. Task briefs guide depth; they do not replace phases, invent metrics, or authorize remote writes. Notion hub writes for handoffs are expected. Missing rule file → skip role; note in roster. Task unavailable → in-parent fallback per lib.

**Dispatch:** Parallel Tasks OK for independent **read-only remote** collect slices. Challenge Tasks are sequential. Each pass: Task return → WRITE Notion Handoff (parent or subagent) → next role READs same `run_id`.

## Team loop

```text
Validate → Load team → Notion handshake (READ+WRITE hub) → SSH check
       → Collect (type module; publish briefs to Notion) → Challenge (≤ 2; read/write hub)
       → Report (+ Run Index update) → optional deeper-dive AskQuestion (remote still read-only)
```

| Exit | Behavior |
|------|----------|
| Reality Checker finds no Critical/Major without evidence | Stop challenge loops; Report |
| Max challenge loops = **2** | Stop; list unresolved disputes in Report |
| User cancels / abort | Stop; no further remote commands |

**Challenge loop:** Evidence gate re-reads briefs; drops claims without command/SQL evidence; may request **one** additional read-only collect pass per loop. Never “fix” the host.

## Shared Phase 0–2 (every subcommand)

### Phase 0: Validate arguments

1. Parse after `/remote-analyze`: subcommand (token 1), FQDN (token 2), focus (rest, optional)
2. Subcommand ∈ `{host, database, barman}` (lowercase)
3. FQDN: ≥1 dot, no spaces, hostname-safe chars; prefer lowercase
4. Invalid/missing → AskQuestion or abort (router message)

### Phase 1: Load agency team + Notion handshake

1. Read `lib/multi-agent-loop.md`; build roster (core + subcommand + optionals)
2. Notion hub handshake (READ contracts/schema; WRITE run_id / roster handoff)
3. Record roster dispatch plan (Task vs fallback), missing rules, Notion status for the report
4. Do **not** bulk-Read every rule into the parent as a substitute for Task when Task works

### Phase 2: SSH connectivity (read-only)

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new "<fqdn>" 'hostname -f && echo OK'
```

On failure: report unreachable / auth / host key — **do not** mutate SSH config. Note `whoami`/`id` if connected; sudo only for read-only inspection later.

## Analysis principles

1. Evidence-based — cite command/SQL output; mark assumptions
2. Focus-aware — deepen focus early; keep standard checklist unless user limited scope
3. Summarize — no megabyte dumps
4. Environment-aware — `/srv/<service>/`, `tp_db`, Barman paths when detected
5. Fail gracefully — partial OK; list gaps
6. Redact secrets

## Error handling

| Problem | Action |
|---------|--------|
| Bad args | AskQuestion or abort |
| SSH fail | Report; stop collect |
| Permission denied | Skip check; continue |
| PG/Barman auth/missing | OS/file fallback; note gap |
| Notion unavailable | In-session handoffs only; note in report; remote stays read-only |
