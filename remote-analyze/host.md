# Remote Analyze — subcommand: host

Loaded by `/remote-analyze host …` after `remote-analyze/_common.md`. Do not run as a standalone command.

**Invocation:** `/remote-analyze host <fqdn> [focus]`

Shared safety, Notion, agency, Phase 0–2: see `_common.md`.

## Arguments

| # | Argument | Required | Description |
|---|----------|----------|-------------|
| 1 | **FQDN** | yes | Target host |
| 2 | **Focus** | no | e.g. `high load and OOM since 06:00` |

```
✅ /remote-analyze host db01.example.com
✅ /remote-analyze host app01.example.com high load and OOM in logs since 06:00
```

## Team loop role map

| Pass | Persona | Output brief fields |
|------|---------|---------------------|
| Collect | SRE | identity, resources, load, services, network, journal errors |
| Optional | Network Engineer | ports, listeners, path issues (if focus) |
| Challenge | Reality Checker | READ Notion handoffs; drop unevidenced claims; WRITE challenge notes |
| Optional | Incident Commander | SEV framing if Critical (Notion handoff OK) |

## Workflow (after Phase 0–2)

### Phase 3: Collect (read-only)

Prefer parallel SSH checks. Cap logs (`journalctl … -n 50`, `tail -n 200`).

| Area | Checks |
|------|--------|
| Identity | `hostname -f`, `/etc/os-release`, `uname -a`, `uptime` |
| Resources | `df -h`, `df -i`, `free -h`, `lsblk`, `findmnt` |
| Load | `nproc`, `ps aux --sort=-%mem \| head -20` |
| Services | `systemctl list-units --type=service --state=running --no-pager` (summary) |
| Network | `ss -tln` / `ss -tlnp` if allowed, `ip -br addr` |
| Errors | `journalctl -p err --since "24 hours ago" --no-pager -n 50` |
| Security signals | `/var/run/reboot-required`, `last -n 10` if available |

Flag: disk >85%, inode pressure, high load, failed units, OOM hints.

**Focus:** map early (disk → `df`/`findmnt`; OOM → journal/mem; network → `ss`/`ip`) with extra depth; keep full checklist unless user limited scope.

Publish collect brief to Notion **Handoffs** (`phase=collect`, no secrets). Remote stays read-only.

### Phase 4: Challenge loop (≤ 2)

Reality Checker: every Warning/Critical must cite output. Orchestrator may re-collect **read-only** once per loop for disputed items only.

### Phase 5: Report

```markdown
# Remote Analysis: host @ <fqdn>

## Executive Summary
<2–4 sentences; lead with focus findings if set>

**Overall status:** ✅ Healthy / ⚠️ Warnings / ❌ Critical issues

## Target
| Field | Value |
|-------|-------|
| System type | host |
| FQDN | … |
| Focus / context | … or — |
| SSH user | … |
| Analysis time | … (UTC) |
| Agency roster | loaded / missing |
| Notion bus | ok / unavailable (hub handoffs) |

## Findings
### ✅ Healthy
- …

### ⚠️ Warnings
| # | Area | Detail | Recommendation |
|---|------|--------|----------------|

### ❌ Critical
| # | Area | Detail | Recommendation |
|---|------|--------|----------------|

## Detailed observations
<metrics tables + short interpretation>

## Focus investigation
<only if focus set>

## Commands skipped
- <cmd> — <reason>

## Recommended next steps
1. …
```

Recommendations are **advisory** — this command never applies them on the remote host. Update Notion **Run Index** to `done`/`partial`.

### Phase 6: Optional deeper dive

`AskQuestion` unless focus already fully answered. Options: disk, memory/OOM, services, network, raw excerpts, **No — done**. Still read-only.
