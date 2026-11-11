# Modul 06 - Historie, Rewriting und Recovery

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (6.1 bis 6.9), danach folgen „Übungen zu Modul 06“ (8 Aufgaben) und „Lösungen zu Modul 06“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 12 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 12 h, die Summe der Lerneinheiten ergibt 9 h. Dieses Skript folgt dem höheren Wert und führt deshalb die Wiederherstellungsszenarien ausführlicher aus.

**Lernziele des Moduls:** Log-Analyse, Rebase, Reset, Reflog und Wiederherstellung sicher nutzen; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Bisher haben wir Historie vor allem erzeugt. Dieses Modul zeigt, wie man sie liest, wie man Fehler darin findet, wie man sie bei Bedarf umschreibt und, vor allem, wie man sich von Fehlbedienungen erholt. Das Umschreiben von Historie ist mächtig und gefährlich; die Sicherheitsnetze, die Git dafür bereithält, sind deshalb ein Schwerpunkt.

| **Nr.** | **Lerneinheit**      | **Dauer (h)** | **Prio** |
|:--------|:---------------------|--------------:|:--------:|
| 6.1     | git log und Historie |           1,0 |    1     |
| 6.2     | git blame            |           1,0 |    1     |
| 6.3     | git bisect           |           1,0 |    1     |
| 6.4     | Reset und Restore    |           1,0 |    1     |
| 6.5     | Revert               |           1,0 |    1     |
| 6.6     | Rebase interaktiv    |           1,0 |    1     |
| 6.7     | Reflog               |           1,0 |    2     |
| 6.8     | Stash                |           1,0 |    2     |
| 6.9     | Recovery-Praxis      |           1,0 |    2     |
|         | **Summe**            |       **9,0** |          |

## 6.1 git log und Historie

Die Historie eines Repositories ist ein reichhaltiges Archiv, aber nur, wenn man sie gezielt befragen kann. `git log` ohne Optionen listet alle Commits ab dem aktuellen rückwärts, was bei großen Projekten kaum nutzbar ist. Die Kunst liegt im Filtern und Formatieren. Zur Auswahl von Commits gibt es vier Gruppen von Optionen. Nach *Zeit und Person* filtern `--since`, `--until` und `--author`. Nach *Nachrichtentext* filtert `--grep`. Nach *Pfad* filtert man mit `-- <pfad>` am Ende des Aufrufs; die Trennung durch `--` verhindert Verwechslungen mit Branch-Namen. Nach *Inhalt der Änderung* suchen `-S` (*Pickaxe*) und `-G`: `git log -S"text"` findet Commits, die die Anzahl der Vorkommen von „text“ in der Datei verändern, also solche, die es einführen oder entfernen. `-G"regex"` findet Commits, in deren Diff eine Zeile zu einem regulären Ausdruck passt. Damit beantwortet man Fragen wie „Wann wurde diese Konstante entfernt?“.

Zur *Darstellung* dienen `--oneline` für kompakte Listen, `--graph` für die Verzweigungsstruktur, `--decorate` für Branch- und Tag-Namen, `-p` für das Diff jedes Commits, `--stat` für eine Dateiübersicht und `--format` für eigene Formate (etwa `%h` Kurzhash, `%an` Autor, `%ad` Datum, `%s` Betreff). Bei Historien mit vielen Merges hilft `--first-parent`, das nur dem ersten Elternteil jedes Merge-Commits folgt und so die Hauptlinie zeigt, ohne die Commits der gemergten Feature-Branches. `--no-merges` blendet Merge-Commits aus. Für eine Datei, die umbenannt wurde, verfolgt `--follow` die Historie über den alten Namen hinweg. Und `git log -L :funktion:datei` zeigt die Entwicklung einer einzelnen Funktion oder eines Zeilenbereichs.

Um Commits anzusprechen, nutzt man die *Revisionssyntax*. `HEAD~2` ist der Großelterncommit, `HEAD^` der erste Elterncommit, `HEAD^2` der zweite Elterncommit eines Merges. `main..feature` bezeichnet die Commits, die in `feature` liegen, aber nicht in `main`; `main...feature` (drei Punkte) die symmetrische Differenz. `@{yesterday}` und `@{3}` greifen über das Reflog auf frühere Zustände zu (Abschnitt 6.7). `:/text` findet den jüngsten Commit, dessen Nachricht den Text enthält.

Das Beispiel baut eine kleine Historie und stellt gezielt Fragen an sie.

```bash
git init -b main hist && cd hist
git config user.name "Anna"; git config user.email anna@example.org
echo 'TIMEOUT=30' > config.txt && git add . && git commit -q -m "feat: Konfiguration angelegt"
echo 'name=app' >> config.txt && git commit -q -am "feat: Name ergaenzt"
git config user.name "Ben"; git config user.email ben@example.org
echo 'TIMEOUT=60' > config.txt && echo 'name=app' >> config.txt
git commit -q -am "fix: Timeout auf 60 erhoeht (Ticket 42)"

git switch -q -c feature/x
echo "x" > x.txt && git add . && git commit -q -m "feat: x"
git switch -q main
git merge -q --no-ff feature/x -m "Merge feature/x"
git mv x.txt y.txt && git commit -q -m "refactor: x in y umbenannt"

git log --oneline --graph --decorate
git log --first-parent --oneline
git log --author=Ben --oneline
git log --grep="Ticket 42" --oneline
git log -S"TIMEOUT=60" --oneline -- config.txt
git log --follow --oneline -- y.txt
git log --format='%h %an %ad %s' --date=short -3
git diff HEAD~3..HEAD --stat
git log :/Name --oneline -1
git shortlog -sn HEAD
```

Die ersten beiden Aufrufe zeigen den Unterschied zwischen vollem Graph und der Hauptlinie mit `--first-parent`: Dort erscheint der Merge-Commit, nicht aber der darin gemergte Commit „feat: x“. `--author` und `--grep` filtern nach Person und Text. Besonders aufschlussreich ist `-S"TIMEOUT=60"`: Es liefert genau den Commit, der diesen Wert eingeführt hat. `--follow` zeigt auch die Commits aus der Zeit, als die Datei noch `x.txt` hieß. Mit `--format` und `--date=short` baut man ein eigenes, kompaktes Format, und `git shortlog -sn HEAD` fasst die Anzahl der Commits je Person zusammen (die Angabe von `HEAD` verhindert, dass der Befehl in Skripten ohne Terminal auf Eingaben wartet).

### Typische Fehler und Stolperfallen.

Ein häufiger Fehler ist, auf dem Standardlog zu bestehen und dann bei großen Repositories im Pager zu versinken. Falsch: `git log` scrollen, bis man etwas findet. Richtig: erst eingrenzen (Pfad, Zeitraum, Text, `-S`). Ein zweiter Fehler betrifft `-S` und `-G`: Sie werden verwechselt. `-S` zählt Vorkommen, findet also Einführung und Entfernung, aber keine Änderung innerhalb einer Zeile, die das Vorkommen gleich lässt; für Zeilen-Muster passt `-G`. Drittens vergisst man bei Pfadfiltern den Doppelstrich `--`. Ohne ihn kann Git einen Pfad mit einem Branch-Namen verwechseln und meldet mehrdeutige Argumente.

### Praxisbezug.

Die Log-Abfrage gehört zum Handwerkszeug bei Fehlersuche, Audit und Release-Vorbereitung. Typische Fragen: Was ist seit dem letzten Release (`git log v1.2..main`) passiert? Wer hat diese Datei zuletzt angefasst? Wann kam diese Funktion hinein? Auf Plattformen entsprechen die „History“ und „Compare“-Ansichten genau diesen Befehlen. Viele Teams legen sich Aliase für ihre bevorzugten Log-Formate an (Modul 01).

## 6.2 git blame

`git blame` annotiert jede Zeile einer Datei mit dem Commit, der sie zuletzt geändert hat, samt Autor und Datum. Die Git-Dokumentation beschreibt die Ausgabe als abgekürzten Commit-Namen, Autor und Zeilennummer vor dem Zeileninhalt (abgerufen am 28.10.2026). Der Name stammt aus dem Englischen „to blame“ (beschuldigen), ist aber irreführend. Sinnvoll eingesetzt dient blame nicht dazu, Schuldige zu finden, sondern dazu, den Kontext einer Zeile zu verstehen: Aus welchem Anlass ist sie entstanden, welcher Commit enthält die Begründung, wen kann man fragen? Wer blame als Schuldzuweisung missbraucht, schadet der Fehlerkultur des Teams. Ein guter Umgang ist, blame als Einstieg in die Commit-Nachricht und das zugehörige Ticket zu nutzen.

Die wichtigsten Optionen: `-L 40,60` beschränkt auf einen Zeilenbereich (auch `-L 40,+21` oder ein Funktionsname `-L :funktion`). `-w` ignoriert Leerzeichenänderungen, sodass reine Umformatierungen nicht als Änderung zählen. `-M` erkennt innerhalb einer Datei verschobene Zeilenblöcke, `-C` auch solche, die aus anderen Dateien kopiert wurden; wiederholt angegeben (`-C -C`, `-C -C -C`) sucht Git in immer mehr Commits. Ohne diese Optionen schreibt blame verschobenen Code dem Commit zu, der ihn verschoben hat, nicht dem, der ihn geschrieben hat.

Ein häufiges Problem sind Commits, die ganze Dateien umformatieren (etwa ein Code-Formatter, der den Stil vereinheitlicht): Danach zeigt blame für fast jede Zeile diesen einen Commit, was den Nutzen zerstört. Dafür gibt es `--ignore-rev <hash>`, das den Commit behandelt, als hätte er nie stattgefunden. Die Dokumentation beschreibt, dass die betroffenen Zeilen dann dem vorherigen Commit zugeschrieben werden, der sie oder benachbarte Zeilen angefasst hat. Viele Hosting-Plattformen und IDEs lesen zusätzlich eine Datei `.git-blame-ignore-revs`; für Git selbst richtet man sie mit `git config blame.ignoreRevsFile .git-blame-ignore-revs` ein. Die Datei enthält je Zeile einen vollständigen Commit-Hash, Kommentare beginnen mit `#` (abgerufen am 28.10.2026). Die Datei formatiert nichts, sie nennt Git nur die Commits, die bei der Zuordnung übergangen werden sollen.

Das Beispiel zeigt eine Datei mit einem Formatierungs-Commit und den Unterschied, den `ignore-revs` macht.

```bash
git init -b main bl && cd bl
git config user.name "Anna"; git config user.email anna@example.org
cat > rechne.py <<'EOF'
def netto(brutto):
    return brutto / 1.2
def steuer(brutto):
    return brutto - netto(brutto)
EOF
git add . && git commit -q -m "feat: Netto und Steuer berechnen"

git config user.name "Ben"; git config user.email ben@example.org
# Formatierung: Einrueckung von 4 auf 2 Leerzeichen
sed -i 's/^    /  /' rechne.py
git commit -q -am "style: Code formatiert"
FORMAT=$(git rev-parse HEAD)

git config user.name "Clara"; git config user.email clara@example.org
sed -i 's|1.2|1.19|' rechne.py
git commit -q -am "fix: Steuersatz korrigiert"

git blame rechne.py
git blame -w rechne.py

echo "$FORMAT" > .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs
git blame rechne.py
git blame -L '/def steuer/,+2' rechne.py
```

Der erste Aufruf schreibt die Rückgabezeile von `steuer` dem Formatierungs-Commit von Ben und die Rückgabezeile von `netto` Claras Commit zu, obwohl Anna die Logik geschrieben hat; nur die beiden `def`-Zeilen verweisen noch auf Annas Commit (Git kennzeichnet ihn mit einem vorangestellten `^`, weil es der Anfangscommit ist). Mit `-w` erkennt Git, dass an der Rückgabezeile von `steuer` nur Leerraum geändert wurde, und führt sie auf Annas ursprünglichen Commit zurück. Die Datei `.git-blame-ignore-revs` mit dem Hash des Formatierungs-Commits erreicht dasselbe dauerhaft und hat den Vorteil, dass sie auch dann hilft, wenn ein Formatierer mehr als Leerraum geändert hat, etwa Zeilen umbricht. Die Zeile mit dem Steuersatz bleibt Claras Commit zugeordnet, weil sie dort tatsächlich inhaltlich geändert wurde. Der letzte Aufruf zeigt `-L` mit einem Regex-Anfang und einer Zeilenzahl. Wichtig bleibt: Blame zeigt nur, *wer die Zeile zuletzt geändert hat*; der Ursprung kann weiter zurückliegen.

### Typische Fehler und Stolperfallen.

Falsch ist es, auf Grundlage eines blame-Treffers jemanden zur Rede zu stellen. Die letzte Änderung an einer Zeile ist oft ein Refactoring, eine Formatierung oder ein Merge, nicht die inhaltliche Ursache. Richtig ist, den Commit-Hash zu öffnen, die Nachricht zu lesen und mit `git log -S` oder `git blame <hash>^ -- datei` weiterzugraben. Ein zweiter Fehler ist, nach einem Massen-Formatierungs-Commit nicht dafür zu sorgen, dass blame brauchbar bleibt: Die Datei `.git-blame-ignore-revs` gehört gleich im Anschluss in das Repository. Drittens: Wer `git blame` auf eine umbenannte Datei anwendet, folgt der Historie automatisch, aber nur innerhalb des angegebenen Pfades; bei Zweifeln hilft `git log --follow`.

### Praxisbezug.

In der Fehlersuche ist blame meist der erste Schritt: Eine verdächtige Zeile führt zum Commit, der Commit zum Pull Request und das Ticket zur Anforderung. Auf Plattformen findet man die Funktion als „Blame“-Ansicht einer Datei mit einem Link auf den ursprünglichen Pull Request. Viele Teams führen die Ignore-Datei nach der Einführung eines Formatters als Standardschritt aus.

## 6.3 git bisect

Wenn ein Fehler auftritt, den es früher nicht gab, ist die wichtigste Frage: Welche Änderung hat ihn eingeführt? Bei hunderten Commits dazwischen wäre es mühsam, jeden einzeln zu prüfen. `git bisect` macht daraus eine *binäre Suche*. Man nennt einen Commit, in dem der Fehler sicher vorhanden ist (*bad*), und einen, in dem er sicher fehlt (*good*). Git checkt einen Commit in der Mitte aus, man prüft, ob der Fehler dort auftritt, und meldet das Ergebnis. Nach jedem Schritt halbiert sich der Suchraum; bei 1000 Commits genügen etwa zehn Schritte. Am Ende nennt Git den ersten fehlerhaften Commit.

Der Ablauf nach der Dokumentation (abgerufen am 28.10.2026): `git bisect start`, dann `git bisect bad` für den aktuellen Stand und `git bisect good <commit>` für einen funktionierenden. Git wählt einen Commit in der Mitte; nach Test ruft man `git bisect good` oder `git bisect bad`. Lässt sich ein Commit nicht testen, etwa weil er nicht baut, überspringt man ihn mit `git bisect skip`. Am Ende führt `git bisect reset` zurück zum ursprünglichen Stand. Für Änderungen, die keine Fehler sind (etwa „ab wann ist es schneller?“), gibt es die neutralen Begriffe `old` und `new`.

Am wertvollsten ist die Automatisierung mit `git bisect run <skript>`: Git führt das Skript bei jedem Schritt aus und wertet den Rückgabewert aus. Laut Dokumentation bedeutet 0 „gut“, 1 bis 127 außer 125 „schlecht“, 125 „nicht testbar, überspringen“ und jeder andere Wert bricht die Suche ab. Ein gutes Testskript baut das Projekt (und meldet 125, wenn der Build nicht klappt) und prüft das konkrete Fehlerbild.

Das Beispiel erzeugt eine Historie mit zehn Commits, von denen einer einen Fehler einführt, und findet ihn automatisch.

```bash
git init -b main bs && cd bs
git config user.name "Dev"; git config user.email dev@example.org
echo 'rechne() { echo $(( $1 + $2 )); }' > lib.sh
git add . && git commit -q -m "Commit 1: rechne() addiert"
for i in 2 3 4 5; do
  echo "# Aenderung $i" >> lib.sh
  git commit -q -am "Commit $i: Kommentar"
done
# Commit 6 fuehrt den Fehler ein
sed -i 's/\$1 + \$2/$1 - $2/' lib.sh
git commit -q -am "Commit 6: Refactoring (fehlerhaft)"
for i in 7 8 9 10; do
  echo "# Aenderung $i" >> lib.sh
  git commit -q -am "Commit $i: Kommentar"
done

cat > test.sh <<'EOF'
#!/bin/sh
. ./lib.sh
[ "$(rechne 2 3)" = "5" ]
EOF
sh test.sh && echo "HEAD ok" || echo "HEAD fehlerhaft"

git bisect start HEAD HEAD~9
git bisect run sh test.sh
git bisect log | tail -5
git bisect reset
```

Die Funktion `rechne` addiert zunächst korrekt; Commit 6 ändert das Plus in ein Minus. Das Testskript lädt die Bibliothek und prüft, ob `rechne 2 3` den Wert 5 liefert; der Rückgabewert des Tests ist 0 bei Erfolg und 1 bei Misserfolg. `git bisect start HEAD HEAD~9` gibt gleich beide Endpunkte an: Der aktuelle Stand ist schlecht, `HEAD~9` (Commit 1) gut. `git bisect run` führt die Suche selbstständig durch und endet mit der Meldung „is the first bad commit“ und dem Hash und Betreff von Commit 6. Das Protokoll mit `git bisect log` zeigt die einzelnen Schritte, und `git bisect reset` stellt den Ausgangszustand her. Das Skript `test.sh` liegt im Arbeitsverzeichnis, ist aber nicht im Repository versioniert, sondern bleibt beim Wechsel von Commits unverändert; deshalb funktioniert es über alle Stände hinweg.

### Typische Fehler und Stolperfallen.

Falsch ist es, mit einem Testverfahren zu bisecten, das nicht deterministisch ist. Wenn ein Test manchmal besteht und manchmal nicht (Zeitabhängigkeit, Zufall), führt die Suche zu einem falschen Ergebnis. Richtig ist, den Test erst mehrfach auf einem bekannten schlechten Stand zu wiederholen und ihn robust zu machen. Zweiter Fehler: Den Test im Repository ablegen und ihn dann vom Wechsel der Commits überschreiben oder löschen lassen. Das Skript muss außerhalb des Repositories oder untracked liegen. Dritter Fehler: Commits, die nicht bauen, als „schlecht“ zu werten; sie gehören mit Rückgabewert 125 übersprungen. Viertens: Zu vergessen, `git bisect reset` auszuführen, und anschließend auf einem losgelösten HEAD weiterzuarbeiten.

### Praxisbezug.

Bisect lohnt sich vor allem bei Regressionen in einer großen Codebasis. Teams kombinieren es mit einer reproduzierbaren Fehlerbeschreibung als Test: Aus dem Fehlerticket wird ein Skript, das Bisect nutzt, und das Ergebnis geht an die verantwortliche Person. Voraussetzung sind kleine, einzeln lauffähige Commits, was ein weiterer Grund für die Disziplin aus Modul 02 ist. Squash-Merges verringern die Auflösung, weil ein Commit dann viele Änderungen enthält (Modul 03).

## 6.4 Reset und Restore

Mit `git reset` und `git restore` nimmt man Zustände zurück. Beide Befehle werden oft verwechselt, weil sie ähnlich klingen, aber an verschiedenen Stellen ansetzen. Zur Orientierung hilft das Bild aus Modul 02 mit drei Bereichen: Commit-Historie (HEAD), Staging Area (Index) und Arbeitsverzeichnis.

`git reset` setzt den *aktuellen Branch* auf einen anderen Commit zurück und verändert je nach Modus auch Index und Arbeitsverzeichnis. Die Dokumentation (abgerufen am 28.10.2026) unterscheidet drei Hauptmodi. Bei `--soft` wird nur der Branch-Zeiger bewegt; Index und Arbeitsverzeichnis bleiben unverändert, die Änderungen der entfernten Commits stehen daher als vorgemerkt bereit. Das ist nützlich, um mehrere Commits zu einem zusammenzufassen (`git reset --soft HEAD~3` gefolgt von einem neuen Commit). Bei `--mixed`, dem Standardmodus, wird zusätzlich der Index zurückgesetzt, das Arbeitsverzeichnis bleibt unverändert: Die Änderungen sind noch da, aber nicht vorgemerkt. Bei `--hard` wird auch das Arbeitsverzeichnis überschrieben; nicht committete Änderungen gehen dabei verloren. Dieser Modus ist destruktiv und verdient entsprechende Vorsicht. Vor der Operation merkt sich Git den bisherigen Stand in `ORIG_HEAD`; `git reset --hard ORIG_HEAD` macht den Reset daher oft rückgängig. Daneben gibt es `--merge` (zum Zurücksetzen nach einem fehlgeschlagenen Merge) und `--keep` (zum Entfernen von Commits bei Beibehaltung lokaler Änderungen; bricht ab, wenn es dabei lokale Änderungen überschreiben müsste).

| **Modus**           | **Branch / HEAD** | **Index**     | **Arbeitsverzeichnis**          |
|:--------------------|:------------------|:--------------|:--------------------------------|
| `--soft`            | bewegt            | unverändert   | unverändert                     |
| `--mixed` (Standard) | bewegt            | zurückgesetzt | unverändert                     |
| `--hard`            | bewegt            | zurückgesetzt | überschrieben (Verlust möglich) |

`git restore` dagegen verändert *keinen Branch*, sondern stellt Dateien wieder her. Ohne Option holt `git restore <datei>` die Datei aus dem Index ins Arbeitsverzeichnis und verwirft lokale Änderungen. Mit `--staged` nimmt man eine Datei aus der Staging Area zurück (die Änderung bleibt im Arbeitsverzeichnis). Mit `--source=<commit>` holt man die Fassung der Datei aus einem beliebigen Commit. Die Aufteilung in `switch` und `restore` entlastet `git checkout`, das früher beide Aufgaben vermischte.

Im folgenden Beispiel durchlaufen wir alle drei Reset-Modi nacheinander und stellen danach mit `restore` Dateien her. Zwischendurch zeigt `git status`, was sich jeweils verändert hat.

```bash
git init -b main rs && cd rs
git config user.name "Dev"; git config user.email dev@example.org
for i in 1 2 3; do echo "Stand $i" > datei.txt; git add .; git commit -q -m "Commit $i"; done

# --soft: Commits 2 und 3 zusammenfassen
git reset --soft HEAD~2
git status -s
git commit -q -m "Commit 2 und 3 zusammengefasst"
git log --oneline

# --mixed: Commit zurueck, Aenderung bleibt unvorgemerkt
git reset HEAD~1
git status -s
cat datei.txt

# --hard: Aenderung endgueltig verwerfen
git reset --hard
git status -s
cat datei.txt

# Rueckgaengig machen mit ORIG_HEAD
echo "Notiz" > notiz.txt && git add . && git commit -q -m "Notiz"
git reset --hard HEAD~1
ls
git reset --hard ORIG_HEAD
ls

# restore
echo "kaputt" > datei.txt
git restore datei.txt
cat datei.txt
echo "neu" > extra.txt && git add extra.txt
git restore --staged extra.txt
git status -s
git restore --source=HEAD~1 notiz.txt
ls
```

Der `--soft`-Reset lässt `datei.txt` mit dem Stand 3 im Index; der neue Commit fasst beide Änderungen zusammen. Der gemischte Reset (Standard) bewegt den Branch zurück, der Dateiinhalt „Stand 3“ bleibt im Arbeitsverzeichnis stehen, erscheint im Status aber als unvorgemerkt (`M` in der zweiten Spalte). Der harte Reset macht daraus wieder den Stand des Commits. Der folgende Abschnitt ist die Sicherheitsübung: Mit `git reset --hard HEAD~1` geht der Commit „Notiz“ verloren, `git reset --hard ORIG_HEAD` bringt ihn zurück, solange man zwischendurch keine weitere Operation ausgeführt hat, die `ORIG_HEAD` überschreibt. Die letzten Zeilen zeigen `restore`. Der letzte Befehl holt `notiz.txt` aus einem Commit, in dem die Datei noch nicht existierte. Das erzeugt keine Fehlermeldung: `git restore --source` macht den Pfad zum Spiegel des Quell-Commits und *entfernt* die Datei deshalb aus dem Arbeitsverzeichnis. Auch das sollte man vor dem Aufruf wissen.

### Typische Fehler und Stolperfallen.

Falsch ist `git reset --hard` als „Aufräumen“ ohne vorherigen Blick auf `git status`. Alle nicht committeten Änderungen an verfolgten Dateien sind danach weg und lassen sich durch nichts wiederherstellen, auch nicht über das Reflog, denn sie waren nie in einem Commit. Richtig ist, vorher zu committen oder mit `git stash` zu sichern. Der zweite große Fehler betrifft geteilte Historie: Ein Reset auf einen älteren Commit, gefolgt von `git push --force`, entfernt Commits, die andere bereits übernommen haben. Die Dokumentation von `git rebase` beschreibt dieselbe Problematik unter „Recovering from upstream rebase“: Wer geteilte Historie umschreibt, zwingt alle anderen zu aufwendigen Reparaturen. Richtig ist, auf gemeinsamen Branches mit `git revert` zu arbeiten (nächster Abschnitt). Dritter Fehler: `git reset <datei>` und `git reset --hard <datei>` zu verwechseln. Mit Pfad angegeben ist `git reset` nur das alte Gegenstück zu `restore --staged`; `--hard` mit Pfad ist nicht erlaubt.

### Praxisbezug.

In der täglichen Arbeit nutzt man `reset --soft` und `--mixed` zum Aufräumen eigener lokaler Commits vor einem Pull Request und `restore` zum Zurücknehmen einzelner Dateien. `reset --hard` bleibt Ausnahmesituationen vorbehalten, etwa um nach einem fehlgeschlagenen Experiment auf den letzten Commit zurückzugehen. Wer unsicher ist, legt vorher einen Sicherungs-Branch an: `git branch backup`. Er kostet nichts und rettet im Zweifel alles.

## 6.5 Revert

Während `git reset` Commits aus der Historie entfernt, macht `git revert` einen Commit *durch einen neuen Commit rückgängig*. Es berechnet die Änderungen des angegebenen Commits, kehrt sie um und legt einen neuen Commit an. Die Historie wird nicht umgeschrieben, sondern fortgeschrieben (Dokumentation zu `git revert`, abgerufen am 28.10.2026). Das macht Revert zum richtigen Werkzeug für alles, was bereits veröffentlicht ist: Die anderen Teammitglieder können den Revert wie jeden anderen Commit pullen, ohne dass ihre Historie bricht. Außerdem ist er *auditierbar*: Die Historie zeigt, dass eine Änderung gemacht und später zurückgenommen wurde, samt Begründung.

Eine gute Praxis, die wir hier empfehlen, ist es, in der Nachricht zu erklären, *warum* der Commit zurückgenommen wird; die Standardnachricht „Revert ...“ nennt nur das Was. Mit `--no-commit` (`-n`) wendet man die Umkehrung nur im Arbeitsverzeichnis an, was nützlich ist, um mehrere Reverts zu einem Commit zu bündeln. Ein Revert kann Konflikte erzeugen, wenn spätere Commits dieselben Zeilen verändert haben; dann löst man sie wie bei einem Merge und fährt mit `git revert --continue` fort.

Eine Besonderheit sind Merge-Commits. Ein Merge-Commit hat zwei Eltern, und Git kann nicht wissen, welcher davon die Hauptlinie ist. Mit `git revert -m 1 <merge>` wählt man den ersten Elternteil (die Hauptlinie, auf der gemergt wurde) als Referenz. Wichtig ist die Folge, die im Git-Howto „Revert a faulty merge“ beschrieben wird (weiterführend, nicht abgerufen): Das Revert eines Merges erklärt, dass man die Änderungen des gemergten Branches nie haben will. Wird derselbe Branch später erneut gemergt, bringt Git nur Änderungen mit, die nicht schon im zurückgenommenen Merge enthalten waren. Um die Funktion zurückzubekommen, muss man zuerst das Revert selbst zurücknehmen (ein „Revert des Reverts“) und dann erst neu mergen.

Das Beispiel zeigt einen einfachen Revert, den Revert eines Merges und die Falle beim erneuten Merge.

```bash
git init -b main rv && cd rv
git config user.name "Dev"; git config user.email dev@example.org
echo "A" > a.txt && git add . && git commit -q -m "A"

# 1) Einfacher Revert
echo "Fehler" >> a.txt && git commit -q -am "Fehlerhafte Aenderung"
git revert --no-edit HEAD
git log --oneline
cat a.txt

# 2) Revert eines Merge-Commits
git switch -q -c feature/f
echo "F1" > f.txt && git add . && git commit -q -m "F1"
echo "F2" >> f.txt && git commit -q -am "F2"
git switch -q main
git merge -q --no-ff feature/f -m "Merge feature/f"
MERGE=$(git rev-parse HEAD)
git revert -m 1 --no-edit $MERGE
ls

# 3) Die Falle: erneut mergen bringt nichts
git merge feature/f
ls

# Loesung: Revert des Reverts, dann ist die Funktion wieder da
git revert --no-edit HEAD
ls
git log --oneline --graph
```

Der erste Revert legt einen Commit „Revert …“ an; `a.txt` ist wieder bei „A“, die Historie zeigt beide Commits. Beim Merge-Revert wird `f.txt` wieder aus dem Arbeitsverzeichnis entfernt. Versucht man dann, `feature/f` erneut zu mergen, antwortet Git mit „Already up to date“: Alle Commits des Branches sind bereits Teil der Historie, und die Dateien bleiben verschwunden. Erst das Zurücknehmen des Reverts mit einem weiteren `git revert` bringt `f.txt` wieder zurück. Wer nach dem Revert weitere Änderungen am Feature-Branch vornimmt, bekommt später nur diese neuen Commits beim Merge, aber nicht die ursprünglichen. Auch dann ist das Zurücknehmen des Reverts die sauberere Lösung. Beachten Sie die Commit-Nachricht des zweiten Reverts: Git erkennt, dass hier ein Revert zurückgenommen wird, und schreibt „Reapply“ statt eines verschachtelten „Revert“.

### Typische Fehler und Stolperfallen.

Falsch ist, einen veröffentlichten fehlerhaften Commit mit `reset` und `--force` zu entfernen. Die Kolleginnen und Kollegen haben ihn schon, und ihre nächsten Pushes bringen ihn zurück oder erzeugen Konflikte. Richtig ist `git revert`. Ein zweiter Fehler: Den Merge-Revert ohne `-m` zu versuchen. Git meldet dann, dass der Commit ein Merge ist und ein Elternteil angegeben werden muss. Der dritte Fehler ist die oben beschriebene Falle: Nach einem Merge-Revert denkt man, man könne den Branch „einfach nochmal“ mergen. Und viertens: Ohne Begründung zu reverten. Eine Nachricht wie „Revert: Zahlungsmodul verursachte Doppelbuchungen (INC-381)“ spart später viel Recherche.

### Praxisbezug.

Im Incident-Management ist der Revert das erste Mittel: Wenn ein Release Probleme macht, nimmt man den verantwortlichen Merge zurück und liefert aus, bevor man in Ruhe nach der Ursache sucht. Viele Plattformen bieten dafür einen „Revert“-Knopf am Pull Request, der den Revert als eigenen Pull Request vorbereitet. Dass dies auch Merge-Commits abdeckt, ist ein Grund für die Entscheidung „Merge-Commit“ oder „Squash“ in Modul 03: Bei Squash genügt ein einfacher Revert.

## 6.6 Rebase interaktiv

Der *interaktive Rebase* (`git rebase -i`) ist das Werkzeug, mit dem man eine Serie von Commits umgestalten kann. Aus Modul 03 kennen Sie den normalen Rebase, der Commits auf eine neue Basis setzt. Der interaktive Modus öffnet zusätzlich einen Editor mit einer *Todo-Liste*: Jede Zeile ist ein Commit mit einem Befehl davor, und Git arbeitet die Liste von oben nach unten ab. Die wichtigsten Befehle sind `pick` (Commit unverändert übernehmen), `reword` (Commit übernehmen, Nachricht ändern), `edit` (anhalten, um den Commit zu ändern oder aufzuteilen), `squash` (mit dem vorherigen Commit verschmelzen und die Nachrichten zusammenführen), `fixup` (wie squash, aber die Nachricht des Fixup-Commits verwerfen) und `drop` (Commit entfernen). Durch Umsortieren der Zeilen ändert man die Reihenfolge der Commits.

Die Dokumentation (abgerufen am 28.10.2026) nennt weitere nützliche Möglichkeiten. Mit `--autosquash` werden Commits, deren Nachricht mit `fixup!`, `squash!` oder `amend!` beginnt, automatisch hinter ihren Zielcommit sortiert und entsprechend markiert; `git commit --fixup=<hash>` erzeugt solche Commits (Modul 04). Mit `--update-refs` werden Branches, die auf umgeschriebene Commits zeigen, automatisch mitgezogen, was bei gestapelten Branches hilft; Branches, die in einem anderen Arbeitsverzeichnis ausgecheckt sind, werden nicht aktualisiert. Mit `--rebase-merges` bleiben Merge-Commits erhalten. Für den Notfall gelten `git rebase --abort` (zurück zum Ausgangszustand) und `git rebase --continue` nach der Lösung von Konflikten.

Die goldene Regel steht wieder im Vordergrund: Schreiben Sie nur Historie um, die noch niemand anderes besitzt. Die Dokumentation zu `git rebase` warnt, dass das Rebasen eines Branches, auf dem andere aufgebaut haben, diese zwingt, ihre Historie von Hand zu reparieren, und dass sich Doppelungen ansammeln können. Auf dem eigenen, noch nicht geteilten Feature-Branch dagegen ist der interaktive Rebase der Weg, aus einer chronologischen „Arbeitshistorie“ eine saubere „Erzählhistorie“ zu machen.

Um das Beispiel ohne Editor ablaufen zu lassen, wird die Todo-Liste mit `GIT_SEQUENCE_EDITOR` automatisch umgeschrieben. Im Alltag öffnet sich stattdessen ein Editor, in dem man die Befehle von Hand ändert.

```bash
git init -b main ri && cd ri
git config user.name "Dev"; git config user.email dev@example.org
echo "Basis" > basis.txt && git add . && git commit -q -m "Basis"

echo "Funktion" > f.txt && git add . && git commit -q -m "feat: Funktion"
echo "debug" > debug.txt && git add . && git commit -q -m "WIP Debug-Ausgabe"
echo "Funktion v2" > f.txt && git commit -q -am "fix tippfehler"
echo "Test" > t.txt && git add . && git commit -q -m "test: Test fuer Funktion"
git log --oneline

# Todo-Liste automatisch aendern:
#   - Debug-Commit entfernen (drop)
#   - Tippfehler-Commit mit dem vorherigen verschmelzen (fixup)
#   - Test-Commit umformulieren (reword)
GIT_SEQUENCE_EDITOR="sed -i -e 's/^pick \(.*\) WIP Debug-Ausgabe/drop \1 WIP Debug-Ausgabe/' \
  -e 's/^pick \(.*\) fix tippfehler/fixup \1 fix tippfehler/' \
  -e 's/^pick \(.*\) test: Test fuer Funktion/reword \1 test: Test fuer Funktion/'" \
GIT_EDITOR="sed -i 's/^test: .*/test: Funktion abgesichert/'" \
git rebase -i HEAD~4

git log --oneline
git show --stat --oneline HEAD~1
git ls-files
```

Ausgangspunkt sind vier Commits: die Funktion, ein Debug-Commit, ein Tippfehler-Fix und ein Test. Der Befehl `git rebase -i HEAD~4` öffnet die Todo-Liste für diese vier Commits. Durch die automatischen Ersetzungen entsteht daraus: `pick` für die Funktion, `drop` für die Debug-Ausgabe, `fixup` für den Tippfehler (er wandert in den ersten Commit, weil der Debug-Commit gestrichen wurde) und `reword` für den Test. Beim `reword` ruft Git den Editor mit der alten Nachricht auf; die Umgebungsvariable `GIT_EDITOR` ersetzt ihn hier durch einen `sed`-Aufruf, der die Nachricht umschreibt. Das Ergebnis im Log sind drei statt vier Commits: „Basis“, „feat: Funktion“ (mit dem korrigierten Inhalt „Funktion v2“) und „test: Funktion abgesichert“; die Datei `debug.txt` ist verschwunden. Alle neuen Commits haben neue Hashes.

### Typische Fehler und Stolperfallen.

Falsch ist, bereits gepushte und von anderen genutzte Commits interaktiv umzuschreiben. Falsch ist auch, bei Konflikten mitten im Rebase aufzugeben, indem man Dateien in `.git` löscht. Richtig ist, in solchen Fällen `git rebase --abort` zu verwenden. Ein weiterer Fehler: Beim `drop` eine ganze Zeile versehentlich zu löschen. Wird beim Bearbeiten der Todo-Liste eine Zeile versehentlich gelöscht, entfernt Git den Commit nach der Standardeinstellung stillschweigend; mit `git config rebase.missingCommitsCheck warn` (oder `error`) lässt sich das absichern. Beim Bearbeiten der Liste ist daher Sorgfalt gefragt. Außerdem: Mit `squash` und `fixup` an erster Stelle der Liste gibt es nichts, womit verschmolzen werden könnte; Git meldet einen Fehler.

### Praxisbezug.

Vor dem Öffnen eines Pull Requests räumen viele Entwickelnde ihren lokalen Branch auf: Tippfehler-Commits verschmelzen, Debug-Commits streichen, Nachrichten verbessern. Das Ergebnis lässt sich leichter prüfen. Wichtig ist die Reihenfolge im Team: Umschreiben *vor* der Veröffentlichung oder zumindest vor dem Review-Beginn. Nach einem Force-Push auf den eigenen Branch verwendet man `--force-with-lease` (Modul 04).

## 6.7 Reflog

Das *Reflog* ist das Sicherheitsnetz von Git. Es ist ein lokales Protokoll darüber, wohin die Spitzen von Branches und `HEAD` im Lauf der Zeit gezeigt haben. Jedes Mal, wenn sich `HEAD` oder ein Branch bewegt, sei es durch Commit, Checkout, Merge, Rebase, Reset oder Amend, schreibt Git einen Eintrag. Laut Dokumentation (abgerufen am 28.10.2026) sind Reflogs lokal und werden nicht mit Klonen oder Remotes geteilt. Das Reflog beantwortet Fragen wie „Wo stand mein Branch vor dem Rebase?“ oder „Welcher Commit war HEAD vor drei Befehlen?“.

Wichtig ist, dass Commits nicht sofort verschwinden, wenn kein Branch mehr auf sie zeigt. Sie bleiben als Objekte in der Datenbank, bis die Speicherbereinigung (`git gc`) sie löscht. Die Reflog-Einträge halten sie bis dahin erreichbar. Die Dokumentation nennt als Standardfristen 90 Tage für Einträge, die von der aktuellen Spitze aus erreichbar sind (`gc.reflogExpire`), und 30 Tage für nicht mehr erreichbare Einträge (`gc.reflogExpireUnreachable`). Das bedeutet praktisch: Nach einer Fehlbedienung hat man mindestens etwa einen Monat Zeit, verlorene Commits zu retten.

Die Syntax `HEAD@{2}` bedeutet „wo HEAD vor zwei Bewegungen stand“; `main@{yesterday}` oder `main@{one.week.ago}` nutzen Zeitangaben. `git reflog` (kurz für `git reflog show HEAD`) listet die Einträge, `git reflog show <branch>` die eines einzelnen Branches. Zur Rettung findet man den Eintrag und legt dort einen Branch an (`git branch rettung <hash>`) oder setzt den aktuellen Branch dorthin zurück. Das Beispiel spielt das mit dem klassischen Unfall durch: einem harten Reset, einem gelöschten Branch und einem fehlgeschlagenen Rebase.

```bash
git init -b main rl && cd rl
git config user.name "Dev"; git config user.email dev@example.org
for i in 1 2 3 4; do echo "v$i" > datei.txt; git add .; git commit -q -m "Commit $i"; done

# Unfall 1: harter Reset verwirft zwei Commits
git reset --hard HEAD~2
git log --oneline
git reflog -5
git reset --hard HEAD@{1}
git log --oneline

# Unfall 2: Branch geloescht, bevor er gemergt war
git switch -q -c feature/wichtig
echo "wichtig" > w.txt && git add . && git commit -q -m "Wichtige Arbeit"
git switch -q main
git branch -D feature/wichtig
git reflog | head -5
git branch feature/wichtig-rettung HEAD@{1}
git log --oneline feature/wichtig-rettung -1

# Unfall 3: Amend ueberschreibt Commit
git commit -q --amend -m "Commit 4 (neu formuliert)"
git reflog -3
git show --oneline -s HEAD@{1}
```

Beim ersten Unfall zeigt `git reflog` als obersten Eintrag den Reset; der Eintrag darunter ist der Stand davor. `git reset --hard HEAD@{1}` setzt den Branch dorthin zurück, die beiden „verlorenen“ Commits sind wieder da. Beim zweiten Unfall wurde ein Branch mit ungemergter Arbeit gelöscht: Git gab beim Löschen den Hash aus, aber auch ohne ihn findet man im Reflog des `HEAD` den Eintrag, der die Arbeit enthielt (der Eintrag `HEAD@{1}` ist der letzte Commit des Branches, der Eintrag darüber, „checkout: moving from feature/wichtig to main“, zeigt, wann man ihn verlassen hat). Die Zahl hinter `@` hängt davon ab, wie viele Bewegungen seither erfolgten; im Beispiel ist es `HEAD@{1}`, im Alltag liest man sie aus der Liste ab. Beim dritten Unfall zeigt das Reflog den Stand vor dem Amend (`HEAD@{1}`). Wichtig: Alle diese Rettungen funktionieren nur für Commits. Nie committete Änderungen sind nicht im Reflog.

### Typische Fehler und Stolperfallen.

Falsch ist die Annahme, das Reflog sei ein Backup: Es ist lokal, es verfällt, und es kennt keine uncommitteten Änderungen. Richtig ist, es als erste Hilfe zu verstehen und bei wichtigen Dingen zu pushen. Ein zweiter Fehler: Nach einem Unfall weiterzuarbeiten, die Speicherbereinigung laufen zu lassen oder gar `git reflog expire --expire=now --all` mit `git gc --prune=now` auszuführen. Dann sind die Objekte wirklich weg. Wer ein Problem bemerkt, hört auf, committet nichts Neues in Richtung des Problems und sucht zuerst im Reflog. Und drittens: Auf einem anderen Rechner oder in einem frischen Klon gibt es das Reflog nicht. Hat ein Kollege den Stand lokal, kann man ihn bitten, den Branch zu pushen.

### Praxisbezug.

Das Reflog ist der Grund, warum erfahrene Entwickelnde keine Angst vor Rebase und Reset haben. Es ist auch das Instrument, das bei Force-Push-Unfällen hilft: Der Remote-Tracking-Branch `origin/main` hat ebenfalls ein Reflog, dessen früherer Stand die verlorenen Commits enthält (Modul 04). Auf dem Server selbst gibt es in der Regel kein Reflog, weshalb viele Plattformen Force-Pushes verbieten oder ein eigenes Aktivitätsprotokoll führen.

## 6.8 Stash

Manchmal ist man mitten in der Arbeit, muss aber dringend den Branch wechseln: ein Fehler in Produktion, eine Rückfrage zu einem anderen Stand. Die Änderungen sind noch nicht commitfertig. Für solche Fälle gibt es `git stash`, einen Ablagestapel für unfertige Arbeit. Der Befehl speichert die vorgemerkten und nicht vorgemerkten Änderungen und setzt das Arbeitsverzeichnis auf den Stand von `HEAD` zurück. Später holt man die Änderungen vom Stapel zurück. Die Dokumentation (abgerufen am 28.10.2026) beschreibt `git stash` ohne Argument als gleichbedeutend mit `git stash push`.

Die wichtigsten Befehle: `git stash push -m "Beschreibung"` legt einen Eintrag mit sprechender Nachricht an; ohne Nachricht erscheinen automatisch Branch und letzter Commit. `-u` (`--include-untracked`) nimmt auch neue, unverfolgte Dateien mit, `-a` zusätzlich ignorierte. `-p` erlaubt, Hunks einzeln auszuwählen, und `-- <pfad>` begrenzt auf bestimmte Dateien. `git stash list` zeigt alle Einträge (`stash@{0}` ist der jüngste). `git stash show -p` zeigt den Inhalt als Diff. `git stash apply` holt einen Eintrag zurück und behält ihn auf dem Stapel, `git stash pop` holt ihn zurück und entfernt ihn. `git stash drop` entfernt einen Eintrag, `git stash clear` alle. `git stash branch <name>` legt einen neuen Branch am Ursprungscommit des Eintrags an und wendet ihn dort an, was Konflikte vermeidet, wenn der alte Branch inzwischen weitergelaufen ist.

Eine Besonderheit nennt die Dokumentation: Gibt es beim `pop` Konflikte, wird der Eintrag *nicht* vom Stapel entfernt. Man löst die Konflikte, markiert sie mit `git add` und entfernt den Eintrag anschließend von Hand mit `git stash drop`. So geht nichts verloren.

```bash
git init -b main st && cd st
git config user.name "Dev"; git config user.email dev@example.org
printf "Zeile 1\nZeile 2\n" > app.txt
git add . && git commit -q -m "Start"

# Unfertige Arbeit
echo "Zeile 3 (unfertig)" >> app.txt
echo "Notiz" > neu.txt
git status -s

git stash push -u -m "WIP: Zeile 3 und Notiz"
git status -s
git stash list

# Dringende Aufgabe auf anderem Branch
git switch -q -c hotfix
echo "Hotfix" > hotfix.txt && git add . && git commit -q -m "Hotfix"
git switch -q main

git stash show -p stash@{0} | head -8
git stash pop
git status -s
git stash list

# Konflikt beim Pop
git stash push -m "Aenderung A"
printf "Zeile 1\nZeile 2 geaendert\n" > app.txt
git commit -q -am "app.txt geaendert"
git stash pop
git status -s
git stash list
```

Der erste Stash mit `-u` sichert sowohl die Änderung an `app.txt` als auch die neue Datei `neu.txt`; danach zeigt der Status ein sauberes Arbeitsverzeichnis. Auf dem Branch `hotfix` lässt sich nun in Ruhe arbeiten. Zurück auf `main` holt `git stash pop` die Änderungen zurück und entfernt den Eintrag, die Liste ist wieder leer. Im zweiten Teil wird absichtlich ein Konflikt provoziert: Zuerst wird die Änderung A in den Stash gelegt, danach ändert ein neuer Commit dieselbe Datei an derselben Stelle. Der `pop` meldet einen Konflikt, und die Stash-Liste zeigt, dass der Eintrag erhalten blieb. Zum Abschluss löst man den Konflikt und entfernt den Eintrag mit `git stash drop`.

### Typische Fehler und Stolperfallen.

Falsch ist es, den Stash als Langzeitspeicher zu nutzen. Einträge verschwinden aus dem Blick, tragen kaum erkennbare Namen, und nach Wochen weiß niemand mehr, was darin ist. Richtig ist, für Arbeit, die länger als einen Tag liegen bleibt, einen eigenen Branch anzulegen und zu committen (auch als „WIP“, später per Rebase aufzuräumen). Ein zweiter Fehler: `git stash clear` oder `git stash drop` unbedacht auszuführen. Ein verlorener Eintrag lässt sich über `git fsck --no-reflogs --unreachable` (im nächsten Abschnitt) noch finden, aber nur solange die Speicherbereinigung ihn nicht entfernt hat. Dritter Fehler: Vergessen, dass ohne `-u` neue Dateien nicht im Stash landen und beim Branchwechsel mitwandern.

### Praxisbezug.

Der Stash ist ein Werkzeug für kurze Unterbrechungen: Branch wechseln, Pull durchführen, Rebase starten, wenn das Arbeitsverzeichnis nicht sauber ist. Viele benutzen alternativ `git pull --autostash` oder `git rebase --autostash`, die den Stash automatisch anlegen und zurückspielen. Für parallele Arbeit an zwei Branches ist `git worktree add` oft die bessere Wahl: Es legt ein zweites Arbeitsverzeichnis zum selben Repository an, sodass kein Wechsel nötig ist. Die Dokumentation (abgerufen am 28.10.2026) weist darauf hin, dass ein Branch immer nur in einem Arbeitsverzeichnis ausgecheckt sein kann.

## 6.9 Recovery-Praxis

Diese Lerneinheit führt die Werkzeuge des Moduls zusammen. Die Frage, die sich nach einer Fehlbedienung stellt, ist immer dieselbe: Was ist passiert, was ist verloren, und welches Werkzeug holt es zurück? Dazu hilft eine Entscheidungshilfe.

| **Situation**                                  | **Werkzeug**                                      | **Hinweis**                            |
|:-----------------------------------------------|:--------------------------------------------------|:---------------------------------------|
| Eigenen, unveröffentlichten Commit korrigieren | `git commit --amend`, `reset --soft`, interaktiver Rebase           | nur lokal, vor dem Push                |
| Veröffentlichten Commit zurücknehmen           | `git revert`                                          | Historie bleibt intakt, auditierbar    |
| Commits nach Reset, Rebase oder Amend verloren | Reflog, dann Branch oder `reset --hard` auf alten Stand | Fristen von 30 bzw. 90 Tagen beachten  |
| Branch gelöscht                                | Reflog, `git branch name hash`                                  | Hash aus Löschmeldung oder Reflog      |
| Uncommittete Änderung verworfen                | meist nicht rettbar                               | Editor-Verlauf, Backup; vorher stashen |
| Gedroppter Stash                               | `git fsck --unreachable`                                          | nur bis zur Speicherbereinigung        |
| Geheimnis committet und gepusht                | Rotieren, dann Historie bereinigen                | siehe Modul 08                         |

Die Tabelle verbindet die Regeln: Für *lokale* Historie ist Umschreiben erlaubt, für *geteilte* Historie gilt Revert. Das Reflog fängt Fehler bei Commits auf, aber nicht bei uncommitteten Änderungen. Ein praktisches Hilfsmittel für den Notfall ist eine Hilfsfrage: „Habe ich das schon veröffentlicht?“ Wenn ja, ist Revert der Weg, wenn nein, hat man alle Freiheiten.

Das folgende Skript simuliert drei Pannen hintereinander und rettet jeweils. Es endet mit der Rettung eines gedroppten Stash-Eintrags, der sich nicht über das Reflog, sondern über `git fsck` finden lässt.

```bash
git init -b main rec && cd rec
git config user.name "Dev"; git config user.email dev@example.org
for i in 1 2 3; do echo "v$i" > f.txt; git add .; git commit -q -m "Commit $i"; done
git tag sicher

# Panne 1: ein Rebase entfernt versehentlich alle Commits des Branches
git switch -q -c feature/a
echo "a1" > a.txt && git add . && git commit -q -m "a1"
echo "a2" >> a.txt && git commit -q -am "a2"
GIT_SEQUENCE_EDITOR="sed -i 's/^pick/drop/'" git rebase -i main 2>/dev/null
git log --oneline
ORIG=$(git rev-parse "feature/a@{1}")
git reset --hard "$ORIG"
git log --oneline

# Panne 2: Stash versehentlich gedroppt
git switch -q main
echo "wichtiger Gedanke" > idee.txt
git stash push -u -m "Idee"
STASH=$(git rev-parse stash@{0})
git stash drop
git stash list
git fsck --no-reflogs --unreachable 2>/dev/null | grep commit | cut -d' ' -f3 | head -3
git stash apply $STASH
git status -s

# Panne 3: Commit auf falschem Branch
echo "hotfix" > hot.txt && git add hot.txt && git commit -q -m "Fix (auf main statt feature)"
git branch feature/b
git reset --hard HEAD~1
git log --oneline -2
git log --oneline feature/b -1
```

In der ersten Panne entfernt ein Rebase alle Commits des Branches. Das Reflog des Branches (`feature/a@{1}` ist sein Stand vor der letzten Bewegung) liefert den Hash, `git reset --hard` stellt ihn wieder her. In der zweiten Panne wird ein Stash-Eintrag mit `git stash drop` gelöscht. Weil Git beim Löschen den Hash ausgibt, hätte man ihn notieren können; hier wurde er vorher gesichert. Der Befehl `git fsck --no-reflogs --unreachable` listet Objekte, die nicht mehr von Referenzen erreicht werden; der Stash-Commit ist darunter zu finden und lässt sich mit `git stash apply <hash>` wieder anwenden. In der dritten Panne wurde ein Commit auf dem falschen Branch gemacht: Man legt am aktuellen Commit zuerst einen Branch an, sichert ihn also, und setzt `main` dann zurück. So wandert die Arbeit auf den richtigen Branch, und `main` bleibt sauber. Dieses Muster, erst sichern, dann zurücksetzen, ist die wichtigste Gewohnheit bei Rettungsaktionen.

> **Merke**
>
> Bei jeder Rettungsaktion gilt: *Erst sichern, dann reparieren.* Ein zusätzlicher Branch (`git branch backup`) kostet nichts. Danach in Ruhe das Reflog lesen, nicht raten. Und: Auf geteilten Branches wird nicht umgeschrieben, sondern reverted.

### Typische Fehler und Stolperfallen.

Falsch ist, nach einem Fehler hektisch weitere Befehle auszuführen. Jeder zusätzliche Befehl kann Reflog-Einträge verschieben und die Lage verschlimmern. Richtig ist: anhalten, `git status` und `git reflog` lesen, einen Sicherungs-Branch anlegen und erst dann handeln. Ein zweiter Fehler ist, die Verantwortung für ein veröffentlichtes Geheimnis auf Historienumschreibung zu verlagern, ohne es zu rotieren; wer das Geheimnis einmal gepusht hat, muss es als kompromittiert betrachten (Modul 08). Dritter Fehler: Wiederherstellung nur zu üben, wenn es ernst wird. Es lohnt sich, die Szenarien dieses Moduls einmal in Ruhe in einem Übungsrepository durchzuspielen.

### Praxisbezug.

In Teams gehört ein Runbook (Modul 10) dazu: Wer ruft wen, wenn ein Branch überschrieben wurde? Wo liegt ein Mirror-Backup? Wer darf Force-Push freischalten? In den meisten Fällen genügt aber das, was Sie in diesem Modul gelernt haben. Und wenn ein Kollege in Panik ist, hilft meist die ruhige Frage: „Was hast du als Letztes gemacht, und was zeigt `git reflog`?“

## Übungen zu Modul 06

**Aufgabe 1.** Was ist der Unterschied zwischen `git log -S"text"` und `git log -G"regex"`? Nennen Sie eine Frage, die man mit `-S` beantwortet.

**Aufgabe 2.** Warum ist `git blame` kein Werkzeug zur Schuldzuweisung, und wie sorgt man dafür, dass ein Formatierungs-Commit die Aussagekraft nicht zerstört?

**Aufgabe 3.** Beschreiben Sie, wie `git bisect run` entscheidet, ob ein Commit gut oder schlecht ist, und welche Bedeutung der Rückgabewert 125 hat.

**Aufgabe 4.** Erklären Sie die Wirkung von `--soft`, `--mixed` und `--hard` auf Branch, Index und Arbeitsverzeichnis. Welcher Modus kann nicht committete Arbeit vernichten?

**Aufgabe 5.** Praktisch: Bauen Sie ein Repository mit 12 Commits, von denen einer einen Fehler einführt (zum Beispiel eine kaputte Rechenfunktion). Schreiben Sie ein Testskript und finden Sie den Commit mit `git bisect run`. Wie viele Schritte hat Git gebraucht?

**Aufgabe 6.** Praktisch: Erzeugen Sie vier Commits, darunter einen „WIP“- und einen Tippfehler-Commit. Räumen Sie die Serie mit einem interaktiven Rebase auf (drop, fixup, reword) und vergleichen Sie vorher/nachher.

**Aufgabe 7.** Praktisch: Führen Sie einen harten Reset auf einen älteren Commit aus, lösen Sie danach einen Branch-Löschfehler aus und retten Sie beides über das Reflog. Beschreiben Sie, welche Einträge Sie genutzt haben.

**Aufgabe 8.** Praktisch (Transfer): Mergen Sie einen Feature-Branch mit `--no-ff`, nehmen Sie den Merge mit `git revert -m 1` zurück, versuchen Sie erneut zu mergen und erklären Sie das Ergebnis. Stellen Sie die Funktion dann wieder her. Legen Sie zusätzlich einen Stash an, droppen Sie ihn und holen Sie ihn über `git fsck` zurück.

## Lösungen zu Modul 06

**Lösung 1.** `-S"text"` findet Commits, die die Anzahl der Vorkommen von „text“ ändern (Einführung oder Entfernung). `-G"regex"` findet Commits, in deren Diff eine geänderte Zeile zum regulären Ausdruck passt. Beispiel für `-S`: „Wann wurde die Konstante `TIMEOUT=60` eingeführt?“

**Lösung 2.** Die zuletzt ändernde Person ist oft nur für ein Refactoring oder eine Formatierung verantwortlich, nicht für die Ursache. Blame dient dem Verstehen des Kontexts (Commit-Nachricht, Ticket). Formatierungs-Commits trägt man in eine Datei `.git-blame-ignore-revs` ein und setzt `blame.ignoreRevsFile`; alternativ `git blame -w`.

**Lösung 3.** Das Skript wird bei jedem Schritt ausgeführt: Rückgabewert 0 heißt gut, 1 bis 127 (außer 125) schlecht, 125 bedeutet „nicht testbar, überspringen“, andere Werte brechen die Suche ab.

**Lösung 4.** `--soft`: nur der Branch-Zeiger wandert, Index und Arbeitsverzeichnis bleiben. `--mixed`: Branch und Index werden zurückgesetzt, das Arbeitsverzeichnis bleibt. `--hard`: auch das Arbeitsverzeichnis wird überschrieben. Nur `--hard` vernichtet uncommittete Änderungen an verfolgten Dateien.

**Lösung 5.** Ablauf wie in Abschnitt 6.3 mit zwölf Commits. Für zwölf Commits sind höchstens etwa vier Schritte nötig (binäre Suche, ⌈log₂ 12⌉ = 4). Die Zahl der Schritte steht in `git bisect log`.

**Lösung 6.** Ablauf wie in Abschnitt 6.6: `GIT_SEQUENCE_EDITOR` schreibt die Todo-Liste um (`drop`, `fixup`, `reword`). Nachher: weniger Commits, neue Hashes, WIP-Datei verschwunden.

**Lösung 7.** Ablauf wie in Abschnitt 6.7. Genutzt werden `git reflog` und `git reset --hard HEAD@{n}` (Reset) beziehungsweise `git branch rettung HEAD@{n}` (gelöschter Branch).

**Lösung 8.** Nach `git revert -m 1 <merge>` fehlen die Dateien des Branches; ein erneuter `git merge feature` meldet „Already up to date“, weil die Commits schon Teil der Historie sind. Wiederherstellung durch Revert des Reverts. Stash: `git stash drop`, dann `git fsck --no-reflogs --unreachable | grep commit`, dann `git stash apply <hash>`.
