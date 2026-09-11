---
name: jira-comment
description: Setzt einen Kommentar auf ein bestehendes Jira-Ticket. Erfordert Ticket-Key. Kommentar aus User-Input oder Kontext. Bestätigung vor Posten via AskQuestion.
---

# Jira Comment

Setzt einen **Kommentar** auf ein bestehendes Jira-Ticket. Der Kommentar-Inhalt kommt entweder aus dem User-Prompt oder wird aus dem **bisherigen Chat-Kontext** abgeleitet.

Wird manuell via `/jira-comment <ticket-key> [comment text]` ausgelöst.

## Output language

**Always write all user-facing output in English** — status messages, confirmation messages, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

**Jira comments must always be written in English** — regardless of the language used in chat. This matches team conventions for external-facing Jira content.

---

## ⚠️ Pflichtargument: Ticket-Key

**Ohne Ticket-Key sofort abbrechen.** Kein Kommentar posten, kein Raten, keine Alternativen anbieten.

```
❌ /jira-comment                 → ABBRECHEN
❌ /jira-comment fix this        → ABBRECHEN (kein gültiger Key)
❌ /jira-comment 123456          → ABBRECHEN (kein Projekt-Präfix)
✅ /jira-comment PITOPS-144486  → Kommentar aus Kontext ableiten
✅ /jira-comment PITOPS-144486 Investigation complete, FS check passed → Kommentar aus Prompt
```

**Abort message** (report exactly as follows):

> This command requires a Jira ticket key as the first argument.
> Usage: `/jira-comment <ticket-key> [comment text]`
> Example: `/jira-comment PITOPS-144486 Root cause identified and fix deployed.`

Der Ticket-Key ist das **erste Argument** nach dem Command-Namen. Optionaler Freitext danach = expliziter Kommentar-Inhalt.

---

## Voraussetzungen

- Jira-Zugang über MCP **oder** Credentials in `~/.cursor/.env` (`JIRA_URL`, `JIRA_TOKEN`)
- Gültiger Ticket-Key im Format `<PROJECT>-<NUMBER>`
- Ticket existiert und ist kommentierbar

## Security (strikt einhalten)

- **Niemals** Token-Werte ausgeben, loggen oder in Code/Chat kopieren
- **Niemals** Credentials committen oder in Dateien schreiben
- **Keine Secrets** in Kommentare schreiben (Passwörter, Tokens, API-Keys, interne Credentials)
- Credentials nur aus `~/.cursor/.env` laden

## Content formatting — Jira (strict, MCP and REST)

**Mandatory for every write path.** Whether the comment is posted via **Jira MCP**, **Service Desk API**, or **REST API 2/3**, the body MUST use **Jira-renderable markup** — never GitHub/GitLab GFM conventions that Jira does not render, and never raw HTML.

### Dialect (detect once, then stick to it)

| Preference | When | Body format |
|------------|------|-------------|
| 1 | MCP tool expects ADF / `document` structure | Build valid **Atlassian Document Format** per tool schema |
| 2 | Cloud / newer Server-DC with Markdown renderer | **Jira Markdown** (string `body`) |
| 3 | Older Wiki-Markup instances | **Jira Wiki Markup** |

If unsure which dialect the instance renders, prefer the format already used in existing comments on the same ticket (load one comment first). Do **not** mix dialects in one comment.

### Jira Markdown (default string body)

| Element | Use | Do not use |
|---------|-----|------------|
| **Headings** | `## Investigation Summary`, `## Next Steps` | Single `#` (often wrong in Jira); GitHub-only heading tricks |
| **Bold / italic** | `**bold**`, `*italic*` | HTML `<b>` / `<i>` |
| **Lists** | `-` bullets for findings; `1.` for ordered steps | Nested GFM task lists if unsupported |
| **Code** | Fenced ```` ``` ```` blocks or `` `inline` `` | Bare indented code that Jira flattens |
| **Links** | Full URLs (`https://…`) or Jira `[text\|url]` if Wiki | Cursor deep-links, relative paths |
| **Quotes** | `>` blockquotes sparingly | Nested quote stacks |

### Jira Wiki Markup (legacy string body)

| Element | Syntax |
|---------|--------|
| Headings | `h2. Investigation Summary` |
| Bold / italic | `*bold*`, `_italic_` |
| Bullets | `* item` / `# item` (ordered) |
| Code | `{code}…{code}` or `{noformat}…{noformat}` |
| Links | `[label\|https://example.com]` |

### Technical encoding (MCP and REST)

Applies to **every** transport — MCP tool arguments, `curl` JSON, Service Desk payloads:

1. **Never hand-escape JSON** — build the payload with `json.dumps()` (or the MCP client's native JSON args)
2. **Newlines** — real paragraphs as `\n` in JSON strings; blank line between sections
3. **Special characters** — `"`, `\`, backticks only via proper JSON encoding
4. **No raw HTML** in `body`
5. **Preview** — Phase 4 shows the draft as Jira would render it (Markdown/Wiki view), not as escaped JSON
6. **MCP schema first** — if the tool wants ADF/`content` nodes, convert the draft to that structure; do not pass GFM into an ADF field

### Pre-post checklist

- [ ] Dialect matches the instance (Markdown, Wiki, or ADF)
- [ ] Headings and blank lines between sections
- [ ] Lists instead of wall-of-text
- [ ] Code in Jira code fences / `{code}` / ADF code blocks
- [ ] Payload valid for the chosen API (MCP schema or REST JSON)
- [ ] No secrets, no agent/Cursor meta references

---

## Workflow

### Phase 1: Argument validieren

1. Ticket-Key aus dem User-Prompt extrahieren (erstes Token nach `/jira-comment`)
2. Normalisieren: Trimmen, Projekt-Präfix großschreiben (`pitops-123` → `PITOPS-123`)
3. Format prüfen: Regex `^[A-Z][A-Z0-9]+-\d+$`
4. Restlichen Text nach dem Key als **expliziten Kommentar** behandeln (falls vorhanden)
5. Wenn ungültiger Key → **sofort abbrechen** (Abort message)

### Phase 2: Kommentar-Inhalt ermitteln

**Zwei Quellen** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Expliziter Text** im User-Prompt nach dem Ticket-Key | User hat Kommentar direkt angegeben |
| 2 | **Chat-Kontext** | Kein expliziter Text — aus bisheriger Konversation ableiten |

**Aus Kontext ableiten — Regeln:**

1. Den **Kern der letzten Arbeit/Diskussion** zusammenfassen (Investigation, Fix, Status-Update, etc.)
2. Nur Informationen einbeziehen, die **für Jira-Teilnehmer relevant** sind
3. Keine internen Tool-/Agent-Referenzen, keine Meta-Kommentare über Automatisierung
4. Professioneller, sachlicher Ton — als hätte der User persönlich geschrieben
5. Strukturiert im **Jira-Dialekt** formatieren (siehe **Content formatting — Jira**): Überschriften, Listen, Code-Blöcke — nicht GitHub/GitLab GFM

**Bei Unklarheiten → User fragen (Pflicht):**

Wenn der gewünschte Kommentar-Inhalt **nicht eindeutig** ist (weder aus Prompt noch aus Kontext), **`AskQuestion`** nutzen (Pflicht — kein Fallback auf Chat-Fragen):

Frage:

> What should the Jira comment say?

Optionen (via AskQuestion):

1. **Post investigation/status summary from context** — Zusammenfassung der bisherigen Arbeit
2. **Post a short status update** — Kurzes Status-Update (User liefert Text im Freitext-Feld)
3. **Let me specify the comment** — User schreibt den exakten Kommentar
4. **Cancel** — Abbrechen, nichts posten

**Nicht posten**, solange der Kommentar-Inhalt nicht bestätigt ist, wenn Unklarheit besteht.

### Phase 3: Ticket verifizieren

Vor dem Posten prüfen, dass das Ticket existiert:

```bash
source ~/.cursor/.env
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>"
```

| HTTP-Code | Aktion |
|-----------|--------|
| 200 | Fortfahren |
| 404 | Abbrechen: Ticket nicht gefunden |
| 401/403 | Abbrechen: Auth-Problem |

Optional: Ticket-Kurzinfo laden (Summary, Status) für Bestätigung an User.

### Phase 4: Kommentar-Entwurf zeigen und bestätigen lassen (Pflicht)

**Immer vor dem Posten** den vollständigen Kommentar-Entwurf zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch posten** — erst nach expliziter Bestätigung.

```markdown
## Comment draft for <TICKET-KEY>

<comment body>

---
Post this comment to <TICKET-KEY>?
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> Post this comment to <TICKET-KEY>?

Optionen:

1. **Yes, post comment** — Kommentar absenden
2. **Edit first** — User gibt Korrekturen, Entwurf anpassen, erneut bestätigen
3. **Cancel** — Nichts posten

Bei **Cancel**: Kurz bestätigen, dass nichts gepostet wurde.

### Phase 5: Kommentar posten (nur nach Bestätigung)

**Formatierung:** Siehe **Content formatting — Jira** — gilt für **MCP und REST** gleichermaßen (Jira Markdown / Wiki / ADF laut Schema).

**Priorität** (wie in `service-credentials.mdc`):

1. Jira MCP prüfen und nutzen, falls verfügbar (Schema vor Aufruf prüfen; Body im geforderten Jira-Format)
2. **Fallback**: REST API (string `body` als Jira Markdown oder Wiki Markup)

**REST API:**

```bash
source ~/.cursor/.env
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>/comment" \
  -d "$(python3 <<'PYEOF'
import json
body = """<comment text here>"""
print(json.dumps({"body": body}))
PYEOF
)"
```

Bei HTTP 201/200: Erfolg.
Bei Fehler: HTTP-Code und Fehlermeldung (ohne Token) melden.

### Phase 6: Ergebnis melden

Bei Erfolg:

- Bestätigung: „Comment posted to `<TICKET-KEY>`"
- Link zum Ticket: `<JIRA_URL>/browse/<TICKET-KEY>`
- Kommentar-ID falls aus Response verfügbar
- Geposteten Kommentar nochmals kurz anzeigen (erste ~200 Zeichen)

Bei Abbruch:

- Kurz bestätigen, dass nichts gepostet wurde

---

## Kommentar-Templates

### Status-Update

```markdown
## Status Update

<1-2 sentences on current state>

**Next steps:**
- <step 1>
- <step 2>
```

### Investigation Summary

```markdown
## Investigation Summary

**Scope:** <what was investigated>

**Findings:**
- <finding 1>
- <finding 2>

**Conclusion:** <conclusion>

**Actions taken:**
- <action 1>
```

### Fix Deployed

```markdown
## Fix Applied

**Problem:** <brief problem description>
**Solution:** <what was done>
**Verification:** <how it was verified>

Related: <MR/PR link if applicable>
```

---

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Kein Ticket-Key | Sofort abbrechen (Abort message) |
| Ungültiges Key-Format | Abort message |
| Ticket nicht gefunden | Key prüfen lassen |
| Kommentar-Inhalt unklar | AskQuestion — nicht raten |
| User bricht ab | Bestätigen, nichts posten |
| Auth fehlgeschlagen | Credentials prüfen |
| Secrets im Entwurf | Warnen, aus Kommentar entfernen, erneut bestätigen |
| Permission denied (403) | User informieren — evtl. kein Kommentar-Recht |

## Verwandte Commands

- `/jira-read <ticket-key>` — Ticket lesen
- `/jira-create` — Neues Ticket erstellen
