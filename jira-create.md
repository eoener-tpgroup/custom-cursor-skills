---
name: jira-create
description: Erstellt ein neues Jira-Ticket. Inhalt aus User-Beschreibung oder Chat-Kontext. Bestätigung vor Erstellung via AskQuestion.
---

# Jira Create

Erstellt ein **neues Jira-Ticket** in unserem Jira (Transporeon Service Desk, Projekt **PITOPS**). Der Ticket-Inhalt kommt entweder aus dem User-Prompt oder wird aus dem **bisherigen Chat-Kontext** abgeleitet.

Wird manuell via `/jira-create [description]` ausgelöst.

## Output language

**Always write all user-facing output in English** — status messages, confirmation messages, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

**Jira ticket content (summary, description) must always be written in English** — regardless of the language used in chat.

---

## Voraussetzungen

- Jira-Zugang über MCP **oder** Credentials in `~/.cursor/.env` (`JIRA_URL`, `JIRA_TOKEN`)
- Ausreichend Informationen für Pflichtfelder (siehe Phase 3)

## Security (strikt einhalten)

- **Niemals** Token-Werte ausgeben, loggen oder in Code/Chat kopieren
- **Niemals** Credentials committen oder in Dateien schreiben
- **Keine Secrets** in Tickets schreiben (Passwörter, Tokens, API-Keys, interne Credentials)
- Credentials nur aus `~/.cursor/.env` laden — **nicht** den User nach Token fragen, wenn die Datei existiert

## Content formatting — Jira (strict, MCP and REST)

**Mandatory for every write path.** Whether the ticket is created via **Jira MCP**, **Service Desk API**, or **REST API 2/3**, fields MUST use **Jira Wiki Markup** — never GitHub/GitLab GFM, never Jira Markdown (`##`, `` `code` ``, fenced ```` ``` ````), never raw HTML.

### Dialect (fixed)

**Always author and submit description as Jira Wiki Markup** (string `description`). Do **not** detect or switch dialects. Do **not** use Jira Markdown or hand-written ADF as the source format.

| Preference | Transport | Action |
|------------|-----------|--------|
| 1 | REST / Service Desk with string `description` | Pass Wiki Markup string as-is |
| 2 | MCP tool that only accepts ADF / `document` | Prefer REST with Wiki string; if MCP is required, convert the **Wiki Markup draft** to ADF per schema — never invent Markdown first |

### Field rules

| Feld | Regeln |
|------|--------|
| **Summary** | Plain text only — **no** Wiki/Markdown (`h2.`, `*`, `#`, `**`, lists). Max. ~255 chars. Short, specific, single line. Put hostname/service first when relevant |
| **Description** | **Jira Wiki Markup only**. Structured sections; blank line between sections; follow „Description — Best Practices" |
| **Links** | `[label\|https://example.com]` or full URLs; no relative paths |
| **Error messages / logs** | Verbatim inside `{code}…{code}` or `{noformat}…{noformat}` — do not paraphrase |

### Jira Wiki Markup (required for description)

| Element | Syntax | Do not use |
|---------|--------|------------|
| Headings | `h2. Problem`, `h3. …` | `## Problem`, `#` |
| Bold / italic | `*bold*`, `_italic_` | `**bold**`, HTML |
| Bullets | `* item` / `# item` (ordered) | `- item` GFM lists |
| Inline / blocks | `{{hostname}}`, `{code}…{code}`, `{noformat}…{noformat}` | Fenced ```` ``` ````, `` `inline` `` |
| Links | `[label\|https://example.com]` | Markdown `[text](url)` |

### Technical encoding (MCP and REST)

Applies to **every** transport — MCP tool arguments and `curl` JSON:

1. **Never hand-escape JSON** — always `json.dumps()` for the full payload (or native MCP JSON args)
2. **Newlines** — in string `description` as `\n`; blank line between sections
3. **Special characters** — quotes in logs/paths via proper JSON encoding only
4. **No HTML** in description
5. **Preview** — Phase 4 shows Summary + Description as Wiki Markup (as Jira would render), not escaped JSON
6. **Wiki first** — draft in Wiki Markup; only convert to ADF if an MCP tool schema forces it

```bash
# Correct: full payload via json.dumps (REST / shell fallback) — Wiki Markup description
python3 <<'PYEOF'
import json
payload = {
    "serviceDeskId": "5",
    "requestTypeId": "657",
    "requestFieldValues": {
        "summary": "db01.example.com - disk alert investigation",
        "description": "h2. Problem\n\nFilesystem usage above threshold.\n\nh2. Details\n\n{code}\n<log excerpt>\n{code}",
        ...
    }
}
print(json.dumps(payload))
PYEOF
```

### Pre-create checklist

- [ ] Summary plain text, under 255 chars, no markup
- [ ] Description is **Wiki Markup only** (`h2.`, `*`, `{code}`) — no Markdown/ADF as source
- [ ] `h2.` / `h3.` sections with blank lines
- [ ] Logs/commands in `{code}` / `{noformat}`
- [ ] Payload valid for MCP schema or REST JSON
- [ ] No secrets, no agent/Cursor meta references

---

## Workflow

### Phase 1: Ticket-Inhalt ermitteln

**Zwei Quellen** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Expliziter Text** im User-Prompt nach `/jira-create` | User hat Beschreibung direkt angegeben |
| 2 | **Chat-Kontext** | Kein expliziter Text — aus bisheriger Konversation ableiten |

**Aus Kontext ableiten — Regeln:**

1. **Problem/Anfrage** klar identifizieren (Was? Warum? Welches System?)
2. **Relevante Details** einbeziehen: Hostnamen, Fehlermeldungen, Zeiträume, betroffene Services
3. **Keine internen Meta-Referenzen** (Tools, Agent, Automatisierung)
4. Professioneller Ton — als hätte der User persönlich das Ticket erstellt
5. Wenn der Kontext **kein klares Ticket-Thema** liefert → **nicht raten**, User fragen

**Bei Unklarheiten → User fragen (Pflicht):**

Wenn Pflichtinformationen fehlen oder mehrdeutig sind, **`AskQuestion`** nutzen (Pflicht — kein Fallback auf Chat-Fragen). Fragen **einzeln oder gebündelt**, je nachdem was fehlt:

**Pflichtfragen (wenn nicht aus Prompt/Kontext ableitbar):**

| Feld | Frage | Optionen (Beispiel) |
|------|-------|---------------------|
| **Summary** | What should the ticket title (summary) be? | Freitext oder Vorschläge aus Kontext |
| **Request Type** | What type of request is this? | Service Request Platform / Platform Incident / Change Request / Update Request / Other |
| **Environment** | Which environment is affected? | Productive / Acceptance / Integration / Development / HQ / Enterprise / Other |
| **Platform** | Which platform? | Transporeon / Ticontract / Mercareon / Tim / ControlPay / All / Other |
| **Component** | Which component area? | Database / Network / Storage / Kubernetes / Monitoring / VMware / Other / … |
| **Description** | What details should the ticket description include? | Freitext |

**AskQuestion-Beispiel** (wenn mehrere Felder unklar):

> I need a few details to create the Jira ticket:

Optionen für Request Type, Environment, Platform, Component — jeweils als Multiple-Choice wo möglich, plus „Other (I'll specify)".

**Nicht erstellen**, solange mindestens Summary, Request Type, Environment, Platform und Component nicht geklärt sind.

### Phase 2: Jira-Zugang prüfen

**Priorität:**

1. Jira MCP prüfen (`GetDynamicTools`, Pattern `jira`)
2. Bei verfügbarem MCP: MCP-Tools für Issue-Erstellung nutzen
3. **Fallback**: REST API via `~/.cursor/.env`

Bei fehlenden Credentials oder Auth-Fehler: User informieren, **nicht** ohne Auth fortfahren.

### Phase 3: Request Type und Pflichtfelder bestimmen

**Standard-Projekt:** `PITOPS` (Platforms and IT Operation)
**Service Desk ID:** `5`

**Request Types** (häufigste — bei Unklarheit User fragen):

| Request Type | ID | Wann verwenden |
|--------------|-----|----------------|
| Service Request Platform | `657` | Standard für Platform-Anfragen, Investigation, Support |
| Platform Incident | `153` | Produktions-Störung, Ausfall |
| Change Request | `155` | Geplante Änderung |
| Update Request | `154` | Update/Patch-Anfrage |
| Service Request | `149` | Allgemeine Service-Anfrage |
| Internal | — (via REST API 2) | Internes Ticket, kein Service Desk |

**Default:** `657` (Service Request Platform), wenn aus Kontext keine andere Kategorie klar ist.

**Pflichtfelder für Request Type 657** (Service Request Platform):

| Feld | Field ID | Format | Beispiel |
|------|----------|--------|----------|
| Summary | `summary` | string, max ~255 Zeichen | `dbamq08.pd.tp.nil - FS corruption alert` |
| Description | `description` | string (**Jira Wiki Markup**) | Detaillierte Beschreibung |
| Environment | `customfield_12929` | multiselect (array) | `[{"id": "13339"}]` = Productive |
| Platform | `customfield_12930` | select | `{"id": "13345"}` = Transporeon |
| Component/s | `components` | array | `[{"id": "20471"}]` = Storage |

**Environment-Werte:**

| Label | ID |
|-------|-----|
| Productive | `13339` |
| Acceptance | `13340` |
| Integration | `13341` |
| Development | `13342` |
| HQ / Enterprise | `13343` |
| Other | `13344` |

**Platform-Werte:**

| Label | ID |
|-------|-----|
| Transporeon | `13345` |
| Ticontract | `13346` |
| Mercareon | `13347` |
| Tim | `13348` |
| ControlPay | `20970` |
| All | `13359` |
| Other | `13349` |

**Component-Werte** (Auswahl — bei Unklarheit User fragen oder aus Kontext ableiten):

| Label | ID |
|-------|-----|
| Database | `14972` |
| Network | `14895` |
| Storage | `20471` |
| VMware | `20470` |
| Kubernetes | `22870` |
| Monitoring | `23874` |
| Server Hardware | `14896` |
| Other | `14894` |
| … | weitere via API abrufbar |

**Feldwerte aus Kontext ableiten:**

- Fehlermeldung auf Produktions-Server → Environment: Productive
- Hostname enthält `.pd.` → meist Productive
- DB/PostgreSQL/MySQL-Thema → Component: Database
- Filesystem/Disk → Component: Storage
- Bei Unsicherheit → **User fragen**, nicht raten

### Phase 4: Ticket-Entwurf zeigen und bestätigen lassen (Pflicht)

**Immer vor dem Erstellen** den vollständigen Entwurf zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch erstellen** — erst nach expliziter Bestätigung.

```markdown
## Ticket draft

| Field | Value |
|-------|-------|
| Project | PITOPS |
| Request Type | Service Request Platform |
| Summary | ... |
| Environment | Productive |
| Platform | Transporeon |
| Component | Storage |

### Description
<full description>

---
Create this ticket in Jira?
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> Create this Jira ticket?

Optionen:

1. **Yes, create ticket** — Ticket anlegen
2. **Edit first** — User gibt Korrekturen, Entwurf anpassen, erneut bestätigen
3. **Cancel** — Nichts erstellen

Bei **Cancel**: Kurz bestätigen, dass kein Ticket erstellt wurde.

### Phase 5: Ticket erstellen (nur nach Bestätigung)

**Formatierung:** Siehe **Content formatting — Jira** — gilt für **MCP und REST** gleichermaßen. Summary plain text; Description **immer Jira Wiki Markup**; Payload via `json.dumps` oder MCP-JSON-Args.

**Service Desk API** (bevorzugt für Request Types):

```bash
source ~/.cursor/.env
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/servicedeskapi/request" \
  -d "$(python3 <<'PYEOF'
import json
payload = {
    "serviceDeskId": "5",
    "requestTypeId": "<REQUEST_TYPE_ID>",
    "requestFieldValues": {
        "summary": "<summary>",
        "description": "<description>",
        "customfield_12929": [{"id": "<environment_id>"}],
        "customfield_12930": {"id": "<platform_id>"},
        "components": [{"id": "<component_id>"}]
    }
}
print(json.dumps(payload))
PYEOF
)"
```

**Fallback — REST API 2** (für Issue Types ohne Service Desk, z.B. Task, Internal):

```bash
curl -s -w "\nHTTP_CODE:%{http_code}\n" -X POST \
  -H "Authorization: Bearer $JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue" \
  -d "$(python3 <<'PYEOF'
import json
payload = {
    "fields": {
        "project": {"key": "PITOPS"},
        "issuetype": {"id": "<ISSUE_TYPE_ID>"},
        "summary": "<summary>",
        "description": "<description>"
    }
}
print(json.dumps(payload))
PYEOF
)"
```

**Issue Type IDs** (REST API 2 Fallback):

| Type | ID |
|------|-----|
| Service Request | `10902` |
| Task | `3` |
| Internal | `11000` |

Bei HTTP 201: Erfolg — `issueKey` aus Response extrahieren.
Bei Fehler: HTTP-Code und Fehlermeldung (ohne Token) melden. Fehlende Pflichtfelder identifizieren und User erneut fragen.

### Phase 6: Ergebnis melden

Bei Erfolg:

```markdown
## Ticket created

| Field | Value |
|-------|-------|
| Key | <KEY> |
| Summary | ... |
| URL | <JIRA_URL>/browse/<KEY> |
```

- Ticket-Key und Browse-URL
- Kurze Bestätigung
- Hinweis auf verwandte Commands:
  - `/jira-read <key>` — Ticket lesen
  - `/jira-comment <key>` — Kommentar hinzufügen

Bei Abbruch:

- Kurz bestätigen, dass kein Ticket erstellt wurde

---

## Description — Best Practices

1. **Problem statement**: Was ist das Problem / was wird angefragt?
2. **Context**: Betroffene Systeme, Hostnamen, Umgebung
3. **Details**: Fehlermeldungen, Logs, Zeitpunkt (wörtlich zitieren)
4. **Impact**: Auswirkung auf Nutzer/Services (falls bekannt)
5. **Steps taken**: Was wurde bereits unternommen (falls aus Kontext bekannt)
6. **Expected outcome**: Was soll das Ticket erreichen?

**Template (Wiki Markup):**

```text
h2. Problem
<what is wrong or what is requested>

h2. Affected Systems
* Host: <hostname>
* Environment: <env>
* Service: <service>

h2. Details
{code}
<error messages, logs, timestamps>
{code}

h2. Impact
<business/operational impact>

h2. Steps Already Taken
* <step 1>
* <step 2>

h2. Expected Outcome
<what resolution is needed>
```

---

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Unklarer Ticket-Inhalt | AskQuestion — nicht raten |
| Pflichtfelder fehlen | User fragen, nicht mit Defaults füllen |
| User bricht ab | Bestätigen, nichts erstellen |
| Auth fehlgeschlagen | Credentials prüfen |
| Validation error (400) | Fehlende/ungültige Felder identifizieren, User fragen |
| Secrets im Entwurf | Warnen, entfernen, erneut bestätigen |
| Duplikat-Verdacht | User warnen wenn ähnliches Ticket im Kontext erwähnt wurde |

## Verwandte Commands

- `/jira-read <ticket-key>` — Ticket lesen
- `/jira-comment <ticket-key>` — Kommentar zu einem Ticket hinzufügen
