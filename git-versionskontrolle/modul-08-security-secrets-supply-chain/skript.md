# Modul 08 - Security, Secrets und Supply Chain

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (8.1 bis 8.9), danach folgen „Übungen zu Modul 08“ (8 Aufgaben) und „Lösungen zu Modul 08“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 12 h | **Priorität:** 1

> Lehrplan-Hinweis: Die Modulübersicht nennt 12 h, die Summe der Lerneinheiten ergibt 9 h. Dieses Skript folgt dem höheren Wert und behandelt die Lerneinheiten deshalb ausführlicher.

**Lernziele des Moduls:** Repositories gegen Secret-Leaks, Manipulation und Berechtigungsrisiken härten; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Git wurde als Werkzeug für Zusammenarbeit entworfen, nicht als Tresor. Alles, was einmal committet wurde, bleibt in der Historie und wird mit jedem Klon vervielfältigt. Dieses Modul zeigt, wie man verhindert, dass Geheimnisse hineingelangen, was man tut, wenn es doch passiert, wie man die Herkunft von Änderungen belegbar macht und wie man die Lieferkette um das Repository herum absichert.

| **Nr.** | **Lerneinheit**             | **Dauer (h)** | **Prio** |
|:--------|:----------------------------|--------------:|:--------:|
| 8.1     | Secret-Leaks vermeiden      |           1,0 |    1     |
| 8.2     | Secret Recovery             |           1,0 |    1     |
| 8.3     | Signierte Commits           |           1,0 |    1     |
| 8.4     | Zugriffskontrolle           |           1,0 |    1     |
| 8.5     | Branch- und Tag-Schutz      |           1,0 |    1     |
| 8.6     | Dependency und Supply Chain |           1,0 |    1     |
| 8.7     | Audit Logging               |           1,0 |    2     |
| 8.8     | Datenschutz im Repository   |           1,0 |    2     |
| 8.9     | Security Policies           |           1,0 |    2     |
|         | **Summe**                   |       **9,0** |          |

## 8.1 Secret-Leaks vermeiden

Ein *Secret* ist jede Information, die Zugriff gewährt und deshalb geheim bleiben muss: Passwörter, API-Schlüssel, Zugriffstokens, private Schlüssel von SSH oder TLS, Verbindungszeichenfolgen mit Passwort, Signierschlüssel. Dass solche Daten in Repositories landen, ist einer der häufigsten und zugleich vermeidbarsten Sicherheitsvorfälle. Der Grund liegt im Datenmodell aus Modul 01: Jeder Commit ist ein unveränderlicher Schnappschuss, und die Historie wird bei jedem Klon vollständig mitgeliefert. Wer ein Passwort committet und es im nächsten Commit wieder löscht, hat es damit nicht entfernt, sondern nur aus dem jeweils neuesten Stand genommen. Jeder, der das Repository je geklont hat, besitzt das Passwort weiterhin.

Die wirksamste Gegenmaßnahme ist deshalb nicht das Aufräumen, sondern das Vermeiden, und zwar in mehreren Schichten, die sich gegenseitig auffangen. Die erste Schicht ist eine *Architekturentscheidung*: Secrets gehören nicht in den Code, sondern werden zur Laufzeit geliefert, etwa über Umgebungsvariablen, einen Secret-Manager der Plattform oder eine Konfigurationsdatei außerhalb des Repositories. Im Repository liegt allenfalls eine Vorlage ohne echte Werte, üblicherweise `.env.example`. Die zweite Schicht ist die *Ignore-Regel*: Dateien wie `.env` stehen in der `.gitignore` (Modul 02), damit sie nicht versehentlich mit `git add .` aufgenommen werden. Die dritte Schicht ist ein *Scan vor dem Commit*, ein Hook, der die vorgemerkten Änderungen nach typischen Mustern absucht. Die vierte Schicht ist ein *serverseitiger Scan* beim Push, der sich nicht umgehen lässt, und die fünfte ein regelmäßiger Scan der gesamten Historie in der CI-Pipeline.

Warum braucht es mehrere Schichten? Clientseitige Hooks sind eine Komfortfunktion und kein Schutzmechanismus. Die Dokumentation zu den Git-Hooks (abgerufen am 28.10.2026) beschreibt, dass `pre-commit` und `commit-msg` mit `--no-verify` übersprungen werden können. Dazu kommt, dass Hooks im Verzeichnis `.git/hooks` liegen und deshalb nicht mit dem Repository versioniert werden (allgemeines Git-Wissen, nicht Wortlaut der Seite). Wer sie im Team nutzen will, legt sie in ein versioniertes Verzeichnis und setzt `git config core.hooksPath .githooks`; erzwingen kann man sie damit nicht. Verbindlich wird die Regel erst auf dem Server.

> **Ergänzung aus der Nachrecherche**
>
> **Push Protection bei GitHub.** Laut GitHub-Dokumentation (abgerufen am 28.10.2026) blockiert *Push Protection* Pushes, die erkannte Zugangsdaten enthalten, und zwar für Pushes per Kommandozeile, für Commits in der Weboberfläche, für Datei-Uploads und für Anfragen über die REST-API. Standardmäßig kann jede Person mit Schreibrecht die Sperre unter Angabe eines Grundes umgehen („wird in Tests verwendet“, „Fehlalarm“, „behebe ich später“); jeder Grund erzeugt einen eigenen Alarm, und die Umgehung wird im Audit-Log protokolliert. Mit *delegiertem Bypass* lässt sich festlegen, wer überhaupt umgehen darf. Das Feature setzt je nach Plan *GitHub Secret Protection* voraus und ist für Repositories standardmäßig nicht eingeschaltet. Andere Plattformen bieten vergleichbare Funktionen unter anderen Namen; prüfen Sie den Stand in Ihrer Plattform.

Das folgende Beispiel baut die dritte Schicht nach: einen kleinen Pre-commit-Hook, der die vorgemerkten Zeilen mit einem regulären Ausdruck prüft. Er sucht nach dem Muster von AWS-Zugangsschlüsseln (`AKIA` gefolgt von 16 Zeichen), nach Kopfzeilen privater Schlüssel und nach Zuweisungen wie `token = ...` mit einem langen Wert. Der Hook liegt im versionierten Verzeichnis `.githooks` und wird über `core.hooksPath` aktiviert. Der Wert `AKIAIOSFODNN7EXAMPLE` ist ein bekannter Beispielschlüssel aus Dokumentationen und kein echter Zugang.

```bash
git init -q -b main sec && cd sec
git config user.name Dev; git config user.email dev@example.org
mkdir .githooks
cat > .githooks/pre-commit <<'EOF'
#!/bin/sh
PATTERN='(AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY-----|(password|passwd|secret|token|api[_-]?key)[[:space:]]*[=:][[:space:]]*.?[A-Za-z0-9/+_-]{12,})'
FUNDE=$(git diff --cached -U0 --no-color | grep '^+' | grep -v '^+++' | grep -Ei "$PATTERN")
if [ -n "$FUNDE" ]; then
  echo "pre-commit: moegliches Secret in den gestagten Aenderungen:" >&2
  echo "$FUNDE" | sed 's/^/  /' >&2
  exit 1
fi
EOF
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks

echo 'DB_HOST=localhost' > .env.example
git add . && git commit -q -m "Beispielkonfiguration"

echo 'api_key = "AKIAIOSFODNN7EXAMPLE"' > config.py
git add config.py
git commit -m "Zugang eingetragen"; echo "Exit-Code: $?"
git restore --staged config.py
echo 'api_key = os.environ["API_KEY"]' > config.py
git add config.py && git commit -q -m "Zugang aus Umgebung lesen"; echo "Exit-Code: $?"

echo 'TOKEN=abcdef0123456789abcd' > .env
git add .env && git commit -q -m "env"; echo "Exit-Code: $?"
git commit -q --no-verify -m "env mit --no-verify"; echo "Exit-Code: $?"
git log --oneline
echo ".env" >> .gitignore
git rm -q --cached .env
git add .gitignore && git commit -q -m ".env ignorieren"
git ls-files
```

Der erste Commit mit `.env.example` geht durch, weil `DB_HOST=localhost` weder zum Schlüsselmuster noch zu einer langen Zuweisung passt. Der Versuch, `config.py` mit dem AWS-Beispielschlüssel zu committen, scheitert: Der Hook gibt die verdächtige Zeile aus und beendet sich mit Fehlercode 1, wodurch Git den Commit abbricht. Mit `git restore --staged` nimmt man die Datei aus der Staging Area, ersetzt den festen Wert durch den Zugriff auf eine Umgebungsvariable und committet erfolgreich; das ist die eigentliche Lösung des Problems. Die folgenden Zeilen zeigen die Grenze des Verfahrens: Der Hook blockiert zunächst auch die Datei `.env`, doch mit `--no-verify` lässt er sich umgehen. Das Secret steckt damit in der Historie. Die letzten Befehle entfernen die Datei aus dem Index (`git rm --cached`) und tragen sie in die `.gitignore` ein, aber der Commit „env mit –no-verify“ bleibt in der Historie; wie man damit umgeht, ist Thema der nächsten Lerneinheit.

### Typische Fehler und Stolperfallen.

Der häufigste Fehler ist, ein versehentlich committetes Secret im nächsten Commit einfach zu löschen und die Sache damit für erledigt zu halten. Richtig ist, es als kompromittiert zu behandeln (Abschnitt 8.2). Der zweite klassische Fehler ist der Glaube, die `.gitignore` würde bereits versionierte Dateien aus dem Repository entfernen. Sie wirkt nur für Dateien, die noch nicht verfolgt werden; eine schon committete Datei bleibt verfolgt, bis man sie mit `git rm --cached` aus dem Index nimmt. Drittens verlässt man sich allein auf einen lokalen Hook. Falsch ist: „Wir haben einen Pre-commit-Scan, also sind wir sicher.“ Richtig ist: Der lokale Scan gibt schnelles Feedback, der serverseitige Scan oder die Pipeline setzt die Regel durch. Viertens: Zu aggressive Muster erzeugen so viele Fehlalarme, dass Teams den Scan abschalten; ein Scan braucht eine Möglichkeit, bekannte Ausnahmen zu dokumentieren.

> **Merke**
>
> Secrets gehören in einen Secret-Manager oder in die Umgebung, nie in den Code. Mehrere Schichten (Ignore-Regel, Pre-commit-Scan, serverseitiger Scan, CI-Scan der Historie) fangen Fehler auf; verbindlich ist nur, was der Server durchsetzt.

### Praxisbezug.

In echten Projekten setzt man selten selbst gebaute Muster ein, sondern spezialisierte Scanner wie gitleaks oder die eingebauten Funktionen der Plattform (weiterführend, nicht abgerufen). Das Prinzip bleibt dasselbe wie im Beispiel: Ein Hook im Repository für schnelle Rückmeldung, eine Pipeline-Stufe für die Prüfung der Historie und ein serverseitiger Schutz für den verbindlichen Teil. Dazu gehört ein Eintrag im Onboarding: Neue Teammitglieder erfahren am ersten Tag, wo Secrets liegen und wie man sie bezieht.

## 8.2 Secret Recovery

Ist ein Secret in ein Repository gelangt, gilt eine einfache Reihenfolge: zuerst *rotieren*, dann *bewerten*, dann erst *aufräumen*. Die Dokumentation von GitHub zum Entfernen sensibler Daten (abgerufen am 28.10.2026) nennt als ersten Schritt genau das: Wenn es sich um ein Secret handelt, muss es widerrufen oder rotiert werden. Danach kann es nicht mehr für Zugriffe genutzt werden, und das kann nach Aussage der Seite bereits ausreichen, sodass ein Umschreiben der Historie nicht zwingend nötig ist. Rotieren bedeutet, ein neues Secret auszustellen, alle berechtigten Verbraucher darauf umzustellen und das alte ungültig zu machen. Nur dieser Schritt beendet die Gefahr tatsächlich, denn man kann nicht ausschließen, dass das Secret bereits kopiert wurde. Das Umschreiben der Historie räumt auf, macht ein bereits kompromittiertes Secret aber nicht wieder sicher.

Der zweite Schritt ist die *Bewertung*: Seit wann steckt das Secret im Repository, wer hatte Zugriff (nur ein privates Repository oder öffentlich einsehbar?), wurde es nachweislich genutzt? Viele Dienste protokollieren Zugriffe pro Schlüssel; die Auswertung dieser Protokolle zeigt, ob es einen Missbrauch gab. Mit `git log -S"geheimer-wert" --all` findet man die Commits, die den Wert eingeführt oder entfernt haben, und mit `git grep` über alle Revisionen sieht man, in welchen Ständen er noch vorkommt. Dieses Ergebnis gehört in die Dokumentation des Vorfalls.

Erst der dritte Schritt ist die *Bereinigung der Historie*. Das empfohlene Werkzeug ist `git filter-repo`; die Projektseite beschreibt es als Ersatz für das langsame und fehleranfällige `git filter-branch` (abgerufen am 28.10.2026). Es ist darauf ausgelegt, in einem frischen Klon zu laufen und bricht sonst ab, sofern man es nicht mit `--force` erzwingt. Mit `--replace-text` ersetzt es Texte in allen Commits anhand einer Regelliste. Weil dabei alle betroffenen Commits neue Hashes bekommen, ist das Ergebnis eine neue Historie, die man per Force-Push veröffentlichen müsste, und alle Beteiligten müssen anschließend neu klonen oder ihre Arbeit sorgfältig umsetzen.

Das Beispiel spielt den Vorgang durch: ein Repository, in dem ein Passwort committet und später nur aus dem aktuellen Stand entfernt wurde, die Suche in der Historie und die Bereinigung in einem frischen Klon.

```bash
git init -q -b main leak && cd leak
git config user.name Dev; git config user.email dev@example.org
echo 'print("hallo")' > app.py && git add . && git commit -q -m "App"
echo 'DB_PASSWORD=hunter2-geheim-42' > .env && git add .env && git commit -q -m "Konfiguration"
echo '# weiter' >> app.py && git commit -q -am "Weiterentwicklung"
git rm -q --cached .env && echo ".env" > .gitignore && git add .gitignore && git commit -q -m "Secret aus dem Stand entfernt"
cat .env

echo "--- Suche in der Historie"
git log --oneline -S"hunter2-geheim" --all
git grep -c "hunter2" $(git rev-list --all) -- .env

echo "--- Bereinigung im frischen Klon"
cd .. && git clone -q --no-local leak leak-clean && cd leak-clean
printf 'hunter2-geheim-42==>***ENTFERNT***\n' > ../ersetzen.txt
git filter-repo --replace-text ../ersetzen.txt --force 2>&1 | tail -3
git log --oneline
git log --all -p | grep -c "hunter2"
git show HEAD~2:.env
git remote -v | wc -l

echo "--- Der urspruengliche Klon enthaelt das Secret weiter"
git -C ../leak log --oneline -S"hunter2-geheim" --all
```

Im Ausgangsrepository steht das Passwort noch in der Datei `.env` im Arbeitsverzeichnis und, was entscheidend ist, in den beiden Commits der Historie. Die Suche mit `-S` liefert zwei Treffer: den Commit, der das Passwort eingeführt hat, und den, der es entfernt hat. Die Schleife über `git rev-list --all` zeigt, in welchen Revisionen die Datei `.env` das Wort noch enthält. Im frischen Klon liest `git filter-repo` die Ersetzungsregel aus der Datei `ersetzen.txt`; die Schreibweise `alt==>neu` ersetzt den alten Text durch den neuen. Danach hat die Historie die gleiche Form, aber neue Hashes, und der Zähler für das Wort „hunter2“ in allen Diffs steht auf null. Die alte Revision zeigt nur noch den Platzhalter. Zwei Beobachtungen sind wichtig: Das Werkzeug entfernt die Verbindung zum `origin` (die Anzahl der Zeilen von `git remote -v` ist null), damit man nicht versehentlich ohne Überlegung pusht, und der ursprüngliche Klon enthält das Passwort unverändert weiter, so wie jeder andere bereits existierende Klon.

> **Ergänzung aus der Nachrecherche**
>
> **Was nach dem Umschreiben bleibt.** Die GitHub-Dokumentation (abgerufen am 28.10.2026) weist darauf hin, dass sich sensible Daten nicht aus fremden Klonen entfernen lassen, dass Forks sie weiter enthalten und dass sie über zwischengespeicherte Ansichten und Pull Requests, die auf die alten Commits verweisen, erreichbar bleiben können. Beim Force-Push der bereinigten Historie scheitern laut Seite Referenzen unter `refs/pull/`, weil GitHub sie als schreibgeschützt behandelt; für das endgültige Entfernen bietet GitHub den Support an, aber nur in Fällen, in denen sich das Risiko nicht durch Rotieren der Zugangsdaten beheben lässt. Ein Merge-Commit eines Kollegen mit der alten Historie kann die Daten wieder einschleppen; die Seite empfiehlt deshalb, dass Kollegen ihre Arbeit auf die neue Historie *rebasen* und nicht mergen. Bei anderen Plattformen gelten ähnliche Grenzen.

Aus diesen Gründen definieren reife Teams einen *Incident-Prozess für Leaks*, bevor der erste Fall eintritt. Er legt fest, wer den Vorfall meldet und wer ihn führt, wer rotieren darf, wie schnell das passieren muss, wie die Bewertung dokumentiert wird und wann Kunden oder Behörden zu informieren sind (je nach Branche und Inhalt der Daten gibt es gesetzliche Meldepflichten; das klären Sie mit Ihrer Rechts- und Compliance-Stelle, dieses Skript ist keine Rechtsberatung).

### Typische Fehler und Stolperfallen.

Falsch ist, zuerst die Historie umzuschreiben und das Rotieren auf später zu verschieben. Die Zeit zwischen Leak und Rotation ist die gefährliche, und Bots durchsuchen öffentliche Repositories laufend nach Zugangsdaten; das ist die gängige Annahme in der Branche, auch wenn sie sich im Einzelfall nicht belegen lässt. Richtig ist: erst rotieren, dann aufräumen. Der zweite Fehler ist, `git filter-repo` im produktiv genutzten Klon laufen zu lassen. Richtig ist ein frischer Klon, den man bei Problemen verwerfen kann. Drittens wird häufig vergessen, die Kolleginnen und Kollegen zu informieren: Wer die alte Historie lokal hat und sie wieder pusht, bringt das Secret zurück. Viertens: Ein Secret, das nur im Commit-Text, in einem Issue oder in einem Pull-Request-Kommentar auftauchte, wird beim Umschreiben der Dateien nicht erfasst, und es muss dort separat entfernt werden.

### Praxisbezug.

Im Alltag begegnet einem das Thema als Alarm der Plattform („Secret scanning alert“) oder als Meldung der Pipeline. Der Ablauf ist immer derselbe: Alarm bestätigen, Secret rotieren, Zugriffsprotokolle prüfen, bei Bedarf die Historie bereinigen, Vorfall dokumentieren und die Ursache beseitigen, etwa indem der Pre-commit-Scan nachgeschärft wird. Üben Sie diesen Ablauf in einem Testrepository, bevor er ernst wird.

## 8.3 Signierte Commits

Ein Commit enthält den Namen und die E-Mail-Adresse des Autors, aber Git prüft diese Angaben nicht. Mit `git config user.name` und `user.email` kann jeder eintragen, was er will. Es ist also technisch trivial, einen Commit unter dem Namen einer anderen Person zu erstellen. Für die Nachvollziehbarkeit in sicherheitskritischen Umgebungen reicht das nicht aus. Hier setzen *signierte Commits* an: Der Commit (oder Tag) wird mit einem privaten Schlüssel kryptografisch signiert, und jeder mit dem passenden öffentlichen Schlüssel kann prüfen, dass der Inhalt seither nicht verändert wurde und dass der Besitzer des Schlüssels ihn signiert hat. Die GitHub-Dokumentation beschreibt, dass sich damit „Vertrauen in die Herkunft“ einer Änderung herstellen lässt (abgerufen am 28.10.2026).

Git unterstützt drei Formate. *OpenPGP* (GPG) ist das klassische Verfahren mit eigenem Schlüsselbund, Vertrauensmodell und Ablauf. *SSH-Signaturen* nutzen die SSH-Schlüssel, die Entwicklerinnen und Entwickler ohnehin besitzen. *S/MIME* nutzt X.509-Zertifikate und kommt vor allem in größeren Organisationen vor, die eine eigene Zertifikatsinfrastruktur haben. Die Git-Konfiguration `gpg.format` wählt das Format; erlaubt sind laut Git-Dokumentation `openpgp` (Standard), `x509` und `ssh`. Nach der GitLab-Dokumentation (abgerufen am 28.10.2026) verlangt SSH-Signing Git 2.34 oder neuer und OpenSSH 8.1 oder neuer; die Version 8.7 sei fehlerhaft, empfohlen wird 8.8 oder neuer. Aus Sicht der Einfachheit ist SSH meist die beste Wahl, weil keine zweite Schlüsselverwaltung nötig ist.

Zum Prüfen braucht Git eine Zuordnung von Personen zu öffentlichen Schlüsseln. Bei SSH ist das die Datei *allowed signers*, auf die `gpg.ssh.allowedSignersFile` verweist. Jede Zeile enthält eine Identität (E-Mail-Adresse) und den öffentlichen Schlüssel. Die Git-Dokumentation erklärt, dass es bei SSH keine Vertrauensstufen gibt: Ein Schlüssel in der Datei gilt als voll vertrauenswürdig, ein Schlüssel außerhalb der Datei als undefiniert, und `git verify-commit` scheitert dann (abgerufen am 28.10.2026). Das Beispiel richtet alles ein und probiert aus, wie sich eine Fremdsignatur verhält.

```bash
mkdir -p ~/.ssh ~/.config/git
ssh-keygen -q -t ed25519 -N "" -C "anna@example.org" -f ~/.ssh/id_signing
echo "anna@example.org namespaces=\"git\" $(cut -d' ' -f1,2 ~/.ssh/id_signing.pub)" > ~/.config/git/allowed_signers

git init -q -b main sign && cd sign
git config user.name "Anna"; git config user.email anna@example.org
git config gpg.format ssh
git config user.signingkey ~/.ssh/id_signing.pub
git config gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers

echo a > a.txt && git add . && git commit -q -S -m "signierter Commit"
echo b > b.txt && git add . && git commit -q -m "unsignierter Commit"
git log --format='%h %G? %GS | %s'
git verify-commit HEAD~1; echo "Exit: $?"
git verify-commit HEAD; echo "Exit: $?"

git config commit.gpgsign true
git config tag.gpgSign true
echo c > c.txt && git add . && git commit -q -m "automatisch signiert"
git tag -m "Release 1.0" v1.0
git tag -v v1.0; echo "Exit: $?"

# Signatur mit einem fremden, nicht eingetragenen Schluessel
ssh-keygen -q -t ed25519 -N "" -C "mallory@example.org" -f ~/.ssh/id_fremd
git -c user.signingkey=$HOME/.ssh/id_fremd.pub commit -q --allow-empty -m "Fremdsignatur"
git log -1 --format='%h %G? %s'
git verify-commit HEAD; echo "Exit: $?"
```

Zuerst entsteht ein Schlüsselpaar nur für das Signieren; die Datei `allowed_signers` ordnet ihm die E-Mail-Adresse zu und beschränkt ihn auf den Namensraum `git`. Die Repository-Konfiguration schaltet das Format auf SSH, nennt den öffentlichen Schlüssel und verweist auf die Datei der erlaubten Signierer. Der erste Commit trägt `-S` und wird signiert, der zweite nicht. Die Log-Ausgabe zeigt in der Spalte `%G?` den Status: `G` für eine gültige Signatur, `N` für keine; in der Ausgabe daneben steht der Principal, dem der Schlüssel gehört. `git verify-commit` endet bei einer gültigen Signatur mit Exit-Code 0 und meldet „Good "git" signature for anna@example.org“, bei einem unsignierten Commit mit 1. Nach dem Setzen von `commit.gpgsign` und `tag.gpgSign` signiert Git künftig jeden Commit und jeden annotierten Tag automatisch; `git tag -v` prüft die Signatur des Tags. Zuletzt signiert ein anderer Schlüssel, der nicht in der Datei steht: Der Status ist `U` (die Signatur ist kryptografisch gültig, aber der Schlüssel ist keiner Person zugeordnet), und `git verify-commit` meldet „No principal matched“ mit Exit-Code 1. Genau das ist der Unterschied zwischen *gültiger* und *vertrauenswürdiger* Signatur: Eine mathematisch korrekte Signatur beweist nur, dass jemand den Schlüssel besaß, nicht, dass es die gemeinte Person war.

> **Ergänzung aus der Nachrecherche**
>
> **Plattformen und Verified-Status.** GitHub zeigt laut Dokumentation (abgerufen am 28.10.2026) *Verified*, wenn die Signatur kryptografisch gültig ist, *Unverified*, wenn der Commit signiert ist, die Signatur aber nicht geprüft werden konnte, und keinen Status bei unsignierten Commits. Commits, die über die Weboberfläche erstellt werden, signiert GitHub selbst. Der optionale *Vigilant Mode* (standardmäßig aus) kennzeichnet auch unsignierte Commits der eigenen Konten als *Unverified* und führt *Partially verified* ein, wenn ein Autor, der nicht der Committer ist, den Modus aktiviert hat. GitLab prüft SSH-Signaturen gegen die Schlüssel am Benutzerkonto; wird ein Schlüssel widerrufen, gelten die Commits als nicht verifiziert (Dokumentation abgerufen am 28.10.2026). Die Plattformen kennen den Schlüssel also aus dem Konto und prüfen nicht gegen Ihre lokale Datei.

### Typische Fehler und Stolperfallen.

Falsch ist, die „Verified“-Anzeige mit Code-Qualität oder gar Code-Sicherheit zu verwechseln. Eine Signatur sagt nur, *wer* committet hat und dass nichts verändert wurde, nicht, ob die Änderung gut ist. Zweiter Fehler: Signierte Commits einführen, aber nicht prüfen. Eine Signatur, die niemand verifiziert, schützt nicht; die Prüfung gehört in die Branch-Regeln der Plattform oder in die Pipeline (Modul 05, Branch Protection). Dritter Fehler: Den privaten Schlüssel ungeschützt auf mehreren Geräten kopieren oder ohne Passphrase ablegen; ein gestohlener Signierschlüssel macht Fälschungen unauffällig möglich. Vierter Fehler: Signaturen und Rebase werden nicht zusammen gedacht. Ein Rebase erzeugt neue Commits, und die alte Signatur geht dabei verloren (im Test ohne `commit.gpgsign` fehlte sie danach). Ist die Option gesetzt, signiert Git die neuen Commits mit dem Schlüssel der Person, die den Rebase ausführt; die Signatur belegt dann nicht mehr die ursprüngliche Autorenschaft, sondern die Person, die umgeschrieben hat.

### Praxisbezug.

Signaturen spielen vor allem bei Release-Tags und bei Repositories mit hohen Integritätsanforderungen eine Rolle: Open-Source-Projekte signieren Releases, damit Nutzer prüfen können, dass der Tag vom Maintainer stammt; Unternehmen fordern signierte Commits auf geschützten Branches im Rahmen ihrer Compliance-Vorgaben. Auch die Anbindung an Hardware-Token oder an eine zentrale Schlüsselverwaltung ist üblich. In Modul 07 kommt das Signieren von Release-Tags ausführlich vor.

## 8.4 Zugriffskontrolle

Zugriffskontrolle in Git-Umgebungen lässt sich auf einen Grundsatz reduzieren: *Least Privilege*, jede Person und jedes System bekommt genau die Rechte, die für die Aufgabe nötig sind, und nicht mehr. Dahinter stehen mehrere Ebenen. Auf der Ebene der *Personen* geht es um Rollen und Teams (Modul 04): Lesen für alle, die den Code brauchen, Schreiben für das Team, das ihn pflegt, Administration für wenige, namentlich bekannte Personen. Auf der Ebene der *Maschinen* geht es um Zugangstokens, Deploy Keys und Service-Konten. Hier ist die Versuchung groß, ein persönliches Zugriffstoken mit weitreichenden Rechten in die Pipeline zu hängen, weil es schnell funktioniert. Das Ergebnis ist ein langlebiger, mächtiger Schlüssel, der im Zweifel an eine Person gebunden ist, die das Unternehmen irgendwann verlässt.

Gute Praxis für Tokens ist eine kurze Liste. Tokens sollen die *kleinste nötige Reichweite* haben (nur ein Repository statt aller, nur Lesen statt Schreiben), ein *Ablaufdatum* besitzen und einem *Service-Konto* oder einer App gehören statt einer Privatperson. Für reine Lesezugriffe von Servern bieten sich Deploy Keys an, die an ein einzelnes Repository gebunden sind. Wo die Plattform es unterstützt, ersetzen kurzlebige, automatisch ausgestellte Tokens (etwa der in jeder Pipeline-Ausführung erzeugte Token) dauerhafte Geheimnisse ganz. Welche konkreten Token-Typen Ihre Plattform anbietet und wie ihre Rechte heißen, unterscheidet sich je nach Anbieter und ändert sich; schlagen Sie es in der aktuellen Dokumentation nach (hier nicht abgerufen). Für die Administration gilt: möglichst wenige Admin-Konten, getrennt vom Alltagskonto, mit Mehrfaktor-Authentifizierung, und regelmäßige Überprüfung, wer sie noch braucht.

Ein Punkt, den man ohne Plattform prüfen kann, ist die Hygiene der lokalen Konfiguration. Besonders häufig findet man Zugangsdaten direkt in der Remote-URL, also `https://benutzer:token@host/pfad.git`. Das ist bequem, hat aber Nachteile: Der Token steht im Klartext in `.git/config`, taucht in Prozesslisten, Logdateien und Fehlermeldungen auf und wandert bei Backups des Verzeichnisses mit. Das folgende Skript prüft alle Remotes eines Repositories und meldet solche URLs, ohne die Zugangsdaten selbst auszugeben.

```bash
git init -q -b main acc && cd acc
git remote add origin https://anna:ghp_ABCDEFGHIJ1234567890@git.example.org/team/app.git
git remote add upstream https://git.example.org/team/app.git
cat > audit-remotes.sh <<'EOF'
#!/bin/sh
# Meldet Remotes, deren URL Zugangsdaten enthaelt (ohne sie auszugeben)
status=0
for r in $(git remote); do
  url=$(git remote get-url "$r")
  case "$url" in
    https://*@*|http://*@*)
      echo "WARNUNG: Remote '$r' enthaelt Zugangsdaten in der URL"
      status=1 ;;
    *)
      echo "ok: $r" ;;
  esac
done
exit $status
EOF
sh audit-remotes.sh; echo "Exit-Code: $?"

# Behebung: URL bereinigen, Zugangsdaten ueber einen Credential Helper beziehen
git remote set-url origin https://git.example.org/team/app.git
git config credential.helper "cache --timeout=3600"
sh audit-remotes.sh; echo "Exit-Code: $?"
git config --get-regexp '^(remote\..*\.url|credential\.helper)$'
```

Das Skript durchläuft alle Remotes und prüft mit einem `case`-Muster, ob in einer HTTP(S)-URL vor dem ersten Schrägstrich ein `@` steht, was auf eingebettete Zugangsdaten hindeutet. Es gibt nur den Namen des Remotes aus, nicht die URL, damit der Token nicht in einem Pipeline-Log landet. Der Exit-Code 1 lässt sich in einer CI-Prüfung nutzen. Nach der Bereinigung mit `git remote set-url` bezieht Git die Zugangsdaten über einen *Credential Helper*. Hier steht `cache` stellvertretend für die Mechanismen der Betriebssysteme (unter Windows üblicherweise der Credential Manager); wichtig ist, dass die Zugangsdaten außerhalb der Repository-Konfiguration liegen. Die letzte Zeile zeigt, dass nur noch die saubere URL und die Helper-Einstellung im Repository stehen.

### Typische Fehler und Stolperfallen.

Falsch ist ein einzelner, mächtiger Token für alle Pipelines und Menschen, „damit es einfach ist“. Richtig ist je ein eigener Token mit minimalen Rechten und Ablauf. Falsch ist es ebenfalls, Admin-Rechte großzügig zu vergeben, weil jemand „nur kurz“ etwas einstellen muss. Admin-Rechte werden zeitlich begrenzt vergeben und danach entzogen. Dritter Fehler: Ausgeschiedene Personen und nicht mehr genutzte Tokens bleiben aktiv; es braucht einen festen Prozess und regelmäßige Überprüfungen der Mitgliederlisten, Deploy Keys und Tokens. Vierter Fehler: Tokens in Pipeline-Skripten per `echo` ausgeben oder mit `set -x` mitprotokollieren, sodass sie im Log stehen.

### Praxisbezug.

Praktisch führt man einmal pro Quartal einen *Access Review* durch: Wer hat Zugriff auf welche Repositories, wer ist Admin, welche Tokens und Deploy Keys existieren, wann laufen sie ab? Viele Plattformen bieten dafür Übersichten. Das Ergebnis dokumentiert man, denn bei Audits wird genau danach gefragt (Abschnitt 8.7).

## 8.5 Branch- und Tag-Schutz

Kritische Referenzen, also die Hauptbranches und die Release-Tags, müssen davor geschützt sein, versehentlich oder absichtlich gelöscht, überschrieben oder ohne Freigabe verändert zu werden. Das Prinzip kennen Sie aus Modul 05 (Branch Protection auf Plattformebene, Regeln für Pflichtprüfungen und Reviews). Dieser Abschnitt betrachtet die Mechanismen darunter, denn sie zeigen, *was* eigentlich geschützt wird: Auf Git-Ebene ist ein Branch ein Zeiger, und ein Push ist der Versuch, einen Zeiger auf dem Server zu verschieben. Es gibt genau drei gefährliche Operationen: einen Zeiger auf einen Commit setzen, der den bisherigen nicht als Vorfahren hat (Force-Push, die Historie wird überschrieben), einen Zeiger löschen und einen Zeiger anlegen oder verschieben, ohne dass jemand zugestimmt hat.

Ein Git-Server (auch ein einfacher, selbst betriebener) kennt dafür Konfigurationsschalter. `receive.denyNonFastForwards` verbietet Pushes, die keine Fast-Forward-Updates sind, und `receive.denyDeletes` verbietet das Löschen von Referenzen. Und schließlich gibt es den `update`-Hook, der pro Referenz entscheiden kann (Modul 04, Modul 05). Gehostete Plattformen bringen diese Funktionen als Regeln mit; die GitHub-Dokumentation listet für Rulesets Regeltypen wie das Verhindern von Löschen und Force-Push (abgerufen am 28.10.2026, siehe Modul 05). Das folgende Experiment mit einem lokalen Bare-Repository zeigt, was genau die einzelnen Schalter abdecken und wo eine Lücke bleibt.

```bash
git init -q --bare -b main server.git
git -C server.git config receive.denyDeletes true
git -C server.git config receive.denyNonFastForwards true
git clone -q server.git work 2>/dev/null && cd work
git config user.name Dev; git config user.email dev@example.org
echo 1 > a.txt && git add . && git commit -q -m "Release-Stand"
git push -q origin main
git tag -a v1.0 -m "Release 1.0" && git push -q origin v1.0

echo "--- Force-Push auf main"
git commit -q --amend -m "Release-Stand (umgeschrieben)"
git push --force origin main; echo "Exit: $?"
echo "--- Branch loeschen"
git push origin --delete main; echo "Exit: $?"
echo "--- Tag loeschen"
git push origin --delete v1.0; echo "Exit: $?"
echo "--- Tag verschieben"
git tag -f -a v1.0 -m "anderer Stand"
git push --force origin v1.0; echo "Exit: $?"
```

Der Force-Push auf `main` wird mit der Meldung „denying non-fast-forward“ abgewiesen, und das Löschen des Branches mit „deletion prohibited“. Das zeigt, dass die beiden Schalter ihren Zweck für Branches erfüllen. Dann aber folgt die Überraschung: Das Löschen des Tags `v1.0` gelingt, und das Verschieben des Tags mit `--force` ebenfalls. Die Schalter schützen in dieser Konfiguration nur Branches; Tags werden von ihnen nicht abgedeckt (so verhielt sich Git 2.43 im Test). Für unveränderliche Release-Tags braucht es deshalb eine eigene Regel, entweder die Tag-Schutzfunktion der Plattform oder einen `update`-Hook auf dem Server. Das nächste Beispiel erweitert den Server um einen solchen Hook, der bestehende Tags schützt und neue Tags zulässt.

```bash
git init -q --bare -b main server.git
cat > server.git/hooks/update <<'EOF'
#!/bin/sh
# update-Hook: einmal angelegte Tags sind unveraenderlich
ref="$1"; old="$2"
case "$ref" in
  refs/tags/*)
    case "$old" in
      *[!0]*)
        echo "Tag $ref ist geschuetzt: Aendern und Loeschen sind nicht erlaubt." >&2
        exit 1 ;;
    esac ;;
esac
exit 0
EOF
chmod +x server.git/hooks/update
git clone -q server.git work 2>/dev/null && cd work
git config user.name Dev; git config user.email dev@example.org
echo 1 > a.txt && git add . && git commit -q -m "Release-Stand"
git push -q origin main
git tag -a v1.0 -m "Release 1.0" && git push -q origin v1.0
git tag -a v1.1 -m "Release 1.1" && git push -q origin v1.1; echo "neuer Tag: Exit $?"
git push origin --delete v1.0; echo "Tag loeschen: Exit $?"
echo 2 >> a.txt && git commit -q -am "Weiter"
git tag -f -a v1.0 -m "anderer Stand"
git push --force origin v1.0; echo "Tag verschieben: Exit $?"
git ls-remote --tags origin
```

Der Hook bekommt für jede Referenz den Namen, den alten und den neuen Hash. Bei einem neuen Tag besteht der alte Hash nur aus Nullen (die Prüfung `*[!0]*` fragt, ob ein Zeichen ungleich Null vorkommt, und funktioniert damit unabhängig von der Hashlänge), und der Hook lässt ihn zu. Jede Änderung oder Löschung eines bestehenden Tags beendet er mit Exit-Code 1 und der Meldung für die pushende Person, woraufhin Git die Referenz ablehnt. Die Ausgabe zeigt, dass der neue Tag `v1.1` angenommen wurde, während Löschen und Verschieben von `v1.0` scheitern.

### Typische Fehler und Stolperfallen.

Falsch ist die Annahme, „Branch-Schutz“ schütze automatisch auch Tags. Wie das Experiment zeigt, sind das getrennte Regeln, und Release-Tags brauchen eine eigene, auf der Plattform meist „Tag-Schutz“ oder eine Regel mit Muster wie `v*`. Zweiter Fehler: Schutzregeln nur für `main` anzulegen, aber nicht für `release/*`-Branches, auf denen Hotfixes entstehen. Dritter Fehler: Administratoren von den Regeln auszunehmen, „weil wir ihnen vertrauen“. Gerade deren Konten sind ein attraktives Ziel; sinnvoller ist, dass auch Admins die Regeln befolgen und Ausnahmen bewusst und protokolliert gemacht werden. Vierter Fehler: Regeln einzurichten und nie zu testen. Der Test im Beispiel (Force-Push versuchen, Tag löschen) gehört in jede Einrichtung.

### Praxisbezug.

Beim Einrichten eines neuen Repositories auf der Plattform legt man als Erstes die Schutzregeln für den Standardbranch, für `release/*` und für Release-Tags an und prüft sie per Test-Push. Wenn jemand nach einem Incident fragt, wer `main` überschrieben hat, ist die beste Antwort: „Das war technisch nicht möglich.“

## 8.6 Dependency und Supply Chain

Ein modernes Projekt besteht zum größten Teil aus fremdem Code: Bibliotheken, Frameworks, Build-Werkzeuge, Container-Images und die Aktionen oder Plugins der CI-Pipeline. Jede dieser Abhängigkeiten ist ein Angriffspunkt, denn wer sie verändert, verändert indirekt Ihr Produkt. Man spricht von der *Lieferkette* (Supply Chain). Im Zusammenhang mit Git gibt es zwei Stellen, an denen die Lieferkette direkt sichtbar wird: *Submodule*, bei denen ein fremdes Repository in das eigene eingebunden wird, und *externe Aktionen* in CI-Workflows, die per Namen und Version referenziert werden.

Beide Fälle haben dasselbe Grundproblem: Eine *Version* ist nur dann eine feste Zusage, wenn sie sich nicht ändern kann. Ein Git-Tag oder ein Branch-Name ist ein beweglicher Zeiger; der Eigentümer des fremden Repositories (oder jemand, der dessen Konto übernimmt) kann ihn auf neuen, manipulierten Code umsetzen. Ein vollständiger Commit-Hash dagegen ist ein Fingerabdruck des Inhalts und kann nicht umgelenkt werden. Deshalb lautet die Regel *Version Pinning*: Abhängigkeiten werden auf einen unveränderlichen Stand festgelegt, und Aktualisierungen erfolgen bewusst, reviewt und nachvollziehbar.

Submodule setzen dieses Prinzip von Haus aus um. Die Dokumentation (abgerufen am 28.10.2026) beschreibt, dass das übergeordnete Repository den Stand des Submoduls als Commit festhält und dass `git submodule update` den Submodul-Arbeitsbaum auf genau diesen Commit setzt. Ein Update auf den neuesten Upstream-Stand ist ein eigener, expliziter Schritt (`--remote`), und der neue Zeiger muss anschließend im übergeordneten Repository committet werden. Das Beispiel zeigt das.

```bash
export GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=protocol.file.allow GIT_CONFIG_VALUE_0=always
git init -q -b main libsrc && cd libsrc
git config user.name Dev; git config user.email dev@example.org
echo "v1" > lib.txt && git add . && git commit -q -m "lib v1"
cd ..
git init -q -b main app && cd app
git config user.name Dev; git config user.email dev@example.org
git submodule add -q ../libsrc vendor/lib
git commit -q -m "Bibliothek als Submodul eingebunden"
git submodule status
git ls-tree HEAD vendor/lib
cat .gitmodules

(cd ../libsrc && echo "v2" > lib.txt && git commit -q -am "lib v2")
echo "--- nach Aenderung im Upstream:"
git submodule status
git status -s
git submodule update --remote -q
echo "--- nach update --remote:"
git status -s
git diff --submodule=log
git add vendor/lib && git commit -q -m "lib bewusst auf v2 angehoben"
git submodule status
```

Die ersten Zeilen richten ein Bibliotheks-Repository `libsrc` und ein Anwendungs-Repository `app` ein. Die Umgebungsvariablen am Anfang erlauben den Dateiprotokoll-Zugriff nur für dieses lokale Experiment; Git sperrt ihn aus Sicherheitsgründen standardmäßig. `git submodule add` legt die Datei `.gitmodules` an (sie enthält Pfad und URL) und trägt im Verzeichnisbaum einen Eintrag vom Typ `160000 commit` ein. Diese Zeile aus `git ls-tree` ist der *Pin*: ein fester Commit-Hash. Ändert sich das Upstream-Repository, bleibt das Anwendungs-Repository unberührt, und `git status` zeigt keine Änderung. Erst `git submodule update --remote` holt den neuen Stand, danach meldet `git status` eine Änderung (`M vendor/lib`), und `git diff --submodule=log` zeigt die neuen Commits der Bibliothek. Dieser Zwang zum bewussten Commit ist der Sicherheitsgewinn: Jede Aktualisierung erscheint als Änderung im Pull Request und kann geprüft werden.

Für externe CI-Aktionen gilt dasselbe, nur ohne eingebaute Absicherung. Wer in einem Workflow `uses: example/lint-action@main` schreibt, führt bei jedem Lauf den aktuellen Stand des Branches aus. Auch ein Versions-Tag wie `@v4` kann nachträglich verschoben werden. Eine Prüfregel, die laut Datadog-Dokumentation Aktionen auf einen vollständigen, 40 Zeichen langen Commit-SHA festlegt, begründet das damit, dass der Code der Aktion dann unveränderlich sei und das Risiko von Manipulationen in der Lieferkette sowie von unerwartetem Verhalten durch Upstream-Updates sinke (abgerufen am 28.10.2026; Drittanbieter-Dokumentation einer Prüfregel, keine Vorgabe von GitHub). Dort steht auch der Hinweis, dass der SHA aus dem Original-Repository der Aktion stammen soll. Das folgende Skript prüft einen Workflow auf ungepinnte Aktionen.

```bash
mkdir -p .github/workflows
cat > .github/workflows/ci.yml <<'EOF'
name: ci
on: [push]
permissions:
  contents: read
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: example/deploy-action@3a1f5c9e8b7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b
      - uses: example/lint-action@main
      - uses: ./.github/actions/lokal
EOF
cat > pin-check.sh <<'EOF'
#!/bin/sh
# Meldet "uses:"-Zeilen, die nicht auf einen vollstaendigen Commit-SHA zeigen
grep -rEn '^[[:space:]-]*uses:' .github/workflows | while IFS= read -r zeile; do
  case "$zeile" in
    *"uses: ./"*) continue ;;
  esac
  if ! echo "$zeile" | grep -Eq '@[0-9a-f]{40}([[:space:]]|$)'; then
    echo "nicht gepinnt: $zeile"
  fi
done
EOF
sh pin-check.sh
```

Der Workflow enthält vier Aktionen. Die lokale Aktion (`./...`) liegt im selben Repository und wird vom Skript übersprungen. Die Aktion `example/deploy-action` ist auf einen 40-stelligen Hash festgelegt und gilt als gepinnt. `actions/checkout@v4` und `example/lint-action@main` zeigen dagegen auf bewegliche Zeiger und werden gemeldet; letzterer, ein Branch, ist der riskanteste Fall. Der Block `permissions: contents: read` im Workflow begrenzt die Rechte des automatisch erzeugten Tokens auf Lesen. Das ist dasselbe Least-Privilege-Prinzip wie in Abschnitt 8.4; laut GitHub-Dokumentation zur Workflow-Syntax (abgerufen am 28.10.2026) sind Bereiche, die man nicht nennt, sobald man irgendeinen Bereich festlegt, auf `none` gesetzt.

> **Ergänzung aus der Nachrecherche**
>
> **Pinning ist nicht alles.** Ein festgelegter Hash verhindert heimliche Änderungen, aber auch automatische Sicherheitsupdates. Deshalb kombiniert man Pinning mit einem Werkzeug, das Aktualisierungen als Pull Request vorschlägt (zum Beispiel Dependabot oder Renovate, weiterführend, nicht abgerufen). So wird jede Änderung der Abhängigkeit zu einer sichtbaren, prüfbaren Änderung im Repository. Dasselbe gilt für Paketmanager: Lock-Dateien (`package-lock.json`, `poetry.lock` und ähnliche) halten die genauen Versionen und Prüfsummen fest und gehören ins Repository. Die Prüfung vertrauenswürdiger Quellen ersetzt das Pinning nicht: Es nützt wenig, einen Stand festzunageln, den man nie angesehen hat.

### Typische Fehler und Stolperfallen.

Falsch ist, bei Submodulen auf der Version „main“ zu bleiben, indem man ständig `--remote` ausführt und den neuen Zeiger ungeprüft committet. Richtig ist, Updates als eigene, benannte Änderungen mit Blick auf die Differenz (`--submodule=log`) zu reviewen. Falsch ist auch, Aktionen oder Skripte mit `curl | sh` aus dem Netz zu laden: Was heute ein harmloses Installationsskript ist, kann morgen etwas anderes sein, und ein Hash-Vergleich fehlt. Dritter Fehler: Den Quell-Repositories der Abhängigkeiten blind zu vertrauen; Namensähnlichkeiten (Typosquatting) und übernommene Konten sind bekannte Angriffswege. Vierter Fehler: Zugriffstokens in Submodul-URLs. Ein Submodul mit eingebetteter Zugangskennung landet in `.gitmodules` im Klartext im Repository (Abschnitt 8.4).

### Praxisbezug.

Bei Sicherheits-Audits steht heute regelmäßig die Frage, wie die Lieferkette abgesichert ist. Die Antworten sind überschaubar: Abhängigkeiten sind gepinnt, Updates laufen über Pull Requests, Lock-Dateien sind eingecheckt, Pipelines haben minimale Rechte, und es gibt eine Liste der eingesetzten Komponenten. Das lässt sich mit den Werkzeugen dieses Abschnitts bereits weitgehend belegen.

## 8.7 Audit Logging

Ein *Audit-Log* beantwortet die Fragen „Wer hat wann was getan?“ auf eine Weise, die sich nicht nachträglich ändern lässt. Für Git ist das überraschend schwierig, denn Git selbst kennt Autor und Zeitpunkt nur als Angaben im Commit, und diese Angaben sind, wie in Abschnitt 8.3 gezeigt, frei wählbar. Hinzu kommt, dass die *Historie veränderbar* ist: Wer Force-Pushes durchführen darf, kann die Spuren seiner Änderungen verwischen. Das Reflog (Modul 06) ist lokal und taugt nicht als Beweismittel. Was man für ein belastbares Audit braucht, entsteht deshalb *auf dem Server*.

Drei Informationsquellen sind zu unterscheiden. Die erste ist die *Commit-Historie* selbst: Sie zeigt Inhalte, Autoren, Committer und, wenn Commits signiert sind, auch die verifizierbare Herkunft. Die zweite sind die *Server-Ereignisse*: wer hat welche Referenz wann von welchem Stand auf welchen gesetzt (Push, Force-Push, Löschung), wer hat Rechte geändert, wer hat sich angemeldet, wer hat einen Schutz umgangen. Die dritte sind die *Plattform-Aktionen*: Pull Requests, Freigaben, Merge-Entscheidungen, Änderungen der Einstellungen. Gehostete Plattformen führen dafür ein Audit-Log; laut GitHub-Dokumentation zur Push Protection werden etwa Umgehungen der Sperre dort protokolliert (abgerufen am 28.10.2026). Wie lange Einträge aufbewahrt werden, ob sie sich exportieren lassen und in welches Überwachungssystem sie gespeist werden können, hängt von Anbieter und Vertrag ab und ist vor einem Audit zu klären.

Bei einem selbst betriebenen Server kann man die zweite Quelle mit einem `post-receive`-Hook selbst erzeugen. Der Hook bekommt für jede aktualisierte Referenz eine Zeile mit altem Hash, neuem Hash und Referenzname und kann sie zusammen mit Zeitstempel und Benutzer in eine Protokolldatei schreiben. Das Beispiel richtet das ein, erzeugt einige Ereignisse (darunter einen Force-Push und das Löschen eines Branches) und wertet dann die Historie mit `git log` aus.

```bash
git init -q --bare -b main srv.git
cat > srv.git/hooks/post-receive <<'EOF'
#!/bin/sh
# Schreibt fuer jede aktualisierte Referenz eine Zeile in ein Audit-Protokoll
while read old new ref; do
  printf '%s user=%s ref=%s old=%s new=%s\n' \
    "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$(id -un)" "$ref" \
    "$(echo "$old" | cut -c1-7)" "$(echo "$new" | cut -c1-7)" >> "$PWD/audit.log"
done
EOF
chmod +x srv.git/hooks/post-receive
git clone -q srv.git work 2>/dev/null && cd work
git config user.name Anna; git config user.email anna@example.org
echo 1 > a.txt && git add . && git commit -q -m "Start" && git push -q origin main
git switch -q -c feature/x && echo x > x.txt && git add . && git commit -q -m "x" && git push -q origin feature/x
git switch -q main && git commit -q --amend -m "Start (umgeschrieben)" && git push -q --force origin main
git push -q origin --delete feature/x
sed 's/^[^ ]* /<zeit> /' ../srv.git/audit.log

echo "--- Historien-Auswertung"
git config user.name Ben; git config user.email ben@example.org
echo 2 > b.txt && git add . && git commit -q -m "direkt auf main"
git log --format='%G?' | sort | uniq -c
git log --first-parent --no-merges --format='%h %an %s' main
git log --format='%an' | sort | uniq -c | sort -rn
```

Das Protokoll enthält vier Zeilen: das Anlegen von `main`, das Anlegen von `feature/x`, das Überschreiben von `main` und das Löschen von `feature/x`. Beim Anlegen besteht der alte Hash nur aus Nullen, beim Löschen der neue; beim Überschreiben stehen zwei echte Hashes. Aus dem Protokoll allein sieht man nicht, ob ein Update ein Fast-Forward war; dafür vergleicht man die beiden Hashes im Repository mit `git merge-base --is-ancestor`. Der Zeitstempel in der Ausgabe wurde zur Lesbarkeit ersetzt. Das Beispiel nutzt `id -un` als Benutzername; auf echten Servern ist das meist ein gemeinsames Dienstkonto, sodass die Zuordnung zur Person über die SSH-Authentifizierung oder die Plattform erfolgen muss. Die Auswertung der Historie im zweiten Teil beantwortet drei typische Prüffragen: wie viele Commits signiert sind (hier zweimal `N`, also keiner), welche Commits direkt auf dem Hauptbranch gelandet sind (`--first-parent --no-merges`) und wer wie viele Commits beigetragen hat.

### Typische Fehler und Stolperfallen.

Falsch ist, dem Autorenfeld im Commit zu trauen, wenn es um Verantwortlichkeit geht. Richtig ist, die Server-Protokolle der Plattform und die Signatur heranzuziehen. Zweiter Fehler: Audit-Daten ungeschützt und veränderbar auf demselben Server zu speichern, auf dem auch die Administratoren arbeiten; die Protokolle sollten an ein getrenntes System weitergeleitet werden. Dritter Fehler: Protokolle nur anzulegen und nie auszuwerten. Ein Audit-Log, das erst nach dem Vorfall gelesen wird, kommt zu spät; sinnvoll sind Alarme auf auffällige Ereignisse wie Force-Push auf geschützte Branches, neue Admin-Rechte oder viele fehlgeschlagene Anmeldungen. Vierter Fehler: Die Aufbewahrungsdauer ungeprüft zu lassen. Zu kurze Fristen verhindern eine spätere Forensik, zu lange Speicherung kann datenschutzrechtliche Fragen aufwerfen (Abschnitt 8.8).

### Praxisbezug.

Compliance-Anforderungen wie Änderungsnachweise im Rahmen von Zertifizierungen führen dazu, dass man belegen muss, dass jede Änderung an Produktionscode durch einen Pull Request mit Freigabe gelaufen ist. Die Auswertung aus dem Beispiel (direkte Commits auf dem Hauptbranch, unsignierte Commits) ist die Rohform eines solchen Nachweises. Im Incident-Fall ist das Audit-Log das Werkzeug der Forensik: Ab wann war das Konto kompromittiert, welche Repositories wurden geklont, welche Branches verändert?

## 8.8 Datenschutz im Repository

Repositories enthalten häufig mehr personenbezogene Daten, als man denkt. Dazu gehören offensichtliche Fälle wie Kundendaten in Testdateien oder ein Produktionsdump, der „nur kurz zum Testen“ eingecheckt wurde, aber auch weniger offensichtliche: Namen und E-Mail-Adressen der Entwicklerinnen und Entwickler in jedem Commit, Namen in Kommentaren, Zugangskennungen von Personen in Konfigurationsdateien, Logdateien mit IP-Adressen. Sobald ein Repository nach dem Datenschutzrecht personenbezogene Daten enthält, gelten dafür die üblichen Grundsätze. In Europa ist das die Datenschutz-Grundverordnung (DSGVO). Der Grundsatz der *Datenminimierung* verlangt, nur so viele Daten zu verarbeiten wie nötig; der Grundsatz der *Speicherbegrenzung* verlangt, Daten nicht länger aufzubewahren, als es der Zweck erfordert. (Dieses Skript ist keine Rechtsberatung. Die Abwägung im Einzelfall liegt bei Ihrer Datenschutzstelle oder Ihrem Rechtsbeistand; die Artikel der DSGVO wurden hier nicht im Originaltext abgerufen.)

Die Schwierigkeit liegt in der Unveränderlichkeit von Git. Das Recht auf Löschung nach Art. 17 DSGVO verpflichtet nach einer Zusammenfassung (Sekundärquelle, abgerufen am 28.10.2026) dazu, Daten ohne unangemessene Verzögerung zu löschen, wenn bestimmte Gründe vorliegen, etwa wenn die Daten für den Zweck nicht mehr erforderlich sind, eine Einwilligung widerrufen wurde oder die Verarbeitung unrechtmäßig war; zugleich gibt es Ausnahmen, etwa zur Geltendmachung oder Verteidigung von Rechtsansprüchen. Ein Löschverlangen für Daten, die in der Git-Historie stecken, verlangt deshalb dieselbe Bereinigung wie ein Secret-Leak (Abschnitt 8.2), mit denselben Grenzen: Klone und Forks anderer bleiben unberührt. Die beste Lösung ist, personenbezogene Daten gar nicht erst in Repositories zu bringen. Testdaten werden synthetisch erzeugt oder anonymisiert, produktive Daten bleiben in den dafür vorgesehenen Systemen, und Dateien mit Nutzerdaten stehen in der `.gitignore`.

Ein Sonderfall sind die Metadaten der Commits selbst. Autorenname und E-Mail-Adresse sind personenbezogene Daten der Mitarbeitenden. In öffentlichen Repositories sind sie weltweit einsehbar. Wer das vermeiden will, nutzt eine neutrale oder von der Plattform bereitgestellte Adresse als `user.email`. Bereits vorhandene unterschiedliche Adressen derselben Person kann man für die Anzeige mit einer `.mailmap`-Datei zusammenführen; das ändert die Commits nicht, sondern nur die Darstellung bestimmter Befehle. Das Beispiel zeigt beides: die Mailmap und ein kleines Prüfskript, das versionierte Dateien nach E-Mail-Adressen und IBAN-ähnlichen Zeichenketten durchsucht.

```bash
git init -q -b main pii && cd pii
git config user.name "Anna Beispiel"; git config user.email anna.privat@gmail.example
echo "Start" > README.md && git add . && git commit -q -m "Start"
git config user.email anna@example.org
echo "x" > x.txt && git add . && git commit -q -m "Arbeit"
git shortlog -sne HEAD
cat > .mailmap <<'EOF'
Anna Beispiel <anna@example.org> <anna.privat@gmail.example>
EOF
git shortlog -sne HEAD
git log --format='%h %aN <%aE>'

mkdir -p testdaten
cat > testdaten/kunden.csv <<'EOF'
name,email,iban
Max Muster,max.muster@kunde.example,AT611904300234573201
EOF
cat > pii-check.sh <<'EOF'
#!/bin/sh
# Sucht in versionierten Dateien nach E-Mail-Adressen und IBANs
git grep -nEI '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}|\b[A-Z]{2}[0-9]{2}[0-9A-Z]{11,30}\b' -- . ':!.mailmap' ':!pii-check.sh'
if [ $? -eq 0 ]; then
  echo "Moegliche personenbezogene Daten gefunden." >&2
  exit 1
fi
echo "Keine Treffer."
EOF
git add . && git commit -q -m "Testdaten"
sh pii-check.sh; echo "Exit-Code: $?"
```

Die erste Ausgabe von `git shortlog -sne` zeigt Anna zweimal, einmal je E-Mail-Adresse. Nach dem Anlegen der `.mailmap` wird sie in der Zusammenfassung unter der neuen Adresse zusammengefasst (zwei Commits). Das `%aN`/`%aE` im Log-Format bedeutet „Autorname/E-Mail unter Anwendung der Mailmap“; mit den Kleinbuchstaben-Varianten `%an`/`%ae` sieht man die tatsächlich gespeicherten Werte, und dort steht weiterhin die alte Adresse. Das ist der entscheidende Punkt: Die Mailmap ist Kosmetik und löscht nichts. Wer eine Adresse wirklich aus der Historie entfernen muss, braucht wieder eine Historienbereinigung. Das Prüfskript findet in der Testdatei sowohl die E-Mail-Adresse als auch die IBAN und beendet sich mit Exit-Code 1; es nimmt sich selbst und die Mailmap aus, weil dort naturgemäß Adressen stehen. In der Praxis wäre es eine Stufe in der Pipeline oder im Pre-commit-Hook.

### Typische Fehler und Stolperfallen.

Falsch ist, einen Produktionsdump „nur vorübergehend“ ins Repository zu legen. Nach dem Push ist er in der Historie. Richtig ist, mit synthetischen Daten zu arbeiten und Dateien mit echten Daten nie zu committen. Zweiter Fehler: Anzunehmen, ein privates Repository sei ein sicherer Ort für personenbezogene Daten. Auch dort können viele Personen lesen, und Klone liegen auf Laptops. Dritter Fehler: Das Löschen aus dem aktuellen Stand für eine Löschung im Sinne des Datenschutzrechts zu halten; in der Historie bleibt der Inhalt. Vierter Fehler: Nach einem Löschverlangen die Historie ohne Abstimmung umzuschreiben. Das betrifft alle Mitarbeitenden und alle Klone; klären Sie das Vorgehen vorher mit der Datenschutzstelle und dem Team, und prüfen Sie, ob eine Ausnahme (zum Beispiel für Rechtsansprüche) greift.

### Praxisbezug.

In Unternehmen gibt es meist ein Verzeichnis der Verarbeitungstätigkeiten und Löschfristen. Repositories mit personenbezogenen Daten gehören dort hinein. Praktisch bedeutet das: Eine Leitlinie „Keine echten Daten in Repositories“, ein automatischer Scan in der Pipeline, ein dokumentierter Ablauf für Löschverlangen und eine Richtlinie für die Commit-Adresse der Mitarbeitenden. Bei Open-Source-Projekten kommt hinzu, dass die Autorendaten öffentlich sind; Beitragende sollten darauf hingewiesen werden.

## 8.9 Security Policies

Eine *Security Policy* legt fest, wie mit Sicherheitslücken in einem Projekt umgegangen wird, vor allem: wie sie gemeldet werden. Ohne diese Information bleibt Meldenden nur der Weg über ein öffentliches Issue, und damit ist die Lücke öffentlich, bevor jemand sie beheben konnte. Die verbreitete Konvention ist eine Datei `SECURITY.md` im Repository. Die GitHub-Dokumentation (abgerufen am 28.10.2026) beschreibt, dass sie Informationen über unterstützte Versionen und über das Melden einer Schwachstelle enthalten soll, und dass GitHub sie auf einer eigenen Seite im Bereich der Sicherheit des Repositories anzeigt. Wo genau die Datei liegen darf, konnte ich auf der abgerufenen Seite nicht bestätigen; üblich sind das Wurzelverzeichnis oder die Verzeichnisse `docs` und `.github`, prüfen Sie das für Ihre Plattform.

Eine brauchbare Policy beantwortet vier Fragen. Welche *Versionen* bekommen Sicherheitsupdates? Wie und an wen wird *gemeldet*, vertraulich und über einen definierten Kanal? Was können Meldende *erwarten*, also welche Bestätigungs- und Bearbeitungszeiten gelten? Und wie läuft die *Veröffentlichung* ab, in der Regel als koordinierte Offenlegung, bei der die Details erst nach Verfügbarkeit eines Fixes bekannt gegeben werden? Die genannten Fristen im Beispiel sind Vorschläge für ein Team und keine gesetzlichen Werte. Je nach Branche und Produkt können gesetzliche Meldepflichten und Fristen hinzukommen; das ist mit Rechts- und Compliance-Stelle zu klären (keine Rechtsberatung).

Das Beispiel zeigt eine kompakte `SECURITY.md` und ein Prüfskript, das sie in einer Pipeline auf Vollständigkeit kontrolliert.

```bash
cat > SECURITY.md <<'EOF'
# Sicherheitsrichtlinie

## Unterstuetzte Versionen
| Version | Sicherheitsupdates |
|---------|--------------------|
| 2.x     | ja                 |
| 1.x     | nur kritische      |
| < 1.0   | nein               |

## Schwachstelle melden
Bitte **nicht** oeffentlich als Issue melden.
Schreiben Sie an security@example.org (PGP-Schluessel siehe unten).

## Ablauf
- Eingangsbestaetigung innerhalb von 3 Werktagen
- Erste Einschaetzung innerhalb von 10 Werktagen
- Koordinierte Veroeffentlichung nach Absprache
EOF
cat > check-security.sh <<'EOF'
#!/bin/sh
datei=""
for p in SECURITY.md .github/SECURITY.md docs/SECURITY.md; do
  [ -f "$p" ] && datei="$p" && break
done
if [ -z "$datei" ]; then echo "FEHLT: SECURITY.md"; exit 1; fi
fehlt=0
for abschnitt in "Unterstuetzte Versionen" "Schwachstelle melden"; do
  if ! grep -q "^## $abschnitt" "$datei"; then echo "FEHLT: Abschnitt '$abschnitt'"; fehlt=1; fi
done
if ! grep -Eq '[A-Za-z0-9._-]+@[A-Za-z0-9.-]+' "$datei"; then echo "FEHLT: Kontaktadresse"; fehlt=1; fi
[ "$fehlt" -eq 0 ] && echo "OK: $datei enthaelt die Pflichtabschnitte"
exit "$fehlt"
EOF
sh check-security.sh; echo "Exit-Code: $?"
```

Die Datei ist bewusst knapp. Die Tabelle nennt unterstützte Versionen, der Abschnitt zum Melden nennt einen nichtöffentlichen Kanal, und der Ablauf setzt Erwartungen. Das Prüfskript sucht die Datei an den üblichen Orten, prüft die beiden Pflichtabschnitte und das Vorhandensein einer Kontaktadresse und gibt bei Mängeln Exit-Code 1 zurück. Entfernt man einen Abschnitt oder die Adresse, meldet das Skript genau, was fehlt. So bleibt die Policy aktuell, auch wenn Personen und Adressen wechseln.

### Typische Fehler und Stolperfallen.

Falsch ist eine Policy, die nur sagt „Bitte melden Sie Lücken“, ohne Kanal und Zeitrahmen. Richtig sind konkrete Adresse, Erwartungen und die Bitte, nicht öffentlich zu melden. Zweiter Fehler: Die Mailbox hinter der Adresse wird von niemandem gelesen. Eine Policy ohne Zuständige ist schlimmer als keine, weil sie Erwartungen weckt. Dritter Fehler: Keine Liste der unterstützten Versionen; Meldende wissen dann nicht, ob ihr Fund noch relevant ist. Vierter Fehler: Sicherheitskorrekturen im öffentlichen Pull Request mit sprechender Beschreibung vorzubereiten, bevor der Fix verteilt ist. Dafür gibt es bei Plattformen private Arbeitsbereiche für Sicherheits-Advisories (weiterführend, nicht abgerufen).

### Praxisbezug.

Die Security Policy ist Teil der Standard-Dateien eines Repositories neben README, CODEOWNERS und CONTRIBUTING (Modul 05) und gehört in jede Repository-Vorlage. Zum Prozess dahinter gehören ein Verantwortlicher, ein Posteingang, ein Ablauf zum Bewerten (zum Beispiel mit dem Bewertungssystem CVSS, weiterführend) und ein Ablauf zum Ausliefern von Fixes über die Release-Verfahren aus Modul 07. Das Beispiel dieses Moduls schließt den Kreis: Vermeiden (8.1), Reagieren (8.2), Belegen (8.3 und 8.7), Begrenzen (8.4 bis 8.6), Schützen (8.8) und Melden (8.9).

## Übungen zu Modul 08

**Aufgabe 1.** Nennen Sie die Schichten, mit denen man Secret-Leaks vermeiden kann, und erklären Sie, warum ein Pre-commit-Hook allein nicht ausreicht.

**Aufgabe 2.** Ein Kollege hat versehentlich ein Passwort committet und gepusht und im nächsten Commit gelöscht. Beschreiben Sie in der richtigen Reihenfolge, was zu tun ist, und begründen Sie die Reihenfolge.

**Aufgabe 3.** Was unterscheidet eine *gültige* von einer *vertrauenswürdigen* Signatur? Wie kommt der Status `U` bei `%G?` zustande?

**Aufgabe 4.** Warum schützt `receive.denyDeletes` Release-Tags nicht, und wie lässt sich der Schutz herstellen?

**Aufgabe 5.** Praktisch: Bauen Sie den Pre-commit-Hook aus Abschnitt 8.1 nach und erweitern Sie ihn um ein weiteres Muster (zum Beispiel Datenbank-URLs mit Passwort der Form `postgres://benutzer:passwort@host`). Testen Sie, dass ein Commit mit einer solchen Zeile abgelehnt wird.

**Aufgabe 6.** Praktisch: Erzeugen Sie in einem Testrepository ein Secret in der Historie und entfernen Sie es mit `git filter-repo` in einem frischen Klon. Prüfen Sie mit `git log -S`, dass der Klon es nicht mehr enthält und dass das Original es weiterhin enthält.

**Aufgabe 7.** Praktisch: Richten Sie SSH-Signaturen ein, signieren Sie einen Commit und einen Tag, und prüfen Sie beide. Ändern Sie die Datei `allowed_signers` so, dass die Prüfung fehlschlägt, und erklären Sie das Ergebnis.

**Aufgabe 8.** Praktisch (Transfer): Schreiben Sie ein Skript, das in einem Repository (a) Remote-URLs mit Zugangsdaten, (b) nicht auf einen Commit-SHA gepinnte Aktionen in `.github/workflows` und (c) das Fehlen einer `SECURITY.md` prüft und mit Exit-Code 1 endet, sobald eine der Prüfungen anschlägt. Beschreiben Sie, wo Sie es in einen Prozess einbinden würden.

## Lösungen zu Modul 08

**Lösung 1.** Schichten: Secrets nicht im Code (Umgebung, Secret-Manager, Vorlage `.env.example`), Ignore-Regeln, Pre-commit-Scan, serverseitiger Scan beim Push, Scan der gesamten Historie in der CI. Der Hook liegt lokal, wird nicht mit dem Repository ausgeliefert und lässt sich mit `--no-verify` umgehen; er gibt schnelles Feedback, erzwingt aber nichts. Verbindlich ist die Prüfung auf dem Server.

**Lösung 2.** Erstens das Secret rotieren beziehungsweise widerrufen, denn nur das beendet die Gefahr; zweitens bewerten (seit wann, wer hatte Zugriff, Nutzung in Zugriffsprotokollen); drittens bei Bedarf die Historie in einem frischen Klon mit `git filter-repo` bereinigen, Force-Push und Team informieren (Rebase statt Merge); viertens den Vorfall dokumentieren und die Ursache beseitigen. Die Reihenfolge begründet sich damit, dass das Secret ab dem Leak als kopiert gilt und das Umschreiben nichts daran ändert.

**Lösung 3.** Gültig: die Signatur ist mathematisch korrekt für den verwendeten Schlüssel. Vertrauenswürdig: der Schlüssel gehört nachweislich zu der Person, die man meint (Eintrag in `allowed_signers`, Schlüssel am Plattformkonto). `U` entsteht, wenn die Signatur gültig ist, der Schlüssel aber keiner Identität zugeordnet ist, wie bei dem fremden Schlüssel im Beispiel; `git verify-commit` meldet „No principal matched“ und endet mit Exit-Code 1.

**Lösung 4.** Die Schalter `receive.denyDeletes` und `receive.denyNonFastForwards` wirkten im Test nur auf Branches; das Löschen und Verschieben eines Tags gelang trotzdem. Schutz entsteht durch eine Tag-Schutzregel der Plattform oder einen `update`-Hook, der Änderungen an bestehenden Tags ablehnt (alter Hash ungleich Null).

**Lösung 5.** Beispielmuster: `[a-z]+://[^:/@ ]+:[^@ ]+@`. Wichtig: Die Zeile im Test darf nur ein erfundenes Passwort enthalten. Der Hook gibt die Fundzeile aus und endet mit Exit-Code 1, der Commit wird abgebrochen. Zum Prüfen `git commit` ohne `--no-verify` verwenden.

**Lösung 6.** Ablauf wie in 8.2: Klon mit `git clone --no-local`, Regelliste mit `alt==>neu`, `git filter-repo --replace-text datei --force`. `git log -S"wert" --all` liefert im bereinigten Klon keine Treffer und im Original zwei (einführender und entfernender Commit).

**Lösung 7.** Ablauf wie in 8.3. Entfernt man die Zeile des Schlüssels aus `allowed_signers`, meldet `git verify-commit` „No principal matched“ und `%G?` zeigt `U`: Die Signatur ist gültig, aber der Schlüssel keiner Person zugeordnet.

**Lösung 8.** Eine Möglichkeit ist ein Shell-Skript, das die drei Prüfungen aus 8.4, 8.6 und 8.9 hintereinander ausführt und die Exit-Codes zusammenfasst (`status=1`, sobald eine Prüfung fehlschlägt). Einbindung: als Stufe in der CI-Pipeline bei jedem Pull Request und als regelmäßig laufender Job über alle Repositories; zusätzlich als Pflichtprüfung in den Branch-Regeln des Hauptbranches.
