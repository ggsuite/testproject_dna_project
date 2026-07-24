---
name: review
description: Fuehrt einen vollstaendigen Review des aktuellen Branches in einem Grace-Cloud-/GG-Projekt durch. Prueft zuerst das Tooling (`gg do check`, `gg do test`, `dart pub upgrade --tighten`, `gg_dna sync --check`, `gg_dna apply-conventions --check`), bietet Fixes interaktiv an, und macht danach einen LLM-Review gegen die Conventions, plus Pruefung auf redundanten/unuebersichtlichen Code, falsche Dokumentation, Performance und Sicherheit. Findings werden als strukturierter Bericht ausgegeben, jeder Fix-Vorschlag wird vor Anwendung einzeln bestaetigt. Verwende diesen Skill automatisch, wenn der Nutzer sinngemaess sagt "review", "pruefe meinen Branch", "ist das mergebar", "code review", "vor dem Mergen pruefen" — insbesondere wenn es um Grace Cloud, GG, `gg_*`, `kidney_*` oder `ds_*` geht.
---

# Grace-Cloud Code Review

Du fuehrst einen vollstaendigen Review des aktuellen Branches durch. Der Skill laeuft in **vier Phasen**: zuerst werden alle deterministischen Tooling-Checks gruen gezogen, dann der LLM-Review gemacht, dann der Bericht zusammengestellt, und am Ende werden Fixes interaktiv angewendet.

**Goldene Regeln:**

- Niemals ungefragt aendern. Jeder Schritt, der Dateien anfasst, wird vorher angekuendigt und einzeln bestaetigt.
- Keine Erfindungen. Wenn ein Befehl nicht verfuegbar ist (z. B. `gg` nicht installiert), das Finding melden und weiterlaufen, statt zu mocken.
- Tooling-Wahrheit schlaegt LLM-Geschmack. Was der Analyzer/Test sagt, gilt; subjektive Punkte sind als "Suggestion" markiert.
- Sprache: **Deutsch** in Berichten und Prompts, ausser der Nutzer fragt explizit auf Englisch.

---

## 0. Scope ermitteln

Bevor irgendetwas geprueft wird, klaere und melde den Scope:

1. **Repo-Root** finden: `git rev-parse --show-toplevel`. Wenn das fehlschlaegt, melde "kein Git-Repo" und brich ab.
2. **Branch-Basis** ermitteln:
   - Default-Branch des Remotes auslesen: `git symbolic-ref refs/remotes/origin/HEAD` → typischerweise `refs/remotes/origin/main`.
   - Wenn das fehlschlaegt, Fallback `main`. Wenn `main` nicht existiert, `master` probieren.
   - Wenn keine Basis ermittelbar ist, den Nutzer nach dem Basis-Branch fragen.
3. **Diff-Range** = `<basis>...HEAD` (drei Punkte, also gegen den Merge-Base).
4. **Geaenderte Dateien** ermitteln: `git diff --name-status <basis>...HEAD`, plus `git status --porcelain` fuer untracked / nicht-commited.
5. **Kidney-Workspace?** Wenn ein `.master/`-Ordner im aktuellen Pfad oder einem Vorgaenger existiert, ist das ein Workspace. Liste die Sub-Repos auf und kuendige an, dass die Phasen 1–3 pro Sub-Repo seriell laufen, der Bericht in Phase 3 aber zusammengefasst wird.

Gib eine kurze Scope-Meldung aus, etwa:

```
Review-Scope:
  Repo:     <abs-path>
  Branch:   <feature-branch> vs <basis-branch>
  Dateien:  N geaendert (+L / -L), M untracked
  Modus:    Single-Repo  |  Kidney-Workspace mit K Sub-Repos: …
```

---

## 1. Phase 1 — Tooling-Fixes (interaktiv, blockierend)

Diese Phase laeuft **so lange, bis alle Checks gruen sind**. Bei Fehlern werden Fix-Vorschlaege gemacht; jeder Fix wird vom Nutzer einzeln bestaetigt. Erst dann geht es weiter.

Reihenfolge ist wichtig — billige/lokale Checks zuerst, damit man nicht 2 Minuten auf Tests wartet, nur damit hinterher `dart format` rot ist.

### 1.1 `dart pub upgrade --tighten`

- Vorher kurz `git status --porcelain pubspec.yaml pubspec.lock` pruefen — wenn dirty, den Nutzer warnen, dass jetzt zusaetzliche Aenderungen kommen koennen.
- Ausfuehren:
  ```bash
  dart pub upgrade --tighten
  ```
- Wenn `pubspec.yaml`/`pubspec.lock` veraendert wurden:
  - Diff zeigen (`git diff -- pubspec.yaml pubspec.lock`).
  - Den Nutzer fragen, ob diese Aenderungen Teil des Commits sein sollen.
  - Bei "ja" am Ende der Phase einen Commit-Vorschlag vorbereiten (Message-Vorschlag: `chore: dart pub upgrade --tighten`), aber **nicht** sofort committen — das passiert gebuendelt am Ende von Phase 4.
- Wenn `--tighten` mit Fehler abbricht (z. B. Konflikte): Fehler als Finding aufnehmen, Fix-Vorschlag generieren, Nutzer bestaetigen lassen.

### 1.2 `gg_dna sync --check` und `gg_dna apply-conventions --check`

- `gg_dna sync --check` — pruefen, ob `dna/` aktuell ist.
- `gg_dna apply-conventions --check` — pruefen, ob `.claude/conventions/` + `CLAUDE.md`-Block aktuell sind.
- Bei Fehler: Fix ist `gg_dna sync` bzw. `gg_dna apply-conventions` ohne `--check`. Nutzer fragen, dann ausfuehren.
- Wenn `gg_dna` nicht installiert ist: als Suggestion in den spaeteren Bericht uebernehmen, nicht blockieren.

### 1.3 `gg do check`

- Ausfuehren:
  ```bash
  gg do check
  ```
- Bei Fehler:
  - Ausgabe parsen. Typische Befunde: Formatter-Diff, Analyzer-Warnings, fehlende Doc-Comments, ungenutzte Imports.
  - **Pro Befund** einen konkreten Fix-Vorschlag generieren (Patch oder Befehl, z. B. `dart format .`).
  - Patches mit `AskUserQuestion` einzeln zur Bestaetigung anbieten ("apply / skip / edit").
  - Nach jedem angewendeten Fix `gg do check` erneut ausfuehren, bis gruen.
- Wenn `gg` nicht installiert ist: als Blocker melden mit Hinweis `dart pub global activate gg`, abbrechen lassen oder Tooling-Phase ueberspringen (Nutzer fragen).

### 1.4 `gg do test`

- Ausfuehren:
  ```bash
  gg do test
  ```
- Bei Fehler:
  - Fehlschlagende Tests einzeln auflisten (Datei, Testname, Fehlermeldung).
  - Pro Test entscheiden: ist der Test falsch oder der Code? Fix-Vorschlag konkret als Patch generieren.
  - Nutzer einzeln bestaetigen lassen, danach `gg do test` erneut laufen lassen, bis gruen.
- **Coverage**: wenn die Test-Ausgabe einen Coverage-Wert liefert und er unter **100 %** liegt, das als Blocker aufnehmen (gg-Konvention). Die nicht abgedeckten Zeilen lokalisieren und in Phase 2 als Finding behandeln.

### 1.5 Phase-1-Abschluss

Wenn alle Checks gruen sind, kurz melden:

```
Phase 1 abgeschlossen — Tooling ist gruen.
  pub upgrade --tighten: angewendet (Pubspec geaendert: ja/nein)
  gg_dna sync/apply-conventions --check: ok
  gg do check: ok
  gg do test: ok (coverage NN%)
```

Erst danach in Phase 2 gehen.

---

## 2. Phase 2 — LLM-Review

Jetzt den eigentlichen Code-Review machen, **ausschliesslich auf den geaenderten Dateien aus Phase 0**. Andere Dateien werden nur gelesen, wenn sie als Kontext fuer ein Finding noetig sind (z. B. Callers einer geaenderten Funktion).

Vorgehen pro geaenderter Datei:

1. Diff lesen (`git diff <basis>...HEAD -- <datei>`).
2. Die volle Datei lesen, um den Kontext der Aenderung zu verstehen.
3. Die geaenderten Stellen gegen jede der folgenden Achsen pruefen.

### 2.1 Conventions

Lade und referenziere explizit:

- `.claude/conventions/code-conventions.md`
- `.claude/conventions/test-conventions.md`
- `.claude/conventions/documentation-conventions.md`

Jede Convention-Verletzung wird mit Zitat aus der Convention-Datei untermauert — das schuetzt vor Geschmacks-Findings.

### 2.2 Redundanz / DRY

- Identische oder fast identische Codebloecke im Diff oder in der unmittelbaren Nachbarschaft.
- Funktionen, die im Repo bereits existieren und nicht wiederverwendet werden.
- Doppelte Imports, doppelte Test-Setups, kopierte Konstanten.

### 2.3 Uebersichtlichkeit

- Funktionen > ~40 Zeilen — Vorschlag zur Extraktion, aber nur wenn die Extraktion klar besser liest.
- Verschachtelung > 3 Ebenen (`if`/`for`/`try`) — Early-Return-Vorschlag.
- Namen, die nicht zur Convention passen oder dem Zweck nicht entsprechen.
- Magic Numbers / Magic Strings, die als benannte Konstanten klarer waeren.

### 2.4 Dokumentation

- **Korrektheit**: Dartdoc-Kommentare gegen die tatsaechliche Signatur abgleichen. Parameter umbenannt, aber Doc nicht? Return-Typ geaendert, aber Doc beschreibt den alten? Exceptions dokumentiert, die gar nicht mehr geworfen werden?
- **Vollstaendigkeit**: oeffentliche API ohne Dartdoc → Blocker (analog `documentation-conventions.md`).
- **README/CHANGELOG**: wenn das oeffentliche Verhalten sich geaendert hat, sollten README/CHANGELOG das reflektieren. Pruefen, ob sie im Diff mitgeaendert wurden.

### 2.5 Performance

Pruefe gezielt auf typische Dart-Fallen — nur was im Diff steht oder direkt davon getriggert wird:

- `await` in einer Schleife, das parallelisierbar waere (`Future.wait`).
- Wiederholtes `toList()`/`.where().toList()` in heissen Pfaden.
- `List.add` in engen Loops, wo `List.generate` oder ein vor-allokierter Buffer besser waere.
- Stream-Subscriptions ohne `cancel`, Timer ohne `cancel`, `StreamController` ohne `close`.
- Synchrone IO (`readAsStringSync`, `existsSync`) in async-Code-Pfaden.
- Wiederholtes Parsen/Berechnen, das ausserhalb der Schleife gehoeren wuerde.

Nicht spekulieren — ein Finding nur, wenn der Hot Path plausibel ist (z. B. Code laeuft pro Frame, pro Request, pro Element einer grossen Collection).

### 2.6 Sicherheit

- **Secrets im Diff**: `grep` auf `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`, plus Heuristik fuer JWT/Base64-aehnliche lange Strings in neuen Zeilen.
- **`Process.run` / `Process.start`** mit interpoliertem User-Input → Shell-Injection-Risiko.
- **Input-Validierung** an Systemgrenzen (HTTP-Handler, CLI-args, Datei-Pfade aus externer Quelle).
- **Neue Dependencies** in `pubspec.yaml`: ist das Paket aktiv gepflegt? Bekannte Maintainer? Plausibles Pub-Score? Wenn nicht beurteilbar, als Suggestion melden, nicht als Blocker.
- **`dart:io`-Datei-Pfade**, die ohne Normalisierung aus externer Quelle kommen → Path-Traversal-Risiko.

---

## 3. Phase 3 — Bericht

Sammle alle Findings aus Phase 1 und Phase 2 und gib **einen einzigen strukturierten Bericht** aus, bevor irgendein Fix angewendet wird. Klassifikation:

- **Blocker** — verhindern Merge. Tooling-Fehler (auch wenn in Phase 1 schon gefixt: hier dokumentieren), Coverage < 100 %, Security-Findings mit klarer Risiko-Begruendung, fehlende Dartdoc auf oeffentlicher API, Convention-Verletzungen.
- **Suggestions** — sollten gefixt werden, aber kein Hard-Stop. DRY/Performance/Uebersichtlichkeit mit klarer Begruendung, README/CHANGELOG-Updates.
- **Nits** — Stilfragen, optional. Naming-Mikro-Optimierungen, leichte Lesbarkeit.

Format:

```markdown
## Review: <branch> vs <basis>

**Tooling**
- gg do check: PASS / FAIL (gefixt in Phase 1: ja/nein)
- gg do test:  PASS / FAIL (coverage: NN%)
- dart pub upgrade --tighten: pubspec geaendert (ja/nein)
- gg_dna sync --check / apply-conventions --check: PASS / FAIL

**Statistik**
- Dateien: N geaendert, +L / -L
- Findings: X Blocker, Y Suggestions, Z Nits

---

### Blocker

#### B1. <kurzer Titel> — `<datei>:<zeile>`
**Convention/Achse**: <code-conventions.md §… | Performance | Security | …>
**Befund**: <ein bis drei Saetze>
**Patch**:
```diff
- alte zeile
+ neue zeile
```

#### B2. …

### Suggestions

#### S1. …

### Nits

#### N1. …
```

Wenn es keine Findings einer Kategorie gibt, die Sektion mit `(keine)` markieren statt komplett weglassen — der Nutzer sieht so, dass tatsaechlich geprueft wurde.

---

## 4. Phase 4 — Interaktiver Fix-Loop

Nach dem Bericht den Nutzer fragen, wie er fortfahren will. Drei Modi anbieten:

1. **Alle Blocker durchgehen** — pro Blocker Patch zeigen, "apply / skip / edit" via `AskUserQuestion`. `edit` heisst: der Nutzer beschreibt eine Alternative, du schlaegst einen neuen Patch vor.
2. **Cherry-Pick** — der Nutzer waehlt aus der gesamten Liste (Blocker + Suggestions + Nits) per Nummer aus, welche gefixt werden sollen.
3. **Abbrechen** — Bericht steht, Nutzer fixt selbst.

Wenn Patches angewendet wurden:

1. **Regressionspruefung**: `gg do check` und `gg do test` erneut laufen lassen. Wenn rot, melde es und biete einen Mini-Fix-Loop an (zurueck in Phase 1).
2. **Commit-Vorschlag**: bereite eine Commit-Message vor, die zusammenfasst, was im Review gefixt wurde. Wenn `pub upgrade --tighten` Pubspec geaendert hat, separater Commit-Vorschlag dafuer. Beispiel:
   ```
   review: fix blockers from /review run

   - <B1-Titel>
   - <B2-Titel>
   - …
   ```
   Den Vorschlag zeigen, aber **nicht** ungefragt committen — der Nutzer bestaetigt explizit.
3. **Push**: niemals von selbst pushen.

---

## Wichtig

- **Niemals** ohne Bestaetigung Dateien aendern, committen, pushen oder Tickets schliessen.
- **Niemals** Tooling-Fehler als gefixt melden, ohne den Check erneut laufen gelassen zu haben.
- **Niemals** ein Finding "erfinden", nur damit die Sektion nicht leer ist. Leere Sektionen sind ein gutes Zeichen.
- **Niemals** Performance-/Security-Findings ohne konkrete Risiko-/Hot-Path-Begruendung als Blocker einstufen — sonst werden die Reports nutzlos.
- Bei Kidney-Workspace: Phasen 1–2 pro Sub-Repo seriell, Phase 3 gesammelt, Phase 4 pro Sub-Repo separat (sonst werden Patches in falsche Repos geschrieben).
- Wenn der Nutzer Teile ueberspringen will (z. B. "nur Phase 2, ich hab Tooling selbst geprueft"), das respektieren und im Bericht oben dokumentieren ("Phase 1 uebersprungen auf Wunsch").
