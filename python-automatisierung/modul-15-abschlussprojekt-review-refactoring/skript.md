# Modul 15 - Abschlussprojekt, Review, Refactoring

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (15.1 bis 15.5), danach folgen „Übungsaufgaben zu Modul 15“ (8 Aufgaben) und „Lösungen zu Modul 15“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 22.02.2027 bis Fr 26.02.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 3 (Annahme: Der Lehrplan nennt keine Priorität; die Module 13 bis 15 bilden die Phase „Qualität und Abschluss“)

**Meilenstein:** Abschlussprojekt aus dem Alltag

**Lernziele.** Nach diesem Modul können Sie ein eigenes Werkzeug aus dem Admin-Alltag von der Idee bis zur Auslieferung entwickeln: die Aufgabe eingrenzen und in Module zerlegen, eine Konfiguration und eine Kommandozeile entwerfen, in kleinen Schritten mit Tests und Git umsetzen, den eigenen Code mit Linter, Typprüfer und Testabdeckung prüfen, einen Code-Review durchführen und Rückmeldungen einarbeiten, Code umbauen, ohne sein Verhalten zu ändern (Refactoring), und das Ergebnis mit Dokumentation, Versionsnummer und Release-Tag übergeben. Das Modul führt dazu ein vollständiges Beispielprojekt vor, das alle früheren Module zusammenführt; Sie wählen dann ein eigenes Thema und gehen denselben Weg.

| **Nr.** | **Lerneinheit**                    | **Datum**     | **Dauer** | **Prio** |
|:--------|:-----------------------------------|:--------------|:----------|:---------|
| 15.1    | Abschlussprojekt: Planung          | Mo 22.02.2027 | 2 h       | 3        |
| 15.2    | Abschlussprojekt: Umsetzung 1      | Di 23.02.2027 | 2 h       | 3        |
| 15.3    | Abschlussprojekt: Umsetzung 2      | Mi 24.02.2027 | 2 h       | 3        |
| 15.4    | Abschlussprojekt: Tests und Review | Do 25.02.2027 | 2 h       | 3        |
| 15.5    | Refactoring, Doku und Abschluss    | Fr 26.02.2027 | 2 h       | 3        |
|         | **Summe**                          |               | **10 h**  |          |

> **Merke**
>
> Das Beispielprojekt dieses Moduls heißt `backupwacht` und prüft, ob Sicherungen von Servern vorhanden, frisch und nicht verdächtig klein sind. Es verwendet, was Sie in den Modulen 1 bis 14 gelernt haben: Dateien und Pfade (4), Argumente und Exit-Codes (5, 6), Protokollierung (6), Konfigurationsdateien (7), Zeit (8), zeitgesteuerten Betrieb (9), Meldungen (12), Git und Projektstruktur (13) sowie Tests und Qualitätswerkzeuge (14). Sie brauchen Python 3.11 oder neuer, Git und Internetzugang für die Installation von Werkzeugen. Getestet wurde mit Python 3.13.18, Git 2.43.0, pytest 9.2.0, ruff 0.17.4, mypy 2.4.1 und setuptools 84.2.1. Alle Mitschnitte stammen aus echten Läufen; Kennungen von Commits und Zeitangaben hängen von Ihrem Rechner ab.

## 15.1 Abschlussprojekt: Planung

Alles bisher Gelernte hat Einzelteile geübt: Dateien lesen, Prozesse starten, Meldungen senden, Tests schreiben. Ein Abschlussprojekt prüft etwas anderes, nämlich ob Sie aus diesen Teilen ein *Ganzes* bauen können, das ein echtes Problem löst, das jemand anderes ohne Ihre Hilfe benutzen kann und das Sie in einem halben Jahr noch verstehen. Der Teil, an dem die meisten Projekte scheitern, ist nicht das Programmieren, sondern das *Planen*: Ohne klares Ziel wächst ein Skript in alle Richtungen, wird nie fertig und ist am Ende schwerer zu ändern als neu zu schreiben. Diese erste Einheit des Moduls gehört deshalb dem Entwurf, bevor die erste Programmzeile entsteht.

**Ein Thema wählen.** In Modul 12 haben Sie sich ein Thema überlegt; hier noch einmal die Kriterien, nach denen man es prüft. Das Problem sollte *aus Ihrem Alltag* stammen, damit Sie die Anforderungen aus eigener Erfahrung kennen und beurteilen können, ob das Ergebnis taugt. Es sollte *klein genug* sein, dass eine erste brauchbare Fassung in etwa zwei Tagen steht und der Rest der Zeit auf Tests, Prüfung und Politur fällt; eine zu große Idee scheitert, eine zu kleine lehrt nichts. Es sollte *prüfbar* sein: Sie müssen sagen können, woran man erkennt, dass es funktioniert. Und es sollte mehrere der gelernten Bausteine verbinden, etwa eine Konfigurationsdatei, eine Prüfung des Systems, eine Ausgabe für Menschen und Maschinen und einen sinnvollen Exit-Code. Brauchbare Ideen sind der Wächter für ablaufende Zertifikate, ein Auswertungsskript für Anmeldeversuche in Protokollen, ein Werkzeug, das Konten ohne Anmeldung der letzten 90 Tage auflistet, ein Prüfer für Dateiberechtigungen in einem Verzeichnis, ein Sammler für die Plattenbelegung mehrerer Server oder der Wächter für Datensicherungen, den wir hier als Beispiel ausführen. Ungeeignet sind Ideen, die ein bestehendes Großprojekt nachbauen (ein eigenes Monitoring-System), und Ideen, die ohne Zugriff auf fremde Systeme nicht testbar sind.

**Unser Beispiel: backupwacht.** Das Problem kennt jeder, der Server betreut: Eine Datensicherung läuft nachts, und niemand merkt, wenn sie ausfällt, bis jemand eine Datei braucht. Die Sicherung kann fehlen, veraltet oder auf wenige Bytes geschrumpft sein, weil das Quellverzeichnis leer war. Wir bauen ein Werkzeug, das pro Sicherungsziel prüft, ob im Verzeichnis eine passende Datei liegt, ob sie neu genug ist und ob sie nicht verdächtig klein ist, und das Ergebnis so meldet, dass man es von Hand lesen und von `cron` oder einem Timer auswerten kann.

**Anforderungen formulieren.** Aus einer Idee wird ein Auftrag, indem man ihn aufschreibt. Wir halten fest, was das Programm tun soll, was nicht, und woran man erkennt, dass es fertig ist. Die Abgrenzung (*Nicht-Ziele*) ist genauso wichtig wie die Ziele, weil sie das Wachsen des Projekts begrenzt. Die Kriterien am Ende heißen *Akzeptanzkriterien*: Sie sind so formuliert, dass man sie mit Ja oder Nein beantworten kann, und sie bilden später die Grundlage der Tests. Für `backupwacht` legen wir fest:

- **Ziele:** Das Programm liest eine Konfigurationsdatei mit beliebig vielen Sicherungszielen. Für jedes prüft es, ob im Verzeichnis mindestens eine Datei zum Muster passt, ob die neueste nicht älter ist als erlaubt und ob sie mindestens die Mindestgröße hat. Es gibt eine lesbare Ausgabe und auf Wunsch JSON, und es setzt den Exit-Code nach der schlimmsten Stufe (0 OK, 1 Warnung, 2 kritisch, 3 Konfigurationsfehler).

- **Nicht-Ziele:** Es startet keine Sicherungen, es prüft nicht den Inhalt der Dateien (keine Wiederherstellungstests), es verschickt keine Meldungen selbst (das übernimmt der Aufrufer anhand des Exit-Codes), und es hat keine grafische Oberfläche.

- **Akzeptanzkriterien:** Eine frische, ausreichend große Datei ergibt OK. Eine zu alte oder zu kleine Datei ergibt kritisch. Ein fehlendes Verzeichnis oder keine passende Datei ergibt kritisch. Ein Ziel mit nur einer einzigen Sicherungsdatei ergibt eine Warnung, weil keine Rotation erkennbar ist. Eine fehlerhafte Konfiguration ergibt eine verständliche Meldung und den Exit-Code 3. Alles ist durch Tests abgedeckt, `ruff` und `mypy` sind sauber.

An dieser Stelle verwenden wir ausnahmsweise eine Aufzählung, weil es sich um parallele Kurzangaben handelt. Beachten Sie, dass die Akzeptanzkriterien keine Aussage über die Umsetzung enthalten, sondern nur über das beobachtbare Verhalten. Das ist Absicht: Wie das Programm intern gebaut ist, darf sich ändern (Refactoring in 15.5), ohne dass die Kriterien ungültig werden.

**Die Architektur entwerfen.** Aus 13.3 wissen wir, dass sich Logik von Oberfläche trennen lässt. Wir zerlegen das Programm in Teile, die jeweils *eine* Aufgabe haben und über kleine, klare Schnittstellen sprechen. Die Konfiguration (`config.py`) liest die TOML-Datei, prüft sie und liefert fertige Objekte. Die Prüfung (`pruefung.py`) bekommt ein Sicherungsziel, schaut ins Dateisystem und liefert ein Ergebnis mit Stufe und Meldung; sie weiß nichts von Konfigurationsdateien oder Ausgabeformaten. Der Bericht (`bericht.py`) macht aus einer Liste von Ergebnissen Text oder JSON und bestimmt die Gesamtstufe. Die Kommandozeile (`cli.py`) fügt alles zusammen: Argumente lesen, Konfiguration laden, prüfen, berichten, Exit-Code zurückgeben. Die Abhängigkeiten laufen in eine Richtung, von `cli` zu den anderen, nie zurück. Als Datentypen dienen kleine, unveränderliche Klassen (`dataclass`), damit klar ist, was zwischen den Teilen fließt.

Als Format der Konfiguration wählen wir TOML. Seit Python 3.11 gehört `tomllib` zur Standardbibliothek, sodass keine Zusatzpakete nötig sind, TOML erlaubt Kommentare (was JSON nicht tut) und ist weniger fehleranfällig als YAML, das bei Einrückungen empfindlich ist. Eine Konfiguration sieht so aus: Für jedes Ziel gibt es einen Abschnitt `[[pruefung]]` (die doppelten Klammern bedeuten, dass der Abschnitt mehrfach vorkommen darf und eine Liste ergibt) mit den Schlüsseln `name`, `verzeichnis`, `muster`, `max_alter_stunden` und `min_groesse_byte`.

**Die Arbeit in Schritte zerlegen.** Die wichtigste Regel der Planung lautet: Beginnen Sie mit der *dünnsten durchgehenden Scheibe*, nicht mit dem ersten Baustein. Man nennt das ein *Walking Skeleton* (ein laufendes Gerippe): ein Programm, das schon von Anfang bis Ende läuft, auch wenn es fast nichts kann. Wir beginnen also mit einem Paket, das sich installieren lässt, ein Kommando bereitstellt und einen Test hat, und füllen es dann Stück für Stück. Der Vorteil: Jeder Schritt ist ein lauffähiger Zustand, Git-Commits sind klein und aussagekräftig, und Sie sehen früh, ob die Grundstruktur trägt. Für `backupwacht` ergibt sich diese Reihenfolge: erstens Gerüst und erster Test (heute), zweitens Konfiguration einlesen (15.2), drittens Prüfung eines Ziels (15.2), viertens Bericht (15.3), fünftens Kommandozeile und Gesamtablauf (15.3), sechstens Tests vervollständigen und Review (15.4), siebtens Aufräumen und Dokumentation (15.5). Schätzen Sie für jeden Schritt grob die Dauer und rechnen Sie einen Puffer ein; Schätzungen sind fast immer zu optimistisch.

**Risiken benennen.** Fragen Sie sich vor dem Start, was schiefgehen kann, und planen Sie das Wichtigste zuerst. Bei uns: Wie bestimmt man das Alter einer Datei zuverlässig (Änderungszeit)? Was passiert bei Dateien aus der Zukunft, etwa wenn die Uhr eines Servers falsch ging? Wie testet man zeitabhängigen Code, ohne auf die Uhr zu warten? Letzteres lösen wir, indem wir die aktuelle Zeit als *Parameter* hereinreichen, statt sie im Code abzufragen; eine bewährte Technik, die wir in 15.2 anwenden. Die anderen beiden Fragen halten wir in der README und im Review (15.4) fest.

**Das Gerüst anlegen.** Jetzt setzen wir das Gerüst auf, so wie wir es in Modul 13 gelernt haben: neues Verzeichnis, Git-Repository, Paket im `src`-Layout, `pyproject.toml` mit Einstiegspunkt, `.gitignore`, README. Zuerst Repository und Struktur:

```pycon
~$ mkdir -p ~/backupwacht/src/backupwacht ~/backupwacht/tests ~/backupwacht/docs
~$ cd ~/backupwacht
~/backupwacht$ git init -q -b main
~/backupwacht$ ls
docs
src
tests
```

**Datei** `docs/entwurf.md`**:**

```text
# Entwurf: backupwacht

## Problem
Datensicherungen fallen unbemerkt aus. backupwacht prueft pro Ziel, ob eine
passende, frische und nicht verdaechtig kleine Datei vorhanden ist.

## Konfiguration (TOML)
    [[pruefung]]
    name = "datenbank"
    verzeichnis = "/srv/backup/db"
    muster = "*.sql.gz"
    max_alter_stunden = 26
    min_groesse_byte = 1000

## Module
- config.py    liest und prueft die Konfiguration (Konfiguration, Pruefung)
- pruefung.py  prueft ein Ziel im Dateisystem (Ergebnis)
- bericht.py   formatiert Ergebnisse als Text oder JSON, Gesamtstufe
- cli.py       Argumente, Ablauf, Exit-Code

## Exit-Codes
0 OK, 1 WARNUNG, 2 KRITISCH, 3 Konfigurationsfehler

## Nicht-Ziele
Keine Sicherungen starten, keine Inhaltspruefung, keine eigenen Meldungen.
```

Der Entwurf ist bewusst kurz; er passt auf eine Seite und lebt im Repository, damit er mit dem Code versioniert wird. Ein Entwurf, der in einem Chat verloren geht, hilft in einem halben Jahr niemandem. Als Nächstes die Metadaten des Projekts. Die Datei `pyproject.toml` folgt dem Muster aus 13.4; neu ist, dass wir die Einstellungen für `pytest`, `mypy` und `ruff` von Anfang an mitliefern, damit die Qualitätskette aus Modul 14 vom ersten Commit an gilt.

**Datei** `pyproject.toml`**:**

```text
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "backupwacht"
dynamic = ["version"]
description = "Prueft, ob Datensicherungen vorhanden, frisch und nicht verdaechtig klein sind."
readme = "README.md"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-cov", "ruff", "mypy"]

[project.scripts]
backupwacht = "backupwacht.cli:main"

[tool.setuptools.dynamic]
version = {attr = "backupwacht.__version__"}

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.mypy]
strict = true
files = ["src"]

[tool.ruff]
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "PLW"]
```

**Datei** `.gitignore`**:**

```text
__pycache__/
*.pyc
*.egg-info/
build/
dist/
.venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
.env
*.log
```

Neu in der `.gitignore` ist die Datei `.coverage`, in der `pytest-cov` in 15.4 die Testabdeckung ablegt. Jetzt das Paket mit dem minimalen Einstieg: die Versionsnummer und eine Kommandozeile, die vorerst nur `--version` beherrscht.

**Datei** `src/backupwacht/__init__.py`**:**

```python
"""backupwacht - prueft Datensicherungen."""

__version__ = "0.1.0"
```

**Datei** `src/backupwacht/cli.py`**:**

```python
"""Kommandozeile von backupwacht."""
import argparse

from . import __version__


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="backupwacht", description="Prueft Datensicherungen.")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    parser.parse_args(argv)
    return 0
```

**Datei** `src/backupwacht/__main__.py`**:**

```python
"""Ermoeglicht den Aufruf mit: python -m backupwacht"""
import sys

from .cli import main

sys.exit(main())
```

**Datei** `README.md`**:**

```text
# backupwacht

Prueft, ob Datensicherungen vorhanden, frisch und nicht verdaechtig klein
sind. Das Ergebnis steht im Exit-Code und in einer lesbaren oder JSON-Ausgabe.

Siehe docs/entwurf.md fuer den Entwurf.
```

Das Gerüst ist komplett. Jetzt schreiben wir den ersten Test. Er prüft nur, dass das Kommando existiert und die Version nennt; er ist unser Gerippe, an dem alles Weitere hängt. Wir testen die Funktion `main` direkt mit `capsys`, wie in 14.2.

**Datei** `tests/test_cli.py`**:**

```python
import pytest

from backupwacht import __version__
from backupwacht.cli import main


def test_version_wird_ausgegeben(capsys):
    with pytest.raises(SystemExit) as info:
        main(["--version"])
    assert info.value.code == 0
    assert capsys.readouterr().out.strip() == f"backupwacht {__version__}"
```

Jetzt die virtuelle Umgebung, die Installation und der erste Lauf der gesamten Qualitätskette. Wir installieren mit den Entwicklungswerkzeugen (`".[dev]"`) und führen die vier Prüfungen aus Modul 14 aus.

```pycon
~/backupwacht$ python3 -m venv .venv
~/backupwacht$ . .venv/bin/activate
(.venv) ~/backupwacht$ pip install -q -e ".[dev]"
(.venv) ~/backupwacht$ backupwacht --version
backupwacht 0.1.0
(.venv) ~/backupwacht$ ruff format
2 files reformatted, 4 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 3 source files
(.venv) ~/backupwacht$ pytest -q
.                                                                        [100%]
1 passed in 0.01s
```

Das Gerippe läuft: Das Kommando existiert, die Prüfungen sind sauber, und der erste Test besteht. Der Befehl `ruff format` hat dabei die Dateien einmal in die einheitliche Form gebracht (er meldet, wie viele Dateien er geändert hat); ab jetzt genügt `ruff format --check` als Prüfung. Wir committen jetzt den Anfang in mehreren kleinen Schritten, wie es in 13.5 empfohlen wurde: erst Entwurf und Dokumentation, dann das Gerüst mit Tests.

```pycon
(.venv) ~/backupwacht$ git add docs README.md .gitignore
(.venv) ~/backupwacht$ git commit -q -m "Entwurf und README anlegen"
(.venv) ~/backupwacht$ git add pyproject.toml src tests
(.venv) ~/backupwacht$ git commit -q -m "Gerüst: Paket, Kommandozeile mit --version, erster Test"
(.venv) ~/backupwacht$ git log --oneline
1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
95754fb Entwurf und README anlegen
(.venv) ~/backupwacht$ git status --short
```

**Typische Fehler und Stolperfallen.** Der klassische Planungsfehler ist, das Projekt zu groß zu denken: Aus dem Wächter für Sicherungen wird in Gedanken ein Monitoring-System mit Weboberfläche. Schützen Sie sich mit den Nicht-Zielen und mit der Frage bei jeder neuen Idee, ob sie zu einem Akzeptanzkriterium gehört. Ideen, die es nicht tun, kommen auf eine Liste für später. Ein zweiter Fehler ist, mit dem *interessantesten* Teil zu beginnen (einer cleveren Auswertung), statt mit dem durchgehenden Gerippe; dann hängt der Rest in der Luft, und beim Zusammenbau zeigt sich, dass die Teile nicht zueinander passen. Drittens: Wer keine Akzeptanzkriterien aufschreibt, weiß am Ende nicht, ob das Projekt fertig ist, und feilt endlos. Viertens: Entscheidungen, die man nicht aufschreibt, vergisst man; deshalb gehört der Entwurf in das Repository. Fünftens: Ein Programm, das die Uhr, das Netz oder das Dateisystem direkt anfasst, ist schwer zu testen; planen Sie von Anfang an die Stellen ein, an denen Sie solche Abhängigkeiten hereinreichen können (Parameter, wie bei `jetzt` und `argv`). Sechstens schließlich: Das Gerüst zuerst zu committen, bevor Code existiert, wirkt formell, spart aber später die Frage, warum das Projekt mitten in der Arbeit nicht mehr installierbar ist.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Aussagen zu TOML und `tomllib` (Teil der Standardbibliothek seit Python 3.11, `[[...]]` für Listen von Tabellen) wurden an der installierten Python-Version 3.13.18 ausprobiert (siehe 15.2); die Dokumentationsseite des Moduls `tomllib` wurde nicht abgerufen. Die Begriffe Akzeptanzkriterien, Nicht-Ziele und Walking Skeleton stammen aus der allgemeinen Praxis der Softwareentwicklung und sind Fachwissen; sie wurden nicht aus einer abgerufenen Quelle belegt. Die Struktur der `pyproject.toml` folgt den in Modul 13 abgerufenen Seiten des Python Packaging User Guide.

> **Merke**
>
> Ein Abschlussprojekt wird geplant, bevor es programmiert wird: Thema aus dem eigenen Alltag, klein genug, prüfbar. Halten Sie Ziele, Nicht-Ziele und Akzeptanzkriterien schriftlich im Repository fest. Zerlegen Sie das Programm in Teile mit je einer Aufgabe und einer Abhängigkeitsrichtung. Beginnen Sie mit einem laufenden Gerippe (Walking Skeleton) und füllen Sie es in kleinen, getesteten Schritten. Reichen Sie Abhängigkeiten wie die Uhrzeit als Parameter herein, damit der Code testbar bleibt.

> **Praxisbeispiel**
>
> Im Betrieb beginnt jedes brauchbare Werkzeug mit einem Ärgernis, das jemand zum dritten Mal von Hand erledigt. Ein kurzer Entwurf von einer Seite, den Kolleginnen und Kollegen lesen können, spart später lange Diskussionen: Er zeigt, was das Werkzeug tun soll und was ausdrücklich nicht, und er ist der erste Ort, an dem jemand nachsieht, wenn nachts etwas ausfällt. Viele Teams verlangen für neue Werkzeuge genau so ein Dokument, bevor jemand einen Tag Arbeit hineinsteckt.

## 15.2 Abschlussprojekt: Umsetzung 1

Das Gerippe steht, jetzt wird es mit Leben gefüllt. In dieser Einheit bauen wir die beiden Teile, von denen alles Weitere abhängt: das Einlesen der Konfiguration und die Prüfung eines einzelnen Sicherungsziels. Wir arbeiten dabei nach einem Muster, das Sie schon aus 14.1 kennen und das in der Praxis *testgetriebene Entwicklung* heißt (englisch *test-driven development*, TDD): Zuerst schreibt man einen Test, der das gewünschte Verhalten beschreibt und natürlich fehlschlägt, weil es den Code noch nicht gibt. Dann schreibt man gerade so viel Code, dass der Test besteht, und räumt danach auf. Das klingt umständlich, hat aber zwei handfeste Vorteile: Man überlegt sich vor dem Programmieren genau, was man eigentlich will (der Test ist die genaueste Form der Anforderung), und man hat am Ende zu jedem Verhalten einen Test, weil keiner „später“ geschrieben werden muss.

**Die Konfiguration: Datentypen.** Wir beginnen mit den Datentypen, denn sie legen fest, was zwischen den Teilen fließt. Eine `dataclass` ist eine Klasse, die vor allem Daten hält; Python erzeugt für sie automatisch die Methode, die Felder beim Erzeugen zu setzen, einen Vergleich und eine lesbare Ausgabe. Mit `frozen=True` macht man die Objekte *unveränderlich*: Nach dem Erzeugen kann niemand ein Feld überschreiben, was eine ganze Klasse von Fehlern ausschließt. Ein Sicherungsziel (`Pruefung`) hat einen Namen, ein Verzeichnis, ein Dateimuster, ein höchstes Alter in Stunden und eine Mindestgröße in Byte; die Konfiguration (`Konfiguration`) ist eine Liste davon. Dazu kommt eine eigene Ausnahme, `KonfigurationsFehler`, die von `ValueError` abgeleitet ist: Wer die Konfiguration lädt, kann genau diesen Fehler abfangen und dem Benutzer eine verständliche Meldung zeigen, statt einen Python-Traceback.

Zuerst der Test. Er schreibt eine Konfigurationsdatei in ein temporäres Verzeichnis (`tmp_path`, 14.2), lädt sie und prüft das Ergebnis. Es folgen die Fehlerfälle, die wir aus den Akzeptanzkriterien kennen: fehlende Datei, kaputtes TOML, keine Abschnitte, ein Schreibfehler im Schlüssel (der still ignoriert würde, wenn wir nicht dagegen vorgehen), fehlende Pflichtangaben und unsinnige Werte. Die Hilfsfunktion `schreibe` legt eine Datei an und gibt ihren Pfad zurück.

**Datei** `tests/test_config.py`**:**

```python
from pathlib import Path

import pytest

from backupwacht.config import Konfiguration, KonfigurationsFehler, Pruefung, lade_konfiguration

GUELTIG = """
[[pruefung]]
name = "datenbank"
verzeichnis = "/srv/backup/db"
muster = "*.sql.gz"
max_alter_stunden = 26
min_groesse_byte = 1000

[[pruefung]]
name = "dateien"
verzeichnis = "/srv/backup/files"
"""


def schreibe(tmp_path: Path, inhalt: str) -> Path:
    pfad = tmp_path / "backupwacht.toml"
    pfad.write_text(inhalt, encoding="utf-8")
    return pfad


def test_gueltige_konfiguration_wird_gelesen(tmp_path):
    konfig = lade_konfiguration(schreibe(tmp_path, GUELTIG))
    assert isinstance(konfig, Konfiguration)
    assert konfig.pruefungen[0] == Pruefung(
        name="datenbank",
        verzeichnis=Path("/srv/backup/db"),
        muster="*.sql.gz",
        max_alter_stunden=26.0,
        min_groesse_byte=1000,
    )


def test_fehlende_angaben_bekommen_standardwerte(tmp_path):
    zweite = lade_konfiguration(schreibe(tmp_path, GUELTIG)).pruefungen[1]
    assert zweite.muster == "*"
    assert zweite.max_alter_stunden == 26.0
    assert zweite.min_groesse_byte == 1


def test_fehlende_datei(tmp_path):
    with pytest.raises(KonfigurationsFehler, match="nicht gefunden"):
        lade_konfiguration(tmp_path / "gibt_es_nicht.toml")


def test_kaputtes_toml(tmp_path):
    with pytest.raises(KonfigurationsFehler, match="Ungueltiges TOML"):
        lade_konfiguration(schreibe(tmp_path, "name = = 3"))


def test_ohne_abschnitt(tmp_path):
    with pytest.raises(KonfigurationsFehler, match=r"\[\[pruefung\]\]"):
        lade_konfiguration(schreibe(tmp_path, "# leer\n"))


@pytest.mark.parametrize(
    "inhalt, erwartet",
    [
        ('[[pruefung]]\nname = "a"\n', "'verzeichnis' fehlt"),
        ('[[pruefung]]\nverzeichnis = "/x"\n', "'name' fehlt"),
        ('[[pruefung]]\nname = "a"\nverzeichnis = "/x"\nmax_alter = 5\n', "unbekannte Schluessel"),
        ('[[pruefung]]\nname = "a"\nverzeichnis = "/x"\nmax_alter_stunden = -1\n', "max_alter_stunden"),
        ('[[pruefung]]\nname = "a"\nverzeichnis = "/x"\nmax_alter_stunden = true\n', "max_alter_stunden"),
        ('[[pruefung]]\nname = "a"\nverzeichnis = "/x"\nmin_groesse_byte = 1.5\n', "min_groesse_byte"),
    ],
)
def test_fehlerhafte_eintraege(tmp_path, inhalt, erwartet):
    with pytest.raises(KonfigurationsFehler, match=erwartet):
        lade_konfiguration(schreibe(tmp_path, inhalt))
```

Beachten Sie die Tests der fehlerhaften Einträge: Sie sind *parametrisiert*, damit ein Fall pro Zeile steht. Der Test mit dem Schlüssel `max_alter` (statt `max_alter_stunden`) deckt den klassischen Fehler ab, dass sich jemand verschreibt und das Programm ohne Meldung mit dem Standardwert weiterläuft. Wir führen die Tests aus und sehen, dass sie scheitern, weil das Modul noch nicht existiert.

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -6
    from backupwacht.config import Konfiguration, KonfigurationsFehler, Pruefung, lade_konfiguration
E   ModuleNotFoundError: No module named 'backupwacht.config'
=========================== short test summary info ============================
ERROR tests/test_config.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.01s
```

Der Fehler lautet `ModuleNotFoundError`: Das ist der erwartete rote Zustand, von dem aus wir beginnen. Jetzt schreiben wir das Modul. Die Funktion `lade_konfiguration` öffnet die Datei im Binärmodus, wie es `tomllib` verlangt, übersetzt die beiden möglichen Systemfehler in unseren eigenen Fehler und prüft dann Eintrag für Eintrag. Die Prüfung übernimmt eine Hilfsfunktion, die jeden Fehler mit der Nummer des Eintrags versieht, damit man in einer langen Datei die Stelle findet.

**Datei** `src/backupwacht/config.py`**:**

```python
"""Einlesen und Pruefen der Konfiguration."""
import tomllib
from dataclasses import dataclass
from pathlib import Path
from typing import Any

ERLAUBTE_SCHLUESSEL = {"name", "verzeichnis", "muster", "max_alter_stunden", "min_groesse_byte"}


class KonfigurationsFehler(ValueError):
    """Die Konfiguration fehlt, ist kein gueltiges TOML oder enthaelt falsche Werte."""


@dataclass(frozen=True)
class Pruefung:
    """Ein Sicherungsziel: wo gesucht wird und was als gesund gilt."""

    name: str
    verzeichnis: Path
    muster: str = "*"
    max_alter_stunden: float = 26.0
    min_groesse_byte: int = 1


@dataclass(frozen=True)
class Konfiguration:
    pruefungen: list[Pruefung]


def lade_konfiguration(pfad: str | Path) -> Konfiguration:
    """Liest die TOML-Datei und gibt die geprueften Sicherungsziele zurueck."""
    try:
        with open(pfad, "rb") as datei:
            daten = tomllib.load(datei)
    except FileNotFoundError as fehler:
        raise KonfigurationsFehler(f"Konfigurationsdatei nicht gefunden: {pfad}") from fehler
    except tomllib.TOMLDecodeError as fehler:
        raise KonfigurationsFehler(f"Ungueltiges TOML in {pfad}: {fehler}") from fehler
    eintraege = daten.get("pruefung")
    if not isinstance(eintraege, list) or not eintraege:
        raise KonfigurationsFehler("Mindestens ein Abschnitt [[pruefung]] ist noetig")
    return Konfiguration([_baue_pruefung(nr, e) for nr, e in enumerate(eintraege, start=1)])


def _baue_pruefung(nummer: int, eintrag: dict[str, Any]) -> Pruefung:
    ort = f"[[pruefung]] Nr. {nummer}"
    unbekannt = set(eintrag) - ERLAUBTE_SCHLUESSEL
    if unbekannt:
        raise KonfigurationsFehler(f"{ort}: unbekannte Schluessel {sorted(unbekannt)}")
    for pflicht in ("name", "verzeichnis"):
        if pflicht not in eintrag:
            raise KonfigurationsFehler(f"{ort}: Schluessel '{pflicht}' fehlt")
    max_alter = eintrag.get("max_alter_stunden", 26)
    if isinstance(max_alter, bool) or not isinstance(max_alter, int | float) or max_alter <= 0:
        raise KonfigurationsFehler(f"{ort}: max_alter_stunden muss eine Zahl groesser 0 sein")
    min_groesse = eintrag.get("min_groesse_byte", 1)
    if isinstance(min_groesse, bool) or not isinstance(min_groesse, int) or min_groesse < 0:
        raise KonfigurationsFehler(f"{ort}: min_groesse_byte muss eine ganze Zahl ab 0 sein")
    return Pruefung(
        name=str(eintrag["name"]),
        verzeichnis=Path(str(eintrag["verzeichnis"])),
        muster=str(eintrag.get("muster", "*")),
        max_alter_stunden=float(max_alter),
        min_groesse_byte=min_groesse,
    )
```

Einige Einzelheiten: Das Wörterbuch `ERLAUBTE_SCHLUESSEL` ist eine Menge; die Differenz `set(eintrag) - ERLAUBTE_SCHLUESSEL` liefert alle Schlüssel, die wir nicht kennen. In Python ist `True` eine ganze Zahl (`isinstance(True, int)` ist wahr), deshalb schließen wir Wahrheitswerte ausdrücklich aus, sonst ginge `max_alter_stunden = true` als „1“ durch. Die Schreibweise `int | float` in `isinstance` ist seit Python 3.10 erlaubt. Die führende Unterstreichung bei `_baue_pruefung` bedeutet nach Konvention „nur für den internen Gebrauch dieses Moduls“. Wir führen die Qualitätskette aus.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
2 files reformatted, 4 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 4 source files
(.venv) ~/backupwacht$ pytest -q
............                                                             [100%]
12 passed in 0.01s
```

Alle Tests bestehen, und `mypy` und `ruff` sind zufrieden. Wir sichern den Fortschritt. Zwei Commits wären sauberer (Test und Code getrennt), aber die Regel „jeder Commit ist ein lauffähiger Zustand“ ist wichtiger als die Trennung, und ein roter Test im Repository stört das Zusammenarbeiten.

```pycon
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "Konfiguration aus TOML einlesen und pruefen"
(.venv) ~/backupwacht$ git log --oneline -3
5378f42 Konfiguration aus TOML einlesen und pruefen
1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
95754fb Entwurf und README anlegen
```

**Die Prüfung: erst der Test, dann der Code.** Jetzt der Kern. Wie testet man etwas, das vom Alter von Dateien abhängt? Zwei Dinge sind nötig. Erstens erzeugen wir im Test Dateien mit *vorgegebenem Alter*, indem wir die Änderungszeit mit `os.utime` setzen (Modul 4). Zweitens reicht die Funktion die aktuelle Zeit als Parameter `jetzt` herein, statt `time.time()` selbst zu fragen. Ein Test legt `jetzt` auf einen festen Wert, und das Alter jeder Datei ist dann exakt berechenbar, ohne dass man auf die Uhr warten muss. Das ist die Technik aus der Planung: Abhängigkeiten als Parameter hereinreichen. Der Test beschreibt jede Zeile der Akzeptanzkriterien.

**Datei** `tests/test_pruefung.py`**:**

```python
import os
from pathlib import Path

from backupwacht.config import Pruefung
from backupwacht.pruefung import pruefe_ziel

JETZT = 1_800_000_000.0


def lege_datei(ordner: Path, name: str, groesse: int, alter_stunden: float) -> Path:
    """Legt eine Datei mit vorgegebener Groesse und vorgegebenem Alter an."""
    pfad = ordner / name
    pfad.write_bytes(b"x" * groesse)
    zeit = JETZT - alter_stunden * 3600
    os.utime(pfad, (zeit, zeit))
    return pfad


def ziel(ordner: Path, **angaben) -> Pruefung:
    return Pruefung(name="test", verzeichnis=ordner, **angaben)


def test_frische_grosse_datei_und_rotation_ist_ok(tmp_path):
    lege_datei(tmp_path, "a.tar.gz", 5000, alter_stunden=2)
    lege_datei(tmp_path, "b.tar.gz", 5000, alter_stunden=26)
    ergebnis = pruefe_ziel(ziel(tmp_path, muster="*.tar.gz", min_groesse_byte=1000), JETZT)
    assert ergebnis.stufe == "OK"
    assert ergebnis.alter_stunden == 2.0
    assert "a.tar.gz" in ergebnis.meldung


def test_zu_alte_datei_ist_kritisch(tmp_path):
    lege_datei(tmp_path, "a.tar.gz", 5000, alter_stunden=30)
    lege_datei(tmp_path, "b.tar.gz", 5000, alter_stunden=50)
    ergebnis = pruefe_ziel(ziel(tmp_path, max_alter_stunden=26), JETZT)
    assert ergebnis.stufe == "KRITISCH"
    assert "30.0 Stunden alt" in ergebnis.meldung


def test_genau_am_grenzalter_ist_noch_ok(tmp_path):
    lege_datei(tmp_path, "a", 10, alter_stunden=26)
    lege_datei(tmp_path, "b", 10, alter_stunden=50)
    assert pruefe_ziel(ziel(tmp_path, max_alter_stunden=26), JETZT).stufe == "OK"


def test_zu_kleine_datei_ist_kritisch(tmp_path):
    lege_datei(tmp_path, "a", 10, alter_stunden=1)
    lege_datei(tmp_path, "b", 10, alter_stunden=25)
    ergebnis = pruefe_ziel(ziel(tmp_path, min_groesse_byte=1000), JETZT)
    assert ergebnis.stufe == "KRITISCH"
    assert "10 Byte" in ergebnis.meldung


def test_fehlendes_verzeichnis_ist_kritisch(tmp_path):
    ergebnis = pruefe_ziel(ziel(tmp_path / "gibt_es_nicht"), JETZT)
    assert ergebnis.stufe == "KRITISCH"
    assert "Verzeichnis fehlt" in ergebnis.meldung


def test_keine_passende_datei_ist_kritisch(tmp_path):
    lege_datei(tmp_path, "notizen.txt", 100, alter_stunden=1)
    ergebnis = pruefe_ziel(ziel(tmp_path, muster="*.sql.gz"), JETZT)
    assert ergebnis.stufe == "KRITISCH"
    assert "*.sql.gz" in ergebnis.meldung


def test_nur_eine_sicherung_ist_eine_warnung(tmp_path):
    lege_datei(tmp_path, "einzige.tar", 100, alter_stunden=1)
    assert pruefe_ziel(ziel(tmp_path), JETZT).stufe == "WARNUNG"


def test_die_neueste_datei_zaehlt(tmp_path):
    lege_datei(tmp_path, "alt", 100, alter_stunden=100)
    lege_datei(tmp_path, "neu", 100, alter_stunden=3)
    ergebnis = pruefe_ziel(ziel(tmp_path), JETZT)
    assert ergebnis.stufe == "OK"
    assert "neu" in ergebnis.meldung


def test_unterverzeichnisse_werden_nicht_als_sicherung_gezaehlt(tmp_path):
    (tmp_path / "ordner.tar").mkdir()
    ergebnis = pruefe_ziel(ziel(tmp_path, muster="*.tar"), JETZT)
    assert ergebnis.stufe == "KRITISCH"
```

Der Aufbau der Tests ist bewusst gleichförmig: Vorbereiten mit `lege_datei`, Aufrufen mit `pruefe_ziel`, Prüfen. Die Konstante `JETZT` ist ein beliebiger fester Zeitpunkt (in Sekunden seit dem 1. Januar 1970, wie `time.time()` ihn liefert); entscheidend ist nur, dass er in allen Tests derselbe ist. Der Test `test_genau_am_grenzalter_ist_noch_ok` legt fest, dass die Grenze *eingeschlossen* ist: Eine Sicherung von genau 26 Stunden bei erlaubten 26 ist in Ordnung. Solche Entscheidungen müssen irgendwo festgeschrieben werden, und ein Test ist der beste Platz. Jetzt die Umsetzung. Das Ergebnis ist wieder eine unveränderliche `dataclass`; die Stufe ist ein `Literal` wie in 14.4.

**Datei** `src/backupwacht/pruefung.py`**:**

```python
"""Pruefung eines Sicherungsziels im Dateisystem."""
import time
from dataclasses import dataclass
from typing import Literal

from .config import Pruefung

Stufe = Literal["OK", "WARNUNG", "KRITISCH"]


@dataclass(frozen=True)
class Ergebnis:
    """Ergebnis der Pruefung eines Ziels."""

    name: str
    stufe: Stufe
    meldung: str
    alter_stunden: float | None = None


def pruefe_ziel(ziel: Pruefung, jetzt: float | None = None) -> Ergebnis:
    """Prueft ein Sicherungsziel; jetzt (Sekunden seit 1970) dient dem Testen."""
    jetzt = time.time() if jetzt is None else jetzt
    if not ziel.verzeichnis.is_dir():
        return Ergebnis(ziel.name, "KRITISCH", f"Verzeichnis fehlt: {ziel.verzeichnis}")
    dateien = [d for d in ziel.verzeichnis.glob(ziel.muster) if d.is_file()]
    if not dateien:
        return Ergebnis(ziel.name, "KRITISCH", f"keine Datei passt zu '{ziel.muster}'")
    neueste = max(dateien, key=lambda d: d.stat().st_mtime)
    status = neueste.stat()
    alter = (jetzt - status.st_mtime) / 3600
    if status.st_size < ziel.min_groesse_byte:
        text = f"{neueste.name} ist nur {status.st_size} Byte gross"
        return Ergebnis(ziel.name, "KRITISCH", text, alter)
    if alter > ziel.max_alter_stunden:
        text = f"{neueste.name} ist {alter:.1f} Stunden alt (erlaubt: {ziel.max_alter_stunden:g})"
        return Ergebnis(ziel.name, "KRITISCH", text, alter)
    if len(dateien) < 2:
        return Ergebnis(ziel.name, "WARNUNG", f"nur eine Sicherung vorhanden: {neueste.name}", alter)
    return Ergebnis(ziel.name, "OK", f"{neueste.name}, {alter:.1f} Stunden alt", alter)
```

Die Funktion arbeitet von den schwersten zu den leichtesten Fällen: Erst werden die Ursachen geprüft, die eine Auswertung unmöglich machen (Verzeichnis oder Datei fehlt), dann die kritischen Fälle (zu klein, zu alt), dann die Warnung, zuletzt der Normalfall. Man nennt dieses Muster *frühe Rückkehr* (englisch *early return*): Jeder Fehlerfall wird sofort behandelt, und der Rest der Funktion kann davon ausgehen, dass alles in Ordnung ist. Die Auswahl der neuesten Datei erledigt `max` mit einer Schlüsselfunktion (Modul 2 und 3); `d.is_file()` schließt Verzeichnisse aus, die zufällig zum Muster passen. Die Formatangabe `{...:g}` gibt die Zahl ohne überflüssige Nachkommastellen aus (26.0 wird zu 26). Wir lassen die Kette laufen.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 7 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 5 source files
(.venv) ~/backupwacht$ pytest -q
.....................                                                    [100%]
21 passed in 0.01s
```

Alle Tests bestehen. Damit ist der Kern des Programms fertig und durch Tests abgesichert, bevor es eine einzige Zeile Kommandozeile gibt. Das ist der eigentliche Gewinn der Trennung aus der Planung: Die Prüfung lässt sich unabhängig von Argumenten, Ausgabe und Konfiguration entwickeln und testen. Wir sichern den Stand.

```pycon
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "Sicherungsziel pruefen: Alter, Groesse, Rotation"
(.venv) ~/backupwacht$ git log --oneline -4
3c795ff Sicherungsziel pruefen: Alter, Groesse, Rotation
5378f42 Konfiguration aus TOML einlesen und pruefen
1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
95754fb Entwurf und README anlegen
```

**Ein Blick auf echte Dateien.** Tests mit künstlichem Alter sind genau, aber wir wollen auch sehen, wie sich die Funktion an echten Dateien verhält. Mit dem Befehl `touch -d` setzen wir Änderungszeiten von Hand. Wir bauen ein Verzeichnis mit zwei Sicherungen und rufen die Funktion aus der Python-Shell auf. Beachten Sie, dass die Angabe des Alters hier wirklich von der Uhr Ihres Rechners abhängt, weshalb wir nur ganze Stunden ausgeben.

```pycon
(.venv) ~/backupwacht$ mkdir -p ~/demo/db
(.venv) ~/backupwacht$ cd ~/demo/db
(.venv) ~/demo/db$ head -c 5000 /dev/zero > nacht_1.sql.gz
(.venv) ~/demo/db$ head -c 5000 /dev/zero > nacht_2.sql.gz
(.venv) ~/demo/db$ touch -d "3 hours ago" nacht_2.sql.gz
(.venv) ~/demo/db$ touch -d "27 hours ago" nacht_1.sql.gz
(.venv) ~/demo/db$ ls -l --time-style=+%H:%M | awk '{print $5, $7}'
 
5000 nacht_1.sql.gz
5000 nacht_2.sql.gz
(.venv) ~/demo/db$ cd ~/backupwacht
(.venv) ~/backupwacht$ python3 -c "from pathlib import Path; from backupwacht.config import Pruefung; from backupwacht.pruefung import pruefe_ziel; e = pruefe_ziel(Pruefung('db', Path.home() / 'demo' / 'db', '*.sql.gz', 26, 1000)); print(e.stufe, round(e.alter_stunden))"
OK 3
```

Die Funktion findet die neuere der beiden Dateien (drei Stunden alt), bewertet die Sicherung als in Ordnung und liefert das Alter. Das gerundete Alter beträgt, wie erwartet, drei Stunden.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Alter von Dateien ist die Verwechslung der Zeiten: Das Dateisystem kennt neben der Änderungszeit (`st_mtime`) auch die Zeit der letzten Statusänderung (`st_ctime`, die sich bei Umbenennen oder Rechteänderung ändert, unter Unix *nicht* die Erstellungszeit) und die Zugriffszeit (`st_atime`, die viele Systeme gar nicht mehr genau führen). Für Sicherungen ist die Änderungszeit richtig, weil sie anzeigt, wann der Inhalt zuletzt geschrieben wurde. Ein zweiter Fehler ist, die Uhrzeit im Code zu fragen und dann nicht testen zu können; wir haben das mit dem Parameter `jetzt` gelöst. Drittens: `glob` ohne Prüfung auf Dateien liefert auch Verzeichnisse, die zum Muster passen, und `max` auf einer leeren Liste löst einen `ValueError` aus; beides ist hier abgedeckt (`is_file()` und der Fall „keine Datei“). Viertens sind Vergleiche mit der Grenze eine Quelle von Fehlern um genau eins; wir haben ihr Verhalten durch einen Test festgelegt. Fünftens: Eine Konfiguration, die Tippfehler stillschweigend ignoriert, ist tückisch. Unser Test mit `max_alter` stellt sicher, dass unbekannte Schlüssel einen Fehler auslösen. Sechstens reichen die Tests nicht aus, solange sie nur Dateien in einem temporären Verzeichnis kennen; was bei Berechtigungsfehlern oder bei einem Verzeichnis passiert, das kein Lesen erlaubt, bleibt offen, und wir halten es für die Durchsicht in 15.4 fest.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Beispiele dieses Abschnitts wurden mit Python 3.13.18 und pytest 9.2.0 ausgeführt; die Tests und die Ausgaben stammen aus echten Läufen. Das Verhalten von `tomllib` (Binärmodus, `TOMLDecodeError`, Listen von Tabellen) und die Zeitstempel von Dateien (`st_mtime`, `st_ctime`, `st_atime`) sind Fachwissen und wurden hier nicht aus der Dokumentation belegt, aber durch die Läufe bestätigt. Die testgetriebene Entwicklung ist eine verbreitete Arbeitsweise; die Darstellung folgt der allgemeinen Praxis.

> **Merke**
>
> Beginnen Sie mit den Datentypen (`dataclass(frozen=True)`), dann mit einem Test, dann mit dem Code. Prüfen Sie Konfigurationen streng: unbekannte Schlüssel, fehlende Pflichtangaben und unsinnige Werte sind Fehler, keine Standardwerte. Reichen Sie die Uhrzeit als Parameter herein, damit zeitabhängiger Code testbar bleibt, und setzen Sie Dateizeiten im Test mit `os.utime`. Frühe Rückkehr (*early return*) hält Funktionen flach und lesbar. Jeder Commit sollte ein lauffähiger Zustand sein.

> **Praxisbeispiel**
>
> Das Muster „Konfiguration lesen, prüfen, Objekte bauen, Ergebnisse berichten“ steckt in fast jedem Betriebswerkzeug, von Monitoring-Agenten bis zu Backup-Skripten. Wer es einmal sauber getrennt hat, kann jeden Teil einzeln testen und austauschen: Heute liest das Programm TOML, morgen vielleicht eine Datenbank, ohne dass die Prüfung davon etwas merkt. Streng geprüfte Konfigurationen verhindern eine der häufigsten Betriebsstörungen, den stillen Tippfehler, der erst im Ernstfall auffällt.

## 15.3 Abschlussprojekt: Umsetzung 2

Konfiguration und Prüfung funktionieren, doch das Programm kann noch nichts anzeigen und noch nichts melden. In dieser Einheit fügen wir die letzten beiden Teile hinzu: den Bericht, der Ergebnisse als lesbaren Text oder als JSON aufbereitet, und die Kommandozeile, die alles zusammensteckt und einen sinnvollen Exit-Code liefert. Danach läuft das Programm zum ersten Mal von Anfang bis Ende, und wir betrachten, wie man es im Betrieb zeitgesteuert startet.

**Der Bericht.** Der Bericht hat drei Aufgaben. Er bestimmt aus mehreren Ergebnissen die *Gesamtstufe*: Sobald ein Ziel kritisch ist, ist das Gesamtergebnis kritisch; gibt es keinen kritischen, aber einen Warnfall, ist es eine Warnung; sonst ist alles in Ordnung. Er übersetzt die Gesamtstufe in einen Exit-Code, den Aufrufer wie `cron`, `systemd` oder ein Überwachungssystem auswerten (0, 1 und 2 entsprechen der Konvention vieler Überwachungswerkzeuge; der Wert 3 ist bei uns dem Konfigurationsfehler vorbehalten). Und er erzeugt die Ausgabe: für Menschen eine Zeile pro Ziel, für Maschinen JSON. Die Trennung vom Rest ist gewollt: Ob das Ergebnis auf dem Bildschirm, in einer Datei oder in einer Mail landet, entscheidet später die Kommandozeile, nicht die Logik.

Wie üblich beginnen wir mit dem Test. Die Ergebnisse legen wir von Hand an, ganz ohne Dateisystem, weil der Bericht nur mit fertigen `Ergebnis`-Objekten arbeitet. Das macht die Tests schnell und einfach. Für die Gesamtstufe und den Exit-Code verwenden wir `parametrize` (14.1), damit jede Kombination eine eigene Zeile hat.

**Datei** `tests/test_bericht.py`**:**

```python
import json

import pytest

from backupwacht.bericht import als_json, als_text, exit_code, gesamtstufe
from backupwacht.pruefung import Ergebnis

OK = Ergebnis("a", "OK", "alles gut", 2.0)
WARN = Ergebnis("b", "WARNUNG", "nur eine Sicherung", 5.26)
KRIT = Ergebnis("c", "KRITISCH", "Verzeichnis fehlt")


@pytest.mark.parametrize(
    "ergebnisse, erwartet",
    [
        ([OK], "OK"),
        ([OK, WARN], "WARNUNG"),
        ([WARN, KRIT, OK], "KRITISCH"),
        ([], "OK"),
    ],
)
def test_gesamtstufe_ist_die_schlechteste(ergebnisse, erwartet):
    assert gesamtstufe(ergebnisse) == erwartet


@pytest.mark.parametrize("stufe, code", [("OK", 0), ("WARNUNG", 1), ("KRITISCH", 2)])
def test_exit_code(stufe, code):
    assert exit_code(stufe) == code


def test_text_hat_eine_zeile_pro_ziel_und_eine_summe():
    zeilen = als_text([OK, KRIT]).splitlines()
    assert zeilen[0].startswith("OK ")
    assert "a: alles gut" in zeilen[0]
    assert zeilen[1].startswith("KRITISCH ")
    assert "c: Verzeichnis fehlt" in zeilen[1]
    assert zeilen[-1] == "Gesamt: KRITISCH"


def test_json_ist_gueltig_und_gerundet():
    daten = json.loads(als_json([WARN, KRIT]))
    assert daten["gesamt"] == "KRITISCH"
    assert daten["ergebnisse"][0]["alter_stunden"] == 5.3
    assert daten["ergebnisse"][1]["alter_stunden"] is None
    assert daten["ergebnisse"][1]["name"] == "c"
```

Beachten Sie, dass auch der Sonderfall der leeren Liste festgelegt wird: Ohne Ergebnisse gibt es nichts zu beanstanden, die Gesamtstufe ist `"OK"`. Im Programm kann das nicht vorkommen, weil die Konfiguration mindestens ein Ziel verlangt, aber eine Funktion sollte für jede erlaubte Eingabe etwas Sinnvolles tun. Die Umsetzung ist kurz. Die Reihenfolge der Stufen steht in einem Wörterbuch; das hat den Vorteil, dass sie an genau einer Stelle festgelegt ist und sowohl für den Vergleich als auch für den Exit-Code dient.

**Datei** `src/backupwacht/bericht.py`**:**

```python
"""Aufbereitung der Ergebnisse als Text oder JSON."""
import json
from collections.abc import Sequence
from dataclasses import asdict

from .pruefung import Ergebnis, Stufe

REIHENFOLGE: dict[Stufe, int] = {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}


def gesamtstufe(ergebnisse: Sequence[Ergebnis]) -> Stufe:
    """Die schlechteste Stufe aller Ergebnisse; ohne Ergebnisse OK."""
    stufe: Stufe = "OK"
    for ergebnis in ergebnisse:
        if REIHENFOLGE[ergebnis.stufe] > REIHENFOLGE[stufe]:
            stufe = ergebnis.stufe
    return stufe


def exit_code(stufe: Stufe) -> int:
    """0 = OK, 1 = Warnung, 2 = kritisch (wie bei vielen Ueberwachungswerkzeugen)."""
    return REIHENFOLGE[stufe]


def als_text(ergebnisse: Sequence[Ergebnis]) -> str:
    zeilen = [f"{e.stufe:<8} {e.name}: {e.meldung}" for e in ergebnisse]
    zeilen.append(f"Gesamt: {gesamtstufe(ergebnisse)}")
    return "\n".join(zeilen)


def als_json(ergebnisse: Sequence[Ergebnis]) -> str:
    eintraege = []
    for ergebnis in ergebnisse:
        daten = asdict(ergebnis)
        if ergebnis.alter_stunden is not None:
            daten["alter_stunden"] = round(ergebnis.alter_stunden, 1)
        eintraege.append(daten)
    ausgabe = {"gesamt": gesamtstufe(ergebnisse), "ergebnisse": eintraege}
    return json.dumps(ausgabe, indent=2, ensure_ascii=False)
```

Der Typ `Sequence` aus `collections.abc` bedeutet „irgendeine Folge, die man durchlaufen kann“; die Funktionen funktionieren damit für Listen genauso wie für Tupel, und der Aufrufer ist nicht auf eine Liste festgelegt. Das Format `{e.stufe:<8}` füllt die Stufe linksbündig auf acht Zeichen auf, damit die Spalten untereinander stehen. Die Funktion `asdict` wandelt eine `dataclass` in ein Wörterbuch um, und `ensure_ascii=False` sorgt dafür, dass Umlaute im JSON nicht als `ä` erscheinen. Wir lassen die Kette laufen.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 9 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/backupwacht$ pytest -q
..............................                                           [100%]
30 passed in 0.01s
```

**Die Kommandozeile.** Jetzt kommt der letzte Baustein. Die Kommandozeile hat vier Aufgaben: Sie liest die Argumente (Konfigurationsdatei und Ausgabeformat), lädt die Konfiguration, ruft für jedes Ziel die Prüfung auf und gibt den Bericht aus. Fehler in der Konfiguration fängt sie ab, schreibt eine verständliche Meldung auf die *Fehlerausgabe* (`stderr`, nicht auf die normale Ausgabe, damit eine Weiterverarbeitung der Ausgabe nicht gestört wird) und liefert den Exit-Code 3. Alle anderen Teile des Programms bleiben dabei unberührt; die Kommandozeile ist nur der Dirigent.

Wir erweitern die Testdatei `tests/test_cli.py`, behalten den Versionstest und fügen Ende-zu-Ende-Tests hinzu. Sie legen eine Konfiguration und Sicherungsdateien in einem temporären Verzeichnis an, rufen `main` auf und prüfen Exit-Code und Ausgabe. Weil die Kommandozeile die aktuelle Zeit selbst verwendet, setzen wir das Alter der Dateien relativ zur Gegenwart; die Spielräume (1 und 25 Stunden gegenüber der Grenze von 26) sind so gewählt, dass die Tests auch auf einem langsamen Rechner zuverlässig bleiben.

**Datei** `tests/test_cli.py`**:**

```python
import json
import os
import time
from pathlib import Path

import pytest

from backupwacht import __version__
from backupwacht.cli import main


def lege_sicherung(ordner: Path, name: str, alter_stunden: float) -> None:
    ordner.mkdir(exist_ok=True)
    pfad = ordner / name
    pfad.write_bytes(b"x" * 100)
    zeit = time.time() - alter_stunden * 3600
    os.utime(pfad, (zeit, zeit))


def schreibe_konfig(tmp_path: Path) -> Path:
    pfad = tmp_path / "konfig.toml"
    text = f'[[pruefung]]\nname = "demo"\nverzeichnis = "{tmp_path / "sicherungen"}"\n'
    pfad.write_text(text, encoding="utf-8")
    return pfad


def test_version_wird_ausgegeben(capsys):
    with pytest.raises(SystemExit) as info:
        main(["--version"])
    assert info.value.code == 0
    assert capsys.readouterr().out.strip() == f"backupwacht {__version__}"


def test_gesunde_sicherung_ergibt_exit_0(tmp_path, capsys):
    lege_sicherung(tmp_path / "sicherungen", "neu.tar", 1)
    lege_sicherung(tmp_path / "sicherungen", "alt.tar", 25)
    assert main(["-c", str(schreibe_konfig(tmp_path))]) == 0
    ausgabe = capsys.readouterr().out
    assert "demo" in ausgabe
    assert "Gesamt: OK" in ausgabe


def test_alte_sicherung_ergibt_exit_2(tmp_path, capsys):
    lege_sicherung(tmp_path / "sicherungen", "a.tar", 40)
    lege_sicherung(tmp_path / "sicherungen", "b.tar", 60)
    assert main(["-c", str(schreibe_konfig(tmp_path))]) == 2
    assert "Gesamt: KRITISCH" in capsys.readouterr().out


def test_einzige_sicherung_ergibt_exit_1(tmp_path):
    lege_sicherung(tmp_path / "sicherungen", "einzige.tar", 1)
    assert main(["-c", str(schreibe_konfig(tmp_path))]) == 1


def test_konfigurationsfehler_ergibt_exit_3_und_meldung_auf_stderr(tmp_path, capsys):
    assert main(["-c", str(tmp_path / "gibt_es_nicht.toml")]) == 3
    aufgenommen = capsys.readouterr()
    assert aufgenommen.out == ""
    assert "Konfigurationsfehler" in aufgenommen.err


def test_json_ausgabe(tmp_path, capsys):
    lege_sicherung(tmp_path / "sicherungen", "neu.tar", 1)
    lege_sicherung(tmp_path / "sicherungen", "alt.tar", 25)
    assert main(["-c", str(schreibe_konfig(tmp_path)), "--format", "json"]) == 0
    daten = json.loads(capsys.readouterr().out)
    assert daten["gesamt"] == "OK"
    assert daten["ergebnisse"][0]["name"] == "demo"


def test_standardkonfiguration_liegt_im_aktuellen_verzeichnis(tmp_path, monkeypatch, capsys):
    monkeypatch.chdir(tmp_path)
    assert main([]) == 3
    assert "backupwacht.toml" in capsys.readouterr().err


def test_unbekanntes_format_wird_abgelehnt():
    with pytest.raises(SystemExit) as info:
        main(["--format", "xml"])
    assert info.value.code == 2
```

Zwei Dinge sind bemerkenswert. Erstens prüft der Test mit dem Konfigurationsfehler, dass die normale Ausgabe leer bleibt und die Meldung auf der Fehlerausgabe erscheint; das hat in der Praxis Folgen, weil eine Weiterverarbeitung (`backupwacht --format json | jq`) sonst mit Fehlertext vermischt würde. Zweitens nutzt der Test des Standardwerts `monkeypatch.chdir`, um in ein leeres Verzeichnis zu wechseln, in dem es die Standarddatei `backupwacht.toml` nicht gibt; nach dem Test wird das Arbeitsverzeichnis automatisch zurückgesetzt. Die Tests scheitern jetzt noch, weil die Kommandozeile nur `--version` kennt.

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -4
FAILED tests/test_cli.py::test_konfigurationsfehler_ergibt_exit_3_und_meldung_auf_stderr
FAILED tests/test_cli.py::test_json_ausgabe - SystemExit: 2
FAILED tests/test_cli.py::test_standardkonfiguration_liegt_im_aktuellen_verzeichnis
6 failed, 31 passed in 0.01s
```

Jetzt die Umsetzung. Gegenüber dem Gerippe aus 15.1 wird der Parser in eine eigene Funktion verlegt, damit `main` übersichtlich bleibt, und `main` führt die vier Schritte aus. Die Konstante für den Exit-Code 3 bekommt einen Namen, damit im Code keine „magische Zahl“ steht.

**Datei** `src/backupwacht/cli.py`**:**

```python
"""Kommandozeile von backupwacht."""
import argparse
import sys

from . import __version__
from .bericht import als_json, als_text, exit_code, gesamtstufe
from .config import KonfigurationsFehler, lade_konfiguration
from .pruefung import pruefe_ziel

EXIT_KONFIGURATIONSFEHLER = 3


def baue_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="backupwacht", description="Prueft Datensicherungen.")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    parser.add_argument(
        "-c",
        "--config",
        default="backupwacht.toml",
        help="Konfigurationsdatei (Standard: %(default)s)",
    )
    parser.add_argument(
        "--format",
        choices=["text", "json"],
        default="text",
        help="Ausgabeformat (Standard: %(default)s)",
    )
    return parser


def main(argv: list[str] | None = None) -> int:
    args = baue_parser().parse_args(argv)
    try:
        konfig = lade_konfiguration(args.config)
    except KonfigurationsFehler as fehler:
        print(f"Konfigurationsfehler: {fehler}", file=sys.stderr)
        return EXIT_KONFIGURATIONSFEHLER
    ergebnisse = [pruefe_ziel(ziel) for ziel in konfig.pruefungen]
    print(als_json(ergebnisse) if args.format == "json" else als_text(ergebnisse))
    return exit_code(gesamtstufe(ergebnisse))
```

Die Zeile mit dem bedingten Ausdruck (`A if Bedingung else B`) wählt je nach Argument das Ausgabeformat. Die Option `choices` lässt `argparse` ungültige Formate selbst ablehnen, mit Exit-Code 2 und einer Meldung; dafür ist der letzte Test da. Wir lassen die gesamte Kette laufen.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 9 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/backupwacht$ pytest -q
.....................................                                    [100%]
37 passed in 0.01s
```

Alles besteht. Die Kommandozeile selbst enthält kaum Logik und ist deshalb einfach zu prüfen, während die Logik in den Teilen steckt, die wir schon einzeln getestet haben. Wir sichern den Stand.

```pycon
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "Bericht (Text, JSON, Gesamtstufe) und Kommandozeile mit Exit-Codes"
(.venv) ~/backupwacht$ git log --oneline -3
a18c7ac Bericht (Text, JSON, Gesamtstufe) und Kommandozeile mit Exit-Codes
3c795ff Sicherungsziel pruefen: Alter, Groesse, Rotation
5378f42 Konfiguration aus TOML einlesen und pruefen
```

**Der erste echte Lauf.** Tests sind gut, aber ein Programm ist erst fertig, wenn man es selbst benutzt hat. Wir bauen eine kleine Umgebung mit drei Sicherungszielen, die je einen anderen Fall zeigen: eine gesunde Datenbanksicherung (die wir schon in 15.2 angelegt haben), ein Verzeichnis mit einer zu alten Sicherung und ein Verzeichnis mit nur einer, aber frischen Datei. Dazu eine Konfiguration, die auf diese Verzeichnisse zeigt. Der Pfad `~/demo` steht dabei für das Heimverzeichnis; in der Konfiguration brauchen wir den ausgeschriebenen Pfad, weil TOML keine Tilde auflöst.

```pycon
(.venv) ~/backupwacht$ mkdir -p ~/demo/dateien ~/demo/konfig
(.venv) ~/backupwacht$ head -c 4000 /dev/zero > ~/demo/dateien/alles.tar
(.venv) ~/backupwacht$ touch -d "30 hours ago" ~/demo/dateien/alles.tar
(.venv) ~/backupwacht$ head -c 2000 /dev/zero > ~/demo/konfig/einstellungen.tar
(.venv) ~/backupwacht$ echo "$HOME"
/home/max
```

**Datei** `../demo/backupwacht.toml`**:**

```text
[[pruefung]]
name = "datenbank"
verzeichnis = "/home/max/demo/db"
muster = "*.sql.gz"
min_groesse_byte = 1000

[[pruefung]]
name = "dateien"
verzeichnis = "/home/max/demo/dateien"
muster = "*.tar"

[[pruefung]]
name = "einstellungen"
verzeichnis = "/home/max/demo/konfig"
muster = "*.tar"
```

Die Pfade in dieser Datei sind ein Platzhalter; in Wirklichkeit gehört dort Ihr eigenes Heimverzeichnis hin. Der folgende Befehl ersetzt den Platzhalter `/home/max` durch den Wert der Umgebungsvariablen `HOME`.

```pycon
(.venv) ~/backupwacht$ sed -i "s|/home/max|$HOME|" ~/demo/backupwacht.toml
```

Jetzt der erste Lauf. Wir hängen den Exit-Code an die Ausgabe, damit er sichtbar wird.

```pycon
(.venv) ~/backupwacht$ backupwacht -c ~/demo/backupwacht.toml; echo "Exit-Code: $?"
OK       datenbank: nacht_2.sql.gz, 3.0 Stunden alt
KRITISCH dateien: alles.tar ist 30.0 Stunden alt (erlaubt: 26)
WARNUNG  einstellungen: nur eine Sicherung vorhanden: einstellungen.tar
Gesamt: KRITISCH
Exit-Code: 2
```

Alle drei Fälle erscheinen in der Ausgabe: Die Datenbank ist in Ordnung, die Dateisicherung ist mit rund 30 Stunden zu alt, und bei den Einstellungen gibt es nur eine Sicherungsdatei, was als Warnung gilt. Die Gesamtstufe ist deshalb kritisch und der Exit-Code 2. Die Altersangaben hängen von der Uhr Ihres Rechners ab und weichen um Minuten ab. Nun das JSON-Format und der Aufruf als Modul, wie ihn `__main__.py` ermöglicht. Das Werkzeug `jq`, das JSON lesbar filtert, ist optional; hier nutzen wir stattdessen Python als Filter, damit nichts Zusätzliches nötig ist.

```pycon
(.venv) ~/backupwacht$ backupwacht -c ~/demo/backupwacht.toml --format json | python3 -c "import json, sys; d = json.load(sys.stdin); print(d['gesamt'], [e['name'] + '=' + e['stufe'] for e in d['ergebnisse']])"
KRITISCH ['datenbank=OK', 'dateien=KRITISCH', 'einstellungen=WARNUNG']
(.venv) ~/backupwacht$ python3 -m backupwacht --version
backupwacht 0.1.0
(.venv) ~/backupwacht$ backupwacht -c ~/demo/fehlt.toml; echo "Exit-Code: $?"
Konfigurationsfehler: Konfigurationsdatei nicht gefunden: /home/max/demo/fehlt.toml
Exit-Code: 3
```

Der letzte Aufruf zeigt den Konfigurationsfehler: eine verständliche Meldung statt eines Tracebacks und der Exit-Code 3. Damit sind alle Wege des Programms einmal durchlaufen.

**Zeitgesteuert starten.** Ein Wächter, den man von Hand starten muss, ist keiner. In Modul 9 haben Sie gesehen, wie `cron` und `systemd`-Zeitgeber Programme regelmäßig starten. Für `backupwacht` bietet sich ein `systemd`-Zeitgeber an: Die *Service-Datei* beschreibt, was gestartet wird, die *Timer-Datei*, wann. Weil das Programm mit einem Exit-Code ungleich null endet, sobald etwas nicht stimmt, markiert `systemd` die Einheit als fehlgeschlagen; das sehen Sie mit `systemctl --failed`, und Überwachungssysteme können darauf reagieren. Wir legen beide Dateien als Vorlage im Projekt ab.

**Datei** `contrib/backupwacht.service`**:**

```text
[Unit]
Description=Prueft Datensicherungen (backupwacht)

[Service]
Type=oneshot
ExecStart=/opt/backupwacht/bin/backupwacht --config /etc/backupwacht.toml
```

**Datei** `contrib/backupwacht.timer`**:**

```text
[Unit]
Description=Taeglich die Sicherungen pruefen

[Timer]
OnCalendar=*-*-* 07:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Die Service-Datei verwendet `Type=oneshot`, weil das Programm kurz läuft und sich beendet. Die Pfade sind Annahmen: Das Programm liegt in einer eigenen virtuellen Umgebung unter `/opt/backupwacht` (Modul 13), die Konfiguration unter `/etc`. Die Timer-Datei startet den Dienst täglich um 7:30 Uhr; `Persistent=true` holt einen verpassten Lauf nach, wenn der Rechner zur geplanten Zeit ausgeschaltet war. Installiert werden beide Dateien nach `/etc/systemd/system/`, dann folgen `systemctl daemon-reload` und `systemctl enable --now backupwacht.timer`. Wir haben diese Dateien in unserer Arbeitsumgebung nicht installiert, weil dort kein `systemd` läuft; sie folgen dem Aufbau aus Modul 9. Sichern wir sie im Repository.

```pycon
(.venv) ~/backupwacht$ git add contrib
(.venv) ~/backupwacht$ git commit -q -m "systemd-Vorlagen fuer den taeglichen Lauf"
(.venv) ~/backupwacht$ git log --oneline -5
cef0774 systemd-Vorlagen fuer den taeglichen Lauf
a18c7ac Bericht (Text, JSON, Gesamtstufe) und Kommandozeile mit Exit-Codes
3c795ff Sicherungsziel pruefen: Alter, Groesse, Rotation
5378f42 Konfiguration aus TOML einlesen und pruefen
1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
```

**Typische Fehler und Stolperfallen.** Der häufigste Fehler an dieser Stelle ist, Ausgabe und Fehlermeldung zu vermischen: Wer `print(...)` für Fehler benutzt, landet auf der normalen Ausgabe, und eine Weiterverarbeitung bricht, sobald Fehltext dazwischen steht. Fehlermeldungen gehören nach `stderr`. Ein zweiter Fehler ist, den Exit-Code zu vergessen: Wer in `main` keinen Wert zurückgibt, bekommt immer 0, und die Überwachung sieht nie einen Fehler. Prüfen Sie nach jeder Änderung mit `echo $?`. Drittens passiert es leicht, in der Kommandozeile Logik unterzubringen, die dort nicht hingehört (etwa die Berechnung der Gesamtstufe); dann lässt sie sich nur noch über die Kommandozeile testen. Viertens: Relative Pfade in der Konfiguration beziehen sich auf das Arbeitsverzeichnis des Aufrufers, und das ist bei `cron` und `systemd` ein anderes als in Ihrer Shell; verwenden Sie in Betriebsdateien absolute Pfade. Fünftens ist ein Test, der auf der Uhr beruht, ohne Spielraum anfällig: Unsere Tests legen die Dateien eine Stunde bzw. 25 Stunden alt an, nicht 26 Stunden genau, weil sonst wenige Millisekunden Laufzeit über Erfolg entscheiden könnten. Falsch und richtig nebeneinander:

```python
# falsch: Fehler auf stdout, kein Exit-Code
def main(argv=None):
    try:
        konfig = lade_konfiguration(args.config)
    except KonfigurationsFehler as fehler:
        print("Fehler:", fehler)

# richtig: Fehler auf stderr, eindeutiger Exit-Code
def main(argv=None):
    try:
        konfig = lade_konfiguration(args.config)
    except KonfigurationsFehler as fehler:
        print(f"Konfigurationsfehler: {fehler}", file=sys.stderr)
        return EXIT_KONFIGURATIONSFEHLER
```

Im falschen Beispiel läuft das Programm nach der Meldung weiter, kennt `konfig` nicht und bricht an späterer Stelle mit einem undeutlichen `NameError` ab; im richtigen Beispiel beendet die Rückgabe die Funktion sofort mit dem passenden Code.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Exit-Codes 0, 1 und 2 für OK, Warnung und kritisch folgen einer in der Überwachung weit verbreiteten Konvention (bekannt von Nagios-kompatiblen Werkzeugen); sie wurde hier nicht aus einer abgerufenen Quelle belegt, und Sie können die Werte in Ihrem Umfeld abweichend festlegen. Die Programme in dieser Einheit wurden mit Python 3.13.18 und pytest 9.2.0 ausgeführt. Die `systemd`-Vorlagen wurden *nicht* in einer Umgebung mit `systemd` ausprobiert; prüfen Sie sie vor dem Einsatz mit `systemd-analyze verify` und beachten Sie die Dokumentation Ihrer Distribution.

> **Merke**
>
> Die Kommandozeile ist nur der Dirigent: Argumente lesen, Teile aufrufen, ausgeben, Exit-Code liefern. Fehlermeldungen gehören auf `stderr`, nützliche Ergebnisse auf `stdout`. Legen Sie die Bedeutung der Exit-Codes fest und dokumentieren Sie sie. Rückgabewert von `main` und Exit-Code des Prozesses sind dasselbe, solange der Einstieg `sys.exit(main())` aufruft. Testen Sie die Kommandozeile mit `main(argv)` direkt, statt einen Prozess zu starten; das ist schneller und liefert dieselbe Aussage.

> **Praxisbeispiel**
>
> Ein Werkzeug wie `backupwacht` wird fast immer von einem anderen Programm gestartet, nicht von einem Menschen: von `cron`, einem `systemd`-Zeitgeber oder einem Überwachungssystem. Dieses sieht nur den Exit-Code und gegebenenfalls die Ausgabe. Wer Exit-Codes sauber festlegt und JSON anbietet, macht sein Werkzeug anschlussfähig, ohne dafür Code für jedes Zielsystem zu schreiben.

## 15.4 Abschlussprojekt: Test, Review und Fehlerbehebung

Das Programm läuft, und seine 37 Tests bestehen. Heißt das, dass es richtig ist? Nein, es heißt nur, dass es sich so verhält, wie wir es *geprüft* haben. Diese Einheit beschäftigt sich mit der Frage, wie man die Lücken zwischen „getestet“ und „richtig“ findet. Wir verwenden dafür drei Werkzeuge, die ein ordentliches Projekt vor der Fertigstellung durchläuft: die Messung der *Testabdeckung*, die *Durchsicht* (englisch *code review*) und die *Fehlerbehebung nach dem Test-zuerst-Prinzip*. Am Ende steht ein robusteres Programm und, mindestens so wichtig, ein geübter Blick dafür, wo Programme typischerweise brechen.

**Testabdeckung messen.** Die Testabdeckung (englisch *coverage*) gibt an, welcher Anteil der Programmzeilen beim Ausführen der Tests tatsächlich durchlaufen wurde. Das Werkzeug dafür heißt `coverage`; als `pytest`-Erweiterung bringt `pytest-cov` es in die gewohnte Befehlszeile. Wir haben es bereits in den Entwicklungswerkzeugen des Projekts (`.[dev]`) mitinstalliert. Zusätzlich zur Zeilenabdeckung messen wir die *Zweigabdeckung* (englisch *branch coverage*): Sie prüft nicht nur, ob jede Zeile lief, sondern auch, ob bei jeder Verzweigung (`if`, Schleife) *beide* Wege genommen wurden. Eine Zeile wie `if x: return 1` ist zeilenweise schon abgedeckt, wenn die Bedingung einmal wahr war; die Zweigabdeckung verlangt auch den Fall „falsch“. Die Einstellungen legen wir in der `pyproject.toml` ab, damit der Befehl kurz bleibt und jeder im Team dasselbe misst: `branch = true` schaltet die Zweigabdeckung ein, `source` nennt das Paket, das gemessen werden soll (sonst würden auch die Tests selbst mitgezählt), und `show_missing` zeigt die Nummern der nicht ausgeführten Zeilen.

```pycon
(.venv) ~/backupwacht$ printf '\n[tool.coverage.run]\nbranch = true\nsource = ["backupwacht"]\n\n[tool.coverage.report]\nshow_missing = true\n' >> pyproject.toml
(.venv) ~/backupwacht$ tail -7 pyproject.toml

[tool.coverage.run]
branch = true
source = ["backupwacht"]

[tool.coverage.report]
show_missing = true
```

Der Befehl hängt die drei Abschnitte an die Projektdatei an. Jetzt die Messung mit `pytest --cov`. Die Tabelle zeigt pro Datei die Anzahl der Anweisungen (*Stmts*), die nicht ausgeführten (*Miss*), die Verzweigungen (*Branch*), die davon nur teilweise durchlaufenen (*BrPart*) und den Prozentwert; die letzte Spalte nennt die Zeilen, die nie liefen.

```pycon
(.venv) ~/backupwacht$ pytest -q --cov
.....................................                                    [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      3      0      0     0%   3-7
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      30      0     10      0   100%
-------------------------------------------------------------------------
TOTAL                           126      3     30      0    98%
37 passed in 0.01s
```

Der Befund ist eindeutig: Alles ist abgedeckt bis auf die Datei `__main__.py`, deren drei Zeilen nie ausgeführt wurden. Das ist verständlich, denn die Tests rufen `main` direkt auf und nie den Weg über `python -m backupwacht`. Trotzdem ist es eine echte Lücke: Würde jemand in dieser Datei einen Tippfehler machen, bliebe er unbemerkt, bis ein Anwender ihn findet. Wir schließen sie mit einem Test, der das Modul mit `runpy` ausführt, also so, als hätte man es mit `-m` gestartet, aber im selben Prozess, damit die Abdeckung mitgezählt wird. Das Argument `run_name="__main__"` sorgt dafür, dass der Code so läuft, als wäre das Modul das Hauptprogramm. Die Argumente setzen wir mit `monkeypatch` auf `sys.argv`, weil `main` ohne Parameter dort liest.

**Datei** `tests/test_main.py`**:**

```python
import runpy
import sys

import pytest


def test_modulaufruf_liefert_die_version(monkeypatch, capsys):
    monkeypatch.setattr(sys, "argv", ["backupwacht", "--version"])
    with pytest.raises(SystemExit) as info:
        runpy.run_module("backupwacht", run_name="__main__")
    assert info.value.code == 0
    assert capsys.readouterr().out.startswith("backupwacht ")
```

Warum `SystemExit`? Die Datei `__main__.py` ruft `sys.exit(main())` auf, und `sys.exit` beendet den Prozess, indem es diese Ausnahme auslöst. Der Test fängt sie ab und prüft den Exit-Code, genau wie wir es schon beim Versionstest getan haben. Wir messen erneut.

```pycon
(.venv) ~/backupwacht$ pytest -q --cov
......................................                                   [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      30      0     10      0   100%
-------------------------------------------------------------------------
TOTAL                           126      0     30      0   100%
38 passed in 0.01s
```

Hundert Prozent. Jetzt kommt der wichtigste Satz dieser Einheit: *Das beweist nicht, dass das Programm fehlerfrei ist.* Die Abdeckung zeigt nur, was *nie* geprüft wurde, nicht, ob das, was geprüft wurde, ausreichend geprüft ist. Ein Test, der eine Funktion aufruft und nichts überprüft, erhöht die Abdeckung genauso wie ein gründlicher Test. Betrachten Sie die Zahl deshalb als Frühwarnsystem für vergessene Ecken, nicht als Ziel. Wer die Zahl zum Ziel macht, schreibt Tests, die nur Zeilen abhaken. Man sollte stattdessen mit Verstand fragen: Welche Fälle sind denkbar, und welche davon prüfe ich?

**Die Durchsicht.** Genau diese Frage stellt die Durchsicht. Man liest den eigenen Code, möglichst nach einer Pause und mit einer Prüfliste, so als hätte ein Fremder ihn geschrieben. Im Team übernimmt das ein Kollege, in einem Einzelprojekt müssen Sie es selbst tun, und das Verfahren funktioniert überraschend gut. Eine brauchbare Prüfliste für ein Programm wie unseres hat fünf Gruppen von Fragen. Erstens die *Eingaben*: Was passiert bei leerer, fehlender, riesiger oder verdrehter Eingabe? Zweitens die *Umgebung*: Was geschieht, wenn das Dateisystem nicht mitspielt, wenn Rechte fehlen, wenn eine Datei zwischen zwei Zugriffen verschwindet, wenn die Uhr falsch geht? Drittens die *Grenzen*: Stimmen die Vergleiche an den Rändern? Viertens die *Fehlermeldungen*: Erklärt die Meldung, was zu tun ist? Fünftens die *Lesbarkeit*: Sagen Namen und Kommentare, was gemeint ist, und ist jede Funktion kurz genug, um sie auf einmal zu verstehen?

Wenden wir die Liste auf `pruefung.py` an, finden wir bei der Umgebung zwei Schwächen. Die erste betrifft die Uhr: Hat eine Sicherungsdatei einen Zeitstempel in der *Zukunft* (etwa weil die Uhr des Servers falsch ging, als sie geschrieben wurde), ist ihr „Alter“ negativ, und das Programm meldet freundlich OK, weil sie ja nicht zu alt ist. Eine solche Meldung ist trügerisch, denn man weiß nicht, wann die Sicherung wirklich entstand. Wir wollen eine Warnung. Die zweite Schwäche betrifft die Rechte: Ist das Verzeichnis zwar vorhanden, aber für den Benutzer nicht lesbar, löst `glob` eine `PermissionError` aus, und das Programm bricht mit einem Traceback ab, statt dem Benutzer eine verständliche Meldung zu geben und die übrigen Ziele zu prüfen. Beides sind keine Programmierfehler im engen Sinn, sondern Annahmen („die Welt ist in Ordnung“), die im Betrieb früher oder später falsch werden.

**Fehlerbehebung nach dem Test-zuerst-Prinzip.** Für jeden gefundenen Fehler gilt dieselbe Reihenfolge: Erst einen Test schreiben, der den Fehler zeigt, und zusehen, wie er scheitert. Dann den Fehler beheben, bis der Test besteht. Dieser Ablauf hat zwei Zwecke. Der Test beweist, dass wir den Fehler wirklich verstanden haben, denn wir können ihn auslösen; und er bleibt für immer im Projekt und verhindert, dass der Fehler unbemerkt zurückkehrt. Ein Fehler ohne Test wird in aller Regel wiederkommen. Wir arbeiten in einem eigenen Zweig, wie es in 13.2 gelernt wurde, damit `main` jederzeit lauffähig bleibt.

```pycon
(.venv) ~/backupwacht$ git switch -c fix/robustheit
Switched to a new branch 'fix/robustheit'
```

Die beiden Tests liegen in einer neuen Datei. Der erste legt zwei Dateien an, von denen eine einen Zeitstempel fünf Stunden in der Zukunft hat. Der zweite ersetzt mit `monkeypatch` die Methode `Path.glob` durch eine Funktion, die den Fehler auslöst. Das ist hier der saubere Weg, denn Rechte lassen sich in dieser Umgebung nicht verlässlich entziehen: Wer als Administrator (`root`) arbeitet, darf trotzdem lesen, und der Test würde dort nicht fehlschlagen. Ein solcher Test, der nur unter bestimmten Benutzern funktioniert, wäre selbst eine Fehlerquelle.

**Datei** `tests/test_robustheit.py`**:**

```python
import os
from pathlib import Path

from backupwacht.config import Pruefung
from backupwacht.pruefung import pruefe_ziel

JETZT = 1_800_000_000.0


def lege_datei(ordner: Path, name: str, alter_stunden: float) -> None:
    pfad = ordner / name
    pfad.write_bytes(b"x" * 100)
    zeit = JETZT - alter_stunden * 3600
    os.utime(pfad, (zeit, zeit))


def test_zeitstempel_in_der_zukunft_ist_eine_warnung(tmp_path):
    lege_datei(tmp_path, "zukunft.tar", -5)
    lege_datei(tmp_path, "alt.tar", 20)
    ergebnis = pruefe_ziel(Pruefung("test", tmp_path), JETZT)
    assert ergebnis.stufe == "WARNUNG"
    assert "Zukunft" in ergebnis.meldung


def test_nicht_lesbares_verzeichnis_ist_kritisch(tmp_path, monkeypatch):
    def verweigert(self, muster):
        raise PermissionError("Zugriff verweigert")

    monkeypatch.setattr(Path, "glob", verweigert)
    ergebnis = pruefe_ziel(Pruefung("test", tmp_path), JETZT)
    assert ergebnis.stufe == "KRITISCH"
    assert "nicht lesbar" in ergebnis.meldung
```

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -6

tests/test_robustheit.py:27: PermissionError
=========================== short test summary info ============================
FAILED tests/test_robustheit.py::test_zeitstempel_in_der_zukunft_ist_eine_warnung
FAILED tests/test_robustheit.py::test_nicht_lesbares_verzeichnis_ist_kritisch
2 failed, 38 passed in 0.01s
```

Beide neuen Tests scheitern, und zwar aus genau den erwarteten Gründen: Der erste bekommt `OK` statt `WARNUNG`, der zweite die ungefangene `PermissionError`. Jetzt die Korrektur in `pruefung.py`. Alle Zugriffe auf das Dateisystem, die scheitern können, kommen in einen `try`-Block, der `OSError` fängt; `PermissionError` und `FileNotFoundError` sind beides Unterklassen davon (Modul 4). Das deckt nebenbei den Fall ab, dass eine Datei zwischen dem Auflisten und dem Lesen ihrer Daten verschwindet. Dazu kommt die Prüfung auf eine negative Altersangabe, die wir vor der Prüfung auf das Höchstalter einfügen.

**Datei** `src/backupwacht/pruefung.py`**:**

```python
"""Pruefung eines Sicherungsziels im Dateisystem."""
import time
from dataclasses import dataclass
from typing import Literal

from .config import Pruefung

Stufe = Literal["OK", "WARNUNG", "KRITISCH"]


@dataclass(frozen=True)
class Ergebnis:
    """Ergebnis der Pruefung eines Ziels."""

    name: str
    stufe: Stufe
    meldung: str
    alter_stunden: float | None = None


def pruefe_ziel(ziel: Pruefung, jetzt: float | None = None) -> Ergebnis:
    """Prueft ein Sicherungsziel; jetzt (Sekunden seit 1970) dient dem Testen."""
    jetzt = time.time() if jetzt is None else jetzt
    if not ziel.verzeichnis.is_dir():
        return Ergebnis(ziel.name, "KRITISCH", f"Verzeichnis fehlt: {ziel.verzeichnis}")
    try:
        dateien = [d for d in ziel.verzeichnis.glob(ziel.muster) if d.is_file()]
        if not dateien:
            return Ergebnis(ziel.name, "KRITISCH", f"keine Datei passt zu '{ziel.muster}'")
        neueste = max(dateien, key=lambda d: d.stat().st_mtime)
        status = neueste.stat()
    except OSError as fehler:
        return Ergebnis(ziel.name, "KRITISCH", f"nicht lesbar: {fehler}")
    alter = (jetzt - status.st_mtime) / 3600
    if status.st_size < ziel.min_groesse_byte:
        text = f"{neueste.name} ist nur {status.st_size} Byte gross"
        return Ergebnis(ziel.name, "KRITISCH", text, alter)
    if alter < 0:
        text = f"Zeitstempel von {neueste.name} liegt in der Zukunft"
        return Ergebnis(ziel.name, "WARNUNG", text, alter)
    if alter > ziel.max_alter_stunden:
        text = f"{neueste.name} ist {alter:.1f} Stunden alt (erlaubt: {ziel.max_alter_stunden:g})"
        return Ergebnis(ziel.name, "KRITISCH", text, alter)
    if len(dateien) < 2:
        return Ergebnis(ziel.name, "WARNUNG", f"nur eine Sicherung vorhanden: {neueste.name}", alter)
    return Ergebnis(ziel.name, "OK", f"{neueste.name}, {alter:.1f} Stunden alt", alter)
```

Beachten Sie, dass die Prüfung auf eine leere Liste innerhalb des `try`-Blocks bleibt: Die Liste entsteht dort, und die Rückgabe mit `return` verlässt die Funktion unabhängig vom `try`. Wir lassen die Kette laufen.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 11 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/backupwacht$ pytest -q --cov
........................................                                 [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      36      0     12      0   100%
-------------------------------------------------------------------------
TOTAL                           132      0     32      0   100%
40 passed in 0.01s
```

Alle 40 Tests bestehen, die Abdeckung bleibt bei hundert Prozent, und die beiden Schwächen sind behoben und für immer durch Tests bewacht. Bevor wir den Zweig zusammenführen, sehen wir uns an, was er gegenüber `main` geändert hat. Das ist die Durchsicht der Änderung, die man im Team als *Pull Request* oder *Merge Request* bekommt (13.6): Die Statistik zeigt, welche Dateien betroffen sind, damit man gezielt hinsieht.

```pycon
(.venv) ~/backupwacht$ git add src tests pyproject.toml
(.venv) ~/backupwacht$ git commit -q -m "Zeitstempel in der Zukunft und nicht lesbare Verzeichnisse behandeln"
(.venv) ~/backupwacht$ git diff main --stat
 pyproject.toml              |  7 +++++++
 src/backupwacht/pruefung.py | 16 +++++++++++-----
 tests/test_main.py          | 12 ++++++++++++
 tests/test_robustheit.py    | 32 ++++++++++++++++++++++++++++++++
 4 files changed, 62 insertions(+), 5 deletions(-)
(.venv) ~/backupwacht$ git switch main
Switched to branch 'main'
(.venv) ~/backupwacht$ git merge --no-ff -m "Robustheit: Zukunft-Zeitstempel und Lesefehler" fix/robustheit
Merge made by the 'ort' strategy.
 pyproject.toml              |  7 +++++++
 src/backupwacht/pruefung.py | 16 +++++++++++-----
 tests/test_main.py          | 12 ++++++++++++
 tests/test_robustheit.py    | 32 ++++++++++++++++++++++++++++++++
 4 files changed, 62 insertions(+), 5 deletions(-)
 create mode 100644 tests/test_main.py
 create mode 100644 tests/test_robustheit.py
(.venv) ~/backupwacht$ git log --oneline --graph
*   445559f Robustheit: Zukunft-Zeitstempel und Lesefehler
|\  
| * 7e59ab4 Zeitstempel in der Zukunft und nicht lesbare Verzeichnisse behandeln
|/  
* cef0774 systemd-Vorlagen fuer den taeglichen Lauf
* a18c7ac Bericht (Text, JSON, Gesamtstufe) und Kommandozeile mit Exit-Codes
* 3c795ff Sicherungsziel pruefen: Alter, Groesse, Rotation
* 5378f42 Konfiguration aus TOML einlesen und pruefen
* 1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
* 95754fb Entwurf und README anlegen
```

Mit `--no-ff` (englisch *no fast-forward*) erzeugt Git einen eigenen Zusammenführungs-Commit, auch wenn eine einfache Vorwärtsbewegung möglich wäre. Der Verlauf zeigt dann deutlich, dass hier ein Zweig mit einer abgeschlossenen Aufgabe eingeflossen ist. Ob man das so haben will, ist Geschmackssache; wichtig ist, dass das Team sich auf eine Linie einigt. Den erledigten Zweig räumen wir auf.

```pycon
(.venv) ~/backupwacht$ git branch -d fix/robustheit
Deleted branch fix/robustheit (was 7e59ab4).
(.venv) ~/backupwacht$ git branch
* main
```

**Was die Durchsicht nicht behebt.** Ehrlich bleibt auch die Frage, was wir nicht getan haben. Unsere Durchsicht hat einige Punkte gefunden, die wir bewusst offen lassen: Zwei Ziele dürfen denselben Namen tragen (das ist unschön, aber harmlos); das Muster darf einen Pfad enthalten, zum Beispiel `unterordner/*.tar`, was durchaus erwünscht sein kann; und bei sehr großen Verzeichnissen mit Millionen Dateien wird die Liste im Speicher aufgebaut, was bei einem Sicherungsverzeichnis keine Rolle spielt. Solche Entscheidungen gehören aufgeschrieben, am besten in den Abschnitt „Bekannte Einschränkungen“ der `README.md`, den wir in 15.5 anlegen. Wer ehrlich dokumentiert, was nicht geht, erspart den Anwendern Überraschungen.

**Typische Fehler und Stolperfallen.** Der verbreitetste Fehler ist, die Abdeckung als Qualitätsmaß zu verstehen: Hundert Prozent mit Tests ohne Prüfungen (`assert`) sind wertlos, und niedrige Prozent in unwichtigem Code sind kein Drama. Der zweite ist der Test, der ohne Not vom Benutzer oder vom Betriebssystem abhängt, wie ein Test mit `chmod 000`, der als `root` nicht funktioniert; solche Tests bestehen auf dem einen Rechner und scheitern auf dem anderen. Der dritte ist, einen Fehler zu beheben, ohne ihn vorher durch einen Test sichtbar zu machen: Dann weiß man nicht, ob der Test den Fehler wirklich fängt. Vierter Fehler: zu viel auf einmal in einen Zweig stecken. Eine Fehlerbehebung pro Zweig und pro Commit ist übersichtlich und lässt sich notfalls rückgängig machen. Fünftens: die eigene Durchsicht direkt nach dem Schreiben. Mit Abstand sieht man mehr, und eine Prüfliste hilft, nichts zu vergessen. Der Unterschied zwischen einem schwachen und einem aussagekräftigen Test zeigt sich an folgendem Gegenüber:

```python
# falsch: erhoeht die Abdeckung, prueft aber nichts
def test_pruefung_laeuft(tmp_path):
    pruefe_ziel(Pruefung("t", tmp_path))

# richtig: legt den erwarteten Fall fest
def test_leeres_verzeichnis_ist_kritisch(tmp_path):
    ergebnis = pruefe_ziel(Pruefung("t", tmp_path))
    assert ergebnis.stufe == "KRITISCH"
    assert "keine Datei" in ergebnis.meldung
```

Im ersten Test würde die Funktion jede beliebige Stufe liefern, und der Test bliebe grün. Im zweiten legen wir das erwartete Verhalten fest; ändert jemand die Funktion unbeabsichtigt, schlägt der Test an.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Befehle und Ausgaben dieser Einheit stammen aus echten Läufen mit Python 3.13.18, pytest 9.2.0 und Git 2.43.0. Die Konfigurationsschlüssel von `coverage` (`branch`, `source`, `show_missing`) wurden durch den Lauf bestätigt, aber nicht in der Dokumentation nachgeschlagen; die Dokumentation von `coverage.py` und `pytest-cov` gehört in die Liste der weiterführenden Quellen. Die Prüfliste für die Durchsicht ist eine in der Praxis übliche Zusammenstellung, kein genormtes Verfahren.

> **Merke**
>
> Die Abdeckung zeigt, was nie getestet wurde, nicht, ob genug getestet wurde; messen Sie auch die Zweige (`branch = true`). Gehen Sie den eigenen Code mit einer Prüfliste durch: Eingaben, Umgebung, Grenzen, Fehlermeldungen, Lesbarkeit. Jeder gefundene Fehler bekommt zuerst einen fehlschlagenden Test, dann die Korrektur, in einem eigenen Zweig. Tests dürfen nicht vom Benutzer, der Uhrzeit oder den Dateirechten abhängen; ersetzen Sie solche Abhängigkeiten mit `monkeypatch`. Halten Sie bewusst offene Punkte schriftlich fest.

> **Praxisbeispiel**
>
> In der Praxis bricht Software selten an den Stellen, die der Entwickler im Kopf hatte, sondern an den Rändern: volle Platten, fehlende Rechte, falsch gehende Uhren, Dateien, die gerade gelöscht werden. Eine Durchsicht mit Prüfliste und ein Zweig pro Fehlerbehebung sind Gewohnheiten, die man im Berufsleben ständig braucht, ob im Team mit Pull Requests oder allein für das eigene Skript. Dass Fehlerbehebungen mit Test beginnen, ist in vielen Firmen Regel, weil es verhindert, dass derselbe Fehler zweimal Ärger macht.

## 15.5 Abschlussprojekt: Refactoring, Dokumentation und Abschluss

Das Programm tut, was es soll, und die Tests belegen es. Was bleibt, ist die Arbeit, die ein Projekt von „läuft bei mir“ zu „kann man übergeben“ macht: den Code aufräumen, ohne sein Verhalten zu ändern, die Dokumentation vollständig machen, eine Version festlegen, ein installierbares Paket bauen und den Abschluss prüfen. Diese Einheit führt durch alle fünf Schritte. Sie ist zugleich eine Wiederholung: Fast jeder Befehl stammt aus den Modulen 13 und 14.

**Refactoring: aufräumen, ohne zu verändern.** Der englische Begriff *Refactoring* (deutsch etwa „Umgestaltung“) bezeichnet eine Änderung an der inneren Struktur des Codes, die sein äußeres Verhalten *nicht* verändert. Man tut das, weil Code, der zum Lesen mühsam ist, auch mühsam zu ändern ist und Fehler anzieht. Die entscheidende Voraussetzung sind Tests: Sie sind das Sicherheitsnetz, das meldet, wenn man versehentlich doch etwas verändert hat. Genau deshalb haben wir sie vorher geschrieben. Die Regel lautet: Refactoring und neue Funktionen niemals im selben Schritt. Erst der Test wird grün, dann wird umgebaut, dann sind alle Tests *immer noch* grün.

Betrachten wir die Funktion `pruefe_ziel` aus 15.4. Sie ist etwa 30 Zeilen lang und erledigt vier Dinge nacheinander: das Verzeichnis prüfen, die neueste Datei suchen, ihr Alter berechnen und das Ergebnis bewerten. Das Bewerten besteht aus einer Kette von Bedingungen, und in jeder Zeile wiederholt sich die Konstruktion von `Ergebnis(ziel.name, ..., alter)`. Das Programm funktioniert, aber die Funktion vermischt zwei Verantwortungen, nämlich *Dateisystem lesen* (kann scheitern, braucht ein Verzeichnis) und *Werte bewerten* (reine Rechnung, braucht nur Zahlen). Wir trennen sie in zwei kleine Hilfsfunktionen. Dafür führen wir eine kleine, unveränderliche `dataclass` `_Fund` ein, die genau das enthält, was die Bewertung wissen muss: den Namen der neuesten Datei, ihre Größe, ihre Änderungszeit und die Anzahl der gefundenen Dateien. Die Bewertung erhält damit keine Pfade mehr, sondern Zahlen, und könnte künftig ohne ein Dateisystem getestet werden. Die öffentliche Funktion `pruefe_ziel` und das Verhalten bleiben unverändert, deshalb dürfen die Tests nicht angefasst werden.

**Datei** `src/backupwacht/pruefung.py`**:**

```python
"""Pruefung eines Sicherungsziels im Dateisystem."""
import time
from dataclasses import dataclass
from typing import Literal

from .config import Pruefung

Stufe = Literal["OK", "WARNUNG", "KRITISCH"]


@dataclass(frozen=True)
class Ergebnis:
    """Ergebnis der Pruefung eines Ziels."""

    name: str
    stufe: Stufe
    meldung: str
    alter_stunden: float | None = None


@dataclass(frozen=True)
class _Fund:
    """Was die Suche im Verzeichnis ergeben hat."""

    name: str
    groesse: int
    geaendert: float
    anzahl: int


def _suche(ziel: Pruefung) -> _Fund | None:
    """Sucht die neueste passende Datei; None, wenn keine passt."""
    dateien = [d for d in ziel.verzeichnis.glob(ziel.muster) if d.is_file()]
    if not dateien:
        return None
    neueste = max(dateien, key=lambda d: d.stat().st_mtime)
    status = neueste.stat()
    return _Fund(neueste.name, status.st_size, status.st_mtime, len(dateien))


def _bewerte(ziel: Pruefung, fund: _Fund, jetzt: float) -> Ergebnis:
    """Bewertet den Fund; reine Rechnung ohne Dateizugriff."""
    alter = (jetzt - fund.geaendert) / 3600

    def ergebnis(stufe: Stufe, text: str) -> Ergebnis:
        return Ergebnis(ziel.name, stufe, text, alter)

    if fund.groesse < ziel.min_groesse_byte:
        return ergebnis("KRITISCH", f"{fund.name} ist nur {fund.groesse} Byte gross")
    if alter < 0:
        return ergebnis("WARNUNG", f"Zeitstempel von {fund.name} liegt in der Zukunft")
    if alter > ziel.max_alter_stunden:
        erlaubt = f"{ziel.max_alter_stunden:g}"
        return ergebnis("KRITISCH", f"{fund.name} ist {alter:.1f} Stunden alt (erlaubt: {erlaubt})")
    if fund.anzahl < 2:
        return ergebnis("WARNUNG", f"nur eine Sicherung vorhanden: {fund.name}")
    return ergebnis("OK", f"{fund.name}, {alter:.1f} Stunden alt")


def pruefe_ziel(ziel: Pruefung, jetzt: float | None = None) -> Ergebnis:
    """Prueft ein Sicherungsziel; jetzt (Sekunden seit 1970) dient dem Testen."""
    jetzt = time.time() if jetzt is None else jetzt
    if not ziel.verzeichnis.is_dir():
        return Ergebnis(ziel.name, "KRITISCH", f"Verzeichnis fehlt: {ziel.verzeichnis}")
    try:
        fund = _suche(ziel)
    except OSError as fehler:
        return Ergebnis(ziel.name, "KRITISCH", f"nicht lesbar: {fehler}")
    if fund is None:
        return Ergebnis(ziel.name, "KRITISCH", f"keine Datei passt zu '{ziel.muster}'")
    return _bewerte(ziel, fund, jetzt)
```

Die Hilfsfunktion `ergebnis` in `_bewerte` ist eine *verschachtelte Funktion*: Sie sieht die Variablen der äußeren Funktion (hier `ziel` und `alter`), sodass wir diese nicht bei jedem Aufruf wiederholen müssen. Die Unterstriche vor `_Fund`, `_suche` und `_bewerte` kennzeichnen sie als intern; die einzige öffentliche Funktion des Moduls bleibt `pruefe_ziel`. Die Hauptfunktion ist jetzt kurz und liest sich wie die Beschreibung: Verzeichnis prüfen, suchen, bewerten. Alle Fehler beim Zugriff auf das Dateisystem bleiben in einem einzigen `try`-Block, der nur die Suche umschließt. Jetzt der Beweis, dass sich nichts verändert hat: die gesamte Kette, zusammen mit der Abdeckung.

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 11 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/backupwacht$ pytest -q --cov
........................................                                 [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      49      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           145      0     34      0   100%
40 passed in 0.01s
```

Alle 40 Tests bestehen weiterhin, die Abdeckung liegt bei hundert Prozent, und die Typprüfung ist zufrieden. Das ist der Gewinn der Tests: Wir haben eine Funktion von Grund auf umgebaut und wissen in zwei Sekunden, dass sie sich noch genauso verhält. Ohne Tests hätte man jede Meldung von Hand ausprobieren müssen. Wir sichern den Zustand mit einem Commit, dessen Nachricht ausdrücklich sagt, dass es sich um ein Refactoring ohne Verhaltensänderung handelt.

```pycon
(.venv) ~/backupwacht$ git add src
(.venv) ~/backupwacht$ git commit -q -m "Refactoring: pruefe_ziel in Suche und Bewertung trennen"
(.venv) ~/backupwacht$ git log --oneline -3
1baf16c Refactoring: pruefe_ziel in Suche und Bewertung trennen
445559f Robustheit: Zukunft-Zeitstempel und Lesefehler
7e59ab4 Zeitstempel in der Zukunft und nicht lesbare Verzeichnisse behandeln
```

**Dokumentation: die README.** Die README ist das Erste, was jemand von Ihrem Projekt liest, und oft das Einzige. Sie sollte fünf Fragen beantworten: Was macht das Werkzeug? Wie installiere ich es? Wie benutze ich es (mit einem Beispiel)? Was bedeuten Ausgabe und Exit-Codes? Wie arbeite ich an dem Projekt mit? Dazu kommen die bekannten Einschränkungen aus der Durchsicht, die wir bewusst offen gelassen haben, und ein Hinweis zur Lizenz. Beachten Sie, dass die README in der `pyproject.toml` als `readme` eingetragen ist und beim Bauen in die Metadaten wandert. Wir ersetzen das Platzhalter-Dokument aus 15.1 durch eine vollständige Fassung. Die Markdown-Auszeichnung ist hier bewusst einfach gehalten.

**Datei** `README.md`**:**

```text
# backupwacht

Prueft, ob Datensicherungen vorhanden, frisch und nicht verdaechtig klein
sind. Das Ergebnis steht im Exit-Code und in einer lesbaren oder JSON-Ausgabe.
Gedacht fuer den Betrieb unter Linux, zum Beispiel per systemd-Timer.

## Installation

    python3 -m venv .venv
    . .venv/bin/activate
    pip install .

Benoetigt wird Python 3.11 oder neuer. Es gibt keine weiteren Abhaengigkeiten.

## Benutzung

    backupwacht --config /etc/backupwacht.toml
    backupwacht --config backupwacht.toml --format json

Ohne --config wird backupwacht.toml im aktuellen Verzeichnis gelesen.

## Konfiguration

Pro Sicherungsziel ein Abschnitt [[pruefung]] in einer TOML-Datei:

    [[pruefung]]
    name = "datenbank"             # Pflicht
    verzeichnis = "/srv/backup/db" # Pflicht, am besten absolut
    muster = "*.sql.gz"            # Standard: *
    max_alter_stunden = 26         # Standard: 26
    min_groesse_byte = 1000        # Standard: 1

Unbekannte Schluessel sind ein Fehler, damit Tippfehler auffallen.

## Ausgabe und Exit-Codes

| Code | Bedeutung                                              |
|------|--------------------------------------------------------|
| 0    | alle Ziele in Ordnung                                  |
| 1    | mindestens eine Warnung (zum Beispiel nur 1 Sicherung) |
| 2    | mindestens ein Ziel kritisch (alt, klein, fehlt)       |
| 3    | Konfigurationsfehler                                   |

Vorlagen fuer einen taeglichen Lauf mit systemd liegen in contrib/.

## Bekannte Einschraenkungen

- Geprueft werden nur Aenderungszeit und Groesse, nicht der Inhalt.
- Zwei Ziele duerfen denselben Namen tragen.
- Alle passenden Dateien eines Verzeichnisses werden im Speicher gelistet.

## Entwicklung

    pip install -e ".[dev]"
    ruff format --check && ruff check && mypy && pytest --cov

## Lizenz

Noch nicht festgelegt. Ohne Lizenz duerfen andere den Code nicht
weiterverwenden; vor einer Veroeffentlichung eine Lizenz waehlen.
```

Der Text ist so geschrieben, dass er für jemanden verständlich ist, der das Projekt zum ersten Mal sieht: ein Satz zum Zweck, ein Beispiel pro Anwendungsfall, eine Tabelle für die Exit-Codes. Die „Entwicklung“ beschreibt genau die Kette, die wir in diesem Modul benutzt haben. Auch der Hinweis zur Lizenz ist ehrlich: Wir haben noch keine gewählt. Welche für Ihr Projekt passt, ist eine Entscheidung, die nur Sie treffen können (keine Rechtsberatung; Hilfen zur Auswahl bieten Seiten wie `choosealicense.com`).

**Das Änderungsprotokoll und die Version.** Wer ein Programm weitergibt, sollte sagen, was sich zwischen Versionen geändert hat. Dafür gibt es das Änderungsprotokoll (englisch *changelog*), üblicherweise eine Datei `CHANGELOG.md`. Eine verbreitete Form gliedert jede Version nach „Neu“, „Geändert“, „Behoben“. Wir legen eines an und heben die Version auf 1.0.0 an. Nach der Regel der semantischen Versionierung (13.4) ist die 1.0.0 die erste stabile Fassung: Sie sagt dem Anwender, dass sich Konfigurationsformat und Exit-Codes künftig nur noch mit einer neuen Hauptversion unverträglich ändern. Weil wir die Version an genau einer Stelle führen (13.4), genügt es, die Konstante in `__init__.py` zu ändern; die `pyproject.toml` liest sie von dort.

**Datei** `CHANGELOG.md`**:**

```text
# Aenderungsprotokoll

## 1.0.0

### Neu
- Konfiguration in TOML mit beliebig vielen Sicherungszielen.
- Pruefung auf Alter, Mindestgroesse und Rotation (mindestens 2 Dateien).
- Ausgabe als Text oder JSON, Exit-Codes 0, 1, 2 und 3.
- Vorlagen fuer systemd-Service und -Timer in contrib/.

### Behoben
- Zeitstempel in der Zukunft ergeben eine Warnung statt OK.
- Nicht lesbare Verzeichnisse ergeben ein kritisches Ergebnis statt eines
  Programmabbruchs.

## 0.1.0

- Geruest mit Kommandozeile und --version.
```

**Datei** `src/backupwacht/__init__.py`**:**

```python
"""backupwacht - prueft Datensicherungen."""

__version__ = "1.0.0"
```

Da `pip` die Metadaten bei der editierbaren Installation einmal einliest, muss die Installation nach einer Versionsänderung erneuert werden, damit `pip list` die neue Nummer zeigt (13.4). Für unsere Tests spielt das keine Rolle, denn sie lesen die Konstante direkt. Jetzt der Abschluss der Qualitätskette und der Commit.

```pycon
(.venv) ~/backupwacht$ pip install -q -e ".[dev]"
(.venv) ~/backupwacht$ ruff format --check
15 files already formatted
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 6 source files
(.venv) ~/backupwacht$ pytest -q --cov
........................................                                 [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      49      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           145      0     34      0   100%
40 passed in 0.01s
(.venv) ~/backupwacht$ backupwacht --version
backupwacht 1.0.0
```

Die Versionsnummer in der Ausgabe stimmt mit dem Protokoll überein. Wir committen die Dokumentation und die neue Version getrennt, weil es zwei verschiedene Absichten sind, und markieren den Stand mit einem *annotierten Tag* (13.5): Er ist ein dauerhafter, benannter Zeiger auf genau diesen Commit, und der Name `v1.0.0` folgt der üblichen Schreibweise.

```pycon
(.venv) ~/backupwacht$ git add README.md CHANGELOG.md
(.venv) ~/backupwacht$ git commit -q -m "Dokumentation: README vollstaendig, Aenderungsprotokoll"
(.venv) ~/backupwacht$ git add src
(.venv) ~/backupwacht$ git commit -q -m "Version 1.0.0"
(.venv) ~/backupwacht$ git tag -a v1.0.0 -m "Version 1.0.0: erste stabile Fassung"
(.venv) ~/backupwacht$ git tag
v1.0.0
(.venv) ~/backupwacht$ git log --oneline --decorate -4
41faea5 (HEAD -> main, tag: v1.0.0) Version 1.0.0
f380b22 Dokumentation: README vollstaendig, Aenderungsprotokoll
1baf16c Refactoring: pruefe_ziel in Suche und Bewertung trennen
445559f Robustheit: Zukunft-Zeitstempel und Lesefehler
(.venv) ~/backupwacht$ git status --short
```

Die Ausgabe zeigt hinter dem neuesten Commit die Dekoration `tag: v1.0.0`, und `git status --short` ist leer: Es gibt keine ungesicherten Änderungen. Von einem Server aus würde man den Tag mit `git push --tags` veröffentlichen; da wir in diesem Kurs keinen entfernten Server verwenden, bleibt er lokal (siehe 13.6).

**Das Paket bauen und ausprobieren.** Ein Programm ist erst dann fertig, wenn man es auf einem *anderen* Rechner installieren kann, ohne den Quelltext. Das prüfen wir mit einem Wheel (13.4): Wir bauen es, richten eine *frische* virtuelle Umgebung ein, installieren das Wheel dorthinein und starten das Kommando von dort. Die frische Umgebung ist der entscheidende Teil. In unserer Entwicklungsumgebung ist vieles installiert, das ein Anwender nicht hat, und Fehler wie vergessene Dateien oder ein fehlender Eintrag in den Paketdaten würden sich dort verstecken.

```pycon
(.venv) ~/backupwacht$ pip install -q --disable-pip-version-check build
(.venv) ~/backupwacht$ python -m build --wheel > /dev/null 2>&1
(.venv) ~/backupwacht$ ls dist
backupwacht-1.0.0-py3-none-any.whl
(.venv) ~/backupwacht$ python -m zipfile -l dist/*.whl | awk '{print $1}'
File
backupwacht/__init__.py
backupwacht/__main__.py
backupwacht/bericht.py
backupwacht/cli.py
backupwacht/config.py
backupwacht/pruefung.py
backupwacht-1.0.0.dist-info/METADATA
backupwacht-1.0.0.dist-info/WHEEL
backupwacht-1.0.0.dist-info/entry_points.txt
backupwacht-1.0.0.dist-info/top_level.txt
backupwacht-1.0.0.dist-info/RECORD
```

Der Dateiname enthält die Version 1.0.0, und im Archiv liegen genau die Module des Pakets sowie die Metadaten. Jetzt die frische Umgebung. Wir rufen das Kommando mit seinem vollen Pfad auf, ohne die Umgebung zu aktivieren, damit sichergestellt ist, dass es sich um die neue und nicht um die Entwicklungsinstallation handelt. Die Ausgabe der Prüfung unterdrücken wir, weil nur der Exit-Code interessiert; die Zeitangaben der Demo-Dateien haben sich seit dem ersten Lauf verschoben.

```pycon
(.venv) ~/backupwacht$ python3 -m venv ~/probe
(.venv) ~/backupwacht$ ~/probe/bin/pip install -q --disable-pip-version-check dist/*.whl
(.venv) ~/backupwacht$ ~/probe/bin/backupwacht --version
backupwacht 1.0.0
(.venv) ~/backupwacht$ ~/probe/bin/backupwacht -c ~/demo/backupwacht.toml > /dev/null; echo "Exit-Code: $?"
Exit-Code: 2
(.venv) ~/backupwacht$ ~/probe/bin/pip list 2>/dev/null | grep -i backupwacht
backupwacht 1.0.0
```

Das installierte Paket läuft, meldet 1.0.0 und liefert für unsere Demo-Konfiguration den erwarteten Exit-Code 2 (ein Ziel ist zu alt). Damit ist belegt, dass die Auslieferung funktioniert. Räumen wir die Probeumgebung auf; sie war nur ein Test.

```pycon
(.venv) ~/backupwacht$ rm -rf ~/probe
(.venv) ~/backupwacht$ ls ~
backupwacht
demo
```

**Der Abschluss: die letzte Kette und die Prüfliste.** Zum Schluss gehen wir das Projekt noch einmal als Ganzes durch, mit einer Prüfliste, die Sie auf jedes Ihrer künftigen Projekte anwenden können. Zuerst die automatische Seite, die vier Prüfungen aus Modul 14 in einer Zeile. Das Zeichen `&&` verbindet die Befehle so, dass der nächste nur läuft, wenn der vorige erfolgreich war; schlägt einer fehl, bricht die Kette ab und der Exit-Code der Zeile ist ungleich null. Genau diese Zeile steht auch in der README unter „Entwicklung“.

```pycon
(.venv) ~/backupwacht$ ruff format --check && ruff check --output-format=concise && mypy && pytest -q --cov; echo "Exit-Code: $?"
15 files already formatted
All checks passed!
Success: no issues found in 6 source files
........................................                                 [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      49      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           145      0     34      0   100%
40 passed in 0.01s
Exit-Code: 0
(.venv) ~/backupwacht$ git status --short
(.venv) ~/backupwacht$ git log --oneline
41faea5 Version 1.0.0
f380b22 Dokumentation: README vollstaendig, Aenderungsprotokoll
1baf16c Refactoring: pruefe_ziel in Suche und Bewertung trennen
445559f Robustheit: Zukunft-Zeitstempel und Lesefehler
7e59ab4 Zeitstempel in der Zukunft und nicht lesbare Verzeichnisse behandeln
cef0774 systemd-Vorlagen fuer den taeglichen Lauf
a18c7ac Bericht (Text, JSON, Gesamtstufe) und Kommandozeile mit Exit-Codes
3c795ff Sicherungsziel pruefen: Alter, Groesse, Rotation
5378f42 Konfiguration aus TOML einlesen und pruefen
1cd4ea7 Gerüst: Paket, Kommandozeile mit --version, erster Test
95754fb Entwurf und README anlegen
```

Die Kette ist grün, der Arbeitsbaum sauber, und der Verlauf erzählt die Geschichte des Projekts: Gerüst, Konfiguration, Prüfung, Bericht, Kommandozeile, Robustheit, Refactoring, Dokumentation, Version. Zu der Prüfliste gehört außerdem, was kein Werkzeug entscheidet. Sind die Ziele und Nicht-Ziele aus dem Entwurf erreicht, und sind die Akzeptanzkriterien jeweils durch einen Test belegt? Steht in der README, wie man das Programm installiert, benutzt und weiterentwickelt, und stimmen die Beispiele? Sind Geheimnisse (Passwörter, Tokens) aus dem Repository und aus dem Verlauf ferngehalten (13.2)? Gibt das Programm bei Fehlern verständliche Meldungen und die richtigen Exit-Codes? Sind die bekannten Einschränkungen aufgeschrieben? Ist die Version gesetzt und markiert? Und schließlich: Läuft das Paket in einer frischen Umgebung?

**Wie es weitergeht.** Das Projekt ist fertig, doch es ist nicht das Ende. Drei Wege liegen nahe. Erstens die Automatisierung der Kette in einem Dienst für *kontinuierliche Integration* (englisch *continuous integration*, CI), der die Tests bei jedem Push automatisch ausführt; GitHub Actions und GitLab CI sind verbreitet. Zweitens die Veröffentlichung auf einem Paketverzeichnis oder die Installation per `pipx` auf den Servern. Drittens neue Funktionen: Meldungen per E-Mail oder Webhook (Modul 12), eine Prüfung des Inhalts oder ein Stundenplan für Wiederholungen. Jede davon beginnt wie dieses Projekt: mit einem kurzen Entwurf, einem Test und einem kleinen Schritt. Die Werkzeuge der Kette haben wir nicht in einer CI-Umgebung, nicht gegen einen echten Git-Server und nicht auf einem Rechner mit `systemd` ausprobiert; sie arbeiten dort aber nach denselben Regeln.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Refactoring ist, gleichzeitig Verhalten zu ändern: Wer „bei der Gelegenheit“ eine Meldung umformuliert, kann nicht mehr sagen, ob ein roter Test vom Umbau oder von der Änderung kommt. Zweitens: Tests anpassen, damit sie nach dem Umbau grün werden, statt zu fragen, warum sie rot wurden; ein Refactoring darf Tests nur dann ändern, wenn sie Interna statt Verhalten prüften. Drittens: nur in der Entwicklungsumgebung testen. Ein Paket, das auf dem eigenen Rechner läuft, kann auf einem anderen scheitern, weil eine Datei im Wheel fehlt oder eine Abhängigkeit nur zufällig installiert war. Viertens: eine README, die beim Schreiben stimmte und nie mehr geprüft wurde. Probieren Sie die Beispiele aus, bevor Sie eine Version markieren. Fünftens: eine Version markieren und danach noch ändern. Ein Tag ist ein Versprechen, dass dieser Stand veröffentlicht ist; wer ihn verschiebt, zerstört das Vertrauen. Der Unterschied, den ein Refactoring macht, sieht im Kleinen so aus:

```python
# vorher: Wiederholung und viele Argumente
if alter > ziel.max_alter_stunden:
    return Ergebnis(ziel.name, "KRITISCH", text, alter)
if len(dateien) < 2:
    return Ergebnis(ziel.name, "WARNUNG", text2, alter)

# nachher: eine kleine Hilfsfunktion
if alter > ziel.max_alter_stunden:
    return ergebnis("KRITISCH", text)
if fund.anzahl < 2:
    return ergebnis("WARNUNG", text2)
```

Beide Fassungen tun dasselbe; die zweite ist kürzer, enthält weniger Wiederholung und lässt sich leichter ändern, etwa wenn das Ergebnis ein weiteres Feld bekommt. Das ist der Zweck jedes Refactorings.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Befehle, Tests und Ausgaben stammen aus echten Läufen mit Python 3.13.18, Git 2.43.0, pytest 9.2.0, ruff 0.17.4, mypy 2.4.1 und setuptools 84.2.1; die Wheel wurde tatsächlich gebaut und in einer frischen virtuellen Umgebung installiert. Die Form des Änderungsprotokolls und die semantische Versionierung sind verbreitete Konventionen (Projekte *Keep a Changelog* und *Semantic Versioning*), die hier nicht abgerufen wurden und als weiterführende Quellen im Quellenverzeichnis stehen. Zur Lizenzwahl gilt: keine Rechtsberatung; Lizenztexte sind rechtlich bindend, lassen Sie sich im Zweifel beraten. Nicht ausprobiert wurden CI-Dienste, die Veröffentlichung auf einem Paketverzeichnis und der Betrieb mit `systemd`.

> **Merke**
>
> Refactoring verändert die Struktur, nicht das Verhalten, und braucht Tests als Sicherheitsnetz; trennen Sie es immer von neuen Funktionen. Eine fertige README beantwortet Zweck, Installation, Benutzung, Ausgabe, Entwicklung und Einschränkungen. Die Version steht an einer Stelle, das Änderungsprotokoll beschreibt jede Version, und ein annotierter Tag markiert den veröffentlichten Stand. Prüfen Sie die Auslieferung in einer frischen virtuellen Umgebung, nicht in der Entwicklungsumgebung. Eine Prüfliste am Ende verhindert, dass etwas Wichtiges fehlt.

> **Praxisbeispiel**
>
> Im Beruf erkennt man ein gutes Werkzeug daran, dass ein neuer Kollege es ohne Rückfrage installieren, benutzen und weiterentwickeln kann. Genau das leisten README, Änderungsprotokoll, Tests, Versionsmarken und ein sauberer Verlauf. Wer diese Gewohnheit früh übt, spart sich die Frage „Wie ging das noch mal?“ in einem halben Jahr, wenn der Server ausfällt und das Skript, das man vergessen hat, wieder gebraucht wird.

## Übungsaufgaben zu Modul 15

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Urteilsvermögen, die Aufgaben 5 bis 8 verlangen eigenen Code und eigene Projektarbeit. Die Aufgaben 5 bis 7 bauen auf dem Beispielprojekt `backupwacht` aus diesem Modul auf; arbeiten Sie dort in einem eigenen Zweig, mit aktivierter virtueller Umgebung und der Qualitätskette aus 15.5. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Quiz zur Planung: Was gehört in einen kurzen Projektentwurf, und was ist der Unterschied zwischen einem Ziel, einem Nicht-Ziel und einem Akzeptanzkriterium? Was versteht man unter einem *Walking Skeleton*, und warum beginnt man damit? Warum wählt man für die Konfiguration TOML und nicht JSON? Warum trennt man Konfiguration, Prüfung, Bericht und Kommandozeile in eigene Module, und in welche Richtung dürfen die Abhängigkeiten zeigen?

**Aufgabe 2.** Quiz zur Umsetzung: Wozu dient `dataclass(frozen=True)`? Warum reicht `backupwacht` die aktuelle Uhrzeit als Parameter `jetzt` herein, statt sie in der Prüfung selbst zu lesen? Was bedeutet *frühe Rückkehr* (early return)? Warum gehören Fehlermeldungen auf die Fehlerausgabe, und wofür stehen die Exit-Codes 0, 1, 2 und 3 im Projekt? Warum ist ein Wahrheitswert in der Konfiguration kein gültiger Zahlenwert, obwohl `True` in Python eine ganze Zahl ist?

**Aufgabe 3.** Quiz zur Qualität: Was misst die Zweigabdeckung zusätzlich zur Zeilenabdeckung? Warum beweist eine Abdeckung von hundert Prozent nicht, dass ein Programm richtig ist? Wie läuft eine Fehlerbehebung nach dem Test-zuerst-Prinzip ab, und warum arbeitet man dafür in einem eigenen Zweig? Was ist ein Refactoring, was darf dabei nicht geschehen, und was ist die Voraussetzung dafür? Warum prüft man ein Paket in einer *frischen* virtuellen Umgebung?

**Aufgabe 4.** Führen Sie eine Durchsicht (Code Review) der folgenden Funktion durch. Finden Sie mindestens sechs Probleme, ordnen Sie jedes einer Gruppe der Prüfliste aus 15.4 zu (Eingaben, Umgebung, Grenzen, Fehlermeldungen, Lesbarkeit, Testbarkeit) und beschreiben Sie, wie Sie es beheben würden.

```python
import os, json


def pruefe(pfad, max_alter=24):
    dateien = os.listdir(pfad)
    neueste = None
    for d in dateien:
        if d.endswith(".tar"):
            if neueste is None or os.path.getmtime(pfad + "/" + d) > os.path.getmtime(pfad + "/" + neueste):
                neueste = d
    alter = (time.time() - os.path.getmtime(pfad + "/" + neueste)) / 3600
    if alter > max_alter:
        print("FEHLER: zu alt")
        return 1
    print(json.dumps({"alter": alter}))
```

**Aufgabe 5.** **Code:** Erweitern Sie `backupwacht` nach dem Test-zuerst-Prinzip um eine Funktion `nur_ab(ergebnisse, mindest)` in einem neuen Modul `auswahl.py`. Sie gibt aus einer Liste von Ergebnissen nur jene zurück, deren Stufe mindestens so schlecht ist wie die angegebene Mindeststufe (zum Beispiel `"WARNUNG"` liefert Warnungen und kritische Ergebnisse, aber keine OK-Ergebnisse); die Reihenfolge bleibt erhalten. Schreiben Sie zuerst die parametrisierten Tests (alle drei Stufen, leere Liste, Reihenfolge), sehen Sie sie scheitern, und setzen Sie dann die Funktion um, sodass `mypy --strict` keine Meldung ausgibt. Nutzen Sie die Reihenfolge der Stufen aus `bericht.py`, statt sie zu wiederholen.

**Aufgabe 6.** **Code:** Die Mindestzahl der Sicherungsdateien (Rotation) ist im Projekt fest auf zwei eingestellt. Machen Sie sie konfigurierbar: Ein neuer, optionaler Schlüssel `min_anzahl` im Abschnitt `[[pruefung]]` (Standard 2, nur ganze Zahlen ab 1, keine Wahrheitswerte) soll in `config.py` geprüft und in `pruefung.py` verwendet werden. Schreiben Sie zuerst die Tests für die Konfiguration (Standardwert, gültiger Wert, ungültige Werte) und für die Prüfung (zum Beispiel drei geforderte Dateien, aber nur zwei vorhanden ergibt eine Warnung; eine geforderte Datei und eine vorhandene ergibt OK). Die bestehenden Tests müssen ohne Änderung grün bleiben. Zeigen Sie Ihre Änderung mit `git diff`.

**Aufgabe 7.** **Code:** Vor dem Einsatz auf einem Server möchte man die Konfiguration prüfen, ohne dass Dateien untersucht werden. Fügen Sie der Kommandozeile die Option `--pruefe-konfiguration` hinzu: Sie lädt die Konfiguration, gibt bei Erfolg `Konfiguration in Ordnung: N Ziele` aus und endet mit dem Exit-Code 0, ohne die Verzeichnisse der Ziele zu prüfen. Bei einem Konfigurationsfehler gilt wie bisher Exit-Code 3. Schreiben Sie zuerst die Tests (gültige Konfiguration auch dann, wenn die Verzeichnisse nicht existieren; fehlerhafte Konfiguration; der Test darf kein Verzeichnis anlegen), dann die Umsetzung, und lassen Sie die gesamte Kette mit Abdeckung laufen.

**Aufgabe 8.** **Projekt:** Führen Sie den vollständigen Ablauf dieses Moduls mit einem eigenen kleinen Werkzeug durch, das Sie sich selbst ausdenken (Vorschläge: ein Prüfer für Logdateien, der meldet, wenn eine Logdatei zu lange nicht beschrieben wurde; ein Werkzeug, das den freien Speicherplatz mehrerer Verzeichnisse prüft; ein Zähler für Fehlerzeilen in einer Logdatei). Verlangt werden: ein schriftlicher Entwurf mit Zielen, Nicht-Zielen und Akzeptanzkriterien in `docs/entwurf.md`; ein Git-Repository mit `src`-Layout und `pyproject.toml`; ein Walking Skeleton als erster Commit; die Qualitätskette (ruff, mypy, pytest mit Abdeckung) grün; mindestens drei weitere Commits mit sinnvollen Nachrichten; eine README mit Installation, Benutzung und Einschränkungen; eine Version mit annotiertem Tag; ein gebautes und in einer frischen virtuellen Umgebung ausprobiertes Wheel. Beschreiben Sie am Ende in fünf Sätzen, was Sie bei der Durchsicht Ihres eigenen Codes noch verbessert haben.

## Lösungen zu Modul 15

**Aufgabe 1.** Ein kurzer Entwurf enthält das Problem in ein bis zwei Sätzen, die Ziele, die Nicht-Ziele, die Akzeptanzkriterien, die Aufteilung in Teile (mit je einer Aufgabe), das Format der Konfiguration und die erkannten Risiken. Ein *Ziel* sagt, was das Werkzeug leisten soll; ein *Nicht-Ziel* sagt ausdrücklich, was es nicht tun soll, und schützt vor dem schleichenden Wachsen des Projekts; ein *Akzeptanzkriterium* ist eine überprüfbare Aussage, an der man erkennt, dass ein Ziel erreicht ist (zum Beispiel „eine Datei, die älter ist als erlaubt, ergibt ‚kritisch‘“), und lässt sich fast eins zu eins in einen Test übersetzen. Das *Walking Skeleton* ist die dünnste durchgehende Fassung des Programms, die von der Eingabe bis zur Ausgabe läuft, mit Paket, Kommando, Test und Qualitätskette, aber fast ohne Funktion. Man beginnt damit, weil sich dann Fehler im Aufbau (Pakete, Installation, Importe) früh zeigen und jeder weitere Schritt auf einem lauffähigen Stand aufsetzt. TOML ist ab Python 3.11 mit `tomllib` in der Standardbibliothek, erlaubt Kommentare (JSON nicht), kennt Listen von Tabellen (`[[pruefung]]`) und ist für Menschen gut lesbar; JSON wäre für Maschinen bequem, aber als handgepflegte Konfiguration unfreundlich. Die Trennung in Module hat den Zweck, dass jeder Teil allein getestet und ausgetauscht werden kann. Die Abhängigkeiten zeigen in eine Richtung: Die Kommandozeile kennt Bericht, Prüfung und Konfiguration; die Prüfung kennt die Konfigurationstypen; Konfiguration und Bericht kennen die Kommandozeile nicht. Zyklen (A importiert B, B importiert A) vermeidet man.

**Aufgabe 2.** `dataclass(frozen=True)` erzeugt eine Klasse, die vor allem Daten hält, und macht die Objekte *unveränderlich*: Nach dem Erzeugen lässt sich kein Feld überschreiben, was eine ganze Klasse von Fehlern (unbeabsichtigte Änderungen, geteilter Zustand) ausschließt; zusätzlich sind solche Objekte hashbar. Die Uhrzeit wird als Parameter hereingereicht, damit die Funktion *testbar* bleibt: Ein Test kann `jetzt` auf einen festen Wert setzen, und das Alter jeder Datei ist exakt berechenbar, ohne auf die Uhr zu warten oder von ihr abhängig zu sein. Die frühe Rückkehr behandelt jeden Fehler- oder Sonderfall sofort mit `return` und hält den Rest der Funktion flach, weil dieser davon ausgehen darf, dass alles in Ordnung ist; das verhindert tief verschachtelte `if`-`else`-Bäume. Fehlermeldungen gehören auf `stderr`, weil die normale Ausgabe oft weiterverarbeitet wird (zum Beispiel als JSON); Fehltext dazwischen würde sie unbrauchbar machen. Die Exit-Codes: 0 alle Ziele in Ordnung, 1 mindestens eine Warnung, 2 mindestens ein kritisches Ziel, 3 Konfigurationsfehler. Ein Wahrheitswert ist in Python eine ganze Zahl (`isinstance(True, int)` ist wahr, `True` verhält sich wie 1), sodass `max_alter_stunden = true` sonst als „1 Stunde“ durchginge; da das fast sicher ein Versehen ist, schließt die Prüfung Wahrheitswerte ausdrücklich aus.

**Aufgabe 3.** Die Zweigabdeckung prüft zusätzlich, ob bei jeder Verzweigung (`if`, Schleife, Ausnahmebehandlung) *beide* Wege mindestens einmal genommen wurden; die Zeilenabdeckung ist schon zufrieden, wenn eine Zeile einmal lief. Hundert Prozent beweisen nichts, weil sie nur zeigen, dass Code *ausgeführt* wurde, nicht, dass sein Ergebnis *geprüft* wurde: Ein Test ohne `assert` erhöht die Abdeckung ebenso, und Fälle, an die man nicht gedacht hat (Eingaben, die nie getestet wurden), tauchen in der Zahl gar nicht auf. Eine Fehlerbehebung nach dem Test-zuerst-Prinzip läuft so ab: Zuerst schreibt man einen Test, der den Fehler zeigt, und sieht ihn scheitern (das beweist, dass man den Fehler versteht und der Test ihn fängt); dann behebt man den Fehler, bis der Test besteht; dann läuft die gesamte Kette. Der Zweig sorgt dafür, dass `main` lauffähig bleibt, macht die Änderung als Einheit sichtbar (`git diff main`) und lässt sich notfalls einfach verwerfen. Ein Refactoring ändert die innere Struktur des Codes, aber *nicht* sein äußeres Verhalten; dabei dürfen keine neuen Funktionen und keine Verhaltensänderungen einfließen, und Tests dürfen nur angepasst werden, wenn sie Interna statt Verhalten prüften. Die Voraussetzung ist ein Satz verlässlicher Tests als Sicherheitsnetz. Die frische virtuelle Umgebung enthält nur, was die Wheel mitbringt; so zeigen sich fehlende Dateien, fehlende Abhängigkeiten oder falsche Metadaten, die in der Entwicklungsumgebung durch zufällig Installiertes verdeckt würden.

**Aufgabe 4.** Die Durchsicht findet unter anderem diese Probleme. *Eingaben:* Gibt es keine Datei mit der Endung `.tar`, bleibt `neueste` gleich `None`, und die Berechnung des Alters bricht mit einem `TypeError` ab; der Fall „keine passende Datei“ ist nicht behandelt. Das Muster `.tar` ist fest eingebaut, und Verzeichnisse mit dieser Endung würden als Datei gezählt. *Umgebung:* `os.listdir` liefert bei fehlendem oder nicht lesbarem Verzeichnis einen Traceback; zwischen Auflisten und `getmtime` kann eine Datei verschwinden. *Grenzen:* Ein Zeitstempel in der Zukunft ergibt ein negatives Alter, das unbemerkt durchgeht; eine Mindestgröße wird nicht geprüft, sodass eine leere Datei als Sicherung gilt. *Fehlermeldungen:* „FEHLER: zu alt“ nennt weder Ziel noch Alter noch Grenze und steht auf der normalen Ausgabe statt auf der Fehlerausgabe. Die Funktion liefert außerdem uneinheitlich 1 oder (am Ende) gar nichts statt 0. *Lesbarkeit:* Der Import `import os, json` in einer Zeile verstößt gegen die Regel E401; das Zusammensetzen von Pfaden mit `pfad + "/" + d` ist fehleranfällig (besser `pathlib` oder `os.path.join`); die Zeile mit den zwei `getmtime`-Aufrufen ist zu lang und ruft die Dateizeit doppelt ab; `max_alter` nennt die Einheit nicht (besser `max_alter_stunden`); es gibt weder Typangaben noch Docstring. Ein Import fehlt ganz: `time` wird benutzt, aber nicht importiert, was zu einem `NameError` führt. *Testbarkeit:* Die Funktion liest die Uhr selbst und vermischt Prüfen, Rechnen und Ausgeben, sodass sie sich nur schwer testen lässt. Die Behebung folgt dem Muster des Projekts: Pfade mit `pathlib`, Uhrzeit als Parameter, Ergebnis als Objekt statt Ausgabe, frühe Rückkehr für Fehlerfälle und ein Test pro Fall.

**Aufgabe 5.** Wir arbeiten in einem eigenen Zweig, damit die Version 1.0.0 auf `main` unberührt bleibt. Die Tests kommen zuerst. Sie prüfen jede Mindeststufe mit `parametrize`, die leere Liste und die erhaltene Reihenfolge.

```pycon
(.venv) ~/backupwacht$ cd ~/backupwacht
(.venv) ~/backupwacht$ git switch -q -c uebung
(.venv) ~/backupwacht$ git branch
  main
* uebung
```

**Datei** `tests/test_auswahl.py`**:**

```python
import pytest

from backupwacht.auswahl import nur_ab
from backupwacht.pruefung import Ergebnis

OK = Ergebnis("a", "OK", "gut")
WARN = Ergebnis("b", "WARNUNG", "hmm")
KRIT = Ergebnis("c", "KRITISCH", "schlecht")
ALLE = [OK, WARN, KRIT, WARN]


@pytest.mark.parametrize(
    "mindest, erwartet",
    [
        ("OK", ALLE),
        ("WARNUNG", [WARN, KRIT, WARN]),
        ("KRITISCH", [KRIT]),
    ],
)
def test_nur_ergebnisse_ab_der_mindeststufe(mindest, erwartet):
    assert nur_ab(ALLE, mindest) == erwartet


def test_leere_liste_bleibt_leer():
    assert nur_ab([], "WARNUNG") == []


def test_reihenfolge_bleibt_erhalten():
    assert nur_ab([KRIT, OK, WARN], "WARNUNG") == [KRIT, WARN]
```

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -4
=========================== short test summary info ============================
ERROR tests/test_auswahl.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.01s
```

Der Test scheitert beim Import, weil das Modul fehlt; das ist der erwartete rote Zustand. Die Umsetzung vergleicht die Rangzahlen der Stufen aus `bericht.py`. Die Reihenfolge wird an genau einer Stelle festgelegt; wer sie hier wiederholte, müsste sie bei jeder Änderung an zwei Stellen pflegen.

**Datei** `src/backupwacht/auswahl.py`**:**

```python
"""Auswahl von Ergebnissen nach Stufe."""
from collections.abc import Sequence

from .bericht import REIHENFOLGE
from .pruefung import Ergebnis, Stufe


def nur_ab(ergebnisse: Sequence[Ergebnis], mindest: Stufe) -> list[Ergebnis]:
    """Gibt die Ergebnisse zurueck, deren Stufe mindestens so schlecht ist wie mindest."""
    return [e for e in ergebnisse if REIHENFOLGE[e.stufe] >= REIHENFOLGE[mindest]]
```

```pycon
(.venv) ~/backupwacht$ ruff format src tests
1 file reformatted, 13 files left unchanged
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 7 source files
(.venv) ~/backupwacht$ pytest -q --cov
.............................................                            [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/auswahl.py        5      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        43      0     12      0   100%
src/backupwacht/pruefung.py      49      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           150      0     34      0   100%
45 passed in 0.01s
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "Auswahl von Ergebnissen nach Mindeststufe"
```

**Aufgabe 6.** Die Tests für die Konfiguration und die Prüfung kommen zuerst. Die ungültigen Werte (null, negativ, Kommazahl, Wahrheitswert, Text) prüfen wir mit `parametrize`; die Prüfung selbst testen wir mit Dateien von festem Alter, wie in 15.2.

**Datei** `tests/test_min_anzahl.py`**:**

```python
import os
from pathlib import Path

import pytest

from backupwacht.config import KonfigurationsFehler, Pruefung, lade_konfiguration
from backupwacht.pruefung import pruefe_ziel

JETZT = 1_800_000_000.0


def konfig(tmp_path: Path, zusatz: str = "") -> Path:
    pfad = tmp_path / "k.toml"
    pfad.write_text(f'[[pruefung]]\nname = "a"\nverzeichnis = "/x"\n{zusatz}', encoding="utf-8")
    return pfad


def lege(ordner: Path, name: str) -> None:
    pfad = ordner / name
    pfad.write_bytes(b"x" * 10)
    zeit = JETZT - 3600
    os.utime(pfad, (zeit, zeit))


def test_standard_ist_zwei(tmp_path):
    assert lade_konfiguration(konfig(tmp_path)).pruefungen[0].min_anzahl == 2


def test_gueltiger_wert_wird_gelesen(tmp_path):
    assert lade_konfiguration(konfig(tmp_path, "min_anzahl = 3\n")).pruefungen[0].min_anzahl == 3


@pytest.mark.parametrize("wert", ["0", "-1", "1.5", "true", '"zwei"'])
def test_ungueltige_werte(tmp_path, wert):
    with pytest.raises(KonfigurationsFehler, match="min_anzahl"):
        lade_konfiguration(konfig(tmp_path, f"min_anzahl = {wert}\n"))


def test_zu_wenige_dateien_ergeben_eine_warnung(tmp_path):
    lege(tmp_path, "a")
    lege(tmp_path, "b")
    ergebnis = pruefe_ziel(Pruefung("t", tmp_path, min_anzahl=3), JETZT)
    assert ergebnis.stufe == "WARNUNG"
    assert "2 von 3" in ergebnis.meldung


def test_eine_datei_genuegt_wenn_eine_verlangt_ist(tmp_path):
    lege(tmp_path, "a")
    assert pruefe_ziel(Pruefung("t", tmp_path, min_anzahl=1), JETZT).stufe == "OK"
```

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -4
FAILED tests/test_min_anzahl.py::test_gueltiger_wert_wird_gelesen - backupwac...
FAILED tests/test_min_anzahl.py::test_zu_wenige_dateien_ergeben_eine_warnung
FAILED tests/test_min_anzahl.py::test_eine_datei_genuegt_wenn_eine_verlangt_ist
4 failed, 50 passed in 0.01s
```

Die Tests scheitern, weil das Feld `min_anzahl` noch nicht existiert. Die Änderung berührt vier Stellen in `config.py` (erlaubte Schlüssel, Feld mit Standardwert, Prüfung, Übergabe an das Objekt) und eine in `pruefung.py` (die feste Zahl 2 wird durch den Wert aus der Konfiguration ersetzt; dabei nennt die Meldung nun auch die Soll-Zahl). Wir nehmen die Änderung mit einem kleinen Hilfsskript vor, das jede Textstelle genau einmal ersetzt und sonst abbricht; in Ihrem Editor machen Sie dieselben Änderungen von Hand. Entscheidend ist das Ergebnis, das wir mit `git diff` ansehen.

```pycon
(.venv) ~/backupwacht$ git diff src
diff --git a/src/backupwacht/config.py b/src/backupwacht/config.py
index 6197e19..aaa9bb8 100644
--- a/src/backupwacht/config.py
+++ b/src/backupwacht/config.py
@@ -5,7 +5,14 @@ from dataclasses import dataclass
 from pathlib import Path
 from typing import Any
 
-ERLAUBTE_SCHLUESSEL = {"name", "verzeichnis", "muster", "max_alter_stunden", "min_groesse_byte"}
+ERLAUBTE_SCHLUESSEL = {
+    "name",
+    "verzeichnis",
+    "muster",
+    "max_alter_stunden",
+    "min_groesse_byte",
+    "min_anzahl",
+}
 
 
 class KonfigurationsFehler(ValueError):
@@ -21,6 +28,7 @@ class Pruefung:
     muster: str = "*"
     max_alter_stunden: float = 26.0
     min_groesse_byte: int = 1
+    min_anzahl: int = 2
 
 
 @dataclass(frozen=True)
@@ -57,10 +65,14 @@ def _baue_pruefung(nummer: int, eintrag: dict[str, Any]) -> Pruefung:
     min_groesse = eintrag.get("min_groesse_byte", 1)
     if isinstance(min_groesse, bool) or not isinstance(min_groesse, int) or min_groesse < 0:
         raise KonfigurationsFehler(f"{ort}: min_groesse_byte muss eine ganze Zahl ab 0 sein")
+    min_anzahl = eintrag.get("min_anzahl", 2)
+    if isinstance(min_anzahl, bool) or not isinstance(min_anzahl, int) or min_anzahl < 1:
+        raise KonfigurationsFehler(f"{ort}: min_anzahl muss eine ganze Zahl ab 1 sein")
     return Pruefung(
         name=str(eintrag["name"]),
         verzeichnis=Path(str(eintrag["verzeichnis"])),
         muster=str(eintrag.get("muster", "*")),
         max_alter_stunden=float(max_alter),
         min_groesse_byte=min_groesse,
+        min_anzahl=min_anzahl,
     )
diff --git a/src/backupwacht/pruefung.py b/src/backupwacht/pruefung.py
index 73397cc..d71d92f 100644
--- a/src/backupwacht/pruefung.py
+++ b/src/backupwacht/pruefung.py
@@ -53,8 +53,9 @@ def _bewerte(ziel: Pruefung, fund: _Fund, jetzt: float) -> Ergebnis:
     if alter > ziel.max_alter_stunden:
         erlaubt = f"{ziel.max_alter_stunden:g}"
         return ergebnis("KRITISCH", f"{fund.name} ist {alter:.1f} Stunden alt (erlaubt: {erlaubt})")
-    if fund.anzahl < 2:
-        return ergebnis("WARNUNG", f"nur eine Sicherung vorhanden: {fund.name}")
+    if fund.anzahl < ziel.min_anzahl:
+        text = f"nur {fund.anzahl} von {ziel.min_anzahl} Sicherungen vorhanden: {fund.name}"
+        return ergebnis("WARNUNG", text)
     return ergebnis("OK", f"{fund.name}, {alter:.1f} Stunden alt")
```

Die Änderung ist klein und übersichtlich. Die Standardwerte stellen sicher, dass bestehende Konfigurationsdateien unverändert weiterlaufen; neue Fähigkeiten sollten bestehende Anwender nicht zu Änderungen zwingen. Nun die gesamte Kette: Die bisherigen Tests laufen ohne Änderung durch, was zeigt, dass wir nichts gebrochen haben.

```pycon
(.venv) ~/backupwacht$ ruff format --check
18 files already formatted
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 7 source files
(.venv) ~/backupwacht$ pytest -q --cov
......................................................                   [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/auswahl.py        5      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           23      0      0      0   100%
src/backupwacht/config.py        47      0     14      0   100%
src/backupwacht/pruefung.py      50      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           155      0     36      0   100%
54 passed in 0.01s
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "min_anzahl: Mindestzahl der Sicherungen konfigurierbar"
```

**Aufgabe 7.** Die neue Option gehört in die Kommandozeile. Wieder beginnen wir mit den Tests. Der erste prüft, dass eine gültige Konfiguration auch dann akzeptiert wird, wenn die Verzeichnisse nicht existieren, und dass dabei nichts im Dateisystem entsteht; der zweite prüft den Fehlerfall.

**Datei** `tests/test_pruefe_konfiguration.py`**:**

```python
from backupwacht.cli import main


def test_gueltige_konfiguration_ohne_vorhandene_verzeichnisse(tmp_path, capsys):
    konfig = tmp_path / "k.toml"
    konfig.write_text(
        '[[pruefung]]\nname = "a"\nverzeichnis = "/gibt/es/nicht"\n\n'
        '[[pruefung]]\nname = "b"\nverzeichnis = "/auch/nicht"\n',
        encoding="utf-8",
    )
    assert main(["-c", str(konfig), "--pruefe-konfiguration"]) == 0
    assert capsys.readouterr().out.strip() == "Konfiguration in Ordnung: 2 Ziele"
    assert list(tmp_path.iterdir()) == [konfig]


def test_fehlerhafte_konfiguration_ergibt_exit_3(tmp_path, capsys):
    konfig = tmp_path / "k.toml"
    konfig.write_text("# leer\n", encoding="utf-8")
    assert main(["-c", str(konfig), "--pruefe-konfiguration"]) == 3
    assert "Konfigurationsfehler" in capsys.readouterr().err
```

```pycon
(.venv) ~/backupwacht$ pytest -q 2>&1 | tail -4
=========================== short test summary info ============================
FAILED tests/test_pruefe_konfiguration.py::test_gueltige_konfiguration_ohne_vorhandene_verzeichnisse
FAILED tests/test_pruefe_konfiguration.py::test_fehlerhafte_konfiguration_ergibt_exit_3
2 failed, 54 passed in 0.01s
```

Beide Tests scheitern, weil `argparse` die unbekannte Option mit Exit-Code 2 ablehnt, noch bevor die Konfiguration geladen wird. Jetzt die Umsetzung in `cli.py`: Die Option ist ein Schalter (`store_true`), und in `main` beenden wir den Ablauf direkt nach dem erfolgreichen Laden der Konfiguration.

```pycon
(.venv) ~/backupwacht$ git diff src/backupwacht/cli.py
diff --git a/src/backupwacht/cli.py b/src/backupwacht/cli.py
index 1e3ad0e..5e5ccdb 100644
--- a/src/backupwacht/cli.py
+++ b/src/backupwacht/cli.py
@@ -26,6 +26,11 @@ def baue_parser() -> argparse.ArgumentParser:
         default="text",
         help="Ausgabeformat (Standard: %(default)s)",
     )
+    parser.add_argument(
+        "--pruefe-konfiguration",
+        action="store_true",
+        help="nur die Konfiguration pruefen, keine Verzeichnisse untersuchen",
+    )
     return parser
 
 
@@ -36,6 +41,9 @@ def main(argv: list[str] | None = None) -> int:
     except KonfigurationsFehler as fehler:
         print(f"Konfigurationsfehler: {fehler}", file=sys.stderr)
         return EXIT_KONFIGURATIONSFEHLER
+    if args.pruefe_konfiguration:
+        print(f"Konfiguration in Ordnung: {len(konfig.pruefungen)} Ziele")
+        return 0
     ergebnisse = [pruefe_ziel(ziel) for ziel in konfig.pruefungen]
     print(als_json(ergebnisse) if args.format == "json" else als_text(ergebnisse))
     return exit_code(gesamtstufe(ergebnisse))
(.venv) ~/backupwacht$ ruff format --check
19 files already formatted
(.venv) ~/backupwacht$ ruff check --output-format=concise
All checks passed!
(.venv) ~/backupwacht$ mypy
Success: no issues found in 7 source files
(.venv) ~/backupwacht$ pytest -q --cov
........................................................                 [100%]
================================ tests coverage ================================
_______________ coverage: platform linux, python 3.13.18-final-0 _______________

Name                          Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------------
src/backupwacht/__init__.py       1      0      0      0   100%
src/backupwacht/__main__.py       3      0      0      0   100%
src/backupwacht/auswahl.py        5      0      0      0   100%
src/backupwacht/bericht.py       26      0      8      0   100%
src/backupwacht/cli.py           27      0      2      0   100%
src/backupwacht/config.py        47      0     14      0   100%
src/backupwacht/pruefung.py      50      0     14      0   100%
-------------------------------------------------------------------------
TOTAL                           159      0     38      0   100%
56 passed in 0.01s
```

Alle Tests bestehen, und die Abdeckung bleibt bei hundert Prozent, weil beide Wege der neuen Verzweigung getestet sind. Beachten Sie, dass die Option auch in der Hilfe auftaucht (`backupwacht --help`). Wir sichern die Lösung, kehren zu `main` zurück und lassen den Übungszweig unberührt liegen.

```pycon
(.venv) ~/backupwacht$ git add src tests
(.venv) ~/backupwacht$ git commit -q -m "Option --pruefe-konfiguration"
(.venv) ~/backupwacht$ git switch -q main
(.venv) ~/backupwacht$ git branch
* main
  uebung
(.venv) ~/backupwacht$ git log --oneline --decorate -3
41faea5 (HEAD -> main, tag: v1.0.0) Version 1.0.0
f380b22 Dokumentation: README vollstaendig, Aenderungsprotokoll
1baf16c Refactoring: pruefe_ziel in Suche und Bewertung trennen
```

Der Zweig `uebung` enthält die drei Lösungen; `main` mit der markierten Version 1.0.0 ist unverändert. Würde man die Änderungen veröffentlichen wollen, folgten Zusammenführung, Eintrag im Änderungsprotokoll und eine neue Version (nach SemVer 1.1.0, denn es kamen abwärtskompatible Funktionen hinzu).

**Aufgabe 8.** Hier gibt es keine einzelne richtige Lösung, sondern eine Reihenfolge, nach der Sie Ihre Arbeit beurteilen können. Beginnen Sie mit dem Entwurf: ein Problem in einem Satz, drei bis fünf Ziele, mindestens zwei Nicht-Ziele und Akzeptanzkriterien, die sich als Tests schreiben lassen. Legen Sie dann das Gerüst an und committen Sie es, bevor irgendeine Funktion existiert. Entwickeln Sie jede Funktion Test zuerst, committen Sie nach jedem funktionierenden Schritt, und lassen Sie vor jedem Commit die Kette laufen. Schreiben Sie die README erst, wenn das Programm läuft, und probieren Sie jedes Beispiel daraus aus. Die Befehlsfolge entspricht der aus 15.1 bis 15.5; ein möglicher Ablauf für ein Werkzeug namens `logwacht` sieht so aus (er wurde hier nicht ausgeführt, die einzelnen Befehle aber an `backupwacht`):

```text
mkdir -p logwacht/src/logwacht logwacht/tests logwacht/docs
cd logwacht && git init -b main
# docs/entwurf.md, pyproject.toml, .gitignore, __init__.py, cli.py, test_cli.py
python3 -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"
ruff format && ruff check && mypy && pytest --cov
git add . && git commit -m "Geruest: Paket, Kommandozeile, erster Test"
# ... pro Funktion: Test schreiben, scheitern sehen, umsetzen, Kette, Commit
git tag -a v0.1.0 -m "Erste Fassung"
python -m build --wheel
python3 -m venv /tmp/probe && /tmp/probe/bin/pip install dist/*.whl
/tmp/probe/bin/logwacht --version
```

Prüfen Sie Ihr Ergebnis anhand der Liste aus 15.5: Ziele und Akzeptanzkriterien erreicht und durch Tests belegt, Kette grün, Abdeckung ohne unerklärte Lücken, README stimmig, keine Geheimnisse im Repository, Version gesetzt und markiert, Paket in frischer Umgebung lauffähig. Die fünf Sätze zur Durchsicht sollten konkrete Befunde nennen (zum Beispiel „Dateien, die zwischen Auflisten und Lesen verschwinden, wurden nicht behandelt; jetzt fängt ein `try` den `OSError` ab“) und nicht bloß „ich habe den Code verbessert“. Wer das ehrlich tut, hat den eigentlichen Zweck dieses Moduls erreicht: einen Ablauf, der auch bei größeren Projekten trägt.
