---
name: merge-request
description: Pusht den aktuellen Branch und erstellt einen Merge Request (GitLab) oder Pull Request (GitHub). Optionale Anweisungen oder Chat-Kontext. Bestätigung vor Push/MR via AskQuestion.
---

# Merge Request

Pusht den aktuellen Feature-Branch und erstellt einen Merge/Pull Request gegen den Default-Branch.
Wird manuell via `/merge-request [instructions]` ausgelöst.

## ⚙️ Optionale Argumente & Kontext

Alles **nach** `/merge-request` im User-Prompt ist optionaler Freitext (`[instructions]`). Er kann Leerzeichen enthalten.

**Zwei Quellen** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Expliziter Text** nach `/merge-request` | User hat Anweisungen direkt angegeben |
| 2 | **Chat-Kontext** | Kein expliziter Text — aus bisheriger Konversation ableiten |

**Typische Anweisungen** (explizit oder aus Kontext):

- MR/PR-Titel oder Summary-Hinweise
- Ziel-Branch (`target develop`, `base main`)
- Testplan-Schwerpunkte oder Breaking-Change-Hinweise
- Labels, Reviewer, Draft-Status (wenn Plattform/CLI es unterstützt)
- „draft MR" / „ready for review"

**Aus Kontext ableiten — Regeln:**

1. Titel und Summary aus Commits, Diff und Chat ableiten — besprochene Änderungen und Ticket-Keys einbeziehen
2. Keine fiktiven Testschritte erfinden — aus Diff und Kontext ableiten oder generisch halten
3. Wenn Target-Branch unklar und nicht Default → **User fragen**

```
✅ /merge-request
✅ /merge-request feat(auth): add JWT refresh — target develop
✅ /merge-request draft, focus on migration rollback test plan
✅ /merge-request breaking change: remove v1 API endpoints
```

## Output language

**Always write all user-facing output in English** — status messages, summaries, MR/PR descriptions, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 3), format title and description for **that platform only** — whether creating via **`gh`**, **`glab`**, **GitLab MCP**, **GitHub/GitLab REST API**, or any other client.

| Platform | Description dialect | Do not use |
|----------|---------------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only refs (`!42` as MR shorthand in body when linking PRs), Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax / PR conventions that GitLab ignores, Jira Wiki/ADF |

Never post Jira markup into an MR/PR. Never mix HTML into the body.

### Shared field rules

| Feld | Regeln |
|------|--------|
| **Title** | Plain text — **no** Markdown (`#`, `**`, lists). Short, imperative, max. ~72 chars when possible. No newlines |
| **Description / Body** | Platform Flavored Markdown: `##` headings, `-` lists, `- [ ]` checkboxes in Test plan, `**bold**`, `` `inline` ``, fenced code with language tag |
| **Struktur** | Blank line between sections; fixed order: Summary → Changes → Test plan |
| **Links** | Full URLs (`https://…`); platform-native issue/MR links when appropriate |
| **Tabellen** | Only if needed; simple GFM/GitLab Markdown tables |

### Platform-specific Markdown

| Feature | GitHub (GFM) | GitLab Flavored Markdown |
|---------|--------------|--------------------------|
| Headings / lists / code | Standard GFM | Same common subset — prefer this |
| Task lists | `- [ ]` / `- [x]` | `- [ ]` / `- [x]` |
| Mentions | `@user` | `@user` / `@group` |
| Cross-refs | `#123` (issue/PR), commit SHAs | `#123` (issue), `!42` (MR), `$pipeline_id` where useful |
| Collapsed detail | `<details>` sparingly if needed | Same; prefer open sections for reviewability |
| Alerts / admonitions | Avoid GitHub alert blockquotes unless sure | Prefer plain `##` sections |

When unsure, stick to the **common subset** (headings, bullets, checkboxes, fenced code, links, `@mentions`) — it renders on both.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq -n --arg`, HEREDOC, or `--body-file` / MCP native JSON args
2. **Preserve newlines** — `\n` in JSON strings; real newlines in HEREDOC (do not squash the body onto one line)
3. **Special characters** — `"`, `\`, backticks via proper encoding
4. **No HTML** as structure (`<br>`, `<p>`, `<ul>`) — Markdown only
5. **Preview** — Phase 5 shows title + full description as rendered Markdown for the detected platform, not escaped JSON
6. **MCP schema first** — map draft fields to the tool’s parameters (`title`/`description` or equivalents); body remains platform Markdown

**GitHub (`gh` or REST):**

```bash
gh pr create --base <target> --title "<plain title>" --body-file /tmp/pr-body.md
# or --body "$(cat <<'EOF'
## Summary
...
EOF
)"
```

**GitLab (`glab`, MCP, or REST):**

```bash
glab mr create --target-branch <target> --title "<plain title>" --description "$(cat <<'EOF'
## Summary
...
EOF
)"
```

- MCP / REST: `title` = plain text (JSON-encoded); `description` = **GitLab Markdown** string (JSON-encoded)

### Pre-create checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Title plain text, no markup, no newlines
- [ ] Description uses `##` sections and blank lines
- [ ] Test plan is a checkbox list (`- [ ]`)
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Voraussetzungen

- Git-Repository mit konfiguriertem Remote (`origin` oder erkennbarer Primary-Remote)
- Aktueller Branch ist **nicht** der Default-Branch
- Es gibt mindestens einen Commit auf dem Branch, der noch nicht auf dem Default-Branch ist
- Branch wurde vorher committed (→ `/commit` wenn nötig)

## Git Safety Protocol (strikt einhalten)

- **Niemals** `git config` ändern
- **Niemals** Force-Push auf `main`/`master` — User warnen
- **Niemals** `--force` ohne explizite User-Anfrage
- **Niemals** Hooks überspringen

---

## Workflow

### Phase 0: Anweisungen & Kontext auswerten

1. Text nach `/merge-request` extrahieren — falls vorhanden, als **explizite Anweisungen** behandeln
2. Falls leer: **Chat-Kontext** auf MR-relevante Hinweise prüfen (Feature-Beschreibung, Ticket-Keys, Testhinweise, Breaking Changes)
3. Anweisungen in konkrete Entscheidungen übersetzen:
   - MR/PR-Titel
   - Summary / Changes / Test plan Inhalte
   - Target-Branch (falls abweichend vom Default)
   - Draft vs. ready
4. Unklare oder widersprüchliche Anweisungen → **User fragen** vor dem Erstellen

### Phase 1: Repository-Status prüfen

Führe **parallel** aus:

```bash
git status
git branch -vv
git remote -v
git log --oneline -10
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null || true
```

**Default-Branch ermitteln** (wie bei `/commit`):

1. `git symbolic-ref refs/remotes/origin/HEAD`
2. Fallback: `main` → `master` → `develop`

**Target-Branch**: Aus Anweisungen/Kontext (`target <branch>`, `base <branch>`). Falls nicht angegeben → Default-Branch.

### Phase 2: Vorbedingungen validieren

| Check | Bei Fehler |
|-------|------------|
| Auf Default-Branch? | Abbrechen: „Erst Feature-Branch erstellen mit `/commit`" |
| Uncommitted changes? | User fragen: committen, stashen oder abbrechen |
| Kein Remote? | Abbrechen, Remote konfigurieren lassen |
| Branch hat keine Commits vs. Default? | Abbrechen: „Keine Änderungen zum Pushen" |

Optional:

```bash
git fetch origin
git log origin/<default-branch>..HEAD --oneline
```

### Phase 3: Remote-Plattform erkennen

| Plattform | Erkennung | CLI / Tool |
|-----------|-----------|------------|
| **GitHub** | `gh` verfügbar + `gh auth status` OK, oder Remote-URL enthält `github.com` | `gh` CLI |
| **GitLab** | Remote-URL enthält `gitlab`, oder GitLab MCP authentifiziert | `glab` CLI oder GitLab MCP |
| **Unbekannt** | — | Generic `git push` + manuelle MR/PR-URL an User |

Priorität: `gh` für GitHub, `glab` für GitLab, GitLab MCP als Fallback.

**Nach Erkennung:** Alle Title-/Description-Texte ausschließlich im Dialekt der erkannten Plattform schreiben (siehe **Content formatting — GitLab or GitHub**). Bei unbekanntem Remote: gemeinsames Markdown-Subset + manuelle MR-URL — kein Jira-Markup.

### Phase 4: MR/PR-Entwurf erstellen

Commits und Diff analysieren, MR/PR-Inhalt entwerfen — **noch kein Push, noch kein MR/PR erstellen**.

```bash
git log origin/<target-branch>..HEAD --oneline
git diff origin/<target-branch>..HEAD --stat
```

**Entwurf festhalten:**

- Titel (Conventional-Commits-Stil; Anweisungen/Kontext haben Vorrang)
- Vollständige Beschreibung (Summary, Changes, Test plan, Breaking changes falls relevant)
- Target-Branch, Source-Branch, Plattform, Draft ja/nein

**MR/PR-Beschreibung** — Template:

```markdown
## Summary
<1-3 Bullet Points — was und warum>

## Changes
<Kurze Liste der wichtigsten Änderungen>

## Test plan
- [ ] <Testschritt 1>
- [ ] <Testschritt 2>
```

### Phase 5: Entwurf zeigen und bestätigen lassen (Pflicht)

**Vor `git push` und MR/PR-Erstellung** den vollständigen Plan zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch pushen oder MR/PR erstellen** — erst nach expliziter Bestätigung.

```markdown
## Merge request plan

| Field | Value |
|-------|-------|
| Platform | GitHub / GitLab |
| Source branch | `<current-branch>` |
| Target branch | `<target-branch>` |
| Commits | `<n>` |
| Changed files | `<n>` (+/- lines) |
| Draft | yes / no |

### Title
`<title>`

### Description
<full MR/PR body — Summary, Changes, Test plan>

### Actions
1. `git push -u origin HEAD`
2. Create MR/PR via `<gh|glab|MCP>`
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> Proceed with push and merge request creation?

Optionen:

1. **Yes, push and create MR** — Push und MR/PR mit obigem Titel und Beschreibung erstellen
2. **Edit first** — User gibt Korrekturen; Entwurf anpassen, erneut bestätigen
3. **Cancel** — Nichts pushen, keinen MR/PR erstellen

Bei **Cancel**: Kurz bestätigen, dass nichts gepusht oder erstellt wurde.

### Phase 6: Branch pushen (nur nach Bestätigung)

```bash
git push -u origin HEAD
```

Bei „upstream rejected" oder diverged history: **nicht** force-pushen — User informieren.

### Phase 7: Merge/Pull Request erstellen (nur nach Bestätigung)

**Formatierung:** Siehe **Content formatting — GitLab or GitHub** — Title plain text; Description als **GitHub GFM** oder **GitLab Flavored Markdown** je nach erkannter Plattform; gilt für `gh` / `glab` / **MCP** / REST.

#### GitHub (via `gh`)

```bash
gh pr create --base <target-branch> --head <current-branch> --title "<title>" --body "$(cat <<'EOF'
<description from Phase 4 draft>
EOF
)"
```

#### GitLab (via `glab`)

```bash
glab mr create --target-branch <target-branch> --title "<title>" --description "$(cat <<'EOF'
<description from Phase 4 draft>
EOF
)"
```

#### GitLab (via MCP, wenn `glab` nicht verfügbar)

1. GitLab MCP prüfen (`GetDynamicTools`, Namespace `GitLab`)
2. Bei `needsAuth`: User bitten zu authentifizieren
3. Verfügbare MCP-Tools für MR-Erstellung nutzen — `title` plain text, `description` als **GitLab Flavored Markdown** (siehe Content formatting)

#### Fallback

Nach Push Remote-URL und Branch ausgeben für manuelle MR-Erstellung.

### Phase 8: Ergebnis melden

- **URL** des erstellten MR/PR
- Branch-Name und Target-Branch
- Anzahl Commits und geänderte Dateien
- Pipeline/CI-Status, falls abrufbar
- Hinweis: `/merge-request-review [id] [focus]` für Code Review, `/merge-request-fix [id] [instructions]` für Review-Fixes

---

## PR/MR-Beschreibung — Best Practices

1. **Summary**: Was wurde geändert und warum
2. **Test plan**: Konkrete, abhakbare Schritte
3. **Breaking changes**: Explizit hervorheben
4. **Scope**: Kleine, fokussierte MRs bevorzugen

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Auf Default-Branch | Abbrechen, `/commit` empfehlen |
| Push rejected | Fehler zeigen, kein Force-Push |
| `gh`/`glab` nicht installiert | Push durchführen, manuelle MR-Erstellung verlinken |
| Auth fehlgeschlagen | User informieren |
| MR/PR existiert bereits | Bestehende URL zurückgeben |
| User bricht Bestätigung ab | Nichts pushen, keinen MR/PR erstellen |

## Verwandte Commands

- `/commit [instructions]` — Änderungen committen
- `/merge-request-review [id] [focus]` — Code Review
- `/merge-request-fix [id] [instructions]` — Review-Kommentare umsetzen
