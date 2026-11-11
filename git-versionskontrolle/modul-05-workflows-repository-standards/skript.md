# Modul 05 - Git Workflows und Repository-Standards

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (5.1 bis 5.9), danach folgen „Übungen zu Modul 05“ (8 Aufgaben) und „Lösungen zu Modul 05“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 12 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 12 h, die Summe der Lerneinheiten ergibt 9 h. Dieses Skript folgt dem höheren Wert und führt deshalb die Beispiele ausführlicher aus.

**Lernziele des Moduls:** Trunk-Based Development, GitFlow, Release-Strategien und Governance planen; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Nachdem wir Branches, Merges und Remotes technisch beherrschen, geht es nun um die Frage, wie ein Team sie sinnvoll organisiert: welche Branching-Modelle es gibt, wie man Hauptlinien schützt, welche Dateien und Regeln zu einem gepflegten Repository gehören und wie man das Ganze dauerhaft steuert.

| **Nr.** | **Lerneinheit**          | **Dauer (h)** | **Prio** |
|:--------|:-------------------------|--------------:|:--------:|
| 5.1     | Trunk-Based Development  |           1,0 |    1     |
| 5.2     | GitFlow                  |           1,0 |    1     |
| 5.3     | Release Branching        |           1,0 |    1     |
| 5.4     | Branch Protection        |           1,0 |    1     |
| 5.5     | Repository Standards     |           1,0 |    1     |
| 5.6     | Monorepo vs Multirepo    |           1,0 |    1     |
| 5.7     | Large Repositories       |           1,0 |    2     |
| 5.8     | Commit- und PR-Standards |           1,0 |    2     |
| 5.9     | Governance-Prozess       |           1,0 |    2     |
|         | **Summe**                |       **9,0** |          |

## 5.1 Trunk-Based Development

Beim *Trunk-Based Development* (TBD) arbeitet das Team auf einer einzigen gemeinsamen Hauptlinie, dem *Trunk*, in Git meist `main`. Statt langlebiger Branches integriert jede Person ihre Änderungen in kleinen Schritten, mindestens täglich, in diese Hauptlinie. Der Grundgedanke: Die Kosten einer Zusammenführung wachsen mit der Zeit, in der zwei Linien getrennt leben. Wer ständig integriert, hat nie große Konflikte, weil die Linien sich nie weit voneinander entfernen. Der Trunk bleibt dabei jederzeit in einem auslieferbaren Zustand; das erreicht man durch automatische Tests bei jedem Merge.

Die Beschreibung des Verfahrens (Unleash, abgerufen am 28.10.2026) nennt als Richtwerte: Feature-Branches sind optional, und wenn man sie nutzt, sollten sie sehr kurzlebig sein, grob Stunden bis ein Tag, und nie länger als einige Tage. Teams legen ein maximales Branch-Alter fest und holen regelmäßig den Stand des Trunks. Ein Einwand liegt auf der Hand: Wie baut man eine Funktion, die mehrere Wochen braucht, wenn man alle paar Stunden integriert, ohne dass Halbfertiges bei den Nutzern landet? Die Antwort sind *Feature-Flags* (auch *Feature-Toggles*): Der neue Code wird zwar in den Trunk gemergt, ist aber durch einen Schalter deaktiviert. Das entkoppelt das *Deployment* (Code gelangt in Produktion) vom *Release* (Nutzer sehen die Funktion). Dieselbe Quelle nennt als Vorteil, dass sich eine problematische Funktion schnell abschalten lässt, ohne das gesamte Deployment zurückzurollen, und dass man Funktionen zunächst nur ausgewählten Nutzern oder Umgebungen freigeben kann.

Das Beispiel zeigt das Muster in einem minimalen Projekt: Eine neue Funktion wird in kleinen Schritten auf `main` integriert, aber erst durch den Flag in einer Konfigurationsdatei eingeschaltet. Dazu kommt eine Abfrage, die zu alte Branches findet, ein Messinstrument für die Branch-Disziplin.

```bash
git init -b main tbd && cd tbd
git config user.name "Dev"; git config user.email dev@example.org

cat > app.sh <<'EOF'
#!/bin/sh
. ./flags.conf
echo "Startseite"
if [ "$NEUE_SUCHE" = "an" ]; then
  echo "Suche (neu)"
fi
EOF
echo 'NEUE_SUCHE=aus' > flags.conf
git add . && git commit -m "App mit Feature-Flag NEUE_SUCHE (ausgeschaltet)"

# Schritt 1: kurzlebiger Branch, Rohbau der Suche, Flag bleibt aus
git switch -c feature/suche-1
echo 'echo "Suchlogik: Platzhalter"' >> app.sh
git commit -am "Suche: Platzhalter fuer Logik"
git switch main
git merge --ff-only feature/suche-1
git branch -d feature/suche-1

sh app.sh
# Schritt 2: Funktion einschalten, wenn sie fertig ist
echo 'NEUE_SUCHE=an' > flags.conf
git commit -am "NEUE_SUCHE eingeschaltet"
sh app.sh

# Disziplin pruefen: Branches ohne Aktivitaet seit 3 Tagen
git for-each-ref refs/heads --format='%(refname:short) %(committerdate:unix)' |
while read name ts; do
  alter=$(( ( $(date +%s) - ts ) / 86400 ))
  if [ "$alter" -gt 3 ]; then echo "ALT: $name ($alter Tage)"; fi
done
```

Das Skript `app.sh` liest seine Einstellungen aus `flags.conf`. Solange dort `NEUE_SUCHE=aus` steht, ist der neue Code unsichtbar, auch wenn er schon in `main` liegt. Der kurzlebige Branch wird mit `git merge --ff-only` zurückgeführt; das ist die Absicherung, dass `main` linear bleibt und der Branch vorher aktualisiert wurde. Wenn die Funktion fertig ist, genügt ein winziger Commit, der den Flag umlegt, und dieser Commit lässt sich bei Problemen ebenso einfach zurücknehmen. Die Schleife am Ende berechnet das Alter jedes lokalen Branches über das Datum seines letzten Commits und meldet solche, die älter sind als drei Tage. In einer Pipeline könnte man dies als Warnung ausgeben.

> **Merke**
>
> Trunk-Based Development ersetzt nicht den Review, sondern verlangt kleine Änderungen, schnelle Reviews, starke automatische Tests und Feature-Flags für unfertige Funktionen. Ohne diese Voraussetzungen wird aus „täglich integrieren“ schnell „täglich kaputt machen“.

### Typische Fehler und Stolperfallen.

Falsch ist es, TBD einzuführen, ohne Testautomatisierung zu haben. Dann landet Halbfertiges ungeprüft im Trunk, und das Team verliert das Vertrauen in die Hauptlinie. Richtig ist, zuerst eine verlässliche Pipeline aufzubauen, bei der jeder Merge Build und Tests durchläuft. Ein zweiter Fehler sind Feature-Flags, die nie entfernt werden: Jeder Flag verdoppelt die Zahl möglicher Zustände des Systems und wird zur technischen Schuld. Nach dem vollständigen Rollout sollte man den Flag samt totem Code entfernen. Dritter Fehler: Kurzlebig heißt nicht, dass jeder Branch einen Pull Request mit mehrstündigem Review benötigt. Teams, die Pull Requests nutzen, brauchen dafür sehr kleine Änderungen und schnelle Reaktionszeiten; sonst blockiert der Prozess das Prinzip.

### Praxisbezug.

TBD ist verbreitet bei Teams mit kontinuierlicher Auslieferung (mehrmals täglich in Produktion), etwa bei Webdiensten. Dort ergänzt man es um Mechanismen wie Dark Launches, bei denen Code ausgeliefert, aber nicht aktiviert wird, und schrittweise Aktivierung für Prozentsätze der Nutzer. Für Software mit festen Versionen und lange gepflegten Auslieferungen passt es nur mit Zusatzmaßnahmen, etwa kurzlebigen Release-Branches (siehe Abschnitt 5.3).

## 5.2 GitFlow

*GitFlow* ist ein Branching-Modell, das Vincent Driessen 2010 in einem vielbeachteten Beitrag beschrieben hat; es wurde zum Vorbild für viele Teams. Es kennt zwei langlebige Branches: `main` (oder `master`) enthält ausschließlich ausgelieferte Versionen, jeder Commit dort entspricht einem Release und trägt einen Tag; `develop` ist die Integrationslinie für die laufende Entwicklung. Dazu kommen drei Arten kurzlebiger Branches. *Feature-Branches* zweigen von `develop` ab und kehren dorthin zurück. *Release-Branches* zweigen von `develop` ab, sobald genug Funktionen für eine Version beisammen sind; auf ihnen wird nur noch stabilisiert (Fehlerbehebung, Versionsnummern), neue Funktionen sind tabu. Am Ende wird der Release-Branch in `main` gemergt, getaggt und zurück in `develop` gemergt. *Hotfix-Branches* zweigen von `main` ab, wenn in Produktion ein dringender Fehler auftritt, und werden in `main` und `develop` gemergt.

Das Modell passt zu Software, die in diskreten Versionen ausgeliefert wird und von der mehrere Versionen parallel in Betrieb sind, etwa Desktop-Anwendungen, Bibliotheken oder Firmware. Der Preis ist Komplexität: Mehr Branches, mehr Merges, mehr Gelegenheiten für Fehler und lange Zeiträume ohne Integration. Der Urheber selbst hat später, soweit dem Autor bekannt, ergänzend darauf hingewiesen, dass das Modell für kontinuierlich ausgelieferte Webanwendungen weniger geeignet ist und dort einfachere Abläufe vorzuziehen sind. (Diese Angabe stammt aus dem Wissensstand des Autors; der Originalbeitrag wurde in dieser Sitzung nicht abgerufen. Prüfen Sie sie bei Bedarf selbst.)

Das folgende Beispiel durchläuft einen kompletten GitFlow-Zyklus mit gewöhnlichen Git-Befehlen, ohne Zusatzwerkzeug: Feature, Release mit Fehlerbehebung, Auslieferung mit Tag, Rückführung in `develop` und schließlich einen Hotfix.

```bash
git init -b main gf && cd gf
git config user.name "Dev"; git config user.email dev@example.org
echo "v0" > version.txt && git add . && git commit -m "Initial"
git switch -c develop

# Feature
git switch -c feature/login develop
echo "login" > login.txt && git add . && git commit -m "Login"
git switch develop && git merge --no-ff feature/login -m "Merge feature/login"
git branch -d feature/login

# Release vorbereiten
git switch -c release/1.0.0 develop
echo "1.0.0" > version.txt && git commit -am "Version 1.0.0"
echo "fix" > fix.txt && git add . && git commit -m "Fehler in Release behoben"

# Release abschliessen
git switch main && git merge --no-ff release/1.0.0 -m "Release 1.0.0"
git tag -a v1.0.0 -m "Release 1.0.0"
git switch develop && git merge --no-ff release/1.0.0 -m "Release 1.0.0 zurueck in develop"
git branch -d release/1.0.0

# Hotfix auf Produktion
git switch -c hotfix/1.0.1 main
echo "hotfix" > hotfix.txt && git add . && git commit -m "Hotfix"
echo "1.0.1" > version.txt && git commit -am "Version 1.0.1"
git switch main && git merge --no-ff hotfix/1.0.1 -m "Hotfix 1.0.1"
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git switch develop && git merge --no-ff hotfix/1.0.1 -m "Hotfix 1.0.1 nach develop"
git branch -d hotfix/1.0.1

git log --oneline --graph --all --decorate
```

Alle Zusammenführungen verwenden `--no-ff`, damit jede Funktion und jedes Release als eigener Merge-Commit sichtbar bleibt; das ist ein Kernmerkmal des Modells. Das Release wird zuerst in `main` gemergt und getaggt, dann in `develop`: Würde man den Rückweg vergessen, gingen Korrekturen aus dem Release für künftige Versionen verloren. Genau diese doppelte Zusammenführung ist die häufigste Fehlerquelle des Modells. Beim Hotfix gilt dasselbe: Er geht in `main` und in `develop` (oder, falls gerade ein Release-Branch existiert, in diesen). Der abschließende Graph zeigt die typische Struktur mit mehreren parallel laufenden Linien.

### Typische Fehler und Stolperfallen.

Ein klassischer Fehler ist, den Merge zurück nach `develop` zu vergessen. Dann fehlt die Korrektur in der nächsten Version, und der Fehler taucht nach dem nächsten Release erneut auf. Falsch ist, nach dem Hotfix nur `main` zu aktualisieren; richtig ist eine Checkliste oder Automatisierung, die beide Merges erzwingt. Ein zweiter Fehler ist, Feature-Branches lange offen zu halten, weil `develop` „ja ohnehin nicht produktiv“ ist. Das widerspricht dem Integrationsgedanken und führt zu den bekannten großen Konflikten. Drittens: Wer GitFlow wählt, weil es „professionell“ wirkt, aber kontinuierlich ausliefert, zahlt Komplexität ohne Nutzen. Die Frage sollte lauten: Gibt es bei uns wirklich mehrere gleichzeitig gepflegte Versionen?

### Praxisbezug.

GitFlow begegnet Ihnen vor allem in älteren Projekten und Unternehmen mit festen Release-Zyklen. Wer ein Projekt übernimmt, erkennt es an den Branches `develop` und `release/*`. Ein Wechsel auf ein einfacheres Modell ist oft möglich, braucht aber eine Absprache mit Qualitätssicherung und Betrieb, weil sich Abnahmeprozesse an das bestehende Modell anlehnen.

## 5.3 Release Branching

Wann immer mehrere Versionen einer Software gleichzeitig gepflegt werden, braucht man *Release-Branches*: Zweige, die den Stand einer ausgelieferten Version über die Zeit hinweg festhalten und pflegen. Ein Unternehmen liefert beispielsweise Version 2.0 aus und entwickelt auf `main` bereits 3.0, während Kunden noch auf 2.0 sind und Sicherheitskorrekturen erwarten. Dafür zweigt man zum Zeitpunkt der Auslieferung `release/2.0` ab. Auf diesem Branch landen von nun an nur noch Fehlerbehebungen; jede Auslieferung wird mit einem Tag markiert (`v2.0.0`, `v2.0.1`, …).

Zentral ist die Frage, in welcher Richtung Korrekturen fließen. Die bewährte Regel lautet *upstream first*: Eine Korrektur wird zuerst auf `main` gemacht und von dort auf die gepflegten Release-Branches übertragen (*Backport*, per `git cherry-pick -x`, siehe Modul 03). So kann es nicht passieren, dass ein Fehler in der alten Version behoben wird, aber in der neuen bestehen bleibt. Dazu gehört eine *Support-Politik*: Welche Versionen werden wie lange gepflegt? Ohne klare Antwort wächst die Zahl der Release-Branches ungebremst und der Backport-Aufwand mit. Üblich sind Regeln wie „die letzten beiden Minor-Versionen“ oder „die aktuelle und die Langzeit-Version“.

Das Beispiel zeigt einen Release-Branch mit einem Backport und ein hilfreiches Werkzeug, um offene Backports zu finden: `git cherry` listet Commits, die in einem Branch noch fehlen, wobei bereits per Cherry-pick übernommene Änderungen erkannt werden.

```bash
git init -b main rel && cd rel
git config user.name "Dev"; git config user.email dev@example.org
echo "Funktion A" > a.txt && git add . && git commit -m "A"
echo "2.0.0" > version.txt && git add . && git commit -m "Version 2.0.0"
git branch release/2.0
git tag -a v2.0.0 -m "Release 2.0.0"

# Entwicklung geht weiter (3.0)
echo "Funktion B" > b.txt && git add . && git commit -m "Neue Funktion B (nur 3.0)"
echo "fix 1" > fix1.txt && git add . && git commit -m "Fix: Absturz bei leerer Eingabe"
FIX1=$(git rev-parse HEAD)
echo "fix 2" > fix2.txt && git add . && git commit -m "Fix: Zeitzonenfehler"

# Welche Commits aus main fehlen im Release? (+ = fehlt)
git cherry -v release/2.0 main

# Backport des ersten Fixes
git switch release/2.0
git cherry-pick -x $FIX1
git tag -a v2.0.1 -m "Release 2.0.1"
git switch main

git cherry -v release/2.0 main
git log --oneline --decorate release/2.0
```

Die erste Ausgabe von `git cherry -v release/2.0 main` listet drei Commits mit einem Pluszeichen: Alles auf `main`, was im Release fehlt. Nach dem Backport zeigt dieselbe Abfrage beim übertragenen Fix ein Minuszeichen: Git erkennt, dass die gleiche Änderung im Release schon enthalten ist, auch wenn der Commit einen anderen Hash hat. Das funktioniert über einen Vergleich des Patch-Inhalts (*patch-id*). So behält man den Überblick, welche Fixes noch nicht in welche Release-Linie übertragen wurden. Der Tag `v2.0.1` markiert die ausgelieferte Korrekturversion auf dem Release-Branch.

### Typische Fehler und Stolperfallen.

Falsch ist, Korrekturen nur im Release-Branch zu machen und in `main` zu vergessen (*downstream-only*). Der Fehler kehrt dann mit der nächsten Hauptversion zurück. Richtig ist *upstream first*. Ein zweiter Fehler ist, Release-Branches mit neuen Funktionen anzureichern, „weil der Kunde es braucht“. Das zerstört den Zweck der Stabilisierung. Dritter Fehler: Die Release-Linien nie zu beenden. Wer 2.0, 2.1, 2.2 und 2.3 parallel pflegt, bindet die Entwicklung. Eine dokumentierte Support-Politik mit Enddaten verhindert das.

### Praxisbezug.

Bei Standardsoftware und Bibliotheken ist die Release-Pflege ein feststehender Prozess: Die Fehlerverfolgung markiert, in welche Versionen eine Korrektur gehört, und ein Build-System erzeugt aus den Tags die Auslieferungspakete. Wichtig ist die Nachvollziehbarkeit vom Tag bis zum Artefakt (Modul 07). Auch bei Linux-Distributionen und Frameworks sind *LTS-Versionen* Release-Branches, die über Jahre mit Sicherheitskorrekturen versorgt werden.

## 5.4 Branch Protection

Regeln, die nur auf Absprachen beruhen, werden irgendwann gebrochen, meist aus Versehen. *Branch Protection* setzt wichtige Regeln auf dem Server technisch durch, sodass sie für alle gelten, auch unter Zeitdruck. Geschützt werden typischerweise der Hauptbranch, Release-Branches und Tags. Typische Regeln sind: kein direkter Push, Änderungen nur über Pull Requests, mindestens eine oder zwei Freigaben, bestandene automatische Prüfungen, kein Force-Push, kein Löschen und, wo gefordert, signierte Commits.

Die Plattformen setzen das unterschiedlich um. Auf GitHub haben *Rulesets* die klassischen Branch-Protection-Regeln ergänzt. Die Dokumentation (abgerufen am 28.10.2026) nennt unter anderem folgende Regeln: Erzeugen, Aktualisieren und Löschen von Branches oder Tags einschränken (das Löschen ist standardmäßig eingeschränkt), Force-Pushes blockieren (standardmäßig aktiv), lineare Historie verlangen, signierte Commits verlangen, Pull Requests mit Pflichtfreigaben, Code-Owner-Review, Zurückziehen veralteter Freigaben bei neuen Änderungen, Freigabe durch eine andere Person als den letzten Pusher, Auflösen aller Kommentare, erlaubte Merge-Methoden einschränken, bestandene Statusprüfungen verlangen sowie Dateipfade, Dateierweiterungen und Dateigrößen einschränken. Weitere Regeln betreffen Scans (etwa auf offene Secret-Funde). Weil Ruleset-Funktionen sich mit dem Tarif und der Zeit ändern, prüfen Sie die aktuelle Seite der Plattform, bevor Sie eine Richtlinie festschreiben. GitLab (*Protected Branches*, Merge-Request-Freigaben) und Azure Repos (*Branch Policies*) bieten vergleichbare Funktionen unter anderen Namen; diese Angaben stammen aus dem Wissensstand des Autors und wurden nicht abgerufen. Der Vergleich der Plattformen folgt in Modul 11.

Wie funktioniert die Durchsetzung im Kern? Bei jedem Push prüft der Server, ob er erlaubt ist, und lehnt ihn sonst ab. In reinem Git geschieht das durch serverseitige Hooks (`pre-receive` für den gesamten Push, `update` je Referenz) und Konfigurationen wie `receive.denyNonFastForwards`. Das folgende Beispiel baut zwei solche Regeln nach: Tags sind unveränderlich (ein bestehender Tag darf nicht verschoben oder gelöscht werden), und der Branch `main` akzeptiert nur Pushes, die Fast-forward sind und bei denen alle neuen Commits mit `Signed-off-by` versehen sind. Es ist ein Lehrbeispiel, um den Mechanismus zu verstehen.

```bash
git init --bare -b main server.git
cat > server.git/hooks/update <<'EOF'
#!/bin/sh
# Argumente: Referenzname, alter Hash, neuer Hash
ref="$1"; old="$2"; new="$3"
zero="0000000000000000000000000000000000000000"

# Regel 1: Tags sind unveraenderlich
case "$ref" in
  refs/tags/*)
    if [ "$old" != "$zero" ]; then
      echo "Tag $ref existiert bereits und darf nicht veraendert werden." >&2
      exit 1
    fi ;;
esac

# Regel 2: main nur mit Signed-off-by in allen neuen Commits
if [ "$ref" = "refs/heads/main" ]; then
  if [ "$old" = "$zero" ]; then range="$new"; else range="$old..$new"; fi
  for c in $(git rev-list $range); do
    git log -1 --format=%B $c | grep -q '^Signed-off-by: ' || {
      echo "Commit $(git rev-parse --short $c) ohne Signed-off-by." >&2
      exit 1
    }
  done
fi
exit 0
EOF
chmod +x server.git/hooks/update
git config -f server.git/config receive.denyNonFastForwards true

git clone server.git dev && cd dev
git config user.name "Dev"; git config user.email dev@example.org
echo "a" > a.txt && git add .
git commit -m "Ohne Signatur"
git push -u origin main
git commit --amend -s --no-edit
git push -u origin main

git tag -a v1.0 -m "Release 1.0"
git push origin v1.0
git tag -d v1.0
echo "b" > b.txt && git add . && git commit -s -m "Zweiter Commit"
git tag -a v1.0 -m "Verschoben"
git push --force origin v1.0
```

Der erste Push wird abgelehnt, weil der Commit keine `Signed-off-by`-Zeile hat. Der Zusatz `-s` bei `git commit --amend -s` fügt sie hinzu, danach wird der Push angenommen. Ein neuer Tag `v1.0` wird angenommen. Das Verschieben auf einen anderen Commit (hier: lokal löschen, neu anlegen, mit `--force` pushen) lehnt der Hook dagegen ab. Entscheidend ist, dass der Hook *auf dem Server* liegt; ein lokaler Hook könnte jeder umgehen, indem er ihn nicht installiert oder mit `--no-verify` überspringt. Die Plattformregeln wirken aus demselben Grund serverseitig.

### Typische Fehler und Stolperfallen.

Ein häufiger Fehler ist, Schutzregeln einzurichten, aber Administratoren davon auszunehmen. Wenn Admins direkt pushen dürfen, passiert genau das irgendwann im Notfall, und der Notfall wird zum Dauerzustand. Falsch: „Regeln gelten für alle außer uns“. Richtig: Regeln gelten für alle; für echte Notfälle gibt es einen dokumentierten Ausnahmeprozess mit Nachweis. Ein zweiter Fehler: Zu viele Pflichten ohne Automatisierung. Wer zwei Freigaben, signierte Commits und lineare Historie verlangt, aber keine Vertretungsregeln hat, blockiert sein Team bei Urlaub und Krankheit. Dritter Fehler: Nur den Hauptbranch zu schützen. Release-Branches und Tags sind ebenso kritisch, weil Auslieferungen daraus gebaut werden.

### Praxisbezug.

In Audits ist die Branch-Schutz-Konfiguration ein Standardnachweis: Es wird gezeigt, dass kein Code ohne Freigabe in Produktion gelangt. Viele Unternehmen verwalten diese Einstellungen als Code (*Configuration as Code*) oder prüfen sie regelmäßig automatisch gegen eine Vorgabe, damit sich keine Abweichungen einschleichen (Modul 11).

## 5.5 Repository Standards

Ein Repository ist mehr als Code. Wer es betritt, sollte binnen Minuten verstehen, worum es geht, wie man es baut, wie man beiträgt und an wen man sich bei Problemen wendet. Dafür haben sich Standarddateien etabliert. Die `README.md` erklärt Zweck, Installation, Nutzung und Verweise. `CONTRIBUTING.md` beschreibt, wie man beiträgt: Branch-Namen, Commit-Konventionen, Testanforderungen, Ablauf des Pull Requests. `CODEOWNERS` legt Zuständigkeiten fest (Modul 04). `SECURITY.md` sagt, wie Sicherheitslücken gemeldet werden (Modul 08). `LICENSE` nennt die Lizenz, unter der der Code genutzt werden darf. Dazu kommen technische Dateien: `.gitignore` (Modul 02), `.gitattributes` für Zeilenenden und Binärdateien und `.editorconfig` für einheitliche Editor-Einstellungen.

Der Nutzen liegt im *Onboarding* und in der *Governance*. Neue Teammitglieder brauchen weniger Hilfe, wenn die Antworten im Repository stehen, und Reviewer können auf `CONTRIBUTING.md` verweisen, statt Regeln immer neu zu erklären. Für die Auswahl der Lizenz gilt: Sie ist eine rechtliche Entscheidung mit Folgen für Nutzung und Weitergabe, die das Unternehmen oder Projekt bewusst treffen muss; dieses Skript gibt dazu keine Rechtsberatung. In der Praxis legt man Vorlagen fest, damit alle neuen Repositories gleich starten, entweder als *Template-Repository* auf der Plattform oder als Skript.

Das Beispiel erzeugt einen Satz Standarddateien mit einem Skript und prüft danach in einem zweiten Skript, ob ein Repository alle erwarteten Dateien enthält. Ein solcher Check lässt sich in eine Pipeline einbauen.

```bash
git init -b main standards && cd standards
git config user.name "Dev"; git config user.email dev@example.org

cat > README.md <<'EOF'
# Projektname
Kurze Beschreibung: Was tut das Projekt, fuer wen?

## Schnellstart
1. Repository klonen
2. Abhaengigkeiten installieren
3. Tests ausfuehren

## Mitwirken
Siehe CONTRIBUTING.md.
EOF

cat > CONTRIBUTING.md <<'EOF'
# Mitwirken
- Branch-Namen: feature/TICKET-kurzbeschreibung
- Commits: Conventional Commits (feat, fix, docs ...)
- Pull Requests: klein halten, Template ausfuellen, Tests ergaenzen
- Mindestens eine Freigabe, Pipeline muss gruen sein
EOF

cat > SECURITY.md <<'EOF'
# Sicherheit
Sicherheitsluecken bitte nicht oeffentlich melden, sondern an security@firma.example.
EOF

mkdir -p .github
printf "*  @firma/kernteam\n" > .github/CODEOWNERS
printf "* text=auto\n*.png binary\n" > .gitattributes
printf "root = true\n[*]\nindent_style = space\nindent_size = 2\nend_of_line = lf\ninsert_final_newline = true\n" > .editorconfig
printf "*.log\n.env\nbuild/\n" > .gitignore
echo "Lizenz: vom Projekt festzulegen (Rechtsfrage)" > LICENSE.todo

cat > pruefe-standards.sh <<'EOF'
#!/bin/sh
fehler=0
for f in README.md CONTRIBUTING.md SECURITY.md .gitignore .gitattributes \
         .editorconfig .github/CODEOWNERS; do
  if [ -f "$f" ]; then echo "OK      $f"; else echo "FEHLT   $f"; fehler=1; fi
done
if ls LICENSE* >/dev/null 2>&1; then echo "OK      Lizenzdatei vorhanden"
else echo "FEHLT   LICENSE"; fehler=1; fi
exit $fehler
EOF
git add . && git commit -m "Repository-Standards angelegt"
sh pruefe-standards.sh; echo "Rueckgabewert: $?"
```

Die Dateien bilden das Grundgerüst. Die `README.md` folgt dem Muster Zweck, Schnellstart, Mitwirken. Die `CONTRIBUTING.md` hält fest, was in Modul 02 bis 04 besprochen wurde. Die Datei `.gitattributes` mit `* text=auto` überlässt Git die Normalisierung der Zeilenenden, `.editorconfig` legt Einrückung und Zeilenende für Editoren fest. Die Lizenz ist bewusst nur ein Platzhalter `LICENSE.todo`; die Wahl der Lizenz ist Sache der Verantwortlichen. Das Prüfskript meldet `OK` oder `FEHLT` je Datei und gibt den Rückgabewert 0 oder 1 zurück, was Pipelines auswerten. Das Skript akzeptiert jede Datei, die mit `LICENSE` beginnt, sodass später `LICENSE` oder `LICENSE.md` genügt; der Platzhalter `LICENSE.todo` erfüllt die Prüfung nur formal und sollte ersetzt werden.

### Typische Fehler und Stolperfallen.

Falsch ist, Standarddateien einmal anzulegen und nie zu pflegen. Eine `README.md` mit veralteten Installationsschritten ist schlimmer als keine, weil sie Zeit kostet und Vertrauen zerstört. Richtig ist, Änderungen am Verfahren immer zusammen mit der Dokumentation im selben Pull Request zu machen. Ein zweiter Fehler: Jedes Repository mit anderen Regeln. Ohne Vorlage entstehen Inseln, in denen niemand sich zurechtfindet. Dritter Fehler: Vertrauliche Informationen (interne URLs, Kontaktdaten, Schlüssel) in Standarddateien öffentlicher Repositories aufzunehmen.

### Praxisbezug.

Viele Plattformen zeigen fehlende Standarddateien in einer „Community-Übersicht“ an und erlauben Vorlagen auf Organisationsebene, die für alle Repositories gelten, die selbst keine eigene Datei haben. Für interne Repositories lohnt sich ein Template-Repository mit CI-Konfiguration, Linter-Einstellungen und Standarddateien, aus dem alle neuen Projekte erzeugt werden.

## 5.6 Monorepo vs Multirepo

Eine der grundlegenden Strukturentscheidungen ist, ob man den gesamten Code in *einem* Repository führt (*Monorepo*) oder jedes Projekt, jede Bibliothek, jeden Dienst in einem eigenen (*Multirepo*, auch *Polyrepo*). Beide Ansätze werden in großen Organisationen erfolgreich eingesetzt, beide haben klare Kosten. Ein Monorepo macht Änderungen über Projektgrenzen hinweg leicht: Eine Schnittstelle und alle ihre Aufrufer lassen sich in *einem* Commit und *einem* Pull Request ändern, Abhängigkeitsversionen sind automatisch konsistent, und Werkzeuge wie Refactoring und Suche sehen alles. Der Preis: Das Repository wächst, Klon- und Statusoperationen werden langsamer, die Pipeline muss herausfinden, welche Teile betroffen sind, und Zugriffsrechte lassen sich nur schwer pro Teilprojekt trennen, weil Git Rechte auf Repository-Ebene kennt.

Ein Multirepo ist das Gegenstück: Jedes Repository hat eigene Versionen, eigene Rechte, eine kleine, schnelle Pipeline und klare Verantwortung. Der Preis sind Koordinationskosten. Eine Änderung, die mehrere Repositories betrifft, braucht mehrere Pull Requests, abgestimmte Versionen und Release-Reihenfolgen; Abhängigkeiten driften auseinander, wenn man sie nicht aktiv aktualisiert. Die Entscheidung hängt an Fragen wie: Wie stark sind die Teile gekoppelt? Wer besitzt welchen Teil? Wie oft ändert man gleichzeitig mehrere? Müssen Rechte getrennt sein? Wie groß wird das Repository?

|                          | **Monorepo**                          | **Multirepo**                      |
|:-------------------------|:--------------------------------------|:-----------------------------------|
| Änderungen über Projekte | ein Commit, ein Pull Request          | mehrere PRs, Versionsabstimmung    |
| Abhängigkeiten           | immer konsistent (ein Stand)          | Versionen können auseinanderlaufen |
| Pipeline                 | muss Betroffenes erkennen             | klein und einfach je Repository    |
| Zugriffsrechte           | schwer pro Teilprojekt trennbar       | pro Repository fein steuerbar      |
| Größe und Tempo          | Klon und Status können langsam werden | klein und schnell                  |
| Wiederverwendung         | Code direkt teilbar                   | über Pakete und Versionen          |

Wer ein Monorepo betreibt, braucht Werkzeuge für die zwei zentralen Probleme: das Erkennen betroffener Teile und das Begrenzen dessen, was ein Entwickler auf der Platte haben muss. Das erste löst man über *Pfadfilter*: Man ermittelt aus dem Diff, welche Verzeichnisse sich geändert haben, und baut nur diese (Modul 09). Das zweite löst *Sparse Checkout*, bei dem im Arbeitsverzeichnis nur ausgewählte Verzeichnisse erscheinen. Die Git-Dokumentation zu `sparse-checkout` (abgerufen am 28.10.2026) beschreibt, dass der Cone-Modus Verzeichnisse angibt und dabei alles darunter sowie die Dateien der übergeordneten Verzeichnisse einschließt; er ist schneller und der empfohlene Weg, während der Nicht-Cone-Modus mit gitignore-artigen Mustern flexibler ist, aber Leistungsprobleme haben kann.

Das Beispiel baut ein kleines Monorepo mit zwei Diensten und zeigt beide Techniken.

```bash
git init -b main mono && cd mono
git config user.name "Dev"; git config user.email dev@example.org
mkdir -p dienste/api dienste/web libs/gemeinsam
echo "api v1" > dienste/api/main.txt
echo "web v1" > dienste/web/main.txt
echo "lib v1" > libs/gemeinsam/lib.txt
echo "Monorepo" > README.md
git add . && git commit -m "Monorepo angelegt"

echo "api v2" > dienste/api/main.txt
git commit -am "api: Aenderung"
echo "lib v2" > libs/gemeinsam/lib.txt
echo "web v2" > dienste/web/main.txt
git commit -am "lib und web: Aenderung"

# Welche Teilprojekte sind betroffen?
git diff --name-only HEAD~2 HEAD | cut -d/ -f1-2 | sort -u
# Historie nur eines Teilprojekts
git log --oneline -- dienste/api

# Nur das Noetigste auschecken
git sparse-checkout init --cone
git sparse-checkout set dienste/api libs/gemeinsam
git sparse-checkout list
ls
ls dienste
git sparse-checkout disable
ls dienste
```

Die Pfadermittlung schneidet die ersten zwei Pfadelemente ab und liefert eindeutig `dienste/api`, `dienste/web` und `libs/gemeinsam`. Ein Skript könnte daraus ableiten, welche Pipelines laufen müssen. `git log -- dienste/api` zeigt nur die Historie dieses Teilprojekts. Der Sparse Checkout im Cone-Modus mit `set` lässt `dienste/web` aus dem Arbeitsverzeichnis verschwinden; es bleibt im Repository erhalten und erscheint nach `disable` wieder. Im Cone-Modus bleiben außerdem die Dateien im Wurzelverzeichnis (hier `README.md`) sichtbar.

### Typische Fehler und Stolperfallen.

Falsch ist die Entscheidung nach Mode oder Vorbildern großer Firmen. Ein Monorepo mit Tausenden Entwicklern setzt eigene Werkzeuge und Infrastruktur voraus; wer sie nicht hat, erlebt langsame Pipelines und Frust. Ebenso falsch ist ein Multirepo für Komponenten, die immer gemeinsam geändert werden: Das Team führt dann ständig synchronisierte Pull Requests. Richtig ist, Kopplung und Änderungsmuster zu betrachten, bevor man entscheidet. Ein weiterer Fehler beim Monorepo ist eine Pipeline, die bei jeder Änderung alles baut; ohne Pfadfilter verliert man den Geschwindigkeitsvorteil.

### Praxisbezug.

Viele Organisationen landen bei einer Mischform: ein Monorepo pro Produkt oder Team, eigene Repositories für unabhängige, wiederverwendbare Bibliotheken. Auch die Umkehrung ist möglich und aufwendig: Das Zusammenführen vieler Repositories in eines (oder das Aufteilen) ist ein Migrationsprojekt (Modul 12). Daher lohnt es, die Entscheidung bewusst zu treffen und zu dokumentieren.

## 5.7 Large Repositories

Wenn ein Repository sehr groß wird, leidet die Arbeit: Klonen dauert lange, Plattenplatz wird knapp, und Befehle wie `git status` werden träge. Die Ursachen sind meist dieselben: viele große Binärdateien in der Historie, sehr viele Dateien, oder eine sehr lange Historie. Git hat dafür vier Gegenmittel, die man kennen sollte, weil sie an unterschiedlichen Stellen ansetzen.

*Git LFS* (Large File Storage) ersetzt große Dateien im Repository durch kleine Textverweise, sogenannte *Pointer-Dateien*; die eigentlichen Inhalte liegen in einem separaten Speicher auf dem Server. Laut der GitLab-Dokumentation (abgerufen am 28.10.2026) enthält ein Pointer die LFS-Spezifikationsversion, eine Objekt-ID der Form `{Hash-Verfahren}:{Hash}` und die Dateigröße in Byte. Welche Dateien über LFS laufen, wird in `.gitattributes` festgelegt; `git lfs install` richtet einen `pre-push`-Hook und diese Datei ein. Die Quelle weist auch auf Grenzen hin: Das Entfernen der Verfolgung (*untrack*) lässt Dateien in der bestehenden Historie, sodass ältere Branches und Tags die LFS-Objekte weiter brauchen, und das endgültige Entfernen verlangt eine Historienumschreibung, die als destruktiv und unumkehrbar beschrieben wird. Auch Größenlimits der Plattform gelten weiter. *Sparse Checkout* begrenzt, welche Dateien im Arbeitsverzeichnis erscheinen (Abschnitt 5.6). Ein *Partial Clone* (`git clone --filter=blob:none`) lädt zunächst nur Commits und Trees und holt Dateiinhalte (Blobs) erst bei Bedarf nach; kombiniert mit Sparse Checkout lädt man nur, was man braucht. Die Git-Designdokumentation zu Sparse Checkout (abgerufen am 28.10.2026) erwähnt diese Kombination und weist darauf hin, dass Git bei Zugriff auf Pfade außerhalb der Auswahl fehlende Blobs nachzuladen versucht und offline scheitert. Ein *Shallow Clone* (`git clone --depth 1`) holt nur die jüngste Historie, was für CI-Läufe sinnvoll ist, aber Befehle einschränkt, die Historie benötigen.

Das Beispiel zeigt LFS, Partial Clone und Shallow Clone lokal. Das LFS-Beispiel arbeitet ohne Server und zeigt, wie ein Pointer aussieht.

```bash
git init -b main gross && cd gross
git config user.name "Dev"; git config user.email dev@example.org

# --- Git LFS: Pointer statt Inhalt ---
git lfs install --local
git lfs track "*.bin"
head -c 200000 /dev/urandom > modell.bin
echo "Code" > main.txt
git add .gitattributes modell.bin main.txt
git commit -m "Modell ueber LFS"
git lfs ls-files
git cat-file -p HEAD:modell.bin
git lfs pointer --file modell.bin

# --- Partial und Shallow Clone ---
echo "Code v2" > main.txt
git commit -am "main.txt aktualisiert"
git config uploadpack.allowFilter true
cd ..
git clone --no-local --filter=blob:none file://$PWD/gross partial
git -C partial config remote.origin.partialclonefilter
git -C partial rev-list --objects --all --missing=print | grep '^?'
git -C partial log --oneline
git clone --no-local --depth 1 file://$PWD/gross flach
git -C flach log --oneline
git -C flach rev-parse --is-shallow-repository
```

Nach `git lfs track "*.bin"` schreibt LFS eine Regel in `.gitattributes`. Das Committen der Binärdatei speichert in Git nur den Pointer: `git cat-file -p HEAD:modell.bin` zeigt ein paar Zeilen Text mit `version`, `oid sha256:...` und `size 200000`, nicht den Binärinhalt. `git lfs ls-files` listet die über LFS verwalteten Dateien. Der echte Inhalt liegt im LFS-Cache unter `.git/lfs`. Für den Partial Clone muss der Quell-Server Filter erlauben (`uploadpack.allowFilter`); große Plattformen haben das aktiviert. Der Klon mit `--filter=blob:none` enthält die gesamte Historie mit allen Commits und Trees, aber nicht alle Blobs: Die Dateien der ausgecheckten Version werden für das Arbeitsverzeichnis geladen, ältere Fassungen fehlen zunächst. `--missing=print` markiert solche fehlenden Objekte mit einem vorangestellten Fragezeichen; hier ist es die frühere Fassung von `main.txt`. Sie wird bei Bedarf (etwa bei `git show` auf einen älteren Commit) automatisch nachgeladen. Der Shallow Clone mit `--depth 1` zeigt nur einen Commit, und `is-shallow-repository` bestätigt die Verkürzung. Die Option `--no-local` erzwingt in diesem lokalen Versuch den echten Transportweg, sonst würde Git bei Pfaden die Daten einfach kopieren.

### Typische Fehler und Stolperfallen.

Falsch ist es, große Dateien erst committen und später „in LFS verschieben“ zu wollen. Die ursprünglichen Blobs bleiben in der Historie, das Repository bleibt groß, und die Umstellung bestehender Historie braucht ein Migrationswerkzeug und Absprache, weil sie Hashes ändert. Richtig ist, vor dem ersten Commit festzulegen, welche Dateitypen über LFS laufen, und das Team zu informieren, dass `git lfs install` auf jedem Rechner nötig ist. Ein zweiter Fehler: Shallow Clones für Arbeit zu nutzen, die Historie braucht (etwa `git blame`, Merge-Base-Berechnung oder Versionsermittlung aus Tags). Ein dritter: Annehmen, LFS löse jedes Größenproblem. Die Zahl der Dateien, die Historie und die Plattformlimits bleiben davon unberührt.

### Praxisbezug.

Bei Spiele- und Medienprojekten, Machine-Learning-Modellen oder Designdateien ist LFS Standard. In CI-Pipelines setzt man häufig Shallow Clones ein, um Zeit zu sparen; benötigt man dort Tags für die Versionierung, holt man sie gezielt nach (Modul 09). Performance-Messung und Ursachenanalyse bei großen Repositories folgen in Modul 10, die Kosten- und Skalierungsfragen in Modul 12.

## 5.8 Commit- und PR-Standards

Einheitliche Commit-Nachrichten und Pull Requests (PRs) zahlen auf drei Dinge ein: Lesbarkeit der Historie, Qualität der Reviews und Automatisierbarkeit. Grundlagen dazu kennen Sie aus Modul 02 (Form der Nachricht, Conventional Commits, Trailer) und Modul 04 (Pull-Request-Template, Definition of Done). In diesem Abschnitt geht es darum, sie als *Standards* im Team zu verankern, also verbindlich, dokumentiert und möglichst automatisch geprüft.

Ein Standard für Commit-Nachrichten legt fest: Format der Betreffzeile (zum Beispiel Conventional Commits mit Typ, optionalem Bereich und Beschreibung), maximale Länge, Sprache (Deutsch oder Englisch, aber einheitlich), Pflichtangaben wie Ticketnummer und Regeln für Trailer. Ein Standard für Pull Requests legt fest: Titel nach demselben Schema (besonders wichtig bei Squash-Merge, wo der PR-Titel zur Commit-Nachricht in `main` wird), Pflichtabschnitte der Beschreibung, Größenrichtwert, Labels (zum Beispiel für Art der Änderung oder Risiko) und Zuordnung zu Tickets. Der Wert zeigt sich bei der Automatisierung: Aus einheitlichen Nachrichten lassen sich Changelogs und Versionsnummern ableiten (Modul 07 und 09), und Labels steuern Pipelines oder Release-Notizen.

Die Durchsetzung geschieht in zwei Schichten. Lokal prüft ein `commit-msg`-Hook die Nachricht, bevor der Commit entsteht (schnelles Feedback, aber umgehbar). In der Pipeline prüft ein Job alle Commit-Nachrichten oder den PR-Titel (verbindlich, weil nicht umgehbar). Das folgende Beispiel zeigt einen lokalen Hook für das Format „Typ(Bereich): Beschreibung“ und die Prüfung im Skriptstil, wie sie auch in einer Pipeline liefe. Hooks im Detail folgen in Modul 09.

```bash
git init -b main cm && cd cm
git config user.name "Dev"; git config user.email dev@example.org
cat > .git/hooks/commit-msg <<'EOF'
#!/bin/sh
# Prueft die erste Zeile der Commit-Nachricht
msg=$(head -n 1 "$1")
muster='^(feat|fix|docs|refactor|test|chore)(\([a-z0-9-]+\))?!?: .{3,72}$'
if ! printf '%s' "$msg" | grep -Eq "$muster"; then
  echo "Commit-Nachricht entspricht nicht dem Standard:" >&2
  echo "  erwartet: typ(bereich): beschreibung   (typ: feat|fix|docs|refactor|test|chore)" >&2
  echo "  erhalten: $msg" >&2
  exit 1
fi
EOF
chmod +x .git/hooks/commit-msg

echo "a" > a.txt && git add .
git commit -m "Dinge geaendert"
git commit -m "feat(login): Passwort-Reset per E-Mail"
git commit --allow-empty -m "fix!: Schnittstelle inkompatibel angepasst"

# Dieselbe Pruefung als Pipeline-Schritt ueber einen Commit-Bereich
git commit --allow-empty --no-verify -m "unsauber"
git log --format='%h %s' | grep -Ev '^[0-9a-f]+ (feat|fix|docs|refactor|test|chore)(\([a-z0-9-]+\))?!?: ' \
  && echo "Pipeline: FEHLER, nicht konforme Commits gefunden"
```

Der Hook erhält als erstes Argument den Pfad der Datei mit der Nachricht. Er liest die erste Zeile und vergleicht sie mit einem regulären Ausdruck: Typ aus einer Liste, optionaler Bereich in Klammern, optionales Ausrufezeichen für inkompatible Änderungen, Doppelpunkt, Leerzeichen und eine Beschreibung von drei bis 72 Zeichen. Der erste Commit mit „Dinge geaendert“ wird abgelehnt, die beiden folgenden werden angenommen. Mit `--no-verify` lässt sich der Hook umgehen, weshalb die zweite Prüfung wichtig ist: Der `git log`-Aufruf listet alle Nachrichten und `grep -Ev` behält nur solche, die dem Muster *nicht* entsprechen. Findet er welche, endet der Befehl erfolgreich, und die Nachricht „FEHLER“ erscheint. In einer echten Pipeline würde man das Ergebnis umkehren und den Job fehlschlagen lassen.

### Typische Fehler und Stolperfallen.

Falsch ist ein Standard, der zu streng ist, um gelebt zu werden: Wer die Nachricht für jeden Tippfehler ablehnt und keine Ausnahmen für Merge- und Revert-Commits vorsieht, erntet Umgehungen. Richtig ist, den Standard klein zu halten, die automatische Prüfung auf Format und Länge zu beschränken und Merge-Commits sowie automatisch erzeugte Nachrichten auszunehmen. Ein zweiter Fehler: Nur lokale Hooks einzusetzen. Sie liegen in `.git/hooks`, werden nicht mitversioniert und sind nicht bei allen installiert. Verlässlich ist erst die Prüfung in der Pipeline. Teams verteilen lokale Hooks daher über ein versioniertes Verzeichnis (zum Beispiel mit `core.hooksPath`) oder ein Werkzeug (Modul 09).

### Praxisbezug.

Bei Squash-Merge ist der PR-Titel der wichtigste Text: Er wird zur Commit-Nachricht der Hauptlinie. Viele Teams prüfen deshalb den PR-Titel automatisch gegen das Commit-Schema. Aus dem Schema entstehen dann Release-Notizen und Versionssprünge ohne Handarbeit. Entscheidend ist, dass alle verstehen, warum der Standard existiert; dann wird er auch befolgt.

## 5.9 Governance-Prozess

*Governance* bezeichnet hier die Summe aller Regeln, Verantwortlichkeiten und Kontrollen, mit denen eine Organisation ihre Repositories steuert. Das klingt bürokratisch, löst aber ein handfestes Problem: Ohne Steuerung driften Repositories auseinander. Jedes Team entwickelt eigene Gewohnheiten, Schutzregeln sind unterschiedlich, Standarddateien fehlen, alte Repositories verwaisen und niemand weiß mehr, wer zuständig ist. Ein Governance-Prozess verhindert das auf drei Ebenen. Er legt *Standards* fest (welche Regeln gelten), er regelt *Ausnahmen und Freigaben* (wer darf wann abweichen), und er sorgt für *regelmäßige Überprüfung* (stimmt die Wirklichkeit mit den Regeln überein).

Für das Festhalten von Entscheidungen haben sich *Architecture Decision Records* (ADR) bewährt: kurze Textdateien im Repository, die zu einer Entscheidung Kontext, Entscheidung und Folgen beschreiben. Der Vorteil: Auch Jahre später ist nachvollziehbar, warum man sich etwa für Squash-Merge oder gegen ein Monorepo entschieden hat. Ausnahmen werden in einem *Ausnahmeregister* geführt, mit Begründung, Verantwortlichem und Ablaufdatum; abgelaufene Ausnahmen werden erneuert oder beendet. Die regelmäßigen Überprüfungen (*Repository Reviews*) laufen idealerweise halbautomatisch, mit Skripten, die Abweichungen sammeln, und einem Termin (zum Beispiel vierteljährlich), an dem Verantwortliche Befunde bearbeiten.

Das Beispiel legt ein ADR und ein Ausnahmeregister an und enthält ein Prüfskript für ein Repository: Es meldet Branches ohne Aktivität, fehlende Standarddateien und die Zahl der Commits ohne Signatur. Es zeigt, wie man aus Git-Daten Governance-Kennzahlen gewinnt.

```bash
git init -b main gov && cd gov
git config user.name "Dev"; git config user.email dev@example.org
mkdir -p docs/adr docs/governance
cat > docs/adr/0001-merge-strategie.md <<'EOF'
# ADR 0001: Squash-Merge als Standard
Status: angenommen
Datum: 2026-10-28

## Kontext
Die Historie von main soll pro Pull Request genau einen Commit enthalten.

## Entscheidung
Pull Requests werden per Squash-Merge integriert. Der PR-Titel folgt dem
Commit-Schema (Conventional Commits).

## Folgen
Feine Zwischenschritte gehen verloren; Changelogs lassen sich automatisch erzeugen.
EOF
cat > docs/governance/ausnahmen.md <<'EOF'
# Ausnahmeregister
| Repository | Abweichung | Grund | Verantwortlich | Gueltig bis |
|------------|-----------|-------|----------------|-------------|
| legacy-app | kein Branch-Schutz | Migration laeuft | m.muster | 2027-03-31 |
EOF
echo "# Projekt" > README.md
git add . && git commit -m "docs: ADR und Ausnahmeregister"
git switch -c feature/alt
GIT_AUTHOR_DATE="2026-06-01T12:00:00" GIT_COMMITTER_DATE="2026-06-01T12:00:00" \
  git commit --allow-empty -m "feat: alt"
git switch main

cat > audit.sh <<'EOF'
#!/bin/sh
tage=${1:-30}
echo "== Branches ohne Aktivitaet seit mehr als $tage Tagen =="
git for-each-ref refs/heads --format='%(refname:short) %(committerdate:unix)' |
while read name ts; do
  alter=$(( ( $(date +%s) - ts ) / 86400 ))
  if [ "$alter" -gt "$tage" ]; then echo "  $name ($alter Tage)"; fi
done
echo "== Standarddateien =="
for f in README.md CONTRIBUTING.md SECURITY.md .github/CODEOWNERS; do
  [ -f "$f" ] || echo "  fehlt: $f"
done
echo "== Signaturstatus der letzten Commits (N = nicht signiert) =="
git log -50 --format='%G?' | sort | uniq -c
EOF
sh audit.sh 30
```

Das ADR-Dokument folgt dem üblichen Aufbau Status, Kontext, Entscheidung, Folgen. Das Ausnahmeregister enthält Gültigkeitsdatum und Verantwortlichen; so wird aus einer Ausnahme keine Dauerlösung. Das Skript `audit.sh` liest Daten, die Git ohnehin hat: `git for-each-ref` liefert Branch-Alter, ein Dateitest die Standarddateien, und `git log --format='%G?'` gibt pro Commit den Signaturstatus aus, wobei `N` für „nicht signiert“ steht (Signaturen sind Thema von Modul 08). Im Beispiel wird ein Branch rückdatiert, damit der Audit einen Treffer meldet. Die Ausgabe ist eine Liste, die in einem Review-Termin abgearbeitet wird.

### Typische Fehler und Stolperfallen.

Falsch ist Governance als einmaliges Dokument, das danach niemand mehr liest. Richtig ist ein laufender Prozess mit Zuständigkeit, Termin und Werkzeugen. Ein zweiter Fehler ist Kontrolle ohne Konsequenz: Wer Abweichungen sammelt, aber nie nachfasst, entwertet das Verfahren. Dritter Fehler: Regeln ohne Begründung. Wenn niemand versteht, warum etwas gilt, wird es umgangen. Das ADR-Format hilft, weil es das „Warum“ festhält. Und viertens: Zu zentrale Steuerung, die Teams ausbremst. Gute Governance gibt Leitplanken vor und lässt innerhalb Freiheit.

### Praxisbezug.

In größeren Unternehmen liegt Governance oft bei einem Plattformteam, das Vorlagen, Richtlinien als Code und Auswertungen bereitstellt; die Teams sehen ihre Abweichungen in einem Dashboard. In kleineren Organisationen genügt ein halbjährlicher Termin mit Checkliste. Wichtig ist, dass die Ergebnisse für Audits nachweisbar sind: Wer hat wann was geprüft und entschieden? Das Thema vertieft Modul 11 mit Enterprise-Plattformen und Modul 12 mit der Governance-Roadmap.

## Übungen zu Modul 05

**Aufgabe 1.** Erklären Sie in eigenen Worten, wie Trunk-Based Development lange Funktionen ermöglicht, ohne dass Halbfertiges bei Nutzern landet. Welche Voraussetzungen braucht das Verfahren?

**Aufgabe 2.** Nennen Sie die Branch-Arten von GitFlow und beschreiben Sie, wohin ein Hotfix zurückgeführt werden muss. Welche Folge hat es, wenn man einen der beiden Wege vergisst?

**Aufgabe 3.** Was bedeutet „upstream first“ bei der Pflege von Release-Branches, und mit welchem Befehl findet man offene Backports?

**Aufgabe 4.** Nennen Sie fünf Regeln, die man auf dem Hauptbranch typischerweise schützt, und erklären Sie, warum Regeln serverseitig wirken müssen.

**Aufgabe 5.** Praktisch: Bauen Sie ein Repository mit einem Feature-Flag nach (Schritte: Flag aus, Funktion in kleinen Commits einbauen, Flag an). Schreiben Sie eine Abfrage, die Branches findet, die älter als zwei Tage sind.

**Aufgabe 6.** Praktisch: Spielen Sie einen GitFlow-Zyklus mit Feature, Release und Hotfix durch und zeigen Sie mit `git log --graph --all`, dass der Hotfix in `main` und `develop` angekommen ist.

**Aufgabe 7.** Praktisch: Richten Sie in einem Bare-Repository einen `update`-Hook ein, der das Verschieben bestehender Tags verbietet, und weisen Sie nach, dass `git push --force` auf einen bestehenden Tag abgelehnt wird.

**Aufgabe 8.** Praktisch (Transfer): Erstellen Sie ein kleines Monorepo mit zwei Diensten. Schreiben Sie ein Skript, das aus dem Diff der letzten Änderung ermittelt, welche Dienste neu gebaut werden müssen, und testen Sie Sparse Checkout im Cone-Modus. Legen Sie dazu einen `commit-msg`-Hook an, der Conventional Commits erzwingt.

## Lösungen zu Modul 05

**Lösung 1.** Unfertiger Code wird in kleinen Schritten in `main` gemergt, bleibt aber durch einen Feature-Flag deaktiviert; Deployment und Release sind entkoppelt. Der Flag wird erst nach Fertigstellung eingeschaltet und danach samt totem Code entfernt. Voraussetzungen: zuverlässige automatische Tests und Pipeline bei jedem Merge, kleine Änderungen, schnelle Reviews, kurzlebige Branches.

**Lösung 2.** `main`, `develop` (langlebig); `feature/*`, `release/*`, `hotfix/*` (kurzlebig). Der Hotfix geht in `main` (mit Tag) und in `develop` (oder in einen aktuell existierenden Release-Branch). Fehlt der Merge nach `develop`, taucht der Fehler im nächsten Release wieder auf; fehlt der Merge nach `main`, ist die Produktion nicht korrigiert.

**Lösung 3.** Korrekturen entstehen zuerst auf `main` und werden von dort per `git cherry-pick -x` in die Release-Branches übertragen, damit kein Fehler nur in alten Versionen behoben wird. Offene Backports zeigt `git cherry -v <release> main`: `+` fehlt noch, `-` ist (inhaltlich) schon enthalten.

**Lösung 4.** Beispiele: kein direkter Push, Pull Request mit Freigabe, bestandene Statusprüfungen, kein Force-Push, kein Löschen, lineare Historie, signierte Commits, Code-Owner-Review. Serverseitig müssen sie wirken, weil lokale Hooks und Einstellungen jede Person umgehen oder auslassen kann; nur der Server sieht jeden Push.

**Lösung 5.** Aufbau wie in Abschnitt 5.1. Die Abfrage nutzt `git for-each-ref refs/heads --format='%(refname:short) %(committerdate:unix)'` und vergleicht mit `date +%s`; die Schwelle ist `2*86400` Sekunden.

**Lösung 6.** Ablauf wie in Abschnitt 5.2. Im Graphen müssen `v1.0.1`, der Hotfix-Merge in `main` und der Hotfix-Merge in `develop` sichtbar sein.

**Lösung 7.** Hook-Datei `hooks/update`: bei Referenzen unter `refs/tags/` und altem Hash ungleich Nullen eine Fehlermeldung ausgeben und mit 1 beenden. Der erste Push eines neuen Tags gelingt, ein zweiter Push desselben Tags mit `--force` meldet `remote rejected`.

**Lösung 8.** Pfadermittlung: `git diff --name-only HEAD~1 HEAD | cut -d/ -f1-2 | sort -u`. Sparse Checkout: `git sparse-checkout init --cone` und `set dienste/api`. Der Hook entspricht dem Beispiel in Abschnitt 5.8; ein Test mit einer nicht konformen Nachricht muss abgelehnt werden.
