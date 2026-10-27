# Agent Communication — Notion schema

**Not a slash-command.** Shared hub for `/remote-analyze` and `/postgres-optimize` agent teams.

**Hub:** https://app.notion.com/p/trimble-tl/Agent-Communication-402c1b3dd25680b79d40cfc13c3d9211  
**Page id:** `402c1b3dd25680b79d40cfc13c3d9211`

## Hard rules

1. **Remote hosts:** absolute read-only — never write, restart, DDL/DML, Barman mutate, etc.
2. **Notion:** agents **may write only under this hub** (child pages/DBs) to communicate and pass information.
3. **Never** write to any other Notion page/workspace from these workflows.
4. **No secrets** in any Notion property (passwords, tokens, keys, credentialed URLs).

## Children under the hub

Create under the hub if missing (agents may create these children; humans may pre-seed).

### 1. Database: Agent Contracts

| Property | Type | Notes |
|----------|------|-------|
| Name | Title | e.g. `Orchestrator`, `DBRE`, `Reality Checker` |
| Command | Select | `remote-analyze` / `postgres-optimize` / `both` |
| Role key | Text | `orchestrator`, `sre`, `dbre`, `optimizer`, … |
| Rules path | Text | `~/.cursor/rules/….mdc` |
| Inputs | Text | Expected brief fields |
| Outputs | Text | Produced brief fields |
| Tool scope | Text | `remote: read-only` · `notion: hub write OK` |

### 2. Database: Handoff Schema

| Property | Type | Notes |
|----------|------|-------|
| Name | Title | Field name (`run_id`, `fqdn`, …) |
| Phase | Select | `handshake` / `collect` / `classify` / `recommend` / `challenge` / `report` |
| Type | Select | `string` / `enum` / `list` / `object` |
| Required | Checkbox | |
| Description | Text | |

Minimum fields: `run_id`, `fqdn`, `command`, `phase`, `from_role`, `to_role`, `findings`/`recommendations`, `skipped`, `severity`, `evidence_refs`.

### 3. Database: Handoffs (agent bus)

| Property | Type | Notes |
|----------|------|-------|
| Name | Title | Short label (`collect-host`, `challenge-1`, …) |
| run_id | Text | Correlates one analysis run |
| Command | Select | `remote-analyze` / `postgres-optimize` |
| Phase | Select | Same as schema phases |
| from_role | Text | |
| to_role | Text | `orchestrator` / next role / `all` |
| FQDN | Text | |
| Payload | Text | Brief body (summarized metrics/findings — no secrets, no huge dumps) |
| Severity | Select | `info` / `warning` / `critical` / `n/a` |
| Created | Date | |

**Primary write path** for inter-agent talk. Downstream roles READ rows for the same `run_id` before acting.

### 4. Database: Environment Notes

| Property | Type | Notes |
|----------|------|-------|
| Name | Title | Convention name |
| Pattern | Text | e.g. `/srv/<service>/` |
| Applies to | Multi-select | `postgres`, `barman`, `host` |
| Notes | Text | Non-secret only |

### 5. Database: Run Index

| Property | Type | Notes |
|----------|------|-------|
| Name | Title | Short label |
| run_id | Text | |
| FQDN | Text | |
| Command | Select | |
| Type | Select | `host` / `database` / `barman` / `optimize` |
| Summary | Text | Executive digest (agent-written OK) |
| Status | Select | `running` / `done` / `partial` / `aborted` |
| Analyzed at | Date | |
| Handoff count | Number | Optional |

### 6. Page: Protocol (doc)

1. Inter-agent traffic = **Handoffs** DB rows (plus optional in-session cache)  
2. Remote inspection is read-only forever  
3. Notion writes stay under this hub only  
4. No secrets  

## Agent handshake checklist

- [ ] Resolve hub by URL/id  
- [ ] READ Agent Contracts + Handoff Schema  
- [ ] Ensure Handoffs + Run Index exist (create under hub if missing)  
- [ ] WRITE Run Index row (`running`) + initial handshake handoff  
- [ ] Each pass: WRITE Handoff → next role READs same `run_id`  
- [ ] Final report: update Run Index (`done` / `partial`)  
- [ ] If Notion down: in-session only; **never** compensate with remote writes  
- [ ] Refuse Notion writes outside this hub  
