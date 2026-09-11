---
name: postgres-optimize
description: Read-only SSH analysis and performance tuning recommendations for PostgreSQL. Requires FQDN; optional focus or context (/postgres-optimize db01.example.com OLTP write-heavy).
---

# PostgreSQL Optimize

Performs a **read-only** performance analysis of PostgreSQL on a remote host via SSH. Collects host and database metrics relevant to configuration tuning, classifies the workload (read-heavy, write-heavy, or mixed), and delivers actionable optimization recommendations — **without modifying anything** on the remote system.

Triggered manually via `/postgres-optimize <fqdn> [focus]`.

## Output language

**Always write all user-facing output in English** — status messages, analysis reports, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

## ⚠️ Arguments: FQDN (+ optional focus)

One argument is required **after the command name**; a second is optional:

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **FQDN** | yes | Target host (fully qualified domain name), e.g. `db01.example.com` |
| 2 | **Focus / context** | no | Free text — workload hints, symptoms, constraints, or areas to prioritize |

The optional **focus** is everything after the FQDN (may contain spaces). Use it to:

- State known workload characteristics (e.g. `OLTP write-heavy`, `analytics read-mostly`, `mixed reporting + writes`)
- Name symptoms (e.g. `high checkpoint frequency`, `slow sequential scans`, `WAL disk saturated`)
- Provide constraints (e.g. `cannot increase RAM`, `service name dbtexas`, `PostgreSQL 15 only`)
- Limit or prioritize scope (e.g. `focus on memory and WAL tuning only`)

When a focus is provided, still run the full standard collection, but **prioritize and expand** analysis in the named areas.

```
❌ /postgres-optimize                              → missing FQDN
❌ /postgres-optimize db01                         → FQDN must be fully qualified (contains a dot)
✅ /postgres-optimize db01.example.com             → full optimization analysis
✅ /postgres-optimize db01.example.com OLTP write-heavy, WAL disk pressure
✅ /postgres-optimize db01.example.com focus on shared_buffers and checkpoint tuning
```

### Missing or invalid arguments

**Do not guess.** If the FQDN is missing or invalid:

1. **Prefer `AskQuestion`** (when available) to ask for the target FQDN
2. If `AskQuestion` is unavailable or the user does not provide a valid FQDN → **abort** with the message below
3. Re-validate the FQDN before starting any SSH activity

**Abort message** (report exactly as follows):

> This command requires a FQDN as an argument.
> Usage: `/postgres-optimize <fqdn> [focus]`
> Example: `/postgres-optimize db01.example.com`

---

## Read-only safety protocol (strict)

**This command is inspection-only.** Every remote command must be read-only. No exceptions.

### Allowed

- Read files: `cat`, `head`, `tail` (non-interactive only)
- Inspect state: `systemctl status`, `systemctl is-active`, `systemctl is-enabled` (never `start`/`stop`/`restart`/`reload`)
- Metrics & inventory: `df`, `df -i`, `free`, `uptime`, `uname`, `hostname`, `ls`, `find` (without `-delete`/`-exec` writes), `ss`, `ip`, `lsblk`, `mount`, `findmnt`, `ps`, `vmstat`, `iostat`, `pidstat`, `cat /proc/meminfo`, `cat /proc/cpuinfo`, `sysctl -a` (read-only), `journalctl` (read-only, limited lines)
- PostgreSQL: `psql` with **SELECT-only** queries against catalogs, stats views, and `pg_settings`; meta-commands `\l`, `\du`, `\dx` that do not modify data
- Config files: read `postgresql.conf`, `postgresql.auto.conf`, `pg_hba.conf` via `cat` only

### Forbidden (never run remotely)

- Any write: `rm`, `mv`, `cp`, `touch`, `chmod`, `chown`, `mkdir`, redirects (`>`/`>>`), `tee`, `sed -i`, editors
- Service changes: `systemctl start|stop|restart|reload|enable|disable|mask`
- Package management: `apt`, `yum`, `dnf`, `pip install`
- PostgreSQL writes or side effects: `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `TRUNCATE`, `VACUUM`, `ANALYZE`, `REINDEX`, `CHECKPOINT`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`, `pg_reload_conf()`, `pg_rotate_logfile()`, `SELECT pg_terminate_backend()`, `SELECT pg_cancel_backend()`
- Benchmark or load tools that generate I/O: `pgbench` (with writes), `fio` (write tests), stress tools
- `sudo` commands that could write or change state (read-only `sudo cat`, `sudo -u <pg_user> psql ...` only when strictly necessary and still read-only)

When in doubt, **do not run the command** — note it in the report as "skipped (would require write access or have side effects)".

---

## Workflow

### Phase 1: Validate arguments

1. Parse from the user prompt (tokens after `/postgres-optimize`):
   - **FQDN** — first token
   - **Focus / context** — optional; all remaining text joined (trimmed). Empty if omitted.
2. Normalize FQDN to lowercase (preserve case only if required for DNS — default lowercase)
3. Validate FQDN looks like a hostname (contains at least one dot, no spaces, valid characters)
4. If FQDN missing/invalid → prompt via `AskQuestion` or abort (see above). **Do not require focus** — proceed without it if absent.
5. If focus is present, treat it as investigation priority for Phases 3–5.

### Phase 2: SSH connectivity check

1. Verify `ssh` is available locally
2. Test connection (read-only):

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new "<fqdn>" 'hostname -f && echo OK'
```

3. On failure, report clearly:
   - Host unreachable / timeout
   - Auth failure (keys, agent, user)
   - Host key verification issues
4. Detect remote user context (`whoami`, `id`) — note if `sudo` is needed for read-only inspection

**Do not** modify `~/.ssh/config` or remote files to fix connectivity.

### Phase 3: Collect host metrics (read-only)

Run commands via SSH. Prefer **parallel** independent read-only checks where sensible.

| Area | Read-only checks | Purpose |
|------|------------------|---------|
| **Identity** | `hostname -f`, `/etc/os-release`, `uname -a`, `uptime` | Environment context |
| **CPU** | `nproc`, `lscpu` (if available), `grep -E 'model name|cpu MHz' /proc/cpuinfo`, load from `uptime` | Sizing `max_worker_processes`, `max_parallel_workers*` |
| **Memory** | `free -h`, `cat /proc/meminfo` (MemTotal, HugePages, Swap) | Sizing `shared_buffers`, `effective_cache_size`, `work_mem` |
| **Disk / I/O** | `df -h`, `df -i`, `lsblk`, `findmnt`, `iostat -x 1 3` (if available) | Data vs WAL mount separation, I/O saturation |
| **Kernel / VM** | `sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio vm.overcommit_memory` | OS-level tuning hints |
| **Network** | `ss -tln`, `ip -br addr` | Connection volume, replication ports |
| **PostgreSQL OS context** | `ps aux \| grep -E '[p]ostgres|[p]g_'`, `systemctl status 'postgresql*' --no-pager` | Process layout, memory per backend |

Flag host-level constraints: RAM pressure, swap in use, disk >85% full, high iowait, inode exhaustion.

### Phase 4: Discover PostgreSQL layout

- Detect version: `psql --version` or `SELECT version();`
- Common paths in this environment: `/srv/*/`, `/etc/postgresql/`, `/var/lib/postgresql/`
- Detect `tp_db`-style layout: socket under `/srv/<service_name>/`, service user matching service name
- Read config files (read-only): `postgresql.conf`, `postgresql.auto.conf` — summarize non-default settings
- Cluster role: `SELECT pg_is_in_recovery();` — primary vs standby affects tuning recommendations
- Extensions: especially `pg_stat_statements` (critical for workload analysis)

### Phase 5: Connect read-only

Prefer local socket as service user (typical in managed environments):

```bash
sudo -u <pg_service_user> psql -h /srv/<service_name> -U <pg_service_user> -d postgres -Atqc "SELECT version();"
```

If `sudo` is unavailable, try direct `psql` with the detected socket/port. **Never** prompt for or embed passwords in commands — use peer/unix auth only. If auth fails, report and continue with OS-level + file-based discovery only.

### Phase 6: Collect PostgreSQL configuration & stats (SELECT-only)

Run and summarize (aggregate metrics; do not dump raw megabytes).

#### 6a. Configuration baseline

| Category | Source | Key parameters |
|----------|--------|----------------|
| **Memory** | `pg_settings` | `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem`, `huge_pages`, `temp_buffers` |
| **Connections** | `pg_settings` + `pg_stat_activity` | `max_connections`, `superuser_reserved_connections`, current connection count by state |
| **WAL / durability** | `pg_settings` | `wal_level`, `fsync`, `synchronous_commit`, `wal_buffers`, `max_wal_size`, `min_wal_size`, `checkpoint_timeout`, `checkpoint_completion_target` |
| **Planner** | `pg_settings` | `random_page_cost`, `seq_page_cost`, `effective_io_concurrency`, `default_statistics_target` |
| **Parallelism** | `pg_settings` | `max_worker_processes`, `max_parallel_workers`, `max_parallel_workers_per_gather`, `max_parallel_maintenance_workers` |
| **Autovacuum** | `pg_settings` | `autovacuum`, `autovacuum_max_workers`, `autovacuum_naptime`, `autovacuum_vacuum_scale_factor`, `autovacuum_analyze_scale_factor` |
| **Logging** | `pg_settings` | `log_min_duration_statement`, `log_checkpoints`, `log_autovacuum_min_duration` |

Compare current values against PostgreSQL defaults and note non-default overrides from `postgresql.conf` / `postgresql.auto.conf`.

#### 6b. Cluster & database inventory

| Query / view | Purpose |
|--------------|---------|
| `version()`, `pg_control_system()` | Version and cluster identity |
| `pg_database` + `pg_database_size()` | Database count and sizes |
| `pg_tablespace` + `pg_tablespace_location()` | Tablespace layout |
| Largest tables (`pg_class` + `pg_namespace` + size estimates) | Top 10 by size |
| Installed extensions per database | Feature availability |

#### 6c. Workload signals (for classification)

Use **cluster-wide and per-database** stats from `pg_stat_database` (since last reset):

| Metric | Interpretation |
|--------|----------------|
| `tup_returned` vs `tup_fetched` | Read volume (sequential vs index-assisted) |
| `tup_inserted` + `tup_updated` + `tup_deleted` | Write volume |
| `xact_commit` vs `xact_rollback` | Transaction rate and error rate |
| `blks_hit` vs `blks_read` | Buffer cache hit ratio |
| `temp_files`, `temp_bytes` | Sort/hash spill to disk (work_mem pressure) |
| `deadlocks` | Concurrency issues |
| `checksum_failures` | Hardware/storage integrity (if checksums enabled) |

**Hit ratio** (per database):

```sql
SELECT datname,
       blks_hit,
       blks_read,
       CASE WHEN blks_hit + blks_read = 0 THEN NULL
            ELSE round(100.0 * blks_hit / (blks_hit + blks_read), 2)
       END AS cache_hit_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY blks_read DESC;
```

#### 6d. Background writer & checkpoint behavior

From `pg_stat_bgwriter` and `pg_stat_checkpointer` (PG15+) or `pg_stat_bgwriter` alone (older versions):

| Metric | Interpretation |
|--------|----------------|
| `checkpoints_timed` vs `checkpoints_req` | Checkpoint pressure (too many requested = tuning needed) |
| `checkpoint_write_time`, `checkpoint_sync_time` | I/O cost of checkpoints |
| `buffers_checkpoint`, `buffers_clean`, `buffers_backend` | Who writes dirty buffers |
| `maxwritten_clean` | Background writer falling behind |

#### 6e. WAL & archiving (if applicable)

| Source | Purpose |
|--------|---------|
| `pg_stat_archiver` | Archive success/failure rate |
| `pg_stat_replication` | Replication lag (bytes/time) |
| `pg_replication_slots` | Slot lag, inactive slots |
| WAL directory size (`du -sh` on pg_wal path) | WAL disk pressure |
| `pg_current_wal_lsn()` vs replay LSN (on standby) | Recovery lag |

#### 6f. Query-level stats (if `pg_stat_statements` available)

```sql
SELECT EXISTS (
  SELECT 1 FROM pg_extension WHERE extname = 'pg_stat_statements'
);
```

If available, summarize (top N by total time, calls, mean time):

- Read-heavy indicators: high `shared_blks_hit`, `shared_blks_read`, SELECT-dominated query text
- Write-heavy indicators: high `wal_bytes`, `blk_write_time`, INSERT/UPDATE/DELETE-dominated query text
- CPU-heavy: high `total_exec_time` with low I/O blocks
- I/O-heavy: high `blk_read_time` + `blk_write_time`

If not available, note it and rely on `pg_stat_database` + `pg_stat_user_tables` only.

#### 6g. Table & index health

| Source | Purpose |
|--------|---------|
| `pg_stat_user_tables` | Seq scans vs idx scans, `n_tup_ins/upd/del`, `n_live_tup`, `n_dead_tup`, last autovacuum/autanalyze |
| `pg_stat_user_indexes` | Index usage (`idx_scan`), unused indexes |
| `pg_statio_user_tables` | Heap vs index block reads |
| Bloat estimates (catalog-based, read-only) | Tables needing vacuum attention |

#### 6h. Locks & long transactions

| Source | Purpose |
|--------|---------|
| `pg_stat_activity` | Counts by state, queries running >5 min, `idle in transaction` |
| `pg_locks` + `pg_stat_activity` | Blocking chains summary |

### Phase 7: Workload classification

Classify the database workload based on collected evidence. Use **quantitative thresholds** where possible; state assumptions when stats were recently reset.

#### Classification matrix

| Profile | Indicators | Typical tuning priority |
|---------|------------|------------------------|
| **Read-heavy** | `tup_returned` >> write tuples; high SELECT share in `pg_stat_statements`; high `blks_hit` ratio; many index scans | `effective_cache_size`, `shared_buffers`, `work_mem` (careful), parallel reads, index optimization, `random_page_cost` |
| **Write-heavy** | High `tup_inserted/updated/deleted`; high `wal_bytes`; frequent requested checkpoints; high `buffers_backend` | WAL sizing (`max_wal_size`, `wal_buffers`), `checkpoint_completion_target`, `synchronous_commit` trade-offs, autovacuum aggressiveness, connection pooling |
| **Mixed (OLTP)** | Balanced read/write; moderate connection count; mix of short queries | Balanced memory, right-size `work_mem`, pool connections, autovacuum tuned for churn |
| **Analytics / reporting** | Large seq scans, high `temp_bytes`, parallel query candidates | `max_parallel_workers*`, `work_mem`, `maintenance_work_mem`, separate read replica consideration |
| **Replication standby** | `pg_is_in_recovery() = true` | `hot_standby_feedback`, `max_standby_streaming_delay`, recovery prefetch (version-dependent) — read-only tuning only |

#### Scoring approach

Compute a simple read vs write ratio from `pg_stat_database` (sum across user databases):

```
read_score  = sum(tup_returned + tup_fetched)
write_score = sum(tup_inserted + tup_updated + tup_deleted)
ratio       = read_score / NULLIF(write_score, 0)
```

| Ratio | Classification |
|-------|----------------|
| ratio > 10 | **Read-heavy** |
| ratio 0.1 – 10 | **Mixed** |
| ratio < 0.1 | **Write-heavy** |

Cross-check with `pg_stat_statements` (if available) and table-level stats. If focus text contradicts metrics, note both and explain.

### Phase 8: Optimization recommendations

Produce **actionable, prioritized** recommendations tailored to:

1. **Detected workload class** (Phase 7)
2. **Host resources** (Phase 3) — RAM, CPU, disk type/layout
3. **Current PostgreSQL settings** vs best-practice ranges for the detected PG version
4. **Observed pain points** (checkpoint pressure, cache misses, bloat, replication lag, temp file spill)

#### Recommendation structure

For each recommendation include:

| Field | Content |
|-------|---------|
| **Parameter / area** | e.g. `shared_buffers`, WAL layout, index, OS sysctl |
| **Current value** | From `pg_settings` or host |
| **Suggested direction** | Increase / decrease / change / add index / restructure |
| **Suggested range or value** | With rationale tied to observed RAM/CPU/disk |
| **Workload fit** | Why this helps read-heavy / write-heavy / mixed |
| **Risk / trade-off** | Memory pressure, durability impact, restart required (`postmaster` vs `reload`) |
| **Priority** | 🔴 High / 🟡 Medium / 🔵 Low |

#### Standard tuning reference (adapt to version and evidence)

**Memory (primary / mixed OLTP):**

- `shared_buffers`: typically 25% of RAM on dedicated DB host (lower if shared with app); compare to current
- `effective_cache_size`: ~50–75% of RAM (OS cache estimate for planner)
- `work_mem`: derive from `max_connections` and RAM — warn if high connection count makes global `work_mem` dangerous; prefer per-role or per-query limits
- `maintenance_work_mem`: higher for write-heavy / bloat-prone (autovacuum, CREATE INDEX)

**Write-heavy / WAL:**

- `max_wal_size` / `min_wal_size`: reduce checkpoint frequency if `checkpoints_req` dominates
- `checkpoint_completion_target`: 0.9 on SSD/NVMe (spread checkpoint I/O)
- `wal_buffers`: increase if WAL write wait observed (version-dependent defaults)
- Separate WAL and data on different mounts if same disk saturated

**Read-heavy / analytics:**

- `effective_cache_size` and `shared_buffers` alignment with RAM
- `random_page_cost`: lower on SSD (e.g. 1.1–1.5)
- `effective_io_concurrency`: higher on SSD/NVMe
- Parallel query workers if CPU headroom and large scans dominate

**Connections:**

- If `max_connections` high and many idle → recommend pooler (PgBouncer) — note if not present
- `idle in transaction` sessions → application/session timeout recommendations

**Autovacuum:**

- High `n_dead_tup` / bloat → lower scale factors or per-table settings (recommendation only — do not run VACUUM)
- Write-heavy tables → more aggressive autovacuum for hot tables

**OS / host (recommendations only — no changes):**

- `vm.swappiness = 1` on dedicated PostgreSQL hosts
- Ensure `transparent huge pages` disabled (check `/sys/kernel/mm/transparent_hugepage/enabled` read-only)
- Filesystem mount options (`noatime` on data volumes)
- `ulimit` / `nofile` for high connection counts

**Explicitly mark** recommendations that require:

- `postgresql.conf` change + **reload** (`pg_reload_conf` — user must apply; this command does not)
- `postgresql.conf` change + **restart** (e.g. `shared_buffers`, `max_connections`)
- OS-level changes (sysctl, mounts)
- DDL (indexes, partitioning) — query tuning layer

### Phase 9: Optimization report

Always produce a structured report:

```markdown
# PostgreSQL Optimization: <fqdn>

## Executive Summary
<2-4 sentences: workload class, overall tuning posture, top 3 opportunities>

**Workload profile:** 📖 Read-heavy / ✍️ Write-heavy / 🔀 Mixed / 📊 Analytics / 🔄 Standby (replica)
**Tuning posture:** ✅ Well tuned / ⚠️ Room for improvement / ❌ Significant gaps

## Target
| Field | Value |
|-------|-------|
| FQDN | ... |
| Focus / context | ... (or "—") |
| SSH user | ... |
| PostgreSQL version | ... |
| Cluster role | Primary / Standby |
| Analysis time | ... (UTC) |

## Host resources
| Resource | Value | Notes |
|----------|-------|-------|
| CPU cores | ... | ... |
| RAM | ... | ... |
| Data disk | ... | usage, mount, filesystem |
| WAL disk | ... | separate? usage |
| Swap in use | ... | ... |
| Load average | ... | ... |

## Workload analysis
### Classification
<Read-heavy / Write-heavy / Mixed — with evidence table>

| Signal | Value | Interpretation |
|--------|-------|----------------|
| Read/write ratio | ... | ... |
| Cache hit ratio | ...% | ... |
| Commits/sec (approx) | ... | ... |
| Checkpoint behavior | ... | timed vs requested |
| Top wait / pain signals | ... | ... |

### Query patterns
<Summary from pg_stat_statements or fallback stats>

## Current configuration highlights
| Parameter | Current | Default | Assessment |
|-----------|---------|---------|------------|
| shared_buffers | ... | ... | ... |
| effective_cache_size | ... | ... | ... |
| work_mem | ... | ... | ... |
| max_connections | ... | ... | ... |
| max_wal_size | ... | ... | ... |
| ... | ... | ... | ... |

## Optimization recommendations
### 🔴 High priority
| # | Area | Current | Recommendation | Rationale | Apply method |
|---|------|---------|----------------|-----------|--------------|
| 1 | ... | ... | ... | ... | reload / restart / OS / DDL |

### 🟡 Medium priority
...

### 🔵 Low priority / nice-to-have
...

## Risks & trade-offs
- ...

## Focus investigation
<Only when focus was provided: dedicated section addressing the user's question>

## Commands skipped
- <command> — <reason>

## Recommended next steps
1. ...
```

### Phase 10: Optional deeper dive

After the report, optionally ask (via `AskQuestion` when available) **unless the user already supplied a focus and the report fully addressed it**:

> Do you want a deeper dive into any specific area?

Options: memory tuning, WAL/checkpoints, query analysis, index review, replication, export raw metrics, or **No — done**.

**Do not** run deeper checks that violate read-only rules.

---

## Analysis principles

1. **Read-only always** — when a check might mutate state, skip it and document why
2. **Evidence-based** — cite actual metrics; distinguish facts from recommendations
3. **Workload-driven** — every tuning suggestion must tie back to the classified workload and observed bottlenecks
4. **Version-aware** — parameter names and defaults differ by PostgreSQL major version; note version-specific advice
5. **Holistic** — consider host RAM/CPU/disk together with `pg_settings`; avoid generic boilerplate
6. **Summarize, don't dump** — aggregate metrics; raw output only on request
7. **Environment-aware** — prefer `/srv/<service_name>/` layout and `tp_db` conventions when detected
8. **Fail gracefully** — partial analysis is OK; clearly mark what could not be checked
9. **No secrets** — redact passwords, connection strings with credentials
10. **No changes** — recommendations are advisory; the user applies them outside this command

## Error handling

| Problem | Action |
|---------|--------|
| Missing/invalid FQDN | `AskQuestion` or abort with usage message |
| SSH unreachable | Report error, suggest connectivity/auth checks locally |
| Permission denied | Note which checks failed; continue with permitted read-only checks |
| PostgreSQL auth failed | OS-level + file-based discovery only; report auth gap |
| `pg_stat_statements` not available | Note limitation; use `pg_stat_database` / table stats |
| Stats recently reset | Warn that ratios may not reflect long-term workload |
| Standby / replica | Tune for recovery/read workload only; note primary may differ |
| Command not found on remote | Skip section, note in "Commands skipped" |
| `sudo` required but unavailable | Skip privileged checks, document limitation |

## Verwandte Commands

- `/remote-analyze database <fqdn> [focus]` — General read-only PostgreSQL health analysis (broader than tuning)
- `/remote-analyze host <fqdn> [focus]` — General host analysis without optimization focus
- `/commit` — Local Git commit
