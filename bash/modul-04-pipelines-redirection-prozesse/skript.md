# Modul 04 - Pipelines, Redirection und Prozesssteuerung

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (4.1 bis 4.8; 4.9 Übungsaufgaben, 4.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 10 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (8 Einheiten à 1,25 h). Beide Angaben stimmen überein.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit              | Dauer (h) | Prio |
|:----|:-------------------------|----------:|:----:|
| 4.1 | Redirection              |      1,25 |  1   |
| 4.2 | Pipelines und Exit Codes |      1,25 |  1   |
| 4.3 | Command Substitution     |      1,25 |  1   |
| 4.4 | Prozessverwaltung        |      1,25 |  1   |
| 4.5 | Job Control              |      1,25 |  1   |
| 4.6 | Subshells und Grouping   |      1,25 |  1   |
| 4.7 | Trap und Signale         |      1,25 |  2   |
| 4.8 | Named Pipes und tee      |      1,25 |  2   |
|     | **Summe**                | **10,00** |      |

Dieses Modul vertieft, was in den Modulen 1 und 2 angedeutet wurde: wie Daten zwischen Prozessen fließen, wie Erfolg und Misserfolg gemeldet werden und wie die Shell Prozesse startet, überwacht und beendet. Wer diese Mechanik versteht, kann nicht nur robuste Skripte schreiben, sondern auch rätselhaftes Verhalten wie verschwundene Variablen, hängende Pipelines oder verwaiste Hintergrundprozesse erklären.

## 4.1 Redirection

Jeder Prozess arbeitet mit *Dateideskriptoren*, kleinen Ganzzahlen, die auf geöffnete Dateien, Geräte oder Datenströme verweisen. Drei sind immer vorbelegt: 0 (stdin), 1 (stdout) und 2 (stderr). *Redirection* bedeutet, dass die Shell vor dem Start eines Programms diese Zuordnungen ändert. Das Programm selbst merkt davon nichts; es schreibt weiter auf Deskriptor 1, der nun aber zufällig in einer Datei endet. Diese Trennung ist der Grund, warum sich jedes Unix-Programm ohne Änderung umleiten lässt.

Die grundlegenden Operatoren: `> datei` leitet stdout in eine Datei um und *überschreibt* sie (die Datei wird beim Öffnen auf Länge null gekürzt); `>> datei` hängt an; `< datei` liest stdin aus einer Datei; `2> datei` leitet stderr um; `&> datei` (Bash-spezifisch) leitet stdout und stderr gemeinsam um; `2>&1` macht Deskriptor 2 zu einer Kopie von Deskriptor 1. Allgemein bedeutet `n>&m`: „Deskriptor n zeigt künftig dorthin, wohin m *im Moment* zeigt“. Die Auswertung geschieht streng von links nach rechts, was die Reihenfolge entscheidend macht.

Darüber hinaus gibt es das *Here-Document* und den *Here-String*. Ein Here-Document (`<<ENDE` ... `ENDE`) übergibt mehrere Zeilen Text direkt im Skript als stdin an einen Befehl. Steht das Begrenzerwort in Anführungszeichen (`<<'ENDE'`), findet im Text keine Variablen- und Befehlsersetzung statt; sonst werden Variablen im Text ersetzt. Mit `<<-` werden führende Tabulatoren entfernt, was eingerückte Dokumente erlaubt. Der Here-String (`<<< "text"`) übergibt eine einzelne Zeichenkette. Für beliebige Deskriptoren gibt es `exec`: `exec 3> logdatei` öffnet Deskriptor 3 dauerhaft für die weitere Skriptlaufzeit, und `exec 3>&-` schließt ihn. Praktisch ist auch `exec > datei 2>&1` am Skriptanfang, um die gesamte Ausgabe des Skripts umzuleiten.

```bash
#!/usr/bin/env bash
# redirect-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1

echo "--- 1. Reihenfolge von 2>&1"
touch vorhanden
ls vorhanden gibtsnicht > richtig.txt 2>&1      # beides in die Datei
ls vorhanden gibtsnicht 2>&1 > falsch.txt       # Fehler bleiben auf dem Terminal
echo "richtig.txt hat $(wc -l < richtig.txt) Zeile(n), falsch.txt hat $(wc -l < falsch.txt)"

echo "--- 2. Here-Document mit und ohne Ersetzung"
name="Anna"
cat <<ENDE
Hallo $name, heute ist $(date +%F).
ENDE
cat <<'ENDE'
Hier bleibt $name woertlich und $(date) ebenso.
ENDE

echo "--- 3. Here-String"
tr 'a-z' 'A-Z' <<< "kleiner text"

echo "--- 4. Eigener Deskriptor fuer ein Log"
exec 3> log.txt
echo "Zeile fuer Deskriptor 3" >&3
echo "noch eine" >&3
exec 3>&-                                       # schliessen
cat log.txt

echo "--- 5. Ganzes Skript umleiten"
(
    exec > komplett.log 2>&1
    echo "stdout im Log"
    ls gibtsnicht
)
cat komplett.log

echo "--- 6. noclobber schuetzt vor Ueberschreiben"
set -o noclobber
echo "erste" > schutz.txt
echo "zweite" 2>/dev/null > schutz.txt || echo "Ueberschreiben verhindert"
echo "zweite" >| schutz.txt                     # >| erzwingt es bewusst
set +o noclobber
cat schutz.txt

cd / && rm -rf -- "$verz"
```

Im ersten Block zeigt sich die Wichtigkeit der Reihenfolge. Bei `> richtig.txt 2>&1` wird zuerst stdout in die Datei gelegt und dann stderr auf dasselbe Ziel, also ebenfalls in die Datei. Bei `2>&1 > falsch.txt` wird zuerst stderr auf das *aktuelle* Ziel von stdout (das Terminal) gelegt und *danach* stdout in die Datei; die Fehlermeldung erscheint daher auf dem Bildschirm. Beim Here-Document sieht man den Unterschied zwischen einem Begrenzer mit und ohne Anführungszeichen. Der eigene Deskriptor 3 ist praktisch, um Protokollmeldungen von der normalen Ausgabe zu trennen, ohne bei jedem Befehl einen Dateinamen zu wiederholen. Der Block mit der Subshell zeigt `exec` ohne Befehl: Es ändert die Umleitungen der laufenden Shell dauerhaft; hier in der Subshell, damit der Rest des Skripts unberührt bleibt. Die Option `noclobber` (*clobber* = überschreiben) verhindert versehentliches Überschreiben mit `>`; wer es dennoch will, schreibt `>|`.

Hier lauern die klassischen Fehler. Erstens das gleichzeitige Lesen und Schreiben derselben Datei (`sort d > d`, siehe Modul 2.6). Zweitens die Annahme, dass `2>&1` mit der Pipe *vor* der Umleitung funktioniert: In `befehl 2>&1 | grep x` landen beide Ströme in der Pipe (die Pipe-Verbindung wird zuerst hergestellt), in `befehl | grep x 2>&1` betrifft `2>&1` dagegen nur `grep`. Bash bietet für das Erste die Kurzform `|&`. Drittens vergisst man bei Here-Dokumenten, dass Variablen ohne Anführungszeichen im Begrenzer ersetzt werden; wer Skriptcode oder Konfiguration mit `$`-Zeichen ausgibt, muss `<<'ENDE'` verwenden. Viertens: Das Begrenzerwort muss am Zeilenanfang und ohne nachfolgende Leerzeichen stehen; sonst endet das Dokument nie, und die Shell meldet „here-document delimited by end-of-file“.

```bash
# falsch: Fehler gehen NICHT in die Pipe
make 2>&1 > build.log | grep Fehler      # Reihenfolge verdreht

# richtig: erst stdout in die Datei, dann stderr dorthin
make > build.log 2>&1

# richtig: stdout und stderr zusammen in die Pipe
make 2>&1 | grep Fehler
```

> **Merke**
>
> Umleitungen werden von links nach rechts ausgewertet. `> datei 2>&1` leitet beides um, `2>&1 > datei` nicht. Here-Documents mit `<<'ENDE'` unterdrücken Ersetzungen. `exec` ändert Deskriptoren der laufenden Shell; `noclobber` schützt vor Überschreiben.

> **Praxisbeispiel**
>
> Ein Cron-Job schreibt Meldungen nach `job.log`, Fehler zusätzlich nach `job.err`. Ein Installationsskript verwendet ein Here-Document, um eine Konfigurationsdatei zu erzeugen: `cat > /etc/app.conf <<'ENDE'` mit dem fertigen Text. Die Anführungszeichen stellen sicher, dass Variablen in der Konfiguration (etwa `$HOME`) nicht schon beim Erzeugen aufgelöst werden.

## 4.2 Pipelines und Exit Codes

Eine *Pipeline* verbindet mehrere Befehle mit dem Zeichen `|`, sodass stdout des einen zu stdin des nächsten wird. Technisch erzeugt die Shell für jeden Befehl einen eigenen Prozess und verbindet sie über *Pipes* des Kernels, kleine Puffer (unter Linux typischerweise 64 KiB), die der schreibende Prozess füllt und der lesende leert. Ist der Puffer voll, wird der Schreiber blockiert, ist er leer, wartet der Leser. So laufen alle Stufen gleichzeitig, und große Datenmengen lassen sich verarbeiten, ohne sie ganz im Speicher zu halten. Endet der Leser früher (zum Beispiel `head -n 1`), erhält der Schreiber beim nächsten Schreiben das Signal `SIGPIPE` und beendet sich ebenfalls; das ist erwünscht und spart Arbeit.

Jedes Programm meldet beim Ende einen *Exit-Code* (Rückgabewert, Statuscode) zwischen 0 und 255. 0 bedeutet Erfolg, alles andere einen Fehler; die genaue Bedeutung legt jedes Programm fest. Konventionen: 1 für allgemeine Fehler, 2 für falsche Benutzung, 126 für „nicht ausführbar“, 127 für „Befehl nicht gefunden“, 128+n für „durch Signal n beendet“ (etwa 130 nach Strg+C, Signal 2, und 137 nach `SIGKILL`, Signal 9). Die Shell stellt den Code des zuletzt beendeten Vordergrundbefehls in der Variable `$?` bereit. Diese wird bei *jedem* Befehl überschrieben, auch bei `echo`; wer den Wert braucht, sichert ihn sofort (`status=$?`).

Bei einer Pipeline ist der Exit-Code standardmäßig der des *letzten* Befehls. Fehler in früheren Stufen bleiben unsichtbar: `falschesprogramm | wc -l` meldet Erfolg, solange `wc` läuft. Dagegen hilft zweierlei. Das Array `PIPESTATUS` enthält die Exit-Codes *aller* Stufen der letzten Pipeline. Die Option `set -o pipefail` lässt den Code der Pipeline zum *letzten Fehlercode ungleich null* werden (oder 0, wenn alle Stufen erfolgreich waren). Zusammen mit `set -e` (Abbruch beim ersten Fehler) und `set -u` (Fehler bei ungesetzten Variablen) bildet es den Kern des sogenannten „Strict Mode“, der in Modul 6 genauer behandelt wird.

Zur Verknüpfung von Befehlen gibt es mehrere Operatoren. `a ; b` führt beide nacheinander aus, unabhängig vom Ergebnis. `a && b` führt `b` nur aus, wenn `a` erfolgreich war; `a || b` nur, wenn `a` fehlschlug. `!` vor einem Befehl kehrt seinen Status um. Diese Operatoren eignen sich für Kurzformen wie `mkdir x && cd x`, taugen aber nicht als Ersatz für `if`-`else` (siehe Modul 3.6).

```bash
#!/usr/bin/env bash
# pipeline-demo.sh

echo "--- 1. Standard: nur der letzte Status zaehlt"
false | true
echo "Status der Pipeline: $?"                    # 0, obwohl false fehlschlug

echo "--- 2. PIPESTATUS zeigt alle Stufen"
false | true | false
echo "Alle Status: ${PIPESTATUS[*]}"              # 1 0 1

echo "--- 3. pipefail"
set -o pipefail
false | true
echo "Status mit pipefail: $?"                    # 1
set +o pipefail

echo "--- 4. Status sofort sichern"
grep -q "root" /etc/passwd
status=$?
echo "grep meldet: $status"
case $status in
    0) echo "Treffer" ;;
    1) echo "kein Treffer (kein Fehler!)" ;;
    *) echo "echter Fehler" ;;
esac

echo "--- 5. Verknuepfungen"
true  && echo "und: laeuft"
false && echo "und: wird uebersprungen"
false || echo "oder: laeuft"
! false && echo "negiert: laeuft"

echo "--- 6. Typische Exit-Codes"
bash -c 'exit 3';            echo "exit 3        -> $?"
nichtvorhanden_xyz 2>/dev/null; echo "unbekannt     -> $?"
bash -c 'kill -TERM $$';     echo "SIGTERM       -> $?"
```

Der erste Block beweist, dass Fehler in frühen Stufen verloren gehen. Der zweite liest sie aus `PIPESTATUS`; das Array ist nur direkt nach der Pipeline gültig und wird beim nächsten Befehl überschrieben. Der dritte Block zeigt `pipefail` in Aktion. Der vierte demonstriert, dass `grep` bei „kein Treffer“ den Code 1 liefert, was kein Fehler im eigentlichen Sinn ist; `case` unterscheidet die Fälle. Der fünfte zeigt die Logik der Verknüpfungsoperatoren, der sechste typische Codes: 127 für ein unbekanntes Kommando und 143 (128 + 15) für ein Signal `SIGTERM`.

Fehler, die hier immer wieder auftreten: Erstens glaubt man, eine Pipeline sei fehlerfrei, weil das Skript „durchläuft“. Zweitens setzt man `set -e` ein, ohne zu wissen, dass es in Bedingungen, in Teilen von `&&`-Ketten und in Pipelines (ohne `pipefail`) *nicht* greift. Drittens führt `set -o pipefail` zusammen mit `SIGPIPE` zu überraschenden Ergebnissen: `yes | head -n 1` beendet `yes` mit Signal 13 (Code 141), und die Pipeline meldet bei `pipefail` einen Fehler, obwohl alles wie vorgesehen verlief. Hier muss man den Code 141 bewusst tolerieren.

```bash
# falsch: Fehler in der ersten Stufe bleibt unbemerkt
daten=$(curl -s "$url" | jq '.items[]')
echo "Status: $?"      # Status von jq, nicht von curl!

# richtig: pipefail aktivieren oder Stufen einzeln pruefen
set -o pipefail
daten=$(curl -sf "$url" | jq '.items[]') || { echo "Abruf fehlgeschlagen" >&2; exit 1; }
```

> **Merke**
>
> Exit-Code 0 = Erfolg. `$?` sofort sichern. Pipeline-Status ist standardmäßig der des letzten Befehls; `PIPESTATUS` und `set -o pipefail` machen Fehler sichtbar. 126/127/128+n haben feste Bedeutungen. `grep` liefert 1 bei „kein Treffer“.

> **Praxisbeispiel**
>
> Ein CI-Job führt `build.sh | tee build.log` aus, um die Ausgabe zu protokollieren. Ohne `pipefail` gilt der Job als erfolgreich, selbst wenn der Build fehlschlägt, denn `tee` meldet 0. Das Ergebnis sind „grüne“ Pipelines mit kaputten Artefakten. Die Zeile `set -o pipefail` am Skriptanfang behebt das.

## 4.3 Command Substitution

Die *Befehlsersetzung* (Command Substitution) führt einen Befehl aus und setzt dessen Ausgabe als Text an die Stelle des Ausdrucks ein. Die moderne Schreibweise ist `$(befehl)`; die alte mit Backticks (```befehl```) ist veraltet. Der Grund: `$(...)` lässt sich ohne Escape-Orgien *schachteln* (`$(basename $(pwd))`), behandelt Anführungszeichen im Inneren sauber und ist besser lesbar. Backticks verwendet man höchstens in sehr alten Skripten, die man nicht anfasst.

Zwei Eigenschaften sind wichtig. Erstens entfernt die Shell *alle abschließenden Zeilenumbrüche* aus der Ausgabe. Das ist meist erwünscht, kann aber überraschen, wenn man genau diese Umbrüche braucht. Zweitens läuft der Befehl in einer *Subshell*, einer Kopie der aktuellen Shell. Variablenänderungen in der Substitution wirken sich daher nicht auf die äußere Shell aus. Außerdem unterliegt das Ergebnis, wenn man es nicht quotet, dem Word Splitting und Globbing; deshalb gilt wieder die Regel: `"$(befehl)"` in doppelten Anführungszeichen. Der Exit-Code der Substitution ist der des Befehls; bei einer reinen Zuweisung (`x=$(befehl)`) wird er zum Status der Zuweisung und lässt sich mit `$?` oder in `if` prüfen. Bei `local x=$(befehl)` oder `export x=$(befehl)` dagegen geht er verloren (siehe 3.8).

Verwandt ist die *Prozesssubstitution*: `<(befehl)` und `>(befehl)` liefern einen Dateinamen (typisch `/dev/fd/63`), hinter dem die Ausgabe beziehungsweise Eingabe des Befehls steckt. Man kann sie überall verwenden, wo ein Programm Dateiargumente erwartet, etwa `diff <(sort a.txt) <(sort b.txt)`, ohne Zwischendateien. Sie ist, wie `[[`, ein Bash-Feature und in POSIX-`sh` nicht vorhanden. Seit Bash 5.3 gibt es zusätzlich Varianten, die den Befehl ohne Subshell in der aktuellen Shell ausführen (`${ befehl; }`), was Variablenänderungen erhält und einen Prozessstart spart.

> **Ergänzung aus der Nachrecherche**
>
> Nach den ausgewerteten Berichten zu Bash 5.3 (OMG! Ubuntu, Juli 2025) bringt diese Version die neue Form der Befehlsersetzung `${ befehl; }`, die im aktuellen Shell-Kontext läuft, und `${| befehl; }`, das das Ergebnis in der Variable `REPLY` ablegt. Für Skripte, die auf verschiedenen Systemen laufen sollen, bleibt `$(...)` die richtige Wahl, bis Bash 5.3 überall verbreitet ist.

```bash
#!/usr/bin/env bash
# substitution-demo.sh

heute="$(date +%F)"
echo "Heute: $heute"

# Schachtelung ohne Escapes
elternname="$(basename "$(dirname "$PWD")")"
echo "Name des Elternverzeichnisses: $elternname"

# Abschliessende Zeilenumbrueche werden entfernt
x="$(printf 'text\n\n\n')"
echo "Laenge: ${#x}"                          # 4

# Variable in Substitution aendert nichts draussen
zaehler=0
ergebnis="$(zaehler=99; echo "innen: $zaehler")"
echo "$ergebnis, aussen: $zaehler"            # aussen bleibt 0

# Exit-Code pruefen
if ausgabe="$(ls /nicht/vorhanden 2>&1)"; then
    echo "ok"
else
    echo "Fehler (Code $?): $ausgabe"
fi

# Mehrere Zeilen in ein Array
mapfile -t dateien < <(printf '%s\n' "a b" "c")
echo "${#dateien[@]} Eintraege, erster: '${dateien[0]}'"

# Prozesssubstitution: zwei Befehle vergleichen
diff <(printf '1\n2\n3\n') <(printf '1\n3\n') && echo "gleich" || echo "unterschiedlich"

# Rechnen mit $(( ))
echo "Sekunden pro Tag: $((24 * 60 * 60))"
```

Die Schachtelung `"$(basename "$(dirname "$PWD")")"` zeigt, dass innerhalb von `$(...)` ein neuer Quoting-Kontext beginnt; die inneren Anführungszeichen stören die äußeren nicht. Der Test mit `printf` beweist das Entfernen der Zeilenumbrüche. Das Beispiel mit der Variablen `zaehler` beweist die Subshell-Natur. In der `if`-Bedingung wird die Zuweisung selbst geprüft; der Code ist der von `ls`. `mapfile` liest mit der Prozesssubstitution Zeilen ohne Subshell-Problem in ein Array. `diff` vergleicht zwei „virtuelle Dateien“. Die Ausdrücke `$((...))` rechnen mit Ganzzahlen; Gleitkomma beherrscht Bash nicht, dafür nimmt man `bc` oder `awk`.

Typische Fehler: Erstens das Vergessen der Anführungszeichen (`rm $(find ...)` bricht bei Leerzeichen). Zweitens das Auslesen von Befehlsausgaben, wenn eine Datei einfacher ist: `$(cat datei)` ist meist durch `$(< datei)` (schneller, ohne Prozess) oder besser durch direkte Weiterverarbeitung ersetzbar. Drittens das Parsen von menschenlesbarer Ausgabe (`$(ls -l | awk ...)`), die sich mit dem System ändern kann; besser sind Werkzeuge mit maschinenlesbarer Ausgabe (`stat -c`, `find -printf`). Viertens das Ignorieren der Subshell: Wer Variablen in `$(...)` setzt, hat sie danach nicht.

```bash
# falsch: Backticks, Quoting fehlt, Ausgabe wird gesplittet
dateien=`ls *.txt`
for f in $dateien; do echo $f; done

# richtig: Glob in Array, korrekt gequotet
dateien=(*.txt)
for f in "${dateien[@]}"; do echo "$f"; done
```

> **Merke**
>
> Immer `"$(befehl)"` statt Backticks. Abschließende Zeilenumbrüche fallen weg, Variablenänderungen im Inneren gehen verloren, der Exit-Code geht bei `local x=$(...)` verloren. Prozesssubstitution `<(...)` ersetzt Zwischendateien.

> **Praxisbeispiel**
>
> Ein Release-Skript bildet den Versionstext aus Git: `version="$(git describe --tags --always)"`. Die Ausgabe ersetzt die Versionsnummer in Dateinamen, Docker-Tags und Logs, ohne dass jemand sie von Hand pflegt.

## 4.4 Prozessverwaltung

Ein *Prozess* ist ein laufendes Programm mit eigenem Speicher, eigener Umgebung und eigener Prozess-ID (*PID*). Jeder Prozess (außer dem allerersten, `init` bzw. `systemd` mit PID 1) wurde von einem *Elternprozess* gestartet und hat dessen PID als *PPID*. Die Shell startet Programme, indem sie sich selbst kopiert (*fork*) und die Kopie das neue Programm lädt (*exec*); danach wartet sie auf das Ende (*wait*) und liest den Exit-Code. Dieses Zusammenspiel erklärt viele Eigenschaften: Warum Kindprozesse die Umgebung erben, warum `cd` ein Builtin sein muss und warum Variablen nie nach oben wandern.

Zum Beobachten dient `ps`. Die verbreiteten Aufrufe sind `ps aux` (alle Prozesse, BSD-Stil) und `ps -ef` (alle Prozesse, System-V-Stil); mit `ps -o pid,ppid,stat,cmd -p PID` wählt man Spalten selbst. Die Spalte `STAT` nennt den Zustand: `R` läuft, `S` schläft (wartet), `T` gestoppt, `Z` Zombie. Ein *Zombie* ist ein beendeter Prozess, dessen Eltern den Exit-Code noch nicht abgeholt haben; er belegt nur noch einen Tabelleneintrag und verschwindet, sobald der Elternprozess `wait` aufruft oder selbst endet. `top` und `htop` zeigen Prozesse live.

Zum Suchen eignet sich `pgrep muster`, das PIDs passend zu Namen oder (mit `-f`) zur gesamten Kommandozeile liefert; `pgrep -a` zeigt auch die Kommandozeile. Zum Beenden dienen `kill PID` und `pkill muster`. Trotz des Namens senden diese Befehle zunächst nur *Signale*, kurze Benachrichtigungen an einen Prozess. Standardmäßig ist es `SIGTERM` (15), die höfliche Aufforderung zum Beenden, die ein Programm abfangen kann, um aufzuräumen. `SIGKILL` (9) kann nicht abgefangen werden: Der Kernel beendet den Prozess sofort ohne Aufräumen. `SIGINT` (2) entspricht Strg+C, `SIGHUP` (1) bedeutet ursprünglich „Terminal aufgelegt“ und wird heute oft als „Konfiguration neu laden“ genutzt, `SIGSTOP`/`SIGCONT` (19/18) halten an und setzen fort. Die Reihenfolge ist wichtig: Erst `SIGTERM`, kurz warten, erst im Notfall `SIGKILL`.

```bash
#!/usr/bin/env bash
# prozess-demo.sh

echo "Meine PID: $$, Eltern-PID: $PPID"
ps -o pid,ppid,stat,cmd -p $$

echo "--- Hintergrundprozess starten und finden"
sleep 300 &
kind=$!                                          # PID des letzten Hintergrundprozesses
echo "Kind-PID: $kind"
ps -o pid,ppid,stat,etime,cmd -p "$kind"
pgrep -a -f 'sleep 300' | head -n 1

echo "--- Existiert der Prozess? (Signal 0 sendet nichts)"
if kill -0 "$kind" 2>/dev/null; then echo "laeuft"; fi

echo "--- Hoeflich beenden, dann notfalls hart"
kill -TERM "$kind"
for i in 1 2 3 4 5; do
    kill -0 "$kind" 2>/dev/null || break
    sleep 0.2
done
if kill -0 "$kind" 2>/dev/null; then
    echo "reagiert nicht, sende SIGKILL"
    kill -KILL "$kind"
fi
wait "$kind" 2>/dev/null
echo "Exit-Code des Kindes: $? (128+15 = 143 bei SIGTERM)"

echo "--- Prozessbaum"
ps -o pid,ppid,cmd --forest -p $$ --ppid $$ 2>/dev/null | head -n 5
```

Die Variablen `$$` und `$PPID` geben die eigene und die Eltern-PID an; `$!` liefert die PID des zuletzt gestarteten Hintergrundprozesses (das `&` dahinter wird in 4.5 erklärt). `kill -0` sendet *kein* Signal, sondern prüft nur, ob der Prozess existiert und man ihn signalisieren darf, ein Standardmuster für Existenzprüfungen. Das Beenden folgt dem höflichen Muster: erst `SIGTERM`, dann eine kurze Wartezeit mit Prüfschleife, erst danach `SIGKILL`. `wait` holt den Exit-Code des Kindes ab: 143, weil 128 plus Signalnummer 15.

Zu den häufigen Fehlern zählt `kill -9` als erste Reaktion. Das verhindert jede Aufräumarbeit: Lock-Dateien bleiben liegen, temporäre Dateien werden nicht gelöscht, Datenbanken beenden Schreibvorgänge nicht sauber. Ein weiterer Fehler ist `pkill name` mit zu allgemeinem Muster, das unbeteiligte Prozesse trifft (`pkill java` beendet *alle* Java-Prozesse des Benutzers). Besser ist es, PID-Dateien zu verwenden oder mit `pgrep -x` (exakter Name) und `-u` (Benutzer) einzugrenzen. Und: Eine ältere PID kann nach dem Ende neu vergeben werden; wer PID-Dateien nicht prüft, signalisiert womöglich einen fremden Prozess.

```bash
# falsch: trifft jeden Prozess, dessen Kommandozeile "app" enthaelt
pkill -f app

# richtig: exakter Name und eigener Benutzer
pkill -x -u "$USER" meinedienst

# falsch: sofort hart beenden
kill -9 "$pid"

# richtig: erst TERM, kontrolliert warten, dann KILL
kill "$pid"; sleep 2; kill -0 "$pid" 2>/dev/null && kill -9 "$pid"
```

> **Merke**
>
> Prozess = Programm + PID + PPID. `fork` und `exec` erklären, warum Variablen nicht nach oben wandern. `kill` sendet Signale: TERM (15) bittet, KILL (9) erzwingt, `kill -0` prüft. Erst höflich, dann hart. Zombies sind beendete Prozesse ohne abgeholten Exit-Code.

> **Praxisbeispiel**
>
> Ein Dienst-Skript speichert beim Start die PID in `/run/dienst.pid`. Beim Stoppen liest es die Datei, prüft mit `kill -0`, sendet `SIGTERM`, wartet bis zu zehn Sekunden und löscht danach die PID-Datei. Moderne Systeme überlassen diese Aufgabe `systemd` (Modul 7), aber das Prinzip bleibt dasselbe.

## 4.5 Job Control

Die *Jobsteuerung* (Job Control) ist eine Funktion interaktiver Shells, mit der man mehrere Prozesse von einem Terminal aus verwaltet. Ein *Job* ist eine Pipeline oder ein Befehl, den die Shell als Einheit führt. Er läuft entweder im *Vordergrund* (die Shell wartet, Tastatureingaben gehen an den Job) oder im *Hintergrund* (die Shell gibt sofort wieder den Prompt frei).

Ein Befehl mit angehängtem `&` startet im Hintergrund: `langer_befehl &`. Die Shell gibt Jobnummer und PID aus (etwa `[1] 4711`). Mit `jobs` listet man alle Jobs, `fg %1` holt Job 1 in den Vordergrund, `bg %1` lässt einen gestoppten Job im Hintergrund weiterlaufen. Strg+Z sendet `SIGTSTP` und hält den Vordergrundjob an; Strg+C sendet `SIGINT` und beendet ihn. Mit `disown` löst man einen Job von der Shell, damit er deren Beenden überlebt, und `nohup befehl &` startet einen Prozess, der `SIGHUP` (Terminal geschlossen) ignoriert und seine Ausgabe nach `nohup.out` schreibt. Für langlaufende Arbeit über SSH sind `tmux` oder `screen` die komfortablere Lösung.

In Skripten läuft die Jobsteuerung nicht (sie ist nur in interaktiven Shells aktiv), aber `&` und `wait` funktionieren dort sehr wohl und sind die Grundlage für einfache Parallelverarbeitung. Die Variable `$!` enthält die PID des zuletzt gestarteten Hintergrundprozesses. `wait` ohne Argument wartet auf *alle* Hintergrundprozesse der Shell, `wait PID` auf einen bestimmten und liefert dessen Exit-Code, `wait -n` (Bash ≥ 4.3) auf den *nächsten fertigen*. Damit lassen sich Aufgaben parallel starten, die Zahl der gleichzeitigen Prozesse begrenzen und Fehler einsammeln.

```bash
#!/usr/bin/env bash
# parallel-demo.sh - Aufgaben parallel ausfuehren, Fehler einsammeln

arbeite() {                      # simuliert eine Aufgabe
    local id=$1
    sleep "0.$((RANDOM % 5 + 1))"
    (( id % 4 == 0 )) && return 1     # jede vierte schlaegt fehl
    return 0
}

max_parallel=3
declare -A pid_zu_id
fehler=0

echo "--- 1. Alle parallel starten, dann auf alle warten"
for id in 1 2 3 4 5 6; do
    arbeite "$id" &
    pid_zu_id[$!]=$id
done
for pid in "${!pid_zu_id[@]}"; do
    if ! wait "$pid"; then
        echo "Aufgabe ${pid_zu_id[$pid]} fehlgeschlagen"
        fehler=$((fehler + 1))
    fi
done
echo "Fehler gesamt: $fehler"

echo "--- 2. Hoechstens $max_parallel gleichzeitig (wait -n)"
laufend=0
for id in $(seq 1 8); do
    while (( laufend >= max_parallel )); do
        wait -n || true              # einer ist fertig; Fehler hier ignorieren
        laufend=$((laufend - 1))
    done
    arbeite "$id" &
    laufend=$((laufend + 1))
done
wait                                 # Rest abwarten
echo "alle Aufgaben abgeschlossen"

echo "--- 3. Einfache Variante mit xargs"
printf '%s\n' 1 2 3 4 5 6 | xargs -P 3 -I{} sh -c 'sleep 0.1; echo "fertig {}"' | sort
```

Der erste Block startet sechs Aufgaben gleichzeitig; das assoziative Array `pid_zu_id` merkt sich, welche PID zu welcher Aufgabe gehört, damit der Fehlerbericht lesbar bleibt. Die Schleife mit `wait "$pid"` holt jeden Exit-Code einzeln ab; `wait` gibt den Code des beendeten Prozesses zurück, was in der `if`-Bedingung ausgewertet wird. Der zweite Block begrenzt die Zahl paralleler Prozesse: Solange `max_parallel` erreicht ist, blockiert `wait -n` bis irgendein Prozess fertig ist; das `|| true` verhindert einen Abbruch bei fehlgeschlagenen Aufgaben, wenn `set -e` aktiv wäre. Der dritte Block zeigt die Alternative: `xargs -P` begrenzt die Parallelität selbst und erspart viel Verwaltungscode (mehr in Modul 10).

Typische Stolperfallen: Erstens vergisst man `wait`, und das Skript endet, bevor die Hintergrundarbeit fertig ist. Dann gehen Ergebnisse verloren, oder es bleiben verwaiste Prozesse zurück. Zweitens liest man Ausgaben mehrerer paralleler Prozesse durcheinander, weil alle gleichzeitig auf dasselbe Terminal schreiben; Abhilfe: pro Aufgabe eine eigene Logdatei. Drittens teilen sich parallele Prozesse Dateien oder Ressourcen und überschreiben sich gegenseitig (*Race Condition*). Viertens werden Variablen im Hintergrundprozess nicht nach außen gegeben: Er läuft in einer Subshell, ein `zaehler=$((zaehler+1))` dort ändert die Variable des Hauptskripts nicht.

```bash
# falsch: Skript endet, Hintergrundjob wird zur Waise
./grosser-import.sh &
echo "Fertig"

# richtig: gezielt warten und Status auswerten
./grosser-import.sh &
import_pid=$!
# ... andere Arbeit ...
if wait "$import_pid"; then echo "Import erfolgreich"; else echo "Import fehlgeschlagen" >&2; fi
```

> **Merke**
>
> `&` startet im Hintergrund, `$!` liefert die PID, `wait` sammelt Status ein, `wait -n` wartet auf den nächsten. Jobsteuerung (`fg`, `bg`, `jobs`, Strg+Z) gibt es nur interaktiv. Hintergrundprozesse laufen in Subshells, brauchen eigene Logdateien und müssen abgewartet werden.

> **Praxisbeispiel**
>
> Ein Deployment-Skript verteilt ein Release auf zehn Server. Statt zehn Mal nacheinander zu warten, startet es alle Kopiervorgänge parallel (maximal vier gleichzeitig), sammelt die Exit-Codes und meldet am Ende genau, welche Server fehlgeschlagen sind. Die Dauer sinkt von zehn Minuten auf drei.

## 4.6 Subshells und Grouping

Die Shell kennt zwei Arten, Befehle zusammenzufassen, und sie unterscheiden sich wesentlich. Runde Klammern `( befehle )` führen die Befehle in einer *Subshell* aus, einem Kindprozess, der eine Kopie der aktuellen Shell ist. Geschweifte Klammern `{ befehle; }` bilden eine *Gruppe*, die in der *aktuellen* Shell läuft. Das sieht ähnlich aus, hat aber andere Folgen.

In einer Subshell sind Änderungen an Variablen, am Arbeitsverzeichnis (`cd`), an Shell-Optionen (`set`) und an der `umask` lokal: Sie verschwinden beim Ende der Subshell. Das kann man gezielt nutzen: `( cd /tmp && tar -czf ... )` wechselt das Verzeichnis nur vorübergehend und lässt das Skript am alten Ort zurück. Auch ein `exit` in der Subshell beendet nur diese. Nachteil: Der Start einer Subshell kostet einen Prozess, und Ergebnisse müssen über stdout oder Exit-Code zurückkommen.

Bei einer Gruppe gelten Änderungen weiter, dafür bündelt sie mehrere Befehle für eine gemeinsame Umleitung oder Pipe: `{ echo kopf; date; } > datei`. Die Syntax verlangt ein Leerzeichen nach der öffnenden Klammer und ein Semikolon (oder Zeilenumbruch) vor der schließenden, weil `{` und `}` Schlüsselwörter sind.

Daneben entstehen Subshells *unsichtbar* in anderen Konstrukten: Jeder Teil einer Pipeline (außer bei `shopt -s lastpipe` der letzte), Befehlsersetzung `$(...)`, Prozesssubstitution, Hintergrundprozesse mit `&` und die Klammern selbst. Dieses Wissen löst viele „Variable ist verschwunden“-Rätsel. Die Variable `$BASH_SUBSHELL` zeigt die Schachtelungstiefe.

```bash
#!/usr/bin/env bash
# subshell-demo.sh
start="$PWD"
x=1

echo "--- 1. Subshell: Aenderungen bleiben lokal"
( cd /tmp; x=99; echo "innen: pwd=$PWD x=$x subshell=$BASH_SUBSHELL" )
echo "aussen: pwd=$PWD x=$x subshell=$BASH_SUBSHELL"

echo "--- 2. Gruppe: Aenderungen bleiben erhalten"
{ cd /tmp; x=42; }
echo "aussen: pwd=$PWD x=$x"
cd "$start"

echo "--- 3. Gruppe fuer gemeinsame Umleitung"
{ echo "Kopf"; printf 'a\nb\n'; echo "Fuss"; } | sed 's/^/> /'

echo "--- 4. Das Pipeline-Problem"
summe=0
printf '1\n2\n3\n' | while read -r n; do summe=$((summe + n)); done
echo "Summe nach Pipe: $summe"                # 0 - Schleife lief in Subshell

summe=0
while read -r n; do summe=$((summe + n)); done < <(printf '1\n2\n3\n')
echo "Summe mit Prozesssubstitution: $summe"  # 6

shopt -s lastpipe
summe=0
printf '1\n2\n3\n' | while read -r n; do summe=$((summe + n)); done
echo "Summe mit lastpipe: $summe"             # 6 (in nicht interaktiven Shells)
shopt -u lastpipe

echo "--- 5. exit in Subshell beendet nur diese"
( echo "vor exit"; exit 3; echo "wird nie gedruckt" )
echo "Status der Subshell: $?"

echo "--- 6. Voruebergehend Umgebung aendern"
( export LANG=C; sort <<< $'b\nA\na' | tr '\n' ' ' ); echo
( umask 077; touch /tmp/nur-ich-$$; ls -l /tmp/nur-ich-$$ | cut -c1-10; rm /tmp/nur-ich-$$ )
```

Im ersten Block endet die Änderung von Arbeitsverzeichnis und Variable mit der Subshell; die Variable `BASH_SUBSHELL` zeigt 1 innen und 0 außen. Im zweiten Block bleiben beide Änderungen erhalten, weil die Gruppe in der aktuellen Shell läuft; man beachte das Semikolon vor der schließenden Klammer. Der dritte Block bündelt drei Befehle für eine gemeinsame Pipe. Der vierte Block demonstriert das berühmteste Subshell-Problem: Die Variable bleibt 0, weil `while` als Teil der Pipeline in einer Subshell läuft. Zwei Auswege sind gezeigt: die Schleife mit Prozesssubstitution zu füttern, oder `shopt -s lastpipe`, das den letzten Pipeline-Teil in der aktuellen Shell ausführt (wirkt nur, wenn die Jobsteuerung aus ist, also in Skripten). Der fünfte Block zeigt, dass `exit` in der Subshell nur diese beendet und den Code 3 liefert. Der sechste verwendet Subshells, um `LANG` und `umask` lokal zu setzen.

Fehler und Fallstricke: Erstens hält man eine Gruppe in der Pipeline für „normal“, obwohl sie dort ebenfalls in einer Subshell läuft. Zweitens schreibt man `{ befehl }` ohne Semikolon, was zu einem Syntaxfehler führt. Drittens verwendet man `( cd dir; befehl )` und vergisst, dass bei einem fehlschlagenden `cd` der Befehl trotzdem im *alten* Verzeichnis läuft. Richtig ist `( cd dir && befehl )`. Viertens erwartet man, dass `set -e` aus der Subshell in die Außenwelt wirkt; der Fehlercode der Subshell muss von außen geprüft werden.

```bash
# falsch: bei fehlgeschlagenem cd wird im aktuellen Verzeichnis geloescht!
( cd "$verz"; rm -rf ./* )

# richtig: nur bei erfolgreichem cd weitermachen
( cd "$verz" && rm -rf -- ./* )
```

> **Merke**
>
> `( )` = Subshell (Kopie, Änderungen lokal), `{ ; }` = Gruppe (aktuelle Shell, Änderungen bleiben). Pipeline-Teile, `$(...)` und `&` erzeugen unsichtbar Subshells. Variablen aus Schleifen hinter Pipes gehen verloren; `< <(...)` oder `lastpipe` helfen.

> **Praxisbeispiel**
>
> Ein Build-Skript wechselt für jedes Teilprojekt in dessen Ordner und ruft dort `make` auf. Mit `( cd "$projekt" && make )` bleibt das Hauptskript stets im Wurzelverzeichnis, und jede Aufgabe kann das Verzeichnis ändern, ohne die nächste zu beeinflussen.

## 4.7 Trap und Signale

Mit `trap` legt ein Skript fest, was bei einem Signal oder bei bestimmten Ereignissen geschehen soll. Das ist die Grundlage für ordentliches *Aufräumen* (Cleanup): Temporäre Dateien löschen, Sperrdateien entfernen, Hintergrundprozesse beenden, Teilergebnisse zurückrollen, auch wenn jemand Strg+C drückt oder ein Dienst das Skript beendet. Die Syntax lautet `trap 'befehl' SIGNAL...`; der Befehl (meist der Aufruf einer Funktion) steht in einfachen Anführungszeichen, damit Variablen erst beim Auslösen aufgelöst werden. `trap - SIGNAL` stellt das Standardverhalten wieder her, `trap '' SIGNAL` ignoriert das Signal, `trap -p` listet die gesetzten Traps.

Neben echten Signalen kennt Bash Pseudo-Signale. `EXIT` wird *immer* beim Ende der Shell ausgeführt, egal ob normal, durch `exit`, durch Fehler oder durch ein abgefangenes Signal; dadurch ist es der beste Ort für Aufräumcode. `ERR` wird bei jedem fehlgeschlagenen Befehl ausgelöst (nützlich für Fehlerdiagnose mit `set -e`), `DEBUG` vor jedem Befehl, `RETURN` beim Verlassen einer Funktion. Die wichtigsten echten Signale: `SIGINT` (Strg+C), `SIGTERM` (höflicher Abbruch, zum Beispiel durch `kill`, Docker oder systemd), `SIGHUP` (Terminal geschlossen oder Reload), `SIGUSR1`/`SIGUSR2` (frei verwendbar), `SIGPIPE` (Schreiben in geschlossene Pipe). `SIGKILL` und `SIGSTOP` können nicht abgefangen werden.

```bash
#!/usr/bin/env bash
# trap-demo.sh - sauberes Aufraeumen und Signalbehandlung
set -u

arbeitsverz="$(mktemp -d)"
sperre="/tmp/trap-demo.lock"
abbruch=0

aufraeumen() {
    local code=$?                     # Exit-Code des Skripts sichern
    trap - EXIT                       # nicht doppelt ausfuehren
    echo "[cleanup] loesche $arbeitsverz und $sperre (Code $code)"
    rm -rf -- "$arbeitsverz"
    rm -f -- "$sperre"
    exit "$code"
}

beim_abbruch() {
    echo "[signal] $1 empfangen - beende kontrolliert"
    abbruch=1
}

trap aufraeumen EXIT
trap 'beim_abbruch SIGINT'  INT
trap 'beim_abbruch SIGTERM' TERM
trap 'echo "[err] Zeile $LINENO: Befehl fehlgeschlagen" >&2' ERR

# Sperrdatei: nur ein Lauf gleichzeitig
if ! ( set -o noclobber; echo "$$" > "$sperre" ) 2>/dev/null; then
    echo "Skript laeuft bereits (PID $(cat "$sperre"))" >&2
    trap - EXIT
    rm -rf -- "$arbeitsverz"
    exit 1
fi

echo "Arbeite in $arbeitsverz (PID $$) ..."
for schritt in 1 2 3 4 5; do
    (( abbruch )) && { echo "Abbruch vor Schritt $schritt"; exit 130; }
    echo "Schritt $schritt" > "$arbeitsverz/schritt$schritt.txt"
    sleep 0.3
    [ "$schritt" -eq 2 ] && kill -TERM $$    # simuliert externes Signal
done
echo "fertig"
```

Die Funktion `aufraeumen` sichert zuerst `$?`, denn schon der erste Befehl in der Funktion würde den Code überschreiben. `trap - EXIT` verhindert, dass der Trap beim `exit` in der Funktion erneut ausgelöst wird. Die Funktion löscht Arbeitsverzeichnis und Sperrdatei und beendet das Skript mit dem ursprünglichen Code. Für `INT` und `TERM` wird nur ein Merker gesetzt, den die Hauptschleife an definierten Stellen prüft, damit nie mitten in einem kritischen Schritt abgebrochen wird. Der `ERR`-Trap gibt die Zeilennummer aus `$LINENO` aus. Die Sperrdatei wird mit `noclobber` angelegt: Die Umleitung `>` scheitert, wenn die Datei schon existiert, was einen einfachen, wenn auch nicht perfekten Schutz gegen doppelte Läufe ergibt (robuster ist `flock`, siehe Modul 10). In der Schleife schickt sich das Skript selbst ein `SIGTERM`, um die Behandlung vorzuführen; in Schritt drei wird der Merker erkannt, das Skript beendet sich mit 130, und der `EXIT`-Trap räumt auf.

Typische Fehler: Erstens Anführungszeichen vergessen: `trap "echo $x" EXIT` löst `x` sofort auf, nicht erst beim Auslösen. Zweitens Aufräumen nur in `SIGINT` und `SIGTERM` behandeln und `EXIT` auslassen, sodass bei normalem Ende oder Fehlern nichts passiert. Drittens Traps in Funktionen, die globale Traps überschreiben; ein neuer Trap für dasselbe Signal *ersetzt* den alten. Viertens lange Aufräumarbeiten im Trap, die selbst durch ein zweites Signal unterbrochen werden. Fünftens: `SIGKILL` lässt sich nicht abfangen, ein Skript, das so beendet wird (zum Beispiel vom OOM-Killer), räumt nie auf; deshalb sollten Sperren und temporäre Dateien nach einem Absturz erkennbar und bereinigbar sein.

```bash
# falsch: Variable wird beim Setzen aufgeloest, Cleanup loescht die falsche Datei
tmp=/tmp/alt
trap "rm -f $tmp" EXIT
tmp=$(mktemp)

# richtig: einfache Quotes, Aufloesung beim Ausloesen (oder Funktion)
tmp=$(mktemp)
trap 'rm -f -- "$tmp"' EXIT
```

> **Merke**
>
> `trap 'funktion' EXIT` ist der Standardweg zum Aufräumen. `EXIT` deckt normales Ende, Fehler und abgefangene Signale ab. Trap-Befehl in einfache Quotes setzen, Exit-Code sichern, `SIGKILL` ist nicht abfangbar. Bei `INT` und `TERM` lieber Merker setzen und kontrolliert enden.

> **Praxisbeispiel**
>
> Ein Backup-Skript legt Arbeitsdateien in einem temporären Verzeichnis an und sperrt die Datenbank. Wird es bei `docker stop` mit `SIGTERM` beendet, sorgt der `EXIT`-Trap dafür, dass Sperre und Temp-Dateien verschwinden. Ohne ihn bliebe die Datenbank gesperrt, und der nächste Lauf scheiterte.

## 4.8 Named Pipes und tee

Eine normale Pipe (`|`) existiert nur für die Dauer einer Befehlskette und verbindet Prozesse, die die Shell gemeinsam startet. Eine *Named Pipe* (FIFO, „first in, first out“) ist dagegen ein Eintrag im Dateisystem, über den *unabhängige* Prozesse Daten austauschen können. Man erzeugt sie mit `mkfifo name`; `ls -l` zeigt sie mit dem Typbuchstaben `p`. Ein Prozess schreibt in die Pipe (`echo hallo > name`), ein anderer liest (`cat < name`). Beide Seiten sind *blockierend*: Der Schreiber wartet, bis ein Leser da ist, und umgekehrt. Es werden keine Daten auf die Platte geschrieben; der Kernel puffert sie nur im Speicher. Die Named Pipe erlaubt damit Konstrukte, die mit normalen Pipes nicht gehen, zum Beispiel einen langlaufenden Dienst, der Aufträge aus einer Pipe liest, oder zwei Verarbeitungsketten, die denselben Datenstrom gemeinsam nutzen.

`tee` (benannt nach dem T-Stück in der Rohrleitung) liest stdin und schreibt es sowohl auf stdout als auch in eine oder mehrere Dateien. Damit „zapft“ man einen Datenstrom an, ohne ihn zu unterbrechen: `befehl | tee log.txt | weiter`. Wichtige Optionen sind `-a` (anhängen statt überschreiben) und `-i` (Unterbrechungssignale ignorieren). Ein häufiger Trick betrifft Rechte: `sudo echo text > /etc/datei` scheitert, weil die *Umleitung* von der unprivilegierten Shell ausgeführt wird; `echo text | sudo tee /etc/datei` funktioniert, da `tee` selbst mit erhöhten Rechten schreibt. Zusammen mit der Prozesssubstitution erlaubt `tee` auch das Aufteilen eines Stroms an mehrere Verbraucher: `befehl | tee >(verbraucher1) >(verbraucher2) > /dev/null`.

```bash
#!/usr/bin/env bash
# fifo-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'rm -rf -- "$verz"' EXIT

echo "--- 1. Named Pipe: zwei unabhaengige Prozesse"
mkfifo kanal
ls -l kanal | cut -c1-10                       # prwx... : p = Pipe

( for i in 1 2 3; do echo "Nachricht $i"; sleep 0.1; done > kanal ) &
while IFS= read -r zeile; do
    echo "Leser bekam: $zeile"
done < kanal
wait

echo "--- 2. tee: protokollieren und weiterreichen"
printf 'apfel\nbirne\napfel\n' | tee ausgabe.txt | sort | uniq -c
echo "Gespeichert wurde:"; cat ausgabe.txt

echo "--- 3. tee -a: anhaengen"
echo "Zeile 1" | tee log.txt   > /dev/null
echo "Zeile 2" | tee -a log.txt > /dev/null
cat log.txt

echo "--- 4. Strom auf mehrere Verbraucher verteilen"
seq 1 5 | tee >(awk '{s+=$1} END {print "Summe:", s}' > summe.txt) \
              >(wc -l > anzahl.txt) > /dev/null
sleep 0.3
cat summe.txt anzahl.txt

echo "--- 5. Skriptausgabe gleichzeitig auf den Schirm und ins Log"
exec > >(tee -a lauf.log) 2>&1
echo "Diese Zeile steht auf dem Terminal und in lauf.log"
```

Beim ersten Block startet ein Hintergrundprozess, der in die Pipe schreibt, während der Hauptteil sie liest; beide laufen gleichzeitig, weil jede Seite beim Öffnen blockiert, bis die Gegenseite da ist. Die Schreibschleife steht komplett in einer Gruppe mit einer gemeinsamen Umleitung, damit die Pipe nur einmal geöffnet wird; würde man jede `echo`-Zeile einzeln umleiten, wäre sie dreimal geöffnet und geschlossen worden. Der zweite Block protokolliert Zwischenergebnisse mitten in einer Pipeline. Der dritte zeigt `-a`. Im vierten Block verteilt `tee` einen Strom an zwei Prozesse, die per Prozesssubstitution gestartet werden; die kurze Pause gibt ihnen Zeit, die Ausgabe zu beenden, denn die Shell wartet auf Prozesssubstitutionen nicht zuverlässig. Der letzte Block ist das verbreitete Muster, *alle* Ausgaben eines Skripts zugleich auf dem Terminal und in einer Logdatei zu haben: `exec > >(tee -a lauf.log) 2>&1` leitet stdout in eine Prozesssubstitution um, in der `tee` läuft, und stderr dorthin.

Fehler und Fallstricke: Erstens *Verklemmungen* (Deadlocks) bei Named Pipes: Öffnet ein Prozess die Pipe zum Schreiben und es gibt nie einen Leser, bleibt er für immer hängen. Abhilfe: Beide Seiten in Hintergrund oder separaten Prozessen starten, oder die Pipe mit Lese-Schreib-Modus öffnen (`exec 3<>kanal`). Zweitens bleiben FIFOs nach einem Absturz im Dateisystem liegen; räumen Sie sie per `EXIT`-Trap auf. Drittens leert `tee` ohne `-a` die Zieldatei; wer versehentlich auf eine bestehende Logdatei schreibt, überschreibt sie. Viertens wird in `befehl | tee datei` der Exit-Code von `befehl` verschluckt (Code von `tee`); hier hilft `set -o pipefail` oder `${PIPESTATUS[0]}`.

```bash
# falsch: Umleitung laeuft mit den Rechten des Aufrufers
sudo echo "10.0.0.5 intern" >> /etc/hosts      # Permission denied

# richtig: tee schreibt mit den Rechten von sudo
echo "10.0.0.5 intern" | sudo tee -a /etc/hosts > /dev/null
```

> **Merke**
>
> `mkfifo` erzeugt eine Named Pipe: Schreiber und Leser blockieren, bis beide da sind. `tee` verzweigt einen Strom auf stdout und Dateien (`-a` hängt an). `sudo tee` löst das Rechteproblem bei Umleitungen. Mit `exec > >(tee -a log) 2>&1` geht die Skriptausgabe in Terminal und Log.

> **Praxisbeispiel**
>
> Ein Installationsskript zeigt dem Anwender den Fortschritt im Terminal und legt zugleich ein vollständiges Protokoll unter `/var/log/installer.log` ab. Bei Problemen schickt der Anwender einfach diese Datei an den Support, ohne dass er die Ausgabe nachstellen muss.

## 4.9 Übungsaufgaben zu Modul 04

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Was bewirken die beiden Zeilen `befehl > a.txt 2>&1` und `befehl 2>&1 > a.txt` jeweils, und warum unterscheiden sie sich? Skizzieren Sie, wohin Deskriptor 1 und 2 nach jedem Schritt zeigen.

**Aufgabe 2.** Nach `false | true` zeigt `$?` den Wert 0. Welche zwei Möglichkeiten gibt es, den Fehler der ersten Stufe zu bemerken, und welche Nebenwirkung kann `pipefail` bei `yes | head -n 1` haben?

**Aufgabe 3.** Nennen Sie je zwei Unterschiede zwischen `( ... )` und `{ ...; }`. Warum bleibt die Variable `summe` nach `seq 3 | while read n; do summe=$((summe+n)); done` leer, und wie behebt man das?

**Aufgabe 4.** Was ist der Unterschied zwischen `SIGTERM` und `SIGKILL`? Warum sollte man zuerst `SIGTERM` senden und welcher Exit-Code ergibt sich für einen mit `SIGTERM` beendeten Prozess?

**Aufgabe 5.** Schreiben Sie ein Skript `mit-log.sh BEFEHL [ARGUMENTE...]`, das einen beliebigen Befehl ausführt, dessen stdout und stderr getrennt in `out.log` und `err.log` schreibt *und* auch auf dem Terminal anzeigt, und am Ende mit dem Exit-Code des Befehls endet.

**Aufgabe 6.** Schreiben Sie ein Skript `stoppe.sh NAME`, das alle Prozesse des eigenen Benutzers mit exakt dem Namen `NAME` zuerst mit `SIGTERM` beendet, bis zu 5 Sekunden wartet und übrig gebliebene Prozesse mit `SIGKILL` beendet. Das Skript meldet, wie viele Prozesse gefunden und wie viele hart beendet wurden.

**Aufgabe 7.** Schreiben Sie ein Skript `parallel-pruefen.sh`, das eine Liste von Hostnamen aus einer Datei liest (ein Name pro Zeile), jeden mit `ping -c 1 -W 1` prüft, höchstens vier Prüfungen gleichzeitig laufen lässt und am Ende die Namen der nicht erreichbaren Hosts ausgibt. Der Exit-Code ist die Anzahl der nicht erreichbaren Hosts (maximal 255).

**Aufgabe 8.** Schreiben Sie ein Skript `sicher-arbeiten.sh`, das ein temporäres Verzeichnis anlegt, einen Datensatz darin verarbeitet (simulieren Sie Arbeit mit `sleep`), eine Sperrdatei verwendet und bei jedem Ende (normal, Fehler, Strg+C, `SIGTERM`) per Trap Verzeichnis und Sperrdatei entfernt. Testen Sie jeden Fall.

## 4.10 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Bei `befehl > a.txt 2>&1` wird zuerst Deskriptor 1 auf `a.txt` gelegt; dann wird Deskriptor 2 zu einer Kopie von Deskriptor 1, zeigt also ebenfalls auf `a.txt`. Beides landet in der Datei. Bei `befehl 2>&1 > a.txt` wird zuerst Deskriptor 2 zu einer Kopie von Deskriptor 1, der noch auf das Terminal zeigt; erst danach wird Deskriptor 1 auf `a.txt` gelegt. Ergebnis: stdout in der Datei, stderr weiter auf dem Terminal. Redirections werden von links nach rechts ausgewertet, und `2>&1` kopiert den *aktuellen* Zustand, kein dauerhafter Verweis.

**Lösung zu Aufgabe 2.** Erstens `set -o pipefail`, wodurch der Pipeline-Status zum letzten Fehlercode ungleich null wird. Zweitens das Array `PIPESTATUS`, das die Codes aller Stufen direkt nach der Pipeline enthält (`false | true` ergibt `1 0`). Bei `yes | head -n 1` beendet sich `yes`, sobald `head` die Pipe schließt, durch `SIGPIPE` (Code 141); mit `pipefail` meldet die Pipeline daher einen Fehler, obwohl alles korrekt lief. Man muss den Code 141 in solchen Fällen gezielt tolerieren.

**Lösung zu Aufgabe 3.** Subshell: läuft in einem Kindprozess, Änderungen an Variablen, Verzeichnis und Optionen verschwinden; `exit` beendet nur sie. Gruppe: läuft in der aktuellen Shell, Änderungen bleiben; Syntax verlangt Leerzeichen nach `{` und Semikolon vor `}`. Die Variable bleibt leer, weil `while` als Teil der Pipeline in einer Subshell läuft und die Änderungen mit ihr verschwinden. Behebung: `while ...; done < <(seq 3)` (Prozesssubstitution) oder `shopt -s lastpipe` (nur in Skripten wirksam).

**Lösung zu Aufgabe 4.** `SIGTERM` (15) ist eine Bitte zum Beenden, die der Prozess abfangen kann, um aufzuräumen (Dateien schließen, Sperren lösen, Puffer schreiben). `SIGKILL` (9) wird vom Kernel ohne Rückfrage durchgesetzt und kann weder abgefangen noch ignoriert werden. Deshalb zuerst `SIGTERM` und erst bei ausbleibender Reaktion `SIGKILL`. Der Exit-Code eines durch ein Signal beendeten Prozesses ist 128 plus Signalnummer, also 143 bei `SIGTERM` und 137 bei `SIGKILL`.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# mit-log.sh BEFEHL [ARGUMENTE...]
if [ "$#" -eq 0 ]; then
    echo "Aufruf: $0 BEFEHL [ARGUMENTE...]" >&2
    exit 2
fi

: > out.log
: > err.log

"$@" > >(tee -a out.log) 2> >(tee -a err.log >&2)
status=$?
sleep 0.2           # den tee-Prozessen Zeit zum Beenden geben
exit "$status"
```

Die beiden Prozesssubstitutionen starten je ein `tee`: das erste schreibt stdout in `out.log` und zeigt es an, das zweite schreibt stderr in `err.log` und gibt es über `>&2` auf stderr aus (sonst würden Fehler auf stdout erscheinen). `"$@"` führt den Befehl mit allen Argumenten korrekt gequotet aus. Der Status wird sofort gesichert. Die kurze Pause gibt den `tee`-Prozessen Zeit, ihre Puffer zu leeren, weil die Shell nicht auf Prozesssubstitutionen wartet.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# stoppe.sh NAME
[ "$#" -eq 1 ] || { echo "Aufruf: $0 NAME" >&2; exit 2; }
name="$1"

mapfile -t pids < <(pgrep -x -u "$(id -u)" -- "$name")
echo "Gefunden: ${#pids[@]} Prozess(e) namens '$name'"
(( ${#pids[@]} > 0 )) || exit 0

kill -TERM "${pids[@]}" 2>/dev/null

for _ in 1 2 3 4 5 6 7 8 9 10; do          # 10 x 0,5 s = 5 s
    rest=()
    for p in "${pids[@]}"; do
        kill -0 "$p" 2>/dev/null && rest+=("$p")
    done
    (( ${#rest[@]} == 0 )) && { echo "Alle sauber beendet"; exit 0; }
    sleep 0.5
done

echo "Hart beendet: ${#rest[@]}"
kill -KILL "${rest[@]}" 2>/dev/null
```

Das Skript sammelt PIDs mit `pgrep -x` (exakter Name) und `-u` (nur eigener Benutzer), sendet `SIGTERM` und prüft in einer Schleife mit `kill -0`, wer noch lebt. Nach fünf Sekunden werden die übrigen mit `SIGKILL` beendet und gezählt. Die Option `--` schützt vor Namen, die mit Bindestrich beginnen.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# parallel-pruefen.sh HOSTDATEI
[ "$#" -eq 1 ] && [ -f "$1" ] || { echo "Aufruf: $0 HOSTDATEI" >&2; exit 2; }
max=4
verz="$(mktemp -d)"
trap 'rm -rf -- "$verz"' EXIT

pruefe() {
    local host=$1
    if ping -c 1 -W 1 -- "$host" >/dev/null 2>&1; then
        return 0
    else
        echo "$host" > "$verz/$host.fehler"
        return 1
    fi
}

laufend=0
while IFS= read -r host; do
    [[ -z $host || $host == \#* ]] && continue      # leere Zeilen, Kommentare
    while (( laufend >= max )); do
        wait -n || true
        laufend=$((laufend - 1))
    done
    pruefe "$host" &
    laufend=$((laufend + 1))
done < "$1"
wait

fehlend=0
for f in "$verz"/*.fehler; do
    [ -e "$f" ] || break
    cat "$f"
    fehlend=$((fehlend + 1))
done
(( fehlend > 255 )) && fehlend=255
exit "$fehlend"
```

Parallele Prozesse laufen in Subshells und können keine Variablen zurückgeben; deshalb schreibt jeder Fehlerfall eine kleine Datei in ein temporäres Verzeichnis, das nach dem `wait` ausgewertet wird. Die Zahl gleichzeitiger Prüfungen wird mit `wait -n` begrenzt. Der `EXIT`-Trap entfernt das Verzeichnis. Die Mustererkennung `\#*` überspringt Kommentarzeilen, und die Begrenzung auf 255 beachtet, dass Exit-Codes nur 8 Bit haben.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# sicher-arbeiten.sh
set -u
sperre="/tmp/sicher-arbeiten.lock"
arbeit="$(mktemp -d)"
eigene_sperre=0                  # 1 erst, wenn WIR die Sperre angelegt haben

aufraeumen() {
    local code=$?
    trap - EXIT
    rm -rf -- "$arbeit"
    (( eigene_sperre )) && rm -f -- "$sperre"
    [ "$code" -eq 0 ] || echo "[cleanup] Ende mit Code $code" >&2
    exit "$code"
}
trap aufraeumen EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

if ! ( set -o noclobber; echo $$ > "$sperre" ) 2>/dev/null; then
    echo "Laeuft bereits (PID $(cat "$sperre" 2>/dev/null))" >&2
    exit 1
fi
eigene_sperre=1

for i in 1 2 3 4 5; do
    echo "Verarbeite Datensatz $i"
    echo "ergebnis $i" > "$arbeit/ergebnis$i"
    sleep 1
    [ "${FEHLER_BEI:-0}" -eq "$i" ] && { echo "Simulierter Fehler" >&2; exit 1; }
done
echo "Fertig, $(ls "$arbeit" | wc -l) Ergebnisse"
```

Der `EXIT`-Trap räumt in jedem Fall auf, auch bei `exit 1` oder nach den Signalen, bei denen die Handler absichtlich mit passendem Code `exit` aufrufen (130 und 143). Beim zweiten Lauf, der die Sperre nicht erhält, darf der Aufräumcode die fremde Sperre nicht löschen; deshalb merkt sich die Variable `eigene_sperre`, ob *dieser* Lauf die Sperre angelegt hat. Testen Sie: normal (`./sicher-arbeiten.sh`), Fehler (`FEHLER_BEI=3 ./sicher-arbeiten.sh`), Strg+C während des Laufs, `kill -TERM` aus einem zweiten Terminal und einen zweiten Start während des ersten Laufs. Prüfen Sie nach jedem Test mit `ls /tmp/sicher-arbeiten.lock`, ob die Sperre wie erwartet entfernt oder erhalten wurde.
