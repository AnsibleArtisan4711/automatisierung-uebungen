# Modul 10 - Automation, Scheduling und CI/CD

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (10.1 bis 10.8; 10.9 Übungsaufgaben, 10.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 10 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (8 Einheiten à 1,25 h). Beide Angaben stimmen überein.  
**Priorität:** 2 (mittel)

| Nr.  | Lerneinheit             | Dauer (h) | Prio |
|:-----|:------------------------|----------:|:----:|
| 10.1 | Cron und systemd Timer  |      1,25 |  2   |
| 10.2 | Batch-Jobs              |      1,25 |  2   |
| 10.3 | Git Hooks               |      1,25 |  2   |
| 10.4 | CI/CD Shell Steps       |      1,25 |  2   |
| 10.5 | Deployment-Skripte      |      1,25 |  2   |
| 10.6 | Parallelisierung        |      1,25 |  2   |
| 10.7 | Konfigurationsparameter |      1,25 |  3   |
| 10.8 | Automationsstandards    |      1,25 |  3   |
|      | **Summe**               | **10,00** |      |

Automation bedeutet, dass Aufgaben ohne menschliches Zutun zur richtigen Zeit, im richtigen Zusammenhang und mit nachvollziehbarem Ergebnis laufen: zeitgesteuert (Cron, Timer), ereignisgesteuert (Git-Hooks, Pipelines) oder als geordnete Folge (Batch, Deployment). In diesem Modul setzen Sie die Bausteine der Module 6 bis 9 ein: robuste Skripte, Exit-Codes, Logging und Sicherheit. Wiederkehrendes Prinzip ist, dass ein unbeaufsichtigter Lauf *keine Rückfragen* stellt, *überlappende Läufe verhindert*, *Fehler sichtbar macht* und *wiederholt werden kann*.

## 10.1 Cron und systemd Timer

*Cron* ist der klassische Zeitplaner. Jeder Benutzer hat eine *Crontab* (`crontab -e` zum Bearbeiten, `crontab -l` zum Anzeigen), dazu gibt es systemweite Dateien in `/etc/cron.d/`, die zusätzlich eine Spalte für den Benutzer haben. Eine Zeile besteht aus fünf Zeitfeldern und dem Befehl: Minute (0–59), Stunde (0–23), Tag des Monats (1–31), Monat (1–12) und Wochentag (0–7, wobei 0 und 7 Sonntag sind). Ein Stern bedeutet „jeder Wert“, `*/5` „alle fünf“, `1-5` einen Bereich, `1,15` eine Liste. `30 2 * * 1-5` heißt: werktags um 02:30 Uhr. Wichtig ist eine Besonderheit: Sind *Tag des Monats und Wochentag beide* eingeschränkt, genügt es, wenn *eines* von beiden passt.

Cron startet Befehle mit einer *minimalen Umgebung* (kurzer `PATH`, `SHELL=/bin/sh`, kein Terminal, Home des Benutzers; siehe Modul 9.6). Daher gehören absolute Pfade oder ein gesetzter `PATH` an den Anfang, und der Befehl wird mit `sh` interpretiert, nicht mit Bash, sofern man nichts anderes festlegt (`SHELL=/bin/bash` in der Crontab). Das Zeichen `%` hat in Crontab-Zeilen eine Sonderbedeutung (Zeilenumbruch) und muss als `\%` geschrieben werden, was bei `date +%F` häufig vergessen wird. Die Ausgabe eines Cron-Jobs geht standardmäßig per Mail an den Besitzer (`MAILTO`); ohne konfigurierten Mailserver verschwindet sie. Deshalb leitet man Ausgaben bewusst um (`>> /var/log/job.log 2>&1` oder `` logger -t job\|) und macht Fehler über Exit-Code und Monitoring sichtbar (Modul 9).

Der *systemd Timer* ist die moderne Alternative: Eine `.timer`-Unit löst eine gleichnamige `.service`-Unit aus. Die Vorteile: Die Ausgabe landet im Journal (`journalctl -u job.service`), der Dienst erbt Härtungsoptionen und Ressourcenlimits (`User=`, `ProtectSystem=`, `MemoryMax=`), `systemctl list-timers` zeigt letzte und nächste Läufe, und `Persistent=true` holt verpasste Läufe nach, wenn der Rechner zum Zeitpunkt ausgeschaltet war. Mit `RandomizedDelaySec` verteilt man Läufe vieler Rechner, damit nicht alle gleichzeitig auf denselben Server zugreifen. Ein Dienst, der noch läuft, wird nicht ein zweites Mal gestartet; das schützt zumindest teilweise vor Überlappung.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Die Handbuchseite `systemd.timer` (abgerufen, siehe Quellenverzeichnis) beschreibt: `OnCalendar=` legt Echtzeit-Zeitpunkte fest und darf mehrfach angegeben werden. `Persistent=` (Standard: nein) speichert den Zeitpunkt des letzten Auslösens auf Platte; wurde ein Lauf verpasst, startet der Dienst bei der Aktivierung sofort. `AccuracySec=` (Standard: eine Minute) erlaubt systemd, Auslösungen innerhalb dieses Fensters zusammenzufassen, `RandomizedDelaySec=` (Standard: 0) addiert eine Zufallsverzögerung. Ein Timer aktiviert standardmäßig die Service-Unit mit gleichem Namen; ist diese bereits aktiv, wird sie nicht neu gestartet.

Ein Skript, das unter Cron oder Timer läuft, sollte zusätzlich selbst für Sicherheit sorgen: eine *Sperre* gegen überlappende Läufe (`flock`), ein *Zeitlimit* (`timeout`), *Protokollierung* mit Start, Ende, Dauer und Exit-Code und eine *definierte Umgebung*. Das erledigt ein kleiner Wrapper, den man für alle Jobs wiederverwendet.

```bash
#!/usr/bin/env bash
# job-wrapper.sh [-t SEKUNDEN] NAME BEFEHL [ARG...]  - Sperre, Zeitlimit, Log, Exit-Code
set -uo pipefail
limit=3600
if [[ ${1:-} == -t ]]; then limit="${2:?}"; shift 2; fi
name="${1:?Aufruf: $0 [-t SEK] NAME BEFEHL...}"; shift
(( $# >= 1 )) || { echo "BEFEHL fehlt" >&2; exit 2; }
[[ $name =~ ^[a-z0-9_-]+$ ]] || { echo "ungueltiger Jobname" >&2; exit 2; }

export PATH="/usr/local/bin:/usr/bin:/bin"          # definierte Umgebung
LOGVERZ="${LOGVERZ:-${TMPDIR:-/tmp}/job-logs}"; mkdir -p "$LOGVERZ"
sperre="${TMPDIR:-/tmp}/job-$name.lock"
log() { printf '%s %s [%s] %s\n' "$(date -u +%FT%TZ)" "$name" "$$" "$*" >> "$LOGVERZ/$name.log"; }

exec 9> "$sperre"
if ! flock -n 9; then log "uebersprungen: vorheriger Lauf aktiv"; echo "$name laeuft bereits" >&2; exit 75; fi

log "Start: $*"
start=$SECONDS
timeout --kill-after=10 "$limit" "$@" >> "$LOGVERZ/$name.log" 2>&1
rc=$?
dauer=$((SECONDS - start))
case $rc in
    0)   log "Ende: ok (${dauer}s)" ;;
    124) log "ABBRUCH: Zeitlimit ${limit}s ueberschritten" ;;
    *)   log "FEHLER: Exit-Code $rc nach ${dauer}s" ;;
esac
exit "$rc"
```

Der Wrapper prüft zuerst seine Argumente (Modul 8.2) und setzt einen festen `PATH`. Die Sperre `flock -n 9` auf einer Sperrdatei ist *nicht blockierend*: Läuft schon ein Exemplar, protokolliert der Wrapper das Überspringen und endet mit Code 75 (`EX_TEMPFAIL` aus `sysexits.h`: vorübergehender Fehler). Der eigentliche Befehl läuft unter `timeout`: Nach dem Limit erhält er `SIGTERM`, nach weiteren zehn Sekunden `SIGKILL`; der Code 124 zeigt die Zeitüberschreitung. Die Ausgabe des Befehls geht in dasselbe Log wie die Wrapper-Meldungen. Der Exit-Code wird unverändert durchgereicht, damit Cron-Mails, systemd oder das Monitoring das Ergebnis erkennen. Die Sperre wird mit dem Prozess automatisch freigegeben, auch bei Absturz, ein Vorteil gegenüber selbstgebauten Sperrdateien (Modul 4.8).

Typisch ist die Einbindung in der Crontab wie in der folgenden Zeile:

> 2 \* \* 1-5 /usr/local/bin/job-wrapper.sh -t 1800 backup /usr/local/bin/backup.sh

Alternativ läuft der Wrapper als Timer. Die zugehörigen Unit-Dateien und die Prüfung zeigt der folgende Block; er schreibt in ein Testverzeichnis und installiert nichts.

```bash
#!/usr/bin/env bash
# timer-demo.sh - Unit-Dateien erzeugen und mit systemd-analyze pruefen (ohne Installation)
set -euo pipefail
u="$(mktemp -d)"; trap 'rm -rf -- "$u"' EXIT

cat > "$u/backup.service" <<'ENDE'
[Unit]
Description=Nachtbackup

[Service]
Type=oneshot
ExecStart=/bin/true
DynamicUser=yes
NoNewPrivileges=true
ProtectSystem=strict
PrivateTmp=true
TimeoutStartSec=30min
ENDE

cat > "$u/backup.timer" <<'ENDE'
[Unit]
Description=Nachtbackup werktags 02:30

[Timer]
OnCalendar=Mon..Fri 02:30
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
ENDE

echo "--- Syntaxpruefung:"
systemd-analyze verify "$u/backup.service" "$u/backup.timer" && echo "Units ok"

echo "--- Wie versteht systemd den Zeitplan?"
systemd-analyze calendar "Mon..Fri 02:30" | sed -n '1,3p'
echo "--- Vergleichbare Crontab-Zeile:"
echo '30 2 * * 1-5  /usr/local/bin/job-wrapper.sh -t 1800 backup /usr/local/bin/backup.sh'

echo "--- Wrapper: zweiter Lauf waehrend der erste noch laeuft"
export LOGVERZ="$u/logs"
printf '#!/bin/sh\nsleep 2\necho fertig\n' > "$u/lang.sh"; chmod +x "$u/lang.sh"
w="${WRAPPER:-./job-wrapper.sh}"
if [[ -x $w ]]; then
    "$w" demo "$u/lang.sh" & sleep 0.5
    rc=0; "$w" demo "$u/lang.sh" || rc=$?
    echo "zweiter Lauf: Exit-Code $rc (75 = uebersprungen)"; wait
    cut -d' ' -f2- "$LOGVERZ/demo.log"
fi
```

`systemd-analyze verify` meldet Syntaxfehler und unbekannte Optionen schon vor der Installation; `systemd-analyze calendar` zeigt die normalisierte Form des Zeitplans samt nächstem Lauf, ein hervorragendes Mittel, um Kalenderausdrücke wie `Mon..Fri 02:30` oder `*-*-01 03:00` (am Monatsersten) zu prüfen. Der Dienst ist `Type=oneshot`: systemd wartet, bis der Befehl endet, und wertet seinen Exit-Code. Er läuft mit `DynamicUser=yes` unter einem von systemd automatisch angelegten, unprivilegierten Benutzer und mit Härtungsoptionen. Der Wrapper-Test (nur, wenn er im aktuellen Verzeichnis liegt) demonstriert die Sperre.

Typische Fehler: Erstens verlässt man sich auf die interaktive Umgebung (`PATH`, Variablen, Locale), und der Job läuft nur im Terminal. Zweitens fehlt `\` vor `%` in der Crontab. Drittens überlappen sich Läufe, weil der Job länger dauert als das Intervall, und zwei Instanzen zerstören gegenseitig ihre Daten. Viertens verschwindet die Ausgabe in einer nie gelesenen Mailbox. Fünftens unterschätzt man Zeitzonen und Sommerzeit: Ein Job um 02:30 Uhr läuft bei der Umstellung zweimal oder gar nicht; für kritische Jobs wählt man Zeiten außerhalb von 02:00 bis 03:00 Uhr oder rechnet in UTC. Sechstens laufen alle Rechner um Punkt Mitternacht auf denselben Server (`RandomizedDelaySec` oder ein Versatz). Siebtens läuft ein Job als `root`, obwohl ein eigener Benutzer reichen würde (Modul 8.5). Achtens gibt es keinen Hinweis, wenn der Job *nicht mehr läuft* (Modul 9.3: Zeitstempel des letzten Erfolgs).

```bash
# falsch: Prozent ungeschuetzt, kein Pfad, keine Sperre, Ausgabe weg
0 3 * * * backup.sh > /dev/null
0 4 * * * tar czf /backup/x-$(date +%F).tgz /srv

# richtig: Prozent maskiert, absolute Pfade, Wrapper mit Sperre und Log
SHELL=/bin/bash
0 3 * * * /usr/local/bin/job-wrapper.sh -t 1800 backup /usr/local/bin/backup.sh
0 4 * * * /usr/local/bin/job-wrapper.sh tar /usr/local/bin/tar-backup.sh
```

> **Merke**
>
> Cron: fünf Felder, minimale Umgebung, `\%`, absolute Pfade, Ausgabe umleiten. Timer: Journal, Härtung, `Persistent`, `RandomizedDelaySec`; Prüfen mit `systemd-analyze verify/calendar`. Jeder Job: Sperre (`flock`), Zeitlimit (`timeout`), Log, durchgereichter Exit-Code, Alarm bei Ausbleiben.

> **Praxisbeispiel**
>
> Alle nächtlichen Jobs eines Teams laufen über denselben Wrapper. Bleibt ein Lauf hängen, beendet ihn das Zeitlimit und der Exit-Code 124 löst einen Alarm aus; läuft ein Job zwei Nächte nicht, meldet das Monitoring den veralteten Erfolgs-Zeitstempel.

## 10.2 Batch-Jobs

Ein *Batch-Job* verarbeitet einen Stapel von Eingaben ohne Interaktion: Dateien eines Verzeichnisses, Zeilen einer Tabelle, Datensätze aus einer Warteschlange. Gute Batch-Jobs folgen dem Muster *Eingabe, Verarbeitung, Ausgabe* mit klaren Schnittstellen, und sie sind auf Fehler vorbereitet: Bei Datensatz 4711 von 10 000 darf nicht alles von vorn beginnen, und es darf nichts doppelt verarbeitet werden.

Bewährt hat sich eine *Verzeichnisstruktur als Zustandsmaschine*: `eingang/` (neue Dateien), `arbeit/` (gerade in Bearbeitung), `fertig/` (erfolgreich verarbeitet) und `fehler/` (nicht verarbeitbar, mit Fehlerbericht). Jeder Zustandswechsel ist ein `mv` innerhalb desselben Dateisystems, also atomar. Das Verschieben nach `arbeit/` dient zugleich als *Beanspruchung* (Claim): Schlägt es fehl, hat ein anderer Prozess die Datei schon genommen. Diese Struktur macht den Zustand *sichtbar* (ein Blick ins Verzeichnis genügt), *wiederanlauffähig* (Reste in `arbeit/` nach einem Absturz werden beim nächsten Start zurück nach `eingang/` geschoben) und *idempotent* (fertige Dateien werden nie erneut verarbeitet). Bei einer einzigen großen Eingabedatei arbeitet man stattdessen mit einem *Checkpoint*: Man merkt sich die Nummer der letzten erfolgreich verarbeiteten Zeile in einer Statusdatei und setzt dort fort.

Die Standardisierung betrifft auch die *Fehlerstrategie*: Bricht der ganze Lauf beim ersten Fehler ab (*fail fast*) oder verarbeitet er den Rest und meldet die Fehler gesammelt (*continue on error*)? Beides ist legitim; entscheidend ist, dass es bewusst gewählt und dokumentiert ist, und dass der *Exit-Code* das Ergebnis ausdrückt (zum Beispiel 0 alles gut, 3 teilweise fehlgeschlagen, 1 abgebrochen). Außerdem begrenzt man Wiederholungen: Eine Datei, die dreimal scheitert, landet in `fehler/`, statt den Job für immer zu blockieren (*Poison Message*).

```bash
#!/usr/bin/env bash
# batch.sh BASIS  - Verzeichnis-Zustandsmaschine: eingang -> arbeit -> fertig | fehler
set -uo pipefail
basis="${1:?Aufruf: $0 BASIS}"
mkdir -p "$basis"/{eingang,arbeit,fertig,fehler}
umask 027
ok=0; fehl=0

# Verarbeitung einer Datei: hier "Zeilen zaehlen", Dateien ohne Kopfzeile "KOPF" schlagen fehl
verarbeite() {
    local f="$1" out="$2"
    head -n 1 "$f" | grep -q '^KOPF' || { echo "Kopfzeile fehlt" >&2; return 3; }
    printf '%s: %d Zeilen\n' "${f##*/}" "$(wc -l < "$f")" > "$out"
}

# 1. Wiederanlauf: Reste eines abgebrochenen Laufs zurueck in den Eingang
for rest in "$basis"/arbeit/*; do
    [[ -e $rest ]] || continue
    echo "Wiederanlauf: ${rest##*/} zurueck in den Eingang"
    mv -n -- "$rest" "$basis/eingang/"
done

# 2. Alle Eingangsdateien nacheinander beanspruchen und verarbeiten
for datei in "$basis"/eingang/*; do
    [[ -f $datei ]] || continue
    name="${datei##*/}"
    mv -n -- "$datei" "$basis/arbeit/$name" 2>/dev/null || { echo "uebersprungen (bereits beansprucht): $name"; continue; }
    if fehlertext="$(verarbeite "$basis/arbeit/$name" "$basis/arbeit/$name.ergebnis" 2>&1)"; then
        mv -- "$basis/arbeit/$name" "$basis/arbeit/$name.ergebnis" "$basis/fertig/"
        echo "fertig:  $name"; ok=$((ok + 1))
    else
        printf '%s\n' "$fehlertext" > "$basis/arbeit/$name.fehler"
        mv -- "$basis/arbeit/$name" "$basis/arbeit/$name.fehler" "$basis/fehler/"
        rm -f -- "$basis/arbeit/$name.ergebnis"
        echo "FEHLER:  $name ($fehlertext)"; fehl=$((fehl + 1))
    fi
done

echo "Zusammenfassung: $ok ok, $fehl fehlgeschlagen"
(( fehl == 0 )) && exit 0 || exit 3      # 3 = teilweise fehlgeschlagen
```

Der Ablauf hat zwei Phasen. Zuerst räumt der Wiederanlauf `arbeit/` auf: Was dort liegt, stammt aus einem abgebrochenen Lauf und geht zurück in den Eingang (`mv -n` überschreibt nichts). Danach beansprucht die Schleife jede Datei mit `mv -n` nach `arbeit/`; läuft versehentlich ein zweiter Prozess, bekommt nur einer jede Datei. Die Funktion `verarbeite` gibt Fehler über Exit-Code und stderr zurück; die Schleife fängt beides in `fehlertext` ab. Bei Erfolg wandern Datei und Ergebnis nach `fertig/`, bei einem Fehler Datei und Fehlerbericht nach `fehler/`, sodass eine Person die Ursache dort sieht. Der Exit-Code 3 meldet „teilweise fehlgeschlagen“; das Skript läuft nicht `set -e`, denn Fehler einzelner Dateien sind erwartet und werden gezielt behandelt (Modul 6.1). Man probiert es mit zwei Dateien, von denen eine mit `KOPF` beginnt und eine nicht, und startet das Skript zweimal: Beim zweiten Mal ist nichts zu tun.

Typische Fehler: Erstens werden Dateien *kopiert* statt verschoben und beim nächsten Lauf erneut verarbeitet. Zweitens beginnt die Verarbeitung, während die Datei noch geschrieben wird (der Lieferant schreibt direkt in `eingang/`); Lieferanten sollen in ein Zwischenverzeichnis schreiben und *atomar* umbenennen, oder man wartet, bis sich Größe und Zeitstempel nicht mehr ändern. Drittens kein Wiederanlauf: Nach einem Absturz bleiben Dateien in `arbeit/` ewig liegen. Viertens kein Limit für Wiederholungen. Fünftens fehlerhafte Eingaben bringen den ganzen Lauf zum Abbruch, ohne dass die gute Hälfte verarbeitet wird. Sechstens Dateinamen mit Leerzeichen oder Sonderzeichen werden nicht in Anführungszeichen gesetzt. Siebtens fehlt die Zusammenfassung mit Zahlen am Ende, die später im Monitoring auswertbar wäre.

```bash
# falsch: kopieren, kein Claim, kein Wiederanlauf, Fehler bricht alles ab
set -e
for f in eingang/*; do verarbeite $f; cp $f fertig/; done

# richtig: atomar beanspruchen, Ergebnis sichtbar ablegen, Fehler isolieren
mv -n -- "$f" "arbeit/${f##*/}" || continue
if verarbeite "arbeit/${f##*/}"; then mv -- "arbeit/${f##*/}" fertig/; else mv -- "arbeit/${f##*/}" fehler/; fi
```

> **Merke**
>
> Zustand als Verzeichnisse (eingang, arbeit, fertig, fehler), Wechsel per atomarem `mv`. Wiederanlauf: Reste in `arbeit/` zurückführen. Fehler isolieren, Fehlerbericht ablegen, Wiederholungen begrenzen. Exit-Code unterscheidet vollständigen und teilweisen Erfolg. Lieferanten liefern atomar.

> **Praxisbeispiel**
>
> Ein Lieferant legt jede Nacht CSV-Dateien auf einem Dateiserver ab. Das Batch-Skript verarbeitet sie, verschiebt gute in den Archivordner und schlechte mit einem Fehlerbericht in einen Prüfordner. Am Morgen sieht das Team auf einen Blick, was verarbeitet wurde und was nachgeliefert werden muss.

## 10.3 Git Hooks

*Git-Hooks* sind Skripte, die Git bei bestimmten Ereignissen automatisch ausführt. Sie liegen im Verzeichnis `.git/hooks/` und tragen feste Namen. Die wichtigsten *clientseitigen* Hooks: `pre-commit` (vor dem Anlegen eines Commits; schlägt er fehl, kommt kein Commit zustande), `commit-msg` (prüft die Commit-Meldung, erhält den Dateinamen als Argument), `pre-push` (vor dem Hochladen) und `post-merge` oder `post-checkout` (Aufräumarbeiten). *Serverseitig* gibt es `pre-receive` und `update`, die ein Server beim Empfang von Pushes ausführt und mit denen Richtlinien durchgesetzt werden. Ein Hook ist ein ausführbares Skript (jede Sprache, meist Bash); sein *Exit-Code* entscheidet: 0 lässt die Aktion zu, ungleich 0 bricht sie ab.

Clientseitige Hooks sind eine *Komfortfunktion*, keine Sicherheitsgrenze: Sie liegen lokal, werden nicht mit dem Repository übertragen und lassen sich mit `git commit --no-verify` überspringen. Man nutzt sie deshalb für *schnelles Feedback* (Formatierung, Syntaxfehler, versehentlich eingecheckte Geheimnisse) und stützt sich für verbindliche Prüfungen auf die CI-Pipeline und serverseitige Regeln (Modul 10.4). Damit alle Teammitglieder dieselben Hooks verwenden, legt man sie im Repository ab (zum Beispiel `.githooks/`) und setzt einmalig `git config core.hooksPath .githooks`.

Ein guter `pre-commit`-Hook prüft nur das, was *zum Commit gehört*: die *gestageten* Dateien (`git diff --cached --name-only --diff-filter=ACM`). Er prüft den *gestageten Inhalt*, nicht die Arbeitskopie, denn beides kann sich unterscheiden (Teilcommit mit `git add -p`); den gestageten Stand liest man mit `git show :pfad`. Er muss *schnell* sein (Sekunden), eindeutig melden, *welche* Datei und Zeile ein Problem hat, und den Weg zur Behebung nennen. Weil er auf Entwicklerrechnern läuft, fehlen dort manchmal Werkzeuge; ein Hook sollte ein fehlendes Werkzeug melden, nicht mit einem kryptischen Fehler scheitern.

```bash
#!/usr/bin/env bash
# githook-demo.sh - legt ein Test-Repository mit pre-commit-Hook an und probiert ihn aus
set -uo pipefail
r="$(mktemp -d)"; trap 'rm -rf -- "$r"' EXIT
cd "$r"
git init -q . && git config user.name Test && git config user.email t@example.org
mkdir .githooks && git config core.hooksPath .githooks

cat > .githooks/pre-commit <<'ENDE'
#!/usr/bin/env bash
# pre-commit: prueft gestagete Shell-Skripte (Syntax, CRLF, Geheimnisse, ShellCheck falls vorhanden)
set -uo pipefail
fehler=0
melde() { echo "  - $*" >&2; fehler=$((fehler + 1)); }

while IFS= read -r -d '' datei; do
    case $datei in *.sh) ;; *) continue ;; esac
    inhalt="$(git show ":$datei")"                         # gestagter Stand, nicht Arbeitskopie
    echo "pruefe $datei" >&2
    bash -n <<< "$inhalt" 2>/tmp/syntax.$$ || melde "$datei: Syntaxfehler: $(head -n 1 /tmp/syntax.$$)"
    grep -q $'\r' <<< "$inhalt"                            && melde "$datei: CRLF-Zeilenenden"
    grep -Eiq '(password|passwort|secret|token)[[:space:]]*=[[:space:]]*["'\'']?[A-Za-z0-9/+_-]{8,}' <<< "$inhalt" \
                                                           && melde "$datei: sieht nach eingechecktem Geheimnis aus"
    if command -v shellcheck >/dev/null; then
        shellcheck -S warning - <<< "$inhalt" >/tmp/sc.$$ 2>&1 || melde "$datei: ShellCheck: $(head -n 2 /tmp/sc.$$ | tr '\n' ' ')"
    fi
done < <(git diff --cached --name-only --diff-filter=ACM -z)

rm -f /tmp/syntax.$$ /tmp/sc.$$
if (( fehler > 0 )); then
    echo "Commit abgebrochen: $fehler Problem(e). (Notfalls: git commit --no-verify)" >&2
    exit 1
fi
ENDE
chmod +x .githooks/pre-commit

echo "=== Commit 1: sauberes Skript"
printf '#!/usr/bin/env bash\nset -euo pipefail\necho "hallo"\n' > gut.sh
git add gut.sh && git commit -q -m "Gutes Skript" && echo "Commit angelegt"

echo; echo "=== Commit 2: Skript mit Geheimnis und ungequoteter Variable"
printf '#!/usr/bin/env bash\nTOKEN="abcd1234efgh5678"\nrm -rf $1\n' > schlecht.sh
git add schlecht.sh
git commit -q -m "Schlechtes Skript" && echo "Commit angelegt" || echo "-> Hook hat den Commit verhindert (Exit-Code $?)"
echo; git log --oneline
```

Das Beispiel erzeugt ein Test-Repository, richtet `core.hooksPath` auf ein versioniertes Verzeichnis ein und legt dort den Hook an. Der Hook liest mit `git diff --cached --name-only -z` die gestageten Dateien (mit Nullbyte-Trennung, damit Dateinamen mit Leerzeichen funktionieren) und holt jeden Inhalt mit `git show :datei`. Er führt vier Prüfungen aus: Syntax mit `bash -n`, Zeilenenden, ein einfaches Muster für Geheimnisse und, falls installiert, ShellCheck mit `-S warning`. Jede Prüfung meldet die Datei und das Problem. Beim zweiten Commit schlagen Geheimnis-Erkennung und ShellCheck (ungequotetes `$1`) an, der Hook beendet sich mit 1, und Git legt keinen Commit an. Das Geheimnis-Muster ist bewusst einfach; für den ernsthaften Einsatz gibt es spezialisierte Werkzeuge (zum Beispiel `gitleaks`, `detect-secrets`).

Typische Fehler: Erstens prüft der Hook die Arbeitskopie statt des gestageten Inhalts und meldet Fehler, die gar nicht im Commit stehen, oder übersieht welche. Zweitens ist er so langsam, dass Entwickler ihn mit `-{}-no-verify` umgehen. Drittens liegt er nur in `.git/hooks/` und existiert deshalb nur auf einem Rechner; ein versioniertes Verzeichnis mit `core.hooksPath` (oder ein Werkzeug wie `pre-commit`) verteilt ihn. Viertens hält man Hooks für verbindlich und verzichtet auf die Prüfung in der CI. Fünftens verändert der Hook Dateien stillschweigend (automatisches Formatieren), ohne sie neu zu stagen. Sechstens fehlt das Ausführrecht, und Git ignoriert den Hook ohne Meldung (neuere Versionen warnen mit einem Hinweis). Siebtens bricht der Hook mit nichtssagender Meldung ab; er soll Datei, Zeile und Abhilfe nennen.

```bash
# falsch: Arbeitskopie pruefen, Hook nur lokal, nicht ausfuehrbar
shellcheck *.sh || exit 1          # in .git/hooks/pre-commit, ohne chmod +x

# richtig: gestagten Stand pruefen, Hook versioniert und aktiviert
git diff --cached --name-only --diff-filter=ACM -z | while IFS= read -r -d '' f; do
    git show ":$f" | shellcheck -S warning - || exit 1
done
git config core.hooksPath .githooks
```

> **Merke**
>
> Hooks sind Komfort, keine Sicherheitsgrenze: verbindliche Prüfungen gehören in CI und Server. Gestagten Inhalt prüfen (`git show :datei`), schnell bleiben, klar melden. Hooks versionieren und mit `core.hooksPath` aktivieren. Exit-Code ungleich 0 bricht ab; `-{}-no-verify` umgeht den Hook.

> **Praxisbeispiel**
>
> Ein Repository mit Betriebsskripten hat einen `pre-commit`-Hook, der ShellCheck, Syntaxprüfung und einen Geheimnis-Scan ausführt. Derselbe Prüfaufruf läuft zusätzlich in der CI; so fallen die häufigsten Fehler schon vor dem Commit auf, und die Pipeline bleibt die verbindliche Instanz.

## 10.4 CI/CD Shell Steps

In *Continuous Integration* und *Continuous Delivery* (CI/CD) führt ein Server bei jeder Änderung automatisch Schritte aus: bauen, testen, prüfen, ausliefern. Jeder Schritt (*Step*) eines Jobs ist meist ein Shell-Skript. Das gilt für GitHub Actions (`run:`), GitLab CI (`script:`) und Azure Pipelines (`script:` oder `bash:`). Die Plattformen unterscheiden sich in der Konfigurationssprache (YAML), im Aufbau und in den Variablen, aber die Prinzipien für Bash sind gleich: Der Schritt *gelingt, wenn der Exit-Code 0 ist*, und jeder andere Code lässt den Job fehlschlagen. Daher gilt alles aus Modul 6: strenge Optionen, saubere Exit-Codes, keine ignorierten Fehler.

Ein wichtiger Punkt ist, *welche Shell mit welchen Optionen* der Runner verwendet. Die Plattform startet das Skript oft mit eigenen Voreinstellungen, die sich unterscheiden und sich auch ändern können; verlassen Sie sich deshalb nicht darauf, sondern setzen Sie am Anfang mehrzeiliger Skripte selbst `set -euo pipefail` beziehungsweise legen Sie die Shell ausdrücklich fest (bei GitHub Actions mit `shell: bash` oder `defaults.run.shell`). Mehrzeilige `run:`-Blöcke laufen als *ein* Skript; ohne `set -e` gilt oft nur der Exit-Code des *letzten* Befehls. Komplexe Logik gehört nicht in YAML, sondern in ein *Skript im Repository*, das man lokal ausführen, testen und mit ShellCheck prüfen kann; der Pipeline-Schritt ruft es nur auf.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Die GitHub-Dokumentation zu Standardwerten für Jobs (abgerufen, siehe Quellenverzeichnis) beschreibt, dass sich Shell und Arbeitsverzeichnis aller `run`-Schritte über `defaults.run` setzen lassen (auf Workflow- oder Job-Ebene, wobei die Job-Ebene gewinnt), zum Beispiel `defaults: run: shell: bash`; in diesem Schlüssel sind keine Kontexte oder Ausdrücke erlaubt. *Hinweis (nicht abgerufen, bitte in der aktuellen Dokumentation prüfen):* Nach meinem Kenntnisstand startet GitHub `shell: bash` mit `-eo pipefail`, während ohne ausdrückliche Angabe nur `-e` gilt; genau deshalb ist es sicherer, die Optionen im Skript selbst zu setzen.

Zur Zusammenarbeit zwischen Skript und Plattform gehören *Annotationen* (Fehlermeldungen, die die Plattform in der Oberfläche hervorhebt: bei GitHub `::error file=x,line=3::Text`, bei Azure `##vso[task.logissue type=error]Text`), *Ausgabewerte* (bei GitHub durch Anhängen von `name=wert` an die Datei, deren Pfad in `$GITHUB_OUTPUT` steht), *Zusammenfassungen* (Markdown nach `$GITHUB_STEP_SUMMARY`), *Artefakte* (Dateien, die der Job für nachfolgende Jobs oder zum Herunterladen speichert) und das *Maskieren* von Geheimnissen (`::add-mask::`). Geheimnisse der Plattform werden in Logs durch Sternchen ersetzt, aber nur, wenn sie unverändert ausgegeben werden; Ableitungen (Base64, Teilstrings) werden nicht erkannt. Bei *Exit-Codes* sollte ein Skript zwischen „Prüfung fehlgeschlagen“ (Job rot) und „nicht zutreffend“ unterscheiden können, und Schritte, die Fehler tolerieren sollen, sagen das ausdrücklich (`continue-on-error` bei GitHub).

```bash
#!/usr/bin/env bash
# ci-schritt.sh - Pruefschritt, der unter CI Annotationen, Ausgabewerte und Zusammenfassung liefert
set -euo pipefail

in_github() { [[ ${GITHUB_ACTIONS:-} == true ]]; }

fehler_melden() {                                    # fehler_melden DATEI ZEILE TEXT
    if in_github; then
        # Annotation: erscheint direkt am Code im Pull Request
        printf '::error file=%s,line=%s::%s\n' "$1" "$2" "$3"
    else
        printf 'FEHLER %s:%s: %s\n' "$1" "$2" "$3" >&2
    fi
}

setze_ausgabe() {                                    # setze_ausgabe NAME WERT (fuer nachfolgende Schritte)
    if in_github && [[ -n ${GITHUB_OUTPUT:-} ]]; then printf '%s=%s\n' "$1" "$2" >> "$GITHUB_OUTPUT"
    else printf '[Ausgabe] %s=%s\n' "$1" "$2"; fi
}

zusammenfassung() {                                  # Markdown-Zusammenfassung
    if in_github && [[ -n ${GITHUB_STEP_SUMMARY:-} ]]; then cat >> "$GITHUB_STEP_SUMMARY"
    else cat; fi
}

# --- Pruefung: alle *.sh unter VERZ muessen eine Shebang-Zeile haben und ohne CR sein
verz="${1:?Aufruf: $0 VERZ}"
geprueft=0; probleme=0
while IFS= read -r -d '' f; do
    geprueft=$((geprueft + 1))
    head -n 1 "$f" | grep -q '^#!' || { fehler_melden "$f" 1 "Shebang-Zeile fehlt"; probleme=$((probleme + 1)); }
    if n="$(grep -n $'\r' "$f" | head -n 1 | cut -d: -f1)" && [[ -n $n ]]; then
        fehler_melden "$f" "$n" "CRLF-Zeilenende"; probleme=$((probleme + 1))
    fi
done < <(find "$verz" -type f -name '*.sh' -print0 | sort -z)

setze_ausgabe geprueft "$geprueft"
setze_ausgabe probleme "$probleme"
{
    echo "### Skript-Pruefung"
    echo "| Geprueft | Probleme |"; echo "|---:|---:|"; echo "| $geprueft | $probleme |"
} | zusammenfassung

(( probleme == 0 ))                                  # Exit-Code 0 = Schritt gruen, sonst rot
```

Das Skript erkennt mit `GITHUB_ACTIONS=true`, ob es unter GitHub Actions läuft, und verhält sich dann plattformgerecht: Fehler als `::error`-Annotation mit Datei und Zeile, Ausgabewerte in die Datei `$GITHUB_OUTPUT`, eine Markdown-Tabelle an `$GITHUB_STEP_SUMMARY`. Lokal gibt es dieselben Informationen in gewöhnlicher Form aus, und genau deshalb lässt sich das Skript auf dem Entwicklerrechner testen, bevor es in die Pipeline wandert. Die letzte Zeile macht die Bedingung `(( probleme == 0 ))` zum Exit-Code: Bei null Problemen ist der Schritt grün. Die Testläufe: einmal in einem Verzeichnis ohne Probleme, einmal mit `GITHUB_ACTIONS=true GITHUB_OUTPUT=/tmp/out.txt`, um die Annotationen und die Ausgabedatei zu sehen. Die zugehörige Konfiguration in der Pipeline ist kurz: Ein Schritt `run: ./ci/ci-schritt.sh scripts` genügt, und `shell: bash` legt die Shell fest.

Typische Fehler: Erstens steckt die gesamte Logik in YAML-Blöcken, die niemand lokal ausführen oder prüfen kann. Zweitens fehlt `set -euo pipefail`; ein fehlgeschlagener Befehl in der Mitte wird übergangen, und der Job bleibt grün. Drittens `curl ` bash\| und Abhängigkeiten ohne feste Version oder Prüfsumme (Modul 8.6): Die Pipeline ist nicht reproduzierbar und angreifbar. Viertens Geheimnisse werden in Befehlszeilen oder durch `set -x` geloggt (Modul 8.4). Fünftens Artefakte und Zwischenstände werden nicht zwischen Jobs weitergegeben, und jeder Job baut neu oder nutzt veraltete Reste. Sechstens unterscheiden sich Runner (Distribution, Bash-Version, installierte Werkzeuge) von der lokalen Umgebung, und „funktioniert bei mir“ gilt nicht (Modul 11.8). Siebtens Fehler werden mit `|| true` unterdrückt, damit die Pipeline „grün“ wird; das verschiebt das Problem in die Produktion.

```bash
# falsch: Logik in YAML, Fehler unterdrueckt, Geheimnis im Log
# - run: |
#     npm test || true
#     echo "Token: ${{ secrets.TOKEN }}"
#     curl https://example.org/install.sh | bash

# richtig (GitHub Actions): Skript im Repository, Shell und Optionen explizit
# - name: Pruefen
#   shell: bash
#   run: |
#     set -euo pipefail
#     ./ci/ci-schritt.sh scripts
```

> **Merke**
>
> Ein CI-Schritt ist grün bei Exit-Code 0. Optionen (`set -euo pipefail`) und Shell selbst festlegen. Logik als Skript ins Repository, lokal testbar und mit ShellCheck geprüft. Plattformfunktionen (Annotationen, Ausgabewerte, Zusammenfassung, Artefakte) nur über kleine Hilfsfunktionen. Keine Geheimnisse im Log; Fehler nie mit `|| true` wegreden.

> **Praxisbeispiel**
>
> Das Team ruft in jeder Pipeline dasselbe Skript `ci/pruefen.sh` auf, das auch auf dem Laptop läuft. Fehler erscheinen als Annotation direkt im Pull Request, die Zusammenfassung zeigt Zahlen, und kein Schritt besteht nur aus langem YAML.

## 10.5 Deployment-Skripte

Ein *Deployment* bringt eine neue Version einer Anwendung in Betrieb. Ein gutes Deployment-Skript macht den Übergang *kontrolliert*: Es baut oder holt das Artefakt, lädt es auf das Ziel, prüft es, schaltet um, startet neu und prüft erneut, und wenn etwas schiefgeht, kehrt es zur vorigen Version zurück. Die Grundidee ist die *Release-Verzeichnisstruktur*: Jede Version liegt in einem eigenen Verzeichnis (`releases/20261030-1415`), und ein Symlink `current` zeigt auf die aktive. Das Umschalten ist das Umsetzen des Symlinks, eine *atomare* Operation: Die Anwendung sieht entweder die alte oder die neue Version, nie eine halb kopierte. Der Rollback ist ebenfalls nur das Zurücksetzen des Symlinks auf das vorige Release, ohne Neu-Kopieren.

Der Ablauf gliedert sich in feste Schritte. (1) *Vorprüfungen*: Ziel erreichbar, genug Platz, Artefakt vorhanden und mit Prüfsumme verifiziert (Modul 8.6). (2) *Hochladen* in ein neues Release-Verzeichnis (`rsync`, `tar` über SSH, Kopie). (3) *Vorbereiten*: Konfiguration verknüpfen (gemeinsame Dateien liegen in `shared/` und werden eingebunden), Migrationen ausführen. (4) *Umschalten*: Symlink setzen, Dienst neu starten oder neu laden. (5) *Smoke Test*: ein schneller Funktionstest der laufenden Anwendung (Statusseite antwortet, Version stimmt). (6) Bei Fehler *Rollback*, bei Erfolg *Aufräumen* (nur die letzten N Releases behalten). Symlinks setzt man atomar mit `ln -sfn ziel link.neu && mv -T link.neu link`: Das einfache `ln -sfn` entfernt und legt den Link in zwei Schritten an (kurzes Zeitfenster ohne Link), das Umbenennen mit `mv -T` ist dagegen atomar.

Ein Deployment sollte außerdem *wiederholbar* und *gegen Parallelität gesichert* sein (Sperre mit `flock`), jeden Schritt protokollieren (Modul 8.8, 9.1) und mit eindeutigen Exit-Codes enden. Datenbank-Migrationen sind die schwierigste Stelle, denn ein Rollback des Codes macht sie nicht rückgängig; man schreibt sie *abwärtskompatibel* (erst Spalte hinzufügen, später entfernen), sodass alte und neue Version mit dem Schema leben können.

```bash
#!/usr/bin/env bash
# deploy.sh BASIS ARTEFAKTVERZ [VERSION]  - Releases, atomarer Wechsel, Smoke Test, Rollback
set -euo pipefail
basis="${1:?Aufruf: $0 BASIS ARTEFAKTVERZ [VERSION]}"; artefakt="${2:?}"
version="${3:-$(date +%Y%m%d-%H%M%S)}"
BEHALTEN="${BEHALTEN:-3}"
basis="$(realpath -m -- "$basis")"; artefakt="$(realpath -m -- "$artefakt")"   # absolute Pfade, sonst zeigen Symlinks ins Leere
[[ $version =~ ^[A-Za-z0-9._-]+$ ]] || { echo "ungueltige Version" >&2; exit 2; }
[[ -d $artefakt ]] || { echo "Artefakt fehlt: $artefakt" >&2; exit 2; }

releases="$basis/releases"; shared="$basis/shared"; current="$basis/current"
mkdir -p "$releases" "$shared"
exec 9> "$basis/.deploy.lock"; flock -n 9 || { echo "Deployment laeuft bereits" >&2; exit 75; }
log() { printf '%s %s\n' "$(date -u +%H:%M:%S)" "$*"; }

# Platzhalter fuer echte Aktionen (hier simuliert)
restart_dienst() { log "  (Dienst wird neu gestartet)"; }
smoke_test() {                                  # prueft, ob das aktive Release 'OK' meldet
    [[ "$(cat "$current/status.txt" 2>/dev/null)" == "OK" ]]
}

# 1. Neues Release anlegen und befuellen (noch nicht aktiv)
neu="$releases/$version"
[[ ! -e $neu ]] || { echo "Release $version existiert bereits" >&2; exit 2; }
log "Upload: $version"
cp -a -- "$artefakt" "$neu"
ln -s "$shared/config.env" "$neu/config.env"      # gemeinsame Datei einhaengen

# 2. Voriges Release merken (fuer Rollback), dann atomar umschalten
vorher=""; [[ -L $current ]] && vorher="$(readlink -f "$current")"
log "Umschalten auf $version"
ln -sfn "$neu" "$current.neu" && mv -T -- "$current.neu" "$current"
restart_dienst

# 3. Smoke Test; bei Fehler Rollback
if smoke_test; then
    log "Smoke Test bestanden"
else
    log "Smoke Test FEHLGESCHLAGEN"
    if [[ -n $vorher ]]; then
        log "Rollback auf ${vorher##*/}"
        ln -sfn "$vorher" "$current.neu" && mv -T -- "$current.neu" "$current"
        restart_dienst
    fi
    rm -rf -- "$neu"
    exit 1
fi

# 4. Aufraeumen: nur die letzten BEHALTEN Releases behalten (aktives nie loeschen)
mapfile -t alle < <(find "$releases" -mindepth 1 -maxdepth 1 -type d | sort)
aktiv="$(readlink -f "$current")"
if (( ${#alle[@]} > BEHALTEN )); then
    for alt in "${alle[@]:0:${#alle[@]}-BEHALTEN}"; do
        [[ $(readlink -f "$alt") == "$aktiv" ]] || { log "Entferne altes Release ${alt##*/}"; rm -rf -- "${alt:?}"; }
    done
fi
log "Fertig: aktiv ist ${aktiv##*/}"
```

Das Skript hält sich an die beschriebene Reihenfolge. Zuerst wird die Sperre genommen und das neue Release in sein eigenes Verzeichnis kopiert, ohne dass es schon aktiv wäre. Der Wechsel geschieht über einen neuen Symlink und `mv -T`, also atomar. Der Smoke Test prüft hier vereinfacht eine Statusdatei im aktiven Release; in der Praxis fragt er eine URL ab (Modul 7.5) oder ruft `systemctl is-active` auf (Modul 7.3). Schlägt er fehl, setzt das Skript den Link auf das vorige Release zurück, startet den Dienst neu, löscht das fehlerhafte Release und endet mit Exit-Code 1. Das Aufräumen löscht ältere Releases, behält aber mindestens `BEHALTEN` und *nie das aktive*. Zum Ausprobieren legt man ein Artefaktverzeichnis mit einer Datei `status.txt` an: mit Inhalt `OK` gelingt das Deployment, mit anderem Inhalt kommt es zum Rollback.

Typische Fehler: Erstens wird direkt in das aktive Verzeichnis kopiert, und die Anwendung läuft während des Kopierens mit einem Mischzustand. Zweitens kein Rollback-Weg: Das alte Release wurde überschrieben. Drittens fehlt der Smoke Test, und ein Deployment gilt als erfolgreich, obwohl die Anwendung nicht antwortet. Viertens läuft Aufräumen *vor* dem Umschalten oder löscht das aktive Release. Fünftens zwei parallele Deployments ohne Sperre. Sechstens Konfiguration und Geheimnisse liegen im Release und gehen beim Wechsel verloren oder landen im Repository (`shared/` und Secret Store, Modul 8.4). Siebtens Datenbank-Migrationen sind nicht abwärtskompatibel, sodass das Zurücksetzen des Codes nicht mehr genügt. Achtens Deployment als `root` über SSH mit einem allmächtigen Schlüssel (Modul 7.8, 8.5).

```bash
# falsch: ueberschreiben, kein Rollback, kein Test
rsync -a --delete build/ /srv/app/
systemctl restart app

# richtig: neues Release daneben, atomar umschalten, testen, sonst zurueck
rsync -a build/ "/srv/app/releases/$v/" && ln -sfn "/srv/app/releases/$v" /srv/app/current.neu
mv -T /srv/app/current.neu /srv/app/current && systemctl restart app
curl -fsS --max-time 5 http://localhost/health || { ln -sfn "$vorher" /srv/app/current.neu; mv -T /srv/app/current.neu /srv/app/current; systemctl restart app; exit 1; }
```

> **Merke**
>
> Releases nebeneinander, `current` als Symlink, Umschalten atomar (`ln -sfn` auf Hilfslink, dann `mv -T`). Danach Smoke Test; bei Fehler automatisch Rollback. Sperre gegen Parallelität, Aufräumen mit Mindestanzahl, aktives Release nie löschen. Migrationen abwärtskompatibel.

> **Praxisbeispiel**
>
> Jedes Deployment legt ein Release an, schaltet um und prüft eine Health-URL. Die letzten fünf Releases bleiben auf dem Server. Als nach einem Update die Anmeldung nicht mehr funktionierte, genügte ein Rollback auf das vorige Release in Sekunden, die Fehlersuche folgte danach in Ruhe.

## 10.6 Parallelisierung

Viele Aufgaben bestehen aus unabhängigen Teilen: tausend Dateien komprimieren, zwanzig Server abfragen, hundert Bilder skalieren. Nacheinander ausgeführt dauert das lange, obwohl mehrere Prozessorkerne oder Netzwerkverbindungen zur Verfügung stehen. *Parallelisierung* führt die Teile gleichzeitig aus und verkürzt die Gesamtzeit. Sie ist *nicht kostenlos*: Zu viele gleichzeitige Prozesse überlasten Rechner und Ziele (CPU, Speicher, Platte, Netz, Datenbanken, Rate Limits von Diensten), und Ausgaben, Fehler und Abhängigkeiten werden komplizierter. Deshalb gehört zu jeder Parallelisierung ein *Limit*.

Das einfachste Werkzeug ist `xargs -P N`: Es führt einen Befehl für die gelesenen Argumente aus, höchstens N gleichzeitig, zum Beispiel `find . -name '*.log' -print0 | xargs -0 -P 4 -n 1 gzip`. Mit `-n 1` bekommt jeder Aufruf ein Argument, `-0` liest nullbyte-getrennt (sicher bei Dateinamen mit Leerzeichen), `-P $(nproc)` passt N an die Zahl der Kerne an. Wichtig: `xargs` liefert den Exit-Code 123, wenn *ein* Aufruf mit 1 bis 125 endete, 124 wenn ein Aufruf mit 255 endete und 125/126/127 in besonderen Fällen; ein Fehler eines Teilauftrags wird also gemeldet, aber nicht welcher. Für komplexere Befehle ruft man `bash -c 'funktion "$1"' _ {}` mit exportierter Funktion (`export -f funktion`) auf. Das Programm *GNU Parallel* bietet mehr Komfort (Zusammenführung der Ausgaben, Wiederholung, Fortschritt), ist aber nicht überall installiert.

In Bash selbst startet man Hintergrundjobs mit `&` und wartet mit `wait`. Für eine *Obergrenze* zählt man die laufenden Jobs (`jobs -rp`) und wartet mit `wait -n` (ab Bash 4.3), bis *irgendeiner* fertig ist, bevor man den nächsten startet. `wait PID` liefert den Exit-Code eines bestimmten Jobs; so sammelt man Ergebnisse. Auch das *Drosseln* (Rate Limiting) gehört dazu: ein `sleep` zwischen Starts oder ein Werkzeug mit Ratenbegrenzung, wenn ein Zielsystem nur eine bestimmte Zahl Anfragen pro Sekunde verträgt. Die Last begrenzt man auch von außen: `nice -n 10` senkt die Priorität, `ionice -c3` die Platten-Priorität.

```bash
#!/usr/bin/env bash
# parallel-demo.sh - xargs -P und ein eigener Job-Pool mit Limit und Ergebnissammlung
set -uo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
aufgabe() {                                   # simuliert Arbeit; Aufgabe 4 scheitert
    local n="$1"
    sleep 0.3
    (( n == 4 )) && { echo "Aufgabe $n: FEHLER" >&2; return 7; }
    echo "Aufgabe $n fertig"
}
export -f aufgabe

echo "=== 1) Sequentiell, 8 Aufgaben"
start=$SECONDS; for n in 1 2 3 4 5 6 7 8; do aufgabe "$n" 2>/dev/null; done >/dev/null
echo "Dauer: $((SECONDS - start)) s"

echo "=== 2) xargs -P 4 (4 gleichzeitig)"
start=$(date +%s.%N)
ausgabe="$(seq 1 8 | xargs -P 4 -I{} bash -c 'aufgabe {}' 2>/dev/null)"; rc=$?
printf '%s\n' "$ausgabe" | sort | tr '\n' ' '; echo
echo "Exit-Code von xargs: $rc  (123 = mindestens ein Aufruf scheiterte)"
awk -v a="$start" -v b="$(date +%s.%N)" 'BEGIN { printf "Dauer: %.1f s\n", b - a }'

echo "=== 3) Eigener Pool mit Limit 3 und Sammeln der Exit-Codes"
LIMIT=3
declare -A pid_zu_aufgabe=()
fehlgeschlagen=()
sammle() {                                    # wartet auf einen beliebigen Job und wertet ihn aus
    local fertig
    wait -n -p fertig "${!pid_zu_aufgabe[@]}"; local rc=$?
    (( rc == 0 )) || fehlgeschlagen+=("${pid_zu_aufgabe[$fertig]} (rc=$rc)")
    unset "pid_zu_aufgabe[$fertig]"
}
for n in 1 2 3 4 5 6 7 8; do
    while (( ${#pid_zu_aufgabe[@]} >= LIMIT )); do sammle; done      # Obergrenze einhalten
    aufgabe "$n" >"$t/aus.$n" 2>&1 &
    pid_zu_aufgabe[$!]="Aufgabe $n"
done
while (( ${#pid_zu_aufgabe[@]} > 0 )); do sammle; done              # Rest abwarten
echo "fehlgeschlagen: ${fehlgeschlagen[*]:-keine}"
cat "$t"/aus.* | sort | tr '\n' ' '; echo
(( ${#fehlgeschlagen[@]} == 0 ))
```

Der erste Block misst die sequentielle Dauer (8 mal 0,3 Sekunden). Block 2 nutzt `xargs -P 4`; die Funktion wurde mit `export -f` für die Kind-Shells sichtbar gemacht. Die Gesamtdauer sinkt auf etwa ein Viertel plus Startkosten, und der Exit-Code 123 zeigt, dass ein Aufruf scheiterte, ohne zu verraten welcher. Block 3 baut einen *Job-Pool* mit `wait -n -p` (die Option `-p` liefert die PID des fertigen Jobs und ist ab Bash 5.1 verfügbar): Das Array `pid_zu_aufgabe` merkt sich, welcher Prozess welche Aufgabe bearbeitet; `sammle` wartet auf irgendeinen davon, wertet den Exit-Code aus und entfernt ihn. Solange das Limit erreicht ist, wartet die Schleife, bevor sie die nächste Aufgabe startet. Jede Aufgabe schreibt in eine eigene Datei, damit sich Ausgaben nicht vermischen. Am Ende wird das Ergebnis gesammelt und ein Gesamt-Exit-Code abgeleitet.

Typische Fehler: Erstens keine Obergrenze (`for ...; do aufgabe & done` mit 10 000 Hintergrundprozessen). Zweitens vermischte Ausgaben, weil alle Prozesse gleichzeitig auf stdout schreiben; besser jede Aufgabe in eine eigene Datei oder die Ausgabe zeilenweise puffern. Drittens ungeschützter gemeinsamer Zustand: Zwei Prozesse schreiben in dieselbe Datei oder zählen in derselben Variablen (Subshells sehen Änderungen anderer nicht). Viertens gehen Exit-Codes verloren, weil man nach dem Start nie `wait` aufruft. Fünftens überlastet die Parallelität das *Ziel* (Datenbank, API mit Rate Limit) trotz geringer Last auf dem eigenen Rechner. Sechstens startet man `xargs` ohne `-0` bei Dateinamen mit Leerzeichen. Siebtens verlässt man sich auf `wait -n -p`, das auf älteren Systemen (Bash unter 5.1) nicht existiert; dort sammelt man Codes mit `wait PID` in Startreihenfolge.

```bash
# falsch: unbegrenzt starten, Ausgaben vermischt, Fehler ignoriert
for h in $(cat hosts.txt); do ssh "$h" uptime & done

# richtig: Limit, nullbyte-sicher, Exit-Code beachten, ssh -n
tr '\n' '\0' < hosts.txt | xargs -0 -P 5 -n 1 ssh -n -o BatchMode=yes -o ConnectTimeout=10 -- uptime
rc=$?; (( rc == 0 )) || echo "mindestens ein Host fehlgeschlagen (xargs-Code $rc)"
```

> **Merke**
>
> Parallelisieren mit Limit: `xargs -P N` (mit `-0`) oder Job-Pool mit `wait -n`. Ausgaben trennen, Exit-Codes sammeln, gemeinsamen Zustand vermeiden. Ziele und Rate Limits schonen; `nice` und `ionice` für Fairness. `xargs` meldet Teilfehler als 123.

> **Praxisbeispiel**
>
> Ein Wartungsskript prüft 200 Server per SSH. Sequentiell dauert es 40 Minuten, mit `xargs -P 10` etwa vier. Das Limit 10 stellt sicher, dass der Jump-Host nicht überlastet wird, und die Ergebnisse stehen pro Server in einer eigenen Datei, aus der der Bericht entsteht.

## 10.7 Konfigurationsparameter

Ein Skript, das in mehreren Umgebungen läuft (Entwicklung, Test, Produktion), darf Werte wie Hostnamen, Pfade, Limits oder Schalter nicht fest im Code haben. Man trennt *Code* (gleich in allen Umgebungen) von *Konfiguration* (pro Umgebung verschieden). Dafür gibt es vier Quellen mit einer festen *Rangfolge*, von schwach zu stark: eingebaute *Standardwerte*, eine *Konfigurationsdatei*, *Umgebungsvariablen* und *Kommandozeilenoptionen*. Was die Person beim Aufruf ausdrücklich angibt, gewinnt gegenüber der Umgebung, diese gegenüber der Datei, und die Datei gegenüber dem Standard. Diese Reihenfolge entspricht den Erwartungen (und den Zwölf-Faktor-Prinzipien für Anwendungen: Konfiguration über die Umgebung) und macht das Verhalten vorhersagbar.

Welche Quelle wofür? *Umgebungsvariablen* eignen sich für Werte, die je Umgebung oder Lauf wechseln und für Secrets in Containern und CI (Modul 8.4); sie sind aber unstrukturiert und werden an Kindprozesse vererbt. *Konfigurationsdateien* eignen sich für viele zusammengehörige Einstellungen; sie liegen im Repository (ohne Geheimnisse) oder werden von der Konfigurationsverwaltung erzeugt (Modul 7.7). Das einfachste Format ist `SCHLUESSEL=wert`. Man liest es mit `source` nur dann, wenn die Datei *vertrauenswürdig* ist, denn sie führt beliebigen Code aus (`wert=$(rm -rf ...)`). Sicherer ist es, die Datei *zu parsen*: Zeile für Zeile, Schlüssel gegen eine Allowlist prüfen und Werte als Daten übernehmen. Pflichtwerte werden *früh geprüft* (`${VAR:?Meldung}`), und Werte werden validiert (Zahlenbereiche, Pfade, erlaubte Auswahl).

Die Trennung je Umgebung löst man mit einer Datei pro Umgebung (`dev.env`, `prod.env`), die man über eine Variable auswählt (`APP_ENV=prod`). Gemeinsame Werte stehen in einer Basisdatei, die Umgebungsdatei überschreibt einzelne. Wichtig ist, *die wirksame Konfiguration anzeigen* zu können (ein `-{}-print-config`), mit Maskierung von Geheimnissen; das ist die erste Frage bei jeder Fehlersuche.

```bash
#!/usr/bin/env bash
# konfig-demo.sh - Rangfolge: Standard < Datei < Umgebung < Kommandozeile, sicheres Parsen
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT

# --- 1. Erlaubte Schluessel und Standardwerte
ERLAUBT=(HOST PORT LOGLEVEL MODUS TOKEN)
declare -A KONF=([HOST]=localhost [PORT]=8080 [LOGLEVEL]=info [MODUS]=dev [TOKEN]="")

# --- 2. Datei sicher parsen (kein source): nur KEY=wert, nur erlaubte Schluessel
lade_datei() {                                # lade_datei DATEI
    local datei="$1" zeile k v nr=0
    [[ -r $datei ]] || { echo "Konfigdatei nicht lesbar: $datei" >&2; return 1; }
    while IFS= read -r zeile || [[ -n $zeile ]]; do
        nr=$((nr + 1))
        [[ -z $zeile || $zeile == \#* ]] && continue
        if [[ $zeile =~ ^([A-Z_][A-Z0-9_]*)=(.*)$ ]]; then
            k="${BASH_REMATCH[1]}"; v="${BASH_REMATCH[2]}"
            v="${v#\"}"; v="${v%\"}"                         # einfache Anfuehrungszeichen entfernen
            if [[ " ${ERLAUBT[*]} " == *" $k "* ]]; then KONF[$k]="$v"
            else echo "Zeile $nr: unbekannter Schluessel $k (ignoriert)" >&2; fi
        else echo "Zeile $nr: ungueltig (ignoriert)" >&2; fi
    done < "$datei"
}

# --- 3. Umgebungsvariablen mit Praefix APP_ ueberschreiben die Datei
lade_umgebung() {
    local k
    for k in "${ERLAUBT[@]}"; do
        local name="APP_$k"
        [[ -n ${!name:-} ]] && KONF[$k]="${!name}"
    done
    return 0
}

# --- 4. Kommandozeile gewinnt: --port 9000 --loglevel debug
lade_argumente() {
    while (( $# )); do
        case $1 in
            --host)     KONF[HOST]="${2:?--host braucht Wert}";     shift 2 ;;
            --port)     KONF[PORT]="${2:?--port braucht Wert}";     shift 2 ;;
            --loglevel) KONF[LOGLEVEL]="${2:?--loglevel braucht Wert}"; shift 2 ;;
            *) echo "unbekannte Option: $1" >&2; return 2 ;;
        esac
    done
}

pruefe() {
    [[ ${KONF[PORT]} =~ ^[0-9]+$ ]] && (( KONF[PORT] >= 1 && KONF[PORT] <= 65535 )) || { echo "PORT ungueltig" >&2; return 1; }
    case ${KONF[LOGLEVEL]} in debug|info|warn|error) ;; *) echo "LOGLEVEL ungueltig" >&2; return 1 ;; esac
    [[ ${KONF[MODUS]} != prod || -n ${KONF[TOKEN]} ]] || { echo "MODUS=prod braucht TOKEN" >&2; return 1; }
}

zeige() {                                     # wirksame Konfiguration, Geheimnisse maskiert
    local k v
    for k in "${ERLAUBT[@]}"; do
        v="${KONF[$k]}"; [[ $k == TOKEN && -n $v ]] && v="***"
        printf '  %-9s = %s\n' "$k" "$v"
    done
}

cat > "$t/app.env" <<'ENDE'
# Konfiguration der Anwendung
HOST=db.test.example
PORT=5432
LOGLEVEL=warn
EVIL=$(echo boese)
MODUS=prod
TOKEN="abc123geheim"
ENDE

echo "--- nur Standardwerte:"; zeige
echo "--- + Datei:";           lade_datei "$t/app.env"; zeige
echo "--- + Umgebung (APP_PORT=6543):"; APP_PORT=6543 lade_umgebung; zeige
export APP_PORT=6543; lade_umgebung
echo "--- + Kommandozeile (--port 9000 --loglevel debug):"
lade_argumente --port 9000 --loglevel debug; zeige
pruefe && echo "Konfiguration gueltig"
```

Die Quelle mit der niedrigsten Priorität steht am Anfang (Standardwerte im assoziativen Array), danach werden Datei, Umgebung und Argumente nacheinander darübergelegt; jede Ausgabe von `zeige` zeigt, wie sich der Wert ändert (der Port wandert von 8080 über 5432 und 6543 zu 9000). `lade_datei` liest Zeilen und akzeptiert nur das Muster `SCHLUESSEL=wert` mit erlaubtem Schlüssel; die Zeile `EVIL=$(echo boese)` wird mit Hinweis ignoriert, *und selbst wenn sie erlaubt wäre, würde der Wert als Text übernommen und nicht ausgeführt*. Das ist der Unterschied zu `source`. Die Umgebung wirkt über den Präfix `APP_`, der Kollisionen mit Systemvariablen verhindert. `pruefe` validiert Wertebereiche und eine Abhängigkeit (Produktionsmodus erfordert ein Token), `zeige` druckt die wirksame Konfiguration und maskiert das Geheimnis.

Typische Fehler: Erstens stehen Werte im Code, und jede Umgebung bekommt eine eigene Skriptkopie (Drift). Zweitens `source datei.env` für Dateien, die nicht vertrauenswürdig sind. Drittens eine unklare Rangfolge: Niemand weiß, ob Datei oder Umgebung gewinnt. Viertens fehlen Pflichtwerte erst mitten im Lauf auf (`unbound variable` nach einer Stunde); alles wird *zu Beginn* validiert. Fünftens landen Geheimnisse in der Konfigurationsdatei im Repository (Modul 8.4). Sechstens Umgebungsvariablen ohne Präfix kollidieren mit `PATH`, `HOME`, `USER`. Siebtens fehlt die Möglichkeit, die wirksame Konfiguration anzuzeigen, und Fehlersuche wird zum Rätselraten.

```bash
# falsch: unbekannte Datei ausfuehren, keine Pruefung, Geheimnis im Klartext im Log
source "$1"
echo "Verbinde mit $HOST:$PORT mit Token $TOKEN"

# richtig: parsen, validieren, wirksame Werte maskiert zeigen
[[ $zeile =~ ^([A-Z_][A-Z0-9_]*)=(.*)$ ]] && KONF[${BASH_REMATCH[1]}]="${BASH_REMATCH[2]}"
echo "Verbinde mit ${KONF[HOST]}:${KONF[PORT]} (Token ${KONF[TOKEN]:+gesetzt})"
```

> **Merke**
>
> Code und Konfiguration trennen. Rangfolge: Standard, Datei, Umgebung, Kommandozeile. Dateien parsen statt `source`, Schlüssel per Allowlist, früh validieren, Präfix für Umgebungsvariablen. Wirksame Konfiguration anzeigbar, Geheimnisse maskiert. Eine Datei pro Umgebung, gemeinsame Werte in einer Basis.

> **Praxisbeispiel**
>
> Ein Backup-Skript läuft auf Entwicklungs-, Test- und Produktionsservern. Es liest `/etc/backup/basis.env` und `/etc/backup/$APP_ENV.env`; für einen einzelnen Testlauf genügt `APP_RETENTION=1` oder `-{}-retention 1`. `backup.sh --print-config` zeigt in jedem Ticket, mit welchen Werten ein Lauf stattfand.

## 10.8 Automationsstandards

Je mehr Skripte ein Team betreibt, desto wichtiger wird Einheitlichkeit. Standards sparen Einarbeitungszeit, machen Reviews schneller, verhindern wiederkehrende Fehler und erlauben es, Prüfungen zu automatisieren. Ein Standard besteht aus fünf Teilen. Erstens *Namen und Ablage*: Skripte heißen nach dem Muster `verb-objekt.sh` in Kleinbuchstaben mit Bindestrich (`backup-datenbank.sh`), liegen in festen Verzeichnissen (`bin/`, `lib/`, `tests/`, `docs/`) und haben immer ein README. Zweitens ein *Skript-Gerüst* (Template) mit Shebang, Kopfkommentar, `set -euo pipefail`, `usage`, Argumentverarbeitung, Logging, `main`, `trap`-Aufräumen (Module 3, 6, 9). Drittens ein *Stilleitfaden*: Einrückung, Quoting-Regeln, Namenskonventionen für Variablen (Konstanten in Großbuchstaben, lokale in Kleinbuchstaben), `[[ ]]` statt `[ ]`, `$(...)` statt Backticks, `local` in Funktionen. Viertens *automatische Prüfung*: ShellCheck (Fehler und Stolperfallen), `shfmt` (einheitliche Formatierung, wo verfügbar), `bash -n` (Syntax) und Tests; dies läuft in Pre-Commit-Hook und CI (10.3, 10.4). Fünftens *Review und Freigabe*: Änderungen an Produktionsskripten gehen über Pull Requests mit mindestens einer Prüfung nach einer Checkliste und werden versioniert veröffentlicht (Modul 12.7).

Ein praktischer Hebel ist ein *einheitlicher Prüfbefehl*, den Entwickler lokal und die CI gleich aufrufen (`make lint`, `./ci/lint.sh`). Ein zweiter Hebel sind *ShellCheck-Direktiven* (`# shellcheck disable=SC2034 # Begründung`): Unterdrückungen sind erlaubt, aber nur einzeln und mit Begründung. Ein dritter sind *Mindestanforderungen* an jedes Skript, die ein Linter prüft: Shebang vorhanden, strenger Modus gesetzt, Ausführrecht korrekt, keine Windows-Zeilenenden, kein `eval` ohne Ausnahmevermerk, Hilfetext vorhanden.

```bash
#!/usr/bin/env bash
# lint-alle.sh [VERZ]  - einheitlicher Prueflauf: Regeln des Teams + ShellCheck
set -uo pipefail
verz="${1:-.}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }
fehler=0
melde() { printf '%s: %s\n' "$1" "$2"; fehler=$((fehler + 1)); }

while IFS= read -r -d '' f; do
    # Regel 1: Shebang mit env bash (oder sh bei reinen POSIX-Skripten)
    erste="$(head -n 1 "$f")"
    [[ $erste == '#!/usr/bin/env bash' || $erste == '#!/bin/sh' ]] || melde "$f" "Shebang soll '#!/usr/bin/env bash' sein"
    # Regel 2: strenger Modus (nur fuer bash-Skripte)
    if [[ $erste == *bash* ]]; then
        grep -Eq '^set -[A-Za-z]*e[A-Za-z]*u|^set -euo pipefail|^set -uo pipefail' "$f" || melde "$f" "kein strenger Modus (set -euo pipefail)"
    fi
    # Regel 3: kein CRLF
    grep -q $'\r' "$f" && melde "$f" "CRLF-Zeilenenden"
    # Regel 4: eval nur mit Ausnahmevermerk
    if grep -Eq '(^|[;&|[:space:]])eval[[:space:]]' "$f" && ! grep -q 'lint-ausnahme: eval' "$f"; then
        melde "$f" "eval ohne Vermerk '# lint-ausnahme: eval <Begruendung>'"
    fi
    # Regel 5: ShellCheck, falls installiert
    if command -v shellcheck >/dev/null; then
        sc="$(shellcheck -S warning -f gcc "$f" 2>&1)" || melde "$f" "ShellCheck: $(head -n 1 <<< "$sc" | sed 's/^[^ ]* //')"
    fi
done < <(find "$verz" -type f -name '*.sh' -not -path '*/.git/*' -print0 | sort -z)

if (( fehler > 0 )); then echo "$fehler Verstoss/Verstoesse"; exit 1; fi
echo "Alle Pruefungen bestanden"
```

Das Prüfskript formuliert die Teamregeln als Code, damit niemand sie auswendig kennen muss. Jede Regel hat eine klare Meldung mit Dateiname, sodass sich Verstöße schnell beheben lassen. Die Ausnahme für `eval` zeigt ein wichtiges Muster: Regeln dürfen Ausnahmen zulassen, aber nur *ausdrücklich dokumentiert* im Code, wo sie im Review sichtbar werden. ShellCheck läuft im Format `gcc` (eine Zeile pro Befund), das sich leicht zusammenfassen lässt. Der Aufruf liefert den Exit-Code 1 bei Verstößen und lässt sich daher direkt als Schritt in Pre-Commit-Hook und CI (10.3, 10.4) einsetzen.

Typische Fehler: Erstens gibt es einen Leitfaden, den niemand liest und nichts erzwingt; ohne Automatisierung bleibt er Papier. Zweitens ist er so streng oder detailverliebt, dass Teams ihn umgehen. Drittens ShellCheck-Warnungen werden pauschal abgeschaltet (`-e SC2086`) statt einzeln begründet. Viertens Skripte liegen verstreut (Home-Verzeichnisse, Wikis, E-Mails) statt im Repository. Fünftens kein Eigentümer: Niemand ist für ein Skript zuständig. Sechstens keine Versionierung und keine Änderungshistorie. Siebtens Standards werden einmal beschlossen und nie angepasst; sie sollten sich mit den Erfahrungen aus Störungen weiterentwickeln (Modul 12.9).

```bash
# falsch: Warnungen global abschalten, Ausnahmen ohne Begruendung
shellcheck -e SC2086,SC2155,SC2046 *.sh
# shellcheck disable=SC2086

# richtig: einzeln, begruendet, nah an der Stelle
# shellcheck disable=SC2086  # Wortaufspaltung ist hier gewollt: $FLAGS enthaelt mehrere Optionen
rsync $FLAGS -- "$quelle" "$ziel"
```

> **Merke**
>
> Standards: Namen und Ablage, Skript-Gerüst, Stilleitfaden, automatische Prüfung (ShellCheck, `bash -n`, Linter-Skript), Review und Freigabe. Ein Prüfbefehl für Entwickler und CI. Ausnahmen einzeln und begründet. Jedes Skript hat Eigentümer, README, Versionierung.

> **Praxisbeispiel**
>
> Ein neues Teammitglied kopiert das Gerüst `vorlage.sh`, füllt Zweck und Parameter aus und ruft `./ci/lint-alle.sh` auf; erst wenn es grün ist, eröffnet es den Pull Request. Das Review konzentriert sich dadurch auf Logik und Sicherheit statt auf Formalien.

## 10.9 Übungsaufgaben zu Modul 10

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Was bedeutet die Crontab-Zeile `15 3 * * 1,4` und warum ist `0 4 * * * tar czf /b/x-$(date +%F).tgz /srv` fehlerhaft? Nennen Sie drei Unterschiede zwischen Cron und systemd Timer.

**Aufgabe 2.** Beschreiben Sie das Zustandsmodell eines Batch-Jobs mit Verzeichnissen. Wie sorgen Sie für Wiederanlauf nach einem Absturz, und wozu dient `mv -n` als „Claim“?

**Aufgabe 3.** Warum sind clientseitige Git-Hooks keine Sicherheitsgrenze? Welchen Inhalt prüft ein guter `pre-commit`-Hook, und wie verteilen Sie Hooks an ein Team?

**Aufgabe 4.** Erklären Sie, warum ein Deployment mit Symlink-Wechsel atomar ist, welche Rolle `mv -T` spielt und wie ein Rollback abläuft. Was ist bei Datenbank-Migrationen zu beachten?

**Aufgabe 5.** Schreiben Sie `lauf-geschuetzt.sh NAME BEFEHL...`, das mit `flock` verhindert, dass derselbe NAME zweimal gleichzeitig läuft (Exit-Code 75), ein Zeitlimit aus der Variable `LIMIT` (Standard 60 s) setzt, Start und Ende mit Zeitstempel loggt und den Exit-Code des Befehls zurückgibt.

**Aufgabe 6.** Schreiben Sie `parallel-pruefen.sh LISTE`, das für jede URL aus einer Datei (eine pro Zeile) mit `curl` den HTTP-Statuscode abfragt, höchstens 4 Abfragen gleichzeitig durchführt (`xargs -P`), je URL eine Zeile `STATUS URL` ausgibt und mit Exit-Code 1 endet, wenn mindestens eine URL keinen 2xx- oder 3xx-Code liefert. Zum Testen genügt ein lokaler Webserver.

**Aufgabe 7.** Schreiben Sie `lade-konfig.sh DATEI`, das eine `KEY=wert`-Datei sicher parst (Allowlist `HOST`, `PORT`, `MODUS`), Umgebungsvariablen mit Präfix `APP_` darüber legt, `PORT` auf den Bereich 1 bis 65535 prüft und die wirksame Konfiguration mit Herkunft (default, datei, env) je Schlüssel ausgibt.

**Aufgabe 8.** Schreiben Sie ein `commit-msg`-Hook-Skript, das die Commit-Meldung (Datei als Argument) prüft: erste Zeile 10 bis 72 Zeichen, beginnt mit einem der Präfixe `feat:`, `fix:`, `docs:`, `chore:`, danach (falls weitere Zeilen existieren) eine Leerzeile. Bei Verstoß Exit-Code 1 mit verständlicher Meldung. Testen Sie es in einem Test-Repository mit `core.hooksPath`.

## 10.10 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** `15 3 * * 1,4`: um 03:15 Uhr am Montag und Donnerstag (Wochentag 1 und 4). Die zweite Zeile ist fehlerhaft, weil `%` in der Crontab einen Zeilenumbruch bedeutet; `date +%F` muss als `date +\%F` geschrieben werden, sonst wird der Befehl abgeschnitten. Außerdem fehlen absolute Pfade, Sperre und Ausgabebehandlung. Unterschiede: Timer-Ausgaben landen im Journal, Cron-Ausgaben per Mail; Timer holen mit `Persistent=true` verpasste Läufe nach, Cron nicht; Timer-Dienste können Härtung und Ressourcenlimits nutzen und zeigen sich in `systemctl list-timers`; Cron hat eine kleinere, einfachere Syntax und ist überall vorhanden.

**Lösung zu Aufgabe 2.** Verzeichnisse `eingang`, `arbeit`, `fertig`, `fehler`; jeder Zustandswechsel ist ein `mv` innerhalb eines Dateisystems (atomar). Wiederanlauf: Zu Beginn werden Reste aus `arbeit/` zurück in `eingang/` verschoben, denn sie stammen aus einem abgebrochenen Lauf. `mv -n` überschreibt keine bestehende Datei und schlägt fehl, wenn das Ziel schon existiert; wer die Datei als Erster nach `arbeit/` verschiebt, hat sie „beansprucht“, ein zweiter Prozess bekommt sie nicht mehr. Ergänzend: Wiederholungen begrenzen und Fehlerberichte in `fehler/` ablegen.

**Lösung zu Aufgabe 3.** Sie liegen lokal, werden nicht mit dem Repository übertragen und lassen sich mit `-{}-no-verify` übergehen; verbindliche Prüfungen gehören in CI und Server. Ein guter `pre-commit`-Hook prüft den *gestageten* Inhalt (`git show :datei`), nicht die Arbeitskopie, und nur die geänderten Dateien (`git diff --cached --name-only --diff-filter=ACM`). Verteilung: Hooks im Repository ablegen (`.githooks/`) und einmalig `git config core.hooksPath .githooks` setzen, gegebenenfalls über ein Setup-Skript oder ein Werkzeug wie `pre-commit`.

**Lösung zu Aufgabe 4.** Ein Symlink-Wechsel besteht aus dem Umbiegen eines einzelnen Verzeichniseintrags; Leser sehen entweder den alten oder den neuen Zeiger, nie einen Zwischenzustand. Das einfache `ln -sfn` ist zweistufig (entfernen, anlegen), `mv -T neu current` (mit `-T`, damit `current` als Datei und nicht als Zielverzeichnis behandelt wird) ersetzt den Eintrag dagegen atomar (`rename`). Rollback: Link zurück auf das gemerkte vorige Release, Dienst neu starten, fehlerhaftes Release entfernen oder zur Analyse behalten. Migrationen: abwärtskompatibel gestalten (additiv, Entfernen erst in einem späteren Release), weil der Rollback des Codes die Datenbank nicht zurücksetzt.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# lauf-geschuetzt.sh NAME BEFEHL...
set -uo pipefail
name="${1:?Aufruf: $0 NAME BEFEHL...}"; shift
(( $# >= 1 )) || { echo "BEFEHL fehlt" >&2; exit 2; }
[[ $name =~ ^[a-z0-9_-]+$ ]] || { echo "ungueltiger NAME" >&2; exit 2; }
limit="${LIMIT:-60}"
[[ $limit =~ ^[1-9][0-9]*$ ]] || { echo "LIMIT ungueltig" >&2; exit 2; }

log() { printf '%s %s %s\n' "$(date -u +%FT%TZ)" "$name" "$*" >&2; }

exec 9> "${TMPDIR:-/tmp}/lauf-$name.lock"
flock -n 9 || { log "uebersprungen: laeuft bereits"; exit 75; }

log "Start: $*"
timeout --kill-after=5 "$limit" "$@"
rc=$?
if (( rc == 124 )); then log "Zeitlimit von ${limit}s ueberschritten"
else log "Ende mit Exit-Code $rc"; fi
exit "$rc"
```

Die Sperre liegt auf dem Dateideskriptor 9 und wird beim Prozessende automatisch freigegeben, auch bei einem Absturz. Die Meldungen gehen nach stderr, damit stdout dem Befehl gehört. Der Exit-Code 124 stammt von `timeout` und wird im Log erklärt. Test: Zwei Aufrufe mit `sleep 5` kurz nacheinander; der zweite endet mit 75.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# parallel-pruefen.sh LISTE
set -uo pipefail
liste="${1:?Aufruf: $0 LISTE}"
[[ -r $liste ]] || { echo "Liste nicht lesbar" >&2; exit 2; }

pruefe_url() {
    local url="$1" code
    code="$(curl -sS -o /dev/null --max-time 5 -w '%{http_code}' -- "$url" 2>/dev/null)" || code=000
    printf '%s %s\n' "$code" "$url"
}
export -f pruefe_url

ergebnis="$(grep -v '^[[:space:]]*\(#\|$\)' "$liste" | tr '\n' '\0' |
            xargs -0 -P 4 -I{} bash -c 'pruefe_url "$1"' _ {})"
printf '%s\n' "$ergebnis" | sort -k2
schlecht="$(printf '%s\n' "$ergebnis" | awk '$1 !~ /^[23][0-9][0-9]$/' | wc -l)"
(( schlecht == 0 )) || { echo "$schlecht URL(s) ohne 2xx/3xx" >&2; exit 1; }
```

Leerzeilen und Kommentare werden entfernt, die URLs nullbyte-getrennt an `xargs -P 4` übergeben und von einer exportierten Funktion bearbeitet. `curl -w '%{http_code}'` liefert nur den Statuscode; bei Verbindungsfehlern ergibt sich `000`. Die Ergebnisse werden in einer Variable gesammelt (so gehen die Zeilen nicht in der Subshell verloren), sortiert ausgegeben und gezählt. Test mit `python3 -m http.server 8099`: `http://localhost:8099/` liefert 200, `http://localhost:8099/nix` liefert 404.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# lade-konfig.sh DATEI
set -euo pipefail
datei="${1:?Aufruf: $0 DATEI}"
ERLAUBT=(HOST PORT MODUS)
declare -A WERT=([HOST]=localhost [PORT]=8080 [MODUS]=dev)
declare -A QUELLE=([HOST]=default [PORT]=default [MODUS]=default)

while IFS= read -r zeile || [[ -n $zeile ]]; do
    [[ -z $zeile || $zeile == \#* ]] && continue
    if [[ $zeile =~ ^([A-Z_][A-Z0-9_]*)=(.*)$ ]]; then
        k="${BASH_REMATCH[1]}"; v="${BASH_REMATCH[2]}"
        if [[ " ${ERLAUBT[*]} " == *" $k "* ]]; then WERT[$k]="$v"; QUELLE[$k]=datei
        else echo "ignoriert (unbekannt): $k" >&2; fi
    fi
done < "$datei"

for k in "${ERLAUBT[@]}"; do
    name="APP_$k"
    if [[ -n ${!name:-} ]]; then WERT[$k]="${!name}"; QUELLE[$k]=env; fi
done

[[ ${WERT[PORT]} =~ ^[0-9]+$ ]] && (( 10#${WERT[PORT]} >= 1 && 10#${WERT[PORT]} <= 65535 )) \
    || { echo "PORT ausserhalb 1..65535: ${WERT[PORT]}" >&2; exit 1; }

for k in "${ERLAUBT[@]}"; do printf '%-6s = %-12s (%s)\n' "$k" "${WERT[$k]}" "${QUELLE[$k]}"; done
```

Der Test: Datei mit `HOST=db.example` und `EVIL=$(touch x)`; die Zeile wird mit Hinweis ignoriert und nie ausgeführt, weil nicht `source` verwendet wird. Ein Aufruf mit `APP_PORT=99999` endet mit Fehler, `APP_PORT=9000` ergibt die Herkunft `env`. Die Zahlenprüfung wertet mit `10#` dezimal aus, damit führende Nullen nicht als Oktalzahl gelten.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# .githooks/commit-msg DATEI
set -uo pipefail
datei="${1:?}"
erste="$(head -n 1 "$datei")"
laenge=${#erste}
fehler() { echo "Commit-Meldung abgelehnt: $*" >&2; echo "Beispiel: fix: Rechte der Sperrdatei korrigieren" >&2; exit 1; }

(( laenge >= 10 && laenge <= 72 )) || fehler "erste Zeile hat $laenge Zeichen (erlaubt: 10 bis 72)"
[[ $erste =~ ^(feat|fix|docs|chore):[[:space:]]+[^[:space:]] ]] || fehler "Praefix feat:, fix:, docs: oder chore: fehlt"
zweite="$(sed -n '2p' "$datei")"
if [[ $(grep -vc '^#' "$datei") -gt 1 && -n $zweite && ${zweite:0:1} != '#' ]]; then
    fehler "zweite Zeile muss leer sein"
fi
```

Der Hook erhält den Dateinamen der Commit-Meldung als erstes Argument. Die Länge wird mit `${#erste}` bestimmt, das Präfix mit einem regulären Ausdruck geprüft. Zeilen, die mit `#` beginnen, sind Kommentare von Git und zählen nicht. Im Test-Repository: `git config core.hooksPath .githooks`, dann `git commit -m "kaputt"` (abgelehnt, zu kurz und ohne Präfix) und `git commit -m "fix: Rechte der Sperrdatei korrigieren"` (angenommen).
