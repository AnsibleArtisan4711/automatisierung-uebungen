# Modul 01 - Git Grundlagen und Versionskontrollarchitektur

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (1.1 bis 1.8), danach folgen „Übungen zu Modul 01“ (8 Aufgaben) und „Lösungen zu Modul 01“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 8 h, die Summe der Lerneinheiten ergibt 10 h. Dieses Skript folgt dem höheren Wert.

**Lernziele des Moduls:** Versionskontrolle, das Git-Datenmodell und lokale Workflows verstehen; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. In diesem ersten Modul legen wir das Fundament: Wir klären, warum es Versionskontrolle gibt, wie Git intern Daten speichert und wie ein täglicher lokaler Arbeitsablauf aussieht.

| **Nr.** | **Lerneinheit**             | **Dauer (h)** | **Prio** |
|:--------|:----------------------------|--------------:|:--------:|
| 1.1     | Versionskontrolle einordnen |          1,25 |    1     |
| 1.2     | Git-Datenmodell             |          1,25 |    1     |
| 1.3     | Repository-Struktur         |          1,25 |    1     |
| 1.4     | Lokaler Workflow            |          1,25 |    1     |
| 1.5     | Git CLI und Tools           |          1,25 |    1     |
| 1.6     | Konfiguration               |          1,25 |    1     |
| 1.7     | Hilfesystem                 |          1,25 |    2     |
| 1.8     | Erstes Repository           |          1,25 |    2     |
|         | **Summe**                   |      **10,0** |          |

## 1.1 Versionskontrolle einordnen

Jedes Projekt, das länger als einen Nachmittag lebt, steht vor demselben Problem: Dateien ändern sich, und irgendwann will man wissen, was sich wann und warum geändert hat. Ohne Werkzeug entstehen Ordner mit Namen wie `bericht_final`, `bericht_final_neu` und `bericht_final_wirklich_final`. Das funktioniert für eine Person gerade noch, scheitert aber, sobald zwei Personen gleichzeitig an denselben Dateien arbeiten: Wer hat welche Änderung gemacht, welche Fassung ist die aktuelle, und wie bringt man zwei parallel bearbeitete Fassungen zusammen, ohne Arbeit zu verlieren? Ein *Versionskontrollsystem* (englisch *Version Control System*, kurz VCS) löst genau das. Es zeichnet jede Änderung an einer Dateisammlung auf, versieht sie mit Autor, Zeitpunkt und Begründung und erlaubt es, zu jedem früheren Zustand zurückzukehren.

Historisch gibt es zwei große Architekturen. Bei *zentralen* Systemen wie CVS, Subversion (SVN) oder dem Team Foundation Version Control (TFVC) liegt die gesamte Historie auf einem Server. Entwickelnde holen sich nur eine Arbeitskopie und müssen für fast jede Operation, die Historie betrifft, mit dem Server sprechen: für den Log, für Vergleiche, für das Anlegen eines Zweigs. Fällt der Server oder das Netz aus, ist die Arbeit blockiert, und ein Datenverlust am Server betrifft alle. Bei *verteilten* Systemen wie Git, Mercurial oder Bazaar besitzt dagegen jede Arbeitskopie die vollständige Historie. Ein *Klon* (englisch *clone*) ist ein vollwertiges Repository, mit dem man offline committen, Historie durchsuchen und Zweige anlegen kann. Erst zum Austausch mit anderen verbindet man sich mit einem anderen Repository. Dass es meist einen Server wie GitHub oder GitLab gibt, ist eine Konvention des Teams und keine technische Notwendigkeit.

Der Nutzen geht weit über das Zurückspulen hinaus. Für die *Entwicklung* bedeutet Versionskontrolle, dass mehrere Personen parallel an Funktionen arbeiten können, ohne sich zu überschreiben, weil jede Arbeitslinie als eigener Zweig geführt und später zusammengeführt wird. Für den *Betrieb* liefert sie die Antwort auf die Frage, welcher Stand eigentlich in Produktion läuft und was sich seit dem letzten Release geändert hat. Für das *Audit* schließlich ist die Historie ein Nachweis: Wer hat eine Änderung vorgenommen, wer hat sie freigegeben, und zu welchem Ticket gehört sie? In regulierten Umgebungen ist genau diese Nachvollziehbarkeit oft Pflicht.

Ein häufiges Missverständnis sollte man früh ausräumen: Git ist nicht GitHub. Git ist das Programm, das lokal auf dem eigenen Rechner läuft und die Daten verwaltet. GitHub, GitLab, Bitbucket oder Azure Repos sind Dienste, die Git-Repositories hosten und um Funktionen wie Pull Requests, Tickets und Pipelines ergänzen. Man kann Git vollständig ohne diese Dienste nutzen, etwa mit einem Repository auf dem eigenen Rechner oder auf einem Netzlaufwerk.

Das folgende Beispiel zeigt, wie wenig nötig ist, um Versionskontrolle zu erleben. Es legt ein Verzeichnis an, macht es zu einem Git-Repository, speichert zwei Fassungen einer Datei und fragt die Historie ab. Es läuft ohne Netzwerk, ohne Konto und ohne Server.

```bash
mkdir bericht && cd bericht
git init -b main
echo "Entwurf der Einleitung" > bericht.txt
git add bericht.txt
git commit -m "Einleitung als Entwurf angelegt"
echo "Einleitung ueberarbeitet und ergaenzt" > bericht.txt
git commit -am "Einleitung ueberarbeitet"
git log --oneline
git show HEAD~1:bericht.txt
```

Die erste Zeile legt das Verzeichnis an und wechselt hinein. `git init -b main` erzeugt das Repository und nennt den ersten Zweig `main`. Mit `git add` wird die Datei für den nächsten Commit vorgemerkt, `git commit -m` speichert sie dauerhaft mit einer Begründung. Die Option `-a` im zweiten Commit bedeutet, dass alle bereits bekannten, geänderten Dateien automatisch einbezogen werden. `git log --oneline` listet die beiden Fassungen kurz auf. Der letzte Befehl ist der eigentliche Gewinn: `git show HEAD~1:bericht.txt` gibt den Inhalt der Datei aus, wie er einen Commit zuvor war, ohne dass die aktuelle Datei angetastet wird.

> **Merke**
>
> Zentral heißt: eine Historie, ein Server, Zugriff nötig. Verteilt heißt: jede Kopie hat die gesamte Historie. Git ist verteilt, und „verteilt“ beschreibt die Technik, nicht den Arbeitsablauf. Ob ein Team einen zentralen Server als Abgleichspunkt nutzt, ist eine organisatorische Entscheidung.

### Typische Fehler und Stolperfallen.

Der häufigste Fehler ist, Git mit einem Backup- oder Dateisynchronisationsdienst zu verwechseln. Wer sein Repository in einem Cloud-Sync-Ordner ablegt und gleichzeitig auf zwei Rechnern arbeitet, riskiert beschädigte Objekte, weil der Dienst einzelne Dateien in .git zu unterschiedlichen Zeitpunkten synchronisiert. Falsch ist also, das Arbeitsverzeichnis mit Git per Cloud-Ordner zu „teilen“; richtig ist, einen Remote-Server zu verwenden und mit `git push` und `git pull` auszutauschen. Ein zweiter Fehler ist die Gewohnheit, Fassungen weiter über Dateinamen zu kennzeichnen, etwa `angebot_v2.docx`, obwohl ein Repository vorhanden ist. Die Version steckt dann doppelt in Dateiname und Historie, und beides driftet auseinander.

### Praxisbezug.

Im Projektalltag begegnet Ihnen Versionskontrolle an drei Stellen: beim Zusammenarbeiten (Pull Requests), beim Ausliefern (Tags und Releases) und bei der Fehlersuche (wer hat wann diese Zeile geändert?). Wird ein Altsystem auf Git umgestellt, ist die erste Frage meist, ob die vorhandene SVN-Historie mitkommt. Das ist machbar, wird aber erst in Modul 12 behandelt.

## 1.2 Git-Datenmodell

Wer Git nur über die Befehle kennt, wundert sich oft über scheinbar seltsames Verhalten. Das Datenmodell erklärt fast alles davon. Der wichtigste Gedanke lautet: Git speichert *Schnappschüsse* (englisch *snapshots*) des gesamten Projektzustands und keine Listen von Änderungen. Ein Commit ist ein Foto des Projekts zu einem bestimmten Zeitpunkt. Dass Git Unterschiede trotzdem schnell anzeigen kann, liegt daran, dass es zwei Fotos bei Bedarf vergleicht. Intern werden zudem unveränderte Dateien nicht erneut gespeichert, sondern nur referenziert, sodass Schnappschüsse nicht verschwenderisch sind.

Technisch ist Git eine *inhaltsadressierte Objektdatenbank*. Das heißt: Jedes gespeicherte Objekt wird über den Hashwert seines Inhalts angesprochen. Ein *Hash* ist ein kurzer Fingerabdruck fester Länge, der aus einem beliebig langen Inhalt berechnet wird; ändert sich auch nur ein Zeichen, ergibt sich ein völlig anderer Hash. Git verwendet dafür bisher SHA-1, einen 40-stelligen Hexadezimalwert, in neueren Versionen optional auch SHA-256 mit 64 Stellen. Es gibt vier Objektarten. Ein *Blob* (binary large object) enthält den reinen Dateiinhalt, ohne Dateinamen. Ein *Tree* entspricht einem Verzeichnis und listet Namen, Rechte und die Hashes der enthaltenen Blobs und Unter-Trees. Ein *Commit* verweist auf genau einen Tree (den Projektzustand), auf null, einen oder mehrere Eltern-Commits und trägt Autor, Zeitpunkt und Nachricht. Ein *annotierter Tag* schließlich ist ein benanntes, optional signiertes Etikett auf einem anderen Objekt; er wird in Modul 7 vertieft.

Aus dieser Konstruktion folgt die Integrität. Da der Hash eines Commits aus dem Inhalt berechnet wird, der den Hash des Trees und des Eltern-Commits einschließt, hängt jeder Commit kryptografisch an seiner gesamten Vorgeschichte. Wer irgendwo in der Vergangenheit etwas ändert, ändert alle nachfolgenden Hashes. Das macht Manipulationen sichtbar und erlaubt es, einen Stand durch einen einzigen Hash eindeutig zu bezeichnen. Gleichzeitig sind die Objekte *unveränderlich*: Git überschreibt sie nie, es legt neue an.

Dies lässt sich selbst nachprüfen. Das folgende Beispiel berechnet den Blob-Hash von Hand und vergleicht ihn mit dem von Git, legt dann einen Commit an und seziert ihn Objekt für Objekt. Die Hashes des Commits und des Trees werden bei Ihnen abweichen, weil sie Zeitstempel und Autor enthalten; der Blob-Hash des Textes „hello“ ist dagegen auf jedem Rechner derselbe.

```bash
echo 'hello' | git hash-object --stdin
printf 'blob 6\0hello\n' | sha1sum

git init -b main demo && cd demo
echo 'hello' > gruss.txt
mkdir src && echo 'print("hi")' > src/app.py
git add .
git commit -m "Erster Commit"

git cat-file -t HEAD
git cat-file -p HEAD
git cat-file -p 'HEAD^{tree}'
git cat-file -p HEAD:gruss.txt
```

Die ersten beiden Zeilen liefern beide `ce013625030ba8dba906f756967f9e9ca394464a`. Das zeigt, was Git tatsächlich hasht: den Text `blob`, die Länge des Inhalts in Byte (hier 6, weil der Zeilenumbruch mitzählt), ein Nullbyte und dann den Inhalt. Der Dateiname geht nicht in den Hash ein; zwei Dateien mit identischem Inhalt teilen sich daher ein Objekt. `git cat-file -t` nennt den Typ des Objekts (hier `commit`), `-p` gibt es lesbar aus. Die Ausgabe des Commits enthält die Zeile `tree ...` und, ab dem zweiten Commit, `parent ...`. Mit `HEAD^{tree}` greift man auf den Tree des Commits zu, der zwei Einträge zeigt: den Blob `gruss.txt` und den Unter-Tree `src`. Die vierte Zeile schließlich gibt den Blob selbst aus.

> **Ergänzung aus der Nachrecherche**
>
> **SHA-1 und SHA-256.** Die Git-Dokumentation zu den geplanten Breaking Changes (abgerufen am 28.10.2026) nennt als Ziel für Git 3.0, dass neue Repositories standardmäßig das Objektformat `sha256` statt `sha1` verwenden. Begründet wird das mit der NIST-Abkündigung von SHA-1 und den praktischen Angriffen SHAttered (2017) und Shambles (2020). Die Seite betont, dass bestehende SHA-1-Repositories nicht betroffen sind, dass kein Plan besteht, `sha1` abzukündigen, und dass die Umstellung vom Ökosystem (Bibliotheken, Anwendungen, Hosting-Plattformen) abhängt. Einen Termin für Git 3.0 nennt die Seite nicht. Wer heute ein SHA-256-Repository anlegen möchte, kann das mit `git init --object-format=sha256` tun, sollte aber prüfen, ob die eingesetzte Plattform das unterstützt.

### Typische Fehler und Stolperfallen.

Ein verbreiteter Irrtum ist, Git speichere „Diffs“ und ein Commit sei die Differenz zum Vorgänger. Das führt zu falschen Erwartungen, etwa dass man „einen Commit herauslösen“ könne wie einen Patch. Richtig ist: Ein Commit ist ein vollständiger Zustand; `git show` berechnet die Differenz erst bei der Anzeige. (Beim Packen komprimiert Git ähnliche Objekte intern über Deltas, das ist aber ein reines Speicherdetail.) Ein zweiter Irrtum betrifft die Kurzhashes: Wer in einer Nachricht oder einem Ticket nur wenige Stellen eines Hashes notiert, riskiert Mehrdeutigkeit in sehr großen Repositories. Üblich sind mindestens sieben bis zwölf Stellen; `git rev-parse --short=12 HEAD` liefert die gewünschte Länge.

### Praxisbezug.

Das Datenmodell begegnet Ihnen, sobald etwas schiefgeht. Wenn ein Commit „verschwunden“ scheint, ist er meist noch als Objekt vorhanden und lässt sich wiederfinden (Modul 6). Wenn ein Repository riesig wird, liegt das meist an großen Blobs in der Historie, nicht an der Zahl der Commits (Modul 10). Und in CI/CD-Pipelines dient der Commit-Hash als eindeutige Kennung für Build-Artefakte, weil er bis auf die letzte Datei den Quellstand festlegt.

## 1.3 Repository-Struktur

Ein Git-Repository besteht aus zwei klar getrennten Teilen: dem *Arbeitsverzeichnis* (englisch *working tree*), in dem Sie Dateien normal bearbeiten, und dem Verzeichnis `.git`, in dem Git alle Metadaten und die komplette Objektdatenbank ablegt. Löscht man `.git`, bleiben die Dateien übrig, aber die gesamte Historie ist verloren. Umgekehrt kann man das Arbeitsverzeichnis leeren und mit `git restore .` aus `.git` wiederherstellen. Diese Trennung ist der Grund, weshalb man ein Repository kopieren, sichern oder als Remote nutzen kann, indem man nur `.git` betrachtet. Ein *Bare-Repository*, wie es auf Servern üblich ist, enthält nur diesen Inhalt, ohne Arbeitsverzeichnis.

Innerhalb von `.git` sind einige Einträge besonders wichtig. Die Datei `HEAD` sagt, auf welchem Zweig (oder Commit) Sie gerade stehen. Das Verzeichnis `objects` ist die oben beschriebene Objektdatenbank; die ersten zwei Stellen des Hashes bilden den Unterordner, die restlichen den Dateinamen. Unter `refs` stehen die *Referenzen*, das sind lesbare Namen für Hashes: `refs/heads` enthält die lokalen Zweige, `refs/tags` die Tags, `refs/remotes` die zuletzt bekannten Stände von Remote-Zweigen. Die Datei `index` ist die *Staging Area*, also der Zwischenspeicher für den nächsten Commit (siehe Abschnitt zum lokalen Workflow und Modul 2). `config` enthält die repositoryspezifische Konfiguration, `hooks` Skripte, die bei Ereignissen laufen (Modul 9), und `info/exclude` lokale Ignore-Regeln, die nicht mitversioniert werden.

Das folgende Beispiel erkundet die Struktur eines frischen Repositories mit gewöhnlichen Shell-Befehlen. So sehen Sie, dass es sich um ganz normale Dateien handelt.

```bash
git init -b main struktur && cd struktur
echo "Inhalt" > a.txt
git add a.txt && git commit -m "a.txt hinzugefuegt"

ls .git
cat .git/HEAD
cat .git/refs/heads/main
find .git/objects -type f
git rev-parse --git-dir --show-toplevel
```

`ls .git` zeigt unter anderem `HEAD`, `config`, `objects`, `refs` und `index`. Die Ausgabe von `cat .git/HEAD` ist `ref: refs/heads/main`: HEAD verweist symbolisch auf den Zweig `main`. Die darauffolgende Datei `refs/heads/main` enthält nichts anderes als den 40-stelligen Hash des letzten Commits, und das ist alles, was ein Zweig technisch ist: eine Datei mit einem Hash. Das `find` zählt drei Objekte auf (Blob, Tree, Commit). Zuletzt zeigt `git rev-parse`, wo Git die Metadaten sucht und wo das oberste Arbeitsverzeichnis liegt. Git findet das Repository, indem es vom aktuellen Verzeichnis aus nach oben nach einem `.git` sucht; deshalb funktionieren Befehle auch in Unterordnern.

> **Merke**
>
> Ein Zweig ist eine Datei mit einem Hash. HEAD ist eine Datei mit dem Namen eines Zweiges. Wenn das so einfach ist, erklärt sich, warum das Anlegen von Zweigen in Git praktisch nichts kostet.

### Typische Fehler und Stolperfallen.

Ein klassischer Fehler ist das *verschachtelte Repository*: Wer `git init` in einem Unterordner eines bestehenden Repositories ausführt, erzeugt ein zweites `.git`. Das äußere Repository behandelt den Ordner anschließend als Sonderfall und verfolgt seinen Inhalt nicht mehr, was oft erst beim Kollegen auffällt, der nur einen leeren Ordner sieht. Falsch ist also ein zweites `git init` im Projektordner; richtig ist, vorher mit `git rev-parse --show-toplevel` zu prüfen, ob man bereits in einem Repository steht. Der zweite Fehler ist, Dateien in `.git` von Hand zu bearbeiten, um etwa einen Zweig zu verschieben. Das geht technisch, umgeht aber alle Sicherheitsnetze wie das Reflog; dafür gibt es Befehle wie `git branch -f`.

### Praxisbezug.

Die Struktur zählt, sobald man Repositories sichert oder repariert. Ein Backup eines Repositories ist im Kern eine Kopie von `.git` oder besser ein Mirror-Klon (Modul 12). Bei Performance-Problemen lohnt ein Blick auf die Größe von `.git/objects`. Und bei Sicherheitsfragen ist wichtig, dass `.git/config` und `.git/hooks` ausführbaren Einfluss haben: Wer ein fremdes Repository klont, sollte darauf achten, dass dort keine unerwarteten Hooks aktiv werden.

## 1.4 Lokaler Workflow

Der tägliche Arbeitsablauf mit Git folgt einem einfachen Dreiklang: ändern, vormerken, speichern. Man bearbeitet Dateien im Arbeitsverzeichnis. Dann entscheidet man, welche Änderungen in den nächsten Schnappschuss gehören, und merkt sie mit `git add` in der Staging Area vor. Schließlich speichert `git commit` den vorgemerkten Zustand dauerhaft in der Historie. Die Zwischenstation, die Staging Area oder der *Index*, wirkt zunächst umständlich, ist aber der Grund, warum Git so saubere Historien erlaubt: Man kann aus einem unaufgeräumten Arbeitsverzeichnis gezielt thematisch zusammengehörige Änderungen in einen Commit packen und den Rest später separat speichern.

Zum Prüfen gibt es zwei Standardbefehle. `git status` zeigt, welche Dateien geändert, vorgemerkt oder unbekannt (englisch *untracked*) sind. `git diff` zeigt die noch nicht vorgemerkten Änderungen, `git diff --staged` diejenigen, die im nächsten Commit enthalten sein werden. Wer vor jedem Commit beide Befehle ansieht, vermeidet den größten Teil aller Fehlcommits. Als Leitlinie gilt: kleine, nachvollziehbare Änderungen bevorzugen. Ein Commit sollte genau einen Gedanken enthalten, den man in einem Satz beschreiben kann. Kleine Commits sind leichter zu prüfen, leichter rückgängig zu machen und liefern bei der Fehlersuche präzisere Treffer.

Das nächste Beispiel zeigt einen vollständigen Durchlauf mit zwei getrennten Commits aus einer gemeinsamen Arbeitsphase. Dazu wird `git add -p` genutzt, mit dem man Änderungen stückweise (*Hunks*) auswählt.

```bash
git init -b main workflow && cd workflow
printf "Zeile 1\nZeile 2\nZeile 3\n" > notiz.txt
git add notiz.txt && git commit -m "notiz.txt angelegt"

sed -i 's/Zeile 1/Zeile eins/' notiz.txt
echo "Zeile 4" >> notiz.txt
printf "Neue Datei\n" > todo.txt

git status -s
git diff
git add todo.txt
git diff --staged
git commit -m "todo.txt hinzugefuegt"
git commit -am "notiz.txt: Zeile 1 umbenannt, Zeile 4 ergaenzt"
git log --oneline --stat
```

Nach den Änderungen zeigt `git status -s` (die Kurzform) in zwei Spalten den Zustand: `M` für modifiziert und `??` für unbekannt. Das erste `git diff` listet die Änderungen an `notiz.txt`, die neue Datei erscheint dort nicht, weil sie noch nicht verfolgt wird. Nach `git add todo.txt` sieht man sie in `git diff --staged`. Der erste Commit enthält nur `todo.txt`, der zweite mit `-a` die bereits verfolgte, geänderte `notiz.txt`. Das Resultat sind zwei sinnvoll getrennte Commits aus einer Arbeitssitzung. `--stat` fasst am Ende pro Commit die Dateien zusammen.

Eine Commit-Nachricht besteht üblicherweise aus einer kurzen Betreffzeile (etwa bis 50 Zeichen, im Imperativ oder als knappe Feststellung) und, durch eine Leerzeile getrennt, einem erklärenden Text, der das *Warum* festhält. Das *Was* steht ohnehin im Diff, das Warum nicht. Wer ohne `-m` committet, bekommt den konfigurierten Editor geöffnet (siehe Abschnitt zur Konfiguration).

### Typische Fehler und Stolperfallen.

Der folgende Fehler ist besonders verbreitet: `git add .` oder `git commit -a` ohne vorherigen Blick auf `git status`. So landen Logdateien, lokale Konfigurationen oder gar Zugangsdaten im Repository. Falsch ist `git add .` als Reflex; richtig ist, erst `git status` und `git diff` zu lesen und dann gezielt Dateien (oder mit `git add -p` einzelne Hunks) vorzumerken. Ebenso problematisch sind Sammel-Commits mit Nachrichten wie „Änderungen“ oder „wip“: Sie erfüllen den Zweck der Historie nicht, weil später niemand erkennt, warum etwas geändert wurde.

### Praxisbezug.

In Teams entscheidet die Qualität der Commits über die Qualität von Code Reviews. Ein Review von drei klar getrennten, benannten Commits dauert deutlich weniger als der Review eines großen Sammel-Commits. Dieselbe Disziplin zahlt sich später beim Zurückrollen eines Fehlers aus: Ist die fehlerhafte Änderung ein einzelner, kleiner Commit, lässt sie sich mit `git revert` sauber entfernen (Modul 6).

> **Praxisbeispiel**
>
> Am Freitagnachmittag ändern Sie in einer Rechenfunktion die Rundungslogik und räumen nebenbei die Variablennamen auf. Am Montag meldet die Buchhaltung falsche Beträge. Haben Sie beides in einem Commit „Änderungen“ gespeichert, müssen Sie zwischen Aufräumen und Rundung mühsam trennen. Mit zwei Commits („Rundung auf zwei Nachkommastellen“ und „Variablennamen vereinheitlicht“) nehmen Sie nur den ersten mit einem einzigen Befehl zurück und behalten die Aufräumarbeit.

## 1.5 Git CLI und Tools

Git ist im Kern ein Kommandozeilenprogramm (englisch *Command Line Interface*, CLI). Alle anderen Werkzeuge, ob grafische Clients oder IDE-Integrationen, rufen letztlich dieselben Funktionen auf oder bilden sie nach. Die Kommandozeile hat dabei einen entscheidenden Vorteil: Sie bietet den vollständigen Funktionsumfang, sie ist skriptbar, und die Befehle lassen sich in Dokumentation und Fehlerberichten exakt wiedergeben. Der Nachteil ist die Einstiegshürde. Grafische Oberflächen dagegen visualisieren die Historie als Graph, erlauben das Stagen einzelner Zeilen per Mausklick und zeigen Konflikte nebeneinander an.

Die gängigen Werkzeugklassen sind schnell aufgezählt. Zur ersten Klasse gehören die IDE-Integrationen in Visual Studio Code, JetBrains-Produkten und Visual Studio; sie sind praktisch, weil sie Diff, Commit und Konfliktlösung direkt neben dem Code anzeigen. Zur zweiten Klasse zählen eigenständige grafische Clients wie GitHub Desktop, Sourcetree, GitKraken oder das mit Git mitgelieferte `git gui` und `gitk`. Eine dritte Klasse sind Terminaloberflächen wie `lazygit` oder `tig`, die die Tastaturgeschwindigkeit der Kommandozeile mit einer Übersicht verbinden. Welche Werkzeuge ein Team einsetzt, ist eine Standardisierungsfrage: Unterschiedliche Programme haben bisweilen unterschiedliche Voreinstellungen etwa bei Zeilenenden oder beim Pull-Verhalten, und das erzeugt Reibung, wenn die Konfiguration nicht abgestimmt ist. Die Toolauswahl sollte sich deshalb nach dem Teamstandard richten, ohne dass jemand die Kommandozeile gänzlich meiden kann, denn bei Problemen und in der Automatisierung führt an ihr kein Weg vorbei.

> **Ergänzung aus der Nachrecherche**
>
> **Aktueller Stand unter Windows.** Die Installationsseite von Git für Windows nannte beim Abruf am 28.10.2026 als aktuelle Version 2.56.0, veröffentlicht am 26.10.2026. Aus der Praxis (Wissensstand der Autorin bzw. des Autors, auf der abgerufenen Seite nicht geprüft): Die Distribution bringt in der Regel eine Bash-Umgebung, `git gui` und einen Credential Helper mit; der Umfang hängt von den Installer-Optionen ab. Welche Version in Ihrer Firma freigegeben ist, kann abweichen; prüfen Sie Ihre lokale Version mit `git --version`.

Das folgende Beispiel richtet einige Aliase ein und zeigt einen kompakten Graphen der Historie. Aliase sind Abkürzungen, die in der Konfiguration abgelegt werden; sie sparen bei häufigen Befehlen Tipparbeit und bleiben dennoch transparent, weil sie nur Git-Aufrufe ersetzen.

```bash
git --version
git init -b main werkzeuge && cd werkzeuge
git commit --allow-empty -m "Start"
git config --global alias.st "status -sb"
git config --global alias.lg "log --oneline --graph --decorate --all"

git st
git lg
git help -a | head -20
```

`git --version` zeigt die installierte Version. Die beiden `git config --global alias....`-Zeilen definieren die Kürzel `st` und `lg`. Danach ruft `git st` den Status in Kurzform mit Zweiginformation auf, `git lg` die Historie als ASCII-Graph über alle Zweige. Die letzte Zeile listet die verfügbaren Git-Befehle auf und leitet den Anfang der Ausgabe weiter, um zu zeigen, wie groß der Funktionsumfang ist.

### Typische Fehler und Stolperfallen.

Zwei Stolperfallen treten gehäuft auf. Erstens verführt die Bequemlichkeit grafischer Oberflächen dazu, Knöpfe wie „Sync“ oder „Pull“ zu klicken, ohne zu wissen, was dahinter passiert; die Folge sind unerwartete Merge-Commits oder überschriebene Änderungen. Falsch ist das blinde Vertrauen auf einen „Sync“-Knopf; richtig ist, zumindest einmal die zugrunde liegenden Befehle (`fetch`, `merge` oder `rebase`, `push`) manuell durchgespielt zu haben. Zweitens gibt es unter Windows Konflikte zwischen mehreren installierten Git-Versionen: Die Git-Bash, die IDE und der Client bringen eigene Binaries mit, und je nach `PATH` läuft in der Konsole eine andere Version als im Client.

### Praxisbezug.

In Projekten gibt es oft eine Dokumentationsseite „Entwicklungsumgebung einrichten“, die genau festlegt, welche Git-Version, welcher Editor und welche Aliase empfohlen werden. Das ist kein Detail: Einheitliche Werkzeuge verringern Supportaufwand beim Onboarding neuer Kolleginnen und Kollegen und vermeiden Phänomene wie „bei mir geht es“ durch unterschiedliche Zeilenende-Behandlung.

## 1.6 Konfiguration

Git liest seine Einstellungen aus Konfigurationsdateien auf mehreren Ebenen, sogenannten *Scopes*. Die Ebene *system* gilt für alle Benutzer des Rechners und liegt in der Git-Installation. Die Ebene *global* gilt für den angemeldeten Benutzer und steht im Home-Verzeichnis, meist in `~/.gitconfig` oder unter `~/.config/git/config`. Die Ebene *local* gilt nur für ein Repository und liegt in `.git/config`. Dazu kommt die seltener genutzte Ebene *worktree* für zusätzliche Arbeitsverzeichnisse. Bei widersprüchlichen Werten gilt die spezifischere Ebene: local überschreibt global, global überschreibt system. Mit `git config --list --show-origin` sieht man jeden Wert zusammen mit der Datei, aus der er stammt, und das ist das wichtigste Diagnosewerkzeug bei unerwartetem Verhalten.

Es gibt eine Handvoll Einstellungen, die jeder gleich zu Beginn setzen sollte. `user.name` und `user.email` werden in jeden Commit geschrieben; ohne sie verweigert Git den Commit oder rät falsch. `core.editor` bestimmt, welches Programm sich für Commit-Nachrichten öffnet. `init.defaultBranch` legt den Namen des ersten Zweigs neuer Repositories fest. `pull.rebase` regelt, ob `git pull` mergen oder rebasen soll. `merge.conflictStyle` mit dem Wert `zdiff3` zeigt bei Konflikten zusätzlich den gemeinsamen Ausgangstext, was die Lösung erleichtert. Schließlich bestimmt `credential.helper`, wo Zugangsdaten sicher zwischengespeichert werden.

Das Beispiel setzt diese Grundeinstellungen und zeigt eine bedingte Konfiguration, mit der man für private und berufliche Projekte unterschiedliche E-Mail-Adressen verwendet, ohne sie pro Repository setzen zu müssen.

```bash
git config --global user.name "Max Muster"
git config --global user.email "max.muster@firma.example"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
git config --global pull.rebase false
git config --global merge.conflictStyle zdiff3

git config --list --show-origin
git config user.email
```

Der folgende Auszug ist kein Shell-Befehl, sondern Dateiinhalt:

**Listing:** Ganz unten in der Datei `~/.gitconfig`

```text
[includeIf "gitdir:~/privat/"]
    path = ~/.gitconfig-privat
```

Die ersten sechs Zeilen schreiben in die globale Datei. `core.editor "code --wait"` startet Visual Studio Code und wartet, bis das Fenster geschlossen wird; ohne `--wait` würde Git die Nachricht leer lesen. `git config --list --show-origin` listet alles samt Herkunft, `git config user.email` gibt den *wirksamen* Wert aus. Der separate Auszug wird in die Datei `~/.gitconfig` geschrieben: Für alle Repositories unter `~/privat/` bindet Git zusätzlich die Datei `.gitconfig-privat` ein, in der man eine andere E-Mail-Adresse hinterlegt.

> **Ergänzung aus der Nachrecherche**
>
> **Standardzweig und Git 3.0.** Laut der Git-Dokumentation zu den geplanten Breaking Changes (abgerufen am 28.10.2026) soll in Git 3.0 neuen Repositories standardmäßig der Zweig `main` statt `master` zugewiesen werden. Wer den alten Namen behalten möchte, setzt `git config --global init.defaultBranch master`. Dieselbe Seite nennt weitere geplante Änderungen, etwa `reftable` als Standardformat für Referenzen und `safe.bareRepository=explicit`. Letzteres verhindert, dass Git versteckte Bare-Repositories in übergeordneten Verzeichnissen implizit nutzt, ein Angriffsweg über präparierte Klone. Die Seite nennt keinen Veröffentlichungstermin; setzen Sie `init.defaultBranch` sicherheitshalber schon heute explizit.

### Typische Fehler und Stolperfallen.

Der folgende Fehler kommt in fast jedem Team vor: Commits erscheinen mit der falschen E-Mail-Adresse, weil global die private Adresse gesetzt ist und im Firmenprojekt niemand die lokale überschrieben hat. Im Repository ist das später nur mit einer Historienbereinigung zu korrigieren. Falsch ist, die globale Adresse zu vergessen oder das Setzen dem Zufall zu überlassen; richtig ist, im Projekt `git config user.email ...` ohne `--global` zu setzen oder `includeIf` einzusetzen. Ein zweiter, sicherheitsrelevanter Fehler: Zugangsdaten in der Konfiguration, zum Beispiel ein Token direkt in einer Remote-URL wie `https://user:token@host/repo.git`. Dieser Wert steht im Klartext in `.git/config` und taucht in Logs und Screenshots auf. Verwenden Sie stattdessen einen Credential Helper oder SSH-Schlüssel.

### Praxisbezug.

In Unternehmen wird eine Basiskonfiguration oft per Skript oder über eine „Systemkonfiguration“ verteilt, etwa mit einheitlichen Werten für `pull.rebase` und Proxy-Einstellungen. Bei Problemen in CI-Pipelines lohnt sich ein `git config --list --show-origin` im Job-Log, um sicherzustellen, dass der Runner dieselben Voreinstellungen hat wie der eigene Rechner.

> **Praxisbeispiel**
>
> Bei einem Audit soll eine Firma nachweisen, wer eine sicherheitsrelevante Änderung vorgenommen hat. Die Commits stammen von `max@privat.example`, einer privaten Adresse, weil auf dem Laptop nur global konfiguriert war. Der Nachweis zur Person ist dadurch schwach. Mit `includeIf` je Projektordner oder einer Projektvorgabe zur lokalen Konfiguration wäre das nicht passiert.

## 1.7 Hilfesystem

Git hat über hundert Befehle mit teils Dutzenden Optionen, und niemand kennt sie alle auswendig. Wichtiger als das Auswendiglernen ist deshalb, die eingebaute Dokumentation effizient zu nutzen. Es gibt drei Stufen. Die kürzeste ist `git <befehl> -h`, die nur eine knappe Syntaxübersicht im Terminal ausgibt. Die ausführliche ist `git help <befehl>` beziehungsweise `git <befehl> --help`, die die vollständige Handbuchseite öffnet, je nach Konfiguration im Pager oder im Browser. Die dritte Stufe sind die Konzeptseiten: `git help -g` listet Leitfäden wie `tutorial`, `everyday`, `glossary` und `revisions` auf, und `git help -a` listet alle Befehle.

Eine Handbuchseite folgt immer demselben Aufbau, und wer ihn kennt, findet Antworten schnell. Oben steht die *Synopsis* (Aufruf), darunter die *Beschreibung*, dann die *Optionen* und weiter unten oft *Beispiele*. In der Synopsis bedeuten eckige Klammern optionale Teile, spitze Klammern Platzhalter, ein senkrechter Strich Alternativen und drei Punkte Wiederholbarkeit. Aus `git branch [-d | -D] <branchname>...` liest man also: Die Option `-d` oder `-D` ist optional, danach folgen ein oder mehrere Zweignamen. Das Glossar (`git help glossary`) erklärt alle Fachbegriffe, was bei unbekannten Wörtern wie „pathspec“ oder „refspec“ hilft.

Im folgenden Beispiel schlägt man gezielt etwas nach, ohne den Terminalfluss zu verlassen: eine Kurzübersicht, eine Suche innerhalb der Handbuchseite und die Konzeptseiten.

```bash
git commit -h
git help commit
git help -g
git help glossary
git help -a | grep -i rebase
git help config
```

Die erste Zeile gibt die kompakte Optionsübersicht für `commit` aus. `git help commit` öffnet die vollständige Seite; im Pager sucht man mit `/Suchbegriff` nach Stichwörtern und verlässt ihn mit `q`. Mit `git help -g` erhält man die Liste der Konzeptleitfäden, `git help glossary` öffnet davon das Glossar. Die fünfte Zeile filtert die Liste aller Befehle nach „rebase“. Die letzte öffnet die Referenz aller Konfigurationsvariablen, aus der sich Einstellungen wie `merge.conflictStyle` nachschlagen lassen.

### Typische Fehler und Stolperfallen.

Ein häufiger Fehler ist, bei Problemen sofort in Foren zu suchen und dort Befehle zu kopieren, deren Wirkung man nicht verstanden hat. Besonders gefährlich sind Antworten, die `--force` oder `reset --hard` enthalten. Falsch ist das Kopieren ohne Prüfung; richtig ist, jeden unbekannten Befehl und jede Option zuerst mit `git help` nachzuschlagen. Ebenso häufig ist das Vermischen von Dokumentationsständen: Online-Anleitungen zeigen oft noch `git checkout` für Dinge, die moderne Git-Versionen mit `git switch` und `git restore` trennen. Die lokale Handbuchseite entspricht stets der installierten Version.

### Praxisbezug.

Erfahrene Entwickelnde schlagen sehr oft nach, nicht weil sie es nicht wissen, sondern weil sie die Optionen nicht für jeden Befehl im Kopf haben müssen. In Incident-Situationen, wenn etwa ein Merge schiefgegangen ist, ist die lokale Dokumentation die zuverlässigste Quelle, weil sie offline und versionsgenau verfügbar ist.

## 1.8 Erstes Repository

Zum Abschluss des Moduls bringen wir alles zusammen und legen ein erstes, sauberes Repository an. Es gibt dafür zwei Wege. Mit `git init` macht man ein bestehendes oder neues Verzeichnis zu einem Repository. Mit `git clone <url>` holt man ein bereits bestehendes Repository samt Historie. Wir nehmen den ersten Weg. Beim ersten Commit lohnt es sich, einige Regeln einzuhalten, die später Mühe sparen: ein sinnvoller Startinhalt (eine `README.md` mit einem Satz zum Zweck), eine `.gitignore` gegen Datenmüll, eine klare erste Commit-Nachricht und ein bewusst gewählter Zweigname.

Das Beispiel richtet ein kleines Projekt vollständig ein und prüft danach den Zustand. Es nutzt `git init -b main`, damit der Zweigname unabhängig von der globalen Einstellung festliegt.

```bash
mkdir hallo-git && cd hallo-git
git init -b main

cat > README.md <<'EOF'
# hallo-git
Uebungsprojekt zu Git-Grundlagen.
EOF

printf "*.log\n.env\nnode_modules/\n" > .gitignore
echo "Beispiel" > daten.log

git status
git add README.md .gitignore
git status -s
git commit -m "Projekt angelegt: README und .gitignore"

git log --stat
git branch --show-current
git status
```

Das erste `git status` zeigt `README.md` und `.gitignore` als unbekannt, `daten.log` dagegen nicht, denn die Ignore-Regel `*.log` greift bereits, auch wenn die Datei noch nicht committet ist. Wir merken nur die beiden gewünschten Dateien vor und committen sie. `git log --stat` zeigt den Commit mit den zwei Dateien, `git branch --show-current` gibt `main` aus. Das abschließende `git status` meldet „nothing to commit, working tree clean“: Das Arbeitsverzeichnis stimmt mit dem letzten Commit überein, und die ignorierte Datei wird bewusst nicht als Abweichung gezählt. Diesen sauberen Zustand sollten Sie als Normalfall zwischen zwei Arbeitsschritten kennen.

Ein Detail am Rand: Bei einem neuen Repository ohne Commit existiert der Zweig noch gar nicht, weil `refs/heads/main` erst mit dem ersten Commit entsteht. Wer vorher `git log` aufruft, bekommt die Meldung, dass der Zweig noch keine Commits enthält; das ist kein Fehler.

### Typische Fehler und Stolperfallen.

Der häufigste Anfängerfehler ist, Git im falschen Verzeichnis zu initialisieren, etwa im Home-Verzeichnis oder auf dem Desktop. Dann versucht Git, alles darunter zu verfolgen, und `git status` listet Tausende Dateien auf. Falsch ist `git init` im Home-Verzeichnis; richtig ist, vorher mit `pwd` den Ort zu prüfen und sofort danach `git status` zu lesen. Der zweite Fehler: Die `.gitignore` nachträglich anzulegen und zu erwarten, dass sie bereits committete Dateien entfernt. Ignore-Regeln wirken nur auf unverfolgte Dateien; versehentlich committete Dateien müssen mit `git rm --cached <datei>` aus der Verfolgung genommen werden (Modul 2).

### Praxisbezug.

In Projekten legt man das erste Repository selten allein auf dem Laptop an; meist erzeugt man zuerst ein leeres Repository bei GitHub oder GitLab und klont es. Der Ablauf bleibt derselbe: README, `.gitignore`, Lizenz und ein erster Commit. Viele Teams haben dafür Repository-Vorlagen, damit alle neuen Projekte mit denselben Standards starten (Modul 5).

## Übungen zu Modul 01

**Aufgabe 1.** Erklären Sie in zwei bis drei Sätzen den Unterschied zwischen zentraler und verteilter Versionskontrolle und nennen Sie je zwei Systeme.

**Aufgabe 2.** Warum erhalten zwei Dateien mit identischem Inhalt aber unterschiedlichem Namen im selben Repository denselben Blob-Hash? Was folgt daraus für den Speicherverbrauch?

**Aufgabe 3.** Nennen Sie die vier Git-Objektarten und beschreiben Sie, auf was ein Commit-Objekt verweist.

**Aufgabe 4.** Wo wird festgehalten, auf welchem Zweig Sie sich befinden, und was steht in der Datei eines Zweigs unter `refs/heads`?

**Aufgabe 5.** Praktisch: Legen Sie ein neues Repository an, committen Sie eine Datei `a.txt` mit dem Inhalt `hello`, und weisen Sie nach, dass Git den Blob-Hash `ce013625030ba8dba906f756967f9e9ca394464a` vergibt, einmal mit `git hash-object`, einmal mit `git ls-tree`.

**Aufgabe 6.** Praktisch: Erzeugen Sie eine Änderung an zwei Dateien und bilden Sie daraus zwei getrennte Commits. Prüfen Sie vor jedem Commit mit `git diff --staged`, was enthalten ist.

**Aufgabe 7.** Praktisch: Setzen Sie global Name, E-Mail, `init.defaultBranch` und `merge.conflictStyle zdiff3`. Überschreiben Sie dann die E-Mail-Adresse nur für ein Repository und weisen Sie mit `git config --list --show-origin` nach, woher jeder Wert stammt.

**Aufgabe 8.** Praktisch: Richten Sie zwei Aliase ein (`st` und `lg`) und beschreiben Sie, wie Sie ohne Internet nachlesen, welche Optionen `git log` kennt.

## Lösungen zu Modul 01

**Lösung 1.** Bei zentralen Systemen (z. B. Subversion, CVS) liegt die vollständige Historie auf einem Server, Clients halten nur eine Arbeitskopie. Bei verteilten Systemen (z. B. Git, Mercurial) hat jede Kopie die vollständige Historie und ist offline voll nutzbar. Ein Server ist bei verteilten Systemen eine organisatorische Konvention.

**Lösung 2.** Git berechnet den Blob-Hash aus Typ, Länge und Inhalt, nicht aus dem Dateinamen. Gleicher Inhalt ergibt also immer denselben Hash und damit dasselbe Objekt. Der Name steht im Tree, nicht im Blob. Doppelter Inhalt kostet deshalb keinen zusätzlichen Speicher.

**Lösung 3.** Blob (Dateiinhalt), Tree (Verzeichnis), Commit (Schnappschuss mit Metadaten) und annotierter Tag. Ein Commit verweist auf genau einen Tree, auf null oder mehr Eltern-Commits und enthält Autor, Zeitpunkt und Nachricht.

**Lösung 4.** In der Datei `.git/HEAD` (z. B. `ref: refs/heads/main`). Die Zweigdatei enthält den Hash des Commits, auf den der Zweig zeigt.

**Lösung 5.** Beispiel: `git init -b main t && cd t && echo hello > a.txt && git add a.txt && git commit -m x`, dann `git hash-object a.txt` und `git ls-tree HEAD`. Beide nennen `ce01362...`. Das stimmt, weil Git `blob 6\0hello\n` hasht und der Zeilenumbruch mitzählt.

**Lösung 6.** Beispiel: Beide Dateien ändern, `git add datei1`, `git diff --staged` prüfen, committen, dann `datei2` ebenso. Die Aufteilung gelingt, weil die Staging Area bestimmt, was in den nächsten Commit kommt, und nicht das Arbeitsverzeichnis.

**Lösung 7.** Die Befehle stehen im Abschnitt Konfiguration. Die lokale Überschreibung erfolgt mit `git config user.email x@y` im Repository. `--show-origin` zeigt für den globalen Wert die Datei `.gitconfig` und für den lokalen `.git/config`; der lokale Wert gewinnt, weil die spezifischere Ebene Vorrang hat.

**Lösung 8.** Aliase: `git config --global alias.st "status -sb"` und `alias.lg "log --oneline --graph --decorate --all"`. Offline lesen mit `git help log` oder kurz mit `git log -h`.
