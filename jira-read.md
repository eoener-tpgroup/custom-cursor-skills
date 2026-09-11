---
name: jira-read
description: Liest ein Jira-Ticket anhand der Ticket-Nummer. Erfordert Ticket-Key (/jira-read PITOPS-123456).
---

# Jira Read

Lädt ein Jira-Ticket anhand des Ticket-Keys und gibt eine **strukturierte, vollständige Zusammenfassung** zurück — inkl. Status, Felder, Beschreibung und Kommentare.

Wird manuell via `/jira-read <ticket-key>` ausgelöst.

## Output language

**Always write all user-facing output in English** — status messages, ticket summaries, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

Ticket content (summary, description, comments) is quoted **as-is** from Jira — do not translate unless the user explicitly asks.

---

## ⚠️ Pflichtargument: Ticket-Key

**Ohne Ticket-Key sofort abbrechen.** Kein Ticket laden, kein Raten, keine Alternativen anbieten.

```
❌ /jira-read                    → ABBRECHEN
❌ /jira-read read               → ABBRECHEN (kein gültiger Key)
❌ /jira-read 123456             → ABBRECHEN (kein Projekt-Präfix)
✅ /jira-read PITOPS-144486      → Ticket PITOPS-144486
✅ /jira-read pitops-144486      → Normalisieren zu PITOPS-144486
```

**Abort message** (report exactly as follows):

> This command requires a Jira ticket key as an argument.
> Usage: `/jira-read <ticket-key>`
> Example: `/jira-read PITOPS-144486`

Der Ticket-Key steht im User-Prompt **nach dem Command-Namen** (z.B. `/jira-read PITOPS-144486`).

---

## Voraussetzungen

- Jira-Zugang über MCP **oder** Credentials in `~/.cursor/.env` (`JIRA_URL`, `JIRA_TOKEN`)
- Gültiger Ticket-Key im Format `<PROJECT>-<NUMBER>` (z.B. `PITOPS-144486`)

## Security (strikt einhalten)

- **Niemals** Token-Werte ausgeben, loggen oder in Code/Chat kopieren
- **Niemals** Credentials committen oder in Dateien schreiben
- Credentials nur aus `~/.cursor/.env` laden — **nicht** den User nach Token fragen, wenn die Datei existiert

---

## Workflow

### Phase 1: Argument validieren

1. Ticket-Key aus dem User-Prompt extrahieren (Text nach `/jira-read`)
2. Normalisieren: Trimmen, Großschreibung des Projekt-Präfixes (`pitops-123` → `PITOPS-123`)
3. Format prüfen: Regex `^[A-Z][A-Z0-9]+-\d+$`
4. Wenn ungültig → **sofort abbrechen** (Abort message ausgeben)

### Phase 2: Jira-Zugang prüfen

**Priorität** (wie in `service-credentials.mdc`):

1. Jira MCP prüfen (`GetDynamicTools`, Pattern `jira`)
2. Bei verfügbarem, authentifiziertem MCP: MCP-Tools für Issue-Abruf nutzen (Schema vor Aufruf prüfen)
3. **Fallback**: REST API via `~/.cursor/.env`

**REST API — Credentials laden:**

```bash
source ~/.cursor/.env
# JIRA_URL und JIRA_TOKEN müssen gesetzt sein
```

Bei fehlenden Credentials oder Auth-Fehler (401/403): User informieren, **nicht** ohne Auth fortfahren.

### Phase 3: Ticket laden

**REST API (Fallback):**

```bash
source ~/.cursor/.env
curl -s -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>?expand=renderedFields,names,changelog"
```

Zusätzlich Kommentare laden (falls nicht vollständig in `fields.comment`):

```bash
curl -s -H "Authorization: Bearer $JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/<TICKET-KEY>/comment?orderBy=created&maxResults=100"
```

**Zu ladende Daten (Pflicht):**

| Daten | Zweck |
|-------|-------|
| Key, Summary, Description | Kerninhalt |
| Status, Resolution | Aktueller Stand |
| Issue Type, Priority | Klassifizierung |
| Reporter, Assignee, Created, Updated | Metadaten |
| Components, Labels | Zuordnung |
| Custom Fields (falls vorhanden) | Environment, Platform, Request Type, etc. |
| Comments (alle) | Vollständiger Kommentarverlauf |
| Changelog (letzte 20 Einträge) | Status-/Feldänderungen |

Bei HTTP 404: Ticket nicht gefunden — Key prüfen lassen.
Bei anderen Fehlern: HTTP-Status und Fehlermeldung (ohne Token) an User melden.

### Phase 4: Ticket zusammenfassen

Strukturierten Bericht ausgeben:

```markdown
# Jira Ticket: <KEY> — <summary>

## Overview
| Field | Value |
|-------|-------|
| Key | <KEY> |
| Summary | ... |
| Type | ... |
| Status | ... |
| Priority | ... |
| Reporter | ... |
| Assignee | ... |
| Created | ... |
| Updated | ... |
| Components | ... |
| Labels | ... |
| URL | <JIRA_URL>/browse/<KEY> |

## Description
<description — rendered/plain, preserve formatting>

## Custom Fields
| Field | Value |
|-------|-------|
| Environment | ... |
| Platform | ... |
| ... | ... |

## Comments (<count>)
| # | Author | Date | Content |
|---|--------|------|---------|
| 1 | ... | ... | ... |

## Recent Activity (Changelog)
| Date | Author | Change |
|------|--------|--------|
| ... | ... | status: Open → In Progress |

## Links
- Browse: <JIRA_URL>/browse/<KEY>
```

**Regeln für die Zusammenfassung:**

1. **Vollständig**: Alle Kommentare einbeziehen, nicht nur die letzten
2. **Chronologisch**: Kommentare nach Erstellungsdatum sortiert
3. **Keine Interpretation**: Beschreibung und Kommentare wörtlich wiedergeben
4. **Lange Inhalte**: Bei sehr langen Beschreibungen/Kommentaren (>2000 Zeichen) gekürzt mit Hinweis „(truncated — full content in Jira)"
5. **Attachments**: Nur auflisten (Dateiname, Autor, Datum) — nicht herunterladen, es sei denn der User fordert es explizit

### Phase 5: Ergebnis melden

- Ticket-URL (`<JIRA_URL>/browse/<KEY>`)
- Kurze Ein-Satz-Zusammenfassung des Ticket-Inhalts
- Hinweis auf verwandte Commands:
  - `/jira-comment <key>` — Kommentar hinzufügen
  - `/jira-create` — Neues Ticket erstellen

---

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Kein Ticket-Key | Sofort abbrechen (Abort message) |
| Ungültiges Key-Format | Abort message, Beispiel geben |
| Ticket nicht gefunden (404) | Key und Projekt prüfen lassen |
| Auth fehlgeschlagen (401/403) | Credentials in `~/.cursor/.env` prüfen |
| MCP nicht authentifiziert | Auth anleiten, REST-Fallback versuchen |
| Netzwerkfehler | Fehler melden, Retry anbieten |
| Ticket ohne Beschreibung/Kommentare | Felder als „(empty)" markieren, nicht abbrechen |

## Verwandte Commands

- `/jira-create` — Neues Jira-Ticket erstellen
- `/jira-comment <ticket-key>` — Kommentar zu einem Ticket hinzufügen
