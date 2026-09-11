---
name: commit
description: Erstellt einen Git-Commit nach Conventional Commits. Erstellt bei Bedarf einen Feature-Branch vom Default-Branch. Für pitops/ansible niemals auf development/integration/acceptance/production committen — Feature-Branch von development. Optionale Anweisungen oder Chat-Kontext. Bestätigung vor Commit via AskQuestion.
---

# Commit

Erstellt einen sauberen Git-Commit nach [Conventional Commits](https://www.conventionalcommits.org/).
Wird manuell via `/commit [instructions]` ausgelöst.

## ⚙️ Optionale Argumente & Kontext

Alles **nach** `/commit` im User-Prompt ist optionaler Freitext (`[instructions]`). Er kann Leerzeichen enthalten.

**Zwei Quellen** (Priorität):

| Priorität | Quelle | Wann |
|-----------|--------|------|
| 1 | **Expliziter Text** nach `/commit` | User hat Anweisungen direkt angegeben |
| 2 | **Chat-Kontext** | Kein expliziter Text — aus bisheriger Konversation ableiten |

**Typische Anweisungen** (explizit oder aus Kontext):

- Commit-Subject oder vollständige Message-Hinweise (`fix login timeout`, `use type feat scope auth`)
- Welche Dateien/Änderungen einbeziehen oder ausschließen
- Branch-Name-Vorschlag (`branch fix/oauth-redirect`)
- Ob auf Default-Branch ein neuer Feature-Branch erstellt werden soll oder nicht
- Issue-Referenzen (`refs PROJ-456`, `closes #123`)

**Aus Kontext ableiten — Regeln:**

1. Nur verwenden, was im Chat **eindeutig** zur aktuellen Änderung passt
2. Keine Commit-Message erfinden, wenn weder Diff noch Kontext genug Information liefern — dann aus `git diff` ableiten
3. Wenn Kontext und Diff widersprechen → **User fragen**, nicht raten

```
✅ /commit
✅ /commit fix(api): handle null response in webhook retry
✅ /commit only stage auth files, type fix scope auth
✅ /commit refs PROJ-789 — branch fix/webhook-race
```

## Output language

**Always write all user-facing output in English** — status messages, summaries, error messages, and questions to the user. This applies even if the user writes in another language or this command definition is in German.

## Voraussetzungen

- Das aktuelle Verzeichnis ist ein Git-Repository (`git rev-parse --is-inside-work-tree`).
- Es gibt Änderungen zum Committen (staged oder unstaged). Ohne Änderungen: abbrechen und melden.

## Git Safety Protocol (strikt einhalten)

- **Niemals** `git config` ändern
- **Niemals** destruktive Git-Befehle (`push --force`, `reset --hard`, etc.) ohne explizite User-Anfrage
- **Niemals** Hooks überspringen (`--no-verify`, `--no-gpg-sign`, etc.)
- **Niemals** `git commit --amend`, außer alle Bedingungen aus den User-Rules sind erfüllt
- **Keine** Secrets committen (`.env`, `credentials.json`, API-Keys, Tokens). Warnen und ausschließen

---

## Workflow

### Phase 0: Anweisungen & Kontext auswerten

1. Text nach `/commit` extrahieren — falls vorhanden, als **explizite Anweisungen** behandeln
2. Falls leer: **Chat-Kontext** auf Commit-relevante Hinweise prüfen (besprochene Fixes, Ticket-Keys, gewünschter Scope)
3. Anweisungen in konkrete Entscheidungen übersetzen:
   - Commit-Type / Scope / Subject-Vorgabe
   - Dateiauswahl (nur bestimmte Pfade stagen)
   - Branch-Strategie (neuer Branch ja/nein, Name)
   - Footer (Issue-Referenzen)
4. Unklare oder widersprüchliche Anweisungen → **User fragen** vor dem Commit

### Phase 1: Repository-Status prüfen

Führe **parallel** aus:

```bash
git status
git diff
git diff --staged
git log --oneline -10
```

Prüfe zusätzlich:

```bash
git rev-parse --abbrev-ref HEAD
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null || true
git remote -v
```

**Default-Branch ermitteln** (in dieser Reihenfolge):

1. `git symbolic-ref refs/remotes/origin/HEAD` → z.B. `refs/remotes/origin/main`
2. Fallback: `main`, dann `master`, dann `develop`
3. Wenn unklar: User fragen

**Repo-Erkennung — pitops/ansible:**

Nach `git remote -v` prüfen, ob **irgendeine** Remote-URL auf das Ansible-Repo zeigt:

- SSH: `git@gitlab.office.transporeon.com:pitops/ansible.git` (auch ohne `.git`)
- HTTPS: Host `gitlab.office.transporeon.com` und Pfad `…/pitops/ansible`

Match → **Ansible-Modus** aktiv (Phase 2 Sonderregeln). Kein Match → normale Default-Branch-Logik.

### Phase 2: Branch-Strategie

#### Standard (nicht Ansible)

| Situation | Aktion |
|-----------|--------|
| Aktueller Branch = Default-Branch | **Feature-Branch erstellen** (siehe unten) |
| Aktueller Branch ≠ Default-Branch | Auf aktuellem Branch committen |
| Detached HEAD | User fragen, wie fortzufahren |

**Explizite Branch-Anweisung** (aus Phase 0): Wenn der User einen Branch-Namen vorgegeben hat (`branch fix/foo`), diesen verwenden — auch auf Default-Branch. Wenn der User „commit on current branch" o.ä. verlangt hat, keinen neuen Branch erstellen.

#### Ansible-Modus (`pitops/ansible`)

**Geschützte Branches** (niemals darauf committen): `development`, `integration`, `acceptance`, `production`.

Diese Regel ist **nicht überschreibbar** — auch nicht durch explizite Anweisungen wie „commit on development“ / „commit on current branch“. Solche Anweisungen **ablehnen**, kurz erklären, und weiterhin einen Feature-Branch von `development` planen (oder abbrechen, wenn der User das wählt).

| Situation | Aktion |
|-----------|--------|
| Auf `development` | Feature-Branch von `development` erstellen (User-Branchname oder abgeleitet) |
| Auf `integration`, `acceptance` oder `production` | **AskQuestion** (siehe unten) — nicht automatisch wechseln |
| Auf anderem Branch (z.B. bestehender Feature-Branch) | Auf aktuellem Branch committen, sofern nicht geschützt |
| Detached HEAD | User fragen, wie fortzufahren |
| User fordert Commit auf geschütztem Branch | Ablehnen; Feature-Branch von `development` anbieten |

**AskQuestion** wenn aktuell auf `integration` / `acceptance` / `production`:

> Current branch `<branch>` is protected in pitops/ansible. How to proceed?

Optionen:

1. **Branch from development** — uncommitted Changes behalten, `development` aktualisieren/auschecken soweit nötig, Feature-Branch von `development` anlegen, dann Commit-Plan fortsetzen
2. **Cancel** — nichts ändern

**Feature-Branch-Basis in Ansible-Modus:** immer von `development` (lokal syncen falls nötig: `git fetch origin development`, dann von `origin/development` bzw. lokalem `development` branchen — nicht von `integration`/`acceptance`/`production`).

**Feature-Branch erstellen** (Standard: nur auf Default-Branch; Ansible: auf jedem geschützten Branch bzw. wenn neuer Branch geplant):

1. Branch-Namen aus Anweisungen, Änderungen oder Kontext ableiten (nicht generisch wie `feature` oder `fix`)
2. Format: `<type>/<kurze-beschreibung>` — z.B. `feat/user-auth`, `fix/login-timeout`
3. `type` aus dem dominanten Conventional-Commit-Typ der Änderung wählen
4. Kleinbuchstaben, Bindestriche, max. ~50 Zeichen
5. Branch erstellen und wechseln — **erst nach Bestätigung in Phase 5**:

```bash
# Standard (vom aktuellen Default-Branch-HEAD):
git checkout -b <branch-name>

# Ansible (explizit von development):
git fetch origin development
git checkout -b <branch-name> origin/development
```

### Phase 3: Änderungen planen (noch nicht stagen)

1. **Secrets-Check**: Keine `.env`, Credentials, Keys, Passwörter stagen
2. Nur relevante Dateien einplanen — keine unrelated changes mitschleppen
3. **Dateiauswahl aus Anweisungen/Kontext**: Wenn nur bestimmte Pfade genannt wurden, nur diese einplanen; explizit ausgeschlossene Pfade nicht stagen
4. Wenn nur Teile einer Datei relevant sind: gezielt stagen (`git add -p` nur wenn nötig, **nach Bestätigung**)

**Noch kein `git add`** — nur die geplante Dateiliste für die Vorschau in Phase 5 festhalten.

### Phase 4: Commit-Message (Conventional Commits)

#### Format

```
<type>(<scope>): <description>

[optional body — erklärt WARUM, nicht WAS]

[optional footer — BREAKING CHANGE, Issue-Referenzen]
```

#### Erlaubte Types

| Type | Wann |
|------|------|
| `feat` | Neue Funktionalität |
| `fix` | Bugfix |
| `docs` | Nur Dokumentation |
| `style` | Formatierung, Whitespace (keine Logik) |
| `refactor` | Umstrukturierung ohne Feature/Fix |
| `perf` | Performance-Verbesserung |
| `test` | Tests hinzufügen/ändern |
| `build` | Build-System, Dependencies |
| `ci` | CI/CD-Konfiguration |
| `chore` | Wartung, kein src/test-Impact |
| `revert` | Revert eines früheren Commits |

#### Regeln

- **Subject**: Imperativ, Kleinbuchstaben am Anfang, max. **72 Zeichen**, spezifisch (nicht „fix bug")
- **Scope**: Optional, lowercase, konsistent im Projekt (z.B. `auth`, `api`, `ui`)
- **Body**: Erklärt **warum** die Änderung nötig war; Leerzeile nach Subject
- **Breaking Changes**: `!` nach type/scope ODER Footer `BREAKING CHANGE: <beschreibung>`
- **Issue-Referenzen**: Im Footer, z.B. `Closes #123`, `Refs PROJ-456`
- **Ein Commit = eine logische Änderung**. Gemischte Types splitten in mehrere Commits

#### Beispiele

```
feat(auth): add JWT refresh token rotation

Implement sliding expiration to reduce re-login friction while
keeping stolen tokens short-lived.
```

```
fix(api): prevent race condition in webhook retry queue

Refs PROJ-789
```

```
feat!: remove legacy v1 API endpoints

BREAKING CHANGE: /api/v1/* routes are removed. Migrate to /api/v2/*.
```

#### Message aus Anweisungen, Kontext und Diff ableiten

1. Wenn der User eine vollständige oder teilweise Message vorgegeben hat → **übernehmen** (Conventional-Commits-Format prüfen, ggf. minimal korrigieren)
2. `git diff --staged` analysieren
3. Dominanten Type und Scope bestimmen — **Anweisungen/Kontext haben Vorrang** vor reiner Diff-Heuristik
4. Subject aus dem **Hauptziel** der Änderung formulieren
5. Body nur wenn der Grund nicht offensichtlich ist oder der Kontext ein „Warum" liefert
6. Issue-Referenzen aus Anweisungen/Kontext in den Footer übernehmen
7. Commit-Stil aus `git log` des Projekts übernehmen

### Phase 5: Entwurf zeigen und bestätigen lassen (Pflicht)

**Vor jeder schreibenden Git-Operation** (`git checkout -b`, `git add`, `git commit`) den vollständigen Plan zeigen und mit **`AskQuestion`** auf Bestätigung warten.

**Nicht automatisch committen** — erst nach expliziter Bestätigung.

```markdown
## Commit plan

| Field | Value |
|-------|-------|
| Current branch | `<branch>` |
| New branch | `<branch-name>` or *(none — commit on current branch)* |
| Base (ansible) | `origin/development` *(only in ansible mode when creating a branch)* |
| Files to stage | `<n>` files |
| Excluded | `<paths or —>` |

### Files
- `path/to/file1`
- `path/to/file2`

### Commit message
```
<type>(<scope>): <description>

<optional body>

<optional footer>
```

### Actions
1. Create and switch to branch `<branch-name>` *(if planned)*
2. Stage listed files
3. Run `git commit` with the message above
```

**AskQuestion** (Pflicht — kein Fallback auf implizite Fortsetzung):

> Proceed with this commit?

Optionen:

1. **Yes, commit** — Branch erstellen (falls geplant), stagen, committen
2. **Edit first** — User gibt Korrekturen (Message, Dateien, Branch); Entwurf anpassen, erneut bestätigen
3. **Cancel** — Nichts ändern, keine Git-Operationen ausführen

Bei **Cancel**: Kurz bestätigen, dass nichts committed wurde.

### Phase 6: Ausführen (nur nach Bestätigung)

1. Branch erstellen/wechseln (falls geplant)
2. Dateien stagen:

```bash
git add <relevante-dateien>
```

3. Commit ausführen:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <description>

<optional body>

EOF
)"
```

### Phase 7: Ergebnis melden

- Branch-Name (neu erstellt oder bestehend)
- Commit-Hash (`git rev-parse --short HEAD`)
- Commit-Message (vollständig)
- Anzahl geänderter Dateien
- Hinweis auf `/merge-request`, wenn Push/MR als nächster Schritt sinnvoll ist

---

## Fehlerbehandlung

| Problem | Aktion |
|---------|--------|
| Kein Git-Repo | Abbrechen, User informieren |
| Keine Änderungen | Abbrechen |
| User bricht Bestätigung ab | Nichts committen, Plan verworfen |
| Pre-commit Hook schlägt fehl | Hook-Fehler zeigen, **nicht** amend — Problem fixen, **neuer** Commit |
| Merge-Konflikte | Nicht committen; User informieren |
| Secrets in Diff | Nicht committen; Dateien warnen |
| Ansible: Commit auf geschütztem Branch verlangt | Ablehnen; Feature-Branch von `development` anbieten oder abbrechen |
| Ansible: auf integration/acceptance/production | AskQuestion — von development branchen oder Cancel |

## Verwandte Commands

- `/merge-request [instructions]` — Branch pushen und MR/PR erstellen
- `/merge-request-review [id] [focus]` — Code Review eines MR
- `/merge-request-fix [id] [instructions]` — Review-Kommentare umsetzen
