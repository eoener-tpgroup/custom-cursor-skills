---
name: merge-request-fix
description: Wendet Fixes für offene Code-Review-Kommentare eines Merge Requests an. MR-ID optional. Bestätigung vor Fixes und vor Commit/Push via AskQuestion.
---

# Merge Request Fix

Liest **Code-Review-Kommentare** eines Merge/Pull Requests anhand der MR-ID, setzt **gültige, umsetzbare Änderungswünsche** im lokalen Code um und erstellt eine **Zusammenfassung**: was gefordert wurde vs. was gefixt wurde.

Wird manuell via `/merge-request-fix [id] [instructions]` ausgelöst.

## ⚙️ Argumente & Kontext

| # | Argument | Required | Beschreibung |
|---|----------|----------|--------------|
| 1 | **MR-ID** | bedingt | Numerische IID/PR-Nummer (`42`, `!42`, `#42`) |
| 2 | **Instructions** | nein | Freitext — welche Kommentare fixen, wie vorgehen, was ignorieren |

Alles nach der MR-ID (falls vorhanden) ist optionaler **Instructions**-Text und kann Leerzeichen enthalten.

**MR-ID ermitteln** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Explizite ID** im User-Prompt | Erstes gültiges Token nach `/merge-request-fix` (`42`, `!42`, `#42`) |
| 2 | **Chat-Kontext** | MR-URL, `!42`, `MR 42`, `PR #42` in der Konversation |
| 3 | **Aktueller Branch** | Offenen MR/PR für den lokalen Branch per `gh`/`glab`/GitLab MCP suchen |

**Instructions** (explizit oder aus Kontext):

- Welche Kommentare/Threads adressieren (`only inline comments on api.ts`, `fix critical and major only`)
- Was **nicht** umsetzen (`skip the config restructure suggestion`)
- Vorgehen (`add tests for edge case X`, `reply to @reviewer in MR comment`)
- Bezug auf vorherige Review-Diskussion im Chat

**Aus Kontext ableiten — Regeln:**

1. Instructions aus Chat übernehmen, wenn im Prompt kein Freitext nach der ID steht
2. **Nicht raten** — wenn keine ID ermittelbar und kein eindeutiger MR im Kontext → abbrechen oder User fragen
3. Wenn mehrere MRs möglich sind → **User fragen**

```
✅ /merge-request-fix 42
✅ /merge-request-fix !42 only fix null-check comments, skip architecture feedback
✅ /merge-request-fix address @reviewer's SQL injection comment
✅ /merge-request-fix                    → ID aus Branch oder Chat-Kontext
❌ /merge-request-fix fix                → kein gültiges ID-Token, kein Kontext → abbrechen
```

### Fehlende oder ungültige MR-ID

Wenn weder Argument noch Kontext noch Branch-Suche eine eindeutige MR-ID liefern:

1. **Prefer `AskQuestion`**: MR-ID oder Branch/URL erfragen
2. Falls unbeantwortet → **abbrechen** mit:

> This command requires a merge request ID (or enough context to identify one).
> Usage: `/merge-request-fix [id] [instructions]`
> Example: `/merge-request-fix 42 only fix critical inline comments`

Die MR-ID steht im User-Prompt **nach dem Command-Namen** (z.B. `/merge-request-fix 42`). Optionale Instructions folgen danach.

---

## Output language

**Always write all user-facing output in English** — status messages, fix summaries, error messages, questions to the user, and MR/PR comments. This applies even if the user writes in another language or this command definition is in German.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 2), format MR/PR reply comments for **that platform only** — whether posting via **`gh`**, **`glab`**, **GitLab MCP**, or **GitHub/GitLab REST API**.

| Platform | Comment dialect | Do not use |
|----------|-----------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only `!MR` shorthand as primary link style, Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax, Jira Wiki/ADF |

Never post Jira markup into MR/PR comments. Never use raw HTML for structure.

### Comment body rules

| Feld | Regeln |
|------|--------|
| **Kommentar-Body** | Platform Flavored Markdown: `## Review feedback addressed`, one bullet per fix, `@mention` reviewers when useful |
| **Struktur** | Short intro → addressed items → still-open items (e.g. **Still open (non-blocking):**) |
| **Dateireferenzen** | `` `path/to/file.ts` `` or `file.ts:42` — match existing review comment style on that platform |
| **Status** | Per item: fixed / skipped / why |
| **Cross-refs** | GitHub: `#123`, commit SHA · GitLab: `#123`, `!42` when linking MRs |

Prefer the **common Markdown subset** (headings, bullets, code fences, links, `@mentions`) so rendering is reliable on both platforms.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq`, HEREDOC, `--body-file`, or MCP native JSON args
2. **Preserve newlines** — lists/paragraphs need real newlines (`\n` in JSON)
3. **Special characters** — code snippets and paths via proper JSON encoding
4. **No HTML** — Markdown instead of `<br>` / `<ul>`
5. **Preview** — Phase 5 and Phase 8 show the MR comment as platform Markdown, not escaped JSON
6. **MCP schema first** — put the Markdown string in the tool’s `body` / `note` field as required by the schema

**GitHub (`gh` or REST) — GFM:**

```bash
gh pr comment <id> --body-file /tmp/pr-reply.md
```

**GitLab (`glab`, MCP, or REST) — GitLab Flavored Markdown:**

```bash
glab mr note <id> -m "$(cat <<'EOF'
## Review feedback addressed
...
EOF
)"
```

- MCP / REST: `body` = platform Markdown string, JSON-encoded via tool args or `json.dumps`

### Pre-post checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Each addressed review point is its own bullet
- [ ] Open / not-implemented items called out explicitly
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Workflow

### Phase 1: Argumente validieren & MR-ID ermitteln

1. User-Prompt nach `/merge-request-fix` parsen
2. Erstes Token prüfen: numerische IID/PR-Nummer (`42`, `!42`, `#42`) → **MR-ID**
3. Restlicher Text → **Instructions** (Scope, Prioritäten, Ausschlüsse)
4. Falls kein gültiges ID-Token: **Chat-Kontext** und **aktuellen Branch** prüfen (siehe oben)
5. Instructions aus Kontext ergänzen, wenn im Prompt kein Freitext nach der ID steht
6. Wenn MR-ID weiterhin unklar → `AskQuestion` oder abbrechen (siehe oben)
7. Projekt-Pfad aus Remote-URL oder User-Kontext ableiten

### Phase 2: Plattform erkennen und MR-Daten laden

**Order is mandatory:** resolve the remote URL **before** choosing a client or loading MR/PR data.

1. Read `git remote get-url origin` (fallback: recognizable primary remote from `git remote -v`).
2. Classify from the URL only:

| Plattform | Erkennung (URL only) | Client danach |
|-----------|----------------------|---------------|
| **GitHub** | URL enthält `github.com`, oder Hostname enthält `github` (z.B. `github.mycompany.com`) | `gh` CLI — danach `gh auth status` prüfen |
| **GitLab** | Hostname enthält `gitlab` (inkl. Self-Hosted, z.B. `gitlab.office.transporeon.com`) | `glab` **oder** GitLab MCP (siehe Client-Wahl unten) |
| **Unbekannt** | Keine der Regeln trifft | **Nicht raten** — `AskQuestion` (GitHub vs GitLab) oder abbrechen. Kein Probe von `gh`/`glab`/MCP zur Erkennung. Kein „push + manuelle URL"-Pfad |

**Wichtig:** Verfügbarkeit von `gh`, `glab` oder GitLab MCP **darf die Plattform nicht überschreiben**. Erst URL → Plattform, dann Client.

**GitLab Client-Wahl:** Was verfügbar und authentifiziert ist. Wenn **beides** geht → **`glab` bevorzugen**; sonst GitLab MCP. Beide nicht nötig.

**Nach Erkennung:** Fix plan, summary, commit footer, and reply drafts use the detected platform only — GitHub → `PR #<id>`, `#n`, GFM; GitLab → `MR !<id>`, `!n`, GitLab Flavored Markdown (Phases 5, 7, 8). See **Content formatting — GitLab or GitHub**.

Then load data via the matching subsection below (**GitLab** or **GitHub**).

#### GitLab (via `glab` or MCP)

**Client:** Prefer **`glab`** when both `glab` and GitLab MCP are available and authenticated; otherwise GitLab MCP. Tool discovery for MCP: `GetDynamicTools`, Namespace `GitLab`. Bei `needsAuth`: User bitten zu authentifizieren, **nicht** ohne Auth fortfahren.

**Via `glab` — Daten laden:**

```bash
glab mr view <id>
glab mr diff <id>
glab mr view <id> --comments
# Resolve project path, then load discussions (URL-encode namespace/project)
PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)
glab api "projects/${PROJECT}/merge_requests/<id>/discussions"
```

**Via MCP — Daten laden:**

1. Tool-Schema via `GetDynamicTools` prüfen (z.B. `get_merge_request`, `get_merge_request_notes`, `list_merge_request_discussions`)
2. Entsprechende MCP-Tools aufrufen (gleiche Datentypen wie unten)

| Daten | Zweck |
|-------|-------|
| MR-Details | Titel, Beschreibung, Autor, Status, Source/Target-Branch |
| MR-Diffs | Kontext für Inline-Kommentare und betroffene Dateien |
| MR-Commits | Commit-Historie |
| MR-Diskussionen / Notes | **Alle** Review-Kommentare, Threads, Inline-Notes |
| MR-Approvals | Review-States (`CHANGES_REQUESTED`, etc.) |

**GitLab — Kommentare laden (Pflicht):**

Via **`glab`** (wenn als Client gewählt):

1. `glab mr view <id> --comments` — schneller Überblick über Notes
2. Resolve/encode project path, then `glab api "projects/${PROJECT}/merge_requests/<id>/discussions"` — vollständige Threads inkl. resolved/unresolved und Inline-Positionen (`PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)`)

Via **GitLab MCP** (wenn als Client gewählt):

1. Tool-Schema via `GetDynamicTools` prüfen (z.B. `list_merge_request_discussions`, `get_merge_request_notes`, `get_merge_request`)
2. **Alle** Diskussionen und Notes laden — inkl.:
   - Allgemeine MR-Kommentare (Discussion-Notes)
   - Inline-Kommentare an Code-Zeilen (Diff-Notes)
   - Antworten in Threads (vollständige Konversation)
   - Review-Notes mit `CHANGES_REQUESTED`-Status
3. Pro Kommentar erfassen: Autor, Datum, Datei/Zeile (falls inline), Inhalt, resolved/unresolved, Thread-ID

**GitLab — Kommentar posten (Phase 8, nach Bestätigung):**

- `glab mr note <id> -m "..."` or MCP (`create_workitem_note` or equivalent — schema first)
- Body = **GitLab Flavored Markdown** (see **Content formatting — GitLab or GitHub**)

#### GitHub (via `gh`)

```bash
gh pr view <id> --json title,body,state,author,baseRefName,headRefName,files,reviews,reviewDecision
gh pr diff <id>
gh api repos/{owner}/{repo}/pulls/<id>/comments
gh api repos/{owner}/{repo}/issues/<id>/comments
gh api graphql -f query='...'  # für unresolved review threads, falls nötig
```

**GitHub — Kommentare laden (Pflicht):**

- Issue-Kommentare (allgemeine PR-Diskussion)
- Review-Kommentare (`pulls/{id}/comments` — inline an Code-Zeilen)
- Review-Submissions (`CHANGES_REQUESTED` + Body)
- Threads und Antworten vollständig einbeziehen
- Resolved/unresolved-Status berücksichtigen

**GitHub — Kommentar posten (Phase 8, nach Bestätigung):**

- `gh pr comment <id> --body-file /tmp/pr-reply.md` or `--body` with HEREDOC
- Body = **GitHub GFM** (see **Content formatting — GitLab or GitHub**)
- **Resolve implemented review threads (Pflicht, wenn vorhanden):** After posting the reply (or as part of the same Option-4 path), mark every **unresolved** review thread whose finding was **✅ Fixed** in this run as resolved. Do **not** resolve threads that remain open / skipped / not implemented.

```bash
# List unresolved review threads on the PR
gh api graphql -f query='
query($owner:String!,$name:String!,$number:Int!) {
  repository(owner:$owner, name:$name) {
    pullRequest(number:$number) {
      reviewThreads(first: 100) {
        nodes { id isResolved path comments(first: 1) { nodes { body } } }
      }
    }
  }
}' -F owner=OWNER -F name=REPO -F number=<id>

# Resolve one thread (repeat for each fixed finding)
gh api graphql -f query='
mutation($id:ID!) {
  resolveReviewThread(input:{threadId:$id}) { thread { id isResolved } }
}' -f id='PRRT_...'
```

### Phase 3: Lokales Repository vorbereiten

1. Prüfen, ob aktuelles Verzeichnis ein Git-Repository ist
2. MR-Source-Branch mit lokalem Branch abgleichen:

```bash
git status
git branch -vv
git fetch origin
```

3. **Branch-Wechsel** falls nötig:

   - Wenn lokaler Branch ≠ MR-Source-Branch → auf MR-Source-Branch wechseln (`git checkout <source-branch>`)
   - Branch fehlt lokal → `git fetch origin <source-branch>:<source-branch>` oder `git checkout -b <source-branch> origin/<source-branch>`

4. Uncommitted changes prüfen:
   - Bei uncommitted changes: User informieren und **nicht** überschreiben — stashen oder committen lassen
5. Optional: `git pull --rebase origin <source-branch>` um aktuell zu sein (kein Force)

### Phase 4: Review-Kommentare triagieren

**Instructions aus Phase 1 beachten:** Nur Kommentare bearbeiten, die in den Instructions genannt sind oder dem Scope entsprechen. Explizit ausgeschlossene Punkte in der Summary als „nicht umgesetzt (per Anweisung)" markieren.

**Nur umsetzbare, offene Review-Punkte** bearbeiten. Kommentare klassifizieren:

| Kategorie | Aktion |
|-----------|--------|
| ✅ **Actionable** | Konkreter Änderungswunsch, Bug, fehlender Test, Naming-Fix | → Fix planen |
| ⏭️ **Resolved** | Bereits als resolved markiert oder Autor hat bestätigt | → Überspringen |
| 💬 **Question** | Rückfrage ohne klaren Fix | → Überspringen, in Summary als „offen" |
| ❌ **Invalid / Out of scope** | Feedback widerspricht Anforderung oder ist fachlich falsch | → Nicht fixen, in Summary begründen |
| 🤖 **Bot (Bugbot etc.)** | Automatischer Review-Kommentar | → Inhalt prüfen, nur bei **validem** Finding fixen |
| 📝 **Nur Review-Bericht** | Allgemeiner Review ohne konkreten Fix (z.B. `/merge-request-review`-Output) | → Einzelne Findings extrahieren und als Actionable behandeln |

**Priorität:**

0. **Instructions/Kontext** — vom User genannte Kommentare, Dateien oder Severities zuerst
1. Unresolved Inline-Kommentare an Code-Zeilen
2. `CHANGES_REQUESTED`-Reviews
3. Allgemeine MR-Kommentare mit konkretem Änderungswunsch
4. Findings aus automatisierten Review-Kommentaren (Critical/Major zuerst)

**Duplikate zusammenführen:** Mehrere Kommentare zum gleichen Problem → ein Fix, mehrere Referenzen in der Summary.

**Noch keine Code-Änderungen** — nur den Fix-Plan für Phase 5 festhalten.

### Phase 5: Fix-Plan zeigen und bestätigen lassen (Pflicht)

**Vor jeder Code-Änderung** den vollständigen Fix-Plan und alle generierten Texte zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch Fixes anwenden** — erst nach expliziter Bestätigung.

**Labels (strict — match detected platform):**

- **GitHub:** `PR #<id>` in the fix plan table and summary; commit footer `address PR #<id> review comments`
- **GitLab:** `MR !<id>`; commit footer `address MR !<id> review comments`

Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
## Fix plan

| Field | Value |
|-------|-------|
| PR | #<id> — <title> |
| Branch | `<source>` → `<target>` |
| Actionable comments | <n> |
| To fix | <n> |
| To skip | <n> |
| Open / manual | <n> |

### Planned fixes
| # | Source | File | Requirement | Planned change |
|---|--------|------|-------------|----------------|
| 1 | @reviewer, `file.ts:42` | `file.ts` | Null-check missing | Add guard in `handleRequest()` |
| 2 | ... | ... | ... | ... |

### Skipped / not implemented
| # | Source | Reason |
|---|--------|--------|
| 3 | @reviewer | Already resolved |
| 4 | @reviewer | Architecture decision — needs discussion |

### Draft commit message
```
fix(review): address PR #<id> review comments

- <change 1>
- <change 2>
```

### Draft PR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro if replying to a specific thread>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Actions
1. Apply listed code changes locally
2. Produce fix summary report
3. *(Later, separate confirmation)* Commit / push / post PR comment
```

```markdown
<!-- GitLab skeleton (MR / !) -->
## Fix plan

| Field | Value |
|-------|-------|
| MR | !<id> — <title> |
| Branch | `<source>` → `<target>` |
| Actionable comments | <n> |
| To fix | <n> |
| To skip | <n> |
| Open / manual | <n> |

### Planned fixes
| # | Source | File | Requirement | Planned change |
|---|--------|------|-------------|----------------|
| 1 | @reviewer, `file.ts:42` | `file.ts` | Null-check missing | Add guard in `handleRequest()` |
| 2 | ... | ... | ... | ... |

### Skipped / not implemented
| # | Source | Reason |
|---|--------|--------|
| 3 | @reviewer | Already resolved |
| 4 | @reviewer | Architecture decision — needs discussion |

### Draft commit message
```
fix(review): address MR !<id> review comments

- <change 1>
- <change 2>
```

### Draft MR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro if replying to a specific thread>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Actions
1. Apply listed code changes locally
2. Produce fix summary report
3. *(Later, separate confirmation)* Commit / push / post MR comment
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> Proceed with applying these fixes?

Optionen:

1. **Yes, apply fixes** — Geplante Änderungen im Code umsetzen
2. **Edit plan first** — User passt Scope, einzelne Fixes oder Texte an; Plan aktualisieren, erneut bestätigen
3. **Cancel** — Keine Code-Änderungen vornehmen

Bei **Cancel**: Kurz bestätigen, dass keine Fixes angewendet wurden.

### Phase 6: Fixes anwenden (nur nach Bestätigung)

Für jeden **actionable** Punkt:

1. **Betroffene Datei(en) lesen** — nicht nur Diff-Hunk, sondern umgebenden Kontext
2. **Projekt-Konventionen** beachten — `.cursor/rules/`, `CONTRIBUTING.md`, bestehender Stil
3. **Minimaler, fokussierter Fix** — nur das Nötige ändern, kein Refactoring „nebenbei"
4. **Scope einhalten** — keine CI-Workflows ändern, nur um grüne Checks zu erzwingen; keine unrelated Changes
5. Nach jedem Fix kurz verifizieren (Syntax, offensichtliche Logik)

**Nicht automatisch fixen ohne klare Grundlage:**

- Architektur-Entscheidungen, die Diskussion brauchen
- Widersprüchliches Feedback zwischen Reviewern
- Breaking Changes ohne explizite Anforderung im Kommentar
- Security-relevante Änderungen, die Design-Entscheidung erfordern → als „manuell klären" markieren

**Tests:** Wenn ein Kommentar fehlende Tests fordert und der Fix klar ist → Tests ergänzen.

### Phase 7: Zusammenfassung erstellen

Am Ende **immer** folgenden Bericht ausgeben (auch wenn nichts gefixt wurde):

**Chat title (strict — match detected platform):**

- **GitHub:** `# Fix Summary: PR #<id> — <title>`
- **GitLab:** `# Fix Summary: MR !<id> — <title>`

Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
# Fix Summary: PR #<id> — <title>

## Overview
| Metric | Value |
|--------|-------|
| PR | #<id> — <title> |
| Branch | `<source>` → `<target>` |
| Comments total | <n> |
| Actionable | <n> |
| Fixed | <n> |
| Skipped | <n> |
| Open / manual | <n> |

## Requested → Fixed

| # | Source | File | Requirement | Status | Implementation |
|---|--------|------|-------------|--------|----------------|
| 1 | @reviewer, inline `file.ts:42` | `file.ts` | Missing null-check | ✅ Fixed | Guard added in `handleRequest()` |
| 2 | ... | ... | ... | ... | ... |

**Status legend:** ✅ Fixed · ⏭️ Skipped (resolved/duplicate) · ❌ Not implemented (with reason) · 💬 Open (needs clarification)

## Changed files
- `path/to/file.ts` — <short description>
- ...

## Not implemented (reason)
1. ...

## Next steps
1. Review changes (`git diff`)
2. Commit with `/commit` or manually
3. Push and update PR (`/merge-request` or `git push`)
4. Optional: Reply on PR; on GitHub, resolve review threads for ✅ Fixed findings
```

```markdown
<!-- GitLab skeleton (MR / !) -->
# Fix Summary: MR !<id> — <title>

## Overview
| Metric | Value |
|--------|-------|
| MR | !<id> — <title> |
| Branch | `<source>` → `<target>` |
| Comments total | <n> |
| Actionable | <n> |
| Fixed | <n> |
| Skipped | <n> |
| Open / manual | <n> |

## Requested → Fixed

| # | Source | File | Requirement | Status | Implementation |
|---|--------|------|-------------|--------|----------------|
| 1 | @reviewer, inline `file.ts:42` | `file.ts` | Missing null-check | ✅ Fixed | Guard added in `handleRequest()` |
| 2 | ... | ... | ... | ... | ... |

**Status legend:** ✅ Fixed · ⏭️ Skipped (resolved/duplicate) · ❌ Not implemented (with reason) · 💬 Open (needs clarification)

## Changed files
- `path/to/file.ts` — <short description>
- ...

## Not implemented (reason)
1. ...

## Next steps
1. Review changes (`git diff`)
2. Commit with `/commit` or manually
3. Push and update MR (`/merge-request` or `git push`)
4. Optional: Reply on MR / resolve discussion threads
```

### Phase 8: Nächste Schritte — Texte zeigen und bestätigen lassen (Pflicht)

**Nach der Summary** die geplanten **Commit-Message** und **MR-Kommentar** nochmals vollständig zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch committen, pushen oder MR/PR-Kommentare posten** — erst nach expliziter Bestätigung.

Use **PR #<id>** / PR wording on GitHub; **MR !<id>** / MR wording on GitLab. Output **exactly one** skeleton below.

```markdown
<!-- GitHub skeleton (PR / #) -->
## Next steps plan

### Commit message
```
fix(review): address PR #<id> review comments

- <change 1>
- <change 2>
```

### PR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Changed files
- `path/to/file.ts` — <description>
```

```markdown
<!-- GitLab skeleton (MR / !) -->
## Next steps plan

### Commit message
```
fix(review): address MR !<id> review comments

- <change 1>
- <change 2>
```

### MR reply comment
```markdown
## Review feedback addressed

@reviewer <brief intro>

- <change 1>
- <change 2>

**Still open (non-blocking):**
- <item> — <reason>
```

### Changed files
- `path/to/file.ts` — <description>
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> How should the fixes be handled next?

Optionen:

1. **Summary only** — Fixes bleiben lokal uncommitted, nichts posten
2. **Commit** — Commit mit obiger Message erstellen
3. **Commit + push** — Commit und `git push` auf PR/MR-Branch
4. **Commit + push + PR/MR comment** — Zusätzlich obigen PR/MR-Kommentar posten; **auf GitHub** danach umgesetzte Review-Threads resolven (siehe unten)
5. **Edit texts first** — User passt Commit-Message oder PR/MR-Kommentar an; erneut bestätigen

Bei **Commit** / **Commit + push** / **Commit + push + PR/MR comment**:

- Git Safety Protocol einhalten (siehe `/commit`)
- Commit-Message referenziert MR-ID und kurz die Hauptfixes
- Keine Secrets committen

Bei **MR/PR-Kommentar posten** (Option 4 — Text aus Entwurf oben, English, no automation markers; **Content formatting — GitLab or GitHub** beachten — MCP und REST/CLI):
- **GitLab**: `glab mr note <id> -m "..."` or MCP note tool — body = **GitLab Flavored Markdown**
- **GitHub**: `gh pr comment <id> --body-file /tmp/pr-reply.md` or `--body` with HEREDOC — body = **GitHub GFM**
- **GitHub — resolve threads:** Wenn Review-Threads zu den **✅ Fixed**-Findings noch unresolved sind → per GraphQL `resolveReviewThread` resolven (see **GitHub — Kommentar posten**). Skip threads for skipped / open / not-implemented items. Report how many threads were resolved.

Bei Erfolg: **URL des Kommentars** zurückgeben (und kurz: `Resolved <n> review thread(s)` on GitHub when applicable).

Bei **Summary only** / **Cancel**: Kurz bestätigen, dass nichts committed, gepusht oder gepostet wurde.

---

## Fix-Grundsätze

1. **Review-Kommentare sind die Quelle der Wahrheit** — nur fixen, was explizit oder klar implizit gefordert ist
2. **Minimaler Scope** — kleinster korrekter Fix pro Punkt
3. **Validieren statt blind folgen** — falsches oder widersprüchliches Feedback nicht umsetzen, sondern begründen
4. **Keine Duplikate** — resolved/already-addressed Kommentare nicht erneut bearbeiten
5. **Kontext lesen** — Thread-Verlauf und Autor-Antworten berücksichtigen
6. **Transparenz** — Summary muss jeden Review-Punkt mit Status abbilden
7. **GitHub: resolve when fixed** — After Option 4 (PR comment), resolve review threads for findings that were implemented; leave others open

## Git Safety Protocol (strikt einhalten)

- **Niemals** `git config` ändern
- **Niemals** destruktive Git-Befehle (`push --force`, `reset --hard`, etc.) ohne explizite User-Anfrage
- **Niemals** Hooks überspringen (`--no-verify`, `--no-gpg-sign`, etc.)
- **Niemals** Force-Push auf `main`/`master`
- **Keine** Secrets committen

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Keine MR-ID ermittelbar | AskQuestion oder abbrechen (siehe oben) |
| Mehrere MRs im Kontext | User fragen, welcher MR/PR gemeint ist |
| MR/PR nicht gefunden | ID und Projekt prüfen |
| Remote-Host unbekannt | AskQuestion oder abbrechen — nicht raten. Kein „push + manuelle URL"-Pfad |
| GitHub ohne `gh` Auth | Abbrechen, `gh auth login` anleiten |
| GitLab ohne `glab` und ohne MCP | Abbrechen, CLI- oder MCP-Setup anleiten |
| MCP nicht authentifiziert | Auth anleiten, abbrechen |
| Kein lokales Repo / falscher Branch | User informieren, Branch-Checkout anbieten |
| Keine offenen Review-Kommentare | Melden und Summary mit 0 actionable |
| Widersprüchliches Feedback | Nicht fixen, in Summary als „manuell klären" |
| Fix würde CI-Workflow ändern | Nicht automatisch — User informieren |
| User bricht Bestätigung ab | Keine Fixes anwenden bzw. nichts committen/pushen/posten |

## Verwandte Commands

- `/merge-request-review [id] [focus]` — Intensive Code Review (Findings erzeugen)
- `/commit` — Änderungen committen
- `/merge-request` — Branch pushen und MR erstellen/aktualisieren
