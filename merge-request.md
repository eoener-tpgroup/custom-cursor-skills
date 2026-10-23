---
name: merge-request
description: GitHub PR / GitLab MR workflows via subcommands create|review|fix. Requires subcommand. Confirmation before push/create/fixes via AskQuestion.
---

# Merge Request

Dual-platform GitHub Pull Request / GitLab Merge Request workflows.

Triggered manually via `/merge-request <subcommand> …`.

## Output language

**Always write all user-facing output in English** — status messages, summaries, MR/PR descriptions, review reports, error messages, and questions to the user. This applies even if the user writes in another language.

## ⚠️ Arguments: subcommand (required)

| # | Argument | Required | Values | Description |
|---|----------|----------|--------|-------------|
| 1 | **Subcommand** | yes | `create` \| `review` \| `fix` | Workflow to run |
| 2+ | **Rest** | per subcommand | see module | Instructions, MR/PR ID, focus, etc. |

```
❌ /merge-request
❌ /merge-request feat(auth): add JWT
❌ /merge-request update 42
✅ /merge-request create
✅ /merge-request create feat(auth): add JWT refresh — target develop
✅ /merge-request review !42 focus on auth
✅ /merge-request fix 42 only fix critical inline comments
```

### Missing or invalid subcommand

**Do not guess.** If subcommand is missing or not in the allowed set:

1. Prefer `AskQuestion` with options: `create`, `review`, `fix`
2. If unanswered → **abort** with:

> This command requires a subcommand.
> Usage: `/merge-request <create|review|fix> …`
> Examples: `/merge-request create [instructions]` · `/merge-request review [id] [focus]` · `/merge-request fix [id] [instructions]`

## Module load (required — before Phase 0/1 of the subcommand)

Cursor does not inline nested files. **Read** these modules with the Read tool, resolving paths relative to this command file:
1. Prefer files in the **same directory** as this file: `_common.md`, then `<subcommand>.md` (skills layout: `…/skills/merge-request/SKILL.md` + modules beside it).
2. Else prefer the **sibling directory** named like this command: `merge-request/_common.md`, `merge-request/<subcommand>.md` (commands layout: `…/commands/merge-request.md` + `…/commands/merge-request/`).

| Order | File (commands layout) | File (skills layout) | Purpose |
|-------|------------------------|----------------------|---------|
| 1 | `merge-request/_common.md` | `_common.md` | Shared language, platform detect, formatting, git safety |
| 2 | `merge-request/<subcommand>.md` | `<subcommand>.md` | Subcommand workflow (`create.md` / `review.md` / `fix.md`) |

**Then** execute only that subcommand’s workflow. Pass remaining prompt text (everything after the subcommand) as its arguments.

**Precedence:** This router’s abort rules and AskQuestion gates win on conflict. Subcommand modules win over `_common` for phase numbering and workflow-specific rules. Agency agents guide judgment only — they do not replace phases.

## Related commands

- `/commit` — Conventional Commits before create
- `/jira read|create|comment …` — ticket context
