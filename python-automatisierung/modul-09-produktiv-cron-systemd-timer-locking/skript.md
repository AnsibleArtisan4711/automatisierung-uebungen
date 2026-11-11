# Modul 09 - Produktiv- cron, systemd-Timer, Locking

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (9.1 bis 9.5), danach folgen „Übungsaufgaben zu Modul 09“ (8 Aufgaben) und „Lösungen zu Modul 09“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 11.01.2027 bis Fr 15.01.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul macht aus Skripten Dienste, die zuverlässig und unbeaufsichtigt laufen)

**Meilenstein:** Zeitgesteuerter Report

**Lernziele.** Nach diesem Modul können Sie ein Python-Skript als ausführbares Programm bereitstellen (Shebang, Dateirechte), es mit cron oder mit einem systemd-Timer zeitgesteuert starten, die typischen Unterschiede zwischen Ihrer Shell und der Umgebung eines Zeitgesteuerten Laufs beherrschen, verhindern, dass sich zwei Läufe überschneiden (Locking), und Skripte so schreiben, dass ein wiederholter oder unterbrochener Lauf keinen Schaden anrichtet (Idempotenz). Am Ende steht ein Tagesreport, der jeden Morgen von selbst läuft.

| **Nr.** | **Lerneinheit**                      | **Datum**     | **Dauer** | **Prio** |
|:--------|:-------------------------------------|:--------------|:----------|:---------|
| 9.1     | Shebang, Rechte, ausführbare Skripte | Mo 11.01.2027 | 2 h       | 2        |
| 9.2     | cron                                 | Di 12.01.2027 | 2 h       | 2        |
| 9.3     | systemd-Service & -Timer             | Mi 13.01.2027 | 2 h       | 2        |
| 9.4     | Locking & Idempotenz                 | Do 14.01.2027 | 2 h       | 2        |
| 9.5     | Zeitgesteuerter Report + Quiz        | Fr 15.01.2027 | 2 h       | 2        |
|         | **Summe**                            |               | **10 h**  |          |

> **Merke**
>
> Die Beispiele laufen in temporären Verzeichnissen und verändern Ihr System nicht. Was sich nicht gefahrlos ausführen lässt (Eintragen eines echten cron-Jobs, Aktivieren eines Timers mit `systemctl enable`), steht in Konsolenblöcken ohne Ausgabe und ist als Anleitung zu lesen; die Konfigurationsdateien selbst wurden aber mit den Werkzeugen des Systems geprüft (siehe 9.3). Zeitangaben wie „nächster Lauf“ hängen von der Uhrzeit ab und werden deshalb nur teilweise abgedruckt.

## 9.1 Shebang, Rechte, ausführbare Skripte

Bisher haben Sie Ihre Programme mit `python3 skript.py` gestartet. Ein Administrator ruft Programme aber meist unmittelbar auf: `./backup.py`, oder nur `backup`, wenn die Datei in einem Verzeichnis des Suchpfads liegt. Und genau so ruft sie auch ein Zeitplaner wie cron auf: Er kennt Ihr Python nicht und weiß nicht, ob Ihr Skript in Python oder Bash geschrieben ist. Damit das Betriebssystem ein Skript selbst starten kann, braucht es zwei Dinge: die Angabe, *welcher Interpreter* das Skript ausführen soll (den *Shebang*), und die Erlaubnis, die Datei *auszuführen* (das Ausführungsrecht).

**Der Shebang.** Der Shebang (auch Hash-Bang genannt, nach den beiden Zeichen `#!`) ist die *allererste Zeile* der Datei und hat die Form `#!/pfad/zum/interpreter`. Wenn der Linux-Kern eine Datei starten soll, sieht er sich die ersten Bytes an. Beginnen sie mit `#!`, startet er stattdessen den dort genannten Interpreter und übergibt ihm den Dateinamen des Skripts als Argument. Aus `./hallo` wird so `/usr/bin/python3 ./hallo`. Das ist eine Eigenschaft des Betriebssystems, nicht von Python; für Python ist die Zeile ein gewöhnlicher Kommentar, weil sie mit `#` beginnt. Zwei Schreibweisen sind verbreitet. Die erste nennt den Interpreter mit seinem festen Pfad (`#!/usr/bin/python3`); sie ist eindeutig, funktioniert aber nur, wenn Python genau dort liegt. Die zweite lautet `#!/usr/bin/env python3`: Hier startet der Kern das Programm `env`, das `python3` im Suchpfad (`PATH`) sucht und ausführt. Das ist flexibler, weil es auch eine aktivierte virtuelle Umgebung (Modul 7) findet, deren `python3` im Suchpfad vorn steht, hat aber den Nachteil, dass das Ergebnis vom Suchpfad abhängt; und der Suchpfad eines Zeitplaners ist ein anderer als der Ihrer Shell (siehe 9.2).

Das Ausführungsrecht ist eines der Rechte, die jede Datei unter Linux hat. Sie kennen die Rechte aus Modul 4: Es gibt drei Gruppen (Besitzer, Gruppe, alle anderen) mit je drei Rechten (lesen, schreiben, ausführen). Eine neu angelegte Datei hat das Ausführungsrecht *nicht*; man muss es ausdrücklich setzen, in der Shell mit `chmod +x skript` oder in Python mit `Path.chmod()` beziehungsweise `os.chmod()`. Das folgende Beispiel legt ein Skript an, versucht es zu starten, setzt das Recht und startet es erneut. Die Funktion `stat.filemode` aus der Standardbibliothek übersetzt den Zahlenwert der Rechte in die vertraute Schreibweise `-rwxr-xr-x`.

```python
import stat
import subprocess
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    skript = Path(tmp) / "hallo"
    skript.write_text('#!/usr/bin/env python3\nprint("Hallo aus dem Skript")\n', encoding="utf-8")
    skript.chmod(0o644)                                   # ausdrücklich: lesen/schreiben, kein Ausführen
    print("Rechte:", stat.filemode(skript.stat().st_mode))

    try:
        subprocess.run([str(skript)])
    except PermissionError as fehler:
        print("PermissionError:", fehler.strerror)

    skript.chmod(0o755)
    print("Rechte:", stat.filemode(skript.stat().st_mode))
    ergebnis = subprocess.run([str(skript)], capture_output=True, text=True)
    print(ergebnis.stdout.strip(), "| Exit-Code", ergebnis.returncode)
```

```text
Rechte: -rw-r--r--
PermissionError: Permission denied
Rechte: -rwxr-xr-x
Hallo aus dem Skript | Exit-Code 0
```

Die Datei `hallo` hat keine Endung `.py`; das ist bei Skripten, die als Befehle gedacht sind, üblich (niemand will `backup.py` tippen), unter Linux aber nur eine Konvention. Ohne Ausführungsrecht verweigert das Betriebssystem den Start mit `Permission denied`, was Python als `PermissionError` meldet. Nach dem `chmod` mit `0o755` läuft das Skript. Die Zahl `0o755` ist *oktal* (die Vorsilbe `0o` kennzeichnet das): Jede Ziffer steht für eine Gruppe, und sie ist die Summe aus 4 (lesen), 2 (schreiben) und 1 (ausführen). Also: 7 = 4+2+1 (alles) für den Besitzer, 5 = 4+1 (lesen und ausführen) für Gruppe und andere.

**Rechte als Zahlen und Bits.** Wenn Sie Rechte in einem Programm *prüfen* oder *verändern*, ist die Zahl bequemer als der Text. Das Ergebnis von `stat()` enthält in `st_mode` die Dateiart und die Rechte in einer einzigen Zahl; mit `stat.S_IMODE` schneidet man die Rechte heraus. Die Konstanten im Modul `stat` (`S_IXUSR` für „Ausführen durch den Besitzer“, `S_IWOTH` für „Schreiben für andere“ und so weiter) lassen sich mit `|` (ODER) zusammensetzen und mit `&` (UND) abfragen. Solche Bit-Operationen sind in diesem Zusammenhang unvermeidbar; sie sind einfacher, als sie aussehen: Man setzt ein Bit mit `|=` und prüft es mit `&`.

```python
import os
import stat
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    datei = Path(tmp) / "skript.py"
    datei.write_text("print('x')\n", encoding="utf-8")
    datei.chmod(0o640)

    modus = datei.stat().st_mode
    print(oct(stat.S_IMODE(modus)), stat.filemode(modus))

    # Ausführungsrecht für den Besitzer ergänzen, alles andere unverändert lassen
    datei.chmod(stat.S_IMODE(modus) | stat.S_IXUSR)
    modus = datei.stat().st_mode
    print(oct(stat.S_IMODE(modus)), stat.filemode(modus))

    # Abfragen
    print("Besitzer darf ausführen:", bool(modus & stat.S_IXUSR))
    print("Andere dürfen lesen   :", bool(modus & stat.S_IROTH))
    print("Andere dürfen schreiben:", bool(modus & stat.S_IWOTH))
    print("os.access X_OK        :", os.access(datei, os.X_OK))
```

```text
0o640 -rw-r-----
0o740 -rwxr-----
Besitzer darf ausführen: True
Andere dürfen lesen   : False
Andere dürfen schreiben: False
os.access X_OK        : True
```

Das Ergebnis `0o640` bedeutet: Besitzer darf lesen und schreiben (6), die Gruppe lesen (4), andere haben keine Rechte (0). Nach dem Ergänzen steht dort `0o740`. Die Funktion `os.access` fragt das Betriebssystem, ob der *aktuelle Benutzer* das Recht hat; das ist nicht dasselbe wie ein Blick auf die Bits: Für `root` kann `os.access` auch dann wahr sein, wenn die Bits etwas anderes sagen (Root darf fast alles, mit Ausnahme des Ausführens einer Datei ohne irgendein Ausführungsbit). Deshalb prüft man in Sicherheitsüberlegungen die Bits selbst.

**Welche Rechte sind sinnvoll?** Für ein Skript, das von cron oder von Kollegen aufgerufen wird, genügt in der Regel `0o755` (alle dürfen lesen und ausführen, nur der Besitzer darf ändern) oder `0o750` (nur Besitzer und Gruppe). Entscheidend ist das *Schreibrecht*: Ein Skript, das *jeder* ändern darf (`0o777` oder `0o666`), ist ein Einfallstor, denn wer es ändert, bestimmt, was das Programm beim nächsten Lauf tut. Läuft das Skript unter einem mächtigen Benutzer (etwa als `root` aus einem Cron-Job), kann ein Angreifer mit einem Schreibrecht auf das Skript, das Verzeichnis oder eine von ihm eingebundene Datei die Rechte dieses Benutzers erlangen. Das ist eine allgemeine Regel der Systemsicherheit und wurde hier nicht in einer Quelle nachgeschlagen; sie gilt auch für Konfigurationsdateien, die das Skript liest. Dateien mit Zugangsdaten (Modul 10) bekommen darüber hinaus keine Leserechte für andere: `0o600`.

Das Verhalten beim *Anlegen* neuer Dateien bestimmt die *umask*, eine Maske, die vom Standard (für Dateien `0o666`) bestimmte Rechte abzieht. Mit der üblichen umask `0o022` entstehen Dateien mit `0o644`; mit `0o077` entstehen Dateien nur für den Besitzer lesbar. Wer in Python Dateien mit Zugangsdaten anlegt, sollte die Rechte *beim Anlegen* festlegen, statt sie nachträglich zu ändern, denn zwischen Anlegen und `chmod` besteht ein kurzes Zeitfenster, in dem die Datei zu offen ist. Das geht mit `os.open` und dem Modus als drittem Argument.

```python
import os
import stat
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    pfad = Path(tmp) / "geheim.txt"

    # Datei von Anfang an nur für den Besitzer: O_EXCL verhindert, dass eine vorhandene Datei überschrieben wird
    fd = os.open(pfad, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        f.write("passwort=...\n")
    print(stat.filemode(pfad.stat().st_mode))

    try:
        os.open(pfad, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    except FileExistsError:
        print("O_EXCL: Datei existiert schon, nichts überschrieben")
```

```text
-rw-------
O_EXCL: Datei existiert schon, nichts überschrieben
```

Die Rechte `-rw-------` gelten vom ersten Augenblick an. Die Kombination `O_CREAT | O_EXCL` bewirkt, dass das Anlegen fehlschlägt, wenn die Datei schon existiert; das ist auch die Grundlage für einfache Sperrdateien in 9.4. (Die tatsächlichen Rechte entstehen aus dem angegebenen Modus abzüglich der umask des Prozesses; bei einer sehr strengen umask können sie noch enger sein, nie weiter.)

**Fallstricke beim Shebang.** Wenn ein Skript mit `./skript` nicht startet, obwohl es mit `python3 skript` läuft, liegt es fast immer an einer von drei Ursachen. Die erste ist das fehlende Ausführungsrecht (siehe oben). Die zweite ist ein falscher Interpreterpfad: Existiert der Pfad hinter `#!` nicht, meldet die Shell `bad interpreter: No such file or directory`, und das irritiert, weil die Skriptdatei ja existiert; gemeint ist der Interpreter. Ruft man das Skript aus Python mit `subprocess` auf, entsteht dieselbe Verwirrung als `FileNotFoundError` für die Skriptdatei. Die dritte Ursache sind Windows-Zeilenenden: Wurde die Datei unter Windows mit `\r\n` am Zeilenende gespeichert, sucht der Kern einen Interpreter namens `python3\r` mit einem unsichtbaren Wagenrücklaufzeichen am Ende, und der existiert nicht. Das ist ein klassischer Fehler, wenn man auf Windows schreibt und auf Linux ausführt, und deshalb für Sie besonders relevant: Stellen Sie Ihren Editor auf Unix-Zeilenenden (`LF`) ein, und kontrollieren Sie notfalls mit `file skript`.

```python
import subprocess
import tempfile
from pathlib import Path

def starte(inhalt):
    with tempfile.TemporaryDirectory() as tmp:
        skript = Path(tmp) / "skript"
        skript.write_bytes(inhalt)
        skript.chmod(0o755)
        try:
            r = subprocess.run([str(skript)], capture_output=True, text=True)
        except OSError as fehler:
            return f"{type(fehler).__name__}: {fehler.strerror}"
        return f"Exit-Code {r.returncode} | stdout={r.stdout.strip()!r} | stderr={r.stderr.translate({0x2018: 39, 0x2019: 39}).strip().splitlines()[0] if r.stderr else ''!r}"

print("1 richtig            :", starte(b'#!/usr/bin/env python3\nprint("ok")\n'))
print("2 Interpreter fehlt  :", starte(b'#!/usr/bin/gibt-es-nicht\nprint("ok")\n'))
print("3 Windows-Zeilenende :", starte(b'#!/usr/bin/env python3\r\nprint("ok")\r\n'))
print("4 Shebang nicht oben :", starte(b'\n#!/usr/bin/env python3\nprint("ok")\n'))
```

```text
1 richtig            : Exit-Code 0 | stdout='ok' | stderr=''
2 Interpreter fehlt  : FileNotFoundError: No such file or directory
3 Windows-Zeilenende : Exit-Code 127 | stdout='' | stderr="/usr/bin/env: 'python3\\r': No such file or directory"
4 Shebang nicht oben : OSError: Exec format error
```

Im zweiten Fall gibt Python einen `FileNotFoundError` aus, obwohl die Datei `skript` da ist; gemeint ist der fehlende Interpreter. Im dritten Fall startet `env` zwar, findet aber ein Programm namens `python3\r` nicht und endet mit dem Code 127. Im vierten Fall steht vor dem Shebang eine Leerzeile; dann ist es für den Kern kein Shebang mehr, und er kann die Datei nicht ausführen (`Exec format error`). Eine interaktive Shell würde in dieser Lage die Datei als Shell-Skript behandeln und mit Syntaxfehlern des Python-Codes scheitern; ein Zeitplaner oder ein `subprocess`-Aufruf ohne Shell zeigt die Fehlermeldung des Kerns. Der Shebang muss *in der ersten Zeile, ab dem ersten Zeichen* stehen. (Der genaue Wortlaut der Fehlermeldungen hängt von Shell und Betriebssystem ab; das Verhalten, also dass das Skript in den Fällen 2 bis 4 nicht korrekt läuft, ist der Punkt.)

**Der Interpreter einer virtuellen Umgebung.** Für Skripte, die Pakete aus einer virtuellen Umgebung brauchen (Modul 7), gibt es eine robuste Lösung: den *vollständigen Pfad* zum Python-Programm der Umgebung im Shebang. Dann findet das Skript seine Pakete, ohne dass jemand die Umgebung vorher aktivieren muss, und das ist genau die Situation bei einem Zeitplaner. Ein Nachteil ist, dass der Pfad auf ein bestimmtes System festgelegt ist (und dass die virtuelle Umgebung nicht verschoben werden darf, wie in 7.2 gezeigt). Das Beispiel legt eine kleine virtuelle Umgebung ohne `pip` an und lässt ein Skript über deren Interpreter laufen.

```python
import subprocess
import sys
import tempfile
import venv
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    umgebung = Path(tmp) / ".venv"
    venv.create(umgebung, with_pip=False)
    python = umgebung / "bin" / "python"

    skript = Path(tmp) / "zeige-umgebung"
    skript.write_text(f"#!{python}\nimport sys\nprint(sys.prefix.endswith('.venv'))\n", encoding="utf-8")
    skript.chmod(0o755)

    print("Erste Zeile endet auf:", skript.read_text().splitlines()[0].split("/")[-3:])
    r = subprocess.run([str(skript)], capture_output=True, text=True)
    print("Läuft in der venv:", r.stdout.strip())
    print("Das aufrufende Python ist ein anderes:", Path(sys.prefix).name != ".venv")
```

```text
Erste Zeile endet auf: ['.venv', 'bin', 'python']
Läuft in der venv: True
Das aufrufende Python ist ein anderes: True
```

Die Skriptdatei hat als erste Zeile den absoluten Pfad zu `.venv/bin/python`; beim Start nimmt der Kern genau diesen Interpreter, und `sys.prefix` verrät, dass das Skript in der Umgebung läuft, ohne dass wir sie aktiviert haben. In einem echten Projekt würden Sie die Umgebung an einem festen Ort anlegen (etwa unter `/opt/projekt/.venv`) und den Pfad im Shebang oder, noch klarer, in der Startzeile des Zeitplaners angeben (siehe 9.2 und 9.3).

**Ein Skript ordentlich aufbauen.** Aus Modul 6 kennen Sie den Rahmen, den jedes Programm bekommen sollte, das ein anderes Programm (hier: der Zeitplaner) startet: ein Shebang, eine Funktion `main` mit den Argumenten, der Aufruf `sys.exit(main())` unter `if __name__ == "__main__":`, Fehlermeldungen auf `stderr`, aussagekräftige Exit-Codes. Der Zeitplaner wertet den Exit-Code aus (oder protokolliert zumindest, dass er nicht 0 war), aber er liest keine Meldungen, die ein Mensch später einmal sehen wird: Was Ihr Skript nicht selbst protokolliert (Modul 6, `logging`), geht verloren. Ein kleines Gerüst, das Sie als Vorlage für alle Skripte dieses Moduls verwenden können:

```python
#!/usr/bin/env python3
"""Vorlage fuer ein Skript, das ein Zeitplaner startet."""
import argparse
import logging
import sys

log = logging.getLogger("vorlage")


def main(argv=None):
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--ausfuehrlich", action="store_true")
    args = parser.parse_args(argv)
    logging.basicConfig(
        level=logging.DEBUG if args.ausfuehrlich else logging.INFO,
        format="%(asctime)s %(levelname)s %(message)s",
    )
    log.info("Lauf beginnt")
    try:
        ...                                   # hier steht die eigentliche Arbeit
    except OSError as fehler:
        log.error("Arbeit fehlgeschlagen: %s", fehler)
        return 1
    log.info("Lauf beendet")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Die Auslassungspunkte `...` sind in Python ein gültiger Ausdruck (das Objekt `Ellipsis`) und dienen hier als Platzhalter. Das Gerüst zeigt, wie Sie das Skript später entweder als Befehl (`./vorlage.py`) oder als Modul (`import vorlage`) verwenden und testen können. Beachten Sie, dass `logging` standardmäßig auf den Fehlerkanal (`stderr`) schreibt, was für Zeitplaner genau richtig ist: cron schickt bei einem Lauf mit Ausgabe eine Nachricht an den Besitzer (siehe 9.2), und systemd fängt beide Kanäle im Journal ab (siehe 9.3).

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler dieser Lerneinheit sind: das vergessene Ausführungsrecht (`Permission denied`), der Shebang, der nicht in Zeile 1 steht oder auf ein nicht vorhandenes Programm zeigt, die Windows-Zeilenenden, ein Skript, das *jeder* ändern darf, und die Annahme, dass `python` existiert (auf neueren Debian- und Ubuntu-Systemen heißt der Befehl `python3`; ob zusätzlich ein `python` vorhanden ist, hängt von der Installation ab). Zum Test, ob Ihr Skript sich wie ein echter Befehl verhält, hilft eine einfache Probe: Rufen Sie es aus einem *anderen Verzeichnis* mit vollem Pfad und in einer *leeren Umgebung* auf (`env -i /pfad/skript`). Läuft es so, läuft es in der Regel auch unter einem Zeitplaner.

```python
import stat
import tempfile
from pathlib import Path

def pruefe_skript(pfad):
    """Liefert eine Liste von Warnungen zu Rechten und Shebang eines Skripts."""
    pfad = Path(pfad)
    warnungen = []
    modus = pfad.stat().st_mode
    if not modus & stat.S_IXUSR:
        warnungen.append("Besitzer darf nicht ausführen")
    if modus & stat.S_IWOTH:
        warnungen.append("für alle schreibbar")
    elif modus & stat.S_IWGRP:
        warnungen.append("für die Gruppe schreibbar")
    erste = pfad.read_bytes().split(b"\n", 1)[0]
    if not erste.startswith(b"#!"):
        warnungen.append("kein Shebang in Zeile 1")
    if erste.endswith(b"\r"):
        warnungen.append("Windows-Zeilenende im Shebang")
    return warnungen

with tempfile.TemporaryDirectory() as tmp:
    fälle = {
        "gut": (b"#!/usr/bin/env python3\nprint(1)\n", 0o755),
        "offen": (b"#!/usr/bin/env python3\nprint(1)\n", 0o777),
        "ohne x und Shebang": (b"print(1)\n", 0o644),
        "crlf": (b"#!/usr/bin/env python3\r\nprint(1)\r\n", 0o750),
    }
    for name, (inhalt, rechte) in fälle.items():
        pfad = Path(tmp) / name.replace(" ", "_")
        pfad.write_bytes(inhalt)
        pfad.chmod(rechte)
        print(f"{name:20}", pruefe_skript(pfad) or "in Ordnung")
```

```text
gut                  in Ordnung
offen                ['für alle schreibbar']
ohne x und Shebang   ['Besitzer darf nicht ausführen', 'kein Shebang in Zeile 1']
crlf                 ['Windows-Zeilenende im Shebang']
```

Die Funktion fasst die Fehlerquellen dieser Lerneinheit in einer Prüfung zusammen: Besitzer-Ausführungsrecht, Schreibrechte für Gruppe und andere, Shebang in der ersten Zeile, Windows-Zeilenende. Eine solche Prüfung kann man als Vorbedingung in ein Rollout-Skript einbauen, das Skripte auf Server kopiert.

> **Ergänzung aus der Nachrecherche**
>
> **Beobachtet in der Prüfumgebung (Debian/Ubuntu-cron, Python 3.13.18).** Ein per cron gestarteter Testjob sah die Umgebungsvariablen `HOME`, `LOGNAME`, `PATH=/usr/bin:/bin`, `PWD`, `SHELL=/bin/sh` und `LANG`; der Suchpfad war also deutlich kürzer als der einer interaktiven Shell. Das ist eine Beobachtung an einem System; welche Variablen ein anderes System setzt, hängt von der cron-Version und der Konfiguration ab. Die Beschreibung des Shebang-Mechanismus (der Kern startet den genannten Interpreter und übergibt ihm den Skriptnamen), die Rolle der umask und die Wirkung von Windows-Zeilenenden beruhen auf Fachwissen und auf den hier ausgeführten Beispielen, nicht auf einer abgerufenen Quelle. Die Dokumentation von cron (abgerufen) folgt in 9.2.

> **Merke**
>
> Ein ausführbares Skript braucht einen Shebang in Zeile 1 (`#!/usr/bin/env python3` oder den festen Pfad, bei virtuellen Umgebungen deren `bin/python`) und das Ausführungsrecht (`chmod +x`, in Python `Path.chmod(0o755)`). Rechte sind Oktalzahlen aus 4 (lesen), 2 (schreiben), 1 (ausführen). Skripte nie für alle schreibbar machen; Dateien mit Zugangsdaten mit `os.open(..., 0o600)` anlegen. Unix-Zeilenenden verwenden. Ein Skript, das ein Zeitplaner startet, protokolliert selbst und liefert einen Exit-Code.

> **Praxisbeispiel**
>
> Schon in der ersten Woche als Administrator liegt Ihr erstes Skript in `/usr/local/bin`, damit Kollegen es als Befehl aufrufen können, oder in `/opt/skripte`, damit ein Cron-Job es findet. Wenn dann nichts passiert, ist die Fehlersuche dieser Lerneinheit Ihr Werkzeug: Rechte, Shebang, Zeilenenden, Besitzer. Rechte, die Sie einmal falsch gesetzt haben (ein Skript für alle schreibbar), sind außerdem ein typischer Befund bei Sicherheitsprüfungen von Servern.

## 9.2 cron

*cron* ist der klassische Zeitplaner von Unix und Linux: ein Dienst (ein *Daemon*, also ein im Hintergrund dauernd laufendes Programm), der jede Minute nachsieht, ob irgendein eingetragener Auftrag fällig ist, und diesen dann startet. Die Aufträge stehen in Textdateien, den *crontabs* (von „cron table“). Jede Zeile ist ein Auftrag: fünf Felder, die sagen, *wann* er läuft, und danach der Befehl, der dann ausgeführt wird. Das ist denkbar einfach, und deshalb läuft cron seit Jahrzehnten auf praktisch jedem Server, obwohl es modernere Alternativen gibt (systemd-Timer, siehe 9.3). Als Administrator werden Sie ständig auf cron-Einträge treffen, und Sie sollten sie lesen und schreiben können.

Auf Debian und Ubuntu ist cron üblicherweise installiert und aktiv. Jeder Benutzer hat eine eigene crontab, die man mit `crontab -e` bearbeitet (der Befehl öffnet einen Editor und prüft beim Speichern die Syntax), mit `crontab -l` anzeigt und mit `crontab -r` löscht. Vorsicht bei der letzten Variante: Sie entfernt die gesamte crontab ohne Rückfrage (das Verhalten von `-r` ist Fachwissen und wurde hier nicht nachgeschlagen; `crontab -r` und `crontab -e` liegen auf der Tastatur gefährlich nah beieinander). Außerdem gibt es systemweite Einträge in `/etc/crontab` und in Dateien unter `/etc/cron.d/`, bei denen zusätzlich der Benutzer angegeben wird, unter dem der Befehl läuft, sowie die Verzeichnisse `/etc/cron.hourly`, `/etc/cron.daily` und weitere, in die man einfach ausführbare Skripte legt. Auch das ist Fachwissen über Debian-artige Systeme und hier nicht nachgeschlagen; Details unterscheiden sich je nach Distribution.

**Die fünf Felder.** Die Handbuchseite `crontab(5)` (Debian-Ausgabe, abgerufen) beschreibt die Felder von links nach rechts: Minute (0 bis 59), Stunde (0 bis 23), Tag des Monats (1 bis 31), Monat (1 bis 12, oder englische Namen) und Wochentag (0 bis 7, wobei sowohl 0 als auch 7 für Sonntag stehen, oder englische Namen). Die Felder sind durch Leerzeichen oder Tabulatoren getrennt. In jedem Feld ist ein Stern `*` „jeder Wert“; man kann Listen (`1,15`), Bereiche (`8-11`) und Schritte (`*/2`, `0-23/2`) angeben. Zu beachten ist, dass cron nur *auf die Minute genau* arbeitet: Es gibt keine Sekundenangabe, und der kürzeste Abstand zwischen zwei Läufen ist eine Minute.

| **Eintrag** | **Bedeutung**                                                         |
|:------------|:----------------------------------------------------------------------|
| `* * * * *`    | jede Minute                                                           |
| `*/15 * * * *`    | alle 15 Minuten (zur Minute 0, 15, 30, 45)                            |
| `30 4 * * *`    | täglich um 04:30 Uhr                                                  |
| `30 4 * * 1-5`    | montags bis freitags um 04:30 Uhr                                     |
| `0 3 * * 0`    | sonntags um 03:00 Uhr                                                 |
| `0 0 1 * *`    | am Ersten jedes Monats um Mitternacht                                 |
| `0 9,17 * * *`    | täglich um 09:00 und um 17:00 Uhr                                     |
| `@daily`    | Kurzform für `0 0 * * *` (ebenso `@hourly`, `@weekly`, `@monthly`, `@yearly`) |
| `@reboot`    | einmal beim Start des cron-Dienstes                                   |

Die Kurzformen mit `@` ersetzen die ersten fünf Felder. Nach der Handbuchseite entspricht `@daily` dem Eintrag `0 0 * * *`, `@hourly` dem Eintrag `0 * * * *`, `@weekly` dem Eintrag `0 0 * * 0` und `@monthly` dem Eintrag `0 0 1 * *`. Die Form `@reboot` läuft einmal beim Start des cron-Dienstes, möglicherweise noch bevor andere Dienste hochgefahren sind; für Aufträge, die auf Netzwerk oder Datenbank angewiesen sind, ist sie deshalb mit Vorsicht zu verwenden.

Ein vollständiger Eintrag sieht so aus: `30 4 * * * /opt/skripte/backup.py --ziel /srv/backup`. Ein Beispiel für eine Zeile, die Sie in Ihre crontab eintragen könnten, lautet: `30 4 * * 1-5 /usr/bin/python3 /opt/skripte/tagesreport.py >> /var/log/tagesreport.log 2>&1`. Die Zeitangabe bedeutet „an Werktagen um 04:30 Uhr“; der Befehl ruft den Interpreter mit *vollem Pfad* auf, übergibt das Skript mit vollem Pfad und leitet beide Ausgabekanäle in eine Protokolldatei um. Warum das alles so ausführlich geschrieben wird, erklären die nächsten Abschnitte.

**Cron-Ausdrücke lesen lernen: ein kleiner Rechner.** Cron-Ausdrücke sind kompakt und deshalb fehleranfällig. Ein Mensch liest `30 4 1,15 * 5` und weiß nicht ohne Weiteres, wann der Auftrag läuft. Zum Verstehen (und zum Testen von Ausdrücken, bevor man sie in eine echte crontab einträgt) lohnt sich ein kleiner Rechner, der aus einem Ausdruck die nächsten Laufzeiten berechnet. Er setzt Python-Kenntnisse aus den Modulen 2, 3 und 8 ein: Mengen, Funktionen, `datetime` und `timedelta`. Ein Feld wird zu einer *Menge* erlaubter Werte; ein Zeitpunkt „passt“, wenn jeder seiner Bestandteile in der jeweiligen Menge liegt. Eine Sonderregel betrifft die beiden Tagesfelder, auf die wir gleich zurückkommen.

```python
"""cronzeit - Cron-Ausdruecke pruefen und die naechsten Laufzeiten berechnen."""
from datetime import datetime, timedelta

GRENZEN = [(0, 59), (0, 23), (1, 31), (1, 12), (0, 7)]
SPEZIAL = {
    "@hourly": "0 * * * *", "@daily": "0 0 * * *", "@midnight": "0 0 * * *",
    "@weekly": "0 0 * * 0", "@monthly": "0 0 1 * *", "@yearly": "0 0 1 1 *", "@annually": "0 0 1 1 *",
}


def zahl(text, feld):
    try:
        return int(text)
    except ValueError:
        raise ValueError(f"Keine Zahl {text!r} in Feld {feld!r}") from None


def parse_feld(text, minimum, maximum):
    """Liefert die Menge der Werte, die ein Feld wie '*/15' oder '1,5-8' beschreibt."""
    werte = set()
    for teil in text.split(","):
        schritt = 1
        if "/" in teil:
            teil, rest = teil.split("/", 1)
            schritt = zahl(rest, text)
            if schritt < 1:
                raise ValueError(f"Schrittweite muss mindestens 1 sein: {text!r}")
        if teil == "*":
            von, bis = minimum, maximum
        elif "-" in teil:
            a, b = teil.split("-", 1)
            von, bis = zahl(a, text), zahl(b, text)
        else:
            von = bis = zahl(teil, text)
        if not (minimum <= von <= bis <= maximum):
            raise ValueError(f"Wert ausserhalb von {minimum}-{maximum}: {text!r}")
        werte.update(range(von, bis + 1, schritt))
    return werte


def parse_ausdruck(ausdruck):
    if ausdruck.strip() in SPEZIAL:
        ausdruck = SPEZIAL[ausdruck.strip()]
    elif ausdruck.strip().startswith("@"):
        raise ValueError(f"{ausdruck.strip()!r} hat keine berechenbare Laufzeit")
    felder = ausdruck.split()
    if len(felder) != 5:
        raise ValueError(f"Erwartet 5 Felder, gefunden {len(felder)}: {ausdruck!r}")
    mengen = [parse_feld(f, lo, hi) for f, (lo, hi) in zip(felder, GRENZEN)]
    mengen[4] = {0 if w == 7 else w for w in mengen[4]}          # 7 ist ebenfalls Sonntag
    beide_eingeschraenkt = not felder[2].startswith("*") and not felder[4].startswith("*")
    return mengen, beide_eingeschraenkt


def tag_passt(tag, monat_menge, tag_menge, wochentag_menge, oder_regel):
    wochentag = (tag.weekday() + 1) % 7                           # Python: Montag=0; cron: Sonntag=0
    if tag.month not in monat_menge:
        return False
    if oder_regel:
        return tag.day in tag_menge or wochentag in wochentag_menge
    return tag.day in tag_menge and wochentag in wochentag_menge


def naechste_laeufe(ausdruck, ab, anzahl=3):
    """Die naechsten Laufzeiten nach dem Zeitpunkt 'ab' (naiv, Ortszeit des Servers)."""
    (minuten, stunden, tage, monate, wochentage), oder_regel = parse_ausdruck(ausdruck)
    zeit = ab.replace(second=0, microsecond=0) + timedelta(minutes=1)
    ende = ab + timedelta(days=366 * 5)
    ergebnis = []
    while len(ergebnis) < anzahl and zeit < ende:
        if not tag_passt(zeit, monate, tage, wochentage, oder_regel):
            zeit = (zeit + timedelta(days=1)).replace(hour=0, minute=0)
        elif zeit.hour not in stunden:
            zeit = (zeit + timedelta(hours=1)).replace(minute=0)
        elif zeit.minute not in minuten:
            zeit += timedelta(minutes=1)
        else:
            ergebnis.append(zeit)
            zeit += timedelta(minutes=1)
    return ergebnis
```

Die Funktion `parse_feld` zerlegt ein Feld an den Kommas und behandelt jedes Teilstück einzeln: Ein Stern bedeutet den ganzen erlaubten Bereich, ein Bindestrich einen Bereich, eine einzelne Zahl diesen einen Wert; ein Schrägstrich dahinter gibt die Schrittweite an, die an `range(von, bis + 1, schritt)` weitergegeben wird. Ungültige Eingaben lösen einen `ValueError` mit verständlicher Meldung aus. `parse_ausdruck` verarbeitet die Kurzformen, prüft, dass es genau fünf Felder gibt, und wandelt den Wochentag 7 in 0 um. `naechste_laeufe` läuft die Zeit *minutenweise* vor, springt aber, wo möglich, größere Schritte: Passt der Tag nicht, geht es direkt zum nächsten Tag; passt die Stunde nicht, zur nächsten Stunde. Das ist eine Optimierung, die nichts am Ergebnis ändert, aber verhindert, dass für einen Ausdruck wie `0 0 29 2 *` (nur am 29. Februar) Millionen Minuten einzeln geprüft werden. Zeitpunkte sind hier *naiv*: cron arbeitet in der Ortszeit des Servers (siehe unten).

Probieren wir es an den Beispielen der Tabelle und an einigen Fällen aus, die oft missverstanden werden. Als Startpunkt verwenden wir einen festen Zeitpunkt, Montag, den 11. Januar 2027, 09:17 Uhr, damit das Ergebnis reproduzierbar ist.

```python
from datetime import datetime

import cronzeit

ab = datetime(2027, 1, 11, 9, 17, 42)

for ausdruck in ("*/15 * * * *", "30 4 * * 1-5", "0 3 * * 7", "@daily", "0 9,17 * * *"):
    laeufe = cronzeit.naechste_laeufe(ausdruck, ab, 3)
    print(f"{ausdruck:15}", [t.strftime("%a %d.%m. %H:%M") for t in laeufe])

for falsch in ("61 * * * *", "*/0 * * * *", "* * * *", "@reboot", "a * * * *"):
    try:
        cronzeit.naechste_laeufe(falsch, ab)
    except ValueError as fehler:
        print(f"{falsch!r:15}", "->", fehler)
```

```text
*/15 * * * *    ['Mon 11.01. 09:30', 'Mon 11.01. 09:45', 'Mon 11.01. 10:00']
30 4 * * 1-5    ['Tue 12.01. 04:30', 'Wed 13.01. 04:30', 'Thu 14.01. 04:30']
0 3 * * 7       ['Sun 17.01. 03:00', 'Sun 24.01. 03:00', 'Sun 31.01. 03:00']
@daily          ['Tue 12.01. 00:00', 'Wed 13.01. 00:00', 'Thu 14.01. 00:00']
0 9,17 * * *    ['Mon 11.01. 17:00', 'Tue 12.01. 09:00', 'Tue 12.01. 17:00']
'61 * * * *'    -> Wert ausserhalb von 0-59: '61'
'*/0 * * * *'   -> Schrittweite muss mindestens 1 sein: '*/0'
'* * * *'       -> Erwartet 5 Felder, gefunden 4: '* * * *'
'@reboot'       -> '@reboot' hat keine berechenbare Laufzeit
'a * * * *'     -> Keine Zahl 'a' in Feld 'a'
```

Die erste Zeile zeigt, dass `*/15` ab 09:17 den nächsten Lauf um 09:30 liefert (die Minuten 0, 15, 30 und 45 jeder Stunde sind erlaubt). Der Ausdruck mit `1-5` überspringt das Wochenende; die Zeile mit dem Wochentag `7` liefert Sonntage. Die Wochentage erscheinen hier in der Sprache des laufenden Programms, in der Standardeinstellung als englische Abkürzungen. Die Fehlerfälle zeigen den Nutzen der Prüfung: ein Wert außerhalb des Bereichs, eine Schrittweite von null, zu wenige Felder, ein `@reboot` (keine feste Uhrzeit) und ein Text statt einer Zahl.

**Die Falle mit Tag und Wochentag.** Eine Besonderheit von cron verwirrt fast jeden einmal. Sind *beide* Tagesfelder (Tag des Monats und Wochentag) eingeschränkt, das heißt, keines von ihnen ist ein Stern, dann läuft der Auftrag, wenn *eines von beiden* passt, nicht beide zugleich. Die Handbuchseite nennt als Beispiel `30 4 1,15 * 5`: Der Befehl läuft um 04:30 Uhr am 1. und am 15. des Monats *und zusätzlich* an jedem Freitag. Wer „am 15., wenn es ein Freitag ist“ meint, kann das in cron nicht direkt ausdrücken; man prüft den Wochentag dann im Befehl selbst. Ein zweiter, noch subtilerer Punkt betrifft Felder, die mit einem Stern *beginnen*: Die Handbuchseite beschreibt im Abschnitt über Einschränkungen, dass die Debian-Implementierung nur prüft, ob das erste Zeichen ein Stern ist. Deshalb läuft `0 0 */2 * sun` nach dieser Implementierung nur an den Sonntagen, die zugleich ein ungerader Kalendertag sind, während die POSIX-Norm *jeden* Sonntag und *jeden* ungeraden Tag vorsehen würde. Unser Rechner folgt der Debian-Beschreibung.

```python
from datetime import datetime

import cronzeit

ab = datetime(2027, 1, 11, 9, 0)

def zeige(ausdruck, anzahl=4):
    laeufe = cronzeit.naechste_laeufe(ausdruck, ab, anzahl)
    print(f"{ausdruck:15}", ", ".join(t.strftime("%a %d.%m.") for t in laeufe))

zeige("0 6 1,15 * *")      # nur Kalendertage eingeschränkt: der 1. und der 15.
zeige("0 6 * * 5")         # nur der Wochentag eingeschränkt: jeden Freitag
zeige("0 6 1,15 * 5")      # beide eingeschränkt: der 1., der 15. UND jeder Freitag
zeige("0 6 */2 * 0")       # Debian: nur Sonntage mit ungeradem Datum
```

```text
0 6 1,15 * *    Fri 15.01., Mon 01.02., Mon 15.02., Mon 01.03.
0 6 * * 5       Fri 15.01., Fri 22.01., Fri 29.01., Fri 05.02.
0 6 1,15 * 5    Fri 15.01., Fri 22.01., Fri 29.01., Mon 01.02.
0 6 */2 * 0     Sun 17.01., Sun 31.01., Sun 07.02., Sun 21.02.
```

In der dritten Zeile stehen sowohl die Freitage (15., 22., 29. Januar) als auch der 1. Februar (ein Montag): Die beiden Bedingungen werden mit ODER verknüpft. In der vierten Zeile erscheinen nur Sonntage, und zwar die mit ungeradem Datum (17. und 31. Januar, 7. Februar); der 24. Januar, ein gerader Tag, fehlt. Wer sich auf solche Feinheiten nicht verlassen will, schreibt in die Tagesfelder jeweils nur *eine* Einschränkung und prüft zusätzliche Bedingungen im Skript.

**Die Umgebung eines cron-Jobs.** Der wichtigste Unterschied zwischen einem Lauf in Ihrer Shell und einem Lauf durch cron ist die *Umgebung*. cron startet Ihren Befehl nicht in einer Anmelde-Sitzung. Es liest weder Ihre `.bashrc` noch Ihre `.profile`, es aktiviert keine virtuelle Umgebung, und es kennt Ihre Aliase nicht. Die Handbuchseite sagt dazu: cron setzt einige Umgebungsvariablen automatisch; `SHELL` ist `/usr/bin/sh` (der Wert der Handbuchseite; auf manchen Systemen steht `/bin/sh`), und `LOGNAME` und `HOME` stammen aus der Zeile des Benutzers in `/etc/passwd`. `HOME` und `SHELL` lassen sich in der crontab überschreiben, `LOGNAME` nicht. Zum `PATH` macht die Seite keine Aussage; in unserer Prüfumgebung hatte ein per cron gestarteter Job `PATH=/usr/bin:/bin` (siehe Ergänzung). Dieser Suchpfad ist viel kürzer als der einer interaktiven Shell. Programme, die in `/usr/local/bin`, in `~/.local/bin` oder in einer virtuellen Umgebung liegen, findet cron also nicht. Das Arbeitsverzeichnis ist das Heimatverzeichnis des Benutzers, nicht das Verzeichnis, in dem Ihr Skript liegt.

Aus diesen Eigenschaften ergeben sich Regeln, die Sie sich merken sollten. Erstens: Verwenden Sie in der crontab *absolute Pfade* für den Interpreter und für das Skript. Zweitens: Verwenden Sie in Ihrem Skript keine relativen Pfade, es sei denn, Sie ermitteln sie vom Ort des Skripts aus (`Path(__file__).resolve().parent`, siehe Modul 4). Drittens: Wenn Ihr Skript Pakete aus einer virtuellen Umgebung braucht, rufen Sie den Interpreter der Umgebung direkt auf (`/opt/projekt/.venv/bin/python`, siehe 9.1). Viertens: Setzen Sie bei Bedarf Variablen am Anfang der crontab, zum Beispiel `PATH=/usr/local/bin:/usr/bin:/bin`. Die Handbuchseite weist darauf hin, dass die Werte *nicht expandiert* werden: `PATH=$HOME/bin:$PATH` funktioniert also nicht, man schreibt den Pfad ganz aus. Das folgende Beispiel simuliert die Umgebung von cron, indem es einen Befehl mit einer minimalen Umgebung startet, und zeigt, welche Programme dort gefunden werden. Die Probe ist ein guter Test für jedes Skript, bevor Sie es in die crontab eintragen.

```python
import subprocess

minimal = {"PATH": "/usr/bin:/bin", "HOME": "/root", "SHELL": "/bin/sh"}

for programm in ("python3", "sh", "cat", "meinprogramm"):
    r = subprocess.run(["/bin/sh", "-c", f"command -v {programm} || echo '{programm}: NICHT GEFUNDEN'"],
                       env=minimal, capture_output=True, text=True)
    print(r.stdout.strip())

r = subprocess.run(["/bin/sh", "-c", "echo $PATH; echo ${VIRTUAL_ENV:-keine virtuelle Umgebung aktiv}"],
                   env=minimal, capture_output=True, text=True)
print(r.stdout.strip())
```

```text
/usr/bin/python3
/usr/bin/sh
/usr/bin/cat
meinprogramm: NICHT GEFUNDEN
/usr/bin:/bin
keine virtuelle Umgebung aktiv
```

Die Standardwerkzeuge werden gefunden, ein Programm, das nur in Ihrem persönlichen `PATH` liegt (hier das ausgedachte `meinprogramm`), nicht, und es ist keine virtuelle Umgebung aktiv. Das ist die Welt, in der Ihr Skript unter cron läuft. Die Ausgabe der ersten Zeilen ist systemabhängig; auf Ihrem Rechner liegen `python3`, `sh` und `cat` womöglich an etwas anderen Stellen, aber alle sollten gefunden werden. Das Beispiel verwendet `env=` von `subprocess` (Modul 5), das die Umgebungsvariablen des Kindprozesses *ersetzt*, nicht ergänzt.

**Ausgabe und Protokollierung.** Alles, was ein cron-Job auf `stdout` oder `stderr` ausgibt, wird gesammelt und nach dem Lauf als E-Mail an den Besitzer der crontab geschickt. Die Handbuchseite beschreibt: Ist die Variable `MAILTO` gesetzt und nicht leer, geht die Nachricht an diese Adresse; ist sie auf einen leeren Text gesetzt (`MAILTO=""`), wird *keine* Mail verschickt; ist sie nicht gesetzt, geht die Mail an den Besitzer. Auf den meisten Servern ist aber gar kein Mailprogramm eingerichtet. Dann verschwindet die Ausgabe stillschweigend, und ein fehlgeschlagener Job bleibt unbemerkt. Das ist die häufigste Ursache für das berüchtigte „der Backup-Job läuft seit Monaten nicht mehr, und niemand hat es gemerkt“. Die Regel lautet deshalb: Jeder cron-Job schreibt in eine *eigene Protokolldatei* (mit der Umleitung `>> protokoll.log 2>&1`, oder besser mit `logging` im Skript, Modul 6) und meldet Fehler über einen Exit-Code und, wo es darauf ankommt, über eine eigene Benachrichtigung (Modul 12).

**Das Prozentzeichen.** In einer crontab-Zeile hat das Zeichen `%` eine Sonderbedeutung. Laut Handbuchseite wird es, wenn es nicht mit einem Rückwärtsschrägstrich geschützt ist, in einen Zeilenumbruch verwandelt, und alles nach dem ersten `%` geht als Standardeingabe an den Befehl. Das trifft vor allem Aufrufe wie `date +%Y-%m-%d`, die in der Shell funktionieren und in der crontab *stillschweigend falsch* sind. In der crontab schreibt man `date +\%Y-\%m-\%d`. Wer das umgehen will, legt die Logik in ein Skript (wo `%` keine Sonderbedeutung hat) und trägt in die crontab nur den Skriptaufruf ein. Das ist auch die Empfehlung der Handbuchseite für lange Befehle: Sie rät, Zeilen kurz zu halten und für lange Befehle ein Skript zu verwenden. Die Befehlszeile darf bis zu 998 Zeichen lang sein.

**Zeitzone und Zeitumstellung.** Laut Handbuchseite arbeitet der cron-Dienst mit *einer* konfigurierten Zeitzone; Zeitzonen je Benutzer gibt es nicht, und eine Variable `TZ` in der crontab wirkt nur auf die gestarteten Befehle, nicht auf die Planung. Ein Eintrag `30 2 * * *` bezieht sich also auf die Ortszeit des Servers. Zur Sommerzeit macht die Seite keine Angaben. Aus allgemeinem Fachwissen (nicht nachgeschlagen) ist bekannt, dass Aufträge, die in die Stunde der Zeitumstellung fallen, je nach cron-Version ausfallen oder zweimal laufen können; im März entfällt die Stunde zwischen 02:00 und 03:00 Uhr, im Oktober gibt es sie zweimal (siehe 8.3). Praktische Konsequenz: Legen Sie wichtige Aufträge nicht in diese Zeitspanne, oder stellen Sie den Server auf UTC um, wie es viele Betreiber tun.

**Fehler beim Eintragen vermeiden.** Zwei weitere Hinweise aus der Handbuchseite: Jede Zeile der crontab muss mit einem Zeilenumbruch enden; fehlt er bei der letzten Zeile, kann cron die gesamte crontab als fehlerhaft ansehen und im Systemprotokoll warnen. Und die Befehle laufen mit `/usr/bin/sh` (oder dem Wert von `SHELL`), nicht notwendigerweise mit Bash; Bash-spezifische Schreibweisen wie `[[ ... ]]` funktionieren dort unter Umständen nicht. Schließlich sollten Sie Einträge niemals von Hand in die Dateien im Systemverzeichnis schreiben, ohne die Syntax zu prüfen; `crontab -e` prüft beim Speichern, `/etc/cron.d` nicht.

**Einträge aus einem Skript pflegen.** Wenn Sie Server automatisch einrichten (Modul 11), wollen Sie crontab-Einträge per Programm setzen. Dabei kommt es auf *Idempotenz* an, ein Begriff, den wir in 9.4 genau besprechen: Das Programm soll auch beim dritten Aufruf nicht drei Einträge anlegen, sondern genau einen. Eine bewährte Technik ist ein *markierter Block*: Das Programm schreibt seine Zeilen zwischen zwei Kommentarzeilen und ersetzt beim nächsten Lauf nur diesen Block. Die folgende Funktion arbeitet mit Text (sie verändert keine echte crontab) und lässt sich dadurch gefahrlos testen.

```python
def setze_block(crontab, name, zeilen):
    """Ersetzt den markierten Block 'name' in einer crontab oder fügt ihn an."""
    anfang = f"# BEGIN {name}"
    ende = f"# END {name}"
    vorhanden = crontab.splitlines()
    ergebnis, im_block = [], False
    for zeile in vorhanden:
        if zeile == anfang:
            im_block = True
        elif zeile == ende:
            im_block = False
        elif not im_block:
            ergebnis.append(zeile)
    ergebnis += [anfang, *zeilen, ende]
    return "\n".join(ergebnis) + "\n"

alt = "MAILTO=\"\"\n0 1 * * * /opt/skripte/aufraeumen.sh\n"
neu = setze_block(alt, "tagesreport", ["30 6 * * 1-5 /opt/skripte/tagesreport.py"])
print(neu)
zweimal = setze_block(neu, "tagesreport", ["30 6 * * 1-5 /opt/skripte/tagesreport.py"])
print("Zweiter Lauf ändert nichts:", zweimal == neu)
geaendert = setze_block(neu, "tagesreport", ["15 7 * * 1-5 /opt/skripte/tagesreport.py"])
print(geaendert.count("tagesreport.py"), "Eintrag nach Änderung")
```

```text
MAILTO=""
0 1 * * * /opt/skripte/aufraeumen.sh
# BEGIN tagesreport
30 6 * * 1-5 /opt/skripte/tagesreport.py
# END tagesreport

Zweiter Lauf ändert nichts: True
1 Eintrag nach Änderung
```

Der erste Lauf fügt den Block an die vorhandenen Zeilen an; der zweite Lauf mit denselben Zeilen ändert nichts (das Ergebnis ist gleich); bei einer Änderung wird der Block ersetzt, und es gibt weiterhin nur einen Eintrag. Eigene Zeilen außerhalb des Blocks (hier die Variable `MAILTO` und der Auftrag zum Aufräumen) bleiben unberührt. Wollte man den Block tatsächlich installieren, läse man die aktuelle crontab mit `subprocess.run(["crontab", "-l"], capture_output=True, text=True)` (bei einer leeren crontab liefert der Befehl einen Fehlercode, den man als „leer“ behandeln muss) und schriebe das Ergebnis mit `subprocess.run(["crontab", "-"], input=neu, text=True)` zurück.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler mit cron lassen sich in einer Liste zusammenfassen, die Sie vor jedem neuen Eintrag durchgehen können: relative Pfade und Programme, die nur im persönlichen `PATH` liegen; fehlende virtuelle Umgebung; ein unmaskiertes `%`; keine Protokolldatei, sodass Fehler verloren gehen; die Annahme, die Zeit sei UTC (sie ist die Ortszeit des Servers); zu enge Zeitabstände, bei denen ein Lauf noch läuft, wenn der nächste beginnt (siehe 9.4); und die Verknüpfung der beiden Tagesfelder. Hinzu kommt ein praktischer Fehler beim Testen: Wer einen Eintrag auf „jede Minute“ stellt, um ihn zu beobachten, und ihn vergisst, bekommt Hunderte Mails oder wächst die Protokolldatei. Der folgende Versuch zeigt zwei typische Fehler im Vergleich zur richtigen Variante, indem er die Zeilen als Text mit unserem Rechner auf ihre Laufzeit prüft.

```python
from datetime import datetime

import cronzeit

ab = datetime(2027, 1, 11, 9, 0)

faelle = {
    "FALSCH  jede Minute statt täglich": "* 3 * * *",
    "RICHTIG täglich um 03:00": "0 3 * * *",
    "FALSCH  Wochentag 0-7 vertauscht (Sonntag=1)": "0 3 * * 1",
    "RICHTIG Sonntag": "0 3 * * 0",
}
for beschreibung, ausdruck in faelle.items():
    laeufe = cronzeit.naechste_laeufe(ausdruck, ab, 60)
    erste = laeufe[0].strftime("%a %d.%m. %H:%M")
    am_selben_tag = sum(1 for t in laeufe if t.date() == laeufe[0].date())
    print(f"{beschreibung:46} erster Lauf {erste}, Läufe am ersten Tag: {am_selben_tag}")
```

```text
FALSCH  jede Minute statt täglich              erster Lauf Tue 12.01. 03:00, Läufe am ersten Tag: 60
RICHTIG täglich um 03:00                       erster Lauf Tue 12.01. 03:00, Läufe am ersten Tag: 1
FALSCH  Wochentag 0-7 vertauscht (Sonntag=1)   erster Lauf Mon 18.01. 03:00, Läufe am ersten Tag: 1
RICHTIG Sonntag                                erster Lauf Sun 17.01. 03:00, Läufe am ersten Tag: 1
```

Der erste Eintrag `* 3 * * *` sieht fast richtig aus, läuft aber in der Stunde 03 *jede Minute*, also 60 Mal; wer täglich um 03:00 Uhr meinte, hat ein Minutenfeld vergessen (`0`). Beim Wochentag ist 1 der Montag und nicht der Sonntag; Sonntag ist 0 oder 7. Ein Rechner wie der unsere (oder `systemd-analyze calendar` für systemd-Timer, siehe 9.3) macht solche Fehler vor dem Einsatz sichtbar.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentation von cron (abgerufen).** Die Debian-Ausgabe der Handbuchseite `crontab(5)` (<https://manpages.debian.org/testing/cron/crontab.5.en.html>) hält fest: fünf Zeitfelder mit den genannten Wertebereichen; Listen, Bereiche und Schritte; Kurzformen `@reboot`, `@yearly`, `@monthly`, `@weekly`, `@daily`, `@hourly`; `SHELL` ist `/usr/bin/sh`, `LOGNAME` und `HOME` kommen aus `/etc/passwd`; Variablen werden nicht expandiert; `MAILTO` steuert den Mailversand der Ausgabe, ein leerer Wert unterdrückt ihn; Prozentzeichen werden zu Zeilenumbrüchen, wenn sie nicht mit einem Rückwärtsschrägstrich geschützt sind; sind beide Tagesfelder eingeschränkt, genügt, dass eines passt; die Debian-Implementierung prüft nur, ob ein Feld mit einem Stern *beginnt* (daher läuft `0 0 */2 * sun` nur an Sonntagen mit ungeradem Datum, während POSIX jeden Sonntag und jedes ungerade Datum vorsähe); cron prüft einmal pro Minute und kennt keine Sekunden; es gibt nur eine Zeitzone für den Dienst; die Befehlszeile darf 998 Zeichen lang sein. *Beobachtet in der Prüfumgebung* (Debian/Ubuntu-cron): Ein per cron gestarteter Job sah `HOME`, `LOGNAME`, `PATH=/usr/bin:/bin`, `PWD`, `SHELL=/bin/sh` und `LANG`. Das Verhalten von `crontab -r`, die Verzeichnisse `/etc/cron.d` und `/etc/cron.daily` sowie das Verhalten bei der Zeitumstellung beruhen auf Fachwissen und sind nicht nachgeschlagen.

> **Merke**
>
> cron prüft jede Minute die crontab: fünf Felder (Minute, Stunde, Tag, Monat, Wochentag) plus Befehl. Absolute Pfade verwenden (Interpreter, Skript, venv). Die Umgebung ist minimal (kurzer `PATH`, keine `.bashrc`). Ausgabe wird per Mail zugestellt oder geht verloren, also immer in eine Protokolldatei umleiten oder selbst protokollieren. `%` mit Rückwärtsschrägstrich schützen. Zeit ist die Ortszeit des Servers. Sind Tag und Wochentag beide eingeschränkt, genügt einer von beiden. `crontab -e` prüft die Syntax.

> **Praxisbeispiel**
>
> Fast jedes Produktionssystem hat cron-Einträge für Backups, Aufräumarbeiten, Berichte und Zertifikatserneuerung. Beim Übernehmen eines fremden Servers lesen Sie als Erstes die crontabs aller Benutzer (`crontab -l -u benutzer` als Root) und die Dateien in `/etc/cron.d`: Dort stehen die Aufgaben, von denen der Server lebt. Ein Skript, das lokal einwandfrei lief und im cron-Betrieb nicht, ist ein Klassiker; die Regeln dieser Lerneinheit (volle Pfade, minimale Umgebung, Protokoll) lösen die meisten dieser Fälle.

## 9.3 systemd-Timer und -Services

Auf modernen Linux-Distributionen ist *systemd* das Init-System, also das Programm mit der Prozessnummer 1, das alle anderen Dienste startet und überwacht. systemd kennt neben Diensten auch *Timer*, die zeitgesteuert andere Einheiten starten, und ist damit eine vollwertige Alternative zu cron. Die Einheiten heißen *Units*; sie sind kleine Konfigurationsdateien im INI-Format (Abschnitte in eckigen Klammern, darunter *Schlüssel*`=`*Wert*-Zeilen, vergleiche Modul 8). Für einen zeitgesteuerten Auftrag braucht man zwei Units mit gleichem Namen: eine `.service`-Datei, die beschreibt, *was* läuft, und eine `.timer`-Datei, die beschreibt, *wann*. Der Timer startet die gleichnamige Service-Unit, sofern man nichts anderes angibt.

Warum der Aufwand, wenn cron eine Zeile genügt? Weil systemd mehrere Schwächen von cron beseitigt. Die Ausgabe des Jobs landet automatisch im *Journal* (dem zentralen Protokoll von systemd) und lässt sich mit `journalctl -u tagesreport.service` lesen, ohne dass man Umleitungen schreiben muss. Der Job läuft in einer wohldefinierten Umgebung, die man in der Unit-Datei festlegt (Benutzer, Arbeitsverzeichnis, Umgebungsvariablen), statt von einem zufälligen Minimal-`PATH` auszugehen. Ein Lauf, der noch aktiv ist, wird nicht ein zweites Mal gestartet, solange die Service-Unit aktiv ist; und mit `Persistent=true` holt systemd einen verpassten Lauf nach, wenn der Rechner zur geplanten Zeit ausgeschaltet war. Der Preis ist, dass es mehr Dateien sind und dass die Syntax für Zeitangaben eine andere ist als bei cron.

**Die Service-Unit.** Beginnen wir mit der Service-Datei für unseren späteren Tagesreport (9.5). Sie besteht aus den Abschnitten `[Unit]` und `[Service]`. Im Abschnitt `[Unit]` steht eine Beschreibung, die in Listen und Protokollen erscheint. Im Abschnitt `[Service]` legt `Type=oneshot` fest, dass der Dienst eine Aufgabe erledigt und sich dann beendet; systemd betrachtet ihn erst nach dem Ende des Prozesses als fertig. Das ist die richtige Wahl für Skripte, die eine Aufgabe abarbeiten, im Gegensatz zu Daemons, die dauerhaft laufen. `ExecStart` nennt den Befehl. Wichtig: Die Zeile wird *nicht von einer Shell* interpretiert. Pipes, Umleitungen, Variablenexpansion mit Dollarzeichen und mehrere Befehle mit Semikolon funktionieren dort nicht wie in der Shell; der Programmpfad muss absolut sein. Wer Shell-Syntax braucht, ruft ausdrücklich `/bin/sh -c '...'` auf, besser noch: legt die Logik in das Python-Skript. Mit `User=` wählt man das Benutzerkonto, mit `WorkingDirectory=` das Arbeitsverzeichnis (damit relative Pfade im Skript nicht vom Zufall abhängen).

```ini
# /etc/systemd/system/tagesreport.service
[Unit]
Description=Tagesreport erzeugen

[Service]
Type=oneshot
User=report
WorkingDirectory=/opt/tagesreport
ExecStart=/opt/tagesreport/.venv/bin/python /opt/tagesreport/tagesreport.py --ausgabe /var/lib/tagesreport
```

Hier läuft das Skript unter dem Benutzer `report` (der angelegt sein muss), mit dem Interpreter der virtuellen Umgebung aus 9.1, und schreibt seine Ergebnisse in ein Verzeichnis, auf das dieser Benutzer Schreibrechte hat. Alles, was das Skript auf `stdout` und `stderr` ausgibt, erscheint im Journal; dort wird auch der Exit-Code festgehalten. Beendet sich das Skript mit einem Wert ungleich null, gilt die Unit als *fehlgeschlagen* (`failed`), und das sieht man mit `systemctl status tagesreport.service` sofort. Das ist ein großer Vorteil gegenüber cron, bei dem ein Fehlschlag ohne Mailprogramm unsichtbar bleibt. Deshalb ist es wichtig, dass Ihre Skripte (siehe Modul 6) bei Fehlern einen Exit-Code ungleich null liefern.

**Die Timer-Unit.** Die Timer-Datei hat neben `[Unit]` die Abschnitte `[Timer]` und `[Install]`. Im Abschnitt `[Timer]` steht, wann gestartet wird. `OnCalendar=` nimmt einen Kalenderausdruck, `Persistent=true` lässt verpasste Läufe nachholen (systemd merkt sich dazu den Zeitpunkt des letzten Laufs auf der Festplatte), und `RandomizedDelaySec=` verteilt den Start zufällig über ein Zeitfenster. Die letzte Option ist nützlich, wenn viele Rechner zur selben Zeit denselben Server abfragen würden; ein Versatz von zwei Minuten verhindert den Ansturm. Im Abschnitt `[Install]` steht `WantedBy=timers.target`: Das bedeutet, dass der Timer beim Aktivieren (`enable`) in die Gruppe der beim Systemstart aktiven Timer eingehängt wird.

```ini
# /etc/systemd/system/tagesreport.timer
[Unit]
Description=Tagesreport werktags um 06:30

[Timer]
OnCalendar=Mon..Fri 06:30
Persistent=true
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
```

Nach dem Anlegen der beiden Dateien sagt man systemd, dass sich Konfigurationsdateien geändert haben (`systemctl daemon-reload`), und aktiviert den *Timer* (nicht den Service): `systemctl enable --now tagesreport.timer`. Das `--now` startet ihn zugleich sofort. Mit `systemctl list-timers` sieht man alle Timer samt dem nächsten und dem letzten Lauf, und mit `systemctl start tagesreport.service` kann man den Auftrag zum Testen von Hand auslösen, ohne den Zeitpunkt abzuwarten. Dieser Handstart ist ein großer Vorteil beim Entwickeln: Sie testen genau die Umgebung, in der später der Timer das Skript startet. Diese Befehle verändern ein laufendes System und sind in unserer Prüfumgebung *nicht ausgeführt* worden, weil dort kein systemd als Init-Prozess läuft; geprüft haben wir die Dateien selbst mit `systemd-analyze`, wie gleich gezeigt.

**Kalenderausdrücke.** Die Syntax von `OnCalendar` lautet in ihrer allgemeinen Form `Wochentag Jahr-Monat-Tag Stunde:Minute:Sekunde`; jeder Teil kann entfallen oder ein Stern sein. Anders als bei cron sind Sekunden möglich, Wochentage schreibt man als englische Abkürzungen (`Mon`, `Tue`, …), und Bereiche notiert man mit zwei Punkten (`Mon..Fri`). Schritte schreibt man mit einem Schrägstrich (`*:0/15` bedeutet alle 15 Minuten ab Minute 0). Es gibt außerdem Kurzformen wie `hourly`, `daily`, `weekly`, `monthly`. Der Wochentag und das Datum werden bei systemd mit UND verknüpft (cron verknüpft sie mit ODER, siehe 9.2): `Mon *-*-01` meint nur Montage, die zugleich der Erste des Monats sind. Und die Kurzform `weekly` bedeutet bei systemd Montag 00:00 Uhr, während `@weekly` bei cron Sonntag 00:00 Uhr bedeutet; kleine Unterschiede, die leicht übersehen werden.

Die große Stärke von systemd: Man muss einen Ausdruck nicht raten, sondern kann ihn prüfen. `systemd-analyze calendar "Mon..Fri 06:30"` zeigt die normalisierte Schreibweise und den nächsten Zeitpunkt; `systemd-analyze verify` prüft Unit-Dateien auf Fehler. Beides lässt sich aus Python heraus aufrufen, etwa um beim Einrichten eines Servers automatisch zu testen. Das folgende Programm erzeugt die beiden Dateien in einem temporären Ordner (mit dem aktuell laufenden Interpreter und einem Mini-Skript als Ziel von `ExecStart`, damit die Prüfung auf eine existierende ausführbare Datei positiv ausfällt), lässt sie prüfen und normalisiert einige Ausdrücke. Dass zeitabhängige Ausgaben wie `Next elapse` fehlen, ist Absicht: Sie würden sich bei jedem Lauf ändern.

```python
import subprocess
import sys
import tempfile
from pathlib import Path

SERVICE = """[Unit]
Description=Tagesreport erzeugen

[Service]
Type=oneshot
WorkingDirectory=/tmp
ExecStart={python} {skript} --ausgabe /tmp
"""

TIMER = """[Unit]
Description=Tagesreport werktags um 06:30

[Timer]
OnCalendar=Mon..Fri 06:30
Persistent=true
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
"""

with tempfile.TemporaryDirectory() as ordner:
    ordner = Path(ordner)
    skript = ordner / "tagesreport.py"
    skript.write_text("print('Report')\n", encoding="utf-8")
    skript.chmod(0o755)
    (ordner / "tagesreport.service").write_text(
        SERVICE.format(python=sys.executable, skript=skript), encoding="utf-8")
    (ordner / "tagesreport.timer").write_text(TIMER, encoding="utf-8")

    r = subprocess.run(["systemd-analyze", "verify",
                        str(ordner / "tagesreport.service"), str(ordner / "tagesreport.timer")],
                       capture_output=True, text=True)
    meldungen = [z for z in r.stderr.splitlines() if "tagesreport" in z]
    print("verify Rückgabecode:", r.returncode, "| Meldungen zu den Units:", meldungen or "keine")

    for ausdruck in ("Mon..Fri 06:30", "daily", "weekly", "*:0/15", "Mon..Fri 25:00"):
        r = subprocess.run(["systemd-analyze", "calendar", ausdruck],
                           capture_output=True, text=True)
        text = r.stdout + r.stderr
        normal = [z.split(":", 1)[1].strip() for z in text.splitlines() if "Normalized form" in z]
        print(f"{ausdruck:16} ->", normal[0] if normal else "ungültig")
```

```text
verify Rückgabecode: 0 | Meldungen zu den Units: keine
Mon..Fri 06:30   -> Mon..Fri *-*-* 06:30:00
daily            -> *-*-* 00:00:00
weekly           -> Mon *-*-* 00:00:00
*:0/15           -> *-*-* *:00/15:00
Mon..Fri 25:00   -> ungültig
```

Die Prüfung der Units meldet nichts, der Rückgabecode ist null. Die Ausdrücke werden normalisiert: `Mon..Fri 06:30` wird zu `Mon..Fri *-*-* 06:30:00` (jeder Tag und jeder Monat, Sekunde null), `daily` zu Mitternacht, und `weekly` zu Montag Mitternacht. Der letzte Ausdruck mit der Stunde 25 ist ungültig, und `systemd-analyze` lehnt ihn ab. Das Programm benutzt `tempfile.TemporaryDirectory` (Modul 4), damit nichts auf der Festplatte zurückbleibt. Sollten Sie das Beispiel selbst ausführen und der Befehl `systemd-analyze` fehlt, ist das kein Fehler Ihres Programms: Auf Systemen ohne systemd gibt es das Werkzeug nicht, und `subprocess.run` löst dann eine `FileNotFoundError` aus (Modul 5).

**Benutzer-Units, Umgebung und Abhängigkeiten.** Nicht jeder Auftrag soll als Root laufen. systemd kennt neben den systemweiten Units (unter `/etc/systemd/system`) auch *Benutzer-Units*, die ein normaler Benutzer unter `~/.config/systemd/user` ablegt und mit `systemctl --user` steuert. Damit lassen sich persönliche Aufgaben ohne Administratorrechte planen (das Verhalten, dass solche Timer nur laufen, solange der Benutzer angemeldet ist, sofern man nicht `loginctl enable-linger` setzt, ist Fachwissen und hier nicht geprüft). Umgebungsvariablen setzt man in der Service-Unit mit `Environment=NAME=wert` oder liest sie mit `EnvironmentFile=/etc/tagesreport.env` aus einer Datei; das ist der saubere Ort für Zugangsdaten, die nicht im Skript stehen sollen (die Datei erhält Rechte `0600`, siehe 9.1 und Modul 10). Wartet ein Auftrag auf das Netzwerk, ergänzt man im Abschnitt `[Unit]` die Zeilen `After=network-online.target` und `Wants=network-online.target`; ob das auf einem bestimmten System zuverlässig funktioniert, hängt von der Netzwerkverwaltung ab.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, den *Service* statt des *Timers* zu aktivieren: Dann läuft der Auftrag einmal beim Hochfahren oder gar nicht, aber nie nach Zeitplan. Der zweite ist, nach einer Änderung an einer Unit-Datei `systemctl daemon-reload` zu vergessen; systemd verwendet dann die alte Fassung. Der dritte betrifft `ExecStart`, wie oben erwähnt: Shell-Syntax und relative Pfade funktionieren nicht. Der folgende Vergleich zeigt zwei Zeilen, von denen die erste in der Shell funktionieren würde und in systemd nicht.

```text
# FALSCH: Umleitung und relativer Pfad werden nicht von einer Shell ausgewertet
ExecStart=python3 tagesreport.py >> /var/log/tagesreport.log 2>&1

# RICHTIG: absolute Pfade, Protokoll geht automatisch ins Journal
ExecStart=/usr/bin/python3 /opt/tagesreport/tagesreport.py

# RICHTIG, wenn Shell-Syntax wirklich nötig ist
ExecStart=/bin/sh -c '/usr/bin/python3 /opt/tagesreport/tagesreport.py >> /var/log/tagesreport.log 2>&1'
```

Bei der ersten Zeile wird `>>` nicht als Umleitung verstanden, sondern als ein weiteres Argument an das Skript übergeben, und `python3` ohne Pfad wird von systemd nicht über einen Suchpfad gefunden (systemd verlangt für den Befehl einen absoluten Pfad). Die zweite Zeile ist die übliche Lösung, weil das Journal die Ausgabe ohnehin sammelt. Die dritte Zeile zeigt den Notbehelf. Ein vierter Fehler betrifft den Zeitausdruck: Wer `OnCalendar=*-*-* 06:30` schreibt und dabei den Zeitzonenfaktor vergisst, hat einen Lauf in der Ortszeit des Servers. Ist ein Server auf UTC gestellt, ist 06:30 nicht 06:30 in Wien; systemd erlaubt eine Zeitzone am Ende des Ausdrucks (zum Beispiel `Europe/Vienna`), was hier nicht geprüft wurde und in der Dokumentation nachgelesen werden sollte.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentation zu systemd (abgerufen).** Die Handbuchseiten `systemd.timer(5)`, `systemd.service(5)` und `systemd.time(7)` (Quellen im Verzeichnis am Ende des Skripts) beschreiben die Optionen `OnCalendar`, `Persistent`, `RandomizedDelaySec` und `AccuracySec` sowie die Typen `oneshot` und `simple`, die Auswertung von `ExecStart` ohne Shell und die Syntax der Kalenderausdrücke. *Selbst geprüft* in der Prüfumgebung: die oben gezeigten Unit-Dateien mit `systemd-analyze verify`, die Ausdrücke mit `systemd-analyze calendar`; das Starten und Aktivieren von Timern war nicht möglich, weil dort kein systemd als Init-Prozess läuft. Auf Fachwissen beruhen: das Verhalten, dass ein noch aktiver Service nicht erneut gestartet wird, `loginctl enable-linger` und die Angaben zu `network-online.target`; bitte prüfen Sie diese Punkte auf Ihrem System.

> **Merke**
>
> Ein zeitgesteuerter Auftrag in systemd besteht aus einer Service-Unit (was) und einer Timer-Unit (wann) mit gleichem Namen. `Type=oneshot`, absolute Pfade, kein Shell-Syntax in `ExecStart`. `Persistent=true` holt verpasste Läufe nach. Den *Timer* aktivieren (`enable --now`), nach Änderungen `daemon-reload`. Prüfen mit `systemd-analyze verify` und `systemd-analyze calendar`; Protokoll mit `journalctl -u`.

> **Praxisbeispiel**
>
> Auf aktuellen Servern ersetzen systemd-Timer zunehmend cron-Einträge. Beim Betrieb eines Skripts sehen Sie mit `systemctl list-timers`, wann es zuletzt und wann es als Nächstes läuft, und mit `journalctl -u tagesreport.service --since today`, was es ausgegeben hat. Fehlgeschlagene Units fallen in der Übersicht `systemctl --failed` sofort auf. Bei der Übernahme eines Servers lohnt sich deshalb ein Blick auf beide Zeitplaner: cron *und* systemd-Timer.

## 9.4 Locking und Idempotenz

Zwei Eigenschaften unterscheiden ein Skript, das *einmal von Hand* läuft, von einem, das *unbeaufsichtigt und wiederholt* läuft: Es darf sich nicht selbst in die Quere kommen, und es muss gefahrlos mehrfach ausgeführt werden können. Die erste Eigenschaft heißt *Locking* (Sperren), die zweite *Idempotenz*. Beide sind unspektakulär, wenn man sie beachtet, und verursachen Datenverlust oder nächtliche Alarme, wenn man sie vergisst.

Betrachten wir zunächst das Problem. Ein Backup-Skript wird von cron jede Stunde gestartet und braucht normalerweise zehn Minuten. Eines Tages ist das Ziel langsam, und der Lauf dauert 70 Minuten. Zur vollen Stunde startet cron den nächsten Lauf, obwohl der erste noch arbeitet; beide schreiben in dasselbe Zielverzeichnis, beide verlangsamen sich gegenseitig, und im schlimmsten Fall zerstören sie sich die Daten. cron kennt keine Regel „starte nicht, wenn der letzte Lauf noch läuft“, es startet stur zur angegebenen Zeit. Die Lösung ist eine *Sperre*: Der erste Lauf besetzt sie, der zweite stellt beim Start fest, dass sie besetzt ist, und beendet sich sofort und leise.

**Sperren mit flock.** Das Betriebssystem bietet für diesen Zweck den Aufruf `flock` an: Man öffnet eine Datei (die *Sperrdatei*, meist mit der Endung `.lock`) und bittet das Betriebssystem, sie exklusiv zu sperren. Solange ein Prozess die Sperre hält, scheitert jeder andere Versuch, sie zu bekommen. Der entscheidende Vorteil gegenüber selbstgebauten Lösungen: Das Betriebssystem gibt die Sperre automatisch frei, wenn der Prozess endet, auch wenn er abstürzt oder gewaltsam beendet wird. Es kann also keine „vergessene“ Sperre geben, die alle künftigen Läufe blockiert. Python bietet den Aufruf im Modul `fcntl` (nur unter Unix; auf Windows gibt es dafür andere Wege). Mit dem Schalter `LOCK_NB` (non-blocking) wartet der Aufruf nicht, bis die Sperre frei wird, sondern löst sofort eine `BlockingIOError` aus, wenn sie besetzt ist.

Das folgende Beispiel kapselt das in einen Kontextmanager (Modul 4: `with`), damit die Sperre garantiert freigegeben wird. Die Funktion `exklusiv` öffnet die Sperrdatei im Modus `"a+"` (anhängen und lesen, die Datei wird bei Bedarf angelegt, aber nicht geleert), fordert die exklusive Sperre an und übersetzt den Fehler in eine eigene Ausnahme mit verständlicher Meldung. Das `finally` schließt die Datei, was die Sperre löst. Zum Test verwenden wir zwei *verschachtelte* Sperren in demselben Programm. Das funktioniert als Demonstration, weil `flock` an die geöffnete Datei und nicht an den Prozess gebunden ist: Der zweite `open` erzeugt eine unabhängige Sperrbeziehung, die mit der ersten kollidiert.

```python
import fcntl
import tempfile
from contextlib import contextmanager
from pathlib import Path


class SchonGesperrt(Exception):
    """Ein anderer Lauf hält die Sperre bereits."""


@contextmanager
def exklusiv(sperrdatei):
    datei = open(sperrdatei, "a+")
    try:
        try:
            fcntl.flock(datei, fcntl.LOCK_EX | fcntl.LOCK_NB)
        except BlockingIOError:
            raise SchonGesperrt(f"{Path(sperrdatei).name} ist gesperrt") from None
        yield
    finally:
        datei.close()          # gibt die Sperre frei


sperre = Path(tempfile.mkdtemp()) / "demo.lock"

with exklusiv(sperre):
    print("Erster Lauf hat die Sperre.")
    try:
        with exklusiv(sperre):
            print("Das sollte nicht passieren.")
    except SchonGesperrt as fehler:
        print("Zweiter Lauf:", fehler)

with exklusiv(sperre):
    print("Nach Freigabe klappt es wieder.")
```

```text
Erster Lauf hat die Sperre.
Zweiter Lauf: demo.lock ist gesperrt
Nach Freigabe klappt es wieder.
```

In der Ausgabe sieht man die drei Phasen: Der erste Lauf bekommt die Sperre, der zweite (innen) wird abgewiesen, und nachdem der äußere Block verlassen wurde, ist die Sperre wieder frei. Das `from None` in der Ausnahmebehandlung unterdrückt die Anzeige der ursprünglichen `BlockingIOError`, damit die Fehlermeldung für den Anwender kurz bleibt. Beachten Sie die Wahl des Ortes für die Sperrdatei: Der Pfad muss für alle Läufe derselbe sein und für den ausführenden Benutzer beschreibbar. Üblich sind `/run/lock` oder `/var/lock` für Systemdienste, `/tmp` für schnelle Skripte; im `/tmp` können sich Benutzer allerdings gegenseitig Dateien vor die Füße legen, und das Verzeichnis hat daher nur für private Skripte einen Platz.

**Automatische Freigabe bei Abstürzen.** Dass die Freigabe auch bei Abstürzen geschieht, sollte man einmal gesehen haben, denn auf dieser Eigenschaft beruht der ganze Nutzen. Das folgende Programm startet einen Kindprozess, der die Sperre besetzt und dann mit `os._exit(3)` abrupt endet (ohne Aufräumen, wie bei einem Absturz). Danach versucht das Hauptprogramm, die Sperre zu bekommen, und es gelingt.

```python
import fcntl
import subprocess
import sys
import tempfile
from pathlib import Path

sperre = Path(tempfile.mkdtemp()) / "absturz.lock"
kind = """
import fcntl, os, sys
datei = open(sys.argv[1], "a+")
fcntl.flock(datei, fcntl.LOCK_EX | fcntl.LOCK_NB)
os.write(1, b"Kind hat die Sperre und stuerzt jetzt ab\\n")
os._exit(3)
"""
r = subprocess.run([sys.executable, "-c", kind, str(sperre)], capture_output=True, text=True)
print(r.stdout.strip(), "| Exit-Code", r.returncode)

with open(sperre, "a+") as datei:
    try:
        fcntl.flock(datei, fcntl.LOCK_EX | fcntl.LOCK_NB)
        print("Sperre ist frei: das Betriebssystem hat sie beim Ende des Kindes freigegeben")
    except BlockingIOError:
        print("Sperre haengt")
```

```text
Kind hat die Sperre und stuerzt jetzt ab | Exit-Code 3
Sperre ist frei: das Betriebssystem hat sie beim Ende des Kindes freigegeben
```

Genau das leistet eine einfache Datei als „Besetzt“-Marke nicht: Würde das Kind eine Datei `laeuft` anlegen und beim Absturz nicht löschen, bliebe sie liegen und würde alle weiteren Läufe für immer blockieren. Das führt zur zweiten Technik.

**PID-Datei mit O_EXCL.** Oft sieht man Skripte, die beim Start eine *PID-Datei* anlegen (die Prozessnummer, kurz PID) und beim Ende löschen. Damit das Anlegen selbst nicht zu einem Wettlauf wird (zwei Läufe prüfen gleichzeitig „existiert nicht“ und legen beide an), verwendet man `os.open` mit den Schaltern `O_CREAT | O_EXCL`: Der Aufruf gelingt nur, wenn die Datei noch nicht existiert, und das Prüfen und Anlegen geschieht in einem einzigen, unteilbaren Schritt (siehe 9.1). Das Problem sind veraltete Dateien nach einem Absturz; deshalb muss man beim Fund einer vorhandenen Datei prüfen, ob der darin genannte Prozess noch lebt. Das erledigt `os.kill(pid, 0)`: Das Signal 0 sendet nichts, löst aber dieselben Fehler aus wie ein echtes Signal, nämlich `ProcessLookupError`, wenn es den Prozess nicht gibt, und `PermissionError`, wenn er jemand anderem gehört (er lebt dann also).

```python
import os
import tempfile
from pathlib import Path


def pid_lebt(pid):
    try:
        os.kill(pid, 0)          # Signal 0 prueft nur, ob der Prozess existiert
    except ProcessLookupError:
        return False
    except PermissionError:
        return True              # existiert, gehoert aber jemand anderem
    return True


def pidfile_anlegen(pfad):
    pfad = Path(pfad)
    for versuch in range(2):
        try:
            fd = os.open(pfad, os.O_CREAT | os.O_EXCL | os.O_WRONLY, 0o644)
        except FileExistsError:
            try:
                alt = int(pfad.read_text().strip())
            except ValueError:
                alt = None
            if alt is not None and pid_lebt(alt):
                raise RuntimeError(f"Läuft bereits (PID {alt})") from None
            print(f"Veraltete PID-Datei von {alt} wird entfernt")
            pfad.unlink()
            continue
        with os.fdopen(fd, "w") as datei:
            datei.write(f"{os.getpid()}\n")
        return
    raise RuntimeError("PID-Datei konnte nicht angelegt werden")


pfad = Path(tempfile.mkdtemp()) / "demo.pid"
pidfile_anlegen(pfad)
print("Angelegt:", pfad.read_text().strip() == str(os.getpid()))
try:
    pidfile_anlegen(pfad)
except RuntimeError as fehler:
    print("Zweiter Versuch:", fehler.args[0].split("(")[0].strip())
pfad.write_text("999999\n")         # PID, die es (hoffentlich) nicht gibt
pidfile_anlegen(pfad)
print("Danach:", pfad.read_text().strip() == str(os.getpid()))
pfad.unlink()
```

```text
Angelegt: True
Zweiter Versuch: Läuft bereits
Veraltete PID-Datei von 999999 wird entfernt
Danach: True
```

Das Programm legt die Datei an, scheitert beim zweiten Versuch mit „Läuft bereits“ und erkennt eine veraltete Datei mit einer nicht vorhandenen PID, entfernt sie und legt eine neue an. Die Schleife `for versuch in range(2)` erlaubt genau einen zweiten Anlauf nach dem Entfernen der veralteten Datei. Diese Technik hat Schwächen: Die Prozessnummern werden von Linux wiederverwendet, sodass eine veraltete PID zufällig zu einem fremden, neuen Prozess gehören kann; zwischen `unlink` und erneutem Anlegen kann ein anderer Lauf dazwischenkommen. Deshalb ist `flock` die bessere Wahl für das reine Aussperren; die PID-Datei ist nützlich, wenn man zusätzlich wissen will, *wer* läuft (etwa um ihn gezielt zu beenden), oder auf Systemen ohne `fcntl`.

**Der Befehl flock in der Shell.** Man braucht für die Sperre gar kein Python: Das Werkzeug `flock(1)` aus dem Paket util-linux verpackt jeden Befehl in eine Sperre. In der crontab oder in einer systemd-Unit schreibt man `flock -n /run/lock/tagesreport.lock /opt/tagesreport/tagesreport.py`. Der Schalter `-n` lässt den Befehl sofort mit Exit-Code 1 scheitern, wenn die Sperre besetzt ist, `-w Sekunden` wartet höchstens so lange, und `-E Zahl` ändert den Exit-Code für den Fall „gesperrt“, damit man ihn von einem Fehler des Befehls selbst unterscheiden kann. Wir haben die drei Varianten in der Prüfumgebung ausprobiert, während ein anderer Prozess die Sperre für zwei Sekunden hielt.

```bash
$ flock -n demo.lock -c 'sleep 2' &
$ flock -n demo.lock -c 'echo drin'; echo "rc=$?"
rc=1
$ flock -n -E 75 demo.lock -c 'echo drin'; echo "rc=$?"
rc=75
$ flock -w 5 demo.lock -c 'echo spaeter drin'; echo "rc=$?"
spaeter drin
rc=0
```

Der erste Aufruf scheitert mit dem Code 1, der zweite mit dem selbst gewählten Code 75, und der dritte wartet, bis die Sperre frei ist, und führt dann den Befehl aus. Dieses Wartenlassen mit `-w` ist sinnvoll, wenn zwei Aufgaben nacheinander laufen sollen, statt sich gegenseitig auszuschließen. Für Auftragsplaner, die die Fehlercodes protokollieren, ist ein eigener Code wie 75 nützlich: Er zeigt im Protokoll eindeutig „übersprungen, weil schon aktiv“, nicht „fehlgeschlagen“.

**Idempotenz.** Ein Vorgang ist *idempotent*, wenn mehrmaliges Ausführen dasselbe Ergebnis liefert wie einmaliges. Das Wort stammt aus der Mathematik, bedeutet für Skripte aber etwas sehr Praktisches: Man darf das Skript ohne Angst wiederholen. Das ist aus mehreren Gründen wichtig. Läuft ein Job zweimal (weil die Sperre fehlte, weil er nach einem Fehler automatisch wiederholt wurde, weil jemand ihn zur Sicherheit noch einmal startet), darf nichts Schlimmes passieren. Wird ein Lauf mitten im Betrieb abgebrochen, muss der nächste Lauf sauber fortsetzen können. Und beim Einrichten von Servern (Modul 11) will man ein Skript gegen eine ganze Flotte laufen lassen, bei der einige Rechner schon fertig konfiguriert sind. Typische Verletzungen sind das Anhängen an eine Datei (jeder Lauf fügt dieselbe Zeile erneut ein), das Anlegen eines Ordners ohne `exist_ok=True`, die fortlaufende Nummerierung von Ausgaben und das „Löschen und neu Anlegen“ ohne Absicherung. Die Gegenmittel sind: *Zielzustand beschreiben statt Schritte ausführen* („die Datei soll diesen Inhalt haben“, nicht „hänge diese Zeile an“), vor dem Schreiben prüfen, ob schon alles stimmt, und Dateien *atomar* schreiben.

Atomar heißt: Ein Leser sieht entweder die komplette alte oder die komplette neue Fassung, niemals eine halbfertige. Unter Linux erreicht man das, indem man die neue Fassung in eine temporäre Datei *im selben Verzeichnis* schreibt, auf die Platte zwingt (`os.fsync`) und dann mit `os.replace` über das Ziel schiebt; das Umbenennen innerhalb eines Dateisystems ist unteilbar. Würde man dagegen direkt in die Zieldatei schreiben und stürbe das Skript mittendrin, bliebe eine halbe Datei zurück. Die Temporärdatei muss im selben Verzeichnis liegen, weil das Umbenennen zwischen verschiedenen Dateisystemen nicht atomar möglich ist (Python meldet dort einen Fehler). Das Programm zeigt beides: das atomare Schreiben und die Idempotenz durch Vergleich des Inhalts.

```python
import os
import tempfile
from pathlib import Path


def schreibe_atomar(ziel, text):
    ziel = Path(ziel)
    fd, temp = tempfile.mkstemp(dir=ziel.parent, prefix=ziel.name + ".")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as datei:
            datei.write(text)
            datei.flush()
            os.fsync(datei.fileno())
        os.replace(temp, ziel)
    except BaseException:
        Path(temp).unlink(missing_ok=True)
        raise


def stelle_sicher(ziel, text):
    ziel = Path(ziel)
    if ziel.exists() and ziel.read_text(encoding="utf-8") == text:
        return "unverändert"
    schreibe_atomar(ziel, text)
    return "geschrieben"


ziel = Path(tempfile.mkdtemp()) / "motd.txt"
for text in ("Willkommen\n", "Willkommen\n", "Wartung ab 22 Uhr\n"):
    print(stelle_sicher(ziel, text))
print(sorted(p.name for p in ziel.parent.iterdir()))
```

```text
geschrieben
unverändert
geschrieben
['motd.txt']
```

Der erste Aufruf schreibt, der zweite stellt fest, dass der Inhalt bereits stimmt, und tut nichts (die Datei wird nicht einmal angefasst, ihre Änderungszeit bleibt gleich); der dritte schreibt den neuen Text. Zum Schluss liegt nur die Zieldatei im Ordner, keine Temporärdatei. Das `except BaseException` im Hilfscode räumt auch bei einem Abbruch durch Strg+C auf und löst die Ausnahme danach erneut aus (`raise`). Dieselbe Idee steckt hinter vielen Werkzeugen: Konfigurationsverwaltungen wie Ansible arbeiten nach genau diesem Muster („stelle sicher, dass die Datei so aussieht“) und melden je Aufgabe „geändert“ oder „unverändert“.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, gar keine Sperre zu verwenden und darauf zu vertrauen, dass „der Lauf ja nur zehn Minuten dauert“. Ein zweiter Fehler: Die Sperre wird auf eine Datei gelegt, die das Skript selbst überschreibt, ersetzt oder löscht. Ersetzt man die Sperrdatei durch `os.replace`, hält der alte Lauf die Sperre auf einer Datei, die es unter diesem Namen nicht mehr gibt, und ein neuer Lauf sperrt die neue Datei; beide laufen gleichzeitig. Sperrdateien bleiben deshalb dauerhaft liegen und werden nie gelöscht oder ersetzt. Ein dritter Fehler ist `open(sperre, "w")` statt `"a+"`, was die Sperrdatei bei jedem Lauf leert; harmlos, solange nichts darin steht, aber schlechte Gewohnheit. Der vierte betrifft die Sperre über Netzwerkdateisysteme wie NFS: Dort ist das Verhalten von `flock` nicht zuverlässig (Fachwissen, hier nicht geprüft); für Sperren über mehrere Rechner braucht man eine Datenbank oder einen Koordinationsdienst.

```python
# FALSCH: haengt bei jedem Lauf dieselbe Zeile an
with open("/etc/hosts", "a") as f:
    f.write("10.0.0.5 server\n")

# RICHTIG: nur schreiben, wenn die Zeile fehlt
zeile = "10.0.0.5 server"
inhalt = open("/etc/hosts").read().splitlines()
if zeile not in inhalt:
    with open("/etc/hosts", "a") as f:
        f.write(zeile + "\n")
```

Im falschen Beispiel steht nach dem zehnten Lauf die Zeile zehnmal in der Datei. Im richtigen Beispiel wird zuerst gelesen und nur bei Bedarf geschrieben. Die richtige Variante ist nicht atomar und nicht gegen gleichzeitige Läufe abgesichert; für Systemdateien kombiniert man sie mit einer Sperre und dem atomaren Schreiben von oben.

> **Ergänzung aus der Nachrecherche**
>
> **Selbst geprüft.** Alle Programme dieser Lerneinheit sind ausgeführt worden, ebenso die `flock`-Aufrufe in der Shell (Exit-Codes 1, 75 und 0 wie gezeigt). Das Verhalten von `flock(1)` (die Schalter `-n`, `-w`, `-E`) stimmt mit der Handbuchseite überein, die zu diesem Skript gelesen wurde. Auf Fachwissen beruhen die Angaben zu NFS, zur Wiederverwendung von PIDs und zu den Verzeichnissen `/run/lock` und `/var/lock`.

> **Merke**
>
> Locking verhindert, dass sich Läufe überschneiden: `fcntl.flock` mit `LOCK_EX | LOCK_NB` auf eine dauerhaft vorhandene Sperrdatei, oder in der Shell `flock -n`. Das Betriebssystem gibt die Sperre bei jedem Ende des Prozesses frei. PID-Dateien (`O_EXCL`) brauchen eine Prüfung auf veraltete Einträge. Idempotenz: Zielzustand beschreiben, vor dem Schreiben vergleichen, Dateien atomar mit Temporärdatei und `os.replace` schreiben.

> **Praxisbeispiel**
>
> Fast jedes regelmäßig laufende Skript im Betrieb (Backup, Synchronisation, Datenimport) braucht ein `flock`, sobald seine Laufzeit schwanken kann. Beim Nachrüsten einer bestehenden crontab genügt oft, den Befehl in `flock -n /run/lock/name.lock` einzupacken. Und wer Konfigurationsdateien verändert, schreibt sie atomar: Ein halb geschriebenes `sshd_config` oder ein halbes Zertifikat nach einem Absturz sperrt Sie sonst im schlimmsten Fall aus dem eigenen Server aus.

## 9.5 Projekt: zeitgesteuerter Report

In dieser Lerneinheit fügen wir zusammen, was die Lerntage 1 bis 4 einzeln gezeigt haben: ein Skript, das Rechte und Shebang beherrscht, von cron oder einem systemd-Timer gestartet wird, sich nicht mit sich selbst überschneidet und seine Ergebnisse so schreibt, dass ein Abbruch nichts zerstört. Der Meilenstein des Moduls lautet „Zeitgesteuerter Report“. Wir bauen dazu ein Programm `tagesreport.py`, das jeden Werktag morgens einen kurzen Zustandsbericht über den Server erzeugt: Belegung von Festplatten, Systemlast und Speicherverbrauch. Das ist bewusst schlicht gehalten; wichtig ist nicht der Inhalt des Berichts, sondern die *Betriebstauglichkeit* des Programms. In Modul 11 lernen Sie mit `psutil` reichere Messwerte kennen, in Modul 12 verschicken wir den Bericht per E-Mail.

**Anforderungen.** Bevor man Code schreibt, hält man fest, was das Programm können muss, denn aus den Anforderungen folgen fast alle Entwurfsentscheidungen. Der Bericht entsteht in einer Datei mit dem Datum im Namen (`report-2027-01-11.txt`), damit sich frühere Berichte nicht überschreiben und trotzdem ein zweiter Lauf am selben Tag denselben Bericht ersetzt, statt einen zweiten anzulegen: Das ist die Idempotenz aus 9.4. Das Schreiben erfolgt atomar, sodass ein Leser (etwa ein Programm, das den Bericht per Mail verschickt) nie eine halbe Datei sieht. Eine Sperre verhindert überlappende Läufe; ein abgewiesener Lauf beendet sich mit dem Exit-Code 75, damit er im Protokoll vom Fehlerfall zu unterscheiden ist. Ein Plattenbelegungs-Schwellwert löst eine Warnung aus; mit der Option `--streng` wird daraus der Exit-Code 2, damit ein Zeitplaner (systemd) die Unit als fehlgeschlagen markiert und die Warnung nicht im Protokoll untergeht. Alle Meldungen laufen über `logging` (Modul 6) nach `stderr`, wo cron-Umleitungen und das systemd-Journal sie einsammeln. Für den Test lässt sich das Datum vorgeben (`--datum`); ohne Option gilt der heutige Tag.

Der Entwurf trennt zwei Dinge: Die Funktion `baue_bericht` erzeugt den Text und die Liste der Warnungen aus ihren Parametern, ohne irgendetwas zu schreiben; sie ist dadurch leicht zu testen. Die Funktion `main` kümmert sich um Argumente, Sperre, Fehlerbehandlung und Exit-Codes. Sie nimmt eine Argumentliste als Parameter (`main(argv=None)`), statt `sys.argv` zu lesen; so kann ein Test sie direkt aufrufen, wie wir gleich sehen werden. Die Exit-Codes stehen als benannte Konstanten am Anfang, damit im Code keine „magischen“ Zahlen stehen.

```python
#!/usr/bin/env python3
"""tagesreport - schreibt einen Textbericht ueber Platte, Last und Speicher."""
import argparse
import fcntl
import logging
import os
import shutil
import sys
import tempfile
from datetime import date
from pathlib import Path

log = logging.getLogger("tagesreport")

EXIT_OK, EXIT_FEHLER, EXIT_WARNUNG, EXIT_GESPERRT = 0, 1, 2, 75


def speicher_info():
    """Liest MemTotal und MemAvailable (in MiB) aus /proc/meminfo."""
    werte = {}
    with open("/proc/meminfo", encoding="ascii") as datei:
        for zeile in datei:
            name, _, rest = zeile.partition(":")
            if name in ("MemTotal", "MemAvailable"):
                werte[name] = int(rest.split()[0]) // 1024
    return werte["MemTotal"], werte["MemAvailable"]


def baue_bericht(tag, pfade, schwelle):
    """Gibt (Text, Warnungen) zurueck. Enthaelt nichts, was von der Uhrzeit abhaengt."""
    zeilen = [f"Tagesreport {tag.isoformat()}", "=" * 30]
    warnungen = []
    for pfad in pfade:
        nutzung = shutil.disk_usage(pfad)
        prozent = round(100 * nutzung.used / nutzung.total)
        marke = ""
        if prozent >= schwelle:
            marke = "  <-- WARNUNG"
            warnungen.append(f"{pfad} zu {prozent} % belegt")
        zeilen.append(f"Platte {pfad}: {prozent} % belegt{marke}")
    last1, last5, last15 = os.getloadavg()
    zeilen.append(f"Last (1/5/15 min): {last1:.2f} {last5:.2f} {last15:.2f}")
    gesamt, frei = speicher_info()
    zeilen.append(f"Speicher: {gesamt - frei} von {gesamt} MiB belegt")
    return "\n".join(zeilen) + "\n", warnungen


def schreibe_atomar(ziel, text):
    fd, temp = tempfile.mkstemp(dir=ziel.parent, prefix=ziel.name + ".")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as datei:
            datei.write(text)
            datei.flush()
            os.fsync(datei.fileno())
        os.chmod(temp, 0o644)
        os.replace(temp, ziel)
    except BaseException:
        Path(temp).unlink(missing_ok=True)
        raise


def main(argv=None):
    parser = argparse.ArgumentParser(description="Tagesreport erzeugen")
    parser.add_argument("--ausgabe", type=Path, required=True, help="Zielordner")
    parser.add_argument("--pfad", action="append", default=None, help="zu pruefender Pfad (mehrfach)")
    parser.add_argument("--schwelle", type=int, default=90, help="Warnschwelle in Prozent")
    parser.add_argument("--datum", type=date.fromisoformat, default=None, help="ISO-Datum, Standard: heute")
    parser.add_argument("--streng", action="store_true", help="Exit-Code 2 bei Warnungen")
    args = parser.parse_args(argv)

    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    tag = args.datum or date.today()
    pfade = args.pfad or ["/"]
    try:
        args.ausgabe.mkdir(parents=True, exist_ok=True)
        with open(args.ausgabe / ".tagesreport.lock", "a+") as sperre:
            try:
                fcntl.flock(sperre, fcntl.LOCK_EX | fcntl.LOCK_NB)
            except BlockingIOError:
                log.warning("Anderer Lauf aktiv, ueberspringe")
                return EXIT_GESPERRT
            text, warnungen = baue_bericht(tag, pfade, args.schwelle)
            ziel = args.ausgabe / f"report-{tag.isoformat()}.txt"
            schreibe_atomar(ziel, text)
            log.info("Bericht geschrieben: %s", ziel.name)
    except OSError as fehler:
        log.error("Bericht nicht moeglich: %s", fehler)
        return EXIT_FEHLER
    for w in warnungen:
        log.warning(w)
    return EXIT_WARNUNG if (warnungen and args.streng) else EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Gehen wir den Code in der Reihenfolge durch. Die Shebang-Zeile (9.1) erlaubt den direkten Aufruf. `speicher_info` liest die Datei `/proc/meminfo`, ein vom Linux-Kernel bereitgestelltes Pseudo-Dateisystem, in dem Zeilen wie `MemTotal:  8219876 kB` stehen; `partition(":")` trennt den Namen vom Rest ab, `rest.split()[0]` nimmt die Zahl, und die ganzzahlige Division durch 1024 macht aus Kibibyte Mebibyte. `MemAvailable` ist der Teil des Speichers, den neue Programme bekommen könnten, ohne dass andere ausgelagert werden müssen; das ist aussagekräftiger als der Wert für „frei“. In `baue_bericht` liefert `shutil.disk_usage` (Modul 4) Gesamtgröße und Belegung des Dateisystems zu einem Pfad, `os.getloadavg` die durchschnittliche Systemlast der letzten 1, 5 und 15 Minuten. Der Bericht entsteht als Liste von Zeilen, die zum Schluss mit Zeilenumbrüchen zusammengefügt wird. `schreibe_atomar` ist das Verfahren aus 9.4, ergänzt um `os.chmod(temp, 0o644)`: Eine mit `mkstemp` erzeugte Datei hat aus Sicherheitsgründen die Rechte `0600`, und ein Bericht, den andere Benutzer lesen sollen, braucht daher eine ausdrückliche Anpassung. In `main` definiert `argparse` (Modul 5) die Optionen; `type=date.fromisoformat` wandelt den Text direkt in ein Datum um und erzeugt bei falscher Eingabe eine passende Fehlermeldung. Die Sperre wird im Zielordner angelegt (`.tagesreport.lock`), nicht an einem fest verdrahteten Ort, damit die Tests mit eigenen Ordnern arbeiten können. Der `try`-Block fängt `OSError` ab: Dazu gehören ein nicht beschreibbarer Zielordner, eine volle Platte und ein Zielpfad, der eine Datei ist. Diese Fehler werden protokolliert und mit dem Exit-Code 1 beantwortet, statt einen Traceback auszugeben; ein Programmierfehler (etwa ein `KeyError`) bleibt dagegen sichtbar. Die Warnungen werden *nach* dem erfolgreichen Schreiben ausgegeben, weil sie das Ergebnis nicht verhindern. Zuletzt entscheidet `--streng`, ob Warnungen sich im Exit-Code niederschlagen.

**Das Programm testen.** Ein Programm für den Dauerbetrieb testet man, bevor es in den Zeitplaner kommt. Weil `main` die Argumente als Parameter erhält, lässt es sich aus einem anderen Python-Programm aufrufen und der Rückgabewert prüfen. Das folgende Testprogramm spielt sechs Fälle durch: den normalen Lauf (und den Inhalt des Berichts, dessen Zahlen wir durch ein `N` ersetzen, da sie sich bei jedem Lauf ändern), den wiederholten Lauf, die Warnschwelle ohne und mit `--streng`, den Lauf bei gehaltener Sperre und den Fall, dass der Zielordner gar keiner ist. Die Meldungen der `logging`-Ausgabe gehen nach `stderr` und erscheinen hier nicht; zu sehen sind die Rückgabewerte.

```python
import fcntl
import re
import tempfile
from pathlib import Path

import tagesreport

ordner = Path(tempfile.mkdtemp())

print("1. Normaler Lauf:     ", tagesreport.main(["--ausgabe", str(ordner), "--datum", "2027-01-11"]))
bericht = (ordner / "report-2027-01-11.txt").read_text(encoding="utf-8")
for zeile in bericht.splitlines():
    print("   |", re.sub(r"\d+(\.\d+)?", "N", zeile) if zeile[0] != "T" else zeile)

print("2. Zweiter Lauf:      ", tagesreport.main(["--ausgabe", str(ordner), "--datum", "2027-01-11"]))
print("   Berichte im Ordner:", sorted(p.name for p in ordner.glob("report-*")))

print("3. Schwelle 0, ohne --streng:", tagesreport.main(["--ausgabe", str(ordner), "--schwelle", "0"]))
print("4. Schwelle 0, mit --streng: ", tagesreport.main(["--ausgabe", str(ordner), "--schwelle", "0", "--streng"]))

with open(ordner / ".tagesreport.lock", "a+") as sperre:
    fcntl.flock(sperre, fcntl.LOCK_EX | fcntl.LOCK_NB)
    print("5. Sperre gehalten:    ", tagesreport.main(["--ausgabe", str(ordner)]))

print("6. Ziel ist eine Datei:", tagesreport.main(["--ausgabe", str(ordner / "report-2027-01-11.txt")]))
```

```text
1. Normaler Lauf:      0
   | Tagesreport 2027-01-11
   | ==============================
   | Platte /: N % belegt
   | Last (N/N/N min): N N N
   | Speicher: N von N MiB belegt
2. Zweiter Lauf:       0
   Berichte im Ordner: ['report-2027-01-11.txt']
3. Schwelle 0, ohne --streng: 0
4. Schwelle 0, mit --streng:  2
5. Sperre gehalten:     75
6. Ziel ist eine Datei: 1
```

Fall 1 liefert den Code 0 und einen Bericht mit der erwarteten Gestalt. Fall 2 zeigt die Idempotenz: Der zweite Lauf mit demselben Datum endet ebenfalls mit 0, und im Ordner liegt weiterhin genau ein Bericht. In Fall 3 und 4 liegt die Schwelle bei 0, sodass jede Platte „zu voll“ ist (die Datei wird also trotzdem geschrieben, die Warnung steht nur im Protokoll); ohne `--streng` bleibt der Code 0, mit `--streng` wird er 2. In Fall 5 hält das Testprogramm selbst die Sperre, und der Lauf wird mit 75 abgewiesen. Fall 6 gibt dem Programm als Zielordner den Pfad einer vorhandenen Datei: `mkdir` scheitert mit `FileExistsError`, und das Programm antwortet mit Exit-Code 1.

**In den Betrieb nehmen.** Installiert wird das Skript an einem festen Ort, hier `/opt/tagesreport`. Die Schritte sind: Verzeichnis anlegen, Skript kopieren, ausführbar machen (`chmod 755`), virtuelle Umgebung anlegen, falls Pakete nötig werden (hier nicht, das Programm braucht nur die Standardbibliothek, also genügt auch `/usr/bin/python3`), einen Benutzer `report` anlegen und den Zielordner `/var/lib/tagesreport` für ihn beschreibbar machen. Dann wählt man den Zeitplaner. Mit cron lautet der Eintrag (alles in einer Zeile) so:

```text
30 6 * * 1-5 /usr/bin/python3 /opt/tagesreport/tagesreport.py --ausgabe /var/lib/tagesreport --pfad / --pfad /var --streng >> /var/log/tagesreport.log 2>&1
```

Er startet das Skript werktags um 06:30 Uhr, übergibt zwei zu prüfende Pfade und leitet beide Ausgabekanäle in eine Protokolldatei. Mit systemd verwendet man die beiden Units aus 9.3, wobei die Service-Datei so aussieht:

```ini
[Service]
Type=oneshot
User=report
ExecStart=/usr/bin/python3 /opt/tagesreport/tagesreport.py --ausgabe /var/lib/tagesreport --pfad / --pfad /var --streng
SuccessExitStatus=75
```

Neu ist die Zeile `SuccessExitStatus=75`. Normalerweise gilt jeder Exit-Code außer 0 als Fehlschlag; mit dieser Option erklärt man den Code 75 („Sperre besetzt, übersprungen“) zum Erfolg, damit ein harmloser Überschneidungsfall die Unit nicht rot färbt. Der Exit-Code 2 (Warnung bei `--streng`) bleibt ein Fehlschlag und erscheint in `systemctl --failed`: gewollt, denn eine volle Platte soll auffallen. Der Zeitplan in der Timer-Datei ist `OnCalendar=Mon..Fri 06:30`. Nach einem Probelauf von Hand (`systemctl start tagesreport.service`, dann `journalctl -u tagesreport.service`) aktiviert man den Timer.

**Typische Fehler und Stolperfallen.** Aus der Erfahrung treten bei solchen Projekten immer dieselben Fehler auf, und alle haben wir bereits einzeln besprochen. Das Skript funktioniert von Hand und scheitert unter cron, weil ein relativer Pfad oder ein Programm aus dem persönlichen `PATH` verwendet wurde (9.2). Die Rechte des Zielordners passen nicht zum Benutzer der Unit: Dann endet jeder Lauf mit dem Exit-Code 1 und der Meldung „Permission denied“ im Journal; ein Blick in `journalctl -u` genügt zur Diagnose. Die Berichte werden nicht aufgeräumt und füllen im Lauf der Jahre die Platte; ein kleines Zusatzprogramm, das Berichte über 90 Tage löscht, gehört in die Praxis und ist eine gute Ergänzung zum Projekt. Der Bericht wird nicht atomar geschrieben, und ein Empfänger liest zufällig eine halbe Datei. Und schließlich der Klassiker, den kein Test findet: Das Skript meldet Erfolg, läuft aber gar nicht, weil der Timer nie aktiviert wurde. Dagegen hilft nur der Blick auf `systemctl list-timers` nach der Einrichtung und, wo es darauf ankommt, eine Überwachung, die Alarm schlägt, wenn der Bericht ausbleibt (siehe Modul 12).

> **Ergänzung aus der Nachrecherche**
>
> **Was in der Prüfumgebung ausgeführt wurde.** Das Skript `tagesreport.py` und das Testprogramm wurden ausgeführt (Exit-Codes 0, 2, 75, 1 wie gezeigt). Nicht ausgeführt wurden der cron-Eintrag und die systemd-Units im Dauerbetrieb, da dort systemd nicht als Init-System läuft; die Unit-Syntax wurde mit `systemd-analyze verify` geprüft (siehe 9.3), die Option `SuccessExitStatus` entstammt der Dokumentation von `systemd.service(5)`. Das Skript liest `/proc/meminfo` und `os.getloadavg()`; beides gibt es nur unter Linux und Unix. Für Windows-Server ist ein anderer Ansatz nötig (der Aufgabenplaner und das Paket `psutil`, siehe Modul 11).

> **Merke**
>
> Ein betriebstaugliches Skript: Hauptfunktion mit Parameterliste (testbar), benannte Exit-Codes, Protokoll über `logging`, Sperre gegen Überschneidung, atomares Schreiben, idempotentes Ergebnis (Datum im Dateinamen), absolute Pfade, eigener Benutzer mit minimalen Rechten. Danach erst in den Zeitplaner, und *vorher* von Hand und über `systemctl start` testen.

> **Praxisbeispiel**
>
> Dieses Gerüst (Argumente, Sperre, Protokoll, Exit-Codes, atomar schreiben) ist die Vorlage für nahezu jedes Betriebsskript: Datenexport, Zertifikatsprüfung, Datenbank-Aufräumen, Backup. Wer es einmal beherrscht, ersetzt nur `baue_bericht` durch die eigentliche Aufgabe. In den Modulen 10 bis 12 füllen wir es mit Web-Abfragen, Systemmesswerten, SSH und Benachrichtigungen.

## Übungsaufgaben zu Modul 09

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Für die Aufgaben 2 und 5 benötigen Sie das Modul `cronzeit` aus 9.2; die Aufgabe 6 setzt voraus, dass `systemd-analyze` installiert ist.

**Aufgabe 1.** Was bewirkt die Zeile `#!/usr/bin/env python3` am Anfang einer Datei, und was ist der Unterschied zu `#!/usr/bin/python3`? Was muss gegeben sein, damit `./skript.py` überhaupt startet? Welche Rechte (in Oktalschreibweise) geben Sie einem Skript, das alle ausführen, aber nur Sie ändern dürfen, und welche einer Datei mit Zugangsdaten? Welche Wirkung hat die `umask` auf neu angelegte Dateien?

**Aufgabe 2.** Lesen Sie die folgenden cron-Ausdrücke und beschreiben Sie in einem Satz, wann sie laufen. Prüfen Sie Ihre Antworten mit dem Rechner aus 9.2 (Startpunkt Montag, 11.01.2027, 09:17 Uhr): `15 */6 * * *`, `0 8-18/2 * * 1-5`, `30 23 28-31 * *`, `0 0 13 * 5`, `@weekly`. Warum ist der dritte Ausdruck *kein* Weg, um „am letzten Tag des Monats“ zu planen?

**Aufgabe 3.** Die folgende crontab-Zeile soll werktags um 02:30 Uhr eine Sicherung mit Datum im Dateinamen erzeugen und funktioniert nicht. Finden Sie mindestens fünf Probleme und beheben Sie sie.

```text
30 2 * * 1-5 cd backup && python3 sichern.py > /backup/sicherung-`date +%F`.log
```

**Aufgabe 4.** Erklären Sie die Begriffe *Locking* und *Idempotenz* mit eigenen Worten. Warum ist `flock` einer PID-Datei überlegen, wenn es um das Aussperren paralleler Läufe geht? Was passiert mit einer `flock`-Sperre, wenn der Prozess abstürzt, und was mit einer PID-Datei? Nennen Sie zwei Beispiele für Operationen, die nicht idempotent sind, und wie man sie idempotent macht.

**Aufgabe 5.** Schreiben Sie eine Funktion `ist_faellig(ausdruck, zeitpunkt)`, die mit Hilfe von `cronzeit` entscheidet, ob ein cron-Ausdruck zu einem gegebenen `datetime` auslöst (Sekunden spielen keine Rolle). Testen Sie mit mindestens fünf Fällen, darunter einen für die ODER-Regel der Tagesfelder.

**Aufgabe 6.** Schreiben Sie eine Funktion `erzeuge_units(name, beschreibung, befehl, kalender)`, die den Text einer `.service`- und einer `.timer`-Datei als Dictionary liefert, schreiben Sie die Dateien in einen temporären Ordner und lassen Sie sie mit `systemd-analyze verify` prüfen.

**Aufgabe 7.** Schreiben Sie einen Dekorator `nur_einmal_gleichzeitig(sperrdatei)`, der die dekorierte Funktion nur startet, wenn er die `flock`-Sperre bekommt, und andernfalls eine Meldung ausgibt und `None` liefert. Zeigen Sie die Wirkung mit einem Aufruf der Funktion aus sich selbst heraus.

**Aufgabe 8.** Schreiben Sie eine Funktion `stelle_zeile_sicher(datei, zeile)`, die eine Zeile an eine Textdatei anhängt, *falls sie noch nicht vorkommt*, die Datei dabei atomar ersetzt, ein fehlendes Zeilenende am Dateiende ergänzt und `True` zurückgibt, wenn sie etwas geändert hat, sonst `False`. Zeigen Sie, dass dreimaliges Aufrufen nur einmal ändert.

## Lösungen zu Modul 09

**Aufgabe 1.** Die Zeile heißt *Shebang*. Beim Start einer Datei, die ausführbar ist, liest der Kernel die erste Zeile und startet das dort genannte Programm mit dem Skript als Argument. Bei `#!/usr/bin/env python3` sucht `env` den ersten `python3` im `PATH`; das ist portabel und folgt einer aktivierten virtuellen Umgebung. `#!/usr/bin/python3` nennt dagegen einen festen Pfad: vorhersagbar, aber nur gültig, wo der Interpreter genau dort liegt. Damit `./skript.py` startet, muss die Datei das Ausführungsrecht haben (`chmod +x`), die Shebang-Zeile muss in der *ersten Zeile* stehen und der Interpreter existieren, und das Dateiformat muss Unix-Zeilenenden haben (bei Windows-Zeilenenden sucht der Kernel einen Interpreter mit einem Wagenrücklauf im Namen). Ein Skript, das alle ausführen, aber nur der Besitzer ändern darf, erhält `0o755`; eine Datei mit Zugangsdaten `0o600`. Die `umask` ist eine Maske, deren gesetzte Bits beim Anlegen *abgezogen* werden: Bei `022` wird eine neue Datei mit `0o666` zu `0o644`, ein Ordner mit `0o777` zu `0o755`. Die Maske begrenzt also die Rechte, die ein Programm vergeben kann, sie vergibt keine.

**Aufgabe 2.** Die Ausdrücke bedeuten der Reihe nach: `15 */6 * * *` läuft alle sechs Stunden zur Minute 15 (um 00:15, 06:15, 12:15 und 18:15). `0 8-18/2 * * 1-5` läuft werktags zu den geraden Stunden von 8 bis 18 Uhr (8, 10, 12, 14, 16, 18 Uhr). `30 23 28-31 * *` läuft am 28., 29., 30. und 31. um 23:30 Uhr, soweit der Tag im Monat existiert. `0 0 13 * 5` läuft um Mitternacht am 13. jedes Monats *und zusätzlich* an jedem Freitag (ODER-Regel). `@weekly` läuft sonntags um Mitternacht. Die Kontrolle mit dem Rechner:

```python
from datetime import datetime

import cronzeit

ab = datetime(2027, 1, 11, 9, 17)
for ausdruck, anzahl in (("15 */6 * * *", 5), ("0 8-18/2 * * 1-5", 7), ("30 23 28-31 * *", 5),
                         ("0 0 13 * 5", 5), ("@weekly", 2)):
    laeufe = cronzeit.naechste_laeufe(ausdruck, ab, anzahl)
    print(ausdruck, "->", ", ".join(t.strftime("%a %d.%m. %H:%M") for t in laeufe))
```

```text
15 */6 * * * -> Mon 11.01. 12:15, Mon 11.01. 18:15, Tue 12.01. 00:15, Tue 12.01. 06:15, Tue 12.01. 12:15
0 8-18/2 * * 1-5 -> Mon 11.01. 10:00, Mon 11.01. 12:00, Mon 11.01. 14:00, Mon 11.01. 16:00, Mon 11.01. 18:00, Tue 12.01. 08:00, Tue 12.01. 10:00
30 23 28-31 * * -> Thu 28.01. 23:30, Fri 29.01. 23:30, Sat 30.01. 23:30, Sun 31.01. 23:30, Sun 28.02. 23:30
0 0 13 * 5 -> Wed 13.01. 00:00, Fri 15.01. 00:00, Fri 22.01. 00:00, Fri 29.01. 00:00, Fri 05.02. 00:00
@weekly -> Sun 17.01. 00:00, Sun 24.01. 00:00
```

Beim dritten Ausdruck sieht man, warum er den letzten Tag des Monats nicht trifft: Er läuft an mehreren Tagen hintereinander (hier vier Mal im Januar), im Februar 2027 nur am 28., weil es keinen 29. bis 31. gibt. Cron kennt keine Angabe wie „letzter Tag“. Üblich ist, den Eintrag an jedem Tag ab 28. auszulösen und im Skript zu prüfen, ob morgen der 1. des Folgemonats ist (`(heute + timedelta(days=1)).day == 1`).

**Aufgabe 3.** Die Zeile hat mehr Fehler, als man auf den ersten Blick sieht. Erstens: `cd backup` ist ein *relativer* Pfad; cron startet im Heimatverzeichnis, und ob dort ein Ordner `backup` existiert, ist Zufall. Es gehört ein absoluter Pfad hin (oder man verzichtet auf `cd` und gibt den absoluten Pfad des Skripts an, das seine Pfade selbst bestimmt). Zweitens: `python3` ohne Pfad; cron hat einen kurzen `PATH`, die Angabe `/usr/bin/python3` (oder der Interpreter der virtuellen Umgebung) ist sicherer. Drittens: Das `%` in `date +%F` wird von cron als Zeilenumbruch gedeutet und muss mit Rückwärtsschrägstrich geschützt werden (`\%F`). Viertens: Es wird nur `stdout` umgeleitet; Fehlermeldungen auf `stderr` gehen verloren oder in eine Mail, die nie ankommt, es fehlt also `2>&1`. Fünftens: Ein Protokoll mit einem Datum im Namen erzeugt täglich eine neue Datei und räumt nie auf; ein Skript, das über `logging` selbst protokolliert (Modul 6), ist besser. Sechstens: Es gibt keine Sperre, ein langsamer Lauf überschneidet sich mit dem nächsten (`flock -n`). Eine verbesserte Zeile (die Logik mit dem Datum gehört ins Skript, das den Dateinamen selbst erzeugt):

```text
30 2 * * 1-5 flock -n /run/lock/sichern.lock /usr/bin/python3 /opt/backup/sichern.py >> /var/log/sichern.log 2>&1
```

**Aufgabe 4.** *Locking* bedeutet, dass ein Lauf eine Ressource exklusiv für sich beansprucht, sodass sich parallele Läufe nicht gegenseitig stören. *Idempotenz* bedeutet, dass mehrfaches Ausführen dasselbe Ergebnis hat wie einmaliges. `flock` ist der PID-Datei überlegen, weil das Betriebssystem die Sperre bei jedem Prozessende freigibt, auch bei Absturz oder `kill -9`; eine PID-Datei bleibt nach einem Absturz liegen und blockiert (oder täuscht) alle folgenden Läufe, wenn man nicht ausdrücklich prüft, ob der eingetragene Prozess noch lebt, und selbst diese Prüfung kann durch wiederverwendete Prozessnummern irren. Nicht idempotent sind zum Beispiel das Anhängen einer Zeile an eine Datei (Lösung: vorher prüfen, ob sie schon vorkommt, oder den Zielzustand schreiben) und das Anlegen eines Ordners mit `os.mkdir` (Lösung: `mkdir(exist_ok=True)`). Ebenso nicht idempotent: ein Zähler, der bei jedem Lauf erhöht wird (Lösung: den Wert aus der Quelle neu berechnen und setzen).

**Aufgabe 5.** Die Funktion zerlegt den Ausdruck mit `parse_ausdruck` und prüft Minute, Stunde und die Tagesregel. Dabei verwendet sie die Hilfsfunktion `tag_passt` aus 9.2, die Monat, Tag und Wochentag samt ODER-Regel erledigt.

```python
from datetime import datetime

import cronzeit


def ist_faellig(ausdruck, zeitpunkt):
    (minuten, stunden, tage, monate, wochentage), oder_regel = cronzeit.parse_ausdruck(ausdruck)
    return (zeitpunkt.minute in minuten
            and zeitpunkt.hour in stunden
            and cronzeit.tag_passt(zeitpunkt, monate, tage, wochentage, oder_regel))


print(ist_faellig("30 4 * * 1-5", datetime(2027, 1, 12, 4, 30)))
print(ist_faellig("30 4 * * 1-5", datetime(2027, 1, 16, 4, 30)))
print(ist_faellig("*/15 * * * *", datetime(2027, 1, 12, 4, 45)))
print(ist_faellig("*/15 * * * *", datetime(2027, 1, 12, 4, 50)))
print(ist_faellig("0 6 1,15 * 5", datetime(2027, 1, 22, 6, 0)))
```

```text
True
False
True
False
True
```

Die ersten beiden Zeilen testen den Wochentag (Dienstag, 12.01.2027, passt, Samstag nicht), die nächsten beiden die Schrittangabe (04:45 passt, 04:50 nicht). Die letzte Zeile ist ein Freitag, der 22. Januar; weil beide Tagesfelder eingeschränkt sind (`1,15` und `5`), genügt der Freitag, obwohl der 22. weder der 1. noch der 15. ist. Der häufigste Fehler in eigenen Lösungen ist, die Tagesfelder mit UND zu prüfen und damit die ODER-Regel zu verfehlen.

**Aufgabe 6.** Die Funktion baut die Texte mit f-Strings zusammen. Die Prüfung läuft über `systemd-analyze verify`, dem man die Pfade beider Dateien übergibt; der Rückgabecode ist 0, wenn nichts zu beanstanden ist. Für den Befehl wird `/usr/bin/true` verwendet, weil das Werkzeug prüft, ob die Datei existiert und ausführbar ist.

```python
import subprocess
import tempfile
from pathlib import Path


def erzeuge_units(name, beschreibung, befehl, kalender):
    service = (f"[Unit]\nDescription={beschreibung}\n\n"
               f"[Service]\nType=oneshot\nExecStart={befehl}\n")
    timer = (f"[Unit]\nDescription={beschreibung} (Zeitplan)\n\n"
             f"[Timer]\nOnCalendar={kalender}\nPersistent=true\n\n"
             f"[Install]\nWantedBy=timers.target\n")
    return {f"{name}.service": service, f"{name}.timer": timer}


units = erzeuge_units("sicherung", "Sicherung", "/usr/bin/true", "*-*-* 02:15")
with tempfile.TemporaryDirectory() as ordner:
    pfade = []
    for dateiname, inhalt in units.items():
        pfad = Path(ordner) / dateiname
        pfad.write_text(inhalt, encoding="utf-8")
        pfade.append(str(pfad))
    r = subprocess.run(["systemd-analyze", "verify", *pfade], capture_output=True, text=True)
    meldungen = [z for z in r.stderr.splitlines() if "sicherung" in z]
    print("Rückgabecode:", r.returncode, "Meldungen:", meldungen or "keine")
```

```text
Rückgabecode: 0 Meldungen: keine
```

Wichtig ist, dass der Befehl *existiert*: Würde man einen erfundenen Pfad verwenden, meldete `verify` „Command ... is not executable“. Das ist ein nützlicher Test beim Einrichten, kein Fehler des Werkzeugs. Beim Kalender kann man die Eingabe zusätzlich mit `systemd-analyze calendar` testen, bevor man die Datei schreibt.

**Aufgabe 7.** Der Dekorator ist eine Funktion, die eine Funktion zurückgibt (Modul 3). Die innere Hülle öffnet die Sperrdatei, versucht die Sperre mit `LOCK_NB` und ruft entweder die dekorierte Funktion auf oder meldet, dass der Lauf übersprungen wird. Das `with` sorgt dafür, dass die Datei und damit die Sperre am Ende freigegeben werden; `functools.wraps` bewahrt Name und Dokumentation der dekorierten Funktion.

```python
import fcntl
import functools
import tempfile
from pathlib import Path


def nur_einmal_gleichzeitig(sperrdatei):
    def dekorator(funktion):
        @functools.wraps(funktion)
        def huelle(*args, **kwargs):
            with open(sperrdatei, "a+") as datei:
                try:
                    fcntl.flock(datei, fcntl.LOCK_EX | fcntl.LOCK_NB)
                except BlockingIOError:
                    print(f"{funktion.__name__}: läuft schon, übersprungen")
                    return None
                return funktion(*args, **kwargs)
        return huelle
    return dekorator


sperre = Path(tempfile.mkdtemp()) / "job.lock"


@nur_einmal_gleichzeitig(sperre)
def job(tiefe=0):
    print(f"job läuft (Tiefe {tiefe})")
    if tiefe < 1:
        job(tiefe + 1)           # zweiter, gleichzeitiger Lauf im selben Prozess


job()
job()
```

```text
job läuft (Tiefe 0)
job: läuft schon, übersprungen
job läuft (Tiefe 0)
job: läuft schon, übersprungen
```

Der erste Aufruf von `job()` hält die Sperre und ruft sich selbst auf; der innere Aufruf öffnet die Sperrdatei ein zweites Mal, bekommt die Sperre nicht und wird übersprungen. Danach läuft der zweite äußere Aufruf wieder normal durch, weil die Sperre inzwischen freigegeben wurde. Dass die Funktion hier sich selbst aufruft, ist nur eine Vorführung; in der Praxis kommt das Gegenstück aus einem zweiten Prozess.

**Aufgabe 8.** Die Funktion liest den bisherigen Inhalt (oder einen leeren Text, falls die Datei fehlt), prüft mit `splitlines()`, ob die Zeile schon vorkommt, ergänzt bei Bedarf ein fehlendes Zeilenende und schreibt das Ergebnis in eine Temporärdatei im selben Ordner, die anschließend mit `os.replace` über das Ziel geschoben wird. Die Rechte einer bestehenden Datei werden übernommen, weil `mkstemp` sonst `0o600` vergibt.

```python
import os
import tempfile
from pathlib import Path


def stelle_zeile_sicher(datei, zeile):
    datei = Path(datei)
    inhalt = datei.read_text(encoding="utf-8") if datei.exists() else ""
    if zeile in inhalt.splitlines():
        return False
    if inhalt and not inhalt.endswith("\n"):
        inhalt += "\n"
    neu = inhalt + zeile + "\n"
    fd, temp = tempfile.mkstemp(dir=datei.parent, prefix=datei.name + ".")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as ziel:
            ziel.write(neu)
        if datei.exists():
            os.chmod(temp, datei.stat().st_mode & 0o7777)
        os.replace(temp, datei)
    except BaseException:
        Path(temp).unlink(missing_ok=True)
        raise
    return True


ordner = Path(tempfile.mkdtemp())
hosts = ordner / "hosts"
hosts.write_text("127.0.0.1 localhost")          # ohne Zeilenumbruch am Ende
for _ in range(3):
    print(stelle_zeile_sicher(hosts, "10.0.0.5 server"))
print(repr(hosts.read_text()))
```

```text
True
False
False
'127.0.0.1 localhost\n10.0.0.5 server\n'
```

Der erste Aufruf ändert die Datei (und ergänzt dabei das fehlende Zeilenende nach der ersten Zeile), die beiden folgenden finden die Zeile vor und ändern nichts. Das Ergebnis enthält die Zeile genau einmal. Zu beachten bleibt, dass die Funktion nicht gegen *gleichzeitige* Aufrufe schützt: Für Dateien, die mehrere Prozesse verändern, kombiniert man sie mit der Sperre aus Aufgabe 7.
