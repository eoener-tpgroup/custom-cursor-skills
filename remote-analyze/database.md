# Remote Analyze — subcommand: database

Loaded by `/remote-analyze database …` after `remote-analyze/_common.md`. Do not run as a standalone command.

**Invocation:** `/remote-analyze database <fqdn> [focus]`

Shared safety, agency, Phase 0–2: see `_common.md`. For tuning recommendations use `/postgres-optimize`.

## Arguments

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **FQDN** | yes | PostgreSQL host |
| 2 | **Focus** | no | e.g. `replication lag and long-running queries` |

```
✅ /remote-analyze database db01.example.com
✅ /remote-analyze database db01.example.com replication lag and long-running queries
```

## Team loop role map

| Pass | Persona | Output brief fields |
|------|---------|---------------------|
| Collect OS | SRE | mounts, PG processes, units, disk on data/WAL |
| Collect PG | DBRE | cluster role, replication, WAL/archiver, connections, locks |
| Optional | Database Optimizer | perf signals if focus is slow queries / cache / bloat |
| Challenge | Reality Checker | READ prior in-session briefs; evidence gate |
| Optional | Incident Commander | SEV if replication/archiver Critical |

## Workflow (after Phase 0–2)

### Phase 3: Discover layout + connect (read-only)

- Processes: `ps aux | grep -E '[p]ostgres|[p]g_'`
- Units: `systemctl status 'postgresql*' --no-pager`
- Paths: `/srv/*/`, `/etc/postgresql/`, `/var/lib/postgresql/`; `tp_db` → socket `/srv/<service>/`, user ≈ service name
- Version: `psql --version` or `SELECT version();`

Prefer:

```bash
sudo -u <pg_service_user> psql -h /srv/<service_name> -U <pg_service_user> -d postgres -Atqc "SELECT version();"
```

No passwords in commands — peer/unix only. Auth fail → OS/file discovery only.

### Phase 4: Collect PostgreSQL (SELECT-only)

Summarize; do not dump raw megabytes. Cap activity/log tails.

| Category | Sources |
|----------|---------|
| Cluster | `version()`, `pg_is_in_recovery()`, `pg_control_system()` |
| Settings | `max_connections`, `shared_buffers`, `work_mem`, `wal_level`, `archive_mode`, `max_wal_senders`, `hot_standby` |
| Databases | sizes, encoding, conn limits |
| Connections | `pg_stat_activity` by state; >5 min; `idle in transaction` |
| Replication | `pg_stat_replication`, slots |
| WAL / archive | `pg_stat_archiver`, WAL dir size, LSN vs replay on standby |
| Perf signals | `pg_stat_database`, bgwriter/checkpointer, temp files, deadlocks |
| Maintenance hints | `pg_stat_user_tables` last vacuum/analyze; catalog bloat estimates only |
| Extensions / roles / tablespaces | installed exts; roles **without** password hashes; tablespace locations |
| Locks | blocking summary `pg_locks` + activity |

### Phase 5: OS database context

Data/WAL `df -h`; `systemctl is-active`; `journalctl -u <pg_unit> --no-pager -n 100` or PG log tail (read-only).

**Focus:** e.g. replication → lag/slots/recovery first with extra depth; keep checklist unless scope limited.

Publish collect briefs in-session (OS + PG passes). Remote stays read-only.

### Phase 6: Challenge loop (≤ 2)

Same as `_common`: no unevidenced Critical/Warning; optional one read-only recollect per loop.

### Phase 7: Report

```markdown
# Remote Analysis: database @ <fqdn>

## Executive Summary
…

**Overall status:** ✅ Healthy / ⚠️ Warnings / ❌ Critical issues

## Target
| Field | Value |
|-------|-------|
| System type | database |
| FQDN | … |
| Focus / context | … or — |
| SSH user | … |
| PostgreSQL version | … |
| Cluster role | Primary / Standby |
| Analysis time | … (UTC) |
| Agency roster | … |

## Findings
### ✅ Healthy
### ⚠️ Warnings
### ❌ Critical

## Detailed observations
### Cluster & configuration
### Connections & locks
### Replication & WAL
### Performance / maintenance signals

## Focus investigation
## Commands skipped
## Recommended next steps
```

Advisory only on the remote host. Point to `/postgres-optimize <fqdn>` when tuning is the next step.

### Phase 8: Optional deeper dive

Ask unless focus fully covered. Options: replication, connections/locks, WAL/archive, vacuum/bloat, logs, **No — done**. Still read-only.
