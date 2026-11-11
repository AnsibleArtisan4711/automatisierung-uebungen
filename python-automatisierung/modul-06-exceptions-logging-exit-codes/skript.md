# Modul 06 - Exceptions, logging, Exit-Codes

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (6.1 bis 6.5), danach folgen „Übungsaufgaben zu Modul 06“ (8 Aufgaben) und „Lösungen zu Modul 06“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 01.12.2026 bis Mi 09.12.2026; Fr 04.12. und Di 08.12.2026 sind frei)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; ein Skript ohne Fehlerbehandlung und Protokoll ist im Betrieb nicht verwendbar)

**Meilenstein:** Backup-Skript

**Lernziele.** Nach diesem Modul können Sie Fehler gezielt abfangen, eigene Fehlerarten definieren und Fehler sinnvoll weiterreichen, statt sie zu verschlucken. Sie können Ihrem Skript Rückgabecodes geben, die ein aufrufendes Programm auswerten kann, und mit dem Modul `logging` ein ordentliches Protokoll schreiben statt `print`-Ausgaben. Am Ende steht ein Sicherungsskript, das ein Verzeichnis in ein Archiv packt, alte Sicherungen aufräumt, jeden Schritt protokolliert und mit einem aussagekräftigen Rückgabecode endet.

| **Nr.** | **Lerneinheit**                        | **Datum**     | **Dauer** | **Prio** |
|:--------|:---------------------------------------|:--------------|:----------|:---------|
| 6.1     | Exceptions (try/except/finally)        | Di 01.12.2026 | 2 h       | 1        |
| 6.2     | Fehlerbehandlung & Exit-Codes          | Mi 02.12.2026 | 2 h       | 1        |
| 6.3     | logging Grundlagen                     | Do 03.12.2026 | 2 h       | 1        |
| 6.4     | Backup-Skript Teil 1 (tar/rsync)       | Mo 07.12.2026 | 2 h       | 1        |
| 6.5     | Backup-Skript Teil 2 (Rotation) + Quiz | Mi 09.12.2026 | 2 h       | 1        |
|         | **Summe**                              |               | **10 h**  |          |

> **Merke**
>
> Alle Dateibeispiele dieses Moduls arbeiten in einem temporären Verzeichnis (`tempfile.TemporaryDirectory()`), das nach dem Lauf verschwindet. Sicherungsskripte sind Programme, die Dateien lesen, schreiben und löschen; probieren Sie eigene Varianten erst mit einem Probelauf und mit Testdaten aus, bevor Sie sie auf echte Daten loslassen.

## 6.1 Exceptions (try/except/finally)

In Modul 3 haben Sie Tracebacks gelesen: die Meldungen, die Python ausgibt, wenn ein Programm abstürzt. Hinter jedem Traceback steckt eine *Ausnahme* (englisch *exception*), ein Objekt, das beschreibt, was schiefgegangen ist. Python löst eine Ausnahme aus, sobald es eine Anweisung nicht ausführen kann: eine Datei fehlt (`FileNotFoundError`), eine Zeichenkette lässt sich nicht in eine Zahl umwandeln (`ValueError`), ein Schlüssel ist nicht im Dictionary (`KeyError`). Fängt niemand die Ausnahme ab, bricht das Programm mit einem Traceback ab. Genau das ist für ein Skript, das nachts auf einem Server läuft, selten gewollt: Es soll entweder den Fehler behandeln oder verständlich scheitern und vorher aufräumen.

Das Werkzeug dafür ist die `try`-Anweisung. Der Code, der scheitern könnte, steht im `try`-Block. Tritt dort eine Ausnahme auf, springt Python sofort in den passenden `except`-Block; die restlichen Zeilen des `try`-Blocks werden übersprungen. Nach den `except`-Blöcken dürfen zwei weitere Abschnitte folgen. Der `else`-Block läuft nur, wenn im `try`-Block *keine* Ausnahme auftrat, und der `finally`-Block läuft *immer*, egal ob Fehler oder nicht, selbst wenn ein `return` oder eine nicht behandelte Ausnahme dazwischenkommt. In `finally` gehört deshalb das Aufräumen hinein: Dateien schließen, temporäre Dateien löschen, Sperren freigeben.

Das erste Beispiel zeigt alle vier Teile an einer kleinen Funktion, die einen Prozentwert aus einem Text liest, wie ihn etwa `df` liefert (`"78%"`). Die Ausgaben zeigen die Reihenfolge, in der Python die Blöcke durchläuft.

```python
def lies_prozent(text):
    try:
        zahl = int(text.rstrip("%"))
        anteil = 100 / zahl
    except ValueError:
        print("  -> ValueError: keine Zahl")
        return None
    except ZeroDivisionError as fehler:
        print("  -> ZeroDivisionError:", fehler)
        return None
    else:
        print("  -> kein Fehler, Ergebnis berechnet")
        return anteil
    finally:
        print("  -> finally läuft immer")

for eingabe in ["50%", "abc", "0%"]:
    print(f"Eingabe {eingabe!r}")
    print("  Ergebnis:", lies_prozent(eingabe))
```

```text
Eingabe '50%'
  -> kein Fehler, Ergebnis berechnet
  -> finally läuft immer
  Ergebnis: 2.0
Eingabe 'abc'
  -> ValueError: keine Zahl
  -> finally läuft immer
  Ergebnis: None
Eingabe '0%'
  -> ZeroDivisionError: division by zero
  -> finally läuft immer
  Ergebnis: None
```

Bei `50%` läuft der `try`-Block ohne Fehler durch, danach der `else`-Block (er enthält das `return`), und *vor* der Rückgabe noch der `finally`-Block. Bei `abc` scheitert `int()` mit einem `ValueError`, und die Division wird gar nicht erst versucht. Bei `0%` gelingt die Umwandlung, aber die Division scheitert mit `ZeroDivisionError`; die Meldung der Ausnahme lässt sich über die Variable hinter `as` ausgeben. Beachten Sie, dass in allen Fällen `finally` ausgeführt wurde.

Ein wichtiger Gestaltungsgrundsatz lautet: *Fangen Sie so wenig wie möglich und so genau wie möglich.* Der `try`-Block soll nur die Zeilen enthalten, die wirklich scheitern können, und der `except`-Block nur die Ausnahmen, mit denen Sie etwas Sinnvolles anfangen können. Wer jede Ausnahme abfängt, versteckt Programmierfehler: Ein Tippfehler im Variablennamen (`NameError`) würde dann wie ein erwarteter Fehler behandelt. Um zu wissen, welche Ausnahmen es gibt, muss man ihre Familie kennen. Alle Ausnahmen sind Klassen und bilden eine Hierarchie. An der Spitze steht `BaseException`. Darunter liegt `Exception`, von der fast alle Ausnahmen für „normale“ Fehler erben, und daneben stehen einige Sonderfälle, die *keine* gewöhnlichen Fehler sind: `KeyboardInterrupt` (der Benutzer drückt Strg+C) und `SystemExit` (das Programm soll beenden). Eine `except`-Klausel fängt die genannte Klasse und alle Unterklassen. Für Dateioperationen wichtig ist `OSError`: Von ihr erben unter anderem `FileNotFoundError`, `FileExistsError`, `PermissionError`, `IsADirectoryError`, `NotADirectoryError` und `TimeoutError`. Wer `OSError` fängt, fängt also alle Datei- und Systemfehler; wer genauer reagieren will, fängt die Unterklassen einzeln. Dabei gilt: Python prüft die `except`-Zweige von oben nach unten und nimmt den ersten passenden. Der speziellere Zweig muss deshalb vor dem allgemeineren stehen.

```python
import tempfile
from pathlib import Path

def lese(pfad):
    try:
        return Path(pfad).read_text(encoding="utf-8")
    except FileNotFoundError:
        return "(Datei fehlt)"
    except IsADirectoryError:
        return "(ist ein Verzeichnis)"
    except PermissionError:
        return "(keine Berechtigung)"
    except OSError as fehler:
        return f"(anderer Systemfehler: {fehler.strerror})"

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "notiz.txt"
    datei.write_text("Hallo", encoding="utf-8")
    print(lese(datei))
    print(lese(Path(tmp) / "gibt-es-nicht.txt"))
    print(lese(tmp))

try:
    open("/gibt/es/nicht.txt")
except OSError as fehler:
    print(type(fehler).__name__, fehler.errno, fehler.strerror)
    print(isinstance(fehler, FileNotFoundError))
```

```text
Hallo
(Datei fehlt)
(ist ein Verzeichnis)
FileNotFoundError 2 No such file or directory
True
```

Die Funktion `lese` unterscheidet die wichtigsten Fehlerfälle beim Lesen einer Datei. Der letzte `except`-Zweig fängt alle übrigen Systemfehler und steht bewusst ganz unten, weil `OSError` die Oberklasse der drei vorigen ist; stünde er oben, würden die anderen nie erreicht. Im zweiten Teil sehen Sie, dass jede `OSError`-Ausnahme eine Fehlernummer (`errno`, hier 2 für „Datei nicht gefunden“) und einen Klartext (`strerror`) mitbringt und dass Python automatisch die passende Unterklasse auswählt.

Manchmal will man eine Ausnahme nicht behandeln, sondern nur etwas protokollieren und sie dann *weiterreichen*. Dafür dient `raise` ohne Argument innerhalb des `except`-Blocks, das die aktuelle Ausnahme erneut auslöst. Mit `raise NeueAusnahme(...)` erzeugt man eine eigene, aussagekräftigere Ausnahme. Eigene Ausnahmen definiert man als Klassen, die von `Exception` (oder einer passenden Unterklasse) erben; der Rumpf kann leer bleiben. Sie lohnen sich, weil der Aufrufer dann gezielt auf *Ihre* Fehlerart reagieren kann, ohne auf Fehlertexte zu schauen. Wird eine Ausnahme als Folge einer anderen ausgelöst, schreibt man `raise Neu(...) from ursache`: Python merkt sich die Ursache im Attribut `__cause__` und zeigt im Traceback beide Fehler an („The above exception was the direct cause of the following exception“). Ohne `from` verkettet Python die Ausnahmen trotzdem automatisch (Attribut `__context__`). Mit `from None` unterdrückt man die Anzeige der Ursache, was man nur verwendet, wenn sie den Leser verwirren würde.

```python
class KonfigFehler(Exception):
    """Die Konfiguration ist ungültig."""

def lade_port(text):
    try:
        port = int(text)
    except ValueError as ursache:
        raise KonfigFehler(f"Port {text!r} ist keine Zahl") from ursache
    if not 1 <= port <= 65535:
        raise KonfigFehler(f"Port {port} liegt außerhalb von 1-65535")
    return port

for eingabe in ["8080", "http", "70000"]:
    try:
        print("Port:", lade_port(eingabe))
    except KonfigFehler as fehler:
        print("Konfigurationsfehler:", fehler)
        if fehler.__cause__ is not None:
            print("  Ursache:", type(fehler.__cause__).__name__, "-", fehler.__cause__)

try:
    lade_port("abc")
except Exception as fehler:
    print(isinstance(fehler, KonfigFehler), isinstance(fehler, ValueError))
```

```text
Port: 8080
Konfigurationsfehler: Port 'http' ist keine Zahl
  Ursache: ValueError - invalid literal for int() with base 10: 'http'
Konfigurationsfehler: Port 70000 liegt außerhalb von 1-65535
True False
```

Die Funktion `lade_port` wandelt den technischen `ValueError` in einen fachlichen `KonfigFehler` um und behält die Ursache über `from`. Der Aufrufer fängt nur `KonfigFehler`; er muss nicht wissen, dass intern `int()` verwendet wurde. Im zweiten Fehlerfall (Port 70000) gibt es keine Ursache, weil keine andere Ausnahme vorausging; deshalb fehlt dort die Zeile „Ursache“. Der Test am Ende zeigt, dass ein `KonfigFehler` ein `KonfigFehler` ist, aber kein `ValueError`: Die Ursache ist verknüpft, die Klasse nicht vererbt.

Ein weiteres Muster, das in Administrationsskripten ständig vorkommt, ist das *Aufräumen*. Wer eine Datei öffnet, eine Sperrdatei anlegt oder eine Verbindung aufbaut, muss das auch dann wieder freigeben, wenn zwischendurch etwas schiefgeht. Zwei Werkzeuge leisten das: `try/finally` und die `with`-Anweisung, die Sie aus Modul 4 kennen. Die `with`-Anweisung ist nichts anderes als ein eingebautes `try/finally`, bei dem das Objekt (hier die Datei oder das Verzeichnis) selbst weiß, wie es aufzuräumen ist. Das Beispiel legt eine Sperrdatei an, die verhindern soll, dass zwei Instanzen eines Skripts gleichzeitig laufen, und gibt sie in jedem Fall wieder frei. Das Thema Sperren (Locking) kommt in Modul 9 ausführlicher.

```python
import tempfile
from pathlib import Path

def arbeite(sperrdatei, fehler_ausloesen):
    sperrdatei.write_text("läuft")          # Sperre setzen
    print("  Sperre gesetzt:", sperrdatei.exists())
    try:
        print("  arbeite ...")
        if fehler_ausloesen:
            raise RuntimeError("etwas ging schief")
        print("  fertig")
    finally:
        sperrdatei.unlink()                  # läuft in jedem Fall
        print("  Sperre entfernt:", not sperrdatei.exists())

with tempfile.TemporaryDirectory() as tmp:
    sperre = Path(tmp) / "skript.lock"
    print("Lauf 1 (ohne Fehler):")
    arbeite(sperre, False)
    print("Lauf 2 (mit Fehler):")
    try:
        arbeite(sperre, True)
    except RuntimeError as fehler:
        print("  Fehler beim Aufrufer angekommen:", fehler)
```

```text
Lauf 1 (ohne Fehler):
  Sperre gesetzt: True
  arbeite ...
  fertig
  Sperre entfernt: True
Lauf 2 (mit Fehler):
  Sperre gesetzt: True
  arbeite ...
  Sperre entfernt: True
  Fehler beim Aufrufer angekommen: etwas ging schief
```

In beiden Läufen wird die Sperre entfernt. Beim zweiten Lauf geschieht das, obwohl die Ausnahme den `try`-Block verlässt: Python führt zuerst den `finally`-Block aus und reicht die Ausnahme erst danach an den Aufrufer weiter. Ohne `finally` bliebe die Sperrdatei nach einem Fehler liegen, und das Skript würde bei jedem weiteren Start glauben, es laufe schon eine Instanz.

Typische Fehler beim Umgang mit Ausnahmen sind schnell aufgezählt, aber sie richten viel Schaden an. Der schlimmste ist das *nackte* `except:`, das *alles* fängt, auch `KeyboardInterrupt` und `SystemExit`. Ein Skript mit einem nackten `except` lässt sich nicht mehr mit Strg+C abbrechen. Fast ebenso schlimm ist `except Exception: pass`: Der Fehler wird gefangen und verschwindet spurlos, das Programm läuft mit falschen Daten weiter, und niemand erfährt etwas. Wer eine Ausnahme fängt, muss etwas damit tun: behandeln, protokollieren oder weiterreichen. Dazu kommt der zu große `try`-Block, in dem man nach einem Fehler nicht mehr erkennt, welche Zeile ihn verursacht hat.

```python
def falsch():
    try:
        raise KeyboardInterrupt      # Strg+C des Benutzers
    except:                          # FALSCH: nacktes except fängt alles
        return "verschluckt"

def richtig():
    try:
        raise KeyboardInterrupt
    except Exception:                # fängt KeyboardInterrupt NICHT
        return "gefangen"

print(falsch())
try:
    richtig()
except KeyboardInterrupt:
    print("KeyboardInterrupt ist durchgekommen")

# FALSCH: Fehler spurlos verschlucken
daten = {"a": 1}
try:
    wert = daten["b"]
except Exception:
    pass
try:
    print(wert)
except NameError as fehler:
    print("Folgefehler an anderer Stelle:", fehler)

# RICHTIG: gezielt fangen und eine Entscheidung treffen
wert = daten.get("b", 0)
print("Mit Standardwert:", wert)
```

```text
verschluckt
KeyboardInterrupt ist durchgekommen
Folgefehler an anderer Stelle: name 'wert' is not defined
Mit Standardwert: 0
```

Die erste Funktion fängt den simulierten Abbruch durch den Benutzer und tut so, als sei nichts gewesen; die zweite lässt ihn passieren, weil `KeyboardInterrupt` nicht von `Exception` erbt. Im mittleren Teil sehen Sie, wie ein verschluckter Fehler später an einer ganz anderen Stelle als `NameError` wieder auftaucht, und man sucht an der falschen Stelle nach der Ursache. Der letzte Teil zeigt die Alternative: Fehlt ein Schlüssel, legt man bewusst einen Standardwert fest.

Eine letzte Grundsatzfrage: Soll man *vorher prüfen* oder *es versuchen und den Fehler abfangen*? Die erste Haltung heißt in der Python-Gemeinde „Look before you leap“ (schau, bevor du springst), die zweite „Easier to ask forgiveness than permission“ (leichter um Verzeihung bitten als um Erlaubnis), kurz EAFP. In Python ist EAFP meist die bessere Wahl, besonders bei Dateien. Der Grund ist eine *Wettlaufsituation* (englisch *race condition*): Prüft man mit `exists()`, ob eine Datei da ist, und öffnet sie danach, kann sie in der Zwischenzeit von einem anderen Prozess gelöscht worden sein. Die Prüfung war dann wertlos. Wer die Datei einfach öffnet und `FileNotFoundError` abfängt, hat keine Lücke, in der etwas passieren kann.

> **Ergänzung aus der Nachrecherche**
>
> **Ausnahmen in der Python-Dokumentation (abgerufen: Built-in Exceptions, Version 3.14).** Die Dokumentation bestätigt die hier beschriebene Hierarchie: `KeyboardInterrupt` und `SystemExit` erben von `BaseException`, damit sie „nicht versehentlich“ von Code gefangen werden, der `Exception` fängt; eigene Ausnahmen sollen von `Exception` oder einer Unterklasse abgeleitet werden. Zu `raise ... from` steht dort: Die Verkettung setzt `__cause__`, `raise ... from None` blendet die Ursache im Traceback aus, sie bleibt aber in `__context__` erreichbar. Zwei Neuerungen seit Python 3.11 sind hilfreich: `add_note()` hängt einer Ausnahme eine zusätzliche Notiz an, die im Traceback nach der Meldung erscheint (etwa „beim Verarbeiten von Zeile 42“), und `ExceptionGroup` mit der Syntax `except*` bündelt mehrere voneinander unabhängige Ausnahmen, die man getrennt behandeln kann. Letzteres brauchen Sie vor allem bei nebenläufigem Code; im Rahmen dieses Skripts wird es nicht vertieft und die Beispiele wurden dazu nicht ausgeführt. Quelle: <https://docs.python.org/3.14/builtins/exceptions.html>.

> **Merke**
>
> `try` enthält nur die Zeilen, die scheitern können; `except` nennt konkrete Ausnahmen (speziellere zuerst); `else` läuft nur ohne Fehler; `finally` läuft immer und gehört zum Aufräumen. Fangen Sie nie `BaseException` oder nackt (`except:`) und schlucken Sie keine Fehler. Eigene Ausnahmen erben von `Exception`; mit `raise ... from ursache` bleibt die Ursache erhalten.

> **Praxisbeispiel**
>
> Ein Skript, das Konfigurationsdateien einliest, Dienste anspricht und Dateien schreibt, erlebt im Betrieb ständig erwartbare Störungen: eine Datei ist gesperrt, ein Verzeichnis fehlt, ein Server antwortet nicht. Gute Skripte fangen diese bekannten Fälle mit eigenen, sprechenden Ausnahmen ab (`KonfigFehler`, `SicherungFehler`), melden sie verständlich und beenden sich geordnet, während unbekannte Fehler mit vollem Traceback sichtbar bleiben. So unterscheidet man im Protokoll auf einen Blick „Das haben wir erwartet“ von „Hier stimmt etwas Grundsätzliches nicht“.

## 6.2 Fehlerbehandlung & Exit-Codes

Ein Skript, das Fehler abfängt, muss dem Rest der Welt auch mitteilen, ob es erfolgreich war. In einer Shell-Pipeline, einem Cronjob oder einem Monitoring-System liest niemand die Ausgabe; entscheidend ist der *Rückgabecode* (Exit-Code), den Sie in Modul 5 als Gegenstück zu `subprocess` kennengelernt haben. Jetzt wechseln wir die Perspektive: Nicht Ihr Skript wertet den Code eines Programms aus, sondern andere werten den Code *Ihres* Skripts aus.

Python bietet dafür die Funktion `sys.exit(code)`. Sie beendet das Programm, indem sie die Ausnahme `SystemExit` auslöst (deshalb laufen `finally`-Blöcke und `with`-Aufräumarbeiten vorher noch ab). Die Regeln sind einfach und stehen in der Python-Dokumentation zu `SystemExit`: Eine ganze Zahl wird zum Rückgabecode, `None` ergibt 0, und jedes andere Objekt, etwa ein Text, wird ausgegeben und liefert den Code 1. Endet das Skript regulär ohne `sys.exit`, ist der Code 0. Bricht es durch eine nicht abgefangene Ausnahme ab, gibt Python den Traceback auf `stderr` aus und beendet sich mit dem Code 1. Das folgende Beispiel startet kleine Kindprogramme und zeigt ihre Rückgabecodes und ihre Fehlerausgaben, damit man die Regeln sieht, wie ein anderes Programm sie sieht.

```python
import subprocess
import sys

def teste(beschreibung, code):
    r = subprocess.run([sys.executable, "-c", code], capture_output=True, text=True)
    letzte_zeile = r.stderr.strip().splitlines()[-1] if r.stderr.strip() else ""
    print(f"{beschreibung:<28} Code {r.returncode:>3}  stderr: {letzte_zeile!r}")

teste("Ende ohne sys.exit",        "print('fertig')")
teste("sys.exit(0)",               "import sys; sys.exit(0)")
teste("sys.exit(3)",               "import sys; sys.exit(3)")
teste("sys.exit('Text')",          "import sys; sys.exit('Fehler: Datei fehlt')")
teste("sys.exit(None)",            "import sys; sys.exit(None)")
teste("raise SystemExit(5)",       "raise SystemExit(5)")
teste("nicht gefangene Ausnahme",  "int('abc')")
teste("sys.exit(256)",             "import sys; sys.exit(256)")
```

```text
Ende ohne sys.exit           Code   0  stderr: ''
sys.exit(0)                  Code   0  stderr: ''
sys.exit(3)                  Code   3  stderr: ''
sys.exit('Text')             Code   1  stderr: 'Fehler: Datei fehlt'
sys.exit(None)               Code   0  stderr: ''
raise SystemExit(5)          Code   5  stderr: ''
nicht gefangene Ausnahme     Code   1  stderr: "ValueError: invalid literal for int() with base 10: 'abc'"
sys.exit(256)                Code   0  stderr: ''
```

Die Tabelle fasst die Regeln zusammen und enthält eine Überraschung in der letzten Zeile: Rückgabecodes sind unter Linux nur ein einzelnes Byte, also Zahlen von 0 bis 255. Größere Werte werden modulo 256 abgeschnitten, aus 256 wird 0, und ein Fehlerskript meldet plötzlich Erfolg. Bleiben Sie deshalb bei kleinen Zahlen. Beachten Sie auch, dass `sys.exit('Text')` den Text auf `stderr` ausgibt, wo auch Fehlermeldungen hingehören, und den Code 1 liefert. Das ist für kleine Skripte eine bequeme Abkürzung.

Welche Zahl soll man nun bei welchem Fehler wählen? Die Betriebssystem-Konvention legt nur 0 (Erfolg) fest. Darüber hinaus gelten Gepflogenheiten, die man kennen sollte, damit Codes nicht kollidieren. Der Code 1 steht für einen allgemeinen Fehler; 2 steht bei vielen Programmen, auch bei `argparse`, für falsche Benutzung. Die Shell (Bash) vergibt selbst 126 („Befehl gefunden, aber nicht ausführbar“) und 127 („Befehl nicht gefunden“), und ein durch Signal beendetes Programm erscheint dort als 128 plus Signalnummer; Strg+C (Signal 2) ergibt also 130. Diese Zuordnung zur Shell ist verbreitetes Fachwissen und wurde hier nicht in einer Quelle nachgeschlagen. Für eigene Skripte legt man eine kleine Tabelle fest, dokumentiert sie in der Hilfe und hält sie stabil, weil andere Programme sich darauf verlassen. Für Monitoring-Werkzeuge kennen Sie die Vierer-Konvention aus Modul 5 (0, 1, 2, 3 für OK, Warnung, kritisch, unbekannt).

Wie baut man das sauber in ein Programm ein? Das bewährte Muster: Eigene Ausnahmen tragen den passenden Rückgabecode als Attribut; `main` fängt sie an *einer* zentralen Stelle, gibt eine verständliche Meldung auf `stderr` aus und liefert den Code zurück; und nur dort steht `sys.exit(main())`. Unerwartete Ausnahmen fängt man nicht ab, damit der Traceback sichtbar wird, oder man protokolliert sie und gibt einen eigenen Code zurück.

```python
import contextlib
import sys

class AppFehler(Exception):
    """Basisklasse für erwartete Fehler; trägt den Rückgabecode."""
    exit_code = 1

class KonfigFehler(AppFehler):
    exit_code = 10

class ZielFehler(AppFehler):
    exit_code = 11

def arbeite(modus):
    if modus == "konfig":
        raise KonfigFehler("Konfigurationsdatei fehlt")
    if modus == "ziel":
        raise ZielFehler("Zielverzeichnis nicht beschreibbar")
    if modus == "bug":
        return 1 / 0                   # Programmierfehler, nicht erwartet
    print("Arbeit erledigt")

def main(modus):
    try:
        arbeite(modus)
    except AppFehler as fehler:
        print(f"Fehler: {fehler}", file=sys.stderr)
        return fehler.exit_code
    except KeyboardInterrupt:
        print("Abbruch durch den Benutzer", file=sys.stderr)
        return 130
    return 0

with contextlib.redirect_stderr(sys.stdout):      # nur damit man stderr hier sieht
    for modus in ["ok", "konfig", "ziel"]:
        print(modus, "->", main(modus))

try:
    main("bug")
except ZeroDivisionError:
    print("bug -> unerwartete Ausnahme kommt durch (Traceback, Code 1)")
```

```text
Arbeit erledigt
ok -> 0
Fehler: Konfigurationsdatei fehlt
konfig -> 10
Fehler: Zielverzeichnis nicht beschreibbar
ziel -> 11
bug -> unerwartete Ausnahme kommt durch (Traceback, Code 1)
```

Das Programm besitzt eine Basisklasse `AppFehler` für erwartete Fehler. Die Unterklassen unterscheiden sich nur im Rückgabecode. Durch den Klassenaufbau braucht `main` nur einen `except`-Zweig für alle erwarteten Fehler, weil die Unterklassen mitgefangen werden. Die Fehlermeldungen gehen mit `file=sys.stderr` auf den Fehlerkanal, die normale Ausgabe bleibt auf `stdout`: Ein aufrufendes Skript kann so Ergebnis und Fehler getrennt weiterverwenden. Den Programmierfehler (`ZeroDivisionError`) fängt `main` bewusst nicht ab; im echten Betrieb würde das Skript hier mit Traceback und Code 1 enden, und das ist gewollt, denn ein Bug soll auffallen.

Wie sieht der Code von außen aus? Die Shell speichert ihn in der Variablen `$?`. Das nächste Beispiel startet `sh` mit einer kleinen Befehlszeile, die ein Python-Kindprogramm ausführt und danach `$?` anzeigt. Zusätzlich sehen Sie, was bei Strg+C passiert. Ein Python-Programm, das durch Strg+C unterbrochen wird, löst `KeyboardInterrupt` aus. Wird die Ausnahme nicht gefangen, beendet sich Python seit Version 3.8 so, dass das Betriebssystem das Programm als durch Signal 2 beendet sieht; in Python erscheint das als Rückgabecode `-2`, in der Shell als 130. Fängt das Programm die Ausnahme und ruft `sys.exit(130)` auf, sieht man direkt 130.

```python
import subprocess
import sys

py = sys.executable

# 1) Die Shell liest den Code des Kindes aus $?
befehl = f'"{py}" -c "import sys; sys.exit(7)"; echo "Shell sieht: $?"'
r = subprocess.run(["sh", "-c", befehl], capture_output=True, text=True)
print(r.stdout.strip())

# 2) Strg+C: Signal SIGINT an sich selbst senden
simuliere = "import os, signal; os.kill(os.getpid(), signal.SIGINT)"
r = subprocess.run([py, "-c", simuliere], capture_output=True, text=True)
print("Ohne Behandlung:", r.returncode)

behandelt = (
    "import os, signal, sys\n"
    "try:\n"
    "    os.kill(os.getpid(), signal.SIGINT)\n"
    "    import time; time.sleep(1)\n"
    "except KeyboardInterrupt:\n"
    "    sys.exit(130)\n"
)
r = subprocess.run([py, "-c", behandelt], capture_output=True, text=True)
print("Mit Behandlung: ", r.returncode)
```

```text
Shell sieht: 7
Ohne Behandlung: -2
Mit Behandlung:  130
```

Die dritte Zeile zeigt, warum man es bewusst entscheidet: Ob ein aufrufendes Programm „-2“ (Python-Sicht auf ein Signal) oder „130“ (Shell-Konvention) sieht, hängt davon ab, wer fragt und ob Ihr Skript die Unterbrechung selbst behandelt.

Typische Fehler beim Thema Exit-Codes sind: Der Fehler wird zwar gemeldet, aber das Skript endet trotzdem mit 0, weil man vergessen hat, den Code zu setzen (das Monitoring zeigt dann „grün“); Meldungen gehen mit `print` auf `stdout` und stören damit die Ausgabe, die ein anderes Programm lesen will; Rückgabecodes über 255 werden zu etwas ganz anderem; und `sys.exit` wird tief in einer Hilfsfunktion aufgerufen, sodass sich die Funktion nicht mehr testen lässt, weil der Test selbst beendet wird. Besser ist: Hilfsfunktionen lösen Ausnahmen aus, nur `main` (oder die oberste Ebene) übersetzt sie in Rückgabecodes.

```python
import subprocess
import sys

py = sys.executable

falsch = (
    "try:\n"
    "    open('/gibt/es/nicht')\n"
    "except OSError as fehler:\n"
    "    print('Fehler:', fehler)\n"        # nur gemeldet, Code bleibt 0
)
r = subprocess.run([py, "-c", falsch], capture_output=True, text=True)
print("FALSCH :", r.returncode, repr(r.stdout.strip()))

richtig = (
    "import sys\n"
    "try:\n"
    "    open('/gibt/es/nicht')\n"
    "except OSError as fehler:\n"
    "    print('Fehler:', fehler, file=sys.stderr)\n"
    "    sys.exit(2)\n"
)
r = subprocess.run([py, "-c", richtig], capture_output=True, text=True)
print("RICHTIG:", r.returncode, repr(r.stdout.strip()), repr(r.stderr.strip()))
```

```text
FALSCH : 0 "Fehler: [Errno 2] No such file or directory: '/gibt/es/nicht'"
RICHTIG: 2 '' "Fehler: [Errno 2] No such file or directory: '/gibt/es/nicht'"
```

Im falschen Fall sieht ein Aufrufer nur den Code 0 und glaubt an einen Erfolg; die Fehlermeldung steht obendrein im Ausgabestrom, wo sie als Ergebnis missverstanden werden kann. Im richtigen Fall melden sowohl Code als auch Kanal den Fehler.

> **Ergänzung aus der Nachrecherche**
>
> **sys.exit und SystemExit (abgerufen: Python-Dokumentation und ein Lernartikel).** Die Dokumentation der eingebauten Ausnahmen bestätigt: `SystemExit` wird von `sys.exit()` ausgelöst; ein ganzzahliges Argument setzt den Rückgabecode, `None` ergibt 0, und ein Argument anderen Typs wird ausgegeben und liefert 1. Ein Lernartikel auf <https://www.pythonmorsels.com/exiting-a-python-program/> empfiehlt, Rückgabecodes im Bereich 0 bis 127 zu halten, weil größere Werte auf manchen Systemen nicht eindeutig sind; unsere Messung bestätigt auf Linux den Wert 256 als 0. Wichtig für Skripte: Weil `SystemExit` von `BaseException` erbt, fängt ein `except Exception` den Aufruf von `sys.exit()` nicht, ein nacktes `except` dagegen schon.

> **Merke**
>
> Der Rückgabecode ist die Sprache, in der Ihr Skript mit der Außenwelt spricht: 0 für Erfolg, alles andere für einen Fehler, höchstens bis 255. Hilfsfunktionen lösen Ausnahmen aus, `main` übersetzt sie in Meldung (auf `stderr`) und Code; nur die oberste Ebene ruft `sys.exit(main())` auf. Dokumentieren Sie Ihre Codes in der Hilfe, und ändern Sie sie später nicht mehr.

> **Praxisbeispiel**
>
> Cron führt Ihr Skript nachts aus und schickt bei Ausgaben eine Mail; systemd-Dienste und -Timer werten den Rückgabecode aus und markieren die Einheit bei einem Fehler als „failed“ (Modul 9); eine CI-Pipeline stoppt bei einem Code ungleich 0 den nächsten Schritt. In all diesen Fällen entscheidet der Code, ob jemand geweckt wird oder nicht. Ein Skript, das bei einem Fehler 0 zurückgibt, ist gefährlicher als eines, das gar nicht läuft, weil es Sicherheit vortäuscht.

## 6.3 logging Grundlagen

Ein Skript, das unbeaufsichtigt läuft, muss Spuren hinterlassen: Was hat es getan, wann, und was ist schiefgegangen? Bisher haben Sie dafür `print` benutzt. Das reicht für ein Experiment, aber nicht für den Betrieb. Eine `print`-Zeile hat keinen Zeitstempel, keine Wichtigkeit und keinen Absender; man kann sie nicht nach Dringlichkeit filtern, nicht gleichzeitig in eine Datei und auf den Bildschirm schreiben und nicht abschalten, ohne den Code zu ändern. Das Modul `logging` aus der Standardbibliothek löst genau diese Probleme.

Das Modul kennt vier Grundbegriffe. Ein *Logger* ist das Objekt, an das Ihr Code Meldungen schickt (`logger.info("...")`). Jede Meldung hat eine *Stufe* (englisch *level*), die ihre Wichtigkeit angibt. Ein *Handler* entscheidet, wohin die Meldung geschrieben wird: auf den Bildschirm, in eine Datei, an den Systemlog. Ein *Formatter* bestimmt, wie die Zeile aussieht, also ob sie Zeitstempel, Stufe und Absender enthält. Die Trennung ist der Kern der Sache: Der Code, der meldet, weiß nichts davon, wohin die Meldung geht; das legt man einmal zentral beim Programmstart fest.

Die fünf Stufen sind nach Wichtigkeit geordnet. Die Python-Dokumentation beschreibt sie so: `DEBUG` (10) für Detailinformationen, die nur bei der Fehlersuche interessieren; `INFO` (20) als Bestätigung, dass alles wie erwartet läuft; `WARNING` (30) für etwas Unerwartetes oder ein künftiges Problem; `ERROR` (40), wenn etwas nicht ausgeführt werden konnte; `CRITICAL` (50) für einen schweren Fehler, der das Programm gefährdet. Ein Logger gibt nur Meldungen aus, die mindestens seine eingestellte Stufe haben. Die voreingestellte Stufe ist `WARNING`: Ohne Konfiguration verschwinden `DEBUG` und `INFO` spurlos. Das überrascht Anfänger regelmäßig.

Die einfachste Konfiguration ist `logging.basicConfig(...)`. Man ruft sie einmal am Programmanfang auf und gibt Stufe, Format und Ziel an. Das Format ist ein Text mit Platzhaltern wie `%(levelname)s` (Stufe), `%(name)s` (Name des Loggers), `%(message)s` (die Meldung) und `%(asctime)s` (Zeitstempel). Wir lassen in den Beispielen den Zeitstempel zunächst weg, damit die Ausgabe zum Vergleichen stabil bleibt. Standardmäßig schreibt `basicConfig` auf `stderr`; wir leiten es hier mit `stream=sys.stdout` um, damit die Ausgabe im Beispiel erscheint.

```python
import logging
import sys

logging.basicConfig(
    level=logging.DEBUG,
    format="%(levelname)-8s %(name)s: %(message)s",
    stream=sys.stdout,
)
log = logging.getLogger("backup")

log.debug("Detail für die Fehlersuche")
log.info("Sicherung gestartet")
log.warning("Zielplatte zu 85 Prozent voll")
log.error("Archiv konnte nicht geschrieben werden")
log.critical("Kein Speicherplatz mehr")
```

```text
DEBUG    backup: Detail für die Fehlersuche
INFO     backup: Sicherung gestartet
WARNING  backup: Zielplatte zu 85 Prozent voll
ERROR    backup: Archiv konnte nicht geschrieben werden
CRITICAL backup: Kein Speicherplatz mehr
```

Mit `logging.getLogger("backup")` holt man einen benannten Logger. Der Platzhalter `%(levelname)-8s` gibt die Stufe linksbündig mit acht Zeichen Breite aus, damit die Spalten untereinander stehen. Wäre die Stufe auf `INFO` gestellt, fehlte die erste Zeile; bei `WARNING` (der Voreinstellung) blieben nur die letzten drei.

Was passiert ganz ohne Konfiguration? Dann arbeitet der „Root“-Logger mit seinen Standardeinstellungen: Stufe `WARNING`, Ausgabe auf `stderr`, Format `STUFE:Name:Meldung`. Das zeigt das nächste Beispiel; es leitet `stderr` nach `stdout` um, damit man es sieht.

```python
import contextlib
import logging
import sys

with contextlib.redirect_stderr(sys.stdout):
    logging.debug("wird nicht angezeigt")
    logging.info("wird nicht angezeigt")
    logging.warning("wird angezeigt")
    logging.error("wird angezeigt")
```

```text
WARNING:root:wird angezeigt
ERROR:root:wird angezeigt
```

Dies erklärt auch eine häufige Falle: Ruft man `logging.warning()` (oder `info`, `error`) direkt auf, richtet das Modul automatisch eine Standardkonfiguration ein, *falls noch keine existiert*. Ein späterer Aufruf von `basicConfig` bleibt dann wirkungslos, weil schon ein Handler vorhanden ist. Die Regel lautet daher: `basicConfig` zuerst, am Programmanfang, vor jeder Meldung. Seit Python 3.8 kann man mit `force=True` eine vorhandene Konfiguration ersetzen.

Für richtige Programme genügt `basicConfig` bald nicht mehr, denn man will häufig die Konsole knapp und die Datei ausführlich haben. Dann baut man die Handler selbst. Jeder Handler hat eine eigene Stufe und einen eigenen Formatter. Das folgende Beispiel schreibt `INFO` und höher auf den Bildschirm (kurz) und alles ab `DEBUG` in eine Datei (mit Zeitstempel). Der Logger selbst muss auf die niedrigste benötigte Stufe stehen, sonst kommen die Meldungen gar nicht erst bei den Handlern an. Um die Datei lesbar vorzuführen, geben wir ihre Zeilen ohne den Zeitstempel aus; der Zeitstempel hat das Format `2026-12-03 19:04:12,345`, ist also 23 Zeichen lang.

```python
import logging
import sys
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    logdatei = Path(tmp) / "backup.log"

    log = logging.getLogger("backup")
    log.setLevel(logging.DEBUG)

    konsole = logging.StreamHandler(sys.stdout)
    konsole.setLevel(logging.INFO)
    konsole.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

    datei = logging.FileHandler(logdatei, encoding="utf-8")
    datei.setLevel(logging.DEBUG)
    datei.setFormatter(logging.Formatter("%(asctime)s %(levelname)-7s %(name)s: %(message)s"))

    log.addHandler(konsole)
    log.addHandler(datei)

    log.debug("Quelle: /srv/daten")
    log.info("Sicherung gestartet")
    log.warning("Datei gesperrt, übersprungen")

    print("--- Inhalt der Logdatei (ohne Zeitstempel) ---")
    for zeile in logdatei.read_text(encoding="utf-8").splitlines():
        print(zeile[24:])

    log.removeHandler(datei)
    datei.close()
```

```text
INFO: Sicherung gestartet
WARNING: Datei gesperrt, übersprungen
--- Inhalt der Logdatei (ohne Zeitstempel) ---
DEBUG   backup: Quelle: /srv/daten
INFO    backup: Sicherung gestartet
WARNING backup: Datei gesperrt, übersprungen
```

Auf dem Bildschirm erscheinen nur die beiden wichtigeren Meldungen im knappen Format; die Datei enthält alle drei, mit Zeitstempel, Stufe und Loggername. Zum Aufräumen am Ende entfernen wir den Handler und schließen die Datei, damit das temporäre Verzeichnis gelöscht werden kann; in einem echten Programm bleibt der Handler bis zum Programmende bestehen.

Wie benennt man Logger? Die Dokumentation empfiehlt, in jedem Modul einen Logger mit dem Namen des Moduls anzulegen: `log = logging.getLogger(__name__)`. Die Namen bilden eine Hierarchie, die durch Punkte getrennt ist: `app.backup` ist ein Kind von `app`, und dieses von der Wurzel (Root). Meldungen eines Kindes werden an die Handler der Eltern weitergereicht („propagiert“). Dadurch genügt es, Handler und Stufe *einmal* an der Wurzel oder an `app` einzurichten; einzelne Teile des Programms lassen sich über ihren Loggernamen gezielt lauter oder leiser stellen.

```python
import logging
import sys

logging.basicConfig(level=logging.INFO, stream=sys.stdout,
                    format="%(levelname)-7s %(name)-12s %(message)s")

backup = logging.getLogger("app.backup")
netz = logging.getLogger("app.netz")

netz.setLevel(logging.WARNING)      # nur dieser Teil wird leiser

backup.info("Archiv angelegt")
netz.info("Verbindung aufgebaut")   # unterdrückt
netz.warning("Antwort langsam")
backup.debug("zu detailliert für INFO")   # unterdrückt
```

```text
INFO    app.backup   Archiv angelegt
WARNING app.netz     Antwort langsam
```

Die Handler hängen am Root-Logger (`basicConfig` hat ihn eingerichtet), die beiden Logger darunter erben sie. Nur `app.netz` bekam eine eigene, strengere Stufe. In größeren Programmen setzt man dieses Prinzip um, indem man beim Start eine zentrale Konfiguration lädt; Bibliotheken dürfen dagegen keine Handler konfigurieren, sondern nur Logger anlegen und melden.

Besondere Aufmerksamkeit verdienen Fehlermeldungen. Wenn man eine Ausnahme abfängt und nur `log.error(str(fehler))` schreibt, geht der Traceback verloren, und damit die wichtigste Information zur Fehlersuche. Dafür gibt es `log.exception("Text")`: Sie schreibt eine Meldung der Stufe `ERROR` *und* hängt den vollständigen Traceback an. Man darf sie nur innerhalb eines `except`-Blocks aufrufen. Alternativ erhält jede Logmethode den Parameter `exc_info=True`.

```python
import logging
import sys

logging.basicConfig(level=logging.INFO, stream=sys.stdout,
                    format="%(levelname)s %(message)s")
log = logging.getLogger("demo")

def teile(a, b):
    return a / b

try:
    teile(1, 0)
except ZeroDivisionError:
    log.exception("Berechnung fehlgeschlagen")
```

```text
ERROR Berechnung fehlgeschlagen
Traceback (most recent call last):
  File "beispiel.py", line 12, in <module>
    teile(1, 0)
    ~~~~~^^^^^^
  File "beispiel.py", line 9, in teile
    return a / b
           ~~^~~
ZeroDivisionError: division by zero
```

Die erste Zeile ist unsere Meldung, die übrigen sind der Traceback, den Sie aus Modul 3 kennen. Die Pfeile und Wellenlinien unter den Codezeilen zeigen die genaue Stelle; ihr Aussehen hängt von der Python-Version ab und kann bei Ihnen abweichen. Der Pfad der Datei erscheint bei Ihnen vollständig.

Ein weiterer Punkt betrifft die Art, wie man Texte in Meldungen einsetzt. Statt eines f-Strings übergibt man die Werte als getrennte Argumente: `log.info("Datei %s: %d Bytes", name, groesse)`. Das Modul setzt den Text erst zusammen, wenn die Meldung tatsächlich ausgegeben wird. Ist die Stufe niedriger eingestellt, entfällt die Arbeit. Die Dokumentation beschreibt das so: Die Formatierung der Argumente wird zurückgestellt, bis sie unvermeidbar ist. Der Unterschied ist bei teuren Ausdrücken spürbar, wie das Beispiel mit einer Klasse zeigt, deren Textdarstellung aufwendig ist.

```python
import logging

class Teuer:
    def __str__(self):
        print("  (Textdarstellung wird berechnet)")
        return "Ergebnis"

logging.basicConfig(level=logging.WARNING)      # INFO ist ausgeschaltet
log = logging.getLogger("demo")

print("Mit f-String:")
log.info(f"Wert: {Teuer()}")

print("Mit Argumenten:")
log.info("Wert: %s", Teuer())
```

```text
Mit f-String:
  (Textdarstellung wird berechnet)
Mit Argumenten:
```

Beim f-String wird der Text berechnet, obwohl die Meldung gar nicht ausgegeben wird. Bei der Argumentform entfällt die Arbeit. Mehr noch: Wenn die Berechnung der Argumente selbst teuer ist, schützt man sie mit `if log.isEnabledFor(logging.DEBUG):`, wie es die Dokumentation empfiehlt.

Zum Schluss noch ein Wort zu Logdateien auf dem Server. Eine Datei, in die ein Skript unbegrenzt hineinschreibt, füllt irgendwann die Platte. Deshalb gibt es die *Rotation*: Ist die Datei groß genug, wird sie umbenannt (`app.log.1`), und eine neue beginnt; alte Dateien werden nach einer Anzahl gelöscht. Linux-Server erledigen das meist mit dem Programm `logrotate`; Python bietet den `RotatingFileHandler` (nach Größe) und den `TimedRotatingFileHandler` (nach Zeit). Die Dokumentation führt beide in der Liste der verfügbaren Handler auf.

```python
import logging
import tempfile
from logging.handlers import RotatingFileHandler
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "app.log"
    handler = RotatingFileHandler(datei, maxBytes=300, backupCount=2, encoding="utf-8")
    handler.setFormatter(logging.Formatter("%(message)s"))
    log = logging.getLogger("rotation")
    log.setLevel(logging.INFO)
    log.addHandler(handler)

    for i in range(1, 31):
        log.info("Meldung Nummer %02d mit etwas Text", i)

    print(sorted(p.name for p in Path(tmp).iterdir()))
    letzte = (Path(tmp) / "app.log").read_text(encoding="utf-8").splitlines()
    print(letzte[-1])

    log.removeHandler(handler)
    handler.close()
```

```text
['app.log', 'app.log.1', 'app.log.2']
Meldung Nummer 30 mit etwas Text
```

Mit `maxBytes=300` wird die Datei nach etwa 300 Byte rotiert, und `backupCount=2` behält nur zwei alte Dateien. Nach 30 Meldungen existieren `app.log` (die aktuelle), `app.log.1` und `app.log.2`; ältere Meldungen sind bereits gelöscht. In der Praxis wählt man deutlich größere Werte, etwa einige Megabyte.

Zu den typischen Fehlern gehören: `print` statt `logging`; `basicConfig` nach der ersten Meldung aufrufen (wirkungslos); `DEBUG`-Meldungen erwarten, ohne die Stufe zu senken; eine Ausnahme mit `log.error(str(e))` melden und den Traceback verlieren; und Geheimnisse wie Passwörter, Tokens oder personenbezogene Daten in den Log schreiben. Der letzte Punkt ist besonders heikel: Logdateien werden oft lange aufbewahrt, kopiert und an Support-Mitarbeiter weitergegeben. Die Beispiele von Modul 10 (Secrets) greifen das auf.

```python
import contextlib
import logging
import sys

# FALSCH: erst melden, dann konfigurieren; basicConfig bleibt wirkungslos
with contextlib.redirect_stderr(sys.stdout):
    logging.warning("erste Meldung")
    logging.basicConfig(level=logging.DEBUG, format="%(levelname)s - %(message)s")
    logging.debug("diese Debug-Meldung fehlt")

# RICHTIG: force=True ersetzt die vorhandene Konfiguration (ab Python 3.8)
logging.basicConfig(level=logging.DEBUG, format="%(levelname)s - %(message)s",
                    stream=sys.stdout, force=True)
logging.debug("jetzt ist sie da")

# FALSCH: Traceback geht verloren
try:
    int("x")
except ValueError as fehler:
    logging.error(str(fehler))
```

```text
WARNING:root:erste Meldung
DEBUG - jetzt ist sie da
ERROR - invalid literal for int() with base 10: 'x'
```

> **Ergänzung aus der Nachrecherche**
>
> **Aussagen der logging-HOWTO (abgerufen: docs.python.org/3/howto/logging.html).** Die Stufen mit ihren Zahlenwerten (10, 20, 30, 40, 50) und die Voreinstellung `WARNING` sind dort so beschrieben, wie oben dargestellt. Für `basicConfig` nennt die Seite die Parameter `filename`, `filemode` (`'w'` überschreibt bei jedem Lauf, sonst wird angehängt), `format`, `datefmt` und `level`; `basicConfig` soll aufgerufen werden, bevor ein Logger Meldungen schickt. Als Konvention steht dort: ein Logger je Modul, benannt mit `logging.getLogger(__name__)`. Zu `logger.exception()` heißt es, es sei wie `error()`, gebe aber zusätzlich den Traceback aus und dürfe nur in einem Exception-Handler aufgerufen werden. Der Parameter `handlers` von `basicConfig` sowie `force` (3.8) werden auf dieser Seite nicht beschrieben; sie stehen in der Dokumentation des Moduls `logging`, die hier nicht abgerufen wurde, und wurden durch Ausführen mit Python 3.13 bestätigt (`force`). Quelle: <https://docs.python.org/3/howto/logging.html>.

> **Merke**
>
> Melden Sie mit `logging`, nicht mit `print`: ein Logger pro Modul (`getLogger(__name__)`), Konfiguration einmal am Programmanfang (`basicConfig` oder eigene Handler), Werte als Argumente statt f-String, Ausnahmen mit `log.exception`. Wählen Sie die Stufe nach Bedeutung: `INFO` für den normalen Ablauf, `WARNING` für Auffälliges, `ERROR` für Fehlgeschlagenes. Schreiben Sie keine Geheimnisse in den Log.

> **Praxisbeispiel**
>
> Wenn ein Cronjob nachts um drei Uhr scheitert, ist das Protokoll Ihre einzige Quelle. Ein guter Log beantwortet am Morgen: Wann lief das Skript, was hat es getan, was war der erste Fehler, und welcher Traceback gehört dazu? Läuft das Skript als systemd-Dienst, landen die Ausgaben auf `stdout` und `stderr` im Journal; dort genügt ein Handler, der auf `stderr` schreibt, und Sie lesen die Meldungen mit `journalctl` (siehe Modul 9). Läuft es per Cron, leiten Sie die Ausgabe in eine Datei um oder nutzen einen `FileHandler`.

## 6.4 Backup-Skript Teil 1 (tar/rsync)

Mit Ausnahmen, Rückgabecodes und Protokollierung haben Sie jetzt das Handwerkszeug für ein Programm, dem man etwas Wichtiges anvertrauen kann. Die klassische Aufgabe dieser Art ist die *Datensicherung*. Ein Backup ist eine Kopie von Daten, die man benutzt, wenn das Original verloren geht: durch einen Hardwaredefekt, einen Bedienfehler, einen Softwarefehler oder einen Angriff mit Verschlüsselungstrojanern. Dabei gilt ein Grundsatz, der in der Praxis oft vergessen wird: *Ein Backup ist erst dann ein Backup, wenn man die Wiederherstellung getestet hat.* Eine Sicherung, die nie gelesen wurde, kann beschädigt, unvollständig oder leer sein, und das erfährt man sonst im schlimmsten Moment.

Zwei Begriffe gehören zum Fachwortschatz. Eine *vollständige Sicherung* (Vollbackup) kopiert alle Daten. Eine *inkrementelle Sicherung* kopiert nur, was sich seit der letzten Sicherung geändert hat; sie spart Platz und Zeit, braucht zur Wiederherstellung aber die ganze Kette. Eine bekannte Faustregel für den Aufbewahrungsort ist die 3-2-1-Regel: drei Kopien der Daten (das Original und zwei Sicherungen), auf zwei verschiedenen Medienarten, eine davon an einem anderen Ort. Ein Artikel von NAKIVO, den wir dazu gelesen haben, nennt sie einen bewährten Ansatz, den auch Sicherheitsfachleute und die US-Behörde CISA empfehlen; er schreibt sie keiner Person zu. Für unser Skript heißt das: Es kann nur *eine* der Kopien erzeugen. Dass sie auf ein zweites Medium und an einen anderen Ort gelangt, ist eine Frage der Einrichtung (zum Beispiel ein Netzlaufwerk als Ziel oder ein anschließendes `rsync` auf einen anderen Server).

Für das Packen von Dateien gibt es unter Linux seit Jahrzehnten das Programm `tar` (von *tape archive*). Es fasst viele Dateien und Verzeichnisse zu einer einzigen Datei zusammen, dem *Archiv*; ein Archiv ist noch nicht komprimiert. Die Kompression übernimmt ein zweites Programm, meist `gzip`, was zu Dateien mit der Endung `.tar.gz` führt. Python braucht dafür kein externes Programm: Das Modul `tarfile` der Standardbibliothek liest und schreibt dasselbe Format. Der Vorteil gegenüber dem Aufruf von `tar` über `subprocess` ist, dass man Fehler als Python-Ausnahmen bekommt, einzelne Dateien in einer Filterfunktion ausschließen kann und nicht von der Version des Programms abhängt. Ein Archiv öffnet man mit `tarfile.open(pfad, "w:gz")` zum Schreiben mit gzip-Kompression und mit `"r:gz"` zum Lesen. Mit `tar.add(pfad, arcname=..., filter=...)` fügt man ein Verzeichnis samt Inhalt hinzu. Der Parameter `arcname` legt fest, unter welchem Namen es im Archiv erscheint, und `filter` ist eine Funktion, die für jeden Eintrag aufgerufen wird; gibt sie `None` zurück, wird der Eintrag (und bei einem Verzeichnis alles darin) übersprungen.

Das folgende Beispiel sichert ein kleines Verzeichnis, schließt Dateien mit der Endung `.tmp` aus, liest das Archiv wieder ein und stellt es in einem anderen Verzeichnis wieder her. Danach vergleicht es Original und Wiederherstellung mit `filecmp.dircmp`. Genau dieser Vergleich ist der *Wiederherstellungstest* im Kleinen.

```python
import filecmp
import tarfile
import tempfile
from pathlib import Path

def ohne_tmp(eintrag):
    """Filter: Dateien mit der Endung .tmp nicht ins Archiv aufnehmen."""
    if eintrag.name.endswith(".tmp"):
        return None
    return eintrag

with tempfile.TemporaryDirectory() as tmp:
    quelle = Path(tmp) / "daten"
    (quelle / "unter").mkdir(parents=True)
    (quelle / "a.txt").write_text("Alpha\n", encoding="utf-8")
    (quelle / "unter" / "b.txt").write_text("Beta\n", encoding="utf-8")
    (quelle / "cache.tmp").write_text("flüchtig", encoding="utf-8")
    archiv = Path(tmp) / "sicherung.tar.gz"

    with tarfile.open(archiv, "w:gz") as tar:
        tar.add(quelle, arcname="daten", filter=ohne_tmp)

    ziel = Path(tmp) / "wiederhergestellt"
    with tarfile.open(archiv, "r:gz") as tar:
        print(sorted(tar.getnames()))
        tar.extractall(ziel, filter="data")

    vergleich = filecmp.dircmp(quelle, ziel / "daten")
    print("nur in der Quelle:", vergleich.left_only)
    print("unterschiedlich:  ", vergleich.diff_files)
```

```text
['daten', 'daten/a.txt', 'daten/unter', 'daten/unter/b.txt']
nur in der Quelle: ['cache.tmp']
unterschiedlich:   []
```

Das Archiv enthält vier Einträge: das Verzeichnis `daten`, die beiden Textdateien und das Unterverzeichnis; `cache.tmp` fehlt, wie gewünscht. Beim Vergleich meldet `dircmp`, dass nur die ausgeschlossene Datei im Original, aber nicht in der Kopie vorhanden ist und dass keine Datei unterschiedlichen Inhalt hat. Beachten Sie das Argument `filter="data"` beim Entpacken. Es schützt vor manipulierten Archiven, die versuchen, Dateien außerhalb des Zielverzeichnisses zu überschreiben (etwa mit Namen wie `../../etc/passwd`). Die Hintergründe stehen in der Ergänzung weiter unten.

Wer lieber das Programm `tar` selbst benutzt, kann es mit `subprocess` aufrufen, wie Sie es in Modul 5 gelernt haben: Befehl als Liste, Rückgabecode prüfen. Die wichtigsten Optionen sind `-c` (create, Archiv anlegen), `-z` (mit gzip komprimieren), `-f` (Dateiname des Archivs), `-t` (Inhalt auflisten), `-x` (entpacken), `-C` (vor dem Packen in ein Verzeichnis wechseln, damit keine langen Pfade im Archiv stehen) und `--exclude` (Muster ausschließen). Das nächste Beispiel erzeugt dasselbe Archiv mit `tar` und listet den Inhalt auf.

```python
import subprocess
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    quelle = Path(tmp) / "daten"
    (quelle / "unter").mkdir(parents=True)
    (quelle / "a.txt").write_text("Alpha\n", encoding="utf-8")
    (quelle / "unter" / "b.txt").write_text("Beta\n", encoding="utf-8")
    (quelle / "cache.tmp").write_text("flüchtig", encoding="utf-8")
    archiv = Path(tmp) / "sicherung.tar.gz"

    packen = ["tar", "-czf", str(archiv), "-C", tmp, "--exclude=*.tmp", "daten"]
    r = subprocess.run(packen, capture_output=True, text=True)
    print("tar endete mit Code", r.returncode)

    r = subprocess.run(["tar", "-tzf", str(archiv)],
                       capture_output=True, text=True, check=True)
    print(sorted(r.stdout.split()))
```

```text
tar endete mit Code 0
['daten/', 'daten/a.txt', 'daten/unter/', 'daten/unter/b.txt']
```

Beachten Sie die Unterschiede zum Python-Modul: `tar` schreibt Verzeichnisse mit einem Schrägstrich am Ende, und der Ausschluss funktioniert über ein Muster in der Befehlszeile. Der Rückgabecode des echten `tar` verdient Aufmerksamkeit: Er ist 0 bei Erfolg; GNU `tar` benutzt außerdem 1 für „einige Dateien haben sich während des Packens geändert“ (die Sicherung ist dann vorhanden, aber möglicherweise inkonsistent) und 2 für einen schweren Fehler. Diese Zuordnung beruht auf Fachwissen und wurde hier nicht in der Dokumentation nachgeschlagen; lesen Sie bei Ihrer `tar`-Version `man tar` im Abschnitt über den Exit-Status. Das ist ein Beispiel für den Grundsatz aus Modul 5: Der Code „ungleich 0“ muss differenziert gelesen werden.

Das zweite wichtige Sicherungswerkzeug ist `rsync`. Es kopiert nicht in ein Archiv, sondern *synchronisiert* zwei Verzeichnisse und überträgt dabei nur die Unterschiede, auch über das Netzwerk (per SSH). Das macht es ideal für regelmäßige Spiegelungen großer Datenmengen. Die wichtigsten Optionen: `-a` (Archivmodus: rekursiv, mit Rechten, Zeitstempeln, Besitzern und symbolischen Links), `-n` oder `--dry-run` (Probelauf: zeigt, was passieren würde), `--delete` (löscht im Ziel, was in der Quelle nicht mehr existiert) und `--exclude=MUSTER`. Zwei Fallen sind typisch. Erstens ändert ein Schrägstrich am Ende der Quelle die Bedeutung: `rsync -a quelle/ ziel/` kopiert den *Inhalt* von `quelle` nach `ziel`, während `rsync -a quelle ziel/` das Verzeichnis `quelle` *selbst* in `ziel` anlegt. Zweitens kann `--delete` Daten vernichten, wenn Quelle und Ziel verwechselt werden; deshalb läuft man *immer zuerst mit* `-n`. Diese Eigenschaften beruhen auf Fachwissen; prüfen Sie sie in `man rsync`, bevor Sie sie auf echten Daten einsetzen.

Die Rückgabecodes von `rsync` sind reichhaltig. Die Quelltextdatei `errcode.h` aus dem rsync-Paket (siehe Ergänzung) nennt unter anderem 0 (Erfolg), 1 (Syntax- oder Aufruffehler), 23 (Teilweise Übertragung), 24 (Dateien auf der Senderseite verschwunden) und 30 (Zeitüberschreitung). Besonders der Wert 24 ist im Betrieb interessant: Während einer Sicherung verschwinden Dateien, etwa Zwischenspeicher oder rotierte Logs, und `rsync` meldet das. Das ist meist harmlos und verdient eine Warnung, aber keinen Alarm; der Code 23 dagegen deutet auf echte Probleme hin. Das folgende Programm baut den Befehl *ohne ihn auszuführen* und wertet Rückgabecodes aus. Wir führen `rsync` hier absichtlich nicht aus, weil es in der Testumgebung nicht installiert war; die Befehlsliste und die Auswertung sind dagegen reiner Python-Code und wurden getestet.

```python
RSYNC_TEXTE = {
    0: "Erfolg",
    1: "Syntax- oder Aufruffehler",
    23: "Teilweise Übertragung",
    24: "Dateien auf der Senderseite verschwunden",
    30: "Zeitüberschreitung beim Senden oder Empfangen",
}

def rsync_befehl(quelle, ziel, loeschen=False, probelauf=True, ausschluesse=()):
    """Baut die rsync-Befehlsliste. Die Quelle bekommt einen Schrägstrich am Ende."""
    befehl = ["rsync", "-a"]
    if probelauf:
        befehl.append("--dry-run")
    if loeschen:
        befehl.append("--delete")
    befehl += [f"--exclude={muster}" for muster in ausschluesse]
    befehl += [str(quelle).rstrip("/") + "/", str(ziel)]
    return befehl

def bewerte_rsync(code):
    """Liefert (Status, Text): 'ok', 'warnung' oder 'fehler'."""
    text = RSYNC_TEXTE.get(code, f"unbekannter rsync-Code {code}")
    if code == 0:
        return "ok", text
    if code == 24:
        return "warnung", text
    return "fehler", text

print(rsync_befehl("/srv/daten", "/mnt/backup/daten", ausschluesse=["*.tmp", ".cache"]))
print(rsync_befehl("/srv/daten/", "/mnt/backup/daten", loeschen=True, probelauf=False))
for code in (0, 24, 23, 12):
    print(code, bewerte_rsync(code))
```

```text
['rsync', '-a', '--dry-run', '--exclude=*.tmp', '--exclude=.cache', '/srv/daten/', '/mnt/backup/daten']
['rsync', '-a', '--delete', '/srv/daten/', '/mnt/backup/daten']
0 ('ok', 'Erfolg')
24 ('warnung', 'Dateien auf der Senderseite verschwunden')
23 ('fehler', 'Teilweise Übertragung')
12 ('fehler', 'unbekannter rsync-Code 12')
```

Die Funktion `rsync_befehl` stellt sicher, dass die Quelle immer mit Schrägstrich endet (auch wenn jemand ihn vergessen hat) und dass der Probelauf der *Standard* ist; wer wirklich synchronisieren will, muss `probelauf=False` angeben. Das ist derselbe Schutzgedanke wie in Modul 4. Die Funktion `bewerte_rsync` übersetzt Codes in einen Status; für jeden unbekannten Code ist die Antwort „fehler“, was die sichere Voreinstellung ist.

> **Ergänzung aus der Nachrecherche**
>
> **Gefahr beim Entpacken: tarfile-Filter (PEP 706, abgerufen).** Beim Entpacken eines Archivs aus unbekannter Quelle können Dateien außerhalb des Zielverzeichnisses überschrieben werden. PEP 706 beschreibt dieses Problem (mit Verweis auf die Sicherheitslücke CVE-2007-4559) und führt für `extractall()` einen Parameter `filter` ein. Die benannten Filter sind `'fully_trusted'` (keine Prüfung), `'tar'` (ähnlich den Voreinstellungen von GNU `tar`) und `'data'` (die strengste Variante, die unter anderem Verweise nach außerhalb und Gerätedateien verweigert). Laut PEP erscheint in Python 3.12 und 3.13 ohne Angabe eines Filters eine `DeprecationWarning`, und ab Python 3.14 ist `'data'` die Voreinstellung. Der PEP sieht auch eine Nachrüstung in älteren Python-Versionen vor; dort bleibt die Voreinstellung `'fully_trusted'`. Ob Ihre Python-Version den Parameter kennt, prüfen Sie mit `hasattr(tarfile, "data_filter")`. Geben Sie den Filter deshalb immer ausdrücklich an. Quelle: <https://peps.python.org/pep-0706/>.
>
> **rsync-Rückgabecodes (abgerufen: Quelltextdatei `errcode.h`).** Abgerufen wurde nicht die Handbuchseite, sondern die Datei `errcode.h` aus dem Quelltext des rsync-Pakets in einem Spiegel einer CentOS-Quellcodeverwaltung. Sie definiert unter anderem 0 (RERR_OK), 1 (Syntax- oder Aufruffehler), 2 (Protokollfehler), 3 (Fehler beim Auswählen von Dateien oder Verzeichnissen), 10 und 11 (Socket- und Datei-Ein-/Ausgabefehler), 20 (Signal SIGINT, SIGTERM oder SIGHUP empfangen), 23 (teilweise Übertragung), 24 (Dateien auf der Senderseite verschwunden), 25 (Löschungen wegen `--max-delete` übersprungen), 30 (Zeitüberschreitung bei der Datenübertragung) und 35 (Zeitüberschreitung beim Warten auf einen Daemon). Die Datei stammt aus einer älteren Paketversion; aktuelle Versionen können weitere Codes kennen.
>
> **Backups und Datenschutz (keine Rechtsberatung).** Sicherungen enthalten oft personenbezogene Daten, für die Aufbewahrungs- und Löschfristen gelten können. Wir haben dazu den Baustein „Aufbewahren“ des Standard-Datenschutzmodells (SDM, Landesbeauftragter für Datenschutz Mecklenburg-Vorpommern) gelesen. Er behandelt Aufbewahrung und Löschung nach Fristen, *schließt Datensicherungen (Backups) aber ausdrücklich aus* und enthält keine Aussage dazu, wie mit personenbezogenen Daten in Backups umzugehen ist. Wir können deshalb hier keine rechtliche Vorgabe nennen. Klären Sie Aufbewahrungsdauer, Verschlüsselung und Löschung von Sicherungen mit Ihrem Datenschutzbeauftragten; dieses Skript ist keine Rechtsberatung. Quelle: <https://www.datenschutz-mv.de/static/DS/Dateien/Datenschutzmodell/Bausteine/SDM-V2.0_Aufbewahren_V1.0.pdf>.

Typische Fehler bei Sicherungsskripten zeigt das nächste Beispiel in kleinen Ausschnitten. Erstens: Fehlt `arcname`, legt `tarfile` den Pfad so ab, wie er übergeben wurde (ohne den führenden Schrägstrich); das Archiv enthält dann ein tief verschachteltes Verzeichnis statt des gewünschten Namens, und beim Entpacken entsteht unerwartet `tmp/tmpabc123/daten`. Zweitens: Ein Archiv wird nie geöffnet und geprüft; ein beschädigtes oder leeres Archiv fällt erst beim Wiederherstellen auf. Drittens: Das Archiv wird im Quellverzeichnis abgelegt, das es sichern soll, sodass es sich selbst mit einpackt.

```python
import tarfile
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    quelle = Path(tmp) / "daten"
    quelle.mkdir()
    (quelle / "a.txt").write_text("x", encoding="utf-8")

    # FALSCH: ohne arcname steckt der ganze Pfad im Archiv
    with tarfile.open(Path(tmp) / "falsch.tar.gz", "w:gz") as tar:
        tar.add(quelle)
    with tarfile.open(Path(tmp) / "falsch.tar.gz", "r:gz") as tar:
        namen = tar.getnames()
    print("falsch:  ", namen[0] != "daten", namen[0].endswith("daten"))

    # RICHTIG: arcname legt den Namen im Archiv fest
    with tarfile.open(Path(tmp) / "richtig.tar.gz", "w:gz") as tar:
        tar.add(quelle, arcname="daten")
    with tarfile.open(Path(tmp) / "richtig.tar.gz", "r:gz") as tar:
        print("richtig: ", tar.getnames())

    # FALSCH: das Archiv nie prüfen. Ein beschädigtes Archiv fällt erst beim Lesen auf.
    kaputt = Path(tmp) / "kaputt.tar.gz"
    kaputt.write_bytes(b"das ist kein gzip-Archiv")
    try:
        with tarfile.open(kaputt, "r:gz") as tar:
            tar.getnames()
    except (tarfile.TarError, OSError) as fehler:
        print("Prüfung schlägt an:", type(fehler).__name__, "-", fehler)
```

```text
falsch:   True True
richtig:  ['daten', 'daten/a.txt']
Prüfung schlägt an: ReadError - not a gzip file
```

Die erste Zeile zeigt, dass der Name im Archiv nicht `daten` ist, aber auf `daten` endet, also einen langen Pfad davor hat. Mit `arcname="daten"` entsteht das erwartete Archiv. Die letzte Zeile ist die Grundlage der Prüfung, die wir in der nächsten Lerneinheit fest in das Skript einbauen: Das frisch geschriebene Archiv wird wieder geöffnet und gelesen, und ein Fehler führt zu einem eigenen Rückgabecode.

> **Merke**
>
> Ein Backup besteht aus Erzeugen, Prüfen und Aufräumen. `tarfile` schreibt und liest `.tar.gz`-Archive ohne externes Programm; `arcname` legt den Namen im Archiv fest, `filter` schließt Dateien aus, `extractall(..., filter="data")` schützt beim Entpacken. `rsync` synchronisiert Verzeichnisse; zuerst immer mit `--dry-run`. Ein Code ungleich 0 von `tar` oder `rsync` muss differenziert gelesen werden.

> **Praxisbeispiel**
>
> Auf Servern finden Sie meist beide Werkzeuge nebeneinander: Ein nächtlicher `tar`-Job packt Konfigurationen und Anwendungsdaten zu datierten Archiven, ein `rsync`-Job spiegelt die Archive anschließend auf einen anderen Rechner. Datenbanken sichert man nicht durch Kopieren der laufenden Dateien, sondern mit ihrem eigenen Werkzeug (zum Beispiel `pg_dump` oder `mysqldump`); das Ergebnis wird dann mit `tar` oder `rsync` weiterverarbeitet. Genau dieses Zusammenspiel baut sich ein Skript, das Sie in der nächsten Lerneinheit fertigstellen.

## 6.5 Backup-Skript Teil 2 (Rotation) + Quiz

Ein Sicherungsskript, das täglich läuft, erzeugt täglich eine neue Datei. Ohne Aufräumen füllt sich das Ziel nach einigen Wochen. Die Antwort heißt *Rotation* (englisch auch *retention*): Man behält die neuesten N Sicherungen und löscht die älteren. Das ist riskant, weil das Skript dabei *löscht*. Zwei Schutzmaßnahmen sind Pflicht. Erstens darf die Rotation nur Dateien anfassen, die zweifelsfrei von diesem Skript stammen: Der Name muss genau dem Muster `NAME-JJJJMMTT-HHMMSS.tar.gz` entsprechen. Zweitens braucht auch das Löschen einen Probelauf (`--dry-run`). Eine häufige Variante ist eine gestaffelte Aufbewahrung („täglich sieben, wöchentlich vier, monatlich zwölf“); wir beschränken uns auf die einfache Variante „die letzten N“.

Der Zeitstempel im Dateinamen hat ein praktisches Merkmal: Mit der Form Jahr-Monat-Tag-Stunde-Minute-Sekunde sortieren die Namen alphabetisch genauso wie zeitlich. Zum Erzeugen verwendet man das Modul `datetime` (das in Modul 8 vertieft wird); hier genügt: `jetzt.strftime("%Y%m%d-%H%M%S")` formt ein Datum in Text um, und `datetime.strptime(text, format)` liest den Text wieder als Datum. Wir verwenden `strptime` auch, um zu prüfen, ob ein Dateiname wirklich einen gültigen Zeitstempel trägt.

Das vollständige Programm folgt. Es verbindet alles aus diesem Modul: eigene Ausnahmen mit Rückgabecodes (Lerneinheit 6.2), Protokollierung mit `logging` (6.3), `tarfile` mit Filter und Prüfung (6.4) und die Rotation. Es schreibt das Archiv zunächst unter einem vorläufigen Namen (`.part`) und benennt es erst nach dem erfolgreichen Schreiben um. So liegt nie ein halbfertiges Archiv unter dem endgültigen Namen, das die Rotation für gültig halten könnte. Speichern Sie es als `backup.py`. Wie in Lerneinheit 5.5 hängt am Ende ein Aufrufschutz, sodass man es auch als Modul importieren kann; ohne Argumente beendet es sich mit dem Code 2 und einer Meldung von `argparse`.

```python
import argparse
import fnmatch
import logging
import sys
import tarfile
from datetime import datetime
from pathlib import Path

log = logging.getLogger("backup")
SUFFIX = ".tar.gz"
ZEITFORMAT = "%Y%m%d-%H%M%S"


class SicherungFehler(Exception):
    """Basisklasse für erwartete Fehler; trägt den Rückgabecode."""
    exit_code = 1


class QuelleFehler(SicherungFehler):
    exit_code = 10


class ZielFehler(SicherungFehler):
    exit_code = 11


class PruefFehler(SicherungFehler):
    exit_code = 12


def archivname(name, jetzt):
    return f"{name}-{jetzt.strftime(ZEITFORMAT)}{SUFFIX}"


def ist_sicherung(datei, name):
    """True, wenn der Dateiname exakt NAME-JJJJMMTT-HHMMSS.tar.gz lautet."""
    praefix = f"{name}-"
    if not (datei.name.startswith(praefix) and datei.name.endswith(SUFFIX)):
        return False
    zeit = datei.name[len(praefix):-len(SUFFIX)]
    try:
        datetime.strptime(zeit, ZEITFORMAT)
    except ValueError:
        return False
    return True


def erstelle_archiv(quelle, archiv, ausschluesse):
    """Packt quelle nach archiv (über eine .part-Datei). Liefert die Zahl der Einträge."""
    eintraege = 0

    def filtere(info):
        nonlocal eintraege
        if any(fnmatch.fnmatch(Path(info.name).name, m) for m in ausschluesse):
            log.debug("ausgeschlossen: %s", info.name)
            return None
        eintraege += 1
        return info

    vorlaeufig = archiv.with_name(archiv.name + ".part")
    try:
        with tarfile.open(vorlaeufig, "w:gz") as tar:
            tar.add(quelle, arcname=quelle.name, filter=filtere)
        vorlaeufig.replace(archiv)
    except OSError as ursache:
        vorlaeufig.unlink(missing_ok=True)
        raise ZielFehler(f"Archiv konnte nicht geschrieben werden: {ursache}") from ursache
    return eintraege


def pruefe_archiv(archiv, erwartet):
    """Öffnet das Archiv und prüft, ob alle Einträge lesbar vorhanden sind."""
    try:
        with tarfile.open(archiv, "r:gz") as tar:
            vorhanden = len(tar.getnames())
    except (tarfile.TarError, OSError) as ursache:
        raise PruefFehler(f"Archiv nicht lesbar: {archiv.name}") from ursache
    if vorhanden != erwartet:
        raise PruefFehler(f"Archiv unvollständig: {vorhanden} statt {erwartet} Einträge")


def rotiere(zielverzeichnis, name, behalten, probelauf):
    """Behält die neuesten 'behalten' Sicherungen und löscht die übrigen."""
    sicherungen = sorted(p for p in zielverzeichnis.iterdir() if ist_sicherung(p, name))
    zu_loeschen = sicherungen[:-behalten] if behalten > 0 else sicherungen
    for datei in zu_loeschen:
        if probelauf:
            log.info("Probelauf: würde löschen: %s", datei.name)
        else:
            datei.unlink()
            log.info("gelöscht: %s", datei.name)
    return [p.name for p in zu_loeschen]


def baue_parser():
    parser = argparse.ArgumentParser(
        prog="backup",
        description="Sichert ein Verzeichnis als .tar.gz und behält die neuesten Sicherungen. "
                    "Rückgabecodes: 0 OK, 10 Quelle, 11 Ziel, 12 Prüfung, 130 Abbruch.")
    parser.add_argument("quelle", help="Verzeichnis, das gesichert wird")
    parser.add_argument("ziel", help="Verzeichnis für die Archive")
    parser.add_argument("--name", help="Namensbestandteil (Standard: Name der Quelle)")
    parser.add_argument("--behalten", type=int, default=7, metavar="N",
                        help="Zahl der aufzubewahrenden Sicherungen (Standard: %(default)s)")
    parser.add_argument("--ausschluss", action="append", default=[], metavar="MUSTER",
                        help="Dateinamen-Muster, die ausgelassen werden (mehrfach möglich)")
    parser.add_argument("--dry-run", action="store_true", help="nichts schreiben oder löschen")
    parser.add_argument("-v", "--verbose", action="store_true", help="auch Details protokollieren")
    return parser


def main(argv=None, jetzt=None):
    parser = baue_parser()
    args = parser.parse_args(argv)
    if args.behalten < 1:
        parser.error("--behalten muss mindestens 1 sein")
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO,
                        format="%(levelname)-7s %(message)s", stream=sys.stdout, force=True)
    jetzt = jetzt or datetime.now()
    quelle, ziel = Path(args.quelle), Path(args.ziel)
    name = args.name or quelle.name
    try:
        if not quelle.is_dir():
            raise QuelleFehler(f"Quelle ist kein Verzeichnis: {quelle}")
        if ziel.resolve().is_relative_to(quelle.resolve()):
            raise ZielFehler("Das Ziel darf nicht innerhalb der Quelle liegen")
        archiv = ziel / archivname(name, jetzt)
        if args.dry_run:
            log.info("Probelauf: würde %s anlegen", archiv.name)
        else:
            ziel.mkdir(parents=True, exist_ok=True)
            eintraege = erstelle_archiv(quelle, archiv, args.ausschluss)
            pruefe_archiv(archiv, eintraege)
            log.info("Archiv %s angelegt und geprüft (%d Einträge)", archiv.name, eintraege)
        if ziel.is_dir():
            rotiere(ziel, name, args.behalten, args.dry_run)
    except SicherungFehler as fehler:
        log.error("%s", fehler)
        return fehler.exit_code
    except KeyboardInterrupt:
        log.warning("Abbruch durch den Benutzer")
        return 130
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

```text
usage: backup [-h] [--name NAME] [--behalten N] [--ausschluss MUSTER]
              [--dry-run] [-v]
              quelle ziel
backup: error: the following arguments are required: quelle, ziel
```

Das Programm ist das längste dieses Skripts; es lohnt sich, es Funktion für Funktion zu lesen. `archivname` und `ist_sicherung` sind reine Funktionen für den Dateinamen. In `erstelle_archiv` zählt die Filterfunktion (eine *Closure*, die auf die Variable `eintraege` der äußeren Funktion zugreift; `nonlocal` macht das möglich) die aufgenommenen Einträge und lässt Muster aus der Option `--ausschluss` weg. Fehler beim Schreiben (Platte voll, keine Berechtigung) werden als `ZielFehler` weitergegeben, die Ursache bleibt über `from` erhalten, und die halbfertige `.part`-Datei wird entfernt. `pruefe_archiv` öffnet das Ergebnis und vergleicht die Zahl der Einträge mit der erwarteten. `rotiere` sortiert die passenden Dateien (neueste zuletzt), behält die letzten `behalten` und löscht den Rest; im Probelauf meldet sie nur. In `main` wird alles zusammengeführt: Die Prüfung `ziel.resolve().is_relative_to(quelle.resolve())` verhindert, dass das Archiv im Quellverzeichnis landet, und der `try`-Block übersetzt die eigenen Ausnahmen in Meldung und Rückgabecode. Der Parameter `jetzt` von `main` erlaubt es, in Tests eine feste Zeit vorzugeben, damit Dateinamen vorhersagbar sind; im Normalbetrieb ist er `None`, und es gilt `datetime.now()`.

Das Testprogramm unten legt in einem temporären Verzeichnis Daten an, ruft das Sicherungsprogramm mit künstlich fortlaufenden Zeiten fünfmal auf (mit `--behalten 3`) und zeigt, was übrig bleibt. Es prüft außerdem den Probelauf, die Fehlerfälle und einen Fremdling im Zielverzeichnis. Speichern Sie es neben `backup.py`. Damit die Ausgabe nicht vom temporären Pfad abhängt, ersetzt das Programm ihn durch `<tmp>`.

```python
import contextlib
import io
import tempfile
from datetime import datetime
from pathlib import Path

import backup

def lauf(argv, jetzt, tmp):
    puffer = io.StringIO()
    with contextlib.redirect_stdout(puffer):
        code = backup.main(argv, jetzt=jetzt)
    return code, puffer.getvalue().replace(tmp, "<tmp>").rstrip()

def zeige(ziel):
    print("  Ziel enthält:", sorted(p.name for p in ziel.iterdir()))

with tempfile.TemporaryDirectory() as tmp:
    quelle = Path(tmp) / "daten"
    ziel = Path(tmp) / "sicherungen"
    (quelle / "unter").mkdir(parents=True)
    (quelle / "a.txt").write_text("Alpha\n", encoding="utf-8")
    (quelle / "unter" / "b.txt").write_text("Beta\n", encoding="utf-8")
    (quelle / "cache.tmp").write_text("flüchtig", encoding="utf-8")

    print("== Fünf Läufe mit --behalten 3 ==")
    ziel.mkdir()
    (ziel / "daten-notizen.tar.gz").write_text("Fremdling")     # darf nie gelöscht werden
    for tag in range(1, 6):
        jetzt = datetime(2026, 12, tag, 3, 0, 0)
        code, text = lauf([str(quelle), str(ziel), "--behalten", "3",
                           "--ausschluss", "*.tmp"], jetzt, tmp)
        print(f"Lauf {tag}: Code {code}")
    print(text)
    zeige(ziel)

    print("== Probelauf ==")
    code, text = lauf([str(quelle), str(ziel), "--behalten", "1", "--dry-run"],
                      datetime(2026, 12, 6, 3, 0, 0), tmp)
    print(code)
    print(text)
    zeige(ziel)

    print("== Fehlerfälle ==")
    code, text = lauf([str(Path(tmp) / "fehlt"), str(ziel)], datetime(2026, 12, 7), tmp)
    print(code, text)
    code, text = lauf([str(quelle), str(quelle / "unter")], datetime(2026, 12, 7), tmp)
    print(code, text)
```

```text
== Fünf Läufe mit --behalten 3 ==
Lauf 1: Code 0
Lauf 2: Code 0
Lauf 3: Code 0
Lauf 4: Code 0
Lauf 5: Code 0
INFO    Archiv daten-20261205-030000.tar.gz angelegt und geprüft (4 Einträge)
INFO    gelöscht: daten-20261202-030000.tar.gz
  Ziel enthält: ['daten-20261203-030000.tar.gz', 'daten-20261204-030000.tar.gz', 'daten-20261205-030000.tar.gz', 'daten-notizen.tar.gz']
== Probelauf ==
0
INFO    Probelauf: würde daten-20261206-030000.tar.gz anlegen
INFO    Probelauf: würde löschen: daten-20261203-030000.tar.gz
INFO    Probelauf: würde löschen: daten-20261204-030000.tar.gz
  Ziel enthält: ['daten-20261203-030000.tar.gz', 'daten-20261204-030000.tar.gz', 'daten-20261205-030000.tar.gz', 'daten-notizen.tar.gz']
== Fehlerfälle ==
10 ERROR   Quelle ist kein Verzeichnis: <tmp>/fehlt
11 ERROR   Das Ziel darf nicht innerhalb der Quelle liegen
```

Nach fünf Läufen sind genau die drei neuesten Archive übrig; die beiden ältesten wurden gelöscht, und die Datei `daten-notizen.tar.gz`, die dem Namensmuster nicht entspricht, blieb unangetastet. Im Probelauf mit `--behalten 1` erscheinen die Löschmeldungen, aber das Zielverzeichnis bleibt unverändert, und es wird kein neues Archiv angelegt. Die beiden Fehlerfälle liefern die dokumentierten Rückgabecodes 10 und 11 und eine verständliche Meldung. Die vier Einträge im ersten Lauf sind das Verzeichnis `daten`, das Unterverzeichnis `unter` und die beiden Textdateien (die Datei `cache.tmp` wurde ausgeschlossen).

Zu den typischen Fehlern bei der Rotation gehört ein zu weit gefasstes Löschmuster: Wer einfach `zielverzeichnis.glob("*.tar.gz")` nimmt, löscht auch Archive anderer Skripte und Hand-Sicherungen. Ebenso gefährlich ist es, nach dem *Änderungsdatum* statt nach dem Namen zu sortieren, wenn Dateien kopiert oder wiederhergestellt wurden (dann sind alte Archive plötzlich „neu“). Und der Wert `behalten=0` würde alle Sicherungen löschen, einschließlich der soeben erstellten; das Programm verbietet ihn deshalb schon beim Aufruf. Die Gegenüberstellung zeigt das als Kurzbeispiel.

```python
import tempfile
from datetime import datetime
from pathlib import Path

def ist_sicherung(datei, name):
    praefix, suffix = f"{name}-", ".tar.gz"
    if not (datei.name.startswith(praefix) and datei.name.endswith(suffix)):
        return False
    try:
        datetime.strptime(datei.name[len(praefix):-len(suffix)], "%Y%m%d-%H%M%S")
    except ValueError:
        return False
    return True

with tempfile.TemporaryDirectory() as tmp:
    for name in ["daten-20261201-030000.tar.gz", "daten-20261202-030000.tar.gz",
                 "daten-notizen.tar.gz", "andere-20261201-030000.tar.gz", "daten.tar.gz"]:
        (Path(tmp) / name).write_text("x")

    alle = sorted(p.name for p in Path(tmp).glob("*.tar.gz"))
    print("FALSCH, Muster *.tar.gz trifft:", len(alle), "Dateien")

    eigene = sorted(p.name for p in Path(tmp).iterdir() if ist_sicherung(p, "daten"))
    print("RICHTIG, nur eigene Archive:   ", eigene)

    liste = [1, 2, 3]
    print("liste[:-3] =", liste[:-3])
    print("liste[:-0] =", liste[:-0])
```

```text
FALSCH, Muster *.tar.gz trifft: 5 Dateien
RICHTIG, nur eigene Archive:    ['daten-20261201-030000.tar.gz', 'daten-20261202-030000.tar.gz']
liste[:-3] = []
liste[:-0] = []
```

Das Beispiel enthält eine Feinheit, die in `rotiere` berücksichtigt ist: Bei drei Sicherungen und `behalten=3` bleibt die Löschliste leer, das ist richtig. Dagegen ist `liste[:-0]` *nicht* „alles löschen“, sondern ebenfalls leer, weil `-0` gleich `0` ist und die Schreibweise `[:0]` bedeutet. Wer die Rotation naiv mit `sicherungen[:-behalten]` schreibt, hat bei `behalten=0` das gegenteilige Verhalten von dem, was man erwartet. Deshalb behandelt `rotiere` den Fall `behalten > 0` gesondert, und `main` lehnt Werte unter 1 ab. Solche Randfälle sind ein guter Grund, jede Löschfunktion mit Testdaten zu prüfen.

> **Ergänzung aus der Nachrecherche**
>
> **Offene Punkte beim Sicherungsskript.** Das Skript sichert Dateien, aber nicht Rechte und Besitzer in jedem Fall (`tarfile` speichert sie, das Wiederherstellen als normaler Benutzer kann sie aber nicht setzen), keine laufenden Datenbanken (siehe Praxisbox oben), keine Verschlüsselung und keine Übertragung an einen zweiten Ort. Es nutzt auch keine inkrementelle Sicherung. Für produktive Umgebungen gibt es spezialisierte Werkzeuge (zum Beispiel `borg` oder `restic`); der Wert dieser Lerneinheit liegt im Verständnis der Bausteine. Die Aussagen zu den Werkzeugen `borg` und `restic` beruhen auf Fachwissen und wurden hier nicht nachgeschlagen.

Zum Abschluss des Moduls dient wieder ein Quiz: die Übungsaufgaben 1 bis 4. Die Aufgaben 5 bis 8 verlangen eigene Programme, darunter eine Erweiterung des Sicherungsskripts um eine gestaffelte Rotation und ein Programm, das eine Sicherung testweise wiederherstellt und mit dem Original vergleicht.

> **Merke**
>
> Rotation ist eine Löschoperation und braucht dieselben Schutzmaßnahmen wie jede andere: ein enges Namensmuster, einen Probelauf, keine unsinnigen Parameter, Tests mit Fremddateien. Schreiben Sie Archive unter einem vorläufigen Namen und benennen Sie sie erst nach dem Prüfen um; ein Archiv ohne Prüfung ist kein Backup.

> **Praxisbeispiel**
>
> Das fertige Skript lässt sich direkt in einen Cronjob oder einen systemd-Timer einbauen (Modul 9): `backup.py /srv/daten /mnt/backup --behalten 14` jede Nacht, Ausgabe im Journal, Rückgabecode 0 bei Erfolg. Dass jeder Fehlerfall einen eigenen Code hat, zahlt sich im Monitoring aus: Code 11 weist auf ein volles oder fehlendes Zielmedium, Code 12 auf ein beschädigtes Archiv, Code 10 auf einen falschen Pfad. In Modul 12 wird aus einem Code ungleich 0 eine Alarmmeldung per Mail.

## Übungsaufgaben zu Modul 06

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Arbeiten Sie bei den Programmieraufgaben in einem temporären Verzeichnis.

**Aufgabe 1.** In welcher Reihenfolge laufen `try`, `except`, `else` und `finally`, wenn (a) kein Fehler auftritt, (b) ein passender Fehler auftritt, (c) ein nicht passender Fehler auftritt? Warum ist ein nacktes `except:` gefährlich, und warum fängt `except Exception` ein `KeyboardInterrupt` nicht? Wozu dient `raise ... from ursache`?

**Aufgabe 2.** Welche Rückgabecodes gibt ein Python-Skript in diesen Fällen zurück: Ende ohne `sys.exit`, `sys.exit("Fehler")`, nicht abgefangene Ausnahme, `sys.exit(256)`, Strg+C ohne Behandlung beziehungsweise mit `sys.exit(130)`? Auf welchen Kanal gehören Fehlermeldungen? An welcher Stelle eines Programms sollte `sys.exit` aufgerufen werden?

**Aufgabe 3.** Nennen Sie die fünf Logstufen mit ihren Zahlenwerten und die Voreinstellung. Wozu dienen Logger, Handler und Formatter? Warum schreibt man `log.info("Datei %s", name)` statt eines f-Strings? Wann verwendet man `log.exception`? Warum legt das Sicherungsskript Archive zuerst als `.part`-Datei an, und warum prüft es das Archiv nach dem Schreiben?

**Aufgabe 4.** Finden Sie mindestens fünf Fehler im folgenden Programm und erklären Sie jeden.

```python
import logging

logging.info("Start")
logging.basicConfig(level=logging.INFO)

try:
    zahl = int(input("Zahl: "))
except:
    pass

logging.error(f"Fehler: {e}")
print("Fehler aufgetreten")
sys.exit(256)
```

**Aufgabe 5.** Schreiben Sie eine Funktion `lies_zahl(pfad)`, die eine ganze Zahl aus einer Textdatei liest. Fehlt die Datei oder ist der Inhalt keine Zahl, soll sie eine eigene Ausnahme `DateiFehler` auslösen (mit erhaltener Ursache). Schreiben Sie dazu ein `main`, das `DateiFehler` abfängt, die Meldung auf `stderr` ausgibt und die Rückgabecodes 0 (Erfolg), 10 (Datei fehlt) und 11 (Inhalt ungültig) liefert. Testen Sie alle drei Fälle.

**Aufgabe 6.** Schreiben Sie eine Funktion `richte_logging_ein(logdatei, ausfuehrlich)`, die die Konsole mit `INFO` (oder mit `DEBUG`, wenn `ausfuehrlich` wahr ist) und eine Datei immer mit `DEBUG` protokolliert. Schreiben Sie ein kleines Programm, das eine Liste von Werten verarbeitet, ungültige Werte als Warnung protokolliert und eine Division durch null mit `log.exception` meldet. Zeigen Sie den Inhalt der Logdatei.

**Aufgabe 7.** Erweitern Sie die Rotation um eine gestaffelte Aufbewahrung: Eine Funktion `waehle_zu_behalten(namen, taeglich, monatlich)` bekommt eine Liste von Archivnamen der Form `daten-JJJJMMTT-HHMMSS.tar.gz` und liefert die Namen, die *behalten* werden: die neuesten `taeglich` Archive und zusätzlich je das neueste Archiv der letzten `monatlich` Monate, aus denen Archive vorliegen. Testen Sie mit 40 täglichen Archiven.

**Aufgabe 8.** Schreiben Sie ein Programm, das ein Sicherungsarchiv testweise wiederherstellt und mit dem Original vergleicht. Es soll eine Liste der Unterschiede zurückgeben (fehlende Dateien, zusätzliche Dateien, abweichender Inhalt, auch in Unterverzeichnissen). Zeigen Sie, dass `filecmp.dircmp` allein eine Änderung übersieht, bei der Größe und Zeitstempel einer Datei gleich bleiben, und wie Ihr Programm sie trotzdem findet.

## Lösungen zu Modul 06

**Aufgabe 1.** (a) Ohne Fehler: `try`, dann `else`, dann `finally`. (b) Bei einem passenden Fehler: `try` bis zur fehlerhaften Zeile, dann der passende `except`-Block, dann `finally`; `else` entfällt. (c) Bei einem nicht passenden Fehler: `try` bis zur fehlerhaften Zeile, dann `finally`, anschließend wird die Ausnahme an den Aufrufer weitergereicht. Ein nacktes `except:` fängt auch `KeyboardInterrupt` und `SystemExit`; das Skript lässt sich nicht mehr abbrechen, und Programmierfehler werden als erwartete Fehler behandelt. `KeyboardInterrupt` erbt von `BaseException`, nicht von `Exception`; `except Exception` fängt nur Klassen unterhalb von `Exception`. Mit `raise Neu(...) from ursache` bleibt die Ursache im Attribut `__cause__` erhalten, und der Traceback zeigt beide Ausnahmen.

**Aufgabe 2.** Ende ohne `sys.exit`: 0. `sys.exit("Fehler")`: Text auf `stderr`, Code 1. Nicht abgefangene Ausnahme: Traceback auf `stderr`, Code 1. `sys.exit(256)`: Code 0, weil Rückgabecodes nur ein Byte lang sind (modulo 256). Strg+C ohne Behandlung: in Python `-2` (Signal 2), in der Shell 130; mit `except KeyboardInterrupt: sys.exit(130)`: 130. Fehlermeldungen gehören auf `stderr`. `sys.exit` ruft man nur auf der obersten Ebene auf (`sys.exit(main())`); Hilfsfunktionen lösen Ausnahmen aus, sonst lassen sie sich nicht testen.

**Aufgabe 3.** `DEBUG` 10, `INFO` 20, `WARNING` 30, `ERROR` 40, `CRITICAL` 50; Voreinstellung `WARNING`. Der Logger nimmt Meldungen entgegen und kennt eine Stufe, der Handler bestimmt das Ziel (Konsole, Datei), der Formatter das Aussehen der Zeile. Mit Argumenten statt f-String wird der Text nur gebaut, wenn die Meldung wirklich ausgegeben wird. `log.exception` gehört in einen `except`-Block und hängt den Traceback an. Die `.part`-Datei verhindert, dass ein halbfertiges Archiv unter dem endgültigen Namen liegt (und von der Rotation als gültig gezählt wird); die Prüfung stellt sicher, dass das Archiv lesbar und vollständig ist, bevor ältere Sicherungen gelöscht werden.

**Aufgabe 4.** (1) `logging.info("Start")` vor `basicConfig` richtet automatisch eine Standardkonfiguration ein; das spätere `basicConfig` bleibt wirkungslos (Abhilfe: `basicConfig` zuerst oder `force=True`), und die Meldung `INFO` wird wegen der Standardstufe `WARNING` nicht angezeigt. (2) Das nackte `except:` fängt alles. (3) `pass` verschluckt den Fehler; der Benutzer erfährt nicht, dass die Eingabe ungültig war. (4) Die Variable `e` ist nie definiert (`except ... as e` fehlt), das führt zu einem `NameError`. (5) Der f-String im `logging`-Aufruf wird immer ausgewertet; besser `log.error("Fehler: %s", fehler)`. (6) `print` schreibt die Fehlermeldung auf `stdout` statt `logging` oder `stderr`. (7) `import sys` fehlt. (8) `sys.exit(256)` ergibt den Code 0 und täuscht Erfolg vor; außerdem wird `sys.exit` aufgerufen, obwohl vorher gar kein Fehlerfall unterschieden wurde.

**Aufgabe 5.**

```python
import contextlib
import sys
import tempfile
from pathlib import Path

class DateiFehler(Exception):
    exit_code = 1

class DateiFehltFehler(DateiFehler):
    exit_code = 10

class InhaltFehler(DateiFehler):
    exit_code = 11

def lies_zahl(pfad):
    try:
        text = Path(pfad).read_text(encoding="utf-8")
    except FileNotFoundError as ursache:
        raise DateiFehltFehler(f"Datei fehlt: {Path(pfad).name}") from ursache
    try:
        return int(text.strip())
    except ValueError as ursache:
        raise InhaltFehler(f"Keine Zahl in {Path(pfad).name}: {text.strip()!r}") from ursache

def main(pfad):
    try:
        print("Zahl:", lies_zahl(pfad))
    except DateiFehler as fehler:
        print(f"Fehler: {fehler}", file=sys.stderr)
        return fehler.exit_code
    return 0

with tempfile.TemporaryDirectory() as tmp, contextlib.redirect_stderr(sys.stdout):
    gut = Path(tmp) / "gut.txt"
    gut.write_text("42\n", encoding="utf-8")
    schlecht = Path(tmp) / "schlecht.txt"
    schlecht.write_text("viel", encoding="utf-8")
    for datei in (gut, Path(tmp) / "fehlt.txt", schlecht):
        print("->", main(datei))
```

```text
Zahl: 42
-> 0
Fehler: Datei fehlt: fehlt.txt
-> 10
Fehler: Keine Zahl in schlecht.txt: 'viel'
-> 11
```

Die eigene Ausnahme hat zwei Unterklassen mit je einem Rückgabecode, `main` fängt nur die Basisklasse. Die Ursache bleibt über `from` erhalten, die Meldung nennt nur den Dateinamen. Das Umleiten von `stderr` nach `stdout` dient nur der Anzeige in diesem Beispiel.

**Aufgabe 6.**

```python
import logging
import sys
import tempfile
from pathlib import Path

def richte_logging_ein(logdatei, ausfuehrlich):
    log = logging.getLogger("verarbeitung")
    log.setLevel(logging.DEBUG)
    log.handlers.clear()

    konsole = logging.StreamHandler(sys.stdout)
    konsole.setLevel(logging.DEBUG if ausfuehrlich else logging.INFO)
    konsole.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

    datei = logging.FileHandler(logdatei, encoding="utf-8")
    datei.setLevel(logging.DEBUG)
    datei.setFormatter(logging.Formatter("%(asctime)s %(levelname)-7s %(message)s"))

    log.addHandler(konsole)
    log.addHandler(datei)
    return log

def verarbeite(werte, log):
    for wert in werte:
        log.debug("prüfe %r", wert)
        try:
            zahl = int(wert)
        except ValueError:
            log.warning("ungültiger Wert übersprungen: %r", wert)
            continue
        try:
            log.info("100 / %d = %.1f", zahl, 100 / zahl)
        except ZeroDivisionError:
            log.exception("Division durch null bei %r", wert)

with tempfile.TemporaryDirectory() as tmp:
    logdatei = Path(tmp) / "lauf.log"
    log = richte_logging_ein(logdatei, ausfuehrlich=False)
    verarbeite(["4", "x", "0"], log)

    print("--- Logdatei (ohne Zeitstempel, ohne Traceback-Zeilen) ---")
    for zeile in logdatei.read_text(encoding="utf-8").splitlines():
        if zeile.startswith("20"):
            print(zeile[24:])
    for handler in list(log.handlers):
        handler.close()
        log.removeHandler(handler)
```

```text
INFO: 100 / 4 = 25.0
WARNING: ungültiger Wert übersprungen: 'x'
ERROR: Division durch null bei '0'
Traceback (most recent call last):
  File "beispiel.py", line 32, in verarbeite
    log.info("100 / %d = %.1f", zahl, 100 / zahl)
                                      ~~~~^~~~~~
ZeroDivisionError: division by zero
--- Logdatei (ohne Zeitstempel, ohne Traceback-Zeilen) ---
DEBUG   prüfe '4'
INFO    100 / 4 = 25.0
DEBUG   prüfe 'x'
WARNING ungültiger Wert übersprungen: 'x'
DEBUG   prüfe '0'
ERROR   Division durch null bei '0'
```

Die Konsole zeigt nur `INFO` und höher, die Datei alles. Dass die Datei zusätzlich den Traceback enthält, sieht man hier nicht, weil wir nur Zeilen ausgeben, die mit einem Zeitstempel beginnen. Die Division durch null im Argument des Logaufrufs wird *vor* dem Aufruf von `log.info` berechnet, deshalb ist die Fehlerzeile im Traceback genau die Logzeile. Mit `handlers.clear()` am Anfang vermeidet die Funktion doppelte Handler, falls sie mehrfach aufgerufen wird.

**Aufgabe 7.**

```python
from datetime import datetime, timedelta

def zeitpunkt(name):
    """Liest den Zeitstempel aus 'daten-JJJJMMTT-HHMMSS.tar.gz'."""
    stempel = name[len("daten-"):-len(".tar.gz")]
    return datetime.strptime(stempel, "%Y%m%d-%H%M%S")

def waehle_zu_behalten(namen, taeglich, monatlich):
    sortiert = sorted(namen, key=zeitpunkt)
    behalten = set(sortiert[-taeglich:]) if taeglich > 0 else set()
    neueste_je_monat = {}
    for name in sortiert:                       # aufsteigend, der letzte je Monat gewinnt
        z = zeitpunkt(name)
        neueste_je_monat[(z.year, z.month)] = name
    letzte_monate = sorted(neueste_je_monat)[-monatlich:] if monatlich > 0 else []
    behalten.update(neueste_je_monat[m] for m in letzte_monate)
    return sorted(behalten)

start = datetime(2026, 10, 1, 3, 0, 0)
namen = [f"daten-{(start + timedelta(days=i)).strftime('%Y%m%d-%H%M%S')}.tar.gz"
         for i in range(40)]
print(namen[0], "...", namen[-1])
for n in waehle_zu_behalten(namen, taeglich=3, monatlich=2):
    print(n)
print(len(waehle_zu_behalten(namen, taeglich=7, monatlich=0)))
```

```text
daten-20261001-030000.tar.gz ... daten-20261109-030000.tar.gz
daten-20261031-030000.tar.gz
daten-20261107-030000.tar.gz
daten-20261108-030000.tar.gz
daten-20261109-030000.tar.gz
7
```

Die 40 Archive laufen vom 1. Oktober bis zum 9. November. Behalten werden die letzten drei (7., 8. und 9. November), dazu das jeweils neueste Archiv der beiden Monate Oktober (31. Oktober) und November (das ist das schon vorhandene vom 9. November). Die Funktion arbeitet nur mit Namen und ist deshalb leicht zu testen; sie löscht nichts. Die eigentliche Löschung ergäbe sich als Differenz: alle passenden Namen ohne die Rückgabe dieser Funktion.

**Aufgabe 8.**

```python
import filecmp
import os
import tarfile
import tempfile
from pathlib import Path

def unterschiede(original, kopie, praefix=""):
    """Vergleicht zwei Verzeichnisbäume nach Namen UND Inhalt."""
    d = filecmp.dircmp(original, kopie)
    probleme = [f"fehlt in der Kopie: {praefix}{n}" for n in d.left_only]
    probleme += [f"zusätzlich in der Kopie: {praefix}{n}" for n in d.right_only]
    _, abweichend, _ = filecmp.cmpfiles(original, kopie, d.common_files, shallow=False)
    probleme += [f"Inhalt verschieden: {praefix}{n}" for n in abweichend]
    for n in d.common_dirs:
        probleme += unterschiede(Path(original) / n, Path(kopie) / n, f"{praefix}{n}/")
    return sorted(probleme)

def teste_wiederherstellung(archiv, original):
    with tempfile.TemporaryDirectory() as ziel:
        with tarfile.open(archiv, "r:gz") as tar:
            tar.extractall(ziel, filter="data")
        return unterschiede(original, Path(ziel) / Path(original).name)

with tempfile.TemporaryDirectory() as tmp:
    quelle = Path(tmp) / "daten"
    (quelle / "unter").mkdir(parents=True)
    (quelle / "a.txt").write_text("Alpha", encoding="utf-8")
    (quelle / "unter" / "b.txt").write_text("Beta", encoding="utf-8")
    archiv = Path(tmp) / "s.tar.gz"
    with tarfile.open(archiv, "w:gz") as tar:
        tar.add(quelle, arcname="daten")

    print("unverändert:", teste_wiederherstellung(archiv, quelle))

    # Änderung gleicher Länge, Zeitstempel zurückgesetzt: dircmp sieht nichts
    a = quelle / "a.txt"
    st = a.stat()
    a.write_text("Gamma", encoding="utf-8")
    os.utime(a, ns=(st.st_atime_ns, st.st_mtime_ns))
    (quelle / "neu.txt").write_text("neu", encoding="utf-8")
    (quelle / "unter" / "b.txt").unlink()

    with tempfile.TemporaryDirectory() as ziel:
        with tarfile.open(archiv, "r:gz") as tar:
            tar.extractall(ziel, filter="data")
        d = filecmp.dircmp(quelle, Path(ziel) / "daten")
        print("dircmp diff_files:", d.diff_files)
    for problem in teste_wiederherstellung(archiv, quelle):
        print(problem)
```

```text
unverändert: []
dircmp diff_files: []
Inhalt verschieden: a.txt
fehlt in der Kopie: neu.txt
zusätzlich in der Kopie: unter/b.txt
```

Der Test stellt das Archiv in einem temporären Verzeichnis wieder her und vergleicht Namen und Inhalt rekursiv. Die Unterschiede sind gewollt: `a.txt` hat einen neuen Inhalt gleicher Länge, dessen Zeitstempel wir zurückgesetzt haben, `neu.txt` wurde nach der Sicherung angelegt, und `unter/b.txt` wurde nach der Sicherung gelöscht (aus Sicht der Wiederherstellung ist die Datei „zusätzlich in der Kopie“, weil `original` jetzt das veränderte Verzeichnis ist). `dircmp.diff_files` meldet keine Abweichung, weil der Standardvergleich nur Größe und Zeitstempel prüft (die Dokumentation nennt das einen oberflächlichen, englisch *shallow*, Vergleich). `cmpfiles(..., shallow=False)` vergleicht dagegen den Inhalt und findet die Änderung. Für einen Wiederherstellungstest ist der Inhaltsvergleich Pflicht. Auch diese Aussage zum *shallow*-Verhalten wurde durch den Lauf bestätigt, nicht in der Dokumentation nachgelesen.
