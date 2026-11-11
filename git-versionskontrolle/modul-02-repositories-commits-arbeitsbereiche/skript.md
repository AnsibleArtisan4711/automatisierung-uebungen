# Modul 02 - Repositories, Commits und Arbeitsbereiche

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (2.1 bis 2.9), danach folgen „Übungen zu Modul 02“ (8 Aufgaben) und „Lösungen zu Modul 02“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 10 h, die Summe der Lerneinheiten ergibt 9 h. Dieses Skript folgt dem höheren Wert.

**Lernziele des Moduls:** Working Tree, Staging Area, Commits und Historie sicher anwenden; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Dieses Modul vertieft den lokalen Arbeitsablauf aus Modul 01: Wir betrachten die drei Bereiche, zwischen denen Änderungen wandern, lernen, Commits sauber zu schneiden und zu benennen, und räumen das Repository auf.

| **Nr.** | **Lerneinheit**                 | **Dauer (h)** | **Prio** |
|:--------|:--------------------------------|--------------:|:--------:|
| 2.1     | Working Tree                    |           1,0 |    1     |
| 2.2     | Staging Area                    |           1,0 |    1     |
| 2.3     | Commits                         |           1,0 |    1     |
| 2.4     | Status und Diff                 |           1,0 |    1     |
| 2.5     | Ignore-Regeln                   |           1,0 |    1     |
| 2.6     | Dateien verschieben und löschen |           1,0 |    1     |
| 2.7     | Commit-Konventionen             |           1,0 |    2     |
| 2.8     | Repository-Hygiene              |           1,0 |    2     |
| 2.9     | Praxis mit lokalen Commits      |           1,0 |    2     |
|         | **Summe**                       |       **9,0** |          |

## 2.1 Working Tree

Der *Working Tree*, auf Deutsch Arbeitsverzeichnis, ist der Ort, an dem Sie tatsächlich arbeiten: die Dateien und Ordner, die Sie im Editor öffnen, kompilieren und ausführen. Er ist eine gewöhnliche Verzeichnisstruktur und unterscheidet sich von einem normalen Ordner nur dadurch, dass Git ihn mit dem letzten Commit vergleichen kann. Aus Sicht von Git befindet sich jede Datei in einem von vier Zuständen. *Untracked* (unverfolgt) ist eine Datei, die Git noch nie gesehen hat. *Unmodified* ist eine verfolgte Datei, die dem letzten Commit gleicht. *Modified* ist eine verfolgte Datei, die sich seit dem letzten Commit geändert hat. *Staged* schließlich ist eine Datei, deren aktuelle Fassung für den nächsten Commit vorgemerkt ist. Diese Zustände sind kein Dateiattribut, sondern ergeben sich laufend aus einem Vergleich: Git stellt den Working Tree, den Index und den letzten Commit gegenüber und leitet daraus ab, was zu berichten ist.

Das ist eine wichtige Einsicht: Git speichert den Zustand nicht, es berechnet ihn. Ändert man eine Datei und stellt den ursprünglichen Inhalt von Hand wieder her, zeigt `git status` sie wieder als unverändert. Ebenso wird eine gelöschte Datei als „deleted“ gemeldet, solange sie im Commit enthalten ist, aber im Arbeitsverzeichnis fehlt. Dass Git Änderungen aus dem Vergleich erkennt und nicht aus aufgezeichneten Editor-Aktionen, ist der Grund, weshalb es mit jedem Editor und jedem Werkzeug zusammenarbeitet.

Das folgende Beispiel erzeugt gezielt alle drei Fälle, nämlich eine geänderte, eine gelöschte und eine neue Datei, und liest sie am Status ab. Anschließend wird gezeigt, wie man das Arbeitsverzeichnis kontrolliert in den letzten Commit zurückversetzt.

```bash
git init -b main wt && cd wt
printf "a\nb\nc\n" > x.txt
echo "alt" > old.txt
git add . && git commit -m "Ausgangszustand"

echo d >> x.txt
rm old.txt
echo "neu" > n.txt

git status
git status -sb

git restore x.txt old.txt
git clean -n
git clean -f
```

Nach den drei Änderungen zeigt `git status` zwei Gruppen: unter „Changes not staged for commit“ die geänderte `x.txt` und die gelöschte `old.txt`, unter „Untracked files“ die neue `n.txt`. Die Kurzform `git status -sb` zeigt dasselbe in zwei Spalten, in der ersten Zeile mit dem Zweignamen. Es ist die Form, die man im Alltag am schnellsten liest. `git restore` verwirft die Änderungen an den genannten Dateien und stellt sie aus dem Index wieder her; dabei geht die Änderung unwiederbringlich verloren, weil sie nie committet wurde. Deshalb ist `git clean -n` als *Dry Run* wichtig: Er listet nur auf, welche unverfolgten Dateien gelöscht würden. Erst `git clean -f` löscht sie tatsächlich.

> **Merke**
>
> Was nie committet wurde, kann Git nicht wiederherstellen. Befehle wie `git restore <datei>` und `git clean -f` wirken auf das Arbeitsverzeichnis und sind nicht umkehrbar. Vor jedem Verwerfen: `git status` und `git diff` lesen, bei `git clean` immer zuerst `-n`.

### Typische Fehler und Stolperfallen.

Der häufigste Fehler ist, Änderungen zu verwerfen, die man noch braucht. Wer in Eile `git restore .` ausführt, um „aufzuräumen“, verliert alle ungesicherten Änderungen in einem Zug. Falsch ist das pauschale `git restore .` ohne Blick auf den Status; richtig ist, gezielt einzelne Dateien zu nennen oder die Änderungen vorher mit `git stash` beiseitezulegen (Modul 6). Ein weiterer Stolperstein: Wer Dateien im Dateimanager statt mit Git löscht oder verschiebt, erzeugt „deleted“ und „untracked“ nebeneinander; Git erkennt die Umbenennung erst später beim Vergleich (siehe Abschnitt 2.6).

### Praxisbezug.

Im Projekt ist ein *sauberer* Working Tree die Voraussetzung für fast alles andere: Zweig wechseln, Pull, Rebase, Merge. Wer mit schmutzigem Arbeitsverzeichnis einen Zweig wechselt, bekommt entweder Fehler oder nimmt unbeabsichtigt Änderungen mit. Gute Gewohnheit ist daher, vor jedem dieser Schritte kurz `git status -sb` zu prüfen.

## 2.2 Staging Area

Zwischen Arbeitsverzeichnis und Historie steht die *Staging Area*, auch *Index* genannt. Technisch ist sie die Datei `.git/index`, die eine Liste aller Dateien samt Hashes für den nächsten Commit enthält. Sie ist der Entwurf des nächsten Schnappschusses. Wenn Sie `git add datei` ausführen, berechnet Git den Hash des aktuellen Dateiinhalts, legt ihn als Blob in der Objektdatenbank ab und trägt ihn im Index ein. Wenn Sie anschließend `git commit` ausführen, wird aus dem Index ein Tree gebaut und ein Commit darum gewickelt. Wichtig ist, dass `git add` den Inhalt *zu diesem Zeitpunkt* festhält: Ändern Sie die Datei danach weiter, ist die neuere Änderung nicht im Commit, bis Sie sie erneut vormerken.

Wozu das Ganze? Der Index erlaubt, Commits fachlich sauber zu schneiden. In der Praxis arbeitet man selten in perfekter Reihenfolge: Man behebt einen Fehler, bemerkt unterwegs einen Tippfehler an anderer Stelle, formatiert nebenbei eine Datei neu. Ohne Staging würde alles in einem Commit landen. Mit Staging merkt man erst die Fehlerbehebung vor, committet sie, und behandelt Tippfehler und Formatierung separat. Dafür gibt es mehrere Varianten von `git add`. `git add <datei>` merkt eine Datei vor, `git add <ordner>` alles darunter. `git add -u` merkt alle bereits verfolgten Dateien vor (Änderungen und Löschungen), aber keine neuen. `git add -A` merkt alles vor, auch neue Dateien. Am feinsten arbeitet `git add -p`, das jede zusammenhängende Änderung (einen *Hunk*) einzeln anzeigt und fragt, ob sie vorgemerkt werden soll.

Das folgende Beispiel nutzt eine Datei, in der zwei unabhängige Änderungen stecken. Mit `git add -p` wird nur die erste vorgemerkt. Weil dieser Befehl interaktiv ist, ist hier zusätzlich ein Ausschnitt der Sitzung abgedruckt. Anschließend wird gezeigt, wie sich der Index prüfen und eine Vormerkung zurücknehmen lässt.

```bash
git init -b main stage && cd stage
seq 1 30 > zahlen.txt
git add . && git commit -m "Zahlen 1 bis 30"

sed -i '2s/.*/zwei/' zahlen.txt
sed -i '29s/.*/neunundzwanzig/' zahlen.txt

git add -p zahlen.txt
git diff --staged
git diff
git commit -m "Zeile 2 ausgeschrieben"

git restore --staged zahlen.txt
```

Die beiden `sed`-Aufrufe ändern Zeile 2 und Zeile 29. Weil mehr als drei Zeilen dazwischen liegen, teilt Git das in zwei Hunks. Bei `git add -p` fragt es für jeden Hunk nach, etwa mit `Stage this hunk [y,n,q,a,d,s,e,?]?`. Antwortet man beim ersten mit `y` (ja) und beim zweiten mit `n` (nein), ist nur die erste Änderung im Index. Danach zeigt `git diff --staged` die vorgemerkte Änderung (Zeile 2) und `git diff` die noch nicht vorgemerkte (Zeile 29). Der Commit enthält also nur die erste. Die letzte Zeile zeigt das Gegenstück zum Vormerken: `git restore --staged` nimmt Dateien aus dem Index, ohne das Arbeitsverzeichnis zu verändern. Praktisch ist auch die Antwort `s` (split), die einen Hunk weiter aufteilt, und `e`, mit der man den Hunk von Hand bearbeitet.

### Typische Fehler und Stolperfallen.

Zwei Fehler sind typisch. Erstens vergisst man, dass `git add` den Stand zum Zeitpunkt des Befehls einfriert. Wer nach dem `git add` noch weiter an der Datei arbeitet und dann nur `git commit` ruft, committet die ältere Fassung, und der Status zeigt dieselbe Datei gleichzeitig als „staged“ und „modified“. Falsch ist, aus dem Gedächtnis zu committen; richtig ist, vor dem Commit `git status` zu lesen, und bei zwei Einträgen für eine Datei neu vorzumerken. Zweitens wird `git add .` gedankenlos genutzt und bringt Dateien mit, die nicht hineingehören. Die Alternative `git add -p` zwingt zum Lesen jeder Änderung und fängt solche Fehler zuverlässig ab.

### Praxisbezug.

In Code Reviews fällt auf, wenn ein Commit mehrere Themen mischt. Wer seine Änderungen mit `git add -p` aufteilt, liefert Commits, die der Reviewer einzeln verstehen kann, und die sich bei Problemen einzeln zurücknehmen lassen. Viele IDEs bieten dieselbe Funktion grafisch an, indem man in der Diff-Ansicht einzelne Zeilen oder Blöcke anhakt.

## 2.3 Commits

Ein *Commit* ist der Schnappschuss, den `git commit` aus dem Index erzeugt. Wie in Modul 01 beschrieben, besteht er aus einem Verweis auf einen Tree, den Verweisen auf seine Eltern, Metadaten und einer Nachricht. Die Metadaten sind *Autor* (Name, E-Mail, Zeitpunkt der ursprünglichen Änderung) und *Committer* (Name, E-Mail, Zeitpunkt des Anlegens des Commits). In vielen Fällen sind beide identisch. Sie unterscheiden sich, wenn jemand anderes Ihren Commit übernimmt, etwa bei `git cherry-pick`, `git rebase` oder `git am`: Der Autor bleibt, der Committer wird die ausführende Person, und der Committer-Zeitstempel ändert sich. Der Commit-Hash ist der Fingerabdruck aus allem davon; ändert sich ein Detail, entsteht ein neuer Hash.

Daraus folgt eine grundlegende Regel: Commits sind unveränderlich. Der Befehl `git commit --amend`, mit dem man den letzten Commit „korrigiert“, ändert ihn in Wahrheit nicht, sondern erzeugt einen neuen Commit mit einem neuen Hash und lässt den alten Commit liegen, auf den nur noch kein Zweig zeigt. Das ist harmlos, solange der Commit nur lokal existiert. Wurde er bereits an andere weitergegeben, entsteht dadurch eine abweichende Historie, die später aufwendig gerade zu ziehen ist. Deshalb gilt die Faustregel: *amend* nur für Commits, die noch nicht veröffentlicht sind.

Die Commit-Nachricht ist der wichtigste Teil für Menschen. Das folgende Beispiel zeigt einen Commit mit sinnvoller Nachricht, das Betrachten seiner Metadaten und die Korrektur eines vergessenen Details mit `--amend`.

```bash
git init -b main cm && cd cm
echo "Version 1" > app.txt
git add app.txt
git commit -m "app.txt: erste Fassung" \
  -m "Die Datei enthaelt den Startzustand der Anwendung. Sie wird als
Ausgangspunkt fuer die weiteren Aenderungen benoetigt."

git show --stat --format=fuller HEAD
git log -1 --format='%h  %an  %s'

echo "Hinweis" > hinweis.txt
git add hinweis.txt
git commit --amend --no-edit
git log --oneline
git reflog -3
```

Die zwei `-m`-Angaben im ersten Commit bilden zwei Absätze: Der erste ist die Betreffzeile, der zweite der erklärende Text; Git trennt sie automatisch durch eine Leerzeile. Die Ausgabe von `git show --format=fuller` zeigt Author und Committer getrennt mit ihren Zeitstempeln. `git log -1 --format=...` zeigt, wie man gezielt Felder ausgibt: `%h` ist der Kurzhash, `%an` der Autorname, `%s` der Betreff. Danach wird eine vergessene Datei nachträglich in den letzten Commit aufgenommen. `--no-edit` übernimmt die Nachricht unverändert. Das abschließende `git reflog` zeigt, dass der alte Commit weiterhin im Protokoll steht, solange er nicht aufgeräumt wird; das macht `--amend` in den ersten Wochen nach dem Fehler rückgängig machbar (Modul 6).

### Typische Fehler und Stolperfallen.

Verbreitet sind unbrauchbare Nachrichten. „Fix“, „Update“ oder „Änderungen“ sagen weder, was noch warum etwas geändert wurde. Falsch: `git commit -m "fix"`. Richtig: `git commit -m "Login: Fehler bei leerem Passwort abgefangen"`, ergänzt um einen Absatz mit dem Grund, wenn dieser nicht offensichtlich ist. Ein weiterer klassischer Fehler ist `--amend` nach dem `push`: Das erzwingt anschließend ein `git push --force`, das die Arbeit anderer überschreiben kann. Wer hier unsicher ist, setzt lieber einen neuen, korrigierenden Commit obenauf.

### Praxisbezug.

Commit-Nachrichten werden bei Fehlern wichtig: `git blame` (Modul 6) zeigt für jede Zeile den zuletzt ändernden Commit, und dessen Nachricht ist oft die einzige Erklärung, warum der Code so aussieht. Teams legen deshalb Wert auf aussagekräftige Nachrichten und verknüpfen sie mit Tickets, etwa durch eine Ticketnummer in der Betreffzeile.

> **Praxisbeispiel**
>
> Ein Fehler tritt nur bei Kunden mit alten Datenbeständen auf. Per `git blame` landen Sie bei einem Commit mit der Nachricht „Fix“ und keinerlei Beschreibung. Zwei Monate später ist niemand sicher, warum die Prüfung entfernt wurde. Mit einer Nachricht wie „Altdaten-Prüfung entfernt, da Importer seit Release 4.1 bereits bereinigt“ hätten Sie die Antwort in Sekunden.

## 2.4 Status und Diff

Wer Git sicher bedienen will, muss zwei Befehle blind beherrschen: `git status` und `git diff`. Der Status beantwortet die Frage „Was ist los?“, indem er zeigt, welche Dateien sich in welchem Zustand befinden. Das Diff beantwortet die Frage „Was genau hat sich geändert?“, indem es Zeile für Zeile Unterschiede zeigt. Beide arbeiten immer mit demselben Prinzip: Sie vergleichen zwei der drei Bereiche. Ohne Option zeigt `git diff` die Unterschiede zwischen Working Tree und Index, also das, was noch *nicht* vorgemerkt ist. Mit `--staged` (oder `--cached`) vergleicht es Index und letzten Commit, also das, was der nächste Commit enthalten wird. Mit `git diff HEAD` vergleicht man Working Tree und letzten Commit, also die Summe von beidem.

Darüber hinaus kann man beliebige Stände vergleichen. `git diff A B` zeigt die Unterschiede zwischen zwei Commits, Zweigen oder Tags. Die Schreibweise `A..B` ist beim Diff gleichbedeutend mit `A B`; `A...B` vergleicht dagegen *B* mit dem gemeinsamen Vorfahren von A und B, was bei Zweigen die Frage beantwortet „Was hat B seit der Abzweigung getan?“. Zur Übersicht gibt es `--stat` (geänderte Dateien mit Zeilenzahl), `--name-only` und `--name-status` (nur Dateinamen, letzterer mit Art der Änderung) sowie `--word-diff` (wortweise statt zeilenweise, praktisch für Prosa). Und `git diff --check` warnt vor Leerzeichenfehlern wie Zeilenendleerzeichen oder übrig gebliebenen Konfliktmarkern, ein guter letzter Blick vor dem Commit.

Das Beispiel durchläuft die drei Vergleichsebenen an einer Datei und zeigt danach den Vergleich zweier Commits.

```bash
git init -b main df && cd df
printf "Apfel\nBirne\nKirsche\n" > obst.txt
git add . && git commit -m "Obstliste"

echo "Dattel" >> obst.txt
git add obst.txt
echo "Erdbeere" >> obst.txt

git status -sb
git diff
git diff --staged
git diff HEAD

git commit -m "Dattel ergaenzt"
git add obst.txt && git commit -m "Erdbeere ergaenzt"
git diff HEAD~2 HEAD --stat
git diff HEAD~2 HEAD --name-status
```

Nach den beiden `echo`-Zeilen hat `obst.txt` eine vorgemerkte (Dattel) und eine nicht vorgemerkte Änderung (Erdbeere). Der Status zeigt dafür `MM`: Die erste Spalte steht für den Index, die zweite für das Arbeitsverzeichnis. `git diff` zeigt dann nur die Erdbeere, `git diff --staged` nur die Dattel und `git diff HEAD` beide. Nach den zwei Commits liefert der Vergleich `HEAD~2` mit `HEAD` die Gesamtänderung; `--stat` fasst sie knapp zusammen, `--name-status` gibt die Dateiliste mit dem Buchstaben `M` für modifiziert aus.

### Typische Fehler und Stolperfallen.

Der klassische Verwirrungsfall: Jemand führt `git add` aus, tippt anschließend `git diff` und sieht nichts. Er schließt, die Änderung sei verschwunden. Sie ist aber nur im Index, und das einfache Diff zeigt eben nur Nicht-Vorgemerktes. Falsch ist, aus einer leeren `git diff`-Ausgabe auf „nichts geändert“ zu schließen; richtig ist, zusätzlich `git diff --staged` oder gleich `git status` zu lesen. Ein zweiter Fehler betrifft Binärdateien und riesige Diffs: Wer ein Diff mit tausenden Zeilen im Pager durchscrollt, übersieht Wichtiges. Für Überblick gibt es `--stat`, für gezielte Prüfung `git diff -- <pfad>`.

### Praxisbezug.

Das Diff ist das tägliche Werkzeug beim Review: Pull Requests auf Plattformen sind im Kern `git diff main...feature` mit Oberfläche. Wer das Kommando kennt, kann auch lokal prüfen, was ein Zweig eingebracht hat, bevor er ihn zusammenführt. Und in Pipelines wird `git diff --name-only` genutzt, um nur geänderte Dateien zu prüfen oder zu bauen (Modul 9).

## 2.5 Ignore-Regeln

Nicht alles im Projektordner gehört in die Versionskontrolle. Build-Ausgaben, Logdateien, lokale Konfigurationen, Abhängigkeitsordner wie `node_modules` und vor allem Dateien mit Geheimnissen wie `.env` sollen nie im Repository landen. Git bietet dafür Ignore-Regeln. Sie wirken nur auf *unverfolgte* Dateien: Eine Datei, die bereits committet ist, wird von einer später hinzugefügten Regel nicht mehr ignoriert, sondern weiter verfolgt. Das ist der meistmissverstandene Punkt bei diesem Thema.

Regeln stehen in einer `.gitignore`-Datei im Arbeitsverzeichnis, die man ins Repository committet und damit mit dem Team teilt. Daneben gibt es zwei persönliche Orte, die nicht geteilt werden: `.git/info/exclude` gilt nur für dieses Repository, und die globale Datei, auf die `core.excludesFile` zeigt (Standard: `~/.config/git/ignore`), gilt für alle Repositories des Benutzers. Die Faustregel lautet: Projektspezifische Dateien (Build-Ordner, Abhängigkeiten) gehören in die geteilte `.gitignore`; persönliche (Editor-Dateien, Betriebssystem-Dateien wie `.DS_Store` oder `Thumbs.db`) gehören in die globale Datei.

Die Mustersyntax ist knapp. Ein Stern `*` steht für beliebig viele Zeichen innerhalb eines Pfadelements, ein Fragezeichen für genau ein Zeichen. `**` greift über Verzeichnisgrenzen hinweg: `**/temp` trifft jeden Ordner namens `temp`, `a/**/b` trifft `a/b`, `a/x/b` und so weiter. Ein Schrägstrich am Ende (`build/`) beschränkt das Muster auf Verzeichnisse. Ein Schrägstrich am Anfang oder in der Mitte verankert das Muster am Ort der `.gitignore`; `/secret.txt` trifft also nur die Datei im Wurzelverzeichnis, `secret.txt` dagegen auf jeder Ebene. Ein Ausrufezeichen vorn (`!`) negiert eine frühere Regel. Allerdings lässt sich keine Datei wieder einschließen, wenn ein übergeordneter Ordner ausgeschlossen ist. Bei mehreren passenden Regeln gewinnt die zuletzt stehende.

> **Ergänzung aus der Nachrecherche**
>
> **Rangfolge der Quellen.** Die Git-Dokumentation zu `gitignore` (abgerufen am 28.10.2026) nennt als Rangfolge von hoch nach niedrig: Muster auf der Kommandozeile, `.gitignore`-Dateien (tiefer liegende überstimmen höhere), `.git/info/exclude` und zuletzt die Datei aus `core.excludesFile`. Auch dort steht der Hinweis, dass bereits verfolgte Dateien nicht betroffen sind. Sie nehmen Sie mit `git rm --cached <datei>` aus der Verfolgung und tragen sie dann in die `.gitignore` ein.

Das Beispiel richtet Regeln ein und prüft mit `git check-ignore -v`, welche Regel auf welche Datei wirkt. Dieser Befehl ist das Hauptwerkzeug, wenn eine Ignore-Regel nicht so wirkt wie gedacht.

```bash
git init -b main ign && cd ign
printf "*.log\nbuild/\n!keep.log\n/secret.txt\n" > .gitignore
mkdir -p build sub
touch a.log keep.log build/out.bin secret.txt sub/secret.txt sub/b.log

git status -s
git check-ignore -v a.log keep.log build/out.bin secret.txt sub/secret.txt

echo "token=abc" > .env
git add .env && git commit -m "versehentlich .env" 2>&1 | tail -1
echo ".env" >> .gitignore
git status -s
git rm --cached .env
git status -s
```

Die Regel `*.log` ignoriert alle Logdateien, aber `!keep.log` nimmt diese eine wieder aus; daher erscheint `keep.log` im Status als unverfolgt, `a.log` und `sub/b.log` dagegen nicht. `build/` ignoriert den Ordner, `/secret.txt` nur die Datei im Wurzelverzeichnis; `sub/secret.txt` bleibt sichtbar. `git check-ignore -v` nennt für jede Datei die treffende Regel mit Datei und Zeilennummer. Beachten Sie, dass es auch bei `keep.log` eine Zeile ausgibt, nämlich die Negationsregel: Die Datei „trifft“ also eine Regel, wird aber nicht ignoriert. Der zweite Teil spielt den Ernstfall durch. Die Datei `.env` wurde versehentlich committet; die nachträglich eingetragene Regel greift nicht, denn die Datei ist bereits verfolgt. Erst `git rm --cached .env` entfernt sie aus dem Index, lässt sie aber im Arbeitsverzeichnis liegen. Danach greift die Regel.

### Typische Fehler und Stolperfallen.

Falsch ist die Annahme, dass das Eintragen in die `.gitignore` Geheimnisse schützt, die schon im Repository stehen. Selbst nach `git rm --cached` bleibt der Inhalt in der Historie früherer Commits abrufbar. Ein einmal committetes Geheimnis gilt als kompromittiert und muss rotiert werden (Modul 8). Richtig ist, die `.gitignore` *vor* dem ersten Commit anzulegen, am besten aus einer Vorlage für die eingesetzte Sprache. Ein weiterer Fehler ist die Negation unter ausgeschlossenem Ordner: Wer `build/` ignoriert und `!build/wichtig.txt` hinterherschreibt, erlebt, dass die Datei nicht eingeschlossen wird.

### Praxisbezug.

Jedes Repository sollte beim Anlegen eine `.gitignore` erhalten. Vorlagen für fast alle Sprachen und Frameworks stehen öffentlich zur Verfügung; Plattformen bieten sie beim Anlegen eines neuen Repositories zur Auswahl an. In Unternehmen ergänzen Secret-Scanner (Modul 8) die Ignore-Regeln als zweite Verteidigungslinie.

> **Praxisbeispiel**
>
> Ein neuer Kollege committet am ersten Tag seine `.env`-Datei mit dem Zugangsschlüssel zur Testdatenbank und pusht sie. Ein Secret-Scanner meldet den Fund nach wenigen Minuten. Das Löschen der Datei im nächsten Commit genügt nicht, denn der Schlüssel steht in der Historie; der Schlüssel wird rotiert und die Datei in die `.gitignore` aufgenommen. Hätte die Projektvorlage die Regel von Anfang an enthalten, wäre der Vorfall ausgeblieben.

## 2.6 Dateien verschieben und löschen

Dateien umbenennen und löschen sind im Alltag so häufig, dass man ihre Behandlung durch Git verstehen sollte. Für Git gibt es streng genommen gar keine „Umbenennung“: Es speichert Schnappschüsse, und in einem Schnappschuss ist die Datei mit dem alten Namen verschwunden, während eine Datei mit dem neuen Namen und (fast) gleichem Inhalt hinzugekommen ist. Erst beim Anzeigen *erkennt* Git die Umbenennung, indem es gelöschte und neue Dateien mit ähnlichem Inhalt (standardmäßig ab etwa 50 % Übereinstimmung) einander zuordnet. Das heißt auch: Es spielt keine Rolle, ob man `git mv` nutzt oder die Datei im Dateimanager umbenennt und dann alles mit `git add -A` vormerkt. Das Ergebnis im Commit ist dasselbe. `git mv alt neu` ist lediglich eine Abkürzung für Verschieben und Vormerken in einem Schritt.

Ähnlich ist `git rm <datei>` die Kombination aus Löschen im Arbeitsverzeichnis und Vormerken der Löschung. Mit `git rm --cached <datei>` wird die Datei nur aus dem Index genommen und bleibt auf der Platte, was nach versehentlich committeten Dateien nötig ist. Damit Umbenennungen in der Historie nachvollziehbar bleiben, gibt es `git log --follow -- <datei>`, das die Historie über die Umbenennung hinweg verfolgt. Wichtig ist dabei die Disziplin: Wer eine Datei umbenennt *und* stark verändert, riskiert, dass Git die Zuordnung verliert. Deshalb gilt als gute Praxis, Verschieben und Ändern in getrennte Commits zu legen.

Das Beispiel benennt eine Datei um, löscht eine andere und zeigt, wie Git beides in der Historie erkennt. Danach folgt der Sonderfall der Schreibweise unter Windows und macOS.

```bash
git init -b main mv && cd mv
printf "Zeile 1\nZeile 2\nZeile 3\nZeile 4\n" > bericht.txt
echo "alt" > temp.txt
git add . && git commit -m "Ausgangsstand"

git mv bericht.txt report.txt
git status -s
git commit -m "bericht.txt in report.txt umbenannt"

git rm temp.txt
git commit -m "temp.txt entfernt"

git log --follow --oneline -- report.txt
git log --stat --oneline -3

# nur Gross-/Kleinschreibung aendern (case-insensitive Dateisysteme)
git mv -f report.txt Report.txt
git status -s
```

Nach `git mv` zeigt der Status `R  bericht.txt -> report.txt`: Git meldet die Umbenennung (*Rename*), obwohl technisch nur ein neuer Tree entstanden ist. `git rm temp.txt` löscht die Datei und merkt die Löschung vor. Der Log mit `--follow` zeigt auch die Commits aus der Zeit vor der Umbenennung; ohne die Option würde die Historie bei der Umbenennung enden. Der letzte Teil betrifft Dateisysteme, die Groß- und Kleinschreibung nicht unterscheiden (Windows, standardmäßig macOS): Dort kann man `report.txt` nicht ohne Weiteres in `Report.txt` umbenennen, weil beide Namen dieselbe Datei meinen. `git mv -f` erzwingt es; als Alternative geht der Umweg über einen Zwischennamen.

### Typische Fehler und Stolperfallen.

Falsch ist, eine Datei zu verschieben und im selben Commit komplett umzuschreiben; die Ähnlichkeit fällt unter die Schwelle, Git zeigt eine Löschung und eine Neuanlage, und die Historie ist unterbrochen. Richtig ist: erst verschieben und committen, dann ändern. Ein zweiter Fehler: Wer mit `rm` statt `git rm` löscht, hat die Datei zwar entfernt, aber die Löschung ist nicht vorgemerkt; ein anschließendes `git commit` ohne `-a` nimmt sie nicht mit. `git add -A` oder `git commit -a` holen das nach. Dritter Fall: Wer eine Datei wirklich aus der gesamten Historie entfernen will (zum Beispiel wegen eines Geheimnisses), ist mit `git rm` nicht am Ziel; das ist eine Historienumschreibung und gehört nach Modul 6 und Modul 8.

### Praxisbezug.

Umbenennungen sind bei Refactorings alltäglich: Pakete werden verschoben, Verzeichnisse neu geordnet. Damit `git blame` und `git log --follow` danach noch brauchbare Ergebnisse liefern, lohnt es sich, Umbenennungen in eigene, reine Verschiebe-Commits zu packen. Bei plattformübergreifenden Teams sind Groß-/Kleinschreibungskonflikte ein bekanntes Ärgernis, das man mit Namenskonventionen (durchgängig klein, mit Bindestrichen) vermeidet.

## 2.7 Commit-Konventionen

Wenn mehrere Personen in ein Repository committen, entsteht ohne Absprachen eine uneinheitliche Historie: manche schreiben Sätze, andere Stichworte, wieder andere nur Ticketnummern. Commit-Konventionen legen deshalb Form und Inhalt von Nachrichten fest. Die älteste und am weitesten verbreitete Konvention ist handwerklicher Natur: Eine Betreffzeile von höchstens etwa 50 Zeichen, im Imperativ oder als knappe Feststellung formuliert, ohne Schlusspunkt; danach eine Leerzeile; danach ein Fließtext, der auf etwa 72 Zeichen Breite umbricht und erklärt, *warum* die Änderung nötig ist und welche Alternativen verworfen wurden. Der Hintergrund: Viele Werkzeuge zeigen nur die Betreffzeile (`git log --oneline`, Plattformlisten), der Rest ist für die Detailansicht.

Auf diese Grundform bauen strukturierte Konventionen auf. Die bekannteste sind die *Conventional Commits*. Dort beginnt jede Nachricht mit einem Typ, optional gefolgt von einem Bereich in Klammern, einem Doppelpunkt und der Beschreibung, etwa `feat(login): Passwort-Reset per E-Mail ergänzt`. Gängige Typen sind `feat` für neue Funktionen und `fix` für Fehlerbehebungen; hinzu kommen je nach Team `docs`, `refactor`, `test`, `chore` und andere. Eine inkompatible Änderung wird mit einem `!` nach dem Typ oder einer Fußzeile `BREAKING CHANGE:` gekennzeichnet. Der Nutzen der Struktur liegt in der Maschinenlesbarkeit: Werkzeuge können daraus Changelogs erzeugen und nach Semantic Versioning (Modul 7) die nächste Versionsnummer ableiten. Ein `fix` entspricht dann einem Patch-Release, ein `feat` einem Minor-Release, eine inkompatible Änderung einem Major-Release. Die Details zu SemVer und Release-Automatisierung folgen in Modul 7 und Modul 9; hier genügt, die Konvention zu kennen und anwenden zu können.

> **Ergänzung aus der Nachrecherche**
>
> **Hinweis zur Quellenlage.** Die Beschreibung der Conventional Commits stützt sich auf die Spezifikation in Version 1.0.0. Die Spezifikationsseite war in dieser Sitzung nicht abrufbar; die Darstellung beruht deshalb auf dem Wissensstand des Autors und ist im Quellenverzeichnis unter „weiterführend“ aufgeführt. Prüfen Sie Details (etwa die erlaubten Typen) vor der Einführung im Team gegen die Spezifikation.

Das Beispiel zeigt, wie man eine Nachrichtenvorlage hinterlegt, damit der Editor beim Commit die Struktur vorgibt, und wie ein Commit nach Conventional Commits aussieht.

```bash
cat > ~/.gitmessage <<'EOF'
# <typ>(<bereich>): <kurze Beschreibung>   (max. 50 Zeichen)
#
# Warum ist die Aenderung noetig? (Umbruch bei ca. 72 Zeichen)
#
# Refs: TICKET-123
EOF
git config --global commit.template ~/.gitmessage

git init -b main konv && cd konv
echo "x" > login.txt && git add login.txt
git commit -m "fix(login): Fehler bei leerem Passwort abgefangen" \
  -m "Bisher fuehrte ein leeres Passwortfeld zu einem Serverfehler (500).
Die Eingabe wird nun vor dem Absenden geprueft.

Refs: TICKET-123"

git log -1
git log --oneline --grep='^feat'
git interpret-trailers --parse < <(git log -1 --format=%B)
```

Die Vorlage in `~/.gitmessage` enthält nur Kommentarzeilen mit `#`; sie werden beim Commit ignoriert, dienen aber als Gedächtnisstütze. `commit.template` bindet die Datei ein. Der Commit besteht aus einer Betreffzeile mit Typ und Bereich, einem erklärenden Absatz und einer *Trailer*-Zeile am Ende. Trailer sind Schlüssel-Wert-Zeilen im Format `Schlüssel: Wert`, die Git erkennt, zum Beispiel `Signed-off-by`, `Co-authored-by` oder hier `Refs`. `git log --grep='^feat'` filtert die Historie nach dem Typ, und `git interpret-trailers --parse` gibt die Trailer maschinenlesbar aus.

### Typische Fehler und Stolperfallen.

Eine Konvention nützt nur, wenn sie eingehalten wird. Falsch ist, sie nur in einem Dokument festzuhalten und auf guten Willen zu hoffen; richtig ist, sie technisch zu prüfen, etwa mit einem `commit-msg`-Hook lokal (Modul 9) und mit einer Prüfung in der Pipeline. Ein weiterer Fehler ist Übertypisierung: Teams, die zehn Typen und Pflicht-Bereiche einführen, erleben schnell Frust und uneinheitliche Anwendung. Beginnen Sie mit wenigen Typen. Und schließlich: Eine Betreffzeile wie `feat: fixes` erfüllt zwar die Form, aber nicht den Zweck.

### Praxisbezug.

Viele Open-Source-Projekte und Firmen nutzen Conventional Commits in Kombination mit Release-Werkzeugen, die aus den Nachrichten Changelogs und Versionsnummern erzeugen. Auch ohne Automatisierung lohnt sich die Regel „Ticketnummer plus Kurzbeschreibung“, weil sie später den Weg vom Code zur Anforderung ebnet, was in Audits gern verlangt wird.

## 2.8 Repository-Hygiene

Ein Repository ist ein langlebiges Archiv, und was einmal committet ist, bleibt in der Historie, auch wenn man es später löscht. Hygiene heißt deshalb, die Aufnahme von Ballast zu verhindern, bevor sie passiert. Dazu gehören zuerst *generierte Artefakte*: Kompilate, Build-Ordner, Paketarchive, Minifizierungen und Testberichte entstehen aus dem Quelltext und lassen sich jederzeit neu erzeugen. Sie im Repository zu halten, bläht es auf, erzeugt ständig Merge-Konflikte in nicht lesbaren Dateien und verschleiert, welche Änderungen von Menschen stammen. Ähnlich ist es mit großen Binärdateien wie Videos, Datenbankabzügen und Installationsprogrammen: Git speichert jede Fassung vollständig, sodass sich schon wenige Änderungen an einer 100-MB-Datei zu Gigabytes summieren. Dafür gibt es Alternativen wie Git LFS (Modul 5 und 10) oder externe Artefaktspeicher.

Zur Hygiene gehört weiter die Kontrolle von Zeilenenden. Windows nutzt CRLF, Linux und macOS LF; ohne Regelung entstehen „Phantomänderungen“, bei denen jede Zeile als geändert erscheint. Die Lösung ist eine Datei `.gitattributes` im Repository mit einer Regel wie `* text=auto`, die Git die Normalisierung überlässt, und optional eine `.editorconfig`, die Editoren einheitlich einstellt. Und schließlich gehört dazu die Gewohnheit kleiner, reviewbarer Änderungen: Ein Commit mit tausend geänderten Zeilen ist faktisch unreviewbar; wer Formatierungsänderungen von inhaltlichen trennt, hält das Diff lesbar.

Das nächste Beispiel zeigt, wie man ein Repository auf Ballast untersucht und die Zeilenendbehandlung festlegt. Es sucht die größten Objekte in der Historie und listet verfolgte Dateien, die nach Artefakten aussehen.

```bash
git init -b main hyg && cd hyg
printf "* text=auto\n*.png binary\n" > .gitattributes
mkdir build && head -c 3000000 /dev/urandom > build/app.bin
echo "code" > main.txt
git add . && git commit -m "Start (mit Fehler: build/ ist drin)"

git count-objects -vH
git ls-files | grep -E '^(build|dist|target)/'
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob"' | sort -k3 -n -r | head -3

git rm -r --cached build
echo "build/" >> .gitignore
git add .gitignore
git commit -m "build/ aus der Verfolgung genommen"
git diff --check
```

`git count-objects -vH` zeigt die Größe der Objektdatenbank. Die `git ls-files`-Zeile findet verfolgte Dateien in typischen Artefaktordnern. Die lange Pipeline darunter ist ein bewährtes Rezept: `git rev-list --objects --all` listet alle Objekte der Historie, `git cat-file --batch-check` ergänzt Typ und Größe, `awk` behält nur Blobs, und `sort` reiht sie nach Größe. So findet man die größten Dateien aller Zeiten, auch wenn sie längst gelöscht sind. Danach wird der Fehler repariert: `git rm -r --cached build` nimmt den Ordner aus der Verfolgung, die `.gitignore` verhindert die Rückkehr, und `git diff --check` prüft auf Leerzeichenfehler. Wichtig: Der große Blob bleibt in der Historie des ersten Commits erhalten; er verschwindet nur durch eine Historienumschreibung (Modul 6).

### Typische Fehler und Stolperfallen.

Falsch ist das Muster „erst committen, später aufräumen“: Das Löschen in einem Folgecommit entfernt nichts aus der Historie, die Repositorygröße bleibt. Richtig ist, vor dem Commit mit `git status` zu prüfen und größere Dateien bewusst zu entscheiden. Ein zweiter Fehler: Zeilenenden werden dem Zufall überlassen. Falsch ist, `core.autocrlf` nur bei einzelnen Entwicklern zu setzen; richtig ist, die Regel mit `.gitattributes` ins Repository zu legen, sodass sie für alle gleich gilt.

### Praxisbezug.

Beim Übernehmen fremder Repositories ist ein erster Blick auf Größe und größte Objekte Pflicht, weil er teure Überraschungen bei Klon-Zeiten und Speicherquoten verhindert. Plattformen setzen oft Größengrenzen (etwa pro Datei), und CI-Systeme klonen bei jedem Lauf, sodass ein aufgeblähtes Repository jede Pipeline verlangsamt.

## 2.9 Praxis mit lokalen Commits

Diese Lerneinheit ist eine zusammenhängende Übung, die alles aus diesem Modul verbindet. Wir bauen ein kleines Projekt in mehreren logischen Schritten auf und achten darauf, dass jeder Commit ein Thema hat. Die Reihenfolge entspricht dem Alltag: Projekt anlegen, Funktion ergänzen, Fehler beheben, Doku nachziehen. Danach untersuchen wir die entstandene Historie mit den Vergleichswerkzeugen, die wir gelernt haben.

Das Skript unten legt ein Mini-Projekt an, einen Rechner als Shell-Skript, und führt es durch fünf Commits. Zwischendurch wird absichtlich ein Fehler eingebaut und per eigenem Commit korrigiert, wie es im echten Projekt vorkommt. Lesen Sie die Kommentare mit: Sie benennen die Absicht jedes Schritts.

```bash
git init -b main rechner && cd rechner
printf "*.log\n*.tmp\n" > .gitignore
printf "# Rechner\nKleines Uebungsprojekt.\n" > README.md
git add . && git commit -m "Projekt angelegt: README und .gitignore"

# Funktion 1: Addition (mit absichtlichem Fehler)
cat > rechne.sh <<'EOF'
#!/bin/sh
add() { echo $(( $1 - $2 )); }
EOF
git add rechne.sh && git commit -m "feat: Addition hinzugefuegt"

# Fehler beheben
sed -i 's/\$1 - \$2/$1 + $2/' rechne.sh
git commit -am "fix: Addition rechnete Differenz statt Summe"

# Funktion 2
echo 'mul() { echo $(( $1 * $2 )); }' >> rechne.sh
git commit -am "feat: Multiplikation hinzugefuegt"

# Doku nachziehen
echo "Funktionen: add, mul" >> README.md
git commit -am "docs: Funktionsliste in README"

git log --oneline --graph
git log -p --follow -- rechne.sh | head -40
git diff HEAD~3 HEAD --stat
git show HEAD~2 --stat
git diff HEAD~4 HEAD -- rechne.sh
```

Nach dem Skript hat das Repository fünf Commits. `git log --oneline --graph` zeigt sie als lineare Kette. `git log -p --follow -- rechne.sh` listet nur die Commits, die `rechne.sh` berührt haben, jeweils mit dem Diff; im zweiten Eintrag sieht man den Fehler und im dritten die Korrektur. `git diff HEAD~3 HEAD --stat` zeigt die Gesamtänderung der letzten drei Commits. `git show HEAD~2 --stat` betrachtet den Fehlerbehebungs-Commit isoliert. Der letzte Befehl vergleicht den ersten Stand von `rechne.sh` mit dem aktuellen. Aus einer Historie wie dieser lässt sich später jeder Schritt einzeln nachvollziehen, und der fehlerhafte Zwischenstand ist ehrlich dokumentiert, statt versteckt.

### Typische Fehler und Stolperfallen.

Falsch ist es, am Ende einer Arbeitssitzung alles in einen Commit „Stand heute“ zu packen. Richtig ist, unterwegs schon klein zu committen; und wenn man doch zu viel gesammelt hat, mit `git add -p` nachträglich aufzuteilen. Ein anderer Fehler ist, die Historie nach dem Muster „alles muss perfekt aussehen“ zu manipulieren, bevor man die Grundlagen sicher beherrscht. Das Umschreiben erlernen Sie in Modul 6; bis dahin ist ein ehrlicher Fehler-Commit mit nachfolgendem Fix besser als eine verfälschte Historie.

### Praxisbezug.

Genau dieser Ablauf (kleine Commits, Typ in der Nachricht, Vergleich über Diff und Log) ist das Fundament für alles, was folgt: Branches (Modul 3), Pull Requests (Modul 4) und Release-Automatisierung (Modul 7 und 9). Wer die lokalen Grundlagen sicher beherrscht, kann sich später auf die Zusammenarbeit konzentrieren.

## Übungen zu Modul 02

**Aufgabe 1.** Nennen Sie die vier Zustände einer Datei aus Sicht von Git und beschreiben Sie, durch welchen Befehl eine Datei vom einen in den nächsten Zustand wechselt.

**Aufgabe 2.** Was zeigen `git diff`, `git diff --staged` und `git diff HEAD`? Welcher Befehl liefert die Summe der beiden anderen?

**Aufgabe 3.** Erklären Sie, warum `git commit --amend` den Commit nicht wirklich ändert und wann man ihn nicht verwenden sollte.

**Aufgabe 4.** Welche Ignore-Muster treffen die folgenden Dateien: `build/out.o`, `docs/a.log`, `secret.txt`, `sub/secret.txt`, bei den Regeln `*.log`, `build/`, `/secret.txt`?

**Aufgabe 5.** Praktisch: Legen Sie ein Repository an, committen Sie eine Datei `.env`, tragen Sie sie danach in `.gitignore` ein und zeigen Sie, dass die Regel zunächst nicht wirkt. Beheben Sie das, ohne die Datei von der Platte zu löschen.

**Aufgabe 6.** Praktisch: Erzeugen Sie eine Datei mit 40 Zeilen, ändern Sie Zeile 3 und Zeile 38, und committen Sie die beiden Änderungen als zwei getrennte Commits mit `git add -p`.

**Aufgabe 7.** Praktisch: Benennen Sie eine Datei um und ändern Sie ihren Inhalt in einem zweiten Commit. Zeigen Sie mit `git log --follow`, dass die Historie erhalten bleibt, und erklären Sie, warum Sie beides getrennt haben.

**Aufgabe 8.** Praktisch: Formulieren Sie für eine Änderung (Sie denken sich die Änderung aus) eine Commit-Nachricht nach Conventional Commits mit Bereich, Erklärungsabsatz und Ticket-Trailer. Prüfen Sie mit `git log -1` und `git interpret-trailers --parse`.

## Lösungen zu Modul 02

**Lösung 1.** Untracked (`git add` macht sie zu staged), unmodified (Änderung macht sie modified), modified (`git add` macht sie staged), staged (`git commit` macht sie unmodified). Die Zustände ergeben sich aus dem Vergleich von Working Tree, Index und letztem Commit; Git speichert sie nicht.

**Lösung 2.** `git diff`: Working Tree gegen Index (noch nicht Vorgemerktes). `git diff --staged`: Index gegen letzten Commit (was der nächste Commit enthält). `git diff HEAD`: Working Tree gegen letzten Commit, also die Summe der beiden.

**Lösung 3.** Commits sind unveränderlich, ihr Hash ergibt sich aus dem Inhalt. `--amend` erzeugt einen neuen Commit mit neuem Hash und setzt den Zweig darauf um. Nicht anwenden bei bereits veröffentlichten Commits, weil sonst abweichende Historien entstehen und ein Force-Push nötig wird.

**Lösung 4.** `build/out.o`: ja (Regel `build/`). `docs/a.log`: ja (`*.log` gilt auf jeder Ebene). `secret.txt` im Wurzelverzeichnis: ja (`/secret.txt`). `sub/secret.txt`: nein, denn der führende Schrägstrich verankert das Muster am Wurzelverzeichnis.

**Lösung 5.** Ablauf: `echo x > .env; git add .env; git commit -m a; echo .env >> .gitignore; git status -s` zeigt `.env` weiterhin als verfolgt (keine Wirkung der Regel). Lösung: `git rm --cached .env`, dann committen. Die Datei bleibt auf der Platte, die Regel greift. Der Inhalt bleibt in der Historie, weshalb ein echtes Geheimnis zusätzlich rotiert werden müsste.

**Lösung 6.** `seq 1 40 > z.txt`, committen, dann `sed -i '3s/.*/drei/' z.txt` und `sed -i '38s/.*/achtunddreissig/' z.txt`. `git add -p z.txt`: ersten Hunk `y`, zweiten `n`; committen; dann die zweite Änderung normal vormerken und committen.

**Lösung 7.** `git mv a.txt b.txt`, committen; danach Inhalt ändern, committen. `git log --follow --oneline -- b.txt` zeigt beide Commits und die früheren von `a.txt`. Getrennte Commits halten die Ähnlichkeit hoch, sodass Git die Umbenennung zuverlässig erkennt.

**Lösung 8.** Beispiel: `git commit -m "fix(export): CSV-Trennzeichen korrigiert" -m "Excel erwartet in deutschen Systemen das Semikolon.\n\nRefs: TICKET-42"`. `git log -1` zeigt die Nachricht, `git interpret-trailers --parse` gibt `Refs: TICKET-42` aus. Wichtig ist die Leerzeile vor dem Trailer.
