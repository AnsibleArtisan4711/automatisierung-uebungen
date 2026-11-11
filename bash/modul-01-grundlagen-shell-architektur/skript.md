# Modul 01 - Bash Grundlagen und Shell-Architektur

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (1.1 bis 1.8; 1.9 Übungsaufgaben, 1.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 8 h laut Blatt „Lehrplan in Modulen“, 10 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (8 Einheiten à 1,25 h). Als Richtwert für den Umfang gilt der höhere Wert von 10 h.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit                   | Dauer (h) | Prio |
|:----|:------------------------------|----------:|:----:|
| 1.1 | Shell, Terminal und TTY       |      1,25 |  1   |
| 1.2 | Unix-Philosophie              |      1,25 |  1   |
| 1.3 | Interaktive Nutzung vs Skript |      1,25 |  1   |
| 1.4 | Bash-Versionen und POSIX      |      1,25 |  1   |
| 1.5 | Command Lookup                |      1,25 |  1   |
| 1.6 | Manpages und Hilfe            |      1,25 |  1   |
| 1.7 | Shell-Initialisierung         |      1,25 |  2   |
| 1.8 | Praxisumgebung                |      1,25 |  2   |
|     | **Summe**                     | **10,00** |      |

In diesem ersten Modul legen wir das Fundament für alles Weitere. Bevor wir Skripte schreiben, müssen wir verstehen, was eine Shell eigentlich ist, wie sie mit dem Terminal und dem Betriebssystem zusammenspielt und nach welchen Regeln sie Kommandos findet und ausführt. Wer diese Architektur kennt, versteht später auch Fehlermeldungen, die auf den ersten Blick rätselhaft wirken, etwa warum ein Skript im Terminal funktioniert, aber im Cron-Job scheitert.

## 1.1 Shell, Terminal und TTY

Eine *Shell* ist ein Programm, das Kommandos entgegennimmt, sie interpretiert und das Betriebssystem bittet, entsprechende Programme zu starten. Sie ist die Vermittlerin zwischen Mensch und Betriebssystemkern (Kernel). Der Name kommt vom englischen Wort für „Schale“: Die Shell legt sich wie eine Schale um den Kern. Bash steht für „Bourne-Again Shell“, ein Wortspiel auf die ältere Bourne-Shell (`sh`) von Stephen Bourne. Bash wurde im Rahmen des GNU-Projekts entwickelt und ist auf den meisten Linux-Systemen die Standard-Shell für interaktive Anwender.

Die Shell ist dabei zweierlei zugleich: ein *Kommandozeileninterpreter*, der eingetippte Zeilen ausführt, und eine *Programmiersprache* mit Variablen, Bedingungen, Schleifen und Funktionen. Genau diese Doppelrolle macht Bash so nützlich. Alles, was Sie interaktiv tippen, können Sie später unverändert in eine Datei schreiben und damit automatisieren.

Oft werden die Begriffe Shell und Terminal vermischt, sie bezeichnen aber verschiedene Dinge. Das *Terminal* (genauer: Terminalemulator, etwa GNOME Terminal, Windows Terminal oder xterm) ist ein Fenster, das Tastatureingaben entgegennimmt und Text darstellt. Die Shell läuft *in* diesem Terminal als gewöhnlicher Prozess. Historisch war ein Terminal ein eigenes Gerät, eine Tastatur mit Bildschirm oder Drucker, das über eine serielle Leitung an einen Großrechner angeschlossen war. Unter Unix heißen solche Geräte *TTY* (von „teletypewriter“, Fernschreiber). Heutige Terminalfenster simulieren dieses Gerät in Software; man spricht dann von einem *Pseudo-Terminal* (PTY). Der Kernel stellt dafür ein Paar aus Master- und Slave-Seite bereit: Der Terminalemulator hält den Master, die Shell hängt am Slave, der unter einem Namen wie `/dev/pts/0` sichtbar ist.

Dass dieser Unterschied praktisch relevant ist, zeigt der Befehl `tty`. Er gibt den Namen des Terminalgeräts aus, an dem die Standardeingabe hängt. Ebenso wichtig ist die Frage, ob ein Programm *interaktiv* läuft, also mit einem Menschen am anderen Ende. Das prüft man in Bash mit dem Test `[ -t 0 ]` (ist Dateideskriptor 0, die Standardeingabe, ein Terminal?). Viele Werkzeuge verhalten sich je nach Antwort anders: `ls` färbt die Ausgabe nur im Terminal, und `git` öffnet nur dort einen Pager.

Das folgende Beispiel zeigt eine kleine Diagnose. Es gibt aus, welche Shell läuft, an welchem Terminalgerät sie hängt, ob es sich um eine Login-Shell handelt und ob Ein- und Ausgabe mit einem Terminal verbunden sind.

```bash
#!/usr/bin/env bash
# diagnose.sh - zeigt, in welcher Umgebung dieses Skript laeuft

echo "Bash-Version : $BASH_VERSION"
echo "Prozess-ID   : $$"
echo "Terminal     : $(tty 2>/dev/null || echo 'keines')"

if [ -t 0 ]; then
    echo "stdin  ist ein Terminal"
else
    echo "stdin  ist KEIN Terminal (Pipe oder Datei)"
fi

if [ -t 1 ]; then
    echo "stdout ist ein Terminal"
else
    echo "stdout ist KEIN Terminal"
fi

if shopt -q login_shell; then
    echo "Login-Shell  : ja"
else
    echo "Login-Shell  : nein"
fi
```

Die erste Zeile, die *Shebang*-Zeile, sagt dem Kernel, mit welchem Interpreter die Datei ausgeführt werden soll; sie wird in Modul 3 ausführlich behandelt. `$BASH_VERSION` ist eine von Bash selbst gesetzte Variable, `$$` enthält die Prozess-ID der laufenden Shell. In der Zeile mit `tty` fängt die *Kommandosubstitution* `$(...)` die Ausgabe des Befehls ab und setzt sie in den Text ein; `2>/dev/null` verwirft Fehlermeldungen, und `||` liefert den Ersatztext, falls `tty` scheitert. Die `if`-Blöcke prüfen mit `-t`, ob der jeweilige Dateideskriptor an ein Terminal angeschlossen ist. Der Befehl `shopt -q login_shell` fragt still ab, ob die Shell als Login-Shell gestartet wurde.

Führen Sie das Skript auf drei Arten aus und vergleichen Sie die Ausgabe: direkt im Terminal, mit `./diagnose.sh | cat` (dann ist stdout kein Terminal mehr) und mit `./diagnose.sh < /dev/null` (dann ist stdin kein Terminal). Diese kleine Übung macht greifbar, dass „Terminal“ eine Eigenschaft der Verbindung ist, nicht des Programms.

> **Merke**
>
> Terminal, Shell und Kommando sind drei getrennte Dinge: Das Terminal zeigt Text an und nimmt Tasten entgegen, die Shell interpretiert Zeilen, das Kommando (ein eigener Prozess) erledigt die Arbeit. Ob ein Programm *interaktiv* läuft, entscheidet sich daran, ob seine Ein- und Ausgabe an ein TTY angeschlossen sind.

Eine weitere Unterscheidung betrifft den Startmodus der Shell. Eine *Login-Shell* entsteht, wenn Sie sich anmelden (an der Textkonsole, per SSH) oder wenn Bash mit der Option `--login` gestartet wird. Eine *Nicht-Login-Shell* entsteht typischerweise, wenn Sie in einer grafischen Oberfläche ein neues Terminalfenster öffnen oder in einer laufenden Shell `bash` eintippen. Dazu kommt die Unterscheidung *interaktiv* oder *nicht interaktiv*: Ein Skript läuft in einer nicht interaktiven Shell, in der es weder Prompt noch Alias-Erweiterung noch Jobsteuerung gibt. Welche Konfigurationsdateien jeweils gelesen werden, behandelt Lerneinheit 1.7.

Typische Fehler entstehen, wenn man die Rollen vermischt. Ein Klassiker: Jemand schreibt `read antwort` in ein Skript, das nachts per Cron läuft. Es gibt dort kein Terminal, also keine Eingabe, und das Skript hängt oder bricht ab. Falsch ist, interaktive Eingaben ungeprüft vorauszusetzen; richtig ist, zuerst mit `[ -t 0 ]` zu prüfen oder Werte per Argument bzw. Umgebungsvariable zu übergeben.

```bash
# falsch: haengt im Cron-Job, weil es keine Eingabe gibt
read -r -p "Wirklich loeschen? [j/N] " antwort

# richtig: nur fragen, wenn ein Mensch am Terminal sitzt
if [ -t 0 ]; then
    read -r -p "Wirklich loeschen? [j/N] " antwort
else
    antwort="n"   # sichere Voreinstellung ohne Terminal
fi
```

Das zweite Beispiel setzt für den nicht interaktiven Fall eine *sichere Voreinstellung*: Ohne ausdrückliches „ja“ wird nichts gelöscht. Dieses Prinzip, im Zweifel das ungefährlichere Verhalten zu wählen, zieht sich durch das gesamte Skriptdesign.

> **Praxisbeispiel**
>
> In CI-Systemen wie GitHub Actions oder GitLab CI läuft jeder Schritt ohne Terminal. Skripte, die dort Farben ausgeben oder Rückfragen stellen, erzeugen kaputte Logs oder hängende Jobs. Wer früh mit `[ -t 1 ]` entscheidet, ob Farbcodes erlaubt sind, spart sich später lange Fehlersuche.

## 1.2 Unix-Philosophie

Die Unix-Philosophie ist weniger ein Regelwerk als eine Haltung, die sich aus den Erfahrungen der Entwickler in den 1970er-Jahren bei den Bell Labs entwickelt hat. Doug McIlroy, der Erfinder der Pipes, fasste sie sinngemäß so zusammen: Schreibe Programme, die eine Sache tun und diese gut. Schreibe Programme, die zusammenarbeiten. Schreibe Programme, die Textströme verarbeiten, denn das ist eine universelle Schnittstelle.

Daraus ergibt sich ein Werkzeugkasten aus vielen kleinen, spezialisierten Programmen. `sort` sortiert, `uniq` entfernt oder zählt Duplikate, `wc` zählt, `grep` filtert. Keines dieser Programme weiß etwas über die anderen, und doch lassen sie sich beliebig kombinieren. Die Shell ist der Klebstoff, der diese Bausteine verbindet. Der Vorteil gegenüber einem großen Programm mit allen Funktionen: Jedes Werkzeug ist leicht zu verstehen, zu testen und zu ersetzen, und neue Aufgaben lösen Sie oft, ohne eine einzige Zeile „richtigen“ Code zu schreiben.

Der Schlüssel dazu ist die einheitliche Schnittstelle. Jeder Prozess erhält beim Start drei Datenkanäle: Standardeingabe (*stdin*, Dateideskriptor 0), Standardausgabe (*stdout*, 1) und Standardfehlerausgabe (*stderr*, 2). Ein Programm liest nach Möglichkeit von stdin, schreibt Ergebnisse nach stdout und Fehlermeldungen nach stderr. Die Shell kann diese Kanäle umleiten oder mit dem Pipe-Zeichen `|` zwischen Prozessen verketten. Die Details dazu folgen in Modul 2 (Grundlagen) und Modul 4 (vertieft).

Betrachten wir ein typisches Beispiel: Welche zehn Befehle benutze ich am häufigsten? Dafür gibt es kein Spezialprogramm, aber die Antwort lässt sich aus fünf Bausteinen zusammensetzen.

```bash
# interaktiv in der Shell eingeben (die History gibt es nur dort):
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -n 10
```

Gelesen wird die Kette von links nach rechts. `history` gibt die Liste der früheren Eingaben aus, jede Zeile beginnt mit einer laufenden Nummer. `awk '{print $2}'` gibt von jeder Zeile nur das zweite Feld aus, also das erste Wort des Befehls. `sort` ordnet die Wörter alphabetisch, damit gleiche Einträge nebeneinander stehen. Das ist nötig, weil `uniq -c` nur *direkt aufeinanderfolgende* gleiche Zeilen zusammenfasst und zählt. Das zweite `sort -rn` sortiert nach der Zahl (`-n` numerisch) in absteigender Reihenfolge (`-r` reverse), und `head -n 10` behält die ersten zehn Zeilen. Jeder Baustein ist trivial; die Leistung entsteht durch die Kombination.

> **Merke**
>
> Textströme sind die gemeinsame Sprache der Unix-Werkzeuge. Wer Ausgaben seiner Skripte so gestaltet, dass das nächste Werkzeug sie lesen kann (eine Zeile pro Datensatz, klare Trennzeichen, Ergebnisse auf stdout, Meldungen auf stderr), macht sie wiederverwendbar.

Aus der Philosophie folgen Regeln für eigene Skripte. Ein Skript sollte genau eine Aufgabe haben und im Namen ausdrücken. Es sollte von stdin lesen können, wenn keine Datei angegeben ist. Es sollte Erfolg oder Misserfolg über den *Exit-Code* melden (0 bedeutet Erfolg, alles andere Fehler), damit andere Programme darauf reagieren können. Und es sollte schweigsam sein, solange alles gut geht: Die Konvention „Keine Nachricht ist eine gute Nachricht“ macht Ausgaben in Pipelines erst nutzbar.

Ein Gegenbeispiel zeigt, wie man diese Idee verletzt. Das folgende Skript vermischt Ergebnis und Statusmeldungen auf stdout.

```bash
# falsch: Meldung und Ergebnis landen beide auf stdout
#!/usr/bin/env bash
echo "Zaehle Zeilen..."
wc -l < "$1"
```

Wer die Ausgabe mit `zeilen.sh datei | sort -n` weiterverarbeitet, erhält zusätzlich die Zeile „Zaehle Zeilen...“ und damit Datenmüll. Richtig ist, Hinweise nach stderr zu schicken:

```bash
#!/usr/bin/env bash
echo "Zaehle Zeilen..." >&2   # Meldung auf stderr
wc -l < "$1"                  # Ergebnis auf stdout
```

Die Umleitung `>&2` leitet stdout des `echo`-Befehls auf Kanal 2 um. Im Terminal sehen Sie weiterhin beides; in einer Pipeline erreicht aber nur das Ergebnis den nächsten Prozess.

> **Ergänzung aus der Nachrecherche**
>
> Die Unix-Philosophie wird heute auch kritisch diskutiert: Moderne Werkzeuge liefern zunehmend strukturierte Daten wie JSON statt reinen Text, weil Text ohne feste Grammatik fehleranfällig zu parsen ist. Das Werkzeug `jq` (Modul 5) verbindet beide Welten: Es bleibt ein kleines Filterprogramm in der Pipeline, versteht aber JSON als Datenformat.

> **Praxisbeispiel**
>
> Ein Betriebsteam will wissen, welche IP-Adressen den Webserver am häufigsten ansprechen. Statt ein Programm zu schreiben, kombiniert es `awk` (IP-Spalte extrahieren), `sort`, `uniq -c` und `head`. Die gleiche Kette lässt sich in drei Minuten anpassen, wenn stattdessen die häufigsten URLs gefragt sind.

## 1.3 Interaktive Nutzung vs Skript

Im Alltag beginnt fast jede Automation interaktiv: Man probiert einen Befehl aus, korrigiert ihn, probiert weiter. Diese Experimentierphase ist wertvoll, denn die Shell zeigt sofort das Ergebnis. Irgendwann stellt man fest, dass man dieselbe Kette zum dritten Mal tippt. Das ist der Moment, in dem aus dem Ad-hoc-Kommando ein Skript werden sollte.

Der Unterschied ist mehr als bloß „in einer Datei gespeichert“. Interaktive Shells bieten Komfort, der in Skripten bewusst fehlt oder anders wirkt: Es gibt einen Prompt, eine History, Tab-Vervollständigung, Aliase und Jobsteuerung. Skripte laufen dagegen nicht interaktiv. Aliase werden dort standardmäßig nicht erweitert, die History existiert nicht, und eine Umgebung, die nur Ihre persönliche `.bashrc` herstellt, fehlt womöglich ganz. Ein Skript muss deshalb selbst dafür sorgen, dass alles, was es braucht, vorhanden ist.

Der wichtigste Nutzen eines Skripts ist *Wiederholbarkeit*. Eine Datei lässt sich versionieren, prüfen lassen (Code-Review), testen und an Kolleginnen und Kollegen weitergeben. Ein Befehl, den Sie nur in Ihrer History haben, ist dagegen weder dokumentiert noch reproduzierbar. Dazu kommt die *Dokumentation*: Kommentare erklären das Warum, ein Usage-Text das Wie.

Das folgende Beispiel macht aus einem typischen Einzeiler ein kleines, wiederverwendbares Skript. Die Ausgangslage: Sie wollen die größten Dateien eines Verzeichnisses finden. Interaktiv tippen Sie vielleicht `du -ah . | sort -rh | head -n 10`. Als Skript sieht das so aus:

```bash
#!/usr/bin/env bash
# top-dateien.sh - zeigt die groessten Dateien/Verzeichnisse
# Aufruf: top-dateien.sh [VERZEICHNIS] [ANZAHL]

verzeichnis="${1:-.}"     # Standard: aktuelles Verzeichnis
anzahl="${2:-10}"         # Standard: 10 Eintraege

if [ ! -d "$verzeichnis" ]; then
    echo "Fehler: '$verzeichnis' ist kein Verzeichnis" >&2
    exit 1
fi

du -ah "$verzeichnis" 2>/dev/null | sort -rh | head -n "$anzahl"
```

Zeile für Zeile: Der Kommentar nach der Shebang-Zeile nennt Zweck und Aufruf; wer das Skript in einem Jahr findet, weiß sofort Bescheid. Die Konstruktion `${1:-.}` bedeutet „nimm das erste Argument; ist es nicht gesetzt oder leer, nimm einen Punkt“ (genaueres in Modul 3). Der Test `[ ! -d ... ]` prüft, ob der Pfad kein Verzeichnis ist; dann geht eine Fehlermeldung nach stderr, und `exit 1` beendet das Skript mit einem Fehlercode. Die eigentliche Arbeit steht in der letzten Zeile, es ist der ursprüngliche Einzeiler, nun mit Variablen und *in Anführungszeichen* gesetzt. Damit funktioniert er auch für Verzeichnisse mit Leerzeichen im Namen.

Damit das Skript wie ein Programm aufgerufen werden kann, braucht die Datei das Ausführungsrecht: `chmod +x top-dateien.sh`. Anschließend startet `./top-dateien.sh /var/log 5` es. Alternativ lässt sich jede Datei auch ohne Ausführungsrecht mit `bash top-dateien.sh` starten, dann wird die Shebang-Zeile ignoriert und ausdrücklich Bash verwendet.

Eine häufige Stolperfalle betrifft den Unterschied zwischen *Ausführen* und *Sourcen*. Mit `./skript.sh` läuft das Skript in einem eigenen Kindprozess; Variablen, die es setzt, und Verzeichniswechsel (`cd`) gehen mit dem Prozessende verloren. Mit `source skript.sh` (oder kurz `. skript.sh`) werden die Zeilen dagegen in der aktuellen Shell ausgeführt.

```bash
# wechsle.sh
cd /tmp
EDITOR=vim

# falsch: wirkt nicht auf die aufrufende Shell
./wechsle.sh ; pwd        # zeigt das alte Verzeichnis

# richtig, wenn die Wirkung gewollt ist
source ./wechsle.sh ; pwd # zeigt /tmp
```

Beim ersten Aufruf wechselt nur der Kindprozess nach `/tmp` und endet; Ihre Shell bleibt, wo sie war. Erst `source` führt die Befehle in der aktuellen Shell aus. Dieser Mechanismus wird genutzt, um Konfigurationen oder Funktionsbibliotheken zu laden (Modul 6).

> **Merke**
>
> Interaktiv experimentieren, dann als Skript festhalten. Ein gutes Skript hat eine Shebang-Zeile, Kommentare zu Zweck und Aufruf, Argumentprüfung, Fehlermeldungen auf stderr und aussagekräftige Exit-Codes.

> **Praxisbeispiel**
>
> Im Betrieb entsteht aus „Ich schaue kurz in die Logs“ schnell eine feste Routine. Teams legen solche Skripte in einem Git-Repository ab (häufig `scripts/` oder `bin/`). Ein neues Teammitglied bekommt damit nicht eine mündliche Anleitung, sondern lauffähigen, überprüften Code.

## 1.4 Bash-Versionen und POSIX

Bash ist nicht gleich Bash: Es gibt Unterschiede zwischen den Versionen, und Bash ist nicht die einzige Shell. Wer Skripte schreibt, die auf verschiedenen Systemen laufen sollen, muss diese Landschaft kennen. Die Versionsnummer erfahren Sie mit `bash --version` oder aus der Variablen `$BASH_VERSION`; die Einzelteile stehen in dem Array `BASH_VERSINFO`.

Wichtige Meilensteine der Bash-Geschichte: Bash 3.2 ist die uralte Version, die Apple aus Lizenzgründen bis heute auf macOS ausliefert (die GPLv3-Lizenz neuerer Bash-Versionen wollte Apple nicht übernehmen). Bash 4.0 brachte assoziative Arrays (`declare -A`), `mapfile` und die `**`-Globbing-Option. Bash 4.4 und 5.0 ergänzten weitere Features wie die Variablen `EPOCHSECONDS` und `EPOCHREALTIME`. Die aktuelle Hauptversion ist Bash 5.3 (Juli 2025) mit einer neuen Form der Kommandosubstitution, die ohne Unterprozess auskommt.

> **Ergänzung aus der Nachrecherche**
>
> Nach dem Stand der Nachrecherche (Berichte von OMG! Ubuntu und LWN zur Veröffentlichung) erschien Bash 5.3 im Juli 2025 und führte unter anderem die Syntax `${ befehl; }` ein, die einen Befehl im aktuellen Shell-Kontext ausführt und seine Ausgabe einsetzt, sowie `${| befehl; }` für das Ergebnis in der Variablen `REPLY`. Für portable Skripte sind diese Neuerungen vorerst tabu, weil viele Distributionen noch ältere Bash-Versionen mitliefern. Prüfen Sie die Version auf Ihren Zielsystemen, bevor Sie neue Features nutzen.

Nun zu POSIX. Das ist ein Standard (Portable Operating System Interface), der unter anderem festlegt, wie eine Standard-Shell sich verhalten muss. Eine POSIX-konforme Shell wie `dash` (unter Debian und Ubuntu als `/bin/sh` installiert) versteht nur diesen Sprachumfang. Bash kann ebenfalls im POSIX-Modus laufen (Aufruf als `sh` oder mit `--posix`), bietet aber im Normalbetrieb viele Erweiterungen, die sogenannten *Bashismen*: `[[ ... ]]`, Arrays, `(( ... ))`, die Prozesssubstitution `<(...)` und anderes.

Welche Shell man wählt, ist eine bewusste Entscheidung. Schreiben Sie `#!/bin/sh`, dürfen Sie nur POSIX-Syntax verwenden; dafür läuft das Skript auch auf Alpine Linux (Busybox-`ash`), in minimalen Containern und auf älteren Unix-Systemen. Schreiben Sie `#!/usr/bin/env bash`, dürfen Sie Bash-Features nutzen, setzen aber voraus, dass Bash installiert ist.

Das folgende Beispiel zeigt denselben Test in beiden Welten und beweist, warum das Unterscheiden wichtig ist.

```bash
#!/bin/sh
# POSIX-konform: funktioniert in dash, ash, bash
name="Welt"
if [ "$name" = "Welt" ]; then
    echo "Hallo, $name"
fi

#!/usr/bin/env bash
# Bash-spezifisch: [[ ]] mit Mustervergleich
name="Welt"
if [[ $name == W* ]]; then
    echo "beginnt mit W"
fi
```

(Die beiden Fragmente gehören in zwei verschiedene Dateien; hier stehen sie nur zum Vergleich untereinander.) Der POSIX-Test benutzt die klassische Form `[ ... ]` mit einem einfachen `=`. Der Bash-Test `[[ ... ]]` erlaubt Mustervergleiche (`W*`) und braucht keine Anführungszeichen um Variablen, weil dort kein Word Splitting stattfindet. Läuft das zweite Fragment unter `dash`, erscheint ein Fehler wie „`[[: not found`“.

Der klassische Fehler: ein Skript mit `#!/bin/sh` zu beginnen und Bash-Syntax zu verwenden. Auf einem System, wo `/bin/sh` auf Bash zeigt (CentOS, Fedora), funktioniert das scheinbar; auf Ubuntu oder Debian bricht es. Falsch: `#!/bin/sh` plus Arrays. Richtig: entweder Shebang auf `bash` ändern oder auf POSIX-Konstrukte umstellen. Das Werkzeug ShellCheck (siehe 1.8) erkennt solche Fälle und warnt, wenn Bashismen in einem `sh`-Skript stehen.

> **Merke**
>
> Wählen Sie die Shebang passend zum Code: `sh` bedeutet „nur POSIX“, `bash` bedeutet „Bash-Features erlaubt“. Verwenden Sie `#!/usr/bin/env bash`, damit Bash auch dann gefunden wird, wenn es nicht unter `/bin/bash` liegt (etwa auf BSD oder mit Homebrew auf macOS).

> **Praxisbeispiel**
>
> Container-Images auf Alpine-Basis haben standardmäßig keine Bash. Ein Deployment-Skript mit Bash-Features schlägt dort mit „not found“ fehl, obwohl die Datei existiert, weil der in der Shebang genannte Interpreter fehlt. Teams entscheiden daher früh: entweder Bash im Image installieren oder die Skripte POSIX-konform halten.

## 1.5 Command Lookup

Wenn Sie in der Shell einen Befehl wie `ls` eintippen, muss die Shell herausfinden, *was* damit gemeint ist. Dafür durchsucht Bash mehrere Stellen in einer festen Reihenfolge, und wer diese Reihenfolge kennt, versteht, warum manchmal ein anderer Befehl läuft als erwartet.

Die Reihenfolge lautet: Zuerst prüft Bash, ob das erste Wort ein *Alias* ist, also eine Abkürzung, die Sie mit `alias ll='ls -l'` definiert haben (nur in interaktiven Shells aktiv). Danach kommen Schlüsselwörter der Sprache wie `if`, `for` oder `[[`. Es folgen *Funktionen*, die Sie selbst definiert haben, dann *Builtins*, also Befehle, die in Bash eingebaut sind, etwa `cd`, `echo`, `read` oder `type`. Erst zuletzt sucht Bash in den Verzeichnissen der Umgebungsvariablen `PATH` nach einer ausführbaren Datei mit dem Namen.

Warum gibt es Builtins? Manche Befehle *müssen* in der Shell selbst laufen, weil sie deren Zustand verändern: `cd` ändert das Arbeitsverzeichnis der Shell, und das könnte ein separater Prozess nie erreichen, weil er nur sein eigenes ändert. Andere Builtins wie `echo` und `test` gibt es aus Geschwindigkeitsgründen, denn sie sparen den Start eines Prozesses. Viele davon existieren zusätzlich als eigenständiges Programm (`/bin/echo`), das sich in Details anders verhalten kann.

Die Variable `PATH` ist eine durch Doppelpunkte getrennte Liste von Verzeichnissen, zum Beispiel `/usr/local/bin:/usr/bin:/bin`. Bash probiert sie von links nach rechts durch und nimmt den ersten Treffer. Aus Effizienzgründen merkt sich Bash gefundene Pfade in einer Hash-Tabelle (Befehl `hash`). Wird ein Programm verschoben oder neu installiert, kann diese Tabelle veraltet sein, was zu Fehlermeldungen wie „No such file or directory“ für einen Befehl führt, der eigentlich existiert; `hash -r` leert die Tabelle.

Zum Untersuchen dieser Auflösung dienen `type`, `command -v` und `which`. `type -a name` zeigt *alle* Fundstellen in der Reihenfolge der Auflösung, `command -v name` gibt in POSIX-konformer Weise den Pfad oder die Art des Befehls aus und liefert einen Fehlercode, wenn nichts gefunden wird. `which` ist dagegen ein externes Programm, das Aliase und Builtins nicht kennt und je nach System unterschiedlich arbeitet; für Skripte ist es daher die schlechtere Wahl.

```bash
#!/usr/bin/env bash
# lookup-demo.sh - zeigt die Aufloesung von Befehlen

type -a echo          # Builtin UND /bin/echo
type -t cd            # nur der Typ: builtin
type -t ls            # file (oder alias, wenn interaktiv definiert)

# Eigene Funktion ueberdeckt das externe Programm
date() { echo "ich bin nicht /usr/bin/date"; }
date                  # ruft die Funktion auf
command date          # 'command' umgeht Funktionen und Aliase
unset -f date         # Funktion wieder entfernen
date                  # jetzt wieder das echte Programm

# Pruefen, ob ein benoetigtes Werkzeug vorhanden ist
if ! command -v jq >/dev/null 2>&1; then
    echo "Fehler: jq ist nicht installiert" >&2
    exit 127
fi
```

`type -a echo` listet zwei Treffer: das Builtin und die Datei `/usr/bin/echo`. `type -t` gibt nur ein Stichwort wie `builtin`, `file`, `alias`, `function` oder `keyword` aus, was sich gut in Skripten auswerten lässt. Danach überdeckt eine selbst definierte Funktion `date` das Programm; mit dem Präfix `command` ruft man bewusst das Original auf, und `unset -f` entfernt die Funktion. Der letzte Block ist das Standardmuster, um Abhängigkeiten zu prüfen: `command -v jq` ist erfolgreich, wenn `jq` auffindbar ist, die Umleitungen verwerfen jede Ausgabe, und der Exit-Code 127 entspricht der Konvention „command not found“.

Typische Fehler: Wer `which` in Skripten benutzt, erhält je nach System unterschiedliche Ergebnisse (manche Varianten geben auch bei Fehlschlag Text aus statt nur einen Fehlercode). Falsch: `if [ -z "$(which jq)" ]; then ...`. Richtig: `if ! command -v jq >/dev/null; then ...`. Ein zweiter Klassiker ist der Aufruf eines Programms im aktuellen Verzeichnis. Tippen Sie `skript.sh`, sucht Bash nur im PATH und findet die Datei im aktuellen Ordner nicht; man schreibt `./skript.sh`. Ein Verzeichnis `.` im PATH gilt als Sicherheitsrisiko, weil eine manipulierte Datei in einem fremden Ordner dann ungewollt ausgeführt werden könnte.

> **Merke**
>
> Auflösungsreihenfolge: Alias, Schlüsselwort, Funktion, Builtin, PATH. Mit `type -a` sehen Sie alle Kandidaten, mit `command -v` prüfen Sie in Skripten portabel, ob etwas existiert.

> **Praxisbeispiel**
>
> Ein Skript läuft auf dem Laptop, aber im Cron-Job meldet es „command not found“. Ursache: Cron startet mit einem minimalen `PATH` (oft nur `/usr/bin:/bin`). Abhilfe: Im Skript den PATH explizit setzen oder absolute Pfade verwenden.

## 1.6 Manpages und Hilfe

Niemand merkt sich alle Optionen aller Befehle. Wer Linux professionell nutzt, ist deshalb vor allem gut darin, schnell die richtige Information zu finden. Es gibt drei Quellen, die sich in Tiefe und Verfügbarkeit unterscheiden: `man` für die ausführliche Referenz externer Programme, `help` für Bash-Builtins und die Option `--help` für eine Kurzübersicht, die die meisten Programme selbst ausgeben.

Das Handbuch `man` (Manual Pages) ist in nummerierte *Sektionen* gegliedert: 1 für Benutzerbefehle, 2 für Systemaufrufe, 3 für Bibliotheksfunktionen, 5 für Dateiformate und Konfigurationsdateien, 7 für Überblicke und Konventionen, 8 für Administrationsbefehle. Der Aufruf `man 5 passwd` zeigt die Beschreibung der Datei `/etc/passwd`, während `man passwd` den Befehl zum Passwortändern beschreibt. Gleiche Namen in verschiedenen Sektionen sind der Grund, warum man die Sektion angeben kann. Das Handbuch öffnet sich in einem Pager (meist `less`): Mit Leertaste blättern, mit `/` suchen, mit `n` zum nächsten Treffer, mit `q` beenden.

Eine Manpage ist nach festem Muster aufgebaut: NAME (Kurzbeschreibung), SYNOPSIS (Aufrufsyntax), DESCRIPTION, OPTIONS, dann oft EXIT STATUS, EXAMPLES und SEE ALSO. In der Synopsis gilt die Konvention: Eckige Klammern `[ ]` bedeuten optional, Pünktchen `...` bedeuten Wiederholung, ein senkrechter Strich bedeutet Alternative, Unterstrichenes oder Kursives sind Platzhalter. Wer das Schema kennt, liest eine Seite in Sekunden.

Für Bash-Builtins ist `man` ungeeignet, denn `man cd` zeigt auf manchen Systemen nur die generelle Bash-Seite. Besser ist `help cd`. Ohne Argument listet `help` alle Builtins. Die komplette Bash-Referenz steht in `man bash`; sie ist lang, aber mit der Suche (`/` gefolgt vom Stichwort, etwa `/Parameter Expansion`) gut navigierbar. Nützlich sind außerdem `apropos stichwort` (oder `man -k`), das Handbuchseiten nach Stichwort durchsucht, wenn man den Befehlsnamen nicht kennt, und `tldr` als Community-Werkzeug mit kurzen Beispielen (muss nachinstalliert werden).

```bash
man 7 regex          # Ueberblick ueber reguläre Ausdruecke
man -k "copy file"   # Suche: welche Befehle kopieren Dateien?
help test            # Bash-Builtin 'test' bzw. '['
help -d cd           # Kurzbeschreibung eines Builtins
ls --help | head -n 20
type -a printf       # Builtin und/oder Programm?

# Exit-Code eines Befehls nachlesen
grep --version >/dev/null; echo "Exit: $?"
```

Die Beispiele zeigen die Vielfalt der Zugänge. Der Aufruf mit `-k` durchsucht Kurzbeschreibungen. `help test` erklärt die Bedingungen für `[ ]`. `ls --help` liefert eine Kurzreferenz, die man mit `head` auf die ersten Zeilen begrenzt, weil sie lang sein kann. Die letzte Zeile gibt mit `$?` den Exit-Code des vorherigen Befehls aus.

Besonders wichtig für Skripte ist der Abschnitt *EXIT STATUS*, den viele Manpages enthalten. `grep` etwa beendet sich mit 0, wenn mindestens eine Zeile passt, mit 1, wenn nichts passt, und mit 2 bei einem Fehler. Wer das nicht weiß, behandelt „keine Treffer“ fälschlich als Fehler oder umgekehrt. In einem Skript mit `set -e` (Modul 6) führt der Exit-Code 1 von `grep` ohne Treffer sogar zum Abbruch, wenn man ihn nicht bewusst abfängt.

```bash
# falsch: Exit-Code 1 (kein Treffer) wird wie ein Fehler behandelt
grep "FEHLER" logdatei.txt || { echo "grep ist kaputt"; exit 1; }

# richtig: die drei Faelle unterscheiden
grep -q "FEHLER" logdatei.txt
status=$?
case "$status" in
    0) echo "Fehler im Log gefunden" ;;
    1) echo "Log ist sauber" ;;
    *) echo "grep selbst ist fehlgeschlagen (Code $status)" >&2 ;;
esac
```

Das Beispiel nutzt `grep -q` (quiet: keine Ausgabe, nur Exit-Code). Die Variable `status` sichert den Code sofort, denn `$?` wird nach jedem Befehl überschrieben. Die `case`-Anweisung unterscheidet dann die drei Fälle sauber, was die Dokumentation von `grep` vorgibt.

> **Merke**
>
> Die Frage nach dem richtigen Werkzeug hat drei Antworten: `help` für Bash-Builtins, `man` für Programme und Konfigurationsdateien, `--help` als Schnellübersicht. Vergessen Sie nie den Abschnitt EXIT STATUS, bevor Sie sich in einem Skript auf einen Exit-Code verlassen.

> **Praxisbeispiel**
>
> Bei Störungen um zwei Uhr nachts ist keine Zeit für Websuche. Wer in der Manpage von `journalctl` oder `ss` gezielt nach Beispielen oder Optionen sucht, findet die Lösung offline und mit der zum installierten Version passenden Dokumentation, was bei Webartikeln zu anderen Programmversionen nicht garantiert ist.

## 1.7 Shell-Initialisierung

Wenn Bash startet, liest sie je nach Startmodus bestimmte Konfigurationsdateien. Wer nicht weiß, welche, platziert Einstellungen an der falschen Stelle und wundert sich, warum sie manchmal wirken und manchmal nicht. Diese Lerneinheit ordnet die Dateien ein.

Zuerst die vier Startmodi, die sich aus den zwei Fragen „Login oder nicht?“ und „interaktiv oder nicht?“ ergeben. Eine *interaktive Login-Shell* entsteht bei der Anmeldung an der Konsole oder per SSH. Sie liest zuerst die systemweite Datei `/etc/profile` und danach die erste vorhandene Datei aus der Liste `\~{}/.bash_profile`, `\~{}/.bash_login`, `\~{}/.profile`. Eine *interaktive Nicht-Login-Shell*, wie ein neues Terminalfenster, liest dagegen `\~{}/.bashrc` (auf manchen Distributionen zusätzlich `/etc/bash.bashrc`). Eine *nicht interaktive Shell*, also ein Skript, liest keine dieser Dateien, außer die Variable `BASH_ENV` verweist auf eine Datei. Beim Beenden einer Login-Shell wird `\~{}/.bash_logout` ausgeführt.

Das führt zu einer verbreiteten Konvention. Weil Login-Shells `.bashrc` nicht automatisch lesen, enthält die `.bash_profile` üblicherweise eine Zeile, die `.bashrc` einbindet. So gilt die Konfiguration einheitlich, egal wie die Shell gestartet wurde.

```bash
# ~/.bash_profile  -- nur fuer Login-Shells
# Umgebungsvariablen gehoeren hierher (einmal pro Anmeldung)
export PATH="$HOME/bin:$PATH"
export EDITOR=vim

# .bashrc einbinden, damit Login- und Nicht-Login-Shells gleich sind
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi

# ~/.bashrc  -- fuer interaktive Shells
# Nichts tun, wenn nicht interaktiv (z. B. bei scp oder rsync ueber SSH)
case $- in
    *i*) ;;
    *) return ;;
esac

alias ll='ls -lah --color=auto'
PS1='\u@\h:\w\$ '
HISTSIZE=10000
HISTCONTROL=ignoredups:erasedups
```

In der `.bash_profile` stehen Umgebungsvariablen mit `export`, die an alle Kindprozesse weitergegeben werden sollen. Danach wird `.bashrc` mit dem Punkt-Befehl (Synonym für `source`) eingebunden. In der `.bashrc` steht am Anfang ein Schutz: Die Variable `$-` enthält die aktiven Shell-Optionen, und der Buchstabe `i` bedeutet „interaktiv“. Ist er nicht dabei, beendet `return` die Datei sofort. Das ist wichtig, denn Ausgaben oder Rückfragen in einer `.bashrc` würden Werkzeuge wie `scp` oder `rsync` stören, die über SSH eine nicht interaktive Shell starten und eine saubere Datenleitung erwarten. Danach folgen Aliase, der Prompt (`PS1`) und History-Einstellungen, also reine Komfortfunktionen.

Eine klare Regel hilft bei der Verteilung: Umgebungsvariablen und PATH in die Login-Datei, Komfort für die interaktive Nutzung (Aliase, Prompt, Farben) in die `.bashrc`. Skripte sollten sich nie auf Ihre `.bashrc` verlassen: Sie laufen nicht interaktiv, werden eventuell von anderen Benutzern oder Diensten gestartet und müssen alles Nötige selbst setzen.

Der häufigste Fehler ist das Gegenteil: Jemand definiert in der `.bashrc` einen Alias oder eine Variable, nutzt sie in einem Skript und wundert sich, dass sie dort fehlt. Falsch: Ein Skript, das den Alias `ll` verwendet. Richtig: das Skript ruft `ls -lah` direkt auf. Ebenfalls problematisch sind Ausgaben (`echo`) in Startdateien, die Dateiübertragungen mit `scp` zum Scheitern bringen können.

> **Merke**
>
> Login-Shell: `/etc/profile`, dann `.bash_profile` (oder `.profile`). Interaktive Nicht-Login-Shell: `.bashrc`. Skript: keine der Dateien. Umgebung in die Profile-Datei, Komfort in die `.bashrc`, und nie darauf verlassen, dass ein Skript Ihre persönliche Konfiguration sieht.

> **Praxisbeispiel**
>
> Ein Entwickler installiert ein Werkzeug nach `\~{}/.local/bin` und ergänzt den PATH in der `.bashrc`. Das Terminal funktioniert, aber der Cron-Job und das CI-System finden das Werkzeug nicht. Der Fix: Entweder den vollen Pfad im Skript verwenden oder den PATH im Skript selbst setzen. Und: Was im Terminal geht, muss nicht im Dienst gehen.

## 1.8 Praxisumgebung

Bevor die eigentliche Arbeit beginnt, brauchen Sie eine Umgebung, in der Sie gefahrlos experimentieren können. Skripte, die Dateien löschen oder Systemeinstellungen ändern, gehören nicht zum ersten Test auf Ihrem Arbeitsrechner. Drei Wege eignen sich: eine Linux-Virtual-Machine, das Windows Subsystem for Linux (WSL) oder ein Container.

Eine *Linux-VM* (etwa mit VirtualBox, VMware oder Hyper-V) bildet ein komplettes System nach, inklusive Kernel und Systemdiensten. Das ist die realistischste Variante, besonders für Modul 7 (Systemadministration), in dem systemd, Benutzer und Pakete eine Rolle spielen. Ein Nachteil ist der höhere Ressourcenbedarf. *WSL* lässt unter Windows ein echtes Linux (zum Beispiel Ubuntu) neben der Windows-Umgebung laufen. Das Einrichten ist mit `wsl --install` in einer Administrator-Eingabeaufforderung erledigt. Ein *Container* (Docker oder Podman) startet in Sekunden eine frische, wegwerfbare Umgebung und eignet sich gut zum Testen auf verschiedenen Distributionen.

Als Grundausstattung benötigen Sie einen Editor, ShellCheck und Git. Der Editor kann `nano` (einfach), `vim` oder Visual Studio Code sein; entscheidend ist, dass er Zeilenenden im Unix-Format (LF) speichert. Windows-Zeilenenden (CRLF) führen zu rätselhaften Fehlern wie „`bad interpreter: /usr/bin/env bash\r`“, weil das unsichtbare Zeichen Wagenrücklauf als Teil des Interpreternamens gelesen wird. *ShellCheck* ist ein statischer Analysator, der Skripte liest, ohne sie auszuführen, und typische Fehler meldet. Er warnt zum Beispiel vor fehlenden Anführungszeichen (Regel SC2086), und jede Warnung hat eine Nummer, zu der es im Wiki eine Erklärung mit Falsch-/Richtig-Beispiel gibt. *Git* ist das Versionsverwaltungssystem, das jede Änderung nachvollziehbar macht.

> **Ergänzung aus der Nachrecherche**
>
> Laut der ShellCheck-Wiki-Seite zu SC2086 (abgerufen) warnt das Werkzeug vor unquotierten Variablen, weil Word Splitting und Globbing sonst unerwartete Ergebnisse liefern. Dort empfohlen: `echo "$1"` statt `echo $1`, `"$@"` statt `$@`, und Arrays statt gewollter Aufteilung eines Strings.

Das folgende Skript richtet eine kleine Arbeitsumgebung ein und prüft, ob die Werkzeuge vorhanden sind. Es ist ein vollständiges Beispiel für eine Vorbedingungsprüfung.

```bash
#!/usr/bin/env bash
# setup-check.sh - prueft die Arbeitsumgebung fuer den Bash-Kurs

benoetigt=(bash git shellcheck jq curl)
fehlt=()

for werkzeug in "${benoetigt[@]}"; do
    if command -v "$werkzeug" >/dev/null 2>&1; then
        printf '%-12s OK  (%s)\n' "$werkzeug" "$(command -v "$werkzeug")"
    else
        printf '%-12s FEHLT\n' "$werkzeug"
        fehlt+=("$werkzeug")
    fi
done

if [ "${#fehlt[@]}" -gt 0 ]; then
    echo "Bitte installieren: ${fehlt[*]}" >&2
    exit 1
fi

mkdir -p "$HOME/bash-kurs" && cd "$HOME/bash-kurs" || exit 1
git init -q 2>/dev/null && echo "Arbeitsverzeichnis bereit: $PWD"
```

Die Liste der benötigten Werkzeuge steht in einem Array (Modul 3 behandelt Arrays im Detail). Die Schleife prüft jedes Element mit `command -v`; `printf` mit der Formatangabe `%-12s` gibt Spalten ausgerichtet aus. Fehlende Werkzeuge werden mit `+=` an ein zweites Array angehängt. `${#fehlt[@]}` liefert die Anzahl der Elemente; ist sie größer als null, endet das Skript mit Hinweis und Fehlercode. Am Ende legt es ein Arbeitsverzeichnis an und initialisiert ein Git-Repository. Das `||` nach `cd` sorgt dafür, dass das Skript abbricht, falls der Wechsel scheitert, und nicht versehentlich in einem falschen Verzeichnis weiterarbeitet.

Rufen Sie dann `shellcheck setup-check.sh` auf. Ein sauberes Skript erzeugt keine Ausgabe. Gewöhnen Sie sich an, jedes Skript mit ShellCheck zu prüfen, bevor Sie es einsetzen; viele Editoren binden das Werkzeug als Erweiterung ein und zeigen Warnungen direkt beim Tippen.

Zwei typische Fehler der Einrichtung: Erstens, Skripte in einem Windows-Editor schreiben und unter Linux ausführen (CRLF-Problem). Falsch: Datei mit Windows-Zeilenenden. Richtig: Editor auf LF einstellen oder mit `dos2unix datei.sh` bzw. `sed -i 's/\r$//' datei.sh` konvertieren. Zweitens, mit Dateien in `/mnt/c` (WSL) zu arbeiten: Der Zugriff auf das Windows-Dateisystem ist langsam, und Linux-Dateirechte werden dort nur eingeschränkt abgebildet. Besser ist es, Projekte im Linux-Home-Verzeichnis abzulegen.

> **Merke**
>
> Arbeiten Sie in einer Umgebung, in der ein Fehler nichts kaputt macht (VM, WSL, Container). Editor mit LF-Zeilenenden, ShellCheck und Git gehören zur Grundausstattung; ShellCheck prüfen Sie vor jedem Einsatz eines Skripts.

> **Praxisbeispiel**
>
> Viele Teams hängen ShellCheck als Pflichtschritt in die CI-Pipeline und als Pre-commit-Hook (siehe Modul 10) ein. Dann kommt kein Skript mehr ins Repository, das offensichtliche Fehler enthält, und Reviews können sich auf die fachliche Logik konzentrieren.

## 1.9 Übungsaufgaben zu Modul 01

Die Aufgaben sind nach Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen das Verständnis der Konzepte, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben. Versuchen Sie jede Aufgabe zuerst selbst, bevor Sie die Lösung am Ende des Moduls lesen.

**Aufgabe 1.** Erklären Sie in eigenen Worten den Unterschied zwischen Shell, Terminal und TTY. Welche der drei Komponenten erledigt das Interpretieren einer eingetippten Zeile?

**Aufgabe 2.** Ein Kollege behauptet: „Mein Alias `ll` funktioniert im Terminal, aber nicht in meinem Skript. Bash ist kaputt.“ Erklären Sie, was tatsächlich passiert, und nennen Sie zwei Lösungen.

**Aufgabe 3.** Nennen Sie die Reihenfolge, in der Bash ein Kommando auflöst. Welcher Befehl zeigt alle Fundstellen eines Namens an? Warum ist `command -v` für Skripte besser geeignet als `which`?

**Aufgabe 4.** In welche Datei gehört die Zeile `export PATH="$HOME/bin:$PATH"`, und welche Datei sollte `.bash_profile` zusätzlich einbinden? Begründen Sie die Antwort mit den vier Startmodi der Shell.

**Aufgabe 5.** Schreiben Sie ein Skript `wer-bin-ich.sh`, das Benutzername, Hostname, Bash-Version und das aktuelle Datum in jeweils einer Zeile ausgibt und zusätzlich meldet, ob es interaktiv gestartet wurde.

**Aufgabe 6.** Schreiben Sie ein Skript `pruefe-tools.sh`, das als Argumente eine beliebige Liste von Programmnamen entgegennimmt und für jedes meldet, ob es installiert ist. Der Exit-Code soll 0 sein, wenn alle vorhanden sind, sonst 1. Meldungen über fehlende Werkzeuge gehören auf stderr.

**Aufgabe 7.** Schreiben Sie ein Skript `haeufige-befehle.sh`, das aus einer Textdatei (Argument, ersatzweise stdin) die zehn häufigsten Wörter ermittelt und mit ihrer Anzahl ausgibt. Das Skript soll eine Fehlermeldung ausgeben, wenn die angegebene Datei nicht existiert.

**Aufgabe 8.** Untersuchen Sie das Verhalten von `./skript.sh` und `source skript.sh` anhand eines Skripts, das ein Verzeichnis wechselt und eine Variable setzt. Schreiben Sie ein kurzes Testskript, das beide Aufrufarten automatisch ausführt und die Unterschiede ausgibt.

## 1.10 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Das *Terminal* (heute meist ein Terminalemulator) ist das Fenster, das Tasteneingaben entgegennimmt und Text anzeigt. *TTY* bezeichnet historisch das Fernschreibergerät, heute das vom Kernel bereitgestellte Terminalgerät (praktisch ein Pseudo-Terminal wie `/dev/pts/0`), an dem die Ein- und Ausgabe eines Prozesses hängt. Die *Shell* ist ein ganz normales Programm, das auf diesem Gerät läuft und Zeilen interpretiert; nur sie entscheidet, was ein Kommando bedeutet und welche Programme zu starten sind. Das Interpretieren übernimmt also die Shell, nicht das Terminal.

**Lösung zu Aufgabe 2.** Aliase werden nur in *interaktiven* Shells erweitert. Ein Skript läuft nicht interaktiv und liest zudem die `.bashrc` nicht, in der der Alias definiert ist. Bash ist also nicht kaputt, sondern verhält sich wie dokumentiert. Lösung 1: Im Skript den vollen Befehl schreiben (`ls -l` statt `ll`). Lösung 2: Statt eines Alias eine Funktion oder ein eigenes kleines Skript im PATH verwenden. (Aliase im Skript zu aktivieren, ist mit `shopt -s expand_aliases` möglich, wird aber nicht empfohlen.)

**Lösung zu Aufgabe 3.** Alias, Schlüsselwort, Funktion, Builtin, dann die Suche im PATH. Alle Fundstellen zeigt `type -a name`. `command -v` ist ein Builtin mit festgelegtem POSIX-Verhalten: Es gibt den Pfad bzw. die Art des Befehls aus und liefert bei Misserfolg einen Fehlercode. `which` ist je nach System ein externes Programm mit abweichendem Verhalten, kennt Builtins und Funktionen nicht und gibt manchmal auch bei Fehlschlag Text auf stdout aus.

**Lösung zu Aufgabe 4.** Die Zeile gehört in die `.bash_profile` (oder `.profile`), weil Umgebungsvariablen einmal pro Anmeldung gesetzt werden und an alle Kindprozesse vererbt werden. Da eine Login-Shell die `.bashrc` nicht automatisch liest, bindet die `.bash_profile` sie per `. ~/.bashrc` ein; so gelten Aliase und Prompt auch in Login-Shells. Nicht interaktive Shells (Skripte) lesen keine der beiden Dateien.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# wer-bin-ich.sh
echo "Benutzer : $(id -un)"
echo "Host     : $(hostname)"
echo "Bash     : $BASH_VERSION"
echo "Datum    : $(date +%F)"
if [ -t 0 ] && [ -t 1 ]; then
    echo "Modus    : interaktiv (Terminal)"
else
    echo "Modus    : nicht interaktiv"
fi
```

Die Kommandosubstitution `$(...)` setzt die Ausgaben der Befehle ein; `date +%F` liefert das Datum im Format JJJJ-MM-TT. Die Interaktivität wird über die Terminaltests für stdin (`-t 0`) und stdout (`-t 1`) bestimmt; mit `&&` müssen beide erfüllt sein.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# pruefe-tools.sh PROGRAMM [PROGRAMM ...]
if [ "$#" -eq 0 ]; then
    echo "Aufruf: $0 PROGRAMM [PROGRAMM ...]" >&2
    exit 2
fi

fehlend=0
for tool in "$@"; do
    if command -v "$tool" >/dev/null 2>&1; then
        echo "OK     $tool"
    else
        echo "FEHLT  $tool" >&2
        fehlend=1
    fi
done
exit "$fehlend"
```

`$#` enthält die Anzahl der Argumente, `"$@"` die Argumente einzeln und korrekt gequotet. Ohne Argumente zeigt das Skript eine Usage-Meldung und endet mit Code 2 (üblich für falsche Benutzung). Die Variable `fehlend` merkt sich, ob etwas fehlte; erst am Ende wird mit dem passenden Code beendet, damit alle Werkzeuge geprüft werden und nicht nur das erste.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# haeufige-befehle.sh [DATEI]   (ohne Argument: stdin)
datei="${1:-/dev/stdin}"

if [ "$datei" != "/dev/stdin" ] && [ ! -f "$datei" ]; then
    echo "Fehler: Datei '$datei' nicht gefunden" >&2
    exit 1
fi

tr -cs '[:alpha:]' '\n' < "$datei" \
    | tr '[:upper:]' '[:lower:]' \
    | sort | uniq -c | sort -rn | head -n 10
```

Der Standardwert `/dev/stdin` sorgt dafür, dass das Skript auch in einer Pipeline (`cat text.txt | ./haeufige-befehle.sh`) arbeitet. `tr -cs '[:alpha:]' '\n'` ersetzt alles, was kein Buchstabe ist, durch Zeilenumbrüche (`-c` Komplement, `-s` Wiederholungen zusammenfassen), sodass ein Wort pro Zeile entsteht. Die weitere Kette ist die bekannte Zählung. Der Backslash am Zeilenende erlaubt es, die Pipeline lesbar auf mehrere Zeilen zu verteilen.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# test-source.sh - Unterschied zwischen Ausfuehren und Sourcen
tmp="$(mktemp -d)"
cat > "$tmp/wechsle.sh" <<'INNER'
cd /tmp
MEINE_VAR="gesetzt"
INNER

start="$PWD"
unset MEINE_VAR

bash "$tmp/wechsle.sh"
echo "Nach AUSFUEHREN: pwd=$PWD  var='${MEINE_VAR:-}'"

cd "$start"
source "$tmp/wechsle.sh"
echo "Nach SOURCEN   : pwd=$PWD  var='${MEINE_VAR:-}'"

cd "$start"; rm -rf "$tmp"
```

Das Skript erzeugt mit `mktemp -d` ein temporäres Verzeichnis und schreibt über ein Here-Document eine Hilfsdatei hinein. Beim Ausführen mit `bash` bleibt das Arbeitsverzeichnis unverändert und die Variable leer, weil beides im Kindprozess geschah. Beim Sourcen wechselt die aktuelle Shell nach `/tmp`, und die Variable ist gesetzt. Zum Schluss räumt das Skript auf. Mit `${MEINE_VAR:-}` vermeidet man eine Fehlermeldung bei ungesetzter Variable.
