# PostgreSQL Optimize — collect module

Loaded by `/postgres-optimize` after `_common.md` Phase 2 SSH OK. Do not run standalone.

Produces **orchestrator briefs** for analyze. All checks read-only per `_common`.

## Role map (this module)

| Pass | Persona | Brief |
|------|---------|-------|
| Host | SRE | CPU, RAM, disk/WAL mounts, iostat, swappiness, PG processes |
| Layout + config | DBRE | version, role, paths, non-default conf, extensions |
| Stats | Database Optimizer + Performance Benchmarker | pg_settings snapshot, workload signals, statements if available |

Prefer parallel SSH. Aggregate — no raw megabyte dumps. Cap logs/query rows.

## Phase 3: Host metrics

| Area | Checks | Tuning relevance |
|------|--------|------------------|
| Identity | `hostname -f`, os-release, `uname -a`, `uptime` | Context |
| CPU | `nproc`, `lscpu`, model/MHz from cpuinfo | `max_worker_processes`, parallel workers |
| Memory | `free -h`, MemTotal/HugePages/Swap from meminfo | `shared_buffers`, `effective_cache_size`, `work_mem` |
| Disk / I/O | `df -h`, `df -i`, `lsblk`, `findmnt`, `iostat -x 1 3` if present | Data vs WAL separation, saturation |
| Kernel | `sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio vm.overcommit_memory`; THP enabled file (read) | OS hints |
| Network | `ss -tln`, `ip -br addr` | Connections / repl ports |
| PG OS | `ps aux \| grep -E '[p]ostgres|[p]g_'`, `systemctl status 'postgresql*' --no-pager` | Backend memory layout |

Flag: RAM pressure, swap in use, disk >85%, high iowait, inode exhaustion.

## Phase 4: PostgreSQL layout

- Version; paths `/srv/*/`, `/etc/postgresql/`, `/var/lib/postgresql/`; `tp_db` socket `/srv/<service>/`
- Read conf / auto.conf — non-default summary only
- `SELECT pg_is_in_recovery();` — primary vs standby
- Extensions — especially `pg_stat_statements`

## Phase 5: Connect (read-only)

```bash
sudo -u <pg_service_user> psql -h /srv/<service_name> -U <pg_service_user> -d postgres -Atqc "SELECT version();"
```

Peer/unix only — never embed passwords. Fail → continue with OS + files only.

## Phase 6: Configuration & stats (SELECT-only)

### 6a. Settings baseline (`pg_settings`)

Memory: `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem`, `huge_pages`, `temp_buffers`  
Connections: `max_connections`, reserved; activity counts by state  
WAL: `wal_level`, `fsync`, `synchronous_commit`, `wal_buffers`, `max_wal_size`, `min_wal_size`, checkpoint_*  
Planner: `random_page_cost`, `seq_page_cost`, `effective_io_concurrency`, `default_statistics_target`  
Parallel: `max_worker_processes`, `max_parallel_workers*`  
Autovacuum: on/off, workers, naptime, scale factors  
Logging: `log_min_duration_statement`, checkpoint/autovacuum log knobs  

Note non-defaults vs version defaults.

### 6b. Inventory

`version()`, `pg_control_system()`; DB sizes; tablespaces; top 10 tables by size; extensions per DB.

### 6c. Workload signals (`pg_stat_database`)

`tup_*`, commits/rollbacks, `blks_hit`/`blks_read`, temp files/bytes, deadlocks, checksum failures.

Cache hit example:

```sql
SELECT datname, blks_hit, blks_read,
       CASE WHEN blks_hit + blks_read = 0 THEN NULL
            ELSE round(100.0 * blks_hit / (blks_hit + blks_read), 2)
       END AS cache_hit_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY blks_read DESC;
```

### 6d. Checkpoints / bgwriter

`pg_stat_checkpointer` (PG15+) and/or `pg_stat_bgwriter` — timed vs requested, write/sync times, buffers_*, `maxwritten_clean`.

### 6e. WAL / replication

`pg_stat_archiver`, `pg_stat_replication`, slots; `du -sh` on pg_wal; LSN vs replay on standby.

### 6f. `pg_stat_statements` (if installed)

Top N by total/mean time and calls — read vs write vs I/O vs CPU indicators. Else note and rely on DB/table stats.

### 6g. Table / index health

`pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_statio_user_tables`; catalog-only bloat estimates.

### 6h. Locks / long txs

Activity >5 min, `idle in transaction`; blocking chains summary.

**Focus:** run focus-related queries early with extra depth; keep full collect unless user limited scope.

## Collect handoff

Publish a brief matching Notion **Handoff Schema** to the hub **Handoffs** DB (and keep an in-session copy):

- `run_id`, `fqdn`, `focus`, `pg_version`, `cluster_role`, `from_role`, `phase=collect`
- `host_summary`, `settings_highlights`, `workload_signals`, `pain_signals[]`, `skipped[]`
- `notion_bus`: ok | unavailable

If Notion is up, analyze roles **READ** this handoff from the hub before drafting. Then **Read** `analyze.md`.
