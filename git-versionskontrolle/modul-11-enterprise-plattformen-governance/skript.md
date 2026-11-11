# Modul 11 - Enterprise Git, Plattformen und Governance

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Git und Versionskontrolle“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (11.1 bis 11.8), danach folgen „Übungen zu Modul 11“ (8 Aufgaben) und „Lösungen zu Modul 11“. Listings mit Überschrift in der Quelle tragen davor eine Zeile „Listing: …“. **Prüfstand der Beispiele für dieses Repository:** Die Shell-Beispiele (`bash`-Codeblöcke) wurden mit `bash -n` auf Syntax geprüft und in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt (Einzelheiten in der README); die ausgegebenen Texte wurden nicht mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt. Viele Beispiele ändern Repositories, Konfiguration oder Verlauf: nur in einem leeren Übungsordner ausprobieren. Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h | **Priorität:** 2

> Die Modulübersicht und die Summe der Lerneinheiten stimmen überein (acht Einheiten zu je 1,25 h). Prioritäten der Einheiten: 11.1 bis 11.6 Prio 2, 11.7 und 11.8 Prio 3.

**Lernziele des Moduls:** GitHub, GitLab, Azure Repos und Enterprise-Standards verwalten; Git-Workflows, Security und Betriebspraxis einordnen; Versionierungsprozesse für Teams und CI/CD produktionsnah umsetzen. Git selbst ist ein Werkzeug ohne Benutzerverwaltung, ohne Berechtigungen und ohne Pull Requests. Alles, was ein Unternehmen im Alltag mit Git verbindet, also Organisationen, Rollen, Freigaben, Single Sign-on, Pipelines und Richtlinien, stammt von der Plattform, auf der die Repositories liegen. Dieses Modul erklärt die gemeinsamen Konzepte der großen Plattformen, zeigt die Unterschiede und übt, Regeln so zu formulieren, dass man sie prüfen und automatisieren kann.

| **Nr.** | **Lerneinheit**             | **Dauer (h)** | **Prio** |
|:--------|:----------------------------|--------------:|:--------:|
| 11.1    | GitHub Enterprise           |          1,25 |    2     |
| 11.2    | GitLab Enterprise           |          1,25 |    2     |
| 11.3    | Azure Repos                 |          1,25 |    2     |
| 11.4    | Jenkins und Git Integration |          1,25 |    2     |
| 11.5    | Identity und SSO            |          1,25 |    2     |
| 11.6    | Enterprise Policies         |          1,25 |    2     |
| 11.7    | Developer Experience        |          1,25 |    3     |
| 11.8    | Plattformvergleich          |          1,25 |    3     |
|         | **Summe**                   |      **10,0** |          |

> **Hinweis zu den Beispielen und zur Aktualität**
>
> Plattformen ändern Funktionen, Tarife und Oberflächen laufend. Die Aussagen zu den Plattformen in diesem Modul stützen sich auf die jeweils am 28.10.2026 abgerufene Dokumentation und sind im Quellenverzeichnis aufgeführt; Preise werden bewusst nicht genannt. Konfigurationsdateien der Plattformen (YAML, JSON, Groovy) lassen sich ohne die Plattform nicht ausführen und sind deshalb nur auf Syntax geprüft, das steht jeweils dabei. Was Git betrifft (Hooks, Push-Optionen, Vorlagen, Prüfskripte), wurde in einer Testumgebung mit Git 2.43 tatsächlich ausgeführt.

## 11.1 GitHub Enterprise

GitHub organisiert Zusammenarbeit in mehreren Ebenen. Auf der untersten liegt das *Repository*. Darüber steht die *Organization*, die Repositories, Mitglieder und Teams bündelt und Einstellungen für alle Repositories gemeinsam trägt. Ein *Team* ist eine Gruppe von Mitgliedern, der man Rollen auf Repositories zuweist, sodass man Zugriffe nicht an Einzelpersonen, sondern an Gruppen bindet. Bei GitHub Enterprise Cloud kommt darüber noch das *Enterprise-Konto* hinzu, das mehrere Organisationen zusammenfasst und Richtlinien, Abrechnung und Identitätsverwaltung zentral steuert. Dieses Muster, Einzelperson in Team, Team in Organisation, Organisation in Enterprise, findet man in allen großen Plattformen wieder (bei GitLab Gruppen und Untergruppen, bei Azure DevOps Organisation, Projekt und Team), und es hat einen praktischen Zweck: Regeln werden einmal auf der richtigen Ebene festgelegt und wirken dann für alle darunter liegenden Repositories.

Das zentrale Werkzeug für solche Regeln sind *Rulesets*. Laut der Dokumentation (abgerufen am 28.10.2026) sind Rulesets benannte Regelsammlungen, die steuern, wie Personen mit Branches und Tags interagieren dürfen, etwa durch signierte Commits, durch das Blockieren von Force-Pushes oder das Verhindern des Löschens von Tags. Gegenüber der älteren Branch Protection haben sie mehrere Vorteile. Auf einen Branch wirkt immer nur eine Branch-Protection-Regel, aber mehrere Rulesets gleichzeitig; der Enforcement-Status lässt sich ändern, ohne das Ruleset zu löschen; jede Person mit Lesezugriff kann die aktiven Rulesets sehen; und Rulesets können zusätzlich Commit-Metadaten wie Nachrichten oder Autoren-Adressen regeln. Treffen mehrere Rulesets und Branch-Protection-Regeln auf denselben Branch zu, werden sie zusammengeführt, und bei Überschneidungen gilt die strengste Variante. Ein Ruleset hat keine Priorität.

Zwei Eigenschaften machen Rulesets für Unternehmen wertvoll. Erstens der *Enforcement-Status*: „Active“ erzwingt die Regeln sofort, „Evaluate“ setzt sie nicht durch, protokolliert aber auf der Seite „Rule Insights“, welche Aktionen gegen sie verstoßen würden, und „Disabled“ schaltet alles ab. Der Modus Evaluate ist der sichere Weg, eine neue Regel einzuführen: Man beobachtet eine Woche, wen sie getroffen hätte, und erst dann schaltet man sie scharf. Zweitens die *Bypass-Liste*: Man legt fest, welche Rollen, Teams oder GitHub Apps die Regeln umgehen dürfen, zum Beispiel ein Release-Bot (Modul 09, Abschnitt 9.7). Die Ebenen sind Repository, Organisation und Enterprise; organisationsweite Rulesets setzen laut Dokumentation einen Team- oder Enterprise-Tarif voraus, Push-Rulesets, die nach Dateipfaden, -endungen und -größen blockieren können, den Enterprise-Cloud-Tarif. Die Dokumentation nennt Obergrenzen von je 75 Rulesets pro Repository und für die Organisation; sie werden mit der Zeit angepasst, sodass man sie in der aktuellen Fassung nachliest.

Weil ein Ruleset nichts weiter ist als eine strukturierte Beschreibung, lässt es sich als Datei im Repository ablegen und über die API der Plattform anwenden. Das folgende Beispiel zeigt ein Ruleset als JSON für den Standardbranch: zwei Reviews, Pflichtprüfung, kein Force-Push, kein Löschen, und einen Release-Bot auf der Bypass-Liste. Es wurde nur auf gültiges JSON geprüft; die Feldnamen entsprechen dem Wissensstand zur REST-API von GitHub, nicht einer abgerufenen Seite, und müssen vor dem Einsatz gegen die aktuelle API-Dokumentation geprüft werden. Der Befehl zum Anlegen, etwa `gh api -X POST /orgs/<org>/rulesets --input ruleset.json`, wurde nicht ausgeführt.

**Listing:** Ruleset als JSON (nur JSON-Syntax geprueft; Feldnamen gegen aktuelle API-Doku pruefen)

```json
{
  "name": "standard-branch-schutz",
  "target": "branch",
  "enforcement": "evaluate",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "bypass_actors": [
    { "actor_id": 12345, "actor_type": "Integration", "bypass_mode": "always" }
  ],
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request",
      "parameters": { "required_approving_review_count": 2,
                      "dismiss_stale_reviews_on_push": true,
                      "require_code_owner_review": true } },
    { "type": "required_status_checks",
      "parameters": { "required_status_checks": [ { "context": "build" } ] } }
  ]
}
```

Die Datei beginnt im Modus `evaluate`, sodass sie zunächst nur beobachtet. Die Bedingung `~DEFAULT_BRANCH` bezeichnet den Standardbranch unabhängig von seinem Namen, was bei Organisationen, in denen Repositories teils `master`, teils `main` heißen, bequem ist. Die Regeln enthalten die Dinge aus Modul 05 und Modul 09: Pull-Request-Pflicht mit zwei Freigaben, die bei neuen Pushes verfallen, Code-Owner-Review und die Pflichtprüfung „build“, deren Name wie in Modul 09 beschrieben stabil bleiben muss. Da solche JSON-Dateien versioniert werden können, entsteht aus Regeln *Konfiguration als Code*: Änderungen sind sichtbar, reviewbar und rückgängig zu machen.

Neben Rulesets gehören zur Governance auf GitHub die Sicherheitsfunktionen der Plattform, die in Modul 08 behandelt sind (Push Protection gegen Secrets, Abhängigkeitswarnungen, Signaturanzeige, Security Policy) und die Verwaltung der Actions (erlaubte Actions, Rechte des Tokens, Modul 09). Die Zuordnung einer Aufgabe zur richtigen Ebene ist dabei die eigentliche Entwurfsentscheidung: Was gilt für alle (Enterprise), was für eine Abteilung (Organisation), was für ein Projekt (Repository)?

### Typische Fehler und Stolperfallen.

Falsch ist, Regeln pro Repository von Hand zu klicken. Das skaliert nicht, ist nicht überprüfbar und führt zu Abweichungen. Richtig sind organisationsweite Rulesets plus eine Prüfung, welche Repositories ausgenommen sind. Zweiter Fehler: Neue Regeln sofort auf „Active“ zu setzen und das Team zu überraschen; erst mit Evaluate beobachten. Dritter Fehler: Die Bypass-Liste zu großzügig zu füllen; wer alles umgehen darf, hat keine Regel. Vierter Fehler: Zugriffe an Einzelpersonen statt an Teams zu vergeben; beim Ausscheiden bleibt dann ein Wildwuchs an Einzelrechten zurück (siehe 11.5).

### Praxisbezug.

Bei der Einführung von GitHub Enterprise legt man zuerst die Struktur fest (welche Organisationen, welche Teams, wer verwaltet was), danach die Basisregeln als Rulesets und erst danach die Details je Team. Wer Branch Protection aus früheren Zeiten hat, lässt beides parallel laufen, bis die Rulesets alles abdecken; die Dokumentation sagt, dass beide gleichzeitig bestehen können und alle Regeln gelten.

## 11.2 GitLab Enterprise

GitLab ist als vollständige DevOps-Plattform angelegt: Repository, Merge Requests, CI/CD, Sicherheitsfunktionen und Paketverwaltung in einem Produkt. Die Gliederung heißt dort *Gruppen* und *Projekte*. Ein Projekt ist ein Repository mit allem, was dazugehört; Gruppen können verschachtelt werden, und Einstellungen und Berechtigungen vererben sich nach unten. Wer in einer Gruppe Mitglied ist, hat in allen Projekten darunter die zugewiesene Rolle. Das macht Gruppen zum Hebel für Governance: Die oberste Gruppe eines Bereichs legt fest, was für alle gilt. Merge Requests sind GitLabs Gegenstück zu Pull Requests.

Der Schutz des Quellcodes besteht laut Dokumentation (abgerufen am 28.10.2026) aus mehreren Bausteinen, die auf der Seite zu den Branch Rules zusammengeführt sind. *Protected Branches* legen fest, wer pushen und mergen darf, ob Force-Pushes erlaubt sind und ob Änderungen an `CODEOWNERS`-Dateien direkt gepusht werden dürfen. *Approval Rules* verlangen Freigaben, wobei die Anzahl zwischen 0 (optional) und 100 (verpflichtend) liegt. *Status Checks* binden externe Dienste ein, die bestätigen, dass ein Branch Qualitätsvorgaben erfüllt. Dazu kommen Einstellungen für Squash-Merges. Zum Anlegen und Ändern der Regeln braucht man die Rolle Maintainer oder Owner; Approval Rules und Squash-Einstellungen gibt es in den Tarifen Premium und Ultimate, Status Checks nur in Ultimate. Daneben nennt die Seite „Protect your repository“ (abgerufen am 28.10.2026) *Push Rules*: Das sind serverseitige Prüfungen (Pre-receive-Hooks), die Commits, Dateien und Tags prüfen, etwa Namensregeln für Branches, das Verbot, Tags zu löschen, oder die Pflicht zu signierten Commits. Push Rules sind auf Instanz-, Gruppen- und Projektebene verfügbar, Protected Branches auf Gruppen- und Projektebene. Die Zuordnung der einzelnen Funktionen zu Tarifen ändert sich; sie ist in der Dokumentation des jeweiligen Tarifs nachzulesen.

Die CI-Konfiguration liegt in der Datei `.gitlab-ci.yml` im Repository. Die Regeln, wann eine Pipeline läuft, heißen `rules` und `workflow:rules`; sie entsprechen den Trigger-Filtern aus Modul 09, Abschnitt 9.1, und wurden dort nur über eine Drittanbieter-Quelle belegt. Das folgende Beispiel zeigt eine typische Konfiguration (nur auf YAML-Syntax geprüft, nicht auf GitLab ausgeführt), die Pipelines für Merge Requests und den Standardbranch zulässt und Doppelläufe vermeidet. Dass das auf einer Plattform gültig ist, kann nur deren Linter bestätigen; für GitLab gibt es dafür ein eigenes Prüfwerkzeug in der Oberfläche (Wissensstand).

**Listing:** .gitlab-ci.yml (nur YAML-Syntax geprueft)

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

stages: [pruefen, bauen]

pruefen:
  stage: pruefen
  script:
    - sh test.sh
  rules:
    - changes: [src/**/*, tests/**/*]

bauen:
  stage: bauen
  script:
    - sh build.sh "$CI_COMMIT_TAG"
  rules:
    - if: $CI_COMMIT_TAG
```

Die Datei lässt Pipelines nur für Merge-Request-Ereignisse, den Standardbranch und Tags zu; dadurch läuft ein Push auf einen Feature-Branch mit offenem Merge Request nicht doppelt. Der Job `pruefen` startet nur bei Änderungen in `src` oder `tests`, der Job `bauen` nur bei Tags. Die Variablen mit `CI_` sind vordefinierte Variablen der Plattform (Wissensstand, nicht aus einer abgerufenen Seite belegt).

Ein Git-Mechanismus, den GitLab nutzt und den man auch ohne die Plattform verstehen und testen kann, sind *Push-Optionen*. Mit `git push -o <option>` übergibt man dem Server zusätzliche Hinweise, die ein serverseitiger Hook auswerten kann; GitLab verwendet sie unter anderem, um bei einem Push direkt einen Merge Request anzulegen oder die Pipeline zu überspringen (Wissensstand, Einzelheiten in der GitLab-Dokumentation). Die Dokumentation zu den Hooks (abgerufen am 28.10.2026) beschreibt, dass der Server die Optionen mit `receive.advertisePushOptions` anbieten muss und dass die Hooks sie als Umgebungsvariablen `GIT_PUSH_OPTION_COUNT` und `GIT_PUSH_OPTION_0`, `GIT_PUSH_OPTION_1` und so weiter erhalten. Das folgende Beispiel baut genau diesen Weg nach: Ein `post-receive`-Hook liest die Optionen und reagiert auf `ci.skip` und `review=`.

```bash
git init -q --bare -b main server.git
git -C server.git config receive.advertisePushOptions true
cat > server.git/hooks/post-receive <<'EOF'
#!/bin/sh
# Wertet Push-Optionen aus (so macht es auch eine Plattform, nur mit mehr Funktionen)
n=${GIT_PUSH_OPTION_COUNT:-0}
i=0
while [ $i -lt $n ]; do
  eval "opt=\$GIT_PUSH_OPTION_$i"
  case "$opt" in
    ci.skip)    echo "Server: Pipeline wird uebersprungen" >&2 ;;
    review=*)   echo "Server: Review angefordert bei ${opt#review=}" >&2 ;;
    *)          echo "Server: unbekannte Option '$opt' ignoriert" >&2 ;;
  esac
  i=$((i + 1))
done
[ $n -eq 0 ] && echo "Server: keine Push-Optionen, normale Pipeline" >&2
exit 0
EOF
chmod +x server.git/hooks/post-receive
git clone -q server.git w 2>/dev/null && cd w
git config user.name Dev; git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: eins"
echo "--- Push ohne Optionen"
git push -q origin main
echo 2 >> a && git commit -q -am "docs: Text"
echo "--- Push mit zwei Optionen"
git push -q -o ci.skip -o review=team-backend origin main
echo 3 >> a && git commit -q -am "docs: mehr"
echo "--- Push mit unbekannter Option"
git push -q -o foo=bar origin main
```

Ohne Optionen meldet der Hook den normalen Fall. Mit zwei Optionen erhält er `GIT_PUSH_OPTION_COUNT=2` und die Werte `ci.skip` und `review=team-backend`; der Hook zeigt, dass beide angekommen sind. Die dritte Variante zeigt das defensive Verhalten bei unbekannten Optionen. Die Mitteilungen des Hooks erscheinen beim Pushenden mit dem Präfix „remote:“. Der Vorteil dieses Mechanismus liegt darin, dass der Entwickler beim Push Entscheidungen mitgeben kann, ohne die Weboberfläche zu öffnen.

### Typische Fehler und Stolperfallen.

Falsch ist, GitLab wie GitHub zu betrachten und Begriffe gleichzusetzen. Ein Merge Request ist kein Pull Request mit anderem Namen in jeder Hinsicht: Genehmigungen, Pflichtprüfungen und Berechtigungen sind anders aufgebaut, und die Rollen (Guest, Reporter, Developer, Maintainer, Owner) haben eigene Bedeutungen. Zweiter Fehler: Schutz nur im Projekt, nicht auf Gruppenebene einzurichten; neue Projekte starten dann ungeschützt. Dritter Fehler: `rules` und das ältere `only/except` in einer Pipeline zu mischen (Wissensstand). Vierter Fehler: Annehmen, dass jede Funktion in jedem Tarif verfügbar ist, und eine Regel zu planen, die der Tarif gar nicht bietet.

### Praxisbezug.

Bei Self-Managed-Installationen betreibt das Unternehmen GitLab selbst und trägt Updates, Backups und Sicherheit (Modul 12 behandelt Backups und Wiederanlauf). Bei GitLab.com liegt der Betrieb beim Anbieter, dafür gelten die Grenzen des Tarifs. Die Wahl ist eine Plattformentscheidung (11.8) und keine Frage des Geschmacks.

## 11.3 Azure Repos

Azure Repos ist der Git-Teil von Azure DevOps, zu dem außerdem Boards (Aufgaben und Backlogs), Pipelines (CI/CD), Test Plans und Artifacts gehören. Die Struktur lautet Organisation, Projekt, Repository; ein Projekt kann mehrere Git-Repositories enthalten. Ein Merkmal, das Azure DevOps von den anderen Plattformen unterscheidet, ist die enge Verbindung mit Boards: Pull Requests lassen sich mit Work Items verknüpfen, und Commits können sie über Nummern in der Nachricht referenzieren (Wissensstand). Das stützt Nachvollziehbarkeit, wie sie Audits verlangen, ohne dass man etwas Zusätzliches tun muss.

Der Schutz von Branches heißt in Azure Repos *Branch Policies*. Die Dokumentation (abgerufen am 28.10.2026) beschreibt sie so: Ein Branch mit erforderlichen Richtlinien kann nicht gelöscht werden, und Änderungen müssen über Pull Requests laufen. Verfügbar sind unter anderem eine Mindestzahl an Reviewern, die Prüfung auf verknüpfte Work Items, die Prüfung, dass alle Kommentare aufgelöst sind, die Beschränkung der erlaubten Merge-Typen, die *Build Validation*, Status-Prüfungen externer Dienste und automatisch hinzugefügte Reviewer für bestimmte Dateien oder Ordner. Bei den Reviewern lässt sich festlegen, ob Autoren ihre eigenen Änderungen genehmigen dürfen, und die Option, dass der Letzte, der gepusht hat, nicht selbst freigeben darf, setzt die Funktionstrennung (Segregation of Duties) durch, die Compliance-Vorgaben oft fordern. Bei neuen Pushes kann man verlangen, dass Genehmigungen zurückgesetzt werden. Die Build Validation startet eine Pipeline für jeden Pull Request, automatisch oder manuell, und kann zwingend oder optional sein; als Ablauf des Ergebnisses lässt sich einstellen, ob es sofort bei jedem Update, nach einer Anzahl Stunden oder nie verfällt. Voraussetzung ist, dass die Pipeline schon existiert. Als Merge-Typen nennt die Dokumentation Basic merge (ohne Fast-Forward), Squash merge, Rebase and fast-forward und Rebase with merge commit; standardmäßig sind alle erlaubt, die Richtlinie kann sie einschränken (vergleiche Modul 03 und 05).

Für das Umgehen der Regeln gibt es eigene Berechtigungen: Pull Requests trotz nicht erfüllter Richtlinien abzuschließen und direkt in geschützte Branches zu pushen. Microsoft warnt in der Dokumentation ausdrücklich zur Vorsicht, vor allem bei Vergabe auf Projekt- oder Repository-Ebene statt auf einem einzelnen Branch. Richtlinien lassen sich außerdem über *Pfadfilter* auf bestimmte Dateien einschränken; die Pfade beginnen mit einem Schrägstrich, Platzhalter sind erlaubt, mehrere Pfade trennt man mit Semikolon, ein Ausrufezeichen schließt aus. Zum Bearbeiten der Richtlinien braucht man die Gruppe Project Administrators oder das Recht „Edit policies“.

Die Azure Pipelines, die dazugehören, werden in einer YAML-Datei beschrieben. Das folgende Beispiel (nur YAML-Syntax geprüft, nicht auf Azure ausgeführt) zeigt Trigger für den Hauptbranch und Release-Tags sowie Pfadfilter. Danach kommt ein Git-Skript, das etwas anderes beantwortet, was in Azure-Umgebungen ständig gebraucht wird: Aus einer Remote-URL erkennt es Plattform und Protokoll. Das ist hilfreich bei Migrationen (Modul 12) und beim Prüfen, ob Zugangsdaten in URLs stecken (Modul 08).

**Listing:** azure-pipelines.yml (nur YAML-Syntax geprueft)

```yaml
trigger:
  branches:
    include: [main]
  tags:
    include: ['v*']
  paths:
    exclude: [docs/*]

pr:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

steps:
  - checkout: self
    fetchDepth: 0
  - script: sh test.sh
    displayName: Tests
```

Die Option `fetchDepth: 0` lädt die vollständige Historie und löst damit das Problem der flachen Klone aus Modul 09, Abschnitt 9.8; dass diese Schlüssel so heißen und wirken, entspricht dem Wissensstand und ist gegen die aktuelle Azure-Dokumentation zu prüfen.

```bash
cat > remote-art.sh <<'EOF'
#!/bin/sh
# Erkennt Plattform und Protokoll einer Git-Remote-URL; warnt vor Zugangsdaten in der URL
url=$1
case "$url" in
  *github.com*)                                 plattform="GitHub" ;;
  *gitlab*)                                     plattform="GitLab" ;;
  *dev.azure.com*|*visualstudio.com*)           plattform="Azure DevOps" ;;
  *)                                            plattform="unbekannt" ;;
esac
case "$url" in
  https://*@*|http://*@*)
    ziel=${url#*://}; kopf=${ziel%%@*}
    case "$kopf" in *:*) proto="https MIT PASSWORT/TOKEN IN DER URL" ;; *) proto="https (Benutzername in der URL)" ;; esac ;;
  https://*|http://*) proto="https" ;;
  git@*|ssh://*)      proto="ssh" ;;
  /*|file://*|../*)   proto="lokal" ;;
  *)                  proto="unbekannt" ;;
esac
printf "%-55s %-13s %s\n" "$url" "$plattform" "$proto"
EOF
for u in git@github.com:firma/projekt.git \
         https://github.com/firma/projekt.git \
         https://gitlab.com/gruppe/untergruppe/projekt.git \
         git@ssh.dev.azure.com:v3/firma/projekt/repo \
         https://firma@dev.azure.com/firma/projekt/_git/repo \
         https://bot:geheim@github.com/firma/projekt.git \
         /srv/git/projekt.git; do
  sh remote-art.sh "$u"
done
```

Das Skript trennt zwei Fragen. Zuerst bestimmt es anhand typischer URL-Muster die Plattform, dann das Protokoll. Der Fall mit „bot:geheim@“ zeigt, woran man URLs mit eingebetteten Zugangsdaten erkennt: Zwischen `://` und `@` steht ein Doppelpunkt. Das Muster für Azure DevOps zeigt, dass dieselbe Plattform mehrere URL-Formen hat (`dev.azure.com` in neuer, `visualstudio.com` in älterer Schreibweise sowie SSH mit Präfix `v3`); die Erkennung gilt für die gezeigten Muster und ist keine vollständige Liste (Wissensstand).

### Typische Fehler und Stolperfallen.

Falsch ist, Build Validation mit einer Pipeline zu verbinden, die ewig läuft oder oft zufällig scheitert; die Policy blockiert dann jeden Pull Request. Richtig sind schnelle, stabile Prüfungen (Modul 09, Abschnitt 9.2). Zweiter Fehler: Die Rechte zum Umgehen von Policies auf Projektebene zu vergeben, sodass sie für jedes Repository gelten; besser auf dem einzelnen Branch und nur für wenige Konten. Dritter Fehler: Pfadfilter ohne führenden Schrägstrich zu schreiben; laut Dokumentation haben sie dann keine Wirkung. Vierter Fehler: Branch Policies zu konfigurieren, aber Work-Item-Verknüpfungen nicht zu pflegen, sodass die Prüfung nur Pflichtübung bleibt.

### Praxisbezug.

Azure DevOps findet man häufig in Unternehmen, die stark auf Microsoft-Produkte setzen und Entra ID für die Anmeldung nutzen (siehe 11.5). Die Boards erleichtern dort die Verbindung von Anforderung, Commit und Release, was in regulierten Branchen als Nachweis gebraucht wird.

## 11.4 Jenkins und Git Integration

Jenkins ist ein älterer, selbst betriebener Automatisierungsserver, der in vielen Unternehmen seit Jahren läuft und mit Hunderten Plugins erweitert wurde. Anders als bei GitHub Actions oder GitLab CI gehört er zu keiner Git-Plattform; die Verbindung zum Repository stellen Plugins und Konfiguration her. Für die Modernisierung ist wichtig, zwei Dinge zu trennen: *wie Jenkins erfährt, dass es etwas zu tun gibt* (der Trigger), und *wie die Pipeline beschrieben ist* (im Repository oder in der Oberfläche).

Es gibt zwei grundsätzlich verschiedene Wege für den Trigger. Beim *Polling* fragt Jenkins in festen Abständen beim Git-Server nach, ob sich Referenzen geändert haben (in Pipelines `pollSCM` mit einem Cron-Muster). Beim *Webhook* meldet umgekehrt der Git-Server jede Änderung sofort an Jenkins. Ein Beitrag eines Drittanbieters (abgerufen am 28.10.2026) fasst die Empfehlung zusammen: Webhooks bevorzugen, weil sie sofortiges Feedback geben, Polling nur als Ausweichlösung, wenn Webhooks nicht möglich sind, mit Intervallen von mindestens 15 Minuten, und das Zeichen `H` in Cron-Ausdrücken verwenden, damit Jenkins die Last verteilt. Nachteile des Pollings sind die Verzögerung bis zur nächsten Abfrage und die ständige, oft ergebnislose Last, die bei vielen Projekten auf dem Git-Server entsteht. Das `H` steht für einen vom Jobnamen abhängigen Hash, sodass nicht alle Jobs zur gleichen Minute abfragen.

Beide Wege lassen sich mit Git nachbauen, und das macht den Unterschied anschaulich. Das Polling-Skript merkt sich den letzten bekannten Stand eines Branches und vergleicht ihn bei jeder Abfrage mit dem Ergebnis von `git ls-remote`. Der Webhook besteht aus einem `post-receive`-Hook, der nach jedem Push eine HTTP-Anfrage an einen Empfänger schickt (hier ein winziger Python-Server als Ersatz für Jenkins). Das Beispiel zeigt beide für denselben Push, sodass man sieht, dass das Polling die Änderung erst bei der nächsten Abfrage bemerkt, der Webhook sofort.

```bash
cat > empfaenger.py <<'EOF'
import http.server, sys, json
class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        n = int(self.headers.get("Content-Length", 0))
        daten = json.loads(self.rfile.read(n) or b"{}")
        with open(sys.argv[2], "a") as f:
            f.write("Jenkins erhaelt Webhook: %s -> %s\n" % (daten.get("ref"), daten.get("after")[:7]))
        self.send_response(200); self.end_headers()
    def log_message(self, *a): pass
s = http.server.HTTPServer(("127.0.0.1", 0), H)
open(sys.argv[1], "w").write(str(s.server_port))
s.serve_forever()
EOF
python3 empfaenger.py port.txt eingang.log & SRV=$!
sleep 1; P=$(cat port.txt)

git init -q --bare -b main server.git
cat > server.git/hooks/post-receive <<EOF
#!/bin/sh
# Webhook: meldet jeden Ref-Update sofort an den Empfaenger (hier: Jenkins-Ersatz)
while read alt neu ref; do
  curl -s -o /dev/null -X POST -H 'Content-Type: application/json' \\
    -d "{\\"ref\\":\\"\$ref\\",\\"after\\":\\"\$neu\\"}" http://127.0.0.1:$P/git-hook
done
EOF
chmod +x server.git/hooks/post-receive

cat > poll.sh <<'EOF'
#!/bin/sh
# Polling: vergleicht den Stand von main mit dem zuletzt gesehenen
# Aufruf: poll.sh <repo> <statusdatei>
aktuell=$(git ls-remote "$1" refs/heads/main | cut -c1-7)
bekannt=$(cat "$2" 2>/dev/null)
if [ "$aktuell" != "$bekannt" ]; then
  echo "Polling: Aenderung erkannt ($bekannt -> $aktuell), Build wird gestartet"
  echo "$aktuell" > "$2"
else
  echo "Polling: keine Aenderung"
fi
EOF

git clone -q server.git w 2>/dev/null && cd w
git config user.name Dev; git config user.email dev@example.org
echo 1 > a && git add . && git commit -q -m "feat: eins"
git push -q origin main; sleep 1
echo "--- direkt nach dem Push, Webhook-Empfaenger:"
cat ../eingang.log
echo "--- Polling-Abfrage 1 (erster Lauf):"
sh ../poll.sh ../server.git ../stand.txt
echo "--- Polling-Abfrage 2 (nichts passiert):"
sh ../poll.sh ../server.git ../stand.txt
echo 2 >> a && git commit -q -am "feat: zwei" && git push -q origin main; sleep 1
echo "--- Webhook nach zweitem Push:"; tail -n 1 ../eingang.log
echo "--- Polling-Abfrage 3:"
sh ../poll.sh ../server.git ../stand.txt
kill $SRV
```

Nach dem ersten Push steht der Webhook-Eintrag sofort im Eingang des Empfängers; das Polling erfährt davon erst bei seiner nächsten Abfrage und meldet dann die Änderung. Die zweite Abfrage ohne Änderung kostet trotzdem einen Netzwerkzugriff, im Beispiel unbedeutend, bei hundert Projekten alle fünf Minuten spürbar. Nach dem zweiten Push zeigt der Webhook wieder sofort den neuen Stand. Das Hook-Skript entsteht mit einem Heredoc ohne Anführungszeichen um `EOF`: Dadurch setzt die Shell den Port sofort ein, während die mit Backslash geschützten Variablen `$ref` und `$neu` erst bei der Ausführung des Hooks ausgewertet werden. Das ist ein häufiger Stolperstein beim Schreiben von Hook-Skripten per Heredoc.

Die zweite Entscheidung betrifft die Beschreibung der Pipeline. Eine *Multibranch Pipeline* liest ein `Jenkinsfile` aus dem Repository, findet selbst die Branches (und je nach Konfiguration Pull Requests) und legt für jeden einen eigenen Job an; verschwindet der Branch, räumt Jenkins den Job nach einer eingestellten Strategie auf. Die Pipeline lebt dann im Repository neben dem Code: versioniert, reviewbar, pro Branch verschieden. Das folgende Jenkinsfile (deklarativ, Groovy, nicht ausgeführt) zeigt eine typische Datei. Der Abschnitt `triggers` mit `pollSCM` ist nur als Ausweichlösung gedacht, wenn kein Webhook möglich ist; der Checkout in Multibranch-Jobs erfolgt automatisch.

**Listing:** Jenkinsfile, deklarativ (nicht ausgefuehrt)

```groovy
pipeline {
  agent any
  options { disableConcurrentBuilds() ; timeout(time: 20, unit: 'MINUTES') }
  triggers { pollSCM('H/15 * * * *') }   // nur Ausweichloesung; besser: Webhook
  stages {
    stage('Pruefen') { steps { sh 'sh test.sh' } }
    stage('Bauen')   {
      when { buildingTag() }
      steps { sh 'sh build.sh "$TAG_NAME"' }
    }
  }
}
```

Die Modernisierung von Legacy-CI geschieht am besten in Schritten und nicht als Urknall. Der erste Schritt ist, Webhooks statt Polling einzurichten und damit Last und Wartezeit zu senken. Der zweite ist, Pipelines aus der Oberfläche in Jenkinsfiles im Repository zu verlegen, denn erst dann sind sie versioniert und prüfbar. Der dritte besteht darin, die Logik aus den Pipelines in Skripte des Repositories zu verlagern (wie in Modul 09 empfohlen), sodass sie auf jedem System laufen. Wer dann auf eine andere Plattform wechselt, tauscht im Wesentlichen die dünne Hülle aus. Die Zugangsdaten gehören in den Jenkins-Credential-Speicher und nicht in die Job-Konfiguration (Modul 08).

### Typische Fehler und Stolperfallen.

Falsch ist, das Polling auf eine Minute zu stellen, „damit es schnell geht“. Das belastet Jenkins und den Git-Server und ersetzt keinen Webhook. Zweiter Fehler: Webhooks ohne Absicherung (Token, Netzwerkfreigabe) aus dem Internet zu erlauben. Dritter Fehler: Pipeline-Konfiguration nur in der Oberfläche zu pflegen, sodass niemand weiß, was sich wann geändert hat. Vierter Fehler: Jenkins-Plugins ungeprüft anzusammeln; jedes Plugin ist Wartungs- und Sicherheitsrisiko.

### Praxisbezug.

Viele Unternehmen betreiben Jenkins neben einer neueren Plattform. Eine sinnvolle Reihenfolge der Ablösung ist, zuerst neue Projekte auf der neuen Plattform zu starten und dann die bestehenden nach Risiko und Aufwand zu migrieren, nicht umgekehrt. Bis dahin hilft ein Standard-Jenkinsfile als Vorlage, damit die verbleibenden Projekte einheitlich sind.

## 11.5 Identity und SSO

Wenn hunderte Menschen Zugriff auf Repositories brauchen, kann man Konten nicht von Hand pflegen. Die Plattformen binden sich deshalb an das zentrale Identitätssystem des Unternehmens an, den *Identity Provider* (IdP), zum Beispiel Microsoft Entra ID oder Okta. Das bringt drei Dinge. *Single Sign-on* (SSO): Die Anmeldung geschieht beim IdP, der der Plattform bestätigt, wer man ist; dafür verwendet man SAML oder OpenID Connect (OIDC) (Wissensstand; die Spezifikationen wurden nicht abgerufen). Mit der Anmeldung gelten die Regeln des Unternehmens, etwa Mehrfaktor-Authentifizierung. *Provisionierung*: Konten werden automatisch angelegt, geändert und deaktiviert. Dafür dient der Standard SCIM (System for Cross-domain Identity Management). *Gruppen-Synchronisierung*: Mitgliedschaften in Gruppen des IdP werden in Teams der Plattform übertragen, sodass die Zuordnung von Rechten an einer einzigen Stelle gepflegt wird.

Der Standard hinter der Provisionierung ist in RFC 7643 beschrieben (abgerufen am 28.10.2026): SCIM soll die Benutzerverwaltung in Cloud-Anwendungen vereinfachen und definiert dazu ein plattformneutrales Schema im JSON-Format; das Protokoll für den Austausch über HTTP steht in RFC 7644, das hier nicht gelesen wurde. Es gibt zwei Ressourcentypen, `User` und `Group`. Wichtige Attribute des Benutzers sind `userName` (Pflicht, eindeutig, meist die Anmeldekennung), `active` (administrativer Status, `true` bedeutet in der Regel, dass die Person sich anmelden kann), `externalId` (vom Client vergebene Kennung) und `groups` (schreibgeschützt, Änderungen erfolgen über die Group-Ressource). Eine Gruppe hat ein Pflichtattribut `displayName` und eine Liste `members`. Das Deaktivieren eines Kontos geschieht in der Praxis, indem der IdP `active` auf `false` setzt oder den Benutzer entfernt.

Bei GitHub Enterprise Cloud mit Enterprise Managed Users (EMU) beschreibt die Dokumentation (abgerufen am 28.10.2026), wie das aussieht. Partner-Anwendungen gibt es für Microsoft Entra ID (OIDC und SAML), Okta (SAML) und PingFederate (SAML); andere Systeme können über die SCIM-REST-API provisionieren, bei OIDC-Enterprises ist diese API nicht unterstützt. Der IdP legt Konten an und fügt sie dem Enterprise hinzu; wird eine Gruppe der Anwendung zugewiesen, werden alle Mitglieder provisioniert. Teams, Organisationsmitgliedschaften und Repository-Zugriffe werden über IdP-Gruppen gesteuert. Bei Entfernung aus der Anwendung oder Deaktivierung im IdP werden Sitzungen beendet und das Konto wird deaktiviert; der Benutzername wird durch einen Hash ersetzt, die Daten bleiben erhalten, und bei erneuter Zuweisung wird das Konto mit dem ursprünglichen Namen reaktiviert. Die Kombination von Okta und Entra ID für SSO und SCIM wird laut Dokumentation nicht unterstützt. Verschachtelte Gruppen provisioniert Entra ID nicht.

Zu einem sauberen Lebenszyklus gehört die Kontrolle, dass wirklich niemand übrig bleibt. Auch wenn die Provisionierung automatisch läuft, gibt es Ausnahmen: Dienstkonten, die nicht im IdP stehen, Konten aus der Zeit vor der Anbindung, Fehler bei der Synchronisierung. Das folgende Beispiel zeigt eine Prüfung, die mit einfachen Mitteln arbeitet: Es vergleicht die Liste der aktiven Mitarbeitenden des IdP mit der Mitgliederliste einer Plattform und mit den Commit-Autoren der letzten Zeit. Dateien mit Listen lassen sich mit `comm` vergleichen; das Ergebnis zeigt Verwaiste (Plattform, aber nicht im IdP), Fehlende (im IdP, aber ohne Plattformzugang) und Autoren, die in beiden Listen nicht vorkommen. Zusätzlich enthält das Beispiel eine SCIM-Nachricht für das Deaktivieren eines Benutzers, die mit `jq` auf Wohlgeformtheit geprüft wird.

```bash
cat > idp.txt <<'EOF'
anna@example.org
bernd@example.org
clara@example.org
EOF
cat > plattform.txt <<'EOF'
anna@example.org
bernd@example.org
david@example.org
ci-bot@example.org
EOF
sort -o idp.txt idp.txt; sort -o plattform.txt plattform.txt

echo "=== Verwaist: Zugang auf der Plattform, aber nicht (mehr) im IdP"
comm -13 idp.txt plattform.txt
echo "=== Fehlend: im IdP, aber ohne Plattformzugang"
comm -23 idp.txt plattform.txt

echo "=== Autoren in der Historie, die nicht im IdP stehen"
git init -q -b main repo && cd repo
c() { GIT_AUTHOR_NAME="$1" GIT_AUTHOR_EMAIL="$2" GIT_COMMITTER_NAME="$1" GIT_COMMITTER_EMAIL="$2" git commit -q --allow-empty -m "$3"; }
c Anna anna@example.org "feat: a"
c David david@example.org "fix: b"
c Bot ci-bot@example.org "chore: c"
c Fremd fremd@private.example "feat: d"
git log --format='%ae' | sort -u > ../autoren.txt
cd ..
comm -13 idp.txt autoren.txt

echo "=== SCIM-Nachricht: Benutzer deaktivieren (RFC 7644 Patch, Form laut Wissensstand)"
cat > deaktivieren.json <<'EOF'
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [ { "op": "replace", "path": "active", "value": false } ]
}
EOF
jq -r '.Operations[0] | "\(.op) \(.path) = \(.value)"' deaktivieren.json
```

Die erste Liste zeigt David und das CI-Konto als Konten, die auf der Plattform Zugriff haben, aber nicht im IdP stehen. David ist ein Fall für Nachfrage (ausgeschieden und nicht deaktiviert?), das Bot-Konto ist erwartet und gehört in eine dokumentierte Ausnahmeliste. Die zweite Liste zeigt Clara, die laut IdP Zugang haben sollte, aber auf der Plattform fehlt, was typisch für Synchronisierungsfehler ist. Der Autorenvergleich nennt dieselben beiden Konten und zusätzlich eine Adresse, die zu keiner Person des Unternehmens gehört; in einem Unternehmens-Repository kann das ein Hinweis auf private Konten oder Fremdbeiträge sein. Das SCIM-Beispiel zeigt die Form einer Patch-Nachricht, die den Status auf `false` setzt; die Schreibweise folgt dem Wissensstand zum SCIM-Protokoll und wurde nur als JSON geprüft.

### Typische Fehler und Stolperfallen.

Falsch ist, SSO einzurichten, aber persönliche Zugriffstoken und SSH-Schlüssel ehemaliger Mitarbeiter weiter gültig zu lassen. Tokens und Schlüssel müssen mit dem Konto verschwinden; die Plattformen bieten dafür Kontrollen, die man regelmäßig prüft. Zweiter Fehler: Rechte an Einzelpersonen statt an IdP-Gruppen zu vergeben; Gruppen-Synchronisierung hilft nur, wenn die Rechte tatsächlich an Gruppen hängen. Dritter Fehler: Dienstkonten außerhalb des IdP ohne Besitzer und Ablaufdatum zu führen. Vierter Fehler: Auf eine Kombination verschiedener IdPs für Anmeldung und Provisionierung zu setzen, die die Plattform nicht unterstützt; die GitHub-Dokumentation nennt das ausdrücklich.

### Praxisbezug.

Die Einführung von SSO ist meist ein Projekt mit der IT-Sicherheit. Für Git-Teams sind zwei Punkte entscheidend: Wie melden sich Skripte und Pipelines an (Service-Konten und Tokens, Modul 09), und wie wird das Ausscheiden eines Mitarbeiters nachweislich umgesetzt? Für beides gehört eine Checkliste ins Runbook (Modul 10, Abschnitt 10.8).

## 11.6 Enterprise Policies

Eine *Policy* ist eine Regel, die für viele Repositories gleichzeitig gilt und nicht von den Beteiligten selbst verändert werden kann: Der Standardbranch ist geschützt, Reviews sind Pflicht, Secrets werden beim Push erkannt, Actions kommen nur aus erlaubten Quellen. Auf den Plattformen gibt es dafür die in 11.1 bis 11.3 beschriebenen Mechanismen. Die schwierige Frage ist weniger die Technik als der Inhalt: Welche Regeln sind sinnvoll, wie streng, und was passiert mit Ausnahmen? Zu strenge Regeln werden umgangen, zu lasche sind wirkungslos. Gut funktioniert ein Dreischritt: wenige verbindliche Basisregeln für alle, zusätzliche Regeln für Repositories mit hohem Schutzbedarf und ein kontrollierter Weg für Ausnahmen.

Die Basisregeln eines typischen Unternehmens betreffen drei Bereiche. Bei *Branches*: Standardbranch geschützt, Änderungen nur über Pull Requests, kein Force-Push, kein Löschen (Modul 05, 08). Bei *Secrets*: Push-Schutz gegen Zugangsdaten und Rotationspflicht bei Fund (Modul 08). Bei *Reviews*: mindestens eine, bei sensiblen Bereichen zwei Freigaben, Code Owner für kritische Verzeichnisse, Funktionstrennung (wer pusht, gibt nicht selbst frei). Dazu kommen Dateien, die jedes Repository enthalten soll: `README`, `SECURITY.md` (Modul 08, Abschnitt 8.9), `CODEOWNERS`, eine Lizenzdatei und gegebenenfalls Vorlagen für Pull Requests.

Ausnahmen sind unvermeidlich, gefährlich werden sie, wenn sie niemand mehr überblickt. Deshalb gehört zu jeder Ausnahme eine *Begründung*, ein *Verantwortlicher* und ein *Ablaufdatum*, nach dem sie neu beantragt werden muss. Das lässt sich in einer einfachen Datei führen, die man prüfen kann. Das folgende Skript prüft eine Reihe von Repositories gegen eine Richtlinie: Es verlangt `README.md`, `SECURITY.md`, `CODEOWNERS` sowie einen Standardbranch namens `main`, liest die Ausnahmen aus einer CSV-Datei und meldet Verstöße, gültige Ausnahmen und abgelaufene Ausnahmen getrennt. Die Idee heißt *Policy as Code*: Die Regeln stehen nicht in einem Dokument, das jemand liest oder nicht, sondern in einem Skript, das jede Woche läuft.

```bash
mkdir repos && cd repos
mk() { git init -q -b "$2" "$1" && (cd "$1" && git config user.name Dev && git config user.email dev@example.org && shift 2 && for f in "$@"; do mkdir -p "$(dirname "$f")"; echo x > "$f"; done && git add -A && git commit -q -m "feat: Start" 2>/dev/null || git commit -q --allow-empty -m "feat: Start"); }
mk alpha main README.md SECURITY.md .github/CODEOWNERS
mk beta main README.md
mk gamma master README.md SECURITY.md .github/CODEOWNERS
mk delta main README.md .github/CODEOWNERS
cd ..

cat > ausnahmen.csv <<'EOF'
repo;regel;grund;verantwortlich;gueltig_bis
beta;SECURITY.md;Legacy-Projekt wird abgeloest;a.meier;2027-03-31
gamma;branch=main;Umbenennung geplant;b.huber;2026-09-30
EOF

cat > policy-check.sh <<'EOF'
#!/bin/sh
# Prueft Repositories gegen die Basisrichtlinie. Aufruf: policy-check.sh <Verzeichnis> <Ausnahmen.csv> <Stichtag>
dir=$1; ausn=$2; stichtag=$3; status=0
ausnahme() {  # ausnahme <repo> <regel> -> gibt gueltig/abgelaufen/keine aus
  zeile=$(grep "^$1;$2;" "$ausn" 2>/dev/null | head -n 1)
  [ -z "$zeile" ] && { echo keine; return; }
  bis=$(echo "$zeile" | cut -d';' -f5)
  if [ "$bis" \< "$stichtag" ]; then echo abgelaufen; else echo gueltig; fi
}
pruefe() {  # pruefe <repo> <regel> <ok:0/1>
  if [ "$3" -eq 0 ]; then return; fi
  case $(ausnahme "$1" "$2") in
    gueltig)    echo "AUSNAHME   $1: $2 (gueltig)" ;;
    abgelaufen) echo "ABGELAUFEN $1: $2 (Ausnahme nicht mehr gueltig)"; status=1 ;;
    keine)      echo "VERSTOSS   $1: $2"; status=1 ;;
  esac
}
for r in "$dir"/*/; do
  name=$(basename "$r")
  for datei in README.md SECURITY.md; do
    [ -f "$r/$datei" ]; pruefe "$name" "$datei" $?
  done
  [ -f "$r/.github/CODEOWNERS" ] || [ -f "$r/CODEOWNERS" ]; pruefe "$name" "CODEOWNERS" $?
  [ "$(git -C "$r" symbolic-ref --short HEAD)" = "main" ]; pruefe "$name" "branch=main" $?
done
exit $status
EOF
sh policy-check.sh repos ausnahmen.csv 2026-10-28; echo "Exit: $?"
```

Die Funktion `pruefe` erhält das Ergebnis des vorherigen Tests (Exit-Code 0 heißt erfüllt) und entscheidet, ob eine Verletzung vorliegt. Das Repository `alpha` erfüllt alles. Bei `beta` fehlen `SECURITY.md` und `CODEOWNERS`; für `SECURITY.md` gibt es eine gültige Ausnahme, für `CODEOWNERS` keine, daher ein Verstoß. `gamma` hat den Standardbranch `master`, die Ausnahme dafür ist aber seit September abgelaufen und wird als solche gemeldet. `delta` hat keine `SECURITY.md` und keine Ausnahme. Der Exit-Code 1 macht das Skript pipelinefähig. Das Datum wird als Text verglichen, was bei der Schreibweise JJJJ-MM-TT korrekt sortiert.

Die Zuständigkeiten für Reviews lassen sich ebenfalls als Datei beschreiben. Die `CODEOWNERS`-Datei bildet Pfadmuster auf Verantwortliche ab; auf GitHub und GitLab erzeugt sie die automatische Review-Anforderung (Modul 04 und 05). Die Regel „der letzte passende Eintrag gilt“ ist dabei eine Eigenheit, die man kennen muss, weil sie bei Überschneidungen überraschen kann (Wissensstand, laut Modul 05 in der GitHub-Dokumentation beschrieben).

> **Ergänzung aus der Nachrecherche**
>
> **Einführung von Regeln ohne Unfrieden.** Der Evaluate-Modus von GitHub-Rulesets (Abschnitt 11.1) ist ein Muster, das man auf jede Plattform übertragen kann: Neue Regeln erst messen, dann ankündigen, dann durchsetzen. Wo die Plattform keinen Evaluate-Modus bietet, genügt ein Prüfskript wie oben, das zunächst nur berichtet und erst später die Pipeline fehlschlagen lässt.

### Typische Fehler und Stolperfallen.

Falsch ist, Richtlinien als Dokument zu verfassen und die Einhaltung dem guten Willen zu überlassen. Richtig sind Prüfungen, die automatisch laufen. Zweiter Fehler: Ausnahmen ohne Ablaufdatum; sie werden zum Dauerzustand. Dritter Fehler: Richtlinien ohne Besitzer; jemand muss sie pflegen und Fragen beantworten. Vierter Fehler: Ein Regelwerk auf einen Schlag einzuführen, ohne Pilot und Kommunikation; die Folge sind Umgehungen und Ablehnung.

### Praxisbezug.

Governance heißt nicht, Teams zu kontrollieren, sondern ihnen verlässliche Leitplanken zu geben. Gut gewählte Regeln sparen Diskussionen in Reviews und schaffen im Audit Nachweise ohne Zusatzaufwand. Das Prüfskript liefert Zahlen für die Governance-Roadmap in Modul 12.

## 11.7 Developer Experience

Die beste Richtlinie bleibt wirkungslos, wenn ihre Einhaltung mühsam ist. *Developer Experience* meint, dass der richtige Weg zugleich der einfachste ist: Neue Repositories entstehen aus Vorlagen, die schon alle Pflichtdateien enthalten; neue Teammitglieder sind nach einer Stunde statt nach einer Woche arbeitsfähig; Regeln werden als Hilfe erlebt, nicht als Hindernis. Die Maßnahmen dafür sind meist unspektakulär, aber wirksam.

Auf der Ebene von Git selbst bieten sich mehrere Hebel. Mit `git init --template=<Verzeichnis>` oder der Einstellung `init.templateDir` kopiert Git beim Anlegen eines Repositories die Dateien einer Vorlage in das Verzeichnis `.git` des neuen Repositories, also vor allem Hooks und die Datei `info/exclude`, nicht in das Arbeitsverzeichnis. Die Pflichtdateien eines Projekts (README, CODEOWNERS) kommen dagegen aus einem Vorlagen-Repository der Plattform. Mit `commit.template` legt man eine Textvorlage für Commit-Nachrichten fest, die den Editor beim Committen vorbefüllt; sie erinnert an die Konvention (Modul 02, 09), ohne dass jemand sie nachschlagen muss. Beide Einstellungen gehören in die globale Konfiguration der Person oder in ein Onboarding-Skript. Plattformseitig kommen Vorlagen-Repositories hinzu (bei GitHub „Template Repository“, Wissensstand), Vorlagen für Issues und Pull Requests und zentral gepflegte Workflow-Bausteine.

Das Beispiel zeigt ein Onboarding-Skript, das mit einem Aufruf alles einrichtet, was ein neues Teammitglied braucht: Name und E-Mail, Commit-Vorlage, Standardverhalten für Pull und Push, ein sinnvolles Verhalten bei Zeilenenden und eine Git-Vorlage für neue Repositories mit Hook und Ignorier-Regeln. Es ist so geschrieben, dass es mehrfach aufgerufen werden kann, ohne etwas zu verdoppeln (idempotent), und dass es anzeigt, was es getan hat. Danach legt das Beispiel ein neues Repository aus der Vorlage an und zeigt, dass Pflichtdateien und Hook vorhanden sind und die Commit-Vorlage greift.

```bash
mkdir -p firma-vorlage/hooks firma-vorlage/info
printf '# Firmenweit ignoriert\n.idea/\n*.swp\n' > firma-vorlage/info/exclude
cat > firma-vorlage/hooks/commit-msg <<'EOF'
#!/bin/sh
erste=$(head -n 1 "$1")
echo "$erste" | grep -Eq '^(feat|fix|docs|chore|refactor|test)(\([a-z0-9-]+\))?!?: .{3,}' || {
  echo "Betreff muss 'typ: text' sein, z.B. 'feat: Suche hinzugefuegt'" >&2; exit 1; }
EOF
chmod +x firma-vorlage/hooks/commit-msg
cat > commit-vorlage.txt <<'EOF'

# typ(bereich): kurze Beschreibung (max. 72 Zeichen)
#
# Typen: feat, fix, docs, chore, refactor, test
# Warum wurde die Aenderung gemacht? Was war vorher, was ist jetzt?
EOF

cat > onboard.sh <<'EOF'
#!/bin/sh
# Richtet Git fuer neue Teammitglieder ein. Aufruf: onboard.sh "Name" "mail@firma.example"
set -e
name=$1; mail=$2
git config --global user.name "$name"
git config --global user.email "$mail"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global push.autoSetupRemote true
git config --global core.autocrlf input
git config --global commit.template "$PWD/commit-vorlage.txt"
git config --global init.templateDir "$PWD/firma-vorlage"
echo "Eingerichtet fuer $name <$mail>:"
git config --global --list | grep -E '^(user|init|pull|push|core.autocrlf|commit)' | sed 's/^/  /'
EOF
sh onboard.sh "Neue Kollegin" "neu@firma.example"
echo "--- zweiter Aufruf (idempotent):"
sh onboard.sh "Neue Kollegin" "neu@firma.example" | head -1
echo "--- neues Repository aus der Vorlage"
git init -q projekt-neu && cd projekt-neu
ls .git/hooks | grep -E "^commit-msg$"; tail -n 3 .git/info/exclude
echo x > a && git add a
git commit -q -m "irgendwas"; echo "Exit: $?"
git commit -q -m "feat: erstes Feature"; echo "Exit: $?"
```

Das Skript setzt alle Einstellungen mit `git config --global` und zeigt sie an. Weil `git config` Werte setzt statt anzuhängen, ändert ein zweiter Aufruf nichts. Beim Anlegen des neuen Repositories kopiert Git den Inhalt der Vorlage in `.git`: Der Hook `commit-msg` liegt in `.git/hooks`, und die Ignorier-Regeln stehen in `.git/info/exclude`; das Arbeitsverzeichnis bleibt leer. Der Commit mit freier Nachricht wird durch den Hook abgelehnt, der Commit mit passendem Betreff gelingt. Die Vorlage für Commit-Nachrichten (`commit.template`) wirkt nur, wenn man ohne `-m` committet und der Editor sich öffnet; sie lässt sich im Skript deshalb nur als Einstellung zeigen. Beim Vorgehen mit `init.templateDir` kopiert Git die Dateien nur beim Anlegen eines neuen Repositories; bereits vorhandene Repositories erhalten sie mit `git init` im bestehenden Verzeichnis erneut (Git überschreibt vorhandene Dateien dabei nicht).

### Typische Fehler und Stolperfallen.

Falsch ist, jedes Team seine Vorlagen selbst erfinden zu lassen. Das ergibt Wildwuchs, und der Aufwand der Pflege vervielfacht sich. Richtig ist eine zentral gepflegte Vorlage, die Teams erweitern dürfen. Zweiter Fehler: Onboarding-Anleitungen als lange Dokumente zu führen, die veralten; besser ein Skript, das getestet wird. Drittens: Automatisierungen, die nur Arbeit abnehmen, aber nicht erklären; wer nicht weiß, warum der Hook ablehnt, umgeht ihn. Gute Fehlermeldungen nennen die Regel und ein Beispiel (wie im Hook oben). Vierter Fehler: Konfiguration in der globalen Datei ohne Rückweg; Skripte sollten sagen, was sie ändern.

### Praxisbezug.

Ein Maß für Developer Experience ist die Zeit bis zum ersten erfolgreichen Pull Request eines neuen Teammitglieds. Wer sie misst, sieht, ob Vorlagen, Dokumentation und Onboarding-Skript tatsächlich helfen. Weitere Bausteine sind ein `CONTRIBUTING.md` mit dem Arbeitsablauf, eine Seite mit den häufigsten Fehlern (Modul 10, Abschnitt 10.4) und ein Kanal für Fragen.

## 11.8 Plattformvergleich

Die Frage „welche Plattform ist die beste?“ hat keine allgemeine Antwort. Sie hängt davon ab, was ein Unternehmen braucht, was es schon hat und was es sich leisten und betreiben kann. Sinnvoller ist ein strukturierter Vergleich mit Kriterien, die man vor der Bewertung festlegt und gewichtet. Vier Gruppen von Kriterien kehren immer wieder. *Funktionen*: Repository-Funktionen, Reviews, CI/CD, Sicherheitsfunktionen, Paketverwaltung, Projektverwaltung. *Kosten*: Lizenzen oder Abos, Pipeline-Minuten, Speicher, Betriebsaufwand bei eigenem Betrieb (Modul 12, Abschnitt Kostenfaktoren). *Security und Compliance*: Identitätsanbindung, Audit-Log, Datenstandort, Zertifizierungen, Schutzregeln. *Betrieb*: Cloud oder Self-Managed, Verfügbarkeit, Backup, Support, Migrationsaufwand.

Die Dokumentationen, die in diesem Modul gelesen wurden, zeigen, dass die Plattformen dieselben Aufgaben mit anderen Begriffen lösen. Die Tabelle fasst den Vergleich der Schutz- und Governance-Mechanismen zusammen, soweit er aus den abgerufenen Seiten belegt ist. Sie ist keine Bewertung und nennt keine Preise, weil diese sich häufig ändern und hier nicht abgerufen wurden.

|                       | **GitHub**                                                               | **GitLab**                                    | **Azure Repos**                             |
|:----------------------|:-------------------------------------------------------------------------|:----------------------------------------------|:--------------------------------------------|
| Struktur              | Enterprise, Organization, Team, Repository                               | Gruppen, Untergruppen, Projekte               | Organisation, Projekt, Repository           |
| Branch-Schutz         | Rulesets (Repository, Organisation, Enterprise), älter Branch Protection | Protected Branches, Branch Rules, Push Rules  | Branch Policies                             |
| Änderungsvorschlag    | Pull Request                                                             | Merge Request                                 | Pull Request                                |
| Freigaben             | Reviews, Code Owner im Ruleset                                           | Approval Rules (Premium, Ultimate), 0 bis 100 | Mindestzahl Reviewer, automatische Reviewer |
| Testlauf neuer Regeln | Enforcement-Status „Evaluate“                                            | nicht in den gelesenen Seiten belegt          | nicht in den gelesenen Seiten belegt        |
| Identität             | SAML/OIDC, SCIM (EMU), Gruppen für Teams                                 | nicht in den gelesenen Seiten belegt          | nicht in den gelesenen Seiten belegt        |

Die Lücken in der Tabelle sind Absicht: Wo eine Aussage nicht aus einer abgerufenen Seite stammt, steht das dort. Wer eine Entscheidung vorbereitet, füllt sie aus den aktuellen Dokumentationen der Anbieter oder in Gesprächen mit ihnen.

Die Bewertung selbst ist am besten mit einer *gewichteten Entscheidungsmatrix* zu machen. Die Gewichte spiegeln, was dem Unternehmen wichtig ist, die Punkte die Einschätzung jeder Plattform je Kriterium. Das Ergebnis ist nicht die Wahrheit, sondern eine nachvollziehbare Begründung, die sich diskutieren und bei geänderten Gewichten neu berechnen lässt. Das folgende Skript berechnet die Matrix aus einer Datei. Wichtig: Die Gewichte und Punkte im Beispiel sind frei erfundene Übungswerte und sagen nichts über echte Produkte aus.

```bash
cat > kriterien.csv <<'EOF'
kriterium;gewicht
Funktionen;25
Kosten;20
Security-Compliance;25
Betrieb;15
Migrationsaufwand;15
EOF
cat > bewertung.csv <<'EOF'
plattform;kriterium;punkte
Plattform A;Funktionen;4
Plattform A;Kosten;2
Plattform A;Security-Compliance;5
Plattform A;Betrieb;4
Plattform A;Migrationsaufwand;3
Plattform B;Funktionen;5
Plattform B;Kosten;3
Plattform B;Security-Compliance;4
Plattform B;Betrieb;2
Plattform B;Migrationsaufwand;4
Plattform C;Funktionen;3
Plattform C;Kosten;5
Plattform C;Security-Compliance;3
Plattform C;Betrieb;5
Plattform C;Migrationsaufwand;2
EOF
cat > matrix.sh <<'EOF'
#!/bin/sh
# Gewichtete Entscheidungsmatrix; Punkte 1 (schlecht) bis 5 (sehr gut), Gewichte in Prozent
# Aufruf: matrix.sh <kriterien.csv> <bewertung.csv>
awk -F';' '
  NR==FNR { if (FNR>1) { g[$1]=$2; summe+=$2 } next }
  FNR>1   { s[$1]+=$3*g[$2]; if (!($1 in n)) { n[$1]=1; liste[++k]=$1 } }
  END {
    if (summe != 100) { printf "WARNUNG: Gewichte ergeben %d statt 100\n", summe }
    for (i=1;i<=k;i++) printf "%-12s %.2f von 5\n", liste[i], s[liste[i]]/summe
  }' "$1" "$2" | sort -k3 -nr
EOF
echo "--- Standardgewichte"
sh matrix.sh kriterien.csv bewertung.csv
echo "--- Sensitivitaet: Kosten doppelt so wichtig (Gewichte neu verteilt)"
sed 's/^Kosten;20/Kosten;40/; s/^Funktionen;25/Funktionen;15/; s/^Security-Compliance;25/Security-Compliance;20/; s/^Betrieb;15/Betrieb;15/; s/^Migrationsaufwand;15/Migrationsaufwand;10/' kriterien.csv > kriterien2.csv
sh matrix.sh kriterien2.csv bewertung.csv
```

Das Skript summiert für jede Plattform Punkte mal Gewicht und teilt durch die Gewichtssumme, sodass das Ergebnis wieder zwischen 1 und 5 liegt; es warnt, wenn die Gewichte nicht 100 ergeben. Mit den Standardgewichten liegt Plattform B vorn, Plattform A knapp dahinter. Mit stärker gewichteten Kosten ändert sich die Reihenfolge: Plattform C rückt nach vorn. Diese *Sensitivitätsprüfung* ist das Wichtigste an der Methode, denn sie zeigt, wie stabil die Rangfolge gegenüber den Gewichten ist. Wenn kleine Änderungen das Ergebnis drehen, ist die Entscheidung in Wahrheit offen, und die Gewichte sollten mit den Beteiligten geklärt werden, bevor man die Zahlen verteidigt.

### Typische Fehler und Stolperfallen.

Falsch ist, eine Plattform nach Gewohnheit oder Marketing zu wählen und die Kriterien nachträglich zu konstruieren. Richtig ist, Kriterien und Gewichte vor der Bewertung festzulegen. Zweiter Fehler: Nur die Lizenzkosten zu vergleichen und Betrieb, Migration und Schulung zu übersehen. Dritter Fehler: Eine Plattform zu wählen, ohne Auszugsstrategie (Export, Backup, Modul 12); der Wechsel ist sonst später teuer. Vierter Fehler: Nur mit Funktionslisten zu vergleichen statt mit einem kleinen Pilotprojekt auf den Favoriten.

### Praxisbezug.

Viele Unternehmen enden mit mehreren Plattformen, weil Übernahmen, Teams und Altlasten das so ergeben. Eine *Plattformstrategie* legt dann fest, welche Plattform für neue Projekte gilt, was mit den alten geschieht und bis wann, und welche Mindeststandards für alle gelten (11.6). Sie ist ein Dokument mit Begründung, Verantwortlichen und Überprüfungstermin.

## Übungen zu Modul 11

**Aufgabe 1.** Erklären Sie die Gliederung Enterprise, Organization, Team, Repository (GitHub) und ordnen Sie die entsprechenden Begriffe bei GitLab und Azure DevOps zu. Warum ist die Wahl der Ebene für eine Regel wichtig?

**Aufgabe 2.** Was bewirken die drei Enforcement-Status eines GitHub-Rulesets? Wie führen Sie eine neue Regel sicher ein?

**Aufgabe 3.** Worin unterscheiden sich Polling und Webhook bei Jenkins, und warum ist das Polling in kurzen Abständen problematisch?

**Aufgabe 4.** Was bedeuten SSO, SCIM und Gruppen-Synchronisierung, und welche Aufgabe löst jedes davon im Lebenszyklus eines Benutzers?

**Aufgabe 5.** Praktisch: Schreiben Sie einen `post-receive`-Hook, der Push-Optionen auswertet. Er soll bei `-o ticket=ABC-123` eine Meldung mit der Ticketnummer ausgeben und bei einer Nummer, die nicht dem Muster `[A-Z]+-[0-9]+` entspricht, eine Warnung. Testen Sie ihn mit beiden Fällen.

**Aufgabe 6.** Praktisch: Erweitern Sie `policy-check.sh` um die Regel „`.gitignore` muss vorhanden sein“ und um eine Ausnahme mit Ablaufdatum. Prüfen Sie am Stichtag vor und nach dem Ablauf.

**Aufgabe 7.** Praktisch: Schreiben Sie ein Skript, das aus einer Liste von IdP-Benutzern und einer Liste von Plattform-Benutzern drei Dateien erzeugt (verwaist, fehlend, in beiden) und eine Liste bekannter Dienstkonten als Ausnahme berücksichtigt.

**Aufgabe 8.** Praktisch (Transfer): Bauen Sie die Entscheidungsmatrix für drei Plattformen Ihrer Wahl mit mindestens fünf Kriterien, führen Sie eine Sensitivitätsprüfung durch und formulieren Sie in drei Sätzen eine Empfehlung mit Begründung und Unsicherheiten.

## Lösungen zu Modul 11

**Lösung 1.** GitHub: Enterprise (mehrere Organisationen), Organization (Repositories, Mitglieder, Teams), Team (Gruppe mit Rollen), Repository. GitLab: Gruppen und Untergruppen mit vererbten Rechten, Projekte. Azure DevOps: Organisation, Projekt (mit Teams), Repository. Die Ebene bestimmt, für wie viele Repositories eine Regel gilt und wer sie ändern darf; eine zu tiefe Ebene führt zu Wildwuchs, eine zu hohe zu Regeln, die nicht überall passen.

**Lösung 2.** Active erzwingt, Evaluate protokolliert nur Verstöße (Rule Insights), Disabled ist aus. Sichere Einführung: zuerst Evaluate, Auswertung der Verstöße, Kommunikation, dann Active; Bypass nur für wenige Akteure.

**Lösung 3.** Polling: Jenkins fragt periodisch, Verzögerung bis zur nächsten Abfrage, Dauerlast auch ohne Änderungen. Webhook: Der Server meldet Änderungen sofort. Zu kurze Intervalle belasten Jenkins und den Git-Server und ersetzen keinen Webhook; empfohlen werden mindestens 15 Minuten und `H` im Cron-Muster zur Lastverteilung.

**Lösung 4.** SSO: Anmeldung beim IdP, Bestätigung an die Plattform (SAML oder OIDC), Regeln wie MFA. SCIM: automatisches Anlegen, Ändern und Deaktivieren von Konten. Gruppen-Synchronisierung: Übertragung von IdP-Gruppen in Teams und damit Rechte. Zusammen erreichen sie, dass Ein- und Austritt an einer Stelle gepflegt werden.

**Lösung 5.** Im Hook: `case "$opt" in ticket=*) t=${opt#ticket=}; echo "$t" | grep -Eq '^[A-Z]+-[0-9]+$' && echo "Ticket $t" >&2 || echo "Warnung: ungueltige Ticketnummer" >&2 ;; esac`. Voraussetzung: `receive.advertisePushOptions true` im Server-Repository. Test: `git push -o ticket=ABC-123` und `-o ticket=abc`.

**Lösung 6.** Zusätzliche Zeile `[ -f "$r/.gitignore" ]; pruefe "$name" ".gitignore" $?`; Ausnahme in der CSV mit Ablaufdatum, etwa `delta;.gitignore;Grund;Name;2026-11-30`. Vor dem Ablauf meldet das Skript „AUSNAHME“, danach „ABGELAUFEN“ und Exit-Code 1.

**Lösung 7.** `comm -13` liefert Verwaiste, `comm -23` Fehlende, `comm -12` Treffer in beiden (Listen vorher mit `sort` sortieren). Dienstkonten: Liste einlesen und mit `grep -vxFf dienstkonten.txt` aus der Verwaisten-Liste entfernen.

**Lösung 8.** Erwartet: eine Tabelle mit Gewichten (Summe 100), Punkten 1 bis 5, Ergebnis je Plattform und eine zweite Rechnung mit verschobenen Gewichten. Die Empfehlung nennt die Spitzenreiterin, den Abstand, ob die Rangfolge bei der Sensitivitätsprüfung stabil bleibt, und nennt Annahmen und Datenlücken.
