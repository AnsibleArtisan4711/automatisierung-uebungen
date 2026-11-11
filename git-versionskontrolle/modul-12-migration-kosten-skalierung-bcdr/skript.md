# Modul 12 - Migration, Kosten, Skalierung und BCDR

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (12.1 bis 12.9), danach folgen „Übungen zu Modul 12“ (8 Aufgaben) und „Lösungen zu Modul 12“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 3

> **Abweichung im Lehrplan:** Die Modulübersicht nennt 10 h, die neun Lerneinheiten in „Lehrplan in Details“ ergeben dagegen je 1 h und damit 9 h. Die Tabelle unten führt die Einheiten wie im Lehrplan auf (Summe 9,0 h). Als Umfang wurde vorsorglich der höhere Wert (10 Seiten) zugrunde gelegt. Alle Einheiten haben Prio 3.

**Lernziele des Moduls:** Git-Landschaften migrieren, optimieren und ausfallsicher betreiben; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Dieses Modul geht vom einzelnen Repository zur ganzen Landschaft: Wie kommt man von Subversion oder TFVC zu Git, wie ordnet man viele Repositories, was kostet das Ganze, wie hält man es schnell, und wie stellt man es wieder her, wenn etwas ausfällt? Zuletzt geht es darum, wie man aus all dem eine dauerhafte Praxis macht: mit Zielbild, Schulung und kontinuierlicher Verbesserung.

| **Nr.** | **Lerneinheit**              | **Dauer (h)** | **Prio** |
|:--------|:-----------------------------|--------------:|:--------:|
| 12.1    | Migration von SVN/TFS zu Git |           1,0 |    3     |
| 12.2    | Repository-Konsolidierung    |           1,0 |    3     |
| 12.3    | Kostenfaktoren               |           1,0 |    3     |
| 12.4    | Skalierung                   |           1,0 |    3     |
| 12.5    | BCDR für Git                 |           1,0 |    3     |
| 12.6    | Backup und Retention         |           1,0 |    3     |
| 12.7    | Governance Roadmap           |           1,0 |    3     |
| 12.8    | Schulung und Enablement      |           1,0 |    3     |
| 12.9    | Kontinuierliche Verbesserung |           1,0 |    3     |
|         | **Summe laut Einzelposten**  |       **9,0** |          |
|         | Angabe in der Modulübersicht |          10,0 |          |

> **Hinweis zu den Beispielen**
>
> Alle Skripte dieses Moduls wurden in einer Testumgebung ausgeführt (Git 2.43, git-svn mit Subversion 1.14.3, git-filter-repo). Die Beträge, Gewichte und Reifegrade in den Beispielen sind frei erfundene Übungswerte und keine Aussagen über echte Preise oder Produkte. Plattformabhängige Angaben stützen sich auf die am 28.10.2026 abgerufene Dokumentation und sind im Quellenverzeichnis aufgeführt.

## 12.1 Migration von SVN/TFS zu Git

Eine Migration ist mehr als ein technischer Kopiervorgang. Subversion (SVN) und Team Foundation Version Control (TFVC) sind *zentrale* Versionskontrollsysteme: Es gibt einen Server mit der einen Wahrheit, Arbeitskopien enthalten nur den aktuellen Stand, und Revisionen werden fortlaufend durchnummeriert. Git ist *verteilt*, jede Kopie enthält die vollständige Historie, und Commits werden über Hashes identifiziert (Modul 01). Daraus folgen Unterschiede, die man bei der Planung beachten muss. In SVN sind Branches und Tags gewöhnliche Verzeichnisse, die man kopiert, in Git sind es leichtgewichtige Referenzen. SVN versioniert Verzeichnisse, auch leere, Git nur Dateiinhalte. SVN kennt Properties an Dateien (`svn:ignore`, `svn:executable`), Git nur wenige davon als Dateimodus. Und Autoren sind in SVN nur Benutzernamen, in Git Name plus E-Mail-Adresse. Wer diese Unterschiede nicht kennt, bekommt eine Migration, die technisch „funktioniert“ und fachlich Teile der Historie verliert.

Das Standardwerkzeug für SVN ist `git svn`. Laut der Dokumentation (abgerufen am 28.10.2026) importiert `git svn clone <url> --stdlayout --prefix svn/` ein Repository mit dem Standardlayout `trunk`, `branches`, `tags`; die SVN-Zweige landen dabei unter `refs/remotes/svn/`. Bei abweichendem Layout gibt man die Pfade explizit mit `-T`, `-b` und `-t` an. Die Autoren ordnet man mit `--authors-file` zu, einer Datei der Form `jdoe = Jane Doe <jane.doe@example.com>`; fehlt ein SVN-Benutzer in der Datei, bricht der Import ab, und man ergänzt ihn und startet neu. Die Dokumentation weist auf Grenzen hin, die man in die Planung einbeziehen muss: Es wird nur `svn:executable` ausgewertet, andere Properties landen in einem Logfile (`unhandled.log`); Verzeichnisumbenennungen und Kopien erkennt Git nicht von sich aus; leere Verzeichnisse werden nicht versioniert (mit `--preserve-empty-dirs` legt der Import Platzhalterdateien an); Tags werden vorsichtshalber als Branches mit dem Präfix `tags/` importiert, weil SVN Änderungen an Tags erlaubt; und die `git-svn-id`-Zeilen in den Commit-Nachrichten entfernt man besser nicht, weil sich alte SVN-Referenzen sonst kaum noch zuordnen lassen. Wer den Trunk allein ohne Layout-Optionen klont, verliert Branches und Tags.

Für TFVC beschreibt die Microsoft-Dokumentation (abgerufen am 28.10.2026) zwei Wege. Der Import-Assistent von Azure Repos übernimmt den Inhalt eines Pfades oder Branches. Seine Grenzen sind laut Dokumentation: nur ein Branch oder Ordner, höchstens 1 GB einschließlich Historie, höchstens 180 Tage Historie ab dem neuesten Changeset. Für Migrationen mit vollständiger Historie oder mit mehreren Branches und Merge-Beziehungen nennt die Seite das Werkzeug Git-TFS, das Microsoft nicht unterstützt. Die Empfehlungen der Seite lassen sich auf jede Migration übertragen: nur einen Branch migrieren und einen Git-Workflow wählen, zunächst nur den aktuellen Stand importieren (Tip-Migration), Binärdateien aus dem Repository entfernen oder über Git LFS führen (der Assistent konfiguriert LFS nicht), TFVC-Konfigurationsdateien umwandeln (`.tfignore` zu `.gitignore`), die Quelle nach der Migration schreibgeschützt online lassen und das Team gezielt schulen.

Das Beispiel führt eine vollständige Migration an einem kleinen SVN-Repository durch, das das Skript zuerst selbst erzeugt: mit drei Autoren, einem Tag und einem Branch. Es zeigt die vier Schritte, die in jedem Projekt vorkommen: die Analyse (Revisionen, Autoren, Layout), den Import mit Autorenzuordnung, die Umwandlung der SVN-Spuren in echte Git-Branches und -Tags und zuletzt die Verifikation durch Vergleich der Dateibäume. Ohne Verifikation weiß man nicht, ob die Migration vollständig ist.

```bash
# --- SVN-Testrepository aufbauen (Standardlayout, drei Autoren, ein Tag, ein Branch)
svnadmin create svnrepo >/dev/null; R=file://$PWD/svnrepo
svn mkdir -q $R/trunk $R/branches $R/tags -m "Layout" --username anna
svn checkout -q $R/trunk wc --username anna && cd wc
echo "v1" > app.txt; svn add -q app.txt; svn commit -q -m "Erste Version" --username anna
echo "v2" >> app.txt; svn commit -q -m "Zweite Version" --username bernd
cd ..
svn copy -q $R/trunk $R/tags/release-1.0 -m "Tag release-1.0" --username anna
svn copy -q $R/trunk $R/branches/feature-x -m "Branch feature-x" --username clara
cd wc && echo "v3" >> app.txt && svn commit -q -m "Dritte Version" --username anna && cd ..

echo "=== 1. Analyse vor der Migration"
echo "Revisionen: $(svnlook youngest svnrepo)"
echo "Layout:     $(svn ls $R | tr '\n' ' ')"
svn log -q $R | grep '^r' | cut -d'|' -f2 | tr -d ' ' | sort -u > svn-autoren.txt
echo "Autoren:    $(tr '\n' ' ' < svn-autoren.txt)"
awk '{print $1" = "$1" <"$1"@example.org>"}' svn-autoren.txt > authors.txt
cat authors.txt

echo "=== 2. Import"
git svn clone -q $R --stdlayout --prefix=svn/ --authors-file=authors.txt git-neu >/dev/null 2>&1
cd git-neu
git log -1 --format=%B svn/trunk | tail -n 2 | cut -c1-60
git branch -a | sed 's/^/  /'

echo "=== 3. Umwandlung: SVN-Spuren werden Git-Tags und -Branches"
git branch -m master main
for ref in $(git for-each-ref --format='%(refname)' refs/remotes/svn/tags); do
  name=${ref#refs/remotes/svn/tags/}
  git tag -a "$name" -m "Aus SVN-Tag $name" "$ref" && git update-ref -d "$ref"
done
for ref in $(git for-each-ref --format='%(refname)' refs/remotes/svn | grep -v '/trunk$'); do
  git branch "${ref#refs/remotes/svn/}" "$ref" && git update-ref -d "$ref"
done
git tag
git branch | sed 's/^/  /'

echo "=== 4. Verifikation: Dateibaeume vergleichen"
cd ..
svn export -q $R/trunk exp-trunk; mkdir g-trunk; git -C git-neu archive main | tar -x -C g-trunk
diff -r exp-trunk g-trunk && echo "trunk == main: identisch"
svn export -q $R/tags/release-1.0 exp-tag; mkdir g-tag; git -C git-neu archive release-1.0 | tar -x -C g-tag
diff -r exp-tag g-tag && echo "tags/release-1.0 == Tag release-1.0: identisch"
echo "Commits in Git: $(git -C git-neu rev-list --count main)   Revisionen in SVN: $(svnlook youngest svnrepo)"
echo "Autoren in Git:"; git -C git-neu log --format='%an <%ae>' --all | sort -u | sed 's/^/  /'
```

Die Analyse nennt sechs Revisionen, obwohl nur drei Dateiänderungen gemacht wurden, weil auch das Anlegen des Layouts sowie die Kopien für Tag und Branch Revisionen sind; im Git-Repository tauchen sie als Commits mit Meldungen wie „Tag release-1.0“ auf, die keine Dateien ändern. Die Autorendatei entsteht aus der Liste der SVN-Benutzer; im Ernstfall pflegt man sie mit den echten Namen und Adressen aus dem Verzeichnisdienst. Der Import legt die Referenzen unter `svn/` ab und hängt jedem Commit eine `git-svn-id`-Zeile an, die die ursprüngliche Revision nennt. Die Umwandlung benennt den Standardbranch um, macht aus den importierten Tag-Zweigen echte, annotierte Git-Tags und aus den übrigen Referenzen lokale Branches; die Referenzen unter `svn/` werden gelöscht, damit nur das Endergebnis im Repository bleibt. Der annotierte Tag zeigt auf den Commit, der in SVN die Kopie erzeugt hat; sein Dateibaum ist der Stand des Tags. Die Verifikation exportiert denselben Stand aus SVN und aus Git und vergleicht beide Dateibäume mit `diff -r`; Übereinstimmung ist der Beleg, dass Inhalte vollständig und unverändert angekommen sind. Die Zahl der Commits muss zur Zahl der Revisionen passen (hier hat der Trunk weniger Commits als SVN Revisionen, weil Revisionen anderer Pfade nicht zum Trunk gehören).

Zur Strategie gehört, nicht alles auf einmal zu migrieren. Ein *Pilotprojekt* wählt man so, dass es repräsentativ, aber nicht kritisch ist: eine mittlere Historie, ein engagiertes Team, ein überschaubarer Funktionsumfang. Am Pilot lernt man, wie lange der Import dauert, welche Autoren und Properties Probleme machen, wie die Pipelines umzustellen sind und welche Schulung nötig ist. Danach plant man Wellen: zuerst Repositories mit wenigen Abhängigkeiten, zuletzt die zentralen Bibliotheken. Während der Umstellung gilt ein klarer Stichtag, nach dem in der alten Quelle nicht mehr geschrieben wird, und die Quelle bleibt schreibgeschützt erreichbar, wie auch die TFVC-Dokumentation empfiehlt.

### Typische Fehler und Stolperfallen.

Falsch ist, einen Import ohne Autorendatei durchzuführen und die Autoren als bloße Kennungen ohne Adresse zu akzeptieren; später lässt sich das nur durch Umschreiben der Historie korrigieren. Zweiter Fehler: Binärdateien und Build-Ergebnisse mitzunehmen, die in SVN unbemerkt im Repository lagen, und damit ein riesiges Git-Repository zu erzeugen (Modul 10, Abschnitt 10.1 und 10.2); besser vor der Migration aufräumen und große Dateien nach LFS führen. Dritter Fehler: Ohne Verifikation zu starten. Vierter Fehler: Den Arbeitsablauf zu vergessen; wer in Git dieselbe Arbeitsweise wie in SVN führt (alles auf einem Branch, keine Reviews), verschenkt den Nutzen. Fünfter Fehler: Die Konvertierung zu wiederholen, nachdem Teams schon auf dem neuen Repository arbeiten; der Import sollte nur einmal als „letzter Lauf“ stattfinden.

### Praxisbezug.

In der Praxis dauert der technische Import oft Stunden und die organisatorische Umstellung Monate. Bewährt hat sich eine Migrationscheckliste: Analyse, Aufräumen, Autorenzuordnung, Testlauf, Verifikation, Pilot, Schulung, Pipeline-Umstellung, Stichtag, Archivierung der Quelle. Wer mehrere hundert Repositories hat, automatisiert die Schritte und führt eine Liste mit Status je Repository.

## 12.2 Repository-Konsolidierung

Mit den Jahren entstehen zu viele oder zu kleine oder falsch geschnittene Repositories: Dienste, die immer zusammen geändert werden, liegen in getrennten Repositories; Bibliotheken ohne Besitzer sammeln sich an; abgeschlossene Projekte werden nie archiviert. Die Konsolidierung ordnet das neu. Drei Maßnahmen kommen vor: *Archivieren* (nicht mehr gebrauchte Repositories stilllegen), *Zusammenlegen* (mehrere Repositories in eines, etwa zu einem Monorepo, Modul 05) und *Aufteilen* (ein zu großes Repository in mehrere). Die Entscheidung hängt von zwei Fragen ab, die technisch klingen, aber organisatorisch sind: Wer ist der *Besitzer*, und welche *Abhängigkeiten* gibt es? Ein Repository ohne Besitzer sollte archiviert oder einem Team zugewiesen werden, und Repositories, die ständig gemeinsam geändert werden, gehören zusammen.

Die technische Hürde beim Zusammenlegen ist die Historie. Eine Kopie der Dateien würde sie verlieren, was Nachvollziehbarkeit und `git blame` zerstört (Modul 06). Das gängige Verfahren schreibt die Historie jedes Quell-Repositories so um, dass alle Dateien in einem Unterverzeichnis liegen (mit `git filter-repo --to-subdirectory-filter`, Modul 08), und führt die Historien dann mit `git merge --allow-unrelated-histories` zusammen, denn die Repositories haben keinen gemeinsamen Vorfahren. Das Aufteilen ist die Umkehrung: `git filter-repo --subdirectory-filter` behält nur die Historie, die ein Unterverzeichnis betrifft, und macht es zur Wurzel. In beiden Fällen arbeitet man an *frischen Klonen*, weil filter-repo die Historie umschreibt (und deshalb bewusst keine Spuren des Originals behält).

Das folgende Beispiel legt zwei Dienste mit je zwei Commits und jeweils einer Datei `README.md` an, führt sie in ein Monorepo zusammen, zeigt, dass die Historie je Dienst erhalten bleibt, und teilt einen Dienst wieder ab. Zum Schluss entscheidet ein kleines Skript, welche Repositories einer Sammlung Kandidaten fürs Archivieren sind: Es sieht sich letzte Aktivität und Zahl der Autoren an und rechnet gegen einen festen Stichtag.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
c() { GIT_AUTHOR_DATE="$1T10:00:00" GIT_COMMITTER_DATE="$1T10:00:00" git commit -q -m "$2" "${@:3}"; }
mkrepo() {  # mkrepo <name> <datum1> <datum2>
  git init -q -b main "$1" && ( cd "$1" && git config user.name Dev && git config user.email dev@example.org \
    && echo "$1" > README.md && git add . && c "$2" "feat: Start $1" \
    && echo "mehr" > code.txt && git add . && c "$3" "feat: Code $1" ); }
mkrepo dienst-a 2026-01-10 2026-02-12
mkrepo dienst-b 2025-03-01 2025-04-20

echo "=== Zusammenlegen: Historie in Unterverzeichnissen"
git init -q -b main mono && cd mono && git config user.name Dev && git config user.email dev@example.org
cd ..
for r in dienst-a dienst-b; do
  git clone -q $r kopie-$r
  git -C kopie-$r filter-repo --to-subdirectory-filter services/$r --force >/dev/null 2>&1
  git -C mono fetch -q ../kopie-$r main:refs/heads/import-$r 2>/dev/null
  git -C mono merge -q --allow-unrelated-histories -m "chore: $r in Monorepo uebernommen" import-$r
  git -C mono branch -q -D import-$r
done
cd mono
ls services/*
echo "Commits gesamt: $(git rev-list --count HEAD)   (2 + 2 plus ein Merge-Commit)"
echo "Historie von dienst-b bleibt erhalten:"
git log --format='  %ad %s' --date=short -- services/dienst-b/
echo "blame zeigt den urspruenglichen Autor und Commit:"
git blame services/dienst-a/code.txt | cut -c1-60

echo "=== Aufteilen: dienst-b wieder herausloesen"
cd ..
git clone -q mono abgetrennt && cd abgetrennt
git filter-repo --subdirectory-filter services/dienst-b --force >/dev/null 2>&1
ls; git log --format='  %ad %s' --date=short
cd ..

echo "=== Archivkandidaten (Stichtag 2026-10-28, Schwelle 365 Tage)"
stichtag=$(date -u -d 2026-10-28 +%s)
for r in dienst-a dienst-b; do
  letzter=$(git -C $r log -1 --format=%cs)
  tage=$(( (stichtag - $(date -u -d "$letzter" +%s)) / 86400 ))
  autoren=$(git -C $r log --format=%ae | sort -u | wc -l)
  [ $tage -gt 365 ] && urteil="ARCHIVIEREN pruefen" || urteil="aktiv"
  printf "%-10s letzte Aenderung %s (%3d Tage)  Autoren: %d  -> %s\n" $r $letzter $tage $autoren "$urteil"
done
```

Das Skript klont die Dienste, schreibt ihre Historien auf Unterverzeichnisse um und holt sie mit `fetch` als Hilfsbranch ins Monorepo, den es dann mit `--allow-unrelated-histories` einmerged und wieder löscht. Im Ergebnis liegen beide Dienste unter `services/`, die Historie eines Dienstes bleibt mit den ursprünglichen Daten erhalten, und `git blame` nennt für jede Zeile den Ursprungscommit. Der erste Import ist auf dem noch leeren Branch ein einfaches Vorspulen, erst der zweite erzeugt einen Merge-Commit, der die Zusammenführung in der Historie sichtbar macht. Beim Aufteilen entsteht ein Repository, dessen Wurzel das frühere Unterverzeichnis ist und das nur dessen Commits enthält. Die Archivprüfung bewertet `dienst-b`, dessen letzte Änderung über ein Jahr zurückliegt, als Kandidaten; die Entscheidung bleibt ein Gespräch mit dem Besitzer und keine Automatik.

Archivieren heißt zwei Dinge: Das Repository wird schreibgeschützt (die Plattformen bieten dafür einen Archiv-Status), und es wird gesichert (Bundle oder Mirror, Abschnitt 12.6), bevor man etwas löscht. Dazu gehört eine Notiz in der README, wo die Nachfolge liegt, und die Prüfung, ob andere Repositories oder Pipelines noch darauf verweisen, etwa als Submodul oder Paketquelle (Modul 10, Abschnitt 10.3). Eine Suche über alle Repositories nach der alten URL gehört zu den Pflichtaufgaben vor dem Abschalten.

### Typische Fehler und Stolperfallen.

Falsch ist, Repositories durch Kopieren der Dateien zusammenzulegen und die alten dann zu löschen; die Historie ist damit endgültig weg. Zweiter Fehler: Mit filter-repo im bestehenden Repository zu arbeiten und dann zu merken, dass die Historie nicht mehr passt; immer am frischen Klon arbeiten. Dritter Fehler: Zusammenlegen ohne Klärung von Besitz und Rechten; im Monorepo sehen plötzlich alle den Code der anderen, was Zugriffsregeln (CODEOWNERS, Modul 11) nötig macht. Vierter Fehler: Dateinamenskollisionen zu übersehen; im Beispiel haben beide Dienste eine `README.md`, was nur wegen der Unterverzeichnisse gut geht. Fünfter Fehler: Alte Tags und Branches mitzunehmen, ohne ihre Namen zu prüfen; zwei Quellen mit dem Tag `v1.0.0` kollidieren, sodass man sie mit Präfix versieht.

### Praxisbezug.

Eine Konsolidierung beginnt mit einer Bestandsaufnahme: Welche Repositories gibt es, wer besitzt sie, wann wurde zuletzt gearbeitet, wie groß sind sie, wovon hängen sie ab? Die Liste ist zugleich der Ausgangspunkt für Kostenbetrachtung (12.3) und Governance (12.7). Auf Plattformen mit Organisations-APIs lässt sich die Liste automatisch erzeugen; die Skripte in diesem Abschnitt zeigen die Auswertung, sobald die Daten vorliegen.

## 12.3 Kostenfaktoren

Was ein Git-Betrieb kostet, setzt sich aus mehr Posten zusammen, als die Rechnung der Plattform zeigt. Es gibt direkte Kosten: Lizenzen oder Abonnements je Benutzer, Pipeline-Minuten auf gehosteten Runnern, Speicher für Artefakte, Caches, Pakete und LFS-Daten, Datenübertragung. Es gibt Betriebskosten bei selbst betriebenen Systemen: Server, Backup, Updates, Bereitschaft, Sicherheitsprüfungen. Und es gibt indirekte Kosten, die selten in einem Budget stehen: Wartezeit auf langsame Pipelines, Zeit, die Menschen mit der Suche nach Fehlern verlieren, aufgeblähte Repositories, die jeden Klon verlangsamen.

Wie genau die Abrechnung funktioniert, zeigt die Dokumentation von GitHub zu den Actions (abgerufen am 28.10.2026), exemplarisch für eine Plattform. Gemessen wird die Laufzeit von Jobs auf GitHub-gehosteten Runnern, wobei die Kosten pro Minute vom Betriebssystem und der Rechenleistung abhängen. Die Minuten werden dem Besitzer des Repositories berechnet, nicht der Person, die den Lauf gestartet hat, und auch fehlgeschlagene Läufe zählen: Ein Lauf, der nach fünf von zehn Minuten abbricht, und ein erneuter erfolgreicher Lauf ergeben zusammen 15 Minuten. Self-hosted Runner sind laut Seite kostenlos (die Plattform berechnet sie nicht; Betrieb und Hardware trägt man selbst), Standard-Runner in öffentlichen Repositories ebenfalls, größere Runner werden immer berechnet. Artefakte und Pakete teilen sich ein gemeinsames Speicherkontingent, der Cache-Speicher gilt pro Repository und beträgt auf der abgerufenen Seite bei allen Plänen 10 GB, abgerechnet wird stündlich in GB-Stunden, und gelöschte Artefakte verhindern künftige, aber keine bereits angefallenen Kosten. Die inkludierten Kontingente reichen laut Seite von 2 000 Minuten und 500 MB Artefaktspeicher im kostenlosen Tarif bis 50 000 Minuten und 50 GB bei Enterprise Cloud; diese Werte ändern sich und sind vor jeder Kalkulation neu nachzulesen. Ohne hinterlegte Zahlungsmethode wird die Nutzung nach Ausschöpfen des Kontingents blockiert. Für andere Plattformen gelten andere Modelle; das Prinzip, dass Zeit, Speicher und Benutzer die drei Treiber sind, bleibt.

Entscheidend ist, die Kosten *sichtbar zu machen*, und zwar in der Einheit, in der Entscheidungen fallen: pro Team und pro Repository. Wenn niemand weiß, was seine Pipeline kostet, hat niemand einen Grund, sie zu verbessern. Das Skript im Beispiel führt zwei Datenquellen zusammen: die tatsächliche Größe der Repositories, die man mit Git selbst messen kann, und eine Nutzungsliste, wie sie Plattformen als Bericht exportieren können (hier als CSV mit erfundenen Zahlen). Die Preise sind ebenfalls Übungswerte. Das Ergebnis ist ein Kostenbericht je Repository und je Team, plus der Hinweis auf das teuerste Repository.

```bash
mkdir daten && cd daten
mkrepo() {  # mkrepo <name> <kilobyte>
  git init -q -b main "$1" && ( cd "$1" && git config user.name Dev && git config user.email dev@example.org \
    && head -c $(( $2 * 1024 )) /dev/urandom > daten.bin && git add . && git commit -q -m "feat: Start" ); }
mkrepo web 200; mkrepo api 800; mkrepo ml-modelle 4000
cd ..
cat > nutzung.csv <<'EOF'
repo;team;runner_minuten;artefakt_gb;lfs_gb
web;frontend;4200;1.5;0
api;backend;9800;3.2;0
ml-modelle;data;1500;0.5;12.0
EOF
cat > kosten.sh <<'EOF'
#!/bin/sh
# Kostenbericht je Repository und Team (Preise sind erfundene Beispielwerte, keine echten Tarife)
# Aufruf: kosten.sh <Repo-Verzeichnis> <nutzung.csv>
P_MIN=0.008      # je Runner-Minute
P_ART=0.50       # je GB Artefakt und Monat
P_LFS=5.00       # je GB LFS und Monat
dir=$1
echo "repo;team;git_mb;kosten"
tail -n +2 "$2" | while IFS=';' read repo team min art lfs; do
  kb=$(git -C "$dir/$repo" count-objects -v | awk '/^size(-pack)?:/ {s+=$2} END {print s}')
  mb=$(awk -v k="$kb" 'BEGIN {printf "%.1f", k/1024}')
  k=$(awk -v m="$min" -v a="$art" -v l="$lfs" -v pm=$P_MIN -v pa=$P_ART -v pl=$P_LFS 'BEGIN {printf "%.2f", m*pm + a*pa + l*pl}')
  echo "$repo;$team;$mb;$k"
done
EOF
sh kosten.sh daten nutzung.csv > bericht.csv
awk -F';' '{printf "%-12s %-10s %8s %10s\n", $1, $2, $3, $4}' bericht.csv
echo "--- je Team"
tail -n +2 bericht.csv | awk -F';' '{t[$2]+=$4; g+=$4} END {for (x in t) printf "%-10s %8.2f  (%4.1f %%)\n", x, t[x], 100*t[x]/g}' | sort -k2 -nr
echo "--- teuerstes Repository"
tail -n +2 bericht.csv | sort -t';' -k4 -nr | head -1 | awk -F';' '{printf "%s (%s EUR/Monat, Git-Groesse %s MB)\n", $1, $4, $3}'
```

Das Skript misst die Größe jedes Repositories mit `git count-objects -v`, indem es lose Objekte und Pack-Größe addiert, und rechnet Nutzungswerte mit den Beispielpreisen in Kosten um. Das teuerste Repository ist das mit den Modellen, weil Speicher für LFS im Beispiel ins Gewicht fällt, obwohl es kaum Pipeline-Minuten verbraucht; das Backend-Team hat dagegen den höchsten Anteil an den Minuten. Die Auswertung nach Team zeigt die Verteilung in Prozent. Solche Zahlen sind weniger für die Abrechnung nützlich als für das Gespräch: Ein Team, das erfährt, dass ein einziger Test die Hälfte seiner Minuten verbraucht, kann etwas tun. Zwischen Rohdaten und Entscheidung steht die Zuordnung von Repositories zu Teams; sie ist selbst ein Pflegethema und gehört in die Eigentümerliste aus 12.2.

Sparen lässt sich an wenigen Stellen besonders gut: Pipelines mit Filtern und Abbruch veralteter Läufe (Modul 09, Abschnitt 9.1), Caches statt wiederholter Downloads, kleine Container-Images, nächtliche statt ständiger schwerer Tests, das Entfernen alter Artefakte durch Aufbewahrungsfristen, die Vermeidung großer Dateien in der Historie und die Prüfung, ob selbst betriebene Runner günstiger sind (der Betrieb kostet Arbeitszeit und ist nicht kostenlos). Eine Faustregel zur Bewertung ist, die Kosten ins Verhältnis zum Nutzen zu setzen: Ein teurer Sicherheitsscan, der Schwachstellen findet, ist gut investiertes Geld, ein teurer Lauf, der bei jedem Tippfehler alles neu baut, nicht.

### Typische Fehler und Stolperfallen.

Falsch ist, nur die Lizenzkosten zu betrachten und Pipelines und Speicher als Nebensache. Zweiter Fehler: Kosten ohne Zuordnung zu Teams zu berichten; dann ist niemand zuständig. Dritter Fehler: Ein Quotenlimit für Pipelines oder Speicher einzustellen und das Team nicht zu informieren, sodass die Pipeline eines Morgens ausfällt (Modul 10, Abschnitt 10.2 beschreibt dasselbe für LFS). Vierter Fehler: Mit erfundenen oder veralteten Preisen zu kalkulieren; Tarife ändern sich, und Aussagen aus Dokumentationen sind nur eine Momentaufnahme.

### Praxisbezug.

Ein monatlicher Kostenbericht je Team, ergänzt um die drei größten Treiber und die drei größten Einsparmöglichkeiten, reicht für den Anfang. Er verwandelt eine unscharfe Sorge („Git ist teuer“) in konkrete Posten mit Verantwortlichen.

## 12.4 Skalierung

Ein Repository wächst entlang dreier Achsen: Zahl und Größe der Dateien im aktuellen Stand, Länge der Historie und Zahl der Beteiligten. Jede Achse belastet andere Operationen. Viele Dateien verlangsamen `git status` und `checkout`, eine lange Historie verlangsamt Klonen und Log-Abfragen, viele Beteiligte erzeugen viele Referenzen und viel Konfliktpotenzial. Für große Repositories, vor allem Monorepos (Modul 05), gibt es zwei Techniken, die dazu passen und die in Modul 05 und 10 eingeführt wurden: *Sparse Checkout* und *Partial Clone*.

Beide wirken auf unterschiedlichen Ebenen, und der Nutzen entsteht in der Kombination. Ein Sparse Checkout steuert, welche Dateien im *Arbeitsverzeichnis* liegen; die Dokumentation spricht vom Cone-Modus, bei dem man ganze Verzeichnisse angibt. Ein Partial Clone steuert, welche *Objekte* aus dem Repository heruntergeladen werden. Die Dokumentation (abgerufen am 28.10.2026) beschreibt den Zweck so: Ein normaler Klon lädt die gesamte Historie mit allen Commits, Trees und Blobs, was bei sehr großen Repositories Stunden dauern und über 100 GiB belegen kann; ein Partial Clone lädt Objekte, die man voraussichtlich nicht braucht, vorab nicht und holt fehlende bei Bedarf nach. Mit `--filter=blob:none` bleiben Commits und Trees, aber keine Dateiinhalte; `blob:limit=<n>` lässt nur große Blobs weg; `tree:0` lässt Trees und Blobs weg. Die Dokumentation nennt `blob:none` als ideal in Verbindung mit Sparse Checkout, weil dann nur die benötigten Dateien nachgeladen werden. Das Remote, das fehlende Objekte nachliefert, heißt *Promisor Remote*. Der Server muss Filter erlauben (`uploadpack.allowFilter`); große Plattformen unterstützen das nach der Dokumentation standardmäßig.

Die Dokumentation nennt auch die Preise dieser Technik, die man vor dem Einsatz kennen sollte. Fehlende Objekte erfordern eine Verbindung zum Promisor Remote, Offline-Arbeit ist eingeschränkt, außer man lädt vorher nach. Der Nachladevorgang fordert Objekte oft einzeln an; Operationen, die viele fehlende Objekte brauchen (etwa `git log -p` oder ein Checkout), erzeugen viel Overhead und möglicherweise wiederholte Anmeldungen. Ältere Git-Versionen können wegen der Erweiterung mitten in einer Operation scheitern, und das Pushen einzelner Objekte an ein Promisor Remote ist nicht möglich. Entsprechend gehört zur Einführung eine Mindestversion für Git und eine Anleitung, die erklärt, wann man vollständig klonen sollte.

Das Beispiel zeigt die Wirkung. Es baut ein Repository mit drei Bereichen zu je 60 Dateien und einer zusätzlichen großen Datei im dritten Bereich, klont es vollständig und dann als Partial Clone mit Sparse Checkout auf einen Bereich, und zählt, wie viele Dateien im Arbeitsverzeichnis liegen und wie viele Objekte noch fehlen. Zum Schluss wird ein weiterer Bereich hinzugenommen, damit man sieht, dass Git die fehlenden Objekte nachlädt.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main gross && cd gross
git config user.name Dev; git config user.email dev@example.org
for b in bereich-a bereich-b bereich-c; do
  mkdir $b; for i in $(seq 1 60); do echo "$b $i" > $b/datei$i.txt; done
done
head -c 2000000 /dev/urandom > bereich-c/daten.bin
echo "Gemeinsame Doku" > README.md
git add . && git commit -q -m "feat: Struktur" && cd ..
git clone -q --bare gross server.git
git -C server.git config uploadpack.allowFilter true

echo "=== Vollstaendiger Klon"
git clone -q file://$PWD/server.git voll
echo "Dateien im Arbeitsverzeichnis: $(git -C voll ls-files | wc -l)   .git: $(du -sk voll/.git | cut -f1) KB"

echo "=== Partial Clone + Sparse Checkout (nur bereich-a)"
git clone -q --filter=blob:none --sparse file://$PWD/server.git teil
git -C teil sparse-checkout set bereich-a
echo "Dateien im Arbeitsverzeichnis: $(find teil -path teil/.git -prune -o -type f -print | wc -l)   .git: $(du -sk teil/.git | cut -f1) KB"
echo "Fehlende Objekte (noch nicht geladen): $(git -C teil rev-list --objects --all --missing=print | grep -c '^?')"
git -C teil config remote.origin.partialclonefilter
git -C teil sparse-checkout list

echo "=== bereich-c hinzunehmen: Git laedt nach"
git -C teil sparse-checkout add bereich-c
echo "Dateien im Arbeitsverzeichnis: $(find teil -path teil/.git -prune -o -type f -print | wc -l)   .git: $(du -sk teil/.git | cut -f1) KB"
echo "Fehlende Objekte: $(git -C teil rev-list --objects --all --missing=print | grep -c '^?')"
```

Im vollständigen Klon liegen alle 182 Dateien im Arbeitsverzeichnis, und das Verzeichnis `.git` enthält auch die große Datei. Beim Partial Clone mit Sparse Checkout enthält das Arbeitsverzeichnis nur die Dateien von `bereich-a` und die Dateien im obersten Verzeichnis (im Cone-Modus gehören die Dateien der Wurzel immer dazu), das Verzeichnis `.git` ist deutlich kleiner, und die Zahl fehlender Objekte zeigt, was Git zurückgehalten hat. Die Konfiguration nennt den Filter `blob:none`. Nach dem Hinzunehmen von `bereich-c` lädt Git die dort benötigten Dateien nach: Das Arbeitsverzeichnis wächst, `.git` auch (durch die große Datei), und die Zahl fehlender Objekte sinkt auf die Dateien von `bereich-b`, die weiter nicht gebraucht werden.

Neben diesen Techniken verbessern drei Dinge die Skalierung: Dateien, die nicht in Git gehören, bleiben draußen (Modul 10, Abschnitt 10.1 und 10.2), der Objektspeicher wird gepflegt (`git maintenance`), und die Strukturierung des Repositories folgt der Struktur der Teams, sodass kaum jemand das ganze Repository braucht. Wenn ein Repository trotzdem an die Grenzen kommt, ist Aufteilen (12.2) eine legitime Antwort, und die Frage „Monorepo oder viele Repositories?“ wird zur Abwägung von Sichtbarkeit und Atomarität gegen Größe und Zugriffsrechte, nicht zum Glaubenssatz.

### Typische Fehler und Stolperfallen.

Falsch ist, einen Partial Clone als allgemeinen Ersatz für den vollständigen Klon zu verteilen; Menschen, die offline arbeiten oder viel in der Historie suchen, leiden. Zweiter Fehler: Sparse Checkout als Sicherheitsmaßnahme zu verstehen; es verbirgt nur Dateien im Arbeitsverzeichnis, die Daten sind bei vollständigem Klon trotzdem vorhanden (für Zugriffsbeschränkung braucht man getrennte Repositories oder Plattformrechte). Dritter Fehler: Skalierungsprobleme mit Hardware zu lösen, die eigentlich aus Repository-Inhalten stammen. Vierter Fehler: Ohne Messung zu optimieren (Modul 10, Abschnitt 10.1).

### Praxisbezug.

Bei Pipelines sind flache Klone oder Partial Clones Standard, bei Menschen der vollständige Klon mit Ausnahmen für sehr große Repositories. Zwischen beiden steht die Frage, welche Historie die Pipeline wirklich braucht (Modul 09, Abschnitt 9.8). Wo die Plattform Sparse-Checkout-Schritte in der Checkout-Aktion anbietet, setzt man sie dort ein, statt eigene Skripte zu bauen.

## 12.5 BCDR für Git

BCDR steht für *Business Continuity and Disaster Recovery*: Wie arbeitet das Unternehmen weiter, wenn etwas ausfällt (Kontinuität), und wie stellt es verlorene Systeme wieder her (Wiederherstellung)? Für Git fragt man zwei Dinge und legt Zielwerte fest. Der *Recovery Point Objective* (RPO) sagt, wie viel Arbeit im schlimmsten Fall verloren gehen darf, ausgedrückt als Zeitraum: Bei einem RPO von 24 Stunden genügt ein tägliches Backup. Der *Recovery Time Objective* (RTO) sagt, wie lange die Wiederherstellung höchstens dauern darf. Die Ziele ergeben sich aus dem Geschäft, nicht aus der Technik: Ein Unternehmen, das stündlich ausliefert, hat andere Werte als eines mit vierteljährlichen Releases.

Ein Git-Remote ist verteilt, und das erweckt den Eindruck, ein Ausfall sei harmlos, weil jeder Klon eine Kopie hat. Das stimmt nur eingeschränkt, wie Modul 10, Abschnitt 10.7 gezeigt hat: Klone sind unvollständig, ungleichmäßig aktuell, und sie enthalten nichts von dem, was die Plattform sonst hält. Zur *Wiederherstellbarkeit* gehört deshalb mehr als der Code. Das Repository: alle Branches, Tags und deren Historie. Große Dateien: LFS-Objekte, die man gesondert sichern muss. Plattformkonfiguration: Schutzregeln, Teams, Berechtigungen, Webhooks, Pipeline-Variablen. Zusammenarbeitsdaten: Issues, Pull Requests, Reviews, Wikis. Und Geheimnisse: Secrets und Tokens, die Pipelines brauchen. Die Dokumentation von GitHub (abgerufen am 28.10.2026) empfiehlt für ein Backup die Kombination aus Spiegelklon (`git clone --mirror`), `git lfs fetch --all` und Klon des Wikis, die Ablage an mindestens einem externen Ort und die Prüfung der Wiederherstellung. Migrationsarchive der API sind laut Seite zur Archivierung gedacht, und es gibt keinen unterstützten Weg, sie auf GitHub wiederherzustellen; sie enthalten nicht alle Daten (genannt werden LFS-Objekte, Discussions und Packages). Für Issues und Pull Requests muss man im Einzelfall klären, was die jeweilige Plattform exportiert.

Was Sie nicht ohne Übung können, können Sie nicht. Der Kern von BCDR ist deshalb der *Restore-Test*, den Modul 10, Abschnitt 10.7 an einem einzelnen Repository gezeigt hat. In diesem Abschnitt geht es um die Übung im Ganzen, als *Disaster-Recovery-Drill*: Man nimmt an, die Plattform sei nicht mehr erreichbar, stellt aus der Sicherung auf einer neuen Plattform her, prüft, dass alles wieder funktioniert, und *misst* dabei die Zeit und den Datenverlust. Das Skript im Beispiel macht genau das im Kleinen. Es sichert ein Repository, lässt danach noch einen Commit entstehen (die Arbeit seit der Sicherung), zerstört das Original, stellt in ein neues Repository wieder her, vergleicht Referenzen, startet als Rauchtest ein Testskript im Klon der Wiederherstellung, und berechnet den tatsächlichen Datenverlust (RPO) und die Dauer der Wiederherstellung (RTO). Die Ergebnisse landen in einem Protokoll.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
c() { GIT_AUTHOR_DATE="$1" GIT_COMMITTER_DATE="$1" git commit -q -m "$2" "${@:3}"; }
git init -q -b main plattform.git --bare
git clone -q plattform.git arbeit 2>/dev/null && cd arbeit
git config user.name Dev; git config user.email dev@example.org
printf 'echo "Tests ok"\n' > test.sh; echo 1 > code.txt
git add . && c "2026-10-27T08:00:00" "feat: Start"
git tag -a v1.0.0 -m "1.0.0"
git switch -q -c release/1.0 && echo 2 >> code.txt && git add . && c "2026-10-27T15:00:00" "fix: Release-Korrektur"
git switch -q main
git push -q origin --all && git push -q origin --tags && cd ..

echo "=== Sicherung (angenommen: naechtlich um 02:00, also 2026-10-28T02:00)"
git clone -q --mirror plattform.git sicherung.git
git -C sicherung.git bundle create ../sicherung.bundle --all 2>/dev/null
git -C sicherung.git bundle verify ../sicherung.bundle | head -1

echo "=== Danach: weitere Arbeit am Vormittag"
cd arbeit && echo 3 >> code.txt && git add . && c "2026-10-28T09:30:00" "feat: Neue Funktion" && git push -q origin main && cd ..
STOERUNG="2026-10-28T11:00:00"
echo "=== Stoerung um $STOERUNG: Plattform weg"
rm -rf plattform.git arbeit

echo "=== Restore-Drill"
START=$(date +%s)
git init -q --bare -b main neue-plattform.git
git -C sicherung.git push -q --mirror ../neue-plattform.git 2>/dev/null
git clone -q neue-plattform.git probe 2>/dev/null
( cd probe && sh test.sh )
ENDE=$(date +%s)

echo "--- Vergleich der Referenzen (Sicherung gegen Wiederherstellung)"
git -C sicherung.git for-each-ref --format='%(objectname) %(refname)' | sort > r1.txt
git -C neue-plattform.git for-each-ref --format='%(objectname) %(refname)' | sort > r2.txt
diff -q r1.txt r2.txt >/dev/null && echo "Referenzen identisch ($(wc -l < r2.txt))" || echo "ABWEICHUNG"
git -C neue-plattform.git fsck --full >/dev/null 2>&1 && echo "fsck ok"
echo "--- RPO-Auswertung"
letzter=$(git -C neue-plattform.git log -1 --format=%cI main)
echo "Letzter wiederhergestellter Commit auf main: $letzter"
echo "Verlust: alles nach der Sicherung, hier der Commit von 2026-10-28T09:30 (Stoerung um 11:00)"
s1=$(date -u -d "2026-10-28T02:00:00" +%s); s2=$(date -u -d "$STOERUNG" +%s)
echo "Tatsaechliches Zeitfenster ohne Sicherung: $(( (s2 - s1) / 3600 )) h (Ziel-RPO 24 h: erfuellt, aber nicht das Ziel von 4 h)"
echo "--- RTO-Messung"
echo "Wiederherstellung dauerte $(( ENDE - START )) Sekunden"
echo "$(date -u +%F);Drill 1;RTO_s=$(( ENDE - START ));Referenzen=$(wc -l < r2.txt);Befund=ok" >> drill-log.csv
cat drill-log.csv | sed 's/^[0-9-]*/<Datum>/'
```

Der Ablauf hat vier Teile. Die Sicherung besteht aus Mirror und Bundle, wie in Modul 10 gezeigt. Nach der Sicherung entsteht noch ein Commit am Vormittag, den die Sicherung nicht enthält; er stellt die Arbeit dar, die zwischen Sicherung und Störung entsteht. Die Störung vernichtet das Original. Beim Drill richtet das Skript eine neue Plattform ein (ein leeres Bare-Repository), spielt den Mirror mit `git push --mirror` ein, klont daraus und führt den Rauchtest aus, der prüft, dass der Code lauffähig ist. Die Auswertung vergleicht die Referenzen von Sicherung und Wiederherstellung, was bestätigt, dass alles übertragen wurde, und stellt den tatsächlichen Verlust fest: Der Commit vom Vormittag fehlt, das Zeitfenster zwischen Sicherung und Störung beträgt neun Stunden. Ein RPO von 24 Stunden wäre damit eingehalten, ein Ziel von vier Stunden nicht; das Beispiel zeigt, wie die Übung Zielwerte überprüfbar macht. Die gemessene Dauer (im Beispiel unter einer Sekunde) ist der Beitrag zum RTO, wobei bei echten Repositories das Übertragen der Daten und das Umstellen aller Beteiligten (neue Remote-URL, Tokens, Webhooks) den größten Teil der Zeit ausmachen.

Das Protokoll in `drill-log.csv` ist kein Beiwerk, sondern der Nachweis. Prüfer fragen nicht, ob es ein Backup gibt, sondern wann zuletzt wiederhergestellt wurde und mit welchem Ergebnis. Ergänzend zur Technik gehört ein Inventar dessen, was ausserhalb von Git liegt: Konfiguration, Teams und Secrets. Für Secrets gilt, dass man sie *nicht* im Backup der Repositories ablegt, sondern in einem eigenen Tresor mit eigener Sicherung, und dass das Inventar nur Namen, Besitzer und Rotationstermine nennt, niemals Werte. Das folgende kleine Skript prüft ein solches Inventar darauf, dass jedes Secret einen Besitzer und einen künftigen Rotationstermin hat.

```bash
cat > secrets-inventar.csv <<'EOF'
name;besitzer;system;naechste_rotation
CI_DEPLOY_TOKEN;team-plattform;pipeline;2027-01-15
REGISTRY_PASSWORT;;pipeline;2027-02-01
BACKUP_SCHLUESSEL;team-betrieb;backup;2026-09-01
EOF
stichtag=2026-10-28
tail -n +2 secrets-inventar.csv | while IFS=';' read name besitzer system rot; do
  problem=""
  [ -z "$besitzer" ] && problem="$problem kein Besitzer;"
  [ "$rot" \< "$stichtag" ] && problem="$problem Rotation ueberfaellig seit $rot;"
  [ -n "$problem" ] && echo "PROBLEM $name:$problem" || echo "ok      $name (naechste Rotation $rot)"
done
```

Das Skript meldet beim Registry-Passwort den fehlenden Besitzer und beim Backup-Schlüssel die überfällige Rotation; so entsteht aus einer Liste eine Prüfung.

> **Ergänzung aus der Nachrecherche**
>
> **Bei Self-Managed-Plattformen** gehört zur Sicherung zusätzlich die Datenbank und Konfiguration der Plattform selbst, und die Wiederherstellung muss auf gleicher oder kompatibler Version erfolgen. Die Anleitungen dazu liefert der Hersteller; sie wurden hier nicht abgerufen und sind Teil der Planung.

### Typische Fehler und Stolperfallen.

Falsch ist, das Backup nach dem Einrichten nie zu testen und im Ernstfall festzustellen, dass die Dateien unbrauchbar sind. Richtig ist ein regelmäßiger Drill mit Protokoll. Zweiter Fehler: Ziele (RPO, RTO) nicht festzulegen, sodass niemand weiß, ob die Sicherung genügt. Dritter Fehler: Sicherung und Original im selben Konto oder Rechenzentrum; wer das Original löschen kann, löscht dann auch die Sicherung. Vierter Fehler: Secrets im selben Backup wie der Code abzulegen. Fünfter Fehler: Den Wiederanlauf der Beteiligten zu vergessen; nach der Wiederherstellung brauchen alle die neue URL, neue Tokens und Webhooks, und das gehört ins Runbook (Modul 10, Abschnitt 10.8).

### Praxisbezug.

Ein vernünftiger Rhythmus ist ein kleiner Restore-Test je Monat (ein Repository, automatisiert) und ein großer Drill je Jahr (alles Wichtige, mit Menschen und Kommunikation). Die Ergebnisse gehen an die Verantwortlichen für Betrieb und Sicherheit. Wo Aufsichtsregeln gelten, ist das Protokoll Teil des Nachweises.

## 12.6 Backup und Retention

Neben der Frage, wie man sichert (12.5), steht die Frage, wie lange man aufbewahrt. *Retention* heißt Aufbewahrungsfrist: Wie viele Sicherungen behält man, und wie alt dürfen sie werden? Zu wenige Generationen sind gefährlich, weil ein Fehler erst Tage später auffällt und die Sicherung dann schon den Fehler enthält. Zu viele kosten Speicher und können Pflichten verletzen: Personenbezogene Daten dürfen nicht beliebig lange aufbewahrt werden, andere Daten müssen dagegen nach gesetzlichen Fristen aufbewahrt werden. Die Abwägung zwischen beidem ist Compliance und Kostenfrage zugleich. Rechtliche Aussagen hierzu sind Sache von Datenschutz und Recht und keine Rechtsberatung dieses Skripts; für den Umgang mit Löschpflichten in Git-Historien siehe Modul 08, Abschnitt 8.8.

Ein verbreitetes Muster ist das *Großvater-Vater-Sohn*-Verfahren (auch Staffelung): Man behält die letzten Tagessicherungen, dazu eine Sicherung je Woche über einige Wochen und eine je Monat über einige Monate. Dadurch hat man nah an der Gegenwart feine Auflösung und weiter zurück grobe, bei begrenztem Speicher. Welche Zahlen passen, hängt von RPO, Kosten und Pflichten ab; im Beispiel sind es sieben tägliche, vier wöchentliche und drei monatliche Sicherungen. Wichtig ist, die Regel in ein Skript zu fassen, dass sie anwendet, nicht von Hand zu löschen, und nach jedem Lauf zu prüfen, dass die Sicherungen noch gültig sind.

Das Beispiel erzeugt zur Demonstration 120 leere Sicherungsdateien mit Datum im Namen (eine pro Tag) und wendet die Regel an. Der Probelauf listet, was behalten und was gelöscht würde, erst die Option `--run` löscht tatsächlich. Danach wird ein echtes Bundle angelegt und geprüft, weil eine Aufbewahrungsregel nichts nützt, wenn die Sicherungen selbst nicht lesbar sind. Als Wochenmarke wird die ISO-Woche verwendet, als Monatsmarke Jahr und Monat; für jede Marke behält das Skript die *älteste* Sicherung des Zeitraums.

```bash
mkdir sicherungen
for i in $(seq 0 119); do
  d=$(date -u -d "2026-10-28 - $i days" +%F)
  : > "sicherungen/repo-$d.bundle"
done
echo "Vorhanden: $(ls sicherungen | wc -l) Dateien"

cat > retention.sh <<'EOF'
#!/bin/sh
# Staffelung: 7 Tage, 4 Wochen, 3 Monate. Aufruf: retention.sh <Verzeichnis> <Stichtag> [--run]
dir=$1; stichtag=$2; run=$3
behalte=""
for datei in $(ls "$dir" | sort); do
  d=${datei#repo-}; d=${d%.bundle}
  alter=$(( ($(date -u -d "$stichtag" +%s) - $(date -u -d "$d" +%s)) / 86400 ))
  woche=$(date -u -d "$d" +%G-W%V); monat=$(echo "$d" | cut -c1-7)
  grund=""
  [ $alter -lt 7 ] && grund="Tag"
  [ -z "$grund" ] && [ $alter -lt 28 ] && ! echo "$seen_w" | grep -q " $woche " && { grund="Woche"; seen_w="$seen_w $woche "; }
  [ -z "$grund" ] && [ $alter -ge 28 ] && [ $alter -lt 100 ] && ! echo "$seen_m" | grep -q " $monat " && { grund="Monat"; seen_m="$seen_m $monat "; }
  if [ -n "$grund" ]; then echo "KEEP   $datei ($grund)"; else echo "DELETE $datei"; [ "$run" = "--run" ] && rm "$dir/$datei"; fi
done
EOF
sh retention.sh sicherungen 2026-10-28 > plan.txt
echo "Plan: $(grep -c KEEP plan.txt) behalten, $(grep -c DELETE plan.txt) loeschen"
echo "--- behaltene Sicherungen:"; grep KEEP plan.txt | sed 's/^KEEP *repo-//; s/.bundle//' | tr '\n' ' '; echo
sh retention.sh sicherungen 2026-10-28 --run > /dev/null
echo "Nach --run: $(ls sicherungen | wc -l) Dateien"

echo "--- Pruefung einer echten Sicherung"
git init -q -b main quelle && cd quelle && git config user.name Dev && git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: eins" && git bundle create ../echt.bundle --all 2>/dev/null
git bundle verify ../echt.bundle 2>&1 | head -1 | sed 's#../##'
```

Das Skript prüft für jede Sicherung, wie alt sie ist, und entscheidet nach Reihenfolge: Alles unter sieben Tagen bleibt als Tagessicherung. Von den älteren bis 28 Tagen bleibt die älteste jeder ISO-Woche, von den noch älteren bis 100 Tagen die älteste jedes Monats; alles andere wird gelöscht. Nach dem Probelauf bleiben 7 tägliche, 4 wöchentliche und 3 monatliche Sicherungen, insgesamt also nur 14 von 120 Dateien, und der Befehl mit `--run` führt die Löschung durch. Dass die Auswahl die älteste Sicherung einer Woche oder eines Monats behält, ist eine Konvention; manche Teams bevorzugen die jüngste. Das abschließende `bundle verify` zeigt die Prüfung der echten Datei und ist der Teil, der in einer realen Umgebung nach jedem Backup laufen müsste.

Die Kosten ergeben sich direkt aus der Regel: Speicher je Generation mal Zahl der Generationen mal Preis. Weil Git-Sicherungen nach dem ersten Mal ähnliche Daten enthalten, sparen inkrementelle Verfahren Platz (Bundles können inkrementell sein, wie die Dokumentation zu `git bundle` beschreibt, abgerufen am 28.10.2026: ein Bundle mit Voraussetzungen enthält nur die seit einem früheren Stand neuen Objekte, und der Empfänger muss diesen Stand besitzen). Das macht die Wiederherstellung allerdings komplizierter, weil man die Kette braucht; in der Praxis ist ein regelmäßiges Vollbackup und dazwischen Inkrementelles ein Kompromiss, den man im Drill (12.5) überprüft. Dokumentiert wird die Regel samt Begründung, damit eine spätere Änderung bewusst geschieht.

### Typische Fehler und Stolperfallen.

Falsch ist, Sicherungen unbegrenzt aufzubewahren, weil Speicher „billig“ sei; er summiert sich, und Datenschutzpflichten kennen Höchstfristen. Zweiter Fehler: Nur eine Generation zu halten; wenn der Fehler vor dem letzten Backup entstand, ist er in allen Sicherungen. Dritter Fehler: Die Löschung von Hand oder per Cron-Einzeiler ohne Probelauf; ein Fehler im Muster löscht alles. Vierter Fehler: Aufbewahrungsfristen für personenbezogene Daten nicht mit dem Datenschutz abzustimmen, vor allem bei Löschanfragen, die sich auch auf Sicherungen auswirken können (keine Rechtsberatung). Fünfter Fehler: Sicherungen nicht zu verifizieren und erst im Ernstfall zu merken, dass sie defekt sind.

### Praxisbezug.

Eine Retention-Richtlinie besteht aus einem Satz je Datenart: Was wird gesichert, wie oft, wie lange aufbewahrt, wo gespeichert, wer ist verantwortlich, wie wird geprüft. Das ist der Text, den man bei einem Audit vorlegt und der zusammen mit dem Drill-Protokoll (12.5) die Nachweise bildet.

## 12.7 Governance Roadmap

Governance bedeutet hier die Gesamtheit der Regeln, Verantwortlichkeiten und Messgrößen, mit denen ein Unternehmen seine Versionskontrolle steuert. Die Bausteine kennen Sie aus den früheren Modulen: Workflows und Branch-Regeln (Modul 05), Sicherheit und Secrets (Modul 08), Automatisierung (Modul 09), Betrieb (Modul 10), Plattform und Richtlinien (Modul 11). Eine *Roadmap* verbindet sie zu einem Weg: Wo stehen wir, wo wollen wir hin, in welchen Schritten und in welcher Reihenfolge? Ohne Roadmap werden Regeln einzeln und ungeordnet eingeführt, und das Ergebnis ist ein Flickenteppich.

Als Hilfsmittel dient ein *Reifegradmodell*. Es beschreibt Stufen, von „Ad hoc“ (jeder macht es anders) über „Definiert“ (Regeln existieren) und „Gesteuert“ (Regeln werden gemessen und durchgesetzt) bis „Optimierend“ (Messwerte treiben Verbesserungen). Das Modell ist keine Norm, sondern ein Werkzeug; wichtig ist, dass es zu den eigenen Dimensionen passt. Im Beispiel sind es fünf: Workflow, Sicherheit, Automatisierung, Betrieb und Wissen. Pro Dimension beantwortet man einige Fragen mit 0 (nein), 1 (teilweise) oder 2 (ja) und erhält durch den Durchschnitt eine Stufe. Das Skript berechnet das, findet die schwächste Dimension und schlägt daraus den nächsten Schritt vor. Alle Fragen und Werte im Beispiel sind erfunden.

```bash
cat > antworten.csv <<'EOF'
dimension;frage;wert
Workflow;Standardbranch ist geschuetzt;2
Workflow;Reviews sind Pflicht;2
Workflow;Branch-Strategie ist dokumentiert;1
Sicherheit;Secret-Scan beim Push aktiv;1
Sicherheit;Signierte Commits oder Tags fuer Releases;0
Sicherheit;Tokens haben Ablaufdatum und Besitzer;1
Automatisierung;Pflichtpruefungen in der Pipeline;2
Automatisierung;Versionierung und Changelog automatisiert;1
Automatisierung;Hooks im Team verteilt;0
Betrieb;Regelmaessiger Restore-Test;0
Betrieb;Runbooks vorhanden und aktuell;1
Betrieb;Repository-Groesse wird ueberwacht;0
Wissen;Onboarding-Skript vorhanden;1
Wissen;Schulung zu Git-Grundlagen;2
Wissen;Supportkanal fuer Git-Fragen;2
EOF
cat > reifegrad.sh <<'EOF'
#!/bin/sh
# Reifegrad je Dimension: Durchschnitt 0..2 -> Stufe 1 (Ad hoc) .. 4 (Optimierend)
awk -F';' '
  NR>1 { s[$1]+=$3; n[$1]++; if (!($1 in k)) { k[$1]=1; d[++m]=$1 } }
  END {
    schwach=""; min=99
    for (i=1;i<=m;i++) {
      x=d[i]; avg=s[x]/n[x]
      stufe = (avg<0.5)?1 : (avg<1.2)?2 : (avg<1.8)?3 : 4
      name = (stufe==1)?"Ad hoc" : (stufe==2)?"Definiert" : (stufe==3)?"Gesteuert" : "Optimierend"
      printf "%-16s %.2f von 2  Stufe %d (%s)\n", x, avg, stufe, name
      if (avg<min) { min=avg; schwach=x }
    }
    printf "Schwaechste Dimension: %s -> naechster Schritt dort zuerst\n", schwach
  }' "$1"
EOF
sh reifegrad.sh antworten.csv
echo "--- Fragen mit Wert 0 (Luecken fuer die Roadmap):"
awk -F';' 'NR>1 && $3==0 {printf "  [%s] %s\n", $1, $2}' antworten.csv
```

Die Ausgabe zeigt die Stufe je Dimension, hier ist Betrieb mit dem Durchschnitt 0,33 die schwächste. Die Liste der Nullwerte ist die rohe Fassung der Roadmap: Restore-Test einführen, Repository-Größe überwachen, Hooks verteilen, Releases signieren. Aus ihr wird eine Reihenfolge, wenn man Aufwand und Wirkung abschätzt und von der Wirkung auf Risiko und Compliance ausgeht: Ein Restore-Test kostet wenig und deckt das größte Risiko ab, also steht er vorn. Eine Roadmap hat Zeitrahmen (zum Beispiel drei Quartale), Verantwortliche und Messgrößen, die sagen, wann ein Schritt erreicht ist.

Messgrößen (KPIs) machen den Fortschritt sichtbar. Gute KPIs sind einfach zu erheben, beziehen sich auf den Zustand und nicht auf Personen (Modul 10, Abschnitt 10.6) und haben ein Ziel. Das folgende Skript erhebt zwei Git-basierte Kennzahlen aus einem Repository und hängt sie als Zeile an eine CSV-Datei an, sodass sich ein Verlauf ergibt: die Konventionsquote der Commit-Nachrichten und den Anteil der Merges unter den Commits des Hauptbranches. Es läuft hier für zwei erfundene Monate nacheinander, um den Trend zu zeigen.

```bash
git init -q -b main kpi-repo && cd kpi-repo
git config user.name Dev; git config user.email dev@example.org
c() { GIT_AUTHOR_DATE="$1T12:00:00" GIT_COMMITTER_DATE="$1T12:00:00" git commit -q --allow-empty -m "$2"; }
cat > ../kpi.sh <<'EOF'
#!/bin/sh
# Haengt eine KPI-Zeile an: Monat;Commits;Konventionsquote;Merge-Anteil (nur Commits bis zum Stichtag)
monat=$1; bis=$2
gesamt=$(git log --until="$bis" --no-merges --format=%s | wc -l)
konform=$(git log --until="$bis" --no-merges --format=%s | grep -Ec '^(feat|fix|docs|chore|refactor|test)(\([a-z0-9-]+\))?!?: ')
merges=$(git log --until="$bis" --merges --format=%h | wc -l)
alle=$(git log --until="$bis" --format=%h | wc -l)
awk -v m="$monat" -v g=$gesamt -v k=$konform -v mg=$merges -v a=$alle 'BEGIN {printf "%s;%d;%.0f %%;%.0f %%\n", m, g, 100*k/g, 100*mg/a}'
EOF
c 2026-09-05 "Quick fix"; c 2026-09-12 "feat: Suche"; c 2026-09-20 "aufgeraeumt"; c 2026-09-28 "fix: Fehler A"
echo "monat;commits;konvention;merge_anteil" > ../kpi.csv
sh ../kpi.sh 2026-09 2026-09-30T23:59:59 >> ../kpi.csv
c 2026-10-03 "feat: Export"; c 2026-10-10 "fix: Fehler B"; c 2026-10-15 "docs: Anleitung"; c 2026-10-22 "chore: Update"
sh ../kpi.sh 2026-10 2026-10-31T23:59:59 >> ../kpi.csv
awk -F';' '{printf "%-8s %-8s %-12s %s\n", $1, $2, $3, $4}' ../kpi.csv
```

Im September sind zwei von vier Commits konform, im Oktober sind es sechs von acht, die Quote steigt von 50 auf 75 Prozent. Ein Verlauf wie dieser zeigt, ob eine Maßnahme (etwa der Commit-Hook aus Modul 09) wirkt. Der Merge-Anteil ist im Beispiel null, weil das Repository keine Merges enthält; auf einem Repository mit Pull-Request-Workflow wäre er die Kennzahl für Direktcommits. Die Zahlen in diesem Beispiel sind erfunden und haben keine Aussagekraft über echte Teams.

### Typische Fehler und Stolperfallen.

Falsch ist, ein Reifegradmodell als Zensur zu verwenden und Teams zu vergleichen, bis sie die Antworten beschönigen. Richtig ist, es als gemeinsame Standortbestimmung zu nutzen. Zweiter Fehler: Eine Roadmap mit zu vielen gleichzeitigen Maßnahmen; besser wenige pro Quartal, die man abschließt. Dritter Fehler: Kennzahlen ohne Ziel und ohne Besitzer; sie werden angeschaut und folgenlos. Vierter Fehler: Kennzahlen, die Verhalten verzerren (zum Beispiel „Commits pro Person“, Modul 10). Fünfter Fehler: Regeln ohne Ausnahmeverfahren und ohne Überprüfung; sie veralten.

### Praxisbezug.

Eine Roadmap wird jährlich überprüft und quartalsweise nachgezogen, und ihr Ergebnis sind greifbare Dinge: ein Rollout von Rulesets, eine eingeführte Prüfung im Pipeline-Standard, ein erfolgreicher Drill. Die in Modul 11 gezeigten Prüfskripte und Plattformberichte liefern dafür die Zahlen.

## 12.8 Schulung und Enablement

Regeln, Werkzeuge und Prozesse wirken nur, wenn Menschen sie verstehen und anwenden können. *Enablement* bedeutet, Teams so auszustatten, dass sie Git sicher und selbstständig nutzen: durch Schulung, durch Dokumentation, durch Vorlagen (Modul 11, Abschnitt 11.7) und durch ein *Supportmodell*, das bei Problemen schnell hilft. Schulung ist dabei weniger eine einmalige Veranstaltung als ein Weg: Einstieg für Neue, Vertiefung für Erfahrene, spezielle Themen für Rollen (Releasemanagement, Sicherheit, Plattformbetrieb).

Drei Inhaltsbereiche tragen die Schulung, und sie entsprechen den Lernzielen dieses Kurses. Die *Grundlagen*: Repository, Commit, Branch, Merge, Remote, Fehlerbehebung (Modul 01 bis 04, 06, 10). Die *Zusammenarbeit*: Workflows, Pull Requests und Reviews, Regeln für Commit-Nachrichten, Umgang mit Konflikten (Modul 03, 05). Die *Sicherheit*: Secrets, Zugriff, Signaturen, Umgang mit Lieferketten (Modul 08). Wirksam ist, wenn Menschen die Dinge selbst tun: Konflikte auflösen, einen detached HEAD retten, einen Commit rückgängig machen. Vorträge allein bleiben nicht hängen. Dafür sind *Übungsrepositories* nützlich, die eine Situation vorbereiten und prüfen, ob sie gelöst wurde. Das Beispiel baut ein solches Labor: Das Skript erzeugt ein Repository mitten in einem Merge-Konflikt, ein Prüfskript beurteilt, ob die Aufgabe richtig gelöst wurde, und zum Schluss durchläuft das Beispiel erst den Fehlversuch und dann die Lösung.

```bash
cat > lab-konflikt.sh <<'EOF'
#!/bin/sh
# Erzeugt ein Uebungsrepository mit einem Merge-Konflikt. Aufgabe: Konflikt loesen und den Merge abschliessen.
set -e
git init -q -b main uebung && cd uebung
git config user.name Lernende; git config user.email lernende@example.org
printf 'Titel: Projektplan\nStart: Montag\n' > plan.txt
git add . && git commit -q -m "feat: Plan"
git switch -q -c vorschlag-a && sed -i 's/Start: Montag/Start: Dienstag/' plan.txt && git commit -q -am "fix: Start Dienstag"
git switch -q main && sed -i 's/Start: Montag/Start: Mittwoch/' plan.txt && git commit -q -am "fix: Start Mittwoch"
git merge vorschlag-a >/dev/null 2>&1 || true
echo "Uebung bereit: Konflikt in plan.txt. Aufgabe: Wunschtermin waehlen, Merge abschliessen."
EOF
cat > lab-pruefen.sh <<'EOF'
#!/bin/sh
# Prueft die Loesung der Uebung (im Verzeichnis 'uebung' ausfuehren)
fehler=0
[ -f .git/MERGE_HEAD ] && { echo "NICHT FERTIG: Merge ist noch nicht abgeschlossen (git commit fehlt)"; fehler=1; }
grep -q '^<<<<<<<\|^=======\|^>>>>>>>' plan.txt && { echo "NICHT FERTIG: Konfliktmarker in plan.txt"; fehler=1; }
[ "$(git rev-list --parents -n 1 HEAD | wc -w)" -eq 3 ] || { echo "NICHT FERTIG: letzter Commit ist kein Merge-Commit"; fehler=1; }
[ -z "$(git status --porcelain)" ] || { echo "NICHT FERTIG: Arbeitsverzeichnis nicht sauber"; fehler=1; }
[ $fehler -eq 0 ] && echo "GELOEST: $(grep '^Start' plan.txt)"
exit $fehler
EOF
sh lab-konflikt.sh
cd uebung
echo "--- Pruefung vor der Loesung"
sh ../lab-pruefen.sh; echo "Ergebnis: $?"
echo "--- Fehlversuch: Datei mit Markern committen"
git add plan.txt && git commit -q -m "wip"
sh ../lab-pruefen.sh; echo "Ergebnis: $?"
echo "--- Richtige Loesung: Marker entfernen, Merge-Commit korrigieren (amend)"
printf 'Titel: Projektplan\nStart: Dienstag\n' > plan.txt
git commit -q -a --amend -m "merge: Start Dienstag gewaehlt"
sh ../lab-pruefen.sh; echo "Ergebnis: $?"
```

Das Laborskript baut den Konflikt aus zwei Branches, die dieselbe Zeile verschieden ändern, und lässt den Merge im Konfliktzustand stehen, wie ihn Lernende in der Praxis vorfinden. Das Prüfskript testet vier Dinge: Der Merge muss abgeschlossen sein (die Datei `MERGE_HEAD` existiert nur während eines laufenden Merges), keine Konfliktmarker mehr in der Datei, der letzte Commit muss zwei Eltern haben, und das Arbeitsverzeichnis muss sauber sein. Der Fehlversuch im Beispiel ist typisch: Wer die Datei mit den Markern committet, schließt den Merge technisch ab und besteht trotzdem nicht, weil die Marker noch in der Datei stehen. Erst die Korrektur des Merge-Commits mit `git commit --amend` (ein Amend behält beide Eltern) besteht alle Prüfungen. So lässt sich eine Reihe von Übungen aufbauen (detached HEAD, abgelehnter Push, falscher Commit, Secret im Repository), die sich in Teams selbstständig bearbeiten lassen und Rückmeldung geben.

Das *Supportmodell* regelt, wohin man sich wendet. Bewährt hat sich eine Stufung: Erstens Selbsthilfe durch Dokumentation und Runbooks (Modul 10, Abschnitt 10.8), zweitens ein Fragekanal im Team (ein Chat-Kanal), in dem auch Erfahrene mitlesen, drittens Ansprechpartner je Team, sogenannte „Git-Botschafter“, viertens das Plattformteam für Störungen und Rechte. Die Antwortzeiten und Zuständigkeiten stehen im Modell, damit niemand raten muss. Gute Fragen und Antworten wandern in die Dokumentation, sodass der Kanal nicht nur Wissen verbraucht, sondern erzeugt.

### Typische Fehler und Stolperfallen.

Falsch ist, eine einmalige Schulung als erledigt zu verbuchen. Wissen verfällt, und neue Personen kommen laufend dazu. Zweiter Fehler: Rein theoretisch zu schulen; ohne eigene Hände am Repository bleibt wenig hängen. Dritter Fehler: Unterschiedliche Vorkenntnisse zu ignorieren; ein Selbsteinschätzungsbogen am Anfang hilft, Gruppen zu bilden. Vierter Fehler: Kein Supportmodell; dann holen sich Teams Hilfe an unterschiedlichen Stellen, und dieselben Fragen werden endlos beantwortet. Fünfter Fehler: Die Schulung von den Regeln zu trennen; die Regeln der Plattform müssen in der Schulung vorkommen, sonst lernen Menschen sie durch Ablehnung.

### Praxisbezug.

Eine Messgröße für den Erfolg ist die Entwicklung der Fehlerbilder aus Modul 10: Sinken Supportanfragen zu Konflikten und detached HEAD, wirkt die Schulung. Eine andere ist die Zeit bis zum ersten Pull Request neuer Kolleginnen und Kollegen (Modul 11, Abschnitt 11.7). Beides lässt sich mit einfachen Mitteln messen und in die Roadmap aufnehmen (12.7).

## 12.9 Kontinuierliche Verbesserung

Ein Betrieb, der nur auf Störungen reagiert, bleibt im Ausnahmezustand. Kontinuierliche Verbesserung heißt, aus dem, was passiert, systematisch zu lernen: Metriken zeigen, wo es hakt, Störungen werden ausgewertet, Rückmeldungen aus den Teams werden gesammelt, und technische Schulden werden planvoll abgebaut. Der Kreislauf ist einfach: messen, auswerten, entscheiden, umsetzen, wieder messen. Schwer ist nur, ihn durchzuhalten, und dafür helfen feste Termine und kleine Schritte.

Aus Störungen lernt man, wenn man sie festhält. Ein *Incident-Protokoll* enthält Zeitpunkt, Dauer, Kategorie, Ursache und Maßnahme. Aus ihm gewinnt man Kennzahlen wie die durchschnittliche Dauer bis zur Behebung (Mean Time To Resolve, MTTR) und die Häufigkeit je Kategorie. Die wichtigste Auswertung ist, welche Kategorien am häufigsten vorkommen und welche am meisten Zeit kosten, denn dort lohnt sich die Verbesserung: ein Runbook, eine Automatisierung, eine Regel oder eine Schulung. Nach größeren Störungen schreibt man ein *Postmortem*, das ohne Schuldzuweisung beschreibt, was geschah, warum und was sich ändert; seine Maßnahmen landen in der Roadmap (12.7) und in den Runbooks.

Das erste Skript im Beispiel wertet ein Incident-Protokoll aus (erfundene Daten): Es berechnet pro Kategorie Anzahl, Gesamtdauer und mittlere Dauer und nennt den größten Zeitfresser. Das zweite Skript erhebt *technische Schulden* im Repository, die man mit Git messen kann: Branches, die älter als 30 Tage sind, große Dateien im aktuellen Stand und offene Merkmarken im Code (`TODO` und `FIXME`). Beide laufen mit festem Stichtag, damit die Ergebnisse reproduzierbar sind.

```bash
cat > incidents.csv <<'EOF'
datum;kategorie;beginn;ende;massnahme
2026-08-04;Anmeldung;2026-08-04T09:00;2026-08-04T09:40;Token erneuert
2026-08-19;Push abgelehnt;2026-08-19T14:00;2026-08-19T14:20;fetch und rebase
2026-09-02;Anmeldung;2026-09-02T08:30;2026-09-02T10:30;SSO-Gruppe korrigiert
2026-09-15;Repository langsam;2026-09-15T10:00;2026-09-15T16:00;Historie bereinigt
2026-10-06;Anmeldung;2026-10-06T07:50;2026-10-06T08:20;Token erneuert
2026-10-20;Push abgelehnt;2026-10-20T11:00;2026-10-20T11:10;fetch und merge
EOF
echo "=== Incident-Auswertung"
tail -n +2 incidents.csv | while IFS=';' read d k b e m; do
  min=$(( ($(date -u -d "$e" +%s) - $(date -u -d "$b" +%s)) / 60 )); echo "$k;$min"
done | awk -F';' '{n[$1]++; s[$1]+=$2; g+=$2} END {
  for (k in n) printf "%-20s %d Faelle  %4d min gesamt  %5.1f min im Schnitt\n", k, n[k], s[k], s[k]/n[k]
  printf "Gesamt: %d min\n", g }' | sort -k4 -nr
echo "=== Technische Schulden im Repository (Stichtag 2026-10-28)"
git init -q -b main schulden && cd schulden
git config user.name Dev; git config user.email dev@example.org
c() { GIT_AUTHOR_DATE="$1T12:00:00" GIT_COMMITTER_DATE="$1T12:00:00" git commit -q -m "$2"; }
echo "// TODO: aufraeumen" > app.js; echo "// FIXME: Fehler bei Leerstring" >> app.js
head -c 1500000 /dev/urandom > export.dat
git add . && c 2026-06-01 "feat: Start"
git switch -q -c feature/alt && echo x > alt.txt && git add . && c 2026-07-15 "feat: alte Idee"
git switch -q main && git switch -q -c feature/neu && echo y > neu.txt && git add . && c 2026-10-22 "feat: neue Idee"
git switch -q main
heute=$(date -u -d 2026-10-28 +%s)
echo "Alte Branches (> 30 Tage ohne Aktivitaet):"
git for-each-ref --format='%(refname:short) %(committerdate:short)' refs/heads | while read b d; do
  tage=$(( (heute - $(date -u -d $d +%s)) / 86400 ))
  [ $tage -gt 30 ] && [ "$b" != main ] && echo "  $b ($tage Tage)"
done
echo "Grosse Dateien im aktuellen Stand (> 1 MB):"
git ls-tree -r -l HEAD | awk '$4 > 1000000 {printf "  %s (%.1f MB)\n", $5, $4/1048576}'
echo "Offene Merkmarken: $(git grep -E -c 'TODO|FIXME' | awk -F: '{s+=$2} END {print s}')"
```

Die Auswertung zeigt, dass „Repository langsam“ nur einmal vorkam, aber 360 Minuten kostete, während „Anmeldung“ dreimal vorkam und im Schnitt 63 Minuten brauchte; beide Kategorien verdienen Aufmerksamkeit, aus unterschiedlichen Gründen: Die Anmeldeprobleme sind häufig und lassen sich durch Token-Verwaltung und Gruppenpflege (Modul 10, 11) vermeiden, das langsame Repository ist selten, aber teuer und lässt sich durch Überwachung der Größe (Modul 10) und Aufräumen früh erkennen. Das Skript für technische Schulden findet einen Branch, der seit über drei Monaten ruht, eine große Datei, die vermutlich nicht ins Repository gehört (Modul 10, Abschnitt 10.2), und zwei Merkmarken im Code. Zahlen wie diese werden durch den Verlauf wertvoll: Sinken die alten Branches und die Zeit pro Incident, dann wirkt die Verbesserung.

Wirksam sind Verbesserungen, die klein genug sind, um sie wirklich umzusetzen. Ein guter Rhythmus ist ein kurzes Treffen einmal im Quartal, bei dem das Team die drei größten Probleme aus Kennzahlen und Rückmeldungen auswählt, je einen Verantwortlichen benennt und beim nächsten Treffen die Ergebnisse ansieht. Feedback aus den Teams ergänzt die Zahlen, denn manches, etwa dass eine Regel verwirrt oder eine Pipeline nervt, taucht in keiner Statistik auf. Technische Schulden in Repositories baut man regelmäßig und in kleinen Portionen ab: ein Aufräumtag für alte Branches, ein Review der großen Dateien, eine Runde Abhängigkeits-Updates (Modul 09, Abschnitt 9.7).

### Typische Fehler und Stolperfallen.

Falsch ist, Störungen zu beheben, aber nicht festzuhalten; dann wiederholen sie sich, ohne dass es auffällt. Zweiter Fehler: Postmortems mit Schuldzuweisung; Menschen verschweigen dann Fehler, und man lernt nichts. Dritter Fehler: Verbesserungsmaßnahmen ohne Verantwortliche und Termin; sie bleiben Absichtserklärung. Vierter Fehler: Technische Schulden nur dann anzugehen, wenn sie schmerzen; kleine, regelmäßige Aufräumarbeiten sind billiger. Fünfter Fehler: Kennzahlen ohne Kontext zu bewerten; ein Anstieg gemeldeter Störungen kann auch bessere Erfassung bedeuten.

### Praxisbezug.

Der Kreis schließt sich mit den Runbooks aus Modul 10: Jede ausgewertete Störung führt zu einer Verbesserung am Prozess, zu einer neuen oder aktualisierten Anleitung oder zu einer Automatisierung. Gemeinsam mit Governance-Roadmap (12.7) und Schulung (12.8) entsteht so ein Betrieb, der aus seinen Fehlern lernt und messbar besser wird.

## Übungen zu Modul 12

**Aufgabe 1.** Nennen Sie fünf Unterschiede zwischen SVN/TFVC und Git, die bei einer Migration beachtet werden müssen, und je eine Konsequenz.

**Aufgabe 2.** Warum wählt man für eine Migration ein Pilotprojekt, und woran erkennt man ein gutes Pilotprojekt?

**Aufgabe 3.** Wie unterscheiden sich Sparse Checkout und Partial Clone? Welche Nachteile hat ein Partial Clone laut Dokumentation?

**Aufgabe 4.** Erklären Sie RPO und RTO. Welche Ziele würden Sie für ein Unternehmen mit täglicher Auslieferung wählen, und warum reicht der Entwicklerklon nicht als Backup?

**Aufgabe 5.** Praktisch: Führen Sie eine Migration eines selbst angelegten SVN-Repositories mit Trunk, zwei Branches und zwei Tags nach Git durch. Wandeln Sie die Tags in annotierte Git-Tags um, und weisen Sie mit `diff -r` nach, dass Trunk und Tag identisch angekommen sind.

**Aufgabe 6.** Praktisch: Legen Sie drei kleine Repositories an, führen Sie sie mit erhaltener Historie in ein Monorepo zusammen und lösen Sie eines wieder heraus. Belegen Sie mit `git log` und `git blame`, dass die Historie erhalten ist.

**Aufgabe 7.** Praktisch: Passen Sie das Retention-Skript so an, dass 14 Tagessicherungen, 8 Wochensicherungen und 6 Monatssicherungen behalten werden. Prüfen Sie die Zahl der behaltenen Dateien im Probelauf und rechnen Sie nach.

**Aufgabe 8.** Praktisch (Transfer): Schreiben Sie ein Drill-Skript, das ein Repository mit LFS-Datei sichert (Mirror plus `git lfs fetch --all`), das Original zerstört, aus der Sicherung wiederherstellt, Referenzen und Dateiinhalt vergleicht und die Dauer sowie den Zeitabstand der letzten Sicherung protokolliert. Beschreiben Sie, welche Daten der Plattform damit nicht gesichert sind.

## Lösungen zu Modul 12

**Lösung 1.** Zentral gegenüber verteilt (volle Historie lokal, Offline-Arbeit); Branches und Tags als Verzeichnisse gegenüber Referenzen (Tags werden als Branches importiert und müssen umgewandelt werden); leere Verzeichnisse und Properties (nur `svn:executable` wird beachtet, leere Verzeichnisse brauchen Platzhalter); Autoren nur als Benutzernamen (Autorendatei nötig); Revisionsnummern gegenüber Hashes (`git-svn-id` erhalten oder Zuordnungstabelle). Weiter: Merge-Information, Umbenennungserkennung, Dateisperren.

**Lösung 2.** Der Pilot deckt Probleme (Autoren, Properties, Pipeline, Schulung) zu geringem Risiko auf und liefert Zeitmaße. Gut ist ein repräsentatives, nicht kritisches Projekt mit engagiertem Team, mittlerer Historie und überschaubaren Abhängigkeiten.

**Lösung 3.** Sparse Checkout steuert, welche Dateien im Arbeitsverzeichnis liegen, Partial Clone, welche Objekte geladen werden. Nachteile des Partial Clone: Verbindungspflicht zum Promisor Remote, viele Einzelanfragen beim Nachladen, Probleme mit alten Git-Versionen, kein Push einzelner Objekte, Overhead bei Operationen über viele fehlende Objekte.

**Lösung 4.** RPO: höchstens tolerierter Datenverlust als Zeitraum; RTO: höchstens tolerierte Wiederherstellungsdauer. Bei täglicher Auslieferung etwa RPO im Stundenbereich und RTO von wenigen Stunden. Klone sind unvollständig, unterschiedlich aktuell und enthalten keine Plattformdaten (Konfiguration, Issues, LFS-Objekte aller Versionen).

**Lösung 5.** Ablauf wie im Beispiel in 12.1: `svnadmin create`, `svn mkdir/copy/commit`, `git svn clone --stdlayout --prefix=svn/ --authors-file`, Tags per `git tag -a <name> refs/remotes/svn/tags/<name>`, Verifikation mit `svn export` und `git archive`. Erwartung: `diff -r` ohne Ausgabe.

**Lösung 6.** Pro Repository `git clone`, `git filter-repo --to-subdirectory-filter services/<name> --force`, im Monorepo `git fetch` und `git merge --allow-unrelated-histories`. Prüfung: `git log -- services/<name>/` zeigt die ursprünglichen Commits mit alten Daten, `git blame` die ursprünglichen Autoren. Herauslösen mit `git filter-repo --subdirectory-filter` in einem frischen Klon.

**Lösung 7.** Bedingungen im Skript ändern: Tage `< 14`, Wochen `< 14 + 56`, Monate `< 14 + 56 + 180`; behalten werden 14 Tagessicherungen, höchstens 8 Wochensicherungen (je älteste einer ISO-Woche) und höchstens 6 Monatssicherungen. Die genaue Zahl hängt davon ab, wie sich Wochen und Monate mit den Tagesgrenzen überschneiden; nachrechnen heißt, Wochen- und Monatsmarken im Plan zu zählen.

**Lösung 8.** Ablauf: Repository mit LFS-Datei anlegen, `git clone --mirror` und `git lfs fetch --all`, Original löschen, in neues Bare-Repository mit `git push --mirror` zurückspielen und LFS-Objekte mit `git lfs push --all` nachliefern (Wissensstand; Server mit LFS-Unterstützung nötig), Referenzen vergleichen, Dateiinhalt per `git lfs pull` prüfen, Dauer und Alter der Sicherung ins Protokoll. Nicht gesichert: Berechtigungen, Schutzregeln, Teams, Webhooks, Pipeline-Variablen, Secrets, Issues, Pull Requests, Wikis, Packages.
