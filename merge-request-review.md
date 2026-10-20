---
name: merge-request-review
description: Intensive Code Review eines Merge Requests inkl. bestehender MR-Kommentare. MR-ID optional (/merge-request-review [id] [focus]) — aus Argument, Branch oder Chat-Kontext.
---

# Merge Request Review

Analysiert einen Merge/Pull Request anhand der MR-ID und führt eine **intensive, strukturierte Code Review** durch.

Wird manuell via `/merge-request-review [id] [focus]` ausgelöst.

## ⚙️ Argumente & Kontext

| # | Argument | Required | Beschreibung |
|---|----------|----------|--------------|
| 1 | **MR-ID** | bedingt | Numerische IID/PR-Nummer (`42`, `!42`, `#42`) |
| 2 | **Focus** | nein | Freitext — Review-Schwerpunkt, Symptome, Dateien, Fragen |

Alles nach der MR-ID (falls vorhanden) ist optionaler **Focus** und kann Leerzeichen enthalten.

**MR-ID ermitteln** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Explizite ID** im User-Prompt | Erstes gültiges Token nach `/merge-request-review` (`42`, `!42`, `#42`) |
| 2 | **Chat-Kontext** | MR-URL, `!42`, `MR 42`, `PR #42` in der Konversation |
| 3 | **Aktueller Branch** | Offenen MR/PR für den lokalen Branch per `gh`/`glab`/GitLab MCP suchen |

**Focus** (explizit oder aus Kontext):

- Review-Schwerpunkte (`security`, `performance`, `only auth module`)
- Konkrete Fragen (`is the retry logic safe under load?`)
- Bereiche ignorieren (`skip generated files`, `ignore docs changes`)
- Bezug auf vorherige Diskussion im Chat

**Aus Kontext ableiten — Regeln:**

1. Focus aus Chat übernehmen, wenn im Prompt kein Freitext nach der ID steht
2. **Nicht raten** — wenn keine ID ermittelbar und kein eindeutiger MR im Kontext → abbrechen oder User fragen
3. Wenn mehrere MRs möglich sind (z.B. mehrere URLs im Chat) → **User fragen**

```
✅ /merge-request-review 42
✅ /merge-request-review !42 focus on SQL injection and auth bypass
✅ /merge-request-review security review for webhook handler
✅ /merge-request-review                    → ID aus Branch oder Chat-Kontext
❌ /merge-request-review review             → kein gültiges ID-Token, kein Kontext → abbrechen
```

### Fehlende oder ungültige MR-ID

Wenn weder Argument noch Kontext noch Branch-Suche eine eindeutige MR-ID liefern:

1. **Prefer `AskQuestion`**: MR-ID oder Branch/URL erfragen
2. Falls unbeantwortet → **abbrechen** mit:

> This command requires a merge request ID (or enough context to identify one).
> Usage: `/merge-request-review [id] [focus]`
> Example: `/merge-request-review 42 security and error handling`

Die MR-ID steht im User-Prompt **nach dem Command-Namen** (z.B. `/merge-request-review 42`). Optionaler Focus folgt danach.

---

## Output language

**Always write all user-facing output in English** — status messages, review reports, error messages, questions to the user, and MR/PR comments. This applies even if the user writes in another language or this command definition is in German.

## Content formatting — GitLab or GitHub (strict, MCP / CLI / REST)

**Mandatory for every write path.** After platform detection (Phase 2), format MR/PR review comments for **that platform only** — whether posting via **`gh`**, **`glab`**, **GitLab MCP**, or **GitHub/GitLab REST API**.

| Platform | Comment dialect | Do not use |
|----------|-----------------|------------|
| **GitHub** | **GitHub Flavored Markdown (GFM)** | GitLab-only `!MR` shorthand as primary link style, Jira Wiki/ADF |
| **GitLab** | **GitLab Flavored Markdown** | GitHub-only alert syntax, Jira Wiki/ADF |

Never post Jira markup into MR/PR comments. Never use raw HTML for structure. Chat review reports may use the same Markdown; when posting to the platform, the body MUST match the detected dialect.

### Comment body rules (review posts)

| Feld | Regeln |
|------|--------|
| **Kommentar-Body** | Platform Flavored Markdown: `##`-headings, numbered/bulleted lists, `**bold**` for severity/recommendation, `` `file.ts:42` `` for file refs |
| **Struktur** | Executive Summary + recommendation first → Findings by severity → Positives → Test plan |
| **Severity** | Icons or **Bold** labels (`**Critical**`, `**Major**`) — consistent throughout |
| **Code-Zitate** | Fenced blocks with language tag or inline backticks; no huge dumps |
| **Links** | Full URLs to files/lines where the platform supports them |
| **Cross-refs** | GitHub: `#123`, commit SHA · GitLab: `#123`, `!42` when linking MRs |
| **Inline (Pflicht wenn möglich)** | Findings mit klarer Diff-Zeile als **line comment** an `path` + `line` posten — nicht nur in einem allgemeinen PR/MR-Kommentar auflisten |
| **GitHub Suggestions** | Wenn ein konkreter Fix für die kommentierten Zeilen formulierbar ist → **`suggestion`-Block** im Inline-Kommentar (siehe unten). Sonst nur Erklärung + optionaler Code-Vorschlag ohne `suggestion` |

Prefer the **common Markdown subset** (headings, bullets, code fences, links, `@mentions`) so rendering is reliable on both platforms.

### GitHub line comments + Suggestions (GFM)

When posting on **GitHub**, prefer **inline review comments** on the changed lines over a single top-level issue comment for actionable findings.

**Suggestion fence** (must be a fenced block whose language tag is exactly `suggestion` — GitHub renders an “Apply suggestion” UI):

````markdown
**Major:** Guard against null before use.

```suggestion
if (value == null) {
  return;
}
doWork(value);
```
````

| Regel | Detail |
|-------|--------|
| Wann Suggestion | Concrete, applyable replacement for the exact commented line(s); same indentation/style as the file |
| Wann keine Suggestion | Architectural questions, “consider X”, multi-file refactors, or when the correct fix is ambiguous → prose + optional non-suggestion code fence |
| Multi-line | Use API `start_line` + `line` (and `start_side`/`side`) so the suggestion covers the full hunk being replaced |
| Side | Almost always `RIGHT` (the PR’s new file version) |
| Commit | Anchor comments to the PR head commit (`headRefOid`) |

Do **not** put a `suggestion` block in a general PR conversation comment — GitHub only applies suggestions on **pull request review comments** attached to a file/line.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq`, HEREDOC, `--body-file`, or MCP native JSON args
2. **Preserve newlines** — paragraphs and lists need real newlines (`\n` in JSON)
3. **Special characters** — backticks, `"`, `\` in code examples via proper JSON encoding
4. **No HTML** — Markdown instead of `<br>` / `<pre>`
5. **Preview** — before posting, show the comment draft as platform Markdown, not escaped JSON
6. **MCP schema first** — put the Markdown string in the tool’s `body` / `note` field as required by the schema

**GitHub (`gh` or REST) — GFM:**

General summary (optional, after inline comments):

```bash
gh pr comment <id> --body-file /tmp/review-summary.md
```

**Preferred — inline line comments** (with optional `suggestion`). Resolve `OWNER`/`REPO` and head SHA first:

```bash
OWNER=$(gh repo view --json owner -q .owner.login)
REPO=$(gh repo view --json name -q .name)
COMMIT=$(gh pr view <id> --json headRefOid -q .headRefOid)

# Single-line inline comment (+ suggestion when possible)
gh api "repos/${OWNER}/${REPO}/pulls/<id>/comments" \
  -f commit_id="$COMMIT" \
  -f path="src/example.ts" \
  -F line=42 \
  -f side=RIGHT \
  -f body="$(cat <<'EOF'
**Major:** Null check missing before use.

```suggestion
if (value == null) {
  return;
}
doWork(value);
```
EOF
)"
```

Multi-line replacement (set `start_line`/`line` to the range being replaced):

```bash
gh api "repos/${OWNER}/${REPO}/pulls/<id>/comments" \
  -f commit_id="$COMMIT" \
  -f path="src/example.ts" \
  -F start_line=40 \
  -F line=45 \
  -f start_side=RIGHT \
  -f side=RIGHT \
  -f body="$(cat <<'EOF'
**Critical:** Replace the whole block with a safe variant.

```suggestion
// replacement lines matching start_line..line
```
EOF
)"
```

Or submit several inline comments as one review (summary body + comments array via `gh api repos/.../pulls/<id>/reviews` with JSON/`--input`). Prefer that when posting Critical/Major batches so the recommendation stays in one review event.

**GitLab (`glab`, MCP, or REST) — GitLab Flavored Markdown:**

Prefer **inline / diff notes** on the relevant lines when the API/MCP supports positions. General note only for summary:

```bash
glab mr note <id> -m "$(cat <<'EOF'
## Code Review
...
EOF
)"
```

GitLab applyable suggestions (when posting an inline note on the exact lines):

````markdown
```suggestion:-0+0
replacement code
```
````

- MCP / REST: `body` = platform Markdown string, JSON-encoded via tool args or `json.dumps`; for inline notes include path/line (or position) per schema

### Pre-post checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Actionable findings posted as **inline line comments** when `path` + line are known from the diff
- [ ] **GitHub:** `suggestion` fence included whenever a concrete line-level fix is clear; anchored to head commit + `RIGHT` side
- [ ] Headings and blank lines between sections (summary comment)
- [ ] Findings with `file:line` where possible
- [ ] Review recommendation (Approve / Request changes) clear and near the top
- [ ] Payload valid for `gh` / `glab` / MCP schema / REST JSON
- [ ] No Cursor-/agent meta references

## Workflow

### Phase 1: Argumente validieren & MR-ID ermitteln

1. User-Prompt nach `/merge-request-review` parsen
2. Erstes Token prüfen: numerische IID/PR-Nummer (`42`, `!42`, `#42`) → **MR-ID**
3. Restlicher Text → **Focus** (Review-Schwerpunkte, Fragen, Einschränkungen)
4. Falls kein gültiges ID-Token: **Chat-Kontext** und **aktuellen Branch** prüfen (siehe oben)
5. Focus aus Kontext ergänzen, wenn im Prompt kein Freitext nach der ID steht
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

**Nach Erkennung:** Chat review title and refs use the detected platform only — GitHub → `Code Review: PR #<id>`, `#n`; GitLab → `Code Review: MR !<id>`, `!n` (Phase 7). Optional posts (Phase 8) use that platform’s Markdown only (see **Content formatting — GitLab or GitHub**).

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
| MR-Details | Titel, Beschreibung, Autor, Status, Labels, Approvals |
| MR-Diffs | Alle Dateiänderungen |
| MR-Commits | Commit-Historie, Message-Qualität |
| MR-Pipelines | CI/CD-Status |
| MR-Diskussionen / Notes | **Alle** bestehenden Kommentare, Threads, Review-Notes |
| MR-Approvals | Bereits erteilte Approvals und Review-States |

**GitLab — Kommentare laden (Pflicht):**

Via **`glab`** (wenn als Client gewählt):

1. `glab mr view <id> --comments` — schneller Überblick über Notes
2. Resolve/encode project path, then `glab api "projects/${PROJECT}/merge_requests/<id>/discussions"` — vollständige Threads inkl. resolved/unresolved und Inline-Positionen (`PROJECT=$(glab repo view -F json | jq -r .path_with_namespace | jq -sRr @uri)`)

Via **GitLab MCP** (wenn als Client gewählt):

1. Tool-Schema via `GetDynamicTools` prüfen (z.B. `get_merge_request_notes`, `list_merge_request_discussions`, `get_merge_request`)
2. **Alle** Diskussionen und Notes laden — inkl.:
   - Allgemeine MR-Kommentare (Discussion-Notes)
   - Inline-Kommentare an Code-Zeilen (Diff-Notes)
   - System-Notes nur wenn relevant (z.B. „resolved", „approved")
   - Antworten in Threads (vollständige Konversation, nicht nur Top-Level)
3. Pro Kommentar erfassen: Autor, Datum, Datei/Zeile (falls inline), Inhalt, resolved/unresolved

**GitLab — Kommentar posten (nur auf explizite User-Anfrage, Phase 8):**

- `create_workitem_note` oder äquivalentes MCP-Tool (Schema vor Aufruf prüfen)
- Body = **GitLab Flavored Markdown** (siehe **Content formatting — GitLab or GitHub**)

#### GitHub (via `gh`)

```bash
gh pr view <id> --json title,body,state,author,baseRefName,headRefName,files,commits,statusCheckRollup,reviews,comments
gh pr diff <id>
gh api repos/{owner}/{repo}/pulls/<id>/comments
gh api repos/{owner}/{repo}/issues/<id>/comments
gh pr view <id> --comments
```

**GitHub — Kommentare laden (Pflicht):**

- Issue-Kommentare (allgemeine PR-Diskussion)
- Review-Kommentare (`pulls/{id}/comments` — inline an Code-Zeilen)
- Review-Submissions (`APPROVED`, `CHANGES_REQUESTED`, `COMMENTED` + Body)
- Threads und Antworten vollständig einbeziehen

### Phase 3: Bestehende MR-Kommentare analysieren

**Vor der eigenen Code Review** alle geladenen Kommentare auswerten:

1. **Zusammenfassen**: Was wurde bereits angemerkt? Von wem (Reviewer, Autor, CI-Bot)?
2. **Status prüfen**: Welche Punkte sind offen vs. resolved/beantwortet?
3. **Duplikate vermeiden**: Bereits diskutierte Findings **nicht** erneut als neu melden
4. **Lücken identifizieren**: Was wurde in den Kommentaren angesprochen, aber im Code noch nicht gelöst?
5. **Kontext nutzen**: Autor-Antworten und Diskussionen in die Bewertung einbeziehen
6. **Widersprüche**: Falls Kommentare sich widersprechen oder veraltet sind → als 💬 Question markieren

Im Review-Bericht einen Abschnitt **„Bestehende MR-Diskussion"** einfügen:

```markdown
## Bestehende MR-Diskussion
| # | Autor | Typ | Status | Kurzinhalt | Relevanz für Review |
|---|-------|-----|--------|------------|---------------------|
| 1 | @user | inline `file.ts:42` | offen | ... | Noch offen — in Findings aufgenommen |
| 2 | @reviewer | allgemein | resolved | ... | Bereits adressiert — nicht wiederholen |
```

### Phase 4: Kontext sammeln

1. **Betroffene Dateien im Repo lesen** — nicht nur Diff-Hunks
2. **Projekt-Konventionen** — `.cursor/rules/`, `CONTRIBUTING.md`
3. **Tests** — Werden geänderte Bereiche getestet?
4. **Abhängigkeiten** — Werden andere Module/APIs beeinflusst?
5. **MR-Kommentare** — Offene Punkte aus Phase 3 bei der Analyse berücksichtigen
6. **Focus aus Phase 1** — genannte Bereiche priorisieren; explizit ausgeschlossene Bereiche nur oberflächlich prüfen

### Phase 5: Intensive Code Review (drei Durchgänge)

**Focus beachten:** Wenn der User Schwerpunkte gesetzt hat (z.B. Security, bestimmte Dateien), diese in allen Durchgängen priorisieren und im Executive Summary explizit adressieren.

#### Durchgang 1: Architektur & Korrektheit

- Erfüllt der MR sein stated goal?
- Logische Fehler, Race Conditions, Edge Cases?
- Breaking Changes dokumentiert?
- Passt zur bestehenden Architektur?

#### Durchgang 2: Code-Qualität & Wartbarkeit

- Lesbarkeit, Funktionsgröße, Naming
- Error Handling vollständig?
- Konsistenz mit Projekt-Stil
- Keine unnötigen Änderungen

#### Durchgang 3: Sicherheit, Performance & Tests

- Security: Injection, XSS, Auth-Bypass, Secrets
- Performance: N+1, fehlende Pagination
- Tests und CI-Status
- Observability bei kritischen Pfaden

### Phase 6: Severity-Klassifizierung

| Stufe | Bedeutung | Merge-Blocker? |
|-------|-----------|----------------|
| 🔴 **Critical** | Bug, Security, Data Loss | Ja |
| 🟠 **Major** | Falsches Verhalten, fehlende kritische Tests | Ja (meist) |
| 🟡 **Minor** | Stil, Naming | Nein |
| 🔵 **Suggestion** | Alternative Ansätze | Nein |
| 💬 **Question** | Unklarheit | Nein |

### Phase 7: Review-Bericht

**Chat title (strict — match detected platform):**

- **GitHub:** `# Code Review: PR #<id> — <title>`
- **GitLab:** `# Code Review: MR !<id> — <title>`

Use `#id` / “Pull Request” wording in the GitHub report; `!id` / “Merge Request” on GitLab. Do not mix dialects in one report. Output **exactly one** skeleton below — the branch that matches the detected platform.

```markdown
<!-- GitHub skeleton (PR / #) -->
# Code Review: PR #<id> — <title>

## Executive Summary
<2-4 Sätze + Merge-Empfehlung>

**Empfehlung:** ✅ Approve / ⚠️ Approve with comments / ❌ Request changes

## PR-Metadaten
| Feld | Wert |
|------|------|
| Autor | ... |
| Branch | `<source>` → `<target>` |
| Commits | ... |
| Dateien | ... (+/- Zeilen) |
| Checks / CI | ✅/❌/⏳ |
| Bestehende Kommentare | <Anzahl> (<offen> offen, <resolved> resolved) |

## Bestehende PR-Diskussion
<Zusammenfassung + Tabelle aus Phase 3>

## Findings
### 🔴 Critical
| # | Datei | Problem | Empfehlung | Bereits kommentiert? |
...

## Positives
- ...

## Commit-Message-Analyse
- Conventional Commits eingehalten? Ja/Nein

## Testplan-Bewertung
- Testplan vorhanden? Fehlende Tests?

## Nächste Schritte
1. ...
```

```markdown
<!-- GitLab skeleton (MR / !) -->
# Code Review: MR !<id> — <title>

## Executive Summary
<2-4 Sätze + Merge-Empfehlung>

**Empfehlung:** ✅ Approve / ⚠️ Approve with comments / ❌ Request changes

## MR-Metadaten
| Feld | Wert |
|------|------|
| Autor | ... |
| Branch | `<source>` → `<target>` |
| Commits | ... |
| Dateien | ... (+/- Zeilen) |
| Pipeline | ✅/❌/⏳ |
| Bestehende Kommentare | <Anzahl> (<offen> offen, <resolved> resolved) |

## Bestehende MR-Diskussion
<Zusammenfassung + Tabelle aus Phase 3>

## Findings
### 🔴 Critical
| # | Datei | Problem | Empfehlung | Bereits kommentiert? |
...

## Positives
- ...

## Commit-Message-Analyse
- Conventional Commits eingehalten? Ja/Nein

## Testplan-Bewertung
- Testplan vorhanden? Fehlende Tests?

## Nächste Schritte
1. ...
```

### Phase 8: MR/PR-Kommentare (optional, nur auf Anfrage)

**Review-Bericht im Chat ausgeben** (Phase 7). **Nicht automatisch** auf die Plattform posten.

Posting nur wenn der User **explizit** danach fragt (z.B. „post as PR comment", „post critical & major only", „post inline with suggestions"). Dann:

1. Pro Finding mit bekannter Diff-Position: **Inline-Zeilenkommentar** an `path` + `line` (nicht alles in einen allgemeinen Thread kippen)
2. **GitHub:** Wenn ein klarer Zeilen-Fix möglich ist → Body enthält einen **`suggestion`-Block** (siehe **GitHub line comments + Suggestions**). Sonst Severity + Erklärung + optionaler normaler Code-Fence
3. Bereits bestehende Kommentare referenzieren statt duplizieren
4. Optional **ein** Summary-Kommentar / Review-Body mit Recommendation + Überblick (Critical/Major counts); Details bleiben an den Zeilen
5. Formatierung — **Content formatting — GitLab or GitHub**
6. Posten:
   - **GitHub**: `gh api .../pulls/<id>/comments` (inline + suggestion) und/oder `.../pulls/<id>/reviews` mit `comments[]`; Summary optional via `gh pr comment` — **GitHub GFM**
   - **GitLab**: Inline/diff notes via `glab api .../discussions` or MCP when position is known; optional `suggestion:-0+0`; Summary via `glab mr note` — **GitLab Flavored Markdown**
7. Erfolg bestätigen und **URLs** der geposteten Inline-Kommentare (und ggf. Summary) zurückgeben

**Vor dem Posten** die geplanten Inline-Bodies (inkl. Suggestions) im Chat zeigen und nur nach Bestätigung senden — gleiches Ask-before-mutating Prinzip wie bei anderen Write-Paths.

**Inline-Kommentar-Template** (GitHub, English, no automation markers):

````markdown
**<Severity>:** <one-sentence problem>

<optional short rationale>

```suggestion
<exact replacement for the commented line(s)>
```
````

**Summary-Template** (optional top-level / review body — English):

```markdown
## Code Review

**Recommendation:** Approve / Approve with comments / Request changes

### Summary
<2-3 sentences>

### Inline findings
Posted <n> line comments (Critical: <n>, Major: <n>, …). See threads on the Files changed tab.

### Positives
<What works well>

### Test plan
<Assessment>
```

---

## Review-Grundsätze

1. **Konkret**: Datei + Zeile + Problem + Fix-Vorschlag — Findings ohne Zeile nur wenn kein Diff-Anker möglich
2. **Inline zuerst**: Beim Posten Zeilen kommentieren; auf **GitHub** Suggestions nutzen, sobald der Fix für genau diese Zeilen klar ist
3. **Konstruktiv**: Positives hervorheben
4. **Priorisiert**: Critical zuerst
5. **Kein Fix ohne Aufforderung**: Review ≠ Implementation (Suggestion ≠ lokal angewendeter Fix)
6. **Keine Duplikate**: Bestehende MR/PR-Kommentare einlesen, zusammenfassen und nicht wiederholen
7. **Diskussion einbeziehen**: Autor-Antworten und resolved/unresolved-Status berücksichtigen

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Keine MR-ID ermittelbar | AskQuestion oder abbrechen (siehe oben) |
| Mehrere MRs im Kontext | User fragen, welcher MR gemeint ist |
| MR/PR nicht gefunden | ID und Projekt prüfen |
| Remote-Host unbekannt | AskQuestion oder abbrechen — nicht raten. Kein „push + manuelle URL"-Pfad |
| GitHub ohne `gh` Auth | Abbrechen, `gh auth login` anleiten |
| GitLab ohne `glab` und ohne MCP | Abbrechen, CLI- oder MCP-Setup anleiten |
| MCP nicht authentifiziert | Auth anleiten, abbrechen |
| MR/PR >500 Zeilen Diff | Modulweise Review, Split empfehlen |

## Verwandte Commands

- `/merge-request-fix [id] [instructions]` — Review-Kommentare umsetzen und Fix-Summary erstellen
- `/commit` — Änderungen committen
- `/merge-request` — MR/PR erstellen
