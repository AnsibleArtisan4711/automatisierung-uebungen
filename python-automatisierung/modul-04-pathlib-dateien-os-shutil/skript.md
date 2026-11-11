# Modul 04 - pathlib, Dateien lesen-schreiben, os-shutil

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (4.1 bis 4.5), danach folgen „Übungsaufgaben zu Modul 04“ (8 Aufgaben) und „Lösungen zu Modul 04“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 17.11.2026 bis Mo 23.11.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; Dateizugriff ist die Kernaufgabe der meisten Admin-Skripte)

**Meilenstein:** Dateisuche nach Alter und Größe

**Lernziele.** Nach diesem Modul können Sie Dateipfade mit `pathlib` plattformunabhängig bilden und zerlegen, Textdateien sicher lesen und schreiben, Verzeichnisse durchlaufen, Dateien kopieren, verschieben und archivieren und Dateien nach Alter und Größe auswählen. Sie wissen, welche Operationen gefährlich sind (Löschen, Überschreiben) und wie man sie mit einem Probelauf absichert.

| **Nr.** | **Lerneinheit**                      | **Datum**     | **Dauer** | **Prio** |
|:--------|:-------------------------------------|:--------------|:----------|:---------|
| 4.1     | pathlib Grundlagen                   | Di 17.11.2026 | 2 h       | 1        |
| 4.2     | Dateien lesen & schreiben (with)     | Mi 18.11.2026 | 2 h       | 1        |
| 4.3     | os/shutil, Verzeichnisse durchlaufen | Do 19.11.2026 | 2 h       | 1        |
| 4.4     | Dateisuche nach Alter & Größe        | Fr 20.11.2026 | 2 h       | 1        |
| 4.5     | Übung abschließen + Quiz             | Mo 23.11.2026 | 2 h       | 1        |
|         | **Summe**                            |               | **10 h**  |          |

> **Merke**
>
> Alle Beispiele dieses Moduls arbeiten in einem frisch angelegten temporären Verzeichnis, das am Ende automatisch verschwindet (`tempfile.TemporaryDirectory()`). So können Sie jedes Beispiel gefahrlos ausführen, ohne Ihre eigenen Dateien zu berühren. Verwenden Sie diese Technik auch für Ihre eigenen Experimente.

## 4.1 pathlib Grundlagen

Jedes Skript, das mit Dateien arbeitet, muss Pfade bilden, zerlegen und prüfen. Früher geschah das mit Strings und Funktionen aus dem Modul `os.path`: Man klebte Teile mit `+` und einem Schrägstrich aneinander und hoffte, dass das System den Rest verstand. Das ist fehleranfällig, denn Windows trennt mit Backslash, Linux mit Schrägstrich, und doppelte oder fehlende Trenner führen zu schwer auffindbaren Fehlern. Das Modul `pathlib` (seit Python 3.4 in der Standardbibliothek) löst das, indem es Pfade als *Objekte* der Klasse `Path` behandelt. Ein `Path` kennt seine Bestandteile, kann sich selbst zerlegen und prüfen, und den Pfadtrenner des jeweiligen Systems setzt es automatisch.

Ein `Path` entsteht aus einem oder mehreren Strings: `Path("/var/log")`. Mit dem Operator `/` hängt man Teile an, was sehr gut lesbar ist: `Path("/var") / "log" / "syslog"`. Die Klasse kennt außerdem nützliche Ausgangspunkte: `Path.cwd()` liefert das aktuelle Arbeitsverzeichnis (*current working directory*), `Path.home()` das Home-Verzeichnis des Benutzers. Mit `expanduser()` wird eine Tilde zum Home-Verzeichnis, und `resolve()` macht aus einem relativen Pfad einen absoluten und löst symbolische Verweise auf.

Ein Pfad besteht aus Teilen, die als Eigenschaften zur Verfügung stehen. Das folgende Beispiel zerlegt den Pfad einer rotierten Logdatei. Beachten Sie die Unterschiede zwischen `name` (Dateiname mit Endung), `stem` (Name ohne die letzte Endung), `suffix` (die letzte Endung) und `suffixes` (alle Endungen).

```python
from pathlib import Path

pfad = Path("/var/log/nginx/access.log.1.gz")

print("name:    ", pfad.name)
print("stem:    ", pfad.stem)
print("suffix:  ", pfad.suffix)
print("suffixes:", pfad.suffixes)
print("parent:  ", pfad.parent)
print("parts:   ", pfad.parts)
print("anchor:  ", pfad.anchor)
print("absolut: ", pfad.is_absolute())

neu = pfad.with_suffix(".bak")
print(neu)
print(pfad.with_name("error.log"))
print(Path("/etc") / "ssh" / "sshd_config")
print(Path("daten") / "bericht.txt")
```

```text
name:     access.log.1.gz
stem:     access.log.1
suffix:   .gz
suffixes: ['.log', '.1', '.gz']
parent:   /var/log/nginx
parts:    ('/', 'var', 'log', 'nginx', 'access.log.1.gz')
anchor:   /
absolut:  True
/var/log/nginx/access.log.1.bak
/var/log/nginx/error.log
/etc/ssh/sshd_config
daten/bericht.txt
```

Das Programm zeigt, dass `pathlib` Pfade rein als Text zerlegt; die Datei muss dafür nicht existieren. Mit `with_suffix` und `with_name` erzeugt man abgeleitete Pfade, ohne Strings zusammenzubauen. Das Muster `datei.with_suffix(".bak")` ersetzt nur die letzte Endung, aus `access.log.1.gz` wird also `access.log.1.bak`. Das letzte Beispiel zeigt, dass der Operator `/` auch bei relativen Pfaden funktioniert. Auf einem Windows-System erscheinen dabei Backslashes; derselbe Code läuft also unverändert auf beiden Plattformen.

Neben dem Zerlegen kann ein `Path` den Dateisystemzustand abfragen und verändern. Die wichtigsten Abfragen sind `exists()`, `is_file()`, `is_dir()` und `is_symlink()`. Mit `stat()` erhält man ein Objekt mit Größe (`st_size`), Zeitstempeln (`st_mtime` für die letzte Änderung) und Rechten (`st_mode`). Verzeichnisse legt man mit `mkdir()` an; die Option `parents=True` erzeugt fehlende Zwischenverzeichnisse, `exist_ok=True` unterdrückt den Fehler, wenn das Verzeichnis schon existiert. Den Inhalt eines Verzeichnisses liefert `iterdir()`, und mit `glob()` sowie `rglob()` sucht man nach Mustern (`*` steht für beliebige Zeichen, `**` für beliebig viele Verzeichnisebenen).

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)

    projekt = basis / "projekt" / "logs"
    projekt.mkdir(parents=True, exist_ok=True)
    projekt.mkdir(parents=True, exist_ok=True)     # zweites Mal: kein Fehler

    (projekt / "a.log").write_text("Zeile 1\n")
    (projekt / "b.log").write_text("Zeile 1\nZeile 2\n")
    (projekt / "notiz.txt").write_text("hallo")
    (basis / "projekt" / "README.md").write_text("# Projekt")

    print(projekt.exists(), projekt.is_dir(), (projekt / "a.log").is_file())
    print((projekt / "fehlt.log").exists())

    print("iterdir:", sorted(p.name for p in projekt.iterdir()))
    print("glob *.log:", sorted(p.name for p in projekt.glob("*.log")))
    print("rglob *:", sorted(str(p.relative_to(basis)) for p in basis.rglob("*")))

    info = (projekt / "b.log").stat()
    print("Größe:", info.st_size, "Byte")
    print("Mode:", oct(info.st_mode & 0o777))
```

```text
True True True
False
iterdir: ['a.log', 'b.log', 'notiz.txt']
glob *.log: ['a.log', 'b.log']
rglob *: ['projekt', 'projekt/README.md', 'projekt/logs', 'projekt/logs/a.log', 'projekt/logs/b.log', 'projekt/logs/notiz.txt']
Größe: 16 Byte
Mode: 0o644
```

Der Block unter `with tempfile.TemporaryDirectory() as tmp` legt ein Verzeichnis an, das nach Ende des Blocks samt Inhalt gelöscht wird. Alle weiteren Pfade werden relativ zu `basis` gebildet. Die Methode `mkdir(parents=True, exist_ok=True)` legt die Verzeichnisse `projekt` und `projekt/logs` in einem Schritt an; der zweite Aufruf wäre ohne `exist_ok=True` mit einem `FileExistsError` gescheitert. `write_text()` schreibt eine kleine Datei (mehr dazu in Lerneinheit 4.2). `relative_to()` kürzt einen Pfad um ein Basisverzeichnis, was Ausgaben lesbar macht. Die Rechte `0o644` (Besitzer lesen und schreiben, alle anderen nur lesen) hängen von der `umask` des Systems ab; bei Ihnen kann ein anderer Wert erscheinen.

> **Ergänzung aus der Nachrecherche**
>
> **Neuerungen in pathlib (abgerufen aus der Python-3.14-Dokumentation).** Die aktuelle Dokumentation vermerkt bei den Methoden von `Path` die folgenden Versionen: `Path.walk()` (Verzeichnisbaum durchlaufen) ist seit Python 3.12 verfügbar, `full_match()` und `from_uri()` seit 3.13, und die Methoden `copy()`, `copy_into()`, `move()`, `move_into()` sowie das Attribut `info` sind in Python 3.14 neu. Bei `exists()` und verwandten Methoden wurde in 3.12 der Parameter `follow_symlinks` ergänzt. Dieses Skript verlangt Python 3.11 oder neuer und setzt deshalb für Kopieren und Verschieben auf `shutil` (Lerneinheit 4.3), das in allen Versionen funktioniert. Wenn Ihr System Python 3.14 oder neuer hat, können Sie `Path.copy()` und `Path.move()` verwenden; die Beispiele hier wurden nur mit Python 3.13 ausgeführt.

Zu den typischen Fehlern gehört das Zusammenbauen von Pfaden mit Textverkettung. Das folgende Beispiel zeigt die fehleranfällige Variante und die richtige.

```python
from pathlib import Path

# FALSCH: Strings verketten, Trenner von Hand
verzeichnis = "/var/log/"
datei = "syslog"
falsch = verzeichnis + "/" + datei
print(falsch)

# RICHTIG: Path-Objekte mit dem Operator /
richtig = Path("/var/log/") / "syslog"
print(richtig)

# FALSCH: ein absoluter Pfad auf der rechten Seite ersetzt alles davor
print(Path("/var/log") / "/etc/passwd")

# Pfade vergleichen: Path normalisiert doppelte Schrägstriche
print(Path("/var//log/") == Path("/var/log"))
```

```text
/var/log//syslog
/var/log/syslog
/etc/passwd
True
```

Das Verketten mit Strings erzeugt hier einen doppelten Schrägstrich. Ein Detail, das überrascht: Ist der rechte Operand von `/` selbst ein absoluter Pfad, verwirft `pathlib` den linken Teil. Das ist beabsichtigt, kann aber gefährlich sein, wenn der rechte Teil aus einer Benutzereingabe stammt. Wer Dateien in einem festen Verzeichnis ablegt, sollte den entstandenen Pfad mit `resolve()` prüfen und sicherstellen, dass er unterhalb des erlaubten Verzeichnisses bleibt, sonst genügt eine Eingabe wie `../../etc/passwd`, um aus dem Verzeichnis auszubrechen (*Path Traversal*).

> **Merke**
>
> Verwenden Sie `pathlib.Path` statt Textverkettung: `Path("/var") / "log"`. Merken Sie sich die Bestandteile `name`, `stem`, `suffix`, `parent`, die Abfragen `exists()`, `is_file()`, `is_dir()` und das Anlegen mit `mkdir(parents=True, exist_ok=True)`. Ein absoluter Pfad rechts von `/` ersetzt den linken Teil.

> **Praxisbeispiel**
>
> Jedes Skript, das auf Linux und Windows laufen soll, oder das mit Benutzereingaben für Dateinamen arbeitet, profitiert von `pathlib`. Ein typischer Fall: Ein Backup-Skript bildet aus Zielverzeichnis, Rechnername und Datum den Namen der Sicherung (`ziel / rechner / f"{datum}.tar.gz"`). Mit `Path` entfallen die Sonderfälle für Trenner, und `with_suffix`, `stem` und `parent` erledigen die Namensarbeit.

## 4.2 Dateien lesen & schreiben (with)

Eine Datei ist für ein Programm ein Datenstrom, den es öffnen, lesen oder beschreiben und wieder schließen muss. Die Funktion `open()` öffnet eine Datei und liefert ein *Dateiobjekt*. Das wichtigste Argument neben dem Pfad ist der *Modus*: `"r"` liest (Vorgabe), `"w"` schreibt und *leert dabei eine vorhandene Datei*, `"a"` hängt am Ende an, `"x"` legt eine neue Datei an und scheitert, wenn sie schon existiert. Ein angehängtes `b` (etwa `"rb"`) öffnet im Binärmodus, bei dem man Bytes statt Text liest; ohne `b` arbeitet Python im Textmodus.

Wichtig ist die Frage der Kodierung (Modul 1): Im Textmodus wandelt Python Bytes in Zeichen um, und dazu muss es wissen, welche Kodierung die Datei hat. Ohne Angabe benutzt Python die Voreinstellung des Systems, die je nach Plattform und Einstellungen variiert (unter Windows war das lange eine Codepage, nicht UTF-8). Deshalb gilt die Regel: *Geben Sie die Kodierung immer ausdrücklich an*, in aller Regel `encoding="utf-8"`.

Datei und Programm sind über ein Betriebssystem-Handle verbunden, das man freigeben muss. Wer das vergisst, riskiert, dass geschriebene Daten nicht vollständig auf die Platte gelangen und dass Handles aufgebraucht werden. Der sichere Weg ist die `with`-Anweisung. Sie öffnet die Datei und schließt sie am Ende des Blocks *garantiert*, auch wenn im Block ein Fehler auftritt. Fast jedes Dateibeispiel sollte so aussehen:

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "hosts.txt"

    # Schreiben: "w" legt an oder leert
    with open(datei, "w", encoding="utf-8") as f:
        f.write("srv01\n")
        f.write("srv02\n")
        f.writelines(["srv03\n", "srv04\n"])
        print("srv05", file=f)

    # Lesen: komplett als ein String
    with open(datei, "r", encoding="utf-8") as f:
        inhalt = f.read()
    print(repr(inhalt))

    # Lesen: zeilenweise in einer Schleife
    with open(datei, encoding="utf-8") as f:
        for nummer, zeile in enumerate(f, start=1):
            print(nummer, zeile.rstrip("\n"))

    # Anhängen
    with open(datei, "a", encoding="utf-8") as f:
        f.write("srv06\n")

    # Alle Zeilen als Liste
    with open(datei, encoding="utf-8") as f:
        zeilen = f.read().splitlines()
    print(len(zeilen), zeilen[-1])
```

```text
'srv01\nsrv02\nsrv03\nsrv04\nsrv05\n'
1 srv01
2 srv02
3 srv03
4 srv04
5 srv05
6 srv06
```

Das Beispiel zeigt die Grundoperationen. `write()` schreibt einen String und fügt *keinen* Zeilenumbruch hinzu; den muss man selbst setzen (`"\n"`). `writelines()` schreibt eine Folge von Strings, ebenfalls ohne Umbrüche. Mit `print(..., file=f)` schreibt man in eine Datei wie auf den Bildschirm, und `print` hängt den Umbruch an. Beim Lesen liefert `read()` den gesamten Inhalt als einen String. Der Zeilenumbruch bleibt in jedem Eintrag erhalten, was die `repr`-Ausgabe zeigt. Eine Schleife über das Dateiobjekt liefert Zeile für Zeile, *mit* dem abschließenden `\n`, das man mit `rstrip("\n")` entfernt. Die Zeilenschleife ist die richtige Wahl für große Dateien: Python liest dann jeweils nur eine Zeile in den Speicher, während `read()` die ganze Datei lädt. Bei einer Logdatei von mehreren Gigabyte ist das der Unterschied zwischen einem laufenden Skript und einem Speicherüberlauf. Die Methode `splitlines()` zerlegt einen String ohne Umbrüche in Zeilen.

Für kleine Dateien gibt es bei `pathlib` Kurzformen: `read_text()`, `write_text()`, `read_bytes()` und `write_bytes()` öffnen, lesen oder schreiben und schließen in einem Schritt. Sie sind bequem, geben aber die gesamte Datei auf einmal zurück.

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "notiz.txt"

    datei.write_text("Größe: 5 GiB\nStatus: OK\n", encoding="utf-8")
    text = datei.read_text(encoding="utf-8")
    print(text.splitlines())

    daten = datei.read_bytes()
    print(len(text), len(daten))

    # Fehlerfall: Datei nicht vorhanden
    try:
        (Path(tmp) / "gibtsnicht.txt").read_text(encoding="utf-8")
    except FileNotFoundError as fehler:
        print("Fehler:", type(fehler).__name__)

    # Modus x: bestehende Datei nicht überschreiben
    try:
        open(datei, "x", encoding="utf-8")
    except FileExistsError:
        print("Datei existiert bereits, nicht überschrieben")
```

```text
['Größe: 5 GiB', 'Status: OK']
24 26
Fehler: FileNotFoundError
Datei existiert bereits, nicht überschrieben
```

Die Datei enthält 24 Zeichen, belegt aber 26 Byte, weil das Zeichen ö und das ß in UTF-8 je zwei Byte brauchen (Modul 1). Wer eine nicht vorhandene Datei liest, erhält einen `FileNotFoundError`; wer mit `"x"` eine vorhandene Datei anlegen will, einen `FileExistsError`. Beide gehören zur Klasse `OSError` und werden in Modul 6 systematisch behandelt. Hier sieht man schon das Muster: Die Operationen, die von der Außenwelt abhängen, packt man in `try`/`except`.

Das häufigste Problem im Alltag ist das versehentliche Überschreiben. Wer eine Datei mit `"w"` öffnet, löscht ihren bisherigen Inhalt sofort, noch bevor etwas Neues geschrieben wurde. Stürzt das Skript danach ab, ist die Datei leer, und alles Bisherige ist verloren. Bei wichtigen Dateien wie Konfigurationen schreibt man deshalb zuerst in eine *temporäre* Datei im selben Verzeichnis und ersetzt dann die Originaldatei in einem Schritt; das Ersetzen ist auf einem Dateisystem eine unteilbare Operation, sodass es nie einen Zustand mit halber Datei gibt.

```python
import tempfile
from pathlib import Path

def schreibe_sicher(ziel, text):
    """Schreibt erst in eine temporäre Datei, ersetzt dann das Ziel."""
    ziel = Path(ziel)
    temp = ziel.with_name(ziel.name + ".tmp")
    temp.write_text(text, encoding="utf-8")
    temp.replace(ziel)          # ersetzt in einem Schritt

with tempfile.TemporaryDirectory() as tmp:
    konfig = Path(tmp) / "app.conf"
    schreibe_sicher(konfig, "port=22\n")
    schreibe_sicher(konfig, "port=2222\n")
    print(konfig.read_text(encoding="utf-8").strip())
    print(sorted(p.name for p in Path(tmp).iterdir()))
```

```text
port=2222
['app.conf']
```

Die Funktion schreibt zuerst `app.conf.tmp` und benennt sie dann mit `replace()` in `app.conf` um; am Ende bleibt nur die Zieldatei übrig. Das Muster begegnet Ihnen bei Konfigurationsdateien, Zustandsdateien und überall dort, wo ein Abbruch mitten im Schreiben keine halbe Datei hinterlassen darf. Ergänzend sollten Sie wissen, dass beim Textmodus Zeilenumbrüche je nach Plattform übersetzt werden: Unter Linux ist der Umbruch das Zeichen `\n`, unter Windows `\r\n`. Python gleicht das beim Lesen und Schreiben für Sie aus.

Zu den klassischen Fehlern gehört, die Kodierung wegzulassen, eine Datei ohne `with` zu öffnen und nicht zu schließen, im Modus `"w"` versehentlich Daten zu löschen und beim Lesen nicht zu bedenken, dass jede Zeile einen Zeilenumbruch am Ende trägt. Der letzte Punkt lässt Vergleiche scheitern: `zeile == "srv01"` ist falsch, wenn die Zeile `"srv01\n"` lautet. Man bereinigt mit `strip()` oder `rstrip()`.

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "hosts.txt"
    datei.write_text("srv01\nsrv02\n", encoding="utf-8")

    # FALSCH: Zeilenumbruch bleibt hängen
    with open(datei, encoding="utf-8") as f:
        erste = f.readline()
    print(erste == "srv01", repr(erste))

    # RICHTIG
    print(erste.strip() == "srv01")

    # FALSCH: "w" löscht den Inhalt sofort
    with open(datei, "w", encoding="utf-8") as f:
        pass
    print(len(datei.read_text(encoding="utf-8")), "Zeichen übrig")
```

```text
False 'srv01\n'
True
0 Zeichen übrig
```

> **Merke**
>
> Öffnen Sie Dateien mit `with open(pfad, modus, encoding="utf-8")`, damit sie garantiert geschlossen werden. `"w"` leert die Datei, `"a"` hängt an, `"x"` verweigert das Überschreiben. Zeilen tragen einen Zeilenumbruch am Ende (`rstrip`). Große Dateien liest man zeilenweise, nicht mit `read()`. Wichtige Dateien schreibt man über eine temporäre Datei und ersetzt sie dann.

> **Praxisbeispiel**
>
> Das Auswerten von Logdateien (Log-Analyzer v2 in Lerneinheit 4.5), das Lesen von Konfigurationsdateien, das Schreiben von Berichten und das Erzeugen von Exportdateien sind alles Anwendungen dieser Lerneinheit. Dabei gilt als Faustregel für Admins: Lesen ist harmlos, Schreiben nicht. Vor jedem Schreiben in eine bestehende Datei steht die Frage „Was passiert, wenn das Skript hier abbricht?“, und die Antwort ist selten „nichts“.

## 4.3 os/shutil, Verzeichnisse durchlaufen

Neben `pathlib` stehen in der Standardbibliothek die älteren Module `os` und `shutil`. Sie sind nicht überholt: `os` bietet den Zugang zum Betriebssystem (Umgebungsvariablen, Prozess-ID, Dateirechte, Verzeichnisbaum), und `shutil` („shell utilities“) liefert die Operationen höherer Ordnung, die man von der Shell kennt: kopieren, verschieben, löschen, archivieren. Sie begegnen beiden Modulen in unzähligen vorhandenen Skripten, weshalb man sie lesen können muss.

Aus `os` brauchen Sie im Alltag vor allem `os.getcwd()` (aktuelles Verzeichnis), `os.environ` und `os.getenv()` (Umgebungsvariablen), `os.listdir()` und `os.scandir()` (Verzeichnisinhalt), `os.walk()` (Baum durchlaufen), `os.stat()` (Dateiinformationen), `os.remove()` (Datei löschen), `os.rename()` (umbenennen), `os.makedirs()` (Verzeichnisse anlegen) und `os.chmod()` (Rechte ändern). Fast jede dieser Funktionen hat inzwischen ein Gegenstück bei `pathlib`; für neuen Code bevorzugt man `pathlib`.

```python
import os
import tempfile
from pathlib import Path

home = os.getenv("HOME")
print(home is not None)
print(os.getenv("GIBT_ES_NICHT", "Vorgabe"))
print(os.name, os.sep)

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    (basis / "a").mkdir()
    (basis / "a" / "x.txt").write_text("x")
    (basis / "b.txt").write_text("bb")

    print(sorted(os.listdir(basis)))

    with os.scandir(basis) as eintraege:
        for eintrag in sorted(eintraege, key=lambda e: e.name):
            art = "Verzeichnis" if eintrag.is_dir() else "Datei"
            print(f"{eintrag.name:6} {art}")

    os.rename(basis / "b.txt", basis / "c.txt")
    os.chmod(basis / "c.txt", 0o600)
    print(oct(os.stat(basis / "c.txt").st_mode & 0o777))
    os.remove(basis / "c.txt")
    print(sorted(os.listdir(basis)))
```

```text
True
Vorgabe
posix /
['a', 'b.txt']
a      Verzeichnis
b.txt  Datei
0o600
['a']
```

Das Beispiel nutzt `os.getenv()`, das den Wert einer Umgebungsvariablen liefert oder bei fehlender Variable den angegebenen Vorgabewert (ohne Angabe `None`). `os.scandir()` ist schneller als `listdir()`, weil es zu jedem Eintrag gleich mitliefert, ob es sich um eine Datei oder ein Verzeichnis handelt, ohne weitere Abfragen; die `with`-Anweisung schließt den Iterator sauber. Die Rechte `0o600` bedeuten: Besitzer darf lesen und schreiben, sonst niemand. Das ist die richtige Einstellung für Dateien mit Zugangsdaten.

Der wichtigste Baustein für Dateisuchen ist `os.walk()`. Die Funktion durchläuft ein Verzeichnis und alle seine Unterverzeichnisse (rekursiv) und liefert für jedes Verzeichnis ein Tupel aus drei Teilen: den Pfad des aktuellen Verzeichnisses, die Liste seiner Unterverzeichnisse und die Liste seiner Dateien. In einer `for`-Schleife entpackt man das Tupel sofort.

```python
import os
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    (basis / "logs" / "alt").mkdir(parents=True)
    (basis / "logs" / "app.log").write_text("1234567890")
    (basis / "logs" / "alt" / "app.log.1").write_text("12345")
    (basis / "readme.txt").write_text("abc")

    for ordner, unterordner, dateien in os.walk(basis):
        relativ = Path(ordner).relative_to(basis)
        print(f"[{relativ}] Unterordner={sorted(unterordner)} Dateien={sorted(dateien)}")

    gesamt = 0
    for ordner, unterordner, dateien in os.walk(basis):
        for name in dateien:
            gesamt += os.path.getsize(os.path.join(ordner, name))
    print("Gesamtgröße:", gesamt, "Byte")

    # dasselbe mit pathlib
    gesamt2 = sum(p.stat().st_size for p in basis.rglob("*") if p.is_file())
    print(gesamt2 == gesamt)
```

```text
[.] Unterordner=['logs'] Dateien=['readme.txt']
[logs] Unterordner=['alt'] Dateien=['app.log']
[logs/alt] Unterordner=[] Dateien=['app.log.1']
Gesamtgröße: 18 Byte
True
```

Das Tupel aus `os.walk` hat die Reihenfolge Ordnerpfad, Unterordner, Dateien. Die Dateinamen enthalten nur den Namen, nicht den Pfad, deshalb setzt man den vollständigen Pfad mit `os.path.join(ordner, name)` zusammen (oder mit `Path(ordner) / name`). Die Summe der Dateigrößen ist ein Mini-Werkzeug zum Berechnen des belegten Platzes, ähnlich wie `du`. Die Variante mit `rglob("*")` aus `pathlib` ist kürzer, liefert aber Dateien und Verzeichnisse gemischt, weshalb man mit `is_file()` filtert. Seit Python 3.12 gibt es zusätzlich `Path.walk()`, das wie `os.walk()` arbeitet und `Path`-Objekte liefert. Beide Funktionen folgen symbolischen Verweisen auf Verzeichnisse standardmäßig *nicht*, was Endlosschleifen durch zirkuläre Verweise vermeidet.

Die Module `shutil` und `os` ergänzen sich bei den Dateioperationen. Aus `shutil` benötigen Sie: `copy(quelle, ziel)` (kopiert Inhalt und Rechte), `copy2()` (zusätzlich Zeitstempel und weitere Metadaten), `copytree()` (kopiert einen ganzen Baum, mit `dirs_exist_ok=True` auch in ein vorhandenes Ziel), `move()` (verschiebt, auch über Dateisystemgrenzen hinweg), `rmtree()` (löscht einen ganzen Baum), `disk_usage()` (Gesamt, belegt und frei in Byte), `which()` (sucht ein Programm im `PATH`) und `make_archive()` (erzeugt ein Archiv, zum Beispiel `.tar.gz` oder `.zip`).

```python
import shutil
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    quelle = basis / "quelle"
    quelle.mkdir()
    (quelle / "a.txt").write_text("Inhalt A")
    (quelle / "unter").mkdir()
    (quelle / "unter" / "b.txt").write_text("Inhalt B")

    # Datei kopieren
    shutil.copy2(quelle / "a.txt", basis / "kopie.txt")
    print((basis / "kopie.txt").read_text())

    # Baum kopieren
    shutil.copytree(quelle, basis / "sicherung")
    print(sorted(str(p.relative_to(basis)) for p in (basis / "sicherung").rglob("*")))

    # Verschieben
    shutil.move(basis / "kopie.txt", basis / "unter_neu.txt")
    print((basis / "kopie.txt").exists(), (basis / "unter_neu.txt").exists())

    # Archiv erzeugen und Inhalt prüfen
    archiv = shutil.make_archive(str(basis / "backup"), "gztar", root_dir=quelle)
    print(Path(archiv).name, Path(archiv).stat().st_size > 0)

    # Baum löschen
    shutil.rmtree(basis / "sicherung")
    print((basis / "sicherung").exists())

gesamt, belegt, frei = shutil.disk_usage("/")
print(gesamt > belegt > 0, frei > 0)
print(shutil.which("python3") is not None, shutil.which("gibt-es-nicht"))
```

```text
Inhalt A
['sicherung/a.txt', 'sicherung/unter', 'sicherung/unter/b.txt']
False True
backup.tar.gz True
False
True True
True None
```

Die Wahl zwischen den Kopierfunktionen sollte bewusst erfolgen. `copy()` reicht für gewöhnliche Fälle; `copy2()` erhält den ursprünglichen Änderungszeitstempel, was für Sicherungen wichtig ist, weil man sonst später nicht mehr erkennt, wie alt eine Datei tatsächlich ist. `make_archive()` hat als erstes Argument den Basisnamen des Archivs ohne Endung, als zweites das Format und gibt den Namen des erzeugten Archivs zurück. `disk_usage()` liefert ein Tupel in Byte; zusammen mit der Funktion `format_bytes()` aus Modul 3 ergibt sich daraus eine Plattenprüfung in wenigen Zeilen.

Die Operationen `rmtree()`, `os.remove()` und das Überschreiben mit `move()` oder `copy()` sind *nicht rückgängig zu machen*: Es gibt keinen Papierkorb. Ein Tippfehler im Pfad, eine leere Variable oder ein falsch gebildeter Pfad kann ein ganzes Verzeichnis löschen. Üblich sind deshalb drei Schutzmaßnahmen. Erstens ein *Probelauf* (*Dry-Run*): Das Skript gibt zunächst nur aus, was es tun würde, und führt es erst nach ausdrücklicher Bestätigung aus. Zweitens eine Prüfung des Pfads vor dem Löschen, etwa dass er nicht `/` oder das Home-Verzeichnis ist und unterhalb eines erwarteten Verzeichnisses liegt. Drittens, bei wichtigen Daten, zuerst verschieben statt löschen (in ein Archivverzeichnis) und erst später endgültig aufräumen.

```python
import shutil
import tempfile
from pathlib import Path

def loesche_baum(pfad, erlaubte_basis, wirklich=False):
    """Löscht einen Verzeichnisbaum nur unterhalb von erlaubte_basis."""
    pfad = Path(pfad).resolve()
    basis = Path(erlaubte_basis).resolve()
    if pfad == basis or basis not in pfad.parents:
        raise ValueError(f"Verweigert: {pfad} liegt nicht unterhalb von {basis}")
    if not wirklich:
        print(f"DRY-RUN: würde {pfad.name} löschen")
        return
    shutil.rmtree(pfad)
    print(f"gelöscht: {pfad.name}")

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    (basis / "alt").mkdir()
    loesche_baum(basis / "alt", basis)                    # Probelauf
    loesche_baum(basis / "alt", basis, wirklich=True)     # echt
    try:
        loesche_baum(basis, basis, wirklich=True)         # Basis selbst: verweigert
    except ValueError as fehler:
        print("Fehler:", str(fehler).split(":")[0])
```

```text
DRY-RUN: würde alt löschen
gelöscht: alt
Fehler: Verweigert
```

Die Funktion löst beide Pfade mit `resolve()` auf (damit weder `..` noch symbolische Verweise täuschen können), verlangt, dass die Basis ein Elternverzeichnis des Ziels ist, und arbeitet standardmäßig im Probelauf. Dieses „Standardmäßig sicher“-Prinzip ist eines der wichtigsten Muster der Systemautomatisierung: Ein Skript, das etwas Zerstörerisches tut, sollte es nur dann tun, wenn man es ausdrücklich verlangt. In Modul 5 erhalten solche Schalter eine saubere Kommandozeilenschnittstelle (`--apply`, `--dry-run`).

> **Merke**
>
> `os` spricht mit dem Betriebssystem (Umgebung, Rechte, `os.walk`), `shutil` kopiert, verschiebt, löscht und archiviert. `os.walk()` liefert je Verzeichnis (Pfad, Unterordner, Dateien). Verwenden Sie `copy2()`, wenn Zeitstempel erhalten bleiben sollen. Löschen ist endgültig: Probelauf, Pfadprüfung und „erst verschieben“ sind Pflicht.

> **Praxisbeispiel**
>
> Aufräumskripte für Logverzeichnisse, Sicherungsskripte, das Verteilen von Konfigurationsdateien und die Kontrolle von Plattenbelegungen sind der klassische Einsatz dieser Module. Ein häufiger Fehler in der Praxis ist ein Aufräumskript, das ein leeres Verzeichnis als Argument bekommt und den Pfad dadurch zu `/` oder einem viel zu hohen Verzeichnis wird. Die Schutzfunktion aus diesem Abschnitt verhindert genau das.

## 4.4 Dateisuche nach Alter & Größe

Eine der häufigsten Aufgaben für Administratoren lautet: „Finde alle Dateien, die älter als 30 Tage und größer als 100 MiB sind.“ Auf der Kommandozeile würde man `find` verwenden; in Python bauen wir dasselbe mit den bisherigen Bausteinen nach und gewinnen dadurch die Möglichkeit, die Treffer weiterzuverarbeiten: formatiert ausgeben, sortieren, summieren, verschieben, in eine Datei schreiben.

Die nötigen Informationen liefert `stat()`. Das Ergebnis hat unter anderem die Größe in Byte (`st_size`) und drei Zeitstempel: `st_mtime` (Zeitpunkt der letzten Änderung des Inhalts), `st_atime` (letzter Zugriff) und `st_ctime` (unter Linux der Zeitpunkt der letzten Änderung der Metadaten, nicht der Erstellung). Die Zeiten sind Sekunden seit dem 1. Januar 1970, UTC (die sogenannte *Unix-Zeit*). Für das „Alter einer Datei“ nimmt man fast immer `st_mtime`. Das Alter in Sekunden ist die aktuelle Zeit, die man mit `time.time()` erhält, minus `st_mtime`; durch 86 400 geteilt ergibt sich das Alter in Tagen.

Damit die Beispiele reproduzierbar sind, setzen wir die Änderungszeit der Testdateien selbst mit `os.utime()`. Das folgende Hilfsstück erzeugt einen kleinen Baum aus Dateien unterschiedlicher Größe und unterschiedlichen Alters. Danach folgt die Suchfunktion.

```python
import os
import tempfile
import time
from pathlib import Path

TAG = 86400

def baue_testbaum(basis):
    """Legt Dateien mit definiertem Alter (Tage) und Größe (Byte) an."""
    spezifikation = [
        ("logs/app.log", 5000, 2),
        ("logs/app.log.1", 8000, 40),
        ("logs/app.log.2", 3000, 90),
        ("backup/db.sql", 20000, 45),
        ("backup/db_alt.sql", 15000, 400),
        ("tmp/klein.tmp", 100, 100),
    ]
    jetzt = time.time()
    for name, groesse, alter_tage in spezifikation:
        pfad = basis / name
        pfad.parent.mkdir(parents=True, exist_ok=True)
        pfad.write_bytes(b"x" * groesse)
        zeit = jetzt - alter_tage * TAG
        os.utime(pfad, (zeit, zeit))

def finde(basis, min_alter_tage, min_groesse=0):
    """Gibt (Pfad, Größe, Alter in Tagen) für passende Dateien zurück."""
    jetzt = time.time()
    treffer = []
    for pfad in basis.rglob("*"):
        if not pfad.is_file():
            continue
        info = pfad.stat()
        alter = (jetzt - info.st_mtime) / TAG
        if alter >= min_alter_tage and info.st_size >= min_groesse:
            treffer.append((pfad, info.st_size, alter))
    return treffer

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    baue_testbaum(basis)

    treffer = finde(basis, min_alter_tage=30, min_groesse=5000)
    treffer.sort(key=lambda t: t[1], reverse=True)
    for pfad, groesse, alter in treffer:
        print(f"{str(pfad.relative_to(basis)):20} {groesse:>7} Byte  {alter:>4.0f} Tage")
    print("Summe:", sum(t[1] for t in treffer), "Byte in", len(treffer), "Dateien")
```

```text
backup/db.sql          20000 Byte    45 Tage
backup/db_alt.sql      15000 Byte   400 Tage
logs/app.log.1          8000 Byte    40 Tage
Summe: 43000 Byte in 3 Dateien
```

Die Funktion `baue_testbaum` dient nur der Vorbereitung. Spannend ist `finde`: Sie durchläuft mit `rglob("*")` alle Einträge, überspringt Verzeichnisse mit `continue`, liest mit `stat()` Größe und Zeit, berechnet das Alter in Tagen und sammelt die Treffer als Tupel. Die Bedingung verknüpft Alter und Größe mit `and`. Danach sortiert `treffer.sort(key=lambda t: t[1], reverse=True)` die Liste absteigend nach Größe, und die Ausgabe formatiert jede Zeile in festen Spalten. Man sieht, dass `app.log` (2 Tage alt) und `app.log.2` (3000 Byte, zu klein) nicht passen und `klein.tmp` wegen der Größe herausfällt.

Mit `datetime` lässt sich der Zeitstempel in ein lesbares Datum umrechnen. Das Modul `datetime` behandelt Modul 8 ausführlich; für den Moment genügt der Hinweis, dass `datetime.fromtimestamp(st_mtime)` aus der Unix-Zeit ein Datumsobjekt macht, das man mit `strftime` formatiert.

```python
from datetime import datetime, timezone

unix_zeit = 1_700_000_000
datum = datetime.fromtimestamp(unix_zeit, tz=timezone.utc)
print(datum)
print(datum.strftime("%Y-%m-%d %H:%M"))
print(datum.isoformat())
```

```text
2023-11-14 22:13:20+00:00
2023-11-14 22:13
2023-11-14T22:13:20+00:00
```

Wir verwenden hier ausdrücklich die Zeitzone UTC. Ohne Angabe verwendet Python die Ortszeit des Systems, was bei Servern in verschiedenen Zeitzonen zu Verwirrung führt.

Bei Dateisuchen gibt es einige Fallstricke. Der erste sind Berechtigungen: Auf Verzeichnisse oder Dateien, für die der Benutzer keine Rechte hat, löst der Zugriff einen `PermissionError` aus, und ein einzelnes unzugängliches Verzeichnis darf nicht das ganze Skript beenden. Der zweite sind symbolische Verweise: `stat()` folgt ihnen, sodass ein Verweis auf eine große Datei wie diese selbst aussieht und doppelt gezählt wird; `lstat()` untersucht dagegen den Verweis selbst. Der dritte sind Dateien, die während der Suche verschwinden: Zwischen dem Auflisten und dem `stat()`-Aufruf kann eine Datei gelöscht werden, was einen `FileNotFoundError` auslöst. Die robuste Variante fängt diese Fehler ab und zählt sie mit.

```python
import os
import tempfile
import time
from pathlib import Path

def finde_robust(basis, min_alter_tage):
    jetzt = time.time()
    treffer = []
    uebersprungen = 0
    for pfad in basis.rglob("*"):
        try:
            if pfad.is_symlink() or not pfad.is_file():
                continue
            info = pfad.stat()
        except (PermissionError, FileNotFoundError):
            uebersprungen += 1
            continue
        if (jetzt - info.st_mtime) / 86400 >= min_alter_tage:
            treffer.append(pfad)
    return treffer, uebersprungen

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    (basis / "alt.log").write_text("x")
    zeit = time.time() - 10 * 86400
    os.utime(basis / "alt.log", (zeit, zeit))
    (basis / "neu.log").write_text("x")
    (basis / "verweis.log").symlink_to(basis / "alt.log")

    treffer, uebersprungen = finde_robust(basis, 7)
    print([p.name for p in treffer], uebersprungen)
```

```text
['alt.log'] 0
```

Der symbolische Verweis `verweis.log` wird ausgelassen, obwohl er auf eine alte Datei zeigt. Ob man Verweise folgen soll, ist eine Entscheidung je nach Aufgabe. Beim Aufräumen ist es meist sicherer, sie zu überspringen, damit man nicht versehentlich Dateien außerhalb des Verzeichnisses trifft.

> **Merke**
>
> Das Alter einer Datei ist `(time.time() - stat().st_mtime) / 86400` in Tagen. Zeitstempel sind Unix-Zeit in UTC; geben Sie bei `datetime` die Zeitzone an. Robuste Suchen fangen `PermissionError` und `FileNotFoundError` ab, überspringen symbolische Verweise (`is_symlink()`) und trennen das Suchen vom Handeln.

> **Praxisbeispiel**
>
> „Logs älter als 30 Tage komprimieren oder löschen“, „die zehn größten Dateien auf dieser Platte finden“, „Dateien ohne Zugriff seit einem Jahr archivieren“: Das sind tägliche Aufräumaufgaben. Das Prinzip, erst alle Treffer zu *sammeln* und auszugeben und erst danach (mit Bestätigung) zu *handeln*, macht ein Skript sicher und nachvollziehbar. Es entspricht der Arbeitsweise von `find` in der Shell, wo man auch erst ohne `-delete` prüft.

## 4.5 Übung abschließen + Quiz

Der Meilenstein dieses Moduls bringt alles zusammen: eine *Dateisuche nach Alter und Größe*, die ihre Treffer nach Größe sortiert in einer Tabelle ausgibt, einen Bericht in eine Datei schreibt und die gefundenen Dateien nach Bestätigung in ein Archivverzeichnis verschiebt, ohne sie zu löschen. Ein Probelauf ist der Standard.

Der Entwurf folgt den Grundsätzen aus diesem Modul. Die Suche (`finde_dateien`) kennt nur Pfade und gibt Daten zurück. Die Ausgabe (`druck_bericht`) macht daraus Text. Das Verschieben (`archiviere`) ist eine getrennte Funktion mit einem Schalter `wirklich`, der standardmäßig falsch ist. Die Hauptfunktion verbindet die Teile. Damit lässt sich jede Funktion einzeln prüfen und später in ein Skript mit Kommandozeilenargumenten verwandeln (Modul 5).

```python
import os
import shutil
import tempfile
import time
from pathlib import Path

TAG = 86400

def format_bytes(anzahl):
    wert = float(anzahl)
    for einheit in ["B", "KiB", "MiB", "GiB", "TiB"]:
        if wert < 1024 or einheit == "TiB":
            break
        wert /= 1024
    return f"{int(wert)} B" if einheit == "B" else f"{wert:.1f} {einheit}"

def finde_dateien(basis, min_alter_tage, min_groesse, ausnahmen=()):
    """Sucht Dateien unter basis; gibt Liste von Dictionaries zurück."""
    jetzt = time.time()
    treffer = []
    for pfad in basis.rglob("*"):
        try:
            if pfad.is_symlink() or not pfad.is_file():
                continue
            if any(teil in ausnahmen for teil in pfad.relative_to(basis).parts):
                continue
            info = pfad.stat()
        except OSError:
            continue
        alter = (jetzt - info.st_mtime) / TAG
        if alter >= min_alter_tage and info.st_size >= min_groesse:
            treffer.append({"pfad": pfad, "groesse": info.st_size, "alter": alter})
    treffer.sort(key=lambda t: t["groesse"], reverse=True)
    return treffer

def druck_bericht(treffer, basis):
    zeilen = [f"{'Datei':24} {'Größe':>10} {'Alter':>6}"]
    for t in treffer:
        name = str(t["pfad"].relative_to(basis))
        zeilen.append(f"{name:24} {format_bytes(t['groesse']):>10} {t['alter']:>4.0f} d")
    summe = sum(t["groesse"] for t in treffer)
    zeilen.append(f"{len(treffer)} Dateien, zusammen {format_bytes(summe)}")
    return "\n".join(zeilen)

def archiviere(treffer, basis, archiv, wirklich=False):
    """Verschiebt die Treffer nach archiv; ohne wirklich=True nur Probelauf."""
    for t in treffer:
        ziel = archiv / t["pfad"].relative_to(basis)
        if not wirklich:
            print(f"DRY-RUN: {t['pfad'].name} -> {ziel.relative_to(basis)}")
            continue
        ziel.parent.mkdir(parents=True, exist_ok=True)
        shutil.move(str(t["pfad"]), str(ziel))
        print(f"verschoben: {t['pfad'].name}")

def baue_testbaum(basis):
    for name, groesse, alter in [("logs/app.log", 5000, 2), ("logs/app.log.1", 8000, 40),
                                 ("backup/db.sql", 20000, 45), ("backup/db_alt.sql", 15000, 400),
                                 ("tmp/klein.tmp", 100, 100)]:
        pfad = basis / name
        pfad.parent.mkdir(parents=True, exist_ok=True)
        pfad.write_bytes(b"x" * groesse)
        zeit = time.time() - alter * TAG
        os.utime(pfad, (zeit, zeit))

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp) / "daten"
    basis.mkdir()
    baue_testbaum(basis)
    archiv = basis / "_archiv"

    treffer = finde_dateien(basis, min_alter_tage=30, min_groesse=1000, ausnahmen=("_archiv",))
    bericht = druck_bericht(treffer, basis)
    print(bericht)
    (Path(tmp) / "bericht.txt").write_text(bericht + "\n", encoding="utf-8")

    print()
    archiviere(treffer, basis, archiv)                 # Probelauf
    print()
    archiviere(treffer, basis, archiv, wirklich=True)  # echt
    print()
    nachher = finde_dateien(basis, 30, 1000, ausnahmen=("_archiv",))
    print("Danach noch passend:", len(nachher))
    print(sorted(str(p.relative_to(basis)) for p in archiv.rglob("*") if p.is_file()))
```

```text
Datei                         Größe  Alter
backup/db.sql              19.5 KiB   45 d
backup/db_alt.sql          14.6 KiB  400 d
logs/app.log.1              7.8 KiB   40 d
3 Dateien, zusammen 42.0 KiB

DRY-RUN: db.sql -> _archiv/backup/db.sql
DRY-RUN: db_alt.sql -> _archiv/backup/db_alt.sql
DRY-RUN: app.log.1 -> _archiv/logs/app.log.1

verschoben: db.sql
verschoben: db_alt.sql
verschoben: app.log.1

Danach noch passend: 0
['_archiv/backup/db.sql', '_archiv/backup/db_alt.sql', '_archiv/logs/app.log.1']
```

Die Struktur ist die Grundform fast aller Admin-Skripte: Suchen, Berichten, Handeln (mit Sicherung). In `finde_dateien` nimmt ein Parameter `ausnahmen` Verzeichnisnamen entgegen, die übersprungen werden; so wird das Archiv bei einem zweiten Lauf nicht erneut durchsucht. Die Ausnahmeprüfung `any(teil in ausnahmen for teil in ...parts)` ist ein Beispiel für einen Generatorausdruck, der in `any()` jeden Teil des relativen Pfads prüft. Die Suchfunktion gibt Dictionaries zurück, `druck_bericht` formatiert sie, und `archiviere` behält die Verzeichnisstruktur im Archiv bei, indem sie den relativen Pfad der Datei an das Archiv hängt. Der Bericht wird zusätzlich in eine Datei geschrieben, damit man ihn später nachlesen kann.

Für die Wiederholung gibt es zwei Hinweise. Erstens steckt in diesem Skript fast jeder Inhalt dieses Moduls: `Path`, `rglob`, `stat`, `shutil.move`, `write_text`, `with` und die Schutzmaßnahmen gegen unbeabsichtigtes Löschen. Zweitens ist die Reihenfolge entscheidend: erst suchen, dann zeigen, dann (mit Bestätigung) handeln. Das Quiz zum Modul sind die Übungsaufgaben 1 bis 4; die Aufgaben 5 bis 8 verlangen eigene Programme.

> **Praxisbeispiel**
>
> Aus diesem Gerüst werden in Modul 5 ein echtes Kommandozeilenwerkzeug mit `argparse` (`--tage 30 --groesse 1MiB --apply`), in Modul 6 ein Backup-Skript mit Protokollierung und Rückgabecodes und in Modul 9 ein Cronjob, der jede Nacht Logverzeichnisse aufräumt. Wer den Aufbau Suchen, Berichten, Handeln verinnerlicht hat, erkennt ihn in fast jeder Aufräum- und Sicherungsaufgabe wieder.

## Übungsaufgaben zu Modul 04

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Arbeiten Sie bei den Programmieraufgaben in einem temporären Verzeichnis.

**Aufgabe 1.** Geben Sie für `Path("/var/log/nginx/access.log.1.gz")` die Werte von `name`, `stem`, `suffix`, `suffixes` und `parent` an. Was ergibt `Path("/var/log") / "/etc/hosts"`, und warum kann das gefährlich sein?

**Aufgabe 2.** Welche Dateimodi sind für folgende Aufgaben richtig: (a) Inhalt lesen, (b) eine Datei neu schreiben, (c) an eine Logdatei anhängen, (d) eine neue Datei anlegen, aber nie eine vorhandene überschreiben, (e) ein Bild kopieren? Warum benutzt man `with` und die Angabe `encoding="utf-8"`?

**Aufgabe 3.** Beschreiben Sie, was `os.walk()` liefert und in welcher Reihenfolge. Worin unterscheiden sich `shutil.copy()` und `shutil.copy2()`? Nennen Sie drei Schutzmaßnahmen vor dem Löschen eines Verzeichnisbaums.

**Aufgabe 4.** Finden Sie fünf Probleme im folgenden Code und erklären Sie jedes.

```python
import os

pfad = "C:\neu\logs" + "/" + datei
f = open(pfad, "w")
inhalt = f.read()
for zeile in f:
    if zeile == "ok":
        print("gut")
os.remove(verzeichnis + "*")
```

**Aufgabe 5.** Schreiben Sie ein Programm, das eine Liste von Hostnamen in eine Datei `hosts.txt` schreibt (eine pro Zeile), die Datei wieder liest, die Zeilen ohne Zeilenumbruch in einer Liste speichert und die Anzahl sowie den alphabetisch letzten Hostnamen ausgibt.

**Aufgabe 6.** Schreiben Sie ein Programm, das alle Dateien unterhalb eines Verzeichnisses nach Endung zählt (`.log`, `.txt`, `.conf`, ohne Endung) und die Tabelle absteigend nach Anzahl ausgibt. Verwenden Sie `rglob` und `Counter`.

**Aufgabe 7.** Schreiben Sie eine Funktion `groesste_dateien(basis, n)`, die die `n` größten Dateien unterhalb von `basis` mit Namen und lesbarer Größe zurückgibt. Testen Sie sie mit einem selbst angelegten Verzeichnisbaum.

**Aufgabe 8.** Schreiben Sie ein Sicherungsprogramm: Es kopiert alle Dateien, die in den letzten 7 Tagen geändert wurden, mit `shutil.copy2` in ein Verzeichnis `sicherung/` und behält die Unterverzeichnisse bei. Es arbeitet standardmäßig im Probelauf und kopiert nur mit einem Schalter `wirklich=True`.

## Lösungen zu Modul 04

**Aufgabe 1.**

```python
from pathlib import Path

p = Path("/var/log/nginx/access.log.1.gz")
print(p.name, p.stem, p.suffix, p.suffixes, p.parent)
print(Path("/var/log") / "/etc/hosts")
```

```text
access.log.1.gz access.log.1 .gz ['.log', '.1', '.gz'] /var/log/nginx
/etc/hosts
```

Ein absoluter Pfad rechts von `/` ersetzt den linken Teil vollständig. Stammt der rechte Teil aus einer Eingabe, kann ein Angreifer damit beliebige Dateien ansprechen. Abhilfe: das Ergebnis mit `resolve()` prüfen und sicherstellen, dass es unterhalb des erlaubten Verzeichnisses liegt.

**Aufgabe 2.** (a) `"r"`. (b) `"w"`, was eine vorhandene Datei leert. (c) `"a"`. (d) `"x"`, es löst `FileExistsError` aus, wenn die Datei existiert. (e) Binärmodus `"rb"` und `"wb"` (oder `shutil.copy`), weil ein Bild kein Text ist. Die `with`-Anweisung schließt die Datei garantiert, auch bei Fehlern. Die Kodierung muss ausdrücklich angegeben werden, weil die Voreinstellung vom System abhängt und sonst Umlaute auf einem anderen Rechner falsch gelesen werden könnten.

**Aufgabe 3.** `os.walk()` liefert für jedes Verzeichnis im Baum ein Tupel (Pfad des Verzeichnisses, Liste der Unterverzeichnisnamen, Liste der Dateinamen). `copy()` kopiert Inhalt und Rechte, `copy2()` zusätzlich Zeitstempel und weitere Metadaten (wichtig für Sicherungen). Schutzmaßnahmen: (1) Probelauf, bei dem nur ausgegeben wird, (2) Pfadprüfung (nicht `/`, nicht das Home-Verzeichnis, unterhalb einer erlaubten Basis, mit `resolve()`), (3) erst in ein Archivverzeichnis verschieben und später endgültig löschen.

**Aufgabe 4.** (1) `"C:\neu\logs"` enthält die Fluchtfolgen `\n` und `\l`; richtig wäre ein Raw-String oder besser `Path`. (2) Strings werden mit `"/"` zusammengebaut; besser `Path(...) / datei`. (3) Die Datei wird ohne `with` geöffnet und nie geschlossen, und ohne `encoding`. (4) Sie wird im Modus `"w"` geöffnet, was sie leert, und danach gelesen; der Modus `"w"` erlaubt kein Lesen (`io.UnsupportedOperation`). (5) `zeile == "ok"` scheitert, weil jede Zeile den Zeilenumbruch enthält (`strip()`). Zusätzlich löscht `os.remove(verzeichnis + "*")` nichts Sinnvolles: `os.remove` kennt keine Platzhalter und löscht keine Verzeichnisse; für Muster nutzt man `glob`, für Bäume `shutil.rmtree` mit Schutzprüfung.

**Aufgabe 5.**

```python
import tempfile
from pathlib import Path

hosts = ["web01", "db01", "srv01", "gw01"]

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "hosts.txt"
    with open(datei, "w", encoding="utf-8") as f:
        for h in hosts:
            f.write(h + "\n")

    with open(datei, encoding="utf-8") as f:
        zeilen = [zeile.rstrip("\n") for zeile in f]

    print(len(zeilen), sorted(zeilen)[-1])
```

```text
4 web01
```

Die List Comprehension liest die Datei zeilenweise und entfernt dabei jeweils den Umbruch. `sorted(zeilen)[-1]` ist der alphabetisch letzte Name; `max(zeilen)` würde dasselbe liefern.

**Aufgabe 6.**

```python
import tempfile
from collections import Counter
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    for name in ["a.log", "b.log", "c.log", "x/d.txt", "x/e.conf", "x/y/f.log", "README"]:
        p = basis / name
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text("x")

    zaehler = Counter(p.suffix or "(keine)" for p in basis.rglob("*") if p.is_file())
    for endung, anzahl in sorted(zaehler.items(), key=lambda t: (-t[1], t[0])):
        print(f"{endung:10} {anzahl}")
```

```text
.log       4
(keine)    1
.conf      1
.txt       1
```

`suffix` ist für Dateien ohne Endung ein leerer String, der als falsch gilt; der Ausdruck `p.suffix or "(keine)"` liefert dann den Ersatztext. Die Reihenfolge, in der `rglob` die Dateien liefert, hängt vom Dateisystem ab. Damit der Bericht immer gleich aussieht, sortiert die Schleife nach Anzahl absteigend und bei Gleichstand nach dem Namen; der Schlüssel `(-anzahl, name)` ist dasselbe Muster wie beim Log-Analyzer in Modul 3.

**Aufgabe 7.**

```python
import tempfile
from pathlib import Path

def format_bytes(anzahl):
    wert = float(anzahl)
    for einheit in ["B", "KiB", "MiB", "GiB"]:
        if wert < 1024 or einheit == "GiB":
            break
        wert /= 1024
    return f"{int(wert)} B" if einheit == "B" else f"{wert:.1f} {einheit}"

def groesste_dateien(basis, n):
    dateien = [(p, p.stat().st_size) for p in Path(basis).rglob("*") if p.is_file()]
    dateien.sort(key=lambda t: t[1], reverse=True)
    return [(p.name, format_bytes(g)) for p, g in dateien[:n]]

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp)
    for name, groesse in [("a.bin", 500), ("b.bin", 5000), ("sub/c.bin", 2 * 1024 * 1024), ("d.bin", 70)]:
        p = basis / name
        p.parent.mkdir(exist_ok=True)
        p.write_bytes(b"0" * groesse)
    for name, groesse in groesste_dateien(basis, 3):
        print(f"{name:8} {groesse:>10}")
```

```text
c.bin       2.0 MiB
b.bin       4.9 KiB
a.bin         500 B
```

Die Funktion gibt eine Liste von Tupeln zurück, statt auszugeben, und lässt sich damit leicht testen. Der Ausschnitt `dateien[:n]` liefert die ersten `n` Einträge der absteigend sortierten Liste; ist die Liste kürzer, gibt er einfach alle zurück, ohne Fehler.

**Aufgabe 8.**

```python
import os
import shutil
import tempfile
import time
from pathlib import Path

def sichere_neue_dateien(basis, ziel, tage=7, wirklich=False):
    grenze = time.time() - tage * 86400
    anzahl = 0
    for pfad in sorted(basis.rglob("*")):
        if not pfad.is_file() or ziel in pfad.parents:
            continue
        if pfad.stat().st_mtime < grenze:
            continue
        ziel_pfad = ziel / pfad.relative_to(basis)
        anzahl += 1
        if wirklich:
            ziel_pfad.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(pfad, ziel_pfad)
        else:
            print("DRY-RUN:", pfad.relative_to(basis))
    return anzahl

with tempfile.TemporaryDirectory() as tmp:
    basis = Path(tmp) / "daten"
    ziel = basis / "sicherung"
    for name, alter in [("a.txt", 1), ("x/b.txt", 3), ("x/alt.txt", 30)]:
        p = basis / name
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(name)
        zeit = time.time() - alter * 86400
        os.utime(p, (zeit, zeit))

    n = sichere_neue_dateien(basis, ziel)
    print(n, "Dateien im Probelauf, kopiert:", ziel.exists())
    n = sichere_neue_dateien(basis, ziel, wirklich=True)
    print(n, "Dateien kopiert:", sorted(str(p.relative_to(ziel)) for p in ziel.rglob("*") if p.is_file()))
```

```text
DRY-RUN: a.txt
DRY-RUN: x/b.txt
2 Dateien im Probelauf, kopiert: False
2 Dateien kopiert: ['a.txt', 'x/b.txt']
```

Die Funktion überspringt Dateien, die älter als sieben Tage sind (`alt.txt`) und die Sicherung selbst (`ziel in pfad.parents`), sonst würde ein zweiter Lauf die Sicherung erneut sichern. Der Probelauf ist der Standard; erst `wirklich=True` kopiert. `copy2` erhält den Änderungszeitstempel der Originale. Bei `mkdir(parents=True, exist_ok=True)` entstehen die Unterverzeichnisse im Ziel, sodass die Struktur erhalten bleibt.
