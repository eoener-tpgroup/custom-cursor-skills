---
name: remote-analyze
description: Read-only SSH analysis of remote hosts (general host, PostgreSQL database, or Barman backup server). Requires system type and FQDN; optional focus or context after FQDN (/remote-analyze database db01.example.com replication lag).
---

# Remote Analyze

Performs a **read-only** analysis of a remote system via SSH. Nothing on the remote host is modified — no writes, no restarts, no configuration changes, no DDL/DML.

Triggered manually via `/remote-analyze <type> <fqdn> [focus]`.

## Output language

**Always write all user-facing output in English** — status messages, analysis reports, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

## ⚠️ Arguments: system type + FQDN (+ optional focus)

Two arguments are required **after the command name**; a third is optional:

| # | Argument | Required | Values | Description |
|---|----------|----------|--------|-------------|
| 1 | **System type** | yes | `host` \| `database` \| `barman` | What to analyze |
| 2 | **FQDN** | yes | e.g. `db01.example.com` | Target host (fully qualified domain name) |
| 3 | **Focus / context** | no | free text | Additional information, symptoms, or an explicit area to investigate |

The optional **focus** is everything after the FQDN (may contain spaces). Use it to:

- Name a **specific area** to prioritize (e.g. `replication lag`, `WAL archiving`, `disk usage on /srv`, `failed backups for dbreporting`)
- Provide **symptoms or background** (e.g. `standby stuck in recovery since this morning`, `alerts on backup age`, `connections maxed out`)
- State **constraints or hints** (e.g. `service name dbtexas`, `check only barman server dbamq`)

When a focus is provided, still run the standard analysis for the system type, but **prioritize and expand** checks related to the focus. Summarize focus-specific findings prominently in the report.

```
❌ /remote-analyze                              → missing both required arguments
❌ /remote-analyze host                         → missing FQDN
❌ /remote-analyze db01.example.com             → missing system type
❌ /remote-analyze vm db01.example.com          → invalid system type
✅ /remote-analyze host db01.example.com        → general host analysis
✅ /remote-analyze database db01.example.com    → PostgreSQL analysis
✅ /remote-analyze barman barman01.example.com  → Barman backup analysis
✅ /remote-analyze database db01.example.com replication lag and long-running queries
✅ /remote-analyze barman barman01.example.com WAL gaps for dbreporting since yesterday
✅ /remote-analyze host app01.example.com high load and OOM in logs since 06:00
```

### Missing or invalid arguments

**Do not guess.** If either argument is missing or the system type is invalid:

1. **Prefer `AskQuestion`** (when available) to collect the missing information:
   - Question 1 — system type: options `host`, `database`, `barman`
   - Question 2 — FQDN: ask the user for the target hostname
2. If `AskQuestion` is unavailable or the user does not provide valid values → **abort** with the message below.
3. After prompting, re-validate both arguments before starting any SSH activity.

**Abort message** (report exactly as follows):

> This command requires a system type and a FQDN.
> Usage: `/remote-analyze <host|database|barman> <fqdn> [focus]`
> Example: `/remote-analyze database db01.example.com replication lag`

---

## Read-only safety protocol (strict)

**This command is inspection-only.** Every remote command must be read-only.

### Allowed

- Read files: `cat`, `head`, `tail`, `less` (non-interactive: `head`/`tail` only)
- Inspect state: `systemctl status`, `systemctl is-active`, `systemctl is-enabled` (never `start`/`stop`/`restart`/`reload`)
- Metrics & inventory: `df`, `free`, `uptime`, `uname`, `hostname`, `ls`, `find` (without `-delete`/`-exec` writes), `ss`, `ip`, `lsblk`, `mount`, `findmnt`, `ps`, `journalctl` (read-only, limited lines)
- PostgreSQL: `psql` with **SELECT-only** queries against catalogs and stats views; `\l`, `\du`, `\dx` meta-commands that do not modify data
- Barman: diagnostic/read commands only — `barman check`, `barman status`, `barman list-server`, `barman list-backup`, `barman show-backup`, `barman show-servers`, `barman diagnose`, `barman list-wal`, `barman list-files` (read-only listing)

### Forbidden (never run remotely)

- Any write: `rm`, `mv`, `cp`, `touch`, `chmod`, `chown`, `mkdir`, redirects (`>`/`>>`), `tee`, `sed -i`, `vim`/`nano`
- Service changes: `systemctl start|stop|restart|reload|enable|disable|mask`
- Package management: `apt`, `yum`, `dnf`, `pip install`
- PostgreSQL writes: `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `TRUNCATE`, `VACUUM`, `REINDEX`, `CHECKPOINT`, `pg_reload_conf()`, `SELECT pg_terminate_backend()` (unless purely listing without side effects — prefer catalog views)
- Barman writes: `barman backup`, `barman recover`, `barman delete`, `barman switch-wal`, `barman cron` (can trigger backup side effects), `barman rebuild-xlogdb`, `barman put-wal`
- `sudo` commands that could write or change state (read-only `sudo cat` / `sudo systemctl status` only when strictly necessary and still read-only)

When in doubt, **do not run the command** — note it in the report as "skipped (would require write access)".

---

## Workflow

### Phase 1: Validate arguments

1. Parse from the user prompt (tokens after `/remote-analyze`):
   - **System type** — first token
   - **FQDN** — second token
   - **Focus / context** — optional; all remaining text joined (trimmed). Empty if omitted.
2. Normalize system type to lowercase
3. Validate system type ∈ `{host, database, barman}`
4. Validate FQDN looks like a hostname (contains at least one dot, no spaces, valid characters)
5. If required arguments missing/invalid → prompt via `AskQuestion` or abort (see above). **Do not require focus** — proceed without it if absent.
6. If focus is present, treat it as the user's investigation priority for Phases 3–4 (see below).

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

### Phase 3: Run analysis (by system type)

Run commands via SSH. Prefer **parallel** independent read-only checks where sensible. Cap log tails (`tail -n 200`) and query result sizes.

**When a focus was provided:**

1. Map the focus to relevant read-only checks for the system type (e.g. "replication" → `pg_stat_replication`, recovery status, WAL replay; "backups" → `barman list-backup`, backup age, `barman check`; "disk" → `df`, mount usage, inode pressure).
2. Run focus-related checks **early** and with **extra depth** (more log lines, targeted queries, time-bounded journal filters if the user mentioned a timeframe).
3. Keep the full standard checklist — do not skip unrelated areas unless the user explicitly asked to limit scope (e.g. "only replication").
4. If the focus is ambiguous, interpret it reasonably from context and note assumptions in the report.

---

#### Type: `host` — General host analysis

Collect and summarize:

| Area | Read-only checks |
|------|------------------|
| **Identity** | `hostname -f`, `/etc/os-release`, `uname -a`, `uptime` |
| **Resources** | `df -h`, `free -h`, `lsblk`, `findmnt` |
| **Load** | `uptime`, `nproc`, optional `ps aux --sort=-%mem \| head -20` |
| **Services** | `systemctl list-units --type=service --state=running --no-pager` (summary) |
| **Network** | `ss -tlnp` or `ss -tln` (if `-p` denied), `ip -br addr` |
| **Disk health signals** | inode usage (`df -i`), mount options |
| **Recent errors** | `journalctl -p err --since "24 hours ago" --no-pager -n 50` (if permitted) |
| **Security posture (read-only)** | pending reboot flag (`/var/run/reboot-required`), last logins (`last -n 10` if available) |

Flag anything abnormal: full disks (>85%), high load, failed units, OOM hints in logs.

---

#### Type: `database` — PostgreSQL analysis

**Goal:** Complete read-only health and configuration overview of PostgreSQL on the target host.

##### 3a. Discover PostgreSQL layout

- Running processes: `ps aux | grep -E '[p]ostgres|[p]g_'`
- systemd units: `systemctl status 'postgresql*' --no-pager` (or specific unit if known)
- Common paths in this environment: `/srv/*/`, `/etc/postgresql/`, `/var/lib/postgresql/`
- Detect `tp_db`-style layout: socket under `/srv/<service_name>/`, service user matching service name
- PostgreSQL version: `psql --version` or server `SELECT version();`

##### 3b. Connect read-only

Prefer local socket as service user (typical in managed environments):

```bash
sudo -u <pg_service_user> psql -h /srv/<service_name> -U <pg_service_user> -d postgres -Atqc "SELECT version();"
```

If `sudo` is unavailable, try direct `psql` with the detected socket/port. **Never** prompt for or embed passwords in commands — use peer/unix auth only. If auth fails, report and continue with OS-level checks only.

##### 3c. PostgreSQL queries (SELECT-only)

Run and summarize (group findings, do not dump raw megabytes):

| Category | Queries / views |
|----------|-----------------|
| **Cluster** | `version()`, `pg_is_in_recovery()`, `pg_control_system()`, current timeline |
| **Settings** | Key parameters: `max_connections`, `shared_buffers`, `work_mem`, `wal_level`, `archive_mode`, `max_wal_senders`, `hot_standby` |
| **Databases** | List DBs, sizes (`pg_database_size`), encoding, connection limits |
| **Connections** | `pg_stat_activity` — counts by state, long-running queries (>5 min), idle in transaction |
| **Replication** | `pg_stat_replication` (lag bytes/time), recovery status, slots (`pg_replication_slots`) |
| **WAL / archiving** | `pg_stat_archiver`, WAL directory size, `pg_current_wal_lsn()` vs replay LSN on standby |
| **Performance signals** | `pg_stat_database` (blks hit/read, deadlocks, temp files), `pg_stat_bgwriter`, checkpoint stats |
| **Bloat / maintenance hints** | Table bloat estimates via catalog (read-only queries only), last autovacuum/autanalyze from `pg_stat_user_tables` |
| **Extensions** | `pg_available_extensions` / installed extensions per database |
| **Roles** | Role list (no password hashes), superuser count, connection limits |
| **Tablespaces** | Locations and usage |
| **Locks** | Blocking/waiting lock summary from `pg_locks` + `pg_stat_activity` |

##### 3d. OS-level database context

- Disk usage for data and WAL mount points
- `systemctl is-active` for PostgreSQL
- Recent PostgreSQL logs: `journalctl -u <pg_unit> --no-pager -n 100` or tail PG log file (read-only)

Highlight: replication broken/lagging, archiver failures, disk pressure on data/WAL volumes, excessive connections, long transactions, missing recent vacuum/analyze.

---

#### Type: `barman` — Barman backup server analysis

**Goal:** Complete read-only overview of Barman installation, configured servers, backup/WAL health, and retention.

##### 3a. Discover Barman layout

- Barman user/processes: `ps aux | grep -E '[b]arman'`
- Common paths: `/srv/barman*/`, `/srv/<service>/conf/barman.conf`, `/srv/<service>/log/barman.log`
- Detect service name from directory layout (e.g. `barman`, `barman-dbreporting`, `barman-dbamq`)
- Barman version: `barman --version` (as barman user if needed)
- Cron: `crontab -l -u barman` or `/etc/cron.d/*barman*` (read-only)

##### 3b. Barman read-only commands

Run as the barman service user when required (`sudo -u barman ...`):

| Command | Purpose |
|---------|---------|
| `barman list-server` | Configured PostgreSQL servers |
| `barman status <server>` | Per-server backup/WAL summary |
| `barman check <server>` | Configuration and connectivity check |
| `barman check all` | All servers (if few; otherwise per-server) |
| `barman list-backup <server>` | Available backups |
| `barman show-backup <server> <backup_id>` | Latest backup details |
| `barman list-wal <server>` | WAL archive status (summary, not full dump) |
| `barman diagnose` | Diagnostic bundle (read-only) |

**Do not run** `barman cron`, `barman backup`, or any command that triggers backups or WAL switches.

##### 3c. Configuration & logs (read-only)

- Read main config: `barman.conf`, server configs in `conf.d/` (`cat` only)
- Summarize: retention policy, backup method, archiver settings, last backup max age, compression
- Tail recent log: `tail -n 200 /srv/<service>/log/barman.log`
- Disk usage: `df -h` on backup mount points under `/srv/`

##### 3d. Cross-checks

- SSH connectivity from barman to configured PostgreSQL servers (from `barman check` output — do not manually modify keys)
- Backup age vs retention policy
- WAL archiving continuity (gaps, failed segments)
- Failed/expired backups count
- Cron schedule presence for `barman cron` (note schedule only — do not execute)

Highlight: stale backups, failed checks, WAL archive gaps, disk full on backup volume, servers in error state.

---

### Phase 4: Analysis report

Always produce a structured report:

```markdown
# Remote Analysis: <type> @ <fqdn>

## Executive Summary
<2-4 sentences: overall health + top concerns; if focus was set, lead with focus-related findings>

**Overall status:** ✅ Healthy / ⚠️ Warnings / ❌ Critical issues

## Target
| Field | Value |
|-------|-------|
| System type | host / database / barman |
| FQDN | ... |
| Focus / context | ... (or "—" if none) |
| SSH user | ... |
| Analysis time | ... (UTC) |

## Findings

### ✅ Healthy
- ...

### ⚠️ Warnings
| # | Area | Detail | Recommendation |
|---|------|--------|----------------|
| 1 | ... | ... | ... |

### ❌ Critical
| # | Area | Detail | Recommendation |
|---|------|--------|----------------|
| 1 | ... | ... | ... |

## Detailed observations
<Type-specific sections with collected metrics, tables, and short interpretation>

## Focus investigation
<Only when focus was provided: dedicated section answering the user's question or investigating the named area; include evidence and conclusion>

## Commands skipped
- <command> — <reason, e.g. requires write access or permission denied>

## Recommended next steps
1. ...
```

### Phase 5: Optional deeper dive

After the report, optionally ask (via `AskQuestion` when available) **unless the user already supplied a focus and the report fully addressed it**:

> Do you want a deeper dive into any specific area?

Options: specific subsystem (replication, backups, disk, logs), export raw command output, or **No — done**.

**Do not** run deeper checks that violate read-only rules.

A visual report as a Canvas can be created separately afterward via the canvas skill — it is not part of this command.

---

## Analysis principles

1. **Read-only first** — when a check might mutate state, skip it and document why
2. **Focus-aware** — when the user named a focus, prioritize it without dropping the standard checklist (unless they explicitly limited scope)
3. **Evidence-based** — cite actual command output, not assumptions
4. **Summarize, don't dump** — aggregate metrics; attach raw output only on request
5. **Environment-aware** — prefer `/srv/<service_name>/` layout and `tp_db` / `barman` conventions when detected
6. **Fail gracefully** — partial analysis is OK; clearly mark what could not be checked
7. **No secrets in output** — redact passwords, connection strings with credentials, private keys

## Error handling

| Problem | Action |
|---------|--------|
| Missing/invalid arguments | `AskQuestion` or abort with usage message |
| SSH unreachable | Report error, suggest connectivity/auth checks locally |
| Permission denied | Note which checks failed; continue with permitted read-only checks |
| PostgreSQL auth failed | OS-level + file-based discovery only; report auth gap |
| Barman not installed | Report clearly; fall back to generic `host` checks if useful |
| Command not found on remote | Skip section, note in "Commands skipped" |
| `sudo` required but unavailable | Skip privileged checks, document limitation |

## Verwandte Commands

- `/commit` — Local Git commit
- `/merge-request` — Push branch and create MR/PR
