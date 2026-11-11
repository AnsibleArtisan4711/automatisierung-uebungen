# Modul 02 - Dateisystem, Navigation und Textwerkzeuge

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (2.1 bis 2.9; 2.10 Übungsaufgaben, 2.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Als Richtwert für den Umfang gilt der höhere Wert von 10 h.  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit                  | Dauer (h) | Prio |
|:----|:-----------------------------|----------:|:----:|
| 2.1 | Dateisystem-Hierarchie       |      1,00 |  1   |
| 2.2 | Navigation und Listing       |      1,00 |  1   |
| 2.3 | Dateioperationen             |      1,00 |  1   |
| 2.4 | Berechtigungen               |      1,00 |  1   |
| 2.5 | Textanzeige                  |      1,00 |  1   |
| 2.6 | Streams und Pipes Grundlagen |      1,00 |  1   |
| 2.7 | Suchen im Dateisystem        |      1,00 |  2   |
| 2.8 | Archivierung und Kompression |      1,00 |  2   |
| 2.9 | Checksummen                  |      1,00 |  2   |
|     | **Summe (Details)**          |  **9,00** |      |

Dieses Modul behandelt das Handwerkszeug, das Sie in jeder Shell-Sitzung und in fast jedem Skript brauchen: die Struktur des Linux-Dateisystems, das Bewegen darin, das sichere Verändern von Dateien, die Vergabe von Rechten und die Kernwerkzeuge zum Lesen, Suchen, Packen und Prüfen. Alle Beispiele können Sie gefahrlos in einem Übungsverzeichnis nachvollziehen; legen Sie dazu zu Beginn mit `mkdir -p ~/bash-kurs/modul02 && cd ~/bash-kurs/modul02` ein eigenes Arbeitsverzeichnis an.

## 2.1 Dateisystem-Hierarchie

Unter Linux gibt es, anders als unter Windows, keine Laufwerksbuchstaben. Alles liegt in einem einzigen Verzeichnisbaum, der an der *Wurzel* (englisch *root*, geschrieben `/`) beginnt. Weitere Datenträger werden an beliebigen Stellen in diesen Baum *eingehängt* (*gemountet*). Aus Sicht eines Programms ist es unerheblich, ob unter `/home` eine Festplatte, ein Netzwerklaufwerk oder ein Stück Arbeitsspeicher steckt; es sieht nur Verzeichnisse und Dateien. Dieses Prinzip heißt „Alles ist eine Datei“: Auch Geräte (`/dev/sda`), Prozessinformationen (`/proc`) und der Terminalzugriff sind als Dateien ansprechbar.

Wie der Baum aufgeteilt wird, regelt der *Filesystem Hierarchy Standard* (FHS), den praktisch alle Distributionen befolgen. Für den Einstieg genügen fünf Verzeichnisse. `/etc` enthält systemweite Konfigurationsdateien als Text, etwa `/etc/passwd` (Benutzerkonten) oder `/etc/hosts`. `/var` enthält veränderliche Daten, vor allem Logdateien unter `/var/log`, aber auch Warteschlangen und Caches. `/home` enthält die persönlichen Verzeichnisse der Benutzer (`/home/anna`); das Verzeichnis des Administrators heißt `/root`. `/usr` enthält installierte Programme und Bibliotheken (`/usr/bin`, `/usr/lib`); auf modernen Systemen sind `/bin` und `/sbin` lediglich Verweise darauf. `/tmp` nimmt temporäre Dateien auf, die jeder Benutzer anlegen darf und die beim Neustart oder durch Aufräumdienste verschwinden können.

> **Ergänzung aus der Nachrecherche**
>
> Zusätzlich zu den genannten Verzeichnissen sind in der Praxis wichtig: `/opt` für eigenständige Fremdsoftware, `/srv` für Daten, die das System ausliefert (Webserver), `/mnt` und `/media` als Einhängepunkte, `/proc` und `/sys` als virtuelle Dateisysteme mit Kernel-Informationen sowie `/run` für Laufzeitdaten wie PID-Dateien. Die Aufteilung kann je nach Distribution geringfügig abweichen; die verbindliche Quelle ist die Manpage `man hier`.

Jeder Eintrag im Baum wird über einen *Pfad* angesprochen. Ein *absoluter Pfad* beginnt mit `/` und gilt unabhängig vom aktuellen Standort, etwa `/var/log/syslog`. Ein *relativer Pfad* beginnt nicht mit `/` und wird ab dem aktuellen *Arbeitsverzeichnis* interpretiert, etwa `logs/app.log`. Zwei besondere Namen existieren in jedem Verzeichnis: `.` steht für das Verzeichnis selbst, `..` für das übergeordnete. Die Tilde `\~{}` ist eine Abkürzung der Shell für das Home-Verzeichnis, die Bash vor der Ausführung ersetzt (*Tilde-Expansion*); ein Programm bekommt also bereits den ausgeschriebenen Pfad.

Welcher Pfadtyp ist der richtige? In interaktiver Arbeit sind relative Pfade bequem. In Skripten sind sie eine Fehlerquelle, weil das Arbeitsverzeichnis davon abhängt, *von wo* jemand das Skript aufruft. Ein Skript, das `daten/eingabe.csv` liest, funktioniert nur, wenn es im richtigen Ordner gestartet wird. Robuste Skripte ermitteln deshalb ihren eigenen Speicherort und bauen daraus absolute Pfade.

```bash
#!/usr/bin/env bash
# pfade.sh - Pfade sicher aufloesen

# Verzeichnis, in dem DIESES Skript liegt (unabhaengig vom Aufrufort)
skript_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "Skript liegt in : $skript_dir"
echo "Aufgerufen aus  : $PWD"

# Eingabedatei relativ zum Skript, nicht zum Aufrufort
eingabe="$skript_dir/daten/eingabe.csv"
echo "Eingabedatei    : $eingabe"

# Einen beliebigen Pfad in einen absoluten verwandeln
echo "Absolut         : $(realpath -m ../daten)"
echo "Basisname       : $(basename /var/log/syslog)"
echo "Verzeichnisteil : $(dirname /var/log/syslog)"
```

Die Variable `BASH_SOURCE[0]` enthält den Pfad der gerade laufenden Skriptdatei, so wie er aufgerufen wurde, also möglicherweise relativ. `dirname` schneidet den Dateinamen ab. Das `cd` in der inneren Kommandosubstitution wechselt in dieses Verzeichnis (in einer Subshell, ändert also nichts am Aufrufer), und `pwd` druckt den nun *absoluten* Ort. Der Rest des Beispiels zeigt `realpath -m`, das einen Pfad normalisiert, auch wenn er nicht existiert, sowie `basename` und `dirname` zum Zerlegen von Pfaden.

Typische Fehler gibt es besonders beim Zusammenbauen von Pfaden. Falsch ist `rm -rf $VERZ/*` mit einer womöglich leeren Variable: Wird `VERZ` nicht gesetzt, lautet der Befehl `rm -rf /*`, und das Löschen beginnt an der Wurzel. Besser ist es, die Variable zu prüfen und zu quoten:

```bash
# falsch: bei leerem VERZ wird /* geloescht
rm -rf $VERZ/*

# besser: Abbruch, wenn die Variable leer oder nicht gesetzt ist
rm -rf "${VERZ:?VERZ ist nicht gesetzt}"/*
```

Die Schreibweise `${VERZ:?Text}` bricht die Ausführung mit einer Fehlermeldung ab, wenn die Variable leer oder nicht gesetzt ist. Die Anführungszeichen schützen vor Leerzeichen im Pfad, und der Stern bleibt außerhalb der Quotes, damit er als Platzhalter wirkt.

> **Merke**
>
> Ein einziger Baum ab `/`; Konfiguration in `/etc`, veränderliche Daten in `/var`, Benutzerdaten in `/home`, Programme in `/usr`, Temporäres in `/tmp`. In Skripten bevorzugt man absolute Pfade oder Pfade relativ zum Skriptort, nie Annahmen über das Arbeitsverzeichnis.

> **Praxisbeispiel**
>
> Ein Cron-Job startet im Home-Verzeichnis, nicht im Projektordner. Das Skript `backup.sh`, das relative Pfade verwendet, findet seine Konfiguration nicht. Die Lösung ist genau das Muster oben: den Skriptordner einmal ermitteln und alle Pfade daraus ableiten.

## 2.2 Navigation und Listing

Das Navigieren im Dateibaum besteht aus wenigen Befehlen. `pwd` (print working directory) zeigt das aktuelle Arbeitsverzeichnis. `cd` (change directory) wechselt es. Ohne Argument führt `cd` ins Home-Verzeichnis, mit `cd -` springt man zum vorherigen Verzeichnis zurück, und `cd ..` geht eine Ebene nach oben. `cd` ist ein Builtin, weil nur die Shell selbst ihr Arbeitsverzeichnis ändern kann (siehe Modul 1).

Zum Anzeigen von Verzeichnisinhalten dient `ls`. Seine wichtigsten Optionen: `-l` (lange Ausgabe mit Rechten, Besitzer, Größe, Datum), `-a` (auch versteckte Einträge, deren Name mit einem Punkt beginnt), `-h` (Größen menschenlesbar, etwa `4,0K`), `-t` (nach Änderungszeit sortieren), `-r` (Reihenfolge umkehren), `-d` (Verzeichnis selbst statt Inhalt anzeigen) und `-R` (rekursiv). Die Kombination `ls -lah` gehört bei vielen zum Dauerrepertoire. Eine Baumdarstellung liefert `tree`, das allerdings nachinstalliert werden muss.

Zu den *versteckten Dateien*: Unter Unix gibt es keine Dateiattribute „versteckt“; es ist lediglich Konvention, dass Namen mit Punkt nicht angezeigt werden. Konfigurationsdateien im Home-Verzeichnis wie `.bashrc` und `.ssh/` folgen dieser Regel, damit das Verzeichnis übersichtlich bleibt.

Eine der wichtigsten Fähigkeiten der Shell ist das *Globbing* (Dateinamen-Expansion). Muster wie `*.log` werden *von der Shell* in die Liste aller passenden Dateinamen umgewandelt, bevor der Befehl überhaupt startet. Das Programm sieht nie den Stern, sondern nur die fertige Liste. Es gibt vier Platzhalter: `*` steht für beliebig viele Zeichen (auch null), `?` für genau ein Zeichen, `[abc]` für eines der Zeichen in der Klammer (auch Bereiche wie `[0-9]`), und `[!abc]` für ein Zeichen, das nicht dazugehört. Mit der Bash-Option `shopt -s extglob` kommen erweiterte Muster wie `+(a|b)` hinzu, mit `shopt -s globstar` passt `**` auf beliebig tiefe Verzeichnisse. Brace-Expansion mit geschweiften Klammern (`{a,b,c}`) ist dagegen *kein* Globbing, sondern erzeugt vorab einfach Textvarianten, auch für nicht vorhandene Dateien.

```bash
#!/usr/bin/env bash
# globbing-demo.sh - im Arbeitsverzeichnis ausprobieren
mkdir -p demo && cd demo || exit 1
touch a.txt b.txt c.log "mit leerzeichen.txt" .versteckt

echo "--- ls -A (inkl. versteckter, ohne . und ..)"
ls -A
echo "--- *.txt"
echo *.txt                 # Achtung: ohne Quotes!
echo "--- ?.txt (genau ein Zeichen)"
echo ?.txt
echo "--- [ab].txt"
echo [ab].txt
echo "--- Brace Expansion"
echo datei_{1,2,3}.bak
echo {a..e}
echo "--- Muster ohne Treffer"
echo *.xyz                 # bleibt unveraendert stehen!

echo "--- sicher in Schleife"
for f in *.txt; do
    printf 'Datei: [%s]\n' "$f"
done
cd .. && rm -r demo
```

Das Beispiel legt Testdateien an und zeigt die Muster. Beachten Sie zwei Besonderheiten. Erstens ersetzt `*.txt` die Datei mit Leerzeichen durch *einen* Eintrag; die Shell expandiert zuerst, *danach* wird nicht erneut an Leerzeichen getrennt. In der Schleife ist `"$f"` deshalb korrekt, solange die Variable gequotet wird. Zweitens bleibt ein Muster ohne Treffer standardmäßig als Text stehen (`echo *.xyz` gibt `*.xyz` aus). Eine Schleife über `*.xyz` würde dann einmal mit dem Text `*.xyz` als Dateiname laufen, was zu Fehlern führt. Abhilfe schafft `shopt -s nullglob` (Muster ohne Treffer ergibt „nichts“) oder die Prüfung `[ -e "$f" ] || continue` am Anfang des Schleifenkörpers.

Dateien mit führendem Punkt werden von `*` nicht erfasst. Wollen Sie diese einschließen, aktivieren Sie `shopt -s dotglob`. Beachten Sie auch, dass Dateinamen alles enthalten dürfen außer `/` und dem Nullzeichen, also auch Leerzeichen, Zeilenumbrüche und führende Bindestriche. Ein Dateiname wie `-rf` würde bei `rm *` als Option gelesen. Die sichere Schreibweise lautet `rm -- *` oder `rm ./*`; das doppelte Minus signalisiert „ab hier folgen keine Optionen mehr“.

```bash
# falsch: Dateiname mit Leerzeichen zerfaellt in zwei Argumente
for f in $(ls *.txt); do cat $f; done

# richtig: Glob direkt verwenden und Variable quoten
for f in *.txt; do cat -- "$f"; done
```

Die Ausgabe von `ls` zu parsen, ist ein klassischer Anfängerfehler: `ls` ist für Menschen gedacht, und die Kommandosubstitution trennt an Leerzeichen und Zeilenumbrüchen. Die Schleife direkt über das Glob-Muster ist kürzer und sicher.

> **Merke**
>
> Die Shell, nicht das Programm, expandiert Platzhalter. Muster ohne Treffer bleiben ohne `nullglob` stehen. Versteckte Dateien erfasst `*` nicht. Nie die Ausgabe von `ls` parsen; Dateinamen immer quoten und mit `--` absichern.

> **Praxisbeispiel**
>
> Ein Log-Aufräumskript soll `/var/log/app/*.log` verarbeiten. Nach einer Rotation ohne Treffer läuft die Schleife trotzdem einmal mit dem Wort `*.log` und meldet „No such file“. Mit `nullglob` oder einer Existenzprüfung ist das Skript auch an Tagen ohne Logs ruhig.

## 2.3 Dateioperationen

Die Grundoperationen auf Dateien sind `cp` (kopieren), `mv` (verschieben oder umbenennen), `rm` (löschen), `mkdir` (Verzeichnis anlegen) und `ln` (Verknüpfung erzeugen). Sie wirken unspektakulär, richten aber den größten Schaden an, weil die Shell *nicht nachfragt*: Unter Unix gibt es keinen Papierkorb auf der Kommandozeile. Was `rm` entfernt, ist weg.

`cp quelle ziel` kopiert eine Datei. Für Verzeichnisse braucht es `-r` (rekursiv). Häufig nützlich ist `-a` (Archivmodus), das Rechte, Besitzer und Zeitstempel erhält und Links als Links behält; `-i` fragt vor dem Überschreiben, `-n` überschreibt nie, `-v` zeigt, was passiert. `mv` verschiebt innerhalb eines Dateisystems nur den Verzeichniseintrag, ist also unabhängig von der Dateigröße praktisch sofort; über Dateisystemgrenzen hinweg kopiert es und löscht dann. Es dient zugleich zum Umbenennen. `mkdir -p a/b/c` legt die komplette Kette an und meldet keinen Fehler, wenn sie schon existiert, was für Skripte ideal ist (Idempotenz, siehe Modul 6).

`rm datei` löscht eine Datei, `rm -r verz` ein Verzeichnis samt Inhalt, `-f` unterdrückt Rückfragen und Fehler für nicht vorhandene Dateien, `-i` fragt nach. Die Kombination `rm -rf` ist das mächtigste und gefährlichste Kommando im Alltag. Vor jeder rekursiven Löschung sollten Sie das Ziel zuerst mit `ls` oder `echo` *ansehen*, also denselben Ausdruck ohne `rm` ausführen.

Zu `ln`: Es gibt zwei Arten von Verknüpfungen. Ein *Hardlink* (`ln quelle ziel`) ist ein zweiter Name für dieselben Daten (*Inode*); beide Namen sind gleichberechtigt, und die Daten verschwinden erst, wenn der letzte Name gelöscht ist. Hardlinks gehen nicht über Dateisystemgrenzen und nicht für Verzeichnisse. Ein *Symlink* (symbolischer Link, `ln -s ziel linkname`) ist dagegen eine eigene kleine Datei, die lediglich einen Pfad enthält. Er kann auf Verzeichnisse und andere Dateisysteme zeigen, kann aber „ins Leere“ zeigen, wenn das Ziel verschwindet (ein *dangling link*). Symlinks sind verbreitet, etwa bei Deployments: `current` zeigt auf das neueste Release-Verzeichnis, und ein Umschalten des Links ist ein sehr schneller, fast atomarer Vorgang.

```bash
#!/usr/bin/env bash
# dateiops.sh - sichere Dateioperationen im Testverzeichnis
set -u
arbeit="$(mktemp -d)"            # frisches, eigenes Verzeichnis
cd "$arbeit" || exit 1

mkdir -p projekt/{src,doc,build}
echo "Hallo" > projekt/src/main.txt

cp -a projekt/src/main.txt projekt/doc/kopie.txt   # Kopie mit Attributen
mv projekt/doc/kopie.txt projekt/doc/readme.txt    # Umbenennen
ln -s src/main.txt projekt/link-zu-main            # Symlink (relativ)
ln projekt/src/main.txt projekt/src/hartlink.txt   # Hardlink

ls -li projekt/src                 # -i zeigt die Inode-Nummer
echo "--- vor dem Loeschen kontrollieren:"
ls -d projekt/build projekt/doc/*
rm -r -- projekt/build             # gezielt ein Verzeichnis entfernen

find projekt -type l -exec ls -l {} +   # welche Links gibt es?

cd / && rm -rf -- "$arbeit"        # Aufraeumen
```

`mktemp -d` erzeugt ein neues, eindeutig benanntes temporäres Verzeichnis, sodass das Skript keine fremden Daten berührt. Die Brace-Expansion `projekt/{src,doc,build}` legt in einem Zug drei Unterordner an. `cp -a` kopiert mit allen Attributen, `mv` benennt um. Beim Symlink ist der Pfad *relativ zum Ort des Links* zu lesen: `src/main.txt` gilt innerhalb von `projekt/`, nicht im aktuellen Verzeichnis, weshalb relative Symlinks häufig falsch angelegt werden. `ls -li` zeigt in der ersten Spalte die Inode-Nummer: `main.txt` und `hartlink.txt` haben dieselbe, weil sie dieselben Daten bezeichnen. Vor dem Löschen kontrolliert `ls -d`, was die Muster treffen. Am Ende räumt das Skript auf; der Wechsel nach `/` verhindert, dass es das Verzeichnis löscht, in dem es gerade steht.

Typische Stolperfallen: Erstens, die Ziel-Semantik von `cp` und `mv`. Existiert das Ziel als Verzeichnis, wird die Datei *hinein* kopiert; existiert es nicht, entsteht eine Datei mit diesem Namen. Zweitens überschreiben `cp` und `mv` ohne Warnung. Falsch: `mv bericht.txt archiv` (wenn `archiv` eine Datei ist, ist sie nun überschrieben). Richtig: `mv -i bericht.txt archiv/` (der Schrägstrich am Ende erzwingt, dass `archiv` ein Verzeichnis sein muss, sonst gibt es einen Fehler). Drittens birgt das Löschen mit Variablen die Gefahr leerer Werte, wie in Lerneinheit 2.1 gezeigt; `rm -rf "${verz:?}"/` ist die defensive Variante.

> **Merke**
>
> Es gibt keinen Papierkorb. Rekursive Löschbefehle immer erst trocken ausprobieren (mit `ls` oder `echo`), Variablen in Pfaden gegen leere Werte absichern (`${var:?}`) und `--` vor Dateinamen setzen. `mkdir -p` ist idempotent, `rm -f` ebenfalls.

> **Praxisbeispiel**
>
> Ein Deployment legt jedes Release unter `/srv/app/releases/2026-10-30_1200` ab und lässt den Symlink `/srv/app/current` auf das aktive Release zeigen. Ein Rollback besteht darin, den Link auf das vorherige Release zu setzen (`ln -sfn alt-release current`). Die Dateien selbst werden nicht angefasst.

## 2.4 Berechtigungen

Linux ist ein Mehrbenutzersystem, und das Dateisystem schützt Daten über ein Rechtemodell. Jede Datei gehört einem *Besitzer* (User) und einer *Gruppe*. Daraus ergeben sich drei Personenkreise: der Besitzer (*u*, user), die Gruppe (*g*, group) und alle anderen (*o*, others). Für jeden dieser Kreise gibt es drei Rechte: *lesen* (`r`, read), *schreiben* (`w`, write) und *ausführen* (`x`, execute). Zusammen ergibt das neun Rechte-Bits, die `ls -l` als Text zeigt, zum Beispiel `-rwxr-xr--`.

Das erste Zeichen dieser Anzeige ist der Dateityp: `-` für gewöhnliche Datei, `d` für Verzeichnis, `l` für Symlink. Es folgen drei Dreiergruppen für u, g und o. Im Beispiel darf der Besitzer lesen, schreiben und ausführen (`rwx`), die Gruppe lesen und ausführen (`r-x`), alle anderen nur lesen (`r--`). Die Rechte haben bei Verzeichnissen eine andere Bedeutung als bei Dateien: `r` erlaubt das Auflisten der Namen, `w` das Anlegen, Umbenennen und Löschen von Einträgen, und `x` das *Betreten* und den Zugriff auf enthaltene Dateien. Ohne `x` auf einem Verzeichnis ist der Zugriff auf seinen Inhalt auch dann gesperrt, wenn die Datei selbst lesbar ist.

Rechte lassen sich auch als *Oktalzahl* schreiben: `r`=4, `w`=2, `x`=1, pro Personenkreis addiert. `rwxr-xr--` ist also 7 (4+2+1), 5 (4+1), 4, zusammen `754`. `chmod 754 datei` setzt genau diese Rechte. Daneben gibt es die symbolische Form: `chmod u+x datei` fügt dem Besitzer das Ausführungsrecht hinzu, `chmod g-w datei` entzieht der Gruppe das Schreiben, `chmod o= datei` nimmt allen anderen alle Rechte, und `chmod a+r datei` gibt allen Leserecht. Der Befehl `chmod -R` arbeitet rekursiv.

Besitzer und Gruppe ändern `chown benutzer:gruppe datei` und `chgrp gruppe datei`. Das Ändern des Besitzers ist normalerweise dem Administrator (`root`) vorbehalten. Zusätzlich gibt es die *umask*, eine Maske, die festlegt, welche Rechte neue Dateien *nicht* bekommen. Standard ist meist `022`: Neue Dateien starten mit `666` minus Maske, also `644`; neue Verzeichnisse mit `777` minus Maske, also `755`. Eine restriktivere Maske `077` gibt nur dem Besitzer Zugriff, was für sensible Skripte sinnvoll ist.

```bash
#!/usr/bin/env bash
# rechte-demo.sh
arbeit="$(mktemp -d)" && cd "$arbeit" || exit 1

touch normal.txt
echo '#!/usr/bin/env bash
echo "ich laufe"' > tool.sh

ls -l normal.txt tool.sh        # Standardrechte (meist 644)
./tool.sh                       # scheitert: Permission denied
chmod u+x tool.sh               # Ausfuehrungsrecht fuer den Besitzer
./tool.sh                       # laeuft jetzt

chmod 600 normal.txt            # nur Besitzer darf lesen/schreiben
stat -c '%a %A %U:%G %n' normal.txt tool.sh   # Rechte in Oktal und Text

echo "umask aktuell: $(umask)"
( umask 077; touch geheim.txt; mkdir geheim-verz )   # Subshell: umask nur dort
ls -ld geheim.txt geheim-verz

mkdir offen && chmod 755 offen
chmod a-x offen
ls offen 2>&1 | head -n 1       # Verzeichnis ohne x: kein Zugriff ...
chmod a+x offen

cd / && rm -rf -- "$arbeit"
```

Die Standardrechte neuer Dateien enthalten kein `x`, deshalb schlägt der erste Aufruf von `./tool.sh` mit „Permission denied“ fehl. `chmod u+x` behebt das. `stat -c` gibt die Rechte formatiert aus; `%a` liefert die Oktalzahl, `%A` den Text. Die Klammern `( ... )` erzeugen eine *Subshell*: Die Änderung der `umask` gilt nur dort und lässt die Einstellung der aufrufenden Shell unberührt, ein Vorgriff auf Modul 4. Zum Schluss sieht man, dass ein Verzeichnis ohne `x` nicht benutzbar ist.

Zwei klassische Fehler: Erstens `chmod 777`. Es gibt allen Benutzern volle Rechte und löst Zugriffsprobleme scheinbar mühelos, öffnet aber Sicherheitslücken und verdeckt die eigentliche Ursache; richtig ist, gezielt die benötigten Rechte zu vergeben und Besitzer/Gruppe korrekt zu setzen. Zweitens `chmod -R 644 verzeichnis`: Das entzieht auch Verzeichnissen das `x`-Recht und macht sie unbenutzbar. Richtig ist die Unterscheidung mit `find`: Verzeichnisse auf `755`, Dateien auf `644` setzen, oder das symbolische große `X` verwenden, das `x` nur auf Verzeichnisse (und bereits ausführbare Dateien) setzt: `chmod -R u=rwX,go=rX verzeichnis`.

> **Merke**
>
> Drei Kreise (u, g, o) mal drei Rechte (r, w, x); oktal 4, 2, 1. Bei Verzeichnissen heißt `x` „betreten“. Die umask zieht Rechte von neuen Dateien ab. Vergeben Sie das *minimal nötige* Recht und vermeiden Sie 777.

> **Praxisbeispiel**
>
> Ein Skript mit Zugangsdaten liegt in der Konfigurationsdatei `/etc/app/secrets.env`. Mit `640` und Gruppe `app` können nur der Dienst und Administratoren lesen. Wer bei einer Fehlersuche stattdessen `chmod 644` oder `777` setzt, gibt allen Benutzern des Systems Zugriff auf Passwörter. Siehe auch Modul 8 (Security).

## 2.5 Textanzeige

Fast alles unter Linux ist Text: Konfigurationsdateien, Logs, Skripte, Ausgaben von Befehlen. Wer Text schnell und gezielt ansehen kann, spart enorm Zeit. Vier Werkzeuge decken die meisten Fälle ab: `cat`, `less`, `head` und `tail`.

`cat` (von „concatenate“, aneinanderhängen) gibt Dateien nacheinander auf stdout aus. Der Name verrät die eigentliche Aufgabe: mehrere Dateien zusammenzufügen (`cat teil1 teil2 > ganz`). Zum bloßen Ansehen kleiner Dateien ist es praktisch; bei großen Dateien flutet es das Terminal. Nützliche Optionen sind `-n` (Zeilennummern) und `-A` (sichtbar machen von Tabs als `\^{}I` und Zeilenenden als `$`, hilfreich beim Aufspüren unsichtbarer Zeichen wie Windows-Zeilenenden). Der verbreitete Aufruf `cat datei | grep muster` ist meist überflüssig, denn `grep muster datei` erledigt es direkt; man spricht scherzhaft vom „Useless Use of Cat“.

`less` ist ein *Pager*: Er zeigt Text seitenweise an, erlaubt das Blättern in beide Richtungen und liest große Dateien, ohne sie komplett zu laden. Bedienung: Leertaste oder `f` eine Seite vor, `b` zurück, `g` an den Anfang, `G` ans Ende, `/muster` sucht vorwärts, `?muster` rückwärts, `n` und `N` springen zum nächsten oder vorherigen Treffer, `q` beendet. Mit `F` (oder dem Aufruf `less +F datei`) folgt `less` dem Dateiende wie `tail -f`; `Strg+C` stoppt das Folgen. Die Option `-S` schaltet den Zeilenumbruch ab, praktisch für breite Tabellen.

`head` zeigt den Anfang einer Datei (standardmäßig zehn Zeilen), `tail` das Ende. Mit `-n 20` bestimmt man die Zeilenzahl; `tail -n +2` gibt alles ab Zeile 2 aus (nützlich, um eine Kopfzeile zu überspringen). Besonders wichtig ist `tail -f` (follow): Es zeigt neu angehängte Zeilen laufend an und ist das Standardwerkzeug, um ein Logfile live zu beobachten. `tail -F` folgt zusätzlich über Logrotation hinweg, indem es die Datei neu öffnet, sobald sie ersetzt wurde.

```bash
#!/usr/bin/env bash
# textanzeige-demo.sh
datei="$(mktemp)"
seq 1 100 | sed 's/^/Zeile /' > "$datei"      # 100 Beispielzeilen

echo "--- erste 3 Zeilen";   head -n 3 "$datei"
echo "--- letzte 2 Zeilen";  tail -n 2 "$datei"
echo "--- ab Zeile 99";      tail -n +99 "$datei"
echo "--- Zeilen 50 bis 52"; head -n 52 "$datei" | tail -n 3
echo "--- mit Nummern";      head -n 2 "$datei" | cat -n

printf 'a\tb\r\n' > crlf.txt                  # Tab und Windows-Zeilenende
echo "--- cat -A macht Unsichtbares sichtbar"
cat -A crlf.txt                               # a^Ib^M$

# Live-Beobachtung simulieren (nur kurz)
( for i in 1 2 3; do echo "Ereignis $i" >> "$datei"; sleep 0.2; done ) &
timeout 1 tail -n 0 -f "$datei"
wait
rm -f "$datei" crlf.txt
```

Das Beispiel erzeugt mit `seq` 100 Zahlen und stellt per `sed` das Wort „Zeile“ voran. `head -n 52 | tail -n 3` ist der klassische Trick für einen Zeilenbereich: erst die ersten 52 Zeilen, davon die letzten 3. In `cat -A` erscheint `\^{}I` für den Tab, `\^{}M` für das Wagenrücklaufzeichen (Windows-Zeilenende) und `$` für das Zeilenende. Am Schluss schreibt ein Hintergrundprozess (das `&`, siehe Modul 4) einige Zeilen, während `tail -f` unter `timeout` eine Sekunde zuschaut.

Fehlerquellen liegen oft in der Dateigröße. Falsch: `cat /var/log/syslog` auf einem System mit Gigabyte-Logs, das Terminal läuft minutenlang voll. Richtig: `less /var/log/syslog`, `tail -n 100` oder gezielt mit `grep` filtern. Auch `tail -f` kann täuschen: Nach einer Logrotation verfolgt es die alte, umbenannte Datei und zeigt nichts Neues mehr; hier ist `tail -F` die richtige Wahl. Eine dritte Falle betrifft Binärdateien: `cat` auf eine Binärdatei kann das Terminal in einen unlesbaren Zustand versetzen. Mit `reset` lässt sich das beheben; zum gefahrlosen Ansehen eignen sich `file` (Dateityp bestimmen), `strings` (lesbare Textstücke) und `hexdump -C`.

> **Merke**
>
> Kleine Dateien `cat`, große `less`, Anfang `head`, Ende `tail`, live beobachten `tail -F`. `cat -A` enthüllt unsichtbare Zeichen. Vor dem Ausgeben unbekannter Dateien mit `file` den Typ prüfen.

> **Praxisbeispiel**
>
> Bei einem Ausfall lässt man in einem Terminal `tail -F /var/log/nginx/error.log` laufen und löst im anderen die fehlerhafte Anfrage aus. Die Fehlerzeile erscheint sofort im ersten Fenster, ohne dass man in der Datei suchen muss.

## 2.6 Streams und Pipes Grundlagen

Jeder Prozess unter Unix besitzt beim Start drei geöffnete Datenkanäle, *Standardströme* genannt. *stdin* (Dateideskriptor 0) ist die Standardeingabe, normalerweise die Tastatur. *stdout* (Deskriptor 1) ist die Standardausgabe, normalerweise der Bildschirm. *stderr* (Deskriptor 2) ist die Standardfehlerausgabe, ebenfalls normalerweise der Bildschirm, aber ein *getrennter* Kanal. Ein *Dateideskriptor* ist dabei eine kleine Nummer, mit der ein Prozess auf eine geöffnete Datei oder einen Datenstrom verweist.

Die Trennung von stdout und stderr ist beabsichtigt. Ergebnisse gehören auf stdout, Diagnosemeldungen und Fehler auf stderr. So lässt sich die Ausgabe in eine Datei oder an das nächste Programm weiterleiten, ohne dass Fehlermeldungen das Ergebnis verunreinigen, während Fehler weiterhin im Terminal sichtbar bleiben. Die Shell steuert die Verbindungen mit *Umleitungen* (Redirection). Mit `>` wird stdout in eine Datei umgeleitet (und diese dabei überschrieben), mit `>>` wird angehängt, mit `<` wird stdin aus einer Datei gelesen, und `2>` leitet stderr um. Die Schreibweise `2>&1` bedeutet „leite Kanal 2 dorthin, wohin Kanal 1 gerade zeigt“. Die Reihenfolge ist dabei entscheidend, wie die Beispiele gleich zeigen. Vertieft wird das in Modul 4.

Eine *Pipe* (Pipeline, geschrieben `|`) verbindet stdout des linken Befehls direkt mit stdin des rechten. Beide Programme laufen *gleichzeitig*; der Kernel puffert die Daten dazwischen. Es werden keine Zwischendateien angelegt, und ein Programm kann bereits mit der Verarbeitung beginnen, während das vorherige noch Daten liefert. Das ist die technische Umsetzung der Unix-Philosophie aus Modul 1. Beachten Sie, dass nur stdout durch die Pipe wandert; stderr geht weiterhin ans Terminal, sofern man es nicht ausdrücklich mit `2>&1` (oder in Bash kürzer `|&`) einbezieht.

```bash
#!/usr/bin/env bash
# streams-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
echo "Inhalt" > vorhanden.txt

# stdout und stderr getrennt erzeugen
ls vorhanden.txt gibtsnicht.txt > ausgabe.txt 2> fehler.txt
echo "--- ausgabe.txt:"; cat ausgabe.txt
echo "--- fehler.txt:";  cat fehler.txt

# beides in dieselbe Datei (Reihenfolge der Umleitungen beachten!)
ls vorhanden.txt gibtsnicht.txt > beides.txt 2>&1
echo "--- beides.txt:"; cat beides.txt

# Anhaengen statt ueberschreiben
echo "Zeile 1" > log.txt
echo "Zeile 2" >> log.txt
wc -l < log.txt                       # stdin aus Datei

# Fehler verwerfen
ls gibtsnicht.txt 2>/dev/null; echo "Exit-Code: $?"

# Pipeline: Wieviele Zeilen enthaelt eine Datei mit 'e'?
printf 'eins\nzwei\ndrei\nvier\n' | grep e | wc -l

# Aus mehreren Befehlen eine Ausgabe machen
{ echo "Kopf"; date +%F; } | cat -n

cd / && rm -rf -- "$verz"
```

Im ersten Block landen der Treffer in `ausgabe.txt` und die Fehlermeldung für die nicht vorhandene Datei in `fehler.txt`. Im zweiten Block zeigt `> beides.txt 2>&1` die richtige Reihenfolge: Erst wird stdout in die Datei gelegt, dann wird stderr auf *dasselbe Ziel wie stdout* gelegt. `/dev/null` ist das „schwarze Loch“ des Systems: Alles, was dorthin geschrieben wird, verschwindet. Der Exit-Code ist trotz verworfener Fehlermeldung ungleich null, die Meldung und das Ergebnis des Befehls sind also zwei verschiedene Dinge. Das Pipeline-Beispiel filtert Zeilen mit „e“ und zählt sie; das Ergebnis ist 4, weil alle vier Wörter ein „e“ enthalten. Die geschweiften Klammern gruppieren mehrere Befehle, deren gemeinsame Ausgabe in die Pipe geht.

Typische Fehler sind die falsche Reihenfolge und das versehentliche Überschreiben. Falsch ist `befehl 2>&1 > datei`: Hier wird stderr zuerst auf das *Terminal* gelegt (dort, wo stdout im Moment hinzeigt), und erst danach stdout in die Datei. Die Fehler erscheinen weiter im Terminal. Richtig ist `befehl > datei 2>&1`. Ein weiterer Klassiker: `sort datei > datei`. Die Shell öffnet und leert die Zieldatei, *bevor* `sort` sie lesen kann, und das Ergebnis ist eine leere Datei. Das Lesen und Schreiben derselben Datei in einem Befehl ist nie sicher; stattdessen schreibt man in eine temporäre Datei und benennt diese um (`sort datei > datei.tmp && mv datei.tmp datei`) oder nutzt Werkzeuge mit eigener Option wie `sort -o datei datei` oder `sed -i`.

> **Merke**
>
> 0 = stdin, 1 = stdout, 2 = stderr. Ergebnisse auf stdout, Meldungen auf stderr. Reihenfolge bei `> datei 2>&1` beachten. Eine Pipe verbindet stdout mit stdin, nicht stderr. Niemals dieselbe Datei im selben Befehl lesen und überschreiben.

> **Praxisbeispiel**
>
> Ein nächtlicher Job soll Ausgabe und Fehler getrennt protokollieren: `job.sh >> job.log 2>> job.err`. Im Fehlerfall schaut das Team zuerst in die kurze `job.err`, ohne sich durch die lange reguläre Ausgabe zu arbeiten.

## 2.7 Suchen im Dateisystem

Mit `find` durchsucht man Verzeichnisbäume nach Dateien anhand ihrer *Eigenschaften*, nicht ihres Inhalts: Name, Typ, Größe, Alter, Besitzer, Rechte. Die Grundform lautet `find STARTPFAD BEDINGUNGEN AKTION`. Ohne Aktion gibt `find` die Treffer aus (`-print`). Die wichtigsten Bedingungen sind `-name 'muster'` (Dateiname mit Glob-Muster, in Anführungszeichen, damit die Shell es nicht vorab expandiert), `-iname` (ohne Beachtung der Groß-/Kleinschreibung), `-type f` (Datei) oder `-type d` (Verzeichnis), `-size +10M` (größer als 10 Megabyte), `-mtime -7` (in den letzten 7 Tagen geändert) und `-mtime +30` (älter als 30 Tage), `-user`, `-perm` und `-empty`.

Bedingungen werden mit `-a` (und, implizit), `-o` (oder) und `!` (nicht) kombiniert; Klammern müssen vor der Shell geschützt werden (`\( ... \)`). Als Aktionen stehen `-print`, `-delete` (löschen), `-exec befehl {} \;` (Befehl für jeden Treffer, `{}` ist der Platzhalter) und `-exec befehl {} +` (Befehl mit vielen Treffern auf einmal, wesentlich schneller) zur Verfügung. Mit `-maxdepth` begrenzt man die Tiefe.

Für die Massenverarbeitung ergänzt `xargs` das Werkzeug: Es liest Zeilen oder Wörter von stdin und baut daraus Befehlsaufrufe. Der Aufruf `find . -name '*.log' | xargs rm` funktioniert nur, solange keine Dateinamen Leerzeichen oder Sonderzeichen enthalten, denn `xargs` trennt standardmäßig an Whitespace. Sicher wird es, wenn Trennzeichen das Nullbyte ist, das in Dateinamen nie vorkommt: `find . -name '*.log' -print0 | xargs -0 rm --`. Mit `xargs -P 4` laufen bis zu vier Prozesse parallel (siehe Modul 10).

```bash
#!/usr/bin/env bash
# find-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
mkdir -p logs/alt logs/neu
echo "a" > logs/neu/app.log
echo "bb" > "logs/neu/mit leerzeichen.log"
echo "ccc" > logs/alt/alt.log
touch -d '45 days ago' logs/alt/alt.log        # kuenstlich altern lassen
head -c 2048 /dev/zero > logs/neu/gross.bin

echo "--- alle .log-Dateien"
find logs -type f -name '*.log'

echo "--- aelter als 30 Tage"
find logs -type f -mtime +30

echo "--- groesser als 1 KiB"
find logs -type f -size +1k

echo "--- .log UND juenger als 30 Tage, mit Aktion"
find logs -type f -name '*.log' -mtime -30 -exec ls -l {} +

echo "--- sicheres Massenloeschen alter Dateien (Trockenlauf)"
find logs -type f -name '*.log' -mtime +30 -print0 | xargs -0 -r echo rm --

echo "--- Zeilen zaehlen ueber mehrere Dateien"
find logs -type f -name '*.log' -print0 | xargs -0 wc -l

cd / && rm -rf -- "$verz"
```

`touch -d` setzt ein künstlich altes Änderungsdatum. Die Bedingungen werden wie oben beschrieben kombiniert; `-exec ls -l {} +` ruft `ls` einmal mit allen Treffern auf. Der Trockenlauf stellt `echo` vor `rm`: Erst wenn die Ausgabe stimmt, entfernt man das `echo`. Die Option `-r` (`--no-run-if-empty`) verhindert, dass `xargs` bei null Treffern den Befehl einmal ohne Argumente startet. Dass Datei `mit leerzeichen.log` korrekt verarbeitet wird, verdankt das Beispiel `-print0` und `-0`.

Häufige Fehler: Das Muster ohne Anführungszeichen. Falsch: `find . -name *.log`. Liegt im aktuellen Verzeichnis eine Datei `a.log`, expandiert die Shell das Muster schon *vor* `find` zu `-name a.log`, und das Ergebnis wird falsch oder bricht mit „paths must precede expression“ ab. Richtig: `find . -name '*.log'`. Ein zweiter Fehler ist Löschen ohne Vorschau: Wer `-delete` *vor* die Bedingungen schreibt, löscht alles, denn `find` wertet die Ausdrücke von links nach rechts aus. Richtig ist, zuerst ohne Aktion zu testen und `-delete` als letztes anzuhängen. Drittens wird `find` gern mit Inhaltssuche verwechselt: Wer *in* Dateien suchen will, nimmt `grep -r`, wie in Modul 5 gezeigt.

> **Merke**
>
> `find` sucht nach Eigenschaften, `grep` nach Inhalt. Muster in Anführungszeichen setzen. Für sichere Massenoperationen `-print0` mit `xargs -0` oder gleich `-exec ... {} +` verwenden und zuerst einen Trockenlauf machen.

> **Praxisbeispiel**
>
> Ein Wartungsskript löscht Backup-Archive, die älter als 30 Tage sind: `find /srv/backup -type f -name '*.tar.gz' -mtime +30 -delete`. Vor dem Produktiveinsatz läuft derselbe Befehl ohne `-delete` und das Ergebnis wird im Review kontrolliert.

## 2.8 Archivierung und Kompression

Zwei Aufgaben werden oft vermischt: das *Archivieren*, also das Bündeln vieler Dateien samt Verzeichnisstruktur und Metadaten in eine einzige Datei, und das *Komprimieren*, also das Verkleinern von Daten durch Entfernen von Redundanz. Unter Unix erledigen das traditionell zwei getrennte Werkzeuge, die man kombiniert: `tar` (Tape Archive) bündelt, und ein Kompressor wie `gzip`, `bzip2` oder `xz` verkleinert. Die Endung `.tar.gz` (oder `.tgz`) beschreibt genau das.

Die wichtigsten `tar`-Optionen sind `-c` (create: Archiv erstellen), `-x` (extract: auspacken), `-t` (list: Inhalt anzeigen), `-f datei` (Name der Archivdatei, *muss* direkt vor dem Namen stehen), `-v` (verbose: zeigt, was passiert), `-z` (mit gzip komprimieren), `-J` (xz) und `-C verz` (vor der Arbeit in dieses Verzeichnis wechseln). Die Merkhilfe für die gängigsten Aufrufe: `tar -czf archiv.tar.gz verz/` zum Packen, `tar -tzf archiv.tar.gz` zum Ansehen, `tar -xzf archiv.tar.gz -C ziel/` zum Entpacken. `gzip datei` ersetzt die Datei durch `datei.gz`, `gunzip` dreht das um, und `zcat` liest komprimierte Textdateien, ohne sie zu entpacken. Für den Austausch mit Windows-Nutzern ist `zip` (`zip -r archiv.zip verz/`, `unzip archiv.zip`) verbreiteter.

Für *reproduzierbare* Archive, etwa Release-Artefakte, ist wichtig, dass derselbe Inhalt zu derselben Datei führt. Das gelingt mit sortierten Dateinamen, festen Zeitstempeln und Eigentümern, zum Beispiel mit `tar --sort=name --mtime='2026-01-01 UTC' --owner=0 --group=0 --numeric-owner`. Bei `gzip` wird ein Zeitstempel im Kopf abgelegt; `gzip -n` lässt ihn weg.

```bash
#!/usr/bin/env bash
# archiv-demo.sh
set -u
quelle="$(mktemp -d)" && cd "$quelle" || exit 1
mkdir -p projekt/src
echo "print('hallo')" > projekt/src/app.py
echo "# Readme" > projekt/README.md

# Packen: -C wechselt vorher, damit im Archiv KEIN absoluter Pfad steht
tar -czf ../projekt.tar.gz -C "$quelle" projekt
cd .. || exit 1

echo "--- Inhalt ansehen (ohne zu entpacken)"
tar -tzf projekt.tar.gz

echo "--- Kompressionsvergleich"
ls -l projekt.tar.gz | awk '{print $5, "Bytes (gz)"}'

echo "--- in anderes Verzeichnis entpacken"
ziel="$(mktemp -d)"
tar -xzf projekt.tar.gz -C "$ziel"
find "$ziel" -type f

echo "--- Vergleich Original/Entpackt"
diff -r "$quelle/projekt" "$ziel/projekt" && echo "identisch"

rm -rf -- "$quelle" "$ziel" projekt.tar.gz
```

Die Option `-C` ist hier der Schlüssel: Packt man `tar -czf x.tar.gz /home/anna/projekt`, enthalten die Einträge den vollen Pfad, den `tar` beim Packen (in neueren Versionen) zwar um den führenden Schrägstrich kürzt, was aber trotzdem lange verschachtelte Pfade erzeugt. Mit `-C pfad name` bleibt im Archiv nur `projekt/...`. Das Auflisten mit `-t` vor dem Entpacken ist eine bewährte Vorsichtsmaßnahme, denn ein Archiv kann Einträge mit `..` oder absoluten Pfaden enthalten, die beim Entpacken fremde Dateien überschreiben würden (der *Zip-Slip*- bzw. *Tar-Slip*-Angriff). `diff -r` vergleicht am Ende beide Verzeichnisbäume rekursiv.

Beliebte Fehler: Erstens die Reihenfolge der Optionen. Falsch: `tar -cfz archiv.tar.gz verz`; hier wird `z` als Dateiname von `-f` gelesen, und das Archiv heißt „z“. Richtig: `tar -czf archiv.tar.gz verz` (Option `f` zuletzt). Zweitens, ein Archiv „in sich selbst“ zu packen: `tar -czf backup.tar.gz .` im selben Verzeichnis nimmt die entstehende Archivdatei mit auf; besser an einen anderen Ort schreiben oder `--exclude` nutzen. Drittens, ein Archiv ohne Prüfung im aktuellen Ordner zu entpacken, wodurch dort Hunderte Dateien verstreut werden; deshalb vorher `tar -tf` und gezielt `-C` verwenden.

> **Merke**
>
> Archivieren (`tar`) und Komprimieren (`gzip`, `xz`) sind zwei Schritte. Merkregel: `c`reate, `x`tract, `t`est/list, `f`ile zuletzt. Archive vor dem Entpacken mit `-t` ansehen und mit `-C` in ein Zielverzeichnis entpacken.

> **Praxisbeispiel**
>
> Ein Backup-Skript erstellt täglich `backup-2026-10-30.tar.gz`, legt daneben eine Prüfsumme ab (siehe nächste Lerneinheit) und löscht Archive, die älter als 14 Tage sind. Beim Restore beweist ein Test auf einem Ersatzsystem, dass das Archiv vollständig ist; erst dann gilt das Backup als echt.

## 2.9 Checksummen

Eine *Prüfsumme* (Hash) ist ein kurzer „Fingerabdruck“ beliebig langer Daten, berechnet durch eine Hashfunktion. Dieselben Daten ergeben immer dieselbe Prüfsumme; schon die Änderung eines einzigen Bits führt zu einem völlig anderen Wert. Damit lässt sich prüfen, ob eine Datei unverändert angekommen ist, nach dem Download, nach dem Kopieren oder nach einer Wiederherstellung. Das Programm `sha256sum` berechnet den SHA-256-Hash (256 Bit, als 64 Hexadezimalzeichen geschrieben). Ältere Verfahren wie MD5 (`md5sum`) und SHA-1 gelten als kryptografisch gebrochen und eignen sich nicht mehr als Schutz vor *absichtlicher* Manipulation, sind aber zum Erkennen *zufälliger* Beschädigung noch gebräuchlich.

Typischer Ablauf: Der Anbieter veröffentlicht neben der Datei eine Prüfsummendatei, etwa `SHA256SUMS`, deren Zeilen aus Hash, zwei Leerzeichen und Dateiname bestehen. Mit `sha256sum -c SHA256SUMS` liest das Programm diese Zeilen, berechnet die Hashes der Dateien neu und meldet pro Datei `OK` oder `FAILED`. Für den Vergleich zweier Dateien byteweise gibt es `cmp`, das beim ersten Unterschied abbricht und bei Gleichheit schweigt (Exit-Code 0). Zeilenweise Unterschiede zeigt `diff`.

Beachten Sie die Grenzen: Eine Prüfsumme beweist Unversehrtheit nur, wenn man der Prüfsumme selbst *vertraut*. Liegt sie auf demselben Server wie die Datei, kann ein Angreifer beide austauschen. Echten Schutz bietet eine kryptografische *Signatur* (GPG), die in Modul 8 (Sichere Downloads) behandelt wird.

```bash
#!/usr/bin/env bash
# checksum-demo.sh
verz="$(mktemp -d)" && cd "$verz" || exit 1
echo "Wichtige Daten" > a.txt
cp a.txt b.txt
echo "Wichtige Daten!" > c.txt          # nur ein Zeichen anders

echo "--- Hashes berechnen"
sha256sum a.txt b.txt c.txt             # a und b gleich, c voellig anders

echo "--- Pruefsummendatei erzeugen und pruefen"
sha256sum a.txt b.txt c.txt > SHA256SUMS
sha256sum -c SHA256SUMS                  # alle OK

echo "--- Manipulation simulieren"
echo "Manipuliert" >> a.txt
sha256sum -c SHA256SUMS || echo "Pruefung fehlgeschlagen (Exit: $?)"

echo "--- nur den Hash in einer Variable"
hash="$(sha256sum c.txt | cut -d' ' -f1)"
echo "Hash von c.txt: $hash"

echo "--- byteweise vergleichen"
cmp b.txt c.txt || echo "b.txt und c.txt unterscheiden sich"
cmp -s b.txt b.txt && echo "identisch"

cd / && rm -rf -- "$verz"
```

Die Zeile `sha256sum -c SHA256SUMS` gibt für jede Datei `OK` oder `FAILED` aus und liefert bei einem Fehler einen von null verschiedenen Exit-Code, den das Beispiel mit `||` abfängt. Die nachträglich veränderte Datei `a.txt` wird sofort erkannt. `cut -d' ' -f1` trennt die Ausgabe am Leerzeichen und behält das erste Feld, also den reinen Hash. `cmp -s` arbeitet still und eignet sich für Bedingungen in Skripten.

Typische Fehler: Erstens, die Prüfsumme der falschen Sache zu berechnen. Falsch: eine Prüfsumme über ein *Archiv* aus `tar -czf` zu veröffentlichen, das bei jedem Lauf wegen des Zeitstempels anders ausfällt; damit sind Vergleiche sinnlos. Richtig: reproduzierbare Archive (siehe 2.8) oder Prüfsumme der Eingabedateien. Zweitens, Prüfsummen mit `echo`-erzeugten Zeilen von Hand zu vergleichen: Das ist fehleranfällig; besser `sha256sum -c`, das den Vergleich selbst erledigt. Drittens wird die Textmodus-/Binärmodus-Markierung im Dateinamenfeld (ein Leerzeichen und ein Stern) von manchen Windows-Werkzeugen anders geschrieben; auf Linux ist dies ohne Bedeutung.

> **Merke**
>
> Prüfsummen erkennen Veränderungen, schützen aber nur, wenn die Prüfsumme selbst vertrauenswürdig ist. Für Integritätsprüfungen: `sha256sum -c`. Für den schnellen Vergleich zweier Dateien: `cmp -s`. MD5 und SHA-1 nicht für Sicherheit einsetzen.

> **Praxisbeispiel**
>
> Eine Deployment-Pipeline lädt ein Installationspaket herunter, prüft `sha256sum -c` gegen die im Repository hinterlegte Prüfsumme und bricht bei Abweichung hart ab. So wird kein manipuliertes oder unvollständig übertragenes Paket installiert.

## 2.10 Übungsaufgaben zu Modul 02

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit. Führen Sie praktische Aufgaben in einem Testverzeichnis aus (`mktemp -d`).

**Aufgabe 1.** Ordnen Sie zu, wo Sie folgende Dinge im Dateisystem finden: die Konfiguration des SSH-Servers, die Logdateien eines Webservers, Ihre persönlichen Dokumente, den Befehl `ls`, eine temporäre Datei, die nach dem Neustart verschwinden darf.

**Aufgabe 2.** Eine Datei hat die Rechte `-rwxr-x---`. Wer darf was? Wie lautet die Oktaldarstellung, und mit welchem `chmod`-Befehl in symbolischer Schreibweise erreichen Sie sie ausgehend von `-rw-r--r--`?

**Aufgabe 3.** Erklären Sie, warum der Befehl `sort daten.txt > daten.txt` die Datei leert, und nennen Sie zwei sichere Alternativen.

**Aufgabe 4.** Was ist der Unterschied zwischen Hardlink und Symlink? Was passiert mit beiden, wenn das ursprüngliche Ziel gelöscht wird?

**Aufgabe 5.** Schreiben Sie ein Skript `anlegen.sh`, das unterhalb eines als Argument übergebenen Verzeichnisses die Struktur `projekt/{src,doc,test}` anlegt, in jedes Unterverzeichnis eine `README.md` mit dem Verzeichnisnamen als Inhalt schreibt und die Rechte so setzt, dass Verzeichnisse `755` und Dateien `644` haben. Das Skript darf mehrfach laufen, ohne Fehler zu melden.

**Aufgabe 6.** Schreiben Sie ein Skript `logs-aufraeumen.sh`, das in einem Verzeichnis alle `.log`-Dateien findet, die älter als N Tage sind (N als zweites Argument, Standard 30). Standardmäßig soll es nur auflisten (Trockenlauf); mit der Option `--loeschen` soll es wirklich löschen. Dateien mit Leerzeichen im Namen müssen korrekt behandelt werden.

**Aufgabe 7.** Schreiben Sie ein Skript `sichern.sh`, das ein Verzeichnis als `.tar.gz` mit Datum im Namen in ein Zielverzeichnis packt, die SHA-256-Prüfsumme daneben ablegt und anschließend die Prüfsumme verifiziert. Bei Fehlern soll das Skript mit einem Fehlercode ungleich null enden.

**Aufgabe 8.** Schreiben Sie ein Skript `doppelte-dateien.sh`, das in einem Verzeichnis (rekursiv) Dateien mit identischem Inhalt anhand ihrer SHA-256-Summen findet und gruppiert ausgibt. Hinweis: Hashes berechnen, sortieren und mit `uniq -w64 -D` auf doppelte Summen filtern.

## 2.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** SSH-Server-Konfiguration: `/etc/ssh/sshd_config` (Konfiguration liegt in `/etc`). Webserver-Logs: `/var/log/nginx/` oder `/var/log/apache2/` (veränderliche Daten in `/var`). Persönliche Dokumente: `/home/\textit{benutzer}/`. Der Befehl `ls`: `/usr/bin/ls` (auf modernen Systemen auch über `/bin/ls` erreichbar, da `/bin` ein Verweis auf `/usr/bin` ist). Eine flüchtige temporäre Datei: `/tmp`.

**Lösung zu Aufgabe 2.** Besitzer: lesen, schreiben, ausführen; Gruppe: lesen und ausführen; alle anderen: nichts. Oktal: `750`. Ausgehend von `rw-r--r--` (`644`): `chmod u+x,g+x,o-r datei`, kurz auch `chmod 750 datei`. Beachten Sie, dass bei `g+x` das Leserecht der Gruppe erhalten bleibt, und `o-r` den anderen das Lesen nimmt.

**Lösung zu Aufgabe 3.** Die Shell wertet die Umleitung `>` *vor* dem Start von `sort` aus, öffnet die Zieldatei zum Schreiben und kürzt sie dabei auf Länge 0. Erst dann startet `sort` und liest eine leere Datei. Sichere Alternativen: `sort -o daten.txt daten.txt` (`sort` schreibt die Ausgabe erst nach dem vollständigen Einlesen) oder über eine temporäre Datei: `sort daten.txt > daten.tmp && mv daten.tmp daten.txt`.

**Lösung zu Aufgabe 4.** Ein Hardlink ist ein zweiter Verzeichniseintrag für denselben Inode, also dieselben Daten; beide Namen sind gleichwertig. Ein Symlink ist eine eigene Datei, die einen Pfad enthält. Löscht man das ursprüngliche Ziel, lebt der Hardlink weiter und liefert die Daten, denn sie existieren, solange mindestens ein Name auf sie zeigt. Der Symlink dagegen zeigt auf einen nun nicht mehr vorhandenen Pfad und ist ein „dangling link“, bei dem Zugriffe mit „No such file or directory“ scheitern.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# anlegen.sh BASISVERZEICHNIS
if [ "$#" -ne 1 ]; then
    echo "Aufruf: $0 BASISVERZEICHNIS" >&2
    exit 2
fi

basis="$1/projekt"
for teil in src doc test; do
    mkdir -p "$basis/$teil"
    printf '%s\n' "$teil" > "$basis/$teil/README.md"
done

find "$basis" -type d -exec chmod 755 {} +
find "$basis" -type f -exec chmod 644 {} +
echo "Struktur angelegt unter: $basis"
```

`mkdir -p` macht das Anlegen wiederholbar, und das erneute Schreiben der README mit `>` überschreibt den gleichen Inhalt, ändert also nichts. Die Rechte werden getrennt für Verzeichnisse und Dateien gesetzt, weil dasselbe `chmod` für beide Typen nicht passt. Ein Aufruf mit falscher Argumentzahl endet mit der Usage-Meldung auf stderr und Code 2.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# logs-aufraeumen.sh VERZEICHNIS [TAGE] [--loeschen]
verz="${1:-}"
tage="${2:-30}"
modus="${3:-}"

if [ -z "$verz" ] || [ ! -d "$verz" ]; then
    echo "Aufruf: $0 VERZEICHNIS [TAGE] [--loeschen]" >&2
    exit 2
fi
case "$tage" in
    ''|*[!0-9]*) echo "TAGE muss eine Zahl sein" >&2; exit 2 ;;
esac

if [ "$modus" = "--loeschen" ]; then
    find "$verz" -type f -name '*.log' -mtime "+$tage" -print -delete
else
    echo "Trockenlauf - diese Dateien wuerden geloescht:"
    find "$verz" -type f -name '*.log' -mtime "+$tage" -print
fi
```

Die `case`-Prüfung mit dem Muster `*[!0-9]*` erkennt jede Eingabe, die ein Nicht-Ziffern-Zeichen enthält, und das leere Muster `''` fängt leere Eingaben. Dadurch landet kein Unfug im `find`-Ausdruck. `-print -delete` gibt zuerst jede Datei aus und löscht sie dann; weil `find` alles selbst erledigt, spielen Leerzeichen im Namen keine Rolle.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# sichern.sh QUELLVERZEICHNIS ZIELVERZEICHNIS
set -euo pipefail

if [ "$#" -ne 2 ]; then
    echo "Aufruf: $0 QUELLE ZIEL" >&2
    exit 2
fi
quelle="${1%/}"
ziel="$2"
[ -d "$quelle" ] || { echo "Quelle fehlt: $quelle" >&2; exit 1; }
mkdir -p "$ziel"

name="$(basename "$quelle")-$(date +%F).tar.gz"
tar -czf "$ziel/$name" -C "$(dirname "$quelle")" "$(basename "$quelle")"

( cd "$ziel" && sha256sum "$name" > "$name.sha256" && sha256sum -c "$name.sha256" )
echo "Sicherung fertig: $ziel/$name"
```

`set -euo pipefail` lässt das Skript bei jedem Fehler abbrechen (Details in Modul 6). `${1%/}` schneidet einen eventuellen Schrägstrich am Ende ab. Das Archiv wird mit `-C` so gepackt, dass nur der Verzeichnisname im Archiv steht. Die Prüfsumme wird in einer Subshell berechnet, die ins Zielverzeichnis wechselt, damit in der Prüfsummendatei nur der Dateiname ohne Pfad steht; so funktioniert die spätere Prüfung an jedem Ort. Schlägt `sha256sum -c` fehl, endet das Skript durch `set -e` mit Fehlercode.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# doppelte-dateien.sh [VERZEICHNIS]
verz="${1:-.}"
[ -d "$verz" ] || { echo "Kein Verzeichnis: $verz" >&2; exit 1; }

find "$verz" -type f -print0 \
    | xargs -0 -r sha256sum \
    | sort \
    | uniq -w64 -D
```

`find -print0` liefert alle Dateien nullterminiert, `xargs -0` ruft `sha256sum` mit ihnen auf (auch mit Leerzeichen im Namen). Danach stehen Zeilen der Form „Hash, zwei Leerzeichen, Pfad“. `sort` bringt gleiche Hashes untereinander. `uniq -w64 -D` vergleicht nur die ersten 64 Zeichen (die Länge eines SHA-256-Hashs in Hexadezimaldarstellung) und gibt *alle* Zeilen aus, deren Hash mehrfach vorkommt. Eindeutige Dateien erscheinen nicht.
