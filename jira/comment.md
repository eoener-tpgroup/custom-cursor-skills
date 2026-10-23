# Jira — subcommand: comment

Loaded by `/jira comment …` after `jira/_common.md`. Do not run as a standalone command.

Setzt einen **Kommentar** auf ein bestehendes Jira-Ticket. Der Kommentar-Inhalt kommt entweder aus dem User-Prompt oder wird aus dem **bisherigen Chat-Kontext** abgeleitet.

**Invocation:** `/jira comment <ticket-key> [comment text]`

**Jira comments must always be written in English** — regardless of the language used in chat.

Wiki Markup, security, and auth: see `jira/_common.md`.

---

## ⚠️ Pflichtargument: Ticket-Key

**Ohne Ticket-Key sofort abbrechen.** Kein Kommentar posten, kein Raten, keine Alternativen anbieten.

```
❌ /jira comment                 → ABBRECHEN
❌ /jira comment fix this        → ABBRECHEN (kein gültiger Key)
❌ /jira comment 123456          → ABBRECHEN (kein Projekt-Präfix)
✅ /jira comment PITOPS-144486  → Kommentar aus Kontext ableiten
✅ /jira comment PITOPS-144486 Investigation complete, FS check passed → Kommentar aus Prompt
```

**Abort message** (report exactly as follows):

> This command requires a Jira ticket key as the first argument.
> Usage: `/jira comment <ticket-key> [comment text]`
> Example: `/jira comment PITOPS-144486 Root cause identified and fix deployed.`

Der Ticket-Key ist das **erste Argument** nach dem Subcommand. Optionaler Freitext danach = expliziter Kommentar-Inhalt.

---

## Workflow

### Phase 1: Argument validieren

1. Ticket-Key aus dem User-Prompt extrahieren (erstes Token nach `/jira comment`)
2. Normalisieren und Format prüfen — siehe `jira/_common.md` (Ticket key rules)
3. Restlichen Text nach dem Key als **expliziten Kommentar** behandeln (falls vorhanden)
4. Wenn ungültiger Key → **sofort abbrechen** (Abort message)

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
5. Strukturiert in **Jira Wiki Markup** formatieren (siehe `jira/_common.md`)

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

**Formatierung:** Siehe `jira/_common.md` — **immer Jira Wiki Markup**.

**Priorität:** MCP first, then REST (siehe `_common`).

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

## Kommentar-Templates (Wiki Markup)

### Status-Update

```text
h2. Status Update

<1-2 sentences on current state>

*Next steps:*
* <step 1>
* <step 2>
```

### Investigation Summary

```text
h2. Investigation Summary

*Scope:* <what was investigated>

*Findings:*
* <finding 1>
* <finding 2>

*Conclusion:* <conclusion>

*Actions taken:*
* <action 1>
```

### Fix Deployed

```text
h2. Fix Applied

*Problem:* <brief problem description>
*Solution:* <what was done>
*Verification:* <how it was verified>

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
