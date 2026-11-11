# Modul 09 - Git in CI-CD und Automation

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (9.1 bis 9.8), danach folgen „Übungen zu Modul 09“ (8 Aufgaben) und „Lösungen zu Modul 09“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 2

> Die Modulübersicht und die Summe der Lerneinheiten stimmen überein (acht Einheiten zu je 1,25 h). Prioritäten der Einheiten: 9.1 bis 9.6 Prio 2, 9.7 und 9.8 Prio 3. Der Dateiname ersetzt „CI/CD“ durch „CI-CD“, weil der Schrägstrich in Windows-Dateinamen nicht erlaubt ist.

**Lernziele des Moduls:** Git-Events, Hooks, Pipeline-Trigger und automatisierte Checks integrieren; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Continuous Integration und Continuous Delivery (CI/CD) sind ohne Git kaum denkbar: Ein Push, ein Pull Request oder ein Tag ist das Ereignis, das eine Pipeline startet, und der Commit-Hash ist die Kennung, an der sich alle Ergebnisse festmachen. Dieses Modul zeigt, wie man Git-Ereignisse nutzt, um Qualität automatisch zu sichern, und wie man typische Probleme an dieser Nahtstelle erkennt und behebt.

| **Nr.** | **Lerneinheit**           | **Dauer (h)** | **Prio** |
|:--------|:--------------------------|--------------:|:--------:|
| 9.1     | Pipeline Trigger          |          1,25 |    2     |
| 9.2     | Status Checks             |          1,25 |    2     |
| 9.3     | Git Hooks                 |          1,25 |    2     |
| 9.4     | Versionierung aus Git     |          1,25 |    2     |
| 9.5     | Changelog Automation      |          1,25 |    2     |
| 9.6     | Monorepo Pipelines        |          1,25 |    2     |
| 9.7     | Bot- und Service Accounts |          1,25 |    3     |
| 9.8     | CI/CD Troubleshooting     |          1,25 |    3     |
|         | **Summe**                 |      **10,0** |          |

> **Hinweis zu den Beispielen in diesem Modul**
>
> Pipeline-Definitionen (YAML) lassen sich ohne die jeweilige Plattform nicht ausführen. Sie werden deshalb als Konfigurationsbeispiele gezeigt und sind nur auf Syntax geprüft. Alles, was *Git* tut, also Diffs, Hooks, Prüfskripte, Versionsberechnung und Diagnose, ist dagegen als Shell-Skript ausgeführt und getestet. Dieser Schnitt ist auch inhaltlich sinnvoll: Die Logik gehört in Skripte, die man lokal testen kann, und die Pipeline ruft sie nur auf.

## 9.1 Pipeline Trigger

Eine Pipeline startet nicht von selbst, sondern wegen eines *Ereignisses*. Die vier wichtigsten Ereignisse stammen direkt aus Git. Ein *Push* auf einen Branch meldet neue Commits, ein *Pull Request* oder Merge Request meldet, dass eine Änderung zur Aufnahme vorgeschlagen wird, ein neuer *Tag* meldet ein Release, und ein *Zeitplan* (Schedule) startet unabhängig von Änderungen, zum Beispiel für nächtliche Prüfungen. Jedes Ereignis liefert der Pipeline einen Kontext: welcher Commit, welcher Branch oder Tag, und welche Dateien sich geändert haben. Die Kunst des Trigger-Designs besteht darin, die Pipeline genau dann zu starten, wenn es etwas zu prüfen gibt, und sonst nicht. Unnötige Läufe kosten Zeit und Geld, verstopfen die Runner und trainieren Teams darauf, rote und grüne Ergebnisse zu ignorieren.

Die Dokumentation zur Workflow-Syntax von GitHub (abgerufen am 28.10.2026) beschreibt die gängigen Filter. Für Pushes lassen sich `branches` und `tags` als Glob-Muster auf Referenznamen angeben; definiert man nur `tags`, läuft der Workflow nicht für Branch-Pushes und umgekehrt. Mit `paths` und `paths-ignore` filtert man nach geänderten Dateien: `paths` startet, wenn mindestens eine geänderte Datei passt, `paths-ignore` überspringt den Lauf nur, wenn *alle* geänderten Dateien passen. Kombiniert man Branch- und Pfadfilter, müssen beide erfüllt sein. Zwei Details sind besonders wichtig: Pfadfilter werden bei Tag-Pushes laut Dokumentation nicht ausgewertet, ein Release-Tag startet also immer, und die Dateiliste wird bei Pushes mit einem Zwei-Punkte-Vergleich, bei Pull Requests mit einem Drei-Punkte-Vergleich ermittelt. Ein Hinweis aus der Dokumentation: Ein durch Pfadfilter übersprungener Workflow lässt seine Prüfungen im Status „Pending“, und das kann den Merge blockieren, wenn diese Prüfungen als Pflicht eingestellt sind. Für GitLab beschreibt ein Blogbeitrag eines Drittanbieters (abgerufen am 28.10.2026) vergleichbare Mittel: `rules:if`, `rules:changes`, `workflow:rules`, wobei die Regeln der Reihe nach ausgewertet werden und die erste zutreffende gewinnt.

Der Unterschied zwischen Zwei- und Drei-Punkte-Vergleich lässt sich mit Git selbst nachvollziehen. Das erste Beispiel baut eine Lage nach, in der ein Feature-Branch einen Push erhält, während `main` inzwischen weitergelaufen ist, und zeigt, welche Dateiliste die beiden Sichten liefern. Danach folgt ein kleines Entscheidungsskript, das aus der Dateiliste ableitet, ob die Pipeline nötig ist.

```bash
git init -q -b main trg && cd trg
git config user.name Dev; git config user.email dev@example.org
mkdir -p src docs && echo 1 > src/app.py && echo 1 > docs/a.md && git add . && git commit -q -m "Basis"
git switch -q -c feature/x
echo 2 >> src/app.py && git commit -q -am "feat: Code"
VOR=$(git rev-parse HEAD)
git switch -q main
echo 2 >> docs/a.md && git commit -q -am "docs: Anleitung auf main"
git switch -q feature/x
echo 1 > src/neu.py && git add . && git commit -q -m "feat: neue Datei"
NACH=$(git rev-parse HEAD)

echo "--- Push-Sicht (zwei Punkte, Stand vor und nach dem Push)"
git diff --name-only $VOR..$NACH
echo "--- Pull-Request-Sicht (drei Punkte, seit der Abzweigung)"
git diff --name-only main...feature/x
echo "--- zum Vergleich: zwei Punkte gegen main (nicht die PR-Sicht)"
git diff --name-only main..feature/x

cat > should-run.sh <<'EOF'
#!/bin/sh
# Entscheidet anhand der geaenderten Dateien, ob die Pipeline laufen muss
# Aufruf: should-run.sh <basis> <ziel>
RELEVANT='src/* tests/* pom.xml package.json'
geaendert=$(git diff --name-only "$1...$2")
for datei in $geaendert; do
  for muster in $RELEVANT; do
    case "$datei" in
      $muster) echo "Pipeline noetig: $datei"; exit 0 ;;
    esac
  done
done
echo "Pipeline kann uebersprungen werden"
exit 1
EOF
sh should-run.sh main feature/x; echo "Exit: $?"
git switch -q -c nur-doku main~1
echo 3 >> docs/a.md && git commit -q -am "docs: noch eine"
sh should-run.sh main nur-doku; echo "Exit: $?"
```

Die Push-Sicht enthält nur die Datei, die der Push neu gebracht hat (`src/neu.py`). Die Pull-Request-Sicht enthält alles, was der Feature-Branch seit der Abzweigung verändert hat (`src/app.py` und `src/neu.py`) und ignoriert, was in der Zwischenzeit auf `main` passiert ist. Der dritte Vergleich ist der naheliegende, aber falsche Ansatz, zwei Branch-Spitzen mit zwei Punkten zu vergleichen: Er zeigt zusätzlich die Änderung der Dokumentation, die gar nicht aus dem Feature stammt, nämlich als „Rückwärtsänderung“. Genau deshalb verwendet man in eigenen Skripten für die Pull-Request-Sicht drei Punkte (`main...feature/x`), was intern die Abzweigung mit `git merge-base` bestimmt. Das Entscheidungsskript wertet die Liste gegen Muster aus und beendet sich mit Exit-Code 0 (Pipeline nötig) oder 1 (kann entfallen). Der Zweig, der nur Dokumentation ändert, löst keine Pipeline aus.

Für eine Pipeline-Definition selbst sieht das Ganze so aus. Das folgende Beispiel für GitHub Actions ist nicht lokal ausführbar und wurde nur auf gültiges YAML geprüft; die Schlüssel entsprechen der oben zitierten Dokumentation. Es startet bei Pushes auf `main` und bei Tags der Form `v*`, bei Pull Requests nur, wenn relevante Pfade betroffen sind, und bricht ältere Läufe desselben Branches ab.

**Listing:** Trigger-Konfiguration (GitHub Actions, nur Syntax geprueft)

```yaml
name: ci
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'
  schedule:
    - cron: '17 3 * * 1'
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sh test.sh
```

Die Gruppe in `concurrency` enthält Workflow-Namen und Referenz; mit `cancel-in-progress` wird ein laufender Lauf abgebrochen, wenn derselbe Branch einen neuen Push bekommt. Laut Dokumentation können Gruppen-Ausdrücke Kontexte wie `github` nutzen, und mit dem Workflow-Namen im Gruppennamen begrenzt man das Abbrechen auf denselben Workflow. Bei Tags ist das Abbrechen oft unerwünscht; die Dokumentation erlaubt deshalb, `cancel-in-progress` als Ausdruck zu formulieren, um Release-Läufe auszunehmen. Der Block `permissions` setzt die Rechte des automatisch erzeugten Tokens auf Lesen, wie in Modul 08 beschrieben. Der `cron`-Eintrag startet jeden Montag um 03:17 Uhr (UTC), eine ungerade Minute ist üblich, damit nicht alle Pipelines zur vollen Stunde starten.

### Typische Fehler und Stolperfallen.

Falsch ist, jede Pipeline bei jedem Push und jedem Pull Request vollständig zu starten, ohne Filter und ohne Abbruch veralteter Läufe. Richtig ist eine bewusste Zuordnung: schnelle Prüfungen bei jedem Pull Request, aufwendige Prüfungen bei Merge auf `main`, Release-Schritte nur bei Tags. Zweiter Fehler: Pfadfilter für Pflichtprüfungen. Wird ein als Pflicht eingestellter Check wegen des Pfadfilters nicht gestartet, bleibt er ausstehend, und der Merge ist blockiert; hier braucht es entweder keinen Pfadfilter oder einen Abschluss-Job, der immer läuft. Dritter Fehler: In Skripten den Zwei-Punkte-Vergleich gegen `main` für die Änderungsliste eines Pull Requests zu verwenden. Vierter Fehler: Pipelines von fremden Forks mit Schreibrechten und Secrets laufen zu lassen; laut Dokumentation werden Schreibrechte bei Pull Requests aus Forks standardmäßig auf Lesen reduziert, und das sollte man nicht aufweichen (Modul 08).

### Praxisbezug.

Beim Einrichten einer Pipeline stellt man drei Fragen: Was soll bei jeder Änderung schnell geprüft werden, was darf teurer und seltener laufen, und was passiert nur bei Releases? Aus den Antworten entsteht eine Trigger-Matrix aus Ereignis, Filter und Prüfumfang, die man als Teamstandard dokumentiert. Plattformübergreifend gelten dieselben Ideen, nur die Schlüsselwörter unterscheiden sich; für Jenkins, Azure Pipelines und GitLab siehe Modul 11.

## 9.2 Status Checks

Ein *Status Check* ist das Ergebnis einer automatischen Prüfung, das an einen Commit gehängt wird: Build erfolgreich, Tests bestanden, Sicherheitsscan ohne Funde. Die Plattform zeigt diese Ergebnisse am Pull Request an. Wirksam werden sie, wenn sie zur *Pflicht* gemacht werden: Dann kann ein Pull Request erst gemergt werden, wenn die genannten Prüfungen grün sind. Das ist die Verbindung von Branch Protection (Modul 05) und CI, von der die Lerneinheit spricht. Auf GitHub gehört der Regeltyp „erforderliche Statusprüfungen“ zu den Regeln eines Rulesets (Modul 05), bei anderen Plattformen heißt er ähnlich. Sinnvoll ist, die Pflichtprüfungen klein und aussagekräftig zu halten: Build, Unit-Tests, Linting, Secret-Scan, gegebenenfalls Abhängigkeitsprüfung. Lange oder instabile Prüfungen gehören nicht in die Pflicht, weil sie den Fluss bremsen und zu Umgehungen verleiten.

Die Idee, dass ein Server nur Stände annimmt, die eine Prüfung bestanden haben, ist nicht an eine Plattform gebunden. Sie lässt sich mit einem `pre-receive`-Hook auch auf einem einfachen Git-Server nachbauen, und genau das zeigt das folgende Beispiel. Der Hook prüft bei jedem Push auf `main` den neuen Stand aus: Er entpackt ihn in ein temporäres Verzeichnis und führt dort das Testskript aus. Besteht der Test nicht, lehnt der Hook den Push ab und gibt die Testausgabe an die pushende Person zurück. Die Dokumentation zu den Git-Hooks (abgerufen am 28.10.2026) beschreibt, dass `pre-receive` einmal pro Push läuft, die Zeilen „alt neu Referenz“ auf der Standardeingabe erhält und mit einem Wert ungleich Null verhindert, dass irgendeine Referenz aktualisiert wird.

```bash
git init -q --bare -b main server.git
cat > server.git/hooks/pre-receive <<'EOF'
#!/bin/sh
# Server-seitiger Check: Pushes auf main werden nur angenommen, wenn die Tests des neuen Stands bestehen
while read old new ref; do
  [ "$ref" = "refs/heads/main" ] || continue
  case "$new" in *[!0]*) ;; *) continue ;; esac     # Loeschungen hier nicht pruefen
  tmp=$(mktemp -d)
  git archive "$new" | tar -x -C "$tmp"
  if [ -f "$tmp/test.sh" ] && (cd "$tmp" && sh test.sh >"$tmp/log.txt" 2>&1); then
    echo "Check bestanden fuer $(echo $new | cut -c1-7)" >&2
  else
    echo "Check FEHLGESCHLAGEN fuer $(echo $new | cut -c1-7):" >&2
    [ -f "$tmp/log.txt" ] && sed 's/^/  | /' "$tmp/log.txt" >&2
    rm -rf "$tmp"; exit 1
  fi
  rm -rf "$tmp"
done
EOF
chmod +x server.git/hooks/pre-receive
git clone -q server.git work 2>/dev/null && cd work
git config user.name Dev; git config user.email dev@example.org
cat > rechne.sh <<'EOF'
rechne() { echo $(( $1 + $2 )); }
EOF
cat > test.sh <<'EOF'
. ./rechne.sh
[ "$(rechne 2 3)" = "5" ] || { echo "rechne 2 3 ergab $(rechne 2 3), erwartet 5"; exit 1; }
echo "alle Tests ok"
EOF
git add . && git commit -q -m "Start mit Test"
git push -q origin main; echo "Push 1: Exit $?"
sed -i 's/+/-/' rechne.sh && git commit -q -am "Refactoring (fehlerhaft)"
git push origin main; echo "Push 2: Exit $?"
sed -i 's/ - / + /' rechne.sh && git commit -q -am "Korrektur"
git push -q origin main; echo "Push 3: Exit $?"
git log --oneline
```

Der erste Push enthält Code und Test und wird angenommen; der Hook meldet das Ergebnis über die Fehlerausgabe, die Git dem Pushenden als Zeilen mit dem Präfix „remote:“ anzeigt. Der zweite Push enthält eine fehlerhafte Änderung: Der Test ergibt „-1“ statt 5, der Hook endet mit Exit-Code 1, Git meldet „pre-receive hook declined“, und der Stand erreicht den Server nicht. Der dritte Push enthält neben dem fehlerhaften auch den korrigierten Commit und wird angenommen, weil der Hook den *neuen Spitzenstand* prüft. Das ist zugleich die Grenze dieses einfachen Verfahrens: Zwischenstände einer Serie von Commits werden nicht einzeln getestet. Plattformen prüfen deshalb üblicherweise die Spitze eines Pull Requests, was für das Mergen genügt, und nur Teams, die jeden Commit für bisektierbar halten wollen (Modul 06), testen jeden einzelnen.

> **Ergänzung aus der Nachrecherche**
>
> **Pending und Pflichtprüfungen.** Die Plattform bewertet eine Pflichtprüfung nach ihrem Namen. Wird der Job umbenannt, bleibt der alte Name als ausstehende Pflicht bestehen, und niemand kann mehr mergen, bis die Regel angepasst wird. Das gleiche Bild entsteht, wenn der Workflow durch Pfadfilter gar nicht erst läuft (siehe Abschnitt 9.1, dort mit Verweis auf die Dokumentation zum Zustand „Pending“). Die Namen von Pflichtprüfungen sind deshalb eine stabile Schnittstelle und werden wie Code gepflegt (allgemeine Erfahrung, nicht aus einer abgerufenen Quelle).

### Typische Fehler und Stolperfallen.

Falsch ist, Pflichtprüfungen zu definieren, die gelegentlich ohne erkennbaren Grund rot werden („flaky Tests“). Das Team lernt, auf „erneut ausführen“ zu klicken, und irgendwann auch bei echten Fehlern. Richtig ist, instabile Tests zu reparieren oder aus der Pflicht zu nehmen. Zweiter Fehler: Prüfungen nur auf dem Feature-Branch zu verlangen, nicht aber nach dem Merge. Ein Pull Request kann grün sein und nach dem Merge mit einem anderen Pull Request zusammen rot werden; manche Plattformen bieten dafür Merge-Warteschlangen, die den Merge-Stand vorab testen (weiterführend, hier nicht abgerufen). Dritter Fehler: Pflichtprüfungen für Administratoren auszusetzen. Vierter Fehler: Sicherheitsprüfungen nur beratend, nicht blockierend einzurichten; dann wird ihre Ausgabe nicht gelesen.

### Praxisbezug.

Die Einführung von Pflichtprüfungen geschieht üblicherweise stufenweise: zuerst Build und Tests, dann Linting, dann Sicherheitsprüfungen. Bei jeder Stufe wird die Fehlalarmquote beobachtet, bevor die Prüfung zur Pflicht wird. Die Verbindung zu Branch Protection (Modul 05) und zu den Schutzregeln für Tags (Modul 08) macht daraus den *Release-Pfad*: Nichts erreicht `main`, was nicht geprüft und reviewt wurde.

## 9.3 Git Hooks

*Git Hooks* sind Skripte, die Git bei bestimmten Ereignissen selbst ausführt. Die Dokumentation (abgerufen am 28.10.2026) unterscheidet *clientseitige* Hooks, die im lokalen Repository laufen (etwa beim Committen, Mergen oder Pushen), und *serverseitige* Hooks, die beim Empfang eines Pushes auf dem Server ausgeführt werden. Ein Hook ist eine ausführbare Datei im Verzeichnis `.git/hooks` oder in dem Verzeichnis, das `core.hooksPath` nennt; Dateien ohne Ausführungsrecht werden ignoriert. Die wichtigsten Clientseiten-Hooks sind `pre-commit` (vor der Commit-Nachricht, kann den Commit verhindern), `commit-msg` (bekommt die Nachricht zur Prüfung), `pre-push` (vor der Übertragung, bekommt Remote und Referenzen) sowie `post-merge` und `post-checkout` für nachgelagerte Aktionen. Serverseitig gibt es `pre-receive` (einmal pro Push, kann alles ablehnen), `update` (einmal pro Referenz) und `post-receive` (nach dem Push, nur Benachrichtigungen).

Hooks sind nützlich, wenn man ihre Rolle richtig einordnet. Die Dokumentation weist darauf hin, dass `pre-commit`, `commit-msg` und `pre-merge-commit` mit `--no-verify` übersprungen werden können; clientseitige Hooks sind deshalb *Komfort* und kein Durchsetzungsmittel. Sie geben schnelles Feedback in Sekunden, bevor man auf die Pipeline wartet, und sparen Pipeline-Läufe. Verbindlich ist, was der Server oder die Pipeline durchsetzt. Das Verteilen der Hooks im Team ist ein eigenes Thema: Hooks liegen nicht im versionierten Teil des Repositories. Die übliche Lösung ist ein versioniertes Verzeichnis (`.githooks`) und die Einstellung `core.hooksPath`, die man einmalig nach dem Klonen setzt, oft über ein kleines Setup-Skript oder durch Werkzeuge, die Hooks verwalten (weiterführend, hier nicht abgerufen).

Gute Hooks sind schnell, deterministisch und prüfen nur das, was sie prüfen müssen. Ein `pre-commit`-Hook prüft nur die vorgemerkten Dateien und liest sie aus der Staging Area (`git show :datei`), nicht aus dem Arbeitsverzeichnis. Das ist ein häufiger Fehler: Wer das Arbeitsverzeichnis prüft, prüft eventuell etwas anderes als das, was committet wird, weil Änderungen teilweise vorgemerkt sind (Modul 02). Das Beispiel enthält drei Hooks und das Setup-Skript: eine Syntaxprüfung für Shell-Skripte, eine Prüfung des Commit-Betreffs und einen Pre-push-Hook, der Tests ausführt und direkte Pushes auf `main` ablehnt.

```bash
git init -q --bare -b main server.git
git clone -q server.git work 2>/dev/null && cd work
git config user.name Dev; git config user.email dev@example.org
mkdir .githooks

cat > .githooks/pre-commit <<'EOF'
#!/bin/sh
# Syntaxpruefung aller gestagten Shell-Skripte (aus der Staging Area, nicht aus dem Arbeitsverzeichnis)
status=0
for datei in $(git diff --cached --name-only --diff-filter=ACM -- '*.sh'); do
  if ! fehler=$(git show ":$datei" | sh -n 2>&1); then
    echo "pre-commit: Syntaxfehler in $datei" >&2
    echo "$fehler" | sed 's/^/  /' >&2
    status=1
  fi
done
exit $status
EOF

cat > .githooks/commit-msg <<'EOF'
#!/bin/sh
# Erste Zeile muss "typ: text" oder "typ(bereich): text" sein
erste=$(head -n 1 "$1")
if ! echo "$erste" | grep -Eq '^(feat|fix|docs|chore|refactor|test)(\([a-z0-9-]+\))?!?: .{3,}'; then
  echo "commit-msg: Betreff entspricht nicht der Konvention 'typ: text':" >&2
  echo "  $erste" >&2
  exit 1
fi
EOF

cat > .githooks/pre-push <<'EOF'
#!/bin/sh
# Vor dem Push laufen die Tests; direkte Pushes auf main sind blockiert
while read lokal_ref lokal_oid remote_ref remote_oid; do
  if [ "$remote_ref" = "refs/heads/main" ]; then
    echo "pre-push: direkter Push auf main ist nicht erlaubt" >&2
    exit 1
  fi
done
[ -f test.sh ] && sh test.sh >/dev/null 2>&1 || { echo "pre-push: Tests fehlgeschlagen" >&2; exit 1; }
exit 0
EOF
chmod +x .githooks/*

cat > setup-hooks.sh <<'EOF'
#!/bin/sh
# Einmalig nach dem Klonen ausfuehren
git config core.hooksPath .githooks
echo "Hooks aktiv: $(git config core.hooksPath)"
EOF
sh setup-hooks.sh

echo 'echo "ok"' > test.sh
echo 'echo hallo' > gut.sh
echo 'if true; then echo x' > kaputt.sh
git add gut.sh test.sh
git commit -m "wip"; echo "Exit: $?"
git commit -q -m "feat: Skript hinzugefuegt"; echo "Exit: $?"
git add kaputt.sh
git commit -q -m "feat: kaputtes Skript"; echo "Exit: $?"
git commit -q --no-verify -m "feat: kaputtes Skript trotzdem"; echo "Exit: $?"
git log --oneline
git push origin main; echo "Push main: Exit $?"
git push -q origin main:feature/ok; echo "Push feature: Exit $?"
```

Nach dem Setup sind die Hooks aktiv, und die folgenden Befehle zeigen jeden Fall. Der Commit mit der Nachricht „wip“ wird vom `commit-msg`-Hook abgelehnt, weil der Betreff nicht zur Konvention passt. Mit einem passenden Betreff gelingt der Commit, denn `gut.sh` hat keine Syntaxfehler. Beim Hinzufügen von `kaputt.sh` meldet der `pre-commit`-Hook einen Syntaxfehler („end of file unexpected (expecting "fi")“) und verhindert den Commit. Mit `--no-verify` lässt sich der Hook trotzdem umgehen, und der Commit entsteht doch: Das zeigt, warum clientseitige Hooks keine Durchsetzung sind. Beim Push meldet der `pre-push`-Hook für `main` einen Verstoß, während der Push auf einen anderen Branch gelingt, weil der Hook dort nichts beanstandet und die Tests durchlaufen. Die Hook-Dateien müssen ausführbar sein; `chmod +x` ist unter Windows relevant, wenn Git for Windows die Hooks ausführt (Dateimodi werden dort von Git verwaltet; im Zweifel prüft man mit `git ls-files --stage`, ob Modus 100755 eingetragen ist).

### Typische Fehler und Stolperfallen.

Falsch ist, einen langsamen Hook einzubauen, der die komplette Testsuite bei jedem Commit ausführt. Dann wird er mit `--no-verify` umgangen oder abgeschaltet. Richtig ist, Hooks schnell zu halten (Sekunden) und die lange Prüfung in die Pipeline zu legen. Zweiter Fehler: Den Zustand des Arbeitsverzeichnisses statt der Staging Area zu prüfen; wer mit `git add -p` teilweise vormerkt, bekommt sonst falsche Ergebnisse. Dritter Fehler: Hooks ohne Verteilungsweg: Sie funktionieren dann bei der Person, die sie geschrieben hat, und sonst nirgends. Vierter Fehler: Hooks, die auf Werkzeuge angewiesen sind, die nicht auf jedem Rechner installiert sind, ohne klare Fehlermeldung; der Hook sollte prüfen, ob sein Werkzeug vorhanden ist, und sagen, was fehlt.

### Praxisbezug.

Typische Pre-commit-Aufgaben sind Formatierung, Linting und Secret-Scan auf den vorgemerkten Dateien (Modul 08), typische Commit-msg-Aufgaben die Konventionsprüfung für automatische Changelogs (Abschnitt 9.5). Serverseitig setzen Betreiber `pre-receive`- oder `update`-Hooks für Richtlinien ein, die sich nicht umgehen lassen sollen, etwa das Verbot von Force-Pushes auf bestimmte Branches oder unveränderliche Tags (Modul 08). Auf gehosteten Plattformen, die keine eigenen Server-Hooks erlauben, übernehmen Regeln der Plattform diese Aufgabe.

## 9.4 Versionierung aus Git

Die Berechnung der Versionsnummer aus Tags und Commits wurde in Modul 07 (Abschnitte 7.2 und 7.8) ausführlich behandelt: Der Tag ist die einzige Quelle der Wahrheit, Release-Builds laufen auf getaggten Commits, Zwischenstände erhalten ein Vorabsuffix mit Commit-Hash. Hier geht es um die zweite Hälfte der Lerneinheit, die *reproduzierbaren Artefakte*. Ein Build ist reproduzierbar, wenn zwei Läufe mit demselben Quellstand dasselbe Ergebnis liefern, bis auf das letzte Bit. Das ist keine akademische Spielerei: Nur dann kann man später beweisen, dass ein ausgeliefertes Artefakt aus dem behaupteten Quellstand entstanden ist, und nur dann haben Prüfsummen einen Wert.

Typische Quellen von Abweichungen sind eingebettete Zeitstempel, die Reihenfolge von Dateien im Archiv, Dateirechte und Eigentümer, Pfade des Build-Rechners und die Zeitangabe im gzip-Kopf. Die Gegenmaßnahmen sind einfach, wenn man sie kennt. Als Zeitstempel verwendet man nicht die Uhr des Rechners, sondern die Zeit des Commits: Die Umgebungsvariable `SOURCE_DATE_EPOCH` ist eine verbreitete Konvention dafür (Wissensstand, nicht abgerufen), die viele Werkzeuge beachten. Man erhält sie mit `git log -1 --format=%ct <tag>`. Die Dateien kommen mit `git archive` direkt aus dem Tag und nicht aus einem Arbeitsverzeichnis, in dem Reste früherer Builds liegen können. Beim Packen sortiert man die Dateinamen, setzt Eigentümer und Gruppe auf feste Werte und lässt den Namen und Zeitstempel aus dem gzip-Kopf weg (`gzip -n`). Das folgende Skript erzeugt aus einem Tag ein Archiv, in dem außerdem eine Versionsdatei mit Tag-Nummer und Commit steht, die beim Build erzeugt und nicht eingecheckt wird. Es läuft dreimal: mit verändertem Arbeitsverzeichnis, mit anderer Dateimaske und in einem frischen Klon an einem anderen Ort.

```bash
git init -q -b main rb && cd rb
git config user.name Dev; git config user.email dev@example.org
mkdir src && echo 'print("hallo")' > src/app.py && echo "doku" > README.md && git add . && git commit -q -m "feat: Start"
git tag -a v1.0.0 -m "Release 1.0.0"
echo '# nach dem Release' >> src/app.py && git commit -q -am "feat: weiter"

cat > build.sh <<'EOF'
#!/bin/sh
# Reproduzierbarer Build eines Tags: gleicher Eingang -> gleiche Pruefsumme
# Aufruf: build.sh <tag> <zieldatei>
umask 022
tag=$1; version=${tag#v}
ziel=$(cd "$(dirname "$2")" && pwd)/$(basename "$2")
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct "$tag")
tmp=$(mktemp -d)
git archive --prefix="app-$version/" "$tag" | tar -x -C "$tmp"
# Versionsdatei wird beim Build erzeugt und nicht eingecheckt
printf 'VERSION=%s\nCOMMIT=%s\n' "$version" "$(git rev-parse "$tag^{commit}")" > "$tmp/app-$version/VERSION.txt"
( cd "$tmp" && tar --sort=name --mtime="@$SOURCE_DATE_EPOCH" --owner=0 --group=0 --numeric-owner \
    -cf - "app-$version" | gzip -n > "$ziel" )
rm -rf "$tmp"
sha256sum "$ziel" | cut -c1-20
EOF
sh build.sh v1.0.0 ../lauf1.tar.gz
sleep 2
touch src/app.py
(umask 077; sh build.sh v1.0.0 ../lauf2.tar.gz)
mkdir ../anderer-ort && git clone -q . ../anderer-ort/klon
(cd ../anderer-ort/klon && sh ../../rb/build.sh v1.0.0 ../../lauf3.tar.gz)
tar xzOf ../lauf1.tar.gz app-1.0.0/VERSION.txt
```

Das Skript bestimmt als Erstes die Zeit des Commits, auf den der Tag zeigt, und reicht sie als `SOURCE_DATE_EPOCH` an `tar` weiter, das damit alle Dateizeiten im Archiv auf diesen Wert setzt. Die Dateien stammen aus `git archive`, das nur versionierte Inhalte enthält, und die erzeugte Versionsdatei `VERSION.txt` nennt Version und vollständigen Commit-Hash. Die Option `--sort=name` sorgt für eine feste Dateireihenfolge, `--owner=0 --group=0 --numeric-owner` für feste Eigentümer. `umask 022` am Anfang fixiert die Dateirechte, die sonst von der Einstellung des Rechners abhängen. Das Ergebnis: Alle drei Läufe liefern dieselbe Prüfsumme (die Ausgabe ist auf die ersten 20 Zeichen gekürzt), obwohl zwischen ihnen zwei Sekunden vergangen sind, die Datei `src/app.py` neu berührt wurde, die Dateimaske eine andere war und der dritte Lauf in einem anderen Verzeichnis eines frischen Klons stattfand. Die letzte Zeile liest die Versionsdatei aus dem Archiv. Dass das Ergebnis auch bei anderen Versionen von `tar` und `gzip` identisch bleibt, ist nicht garantiert; für die Praxis legt man die Werkzeugversionen im Build-Image fest.

### Typische Fehler und Stolperfallen.

Falsch ist, die Versionsnummer in eine Datei zu schreiben, diese einzuchecken und nach dem Release von Hand zu aktualisieren. Dann weichen Tag und Datei irgendwann ab. Richtig ist, die Version beim Build aus dem Tag zu erzeugen, wie im Beispiel. Zweiter Fehler: Den Build auf einem Commit durchzuführen, der nicht getaggt ist, und das Ergebnis trotzdem als Release zu bezeichnen. Dritter Fehler: Aus dem Arbeitsverzeichnis zu archivieren statt aus Git; übrig gebliebene Dateien (Builds, lokale Konfiguration) landen im Artefakt. Vierter Fehler: Auf Reproduzierbarkeit zu verzichten, weil „es sowieso keiner prüft“; im Sicherheitsfall ist sie das Mittel, um einen manipulierten Build zu erkennen.

### Praxisbezug.

In regulierten Umgebungen und bei Software, die Dritte ausliefern oder installieren, wird die Frage gestellt, ob man ein Artefakt aus dem Quellcode nachbauen kann. Die Kombination aus Tag, deterministischem Archiv, festgehaltenen Werkzeugversionen und Manifest (Modul 07, Abschnitt 7.6) beantwortet sie. In der Pipeline gehört der Schritt in eine wiederverwendbare Komponente, damit nicht jedes Projekt seine eigene, leicht abweichende Variante baut.

## 9.5 Changelog Automation

Die Grundlagen der Release Notes, also Zielgruppen, Struktur nach *Keep a Changelog* und das Skript zur Erzeugung eines Abschnitts aus Conventional Commits, stehen in Modul 07, Abschnitt 7.3. Dieser Abschnitt behandelt, wie man daraus einen *automatisierten Ablauf* macht. Der Ablauf hat drei Bausteine. Erstens muss die Konvention eingehalten werden, sonst ist die Automatisierung auf Sand gebaut: Eine Prüfung der Commit-Nachrichten gehört in die Pull-Request-Pipeline (und, als schnelles Feedback, in den `commit-msg`-Hook aus Abschnitt 9.3). Zweitens erzeugt ein Skript bei einem neuen Tag den Text der Release Notes aus dem Bereich zwischen dem vorherigen und dem neuen Tag. Drittens wird der Text veröffentlicht: als Datei im Repository, als Beschreibung der Release-Seite oder in einer Benachrichtigung.

Zwei Feinheiten bestimmen die Zuverlässigkeit. Die Wahl des Bereichs: Der vorherige Tag wird nicht von Hand angegeben, sondern mit `git describe --tags --abbrev=0 <tag>^` ermittelt, also ausgehend vom Vorgänger des neuen Tags. Beim ersten Release gibt es keinen vorherigen Tag, und das Skript muss den gesamten Verlauf verwenden. Und die Prüfung der Konvention muss auf den Commits *des Pull Requests* laufen, also auf dem Bereich zwischen Zielbranch und Spitze, nicht auf der gesamten Historie, sonst scheitert sie an alten Commits, die man nicht mehr ändern kann. Das Beispiel enthält beide Skripte.

```bash
git init -q -b main ch && cd ch
git config user.name Dev; git config user.email dev@example.org
cat > lint-commits.sh <<'EOF'
#!/bin/sh
# Prueft alle Commits eines Bereichs auf die Konvention (fuer die PR-Pipeline)
# Aufruf: lint-commits.sh <bereich>   z.B. origin/main..HEAD
status=0
for c in $(git rev-list --no-merges "$1"); do
  betreff=$(git log -1 --format=%s "$c")
  if ! echo "$betreff" | grep -Eq '^(feat|fix|docs|chore|refactor|test)(\([a-z0-9-]+\))?!?: .{3,}'; then
    echo "Verstoss: $(echo $c | cut -c1-7) \"$betreff\""
    status=1
  fi
done
[ $status -eq 0 ] && echo "Alle Commits entsprechen der Konvention"
exit $status
EOF
cat > release-notes.sh <<'EOF'
#!/bin/sh
# Erzeugt Release Notes fuer einen Tag; Bereich = vorheriger Tag bis Tag (beim ersten Release: alles)
tag=$1
vorher=$(git describe --tags --abbrev=0 --match 'v[0-9]*' "$tag^" 2>/dev/null) || vorher=""
bereich=${vorher:+$vorher..}$tag
echo "# $tag ($(git log -1 --format=%cs "$tag"))"
echo "Aenderungen seit ${vorher:-Projektbeginn}:"
echo
for typ in feat fix; do
  case $typ in feat) titel="Neu";; fix) titel="Behoben";; esac
  zeilen=$(git log --format='%s' "$bereich" | grep -E "^$typ(\([a-z0-9-]+\))?!?: " | sed -E "s/^$typ(\(([a-z0-9-]+)\))?!?: /\2|/; s/^\|//; s/^([^|]*)\|/\1: /")
  [ -n "$zeilen" ] && { echo "## $titel"; echo "$zeilen" | sed 's/^/- /'; echo; }
done
EOF
git commit -q --allow-empty -m "feat: Grundfunktion"
git tag -a v1.0.0 -m "1.0.0"
git commit -q --allow-empty -m "fix: Fehler beim Start"
git commit -q --allow-empty -m "feat(export): CSV-Export"
git commit -q --allow-empty -m "aufgeraeumt"
git commit -q --allow-empty -m "feat: Suche"
git tag -a v1.1.0 -m "1.1.0"

sh lint-commits.sh v1.0.0..HEAD; echo "Exit: $?"
git commit -q --allow-empty -m "fix: Tippfehler in Meldung"
echo "--- nur der neue Commit (Pull-Request-Bereich):"
sh lint-commits.sh HEAD~1..HEAD; echo "Exit: $?"
echo "--- Release Notes fuer v1.1.0:"
sh release-notes.sh v1.1.0
echo "--- Release Notes fuer das erste Release:"
sh release-notes.sh v1.0.0
```

Die Prüfung über den Bereich `v1.0.0..HEAD` meldet den Commit „aufgeraeumt“, weil sein Betreff keinem Typ zugeordnet ist, und endet mit Exit-Code 1; die Pipeline würde den Pull Request blockieren. Anschließend zeigt das Beispiel, dass für einen neuen Pull Request nur der Bereich der neuen Commits geprüft wird: Der alte Verstoß gehört nicht dazu, die Prüfung besteht. Das Skript für die Release Notes bestimmt den Vorgänger über `git describe` ab dem Elterncommit des neuen Tags. Für `v1.1.0` ergibt das `v1.0.0` als Anfang des Bereichs, und der Text enthält die Gruppen „Neu“ und „Behoben“ mit den Einträgen (der Bereich `export:` aus dem Typ `feat(export)` steht dem Eintrag voran). Für das erste Release schlägt die Suche nach einem Vorgänger fehl, das Skript fällt auf „Projektbeginn“ zurück und listet alle Commits bis zum Tag. Der Commit ohne Typ taucht in den Release Notes nicht auf, was die Prüfung der Konvention als vorgelagerten Schritt begründet.

### Typische Fehler und Stolperfallen.

Falsch ist, die Konvention vorauszusetzen, aber nicht zu prüfen. Jede unpassende Nachricht fehlt dann im Changelog, ohne dass es jemand merkt. Richtig ist die Prüfung im Pull Request, sinnvollerweise ergänzt um die Prüfung des *Titels* bei Squash-Merges, weil dort der Titel zum Commit auf `main` wird (Modul 03). Zweiter Fehler: Den Changelog bei jedem Merge neu zu committen und damit Konflikte und Schleifen zu erzeugen; besser entsteht er bei der Release-Erstellung. Dritter Fehler: Den automatisch erzeugten Text unbesehen als Release Notes zu verwenden (Modul 07, Abschnitt 7.3). Vierter Fehler: Pipelines, die den Changelog in den Hauptbranch zurückschreiben, ohne dass der Bot-Commit gekennzeichnet ist und keine weitere Pipeline auslöst (Abschnitt 9.7).

### Praxisbezug.

Teams automatisieren das in unterschiedlichem Grad: Von der reinen Prüfung der Nachrichten mit manuell geschriebenen Release Notes über einen Entwurf, den ein Mensch redigiert, bis zur vollautomatischen Veröffentlichung bei jedem Merge. Der mittlere Weg ist am verbreitetsten, weil er die Mühe spart und trotzdem die Qualität sichert. Wo die Automatisierung Dateien zurück ins Repository schreibt, braucht es einen Service-Account mit minimalen Rechten, was das Thema von Abschnitt 9.7 ist.

## 9.6 Monorepo Pipelines

In einem Monorepo (Modul 05) liegen mehrere Dienste oder Pakete in einem Repository. Würde jede Änderung die Pipeline für alle auslösen, wüchse die Laufzeit mit jedem hinzugefügten Dienst, und eine Änderung an der Dokumentation würde den Build aller Anwendungen anstoßen. Die Lösung ist die *Erkennung geänderter Pfade* und der *selektive Build*: Die Pipeline ermittelt, welche Dateien sich geändert haben, ordnet sie den betroffenen Diensten zu und baut nur diese. Die Plattformen bieten dafür Pfadfilter an (Abschnitt 9.1), aber sie reichen nicht für Abhängigkeiten: Wenn Dienst A eine gemeinsame Bibliothek nutzt und die Bibliothek sich ändert, muss A neu gebaut werden, obwohl keine Datei unter A betroffen ist. Diese Logik gehört in ein eigenes, versioniertes Skript.

Die Grundlage ist der Vergleich `git diff --name-only <basis>...HEAD` (Drei-Punkte-Vergleich, Pull-Request-Sicht). Die schwierige Frage ist die Basis: In einem Pull Request ist es der Zielbranch, bei einem Push auf `main` der Stand vor dem Push. Beim ersten Push eines neuen Branches ist der alte Hash laut Hook-Dokumentation nur Nullen, und es gibt keine Vergleichsbasis; dann baut man vorsichtshalber alles. Dasselbe gilt, wenn der Vergleich scheitert, etwa weil die Basis im flachen Klon fehlt (Abschnitt 9.8), und für Änderungen an der gemeinsamen Build-Konfiguration selbst. Faustregel: Im Zweifel mehr bauen als zu wenig, denn ein fälschlich übersprungener Build ist gefährlicher als ein überflüssiger. Das folgende Skript beherzigt das und verwaltet Abhängigkeiten in einfachen Dateien `DEPS`, die neben jedem Dienst liegen.

```bash
git init -q -b main mono && cd mono
git config user.name Dev; git config user.email dev@example.org
mkdir -p services/api services/web libs/common docs
for d in services/api services/web libs/common docs; do echo 1 > $d/datei.txt; done
echo "api braucht common" > services/api/DEPS; echo "libs/common" >> services/api/DEPS
echo "web braucht nichts" > services/web/DEPS
git add . && git commit -q -m "Monorepo-Start"
git tag basis

cat > changed.sh <<'EOF'
#!/bin/sh
# Ermittelt, welche Services gebaut werden muessen
# Aufruf: changed.sh <basis> ; vergleicht gegen HEAD (Pull-Request-Sicht, drei Punkte)
basis=$1
case "$basis" in
  *[!0]*) ;;                           # echter Hash oder Name
  *) echo "ALLE (keine Vergleichsbasis)"; exit 0 ;;  # nur Nullen: neuer Branch
esac
geaendert=$(git diff --name-only "$basis...HEAD") || { echo "ALLE (Vergleich fehlgeschlagen)"; exit 0; }
[ -z "$geaendert" ] && { echo "nichts zu bauen"; exit 0; }
# Aenderungen an gemeinsamen Build-Dateien betreffen alles
if echo "$geaendert" | grep -Eq '^(Makefile|ci/|\.github/)'; then echo "ALLE (Build-Konfiguration geaendert)"; exit 0; fi
ziele=""
for svc in $(ls services); do
  betroffen=0
  # direkte Aenderung am Service
  echo "$geaendert" | grep -q "^services/$svc/" && betroffen=1
  # Aenderung an einer deklarierten Abhaengigkeit
  for dep in $(grep '^libs/' "services/$svc/DEPS" 2>/dev/null); do
    echo "$geaendert" | grep -q "^$dep/" && betroffen=1
  done
  [ $betroffen -eq 1 ] && ziele="$ziele $svc"
done
echo "${ziele:-nichts zu bauen}" | sed 's/^ //'
EOF

echo "Fall 1: nur docs"
git switch -q -c f1 && echo 2 >> docs/datei.txt && git commit -q -am "docs" && sh changed.sh basis
echo "Fall 2: nur web"
git switch -q -c f2 basis && echo 2 >> services/web/datei.txt && git commit -q -am "web" && sh changed.sh basis
echo "Fall 3: libs/common (api haengt davon ab)"
git switch -q -c f3 basis && echo 2 >> libs/common/datei.txt && git commit -q -am "common" && sh changed.sh basis
echo "Fall 4: api und web gleichzeitig"
git switch -q -c f4 basis && echo 2 >> services/api/datei.txt && echo 2 >> services/web/datei.txt && git commit -q -am "beide" && sh changed.sh basis
echo "Fall 5: Build-Konfiguration"
git switch -q -c f5 basis && mkdir ci && echo x > ci/pipeline.yml && git add . && git commit -q -m "ci" && sh changed.sh basis
echo "Fall 6: neuer Branch (alte Basis nur Nullen)"
sh changed.sh 0000000000000000000000000000000000000000
```

Das Repository enthält zwei Dienste, eine gemeinsame Bibliothek und Dokumentation; die Datei `DEPS` von `api` nennt `libs/common` als Abhängigkeit. Das Skript prüft zuerst, ob eine Vergleichsbasis vorliegt (ein Hash, der nur aus Nullen besteht, kennzeichnet einen neuen Branch), liest die geänderten Dateien und behandelt Änderungen an `ci/` oder `.github/` als „alles neu bauen“. Für jeden Dienst prüft es anschließend, ob eine Datei unterhalb des Dienstes geändert wurde oder eine Datei unterhalb einer deklarierten Abhängigkeit. Die sechs Fälle zeigen das Ergebnis: Eine reine Dokumentationsänderung löst nichts aus; eine Änderung in `services/web` baut nur `web`; eine Änderung der gemeinsamen Bibliothek baut `api`, weil es sie nutzt, aber nicht `web`; zwei geänderte Dienste werden beide gebaut; die Änderung der Pipeline-Konfiguration baut alles, und ein neuer Branch ohne Basis ebenfalls. In der Pipeline würde man die Ausgabe als Liste oder als Matrix an die folgenden Jobs übergeben.

### Typische Fehler und Stolperfallen.

Falsch ist, sich allein auf Pfadfilter der Plattform zu verlassen, wenn es gemeinsame Bibliotheken gibt. Dann bleibt die Pipeline eines Dienstes grün, obwohl seine Abhängigkeit kaputt ist. Richtig sind explizite Abhängigkeiten, die ein Skript oder das Build-System kennt. Zweiter Fehler: Das fehlende Wissen über die Basis stillschweigend als „keine Änderungen“ zu deuten. Der sichere Rückfall ist „alles bauen“. Dritter Fehler: Nach dem selektiven Build nicht zu veröffentlichen, was gebaut wurde, sodass unklar bleibt, welcher Stand in Produktion läuft; auch für selektive Builds braucht jeder Dienst eine eigene Versionierung oder Tag-Namensregeln (zum Beispiel `api/v1.2.0`). Vierter Fehler: Die Erkennung nur lokal zu testen. Sie sollte mit Testfällen wie im Beispiel abgesichert sein, denn Fehler in dieser Logik bleiben lange unentdeckt.

### Praxisbezug.

Große Monorepos nutzen Build-Systeme, die Abhängigkeitsgraphen kennen und selbst ermitteln, was neu zu bauen ist (weiterführend, hier nicht abgerufen). Kleinere Repositories kommen mit einem Skript wie im Beispiel aus. Wichtig ist in beiden Fällen die Kombination mit Sparse Checkout und flachem Klon aus Modul 05, damit die Pipeline nicht zuerst das gesamte Repository laden muss, und die Beachtung des Zusammenspiels mit den Pflichtprüfungen: Ein Dienst, der nicht gebaut wird, darf die Pflicht nicht offen lassen.

## 9.7 Bot- und Service Accounts

Automatisierte Abläufe schreiben ins Repository: Sie erhöhen Versionsnummern, aktualisieren Abhängigkeiten, erzeugen Changelogs, legen Tags an. Dafür brauchen sie eine *Identität*, und die muss zwei Anforderungen erfüllen. Sie muss erkennbar sein, damit man in der Historie und im Audit-Log sofort sieht, welche Änderung von einer Person und welche von einer Maschine stammt, und sie muss minimale Rechte haben (Modul 08, Abschnitt 8.4), damit ein kompromittierter Bot nicht das ganze Repository gefährdet. Ein *Service Account* ist ein Konto ohne menschlichen Besitzer, ein Bot oft eine App der Plattform mit eigener Identität. Beide sind besser als der persönliche Zugang einer Entwicklerin, der in die Pipeline kopiert wurde: Der verschwindet mit dem Mitarbeiter, trägt dessen Namen in Commits, die er nie geschrieben hat, und hat meist viel zu viele Rechte.

Zur Kennzeichnung automatisierter Commits gehören vier Dinge. Eine eigene Autor-Identität (Name und E-Mail-Adresse, zum Beispiel eine „noreply“-Adresse der Organisation) macht Bot-Commits filterbar. Ein klarer Commit-Betreff mit Typ `chore(release):` zeigt den Zweck. *Trailer* wie `Triggered-by` und `Approved-by` halten fest, durch welchen Lauf und mit welcher Freigabe der Commit entstand; `git commit --trailer` und `git interpret-trailers` arbeiten mit ihnen (verfügbar ab Git 2.32, im Test war es 2.43). Und eine Markierung wie `[skip ci]` im Betreff verhindert bei vielen Plattformen den erneuten Pipeline-Lauf. Das Risiko, das hier lauert, ist die *Endlosschleife*: Ein Bot committet, der Commit löst eine Pipeline aus, die Pipeline committet wieder. Schutz bieten die Markierung, ein Wächter am Anfang der Pipeline, der Bot-Commits erkennt, und Pipeline-Regeln, die den Bot-Push von den Triggern ausnehmen.

```bash
git init -q -b main bot && cd bot
git config user.name Anna; git config user.email anna@example.org
echo "version=1.0.0" > VERSION && git add . && git commit -q -m "feat: Start"

cat > bot-commit.sh <<'EOF'
#!/bin/sh
# Ein Bot committet eine Versionserhoehung mit eigener Identitaet und klarer Kennzeichnung
neu=$1
echo "version=$neu" > VERSION
git add VERSION
git -c user.name="release-bot" -c user.email="release-bot@users.noreply.example.org" \
  commit -q -m "chore(release): Version $neu [skip ci]" \
  --trailer "Triggered-by: pipeline-4711" --trailer "Approved-by: anna@example.org"
EOF
sh bot-commit.sh 1.0.1
git log -1 --format='%an <%ae>%n%B'

cat > ci-guard.sh <<'EOF'
#!/bin/sh
# Pipeline-Schutz gegen Endlosschleifen: Commits des Bots loesen keinen neuen Build aus
autor=$(git log -1 --format=%ae)
betreff=$(git log -1 --format=%s)
if [ "$autor" = "release-bot@users.noreply.example.org" ] || echo "$betreff" | grep -q '\[skip ci\]'; then
  echo "Bot-Commit erkannt: Pipeline wird uebersprungen"; exit 0
fi
echo "Normaler Commit: Pipeline laeuft"
EOF
sh ci-guard.sh
echo 2 > datei.txt && git add . && git commit -q -m "fix: normale Aenderung"
sh ci-guard.sh
echo "--- Audit: alle Bot-Commits"
git log --author=release-bot --format='%h %an %s'
git log --format='%ae' | sort | uniq -c
git log --format='%(trailers:key=Approved-by,valueonly)' --author=release-bot | head -1
```

Das Skript `bot-commit.sh` setzt Name und E-Mail-Adresse nur für diesen einen Befehl mit `git -c`, ohne die Konfiguration des Repositories zu verändern; so bleibt die Identität des Menschen unberührt. Der Commit trägt den Betreff mit Typ, Markierung `[skip ci]` und die zwei Trailer am Ende der Nachricht. Die Ausgabe von `git log` zeigt Autor, Betreff und Trailer. Der Wächter `ci-guard.sh` liest Autor und Betreff des letzten Commits: Beim Bot-Commit erkennt er die Adresse und die Markierung und überspringt die Pipeline, beim normalen Commit lässt er sie laufen. Die letzten Befehle zeigen die Auswertung: `git log --author` liefert alle Bot-Commits, die Zusammenfassung nach E-Mail-Adresse zeigt, wie viele Commits von wem stammen, und die Format-Option `%(trailers:...)` gibt den Wert eines Trailers aus.

Die Rechte des Bots sind die zweite Hälfte der Aufgabe. In der Pipeline bedeutet das: ein eigener Token mit dem kleinsten nötigen Umfang, nur für die Schritte, die ihn brauchen. Für GitHub Actions beschreibt die Dokumentation zur Workflow-Syntax (abgerufen am 28.10.2026), dass der Block `permissions` die Bereiche des automatisch erzeugten Tokens festlegt, jeder Bereich `read`, `write` oder `none` sein kann und dass nicht genannte Bereiche auf `none` stehen, sobald man einen nennt. Pull Requests aus Forks erhalten standardmäßig nur Lesezugriff, ebenso Pull Requests des Dependabot, die zudem keinen Zugriff auf Secrets haben. Mit `id-token: write` lässt sich ein OIDC-Token anfordern. Bei `pull_request_target` erhält der Token dagegen Lese- und Schreibrechte, selbst bei Forks, und braucht daher besondere Vorsicht; das ist ein bekanntes Einfallstor, bei dem Code aus fremden Beiträgen mit hohen Rechten läuft.

### Typische Fehler und Stolperfallen.

Falsch ist, das persönliche Zugriffstoken einer Person für die Pipeline zu nehmen. Es gibt keine Trennung zwischen Mensch und Maschine, und beim Ausscheiden bricht die Automatisierung. Richtig sind ein Service-Account oder eine App mit eigenem, eng begrenztem Token. Zweiter Fehler: Bot-Commits unter dem Namen einer realen Person zu erzeugen. Das verwischt Verantwortlichkeiten und macht Audits unzuverlässig. Dritter Fehler: Keine Schleifenerkennung; ein Bot, der den Changelog committet, kann ohne Markierung Hunderte Läufe auslösen. Vierter Fehler: Dem Bot Schreibrechte auf geschützte Branches zu geben, damit er Schutzregeln umgehen kann; besser schreibt er auf einen eigenen Branch und eröffnet einen Pull Request, der den normalen Prüfungen unterliegt.

### Praxisbezug.

Typische Bots sind Abhängigkeits-Updater, Release-Bots und Formatierer. Gute Praxis ist, dass sie Änderungen als Pull Request vorschlagen und nicht direkt pushen. Bleibt es bei Direktpushes (etwa ein Versions-Commit nach dem Release), dann auf einen dafür vorgesehenen Pfad, mit Ausnahme in den Schutzregeln nur für diesen Account und mit Eintrag im Audit-Log (Modul 08, Abschnitt 8.7).

## 9.8 CI/CD Troubleshooting

Wenn eine Pipeline fehlschlägt, aber „bei mir lokal geht es“, liegt die Ursache fast immer in einem Unterschied zwischen lokaler Umgebung und Runner. Bei Git-Themen sind es immer dieselben Verdächtigen. Erstens der *Checkout*: Runner klonen oft flach (nur ein Commit), im Zustand *detached HEAD* (ein bestimmter Commit statt eines Branches, siehe Modul 10) und ohne Tags. Alles, was Historie braucht, versagt dann: `git describe`, `git log` über einen Bereich, Vergleiche gegen den Zielbranch, die Ermittlung der Basis für geänderte Pfade (Abschnitt 9.6). Die Fehlermeldung „No names found, cannot describe anything“ ist das klassische Zeichen. Zweitens die *Credentials*: Der Runner hat keinen Credential Manager und keine SSH-Schlüssel des Entwicklers, sondern ein Pipeline-Token mit eigenen Rechten. Zugriffe auf andere private Repositories, auf Submodule oder auf Pakete scheitern, wenn dieses Token sie nicht umfasst. Drittens die *Submodule*: Sie werden von einem einfachen Klon nicht mitgeladen; die Pipeline muss sie ausdrücklich holen (`git submodule update --init --recursive` oder die entsprechende Option des Checkout-Schritts) und braucht Zugriff auf deren URLs. Viertens die *Eigentumsprüfung*: Neuere Git-Versionen verweigern die Arbeit in einem Repository, das einem anderen Benutzer gehört. Im Test lautete die Meldung „fatal: detected dubious ownership in repository“, gefolgt vom Hinweis, das Verzeichnis mit `safe.directory` freizugeben; das trifft häufig Container, in denen der Checkout von einem anderen Benutzer stammt als der Build.

Wirksame Fehlersuche beginnt mit einer *Diagnose-Ausgabe*, die man in der Pipeline und lokal gleichermaßen erzeugen kann, und mit dem Vergleich beider. Das Skript im Beispiel gibt die für Git relevanten Umgebungsdaten aus: Version, Verzeichnis, aktueller Commit, ob ein Branch ausgecheckt ist, ob der Klon flach ist, wie viele Commits und Tags vorliegen, das Ergebnis von `git describe`, die Remotes (mit Warnung vor Zugangsdaten in URLs), den konfigurierten Credential Helper und den Zustand der Submodule. Es wird einmal in einem vollständigen Entwicklerklon und einmal in einem Klon wie auf einem typischen Runner ausgeführt.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main lib && (cd lib && git config user.name Dev && git config user.email dev@example.org && echo 1 > l.txt && git add . && git commit -q -m "lib")
git init -q -b main app && cd app
git config user.name Dev; git config user.email dev@example.org
echo 1 > a.txt && git add . && git commit -q -m "feat: Start"
git tag -a v1.0.0 -m "1.0.0"
git submodule add -q ../lib vendor/lib && git commit -q -m "feat: Submodul"
for i in 1 2 3; do echo $i >> a.txt; git commit -q -am "fix: $i"; done
cd ..

cat > ci-diagnose.sh <<'EOF'
#!/bin/sh
# Diagnose-Ausgabe fuer Checkout-Probleme; gleiche Ausgabe lokal und im Runner vergleichen
echo "git:            $(git --version)"
echo "Verzeichnis:    $(pwd)"
echo "HEAD:           $(git rev-parse --short HEAD)"
if git symbolic-ref -q HEAD >/dev/null; then echo "Branch:         $(git symbolic-ref --short HEAD)"; else echo "Branch:         (detached HEAD)"; fi
echo "flacher Klon:   $(git rev-parse --is-shallow-repository)"
echo "Commits:        $(git rev-list --count HEAD)"
echo "Tags:           $(git tag | wc -l)"
echo "describe:       $(git describe --tags --long 2>&1 | head -1)"
echo "Remotes:        $(git remote | tr '\n' ' ')"
for r in $(git remote); do
  case "$(git remote get-url "$r")" in
    https://*@*|http://*@*) echo "  WARNUNG: Remote $r enthaelt Zugangsdaten in der URL" ;;
  esac
done
echo "Credential:     $(git config --get-all credential.helper | tr '\n' ' ')"
echo "Submodule:"
git submodule status 2>&1 | sed 's/^/  /'
git submodule status | grep -q '^-' && echo "  HINWEIS: Submodule nicht initialisiert (submodules: recursive bzw. git submodule update --init)"
EOF

echo "=== Lokaler Entwicklerklon (vollstaendig, Submodul initialisiert)"
git clone -q --recurse-submodules file://$PWD/app lokal && (cd lokal && sh ../ci-diagnose.sh)
echo
echo "=== Runner-Checkout (flach, detached, ohne Submodule)"
git clone -q --depth=1 file://$PWD/app runner && (cd runner && git checkout -q --detach HEAD && sh ../ci-diagnose.sh)
```

Die beiden Ausgaben unterscheiden sich genau an den Stellen, die in der Praxis Ärger machen. Im lokalen Klon steht ein Branch `main`, der Klon ist nicht flach, es gibt fünf Commits und einen Tag, `git describe` liefert „v1.0.0-4-g...“, und das Submodul ist ausgecheckt. Im Runner-Klon steht stattdessen „detached HEAD“, der Klon ist flach und enthält nur einen Commit, es gibt keine Tags, `git describe` scheitert mit „No names found“, und das Submodul erscheint mit einem Minuszeichen als nicht initialisiert, worauf das Skript mit einem Hinweis reagiert. Aus jedem dieser Befunde folgt eine konkrete Maßnahme: Historie und Tags nachladen (`git fetch --unshallow --tags`, Modul 07, Abschnitt 7.8), Submodule initialisieren oder den Checkout-Schritt der Pipeline entsprechend konfigurieren, und Zugriffe des Pipeline-Tokens auf die Submodul-URLs prüfen. Der Befund „detached HEAD“ ist dabei nur dann ein Problem, wenn die Pipeline zurückpushen soll; dann muss sie den Branch ausdrücklich auschecken.

> **Merke**
>
> Bei „lokal geht es, in der Pipeline nicht“ zuerst den *Checkout* vergleichen: Tiefe der Historie, Tags, detached HEAD, Submodule, Credentials. Eine Diagnose-Ausgabe, die man lokal und im Runner gleich erzeugt, spart jedes Raten. Die Zugangsdaten des Runners stammen vom Pipeline-Token, nicht von der Person.

### Typische Fehler und Stolperfallen.

Falsch ist, beim ersten Fehler die Pipeline-Konfiguration wild zu ändern oder Befehle zu wiederholen, bis sie zufällig durchlaufen. Richtig ist, mit der Diagnose den tatsächlichen Zustand zu erfassen. Zweiter Fehler: Zugangsdaten zum Debuggen im Log auszugeben (etwa mit `set -x` oder `env`); damit landen Secrets in Pipeline-Protokollen, die viele Personen lesen. Das Skript im Beispiel gibt aus diesem Grund nur den Namen eines Remotes aus, nie die URL. Dritter Fehler: Den Fehler im Git-Befehl zu suchen, wenn er im Zugriff liegt. „Permission denied“ oder eine Passwortabfrage in der Pipeline bedeuten fehlende oder abgelaufene Berechtigung, nicht einen Fehler der Konfiguration von Git. Vierter Fehler: Fehler nur zu beheben, ohne die Ursache zu dokumentieren; derselbe Fehler kehrt zurück, wenn ein Token abläuft oder ein Runner-Image aktualisiert wird.

### Praxisbezug.

Teams pflegen eine kurze Liste bekannter Fehlerbilder mit Ursache und Lösung (siehe Runbooks in Modul 10): „No names found“ (Tags und Tiefe), „dubious ownership“ (`safe.directory` oder Benutzer), „could not read Username“ (kein Credential in der Pipeline), leere Submodul-Verzeichnisse (nicht initialisiert). Mit der Diagnose-Ausgabe aus diesem Abschnitt in jeder Pipeline lassen sich die meisten Fälle schon aus dem Protokoll lösen, ohne den Runner selbst zu untersuchen.

## Übungen zu Modul 09

**Aufgabe 1.** Nennen Sie vier Git-Ereignisse, die Pipelines auslösen können, und ordnen Sie jedem einen typischen Prüfumfang zu.

**Aufgabe 2.** Worin unterscheiden sich der Zwei-Punkte- und der Drei-Punkte-Vergleich bei `git diff`, und warum nutzt man für die Änderungsliste eines Pull Requests den Drei-Punkte-Vergleich?

**Aufgabe 3.** Warum sind clientseitige Hooks keine Durchsetzung? Welche Mechanismen setzen Regeln tatsächlich durch?

**Aufgabe 4.** Welche Maßnahmen machen einen Build reproduzierbar? Nennen Sie mindestens vier.

**Aufgabe 5.** Praktisch: Schreiben Sie einen `commit-msg`-Hook, der den Betreff auf das Muster `typ: text` prüft, legen Sie ihn in `.githooks` ab und aktivieren Sie ihn über `core.hooksPath`. Testen Sie einen gültigen und einen ungültigen Commit sowie `--no-verify`.

**Aufgabe 6.** Praktisch: Bauen Sie den serverseitigen `pre-receive`-Check aus Abschnitt 9.2 nach und erweitern Sie ihn so, dass er zusätzlich prüft, ob die Datei `VERSION` im neuen Stand vorhanden ist. Testen Sie Push mit und ohne die Datei.

**Aufgabe 7.** Praktisch: Schreiben Sie ein Skript, das in einem Monorepo mit mindestens drei Verzeichnissen unter `services/` die geänderten Dienste ermittelt, und legen Sie einen Testfall an, in dem die Änderung nur an einer gemeinsamen Bibliothek liegt. Was geschieht bei einem flachen Klon, und wie sichern Sie das ab?

**Aufgabe 8.** Praktisch (Transfer): Schreiben Sie ein Diagnoseskript für einen Runner, das (a) prüft, ob der Klon flach ist, (b) ob Tags vorhanden sind, (c) ob ein Branch oder ein detached HEAD vorliegt und (d) bei Bedarf die passenden Korrekturbefehle ausgibt. Führen Sie es in einem Vollklon und in einem Klon mit `--depth=1` aus.

## Lösungen zu Modul 09

**Lösung 1.** Push auf einen Branch (schnelle Prüfungen: Build, Unit-Tests, Linting), Pull Request oder Merge Request (volle Prüfung vor dem Merge, Pflichtprüfungen), neuer Tag (Release-Build, Signaturprüfung, Veröffentlichung), Zeitplan (nächtliche, aufwendige oder sicherheitsrelevante Prüfungen). Zusätzlich manuelle Auslöser für Deployments.

**Lösung 2.** Zwei Punkte (`a..b`) vergleichen die Stände der beiden Spitzen direkt; Änderungen, die nur auf `a` liegen, erscheinen als Rückwärtsänderung. Drei Punkte (`a...b`) vergleichen `b` mit dem gemeinsamen Vorfahren (der Abzweigung) und zeigen nur, was auf `b` dazukam. Für einen Pull Request zählt, was der Feature-Branch verändert, nicht, was inzwischen auf dem Zielbranch geschah.

**Lösung 3.** Sie liegen lokal, sind nicht Teil des versionierten Repositories und können mit `--no-verify` übersprungen oder gar nicht installiert werden. Durchgesetzt wird auf dem Server (`pre-receive`, `update`, Plattformregeln wie Branch Protection und Pflichtprüfungen) und in der Pipeline.

**Lösung 4.** Zeitstempel aus dem Commit (`SOURCE_DATE_EPOCH`), Quelldateien aus `git archive` statt aus dem Arbeitsverzeichnis, feste Dateireihenfolge (`--sort=name`), feste Eigentümer (`--owner=0 --group=0`), feste Dateirechte (`umask`), `gzip -n`, festgelegte Werkzeugversionen im Build-Image.

**Lösung 5.** Hook wie in 9.3: `head -n 1 "$1" | grep -Eq '^(feat|fix|docs): .+'`; Datei ausführbar machen (`chmod +x`), `git config core.hooksPath .githooks`. Ungültiger Betreff: Commit wird abgebrochen. `--no-verify`: Commit entsteht trotzdem.

**Lösung 6.** Im Hook zusätzlich `[ -f "$tmp/VERSION" ] || { echo "VERSION fehlt" >&2; exit 1; }`. Push ohne die Datei wird abgelehnt („pre-receive hook declined“), Push mit der Datei angenommen. Zu beachten: Der Hook prüft nur die Spitze des Pushes.

**Lösung 7.** Vorlage: `changed.sh` aus 9.6; Testfall mit Änderung nur in `libs/common` erwartet die Dienste, die sie in `DEPS` nennen. Im flachen Klon fehlt die Vergleichsbasis; der Vergleich scheitert oder liefert eine falsche Liste. Absicherung: Basis nachladen (`git fetch --depth=... origin <basis>`) und bei Fehlern auf „alles bauen“ zurückfallen.

**Lösung 8.** Siehe `ci-diagnose.sh` aus 9.8, ergänzt um Hinweise: bei flachem Klon `git fetch --unshallow --tags origin`, bei fehlenden Tags `git fetch --tags origin`, bei detached HEAD den Hinweis auf `git switch <branch>`, falls die Pipeline zurückschreiben muss. Im Vollklon sollten alle Hinweise entfallen.
