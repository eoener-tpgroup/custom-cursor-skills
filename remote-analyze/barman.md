# Remote Analyze — subcommand: barman

Loaded by `/remote-analyze barman …` after `remote-analyze/_common.md`. Do not run as a standalone command.

**Invocation:** `/remote-analyze barman <fqdn> [focus]`

Shared safety, Notion, agency, Phase 0–2: see `_common.md`.

## Arguments

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **FQDN** | yes | Barman host |
| 2 | **Focus** | no | e.g. `WAL gaps for dbreporting since yesterday` |

```
✅ /remote-analyze barman barman01.example.com
✅ /remote-analyze barman barman01.example.com WAL gaps for dbreporting since yesterday
```

## Team loop role map

| Pass | Persona | Output brief fields |
|------|---------|---------------------|
| Collect | DBRE | servers, check results, backup age, WAL continuity, disk |
| Collect OS | SRE | backup mounts, cron presence (list only), logs |
| Challenge | Reality Checker | READ/WRITE Notion handoffs; evidence gate |
| Optional | Incident Commander | SEV if backups/WAL Critical |

## Workflow (after Phase 0–2)

### Phase 3: Discover layout

- `ps aux | grep -E '[b]arman'`
- Paths: `/srv/barman*/`, `/srv/<service>/conf/barman.conf`, `/srv/<service>/log/barman.log`
- Service name from layout (`barman`, `barman-dbreporting`, …)
- `barman --version` (as barman user if needed)
- Cron: `crontab -l -u barman` or `/etc/cron.d/*barman*` — **list only; never run `barman cron`**

### Phase 4: Barman read-only commands

Prefer `sudo -u barman …` when required:

| Command | Purpose |
|---------|---------|
| `barman list-server` | Configured PG servers |
| `barman status <server>` | Backup/WAL summary |
| `barman check <server>` / `check all` | Config/connectivity (all only if few servers) |
| `barman list-backup <server>` | Available backups |
| `barman show-backup <server> <id>` | Latest details |
| `barman list-wal <server>` | WAL archive summary (not full dump) |
| `barman diagnose` | Diagnostic bundle (read-only) |

**Never:** `backup`, `recover`, `delete`, `switch-wal`, `cron`, `rebuild-xlogdb`, `put-wal`.

### Phase 5: Config, logs, disk

- `cat` `barman.conf` + `conf.d/` — retention, method, archiver, max age, compression
- `tail -n 200` barman log
- `df -h` on `/srv/` backup mounts

### Phase 6: Cross-checks

- SSH/connectivity hints from `barman check` only (do not rewrite keys)
- Backup age vs retention; WAL gaps; failed/expired count; cron schedule **presence** only

**Focus:** deepen early (WAL gaps, stale backups, named server); keep checklist unless scope limited.

Publish collect briefs to Notion **Handoffs**. Remote stays read-only.

### Phase 7: Challenge loop (≤ 2)

Per `_common`.

### Phase 8: Report

```markdown
# Remote Analysis: barman @ <fqdn>

## Executive Summary
…

**Overall status:** ✅ Healthy / ⚠️ Warnings / ❌ Critical issues

## Target
| Field | Value |
|-------|-------|
| System type | barman |
| FQDN | … |
| Focus / context | … or — |
| SSH user | … |
| Barman version | … |
| Analysis time | … (UTC) |
| Agency roster | … |
| Notion bus | ok / unavailable (hub handoffs) |

## Findings
### ✅ Healthy
### ⚠️ Warnings
### ❌ Critical

## Detailed observations
### Servers & checks
### Backups & retention
### WAL archiving
### Disk & cron

## Focus investigation
## Commands skipped
## Recommended next steps
```

Advisory only — never trigger backups or WAL switches on the remote host. Update Notion **Run Index**.

### Phase 9: Optional deeper dive

Ask unless focus covered. Options: one server deep-dive, WAL continuity, disk, logs, **No — done**. Still read-only.
