# Modul 14 - pytest, Mocking, Type Hints, ruff

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (14.1 bis 14.5), danach folgen „Übungsaufgaben zu Modul 14“ (8 Aufgaben) und „Lösungen zu Modul 14“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 15.02.2027 bis Fr 19.02.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 3 (Annahme: Der Lehrplan nennt keine Priorität; die Module 13 bis 15 bilden die Phase „Qualität und Abschluss“)

**Meilenstein:** Tests für ein Projekt

**Lernziele.** Nach diesem Modul können Sie für ein Python-Projekt automatische Tests mit `pytest` schreiben und lesen, Testdaten mit Fixtures bereitstellen, Dateien und Umgebungsvariablen sauber in Tests behandeln, Abhängigkeiten von der Außenwelt (Betriebssystem, Kommandos, Netzwerk) mit Mocks ersetzen, Funktionen mit Typangaben versehen und mit `mypy` prüfen sowie den Quelltext mit `ruff` automatisch auf Fehler prüfen und einheitlich formatieren. Am Ende steht das Projekt aus Modul 13 mit einer Testsuite, die Sie jederzeit mit einem Befehl ausführen können und die Änderungen am Code absichert.

| **Nr.** | **Lerneinheit**                | **Datum**     | **Dauer** | **Prio** |
|:--------|:-------------------------------|:--------------|:----------|:---------|
| 14.1    | pytest Grundlagen              | Mo 15.02.2027 | 2 h       | 3        |
| 14.2    | Fixtures                       | Di 16.02.2027 | 2 h       | 3        |
| 14.3    | Mocking (subprocess, Netzwerk) | Mi 17.02.2027 | 2 h       | 3        |
| 14.4    | Type Hints                     | Do 18.02.2027 | 2 h       | 3        |
| 14.5    | Linter/Formatter (ruff) + Quiz | Fr 19.02.2027 | 2 h       | 3        |
|         | **Summe**                      |               | **10 h**  |          |

> **Merke**
>
> Dieses Modul setzt das Projekt `plattenwacht` aus Modul 13 voraus (Paket im `src`-Layout mit `pyproject.toml`, in einem Git-Repository). Sie brauchen Python 3.11 oder neuer und Internetzugang für die Installation der Werkzeuge. Getestet wurde mit Python 3.13.18, pytest 9.2.0, ruff 0.17.4, mypy 2.4.1 und Git 2.43.0. Alle Mitschnitte stammen aus echten Läufen; die Ausgaben der Werkzeuge sind englisch und hängen von der Version ab, ebenso die Prozentwerte der Plattenbelegung, die auf Ihrem Rechner anders ausfallen. Zeiten in Testläufen (etwa „in 0.02s“) sind hier auf eine feste Angabe gesetzt.

## 14.1 pytest Grundlagen

Software, die niemand testet, funktioniert so lange, bis jemand sie ändert. Bei einem Skript, das auf Servern Alarme auslöst, ist das besonders unangenehm: Wer die Schwelle von 80 auf 85 Prozent ändert und dabei versehentlich einen Vergleich umdreht, bemerkt es vielleicht erst, wenn eine Platte voll läuft, ohne dass jemand gewarnt wurde. Ein *automatischer Test* ist ein kleines Programm, das den Code ausführt und prüft, ob das Ergebnis der Erwartung entspricht. Hat man viele davon, kann man nach jeder Änderung in Sekunden feststellen, ob etwas kaputtgegangen ist. Man nennt das eine *Testsuite*, und das Wiederholen nach Änderungen *Regressionstest*: Es soll sich nichts *zurück*entwickeln.

Python bringt mit `unittest` ein Testwerkzeug in der Standardbibliothek mit, das aber viel Schreibarbeit verlangt (Klassen, spezielle Prüfmethoden). Die Alternative, die sich in der Praxis fast überall durchgesetzt hat, ist `pytest`. Ein Test ist bei `pytest` eine gewöhnliche Funktion, deren Name mit `test_` beginnt und die mit der normalen Python-Anweisung `assert` prüft. Schlägt eine Prüfung fehl, zeigt `pytest` die beteiligten Werte an; das Werkzeug schreibt dafür die `assert`-Anweisungen beim Laden um, sodass man keine Spezialmethoden wie `assertEqual` braucht.

**Einrichtung.** Wir arbeiten im Projekt aus Modul 13. In der `pyproject.toml` hatten wir die Gruppe `dev` vorgesehen, die `pytest`, `ruff` und `mypy` enthält. Wir legen eine neue virtuelle Umgebung an (eine Umgebung lässt sich nicht von einem Rechner auf einen anderen kopieren, deshalb erzeugt man sie jeweils neu) und installieren das Projekt editierbar zusammen mit den Entwicklungswerkzeugen. Die Schreibweise `".[dev]"` bedeutet „dieses Projekt, und zusätzlich die Gruppe dev“; die Anführungszeichen verhindern, dass die Shell die eckigen Klammern als Muster deutet.

```pycon
~/plattenwacht$ cd ~/plattenwacht
~/plattenwacht$ python3 -m venv .venv
~/plattenwacht$ . .venv/bin/activate
(.venv) ~/plattenwacht$ pip install -q -e ".[dev]"
(.venv) ~/plattenwacht$ pytest --version
pytest 9.2.0
(.venv) ~/plattenwacht$ ruff --version
ruff 0.17.4
(.venv) ~/plattenwacht$ mypy --version
mypy 2.4.1 (compiled: yes)
(.venv) ~/plattenwacht$ git status --short
```

Alle drei Werkzeuge sind installiert, und das Repository ist unverändert, denn `.venv` steht in der `.gitignore`. Die Tests legt man nach Konvention in ein Verzeichnis `tests` neben `src`. Damit `pytest` nicht das ganze Projekt durchsuchen muss, tragen wir das in der `pyproject.toml` ein. Der Abschnitt `[tool.pytest.ini_options]` nimmt die Einstellungen von `pytest` auf; Änderungen daran gelten sofort, ohne Neuinstallation.

```pycon
(.venv) ~/plattenwacht$ printf '\n[tool.pytest.ini_options]\ntestpaths = ["tests"]\n' >> pyproject.toml
(.venv) ~/plattenwacht$ tail -4 pyproject.toml
version = {attr = "plattenwacht.__version__"}

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Der erste Test.** Wir testen zunächst die Funktion `bewerte` aus `core.py`. Sie ist ein guter Anfang, weil sie nur rechnet: Sie bekommt einen Prozentwert und liefert eine Stufe, ohne etwas vom Rechner zu lesen. Solche *reinen Funktionen* sind am leichtesten zu testen. Wir schreiben drei Tests. Jeder besteht aus der Vorbereitung (hier trivial), dem Aufruf und der Prüfung mit `assert`. Den Namen wählt man so, dass er beschreibt, was erwartet wird.

**Datei** `tests/test_core.py`**:**

```python
from plattenwacht.core import bewerte


def test_wenig_belegung_ist_ok():
    assert bewerte(10) == "OK"


def test_ab_warnschwelle_gibt_warnung():
    assert bewerte(80) == "WARNUNG"


def test_ab_kritischer_schwelle_gibt_kritisch():
    assert bewerte(90) == "KRITISCH"
```

Die Datei beginnt mit `test_`, die Funktionen ebenfalls; daran erkennt `pytest` sie (die Regeln stehen im Kasten weiter unten). Wir starten die Tests mit dem Befehl `pytest`. Die Ausgabe beginnt mit einem Kopf (Betriebssystem, Versionen, Wurzelverzeichnis und Konfiguration), nennt dann die gefundenen Dateien mit je einem Punkt pro bestandenem Test und endet mit einer Zusammenfassung.

```pycon
(.venv) ~/plattenwacht$ pytest; echo "Exit-Code: $?"
============================= test session starts ==============================
platform linux -- Python 3.13.18, pytest-9.2.0, pluggy-1.7.0
rootdir: /home/max/plattenwacht
configfile: pyproject.toml
testpaths: tests
collected 3 items

tests/test_core.py ...                                                   [100%]

============================== 3 passed in 0.01s ===============================
Exit-Code: 0
```

Drei Punkte, drei bestandene Tests, Exit-Code 0. Der Exit-Code ist wichtig, denn so merken Automatisierungen (Abschnitt 14.5 und Modul 15), ob Tests bestanden haben. `pytest` liefert 0, wenn alle Tests bestanden wurden, 1, wenn mindestens einer fehlgeschlagen ist, und 5, wenn gar keine Tests gefunden wurden; weitere Codes stehen in der Dokumentation (nicht abgerufen). Mit der Option `-v` (*verbose*) erscheint jeder Test mit Namen, was bei vielen Tests hilft.

**Ein fehlgeschlagener Test.** Der Nutzen von Tests zeigt sich erst, wenn sie etwas finden. Wir bauen absichtlich einen falschen Test ein, der behauptet, bei genau 80 Prozent sei noch alles in Ordnung. Das widerspricht unserer Funktion, denn die Schwelle gilt *ab* 80.

**Datei** `tests/test_falsch.py`**:**

```python
from plattenwacht.core import bewerte


def test_80_prozent_sind_noch_ok():
    stufe = bewerte(80)
    assert stufe == "OK"
```

```pycon
(.venv) ~/plattenwacht$ pytest tests/test_falsch.py; echo "Exit-Code: $?"
============================= test session starts ==============================
platform linux -- Python 3.13.18, pytest-9.2.0, pluggy-1.7.0
rootdir: /home/max/plattenwacht
configfile: pyproject.toml
collected 1 item

tests/test_falsch.py F                                                   [100%]

=================================== FAILURES ===================================
_________________________ test_80_prozent_sind_noch_ok _________________________

    def test_80_prozent_sind_noch_ok():
        stufe = bewerte(80)
>       assert stufe == "OK"
E       AssertionError: assert 'WARNUNG' == 'OK'
E         
E         - OK
E         + WARNUNG

tests/test_falsch.py:6: AssertionError
=========================== short test summary info ============================
FAILED tests/test_falsch.py::test_80_prozent_sind_noch_ok - AssertionError: a...
============================== 1 failed in 0.01s ===============================
Exit-Code: 1
```

`pytest` zeigt die Funktion, in der der Fehler auftrat, die Zeile mit einem Pfeil (`>`) und markiert mit `E` die Fehlermeldung. Bemerkenswert ist die Zeile mit `assert 'WARNUNG' == 'OK'`: Das Werkzeug hat die beteiligten Werte selbst ermittelt und zeigt auch noch, was von der Funktion zurückgegeben wurde. Am Ende steht eine Kurzfassung der fehlgeschlagenen Tests (`FAILED`), und der Exit-Code ist 1. Hier liegt der Fehler im Test, nicht im Code; bei einem echten Fehler wäre es umgekehrt. Wichtig ist, dass man beim Lesen einer solchen Ausgabe von oben nach unten vorgeht: Welcher Test, welche Zeile, welcher Wert wich ab? Wir löschen die Datei wieder.

```pycon
(.venv) ~/plattenwacht$ rm tests/test_falsch.py
(.venv) ~/plattenwacht$ pytest -q
...                                                                      [100%]
3 passed in 0.01s
```

**Mehrere Fälle: parametrize.** Der dritte Test oben prüft einen Grenzwert, aber eine ordentliche Prüfung testet die Grenzen von beiden Seiten: knapp unter und genau auf der Schwelle. Wer dafür für jeden Wert eine eigene Funktion schreibt, produziert viel Wiederholung. `pytest` bietet dafür den *Dekorator* `@pytest.mark.parametrize`: Man gibt eine Liste von Wertepaaren an, und `pytest` führt dieselbe Testfunktion für jedes Paar aus, als wären es getrennte Tests. Ein Dekorator ist eine Zeile mit `@` vor einer Funktion, die deren Verhalten erweitert (Modul 3 ist darauf eingegangen). Das erste Argument von `parametrize` nennt die Parameternamen als Zeichenkette, das zweite die Fälle.

**Datei** `tests/test_core.py`**:**

```python
import pytest

from plattenwacht.core import belegung_prozent, bewerte


@pytest.mark.parametrize(
    "prozent, erwartet",
    [
        (0, "OK"),
        (79.9, "OK"),
        (80, "WARNUNG"),
        (89.99, "WARNUNG"),
        (90, "KRITISCH"),
        (100, "KRITISCH"),
    ],
)
def test_bewerte_mit_standardschwellen(prozent, erwartet):
    assert bewerte(prozent) == erwartet


def test_bewerte_mit_eigenen_schwellen():
    assert bewerte(60, warnung=50, kritisch=70) == "WARNUNG"
    assert bewerte(70, warnung=50, kritisch=70) == "KRITISCH"


def test_belegung_ist_ein_prozentwert():
    wert = belegung_prozent("/")
    assert 0 <= wert <= 100


def test_unbekannter_pfad_wirft_fehler():
    with pytest.raises(FileNotFoundError):
        belegung_prozent("/gibt/es/nicht")
```

Die Datei enthält jetzt vier Testfunktionen, davon eine parametrisiert. Der Test `test_belegung_ist_ein_prozentwert` prüft nicht auf einen bestimmten Wert, denn der hängt vom Rechner ab; er prüft eine Eigenschaft, die immer gelten muss: Ein Prozentwert liegt zwischen 0 und 100. Das ist ein wichtiges Muster, wenn die Ergebnisse nicht festgelegt sind. Der letzte Test verwendet `pytest.raises`: Er erwartet, dass der Code eine bestimmte Ausnahme auslöst. Wird die Ausnahme *nicht* ausgelöst, schlägt der Test fehl; wird eine andere ausgelöst, ebenfalls. Mit `-v` sehen wir, wie die Fälle einzeln erscheinen.

```pycon
(.venv) ~/plattenwacht$ pytest -v
============================= test session starts ==============================
platform linux -- Python 3.13.18, pytest-9.2.0, pluggy-1.7.0 -- /home/max/plattenwacht/.venv/bin/python3
cachedir: .pytest_cache
rootdir: /home/max/plattenwacht
configfile: pyproject.toml
testpaths: tests
collecting ... collected 9 items

tests/test_core.py::test_bewerte_mit_standardschwellen[0-OK] PASSED      [ 11%]
tests/test_core.py::test_bewerte_mit_standardschwellen[79.9-OK] PASSED   [ 22%]
tests/test_core.py::test_bewerte_mit_standardschwellen[80-WARNUNG] PASSED [ 33%]
tests/test_core.py::test_bewerte_mit_standardschwellen[89.99-WARNUNG] PASSED [ 44%]
tests/test_core.py::test_bewerte_mit_standardschwellen[90-KRITISCH] PASSED [ 55%]
tests/test_core.py::test_bewerte_mit_standardschwellen[100-KRITISCH] PASSED [ 66%]
tests/test_core.py::test_bewerte_mit_eigenen_schwellen PASSED            [ 77%]
tests/test_core.py::test_belegung_ist_ein_prozentwert PASSED             [ 88%]
tests/test_core.py::test_unbekannter_pfad_wirft_fehler PASSED            [100%]

============================== 9 passed in 0.01s ===============================
```

Jeder Fall der Parametrisierung erscheint als eigener Test, mit den Werten in eckigen Klammern. Fällt einer durch, sieht man sofort, welcher Wert das Problem ist. Zur Erinnerung: Die Zahl 79.9 und die Zahl 80 liegen dicht beieinander; genau solche Grenzfälle (englisch *boundary values*) sind die, bei denen Programmierfehler sitzen, etwa ein `>` statt `>=`.

**Gleitkommazahlen vergleichen.** Eine Falle beim Testen von Rechenergebnissen sind Gleitkommazahlen. Computer speichern sie binär, und viele Dezimalbrüche lassen sich nicht exakt darstellen; `0.1 + 0.2 == 0.3` ist in Python deshalb falsch. Für Tests bietet `pytest` die Hilfsfunktion `pytest.approx`, die Gleichheit innerhalb einer kleinen Toleranz prüft. Wir demonstrieren beides in einem kleinen Test, den wir absichtlich nicht in unsere Suite aufnehmen, sondern einmal ausführen.

**Datei** `/tmp/test_gleitkomma.py`**:**

```python
import pytest


def test_exakter_vergleich_scheitert():
    assert 0.1 + 0.2 == 0.3


def test_approx_besteht():
    assert 0.1 + 0.2 == pytest.approx(0.3)
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_gleitkomma.py; cd ~/plattenwacht
F.                                                                       [100%]
=================================== FAILURES ===================================
_______________________ test_exakter_vergleich_scheitert _______________________

    def test_exakter_vergleich_scheitert():
>       assert 0.1 + 0.2 == 0.3
E       assert (0.1 + 0.2) == 0.3

test_gleitkomma.py:5: AssertionError
=========================== short test summary info ============================
FAILED test_gleitkomma.py::test_exakter_vergleich_scheitert - assert (0.1 + 0...
1 failed, 1 passed in 0.01s
```

Der erste Test schlägt fehl, und `pytest` zeigt dabei die tatsächliche Zahl `0.30000000000000004`. Der zweite besteht. Verwenden Sie `pytest.approx` immer, wenn Sie das Ergebnis einer Rechnung mit Kommazahlen prüfen; es akzeptiert standardmäßig einen kleinen relativen Fehler und lässt sich mit `rel=` und `abs=` anpassen. Für unsere Prozentberechnung brauchen wir es vorerst nicht, weil wir nur mit Bereichen prüfen.

**Tests auswählen und steuern.** In einem wachsenden Projekt will man nicht immer alle Tests ausführen. Die wichtigsten Optionen: `-k ausdruck` wählt Tests nach einem Teil ihres Namens aus, `-x` bricht beim ersten Fehler ab, `--lf` (*last failed*) wiederholt nur die zuletzt fehlgeschlagenen, `-q` verkürzt die Ausgabe, `--tb=short` (oder `=no`) verkürzt die Rückverfolgung bei Fehlern, und mit `pfad::testname` startet man einen einzelnen Test. Mit `-ra` listet `pytest` am Ende zu allen nicht bestandenen oder übersprungenen Tests den Grund.

```pycon
(.venv) ~/plattenwacht$ pytest -q -k "grenze or eigenen"
.                                                                        [100%]
1 passed, 8 deselected in 0.01s
(.venv) ~/plattenwacht$ pytest -q tests/test_core.py::test_unbekannter_pfad_wirft_fehler
.                                                                        [100%]
1 passed in 0.01s
(.venv) ~/plattenwacht$ pytest -q -k "gibt_es_nicht"; echo "Exit-Code: $?"

9 deselected in 0.01s
Exit-Code: 5
```

Die erste Auswahl findet den Test mit `eigenen` im Namen (das Wort `grenze` kommt nirgends vor, `or` verbindet die Bedingungen). Die zweite startet gezielt einen Test aus einer Datei. Die dritte wählt *keinen* Test aus: `pytest` meldet „no tests ran“ und liefert den Exit-Code 5, was für Automatisierungen ein Hinweis auf eine falsche Auswahl oder falsche Namen ist.

**Typische Fehler und Stolperfallen.** Der häufigste Anfängerfehler ist, dass ein Test gar nicht läuft, weil der Name nicht den Regeln folgt. `pytest` sammelt standardmäßig Dateien mit dem Muster `test_*.py` oder `*_test.py` und darin Funktionen, deren Name mit `test` beginnt (außerdem Methoden von Klassen, deren Name mit `Test` beginnt). Eine Funktion `prueft_grenzen` in der Datei `tests/pruefung.py` wird still ignoriert, und das gefährliche daran ist, dass nichts darauf hinweist. Wir stellen es nach:

**Datei** `tests/pruefung.py`**:**

```python
def test_wird_nie_gefunden():
    assert False
```

```pycon
(.venv) ~/plattenwacht$ pytest -q
.........                                                                [100%]
9 passed in 0.01s
(.venv) ~/plattenwacht$ rm tests/pruefung.py
```

Die Suite meldet weiterhin nur die bekannten Tests als bestanden, obwohl in der Datei ein Test steht, der immer fehlschlagen würde. Prüfen Sie deshalb nach dem Anlegen eines neuen Tests, dass die Zahl der Tests gestiegen ist. Ein zweiter Klassiker ist die Klammer um die Prüfung: `assert (x == 1, "Meldung")` prüft ein *Tupel*, und ein nichtleeres Tupel ist immer wahr, der Test besteht also immer. Python warnt davor, wenn man es zulässt.

**Datei** `/tmp/test_tupel.py`**:**

```python
def test_tupel_ist_immer_wahr():
    x = 2
    assert (x == 1, "x sollte 1 sein")
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_tupel.py 2>&1 | head -12; cd ~/plattenwacht
.                                                                        [100%]
1 passed in 0.01s
```

Der Test besteht, obwohl `x` gleich 2 ist, und `pytest` gibt nur eine Warnung aus. Richtig schreibt man die Meldung ohne Klammern um die ganze Prüfung: `assert x == 1, "x sollte 1 sein"`. Ein dritter Fehler ist, dass Tests voneinander abhängen: Test B setzt voraus, dass Test A etwas getan hat (eine Datei angelegt, eine globale Variable gesetzt). `pytest` garantiert keine Reihenfolge, wenn man einzelne Tests oder Zufallsreihenfolgen wählt, und der Fehler tritt dann scheinbar zufällig auf. Jeder Test muss für sich allein laufen können; wie man gemeinsame Vorbereitung sauber löst, zeigt der nächste Abschnitt mit den Fixtures. Viertens: Ein Test ohne `assert` oder `pytest.raises` prüft nichts und besteht immer. Fünftens: Wer Tests schreibt, die vom Rechner abhängen (feste Prozentwerte der Platte, aktuelle Uhrzeit, ein bestimmtes Netzwerk), erzeugt Tests, die auf anderen Rechnern oder zu anderen Zeiten scheitern. Dagegen helfen Eigenschaften statt fester Werte (wie oben), Fixtures (14.2) und Mocks (14.3).

Wir sichern unseren Fortschritt in Git. Die Tests gehören ins Repository, die Zwischenspeicher von `pytest` (`.pytest_cache`) stehen bereits in der `.gitignore`.

```pycon
(.venv) ~/plattenwacht$ git add pyproject.toml tests
(.venv) ~/plattenwacht$ git commit -q -m "Tests fuer core.py mit pytest anlegen"
(.venv) ~/plattenwacht$ git status --short
```

**Was ein guter Test ausmacht.** Ein Test sollte schnell, unabhängig und eindeutig sein. Schnell, damit man ihn bei jeder Änderung ausführt; unabhängig, damit die Reihenfolge keine Rolle spielt; eindeutig, damit ein Fehlschlag genau eine Ursache nahelegt. Gute Tests prüfen das *Verhalten*, nicht die Umsetzung: Ob `bewerte` intern `if` oder ein Wörterbuch verwendet, ist egal, das Ergebnis zählt. Testet man die Umsetzung, brechen die Tests bei jedem Umbau, obwohl nichts kaputt ist, und man hört auf, ihnen zu vertrauen. Und ein Test, der zum ersten Mal *fehlschlägt*, sollte etwas Neues über den Code sagen; deshalb schreibt man bei Fehlerkorrekturen zuerst einen Test, der den Fehler zeigt, und korrigiert dann den Code, bis er besteht. Das Vorgehen heißt *testgetriebene Entwicklung*, wenn man es konsequent so macht, und ist als Fehlerkorrekturverfahren auch für Administrationsskripte ein gutes Werkzeug.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Alle gezeigten Befehle (Parametrisierung, Ausnahmen, Auswahl mit `-k`, die Namenskonvention `test_*.py`) wurden mit dem installierten `pytest` 9.2.0 tatsächlich ausgeführt; die Ausgaben stammen aus diesen Läufen. Die Beschreibung der Optionen (`-k`, `-x`, `--lf`, `-ra`) und der Exit-Codes ist Fachwissen und wurde nicht in der Dokumentation nachgelesen; die Exit-Codes 0, 1 und 5 haben wir im Lauf beobachtet. Die Fixture-Dokumentation von `pytest` wurde für 14.2 abgerufen (nur über eine Spiegelseite, siehe Quellenverzeichnis).

> **Merke**
>
> Ein Test ist eine Funktion `test_...` mit `assert`; Dateien heißen `test_*.py`. `pytest` zeigt bei Fehlern die beteiligten Werte. `@pytest.mark.parametrize` führt eine Funktion mit mehreren Fällen aus, `pytest.raises` erwartet eine Ausnahme, `pytest.approx` vergleicht Kommazahlen. Testen Sie Grenzwerte beidseitig und Eigenschaften statt rechnerabhängiger Werte. Ein Test, der nie fehlschlagen kann oder nicht gefunden wird, ist wertlos; prüfen Sie die Zahl der Tests. Exit-Code 0 heißt bestanden, 1 fehlgeschlagen, 5 keine Tests gefunden.

> **Praxisbeispiel**
>
> Tests sind die Absicherung, die erst ermöglicht, ein Skript, das schon im Betrieb läuft, gefahrlos zu ändern. Im Alltag laufen sie an drei Stellen: lokal vor jedem Commit, automatisch auf dem Server der Versionsverwaltung bei jedem Push (Continuous Integration, in 14.5 und Modul 15 kurz angesprochen) und vor jedem Ausrollen auf Produktionsrechner. Besonders lohnend sind Tests für die Stellen, an denen Alarme entstehen: Schwellenvergleiche, Zeitberechnungen, Formatierung von Meldungen. Dort kostet ein Fehler am meisten, und dort sind Tests am einfachsten zu schreiben.

## 14.2 Fixtures

Sobald Tests mehr als reine Rechnung prüfen, brauchen sie eine Vorbereitung: eine Datei, die vorhanden sein muss, ein Verzeichnis, ein gesetzter Wert in der Umgebung. Man könnte diese Vorbereitung am Anfang jedes Tests von Hand schreiben, aber das führt zu Wiederholung und, schlimmer, zu Tests, die ihre Spuren hinterlassen und andere Tests beeinflussen. Dafür gibt es bei `pytest` *Fixtures*. Eine Fixture ist eine Funktion, die etwas vorbereitet und es dem Test zur Verfügung stellt. Der Test fordert sie an, indem er einen Parameter gleichen Namens in seine Argumentliste schreibt; `pytest` erkennt den Namen, führt die Fixture aus und übergibt das Ergebnis. Das nennt man *Einschleusen von Abhängigkeiten* (englisch *dependency injection*): Der Test sagt, was er braucht, und das Werkzeug liefert es.

**Eingebaute Fixtures: tmp_path und capsys.** Zwei Fixtures bringt `pytest` selbst mit, die man ständig braucht. `tmp_path` liefert ein frisches, leeres temporäres Verzeichnis (als `pathlib.Path`, Modul 4), das für jeden Test neu angelegt wird; dort darf der Test Dateien schreiben, ohne das Projekt zu verschmutzen. `capsys` fängt alles ab, was das Programm mit `print` auf die Standardausgabe oder Fehlerausgabe schreibt, damit man es prüfen kann. Wir verwenden beide, um die Kommandozeile unseres Projekts zu testen. Dafür erweitern wir zuerst das Projekt um eine kleine Funktion, mit der man die Schwellen auch über Umgebungsvariablen vorgeben kann; das ist bei Skripten, die `cron` startet, praktisch. Die Funktion nimmt optional eine Umgebung als Parameter, damit sie sich leicht testen lässt.

**Datei** `src/plattenwacht/core.py`**:**

```python
"""Kernlogik von plattenwacht: messen und bewerten, ohne Ein- und Ausgabe."""
import os
import shutil

SCHWELLE_WARNUNG = 80
SCHWELLE_KRITISCH = 90


def belegung_prozent(pfad):
    """Gibt zurueck, zu wie viel Prozent das Dateisystem von pfad belegt ist."""
    nutzung = shutil.disk_usage(pfad)
    return nutzung.used / nutzung.total * 100


def bewerte(prozent, warnung=SCHWELLE_WARNUNG, kritisch=SCHWELLE_KRITISCH):
    """Ordnet einen Prozentwert einer Stufe zu: OK, WARNUNG oder KRITISCH."""
    if prozent >= kritisch:
        return "KRITISCH"
    if prozent >= warnung:
        return "WARNUNG"
    return "OK"


def pruefe(pfad, warnung=SCHWELLE_WARNUNG, kritisch=SCHWELLE_KRITISCH):
    """Misst und bewertet in einem Schritt; gibt (stufe, prozent) zurueck."""
    prozent = belegung_prozent(pfad)
    return bewerte(prozent, warnung, kritisch), prozent


def schwellen_aus_umgebung(umgebung=None):
    """Liest PLATTENWACHT_WARNUNG und PLATTENWACHT_KRITISCH; sonst gelten die Standardwerte."""
    umgebung = os.environ if umgebung is None else umgebung
    warnung = float(umgebung.get("PLATTENWACHT_WARNUNG", SCHWELLE_WARNUNG))
    kritisch = float(umgebung.get("PLATTENWACHT_KRITISCH", SCHWELLE_KRITISCH))
    return warnung, kritisch
```

Neu ist nur die Funktion am Ende. Sie liest zwei Umgebungsvariablen und wandelt sie mit `float` in Zahlen um; fehlen sie, gelten die bisherigen Werte. Die Kommandozeile soll diese Werte als Vorgabe für ihre Optionen verwenden, sodass eine Option auf der Kommandozeile die Umgebung überstimmt.

**Datei** `src/plattenwacht/cli.py`**:**

```python
"""Kommandozeile von plattenwacht."""
import argparse

from . import __version__
from .core import pruefe, schwellen_aus_umgebung

EXIT_CODES = {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}


def baue_parser():
    warnung_std, kritisch_std = schwellen_aus_umgebung()
    parser = argparse.ArgumentParser(
        prog="plattenwacht",
        description="Meldet, wenn ein Dateisystem fast voll ist.",
    )
    parser.add_argument("pfad", nargs="?", default="/", help="zu pruefender Pfad (Standard: /)")
    parser.add_argument("--warnung", type=float, default=warnung_std, help="Schwelle fuer WARNUNG in Prozent")
    parser.add_argument("--kritisch", type=float, default=kritisch_std, help="Schwelle fuer KRITISCH in Prozent")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    return parser


def main(argv=None):
    args = baue_parser().parse_args(argv)
    stufe, prozent = pruefe(args.pfad, args.warnung, args.kritisch)
    print(f"{stufe}: '{args.pfad}' ist zu {prozent:.1f} % belegt")
    return EXIT_CODES[stufe]
```

Der Parser wird bei jedem Aufruf von `baue_parser()` neu gebaut und liest dabei die Umgebung; das ist wichtig für die Tests, denn so wirkt eine in einem Test gesetzte Variable auch tatsächlich. Jetzt schreiben wir die Tests für die Kommandozeile. Der Test `test_ausgabe_nennt_stufe_und_pfad` verwendet `capsys` und `tmp_path`: Er ruft `main` mit einer Argumentliste auf (dafür haben wir den Parameter `argv` vorgesehen), liest dann mit `capsys.readouterr()` ab, was ausgegeben wurde, und prüft Text und Rückgabewert. Weil Schwellen außerhalb des Bereichs von 0 bis 100 jeden Messwert auf die gleiche Stufe zwingen, sind die Ergebnisse unabhängig vom Rechner: Mit den Schwellen 0 und 0 ist jede Belegung kritisch, mit 101 und 102 ist jede in Ordnung.

**Datei** `tests/test_cli.py`**:**

```python
import pytest

from plattenwacht.cli import main


@pytest.mark.parametrize(
    "warnung, kritisch, stufe, code",
    [
        ("101", "102", "OK", 0),
        ("0", "102", "WARNUNG", 1),
        ("0", "0", "KRITISCH", 2),
    ],
)
def test_exit_code_und_ausgabe(tmp_path, capsys, warnung, kritisch, stufe, code):
    ergebnis = main([str(tmp_path), "--warnung", warnung, "--kritisch", kritisch])
    ausgabe = capsys.readouterr().out
    assert ergebnis == code
    assert ausgabe.startswith(f"{stufe}: '{tmp_path}'")


def test_version_beendet_das_programm(capsys):
    with pytest.raises(SystemExit) as info:
        main(["--version"])
    assert info.value.code == 0
    assert capsys.readouterr().out.startswith("plattenwacht ")
```

Die Parametrisierung aus 14.1 lässt sich mit Fixtures kombinieren: `tmp_path` und `capsys` werden von `pytest` gestellt, `warnung`, `kritisch`, `stufe` und `code` kommen aus der Liste. Der zweite Test prüft, dass `--version` das Programm mit `SystemExit` beendet, was `argparse` so macht; `pytest.raises` liefert dabei ein Objekt, über das man den Wert der Ausnahme ansehen kann. Wir führen die Tests aus.

```pycon
(.venv) ~/plattenwacht$ pytest -q
.............                                                            [100%]
13 passed in 0.01s
```

**Eigene Fixtures und conftest.py.** Eine eigene Fixture ist eine Funktion mit dem Dekorator `@pytest.fixture`. Ihr Rückgabewert ist das, was der Test erhält. Gemeinsam genutzte Fixtures legt man in einer Datei namens `conftest.py` im Testverzeichnis ab; `pytest` lädt sie automatisch, und alle Tests im Verzeichnis (und darunter) sehen die Fixtures darin, ohne etwas importieren zu müssen. Wir schreiben zwei Fixtures. Die erste, `sauberes_umfeld`, entfernt die beiden Umgebungsvariablen des Projekts, damit Tests nicht davon abhängen, was zufällig auf dem Rechner der Entwicklerin gesetzt ist. Dafür nutzt sie `monkeypatch`, eine weitere eingebaute Fixture: Sie ändert Umgebungsvariablen, Attribute oder das Arbeitsverzeichnis und *macht die Änderung nach dem Test automatisch rückgängig*. Die zweite, `arbeitsordner`, legt in `tmp_path` ein Verzeichnis mit einer Beispieldatei an und wechselt für die Dauer des Tests dorthin.

**Datei** `tests/conftest.py`**:**

```python
import pytest


@pytest.fixture
def sauberes_umfeld(monkeypatch):
    """Entfernt die Umgebungsvariablen des Projekts fuer die Dauer eines Tests."""
    monkeypatch.delenv("PLATTENWACHT_WARNUNG", raising=False)
    monkeypatch.delenv("PLATTENWACHT_KRITISCH", raising=False)


@pytest.fixture
def arbeitsordner(tmp_path, monkeypatch):
    """Legt ein Arbeitsverzeichnis mit einer Datei an und wechselt dorthin."""
    ordner = tmp_path / "arbeit"
    ordner.mkdir()
    (ordner / "notiz.txt").write_text("hallo\n", encoding="utf-8")
    monkeypatch.chdir(ordner)
    return ordner
```

Die Option `raising=False` bei `delenv` sorgt dafür, dass es keinen Fehler gibt, wenn die Variable gar nicht gesetzt ist. Beachten Sie, dass die zweite Fixture *selbst* Fixtures anfordert (`tmp_path`, `monkeypatch`); Fixtures dürfen sich gegenseitig verwenden, und `pytest` löst die Reihenfolge automatisch auf. Jetzt Tests, die sie nutzen. Sie prüfen, dass die Umgebung die Standardwerte überstimmt, dass eine Option die Umgebung überstimmt und dass der Pfad `.` relativ zum Arbeitsverzeichnis aufgelöst wird.

**Datei** `tests/test_umgebung.py`**:**

```python
import pytest

from plattenwacht.cli import baue_parser, main
from plattenwacht.core import SCHWELLE_KRITISCH, SCHWELLE_WARNUNG, schwellen_aus_umgebung


def test_ohne_umgebung_gelten_die_standardwerte(sauberes_umfeld):
    assert schwellen_aus_umgebung() == (SCHWELLE_WARNUNG, SCHWELLE_KRITISCH)


def test_umgebung_ueberstimmt_die_standardwerte(sauberes_umfeld, monkeypatch):
    monkeypatch.setenv("PLATTENWACHT_WARNUNG", "55")
    monkeypatch.setenv("PLATTENWACHT_KRITISCH", "66.5")
    assert schwellen_aus_umgebung() == (55.0, 66.5)


def test_option_ueberstimmt_die_umgebung(sauberes_umfeld, monkeypatch):
    monkeypatch.setenv("PLATTENWACHT_WARNUNG", "55")
    args = baue_parser().parse_args(["--warnung", "42"])
    assert args.warnung == 42.0


def test_ungueltiger_wert_in_der_umgebung(sauberes_umfeld):
    with pytest.raises(ValueError):
        schwellen_aus_umgebung({"PLATTENWACHT_WARNUNG": "viel"})


def test_punkt_ist_das_arbeitsverzeichnis(arbeitsordner, capsys):
    main([".", "--warnung", "0", "--kritisch", "102"])
    assert "'.'" in capsys.readouterr().out
    assert (arbeitsordner / "notiz.txt").exists()
```

Der letzte Test fordert die Fixture `arbeitsordner` an und braucht ihren Rückgabewert, um zu prüfen, dass die Beispieldatei existiert. Der Test davor übergibt dem Aufruf von `schwellen_aus_umgebung` ein eigenes Wörterbuch statt der echten Umgebung. Das ist eine weitere Möglichkeit, Abhängigkeiten zu steuern: Wo es der Code erlaubt, übergibt man sie als Parameter, statt sie mit Tricks zu ersetzen. Wir führen die ganze Suite aus.

```pycon
(.venv) ~/plattenwacht$ pytest -q
..................                                                       [100%]
18 passed in 0.01s
```

**Wie Fixtures aufräumen: yield und Gültigkeitsbereich.** Oft muss man nach dem Test aufräumen: eine Verbindung schließen, einen Prozess beenden, eine Datei löschen. Dafür verwendet man in der Fixture statt `return` die Anweisung `yield`. Alles vor dem `yield` ist die Vorbereitung (englisch *setup*), der Wert hinter dem `yield` wird dem Test übergeben, und alles danach ist das Aufräumen (*teardown*). `pytest` führt den Aufräumcode auch dann aus, wenn der Test fehlschlägt. Außerdem hat jede Fixture einen *Gültigkeitsbereich* (*scope*): Standardmäßig wird sie für *jeden Test neu* erzeugt (`scope="function"`). Mit `scope="module"` wird sie einmal je Testdatei, mit `scope="session"` einmal für den ganzen Testlauf erzeugt, was sich für teure Vorbereitungen lohnt. Die Option `--setup-show` macht den Ablauf sichtbar. Wir probieren es in einer Wegwerfdatei aus, die nicht zur Suite gehört.

**Datei** `/tmp/test_ablauf.py`**:**

```python
import pytest


@pytest.fixture
def pro_test():
    print("  -> pro_test: Vorbereitung")
    yield "A"
    print("  -> pro_test: Aufraeumen")


@pytest.fixture(scope="module")
def pro_datei():
    print("  -> pro_datei: Vorbereitung")
    yield "B"
    print("  -> pro_datei: Aufraeumen")


def test_eins(pro_test, pro_datei):
    assert pro_test == "A"


def test_zwei(pro_test, pro_datei):
    assert pro_datei == "B"
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q -s --setup-show test_ablauf.py; cd ~/plattenwacht
  -> pro_datei: Vorbereitung

    SETUP    M pro_datei  -> pro_test: Vorbereitung

        SETUP    F pro_test
        test_ablauf.py::test_eins (fixtures used: pro_datei, pro_test) .  -> pro_test: Aufraeumen

        TEARDOWN F pro_test  -> pro_test: Vorbereitung

        SETUP    F pro_test
        test_ablauf.py::test_zwei (fixtures used: pro_datei, pro_test) .  -> pro_test: Aufraeumen

        TEARDOWN F pro_test  -> pro_datei: Aufraeumen

    TEARDOWN M pro_datei
2 passed in 0.01s
```

Mit `-s` (Ausgaben nicht abfangen) sieht man die Meldungen der Fixtures; sie mischen sich mit den Zeilen von `--setup-show`, weil beides in denselben Ausgabestrom geschrieben wird, und das sieht auf den ersten Blick unordentlich aus. Lesen Sie die Ausgabe an den Stichwörtern `SETUP` und `TEARDOWN` entlang. Die Option `--setup-show` schreibt `SETUP` und `TEARDOWN` mit dem Bereich (`F` für *function*, `M` für *module*). Die Fixture `pro_test` wird zweimal vorbereitet und abgeräumt, die Fixture `pro_datei` nur einmal, und zwar nach dem letzten Test der Datei. Je größer der Bereich, desto weniger Aufwand, aber desto größer die Gefahr, dass Tests sich gegenseitig beeinflussen, wie der nächste Abschnitt zeigt.

**Typische Fehler und Stolperfallen.** Der erste Fehler ist ein Tippfehler im Fixture-Namen. `pytest` bricht den Test ab und nennt, welche Fixtures es gibt:

**Datei** `/tmp/test_tippfehler.py`**:**

```python
def test_mit_falschem_namen(tmp_pfad):
    assert tmp_pfad.exists()
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_tippfehler.py 2>&1 | grep -E "^E  |^ERROR|passed|error"; cd ~/plattenwacht
E                                                                        [100%]
E       fixture 'tmp_pfad' not found
ERROR test_tippfehler.py::test_mit_falschem_namen
1 error in 0.01s
```

Die Meldung `fixture 'tmp_pfad' not found` ist eindeutig, der Test läuft gar nicht erst. Ein zweiter und tückischerer Fehler entsteht durch zu große Gültigkeitsbereiche mit veränderlichen Werten. Wenn eine Fixture mit `scope="module"` eine Liste oder ein Wörterbuch liefert und ein Test sie verändert, sieht der nächste Test die Änderung, und das Ergebnis hängt von der Reihenfolge ab:

**Datei** `/tmp/test_geteilt.py`**:**

```python
import pytest


@pytest.fixture(scope="module")
def liste():
    return []


def test_a_fuegt_etwas_ein(liste):
    liste.append("x")
    assert liste == ["x"]


def test_b_erwartet_leere_liste(liste):
    assert liste == []
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_geteilt.py 2>&1 | grep -E "^E  |passed|failed"; cd ~/plattenwacht
E       AssertionError: assert ['x'] == []
E         
E         Left contains one more item: 'x'
E         Use -v to get more diff
1 failed, 1 passed in 0.01s
```

Der zweite Test scheitert, obwohl er für sich korrekt ist. Läse man ihn allein (`pytest test_geteilt.py::test_b_erwartet_leere_liste`), bestünde er. Solche Fehler sind schwer zu finden, weil sie nur in bestimmten Kombinationen auftreten. Die Regel lautet: Veränderliche Werte gehören in Fixtures mit dem Standardbereich `function`; größere Bereiche sind nur für Dinge gedacht, die der Test nicht verändert (eine teure Konfiguration, ein Testserver). Der dritte Fehler ist das Ändern der echten Umgebung ohne `monkeypatch`:

**Datei** `/tmp/test_umgebung_direkt.py`**:**

```python
import os


def test_setzt_variable_direkt():
    os.environ["MEINE_TESTVARIABLE"] = "1"
    assert os.environ["MEINE_TESTVARIABLE"] == "1"


def test_erwartet_variable_nicht(monkeypatch):
    wert = os.environ.get("MEINE_TESTVARIABLE")
    assert wert is None
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_umgebung_direkt.py 2>&1 | grep -E "^E  |passed|failed"; cd ~/plattenwacht
E       AssertionError: assert '1' is None
1 failed, 1 passed in 0.01s
```

Die direkt gesetzte Variable bleibt bestehen, der zweite Test sieht sie, und er scheitert. Mit `monkeypatch.setenv` wäre sie nach dem ersten Test wieder verschwunden. Dasselbe gilt für das Wechseln des Verzeichnisses mit `os.chdir` und für das Ändern von Modulattributen. Ein vierter Fehler ist, Dateien im Projektverzeichnis statt in `tmp_path` zu schreiben: Der Test hinterlässt Spuren, die den nächsten Lauf beeinflussen und im `git status` auftauchen. Fünftens ist es ein Fehler, in Fixtures zu viel zu tun. Eine Fixture, die zehn Dinge vorbereitet, obwohl der Test nur eines braucht, verlangsamt die Suite und verschleiert, worauf sich der Test stützt.

Wir committen die Änderungen. Die Funktion `schwellen_aus_umgebung` ist neue Funktionalität, die Tests sind dazu da, sie abzusichern.

```pycon
(.venv) ~/plattenwacht$ git add src tests
(.venv) ~/plattenwacht$ git commit -q -m "Schwellen aus der Umgebung lesen; Tests fuer CLI und Umgebung"
(.venv) ~/plattenwacht$ git status --short
(.venv) ~/plattenwacht$ pytest -q
..................                                                       [100%]
18 passed in 0.01s
```

**Fabrik-Fixtures und Parametrisierung von Fixtures.** Zwei fortgeschrittene Muster seien kurz genannt. Braucht ein Test *mehrere* Dinge der gleichen Art mit unterschiedlichen Werten, liefert die Fixture eine *Fabrik*: eine Funktion, die der Test selbst mit Parametern aufruft. So könnte eine Fixture `schreibe_datei` eine Funktion zurückgeben, mit der der Test beliebig viele Dateien mit beliebigem Inhalt im temporären Verzeichnis anlegt. Und mit `@pytest.fixture(params=[...])` führt `pytest` *jeden* Test, der die Fixture nutzt, einmal pro Wert aus; das ist praktisch, um etwa dieselben Tests gegen mehrere Konfigurationen laufen zu lassen. Beides bauen wir nicht aus, es ist aber gut zu wissen, dass es sie gibt, wenn man in der Dokumentation darauf stößt.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Beschreibung von Fixtures (Anforderung über den Parameternamen, `yield` für das Aufräumen, Gültigkeitsbereiche `function`, `module`, `session`, `conftest.py`, die eingebauten Fixtures `tmp_path`, `capsys` und `monkeypatch`) stützt sich auf die abgerufene Fixture-Seite der `pytest`-Dokumentation. Diese konnte nur über die chinesischsprachige Spiegelseite der Dokumentation abgerufen werden, die den englischen Text enthält; sie kann hinter der aktuellen Fassung zurückliegen, weshalb alle gezeigten Verhaltensweisen zusätzlich mit `pytest` 9.2.0 ausgeführt wurden. Die Abschnitte zu Fabrik-Fixtures und `params` sind Fachwissen und wurden nicht ausgeführt.

> **Merke**
>
> Eine Fixture ist eine Funktion mit `@pytest.fixture`, die ein Test über einen gleichnamigen Parameter anfordert. `yield` trennt Vorbereitung und Aufräumen, das auch bei fehlgeschlagenen Tests läuft. Gemeinsame Fixtures stehen in `conftest.py`. Eingebaut sind `tmp_path` (temporäres Verzeichnis), `capsys` (Ausgaben abfangen) und `monkeypatch` (Umgebung, Attribute, Verzeichnis ändern und automatisch zurücksetzen). Veränderliche Werte gehören in Fixtures mit dem Standardbereich je Test; ändern Sie die echte Umgebung nie direkt.

> **Praxisbeispiel**
>
> Bei Administrationsskripten sind es oft Dateien und Umgebungsvariablen, die Tests unzuverlässig machen: Ein Skript, das `/etc/irgendwas` liest oder in `/var/log` schreibt, lässt sich ohne Hilfe nicht gefahrlos testen. Der Ausweg ist, solche Pfade als Parameter zu übergeben (wie wir es bei `pfad` und `argv` getan haben) und im Test auf `tmp_path` zu zeigen. Genauso hilft `capsys`, die Ausgabe eines Kommandos zu prüfen, ohne das Skript als Prozess zu starten, und `monkeypatch`, Umgebungsvariablen wie Zugangsdaten für den Test zu setzen, ohne echte Zugangsdaten zu brauchen.

## 14.3 Mocking (subprocess, Netzwerk)

Bisher haben wir Code getestet, der nur rechnet oder auf Dateien in einem temporären Verzeichnis zugreift. Administrationsskripte tun aber andere Dinge: Sie starten Kommandos, fragen das Betriebssystem, rufen Webdienste auf und verschicken Meldungen. Solche Aufrufe sind für Tests aus mehreren Gründen ungeeignet. Sie sind *langsam* (ein Netzwerkzugriff dauert Millisekunden bis Sekunden, ein Test sollte Mikrosekunden dauern), *unzuverlässig* (der Dienst kann ausgefallen sein, das Netz fehlen) und *nebenwirkungsreich* (ein Test, der wirklich eine Alarmmeldung an den Bereitschaftsdienst schickt, wird schnell unbeliebt). Außerdem will man die *Fehlerfälle* prüfen, die sich mit einer echten Umgebung kaum herbeiführen lassen: Was macht das Programm, wenn das Netz ausfällt, wenn das Kommando fehlt, wenn die Platte plötzlich 100 Prozent voll ist?

Die Lösung heißt *Testdoppel* (englisch *test double*): ein Ersatz, der sich dem Code gegenüber wie das Original verhält, in Wirklichkeit aber vom Test gesteuert wird. Ein solcher Ersatz liefert vorbereitete Antworten (*Stub*), und ein Ersatz, der zusätzlich aufzeichnet, wie er aufgerufen wurde, heißt *Mock*. Im Alltag sagt man „mocken“ zu beidem. Die Standardbibliothek bringt dafür das Modul `unittest.mock` mit; `pytest` bringt zusätzlich die Fixture `monkeypatch`, die wir schon kennen. Die Dokumentation von `unittest.mock` (abgerufen) beschreibt die Klasse `Mock` als flexibles Objekt, das beliebige Aufrufe annimmt und aufzeichnet, und `patch` als Werkzeug, das ein Objekt für die Dauer eines Tests durch ein Mock ersetzt und danach wiederherstellt.

**Das Projekt erweitern.** Damit es etwas zu mocken gibt, bekommt `plattenwacht` zwei neue Fähigkeiten. Das Modul `befehle.py` fragt das Kommando `df` nach der Belegung (ein Beispiel für das Starten eines Programms mit `subprocess`, Modul 5). Das Modul `meldung.py` schickt eine Nachricht an einen Webhook (ein Beispiel für einen Netzwerkzugriff; wir verwenden `urllib.request` aus der Standardbibliothek, damit das Projekt keine Abhängigkeiten braucht). Zuerst die Befehlsseite.

**Datei** `src/plattenwacht/befehle.py`**:**

```python
"""Aufruf externer Kommandos."""
import subprocess


def df_belegung(pfad):
    """Fragt das Kommando df nach der Belegung in Prozent (ganze Zahl)."""
    try:
        ergebnis = subprocess.run(
            ["df", "--output=pcent", str(pfad)],
            capture_output=True,
            text=True,
            timeout=10,
        )
    except FileNotFoundError as fehler:
        raise RuntimeError("Kommando df nicht gefunden") from fehler
    if ergebnis.returncode != 0:
        raise RuntimeError(f"df fehlgeschlagen: {ergebnis.stderr.strip()}")
    zeilen = ergebnis.stdout.strip().splitlines()
    return int(zeilen[-1].strip().rstrip("%"))
```

Die Funktion startet `df` mit einer Liste von Argumenten (nie als zusammengesetzte Zeichenkette, siehe Modul 5), fängt die Ausgabe ab und gibt dem Kommando höchstens zehn Sekunden Zeit. Ist das Kommando nicht vorhanden, wirft `subprocess.run` ein `FileNotFoundError`, das wir in eine verständlichere Meldung übersetzen. Bei einem Fehlercode wird die Fehlerausgabe in die Meldung übernommen. Die Ausgabe von `df --output=pcent` besteht aus einer Kopfzeile `Use%` und einer Zeile mit dem Wert; die letzte Zeile wird von Prozentzeichen und Leerzeichen befreit und in eine Zahl umgewandelt. Zur Anschauung rufen wir das Kommando einmal wirklich auf. Der Wert wird bei Ihnen ein anderer sein.

```pycon
(.venv) ~/plattenwacht$ df --output=pcent /tmp
Use%
 30%
(.venv) ~/plattenwacht$ python3 -c "from plattenwacht.befehle import df_belegung; print(df_belegung('/tmp'))"
30
```

Jetzt die Netzwerkseite. Die Funktion baut aus dem Text ein JSON-Dokument, schickt es als Anfrage an die Adresse und liefert den Statuscode zurück. Fehler (keine Verbindung, Statuscodes wie 404) lösen Ausnahmen aus, die der Aufrufer behandeln muss.

**Datei** `src/plattenwacht/meldung.py`**:**

```python
"""Meldungen an einen Webhook senden."""
import json
import urllib.request


def sende_webhook(url, text, timeout=5):
    """Sendet text als JSON an url und gibt den HTTP-Statuscode zurueck."""
    daten = json.dumps({"text": text}).encode("utf-8")
    anfrage = urllib.request.Request(
        url, data=daten, headers={"Content-Type": "application/json"}
    )
    with urllib.request.urlopen(anfrage, timeout=timeout) as antwort:
        return antwort.status
```

Schließlich lassen wir die Kommandozeile den Webhook benutzen: Mit der Option `--webhook` wird bei einer Warnung oder einem kritischen Zustand eine Meldung gesendet. Schlägt die Meldung fehl, soll das Programm das auf der Fehlerausgabe festhalten, aber den Exit-Code nicht verändern: Der Zustand der Platte ist wichtiger als der Zustand des Meldewegs.

**Datei** `src/plattenwacht/cli.py`**:**

```python
"""Kommandozeile von plattenwacht."""
import argparse
import sys

from . import __version__
from .core import pruefe, schwellen_aus_umgebung
from .meldung import sende_webhook

EXIT_CODES = {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}


def baue_parser():
    warnung_std, kritisch_std = schwellen_aus_umgebung()
    parser = argparse.ArgumentParser(
        prog="plattenwacht",
        description="Meldet, wenn ein Dateisystem fast voll ist.",
    )
    parser.add_argument("pfad", nargs="?", default="/", help="zu pruefender Pfad (Standard: /)")
    parser.add_argument("--warnung", type=float, default=warnung_std, help="Schwelle fuer WARNUNG in Prozent")
    parser.add_argument("--kritisch", type=float, default=kritisch_std, help="Schwelle fuer KRITISCH in Prozent")
    parser.add_argument("--webhook", help="URL, an die bei WARNUNG oder KRITISCH gemeldet wird")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    return parser


def main(argv=None):
    args = baue_parser().parse_args(argv)
    stufe, prozent = pruefe(args.pfad, args.warnung, args.kritisch)
    text = f"{stufe}: '{args.pfad}' ist zu {prozent:.1f} % belegt"
    print(text)
    if args.webhook and stufe != "OK":
        try:
            sende_webhook(args.webhook, text)
        except OSError as fehler:
            print(f"Webhook fehlgeschlagen: {fehler}", file=sys.stderr)
    return EXIT_CODES[stufe]
```

Die Ausnahmen `urllib.error.URLError` und `HTTPError` sind Unterklassen von `OSError`; deshalb genügt `except OSError`. Nach diesem Umbau laufen die bisherigen Tests unverändert weiter, was ein erster Nutzen der Testsuite ist: Sie bestätigt, dass die Erweiterung nichts Altes zerstört hat.

```pycon
(.venv) ~/plattenwacht$ pytest -q
..................                                                       [100%]
18 passed in 0.01s
```

**Kommandos mocken.** Wir testen `df_belegung`, ohne `df` wirklich zu starten. Dazu ersetzen wir `subprocess.run` im Modul `befehle` durch ein Mock. Die Funktion `patch` aus `unittest.mock` erwartet als Ziel den *Namen* des zu ersetzenden Objekts als Zeichenkette und kann als Kontextmanager mit `with` verwendet werden: Innerhalb des Blocks gilt das Mock, danach wieder das Original. Mit `return_value=` legt man fest, was ein Aufruf zurückgibt. Wir bauen als Rückgabewert ein echtes `subprocess.CompletedProcess`, damit unser Code genau die Felder vorfindet, die er erwartet. Zusätzlich prüft der Test mit `assert_called_once_with`, dass das Kommando genau so aufgerufen wurde wie beabsichtigt.

**Datei** `tests/test_befehle.py`**:**

```python
import subprocess
from unittest.mock import patch

import pytest

from plattenwacht.befehle import df_belegung


def fertig(stdout, returncode=0, stderr=""):
    return subprocess.CompletedProcess(args=["df"], returncode=returncode, stdout=stdout, stderr=stderr)


def test_ausgabe_von_df_wird_gelesen():
    with patch("plattenwacht.befehle.subprocess.run", return_value=fertig("Use%\n 42%\n")) as run:
        assert df_belegung("/var") == 42
    run.assert_called_once_with(
        ["df", "--output=pcent", "/var"], capture_output=True, text=True, timeout=10
    )


def test_fehlercode_wird_zur_ausnahme():
    antwort = fertig("", returncode=1, stderr="df: /gibt/es/nicht: Datei nicht gefunden\n")
    with patch("plattenwacht.befehle.subprocess.run", return_value=antwort):
        with pytest.raises(RuntimeError, match="df fehlgeschlagen"):
            df_belegung("/gibt/es/nicht")


def test_fehlendes_kommando_wird_verstaendlich_gemeldet():
    with patch("plattenwacht.befehle.subprocess.run", side_effect=FileNotFoundError("df")):
        with pytest.raises(RuntimeError, match="nicht gefunden"):
            df_belegung("/")


def test_zeitueberschreitung_wird_nicht_verschluckt():
    fehler = subprocess.TimeoutExpired(cmd="df", timeout=10)
    with patch("plattenwacht.befehle.subprocess.run", side_effect=fehler):
        with pytest.raises(subprocess.TimeoutExpired):
            df_belegung("/")
```

Hier tauchen drei wichtige Werkzeuge auf. `return_value` legt den Rückgabewert fest, `side_effect` legt fest, dass ein Aufruf eine *Ausnahme auslöst* (oder, bei einer Liste, nacheinander mehrere Antworten gibt), und `assert_called_once_with` prüft Anzahl und Argumente der Aufrufe. Der zweite Test nutzt den Parameter `match` von `pytest.raises`, der den Text der Ausnahme mit einem regulären Ausdruck vergleicht (Modul 8). Die Tests laufen ohne ein einziges echtes Kommando und brauchen keinen Bruchteil einer Millisekunde.

```pycon
(.venv) ~/plattenwacht$ pytest -q tests/test_befehle.py
....                                                                     [100%]
4 passed in 0.01s
```

**Netzwerk mocken.** Für die Meldung ersetzen wir `urllib.request.urlopen`. Die Funktion wird in unserem Code als Kontextmanager benutzt (`with ... as antwort`), und das Mock muss das mitmachen. Das Mock-Objekt `MagicMock`, das `patch` standardmäßig verwendet, unterstützt dafür die „magischen“ Methoden `__enter__` und `__exit__` selbst; man legt nur fest, was `__enter__` zurückgibt. Zusätzlich kann der Test die Anfrage untersuchen, die an `urlopen` übergeben wurde: `call_args` enthält die Argumente des letzten Aufrufs, und `call_args.args` sind die Positionsargumente.

**Datei** `tests/test_meldung.py`**:**

```python
import json
import urllib.error
from unittest.mock import patch

import pytest

from plattenwacht.meldung import sende_webhook

URL = "https://hooks.example.org/dienst/abc"


def test_sendet_json_und_liefert_den_statuscode():
    with patch("plattenwacht.meldung.urllib.request.urlopen") as urlopen:
        urlopen.return_value.__enter__.return_value.status = 200
        status = sende_webhook(URL, "Platte voll")
    assert status == 200
    anfrage = urlopen.call_args.args[0]
    assert anfrage.full_url == URL
    assert json.loads(anfrage.data) == {"text": "Platte voll"}
    assert anfrage.get_header("Content-type") == "application/json"
    assert urlopen.call_args.kwargs["timeout"] == 5


def test_ausfall_des_netzes_wird_weitergereicht():
    fehler = urllib.error.URLError("keine Verbindung")
    with patch("plattenwacht.meldung.urllib.request.urlopen", side_effect=fehler):
        with pytest.raises(urllib.error.URLError):
            sende_webhook(URL, "egal")


def test_statuscode_404_wird_als_fehler_gemeldet():
    fehler = urllib.error.HTTPError(URL, 404, "Not Found", hdrs=None, fp=None)
    with patch("plattenwacht.meldung.urllib.request.urlopen", side_effect=fehler):
        with pytest.raises(urllib.error.HTTPError) as info:
            sende_webhook(URL, "egal")
    assert info.value.code == 404
```

Der erste Test prüft viel: dass der Statuscode durchgereicht wird, dass die Anfrage an die richtige Adresse geht, dass der Inhalt ein JSON-Dokument mit dem Text ist, dass der Inhaltstyp stimmt und dass ein Zeitlimit gesetzt wurde. Die Schreibweise `Content-type` entsteht, weil `urllib` die Namen von Kopfzeilen intern mit großem Anfangsbuchstaben und sonst kleinen Buchstaben speichert. Die beiden anderen Tests spielen Fehler durch, die man mit einem echten Netz kaum auf Knopfdruck bekommt: einen Ausfall und eine Antwort mit dem Statuscode 404.

```pycon
(.venv) ~/plattenwacht$ pytest -q tests/test_meldung.py
...                                                                      [100%]
3 passed in 0.01s
```

**Dort patchen, wo nachgeschlagen wird.** Die wichtigste Regel beim Mocken steht auch in der abgerufenen Dokumentation und ist die häufigste Fehlerquelle: Man ersetzt ein Objekt dort, wo es *benutzt* (nachgeschlagen) wird, nicht dort, wo es *definiert* ist. Der Grund liegt in der Funktionsweise von `import`. Schreibt ein Modul `from .meldung import sende_webhook`, trägt Python den Namen `sende_webhook` in den Namensraum *dieses* Moduls (`plattenwacht.cli`) ein; es ist eine zweite Referenz auf dieselbe Funktion. Wenn ein Test nun `plattenwacht.meldung.sende_webhook` ersetzt, bleibt die Referenz in `cli` unberührt, und der Code ruft weiter das Original auf. Richtig ist, `plattenwacht.cli.sende_webhook` zu ersetzen. Wir führen beide Fälle vor, zuerst den falschen:

**Datei** `/tmp/test_falsches_ziel.py`**:**

```python
from unittest.mock import patch

from plattenwacht.cli import main


def test_falsches_ziel(tmp_path):
    with patch("plattenwacht.meldung.sende_webhook") as sende:
        main([str(tmp_path), "--warnung", "0", "--webhook", "https://hooks.example.invalid/x"])
    sende.assert_called_once()
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_falsches_ziel.py 2>&1 | grep -E "^E  |[0-9] (passed|failed)"; cd ~/plattenwacht
E           AssertionError: Expected 'sende_webhook' to have been called once. Called 0 times.
1 failed in 0.01s
```

Die Meldung „Expected ’sende_webhook’ to have been called once. Called 0 times.“ sagt es klar: Unser Mock wurde nie aufgerufen, weil die Kommandozeile das Original verwendet hat (und dessen Netzwerkfehler wurde von unserem Code abgefangen). Mit dem richtigen Ziel besteht der Test. Wir schreiben jetzt die Tests für die Kommandozeile mit dem Webhook, mit dem richtigen Ziel `plattenwacht.cli.sende_webhook`.

**Datei** `tests/test_cli_meldung.py`**:**

```python
import urllib.error
from unittest.mock import patch

from plattenwacht.cli import main

URL = "https://hooks.example.org/dienst/abc"


def test_warnung_wird_gemeldet(tmp_path):
    with patch("plattenwacht.cli.sende_webhook") as sende:
        code = main([str(tmp_path), "--warnung", "0", "--kritisch", "102", "--webhook", URL])
    assert code == 1
    sende.assert_called_once()
    url, text = sende.call_args.args
    assert url == URL
    assert text.startswith("WARNUNG: ")


def test_bei_ok_wird_nichts_gemeldet(tmp_path):
    with patch("plattenwacht.cli.sende_webhook") as sende:
        code = main([str(tmp_path), "--warnung", "101", "--kritisch", "102", "--webhook", URL])
    assert code == 0
    sende.assert_not_called()


def test_ausfall_des_meldewegs_aendert_den_exit_code_nicht(tmp_path, capsys):
    fehler = urllib.error.URLError("keine Verbindung")
    with patch("plattenwacht.cli.sende_webhook", side_effect=fehler):
        code = main([str(tmp_path), "--warnung", "0", "--kritisch", "0", "--webhook", URL])
    assert code == 2
    assert "Webhook fehlgeschlagen" in capsys.readouterr().err
```

```pycon
(.venv) ~/plattenwacht$ pytest -q
............................                                             [100%]
28 passed in 0.01s
```

**autospec: Mocks, die das Original kennen.** Ein gewöhnliches Mock akzeptiert *jeden* Aufruf, auch einen, der beim Original gar nicht möglich wäre, etwa mit falscher Argumentzahl oder einem Attribut, das es nicht gibt. Dadurch können Tests bestehen, obwohl der Code in Wirklichkeit abstürzen würde. Mit `autospec=True` erzeugt `patch` ein Mock, das die Signatur des Originals kopiert und bei unpassenden Aufrufen einen `TypeError` auslöst. Eine ähnliche Falle ist, dass ein Mock beliebige Attribute *erfindet*: Ein falsch geschriebenes Attribut führt nicht zwingend zu einem Fehler, sondern oft zu einem Objekt, das als „wahr“ gilt. Wir zeigen alles in einer Wegwerfdatei.

**Datei** `/tmp/test_mock_fallen.py`**:**

```python
from unittest.mock import MagicMock, patch

from plattenwacht.meldung import sende_webhook


def test_tippfehler_in_der_pruefmethode():
    mock = MagicMock()
    mock.aufruf("a")
    assert mock.aufruf.called_once_with("ganz anderes")


def test_erfundenes_attribut_besteht_immer():
    mock = MagicMock()
    mock.aufruf("a")
    assert mock.aufruf.wurde_mit_b_aufgerufen


def test_ohne_autospec_wird_falscher_aufruf_nicht_bemerkt():
    with patch("plattenwacht.meldung.sende_webhook") as ersatz:
        ersatz("url", "text", 5, "ueberzaehlig")


def test_mit_autospec_wird_falscher_aufruf_bemerkt():
    with patch("plattenwacht.meldung.sende_webhook", autospec=True) as ersatz:
        ersatz("url", "text", 5, "ueberzaehlig")
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && pytest -q test_mock_fallen.py 2>&1 | grep -E "^E  |[0-9] (passed|failed)|FAILED"; cd ~/plattenwacht
E               AttributeError: 'called_once_with' is not a valid assertion. Use a spec for the mock if 'called_once_with' is meant to be an attribute.. Did you mean: 'assert_called_once_with'?
E                   TypeError: too many positional arguments
FAILED test_mock_fallen.py::test_tippfehler_in_der_pruefmethode - AttributeEr...
FAILED test_mock_fallen.py::test_mit_autospec_wird_falscher_aufruf_bemerkt - ...
2 failed, 2 passed in 0.01s
```

Der erste Test scheitert, und das ist hier ausnahmsweise erfreulich: `called_once_with` ist ein Tippfehler für `assert_called_once_with`, und neuere Python-Versionen (das zeigt der Lauf mit 3.13.18) erkennen solche Namen und lösen einen `AttributeError` aus. In älteren Versionen bestand ein solcher Test unbemerkt, weil das Mock jedes Attribut erfindet und das Ergebnis eines Aufrufs ein weiteres Mock ist, das als wahr gilt. Der zweite Test zeigt, dass der Schutz Grenzen hat: Ein ganz frei erfundenes Attribut wie `wurde_mit_b_aufgerufen` wird weiterhin klaglos akzeptiert, und der Test besteht, obwohl er nichts prüft. Der dritte Test besteht ebenfalls, obwohl die Funktion nur zwei bis drei Argumente nimmt und der Aufruf mit vier im echten Programm abstürzen würde. Erst der vierte, mit `autospec=True`, schlägt fehl und meldet, dass zu viele Argumente übergeben wurden. Die Folge für die Praxis: Verwenden Sie ausschließlich die Prüfmethoden mit dem Präfix `assert_`, prüfen Sie nie ein Attribut eines Mocks auf Wahrheit, und nehmen Sie `autospec=True`, wann immer das Original eine feste Signatur hat.

**monkeypatch als Alternative.** Statt `unittest.mock.patch` kann man in `pytest` auch `monkeypatch.setattr` verwenden, um ein Attribut für die Dauer eines Tests zu ersetzen, mit demselben Ort-Prinzip. Der Unterschied: `monkeypatch` setzt einfach ein beliebiges Objekt ein (oft eine kleine eigene Funktion oder Klasse), zeichnet aber nichts auf. Das ist nützlich, wenn man nur einen festen Wert braucht. Ein Beispiel ist die Plattenbelegung: Wir wollen `belegung_prozent` prüfen, ohne dass das Ergebnis vom Rechner abhängt. `shutil.disk_usage` liefert ein benanntes Tupel mit `total`, `used` und `free`; wir ersetzen es durch eine Funktion, die ein solches Tupel mit festen Zahlen liefert.

**Datei** `tests/test_belegung.py`**:**

```python
from collections import namedtuple

from plattenwacht import core

Nutzung = namedtuple("Nutzung", "total used free")


def test_belegung_wird_aus_used_und_total_berechnet(monkeypatch):
    monkeypatch.setattr(core.shutil, "disk_usage", lambda pfad: Nutzung(total=200, used=50, free=150))
    assert core.belegung_prozent("/egal") == 25.0


def test_volle_platte_ist_kritisch(monkeypatch):
    monkeypatch.setattr(core.shutil, "disk_usage", lambda pfad: Nutzung(total=100, used=100, free=0))
    stufe, prozent = core.pruefe("/egal")
    assert (stufe, prozent) == ("KRITISCH", 100.0)
```

Diese beiden Tests sind vollständig deterministisch: Egal, wie voll die Platte des Entwicklungsrechners ist, die Zahlen stehen fest. Mit solchen Tests lässt sich endlich auch der Ernstfall prüfen, eine zu 100 Prozent volle Platte, den man mit einer echten Platte kaum herstellen würde. Wir führen die gesamte Suite aus und sichern sie.

```pycon
(.venv) ~/plattenwacht$ pytest -q
..............................                                           [100%]
30 passed in 0.01s
(.venv) ~/plattenwacht$ git add src tests
(.venv) ~/plattenwacht$ git commit -q -m "df-Aufruf und Webhook-Meldung hinzufuegen; Tests mit Mocks"
(.venv) ~/plattenwacht$ git status --short
```

**Typische Fehler und Stolperfallen.** Wir haben die ersten drei bereits gesehen: das falsche Ziel beim Patchen, die Tippfehler und überzähligen Argumente, die ein ungeprüftes Mock schluckt. Der vierte ist *zu viel Mocking*. Ein Test, der alles ersetzt, prüft am Ende nur noch, ob die Mocks wie programmiert antworten, und sagt nichts mehr über den echten Code. Die Faustregel lautet: Ersetzen Sie nur die *Grenzen* zur Außenwelt (Prozesse, Netz, Uhr, Dateisystem außerhalb von `tmp_path`), nicht Ihre eigene Logik. Ein fünfter Fehler ist, nur die Mocks zu testen: Wenn `df_belegung` bei einer geänderten Ausgabe des echten Kommandos bricht, merkt es kein Test, weil das Mock ja die alte Ausgabe liefert. Deshalb ergänzt man bei Bedarf einige wenige *Integrationstests*, die das echte Kommando verwenden und für Läufe auf dem Zielsystem gedacht sind. Sechstens sollte man den echten Netzzugriff in Tests grundsätzlich verbieten, damit ein vergessener Mock nicht still im Netz landet; dafür gibt es Erweiterungen für `pytest`, die wir hier nicht verwenden. Und schließlich: Ein Test, der `time.sleep` aufruft, ist langsam; man ersetzt auch die Uhr, statt zu warten.

**Integrationstest mit dem echten Kommando.** Das Gegenstück zum Mocktest sei zumindest erwähnt: Ein Test, der `df_belegung` wirklich ausführt und prüft, dass ein Prozentwert zwischen 0 und 100 herauskommt. Er hängt vom System ab (`df` muss vorhanden sein), weshalb man ihn mit einer *Markierung* versehen kann, damit man ihn gezielt ein- oder ausschließen kann. Eine solche Markierung ist `@pytest.mark.skipif(...)` für Tests, die nur unter bestimmten Bedingungen laufen sollen, hier etwa nur, wenn `df` gefunden wird.

**Datei** `tests/test_integration.py`**:**

```python
import shutil

import pytest

from plattenwacht.befehle import df_belegung


@pytest.mark.skipif(shutil.which("df") is None, reason="Kommando df nicht vorhanden")
def test_echtes_df_liefert_einen_prozentwert():
    assert 0 <= df_belegung("/") <= 100
```

```pycon
(.venv) ~/plattenwacht$ pytest -q -ra
...............................                                          [100%]
31 passed in 0.01s
(.venv) ~/plattenwacht$ git add tests
(.venv) ~/plattenwacht$ git commit -q -m "Integrationstest mit dem echten df"
```

Hier ist `df` vorhanden, der Test läuft also. Auf einem System ohne `df` würde er übersprungen, und die Option `-ra` würde den Grund in der Zusammenfassung nennen.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Beschreibung von `Mock`, `MagicMock`, `patch`, `return_value`, `side_effect`, `assert_called_once_with`, `autospec` und der Regel „patche dort, wo das Objekt nachgeschlagen wird“ stützt sich auf die abgerufene Dokumentation von `unittest.mock` (in einer älteren, über eine Spiegelseite abrufbaren Fassung des Python-Quelltextbaums; die Aussagen zu diesen Werkzeugen gelten unverändert). Alle gezeigten Tests und Fehlermeldungen wurden mit Python 3.13.18 und pytest 9.2.0 tatsächlich ausgeführt. Hinweise auf Erweiterungen, die den Netzzugriff in Tests verbieten, und auf Integrationstests sind Fachwissen.

> **Merke**
>
> Mocks ersetzen die Grenzen zur Außenwelt (Prozesse, Netz, Zeit) durch vom Test gesteuerte Objekte. `patch("modul.name")` ersetzt *dort*, wo der Name nachgeschlagen wird, nicht dort, wo er definiert ist. `return_value` liefert Antworten, `side_effect` löst Ausnahmen aus, `assert_called_once_with` prüft Aufrufe, `autospec=True` übernimmt die Signatur des Originals. `monkeypatch.setattr` ist die schlichte Alternative. Mocken Sie nur Grenzen, nicht die eigene Logik, und ergänzen Sie wenige Integrationstests mit dem echten System.

> **Praxisbeispiel**
>
> Gerade bei Administrationsskripten sind Mocks unverzichtbar: Man testet, wie sich ein Programm verhält, wenn `systemctl` einen Fehler meldet, wenn die Mail nicht zugestellt werden kann oder wenn die API nach der Anmeldung eine Antwort mit Statuscode 429 liefert, ohne diese Zustände herstellen zu müssen. Die Tests sind in Millisekunden durch und laufen in jeder Umgebung, auch auf einem Build-Server ohne Netzzugang. Eine Faustregel aus der Praxis: Je mehr Fehlerfälle ein Skript abfängt, desto wichtiger werden Tests mit `side_effect`, weil man diese Zweige sonst nur im Ernstfall zu sehen bekommt.

## 14.4 Type Hints

Python ist eine *dynamisch typisierte* Sprache: Eine Variable hat keinen festen Typ, und ob eine Funktion mit einem Text oder einer Zahl aufgerufen wird, merkt Python erst, wenn der Code läuft und etwas schiefgeht. Das macht kleine Skripte angenehm, wird aber in größeren Projekten zum Risiko. Ein Aufruf mit dem falschen Argument, ein vergessener Rückgabewert oder eine Variable, die manchmal `None` ist, führen zu Fehlern, die oft erst im Betrieb auftreten, im ungünstigsten Fall um drei Uhr nachts. *Typangaben* (englisch *type hints*) sind Anmerkungen im Quelltext, die sagen, welche Art von Werten eine Funktion erwartet und zurückgibt. Sie sind zugleich Dokumentation, die nicht veralten kann, ohne dass es auffällt, und Grundlage für Werkzeuge, die Fehler finden, ohne das Programm auszuführen.

**Die Syntax.** Eine Typangabe schreibt man bei Parametern hinter einen Doppelpunkt und beim Rückgabewert hinter einen Pfeil `->`. Die Funktion `def verdopple(x: int) -> int:` sagt: „`x` ist eine ganze Zahl, das Ergebnis auch.“ Für Variablen schreibt man `anzahl: int = 0`. Die üblichen Typen sind `int`, `float`, `str`, `bool`, `bytes` und `None`. Für zusammengesetzte Werte gibt man die Typen der Elemente in eckigen Klammern an: `list[str]` ist eine Liste von Texten, `dict[str, int]` ein Wörterbuch mit Text-Schlüsseln und ganzen Zahlen als Werten, `tuple[str, float]` ein Paar aus Text und Kommazahl, `set[int]` eine Menge. Mit dem senkrechten Strich beschreibt man Alternativen: `int | None` heißt „eine Zahl oder `None`“; diese Schreibweise gibt es seit Python 3.10, davor schrieb man `Optional[int]` aus dem Modul `typing`. Wichtig ist, dass Python diese Angaben zur Laufzeit *nicht durchsetzt*. Wir überzeugen uns davon:

**Datei** `/tmp/verdopple.py`**:**

```python
def verdopple(x: int) -> int:
    return x * 2


print(verdopple("ab"))
```

```pycon
(.venv) ~/plattenwacht$ python3 /tmp/verdopple.py
abab
```

Obwohl die Funktion eine Zahl verlangt, funktioniert der Aufruf mit einem Text, denn Python versteht `'ab' * 2` als Wiederholung und gibt `abab` zurück. Die Typangabe hat die Ausführung nicht beeinflusst. Sie wird vom Interpreter gespeichert und ignoriert. Prüfen kann sie nur ein *Typprüfer* (englisch *type checker*), ein eigenes Programm, das den Quelltext liest, ohne ihn auszuführen. Der bekannteste ist `mypy`, den wir in 14.1 installiert haben; andere sind `pyright` und `pyre`.

**mypy in Aktion.** Wir lassen `mypy` zuerst auf unser Projekt los, wie es ist, ohne jede Typangabe. Das Verzeichnis `src` nennt die Dateien, die geprüft werden sollen.

```pycon
(.venv) ~/plattenwacht$ mypy src
Success: no issues found in 6 source files
```

`mypy` meldet keine Probleme, und das hat einen Grund: Funktionen ohne Typangaben überspringt es standardmäßig, weil es nicht raten kann, was gemeint ist. Der Nutzen beginnt erst mit den Angaben. Wir annotieren jetzt alle Module. Dabei kommen einige neue Dinge vor, die im Anschluss erklärt werden. Zuerst die Kernlogik. Der Typ `Stufe` ist ein `Literal`, eine Typangabe, die nur ganz bestimmte Werte zulässt, hier die drei Zeichenketten, die die Bewertung liefert. Mit der Zeile `Stufe = Literal[...]` gibt man einem Typ einen Namen (*Typalias*), damit man ihn mehrfach verwenden kann.

**Datei** `src/plattenwacht/core.py`**:**

```python
"""Kernlogik von plattenwacht: messen und bewerten, ohne Ein- und Ausgabe."""
import os
import shutil
from collections.abc import Mapping
from pathlib import Path
from typing import Literal

Stufe = Literal["OK", "WARNUNG", "KRITISCH"]

SCHWELLE_WARNUNG: float = 80
SCHWELLE_KRITISCH: float = 90


def belegung_prozent(pfad: str | Path) -> float:
    """Gibt zurueck, zu wie viel Prozent das Dateisystem von pfad belegt ist."""
    nutzung = shutil.disk_usage(pfad)
    return nutzung.used / nutzung.total * 100


def bewerte(
    prozent: float,
    warnung: float = SCHWELLE_WARNUNG,
    kritisch: float = SCHWELLE_KRITISCH,
) -> Stufe:
    """Ordnet einen Prozentwert einer Stufe zu: OK, WARNUNG oder KRITISCH."""
    if prozent >= kritisch:
        return "KRITISCH"
    if prozent >= warnung:
        return "WARNUNG"
    return "OK"


def pruefe(
    pfad: str | Path,
    warnung: float = SCHWELLE_WARNUNG,
    kritisch: float = SCHWELLE_KRITISCH,
) -> tuple[Stufe, float]:
    """Misst und bewertet in einem Schritt; gibt (stufe, prozent) zurueck."""
    prozent = belegung_prozent(pfad)
    return bewerte(prozent, warnung, kritisch), prozent


def schwellen_aus_umgebung(umgebung: Mapping[str, str] | None = None) -> tuple[float, float]:
    """Liest PLATTENWACHT_WARNUNG und PLATTENWACHT_KRITISCH; sonst gelten die Standardwerte."""
    quelle: Mapping[str, str] = os.environ if umgebung is None else umgebung
    warnung = float(quelle.get("PLATTENWACHT_WARNUNG", SCHWELLE_WARNUNG))
    kritisch = float(quelle.get("PLATTENWACHT_KRITISCH", SCHWELLE_KRITISCH))
    return warnung, kritisch
```

Einige Punkte sind erklärungsbedürftig. Die Angabe `str | Path` sagt, dass der Pfad sowohl ein Text als auch ein `Path`-Objekt sein darf; `shutil.disk_usage` akzeptiert beides. Der Typ `Mapping[str, str]` steht für „etwas, das sich wie ein Wörterbuch lesen lässt“, also ein `dict` ebenso wie `os.environ`. Er stammt aus `collections.abc`. Die Faustregel für Parameter lautet, möglichst *allgemeine* Typen zu verlangen (`Mapping` statt `dict`, `Sequence` statt `list`, `Iterable` statt `list`, wenn die Funktion nur liest), und für Rückgabewerte möglichst *genaue*. Dann kann der Aufrufer mehr übergeben, und er weiß genau, was er zurückbekommt. Die Hilfsvariable `quelle` trägt eine eigene Typangabe, weil der Ausdruck mit `if` und `else` zwei verschiedene Typen verbinden würde. Die Konstanten sind als `float` angegeben, auch wenn sie als ganze Zahlen geschrieben sind; Python und `mypy` dürfen eine ganze Zahl dort einsetzen, wo eine Kommazahl erwartet wird.

Nun die beiden Module für Kommandos und Netzwerk. Beide bekommen Typangaben bei Parametern und Rückgabewerten. Die Funktion `sende_webhook` liefert den Statuscode als `int`. Neu ist die Angabe `subprocess.CompletedProcess[str]` nicht nötig, weil wir das Ergebnis nur lokal benutzen.

**Datei** `src/plattenwacht/befehle.py`**:**

```python
"""Aufruf externer Kommandos."""
import subprocess
from pathlib import Path


def df_belegung(pfad: str | Path) -> int:
    """Fragt das Kommando df nach der Belegung in Prozent (ganze Zahl)."""
    try:
        ergebnis = subprocess.run(
            ["df", "--output=pcent", str(pfad)],
            capture_output=True,
            text=True,
            timeout=10,
        )
    except FileNotFoundError as fehler:
        raise RuntimeError("Kommando df nicht gefunden") from fehler
    if ergebnis.returncode != 0:
        raise RuntimeError(f"df fehlgeschlagen: {ergebnis.stderr.strip()}")
    zeilen = ergebnis.stdout.strip().splitlines()
    return int(zeilen[-1].strip().rstrip("%"))
```

**Datei** `src/plattenwacht/meldung.py`**:**

```python
"""Meldungen an einen Webhook senden."""
import json
import urllib.request


def sende_webhook(url: str, text: str, timeout: float = 5) -> int:
    """Sendet text als JSON an url und gibt den HTTP-Statuscode zurueck."""
    daten = json.dumps({"text": text}).encode("utf-8")
    anfrage = urllib.request.Request(
        url, data=daten, headers={"Content-Type": "application/json"}
    )
    with urllib.request.urlopen(anfrage, timeout=timeout) as antwort:
        return int(antwort.status)
```

In `sende_webhook` haben wir `int(antwort.status)` geschrieben statt `antwort.status`; warum, zeigt sich gleich in der Prüfung. Zuletzt die Kommandozeile, deren Hauptfunktion eine optionale Liste von Argumenten nimmt (`list[str] | None`) und einen Exit-Code liefert.

**Datei** `src/plattenwacht/cli.py`**:**

```python
"""Kommandozeile von plattenwacht."""
import argparse
import sys

from . import __version__
from .core import pruefe, schwellen_aus_umgebung
from .meldung import sende_webhook

EXIT_CODES: dict[str, int] = {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}


def baue_parser() -> argparse.ArgumentParser:
    warnung_std, kritisch_std = schwellen_aus_umgebung()
    parser = argparse.ArgumentParser(
        prog="plattenwacht",
        description="Meldet, wenn ein Dateisystem fast voll ist.",
    )
    parser.add_argument("pfad", nargs="?", default="/", help="zu pruefender Pfad (Standard: /)")
    parser.add_argument("--warnung", type=float, default=warnung_std, help="Schwelle fuer WARNUNG in Prozent")
    parser.add_argument("--kritisch", type=float, default=kritisch_std, help="Schwelle fuer KRITISCH in Prozent")
    parser.add_argument("--webhook", help="URL, an die bei WARNUNG oder KRITISCH gemeldet wird")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    return parser


def main(argv: list[str] | None = None) -> int:
    args = baue_parser().parse_args(argv)
    stufe, prozent = pruefe(args.pfad, args.warnung, args.kritisch)
    text = f"{stufe}: '{args.pfad}' ist zu {prozent:.1f} % belegt"
    print(text)
    if args.webhook and stufe != "OK":
        try:
            sende_webhook(args.webhook, text)
        except OSError as fehler:
            print(f"Webhook fehlgeschlagen: {fehler}", file=sys.stderr)
    return EXIT_CODES[stufe]
```

Jetzt prüft `mypy` etwas. Der Aufruf bleibt der gleiche; zuvor lohnt ein Blick auf die Tests: Sie laufen unverändert, denn Typangaben ändern das Laufzeitverhalten nicht.

```pycon
(.venv) ~/plattenwacht$ mypy src
Success: no issues found in 6 source files
(.venv) ~/plattenwacht$ pytest -q
...............................                                          [100%]
31 passed in 0.01s
```

`mypy` meldet „Success“ und die Zahl der geprüften Dateien. Die Testsuite besteht weiter. Wir lassen es jetzt strenger werden. Der Schalter `strict` in den Einstellungen von `mypy` schaltet eine ganze Reihe von Prüfungen ein, darunter das Verbot von Funktionen ohne Typangaben und von Rückgabewerten vom Typ `Any`. Die Einstellungen stehen in der `pyproject.toml` im Abschnitt `[tool.mypy]`.

```pycon
(.venv) ~/plattenwacht$ printf '\n[tool.mypy]\nstrict = true\nfiles = ["src"]\n' >> pyproject.toml
(.venv) ~/plattenwacht$ tail -4 pyproject.toml

[tool.mypy]
strict = true
files = ["src"]
(.venv) ~/plattenwacht$ mypy
Success: no issues found in 6 source files
```

Mit `files = ["src"]` weiß `mypy`, was zu prüfen ist, und der Aufruf `mypy` ohne Argument genügt. Auch unter dem strengen Schalter besteht das Projekt, weil wir alles annotiert haben. Um zu sehen, was die Strenge bewirkt, verändern wir kurz `meldung.py`: Wir nehmen das `int(...)` um den Statuscode wieder heraus.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/return int(antwort.status)/return antwort.status/' src/plattenwacht/meldung.py
(.venv) ~/plattenwacht$ mypy
src/plattenwacht/meldung.py:13: error: Returning Any from function declared to return "int"  [no-any-return]
Found 1 error in 1 file (checked 6 source files)
```

Die Typangabe der Bibliothek für `urlopen` ist so weit gefasst, dass der Typ von `antwort` nicht genau bekannt ist; für `mypy` ist `antwort.status` deshalb `Any`, „irgendetwas“. Unter `strict` ist es verboten, einen `Any` aus einer Funktion zurückzugeben, die einen genauen Typ verspricht, denn `Any` schaltet die Prüfung für alles ab, was damit weiterverarbeitet wird. Mit `int(...)` wird der Wert in einen echten `int` umgewandelt, und die Meldung verschwindet. Wir stellen den Zustand wieder her.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/return antwort.status/return int(antwort.status)/' src/plattenwacht/meldung.py
(.venv) ~/plattenwacht$ mypy
Success: no issues found in 6 source files
```

**Was mypy findet.** Zum Abschluss eine Wegwerfdatei mit typischen Fehlern, damit Sie die Meldungen lesen lernen. Sie enthält drei Klassiker: einen falschen Rückgabetyp, eine Funktion, die manchmal `None` liefert und deren Ergebnis ungeprüft verwendet wird (der berüchtigte Fehler *„NoneType“ hat kein Attribut*), und einen Aufruf mit dem falschen Typ.

**Datei** `/tmp/typfehler.py`**:**

```python
def prozent_text(wert: float) -> str:
    return wert


def finde(namen: list[str], gesucht: str) -> int | None:
    if gesucht in namen:
        return namen.index(gesucht)
    return None


position = finde(["a", "b"], "b")
print(position + 1)
print(prozent_text("viel"))
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && mypy typfehler.py; cd ~/plattenwacht
typfehler.py:2: error: Incompatible return value type (got "float", expected "str")  [return-value]
typfehler.py:12: error: Unsupported operand types for + ("None" and "int")  [operator]
typfehler.py:12: note: Left operand is of type "int | None"
typfehler.py:13: error: Argument 1 to "prozent_text" has incompatible type "str"; expected "float"  [arg-type]
Found 3 errors in 1 file (checked 1 source file)
```

Die erste Meldung sagt, dass in Zeile 2 ein `float` zurückgegeben wird, die Funktion aber einen `str` verspricht. Die zweite ist die wichtigste in der Praxis: `finde` kann `None` liefern, und `None + 1` würde zur Laufzeit abstürzen; `mypy` verlangt, dass man den Fall vorher behandelt. Die dritte bemängelt den Aufruf mit einem Text, wo eine Zahl erwartet wird. Alle drei Fehler hätte man ohne Prüfer erst beim Ausführen gefunden, und den zweiten nur, wenn `finde` tatsächlich `None` liefert. Man behebt ihn mit einer Prüfung, die `mypy` versteht:

**Datei** `/tmp/typfehler_repariert.py`**:**

```python
def prozent_text(wert: float) -> str:
    return f"{wert:.1f} %"


def finde(namen: list[str], gesucht: str) -> int | None:
    if gesucht in namen:
        return namen.index(gesucht)
    return None


position = finde(["a", "b"], "b")
if position is not None:
    print(position + 1)
print(prozent_text(12.34))
```

```pycon
(.venv) ~/plattenwacht$ cd /tmp && mypy typfehler_repariert.py && python3 typfehler_repariert.py; cd ~/plattenwacht
Success: no issues found in 1 source file
2
12.3 %
```

Nach der Prüfung `if position is not None` weiß `mypy`, dass `position` im Block eine Zahl ist (man nennt das *Einengen* des Typs, englisch *narrowing*). Die Meldung „Success“ erscheint, und das Programm läuft.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist die Annahme, Typangaben würden zur Laufzeit geprüft; wir haben gesehen, dass sie das nicht tun. Wer Eingaben von außen (Dateien, Benutzer, Netz) prüfen will, muss das im Programm tun (`isinstance`, Umwandlung mit `int(...)`), denn ein Typprüfer sieht nur den Quelltext. Ein zweiter Fehler ist die Verwendung des Typs `Any` als Ausweg: Er schaltet die Prüfung ab, und ein einziger `Any` kann sich durch ein ganzes Programm fortpflanzen. Der dritte ist das Unterdrücken von Meldungen mit dem Kommentar `# type: ignore` an der betreffenden Zeile. Das ist als letzter Ausweg erlaubt, soll aber mit einem Grund versehen sein, und ein Projekt mit vielen davon hat sich die Prüfung abgewöhnt. Viertens ist ein gefährlicher Standardwert wie `def f(x: list[int] = [])` ein Fehler, den `mypy` nicht findet und der `ruff` (14.5) meldet: Die Liste wird einmal angelegt und von allen Aufrufen geteilt. Fünftens sind Typangaben von Bibliotheken ein eigenes Thema: Viele Pakete bringen sie mit, bei anderen gibt es separate Pakete mit dem Präfix `types-` (zum Beispiel `types-requests`), und bei manchen fehlen sie ganz, was `mypy` unter `strict` meldet. Sechstens muss man beim Annotieren bestehender Projekte nicht alles auf einmal tun: `mypy` lässt sich pro Modul unterschiedlich streng einstellen, und man beginnt mit den wichtigsten Modulen.

Wir sichern den Stand. Damit auch andere Projekte, die `plattenwacht` benutzen, unsere Typangaben nutzen, würde man zusätzlich eine leere Datei `py.typed` in das Paket legen (PEP 561); das ist für ein Kommandozeilenprogramm nicht nötig, für Bibliotheken schon.

```pycon
(.venv) ~/plattenwacht$ git add pyproject.toml src
(.venv) ~/plattenwacht$ git commit -q -m "Typangaben fuer alle Module; mypy im strict-Modus"
(.venv) ~/plattenwacht$ git status --short
```

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Syntax der Typangaben, der Schalter `strict` und die Fehlermeldungen von `mypy` stammen aus Läufen mit Python 3.13.18 und mypy 2.4.1; die Dokumentationsseiten zu `typing` und `mypy` konnten in diesem Arbeitsbereich nicht abgerufen werden und stehen deshalb nur unter „weiterführend“. Die Schreibweise `X | None` (seit Python 3.10), die Typen aus `collections.abc` und die Hinweise zu `py.typed` (PEP 561), zu Paketen mit dem Präfix `types-` und zur Abstufung der Strenge pro Modul sind Fachwissen.

> **Merke**
>
> Typangaben (`def f(x: int) -> str`) dokumentieren Erwartungen und werden zur Laufzeit *nicht* geprüft; das erledigt ein Typprüfer wie `mypy` ohne Ausführung des Programms. Üblich sind `list[str]`, `dict[str, int]`, `tuple[str, float]`, `X | None`, `Literal[...]` und allgemeine Typen für Parameter (`Mapping`, `Sequence`, `Iterable`). Konfiguriert wird `mypy` in `[tool.mypy]`; `strict = true` ist der Zielzustand. Behandeln Sie `None` ausdrücklich, vermeiden Sie `Any` und `# type: ignore` ohne Begründung.

> **Praxisbeispiel**
>
> Typangaben zahlen sich dort aus, wo Funktionen zwischen Menschen und Modulen weitergereicht werden: Wer ein fremdes Skript liest, sieht an der Signatur, was hineingehört und was herauskommt, und der Editor bietet bessere Vervollständigung und warnt vor Fehlern, bevor man speichert. In Teams läuft `mypy` meist automatisch zusammen mit den Tests; ein Commit, der eine Typregel verletzt, wird dann gar nicht erst angenommen. Besonders bei Administrationsskripten, die selten laufen, aber Wichtiges tun, fängt die Prüfung Fehler ab, die man sonst erst im Ernstfall entdeckt.

## 14.5 Linter und Formatter (ruff) und Selbsttest

Tests und Typprüfung finden Fehler im Verhalten. Es gibt eine dritte Gruppe von Problemen, die man schon im Quelltext sehen kann, ohne ihn auszuführen: ein Import, der nie benutzt wird, eine Variable, die gesetzt und nie gelesen wird, ein leerer `except`-Block, der Fehler verschluckt, oder eine Funktion mit einer Liste als Standardwert. Werkzeuge, die solche Stellen finden, heißen *Linter* (nach dem englischen *lint*, Fussel: Sie suchen den Fussel im Code). Davon zu unterscheiden ist der *Formatter*, der den Quelltext nach einheitlichen Regeln umbricht und einrückt, damit Diskussionen über Leerzeichen und Zeilenlängen entfallen und Änderungen in der Versionsverwaltung nur noch den Inhalt zeigen, nicht die Optik.

Lange Zeit brauchte man dafür mehrere Programme: `flake8` und seine Erweiterungen als Linter, `isort` zum Sortieren der Importe, `black` als Formatter. `ruff` ist ein neueres Werkzeug, das diese Aufgaben in einem einzigen, sehr schnellen Programm vereint; es ist in Rust geschrieben und prüft ein mittelgroßes Projekt in Sekundenbruchteilen. Es hat zwei Befehle: `ruff check` ist der Linter, `ruff format` der Formatter. Beide lesen ihre Einstellungen aus der `pyproject.toml`, in der auch `pytest` und `mypy` ihre Einstellungen haben. Die Dokumentation von `ruff` (abgerufen) beschreibt die Konfiguration im Abschnitt `[tool.ruff]` und listet die Regeln mit Kennungen nach dem Schema Buchstabencode plus Zahl, etwa `F401`.

**Der Linter an einem schlechten Beispiel.** Wir legen eine absichtlich schlechte Datei an, die typische Anfängerfehler enthält. Sie soll nicht Teil des Projekts werden, deshalb liegt sie in einem Verzeichnis `beispiel`, das wir hinterher löschen.

**Datei** `beispiel/alt.py`**:**

```python
import os, sys
import json
from pathlib import Path

def lade(pfad, daten=[]):
    if os.path.exists(pfad) == True:
        f = open(pfad)
        text = f.read()
        daten.append(text)
    unbenutzt = 5
    try:
        x = json.loads(text)
    except:
        pass
    return daten
```

Die Datei läuft sogar, ist aber voller Probleme. Wir lassen `ruff` darüber laufen. Die Option `--output-format=concise` gibt pro Fund eine Zeile aus; ohne sie zeigt `ruff` zu jedem Fund einen Ausschnitt mit Markierung und Korrekturvorschlag, was beim Lernen hilfreich, aber lang ist. Jede Zeile nennt Datei, Zeile und Spalte, die Kennung der Regel, das Zeichen `[*]` bei automatisch behebbaren Funden und eine Beschreibung.

```pycon
(.venv) ~/plattenwacht$ ruff check --output-format=concise beispiel
beispiel/alt.py:1:1: I001 [*] Import block is un-sorted or un-formatted
beispiel/alt.py:1:12: F401 [*] `sys` imported but unused
beispiel/alt.py:3:21: F401 [*] `pathlib.Path` imported but unused
beispiel/alt.py:5:22: B006 Do not use mutable data structures for argument defaults
beispiel/alt.py:7:13: SIM115 Use a context manager for opening files
beispiel/alt.py:10:5: F841 Local variable `unbenutzt` is assigned to but never used
beispiel/alt.py:12:9: F841 Local variable `x` is assigned to but never used
beispiel/alt.py:13:5: E722 Do not use bare `except`
beispiel/alt.py:13:5: S110 `try`-`except`-`pass` detected, consider logging the exception
Found 9 errors.
[*] 3 fixable with `--fix` (3 unsafe fixes available with `--fix --unsafe-fixes`).
```

Die Funde im Einzelnen: `I001` meldet unsortierte Importe, `F401` importierte, aber nie benutzte Namen (hier `sys` und `Path`), `B006` den veränderlichen Standardwert `daten=[]` (die Liste wird einmal angelegt und von allen Aufrufen geteilt, ein berüchtigter Fehler), `SIM115` eine Datei, die ohne `with` geöffnet wird und deshalb bei einem Fehler offen bleibt, `F841` zwei lokale Variablen, die gesetzt und nie gelesen werden, `E722` den `except` ohne Angabe der Ausnahme, der auch `KeyboardInterrupt` verschluckt, und `S110` den Block `try` und `except` mit `pass`, der Fehler stillschweigend unterdrückt. Nicht jeder Fund ist ein Fehler, aber jeder ist eine Stelle, an der man hinsehen sollte. Interessant ist auch, was `ruff` *nicht* meldet: Dass in `f.read()` die Datei ohne Zeichenkodierung gelesen wird, überlässt es anderen Regeln.

Wir wenden den Linter auf das echte Projekt an, ohne eigene Einstellungen:

```pycon
(.venv) ~/plattenwacht$ ruff check --output-format=concise src tests
src/plattenwacht/befehle.py:9:20: PLW1510 `subprocess.run` without explicit `check` argument
tests/test_befehle.py:23:5: SIM117 Use a single `with` statement with multiple contexts instead of nested `with` statements
tests/test_befehle.py:29:5: SIM117 Use a single `with` statement with multiple contexts instead of nested `with` statements
tests/test_befehle.py:36:5: SIM117 Use a single `with` statement with multiple contexts instead of nested `with` statements
tests/test_meldung.py:26:5: SIM117 Use a single `with` statement with multiple contexts instead of nested `with` statements
tests/test_meldung.py:33:5: SIM117 Use a single `with` statement with multiple contexts instead of nested `with` statements
tests/test_umgebung.py:1:1: I001 [*] Import block is un-sorted or un-formatted
Found 7 errors.
[*] 1 fixable with `--fix`.
```

Auch unser eigener Code hat Funde. `PLW1510` bemängelt, dass `subprocess.run` ohne die ausdrückliche Angabe `check=...` aufgerufen wird: Die Regel will, dass man bewusst entscheidet, ob ein Fehlercode eine Ausnahme auslösen soll. Wir prüfen den Fehlercode selbst, also ist `check=False` die ehrliche Angabe. `SIM117` schlägt in den Tests vor, geschachtelte `with`-Blöcke zu einem zusammenzufassen, und `I001` betrifft die Importe in einer Testdatei. Zwei Beobachtungen sind wichtig. Erstens ist die Menge der Regeln, die `ruff` *ohne Konfiguration* anwendet, nicht festgeschrieben: Sie steht in der Dokumentation und hat sich zwischen Versionen geändert. In der hier verwendeten Version 0.17.4 gehören, wie die Funde zeigen, auch Regeln aus den Gruppen `B`, `SIM`, `S`, `I` und `PLW` dazu; in früheren Versionen waren es deutlich weniger. Zweitens bedeutet das: Ein Projekt, das sich auf die Standardwerte verlässt, bekommt bei einem Versionswechsel unter Umständen plötzlich neue Funde. Man legt die gewünschten Regeln deshalb ausdrücklich fest.

**Konfiguration in der pyproject.toml.** Die Einstellungen von `ruff` stehen im Abschnitt `[tool.ruff]`. Dort legt man allgemeine Dinge fest, unter `[tool.ruff.lint]` die Regeln des Linters, und unter `[tool.ruff.format]` die Einstellungen des Formatters. Wir setzen die maximale Zeilenlänge auf 100 Zeichen, sagen `ruff` mit `src`, wo der eigene Code liegt (damit es beim Sortieren von Importen eigene Pakete von fremden unterscheiden kann), und wählen mit `select` die Regelgruppen: `E` (Stilfehler nach `pycodestyle`), `F` (echte Fehler wie unbenutzte Namen), `I` (Sortierung der Importe), `B` (häufige Fallen, *bugbear*), `UP` (veraltete Schreibweisen, die neuere Python-Versionen besser ausdrücken), `SIM` (Vereinfachungen) und `PLW` (Warnungen aus `pylint`). Mit `per-file-ignores` schalten wir die Regel `SIM117` für die Tests aus: Geschachtelte `with`-Blöcke sind dort lesbarer als die zusammengezogene Schreibweise.

```pycon
(.venv) ~/plattenwacht$ printf '\n[tool.ruff]\nline-length = 100\nsrc = ["src", "tests"]\n\n[tool.ruff.lint]\nselect = ["E", "F", "I", "B", "UP", "SIM", "PLW"]\n\n[tool.ruff.lint.per-file-ignores]\n"tests/*" = ["SIM117"]\n' >> pyproject.toml
(.venv) ~/plattenwacht$ tail -9 pyproject.toml
[tool.ruff]
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "PLW"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["SIM117"]
(.venv) ~/plattenwacht$ ruff check --output-format=concise beispiel src tests
beispiel/alt.py:1:1: E401 [*] Multiple imports on one line
beispiel/alt.py:1:1: I001 [*] Import block is un-sorted or un-formatted
beispiel/alt.py:1:12: F401 [*] `sys` imported but unused
beispiel/alt.py:3:21: F401 [*] `pathlib.Path` imported but unused
beispiel/alt.py:5:22: B006 Do not use mutable data structures for argument defaults
beispiel/alt.py:6:8: E712 Avoid equality comparisons to `True`; use `os.path.exists(pfad):` for truth checks
beispiel/alt.py:7:13: SIM115 Use a context manager for opening files
beispiel/alt.py:10:5: F841 Local variable `unbenutzt` is assigned to but never used
beispiel/alt.py:11:5: SIM105 Use `contextlib.suppress(BaseException)` instead of `try`-`except`-`pass`
beispiel/alt.py:12:9: F841 Local variable `x` is assigned to but never used
beispiel/alt.py:13:5: E722 Do not use bare `except`
src/plattenwacht/befehle.py:9:20: PLW1510 `subprocess.run` without explicit `check` argument
src/plattenwacht/cli.py:19:101: E501 Line too long (110 > 100)
src/plattenwacht/cli.py:20:101: E501 Line too long (113 > 100)
tests/test_befehle.py:10:101: E501 Line too long (104 > 100)
tests/test_belegung.py:9:101: E501 Line too long (102 > 100)
tests/test_belegung.py:14:101: E501 Line too long (101 > 100)
Found 17 errors.
[*] 4 fixable with `--fix` (5 unsafe fixes available with `--fix --unsafe-fixes`).
```

Mit der ausdrücklichen Auswahl erscheinen jetzt auch `E401` (mehrere Importe in einer Zeile) und `E712` (Vergleich mit `True`, den man weglässt: `if os.path.exists(pfad):`) sowie `SIM105` (statt `try` und `pass` lieber `contextlib.suppress`); die Regel `S110` ist verschwunden, weil die Gruppe `S` (Sicherheit) nicht mehr gewählt ist. Das ist die Konsequenz der Auswahl: `select` ersetzt die Standardregeln, es ergänzt sie nicht. Im Projekt selbst bleiben `PLW1510` und mehrere `E501`-Funde (Zeile zu lang), die `SIM117`- und `I001`-Funde sind verschwunden.

**Automatisch korrigieren: –fix.** Viele Funde kann `ruff` selbst beheben. Das Zeichen `[*]` in der Ausgabe markiert sie. Mit `ruff check --fix` werden sie angewendet. Daneben gibt es *unsichere* Korrekturen (*unsafe fixes*), die das Verhalten des Programms ändern könnten; `ruff` wendet sie nur mit `--unsafe-fixes` an und zeigt sie sonst als Hinweis. Ein Beispiel ist die Entfernung einer unbenutzten Variablen: Steht rechts vom Gleichheitszeichen ein Aufruf mit Nebenwirkung, würde er mit der Zuweisung verschwinden. Prüfen Sie nach `--fix` immer mit `git diff`, was geändert wurde, und führen Sie die Tests aus.

```pycon
(.venv) ~/plattenwacht$ ruff check --fix --output-format=concise beispiel
beispiel/alt.py:5:22: B006 Do not use mutable data structures for argument defaults
beispiel/alt.py:6:8: E712 Avoid equality comparisons to `True`; use `os.path.exists(pfad):` for truth checks
beispiel/alt.py:7:13: SIM115 Use a context manager for opening files
beispiel/alt.py:10:5: F841 Local variable `unbenutzt` is assigned to but never used
beispiel/alt.py:11:5: SIM105 Use `contextlib.suppress(BaseException)` instead of `try`-`except`-`pass`
beispiel/alt.py:12:9: F841 Local variable `x` is assigned to but never used
beispiel/alt.py:13:5: E722 Do not use bare `except`
Found 11 errors (4 fixed, 7 remaining).
5 unsafe fixes available with `--fix --unsafe-fixes`.
(.venv) ~/plattenwacht$ head -6 beispiel/alt.py
import json
import os


def lade(pfad, daten=[]):
    if os.path.exists(pfad) == True:
```

Vier Funde wurden behoben: Die Importe sind aufgeteilt und sortiert, die unbenutzten Namen sind entfernt. Übrig bleiben die Funde, die eine Entscheidung brauchen. Das ist typisch: Der Linter beseitigt Mechanisches, der Mensch löst Inhaltliches. Wir bessern den Rest von Hand und schreiben die Datei so, wie sie sein sollte. Die Datei wird in einem `with`-Block geöffnet, mit Kodierung gelesen, der veränderliche Standardwert durch `None` ersetzt, der Vergleich mit `True` entfällt, und der leere `except` fängt nur noch den Fehler, den wir wirklich erwarten.

**Datei** `beispiel/alt.py`**:**

```python
import json
import os


def lade(pfad, daten=None):
    if daten is None:
        daten = []
    if os.path.exists(pfad):
        with open(pfad, encoding="utf-8") as datei:
            text = datei.read()
        daten.append(text)
        try:
            json.loads(text)
        except json.JSONDecodeError:
            daten.append("(kein gueltiges JSON)")
    return daten
```

```pycon
(.venv) ~/plattenwacht$ ruff check --output-format=concise beispiel
All checks passed!
```

**Der Formatter.** Der Linter prüft Inhalt, der Formatter Optik. Der Befehl `ruff format` schreibt den Quelltext nach festen Regeln um: einheitliche Anführungszeichen, zwei Leerzeilen zwischen Funktionen, eine Leerzeile nach dem Modulkommentar, Umbruch zu langer Zeilen. Entscheidend ist, dass es *keine Wahl* gibt: Der Formatter entscheidet, und niemand diskutiert mehr, wo der Zeilenumbruch hingehört. Wir prüfen zuerst, ohne etwas zu ändern. Die Option `--check` meldet, welche Dateien sich ändern würden (Exit-Code 1), und `--diff` zeigt die Änderungen als Vergleich. Wir sehen uns eine Datei an.

```pycon
(.venv) ~/plattenwacht$ ruff format --check src tests beispiel | tail -1
7 files would be reformatted, 9 files already formatted
(.venv) ~/plattenwacht$ ruff format --check src tests beispiel > /dev/null; echo "Exit-Code: $?"
Exit-Code: 1
(.venv) ~/plattenwacht$ ruff format --diff src/plattenwacht/meldung.py
--- src/plattenwacht/meldung.py
+++ src/plattenwacht/meldung.py
@@ -1,4 +1,5 @@
 """Meldungen an einen Webhook senden."""
+
 import json
 import urllib.request
 
@@ -6,8 +7,6 @@
 def sende_webhook(url: str, text: str, timeout: float = 5) -> int:
     """Sendet text als JSON an url und gibt den HTTP-Statuscode zurueck."""
     daten = json.dumps({"text": text}).encode("utf-8")
-    anfrage = urllib.request.Request(
-        url, data=daten, headers={"Content-Type": "application/json"}
-    )
+    anfrage = urllib.request.Request(url, data=daten, headers={"Content-Type": "application/json"})
     with urllib.request.urlopen(anfrage, timeout=timeout) as antwort:
         return int(antwort.status)

1 file would be reformatted
```

Der Formatter möchte nach dem Modulkommentar eine Leerzeile einfügen und bricht beziehungsweise verbindet Zeilen, wo es den 100 Zeichen entspricht (in `meldung.py` passt der Aufruf von `Request` jetzt in eine Zeile). Das ist reine Optik, der Code bleibt gleichwertig. Wir lassen es geschehen, prüfen mit den Tests und sichern in Git. Die Formatierung committet man *getrennt* von inhaltlichen Änderungen, damit der Commit leicht zu überfliegen ist.

```pycon
(.venv) ~/plattenwacht$ ruff format src tests beispiel
7 files reformatted, 9 files left unchanged
(.venv) ~/plattenwacht$ ruff format --check src tests; echo "Exit-Code: $?"
15 files already formatted
Exit-Code: 0
(.venv) ~/plattenwacht$ pytest -q
...............................                                          [100%]
31 passed in 0.01s
(.venv) ~/plattenwacht$ git diff --stat
 pyproject.toml               | 10 ++++++++++
 src/plattenwacht/__main__.py |  1 +
 src/plattenwacht/befehle.py  |  1 +
 src/plattenwacht/cli.py      |  9 +++++++--
 src/plattenwacht/core.py     |  1 +
 src/plattenwacht/meldung.py  |  5 ++---
 tests/test_befehle.py        |  4 +++-
 tests/test_belegung.py       |  8 ++++++--
 8 files changed, 31 insertions(+), 8 deletions(-)
```

Das Ergebnis zeigt, welche Dateien neu formatiert wurden, und die Tests bestätigen, dass das Verhalten gleich geblieben ist; `git diff --stat` fasst den Umfang zusammen. Das Beispielverzeichnis hat seinen Zweck erfüllt und wird gelöscht. Dann sichern wir die Konfiguration und die reine Umformatierung in einem eigenen Commit.

```pycon
(.venv) ~/plattenwacht$ rm -r beispiel
(.venv) ~/plattenwacht$ git add pyproject.toml src tests
(.venv) ~/plattenwacht$ git commit -q -m "ruff konfigurieren und Code formatieren"
(.venv) ~/plattenwacht$ git status --short
```

Jetzt beheben wir den Fund `PLW1510` im Projekt. Die Änderung ist klein: Wir geben in `df_belegung` mit `check=False` ausdrücklich an, dass wir den Fehlercode selbst auswerten. Nach der Änderung führen wir die Tests aus, denn eine vermeintlich harmlose Korrektur kann etwas anderes brechen.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/            text=True,/            text=True,\n            check=False,/' src/plattenwacht/befehle.py
(.venv) ~/plattenwacht$ pytest -q 2>&1 | grep -E "^E  |[0-9] (passed|failed)"
E           AssertionError: expected call not found.
E           Expected: run(['df', '--output=pcent', '/var'], capture_output=True, text=True, timeout=10)
E             Actual: run(['df', '--output=pcent', '/var'], capture_output=True, text=True, check=False, timeout=10)
1 failed, 30 passed in 0.01s
```

Der Test schlägt fehl, und das ist lehrreich: Unser Test aus 14.3 prüft mit `assert_called_once_with`, dass `subprocess.run` *genau* mit den erwarteten Argumenten aufgerufen wird, und das neue Argument gehört nicht dazu. Das ist kein Fehler im Code, sondern ein Hinweis darauf, dass der Test das Verhalten (den Aufruf) sehr eng festschreibt, was bei Mock-Tests der Preis der Genauigkeit ist. Wir passen die Erwartung an.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/capture_output=True, text=True, timeout=10/capture_output=True, text=True, check=False, timeout=10/' tests/test_befehle.py
(.venv) ~/plattenwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/plattenwacht$ ruff format --check; echo "Exit-Code: $?"
16 files already formatted
Exit-Code: 0
(.venv) ~/plattenwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/plattenwacht$ pytest -q
...............................                                          [100%]
31 passed in 0.01s
```

Alle vier Prüfungen sind sauber: `ruff check`, `ruff format`, `mypy` und `pytest`. Diese Kette gilt als das Mindestmaß an Qualitätssicherung für ein Python-Projekt. Wir sichern sie in einem zweiten Commit, getrennt von der Formatierung.

```pycon
(.venv) ~/plattenwacht$ git add src tests
(.venv) ~/plattenwacht$ git commit -q -m "df_belegung: check=False ausdruecklich angeben"
(.venv) ~/plattenwacht$ git status --short
(.venv) ~/plattenwacht$ git log --oneline -3
14b5f6f df_belegung: check=False ausdruecklich angeben
f4ba53d ruff konfigurieren und Code formatieren
712d569 Typangaben fuer alle Module; mypy im strict-Modus
```

**Prüfungen automatisch erzwingen: ein Git-Hook.** Prüfungen helfen nur, wenn man sie ausführt. Git bietet dafür *Hooks* an: Skripte im Verzeichnis `.git/hooks`, die Git bei bestimmten Ereignissen aufruft. Ein Skript namens `pre-commit` läuft vor jedem Commit; beendet es sich mit einem Fehlercode, bricht Git den Commit ab. Wir schreiben einen einfachen Hook, der Linter und Formatprüfung ausführt. Er muss ausführbar sein. Beachten Sie, dass Hooks im Verzeichnis `.git` liegen und deshalb *nicht* mit dem Repository weitergegeben werden; für Teams gibt es dafür das Werkzeug `pre-commit`, das die Hooks in einer versionierten Konfigurationsdatei beschreibt (hier nicht ausgeführt).

**Datei** `.git/hooks/pre-commit`**:**

```text
#!/bin/sh
# Bricht den Commit ab, wenn ruff Probleme meldet.
ruff check --quiet || exit 1
ruff format --check --quiet || exit 1
```

Wir testen den Hook, indem wir absichtlich einen schlechten Import einbauen, und versuchen zu committen. Der Hook findet die Datei `ruff` nur, wenn die virtuelle Umgebung aktiv ist; sie ist es hier.

```pycon
(.venv) ~/plattenwacht$ printf 'import os\n' >> src/plattenwacht/__init__.py
(.venv) ~/plattenwacht$ git commit -q -am "unbenutzten Import hinzufuegen"; echo "Exit-Code: $?"
F401 `os` imported but unused
 --> src/plattenwacht/__init__.py:4:8
  |
3 | __version__ = "0.1.0"
4 | import os
  |        ^^
help: Remove unused import: `os`

Exit-Code: 1
```

Der Commit wurde abgelehnt, und `ruff` nennt den Grund. Wir machen die Änderung rückgängig; ein sauberer Commit würde den Hook ohne jede Meldung passieren, und die leere Ausgabe des Status zeigt, dass alles wieder unverändert ist.

```pycon
(.venv) ~/plattenwacht$ git restore src/plattenwacht/__init__.py
(.venv) ~/plattenwacht$ git status --short
```

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, Meldungen pauschal abzuschalten, statt sie zu verstehen. Mit `ignore = ["E501"]` oder einem Kommentar `# noqa` am Zeilenende lässt sich jede Regel zum Schweigen bringen; das ist ein legitimes Mittel für begründete Ausnahmen, wird aber schnell zum Dauerzustand, in dem der Linter nichts mehr sagt. Schreiben Sie bei jeder Ausnahme dazu, warum sie nötig ist, und begrenzen Sie sie auf die kleinste Stelle (eine Zeile, eine Datei, wie oben bei den Tests). Ein zweiter Fehler ist der Streit zwischen Linter und Formatter: Wählt man Regeln wie `E501` (Zeilenlänge) und nutzt zugleich den Formatter, sollten beide dieselbe Zeilenlänge verwenden, was die gemeinsame Einstellung `line-length` sicherstellt. Drittens verändern `--fix` und `ruff format` *Dateien*: Wer sie auf einem Stand mit ungesicherten Änderungen ausführt, vermischt Arbeit und automatische Korrekturen. Committen Sie vorher, damit `git diff` eindeutig zeigt, was das Werkzeug getan hat. Viertens hängen die Funde von der Version ab; wer in der Teamarbeit unterschiedliche Versionen benutzt, bekommt unterschiedliche Ergebnisse. Man legt deshalb eine Version fest (zum Beispiel `ruff==` in der Gruppe `dev`) oder aktualisiert bewusst. Fünftens ersetzt ein sauberer Linter keine Tests: Er findet Stilprobleme und häufige Fehlermuster, aber keine Denkfehler.

**Selbsttest.** Beantworten Sie zunächst ohne Nachschlagen: Was unterscheidet einen Linter von einem Formatter? Warum gibt man die gewünschten Regeln ausdrücklich an, statt sich auf die Standardwerte zu verlassen? Was bedeutet `[*]` in der Ausgabe von `ruff check`, und wann ist ein Fund *nicht* automatisch behebbar? Welche vier Befehle bilden die Mindestkette der Qualitätssicherung? Wozu dient ein `pre-commit`-Hook, und weshalb reicht er allein nicht aus? Die Antworten: Der Linter sucht Fehler und Auffälligkeiten im Inhalt, der Formatter vereinheitlicht die Darstellung; die Standardregeln ändern sich mit der Version, und ausdrückliche Regeln machen die Ergebnisse reproduzierbar; `[*]` markiert Funde, die `ruff` selbst beheben kann, unsichere Korrekturen und Funde, die Entscheidungen verlangen, bleiben dem Menschen; die Kette lautet `ruff check`, `ruff format --check`, `mypy` und `pytest`; der Hook verhindert, dass man schlechten Code aus Versehen committet, wird aber nicht mit dem Repository verteilt und lässt sich mit `git commit --no-verify` umgehen, weshalb dieselben Prüfungen zusätzlich auf einem Server laufen sollten.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Konfiguration von `ruff` in der `pyproject.toml` (Abschnitte `[tool.ruff]`, `[tool.ruff.lint]`, `select`, `per-file-ignores`) und die Gliederung der Regeln nach Gruppen mit Kennungen stützen sich auf die abgerufenen Seiten „Configuration“ und „Rules“ der `ruff`-Dokumentation. Alle Funde, Korrekturen und Formatierungen in diesem Abschnitt wurden mit `ruff` 0.17.4 tatsächlich ausgeführt; die Standardregeln und manche Meldungen sind versionsabhängig und können bei Ihnen abweichen. Die Hinweise auf `flake8`, `isort`, `black` als Vorgänger, auf das Werkzeug `pre-commit` und auf Continuous Integration sind Fachwissen und wurden nicht abgerufen.

> **Merke**
>
> Der Linter (`ruff check`) findet Fehlermuster und Stilprobleme, der Formatter (`ruff format`) vereinheitlicht die Optik; beide werden in der `pyproject.toml` konfiguriert. Wählen Sie die Regeln mit `select` ausdrücklich, denn die Standardregeln ändern sich mit der Version. `ruff check --fix` behebt Mechanisches, `--unsafe-fixes` nur mit Prüfung. Die Mindestkette der Qualitätssicherung ist `ruff check`, `ruff format --check`, `mypy`, `pytest`. Ein Hook vor dem Commit erinnert daran, ersetzt aber keine Prüfung auf dem Server.

> **Praxisbeispiel**
>
> In nahezu jedem professionellen Python-Projekt laufen Linter, Formatter, Typprüfung und Tests bei jedem Push automatisch auf einem Server der Versionsverwaltung, in einer sogenannten *Continuous-Integration*-Pipeline (bei GitHub *Actions*, bei GitLab *CI*). Ein Merge in `main` ist erst möglich, wenn die Pipeline grün ist. Für ein Team von Administratoren bedeutet das: Ein Skript, das auf Servern läuft, hat vorher mindestens die vier Prüfungen bestanden, und Stilfragen sind entschieden, bevor jemand den Code liest. Eine solche Pipeline ist im Kern eine Datei, die dieselben vier Befehle in der richtigen Reihenfolge ausführt; sie in Modul 15 selbst zu schreiben, ist ein guter Ausbau des Abschlussprojekts.

## Übungsaufgaben zu Modul 14

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Arbeiten Sie bei den praktischen Aufgaben in einem eigenen Verzeichnis mit aktivierter virtueller Umgebung, in der `pytest`, `mypy` und `ruff` installiert sind (`pip install pytest mypy ruff`). Legen Sie Modul und Testdatei jeweils im selben Verzeichnis ab. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Quiz zu pytest: Nach welchen Regeln findet `pytest` Tests, und was geschieht mit einer Funktion, die diese Regeln verletzt? Was leistet `@pytest.mark.parametrize`, und warum prüft man Grenzwerte von beiden Seiten? Wann braucht man `pytest.approx`, wann `pytest.raises`? Was bedeuten die Exit-Codes 0, 1 und 5 von `pytest`? Warum sind Tests, die von der Reihenfolge abhängen, gefährlich?

**Aufgabe 2.** Quiz zu Fixtures und Mocks: Wie fordert ein Test eine Fixture an, und wofür sind `tmp_path`, `capsys` und `monkeypatch` gedacht? Was trennt bei einer Fixture `yield` von `return`, und was bewirkt `scope="module"` (mit Gefahr)? Warum muss man `patch` dort anwenden, wo ein Objekt *benutzt* wird? Wozu dienen `return_value`, `side_effect` und `autospec=True`? Wann ist ein Test „zu stark gemockt“?

**Aufgabe 3.** Quiz zu Typangaben und ruff: Werden Typangaben zur Laufzeit geprüft, und wer prüft sie dann? Was bedeutet `int | None` für den Aufrufer, und wie behandelt man diesen Fall? Wodurch unterscheiden sich `ruff check` und `ruff format`? Warum legt man die Regeln mit `select` ausdrücklich fest? Was ist der Unterschied zwischen einer sicheren und einer unsicheren automatischen Korrektur? Welche vier Befehle bilden die Mindestkette der Qualitätssicherung?

**Aufgabe 4.** Finden Sie mindestens sechs Fehler oder Probleme in der folgenden Testdatei und erklären Sie jeden. Die Datei liegt im Verzeichnis `tests` des Projekts aus diesem Modul.

```python
import os

from plattenwacht.core import bewerte


def pruefe_grenzen():
    assert bewerte(80) == "WARNUNG"


def test_prozent():
    assert (bewerte(95) == "KRITISCH", "bei 95 kritisch")


def test_summe():
    assert 0.1 + 0.2 == 0.3


def test_umgebung():
    os.environ["PLATTENWACHT_WARNUNG"] = "10"
    assert bewerte(20, warnung=10) == "WARNUNG"


def test_datei():
    f = open("/tmp/test_datei.txt", "w")
    f.write("x")
    assert open("/tmp/test_datei.txt").read() == "x"


def test_nur_aufruf():
    bewerte(50)
```

**Aufgabe 5.** **Code:** Schreiben Sie eine Funktion `prozent_aus_text(text)`, die `"42"`, `"42%"`, `" 42 % "` und `"7,5 %"` in Kommazahlen umwandelt (42.0, 42.0, 42.0, 7.5) und bei Text, der keine Zahl ist, sowie bei Werten außerhalb von 0 bis 100 einen `ValueError` auslöst. Testen Sie sie mit `parametrize`, `pytest.raises` und `pytest.approx`, einschließlich der Grenzwerte 0 und 100 und der Sonderfälle `"nan"` und der leeren Zeichenkette.

**Aufgabe 6.** **Code:** Schreiben Sie eine Funktion `lies_schwellen(pfad)`, die eine Datei mit Zeilen der Form `schluessel=wert` in ein Wörterbuch mit Kommazahlen einliest. Leerzeilen und Zeilen, die mit `#` beginnen, werden ignoriert, eine Zeile ohne `=` löst einen `ValueError` mit der Zeilennummer aus. Testen Sie sie mit `tmp_path` und einer eigenen Fixture, die Dateien mit beliebigem Inhalt anlegt (Fabrik-Fixture), und prüfen Sie auch eine nicht vorhandene Datei.

**Aufgabe 7.** **Code:** Schreiben Sie eine Funktion `ist_dienst_aktiv(name)`, die mit `systemctl is-active --quiet NAME` prüft, ob ein Dienst läuft (Exit-Code 0 heißt aktiv, jeder andere Code inaktiv) und bei fehlendem `systemctl` einen `RuntimeError` auslöst. Testen Sie sie ohne echtes `systemctl` mit `unittest.mock.patch`: Prüfen Sie den aktiven und den inaktiven Fall, den genauen Aufruf mit `assert_called_once_with` und das fehlende Kommando mit `side_effect`.

**Aufgabe 8.** **Code:** Das folgende Modul `statistik.py` besteht aus Funktionen mit mehreren Problemen. Bringen Sie es durch die Qualitätskette: Lassen Sie `ruff` mit einer ausdrücklichen Auswahl der Regelgruppen `E`, `F`, `I`, `B`, `UP` und `SIM` (Zielversion Python 3.11) laufen, beheben Sie die Funde (automatisch, wo es geht), versehen Sie alle Funktionen mit Typangaben, bis `mypy --strict` keine Meldung mehr ausgibt, und schreiben Sie Tests, die auch den Fall einer Division durch null abdecken (die Funktion `anteil` soll dann einen `ValueError` auslösen).

```python
import sys, os
from typing import List, Optional


def mittelwert(werte: List[float]) -> Optional[float]:
    if len(werte) == 0:
        return None
    summe = 0
    for w in werte:
        summe = summe + w
    return summe / len(werte)


def anteil(teil, gesamt):
    return teil / gesamt * 100


def gruppiere(namen, ergebnis={}):
    for n in namen:
        ergebnis[n] = len(n)
    return ergebnis
```

## Lösungen zu Modul 14

**Aufgabe 1.** `pytest` sammelt Dateien mit dem Muster `test_*.py` oder `*_test.py` und darin Funktionen, deren Name mit `test` beginnt (sowie Methoden in Klassen, deren Name mit `Test` beginnt). Eine Funktion, die sich nicht daran hält, wird *stillschweigend übergangen*; deshalb prüft man nach dem Anlegen eines Tests, dass die Zahl der gefundenen Tests gestiegen ist. `@pytest.mark.parametrize` führt dieselbe Testfunktion mit mehreren Wertesätzen aus, und jeder Satz erscheint als eigener Test; das vermeidet Wiederholung und zeigt genau, welcher Wert scheitert. Grenzwerte prüft man von beiden Seiten (79.9 und 80), weil Fehler wie ein `>` statt `>=` genau dort sitzen. `pytest.approx` braucht man beim Vergleich von Kommazahlen, die aus Rechnungen stammen, weil Gleitkommazahlen binär gespeichert werden und `0.1 + 0.2 == 0.3` falsch ist; `pytest.raises` prüft, dass eine bestimmte Ausnahme ausgelöst wird. Der Exit-Code 0 bedeutet, dass alle Tests bestanden haben, 1, dass mindestens einer fehlgeschlagen ist, und 5, dass keine Tests gefunden wurden (meist falsche Namen oder Auswahl). Tests, die von der Reihenfolge abhängen, sind gefährlich, weil `pytest` einzelne Tests oder andere Reihenfolgen ausführen kann; ein solcher Fehler tritt dann scheinbar zufällig auf. Jeder Test muss für sich allein laufen können.

**Aufgabe 2.** Ein Test fordert eine Fixture an, indem er einen Parameter mit ihrem Namen in die Argumentliste schreibt. `tmp_path` liefert ein frisches temporäres Verzeichnis, `capsys` fängt Ausgaben auf Standardausgabe und Fehlerausgabe ab, und `monkeypatch` ändert Umgebungsvariablen, Attribute oder das Arbeitsverzeichnis für die Dauer des Tests und stellt danach alles wieder her. Bei einer Fixture mit `yield` ist der Code vor dem `yield` die Vorbereitung und der Code danach das Aufräumen, das auch bei fehlgeschlagenen Tests läuft; `return` kennt kein Aufräumen. `scope="module"` erzeugt die Fixture einmal je Testdatei; liefert sie einen veränderlichen Wert (Liste, Wörterbuch), können Tests sich über Änderungen gegenseitig beeinflussen. `patch` muss dort ansetzen, wo ein Name *nachgeschlagen* wird, weil `from modul import name` eine zweite Referenz im benutzenden Modul anlegt; ersetzt man das Original im Definitionsmodul, bleibt diese Referenz unberührt. `return_value` legt fest, was ein Aufruf zurückgibt, `side_effect` löst eine Ausnahme aus (oder liefert nacheinander mehrere Antworten), und `autospec=True` übernimmt die Signatur des Originals, sodass falsche Aufrufe einen `TypeError` auslösen. Ein Test ist zu stark gemockt, wenn er so viel ersetzt, dass er nur noch die Mocks prüft: dann sagt er nichts mehr über den echten Code aus. Ersetzen Sie nur die Grenzen zur Außenwelt.

**Aufgabe 3.** Typangaben werden zur Laufzeit nicht geprüft; es prüft ein eigenes Programm, der Typprüfer (zum Beispiel `mypy`), ohne den Code auszuführen. `int | None` bedeutet, dass der Aufrufer eine Zahl oder `None` erhält; er muss den Fall `None` behandeln, bevor er mit dem Wert rechnet, etwa mit `if wert is not None:`, wonach der Prüfer den Typ „einengt“. `ruff check` ist der Linter und sucht Fehlermuster und Stilprobleme im Inhalt, `ruff format` ist der Formatter und vereinheitlicht die Darstellung (Umbrüche, Anführungszeichen, Leerzeilen). Die Regeln legt man mit `select` fest, weil die Standardregeln nicht festgeschrieben sind und sich mit der Version ändern; mit einer ausdrücklichen Auswahl bleiben Ergebnisse reproduzierbar. Eine sichere Korrektur ändert das Verhalten nicht (Importe sortieren, unbenutzten Import entfernen), eine unsichere könnte es ändern (etwa das Entfernen einer Variablen, hinter der ein Aufruf mit Nebenwirkung steht) und wird nur mit `--unsafe-fixes` angewendet. Die Mindestkette lautet `ruff check`, `ruff format --check`, `mypy` und `pytest`.

**Aufgabe 4.** In der Testdatei stecken folgende Probleme: (1) `pruefe_grenzen` beginnt nicht mit `test`, die Funktion wird nie ausgeführt, und es gibt keinen Hinweis darauf. (2) In `test_prozent` steht die Prüfung in Klammern mit einem Komma, das ist ein Tupel; ein nichtleeres Tupel ist immer wahr, der Test besteht also immer (Python warnt davor). Richtig: `assert bewerte(95) == "KRITISCH", "bei 95 kritisch"`. (3) `test_summe` vergleicht Kommazahlen exakt: `0.1 + 0.2 == 0.3` ist falsch; man nimmt `pytest.approx`. (4) `test_umgebung` setzt `os.environ` direkt, die Variable bleibt für alle folgenden Tests gesetzt; außerdem hat sie auf das Ergebnis gar keinen Einfluss (`bewerte` liest die Umgebung nicht), der Test täuscht also etwas vor. Mit `monkeypatch.setenv` wäre die Änderung automatisch zurückgenommen worden. (5) `test_datei` schreibt in eine feste Datei unter `/tmp`, statt `tmp_path` zu verwenden; parallele oder wiederholte Läufe stören sich dann gegenseitig. Außerdem werden die Dateien nicht geschlossen (kein `with`) und ohne Angabe der Kodierung geöffnet. (6) `test_nur_aufruf` enthält keine Prüfung und besteht immer; er findet nur Abstürze, nicht falsche Ergebnisse. Als Bonus: Die Namen der Tests sagen wenig über die Erwartung (`test_prozent`, `test_summe`), besser wären Namen, die das erwartete Verhalten beschreiben.

**Aufgabe 5.** Die Funktion entfernt Leerraum und ein abschließendes Prozentzeichen, ersetzt das Dezimalkomma durch einen Punkt und wandelt mit `float` um; ungültiger Text löst dabei von selbst einen `ValueError` aus. Danach prüft sie den Bereich. Die Prüfung `not 0 <= wert <= 100` fängt auch `nan` ab, denn Vergleiche mit `nan` sind immer falsch, und unendliche Werte liegen außerhalb des Bereichs. Die Tests sind in zwei Gruppen mit `parametrize` geteilt, eine für gültige und eine für ungültige Eingaben.

```pycon
(.venv) ~/plattenwacht$ mkdir -p ~/uebungen/aufgabe5 && cd ~/uebungen/aufgabe5
```

**Datei** `prozent.py`**:**

```python
def prozent_aus_text(text):
    """Wandelt '42', '42%' oder '7,5 %' in eine Kommazahl um (0 bis 100)."""
    bereinigt = text.strip().removesuffix("%").strip()
    wert = float(bereinigt.replace(",", "."))
    if not 0 <= wert <= 100:
        raise ValueError(f"Prozentwert ausserhalb von 0 bis 100: {wert}")
    return wert
```

**Datei** `test_prozent.py`**:**

```python
import pytest

from prozent import prozent_aus_text


@pytest.mark.parametrize(
    "text, erwartet",
    [
        ("42", 42.0),
        ("42%", 42.0),
        (" 42 % ", 42.0),
        ("7,5 %", 7.5),
        ("0", 0.0),
        ("100", 100.0),
    ],
)
def test_gueltige_eingaben(text, erwartet):
    assert prozent_aus_text(text) == pytest.approx(erwartet)


@pytest.mark.parametrize("text", ["abc", "", "%", "101", "-1", "100,1", "nan", "inf"])
def test_ungueltige_eingaben(text):
    with pytest.raises(ValueError):
        prozent_aus_text(text)
```

```pycon
(.venv) ~/uebungen/aufgabe5$ pytest -v 2>&1 | tail -22
============================= test session starts ==============================
platform linux -- Python 3.13.18, pytest-9.2.0, pluggy-1.7.0 -- /home/max/plattenwacht/.venv/bin/python3
cachedir: .pytest_cache
rootdir: /home/max/uebungen/aufgabe5
collecting ... collected 14 items

test_prozent.py::test_gueltige_eingaben[42-42.0] PASSED                  [  7%]
test_prozent.py::test_gueltige_eingaben[42%-42.0] PASSED                 [ 14%]
test_prozent.py::test_gueltige_eingaben[ 42 % -42.0] PASSED              [ 21%]
test_prozent.py::test_gueltige_eingaben[7,5 %-7.5] PASSED                [ 28%]
test_prozent.py::test_gueltige_eingaben[0-0.0] PASSED                    [ 35%]
test_prozent.py::test_gueltige_eingaben[100-100.0] PASSED                [ 42%]
test_prozent.py::test_ungueltige_eingaben[abc] PASSED                    [ 50%]
test_prozent.py::test_ungueltige_eingaben[] PASSED                       [ 57%]
test_prozent.py::test_ungueltige_eingaben[%] PASSED                      [ 64%]
test_prozent.py::test_ungueltige_eingaben[101] PASSED                    [ 71%]
test_prozent.py::test_ungueltige_eingaben[-1] PASSED                     [ 78%]
test_prozent.py::test_ungueltige_eingaben[100,1] PASSED                  [ 85%]
test_prozent.py::test_ungueltige_eingaben[nan] PASSED                    [ 92%]
test_prozent.py::test_ungueltige_eingaben[inf] PASSED                    [100%]

============================== 14 passed in 0.01s ==============================
```

Alle 14 Fälle bestehen. Beachten Sie die Grenzfälle: Die Werte 0 und 100 sind gültig, 101 und `-1` ungültig, und der Fall `"100,1"` prüft, dass das Dezimalkomma nicht an der Bereichsprüfung vorbeischmuggelt. Ein häufiger Fehler bei dieser Aufgabe ist, die Fehlerfälle nur mit `try` und `except` zu prüfen, ohne `pytest.raises`: Wird dann keine Ausnahme ausgelöst, besteht der Test trotzdem.

**Aufgabe 6.** Die Funktion liest die Datei, nummeriert die Zeilen ab 1, überspringt leere Zeilen und Kommentare und zerlegt jede Zeile mit `partition("=")` in Schlüssel, Trenner und Wert. Ist der Trenner leer, fehlte das Gleichheitszeichen. Die Fabrik-Fixture `schreibe` gibt eine Funktion zurück, mit der jeder Test beliebig viele Dateien im temporären Verzeichnis anlegt.

```pycon
(.venv) ~/uebungen/aufgabe5$ mkdir -p ~/uebungen/aufgabe6 && cd ~/uebungen/aufgabe6
```

**Datei** `schwellen.py`**:**

```python
from pathlib import Path


def lies_schwellen(pfad):
    """Liest Zeilen der Form 'schluessel=wert'; Kommentare (#) und Leerzeilen werden ignoriert."""
    ergebnis = {}
    text = Path(pfad).read_text(encoding="utf-8")
    for nummer, zeile in enumerate(text.splitlines(), start=1):
        zeile = zeile.strip()
        if not zeile or zeile.startswith("#"):
            continue
        schluessel, trenner, wert = zeile.partition("=")
        if not trenner:
            raise ValueError(f"Zeile {nummer}: '=' fehlt")
        ergebnis[schluessel.strip()] = float(wert)
    return ergebnis
```

**Datei** `test_schwellen.py`**:**

```python
import pytest

from schwellen import lies_schwellen


@pytest.fixture
def schreibe(tmp_path):
    """Fabrik: legt eine Datei mit dem gegebenen Inhalt an und gibt den Pfad zurueck."""

    def _schreibe(inhalt, name="schwellen.cfg"):
        pfad = tmp_path / name
        pfad.write_text(inhalt, encoding="utf-8")
        return pfad

    return _schreibe


def test_normale_datei(schreibe):
    pfad = schreibe("warnung=70\nkritisch = 85.5\n")
    assert lies_schwellen(pfad) == {"warnung": 70.0, "kritisch": 85.5}


def test_kommentare_und_leerzeilen_werden_ignoriert(schreibe):
    pfad = schreibe("# Schwellen\n\nwarnung=70\n   \n# Ende\n")
    assert lies_schwellen(pfad) == {"warnung": 70.0}


def test_fehlendes_gleichheitszeichen_nennt_die_zeile(schreibe):
    pfad = schreibe("warnung=70\nkritisch 85\n")
    with pytest.raises(ValueError, match="Zeile 2"):
        lies_schwellen(pfad)


def test_keine_zahl_ist_ein_fehler(schreibe):
    with pytest.raises(ValueError):
        lies_schwellen(schreibe("warnung=viel\n"))


def test_nicht_vorhandene_datei(tmp_path):
    with pytest.raises(FileNotFoundError):
        lies_schwellen(tmp_path / "gibt_es_nicht.cfg")


def test_relativer_pfad_im_arbeitsverzeichnis(schreibe, monkeypatch, tmp_path):
    schreibe("warnung=1\n")
    monkeypatch.chdir(tmp_path)
    assert lies_schwellen("schwellen.cfg") == {"warnung": 1.0}
```

```pycon
(.venv) ~/uebungen/aufgabe6$ pytest -q
......                                                                   [100%]
6 passed in 0.01s
```

Die Fabrik-Fixture besteht aus einer Fixture, die eine *innere Funktion* zurückgibt; die Tests rufen sie mit dem gewünschten Inhalt auf. Der Fehlertext enthält die Zeilennummer, und `match="Zeile 2"` prüft das. Der letzte Test zeigt das Zusammenspiel von Fixture und `monkeypatch`: Erst danach wechselt das Arbeitsverzeichnis, und der relative Pfad wird aufgelöst.

**Aufgabe 7.** Die Funktion startet `systemctl` ohne Ausgabe (`--quiet`) und wertet nur den Exit-Code aus. Ein fehlendes Programm zeigt sich als `FileNotFoundError`, das wir in einen `RuntimeError` mit klarer Meldung übersetzen. Die Tests ersetzen `subprocess.run` im Modul `dienst`, wo es aufgerufen wird.

```pycon
(.venv) ~/uebungen/aufgabe6$ mkdir -p ~/uebungen/aufgabe7 && cd ~/uebungen/aufgabe7
```

**Datei** `dienst.py`**:**

```python
import subprocess


def ist_dienst_aktiv(name):
    """True, wenn systemctl den Dienst als aktiv meldet (Exit-Code 0)."""
    try:
        ergebnis = subprocess.run(["systemctl", "is-active", "--quiet", name], check=False)
    except FileNotFoundError as fehler:
        raise RuntimeError("systemctl nicht gefunden") from fehler
    return ergebnis.returncode == 0
```

**Datei** `test_dienst.py`**:**

```python
import subprocess
from unittest.mock import patch

import pytest

from dienst import ist_dienst_aktiv


def antwort(code):
    return subprocess.CompletedProcess(args=[], returncode=code)


def test_aktiver_dienst_und_genauer_aufruf():
    with patch("dienst.subprocess.run", autospec=True, return_value=antwort(0)) as run:
        assert ist_dienst_aktiv("sshd") is True
    run.assert_called_once_with(["systemctl", "is-active", "--quiet", "sshd"], check=False)


def test_inaktiver_dienst():
    with patch("dienst.subprocess.run", return_value=antwort(3)):
        assert ist_dienst_aktiv("sshd") is False


def test_fehlendes_systemctl():
    with patch("dienst.subprocess.run", side_effect=FileNotFoundError("systemctl")):
        with pytest.raises(RuntimeError, match="nicht gefunden"):
            ist_dienst_aktiv("sshd")
```

```pycon
(.venv) ~/uebungen/aufgabe7$ pytest -q
...                                                                      [100%]
3 passed in 0.01s
```

Mit `autospec=True` prüft das Mock beim ersten Test zusätzlich, dass der Aufruf zur echten Signatur von `subprocess.run` passt. Der Rückgabewert ist ein echtes `CompletedProcess`, damit die Funktion genau das Feld `returncode` vorfindet, das sie benutzt. Typische Fehler bei dieser Aufgabe sind das Patchen von `subprocess.run` statt `dienst.subprocess.run` (funktioniert hier zufällig, weil `subprocess` als Modul importiert ist, nicht aber bei `from subprocess import run`) und das Vergessen des Falls, dass das Kommando fehlt.

**Aufgabe 8.** Wir legen das Modul und eine Konfiguration an und lassen `ruff` mit den gewünschten Regeln darüber laufen. Die Zielversion Python 3.11 legt fest, welche modernen Schreibweisen die Gruppe `UP` verlangen darf.

```pycon
(.venv) ~/uebungen/aufgabe7$ mkdir -p ~/uebungen/aufgabe8 && cd ~/uebungen/aufgabe8
```

**Datei** `statistik.py`**:**

```python
import sys, os
from typing import List, Optional


def mittelwert(werte: List[float]) -> Optional[float]:
    if len(werte) == 0:
        return None
    summe = 0
    for w in werte:
        summe = summe + w
    return summe / len(werte)


def anteil(teil, gesamt):
    return teil / gesamt * 100


def gruppiere(namen, ergebnis={}):
    for n in namen:
        ergebnis[n] = len(n)
    return ergebnis
```

**Datei** `pyproject.toml`**:**

```text
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]
```

```pycon
(.venv) ~/uebungen/aufgabe8$ ruff check --output-format=concise statistik.py
statistik.py:1:1: E401 [*] Multiple imports on one line
statistik.py:1:1: I001 [*] Import block is un-sorted or un-formatted
statistik.py:1:8: F401 [*] `sys` imported but unused
statistik.py:1:13: F401 [*] `os` imported but unused
statistik.py:2:1: UP035 `typing.List` is deprecated, use `list` instead
statistik.py:5:23: UP006 [*] Use `list` instead of `List` for type annotation
statistik.py:5:39: UP045 [*] Use `X | None` for type annotations
statistik.py:18:31: B006 Do not use mutable data structures for argument defaults
Found 8 errors.
[*] 6 fixable with `--fix` (1 unsafe fix available with `--fix --unsafe-fixes`).
(.venv) ~/uebungen/aufgabe8$ mypy --strict statistik.py
statistik.py:10: error: Incompatible types in assignment (expression has type "float", variable has type "int")  [assignment]
statistik.py:14: error: Function is missing a type annotation  [no-untyped-def]
statistik.py:18: error: Function is missing a type annotation  [no-untyped-def]
Found 3 errors in 1 file (checked 1 source file)
```

`ruff` findet die mehrfachen Importe in einer Zeile (`E401`), die unbenutzten Namen `sys` und `os` (`F401`), die veralteten Typangaben aus `typing` (`UP`: `List` und `Optional` lassen sich durch `list` und `X | None` ersetzen), die Reihenfolge der Importe und den veränderlichen Standardwert `ergebnis={}` (`B006`). `mypy --strict` meldet, dass `anteil` und `gruppiere` keine Typangaben haben, und außerdem einen echten Fehler in `mittelwert`: Die Variable `summe` wird mit `0` als ganze Zahl angelegt, bekommt aber später eine Kommazahl zugewiesen (Zeile 10). Das ist in Python harmlos, zeigt aber, dass der Typprüfer auch Dinge findet, die der Linter nicht sieht. Unsere Lösung ersetzt die Schleife durch `sum(werte)`, wodurch das Problem verschwindet. Wir wenden zuerst die automatischen Korrekturen an und sehen, was übrig bleibt.

```pycon
(.venv) ~/uebungen/aufgabe8$ ruff check --fix --output-format=concise statistik.py
statistik.py:16:31: B006 Do not use mutable data structures for argument defaults
Found 8 errors (7 fixed, 1 remaining).
1 unsafe fix available with `--fix --unsafe-fixes`.
(.venv) ~/uebungen/aufgabe8$ head -8 statistik.py


def mittelwert(werte: list[float]) -> float | None:
    if len(werte) == 0:
        return None
    summe = 0
    for w in werte:
        summe = summe + w
```

Die sicheren Korrekturen haben Importe entfernt und sortiert und die Typangaben modernisiert. Der veränderliche Standardwert bleibt, denn seine Korrektur ist *unsicher*: Sie ändert das Verhalten (und das ist hier ja beabsichtigt, aber `ruff` kann es nicht wissen). Wir schreiben das Modul selbst richtig, mit Typangaben, einer Prüfung auf Division durch null und ohne geteilten Standardwert.

**Datei** `statistik.py`**:**

```python
def mittelwert(werte: list[float]) -> float | None:
    if not werte:
        return None
    return sum(werte) / len(werte)


def anteil(teil: float, gesamt: float) -> float:
    if gesamt == 0:
        raise ValueError("gesamt darf nicht 0 sein")
    return teil / gesamt * 100


def gruppiere(namen: list[str], ergebnis: dict[str, int] | None = None) -> dict[str, int]:
    if ergebnis is None:
        ergebnis = {}
    for name in namen:
        ergebnis[name] = len(name)
    return ergebnis
```

**Datei** `test_statistik.py`**:**

```python
import pytest

from statistik import anteil, gruppiere, mittelwert


def test_mittelwert():
    assert mittelwert([1, 2, 3, 4]) == pytest.approx(2.5)


def test_mittelwert_der_leeren_liste_ist_none():
    assert mittelwert([]) is None


def test_anteil():
    assert anteil(25, 200) == pytest.approx(12.5)


def test_anteil_durch_null_ist_ein_fehler():
    with pytest.raises(ValueError, match="nicht 0"):
        anteil(1, 0)


def test_gruppiere_teilt_keinen_zustand_zwischen_aufrufen():
    erste = gruppiere(["ab"])
    zweite = gruppiere(["xyz"])
    assert erste == {"ab": 2}
    assert zweite == {"xyz": 3}
```

```pycon
(.venv) ~/uebungen/aufgabe8$ ruff check --output-format=concise
All checks passed!
(.venv) ~/uebungen/aufgabe8$ ruff format --check; echo "Exit-Code: $?"
2 files already formatted
Exit-Code: 0
(.venv) ~/uebungen/aufgabe8$ mypy --strict statistik.py
Success: no issues found in 1 source file
(.venv) ~/uebungen/aufgabe8$ pytest -q
.....                                                                    [100%]
5 passed in 0.01s
```

Der Formatter ist ebenfalls zufrieden, der Typprüfer meldet keine Probleme, und die Tests bestehen. Der letzte Test ist ein *Regressionstest* für den ursprünglichen Fehler: Mit dem alten, geteilten Standardwert hätte die zweite Gruppierung auch den Schlüssel `"ab"` enthalten. Fehler, die man einmal gemacht hat, sichert man so mit einem Test ab.
