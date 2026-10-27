# PostgreSQL Optimize — shared module

Loaded by `/postgres-optimize` before collect/analyze. Do not run as a standalone command.

## Absolute remote read-only (never override)

**Inspection + advisory recommendations only on the remote host.** Never apply settings, reload, restart, VACUUM/ANALYZE/REINDEX, or terminate backends — not even to “complete” the run or because a Notion handoff asked for it.

When in doubt on a remote command → **skip**; list under **Commands skipped**.

Notion is the **only** allowed write surface for agent-to-agent communication (same hub rules as `/remote-analyze`).

### Allowed (remote)

- Files: `cat`, `head`, `tail` (non-interactive)
- State: `systemctl status|is-active|is-enabled` only
- Inventory: `df`, `df -i`, `free`, `uptime`, `uname`, `hostname`, `ls`, `find` (no delete/write exec), `ss`, `ip`, `lsblk`, `mount`, `findmnt`, `ps`, `vmstat`, `iostat`, `pidstat`, `sysctl` (read), `journalctl` (capped)
- PostgreSQL: SELECT-only / `\l` `\du` `\dx` non-mutating; read `postgresql.conf`, `postgresql.auto.conf`, `pg_hba.conf` via `cat`; `sudo -u <pg_user> psql …` only when SELECT-only
- WAL size: `du -sh` on pg_wal path (read)

### Forbidden (remote — never)

- Writes / redirects / `tee` / `sed -i` / editors / package installs
- `systemctl start|stop|restart|reload|enable|disable|mask`
- DML/DDL; `VACUUM`, `ANALYZE`, `REINDEX`, `CHECKPOINT`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`
- `pg_reload_conf()`, `pg_rotate_logfile()`, `pg_terminate_backend()`, `pg_cancel_backend()`
- Write benchmarks: `pgbench` (writes), `fio` write tests, stress tools
- Mutating `~/.ssh/config` or remote files

## Notion agent communication (hub write OK)

**Hub:** [Agent Communication](https://app.notion.com/p/trimble-tl/Agent-Communication-402c1b3dd25680b79d40cfc13c3d9211) — id `402c1b3dd25680b79d40cfc13c3d9211`

Shared with `/remote-analyze`. Agents **READ** contracts/schema and **WRITE** handoffs / run index / status **only under this hub**.

| Artefact | Purpose | Action |
|----------|---------|--------|
| **Agent Contracts** | Role I/O + tool scope | READ (prefer not to churn) |
| **Handoff Schema** | Field definitions | READ; shape writes to match |
| **Handoffs** | Inter-agent briefs | **WRITE** per phase; next roles READ |
| **Environment Notes** | `tp_db`, `/srv/<service>/` | READ; additive non-secret notes OK |
| **Run Index** | Optimize/analyze summaries | **WRITE** per run; READ priors |

### Allowed (Notion — hub only)

- Create/update handoff rows/pages, Run Index entries, team status under the hub
- Create missing child DBs/pages under the hub per `notion-schema.md` if absent

### Forbidden (Notion — never)

- Write outside the Agent Communication hub
- Change permissions / sharing
- Store secrets

Schema: Read `../remote-analyze/notion-schema.md` or `remote-analyze/notion-schema.md` when unclear.

**Handshake + bus:** Notion available → READ contracts/schema; WRITE `run_id` + roster; each collect/classify/recommend pass publishes a brief; challengers READ prior briefs. Unavailable → in-session only + note in report; **remote stays read-only**.

## Agency team (real Task loop — required)

**Before Phase 0:**

1. **Read** `lib/multi-agent-loop.md` (commands: `lib/multi-agent-loop.md` from repo root / `../lib/…` from this folder; skills: installed `lib/` copy).
2. Parent = **Orchestrator** — do **not** role-play the full roster in one agent when Task is available.
3. Dispatch a **Task** (`generalPurpose`) per role below; each prompt requires `Read` of that role’s rule file. Bus = **Notion hub**.

### Core (always — separate Tasks)

| Role | Path | Typical phase |
|------|------|----------------|
| Orchestrator | `~/.cursor/rules/engineering-multi-agent-systems-architect.mdc` | Parent (+ optional Task for loop design) |
| Host / capacity | `~/.cursor/rules/engineering-sre.mdc` | Collect |
| DB operations context | `~/.cursor/rules/engineering-database-reliability-engineer.mdc` | Collect / context |
| Tuning lead | `~/.cursor/rules/engineering-database-optimizer.mdc` | Classify + recommend |
| Metrics / classification | `~/.cursor/rules/testing-performance-benchmarker.mdc` | Classify + challenge |
| Evidence gate | `~/.cursor/rules/testing-reality-checker.mdc` | Challenge (sequential, ≤ 3) |

### Optional (focus signals — Task when matched)

| Signal | Path |
|--------|------|
| Network / replication path | `~/.cursor/rules/engineering-network-engineer.mdc` |
| Outage / SEV language | `~/.cursor/rules/engineering-incident-response-commander.mdc` |

**Precedence:** Phases + absolute **remote** read-only win. Task briefs do not apply remote changes or invent metrics. Notion hub writes for handoffs are expected. Missing file → skip; note in roster. Task unavailable → in-parent fallback per lib.

**Dispatch:** Parallel Tasks OK for independent **read-only remote** collect. Challenge = Reality Checker + Performance Benchmarker Tasks sequentially. Each pass → Notion Handoff for `run_id`.

## Team loop

```text
Validate → Load team → Notion handshake (READ+WRITE hub) → SSH
       → Collect (publish briefs) → Classify + draft recs (Notion handoffs)
       → Challenge loop (≤ 3; read/write hub) → Final report (+ Run Index)
       → optional deeper-dive AskQuestion (remote still read-only)
```

| Exit | Behavior |
|------|----------|
| No High-priority rec without evidence; Reality Checker OK | Finalize report |
| Max challenge loops = **3** | Stop; mark disputed recs as unverified |
| User cancel | Stop; no further remote commands |

**Challenge:** Reality Checker + Performance Benchmarker — every 🔴 High recommendation needs cited metrics + host context. May trigger **one** extra read-only collect per loop. Never mutate the database to “prove” a tip.

## Shared Phase 0–2

### Phase 0: Validate

1. FQDN = first token after `/postgres-optimize`; focus = rest (optional)
2. FQDN: ≥1 dot, no spaces; prefer lowercase
3. Invalid/missing → AskQuestion or abort (router message)

### Phase 1: Load team + Notion handshake

Read `lib/multi-agent-loop.md`; build roster. READ contracts/schema; WRITE `run_id` + roster handoff. Record Task dispatch plan + Notion status. Do not bulk-Read every rule into the parent as a substitute for Task when Task works.

### Phase 2: SSH check

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new "<fqdn>" 'hostname -f && echo OK'
```

On failure: report; do not mutate SSH config. Then **Read** `collect.md`.

## Principles

1. Evidence-based, workload-driven, version-aware
2. Host RAM/CPU/disk + `pg_settings` together — no generic boilerplate
3. Summarize; redact secrets
4. Recommendations advisory — user applies outside this command
5. Fail gracefully; list skipped checks

## Error handling

| Problem | Action |
|---------|--------|
| Bad FQDN | AskQuestion or abort |
| SSH fail | Report; stop |
| PG auth fail | OS + file discovery; note gap |
| No `pg_stat_statements` | Use database/table stats; note limit |
| Stats recently reset | Warn ratios may be misleading |
| Standby | Tune for recovery/read; note primary may differ |
| Notion unavailable | In-session briefs only; remote stays read-only |
