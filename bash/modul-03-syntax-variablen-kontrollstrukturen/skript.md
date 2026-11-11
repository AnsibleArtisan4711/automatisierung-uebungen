# Modul 03 - Bash-Syntax, Variablen und Kontrollstrukturen

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (3.1 bis 3.9; 3.10 Übungsaufgaben, 3.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Als Richtwert für den Umfang gilt der höhere Wert von 10 h.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit             | Dauer (h) | Prio |
|:----|:------------------------|----------:|:----:|
| 3.1 | Shebang und Skriptstart |      1,00 |  1   |
| 3.2 | Variablen und Scope     |      1,00 |  1   |
| 3.3 | Quoting-Regeln          |      1,00 |  1   |
| 3.4 | Parameter Expansion     |      1,00 |  1   |
| 3.5 | Arrays                  |      1,00 |  1   |
| 3.6 | Tests und Conditions    |      1,00 |  1   |
| 3.7 | Schleifen               |      1,00 |  2   |
| 3.8 | Funktionen              |      1,00 |  2   |
| 3.9 | Argument Parsing        |      1,00 |  2   |
|     | **Summe (Details)**     |  **9,00** |      |

Jetzt wird aus Kommandozeilenarbeit echte Programmierung. Dieses Modul führt die Bausteine ein, aus denen jedes Bash-Skript besteht: der Skriptstart, Variablen und ihr Geltungsbereich, die oft unterschätzten Regeln des Quotings, die Erweiterung von Parametern, Arrays, Bedingungen, Schleifen, Funktionen und das Auswerten von Kommandozeilenoptionen. Das Wichtigste in diesem Modul ist das richtige Quoting; die meisten Fehler in Shell-Skripten gehen darauf zurück.

## 3.1 Shebang und Skriptstart

Ein Shellskript ist zunächst nur eine Textdatei mit Befehlen. Damit das Betriebssystem weiß, *welcher Interpreter* sie ausführen soll, beginnt sie mit der *Shebang*-Zeile (auch *Hashbang*, von `#` und `!`). Beim Start einer Datei liest der Kernel die ersten beiden Bytes; lauten sie `#!`, liest er den Rest der Zeile als Pfad zum Interpreter, optional mit einem Argument, und startet diesen Interpreter mit dem Skriptpfad als Parameter. Für die Shell selbst ist die Zeile ein gewöhnlicher Kommentar, weil sie mit `#` beginnt.

Zwei Schreibweisen sind verbreitet. `#!/bin/bash` nennt einen festen Pfad. Das ist verlässlich, wenn Bash garantiert dort liegt, scheitert aber auf Systemen, wo Bash woanders installiert ist (macOS mit Homebrew: `/opt/homebrew/bin/bash`, NixOS, manche BSD). `#!/usr/bin/env bash` ruft das Programm `env` auf, das `bash` im aktuellen `PATH` sucht. Das ist portabler und erlaubt es, eine neuere Bash zu nutzen, die vor der systemeigenen im PATH steht. Der Preis: Das Skript hängt von der PATH-Umgebung ab, was in Dienst-Kontexten mit minimalem PATH schiefgehen kann. Die Entscheidung gehört zur Betriebsvereinbarung eines Teams; in der Praxis überwiegt `env`.

Ein Skript kann auf drei Arten gestartet werden. Erstens direkt (`./skript.sh`): Voraussetzung ist das Ausführungsrecht (`chmod +x`), und die Shebang-Zeile wählt den Interpreter. Zweitens über den Interpreter (`bash skript.sh`): Das Ausführungsrecht ist nicht nötig, die Shebang-Zeile wird ignoriert. Drittens per `source skript.sh` (oder `. skript.sh`): Die Befehle laufen in der *aktuellen* Shell, nicht in einem Kindprozess; Variablen und Verzeichniswechsel bleiben erhalten. Das ist für Bibliotheken gedacht, nicht für eigenständige Programme.

```bash
#!/usr/bin/env bash
# start.sh - Grundgeruest fuer ein Skript
# Aufruf: ./start.sh [NAME]

name="${1:-Welt}"

echo "Skriptname   : $0"
echo "Argumentzahl : $#"
echo "Alle Argumente: $*"
echo "Prozess-ID   : $$"
echo "Hallo, $name!"

# Wurde das Skript ausgefuehrt oder gesourct?
if [ "${BASH_SOURCE[0]}" = "$0" ]; then
    echo "Modus: direkt ausgefuehrt"
else
    echo "Modus: gesourct"
fi

exit 0
```

Die Variable `$0` enthält den Aufrufnamen des Skripts, `$#` die Anzahl der Argumente, `$*` alle Argumente als ein String, `$$` die Prozess-ID. Die Erkennung von `source` nutzt den Unterschied, dass `BASH_SOURCE[0]` die laufende Datei benennt, während `$0` beim Sourcen den Namen der *aufrufenden* Shell trägt. Das Skript endet mit `exit 0`, dem Exit-Code für Erfolg. Fehlt `exit`, ist der Exit-Code der des *zuletzt ausgeführten Befehls*, was ein häufiger Anlass für überraschende Ergebnisse ist.

Probieren Sie nach dem Speichern drei Aufrufe: `./start.sh`, `bash start.sh Anna` und `source ./start.sh`. Beim ersten erhalten Sie „Permission denied“, solange das Ausführungsrecht fehlt; nach `chmod +x start.sh` geht es.

Typische Fehler: Die Shebang-Zeile muss in Zeile 1 stehen, ohne führende Leerzeile oder Leerzeichen, sonst wird sie ignoriert. Zeilenenden müssen LF sein (siehe Modul 1). Falsch: `#!/bin/sh` mit Bash-Syntax wie `[[ ]]` oder Arrays; das gelingt nur dort, wo `sh` auf Bash zeigt. Richtig: `#!/usr/bin/env bash`. Beachten Sie auch, dass bei `#!/usr/bin/env -S` (mehrere Argumente) die Unterstützung von System zu System variiert; Optionen wie `set -e` setzt man daher besser im Skript selbst.

> **Merke**
>
> Shebang in Zeile 1: `#!/usr/bin/env bash`. Ausführen braucht `chmod +x`; `bash datei` ignoriert die Shebang-Zeile; `source` läuft in der aktuellen Shell. Ein Skript sollte seinen Exit-Code bewusst setzen.

> **Praxisbeispiel**
>
> Ein Wartungsskript wird auf einem Mac entwickelt, wo `/bin/bash` die uralte Version 3.2 ist, und läuft auf Linux mit Bash 5. Verwendet es assoziative Arrays, scheitert es auf dem Mac. Dank `env` und einer per Homebrew installierten neuen Bash klappt es auf beiden Plattformen.

## 3.2 Variablen und Scope

Eine *Variable* ist ein benannter Speicherplatz für Text. Bash kennt im Grunde nur Zeichenketten; Zahlen sind Zeichenketten, die bei Bedarf als Zahl interpretiert werden. Zuweisung: `name="Anna"`. Wichtig ist, dass *kein Leerzeichen* um das Gleichheitszeichen steht. Schreibt man `name = "Anna"`, versucht Bash, ein Programm namens `name` mit den Argumenten `=` und `Anna` zu starten und meldet „command not found“. Lesen kann man den Wert mit einem vorangestellten Dollarzeichen (`$name`, sicherer in der Form `${name}`, wenn direkt Text folgt, etwa `${name}_alt`).

Variablennamen bestehen aus Buchstaben, Ziffern und Unterstrichen und beginnen nicht mit einer Ziffer. Üblich ist, Umgebungsvariablen und Konstanten in Großbuchstaben (`PATH`, `MAX_VERSUCHE`) und eigene lokale Variablen in Kleinbuchstaben zu schreiben. So verwechselt man eigene Variablen nicht mit denen der Umgebung, was eine häufige Fehlerquelle ist: Ein Skript, das `PATH` oder `HOME` überschreibt, bricht danach merkwürdig.

Der *Scope* (Geltungsbereich) einer Variablen ist standardmäßig *global innerhalb der Shell*: Eine in einer Funktion gesetzte Variable ist danach auch außerhalb sichtbar. Das schafft Seiteneffekte. Mit dem Schlüsselwort `local` (nur innerhalb von Funktionen) begrenzt man die Variable auf die Funktion und ihre Aufrufe. Mit `readonly` macht man eine Variable unveränderbar, ein Schutz für Konstanten. `declare` ist der allgemeine Befehl für Attribute: `declare -i` (Ganzzahl), `declare -r` (read-only), `declare -x` (exportiert), `declare -A` (assoziatives Array).

Dazu kommt der Unterschied zwischen *Shell-Variablen* und *Umgebungsvariablen*. Eine normale Variable existiert nur in der aktuellen Shell. Erst `export name` macht sie zur Umgebungsvariable, die an *Kindprozesse* vererbt wird. Die Vererbung geht nur nach unten: Ein Kindprozess kann die Variable der Eltern nicht ändern.

```bash
#!/usr/bin/env bash
# scope-demo.sh

readonly MAX_VERSUCHE=3          # Konstante
zaehler=0                        # globale Variable

zaehle() {
    local lokal="nur in der Funktion"
    zaehler=$((zaehler + 1))     # veraendert die globale Variable
    echo "in zaehle: lokal='$lokal' zaehler=$zaehler"
}

zaehle
zaehle
echo "nach den Aufrufen: zaehler=$zaehler, lokal='${lokal:-<nicht gesetzt>}'"

# Export und Kindprozesse
geheim="nur Shell"
export oeffentlich="fuer Kinder sichtbar"
bash -c 'echo "Kind sieht: geheim=${geheim:-<leer>} oeffentlich=${oeffentlich:-<leer>}"'

# Rechnen mit Ganzzahlen
declare -i summe=5
summe+=10
echo "summe=$summe"

# Konstante laesst sich nicht aendern
( MAX_VERSUCHE=5 ) 2>/dev/null || echo "MAX_VERSUCHE ist readonly"
```

In der Funktion `zaehle` ist `lokal` nur dort sichtbar, `zaehler` dagegen wird bewusst global verändert; der Ausdruck `$(( ... ))` ist die arithmetische Erweiterung für Ganzzahlen. Nach den Aufrufen ist `lokal` nicht mehr gesetzt, was die Erweiterung `${lokal:-<nicht gesetzt>}` anzeigt. Beim Kindprozess sieht man die Wirkung von `export`: Nur `oeffentlich` kommt an. Mit `declare -i` rechnet `summe+=10` numerisch (15) statt Texte aneinanderzuhängen. Die letzte Zeile zeigt, dass `readonly` eine Änderung verhindert. Der Änderungsversuch läuft bewusst in einer Subshell `( ... )`, denn eine Zuweisung an eine schreibgeschützte Variable beendet in einer nicht interaktiven Shell das ganze Skript; so wird nur die Subshell beendet, und `||` fängt den Fehlercode auf.

Typische Fehler: Erstens Leerzeichen bei der Zuweisung (siehe oben). Zweitens vergessene `local`-Deklarationen, wodurch Funktionen versehentlich Variablen des Aufrufers überschreiben, zum Beispiel einen Schleifenzähler `i`. Drittens die Annahme, dass `export` rückwirkend wirkt: Sie gilt erst für *danach* gestartete Prozesse. Viertens Variablen in Pipelines: Wer in der Schleife `echo "a b" | read x y` ausführt, findet `x` danach leer, denn jeder Teil einer Pipeline läuft in einer Subshell, und deren Variablen gehen verloren (siehe Modul 4).

> **Merke**
>
> Zuweisung ohne Leerzeichen um das Gleichheitszeichen. In Funktionen `local` verwenden, Konstanten `readonly` machen, nur gewollte Variablen `export`ieren. Kindprozesse erben, ändern aber nie die Variablen der Eltern.

> **Praxisbeispiel**
>
> CI-Systeme übergeben Konfiguration (Branch-Name, Build-Nummer, Zugangsdaten) als Umgebungsvariablen an Skripte. Ein Skript liest sie direkt (`${BUILD_NUMBER:?nicht gesetzt}`) und bricht mit klarer Meldung ab, wenn sie fehlen, statt mit leeren Werten weiterzuarbeiten.

## 3.3 Quoting-Regeln

*Quoting* (Anführungszeichen setzen) ist der wichtigste und am häufigsten missverstandene Teil der Shell-Syntax. Um es zu verstehen, muss man wissen, wie die Shell eine Zeile verarbeitet. Sie zerlegt sie zuerst in Wörter, ersetzt dann Variablen, Befehle und Muster (die *Expansionen*), und trennt *danach* die Ergebnisse unquotierter Expansionen nochmals an Leerzeichen (*Word Splitting*) und ersetzt Muster mit Platzhaltern durch Dateinamen (*Globbing*). Quoting schaltet einzelne dieser Schritte gezielt aus.

Es gibt drei Mechanismen. *Einfache Anführungszeichen* (`'...'`) schützen den gesamten Inhalt vollständig: Nichts wird ersetzt, nicht einmal ein Dollarzeichen. Darin kann kein weiteres einfaches Anführungszeichen vorkommen. *Doppelte Anführungszeichen* (`"..."`) schützen vor Word Splitting und Globbing, erlauben aber weiterhin die Ersetzung von `$variable`, `$(befehl)`, `$((rechnung))` und die Verwendung von Backslash-Escapes. Der *Backslash* (`\`) schützt genau das nachfolgende Zeichen, etwa `\$` für ein Dollarzeichen oder `\"` für ein Anführungszeichen im doppelt gequoteten Text.

Die Faustregel für die Praxis lautet: *Setze jede Variablenexpansion und jede Kommandosubstitution in doppelte Anführungszeichen*, außer du willst ausdrücklich Word Splitting oder Globbing. Ohne Quotes zerfällt der Wert einer Variable, die Leerzeichen enthält, in mehrere Argumente; enthält er einen Stern, wird er durch Dateinamen ersetzt; ist er leer, verschwindet das Argument ganz. ShellCheck meldet diesen Fehler als SC2086.

> **Ergänzung aus der Nachrecherche**
>
> Die Wiki-Seite von ShellCheck zu SC2086 (abgerufen) nennt als Beispiele `echo $1` und `for i in $*`, die bei Leerzeichen, Zeilenumbrüchen oder Glob-Zeichen in den Werten brechen. Die empfohlene Form ist `echo "$1"` und `for i in "$@"`. Ausnahme: gewollte Aufteilung, für die man ein Array nutzen sollte, damit kein String gesplittet werden muss.

```bash
#!/usr/bin/env bash
# quoting-demo.sh
pfad="/tmp/mein Verzeichnis"
muster="*.txt"
name="Anna"

echo '--- einfache Quotes: nichts wird ersetzt'
echo '$name und $(date) bleiben Text'

echo '--- doppelte Quotes: Variablen werden ersetzt'
echo "Hallo $name, heute ist $(date +%F)"
echo "Preis: \$5 und ein \"Zitat\""

echo '--- Word Splitting ohne Quotes'
mkdir -p "$pfad"
printf '[%s]\n' $pfad            # zerfaellt in zwei Argumente
echo '--- mit Quotes'
printf '[%s]\n' "$pfad"          # bleibt ein Argument

echo '--- Globbing'
touch "$pfad/a.txt" "$pfad/b.txt"
cd "$pfad" || exit 1
echo $muster                     # Shell expandiert den Stern
echo "$muster"                   # Stern bleibt Text

echo '--- $@ vs $*'
set -- "eins zwei" drei
printf 'ohne Quotes : '; printf '[%s]' $@;   echo
printf 'mit  Quotes : '; printf '[%s]' "$@"; echo

echo '--- leere Variable'
leer=""
[ -n $leer ] && echo "falsch: wird WAHR (Argument verschwindet)"
[ -n "$leer" ] || echo "richtig: leer erkannt"

cd /; rm -rf -- "$pfad"
```

Gehen wir die Abschnitte durch. Einfache Quotes verhindern jede Ersetzung; man benutzt sie für feste Texte und für Muster (zum Beispiel für `grep` und `find`), die die Shell nicht anfassen soll. Doppelte Quotes erlauben `$name` und `$(date ...)`; Dollarzeichen und Anführungszeichen schützt man mit Backslash. Die beiden `printf`-Zeilen machen Word Splitting sichtbar: Ohne Quotes zerfällt `/tmp/mein Verzeichnis` in zwei Argumente, mit Quotes bleibt es eins. Der Unterschied zwischen `$@` und `"$@"` ist grundlegend: Nur `"$@"` reicht jedes Argument unverändert als einzelnes Wort weiter, auch wenn es Leerzeichen enthält, was für alle Funktionen und Wrapper-Skripte Standard ist. `"$*"` fügt dagegen alle zu einem einzigen Wort zusammen. Der letzte Block zeigt die Tücke der leeren Variable: Ohne Quotes wird aus `[ -n $leer ]` nach dem Verschwinden des Arguments der Test `[ -n ]`, der wahr ist, weil `-n` dann als Text mit einem Zeichen gilt.

Ein Sonderfall sind Ausdrücke mit `[[ ... ]]`: Dort findet kein Word Splitting und kein Globbing statt, weshalb Variablen auch ohne Quotes sicher sind. Rechts von `==` ist ein unquotiertes Muster aber ein Glob-Muster, quotiert ein wörtlicher Text. Ebenfalls nicht gequotet werden muss die rechte Seite einer einfachen Zuweisung (`x=$y`), es schadet aber nicht.

Falsch und richtig im Vergleich:

```text
# falsch                       # richtig
rm $datei                      rm -- "$datei"
cp $quelle $ziel               cp -- "$quelle" "$ziel"
for f in $(ls)                 for f in ./*
if [ $a = $b ]                 if [ "$a" = "$b" ]
wrapper() { prog $@; }         wrapper() { prog "$@"; }
```

> **Merke**
>
> Einfache Quotes: alles wörtlich. Doppelte Quotes: Variablen und Befehle werden ersetzt, aber kein Word Splitting oder Globbing. Faustregel: *Immer* `"$variable"` und `"$(befehl)"`, `"$@"` für Argumentlisten. In `[[ ]]` ist Quoting optional, aber bei Mustern bedeutsam.

> **Praxisbeispiel**
>
> Der bekannte Vorfall, bei dem ein Installationsskript beim Aufräumen `rm -rf $STEAMROOT/*` ausführte und bei leerer Variable das gesamte Home-Verzeichnis löschte, ist ein Mahnmal für fehlende Absicherung. Zwei Maßnahmen verhindern ihn: Quoting und `${VAR:?}`, das bei leerer Variable abbricht.

## 3.4 Parameter Expansion

Die *Parameter Expansion* bietet in der Form `${...}` viel mehr als bloße Ersetzung. Mit ihr lassen sich Standardwerte setzen, Texte kürzen und ersetzen und Variablen prüfen, ohne externe Programme wie `sed` oder `cut` zu starten. Das macht Skripte schneller und robuster.

Fünf Gruppen sind für den Alltag zentral. Erstens *Standardwerte*: `${var:-wert}` liefert `wert`, wenn `var` nicht gesetzt oder leer ist; `${var:=wert}` tut dasselbe und weist den Wert zusätzlich zu; `${var:?meldung}` bricht mit der Meldung ab, wenn `var` fehlt; `${var:+wert}` liefert `wert` nur, wenn `var` gesetzt und nicht leer ist (sonst nichts). Ohne den Doppelpunkt (`${var-wert}`) wird nur auf „nicht gesetzt“ geprüft, nicht auf „leer“. Zweitens die *Länge*: `${#var}` ergibt die Zeichenzahl. Drittens *Teilzeichenketten*: `${var:start:länge}` schneidet aus. Viertens *Entfernen am Anfang oder Ende*: `${var#muster}` entfernt den kürzesten passenden Anfang, `${var##muster}` den längsten; `${var%muster}` und `${var%%muster}` tun das am Ende. Fünftens *Ersetzen*: `${var/muster/neu}` ersetzt das erste Vorkommen, `${var//muster/neu}` alle. Dazu kommen die Groß-/Kleinschreibung (`${var^^}` und `${var,,}` ab Bash 4) und die indirekte Auflösung (`${!name}`).

```bash
#!/usr/bin/env bash
# expansion-demo.sh
datei="/home/anna/berichte/jahresbericht.tar.gz"

echo "Laenge        : ${#datei}"
echo "Basisname     : ${datei##*/}"            # alles bis zum letzten /
echo "Verzeichnis   : ${datei%/*}"             # alles ab dem letzten /
echo "Endung (kurz) : ${datei##*.}"            # gz
echo "ohne .gz      : ${datei%.gz}"
nur="${datei##*/}"
echo "Name vor erstem Punkt: ${nur%%.*}"       # jahresbericht

echo "Teilstring    : ${nur:0:5}"              # jahre
echo "Ersetzt (1x)  : ${datei/berichte/archiv}"
echo "Ersetzt (alle): ${datei//a/A}"
text="Hallo Welt"
echo "GROSS: ${text^^}  klein: ${text,,}"

# Standardwerte
unset ziel
echo "ziel mit Default: ${ziel:-/tmp/ausgabe}"
echo "ziel noch ungesetzt: '${ziel-}'"
: "${ziel:=/var/tmp}"                           # zuweisen, falls leer
echo "ziel jetzt: $ziel"
modus=""
echo "Option: ${modus:+--verbose}"               # leer, da modus leer
modus=1
echo "Option: ${modus:+--verbose}"               # --verbose

# Pflichtvariable
echo "${BENUTZER:?Variable BENUTZER muss gesetzt sein}" 2>&1 | head -n 1
```

Die Muster-Entfernung ist der wichtigste Trick. `${datei##*/}` entfernt den *längsten* Anfang, der auf `*/` passt, also alles bis zum letzten Schrägstrich, und liefert damit den Dateinamen (wie `basename`). `${datei%/*}` entfernt den *kürzesten* Schluss, der auf `/*` passt, und liefert den Verzeichnisteil (wie `dirname`). Die Merkregel: Die Raute steht auf der Tastatur links vom Dollar, wirkt also am *Anfang*; das Prozentzeichen steht rechts und wirkt am *Ende*. Der Doppelzeichen-Form (`##`, `%%`) entspricht dem „gierigen“ Muster. Die Zeile mit `: "${ziel:=/var/tmp}"` nutzt den Doppelpunkt-Befehl, der nichts tut, nur um die Zuweisung als Nebeneffekt auszulösen. Das Muster `${modus:+--verbose}` ist praktisch, um optionale Argumente einzufügen. Die letzte Zeile zeigt `${var:?}`: Ist die Variable leer, bricht Bash ab und gibt die Meldung auf stderr aus (bei nicht interaktiven Shells beendet es das Skript).

Typische Fehler: Erstens verwechselt man `:-` (Standardwert nutzen) mit `:=` (zuweisen); wer später `$var` liest und `:-` verwendet hat, findet die Variable noch immer leer. Zweitens vergisst man, dass `${var:-}` im Zusammenspiel mit `set -u` (Fehler bei ungesetzten Variablen, siehe Modul 6) die elegante Art ist, eine möglicherweise ungesetzte Variable zu lesen. Drittens führen Substring-Operationen bei Unicode-Zeichen je nach Locale zu unerwarteten Ergebnissen, weil Bash *Zeichen*, nicht Bytes zählt, wenn die Locale es erlaubt.

```bash
# falsch: externes Programm fuer eine einfache Aufgabe
name=$(echo "$datei" | sed 's/.*\///')

# richtig: Parameter Expansion, kein Prozess noetig
name="${datei##*/}"
```

> **Merke**
>
> `${v:-x}` Default, `${v:=x}` zuweisen, `${v:?msg}` Pflicht, `${v:+x}` nur wenn gesetzt, `${#v}` Länge, `${v#p}` / `${v%p}` Präfix/Suffix entfernen (doppelt = gierig), `${v/a/b}` ersetzen.

> **Praxisbeispiel**
>
> Ein Build-Skript benennt Artefakte um: aus `app-1.4.2.tar.gz` wird mit `${f%.tar.gz}` der Basisname, der als Verzeichnis dient. Tausende Dateien lassen sich so ohne einen einzigen externen Prozess bearbeiten, was gegenüber `sed` in einer Schleife spürbar schneller ist.

## 3.5 Arrays

Ein *Array* ist eine Variable, die mehrere Werte unter einem Namen hält. Es löst ein Problem, das die Shell sonst hat: Eine Liste von Dingen in einen einzelnen String zu packen und später wieder zu zerlegen, bricht zwangsläufig bei Werten mit Leerzeichen. Arrays behalten jedes Element als eigene Einheit.

Bash kennt zwei Arten. *Indizierte Arrays* werden mit Zahlen ab 0 angesprochen: `farben=(rot "dunkel blau" gruen)`. *Assoziative Arrays* (ab Bash 4) funktionieren mit Textschlüsseln und müssen mit `declare -A` angelegt werden: `declare -A port=([http]=80 [https]=443)`. Wichtige Zugriffe: `${farben[1]}` liefert ein Element, `"${farben[@]}"` alle Elemente als getrennte Wörter (die wichtigste Form), `${#farben[@]}` die Anzahl, `${!farben[@]}` die Indizes beziehungsweise Schlüssel, `farben+=(lila)` hängt an, und `unset 'farben[1]'` löscht ein Element. Mit `"${farben[@]:1:2}"` schneidet man eine Teilliste aus.

Der Unterschied zwischen `"${arr[@]}"` und `"${arr[*]}"` entspricht dem von `"$@"` und `"$*"`: Die erste Form liefert jedes Element als eigenes Wort, die zweite verbindet alle zu einem String (getrennt durch das erste Zeichen von `IFS`). Für Schleifen und Befehlsargumente verwendet man fast immer die `@`-Form.

```bash
#!/usr/bin/env bash
# arrays-demo.sh (benoetigt Bash >= 4 fuer assoziative Arrays)

farben=(rot "dunkel blau" gruen)
farben+=(gelb)

echo "Anzahl     : ${#farben[@]}"
echo "Element 1  : ${farben[1]}"
echo "Letztes    : ${farben[-1]}"

echo "--- Schleife ueber alle Elemente"
for f in "${farben[@]}"; do
    printf '  [%s]\n' "$f"
done

echo "--- Indizes"
for i in "${!farben[@]}"; do
    echo "  $i -> ${farben[$i]}"
done

echo "--- Array als Befehlsargumente (sicher, ohne eval)"
optionen=(-l -a -h)
ls "${optionen[@]}" /tmp | head -n 3

echo "--- Datei zeilenweise einlesen"
printf 'eins\nzwei woerter\ndrei\n' > /tmp/arr-demo.txt
mapfile -t zeilen < /tmp/arr-demo.txt
echo "gelesen: ${#zeilen[@]} Zeilen, die zweite: ${zeilen[1]}"
rm -f /tmp/arr-demo.txt

echo "--- Assoziatives Array"
declare -A port=([http]=80 [https]=443 [ssh]=22)
port[dns]=53
for dienst in "${!port[@]}"; do
    echo "  $dienst = ${port[$dienst]}"
done | sort
echo "https hat Port ${port[https]}"
[ -v 'port[ftp]' ] || echo "ftp ist nicht eingetragen"
```

Die Initialisierung mit Klammern legt vier Elemente an; das zweite enthält ein Leerzeichen und bleibt dank Quoting ein Element. Negative Indizes zählen ab Ende (ab Bash 4.3). Die Schleife mit `"${farben[@]}"` liefert exakt vier Durchläufe, nicht fünf, weil „dunkel blau“ nicht zerfällt. `"${!farben[@]}"` gibt die Indizes aus. Die Variable `optionen` zeigt das *sichere Muster, Befehle zu bauen*: Statt einen String mit Optionen zusammenzusetzen und unquotiert zu verwenden (oder gar `eval`), sammelt man Argumente in einem Array und expandiert es mit `"${optionen[@]}"`. `mapfile -t` liest die Zeilen einer Datei direkt in ein Array (`-t` entfernt die Zeilenenden). Das assoziative Array bildet Dienste auf Ports ab; die Reihenfolge der Schlüssel ist *nicht garantiert*, weshalb die Ausgabe mit `sort` sortiert wird. Der Test `[ -v 'port[ftp]' ]` prüft, ob ein Schlüssel existiert.

Typische Fehler: Der Zugriff `$farben` ohne Index liefert nur das erste Element (Index 0), was oft nicht beabsichtigt ist. Ein anderer Fehler ist die Übergabe an Funktionen: Arrays lassen sich nicht direkt als Argument übergeben; man gibt `"${arr[@]}"` mit und baut es in der Funktion neu auf, oder nutzt einen Namensverweis (`local -n ref=$1`, ab Bash 4.3). Und Arrays gibt es nicht in POSIX-`sh`; in `dash` oder Busybox-`ash` schlägt die Syntax fehl.

```bash
# falsch: Liste als String, zerbricht bei Leerzeichen
dateien="a.txt b b.txt"
for f in $dateien; do echo "[$f]"; done     # 4 Durchlaeufe

# richtig: Array
dateien=("a.txt" "b b.txt")
for f in "${dateien[@]}"; do echo "[$f]"; done   # 2 Durchlaeufe
```

> **Merke**
>
> Arrays halten Listen ohne String-Splitting: `arr=(a "b c")`, `"${arr[@]}"`, `${#arr[@]}`, `arr+=(x)`. Assoziative Arrays brauchen `declare -A` und Bash ≥ 4. Befehle mit variablen Optionen als Array bauen statt als String.

> **Praxisbeispiel**
>
> Ein Deployment-Skript sammelt Zielserver in einem Array und ruft für jeden `ssh` auf. Ein Array von `rsync`-Optionen, zusammengesetzt je nach Umgebung (`opts+=(--dry-run)`), verhindert die gefährliche Alternative, einen Befehlsstring zusammenzusetzen und mit `eval` auszuführen.

## 3.6 Tests und Conditions

Entscheidungen in Bash beruhen auf einem einfachen Prinzip: Jedes Kommando liefert einen *Exit-Code*, und `if` wertet diesen aus. Code 0 gilt als „wahr“ (Erfolg), jeder andere Wert als „falsch“. Das ist das Gegenteil vieler Programmiersprachen und ein häufiger Stolperstein. Die Bedingung in `if` ist also kein Ausdruck, sondern ein *Befehl*: `if grep -q muster datei; then ...` führt `grep` aus und prüft dessen Ergebnis.

Für Vergleiche stehen mehrere Konstrukte bereit, die man unterscheiden muss. `test` und das gleichwertige `[ ... ]` sind der klassische, in POSIX definierte Weg; `[` ist ein Befehl (Builtin), deshalb gehören Leerzeichen innen und um die Klammern. `[[ ... ]]` ist ein Bash-Schlüsselwort mit besserer Syntax: Kein Word Splitting und kein Globbing der Variablen, Mustervergleich (`==` mit Glob-Muster), reguläre Ausdrücke (`=~`) und die logischen Operatoren `&&` und `||` innerhalb des Ausdrucks. `(( ... ))` ist die *arithmetische* Auswertung mit C-ähnlicher Syntax (`(( a > b ))`, `(( n % 2 == 0 ))`); der Rückgabewert ist wahr, wenn das Ergebnis ungleich null ist.

Die wichtigsten Tests: Für Dateien `-e` (existiert), `-f` (reguläre Datei), `-d` (Verzeichnis), `-L` (Symlink), `-r`, `-w`, `-x` (lesbar, schreibbar, ausführbar), `-s` (nicht leer). Für Zeichenketten `-z` (leer), `-n` (nicht leer), `=` beziehungsweise `==` und `!=`. Für Zahlen innerhalb von `[ ]` die Textoperatoren `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`; in `(( ))` dagegen die gewohnten `==`, `<` und `>`. Das ist ein wichtiger Unterschied: Innerhalb von `[ ]` und `[[ ]]` vergleicht `<` Zeichenketten, nicht Zahlen.

```bash
#!/usr/bin/env bash
# tests-demo.sh
datei="${1:-/etc/passwd}"
zahl="${2:-7}"
text="Bericht-2026.pdf"

# 1. Dateitests
if [ -f "$datei" ] && [ -r "$datei" ]; then
    echo "$datei ist eine lesbare Datei"
elif [ -d "$datei" ]; then
    echo "$datei ist ein Verzeichnis"
else
    echo "$datei existiert nicht oder ist nicht lesbar" >&2
fi

# 2. Zahlenvergleich (arithmetisch)
if (( zahl % 2 == 0 )); then
    echo "$zahl ist gerade"
else
    echo "$zahl ist ungerade"
fi
[ "$zahl" -gt 5 ] && echo "$zahl ist groesser als 5"

# 3. Mustervergleich in [[ ]]
if [[ $text == *.pdf ]]; then
    echo "PDF-Datei"
fi

# 4. Regulaerer Ausdruck mit Capture-Gruppen
if [[ $text =~ ^([A-Za-z]+)-([0-9]{4})\.pdf$ ]]; then
    echo "Name=${BASH_REMATCH[1]} Jahr=${BASH_REMATCH[2]}"
fi

# 5. Befehl als Bedingung
if grep -q '^root:' /etc/passwd; then
    echo "root-Eintrag gefunden"
fi

# 6. Mehrfachauswahl mit case
read -r -t 0.1 antwort <<< "ja"
case "${antwort,,}" in
    j|ja|y|yes) echo "Zustimmung" ;;
    n|nein|no)  echo "Ablehnung" ;;
    *)          echo "unklare Antwort" ;;
esac
```

Der erste Block kombiniert zwei Tests mit `&&`: Beide Befehle müssen erfolgreich sein. `elif` ergänzt Alternativen, `else` ist der Rest. Der zweite Block zeigt die Arithmetik: Innerhalb von `(( ))` braucht man kein `$` vor Variablennamen. Der dritte Block nutzt den Glob-Mustervergleich, der vierte einen regulären Ausdruck (Extended Regex) mit den Treffergruppen in `BASH_REMATCH`; der Ausdruck steht dabei *ohne* Anführungszeichen, denn sonst würde er wörtlich verglichen. Der fünfte Block zeigt die Idee, dass *jeder Befehl* eine Bedingung sein kann, und `-q` unterdrückt die Ausgabe. Zuletzt wählt `case` zwischen mehreren Mustern; die Variable wird mit `${antwort,,}` auf Kleinbuchstaben gebracht, und die Alternativen trennt der senkrechte Strich. Jeder Zweig endet mit `;;`.

Typische Fehler beim Testen: Erstens fehlende Leerzeichen. Falsch: `[$a=$b]`; richtig: `[ "$a" = "$b" ]`. Zweitens unquotierte Variablen in `[ ]` (siehe 3.3). Drittens die Verwechslung der Operatoren: `[ "$a" -eq "$b" ]` ist ein *numerischer* Vergleich und scheitert mit „integer expression expected“, wenn der Wert kein Ganzzahltext ist; `=` vergleicht Texte (`"10" = "010"` ist falsch, `10 -eq 010` ist wahr, denn ... `010` wird hier als Dezimalzahl 10 gelesen). Viertens das gefürchtete `[ $a == $b ]` ohne Quotes bei leerem `a`. Fünftens die logische Verkettung `a && b || c`, die *nicht* wie `if-else` wirkt: Schlägt `b` fehl, wird auch `c` ausgeführt. Für echte Verzweigungen verwendet man `if`.

```bash
# falsch: c laeuft auch, wenn b fehlschlaegt
[ -f "$f" ] && echo "da" || echo "weg"    # bei echo-Fehler waere "weg" auch gedruckt

# richtig bei echten Verzweigungen
if [ -f "$f" ]; then echo "da"; else echo "weg"; fi
```

> **Merke**
>
> `if` wertet Exit-Codes aus (0 = wahr). Für Bash-Skripte: `[[ ]]` für Texte, Muster und Regex, `(( ))` für Zahlen, `[ ]` nur für POSIX-Portabilität. Bei `[ ]` alles quoten und auf Leerzeichen achten. Für Mehrfachauswahl `case`.

> **Praxisbeispiel**
>
> Ein Deployment-Skript prüft zu Beginn Vorbedingungen: `[[ -n ${DEPLOY_ENV:-} ]]`, `command -v kubectl`, `[ -r "$KONFIG" ]`. Fehlt etwas, endet es sofort mit einer verständlichen Meldung, statt später mit einem kryptischen Fehler abzubrechen. Dieses Muster nennt man *Guard Clauses*.

## 3.7 Schleifen

Schleifen wiederholen Befehle. Bash bietet drei Grundformen. Die `for`-Schleife durchläuft eine Liste von Werten: `for variable in wert1 wert2 ...; do ...; done`. Die Liste kann aus Literalen, Glob-Mustern (`*.txt`), Brace-Expansion (`{1..5}`) oder einem Array (`"${arr[@]}"`) bestehen. Daneben gibt es eine C-ähnliche Form: `for (( i=0; i<10; i++ )); do ...; done`. Die `while`-Schleife läuft, solange ein Befehl *erfolgreich* ist (Exit-Code 0), die `until`-Schleife, solange er *fehlschlägt*. Mit `break` verlässt man eine Schleife, mit `continue` springt man zum nächsten Durchlauf.

Besonders wichtig ist das sichere Einlesen von Dateien. Das Standardmuster lautet `while IFS= read -r zeile; do ...; done < datei`. Dabei hat jedes Teil einen Zweck. `read` liest eine Zeile von stdin. Die Option `-r` verhindert, dass Backslashes als Escapes interpretiert werden. `IFS=` (nur für diesen Befehl gesetzt) entfernt das Abschneiden von führenden und nachfolgenden Leerzeichen. Die Umleitung `< datei` steht *hinter* dem `done` und versorgt die gesamte Schleife mit der Datei. Die Variable `IFS` (*Internal Field Separator*) legt fest, an welchen Zeichen Bash Wörter trennt; der Standard ist Leerzeichen, Tabulator und Zeilenumbruch.

```bash
#!/usr/bin/env bash
# schleifen-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
printf 'anna;42;Wien\nben;17;Graz\n  clara ;99;Linz\n' > personen.csv
touch "a b.txt" c.txt

echo "--- for ueber Muster (Leerzeichen-sicher)"
for f in *.txt; do
    [ -e "$f" ] || continue
    printf 'Datei: [%s]\n' "$f"
done

echo "--- for mit Zaehler"
for (( i = 1; i <= 3; i++ )); do
    echo "Durchlauf $i"
done
for n in {1..3}; do echo -n "$n "; done; echo

echo "--- Datei zeilenweise lesen und aufteilen"
while IFS=';' read -r name alter ort; do
    echo "Name='$name' Alter=$alter Ort=$ort"
done < personen.csv

echo "--- while mit Zaehler und break"
n=0
while true; do
    n=$((n + 1))
    (( n == 2 )) && continue
    (( n > 4 )) && break
    echo "n=$n"
done

echo "--- until: warten, bis etwas eintritt"
versuch=0
until [ -e fertig.flag ] || (( versuch >= 3 )); do
    versuch=$((versuch + 1))
    echo "Versuch $versuch"
    (( versuch == 2 )) && touch fertig.flag
    sleep 0.1
done

echo "--- Schleife mit Befehlsausgabe (ohne Subshell-Problem)"
summe=0
while read -r zahl; do
    summe=$((summe + zahl))
done < <(printf '10\n20\n30\n')
echo "Summe=$summe"

cd / && rm -rf -- "$verz"
```

Beim Muster `*.txt` bleibt der Dateiname `a b.txt` dank Quotes in `"$f"` intakt; die Zeile `[ -e "$f" ] || continue` schützt vor dem Fall, dass das Muster keinen Treffer hat. In der Lese-Schleife wird `IFS` nur für das `read` auf Semikolon gesetzt, wodurch jede Zeile direkt in die drei Variablen zerlegt wird; die Schleife liest die Datei am Ende per `< personen.csv`. Die `until`-Schleife zeigt ein Warte-Muster mit Obergrenze, das *Endlosschleifen vermeidet*. Der letzte Block ist für ein verbreitetes Problem wichtig: `printf ... | while read ...` würde die Schleife in einer Subshell ausführen, und die Variable `summe` wäre danach wieder 0. Mit der *Prozesssubstitution* `< <(befehl)` läuft die Schleife in der aktuellen Shell, und die Variable bleibt erhalten (Hintergründe in Modul 4).

Häufige Fehler: Erstens `for f in $(ls)`, das bei Leerzeichen und Sonderzeichen zerbricht und gegen einfachere Globs nichts gewinnt. Zweitens das Weglassen von `-r` und `IFS=` in `read`, wodurch Backslashes und Einrückung zerstört werden. Drittens `while read` ohne Umleitung: Die Schleife wartet dann auf Eingaben von der Tastatur und scheint „zu hängen“. Viertens, in einer Schleife über `ssh` oder andere stdin-lesende Programme: `ssh` liest die restliche Eingabe der Schleife auf und beendet diese vorzeitig. Abhilfe: `ssh -n` oder `< /dev/null`.

```bash
# falsch: zerlegt Dateinamen, ignoriert Sonderzeichen
for f in $(find . -name '*.txt'); do wc -l $f; done

# richtig: nullterminiert und mit Quotes
find . -name '*.txt' -print0 | while IFS= read -r -d '' f; do
    wc -l -- "$f"
done
```

Die Variante mit `read -d ''` liest bis zum Nullbyte statt bis zum Zeilenende und verarbeitet damit jeden zulässigen Dateinamen korrekt. Beachten Sie, dass die Schleife hier hinter einer Pipe in einer Subshell läuft; Variablenänderungen darin sind danach nicht sichtbar.

> **Merke**
>
> Dateien zeilenweise: `while IFS= read -r zeile; do ...; done < datei`. Über Dateien: Glob statt `ls`. Warteschleifen immer mit Obergrenze. Pipelines in Schleifen laufen in Subshells; mit `< <(...)` bleiben Variablen erhalten.

> **Praxisbeispiel**
>
> Ein Import-Skript liest eine CSV-Datei mit Benutzern und legt für jede Zeile ein Konto an. Mit `IFS=';' read -r` und Prüfungen pro Zeile (leere Felder, ungültige Zeichen) bricht es nicht bei der ersten schlechten Zeile ab, sondern protokolliert sie und macht weiter.

## 3.8 Funktionen

Eine *Funktion* ist ein benannter Block von Befehlen, den man beliebig oft aufrufen kann. Sie macht Skripte lesbar, vermeidet Wiederholung und lässt sich getrennt testen. Definition: `name() { befehle; }` (oder mit dem Schlüsselwort `function name { ... }`, was in POSIX nicht erlaubt ist). Der Aufruf erfolgt wie bei einem Befehl: `name arg1 arg2`, *ohne* Klammern und Kommas.

Innerhalb der Funktion stehen die Argumente als `$1`, `$2`, ... sowie `$#`, `"$@"` zur Verfügung, völlig getrennt von den Argumenten des Skripts. Eine Funktion hat zwei Arten von Ergebnissen, und man muss sie sauber unterscheiden. Der *Rückgabewert* (`return n`) ist ein Exit-Code von 0 bis 255 und signalisiert Erfolg oder Fehler, nicht Daten. Für Daten gibt die Funktion Text auf stdout aus, den der Aufrufer mit `$(...)` einfängt; oder sie setzt eine globale Variable beziehungsweise (ab Bash 4.3) arbeitet über einen Namensverweis mit `local -n`. Die Standardregel: Status per `return`, Daten per stdout, Diagnose per stderr.

```bash
#!/usr/bin/env bash
# funktionen-demo.sh

log() {                                 # Diagnose auf stderr
    printf '[%s] %s\n' "$(date +%T)" "$*" >&2
}

ist_zahl() {                            # Status per return
    [[ ${1:-} =~ ^-?[0-9]+$ ]]          # Ergebnis des Tests = Rueckgabewert
}

addiere() {                             # Daten per stdout
    local summe=0 x
    for x in "$@"; do
        ist_zahl "$x" || { log "keine Zahl: $x"; return 1; }
        summe=$((summe + x))
    done
    echo "$summe"
}

# Ergebnis einfangen und Status pruefen
if ergebnis="$(addiere 10 20 30)"; then
    echo "Summe: $ergebnis"
fi

if ! ergebnis="$(addiere 5 abc)"; then
    echo "Berechnung fehlgeschlagen (Status $?)"
fi

# Ausgabeparameter ueber Namensverweis (Bash >= 4.3)
teile() {
    local -n ziel=$1                    # ziel ist ein Alias fuer die uebergebene Variable
    local text="$2"
    IFS=',' read -r -a ziel <<< "$text"
}
teile werte "a,b,c"
echo "Elemente: ${#werte[@]}, zweites: ${werte[1]}"

# Rekursion: Fakultaet
fakultaet() {
    local n=$1
    (( n <= 1 )) && { echo 1; return; }
    local vorher
    vorher="$(fakultaet $((n - 1)))"
    echo $((n * vorher))
}
echo "5! = $(fakultaet 5)"
```

`log` schreibt auf stderr, damit der Funktionsaufruf in `$(...)` nicht durch Meldungen verunreinigt wird; sonst enthielte die eingefangene Variable Daten und Logzeilen gemischt. Die Funktion `ist_zahl` besteht nur aus einem Test; der Rückgabewert einer Funktion ist standardmäßig der Exit-Code des *letzten Befehls*, also des `[[ ]]`-Ausdrucks, weshalb kein ausdrückliches `return` nötig ist. In `addiere` werden die Argumente mit `"$@"` durchlaufen, ein Fehler wird per `return 1` gemeldet, das Ergebnis per `echo`. Beim Aufruf fängt die Zuweisung in der `if`-Bedingung sowohl die Ausgabe als auch den Status ein. `teile` zeigt einen Namensverweis: `local -n` lässt `ziel` auf die Variable zeigen, deren *Name* übergeben wurde, so dass die Funktion das Array des Aufrufers füllen kann. `local` darf nur innerhalb einer Funktion stehen. Die Rekursion zeigt, dass Funktionen sich selbst aufrufen dürfen; jede Ebene hat eigene lokale Variablen.

Wichtig ist die Falle bei `local` mit Zuweisung: `local x="$(befehl)"` liefert als Exit-Code den von `local` (immer 0), nicht den des Befehls; Fehler gehen unter. Deshalb deklariert man getrennt (`local x; x="$(befehl)"`) und prüft dann den Status, wie in der Rekursion oben. ShellCheck warnt davor als SC2155.

```bash
# falsch: Fehler von befehl geht verloren
f() { local ergebnis="$(befehl_der_fehlschlaegt)"; echo "$ergebnis"; }

# richtig: getrennt deklarieren und zuweisen
f() { local ergebnis; ergebnis="$(befehl_der_fehlschlaegt)" || return 1; echo "$ergebnis"; }
```

Weitere Stolperfallen: Eine Funktion mit dem Namen eines vorhandenen Befehls überdeckt diesen (siehe Command Lookup in Modul 1); `command name` umgeht die Funktion. Funktionen müssen vor ihrem ersten *Aufruf* (nicht Definition im Text) definiert sein; üblich ist, alle Funktionen oben und den Hauptablauf in einer Funktion `main` zu bündeln, die als letzte Zeile mit `main "$@"` aufgerufen wird. Und Funktionsnamen sollten Bindestriche und Punkte meiden, weil sie in manchen Shells nicht erlaubt sind.

> **Merke**
>
> Status per `return` (0–255), Daten per stdout und `$(...)`, Diagnose per stderr. Variablen mit `local` halten, Deklaration und Zuweisung bei Befehlsausgaben trennen. Hauptlogik in `main "$@"`.

> **Praxisbeispiel**
>
> In einer Bibliotheksdatei `lib/log.sh` liegen `log_info`, `log_warn`, `log_error`. Alle Skripte des Teams binden sie mit `source` ein und erhalten so einheitliche Logzeilen mit Zeitstempel. Änderungen am Format erfolgen an einer einzigen Stelle (siehe Modul 6).

## 3.9 Argument Parsing

Ein brauchbares Skript lässt sich über Argumente steuern. Einfache Fälle (ein oder zwei feste Argumente) löst man mit `$1`, `$2` und prüft deren Anzahl mit `$#`. Sobald Optionen wie `-v`, `-o datei` oder `-n 5` hinzukommen, lohnt sich ein Parser, denn das manuelle Auswerten wird schnell unübersichtlich und fehleranfällig. Dafür gibt es das Builtin `getopts`, das POSIX-konform ist. Es verarbeitet *kurze* Optionen (ein Buchstabe). Lange Optionen (`--help`, `--ausgabe=datei`) kennt `getopts` nicht; sie parst man mit einer `while`-/`case`-Schleife per Hand oder mit dem externen `getopt` (GNU-Variante).

Die Syntax von `getopts` lautet `getopts optstring variable`. Der *optstring* listet die erlaubten Buchstaben; ein nachgestellter Doppelpunkt (`o:`) bedeutet „diese Option braucht einen Wert“, der in der Variable `OPTARG` landet. Beginnt der optstring mit einem Doppelpunkt (`:vho:`), schaltet man in den stillen Modus: Bei Fehlern setzt `getopts` die Variable auf `?` (unbekannte Option) oder `:` (fehlender Wert) und lässt dem Skript die Meldung. `OPTIND` zählt, wie viele Argumente bereits verarbeitet wurden; nach der Schleife entfernt man diese mit `shift $((OPTIND - 1))`, sodass `"$@"` nur noch die übrigen (Positions-)Argumente enthält.

```bash
#!/usr/bin/env bash
# kopiere.sh - Beispiel fuer getopts und Argumentpruefung
set -u

usage() {
    cat <<EOF
Aufruf: ${0##*/} [-v] [-n ANZAHL] [-o AUSGABE] [-h] QUELLE...

  -v          ausfuehrliche Ausgabe
  -n ANZAHL   maximal ANZAHL Dateien verarbeiten (Standard: 10)
  -o AUSGABE  Zielverzeichnis (Pflicht)
  -h          diese Hilfe
EOF
}

verbose=0
anzahl=10
ausgabe=""

while getopts ':vn:o:h' opt; do
    case "$opt" in
        v) verbose=1 ;;
        n) anzahl="$OPTARG" ;;
        o) ausgabe="$OPTARG" ;;
        h) usage; exit 0 ;;
        :) echo "Option -$OPTARG braucht einen Wert" >&2; usage >&2; exit 2 ;;
        \?) echo "Unbekannte Option: -$OPTARG" >&2; usage >&2; exit 2 ;;
    esac
done
shift $((OPTIND - 1))

# Pflichtparameter und Werte validieren
[[ -n $ausgabe ]]            || { echo "Fehler: -o fehlt" >&2; usage >&2; exit 2; }
[[ $anzahl =~ ^[0-9]+$ ]]    || { echo "Fehler: -n erwartet eine Zahl" >&2; exit 2; }
(( $# >= 1 ))                || { echo "Fehler: keine QUELLE angegeben" >&2; usage >&2; exit 2; }

mkdir -p -- "$ausgabe"
zaehler=0
for quelle in "$@"; do
    (( zaehler >= anzahl )) && { echo "Limit erreicht" >&2; break; }
    if [ -f "$quelle" ]; then
        cp -- "$quelle" "$ausgabe/"
        zaehler=$((zaehler + 1))
        (( verbose )) && echo "kopiert: $quelle"
    else
        echo "uebersprungen (keine Datei): $quelle" >&2
    fi
done
echo "$zaehler Datei(en) kopiert nach $ausgabe"
```

Die Funktion `usage` gibt über ein Here-Document (`<<EOF`) mehrzeiligen Text aus; `${0##*/}` zeigt den Skriptnamen ohne Pfad. Die `getopts`-Schleife läuft, bis alle Optionen verbraucht sind; die Variable `opt` enthält jeweils den Buchstaben. Im Fall `n` und `o` nimmt das Skript den Wert aus `OPTARG`. Die Zweige `:` und `\?` fangen Fehlbedienung im stillen Modus ab. Nach der Schleife entfernt `shift` die Optionen. Danach folgt die *Validierung*: Pflicht-Option vorhanden? Anzahl wirklich eine Zahl? Mindestens eine Quelle? Alle Fehler enden mit Exit-Code 2 und Meldung auf stderr; das ist die Konvention für „falsche Benutzung“. Der Hauptteil verarbeitet `"$@"` als die verbleibende Liste von Quellen.

Die Validierung ist kein Beiwerk: Jeder Wert, der von außen kommt (Argument, Umgebungsvariable, Datei), ist unvertrauenswürdig, bis er geprüft ist (siehe Modul 8). Auch Optionen, die mit Bindestrich beginnen, können Dateinamen sein: Mit der Konvention `--` (Ende der Optionen) kann der Aufrufer sagen, dass alles Folgende Argumente sind.

Häufige Fehler: Erstens das Vergessen von `shift $((OPTIND - 1))`, wodurch `"$@"` noch die Optionen enthält. Zweitens Optionen *nach* den Positionsargumenten (`kopiere.sh datei -v`): `getopts` hört beim ersten Nicht-Options-Argument auf, `-v` bleibt unbeachtet. Das ist Standardverhalten und muss dokumentiert werden. Drittens verwechselt man `OPTARG` mit der Option selbst. Viertens muss `OPTIND` auf 1 zurückgesetzt werden (`OPTIND=1`), wenn `getopts` in einem Skript mehrfach, etwa in einer Funktion, verwendet wird.

```bash
# falsch: handgestricktes Parsing ohne Fehlerbehandlung
verbose=$1; ausgabe=$2     # was, wenn -v fehlt? Reihenfolge ist starr

# richtig (kleine Alternative fuer lange Optionen)
while (( $# > 0 )); do
    case "$1" in
        -v|--verbose) verbose=1; shift ;;
        -o|--ausgabe) ausgabe="${2:?--ausgabe braucht einen Wert}"; shift 2 ;;
        --) shift; break ;;
        -*) echo "Unbekannte Option: $1" >&2; exit 2 ;;
        *)  break ;;
    esac
done
```

Das zweite Muster behandelt auch lange Optionen: Die Schleife prüft das erste Argument per `case`, `shift` entfernt es (bei Optionen mit Wert `shift 2`), `--` beendet die Optionsverarbeitung, und das erste Nicht-Options-Argument beendet die Schleife mit `break`.

> **Merke**
>
> Kurzoptionen mit `getopts` (`:` für Werte, führender `:` für stillen Modus), danach `shift $((OPTIND-1))`. Immer `usage`-Text, Pflichtparameter prüfen, Werte validieren, Fehler auf stderr mit Code 2.

> **Praxisbeispiel**
>
> Interne Werkzeuge wie `deploy.sh -e staging -v` oder `backup.sh --dry-run` werden von Dutzenden Kolleginnen und Kollegen aufgerufen. Ein klarer `-h`-Text und eine strenge Validierung verhindern, dass jemand versehentlich ohne Umgebungsangabe gegen Produktion deployt.

## 3.10 Übungsaufgaben zu Modul 03

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Was gibt `printf '[%s]\n' $pfad` aus, wenn `pfad="/tmp/mein Ordner"` ist, und was, wenn man `"$pfad"` schreibt? Erklären Sie den Unterschied mit den Begriffen Word Splitting und Quoting.

**Aufgabe 2.** Wie lauten die Ergebnisse von `${datei##*/}`, `${datei%.*}` und `${datei:-standard}`, wenn `datei="/var/log/app.log"`? Welcher Ausdruck liefert die Dateiendung?

**Aufgabe 3.** Erklären Sie den Unterschied zwischen `[ ]`, `[[ ]]` und `(( ))`. Wann verwenden Sie welches Konstrukt?

**Aufgabe 4.** Warum liefert `echo "a b" | read x y; echo "$x"` eine leere Zeile? Nennen Sie zwei Lösungen.

**Aufgabe 5.** Schreiben Sie eine Funktion `ist_gerade`, die per Rückgabewert meldet, ob ihr Argument eine gerade Zahl ist (mit Prüfung, dass überhaupt eine Zahl übergeben wurde), und verwenden Sie sie in einer Schleife über 1 bis 10.

**Aufgabe 6.** Schreiben Sie ein Skript `wortzaehler.sh`, das eine Datei (Argument) zeilenweise liest und ein assoziatives Array verwendet, um zu zählen, wie oft jedes Wort vorkommt (Groß-/Kleinschreibung ignorieren). Am Ende werden die Wörter mit ihren Anzahlen sortiert ausgegeben.

**Aufgabe 7.** Schreiben Sie ein Skript `umbenennen.sh` mit den Optionen `-p PRAEFIX` (Pflicht) und `-n` (Trockenlauf), das alle Dateien eines Verzeichnisses (Positionsargument) mit dem Präfix versieht. Dateien mit Leerzeichen im Namen müssen funktionieren. Bei `-n` gibt es nur aus, was passieren würde.

**Aufgabe 8.** Schreiben Sie ein Skript `warte-auf.sh HOST PORT [-t SEKUNDEN]`, das in einer Schleife alle 1 Sekunde prüft, ob ein TCP-Port erreichbar ist (Hinweis: `(exec 3<>/dev/tcp/$host/$port) 2>/dev/null`), und nach dem Timeout (Standard 10) mit Fehlercode 1 endet. Argumente sind zu validieren, `-h` zeigt eine Hilfe.

## 3.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Ohne Quotes entstehen zwei Zeilen, `[/tmp/mein]` und `[Ordner]`, weil Bash den Wert der Variable nach der Expansion an Leerzeichen in zwei Wörter zerlegt (Word Splitting) und `printf` das Format für jedes Argument wiederholt. Mit `"$pfad"` bleibt der Wert ein einziges Argument, die Ausgabe ist `[/tmp/mein Ordner]`. Doppelte Anführungszeichen unterdrücken Word Splitting und Globbing, erlauben aber die Variablenersetzung.

**Lösung zu Aufgabe 2.** `${datei##*/}` ergibt `app.log` (längsten Anfang bis zum letzten Schrägstrich entfernen). `${datei%.*}` ergibt `/var/log/app` (kürzestes Ende ab dem letzten Punkt entfernen). `${datei:-standard}` ergibt `/var/log/app.log`, weil die Variable gesetzt und nicht leer ist; der Standardwert greift nur bei leerer oder fehlender Variable. Die Endung liefert `${datei##*.}` (`log`).

**Lösung zu Aufgabe 3.** `[ ]` ist der POSIX-konforme Befehl `test`: portabel, aber mit Fallstricken (Quoting zwingend, `<` und `>` sind Umleitungen, `-a`/`-o` sind veraltet). `[[ ]]` ist ein Bash-Schlüsselwort: kein Word Splitting, Mustervergleich, Regex mit `=~`, `&&` und `||` im Ausdruck. `(( ))` ist arithmetische Auswertung mit C-Syntax und Vergleichsoperatoren für Zahlen. Verwenden Sie in Bash-Skripten `[[ ]]` für Texte und Dateien, `(( ))` für Zahlen und `[ ]` nur, wenn das Skript POSIX-konform bleiben muss.

**Lösung zu Aufgabe 4.** Jeder Teil einer Pipeline läuft in einer eigenen Subshell. `read` setzt `x` und `y` nur in der Subshell, die mit der Pipeline endet; die aufrufende Shell sieht davon nichts. Lösung 1: Here-String statt Pipe, `read x y <<< "a b"`. Lösung 2: Prozesssubstitution, `read x y < <(echo "a b")`. (Eine dritte Möglichkeit in Bash ist `shopt -s lastpipe`, das in nicht interaktiven Skripten den letzten Pipeline-Teil in der aktuellen Shell ausführt.)

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# gerade.sh
ist_gerade() {
    [[ ${1:-} =~ ^-?[0-9]+$ ]] || return 2   # keine Zahl: Status 2
    (( $1 % 2 == 0 ))                        # 0 = gerade, 1 = ungerade
}

for n in {1..10}; do
    ist_gerade "$n"
    case $? in
        0) echo "$n ist gerade" ;;
        1) echo "$n ist ungerade" ;;
        *) echo "$n ist keine Zahl" >&2 ;;
    esac
done
```

Die Funktion nutzt drei Status: 0 für gerade, 1 für ungerade, 2 für fehlerhafte Eingabe. Das ist ein bewusster Einsatz von `return` als Statuskanal. Der letzte Befehl der Funktion (`(( ... ))`) liefert den Status 0, wenn der Ausdruck wahr ist, sonst 1. Die `case`-Auswertung liest `$?` sofort nach dem Aufruf.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# wortzaehler.sh DATEI
[ "$#" -eq 1 ] && [ -f "$1" ] || { echo "Aufruf: $0 DATEI" >&2; exit 2; }

declare -A anzahl
while IFS= read -r zeile; do
    zeile="${zeile,,}"                       # klein schreiben
    zeile="${zeile//[^[:alpha:]]/ }"         # alles ausser Buchstaben -> Leerzeichen
    for wort in $zeile; do                   # hier ist Word Splitting gewollt
        anzahl["$wort"]=$(( ${anzahl["$wort"]:-0} + 1 ))
    done
done < "$1"

for wort in "${!anzahl[@]}"; do
    printf '%5d %s\n' "${anzahl[$wort]}" "$wort"
done | sort -rn
```

Das Skript prüft zuerst die Aufrufargumente. Die Zeile wird mit `${zeile,,}` in Kleinbuchstaben umgewandelt, und `${zeile//[^[:alpha:]]/ }` ersetzt alle Nicht-Buchstaben durch Leerzeichen. Die `for`-Schleife über `$zeile` ohne Quotes nutzt hier *absichtlich* das Word Splitting; an anderer Stelle wäre das ein Fehler. Der Zähler wird mit dem Standardwert `:-0` erhöht, falls das Wort noch nicht im Array steht. Ausgegeben wird sortiert nach Häufigkeit. Hinweis: Ein Dateiname wie `-n` würde bei `< "$1"` nicht stören, da es sich um eine Umleitung handelt.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# umbenennen.sh -p PRAEFIX [-n] VERZEICHNIS
praefix=""
trocken=0
while getopts ':p:nh' opt; do
    case "$opt" in
        p) praefix="$OPTARG" ;;
        n) trocken=1 ;;
        h) echo "Aufruf: $0 -p PRAEFIX [-n] VERZEICHNIS"; exit 0 ;;
        :) echo "Option -$OPTARG braucht einen Wert" >&2; exit 2 ;;
        *) echo "Unbekannte Option -$OPTARG" >&2; exit 2 ;;
    esac
done
shift $((OPTIND - 1))

[[ -n $praefix ]] || { echo "Fehler: -p ist Pflicht" >&2; exit 2; }
[[ $praefix != */* ]] || { echo "Fehler: Praefix darf kein / enthalten" >&2; exit 2; }
[ "$#" -eq 1 ] && [ -d "$1" ] || { echo "Fehler: VERZEICHNIS angeben" >&2; exit 2; }

verz="${1%/}"
for pfad in "$verz"/*; do
    [ -f "$pfad" ] || continue
    name="${pfad##*/}"
    neu="$verz/$praefix$name"
    if (( trocken )); then
        echo "wuerde umbenennen: $pfad -> $neu"
    else
        mv -n -- "$pfad" "$neu"
    fi
done
```

Die Optionen werden mit `getopts` gelesen, danach wird validiert: ein Präfix mit Schrägstrich könnte Dateien aus dem Verzeichnis hinausschieben. Die Schleife über `"$verz"/*` (Stern außerhalb der Quotes) liefert jeden Dateinamen als ein Wort; `[ -f ... ] || continue` überspringt Unterverzeichnisse und den Fall ohne Treffer. `mv -n` überschreibt nie bestehende Dateien. Im Trockenlauf wird nur ausgegeben.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# warte-auf.sh HOST PORT [-t SEKUNDEN]
usage() { echo "Aufruf: ${0##*/} HOST PORT [-t SEKUNDEN]" >&2; }

timeout_s=10
args=()
while (( $# > 0 )); do
    case "$1" in
        -h) usage; exit 0 ;;
        -t) timeout_s="${2:-}"; shift 2 || { usage; exit 2; } ;;
        -*) echo "Unbekannte Option: $1" >&2; usage; exit 2 ;;
        *)  args+=("$1"); shift ;;
    esac
done

(( ${#args[@]} == 2 )) || { usage; exit 2; }
host="${args[0]}"; port="${args[1]}"
[[ $port =~ ^[0-9]+$ ]] && (( port >= 1 && port <= 65535 )) || { echo "Ungueltiger Port" >&2; exit 2; }
[[ $timeout_s =~ ^[0-9]+$ ]] || { echo "Ungueltiges Timeout" >&2; exit 2; }

ende=$(( SECONDS + timeout_s ))
until (exec 3<>"/dev/tcp/$host/$port") 2>/dev/null; do
    if (( SECONDS >= ende )); then
        echo "Timeout: $host:$port nicht erreichbar" >&2
        exit 1
    fi
    sleep 1
done
echo "$host:$port ist erreichbar"
```

Optionen und Positionsargumente dürfen hier in beliebiger Reihenfolge stehen, weil die Schleife per Hand parst und Nicht-Optionen in einem Array sammelt. Die Variable `SECONDS` zählt automatisch die Sekunden seit Skriptstart und dient als einfache Uhr. Der TCP-Test nutzt das Bash-eigene Pseudogerät `/dev/tcp`; die Klammern erzeugen eine Subshell, deren Deskriptor 3 beim Ende der Subshell automatisch geschlossen wird. Dieses Feature ist Bash-spezifisch und in manchen Distributionen deaktiviert (Debian-Pakete haben es aktiv, andere Builds nicht).
