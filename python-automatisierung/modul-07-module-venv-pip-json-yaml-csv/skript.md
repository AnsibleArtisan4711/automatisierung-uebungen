# Modul 07 - Module, venv-pip, JSON-YAML-CSV

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (7.1 bis 7.5), danach folgen „Übungsaufgaben zu Modul 07“ (8 Aufgaben) und „Lösungen zu Modul 07“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Do 10.12.2026 bis Mi 16.12.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul macht aus Einzelskripten wartbare Programme mit Konfiguration und Abhängigkeiten)

**Meilenstein:** Skript mit Config-Datei

**Lernziele.** Nach diesem Modul können Sie Code auf Module und Pakete verteilen und Importe korrekt einsetzen, für jedes Projekt eine eigene virtuelle Umgebung anlegen und Abhängigkeiten mit `pip` und einer `requirements.txt` verwalten, und Daten in den Formaten JSON, YAML, TOML und CSV lesen und schreiben. Am Ende steht ein Skript, das sich über eine Konfigurationsdatei steuern lässt, Einstellungen aus Datei, Umgebung und Kommandozeile in der richtigen Rangfolge zusammenführt und seine Ergebnisse im gewünschten Format ausgibt.

| **Nr.** | **Lerneinheit**                  | **Datum**     | **Dauer** | **Prio** |
|:--------|:---------------------------------|:--------------|:----------|:---------|
| 7.1     | Module & Imports                 | Do 10.12.2026 | 2 h       | 2        |
| 7.2     | venv, pip, requirements.txt      | Fr 11.12.2026 | 2 h       | 2        |
| 7.3     | JSON lesen & schreiben           | Mo 14.12.2026 | 2 h       | 2        |
| 7.4     | YAML & CSV                       | Di 15.12.2026 | 2 h       | 2        |
| 7.5     | Config-gesteuertes Skript + Quiz | Mi 16.12.2026 | 2 h       | 2        |
|         | **Summe**                        |               | **10 h**  |          |

> **Merke**
>
> In diesem Modul legen die Beispiele Module, Pakete und virtuelle Umgebungen immer in einem temporären Verzeichnis an und räumen sie wieder auf. Ein Beispiel (die Installation eines Pakets in 7.2) braucht eine Internetverbindung; es lädt ein kleines, frei verfügbares Paket aus dem Python Package Index (PyPI) herunter. Die Ausgaben der Installation hängen von der Pip-Version ab und sind entsprechend gekennzeichnet.

## 7.1 Module & Imports

Bisher bestand jedes Ihrer Programme aus einer einzigen Datei. Das ist für kleine Skripte in Ordnung, wird aber unübersichtlich, sobald dieselbe Funktion in mehreren Skripten gebraucht wird: Man kopiert sie, und bei jedem Fehler muss man jede Kopie ändern. Die Lösung heißt *Modul*. Ein Modul ist schlicht eine Python-Datei mit der Endung `.py`, deren Funktionen, Klassen und Variablen man aus anderen Dateien benutzen kann. Sie haben das längst getan: `import subprocess` oder `import json` laden Module der Standardbibliothek. Neu ist jetzt, dass Sie eigene Module schreiben und verstehen, wie Python sie findet.

Beim *Importieren* führt Python die Datei aus, legt das Ergebnis als Modulobjekt an und bindet es an einen Namen. Dafür gibt es mehrere Schreibweisen. `import modul` macht das Modul unter seinem Namen verfügbar; man greift mit `modul.funktion()` zu, und die Herkunft jeder Funktion bleibt im Code sichtbar. `from modul import funktion` bindet nur diese eine Funktion direkt an den Namen; kürzer, aber der Bezug zum Modul geht verloren. `import modul as kurz` vergibt einen Spitznamen, üblich bei langen Namen. `from modul import *` importiert alles und gilt als schlechter Stil: Man weiß nicht mehr, woher ein Name stammt, und er kann gleichnamige Namen überschreiben. Ein Modul wird pro Programm nur *einmal* ausgeführt; weitere Importe liefern dasselbe Objekt aus dem Zwischenspeicher `sys.modules`.

Mehrere Module bündelt man in einem *Paket*: einem Verzeichnis, das (üblicherweise) eine Datei `__init__.py` enthält. Diese Datei darf leer sein; ihr Inhalt wird beim Import des Pakets ausgeführt. Innerhalb eines Pakets kann ein Modul die anderen mit einem *relativen Import* erreichen: `from .groesse import lesbar` bedeutet „aus dem Modul `groesse` im selben Paket“, wobei der Punkt für das aktuelle Paket steht. Mit zwei Punkten gelangt man ins übergeordnete Paket. Das folgende Beispiel legt in einem temporären Verzeichnis ein kleines Paket `admintools` mit zwei Modulen an und benutzt es.

```python
import sys
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    paket = Path(tmp) / "admintools"
    paket.mkdir()
    (paket / "__init__.py").write_text('VERSION = "1.0"\n', encoding="utf-8")
    (paket / "groesse.py").write_text(
        "def lesbar(byte):\n"
        "    for einheit in ('B', 'KiB', 'MiB', 'GiB'):\n"
        "        if byte < 1024 or einheit == 'GiB':\n"
        "            return f'{byte:.1f} {einheit}'\n"
        "        byte /= 1024\n", encoding="utf-8")
    (paket / "pfade.py").write_text(
        "from .groesse import lesbar\n"
        "def beschreibe(name, byte):\n"
        "    return f'{name}: {lesbar(byte)}'\n", encoding="utf-8")

    sys.path.insert(0, tmp)               # Python soll hier nach Modulen suchen
    import admintools
    from admintools import groesse
    from admintools.pfade import beschreibe

    print(admintools.VERSION)
    print(groesse.lesbar(1536))
    print(beschreibe("log", 3 * 1024 ** 2))
    print(admintools.__name__, groesse.__name__)
    print("admintools.groesse" in sys.modules, Path(groesse.__file__).name)
    sys.path.remove(tmp)
```

```text
1.0
1.5 KiB
log: 3.0 MiB
admintools admintools.groesse
True groesse.py
```

Das Programm schreibt die Dateien des Pakets mit Python selbst, damit das Beispiel in einem Stück läuft; in der Praxis legen Sie diese Dateien im Editor an. Wichtig ist die Zeile `sys.path.insert(0, tmp)`, die uns zur nächsten Frage führt: Wo sucht Python überhaupt nach Modulen? Antwort: in der Liste `sys.path`, von vorn nach hinten, und der erste Treffer gewinnt. Diese Liste enthält, in dieser Reihenfolge, das Verzeichnis des gestarteten Skripts (bei interaktiver Nutzung das aktuelle Verzeichnis), die Verzeichnisse aus der Umgebungsvariable `PYTHONPATH`, die Standardbibliothek und das Verzeichnis `site-packages`, in dem installierte Fremdpakete liegen (Lerneinheit 7.2). Im Beispiel stellen wir das temporäre Verzeichnis an den Anfang und entfernen es am Ende wieder. Jedes Modul hat zudem einen Namen: Das Paket heißt `admintools`, das Untermodul `admintools.groesse`, und das Attribut `__file__` verrät die Datei, aus der es geladen wurde.

Eine besondere Rolle spielt der Name `__name__`. Wird eine Datei als Programm gestartet (`python3 datei.py`), ist `__name__` gleich `"__main__"`; wird sie importiert, ist er der Modulname. Damit kann eine Datei beides sein: ein importierbares Modul und ein startbares Programm. Der Code unter `if __name__ == "__main__":` läuft dann nur beim direkten Start. Sie haben diese Zeile in Modul 5 im Gerüst `sys.exit(main())` schon benutzt; jetzt wissen Sie, warum sie dort steht.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

modul = (
    'def hallo():\n'
    '    return "Hallo aus dem Modul"\n'
    '\n'
    'print("__name__ ist", __name__)\n'
    'if __name__ == "__main__":\n'
    '    print(hallo())\n'
)

with tempfile.TemporaryDirectory() as tmp:
    (Path(tmp) / "gruss.py").write_text(modul, encoding="utf-8")

    print("--- als Programm gestartet ---")
    r = subprocess.run([sys.executable, "gruss.py"], cwd=tmp, capture_output=True, text=True)
    print(r.stdout, end="")

    print("--- importiert ---")
    r = subprocess.run([sys.executable, "-c", "import gruss; print(gruss.hallo())"],
                       cwd=tmp, capture_output=True, text=True)
    print(r.stdout, end="")
```

```text
--- als Programm gestartet ---
__name__ ist __main__
Hallo aus dem Modul
--- importiert ---
__name__ ist gruss
Hallo aus dem Modul
```

Im ersten Lauf ist `__name__` gleich `__main__`, und der Block unter der `if`-Abfrage gibt den Gruß aus. Im zweiten Lauf wird die Datei nur importiert; die Zeile mit dem `print` läuft trotzdem (sie steht ja auf der obersten Ebene), der `if`-Block aber nicht. Der Gruß in der letzten Zeile stammt aus dem Aufruf `gruss.hallo()` im Befehl.

Zu den Modulen gehört auch ein Überblick über das, was schon mitgeliefert wird. Die Standardbibliothek umfasst hunderte Module; `sys.stdlib_module_names` (seit Python 3.10) listet ihre Namen als Menge. Alles andere ist entweder Ihr eigener Code oder ein *Fremdpaket*, das erst installiert werden muss. Als Konvention (PEP 8) stehen Importe am Dateianfang, in drei Gruppen mit je einer Leerzeile dazwischen: erst die Standardbibliothek, dann Fremdpakete, dann eigene Module. So sieht man auf einen Blick, wovon ein Skript abhängt. Manchmal muss man ein Modul erst zur Laufzeit laden, etwa bei Erweiterungen (Plugins), deren Namen in einer Konfiguration stehen. Dafür dient `importlib.import_module(name)`.

```python
import importlib
import sys

print("json" in sys.stdlib_module_names, "requests" in sys.stdlib_module_names)

# Plugin-Muster: Der Modulname kommt aus einer Konfiguration
konfiguration = {"ausgabe": "json", "formatierer": "pprint"}
for rolle, modulname in konfiguration.items():
    modul = importlib.import_module(modulname)
    print(rolle, "->", modul.__name__)

try:
    importlib.import_module("gibt_es_nicht")
except ModuleNotFoundError as fehler:
    print("Fehler:", fehler)
```

```text
True False
ausgabe -> json
formatierer -> pprint
Fehler: No module named 'gibt_es_nicht'
```

Die erste Zeile zeigt, dass `json` zur Standardbibliothek gehört und `requests` nicht (es muss mit `pip` installiert werden). Fehlt ein Modul, löst Python `ModuleNotFoundError` aus, eine Unterklasse von `ImportError`. Das ist eine häufige Fehlermeldung: Entweder ist das Paket nicht installiert, oder man ist in der falschen virtuellen Umgebung (siehe nächste Lerneinheit), oder der Name ist falsch geschrieben.

Mit Modulen sind einige typische Fallen verbunden. Die häufigste: *Der eigene Dateiname überdeckt ein Standardmodul.* Wer eine Datei `json.py`, `random.py` oder `test.py` nennt, ersetzt für alle Skripte im selben Verzeichnis das gleichnamige Standardmodul, denn das Skriptverzeichnis steht in `sys.path` ganz vorn. Die Folgen sind verwirrende Fehler („module ’json’ has no attribute ’dumps’“). Eine zweite Falle ist der *zirkuläre Import*: Modul `a` importiert `b`, und `b` importiert `a`; beim zweiten Import ist `a` noch nicht fertig ausgeführt, und die gesuchten Namen fehlen. Die Abhilfe ist, die gemeinsame Funktionalität in ein drittes Modul auszulagern oder den Import in die Funktion zu verlegen, die ihn braucht.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

def lauf(code, verzeichnis):
    return subprocess.run([sys.executable, "-c", code], cwd=verzeichnis,
                          capture_output=True, text=True)

with tempfile.TemporaryDirectory() as tmp:
    # FALSCH: eine eigene Datei namens json.py überdeckt die Standardbibliothek
    (Path(tmp) / "json.py").write_text("# meine Hilfsfunktionen\n", encoding="utf-8")
    r = lauf("import json; print(json.dumps({'a': 1}))", tmp)
    print(r.stderr.strip().splitlines()[-1].split(" (")[0])
    (Path(tmp) / "json.py").unlink()

    # RICHTIG: ein eindeutiger Name, die Standardbibliothek bleibt erreichbar
    (Path(tmp) / "meine_json_helfer.py").write_text("# Hilfsfunktionen\n", encoding="utf-8")
    r = lauf("import json; print(json.dumps({'a': 1}))", tmp)
    print(r.stdout.strip())

    # FALSCH: zirkulärer Import
    (Path(tmp) / "a.py").write_text("import b\ndef fa():\n    return 'a'\n", encoding="utf-8")
    (Path(tmp) / "b.py").write_text("from a import fa\ndef fb():\n    return 'b'\n", encoding="utf-8")
    r = lauf("import a", tmp)
    print(r.stderr.strip().splitlines()[-1].split(" (")[0])
```

```text
AttributeError: module 'json' has no attribute 'dumps'
{"a": 1}
ImportError: cannot import name 'fa' from 'a'
```

Im ersten Fall steht `json.py` im Arbeitsverzeichnis und wird statt des Standardmoduls geladen; die Fehlermeldung nennt die Datei in Klammern (hier abgeschnitten). Seit neueren Python-Versionen weisen die Meldungen sogar darauf hin, dass der Dateiname mit einem Bibliotheksmodul übereinstimmen könnte. Im zweiten Fall ist alles in Ordnung. Im dritten Fall scheitert der Import, weil `a` beim Laden von `b` noch nicht bis zur Definition von `fa` gekommen ist.

> **Ergänzung aus der Nachrecherche**
>
> **Modulsuche und Standardbibliothek (Python 3.13 geprüft).** Die Menge `sys.stdlib_module_names` existiert seit Python 3.10; Ihr Lehrplan setzt Python 3.11 oder neuer voraus, sie steht also zur Verfügung. Die Reihenfolge der Suche in `sys.path` und das Zwischenspeichern in `sys.modules` wurden in diesem Skript durch Ausführen bestätigt, aber nicht in der Dokumentation des Importsystems nachgelesen. Bei der Fehlermeldung zu überdeckten Modulen kann der Wortlaut je nach Python-Version abweichen; wir haben deshalb in den Beispielen nur den stabilen Anfang der Meldung ausgegeben.

> **Merke**
>
> Ein Modul ist eine `.py`-Datei, ein Paket ein Verzeichnis mit Modulen. Python sucht Module in `sys.path`, der erste Treffer gewinnt; die Datei `if __name__ == "__main__":` trennt Programm und Modul. Benennen Sie eigene Dateien nie wie Standardmodule, importieren Sie keine Sternchen, und vermeiden Sie zirkuläre Importe durch eine saubere Aufteilung.

> **Praxisbeispiel**
>
> Sobald Ihr Backup-Skript aus Modul 6 mit dem `sysw`-Wrapper aus Modul 5 zusammenarbeiten soll, lagern Sie die gemeinsamen Teile (Fehlerklassen, Logging-Einrichtung, Größenformatierung) in ein eigenes Modul aus, das beide Skripte importieren. Genau diese Aufteilung bildet das Gerüst jedes größeren Projekts. In Modul 13 wird daraus eine ordentliche Projektstruktur mit `pyproject.toml`, die sich wie ein eigenes Programm installieren lässt.

## 7.2 venv, pip, requirements.txt

Die Standardbibliothek deckt viel ab, aber nicht alles. Für HTTP-Abfragen (Modul 10), Systemüberwachung (Modul 11) oder YAML-Dateien (Lerneinheit 7.4) verwendet man Pakete von Drittanbietern. Sie liegen im *Python Package Index* (PyPI), einem öffentlichen Verzeichnis, und werden mit dem Programm `pip` installiert. Das wirft eine Frage auf, die Anfänger oft übersehen: *Wohin* wird installiert? Wenn alles in dieselbe Python-Installation wandert, kommen sich Projekte in die Quere. Projekt A braucht Version 1 einer Bibliothek, Projekt B Version 2; beide gleichzeitig geht nicht. Auf einem Linux-System kommt ein zweites Problem hinzu: Das System selbst benutzt Python für eigene Werkzeuge, und ein `pip install`, das Systempakete überschreibt, kann sie beschädigen.

Die Lösung ist die *virtuelle Umgebung* (englisch *virtual environment*, kurz *venv*). Eine venv ist ein Verzeichnis mit einer eigenen, schlanken Python-Umgebung: ein Verweis auf den Interpreter, ein eigenes Verzeichnis für installierte Pakete und eine eigene Kopie von `pip`. Pakete, die Sie darin installieren, sind nur für diese Umgebung sichtbar. Jedes Projekt bekommt seine eigene, und die Systeminstallation bleibt unberührt. Moderne Linux-Distributionen erzwingen das inzwischen: Ubuntu 24.04 und neuer verweigern `pip install` in der System-Python-Umgebung mit dem Fehler `externally-managed-environment`. Die Hintergründe stehen in der Ergänzung weiter unten.

Eine Umgebung wird mit dem Standardmodul `venv` angelegt: `python3 -m venv .venv`. Das erzeugt das Verzeichnis `.venv` (der Punkt am Anfang macht es zu einer versteckten Datei und ist eine Konvention). Auf Ubuntu und Debian braucht man dafür das Paket `python3-venv`, das Sie mit `sudo apt install python3-venv` nachinstallieren, falls die Meldung kommt, dass `ensurepip` fehlt. Die Python-Dokumentation beschreibt den Inhalt: Es gibt eine Datei `pyvenv.cfg` mit einem Schlüssel `home`, der auf die Python-Installation zeigt, aus der die Umgebung entstand, ein Verzeichnis `bin` (unter Windows `Scripts`) mit einer Kopie oder einem Verweis auf den Interpreter und ein Verzeichnis `lib/pythonX.Y/site-packages` für die installierten Pakete. Das Beispiel legt eine Umgebung in einem temporären Verzeichnis an und untersucht sie. Die Zeile `sys.prefix != sys.base_prefix` ist die empfohlene Methode, um in Python-Code zu erkennen, ob man in einer venv läuft; die Dokumentation warnt ausdrücklich, dass man sich dafür nicht auf die Umgebungsvariable `VIRTUAL_ENV` verlassen kann.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

print("Hier (System-Python): venv aktiv?", sys.prefix != sys.base_prefix)

with tempfile.TemporaryDirectory() as tmp:
    umgebung = Path(tmp) / ".venv"
    subprocess.run([sys.executable, "-m", "venv", str(umgebung)], check=True)

    print(sorted(p.name for p in umgebung.iterdir() if p.name in ("bin", "lib", "pyvenv.cfg")))

    einstellungen = dict(zeile.split(" = ", 1)
                         for zeile in (umgebung / "pyvenv.cfg").read_text().splitlines())
    print("home vorhanden:", "home" in einstellungen)
    print("Systempakete sichtbar:", einstellungen["include-system-site-packages"])

    python_der_venv = umgebung / "bin" / "python"
    r = subprocess.run([str(python_der_venv), "-c",
                        "import sys; print(sys.prefix != sys.base_prefix)"],
                       capture_output=True, text=True, check=True)
    print("In der venv: venv aktiv?", r.stdout.strip())

    r = subprocess.run([str(python_der_venv), "-m", "pip", "list", "--format=freeze"],
                       capture_output=True, text=True, check=True)
    print("Vorinstalliert:", [zeile.split("==")[0] for zeile in r.stdout.split()])
```

```text
Hier (System-Python): venv aktiv? False
['bin', 'lib', 'pyvenv.cfg']
home vorhanden: True
Systempakete sichtbar: false
In der venv: venv aktiv? True
Vorinstalliert: ['pip']
```

Das Beispiel zeigt mehrere Dinge. Die Umgebung enthält `bin`, `lib` und `pyvenv.cfg`; in Python 3.13 legt `venv` außerdem eine `.gitignore`-Datei an (die Dokumentation vermerkt das seit Version 3.13), die uns hier nicht interessiert. Standardmäßig sind die Systempakete *nicht* sichtbar (`include-system-site-packages = false`); der Schalter `--system-site-packages` würde das ändern. Frisch angelegt enthält die Umgebung nur `pip` (bis Python 3.11 zusätzlich `setuptools`; die Dokumentation vermerkt, dass es seit 3.12 nicht mehr zu den Kernabhängigkeiten gehört). Man kann also eine Umgebung beliebig oft wegwerfen und neu aufbauen; die Dokumentation nennt venvs ausdrücklich *wegwerfbar* („disposable“).

Zum *Benutzen* einer venv gibt es zwei Wege. Der übliche ist die *Aktivierung*: `source .venv/bin/activate` stellt das `bin`-Verzeichnis der Umgebung an den Anfang des Suchpfads `PATH`, sodass `python` und `pip` die der Umgebung sind; die Eingabeaufforderung zeigt dann `(.venv)` an, und `deactivate` beendet die Aktivierung. Der zweite Weg ist, die Programme der Umgebung mit vollem Pfad aufzurufen, etwa `.venv/bin/python skript.py`. Die Dokumentation betont, dass eine Aktivierung nicht nötig ist. Für automatische Abläufe (Cron, systemd) ist dieser zweite Weg der richtige, denn dort gibt es keine interaktive Shell, in der man aktivieren könnte.

```text
$ sudo apt install python3-venv          # nur einmal, auf Debian/Ubuntu
$ cd ~/projekte/sysw
$ python3 -m venv .venv                  # Umgebung anlegen
$ source .venv/bin/activate              # aktivieren
(.venv) $ python -m pip install humanize==4.9.0
(.venv) $ python -m pip freeze > requirements.txt
(.venv) $ deactivate
$ .venv/bin/python skript.py             # ohne Aktivierung, z. B. in Cron
```

Das ist der typische Ablauf in der Shell (ein Konsolenmitschnitt, kein Python-Code, deshalb nicht automatisch geprüft). Beachten Sie die Schreibweise `python -m pip`: Sie ruft das `pip` auf, das zu *diesem* Python gehört. Ein nacktes `pip` kann dagegen zu einer anderen Installation gehören, was zu der klassischen Verwirrung führt, dass ein Paket „installiert“ ist, aber nicht gefunden wird.

Damit die Abhängigkeiten eines Projekts nicht in den Köpfen der Entwickler stecken, hält man sie in einer Textdatei fest, der `requirements.txt`. Jede Zeile nennt ein Paket und optional eine Version. Die Dokumentation von `pip` beschreibt das Format so: Jede Zeile gibt etwas an, das installiert werden soll; Beispiele für Versionsangaben sind `docopt == 0.6.1` (genau diese Version) oder `requests >= 2.8.1` (mindestens diese); Zeilen, die mit `#` beginnen, sind Kommentare, und mit `-r andere.txt` bindet man eine weitere Datei ein. Eine weitere übliche Angabe ist `~=1.4` („kompatibel mit 1.4“, also ab 1.4, aber unter 2.0); sie ist in der abgerufenen Seite nicht genannt und beruht auf Fachwissen. Der Befehl `pip freeze` gibt alle in der Umgebung installierten Pakete mit genauen Versionen aus; die Ausgabe leitet man in die Datei um, und `pip install -r requirements.txt` baut die Umgebung später (oder auf einem anderen Rechner) nach. Das folgende Beispiel führt diesen Kreislauf komplett aus: Es installiert ein kleines Hilfspaket in eine Umgebung, erzeugt eine `requirements.txt`, baut mit ihr eine *zweite* Umgebung auf und prüft, dass beide übereinstimmen. Gewählt ist das Paket `humanize`, das Zahlen lesbar formatiert; es braucht eine Internetverbindung.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

def pip(umgebung, *argumente):
    python = Path(umgebung) / "bin" / "python"
    return subprocess.run([str(python), "-m", "pip", *argumente],
                          capture_output=True, text=True, check=True).stdout

with tempfile.TemporaryDirectory() as tmp:
    erste, zweite = Path(tmp) / "erste", Path(tmp) / "zweite"
    for umgebung in (erste, zweite):
        subprocess.run([sys.executable, "-m", "venv", str(umgebung)], check=True)

    pip(erste, "install", "humanize==4.9.0")
    anforderungen = Path(tmp) / "requirements.txt"
    anforderungen.write_text(pip(erste, "freeze"), encoding="utf-8")
    print("requirements.txt:", anforderungen.read_text(encoding="utf-8").split())

    pip(zweite, "install", "-r", str(anforderungen))
    print("zweite Umgebung: ", pip(zweite, "freeze").split())

    code = "import humanize; print(humanize.naturalsize(4_700_000_000, binary=True))"
    r = subprocess.run([str(zweite / "bin" / "python"), "-c", code],
                       capture_output=True, text=True, check=True)
    print("In der venv:", r.stdout.strip())

    r = subprocess.run([sys.executable, "-c", "import humanize"],
                       capture_output=True, text=True)
    print("Im System:  ", r.stderr.strip().splitlines()[-1])
```

```text
requirements.txt: ['humanize==4.9.0']
zweite Umgebung:  ['humanize==4.9.0']
In der venv: 4.4 GiB
Im System:   ModuleNotFoundError: No module named 'humanize'
```

Die Funktion `pip` ruft `python -m pip` der jeweiligen Umgebung auf. Die erste Umgebung bekommt das Paket mit einer *festen* Version (`==4.9.0`); `pip freeze` schreibt es in die Datei. Die zweite Umgebung entsteht allein aus dieser Datei und enthält dasselbe, wie die Ausgabe zeigt. Das letzte Experiment macht die Isolation sichtbar: Das System-Python kennt das Paket nicht. Dieser Kreislauf ist der Grund, warum Sie eine venv jederzeit löschen können; die `requirements.txt` ist die Quelle der Wahrheit, die Umgebung nur ein Ergebnis.

> **Ergänzung aus der Nachrecherche**
>
> **PEP 668 und `externally-managed-environment` (abgerufen: zwei Anleitungen).** Zwei Anleitungen, die wir gelesen haben (UbuntuHandbook zu Ubuntu 24.04 und LinuxCapable zu venv unter Ubuntu), nennen dieselbe Ursache: Seit Ubuntu 23.04 gilt das System-Python als vom Betriebssystem verwaltet (PEP 668), und `pip install` in dieser Umgebung bricht mit `error: externally-managed-environment` ab; LinuxCapable nennt ausdrücklich Ubuntu 24.04 und 26.04. Dadurch soll verhindert werden, dass mit `pip` installierte Pakete mit denen des Paketmanagers (`apt`) kollidieren. Als Lösungen werden genannt: erstens das Paket aus der Distribution installieren, wenn es existiert (Pakete mit Präfix `python3-`, etwa `python3-yaml`); zweitens eine virtuelle Umgebung anlegen und dort `pip` benutzen; drittens für eigenständige Programme `pipx` (`sudo apt install pipx`, dann `pipx install programmname`); ein Zwang mit `--break-system-packages` sowie `sudo pip` werden als riskant bezeichnet, weil sie Dateien überschreiben können, die `apt` verwaltet. LinuxCapable empfiehlt außerdem, `requirements.txt` ins Git-Repository zu übernehmen und `.venv/` in die `.gitignore` einzutragen. Quellen: <https://ubuntuhandbook.org/index.php/2024/03/pip-install-error-ubuntu-2404/> und die Anleitung zu venv auf <https://www.linuxcapable.com>.
>
> **venv-Dokumentation (abgerufen: Python 3.14).** Dort steht, dass Umgebungen „disposable“ sein sollen (leicht zu löschen und neu zu erzeugen), dass man in ihnen keinen Projektcode ablegt, dass sie nicht in die Versionsverwaltung gehören und dass man sie nicht verschiebt oder kopiert, sondern am Zielort neu anlegt, weil sie „inherently non-portable“ sind; die `pyvenv.cfg` enthält den Schlüssel `home`; `--system-site-packages`, `--without-pip` und `--upgrade-deps` sind die wichtigsten Schalter. Quelle: <https://docs.python.org/3.14/library/venv.html>.
>
> **Requirements-Format (abgerufen: pip-Dokumentation, Version 26.1.1).** Die Seite beschreibt Versionsangaben, Kommentare mit `#`, Einbinden anderer Dateien mit `-r`, Zeilenfortsetzung mit Backslash und UTF-8 als Standardkodierung. Zu `pip freeze` macht diese Seite keine Aussage; dessen Verhalten haben wir durch Ausführen bestätigt. Quelle: <https://pradyunsg-pip.readthedocs.io/en/stable/reference/requirements-file-format/>.

Typische Fehler im Umgang mit virtuellen Umgebungen sind schnell aufgezählt. Der häufigste: Man vergisst die Aktivierung und installiert unabsichtlich ins System (auf neuen Ubuntu-Versionen sorgt PEP 668 dann wenigstens für eine Fehlermeldung). Der zweite: `pip` statt `python -m pip` verwenden und dadurch ein anderes `pip` erwischen. Der dritte: `sudo pip install`, das bei aktivierter venv die Umgebung verlässt und ins System schreibt. Der vierte: eine `requirements.txt` ohne Versionen (`humanize`), sodass der Rechner morgen eine neuere Version installiert, die sich anders verhält. Der fünfte: die venv ins Repository einchecken oder in ein anderes Verzeichnis verschieben. Letzteres lässt sich leicht nachvollziehen, weil die Startskripte in `bin` den absoluten Pfad des Interpreters enthalten.

```python
import shutil
import subprocess
import sys
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    alt = Path(tmp) / "projekt-alt" / ".venv"
    alt.parent.mkdir()
    subprocess.run([sys.executable, "-m", "venv", str(alt)], check=True)

    r = subprocess.run([str(alt / "bin" / "pip"), "--version"], capture_output=True, text=True)
    print("vor dem Verschieben, Code:", r.returncode)

    neu = Path(tmp) / "projekt-neu"
    shutil.move(str(alt.parent), str(neu))

    try:
        subprocess.run([str(neu / ".venv" / "bin" / "pip"), "--version"],
                       capture_output=True, text=True)
    except FileNotFoundError as fehler:
        print("nach dem Verschieben:", type(fehler).__name__)

    print("Lösung: neu anlegen und aus requirements.txt füllen")
```

```text
vor dem Verschieben, Code: 0
nach dem Verschieben: FileNotFoundError
Lösung: neu anlegen und aus requirements.txt füllen
```

Das Startskript `pip` in der Umgebung enthält in der ersten Zeile (dem *Shebang*, siehe Modul 9) den absoluten Pfad des alten Interpreters. Nach dem Verschieben existiert dieser Pfad nicht mehr, und das Betriebssystem kann das Programm nicht starten. Die offizielle Empfehlung lautet entsprechend: Löschen und neu anlegen, nicht verschieben.

> **Merke**
>
> Pro Projekt eine eigene virtuelle Umgebung (`python3 -m venv .venv`), Pakete mit `python -m pip` installieren, Abhängigkeiten mit festen Versionen in der `requirements.txt` festhalten (`pip freeze`) und die Umgebung daraus neu aufbauen (`pip install -r`). Die venv gehört nicht ins Repository und wird nie verschoben. Für Automatisierung ruft man `.venv/bin/python` mit vollem Pfad auf.

> **Praxisbeispiel**
>
> Auf einem Server, auf dem Ihr Skript per Cron läuft, legen Sie im Projektverzeichnis eine venv an, installieren die Abhängigkeiten aus der `requirements.txt` und tragen in die Cron-Zeile den vollständigen Pfad `/opt/sysw/.venv/bin/python /opt/sysw/sysw.py` ein. Nach einem Update der Abhängigkeiten ändern Sie die Datei, bauen die Umgebung neu und testen. Wer so vorgeht, kann dieselbe Umgebung auf Entwicklungsrechner, Testserver und Produktion nachbauen und vermeidet das klassische „bei mir geht es aber“.

## 7.3 JSON lesen & schreiben

Programme müssen Daten austauschen: mit anderen Programmen, mit Webdiensten, mit sich selbst zu einem späteren Zeitpunkt. Dafür braucht man ein Format, das Struktur festhält (Listen, Zuordnungen, Zahlen, Texte) und das jede Programmiersprache lesen kann. Das am weitesten verbreitete Format dieser Art ist *JSON* (JavaScript Object Notation). Obwohl der Name von JavaScript stammt, ist es sprachunabhängig. Sie begegnen JSON überall: Webdienste liefern ihre Antworten damit (Modul 10), Konfigurationsdateien und Docker-Werkzeuge verwenden es, und viele Programme geben ihre Logs oder Statusinformationen so aus.

JSON kennt nur wenige Bausteine. Ein *Objekt* ist eine Zuordnung von Namen zu Werten, in geschweiften Klammern: `{"name": "web01", "port": 80}`. Ein *Array* ist eine Liste von Werten in eckigen Klammern: `[1, 2, 3]`. Werte sind Texte in *doppelten* Anführungszeichen, Zahlen, die Wahrheitswerte `true` und `false` oder `null` für „kein Wert“. Es gibt Regeln, die Python-Programmierer anfangs stolpern lassen: Die Namen (Schlüssel) müssen Texte in doppelten Anführungszeichen sein, einfache Anführungszeichen sind ungültig, am Ende einer Liste darf kein Komma stehen, und Kommentare gibt es nicht.

Das Python-Modul `json` aus der Standardbibliothek wandelt zwischen Python-Objekten und JSON-Text um. Die Zuordnung ist einfach: Ein `dict` wird zum Objekt, `list` und `tuple` werden zu Arrays, `str` wird zum Text, `int` und `float` werden zu Zahlen, `True` und `False` werden zu `true` und `false`, und `None` wird zu `null`. Beim Zurücklesen gilt die Umkehrung, und deshalb kommt aus einem Tupel eine Liste zurück. Vier Funktionen genügen für fast alles: `json.dumps(objekt)` erzeugt einen Text (das *s* steht für *string*), `json.loads(text)` liest einen Text, `json.dump(objekt, datei)` schreibt in eine geöffnete Datei und `json.load(datei)` liest aus einer geöffneten Datei. Das erste Beispiel führt eine Hin- und Rückwandlung durch und zeigt die wichtigsten Parameter.

```python
import json

server = {
    "name": "web01",
    "aktiv": True,
    "ports": (80, 443),
    "notiz": None,
    "standort": "Köln",
    "last": 0.75,
    "tags": ["prod", "web"],
}

kompakt = json.dumps(server)
print(kompakt)

lesbar = json.dumps(server, indent=2, ensure_ascii=False, sort_keys=True)
print(lesbar)

zurueck = json.loads(kompakt)
print(zurueck["ports"], type(zurueck["ports"]).__name__)
print(zurueck == server)
```

```text
{"name": "web01", "aktiv": true, "ports": [80, 443], "notiz": null, "standort": "K\u00f6ln", "last": 0.75, "tags": ["prod", "web"]}
{
  "aktiv": true,
  "last": 0.75,
  "name": "web01",
  "notiz": null,
  "ports": [
    80,
    443
  ],
  "standort": "Köln",
  "tags": [
    "prod",
    "web"
  ]
}
[80, 443] list
False
```

Der erste Aufruf liefert die kompakte Form in einer Zeile. Bemerkenswert ist `K\u00f6ln`: Standardmäßig ist `ensure_ascii=True`, und die Dokumentation beschreibt, dass dann alle Nicht-ASCII-Zeichen mit Escape-Folgen geschrieben werden. Das ist gültiges JSON und lässt sich auch korrekt zurücklesen, wirkt aber in lesbaren Dateien störend. Mit `ensure_ascii=False` bleiben Umlaute erhalten; dann muss man die Datei aber mit `encoding="utf-8"` schreiben. Der Parameter `indent=2` formatiert mit Einrückung (die Voreinstellung `None` ergibt die kompakte Form), `sort_keys=True` sortiert die Schlüssel, was bei Vergleichen und Versionsverwaltung hilft. Die letzten beiden Zeilen zeigen die Stolperfalle der Umwandlung: Das Tupel `(80, 443)` kommt als Liste zurück, und deshalb ist das zurückgelesene Objekt nicht gleich dem Original.

Der Normalfall für Skripte ist das Lesen und Schreiben von Dateien. Dabei öffnet man die Datei mit der `with`-Anweisung (Modul 4) und gibt immer `encoding="utf-8"` an. Weil JSON-Dateien meist von Menschen gepflegt werden, ist es üblich, beim Schreiben `indent=2` zu verwenden. Beim Lesen von Konfigurationen wertet man fehlende Einträge nicht mit einem Fehler aus, sondern verwendet Standardwerte: `konfig.get("port", 8080)`.

```python
import json
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "server.json"

    daten = {"hosts": [{"name": "web01", "port": 80}, {"name": "büro-süd"}]}
    with open(datei, "w", encoding="utf-8") as f:
        json.dump(daten, f, indent=2, ensure_ascii=False)
        f.write("\n")                      # Datei mit Zeilenumbruch beenden

    print(datei.read_text(encoding="utf-8"))

    with open(datei, encoding="utf-8") as f:
        gelesen = json.load(f)

    for host in gelesen["hosts"]:
        print(host["name"], host.get("port", 22))
```

```text
{
  "hosts": [
    {
      "name": "web01",
      "port": 80
    },
    {
      "name": "büro-süd"
    }
  ]
}

web01 80
büro-süd 22
```

Das erste Objekt hat einen Port, das zweite nicht; `host.get("port", 22)` liefert dann den Standardwert 22. Die Umlaute im Namen bleiben lesbar, weil `ensure_ascii=False` und die UTF-8-Kodierung zusammenspielen. Die Leerzeile in der Ausgabe stammt vom Zeilenumbruch am Dateiende, den wir mit `write` angehängt haben und den `print` noch einmal ergänzt.

Eine Besonderheit für die Shell: Das Modul `json` bringt ein Kommandozeilenwerkzeug mit, mit dem man JSON schön formatieren und auf Gültigkeit prüfen kann: `python3 -m json.tool`. Es liest von der Standardeingabe oder aus einer Datei und gibt eingerückt aus. Das ist bei der Fehlersuche in API-Antworten sehr nützlich.

```python
import subprocess
import sys

eingabe = '{"b": 1, "a": [1, 2], "text": "Köln"}'
r = subprocess.run([sys.executable, "-m", "json.tool"], input=eingabe,
                   capture_output=True, text=True)
print(r.stdout)
print("Code:", r.returncode)

r = subprocess.run([sys.executable, "-m", "json.tool"], input='{"b": 1,}',
                   capture_output=True, text=True)
print("ungültig -> Code", r.returncode, "-", r.stderr.strip().splitlines()[-1])
```

```text
{
    "b": 1,
    "a": [
        1,
        2
    ],
    "text": "K\u00f6ln"
}

Code: 0
ungültig -> Code 1 - Illegal trailing comma before end of object: line 1 column 8 (char 7)
```

Das Werkzeug erhält die Reihenfolge der Schlüssel; Umlaute schreibt es standardmäßig als Escape-Folge (`\u00f6`), mit der Option `--no-ensure-ascii` bleiben sie lesbar. Bei ungültigem Eingabetext meldet es den Fehler mit Zeile und Spalte und endet mit Rückgabecode 1 (der Wortlaut der Meldung hängt von der Python-Version ab); das lässt sich in Skripten als Gültigkeitsprüfung verwenden.

Fehler beim Lesen sind der Normalfall, nicht die Ausnahme: Menschen bearbeiten Konfigurationsdateien von Hand und vergessen dabei Kommas, setzen überflüssige Kommas oder einfache Anführungszeichen. Beim Parsen löst Python in diesen Fällen die Ausnahme `json.JSONDecodeError` aus, eine Unterklasse von `ValueError` (seit Python 3.5). Sie trägt genaue Angaben mit sich: `msg` (die Meldung), `lineno` und `colno` (Zeile und Spalte) sowie `pos` (Position im Text). Daraus baut man eine hilfreiche Fehlermeldung, die dem Benutzer sagt, *wo* er nachsehen muss, statt ihn mit einem Traceback zu konfrontieren.

```python
import json

def lade(text, name="konfig.json"):
    try:
        return json.loads(text)
    except json.JSONDecodeError as fehler:
        zeile = text.splitlines()[fehler.lineno - 1] if text.strip() else ""
        raise ValueError(
            f"{name}, Zeile {fehler.lineno}, Spalte {fehler.colno}: {fehler.msg}\n"
            f"    {zeile}\n"
            f"    {' ' * (fehler.colno - 1)}^"
        ) from fehler

faelle = {
    "Komma am Ende":       '{\n  "port": 80,\n}',
    "einfache Anführungszeichen": "{'port': 80}",
    "Kommentar":           '{\n  "port": 80 // Standard\n}',
    "leere Datei":         "",
}
for titel, text in faelle.items():
    print("==", titel)
    try:
        lade(text)
    except ValueError as fehler:
        print(fehler)
```

```text
== Komma am Ende
konfig.json, Zeile 2, Spalte 13: Illegal trailing comma before end of object
      "port": 80,
                ^
== einfache Anführungszeichen
konfig.json, Zeile 1, Spalte 2: Expecting property name enclosed in double quotes
    {'port': 80}
     ^
== Kommentar
konfig.json, Zeile 2, Spalte 14: Expecting ',' delimiter
      "port": 80 // Standard
                 ^
== leere Datei
konfig.json, Zeile 1, Spalte 1: Expecting value

    ^
```

Die Funktion `lade` fängt `JSONDecodeError`, holt die betroffene Zeile aus dem Text und zeigt mit einem Pfeil auf die Spalte. Wir haben vier typische Fehler getestet. Beim ersten Fall (Komma am Ende) nennt Python 3.13 das überflüssige Komma ausdrücklich; ältere Versionen meldeten an dieser Stelle „Expecting property name enclosed in double quotes“, die Meldung hängt also von Ihrer Python-Version ab. Beim Kommentar liest der Parser das Objekt bis `80` und erwartet dann ein Komma oder eine schließende Klammer; auch hier kann der Wortlaut abweichen. Einer leeren Datei fehlt schon der erste Wert. Mit `from fehler` bleibt die ursprüngliche Ausnahme erhalten.

Nicht alles, was in Python existiert, lässt sich in JSON ausdrücken. Ein Datum (`datetime`), ein Pfad (`Path`) oder eine Menge (`set`) lösen beim Schreiben einen `TypeError` aus. Man kann dem Schreiber mit dem Parameter `default=funktion` sagen, wie er solche Objekte umwandeln soll: Die Funktion wird nur für Objekte aufgerufen, die er nicht kennt, und gibt eine JSON-taugliche Darstellung zurück oder löst selbst einen `TypeError` aus (so beschreibt es die Dokumentation).

```python
import json
from datetime import datetime
from pathlib import Path

def als_json(objekt):
    if isinstance(objekt, datetime):
        return objekt.isoformat()
    if isinstance(objekt, Path):
        return str(objekt)
    if isinstance(objekt, set):
        return sorted(objekt)
    raise TypeError(f"{type(objekt).__name__} ist nicht JSON-fähig")

ereignis = {
    "zeit": datetime(2026, 12, 14, 19, 30, 5),
    "datei": Path("/var/log/syslog"),
    "hosts": {"web02", "web01"},
}
print(json.dumps(ereignis, default=als_json, indent=1))

try:
    json.dumps({"x": {1, 2}})
except TypeError as fehler:
    print("Ohne default:", fehler)
```

```text
{
 "zeit": "2026-12-14T19:30:05",
 "datei": "/var/log/syslog",
 "hosts": [
  "web01",
  "web02"
 ]
}
Ohne default: Object of type set is not JSON serializable
```

Das Datum wird im Standardformat nach ISO 8601 geschrieben (`JJJJ-MM-TTThh:mm:ss`), das jedes Programm versteht und das alphabetisch wie zeitlich sortiert. Beim Zurücklesen bleibt es ein Text, den Sie bei Bedarf mit `datetime.fromisoformat()` in ein Datum zurückverwandeln (mehr dazu in Modul 8).

Ein weiterer Punkt betrifft das Schreiben von Dateien, die andere Programme sofort lesen. Bricht Ihr Skript beim Schreiben ab (Platte voll, Strg+C, Stromausfall), bleibt eine halbe, ungültige Datei zurück; ein Programm, das sie danach lädt, scheitert. Die Lösung ist ein Muster, das Sie bei der Sicherung kennengelernt haben: erst in eine temporäre Datei im selben Verzeichnis schreiben und diese dann in einem einzigen Schritt über das Ziel *umbenennen*. Eine Umbenennung innerhalb desselben Dateisystems ist unter Linux unteilbar: Andere Programme sehen entweder die alte oder die neue Datei, nie eine halbe. In Python leistet das `os.replace()`.

```python
import json
import os
import tempfile
from pathlib import Path

def speichere_json(pfad, daten):
    """Schreibt JSON atomar: erst temporär, dann umbenennen."""
    pfad = Path(pfad)
    vorlaeufig = pfad.with_name(pfad.name + ".tmp")
    try:
        with open(vorlaeufig, "w", encoding="utf-8") as f:
            json.dump(daten, f, indent=2, ensure_ascii=False)
            f.write("\n")
        os.replace(vorlaeufig, pfad)
    except BaseException:
        vorlaeufig.unlink(missing_ok=True)
        raise

with tempfile.TemporaryDirectory() as tmp:
    ziel = Path(tmp) / "status.json"
    speichere_json(ziel, {"stand": 1})
    speichere_json(ziel, {"stand": 2})
    print(json.loads(ziel.read_text(encoding="utf-8")))

    try:
        speichere_json(ziel, {"stand": {1, 2}})       # nicht serialisierbar
    except TypeError:
        pass
    print(json.loads(ziel.read_text(encoding="utf-8")))     # Datei unversehrt
    print(sorted(p.name for p in Path(tmp).iterdir()))     # kein .tmp zurückgeblieben
```

```text
{'stand': 2}
{'stand': 2}
['status.json']
```

Nach den beiden ersten Speichervorgängen enthält die Datei den Stand 2; der zweite hat den ersten in einem Schritt ersetzt. Beim dritten Versuch bricht das Schreiben mitten in der Datei mit einem `TypeError` ab; die vorhandene Datei `status.json` ist unverändert, und auch die temporäre Datei wurde aufgeräumt. Wir fangen hier `BaseException`, was normalerweise abzulehnen ist; es ist an dieser Stelle gerechtfertigt, weil die Funktion nach dem Aufräumen mit `raise` die Ausnahme sofort weiterreicht, auch bei Strg+C.

Eine verwandte Form ist *JSON Lines* (auch NDJSON): eine Datei, in der jede Zeile ein eigenes JSON-Objekt ist. Sie eignet sich für Logs und Ereignisse, weil man sie Zeile für Zeile lesen und anhängen kann, ohne die ganze Datei zu laden. Dass JSON Lines kein Teil des `json`-Moduls ist, stört nicht: Man schreibt pro Zeile `json.dumps(eintrag)` und liest mit `json.loads(zeile)`.

```python
import json
import tempfile
from pathlib import Path

ereignisse = [
    {"zeit": "19:00", "host": "web01", "stufe": "INFO"},
    {"zeit": "19:05", "host": "web02", "stufe": "ERROR"},
    {"zeit": "19:07", "host": "web01", "stufe": "ERROR"},
]

with tempfile.TemporaryDirectory() as tmp:
    logdatei = Path(tmp) / "ereignisse.jsonl"
    with open(logdatei, "a", encoding="utf-8") as f:
        for e in ereignisse:
            f.write(json.dumps(e, ensure_ascii=False) + "\n")

    fehler = []
    with open(logdatei, encoding="utf-8") as f:
        for zeile in f:
            eintrag = json.loads(zeile)
            if eintrag["stufe"] == "ERROR":
                fehler.append(eintrag["host"])
    print(logdatei.read_text(encoding="utf-8").splitlines()[1])
    print(fehler)
```

```text
{"zeit": "19:05", "host": "web02", "stufe": "ERROR"}
['web02', 'web01']
```

Beim Umgang mit JSON gibt es einige Stolperfallen, die das Modul *stillschweigend* behandelt. Erstens sind Schlüssel in JSON immer Texte. Die Dokumentation sagt es so: Beim Umwandeln eines Dictionarys werden alle Schlüssel zu Texten gemacht, und `loads(dumps(x)) != x`, wenn `x` Schlüssel anderer Typen hat. Aus dem Schlüssel `1` wird also `"1"`. Zweitens ignoriert das Modul bei doppelten Schlüsseln in einem Objekt alles außer dem letzten Paar, ohne eine Warnung. Drittens akzeptiert es standardmäßig auch `NaN` und `Infinity`, die nach dem JSON-Standard gar nicht erlaubt sind (mit `allow_nan=False` beim Schreiben verhindert man das). Viertens erwartet `json.load` eine *geöffnete Datei*, nicht einen Dateinamen.

```python
import json

# FALSCH 1: ganzzahlige Schlüssel kommen als Text zurück
ports = {80: "http", 443: "https"}
zurueck = json.loads(json.dumps(ports))
print(zurueck, zurueck == ports)

# FALSCH 2: doppelte Schlüssel, der letzte gewinnt ohne Warnung
print(json.loads('{"port": 80, "port": 8080}'))

# FALSCH 3: ein Dateiname statt einer Datei
try:
    json.load("konfig.json")
except AttributeError as fehler:
    print("AttributeError:", fehler)

# RICHTIG: loads für Text, load für Dateien, Schlüssel als Text planen
print(json.loads('{"80": "http"}')["80"])
```

```text
{'80': 'http', '443': 'https'} False
{'port': 8080}
AttributeError: 'str' object has no attribute 'read'
http
```

> **Ergänzung aus der Nachrecherche**
>
> **Aussagen der json-Dokumentation (abgerufen: Python 3.11).** Die Dokumentation bestätigt die hier beschriebenen Regeln: Die Zuordnung der Typen (dict object, list und tuple array, `None` null); `indent`, `sort_keys` und `ensure_ascii` (Standard `True`) mit den beschriebenen Wirkungen; der Parameter `default`; `JSONDecodeError` als Unterklasse von `ValueError` mit den Attributen `msg`, `doc`, `pos`, `lineno`, `colno`; die Umwandlung aller Schlüssel in Texte; das Ignorieren doppelter Schlüssel (mit `object_pairs_hook` änderbar); die Annahme von `NaN`, `Infinity` und `-Infinity` (mit `allow_nan=False` beim Schreiben unterbindbar). Außerdem enthält die Seite einen Sicherheitshinweis: Beim Parsen von JSON aus nicht vertrauenswürdigen Quellen kann ein bösartiger Text viel CPU und Speicher verbrauchen; die Dokumentation empfiehlt, die Größe der zu parsenden Daten zu begrenzen. Das ist für Skripte relevant, die JSON aus dem Netz laden (Modul 10). Quelle: <https://docs.python.org/3.11/library/json.html>. Das Werkzeug `python3 -m json.tool` und das atomare Schreiben mit `os.replace` wurden durch Ausführen bestätigt, nicht in der Dokumentation nachgelesen.

> **Merke**
>
> `json.dumps`/`json.loads` arbeiten mit Texten, `json.dump`/`json.load` mit geöffneten Dateien (immer `encoding="utf-8"`). Für lesbare Dateien: `indent=2, ensure_ascii=False`. Tupel kommen als Listen zurück, Schlüssel sind immer Texte, Datum und Pfad brauchen `default=`. Fangen Sie `JSONDecodeError` und zeigen Sie Zeile und Spalte; schreiben Sie wichtige Dateien atomar mit `os.replace`.

> **Praxisbeispiel**
>
> Fast jeder Webdienst, den Sie in Modul 10 abfragen, antwortet mit JSON; Monitoring-Werkzeuge, `docker inspect` und viele Cloud-Programme geben JSON aus. Mit `subprocess` und `json.loads` verbinden Sie beides: Sie rufen ein Programm mit der Option für JSON-Ausgabe auf, lesen die Antwort als Python-Objekt und werten sie aus. Konfigurationen, die ein Mensch pflegt, schreiben Sie eher in YAML oder TOML (nächste Lerneinheit), weil JSON dort keine Kommentare erlaubt.

## 7.4 YAML & CSV

JSON ist ideal für den Austausch zwischen Programmen, aber unbequem, wenn ein Mensch die Datei pflegen soll: Es gibt keine Kommentare, jede Zeichenkette braucht Anführungszeichen, und ein vergessenes Komma macht die ganze Datei ungültig. Für Konfigurationsdateien verwendet man deshalb oft *YAML* (ursprünglich „YAML Ain’t Markup Language“). Eine YAML-Datei sieht aus wie eine gut gegliederte Notiz: Struktur entsteht durch *Einrückung*, Listen bestehen aus Zeilen mit einem Bindestrich, Zuordnungen aus Zeilen der Form `schluessel: wert`, und Kommentare beginnen mit `#`. Docker-Compose, Kubernetes, Ansible und viele CI-Systeme benutzen YAML. Die zweite Hälfte dieser Lerneinheit gilt *CSV*, dem einfachsten Tabellenformat, das jede Tabellenkalkulation lesen und schreiben kann.

YAML gehört *nicht* zur Standardbibliothek. Sie brauchen das Paket `PyYAML`, das Sie in einer virtuellen Umgebung mit `python -m pip install pyyaml` installieren (oder unter Debian und Ubuntu systemweit mit `sudo apt install python3-yaml`, siehe Lerneinheit 7.2). Importiert wird es unter dem Namen `yaml`. Man erkennt hier einen allgemeinen Punkt: Der Name des Pakets bei `pip` und der Name beim `import` müssen nicht übereinstimmen. Die zwei wichtigsten Funktionen heißen `yaml.safe_load(text)` (liest YAML in Python-Objekte) und `yaml.safe_dump(objekt)` (schreibt Python-Objekte als YAML). Das Wort *safe* ist wichtig und wird weiter unten erklärt.

```python
import yaml

text = """
# Konfiguration für den Verzeichnisreport
verzeichnisse:
  - /var/log
  - /srv/daten
endungen: [log, txt]        # Kurzschreibweise für Listen
mindestgroesse: 1024
format: text
optionen:
  rekursiv: true
  tiefe: 3
beschreibung: >
  Dieser Text steht in der Datei
  über zwei Zeilen verteilt.
"""

konfig = yaml.safe_load(text)
print(konfig["verzeichnisse"])
print(konfig["endungen"], konfig["mindestgroesse"], konfig["optionen"])
print(repr(konfig["beschreibung"]))
print({schluessel: type(wert).__name__ for schluessel, wert in konfig.items()})
```

```text
['/var/log', '/srv/daten']
['log', 'txt'] 1024 {'rekursiv': True, 'tiefe': 3}
'Dieser Text steht in der Datei über zwei Zeilen verteilt.\n'
{'verzeichnisse': 'list', 'endungen': 'list', 'mindestgroesse': 'int', 'format': 'str', 'optionen': 'dict', 'beschreibung': 'str'}
```

Das Ergebnis ist ein gewöhnliches Dictionary mit Listen, Zahlen, Texten und einem verschachtelten Dictionary; YAML hat die Typen automatisch erkannt (`1024` ist eine Zahl, `true` ein Wahrheitswert, `[log, txt]` eine Liste). Das Zeichen `>` leitet einen *gefalteten* Textblock ein: Die Zeilen werden zu einer verbunden. Die Kommentare sind nach dem Einlesen verschwunden; deshalb eignet sich YAML zum *Lesen* von Hand gepflegter Dateien, aber schlecht zum automatischen Zurückschreiben (die Kommentare gingen verloren).

Das automatische Erkennen der Typen ist Segen und Fluch. YAML folgt dabei Regeln, die in der Version 1.1 des Standards festgelegt wurden, und die von PyYAML umgesetzt werden. Einige Ergebnisse überraschen: `no`, `yes`, `on` und `off` sind Wahrheitswerte. Die Liste der Ländercodes `[de, no, at]` enthält deshalb statt Norwegens `no` den Wert `False`; man nennt das das „Norwegen-Problem“. Eine Versionsnummer wie `1.10` wird zur Zahl 1,1 und verliert die Null. Und `080` ist keine gültige Oktalzahl und bleibt deshalb ein Text, während `0x1f` als Hexadezimalzahl zur Zahl 31 wird. Die Abhilfe ist einfach: Alles, was Text bleiben soll, setzt man in Anführungszeichen.

```python
import yaml

problematisch = """
laender: [de, no, at]
schalter: on
version: 1.10
port: 080
farbe: 0x1f
"""
print(yaml.safe_load(problematisch))

besser = """
laender: ["de", "no", "at"]
schalter: true
version: "1.10"
port: "080"
farbe: "0x1f"
"""
print(yaml.safe_load(besser))
```

```text
{'laender': ['de', False, 'at'], 'schalter': True, 'version': 1.1, 'port': '080', 'farbe': 31}
{'laender': ['de', 'no', 'at'], 'schalter': True, 'version': '1.10', 'port': '080', 'farbe': '0x1f'}
```

Im ersten Fall hat sich fast jeder Wert anders verhalten als erwartet; im zweiten Fall sind alle Werte durch Anführungszeichen eindeutig. Die Regel für Konfigurationen lautet daher: Texte, die wie Zahlen oder Wahrheitswerte aussehen können (Versionen, Postleitzahlen, Ländercodes, Ports mit führender Null), immer quoten, und Wahrheitswerte als `true` und `false` schreiben.

Nun zum Wort *safe*. YAML ist so mächtig, dass der vollständige Lader Python-Objekte erzeugen kann, sogar Aufrufe von Funktionen. Die Dokumentation des Linters Ruff (siehe Ergänzung) beschreibt das so: `yaml.load` erlaubt die Erzeugung *beliebiger* Python-Objekte, und diese können benutzt werden, um beliebigen Code auszuführen. `yaml.safe_load` dagegen erlaubt nur einfache Objekte wie Zahlen, Texte, Listen und Dictionaries. Wer eine YAML-Datei mit dem vollen Lader liest, die er nicht selbst geschrieben hat, lässt also Fremden die Tür zum eigenen Rechner offen. Das folgende Beispiel zeigt den Unterschied mit einem harmlosen Aufruf: Der Text verlangt, die Funktion `os.getcwd` auszuführen. Der sichere Lader verweigert das, der unsichere gehorcht. Verwenden Sie in Ihren Skripten *immer* `safe_load`. (Bei PyYAML 6 verlangt `yaml.load` ohnehin ein ausdrückliches Lader-Argument.)

```python
import yaml

gefaehrlich = "wert: !!python/object/apply:os.getcwd []"

try:
    yaml.safe_load(gefaehrlich)
except yaml.YAMLError as fehler:
    print("safe_load verweigert:", type(fehler).__name__)

ergebnis = yaml.load(gefaehrlich, Loader=yaml.UnsafeLoader)
print("UnsafeLoader hat eine Funktion ausgeführt:", type(ergebnis["wert"]).__name__)

try:
    yaml.load("a: 1")
except TypeError as fehler:
    print("Ohne Lader-Angabe:", fehler)
```

```text
safe_load verweigert: ConstructorError
UnsafeLoader hat eine Funktion ausgeführt: str
Ohne Lader-Angabe: load() missing 1 required positional argument: 'Loader'
```

Der unsichere Lader hat `os.getcwd()` wirklich aufgerufen und deshalb eine Zeichenkette (den Namen des aktuellen Verzeichnisses) geliefert. Statt `os.getcwd` könnte dort jede andere Funktion stehen, auch `os.system` mit einem beliebigen Befehl. Das ist der Grund für die Regel „nur `safe_load`“.

Auch beim Schreiben von YAML gibt es zwei Parameter, die man kennen sollte: `allow_unicode=True` lässt Umlaute lesbar stehen (sonst schreibt `safe_dump` sie als Escape-Folge), und `sort_keys=False` behält die Reihenfolge der Schlüssel bei (sonst sortiert es alphabetisch). Fehler beim Lesen melden sich als `yaml.YAMLError`; die Fehlermeldung trägt eine Markierung mit Zeile und Spalte (`problem_mark`, nullbasiert gezählt). Besonders häufig sind Tabulatoren (in YAML verboten, nur Leerzeichen sind erlaubt) und falsche Einrückungen.

```python
import yaml

daten = {"name": "Büro Süd", "hosts": ["web01", "web02"], "aktiv": True}
print(yaml.safe_dump(daten, allow_unicode=True, sort_keys=False))

kaputt = "hosts:\n\t- web01\n"          # Tabulator statt Leerzeichen
try:
    yaml.safe_load(kaputt)
except yaml.YAMLError as fehler:
    marke = fehler.problem_mark
    print(f"YAML-Fehler in Zeile {marke.line + 1}, Spalte {marke.column + 1}: {fehler.problem}")
```

```text
name: Büro Süd
hosts:
- web01
- web02
aktiv: true

YAML-Fehler in Zeile 2, Spalte 1: found character '\t' that cannot start any token
```

Beachten Sie, dass `safe_dump` Listen standardmäßig ohne zusätzliche Einrückung schreibt (`- web01` steht unter `hosts:` auf derselben Ebene); das ist gültiges YAML. Die Fehlermeldung benennt den Tabulator als Ursache und die Stelle; die Zählung `problem_mark.line + 1` wandelt die nullbasierte Zeile in eine für Menschen lesbare Zeile um.

Es gibt noch eine dritte Möglichkeit für Konfigurationsdateien, die seit Python 3.11 in der Standardbibliothek steckt: *TOML*. Das Format ähnelt klassischen INI-Dateien, ist eindeutig definiert, kennt Kommentare und Typen (Texte stehen immer in Anführungszeichen, Wahrheitswerte heißen `true` und `false`) und hat keine Einrückungsfallen. Die Python-Packaging-Welt hat es als Standard gewählt, die Projektdatei `pyproject.toml` (Modul 13) ist ein Beispiel. Das Modul `tomllib` kann TOML *lesen*, aber nicht schreiben; die Dokumentation zum zugehörigen PEP 680 sagt ausdrücklich, dass es nur Parser-Funktionen gibt. Die Datei muss im Binärmodus (`"rb"`) geöffnet werden, damit der Parser die Kodierung UTF-8 selbst sicherstellen kann.

```python
import tomllib

text = """
# Konfiguration für den Verzeichnisreport
verzeichnisse = ["/var/log", "/srv/daten"]
endungen = ["log", "txt"]
mindestgroesse = 1024
format = "text"                  # Texte stehen immer in Anführungszeichen

[optionen]                       # eine Tabelle (verschachtelt)
rekursiv = true
tiefe = 3
"""

konfig = tomllib.loads(text)
print(konfig)

try:
    tomllib.loads("port = 80\nport = 81")
except tomllib.TOMLDecodeError as fehler:
    print("Fehler:", fehler)
```

```text
{'verzeichnisse': ['/var/log', '/srv/daten'], 'endungen': ['log', 'txt'], 'mindestgroesse': 1024, 'format': 'text', 'optionen': {'rekursiv': True, 'tiefe': 3}}
Fehler: Cannot overwrite a value (at end of document)
```

TOML liefert dasselbe Dictionary wie die YAML-Konfiguration oben. Dass doppelte Schlüssel einen Fehler auslösen (anders als bei JSON, wo der letzte stillschweigend gewinnt), ist ein Vorteil von TOML: Vertippte Konfigurationen fallen auf. Wenn Sie die Wahl haben, ist TOML für Konfigurationsdateien von Hand die sicherste Wahl; YAML begegnet Ihnen vor allem, weil viele Werkzeuge es verlangen.

Nun zum zweiten Teil der Lerneinheit. *CSV* (Comma-Separated Values) ist ein Tabellenformat: Jede Zeile ist ein Datensatz, die Felder sind durch ein Trennzeichen getrennt, meist ein Komma. Die erste Zeile enthält häufig die Spaltennamen. CSV ist ein einfaches Austauschformat für Exporte (Benutzerlisten, Inventare, Abrechnungen) und für Daten, die in einer Tabellenkalkulation weiterverarbeitet werden. Das Modul `csv` der Standardbibliothek liest und schreibt es korrekt, auch wenn ein Feld selbst ein Komma, Anführungszeichen oder einen Zeilenumbruch enthält; ein `zeile.split(",")` tut das nicht und ist deshalb keine Alternative.

Die wichtigsten Klassen sind `csv.reader` und `csv.writer` (Zeilen als Listen) sowie `csv.DictReader` und `csv.DictWriter` (Zeilen als Dictionaries mit den Spaltennamen als Schlüssel). Die Dokumentation gibt zwei Regeln vor, die man immer befolgen sollte: Datei mit `newline=""` öffnen (sonst werden Zeilenumbrüche innerhalb von Feldern falsch behandelt, und unter Systemen mit `\r\n`-Zeilenenden entsteht beim Schreiben ein zusätzliches `\r`), und die Kodierung ausdrücklich angeben. Außerdem gilt: Der Leser liefert *jedes Feld als Text*; es findet keine Umwandlung in Zahlen statt (nur mit der Option `QUOTE_NONNUMERIC` würden nicht in Anführungszeichen stehende Felder zu Gleitkommazahlen). Die Umwandlung ist Ihre Aufgabe.

```python
import csv
import tempfile
from pathlib import Path

hosts = [
    {"name": "web01", "standort": "Köln", "cpu": 8,  "notiz": ""},
    {"name": "db01",  "standort": "Büro, Süd", "cpu": 16, "notiz": 'Cluster "Nord"'},
]

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "hosts.csv"
    with open(datei, "w", newline="", encoding="utf-8") as f:
        schreiber = csv.DictWriter(f, fieldnames=["name", "standort", "cpu", "notiz"])
        schreiber.writeheader()
        schreiber.writerows(hosts)

    print(datei.read_text(encoding="utf-8"), end="")
    print("----")

    with open(datei, newline="", encoding="utf-8") as f:
        zeilen = list(csv.DictReader(f))

    print(zeilen[1])
    print(type(zeilen[0]["cpu"]).__name__, sum(int(z["cpu"]) for z in zeilen))
    print("Falsch addiert:", zeilen[0]["cpu"] + zeilen[1]["cpu"])
```

```text
name,standort,cpu,notiz
web01,Köln,8,
db01,"Büro, Süd",16,"Cluster ""Nord"""
----
{'name': 'db01', 'standort': 'Büro, Süd', 'cpu': '16', 'notiz': 'Cluster "Nord"'}
str 24
Falsch addiert: 816
```

Das Modul hat beim Schreiben die Felder mit Komma oder Anführungszeichen selbst in Anführungszeichen gesetzt und die Anführungszeichen im Feld verdoppelt, wie es der CSV-Konvention entspricht; beim Lesen kommen die Originalwerte zurück. Die letzte Zeile zeigt die häufigste Falle: `"8" + "16"` ist die Zeichenkette `"816"`, nicht 24. Zahlen müssen mit `int()` oder `float()` umgewandelt werden, bevor man rechnet oder sortiert; als Text sortiert `"9"` nach `"10"`.

In Deutschland, Österreich und anderen Ländern, in denen das Komma als Dezimaltrenner gilt, speichern Tabellenkalkulationen CSV-Dateien meist mit dem Semikolon als Trennzeichen, und Dezimalzahlen mit Komma. Das Modul `csv` verarbeitet das über den Parameter `delimiter=";"`; die Zahlen wandelt man selbst um, indem man das Komma durch einen Punkt ersetzt. Diese Aussage über das Verhalten deutschsprachiger Tabellenkalkulationen beruht auf Erfahrung und wurde nicht in einer Quelle nachgeschlagen; prüfen Sie sie an Ihrer eigenen Datei.

```python
import csv
import io

export = "Name;Preis;Menge\nFestplatte;89,90;3\nKabel;4,50;10\n"

summe = 0.0
for zeile in csv.DictReader(io.StringIO(export, newline=""), delimiter=";"):
    preis = float(zeile["Preis"].replace(",", "."))
    menge = int(zeile["Menge"])
    summe += preis * menge
    print(f"{zeile['Name']:<12}{preis:>8.2f} x {menge:>2}")
print(f"Summe: {summe:.2f}")
```

```text
Festplatte     89.90 x  3
Kabel           4.50 x 10
Summe: 314.70
```

Mit `io.StringIO` kann man einen Text wie eine Datei lesen; das ist praktisch für Tests. Der Parameter `delimiter=";"` verarbeitet das deutsche Format, die Dezimalkommas wandeln wir vor `float()` in Punkte um.

Bleibt die Frage der Kodierung, die bei CSV-Dateien aus Tabellenkalkulationen ständig Ärger macht. Manche Programme (zum Beispiel Excel beim Speichern als „CSV UTF-8“) schreiben am Dateianfang ein unsichtbares Zeichen, das *Byte Order Mark* (BOM). Liest man so eine Datei als `utf-8`, gehört das Zeichen zum ersten Spaltennamen, und `zeile["name"]` schlägt mit `KeyError` fehl, obwohl die Spalte vorhanden zu sein scheint. Die Kodierung `utf-8-sig` entfernt das BOM beim Lesen. Zum Schreiben von Dateien, die Excel korrekt mit Umlauten öffnen soll, verwendet man ebenfalls `utf-8-sig`. Dass diese Kodierung für Excel zweckmäßig ist, ist verbreitetes Fachwissen; die Python-Dokumentation zum Modul `csv` erwähnt sie nicht. Das folgende Beispiel zeigt, wie sich das Problem äußert und wie man es löst. Außerdem sehen Sie, was passiert, wenn man `newline=""` vergisst und ein Feld einen Windows-Zeilenumbruch enthält.

```python
import csv
import io
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "von-excel.csv"
    datei.write_bytes(b"\xef\xbb\xbfname,cpu\r\nweb01,8\r\n")      # mit BOM

    with open(datei, newline="", encoding="utf-8") as f:
        print("FALSCH :", list(csv.DictReader(f).fieldnames))
    with open(datei, newline="", encoding="utf-8-sig") as f:
        print("RICHTIG:", list(csv.DictReader(f).fieldnames))

    # newline="" vergessen: ein Windows-Zeilenumbruch IM Feld wird verändert
    puffer = io.StringIO(newline="")
    csv.writer(puffer).writerow(["a\r\nb", "x"])
    datei2 = Path(tmp) / "feld.csv"
    datei2.write_text(puffer.getvalue(), encoding="utf-8", newline="")
    with open(datei2, encoding="utf-8") as f:                          # ohne newline=""
        print("FALSCH :", repr(next(csv.reader(f))[0]))
    with open(datei2, newline="", encoding="utf-8") as f:              # mit newline=""
        print("RICHTIG:", repr(next(csv.reader(f))[0]))
```

```text
FALSCH : ['\ufeffname', 'cpu']
RICHTIG: ['name', 'cpu']
FALSCH : 'a\nb'
RICHTIG: 'a\r\nb'
```

Beim ersten Fall steht vor `name` das unsichtbare Zeichen `\ufeff`; die Spalte heißt eigentlich anders als erwartet. Beim zweiten Fall wandelt Pythons normales Textlesen den Zeilenumbruch `\r\n` im Feld stillschweigend in `\n` um; die Daten sind danach verändert. Mit `newline=""` übernimmt das Modul `csv` selbst die Behandlung der Zeilenumbrüche, und der Wert bleibt erhalten.

Eine typische Aufgabe zum Schluss: Daten zwischen den Formaten umwandeln. Das Inventar einer Abteilung liegt als CSV vor, ein anderes Programm will JSON. Mit den Bausteinen dieses Moduls sind das wenige Zeilen: lesen, Typen umwandeln, schreiben.

```python
import csv
import io
import json

inventar = "name,cpu,aktiv\nweb01,8,ja\ndb01,16,nein\n"

hosts = []
for zeile in csv.DictReader(io.StringIO(inventar, newline="")):
    hosts.append({
        "name": zeile["name"],
        "cpu": int(zeile["cpu"]),
        "aktiv": zeile["aktiv"].strip().lower() == "ja",
    })

print(json.dumps(hosts, indent=2))
```

```text
[
  {
    "name": "web01",
    "cpu": 8,
    "aktiv": true
  },
  {
    "name": "db01",
    "cpu": 16,
    "aktiv": false
  }
]
```

Das Beispiel macht aus den Textfeldern der CSV-Datei echte Typen: eine Zahl und einen Wahrheitswert. Gerade der Wahrheitswert ist tückisch: `bool("nein")` wäre `True`, weil jeder nichtleere Text wahr ist; man muss deshalb ausdrücklich vergleichen.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentation und Quellen zu diesen Formaten (abgerufen).** *CSV* (Python-Dokumentation, aktuelle Version): Jede gelesene Zeile ist eine Liste von Texten, es findet keine automatische Typumwandlung statt (Ausnahme `QUOTE_NONNUMERIC`); `DictReader` nimmt ohne Angabe von `fieldnames` die erste Zeile als Kopf; `DictWriter` verlangt `fieldnames` und schreibt die Kopfzeile mit `writeheader()`; Dateien sollen mit `newline=""` geöffnet werden, weil sonst eingebettete Zeilenumbrüche falsch gelesen und beim Schreiben unter `\r\n`-Systemen ein zusätzliches `\r` erzeugt wird; Standardtrennzeichen ist das Komma, `QUOTE_MINIMAL` die Standardquotierung. Die Seite erwähnt weder `utf-8-sig` noch Excel. Quelle: <https://docs.python.org/library/csv>. *PyYAML:* Die Dokumentation des Linters Ruff (Regel S506) beschreibt, dass `yaml.load` die Erzeugung beliebiger Python-Objekte erlaubt, die zur Ausführung beliebigen Codes benutzt werden können, und empfiehlt `yaml.safe_load`, das nur einfache Objekte wie Zahlen und Listen erzeugt. Quelle: <https://docs.astral.sh/ruff/rules/unsafe-yaml-load/>. Die Aussagen zu den Typumwandlungen von YAML 1.1 (`no`, `on`, `1.10`, `080`) wurden mit PyYAML 6.0.3 ausgeführt und bestätigt. *TOML:* PEP 680 führte `tomllib` in Python 3.11 ein; es kann nur lesen (`load` mit einer binär geöffneten Datei und `loads` mit einem Text) und löst bei ungültiger Eingabe `tomllib.TOMLDecodeError` aus. Das PEP begründet TOML mit seiner Rolle im Python-Packaging und nennt es einfacher und sicherer zu parsen als YAML. Quelle: <https://peps.pythondiscord.com/pep-0680> (Spiegel der PEP-Seite).

Die Wahl des Formats hängt vom Zweck ab. Die Tabelle fasst die Eigenschaften der vier Formate zusammen, die in diesem Modul vorkamen.

| **Format** | **Kommentare** | **Typen**                                      | **Standardbibliothek**    | **Typischer Einsatz**                        |
|:-----------|:---------------|:-----------------------------------------------|:--------------------------|:---------------------------------------------|
| JSON       | nein           | Zahl, Text, Wahrheitswert, null, Liste, Objekt | ja (`json`)            | Austausch zwischen Programmen, APIs          |
| YAML       | ja             | wie JSON, plus automatische Erkennung          | nein (`PyYAML`)          | Konfiguration, wenn das Werkzeug es verlangt |
| TOML       | ja             | wie JSON, plus Datum                           | ja, nur lesen (`tomllib`) | Konfiguration von Hand                       |
| CSV        | nein           | alles ist Text                                 | ja (`csv`)            | Tabellen, Exporte, Inventare                 |

> **Merke**
>
> `yaml.safe_load`, nie `yaml.load`; Texte, die wie Zahlen oder Wahrheitswerte aussehen, quoten. TOML liest die Standardbibliothek seit Python 3.11 (`tomllib`, Datei binär öffnen). Bei CSV: `newline=""`, Kodierung angeben (bei Dateien aus Excel oft `utf-8-sig`), alle Felder sind Text und müssen von Ihnen in Zahlen verwandelt werden.

> **Praxisbeispiel**
>
> Eine Excel-Liste der Mitarbeiter wird als CSV exportiert und soll in Benutzerkonten umgewandelt werden, ein Ansible-Playbook oder ein Docker-Compose-Projekt liegt als YAML vor und soll von einem Skript ausgewertet werden, und die Konfiguration Ihres eigenen Skripts soll von einem Kollegen ohne Programmierkenntnisse bearbeitet werden können, deshalb in TOML oder YAML mit Kommentaren. Diese Anwendungsfälle begegnen Ihnen im ersten Jahr als Administrator mit Python praktisch jede Woche. Aus personenbezogenen CSV-Dateien (Namen, Adressen) gehört nichts in Protokolle oder in das Repository; vgl. die Hinweise in Modul 6 und 10.

## 7.5 Skript mit Konfigurationsdatei

In dieser Lerneinheit setzen wir zusammen, was Sie in den Modulen 4 bis 7 gelernt haben: Dateien und Verzeichnisse (pathlib), Kommandozeile (argparse), Fehlerbehandlung mit Exit-Codes, ein Konfigurationsformat (TOML, JSON oder YAML) und ein Modul, das man auch importieren und testen kann. Das Ergebnis ist ein kleines, aber vollständiges Werkzeug, `dirreport`: Es listet die Dateien in einem oder mehreren Verzeichnissen auf, filtert nach Dateiendung und Mindestgröße und gibt die Liste als Text, JSON oder CSV aus. Die Verzeichnisse, die Endungen und das Ausgabeformat stehen in einer Konfigurationsdatei. Das ist der Meilenstein des Moduls: „Skript mit Config-Datei“.

Bevor wir den Code lesen, sollten wir uns das Prinzip ansehen, nach dem fast jedes Werkzeug mit Konfiguration arbeitet: Es gibt mehrere *Quellen* für denselben Wert, und es gibt eine feste Rangfolge, wer gewinnt. Ganz unten stehen die *Standardwerte* im Programm selbst; sie sorgen dafür, dass das Programm auch ohne jede Konfiguration läuft. Darüber liegt die *Konfigurationsdatei*, die für eine Maschine oder ein Projekt dauerhaft gilt. Darüber liegen *Umgebungsvariablen*, die sich pro Sitzung oder pro Dienst setzen lassen (zum Beispiel in einer systemd-Unit oder einem Container). Ganz oben stehen die *Kommandozeilen-Optionen*: Wer ausdrücklich etwas eintippt, meint es für diesen einen Aufruf. Diese Reihenfolge ist keine Vorschrift der Sprache, sondern eine verbreitete Konvention (die Rangfolge „Kommandozeile vor Umgebung vor Datei vor Standard“ finden Sie in vielen Unix-Werkzeugen). In unserem Skript setzen wir sie um, indem wir ein Dictionary schrittweise mit `update()` überschreiben: erst die Standardwerte, dann die Datei, dann die Umgebungsvariable, zuletzt die Optionen, die der Anwender angegeben hat.

Zwei Entwurfsentscheidungen verdienen Beachtung. Erstens *validiert* das Skript die zusammengebaute Konfiguration, bevor es irgendetwas tut. Eine Konfiguration von Hand zu pflegen heißt, dass Tippfehler passieren: ein Schlüssel `verzeichnis` statt `verzeichnisse`, ein Text statt einer Liste, ein unbekanntes Format. Ohne Prüfung würde das Programm den falschen Schlüssel stillschweigend ignorieren und scheinbar „funktionieren“, nur eben mit den Standardwerten. Mit Prüfung bekommt der Anwender sofort eine klare Meldung, welcher Schlüssel falsch ist. Zweitens trennt das Skript *Logik* und *Ein-/Ausgabe*: Die Funktionen liefern Werte zurück oder lösen Ausnahmen aus; nur `main()` gibt Meldungen aus und übersetzt die Ausnahmen in Exit-Codes (10 für Konfigurationsfehler, 11 für Datenfehler, wie in Modul 6; argparse selbst beendet sich bei falschen Optionen mit 2). So lassen sich die Funktionen einzeln testen.

Das Skript ist etwa hundert Zeilen lang. Lesen Sie es zunächst als Ganzes durch; die wichtigen Stellen erläutern wir danach. Wir speichern es als `dirreport.py`. Die YAML-Unterstützung ist optional: Das Modul `yaml` wird erst importiert, wenn tatsächlich eine `.yaml`-Datei gelesen wird, sodass das Skript für TOML und JSON ohne Zusatzpaket auskommt.

```python
#!/usr/bin/env python3
"""dirreport - Dateien in Verzeichnissen auflisten, gesteuert per Konfigurationsdatei."""
import argparse
import copy
import csv
import json
import os
import sys
import tomllib
from pathlib import Path

STANDARD = {
    "verzeichnisse": [],
    "endungen": [],
    "format": "text",
    "mindestgroesse": 0,
}
FORMATE = ("text", "json", "csv")


class ConfigFehler(Exception):
    """Die Konfiguration ist unbrauchbar (Exit-Code 10)."""


class DatenFehler(Exception):
    """Ein Verzeichnis lässt sich nicht auswerten (Exit-Code 11)."""


def lade_datei(pfad):
    pfad = Path(pfad)
    try:
        if pfad.suffix == ".json":
            return json.loads(pfad.read_text(encoding="utf-8"))
        if pfad.suffix == ".toml":
            with open(pfad, "rb") as f:
                return tomllib.load(f)
        if pfad.suffix in (".yaml", ".yml"):
            import yaml
            return yaml.safe_load(pfad.read_text(encoding="utf-8")) or {}
    except OSError as fehler:
        raise ConfigFehler(f"{pfad}: {fehler.strerror}") from fehler
    except (ValueError, tomllib.TOMLDecodeError) as fehler:
        raise ConfigFehler(f"{pfad}: ungültiger Inhalt ({fehler})") from fehler
    raise ConfigFehler(f"{pfad}: unbekanntes Dateiformat {pfad.suffix!r}")


def validiere(konfig):
    if not isinstance(konfig, dict):
        raise ConfigFehler("Die Konfiguration muss eine Tabelle aus Schlüsseln sein.")
    unbekannt = sorted(set(konfig) - set(STANDARD))
    if unbekannt:
        raise ConfigFehler("Unbekannte Schlüssel: " + ", ".join(unbekannt))
    if konfig["format"] not in FORMATE:
        raise ConfigFehler(f"format muss eines von {FORMATE} sein, nicht {konfig['format']!r}")
    for schluessel in ("verzeichnisse", "endungen"):
        liste = konfig[schluessel]
        if not isinstance(liste, list) or not all(isinstance(x, str) for x in liste):
            raise ConfigFehler(f"{schluessel} muss eine Liste von Texten sein.")
    groesse = konfig["mindestgroesse"]
    if isinstance(groesse, bool) or not isinstance(groesse, int) or groesse < 0:
        raise ConfigFehler("mindestgroesse muss eine ganze Zahl >= 0 sein.")
    return konfig


def baue_konfig(datei, umgebung, optionen):
    konfig = copy.deepcopy(STANDARD)
    if datei is not None:
        konfig.update(lade_datei(datei))
    if "DIRREPORT_FORMAT" in umgebung:
        konfig["format"] = umgebung["DIRREPORT_FORMAT"]
    konfig.update({k: v for k, v in optionen.items() if v is not None})
    return validiere(konfig)


def sammle(verzeichnis, endungen, mindestgroesse):
    wurzel = Path(verzeichnis)
    if not wurzel.is_dir():
        raise DatenFehler(f"{verzeichnis}: kein Verzeichnis")
    zeilen = []
    for pfad in sorted(wurzel.iterdir()):
        if not pfad.is_file():
            continue
        if endungen and pfad.suffix.lstrip(".") not in endungen:
            continue
        groesse = pfad.stat().st_size
        if groesse >= mindestgroesse:
            zeilen.append({"datei": pfad.name, "groesse": groesse})
    return zeilen


def gib_aus(zeilen, format, ziel):
    if format == "json":
        json.dump(zeilen, ziel, indent=2, ensure_ascii=False)
        ziel.write("\n")
    elif format == "csv":
        schreiber = csv.DictWriter(ziel, fieldnames=["datei", "groesse"], lineterminator="\n")
        schreiber.writeheader()
        schreiber.writerows(zeilen)
    else:
        for z in zeilen:
            ziel.write(f"{z['groesse']:>8}  {z['datei']}\n")


def main(argv=None, umgebung=None):
    umgebung = os.environ if umgebung is None else umgebung
    parser = argparse.ArgumentParser(description="Dateien in Verzeichnissen auflisten.")
    parser.add_argument("verzeichnisse", nargs="*", help="ersetzt die Liste aus der Konfiguration")
    parser.add_argument("--config", help="Konfigurationsdatei (.toml, .json, .yaml)")
    parser.add_argument("--format", choices=FORMATE, dest="format")
    parser.add_argument("--mindestgroesse", type=int)
    args = parser.parse_args(argv)
    try:
        konfig = baue_konfig(args.config, umgebung, {
            "format": args.format,
            "mindestgroesse": args.mindestgroesse,
            "verzeichnisse": args.verzeichnisse or None,
        })
        zeilen = []
        for v in konfig["verzeichnisse"]:
            zeilen += sammle(v, konfig["endungen"], konfig["mindestgroesse"])
    except ConfigFehler as fehler:
        print(f"Konfigurationsfehler: {fehler}", file=sys.stderr)
        return 10
    except DatenFehler as fehler:
        print(f"Datenfehler: {fehler}", file=sys.stderr)
        return 11
    gib_aus(zeilen, konfig["format"], sys.stdout)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Die Funktion `lade_datei` entscheidet anhand der Dateiendung, welcher Leser zuständig ist, und übersetzt jeden Lesefehler in unsere eigene Ausnahme `ConfigFehler`. Dabei sind zwei Dinge zu beachten. Die Reihenfolge der `except`-Zweige: `OSError` (Datei fehlt, keine Berechtigung) wird vor `ValueError` geprüft, und `json.JSONDecodeError` sowie `tomllib.TOMLDecodeError` sind beide Unterklassen von `ValueError`; deshalb fängt ein Zweig beide ab (bei YAML gilt das nicht, dort hat `yaml.YAMLError` eine andere Basisklasse, sodass ein kaputtes YAML als unbehandelter Fehler durchschlüge; das ist Übungsaufgabe 7). Die Anweisung `raise ... from fehler` behält die ursprüngliche Ausnahme als Ursache bei (Modul 6). Und `open(pfad, "rb")` öffnet die TOML-Datei binär, wie es `tomllib` verlangt.

Die Funktion `validiere` prüft in vier Schritten: ob die Konfiguration überhaupt ein Dictionary ist, ob unbekannte Schlüssel vorkommen (das ist die Mengendifferenz `set(konfig) - set(STANDARD)`), ob das Format eines der erlaubten ist, und ob die Werte den richtigen Typ haben. Eine Besonderheit steht in der Zeile mit `isinstance(groesse, bool)`: In Python ist `True` ein Sonderfall von `int`, sodass `isinstance(True, int)` wahr ergibt. Eine Konfiguration `mindestgroesse = true` würde ohne die Extra-Prüfung als Zahl 1 durchgehen.

`baue_konfig` setzt die Rangfolge um. Wichtig ist die erste Zeile: `copy.deepcopy(STANDARD)` erzeugt eine vollständig unabhängige Kopie, damit spätere Änderungen die Standardwerte nicht verändern; warum das nötig ist, zeigt die Fehlerbetrachtung weiter unten. Danach folgen die Datei, die Umgebungsvariable und schließlich die Optionen, wobei `{k: v for k, v in optionen.items() if v is not None}` nur die Optionen übernimmt, die der Anwender wirklich angegeben hat. Das Skript übergibt die Umgebung als Parameter (`umgebung`), statt direkt `os.environ` zu lesen. Dadurch kann man in Tests ein eigenes Dictionary mitgeben, ohne die echte Umgebung des Rechners zu verändern.

In `main` schließlich baut argparse die Befehlszeile (Modul 5): `nargs="*"` erlaubt beliebig viele Verzeichnisse, auch keines, und `choices=FORMATE` lässt nur gültige Formate zu. Die Optionen, die nicht angegeben wurden, haben den Wert `None`; dadurch erkennt `baue_konfig`, dass sie die Datei nicht überschreiben sollen. Der Ausdruck `args.verzeichnisse or None` macht aus der leeren Liste ein `None`, damit die Verzeichnisse aus der Konfigurationsdatei erhalten bleiben. Dass `main` den Exit-Code als Rückgabewert liefert und erst die letzte Zeile `sys.exit(main())` daraus das Prozessende macht, ist eine bewährte Konstruktion: Man kann `main` in Tests aufrufen, ohne dass sich der Testlauf beendet.

Nun probieren wir das Werkzeug aus. Das folgende Demo-Skript importiert `dirreport` und ruft `main` mit verschiedenen Argumenten auf, so als wären es Aufrufe auf der Kommandozeile. Es legt dazu in einem temporären Verzeichnis ein paar Dateien und Konfigurationen an. Beachten Sie die sechs Fälle: eine Konfiguration aus der Datei, dieselbe mit überschriebenen Optionen, die Umgebungsvariable, ein Tippfehler im Schlüssel, eine syntaktisch kaputte Datei und ein Verzeichnis, das es nicht gibt. Die Fehlermeldungen gehen auf den Fehlerkanal; damit sie in der Beispielausgabe erscheinen, leiten wir ihn in den Puffer um.

```python
import contextlib
import io
import os
import sys
import tempfile
from pathlib import Path

import dirreport

def lauf(*argv, umgebung=None):
    vorsatz = "".join(f"{k}={v} " for k, v in (umgebung or {}).items())
    puffer = io.StringIO()
    with contextlib.redirect_stdout(puffer), contextlib.redirect_stderr(puffer):
        code = dirreport.main(list(argv), umgebung if umgebung is not None else {})
    print(f"$ {vorsatz}dirreport {' '.join(argv)}")
    print(puffer.getvalue(), end="")
    print(f"[Exit-Code {code}]")

with tempfile.TemporaryDirectory() as tmp:
    os.chdir(tmp)
    Path("logs").mkdir()
    Path("logs/a.log").write_text("x" * 2000)
    Path("logs/b.log").write_text("x" * 50)
    Path("logs/notiz.txt").write_text("x" * 300)
    Path("config.toml").write_text('verzeichnisse = ["logs"]\nendungen = ["log"]\nformat = "csv"\n')
    Path("tippfehler.toml").write_text('verzeichnis = ["logs"]\n')
    Path("kaputt.toml").write_text('endungen = ["log"\n')

    lauf("--config", "config.toml")
    lauf("--config", "config.toml", "--format", "text", "--mindestgroesse", "100")
    lauf("--config", "config.toml", umgebung={"DIRREPORT_FORMAT": "json"})
    lauf("--config", "tippfehler.toml")
    lauf("--config", "kaputt.toml")
    lauf("gibt-es-nicht")
```

```text
$ dirreport --config config.toml
datei,groesse
a.log,2000
b.log,50
[Exit-Code 0]
$ dirreport --config config.toml --format text --mindestgroesse 100
    2000  a.log
[Exit-Code 0]
$ DIRREPORT_FORMAT=json dirreport --config config.toml
[
  {
    "datei": "a.log",
    "groesse": 2000
  },
  {
    "datei": "b.log",
    "groesse": 50
  }
]
[Exit-Code 0]
$ dirreport --config tippfehler.toml
Konfigurationsfehler: Unbekannte Schlüssel: verzeichnis
[Exit-Code 10]
$ dirreport --config kaputt.toml
Konfigurationsfehler: kaputt.toml: ungültiger Inhalt (Unclosed array (at end of document))
[Exit-Code 10]
$ dirreport gibt-es-nicht
Datenfehler: gibt-es-nicht: kein Verzeichnis
[Exit-Code 11]
```

Im ersten Fall liest das Skript `config.toml` und liefert nur die beiden `.log`-Dateien als CSV; `notiz.txt` wird wegen der Endungsliste ausgelassen. Im zweiten Fall haben die Kommandozeilen-Optionen Vorrang vor der Datei: Das Format ist jetzt Text, und `b.log` mit 50 Bytes fällt unter die Mindestgröße. Im dritten Fall steht in der Datei weiterhin CSV, aber die Umgebungsvariable `DIRREPORT_FORMAT` macht JSON daraus. Der vierte Fall zeigt den Nutzen der Validierung: Der Tippfehler wird beim Namen genannt. Im fünften Fall stammt die Meldung direkt vom TOML-Parser (hier: eine nicht geschlossene Liste); an den Parser-Text hängen wir den Dateinamen. Der sechste Fall liefert den Exit-Code 11, weil das Verzeichnis fehlt. Dass die Fehlermeldung bei einer fehlenden Konfigurationsdatei vom Betriebssystem stammt (`strerror`), hat einen Haken: Sie ist je nach System und Spracheinstellung verschieden. Für Menschen ist das in Ordnung, für Tests eher nicht; testen Sie besser auf den Exit-Code.

Mit einer Konfigurationsdatei im Verzeichnis lässt sich das Skript zu einem Dienst ausbauen: Ein Cron-Eintrag oder ein systemd-Timer (Modul 9) ruft `dirreport --config /etc/dirreport.toml` täglich auf und legt die CSV-Ausgabe ab. Wer die Konfiguration ändern will, muss keinen Python-Code anfassen; wer neue Optionen braucht, ergänzt `STANDARD`, `validiere` und die Kommandozeile an genau drei Stellen.

**Typische Fehler.** Der erste Fehler betrifft die Standardwerte. Wer das Dictionary mit den Standardwerten nicht kopiert, verändert beim Zusammenbauen der Konfiguration die Standardwerte selbst. Das fällt in einem einzelnen Aufruf nicht auf, aber sobald die Funktion in einem Test oder in einem langlebigen Prozess ein zweites Mal aufgerufen wird, steckt der Wert des ersten Aufrufs schon in den Standardwerten. Auch die flache Kopie `dict(STANDARD)` schützt nur die oberste Ebene: Die Listen darin sind weiterhin dieselben Objekte, und ein `append` ändert dann die Standardwerte. Deshalb verwendet unser Skript `copy.deepcopy`.

```python
import copy

def falsch(zusatz):
    konfig = GLOBAL                    # keine Kopie: dasselbe Objekt
    konfig.update(zusatz)
    return konfig

def halbrichtig():
    konfig = dict(GLOBAL)              # flache Kopie
    konfig["verzeichnisse"].append("/tmp")
    return konfig

def richtig():
    konfig = copy.deepcopy(GLOBAL)     # vollständig unabhängig
    konfig["verzeichnisse"].append("/tmp")
    return konfig

GLOBAL = {"verzeichnisse": [], "format": "text"}
falsch({"format": "json"})
print("nach falsch():     ", GLOBAL)
GLOBAL = {"verzeichnisse": [], "format": "text"}
halbrichtig()
print("nach halbrichtig():", GLOBAL)
GLOBAL = {"verzeichnisse": [], "format": "text"}
richtig()
print("nach richtig():    ", GLOBAL)
```

```text
nach falsch():      {'verzeichnisse': [], 'format': 'json'}
nach halbrichtig(): {'verzeichnisse': ['/tmp'], 'format': 'text'}
nach richtig():     {'verzeichnisse': [], 'format': 'text'}
```

Nur im dritten Fall bleiben die Standardwerte unverändert. Das Beispiel benutzt eine globale Variable `GLOBAL`, um das Problem sichtbar zu machen; im echten Skript heißt sie `STANDARD`.

Der zweite typische Fehler ist, Konfigurationsschlüssel nicht zu prüfen. Ohne die Mengendifferenz in `validiere` würde ein vertippter Schlüssel ignoriert, und das Skript liefe mit Standardwerten, ohne dass jemand es merkt; der Betreiber glaubt, er habe ein Verzeichnis konfiguriert, aber es wird nie untersucht. Der dritte Fehler ist, einen Wahrheitswert als Zahl durchgehen zu lassen (wegen `isinstance(True, int)`), der vierte, YAML mit `yaml.load` zu lesen (Sicherheitsrisiko, siehe 7.4), der fünfte, Pfade aus der Konfiguration ungeprüft zu verwenden. Zu Letzterem eine Anmerkung: Unser Skript liest ausschließlich, es löscht oder verändert nichts. Sobald ein Werkzeug schreibt oder löscht, sollten Verzeichnisse aus der Konfiguration zusätzlich auf erlaubte Wurzeln geprüft werden. Und wenn die Konfigurationsdatei Zugangsdaten enthält, gehört sie nicht ins Versionsverwaltungs-Repository und braucht restriktive Dateirechte (Modul 4).

> **Merke**
>
> Rangfolge der Quellen: Standardwerte, Konfigurationsdatei, Umgebungsvariable, Kommandozeile. Konfiguration immer *validieren*, unbekannte Schlüssel ablehnen, Fehler als eigene Ausnahme melden und in `main` in Exit-Codes übersetzen. Standardwerte mit `copy.deepcopy` kopieren. Ausnahmen werfen die Funktionen, Meldungen gibt nur `main` aus.

> **Praxisbeispiel**
>
> Fast jedes Administrationswerkzeug, das Sie im Betrieb selbst schreiben, ist von dieser Art: ein Skript mit festen Standardwerten, das auf jedem Server mit einer eigenen kleinen Konfigurationsdatei läuft. Das Backup-Skript aus Modul 6 könnte seine Quellverzeichnisse, das Ziel und die Anzahl aufzubewahrender Sicherungen aus einer TOML-Datei beziehen; das Überwachungsskript aus Modul 5 die Liste der Dienste und die Schwellwerte. Der Vorteil: Das Skript ist für alle Server gleich, nur die Konfiguration unterscheidet sich.

## Übungsaufgaben zu Modul 07

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Arbeiten Sie bei den Programmieraufgaben in einem temporären Verzeichnis und in einer virtuellen Umgebung, damit Ihr System sauber bleibt.

**Aufgabe 1.** Was ist der Unterschied zwischen Modul, Paket und Skript? Was bewirkt die Abfrage `if __name__ == "__main__":`, und was enthält `__name__`, wenn eine Datei importiert wird? Warum darf Ihre eigene Datei nicht `json.py` oder `random.py` heißen? Was ist ein zirkulärer Import, und wie vermeidet man ihn?

**Aufgabe 2.** Warum arbeitet man in einer virtuellen Umgebung, und warum meldet `pip install` unter Debian und Ubuntu ohne eine solche den Fehler `externally-managed-environment`? Warum schreibt man `python -m pip` statt `pip`? Was bedeuten die Schreibweisen `==1.2.3` und `~=1.2` in einer `requirements.txt`, und warum gehört das Verzeichnis `.venv` nicht in die Versionsverwaltung, die `requirements.txt` aber schon?

**Aufgabe 3.** Wann wählen Sie JSON, YAML, TOML oder CSV? Warum ist `yaml.load` gefährlich, und was ist das „Norwegen-Problem“? Welche drei Regeln gelten beim Umgang mit der Standardbibliothek-Funktion `csv` (Öffnen der Datei, Typen der Felder, Kodierung)? Warum liefert `json.dumps` für ein Dictionary mit einem Datum einen Fehler, und wie löst man das?

**Aufgabe 4.** Finden Sie mindestens sechs Fehler im folgenden Programm und erklären Sie jeden.

```python
import csv
import json
import yaml

konfig = yaml.load(open("konfig.yaml"))

with open("hosts.csv") as f:
    hosts = list(csv.DictReader(f))

gesamt = sum(h["cpu"] for h in hosts)
json.dump(hosts, "ausgabe.json")
print("CPUs insgesamt:", gesamt)
```

**Aufgabe 5.** Eine Tabellenkalkulation exportiert Preise als CSV mit Semikolon, Dezimalkomma und einem BOM am Dateianfang. Schreiben Sie eine Funktion `lies_preise(daten)`, die die Bytes der Datei entgegennimmt und eine Liste von Dictionaries mit `name` (Text), `preis` (Zahl) und `menge` (Ganzzahl) liefert, sowie eine Funktion `speichere_json(daten, pfad)`, die das Ergebnis atomar (über eine `.part`-Datei) als lesbares JSON mit erhaltenen Umlauten schreibt. Geben Sie den Gesamtbetrag auf zwei Nachkommastellen aus.

**Aufgabe 6.** Legen Sie in einem temporären Verzeichnis ein Paket `zahlen` an, das aus `__init__.py`, `rechnen.py` und `__main__.py` besteht. Das Modul `rechnen` stellt die Funktion `addiere(a, b)` bereit; der Aufruf `python -m zahlen 3 4` soll `7` ausgeben. Rufen Sie das Paket mit `subprocess` auf, und zeigen Sie, dass es sich auch importieren lässt, ohne dass etwas ausgegeben wird.

**Aufgabe 7.** Schreiben Sie eine Funktion `lade_yaml(text, name)`, die einen YAML-Text sicher liest und jeden Lesefehler in eine eigene Ausnahme `ConfigFehler` mit Dateiname, Zeile und Spalte übersetzt. Sie soll zusätzlich einen leeren Text als leere Konfiguration behandeln, eine Liste oder Zahl auf der obersten Ebene ablehnen und beim Schlüssel `laender` erkennen, wenn ein Eintrag als Wahrheitswert gelesen wurde (das Norwegen-Problem), und dann einen Hinweis zum Quoten ausgeben.

**Aufgabe 8.** Erweitern Sie das Werkzeug `dirreport` aus Lerneinheit 7.5 um den Konfigurationsschlüssel `rekursiv` (Wahrheitswert, Standard `False`). Ist er gesetzt, sollen auch Unterverzeichnisse durchsucht werden, und der Dateiname wird als relativer Pfad ausgegeben. Schreiben Sie dazu eine Prüffunktion, die einen Nicht-Wahrheitswert mit `ConfigFehler` ablehnt, und eine Sammelfunktion, die `dirreport.DatenFehler` für ein fehlendes Verzeichnis auslöst. Testen Sie beides mit einem kleinen Verzeichnisbaum.

## Lösungen zu Modul 07

**Aufgabe 1.** Ein *Modul* ist eine einzelne `.py`-Datei, die man importieren kann. Ein *Paket* ist ein Verzeichnis mit Modulen (üblicherweise mit einer Datei `__init__.py`). Ein *Skript* ist eine Datei, die man direkt ausführt; technisch ist es dasselbe wie ein Modul, nur die Art des Starts unterscheidet sich. Beim direkten Start ist `__name__` gleich `"__main__"`, beim Import ist es der Modulname. Die Abfrage trennt daher den Code, der nur beim Start laufen soll (Befehlszeile, Ausgaben), von den Funktionen, die man ohne Nebenwirkungen importieren können muss. Eine eigene Datei `json.py` oder `random.py` im Skriptverzeichnis überdeckt das gleichnamige Standardmodul, weil das Verzeichnis des Skripts zuerst durchsucht wird; später schlagen Importe des Standardmoduls mit rätselhaften Fehlermeldungen fehl. Ein zirkulärer Import entsteht, wenn Modul A Modul B importiert und B wiederum A; dann ist eines der beiden beim Import noch unvollständig. Man vermeidet ihn, indem man gemeinsam genutzte Teile in ein drittes Modul auslagert oder den Import in die Funktion verschiebt, die ihn braucht.

**Aufgabe 2.** Eine virtuelle Umgebung gibt jedem Projekt seine eigenen Pakete in eigenen Versionen; so stören sich zwei Projekte nicht, und die Pakete des Systems, auf die Systemwerkzeuge angewiesen sind, bleiben unberührt. Debian und Ubuntu markieren die systemweite Python-Installation nach PEP 668 als „extern verwaltet“; `pip` verweigert dort das Installieren und verweist auf das Paketsystem (`apt`) oder eine virtuelle Umgebung. Mit `python -m pip` läuft garantiert das `pip` des gerade verwendeten Interpreters; ein loses `pip` im Pfad könnte zu einem anderen Python gehören. `==1.2.3` verlangt genau diese Version; `~=1.2` bedeutet „kompatibel“, also mindestens 1.2 und kleiner als 2.0. Das Verzeichnis `.venv` ist maschinen- und pfadabhängig (es lässt sich nicht verschieben, siehe 7.2) und aus der `requirements.txt` jederzeit neu erzeugbar; die `requirements.txt` dagegen beschreibt, wie man die Umgebung neu aufbaut, und gehört daher in die Versionsverwaltung.

**Aufgabe 3.** JSON für den Austausch zwischen Programmen und APIs, YAML dort, wo ein Werkzeug es verlangt (Docker-Compose, Ansible), TOML für eine von Hand gepflegte Konfiguration (Standardbibliothek seit 3.11, aber nur lesend), CSV für flache Tabellen und Exporte. `yaml.load` kann beliebige Python-Objekte erzeugen, darunter Funktionsaufrufe, und führt damit fremden Code aus; `safe_load` erlaubt nur einfache Typen. Das Norwegen-Problem: In YAML 1.1 ist `no` ein Wahrheitswert, deshalb wird die Länderliste `[de, no, at]` zu `['de', False, 'at']`; Abhilfe sind Anführungszeichen. Bei `csv`: Datei mit `newline=""` öffnen, jedes Feld ist ein Text (Zahlen und Wahrheitswerte selbst umwandeln), die Kodierung ausdrücklich angeben (bei Excel-Dateien oft `utf-8-sig`). `json.dumps` kennt keine Datumstypen; mit dem Parameter `default=str` (oder einer eigenen Funktion, die `isoformat()` liefert) wandelt man unbekannte Typen um.

**Aufgabe 4.** (1) `yaml.load(...)` ohne `Loader`-Argument löst bei PyYAML 6 einen `TypeError` aus (und war früher unsicher); richtig ist `yaml.safe_load`. (2) `open("konfig.yaml")` wird nie geschlossen; besser mit `with`, und mit `encoding="utf-8"`. (3) Die Datei `hosts.csv` wird ohne `newline=""` und ohne Kodierung geöffnet. (4) `sum(h["cpu"] ...)` addiert Texte: Der Leser liefert alle Felder als Text, daher entsteht ein `TypeError` (`int` plus `str`); die Felder müssen mit `int()` umgewandelt werden. (5) `json.dump(hosts, "ausgabe.json")` erwartet ein Dateiobjekt, keinen Namen; richtig ist `json.dump(hosts, f)` innerhalb eines `with open(..., "w")`-Blocks (oder `json.dumps` und `write_text`). (6) Das Ergebnis der YAML-Konfiguration (`konfig`) wird nie verwendet. (7) Es gibt keinerlei Fehlerbehandlung für fehlende oder kaputte Dateien, und beim Schreiben der Ausgabe kann eine halbfertige Datei zurückbleiben (atomar schreiben, siehe 7.3).

**Aufgabe 5.**

Die Daten kommen als Bytes; wir dekodieren sie mit `utf-8-sig`, das das BOM entfernt. Die Zahlen werden einzeln umgewandelt, das Dezimalkomma wird durch einen Punkt ersetzt. Beim Schreiben erzeugen wir erst eine `.part`-Datei und benennen sie mit `os.replace` um, damit nie eine halbfertige Ausgabe unter dem endgültigen Namen liegt.

```python
import csv
import io
import json
import os
import tempfile
from pathlib import Path

def lies_preise(daten):
    text = daten.decode("utf-8-sig")
    zeilen = []
    for zeile in csv.DictReader(io.StringIO(text, newline=""), delimiter=";"):
        zeilen.append({
            "name": zeile["Name"],
            "preis": float(zeile["Preis"].replace(",", ".")),
            "menge": int(zeile["Menge"]),
        })
    return zeilen

def speichere_json(daten, pfad):
    pfad = Path(pfad)
    teil = pfad.with_name(pfad.name + ".part")
    teil.write_text(json.dumps(daten, indent=2, ensure_ascii=False) + "\n", encoding="utf-8")
    os.replace(teil, pfad)

export = "Name;Preis;Menge\nFestplatte;89,90;3\nKabel Süd;4,50;10\n".encode("utf-8-sig")
preise = lies_preise(export)
gesamt = sum(p["preis"] * p["menge"] for p in preise)

with tempfile.TemporaryDirectory() as tmp:
    ziel = Path(tmp) / "preise.json"
    speichere_json(preise, ziel)
    print(ziel.read_text(encoding="utf-8"), end="")
    print(sorted(p.name for p in Path(tmp).iterdir()))
print(f"Gesamtbetrag: {gesamt:.2f}")
```

```text
[
  {
    "name": "Festplatte",
    "preis": 89.9,
    "menge": 3
  },
  {
    "name": "Kabel Süd",
    "preis": 4.5,
    "menge": 10
  }
]
['preise.json']
Gesamtbetrag: 314.70
```

Im Verzeichnis liegt am Ende nur die endgültige Datei, die Hilfsdatei `preise.json.part` ist durch `os.replace` verschwunden. Für Geldbeträge ist `float` streng genommen ungeeignet (Rundungsfehler); in einer echten Abrechnung verwendet man `decimal.Decimal`. Für diese Übung genügt die Gleitkommazahl, weil wir nur auf zwei Stellen gerundet ausgeben.

**Aufgabe 6.**

Das Paket besteht aus drei Dateien. In `rechnen.py` steht nur die Funktion, ohne Ausgabe. Die Datei `__main__.py` wird ausgeführt, wenn man das Paket mit `python -m zahlen` startet; sie liest die Argumente und ruft die Funktion auf. Die Datei `__init__.py` darf leer sein; sie kennzeichnet das Verzeichnis als Paket und ist hier leer, weil wir sie nicht brauchen. Der Test ruft den Interpreter mit `subprocess` im temporären Verzeichnis auf, damit dort nach dem Paket gesucht wird.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    paket = Path(tmp) / "zahlen"
    paket.mkdir()
    (paket / "__init__.py").write_text("", encoding="utf-8")
    (paket / "rechnen.py").write_text(
        "def addiere(a, b):\n    return a + b\n", encoding="utf-8")
    (paket / "__main__.py").write_text(
        "import sys\n"
        "from zahlen.rechnen import addiere\n"
        "print(addiere(int(sys.argv[1]), int(sys.argv[2])))\n",
        encoding="utf-8")

    lauf = subprocess.run([sys.executable, "-m", "zahlen", "3", "4"],
                          cwd=tmp, capture_output=True, text=True)
    print("python -m zahlen 3 4 ->", lauf.stdout.strip(), "| Exit-Code", lauf.returncode)

    import_test = subprocess.run(
        [sys.executable, "-c", "import zahlen.rechnen as r; print(r.addiere(1, 2))"],
        cwd=tmp, capture_output=True, text=True)
    print("Import ->", import_test.stdout.strip(), "| Exit-Code", import_test.returncode)
```

```text
python -m zahlen 3 4 -> 7 | Exit-Code 0
Import -> 3 | Exit-Code 0
```

Beim ersten Aufruf läuft `__main__.py` und gibt `7` aus. Beim Import von `zahlen.rechnen` läuft `__main__.py` dagegen nicht, es erscheint nur die eine Ausgabe des Testprogramms. Ein häufiger Fehler ist der Aufruf aus dem falschen Verzeichnis: Dann findet Python das Paket nicht (`No module named zahlen`), weil das aktuelle Verzeichnis nicht im Suchpfad liegt. Eine Alternative zum Verzeichniswechsel ist eine Installation des Pakets in der virtuellen Umgebung, was in Modul 13 behandelt wird.

**Aufgabe 7.**

Die Funktion fängt `yaml.YAMLError` ab und liest die Position aus dem Attribut `problem_mark`, das nicht jede YAML-Ausnahme besitzt; deshalb `getattr(..., None)`. Für das Norwegen-Problem durchsuchen wir die Liste `laender` nach Einträgen, die ein Wahrheitswert sind.

```python
import yaml

class ConfigFehler(Exception):
    pass

def lade_yaml(text, name="konfig.yaml"):
    try:
        daten = yaml.safe_load(text)
    except yaml.YAMLError as fehler:
        marke = getattr(fehler, "problem_mark", None)
        ort = f", Zeile {marke.line + 1}, Spalte {marke.column + 1}" if marke else ""
        grund = getattr(fehler, "problem", None) or str(fehler)
        raise ConfigFehler(f"{name}{ort}: {grund}") from fehler
    if daten is None:
        return {}
    if not isinstance(daten, dict):
        raise ConfigFehler(f"{name}: Auf der obersten Ebene muss eine Tabelle stehen.")
    laender = daten.get("laender", [])
    if isinstance(laender, list) and any(isinstance(x, bool) for x in laender):
        raise ConfigFehler(f"{name}: In 'laender' wurde ein Eintrag als Wahrheitswert "
                           "gelesen (no/yes/on/off?). Bitte in Anführungszeichen setzen.")
    return daten

proben = {
    "gut": "laender: ['de', 'no']\nport: 8080\n",
    "leer": "",
    "liste": "- a\n- b\n",
    "tab": "hosts:\n\t- web01\n",
    "norwegen": "laender: [de, no, at]\n",
}
for titel, text in proben.items():
    try:
        print(f"{titel:9}", lade_yaml(text))
    except ConfigFehler as fehler:
        print(f"{titel:9} ConfigFehler: {fehler}")
```

```text
gut       {'laender': ['de', 'no'], 'port': 8080}
leer      {}
liste     ConfigFehler: konfig.yaml: Auf der obersten Ebene muss eine Tabelle stehen.
tab       ConfigFehler: konfig.yaml, Zeile 2, Spalte 1: found character '\t' that cannot start any token
norwegen  ConfigFehler: konfig.yaml: In 'laender' wurde ein Eintrag als Wahrheitswert gelesen (no/yes/on/off?). Bitte in Anführungszeichen setzen.
```

Alle fünf Fälle werden unterschieden: Die gute Konfiguration kommt unverändert zurück, ein leerer Text ergibt eine leere Konfiguration (`safe_load` würde hier `None` liefern, mit dem das restliche Programm nichts anfangen kann), und die drei Fehler werden in einheitliche Meldungen mit Ort übersetzt. Der Aufrufer muss nur noch eine einzige Ausnahmeklasse behandeln, unabhängig davon, ob der Fehler aus YAML oder aus der eigenen Prüfung stammt.

**Aufgabe 8.**

Die Lösung importiert `dirreport` aus Lerneinheit 7.5 und ergänzt es, ohne die Datei zu verändern. Die Sammelfunktion verwendet `rglob("*")` statt `iterdir()`, durchsucht also auch Unterverzeichnisse, und gibt die Pfade relativ zur Wurzel aus (mit `as_posix()`, damit Schrägstriche verwendet werden). Die Prüffunktion nutzt dieselbe Ausnahme wie das Original, damit `main` sie unverändert in den Exit-Code 10 übersetzen würde.

```python
import tempfile
from pathlib import Path

import dirreport

def pruefe_rekursiv(konfig):
    wert = konfig.get("rekursiv", False)
    if not isinstance(wert, bool):
        raise dirreport.ConfigFehler(f"rekursiv muss true oder false sein, nicht {wert!r}")
    return wert

def sammle_rekursiv(verzeichnis, endungen, mindestgroesse):
    wurzel = Path(verzeichnis)
    if not wurzel.is_dir():
        raise dirreport.DatenFehler(f"{verzeichnis}: kein Verzeichnis")
    zeilen = []
    for pfad in sorted(wurzel.rglob("*")):
        if not pfad.is_file():
            continue
        if endungen and pfad.suffix.lstrip(".") not in endungen:
            continue
        groesse = pfad.stat().st_size
        if groesse >= mindestgroesse:
            zeilen.append({"datei": pfad.relative_to(wurzel).as_posix(), "groesse": groesse})
    return zeilen

with tempfile.TemporaryDirectory() as tmp:
    wurzel = Path(tmp)
    (wurzel / "unter").mkdir()
    (wurzel / "a.log").write_text("x" * 10)
    (wurzel / "unter" / "b.log").write_text("x" * 20)
    (wurzel / "unter" / "c.txt").write_text("x" * 30)

    for z in sammle_rekursiv(wurzel, ["log"], 0):
        print(z)
    print(len(sammle_rekursiv(wurzel, [], 25)), "Datei(en) mit mindestens 25 Bytes")

    for schlecht in ("ja", 1):
        try:
            pruefe_rekursiv({"rekursiv": schlecht})
        except dirreport.ConfigFehler as fehler:
            print("ConfigFehler:", fehler)
    print(pruefe_rekursiv({}), pruefe_rekursiv({"rekursiv": True}))

    try:
        sammle_rekursiv(wurzel / "fehlt", [], 0)
    except dirreport.DatenFehler as fehler:
        print("DatenFehler:", str(fehler).split(":", 1)[1].strip())
```

```text
{'datei': 'a.log', 'groesse': 10}
{'datei': 'unter/b.log', 'groesse': 20}
1 Datei(en) mit mindestens 25 Bytes
ConfigFehler: rekursiv muss true oder false sein, nicht 'ja'
ConfigFehler: rekursiv muss true oder false sein, nicht 1
False True
DatenFehler: kein Verzeichnis
```

Die Zahl `1` wird abgelehnt, obwohl `True == 1` gilt; entscheidend ist `isinstance(wert, bool)`. In einem vollständigen Ausbau würden Sie `rekursiv` außerdem in `STANDARD` aufnehmen, in `validiere` aufrufen und in `main` zwischen `sammle` und `sammle_rekursiv` wählen; dort würden Sie auch die Zeilen in das Format aus 7.5 weitergeben. Die Reihenfolge der Ausgabe hängt von der Sortierung der Pfade ab: `unter/b.log` steht nach `a.log`, weil beim Vergleich der Pfadteile `('a.log',)` vor `('unter', 'b.log')` kommt.
