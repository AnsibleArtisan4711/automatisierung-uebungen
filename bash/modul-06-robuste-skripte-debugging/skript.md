# Modul 06 - Robuste Skripte, Fehlerbehandlung und Debugging

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (6.1 bis 6.9; 6.10 Übungsaufgaben, 6.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 12 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Als Richtwert für den Umfang gilt der höhere Wert von 12 h.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit              | Dauer (h) | Prio |
|:----|:-------------------------|----------:|:----:|
| 6.1 | set-Optionen             |      1,00 |  1   |
| 6.2 | Fehlerbehandlung         |      1,00 |  1   |
| 6.3 | Defensive Programmierung |      1,00 |  1   |
| 6.4 | Debugging                |      1,00 |  1   |
| 6.5 | Logging in Skripten      |      1,00 |  1   |
| 6.6 | Temporäre Dateien        |      1,00 |  1   |
| 6.7 | Idempotenz               |      1,00 |  2   |
| 6.8 | Modularisierung          |      1,00 |  2   |
| 6.9 | Dokumentation            |      1,00 |  2   |
|     | **Summe (Details)**      |  **9,00** |      |

Ein Skript, das auf dem eigenen Rechner einmal funktioniert, ist noch kein gutes Skript. Es soll auch dann zuverlässig arbeiten, wenn Eingaben fehlen, Dateien unerwartet heißen, ein Netzwerk ausfällt oder jemand es zum hundertsten Mal startet. Dieses Modul zeigt, wie man Skripte *robust*, *wartbar* und *nachvollziehbar* macht: durch die richtigen Shell-Optionen, durchdachte Fehlerbehandlung, defensive Programmierung, Debugging-Werkzeuge, Logging, sichere temporäre Dateien, Wiederholbarkeit, Modularisierung und Dokumentation.

## 6.1 set-Optionen

Das Builtin `set` verändert das Verhalten der Shell. Vier Optionen bilden den Kern dessen, was man umgangssprachlich den *Strict Mode* nennt. `set -e` (`errexit`) beendet das Skript, sobald ein einfacher Befehl mit einem Fehlercode ungleich null endet. `set -u` (`nounset`) macht die Verwendung einer nicht gesetzten Variablen zum Fehler, statt sie stillschweigend durch einen leeren String zu ersetzen. `set -o pipefail` lässt eine Pipeline fehlschlagen, wenn irgendeine Stufe fehlschlägt, nicht nur die letzte (siehe Modul 4.2). `set -x` (`xtrace`) gibt jeden Befehl vor der Ausführung aus und dient dem Debugging (6.4). Üblich ist die kombinierte Schreibweise `set -euo pipefail` am Skriptanfang. Mit `set +e` und so weiter schaltet man Optionen wieder ab; `set -o` listet alle Optionen mit ihrem Zustand.

Jede der drei Schutzoptionen löst ein reales Problem. `set -u` fängt Tippfehler in Variablennamen ab: Ohne die Option liest `rm -rf "$VERZEISCHNIS/"` einen leeren Wert und löscht die Wurzel (siehe Modul 2.1 und 3.3). `pipefail` verhindert, dass ein fehlgeschlagener Befehl am Anfang einer Pipeline untergeht. `set -e` verhindert, dass ein Skript nach einem Fehler einfach weiterläuft und womöglich Schaden anrichtet.

Allerdings ist `set -e` *berüchtigt* für Ausnahmen. Die Beschreibung auf der Wiki-Seite BashFAQ/105 (abgerufen) nennt unter anderem diese Kontexte, in denen ein Fehler *nicht* zum Abbruch führt: Befehle in einer `if`- oder `while`-Bedingung, Befehle links von `&&` und `||`, alle Stufen einer Pipeline außer der letzten (ohne `pipefail`), Funktionen, die als Bedingung aufgerufen werden, sowie Befehlsersetzungen, in denen `set -e` nicht automatisch gilt (Abhilfe seit Bash 4.4: `shopt -s inherit_errexit`). Außerdem verdecken `local x=$(...)` und `export x=$(...)` den Fehlercode. Die Seite erwähnt auch das Gegenteil: `let i++` bei `i=0` liefert den Status 1 und beendet das Skript, obwohl nichts schiefging. Die Autoren der Seite sind sich uneins, ob man `set -e` überhaupt verwenden sollte; ein Teil rät, Fehler ausschließlich explizit zu behandeln.

> **Ergänzung aus der Nachrecherche**
>
> Quelle: Wiki-Seite „Why doesn’t set -e (or set -o errexit, or trap ERR) do what I expected?“ (BashFAQ/105, abgerufen am 30.10.2026). Die Seite bewertet den „Strict Mode“ als wenig tragfähig, weil sich die Semantik zwischen Bash-Versionen unterscheidet und man jede Zeile, die fehlschlagen darf, gesondert absichern muss. Dieses Skript empfiehlt deshalb einen pragmatischen Mittelweg: `set -euo pipefail` als *Sicherheitsnetz* verwenden, kritische Schritte aber trotzdem ausdrücklich prüfen.

```bash
#!/usr/bin/env bash
# strict-demo.sh - zeigt Wirkung und Grenzen von set -e, -u, pipefail
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

teste() {   # fuehrt Code in einer eigenen Shell aus und meldet das Ergebnis
    local name="$1" code="$2" out status
    out="$(bash -c "$code" 2>&1)"; status=$?
    printf '%-34s -> Status %-3s Ausgabe: %s\n' "$name" "$status" "$(echo "$out" | tr '\n' ' ')"
}

echo "=== 1. ohne Strict Mode laeuft alles weiter"
teste "false; echo weiter"          'false; echo weiter'

echo "=== 2. set -e bricht ab"
teste "set -e; false; echo weiter"  'set -e; false; echo weiter'

echo "=== 3. Ausnahmen: if, && und ||"
teste "if false"                    'set -e; if false; then :; fi; echo weiter'
teste "false || true"               'set -e; false || true; echo weiter'
teste "false && echo x; echo weiter" 'set -e; false && echo x; echo weiter'

echo "=== 4. Pipeline: erste Stufe geht unter, pipefail faengt sie"
teste "false | cat"                 'set -e; false | cat; echo weiter'
teste "pipefail: false | cat"       'set -eo pipefail; false | cat; echo weiter'

echo "=== 5. set -u"
teste "ohne -u: leere Variable"     'echo "wert=[$nichtda]"'
teste "mit -u: Fehler"              'set -u; echo "wert=[$nichtda]"'
teste "mit -u: Default"             'set -u; echo "wert=[${nichtda:-leer}]"'

echo "=== 6. Fallen von set -e"
teste "((i++)) bei i=0"             'set -e; i=0; ((i++)); echo weiter'
teste "i=\$((i+1)) ist sicher"      'set -e; i=0; i=$((i+1)); echo weiter'
teste "local x=\$(false) verdeckt"  'set -e; f() { local x=$(false); echo weiter; }; f'
teste "local x; x=\$(false)"        'set -e; f() { local x; x=$(false); echo weiter; }; f'
teste "f in if: set -e unterdrueckt" 'set -e; f() { false; echo "in f weiter"; }; if f; then echo ok; fi'

echo "=== 7. grep ohne Treffer ist kein Fehler im Sinne des Skripts"
teste "grep ohne Treffer"           'set -e; echo a | grep b; echo weiter'
teste "grep ... || true"            'set -e; echo a | grep b || true; echo weiter'
```

Das Skript führt die Experimente jeweils in einer *eigenen* Shell aus (`bash -c`), damit `set -e` nicht das Demo-Skript selbst beendet. Man sieht nacheinander: Ohne Strict Mode läuft alles weiter. Mit `set -e` bricht das Skript beim ersten Fehler ab, außer der Fehler tritt in einer Bedingung oder links von `&&` beziehungsweise `||` auf. In einer Pipeline geht der Fehler der ersten Stufe verloren, bis `pipefail` gesetzt ist. `set -u` meldet eine nicht gesetzte Variable; der Standardwert `${var:-}` ist die Standardmethode, eine möglicherweise ungesetzte Variable trotzdem zu lesen. Block 6 zeigt die Fallen: `((i++))` liefert beim Wert 0 den Status 1, weil der Ausdruck den *alten* Wert 0 auswertet und 0 „falsch“ ist; die Zuweisung `i=$((i+1))` ist sicher. `local x=$(false)` verdeckt den Fehler, die getrennte Schreibweise nicht. Eine Funktion in einer `if`-Bedingung läuft ganz ohne `set -e`: Der Fehler von `false` in der Funktion wird ignoriert. Schließlich ist ein `grep` ohne Treffer ein vom Programm vorgesehener Status 1, den man bewusst mit `|| true` quittieren muss, wenn „kein Treffer“ erlaubt ist.

Typische Fehler: Erstens verlässt man sich auf `set -e` als einzige Fehlerbehandlung. Zweitens setzt man es am Skriptende statt am Anfang. Drittens aktiviert man `set -u` in Skripten, die Bibliotheken von Drittanbietern sourcen (etwa Bash-Completion oder `nvm`); diese greifen oft auf ungesetzte Variablen zu und brechen ab. Hier schaltet man `set +u` kurz ab (`set +u; source ...; set -u`). Viertens: `set -e` wird beim Sourcen oder in Subshells unterschiedlich vererbt; wer sich unsicher ist, testet explizit.

```bash
# falsch: set -e als einziger Schutz, Fehler in der Bedingung unbemerkt
set -e
if ! pruefe_konfiguration; then      # Fehler innerhalb der Funktion bleibt unentdeckt
    echo "Konfiguration ungueltig"
fi

# richtig: kritische Schritte ausdruecklich pruefen und Status auswerten
set -euo pipefail
pruefe_konfiguration || { echo "Konfiguration ungueltig" >&2; exit 1; }
```

> **Merke**
>
> `set -euo pipefail` ist ein Sicherheitsnetz, kein Ersatz für Fehlerbehandlung. `-e` greift nicht in Bedingungen, bei `&&`/`||`, in Pipelines ohne `pipefail` und wird durch `local x=$(...)` verdeckt. `-u` verlangt `${var:-}` für optionale Variablen. Kritische Schritte immer ausdrücklich prüfen.

> **Praxisbeispiel**
>
> Fast jedes Team-Skript beginnt mit der Zeile `set -euo pipefail` und einer Fehlerfalle (6.2). Dadurch bricht ein Deployment bei einem fehlgeschlagenen Kopierbefehl sofort ab, statt mit halb kopierten Dateien weiterzulaufen. Skripte mit absichtlich tolerierten Fehlern kennzeichnen diese Stellen mit `|| true` und einem Kommentar.

## 6.2 Fehlerbehandlung

Fehlerbehandlung heißt, dass ein Skript Fehler *bemerkt*, *meldet* und kontrolliert *reagiert*: aufräumen, einen sinnvollen Exit-Code liefern, gegebenenfalls erneut versuchen oder Alternativen nutzen. Die Grundlage ist der Exit-Code jedes Befehls (Modul 4.2). Es gibt drei grundsätzliche Strategien. Erstens *abbrechen* (*fail fast*): Bei einem nicht behebbaren Fehler sofort mit Meldung und Fehlercode enden. Zweitens *fortfahren und sammeln*: Bei Massenverarbeitung nicht beim ersten Fehler stoppen, sondern alle Fehler merken, weitermachen und am Ende gesammelt melden. Drittens *wiederholen* (*retry*): Bei vorübergehenden Fehlern (Netzwerk, ausgelastete Dienste) mehrere Versuche mit Wartezeit.

Praktische Bausteine: Die *Fehlerfunktion* `die()` gibt eine Meldung nach stderr aus und beendet mit einem Code. Ein `trap ... ERR` (Modul 4.7) kann Zeile und Befehl eines unerwarteten Fehlers ausgeben. Ein `trap ... EXIT` räumt in jedem Fall auf. Die Bash-Variablen `$LINENO`, `$BASH_COMMAND`, `${FUNCNAME[@]}` und `${BASH_SOURCE[@]}` liefern Kontext für aussagekräftige Meldungen. Eine gute Fehlermeldung sagt, *was* schiefging, *wo* (Datei, Zeile, Wert) und möglichst *was zu tun ist*.

Für *Exit-Codes* gelten Konventionen. 0 bedeutet Erfolg, 1 allgemeinen Fehler, 2 falsche Benutzung. Größere Werkzeuge definieren eigene Codes (etwa 3 = Konfigurationsfehler, 4 = Netzwerk). Die Bereiche 126 bis 165 sind für Shell und Signale reserviert (siehe Modul 4.2); eigene Codes sollten darunter liegen. Wichtig ist, Codes zu *dokumentieren*, weil Cron, CI und Wrapper-Skripte sie auswerten.

```bash
#!/usr/bin/env bash
# fehler-demo.sh - Fehlerfunktion, ERR-Trap, Retry, Fehler sammeln
set -Eeuo pipefail            # -E: ERR-Trap auch in Funktionen und Subshells

readonly E_USAGE=2 E_NETZ=4 E_DATEN=5

die() {                       # die CODE MELDUNG...
    local code="$1"; shift
    printf 'FEHLER: %s\n' "$*" >&2
    exit "$code"
}

on_err() {                    # laeuft bei jedem unerwarteten Fehler
    local status=$?
    printf 'ABBRUCH in %s Zeile %s: "%s" (Status %s)\n' \
        "${FUNCNAME[1]:-main}" "${BASH_LINENO[0]}" "$BASH_COMMAND" "$status" >&2
}
trap on_err ERR

retry() {                     # retry ANZAHL PAUSE BEFEHL...
    local max="$1" pause="$2" versuch=1; shift 2
    until "$@"; do
        (( versuch >= max )) && return 1
        echo "Versuch $versuch/$max fehlgeschlagen, warte ${pause}s" >&2
        sleep "$pause"
        versuch=$((versuch + 1))
    done
}

# 1. Retry mit unzuverlaessigem Befehl (klappt erst beim 3. Mal)
zaehler=0
unzuverlaessig() { zaehler=$((zaehler + 1)); (( zaehler >= 3 )); }
retry 5 0.1 unzuverlaessig && echo "Erfolg nach $zaehler Versuchen"

# 2. Fehler sammeln statt abbrechen
fehlgeschlagen=()
for datei in a.txt b.txt c.txt; do
    if [[ $datei == b.txt ]]; then
        fehlgeschlagen+=("$datei")      # simuliert Fehler
        continue
    fi
    echo "verarbeitet: $datei"
done
if (( ${#fehlgeschlagen[@]} > 0 )); then
    echo "Fehler bei: ${fehlgeschlagen[*]}" >&2
fi

# 3. erwarteter Fehler: bewusst tolerieren und Status auswerten
if ! grep -q "muster" /etc/hostname; then
    echo "Muster nicht gefunden - das ist hier in Ordnung"
fi

# 4. unerwarteter Fehler loest den ERR-Trap aus
echo "Gleich kommt ein Fehler ..."
ls /gibt/es/nicht
echo "wird nie erreicht"
```

Die Option `-E` (`errtrace`) sorgt dafür, dass der `ERR`-Trap auch innerhalb von Funktionen und Befehlsersetzungen gilt; ohne sie würde er dort nicht ausgelöst. `die` nimmt einen Exit-Code und eine Meldung entgegen; mit `readonly` definierte Codes sind selbsterklärend. Der `ERR`-Trap gibt Funktion, Zeile, den fehlgeschlagenen Befehl (`$BASH_COMMAND`) und den Status aus, ein enormer Gewinn bei der Fehlersuche. Die Funktion `retry` führt einen beliebigen Befehl bis zu `max` Mal aus, mit `"$@"` korrekt gequotet, und wartet dazwischen. Block 2 zeigt das Muster „Fehler sammeln“; die Schleife läuft weiter, und erst nach der Schleife wird gemeldet. Block 3 behandelt den Fall, dass „nicht gefunden“ kein Fehler ist, innerhalb einer `if`-Bedingung, die `set -e` nicht auslöst. Im letzten Block bricht das Skript beim ersten unerwarteten Fehler mit einer Diagnose ab.

Typische Fehler: Erstens sind Fehlermeldungen unbrauchbar („Fehler aufgetreten“). Zweitens schreibt man Fehlermeldungen auf stdout statt stderr und vermischt sie mit Ergebnissen. Drittens verschluckt man Fehler mit `2>/dev/null` oder `|| true`, ohne zu wissen, was man verbirgt. Viertens gibt man immer Code 1 zurück, sodass Aufrufer verschiedene Fehlerarten nicht unterscheiden können. Fünftens hat die Fehlerbehandlung selbst einen Fehler: Der `ERR`-Trap oder Aufräumcode schlägt fehl und überdeckt den ursprünglichen Fehler. Sechstens wiederholt man Befehle, die nicht *idempotent* sind (siehe 6.7), etwa eine Zahlung oder das Anlegen eines Datensatzes.

```bash
# falsch: Fehler wird verschluckt, Ursache bleibt unsichtbar
cp quelle.txt ziel/ 2>/dev/null || true

# richtig: Fehler erkennen, erklaeren und mit passendem Code enden
cp -- quelle.txt ziel/ || die "$E_DATEN" "Kopieren von quelle.txt nach ziel/ fehlgeschlagen"
```

> **Merke**
>
> Fehler bemerken, melden (stderr, mit Ort und Ursache), kontrolliert reagieren (abbrechen, sammeln oder wiederholen). Eigene Exit-Codes definieren und dokumentieren. `trap ... ERR` mit `set -E` liefert Diagnose, `trap ... EXIT` räumt auf. Nichts stillschweigend verschlucken.

> **Praxisbeispiel**
>
> Ein Backup-Skript wird von Cron gestartet. Bei Fehlern schreibt es Zeile und Ursache in das Log und beendet sich mit Code 4 für Netzwerkprobleme oder 5 für Datenprobleme. Das Monitoring wertet die Codes aus: Netzwerkfehler lösen nach drei Wiederholungen einen Alarm aus, Datenfehler sofort.

## 6.3 Defensive Programmierung

*Defensive Programmierung* bedeutet, ein Skript so zu schreiben, als würde alles schiefgehen, was schiefgehen kann: Eingaben sind falsch oder leer, Dateien fehlen, Befehle sind nicht installiert, Variablen enthalten Leerzeichen, Verzeichnisse sind nicht beschreibbar, das Skript wird aus dem falschen Ordner gestartet. Der Grundsatz lautet: *Prüfe Annahmen, bevor du handelst.* Das kostet wenige Zeilen am Anfang und erspart lange Fehlersuche und Schäden.

Ein bewährtes Schema besteht aus fünf Schritten. *Erstens* die Umgebung absichern: `set -euo pipefail`, ein definiertes `IFS`, ein sauberer `PATH` und eine sinnvolle `umask`. *Zweitens* Abhängigkeiten prüfen: alle benötigten Programme mit `command -v` (Modul 1.5), die Bash-Version, notwendige Dateien und Rechte. *Drittens* Eingaben validieren: Argumente, Umgebungsvariablen, Dateiinhalte; die Prüfung erfolgt über *Positivlisten* (was erlaubt ist), nicht über Negativlisten (was verboten ist). *Viertens* Standardwerte setzen (`${var:-default}`) und Pflichtwerte erzwingen (`${var:?meldung}`). *Fünftens* gefährliche Operationen absichern: Löschen und Überschreiben nur nach Prüfung des Ziels, `--` vor Dateinamen, Quoting überall, `cd` nur mit `||` und Fehlerbehandlung, Trockenlauf-Option für Änderungen.

Zu den klassischen gefährlichen Annahmen gehören: „Die Variable ist gesetzt“, „der Dateiname enthält keine Leerzeichen“, „das Verzeichnis existiert“, „der Befehl liefert immer dasselbe Format“, „die Locale ist Deutsch“, „ich bin als der richtige Benutzer angemeldet“, „es läuft nur eine Instanz des Skripts“. Jede dieser Annahmen lässt sich mit einer kurzen Prüfung in eine bewusste Entscheidung verwandeln.

```bash
#!/usr/bin/env bash
# bereinige.sh - loescht alte Dateien in einem Verzeichnis (defensiv geschrieben)
set -euo pipefail
IFS=$'\n\t'                                  # Word Splitting nur an Zeilenumbruch und Tab
export LC_ALL=C PATH="/usr/local/bin:/usr/bin:/bin"
umask 077

readonly PROG="${0##*/}"
usage() { echo "Aufruf: $PROG [-n] [-t TAGE] VERZEICHNIS" >&2; exit 2; }

trocken=0
tage=30
while getopts ':nt:h' opt; do
    case "$opt" in
        n) trocken=1 ;;
        t) tage="$OPTARG" ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))
(( $# == 1 )) || usage
ziel="$1"

# --- Abhaengigkeiten pruefen
for tool in find realpath; do
    command -v "$tool" >/dev/null 2>&1 || { echo "Fehler: '$tool' fehlt" >&2; exit 1; }
done

# --- Eingaben validieren (Positivliste)
[[ $tage =~ ^[0-9]{1,4}$ ]] || { echo "Fehler: TAGE muss eine Zahl sein" >&2; exit 2; }
[[ -d $ziel ]]              || { echo "Fehler: '$ziel' ist kein Verzeichnis" >&2; exit 2; }
[[ -w $ziel ]]              || { echo "Fehler: '$ziel' ist nicht beschreibbar" >&2; exit 2; }

ziel="$(realpath -e -- "$ziel")"                # aufgeloester absoluter Pfad
case "$ziel" in
    /|/bin|/boot|/dev|/etc|/home|/lib*|/opt|/proc|/root|/run|/sbin|/sys|/usr|/var)
        echo "Fehler: Systemverzeichnis '$ziel' ist tabu" >&2; exit 2 ;;
esac

# --- gefaehrliche Operation: erst anzeigen, dann (optional) ausfuehren
anzahl=0
while IFS= read -r -d '' datei; do
    anzahl=$((anzahl + 1))
    if (( trocken )); then
        echo "wuerde loeschen: $datei"
    else
        rm -f -- "$datei"
    fi
done < <(find "$ziel" -xdev -type f -mtime "+$tage" -print0)

echo "$PROG: $anzahl Datei(en) $( ((trocken)) && echo gefunden || echo geloescht ) in $ziel"
```

Das Skript zeigt das Schema in Aktion. Die Umgebung wird festgelegt: `IFS=$'\n\t'` entschärft Word-Splitting-Fehler (Leerzeichen trennen nicht mehr), `LC_ALL=C` macht Ausgaben und Sortierung reproduzierbar, ein fester `PATH` verhindert, dass ein manipulierter PATH fremde Programme einschleust, und `umask 077` sorgt für restriktive Rechte neu erzeugter Dateien. Abhängigkeiten werden in einer Schleife geprüft. Die Validierung nutzt eine Positivliste: Die Tage müssen eine bis vier Ziffern sein, das Ziel muss ein beschreibbares Verzeichnis sein. `realpath -e` löst relative Pfade und Symlinks auf und scheitert, wenn das Ziel nicht existiert; damit lässt sich anschließend zuverlässig prüfen, dass kein Systemverzeichnis getroffen wird. Die Löschschleife liest nullterminierte Namen aus `find ... -print0` (siehe Modul 3.7 und 2.7) und beschränkt die Suche mit `-xdev` auf ein Dateisystem. Mit der Option `-n` gibt es einen Trockenlauf, der exakt dieselbe Schleife durchläuft, aber nichts löscht, was die Fehlerquelle „Trockenlauf macht etwas anderes als der echte Lauf“ vermeidet.

Typische Fehler: Erstens prüft man Eingaben *nach* der ersten gefährlichen Aktion. Zweitens nutzt man Negativlisten („kein Semikolon erlaubt“), die immer eine Lücke lassen. Drittens verlässt man sich auf relative Pfade und das aktuelle Verzeichnis. Viertens stellt man keine Trockenlauf-Option bereit und testet gefährliche Skripte direkt gegen echte Daten. Fünftens misst man „Erfolg“ nur an der Abwesenheit von Fehlern: Ein Skript, das 0 Dateien findet, weil das Ziel falsch war, meldet ebenfalls Erfolg; sinnvoll ist eine Plausibilitätsprüfung („es wurden 0 Dateien gefunden, ist das erwartet?“).

```bash
# falsch: Annahmen ungeprueft, Variable kann leer sein, Pfad kann beliebig sein
cd $BUILD_DIR
rm -rf *

# richtig: Pflichtvariable, cd mit Pruefung, gezieltes Loeschen
: "${BUILD_DIR:?BUILD_DIR ist nicht gesetzt}"
cd -- "$BUILD_DIR" || exit 1
[[ $PWD != / && $PWD == */build* ]] || { echo "Unerwartetes Verzeichnis: $PWD" >&2; exit 1; }
rm -rf -- ./*
```

> **Merke**
>
> Defensiv schreiben: Umgebung festlegen, Abhängigkeiten prüfen, Eingaben per Positivliste validieren, Pflichtwerte erzwingen, Pfade auflösen und prüfen, gefährliche Operationen mit Trockenlauf absichern. Jede unbewusste Annahme ist ein potenzieller Fehler.

> **Praxisbeispiel**
>
> Das Aufräumskript eines Teams hatte keine Pfadprüfung und löschte nach einem Tippfehler im Parameter ein produktives Datenverzeichnis. Seitdem verlangt der Code-Review bei jedem Skript, das `rm -rf` verwendet, eine Prüfung des Zielpfads, einen Trockenlauf-Schalter und eine Positivliste erlaubter Basisverzeichnisse.

## 6.4 Debugging

Fehler in Skripten lassen sich eingrenzen, wenn man systematisch vorgeht (Modul 9.6 behandelt die Methodik vertieft). Zuerst *reproduzieren*: Gibt es einen kleinsten Aufruf, der den Fehler zeigt? Dann *eingrenzen*: In welcher Zeile, mit welchem Wert? Schließlich *verstehen* und *beheben*. Bash bietet dafür mehrere Werkzeuge.

Die *Syntaxprüfung* `bash -n skript.sh` liest das Skript, ohne es auszuführen, und meldet Syntaxfehler (fehlendes `fi`, unbalancierte Anführungszeichen). Sie findet keine Laufzeitfehler. *ShellCheck* (siehe Modul 1.8) ist ein statischer Analysator, der darüber hinaus typische Fehlerquellen aufdeckt: fehlende Quotes (SC2086), nutzlose `cat`, unsichere `rm`-Aufrufe, falsche Test-Syntax, nicht portable Konstrukte. Es ist die wirksamste einzelne Maßnahme zur Fehlervermeidung und lässt sich in Editoren und CI-Pipelines einbinden. Einzelne Warnungen lassen sich mit einem Kommentar `# shellcheck disable=SC2086` begründet unterdrücken.

Die *Ablaufverfolgung* (Tracing) mit `set -x` oder dem Aufruf `bash -x skript.sh` gibt jeden ausgeführten Befehl nach der Expansion aus, mit einem vorangestellten Pluszeichen (der Wert der Variable `PS4`). Damit sieht man, welche Werte tatsächlich eingesetzt wurden. Mit `PS4` lässt sich die Ausgabe um Dateiname, Zeilennummer und Funktionsname ergänzen: `export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '`. Man kann Tracing auf Abschnitte begrenzen (`set -x ... set +x`) oder per Umgebungsvariable zuschaltbar machen (`[[ ${DEBUG:-0} == 1 ]] && set -x`). Weitere Optionen: `set -v` (zeigt Zeilen vor der Expansion), `bash -u`, `bash -e`.

Für gezielte Ausgaben nutzt man *Debug-Meldungen* auf stderr und die *Inspektion* von Variablen mit `declare -p variable` (zeigt Typ und Wert exakt, auch Arrays und Leerzeichen) und `printf '%q\n' "$wert"` (Shell-sichere Darstellung inklusive unsichtbarer Zeichen). Die Pseudo-Signale `DEBUG` und `RETURN` erlauben Schritt-für-Schritt-Debugger-ähnliche Hooks. Das Werkzeug `bashdb` existiert, wird aber selten genutzt. Bei externen Programmen helfen `strace` (Systemaufrufe) und `ltrace`.

```bash
#!/usr/bin/env bash
# debug-demo.sh
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '

berechne() {
    local a="$1" b="$2"
    echo $((a + b))
}

echo "--- 1. Ablaufverfolgung nur fuer einen Abschnitt"
set -x
ergebnis="$(berechne 2 3)"
liste=("a b" "c")
for e in "${liste[@]}"; do :; done
set +x
echo "ergebnis=$ergebnis"

echo "--- 2. Variablen exakt inspizieren"
text=$'zeile1\tTab\nzeile2 '
declare -p text liste
printf '%q\n' "$text"

echo "--- 3. Debug-Ausgabe zuschaltbar machen"
debug() { [[ ${DEBUG:-0} == 1 ]] && printf 'DEBUG: %s\n' "$*" >&2 || true; }
debug "wird nur bei DEBUG=1 gezeigt"
echo "Aufruf mit: DEBUG=1 ./debug-demo.sh"

echo "--- 4. Syntaxpruefung ohne Ausfuehrung"
printf 'if true; then\n  echo hi\n' > /tmp/kaputt.$$.sh
bash -n /tmp/kaputt.$$.sh 2>&1 | head -n 1
rm -f /tmp/kaputt.$$.sh

echo "--- 5. Ein typischer Fehler, den -x sichtbar macht"
datei="mein Bericht.txt"
set -x
[ -f $datei ] 2>/dev/null || echo "nicht gefunden"       # falsch: ohne Quotes
[ -f "$datei" ] 2>/dev/null || echo "nicht gefunden"     # richtig
set +x
```

Mit dem angepassten `PS4` zeigt jede Trace-Zeile Datei, Zeilennummer und Funktion, etwa `+ debug-demo.sh:6:berechne: local a=2 b=3`. Man sieht dort, dass Variablen *nach* der Expansion angezeigt werden, und erkennt so, welche Werte tatsächlich eingesetzt wurden. `declare -p` gibt den exakten Zustand der Variablen aus, inklusive Arrayelementen, und `printf '%q'` zeigt unsichtbare Zeichen wie Tabulatoren und Zeilenumbrüche in der Darstellung `$'...'`. Das Beispiel mit der Variable `datei` im letzten Block zeigt, wie der Trace den Quoting-Fehler offenlegt: Ohne Anführungszeichen wird `[ -f mein Bericht.txt ]` ausgeführt, also mit drei Argumenten, ein Fehler, der ohne Trace schwer zu sehen ist.

Typische Fehler beim Debuggen: Erstens verändert man mehrere Dinge gleichzeitig und weiß hinterher nicht, was gewirkt hat. Zweitens vertraut man der Vermutung statt der Messung; erst `set -x` oder `declare -p` zeigt, was wirklich passiert. Drittens lässt man Debug-Ausgaben in Produktivskripten stehen, die dann Geheimnisse in Logs schreiben: `set -x` gibt *alle* expandierten Werte aus, auch Passwörter in Variablen. Vor sensiblen Befehlen schaltet man Tracing ab. Viertens debuggt man im falschen Kontext: Ein Skript, das im Terminal läuft, verhält sich in Cron anders (andere Umgebung, anderer PATH, kein Terminal). Man reproduziert dort mit `env -i bash --noprofile --norc skript.sh`, was eine sehr karge Umgebung nachbildet.

```bash
# falsch: Passwort landet im Trace und damit im CI-Log
set -x
curl -u "admin:$PASSWORT" "$URL"

# richtig: Tracing fuer sensible Befehle ausschalten
set +x
curl -u "admin:$PASSWORT" "$URL"
[[ ${DEBUG:-0} == 1 ]] && set -x
```

> **Merke**
>
> Werkzeugkasten: `bash -n` (Syntax), ShellCheck (statische Analyse), `set -x` mit angepasstem `PS4` (Ablaufverfolgung), `declare -p` und `printf '%q'` (Variablen exakt ansehen), `env -i` (karge Umgebung). `set -x` gibt Geheimnisse preis; vor sensiblen Befehlen abschalten.

> **Praxisbeispiel**
>
> Ein Skript funktioniert lokal, schlägt aber in der CI-Pipeline fehl. Der Entwickler startet den Job mit `DEBUG=1`, liest im Log den Trace und sieht, dass eine Variable in der CI-Umgebung leer ist. Ursache war eine nur lokal gesetzte Umgebungsvariable, ein Fall für `${VAR:?}` (siehe 6.3).

## 6.5 Logging in Skripten

Logging macht das Verhalten eines Skripts *nachvollziehbar*, auch wenn niemand zusieht. Ohne Logs bleibt bei einem Cron-Job nur die Vermutung, was nachts passiert ist. Gutes Logging beantwortet fünf Fragen: *Wann*, *wer* (welches Skript, welche Instanz), *welche Wichtigkeit*, *was* und *womit* (welche Parameter oder Objekte).

Dazu gehören ein *Zeitstempel* (am besten ISO-8601 in UTC oder mit Zeitzone, etwa `date -u +%Y-%m-%dT%H:%M:%SZ`), ein *Log-Level* und die Nachricht. Üblich sind die Level `DEBUG`, `INFO`, `WARN` und `ERROR`, optional `FATAL`. Ein einstellbares Mindestlevel (etwa über eine Variable `LOG_LEVEL`) entscheidet, was ausgegeben wird. Wichtig ist die Trennung der Kanäle: Das *Ergebnis* des Skripts gehört nach stdout, *Logmeldungen* nach stderr (Modul 1.2). So bleibt die Ausgabe in Pipelines nutzbar, und der Aufrufer entscheidet, wohin die Logs gehen (Terminal, Datei, `logger`). Das Programm `logger` schreibt in das System-Log (syslog beziehungsweise journald), mit `logger -t name -p user.info "Text"`. Dort sammeln, rotieren und durchsuchen vorhandene Werkzeuge die Meldungen (Modul 9.5).

```bash
#!/usr/bin/env bash
# logging-demo.sh - einfache Logging-Bibliothek
set -euo pipefail

# --- Konfiguration (per Umgebungsvariable aenderbar)
LOG_LEVEL="${LOG_LEVEL:-INFO}"          # DEBUG, INFO, WARN, ERROR
LOG_FILE="${LOG_FILE:-}"                # leer = nur stderr
RUN_ID="${RUN_ID:-$(date +%s)-$$}"      # Korrelations-ID dieses Laufs

declare -A _LEVEL=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40)

log() {                                 # log LEVEL NACHRICHT...
    local level="$1"; shift
    (( ${_LEVEL[$level]} >= ${_LEVEL[$LOG_LEVEL]} )) || return 0
    local zeile
    zeile="$(date -u +%Y-%m-%dT%H:%M:%SZ) [$RUN_ID] ${level} ${0##*/}: $*"
    if [[ -t 2 && -z $LOG_FILE ]]; then            # Farbe nur im Terminal
        case $level in
            ERROR) printf '\033[31m%s\033[0m\n' "$zeile" >&2 ;;
            WARN)  printf '\033[33m%s\033[0m\n' "$zeile" >&2 ;;
            *)     printf '%s\n' "$zeile" >&2 ;;
        esac
    else
        printf '%s\n' "$zeile" >&2
    fi
    if [[ -n $LOG_FILE ]]; then
        printf '%s\n' "$zeile" >> "$LOG_FILE"
    fi
}
debug() { log DEBUG "$@"; }
info()  { log INFO  "$@"; }
warn()  { log WARN  "$@"; }
error() { log ERROR "$@"; }

# --- Anwendung
info "Skript gestartet"
debug "dieser Text erscheint nur bei LOG_LEVEL=DEBUG"
warn "Speicher knapp: 85 Prozent"
ergebnis=42
info "Verarbeitung abgeschlossen"
echo "$ergebnis"                        # Ergebnis auf stdout

# Rotation einfach loesen: alte Logs ueber Dateinamen nach Datum trennen
# LOG_FILE="/var/log/mein-skript-$(date +%F).log"
```

Die Bibliothek ist klein, aber vollständig. Ein assoziatives Array ordnet Leveln Zahlen zu, sodass die Funktion `log` entscheiden kann, ob eine Meldung unter dem Mindestlevel liegt; ohne Ausgabe gibt sie 0 zurück, damit `set -e` nicht auslöst. Die *Run-ID* identifiziert alle Zeilen eines Laufs, hilfreich, wenn mehrere Instanzen in dieselbe Datei schreiben. Farbe wird nur verwendet, wenn stderr ein Terminal ist (`-t 2`), damit Logdateien nicht mit Steuerzeichen verschmutzt werden. Das Ergebnis geht als `echo` auf stdout. Rufen Sie das Skript einmal normal, einmal mit `LOG_LEVEL=DEBUG` und einmal mit `LOG_FILE=/tmp/test.log` auf und vergleichen Sie die Ausgaben. Mit `./logging-demo.sh 2>/dev/null` sehen Sie nur das Ergebnis.

Typische Fehler beim Logging: Erstens fehlen Zeitstempel, und das Log ist für die Analyse wertlos. Zweitens schreibt man *Geheimnisse* (Passwörter, Tokens, personenbezogene Daten) in Logs; Logs werden gesichert, kopiert, an Supportstellen geschickt und selten so gut geschützt wie die Originaldaten (siehe Modul 8). Drittens ist alles `INFO`, und wichtige Meldungen gehen im Rauschen unter. Viertens wächst eine Logdatei unbegrenzt, bis die Platte voll ist; man benötigt `logrotate` oder datumsbasierte Dateinamen. Fünftens loggt man nur Erfolge und keine Parameter, sodass bei einem Fehler der Kontext fehlt. Sechstens benutzt man `echo` mit Variablen, die mit Bindestrich beginnen (`echo "$msg"` mit `-n` als Inhalt); `printf '%s\n'` ist robuster.

```bash
# falsch: kein Zeitstempel, Level unklar, Passwort im Log
echo "Verbinde mit $HOST als $USER mit Passwort $PASSWORT"

# richtig: strukturiert, ohne Geheimnisse
info "Verbinde mit host=$HOST user=$USER"
```

> **Merke**
>
> Log-Zeile = Zeitstempel + Run-ID + Level + Nachricht. Logs nach stderr (oder Datei, `logger`), Ergebnisse nach stdout. Mindestlevel einstellbar, Farben nur im Terminal. Nie Geheimnisse loggen. Rotation planen.

> **Praxisbeispiel**
>
> Ein Import-Skript schreibt pro Lauf eine Zeile mit Run-ID, Dateiname, Anzahl gelesener und abgelehnter Datensätze sowie Dauer. Zwei Wochen später fragt jemand, warum am 12. nur halb so viele Datensätze angekommen sind; die Antwort steht im Log, und niemand muss raten.

## 6.6 Temporäre Dateien

Skripte brauchen oft Zwischenablagen: eine sortierte Kopie, eine Teilausgabe, ein entpacktes Archiv, einen Arbeitsordner. Dafür gehört nie ein fester Name wie `/tmp/daten.txt` in ein Skript. Das hat gleich mehrere Probleme. Zwei gleichzeitig laufende Instanzen überschreiben sich gegenseitig (*Race Condition*, ein Fehler, dessen Ergebnis vom Zeitablauf abhängt). Eine Datei vom letzten Lauf wird versehentlich wiederverwendet. Und schließlich ein Sicherheitsproblem: In einem für alle beschreibbaren Verzeichnis wie `/tmp` kann ein Angreifer einen *Symlink* unter dem vorhersagbaren Namen anlegen, der auf eine wichtige Datei zeigt (*Symlink-Angriff*); schreibt ein privilegiertes Skript dann in die „temporäre“ Datei, überschreibt es die Zieldatei.

Die Lösung ist `mktemp`. Es erzeugt Datei oder Verzeichnis mit einem *eindeutigen, zufälligen Namen* und legt sie atomar an, sodass der Name nicht vorhersagbar und das Anlegen nicht manipulierbar ist; neue Dateien erhalten die Rechte `600`, Verzeichnisse `700`. `mktemp` legt eine Datei an, `mktemp -d` ein Verzeichnis, `mktemp -t vorlage.XXXXXX` nutzt eine Vorlage (die `X` werden ersetzt; mindestens drei, besser sechs), `--tmpdir` wählt den Ort, der Standard folgt der Variable `TMPDIR` und fällt auf `/tmp` zurück. Der zweite Teil der Lösung ist das *zuverlässige Aufräumen*: temporäre Dateien müssen bei *jedem* Ende verschwinden, auch bei Fehlern und Signalen. Das erledigt `trap 'rm -rf -- "$tmp"' EXIT` (Modul 4.7).

Für größere Daten gilt: `/tmp` liegt auf manchen Systemen im Arbeitsspeicher (`tmpfs`) und ist knapp; riesige Zwischendateien gehören nach `/var/tmp` oder in ein eigens konfiguriertes Verzeichnis. Wo möglich, vermeidet man Zwischendateien ganz: Pipes und Prozesssubstitution (`<(...)`) sparen sie ein. Für das *atomare Ersetzen* einer Zieldatei schreibt man zuerst in eine temporäre Datei *im selben Verzeichnis* (damit `mv` nur einen Eintrag umhängt und nicht kopiert) und benennt sie dann um.

```bash
#!/usr/bin/env bash
# tmp-demo.sh - sicher mit temporaeren Dateien arbeiten
set -euo pipefail

# --- 1. Arbeitsverzeichnis anlegen und IMMER aufraeumen
tmp="$(mktemp -d "${TMPDIR:-/tmp}/tmp-demo.XXXXXX")"
cleanup() { rm -rf -- "$tmp"; }
trap cleanup EXIT
echo "Arbeitsverzeichnis: $tmp"
ls -ld "$tmp" | cut -c1-10                      # drwx------

# --- 2. Datei mit Standardrechten 600
f="$(mktemp "$tmp/daten.XXXXXX")"
ls -l "$f" | cut -c1-10                         # -rw-------
printf 'c\na\nb\n' > "$f"

# --- 3. Pipeline-Zwischenergebnis: sortiert, dann weiterverarbeitet
sort "$f" > "$tmp/sortiert.txt"
paste -sd, "$tmp/sortiert.txt"

# --- 4. Zwischendatei ganz vermeiden (Prozesssubstitution)
diff <(sort "$f") <(sort -r "$f") >/dev/null || echo "Reihenfolgen unterscheiden sich"

# --- 5. Zieldatei atomar ersetzen
ziel="$tmp/konfig.txt"
echo "alt" > "$ziel"
neu="$(mktemp "$tmp/konfig.XXXXXX")"            # gleiches Verzeichnis wie das Ziel
echo "neu" > "$neu"
chmod 644 "$neu"
mv -f -- "$neu" "$ziel"                          # atomar: Leser sehen alt ODER neu
cat "$ziel"

# --- 6. Fehlerfall: auch beim Abbruch wird aufgeraeumt
trap 'cleanup; echo "aufgeraeumt (auch bei Fehler)" >&2' EXIT
ls "$tmp" | wc -l
false          # loest set -e aus; der EXIT-Trap raeumt trotzdem auf
```

Das Skript erzeugt mit `mktemp -d` ein Verzeichnis mit zufälligem Namensbestandteil und legt gleich danach die Aufräumfunktion als `EXIT`-Trap fest. Wichtig ist die Reihenfolge: Der Trap wird sofort nach dem Anlegen gesetzt, damit keine Lücke entsteht. Die Rechte sind restriktiv, wie die Ausgabe von `ls -l` zeigt. Alle weiteren temporären Dateien liegen *in diesem Verzeichnis*, sodass ein einziger `rm -rf` genügt. Block 4 zeigt, wie Prozesssubstitution Zwischendateien erspart. Block 5 ersetzt eine Datei atomar: Weil die neue Datei im selben Verzeichnis liegt, ist `mv` ein einfaches Umbenennen; ein gleichzeitig lesender Prozess sieht entweder die alte oder die neue Datei, nie eine halb geschriebene. Beachten Sie, dass `mktemp` die Datei mit Rechten `600` anlegt; wenn die Zieldatei andere Rechte braucht, setzt man sie vor dem `mv` (hier `chmod 644`). Im letzten Block bricht `false` das Skript ab, und der Trap räumt dennoch auf.

Typische Fehler: Erstens feste Dateinamen oder `$$` als „eindeutige“ Endung (`/tmp/datei.$$`): Die PID ist erratbar und wird wiederverwendet; `mktemp` ist sicher. Zweitens fehlt der `EXIT`-Trap, sodass nach einem Absturz oder Abbruch Reste liegen bleiben; auf Servern füllen sich so `/tmp`-Verzeichnisse. Drittens traps nur für `SIGINT`: Das normale Ende und Fehler bleiben ohne Aufräumen. Viertens löscht man mit `rm -rf "$tmp"` bei leerer Variable zu viel; deshalb den Wert *vor* dem Setzen des Traps zuweisen und im Trap mit `--` und gegebenenfalls `${tmp:?}` absichern. Fünftens: Geheimnisse in temporären Dateien (entschlüsselte Daten, Token). Sie sollten `600` haben, möglichst nur kurz existieren und werden von `rm` nicht „sicher“ gelöscht.

```bash
# falsch: vorhersagbarer Name, kein Aufraeumen, unsichere Rechte
tmp=/tmp/mein-skript.$$
echo "geheim" > "$tmp"

# richtig: mktemp, Trap sofort danach, Rechte 600
tmp="$(mktemp)"
trap 'rm -f -- "$tmp"' EXIT
echo "geheim" > "$tmp"
```

> **Merke**
>
> Temporäres immer mit `mktemp` anlegen und mit `trap ... EXIT` entfernen. Namen niemals fest oder aus der PID bilden. Ein Arbeitsverzeichnis statt vieler Einzeldateien. Zieldateien atomar ersetzen: temporäre Datei im selben Verzeichnis schreiben, dann `mv`.

> **Praxisbeispiel**
>
> Ein Skript, das jeden Abend einen Export in die Datei `/var/www/export.csv` schreibt, hat zu Randzeiten halb fertige Dateien ausgeliefert, weil der Webserver mitten im Schreiben las. Seit es erst in eine `mktemp`-Datei im selben Verzeichnis schreibt und dann per `mv` umbenennt, sehen Besucher immer eine vollständige Datei.

## 6.7 Idempotenz

Ein Vorgang ist *idempotent*, wenn mehrfaches Ausführen dasselbe Ergebnis hat wie einmaliges. `mkdir -p /srv/app` ist idempotent: Existiert das Verzeichnis, geschieht nichts, und es gibt keinen Fehler. `echo "zeile" >> datei` ist es nicht: Jeder Aufruf hängt eine weitere Zeile an. Für Automation ist Idempotenz eine der wichtigsten Eigenschaften, denn Skripte werden wiederholt: Ein Cron-Job läuft täglich, eine Pipeline wird neu gestartet, ein Administrator führt das Skript nach einem Fehlschlag noch einmal aus, ein Provisioning-Skript wird bei jedem Deployment erneut angewendet. Ein idempotentes Skript lässt sich ohne Angst wiederholen und konvergiert gegen den *Zielzustand*, unabhängig vom Ausgangszustand.

Das Prinzip lautet: *beschreibe den gewünschten Zustand, nicht die Schritte*. Statt „lege Benutzer an“ schreibt man „stelle sicher, dass der Benutzer existiert“: erst prüfen, dann nur bei Abweichung handeln. Dazu gehören *Vorbedingungen* (ist die Aufgabe schon erledigt?), *Zustandsprüfung* (stimmt der Ist-Zustand mit dem Soll überein?) und *sichere Wiederholbarkeit* nach Teilfehlern (ein abgebrochener Lauf darf keinen kaputten Zwischenzustand hinterlassen, den der nächste Lauf nicht reparieren kann). Hilfsmittel: `mkdir -p`, `rm -f`, `ln -sfn` (Symlink ersetzen), `cp -u` (nur neuere), `rsync` (gleicht an), `grep -qxF zeile datei || echo zeile >> datei` (Zeile nur ergänzen, wenn sie fehlt), `id -u name` und `getent` (Existenz prüfen), Marker-Dateien (`touch .done`) und Sperrdateien oder `flock` gegen gleichzeitige Läufe. Konfigurationsmanagement-Werkzeuge wie Ansible oder Puppet bauen das Prinzip systematisch aus; Bash-Skripte können es im Kleinen leisten.

```bash
#!/usr/bin/env bash
# idempotent-demo.sh - "stelle sicher, dass ..." statt "tue ..."
set -euo pipefail
basis="$(mktemp -d)"
trap 'rm -rf -- "$basis"' EXIT
cd "$basis"

# --- Hilfsfunktionen: pruefen, dann nur bei Abweichung handeln
stelle_verzeichnis_sicher() {          # Verzeichnis existiert mit Rechten MODE
    local pfad="$1" mode="$2"
    if [[ ! -d $pfad ]]; then mkdir -p -- "$pfad"; echo "angelegt: $pfad"; fi
    if [[ "$(stat -c '%a' "$pfad")" != "$mode" ]]; then
        chmod "$mode" "$pfad"; echo "Rechte korrigiert: $pfad -> $mode"
    fi
}

stelle_zeile_sicher() {                # Zeile steht genau einmal in Datei
    local zeile="$1" datei="$2"
    touch -- "$datei"
    if ! grep -qxF -- "$zeile" "$datei"; then
        printf '%s\n' "$zeile" >> "$datei"; echo "ergaenzt: $zeile"
    fi
}

stelle_link_sicher() {                 # Symlink LINK zeigt auf ZIEL
    local ziel="$1" link="$2"
    if [[ "$(readlink -- "$link" 2>/dev/null || true)" != "$ziel" ]]; then
        ln -sfn -- "$ziel" "$link"; echo "Link gesetzt: $link -> $ziel"
    fi
}

lauf() {
    echo "--- Lauf $1"
    stelle_verzeichnis_sicher app/daten 750
    stelle_zeile_sicher "port=8080"  app/app.conf
    stelle_zeile_sicher "debug=false" app/app.conf
    stelle_link_sicher release-2 app/current
}

lauf 1          # erster Lauf: macht alles
lauf 2          # zweiter Lauf: soll NICHTS mehr aendern
echo "--- Ergebnis:"; cat app/app.conf; ls -l app/current | awk '{print $9, $10, $11}'

# --- ein nicht-idempotentes Gegenstueck zum Vergleich
echo "--- naiv:"
for i in 1 2; do echo "port=8080" >> naiv.conf; done
wc -l < naiv.conf                       # 2 Zeilen: doppelt!

# --- Marker-Datei fuer Einmal-Schritte
if [[ ! -e .init-done ]]; then
    echo "Initialisierung laeuft"; touch .init-done
else
    echo "Initialisierung bereits erledigt"
fi
```

Die drei Hilfsfunktionen folgen demselben Muster: *Zustand prüfen, nur bei Abweichung handeln, Änderung melden*. Beim zweiten Lauf erscheint keine einzige Meldung, was zugleich der Test auf Idempotenz ist: „Läuft das Skript ein zweites Mal ohne Änderungen durch?“ `grep -qxF` prüft, ob die Zeile als *ganze Zeile* (`-x`) und *wörtlich* (`-F`) vorkommt. Der Symlink wird mit `readlink` auf sein Ziel geprüft und nur dann mit `ln -sfn` neu gesetzt; `-n` verhindert, dass ein vorhandener Link auf ein Verzeichnis *in* dieses Verzeichnis verlinkt wird. Der naive Gegenentwurf zeigt den Unterschied: doppelte Zeilen nach jedem Aufruf. Der Marker `.init-done` ist ein einfaches Mittel für Einmalschritte; er sollte *nach* dem erfolgreichen Abschluss gesetzt werden, nicht davor.

Typische Fehler: Erstens hängt man an Konfigurationsdateien an (`>>`), statt Zustand herzustellen. Zweitens überspringt man Schritte pauschal, wenn „irgendetwas“ existiert, ohne den Inhalt zu prüfen (ein halbfertiges Verzeichnis nach einem Abbruch gilt als „fertig“). Drittens setzt man den Marker zu früh. Viertens vergisst man, dass *Löschen* ebenfalls idempotent sein muss (`rm -f` statt `rm`, das beim zweiten Lauf einen Fehler meldet). Fünftens verändert man sich überschneidende Dinge wie Zähler oder Zeitstempel („Backup-Nummer um eins erhöhen“) und verliert Idempotenz; hier braucht man eine bewusste Entscheidung, was bei Wiederholung gelten soll. Sechstens laufen zwei Instanzen gleichzeitig und stören sich; `flock` schützt davor: `exec 9>/var/lock/skript.lock; flock -n 9 || exit 0`.

```bash
# falsch: bei Wiederholung doppelte Eintraege und Fehler
echo "export PATH=\$PATH:/opt/tool/bin" >> ~/.profile
useradd deploy

# richtig: Zustand pruefen, dann handeln
zeile='export PATH=$PATH:/opt/tool/bin'
grep -qxF -- "$zeile" ~/.profile || printf '%s\n' "$zeile" >> ~/.profile
id -u deploy >/dev/null 2>&1 || useradd deploy
```

> **Merke**
>
> Idempotent = beliebig oft ausführbar mit gleichem Ergebnis. Zielzustand beschreiben, Ist-Zustand prüfen, nur bei Abweichung handeln. Test: zweiter Lauf ändert nichts. Hilfsmittel: `mkdir -p`, `rm -f`, `ln -sfn`, `grep -qxF ... || echo ... >>`, Marker-Dateien, `flock`.

> **Praxisbeispiel**
>
> Ein Provisioning-Skript richtet neue Server ein: Benutzer, Verzeichnisse, Konfiguration, Dienste. Weil es idempotent ist, lässt es sich nach einem abgebrochenen Lauf einfach neu starten und läuft auch gegen bereits eingerichtete Server, um Abweichungen zu korrigieren (*Drift*), statt dass man es nur ein einziges Mal verwenden darf.

## 6.8 Modularisierung

Wächst ein Skript über wenige Dutzend Zeilen hinaus, wird es unübersichtlich. Modularisierung teilt es in überschaubare Teile mit klaren Aufgaben. Die Bausteine kennen Sie bereits: *Funktionen* (Modul 3.8) kapseln einzelne Aufgaben. Das Prinzip „Eine Funktion tut eine Sache“ macht Code lesbar und testbar. Darüber stehen *Bibliotheksdateien*: Dateien, die nur Funktionen und Konstanten definieren und von mehreren Skripten mit `source` (oder `.`) eingebunden werden. So schreibt man das Logging aus 6.5 einmal und verwendet es überall.

Eine Bibliothek soll beim Einbinden *nichts tun*, außer Definitionen anzulegen. Sie hat keinen eigenen Hauptablauf, kein `set -e` im globalen Namensraum (das würde das aufrufende Skript verändern) und vermeidet Ausgaben. Sie wird idealerweise gegen mehrfaches Einbinden geschützt (*Include Guard*). Für die Namen gilt: Um Kollisionen zu vermeiden, bekommen Bibliotheksfunktionen einen Präfix (`log_info`, `fs_ensure_dir`), und interne Hilfsfunktionen beginnen mit einem Unterstrich. Bash kennt keine Namensräume, daher ist die Namenskonvention die einzige Absicherung.

Beim Einbinden ist der *Pfad* die Hauptfehlerquelle: `source lib/log.sh` funktioniert nur, wenn das Skript aus dem passenden Verzeichnis gestartet wird. Robust ist der Pfad relativ zum Skript selbst (siehe Modul 2.1): `source "$SCRIPT_DIR/lib/log.sh"`. Der *Hauptablauf* eines Skripts gehört in eine Funktion `main`, die am Ende mit `main "$@"` aufgerufen wird; so steht alles Definierende oben, und das Skript lässt sich auch einbinden, ohne zu laufen: `[[ ${BASH_SOURCE[0]} == "$0" ]] && main "$@"`.

```bash
#!/usr/bin/env bash
# modul-demo.sh - legt eine kleine Projektstruktur an und nutzt eine Bibliothek
set -euo pipefail
basis="$(mktemp -d)"
trap 'rm -rf -- "$basis"' EXIT
mkdir -p "$basis/lib"

# --- lib/log.sh: Bibliothek (nur Definitionen, mit Include Guard)
cat > "$basis/lib/log.sh" <<'ENDE'
# lib/log.sh - einfache Logging-Funktionen. Wird gesourct, nicht ausgefuehrt.
[[ -n ${_LIB_LOG_LOADED:-} ]] && return 0
_LIB_LOG_LOADED=1

log_info()  { printf '[INFO]  %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }
log_die()   { log_error "$*"; exit 1; }
ENDE

# --- lib/fs.sh: zweite Bibliothek, die die erste benoetigt
cat > "$basis/lib/fs.sh" <<'ENDE'
# lib/fs.sh - Dateisystem-Helfer
_LIB_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$_LIB_DIR/log.sh"

fs_ensure_dir() {
    [[ -d $1 ]] || { mkdir -p -- "$1"; log_info "Verzeichnis angelegt: $1"; }
}
fs_require_file() {
    [[ -f $1 ]] || log_die "Datei fehlt: $1"
}
ENDE

# --- das eigentliche Hauptskript
cat > "$basis/hauptskript.sh" <<'ENDE'
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/log.sh"
source "$SCRIPT_DIR/lib/fs.sh"          # zweites Einbinden von log.sh ist harmlos

usage() { echo "Aufruf: ${0##*/} ZIELVERZEICHNIS" >&2; exit 2; }

main() {
    (( $# == 1 )) || usage
    log_info "Starte Verarbeitung"
    fs_ensure_dir "$1/daten"
    fs_require_file "$1/daten/eingabe.txt"     # fehlt -> sauberer Abbruch
    log_info "fertig"
}

# nur ausfuehren, wenn direkt gestartet (nicht beim Sourcen)
[[ ${BASH_SOURCE[0]} == "$0" ]] && main "$@"
ENDE
chmod +x "$basis/hauptskript.sh"

echo "--- Aufruf aus einem ganz anderen Verzeichnis:"
cd /
"$basis/hauptskript.sh" "$basis/ziel" || echo "(Skript endete mit Status $?)"

echo "--- Bibliothek gesourct: laeuft nichts, Funktionen stehen bereit"
source "$basis/lib/log.sh"
log_info "Hallo aus der Bibliothek"
```

Das Beispiel erzeugt eine kleine Projektstruktur und zeigt die Prinzipien. Die erste Bibliothek hat einen Include Guard: Beim zweiten Einbinden steht die Variable `_LIB_LOG_LOADED` schon, und `return 0` beendet das Einbinden sofort (`return` ist in einer gesourcten Datei erlaubt). Die zweite Bibliothek ermittelt *ihren eigenen Ort* über `BASH_SOURCE[0]` und bindet die erste relativ dazu ein; so funktionieren die Abhängigkeiten, egal woher man das Hauptskript startet, wie der Aufruf aus dem Wurzelverzeichnis beweist. Das Hauptskript ruft `main` nur auf, wenn es direkt gestartet wurde; beim Sourcen passiert nichts, was Tests erleichtert. Die Funktionen tragen Präfixe (`log_`, `fs_`). Die Fehlerbehandlung folgt dem Prinzip „kein stilles Scheitern“: `fs_require_file` beendet sauber mit Meldung. Hier endet das Skript im Beispiel gewollt mit Status 1, weil die Eingabedatei fehlt.

Typische Fehler: Erstens relative `source`-Pfade, die vom Aufrufort abhängen. Zweitens Bibliotheken, die beim Einbinden etwas ausgeben oder ausführen. Drittens globale Variablen in Funktionen ohne `local`, die zwischen Bibliotheken kollidieren. Viertens gibt es keine Konvention für Rückgabewerte, jede Funktion macht es anders. Fünftens: `set -e` oder `exit` in Bibliotheken verändern das Verhalten des aufrufenden Skripts; `exit` in einer gesourcten Funktion beendet das *ganze* Skript, was bei `log_die` gewollt, bei Hilfsfunktionen aber überraschend sein kann. Sechstens führt eine Kette von `source`-Aufrufen ohne Guard zu Endlosrekursion, wenn sich zwei Dateien gegenseitig einbinden. Siebtens wächst die Zahl der Skripte, aber niemand weiß, welche Version der Bibliothek verwendet wird; kopierte Bibliotheken driften auseinander. Besser ist ein gemeinsames Repository und ein festgelegter Ort.

```bash
# falsch: haengt vom Aufrufort ab, fuehrt beim Einbinden Code aus
source lib/log.sh
main "$@"

# richtig: Pfad relativ zum Skript, main nur bei direktem Start
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/log.sh"
[[ ${BASH_SOURCE[0]} == "$0" ]] && main "$@"
```

> **Merke**
>
> Aufteilen in Funktionen und Bibliotheksdateien. Bibliotheken enthalten nur Definitionen, haben Präfixe und Include Guards, und werden relativ zum Skriptort eingebunden. Hauptablauf in `main`, aufgerufen nur bei direktem Start. Globale Zustände und `exit` in Bibliotheken bewusst einsetzen.

> **Praxisbeispiel**
>
> Ein Team pflegt ein Repository `shell-lib` mit `log.sh`, `retry.sh`, `checks.sh` und `args.sh`. Jedes neue Skript bindet die Bibliothek ein und hat in Minuten einheitliche Logs, Wiederholungslogik und Prüfungen. Fehlerkorrekturen an einer Stelle kommen allen Skripten zugute; dafür sind Tests und Versionierung der Bibliothek unverzichtbar (siehe Modul 12).

## 6.9 Dokumentation

Ein Skript wird einmal geschrieben und hundertmal gelesen: von Kollegen, vom Betrieb in der Nacht, von Ihnen selbst nach einem halben Jahr. Dokumentation entscheidet darüber, ob sich ein Skript sicher ändern und betreiben lässt. Sie hat drei Ebenen. Erstens *Kommentare im Code*, die das *Warum* erklären, nicht das *Was*: Dass `i=$((i+1))` den Zähler erhöht, sieht man; warum man vor dem Löschen drei Sekunden wartet, nicht. Zweitens eine *Kopfzeilen-Beschreibung* in jedem Skript: Zweck, Aufruf, Parameter, benötigte Umgebung und Rechte, Exit-Codes, Autor und Änderungshinweis. Drittens ein *README* im Projekt: Überblick, Installation, Beispiele, Abhängigkeiten, Betriebshinweise, bekannte Einschränkungen.

Ein besonders nützliches Muster ist die *selbstdokumentierende Hilfe*: Der `usage`-Text (Modul 3.9) wird nicht doppelt gepflegt, sondern direkt aus den Kopfkommentaren der Datei erzeugt. Für den Betrieb sind *Runbooks* (Modul 9.8) wichtig: Was tun, wenn das Skript fehlschlägt? Welche Exit-Codes bedeuten was? Wie startet man es manuell neu? Wo liegen die Logs? Wer ist zuständig? Dokumentation, die nicht aktuell ist, ist schlimmer als keine; sie lebt deshalb am besten *beim Code* (im Repository, im selben Review) und wird bei Änderungen mit angepasst.

```bash
#!/usr/bin/env bash
#
# backup-db.sh - Sichert die Datenbank in ein Archiv und rotiert alte Sicherungen.
#
# Aufruf:   backup-db.sh [-n] [-k ANZAHL] ZIELVERZEICHNIS
# Optionen: -n          Trockenlauf, nichts wird veraendert
#           -k ANZAHL   so viele Sicherungen behalten (Standard: 7)
#           -h          diese Hilfe
# Umgebung: DB_NAME     Name der Datenbank (Pflicht)
#           DB_PASS_FILE Datei mit dem Passwort (Rechte 600, Pflicht)
# Exit:     0 Erfolg, 2 falsche Benutzung, 4 Datenbank nicht erreichbar, 5 Schreibfehler
# Abhaengigkeiten: bash >= 4.3, tar, gzip
# Autor:    Team Betrieb. Aenderungen siehe git log.
set -euo pipefail

usage() {                                  # Hilfe direkt aus dem Kopfkommentar erzeugen
    sed -n '2,/^set -euo/{/^set -euo/d;s/^# \{0,1\}//;p}' "$0"
}

[[ ${1:-} == -h || ${1:-} == --help ]] && { usage; exit 0; }

# Warum wir 'nice' verwenden: Das Backup laeuft auf dem Produktivsystem und
# darf Anwendungen nicht ausbremsen (siehe Vorfall INC-1234).
echo "Weiter mit der eigentlichen Arbeit ..."
```

Der Kopfkommentar enthält alles Wesentliche: Zweck, Aufruf, Optionen, benötigte Umgebung, Exit-Codes und Abhängigkeiten. Die Funktion `usage` liest ihn mit `sed` wieder ein: Sie gibt alle Zeilen von der zweiten bis zur Zeile mit `set -euo` aus und entfernt das führende `# `. So gibt es genau eine Stelle, an der die Beschreibung gepflegt wird, und `backup-db.sh -h` zeigt immer den aktuellen Stand. Der Kommentar vor `echo` erklärt das *Warum* einer Entscheidung und verweist auf einen Vorfall; so ist der Hintergrund auch später noch nachvollziehbar.

Typische Fehler: Erstens kommentiert man das Offensichtliche (`# erhoeht i um 1`) und lässt das Wichtige weg. Zweitens veralten Kommentare, weil sie nicht mitgepflegt werden; ein falscher Kommentar ist gefährlicher als keiner. Drittens fehlt jede Angabe zu Exit-Codes, sodass der Betrieb nicht weiß, was ein Fehlercode bedeutet. Viertens steckt das Wissen in den Köpfen einzelner Personen, und niemand außer dem Autor kann das Skript betreiben (*Bus-Faktor*). Fünftens enthalten Kommentare und READMEs Geheimnisse (Passwörter, interne URLs mit Tokens). Sechstens werden Beispiele nicht getestet und funktionieren nicht; Beispiele in der Doku sollten kopierbar und lauffähig sein. Gute Ergänzungen sind ein Änderungsprotokoll (`CHANGELOG`) und eine klare Versionierung (siehe Modul 12.7).

```bash
# falsch: beschreibt das Offensichtliche, laesst das Wichtige weg
# Schleife ueber alle Dateien
for f in *.log; do
    gzip "$f"      # komprimiert
done

# richtig: erklaert die Absicht und die Randbedingung
# Logs ueber 30 Tage werden komprimiert, nicht geloescht: Das Audit verlangt
# 12 Monate Aufbewahrung (siehe Betriebshandbuch, Kap. 4).
find "$LOG_DIR" -name '*.log' -mtime +30 -exec gzip -- {} +
```

> **Merke**
>
> Kommentare erklären das *Warum*. Jedes Skript hat einen Kopf mit Zweck, Aufruf, Umgebung, Exit-Codes und Abhängigkeiten; die Hilfe (`-h`) kommt aus derselben Quelle. Ein README und Runbook gehören ins Repository und werden mit dem Code geändert. Keine Geheimnisse in der Doku.

> **Praxisbeispiel**
>
> Ein Betriebsteam übernimmt das Skript eines ausgeschiedenen Kollegen. Dank Kopfkommentar, Exit-Code-Tabelle und README mit Beispielen und Störungsbehandlung ist der Betrieb am ersten Tag sichergestellt. Ohne Dokumentation hätte man das Skript neu geschrieben oder nicht mehr angefasst.

## 6.10 Übungsaufgaben zu Modul 06

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Nennen Sie fünf Situationen, in denen `set -e` einen fehlgeschlagenen Befehl *nicht* zum Abbruch führen lässt. Was ändert `set -o pipefail`, und welche Wirkung hat `set -u`?

**Aufgabe 2.** Warum ist `local datei="$(mktemp)"` problematisch, und wie schreibt man es richtig? Was unterscheidet `ls "$tmp"` im Trap bei leerer Variable von `rm -rf -- "${tmp:?}"`?

**Aufgabe 3.** Erklären Sie, warum `/tmp/skript.$$` als temporärer Dateiname unsicher ist, und nennen Sie drei Vorteile von `mktemp`.

**Aufgabe 4.** Was bedeutet Idempotenz? Nennen Sie je zwei idempotente und nicht idempotente Shell-Befehle und erklären Sie, wie man `echo "zeile" >> datei` idempotent macht.

**Aufgabe 5.** Schreiben Sie ein Skript `kopiere-sicher.sh QUELLE ZIEL`, das mit `set -euo pipefail` arbeitet, Argumente prüft, in eine temporäre Datei im Zielverzeichnis kopiert, deren SHA-256-Summe mit der der Quelle vergleicht und erst dann per `mv` atomar an den Zielort verschiebt. Bei jedem Fehler wird die temporäre Datei entfernt und mit passendem Exit-Code (2 falsche Benutzung, 5 Prüfsumme/Schreibfehler) beendet.

**Aufgabe 6.** Schreiben Sie eine Bibliothek `lib/log.sh` mit `log_debug`, `log_info`, `log_warn`, `log_error` (Zeitstempel, Level, Mindestlevel über `LOG_LEVEL`) und ein Hauptskript, das sie relativ zum eigenen Ort einbindet. Die Bibliothek bekommt einen Include Guard.

**Aufgabe 7.** Schreiben Sie ein idempotentes Skript `einrichten.sh`, das ein Verzeichnis `app/{bin,conf,logs}` anlegt, eine Konfigurationsdatei mit den Zeilen `port=8080` und `mode=prod` genau einmal sicherstellt, einen Link `app/current` auf `app/bin` setzt und bei jedem Schritt nur meldet, was tatsächlich geändert wurde. Ein zweiter Lauf darf nichts ausgeben außer einer Zusammenfassung „0 Änderungen“.

**Aufgabe 8.** Schreiben Sie ein Skript `retry-job.sh`, das einen beliebigen Befehl bis zu N Mal (Option `-n`, Standard 3) mit exponentieller Wartezeit (1, 2, 4 Sekunden) wiederholt, jeden Versuch mit Zeitstempel loggt, eine Sperre gegen gleichzeitige Läufe per `flock` setzt und mit dem Exit-Code des letzten Versuchs endet. Ein `ERR`-Trap ist nicht nötig, aber ein `EXIT`-Trap räumt die Sperre auf.

## 6.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** `set -e` greift unter anderem nicht (1) in Bedingungen von `if`, `while` und `until`, (2) bei Befehlen links von `&&` oder `||`, (3) bei allen Stufen einer Pipeline außer der letzten, (4) in Funktionen, die als Bedingung aufgerufen werden, (5) in Befehlsersetzungen (ohne `inherit_errexit`), außerdem bei `local x=$(...)`, bei Befehlen mit vorangestelltem `!` und bei Process Substitution. `set -o pipefail` macht den Exit-Code einer Pipeline zum letzten von null verschiedenen Code aller Stufen, sodass auch Fehler in frühen Stufen sichtbar werden. `set -u` behandelt die Verwendung nicht gesetzter Variablen als Fehler; optionale Variablen liest man dann mit `${var:-}`.

**Lösung zu Aufgabe 2.** Bei `local datei="$(mktemp)"` ist der Exit-Code der Zeile der des Builtins `local` (immer 0); ein Fehlschlag von `mktemp` bleibt dadurch unbemerkt und `set -e` löst nicht aus. Richtig: `local datei; datei="$(mktemp)"`, dann gilt der Status der Zuweisung. `rm -rf -- "${tmp:?}"` bricht mit Fehlermeldung ab, wenn `tmp` leer oder nicht gesetzt ist, und verhindert so, dass `rm -rf -- ""` oder (bei fehlenden Quotes) ein zu weit gefasster Pfad zum Problem wird. `ls "$tmp"` bei leerer Variable würde dagegen das aktuelle Verzeichnis anzeigen, was nur ein harmloser, aber irreführender Effekt wäre, während ein fehlerhaftes `rm` Daten vernichtet.

**Lösung zu Aufgabe 3.** Die PID ist erratbar und wird nach dem Ende des Prozesses wiederverwendet; ein Angreifer kann unter dem vorhersagbaren Namen einen Symlink anlegen, der auf eine wichtige Datei zeigt, und das privilegierte Skript überschreibt diese (Symlink-Angriff). Außerdem können zwei Läufe mit gleicher PID (nach Wiederverwendung) oder Reste eines alten Laufs kollidieren. `mktemp` legt (1) einen zufälligen, unvorhersagbaren Namen an, (2) die Datei atomar mit exklusivem Anlegen, sodass kein bestehendes Objekt übernommen wird, und (3) mit sicheren Rechten (600 beziehungsweise 700). Zusätzlich respektiert es `TMPDIR`.

**Lösung zu Aufgabe 4.** Ein Vorgang ist idempotent, wenn mehrfaches Ausführen dasselbe Ergebnis hat wie einmaliges. Idempotent: `mkdir -p verz`, `rm -f datei`, `ln -sfn ziel link`, `chmod 640 datei`. Nicht idempotent: `echo x >> datei`, `useradd name` (Fehler beim zweiten Mal), `mkdir verz` (Fehler), `zaehler=$((zaehler+1))`. Idempotent machen: `grep -qxF -- "zeile" datei || echo "zeile" >> datei`, das nur anhängt, wenn die Zeile als ganze Zeile nicht vorhanden ist.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# kopiere-sicher.sh QUELLE ZIEL
set -euo pipefail
tmp=""
cleanup() { [[ -z $tmp ]] || rm -f -- "$tmp"; }
trap cleanup EXIT

fehler() { local code=$1; shift; echo "FEHLER: $*" >&2; exit "$code"; }

(( $# == 2 )) || fehler 2 "Aufruf: $0 QUELLE ZIEL"
quelle="$1"; ziel="$2"
[[ -f $quelle ]]               || fehler 2 "Quelle ist keine Datei: $quelle"
[[ -d $(dirname -- "$ziel") ]] || fehler 2 "Zielverzeichnis fehlt: $(dirname -- "$ziel")"

tmp="$(mktemp "$(dirname -- "$ziel")/.kopie.XXXXXX")" || fehler 5 "tmp-Datei nicht anlegbar"
cp -- "$quelle" "$tmp"                                      || fehler 5 "Kopieren fehlgeschlagen"

summe_q="$(sha256sum < "$quelle")"
summe_z="$(sha256sum < "$tmp")"
[[ $summe_q == "$summe_z" ]] || fehler 5 "Pruefsummen unterscheiden sich"

chmod --reference="$quelle" "$tmp"
mv -f -- "$tmp" "$ziel"
tmp=""                                                      # nichts mehr aufzuraeumen
echo "OK: $quelle -> $ziel"
```

Die temporäre Datei liegt im Zielverzeichnis, damit `mv` ein atomares Umbenennen ist. Der `EXIT`-Trap löscht sie bei jedem Fehler, `tmp=""` nach dem erfolgreichen `mv` verhindert, dass der Trap eine nicht mehr vorhandene Datei bearbeitet. Die Prüfsumme wird über `sha256sum < datei` berechnet, damit der Dateiname nicht in die Ausgabe eingeht und beide Werte vergleichbar sind. Die Rechte der Quelle werden mit `chmod --reference` (GNU) übernommen.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# lib/log.sh - Logging-Bibliothek (wird gesourct)
[[ -n ${_LIB_LOG_LOADED:-} ]] && return 0
_LIB_LOG_LOADED=1

declare -A _LOG_LVL=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40)
LOG_LEVEL="${LOG_LEVEL:-INFO}"

_log() {
    local level="$1"; shift
    (( ${_LOG_LVL[$level]} >= ${_LOG_LVL[${LOG_LEVEL}]:-20} )) || return 0
    printf '%s %-5s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$level" "$*" >&2
}
log_debug() { _log DEBUG "$@"; }
log_info()  { _log INFO  "$@"; }
log_warn()  { _log WARN  "$@"; }
log_error() { _log ERROR "$@"; }
```

```bash
#!/usr/bin/env bash
# haupt.sh - nutzt die Bibliothek unabhaengig vom Aufrufort
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/log.sh"

main() {
    log_info "Start"
    log_debug "nur bei LOG_LEVEL=DEBUG sichtbar"
    log_warn "Achtung"
    log_error "Fehler"
}
[[ ${BASH_SOURCE[0]} == "$0" ]] && main "$@"
```

Die Bibliothek enthält nur Definitionen, einen Include Guard und eine interne Funktion `_log`, die den Level gegen das Mindestlevel prüft; ein unbekanntes Mindestlevel fällt auf 20 (INFO) zurück. Das Hauptskript ermittelt seinen Ort und bindet die Bibliothek relativ dazu ein; `main` läuft nur bei direktem Start.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# einrichten.sh - idempotent
set -euo pipefail
aenderungen=0
melde() { echo "geaendert: $*"; aenderungen=$((aenderungen + 1)); }

for d in app/bin app/conf app/logs; do
    [[ -d $d ]] || { mkdir -p -- "$d"; melde "Verzeichnis $d"; }
done

konf=app/conf/app.conf
[[ -f $konf ]] || { : > "$konf"; melde "Datei $konf"; }
for zeile in "port=8080" "mode=prod"; do
    grep -qxF -- "$zeile" "$konf" || { printf '%s\n' "$zeile" >> "$konf"; melde "Zeile $zeile"; }
done

if [[ "$(readlink -- app/current 2>/dev/null || true)" != "bin" ]]; then
    ln -sfn bin app/current
    melde "Link app/current -> bin"
fi

echo "$aenderungen Aenderungen"
```

Jeder Schritt prüft zuerst den Ist-Zustand und meldet nur, wenn er wirklich etwas ändert. Der Link zeigt relativ auf `bin`, also relativ zum Verzeichnis `app`, in dem er liegt; `readlink` liefert genau diesen Text. Beim zweiten Lauf entfällt jede Änderungsmeldung, es erscheint nur `0 Aenderungen`. Beachten Sie, dass der Zähler in der Hauptshell läuft und `melde` nicht in einer Pipeline oder Befehlsersetzung aufgerufen wird; sonst ginge der Zählerstand in einer Subshell verloren.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# retry-job.sh [-n VERSUCHE] BEFEHL [ARGUMENTE...]
set -uo pipefail
versuche=3
while getopts ':n:' opt; do
    case $opt in
        n) versuche="$OPTARG" ;;
        *) echo "Aufruf: $0 [-n VERSUCHE] BEFEHL..." >&2; exit 2 ;;
    esac
done
shift $((OPTIND - 1))
(( $# >= 1 )) || { echo "Aufruf: $0 [-n VERSUCHE] BEFEHL..." >&2; exit 2; }
[[ $versuche =~ ^[1-9][0-9]*$ ]] || { echo "-n erwartet eine positive Zahl" >&2; exit 2; }

sperre="${TMPDIR:-/tmp}/retry-job.$(id -u).lock"
exec 9> "$sperre"
flock -n 9 || { echo "Laeuft bereits" >&2; exit 3; }
trap 'rm -f -- "$sperre"' EXIT

log() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" >&2; }

pause=1
status=0
for (( i = 1; i <= versuche; i++ )); do
    log "Versuch $i/$versuche: $*"
    "$@"; status=$?
    if (( status == 0 )); then log "Erfolg"; exit 0; fi
    log "Status $status"
    (( i < versuche )) && { log "warte ${pause}s"; sleep "$pause"; pause=$((pause * 2)); }
done
log "Aufgegeben nach $versuche Versuchen"
exit "$status"
```

Das Skript verzichtet bewusst auf `set -e`, weil der Fehlschlag des Befehls erwartet wird und ausgewertet werden soll; `-u` und `pipefail` bleiben aktiv. Die Sperre nutzt `flock` auf Deskriptor 9: `-n` scheitert sofort, wenn schon eine Instanz läuft. Die Wartezeit verdoppelt sich nach jedem Fehlschlag (1, 2, 4 Sekunden). Der Exit-Code ist der des letzten Versuchs; eine eigene Sperrdatei-Entfernung im `EXIT`-Trap räumt auf. Beachten Sie den Hinweis zur Idempotenz: Wiederholt werden darf nur ein Befehl, der bei erneuter Ausführung keinen Schaden anrichtet.
