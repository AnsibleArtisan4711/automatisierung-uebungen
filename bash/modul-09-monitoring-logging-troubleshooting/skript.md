# Modul 09 - Monitoring, Logging und Troubleshooting

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (9.1 bis 9.8; 9.9 Übungsaufgaben, 9.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 10 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (8 Einheiten à 1,25 h). Beide Angaben stimmen überein.  
**Priorität:** 2 (mittel)

| Nr. | Lerneinheit                 | Dauer (h) | Prio |
|:----|:----------------------------|----------:|:----:|
| 9.1 | Skript-Logging Standards    |      1,25 |  2   |
| 9.2 | Exit Codes und Healthchecks |      1,25 |  2   |
| 9.3 | Metriken aus Bash           |      1,25 |  2   |
| 9.4 | Alerting-Integration        |      1,25 |  2   |
| 9.5 | journalctl und syslog       |      1,25 |  2   |
| 9.6 | Troubleshooting Methodik    |      1,25 |  2   |
| 9.7 | Typische Fehlerbilder       |      1,25 |  3   |
| 9.8 | Runbooks                    |      1,25 |  3   |
|     | **Summe**                   | **10,00** |      |

Ein Skript, das nachts unbeobachtet läuft, ist nur so gut wie das, was man über sein Verhalten *erfährt*. Dieses Modul behandelt die drei Säulen der Beobachtbarkeit (Observability) aus der Sicht eines Bash-Skripts: *Logs* (was ist passiert?), *Metriken* (wie viel und wie lange?) und *Alarme* (wer muss jetzt etwas tun?). Dazu kommt die Kunst, Fehler systematisch zu finden (Troubleshooting) und das Wissen für den Ernstfall festzuhalten (Runbooks). Die Beispiele schließen an Modul 6.5 (Logging-Bibliothek), 7 (Systemadministration) und 8.8 (Auditierbarkeit) an; wo es Überschneidungen gibt, wird verwiesen.

## 9.1 Skript-Logging Standards

In Modul 6.5 haben Sie eine einfache Logging-Bibliothek gebaut. Hier geht es um die *Standards*, die aus Logzeilen auswertbare Daten machen. Eine Logzeile hat drei Aufgaben: Sie muss für einen *Menschen* lesbar sein (der nachts nach der Ursache sucht), für ein *Programm* auswertbar sein (grep, awk, jq oder ein Log-System wie Loki oder Elasticsearch) und *vollständig genug*, um ohne weiteren Kontext verstanden zu werden. Daraus folgen feste Bestandteile: ein *Zeitstempel* in UTC nach ISO 8601, ein *Level* (DEBUG, INFO, WARN, ERROR), der *Name des Skripts oder der Komponente*, eine *Meldung* und, wo sinnvoll, *Schlüssel-Wert-Paare* für Details (Datei, Anzahl, Dauer, Exit-Code).

Zwei Formate sind verbreitet. Das *logfmt*-Format schreibt Paare als `key=value`, durch Leerzeichen getrennt (Werte mit Leerzeichen in Anführungszeichen); es ist gut lesbar und mit `grep` und `awk` auswertbar. *JSON Lines* (eine JSON-Zeile pro Ereignis) ist maschinenfreundlich und wird von fast allen Log-Systemen direkt verstanden; in Bash erzeugt man es am sichersten mit `jq -n -c --arg ...`, weil `jq` Sonderzeichen korrekt maskiert (selbstgebautes JSON mit `printf` scheitert an Anführungszeichen, Backslashes und Zeilenumbrüchen). Wichtig ist die *Konsistenz*: Alle Skripte eines Teams verwenden dieselben Feldnamen.

Das zweite Standardthema ist die *Korrelation*. Wenn dasselbe Skript parallel mehrfach läuft oder ein Aufruf mehrere Systeme berührt, braucht jede Zeile eine *Run-ID* (oder Correlation-ID): eine eindeutige Kennung, die der Lauf einmal erzeugt und in jede Zeile schreibt. Damit lassen sich alle Zeilen eines Laufs mit einem `grep` herausfiltern, auch wenn sie zwischen anderen stehen. Unter Linux liefert `cat /proc/sys/kernel/random/uuid` eine UUID; für Kurzform genügt `$(date +%s)-$$`. Reicht ein Skript die Run-ID als Umgebungsvariable (`RUN_ID`) an Kindprozesse und als HTTP-Header (`X-Request-ID`) an Dienste weiter, entsteht eine durchgängige Spur über Systemgrenzen.

Schließlich gehört zu einem Standard die Frage der *Kanäle*: Diagnose und Logs gehen nach *stderr*, Ergebnisse nach *stdout* (Modul 4.1); sonst vermischen sich Daten und Meldungen in Pipelines. Bei Diensten schreibt man Logs einfach nach stderr oder stdout und überlässt `systemd`/`journald` die Speicherung; bei Cron und CI sammelt der Scheduler die Ausgabe. Dateien werden nur geschrieben, wenn es einen Grund gibt, und dann mit Rotation (Modul 7.4).

```bash
#!/usr/bin/env bash
# logging-standard.sh - logfmt und JSON Lines mit Run-ID, Level-Filter und sicheren Werten
set -euo pipefail

SKRIPT="${0##*/}"
RUN_ID="${RUN_ID:-$(cat /proc/sys/kernel/random/uuid | cut -c1-8)}"   # Kindprozesse erben die ID
export RUN_ID
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FORMAT="${LOG_FORMAT:-logfmt}"                 # logfmt | json
declare -A LVL=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40)

# Werte fuer logfmt: bei Leerzeichen, Anfuehrungszeichen oder '=' in Quotes setzen
lf_wert() {
    local v="$1"
    if [[ $v =~ [[:space:]\"=] || -z $v ]]; then
        v="${v//\\/\\\\}"; v="${v//\"/\\\"}"; v="${v//$'\n'/\\n}"
        printf '"%s"' "$v"
    else printf '%s' "$v"; fi
}

log() {                                            # log LEVEL MELDUNG [key=value ...]
    local level="$1" msg="$2"; shift 2
    (( ${LVL[$level]} >= ${LVL[$LOG_LEVEL]:-20} )) || return 0
    local zeit; zeit="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    if [[ $LOG_FORMAT == json ]]; then
        local args=(--arg ts "$zeit" --arg level "$level" --arg skript "$SKRIPT" \
                    --arg run_id "$RUN_ID" --arg msg "$msg")
        local filter='{ts:$ts,level:$level,skript:$skript,run_id:$run_id,msg:$msg'
        local kv k v i=0
        for kv in "$@"; do
            k="${kv%%=*}"; v="${kv#*=}"
            args+=(--arg "v$i" "$v"); filter+=",\"$k\":\$v$i"; i=$((i + 1))
        done
        jq -n -c "${args[@]}" "$filter}" >&2
    else
        local zeile="ts=$zeit level=$level skript=$SKRIPT run_id=$RUN_ID msg=$(lf_wert "$msg")"
        for kv in "$@"; do zeile+=" ${kv%%=*}=$(lf_wert "${kv#*=}")"; done
        printf '%s\n' "$zeile" >&2
    fi
}

# --- Vorfuehrung
log INFO  "Lauf gestartet" pid=$$
log DEBUG "wird bei INFO nicht gezeigt"
log WARN  "Datei fehlt" datei="mein bericht.txt" versuch=2
log ERROR "Eingabe mit \"Anfuehrungszeichen\" und = Zeichen" wert='a="b"'
echo "--- JSON Lines (gleiche Ereignisse):"
export LOG_FORMAT=json
log INFO "Lauf gestartet" pid=$$
log ERROR 'Fehler mit "Quotes" und Zeilenumbruch' ursache=$'zeile1\nzeile2'
echo "--- Kindprozess erbt die Run-ID:"
bash -c 'echo "Kindprozess sieht RUN_ID=$RUN_ID"'
```

Die Funktion `log` nimmt Level, Meldung und beliebig viele `key=value`-Paare. Sie filtert zuerst nach dem Mindestlevel, über das assoziative Array `LVL` (Modul 3.7). Im logfmt-Zweig sorgt `lf_wert` dafür, dass Werte mit Leerzeichen, Anführungszeichen oder Gleichheitszeichen in Anführungszeichen stehen und innere Anführungszeichen und Zeilenumbrüche maskiert werden, damit die Zeile eindeutig zerlegbar bleibt. Im JSON-Zweig baut die Funktion einen `jq`-Filter auf und übergibt *jeden Wert als `-{}-arg`*, nie eingebettet in den Filtertext: Das ist die „Daten statt Code“-Regel aus Modul 8.3 in neuer Form, und sie macht Zeilenumbrüche und Anführungszeichen im Wert unschädlich. Die Run-ID entsteht nur, wenn die Variable noch nicht gesetzt ist; ruft ein Skript ein anderes auf, wird die ID geerbt und die gesamte Aufrufkette lässt sich gemeinsam auswerten. Der Kindprozess am Ende beweist es.

Typische Fehler: Erstens Logzeilen als Freitext ohne feste Struktur („Habe 5 Dateien kopiert!!!“), die niemand maschinell auswerten kann. Zweitens JSON von Hand mit `echo "{\"msg\":\"$msg\"}"` gebaut: Das erzeugt ungültiges JSON, sobald der Text ein Anführungszeichen enthält, und ist zugleich eine Injektionsstelle für gefälschte Felder. Drittens Zeitstempel in lokaler Zeit ohne Zeitzone oder in uneinheitlichem Format. Viertens fehlt die Run-ID, und bei parallelen Läufen kann man Zeilen nicht mehr zuordnen. Fünftens Logs nach stdout, wo sie die Ergebnisse der Skripte verschmutzen. Sechstens werden Geheimnisse und personenbezogene Daten protokolliert (Modul 8.4 und 8.9). Siebtens „Log-Spam“: Eine Meldung pro verarbeitetem Datensatz bei Millionen Datensätzen verstopft das System; aggregiert man (Zwischenstände alle N Einträge, Zusammenfassung am Ende), bleibt das Log lesbar.

```bash
# falsch: Freitext, selbstgebautes JSON, lokale Zeit, stdout
echo "$(date) ERROR: Datei $datei konnte nicht gelesen werden!!!"
echo "{\"msg\":\"$meldung\",\"ts\":\"$(date)\"}"

# richtig: feste Felder, UTC, Werte als Daten an jq, Kanal stderr
jq -n -c --arg ts "$(date -u +%FT%TZ)" --arg msg "$meldung" --arg datei "$datei" \
   '{ts:$ts,level:"ERROR",msg:$msg,datei:$datei}' >&2
```

> **Merke**
>
> Eine Logzeile hat Zeit (UTC, ISO 8601), Level, Komponente, Meldung und Schlüssel-Wert-Paare. Format einheitlich: logfmt oder JSON Lines, JSON nur mit `jq -{}-arg`. Eine Run-ID verbindet alle Zeilen eines Laufs. Logs nach stderr, Ergebnisse nach stdout; keine Geheimnisse; Log-Spam vermeiden.

> **Praxisbeispiel**
>
> Ein nächtlicher Import verarbeitet 200 000 Datensätze. Das Skript meldet pro Lauf Start, alle 10 000 Datensätze einen Zwischenstand und am Ende eine Zusammenfassung mit Dauer, Anzahl und Fehlern, alles mit derselben Run-ID. Bei einer Störung findet der Bereitschaftsdienst mit `grep run_id=3f9a1c2b` genau die Zeilen dieses Laufs.

## 9.2 Exit Codes und Healthchecks

Der Exit-Code ist die kleinste, aber wichtigste Schnittstelle eines Skripts: Scheduler (Cron, systemd), CI-Systeme und andere Skripte entscheiden allein anhand dieser Zahl, ob ein Lauf erfolgreich war. Die Grundregel kennen Sie: `0` bedeutet Erfolg, alles andere Fehler. Doch ein *einziger* Fehlercode wirft Information weg. Besser ist ein *Vertrag*: Jedes Skript dokumentiert seine Exit-Codes (Modul 6.9), und die Zahlen sind *stabil*. Konventionen helfen: `1` allgemeiner Fehler, `2` falsche Benutzung (wie bei vielen GNU-Werkzeugen), `64` bis `78` die aus `sysexits.h` bekannten Codes (`64` Benutzungsfehler, `69` Dienst nicht verfügbar, `73` Ausgabedatei nicht anlegbar, `78` Konfigurationsfehler). Reserviert sind `126` (Befehl gefunden, aber nicht ausführbar), `127` (nicht gefunden), `124` (von `timeout`) und `128+n` für Beendigung durch Signal *n* (also `130` für Ctrl-C, `137` für `SIGKILL`, `143` für `SIGTERM`); eigene Codes sollten diese Bereiche meiden. Exit-Codes sind auf 0 bis 255 begrenzt: `exit 256` wird zu 0, was ein klassischer Fehler ist.

Ein *Healthcheck* ist ein kleines, schnelles Skript, das den Zustand eines Dienstes oder Systems prüft und das Ergebnis über den Exit-Code (und eine kurze Meldung) liefert. Verbreitet ist die *Nagios-Plugin-Konvention*, die viele Monitoring-Systeme verstehen: `0` OK, `1` WARNING, `2` CRITICAL, `3` UNKNOWN, dazu eine Textzeile (optional mit Messwerten nach einem `|`). Auch Container-Orchestrierung nutzt das Muster (Docker `HEALTHCHECK`, Kubernetes Liveness- und Readiness-Probes). Gute Healthchecks sind *schnell* (Zeitgrenze), *ohne Nebenwirkungen* (nur lesen), *aussagekräftig* (prüfen die echte Funktion, nicht nur, dass der Prozess läuft) und *eindeutig* (unterscheiden „kaputt“ von „kann nicht prüfen“, also CRITICAL von UNKNOWN).

Bei der Standardisierung hilft ein gemeinsamer Rahmen: Jeder Check ist eine Funktion, die einen Status und eine Meldung liefert; ein kleiner Treiber führt alle Checks mit Zeitgrenze aus, sammelt die Ergebnisse, gibt eine Zusammenfassung aus und leitet den *schlechtesten* Status als Gesamt-Exit-Code ab. So lassen sich neue Checks hinzufügen, ohne die Auswertung zu ändern.

```bash
#!/usr/bin/env bash
# healthcheck.sh - Checks als Funktionen, schlechtester Status gewinnt (0 OK, 1 WARN, 2 CRIT, 3 UNKNOWN)
set -uo pipefail
OK=0; WARN=1; CRIT=2; UNKNOWN=3
statusname=(OK WARNING CRITICAL UNKNOWN)
worst=$OK
declare -a ergebnis

# --- Treiber: fuehrt einen Check mit Zeitgrenze aus und wertet den Rueckgabecode
fuehre_aus() {                                       # fuehre_aus NAME FUNKTION [ARGUMENTE...]
    local name="$1" fn="$2"; shift 2
    local out rc
    out="$(timeout 5 bash -c "$(declare -f "$fn" _ok _warn _crit); $fn \"\$@\"" _ "$@" 2>&1)"; rc=$?
    case $rc in
        0|1|2|3) ;;
        124)     rc=$UNKNOWN; out="Zeitueberschreitung (5 s)" ;;
        *)       rc=$UNKNOWN; out="Check selbst fehlerhaft (rc=$rc): $out" ;;
    esac
    ergebnis+=("$(printf '%-9s %-14s %s' "${statusname[$rc]}" "$name" "$out")")
    # schlechtester Status: CRIT > WARN > UNKNOWN > OK  (UNKNOWN zaehlt als schlimmer als OK, besser als CRIT)
    case "$worst:$rc" in
        *:2|2:*) worst=2 ;;
        *:1) [[ $worst == 3 ]] || worst=1 ;;
        *:3) [[ $worst == 1 ]] || worst=3 ;;
    esac
}
_ok()   { echo "$*"; return 0; }
_warn() { echo "$*"; return 1; }
_crit() { echo "$*"; return 2; }

# --- Einzelne Checks
check_platz() {              # PFAD WARN CRIT in Prozent belegt
    local p; p="$(df -P -- "$1" | awk 'NR==2 {gsub("%","",$5); print $5}')" || return 3
    [[ $p =~ ^[0-9]+$ ]] || { echo "df liefert keinen Wert"; return 3; }
    (( p >= $3 )) && { _crit "$1 zu ${p}% belegt"; return 2; }
    (( p >= $2 )) && { _warn "$1 zu ${p}% belegt"; return 1; }
    _ok "$1 zu ${p}% belegt (ok)"
}
check_port() {               # HOST PORT
    timeout 2 bash -c ": < /dev/tcp/$1/$2" 2>/dev/null && _ok "$1:$2 erreichbar" \
        || { _crit "$1:$2 nicht erreichbar"; return 2; }
}
check_datei_alter() {        # DATEI MAX_MINUTEN  (z.B. Heartbeat/Backup)
    [[ -e $1 ]] || { echo "$1 fehlt"; return 2; }
    [[ -n $(find "$1" -mmin "-$2") ]] && _ok "$1 aktuell" || { _warn "$1 aelter als $2 min"; return 1; }
}
check_haengt() { sleep 10; }

touch /tmp/heartbeat.$$
fuehre_aus "platz-root"  check_platz / 80 95
fuehre_aus "port-http"   check_port localhost 1        # nichts lauscht: CRITICAL
fuehre_aus "heartbeat"   check_datei_alter /tmp/heartbeat.$$ 5
fuehre_aus "haengender"  check_haengt
rm -f /tmp/heartbeat.$$

printf '%s\n' "${ergebnis[@]}"
echo "Gesamtstatus: ${statusname[$worst]}"
exit "$worst"
```

Der Treiber `fuehre_aus` führt jede Check-Funktion in einem eigenen Prozess mit `timeout 5` aus. Damit die Funktion dort bekannt ist, wird sie mit `declare -f` als Text mitgegeben (ein bewährter Trick, um Bash-Funktionen in einer Subshell mit Zeitgrenze zu starten). Der Exit-Code wird ausgewertet: 0 bis 3 sind gültige Status, 124 bedeutet Zeitüberschreitung und wird als UNKNOWN gewertet (der Check konnte nicht entscheiden, das System ist nicht zwingend kaputt), alle anderen Codes deuten auf einen fehlerhaften Check hin. Die Ergebnisse stehen untereinander, am Ende steht der Gesamtstatus als Exit-Code. Die Reihenfolge der Schwere (CRITICAL vor WARNING vor UNKNOWN vor OK) ist eine Festlegung, die man dokumentieren sollte; manche Teams stufen UNKNOWN höher ein. Das Beispiel zeigt alle Fälle: ein erfolgreicher Check, ein kritischer, ein Heartbeat-Check (die Datei ist frisch, daher OK) und ein hängender Check, der durch die Zeitgrenze abgefangen wird.

Typische Fehler: Erstens meldet jedes Problem denselben Exit-Code 1, sodass der Betrieb nicht zwischen „Platte voll“ und „Konfig fehlt“ unterscheiden kann. Zweitens läuft das Skript bei einem Fehler im Check selbst auf Exit-Code 1 oder 2, und das Monitoring behauptet, der Dienst sei kritisch, obwohl nur der Check kaputt ist (UNKNOWN hätte gestimmt). Drittens fehlen Zeitgrenzen, und ein hängender Check blockiert den Scheduler. Viertens prüft ein Healthcheck nur, ob der Prozess läuft, nicht ob der Dienst funktioniert (Port offen, Anfrage beantwortet, Datenbank erreichbar). Fünftens haben Healthchecks Nebenwirkungen (schreiben Daten, lösen Aktionen aus) oder sind so teuer, dass sie die Last erhöhen. Sechstens `exit $anzahl_fehler` mit mehr als 255 Fehlern ergibt `exit 0`, weil der Wert modulo 256 gerechnet wird. Siebtens werden Exit-Codes nach `set -e` und Pipes ignoriert (Modul 6.1): `cmd | tee log` liefert den Status von `tee`, wenn `pipefail` fehlt.

```bash
# falsch: Anzahl der Fehler als Exit-Code (256 Fehler = Erfolg), alles ist "1"
exit "$fehler_anzahl"
[[ -f $konfig ]] || exit 1; ping -c1 db || exit 1

# richtig: Anzahl nur begrenzen und feste, dokumentierte Codes verwenden
(( fehler_anzahl > 0 )) && exit 1
[[ -f $konfig ]] || exit 78        # EX_CONFIG
ping -c1 db >/dev/null 2>&1 || exit 69   # EX_UNAVAILABLE
```

> **Merke**
>
> Exit-Codes sind ein Vertrag: dokumentieren, stabil halten, reservierte Bereiche meiden (126, 127, 128+n), Wertebereich 0 bis 255. Healthchecks: schnell, ohne Nebenwirkung, mit Zeitgrenze, aussagekräftig; Nagios-Konvention 0/1/2/3. UNKNOWN heißt „Check konnte nicht entscheiden“, nicht „kaputt“.

> **Praxisbeispiel**
>
> Ein Dienst hat ein Healthcheck-Skript, das Datenbankverbindung, freien Speicher und den Zeitstempel des letzten erfolgreichen Jobs prüft. Das Monitoring ruft es jede Minute auf, die Orchestrierung nutzt es als Readiness-Probe; wenn der Datenbank-Check wiederholt fehlschlägt, nimmt der Lastverteiler die Instanz automatisch aus dem Verkehr.

## 9.3 Metriken aus Bash

Logs beantworten „was ist passiert?“, *Metriken* beantworten „wie viel, wie schnell, wie oft?“. Eine Metrik ist ein Zahlenwert mit Name und gegebenenfalls Bezeichnern (Labels), der regelmäßig erfasst und über die Zeit aufgezeichnet wird. Für ein Skript sind vier Kennzahlen besonders aufschlussreich: die *Laufzeit*, der *Zeitpunkt des letzten Erfolgs* (um „läuft nicht mehr“ zu erkennen), die *Anzahl verarbeiteter Elemente* und die *Anzahl Fehler*; hinzu kommen *Größen* (Bytes, Dateien, Warteschlangenlängen). Die Laufzeit misst man in Bash mit der Variablen `SECONDS` (ganze Sekunden seit Shellstart; man kann sie auf 0 setzen) oder genauer mit `EPOCHREALTIME` (ab Bash 5.0: Sekunden mit Mikrosekunden seit 1970). Die Differenz zweier Zeitpunkte berechnet man mit `awk` oder `bc`, weil Bash selbst nur ganze Zahlen rechnet.

Ein verbreitetes Monitoring-System ist *Prometheus*. Es *holt* (scrape) Metriken regelmäßig von Zielen im einfachen Textformat: eine Zeile pro Wert, `name{label="wert"} zahl`, davor optional Zeilen `# HELP` und `# TYPE`. Kurzlebige Skripte können sich nicht scrapen lassen, weil sie zum Abfragezeitpunkt nicht mehr laufen. Dafür gibt es den *Textfile Collector* des `node_exporter`: Das Skript schreibt seine Metriken in eine Datei mit Endung `.prom` in ein Verzeichnis, das der Exporter überwacht, und der Exporter liefert den Inhalt bei jedem Scrape mit aus. Die Datei muss *atomar* geschrieben werden (zuerst in eine temporäre Datei im selben Verzeichnis, dann `mv`), damit der Exporter nie eine halbe Datei liest.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Ein Beitrag auf robustperception.io zum Textfile Collector (abgerufen, siehe Quellenverzeichnis) empfiehlt genau dieses Vorgehen: Die `.prom`-Datei wird in eine temporäre Datei im selben Verzeichnis geschrieben und dann per `rename` über die endgültige Datei gelegt; Leser sehen so die alte oder die neue Fassung, nie eine unvollständige. Eine direkte Umleitung mit `>` wird ausdrücklich abgelehnt, weil sie die Datei vor dem Schreiben kürzt. Die temporäre Datei muss auf demselben Dateisystem liegen, damit `rename` funktioniert.

```bash
#!/usr/bin/env bash
# metriken.sh - Laufzeit, Erfolg, Anzahl, Fehler und Groesse als Prometheus-Textdatei schreiben
set -uo pipefail
ziel_verz="${TEXTFILE_DIR:-$(mktemp -d)}"
job="importjob"
start="${EPOCHREALTIME/,/.}"                       # Komma-Locale absichern
verarbeitet=0; fehler=0; bytes=0

# --- eigentliche Arbeit (hier simuliert)
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
for i in 1 2 3 4 5; do head -c $((i * 1000)) /dev/zero > "$t/f$i.dat"; done
for f in "$t"/*.dat; do
    if [[ $f == *f4.dat ]]; then fehler=$((fehler + 1)); continue; fi   # ein simulierter Fehler
    verarbeitet=$((verarbeitet + 1))
    bytes=$((bytes + $(stat -c %s "$f")))
    sleep 0.05
done
ende="${EPOCHREALTIME/,/.}"
dauer="$(awk -v a="$start" -v b="$ende" 'BEGIN { printf "%.3f", b - a }')"

# --- Metriken atomar schreiben
schreibe_metriken() {
    local endgueltig="$ziel_verz/${job}.prom" tmp
    tmp="$(mktemp "$ziel_verz/.${job}.XXXXXX")"
    cat > "$tmp" <<ENDE
# HELP ${job}_duration_seconds Laufzeit des letzten Laufs.
# TYPE ${job}_duration_seconds gauge
${job}_duration_seconds $dauer
# HELP ${job}_processed_items Anzahl verarbeiteter Elemente im letzten Lauf.
# TYPE ${job}_processed_items gauge
${job}_processed_items $verarbeitet
# HELP ${job}_errors_total Anzahl Fehler im letzten Lauf.
# TYPE ${job}_errors_total gauge
${job}_errors_total $fehler
# HELP ${job}_processed_bytes Verarbeitete Bytes.
# TYPE ${job}_processed_bytes gauge
${job}_processed_bytes $bytes
# HELP ${job}_last_success_timestamp_seconds Unix-Zeit des letzten Erfolgs.
# TYPE ${job}_last_success_timestamp_seconds gauge
${job}_last_success_timestamp_seconds $( (( fehler == 0 )) && date +%s || echo 0 )
ENDE
    chmod 644 "$tmp"                      # mktemp legt 600 an, der Exporter muss lesen koennen
    mv -f -- "$tmp" "$endgueltig"         # atomar ersetzen
}
schreibe_metriken
echo "geschrieben nach $ziel_verz/${job}.prom:"; cat "$ziel_verz/${job}.prom" | grep -v '^#'
```

Das Skript misst die Zeit mit `EPOCHREALTIME` vor und nach der Arbeit; die Ersetzung `${EPOCHREALTIME/,/.}` fängt Locales ab, die ein Dezimalkomma liefern. Die Differenz rechnet `awk`, weil Bash keine Kommazahlen kennt. Die eigentliche Arbeit ist simuliert: Fünf Dateien, eine davon wird als „Fehler“ gezählt. Die Funktion `schreibe_metriken` erzeugt die Textdatei mit `HELP`- und `TYPE`-Zeilen, schreibt zunächst in eine temporäre Datei *im Zielverzeichnis* und tauscht sie per `mv` aus. Beachten Sie die Rechte: `mktemp` legt Dateien mit `600` an; der Exporter läuft meist unter einem anderen Benutzer und braucht Leserechte, daher das `chmod 644`. Als Typ wird *gauge* verwendet, also ein Wert, der steigen und fallen kann; Zähler (*counter*), die nur wachsen, würde ein Skript mit nur einem Lauf nicht korrekt führen, solange es keinen Zustand speichert. Weil im Beispiel ein Fehler auftritt, bleibt der Erfolgs-Zeitstempel bei 0. Der Zeitstempel des letzten Erfolgs ist die wichtigste Metrik für die Alarmierung „Job läuft nicht mehr“: Ein Alarm prüft einfach, ob er älter als erwartet ist.

Typische Fehler: Erstens Metriken in die Zieldatei umleiten (`>`), sodass der Exporter eine leere oder halbe Datei sieht. Zweitens falsche Dateiendung (nur `.prom` wird gelesen) oder zu strenge Rechte. Drittens sehr viele verschiedene Label-Werte (Benutzer-IDs, Dateinamen als Label): Das erzeugt eine *Kardinalitätsexplosion* und belastet das Monitoring; Labels haben wenige, feste Werte. Viertens fehlt ein „letzter Erfolg“-Zeitstempel, und ein Skript, das gar nicht mehr läuft, fällt nie auf, weil nur Erfolgszahlen aus alten Läufen stehen. Fünftens `SECONDS` wird für kurze Läufe verwendet und liefert dann 0. Sechstens rechnet man Dezimalzahlen mit Bash-Arithmetik. Siebtens liegt die Metrikdatei mit alten Werten ewig herum, obwohl der Job längst abgeschaltet ist (Aufräumen oder Zeitstempel prüfen).

```bash
# falsch: direkt umleiten, Dezimalrechnung in Bash, Datei fuer den Exporter unlesbar
echo "job_duration_seconds $((end - start))" > /var/lib/node_exporter/job.prom

# richtig: atomar, Rechte setzen, Dezimalrechnung mit awk
tmp="$(mktemp /var/lib/node_exporter/.job.XXXXXX)"
awk -v a="$start" -v b="$end" 'BEGIN{print "job_duration_seconds", b-a}' > "$tmp"
chmod 644 "$tmp" && mv -f -- "$tmp" /var/lib/node_exporter/job.prom
```

> **Merke**
>
> Vier Kennzahlen pro Job: Laufzeit, letzter Erfolg (Zeitstempel), Anzahl verarbeitet, Anzahl Fehler. Zeit mit `EPOCHREALTIME`, Dezimalrechnung mit `awk`. Prometheus Textfile Collector: `.prom`-Datei atomar schreiben, lesbare Rechte. Labels mit wenigen festen Werten.

> **Praxisbeispiel**
>
> Jeder Backup- und Import-Job schreibt am Ende vier Metriken. Ein Dashboard zeigt Laufzeiten im Verlauf, und eine Alarmregel meldet, wenn der Zeitstempel des letzten Erfolgs älter als 26 Stunden ist, also auch dann, wenn der Job nicht mehr startet und es deshalb keine Fehlermeldung gibt.

## 9.4 Alerting-Integration

Ein *Alarm* ist die Benachrichtigung eines Menschen oder Systems, dass jetzt etwas getan werden muss. Der entscheidende Punkt: Ein guter Alarm ist *handlungsrelevant*. Er kommt, wenn jemand reagieren muss, nicht für jede Kleinigkeit. Zu viele Alarme führen zu *Alarmmüdigkeit* (Alert Fatigue): Man ignoriert sie, auch die wichtigen. Daher gehört zu jeder Alarm-Integration die Überlegung, *wann* (Schwellwerte), *wie oft* (Entprellen, Wiederholung), *an wen* (Eskalation) und *womit* (Kanal) gemeldet wird.

Als Kanäle kommen infrage: *Mail* (einfach, aber langsam und leicht zu übersehen; über `mail`, `mailx` oder einen lokalen MTA wie `msmtp`), *Webhooks* (HTTP-POST mit JSON an Chat-Systeme wie Slack, Microsoft Teams oder Mattermost, oder an Incident-Dienste wie PagerDuty oder Opsgenie; mit `curl` leicht aufzurufen) und der *Weg über das Monitoring-System* (Prometheus Alertmanager, Nagios/Icinga, Zabbix): Das Skript liefert nur Exit-Code oder Metrik, das System entscheidet über Alarmierung. Der letztgenannte Weg ist meist der beste, weil Entprellung, Stille-Schaltung und Eskalation dort bereits gelöst sind; direkte Mail- oder Webhook-Aufrufe sind sinnvoll, wenn kein Monitoring vorhanden ist oder als Notkanal.

Zur *Entprellung* gehört: Nicht beim ersten Fehlschlag alarmieren, sondern erst nach N aufeinanderfolgenden Fehlschlägen (ein einzelner Netzfehler ist kein Notfall), und nicht *bei jedem* weiteren Fehlschlag erneut, sondern in Abständen (*Cooldown*). Dazu braucht das Skript einen *Zustand*, der Läufe überdauert: eine kleine Datei mit Zähler und Zeitpunkt der letzten Meldung. Außerdem sendet man eine *Entwarnung* (Recovery), wenn der Zustand wieder in Ordnung ist. *Schwellwerte* definiert man gestaffelt (WARN ab 80 Prozent, CRIT ab 95 Prozent) und *Eskalation* mehrstufig: zunächst der Kanal des Teams, nach 30 Minuten ohne Quittierung die Rufbereitschaft, danach die Leitung.

Für die Sicherheit gelten die bekannten Regeln: Die URL eines Webhooks ist ein Geheimnis (sie erlaubt jedem, Nachrichten zu senden) und gehört nicht ins Skript oder Repository, sondern in eine Datei mit Rechten `600` oder in den Secret Store (Modul 8.4). Nachrichteninhalte stammen oft aus Fehlermeldungen und müssen sauber in JSON gepackt werden (`jq -n -{}-arg`); sie sollten keine Geheimnisse enthalten. Alarm-Aufrufe brauchen eine Zeitgrenze und dürfen das eigentliche Skript nicht zum Absturz bringen: Fällt der Webhook aus, bleibt die Arbeit des Skripts erhalten, und das Scheitern wird lokal protokolliert.

```bash
#!/usr/bin/env bash
# alarm.sh - entprellter Webhook-Alarm mit Zustandsdatei, Cooldown und Entwarnung
set -uo pipefail
ZUSTAND="${ZUSTAND:-$(mktemp -d)/alarm-zustand}"      # Datei mit: FEHLER_IN_FOLGE LETZTE_MELDUNG_EPOCH
SCHWELLE_FOLGE="${SCHWELLE_FOLGE:-3}"                 # erst ab 3 Fehlern in Folge melden
COOLDOWN_SEK="${COOLDOWN_SEK:-3600}"
WEBHOOK_URL_DATEI="${WEBHOOK_URL_DATEI:-}"            # Datei mit der URL (Rechte 600), nie im Skript
mkdir -p -- "$(dirname "$ZUSTAND")"

sende_webhook() {                                      # sende_webhook TEXT
    local text="$1" url payload
    if [[ -z $WEBHOOK_URL_DATEI || ! -r $WEBHOOK_URL_DATEI ]]; then
        echo "[kein Webhook konfiguriert] $text" >&2; return 0
    fi
    url="$(< "$WEBHOOK_URL_DATEI")"
    payload="$(jq -n -c --arg text "$text" '{text:$text}')"
    curl -sS --fail --max-time 10 -H 'Content-Type: application/json' \
         -d "$payload" -- "$url" >/dev/null || echo "WARNUNG: Webhook fehlgeschlagen" >&2
}

lies_zustand() { local f=0 l=0; [[ -f $ZUSTAND ]] && read -r f l < "$ZUSTAND"; echo "${f:-0} ${l:-0}"; }
schreibe_zustand() { printf '%s %s\n' "$1" "$2" > "$ZUSTAND.neu" && mv -f -- "$ZUSTAND.neu" "$ZUSTAND"; }

melde_ergebnis() {                                     # melde_ergebnis ok|fehler BESCHREIBUNG
    local status="$1" text="$2" folge letzte jetzt; jetzt="$(date +%s)"
    read -r folge letzte <<< "$(lies_zustand)"
    if [[ $status == ok ]]; then
        (( folge >= SCHWELLE_FOLGE )) && sende_webhook "ENTWARNUNG: $text wieder in Ordnung"
        schreibe_zustand 0 0; return 0
    fi
    folge=$((folge + 1))
    if (( folge >= SCHWELLE_FOLGE )) && (( jetzt - letzte >= COOLDOWN_SEK )); then
        sende_webhook "ALARM: $text ($folge Fehler in Folge)"
        letzte=$jetzt
    fi
    schreibe_zustand "$folge" "$letzte"
}

# --- Vorfuehrung: drei Fehler, ein weiterer (Cooldown), dann Erfolg
for ergebnis in fehler fehler fehler fehler ok; do
    echo "Lauf: $ergebnis"
    melde_ergebnis "$ergebnis" "Dienst X"
done
```

Die Funktion `melde_ergebnis` implementiert die Entprellung: Ein Fehlschlag erhöht den Zähler, aber erst ab dem dritten in Folge wird alarmiert; ein weiterer Alarm folgt erst nach dem Cooldown, im Beispiel mit `COOLDOWN_SEK=3600` also nicht beim vierten Fehler. Wird der Dienst wieder gesund, nachdem schon ein Alarm erging, folgt eine Entwarnung und der Zähler wird zurückgesetzt. Der Zustand liegt in einer kleinen Datei, die atomar ersetzt wird. Die URL des Webhooks wird nur aus einer Datei gelesen, die Nachricht entsteht über `jq`, und der Aufruf hat Zeitgrenze und Fehlerbehandlung. Ist kein Webhook konfiguriert, schreibt das Skript die Meldung auf stderr, was sich gut zum Testen eignet. Wer den Webhook ausprobieren will, startet lokal einen kleinen Empfänger (`python3 -m http.server` akzeptiert allerdings nur GET; ein Einzeiler mit `nc -l` reicht zum Anschauen der Anfrage).

Typische Fehler: Erstens Alarm bei jedem einzelnen Fehlschlag (Alarmmüdigkeit). Zweitens kein Zustand, daher bei jedem Lauf die gleiche Meldung (Alarm-Flut) oder nie eine Entwarnung. Drittens Webhook-URL im Skript oder in Git. Viertens Nachrichten mit zusammengesetztem JSON (ungültig bei Anführungszeichen im Fehlertext). Fünftens der Alarmweg ist nicht überwacht: Fällt die Mailzustellung aus, merkt es niemand; ein regelmäßiger Testalarm oder ein „Heartbeat“-Dienst, der *Ausbleiben* meldet, schützt. Sechstens Alarme ohne Handlungsanweisung: Eine Meldung „Fehler im Job“ hilft nachts niemandem; sie sollte Skript, Host, Run-ID, Kurzursache und einen Link zum Runbook (Modul 9.8) enthalten. Siebtens fehlt die Eskalation, und ein Alarm verhallt, weil die zuständige Person im Urlaub ist.

```bash
# falsch: bei jedem Fehler melden, URL im Skript, zusammengesetztes JSON
curl -d "{\"text\":\"Fehler: $fehlertext\"}" https://hooks.example/T000/B000/geheimerToken

# richtig: Schwelle, Zustand, URL aus Datei, JSON ueber jq, mit Runbook-Link
payload="$(jq -n -c --arg t "ALARM $host/$skript run=$RUN_ID: $kurz  Runbook: $link" '{text:$t}')"
curl -sS --fail --max-time 10 -H 'Content-Type: application/json' -d "$payload" -- "$(< "$URL_DATEI")"
```

> **Merke**
>
> Alarme müssen handlungsrelevant sein. Entprellen (N Fehler in Folge), Cooldown, Entwarnung, Eskalation. Zustand in kleiner Datei, atomar geschrieben. Webhook-URL = Geheimnis; JSON nur über `jq`; Zeitgrenzen; Alarmweg selbst überwachen. Meldung enthält Ort, Run-ID, Ursache und Runbook-Link.

> **Praxisbeispiel**
>
> Ein Backup-Skript meldet nach drei aufeinanderfolgenden Fehlschlägen an den Team-Kanal, nach weiteren zwei Stunden ohne Besserung an die Rufbereitschaft. Die Meldung enthält Host, Run-ID, die letzte Fehlermeldung und den Link zur Anleitung im Wiki; wird es wieder grün, kommt automatisch die Entwarnung.

## 9.5 journalctl und syslog

Auf klassischen Unix-Systemen schreiben Programme ihre Meldungen an den *syslog*-Dienst, der sie nach *Facility* (Herkunft: `auth`, `daemon`, `user`, `local0` bis `local7`) und *Priorität* (von `debug` über `info`, `notice`, `warning`, `err`, `crit`, `alert` bis `emerg`) in Dateien wie `/var/log/syslog` oder `/var/log/messages` sortiert, oder an einen zentralen Log-Server weiterleitet. Moderne Systeme setzen *systemd-journald* davor: Es sammelt die Ausgaben aller Dienste (auch stdout und stderr), hält sie in einem strukturierten Binärformat mit Metadaten (Unit, PID, Zeit, Priorität, Host, Benutzer) und reicht sie auf Wunsch an syslog weiter. Man liest es mit `journalctl`.

Für Skripte ist `logger` das Bindeglied: `logger -t mein-skript -p user.notice -i -- "Meldung"` schreibt an syslog beziehungsweise das Journal, mit einem *Tag* (`-t`, damit man danach filtern kann), einer Priorität (`-p facility.level`) und der PID (`-i`). Laufen Skripte als systemd-Dienst, landet alles, was sie nach stdout oder stderr schreiben, automatisch im Journal, und man braucht `logger` gar nicht. Die wichtigsten Filter von `journalctl`: `-u dienst` (Unit), `-t tag` (Syslog-Kennung), `-p warning` (ab dieser Priorität und schlimmer; `-p warning..err` für einen Bereich), `-{}-since "2 hours ago"` und `-{}-until`, `-b` (seit dem letzten Boot, `-b -1` für den vorherigen), `-n 50` (letzte Zeilen), `-f` (mitlesen), `-g muster` (Suche per Regex), `-k` (Kernel-Meldungen) sowie feldbasierte Filter wie `_PID=1234` oder `_SYSTEMD_UNIT=app.service`. Für Skripte unverzichtbar sind `-{}-no-pager` und eine passende Ausgabeform: `-o cat` (nur die Meldung), `-o short-iso` (Zeitstempel mit Zeitzone) und `-o json` (ein JSON-Objekt pro Zeile, mit allen Feldern, ideal zusammen mit `jq`).

Die eigentliche Stärke liegt in der *Korrelation*: Man legt Ereignisse mehrerer Quellen nebeneinander. War der Dienst um 02:14 Uhr neu gestartet? Gab es kurz davor Kernel-Meldungen über Speichermangel (OOM-Killer)? Welche Run-ID hatte das Skript, das gleichzeitig Fehler meldete (Modul 9.1)? Dazu fragt man das Journal mit einem Zeitfenster und mehreren Filtern ab. Beachten Sie die Rechte: Normale Benutzer sehen nur die eigenen Meldungen, für das ganze Journal braucht man die Gruppe `systemd-journal` oder `adm`. Und: Das Journal ist nicht zwingend dauerhaft (je nach Konfiguration `Storage=volatile` nur im Speicher), seine Größe ist begrenzt (`journalctl -{}-vacuum-size=...`).

```bash
#!/usr/bin/env bash
# journal-analyse.sh - Journal-Auszuege (JSON, ein Objekt pro Zeile) mit jq auswerten
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT

# Beispieldaten im Format von "journalctl -o json" (gekuerzt). In der Praxis:
#   journalctl --since "02:00" --until "03:00" -o json --no-pager > journal.json
cat > "$t/journal.json" <<'ENDE'
{"__REALTIME_TIMESTAMP":"1793412840000000","_SYSTEMD_UNIT":"app.service","PRIORITY":"6","MESSAGE":"Dienst gestartet"}
{"__REALTIME_TIMESTAMP":"1793412900000000","_SYSTEMD_UNIT":"app.service","PRIORITY":"3","MESSAGE":"Datenbank nicht erreichbar"}
{"__REALTIME_TIMESTAMP":"1793412901000000","SYSLOG_IDENTIFIER":"import-job","PRIORITY":"4","MESSAGE":"ts=2026-10-30T02:15:01Z level=WARN run_id=ab12cd34 msg=\"Retry 1\""}
{"__REALTIME_TIMESTAMP":"1793412905000000","SYSLOG_IDENTIFIER":"import-job","PRIORITY":"3","MESSAGE":"ts=2026-10-30T02:15:05Z level=ERROR run_id=ab12cd34 msg=\"Abbruch\""}
{"__REALTIME_TIMESTAMP":"1793412906000000","_SYSTEMD_UNIT":"app.service","PRIORITY":"3","MESSAGE":"Datenbank nicht erreichbar"}
{"__REALTIME_TIMESTAMP":"1793413000000000","_TRANSPORT":"kernel","PRIORITY":"2","MESSAGE":"Out of memory: Killed process 4242 (java)"}
ENDE

echo "--- Anzahl je Quelle und Prioritaet"
jq -r '[(._SYSTEMD_UNIT // .SYSLOG_IDENTIFIER // ._TRANSPORT), .PRIORITY] | @tsv' "$t/journal.json" |
    sort | uniq -c | sort -rn

echo "--- Fehler (Prioritaet 0-3) mit Uhrzeit, zeitlich geordnet"
jq -r 'select((.PRIORITY|tonumber) <= 3)
       | [((.__REALTIME_TIMESTAMP|tonumber/1000000)|strftime("%H:%M:%S")),
          (._SYSTEMD_UNIT // .SYSLOG_IDENTIFIER // ._TRANSPORT), .MESSAGE] | @tsv' "$t/journal.json" | sort

echo "--- Alle Zeilen eines Skriptlaufs (Run-ID) aus dem Journal"
jq -r 'select(.MESSAGE | test("run_id=ab12cd34")) | .MESSAGE' "$t/journal.json"

echo "--- Korrelation: was geschah im Fenster +-60 s um den Skriptfehler?"
fehlerzeit="$(jq -r 'select(.MESSAGE|test("level=ERROR")) | (.__REALTIME_TIMESTAMP|tonumber/1000000|floor)' "$t/journal.json" | head -n 1)"
jq -r --argjson z "$fehlerzeit" 'select(((.__REALTIME_TIMESTAMP|tonumber/1000000) - $z | fabs) <= 60)
       | [((.__REALTIME_TIMESTAMP|tonumber/1000000)|strftime("%H:%M:%S")), .MESSAGE] | @tsv' "$t/journal.json"
```

Die Beispieldaten haben das Format von `journalctl -o json` (Zeitstempel in Mikrosekunden als Zeichenkette, Priorität als Zeichenkette, Quelle je nach Eintrag als `_SYSTEMD_UNIT`, `SYSLOG_IDENTIFIER` oder `_TRANSPORT`). Das Beispiel nutzt den `jq`-Operator `//` (Alternative, wenn das Feld fehlt), um aus den drei möglichen Feldern eine einheitliche Quelle zu bilden. Der erste Block zählt Einträge je Quelle und Priorität, der zweite filtert Fehler (Priorität 3 und niedriger, also schlimmer) und wandelt die Zeit lesbar um. Der dritte Block zeigt die Verbindung zur Logging-Konvention aus Modul 9.1: Weil das Skript seine Run-ID in jede Zeile schreibt, lassen sich alle Zeilen des Laufs im Journal finden. Der vierte Block korreliert: Er nimmt den Zeitpunkt des ersten Skriptfehlers und zeigt alle Journal-Einträge im Umkreis von 60 Sekunden, hier also auch die gleichzeitigen Datenbankfehler des Dienstes, ein Hinweis auf die gemeinsame Ursache. Die Zeitumrechnung mit `strftime` erfolgt in UTC; man beachte die Zeitzone beim Vergleich mit Uhrzeiten aus Berichten.

Typische Fehler: Erstens `journalctl` ohne `-{}-no-pager` und ohne Zeitgrenze: Es hängt im Pager oder liefert Gigabytes. Zweitens die Auswertung mit `grep` auf der formatierten Ausgabe statt mit `-o json` und `jq`, sodass jede Änderung des Textformats den Parser bricht. Drittens wird die Priorität falsch verstanden (`-p err` zeigt auch `crit`, `alert`, `emerg`, aber nichts Leichteres). Viertens fehlende Rechte: Das Skript sieht nur einen Teil und der Befund ist unvollständig. Fünftens wurden Journal-Daten mit der Rotation schon gelöscht oder liegen nur flüchtig im Speicher. Sechstens unterscheiden sich Zeitzonen von Journal, Skript und Bericht. Siebtens enthält das Journal personenbezogene Daten und Geheimnisse, die beim Weitergeben von Auszügen (Tickets, Chats) geschwärzt werden müssen (Modul 8.9).

```bash
# falsch: Pager, keine Zeitgrenze, Parsen von Menschentext
journalctl -u app | grep "Datenbank" | awk '{print $3}'

# richtig: begrenzt, ohne Pager, strukturiert ausgewertet
journalctl -u app --since "-1h" -p warning --no-pager -o json | jq -r '.MESSAGE'
logger -t import-job -p user.err -i -- "run_id=ab12cd34 Abbruch"
```

> **Merke**
>
> Skripte schreiben mit `logger -t` (oder einfach nach stderr als Dienst) ins Journal. Auswerten mit Zeitfenster, `-{}-no-pager` und `-o json | jq`. Filter: `-u`, `-t`, `-p`, `-{}-since`, `-b`. Ereignisse verschiedener Quellen über Zeit und Run-ID korrelieren. Rechte und Aufbewahrung beachten.

> **Praxisbeispiel**
>
> Ein Dienst ist nachts einmal ausgefallen. Der Bereitschaftsdienst ruft das Journal der Stunde ab, sieht eine OOM-Meldung des Kernels zwei Sekunden vor dem Neustart und findet in den Skript-Logs mit der Run-ID, dass ein Importjob zur gleichen Zeit den Speicher belegte. Ursache und Gegenmaßnahme (Limit für den Job) sind in zehn Minuten klar.

## 9.6 Troubleshooting Methodik

Fehlersuche ist eine Methode, keine Intuition. Der häufigste Grund für lange Fehlersuchen ist, dass man *ohne Hypothese* Dinge verändert und hofft. Ein geordnetes Vorgehen hat fünf Schritte. Erstens *Symptom genau beschreiben*: Was genau passiert, seit wann, wie oft, wo? Die Fehlermeldung wörtlich notieren, nicht umschreiben. Zweitens *Reproduzieren*: Kann man den Fehler auslösen? Ein Fehler, der sich auf Knopfdruck zeigt, ist halb gelöst; sporadische Fehler brauchen zusätzlich Logging und Geduld. Drittens *Eingrenzen* (isolieren): den Fehler auf das kleinste Beispiel reduzieren, Teile entfernen, bis er verschwindet (*minimal reproducible example*), oder mit *Halbierung* (Bisect) arbeiten: Läuft der erste Teil des Skripts noch korrekt, liegt der Fehler im zweiten. Viertens *Hypothesen bilden und einzeln testen*: Eine Veränderung nach der anderen, jeweils mit klarer Erwartung („Wenn es an den Rechten liegt, muss es als root funktionieren“). Fünftens *Ursache beheben und absichern*: Dokumentieren, Test oder Check ergänzen, damit der Fehler nicht zurückkehrt.

Ein Dauerbrenner sind *Umgebungsunterschiede*: „Auf meinem Rechner läuft es, in Cron/CI nicht.“ Interaktive Shells haben einen langen `PATH`, geladene Profile, ein Terminal, gesetzte Locale und das aktuelle Arbeitsverzeichnis. Cron, systemd und CI starten mit einer minimalen Umgebung: kurzer `PATH` (oft nur `/usr/bin:/bin`), kein Terminal (`tty` fehlt), andere Locale, anderes Home, anderer Benutzer, kein SSH-Agent. Man macht die Unterschiede *sichtbar*, indem man die Umgebung beider Welten festhält und vergleicht: `env | sort > interaktiv.txt` und im anderen Kontext dasselbe, dann `diff`. Eine schnelle Simulation liefert `env -i HOME="$HOME" PATH=/usr/bin:/bin bash -c 'skript'`: Es startet das Skript mit leerer Umgebung, wie Cron es etwa täte.

Für die Beobachtung des Skripts selbst gibt es die Werkzeuge aus Modul 6.4: `bash -x` (Ablaufverfolgung) mit aussagekräftigem `PS4` (`PS4='+ ${BASH_SOURCE##*/}:${LINENO}: '`), `bash -n` (Syntaxprüfung), ShellCheck, gezielte `echo`-Ausgaben nach stderr und `trap ... ERR`. Reicht das nicht, zeigt `strace -f -e trace=file,process` (wo verfügbar), welche Dateien und Programme tatsächlich angefasst werden. Fehler in *Zeit* und *Menge* (nur manchmal, nur bei großen Daten) erfordern zusätzlich Messungen (Modul 9.3) und Zeitstempel in den Logs.

```bash
#!/usr/bin/env bash
# umgebung-vergleich.sh - macht Umgebungsunterschiede zwischen "interaktiv" und "Cron-aehnlich" sichtbar
set -uo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT

# Das "problematische" Skript: braucht ein Werkzeug, das nur im interaktiven PATH liegt
cat > "$t/job.sh" <<'ENDE'
#!/usr/bin/env bash
echo "PATH hat $(tr ':' '\n' <<< "$PATH" | wc -l) Eintraege"
echo "HOME=${HOME:-<nicht gesetzt>}  USER=${USER:-<nicht gesetzt>}  LANG=${LANG:-<nicht gesetzt>}"
echo "Terminal: $( [ -t 0 ] && echo ja || echo nein )"
if command -v meinwerkzeug >/dev/null; then echo "meinwerkzeug gefunden: $(command -v meinwerkzeug)"; else echo "FEHLER: meinwerkzeug nicht gefunden"; exit 127; fi
ENDE
chmod +x "$t/job.sh"
mkdir "$t/tools"; printf '#!/bin/sh\necho werkzeug\n' > "$t/tools/meinwerkzeug"; chmod +x "$t/tools/meinwerkzeug"
export PATH="$t/tools:$PATH"        # nur in dieser "interaktiven" Umgebung vorhanden (z.B. aus .bashrc)

umgebung() { env | sort; }

echo "=== 1) Normale Umgebung"
"$t/job.sh"; echo "(Exit-Code $?)"

echo; echo "=== 2) Cron-aehnliche Umgebung (env -i, kurzer PATH)"
env -i HOME=/root SHELL=/bin/sh PATH=/usr/bin:/bin "$t/job.sh" </dev/null; echo "(Exit-Code $?)  <- 127 = nicht gefunden"

echo; echo "=== 3) Unterschiede der Variablen (nur Namen)"
umgebung | cut -d= -f1 > "$t/a.txt"
env -i HOME=/root SHELL=/bin/sh PATH=/usr/bin:/bin bash -c 'env | sort' | cut -d= -f1 > "$t/b.txt"
echo "nur in der normalen Umgebung: $(comm -23 "$t/a.txt" "$t/b.txt" | wc -l) Variablen, z.B.:"
comm -23 "$t/a.txt" "$t/b.txt" | head -n 6 | sed 's/^/   /'

echo; echo "=== 4) Ablaufverfolgung mit sprechendem PS4 (nur die ersten Zeilen)"
bash -c 'PS4="+ \${BASH_SOURCE##*/}:\${LINENO}: "; set -x; source "$1"' _ "$t/job.sh" 2>&1 | grep '^+' | head -n 4
```

Das Beispielskript verhält sich in zwei Umgebungen unterschiedlich: In der normalen Umgebung findet es das Werkzeug `meinwerkzeug`, das über den erweiterten `PATH` (wie aus einer `.bashrc`) erreichbar ist; in der Cron-ähnlichen Umgebung mit `env -i` und verkürztem `PATH` endet es mit dem klassischen Exit-Code 127. Der dritte Block zeigt die Methode: Namen der Umgebungsvariablen beider Welten in Dateien schreiben und mit `comm -23` die nur in der einen vorkommenden finden. Das ist oft der Aha-Moment: Es fehlen `LANG`, `TERM`, `SSH_AUTH_SOCK`, `XDG_*` und andere. Der vierte Block zeigt die Ablaufverfolgung mit einem `PS4`, das Datei und Zeilennummer ausgibt (gesetzt innerhalb der Shell, direkt vor `set -x`, denn nicht jede Umgebung übernimmt `PS4` zuverlässig aus den Umgebungsvariablen); so wird aus dem Wust von `+`-Zeilen eine lesbare Spur. Wichtig ist die Disziplin dahinter: Man verändert jeweils *eine* Sache (hier die Umgebung), beobachtet und notiert.

Typische Fehler: Erstens wird sofort „repariert“, ohne den Fehler zu verstehen; das Symptom verschwindet, die Ursache bleibt. Zweitens werden mehrere Dinge gleichzeitig geändert, sodass man nicht weiß, was geholfen hat. Drittens wird die Fehlermeldung nur sinngemäß weitergegeben („geht nicht“) statt wörtlich mit Kontext (Skriptversion, Umgebung, Zeitpunkt, Befehlszeile). Viertens wird ein Fehler nur im interaktiven Kontext getestet, obwohl er in Cron oder CI auftritt. Fünftens werden Logs erst eingeschaltet, nachdem der Fehler wieder verschwunden ist; besser ist es, Diagnoseoptionen vorbereitet zu haben (`DEBUG=1`). Sechstens vertraut man dem „Es hat schon immer funktioniert“ und prüft nicht, was sich geändert hat: Aktualisierungen, Konfiguration, Daten, Zeit, Plattenstand? Siebtens wird die gefundene Ursache nicht abgesichert, und derselbe Fehler kommt in drei Monaten zurück.

```bash
# falsch: raten, mehrere Aenderungen auf einmal
chmod -R 777 /srv; export PATH=$PATH:/opt/bin; systemctl restart app; "klappt jetzt?"

# richtig: Hypothese -> eine Aenderung -> Beobachtung
# Hypothese: "Im Cron-Kontext fehlt jq im PATH"
env -i HOME="$HOME" PATH=/usr/bin:/bin bash -c 'command -v jq' || echo "bestaetigt: jq nicht im PATH"
# Behebung: absoluten Pfad oder PATH im Skript setzen, dann denselben Test wiederholen
```

> **Merke**
>
> Symptom genau beschreiben, reproduzieren, eingrenzen (minimal, Halbierung), Hypothese einzeln testen, Ursache beheben und absichern. Umgebungsunterschiede sichtbar machen: `env | sort` vergleichen, `env -i` simuliert Cron. Ablaufverfolgung mit `bash -x` und aussagekräftigem `PS4`. Immer nur eine Änderung gleichzeitig.

> **Praxisbeispiel**
>
> Ein Skript lief seit Monaten unauffällig und scheitert plötzlich nachts. Der Verlauf: Fehlermeldung notieren, Änderungen der letzten Tage durchsehen (eine Paketaktualisierung hatte die Version eines Werkzeugs geändert), im Testlauf mit `env -i` nachstellen, Bedingung (Version) per Check am Skriptanfang absichern und im Runbook vermerken.

## 9.7 Typische Fehlerbilder

Manche Fehlermeldungen begegnen Ihnen immer wieder. Wer die Muster kennt, erkennt die Ursache oft schon an der Meldung. *Permission denied* bedeutet, dass das Betriebssystem den Zugriff verweigert, doch es gibt mehrere Ursachen: Dem Skript fehlt das Ausführrecht (`chmod +x`; alternativ als `bash skript.sh` starten), dem Benutzer fehlt das Recht an Datei oder einem Verzeichnis auf dem Weg dorthin (jedes Verzeichnis braucht das Durchsuchen-Recht `x`), die Partition ist mit `noexec` eingehängt (häufig bei `/tmp` oder USB-Medien), oder eine Sicherheitsrichtlinie (SELinux, AppArmor) blockiert. Hilfreich: `ls -ld` entlang des Pfades, `namei -l pfad` (zeigt die Rechte jeder Pfadkomponente), `mount | grep noexec`, `id`.

*command not found* (Exit-Code 127) heißt: Die Shell findet das Programm nicht. Ursachen: Tippfehler, das Programm ist nicht installiert, es liegt nicht im `PATH` (häufig in Cron und bei `sudo`, siehe Modul 9.6), es fehlt der Pfad bei lokalen Skripten (`./skript.sh` statt `skript.sh`) oder die Variable ist leer (`$BEFEHL` nicht gesetzt). Eine Variante mit Verwirrungspotenzial: `bad interpreter: No such file or directory` bei einem existierenden Skript. Hier fehlt nicht das Skript, sondern der in der Shebang-Zeile genannte Interpreter, sehr oft wegen *Windows-Zeilenenden*: Steht hinter `#!/bin/bash` ein Wagenrücklauf (CR, `\r`), sucht das System einen Interpreter namens `bash\r`. Erkennen: `file skript.sh` meldet „with CRLF line terminators“, `cat -A skript.sh` zeigt `\^{}M` am Zeilenende. Beheben: `sed -i 's/\r$//' skript.sh` oder `dos2unix`; dauerhaft stellt man Git und Editor auf LF ein (`.gitattributes`: `*.sh text eol=lf`).

*bad substitution* entsteht, wenn Bash-spezifische Syntax von einer anderen Shell gelesen wird, typischerweise `dash`, auf Debian und Ubuntu die Standard-`sh`. Ein Skript mit `${name//a/b}`, Arrays oder `[[ ]]`, das mit `sh skript.sh` (oder mit der Shebang `#!/bin/sh`) gestartet wird, scheitert dort. Abhilfe: Shebang `#!/usr/bin/env bash` und mit `bash` starten oder POSIX-konform schreiben (Modul 1.7). Als Fehlerbild kommt dazu: *syntax error: unexpected end of file* (fehlendes `fi`, `done`, `}` oder eine nie beendete Anführungszeichen-Folge, oft durch CRLF), *unbound variable* (`set -u` und eine nicht gesetzte Variable) und *integer expression expected* (Vergleich mit `-eq` auf eine nicht numerische oder leere Variable).

Schließlich die *Locale*: Sie legt Sortierreihenfolge, Zeichenklassen, Dezimaltrenner und Kodierung fest. Das Ergebnis von `sort` hängt von `LC_COLLATE` ab (in vielen Locales werden Groß- und Kleinbuchstaben gemischt sortiert, in `C` stehen alle Großbuchstaben vor den Kleinbuchstaben); Zahlenausgabe von `printf` und `awk` verwendet in deutschen Locales ein Komma statt eines Punktes; `[a-z]` in Regeln deckt in manchen Locales auch Großbuchstaben ab; Zeichen außerhalb von ASCII verhalten sich je nach Kodierung anders (`wc -c` zählt Bytes, `wc -m` Zeichen). Für Skripte, die reproduzierbar sein sollen, setzt man vor dem betreffenden Befehl `LC_ALL=C` (maschinenlesbar, schnell) oder bewusst `C.UTF-8`.

```bash
#!/usr/bin/env bash
# fehlerbilder.sh - typische Fehler gezielt ausloesen und automatisch diagnostizieren
set -uo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
cd "$t"

zeige() { printf '\n### %s\n' "$*"; }

zeige "1) Permission denied: Skript ohne Ausfuehrungsrecht"
printf '#!/usr/bin/env bash\necho laeuft\n' > ohne-x.sh
./ohne-x.sh 2>&1 | head -n 2
echo "Diagnose: $(stat -c 'Rechte %a' ohne-x.sh) -> chmod +x oder 'bash ohne-x.sh'"; bash ohne-x.sh

zeige "2) Windows-Zeilenenden (CRLF)"
printf '#!/usr/bin/env bash\r\necho laeuft\r\n' > crlf.sh; chmod +x crlf.sh
./crlf.sh 2>&1 | head -n 2
diagnose_crlf() { grep -q $'\r' "$1" && echo "Diagnose: $1 enthaelt CR-Zeichen (CRLF)"; }
diagnose_crlf crlf.sh
sed -i 's/\r$//' crlf.sh && ./crlf.sh

zeige "3) command not found"
bash -c 'jq-gibt-es-nicht --version' 2>&1 | head -n 1
echo "Exit-Code: $(bash -c 'jq-gibt-es-nicht' >/dev/null 2>&1; echo $?) (127 = nicht gefunden)"
echo "PATH-Eintraege: $(tr ':' '\n' <<< "$PATH" | wc -l)"

zeige "4) bad substitution: Bash-Syntax unter dash/sh"
printf 'name="a-b-c"\necho "${name//-/_}"\n' > subst.sh
sh subst.sh 2>&1 | head -n 1
echo "unter bash:"; bash subst.sh
echo "Diagnose: /bin/sh -> $(readlink -f /bin/sh)"

zeige "5) integer expression expected (leere Variable im Vergleich)"
bash -c 'x=""; [ $x -gt 5 ] && echo gross' 2>&1 | head -n 1
bash -c 'x=""; [[ ${x:-0} -gt 5 ]] && echo gross || echo "nicht gross (mit Default 0)"'

zeige "6) Locale: Sortierreihenfolge"
printf 'b\nA\na\nB\n' | LC_ALL=C sort | paste -sd' '
if locale -a 2>/dev/null | grep -qi '^en_US.utf-\?8$'; then
    printf 'b\nA\na\nB\n' | LC_ALL=en_US.UTF-8 sort | paste -sd' '
else
    echo "(Locale en_US.UTF-8 ist hier nicht installiert; dort waere die Reihenfolge a A b B)"
fi
```

Das Skript löst sechs Fehlerbilder gezielt aus und zeigt die Diagnose. Der Aufruf einer Datei ohne Ausführungsrecht liefert `Permission denied`; die Abhilfe, die Datei direkt mit `bash` zu starten, funktioniert, weil dann nur Leserecht nötig ist. Bei der CRLF-Datei scheitert der Aufruf an der Shebang-Zeile, die Meldung nennt je nach System einen merkwürdigen Interpreternamen mit Sonderzeichen oder `No such file or directory`; `grep -q $'\r'` erkennt das Problem zuverlässig, `sed -i 's/\r$//'` behebt es. Der `command not found`-Test zeigt den Exit-Code 127. Beim `bad substitution` wird dasselbe Skript mit `sh` (dash) und mit `bash` gestartet: nur Letzteres kennt `${name//-/_}`. Der Fall mit der leeren Variable erklärt die Warnung `integer expression expected`: `[ $x -gt 5 ]` wird bei leerem `x` zu `[ -gt 5 ]`; mit Anführungszeichen, `[[ ]]` und Standardwert vermeidet man das. Zum Schluss zeigt der Sortiertest, dass die Reihenfolge von der Locale abhängt.

Typische Fehler beim Umgang mit diesen Fehlerbildern: Erstens wird bei `Permission denied` mit `chmod 777` reagiert, statt die Ursache (Eigentümer, `noexec`, Pfadrechte) zu klären. Zweitens wird CRLF nicht erkannt, weil der Editor die Zeichen nicht anzeigt; `cat -A` oder `file` zeigen es. Drittens werden Skripte mit `sh skript.sh` gestartet, obwohl sie Bash-Syntax verwenden, und der Fehler wird dann für einen Fehler im Skript gehalten statt für den falschen Interpreter. Viertens verlässt man sich auf die Locale der Entwicklungsmaschine, und die Ausgabe in der CI sieht anders aus (andere Sortierung, anderer Dezimaltrenner). Fünftens übersetzt man „command not found“ nicht in die konkrete Frage „welchen PATH hat dieser Prozess?“. Sechstens ignoriert man den Exit-Code 126 (gefunden, aber nicht ausführbar) und 127 (nicht gefunden) und behandelt beide wie „Fehler“. Siebtens bleiben Fehlermeldungen auf stderr unbemerkt, weil sie in einer Pipeline, in `/dev/null` oder in einem Cron-Mail verschwinden.

```bash
# falsch: Rechte aufreissen, mit sh starten, Locale ignorieren
chmod 777 skript.sh; sh skript.sh
sort daten.txt | uniq -c              # Ergebnis haengt von der Locale ab

# richtig: Ursache pruefen, richtigen Interpreter, Locale festlegen
namei -l /pfad/zum/skript.sh; mount | grep -w noexec
file skript.sh; head -n 1 skript.sh | cat -A
bash skript.sh
LC_ALL=C sort daten.txt | uniq -c
```

> **Merke**
>
> `Permission denied`: Ausführrecht, Pfadrechte, `noexec`, SELinux. `127`: nicht gefunden (PATH!), `126`: nicht ausführbar. `bad interpreter`: oft CRLF (`cat -A`, `sed -i 's/\r$//'`, `.gitattributes`). `bad substitution`: Bash-Syntax unter `sh`/dash. Locale beeinflusst Sortierung und Zahlenformat: `LC_ALL=C`.

> **Praxisbeispiel**
>
> Ein Skript wurde auf einem Windows-Rechner bearbeitet und per Git verteilt. Auf den Servern scheiterte es mit „bad interpreter“. Mit einer `.gitattributes`-Regel (`*.sh text eol=lf`) und einem Check in der CI, der CR-Zeichen in Skripten meldet, kam das Problem nicht wieder.

## 9.8 Runbooks

Ein *Runbook* ist eine Anleitung für wiederkehrende Betriebsaufgaben und Störungen: Wenn Symptom X auftritt, prüfe A, dann B, und tue Folgendes. Es ersetzt das Wissen im Kopf einzelner Personen durch ein Dokument, das auch nachts um drei Uhr mit halbem Wachzustand befolgt werden kann. Ein gutes Runbook ist *kurz, konkret und überprüfbar*: Es beginnt mit dem Symptom oder dem Alarmnamen, beschreibt Auswirkung und Dringlichkeit, listet die Prüfschritte mit *genauen Befehlen* und erwarteten Ergebnissen, nennt die Behebung Schritt für Schritt, bestimmt, wann und an wen *eskaliert* wird, und erklärt, wie man *prüft, dass alles wieder gut ist*.

Ein Standardaufbau, der sich bewährt: (1) *Titel und Zweck*, (2) *Auslöser* (Alarm, Symptom), (3) *Auswirkung und Priorität*, (4) *Voraussetzungen* (Zugänge, Rechte), (5) *Diagnose* (Befehle, was ihre Ausgabe bedeutet), (6) *Behebung* (nummerierte Schritte, riskante Schritte markiert), (7) *Wiederanlauf und Verifikation*, (8) *Eskalation* (Kontakte, Fristen), (9) *Nacharbeit* (Ursachenanalyse, Dokumentation), (10) *Änderungshistorie* und *Verantwortlicher*. Wichtig ist, dass das Runbook *lebt*: Es liegt im selben Repository wie das Skript, wird bei Änderungen am Skript mitgeändert und nach jedem Einsatz verbessert (Was fehlte? Was war falsch?). Veraltete Runbooks sind gefährlicher als keine.

Einen Schritt weiter geht das *ausführbare Runbook*: Die Diagnoseschritte sind selbst ein Skript, das Prüfungen automatisch ausführt, die Ergebnisse gebündelt zeigt und beim ersten fehlgeschlagenen Schritt den passenden Hinweis ausgibt. Das spart Tipparbeit, verhindert Fehler bei der Abschrift von Befehlen und liefert eine *dokumentierte Spur* (die Ausgabe lässt sich ins Ticket kopieren). Riskante Behebungsschritte (Neustart, Löschen) bleiben dagegen bewusst *manuell* oder verlangen eine Bestätigung; das Skript diagnostiziert, der Mensch entscheidet. Zum Wiederanlauf gehört auch, dass ein Lauf *idempotent* wiederholbar ist (Modul 6.7).

```bash
#!/usr/bin/env bash
# runbook-import.sh - ausfuehrbare Diagnose: Schritte, erwartetes Ergebnis, Hinweis, Eskalation
set -uo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT

RUNBOOK_TITEL="Importjob liefert keine neuen Daten"
ESKALATION="Bereitschaft Datenplattform (Chat #data-oncall), nach 30 min ohne Loesung"

# --- Testumgebung: Konfig vorhanden, Eingabeverzeichnis leer, Sperre veraltet
mkdir -p "$t/eingang" "$t/lock"; echo "quelle=ftp" > "$t/import.conf"
echo "4242" > "$t/lock/import.pid"           # PID gibt es nicht (veraltete Sperre)

# --- Schritte: TITEL @@ PRUEFBEFEHL (Erfolg = ok) @@ HINWEIS bei Misserfolg
SCHRITTE=(
 "Konfiguration vorhanden@@test -s $t/import.conf@@Konfig fehlt oder ist leer: aus dem Repository neu ausrollen (siehe Modul 7.7)"
 "Eingabeverzeichnis lesbar@@test -r $t/eingang@@Rechte/Mount pruefen: namei -l PFAD; mount | grep eingang"
 "Neue Eingabedateien vorhanden@@test -n \"\$(find $t/eingang -type f -mmin -60 | head -n 1)\"@@Keine Datei in den letzten 60 min: Lieferant kontaktieren, Quelle pruefen"
 "Keine veraltete Sperre@@! test -e $t/lock/import.pid || kill -0 \$(cat $t/lock/import.pid) 2>/dev/null@@Sperrdatei verweist auf toten Prozess: Sperrdatei entfernen (nach Bestaetigung), Job neu starten"
)

echo "Runbook: $RUNBOOK_TITEL"
nr=0; erstes_problem=""
for eintrag in "${SCHRITTE[@]}"; do
    nr=$((nr + 1))
    titel="${eintrag%%@@*}"; rest="${eintrag#*@@}"
    befehl="${rest%%@@*}"; hinweis="${rest#*@@}"
    if bash -c "$befehl" >/dev/null 2>&1; then
        printf '  [%d] OK    %s\n' "$nr" "$titel"
    else
        printf '  [%d] FAIL  %s\n' "$nr" "$titel"
        printf '        Hinweis: %s\n' "$hinweis"
        [[ -z $erstes_problem ]] && erstes_problem="$titel"
    fi
done

if [[ -z $erstes_problem ]]; then
    echo "Ergebnis: alle Diagnoseschritte bestanden - weiter mit Logs (Run-ID suchen, Modul 9.1)"
else
    echo "Ergebnis: erstes Problem bei: $erstes_problem"
    echo "Eskalation: $ESKALATION"
    exit 1
fi
```

Die Schritte stehen als Datensätze in einem Array: Titel, Prüfbefehl und Hinweis. Eine Schleife führt jeden Prüfbefehl aus und gibt OK oder FAIL aus; bei FAIL erscheint der Hinweis, am Ende der erste fehlgeschlagene Schritt und die Eskalation. Als Trennzeichen dient `@@`, weil die Prüfbefehle selbst Pipes enthalten. Das Beispiel zeigt, dass ein ausführbares Runbook im Kern eine *Tabelle aus Frage, Prüfung und Antwort* ist, eine gute Struktur auch für ein Dokument. Die Daten sind als Zeichenketten abgelegt, die per `bash -c` ausgeführt werden; das ist hier akzeptabel, weil die Befehle *fest im Skript* stehen und keine fremden Eingaben enthalten (Modul 8.3). In der Testumgebung sind zwei Schritte fehlgeschlagen: Es liegen keine neuen Dateien vor, und die Sperrdatei verweist auf einen nicht existierenden Prozess (die Prüfung `kill -0 PID` fragt, ob der Prozess existiert, ohne ein Signal zu senden). Das Skript *behebt nichts* von selbst; es führt die Person zum Problem und nennt den nächsten Schritt.

Typische Fehler: Erstens werden Runbooks als lange Fließtexte geschrieben, die nachts niemand lesen mag; kurze nummerierte Schritte mit Befehlen sind besser. Zweitens stehen Befehle ohne erwartete Ausgabe da: Man weiß nicht, ob das Ergebnis „gut“ ist. Drittens veralten Runbooks, weil niemand zuständig ist; ein Datum der letzten Prüfung und ein Verantwortlicher gehören hinein. Viertens enthalten sie Geheimnisse (Passwörter, Tokens) im Klartext (Modul 8.4); sie verweisen auf den Secret Store. Fünftens fehlt die Verifikation: Nach der Behebung erfährt man nicht, woran man erkennt, dass alles wieder in Ordnung ist. Sechstens werden riskante Schritte (Löschen, Neustart) ohne Warnung und Bestätigung beschrieben. Siebtens fehlt die Eskalation, und die Person versucht stundenlang allein, ein Problem zu lösen, das ohnehin jemand anderes beheben müsste.

```markdown
# Vorlage fuer ein Runbook (Markdown), im selben Repository wie das Skript
# Runbook: <Alarmname / Symptom>
Verantwortlich: <Team>   Zuletzt geprueft: 2026-10-30   Skript: ./importjob.sh

## Auswirkung
Wer/was ist betroffen? Prioritaet? Maximale Ausfallzeit?

## Diagnose
1. `./runbook-import.sh`           -> erwartet: alle Schritte OK
2. `journalctl -u import --since "-1h" -p warning --no-pager`

## Behebung
1. (sicher)  Job neu starten:      `systemctl restart import`
2. (RISKANT, nur nach Absprache)   Sperrdatei entfernen: `rm /var/run/import.pid`

## Verifikation
`systemctl is-active import` -> active; Metrik import_last_success_timestamp_seconds aktuell

## Eskalation
Nach 30 min ohne Besserung: Bereitschaft anrufen (Nummer im Wiki), Ticket mit Ausgabe von Schritt 1 anhaengen
```

> **Merke**
>
> Runbook = Symptom, Auswirkung, Diagnose mit Befehlen und erwarteten Ergebnissen, Behebung, Verifikation, Eskalation, Verantwortlicher, Datum. Beim Code halten und mitpflegen. Ausführbare Diagnose-Skripte sparen Zeit; riskante Schritte bleiben manuell oder mit Bestätigung. Keine Geheimnisse im Text.

> **Praxisbeispiel**
>
> Jeder Alarm im Monitoring enthält einen Link auf das passende Runbook. Nach jeder Störung wird das Runbook in der Nachbesprechung geprüft: Was fehlte, was war überflüssig? Neue Kolleginnen und Kollegen können so schon in der ersten Bereitschaftswoche übliche Störungen allein bearbeiten.

## 9.9 Übungsaufgaben zu Modul 09

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Nennen Sie die Pflichtbestandteile einer guten Logzeile und erklären Sie, wozu eine Run-ID dient. Warum baut man JSON-Logzeilen nicht mit `echo` und Variablen zusammen?

**Aufgabe 2.** Welche Bedeutung haben die Exit-Codes 2, 126, 127, 130 und 143? Warum ist `exit 256` ein Fehler, und was bedeutet UNKNOWN (Exit-Code 3) in der Nagios-Konvention?

**Aufgabe 3.** Warum sollte ein Prometheus-Textfile atomar geschrieben werden, und warum ist der Zeitstempel des letzten Erfolgs oft wichtiger als die Anzahl der Fehler? Nennen Sie ein Beispiel für ein schlechtes Label.

**Aufgabe 4.** Skripte laufen interaktiv, aber nicht in Cron. Nennen Sie fünf Umgebungsunterschiede, die dafür infrage kommen, und beschreiben Sie, wie Sie das Problem reproduzieren, ohne auf den Cron-Lauf zu warten.

**Aufgabe 5.** Schreiben Sie eine Funktion `log LEVEL MELDUNG [k=v ...]` für JSON-Lines-Ausgabe nach stderr mit `jq`, Run-ID aus der Umgebung (wird erzeugt, falls nicht gesetzt) und Mindestlevel aus `LOG_LEVEL`. Testen Sie mit Werten, die Anführungszeichen, Backslashes und Zeilenumbrüche enthalten.

**Aufgabe 6.** Schreiben Sie `job-metriken.sh BEFEHL...`, das einen beliebigen Befehl ausführt, Laufzeit (Sekunden mit Nachkommastellen), Exit-Code und Zeitstempel des Endes misst und die drei Metriken atomar in eine `.prom`-Datei im Verzeichnis `$TEXTFILE_DIR` schreibt. Das Skript soll selbst den Exit-Code des Befehls zurückgeben.

**Aufgabe 7.** Schreiben Sie `journal-fehler.sh DATEI.json`, das aus einem JSON-Lines-Journalauszug (Format wie in 9.5) die Fehlerzeilen (Priorität höchstens 3) gruppiert nach Quelle zählt, je Quelle die erste und letzte Meldung samt Uhrzeit anzeigt und mit Exit-Code 1 endet, wenn mindestens ein Fehler gefunden wurde.

**Aufgabe 8.** Schreiben Sie `lint-skripte.sh VERZ`, das alle `*.sh`-Dateien in VERZ prüft: (a) CRLF-Zeilenenden, (b) fehlende Shebang-Zeile, (c) Ausführrecht fehlt, (d) Syntaxfehler laut `bash -n`. Je Datei eine Zeile mit allen gefundenen Problemen; Exit-Code 1, wenn irgendeine Datei Probleme hat.

## 9.10 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Zeitstempel in UTC (ISO 8601), Level, Komponente/Skriptname, Meldung und Schlüssel-Wert-Paare für Details (Datei, Anzahl, Dauer, Exit-Code); dazu eine Run-ID. Die Run-ID ist eine Kennung, die ein Lauf einmal erzeugt und in jede Zeile schreibt (und an Kindprozesse vererbt): Man kann damit alle Zeilen eines Laufs auch zwischen parallelen Läufen herausfiltern und über Systemgrenzen korrelieren. JSON per `echo` ist ungültig, sobald ein Wert Anführungszeichen, Backslashes oder Zeilenumbrüche enthält, und erlaubt das Einschleusen zusätzlicher Felder (Injection). `jq -n -{}-arg` maskiert korrekt.

**Lösung zu Aufgabe 2.** 2: falsche Benutzung (Konvention vieler Werkzeuge); 126: Befehl gefunden, aber nicht ausführbar (Rechte, falsches Format); 127: Befehl nicht gefunden; 130: Abbruch durch Strg-C (128 + 2, SIGINT); 143: beendet durch SIGTERM (128 + 15). Exit-Codes werden auf 8 Bit begrenzt (modulo 256), daher ergibt `exit 256` den Code 0, also scheinbaren Erfolg; bei Fehlerzählern deshalb auf 1 begrenzen. UNKNOWN (3): Der Check konnte keine Aussage treffen (Zeitüberschreitung, fehlende Daten, eigener Fehler); das ist nicht dasselbe wie „kritisch“ und soll anders behandelt werden.

**Lösung zu Aufgabe 3.** Der Exporter liest die Datei bei jedem Scrape; schreibt man mit `>` direkt hinein, kann er eine leere oder halbe Datei sehen und liefert falsche Werte. Atomar: in eine temporäre Datei im selben Verzeichnis schreiben und per `mv` (rename) ersetzen. Der Zeitstempel des letzten Erfolgs erkennt auch den Fall, dass der Job gar nicht mehr läuft (dann gibt es keine Fehler, nur Stille): Ein Alarm „Zeitstempel älter als erwartet“ deckt das ab. Schlechtes Label: Benutzer-ID, Dateiname oder Request-ID, weil unbegrenzt viele Werte eine Kardinalitätsexplosion auslösen.

**Lösung zu Aufgabe 4.** Kurzer `PATH`, fehlendes Terminal (`tty`, `TERM`), andere oder fehlende Locale (`LANG`, `LC_*`), anderes Home und Arbeitsverzeichnis, anderer Benutzer und Rechte, fehlende Variablen aus Profilen (`.bashrc`, `.profile`), kein SSH-Agent oder Proxy-Einstellungen, andere Shell (`sh` statt `bash`). Reproduktion: `env -i HOME="$HOME" PATH=/usr/bin:/bin SHELL=/bin/sh bash -c 'skript' </dev/null`, gegebenenfalls zusätzlich als der Benutzer, unter dem Cron läuft (`sudo -u benutzer env -i ...`). Zusätzlich die Umgebung beider Läufe mit `env | sort` festhalten und mit `diff` vergleichen.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# jsonlog.sh
set -euo pipefail
RUN_ID="${RUN_ID:-$(cut -c1-8 /proc/sys/kernel/random/uuid)}"; export RUN_ID
LOG_LEVEL="${LOG_LEVEL:-INFO}"
declare -A LVL=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40)

log() {                                    # log LEVEL MELDUNG [k=v ...]
    local level="${1:?}" msg="${2:?}"; shift 2
    [[ -n ${LVL[$level]:-} ]] || { echo "unbekanntes Level: $level" >&2; return 2; }
    (( ${LVL[$level]} >= ${LVL[$LOG_LEVEL]:-20} )) || return 0
    local args=(--arg ts "$(date -u +%FT%TZ)" --arg level "$level" --arg run_id "$RUN_ID" --arg msg "$msg")
    local filter='{ts:$ts,level:$level,run_id:$run_id,msg:$msg' kv i=0
    for kv in "$@"; do
        [[ $kv == *=* ]] || continue
        args+=(--arg "k$i" "${kv#*=}")
        filter+=",$(jq -n -c --arg k "${kv%%=*}" '$k'):\$k$i"      # Schluessel sicher als JSON-String
        i=$((i + 1))
    done
    jq -n -c "${args[@]}" "$filter}" >&2
}

log INFO "Start" pid=$$
log DEBUG "unsichtbar"
log ERROR 'Fehler mit "Quotes", \Backslash und Zeilenumbruch' detail=$'a\nb' pfad='C:\temp\neu'
```

Die Werte kommen ausschließlich als `-{}-arg`, nie als Teil des Filtertexts; auch die Schlüssel werden mit `jq` zu gültigen JSON-Strings, so kann kein Schlüsselname den Filter verändern. Das Level wird gegen das Array geprüft, ein unbekanntes Level liefert Exit-Code 2. Die Ausgabe ist gültiges JSON auch bei Sonderzeichen, was man mit `2>&1 | jq .` prüfen kann.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# job-metriken.sh BEFEHL [ARG...]
set -uo pipefail
(( $# >= 1 )) || { echo "Aufruf: $0 BEFEHL [ARG...]" >&2; exit 2; }
verz="${TEXTFILE_DIR:?TEXTFILE_DIR fehlt}"
[[ -d $verz ]] || { echo "Verzeichnis fehlt: $verz" >&2; exit 2; }
name="${JOB_NAME:-job}"
[[ $name =~ ^[a-z_][a-z0-9_]*$ ]] || { echo "JOB_NAME ungueltig" >&2; exit 2; }

start="${EPOCHREALTIME/,/.}"
"$@"; rc=$?
ende="${EPOCHREALTIME/,/.}"
dauer="$(awk -v a="$start" -v b="$ende" 'BEGIN { printf "%.3f", b - a }')"

tmp="$(mktemp "$verz/.$name.XXXXXX")"
cat > "$tmp" <<ENDE
# TYPE ${name}_duration_seconds gauge
${name}_duration_seconds $dauer
# TYPE ${name}_exit_code gauge
${name}_exit_code $rc
# TYPE ${name}_end_timestamp_seconds gauge
${name}_end_timestamp_seconds ${ende%.*}
ENDE
chmod 644 "$tmp"
mv -f -- "$tmp" "$verz/$name.prom"
exit "$rc"
```

Der Befehl läuft unverändert mit seinen Argumenten (`"$@"`), sein Exit-Code wird sofort gesichert. Die Zeiten kommen aus `EPOCHREALTIME`; die Dezimaldifferenz berechnet `awk`. Geschrieben wird atomar: temporäre Datei im Zielverzeichnis, Leserechte, dann `mv`. Das Skript gibt den Code des Befehls zurück und lässt sich deshalb transparent in Cron- und CI-Aufrufe einfügen: `TEXTFILE_DIR=/var/lib/node_exporter JOB_NAME=backup ./job-metriken.sh ./backup.sh`.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# journal-fehler.sh DATEI.json
set -euo pipefail
datei="${1:?Aufruf: $0 DATEI.json}"
[[ -r $datei ]] || { echo "nicht lesbar: $datei" >&2; exit 2; }

fehler="$(jq -c 'select((.PRIORITY|tonumber) <= 3)
    | {quelle: (._SYSTEMD_UNIT // .SYSLOG_IDENTIFIER // ._TRANSPORT // "unbekannt"),
       zeit: (.__REALTIME_TIMESTAMP|tonumber/1000000|floor),
       msg: .MESSAGE}' "$datei")"

[[ -n $fehler ]] || { echo "keine Fehler gefunden"; exit 0; }

jq -r -s '
    group_by(.quelle)[]
    | sort_by(.zeit) as $z
    | [ $z[0].quelle, ($z|length),
        ($z[0].zeit|strftime("%H:%M:%S")),  $z[0].msg,
        ($z[-1].zeit|strftime("%H:%M:%S")), $z[-1].msg ] | @tsv' <<< "$fehler" |
awk -F'\t' 'BEGIN { printf "%-14s %6s  %s\n", "QUELLE", "ANZAHL", "ERSTE / LETZTE MELDUNG" }
            { printf "%-14s %6d  %s %s\n%-22s %s %s\n", $1, $2, $3, $4, "", $5, $6 }'
exit 1
```

Der erste `jq`-Aufruf filtert die Fehlerzeilen und vereinheitlicht sie zu kleinen Objekten (Quelle mit Alternativen über `//`); der zweite gruppiert mit `-s` (alle Zeilen als Array), sortiert je Gruppe nach Zeit und gibt Anzahl sowie erste und letzte Meldung aus. `awk` formatiert die Tabelle. Der Exit-Code 1 bei Fehlern erlaubt die Nutzung in Prüfungen und Skripten. Mit den Beispieldaten aus 9.5 erscheinen zwei Quellen: der Dienst `app.service` mit zwei Fehlern und `import-job` sowie der Kernel mit je einem.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# lint-skripte.sh VERZ
set -uo pipefail
verz="${1:?Aufruf: $0 VERZ}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }

problematisch=0
while IFS= read -r -d '' datei; do
    probleme=()
    grep -q $'\r' "$datei"                 && probleme+=("CRLF")
    [[ "$(head -c 2 "$datei")" == '#!' ]]  || probleme+=("keine Shebang")
    [[ -x $datei ]]                        || probleme+=("nicht ausfuehrbar")
    if ! fehler="$(bash -n "$datei" 2>&1)"; then probleme+=("Syntaxfehler: ${fehler%%$'\n'*}"); fi
    if (( ${#probleme[@]} )); then
        problematisch=$((problematisch + 1))
        ( IFS=';'; printf '%s: %s\n' "$datei" "${probleme[*]}" )
    else
        printf '%s: ok\n' "$datei"
    fi
done < <(find "$verz" -type f -name '*.sh' -print0 | sort -z)

(( problematisch == 0 )) || { echo "$problematisch Datei(en) mit Problemen" >&2; exit 1; }
```

Dateinamen werden über Nullbytes gelesen (korrekt auch bei Leerzeichen). Jede Prüfung ergänzt ein Array; mit `IFS=';'` in einer Subshell entsteht die Ausgabezeile mit allen Problemen. Die Schleife liest aus einer Prozesssubstitution, damit der Zähler in der Hauptshell bleibt. Bei CRLF-Dateien meldet `bash -n` oft zusätzlich einen Syntaxfehler, der aus dem Wagenrücklauf entsteht; das ist ein nützlicher Hinweis auf die Ursache. Das Skript lässt sich als Pre-Commit-Hook oder CI-Schritt einsetzen (Modul 10.3 und 10.4).
