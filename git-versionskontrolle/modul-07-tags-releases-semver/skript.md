# Modul 07 - Tags, Releases und Semantic Versioning

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (7.1 bis 7.8), danach folgen „Übungen zu Modul 07“ (8 Aufgaben) und „Lösungen zu Modul 07“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 2

> Die Modulübersicht und die Summe der Lerneinheiten stimmen überein (acht Einheiten zu je 1,25 h). Prioritäten der Einheiten: 7.1 bis 7.6 Prio 2, 7.7 und 7.8 Prio 3.

**Lernziele des Moduls:** Release-Artefakte, Tags, Versionierung und Changelogs verwalten; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Ein Release ist der Moment, in dem aus fortlaufender Entwicklung ein benennbarer, auslieferbarer Stand wird. Git liefert dafür die Bausteine: Tags benennen einen Commit dauerhaft, Signaturen belegen die Herkunft, und die Historie erlaubt es, aus Commits Release Notes und Versionsnummern abzuleiten. Dieses Modul verbindet diese Bausteine zu einem nachvollziehbaren Weg vom Code bis zum ausgelieferten Artefakt.

| **Nr.** | **Lerneinheit**               | **Dauer (h)** | **Prio** |
|:--------|:------------------------------|--------------:|:--------:|
| 7.1     | Tags Grundlagen               |          1,25 |    2     |
| 7.2     | Semantic Versioning           |          1,25 |    2     |
| 7.3     | Release Notes                 |          1,25 |    2     |
| 7.4     | Signed Tags                   |          1,25 |    2     |
| 7.5     | Release Branches und Hotfixes |          1,25 |    2     |
| 7.6     | Artefakt-Bezug                |          1,25 |    2     |
| 7.7     | Pre-Releases                  |          1,25 |    3     |
| 7.8     | Versionierung in CI/CD        |          1,25 |    3     |
|         | **Summe**                     |      **10,0** |          |

## 7.1 Tags Grundlagen

Ein Branch ist ein beweglicher Zeiger: Mit jedem neuen Commit rückt er weiter. Für Releases braucht man das Gegenteil, nämlich einen Namen, der dauerhaft auf einen bestimmten Commit zeigt, damit man auch in zwei Jahren noch sagen kann „Das war Version 1.4.2“. Genau dafür gibt es *Tags*. Technisch ist ein Tag eine Referenz unter `refs/tags/`, die auf ein Objekt zeigt, in der Regel auf einen Commit. Es gibt zwei Arten, und der Unterschied ist wichtig. Ein *Lightweight Tag* ist nur dieser Zeiger, ein Name ohne weitere Daten. Ein *annotierter Tag* ist ein eigenes Objekt in der Git-Datenbank (Modul 01) mit Erstellungsdatum, Name und E-Mail-Adresse der taggenden Person, einer Nachricht und optional einer kryptografischen Signatur. Die Git-Dokumentation (abgerufen am 28.10.2026) fasst es so zusammen: Annotierte Tags sind für Releases gedacht, Lightweight Tags für private oder vorübergehende Markierungen.

Die Wahl hat praktische Folgen. Befehle wie `git describe` ignorieren Lightweight Tags standardmäßig. Außerdem lassen sich nur annotierte Tags signieren, und nur sie tragen die Information, wer wann mit welcher Begründung etikettiert hat. Für alles, was nach außen sichtbar ist, nimmt man deshalb annotierte Tags: `git tag -a v1.0.0 -m "Release 1.0.0"`. Die Option `-m` allein genügt ebenfalls, denn sie impliziert `-a`. Ein Tag-Name sollte einem festen Schema folgen; verbreitet ist das Präfix `v` gefolgt von einer Versionsnummer (Abschnitt 7.2).

Zum Umgang mit Tags gehören ein paar Befehle, die man im Schlaf beherrschen sollte. `git tag -l 'v1.*'` listet Tags nach einem Muster, `-n` zeigt die Anfangszeilen der Nachrichten dazu. `git tag --points-at HEAD` zeigt, welche Tags auf den aktuellen Commit zeigen, `--contains <commit>` listet Tags, die einen Commit enthalten (nützlich für die Frage „In welchem Release steckt dieser Fix?“), und `--merged` beziehungsweise `--no-merged` filtern nach Erreichbarkeit. Wichtig ist, dass `git push` Tags nicht automatisch mitschickt: Dafür gibt es `git push origin <tag>`, `git push --tags` (alle, auch Lightweight Tags und temporäre Marken) oder `git push --follow-tags`, das nur annotierte Tags mitnimmt, die auf die gerade gepushten Commits zeigen.

Das Beispiel legt beide Tag-Arten an, vergleicht ihre Objekttypen, nutzt `git describe` und zeigt das Push-Verhalten gegenüber einem Bare-Repository.

```bash
git init -q -b main tg && cd tg
git config user.name Dev; git config user.email dev@example.org
echo 1 > app.txt && git add . && git commit -q -m "Erste Version"
git tag v0.1-test
git tag -a v1.0.0 -m "Release 1.0.0: erste stabile Version"
echo 2 >> app.txt && git commit -q -am "Verbesserung"
git tag -a v1.1.0 -m "Release 1.1.0"
git cat-file -t v0.1-test
git cat-file -t v1.0.0
git tag -n1
git tag --points-at HEAD
git tag --contains HEAD
echo 3 >> app.txt && git commit -q -am "Noch eine"
git describe
git tag wip-marke
git describe
git describe --tags
git describe --tags --long --dirty

git init -q --bare ../srv.git
git remote add origin ../srv.git
git push -q -u origin main
git push -q origin
git ls-remote --tags origin | wc -l
git push -q --follow-tags origin
git ls-remote --tags origin | grep -v '\^{}'
git push -q origin v0.1-test wip-marke
git ls-remote --tags origin | grep -vc '\^{}'
git tag -d v0.1-test
git push origin --delete v0.1-test
```

Die beiden `cat-file`-Aufrufe zeigen den Unterschied: Der Lightweight Tag `v0.1-test` ist ein `commit` (der Name zeigt direkt auf den Commit), der annotierte Tag `v1.0.0` ein `tag`-Objekt. `git tag -n1` gibt die Nachrichten aus; beim Lightweight Tag ist es die Nachricht des Commits, auf den er zeigt. `git tag --contains HEAD` listet nur `v1.1.0`, denn `v1.0.0` enthält den zweiten Commit noch nicht; genau so beantwortet man die Frage, in welchen Releases eine Änderung steckt. Nach einem weiteren Commit liefert `git describe` „v1.1.0-1-g...“: Der Name des nächsten erreichbaren annotierten Tags, die Anzahl der Commits seither (hier eins) und der abgekürzte Hash mit einem vorangestellten `g` für „git“ (Dokumentation zu `git describe`, abgerufen am 28.10.2026). Der Lightweight Tag `wip-marke` bleibt dabei unberücksichtigt; erst mit `--tags` erscheint er. Nach dem Push zeigt `git ls-remote`, dass der gewöhnliche Push keine Tags übertragen hat (null Zeilen), `--follow-tags` aber die annotierten Tags `v1.0.0` und `v1.1.0`. Die beiden Lightweight Tags werden nur durch ausdrückliche Nennung hochgeladen, und das Löschen eines Tags auf dem Server geschieht mit `git push origin --delete <tag>`.

> **Ergänzung aus der Nachrecherche**
>
> **Tags nachträglich ändern: besser nicht.** Die Dokumentation zu `git tag` (Abschnitt „On Re-tagging“, abgerufen am 28.10.2026) unterscheidet zwei Fälle. Wurde ein Tag noch nie veröffentlicht, kann man ihn einfach neu setzen (`-f` ersetzt den alten). Wurde er schon gepusht, lautet die Empfehlung, den Fehler einzugestehen und einen *anderen Namen* zu verwenden, etwa `X.1` statt `X` zu ersetzen. Begründung: Git ändert Tags nicht hinter dem Rücken der Nutzer; wer einen veröffentlichten Tag verschiebt, muss alle Betroffenen informieren, und diese müssen den alten Tag selbst löschen und neu holen. Ein verschobener Release-Tag zerstört das Vertrauen in die Eindeutigkeit von Versionen.

### Typische Fehler und Stolperfallen.

Falsch ist, einen Release mit einem Lightweight Tag zu markieren: Es fehlen Datum, Urheber und Begründung, `git describe` übergeht ihn, und er lässt sich nicht signieren. Richtig ist ein annotierter Tag mit sinnvoller Nachricht. Ein zweiter Fehler ist `git push --tags` als Gewohnheit: Es überträgt alles, was lokal an Tags existiert, einschließlich privater Markierungen und vergessener Test-Tags, und die landen dann dauerhaft im gemeinsamen Repository. Richtig ist, Release-Tags gezielt oder mit `--follow-tags` zu pushen. Drittens wird häufig auf dem falschen Commit getaggt, weil man vorher nicht geprüft hat, wo `HEAD` steht. Richtig ist, den Commit explizit zu nennen (`git tag -a v1.0.0 <commit>`) oder vorher `git log -1` zu lesen.

### Praxisbezug.

Jede Software, die ausgeliefert wird, braucht eine Antwort auf die Frage „Welcher Stand läuft bei Kunde X?“ Die Antwort ist ein Tag. Auf Plattformen sind Tags die Grundlage der Release-Seiten (weiterführend, hier nicht abgerufen): Die Release-Ansicht verknüpft einen Tag mit Beschreibung und Anhängen. Pipelines starten oft bei einem neuen Tag den Release-Build (Modul 09).

## 7.2 Semantic Versioning

Eine Versionsnummer ist ein Versprechen an die Nutzer: Sie sagt, wie viel Aufwand ein Update bedeutet. *Semantic Versioning*, kurz SemVer, macht dieses Versprechen verbindlich. Die Spezifikation (Version 2.0.0, abgerufen am 28.10.2026) schreibt eine Nummer der Form `MAJOR.MINOR.PATCH` vor und erklärt, wann welche Stelle erhöht wird. Die *Major*-Version steigt bei Änderungen der öffentlichen Schnittstelle, die nicht abwärtskompatibel sind. Die *Minor*-Version steigt bei neuer Funktionalität, die abwärtskompatibel ist; auch das Als-veraltet-Markieren einer Funktion ist ein Minor-Schritt. Die *Patch*-Version steigt bei abwärtskompatiblen Fehlerkorrekturen. Wer Minor erhöht, setzt Patch auf 0 zurück, wer Major erhöht, setzt Minor und Patch auf 0 zurück.

Der entscheidende Begriff ist die *öffentliche Schnittstelle* (Public API). Die Spezifikation sagt, dass Version 1.0.0 diese Schnittstelle definiert und alle weiteren Erhöhungen davon abhängen, wie sie sich ändert. Was zur Schnittstelle gehört, muss das Projekt selbst festlegen: bei einer Bibliothek sind es die öffentlichen Funktionen und Klassen, bei einem Webdienst die Endpunkte und Datenformate, bei einem Kommandozeilenwerkzeug die Optionen und Ausgabeformate. Ohne diese Festlegung ist SemVer nicht anwendbar. Die Versionen `0.y.z` sind für die frühe Entwicklung vorgesehen; dort darf sich laut Spezifikation alles jederzeit ändern. Das Verhältnis zu Git ist einfach: Die Versionsnummer steht im Tag (`v1.4.2`), und wer die Commit-Nachrichten einer Konvention folgen lässt, kann die nächste Nummer aus der Historie ableiten.

Eine verbreitete Konvention dafür sind die *Conventional Commits* (Modul 02; die Spezifikation war beim Abruf nicht erreichbar, die folgenden Angaben sind Wissensstand): Ein Commit beginnt mit einem Typ wie `feat:` oder `fix:`, ein Ausrufezeichen hinter dem Typ (`feat!:`) oder eine Fußzeile `BREAKING CHANGE:` kennzeichnet einen Bruch. Daraus folgt die Zuordnung: `fix` bewirkt Patch, `feat` Minor, ein Bruch Major. Das Beispiel setzt dies in ein kleines Skript um, das den letzten Tag sucht, die Commits seitdem liest und die nächste Version berechnet. Danach demonstriert es ein Sortierproblem, das bei Tags häufig überrascht.

```bash
git init -q -b main sv && cd sv
git config user.name Dev; git config user.email dev@example.org
echo 1 > app.txt && git add . && git commit -q -m "feat: erste Funktion"
git tag -a v1.4.2 -m "Release 1.4.2"
echo 2 >> app.txt && git commit -q -am "fix: Rundungsfehler behoben"
echo 3 >> app.txt && git commit -q -am "feat: Export als CSV"
echo 4 >> app.txt && git commit -q -am "docs: README ergaenzt"

cat > next-version.sh <<'EOF'
#!/bin/sh
# Ermittelt die naechste SemVer-Version aus den Commit-Nachrichten seit dem letzten Tag
letzter=$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null) || letzter=""
if [ -z "$letzter" ]; then basis="0.0.0"; bereich="HEAD"; else basis=${letzter#v}; bereich="$letzter..HEAD"; fi
major=${basis%%.*}; rest=${basis#*.}; minor=${rest%%.*}; patch=${rest#*.}
stufe=patch
msgs=$(git log --format='%s%n%b' "$bereich")
if echo "$msgs" | grep -Eq '^[a-z]+(\([^)]*\))?!:|^BREAKING CHANGE:'; then stufe=major
elif echo "$msgs" | grep -Eq '^feat(\([^)]*\))?:'; then stufe=minor; fi
case $stufe in
  major) major=$((major+1)); minor=0; patch=0 ;;
  minor) minor=$((minor+1)); patch=0 ;;
  patch) patch=$((patch+1)) ;;
esac
echo "Letzter Tag: ${letzter:-keiner}  Stufe: $stufe  Naechste Version: $major.$minor.$patch"
EOF
sh next-version.sh
git commit -q --allow-empty -m "feat!: API-Antwortformat geaendert" -m "BREAKING CHANGE: Feld 'name' heisst jetzt 'titel'"
sh next-version.sh

echo "--- Sortierung der Tags"
for t in v1.0.0 v1.0.0-rc.1 v1.0.0-beta.2 v1.0.0-alpha.1 v1.10.0 v1.2.0; do git tag $t; done
git tag --sort=version:refname
echo "--- mit versionsort.suffix"
git -c versionsort.suffix=-alpha -c versionsort.suffix=-beta -c versionsort.suffix=-rc tag --sort=version:refname
```

Das Skript bestimmt zuerst mit `git describe --tags --abbrev=0` den nächsten erreichbaren Tag, zerlegt dessen Nummer mit den Shell-Ersetzungen `%%` und `#` in die drei Teile und liest alle Commit-Betreffs und Nachrichtentexte seit diesem Tag. Standardmäßig ist die Stufe `patch`. Enthält ein Commit `feat:`, wird sie `minor`, und ein Bruchhinweis macht sie `major`. Die erste Ausgabe lautet 1.5.0, weil seit `v1.4.2` ein `feat` dazugekommen ist; der `docs`-Commit hat keinen Einfluss. Nach dem Bruch-Commit ergibt sich 2.0.0. Der zweite Teil zeigt das Sortierproblem: Alphabetisch und selbst mit `--sort=version:refname` steht `v1.0.0-rc.1` *hinter* `v1.0.0`, obwohl es nach SemVer *vor* dem Release liegt. Die Git-Dokumentation erwähnt, dass die Sortierung durch die Einstellung `versionsort.suffix` beeinflusst werden kann (abgerufen am 28.10.2026). Mit den Suffixen `-alpha`, `-beta` und `-rc` stehen die Vorabversionen vor der zugehörigen Release-Version, und die Versionen 1.2.0, 1.4.2, 1.10.0 sind dabei richtig als Zahlen geordnet und nicht als Text.

> **Ergänzung aus der Nachrecherche**
>
> **Regeln der Spezifikation, die oft übersehen werden.** Laut SemVer 2.0.0 (abgerufen am 28.10.2026) haben Vorabversionen eine niedrigere Rangfolge als die zugehörige Normalversion; Bezeichner dürfen nur ASCII-Buchstaben, Ziffern und Bindestriche enthalten, numerische Bezeichner keine führenden Nullen. Beim Vergleich zweier Vorabversionen werden die durch Punkte getrennten Bezeichner von links nach rechts verglichen: rein numerische Bezeichner als Zahlen, andere lexikalisch in ASCII-Reihenfolge, und numerische Bezeichner haben immer eine niedrigere Rangfolge als nichtnumerische (`rc.2` ist also vor `rc.beta`). Sind alle vorangehenden Bezeichner gleich, gewinnt die Version mit mehr Bezeichnern. *Build-Metadaten* (`1.0.0+20130313144700`) werden bei der Rangfolge ignoriert. Praktisch folgt daraus eine Regel, die wir empfehlen (sie steht nicht im abgerufenen Auszug): Eine einmal ausgelieferte Versionsnummer wird nie für anderen Inhalt wiederverwendet, passend zur Empfehlung der Git-Dokumentation für Tags.

### Typische Fehler und Stolperfallen.

Falsch ist, SemVer als reine Zählung zu betrachten und bei jedem Release „irgendwie“ die Minor-Zahl zu erhöhen. Dann ist die Nummer wertlos, weil niemand ihr vertrauen kann. Richtig ist, die Schnittstelle zu definieren und Brüche konsequent mit Major zu kennzeichnen. Zweiter Fehler: Einen Bruch als Patch auszuliefern, weil er „klein“ sei. Für Nutzer zählt, ob ihr Code weiter funktioniert, nicht, wie viele Zeilen sich geändert haben. Dritter Fehler: Die Nummer von Hand in mehreren Dateien pflegen. Das führt zu Abweichungen zwischen Tag, Paketdatei und Anzeige; die Nummer sollte aus einer einzigen Quelle kommen (Abschnitt 7.8). Vierter Fehler: Tags beim Sortieren als Text behandeln, sodass `v1.10.0` vor `v1.2.0` erscheint.

### Praxisbezug.

Nutzer von Bibliotheken verlassen sich auf SemVer, wenn sie Abhängigkeiten mit Bereichen angeben („alle 1.x“). Ein versehentlicher Bruch in einer Minor-Version bricht dann viele fremde Builds. Intern hilft die Konvention bei der Kommunikation: „2.0.0“ heißt für alle Beteiligten, dass Migrationsaufwand wartet. In Teams, die Conventional Commits nutzen, wird die Versionsnummer zunehmend aus den Commits berechnet, wie im Beispiel.

## 7.3 Release Notes

Release Notes beantworten für Nutzer und Beteiligte, was sich in einer Version geändert hat. Wichtig ist die Zielgruppe: Entwicklerinnen und Entwickler möchten technische Details, Betrieb und Support wollen wissen, was beim Update zu beachten ist, Fachbereiche interessieren sich für neue Funktionen. Die Seite *Keep a Changelog* (Version 1.1.0, abgerufen am 28.10.2026) formuliert dazu sieben Grundsätze, deren erster lautet, dass Changelogs für Menschen gemacht werden und nicht für Maschinen. Weiter wird verlangt: Jede Version bekommt einen Eintrag, ähnliche Änderungen werden gruppiert, die neueste Version steht oben, jede Version trägt ein Datum im Format JJJJ-MM-TT, und der Changelog sagt, ob das Projekt SemVer befolgt. Als Gruppen schlägt die Seite *Added*, *Changed*, *Deprecated*, *Removed*, *Fixed* und *Security* vor und empfiehlt einen Abschnitt *Unreleased* am Anfang für kommende Änderungen.

Aus Git lassen sich Release Notes ableiten, aber nicht blind. Das Rohmaterial sind Commit-Nachrichten und die Titel der Pull Requests zwischen zwei Tags: `git log v1.0.0..v1.1.0` liefert genau diese Menge (Modul 06, Bereichssyntax). Das gelingt umso besser, je disziplinierter die Nachrichten sind; mit Conventional Commits lässt sich nach Typ gruppieren. Ein vollautomatischer Changelog ist trotzdem ein Rohentwurf: Commit-Nachrichten sind für Entwickelnde geschrieben („fix: NPE in Parser“), Release Notes sollen für die Zielgruppe verständlich sein („Behoben: Absturz beim Import leerer Dateien“). Gute Teams generieren den Entwurf und redigieren ihn anschließend. Das Beispiel zeigt die Erzeugung eines Abschnitts im Stil von Keep a Changelog.

```bash
git init -q -b main cl && cd cl
git config user.name Dev; git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: Grundfunktion"
git tag -a v1.0.0 -m "Release 1.0.0"
for m in "feat: CSV-Export" "fix: Rundungsfehler bei Summen" "fix(api): Timeout beim Login" "docs: README ergaenzt" "chore: Abhaengigkeiten aktualisiert" "feat!: Antwortformat der API geaendert" "Merge-Ticket ohne Praefix"; do
  git commit -q --allow-empty -m "$m"
done
git tag -a v1.1.0 -m "Release 1.1.0"

cat > changelog.sh <<'EOF'
#!/bin/sh
# Erzeugt einen Changelog-Abschnitt (Keep-a-Changelog-Stil) aus Conventional Commits
von=$1; bis=$2
datum=$(git log -1 --format=%cs "$bis")
echo "## [${bis#v}] - $datum"
abschnitt() {
  titel=$1; muster=$2
  zeilen=$(git log --format='%s' "$von..$bis" | grep -E "$muster" | sed -E 's/^[a-z]+(\(([^)]*)\))?!?: /\2|/; s/^\|//' )
  if [ -n "$zeilen" ]; then
    echo; echo "### $titel"
    echo "$zeilen" | sed 's/^\([^|]*\)|/\1: /; s/^/- /'
  fi
}
abschnitt "Breaking Changes" '^[a-z]+(\([^)]*\))?!:'
abschnitt "Added"  '^feat(\([^)]*\))?:'
abschnitt "Fixed"  '^fix(\([^)]*\))?:'
EOF
sh changelog.sh v1.0.0 v1.1.0
echo "--- Zusammenfassung nach Person"
git shortlog -sn v1.0.0..v1.1.0
echo "--- Alle Commits ohne Praefix"
git log --format='%s' v1.0.0..v1.1.0 | grep -vE '^(feat|fix|docs|chore)(\(.*\))?!?:'
```

Das Skript nimmt zwei Tags entgegen, liest mit `git log -1 --format=%cs` das Datum des neueren (`%cs` ist das Commit-Datum in Kurzform JJJJ-MM-TT) und gruppiert die Commit-Betreffs zwischen den Tags nach Typ. Die Hilfsfunktion `abschnitt` filtert mit `grep` nach einem Muster, schneidet den Typ-Präfix mit `sed` ab und stellt einen optionalen Bereich in Klammern (`api:`) wieder voran. Das Ergebnis enthält die Gruppen „Breaking Changes“, „Added“ und „Fixed“; die Einträge für `docs` und `chore` erscheinen absichtlich nicht, weil sie für Nutzer selten relevant sind. Interessant ist die letzte Ausgabe: Der Commit „Merge-Ticket ohne Praefix“ folgt keiner Konvention und fällt durch das Raster. Genau diese Zeile ist der Grund, warum man einen automatisch erzeugten Changelog durchsehen muss und die Konvention per Hook oder Pipeline durchsetzt (Modul 05, Modul 09).

### Typische Fehler und Stolperfallen.

Falsch ist es, den rohen Ausgabetext von `git log` als Release Notes zu veröffentlichen. Er enthält Interna, Tippfehler-Commits und Merge-Commits und ist für die Leser nicht geschrieben. Richtig ist ein redigierter Entwurf. Zweiter Fehler: Sicherheitsrelevante Änderungen unter „Fixed“ zu verstecken. Gerade die Kategorie *Security* ist wichtig, damit Betreiber wissen, dass sie dringend aktualisieren sollten. Dritter Fehler: Breaking Changes ohne Migrationshinweis; eine Zeile „API geändert“ hilft niemandem, ein Hinweis mit dem Vorher/Nachher und einem Beispiel schon. Vierter Fehler: Der Changelog wird erst am Release-Tag geschrieben, wenn niemand mehr weiß, was die Änderungen sollten; besser wächst der Abschnitt *Unreleased* mit jedem Pull Request.

### Praxisbezug.

Viele Plattformen können beim Anlegen eines Releases einen Textentwurf aus den Pull Requests seit dem letzten Release erzeugen (weiterführend, hier nicht abgerufen). Das Ergebnis dient als Ausgangspunkt für Release Notes, die man für die Zielgruppen aufbereitet: ein technischer Teil für Entwicklung und Betrieb, eine kurze Zusammenfassung für Stakeholder und gegebenenfalls ein Migrationsleitfaden. In regulierten Umgebungen sind Release Notes zugleich Teil des Änderungsnachweises.

## 7.4 Signed Tags

Ein Tag sagt „Version 1.4.2 ist dieser Commit“. Aber wer sagt das, und kann man es glauben? Ohne Signatur kann jeder mit Schreibrecht einen Tag mit beliebigem Namen anlegen, und das Taggerfeld im Tag-Objekt ist wie das Autorenfeld frei wählbar (Modul 08, Abschnitt 8.3). Ein *signierter Tag* bindet das Release kryptografisch an die Person oder das System, die es freigegeben hat. Wer das Release verwendet, kann prüfen, dass der Tag von einem bekannten Schlüssel stammt und dass das Tag-Objekt (und damit der Commit, auf den es zeigt, samt seiner gesamten Historie) seither nicht verändert wurde. Das schützt vor einem Angriff, bei dem ein Tag auf manipulierten Code umgebogen wird.

Die Technik entspricht der der signierten Commits: `git tag -s` signiert mit dem Standardschlüssel, `-u <key>` mit einem bestimmten Schlüssel, und die Dokumentation (abgerufen am 28.10.2026) verweist für das Format auf `gpg.format`; Standard ist `openpgp`, `ssh` und `x509` sind Alternativen. Die Einstellung `tag.gpgSign` signiert alle Tags automatisch, `user.signingKey` nennt den Standardschlüssel. Geprüft wird mit `git tag -v` oder `git verify-tag`. Ein wichtiger Hinweis für die Einrichtung: Für SSH-Signaturen benötigt Git eine Datei erlaubter Signierer (Modul 08), sonst gilt keine Signatur als vertrauenswürdig. Auch hier schreiben wir ein Skript, das als Prüfstufe in einer Pipeline dienen kann: Es lässt nur signierte, gültige annotierte Tags durch.

```bash
mkdir -p ~/.ssh ~/.config/git
ssh-keygen -q -t ed25519 -N "" -C "release@example.org" -f ~/.ssh/id_release
echo "release@example.org namespaces=\"git\" $(cut -d' ' -f1,2 ~/.ssh/id_release.pub)" > ~/.config/git/allowed_signers
git init -q -b main rel && cd rel
git config user.name "Release Manager"; git config user.email release@example.org
git config gpg.format ssh
git config user.signingkey ~/.ssh/id_release.pub
git config gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
echo 1 > app.txt && git add . && git commit -q -m "feat: Start"

git tag -s v1.0.0 -m "Release 1.0.0"
git tag -a v1.0.1 -m "Release 1.0.1 (unsigniert)"
git tag v1.0.2
git cat-file -p v1.0.0 | sed -n '1,6p;/BEGIN/p;/END/p'
git verify-tag v1.0.0; echo "Exit: $?"
git verify-tag v1.0.1; echo "Exit: $?"
git verify-tag v1.0.2; echo "Exit: $?"

cat > pruefe-release.sh <<'EOF'
#!/bin/sh
# Gate fuer die Pipeline: nur signierte, gueltige Release-Tags duerfen gebaut werden
tag=$1
if [ "$(git cat-file -t "refs/tags/$tag" 2>/dev/null)" != "tag" ]; then
  echo "ABBRUCH: $tag ist kein annotierter Tag"; exit 1
fi
if git verify-tag "$tag" >/dev/null 2>&1; then
  echo "OK: $tag ist signiert und gehoert zu einem erlaubten Signierer"
else
  echo "ABBRUCH: Signatur von $tag fehlt oder ist nicht vertrauenswuerdig"; exit 1
fi
EOF
for t in v1.0.0 v1.0.1 v1.0.2; do sh pruefe-release.sh $t; done

# Manipulation: Tag-Objekt wird veraendert
git cat-file tag v1.0.0 | sed 's/Release 1.0.0/Release 9.9.9/' > ../gefaelscht.txt
neu=$(git hash-object -t tag -w ../gefaelscht.txt)
git update-ref refs/tags/v1.0.0-fake "$neu"
git verify-tag v1.0.0-fake; echo "Exit: $?"
```

Der signierte Tag `v1.0.0` enthält im Tag-Objekt, wie die gekürzte Ausgabe von `git cat-file -p` zeigt, die üblichen Felder (Objekt, Typ, Tag-Name, Tagger, Nachricht) und anschließend einen Block `BEGIN SSH SIGNATURE` bis `END SSH SIGNATURE`. `git verify-tag` meldet für ihn „Good "git" signature“ mit Exit-Code 0. Für den annotierten, aber unsignierten Tag meldet Git „no signature found“, für den Lightweight Tag „cannot verify a non-tag object of type commit“, beide mit Exit-Code 1. Das Gate-Skript fasst das zusammen: Es prüft zuerst mit `git cat-file -t`, ob der Name ein Tag-Objekt ist, und lässt danach `git verify-tag` entscheiden. Nur `v1.0.0` kommt durch. Die Fälschung am Ende ersetzt im Tag-Objekt den Nachrichtentext, legt das veränderte Objekt mit `git hash-object` in der Datenbank ab und lässt einen neuen Tag darauf zeigen. Die Signatur bleibt im Objekt erhalten, passt aber nicht mehr zum Inhalt, und `git verify-tag` meldet „incorrect signature“. Eine Veränderung, auch nur eines Zeichens, wird erkannt.

### Typische Fehler und Stolperfallen.

Falsch ist, signierte Tags einzuführen, ohne dass jemand die Signatur prüft. Eine ungeprüfte Signatur schützt nicht; die Prüfung gehört ins Release-Verfahren, in die Pipeline oder in die Installationsanleitung. Zweiter Fehler: Den Signierschlüssel im Repository oder auf einem gemeinsam genutzten Rechner abzulegen. Der Schlüssel eines Release-Managers ist ein hochwertiges Ziel; er gehört auf ein Hardware-Token oder zumindest passwortgeschützt in die Hand weniger Personen. Dritter Fehler: Aus „signiert“ abzuleiten, dass der Inhalt gut oder sicher ist; die Signatur belegt nur die Herkunft. Vierter Fehler: Die Liste der erlaubten Signierer nicht zu pflegen, sodass nach einem Personalwechsel noch ausgeschiedene Schlüssel gelten.

### Praxisbezug.

Bei Open-Source-Projekten ist die Signatur des Release-Tags der Anker, an dem Paketbauer und Nutzer prüfen können, dass sie das echte Release verwenden. In Unternehmen verlangen Compliance-Vorgaben oft, dass nur signierte Tags von einer kleinen Gruppe Release-Verantwortlicher Auslieferungen auslösen dürfen. Verbunden mit dem Tag-Schutz aus Modul 08 (Tags nicht verschiebbar, nur wenige dürfen anlegen) entsteht ein belastbarer Releaseprozess.

## 7.5 Release Branches und Hotfixes

Ein Release-Branch entsteht, wenn man eine Version stabilisieren und pflegen will, während auf dem Hauptbranch bereits die nächste Version entsteht. Die Strategie hinter dieser Entscheidung ist Thema von Modul 05; hier geht es um den Ablauf bei einem Fehler im ausgelieferten Release. Er heißt *Hotfix* oder *Backport*. Die typische Lage: Version 1.2.0 ist bei Kunden im Einsatz, auf `main` ist die Entwicklung schon bei 1.3, und es wird ein schwerer Fehler in 1.2 gemeldet. Man will 1.2.1 ausliefern, und zwar mit genau dem Fix und ohne die unfertigen Funktionen von 1.3.

Der saubere Ablauf hat vier Schritte. Der Fix wird *zuerst auf dem Hauptbranch* entwickelt und reviewt, damit er nicht in der nächsten Version wieder fehlt. Dann wird derselbe Commit mit `git cherry-pick -x` auf den Release-Branch übernommen (Modul 03); die Option `-x` hängt die Zeile „(cherry picked from commit ...)“ an die Nachricht an, was die Herkunft dokumentiert. Auf dem Release-Branch läuft die Pipeline, und dort wird der neue Tag `v1.2.1` gesetzt. Zuletzt dokumentiert man den Backport, damit später nachvollziehbar ist, welche Version den Fix enthält. Das Beispiel führt das einmal durch und zeigt zugleich, warum die einfache Frage „Ist dieser Fix in Release X?“ mit `git tag --contains` nicht ausreicht.

```bash
git init -q -b main hf && cd hf
git config user.name Dev; git config user.email dev@example.org
echo "version 1.2" > app.txt && git add . && git commit -q -m "feat: Release-Stand 1.2"
git tag -a v1.2.0 -m "Release 1.2.0"
git branch release/1.2
echo "neue Funktion" > neu.txt && git add . && git commit -q -m "feat: neue Funktion fuer 1.3"
echo "fix" > fix.txt && git add . && git commit -q -m "fix: Absturz beim Speichern (Ticket 77)"
FIX=$(git rev-parse HEAD)
echo "weiter" >> neu.txt && git commit -q -am "feat: Funktion erweitert"

git switch -q release/1.2
git cherry-pick -x "$FIX" >/dev/null
git tag -a v1.2.1 -m "Release 1.2.1 (Hotfix)"
git log --oneline --decorate
git log -1 --format=%B
git switch -q main
git cherry -v release/1.2 main
git branch --contains "$FIX"
git tag --contains "$FIX"
git describe --tags release/1.2
```

Der Branch `release/1.2` wird beim Release angelegt und bleibt bei diesem Stand stehen. Auf `main` entstehen eine neue Funktion, der Fix und eine Erweiterung. Der Fix wird mit `git cherry-pick -x` in den Release-Branch übernommen, und der Tag `v1.2.1` bezeichnet diesen Stand. Das Log des Release-Branches zeigt nur zwei Commits: den ursprünglichen Release-Stand und den Fix, aber nicht die neue Funktion. Die Nachricht des übernommenen Commits enthält die Herkunftszeile mit dem Hash des Originals. Zurück auf `main` beantwortet `git cherry -v release/1.2 main` die Frage, was noch nicht auf dem Release-Branch ist: Es vergleicht Änderungen statt Hashes und markiert mit `+` Commits, die der Release-Branch nicht hat, mit `-` solche, deren Änderung dort schon enthalten ist. Der Fix trägt das Minuszeichen; die beiden Funktionscommits bleiben `+`. Dagegen listet `git tag --contains` für den ursprünglichen Fix-Hash keinen Tag, und `git branch --contains` nur `main`, denn der Cherry-Pick hat einen *neuen* Commit mit anderem Hash erzeugt. Diese Tücke ist der Grund, warum Backports dokumentiert werden müssen.

### Typische Fehler und Stolperfallen.

Falsch ist, den Fix nur auf dem Release-Branch zu machen. Er fehlt dann in der nächsten Version, und der Fehler kehrt mit 1.3 zurück. Richtig ist: erst `main`, dann Backport (oder, wenn der Fix nur für alte Versionen gilt, bewusst und dokumentiert anders). Zweiter Fehler: Den ganzen `main` auf den Release-Branch zu mergen, um „nur den Fix“ mitzunehmen; so gelangen unfertige Funktionen in die Auslieferung. Dritter Fehler: Backports nicht zu dokumentieren und bei „Ist Fix X in Version 1.2.3?“ auf das Gedächtnis zu vertrauen. Hilfreich ist die `-x`-Zeile, ein einheitlicher Hinweis in der Nachricht oder im Ticket und ein Eintrag im Changelog der Release-Linie. Vierter Fehler: Hotfixes ohne Tests auszuliefern, weil es eilt; gerade dort passieren Folgefehler.

### Praxisbezug.

Teams mit mehreren unterstützten Versionen führen eine Tabelle: Welche Linien (1.2, 1.3, 2.0) bekommen Fixes, bis wann, und wer entscheidet über Backports? Diese Tabelle gehört zusammen mit den unterstützten Versionen in die Security-Policy (Modul 08, Abschnitt 8.9). Plattformen unterstützen Backports teilweise durch Bots, die für gelabelte Pull Requests automatisch Cherry-Pick-Pull-Requests auf Release-Branches erzeugen (weiterführend, hier nicht abgerufen).

## 7.6 Artefakt-Bezug

Ein Release-Tag allein genügt noch nicht, um Auslieferungen nachvollziehbar zu machen. Das, was bei Kunden oder in Produktion läuft, ist ein *Artefakt*: ein Archiv, ein Installationspaket, ein Container-Image. Die Frage in einem Vorfall lautet: „Aus welchem Quellstand wurde genau dieses Artefakt gebaut?“ Die Kette vom Code bis zur Produktion muss lückenlos sein: Commit, Tag, Build, Artefakt, Deployment. Jedes Glied soll das vorherige eindeutig benennen und sich selbst identifizieren lassen. Das Mittel dafür ist die Verwendung stabiler Kennungen: der Commit-Hash (er ist ein Fingerabdruck des Inhalts, Modul 01), der Tag-Name und eine Prüfsumme des Artefakts.

Es gibt drei bewährte Verfahren. Erstens trägt jedes Artefakt seine Herkunft in sich: Die Versionsangabe, die in der Anwendung angezeigt wird (zum Beispiel unter „Über“ oder per `--version`), enthält den Commit, etwa aus `git describe --tags --always --dirty`. Das Suffix `-dirty` warnt, falls aus einem Arbeitsverzeichnis mit uncommitteten Änderungen gebaut wurde. Zweitens entsteht das Quellarchiv aus Git selbst und nicht aus dem Arbeitsverzeichnis: `git archive` packt einen Commit oder Tag; es enthält nur versionierte Dateien, keine Build-Reste. Git schreibt den Commit-Hash in den Archivkopf, sodass sich auch aus einem Archiv ohne Metadaten der Ursprung ermitteln lässt: `git get-tar-commit-id`. Drittens hält ein *Manifest* die Verknüpfung fest: Name des Artefakts, Tag, Commit, Prüfsumme. Die Prüfsumme ermöglicht es, später zu beweisen, dass ein vorliegendes Artefakt das ursprünglich gebaute ist. Für Wiederholbarkeit sorgt, dass der Build deterministisch ist: Zwei Builds desselben Tags sollten bitgleiche Ergebnisse liefern. Das Beispiel prüft das mit `git archive` und `gzip -n`; die Option `-n` lässt Name und Zeitstempel aus dem gzip-Kopf weg, die sonst jedes Archiv verändern würden.

```bash
git init -q -b main art && cd art
git config user.name Dev; git config user.email dev@example.org
echo 'print("hallo")' > app.py && echo "hallo" > README.md && git add . && git commit -q -m "feat: Start"
git tag -a v1.0.0 -m "Release 1.0.0"
echo '# neu' >> app.py && git commit -q -am "feat: weiter"

VERSION=$(git describe --tags --always --dirty)
SHA=$(git rev-parse --short=12 HEAD)
echo "Build von: $VERSION ($SHA)"
git archive --format=tar --prefix=app-1.0.0/ v1.0.0 | gzip -n > ../app-1.0.0.tar.gz
git archive --format=tar --prefix=app-1.0.0/ v1.0.0 | gzip -n > ../app-1.0.0-zweiter-lauf.tar.gz
sha256sum ../app-1.0.0.tar.gz ../app-1.0.0-zweiter-lauf.tar.gz | cut -c1-16,65-
zcat ../app-1.0.0.tar.gz | git get-tar-commit-id
git rev-parse v1.0.0^{commit}
tar tzf ../app-1.0.0.tar.gz

cat > ../manifest.txt <<EOF
artefakt=app-1.0.0.tar.gz
tag=v1.0.0
commit=$(git rev-parse v1.0.0^{commit})
sha256=$(sha256sum ../app-1.0.0.tar.gz | cut -d' ' -f1)
EOF
cut -c1-60 ../manifest.txt
# Rueckverfolgung vom Artefakt zum Commit
ID=$(zcat ../app-1.0.0.tar.gz | git get-tar-commit-id)
git tag --points-at "$ID"
git log -1 --format='%h %an %s' "$ID"
```

Der Build läuft absichtlich auf einem Commit *nach* dem Tag, um den Unterschied zu zeigen: `git describe` liefert „v1.0.0-1-g...“ und macht damit sichtbar, dass dieser Stand nicht dem Release entspricht. Das Archiv wird aus dem *Tag* erzeugt, nicht aus dem Arbeitsverzeichnis, und enthält nur `app.py` und `README.md` unter dem Präfix `app-1.0.0/`. Die beiden Archive aus zwei Läufen haben dieselbe Prüfsumme (die Ausgabe ist zur Lesbarkeit gekürzt), der Build ist also reproduzierbar. `git get-tar-commit-id` liest den Commit aus dem Archiv und gibt denselben Hash aus wie `git rev-parse v1.0.0^{commit}`. Das Manifest hält Tag, Commit und Prüfsumme zusammen. Die Rückverfolgung am Ende geht den umgekehrten Weg: vom Artefakt zum Commit-Hash, zum Tag (`git tag --points-at`) und zum Eintrag in der Historie.

### Typische Fehler und Stolperfallen.

Falsch ist, ein Artefakt aus einem lokalen Arbeitsverzeichnis zu bauen und auszuliefern. Dann stimmt der Quellstand mit nichts überein, was im Repository steht. Richtig ist, nur in der Pipeline aus einem sauberen Checkout eines Tags zu bauen. Zweiter Fehler: Artefakte ohne Versionsangabe, sodass nach der Auslieferung niemand mehr weiß, welche Version läuft. Dritter Fehler: Zeitstempel, Dateireihenfolge oder Hostnamen fließen in das Artefakt ein, sodass jeder Build anders aussieht und Prüfsummen unbrauchbar werden. Vierter Fehler: Das Manifest wird nicht aufbewahrt oder liegt neben dem Artefakt ohne eigenen Schutz; es sollte mit dem Release-Tag zusammen unveränderlich abgelegt werden.

### Praxisbezug.

Bei Container-Images entspricht der Bezug dem Beispiel: Das Image trägt als Tag die Version und als Label den Commit-Hash, und das Deployment verweist auf die Image-Kennung. Im Incident fragt der Betrieb: „Welche Version läuft in Produktion?“, und die Antwort ist ein Hash, der sich in Sekunden bis zum Commit, zum Pull Request und zum Tag zurückverfolgen lässt. Dasselbe Prinzip ist die Grundlage dafür, später Herkunftsnachweise für Builds zu erzeugen und zu prüfen (weiterführend, hier nicht abgerufen).

## 7.7 Pre-Releases

Nicht jedes Release ist sofort für alle bestimmt. Vor einer größeren Version will man Rückmeldung von Testern oder ausgewählten Kunden einholen. Dazu dienen *Vorabversionen*. SemVer bildet sie mit einem Suffix ab, das durch einen Bindestrich an die Version angehängt wird: `2.0.0-alpha.1`, `2.0.0-beta.1`, `2.0.0-rc.1`. Nach der Spezifikation (abgerufen am 28.10.2026) kennzeichnet ein solches Suffix eine instabile Version, die eine niedrigere Rangfolge hat als die zugehörige Normalversion; `2.0.0-rc.2` ist also älter als `2.0.0`. Die gängigen Namen drücken eine Reife aus: *alpha* steht für unvollständige Funktionen und viel Änderung, *beta* für funktionsvollständig, aber noch fehlerbehaftet, *rc* (Release Candidate) für einen Kandidaten, der unverändert zum Release wird, wenn keine Blocker auftreten.

Vorabversionen müssen im Prozess abgebildet werden. Für jede Stufe gehört ein eigener Tag, der Test- und Freigabeprozess legt fest, welche Kriterien zum Wechsel von einer Stufe zur nächsten führen (zum Beispiel: Beta nur mit abgeschlossenen Funktionen, RC nur mit Fehlerfreiheit der Blocker-Klasse), und es muss klar sein, wer Vorabversionen konsumieren darf. Besonders wichtig ist die Regel, dass ein *Release Candidate unverändert befördert wird*: Wird der RC getestet und für gut befunden, ist das Release derselbe Commit mit einem neuen Tag. Baut man nach dem RC noch einmal neu, ist das Release nicht mehr das, was getestet wurde. Das Beispiel zeigt die Tag-Folge, die Sortierung und die Beförderung.

```bash
git init -q -b main pr && cd pr
git config user.name Dev; git config user.email dev@example.org
git config versionsort.suffix -alpha; git config --add versionsort.suffix -beta; git config --add versionsort.suffix -rc
echo 1 > a && git add . && git commit -q -m "feat: Basis"
git tag -a v1.9.0 -m "Release 1.9.0"
echo 2 >> a && git commit -q -am "feat: grosse Neuerung"
git tag -a v2.0.0-alpha.1 -m "Alpha 1"
echo 3 >> a && git commit -q -am "fix: Alpha-Feedback"
git tag -a v2.0.0-beta.1 -m "Beta 1"
echo 4 >> a && git commit -q -am "fix: Beta-Feedback"
git tag -a v2.0.0-rc.1 -m "Release Candidate 1"
echo 5 >> a && git commit -q -am "fix: Blocker aus RC1"
git tag -a v2.0.0-rc.2 -m "Release Candidate 2"
git tag --list --sort=-version:refname
echo "--- neueste stabile Version"
git tag --list 'v[0-9]*' --sort=-version:refname | grep -Ev -- '-(alpha|beta|rc)' | head -1
echo "--- neueste Version inkl. Vorabversionen"
git tag --list 'v[0-9]*' --sort=-version:refname | head -1

# Befoerderung: der getestete RC wird unveraendert zum Release
git tag -a v2.0.0 -m "Release 2.0.0" v2.0.0-rc.2^{commit}
git tag --list --sort=-version:refname | head -3
git rev-parse v2.0.0^{commit} v2.0.0-rc.2^{commit} | uniq | wc -l
git describe --tags
```

Die Konfiguration `versionsort.suffix` sorgt dafür, dass die drei Suffixe vor der zugehörigen Normalversion einsortiert werden. Die absteigende Liste zeigt dann `v2.0.0-rc.2` oben und `v1.9.0` unten. Die Frage „Was ist die neueste *stabile* Version?“ beantwortet die Filterung mit `grep -Ev` auf die Suffixe: Das Ergebnis ist `v1.9.0`. Die Frage nach der neuesten Version einschließlich Vorabversionen liefert `v2.0.0-rc.2`. Diese Unterscheidung ist wichtig, weil automatische Update-Mechanismen standardmäßig nur stabile Versionen anbieten sollten. Bei der Beförderung wird der Tag `v2.0.0` auf denselben Commit gesetzt wie der RC; die Kontrolle mit `git rev-parse` und `uniq` gibt eine einzige Zeile aus und bestätigt damit, dass beide Tags auf denselben Commit zeigen. `git describe --tags` bevorzugt bei gleichem Commit den neueren und gibt jetzt `v2.0.0` aus.

### Typische Fehler und Stolperfallen.

Falsch ist, nach dem letzten RC noch „nur eine Kleinigkeit“ zu ändern und dann zu veröffentlichen; was ausgeliefert wird, wurde dann nicht getestet. Richtig ist: Jede Änderung nach dem RC erzeugt einen neuen RC. Zweiter Fehler: Vorabversionen ohne Suffix zu führen (`2.0.0` als „Beta“), sodass Nutzer sie für stabil halten und automatisches Update sie verteilt. Dritter Fehler: Die Sortierung der Tags ungeprüft zu lassen und dann `v1.0.0-rc.1` als neuer als `v1.0.0` anzusehen (Abschnitt 7.2). Vierter Fehler: Vorabversionen auf unbestimmte Zeit stehen zu lassen. Wenn ein Alpha monatelang der Dauerzustand ist, fehlt der Prozess, der aus ihm ein Release macht.

### Praxisbezug.

Typische Anwendungen sind ein Beta-Programm für Kunden, ein interner RC für die Abnahme durch Fachbereich und Betrieb oder ein Nightly-Kanal, der Entwicklungsstände bereitstellt. Die zugehörigen Freigabeprozesse (wer testet was, wer gibt frei) gehören in ein Runbook (Modul 10), und die Pipeline kennzeichnet Vorabversionen auf der Plattform als solche (weiterführend, hier nicht abgerufen).

## 7.8 Versionierung in CI/CD

Versionsnummern von Hand zu pflegen ist fehleranfällig: Man vergisst, die Nummer zu erhöhen, vertippt sich oder hat Tag und Paketdatei nicht synchron. Deshalb leitet man die Version *automatisch aus Git ab*, und zwar so, dass genau eine Quelle der Wahrheit besteht: der Tag. Das Werkzeug dafür ist `git describe`. Auf einem getaggten Commit liefert es den Tag; auf einem Commit danach liefert es Tag, Anzahl der Commits und Hash. Daraus entsteht ein Schema, das Release-Builds und Zwischenstände klar trennt. Auf einem Tag `v1.4.0` ist die Version `1.4.0`; drei Commits nach dem Tag ist sie `1.4.1-dev.3+g92536d8`: die nächste Patch-Version als *Vorabversion* (niedriger als 1.4.1) mit der Zahl der Commits seit dem Tag und dem Commit-Hash als *Build-Metadaten*. Laut SemVer werden Build-Metadaten bei der Rangfolge ignoriert; sie dienen der Identifikation. Dieses Schema kombiniert Git-SHA und SemVer, wie es der Lehrplan fordert.

Eine Stolperfalle der CI ist der *flache Klon*. Viele Pipeline-Systeme klonen standardmäßig nur einen Commit (`--depth=1`), weil das schneller ist (Modul 05). Dann liegt der Tag, an dem `git describe` ansetzt, weder in der geladenen Historie noch unter den geholten Referenzen, und der Befehl scheitert. Die Lösung ist, in der Versionierungsstufe die Historie nachzuladen: `git fetch --unshallow --tags`. Das folgende Skript berechnet die Version nach dem beschriebenen Schema, bricht verständlich ab, wenn kein Tag erreichbar ist, und zeigt das Verhalten im flachen Klon.

```bash
git init -q -b main ci && cd ci
git config user.name Dev; git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: Start"
git tag -a v1.4.0 -m "Release 1.4.0"
for i in 1 2 3; do echo $i >> a; git commit -q -am "fix: Korrektur $i"; done
cd ..
cat > version.sh <<'EOF'
#!/bin/sh
# Ermittelt die Build-Version aus Git:
#   exakt auf einem Tag        -> 1.4.0
#   n Commits nach dem Tag     -> 1.4.1-dev.n+g<sha>
beschr=$(git describe --tags --match 'v[0-9]*' --long 2>/dev/null) || {
  echo "FEHLER: kein Tag erreichbar (flacher Klon ohne Tags?)" >&2; exit 1; }
sha=${beschr##*-g}; rest=${beschr%-g*}; n=${rest##*-}; tag=${rest%-*}
basis=${tag#v}
if [ "$n" -eq 0 ]; then
  echo "$basis"
else
  major=${basis%%.*}; r=${basis#*.}; minor=${r%%.*}; patch=${r#*.}; patch=${patch%%-*}
  echo "$major.$minor.$((patch+1))-dev.$n+g$sha"
fi
EOF
cd ci && sh ../version.sh
git tag -a v1.4.1 -m "Release 1.4.1"; sh ../version.sh
echo x >> a; git commit -q -am "fix: danach"; sh ../version.sh
cd ..
echo "--- flacher Klon (wie viele CI-Systeme standardmaessig)"
git clone -q --depth=1 file://$PWD/ci flach
cd flach && sh ../version.sh; echo "Exit: $?"
git fetch -q --unshallow --tags origin; sh ../version.sh
```

Das Skript ruft `git describe --tags --long` auf und zerlegt die Ausgabe der Form „v1.4.0-3-g92536d8“ von hinten: erst den Hash hinter „-g“, dann die Anzahl und schließlich den Tag. Steht der Zähler auf null, ist der Commit exakt getaggt, und die Version ist die Tag-Nummer ohne `v`. Andernfalls wird die Patch-Stelle erhöht und der Rest als Suffix angehängt. Die erste Ausgabe ist `1.4.1-dev.3+g...`, nach dem Setzen von `v1.4.1` nur noch `1.4.1`, nach einem weiteren Commit `1.4.2-dev.1+g...`. Im flachen Klon schlägt die Berechnung fehl, weil in der flachen Kopie weder die getaggten Commits noch die Tags selbst liegen (im Test enthielt der Klon null Tags); `git fetch --unshallow --tags` lädt den Rest der Historie nach, danach liefert das Skript dasselbe Ergebnis wie im Original.

> **Merke**
>
> Die Version wird nicht gepflegt, sondern *abgeleitet*: Der Tag ist die einzige Quelle. Release-Builds laufen nur auf getaggten Commits, Zwischenstände tragen ein eindeutiges Vorabsuffix mit Commit-Hash. In der Pipeline muss die Historie vollständig genug sein, damit `git describe` funktioniert.

### Typische Fehler und Stolperfallen.

Falsch ist, die Versionsnummer in der Pipeline per Hand zu setzen oder aus einem Zähler der Pipeline zu übernehmen. Dann besteht kein Bezug mehr zum Quellstand. Richtig ist die Ableitung aus Git. Zweiter Fehler: flacher Klon ohne Tags in der Version-Stufe; das Ergebnis ist ein Abbruch oder, schlimmer, eine falsche Nummer. Dritter Fehler: Zwischenstände mit dem Namen einer künftigen Release-Nummer zu benennen (`1.4.1` für einen Entwicklungsstand), sodass sich später zwei verschiedene Artefakte dieselbe Version teilen. Das Vorabsuffix verhindert das. Vierter Fehler: Die Versionsberechnung in jeder Pipeline anders zu implementieren; sie gehört in ein gemeinsames Skript oder eine wiederverwendbare Pipeline-Komponente.

### Praxisbezug.

In der Praxis kommt dieselbe Idee in vielen Varianten vor: Versionswerkzeuge der Build-Systeme, die aus dem Tag die Paketversion ableiten, oder Pipeline-Schritte, die bei einem Merge auf `main` anhand der Commit-Konvention die nächste Version berechnen, den Tag setzen und den Release-Build anstoßen (weiterführend, hier nicht abgerufen). Das Ergebnis ist ein Prozess, in dem Menschen nur noch Commits und Pull Requests schreiben; Nummer, Tag, Changelog und Artefakt folgen daraus. Dazu gehört dann der Schutz der Tags und die Signaturprüfung aus Abschnitt 7.4 und Modul 08.

## Übungen zu Modul 07

**Aufgabe 1.** Worin unterscheiden sich Lightweight und annotierte Tags, und warum verwendet man für Releases annotierte Tags?

**Aufgabe 2.** Eine Bibliothek ist bei Version 2.3.1. Welche Nummer ergibt sich jeweils bei (a) einem Fehlerfix, (b) einer neuen abwärtskompatiblen Funktion, (c) dem Entfernen einer öffentlichen Funktion?

**Aufgabe 3.** Warum steht `v1.0.0-rc.1` bei der Sortierung mit `--sort=version:refname` hinter `v1.0.0`, und wie behebt man das?

**Aufgabe 4.** Ein Fix wurde per Cherry-Pick in den Release-Branch übernommen. Warum liefert `git tag --contains <ursprünglicher-fix>` keinen Release-Tag, und wie dokumentiert man den Backport?

**Aufgabe 5.** Praktisch: Legen Sie ein Repository mit Conventional Commits und zwei Tags an. Schreiben Sie ein Skript, das aus den Commits zwischen den Tags einen Changelog-Abschnitt mit den Gruppen *Added* und *Fixed* erzeugt. Welche Commits fallen durch das Raster?

**Aufgabe 6.** Praktisch: Richten Sie SSH-Signaturen ein, erzeugen Sie einen signierten und einen unsignierten annotierten Tag und schreiben Sie ein Gate-Skript, das nur den signierten akzeptiert. Verändern Sie danach das Tag-Objekt und beobachten Sie die Prüfung.

**Aufgabe 7.** Praktisch: Erzeugen Sie mit `git archive` aus einem Tag zweimal ein Archiv und weisen Sie mit Prüfsummen nach, dass beide gleich sind. Lesen Sie den Commit-Hash aus dem Archiv und verfolgen Sie ihn bis zum Tag zurück.

**Aufgabe 8.** Praktisch (Transfer): Schreiben Sie ein Versionsskript, das auf einem Tag die Release-Nummer und sonst eine Entwicklungsversion mit Commit-Hash ausgibt. Klonen Sie das Repository flach und zeigen Sie, dass das Skript scheitert; beheben Sie das Problem für eine CI-Umgebung.

## Lösungen zu Modul 07

**Lösung 1.** Ein Lightweight Tag ist nur ein Zeiger auf einen Commit. Ein annotierter Tag ist ein eigenes Objekt mit Datum, Tagger, Nachricht und optionaler Signatur. Für Releases nimmt man annotierte Tags, weil sie Urheber und Begründung festhalten, signierbar sind und von `git describe` standardmäßig berücksichtigt werden (laut Git-Dokumentation sind sie für Releases gedacht, Lightweight Tags für private Markierungen).

**Lösung 2.** (a) 2.3.2, (b) 2.4.0, (c) 3.0.0, denn das Entfernen einer öffentlichen Funktion ist nicht abwärtskompatibel. Beim Erhöhen einer höherwertigen Stelle werden die niedrigeren auf 0 gesetzt.

**Lösung 3.** Die einfache Versionssortierung kennt keine Vorabversionen und behandelt das Suffix wie weitere Zeichen, sodass die längere Zeichenfolge später kommt. Abhilfe: `git config --add versionsort.suffix -rc` (ebenso für `-alpha`, `-beta`). Dann stehen die Vorabversionen vor der zugehörigen Normalversion.

**Lösung 4.** Der Cherry-Pick erzeugt einen neuen Commit mit anderem Hash; der Tag auf dem Release-Branch enthält die neue Kopie, nicht das Original. Dokumentation: `git cherry-pick -x` (Herkunftszeile in der Nachricht), ein einheitlicher Hinweis im Ticket oder Pull Request und ein Eintrag im Changelog der Release-Linie. Mit `git cherry -v` vergleicht man Änderungen statt Hashes.

**Lösung 5.** Vorlage: Skript aus 7.3. Es bleiben alle Commits ohne erkannten Typ-Präfix hängen (im Beispiel „Merge-Ticket ohne Praefix“) sowie Typen, für die keine Gruppe definiert ist (`docs`, `chore`). Diese Lücke ist ein Argument für eine durchgesetzte Konvention.

**Lösung 6.** Ablauf wie in 7.4. Der signierte Tag besteht (Exit-Code 0), der unsignierte („no signature found“) und der Lightweight Tag („cannot verify a non-tag object“) nicht. Nach der Veränderung des Tag-Objekts meldet Git „incorrect signature“: Die Signatur gilt für den ursprünglichen Inhalt.

**Lösung 7.** Ablauf wie in 7.6. Beide Archive sollten dieselbe SHA-256-Prüfsumme haben (mit `gzip -n`). `zcat datei.tar.gz | git get-tar-commit-id` liefert den Commit; `git tag --points-at <hash>` nennt den Tag.

**Lösung 8.** Skript wie in 7.8. Im Klon mit `--depth=1` schlägt `git describe` fehl, weil die getaggten Commits nicht geladen sind. Behebung in der Pipeline: `git fetch --unshallow --tags` vor der Versionsberechnung, oder der Klon-Schritt der Pipeline wird auf volle Tiefe beziehungsweise passende Tiefe eingestellt.
