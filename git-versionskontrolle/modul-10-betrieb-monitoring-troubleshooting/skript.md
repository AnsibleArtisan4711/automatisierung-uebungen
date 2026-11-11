# Modul 10 - Betrieb, Monitoring und Troubleshooting

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (10.1 bis 10.8), danach folgen „Übungen zu Modul 10“ (8 Aufgaben) und „Lösungen zu Modul 10“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 2

> Die Modulübersicht und die Summe der Lerneinheiten stimmen überein (acht Einheiten zu je 1,25 h). Prioritäten der Einheiten: 10.1 bis 10.6 Prio 2, 10.7 und 10.8 Prio 3.

**Lernziele des Moduls:** Repository-Probleme, Performance, Audits und Fehlerbilder analysieren; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Während die früheren Module gezeigt haben, wie man Git benutzt, geht es hier um den Betrieb: Was tut man, wenn ein Repository langsam wird, wenn eine Anmeldung plötzlich scheitert, wenn Binärdateien die Historie aufblähen oder wenn ein Push abgelehnt wird? Und wie stellt man sicher, dass man solche Situationen vorher bemerkt und nachher sauber dokumentiert?

| **Nr.** | **Lerneinheit**            | **Dauer (h)** | **Prio** |
|:--------|:---------------------------|--------------:|:--------:|
| 10.1    | Repository Performance     |          1,25 |    2     |
| 10.2    | Git LFS Betrieb            |          1,25 |    2     |
| 10.3    | Submodules und Subtrees    |          1,25 |    2     |
| 10.4    | Typische Fehlerbilder      |          1,25 |    2     |
| 10.5    | Credential Troubleshooting |          1,25 |    2     |
| 10.6    | Audit und Reports          |          1,25 |    2     |
| 10.7    | Backup-nahe Prüfungen      |          1,25 |    3     |
| 10.8    | Runbooks                   |          1,25 |    3     |
|         | **Summe**                  |      **10,0** |          |

> **Hinweis zu den Beispielen in diesem Modul**
>
> Alle Skripte dieses Moduls wurden in einer Testumgebung mit Git 2.43 und Git LFS 3.4.1 ausgeführt; gezeigte Ausgaben stammen aus diesen Läufen. Zugriffe auf echte Plattformen (Token, Quotas, Reviews) lassen sich dort nicht ausführen. Wo ein Beispiel einen Plattformdienst nachbildet, steht das im Text: Die Anmeldeprobleme in 10.5 werden gegen einen kleinen lokalen Testserver erzeugt, der nur Statuscodes zurückgibt.

## 10.1 Repository Performance

Ein Git-Repository wird nicht von einem Tag auf den anderen langsam. Es wächst, und irgendwann überschreitet es die Schwelle, an der Menschen es merken: Der erste Klon in der Pipeline dauert Minuten statt Sekunden, `git status` braucht spürbar Zeit, `git fetch` lädt Hunderte Megabyte. Wer Performance verbessern will, muss zuerst *messen* und danach die Ursache benennen, denn die Gegenmaßnahmen unterscheiden sich stark. Es gibt im Wesentlichen drei Ursachen. Erstens *große Dateien in der Historie*: Eine einmal committete Binärdatei bleibt in jedem Klon, auch wenn sie längst gelöscht wurde, weil die Historie sie noch referenziert. Zweitens *sehr viele Objekte*, also Commits, Dateien und Verzeichnisse, bei denen Operationen, die den Arbeitsbaum durchlaufen, mit der Zahl der Dateien wachsen. Drittens ein *ungepflegter Objektspeicher*: Git legt neue Objekte zunächst einzeln als lose Dateien ab und packt sie erst später zusammen; ohne Pflege entstehen viele lose Objekte und Pack-Dateien, die langsamer sind.

Der erste Messpunkt ist der Speicherbedarf. Der Befehl `git count-objects -vH` zeigt laut Dokumentation (abgerufen am 28.10.2026) die Zahl und Größe der losen Objekte sowie die Größe der Pack-Dateien in lesbaren Einheiten. Für die Frage „welche Dateien sind schuld?“ braucht man die Objekte, die in der gesamten Historie vorkommen, mit ihrer Größe: Das liefert die Kombination aus `git rev-list --objects --all` und `git cat-file --batch-check`. Das folgende Beispiel baut ein Repository, in dem jemand eine drei Megabyte große Datei eingecheckt und im nächsten Commit wieder gelöscht hat, und macht die Altlast sichtbar. Danach zeigt es zwei Wege, einen Klon trotzdem klein zu halten: einen flachen Klon (`--depth`) und einen Klon ohne Dateiinhalte (`--filter=blob:none`). Die Dokumentation zu `git clone` (abgerufen am 28.10.2026) beschreibt beide Optionen; der Filter funktioniert nur, wenn der Server ihn erlaubt, was im Beispiel mit `uploadpack.allowFilter` nachgebildet wird.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main quelle && cd quelle
git config user.name Dev; git config user.email dev@example.org
echo "Hallo" > README.md && git add . && git commit -q -m "feat: Start"
head -c 3000000 /dev/urandom > messdaten.bin
git add . && git commit -q -m "feat: Messdaten"
git rm -q messdaten.bin && git commit -q -m "fix: Messdaten wieder entfernt"
echo "--- Objektspeicher"
git count-objects -vH | grep -E "^(count|size):"
echo "--- die groessten Objekte der gesamten Historie"
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob"' | sort -k3 -n -r | head -3 | cut -c1-60
echo "--- Datei im Arbeitsbaum vorhanden?"
ls messdaten.bin 2>&1 | head -1
cd ..

git clone -q --bare quelle server.git
git -C server.git config uploadpack.allowFilter true
git clone -q file://$PWD/server.git voll
git clone -q --depth=1 file://$PWD/server.git flach
git clone -q --filter=blob:none file://$PWD/server.git ohne-blobs
echo "--- Groesse der Klone (KB, nur .git)"
for k in voll flach ohne-blobs; do printf "%-10s %s\n" $k "$(du -sk $k/.git | cut -f1)"; done
echo "--- Commits je Klon"
for k in voll flach ohne-blobs; do printf "%-10s %s\n" $k "$(git -C $k rev-list --count HEAD)"; done
```

Die Ausgabe zeigt, dass die gelöschte Datei im Arbeitsbaum nicht mehr vorhanden ist, im Objektspeicher aber mit knapp drei Megabyte als größtes Objekt auftaucht. Genau dieses Muster ist die häufigste Ursache aufgeblähter Repositories. Der vollständige Klon trägt die Altlast mit sich, der flache Klon verzichtet auf die Historie und bekommt nur den letzten Commit, und der Klon mit `--filter=blob:none` lädt zunächst nur Commits und Verzeichnisse und holt Dateiinhalte bei Bedarf nach (ein sogenannter Partial Clone). Der flache Klon ist für Pipelines ideal, die nur bauen; er scheitert aber an allem, was Historie braucht (Modul 09, Abschnitt 9.8). Der Klon ohne Blobs behält die vollständige Commit-Historie, sodass `git log` funktioniert, spart aber den Großteil des Platzes, solange man nicht auf alte Dateistände zugreift. Beim Wechsel zu einem Commit holt Git die dort gebrauchten Dateien dann einzeln nach.

Zeitmessungen sind der zweite Messpunkt. Man misst nicht mit dem Bauchgefühl, sondern mit der Uhr, und zwar mehrfach, weil Dateisystem-Caches die ersten Läufe verfälschen. Ein einfacher Rahmen ist eine kleine Funktion, die einen Befehl mehrfach ausführt und die Zeiten in Millisekunden ausgibt. Die folgende Variante misst Klonen und Status des Beispiels. Sie zeigt zugleich die Pflege des Objektspeichers: Die Dokumentation zu `git gc` und `git maintenance` (abgerufen am 28.10.2026) beschreibt, dass `git gc` lose Objekte in Pack-Dateien zusammenführt und nicht mehr erreichbare Objekte entfernt, und dass `git maintenance` solche Aufgaben im Hintergrund planen kann.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main mess && cd mess
git config user.name Dev; git config user.email dev@example.org
for i in $(seq 1 300); do echo "Zeile $i" > "datei$i.txt"; done
git add . && git commit -q -m "feat: viele Dateien"
for i in $(seq 1 40); do echo "mehr $i" >> datei$i.txt; git commit -q -am "fix: $i"; done
cd ..

zeit() {  # zeit <Wiederholungen> <Befehl...> -> Median in Millisekunden
  n=$1; shift; t=""
  for i in $(seq 1 $n); do
    a=$(date +%s%N); "$@" >/dev/null 2>&1; b=$(date +%s%N)
    t="$t $(( (b - a) / 1000000 ))"
  done
  echo $t | tr ' ' '\n' | sort -n | sed -n "$(( (n + 1) / 2 ))p"
}
klonen() { rm -rf ziel && git clone -q file://$PWD/mess ziel; }
echo "Klonen (Median von 5): $(zeit 5 klonen) ms"
for i in $(seq 1 30); do echo "neu $i" > ziel/neu$i.txt; git -C ziel add -A; git -C ziel -c user.name=Dev -c user.email=dev@example.org commit -q -m "feat: lokal $i"; done
echo "--- vor der Pflege (30 lokale Commits, Objekte noch lose)"
git -C ziel count-objects -v | grep -E "^(count|packs|size-pack):"
git -C ziel gc -q
echo "--- nach git gc"
git -C ziel count-objects -v | grep -E "^(count|packs|size-pack):"
echo "Status (Median von 5): $(zeit 5 git -C ziel status) ms"
git -C ziel fsck --full >/dev/null 2>&1; echo "fsck rc=$?"
```

Das Messskript führt den Befehl fünfmal aus und gibt den Median aus, damit ein einzelner Ausreißer das Ergebnis nicht bestimmt. Absolute Zahlen hängen vom Rechner ab und sind deshalb nicht Teil der Aussage; wichtig ist das Verfahren und der Vergleich vor und nach einer Maßnahme. Die Zählung vor und nach `git gc` zeigt die Wirkung der Pflege: Das Skript erzeugt dafür vorab 30 lokale Commits, die zunächst als lose Objekte vorliegen; `git gc` führt sie in die Pack-Datei zusammen, die Zahl der losen Objekte sinkt auf null. Der abschließende Aufruf von `git fsck --full` prüft die Konsistenz des Objektspeichers und gehört zu den Standardprüfungen vor Wartungsarbeiten (siehe auch Abschnitt 10.7).

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Werkzeuge (aus der Nachrecherche).** Die Dokumentation von `git maintenance` (abgerufen am 28.10.2026) beschreibt, dass sich damit regelmäßige Aufgaben wie das Packen von Objekten oder das Aktualisieren des Commit-Graphen als Hintergrundaufgaben einrichten lassen. Ob bei sehr großen Arbeitsbäumen zusätzlich ein Sparse Checkout (Modul 05) oder ein Dateisystem-Monitor sinnvoll ist, hängt vom Betriebssystem und der Git-Version ab; die konkrete Konfiguration ist hier nicht Teil der abgerufenen Quellen und gehört in die Dokumentation der eingesetzten Version.

### Typische Fehler und Stolperfallen.

Falsch ist, bei einem langsamen Repository sofort die Hardware oder die Netzwerkverbindung zu beschuldigen oder ungeprüft `git gc --aggressive` laufen zu lassen. Richtig ist, zuerst Größe und Objekte zu analysieren und dann gezielt zu handeln: Große Binärdateien gehören nach Git LFS (Abschnitt 10.2) oder gar nicht ins Repository, und eine bereits eingecheckte Altlast wird nur durch ein Umschreiben der Historie wirklich entfernt (Modul 06 und Modul 08), was alle Klone betrifft und abgestimmt werden muss. Zweiter Fehler: Ein flacher Klon als Dauerlösung für Menschen, die Historie brauchen. Dritter Fehler: Messen nach einer einzigen Messung ohne Wiederholung. Vierter Fehler: Dateien aus der Historie „löschen“ und dann wundern, dass das Repository gleich groß bleibt; ein normaler Löschcommit entfernt nichts aus der Vergangenheit.

### Praxisbezug.

In Teams wird die Größe des Repositories regelmäßig kontrolliert, zum Beispiel in der Nachtpipeline, und eine Schwelle löst eine Warnung aus. Bei Pipelines sind flache Klone oder Partial Clones Standard; wo Historie nötig ist, lädt man gezielt nach. Beim Onboarding neuer Kolleginnen und Kollegen lohnt sich die Frage, ob ein Partial Clone ausreicht, bevor ein Gigabyte-Klon mehrere Stunden dauert.

## 10.2 Git LFS Betrieb

Git speichert jede Version jeder Datei vollständig oder als Delta in der Historie. Für Quelltext ist das ideal, für große Binärdateien (Grafiken, Messdaten, Videos, Modelle) nicht, weil sie sich kaum komprimieren lassen und jede Änderung eine neue, riesige Version erzeugt. *Git LFS* (Large File Storage) löst das, indem es statt der Datei nur eine kleine *Zeigerdatei* (Pointer) ins Repository einträgt. Die eigentlichen Daten liegen in einem separaten Speicher des Servers und werden beim Auschecken nachgeladen. Welche Dateien so behandelt werden, legt ein Muster in der Datei `.gitattributes` fest; `git lfs track "*.bin"` trägt es dort ein. Beim Hinzufügen ersetzt ein Filter die Datei durch den Pointer (clean-Filter), beim Auschecken tauscht er den Pointer gegen die Daten (smooth-Filter).

Der Betrieb von LFS hat drei Seiten, die man verstehen muss. Erstens *Speicher und Quotas*: Die Daten liegen nicht in der Git-Historie, sondern im LFS-Speicher der Plattform, und viele Plattformen begrenzen Speicher und Übertragungsvolumen pro Konto oder Organisation. Konkrete Grenzen und Preise ändern sich und sind hier nicht aus einer abgerufenen Quelle belegt; sie gehören vor der Planung in der aktuellen Dokumentation der eingesetzten Plattform nachgelesen. Zweitens *Migration*: Bestehende Repositories, in denen große Dateien bereits in der Historie liegen, lassen sich mit `git lfs migrate` umstellen. Mit `migrate info` analysiert man vorab, welche Dateitypen wie viel Platz belegen, mit `migrate import` schreibt man die Historie um, sodass die Dateien als Pointer erscheinen. Drittens *Wiederherstellbarkeit*: Ein Backup des Git-Repositories allein sichert die LFS-Daten nicht mit (Abschnitt 10.7).

Das folgende Beispiel durchläuft eine Migration. Es startet mit einem Repository, in dem eine drei Megabyte große Datei in der Historie liegt, analysiert es, stellt es mit `migrate import` um und zeigt, dass danach statt der Datei ein Pointer in Git liegt. Weil `migrate import` die Historie umschreibt, entstehen neue Commit-Hashes; der Befehl rührt deshalb gemeinsam genutzte Branches nur an, wenn man ihn ausdrücklich dafür aufruft (hier mit `--everything`). Am Ende werden die alten Objekte entfernt, denn solange die alte Historie noch im Reflog hängt, wird das Repository nicht kleiner.

```bash
git init -q -b main lfs && cd lfs
git config user.name Dev; git config user.email dev@example.org
git lfs install --local >/dev/null
echo "Hallo" > README.md && git add . && git commit -q -m "feat: Start"
head -c 3000000 /dev/urandom > modell.bin
git add . && git commit -q -m "feat: Modell"
echo "kleine Aenderung" >> README.md && git commit -q -am "docs: Hinweis"
echo "--- vorher: .git gross"
git count-objects -vH | grep -E "^size:"
echo "--- Analyse"
git lfs migrate info --everything 2>&1 | grep -E "^\*\.(bin|md)"
echo "--- Migration"
git lfs migrate import --everything --include="*.bin" >/dev/null 2>&1
echo "--- Pointer in Git statt der Datei:"
git cat-file -p HEAD~1:modell.bin
echo "--- LFS verwaltet:"
git lfs ls-files
echo "--- aufraeumen und nachmessen"
git reflog expire --expire=now --all && git gc -q --prune=now
git count-objects -vH | grep -E "^size-pack:"
git lfs fsck 2>&1 | tail -1
cat .gitattributes
```

Die Analyse nennt pro Dateimuster die Gesamtgröße und zeigt, dass `*.bin` den Speicher dominiert. Nach dem Import enthält Git an der Stelle der Datei nur noch einen Pointer aus drei Zeilen: die Version der Pointer-Spezifikation, die SHA-256-Prüfsumme der Daten (`oid`) und die Größe. Die echten Daten liegen jetzt im lokalen LFS-Speicher unterhalb von `.git/lfs`. Nach Reflog-Bereinigung und `git gc` schrumpft der Pack auf wenige Kilobyte, `git lfs fsck` bestätigt, dass Pointer und Daten zusammenpassen, und die Datei `.gitattributes` enthält die Regel für `*.bin`. Die Reihenfolge ist wichtig: Erst die Regel anlegen, dann Dateien hinzufügen, denn Dateien, die vorher ohne Regel committet wurden, liegen weiter als normale Blobs im Repository.

### Typische Fehler und Stolperfallen.

Falsch ist, `git lfs track` auszuführen und zu glauben, damit seien alte Commits bereinigt. Das Kommando wirkt nur auf neue Hinzufügungen; für die Historie braucht es `migrate import`. Zweiter Fehler: LFS in einem Klon zu verwenden, in dem es nicht installiert ist. Dann sieht man statt der Datei die Pointer-Textdatei, Programme melden „Datei beschädigt“, und niemand versteht warum. Der Hinweis lautet: `git lfs install` ausführen und `git lfs pull` holen. Dritter Fehler: Migration auf einem geteilten Branch ohne Ankündigung; alle Kolleginnen und Kollegen müssen anschließend frisch klonen oder ihre Branches mit Sorgfalt neu aufsetzen. Vierter Fehler: Quotas ignorieren; wenn das Kontingent erschöpft ist, scheitern Pushes oder Klone mit Fehlermeldungen der Plattform, und die Pipeline fällt aus.

### Praxisbezug.

Typische LFS-Kandidaten sind Design-Dateien, Testdaten in Binärformat, Dokumente und vortrainierte Modelle. Ein praktischer Gewinn ist, dass Pipelines die LFS-Daten selektiv laden können, etwa nur die für den Build nötigen. Wer LFS einführt, sollte die Regeln im Repository dokumentieren (welche Muster, welche Dateigröße als Schwelle), im CI die Existenz der Regel prüfen und die Kapazität der Plattform beobachten.

## 10.3 Submodules und Subtrees

Manchmal soll ein Projekt den Code eines anderen Repositories enthalten, ohne ihn zu kopieren und von Hand zu pflegen: eine gemeinsame Bibliothek, ein Design-System, ein Vorlagenpaket. Git bietet zwei Mechanismen. Ein *Submodul* verweist auf einen bestimmten Commit eines anderen Repositories. Im Haupt-Repository liegt nur dieser Verweis (ein sogenannter Gitlink, im Index als Modus 160000 geführt, siehe Modul 08) und die Datei `.gitmodules` mit der URL. Ein *Subtree* dagegen kopiert den Inhalt des anderen Repositories samt Historie in ein Unterverzeichnis des Haupt-Repositories; danach ist alles ein gewöhnliches Verzeichnis, und wer klont, merkt von der Herkunft nichts.

Die Wahl hat Folgen für den Alltag. Submodule halten die Repositories sauber getrennt und machen die Version der Abhängigkeit exakt reproduzierbar, aber sie verlangen von allen Beteiligten zusätzliche Handgriffe: Beim Klonen muss man `--recurse-submodules` angeben oder anschließend `git submodule update --init` ausführen (die Dokumentation zu `git submodule` und `git clone` beschreibt beides, abgerufen am 28.10.2026), und beim Wechseln von Branches aktualisieren sich Submodule nicht von selbst. Subtrees sind für Nutzer unsichtbar und brauchen keine zusätzlichen Befehle beim Klonen, dafür bläht die kopierte Historie das Repository auf, und Änderungen an der Abhängigkeit zurück in das Ursprungs-Repository zu bringen ist aufwendiger. Als Faustregel eignen sich Submodule für Abhängigkeiten, die man bewusst auf einem festen Stand hält und selten ändert, und Subtrees für Code, den ein Team als Teil seines Projekts betrachtet und nur gelegentlich mit der Quelle abgleicht. Oft ist die einfachste Lösung ein Paketmanager (npm, Maven, pip), der Abhängigkeiten über Versionsnummern verwaltet; Submodule und Subtrees sind dann die Ausnahme für Quellcode, der nicht als Paket vorliegt.

Das Beispiel zeigt beide Wege nebeneinander mit demselben Bibliotheks-Repository und demonstriert den häufigsten Fehler bei Submodulen: Die Bibliothek bekommt einen neuen Commit, das Haupt-Repository zeigt aber weiter auf den alten Stand, bis jemand den Verweis ausdrücklich aktualisiert. Mit `git submodule update --remote` wird der neue Stand geholt, und danach muss der geänderte Verweis wie jede andere Änderung committet werden. Die Option `protocol.file.allow` ist nur nötig, weil das Beispiel Repositories im lokalen Dateisystem verbindet; bei echten URLs entfällt sie. Der Befehl `git subtree` gehört zu den mitgelieferten Zusatzwerkzeugen und war im Test vorhanden; ob er in jeder Installation enthalten ist, sollte man vorab prüfen.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main lib && (cd lib && git config user.name Dev && git config user.email dev@example.org \
  && echo "v1" > lib.txt && git add . && git commit -q -m "feat: lib v1")

echo "=== Submodul"
git init -q -b main haupt && cd haupt
git config user.name Dev; git config user.email dev@example.org
echo "App" > app.txt && git add . && git commit -q -m "feat: App"
git submodule add -q ../lib vendor/lib && git commit -q -m "feat: Bibliothek als Submodul"
git ls-files --stage vendor/lib | cut -c1-48
cat .gitmodules
(cd ../lib && echo "v2" > lib.txt && git commit -q -am "feat: lib v2")
echo "--- Haupt-Repository zeigt noch auf v1:"
cat vendor/lib/lib.txt
git submodule update --remote --quiet
echo "--- nach update --remote:"
cat vendor/lib/lib.txt
git status --short
git commit -q -am "chore: Bibliothek auf v2 gehoben"
echo "--- Frischer Klon ohne/mit --recurse-submodules"
cd .. && git clone -q file://$PWD/haupt k1 && ls k1/vendor/lib | wc -l
git clone -q --recurse-submodules file://$PWD/haupt k2 && ls k2/vendor/lib

echo "=== Subtree"
git init -q -b main haupt2 && cd haupt2
git config user.name Dev; git config user.email dev@example.org
echo "App" > app.txt && git add . && git commit -q -m "feat: App"
git subtree add -q --prefix=vendor/lib ../lib main --squash 2>/dev/null
cat vendor/lib/lib.txt
(cd ../lib && echo "v3" > lib.txt && git commit -q -am "feat: lib v3")
git subtree pull -q --prefix=vendor/lib ../lib main --squash -m "chore: lib aktualisiert" >/dev/null 2>&1
cat vendor/lib/lib.txt
git clone -q file://$PWD ../k3 && ls ../k3/vendor/lib
```

Beim Submodul zeigt `git ls-files --stage` den Eintrag mit Modus 160000: Git speichert hier keinen Dateiinhalt, sondern den Commit-Hash der Bibliothek. Die Datei `.gitmodules` hält Pfad und URL. Nachdem die Bibliothek weiterentwickelt wurde, sieht das Hauptprojekt weiterhin „v1“, bis `update --remote` den neuen Stand holt; `git status` zeigt dann eine geänderte Zeile für das Submodul, die man committet. Der Klon ohne `--recurse-submodules` hat ein leeres Verzeichnis `vendor/lib` (die Zeile mit der Zahl 0), der Klon mit der Option enthält die Datei. Beim Subtree gibt es diese Unterscheidung nicht: Der Inhalt liegt als gewöhnliche Dateien im Repository, der Klon enthält ihn sofort, und das Aktualisieren geschieht mit `git subtree pull`.

### Typische Fehler und Stolperfallen.

Falsch ist, im Submodul-Verzeichnis zu arbeiten, als wäre es ein Unterordner. Dort befindet sich ein eigenes Repository, meist im Zustand detached HEAD (siehe Abschnitt 10.4); Commits dort gehen verloren, wenn man sie nicht auf einen Branch bringt und in das Ursprungs-Repository pusht. Richtig ist, Änderungen am Submodul im Submodul-Repository zu committen und zu pushen und danach im Hauptprojekt den neuen Verweis zu committen. Zweiter Fehler: Den Verweis im Hauptprojekt pushen, bevor der Submodul-Commit gepusht wurde. Dann zeigt der Verweis auf einen Commit, den niemand sonst holen kann, und der Klon scheitert mit „not our ref“. Dritter Fehler: In der Pipeline vergessen, Submodule mitzuladen, und Zugriff auf deren URLs nicht freizugeben (Modul 09, Abschnitt 9.8). Vierter Fehler: Subtree-Aktualisierung ohne `--squash` beziehungsweise ohne klare Regel, wie mit der Historie der Abhängigkeit umzugehen ist; das Repository wird unnötig groß.

### Praxisbezug.

Ob Submodul oder Subtree, ist eine Teamentscheidung, die man schriftlich festhält, weil sie jeden Klon betrifft. In der Praxis sieht man Submodule für Firmware-Bibliotheken und gemeinsame Konfigurationen, Subtrees für Code, der mit dem Hauptprojekt zusammen wächst. Wer neu in ein Projekt mit Submodulen kommt, bekommt die Befehle zum Klonen und Aktualisieren in die README, denn die Fehlerbilder (leeres Verzeichnis, falscher Stand) sind die häufigsten Supportfragen.

## 10.4 Typische Fehlerbilder

Im Alltag wiederholen sich drei Fehlerbilder so oft, dass man sie auswendig erkennen sollte: ein *detached HEAD*, ein abgelehnter Push wegen *non-fast-forward* und ein *Merge-Konflikt*. Gemeinsam ist ihnen, dass die Meldung zunächst bedrohlich klingt, die Situation aber fast immer ohne Datenverlust zu beheben ist, wenn man das Richtige tut und das Falsche lässt. Das Falsche ist in allen drei Fällen derselbe Impuls: etwas erzwingen, löschen oder neu klonen, bevor man verstanden hat, was los ist. Das Richtige beginnt immer mit der Diagnose: `git status`, `git log --oneline --graph --all` und gegebenenfalls `git reflog` (Modul 06).

Ein *detached HEAD* bedeutet, dass HEAD nicht auf einen Branch zeigt, sondern direkt auf einen Commit, etwa nach dem Auschecken eines Tags, eines Hashes oder in einer Pipeline. Das ist an sich harmlos und für Lesezugriff ideal. Gefährlich wird es, wenn man in diesem Zustand Commits erzeugt: Sie hängen an keinem Branch, und beim Wechsel zu einem Branch sind sie nur noch über das Reflog erreichbar. Ein *non-fast-forward*-Fehler entsteht, wenn der Remote-Branch Commits enthält, die der lokale nicht kennt; Git lehnt den Push ab, weil er sonst fremde Arbeit überschreiben würde. Die richtige Reaktion ist, die Änderungen zu holen und zu integrieren (Merge oder Rebase, Modul 03 und 04) und danach zu pushen. Ein Force-Push ist nur dann die Lösung, wenn man den eigenen Branch bewusst umgeschrieben hat, und dann mit `--force-with-lease`, das fremde Änderungen nicht überfährt (Modul 06). Ein *Merge-Konflikt* schließlich ist keine Fehlfunktion, sondern die Aufforderung, zwei Änderungen an derselben Stelle fachlich zusammenzuführen. Git markiert die Stellen in den Dateien, und man entscheidet. Wer unsicher ist, bricht mit `git merge --abort` ab und kehrt zum Zustand vor dem Merge zurück.

Das Beispiel erzeugt alle drei Situationen nacheinander und zeigt jeweils Diagnose und sichere Behebung. Es beginnt mit der Rettung von Commits aus einem detached HEAD, danach folgt ein abgelehnter Push und seine Behebung, am Ende ein Konflikt, den das Skript erst auflistet und dann bewusst auflöst. Die Kommentare im Skript ordnen die Schritte.

```bash
git init -q --bare -b main server.git
git clone -q server.git alice 2>/dev/null
cd alice && git config user.name Alice && git config user.email alice@example.org
echo "Titel: Entwurf" > text.txt && git add . && git commit -q -m "feat: Start" && git push -q origin main
git tag v1
echo "=== 1. detached HEAD: Commit entsteht ohne Branch"
git checkout -q v1 2>/dev/null
git status | head -1
echo "Idee" >> text.txt && git commit -q -am "feat: Idee im detached HEAD"
GEFAHR=$(git rev-parse --short HEAD)
git switch -q main
echo "Branch main kennt den Commit nicht:"; git branch --contains $GEFAHR | wc -l
echo "Reflog zeigt ihn noch:"; git reflog | sed -n '1,3p' | cut -c1-60
git branch rettung $GEFAHR && echo "gerettet auf Branch rettung:" && git log --oneline -1 rettung

echo "=== 2. non-fast-forward"
cd .. && git clone -q server.git bob && cd bob && git config user.name Bob && git config user.email bob@example.org
echo "Bobs Zeile" >> text.txt && git commit -q -am "feat: Bob" && git push -q origin main
cd ../alice && git switch -q main
echo "Alices Zeile" >> text.txt && git commit -q -am "feat: Alice"
git push origin main 2>&1 | grep -E "rejected|hint" | head -2
echo "--- Diagnose: was hat der Server, was ich nicht habe?"
git fetch -q && git log --oneline main..origin/main
echo "--- Behebung: integrieren, dann pushen (Konflikt moeglich)"
git pull --no-rebase origin main 2>&1 | grep -E "CONFLICT|Merge made|Automatic" | head -2

echo "=== 3. Konflikt"
git diff --name-only --diff-filter=U
git status --short | head -3
sed -n '1,8p' text.txt
cat > text.txt <<'EOF'
Titel: Entwurf
Bobs Zeile
Alices Zeile
EOF
git add text.txt && git commit -q -m "merge: Zeilen von Alice und Bob zusammengefuehrt"
git push -q origin main && echo "Push gelungen"
git log --oneline --graph | cut -c1-60
```

Im ersten Teil meldet `git status` den Zustand „HEAD detached“. Der in diesem Zustand erzeugte Commit gehört zu keinem Branch, `git branch --contains` findet nach dem Zurückwechseln auf `main` keinen Branch (Ausgabe 0), das Reflog nennt ihn aber noch, und ein neuer Branch `rettung` an dieser Stelle macht ihn wieder dauerhaft erreichbar. Im zweiten Teil kommt Bob mit seinem Push zuerst; Alices Push wird mit „rejected“ und einem Hinweis abgelehnt, dass der Remote-Branch Änderungen enthält. Die Diagnose `git log main..origin/main` zeigt genau die Commits, die Alice fehlen. Im dritten Teil führt das Zusammenführen zu einem Konflikt, weil beide dieselbe Zeile am Dateiende verändert haben: `git diff --name-only --diff-filter=U` listet die ungelösten Dateien, die Konfliktmarker `<<<<<<<`, `=======` und `>>>>>>>` zeigen beide Varianten, und das Skript ersetzt die Datei durch die fachlich richtige Fassung, markiert sie mit `git add` als gelöst und schließt den Merge ab. Erst danach gelingt der Push.

### Typische Fehler und Stolperfallen.

Falsch ist, bei „rejected“ sofort `git push --force` zu verwenden. Damit überschreibt man Bobs Arbeit auf dem Server. Richtig: fetch, Unterschiede ansehen, integrieren, pushen. Zweiter Fehler: Konfliktmarker mit einzuchecken, weil man die Datei nur gespeichert, aber nicht geprüft hat; ein einfacher Schutz ist eine Prüfung auf `<<<<<<<` im Pre-commit-Hook (Modul 09, Abschnitt 9.3). Dritter Fehler: Im detached HEAD weiterzuarbeiten, ohne einen Branch anzulegen, und später nach den „verschwundenen“ Commits zu suchen. Vierter Fehler: Bei Konflikten pauschal „meine Version“ zu nehmen (`git checkout --ours`), ohne zu lesen, was die andere Seite geändert hat.

### Praxisbezug.

Support-Anfragen bei Git bestehen zu einem großen Teil aus genau diesen drei Mustern. Teams, die dafür eine kurze, schrittweise Anleitung im Wiki haben (Abschnitt 10.8), sparen Zeit und verhindern Panikreaktionen. Ein wichtiger Teil guter Praxis ist, dass vor riskanten Schritten eine Sicherung existiert: ein Branch oder Tag auf dem aktuellen Stand kostet nichts und macht fast jede Korrektur umkehrbar.

## 10.5 Credential Troubleshooting

Wenn Git eine Anmeldung verlangt oder ablehnt, liegt die Ursache fast nie in Git selbst, sondern in den Zugangsdaten oder in dem, was sie dürfen. Es gibt zwei Wege, sich an einem Remote auszuweisen. Bei *SSH* (URL der Form `git@host:pfad`) authentifiziert ein Schlüsselpaar: Der private Schlüssel bleibt beim Benutzer, der öffentliche liegt beim Server. Bei *HTTPS* schickt Git Benutzername und Passwort oder, heute üblich, ein *Token* (ein anstelle des Passworts verwendetes, in Umfang und Laufzeit beschränktes Geheimnis). Damit man Tokens nicht bei jedem Befehl eintippen muss, nutzt Git einen *Credential Helper*, ein Hilfsprogramm, das Zugangsdaten speichert und liefert, zum Beispiel den Credential Manager von Windows oder den Schlüsselbund von macOS. Welche Helper aktiv sind, steht in der Konfiguration; die Dokumentation von `gitcredentials` konnte nicht abgerufen werden, die folgenden Aussagen beruhen daher auf dem Wissensstand und den Tests unten.

Die häufigsten Fehlerbilder lassen sich anhand der Meldung einordnen. „could not read Username ... terminal prompts disabled“ heißt: Git hat keine gespeicherten Zugangsdaten gefunden und darf nicht nachfragen, typisch in Pipelines. „Authentication failed“ heißt: Der Server hat die Zugangsdaten abgelehnt, etwa weil das Token abgelaufen oder widerrufen ist. „error: 403“ heißt: Die Identität ist bekannt, aber es fehlt die Berechtigung für diese Aktion, zum Beispiel weil das Token nur lesen darf. „repository not found“ kann bedeuten, dass das Repository nicht existiert oder dass der Server aus Sicherheitsgründen nicht verrät, dass man keinen Zugriff hat; die Plattformen verhalten sich hier unterschiedlich. Bei SSH sind es „Permission denied (publickey)“ (der Server kennt keinen der angebotenen Schlüssel) und Probleme mit dem SSH-Agenten, der keine Schlüssel geladen hat.

Um diese Meldungen sicher zu reproduzieren, nutzt das Beispiel einen winzigen lokalen Testserver (ein Python-Skript), der nur HTTP-Statuscodes zurückgibt: ohne Zugangsdaten 401, mit dem Passwort „abgelaufen“ wieder 401, mit „readonly“ den Status 403. Das ist keine Git-Funktionalität, sondern ein Nachbau der Antworten einer Plattform, genügt aber, um zu sehen, wie Git reagiert und wie man die Ursache eingrenzt. Das Skript prüft zuerst, welche Helper konfiguriert sind, und zeigt dann den Weg vom fehlenden über das gespeicherte, aber ungültige Token bis zum Rechtefehler. Die Umgebungsvariable `GIT_TERMINAL_PROMPT=0` verbietet Nachfragen, damit Git wie in einer Pipeline reagiert und nicht wartet.

```bash
cat > server.py <<'EOF'
import http.server, sys, base64
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        a = self.headers.get("Authorization")
        if not a:
            self.send_response(401); self.send_header("WWW-Authenticate", 'Basic realm="git"'); self.end_headers(); return
        user, pw = base64.b64decode(a.split()[1]).decode().split(":", 1)
        if pw == "abgelaufen":
            self.send_response(401); self.send_header("WWW-Authenticate", 'Basic realm="git"')
        elif pw == "readonly":
            self.send_response(403)
        else:
            self.send_response(404)
        self.end_headers()
    def log_message(self, *args): pass
s = http.server.HTTPServer(("127.0.0.1", 0), H)
open(sys.argv[1], "w").write(str(s.server_port))
s.serve_forever()
EOF
python3 server.py port.txt & SRV=$!
sleep 1; P=$(cat port.txt); URL=http://127.0.0.1:$P/projekt.git
git init -q -b main klon && cd klon
export GIT_TERMINAL_PROMPT=0
S="store --file=$OLDPWD/creds"

echo "=== 1. Welche Helper sind aktiv, und woher kommen sie?"
git config --show-origin --get-all credential.helper || echo "(kein Helper konfiguriert)"
echo "=== 2. Keine Zugangsdaten gespeichert"
git ls-remote $URL; echo "rc=$?"
echo "=== 3. Token speichern, das inzwischen abgelaufen ist"
printf 'protocol=http\nhost=127.0.0.1:%s\nusername=bot\npassword=abgelaufen\n' $P | git -c credential.helper="$S" credential approve
echo "Gespeichert (Passwort verdeckt):"; sed 's/:[^:@]*@/:***@/' ../creds
git -c credential.helper="$S" ls-remote $URL; echo "rc=$?"
echo "Datei danach: $(wc -c < ../creds) Byte (Git hat die abgelehnten Zugangsdaten selbst entfernt)"
echo "=== 4. Token gueltig, aber ohne Schreib- bzw. Leserecht"
git ls-remote http://bot:readonly@127.0.0.1:$P/projekt.git; echo "rc=$?"
echo "=== 5. SSH-Schluessel: Fingerabdruck und Kommentar pruefen"
ssh-keygen -q -t ed25519 -N "" -C "bot@ci 2026-10" -f ../id_bot
ssh-keygen -lf ../id_bot.pub | cut -d' ' -f1,3-
kill $SRV
```

Schritt 1 zeigt, wie man die Ursache von „falschen Zugangsdaten“ zuerst sucht: `git config --show-origin --get-all credential.helper` nennt alle Helper und die Datei, aus der sie stammen. Mehrere Helper aus verschiedenen Ebenen (System, Benutzer, Repository) sind ein klassischer Grund für Verwirrung, weil ein alter Eintrag aus einem Helper immer wieder auftaucht. Schritt 2 zeigt die Pipeline-Situation: ohne gespeicherte Daten und mit verbotener Nachfrage endet der Befehl mit der Meldung über das Nicht-Lesen-Können des Benutzernamens und dem Exit-Code 128. In Schritt 3 liegt ein Token im Speicher, das der Server nicht mehr akzeptiert. Die Meldung wechselt zu „Authentication failed“, und die Datei ist anschließend leer, weil Git nach einer Ablehnung den Eintrag aus dem Helper entfernt (es ruft dazu intern `credential reject` auf). Das erklärt, warum nach einem Fehlschlag die nächste Anmeldung wieder nach Zugangsdaten fragt. Schritt 4 liefert den Status 403, Git meldet, dass die Anfrage mit einem Fehler beantwortet wurde, und der Hinweis lautet: Zugangsdaten korrekt, Berechtigung falsch. Schritt 5 prüft einen SSH-Schlüssel; ein sinnvoller Kommentar im Schlüssel (hier Zweck und Ausstelldatum) hilft später, abgelaufene oder vergessene Schlüssel zuzuordnen.

> **Merke**
>
> Die Meldung entscheidet über die Spur: „could not read Username“ heißt, es sind *keine* Zugangsdaten verfügbar. „Authentication failed“ heißt, es sind welche da, aber der Server lehnt sie ab (abgelaufen, widerrufen, falscher Benutzer). „403“ heißt, sie sind gültig, aber *berechtigen nicht* zu dieser Aktion. Wer diese drei Fälle trennt, vermeidet die meisten Fehlsuchen.

### Typische Fehler und Stolperfallen.

Falsch ist, ein Token ungeprüft zu „erneuern“, ohne zu klären, welcher Fehler vorliegt; bei einem 403 hilft ein neues Token mit denselben Rechten nicht. Richtig ist zu unterscheiden (siehe Merkkasten) und bei Rechtefehlern den Umfang des Tokens zu prüfen. Zweiter Fehler: Zugangsdaten in der URL zu hinterlegen (`https://user:token@host/...`). Sie landen in `.git/config`, in Prozesslisten und Protokollen (Modul 08, Abschnitt 8.4). Dritter Fehler: Ein Token ohne Ablaufdatum zu erzeugen, weil das bequemer ist; besser sind kurze Laufzeiten und ein Kalendereintrag für die Erneuerung. Vierter Fehler: Mehrere Credential Helper parallel zu betreiben, ohne es zu wissen. Fünfter Fehler: Bei SSH den falschen Schlüssel zu verwenden, weil der Agent mehrere anbietet; mit `ssh -v` sieht man, welche Schlüssel angeboten werden (der Aufruf erfordert ein erreichbares Ziel und wurde im Test nicht ausgeführt).

### Praxisbezug.

Beim Wechsel von Passwörtern zu Tokens und bei Rotation (Modul 08) treten diese Fehler gehäuft auf, vor allem montagmorgens nach einem Ablauf am Wochenende. Hilfreich sind drei Dinge: ein Verzeichnis der Tokens mit Ablaufdatum und Verantwortlichen, ein kurzes Diagnoseskript, das die Helper und den URL-Typ des Remotes ausgibt, und die Regel, dass Pipelines eigene Service-Accounts mit eng begrenzten Tokens verwenden (Modul 09, Abschnitt 9.7).

## 10.6 Audit und Reports

Ein Audit beantwortet die Frage: Wird so gearbeitet, wie wir es vereinbart haben? Git liefert dafür eine reiche Datenbasis, denn die Historie enthält Autoren, Zeitpunkte, Branches, Merges, Nachrichten und gegebenenfalls Signaturen. Was nicht in Git steht, sind Ereignisse der Plattform: Reviews, Freigaben, Pipeline-Ergebnisse. Sie holt man über die API der Plattform (zum Beispiel mit dem Kommandozeilenwerkzeug `gh`); die konkreten Endpunkte sind hier nicht aus einer abgerufenen Quelle belegt und wurden in der Testumgebung nicht ausgeführt. Berichte, die sich ausschließlich auf Git stützen, kommen ohne Zugriff auf die Plattform aus und sind deshalb ideal für wiederholbare Prüfungen in Skripten.

Wichtig ist, *Governance-Abweichungen* sichtbar zu machen, nicht Menschen zu überwachen. Gute Kennzahlen beschreiben den Zustand des Repositories, nicht die Leistung einzelner Personen: das Alter offener Branches (lange nicht gemergte Branches sind Merge-Risiko und Zeichen mangelnder Pflege), das Verhältnis von Merge-Commits zu Direktcommits auf dem Hauptbranch, die Zahl nicht signierter Commits, falls Signaturen verlangt sind (Modul 08), Commits, die der Nachrichtenkonvention nicht entsprechen (Modul 09), und die Aktivität pro Zeitraum. Für Autorenstatistiken gelten in Europa die Regeln der DSGVO; Namen und E-Mail-Adressen in Commits sind personenbezogene Daten, deren Auswertung einen Zweck und eine Rechtsgrundlage braucht (keine Rechtsberatung, vergleiche Modul 08, Abschnitt 8.8).

Das folgende Skript erzeugt einen Bericht für ein Beispiel-Repository. Damit die Ergebnisse unabhängig vom Ausführungstag sind, legt das Beispiel Commits mit festen Zeitstempeln an und rechnet das Alter gegen einen festen Stichtag. Das Skript listet für jeden Branch das Alter des letzten Commits und markiert Branches ab 30 Tagen, zählt Direktcommits auf dem Hauptbranch (Commits mit nur einem Elternteil auf dem ersten-Eltern-Pfad, wenn Merges erwartet werden), und prüft die Konvention der Nachrichten. Zusätzlich gibt es eine Zeile für Commits pro Monat.

```bash
git init -q --bare -b main server.git
git clone -q server.git w 2>/dev/null && cd w
git config user.name Dev; git config user.email dev@example.org
c() { GIT_AUTHOR_DATE="$1T12:00:00" GIT_COMMITTER_DATE="$1T12:00:00" git commit -q "${@:2}"; }
echo 1 > a && git add . && c 2026-08-03 -m "feat: Start"
git switch -q -c feature/alt && echo 2 > b && git add . && c 2026-08-10 -m "feat: alte Idee"
git switch -q main && git switch -q -c feature/neu && echo 3 > c && git add . && c 2026-10-20 -m "feat: neue Idee"
git switch -q main && echo 4 > d && git add . && c 2026-09-02 -m "Quick fix ohne Konvention"
git merge -q --no-ff feature/neu -m "Merge feature/neu" 2>/dev/null
git push -q origin --all 2>/dev/null

cat > ../audit.sh <<'EOF'
#!/bin/sh
# Audit-Bericht nur aus Git-Daten. Aufruf: audit.sh <Stichtag JJJJ-MM-TT> <Hauptbranch>
stichtag=$1; haupt=${2:-main}
heute=$(date -u -d "$stichtag" +%s)
echo "Bericht zum Stichtag $stichtag"
echo "== 1. Branch-Alter (Schwelle 30 Tage)"
git for-each-ref --format='%(refname:short) %(committerdate:short)' refs/remotes/origin | grep -v HEAD | while read name datum; do
  alter=$(( (heute - $(date -u -d "$datum" +%s)) / 86400 ))
  [ $alter -ge 30 ] && marke="ALT" || marke="ok"
  printf "%-20s %4s Tage  %s\n" "$name" "$alter" "$marke"
done
echo "== 2. nicht in $haupt gemergte Branches"
git branch -r --no-merged "origin/$haupt" | grep -v HEAD | sed 's/^ */  /'
echo "== 3. Direktcommits auf $haupt (ohne Merge, erster-Eltern-Pfad)"
git log --first-parent --no-merges --format='  %h %s' "origin/$haupt"
echo "== 4. Commits ohne Nachrichtenkonvention"
git log --no-merges --format='%h %s' "origin/$haupt" | grep -Ev '^[0-9a-f]+ (feat|fix|docs|chore|refactor|test)(\([a-z0-9-]+\))?!?: ' | sed 's/^/  /'
echo "== 5. Commits pro Monat (Hauptbranch)"
git log --format='%cd' --date=format:%Y-%m "origin/$haupt" | sort | uniq -c | sed 's/^ */  /'
echo "== 6. Signaturstatus (G=gueltig, N=keine)"
git log --format='%G?' "origin/$haupt" | sort | uniq -c | sed 's/^ */  /'
EOF
git fetch -q origin
sh ../audit.sh 2026-10-28 main
```

Der Bericht gliedert sich in sechs Abschnitte. Die Branch-Liste nennt für jeden Branch das Alter seit dem letzten Commit und kennzeichnet den Branch `feature/alt` als alt (79 Tage). Der Abschnitt über nicht gemergte Branches nennt die offenen Entwicklungslinien. Die Direktcommits auf dem Hauptbranch finden sich über `git log --first-parent --no-merges`: Diese Option folgt auf `main` nur dem ersten Elternteil und blendet Merge-Commits aus, übrig bleiben Commits, die direkt dort entstanden sind, hier der „Quick fix“ und der Start-Commit des Projekts, der naturgemäß ebenfalls direkt entstanden ist. Der Abschnitt zur Konvention liefert denselben Commit noch einmal, weil er gegen die Regel verstößt, und die Auswertung der Signaturen zeigt, dass keiner der Commits signiert ist (Status N). Die Zahlen werden für ein Team erst dann aussagekräftig, wenn man sie über Zeit vergleicht; ein einmaliger Bericht zeigt Zustand, eine Reihe zeigt Trend.

> **Ergänzung aus der Nachrecherche**
>
> **Grenzen von Git-Daten.** Squash-Merges erzeugen auf dem Hauptbranch einen gewöhnlichen Commit ohne Merge-Eltern, sodass Direktcommits und Squash-Merges in Abschnitt 3 nicht unterscheidbar sind. Wer das braucht, wertet zusätzlich die Pull-Request-Daten der Plattform aus oder verlangt eine Referenz auf den Pull Request in der Commit-Nachricht (Modul 09, Abschnitt 9.5). Commit-Zeitstempel und Autoren lassen sich außerdem von den Beteiligten selbst festlegen und sind kein Beweismittel; Signaturen (Modul 08) erhöhen die Verlässlichkeit.

### Typische Fehler und Stolperfallen.

Falsch ist, Kennzahlen wie „Commits pro Person“ als Leistungsmaß zu verwenden. Sie belohnen kleine, häufige Commits und bestrafen Reviews, Planung und Pairing. Richtig sind Kennzahlen zum Zustand des Prozesses (Branch-Alter, Konventionsquote, Anteil geprüfter Merges). Zweiter Fehler: Berichte ohne Konsequenz: Ein Audit, dessen Ergebnis niemand liest, erzeugt nur Pflege-Aufwand. Dritter Fehler: Zeitberechnungen ohne festen Stichtag, sodass Berichte nicht reproduzierbar sind. Vierter Fehler: Die Berichte auf lokalen Branches auszuführen statt auf dem Stand des Remotes; maßgeblich ist, was auf dem Server liegt (`origin/...`, nach `git fetch`).

### Praxisbezug.

Solche Berichte laufen als geplante Pipeline (wöchentlich) und schreiben das Ergebnis in ein Artefakt oder eine Ticket-Notiz. Der Gewinn liegt weniger im einzelnen Bericht als in der Gewohnheit: Branches, die älter als ein Monat sind, werden besprochen und entweder gemergt oder gelöscht, und Abweichungen von der Konvention werden früh sichtbar, nicht erst beim Release.

## 10.7 Backup-nahe Prüfungen

Ein Remote-Repository auf einer Plattform ist kein Backup. Es ist eine Kopie, die gelöscht, überschrieben oder durch einen Ausfall unerreichbar werden kann. Entwicklerklone sind zwar faktisch viele Kopien, aber sie sind unvollständig (nicht jeder hat jeden Branch), uneinheitlich und nicht dafür gedacht. Eine *backup-nahe* Prüfung stellt deshalb zwei Fragen: Haben wir eine vollständige Kopie, die wir regelmäßig aktualisieren, und können wir daraus nachweislich wiederherstellen? Beides gehört zusammen. Ein Backup, das nie getestet wurde, ist eine Vermutung.

Git bietet dafür zwei Formen. Ein *Mirror-Klon* (`git clone --mirror`) kopiert alle Referenzen, also alle Branches, Tags und weitere Referenzen, in ein Bare-Repository, das sich mit `git remote update` (oder `git fetch --prune`) auf den aktuellen Stand bringen lässt. Ein *Bundle* (`git bundle create --all`) packt Referenzen und Objekte in eine einzelne Datei, die sich wie ein Remote klonen lässt und die man verschieben, archivieren und prüfen kann (`git bundle verify`). Beides sichert nur, was Git kennt. Nicht enthalten sind serverseitige Hooks und Einstellungen, Zugriffsrechte, Schutzregeln, Issues, Pull Requests, Wikis und LFS-Daten (Abschnitt 10.2), die gesondert gesichert werden müssen (für LFS: alle Objekte mit `git lfs fetch --all` holen und mitsichern).

Das Beispiel zeigt einen vollständigen Zyklus: Sicherung per Mirror, Sicherung per Bundle, Beschädigung der Quelle durch gelöschten Branch (als Simulation eines Unfalls), Wiederherstellung aus dem Bundle und der abschließende *Restore-Test*, der die Referenzen von Original und Wiederherstellung vergleicht. Der Vergleich stützt sich auf die Liste aller Referenzen mit ihren Hashes; sind beide Listen identisch, ist die Wiederherstellung vollständig.

```bash
git init -q -b main original && cd original
git config user.name Dev; git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: eins"
git tag -a v1.0.0 -m "1.0.0"
git switch -q -c feature/x && echo 2 > b && git add . && git commit -q -m "feat: zwei"
git switch -q main
cd ..
echo "=== 1. Mirror-Sicherung und Aktualisierung"
git clone -q --mirror original sicherung.git
(cd original && git switch -q -c feature/y && echo 3 > c && git add . && git commit -q -m "feat: drei")
git -C sicherung.git remote update --prune 2>&1 | grep -E "feature/y" | head -1
echo "=== 2. Bundle-Sicherung und Pruefung"
(cd original && git bundle create ../sicherung.bundle --all 2>&1 | tail -1)
git -C original bundle verify ../sicherung.bundle 2>&1 | grep -E "okay|complete history"
git bundle list-heads sicherung.bundle | cut -d' ' -f2
echo "=== 3. Unfall: Branch und Tag im Original verloren"
(cd original && git switch -q main && git branch -D -q feature/x feature/y && git tag -d v1.0.0 >/dev/null)
git -C original for-each-ref --format='%(refname)' | tr '\n' ' '; echo
echo "=== 4. Wiederherstellung aus dem Bundle"
git clone -q --mirror sicherung.bundle wiederhergestellt.git 2>/dev/null
echo "=== 5. Restore-Test: Referenzen und Integritaet"
git -C sicherung.git for-each-ref --format='%(objectname) %(refname)' | sort > ref-mirror.txt
git -C wiederhergestellt.git for-each-ref --format='%(objectname) %(refname)' | sort > ref-restore.txt
if diff -q ref-mirror.txt ref-restore.txt >/dev/null; then echo "Referenzen identisch ($(wc -l < ref-restore.txt) Stueck)"; else echo "ABWEICHUNG"; diff ref-mirror.txt ref-restore.txt; fi
git -C wiederhergestellt.git fsck --full 2>&1 | tail -1; echo "fsck rc=$?"
echo "Tag v1.0.0 im Restore: $(git -C wiederhergestellt.git rev-parse --short 'v1.0.0^{commit}')"
```

Der Mirror übernimmt beim Aktualisieren den inzwischen angelegten Branch `feature/y` (die gefilterte Zeile zeigt die Übernahme). Das Bundle enthält alle Referenzen; `bundle verify` meldet „okay“ und eine vollständige Historie (der Befehl muss innerhalb eines Repositories laufen, weil er prüft, ob die Voraussetzungen des Bundles dort vorhanden sind), und `list-heads` nennt die enthaltenen Referenzen. Der simulierte Unfall entfernt Branches und Tag im Original, übrig bleibt nur `refs/heads/main`. Die Wiederherstellung klont das Bundle als Mirror und enthält alle Referenzen wieder. Der Restore-Test vergleicht die sortierten Listen der Referenzen mit ihren Hashes; sind sie identisch, ist nicht nur die Zahl der Branches gleich, sondern jeder Branch zeigt auf denselben Commit. Die Prüfung `git fsck --full` bestätigt zusätzlich, dass der Objektspeicher in sich konsistent ist. Dass der Tag im Beispiel auf denselben Commit zeigt wie vorher, ist der eigentliche Beweis der Wiederherstellbarkeit.

### Typische Fehler und Stolperfallen.

Falsch ist, ein Bundle oder Mirror anzulegen und nie zu prüfen, ob man daraus wiederherstellen kann. Richtig ist ein regelmäßiger Restore-Test, am besten automatisiert. Zweiter Fehler: Nur das Git-Repository zu sichern, aber nicht LFS-Daten, Issues und Konfiguration. Dritter Fehler: Die Sicherung am selben Ort und mit denselben Zugangsdaten wie das Original zu halten; wer das Original löschen kann, kann dann auch die Sicherung löschen. Vierter Fehler: Mit `remote update --prune` im Mirror blind Löschungen zu übernehmen, sodass ein versehentlich gelöschter Branch auch aus der Sicherung verschwindet; deshalb hält man mehrere Generationen vor (zum Beispiel tägliche Bundles mit Datum im Dateinamen).

### Praxisbezug.

Die Eckwerte einer Sicherungsstrategie sind Häufigkeit (wie viel Arbeit darf höchstens verloren gehen), Aufbewahrung (wie weit zurück) und Restore-Dauer (wie lange darf die Wiederherstellung dauern). Aus diesen drei Werten ergibt sich, ob ein nächtliches Bundle genügt oder ein laufend aktualisierter Mirror nötig ist. Die Prüfung, ob ein Restore gelingt, gehört in das Runbook (Abschnitt 10.8) und wird zumindest vierteljährlich geübt.

## 10.8 Runbooks

Ein *Runbook* ist eine Anleitung für eine bekannte Störung: Was ist das Symptom, wie stellt man die Ursache fest, was tut man sofort, wen informiert man, wie geht der Betrieb wieder an, und woran erkennt man, dass alles wieder in Ordnung ist? Der Zweck liegt darin, dass die Antwort auf eine Störung nicht von der Erinnerung einer bestimmten Person abhängt und dass sie auch um drei Uhr nachts oder im Urlaub der Experten funktioniert. Gute Runbooks sind kurz, konkret und überprüft. Sie enthalten Befehle zum Kopieren statt Beschreibungen, nennen Entscheidungspunkte („wenn A, dann Schritt 3, sonst Schritt 5“) und enden mit einer Verifikation. Veraltete Runbooks sind gefährlicher als keine, weil sie falsche Sicherheit geben; deshalb gehört zu jedem eine Verantwortliche oder ein Verantwortlicher und ein Datum der letzten Prüfung.

Für Git-Störungen bieten sich Runbooks zu den Themen dieses Moduls an: Repository langsam oder zu groß (10.1), LFS-Quota erschöpft (10.2), Submodul-Probleme (10.3), Push abgelehnt oder Konflikt (10.4), Anmeldung schlägt fehl (10.5), Branch oder Tag versehentlich gelöscht (10.7), Secret im Repository (Modul 08, Abschnitt 8.2). Jedes folgt derselben Struktur, damit man sich im Ernstfall nicht erst orientieren muss. Zwei Punkte werden oft vergessen: die *Eskalation* (ab wann, an wen, mit welchen Informationen) und der *Wiederanlauf* (in welcher Reihenfolge fährt man Systeme wieder hoch, was ist zu prüfen, bevor die Teams weiterarbeiten dürfen).

Runbooks lassen sich teilweise automatisch absichern. Das erste Skript im Beispiel prüft, ob jedes Runbook im Verzeichnis `runbooks/` die geforderten Abschnitte enthält und ein Prüfdatum trägt, das nicht älter als 180 Tage ist; es kann als Pipeline-Schritt laufen. Das zweite Skript ordnet eine Fehlermeldung dem passenden Runbook zu und ist ein erster Baustein für ein Support-Werkzeug: Man gibt die Meldung ein und bekommt den Namen des Runbooks. Beide sind bewusst einfach gehalten, damit man sie verstehen und anpassen kann.

```bash
mkdir -p runbooks && cd runbooks
cat > push-abgelehnt.md <<'EOF'
# Runbook: Push abgelehnt (non-fast-forward)
Verantwortlich: Team Plattform
Geprueft: 2026-09-15
## Symptom
Git meldet "rejected ... (non-fast-forward)" oder "fetch first".
## Diagnose
git fetch && git log --oneline main..origin/main
## Sofortmassnahme
Nicht mit --force pushen. Aenderungen holen und integrieren: git pull --rebase (oder merge).
## Eskalation
Nach 30 Minuten ohne Loesung an die Plattform-Bereitschaft; Ausgabe von git status und git reflog -5 mitsenden.
## Wiederanlauf
Konflikte loesen, Tests lokal ausfuehren, dann pushen.
## Pruefung
git status meldet "up to date"; Pipeline des Branches ist gruen.
EOF
cat > anmeldung.md <<'EOF'
# Runbook: Anmeldung schlaegt fehl
Verantwortlich: Team Plattform
Geprueft: 2025-11-01
## Symptom
"Authentication failed" oder "could not read Username".
## Diagnose
git config --show-origin --get-all credential.helper
## Sofortmassnahme
Token pruefen und erneuern.
EOF
cd ..

cat > runbook-check.sh <<'EOF'
#!/bin/sh
# Prueft Runbooks auf Pflichtabschnitte und Aktualitaet. Aufruf: runbook-check.sh <Verzeichnis> <Stichtag>
dir=$1; heute=$(date -u -d "$2" +%s); status=0
for datei in "$dir"/*.md; do
  fehlt=""
  for abschnitt in Symptom Diagnose Sofortmassnahme Eskalation Wiederanlauf Pruefung; do
    grep -q "^## $abschnitt" "$datei" || fehlt="$fehlt $abschnitt"
  done
  datum=$(sed -n 's/^Geprueft: //p' "$datei")
  if [ -n "$datum" ]; then alter=$(( (heute - $(date -u -d "$datum" +%s)) / 86400 )); else alter=9999; fi
  meldung=""
  [ -n "$fehlt" ] && meldung="$meldung fehlende Abschnitte:$fehlt;"
  [ $alter -gt 180 ] && meldung="$meldung zuletzt geprueft vor $alter Tagen;"
  if [ -n "$meldung" ]; then echo "MANGEL  $(basename "$datei"):$meldung"; status=1; else echo "ok      $(basename "$datei")"; fi
done
exit $status
EOF
sh runbook-check.sh runbooks 2026-10-28; echo "Exit: $?"

cat > zuordnen.sh <<'EOF'
#!/bin/sh
# Ordnet eine Git-Fehlermeldung einem Runbook zu
case "$1" in
  *non-fast-forward*|*"fetch first"*)     echo "push-abgelehnt.md" ;;
  *"Authentication failed"*|*"could not read Username"*|*"403"*) echo "anmeldung.md" ;;
  *"dubious ownership"*)                  echo "safe-directory.md" ;;
  *"No names found"*)                     echo "flacher-klon.md" ;;
  *)                                      echo "unbekannt: an die Bereitschaft eskalieren" ;;
esac
EOF
for m in "! [rejected] main -> main (non-fast-forward)" "fatal: Authentication failed for 'https://x/'" "fatal: No names found, cannot describe anything." "irgendetwas Neues"; do
  printf "%-55s -> %s\n" "$m" "$(sh zuordnen.sh "$m")"
done
```

Das Beispiel legt zwei Runbooks an, von denen eines vollständig und aktuell ist und das andere in zwei Punkten nicht: Es fehlen die Abschnitte Eskalation, Wiederanlauf und Prüfung, und sein Prüfdatum liegt über 180 Tage zurück. Das Prüfskript ruft für jede Datei `grep` nach den Abschnittsüberschriften auf, liest das Datum aus der Zeile `Geprueft:` und rechnet das Alter gegen den Stichtag. Es endet mit Exit-Code 1, sobald ein Runbook beanstandet wird, und taugt damit als Pipeline-Schritt. Das Zuordnungsskript vergleicht die Meldung mit bekannten Mustern. Die vierte Meldung kennt es nicht und verweist auf die Eskalation, was der eigentliche Prüfstein ist: Jedes Runbook-System braucht einen Standardweg für das Unbekannte.

> **Merke**
>
> Ein Runbook ist erst dann gut, wenn es jemand nutzen kann, der das System nicht kennt. Der beste Test ist, es einmal von einer Person ausführen zu lassen, die nicht daran mitgeschrieben hat, und jede Rückfrage ins Runbook einzuarbeiten. Zu jedem Runbook gehören Verantwortliche, Prüfdatum und ein regelmäßiger Termin, an dem es geübt wird.

### Typische Fehler und Stolperfallen.

Falsch ist, Runbooks als lange Prosa zu schreiben, die man im Ernstfall nicht überfliegen kann. Richtig sind nummerierte Schritte mit Befehlen und klaren Entscheidungspunkten. Zweiter Fehler: Runbooks, die nur auf dem Rechner einer Person oder im gleichen System liegen, das gerade ausgefallen ist; wenn das Wiki im selben Verbund wie das Git-Repository liegt, ist es genau dann nicht erreichbar, wenn man es braucht. Besser ist eine Kopie im Repository selbst (wie im Beispiel) und eine ausgedruckte oder lokal gespeicherte Kurzfassung für die Kernstörungen. Dritter Fehler: Keine Eskalation festzulegen, sodass im Zweifel jede Person alleine weiterprobiert. Vierter Fehler: Ein Runbook nach einer Störung nicht zu aktualisieren; die Auswertung (Postmortem) sollte immer zu einer Änderung am Runbook führen.

### Praxisbezug.

Bei Störungen zählt neben der Technik die Kommunikation: Wer informiert Nutzer, wer entscheidet über eine Wiederherstellung aus dem Backup, wer gibt den Betrieb wieder frei? Das gehört in den Abschnitt zur Eskalation. In Unternehmen mit Bereitschaftsdienst sind Runbooks Teil der Übergabe, und neue Teammitglieder lernen die Störungsbilder, indem sie Runbooks in einer Übung durchspielen.

## Übungen zu Modul 10

**Aufgabe 1.** Nennen Sie drei Ursachen für ein langsames oder großes Repository und je eine passende Gegenmaßnahme.

**Aufgabe 2.** Warum wird ein Repository nicht kleiner, wenn man eine große Datei löscht und den Löschcommit pusht? Was wäre nötig?

**Aufgabe 3.** Worin unterscheiden sich Submodul und Subtree bei Klonen, Aktualisieren und Platzbedarf? Für welchen Fall würden Sie welchen Mechanismus wählen?

**Aufgabe 4.** Ordnen Sie zu: „could not read Username“, „Authentication failed“, „403“. Was bedeutet jede Meldung, und was prüfen Sie zuerst?

**Aufgabe 5.** Praktisch: Erzeugen Sie ein Repository mit einer 2 MB großen Zufallsdatei, die im nächsten Commit gelöscht wird. Ermitteln Sie mit `rev-list` und `cat-file` die größten Objekte der Historie, und vergleichen Sie die Größe eines vollen, eines flachen und eines Klons mit `--filter=blob:none`.

**Aufgabe 6.** Praktisch: Stellen Sie das Repository aus Aufgabe 5 mit `git lfs migrate import` auf LFS um, bereinigen Sie Reflog und Objektspeicher, und belegen Sie mit `count-objects`, dass die Größe sinkt.

**Aufgabe 7.** Praktisch: Erzeugen Sie einen detached HEAD, machen Sie darin einen Commit, wechseln Sie zurück auf `main`, und holen Sie den Commit mithilfe des Reflogs auf einen neuen Branch zurück. Erzeugen Sie danach einen abgelehnten Push und lösen Sie ihn ohne `--force`.

**Aufgabe 8.** Praktisch (Transfer): Schreiben Sie ein Skript, das ein Repository per Mirror sichert, ein datiertes Bundle erstellt, das Bundle prüft und die Referenzen eines Restore-Klons mit dem Original vergleicht. Ergänzen Sie ein Mini-Runbook „Branch versehentlich gelöscht“ mit allen Pflichtabschnitten und prüfen Sie es mit einem Skript wie `runbook-check.sh`.

## Lösungen zu Modul 10

**Lösung 1.** Große Dateien in der Historie (nach LFS auslagern oder die Historie bereinigen); sehr viele Objekte beziehungsweise ein riesiger Arbeitsbaum (Partial Clone, Sparse Checkout, flacher Klon für Pipelines); ungepflegter Objektspeicher (`git gc`, `git maintenance`).

**Lösung 2.** Die Historie referenziert die Datei weiterhin; jeder Klon lädt sie mit. Nötig ist ein Umschreiben der Historie (zum Beispiel mit `git filter-repo` oder `git lfs migrate`), anschließende Bereinigung des Reflogs und Garbage Collection sowie ein neues Klonen aller Beteiligten. Gemeinsame Branches müssen abgestimmt umgeschrieben werden.

**Lösung 3.** Das Submodul ist ein Verweis auf einen Commit; Klone brauchen `--recurse-submodules` oder `git submodule update --init`, und der Verweis wird mit `update --remote` plus Commit aktualisiert. Der Subtree kopiert den Inhalt samt Historie ins Haupt-Repository: einfacher Klon, größerer Platzbedarf, Aktualisierung mit `git subtree pull`. Submodul für bewusst festgehaltene, selten geänderte Abhängigkeiten; Subtree für Code, der mit dem Projekt wächst.

**Lösung 4.** „could not read Username“: keine Zugangsdaten verfügbar, Helper und Pipeline-Konfiguration prüfen. „Authentication failed“: Zugangsdaten vorhanden, vom Server abgelehnt (abgelaufen, widerrufen, falscher Benutzer), Token erneuern. „403“: Zugangsdaten gültig, Berechtigung fehlt, Rechte und Umfang des Tokens prüfen.

**Lösung 5.** Block aus 10.1 übernehmen und die Größen vergleichen. Erwartung: Das größte Blob ist die gelöschte Datei, der volle Klon trägt sie mit (rund 3 MB bei 3 MB Datei), der flache Klon und der Klon mit `--filter=blob:none` sind deutlich kleiner. Der Filter-Klon verlangt, dass der (hier simulierte) Server `uploadpack.allowFilter` erlaubt.

**Lösung 6.** `git lfs install --local`, `git lfs migrate info --everything`, `git lfs migrate import --everything --include="*.bin"`, danach `git reflog expire --expire=now --all` und `git gc --prune=now`. `count-objects -vH` zeigt, dass `size-pack` auf wenige Kilobyte sinkt; `git lfs ls-files` listet die verwalteten Dateien.

**Lösung 7.** `git checkout --detach`, Commit erzeugen, `git switch main`, `git reflog` zeigt den Commit, `git branch rettung <Hash>` macht ihn erreichbar. Für den abgelehnten Push: von einem zweiten Klon aus pushen, im ersten `git fetch`, `git pull --rebase` oder `--no-rebase`, Konflikt auflösen, `git push`.

**Lösung 8.** Skeleton: `git clone --mirror`, `git remote update --prune`, `git bundle create "sicherung-$(date +%F).bundle" --all`, `git bundle verify`, `git clone --mirror <bundle> restore.git`, Vergleich der sortierten Ausgaben von `git for-each-ref --format='%(objectname) %(refname)'`. Das Runbook braucht die Abschnitte Symptom, Diagnose, Sofortmassnahme, Eskalation, Wiederanlauf und Pruefung sowie die Zeile `Geprueft:`; Diagnose: `git reflog` und `git branch -a`, Sofortmaßnahme: `git branch <name> <hash>` beziehungsweise Wiederherstellung aus Mirror oder Bundle.
