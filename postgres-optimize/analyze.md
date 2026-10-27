# PostgreSQL Optimize — analyze module

Loaded after `collect.md` handoff. Do not run standalone. Read-only: recommendations are advisory only.

## Role map (this module)

| Pass | Persona | Output |
|------|---------|--------|
| Classify | Performance Benchmarker + Optimizer | Workload profile + evidence table |
| Draft recs | Database Optimizer (+ DBRE for HA/repl context) | Prioritized recommendations with risk |
| Challenge (≤ 3) | Reality Checker (+ Performance) | READ prior Notion handoffs; drop unevidenced 🔴; WRITE challenge notes; optional one remote recollect |
| Report | Orchestrator | Final English report; WRITE Run Index `done`/`partial` |

## Phase 7: Workload classification

| Profile | Indicators | Tuning priority |
|---------|------------|-----------------|
| **Read-heavy** | Reads ≫ writes; SELECT-heavy statements; high hit ratio; index scans | cache, `work_mem` (careful), parallel reads, `random_page_cost` |
| **Write-heavy** | High DML; WAL bytes; requested checkpoints; `buffers_backend` | WAL/checkpoint, sync_commit trade-offs, autovacuum, pooling |
| **Mixed (OLTP)** | Balanced short queries | Balanced memory, pool connections, churn-aware vacuum |
| **Analytics** | Large seq scans, temp spill, parallel candidates | parallel workers, `work_mem`, `maintenance_work_mem`, replica idea |
| **Standby** | `pg_is_in_recovery()` | hot_standby_feedback, standby_streaming_delay — read-side only |

Score (sum user DBs):

```
read_score  = sum(tup_returned + tup_fetched)
write_score = sum(tup_inserted + tup_updated + tup_deleted)
ratio       = read_score / NULLIF(write_score, 0)
```

| Ratio | Class |
|-------|-------|
| > 10 | Read-heavy |
| 0.1–10 | Mixed |
| < 0.1 | Write-heavy |

Cross-check statements/table stats. If focus contradicts metrics, state both.

## Phase 8: Optimization recommendations

Tailor to: class (Ph7) + host (collect) + current settings + pain signals.

Each recommendation:

| Field | Content |
|-------|---------|
| Parameter / area | e.g. `shared_buffers`, WAL layout, index, sysctl |
| Current | From settings/host |
| Direction | increase / decrease / change / DDL idea |
| Suggested range | Tied to observed RAM/CPU/disk |
| Workload fit | Why for this class |
| Risk | Memory, durability, **reload vs restart** |
| Priority | 🔴 High / 🟡 Medium / 🔵 Low |

### Reference ranges (adapt to version + evidence)

**Memory (primary / mixed OLTP):** `shared_buffers` ~25% RAM dedicated; `effective_cache_size` ~50–75% RAM; `work_mem` from connections×RAM (warn if high `max_connections`); `maintenance_work_mem` up for churn/bloat.

**Write / WAL:** raise `max_wal_size` if `checkpoints_req` dominate; `checkpoint_completion_target` ~0.9 on SSD; `wal_buffers` if WAL wait; separate WAL/data mounts if same disk saturated.

**Read / analytics:** lower `random_page_cost` on SSD (e.g. 1.1–1.5); raise `effective_io_concurrency`; parallel workers if CPU headroom + large scans.

**Connections:** high idle → pooler (PgBouncer); idle-in-transaction → app timeouts.

**Autovacuum:** high `n_dead_tup` → lower scale factors / per-table ideas — **do not VACUUM**.

**OS (advice only):** `vm.swappiness=1` dedicated DB; THP off; `noatime` on data; `nofile` for high conn.

Mark apply method: conf **reload** vs **restart** vs OS vs DDL — user applies outside this command.

## Phase 9: Challenge loop (≤ 3)

1. Reality Checker: every 🔴 needs cited metric + host fact
2. Performance Benchmarker: class and ratio consistent with evidence
3. If dispute → one read-only recollect (return to collect checks) then re-draft
4. Exit when clean or loops exhausted (label unverified)

## Phase 10: Report

```markdown
# PostgreSQL Optimization: <fqdn>

## Executive Summary
<2–4 sentences: class, posture, top 3 opportunities>

**Workload profile:** 📖 Read-heavy / ✍️ Write-heavy / 🔀 Mixed / 📊 Analytics / 🔄 Standby
**Tuning posture:** ✅ Well tuned / ⚠️ Room for improvement / ❌ Significant gaps

## Target
| Field | Value |
|-------|-------|
| FQDN | … |
| Focus / context | … or — |
| SSH user | … |
| PostgreSQL version | … |
| Cluster role | Primary / Standby |
| Analysis time | … (UTC) |
| Agency roster | … |
| Notion bus | ok / unavailable (hub handoffs) |

## Host resources
| Resource | Value | Notes |
|----------|-------|-------|

## Workload analysis
### Classification
| Signal | Value | Interpretation |
|--------|-------|----------------|

### Query patterns
<summary>

## Current configuration highlights
| Parameter | Current | Default | Assessment |
|-----------|---------|---------|------------|

## Optimization recommendations
### 🔴 High priority
| # | Area | Current | Recommendation | Rationale | Apply method |
|---|------|---------|----------------|-----------|--------------|

### 🟡 Medium priority
### 🔵 Low priority / nice-to-have

## Risks & trade-offs
## Focus investigation
<only if focus set>
## Commands skipped
## Recommended next steps
```

## Phase 11: Optional deeper dive

`AskQuestion` unless focus fully addressed. Options: memory, WAL/checkpoints, queries, indexes, replication, raw metrics, **No — done**. Still read-only.
