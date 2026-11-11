# Modul 05 - Textverarbeitung mit grep, sed, awk und jq

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (5.1 bis 5.9; 5.10 Übungsaufgaben, 5.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 12 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Als Richtwert für den Umfang gilt der höhere Wert von 12 h.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit         | Dauer (h) | Prio |
|:----|:--------------------|----------:|:----:|
| 5.1 | grep und ripgrep    |      1,00 |  1   |
| 5.2 | sed Grundlagen      |      1,00 |  1   |
| 5.3 | awk Grundlagen      |      1,00 |  1   |
| 5.4 | cut, sort, uniq, wc |      1,00 |  1   |
| 5.5 | CSV-Verarbeitung    |      1,00 |  1   |
| 5.6 | JSON mit jq         |      1,00 |  1   |
| 5.7 | Log-Analyse         |      1,00 |  2   |
| 5.8 | Regex-Praxis        |      1,00 |  2   |
| 5.9 | Datenqualität       |      1,00 |  2   |
|     | **Summe (Details)** |  **9,00** |      |

Dieses Modul behandelt die Werkzeuge, mit denen Skripte Daten lesen, filtern, umformen und auswerten: `grep` zum Suchen, `sed` zum Ersetzen, `awk` für spaltenorientierte Berechnungen, die kleinen Helfer `cut`, `sort`, `uniq` und `wc` sowie `jq` für JSON. Dazu kommen die typischen Anwendungsfälle Log-Analyse, CSV und Datenqualität. Die reguläre-Ausdrücke-Praxis steht in Lerneinheit 5.8, wird aber schon in 5.1 gebraucht; dort wird jeweils darauf verwiesen. Alle Beispiele erzeugen ihre Testdaten selbst.

## 5.1 grep und ripgrep

`grep` (von „global regular expression print“) durchsucht Text zeilenweise nach einem Muster und gibt die passenden Zeilen aus. Es liest Dateien oder stdin und ist das Standardwerkzeug, um in Logs, Konfigurationen und Programmcode etwas zu finden. Ein Muster ist entweder ein fester Text oder ein *regulärer Ausdruck* (Regex), eine Kurzschreibweise für Textmuster, die in 5.8 ausführlich behandelt wird. Für den Einstieg genügt: Ein Punkt steht für ein beliebiges Zeichen, `\^{}` für den Zeilenanfang, `$` für das Zeilenende, `*` für „beliebig oft das vorherige Zeichen“.

Es gibt drei Dialekte. Der Standard sind *Basic Regular Expressions* (BRE), in denen Zeichen wie `+`, `?`, `|` und Klammern erst mit Backslash ihre Sonderbedeutung bekommen. Mit `grep -E` (früher `egrep`) gelten *Extended Regular Expressions* (ERE), in denen diese Zeichen ohne Backslash wirken; das ist für die meisten Aufgaben angenehmer. Mit `grep -F` (früher `fgrep`) wird das Muster als *fester Text* ohne Sonderzeichen behandelt, was schneller und sicherer ist, wenn man nach Dingen wie `a.b*` wörtlich sucht. Die GNU-Variante `grep -P` bietet Perl-kompatible Ausdrücke mit Lookaheads, ist aber nicht überall verfügbar.

Die wichtigsten Optionen: `-i` (Groß-/Kleinschreibung ignorieren), `-v` (Zeilen, die *nicht* passen), `-n` (Zeilennummern), `-c` (nur Anzahl der passenden Zeilen), `-l` (nur Dateinamen mit Treffer), `-L` (Dateien ohne Treffer), `-o` (nur den passenden Teil ausgeben), `-w` (nur ganze Wörter), `-x` (ganze Zeile muss passen), `-q` (still, nur Exit-Code), `-r` (rekursiv durch Verzeichnisse), `-h` (Dateinamen unterdrücken), `-A n`, `-B n`, `-C n` (n Zeilen Kontext danach, davor, rundherum), `-e muster` (mehrere Muster, oder wenn das Muster mit Bindestrich beginnt) und `-f datei` (Muster aus einer Datei). `--include='*.log'` und `--exclude-dir=.git` schränken eine rekursive Suche ein. Der Exit-Code ist 0 bei mindestens einem Treffer, 1 bei keinem und 2 bei einem Fehler (siehe Modul 1.6).

`ripgrep` (Programm `rg`) ist ein modernes Werkzeug mit demselben Grundzweck. Laut der Projektseite (abgerufen) durchsucht es rekursiv das aktuelle Verzeichnis, beachtet standardmäßig `.gitignore`-, `.ignore`- und `.rgignore`-Dateien, überspringt versteckte und binäre Dateien und ist bei vielen Suchen schneller als `grep`; der Autor weist selbst darauf hin, dass ein einzelner Benchmark nie ausreicht. Unicode-Unterstützung ist immer aktiv, und mit `rg -P` steht eine PCRE2-Engine für Lookarounds und Rückverweise bereit. Typische Aufrufe sind `rg 'muster'`, `rg -n -w wort`, `rg -tpy muster` (nur Python-Dateien) und `rg --json muster` (maschinenlesbare Ausgabe).

> **Ergänzung aus der Nachrecherche**
>
> Quelle: README des Projekts ripgrep (abgerufen am 30.10.2026). Beachten Sie für Skripte: `rg` ist nicht auf jedem System installiert und verhält sich bewusst anders als `grep` (rekursiv, filtert nach `.gitignore`). In portablen Skripten bleibt `grep` die sichere Wahl, `rg` lohnt sich für interaktive Suchen in großen Quellbäumen.

```bash
#!/usr/bin/env bash
# grep-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

mkdir -p logs
cat > logs/app.log <<'ENDE'
2026-10-30 08:00:01 INFO  Server gestartet auf Port 8080
2026-10-30 08:00:05 WARN  Speicher bei 85 Prozent
2026-10-30 08:01:12 ERROR Verbindung zu db01 fehlgeschlagen (timeout)
2026-10-30 08:01:13 INFO  Neuer Versuch fuer db01
2026-10-30 08:01:15 ERROR Verbindung zu db01 fehlgeschlagen (refused)
2026-10-30 08:02:00 info  Hintergrundjob beendet
ENDE
echo "Passwort=geheim123" > logs/config.txt

echo "--- 1. einfache Suche mit Zeilennummern"
grep -n "ERROR" logs/app.log

echo "--- 2. ohne Gross-/Kleinschreibung, nur Anzahl"
grep -ic "info" logs/app.log

echo "--- 3. Zeilen, die NICHT passen"
grep -v "INFO" logs/app.log | head -n 2

echo "--- 4. ERE: mehrere Alternativen"
grep -E "WARN|ERROR" logs/app.log

echo "--- 5. nur den Treffer ausgeben (-o)"
grep -oE "db[0-9]+" logs/app.log | sort -u

echo "--- 6. Kontext: eine Zeile davor und danach"
grep -n -B1 -A1 "refused" logs/app.log

echo "--- 7. rekursiv, nur Dateinamen"
grep -rl "geheim" .

echo "--- 8. fester Text mit Sonderzeichen"
grep -F "(timeout)" logs/app.log

echo "--- 9. Exit-Code auswerten"
if grep -q "FATAL" logs/app.log; then echo "kritisch"; else echo "kein FATAL (Code $?)"; fi

echo "--- 10. mehrere Muster aus Datei"
printf 'db01\nPort\n' > muster.txt
grep -c -f muster.txt logs/app.log
```

Zeile für Zeile: Das Beispiel legt mit einem Here-Document ein kleines Log an (der Begrenzer in Anführungszeichen sorgt dafür, dass nichts ersetzt wird). Der erste Aufruf zeigt `-n`. `-ic` kombiniert zwei Optionen und zählt auch die klein geschriebene Zeile `info`. `-v` kehrt die Auswahl um. Das Muster `WARN|ERROR` ist mit `-E` eine Alternative; ohne `-E` müsste man `\|` schreiben. `-o` gibt nur den passenden Teil aus, und zusammen mit `sort -u` entsteht eine Liste der beteiligten Datenbankserver. Der Kontext-Aufruf zeigt Zeilen vor und nach dem Treffer, was beim Lesen von Logs sehr hilft. `-F` sucht die Klammern wörtlich. Im neunten Block wird der Exit-Code genutzt, ohne Ausgabe zu erzeugen; `$?` im `else`-Zweig ist hier 1. Der letzte Block nimmt Muster aus einer Datei.

Häufige Fehler: Erstens vergisst man die Anführungszeichen um das Muster. Ohne sie expandiert die Shell Sonderzeichen wie `*` und `?` oder trennt Wörter. Falsch: `grep Verbindung fehlgeschlagen log`; richtig: `grep "Verbindung fehlgeschlagen" log`. Zweitens beginnt ein Muster mit `-`; dann liest `grep` es als Option. Abhilfe: `grep -e "-muster"` oder `grep -- "-muster"`. Drittens wird `grep` ohne Datei und ohne Pipe aufgerufen und wartet scheinbar „hängend“ auf stdin. Viertens verwendet man bei unsauberen Daten (Binärdateien) `grep` und erhält nur „Binary file matches“; dann hilft `-a` (als Text behandeln). Fünftens ist der Exit-Code 1 bei `set -e` ein vermeintlicher Fehler (siehe Modul 6): Man schreibt `grep ... || true`, wenn „kein Treffer“ erlaubt ist.

```bash
# falsch: Pipe vor grep nur zum Lesen einer Datei, Muster ungequotet
cat logs/app.log | grep ERROR.*db

# richtig: Datei direkt uebergeben, Muster gequotet
grep 'ERROR.*db' logs/app.log

# falsch: Ausgabe zaehlen mit wc, obwohl grep -c reicht
grep ERROR logs/app.log | wc -l     # funktioniert, ist aber umstaendlich
```

> **Merke**
>
> `grep` filtert Zeilen: `-i -v -n -c -o -l -r -q`. `-E` für Alternativen und `+ ? ()`, `-F` für wörtlichen Text. Muster immer in Anführungszeichen. Exit-Code 0 = Treffer, 1 = kein Treffer, 2 = Fehler. `rg` ist komfortabler, aber nicht überall vorhanden.

> **Praxisbeispiel**
>
> Bei einer Störung sucht man in allen Logs der letzten Stunde nach einer Fehlermeldung samt Umgebung: `grep -rn -C3 "OutOfMemory" /var/log/app/`. Ein Monitoring-Skript nutzt `grep -q` in einer Bedingung, um bei bestimmten Meldungen einen Alarm auszulösen.

## 5.2 sed Grundlagen

`sed` (*stream editor*) liest Text zeilenweise, wendet Befehle darauf an und gibt das Ergebnis aus. Anders als ein Editor arbeitet es nicht interaktiv, sondern als Filter in einer Pipeline oder zum Bearbeiten von Dateien. Das Programm hält jeweils eine Zeile im sogenannten *Musterspeicher* (pattern space), führt die Befehle des Skripts darauf aus und gibt den Musterspeicher am Ende aus (es sei denn, man unterdrückt das mit `-n`). Der Aufruf lautet `sed [optionen] 'skript' [datei...]`.

Der wichtigste Befehl ist die Ersetzung `s/muster/ersatz/flags`. Das Muster ist ein regulärer Ausdruck (Standard BRE, mit `-E` ERE). Im Ersatz steht `&` für den gesamten Treffer und `\1` bis `\9` für Teile in Klammern (*Rückverweise*). Als Flags gibt es `g` (alle Vorkommen der Zeile, nicht nur das erste), `i` (Groß-/Kleinschreibung egal), eine Zahl (nur das n-te Vorkommen) und `p` (Zeile ausgeben, meist mit `-n`). Als Trennzeichen kann jedes Zeichen dienen, nicht nur der Schrägstrich: `s|/usr/bin|/opt/bin|g` vermeidet Escapes bei Pfaden.

Weitere Befehle: `d` löscht Zeilen, `p` druckt, `i\\text` und `a\\text` fügen Text davor und danach ein, `c` ersetzt die ganze Zeile, `y/abc/xyz/` ersetzt Zeichen einzeln (wie `tr`), `q` beendet nach der aktuellen Zeile. Vor jedem Befehl kann eine *Adresse* stehen, die bestimmt, auf welche Zeilen er wirkt: eine Zeilennummer (`3d`), ein Bereich (`2,5d`), das Ende (`$d`), ein Muster (`/muster/d`) oder ein Bereich zwischen zwei Mustern (`/start/,/ende/d`). Mit `!` hinter der Adresse kehrt man sie um.

Die Option `-i` bearbeitet die Datei *direkt*; `-i.bak` legt zuvor eine Sicherheitskopie mit Endung `.bak` an. Hier gibt es einen Unterschied zwischen GNU und BSD (macOS): GNU-`sed` akzeptiert `-i` ohne Argument, BSD-`sed` verlangt immer eine Endung (`-i ''`). Wer portabel schreiben will, nutzt `-i.bak` und löscht die Kopie danach, oder schreibt in eine Ausgabedatei.

```bash
#!/usr/bin/env bash
# sed-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

cat > app.conf <<'ENDE'
# Konfiguration
host = localhost
port = 8080
debug = true
pfad = /usr/local/bin
ENDE

echo "--- 1. einfache Ersetzung (erstes Vorkommen je Zeile)"
echo "a-b-c" | sed 's/-/+/'
echo "--- 2. alle Vorkommen"
echo "a-b-c" | sed 's/-/+/g'
echo "--- 3. Rueckverweis: Vor- und Nachname tauschen"
echo "Mustermann, Max" | sed -E 's/^([^,]+), (.+)$/\2 \1/'
echo "--- 4. Treffer mit & wiederverwenden"
echo "Preis 42" | sed 's/[0-9][0-9]*/<&>/'
echo "--- 5. Trennzeichen wechseln (Pfade)"
sed 's|/usr/local|/opt|' app.conf | tail -n 1
echo "--- 6. Zeilen loeschen: Kommentare und Leerzeilen"
sed -e '/^#/d' -e '/^$/d' app.conf
echo "--- 7. nur bestimmte Zeilen ausgeben"
sed -n '2,3p' app.conf
echo "--- 8. nur bei Mustertreffer ersetzen"
sed '/^port/s/8080/9090/' app.conf | grep port
echo "--- 9. Datei direkt aendern mit Sicherung"
sed -i.bak 's/^debug = true/debug = false/' app.conf
diff app.conf.bak app.conf
echo "--- 10. Zeile einfuegen (GNU sed)"
sed '/^host/a\
timeout = 30' app.conf | sed -n '2,4p'
echo "--- 11. Bereich zwischen zwei Mustern"
printf 'A\nSTART\nx\ny\nEND\nB\n' | sed '/START/,/END/d'
echo "--- 12. Leerzeichen am Zeilenende entfernen"
printf 'text   \n' | sed 's/[[:space:]]*$//' | cat -A
```

Im Beispiel wird die Wirkung jeder Variante sichtbar. Die Ersetzung ohne `g` ändert nur das erste Vorkommen pro Zeile; mit `g` alle. Im dritten Block zerlegt der Ausdruck die Zeile in zwei Gruppen (alles bis zum Komma, alles danach) und gibt sie in vertauschter Reihenfolge aus. `&` steht für den gesamten Treffer und umschließt hier die Zahl mit spitzen Klammern. Der Trick mit dem anderen Trennzeichen erspart das Escapen der Schrägstriche. Die Befehle `/^#/d` und `/^$/d` entfernen Kommentar- und Leerzeilen; mit `-e` lassen sich mehrere Befehle kombinieren. `-n` mit `p` druckt nur die gewählten Zeilen. Beim direkten Ändern hält `-i.bak` eine Sicherung bereit, und `diff` zeigt genau, was sich geändert hat. Die Bereichsadresse `/START/,/ENDE/` löscht alles zwischen den Markierungen einschließlich dieser. Die letzte Zeile nutzt die POSIX-Klasse `[[:space:]]`, die auch Tabulatoren erfasst; `cat -A` zeigt, dass kein Leerzeichen übrig bleibt.

Typische Fehler: Erstens die Verwechslung von BRE und ERE: In `sed 's/(a|b)/X/'` haben die Klammern keine Gruppenwirkung; man braucht `-E` oder Backslashes. Zweitens Sonderzeichen im Ersatzstring: `&` und `\` haben dort Bedeutung, und kommt der Ersatz aus einer Variablen, muss man sie schützen. Wer Benutzereingaben in ein `sed`-Skript einsetzt, öffnet außerdem eine Injektionslücke (siehe Modul 8). Drittens führt `sed -i` ohne Sicherung bei einem Fehler im Muster zu unwiederbringlichen Änderungen; darum erst ohne `-i` testen. Viertens greift `.*` gierig: `s/<.*>//` entfernt in `<a><b>` alles, nicht nur `<a>`. Fünftens variiert die Behandlung von `\n`, `\t` und `-i` zwischen GNU- und BSD-`sed`.

```bash
# falsch: Variable mit Sonderzeichen direkt im Ersatz
neu="a/b&c"
sed "s/alt/$neu/" datei        # bricht wegen / und & im Wert

# richtig: Wert fuer sed maskieren
neu_esc="$(printf '%s' "$neu" | sed -e 's/[\/&]/\\&/g')"
sed "s/alt/$neu_esc/" datei
```

> **Merke**
>
> `s/muster/ersatz/g` ist der Kernbefehl. Adressen wählen Zeilen (`3`, `/re/`, `a,b`), `-n ... p` druckt gezielt, `d` löscht, `-i.bak` ändert Dateien mit Sicherung. `-E` für ERE. Erst ohne `-i` testen, Sonderzeichen in Variablen maskieren.

> **Praxisbeispiel**
>
> Ein Deployment-Skript passt eine Konfigurationsvorlage an: `sed "s|@PORT@|$PORT|g" app.conf.tmpl > app.conf`. Platzhalter in Vorlagen werden so je Umgebung ersetzt, ohne dass jede Stufe eine eigene Konfigurationsdatei pflegen muss. Bei heiklen Werten (Passwörter mit Sonderzeichen) nimmt man besser `envsubst` oder ein Skript mit `awk`.

## 5.3 awk Grundlagen

`awk` ist eine kleine Programmiersprache für zeilen- und spaltenorientierte Textverarbeitung, benannt nach den Initialen von Aho, Weinberger und Kernighan. Sein Grundmodell: Es liest Eingabezeilen (*Records*), zerlegt jede in Felder (*Fields*) und führt für jede Zeile ein Programm aus, das aus Paaren `muster { aktion }` besteht. Passt das Muster (oder fehlt es), läuft die Aktion. Fehlt die Aktion, wird die Zeile ausgegeben. Der Aufruf lautet `awk 'programm' datei`, optional mit `-F trenner` für das Feldtrennzeichen (Standard: beliebiger Whitespace) und `-v name=wert` zum Übergeben von Variablen.

Die wichtigsten eingebauten Variablen sind `$0` (die ganze Zeile), `$1`, `$2`, ... (die Felder), `NF` (Zahl der Felder), `NR` (laufende Zeilennummer über alle Dateien), `FNR` (Zeilennummer in der aktuellen Datei), `FS` und `OFS` (Eingabe- und Ausgabe-Feldtrenner) sowie `RS` und `ORS` für die Zeilentrenner. Besondere Muster sind `BEGIN` (läuft einmal vor der ersten Zeile, etwa zum Setzen von `FS`) und `END` (läuft einmal danach, etwa zum Ausgeben von Summen). Variablen müssen nicht deklariert werden; sie sind anfangs leer beziehungsweise 0. Assoziative Arrays (`zaehler[$1]++`) sind eingebaut und machen Gruppierungen und Summen in einer Zeile möglich. Bedingungen und Schleifen entsprechen denen in C; `printf` funktioniert wie in der Shell.

Welcher `awk` läuft, ist systemabhängig: `gawk` (GNU), `mawk` (Debian/Ubuntu-Standard) und `busybox awk` unterscheiden sich in Erweiterungen wie `gensub`, `asort`, `strftime` oder Zeitfunktionen. Wer portabel bleiben will, nutzt nur POSIX-awk-Sprachumfang; die Beispiele hier laufen in `gawk` und `mawk`.

```bash
#!/usr/bin/env bash
# awk-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

cat > umsatz.txt <<'ENDE'
region   produkt   menge  preis
Nord     Apfel     10     0.50
Sued     Birne     5      0.80
Nord     Birne     8      0.80
Ost      Apfel     20     0.45
Sued     Apfel     12     0.50
ENDE

echo "--- 1. Spalten ausgeben (Kopfzeile ueberspringen)"
awk 'NR > 1 { print $1, $3 }' umsatz.txt

echo "--- 2. mit Bedingung (Muster)"
awk '$2 == "Apfel" && $3 > 10' umsatz.txt

echo "--- 3. Umsatz je Zeile berechnen, formatiert"
awk 'NR > 1 { printf "%-5s %-6s %6.2f\n", $1, $2, $3 * $4 }' umsatz.txt

echo "--- 4. Summe ueber alle Zeilen (END)"
awk 'NR > 1 { summe += $3 * $4 } END { printf "Gesamtumsatz: %.2f\n", summe }' umsatz.txt

echo "--- 5. Gruppieren mit assoziativem Array"
awk 'NR > 1 { menge[$1] += $3 } END { for (r in menge) print r, menge[r] }' umsatz.txt | sort

echo "--- 6. Durchschnitt und Maximum"
awk 'NR > 1 { s += $3; if ($3 > max) max = $3 } END { print "avg=" s/(NR-1), "max=" max }' umsatz.txt

echo "--- 7. anderes Trennzeichen"
echo "anna:42:Wien" | awk -F: '{ print $3 " (" $2 ")" }'

echo "--- 8. Variablen von aussen uebergeben"
min=10
awk -v min="$min" 'NR > 1 && $3 >= min { n++ } END { print n " Zeilen mit Menge >= " min }' umsatz.txt

echo "--- 9. Felder neu schreiben, Ausgabetrenner setzen"
awk 'BEGIN { OFS = ";" } NR > 1 { $1 = $1; print }' umsatz.txt | head -n 2

echo "--- 10. Anzahl Felder und Zeilennummer"
awk '{ print NR ": " NF " Felder" }' umsatz.txt | head -n 3
```

Der erste Aufruf nutzt `NR > 1`, um die Kopfzeile zu überspringen: Das Muster steht ohne Klammern vor der Aktion. Der zweite Aufruf besteht *nur* aus einem Muster; weil die Aktion fehlt, werden die passenden Zeilen unverändert ausgegeben. Der dritte rechnet mit Feldern; awk behandelt Zeichenketten automatisch als Zahlen, wenn sie so aussehen. `printf` steuert Breite und Nachkommastellen (`%6.2f`). In Block 4 sammelt `summe += ...` über alle Zeilen, und der `END`-Block gibt das Ergebnis aus. Block 5 zeigt die Stärke der assoziativen Arrays: `menge[$1] += $3` gruppiert nach Region, ohne dass man vorher sortieren müsste (anders als bei `uniq`). Die Reihenfolge der Schleife `for (r in menge)` ist undefiniert, deshalb wird die Ausgabe sortiert. Block 6 berechnet Mittelwert und Maximum; `NR-1` ist die Anzahl der Datenzeilen. Mit `-v` übergibt man Shell-Werte sicher als awk-Variable, statt sie in den Programmtext einzusetzen. Block 9 zeigt einen Trick: Die Zuweisung `$1 = $1` zwingt awk, die Zeile mit dem Ausgabetrenner `OFS` neu zusammenzusetzen.

Typische Fehler: Erstens Anführungszeichen. Das awk-Programm steht in *einfachen* Anführungszeichen, damit die Shell `$1` nicht selbst ersetzt; Shell-Variablen übergibt man mit `-v`. Falsch: `awk "{ print $1 }" datei` (die Shell ersetzt `$1` durch ihr eigenes Argument). Zweitens wird mit `==` auf Zeichenketten und Zahlen unterschiedlich verglichen: `$1 == 10` vergleicht numerisch, `$1 == "10"` als Text. Drittens vergisst man bei CSV-Dateien, dass Felder in Anführungszeichen Kommas enthalten können; `awk -F,` versagt dort (siehe 5.5). Viertens: Bei Leerzeichen im Feldinhalt zerlegt das Standardtrennzeichen falsch; dann explizit `-F'\t'` nutzen. Fünftens ist die Division durch null oder die Mittelwertbildung mit leerer Eingabe ein Laufzeitfehler, der abgefangen werden muss.

```bash
# falsch: Shell ersetzt $1 und $3 in doppelten Anführungszeichen
awk "NR > 1 { print $1, $3 }" umsatz.txt

# richtig: einfache Quotes fuer das Programm, -v fuer Shell-Werte
grenze=10
awk -v g="$grenze" 'NR > 1 && $3 > g { print $1, $3 }' umsatz.txt
```

> **Merke**
>
> awk = Muster + Aktion pro Zeile. `$1..$NF`, `NR`, `BEGIN`/`END`, assoziative Arrays für Gruppierung, `printf` für Formatierung. Programm in einfachen Quotes, Shell-Werte per `-v`. Unterschiede zwischen gawk und mawk beachten.

> **Praxisbeispiel**
>
> Ein Betriebsbericht summiert aus der Abrechnungsdatei die Kosten je Kostenstelle: `awk -F';' 'NR>1 {k[$2]+=$5} END {for (s in k) printf "%s;%.2f\n", s, k[s]}' abrechnung.csv`. Eine Handvoll Zeilen ersetzt hier eine Tabellenkalkulation.

## 5.4 cut, sort, uniq, wc

Neben den großen Werkzeugen gibt es kleine, die jeweils genau eine Aufgabe erledigen und sich ideal in Pipelines zusammensetzen lassen (Unix-Philosophie, siehe Modul 1.2). `cut` schneidet Spalten aus: mit `-d` legt man das Trennzeichen fest, mit `-f` die Felder (`-f1,3` oder `-f2-4`), mit `-c` Zeichenpositionen. `cut` kennt nur ein einzelnes Trennzeichen und keine „mehrere Leerzeichen als eines“-Logik; für uneinheitliche Leerzeichen eignet sich `awk`. `sort` sortiert Zeilen: `-n` numerisch, `-r` umgekehrt, `-u` ohne Duplikate, `-k n` nach Feld n, `-t x` mit Trennzeichen `x`, `-h` menschenlesbare Größen (`2K`, `1G`), `-V` Versionsnummern und `-s` stabil. `uniq` fasst *direkt aufeinanderfolgende* gleiche Zeilen zusammen; `-c` zählt, `-d` zeigt nur Duplikate, `-u` nur Einzelstücke. Deshalb steht davor fast immer `sort`. `wc` zählt: `-l` Zeilen, `-w` Wörter, `-c` Bytes, `-m` Zeichen. Weitere Helfer: `tr` (Zeichen ersetzen oder löschen), `paste` (Dateien spaltenweise verbinden), `join` (Dateien über ein gemeinsames Feld verknüpfen), `head`, `tail` und `tac` (Zeilen umkehren).

Ein Hinweis zu `sort`: Die Sortierreihenfolge hängt von der *Locale* ab. In einer deutschen UTF-8-Locale sortiert es nach Sprachregeln (Groß- und Kleinbuchstaben gemischt, Umlaute bei den Grundbuchstaben), mit `LC_ALL=C` dagegen strikt nach Bytewerten (alle Großbuchstaben vor den Kleinbuchstaben). Für reproduzierbare Ergebnisse in Skripten setzt man `LC_ALL=C` vor `sort`, `uniq` und `join`; das ist außerdem deutlich schneller.

```bash
#!/usr/bin/env bash
# kleinwerkzeuge-demo.sh
export LC_ALL=C                         # reproduzierbare Sortierung

daten='anna;42;Wien
ben;17;Graz
clara;42;Linz
dora;9;Wien
emil;17;Wien'

echo "--- 1. Spalten ausschneiden"
printf '%s\n' "$daten" | cut -d';' -f1,3

echo "--- 2. nach Alter numerisch absteigend sortieren"
printf '%s\n' "$daten" | sort -t';' -k2,2nr

echo "--- 3. Haeufigkeit je Ort (klassisches Muster)"
printf '%s\n' "$daten" | cut -d';' -f3 | sort | uniq -c | sort -rn

echo "--- 4. nur Duplikate (Alter, das mehrfach vorkommt)"
printf '%s\n' "$daten" | cut -d';' -f2 | sort -n | uniq -d

echo "--- 5. Zaehlen"
printf '%s\n' "$daten" | wc -l
printf 'ein zwei drei\n' | wc -w

echo "--- 6. Zeichen ersetzen und loeschen mit tr"
echo "Hallo Welt" | tr 'a-z' 'A-Z'
echo "a1b2c3" | tr -d '0-9'
echo "viele    Leerzeichen" | tr -s ' '

echo "--- 7. Dateien verbinden (join) und nebeneinander (paste)"
printf '1 Anna\n2 Ben\n' > /tmp/n.$$
printf '1 Wien\n2 Graz\n' > /tmp/o.$$
join /tmp/n.$$ /tmp/o.$$
paste -d';' /tmp/n.$$ /tmp/o.$$
rm -f /tmp/n.$$ /tmp/o.$$

echo "--- 8. Top 3 haeufigste Woerter eines Textes"
printf 'a b a c a b d\n' | tr ' ' '\n' | sort | uniq -c | sort -rn | head -n 3

echo "--- 9. Sortierreihenfolge: Locale macht den Unterschied"
printf 'b\nA\na\nB\n' | sort | tr '\n' ' '; echo
printf 'b\nA\na\nB\n' | LC_ALL=en_US.UTF-8 sort 2>/dev/null | tr '\n' ' '; echo
```

Im ersten Block gibt `-d';'` das Trennzeichen an und `-f1,3` wählt zwei Spalten. Block 2 zeigt die feine Syntax von `-k`: `2,2nr` bedeutet „nur Feld 2, numerisch, umgekehrt“; ohne die Endposition (`-k2`) würde der Schlüssel bis zum Zeilenende reichen. Block 3 ist das wohl häufigste Muster der Shell-Datenanalyse: Spalte herausschneiden, sortieren, zählen und nach Häufigkeit ordnen. Die Reihenfolge ist zwingend, weil `uniq` nur benachbarte Zeilen vergleicht. Block 4 zeigt `uniq -d`. Mit `tr` lassen sich Zeichen übersetzen (`a-z A-Z`), löschen (`-d`) und wiederholte Zeichen zusammenfassen (`-s`). `join` verknüpft zwei *sortierte* Dateien über das erste Feld, `paste` klebt sie zeilenweise zusammen. Der Top-Wörter-Block ist die klassische Wortzählung. Der letzte Block zeigt, dass die Locale das Ergebnis beeinflusst: Ist die angeforderte Locale auf dem System installiert, ordnet `sort` nach Sprachregeln (`a A b B`); ist sie es nicht, fällt es stillschweigend auf die Bytereihenfolge zurück, wie hier in der Testumgebung.

Typische Fehler: Erstens `uniq` ohne vorheriges `sort`. Zweitens `sort` ohne `-n` bei Zahlen: Dann gilt „10“ vor „9“, weil Text sortiert wird. Drittens `cut` mit mehreren Leerzeichen als Trenner, das leere Felder liefert; dort ist `awk '{print $2}'` robuster. Viertens zählt `wc -l` Zeilenumbrüche: Eine Datei ohne abschließenden Umbruch hat scheinbar eine Zeile weniger. Fünftens zeigt `wc -c` Bytes, nicht Zeichen; mit Umlauten ergeben `-c` und `-m` unterschiedliche Werte.

```bash
# falsch: Text-Sortierung bei Zahlen und uniq ohne sort
printf '10\n9\n10\n' | uniq -c | sort -r      # "9" vor "10"?? und uniq sieht 10,9,10 als verschieden

# richtig
printf '10\n9\n10\n' | sort -n | uniq -c | sort -rn
```

> **Merke**
>
> Die Standardkette zur Auswertung lautet: Spalte wählen (`cut`/`awk`), `sort`, `uniq -c`, `sort -rn`, `head`. `uniq` braucht sortierte Eingabe, Zahlen brauchen `sort -n`, reproduzierbare Ergebnisse brauchen `LC_ALL=C`.

> **Praxisbeispiel**
>
> Ein Admin will wissen, welche Fehlercodes im Webserver-Log am häufigsten vorkommen: `awk '{print $9}' access.log | sort | uniq -c | sort -rn | head`. In einer Minute weiß er, ob die 500er-Fehler überhandnehmen oder nur harmlose 404er anfallen.

## 5.5 CSV-Verarbeitung

CSV (*comma-separated values*) ist das verbreitetste Austauschformat für Tabellen. Jede Zeile ist ein Datensatz, die Felder sind durch ein Trennzeichen getrennt, meist Komma oder, im deutschsprachigen Raum wegen des Dezimalkommas, Semikolon. Die erste Zeile enthält häufig die Spaltennamen. Die Einfachheit täuscht: CSV ist kein streng standardisiertes Format (die häufig zitierte Beschreibung ist RFC 4180), und die Tücken liegen im Detail. Felder können in Anführungszeichen stehen, wenn sie das Trennzeichen oder Zeilenumbrüche enthalten; ein Anführungszeichen im Feld wird durch Verdopplung maskiert (`"Er sagte ""Hallo"""`). Dazu kommen unterschiedliche Zeichenkodierungen (UTF-8, Windows-1252), ein unsichtbares Byte-Order-Mark (BOM) am Dateianfang und Windows-Zeilenenden (CR LF).

Mit Shell-Bordmitteln lassen sich *einfache* CSV-Dateien gut verarbeiten, nämlich solche, deren Felder weder das Trennzeichen noch Zeilenumbrüche enthalten. Dann genügen `cut -d';'`, `awk -F';'` und `while IFS=';' read -r ...`. Sobald Felder in Anführungszeichen vorkommen, werden diese Werkzeuge falsch. Dafür gibt es spezialisierte Programme: `csvkit` (`csvcut`, `csvgrep`, `csvstat`), `miller` (`mlr`) oder `xsv`, außerdem Python mit dem Modul `csv` oder GNU awk mit `FPAT`. Eine Alternative ist, die Datei zuerst nach JSON zu konvertieren und mit `jq` weiterzuarbeiten. Für Skripte gilt: Prüfen Sie zuerst, ob die Daten wirklich „einfach“ sind, und entscheiden Sie dann bewusst.

```bash
#!/usr/bin/env bash
# csv-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

# Windows-Zeilenenden und BOM wie bei einem Excel-Export
printf '\xef\xbb\xbfname;alter;stadt;gehalt\r\nAnna;42;Wien;3200,50\r\nBen;17;Graz;1800\r\nClara;35;Linz;\r\n' > roh.csv

echo "--- 1. Rohdaten sichtbar machen"
head -n 2 roh.csv | cat -A

echo "--- 2. bereinigen: BOM und CR entfernen"
sed -e '1s/^\xef\xbb\xbf//' -e 's/\r$//' roh.csv > sauber.csv
head -n 2 sauber.csv | cat -A

echo "--- 3. Spalte waehlen"
cut -d';' -f1,3 sauber.csv

echo "--- 4. mit awk filtern und Spaltennamen verwenden"
awk -F';' 'NR==1 { for (i=1; i<=NF; i++) col[$i]=i; next }
           $col["alter"] >= 18 { print $col["name"] " (" $col["stadt"] ")" }' sauber.csv

echo "--- 5. zeilenweise mit read, Kopfzeile ueberspringen"
tail -n +2 sauber.csv | while IFS=';' read -r name alter stadt gehalt; do
    printf '%-6s %3s %-5s [%s]\n' "$name" "$alter" "$stadt" "$gehalt"
done

echo "--- 6. Dezimalkomma in Punkt umwandeln und summieren"
awk -F';' 'NR>1 && $4 != "" { gsub(",", ".", $4); s += $4 } END { printf "Summe: %.2f\n", s }' sauber.csv

echo "--- 7. Falle: Komma im Feld (Anfuehrungszeichen)"
printf 'id,name\n1,"Meier, Hans"\n' > falle.csv
awk -F, 'NR==2 { print "Feld 2 laut awk: " $2 }' falle.csv      # falsch: "Meier

echo "--- 8. Falle loesen mit gawk und FPAT"
gawk -v FPAT='([^,]*)|("[^"]+")' 'NR==2 { print "Feld 2 mit FPAT: " $2 }' falle.csv

echo "--- 9. Spaltenanzahl je Zeile pruefen"
awk -F';' '{ print NF }' sauber.csv | sort | uniq -c
```

Der erste Block macht mit `cat -A` sichtbar, was ein Excel-Export enthält: ein BOM (`M-oM-;M-?`) am Anfang und `\^{}M$` (Wagenrücklauf plus Zeilenende) an jedem Zeilenende. Der zweite Block entfernt beides mit `sed`. Die Auswahl von Spalten und die Filterung nach Namen in awk (Block 4) macht Skripte lesbar: In der Kopfzeile baut awk ein Array `col` von Spaltenname zu Spaltennummer auf, `next` springt zur nächsten Zeile, und später steht `$col["alter"]` statt `$2`. Block 5 liest mit `IFS=';' read -r` direkt in benannte Variablen; `tail -n +2` überspringt die Kopfzeile. Beachten Sie, dass die Schleife hinter einer Pipe in einer Subshell läuft (siehe Modul 4.6). Block 6 verwandelt das Dezimalkomma mit `gsub` in einen Punkt, weil awk Zahlen mit Komma nicht als Zahl erkennt. Die Blöcke 7 und 8 zeigen die Grenze: Mit `-F,` zerschneidet awk das Feld `"Meier, Hans"` in zwei Teile, `FPAT` (nur gawk) beschreibt dagegen, wie ein *Feld* aussieht, statt wie ein Trennzeichen. Der letzte Block ist eine einfache Plausibilitätsprüfung: Hat jede Zeile gleich viele Felder?

Typische Fehler: Erstens wird der Dateikopf (BOM) übersehen: Das erste Feld heißt dann nicht `name`, sondern enthält unsichtbare Bytes, und Vergleiche schlagen fehl. Zweitens bleiben Wagenrücklaufzeichen am Zeilenende im letzten Feld hängen, was Vergleiche wie `$4 == "x"` verfälscht. Drittens verwechselt man Dezimalkomma und -punkt. Viertens werden Felder nicht in Anführungszeichen gesetzt, wenn man CSV *erzeugt*; ein Feld mit Semikolon zerstört dann die Struktur. Fünftens ist die Zeichenkodierung falsch; `file -i datei` und `iconv -f WINDOWS-1252 -t UTF-8` helfen.

```bash
# falsch: Felder mit Trennzeichen zerlegt, CR bleibt am Ende haengen
awk -F, '{ print $2 }' export.csv

# richtig: erst bereinigen, dann spezialisiertes Werkzeug (hier: csvkit/miller, falls vorhanden)
sed 's/\r$//' export.csv | mlr --icsv --ojson cat
```

> **Merke**
>
> Vor der CSV-Verarbeitung prüfen: Trennzeichen, Kodierung, BOM, CR LF, Anführungszeichen. Einfache Dateien: `cut`, `awk -F`, `IFS=';' read`. Komplexe Dateien: `csvkit`, `mlr`, `gawk FPAT` oder Python. Spaltennamen in awk statt Nummern erhöhen die Lesbarkeit.

> **Praxisbeispiel**
>
> Eine Personalabteilung liefert monatlich einen Excel-Export als CSV. Das Importskript entfernt zuerst BOM und CR, prüft die Spaltenzahl jeder Zeile, wandelt Dezimalkommas um und weist fehlerhafte Zeilen in eine separate Datei aus, statt den ganzen Import abzubrechen.

## 5.6 JSON mit jq

JSON (*JavaScript Object Notation*) ist das Standardformat moderner APIs und Cloud-Werkzeuge. Es kennt Objekte (`{"schluessel": wert}`), Arrays (`[1, 2]`), Zeichenketten, Zahlen, Wahrheitswerte und `null`. Textwerkzeuge wie `grep` und `sed` sind für JSON ungeeignet, weil Struktur und Zeilen nicht zusammenhängen: Dasselbe Dokument kann in einer Zeile oder in hundert stehen. Das Werkzeug `jq` versteht JSON und arbeitet wie `sed` oder `awk` als Filter: Es liest JSON von stdin oder aus Dateien, wendet einen *Filterausdruck* an und gibt JSON (oder Text) aus.

Die Grundlagen: `.` ist der Identitätsfilter und gibt die Eingabe formatiert aus. `.name` greift auf ein Feld zu, `.a.b` auf verschachtelte Felder, `.[0]` auf ein Array-Element, `.[]` liefert alle Elemente eines Arrays als einzelne Ausgaben. Mit dem Pipe-Zeichen *innerhalb* des Filters (`.items[] | .name`) verkettet man Filter. `select(bedingung)` lässt nur passende Elemente durch, `map(filter)` wendet einen Filter auf alle Elemente eines Arrays an, `length` zählt, `keys` liefert Schlüssel, `sort_by(.feld)`, `group_by(.feld)`, `unique`, `add`, `min`/`max` und `has("feld")` decken die gängigen Aufgaben ab. Mit `{name: .n, id: .i}` baut man neue Objekte, mit `[...]` neue Arrays. `//` liefert einen Standardwert, wenn der Ausdruck `null` oder `false` ergibt.

Wichtige Optionen: `-r` (`--raw-output`) gibt Zeichenketten ohne Anführungszeichen aus, unverzichtbar für Shell-Variablen; `-c` gibt jedes Ergebnis kompakt in einer Zeile aus; `-e` setzt den Exit-Code nach dem Ergebnis (nützlich für Tests); `-s` (`--slurp`) liest alle Eingaben in ein Array; `-n` liest keine Eingabe; `--arg name wert` und `--argjson name json` übergeben Werte *sicher* aus der Shell; `-R` liest Rohtext statt JSON. Für die Ausgabe in andere Formate gibt es Formatierer: `@csv`, `@tsv`, `@sh` (Shell-sichere Anführung), `@uri`, `@base64`.

> **Ergänzung aus der Nachrecherche**
>
> Nach den Release Notes zu jq 1.8.x (abgerufen am 30.10.2026) bringt die Version 1.8.0 unter anderem die Funktionen `trim`, `ltrim`, `rtrim`, `trimstr`, `toboolean`, `skip` und `@urid`. Es gab auch Verhaltensänderungen: `index`, `rindex` und `indices` rechnen auf Codepoints statt Bytes, `tonumber` lehnt Zahlen mit umgebenden Leerzeichen ab, und `ltrimstr`/`rtrimstr` melden bei Nicht-Zeichenketten einen Fehler. Die Versionen 1.8.1 und 1.8.2 enthalten zahlreiche Sicherheitskorrekturen. Skripte, die auf neue Funktionen angewiesen sind, sollten die jq-Version prüfen (`jq --version`); die Testumgebung dieses Skripts nutzt jq 1.7.

```bash
#!/usr/bin/env bash
# jq-demo.sh (benoetigt jq)
command -v jq >/dev/null || { echo "jq fehlt" >&2; exit 1; }

json='{
  "stadt": "Wien",
  "bewohner": [
    {"name": "Anna", "alter": 42, "rollen": ["admin", "dev"]},
    {"name": "Ben",  "alter": 17, "rollen": ["dev"]},
    {"name": "Clara","alter": 35, "rollen": []}
  ],
  "meta": {"erstellt": "2026-10-30", "version": 3}
}'

echo "--- 1. Feldzugriff, roh ausgegeben"
jq -r '.stadt' <<< "$json"
jq -r '.meta.version' <<< "$json"

echo "--- 2. alle Namen"
jq -r '.bewohner[].name' <<< "$json"

echo "--- 3. filtern: Erwachsene"
jq -c '.bewohner[] | select(.alter >= 18) | {name, alter}' <<< "$json"

echo "--- 4. map, Aggregation"
jq '[.bewohner[].alter] | add / length' <<< "$json"
jq '.bewohner | map(.alter) | max' <<< "$json"

echo "--- 5. Anzahl und Existenzpruefung"
jq '.bewohner | length' <<< "$json"
jq '.bewohner | map(select(.rollen | index("admin"))) | length' <<< "$json"

echo "--- 6. Standardwert fuer fehlende Felder"
jq -r '.bewohner[] | "\(.name): \(.email // "keine E-Mail")"' <<< "$json"

echo "--- 7. Tabellenausgabe als TSV und CSV"
jq -r '.bewohner[] | [.name, .alter] | @tsv' <<< "$json"
jq -r '["name","alter"], (.bewohner[] | [.name, .alter]) | @csv' <<< "$json"

echo "--- 8. Werte sicher aus der Shell uebergeben"
min=30
jq --argjson min "$min" -c '.bewohner[] | select(.alter > $min) | .name' <<< "$json"
gesucht="Ben"
jq --arg n "$gesucht" '.bewohner[] | select(.name == $n) | .alter' <<< "$json"

echo "--- 9. in Shell-Variablen und Arrays uebernehmen"
mapfile -t namen < <(jq -r '.bewohner[].name' <<< "$json")
echo "${#namen[@]} Namen, erster: ${namen[0]}"
stadt="$(jq -r '.stadt' <<< "$json")"
echo "Stadt: $stadt"

echo "--- 10. JSON veraendern und neu ausgeben"
jq -c '.meta.version += 1 | .bewohner |= map(.alter += 1) | del(.bewohner[].rollen)' <<< "$json"

echo "--- 11. jq als Test (Exit-Code)"
if jq -e '.bewohner | any(.alter < 18)' <<< "$json" >/dev/null; then
    echo "Es gibt Minderjaehrige"
fi

echo "--- 12. Fehlerhaftes JSON erkennen"
echo '{"a": 1,}' | jq . 2>&1 | head -n 1
```

Die Beispiele laufen nacheinander gegen dasselbe Dokument. In Block 1 liefert `-r` den Text ohne Anführungszeichen, wie ihn die Shell braucht. `.bewohner[].name` erzeugt eine Ausgabe pro Person. Block 3 kombiniert `select` mit einer Objektkonstruktion; `{name, alter}` ist die Kurzschreibweise für `{name: .name, alter: .alter}`. Block 4 berechnet den Durchschnitt, indem das Array der Alter mit `add` summiert und durch `length` geteilt wird. Block 5 prüft, ob `"admin"` im Rollen-Array vorkommt; `index` liefert die Position oder `null`, und `select` wertet `null` als falsch. Block 6 zeigt die String-Interpolation `"\(...)"` und den Standardwert `//`. Block 7 erzeugt TSV und CSV, wobei für CSV zunächst ein Array als Kopfzeile ausgegeben wird. Block 8 ist der *wichtigste Sicherheitshinweis* dieser Lerneinheit: Werte aus der Shell gehören per `--arg` oder `--argjson` an jq, *nie* per Zeichenketten-Einsetzung in den Filter. Block 9 übernimmt Ergebnisse in Variablen und Arrays. Block 10 ändert Daten mit `+=`, `|=` (Update) und `del`. Block 11 nutzt `-e` und `any`, damit der Exit-Code das Ergebnis trägt. Der letzte Block zeigt, dass jq ungültiges JSON mit Fehlermeldung ablehnt.

Typische Fehler: Erstens vergisst man `-r` und erhält Ausgaben wie `"Wien"` mit Anführungszeichen in Variablen. Zweitens setzt man Shell-Variablen direkt in den Filter ein (`jq ".[] | select(.name == \"$name\")"`), was bei Anführungszeichen im Wert den Filter bricht und eine Injektionslücke öffnet. Drittens verlässt man sich auf den Exit-Code bei fehlenden Feldern: `jq '.fehlt'` liefert `null` und Code 0, erst `-e` macht `null` zum Fehler. Viertens vergisst man das Array-Unwrapping: `.bewohner.name` statt `.bewohner[].name` liefert einen Fehler. Fünftens sind JSON-Zahlen große Ganzzahlen betroffen: jq arbeitet intern mit Gleitkommazahlen, IDs jenseits von $2^{53}$ können verfälscht werden; solche Werte behandelt man als Zeichenketten. Sechstens liest `jq` mehrere aufeinanderfolgende JSON-Dokumente (NDJSON, Logzeilen) einzeln; mit `-s` bekommt man sie als ein Array.

```bash
# falsch: Shell-Wert wird in den Filter eingebaut (bricht bei Anfuehrungszeichen)
name='Ben "der Grosse"'
jq ".bewohner[] | select(.name == \"$name\")" daten.json

# richtig: Wert als Variable uebergeben
jq --arg n "$name" '.bewohner[] | select(.name == $n)' daten.json

# falsch: Anfuehrungszeichen landen in der Variable
stadt=$(jq '.stadt' daten.json)        # "Wien" (mit Quotes)
# richtig
stadt=$(jq -r '.stadt' daten.json)     # Wien
```

> **Merke**
>
> `jq` ist `sed`/`awk` für JSON: `.feld`, `.[]`, `select`, `map`, `-r` für Shell-Variablen, `--arg` für sichere Übergabe, `-e` für Tests, `@csv`/`@tsv` für Tabellen. JSON nie mit `grep` oder `sed` parsen.

> **Praxisbeispiel**
>
> Ein Skript fragt mit `curl` eine REST-API ab und extrahiert aus der Antwort IDs aller Instanzen im Zustand `failed`: `curl -sf "$URL" | jq -r '.items[] | select(.state=="failed") | .id'`. Dasselbe Muster kommt bei Cloud-CLIs (`az`, `aws`, `kubectl -o json`) ständig vor (siehe Modul 11).

## 5.7 Log-Analyse

Logs sind das Tagebuch eines Systems. Die Analyse folgt meist denselben Schritten: Zeitraum eingrenzen, nach Fehlern filtern, Muster extrahieren, zählen und nach Häufigkeit ordnen. Wichtig ist zuerst, das *Format* zu kennen. Das *Combined Log Format* von Apache und nginx besteht pro Zeile aus Client-IP, Identität, Benutzer, Zeitstempel in eckigen Klammern, der Anfrage in Anführungszeichen, dem Statuscode, der Antwortgröße, dem Referrer und dem User-Agent. Anwendungslogs haben meist Zeitstempel, Level (`INFO`, `WARN`, `ERROR`) und Nachricht, zunehmend auch als JSON pro Zeile (*JSON Lines*). Syslog-Zeilen beginnen mit Zeitstempel und Hostname. Bei unbekannten Formaten betrachtet man zuerst `head` und `tail`, bevor man ein Muster schreibt.

Für Betriebsfragen sind „Top-N“-Auswertungen typisch: die häufigsten Fehlermeldungen, die aktivsten IP-Adressen, die langsamsten Anfragen, die Statuscodes pro Minute. Sie lassen sich fast immer mit der Kette `Extraktion, sort, uniq -c, sort -rn, head` bauen, und für Zeitreihen mit `awk` und assoziativen Arrays. Beim Zeitbezug helfen Zeitstempel im ISO-Format (`2026-10-30T08:01:12`): Sie lassen sich lexikografisch vergleichen, sodass `awk '$1 >= "2026-10-30T08:00"'` ohne Datumsfunktionen auskommt. Große Logs liest man nicht mit `cat`, sondern streamt sie; komprimierte Archive mit `zcat` oder `zgrep`.

```bash
#!/usr/bin/env bash
# log-analyse.sh
export LC_ALL=C
verz="$(mktemp -d)" && cd "$verz" || exit 1
trap 'cd /; rm -rf -- "$verz"' EXIT

cat > access.log <<'ENDE'
10.0.0.1 - - [30/Oct/2026:08:00:01 +0100] "GET /index.html HTTP/1.1" 200 1024 "-" "curl/8.0"
10.0.0.2 - - [30/Oct/2026:08:00:03 +0100] "GET /api/users HTTP/1.1" 200 512 "-" "Mozilla/5.0"
10.0.0.1 - - [30/Oct/2026:08:00:07 +0100] "POST /api/login HTTP/1.1" 401 64 "-" "curl/8.0"
10.0.0.3 - - [30/Oct/2026:08:01:10 +0100] "GET /missing HTTP/1.1" 404 128 "-" "Mozilla/5.0"
10.0.0.1 - - [30/Oct/2026:08:01:11 +0100] "GET /api/users HTTP/1.1" 500 32 "-" "curl/8.0"
10.0.0.2 - - [30/Oct/2026:08:01:59 +0100] "GET /api/users HTTP/1.1" 500 32 "-" "Mozilla/5.0"
10.0.0.1 - - [30/Oct/2026:08:02:30 +0100] "GET /index.html HTTP/1.1" 304 0 "-" "curl/8.0"
ENDE

echo "--- 1. Statuscodes zaehlen"
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn

echo "--- 2. Top-IPs"
awk '{ print $1 }' access.log | sort | uniq -c | sort -rn | head -n 3

echo "--- 3. nur Serverfehler (5xx) mit Pfad"
awk '$9 ~ /^5/ { print $4, $7, $9 }' access.log

echo "--- 4. Anteil der Fehler in Prozent"
awk '{ n++; if ($9 >= 400) f++ } END { printf "%.1f%% Fehler (%d von %d)\n", 100*f/n, f, n }' access.log

echo "--- 5. Anfragen pro Minute"
awk '{ split($4, t, ":"); minute = t[2] ":" t[3]; z[minute]++ } END { for (m in z) print m, z[m] }' access.log | sort

echo "--- 6. uebertragene Datenmenge je Pfad"
awk '{ b[$7] += $10 } END { for (p in b) printf "%8d %s\n", b[p], p }' access.log | sort -rn | head -n 3

echo "--- 7. Top-Pfade nur fuer GET"
awk '$6 == "\"GET" { print $7 }' access.log | sort | uniq -c | sort -rn | head -n 2

echo "--- 8. Anwendungslog: Fehlerarten ohne variable Teile"
cat > app.log <<'ENDE'
2026-10-30T08:01:12 ERROR Verbindung zu db01 fehlgeschlagen (timeout)
2026-10-30T08:01:15 ERROR Verbindung zu db02 fehlgeschlagen (timeout)
2026-10-30T08:02:00 ERROR Datei /tmp/x123 nicht gefunden
2026-10-30T08:02:30 ERROR Datei /tmp/y456 nicht gefunden
2026-10-30T08:03:00 INFO  Alles gut
ENDE
grep ' ERROR ' app.log | cut -d' ' -f3- | sed -E 's/db[0-9]+/dbN/; s#/tmp/[a-z0-9]+#/tmp/X#' | sort | uniq -c | sort -rn

echo "--- 9. Zeitraum eingrenzen (ISO-Zeitstempel sind sortierbar)"
awk '$1 >= "2026-10-30T08:01:30" && $1 <= "2026-10-30T08:02:30"' app.log

echo "--- 10. Log als JSON Lines auswerten"
printf '{"level":"error","msg":"a"}\n{"level":"info","msg":"b"}\n{"level":"error","msg":"c"}\n' \
    | jq -r 'select(.level=="error") | .msg'
```

Die Beispiele zeigen die typischen Bausteine. Block 1 und 2 sind die Standardkette mit awk als Spaltenauswahl; im Combined-Log-Format ist das neunte Feld der Statuscode und das erste die IP. Block 3 filtert mit einem regulären Ausdruck auf dem Feld (`$9 ~ /^5/`). Block 4 berechnet eine Fehlerquote in einem Durchlauf; man beachte `%%` in `printf` für ein Prozentzeichen. Block 5 zerlegt den Zeitstempel mit `split` und zählt pro Minute, was eine einfache Lastkurve ergibt. Block 6 summiert Bytes je Pfad. Block 7 filtert auf Feld 6, das in diesem Format mit einem Anführungszeichen beginnt (`"GET`). Block 8 ist besonders lehrreich: Um Meldungen zu *gruppieren*, ersetzt es variable Teile (Hostnummern, Dateinamen) durch Platzhalter, bevor gezählt wird; sonst wäre jede Zeile einzigartig. Block 9 nutzt die lexikografische Ordnung der ISO-Zeitstempel. Block 10 wertet JSON-Zeilen mit `jq` aus.

Fehler und Fallstricke: Erstens wird das Log mit `cat` gelesen und `grep` nachgeschaltet; direkter geht es mit `grep muster datei`. Zweitens zählt man ohne Normalisierung (siehe Block 8). Drittens ignoriert man Zeitzonen: Logs mehrerer Server in verschiedenen Zonen lassen sich nur vergleichen, wenn man sie auf UTC bringt. Viertens ändern Anwendungen ihr Logformat; Auswertungen, die auf Spaltennummern beruhen, brechen unbemerkt. Daher sollte eine Auswertung die erwartete Feldzahl oder das erwartete Muster prüfen und bei Abweichung warnen. Fünftens: Personenbezogene Daten (IP-Adressen, Benutzernamen) in Logs unterliegen Datenschutzregeln; Auswertungen sollten sie nur bei Bedarf weitergeben (siehe Modul 8, Datenschutz).

```bash
# falsch: Zeilen mit Muster zaehlen, aber Variationen werden einzeln gezaehlt
grep ERROR app.log | sort | uniq -c | sort -rn      # jede Zeile einzigartig

# richtig: variable Teile normalisieren, dann zaehlen
grep ERROR app.log | sed -E 's/[0-9]+/N/g' | sort | uniq -c | sort -rn
```

> **Merke**
>
> Log-Analyse: Format erkennen, Spalte oder Muster extrahieren, *variable Teile normalisieren*, zählen, sortieren, begrenzen. ISO-Zeitstempel sind sortier- und vergleichbar. Auswertungen sollen ein unerwartetes Format erkennen. Personenbezogene Daten sparsam behandeln.

> **Praxisbeispiel**
>
> Nach einem Deployment steigt die Fehlerrate. Das Team gruppiert die Fehlermeldungen der letzten 15 Minuten nach normalisiertem Text und sieht sofort, dass eine einzelne Meldung 90 Prozent ausmacht. Die Ursache ist in zehn Minuten gefunden, statt dass jemand Tausende Zeilen liest.

## 5.8 Regex-Praxis

Reguläre Ausdrücke (Regex) beschreiben Textmuster. Sie sind in `grep`, `sed`, `awk`, Bash (`[[ =~ ]]`) und fast jeder Programmiersprache vorhanden, leider in mehreren Dialekten. Diese Lerneinheit ordnet das Wichtigste.

*Bausteine*: Ein normales Zeichen steht für sich. Der Punkt `.` passt auf ein beliebiges Zeichen (außer Zeilenumbruch). Eckige Klammern bilden eine *Zeichenklasse*: `[abc]` passt auf eines der Zeichen, `[a-z0-9]` auf einen Bereich, `[^abc]` auf alle anderen. Vordefinierte POSIX-Klassen sind `[[:digit:]]`, `[[:alpha:]]`, `[[:alnum:]]`, `[[:space:]]`, `[[:upper:]]` und `[[:lower:]]`; sie sind locale-fest und portabel, im Gegensatz zu `\d` oder `\w`, die es nur in PCRE (und als GNU-Erweiterung in einigen Werkzeugen) gibt. *Anker*: `^` (Zeilenanfang), `$` (Zeilenende), `\<` und `\>` (Wortgrenzen bei GNU), `\b` (Wortgrenze, in GNU-Programmen und PCRE). *Quantoren*: `*` (null oder mehr), `+` (eins oder mehr), `?` (null oder eins), `{n}`, `{n,m}` (Anzahl). *Gruppen und Alternativen*: `(...)` fasst zusammen und merkt sich den Treffer, `a|b` bedeutet „a oder b“, `\1` verweist auf die erste Gruppe.

Der *Unterschied BRE und ERE*: In Basic Regular Expressions sind `+`, `?`, `|`, `(`, `)`, `{`, `}` gewöhnliche Zeichen, und man muss sie mit Backslash *aktivieren* (GNU-Erweiterung). In Extended Regular Expressions ist es umgekehrt: Sie sind Operatoren, und ein Backslash macht sie zu normalen Zeichen. `grep -E`, `sed -E` und `awk` arbeiten mit ERE, `grep` und `sed` ohne Option mit BRE, Bash in `[[ =~ ]]` mit ERE. `grep -P` nutzt PCRE mit `\d`, Lookarounds und *nicht gierigen* Quantoren.

Das Konzept der *Gier* (Greediness) ist die häufigste Fehlerquelle. Quantoren sind standardmäßig gierig: Sie passen auf so viel Text wie möglich. Das Muster `".*"` auf `"a" und "b"` trifft deshalb die gesamte Zeichenkette von der ersten bis zur letzten Anführung. Ein nicht gieriger Quantor (`.*?`, nur PCRE) oder besser eine *negierte Zeichenklasse* (`"[^"]*"`) trifft nur das Erste. Die negierte Klasse ist auch in POSIX-Dialekten verfügbar und meist schneller.

```bash
#!/usr/bin/env bash
# regex-demo.sh
echo "--- 1. BRE vs ERE: gleiche Absicht, andere Schreibweise"
printf 'aaa\nab\nb\n' | grep 'a\+b'        # BRE: \+ aktiviert den Quantor (GNU)
printf 'aaa\nab\nb\n' | grep -E 'a+b'      # ERE: + wirkt direkt

echo "--- 2. Gier: .* vs [^\"]*"
text='sagte "Hallo" und "Tschuess" gestern'
echo "$text" | grep -oE '".*"'              # gierig: alles zwischen erstem und letztem "
echo "$text" | grep -oE '"[^"]*"'           # nicht gierig durch negierte Klasse

echo "--- 3. Anker und Zeichenklassen: IPv4-aehnlich"
printf '10.0.0.1\n999.1\nabc\n192.168.1.20\n' | grep -E '^([0-9]{1,3}\.){3}[0-9]{1,3}$'

echo "--- 4. Gruppen und Rueckverweis: doppelte Woerter"
printf 'das das ist\nkein doppel\n' | grep -E '\b([a-z]+) \1\b'

echo "--- 5. Bash-Regex mit Capture-Gruppen"
zeile="2026-10-30 ERROR code=E42"
if [[ $zeile =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})\ ([A-Z]+)\ code=([A-Z0-9]+)$ ]]; then
    echo "Jahr=${BASH_REMATCH[1]} Monat=${BASH_REMATCH[2]} Level=${BASH_REMATCH[4]} Code=${BASH_REMATCH[5]}"
fi

echo "--- 6. Regex in einer Variablen (ungequotet verwenden!)"
muster='^[0-9]+$'
for w in 123 12a ""; do
    [[ $w =~ $muster ]] && echo "'$w' ist eine Zahl" || echo "'$w' ist keine Zahl"
done

echo "--- 7. sed mit Gruppen: Datum umformatieren"
echo "30.10.2026" | sed -E 's/^([0-9]{2})\.([0-9]{2})\.([0-9]{4})$/\3-\2-\1/'

echo "--- 8. PCRE: Lookahead (nur GNU grep)"
echo "Preis: 42 EUR, 17 USD" | grep -oP '\d+(?= EUR)' 2>/dev/null || echo "(grep -P nicht verfuegbar)"

echo "--- 9. Metazeichen wortlich suchen"
echo "Kosten: 3.5 (netto)" | grep -F "3.5 (netto)"
echo "3x5" | grep -c '3.5'            # Punkt passt auf x!
echo "3x5" | grep -c '3\.5'           # maskierter Punkt: kein Treffer
```

Block 1 zeigt den BRE/ERE-Unterschied am selben Muster. In Block 2 trifft das gierige Muster `".*"` den ganzen Satzteil, die negierte Klasse `"[^"]*"` dagegen beide Zitate einzeln. Block 3 prüft eine IPv4-ähnliche Adresse; `\.` ist ein maskierter Punkt, `{3}` wiederholt die Gruppe dreimal. (Das Muster prüft nur die Form, nicht den Wertebereich 0 bis 255; dafür braucht man zusätzliche Logik.) Block 4 nutzt den Rückverweis `\1`, um doppelte Wörter zu finden. Block 5 und 6 zeigen die Bash-eigene Regex-Auswertung mit `BASH_REMATCH`; entscheidend ist, dass das Muster *nicht in Anführungszeichen* stehen darf (sonst wird es wörtlich verglichen), und dass man es zur Lesbarkeit in einer Variablen ablegt. Block 7 formatiert ein Datum mit Rückverweisen im Ersatz um. Block 8 zeigt einen Lookahead, der nur prüft, was folgt, ohne es in den Treffer aufzunehmen. Block 9 erinnert daran, dass der Punkt ein Metazeichen ist.

Typische Fehler: Erstens werden Muster ohne Anführungszeichen übergeben, und die Shell expandiert `*` oder `?`. Zweitens setzt man PCRE-Syntax (`\d`, `\w`, `(?:...)`) in `grep -E` oder `sed` ein, wo sie nicht (oder je nach Version zufällig) funktioniert. Drittens vergisst man, Metazeichen zu maskieren (`.`, `*`, `[`, `$`, `\`). Viertens baut man Muster aus Benutzereingaben zusammen; die sicherere Variante ist `grep -F`. Fünftens: Zu komplizierte Ausdrücke sind schwer zu warten und können durch *katastrophales Backtracking* extrem langsam werden. Regel: Wenn ein Ausdruck länger als eine Zeile ist, lieber in Schritten (mehrere `grep`/`sed`) oder in einer Programmiersprache lösen, und ihn kommentieren.

```bash
# falsch: \d und \w gibt es in POSIX-ERE nicht
echo "id=42" | grep -E 'id=\d+'          # findet nichts (oder je nach grep Warnung)

# richtig: POSIX-Klasse (portabel) oder PCRE (nur GNU grep)
echo "id=42" | grep -E 'id=[[:digit:]]+'
echo "id=42" | grep -P 'id=\d+'
```

> **Merke**
>
> ERE (`-E`) ist für die meisten Aufgaben angenehmer als BRE. Quantoren sind gierig; negierte Klassen `[^x]*` begrenzen. POSIX-Klassen sind portabel. In Bash `[[ =~ ]]` mit ungequotetem Muster in einer Variablen. Metazeichen maskieren oder `grep -F` verwenden.

> **Praxisbeispiel**
>
> Ein Validierungsskript prüft Eingabedateien auf erwartete Muster: Kundennummern (`^K[0-9]{6}$`), Datumsangaben (`^[0-9]{4}-[0-9]{2}-[0-9]{2}$`) und E-Mail-Adressen in einer sehr einfachen Form. Weil Regex nur die *Form* prüft, folgen bei Bedarf inhaltliche Prüfungen (zum Beispiel ob das Datum existiert, mit `date -d`).

## 5.9 Datenqualität

Daten sind selten sauber. Bevor man Zahlen auswertet oder Datensätze importiert, sollte ein Skript prüfen, ob die Eingabe brauchbar ist. *Datenqualität* umfasst mehrere Dimensionen: *Vollständigkeit* (fehlen Pflichtfelder?), *Gültigkeit* (entsprechen Werte dem Format und Wertebereich?), *Eindeutigkeit* (gibt es unerwünschte Duplikate?), *Konsistenz* (stimmen zusammengehörige Felder überein?) und *Plausibilität* (gibt es Ausreißer?). Ein *Validierungsreport* fasst die Befunde zusammen: wie viele Zeilen insgesamt, wie viele fehlerhaft, welche Fehlerart wie oft, und welche Zeilen genau betroffen sind.

Die Shell eignet sich gut für solche Prüfungen, weil sie Dateien streamt und Prüfungen leicht kombinierbar sind. Leere Felder findet man mit awk (`$3 == ""`), Duplikate mit `sort`/`uniq -d` oder awk-Arrays (`seen[$1]++`), Formatfehler mit regulären Ausdrücken, Ausreißer mit Statistik (Mittelwert, Standardabweichung) in awk. Wichtig ist, *fehlerhafte Datensätze nicht stillschweigend zu verwerfen*, sondern getrennt abzulegen (*Quarantäne*-Datei) und zu melden. Der Exit-Code des Prüfskripts sollte erkennen lassen, ob Fehler gefunden wurden, damit nachgelagerte Schritte (Import, Pipeline) stoppen können.

```bash
#!/usr/bin/env bash
# validiere.sh DATEI.csv - prueft eine Semikolon-CSV mit Spalten id;name;email;alter
set -u
export LC_ALL=C
datei="${1:-}"
[ -f "$datei" ] || { echo "Aufruf: $0 DATEI.csv" >&2; exit 2; }

verz="$(mktemp -d)"
trap 'rm -rf -- "$verz"' EXIT
gut="$verz/gut.csv"; schlecht="$verz/schlecht.csv"

# Pruefung in einem awk-Durchlauf: gueltige Zeilen auf Datei 1, fehlerhafte mit Grund auf Datei 2
awk -F';' -v gut="$gut" -v schlecht="$schlecht" '
function fehler(grund) { print grund ";" $0 > schlecht; bad[grund]++; ok = 0 }
NR == 1 { print > gut; next }                      # Kopfzeile unveraendert
{
    ok = 1
    if (NF != 4)                              fehler("Spaltenzahl")
    else {
        if ($1 !~ /^[0-9]+$/)                 fehler("id ungueltig")
        else if (seen[$1]++)                  fehler("id doppelt")
        if ($2 == "")                         fehler("name fehlt")
        if ($3 !~ /^[^@ ]+@[^@ ]+\.[a-z]+$/)  fehler("email ungueltig")
        if ($4 !~ /^[0-9]+$/ || $4 < 0 || $4 > 120) fehler("alter unplausibel")
    }
    if (ok) { print > gut; n_gut++ } else n_schlecht++
    gesamt++
}
END {
    printf "Zeilen: %d, gueltig: %d, fehlerhaft: %d\n", gesamt, n_gut, n_schlecht
    for (g in bad) printf "  %-18s %d\n", g, bad[g]
    exit (n_schlecht > 0)
}' "$datei"
status=$?

if [ -s "$schlecht" ]; then
    echo "Fehlerhafte Zeilen (Grund;Zeile):"
    cat "$schlecht"
fi
exit "$status"
```

Das Skript prüft die Datei in einem einzigen awk-Durchlauf, was auch bei sehr großen Dateien effizient ist. Die Funktion `fehler` schreibt die Zeile samt Grund in die Quarantäne-Datei, zählt die Fehlerart in einem Array und markiert den Datensatz als schlecht. Die Prüfungen decken mehrere Dimensionen ab: Die Spaltenzahl (`NF != 4`) ist eine Strukturprüfung, die ID-Prüfung kombiniert Format (`!~ /^[0-9]+$/`) und Eindeutigkeit (`seen[$1]++` liefert beim zweiten Auftreten einen Wert größer null), der Name wird auf Vollständigkeit geprüft, die E-Mail auf eine einfache Form und das Alter auf Wertebereich. Am Ende gibt `END` einen Bericht aus und beendet awk mit dem Exit-Code 1, falls fehlerhafte Zeilen existieren; die Shell sichert den Code in `status`, zeigt die Fehlerzeilen und beendet sich mit demselben Code. Probieren Sie das Skript mit einer Datei wie dieser aus:

```text
id;name;email;alter
1;Anna;anna@example.org;42
2;Ben;ben@example.org;17
2;Clara;clara@example.org;35
4;;dora@example.org;29
5;Emil;emil-at-example.org;31
6;Fritz;fritz@example.org;250
7;Gerda;gerda@example.org
```

Erwartet werden fünf fehlerhafte Zeilen mit den Gründen `id doppelt`, `name fehlt`, `email ungueltig`, `alter unplausibel` und `Spaltenzahl`.

Ausreißer erkennt man statistisch. Ein einfaches Verfahren: Mittelwert und Standardabweichung berechnen und Werte markieren, die mehr als drei Standardabweichungen vom Mittelwert entfernt liegen. Das ist kein Beweis für einen Fehler, aber ein Hinweis, der eine Sichtprüfung auslöst. Ähnlich prüft man *Konsistenz* zwischen Dateien, etwa ob alle Fremdschlüssel in einer Tabelle in der anderen existieren: `comm -23 <(cut -d';' -f2 a.csv | sort -u) <(cut -d';' -f1 b.csv | sort -u)` zeigt Werte, die nur in der ersten Datei vorkommen.

Häufige Fehler: Erstens wirft man ungültige Zeilen still weg und verschleiert das Problem. Zweitens prüft man nur das Format, aber nicht die Plausibilität (ein gültig formatiertes Alter von 250). Drittens bricht man beim ersten Fehler ab, obwohl der Anwender alle Fehler auf einmal sehen möchte. Viertens verlässt man sich auf Regex für E-Mail-Adressen, die sich nur sehr aufwendig vollständig prüfen lassen; hier genügt eine grobe Prüfung, die endgültige Bestätigung erfolgt durch tatsächlichen Versand. Fünftens vergisst man Leerzeichen am Rand der Felder (`" Anna"`), die Vergleiche und Duplikatprüfungen verfälschen; sie gehören vor der Prüfung bereinigt.

```bash
# falsch: Fehlerhafte Zeilen verschwinden spurlos
awk -F';' 'NF == 4 && $3 != ""' eingabe.csv > ausgabe.csv

# richtig: gueltige und fehlerhafte Zeilen trennen, Anzahl melden
awk -F';' 'NF == 4 && $3 != "" { print > "gut.csv"; next } { print > "schlecht.csv" }' eingabe.csv
echo "Fehlerhaft: $(wc -l < schlecht.csv 2>/dev/null || echo 0)"
```

> **Merke**
>
> Datenqualität prüft Vollständigkeit, Gültigkeit, Eindeutigkeit, Konsistenz und Plausibilität. Fehlerhafte Datensätze in Quarantäne ablegen und berichten, nicht stillschweigend verwerfen. Alle Fehler sammeln, dann mit passendem Exit-Code enden, damit Folgeschritte stoppen können.

> **Praxisbeispiel**
>
> Vor jedem nächtlichen Import prüft ein Validierungsskript die gelieferte Datei. Bei mehr als einem Prozent fehlerhafter Zeilen bricht die Pipeline ab und schickt den Bericht an die Datenlieferanten, statt halbfertige Daten in die Produktivdatenbank zu laden.

## 5.10 Übungsaufgaben zu Modul 05

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit. Testdaten legen Sie am besten mit Here-Documents im Skript selbst an.

**Aufgabe 1.** Erklären Sie den Unterschied zwischen `grep`, `grep -E` und `grep -F`. Wann verwenden Sie welches, und warum gehört das Muster fast immer in Anführungszeichen?

**Aufgabe 2.** Warum liefert `sed 's/<.*>//'` auf `<a>Text<b>` ein überraschendes Ergebnis, und wie lässt es sich beheben? Nennen Sie zwei Varianten.

**Aufgabe 3.** Welche Rolle spielen `BEGIN`, `END`, `NR` und `NF` in awk? Wie würden Sie mit awk die Summe der dritten Spalte einer Datei berechnen, ohne die Kopfzeile mitzuzählen?

**Aufgabe 4.** Weshalb führt `jq ".[] | select(.name == \"$name\")"` zu Problemen, und welche Alternative ist richtig? Erklären Sie auch den Unterschied zwischen `jq '.x'` und `jq -r '.x'`.

**Aufgabe 5.** Schreiben Sie ein Skript `top-woerter.sh DATEI [N]`, das die N häufigsten Wörter (Standard 10) einer Textdatei ermittelt, unabhängig von Groß-/Kleinschreibung und ohne Satzzeichen. Die Ausgabe zeigt Anzahl und Wort.

**Aufgabe 6.** Schreiben Sie ein Skript `statuscodes.sh LOGDATEI`, das aus einem Webserver-Log im Combined-Log-Format für jede Statuscode-Klasse (2xx, 3xx, 4xx, 5xx) die Anzahl und den Prozentanteil ausgibt und die fünf häufigsten URLs mit Statuscode 404 listet.

**Aufgabe 7.** Schreiben Sie ein Skript `csv-zu-json.sh DATEI.csv`, das eine einfache Semikolon-CSV (mit Kopfzeile, ohne Anführungszeichen in den Feldern) in ein JSON-Array von Objekten umwandelt. Verwenden Sie `jq` (nicht Zeichenketten-Bastelei), bereinigen Sie CR und BOM, und wandeln Sie Spalten, die nur aus Ziffern bestehen, in Zahlen um.

**Aufgabe 8.** Schreiben Sie ein Skript `log-bericht.sh LOGDATEI`, das aus einem Anwendungslog (Format wie in 5.7: ISO-Zeitstempel, Level, Meldung) einen Bericht erzeugt: Anzahl je Level, die fünf häufigsten normalisierten Fehlermeldungen und die Zeitspanne mit den meisten Fehlern pro Minute. Der Exit-Code ist 1, wenn mindestens ein `ERROR` vorkommt, sonst 0.

## 5.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** `grep` nutzt Basic Regular Expressions: Zeichen wie `+`, `?`, `|` und `()` sind wörtlich, wirken erst mit Backslash als Operatoren. `grep -E` nutzt Extended Regular Expressions, in denen diese Zeichen direkt Operatoren sind; das ist für Alternativen und Gruppen bequemer. `grep -F` sucht den Text *wörtlich*, ohne jede Sonderbedeutung, ist am schnellsten und sicher bei Eingaben mit Sonderzeichen. Anführungszeichen verhindern, dass die Shell `*`, `?`, `[ ]`, `$` und Leerzeichen selbst interpretiert, bevor `grep` das Muster sieht.

**Lösung zu Aufgabe 2.** Der Quantor `*` ist gierig: `.*` passt auf möglichst viel, hier also von der ersten öffnenden bis zur letzten schließenden Klammer; das Ergebnis ist ein leerer String. Variante 1: negierte Zeichenklasse, `sed 's/<[^>]*>//g'` (entfernt `<a>` und `<b>` einzeln, ergibt `Text`). Variante 2: Ein nicht gieriger Quantor in einem Werkzeug mit PCRE, etwa `perl -pe 's/<.*?>//g'`. Mit POSIX-`sed` gibt es keine nicht gierigen Quantoren, daher ist Variante 1 die portable Lösung.

**Lösung zu Aufgabe 3.** `BEGIN` läuft einmal vor der ersten Zeile (Initialisierung, `FS` setzen), `END` einmal nach der letzten (Ergebnisse ausgeben). `NR` ist die laufende Zeilennummer, `NF` die Anzahl der Felder der aktuellen Zeile. Summe der dritten Spalte ohne Kopfzeile: `awk 'NR > 1 { s += $3 } END { print s }' datei`. Bei einem anderen Trennzeichen kommt `-F';'` hinzu.

**Lösung zu Aufgabe 4.** Der Wert aus der Shell wird in den Filtertext eingesetzt. Enthält er Anführungszeichen oder Backslashes, bricht der Filter oder, schlimmer, ein Angreifer kann den Filter verändern (Injektion). Richtig ist `jq --arg n "$name" '.[] | select(.name == $n)'`: Der Wert gelangt als Variable `$n` in jq und wird nie als Programmtext interpretiert. `jq '.x'` gibt eine Zeichenkette mit Anführungszeichen aus (gültiges JSON), `jq -r '.x'` die rohe Zeichenkette ohne Anführungszeichen, wie sie für Shell-Variablen und Textverarbeitung benötigt wird.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# top-woerter.sh DATEI [N]
export LC_ALL=C.UTF-8 2>/dev/null || true
datei="${1:-}"
n="${2:-10}"
[ -f "$datei" ] || { echo "Aufruf: $0 DATEI [N]" >&2; exit 2; }
[[ $n =~ ^[0-9]+$ ]] || { echo "N muss eine Zahl sein" >&2; exit 2; }

tr '[:upper:]' '[:lower:]' < "$datei" \
    | tr -cs '[:alpha:]' '\n' \
    | grep -v '^$' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -n "$n"
```

Die erste `tr`-Stufe schreibt alles klein, die zweite ersetzt jede Folge von Nicht-Buchstaben durch einen Zeilenumbruch (`-c` Komplement, `-s` Zusammenfassen), sodass ein Wort pro Zeile entsteht. `grep -v '^$'` entfernt leere Zeilen, der Rest ist die bekannte Zählkette. Hinweis: Umlaute werden nur dann als Buchstaben erkannt, wenn die Locale UTF-8 unterstützt und `tr` im Multibyte-Modus arbeitet (GNU-`tr` arbeitet bytebasiert, deshalb zerlegt es Umlaute unter Umständen; für deutsche Texte ist `awk` mit `tolower` oder ein Werkzeug mit Unicode-Unterstützung robuster).

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# statuscodes.sh LOGDATEI
export LC_ALL=C
log="${1:-}"
[ -f "$log" ] || { echo "Aufruf: $0 LOGDATEI" >&2; exit 2; }

echo "Statusklassen:"
awk '
    $9 ~ /^[1-5][0-9][0-9]$/ { klasse[substr($9,1,1) "xx"]++; n++ }
    END {
        for (k in klasse) printf "  %s %6d  %5.1f%%\n", k, klasse[k], 100 * klasse[k] / n
    }' "$log" | sort

echo "Top-5 URLs mit 404:"
awk '$9 == 404 { print $7 }' "$log" | sort | uniq -c | sort -rn | head -n 5
```

Die Klasse ergibt sich aus der ersten Ziffer des Statuscodes plus `xx`. Die Musterprüfung `$9 ~ /^[1-5][0-9][0-9]$/` ignoriert Zeilen mit unerwartetem Format, statt falsch zu zählen; wer das Format absichern will, ergänzt einen Zähler für verworfene Zeilen und gibt ihn aus. Prozente werden relativ zur Zahl gültiger Zeilen berechnet. Die 404-Auswertung ist die Standardkette.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# csv-zu-json.sh DATEI.csv
datei="${1:-}"
[ -f "$datei" ] || { echo "Aufruf: $0 DATEI.csv" >&2; exit 2; }
command -v jq >/dev/null || { echo "jq fehlt" >&2; exit 1; }

sed -e '1s/^\xef\xbb\xbf//' -e 's/\r$//' "$datei" \
    | jq -R -s '
        split("\n") | map(select(length > 0) | split(";"))
        | .[0] as $kopf
        | .[1:]
        | map( . as $zeile
               | [ range(0; $kopf | length) ]
               | map( { key: $kopf[.],
                        value: ( $zeile[.] // ""
                                 | if test("^-?[0-9]+$") then tonumber else . end ) } )
               | from_entries )'
```

`-R -s` lesen den gesamten Text als eine Zeichenkette. `split("\n")` zerlegt sie in Zeilen, leere Zeilen werden verworfen, dann wird jede Zeile am Semikolon geteilt. Die erste Zeile liefert die Spaltennamen (`$kopf`). Für jede Datenzeile wird über die Spaltenindizes ein Objekt gebaut; `from_entries` macht aus Schlüssel-Wert-Paaren ein Objekt. Die Bedingung `test("^-?[0-9]+$")` wandelt reine Ziffernfolgen mit `tonumber` um. Fehlende Felder am Zeilenende werden mit `// ""` zu leeren Zeichenketten. Beachten Sie die Grenze der Lösung: Felder in Anführungszeichen und Semikolons in Werten werden nicht unterstützt.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# log-bericht.sh LOGDATEI
export LC_ALL=C
log="${1:-}"
[ -f "$log" ] || { echo "Aufruf: $0 LOGDATEI" >&2; exit 2; }

echo "== Anzahl je Level =="
awk '{ n[$2]++ } END { for (l in n) printf "%-6s %d\n", l, n[l] }' "$log" | sort

echo "== Haeufigste Fehlermeldungen (normalisiert) =="
awk '$2 == "ERROR" { $1 = $2 = ""; print substr($0, 3) }' "$log" \
    | sed -E 's/[0-9]+/N/g' | sort | uniq -c | sort -rn | head -n 5

echo "== Fehler pro Minute (Spitze) =="
awk '$2 == "ERROR" { z[substr($1, 1, 16)]++ } END { for (m in z) print z[m], m }' "$log" \
    | sort -rn | head -n 1

fehler="$(awk '$2 == "ERROR" { n++ } END { print n + 0 }' "$log")"
(( fehler > 0 )) && exit 1
exit 0
```

Der Level steht im zweiten Feld. Zur Normalisierung der Meldungen werden Zeitstempel und Level aus der Zeile entfernt (die Zuweisung `$1 = $2 = ""` leert die Felder, `substr($0, 3)` entfernt die beiden verbleibenden Trennleerzeichen), danach ersetzt `sed` alle Zahlen durch `N`. Die Minute wird aus den ersten 16 Zeichen des ISO-Zeitstempels gebildet (`2026-10-30T08:01`). Der Exit-Code spiegelt wider, ob Fehler vorkamen. Dass `n + 0` aus einem leeren Wert die Zahl 0 macht, ist ein verbreiteter awk-Trick.
