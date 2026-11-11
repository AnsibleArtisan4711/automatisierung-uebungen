# Modul 05 - subprocess, argparse

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (5.1 bis 5.5), danach folgen „Übungsaufgaben zu Modul 05“ (8 Aufgaben) und „Lösungen zu Modul 05“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 24.11.2026 bis Mo 30.11.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; Programme, die andere Programme starten und selbst Kommandozeilenargumente verstehen, sind der Kern der Linux-Automatisierung)

**Meilenstein:** Wrapper für systemctl/df

**Lernziele.** Nach diesem Modul können Sie aus Python heraus beliebige Linux-Programme starten, deren Ausgabe und Rückgabecode auswerten und dabei die typischen Sicherheitsfallen (`shell=True`, Zeichenketten statt Listen, fehlende Zeitlimits) vermeiden. Sie können außerdem eigene Skripte mit `argparse` so bauen, dass sie sich wie echte Linux-Werkzeuge verhalten: mit `--help`, Optionen, Unterbefehlen und sinnvollen Rückgabecodes. Am Ende steht ein Wrapper, der `systemctl` und `df` kapselt.

| **Nr.** | **Lerneinheit**                               | **Datum**     | **Dauer** | **Prio** |
|:--------|:----------------------------------------------|:--------------|:----------|:---------|
| 5.1     | subprocess.run Grundlagen                     | Di 24.11.2026 | 2 h       | 1        |
| 5.2     | stdout/stderr, Rückgabecodes, ohne shell=True | Mi 25.11.2026 | 2 h       | 1        |
| 5.3     | argparse Grundlagen                           | Do 26.11.2026 | 2 h       | 1        |
| 5.4     | argparse Optionen & Subcommands               | Fr 27.11.2026 | 2 h       | 1        |
| 5.5     | Wrapper für systemctl/df + Quiz               | Mo 30.11.2026 | 2 h       | 1        |
|         | **Summe**                                     |               | **10 h**  |          |

> **Merke**
>
> In diesem Modul starten die Beispiele Programme. Damit sie auf jedem Rechner dasselbe zeigen, rufen sie meist den Python-Interpreter selbst als Kindprozess auf (`sys.executable`) oder benutzen harmlose Standardprogramme wie `echo`, `sort` und `uname`. Programme, die etwas verändern könnten, werden nur in einem temporären Verzeichnis oder als Attrappe (siehe Lerneinheit 5.5) ausgeführt. Vorsicht bei eigenen Experimenten: Alles, was Sie mit `subprocess` starten, hat dieselben Rechte wie Ihr Skript.

## 5.1 subprocess.run Grundlagen

Ein großer Teil der Systemadministration besteht darin, vorhandene Werkzeuge zu bedienen: `systemctl` startet und stoppt Dienste, `df` zeigt freien Plattenplatz, `tar` packt Archive, `ip` verrät Netzwerkadressen. Es wäre unsinnig, diese Programme in Python nachzubauen. Stattdessen soll ein Python-Skript sie *aufrufen*, ihnen Argumente mitgeben, abwarten, bis sie fertig sind, und danach auswerten, was sie ausgegeben haben und ob sie erfolgreich waren. Genau dafür gibt es das Modul `subprocess` der Standardbibliothek.

Um es zu verstehen, braucht man zuerst den Begriff des *Prozesses*. Wenn Linux ein Programm ausführt, entsteht ein Prozess: eine laufende Instanz des Programms mit einer eigenen Prozessnummer (PID), einem eigenen Speicher und einem eigenen Satz an Datenströmen. Jeder Prozess besitzt drei Standardströme: die *Standardeingabe* (`stdin`, Nummer 0), von der er Eingaben liest, die *Standardausgabe* (`stdout`, Nummer 1) für normale Ausgaben und die *Standardfehlerausgabe* (`stderr`, Nummer 2) für Fehlermeldungen. Im Terminal sind alle drei mit Tastatur und Bildschirm verbunden; man kann sie aber umleiten, etwa in eine Datei oder in einen anderen Prozess. Wenn ein Prozess endet, hinterlässt er einen *Rückgabecode* (auch Exit-Code oder Exit-Status genannt), eine Zahl zwischen 0 und 255. Nach einer langen Unix-Konvention bedeutet 0 „alles in Ordnung“, jeder andere Wert „es ist etwas schiefgegangen“. Welche Zahl welchen Fehler bedeutet, legt jedes Programm selbst fest.

Startet Ihr Python-Skript ein anderes Programm, so ist Ihr Skript der *Elternprozess* und das gestartete Programm der *Kindprozess*. Der Elternprozess entscheidet, womit die drei Ströme des Kindes verbunden werden, und kann auf dessen Ende warten. Genau das erledigt die Funktion `subprocess.run()`. Sie startet das Kind, wartet, bis es fertig ist, und liefert ein Objekt vom Typ `CompletedProcess` zurück, das die Argumente, den Rückgabecode und, falls gewünscht, die abgefangene Ausgabe enthält. Die Funktion gibt es seit Python 3.5; sie ist der empfohlene Weg für fast alle Aufgaben. Ältere Skripte verwenden `os.system()` oder `subprocess.call()`; beide können weniger und gelten als überholt. Für Sonderfälle, etwa wenn man die Ausgabe eines langlaufenden Programms Zeile für Zeile verarbeiten will, gibt es die Klasse `Popen`, auf der `run` aufbaut (mehr dazu in Lerneinheit 5.2).

Das einfachste Beispiel startet das Programm `echo`. Wichtig ist die Form des ersten Arguments: Es ist eine *Liste*, deren erstes Element der Programmname ist und deren weitere Elemente die Argumente sind, jedes Argument ein eigenes Listenelement.

```python
import subprocess

ergebnis = subprocess.run(["echo", "Hallo", "Welt"])
print("Rückgabecode:", ergebnis.returncode)
print(ergebnis.args)
```

```text
Hallo Welt
Rückgabecode: 0
['echo', 'Hallo', 'Welt']
```

Die erste Ausgabezeile stammt nicht von `print`, sondern vom Kindprozess `echo`: Er hat auf dieselbe Standardausgabe geschrieben wie das Skript. Danach liefert `run()` das `CompletedProcess`-Objekt, in dem `returncode` den Wert 0 hat und `args` die Liste enthält, mit der der Prozess gestartet wurde. Das Programm `echo` wird dabei über die Umgebungsvariable `PATH` gefunden, genau wie in der Shell; man kann auch einen absoluten Pfad wie `/bin/echo` angeben.

Meistens will man die Ausgabe des Kindes nicht auf dem Bildschirm sehen, sondern in Python weiterverarbeiten. Dazu gibt es zwei Parameter. Mit `capture_output=True` werden Standardausgabe und Standardfehlerausgabe in Python-Objekte umgeleitet, und mit `text=True` werden die Bytes, die ein Prozess tatsächlich liefert, automatisch in einen Text (`str`) umgewandelt. Ohne `text=True` erhalten Sie den Typ `bytes`, den Sie selbst mit `.decode()` in Text verwandeln müssten. Beide Parameter wurden mit Python 3.7 eingeführt; in Anleitungen für ältere Versionen finden Sie dafür `stdout=subprocess.PIPE` und `universal_newlines=True`, was dasselbe bewirkt.

```python
import subprocess

ergebnis = subprocess.run(["uname", "-s"], capture_output=True, text=True)

print(repr(ergebnis.stdout))
print(ergebnis.stdout.strip())
print(repr(ergebnis.stderr))
print(ergebnis.returncode)

roh = subprocess.run(["uname", "-s"], capture_output=True)
print(roh.stdout)
print(type(roh.stdout).__name__)
```

```text
'Linux\n'
Linux
''
0
b'Linux\n'
bytes
```

Das Programm `uname -s` gibt den Namen des Betriebssystems aus. Die Zeichenkette in `stdout` endet mit einem Zeilenumbruch, den Sie fast immer mit `.strip()` entfernen sollten, bevor Sie den Text vergleichen oder weiterverwenden. Die Fehlerausgabe `stderr` ist hier leer, weil das Programm nichts zu beanstanden hatte. Der zweite Aufruf ohne `text=True` zeigt den Unterschied: Das Ergebnis ist ein `bytes`-Objekt (erkennbar am Präfix `b`).

Neben diesen beiden Parametern gibt es einige weitere, die Sie im Alltag regelmäßig brauchen. Die folgende Tabelle fasst sie zusammen; die ausführliche Erklärung folgt im Anschluss an die Tabelle und in der nächsten Lerneinheit.

| **Parameter** | **Wirkung**                                                                        |
|:--------------|:-----------------------------------------------------------------------------------|
| `capture_output`      | Fängt `stdout` und `stderr` ab; die Texte stehen danach im Ergebnisobjekt.         |
| `text`      | Wandelt Ein- und Ausgabe automatisch zwischen Bytes und Text um.                   |
| `input`      | Text (oder Bytes), der an die Standardeingabe des Kindes gesendet wird.            |
| `check`      | Löst bei einem Rückgabecode ungleich 0 eine Ausnahme aus (Lerneinheit 5.2).        |
| `timeout`      | Höchstdauer in Sekunden; danach wird das Kind beendet und eine Ausnahme ausgelöst. |
| `cwd`      | Arbeitsverzeichnis, in dem das Kind startet.                                       |
| `env`      | Umgebungsvariablen des Kindes; ohne Angabe erbt es die des Skripts.                |
| `shell`      | Startet das Kommando über eine Shell. Standard ist `False`; lassen Sie es dabei.  |

Der Parameter `input` verbindet die Standardeingabe des Kindes mit einem Text aus Ihrem Skript, so als hätte jemand ihn getippt. Er eignet sich für Programme, die von der Standardeingabe lesen, zum Beispiel `sort` oder `grep`. Der Parameter `cwd` entspricht einem `cd` vor dem Befehl, ohne dass sich das Arbeitsverzeichnis Ihres eigenen Skripts ändert. Und `env` ist wichtig, wenn Sie die Sprache oder das Format der Ausgabe festnageln wollen: Viele Linux-Programme übersetzen ihre Meldungen je nach Spracheinstellung (`LANG`, `LC_ALL`); wer die Ausgabe automatisch auswertet, setzt deshalb `LC_ALL` auf `C`. Das folgende Beispiel zeigt alle drei Parameter in einem Programm, wobei das Kind ein kleines Python-Programm ist, damit das Ergebnis auf jedem Rechner gleich ausfällt. Der Pfad des laufenden Interpreters steht in `sys.executable`; ihn zu verwenden ist robuster, als sich auf `python3` im `PATH` zu verlassen, weil man so garantiert dieselbe Python-Version startet, die auch das Skript ausführt.

```python
import os
import subprocess
import sys
import tempfile
from pathlib import Path

# 1) input: das Kind liest von der Standardeingabe
kind = [sys.executable, "-c", "import sys; print(sys.stdin.read().upper(), end='')"]
r = subprocess.run(kind, input="hallo\nwelt\n", capture_output=True, text=True)
print(r.stdout)

# 2) cwd: das Kind startet in einem anderen Verzeichnis
with tempfile.TemporaryDirectory() as tmp:
    (Path(tmp) / "a.txt").write_text("a")
    (Path(tmp) / "b.txt").write_text("b")
    r = subprocess.run(["ls"], cwd=tmp, capture_output=True, text=True)
    print(sorted(r.stdout.split()))

# 3) env: eine zusätzliche Umgebungsvariable für das Kind
umgebung = {**os.environ, "MEINE_STUFE": "test"}
kind = [sys.executable, "-c", "import os; print(os.environ['MEINE_STUFE'])"]
r = subprocess.run(kind, env=umgebung, capture_output=True, text=True)
print(r.stdout.strip())
print("MEINE_STUFE" in os.environ)
```

```text
HALLO
WELT

['a.txt', 'b.txt']
test
False
```

Im ersten Teil schickt das Skript den Text `hallo` und `welt` an ein Kind, das die Standardeingabe liest, in Großbuchstaben umwandelt und ausgibt. Weil `print` in unserem Skript zusätzlich einen Zeilenumbruch anhängt, erscheint nach der Ausgabe eine Leerzeile. Im zweiten Teil listet `ls` den Inhalt des temporären Verzeichnisses auf, nicht den des aktuellen Verzeichnisses; wir sortieren die Namen selbst, weil die Sortierung von `ls` von der Spracheinstellung abhängen kann. Der dritte Teil baut mit `{**os.environ, "MEINE_STUFE": "test"}` eine Kopie der Umgebungsvariablen des Skripts, in der eine weitere Variable steht, und reicht sie dem Kind. Die letzte Zeile beweist, dass die Umgebung des Elternprozesses unverändert blieb: Änderungen an `env` wirken nur auf das Kind. Wenn Sie nur `env={"MEINE_STUFE": "test"}` übergäben, hätte das Kind *nur* diese eine Variable und kein `PATH`; das ist selten gewollt.

> **Ergänzung aus der Nachrecherche**
>
> **Parameter und Sicherheitshinweis (abgerufen aus der Python-Dokumentation).** Die Dokumentation von `subprocess.run` (abgerufen wurde die Fassung für Python 3.10, deren englischer Text unverändert in den neueren Versionen steht) beschreibt `capture_output` ausdrücklich als Kurzform für `stdout=PIPE, stderr=PIPE`; zusammen mit `capture_output` dürfen `stdout` und `stderr` nicht angegeben werden. `text` wurde in Python 3.7 ergänzt, `capture_output` ebenfalls. Zur Übergabe der Argumente heißt es dort: „Providing a sequence of arguments is generally preferred, as it allows more careful handling of escape characters and quoting of arguments (e.g. allowing spaces in filenames).“ Zu `shell=True` steht dort, es sei „the application’s responsibility to ensure that all whitespace and metacharacters are quoted appropriately to avoid shell injection vulnerabilities“. Prüfen Sie Details für Ihre Version unter <https://docs.python.org/3/library/subprocess.html>.

Zu den häufigsten Fehlern gehört der Versuch, einen ganzen Befehl als eine einzige Zeichenkette zu übergeben, so wie man ihn in der Shell tippen würde. Das scheitert, denn ohne Shell sucht Python nach einem Programm, dessen Dateiname tatsächlich `echo Hallo` lautet. Der zweite häufige Fehler ist das Vergessen von `text=True`: Dann sind `stdout` und `stderr` Bytes, und ein Vergleich wie `ergebnis.stdout == "Linux"` ist immer falsch, ohne dass Python eine Warnung ausgibt. Drittens ist `stdout` `None`, wenn Sie die Ausgabe nicht abgefangen haben; wer dann `.strip()` aufruft, bekommt einen `AttributeError`.

```python
import subprocess

# FALSCH 1: der ganze Befehl als ein String, ohne Shell
try:
    subprocess.run("echo Hallo", capture_output=True, text=True)
except FileNotFoundError as fehler:
    print("Fehler:", fehler)

# RICHTIG: eine Liste, ein Element pro Argument
r = subprocess.run(["echo", "Hallo"], capture_output=True, text=True)
print(r.stdout.strip())

# FALSCH 2: Bytes mit Text vergleichen
r = subprocess.run(["echo", "ok"], capture_output=True)
print(r.stdout == "ok\n")
# RICHTIG: text=True oder dekodieren
print(r.stdout.decode() == "ok\n")

# FALSCH 3: nichts abgefangen, aber stdout verwendet
r = subprocess.run(["echo", "ok"], stdout=subprocess.DEVNULL)
print(r.stdout)
```

```text
Fehler: [Errno 2] No such file or directory: 'echo Hallo'
Hallo
False
True
None
```

Im ersten Fall meldet Python, dass es keine Datei mit dem Namen `echo Hallo` gibt; die Meldung nennt den Namen in Anführungszeichen, was das Leerzeichen sichtbar macht. Beim zweiten Fall ist der Vergleich der Bytes mit dem Text `False`, obwohl der Inhalt „gleich“ aussieht. Der dritte Fall zeigt mit `subprocess.DEVNULL`, wie man die Ausgabe gezielt verwirft: Das Kind schreibt dann ins Nichts (Linux-Gerät `/dev/null`), und `stdout` im Ergebnis ist `None`.

> **Merke**
>
> Ein Befehl ist für `subprocess` eine Liste: Programmname zuerst, dann jedes Argument als eigenes Element. Mit `capture_output=True, text=True` erhalten Sie die Ausgabe als Text in `stdout` und `stderr`; den Rückgabecode liefert `returncode`. Nehmen Sie `sys.executable`, um denselben Python-Interpreter erneut zu starten.

> **Praxisbeispiel**
>
> Im Administrationsalltag begegnet Ihnen dieses Muster überall: Ein Skript ruft `systemctl is-active nginx` auf und reagiert auf das Ergebnis, es führt `ip -brief address` aus, um Netzwerkadressen zu sammeln, oder startet `tar` für eine Sicherung. Dabei lohnt es sich, bei Programmen, deren Ausgabe Sie weiterverarbeiten, die Spracheinstellung mit `env={**os.environ, "LC_ALL": "C"}` festzulegen. Sonst liefert dasselbe Skript auf einem deutschsprachig eingerichteten Server andere Texte als auf einem englischsprachigen, und Ihre Auswertung bricht dort. In Lerneinheit 5.5 baut sich daraus der Wrapper.

## 5.2 stdout/stderr, Rückgabecodes, ohne shell=True

Die erste Lerneinheit hat gezeigt, wie man ein Programm startet und seine Ausgabe liest. In der Praxis ist die wichtigere Frage: Hat das Programm *funktioniert*? Dafür gibt es drei Informationsquellen, die man unbedingt auseinanderhalten muss. Die erste ist der Rückgabecode, die zweite die Standardfehlerausgabe, die dritte sind die Ausnahmen, die Python selbst auslöst, wenn schon der Start misslingt.

Der Rückgabecode ist die verlässlichste Quelle. Nach der Unix-Konvention bedeutet 0 Erfolg und jede andere Zahl einen Fehler. Das Programm `grep` liefert zum Beispiel 0, wenn es etwas gefunden hat, 1, wenn es nichts gefunden hat, und 2 bei einem echten Fehler; `systemctl is-active` liefert 0 für einen aktiven und 3 für einen nicht aktiven Dienst. Man kann also nie einfach „nicht 0 heißt Fehler“ sagen, sondern muss in der Dokumentation des jeweiligen Programms nachlesen. Wird ein Kind durch ein Signal beendet (etwa mit `kill` oder wegen Speichermangels), ist der Rückgabecode in Python *negativ*: `-15` bedeutet, dass es mit Signal 15 (SIGTERM) beendet wurde, `-9` heißt SIGKILL. Das Beispiel erzeugt diese drei Situationen mit einem kleinen Kindprogramm, das wir selbst steuern.

```python
import subprocess
import sys

def kind(code):
    return subprocess.run([sys.executable, "-c", code],
                          capture_output=True, text=True)

r = kind("import sys; print('fertig'); sys.exit(0)")
print("Fall 1:", r.returncode, repr(r.stdout))

r = kind("import sys; print('Datei fehlt', file=sys.stderr); sys.exit(3)")
print("Fall 2:", r.returncode, repr(r.stdout), repr(r.stderr))

r = kind("import os, signal; os.kill(os.getpid(), signal.SIGTERM)")
print("Fall 3:", r.returncode)
```

```text
Fall 1: 0 'fertig\n'
Fall 2: 3 '' 'Datei fehlt\n'
Fall 3: -15
```

Im ersten Fall endet das Kind normal. Im zweiten Fall schreibt es eine Meldung auf die Standardfehlerausgabe und beendet sich mit Code 3. Beachten Sie, dass `stdout` leer ist und die Meldung ausschließlich in `stderr` steht: Die beiden Ströme sind getrennt, und Fehlermeldungen finden Sie nur dort. Im dritten Fall beendet sich das Kind durch ein Signal; der Rückgabecode ist `-15`.

Nur den Rückgabecode zu prüfen, ist mühsam, wenn jeder Aufruf eine `if`-Abfrage braucht. Deshalb gibt es den Parameter `check=True`. Er sorgt dafür, dass `run` bei einem Rückgabecode ungleich 0 eine Ausnahme vom Typ `subprocess.CalledProcessError` auslöst. Diese Ausnahme trägt alle Informationen mit sich: den Rückgabecode (`returncode`), den Befehl (`cmd`) und, falls abgefangen, die Ausgaben (`stdout` und `stderr`). Das ist eine Anwendung des Prinzips „Fehler dürfen nicht stillschweigend übergangen werden“, das Sie in Modul 6 (Exceptions) noch genauer kennenlernen.

Ein zweiter Schutz ist das Zeitlimit. Ein Programm, das hängt, würde Ihr Skript für immer blockieren, bis jemand eingreift. Mit `timeout=Sekunden` beendet `run` das Kind nach Ablauf der Zeit und löst `subprocess.TimeoutExpired` aus. Gerade in Cronjobs (Modul 9) ist ein Zeitlimit Pflicht, weil sich hängende Skripte sonst Stunde um Stunde stapeln.

Die dritte Fehlerart betrifft den Start selbst. Existiert das Programm nicht oder liegt es nicht im `PATH`, löst Python `FileNotFoundError` aus; darf es nicht ausgeführt werden, entsteht ein `PermissionError`. Das ist etwas anderes als ein Programm, das startet und mit einem Fehlercode endet: Im einen Fall hat *Python* das Programm nie gestartet, im anderen hat das *Programm* einen Fehler gemeldet. Das folgende Beispiel zeigt alle drei Wege und fasst sie in einer Funktion zusammen, die einen kurzen Statustext zurückgibt.

```python
import subprocess
import sys

def pruefen(befehl, limit=2):
    """Startet einen Befehl und beschreibt das Ergebnis in einem Satz."""
    try:
        r = subprocess.run(befehl, capture_output=True, text=True,
                           check=True, timeout=limit)
    except FileNotFoundError:
        return f"Programm nicht gefunden: {befehl[0]}"
    except subprocess.TimeoutExpired:
        return f"Abbruch nach {limit} Sekunden"
    except subprocess.CalledProcessError as fehler:
        meldung = fehler.stderr.strip()
        return f"Fehlercode {fehler.returncode}: {meldung}"
    return f"OK: {r.stdout.strip()}"

py = sys.executable
print(pruefen(["echo", "alles gut"]))
print(pruefen([py, "-c", "import sys; sys.stderr.write('kaputt'); sys.exit(4)"]))
print(pruefen(["gibt-es-nicht", "--x"]))
print(pruefen([py, "-c", "import time; time.sleep(10)"], limit=0.5))
```

```text
OK: alles gut
Fehlercode 4: kaputt
Programm nicht gefunden: gibt-es-nicht
Abbruch nach 0.5 Sekunden
```

Die Funktion `pruefen` führt den Befehl mit `check=True` und einem Zeitlimit aus und fängt die drei möglichen Ausnahmen getrennt ab. Die Reihenfolge der `except`-Zweige spielt hier keine Rolle, weil die drei Ausnahmen nicht voneinander erben. Beim vierten Aufruf schläft das Kind zehn Sekunden, wird aber nach einer halben Sekunde von `run` beendet; die Dokumentation sagt dazu, dass das Kind bei Ablauf der Zeit beendet und auf sein Ende gewartet wird, bevor die Ausnahme weitergereicht wird. Sie verlieren also keine verwaisten Prozesse.

> **Ergänzung aus der Nachrecherche**
>
> **Rückgabecodes von systemctl (abgerufen aus dem Handbuch von systemd 257).** Das Handbuch zu `systemctl` beschreibt im Abschnitt „Exit status“, dass `systemctl` die Rückgabecodes der „Linux Standard Base“ verwendet. Für uns relevant sind: 0 (Einheit ist aktiv bzw. alles in Ordnung), 1 (bei `is-failed`: die Einheit ist nicht fehlgeschlagen), 3 („program is not running“, die Einheit ist nicht aktiv) und 4 („program or service status is unknown“, es gibt keine Einheit dieses Namens). Das Handbuch weist selbst darauf hin, dass diese Zuordnung unvollkommen ist. Wer genau wissen will, in welchem Zustand sich ein Dienst befindet, liest besser die Ausgabe von `systemctl is-active` (sie nennt den Zustand als Wort wie `active`, `inactive` oder `failed`) als nur den Rückgabecode. Für `is-active` gilt laut Handbuch, dass der Rückgabecode 0 ist, wenn mindestens eine der genannten Einheiten aktiv ist, sonst ungleich 0. Das Handbuch von `systemctl` finden Sie unter <https://www.freedesktop.org/software/systemd/man/257/systemctl.html>.

Nun zum Thema `shell=True`. Ohne diese Option (dem Standard) startet `subprocess` das Programm direkt: Jedes Element der Liste kommt unverändert als ein Argument beim Programm an. Mit `shell=True` dagegen startet Python eine Shell (unter Linux `/bin/sh -c`) und übergibt ihr einen *Text*, den die Shell interpretiert. Das erlaubt Pipes (`|`), Umleitungen (`>`), Platzhalter (`*`) und Variablen (`$HOME`) in einem Aufruf, öffnet aber eine ernste Sicherheitslücke, die *Shell-Injection*. Setzt man den Befehl aus Teilen zusammen, die von außen kommen (Dateiname, Hostname, Benutzereingabe), kann ein manipulierter Wert zusätzliche Befehle einschleusen. Das nächste Beispiel zeigt das mit einem harmlosen `echo`; ein Angreifer würde an der Stelle statt `echo` etwas Zerstörerisches ausführen.

```python
import shlex
import subprocess

name = "daten.txt; echo INJIZIERT"     # kommt von außen, z. B. aus einem Formular

print("--- shell=True mit zusammengebautem Text (GEFAHR) ---")
subprocess.run(f"echo Verarbeite {name}", shell=True)

print("--- Liste ohne Shell (sicher) ---")
subprocess.run(["echo", "Verarbeite", name])

print("--- Notlösung, falls eine Shell nötig ist: shlex.quote ---")
subprocess.run(f"echo Verarbeite {shlex.quote(name)}", shell=True)
```

```text
--- shell=True mit zusammengebautem Text (GEFAHR) ---
Verarbeite daten.txt
INJIZIERT
--- Liste ohne Shell (sicher) ---
Verarbeite daten.txt; echo INJIZIERT
--- Notlösung, falls eine Shell nötig ist: shlex.quote ---
Verarbeite daten.txt; echo INJIZIERT
```

Im ersten Fall hat die Shell das Semikolon als Befehlstrenner verstanden und `echo INJIZIERT` als zweiten Befehl ausgeführt. Im zweiten Fall erhält `echo` den Namen als ein einziges Argument, wie gewünscht; das Semikolon ist hier nur ein Zeichen. Im dritten Fall setzt `shlex.quote()` den Text in Anführungszeichen, sodass die Shell ihn als Einheit behandelt. Das ist als Notlösung zulässig, aber fehleranfällig; verwenden Sie, wann immer es geht, die Listenform.

Oft wird `shell=True` nur benutzt, um eine Pipe zu bauen, also die Ausgabe eines Programms an ein zweites weiterzureichen. Das geht auch ohne Shell: Sie fangen die Ausgabe des ersten Programms ab und übergeben sie als `input` an das zweite. Meist ist es sogar besser, die Aufgabe des zweiten Programms in Python selbst zu lösen, denn Sortieren, Filtern und Zählen kann Python ohnehin. Das folgende Beispiel zeigt beide Wege.

```python
import subprocess

# Wie in der Shell:  printf 'b\na\nc\n' | sort
erste = subprocess.run(["printf", "b\\na\\nc\\n"], capture_output=True, text=True)
zweite = subprocess.run(["sort"], input=erste.stdout, capture_output=True, text=True)
print(zweite.stdout.split())

# Besser: die Sortierung in Python erledigen
print(sorted(erste.stdout.split()))
```

```text
['a', 'b', 'c']
['a', 'b', 'c']
```

Das Programm `printf` erhält als Argument die Zeichenfolge `b\na\nc\n`, wobei `printf` die Backslash-Folgen selbst in Zeilenumbrüche umwandelt (in Python muss der Backslash deshalb verdoppelt werden, damit er unverändert ankommt). Die Ausgabe wird abgefangen und an `sort` weitergereicht. Die zweite Variante spart den Prozess und ist besser testbar.

Bleibt noch der Fall, dass ein Programm sehr lange läuft und viel ausgibt, etwa ein Kopiervorgang oder ein Ping. Dann will man nicht warten, bis alles fertig ist, sondern die Ausgabe Zeile für Zeile verarbeiten. Hierfür nimmt man die Klasse `subprocess.Popen`, auf der `run` aufgebaut ist. Sie startet das Kind und kehrt sofort zurück; das Skript kann dann während des Laufs von `p.stdout` lesen. Mit der `with`-Anweisung (Modul 4) wird der Prozess sauber aufgeräumt.

```python
import subprocess
import sys

kindprogramm = (
    "import time\n"
    "for i in range(1, 4):\n"
    "    print(f'Schritt {i}', flush=True)\n"
    "    time.sleep(0.1)\n"
)

with subprocess.Popen([sys.executable, "-c", kindprogramm],
                      stdout=subprocess.PIPE, text=True) as prozess:
    for zeile in prozess.stdout:
        print("empfangen:", zeile.strip())

print("Rückgabecode:", prozess.returncode)
```

```text
empfangen: Schritt 1
empfangen: Schritt 2
empfangen: Schritt 3
Rückgabecode: 0
```

Die Schleife `for zeile in prozess.stdout` liest, sobald das Kind eine Zeile liefert. Wichtig ist das `flush=True` im Kind: Wenn ein Python-Programm in eine Pipe schreibt (statt in ein Terminal), puffert es seine Ausgabe und gibt sie erst bei Programmende oder bei vollem Puffer weiter; ohne `flush=True` (oder den Interpreter-Schalter `-u`) würden alle drei Zeilen erst am Schluss ankommen. Nach dem Ende des `with`-Blocks ist der Prozess beendet, und `prozess.returncode` enthält den Rückgabecode.

> **Merke**
>
> Drei Wege können schiefgehen: Python kann das Programm nicht starten (`FileNotFoundError`, `PermissionError`), das Programm kann zu lange laufen (`TimeoutExpired`) oder es kann mit einem Fehlercode enden (`CalledProcessError` bei `check=True`). Setzen Sie `check=True` und `timeout` zur Regel. Verwenden Sie `shell=True` nur, wenn Sie wirklich Shell-Funktionen brauchen, und niemals mit Text, der von außen kommt.

> **Praxisbeispiel**
>
> Ein Überwachungsskript, das jede Minute `systemctl is-active nginx` aufruft, braucht genau diese Unterscheidung: Ist nginx nicht aktiv, ist das ein *Befund* (Rückgabecode 3), den das Skript meldet. Ist `systemctl` selbst nicht vorhanden (etwa in einem Container), ist das ein *Umgebungsfehler*, der anders behandelt werden muss. Wer beides in einen Topf wirft, schickt nachts Fehlalarme oder übersieht echte Ausfälle.

## 5.3 argparse Grundlagen

Bisher haben Ihre Skripte ihre Eingaben mit `input()` erfragt oder fest im Code stehen gehabt. Echte Linux-Werkzeuge arbeiten anders: Man ruft sie mit Argumenten auf, etwa `ls -l /var/log` oder `tar -czf backup.tgz daten`, und sie erledigen ihre Arbeit ohne Rückfragen. Nur so lassen sich Skripte in Cronjobs, in andere Skripte oder in Pipelines einbauen. Python legt die Argumente, mit denen ein Skript gestartet wurde, in der Liste `sys.argv` ab. Das Element mit Index 0 ist der Name des Skripts, danach folgen die Argumente als Zeichenketten. Man könnte diese Liste selbst durchsuchen; sobald es aber mehrere Optionen, Standardwerte, Fehlermeldungen und einen Hilfetext geben soll, wird das mühsam und fehleranfällig. Deshalb gibt es in der Standardbibliothek das Modul `argparse`, das diese Arbeit übernimmt.

Zuerst ein paar Begriffe. Ein *Argument* ist alles, was hinter dem Programmnamen steht. *Positionsargumente* (englisch *positional arguments*) haben keinen Namen auf der Kommandozeile; ihre Bedeutung ergibt sich aus der Reihenfolge, wie bei `cp quelle ziel`. *Optionen* (englisch *optional arguments* oder *options*) beginnen mit einem Bindestrich: Kurzformen mit einem Strich und einem Buchstaben (`-v`), Langformen mit zwei Strichen und einem Wort (`--verbose`). Eine Option kann einen Wert erwarten (`--warnung 90`) oder nur ein Schalter sein, der vorhanden oder nicht vorhanden ist (ein *Flag*, zum Beispiel `--verbose`). Diese Konventionen stammen aus der GNU-Welt und werden von fast allen Linux-Programmen befolgt; wer sie einhält, macht seine Skripte für Kollegen sofort verständlich.

Der Ablauf mit `argparse` hat immer drei Schritte. Zuerst erzeugt man ein `ArgumentParser`-Objekt und gibt ihm einen Programmnamen und eine Beschreibung. Dann beschreibt man mit `add_argument()` jedes erwartete Argument: Name, Typ, Standardwert, Hilfetext. Schließlich ruft man `parse_args()` auf, das die Kommandozeile prüft und ein `Namespace`-Objekt liefert, dessen Attribute die Werte enthalten. Ohne Parameter liest `parse_args()` die echte Kommandozeile (`sys.argv[1:]`); man kann ihm aber auch eine Liste übergeben, was für Beispiele und Tests unverzichtbar ist. Genau das tun wir hier, damit die Beispiele ohne echte Kommandozeile laufen.

```python
import argparse

parser = argparse.ArgumentParser(
    prog="diskcheck",
    description="Prüft den belegten Platz eines Verzeichnisses.",
)
parser.add_argument("pfad", help="Verzeichnis, das geprüft werden soll")
parser.add_argument("--warnung", type=int, default=80,
                    help="Schwelle in Prozent (Standard: %(default)s)")
parser.add_argument("-v", "--verbose", action="store_true",
                    help="ausführlichere Ausgabe")

args = parser.parse_args(["/var/log", "--warnung", "90", "-v"])
print(args)
print(args.pfad, args.warnung, args.verbose)

args = parser.parse_args(["/tmp"])
print(args)
```

```text
Namespace(pfad='/var/log', warnung=90, verbose=True)
/var/log 90 True
Namespace(pfad='/tmp', warnung=80, verbose=False)
```

Das Programm legt drei Argumente fest. `pfad` hat keinen Bindestrich und ist deshalb ein Positionsargument; es ist Pflicht. `--warnung` ist eine Option mit Wert; durch `type=int` wandelt `argparse` den Text automatisch in eine Zahl um, und `default=80` gilt, wenn die Option fehlt. `-v` und `--verbose` sind zwei Namen für dasselbe Flag. Die Angabe `action="store_true"` bedeutet: Der Wert ist `False`, solange das Flag fehlt, und `True`, sobald es vorkommt. Auf die Werte greift man über den längsten Optionsnamen zu: `args.verbose`. Enthält ein Optionsname einen Bindestrich, wird er im Attribut zum Unterstrich; aus `--dry-run` wird `args.dry_run`.

Das Schöne an `argparse` ist, dass der Hilfetext fast kostenlos entsteht. Jedes Programm, das Sie mit `-h` oder `--help` aufrufen, zeigt ihn an. Er ist keine Zugabe, sondern die Dokumentation Ihres Werkzeugs, und er entsteht aus den `help`-Texten, die Sie bei den einzelnen Argumenten angeben. In den Hilfetexten steht `%(default)s` für den Standardwert, `%(prog)s` für den Programmnamen.

```python
import argparse

parser = argparse.ArgumentParser(
    prog="diskcheck",
    description="Prüft den belegten Platz eines Verzeichnisses.",
)
parser.add_argument("pfad", help="Verzeichnis, das geprüft werden soll")
parser.add_argument("--warnung", type=int, default=80,
                    help="Schwelle in Prozent (Standard: %(default)s)")
parser.add_argument("-v", "--verbose", action="store_true",
                    help="ausführlichere Ausgabe")

parser.print_help()
```

```text
usage: diskcheck [-h] [--warnung WARNUNG] [-v] pfad

Prüft den belegten Platz eines Verzeichnisses.

positional arguments:
  pfad               Verzeichnis, das geprüft werden soll

options:
  -h, --help         show this help message and exit
  --warnung WARNUNG  Schwelle in Prozent (Standard: 80)
  -v, --verbose      ausführlichere Ausgabe
```

Die erste Zeile, die *Usage*-Zeile, fasst die Aufrufsyntax zusammen: Eckige Klammern kennzeichnen optionale Teile, der Großbuchstabe `WARNUNG` ist der Platzhalter für den Wert (sein Name lässt sich mit dem Parameter `metavar` ändern). Darunter stehen die Beschreibung und die beiden Gruppen „positional arguments“ und „options“. Der Eintrag `-h, --help` entsteht von selbst; `argparse` legt diese Option immer an. Die englischen Rahmentexte („positional arguments“, „show this help message and exit“) stammen vom Modul selbst; je nach Sprache und Installation können sie bei Ihnen übersetzt erscheinen, Ihre eigenen Hilfetexte bleiben, wie Sie sie geschrieben haben. In Python bis 3.9 hieß die Überschrift „optional arguments“ statt „options“.

Was passiert, wenn der Aufruf falsch ist? Dann gibt `argparse` die Usage-Zeile und eine Fehlermeldung auf `stderr` aus und beendet das Programm mit dem Rückgabecode *2*. Dasselbe tut es bei einem Aufruf mit `-h`, dann aber mit dem Rückgabecode 0, weil kein Fehler vorliegt. Technisch löst `argparse` dafür die Ausnahme `SystemExit` aus. Im folgenden Beispiel fangen wir sie ab, um den Rückgabecode zu sehen, und leiten `stderr` nach `stdout` um, damit Fehlermeldung und Kommentar in der richtigen Reihenfolge erscheinen.

```python
import argparse
import contextlib
import sys

parser = argparse.ArgumentParser(prog="diskcheck")
parser.add_argument("pfad")
parser.add_argument("--warnung", type=int, default=80)

fehlerhafte_aufrufe = [
    ["/tmp", "--warnung", "abc"],     # falscher Typ
    [],                                # Pflichtargument fehlt
    ["/tmp", "--unbekannt"],           # unbekannte Option
    ["-h"],                            # Hilfe angefordert
]
for aufruf in fehlerhafte_aufrufe:
    print("Aufruf:", aufruf)
    try:
        with contextlib.redirect_stderr(sys.stdout), contextlib.redirect_stdout(sys.stdout):
            parser.parse_args(aufruf)
    except SystemExit as ende:
        print("-> Rückgabecode", ende.code)
```

```text
Aufruf: ['/tmp', '--warnung', 'abc']
usage: diskcheck [-h] [--warnung WARNUNG] pfad
diskcheck: error: argument --warnung: invalid int value: 'abc'
-> Rückgabecode 2
Aufruf: []
usage: diskcheck [-h] [--warnung WARNUNG] pfad
diskcheck: error: the following arguments are required: pfad
-> Rückgabecode 2
Aufruf: ['/tmp', '--unbekannt']
usage: diskcheck [-h] [--warnung WARNUNG] pfad
diskcheck: error: unrecognized arguments: --unbekannt
-> Rückgabecode 2
Aufruf: ['-h']
usage: diskcheck [-h] [--warnung WARNUNG] pfad

positional arguments:
  pfad

options:
  -h, --help         show this help message and exit
  --warnung WARNUNG
-> Rückgabecode 0
```

Die vier Fälle zeigen, was `argparse` Ihnen abnimmt: Es prüft den Typ (`abc` ist keine Ganzzahl), erkennt fehlende Pflichtargumente und unbekannte Optionen, und es setzt für Fehler den Rückgabecode 2. Genau diesen Wert verwenden auch viele andere Linux-Programme für „falsche Benutzung“; Ihr Skript verhält sich also so, wie ein erfahrener Administrator es erwartet. Sie müssen sich um Fehlermeldungen nicht selbst kümmern.

Für jedes Skript mit Argumenten hat sich ein fester Aufbau bewährt. Der Parser wird in einer Funktion gebaut, die eigentliche Arbeit steckt in einer Funktion `main`, und die Anweisung `sys.exit(main())` am Ende gibt deren Rückgabewert als Rückgabecode an das Betriebssystem weiter. Die Funktion `main` nimmt eine Argumentliste entgegen, die standardmäßig `None` ist; dann liest `parse_args` die echte Kommandozeile. Dadurch kann man `main` im Test mit einer eigenen Liste aufrufen.

```python
import argparse
import sys

def baue_parser():
    parser = argparse.ArgumentParser(
        prog="gruss",
        description="Begrüßt jemanden, höflich oder laut.",
    )
    parser.add_argument("name", nargs="?", default="Welt",
                        help="wer begrüßt wird (Standard: %(default)s)")
    parser.add_argument("--stil", choices=["leise", "normal", "laut"],
                        default="normal", help="Lautstärke der Begrüßung")
    parser.add_argument("--anzahl", type=int, default=1, metavar="N",
                        help="wie oft begrüßt wird")
    return parser

def main(argv=None):
    args = baue_parser().parse_args(argv)
    text = f"Hallo, {args.name}!"
    if args.stil == "laut":
        text = text.upper()
    elif args.stil == "leise":
        text = text.lower()
    for _ in range(args.anzahl):
        print(text)
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```text
Hallo, Welt!
```

Das Programm ist ein vollständiges, lauffähiges Skript. Gestartet ohne Argumente, wie hier, benutzt es die Standardwerte und gibt eine Begrüßung aus; mit `python3 gruss.py Anna --stil laut --anzahl 2` würde es zweimal „HALLO, ANNA!“ ausgeben. Die Klammern um `nargs="?"` bedeuten, dass das Positionsargument `name` optional ist (null oder ein Wert) und dann den Standardwert verwendet; ohne `nargs` wäre ein Positionsargument immer Pflicht. Der Parameter `choices` beschränkt die erlaubten Werte; alles andere führt zu einer Fehlermeldung, die die gültigen Werte nennt. Und `sys.exit(main())` sorgt dafür, dass der Rückgabewert von `main` (hier 0) zum Rückgabecode des Programms wird; geben Sie in `main` bei einem Fehler eine andere Zahl zurück, kann ein aufrufendes Skript oder Cron darauf reagieren. Die Prüfung `if __name__ == "__main__":` stellt sicher, dass der Code nur beim direkten Start läuft, nicht beim Import als Modul (mehr dazu in Modul 7).

Weil `main` eine Argumentliste annimmt, lässt sich das Programm testen, ohne eine Kommandozeile zu bauen. Man ruft die Funktion mit einer Liste auf und prüft Ausgabe und Rückgabewert.

```python
import argparse

def baue_parser():
    parser = argparse.ArgumentParser(prog="gruss")
    parser.add_argument("name", nargs="?", default="Welt")
    parser.add_argument("--stil", choices=["leise", "normal", "laut"], default="normal")
    parser.add_argument("--anzahl", type=int, default=1)
    return parser

def main(argv=None):
    args = baue_parser().parse_args(argv)
    text = f"Hallo, {args.name}!"
    if args.stil == "laut":
        text = text.upper()
    for _ in range(args.anzahl):
        print(text)
    return 0

print("Rückgabe:", main([]))
print("Rückgabe:", main(["Anna", "--stil", "laut", "--anzahl", "2"]))
print("Rückgabe:", main(["--anzahl=1", "Bernd"]))
```

```text
Hallo, Welt!
Rückgabe: 0
HALLO, ANNA!
HALLO, ANNA!
Rückgabe: 0
Hallo, Bernd!
Rückgabe: 0
```

Das Beispiel zeigt auch, dass die Reihenfolge von Optionen und Positionsargumenten frei ist und dass Optionswerte mit einem Gleichheitszeichen übergeben werden dürfen (`--anzahl=1`). Lässt man die Liste weg, liest `parse_args` dagegen die echte Kommandozeile; das ist der Normalfall im Betrieb.

Zu den typischen Stolperfallen gehört `type=bool`. Man würde erwarten, dass `--aktiv False` den Wert `False` liefert. Tatsächlich ruft `argparse` einfach `bool("False")` auf, und jede nichtleere Zeichenkette ist wahr; das Ergebnis ist also `True`. Für Schalter gehört `action="store_true"` (oder `store_false`) hin. Ein zweiter Fehler ist ein Positionsargument mit `default`: Es bleibt trotzdem Pflicht, solange Sie nicht `nargs="?"` setzen. Drittens: `args.dry-run` ist in Python eine Subtraktion und führt zu einem `AttributeError`, richtig ist `args.dry_run`.

```python
import argparse
import contextlib
import sys

# FALSCH: type=bool
p = argparse.ArgumentParser()
p.add_argument("--aktiv", type=bool, default=False)
print(p.parse_args(["--aktiv", "False"]).aktiv)      # trotz "False" wahr!

# RICHTIG: ein Schalter
p = argparse.ArgumentParser()
p.add_argument("--aktiv", action="store_true")
print(p.parse_args([]).aktiv, p.parse_args(["--aktiv"]).aktiv)

# FALSCH: default bei Positionsargument ohne nargs
p = argparse.ArgumentParser(prog="x")
p.add_argument("pfad", default="/tmp")
try:
    with contextlib.redirect_stderr(sys.stdout):
        p.parse_args([])
except SystemExit:
    pass

# RICHTIG: nargs="?"
p = argparse.ArgumentParser(prog="x")
p.add_argument("pfad", nargs="?", default="/tmp")
print(p.parse_args([]).pfad)

# Bindestrich wird zum Unterstrich
p = argparse.ArgumentParser()
p.add_argument("--dry-run", action="store_true")
print(p.parse_args(["--dry-run"]).dry_run)
```

```text
True
False True
usage: x [-h] pfad
x: error: the following arguments are required: pfad
/tmp
True
```

> **Ergänzung aus der Nachrecherche**
>
> **Hinweise aus der aktuellen argparse-Dokumentation (abgerufen; Quelltext der Dokumentationsseite).** Die Dokumentation rät ausdrücklich von `type=bool` ab und vermerkt, dass `required`-Optionen (also Pflicht-Optionen mit Bindestrich) „generally considered bad form“ sind: Was zwingend nötig ist, sollte ein Positionsargument sein. Der Rückgabecode bei fehlerhafter Eingabe ist laut Dokumentation 2. Neu in Python 3.14 sind laut Dokumentation zwei Schlüsselwortargumente des `ArgumentParser`: `suggest_on_error` (Standard `False`) schlägt bei falsch geschriebenen Werten oder Unterbefehlen eine Korrektur vor, im Beispiel der Dokumentation `invalid choice: 'sumn', maybe you meant 'sum'?`, und `color` (Standard `True`) färbt Hilfe- und Fehlertexte ein, wobei `color=False` die Einfärbung auch dann abschaltet, wenn Umgebungsvariablen wie `FORCE_COLOR` gesetzt sind. Ob Ihr System schon Python 3.14 hat, prüfen Sie mit `python3 --version`; die Beispiele dieses Skripts laufen unverändert auch ohne diese Neuerungen. Die Ausführung der Beispiele erfolgte mit Python 3.13, die 3.14-Funktionen wurden deshalb nicht selbst ausgeführt.

> **Merke**
>
> Mit `argparse` entsteht ein Kommandozeilenprogramm in drei Schritten: Parser anlegen, Argumente beschreiben, `parse_args()` aufrufen. Positionsargumente sind Pflicht (außer mit `nargs="?"`), Optionen beginnen mit `-` oder `--`, Schalter sind `store_true`. Fehler führen automatisch zu einer Meldung und dem Rückgabecode 2; `main(argv=None)` mit `sys.exit(main())` ist das Standardgerüst.

> **Praxisbeispiel**
>
> Jedes Werkzeug, das Sie in einem Cronjob oder einer Pipeline einsetzen, wird irgendwann mit anderen Werten gestartet: ein anderes Verzeichnis, eine andere Schwelle, ein Probelauf statt einer echten Ausführung. Wer diese Werte als Argumente herausgibt, muss den Code nie ändern. In Modul 4 war der Probelauf noch ein Parameter `wirklich=False` einer Funktion; mit `argparse` wird daraus die Option `--apply` oder `--dry-run`, die Kollegen aus der `--help`-Ausgabe verstehen.

## 5.4 argparse Optionen & Subcommands

Die Grundlagen reichen für kleine Werkzeuge. Größere Programme brauchen mehr: Optionen, die mehrere Werte annehmen, Schalter, die sich zählen lassen (`-vvv`), Optionen, die sich gegenseitig ausschließen, Werte in besonderen Formaten wie „10MiB“ und schließlich mehrere Unterbefehle in einem Programm, so wie `git commit`, `git push` und `git log` drei verschiedene Aktionen eines einzigen Programms sind. Auch dafür bietet `argparse` fertige Bausteine.

Beginnen wir mit der Anzahl der Werte. Der Parameter `nargs` legt fest, wie viele Werte ein Argument verbraucht: eine feste Zahl `N`, `"?"` für null oder einen Wert, `"*"` für null oder mehr und `"+"` für mindestens einen. Mit `nargs="+"` können Sie also eine Liste von Dateien annehmen, wie `rm datei1 datei2 datei3`. Soll eine Option mehrfach auftreten dürfen (`--ausschluss a --ausschluss b`), nimmt man `action="append"`; die Werte sammeln sich dann in einer Liste. Mit `action="count"` zählt `argparse` die Häufigkeit eines Flags, was man für die Gesprächigkeit benutzt: `-v` ist „ausführlich“, `-vv` „sehr ausführlich“. Und `action="version"` gibt auf `--version` eine Versionsnummer aus und beendet das Programm.

Oft reichen die eingebauten Typen `int` und `float` nicht aus. Als `type` darf jede Funktion dienen, die einen Text entgegennimmt und den fertigen Wert zurückgibt. Löst sie dabei `argparse.ArgumentTypeError` aus, wird die Fehlermeldung ordentlich formatiert angezeigt und der Rückgabecode 2 gesetzt. Das folgende Beispiel baut daraus einen Typ für Größenangaben, wie ihn Werkzeuge wie `find -size` kennen, und kombiniert mehrere der genannten Möglichkeiten. Ein *Mutually-Exclusive-Group* (sich gegenseitig ausschließende Gruppe) verhindert, dass der Benutzer zwei Optionen gleichzeitig wählt, die sich widersprechen.

```python
import argparse
import contextlib
import sys

def groesse(text):
    """Wandelt Angaben wie '512', '10KiB' oder '1.5MiB' in Byte um."""
    faktoren = {"KiB": 1024, "MiB": 1024 ** 2, "GiB": 1024 ** 3}
    zahl, faktor = text, 1
    for endung, wert in faktoren.items():
        if text.endswith(endung):
            zahl, faktor = text[:-len(endung)], wert
            break
    try:
        return int(float(zahl) * faktor)
    except ValueError:
        raise argparse.ArgumentTypeError(f"ungültige Größe: {text!r}")

parser = argparse.ArgumentParser(prog="finder")
parser.add_argument("dateien", nargs="+", help="eine oder mehrere Dateien")
parser.add_argument("--min-groesse", type=groesse, default=0)
parser.add_argument("--ausschluss", action="append", default=[], metavar="MUSTER")
parser.add_argument("-v", "--verbose", action="count", default=0)
parser.add_argument("--version", action="version", version="%(prog)s 1.0")
format_gruppe = parser.add_mutually_exclusive_group()
format_gruppe.add_argument("--json", action="store_true")
format_gruppe.add_argument("--csv", action="store_true")

print(parser.parse_args(["a.txt", "b.txt", "--min-groesse", "1.5MiB", "-vv"]))
print(parser.parse_args(["x", "--ausschluss", "*.tmp", "--ausschluss", "*.bak", "--json"]))

for aufruf in (["x", "--min-groesse", "viel"], ["x", "--json", "--csv"], ["--version"]):
    try:
        with contextlib.redirect_stderr(sys.stdout):
            parser.parse_args(aufruf)
    except SystemExit as ende:
        print("-> Rückgabecode", ende.code)
```

```text
Namespace(dateien=['a.txt', 'b.txt'], min_groesse=1572864, ausschluss=[], verbose=2, json=False, csv=False)
Namespace(dateien=['x'], min_groesse=0, ausschluss=['*.tmp', '*.bak'], verbose=0, json=True, csv=False)
usage: finder [-h] [--min-groesse MIN_GROESSE] [--ausschluss MUSTER] [-v]
              [--version] [--json | --csv]
              dateien [dateien ...]
finder: error: argument --min-groesse: ungültige Größe: 'viel'
-> Rückgabecode 2
usage: finder [-h] [--min-groesse MIN_GROESSE] [--ausschluss MUSTER] [-v]
              [--version] [--json | --csv]
              dateien [dateien ...]
finder: error: argument --csv: not allowed with argument --json
-> Rückgabecode 2
finder 1.0
-> Rückgabecode 0
```

Die Funktion `groesse` sucht die Einheit am Ende des Textes, rechnet in Byte um und meldet bei einem Zahlenfehler einen `ArgumentTypeError`. `1.5MiB` wird so zu 1 572 864 Byte. Die Option `--ausschluss` sammelt mit `append` alle Vorkommen in einer Liste; wichtig ist `default=[]`, sonst wäre der Standardwert `None`. Beachten Sie, dass die Gruppe `[--json ` –csv\]\| in der Usage-Zeile als Alternative erscheint und dass der Versuch, beides anzugeben, abgelehnt wird. Bei `--version` wird der Text ausgegeben und das Programm mit Code 0 beendet. In allen Fällen hat `argparse` die Prüfung und die Fehlermeldung geliefert, und Sie haben keine Zeile dafür geschrieben.

Sobald ein Programm mehrere Aufgaben hat, ordnet man sie in *Unterbefehle* (Subcommands). Jeder Unterbefehl hat eigene Argumente und eigenen Hilfetext, das Hauptprogramm legt nur fest, welcher gewählt wurde. Man erzeugt sie mit `add_subparsers()` am Hauptparser und mit `add_parser()` für jeden Unterbefehl. Die Angabe `dest="befehl"` speichert den gewählten Namen im Ergebnis, und `required=True` (seit Python 3.7 möglich) erzwingt, dass überhaupt ein Unterbefehl angegeben wird. Ein bewährtes Muster ist, jedem Unterbefehl mit `set_defaults(func=...)` die zuständige Funktion mitzugeben; `main` muss dann nur noch `args.func(args)` aufrufen, statt eine lange `if`-Kette zu pflegen.

Gemeinsame Argumente, die mehrere Unterbefehle brauchen, legt man in einem eigenen Parser ohne Hilfeoption (`add_help=False`) an und reicht ihn mit `parents=[...]` an die Unterbefehle weiter. Das folgende Programm `lager` hat die Unterbefehle `suche` und `bericht`. Es arbeitet in einem temporären Verzeichnis, das das Beispiel selbst anlegt.

```python
import argparse
import tempfile
from collections import Counter
from pathlib import Path

def cmd_suche(args):
    treffer = [p for p in sorted(Path(args.pfad).rglob("*"))
               if p.is_file() and p.stat().st_size >= args.min_groesse]
    for p in treffer:
        print(f"{p.stat().st_size:>6}  {p.relative_to(args.pfad)}")
    if args.verbose:
        print(f"{len(treffer)} Treffer")
    return 0 if treffer else 1

def cmd_bericht(args):
    zaehler = Counter(p.suffix or "(keine)"
                      for p in Path(args.pfad).rglob("*") if p.is_file())
    for endung, anzahl in sorted(zaehler.items(), key=lambda x: (-x[1], x[0])):
        print(f"{endung:<8}{anzahl}")
    return 0

def baue_parser():
    gemeinsam = argparse.ArgumentParser(add_help=False)
    gemeinsam.add_argument("pfad", help="Basisverzeichnis")

    parser = argparse.ArgumentParser(prog="lager")
    parser.add_argument("-v", "--verbose", action="store_true")
    unter = parser.add_subparsers(dest="befehl", required=True, metavar="BEFEHL")

    suche = unter.add_parser("suche", parents=[gemeinsam],
                             help="Dateien ab einer Mindestgröße auflisten")
    suche.add_argument("--min-groesse", type=int, default=0, metavar="BYTE")
    suche.set_defaults(func=cmd_suche)

    bericht = unter.add_parser("bericht", parents=[gemeinsam],
                               help="Dateien nach Endung zählen")
    bericht.set_defaults(func=cmd_bericht)
    return parser

def main(argv=None):
    args = baue_parser().parse_args(argv)
    return args.func(args)

with tempfile.TemporaryDirectory() as tmp:
    (Path(tmp) / "a.log").write_text("x" * 50)
    (Path(tmp) / "b.log").write_text("x" * 500)
    (Path(tmp) / "c.txt").write_text("x" * 5)
    (Path(tmp) / "README").write_text("x")

    print("== suche ==")
    print("Code:", main(["-v", "suche", tmp, "--min-groesse", "40"]))
    print("== bericht ==")
    print("Code:", main(["bericht", tmp]))
    print("== nichts gefunden ==")
    print("Code:", main(["suche", tmp, "--min-groesse", "99999"]))
```

```text
== suche ==
    50  a.log
   500  b.log
2 Treffer
Code: 0
== bericht ==
.log    2
(keine) 1
.txt    1
Code: 0
== nichts gefunden ==
Code: 1
```

Das Hauptprogramm kennt den globalen Schalter `-v`; die Unterbefehle `suche` und `bericht` teilen sich das Positionsargument `pfad`, das nur einmal im Elternparser `gemeinsam` definiert ist. Weil `set_defaults(func=...)` jedem Unterbefehl seine Funktion zuordnet, besteht `main` aus zwei Zeilen. Die Funktionen geben einen Rückgabecode zurück, den Sie später mit `sys.exit(main())` an das Betriebssystem weitergeben: 0 für Erfolg, 1 für „nichts gefunden“, was Skripte wie bei `grep` unterscheiden können. Ein Hilfetext entsteht für jeden Unterbefehl getrennt; mit `lager suche -h` sieht man nur die Optionen dieses Unterbefehls.

Typische Fehler betreffen hier die Position der Argumente. Optionen des *Hauptparsers* müssen vor dem Unterbefehl stehen, Optionen eines *Unterbefehls* dahinter. Wer `lager suche /tmp -v` schreibt, bekommt die Meldung, dass `-v` unbekannt ist, weil der Unterparser diese Option nicht kennt. Außerdem: Ohne `required=True` ist ein fehlender Unterbefehl kein Fehler, `args.befehl` ist dann `None`, und der Zugriff auf `args.func` scheitert mit einem `AttributeError`, den niemand versteht.

```python
import argparse
import contextlib
import sys

def baue(required):
    parser = argparse.ArgumentParser(prog="lager")
    parser.add_argument("-v", "--verbose", action="store_true")
    unter = parser.add_subparsers(dest="befehl", required=required)
    suche = unter.add_parser("suche")
    suche.add_argument("pfad")
    return parser

# FALSCH: globale Option hinter dem Unterbefehl
try:
    with contextlib.redirect_stderr(sys.stdout):
        baue(True).parse_args(["suche", "/tmp", "-v"])
except SystemExit as ende:
    print("-> Rückgabecode", ende.code)

# RICHTIG: globale Option vor dem Unterbefehl
print(baue(True).parse_args(["-v", "suche", "/tmp"]))

# FALSCH: required fehlt, der Fehler zeigt sich erst später
args = baue(False).parse_args([])
print(args)

# RICHTIG: required=True meldet den Fehler sofort und verständlich
try:
    with contextlib.redirect_stderr(sys.stdout):
        baue(True).parse_args([])
except SystemExit as ende:
    print("-> Rückgabecode", ende.code)
```

```text
usage: lager [-h] [-v] {suche} ...
lager: error: unrecognized arguments: -v
-> Rückgabecode 2
Namespace(verbose=True, befehl='suche', pfad='/tmp')
Namespace(verbose=False, befehl=None)
usage: lager [-h] [-v] {suche} ...
lager: error: the following arguments are required: befehl
-> Rückgabecode 2
```

> **Ergänzung aus der Nachrecherche**
>
> **Unterbefehle und Gruppen laut Dokumentation (abgerufen).** Die argparse-Dokumentation beschreibt `add_subparsers()` mit den Parametern `dest` und `required` (`required` ab Python 3.7) und zeigt das hier verwendete Muster, jedem Unterparser mit `set_defaults(func=...)` eine Funktion mitzugeben. Zu den sich ausschließenden Gruppen heißt es, dass höchstens eine Option der Gruppe vorkommen darf; mit `required=True` in `add_mutually_exclusive_group()` muss genau eine vorkommen. Der Parameter `exit_on_error=False` (seit Python 3.9) verhindert das Beenden des Programms bei Fehlern; stattdessen löst `argparse` dann eine Ausnahme aus, die der Aufrufer abfangen kann. Diese Funktion eignet sich für Programme, die einen Fehler selbst behandeln wollen; für normale Kommandozeilenwerkzeuge ist das Beenden mit Code 2 genau das richtige Verhalten.

Zum Schluss ein Wort zu Standardwerten aus der Umgebung. Viele Werkzeuge lassen sich zusätzlich über Umgebungsvariablen steuern, zum Beispiel `EDITOR` oder `HTTP_PROXY`. Sie lesen die Variable mit `os.environ.get("NAME", "ersatz")` und geben das Ergebnis als `default` an `add_argument`. Dann gilt die Rangfolge: Option auf der Kommandozeile, danach Umgebungsvariable, danach fest eingebauter Wert. Dieselbe Rangfolge werden Sie in Modul 7 für Konfigurationsdateien wiedersehen.

> **Merke**
>
> `nargs` bestimmt, wie viele Werte ein Argument nimmt, `action="append"` sammelt wiederholte Optionen, `action="count"` zählt Schalter wie `-vv`. Eigene Typen sind Funktionen, die bei Fehlern `ArgumentTypeError` auslösen. Unterbefehle entstehen mit `add_subparsers(dest=..., required=True)`; Optionen des Hauptprogramms stehen vor dem Unterbefehl, Optionen des Unterbefehls dahinter.

> **Praxisbeispiel**
>
> Fast jedes professionelle Werkzeug folgt dem Muster „Programm, Unterbefehl, Optionen“: `git commit -m`, `systemctl restart nginx`, `docker run`, `pip install`. Wenn Sie Ihre eigenen Admin-Skripte nach diesem Muster bauen (`lager suche`, `lager bericht`), müssen Sie nicht für jede Aufgabe ein eigenes Skript pflegen, und Kollegen finden sich sofort zurecht. Aus dem Gerüst in diesem Abschnitt entsteht in Lerneinheit 5.5 der Wrapper für `systemctl` und `df`.

## 5.5 Wrapper für systemctl/df + Quiz

In dieser Lerneinheit wird zusammengeführt, was die vier vorigen Einheiten gebracht haben. Das Ziel ist ein kleines, aber vollständiges Kommandozeilenwerkzeug namens `sysw`, das zwei Dinge prüft: ob ein Dienst läuft (mit `systemctl`) und wie voll eine Festplatte ist (mit `df`). Man nennt ein solches Programm einen *Wrapper*, weil es ein vorhandenes Werkzeug umhüllt, seine Ausgabe versteht und ein einheitliches Ergebnis daraus macht. Der Gewinn ist dreifach: Aus Rohtext wird ein klares Urteil, Fehler beim Aufruf werden sauber abgefangen, und das Ergebnis lässt sich über den Rückgabecode in andere Programme einspeisen, etwa in ein Monitoringsystem oder ein Cronskript.

Für die Rückgabecodes verwendet das Werkzeug eine Konvention aus der Überwachungswelt, die viele Monitoringsysteme (zum Beispiel Nagios und dazu kompatible Programme) benutzen: 0 bedeutet *OK*, 1 *Warnung*, 2 *kritisch* und 3 *unbekannt*. Der letzte Wert ist wichtig. „Unbekannt“ heißt: Das Werkzeug konnte gar keine Aussage treffen, weil `systemctl` fehlt oder `df` abgebrochen ist. Das ist etwas anderes als „kritisch“, denn dort wurde ein Problem festgestellt. Diese Unterscheidung wurde in Lerneinheit 5.2 vorbereitet, als wir den Start-Fehler vom Programm-Fehler getrennt haben. Die Konvention stammt aus dem Monitoring-Bereich und wurde hier nicht in einer Quelle nachgeschlagen; sie ist als Entwurfsentscheidung zu verstehen, nicht als Vorschrift von Python oder Linux.

Der Entwurf folgt einigen Grundsätzen, die für alle Wrapper gelten. Erstens trennt er die *Ausführung* (ein Aufruf von `subprocess`) von der *Auswertung* (reine Python-Funktionen, die Text in ein Urteil verwandeln). Reine Funktionen lassen sich mit Beispieltexten testen, ohne dass das echte Programm laufen muss. Zweitens setzt der Aufruf `LC_ALL=C`, damit die Ausgabe nicht je nach Spracheinstellung übersetzt wird, und ein Zeitlimit, damit das Skript nie hängen bleibt. Drittens wird kein `shell=True` verwendet, die Befehle sind Listen. Viertens gibt jede Funktion einen Rückgabecode zurück, und nur ganz am Ende macht `sys.exit(main())` daraus den Rückgabecode des Programms.

Bei `df` nutzen wir die Option `-P` (POSIX-Format) zusammen mit `-k` (Größen in 1024-Byte-Blöcken). Das `-P` sorgt dafür, dass jede Dateisystemzeile auf genau einer Zeile ausgegeben wird; ohne diese Option bricht `df` lange Gerätenamen um, und die Auswertung nach Spalten wäre falsch. Die Spalten sind dann: Dateisystem, Blöcke, belegt, verfügbar, Kapazität in Prozent und Einhängepunkt. Weil ein Einhängepunkt Leerzeichen enthalten kann, teilen wir die Zeile mit `split(None, 5)` in höchstens sechs Teile; der letzte Teil enthält dann den Rest der Zeile unverändert.

Hier ist das Werkzeug in vollem Umfang. Speichern Sie es unter dem Namen `sysw.py`. Gestartet ohne Argumente beendet es sich mit Rückgabecode 2 und einer Meldung, weil ein Unterbefehl Pflicht ist; die Ausgabe zeigt das.

```python
import argparse
import os
import subprocess
import sys

OK, WARNUNG, KRITISCH, UNBEKANNT = 0, 1, 2, 3
STATUSWORT = {OK: "OK", WARNUNG: "WARNUNG", KRITISCH: "KRITISCH",
              UNBEKANNT: "UNBEKANNT"}


def ausfuehren(befehl, limit=10):
    """Führt einen Befehl aus und liefert (Rückgabecode, stdout, stderr).

    Konnte der Befehl nicht gestartet werden, ist der Rückgabecode None
    und stderr enthält die Erklärung."""
    umgebung = {**os.environ, "LC_ALL": "C"}
    try:
        r = subprocess.run(befehl, capture_output=True, text=True,
                           timeout=limit, env=umgebung)
    except FileNotFoundError:
        return None, "", f"Programm nicht gefunden: {befehl[0]}"
    except subprocess.TimeoutExpired:
        return None, "", f"Zeitüberschreitung nach {limit} s: {befehl[0]}"
    return r.returncode, r.stdout.strip(), r.stderr.strip()


def bewerte_dienst(zustand):
    """Ordnet das Zustandswort von 'systemctl is-active' einem Status zu."""
    if zustand == "active":
        return OK
    if zustand in ("activating", "reloading"):
        return WARNUNG
    if zustand in ("inactive", "failed", "deactivating"):
        return KRITISCH
    return UNBEKANNT


def parse_df(text):
    """Liest die Ausgabe von 'df -P -k' und liefert (Mountpunkt, Prozent, frei in KiB)."""
    zeilen = text.strip().splitlines()
    if len(zeilen) < 2:
        raise ValueError("unerwartete df-Ausgabe")
    felder = zeilen[1].split(None, 5)
    if len(felder) != 6 or not felder[4].endswith("%"):
        raise ValueError(f"unerwartete df-Zeile: {zeilen[1]!r}")
    return felder[5], int(felder[4].rstrip("%")), int(felder[3])


def bewerte_platte(prozent, warn, krit):
    if prozent >= krit:
        return KRITISCH
    if prozent >= warn:
        return WARNUNG
    return OK


def cmd_dienst(args):
    code, ausgabe, fehler = ausfuehren([args.systemctl, "is-active", args.name])
    if code is None:
        print(f"DIENST UNBEKANNT - {fehler}")
        return UNBEKANNT
    status = bewerte_dienst(ausgabe)
    print(f"DIENST {STATUSWORT[status]} - {args.name} ist {ausgabe}")
    return status


def cmd_platte(args):
    code, ausgabe, fehler = ausfuehren(["df", "-P", "-k", args.pfad])
    if code != 0:
        print(f"PLATTE UNBEKANNT - df fehlgeschlagen: {fehler}")
        return UNBEKANNT
    try:
        mount, prozent, frei = parse_df(ausgabe)
    except ValueError as problem:
        print(f"PLATTE UNBEKANNT - {problem}")
        return UNBEKANNT
    status = bewerte_platte(prozent, args.warn, args.krit)
    print(f"PLATTE {STATUSWORT[status]} - {mount} zu {prozent} % belegt, "
          f"{frei / 1024 ** 2:.1f} GiB frei")
    return status


def baue_parser():
    parser = argparse.ArgumentParser(
        prog="sysw",
        description="Prüft Dienste und Plattenplatz. Rückgabecodes: "
                    "0 OK, 1 WARNUNG, 2 KRITISCH, 3 UNBEKANNT.")
    unter = parser.add_subparsers(dest="befehl", required=True, metavar="BEFEHL")

    dienst = unter.add_parser("dienst", help="Zustand eines systemd-Dienstes prüfen")
    dienst.add_argument("name", help="Name der Einheit, z. B. nginx")
    dienst.add_argument("--systemctl", default="systemctl", help=argparse.SUPPRESS)
    dienst.set_defaults(func=cmd_dienst)

    platte = unter.add_parser("platte", help="Belegung eines Dateisystems prüfen")
    platte.add_argument("pfad", nargs="?", default="/", help="beliebiger Pfad (Standard: /)")
    platte.add_argument("--warn", type=int, default=80, metavar="PROZENT")
    platte.add_argument("--krit", type=int, default=90, metavar="PROZENT")
    platte.set_defaults(func=cmd_platte)
    return parser


def main(argv=None):
    parser = baue_parser()
    args = parser.parse_args(argv)
    if args.befehl == "platte" and args.warn >= args.krit:
        parser.error("--warn muss kleiner sein als --krit")
    return args.func(args)


if __name__ == "__main__":
    sys.exit(main())
```

```text
usage: sysw [-h] BEFEHL ...
sysw: error: the following arguments are required: BEFEHL
```

Das Programm ist lang, aber jede Funktion hat nur eine Aufgabe. `ausfuehren` ist die einzige Stelle, die `subprocess` kennt; sie meldet einen Startfehler mit dem Rückgabecode `None` und liefert sonst Rückgabecode, Ausgabe und Fehlertext ohne Zeilenumbruch am Ende. `bewerte_dienst` und `parse_df` sind reine Funktionen, die nur mit Zeichenketten arbeiten. Die Unterbefehle `cmd_dienst` und `cmd_platte` verbinden die Teile miteinander und geben den Status zurück. Die Option `--systemctl` ist mit `help=argparse.SUPPRESS` aus der Hilfe ausgeblendet; sie existiert nur, damit wir im Test eine Attrappe statt des echten `systemctl` angeben können. Und `parser.error()` in `main` beendet das Programm mit derselben Meldung und dem Rückgabecode 2 wie jede andere falsche Benutzung, wenn die Schwellen unsinnig gewählt sind.

Wie testet man ein solches Programm, ohne auf einem Rechner mit systemd zu sein und ohne einen Dienst zu stoppen? Man ersetzt `systemctl` durch eine *Attrappe*: ein kleines Skript, das dieselbe Schnittstelle hat (gleiche Argumente, gleiche Ausgabe, gleiche Rückgabecodes), aber nichts Echtes tut. Das ist eine Vorschau auf die Technik des Mockings, die Modul 14 behandelt. Das folgende Testprogramm legt in einem temporären Verzeichnis ein Shell-Skript an, das sich wie `systemctl is-active` verhält, macht es ausführbar (`chmod(0o755)`) und übergibt es über `--systemctl`. Es muss im selben Verzeichnis wie `sysw.py` liegen, weil es das Programm mit `import sysw` einbindet; dank der Prüfung `if __name__ == "__main__"` startet der Import das Programm nicht.

```python
import contextlib
import io
import tempfile
from pathlib import Path

import sysw


def lauf(argv):
    """Ruft sysw.main auf und liefert (Rückgabecode, Ausgabe)."""
    puffer = io.StringIO()
    with contextlib.redirect_stdout(puffer):
        code = sysw.main(argv)
    return code, puffer.getvalue().strip()


ATTRAPPE = """#!/bin/sh
# Verhält sich wie: systemctl is-active NAME
case "$2" in
  nginx)  echo active;     exit 0 ;;
  update) echo activating; exit 3 ;;
  backup) echo failed;     exit 3 ;;
  *)      echo inactive;   exit 3 ;;
esac
"""

with tempfile.TemporaryDirectory() as tmp:
    falsch = Path(tmp) / "systemctl"
    falsch.write_text(ATTRAPPE)
    falsch.chmod(0o755)

    print("--- Dienste ---")
    for name in ("nginx", "update", "backup", "ssh"):
        print(lauf(["dienst", name, "--systemctl", str(falsch)]))
    print(lauf(["dienst", "nginx", "--systemctl", "gibt-es-nicht"]))

    print("--- Platte (echtes df) ---")
    code, text = lauf(["platte", tmp, "--warn", "101", "--krit", "102"])
    print(code, text.split()[:2])
    code, text = lauf(["platte", tmp, "--warn", "0", "--krit", "101"])
    print(code, text.split()[:2])
    code, text = lauf(["platte", str(Path(tmp) / "fehlt")])
    print(code, text.split(":")[0])

print("--- Auswertung mit Beispieltext ---")
beispiel = """Filesystem     1024-blocks     Used Available Capacity Mounted on
/dev/sda1         51474044 38000000  10838044      78% /mnt/daten mit leerzeichen
"""
print(sysw.parse_df(beispiel))
print([sysw.bewerte_platte(p, 80, 90) for p in (78, 80, 89, 90, 99)])
```

```text
--- Dienste ---
(0, 'DIENST OK - nginx ist active')
(1, 'DIENST WARNUNG - update ist activating')
(2, 'DIENST KRITISCH - backup ist failed')
(2, 'DIENST KRITISCH - ssh ist inactive')
(3, 'DIENST UNBEKANNT - Programm nicht gefunden: gibt-es-nicht')
--- Platte (echtes df) ---
0 ['PLATTE', 'OK']
1 ['PLATTE', 'WARNUNG']
3 PLATTE UNBEKANNT - df fehlgeschlagen
--- Auswertung mit Beispieltext ---
('/mnt/daten mit leerzeichen', 78, 10838044)
[0, 1, 1, 2, 2]
```

Der erste Block zeigt die vier Zustände des Dienstes. Die Attrappe antwortet je nach Dienstname anders und gibt immer die Rückgabecodes zurück, die auch das echte `systemctl is-active` nutzt (0 für aktiv, 3 sonst). Unser Wrapper richtet sich aber nach dem ausgegebenen *Zustandswort*, nicht nach dem Rückgabecode, denn aus dem Code 3 allein lässt sich „inaktiv“ nicht von „fehlgeschlagen“ unterscheiden. Der fünfte Aufruf zeigt den Unterschied zwischen einem Befund und einem Umgebungsfehler: Wenn das Programm nicht existiert, lautet das Urteil „unbekannt“ (Code 3). Der zweite Block ruft das echte `df` auf, gibt aber der Ausgabe wegen der rechnerabhängigen Zahlen nur die ersten zwei Wörter aus. Mit absichtlich extremen Schwellen (101 und 102 Prozent beziehungsweise 0 und 101) erzwingen wir einen vorhersagbaren Status. Der letzte Aufruf übergibt einen nicht vorhandenen Pfad: `df` meldet einen Fehler, und der Wrapper meldet „unbekannt“ (die vollständige Fehlermeldung von `df` enthält den temporären Pfad, deshalb zeigen wir nur den ersten Teil). Im dritten Block testen wir die Auswertung mit einem fest vorgegebenen Beispieltext, der absichtlich einen Einhängepunkt mit Leerzeichen enthält, und prüfen die Schwellen an den Grenzwerten (80 ist schon eine Warnung, 90 schon kritisch).

> **Ergänzung aus der Nachrecherche**
>
> **Grenzen des Dienstes-Checks.** Ob eine Einheit existiert, verrät `systemctl is-active` nach unserer Erfahrung nicht: Für einen Namen, den es nicht gibt, kommt ebenfalls das Wort `inactive` und der Rückgabecode 3. Diese Aussage beruht auf Fachwissen und wurde in diesem Skript nicht in der Dokumentation nachgeschlagen; prüfen Sie das auf Ihrem System. Das Handbuch von `systemctl` nennt für „no such unit“ den LSB-Code 4 (siehe Lerneinheit 5.2), der aber nicht bei jedem Unterbefehl auftritt. Wer die Existenz sicher prüfen will, fragt zusätzlich ab, ob die Einheit geladen ist (zum Beispiel mit `systemctl show -p LoadState NAME`). Das Handbuch weist außerdem darauf hin, dass die LSB-Rückgabecodes nur eine unvollkommene Abbildung der Zustände sind und man Zustandswörter auswerten sollte, wie unser Wrapper es tut. In Containern ohne systemd fehlt `systemctl` ganz; dann liefert der Wrapper „unbekannt“, wie gewünscht.

Ein typischer Fehler bei Wrappern ist, die Ausgabe von `df` ohne `-P` zu verwenden und sich dann zu wundern, warum bei langen Gerätenamen (zum Beispiel LVM-Pfade wie `/dev/mapper/vg0-root`) die Spalten verrutschen. Ein zweiter ist, den Rückgabecode des Programms zu verschlucken: Wer `ausfuehren` aufruft und nur die Ausgabe betrachtet, hält eine Fehlermeldung von `df` womöglich für Messwerte. Das dritte ist der Mischtext: Wer Warnung und Messwert in einem Text ausgibt, den ein anderes Programm weiterverarbeiten soll, braucht ein stabiles Format; unser Wrapper beginnt die Zeile immer mit einem festen Wort (`PLATTE OK`), das sich leicht abfragen lässt.

```python
import subprocess

# FALSCH: lange Namen brechen die Zeile um, Spalten stimmen dann nicht
zeilen = """Filesystem                 1K-blocks    Used Available Use% Mounted on
/dev/mapper/vg0-sehr-langer-name
                           51474044 38000000  10838044  78% /
""".splitlines()
print(zeilen[1].split())

# RICHTIG: df -P erzwingt eine Zeile je Dateisystem
r = subprocess.run(["df", "-P", "/"], capture_output=True, text=True)
print(len(r.stdout.strip().splitlines()))

# FALSCH: Rückgabecode ignoriert, Fehlermeldung wird als Ergebnis gelesen
r = subprocess.run(["df", "-P", "/gibt/es/nicht"], capture_output=True, text=True)
print(repr(r.stdout), r.returncode)
```

```text
['/dev/mapper/vg0-sehr-langer-name']
2
'' 1
```

Im ersten Teil zeigt die zweite Zeile nur den Gerätenamen, alle Zahlen stehen auf der nächsten Zeile; eine Auswertung nach Spalten würde dort einen Fehler oder falsche Werte liefern. Beim richtigen Aufruf mit `-P` besteht die Ausgabe immer aus genau zwei Zeilen (Kopf und Dateisystem). Im dritten Teil sieht man, dass bei einem Fehler `stdout` leer ist und der Rückgabecode 1 lautet; die Fehlermeldung steht in `stderr`. Wer das übersieht, wertet leeren Text aus.

Damit ist der Meilenstein erreicht: Ein Wrapper, der Programme sicher aufruft, Fehler unterscheidet, eine eigene Kommandozeile mit Hilfe und Unterbefehlen besitzt und Rückgabecodes liefert, die ein anderes Programm weiterverwenden kann. Zur Wiederholung dient das Quiz: Es sind die Übungsaufgaben 1 bis 4 dieses Moduls, die Aufgaben 5 bis 8 verlangen eigene Programme. Planen Sie für das Quiz etwa eine halbe Stunde ein und lösen Sie die Programmieraufgaben erst danach.

> **Merke**
>
> Ein Wrapper trennt Ausführung (ein einziger `subprocess`-Aufruf mit Zeitlimit, `LC_ALL=C`, Liste statt Text) und Auswertung (reine Funktionen). Er unterscheidet „Befund“ (Dienst läuft nicht, Platte voll) von „Umgebungsfehler“ (Programm fehlt, `df` bricht ab) und liefert dafür verschiedene Rückgabecodes. Mit einer Attrappe für das aufgerufene Programm lässt er sich testen, ohne etwas Echtes zu verändern.

> **Praxisbeispiel**
>
> Genau dieses Werkzeug taucht später mehrfach wieder auf: In Modul 6 bekommt es Protokollierung und eine Fehlerbehandlung für unerwartete Ausnahmen, in Modul 9 läuft es als Cronjob oder systemd-Timer jede Minute, in Modul 12 schickt es bei Rückgabecode 2 eine Mail oder einen Webhook, und in Modul 14 schreiben Sie Tests dafür mit `pytest` und `unittest.mock`. Wer jetzt sauber trennt, spart dort viel Umbau.

## Übungsaufgaben zu Modul 05

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Starten Sie bei den Programmieraufgaben nur harmlose Programme (`echo`, `find`, `sort`) und arbeiten Sie in einem temporären Verzeichnis.

**Aufgabe 1.** Warum übergibt man einen Befehl an `subprocess.run()` als Liste und nicht als Zeichenkette? Was bewirkt `shell=True`, und wann ist es gefährlich? Welcher Unterschied besteht zwischen den Rückgabecodes `9` und `-9`?

**Aufgabe 2.** Welche drei Wege gibt es, wie ein `subprocess.run()`-Aufruf scheitern kann, und welche Ausnahme gehört zu jedem? Welche Informationen trägt ein `CalledProcessError`? Warum sollte jeder Aufruf in einem Cronjob ein `timeout` haben?

**Aufgabe 3.** Was unterscheidet ein Positionsargument von einer Option in `argparse`? Wofür stehen `action="store_true"`, `nargs="+"` und `choices=[...]`? Welchen Rückgabecode hat ein Programm, das mit falschen Argumenten gestartet wurde, und welchen bei `-h`? Warum schreibt man `main(argv=None)`?

**Aufgabe 4.** Finden Sie mindestens fünf Fehler oder Risiken im folgenden Programm und erklären Sie jeden.

```python
import argparse
import subprocess

parser = argparse.ArgumentParser()
parser.add_argument("--anzahl", type=bool)
parser.add_argument("datei", default="log.txt")
args = parser.parse_args()

ergebnis = subprocess.run("grep -c fehler " + args.datei, shell=True)
if ergebnis.stdout.strip() == "0":
    print("keine Fehler")
subprocess.run(["rm", "-rf", args.datei + "/*"])
```

**Aufgabe 5.** Schreiben Sie eine Funktion `starte(befehl, limit=5)`, die einen Befehl ausführt und immer ein Tupel `(code, stdout, stderr)` zurückgibt, ohne selbst eine Ausnahme auszulösen. Bei einem Startfehler oder Zeitlimit soll `code` gleich `None` sein und `stderr` eine kurze Erklärung enthalten. Testen Sie vier Fälle: `echo`, ein Kindprogramm mit Fehlercode, ein nicht vorhandenes Programm, ein Kindprogramm, das zu lange schläft.

**Aufgabe 6.** Schreiben Sie ein Programm `zaehle` nach dem Vorbild von `wc`: Es nimmt eine oder mehrere Dateien entgegen und gibt für jede die Zahl der Zeilen (`-l`) und/oder der Wörter (`-w`) aus. Sind beide Schalter nicht gesetzt, werden beide Zahlen ausgegeben. Verwenden Sie das Gerüst mit `main(argv=None)` und testen Sie in einem temporären Verzeichnis.

**Aufgabe 7.** Schreiben Sie eine Funktion `finde(basis, muster)`, die mit dem Programm `find` (ohne Shell) alle Dateien unterhalb von `basis` sucht, deren Name auf das Muster passt, und eine sortierte Liste relativer Pfade zurückgibt. Testen Sie sie auch mit einer Datei, deren Name Leerzeichen und ein Semikolon enthält.

**Aufgabe 8.** Bauen Sie ein kleines Werkzeug `altercheck` mit dem Unterbefehl `datei PFAD --max-alter TAGE`. Es prüft, ob die Datei existiert und höchstens so alt ist, wie erlaubt (zum Beispiel bei einer Sicherungsdatei), und verwendet die Rückgabecodes 0 (OK), 2 (kritisch) und 3 (unbekannt). Testen Sie es mit selbst angelegten Dateien, deren Änderungsdatum Sie mit `os.utime` setzen.

## Lösungen zu Modul 05

**Aufgabe 1.** Eine Liste übergibt jedes Argument unverändert und als Einheit an das Programm; Leerzeichen, Anführungszeichen oder Sonderzeichen in einem Dateinamen sind dann kein Problem. Ein Text müsste von der Shell zerlegt werden, und genau dabei entstehen Fehler und Sicherheitslücken. Mit `shell=True` startet Python eine Shell, die den Text interpretiert (Pipes, Platzhalter, Variablen, Befehlstrenner wie `;`). Gefährlich ist das, sobald Teile des Textes von außen kommen, weil ein Angreifer dann eigene Befehle einschleusen kann (Shell-Injection). Der Rückgabecode `9` bedeutet, dass das Programm sich selbst mit `exit(9)` beendet hat; `-9` bedeutet, dass es durch das Signal 9 (SIGKILL) beendet wurde, also von außen abgebrochen wurde.

**Aufgabe 2.** Erstens kann das Programm nicht starten: `FileNotFoundError` (nicht gefunden) oder `PermissionError` (nicht ausführbar). Zweitens kann es zu lange dauern: `subprocess.TimeoutExpired`. Drittens kann es mit einem Fehlercode enden; bei `check=True` entsteht `CalledProcessError`. Diese Ausnahme enthält `returncode`, `cmd`, `stdout` und `stderr` (die beiden letzten nur, wenn sie abgefangen wurden). Ein Zeitlimit ist in Cronjobs Pflicht, weil ein hängender Aufruf sonst nie endet und sich bei jedem Start ein weiterer hängender Prozess ansammelt, bis der Rechner überlastet ist.

**Aufgabe 3.** Positionsargumente haben keinen Bindestrich und sind durch ihre Stelle bestimmt; sie sind Pflicht, solange nicht `nargs="?"` oder `"*"` gesetzt ist. Optionen beginnen mit `-` oder `--`, sind in der Regel freiwillig und haben einen Standardwert. `store_true` macht aus einer Option einen Schalter (`False`, solange er fehlt). `nargs="+"` verlangt mindestens einen Wert und liefert eine Liste. `choices` beschränkt die erlaubten Werte. Falsche Argumente beenden das Programm mit Rückgabecode 2, `-h` mit Rückgabecode 0. Die Form `main(argv=None)` erlaubt, `main` im Test mit einer eigenen Liste aufzurufen, ohne eine echte Kommandozeile zu brauchen; im Normalfall liest `argparse` bei `None` die Kommandozeile.

**Aufgabe 4.** (1) `type=bool` ist wirkungslos, jeder nichtleere Text ergibt `True`; außerdem ist `--anzahl` offensichtlich eine Zahl und gehört zu `type=int`. (2) Ein Positionsargument mit `default` bleibt Pflicht, es fehlt `nargs="?"`. (3) Der Befehl wird als Text zusammengebaut und mit `shell=True` ausgeführt; ein Dateiname wie `x; rm -rf ~` würde ausgeführt (Shell-Injection). Richtig ist eine Liste: `["grep", "-c", "fehler", args.datei]`. (4) Die Ausgabe wurde nicht abgefangen (`capture_output=True, text=True` fehlen); `ergebnis.stdout` ist `None`, und `.strip()` löst einen `AttributeError` aus. Außerdem gibt `grep -c` bei null Treffern den Rückgabecode 1 aus, den das Programm ebenfalls nicht beachtet. (5) In der Listenform findet keine Platzhalterersetzung statt, `args.datei + "/*"` wird also wörtlich als Dateiname übergeben und löscht nichts; zugleich ist `rm -rf` mit einer Benutzereingabe ohne Probelauf und Prüfung gefährlich. Besser wäre `shutil.rmtree` oder `Path.unlink` nach einer Pfadprüfung (Modul 4). (6) Weder `check=True` noch ein `timeout` sind gesetzt, beide Aufrufe könnten still scheitern oder hängen.

**Aufgabe 5.**

```python
import subprocess
import sys

def starte(befehl, limit=5):
    try:
        r = subprocess.run(befehl, capture_output=True, text=True, timeout=limit)
    except (FileNotFoundError, PermissionError) as fehler:
        return None, "", f"Start fehlgeschlagen: {fehler.strerror}"
    except subprocess.TimeoutExpired:
        return None, "", f"Zeitlimit von {limit} s überschritten"
    return r.returncode, r.stdout, r.stderr

py = sys.executable
print(starte(["echo", "hallo"]))
print(starte([py, "-c", "import sys; sys.stderr.write('böse'); sys.exit(7)"]))
print(starte(["gibt-es-nicht"]))
print(starte([py, "-c", "import time; time.sleep(5)"], limit=0.3))
```

```text
(0, 'hallo\n', '')
(7, '', 'böse')
(None, '', 'Start fehlgeschlagen: No such file or directory')
(None, '', 'Zeitlimit von 0.3 s überschritten')
```

Die Funktion fängt genau die drei Fehlerwege aus Lerneinheit 5.2 ab, wobei `fehler.strerror` die Systemmeldung ohne den Dateinamen liefert. Einen Rückgabecode ungleich 0 behandelt sie nicht als Ausnahme, sondern gibt ihn dem Aufrufer zurück; ob 7 „schlimm“ ist, entscheidet nur dieser.

**Aufgabe 6.**

```python
import argparse
import os
import tempfile
from pathlib import Path

def main(argv=None):
    parser = argparse.ArgumentParser(prog="zaehle")
    parser.add_argument("dateien", nargs="+", help="eine oder mehrere Textdateien")
    parser.add_argument("-l", "--zeilen", action="store_true", help="Zeilen zählen")
    parser.add_argument("-w", "--woerter", action="store_true", help="Wörter zählen")
    args = parser.parse_args(argv)

    beides = not args.zeilen and not args.woerter
    for name in args.dateien:
        text = Path(name).read_text(encoding="utf-8")
        zahlen = []
        if args.zeilen or beides:
            zahlen.append(f"{len(text.splitlines()):>4}")
        if args.woerter or beides:
            zahlen.append(f"{len(text.split()):>4}")
        print(" ".join(zahlen), name)
    return 0

with tempfile.TemporaryDirectory() as tmp:
    os.chdir(tmp)
    Path("a.txt").write_text("eins zwei\ndrei\n", encoding="utf-8")
    Path("b.txt").write_text("a b c d e f\n", encoding="utf-8")
    main(["a.txt", "b.txt"])
    main(["-l", "a.txt"])
    main(["--woerter", "b.txt", "a.txt"])
    os.chdir("/")
```

```text
   2    3 a.txt
   1    6 b.txt
   2 a.txt
   6 b.txt
   3 a.txt
```

Die Variable `beides` fasst die Regel „ohne Schalter beide Zahlen“ in einem Ausdruck zusammen. Das `os.chdir` in das temporäre Verzeichnis dient nur dazu, dass in der Ausgabe die kurzen Dateinamen erscheinen; am Ende kehrt das Programm in ein Verzeichnis zurück, das sicher existiert, bevor das temporäre gelöscht wird.

**Aufgabe 7.**

```python
import subprocess
import tempfile
from pathlib import Path

def finde(basis, muster):
    r = subprocess.run(
        ["find", str(basis), "-type", "f", "-name", muster],
        capture_output=True, text=True, check=True, timeout=30)
    return sorted(str(Path(z).relative_to(basis)) for z in r.stdout.splitlines())

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    (basis / "unter").mkdir()
    (basis / "a b;c.txt").write_text("1")
    (basis / "x.log").write_text("2")
    (basis / "unter" / "y.txt").write_text("3")
    print(finde(basis, "*.txt"))
    print(finde(basis, "x.*"))
```

```text
['a b;c.txt', 'unter/y.txt']
['x.log']
```

Weil keine Shell beteiligt ist, wird das Muster `*.txt` unverändert an `find` übergeben, das es selbst auswertet, und der Dateiname mit Leerzeichen und Semikolon bleibt harmlos. Mit einer Shell müsste man das Muster in Anführungszeichen setzen, sonst würde die Shell den Stern schon vorher durch Dateinamen ersetzen. Eine Einschränkung: Dateinamen mit Zeilenumbruch würden die Auswertung per `splitlines()` stören; `find -print0` und `split("\0")` wären robuster. Reine Python-Lösung: `sorted(basis.rglob(muster))` leistet dasselbe ohne Kindprozess; `find` lohnt sich vor allem bei Optionen, die `pathlib` nicht kennt.

**Aufgabe 8.**

```python
import argparse
import os
import tempfile
import time
from pathlib import Path

OK, KRITISCH, UNBEKANNT = 0, 2, 3

def cmd_datei(args):
    pfad = Path(args.pfad)
    if not pfad.is_file():
        print(f"DATEI UNBEKANNT - {pfad.name} existiert nicht")
        return UNBEKANNT
    alter = (time.time() - pfad.stat().st_mtime) / 86400
    if alter > args.max_alter:
        print(f"DATEI KRITISCH - {pfad.name} ist {alter:.1f} Tage alt (Grenze {args.max_alter})")
        return KRITISCH
    print(f"DATEI OK - {pfad.name} ist {alter:.1f} Tage alt")
    return OK

def main(argv=None):
    parser = argparse.ArgumentParser(prog="altercheck")
    unter = parser.add_subparsers(dest="befehl", required=True, metavar="BEFEHL")
    datei = unter.add_parser("datei", help="Alter einer Datei prüfen")
    datei.add_argument("pfad")
    datei.add_argument("--max-alter", type=float, default=1.0, metavar="TAGE")
    datei.set_defaults(func=cmd_datei)
    args = parser.parse_args(argv)
    return args.func(args)

with tempfile.TemporaryDirectory() as tmp:
    frisch = Path(tmp) / "backup-neu.tgz"
    alt = Path(tmp) / "backup-alt.tgz"
    frisch.write_text("x")
    alt.write_text("x")
    zwei_tage_her = time.time() - 2.5 * 86400
    os.utime(alt, (zwei_tage_her, zwei_tage_her))

    print(main(["datei", str(frisch)]))
    print(main(["datei", str(alt)]))
    print(main(["datei", str(alt), "--max-alter", "3"]))
    print(main(["datei", str(Path(tmp) / "fehlt.tgz")]))
```

```text
DATEI OK - backup-neu.tgz ist 0.0 Tage alt
0
DATEI KRITISCH - backup-alt.tgz ist 2.5 Tage alt (Grenze 1.0)
2
DATEI OK - backup-alt.tgz ist 2.5 Tage alt
0
DATEI UNBEKANNT - fehlt.tgz existiert nicht
3
```

Die Funktion trennt die drei Fälle sauber: Fehlt die Datei, kann man keine Aussage über das Alter treffen, also „unbekannt“; ist sie zu alt, ist der Befund „kritisch“. Das Änderungsdatum der Testdatei haben wir mit `os.utime` um 2,5 Tage in die Vergangenheit gesetzt. Dieser Check ist in der Praxis wertvoll: Er meldet, wenn die nächtliche Sicherung nicht mehr läuft, weil die neueste Sicherungsdatei zu alt ist. In Modul 6 wird er Teil des Backup-Skripts.
