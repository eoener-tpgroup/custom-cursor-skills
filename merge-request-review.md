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

Prefer the **common Markdown subset** (headings, bullets, code fences, links, `@mentions`) so rendering is reliable on both platforms.

### Technical encoding (MCP, CLI, and REST)

1. **Never hand-escape JSON** — use `json.dumps()`, `jq`, HEREDOC, `--body-file`, or MCP native JSON args
2. **Preserve newlines** — paragraphs and lists need real newlines (`\n` in JSON)
3. **Special characters** — backticks, `"`, `\` in code examples via proper JSON encoding
4. **No HTML** — Markdown instead of `<br>` / `<pre>`
5. **Preview** — before posting, show the comment draft as platform Markdown, not escaped JSON
6. **MCP schema first** — put the Markdown string in the tool’s `body` / `note` field as required by the schema

**GitHub (`gh` or REST) — GFM:**

```bash
gh pr comment <id> --body-file /tmp/review-comment.md
# or gh pr review <id> --comment -b "$(cat <<'EOF'
## Code Review
...
EOF
)"
```

**GitLab (`glab`, MCP, or REST) — GitLab Flavored Markdown:**

```bash
glab mr note <id> -m "$(cat <<'EOF'
## Code Review
...
EOF
)"
```

- MCP / REST: `body` = platform Markdown string, JSON-encoded via tool args or `json.dumps`

### Pre-post checklist

- [ ] Platform detected; dialect matches GitHub GFM **or** GitLab Markdown
- [ ] Headings and blank lines between sections
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

**Formatting dialect:** Once GitLab vs GitHub is known, any optional MR/PR comment (Phase 8) uses that platform’s Markdown only (see **Content formatting — GitLab or GitHub**).

#### GitLab (MCP bevorzugt)

1. GitLab MCP prüfen (`GetDynamicTools`, Namespace `GitLab`)
2. Bei `needsAuth`: User bitten zu authentifizieren, **nicht** ohne Auth fortfahren
3. Daten laden:

| Daten | Zweck |
|-------|-------|
| MR-Details | Titel, Beschreibung, Autor, Status, Labels, Approvals |
| MR-Diffs | Alle Dateiänderungen |
| MR-Commits | Commit-Historie, Message-Qualität |
| MR-Pipelines | CI/CD-Status |
| MR-Diskussionen / Notes | **Alle** bestehenden Kommentare, Threads, Review-Notes |
| MR-Approvals | Bereits erteilte Approvals und Review-States |

**GitLab — Kommentare laden (Pflicht):**

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

```markdown
# Code Review: MR !<id> — <titel>

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

### Phase 8: MR-Kommentar (optional, nur auf Anfrage)

**Review-Bericht im Chat ausgeben** (Phase 7). **Nicht automatisch** als MR-Kommentar posten.

Posting nur wenn der User **explizit** danach fragt (z.B. im Anschluss: „post as MR comment", „post critical & major only"). Dann:

1. Kommentar für die Plattform formatieren — **Content formatting — GitLab or GitHub** (GitHub GFM bzw. GitLab Flavored Markdown; gilt für MCP, CLI und REST)
2. Bereits bestehende Kommentare referenzieren statt duplizieren
3. Review-Empfehlung (Approve / Request changes) klar angeben
4. Posten:
   - **GitLab**: `create_workitem_note` (MCP) oder `glab mr note <id> -m "..."` — **GitLab Flavored Markdown**
   - **GitHub**: `gh pr comment <id> --body "..."` oder `gh pr review <id> --comment -b "..."` — **GitHub GFM**
5. Erfolg bestätigen und **URL des Kommentars** zurückgeben

**Kommentar-Template** (für MR-Posting — English, no automation markers):

```markdown
## Code Review

**Recommendation:** Approve / Approve with comments / Request changes

### Summary
<2-3 sentences>

### Critical / Major
<Findings with file:line>

### Additional notes
<Minor, suggestions, open questions>

### Positives
<What works well>

### Test plan
<Assessment>
```

---

## Review-Grundsätze

1. **Konkret**: Datei + Zeile + Problem + Fix-Vorschlag
2. **Konstruktiv**: Positives hervorheben
3. **Priorisiert**: Critical zuerst
4. **Kein Fix ohne Aufforderung**: Review ≠ Implementation
5. **Keine Duplikate**: Bestehende MR-Kommentare einlesen, zusammenfassen und nicht wiederholen
6. **Diskussion einbeziehen**: Autor-Antworten und resolved/unresolved-Status berücksichtigen

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Keine MR-ID ermittelbar | AskQuestion oder abbrechen (siehe oben) |
| Mehrere MRs im Kontext | User fragen, welcher MR gemeint ist |
| MR nicht gefunden | ID und Projekt prüfen |
| MCP nicht authentifiziert | Auth anleiten, abbrechen |
| MR >500 Zeilen Diff | Modulweise Review, Split empfehlen |

## Verwandte Commands

- `/merge-request-fix [id] [instructions]` — Review-Kommentare umsetzen und Fix-Summary erstellen
- `/commit` — Änderungen committen
- `/merge-request` — MR/PR erstellen
