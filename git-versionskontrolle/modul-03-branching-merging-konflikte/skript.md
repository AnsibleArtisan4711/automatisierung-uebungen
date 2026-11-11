# Modul 03 - Branching, Merging und Konfliktloesung

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (3.1 bis 3.9), danach folgen „Übungen zu Modul 03“ (9 Aufgaben) und „Lösungen zu Modul 03“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 12 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 12 h, die Summe der Lerneinheiten ergibt 9 h. Dieses Skript folgt dem höheren Wert und vertieft deshalb Konflikte und Entscheidungshilfen ausführlicher.

**Lernziele des Moduls:** Branches, Merge-Strategien und Konflikte professionell beherrschen; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Branching ist die Funktion, die Git von seinen Vorgängern abhebt. Wir lernen, was ein Branch technisch ist, wie man Arbeitslinien zusammenführt, wie man mit Konflikten umgeht und wie man zwischen Merge und Rebase entscheidet.

| **Nr.** | **Lerneinheit**       | **Dauer (h)** | **Prio** |
|:--------|:----------------------|--------------:|:--------:|
| 3.1     | Branch-Grundlagen     |           1,0 |    1     |
| 3.2     | Branching-Strategien  |           1,0 |    1     |
| 3.3     | Merge-Grundlagen      |           1,0 |    1     |
| 3.4     | Merge-Konflikte       |           1,0 |    1     |
| 3.5     | Rebase-Grundlagen     |           1,0 |    1     |
| 3.6     | Cherry-pick           |           1,0 |    1     |
| 3.7     | Branch Cleanup        |           1,0 |    2     |
| 3.8     | Strategieentscheidung |           1,0 |    2     |
| 3.9     | Konfliktpraxis        |           1,0 |    2     |
|         | **Summe**             |       **9,0** |          |

## 3.1 Branch-Grundlagen

Ein *Branch* (Zweig) ist eine bewegliche Referenz auf einen Commit. Mehr ist er nicht. In Modul 01 haben wir gesehen, dass ein Branch technisch nur eine kleine Datei unter `.git/refs/heads` ist, die den Hash eines Commits enthält. Wenn man einen neuen Commit anlegt, wandert der aktuelle Branch automatisch zu diesem neuen Commit weiter, weil Git den Hash in der Datei aktualisiert. Welcher Branch gerade „aktuell“ ist, steht in `HEAD`, das normalerweise symbolisch auf einen Branch zeigt. Weil ein Branch nur eine 41 Byte kleine Datei ist, kostet das Anlegen praktisch nichts, und das ist der Grund, warum man in Git großzügig Branches nutzt, während in älteren Systemen ein Branch eine teure Kopie des gesamten Projekts war.

Wozu braucht man Branches? Sie erlauben *parallele Arbeitslinien*. Eine Person entwickelt eine neue Funktion, eine andere behebt einen Fehler, eine dritte bereitet ein Release vor, und keiner stört den anderen, weil jede Linie ihre eigenen Commits sammelt. Erst wenn die Arbeit fertig ist, führt man sie in die Hauptlinie zurück (Merge, siehe Abschnitt 3.3). Man unterscheidet üblicherweise *Feature-Branches* für neue Funktionen, *Bugfix-Branches* für Fehlerbehebungen, *Release-Branches* zur Stabilisierung einer Version und *Hotfix-Branches* für dringende Korrekturen an der Produktivversion. Das sind Konventionen, keine technischen Typen; Git kennt nur Branches.

Die moderne Bedienung nutzt zwei Befehle. `git switch <branch>` wechselt zu einem bestehenden Branch, `git switch -c <branch>` legt einen neuen an und wechselt gleich hinein. `git branch` listet Branches auf. Der ältere Befehl `git checkout` kann all das auch (und mehr), wurde aber in die spezialisierten Befehle `switch` und `restore` aufgeteilt, weil er zu viele Aufgaben vermischte. Laut den Plänen für Git 3.0 (abgerufen am 28.10.2026) bleibt `git checkout` bestehen, alle drei Befehle sollen erhalten bleiben; Sie werden also in vielen Anleitungen noch `checkout` sehen.

Das folgende Beispiel legt ein Repository an, erzeugt einen Feature-Branch, committet darauf und zeigt, wie sich der Verlauf auseinanderentwickelt.

```bash
git init -b main br && cd br
echo "Start" > app.txt
git add . && git commit -m "Start"

git switch -c feature/login
echo "Login-Maske" >> app.txt
git commit -am "Login-Maske hinzugefuegt"

git switch main
echo "Impressum" > impressum.txt
git add . && git commit -m "Impressum hinzugefuegt"

git branch -vv
git log --oneline --graph --all
cat .git/refs/heads/feature/login
```

Nach der Zeile `git switch -c feature/login` zeigt `HEAD` auf den neuen Branch, der anfangs auf denselben Commit wie `main` zeigt. Der erste Commit auf dem Feature-Branch bewegt nur diesen. Der Wechsel zu `main` stellt im Arbeitsverzeichnis den Stand von `main` her; die Zeile „Login-Maske“ verschwindet aus `app.txt`, weil sie nur auf dem anderen Branch existiert. Der Commit auf `main` lässt die beiden Linien auseinanderlaufen. `git branch -vv` listet die Branches mit ihrem letzten Commit, `git log --graph --all` zeigt die Verzweigung als Grafik. Die letzte Zeile beweist, dass der Branch eine Datei mit einem Hash ist.

> **Merke**
>
> Ein Branch ist ein Zeiger, kein Ordner und keine Kopie. Beim Wechsel ändert Git das Arbeitsverzeichnis passend zum Zielstand. Ungespeicherte Änderungen werden mitgenommen, solange sie nicht mit dem Zielstand kollidieren; sonst verweigert Git den Wechsel.

Ein Spezialfall ist der *Detached HEAD*. Wenn Sie einen Commit direkt auswählen (`git switch --detach <hash>` oder `git checkout <hash>`), zeigt HEAD nicht mehr auf einen Branch, sondern auf einen Commit. Das ist nützlich zum Ansehen oder Testen eines alten Stands. Neue Commits in diesem Zustand gehören aber keinem Branch an und gehen bei einem späteren Wechsel verloren, wenn man sie nicht mit `git switch -c <name>` sichert.

### Typische Fehler und Stolperfallen.

Der häufigste Fehler ist, auf dem falschen Branch zu arbeiten, typischerweise direkt auf `main`. Falsch: Mit einer neuen Aufgabe loszuarbeiten, ohne vorher `git branch --show-current` zu prüfen. Richtig: Vor Beginn einen Branch anlegen; wenn man bereits auf `main` Änderungen gemacht hat, ist `git switch -c neuer-branch` der einfache Ausweg, denn die ungespeicherten Änderungen wandern mit. Ein zweiter Fehler ist das Verwechseln von `git branch name` (legt an, wechselt aber nicht) mit `git switch -c name` (legt an und wechselt).

### Praxisbezug.

In jedem Teamprojekt beginnt jede Aufgabe mit einem Branch, meist benannt nach dem Ticket. Plattformen wie GitHub und GitLab bieten dafür „Branch aus Ticket erstellen“ an. Die Gewohnheit, nie direkt auf dem Hauptbranch zu arbeiten, wird in Teams oft durch Branch Protection technisch erzwungen (Modul 5).

## 3.2 Branching-Strategien

Weil Branches so günstig sind, stellt sich die Frage, wie man sie sinnvoll einsetzt. Eine *Branching-Strategie* ist die Absprache im Team, welche Branches es gibt, wie sie heißen, wie lange sie leben und wohin sie zurückgeführt werden. Die ausführliche Behandlung von Trunk-Based Development, GitFlow und Release-Strategien folgt in Modul 5; hier legen wir die Grundlagen, die für jede Strategie gelten.

Die wichtigste Grundregel lautet: *kurzlebige Branches bevorzugen*. Je länger ein Branch von der Hauptlinie getrennt lebt, desto stärker laufen die Stände auseinander, und desto schmerzhafter wird später die Zusammenführung, weil sich Konflikte ansammeln. Ein Branch, der nach zwei Tagen zurückgeführt wird, ist unkompliziert; einer, der nach drei Monaten kommt, ist ein Projekt für sich. Kurzlebig heißt dabei nicht klein im Sinn von unwichtig, sondern in kleine, in sich abgeschlossene Schritte zerlegt. Große Funktionen werden in mehreren Branches nacheinander geliefert, notfalls mit sogenannten Feature-Flags, die unfertige Funktionen im Produktivbetrieb unsichtbar halten.

Die zweite Regel betrifft *Namen*. Einheitliche Branch-Namen machen die Liste lesbar und erlauben automatische Regeln. Verbreitet ist das Schema `typ/ticket-kurzbeschreibung`, zum Beispiel `feature/PROJ-123-login-maske`, `bugfix/PROJ-456-leeres-passwort` oder `hotfix/2.3.1`. Der Schrägstrich ist erlaubt und wirkt wie ein Ordner in der Anzeige; Leerzeichen, Umlaute und Sonderzeichen sollte man vermeiden. Ein Team, das so benennt, kann Branches mit `git branch --list 'feature/*'` gruppiert anzeigen, in der Pipeline nach Präfix reagieren und Branches leichter aufräumen.

Das Beispiel legt mehrere Branches nach dem Schema an, zeigt die Gruppierung und prüft, ob ein Name gültig ist. Der letzte Befehl `git check-ref-format` ist die Referenzprüfung von Git selbst und eignet sich für Skripte und Hooks.

```bash
git init -b main strat && cd strat
git commit --allow-empty -m "Start"

git switch -c feature/PROJ-123-login-maske
git switch main
git switch -c bugfix/PROJ-456-leeres-passwort
git switch main
git switch -c hotfix/2.3.1
git switch main

git branch --list 'feature/*'
git branch --list
git for-each-ref --format='%(refname:short)' refs/heads/

git check-ref-format --branch "feature/PROJ-1-ok" && echo gueltig
git check-ref-format --branch "feature/mit leerzeichen" || echo ungueltig
```

Die ersten Zeilen legen drei Branches an; nach jedem kehrt das Skript zu `main` zurück, damit die neuen Branches alle vom selben Punkt abzweigen. `git branch --list 'feature/*'` filtert nach Muster. `git for-each-ref` ist die skriptfreundliche Variante: Sie gibt die Namen ohne Dekoration aus. Die beiden letzten Zeilen prüfen Namen: Der erste ist gültig, der zweite nicht, weil Leerzeichen in Referenznamen verboten sind. Gleiches gilt für Zeichen wie `~`, `^`, `:` und die Zeichenfolge `..`.

### Typische Fehler und Stolperfallen.

Ein verbreitetes Muster ist der *ewige Feature-Branch*: Er beginnt klein, wird nie zurückgeführt, und nach Wochen traut sich niemand mehr, ihn zu mergen. Falsch ist, einen Branch als Sammelplatz für alles zu verwenden, was „irgendwann fertig wird“; richtig ist, Funktionen aufzuteilen und häufig in die Hauptlinie zu integrieren. Ein zweiter Fehler ist Namens-Wildwuchs: `test`, `neu2`, `max-fix`. Solche Namen verraten weder Zweck noch Besitzer noch Ticket, und niemand weiß später, ob man sie löschen darf.

### Praxisbezug.

Viele Pipelines lösen abhängig vom Branch-Namen unterschiedliche Aktionen aus: `main` baut und liefert aus, `feature/*` baut und testet nur, `release/*` erzeugt Release-Kandidaten. Das funktioniert nur zuverlässig bei konsequenter Benennung. Die Konventionen werden oft durch Plattformregeln erzwungen, die Pushes auf falsch benannte Branches ablehnen.

## 3.3 Merge-Grundlagen

Ein *Merge* führt die Arbeit eines Branches in einen anderen zusammen, in aller Regel in den, auf dem man gerade steht. Der Befehl lautet `git merge <branch>`. Wie Git das erledigt, hängt von der Ausgangslage ab. Der einfachste Fall ist der *Fast-forward*: Hat der Zielbranch seit der Abzweigung keine neuen Commits bekommen, liegt der andere Branch einfach „vor“ ihm in einer geraden Linie. Dann schiebt Git den Zielbranch nur nach vorn auf denselben Commit; es entsteht kein neuer Commit, und die Historie bleibt linear.

Haben sich dagegen beide Branches weiterentwickelt, ist ein *Three-way-Merge* nötig. Git sucht den *Merge-Base*, den letzten gemeinsamen Vorfahren, und betrachtet drei Stände: die Spitze des eigenen Branches, die Spitze des fremden Branches und den Merge-Base. Für jede Datei wird verglichen, was sich seit dem Merge-Base auf der einen und auf der anderen Seite geändert hat. Änderungen, die nur eine Seite betreffen, übernimmt Git automatisch; betreffen sie verschiedene Stellen, kombiniert es sie. Das Ergebnis wird als neuer *Merge-Commit* gespeichert, der zwei Eltern hat und damit beide Linien in der Historie verbindet. Der Standardalgorithmus heißt *ort* und ist seit einiger Zeit der Default; `recursive` ist ein Synonym dafür (Git-Dokumentation zu `git merge`, abgerufen am 28.10.2026).

Man kann das Verhalten steuern. Mit `--no-ff` erzwingt man einen Merge-Commit, auch wenn ein Fast-forward möglich wäre. Das hält fest, dass eine Funktion als Einheit existierte, und erlaubt später, sie als Ganzes zurückzunehmen. Mit `--ff-only` lehnt Git den Merge ab, wenn er kein Fast-forward sein kann; das ist eine Sicherung gegen unbeabsichtigte Merge-Commits. Und `--squash` fasst alle Commits des anderen Branches zu einem einzigen Satz von Änderungen im Arbeitsverzeichnis zusammen, den man selbst committet; der Verlauf des Feature-Branches geht dabei nicht in die Hauptlinie ein.

Das folgende Beispiel führt alle drei Szenarien nacheinander vor: zuerst ein Fast-forward, dann einen echten Merge mit Merge-Commit, schließlich einen Squash.

```bash
git init -b main mg && cd mg
echo "Basis" > a.txt && git add . && git commit -m "Basis"

# 1) Fast-forward
git switch -c feature/ff
echo "ff-Zeile" >> a.txt && git commit -am "ff: Zeile ergaenzt"
git switch main
git merge feature/ff
git log --oneline --graph

# 2) Three-way-Merge
git switch -c feature/b
echo "B" > b.txt && git add . && git commit -m "b.txt angelegt"
git switch main
echo "C" > c.txt && git add . && git commit -m "c.txt angelegt"
git merge feature/b -m "Merge feature/b"
git log --oneline --graph
git merge-base main feature/b

# 3) Squash
git switch -c feature/s
echo "S1" > s.txt && git add . && git commit -m "s1"
echo "S2" >> s.txt && git commit -am "s2"
git switch main
git merge --squash feature/s
git commit -m "Feature s als ein Commit uebernommen"
git log --oneline --graph
```

Im ersten Fall meldet Git „Fast-forward“; der Graph bleibt eine gerade Linie. Im zweiten haben beide Branches Commits, deshalb entsteht ein Merge-Commit mit zwei Eltern, und der Graph zeigt eine Raute. `git merge-base main feature/b` gibt den Commit aus, bei dem sich die Linien getrennt haben, hier den Commit des Fast-forward. Im dritten Fall bringt `--squash` die Änderungen beider Commits `s1` und `s2` als vorgemerkte Änderung ins Arbeitsverzeichnis; erst der manuelle Commit legt sie an. Im Graphen steht danach *ein* Commit, und der Feature-Branch ist nicht als Elternteil verbunden. Das hat eine Folge: Git weiß nicht, dass `feature/s` bereits übernommen wurde, und `git branch --merged` führt ihn deshalb nicht auf (siehe Abschnitt 3.7).

### Typische Fehler und Stolperfallen.

Falsch ist es, Merges blind auszuführen, ohne vorher zu wissen, welchen Branch man gerade ausgecheckt hat. Der Merge geht immer in den *aktuellen* Branch; wer versehentlich auf dem Feature-Branch steht und `git merge main` ausführt, bringt `main` in den Feature-Branch, was zwar oft sinnvoll ist, aber eben das Gegenteil des Gedachten. Richtig ist, vor dem Merge `git status -sb` zu lesen. Ein zweiter, häufiger Fehler: `git merge` mit uncommitteten Änderungen. Git kann das erlauben, aber bei Konflikten wird die Lage unübersichtlich; deshalb vorher committen oder `git stash` nutzen.

### Praxisbezug.

Auf Plattformen entspricht der Button „Merge pull request“ einem Merge mit Merge-Commit, „Squash and merge“ dem Squash, und „Rebase and merge“ einem Rebase mit Fast-forward. Welche Variante ein Team einstellt, bestimmt die Gestalt der Historie auf Jahre. Die Abwägung folgt in Abschnitt 3.8.

## 3.4 Merge-Konflikte

Ein *Merge-Konflikt* entsteht, wenn beide Seiten dieselbe Stelle einer Datei verschieden geändert haben und Git nicht entscheiden kann, welche Fassung die richtige ist. Das ist kein Fehler, sondern die ehrliche Antwort auf eine Frage, die nur Menschen beantworten können, denn nur sie kennen die Absicht hinter den Änderungen. Konflikte entstehen auch, wenn eine Seite eine Datei ändert, die die andere gelöscht hat, oder wenn beide Seiten dieselbe Datei umbenennen. Der Merge hält dann an, schreibt in die betroffenen Dateien *Konfliktmarker* und wartet darauf, dass man den Konflikt löst.

Die Marker sehen so aus: Zwischen `<<<<<<<` und `=======` steht die Fassung des eigenen Branches (*ours*), zwischen `=======` und `>>>>>>>` die des fremden Branches (*theirs*). Mit der Einstellung `merge.conflictStyle` auf `diff3` oder `zdiff3` kommt ein dritter Block hinzu, der den Text im Merge-Base zeigt, getrennt durch `|||||||`. Dieser Ausgangstext ist oft der Schlüssel zum Verständnis: Er zeigt, was ursprünglich dastand, und damit, was die beiden Seiten daran geändert haben. Die Git-Dokumentation empfiehlt `zdiff3` für die meisten Nutzer, da die Konfliktbereiche kleiner werden (abgerufen am 28.10.2026).

Das Lösen folgt einem festen Ablauf: Datei öffnen, Marker finden, entscheiden, was am Ende dastehen soll (eine Seite, die andere oder eine Kombination), die Marker entfernen, die Datei mit `git add` als gelöst markieren, und am Ende den Merge mit `git commit` abschließen. Wer sich verrannt hat, bricht mit `git merge --abort` ab und steht wieder beim Stand vor dem Merge. Das folgende Beispiel erzeugt einen Konflikt absichtlich, zeigt ihn und löst ihn.

```bash
git init -b main kf && cd kf
git config merge.conflictStyle zdiff3
printf "Titel: Alt\nPreis: 10\n" > angebot.txt
git add . && git commit -m "Angebot angelegt"

git switch -c feature/preis
sed -i 's/Preis: 10/Preis: 12/' angebot.txt
git commit -am "Preis auf 12 erhoeht"

git switch main
sed -i 's/Preis: 10/Preis: 9/' angebot.txt
git commit -am "Preis auf 9 gesenkt"

git merge feature/preis
git status -sb
cat angebot.txt
```

Der Merge bricht mit „CONFLICT (content)“ ab. `git status` zeigt `UU angebot.txt` („both modified“) und das Repository im Zustand „merging“. Die Datei enthält jetzt (Anordnung bei `zdiff3`) unter anderem folgenden Inhalt:

```text
Titel: Alt
<<<<<<< HEAD
Preis: 9
||||||| 0b2f3a1
Preis: 10
=======
Preis: 12
>>>>>>> feature/preis
```

Hier steht oben die eigene Fassung (`HEAD`, Preis 9), in der Mitte der Ausgangstext (Preis 10) und unten die fremde Fassung (Preis 12). Die Zeile `Titel: Alt` bleibt außerhalb des Konflikts, weil sie auf beiden Seiten gleich ist; das ist der Effekt von `zdiff3`. Die Entscheidung trifft der Mensch, zum Beispiel nach Rückfrage, welcher Preis gelten soll. Angenommen, es gilt 11. Dann wird die Datei von Hand bearbeitet und der Merge abgeschlossen:

```bash
printf "Titel: Alt\nPreis: 11\n" > angebot.txt
git add angebot.txt
git commit -m "Merge feature/preis: Preis auf 11 festgelegt"
git log --oneline --graph
```

Der Befehl `git add` ist hier kein „Hinzufügen“, sondern die Meldung „gelöst“. Beim abschließenden `git commit` öffnet Git ohne `-m` einen Editor mit einer vorbereiteten Merge-Nachricht, die auch die Liste der Konfliktdateien enthält. Wenn eine Seite komplett gewinnen soll, helfen `git checkout --ours -- <datei>` und `git checkout --theirs -- <datei>`, die eine Fassung vollständig übernehmen, gefolgt von `git add`. Für größere Konflikte gibt es grafische Merge-Werkzeuge: `git mergetool` startet das konfigurierte Programm mit drei Ansichten (eigene, fremde, Basis) und einem Ergebnisfenster.

> **Ergänzung aus der Nachrecherche**
>
> **Erinnern an frühere Lösungen: rerere.** Wer wiederholt denselben Konflikt lösen muss, etwa beim Rebase eines langlebigen Branches, kann mit `git config --global rerere.enabled true` das Feature *reuse recorded resolution* aktivieren: Git merkt sich, wie ein bestimmter Konflikt gelöst wurde, und wendet die Lösung bei identischem Konflikt automatisch erneut an. Das ist ein praktisches Werkzeug, aber keine Garantie für Korrektheit; prüfen Sie jedes Ergebnis.

### Typische Fehler und Stolperfallen.

Der gefährlichste Fehler ist, Konfliktmarker im Code zu *vergessen*. Wer nur `git add` ruft, ohne die Datei zu bereinigen, committet die Zeilen `<<<<<<<` samt Inhalt, und der Code ist kaputt. Falsch: `git add .` nach dem Merge ohne Kontrolle. Richtig: vor dem Commit nach Markern suchen, etwa mit `git diff --check`, das unaufgelöste Konfliktmarker meldet, und anschließend die Tests laufen lassen. Ein zweiter Fehler ist, bei Unsicherheit blind `--theirs` oder `--ours` zu wählen: Dann gehen die Änderungen der anderen Seite stillschweigend verloren. Und ein dritter: Konflikte „lösen“, ohne die Absicht der anderen Seite zu verstehen. Im Zweifel fragt man die Person, die die Änderung gemacht hat.

### Praxisbezug.

Konflikte sind ein normaler Teil der Teamarbeit und kein Zeichen eines Fehlers. Die beste Vorbeugung ist, häufig zu integrieren, kleine Änderungen zu machen und in Dateien mit hoher Änderungsrate (etwa Konfigurationen, Sperrdateien wie `package-lock.json`) klare Absprachen zu treffen. Bei Sperrdateien löst man Konflikte meist nicht von Hand, sondern führt den Paketmanager erneut aus, um die Datei neu zu erzeugen.

> **Praxisbeispiel**
>
> Kurz vor einem Release ändern zwei Teams dieselbe Abhängigkeitsliste, und beim Merge steht ein Konflikt in der Sperrdatei `package-lock.json`. Wer ihn zeilenweise von Hand auflöst, erzeugt leicht eine inkonsistente Datei. Der erprobte Weg: eine Seite der Datei übernehmen, die gemeinsame Abhängigkeitsliste in `package.json` zusammenführen und die Sperrdatei mit dem Paketmanager neu erzeugen lassen. Danach laufen die Tests, bevor der Merge abgeschlossen wird.

## 3.5 Rebase-Grundlagen

*Rebase* ist die zweite Möglichkeit, Arbeit aus zwei Branches zu vereinen. Wörtlich bedeutet es „die Basis neu setzen“: Statt die beiden Linien durch einen Merge-Commit zu verbinden, nimmt Git die Commits des eigenen Branches, spielt sie nacheinander auf der Spitze eines anderen Branches neu ab und lässt den eigenen Branch am Ende auf dem letzten dieser neuen Commits stehen. Die Git-Dokumentation beschreibt es als Ablauf: Commits ermitteln, die im Ziel nicht enthalten sind, auf den Zielstand wechseln, sie nacheinander wie mit `git cherry-pick` anwenden und den Branch dann dorthin setzen (abgerufen am 28.10.2026). Das Ergebnis ist eine lineare Historie, in der es aussieht, als hätte man die Arbeit von Anfang an auf dem neuesten Stand begonnen.

Entscheidend ist ein Detail: Die neu abgespielten Commits sind *neue Commits*. Sie haben neue Eltern und daher neue Hashes, auch wenn Inhalt und Nachricht gleich bleiben. Rebase *schreibt Historie um*. Daraus folgt die bekannteste Regel der Git-Welt: Rebasen Sie niemals Commits, die andere bereits übernommen haben. Wer einen veröffentlichten Branch rebased, erzeugt zu den alten Commits parallele neue, und alle, die auf den alten aufgebaut haben, müssen ihre Arbeit mühsam umhängen. Auf Branches, die nur man selbst nutzt und die noch nicht geteilt wurden, ist Rebase dagegen ein sauberes Werkzeug.

Der typische Einsatz: Der Feature-Branch soll vor dem Zusammenführen auf den aktuellen Stand von `main` gebracht werden. Das folgende Beispiel zeigt Merge und Rebase an derselben Ausgangslage nebeneinander. So wird der Unterschied in der Historie sichtbar.

```bash
git init -b main rb && cd rb
echo "Basis" > a.txt && git add . && git commit -m "Basis"

git switch -c feature/x
echo "X1" > x1.txt && git add . && git commit -m "x1"
echo "X2" > x2.txt && git add . && git commit -m "x2"

git switch main
echo "M1" > m1.txt && git add . && git commit -m "m1 auf main"

git switch feature/x
git log --oneline --graph --all
git rebase main
git log --oneline --graph --all

git switch main
git merge feature/x
git log --oneline --graph
```

Vor dem Rebase zeigt der Graph die Verzweigung: `main` hat `m1`, `feature/x` hat `x1` und `x2` über dem gemeinsamen `Basis`. Nach `git rebase main` liegen `x1` und `x2` (jetzt mit anderen Hashes) auf `m1`; die Historie ist eine gerade Linie. Der anschließende Merge in `main` ist ein Fast-forward, weil `main` direkt unter `feature/x` liegt, es entsteht kein Merge-Commit. Hätten wir stattdessen gemergt, wäre die Raute geblieben. Bei Konflikten hält der Rebase bei dem Commit an, der nicht passt. Dann löst man den Konflikt wie beim Merge, markiert mit `git add` und setzt mit `git rebase --continue` fort. `git rebase --abort` bricht ab und stellt den Ausgangszustand her; `git rebase --skip` überspringt den problematischen Commit.

Eine Variante für Sonderfälle ist `git rebase --onto <neueBasis> <alteBasis> <branch>`: Sie verpflanzt einen Ausschnitt der Historie auf eine neue Basis. Das braucht man, wenn ein Branch versehentlich von einem falschen Ausgangspunkt abgezweigt wurde, beispielsweise von einem anderen Feature-Branch statt von `main`. Mit `git pull --rebase` holt man außerdem Änderungen vom Remote und legt eigene, noch nicht veröffentlichte Commits obenauf, statt einen Merge-Commit anzulegen; das wird in Modul 4 relevant.

### Typische Fehler und Stolperfallen.

Falsch ist, einen Branch zu rebasen, der bereits gepusht und von anderen genutzt wird. Der anschließend nötige `git push --force` überschreibt die Historie auf dem Server. Richtig ist, nur lokale, ungeteilte Commits umzuschreiben, und bei notwendigem Force-Push die sicherere Form `git push --force-with-lease` zu verwenden, die nur überschreibt, wenn niemand sonst inzwischen etwas dazugelegt hat. Ein zweiter Fehler ist, einen Rebase bei Verwirrung „irgendwie“ zu beenden. Wer mitten im Rebase steckt (der Status zeigt „rebase in progress“), nutzt `--abort` und beginnt von vorn; durch halbherziges Löschen von Dateien in `.git` entsteht Chaos.

### Praxisbezug.

Teams, die Wert auf lineare Historie legen, rebasen Feature-Branches vor dem Merge. Dadurch lässt sich der Verlauf später leichter lesen, und Werkzeuge wie `git bisect` (Modul 6) arbeiten auf einer geraden Linie angenehmer. Andere Teams verzichten auf Rebase, weil sie die Ehrlichkeit der tatsächlichen Verläufe und die Sicherheit vor Umschreiben höher bewerten. Beide Haltungen sind legitim; wichtig ist die Absprache.

## 3.6 Cherry-pick

Mit `git cherry-pick` übernimmt man einzelne Commits von einem Branch auf einen anderen, ohne den ganzen Branch zu mergen. Wörtlich heißt das „Kirschen pflücken“: Man greift sich genau die Änderungen heraus, die man braucht. Technisch ermittelt Git die Änderung, die der gewählte Commit gegenüber seinem Vorgänger eingeführt hat, wendet sie auf den aktuellen Branch an und legt dafür einen *neuen* Commit an. Der neue Commit hat denselben Inhalt der Änderung und dieselbe Nachricht, aber einen anderen Hash und einen anderen Elterncommit.

Der klassische Einsatzfall ist das *Backport*: Ein Fehler wurde auf `main` behoben, muss aber auch in der gewarteten Version `release/2.3` korrigiert werden. Man wechselt auf den Release-Branch und pickt den Fix-Commit. Ein weiterer Fall: Man hat eine Korrektur versehentlich auf dem falschen Branch committet und möchte sie auf den richtigen bringen. Wichtige Optionen: `-x` hängt der Nachricht die Zeile „(cherry picked from commit ...)“ an, was bei öffentlichen Branches die Rückverfolgung ermöglicht (laut Git-Dokumentation sinnvoll bei Backports zwischen öffentlich sichtbaren Branches, abgerufen am 28.10.2026). `-n` (`--no-commit`) wendet die Änderung nur an, ohne zu committen, was nützlich ist, um mehrere Picks in einem Commit zu bündeln. Und für Merge-Commits braucht man `-m 1`, um zu sagen, gegenüber welchem Elternteil die Änderung berechnet werden soll.

Das Beispiel simuliert ein Backport: Ein Fix auf `main` wird auf einen Release-Branch übernommen, und danach wird ein Konflikt beim Pick behandelt.

```bash
git init -b main cp && cd cp
printf "v1\nzeile A\nzeile B\n" > app.txt
git add . && git commit -m "Version 1"
git switch -c release/1.0

git switch main
echo "feature" > feature.txt && git add . && git commit -m "Feature (nicht fuer release)"
sed -i 's/zeile B/zeile B (korrigiert)/' app.txt
git commit -am "Fix: Zeile B korrigiert"
git log --oneline

FIX=$(git rev-parse HEAD)
git switch release/1.0
git cherry-pick -x $FIX
git log --oneline
git show --stat HEAD

git switch main
sed -i 's/zeile A/zeile A (neu)/' app.txt
git commit -am "Zeile A geaendert"
git switch release/1.0
sed -i 's/zeile A/zeile A (alt)/' app.txt
git commit -am "Zeile A im Release geaendert"
git cherry-pick main
git status -sb
git cherry-pick --abort
```

Zunächst liegen auf `main` zwei neue Commits, von denen nur der Fix in den Release soll. `git rev-parse HEAD` merkt sich seinen Hash in einer Variablen. Auf dem Release-Branch übernimmt `git cherry-pick -x` genau diesen Fix; der Log zeigt danach nur den Fix und nicht das Feature, und die Nachricht enthält die Herkunftszeile. Im zweiten Teil ändern beide Branches dieselbe Zeile unterschiedlich. Der Pick bricht deshalb mit einem Konflikt ab; der Status zeigt `UU app.txt`. Man löst ihn wie bei einem Merge und setzt mit `git cherry-pick --continue` fort, oder man verwirft den Versuch mit `git cherry-pick --abort`, wie hier.

### Typische Fehler und Stolperfallen.

Falsch ist es, Cherry-pick als Ersatz für Merge zu verwenden. Wer regelmäßig einzelne Commits zwischen Branches kopiert, erzeugt doppelte Commits mit verschiedenen Hashes. Git erkennt später beim Mergen nicht, dass es sich um dieselbe Änderung handelt, und es kommt zu Konflikten oder Dopplungen. Richtig ist, Cherry-pick gezielt für Backports und Einzelfälle einzusetzen und dokumentiert mit `-x`. Ein weiterer Fehler: Commits zu picken, die von anderen abhängen, etwa einen Commit, der eine Funktion nutzt, die erst ein früherer Commit eingeführt hat. Das Ergebnis kompiliert nicht. Dann muss man die Abhängigkeit mit picken oder eine andere Lösung wählen.

### Praxisbezug.

Bei Produkten mit gepflegten Versionen ist Cherry-pick der Normalweg, Fehlerbehebungen in alte Release-Linien zu bringen. Plattformen bieten dafür Schaltflächen an, die automatisch einen Pull Request mit dem gepickten Commit erzeugen. Nach einem Hotfix ist eine übliche Kontrolle, dass die Korrektur in alle gepflegten Linien gelangt ist; die Herkunftszeile aus `-x` macht diese Prüfung per `git log --grep` einfach.

> **Praxisbeispiel**
>
> Ein Kunde nutzt noch Version 2.3, bei der ein Sicherheitsfehler auftritt, der auf `main` bereits behoben ist. Sie wechseln auf `release/2.3`, picken den Fix-Commit mit `-x`, lassen die Tests laufen und veröffentlichen `2.3.1`. Die Herkunftszeile in der Commit-Nachricht erlaubt später den Nachweis, dass der Fix in allen gepflegten Linien angekommen ist.

## 3.7 Branch Cleanup

Branches, die ihren Zweck erfüllt haben, sollten verschwinden. Sonst wächst die Liste ins Unübersichtliche, und niemand weiß mehr, welche Branches noch leben. Das Löschen ist harmlos, weil ein Branch nur ein Zeiger ist: Der Branch verschwindet, die Commits bleiben erhalten, solange sie von anderen Referenzen erreichbar sind. Ist ein Branch bereits gemergt, sind alle seine Commits in der Hauptlinie enthalten, und nichts geht verloren. Ist er nicht gemergt, kämen die Commits ohne Branch in einen Schwebezustand und würden irgendwann von der Speicherbereinigung entfernt. Deshalb schützt Git davor: `git branch -d <name>` verweigert das Löschen eines nicht gemergten Branches, erst `git branch -D <name>` erzwingt es.

Zu unterscheiden sind lokale Branches (auf dem eigenen Rechner), *Remote-Tracking-Branches* (die lokale Kopie vom Stand eines Branches auf dem Server, etwa `origin/feature/x`) und Branches auf dem Remote selbst. Wird ein Branch auf dem Server gelöscht, bleibt der lokale Remote-Tracking-Branch zunächst stehen. Mit `git fetch --prune` räumt man diese veralteten Verweise auf. Einen Branch auf dem Server löscht man mit `git push origin --delete <name>`.

Das folgende Beispiel richtet ein Repository mit einem Remote ein (hier ein lokales Bare-Repository als Ersatz für einen Server), legt einen gemergten und einen ungemergten Branch an und räumt auf.

```bash
git init --bare -b main server.git
git init -b main cl && cd cl
git remote add origin ../server.git
echo "Start" > a.txt && git add . && git commit -m "Start"
git push -u origin main

git switch -c feature/fertig
echo "fertig" > f.txt && git add . && git commit -m "fertig"
git switch main && git merge feature/fertig
git switch -c feature/offen
echo "offen" > o.txt && git add . && git commit -m "offen"
git push -u origin feature/fertig
git switch main

git branch --merged main
git branch --no-merged main

git branch -d feature/fertig
git branch -d feature/offen
git branch -D feature/offen

git push origin --delete feature/fertig
git branch -r
git fetch --prune
```

Die beiden Abfragen `git branch --merged main` und `--no-merged main` teilen die Branches in solche, die vollständig in `main` enthalten sind (löschbar), und solche, die noch eigene Commits haben. `git branch -d feature/fertig` funktioniert, weil der Branch gemergt ist; bei `feature/offen` verweigert Git das Löschen mit dem Hinweis „not fully merged“. Erst `-D` erzwingt es. (Hätten wir `feature/offen` gepusht, würde `-d` funktionieren, weil Git dann gegen den Upstream-Branch prüft; der Schutz gilt also nur für Arbeit, die nirgends sonst existiert.) Die Ausgabe zeigt dabei den Hash des Branches, mit dem man ihn im Notfall wiederherstellen könnte (`git branch <name> <hash>`). Danach wird der Branch auf dem Server gelöscht, `git branch -r` zeigt die Remote-Branches, und `git fetch --prune` entfernt Verweise auf Server-Branches, die es nicht mehr gibt.

### Typische Fehler und Stolperfallen.

Falsch ist, bei der Meldung „not fully merged“ reflexartig `-D` zu verwenden. Die Meldung ist ein Warnsignal: Der Branch enthält Arbeit, die sonst verloren wäre. Richtig ist, erst mit `git log main..<branch>` zu prüfen, welche Commits übrig bleiben. Eine Ausnahme sind squash-gemergte Branches: Weil der Squash-Commit einen anderen Hash hat, hält Git sie für ungemergt, obwohl ihr Inhalt übernommen wurde; hier ist `-D` nach Prüfung legitim. Ein zweiter Fehler: Remote-Branches zu löschen, die andere noch benutzen. Vorher abstimmen, besonders bei geteilten Branches.

### Praxisbezug.

Plattformen können gemergte Branches automatisch löschen; diese Einstellung ist in den meisten Teams sinnvoll. Zusätzlich lohnt sich eine regelmäßige Inventur: Branches, auf denen seit Monaten nichts passiert ist, sind Kandidaten für Rückfrage und Löschung. Wer sich Branch-Alter ansehen will, nutzt `git for-each-ref --sort=committerdate refs/remotes/origin --format='%(committerdate:short) %(refname:short)'`.

## 3.8 Strategieentscheidung

Es gibt drei Wege, die Arbeit eines Branches in die Hauptlinie zu bringen: Merge mit Merge-Commit, Squash-Merge und Rebase mit anschließendem Fast-forward. Keine ist grundsätzlich richtig, jede hat Folgen für Lesbarkeit, Nachvollziehbarkeit und Sicherheit der Historie. Die Entscheidung sollte das Team bewusst treffen und dokumentieren, statt sie dem Zufall oder der Gewohnheit Einzelner zu überlassen.

Der *Merge mit Merge-Commit* bewahrt die tatsächliche Historie vollständig: Jeder Commit bleibt, wie er entstand, und der Merge-Commit markiert, dass und wann eine Arbeitslinie integriert wurde. Das ist ehrlich und sicher, weil nichts umgeschrieben wird. Der Preis ist eine verzweigte Historie mit vielen Merge-Commits, in der sich einzelne Linien schwerer nachverfolgen lassen. Der *Squash-Merge* fasst alle Commits eines Branches zu einem einzigen zusammen. Das ergibt eine schlanke Hauptlinie mit einem Commit pro Funktion, was Reviews und Changelogs vereinfacht. Der Preis: Die Feinstruktur der Arbeit geht verloren, und Branches erscheinen für Git als ungemergt. Der *Rebase mit Fast-forward* schließlich liefert eine lineare Historie, in der jeder Commit einzeln erhalten bleibt. Der Preis ist, dass die Commits beim Rebase neue Hashes bekommen und Rebase auf geteilten Branches gefährlich ist.

|                   | **Merge-Commit**                        | **Squash-Merge**                  | **Rebase + FF**               |
|:------------------|:----------------------------------------|:----------------------------------|:------------------------------|
| Historie          | verzweigt, vollständig                  | linear, ein Commit je Branch      | linear, alle Commits          |
| Umschreiben       | nein                                    | ja (Zusammenfassung)              | ja (neue Hashes)              |
| Rückgängig machen | ein Revert des Merge-Commits            | ein Revert eines Commits          | mehrere Reverts nötig         |
| Bisect            | gut möglich, aber verzweigt             | grob (nur ein Commit je Funktion) | fein und linear               |
| Risiko            | Merge-Commits überfrachten die Historie | Detailverlust                     | Fehler bei geteilten Branches |

Die Tabelle zeigt, dass es keinen Gewinner gibt. Ein vernünftiger Standard für viele Teams ist die Kombination: Feature-Branches werden vor dem Zusammenführen rebased oder mit `main` gemergt, um aktuell zu sein, und die Integration in `main` erfolgt per Squash-Merge (kleine Pull Requests) oder per Merge-Commit (größere, logisch gegliederte Branches). Entscheidend ist, dass *ein* Verfahren gilt. Für das Einstellen gibt es Werkzeuge in der Konfiguration. Das Beispiel zeigt Einstellungen, mit denen man ein gewähltes Verfahren absichert, und eine Möglichkeit, den Ausgang eines Merges vorab zu prüfen.

```bash
# Pull nur als Fast-forward erlauben, sonst Fehler
git config --global pull.ff only

# oder: Pull immer per Rebase
git config --global pull.rebase true

# Merge standardmaessig immer mit Merge-Commit
git config --global merge.ff false

```

Die Einstellung `pull.ff only` erlaubt bei `git pull` nur Fast-forward und meldet sonst einen Fehler, sodass man bewusst entscheiden muss; `pull.rebase true` legt stattdessen lokale Commits auf den geholten Stand. Beide werden meist alternativ gewählt. `merge.ff false` entspricht dauerhaft `--no-ff`.

Für die Merge-Vorschau legen wir zwei Branches an, die dieselbe Zeile ändern, und lassen Git das Ergebnis berechnen, ohne das Arbeitsverzeichnis anzufassen.

```bash
git init -b main vorschau && cd vorschau
echo "eins" > a.txt && git add . && git commit -m "Basis"
git switch -c feature/x
echo "zwei" > a.txt && git commit -am "x"
git switch main
echo "drei" > a.txt && git commit -am "m"

git merge-tree --write-tree main feature/x
echo "Rueckgabewert: $?"
```

Der Befehl `git merge-tree --write-tree` berechnet das Ergebnis eines Merges im Hintergrund und gibt bei Erfolg den Hash des resultierenden Trees aus; bei Konflikten endet er mit dem Rückgabewert 1 und listet die betroffenen Dateien. So kann man in Skripten prüfen, ob ein Merge konfliktfrei möglich wäre, ohne das Arbeitsverzeichnis zu verändern.

### Typische Fehler und Stolperfallen.

Falsch ist, wenn jede Person ihr bevorzugtes Verfahren anwendet. Dann mischt sich eine Historie aus Merge-Commits, Squashes und Rebases, und aus ihr lässt sich weder ein klarer Verlauf lesen noch eine verlässliche Regel für Rückgängigmachung ableiten. Richtig ist eine Teamregel, die technisch erzwungen wird: Auf Plattformen lassen sich die erlaubten Merge-Arten pro Repository einschränken. Ein zweiter Fehler ist, die Strategie nach Gefühl zu wählen: Die Frage „Wie wollen wir später Fehler finden und zurücknehmen?“ führt zu besseren Entscheidungen als die Frage „Was sieht im Graph schöner aus?“.

### Praxisbezug.

Im Projektalltag entscheidet man das einmal bei der Repository-Einrichtung und dokumentiert es in der `CONTRIBUTING.md` (Modul 5). Bei Teams, die Release-Notes aus Commits erzeugen (Modul 7 und 9), ist Squash-Merge mit sauberem Titel besonders beliebt, weil dann jeder Hauptlinien-Commit genau einem Pull Request entspricht.

## 3.9 Konfliktpraxis

Diese letzte Einheit des Moduls trainiert den Umgang mit realistischen Konflikten. Neben dem einfachen Fall „beide haben dieselbe Zeile geändert“ gibt es weitere typische Situationen. Der *Modify/Delete-Konflikt* entsteht, wenn eine Seite eine Datei ändert und die andere sie löscht; hier muss entschieden werden, ob die Datei bleiben soll. Der *Rename-Konflikt* tritt auf, wenn beide Seiten dieselbe Datei verschieden umbenennen oder wenn eine Seite sie umbenennt, während die andere sie ändert (Letzteres löst Git meist selbstständig, indem es die Änderung auf den neuen Namen anwendet). Der *Whitespace-Konflikt* entsteht, wenn eine Seite nur umformatiert hat, und führt zu riesigen Konfliktbereichen, die inhaltlich gar keine sind.

Ein sicherer Ablauf in jeder Lage besteht aus fünf Schritten. Erstens: Vor dem Merge sicherstellen, dass das Arbeitsverzeichnis sauber ist. Zweitens: Merge starten und mit `git status` ermitteln, welche Dateien in welchem Konfliktzustand stehen. Drittens: Jede Datei bewusst lösen, nicht nur mechanisch. Viertens: Mit `git diff --check` und der Testsuite verifizieren, dass keine Marker mehr im Code stehen und das Ergebnis funktioniert. Fünftens: Abschließen und im Zweifel den Merge-Commit mit einem kurzen Hinweis dokumentieren, wie entschieden wurde.

Im folgenden Beispiel werden zwei Konflikte gleichzeitig erzeugt, ein Inhaltskonflikt und ein Modify/Delete-Konflikt. Zusätzlich gibt es einen minimalen Test, der nach der Lösung prüft, ob die Anwendung noch funktioniert.

```bash
git init -b main kp && cd kp
git config merge.conflictStyle zdiff3

cat > preis.sh <<'EOF'
#!/bin/sh
echo $(( 100 * 12 / 10 ))
EOF
cat > test.sh <<'EOF'
#!/bin/sh
[ "$(sh preis.sh)" = "120" ] && echo "TEST OK" || { echo "TEST FEHLER"; exit 1; }
EOF
echo "alte Doku" > doku.txt
git add . && git commit -m "Preisrechner, Test und Doku"
sh test.sh

git switch -c feature/rabatt
sed -i 's/100 \* 12 \/ 10/100 * 12 \/ 10 - 5/' preis.sh
git rm -q doku.txt
git commit -am "Rabatt von 5 eingefuehrt, Doku entfernt"

git switch main
sed -i 's/100 \* 12 \/ 10/100 * 13 \/ 10/' preis.sh
echo "ergaenzte Doku" >> doku.txt
git commit -am "Faktor auf 13 geaendert, Doku ergaenzt"

git merge feature/rabatt
git status -sb
```

Der Merge liefert zwei Konflikte: `UU preis.sh` (beide haben dieselbe Zeile geändert) und `UD doku.txt` („deleted by them“: die andere Seite hat die Datei gelöscht, wir haben sie geändert). Zum Lösen: In `preis.sh` entscheiden wir, dass beides gelten soll, also der neue Faktor und der Rabatt. Bei der Doku entscheiden wir, die Datei zu behalten. Danach wird der Test angepasst und ausgeführt, denn das richtige Ergebnis ist jetzt 100· 13/10 - 5 = 125.

```bash
printf '#!/bin/sh\necho $(( 100 * 13 / 10 - 5 ))\n' > preis.sh
git add preis.sh
git add doku.txt
sed -i 's/= "120"/= "125"/' test.sh
sh test.sh
git diff --check
git commit -am "Merge feature/rabatt: Faktor 13 mit Rabatt 5, Doku behalten"
git log --oneline --graph
```

Bei `doku.txt` bedeutet `git add doku.txt`, dass wir die Datei in unserer Fassung behalten; `git rm doku.txt` hätte die Löschung akzeptiert. Der Test gibt `TEST OK` aus, und erst danach schließt der Merge-Commit ab. Der Test zeigt einen wichtigen Punkt: Ein Konflikt kann *ohne* Marker eine inhaltliche Fehlentscheidung enthalten. Hätten wir nur `--theirs` gewählt, wäre der Faktor 13 verloren gegangen, und der Test hätte 115 statt 125 geliefert und damit den Fehler aufgedeckt.

### Typische Fehler und Stolperfallen.

Falsch ist es, nach dem Lösen „es kompiliert, also ist es richtig“ zu denken. Es gibt *semantische Konflikte*, die Git nicht erkennt: Zwei Branches ändern unterschiedliche Dateien konfliktfrei, aber das Ergebnis passt nicht zusammen, etwa weil ein Branch eine Funktion umbenennt, die der andere neu aufruft. Richtig ist, nach jedem Merge die Tests auszuführen, auch wenn Git keinen Konflikt meldet. Ein weiterer Fehler ist, große Formatierungsänderungen mit inhaltlichen zu mischen; die Whitespace-Konflikte, die dabei entstehen, lassen sich mit `git merge -X ignore-space-change` mildern, aber besser vermeidet man sie durch getrennte Commits.

### Praxisbezug.

Teams, die Pull Requests nutzen, lösen Konflikte meist, indem sie `main` in den Feature-Branch mergen oder diesen rebasen, *bevor* der Pull Request abgeschlossen wird. Die Pipeline läuft dann auf dem konfliktfreien Stand und prüft die Semantik. Dieses Muster, vor der Integration lokal zu integrieren, ist das wichtigste Mittel gegen unangenehme Überraschungen in der Hauptlinie.

## Übungen zu Modul 03

**Aufgabe 1.** Was ist ein Branch technisch, und warum ist das Anlegen eines Branches in Git so günstig?

**Aufgabe 2.** Erklären Sie den Unterschied zwischen Fast-forward und Three-way-Merge. In welcher Ausgangslage ist ein Fast-forward möglich?

**Aufgabe 3.** Was bewirkt `git merge --no-ff`, und warum kann das nützlich sein? Was bewirkt `--squash`, und welche Folge hat das für `git branch --merged`?

**Aufgabe 4.** Nennen Sie zwei Gründe, warum man kurzlebige Branches bevorzugt, und ein Beispiel für ein sinnvolles Namensschema.

**Aufgabe 5.** Praktisch: Erzeugen Sie ein Repository mit einer Datei, legen Sie zwei Branches an, die dieselbe Zeile unterschiedlich ändern, und lösen Sie den Konflikt beim Merge. Stellen Sie vorher `merge.conflictStyle` auf `zdiff3` und erklären Sie die drei Abschnitte der Marker.

**Aufgabe 6.** Praktisch: Stellen Sie dieselbe Ausgangslage (ein Feature-Branch mit zwei Commits, `main` mit einem weiteren Commit) zweimal her. Integrieren Sie das einmal per Merge und einmal per Rebase und vergleichen Sie die Graphen mit `git log --graph`.

**Aufgabe 7.** Praktisch: Simulieren Sie ein Backport. Beheben Sie auf `main` einen Fehler, bringen Sie den Fix per `cherry-pick -x` auf einen Release-Branch, und weisen Sie mit `git log` nach, dass nur der Fix übernommen wurde. Lösen Sie anschließend einen Cherry-pick-Konflikt.

**Aufgabe 8.** Praktisch: Legen Sie drei Branches an (einen gemergten, einen ungemergten, einen squash-gemergten). Ermitteln Sie mit `git branch --merged` und `--no-merged`, welche löschbar sind, und erklären Sie das Ergebnis beim squash-gemergten Branch.

**Aufgabe 9.** Praktisch (Transfer): Erzeugen Sie einen Modify/Delete-Konflikt und einen semantischen Konflikt (Branch A benennt eine Funktion um, Branch B ruft sie unter dem alten Namen neu auf). Zeigen Sie, dass Git beim zweiten keinen Konflikt meldet, und erklären Sie, wie man ihn trotzdem entdeckt.

## Lösungen zu Modul 03

**Lösung 1.** Ein Branch ist eine Referenz, also eine kleine Datei unter `.git/refs/heads` mit dem Hash eines Commits. Das Anlegen schreibt nur diese Datei, es werden keine Dateien kopiert. Darum ist es praktisch kostenlos.

**Lösung 2.** Fast-forward: Der Zielbranch hat seit der Abzweigung keine neuen Commits, Git schiebt den Zeiger nur nach vorn; es entsteht kein Merge-Commit. Three-way-Merge: Beide Seiten haben neue Commits, Git vergleicht beide Spitzen mit dem Merge-Base und legt einen Merge-Commit mit zwei Eltern an.

**Lösung 3.** `--no-ff` erzwingt einen Merge-Commit auch bei möglichem Fast-forward; die Funktion bleibt als Einheit erkennbar und lässt sich als Ganzes zurücknehmen. `--squash` übernimmt alle Änderungen ungebucht ins Arbeitsverzeichnis, man committet selbst; der Feature-Branch ist dann nicht Elternteil, deshalb führt `git branch --merged` ihn nicht als gemergt auf.

**Lösung 4.** Gründe: weniger Konflikte durch geringe Divergenz, schnellere Reviews, geringeres Integrationsrisiko. Schema: `feature/PROJ-123-kurzbeschreibung`.

**Lösung 5.** Ablauf wie im Abschnitt Merge-Konflikte. Oben (`HEAD`) steht die eigene Fassung, in der Mitte (nach `|||||||`) der Text des Merge-Base, unten (nach `=======`) die fremde Fassung. Man schreibt das Ergebnis, entfernt die Marker, `git add`, `git commit`.

**Lösung 6.** Merge ergibt eine Raute mit Merge-Commit, Rebase eine gerade Linie, auf der die Feature-Commits neue Hashes haben. Nach dem Rebase ist der Merge in `main` ein Fast-forward.

**Lösung 7.** Beispiel: `git switch release/1.0; git cherry-pick -x <hash>`; `git log --oneline` zeigt nur den Fix. Für den Konflikt dieselbe Zeile auf beiden Branches ändern, Pick ausführen, Konflikt lösen, `git add`, `git cherry-pick --continue`.

**Lösung 8.** Gemergte Branches erscheinen unter `--merged` und sind mit `-d` löschbar. Ungemergte erscheinen unter `--no-merged`; `-d` verweigert, `-D` erzwingt. Der squash-gemergte Branch erscheint ebenfalls unter `--no-merged`, weil der Squash-Commit einen anderen Hash hat; nach Prüfung mit `git log main..<branch>` und dem Wissen, dass sein Inhalt übernommen wurde, ist `-D` legitim.

**Lösung 9.** Beim Modify/Delete meldet Git `UD` oder `DU`; Lösung durch `git add` (Datei behalten) oder `git rm` (Löschung akzeptieren). Beim semantischen Konflikt merged Git ohne Meldung, weil verschiedene Stellen betroffen sind. Entdeckt wird er durch Tests oder Build nach dem Merge, deshalb gehört die Pipeline-Prüfung auf den gemergten Stand.
