# Modul 04 - Remote Repositories und Kollaboration

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (4.1 bis 4.8), danach folgen „Übungen zu Modul 04“ (8 Aufgaben) und „Lösungen zu Modul 04“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 1

> Die Modulübersicht und die Summe der Lerneinheiten stimmen überein (10 h).

**Lernziele des Moduls:** Remotes, Pull Requests, Reviews und Teamprozesse einsetzen; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Bisher haben wir allein in einem Repository gearbeitet. Dieses Modul zeigt, wie mehrere Repositories zusammenspielen: wie man Änderungen austauscht, wie man sie prüfen lässt und wie ein Team Rechte und Regeln organisiert.

| **Nr.** | **Lerneinheit**                  | **Dauer (h)** | **Prio** |
|:--------|:---------------------------------|--------------:|:--------:|
| 4.1     | Remote Repositories              |          1,25 |    1     |
| 4.2     | Fetch, Pull und Push             |          1,25 |    1     |
| 4.3     | Pull Requests und Merge Requests |          1,25 |    1     |
| 4.4     | Code Reviews                     |          1,25 |    1     |
| 4.5     | Forking Workflows                |          1,25 |    1     |
| 4.6     | Team-Berechtigungen              |          1,25 |    1     |
| 4.7     | Remote-Konflikte                 |          1,25 |    2     |
| 4.8     | Kollaborationsregeln             |          1,25 |    2     |
|         | **Summe**                        |      **10,0** |          |

## 4.1 Remote Repositories

Ein *Remote* ist ein anderes Repository, mit dem das eigene Repository Änderungen austauscht. Meist liegt es auf einem Server, etwa bei GitHub, GitLab oder Azure Repos, es kann aber ebenso ein Verzeichnis auf der Festplatte oder einem Netzlaufwerk sein. Technisch ist ein Remote nur ein Eintrag in der Konfiguration: ein Kurzname und eine URL. Weil Git verteilt arbeitet (Modul 01), ist kein Repository von Natur aus „der Server“. Dass die meisten Teams ein gemeinsames Repository als Abgleichspunkt vereinbaren, ist eine Absprache. Solche gemeinsamen Repositories sind fast immer *Bare-Repositories*, also Repositories ohne Arbeitsverzeichnis, denn niemand arbeitet direkt auf dem Server.

Der Standardname für das Remote, von dem man geklont hat, ist `origin`. Er ist eine reine Konvention; man kann ihn umbenennen oder weitere Remotes hinzufügen. Verbreitet ist der Name `upstream` für das Original-Repository, wenn man mit einem Fork arbeitet (siehe Abschnitt 4.5). Zu jedem Remote gehören die *Remote-Tracking-Branches*, schreibgeschützte lokale Referenzen wie `origin/main`, die den zuletzt bekannten Stand der Branches auf dem Remote festhalten. Sie werden nur durch `git fetch` (und `git pull`, das intern fetcht) aktualisiert, niemals automatisch im Hintergrund. Das erklärt, warum `git status` „up to date with origin/main“ sagen kann, obwohl der Server längst neuere Commits hat: Der Status vergleicht nur mit dem zuletzt geholten Stand.

Der Zugriff auf Remotes läuft über zwei Hauptprotokolle. Bei *HTTPS* (URL wie `https://host/team/projekt.git`) authentifiziert man sich mit Benutzername und einem *Personal Access Token*; moderne Plattformen akzeptieren für Git-Operationen keine normalen Passwörter mehr. Ein *Credential Helper* speichert das Token sicher im Betriebssystem. Bei *SSH* (URL wie `git@host:team/projekt.git`) authentifiziert man sich mit einem Schlüsselpaar: Der private Schlüssel bleibt auf dem eigenen Rechner, der öffentliche wird auf der Plattform hinterlegt. SSH braucht keine Token-Verwaltung und ist bei häufigem Arbeiten bequem; HTTPS funktioniert dagegen auch hinter strengen Firewalls, die nur Webverkehr durchlassen. Welche Variante gilt, hängt am Teamstandard.

Das folgende Beispiel baut eine komplette Remote-Umgebung lokal nach: ein Bare-Repository als „Server“ und zwei Klone, die für zwei Personen stehen. So lässt sich jeder Befehl ausprobieren, ohne Konto bei einer Plattform.

```bash
git init --bare -b main server.git
git clone server.git alice
cd alice
git config user.name "Alice"; git config user.email alice@example.org
echo "Projekt" > README.md
git add . && git commit -m "README angelegt"
git push -u origin main

git remote -v
git remote show origin
git branch -vv
cat .git/config

git remote rename origin zentrale
git remote set-url zentrale ../server.git
git remote rename zentrale origin
```

`git clone server.git alice` legt den Klon an und trägt automatisch das Remote `origin` ein; die Warnung „You appear to have cloned an empty repository“ ist hier erwartet. Beim ersten `git push -u origin main` bedeutet `-u` (`--set-upstream`), dass der lokale Branch `main` künftig `origin/main` als Gegenstück hat. Danach funktionieren `git push` und `git pull` ohne weitere Angaben. `git remote -v` listet die Remotes mit ihren URLs, getrennt nach Holen (*fetch*) und Senden (*push*). `git remote show origin` kontaktiert den Server und zeigt unter anderem, welche Branches es dort gibt und welchen Stand sie im Vergleich zu den lokalen haben. In `.git/config` sieht man die zugrunde liegenden Einträge: den Abschnitt `[remote "origin"]` mit `url` und `fetch`, sowie `[branch "main"]` mit der Zuordnung zum Upstream. Die letzten Zeilen zeigen, wie man ein Remote umbenennt und seine URL ändert, etwa nach einem Umzug des Repositories.

Wer mit SSH arbeiten möchte, erzeugt ein Schlüsselpaar und hinterlegt den öffentlichen Teil auf der Plattform. Die Befehle laufen auf Ihrem eigenen Rechner und lassen sich im Test-Setup oben nicht nachstellen, weil sie ein Konto brauchen; sie sind deshalb separat abgedruckt.

**Listing:** Nicht ausführen im Übungsordner: SSH-Schlüssel anlegen

```text
ssh-keygen -t ed25519 -C "alice@firma.example"
cat ~/.ssh/id_ed25519.pub        # diesen Text bei der Plattform eintragen
ssh -T git@github.com            # Verbindung testen (Beispiel GitHub)
git remote set-url origin git@github.com:firma/projekt.git
```

Der Algorithmus `ed25519` gilt als modern und kompakt. Der Zusatz `-C` ist nur ein Kommentar im Schlüssel. Das Passwort (*Passphrase*), nach dem `ssh-keygen` fragt, schützt den privaten Schlüssel, falls jemand die Datei in die Hände bekommt; dafür gibt es einen `ssh-agent`, der die Passphrase pro Sitzung zwischenspeichert.

> **Merke**
>
> Ein Remote ist nur ein Name für eine URL. `origin/main` ist ein lokaler Merkzettel über den Stand von `main` auf dem Remote, aktualisiert nur beim Fetch. Zugangsdaten gehören nie in die URL; dafür gibt es Credential Helper und SSH-Schlüssel.

### Typische Fehler und Stolperfallen.

Ein häufiger und gefährlicher Fehler ist, das Token in die Remote-URL zu schreiben, zum Beispiel `https://user:ghp_xxxx@github.com/firma/projekt.git`. Das Token steht dann im Klartext in `.git/config`, erscheint in Logs, Screenshots und Fehlermeldungen. Falsch ist diese Abkürzung; richtig ist eine URL ohne Zugangsdaten und ein Credential Helper. Ein zweiter Fehler ist das Mischen der Protokolle: Wer die Remote-URL per HTTPS klont, aber SSH-Schlüssel einrichtet, wundert sich über Passwortabfragen, weil Git die URL verwendet und nicht den Schlüssel. Hier hilft `git remote -v` und gegebenenfalls `git remote set-url`. Ein dritter Fehler ist die Verwechslung von Branch und Remote-Tracking-Branch: `origin/main` lässt sich nicht „committen“; man arbeitet auf dem lokalen `main`.

### Praxisbezug.

In Unternehmen kommt es oft zu Umzügen: Repositories ziehen auf eine andere Plattform oder in eine andere Organisation um. Mit `git remote set-url` passt man alle Klone an, ohne Historie oder lokale Branches zu verlieren. Mehrere Remotes nutzt man zum Beispiel, um Änderungen parallel auf einen internen Server und auf eine öffentliche Plattform zu spiegeln.

## 4.2 Fetch, Pull und Push

Drei Befehle bewegen Änderungen zwischen Remote und lokalem Repository. `git fetch` lädt neue Commits, Branches und Tags vom Remote herunter und aktualisiert die Remote-Tracking-Branches; es verändert weder Ihre lokalen Branches noch das Arbeitsverzeichnis. Die Git-Dokumentation beschreibt es als Herunterladen von Referenzen samt den Objekten, die zur Vervollständigung ihrer Historie nötig sind (abgerufen am 28.10.2026). Fetch ist damit immer gefahrlos: Man kann nachsehen, was sich getan hat, ohne etwas zu riskieren. `git pull` ist die Kurzform für *fetch plus Integration*: Es holt und bringt das Geholte anschließend in den aktuellen Branch, wahlweise per Merge oder per Rebase. `git push` schließlich lädt lokale Commits auf das Remote hoch und verschiebt dort den Branch.

Entscheidend ist, was bei Abweichungen geschieht. Ein Push wird vom Server nur akzeptiert, wenn er ein *Fast-forward* ist, wenn also der bisherige Stand des Branches auf dem Server in Ihrer Historie enthalten ist. Hat jemand anderes inzwischen etwas gepusht, ist Ihr Stand nicht mehr „vorn“, und der Server lehnt mit `rejected (fetch first)` ab. Das schützt vor dem Überschreiben fremder Arbeit. Die Lösung besteht darin, die fremden Änderungen erst zu integrieren (fetch und merge oder rebase) und dann erneut zu pushen. Laut der Dokumentation zu `git pull` ist die Voreinstellung inzwischen `--ff-only`: Pull aktualisiert nur, wenn keine abweichende lokale Historie existiert, und bricht sonst ab, statt stillschweigend einen Merge-Commit oder Rebase zu erzeugen (abgerufen am 28.10.2026). Ältere Git-Versionen verlangen in diesem Fall eine Entscheidung über `pull.rebase` und geben einen Hinweis aus; in beiden Fällen muss man bewusst wählen, wie man zusammenführt.

Das Beispiel nutzt die Umgebung aus Abschnitt 4.1 mit zwei Klonen. Alice und Bob ändern parallel verschiedene Dateien. Bob pusht zuerst; Alices Push wird abgelehnt und mit `git pull --rebase` gelöst.

```bash
git init --bare -b main server.git
git clone server.git alice
git clone server.git bob

cd alice
git config user.name "Alice"; git config user.email alice@example.org
echo "Start" > README.md
git add . && git commit -m "Start"
git push -u origin main

cd ../bob
git config user.name "Bob"; git config user.email bob@example.org
git pull
echo "Bobs Datei" > bob.txt
git add . && git commit -m "bob.txt hinzugefuegt"
git push

cd ../alice
echo "Alices Datei" > alice.txt
git add . && git commit -m "alice.txt hinzugefuegt"
git push

git fetch
git status -sb
git log --oneline --left-right main...origin/main
git pull --rebase
git push
git log --oneline --graph
```

Bobs Push geht durch. Alices `git push` wird mit `! [rejected] main -> main (fetch first)` und dem Hinweis abgelehnt, dass das Remote Arbeit enthält, die lokal fehlt. `git fetch` lädt Bobs Commit herunter, ohne etwas zu integrieren. `git status -sb` zeigt nun `[ahead 1, behind 1]`: Alice hat einen Commit, den der Server nicht kennt, und der Server einen, den Alice nicht hat. `git log --left-right main...origin/main` macht das sichtbar; die Pfeile `<` und `>` markieren, auf welcher Seite ein Commit liegt. `git pull --rebase` legt Alices Commit auf Bobs Stand, danach ist der Push ein Fast-forward. Das Ergebnis ist eine gerade Linie ohne Merge-Commit. Wer lieber mit Merge-Commit integriert, nutzt `git pull --no-rebase`.

Einige Push-Einstellungen sind gut zu kennen. Laut Dokumentation ist `push.default` seit Git 2.0 auf `simple` gesetzt: Ein argumentloser Push sendet den aktuellen Branch an den gleichnamigen Branch auf dem Remote und verlangt, dass dessen Upstream denselben Namen hat (abgerufen am 28.10.2026). Mit `push.autoSetupRemote true` wird beim ersten Push eines neuen Branches der Upstream automatisch gesetzt, sodass das `-u` entfällt. Einen Remote-Branch löscht man mit `git push origin --delete <name>`. Und nach dem Umschreiben eigener, nicht geteilter Historie ist ein Überschreiben nötig. Dafür nimmt man `git push --force-with-lease`: Diese Variante überschreibt nur dann, wenn der Remote-Branch noch auf dem Stand steht, den man zuletzt gesehen hat. Hat jemand inzwischen etwas dazugelegt, bricht sie ab, im Gegensatz zu `--force`, das ohne Rücksicht überschreibt.

> **Ergänzung aus der Nachrecherche**
>
> **Grenzen von `--force-with-lease`.** Die Git-Dokumentation (abgerufen am 28.10.2026) weist darauf hin, dass der Schutz schwach wird, wenn im Hintergrund ein Prozess automatisch `git fetch` ausführt, etwa ein Editor oder ein Cron-Job: Der Fetch aktualisiert den Remote-Tracking-Branch, mit dem der Lease verglichen wird, und das Überschreiben erscheint dann „erlaubt“, obwohl man die neuen Commits nie gesehen hat. Zusätzlich gibt es `--force-if-includes`, das verlangt, dass die Spitze des Remote-Tracking-Branches lokal integriert wurde; mit `push.useForceIfIncludes true` lässt es sich dauerhaft aktivieren.

### Typische Fehler und Stolperfallen.

Der gefährlichste Fehler ist `git push --force` als Reflex nach einer Ablehnung. Falsch: `git push --force`, „damit es endlich geht“. Das überschreibt Commits der Kollegen auf dem Server, und sie bemerken es oft erst, wenn ihr Push scheitert. Richtig: zuerst `git fetch` und die Lage mit `git log --left-right main...origin/main` ansehen, dann integrieren. Ein zweiter Fehler ist, `git pull` auf einem Branch mit uncommitteten Änderungen zu starten; das kann zu verwirrenden Konflikten führen. Und drittens: Wer nie fetcht, arbeitet mit veralteten Remote-Tracking-Branches und sieht Divergenzen zu spät.

### Praxisbezug.

Viele Entwickelnde fetchen mehrfach täglich und pullen erst, wenn sie bereit sind zu integrieren. Mit `fetch.prune true` entfernt Git beim Fetch automatisch Remote-Tracking-Branches, die auf dem Server gelöscht wurden. In IDEs steckt hinter dem Knopf „Sync“ meist genau Pull gefolgt von Push; wer die Einstellung `pull.rebase` kennt, versteht, warum ein Sync einmal einen Merge-Commit erzeugt und ein anderes Mal nicht.

## 4.3 Pull Requests und Merge Requests

Ein *Pull Request* (PR; bei GitLab *Merge Request*, MR) ist keine Git-Funktion, sondern ein Feature der Hosting-Plattformen. Er ist ein Antrag, die Änderungen eines Branches in einen anderen, meist den Hauptbranch, zu übernehmen. Der Antrag bündelt an einem Ort alles, was für die Entscheidung nötig ist: die Änderungen als Diff, eine Beschreibung, die Diskussion, die Ergebnisse automatischer Prüfungen (CI) und die Freigaben der Reviewer. Erst wenn die Bedingungen erfüllt sind, wird der Branch zusammengeführt. Der Pull Request ist damit das zentrale Instrument, um Qualität und Nachvollziehbarkeit in der Teamarbeit sicherzustellen.

Der Ablauf ist in fast jedem Team ähnlich. Man legt einen Branch an, committet, pusht den Branch und eröffnet auf der Plattform einen Pull Request gegen den Zielbranch. Eine gute Beschreibung nennt das Ziel (*warum*), die wichtigsten Änderungen (*was*), Hinweise zum Testen und das zugehörige Ticket. Ein *Draft* (Entwurf) kennzeichnet einen Pull Request, der noch nicht zur Prüfung freigegeben ist. Reviewer kommentieren, fordern Änderungen an oder genehmigen. Nachträgliche Commits auf dem Branch erscheinen automatisch im Pull Request. Zum Abschluss wählt man die Merge-Art (Merge-Commit, Squash oder Rebase, siehe Modul 03) und löscht den Branch.

Technisch ist ein Pull Request ein Vergleich zweier Branches. Git selbst kennt dafür den Befehl `git request-pull`, der eine Zusammenfassung erzeugt, die man per E-Mail verschickt; so arbeitet etwa das Linux-Kernel-Projekt. Das Beispiel zeigt diesen ursprünglichen Mechanismus, der dem Plattform-PR zugrunde liegt, und die Befehle, mit denen man den Inhalt eines Pull Requests lokal prüft.

```bash
git init --bare -b main server.git
git clone server.git dev
cd dev
git config user.name "Dev"; git config user.email dev@example.org
echo "v1" > app.txt
git add . && git commit -m "Start"
git push -u origin main

git switch -c feature/anzeige
echo "Anzeige" > anzeige.txt
git add . && git commit -m "Anzeige hinzugefuegt"
echo "Anzeige v2" > anzeige.txt
git commit -am "Anzeige ueberarbeitet"
git push -u origin feature/anzeige

git request-pull main ../server.git feature/anzeige
git diff main...feature/anzeige --stat
git log main..feature/anzeige --oneline
```

Nach dem Push des Feature-Branches erzeugt `git request-pull` einen Text, der Basis-Commit, Repository-URL, Branch, die enthaltenen Commits und die Diffstatistik nennt. Das ist ein Pull Request in Reinform. Die beiden Befehle darunter sind das lokale Gegenstück zur Ansicht in der Plattform: `git diff main...feature/anzeige` (drei Punkte) zeigt die Änderungen des Feature-Branches seit der Abzweigung, genau das, was ein Reviewer im Reiter „Files changed“ sieht. `git log main..feature/anzeige` listet die Commits, die der Branch zusätzlich enthält. Auf den Plattformen legt man den Pull Request meist im Browser oder mit einem Kommandozeilenwerkzeug an, etwa mit der GitHub-CLI. Die folgenden Zeilen sind ein Beispiel dafür und brauchen ein Konto, daher sind sie nicht Teil des lokalen Übungsablaufs.

**Listing:** Beispiel mit GitHub-CLI (benötigt Konto)

```text
gh pr create --base main --head feature/anzeige \
  --title "Anzeige hinzugefuegt" \
  --body "Warum: Nutzer brauchen eine Statusanzeige. Test: siehe README."
gh pr checks
gh pr merge --squash --delete-branch
```

Für Reviewer ist es wichtig, einen Pull Request lokal ausprobieren zu können. GitHub legt dafür unter `refs/pull/<Nummer>/head` eine Referenz an, die man holen kann: `git fetch origin pull/12/head:pr-12`, danach `git switch pr-12`. GitLab bietet Entsprechendes unter `refs/merge-requests/<Nummer>/head`. (Diese Referenznamen stammen aus dem Wissensstand des Autors und wurden in dieser Sitzung nicht abgerufen.)

### Typische Fehler und Stolperfallen.

Ein typischer Fehler sind riesige Pull Requests mit mehreren Themen. Falsch ist ein Pull Request mit 3000 Zeilen und der Beschreibung „Diverse Änderungen“; niemand prüft ihn sorgfältig, und die Freigabe wird zur Formalität. Richtig sind kleine, thematisch einheitliche Pull Requests mit einer Beschreibung, die den Zweck erklärt, idealerweise mit höchstens einigen hundert geänderten Zeilen. Ein zweiter Fehler ist, den Pull Request nur als Formsache zu betrachten und den Branch trotz roter CI-Prüfungen zu mergen. Ein dritter: Änderungen nach der Freigabe nachzuschieben, ohne die Reviewer zu informieren. Viele Plattformen können Freigaben bei neuen Commits automatisch zurückziehen; das sollte man einschalten.

### Praxisbezug.

Der Pull Request ist der Ort, an dem Teams ihre Qualitätsregeln durchsetzen: Pipeline grün, mindestens eine Freigabe, Tests ergänzt, Ticket verknüpft. In regulierten Umgebungen dient er außerdem als Nachweis, dass jede Änderung von einer zweiten Person gesehen wurde (Vier-Augen-Prinzip). Deshalb werden direkte Pushes auf den Hauptbranch in der Regel technisch gesperrt (siehe Abschnitt 4.6 und Modul 05).

## 4.4 Code Reviews

Ein *Code Review* ist die Prüfung einer Änderung durch eine oder mehrere andere Personen, bevor sie in die Hauptlinie gelangt. Sein Zweck ist breiter, als es zunächst klingt. Natürlich sollen Fehler gefunden werden, doch Studien und Erfahrung zeigen, dass Reviews ebenso dazu dienen, Wissen im Team zu verbreiten, Konventionen durchzusetzen und die Verständlichkeit des Codes zu prüfen: Wenn der Reviewer den Code nicht versteht, ist das ein Befund. Ein gutes Review ist deshalb ein Gespräch über die Lösung und keine Fehlersuche gegen eine Person.

Damit Reviews wirksam und fair sind, braucht es vorab festgelegte Kriterien. Zur *Qualität* gehören Korrektheit, Lesbarkeit, Testabdeckung und die Einhaltung von Standards. Zur *Sicherheit* gehören unter anderem die Prüfung von Eingaben, der Umgang mit Geheimnissen (keine Zugangsdaten im Code), Berechtigungen und neue Abhängigkeiten. Zum *Betrieb* gehören Logging, Fehlerbehandlung, Konfigurierbarkeit, Auswirkungen auf Performance und die Frage, ob die Änderung rückrollbar ist. Eine kurze Checkliste im Pull-Request-Template hilft, nichts zu vergessen. Für das Feedback gilt: konkret und begründet formulieren, zwischen Pflicht („muss geändert werden“) und Vorschlag („könnte man“) unterscheiden, und auch Gutes benennen.

Auf der Autorenseite geht es darum, Feedback nachvollziehbar einzuarbeiten. Dafür gibt es zwei Muster. Beim ersten legt man für jede Korrektur einen zusätzlichen Commit an; der Reviewer sieht in der Plattform genau, was sich seit seiner letzten Prüfung geändert hat. Beim zweiten räumt man vor dem Merge auf: Mit `git commit --fixup=<hash>` markiert man eine Korrektur als Zugehörigkeit zu einem früheren Commit, und `git rebase -i --autosquash` sortiert sie dorthin und fasst sie zusammen. Dieses Umschreiben ist nur auf dem eigenen, noch nicht gemergten Feature-Branch unproblematisch; Reviewer sollten vorher informiert sein, weil ihre Kommentare an Commits hängen können. Ein nützliches Werkzeug dafür ist `git range-diff`: Es vergleicht zwei Fassungen einer Commit-Serie, also etwa den Stand vor und nach dem Rebase, und zeigt, welche Commits sich inhaltlich wirklich geändert haben.

Das Beispiel führt vor, wie eine Korrektur nach dem Review per Fixup eingearbeitet und mit `git range-diff` nachvollzogen wird. Die Variable `GIT_SEQUENCE_EDITOR=true` sorgt dafür, dass der Rebase ohne Editor durchläuft.

```bash
git init --bare -b main server.git
git clone server.git dev && cd dev
git config user.name "Dev"; git config user.email dev@example.org
printf "Zeile 1\nZeile 2\n" > text.txt
git add . && git commit -m "Basis"
git push -u origin main

git switch -c feature/text
echo "Neue Zeile" >> text.txt
git commit -am "Neue Zeile ergaenzt"
echo "Zweite Datei" > zwei.txt
git add . && git commit -m "zwei.txt hinzugefuegt"
git push -u origin feature/text

# Review: Tippfehler in Commit 1 ist aufgefallen
sed -i 's/Neue Zeile/Neue Zeile (korrigiert)/' text.txt
git commit -a --fixup=HEAD~1

git branch vorher
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash main
git log --oneline
git range-diff main vorher feature/text
git push --force-with-lease
```

Der Reviewer hat einen Mangel im ersten Commit bemängelt. `git commit --fixup=HEAD~1` legt einen Commit mit dem Titel `fixup! Neue Zeile ergaenzt` an. Der Befehl `git branch vorher` merkt sich den Stand vor dem Aufräumen. Der Rebase mit `--autosquash` verschiebt den Fixup hinter den Zielcommit und verschmilzt beide; im Log stehen danach wieder nur zwei Commits. `git range-diff main vorher feature/text` vergleicht die Serie vor und nach dem Rebase. Die Ausgabe ist zeilenweise zu lesen: Der ursprüngliche erste Commit (Spalte links) ist entfallen (`<`), weil er im Ergebnis mit dem Fixup verschmolzen wurde. Der Fixup-Commit wird dem neuen, verschmolzenen ersten Commit zugeordnet und als geändert markiert (`!`); darunter zeigt Git genau die inhaltliche Änderung, nämlich die korrigierte Zeile. Der zweite Commit ist unverändert (`=`), nur sein Hash ist neu. Der abschließende `--force-with-lease`-Push ist hier legitim, weil der Branch nur dem Autor gehört.

### Typische Fehler und Stolperfallen.

Falsch ist ein Review, das nur Stilfragen kommentiert, aber die Logik nicht prüft; ebenso falsch ist ein Review ohne Kontext („LGTM“ nach zehn Sekunden). Richtig ist, mindestens zu prüfen, ob die Änderung das Ziel aus dem Ticket erreicht, ob Randfälle und Fehlerpfade bedacht sind und ob Tests dazugekommen sind. Ein Fehler auf der Autorenseite ist, nach der Freigabe die Historie umzuschreiben, ohne es zu sagen: Reviewer verlieren die Zuordnung ihrer Kommentare, und der Merge-Zeitpunkt wird unübersichtlich. Und schließlich: Eine Review-Kultur, in der Kritik persönlich wird, zerstört das Vertrauen. Kommentare beziehen sich auf den Code, nicht auf die Person.

### Praxisbezug.

Reviews sind ein Engpass, wenn sie liegen bleiben. Viele Teams vereinbaren daher Fristen (zum Beispiel erste Reaktion innerhalb eines Arbeitstages) und begrenzen die Größe der Änderungen. Automatisierte Prüfungen (Linter, Tests, Security-Scans) erledigen das Mechanische vorab, damit Menschen sich auf Design, Logik und Risiken konzentrieren können. Das Thema greifen wir in Abschnitt 4.8 und in Modul 09 wieder auf.

## 4.5 Forking Workflows

Bei Open-Source-Projekten und überall dort, wo Außenstehende beitragen sollen, ohne Schreibrechte auf das Hauptrepository zu haben, nutzt man den *Forking-Workflow*. Ein *Fork* ist eine serverseitige Kopie eines Repositories unter dem eigenen Konto oder in der eigenen Organisation. Sie ist unabhängig vom Original: Der Eigentümer des Forks kann dort nach Belieben pushen, ohne das Original zu berühren. Wer eine Änderung beitragen möchte, arbeitet in seinem Fork und stellt dann einen Pull Request vom Fork an das Original-Repository. Die Betreuer des Originals prüfen und übernehmen die Änderung oder lehnen sie ab. Auf diese Weise bleibt der Schreibzugriff auf das Original auf wenige Personen beschränkt, während jeder beitragen kann.

Technisch entstehen zwei Remotes im lokalen Klon des Forks. `origin` zeigt auf den eigenen Fork, in den man pushen darf. `upstream` zeigt auf das Originalrepository, aus dem man nur liest. Weil das Original weiterläuft, während man arbeitet, muss man den Fork regelmäßig *synchronisieren*: Man holt von `upstream`, bringt den Stand in den lokalen `main`, und pusht ihn in den eigenen Fork. Der Hauptbranch des Forks sollte dabei nie eigene Commits enthalten, damit sich dieser Abgleich immer als reiner Fast-forward erledigen lässt. Eigene Arbeit gehört in Feature-Branches.

Das Beispiel spielt die Situation lokal durch: ein Original-Repository, ein Fork (hier als Bare-Klon), ein lokaler Klon des Forks, ein Beitrag und die Synchronisation nach einer Änderung im Original.

```bash
# Original-Repository ("upstream") mit Anfangsstand
git init --bare -b main original.git
git clone original.git pflege && cd pflege
git config user.name "Maintainer"; git config user.email m@example.org
echo "v1" > lib.txt
git add . && git commit -m "Start"
git push -u origin main
cd ..

# Fork (serverseitige Kopie) und lokaler Klon des Forks
git clone --bare original.git fork.git
git clone fork.git beitrag && cd beitrag
git config user.name "Contributor"; git config user.email c@example.org
git remote add upstream ../original.git
git remote -v

# Beitrag in eigenem Branch
git switch -c fix/tippfehler
echo "v1 (korrigiert)" > lib.txt
git commit -am "Tippfehler korrigiert"
git push -u origin fix/tippfehler
cd ..

# Im Original geht es weiter
cd pflege
echo "neu" > neu.txt
git add . && git commit -m "Neue Datei im Original"
git push
cd ../beitrag

# Fork synchronisieren
git switch main
git fetch upstream
git merge --ff-only upstream/main
git push origin main
git log --oneline --all --graph
```

`git clone --bare` erzeugt aus dem Original ein Bare-Repository, das wir als Fork behandeln. Im lokalen Klon des Forks legt `git remote add upstream` die Verbindung zum Original an. Der Beitrag wird in einem eigenen Branch erstellt und in den Fork gepusht; auf der Plattform würde man jetzt den Pull Request vom Fork an das Original eröffnen. Anschließend entwickelt sich das Original weiter. Zum Synchronisieren holt man mit `git fetch upstream`, bringt den Stand mit `git merge --ff-only upstream/main` in den lokalen `main` und schickt ihn mit `git push origin main` in den Fork. Das `--ff-only` ist eine Absicherung: Hätte `main` im Fork eigene Commits, bräche der Befehl ab, und man würde die Abweichung bemerken, statt unbemerkt einen Merge-Commit zu erzeugen. Die Plattformen bieten für diese Synchronisation auch einen Knopf in der Oberfläche an.

### Typische Fehler und Stolperfallen.

Falsch ist es, im `main` des Forks direkt zu arbeiten. Dann weicht er vom Original ab, und jede Synchronisation führt zu Konflikten; der Pull Request an das Original enthält außerdem ungewollte Commits. Richtig ist, für jeden Beitrag einen frischen Branch vom aktuellen `upstream/main` abzuzweigen. Ein zweiter Fehler: Einen alten Fork nach Monaten unverändert für einen Beitrag zu verwenden; der Pull Request zeigt dann viele fremde Änderungen oder kann nicht gemergt werden, bevor man `main` aktualisiert hat. Dritter Fehler: Den Pull Request gegen den falschen Zielbranch eröffnen, etwa gegen `main` des eigenen Forks statt des Originals.

### Praxisbezug.

Auch in Firmen setzt man Forking-Workflows ein, zum Beispiel bei internen Bibliotheken mit wenigen Betreuern und vielen Beitragenden oder bei Auftragnehmern, die keine Schreibrechte erhalten sollen. In Open-Source-Projekten ist es der Normalfall. Beim Beitragen lohnt sich ein Blick in die Datei `CONTRIBUTING.md` des Originals, die Anforderungen an Branches, Commits und Tests beschreibt (siehe Modul 05).

## 4.6 Team-Berechtigungen

Wer darf was in einem Repository? Diese Frage ist keine Nebensache, sondern eine zentrale Sicherheitsfrage: Ein Repository enthält den Quellcode, oft Konfigurationen, und die Pipeline, die daraus Produktionssysteme baut. Schreibrechte auf den Hauptbranch sind deshalb faktisch Rechte auf die Produktion. Der Leitgedanke heißt *Least Privilege* (geringste nötige Rechte): Jede Person und jeder Automatismus erhält genau die Rechte, die für die Aufgabe nötig sind, und nicht mehr.

Die Plattformen bilden dies über Rollen ab. Die Bezeichnungen unterscheiden sich, die Abstufung ist ähnlich: eine reine Leserolle (Lesen, Klonen, Kommentieren), eine Rolle für Mitarbeitende am Code (Branches pushen, Pull Requests eröffnen), eine Rolle für Betreuende (Pull Requests mergen, Einstellungen verwalten) und die Administration (Löschen, Rechte vergeben, Geheimnisse verwalten). Die Rechte vergibt man nicht an Einzelpersonen, sondern an *Teams* oder Gruppen. Das macht Ein- und Austritte beherrschbar, denn mit dem Teamwechsel ändern sich die Rechte automatisch. Dazu kommt die Absicherung kritischer Branches: Direkte Pushes auf `main` werden gesperrt, Änderungen laufen nur über Pull Requests mit Pflichtprüfungen. Auf GitHub heißt das Konzept inzwischen vor allem *Rulesets*; die Dokumentation listet dort Regeln wie das Einschränken von Erzeugen, Aktualisieren und Löschen, das Blockieren von Force-Pushes, verpflichtende Pull Requests mit Freigaben, verpflichtende Statusprüfungen und das Verlangen signierter Commits (abgerufen am 28.10.2026). Einzelheiten und der Vergleich der Plattformen folgen in Modul 05 und Modul 11.

> **Ergänzung aus der Nachrecherche**
>
> **CODEOWNERS.** Mit einer Datei `CODEOWNERS` legt man fest, wer für welche Pfade zuständig ist. Laut GitHub-Dokumentation (abgerufen am 28.10.2026) liegt die Datei in `.github/`, im Wurzelverzeichnis oder in `docs/`; gefunden wird die erste in dieser Reihenfolge. Die Muster ähneln denen der `.gitignore`, doch Negation mit `!` und Zeichenklassen mit `[ ]` funktionieren nicht. Die *letzte* passende Zeile gewinnt. Eigentümer werden bei Pull Requests automatisch als Reviewer angefragt; schaltet man in einem Ruleset oder Branch-Schutz die Pflicht zur Prüfung durch Code Owners ein, genügt die Freigabe eines der genannten Eigentümer. Eigentümer brauchen ausdrückliches Schreibrecht, und nur Organisations-Teams (keine Enterprise-Teams) sind zulässig. Auf anderen Plattformen gibt es Entsprechungen mit teils anderer Syntax.

Das folgende Beispiel zeigt zwei Dinge: eine `CODEOWNERS`-Datei und, serverseitig in Git selbst, wie sich Regeln durchsetzen lassen. Ein Bare-Repository kennt Konfigurationen wie `receive.denyNonFastForwards` und `receive.denyDeletes`, und ein `pre-receive`-Hook kann Pushes ablehnen, bevor sie angenommen werden. Das sind die Mechanismen, auf denen Plattformregeln im Kern beruhen.

```bash
git init --bare -b main server.git
cd server.git
git config receive.denyNonFastForwards true
git config receive.denyDeletes true
cat > hooks/pre-receive <<'EOF'
#!/bin/sh
# Direkte Pushes auf main verbieten, ausser durch den Merge-Dienst
while read old new ref; do
  if [ "$ref" = "refs/heads/main" ] && [ "$(id -un)" != "merge-bot" ] \
     && [ "$old" != "0000000000000000000000000000000000000000" ]; then
    echo "Direkte Pushes auf main sind nicht erlaubt - bitte Pull Request nutzen." >&2
    exit 1
  fi
done
EOF
chmod +x hooks/pre-receive
cd ..

git clone server.git dev && cd dev
git config user.name "Dev"; git config user.email dev@example.org
echo "Start" > a.txt && git add . && git commit -m "Start"
git push -u origin main
echo "Aenderung" >> a.txt && git commit -am "Aenderung"
git push
git switch -c feature/x
git push -u origin feature/x
git push origin --delete feature/x
```

Der erste Push auf `main` gelingt, weil er den Branch erst anlegt (der alte Wert ist nur Nullen). Der zweite wird vom Hook mit der Meldung abgelehnt, dass direkte Pushes verboten sind. Ein Push auf einen Feature-Branch ist erlaubt, doch das Löschen wird durch `receive.denyDeletes` verhindert. Plattformen bieten dieselbe Wirkung als Einstellung in der Oberfläche; der Hook zeigt, dass es sich dabei um eine serverseitige Prüfung handelt, die sich nicht durch lokale Einstellungen umgehen lässt. Eine `CODEOWNERS`-Datei sieht beispielsweise so aus:

**Listing:** Beispiel .github/CODEOWNERS

```text
# Standard: das Kernteam prueft alles
*                   @firma/kernteam
# Infrastruktur und Pipelines braucht das Plattformteam
/infra/             @firma/plattform
/.github/workflows/ @firma/plattform
# Dokumentation darf auch das Redaktionsteam freigeben
*.md                @firma/redaktion
```

Hier gewinnt bei einer Datei wie `README.md` die letzte passende Zeile: Das Redaktionsteam ist zuständig, nicht das Kernteam. Möchte man beide, müssen sie in derselben Zeile stehen.

### Typische Fehler und Stolperfallen.

Ein verbreiteter Fehler ist, allen Administrationsrechte zu geben, „damit niemand blockiert ist“. Falsch: Alle sind Admins. Richtig: wenige Administratoren, Rechte über Teams, regelmäßige Überprüfung. Wer Admin ist, kann Schutzregeln abschalten und Geheimnisse einsehen. Ein zweiter Fehler: Gemeinsam genutzte persönliche Zugänge oder Tokens für Automatisierung. Statt eines privaten Tokens einer Person verwendet man dedizierte Servicekonten oder Deploy-Keys mit minimalen Rechten, die sich auch einzeln widerrufen lassen. Dritter Fehler: Schutzregeln, die für Administratoren nicht gelten. Wenn Admins die Regeln umgehen dürfen, werden Ausnahmen schnell zum Normalfall.

### Praxisbezug.

Bei Audits fragt man typischerweise: Wer hat Schreibrechte auf Produktions-Repositories? Gibt es Personen mit Adminrechten, die sie nicht brauchen? Werden ausgeschiedene Mitarbeitende zuverlässig entfernt? Über Gruppen aus dem Identity-Provider (Modul 11) lässt sich das automatisieren. Eine halbjährliche Prüfung der Zugriffslisten ist in vielen Unternehmen Pflicht.

## 4.7 Remote-Konflikte

Von einem *Remote-Konflikt* spricht man, wenn lokaler Branch und Remote-Branch *divergieren*: Beide haben Commits, die der jeweils andere nicht kennt. Das passiert zwangsläufig in der Teamarbeit, sobald zwei Personen am selben Branch arbeiten. Es ist an sich kein Problem, sondern verlangt eine bewusste Entscheidung, wie man die beiden Linien vereint. Git bietet dafür die zwei Wege aus Modul 03: Merge (die Linien werden durch einen Merge-Commit verbunden) oder Rebase (die lokalen Commits werden auf den Stand des Remotes neu aufgesetzt).

Das Erkennen ist einfach, wenn man weiß, wo man hinschaut. Nach einem Fetch zeigt `git status -sb` in der ersten Zeile `[ahead 1, behind 1]`, wobei `ahead` die Zahl der lokalen und `behind` die der entfernten Commits angibt, die dem anderen fehlen. Steht dort nur `ahead`, kann man einfach pushen; steht dort nur `behind`, genügt ein Fast-forward-Pull. Stehen beide Zahlen da, ist die Historie divergiert. `git log --left-right --graph main...origin/main` zeigt die Commits beider Seiten, und `git diff main...origin/main` zeigt, was das Remote seit der Abzweigung getan hat.

Das folgende Beispiel erzeugt eine Divergenz mit *Konflikt*: Beide ändern dieselbe Zeile derselben Datei. Es zeigt, wie die Lösung beim Rebase aussieht, und endet mit dem Hinweis auf die Rettung, wenn jemand ein Force-Push-Unglück verursacht hat.

```bash
git init --bare -b main server.git
git clone server.git alice && git clone server.git bob
cd alice
git config user.name "Alice"; git config user.email a@example.org
echo "Preis: 10" > preis.txt
git add . && git commit -m "Preis angelegt"
git push -u origin main
cd ../bob
git config user.name "Bob"; git config user.email b@example.org
git pull
echo "Preis: 12" > preis.txt
git commit -am "Preis auf 12"
git push

cd ../alice
echo "Preis: 9" > preis.txt
git commit -am "Preis auf 9"
git fetch
git status -sb
git log --left-right --graph --oneline main...origin/main

git pull --rebase
git status -sb
```

Beim `git pull --rebase` bleibt der Rebase beim Commit „Preis auf 9“ stehen, weil beide dieselbe Zeile verändert haben. Der Status zeigt „rebase in progress“. Jetzt wird wie in Modul 03 gelöst: Datei bearbeiten (hier zum Beispiel `Preis: 11` nach Rücksprache), `git add preis.txt` und `git rebase --continue`. Danach lässt sich normal pushen. Wer keine Lösung findet, bricht mit `git rebase --abort` ab und steht wieder genau dort, wo er vor dem Pull war. Wichtig ist: Dieser Konflikt entsteht durch das parallele Arbeiten an derselben Zeile, nicht durch den gewählten Weg; mit Merge hätte derselbe Konflikt vorgelegen.

Was passiert, wenn jemand versehentlich einen Branch auf dem Remote überschreibt? Dann sind die verlorenen Commits meist noch in den lokalen Klonen aller Beteiligten vorhanden, die den Branch zuvor geholt haben. Im Reflog des Remote-Tracking-Branches (`git reflog show origin/main`) steht der frühere Stand; mit `git push origin <hash>:refs/heads/main --force-with-lease` lässt sich die Spitze wiederherstellen. Das Reflog (Modul 06) ist das wichtigste Rettungswerkzeug und der Grund, weshalb Force-Pushes selten zum endgültigen Datenverlust führen, solange irgendwo ein Klon den alten Stand kennt.

### Typische Fehler und Stolperfallen.

Falsch ist es, bei einer Divergenz „einfach nochmal“ `git pull` zu wiederholen, ohne die Meldung zu lesen, oder bei Konflikten wahllos `--ours` oder `--theirs` zu wählen. Beachten Sie außerdem: Beim Rebase sind die Bedeutungen von „ours“ und „theirs“ vertauscht. „ours“ bezeichnet den Stand, auf den man rebased (die Commits der anderen), „theirs“ den eigenen Commit, der gerade angewendet wird. Wer das nicht weiß, übernimmt versehentlich die falsche Seite. Richtig ist, die Marker zu lesen und bewusst zu entscheiden, oder ein Merge-Werkzeug zu benutzen, das beide Seiten beschriftet anzeigt.

### Praxisbezug.

Divergenzen lassen sich verringern, indem man kleine Pull Requests, kurzlebige Branches und häufiges Aktualisieren kombiniert. Auf dem gemeinsamen Hauptbranch sollte es nie Divergenz geben, weil dort nur über geschützte Merges geschrieben wird. Bei geteilten Feature-Branches vereinbart man klare Absprachen, wer wann pusht, oder vermeidet sie, indem jede Person ihren eigenen Branch hat.

## 4.8 Kollaborationsregeln

Technische Werkzeuge allein machen noch keine gute Zusammenarbeit. Ein Team braucht geteilte Regeln, damit alle wissen, was von einem Pull Request erwartet wird, wer wofür verantwortlich ist und wie schnell Rückmeldungen kommen. Ein zentrales Instrument ist die *Definition of Done* (DoD) für Pull Requests, also eine kurze, überprüfbare Liste von Bedingungen, die erfüllt sein müssen, bevor eine Änderung als fertig gilt. Typische Punkte: Die Pipeline ist grün, Tests für neues Verhalten sind ergänzt, die Dokumentation ist angepasst, keine Geheimnisse sind im Diff, mindestens eine Freigabe liegt vor, und die Änderung ist mit einem Ticket verknüpft. Die DoD gehört in das Repository, damit sie versioniert ist und sich über Pull Requests ändern lässt.

Zu den Regeln gehören außerdem Zuständigkeiten (*Ownership*): Für jeden Teil des Codes sollte es erkennbare Verantwortliche geben, entweder über `CODEOWNERS` (Abschnitt 4.6) oder über Absprachen. Dazu kommt das Thema *Review-Latenz*, also die Zeit, die ein Pull Request auf Rückmeldung wartet. Lange Wartezeiten verleiten dazu, große Pakete zu schnüren und Prüfungen oberflächlich zu halten. Teams, die ihre Latenz messen (zum Beispiel die Zeit bis zur ersten Reaktion und bis zum Merge) und kleine Pull Requests bevorzugen, merken schnell eine Verbesserung. Hilfreich sind verbindliche Zeitfenster für Reviews, Vertretungsregeln und eine Obergrenze für gleichzeitig offene Pull Requests je Person.

Das Beispiel legt ein Pull-Request-Template an, das die DoD als Checkliste enthält, und ein kleines Skript, das vor dem Öffnen eines Pull Requests Kennzahlen zur Größe der Änderung ausgibt. Das Template wird von GitHub aus dem Verzeichnis `.github` automatisch in neue Pull Requests geladen; GitLab kennt ähnliche Vorlagen im Verzeichnis `.gitlab`.

```bash
git init -b main pr-regeln && cd pr-regeln
git config user.name "Dev"; git config user.email dev@example.org
mkdir -p .github
cat > .github/pull_request_template.md <<'EOF'
## Ziel
Warum ist die Aenderung noetig? (Ticket: PROJ-...)

## Aenderungen
- ...

## Testen
Wie kann ein Reviewer die Aenderung pruefen?

## Definition of Done
- [ ] Pipeline ist gruen
- [ ] Tests fuer neues Verhalten ergaenzt
- [ ] Dokumentation angepasst
- [ ] Keine Geheimnisse im Diff
- [ ] Mindestens eine Freigabe
EOF
git add . && git commit -m "PR-Template mit Definition of Done"

cat > pr-groesse.sh <<'EOF'
#!/bin/sh
# Nutzung: sh pr-groesse.sh [Basis-Branch]
base="${1:-main}"
echo "Commits:  $(git rev-list --count $base..HEAD)"
git diff --shortstat $base...HEAD
lines=$(git diff --numstat $base...HEAD | awk '{s+=$1+$2} END {print s+0}')
if [ "$lines" -gt 400 ]; then
  echo "WARNUNG: $lines geaenderte Zeilen - Pull Request besser aufteilen."
else
  echo "Groesse ok: $lines geaenderte Zeilen."
fi
EOF
git add . && git commit -m "Skript zur PR-Groesse"

git switch -c feature/demo
seq 1 500 > daten.txt
git add . && git commit -m "Grosse Datei"
sh pr-groesse.sh main
```

Das Template liegt in `.github/pull_request_template.md`. Es gliedert den Pull Request in Ziel, Änderungen, Testhinweise und die Checkliste der Definition of Done. Das Skript nutzt die Drei-Punkte-Schreibweise für das Diff gegen die Abzweigung, zählt Commits mit `git rev-list --count` und summiert hinzugefügte und gelöschte Zeilen aus der `--numstat`-Ausgabe. Die Schwelle von 400 Zeilen ist eine Richtgröße, die jedes Team selbst festlegt; wichtig ist, dass sie als Gesprächsanlass dient und nicht als starre Regel. Das Skript lässt sich später in eine Pipeline einbauen (Modul 09).

### Typische Fehler und Stolperfallen.

Falsch ist, Regeln nur mündlich weiterzugeben oder in einem Wiki zu verstecken, das niemand liest. Neue Mitarbeitende kennen sie nicht, und alte vergessen sie. Richtig ist, sie als Dateien (`CONTRIBUTING.md`, Pull-Request-Template, `CODEOWNERS`) im Repository zu führen und möglichst automatisch zu prüfen. Ein weiterer Fehler ist, die DoD zu überladen: Eine Liste mit 25 Punkten wird abgehakt, ohne gelesen zu werden. Wenige, klare, prüfbare Punkte wirken besser, und alles Mechanische gehört in die Pipeline. Schließlich: Ownership bedeutet nicht Alleinbesitz. Wer für einen Bereich zuständig ist, sorgt für Qualität und Wissensverteilung, blockiert aber nicht aus Prinzip.

### Praxisbezug.

Kollaborationsregeln zeigen ihren Wert beim Onboarding und in Stresssituationen. Neue Kolleginnen und Kollegen können ihren ersten Pull Request ohne viel Rückfragen stellen, und bei einem Incident ist klar, wer freigeben darf. Viele Unternehmen messen außerdem Kennzahlen wie Durchlaufzeit von Pull Requests, um Engpässe zu finden; entscheidend ist dabei, sie zur Verbesserung der Abläufe zu verwenden und nicht zur Bewertung einzelner Personen.

## Übungen zu Modul 04

**Aufgabe 1.** Was ist `origin`, und was ist `origin/main`? Warum kann `git status` „up to date“ melden, obwohl der Server neuere Commits hat?

**Aufgabe 2.** Erklären Sie den Unterschied zwischen `git fetch` und `git pull`. Warum ist Fetch immer gefahrlos?

**Aufgabe 3.** Warum lehnt ein Server einen Push mit `rejected (fetch first)` ab, und welche zwei Wege gibt es, die Lage zu bereinigen?

**Aufgabe 4.** Nennen Sie zwei Vorteile von SSH und zwei von HTTPS für den Zugriff auf Remotes. Warum gehört ein Token nie in die Remote-URL?

**Aufgabe 5.** Praktisch: Bauen Sie lokal ein Bare-Repository und zwei Klone nach. Erzeugen Sie eine Ablehnung beim Push, und lösen Sie sie einmal mit `git pull --rebase` und einmal mit `git pull --no-rebase`. Vergleichen Sie die Graphen.

**Aufgabe 6.** Praktisch: Richten Sie ein Bare-Repository mit `receive.denyNonFastForwards true` ein. Zeigen Sie, dass `git push --force` abgelehnt wird, nachdem Sie die Historie des Branches umgeschrieben haben. Erklären Sie, was das für die Zusammenarbeit bedeutet.

**Aufgabe 7.** Praktisch: Simulieren Sie einen Forking-Workflow mit `original.git`, `fork.git` und einem Klon mit Remote `upstream`. Bringen Sie einen Beitrag in einem Branch unter, lassen Sie das Original weiterlaufen und synchronisieren Sie den Fork mit `--ff-only`.

**Aufgabe 8.** Praktisch (Transfer): Erzeugen Sie eine Commit-Serie, bauen Sie eine Korrektur per `git commit --fixup` ein und vergleichen Sie vorher/nachher mit `git range-diff`. Schreiben Sie dazu ein Pull-Request-Template mit Definition of Done und ein Skript, das die Größe der Änderung prüft.

## Lösungen zu Modul 04

**Lösung 1.** `origin` ist der Kurzname eines Remotes (meist des Repositories, von dem geklont wurde). `origin/main` ist ein Remote-Tracking-Branch: der lokal gespeicherte, zuletzt geholte Stand von `main` auf dem Remote. Er wird nur bei Fetch/Pull aktualisiert. `git status` vergleicht nur mit diesem lokalen Merkzettel und kennt neuere Server-Commits daher nicht.

**Lösung 2.** `fetch` lädt Objekte und aktualisiert Remote-Tracking-Branches, verändert aber weder lokale Branches noch Arbeitsverzeichnis. `pull` ist Fetch plus Integration (Merge oder Rebase) und kann daher Konflikte erzeugen. Fetch ist gefahrlos, weil es nur hinzufügt und nichts überschreibt.

**Lösung 3.** Der Push wäre kein Fast-forward, weil das Remote Commits enthält, die lokal fehlen. Wege: fremde Änderungen holen und per Merge integrieren (`git pull --no-rebase`) oder eigene Commits darauf neu aufsetzen (`git pull --rebase`); danach normal pushen. Nicht: `--force`.

**Lösung 4.** SSH: keine Token-Verwaltung, Schlüssel bleibt lokal; bequem bei häufigem Zugriff. HTTPS: funktioniert hinter Firewalls, die nur Webverkehr erlauben; Tokens lassen sich in der Plattform feingranular und befristet vergeben. Ein Token in der URL steht im Klartext in `.git/config`, taucht in Logs und Screenshots auf und lässt sich leicht versehentlich teilen.

**Lösung 5.** Aufbau wie im Beispiel in Abschnitt 4.2. Mit Rebase entsteht eine gerade Linie; mit `--no-rebase` ein Merge-Commit mit zwei Eltern, der Graph zeigt eine Raute. Beide enthalten am Ende dieselben Dateien.

**Lösung 6.** Auf dem Server `git config receive.denyNonFastForwards true`. Lokal `git commit --amend` auf einem gepushten Commit, dann `git push --force`: der Server lehnt ab (`remote rejected`). Folge: Die Historie des geschützten Branches lässt sich nicht umschreiben; Korrekturen müssen als neue Commits erfolgen (`git revert`).

**Lösung 7.** Ablauf wie in Abschnitt 4.5. Kernpunkte: `git remote add upstream`, Beitrag auf eigenem Branch pushen, `git fetch upstream`, `git merge --ff-only upstream/main`, `git push origin main`. Schlägt `--ff-only` fehl, hat der lokale `main` eigene Commits, die dort nicht hingehören.

**Lösung 8.** Ablauf wie in Abschnitt 4.4 und 4.8. Das Template enthält Ziel, Änderungen, Testhinweise, DoD-Checkliste; das Skript nutzt `git diff --numstat base...HEAD`. Im `range-diff` ist der Fixup-Commit dem verschmolzenen neuen Commit zugeordnet und als geändert (`!`) markiert, der ursprüngliche erste Commit ist entfallen (`<`), der zweite Commit gleich (`=`).
