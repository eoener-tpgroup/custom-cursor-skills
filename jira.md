---
name: jira
description: Jira workflows via subcommands read|create|comment. Requires subcommand; ticket key for read/comment. Confirmation before writes via AskQuestion.
---

# Jira

Jira Service Desk / ticket workflows for project **PITOPS**.

Triggered manually via `/jira <subcommand> …`.

## Output language

**Always write all user-facing output in English** — status messages, summaries, drafts, error messages, and questions to the user. This applies even if the user writes in another language or module files use another language.

## ⚠️ Arguments: subcommand (required)

| # | Argument | Required | Values | Description |
|---|----------|----------|--------|-------------|
| 1 | **Subcommand** | yes | `read` \| `create` \| `comment` | Workflow to run |
| 2+ | **Rest** | per subcommand | see module | Ticket key, description, comment text, etc. |

```
❌ /jira
❌ /jira PITOPS-144486
❌ /jira update PITOPS-144486
✅ /jira read PITOPS-144486
✅ /jira create db01 disk alert — productive storage
✅ /jira comment PITOPS-144486 Investigation complete
```

### Missing or invalid subcommand

**Do not guess.** If subcommand is missing or not in the allowed set:

1. Prefer `AskQuestion` with options: `read`, `create`, `comment`
2. If unanswered → **abort** with:

> This command requires a subcommand.
> Usage: `/jira <read|create|comment> …`
> Examples: `/jira read PITOPS-144486` · `/jira create <description>` · `/jira comment PITOPS-144486 [text]`

## Module load (required — before Phase 1 of the subcommand)

Cursor does not inline nested files. **Read** these modules with the Read tool, resolving paths relative to this command file:
1. Prefer files in the **same directory** as this file: `_common.md`, then `<subcommand>.md` (skills layout: `…/skills/jira/SKILL.md` + modules beside it).
2. Else prefer the **sibling directory** named like this command: `jira/_common.md`, `jira/<subcommand>.md` (commands layout: `…/commands/jira.md` + `…/commands/jira/`).

| Order | File (commands layout) | File (skills layout) | Purpose |
|-------|------------------------|----------------------|---------|
| 1 | `jira/_common.md` | `_common.md` | Shared language, agency, security, auth, Wiki Markup |
| 2 | `jira/<subcommand>.md` | `<subcommand>.md` | Subcommand workflow (`read.md` / `create.md` / `comment.md`) |

**Then** execute only that subcommand’s workflow. Pass remaining prompt text (everything after the subcommand) as its arguments.

**Precedence:** This router’s abort rules and AskQuestion gates win on conflict. Subcommand modules win over `_common` for phase numbering and field-specific rules. Agency agents guide judgment only — they do not replace phases.

## Related commands

- `/merge-request create|review|fix …` — GitHub PR / GitLab MR workflows
- `/commit` — Conventional Commits
- `/remote-analyze` · `/postgres-optimize` — ops analysis feeding ticket context
