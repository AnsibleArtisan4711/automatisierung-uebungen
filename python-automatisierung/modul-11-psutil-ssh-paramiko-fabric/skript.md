# Modul 11 - psutil, SSH (paramiko-fabric)

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (11.1 bis 11.5), danach folgen „Übungsaufgaben zu Modul 11“ (8 Aufgaben) und „Lösungen zu Modul 11“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 25.01.2027 bis Fr 29.01.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul liefert die Messwerte eines Servers und den Zugang zu mehreren Servern, die Grundlage jeder Verwaltungsautomatisierung)

**Meilenstein:** Mehrere Hosts abfragen

**Lernziele.** Nach diesem Modul können Sie mit der Bibliothek `psutil` Prozessor, Arbeitsspeicher, Datenträger und Prozesse eines Rechners auslesen und deuten, sich mit `paramiko` und `fabric` per SSH mit Servern verbinden, Befehle ausführen und Dateien übertragen, Schlüssel und die Datei `known_hosts` sachgerecht einsetzen, statt Sicherheitsprüfungen abzuschalten, und mehrere Server gleichzeitig abfragen, wobei der Ausfall einzelner Hosts das Gesamtergebnis nicht zerstört.

| **Nr.** | **Lerneinheit**                  | **Datum**     | **Dauer** | **Prio** |
|:--------|:---------------------------------|:--------------|:----------|:---------|
| 11.1    | psutil (CPU & RAM)               | Mo 25.01.2027 | 2 h       | 2        |
| 11.2    | psutil (Disk & Prozesse)         | Di 26.01.2027 | 2 h       | 2        |
| 11.3    | SSH mit paramiko/fabric          | Mi 27.01.2027 | 2 h       | 2        |
| 11.4    | Mehrere Hosts, Keys, known_hosts | Do 28.01.2027 | 2 h       | 2        |
| 11.5    | Mehrere Hosts abfragen + Quiz    | Fr 29.01.2027 | 2 h       | 2        |
|         | **Summe**                        |               | **10 h**  |          |

> **Merke**
>
> Dieses Modul benötigt `psutil`, `paramiko` und `fabric` (`pip install psutil paramiko fabric`, siehe Modul 7). Getestet wurde mit psutil 7.2.2, paramiko 5.1.0, Fabric 3.2.3 und OpenSSH 9.6 unter Linux und Python 3.13. Für die SSH-Beispiele (ab 11.3) wird ein Hilfsmodul `testssh` verwendet, das auf dem eigenen Rechner kleine SSH-Server startet, die nur auf `127.0.0.x` hören, damit Sie ohne fremde Server üben können; es braucht das Programm `sshd` (Paket `openssh-server`). Messwerte von `psutil` hängen vom Rechner und vom Augenblick ab; deshalb prüfen die Beispiele meist Beziehungen (zum Beispiel „Prozent liegt zwischen 0 und 100“) statt feste Zahlen abzudrucken, und die wenigen Beispielausgaben mit echten Zahlen sind ausdrücklich so gekennzeichnet.

## 11.1 psutil: CPU und Arbeitsspeicher

Wer Server betreut, will ständig wissen, wie es ihnen geht: Wie stark ist der Prozessor ausgelastet? Wie viel Arbeitsspeicher ist frei? Läuft das System in eine Engpasssituation? Unter Linux gibt es dafür seit jeher Werkzeuge wie `top`, `free` und `uptime`, und Sie können deren Ausgabe mit `subprocess` (Modul 5) lesen und mit regulären Ausdrücken (Modul 8) zerlegen. Das funktioniert, ist aber mühsam und anfällig: Das Ausgabeformat kann sich zwischen Versionen ändern, Zahlen stehen mal in Kibibyte, mal in Mebibyte, und jedes Werkzeug existiert nur auf bestimmten Systemen. Die Bibliothek `psutil` (*process and system utilities*) löst das Problem, indem sie dieselben Informationen direkt vom Betriebssystem holt und als Python-Objekte liefert. Nach ihrer Beschreibung auf PyPI (abgerufen) läuft sie unter Linux, Windows, macOS, FreeBSD, OpenBSD, NetBSD, Solaris und AIX und deckt Prozessor, Speicher, Datenträger, Netzwerk, Sensoren und Prozesse ab; sie bildet viele Werkzeuge wie `ps`, `top` und `netstat` nach.

**Der Prozessor: Kerne und Auslastung.** Beginnen wir mit den Grundbegriffen. Ein moderner Prozessor besteht aus mehreren *Kernen*, und jeder kann (bei „Hyper-Threading“ oder *SMT*) mehrere Programmabläufe quasi gleichzeitig bearbeiten; das Betriebssystem sieht jeden dieser Abläufe als *logischen Kern*. `psutil.cpu_count()` liefert die Zahl der logischen Kerne, `psutil.cpu_count(logical=False)` die der physischen (auf manchen Systemen `None`, wenn sie sich nicht ermitteln lässt). Die Standardbibliothek kennt `os.cpu_count()` für die logischen Kerne. Die *Auslastung* (`psutil.cpu_percent`) gibt an, welcher Anteil der Zeit die Kerne mit Arbeit statt Leerlauf verbracht haben. Dabei gibt es eine Besonderheit, die jeder einmal falsch macht: Auslastung ist immer ein Mittelwert über einen *Zeitraum*. Der Zähler des Betriebssystems liefert nur kumulierte Zeiten seit dem Start, und `psutil` bildet daraus den Anteil, indem es zwei Messungen vergleicht. Mit `interval=1` nimmt `psutil` zwei Messungen im Abstand von einer Sekunde und blockiert dazwischen; mit `interval=None` (Vorgabe) vergleicht es den *aktuellen Wert mit dem beim vorigen Aufruf*. Der allererste Aufruf hat noch keinen früheren Wert zum Vergleichen und liefert deshalb den bedeutungslosen Wert `0.0`: In der Prüfumgebung haben wir genau das beobachtet. Die Regel lautet: Den ersten Wert wegwerfen oder gleich mit einem Intervall messen.

```python
import os
import time

import psutil

print("logische Kerne :", psutil.cpu_count(), "| physische Kerne:", psutil.cpu_count(logical=False))
print("os.cpu_count() :", os.cpu_count(), "(gleiche Zahl wie die logischen Kerne:", os.cpu_count() == psutil.cpu_count(), ")")

# Erster Aufruf ohne Intervall: Es gibt noch keine Vergleichsmessung, der Wert ist wertlos
erste = psutil.cpu_percent(interval=None)
time.sleep(0.5)
zweite = psutil.cpu_percent(interval=None)       # Mittelwert seit dem vorigen Aufruf
print("Wert in Prozent 0 bis 100:", 0.0 <= zweite <= 100.0)

# Mit Intervall: psutil misst selbst eine Sekunde lang und blockiert so lange
start = time.monotonic()
gesamt = psutil.cpu_percent(interval=1)
print(f"Messung dauerte {time.monotonic() - start:.1f} s, Ergebnis ist eine Zahl: {isinstance(gesamt, float)}")

# Pro Kern
pro_kern = psutil.cpu_percent(interval=0.5, percpu=True)
print("Eintraege je Kern:", len(pro_kern) == psutil.cpu_count())

zeiten = psutil.cpu_times()
print("cpu_times-Felder:", [f for f in zeiten._fields if f in ("user", "system", "idle")])
print("Zeiten sind Sekunden seit dem Start, nicht Prozent:", zeiten.user > 1 and zeiten.idle > 1)
```

```text
logische Kerne : 2 | physische Kerne: 2
os.cpu_count() : 2 (gleiche Zahl wie die logischen Kerne: True )
Wert in Prozent 0 bis 100: True
Messung dauerte 1.0 s, Ergebnis ist eine Zahl: True
Eintraege je Kern: True
cpu_times-Felder: ['user', 'system', 'idle']
Zeiten sind Sekunden seit dem Start, nicht Prozent: True
```

Das Beispiel zeigt die wichtigsten Aufrufe. Die Kerne der Prüfumgebung (hier zwei, bei Ihnen vermutlich mehr) stimmen mit `os.cpu_count()` überein. Die beiden Aufrufe mit `interval=None` bilden ein Paar: Der erste beginnt die Messung, der zweite liefert den Mittelwert über die halbe Sekunde dazwischen, und der Wert liegt zwischen 0 und 100. Der Aufruf mit `interval=1` blockiert eine Sekunde und liefert direkt einen brauchbaren Wert. Mit `percpu=True` bekommt man eine Liste mit einem Wert je logischem Kern, was nützlich ist, um zu erkennen, dass ein einzelner Kern am Limit ist, während die Gesamtauslastung harmlos aussieht (ein klassisches Zeichen für ein Programm, das nur einen Kern nutzen kann). `psutil.cpu_times()` liefert dagegen *kumulierte Sekunden* seit dem Start des Rechners, aufgeteilt nach Verwendung (`user` für Programme, `system` für den Kern, `idle` für Leerlauf, je nach System weitere): Das sind keine Prozentwerte, sondern Zähler, mit denen sich Verläufe berechnen lassen.

Die Auslastung ist eine Momentaufnahme, die schnell schwankt. Aussagekräftiger für die Frage „ist der Rechner überlastet?“ ist die *Last* (*load average*), die Linux in drei Mittelwerten über 1, 5 und 15 Minuten führt (`os.getloadavg()` liefert sie). Sie zählt, wie viele Prozesse im Mittel rechnen *oder auf einen Kern warten*. Eine Last von 2,0 auf einem Rechner mit zwei Kernen heißt: Im Schnitt sind alle Kerne voll beschäftigt. Wesentlich größere Werte bedeuten, dass Prozesse warten müssen. Üblich ist, die Last durch die Kernzahl zu teilen und Werte deutlich über 1 als Überlastung zu werten. Das nächste Beispiel erzeugt absichtlich Last und misst sie: Ein Thread rechnet 1,2 Sekunden lang in einer leeren Schleife, und die CPU-Nutzung *des eigenen Prozesses*, die `Process.cpu_percent` liefert, steigt entsprechend. Prozentwerte eines Prozesses beziehen sich auf *einen* Kern: Ein Programm, das zwei Kerne voll nutzt, kann 200 % erreichen.

```python
import os
import threading
import time

import psutil


def beschaeftigen(sekunden):
    ende = time.monotonic() + sekunden
    while time.monotonic() < ende:
        pass


ich = psutil.Process()
ich.cpu_percent(interval=None)                    # Messung beginnt hier
arbeiter = threading.Thread(target=beschaeftigen, args=(1.2,))
arbeiter.start()
arbeiter.join()
prozent = ich.cpu_percent(interval=None)
print("Mein Prozess war beschäftigt, CPU-Anteil ueber 70 %:", prozent > 70, "| unter 110 %:", prozent < 110)

load1, load5, load15 = os.getloadavg()
print("Lastdurchschnitt hat drei Werte:", all(isinstance(w, float) for w in (load1, load5, load15)))
print("Last je Kern (Last / Kerne):", round(load1 / psutil.cpu_count(), 2) >= 0)
```

```text
Mein Prozess war beschäftigt, CPU-Anteil ueber 70 %: True | unter 110 %: True
Lastdurchschnitt hat drei Werte: True
Last je Kern (Last / Kerne): True
```

Der Prozess war fast die ganze Zeit beschäftigt, die Nutzung lag zwischen 70 und 110 Prozent (die Obergrenze ist großzügig, weil der Zeitraum nicht exakt der Rechenzeit entspricht). Der Lastdurchschnitt wird als drei Zahlen geliefert. Das Beispiel verwendet `time.monotonic()` für die Zeitmessung, denn die Systemzeit kann springen (siehe Modul 8), während die monotone Uhr nur vorwärts läuft.

**Arbeitsspeicher.** Beim Arbeitsspeicher (RAM) ist die Frage „wie viel ist frei?“ nicht so einfach, wie sie klingt. Linux verwendet Speicher, der gerade nicht gebraucht wird, für einen *Zwischenspeicher* (*page cache*), der Dateien im Speicher bereithält, damit sie nicht immer wieder von der Platte gelesen werden müssen. Dieser Speicher ist zwar belegt, aber nicht verloren: Wenn ein Programm Speicher braucht, gibt der Kern ihn sofort wieder her. Ein gesunder, lange laufender Linux-Rechner zeigt deshalb fast immer „wenig freien“ Speicher, und das ist kein Grund zur Sorge. Die ehrliche Größe ist der *verfügbare* Speicher (`available`): der Speicher, den neue Programme bekommen können, ohne dass der Rechner auslagern muss. `psutil.virtual_memory()` liefert unter anderem `total` (Gesamtgröße), `available` (verfügbar), `used` (belegt), `free` (wirklich unbenutzt) und `percent` (die Belegung in Prozent). Wir haben in der Prüfumgebung nachgerechnet, wie `percent` entsteht, und die Beziehung bestätigt: `percent` entspricht (total minus available) geteilt durch total mal 100. Die Folge ist: Für einen Alarm *„Speicher knapp“* wertet man `percent` oder `available` aus, niemals `free`.

Die Werte sind in *Bytes* angegeben. Unter Linux liest `psutil` diese Werte aus der Pseudodatei `/proc/meminfo` (dort in Kibibyte): `total` entspricht `MemTotal`, `available` entspricht `MemAvailable`. `psutil.swap_memory()` liefert dieselben Angaben für den *Auslagerungsspeicher* (*Swap*), den ein Rechner auf der Platte als Notreserve führt; ein Rechner, der dauernd intensiv auslagert, wird spürbar langsam. Hat ein System keinen Swap, sind die Werte null.

```python
import psutil

vm = psutil.virtual_memory()
print("Felder      :", [f for f in vm._fields if f in ("total", "available", "used", "free", "percent")])
print("available <= total:", vm.available <= vm.total, "| free <= available:", vm.free <= vm.available)

berechnet = (vm.total - vm.available) / vm.total * 100
print("percent = (total - available) / total * 100:", abs(vm.percent - berechnet) < 0.2)

meminfo = {}
with open("/proc/meminfo", encoding="ascii") as datei:
    for zeile in datei:
        name, _, rest = zeile.partition(":")
        meminfo[name] = int(rest.split()[0]) * 1024
print("total stimmt mit /proc/meminfo überein:", vm.total == meminfo["MemTotal"])
print("available stimmt mit MemAvailable überein (+/-1 MiB):", abs(vm.available - meminfo["MemAvailable"]) < 1024 * 1024)

swap = psutil.swap_memory()
print("Swap: total >= used:", swap.total >= swap.used, "| percent zwischen 0 und 100:", 0 <= swap.percent <= 100)
```

```text
Felder      : ['total', 'available', 'percent', 'used', 'free']
available <= total: True | free <= available: True
percent = (total - available) / total * 100: True
total stimmt mit /proc/meminfo überein: True
available stimmt mit MemAvailable überein (+/-1 MiB): True
Swap: total >= used: True | percent zwischen 0 und 100: True
```

Die Prüfungen laufen auf jedem Linux-Rechner durch. Sie sind eine gute Gewohnheit: Wer eine Bibliothek benutzt, sollte ihre Werte *einmal quergeprüft* haben (hier gegen `/proc/meminfo`), damit er weiß, was die Zahlen bedeuten.

**Lesbare Größen.** Für eine Ausgabe an Menschen will man keine Zahl wie `7696581394432`, sondern `7.0 TiB`. Dafür schreibt man sich eine kleine Hilfsfunktion, die man in jedem Programm wiederverwendet. Sie teilt so lange durch 1024, bis der Wert unter 1024 liegt, und wählt die passende Einheit. Beachten Sie die Schreibweise: *KiB, MiB, GiB* (Kibibyte, Mebibyte, Gibibyte) bezeichnen Vielfache von 1024, während *kB, MB, GB* (ohne *i*) nach dem Standard Vielfache von 1000 sind; Linux und `psutil` rechnen mit 1024.

```python
def lesbar(zahl, einheiten=("B", "KiB", "MiB", "GiB", "TiB")):
    """Wandelt Bytes in eine gut lesbare Größe um (Basis 1024)."""
    wert = float(zahl)
    for einheit in einheiten:
        if abs(wert) < 1024 or einheit == einheiten[-1]:
            return f"{wert:.0f} {einheit}" if einheit == "B" else f"{wert:.1f} {einheit}"
        wert /= 1024


for probe in (0, 999, 1024, 1536, 5 * 1024**2, 3 * 1024**3 + 512 * 1024**2, 7 * 1024**4):
    print(f"{probe:>16} -> {lesbar(probe)}")
```

```text
               0 -> 0 B
             999 -> 999 B
            1024 -> 1.0 KiB
            1536 -> 1.5 KiB
         5242880 -> 5.0 MiB
      3758096384 -> 3.5 GiB
   7696581394432 -> 7.0 TiB
```

Die Grenzfälle sind getestet: null Bytes, knapp unter einem Kibibyte, genau ein Kibibyte, Zwischenwerte und die größte Einheit. Die Schleife über `einheiten` hat eine Ausnahme für die letzte Einheit, damit auch Werte über 1024 TiB nicht aus der Liste fallen.

**Der Speicher eines einzelnen Prozesses.** Nicht nur der ganze Rechner, auch ein einzelner Prozess hat Kennzahlen. `psutil.Process()` ohne Argument beschreibt den *eigenen* Prozess; mit einer Prozessnummer (*PID*) einen beliebigen anderen (mehr dazu in 11.2). `Process.memory_info()` liefert `rss` (*resident set size*: der Speicher, der tatsächlich im RAM liegt) und `vms` (*virtual memory size*: der gesamte reservierte Adressraum, der weit größer sein kann, weil nicht alles tatsächlich benutzt wird). Für die Frage „wie viel Speicher braucht dieses Programm?“ ist `rss` die richtige Zahl. Das nächste Beispiel belegt absichtlich 200 Mebibyte, indem es einen Block von Bytes erzeugt, und sieht zu, wie `rss` wächst und nach dem Freigeben wieder fällt. Wichtig: Der Block wird *beschrieben* (`b"x" * n` füllt jedes Byte); ein bloß reservierter, aber nie benutzter Speicher würde in `rss` gar nicht auftauchen, weil Linux ihn erst bei der ersten Benutzung zuteilt.

```python
import psutil

ich = psutil.Process()
vorher = ich.memory_info().rss
block = b"x" * (200 * 1024 * 1024)          # 200 MiB, werden tatsächlich beschrieben
nachher = ich.memory_info().rss
zuwachs = (nachher - vorher) / 1024 / 1024
print("RSS gewachsen um mindestens 190 MiB:", zuwachs >= 190, "| höchstens 230 MiB:", zuwachs <= 230)

del block
danach = ich.memory_info().rss
print("Nach dem Freigeben wieder deutlich kleiner:", (nachher - danach) / 1024 / 1024 > 150)
print("memory_info-Felder:", [f for f in ich.memory_info()._fields if f in ("rss", "vms")])
```

```text
RSS gewachsen um mindestens 190 MiB: True | höchstens 230 MiB: True
Nach dem Freigeben wieder deutlich kleiner: True
memory_info-Felder: ['rss', 'vms']
```

Der Zuwachs liegt zwischen 190 und 230 Mebibyte, wie zu erwarten. Nach `del block` gibt Python den Speicher an das System zurück; hier fällt `rss` um mehr als 150 Mebibyte. Dass der Speicher *nicht immer* sofort an das Betriebssystem zurückgegeben wird (kleinere Objekte bleiben oft im Besitz von Python), ist ein bekanntes Verhalten, das man bei der Suche nach „Speicherlecks“ kennen muss.

**Eine Momentaufnahme.** Zum Abschluss fügen wir die Größen zu einer Übersicht zusammen, wie ein sehr einfaches `top`. Das Ergebnis hängt vom Rechner ab, weshalb dieses Beispiel nur Beispielwerte aus der Prüfumgebung zeigt (*keine* geprüfte Ausgabe; Ihre Zahlen werden abweichen).

```python
import os

import psutil


def lesbar(zahl):
    wert = float(zahl)
    for einheit in ("B", "KiB", "MiB", "GiB", "TiB"):
        if wert < 1024 or einheit == "TiB":
            return f"{wert:.1f} {einheit}"
        wert /= 1024


psutil.cpu_percent(interval=None)
cpu = psutil.cpu_percent(interval=1)
vm = psutil.virtual_memory()
swap = psutil.swap_memory()
last = os.getloadavg()

print(f"CPU     : {cpu:5.1f} % bei {psutil.cpu_count()} logischen Kernen")
print(f"Last    : {last[0]:.2f} / {last[1]:.2f} / {last[2]:.2f} (1 / 5 / 15 min)")
print(f"RAM     : {lesbar(vm.used)} von {lesbar(vm.total)} belegt ({vm.percent} %), verfügbar {lesbar(vm.available)}")
print(f"Swap    : {lesbar(swap.used)} von {lesbar(swap.total)} ({swap.percent} %)")
```

```text
CPU     :   0.0 % bei 2 logischen Kernen
Last    : 0.01 / 0.03 / 0.05 (1 / 5 / 15 min)
RAM     : 665.8 MiB von 7.8 GiB belegt (8.3 %), verfügbar 7.2 GiB
Swap    : 0.0 B von 0.0 B (0.0 %)
```

Der Code bildet das Muster, das wir in 11.5 und in Modul 12 für die Überwachung verwenden: Werte holen, schön formatieren, Schwellen vergleichen. Dass `psutil` Werte *liefert* und das Programm sie *bewertet*, ist die richtige Aufteilung: Die Schwellenwerte (zum Beispiel „Alarm ab 90 Prozent Speicher“) sind Betriebsentscheidungen, die in die Konfiguration gehören, nicht in die Messfunktion.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, den ersten `cpu_percent()`-Aufruf ohne Intervall zu verwerten: Er liefert immer `0.0`, und das Programm meldet „alles ruhig“. Der zweite ist, `free` statt `available` oder `percent` auszuwerten und bei einem gesunden Server ständig Alarm wegen „zu wenig Speicher“ zu schlagen. Der dritte ist, die Prozessorauslastung in einer engen Schleife abzufragen (jede Messung mit Intervall blockiert): Wer einmal pro Sekunde messen will, ruft `cpu_percent(interval=None)` im Sekundentakt auf und nutzt die Differenz zum vorigen Aufruf. Der vierte ist, Prozentwerte verschiedener Größen zu vergleichen: 100 % CPU eines Prozesses ist ein Kern, 100 % der Gesamtauslastung sind alle Kerne. Das folgende Gegenüber zeigt die ersten beiden Fehler.

```python
import psutil

# FALSCH
if psutil.cpu_percent() > 90:                    # erster Aufruf: immer 0.0, nie Alarm
    print("CPU überlastet")
if psutil.virtual_memory().free < 500 * 1024**2:  # Zwischenspeicher zählt als "nicht frei"
    print("Speicher knapp")

# RICHTIG
if psutil.cpu_percent(interval=1) > 90:
    print("CPU überlastet")
if psutil.virtual_memory().available < 500 * 1024**2:
    print("Speicher knapp")
```

Im falschen Teil schlägt die erste Prüfung nie an, weil der erste Aufruf null liefert, und die zweite schlägt auf jedem gesunden Linux-Server an, weil der Zwischenspeicher fast den ganzen freien Speicher belegt. Der richtige Teil misst mit Intervall und verwendet den verfügbaren Speicher, der auch den Zwischenspeicher berücksichtigt, der bei Bedarf freigegeben werden kann.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage und Messungen.** Die Dokumentation von `psutil` (psutil.readthedocs.io) war in der Prüfumgebung nicht abrufbar. Abgerufen wurde die Beschreibung auf PyPI (Plattformen, Funktionsbereiche). Alle Aussagen dieser Lerneinheit wurden mit psutil 7.2.2 selbst überprüft: dass der erste Aufruf von `cpu_percent(interval=None)` den Wert 0.0 liefert, dass `percent` der Beziehung (total minus available) durch total entspricht, dass `total` und `available` mit `MemTotal` und `MemAvailable` aus `/proc/meminfo` übereinstimmen und dass `rss` nach dem Beschreiben eines Blocks um dessen Größe wächst. Die Erklärung zum Zwischenspeicher ist Fachwissen über Linux. Die offizielle Dokumentation steht unter *weiterführend* im Quellenverzeichnis.

> **Merke**
>
> `psutil` liest Kennzahlen direkt vom Betriebssystem. `cpu_percent` ist ein Mittelwert über einen Zeitraum: mit `interval=1` messen, der erste Aufruf ohne Intervall ist wertlos. Die Last (`os.getloadavg()`) im Verhältnis zur Kernzahl beurteilt Überlastung besser als eine Momentauslastung. Für Speicher zählt `available` beziehungsweise `percent`, nicht `free`. Prozess-Speicher: `rss`. Größen für Menschen in 1024er-Einheiten (KiB, MiB, GiB) formatieren.

> **Praxisbeispiel**
>
> Fast jedes Überwachungsskript beginnt mit diesen Messungen: Alarm, wenn der Speicher dauerhaft über 90 Prozent liegt, wenn die Last über Minuten die Kernzahl übersteigt oder wenn ein einzelner Kern am Limit ist. Dabei kommt es nicht auf einen einzelnen Messwert an, sondern auf den *Verlauf*: Ein kurzer Ausschlag ist normal, drei Messungen in Folge über der Schwelle sind ein Befund. In 11.5 und Modul 12 bauen wir genau diese Logik in einen Health-Check ein.

## 11.2 psutil: Datenträger und Prozesse

Die zweite Hälfte der Systemüberwachung betrifft den dauerhaften Speicher und die laufenden Programme. Ein voller Datenträger gehört zu den häufigsten und ärgerlichsten Ursachen für Ausfälle: Datenbanken können nicht mehr schreiben, Protokolldateien wachsen nicht mehr, Programme brechen mit rätselhaften Fehlermeldungen ab. Und bei den Prozessen lautet die Alltagsfrage: Läuft das, was laufen soll, und was frisst gerade die Ressourcen? In dieser Lerneinheit beantworten wir beides mit `psutil`.

**Datenträger und Dateisysteme.** Zunächst die Begriffe. Ein Datenträger (Festplatte, SSD) wird in *Partitionen* unterteilt, und jede Partition erhält ein *Dateisystem* (zum Beispiel ext4 oder xfs unter Linux, NTFS unter Windows), das festlegt, wie Dateien auf dem Medium abgelegt werden. Damit man auf ein Dateisystem zugreifen kann, wird es in den Verzeichnisbaum *eingehängt* (*mount*); der Ort im Baum heißt *Mountpunkt*, etwa `/` für das Wurzelverzeichnis oder `/home` für ein getrenntes Heimlaufwerk. Unter Linux erscheinen außerdem viele *Pseudo-Dateisysteme* (`proc`, `sysfs`, `tmpfs` und andere), die gar keinen Platz auf einer Platte belegen, sondern Kernfunktionen als Dateien darstellen oder im Arbeitsspeicher liegen. Für eine Überwachung interessieren nur die echten Dateisysteme. `psutil.disk_partitions()` liefert die echten (`all=False`, Vorgabe), mit `all=True` bekommt man alle.

```python
import psutil

echte = psutil.disk_partitions()                 # nur echte Datenträger (all=False)
alle = psutil.disk_partitions(all=True)          # auch proc, sysfs, tmpfs ...
print("Felder einer Partition:", echte[0]._fields)
print("alle >= echte:", len(alle) >= len(echte))
print("Pseudo-Dateisysteme nur bei all=True:", {"proc", "sysfs"} & {p.fstype for p in alle} != set() and
      {"proc", "sysfs"} & {p.fstype for p in echte} == set())
mountpunkte = [p.mountpoint for p in echte]
print("Das Wurzelverzeichnis ist eingehängt:", "/" in mountpunkte)
```

```text
Felder einer Partition: ('device', 'mountpoint', 'fstype', 'opts')
alle >= echte: True
Pseudo-Dateisysteme nur bei all=True: True
Das Wurzelverzeichnis ist eingehängt: True
```

Das Beispiel zeigt die Felder einer Partition (unter anderem Gerät, Mountpunkt, Dateisystemtyp und Mount-Optionen) und bestätigt, dass die vollständige Liste mindestens so lang ist wie die der echten Datenträger und dass Pseudo-Dateisysteme nur mit `all=True` auftauchen. Die letzte Prüfung zeigt, dass das Wurzelverzeichnis in der Liste vorkommt. In einer Container-Umgebung wie der Prüfumgebung sind die Dateisysteme anders aufgebaut als auf einem physischen Server (ein Container sieht meist nur ein einziges Overlay-Dateisystem); die Auswertung muss mit beidem zurechtkommen. Genau deshalb enthält die spätere Hilfsfunktion eine Ausschlussliste.

Die Belegung eines Mountpunkts liefert `psutil.disk_usage(pfad)`. Das Ergebnis enthält `total`, `used`, `free` (alles Bytes) und `percent`. Dabei gibt es eine Feinheit: Auf ext4-Dateisystemen ist ein Teil des Platzes (üblicherweise fünf Prozent) für den Administrator *reserviert*, damit das System auch bei einer vollen Platte noch arbeiten kann. Dieser Platz taucht weder in `used` noch in `free` auf, weshalb `used + free` kleiner als `total` sein kann. `percent` berechnet `psutil` aus dem, was normale Benutzer tatsächlich nutzen können, also aus `used` im Verhältnis zu `used + free`. Das stimmt mit dem überein, was das Werkzeug `df` in der Spalte „Use%“ zeigt. Das nächste Beispiel prüft diese Beziehungen und vergleicht die Ergebnisse mit `shutil.disk_usage` aus der Standardbibliothek, die dieselben Rohzahlen liefert, aber weder Prozentwert noch Partitionsliste kennt.

```python
import shutil
import tempfile

import psutil

pfad = tempfile.gettempdir()
platte = psutil.disk_usage(pfad)
print("Felder:", platte._fields)
print("total stimmt mit shutil überein:", platte.total == shutil.disk_usage(pfad).total)
print("free stimmt mit shutil überein (+/-10 MiB):", abs(platte.free - shutil.disk_usage(pfad).free) < 10 * 1024 * 1024)
print("percent = used / (used + free):", abs(platte.percent - 100 * platte.used / (platte.used + platte.free)) < 0.2)
print("used + free ist weniger als total (reservierter Platz):", platte.used + platte.free <= platte.total)
```

```text
Felder: ('total', 'used', 'free', 'percent')
total stimmt mit shutil überein: True
free stimmt mit shutil überein (+/-10 MiB): True
percent = used / (used + free): True
used + free ist weniger als total (reservierter Platz): True
```

Beachten Sie bei der Toleranz im zweiten Vergleich, dass sich der freie Platz zwischen den beiden Aufrufen ändern kann (andere Programme schreiben), ein exakter Vergleich wäre also eine Fehlerquelle in Tests. Das ist eine allgemeine Regel: Wer sich ändernde Werte vergleicht, braucht Toleranzen.

Daraus bauen wir eine brauchbare Prüffunktion. Sie geht alle Dateisysteme durch, ignoriert Pseudo-Dateisysteme und doppelte Mountpunkte (derselbe Mountpunkt kann mehrfach aufgelistet sein), fängt Fehler ab, wenn ein Mountpunkt nicht lesbar ist (`PermissionError`), und ordnet jedes Ergebnis einer Stufe zu: „ok“, „WARNUNG“ oder „KRITISCH“. Die Schwellen sind Parameter mit Vorgabewerten, sodass sie sich ohne Eingriff in die Funktion ändern lassen. Zum Testen läuft die Funktion mit absichtlich extremen Schwellen: Bei 101 Prozent kann nichts jemals eine Warnung auslösen, bei 0 Prozent löst jedes Dateisystem eine aus.

```python
import psutil

PSEUDO = {"tmpfs", "devtmpfs", "squashfs", "overlay", "proc", "sysfs", "cgroup", "cgroup2"}


def plattenbericht(warnung=80, kritisch=90):
    """Prüft alle echten Dateisysteme und liefert (Mountpunkt, Prozent, Stufe)."""
    ergebnis = []
    gesehen = set()
    for teil in psutil.disk_partitions(all=True):
        if teil.fstype in PSEUDO or teil.mountpoint in gesehen:
            continue
        gesehen.add(teil.mountpoint)
        try:
            nutzung = psutil.disk_usage(teil.mountpoint)
        except (PermissionError, OSError):
            continue
        stufe = "KRITISCH" if nutzung.percent >= kritisch else "WARNUNG" if nutzung.percent >= warnung else "ok"
        ergebnis.append((teil.mountpoint, nutzung.percent, stufe))
    return sorted(ergebnis, key=lambda e: -e[1])


from collections import Counter

for warnung, kritisch in ((101, 101), (0, 101), (0, 0)):
    stufen = Counter(stufe for _, _, stufe in plattenbericht(warnung, kritisch))
    print(f"Schwellen {warnung:>3}/{kritisch:>3}:", sorted(stufen))

anzahl = len(plattenbericht())
print("Geprüfte Dateisysteme:", anzahl >= 1)
```

```text
Schwellen 101/101: ['ok']
Schwellen   0/101: ['WARNUNG']
Schwellen   0/  0: ['KRITISCH']
Geprüfte Dateisysteme: True
```

Die erste Zeile zeigt, dass bei unerreichbaren Schwellen nur „ok“ vorkommt, die zweite, dass bei Warnschwelle null nur „WARNUNG“ erscheint (die kritische Schwelle liegt noch bei 101), die dritte, dass bei beiden Schwellen auf null nur „KRITISCH“ übrig bleibt. Diese Methode, eine Entscheidungsfunktion mit Grenzwerten zu testen, die jeden Zweig erzwingen, ist auch ohne Zugriff auf echte volle Platten möglich und sollte Standard sein. Die Sortierung nach absteigendem Füllstand sorgt dafür, dass das gefährlichste Dateisystem als Erstes in jedem Bericht steht.

**Prozesse.** Ein *Prozess* ist ein laufendes Programm mit eigenem Speicher, eigener Nummer (*PID*, *process identifier*), einem Elternprozess (der ihn gestartet hat, *PPID*) und einem Besitzer. Das Betriebssystem startet Prozesse in einem Baum, an dessen Wurzel ein Systemprozess (unter Linux meist `systemd` oder `init` mit PID 1) steht. `psutil.Process(pid)` erzeugt ein Objekt, über das sich alle Eigenschaften abfragen lassen: `name()`, `cmdline()` (die vollständige Befehlszeile als Liste), `status()`, `username()`, `create_time()`, `num_threads()`, `ppid()`, `children()`, `memory_info()` und `cpu_percent()`. Wir erzeugen zum Üben selbst einen Prozess, damit die Prüfung unabhängig vom Rechner funktioniert: Ein Kindprozess, der einfach 30 Sekunden schläft, und mit dem `subprocess`-Modul (Modul 5) gestartet wird.

```python
import os
import subprocess
import sys
import time

import psutil

kind = subprocess.Popen([sys.executable, "-c", "import time; time.sleep(30)", "--rolle=demo"])
time.sleep(0.3)

p = psutil.Process(kind.pid)
print("pid stimmt        :", p.pid == kind.pid)
print("Elternprozess ist ich:", p.ppid() == os.getpid())
print("Name              :", p.name().startswith("python"))
print("Befehlszeile      :", p.cmdline()[-1], "| Anzahl Teile:", len(p.cmdline()))
print("Status            :", p.status())
print("Benutzer ist meiner:", p.username() == psutil.Process().username())
print("läuft seit < 5 s  :", time.time() - p.create_time() < 5)
print("Threads           :", p.num_threads())
print("Kinder meines Prozesses:", [c.pid == kind.pid for c in psutil.Process().children()])

p.terminate()                                  # sendet SIGTERM
rueckgabe = p.wait(timeout=5)
print("Rückgabe von wait :", rueckgabe, "(negativ = durch Signal beendet; -15 = SIGTERM)")
print("läuft noch        :", p.is_running())
kind.wait()
```

```text
pid stimmt        : True
Elternprozess ist ich: True
Name              : True
Befehlszeile      : --rolle=demo | Anzahl Teile: 4
Status            : sleeping
Benutzer ist meiner: True
läuft seit < 5 s  : True
Threads           : 1
Kinder meines Prozesses: [True]
Rückgabe von wait : -15 (negativ = durch Signal beendet; -15 = SIGTERM)
läuft noch        : False
```

Das Beispiel liest alle wichtigen Eigenschaften und prüft sie gegen das, was wir wissen. Besonders wichtig sind die letzten Zeilen: `p.terminate()` sendet das Signal `SIGTERM`, die höfliche Aufforderung zum Beenden, und `p.wait(timeout=5)` wartet auf das Ende. Der Rückgabewert `-15` bedeutet: Der Prozess wurde durch das Signal mit der Nummer 15 beendet (negativ, weil er keinen eigenen Exit-Code hat). Wenn ein Prozess auf `terminate()` nicht reagiert, bleibt als letztes Mittel `p.kill()` (`SIGKILL`, das sich nicht abfangen lässt); es sollte erst nach einer Wartezeit eingesetzt werden, weil das Programm keine Gelegenheit zum Aufräumen bekommt. Das Beispiel gehört zu den Dingen, die Sie nur mit eigenen Prozessen oder mit ausreichenden Rechten tun dürfen: Andere Benutzer-Prozesse beenden darf nur `root`.

Um *alle* Prozesse zu durchlaufen, gibt es `psutil.process_iter(attrs)`. Wichtig ist der Parameter `attrs`, der angibt, welche Eigenschaften gleich mitgelesen werden sollen; sie landen im Wörterbuch `proc.info`. Das ist deutlich schneller als später für jeden Prozess einzeln nachzufragen. Das nächste Beispiel sucht die drei Prozesse mit dem größten Speicherbedarf (`rss`). Die Ausgabe zeigt Namen aus der Prüfumgebung und wird deshalb bei Ihnen anders aussehen.

```python
import psutil

alle = []
for proc in psutil.process_iter(["pid", "name", "memory_info"]):
    info = proc.info                              # Wörterbuch mit den angeforderten Feldern
    if info["memory_info"] is not None:
        alle.append((info["memory_info"].rss, info["pid"], info["name"]))

alle.sort(reverse=True)
print("Prozesse gefunden:", len(alle) >= 1)
print("Eigener Prozess dabei:", any(pid == psutil.Process().pid for _, pid, _ in alle))
for rss, pid, name in alle[:3]:
    print(f"  {name[:15]:15} pid={pid:<7} {rss / 1024 / 1024:8.1f} MiB")
```

```text
Prozesse gefunden: True
Eigener Prozess dabei: True
  claude          pid=73         346.0 MiB
  environment-man pid=168         46.0 MiB
  rclone-filestor pid=80          33.0 MiB
```

Die ersten beiden Zeilen sind überprüfbare Aussagen, die dritte bis fünfte sind eine Beispielausgabe. Die Sortierung nach dem Tupel `(rss, pid, name)` funktioniert, weil Python Tupel Element für Element vergleicht; der Speicher entscheidet, die PID nur bei Gleichstand. Beachten Sie die Prüfung auf `None`: Bei Prozessen, deren Informationen nicht lesbar sind, liefert `process_iter` für das betroffene Feld `None`, statt einen Fehler zu melden.

**Wenn Prozesse verschwinden oder gesperrt sind.** Prozesse sind flüchtig. Zwischen dem Moment, in dem Sie die Liste der Prozesse abfragen, und dem Moment, in dem Sie ein Detail nachfragen, kann ein Prozess schon beendet sein, und Sie sehen dann die Ausnahme `psutil.NoSuchProcess`. Möchten Sie die Details eines Prozesses lesen, der einem anderen Benutzer gehört und dessen Daten gesperrt sind, erhalten Sie `psutil.AccessDenied`. Beide sind Unterklassen von `psutil.Error`; ein Prozess, der beendet ist, aber noch nicht von seinem Elternprozess eingesammelt wurde (ein sogenannter *Zombie*), löst `ZombieProcess` aus, eine Unterklasse von `NoSuchProcess`. Eine robuste Auswertung fängt alle drei Fälle und behandelt sie als „diesen Prozess überspringen“, nicht als Programmfehler. Außerdem gibt es den Kontextmanager `proc.oneshot()`: Innerhalb davon werden mehrere Abfragen mit einem einzigen Zugriff auf das Betriebssystem beantwortet, was bei vielen Prozessen merklich schneller ist.

```python
import psutil

try:
    psutil.Process(4_194_305)                   # größer als jede mögliche PID
except psutil.NoSuchProcess as fehler:
    print("NoSuchProcess:", fehler.__class__.__mro__[1].__name__, "| pid =", fehler.pid)

print("AccessDenied erbt von:", psutil.AccessDenied.__mro__[1].__name__)
print("ZombieProcess erbt von:", psutil.ZombieProcess.__mro__[1].__name__)


def sichere_info(proc):
    """Liest Name und Befehlszeile; Prozesse, die verschwinden oder gesperrt sind, liefern None."""
    try:
        with proc.oneshot():                      # mehrere Abfragen in einem Rutsch
            return proc.name(), proc.cmdline()
    except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
        return None


gelesen = sum(1 for p in psutil.process_iter() if sichere_info(p) is not None)
print("Mindestens ein Prozess lesbar:", gelesen >= 1)
```

```text
NoSuchProcess: Error | pid = 4194305
AccessDenied erbt von: Error
ZombieProcess erbt von: NoSuchProcess
Mindestens ein Prozess lesbar: True
```

Die erste Zeile erzeugt absichtlich den Fehler `NoSuchProcess`, indem sie eine Prozessnummer abfragt, die es nicht geben kann (unter Linux liegt die größtmögliche PID bei höchstens 4 194 304). Die Ausgabe zeigt die Vererbung der Ausnahmen, und die Hilfsfunktion `sichere_info` ist das Muster, das Sie überall verwenden, wo Sie viele Prozesse auslesen: Alle drei Ausnahmen fangen, `None` zurückgeben, später filtern.

**Welcher Prozess belegt diesen Port?** Eine klassische Aufgabe im Betrieb: Ein Dienst startet nicht, weil „der Port schon belegt ist“. Mit `psutil.net_connections(kind="tcp")` bekommt man alle TCP-Verbindungen samt PID des Besitzers. Das Beispiel öffnet selbst einen lauschenden Socket auf einem vom System gewählten Port und findet ihn anschließend wieder. Dabei bestimmt `bind(("127.0.0.1", 0))` mit der Portnummer null, dass das System einen freien Port auswählt, den man mit `getsockname()` erfährt. Beachten Sie, dass `net_connections` auf manchen Systemen Administratorrechte benötigt, um die Prozessnummern fremder Verbindungen zu sehen; die eigenen sieht man immer.

```python
import os
import socket

import psutil

server = socket.socket()
server.bind(("127.0.0.1", 0))
server.listen()
port = server.getsockname()[1]

treffer = [v for v in psutil.net_connections(kind="tcp")
           if v.status == psutil.CONN_LISTEN and v.laddr.port == port]
print("Port gefunden:", len(treffer) == 1)
print("gehört zu meinem Prozess:", treffer[0].pid == os.getpid())
print("Name des Prozesses:", psutil.Process(treffer[0].pid).name().startswith("python"))
server.close()
```

```text
Port gefunden: True
gehört zu meinem Prozess: True
Name des Prozesses: True
```

Alle drei Aussagen sind wahr: Es gibt genau einen lauschenden Eintrag mit diesem Port, er gehört zum eigenen Prozess, und der Prozessname beginnt mit „python“. Mit derselben Methode finden Sie in der Praxis, welcher Dienst Port 8080 blockiert: nach `laddr.port == 8080` und Status `CONN_LISTEN` filtern, dann die PID in einen Namen übersetzen.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Arbeiten mit Prozessen ist, davon auszugehen, dass sie existieren und lesbar sind. Der zweite ist, Prozesse über ihren Namen zu suchen und dabei das zu übersehen, dass Skripte in Python alle als `python3` auftauchen: Wer wissen will, ob *sein* Skript läuft, muss die Befehlszeile (`cmdline()`) prüfen. Der dritte ist, `kill()` als erstes Mittel einzusetzen. Der vierte, Datenträger nur nach Prozenten zu beurteilen: Eine 2-Terabyte-Platte mit 90 Prozent hat noch 200 Gigabyte frei, eine 20-Gigabyte-Platte mit 90 Prozent nur noch 2 Gigabyte. Gute Überwachung kennt beide Sichtweisen. Das Gegenüber zu den ersten beiden Fehlern:

```python
import psutil

# FALSCH: Prozess kann zwischen Liste und Abfrage verschwinden; Name sagt nichts
for p in psutil.process_iter():
    if p.name() == "python3":              # NoSuchProcess / AccessDenied möglich
        print(p.pid)

# RICHTIG: Ausnahmen fangen, Befehlszeile prüfen
def laeuft(skriptname):
    for p in psutil.process_iter():
        try:
            if any(skriptname in teil for teil in p.cmdline()):
                return True
        except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
            continue
    return False
```

Die falsche Variante bricht beim ersten verschwundenen oder gesperrten Prozess mit einer Ausnahme ab und findet obendrein jedes Python-Programm statt des gesuchten. Die richtige durchsucht die Befehlszeilen (der Skriptname steht dort als eines der Elemente) und überspringt Prozesse, die sich nicht lesen lassen. Beachten Sie die Einschränkung: Ein Prozess, den Sie nicht lesen dürfen, kann das gesuchte Skript sein; „nicht gefunden“ ist dann nur „nicht gesehen“. Wer das genau wissen muss, prüft stattdessen die Sperrdatei oder den Port (Modul 9, 9.4).

> **Ergänzung aus der Nachrecherche**
>
> **Rechte und Container.** Die Beispiele wurden als Administrator in einer Container-Umgebung ausgeführt. Auf einem normalen Server mit einem unprivilegierten Benutzer werden Sie bei Prozessen anderer Benutzer häufiger `AccessDenied` sehen, und `net_connections` zeigt die Prozessnummer fremder Verbindungen nur mit höheren Rechten. In Containern (Docker, Kubernetes) sieht `psutil` je nach Konfiguration die Werte des *Wirtsrechners* statt der des Containers: Arbeitsspeichergrenzen eines Containers werden von `virtual_memory()` nicht berücksichtigt. Das ist Fachwissen zum Verhalten von Linux-Containern und wurde hier nicht mit der psutil-Dokumentation belegt (diese war nicht abrufbar).

> **Merke**
>
> Datenträger: `disk_partitions()` für die echten Dateisysteme, `disk_usage(mountpunkt)` für die Belegung; Pseudo-Dateisysteme und doppelte Mountpunkte ausschließen, Fehler abfangen, Schwellen als Parameter, Prozentwerte durch freie Gigabyte ergänzen. Prozesse: `Process(pid)`, `process_iter(attrs)`; immer mit `NoSuchProcess`, `AccessDenied` und `ZombieProcess` rechnen; beenden mit `terminate()`, warten, erst dann `kill()`. Port gesucht: `net_connections(kind="tcp")`.

> **Praxisbeispiel**
>
> „Platte voll“ ist der Klassiker unter den Störungen, und fast jede Überwachung hat dafür eine Prüfung, die genau der Funktion `plattenbericht` entspricht. Die Prozess-Funktionen brauchen Sie für „läuft mein Dienst noch?“, „welches Programm blockiert den Port?“ und „wer verbraucht gerade den Speicher?“. In 11.5 werden die Messwerte gebündelt und von mehreren Servern gleichzeitig abgeholt.

## 11.3 SSH mit paramiko und fabric

Bisher haben wir den Rechner untersucht, auf dem unser Programm läuft. In der Praxis betreut man aber fast immer *mehrere* Rechner: Webserver, Datenbankserver, Arbeitsplätze. Der Standardweg, einen entfernten Linux-Rechner zu bedienen, ist *SSH* (*Secure Shell*). Das Protokoll baut eine verschlüsselte Verbindung zwischen zwei Rechnern auf, weist beide Seiten gegenseitig aus (der Server beweist seine Identität gegenüber dem Client durch seinen *Hostschlüssel*, der Benutzer die seine durch ein Passwort oder besser durch einen *Schlüssel*) und erlaubt dann Befehle, Dateiübertragung (*SFTP*) und Weiterleitung von Netzwerkverbindungen. In dieser Lerneinheit steuern wir SSH aus Python heraus, mit zwei Bibliotheken, die aufeinander aufbauen: `paramiko` ist eine reine Python-Umsetzung des SSH-Protokolls, die direkten Zugriff auf alle Einzelheiten bietet; `fabric` setzt darauf auf und bietet eine besonders bequeme Oberfläche für das, was man am häufigsten braucht, nämlich einen Befehl auf einem oder mehreren Servern auszuführen.

**Wie die Beispiele ohne fremde Server laufen.** Zum Üben brauchen Sie Server, an denen Sie nichts kaputt machen können. Das Hilfsmodul `testssh` startet deshalb auf Ihrem eigenen Rechner mehrere kleine SSH-Server (`sshd`), die jeweils auf einer eigenen Adresse im Bereich `127.0.0.11`, `127.0.0.12` und so weiter lauschen (alle Adressen aus `127.0.0.0/8` gehören zum eigenen Rechner). Jeder Server bekommt einen eigenen Hostschlüssel und kennt den öffentlichen Teil eines gemeinsamen Benutzerschlüssels; Anmeldung per Passwort ist abgeschaltet. Über die Umgebungsvariable `HOSTALIAS`, die der Server jedem Befehl mitgibt, erkennen wir, welcher Server geantwortet hat; ansonsten laufen alle Befehle auf Ihrem Rechner mit Ihren Rechten. Das Modul ist reines Übungsgerüst und *kein Muster für einen echten Server*: Es schaltet Prüfungen ab (`StrictModes no`), die ein produktiver Server braucht. Speichern Sie den Code unter dem Namen `testssh.py` im selben Ordner wie die folgenden Beispiele.

```python
"""testssh - startet lokale SSH-Server fuer die Beispiele von Modul 11 (nur auf 127.0.0.x)."""
import getpass
import os
import socket
import subprocess
import tempfile
import time
from pathlib import Path

SSHD = "/usr/sbin/sshd"


class Umgebung:
    """Mehrere sshd-Instanzen mit eigenem Hostschluessel, ein gemeinsamer Clientschluessel."""

    def __init__(self):
        self.ordner = Path(tempfile.mkdtemp(prefix="testssh_"))
        self.benutzer = getpass.getuser()
        self.hosts = {}               # name -> (adresse, port)
        self._prozesse = {}
        self.port = self._freier_port()
        self.schluessel = self.ordner / "client"
        self._keygen(self.schluessel)
        self.autorisiert = self.ordner / "authorized_keys"
        self.autorisiert.write_text(self.schluessel.with_suffix(".pub").read_text())
        self.autorisiert.chmod(0o600)
        self.leer = self.ordner / "authorized_keys_leer"
        self.leer.write_text("")
        self.leer.chmod(0o600)

    @staticmethod
    def _freier_port():
        with socket.socket() as s:
            s.bind(("127.0.0.1", 0))
            return s.getsockname()[1]

    @staticmethod
    def _keygen(pfad):
        subprocess.run(["ssh-keygen", "-q", "-t", "ed25519", "-N", "", "-f", str(pfad)], check=True)

    def starte_host(self, name, nummer, autorisiert=True):
        adresse = f"127.0.0.{10 + nummer}"
        hostkey = self.ordner / f"hostkey_{name}"
        if not hostkey.exists():
            self._keygen(hostkey)
        keys = self.autorisiert if autorisiert else self.leer
        konfig = self.ordner / f"sshd_{name}.conf"
        konfig.write_text(
            f"Port {self.port}\nListenAddress {adresse}\nHostKey {hostkey}\n"
            f"AuthorizedKeysFile {keys}\nPidFile {self.ordner}/sshd_{name}.pid\n"
            "PasswordAuthentication no\nKbdInteractiveAuthentication no\nUsePAM no\n"
            f"StrictModes no\nLogLevel ERROR\nSetEnv HOSTALIAS={name}\n"
            "Subsystem sftp internal-sftp\n")
        if os.geteuid() == 0:
            Path("/run/sshd").mkdir(exist_ok=True)
        self._prozesse[name] = subprocess.Popen([SSHD, "-D", "-f", str(konfig)])
        self.hosts[name] = (adresse, self.port)
        self._warte(adresse)

    def _warte(self, adresse):
        for _ in range(50):
            try:
                socket.create_connection((adresse, self.port), timeout=0.2).close()
                return
            except OSError:
                time.sleep(0.1)
        raise RuntimeError(f"sshd auf {adresse} startet nicht")

    def neuer_hostschluessel(self, name):
        """Simuliert eine Neuinstallation: gleicher Rechner, neuer Hostschluessel."""
        nummer = int(self.hosts[name][0].rsplit(".", 1)[1]) - 10
        self.stoppe(name)
        (self.ordner / f"hostkey_{name}").unlink()
        (self.ordner / f"hostkey_{name}.pub").unlink()
        self.starte_host(name, nummer)

    def stoppe(self, name=None):
        for n in ([name] if name else list(self._prozesse)):
            prozess = self._prozesse.pop(n, None)
            if prozess:
                prozess.terminate()
                prozess.wait()
            if name:
                self.hosts.pop(n, None)


def starte(namen=("web01", "web02", "db01")):
    umgebung = Umgebung()
    for nummer, name in enumerate(namen, start=1):
        umgebung.starte_host(name, nummer)
    return umgebung
```

Sie müssen das Modul nicht im Einzelnen verstehen; die Funktion `starte(namen)` startet für jeden Namen einen Server und liefert ein Objekt, dessen Attribut `hosts` je Namen die Adresse und den Port enthält, dessen `benutzer` und `schluessel` die Anmeldedaten bereitstellen und dessen Methode `stoppe()` alles beendet. Alles, was Sie für die Praxis lernen, steht in den Beispielen selbst; das Hilfsmodul sorgt nur dafür, dass es sie ohne fremde Server gibt. Es benötigt das Programm `sshd` aus dem Paket `openssh-server`; in der Prüfumgebung wurde es als Administrator getestet.

**Zum Vergleich: das Programm `ssh` aus Python.** Der einfachste Weg, SSH aus Python zu benutzen, ist, das Kommandozeilenprogramm `ssh` mit `subprocess` aufzurufen. Das ist völlig legitim und für einzelne Befehle oft ausreichend. Das erste Beispiel startet einen Befehl auf `web01` und liest Ausgabe und Exit-Code. Die Option `-i` gibt die Schlüsseldatei an, `-p` den Port; die Optionen `StrictHostKeyChecking=no` und `UserKnownHostsFile=/dev/null` schalten die Prüfung des Hostschlüssels *ab* und sind hier nur deshalb vertretbar, weil die Testserver bei jedem Start neue Schlüssel bekommen; in Abschnitt 11.4 behandeln wir ausführlich, warum man sie in der Praxis nicht verwenden darf.

```python
import subprocess

import testssh

umgebung = testssh.starte(["web01"])
adresse, port = umgebung.hosts["web01"]

r = subprocess.run(
    ["ssh", "-i", str(umgebung.schluessel), "-p", str(port),
     "-o", "StrictHostKeyChecking=no", "-o", "UserKnownHostsFile=/dev/null", "-o", "LogLevel=ERROR",
     f"{umgebung.benutzer}@{adresse}", "echo Hallo von $HOSTALIAS; exit 3"],
    capture_output=True, text=True)
print("stdout    :", r.stdout.strip())
print("Exit-Code :", r.returncode)
umgebung.stoppe()
```

```text
stdout    : Hallo von web01
Exit-Code : 3
```

Der Befehl `echo Hallo von $HOSTALIAS; exit 3` wird *auf dem Server* ausgeführt, die Variable also dort aufgelöst, deshalb steht der Servername in der Ausgabe. Der Exit-Code 3 des Befehls kommt als Rückgabewert des `ssh`-Programms zurück; Exit-Code 255 hieße, dass SSH selbst scheiterte (Verbindung, Anmeldung). Diese Methode hat Grenzen: Wer viele Befehle nacheinander absetzt, baut jedes Mal eine neue Verbindung auf, und Fehlerarten wie „Verbindung abgelehnt“ und „Befehl fehlgeschlagen“ lassen sich nur an Textmeldungen und Codes unterscheiden. Eine Bibliothek löst beides.

**paramiko: der SSHClient.** Zentral ist die Klasse `paramiko.SSHClient`. Man erzeugt ein Objekt, legt eine Regel für unbekannte Hostschlüssel fest, ruft `connect()` mit Adresse, Benutzer und Schlüssel auf und führt dann mit `exec_command()` Befehle aus. Das Besondere ist die Standardregel: Ein frisch erzeugter `SSHClient` *lehnt unbekannte Server ab* (die Regel heißt `RejectPolicy`), und das ist eine gute Entscheidung der Entwickler: Wer sich mit einem Server verbindet, dessen Schlüssel er nicht kennt, kann nicht wissen, ob er mit dem richtigen Server oder mit einem Angreifer spricht. Das erste Beispiel zeigt beide Seiten. Zuerst schlägt die Verbindung wegen des unbekannten Schlüssels fehl; dann erlauben wir mit `AutoAddPolicy()` ausdrücklich, unbekannte Schlüssel automatisch zu übernehmen (nur zum Ausprobieren!), verbinden uns und lesen Standardausgabe, Fehlerausgabe und Exit-Code.

```python
import paramiko

import testssh

umgebung = testssh.starte(["web01"])
adresse, port = umgebung.hosts["web01"]
optionen = dict(port=port, username=umgebung.benutzer, key_filename=str(umgebung.schluessel),
                timeout=5, allow_agent=False, look_for_keys=False)

# 1. Standard: unbekannte Hostschlüssel werden abgelehnt
client = paramiko.SSHClient()
try:
    client.connect(adresse, **optionen)
except paramiko.SSHException as fehler:
    print("1. Standardverhalten:", type(fehler).__name__, "-", str(fehler).replace(str(port), "PORT"))

# 2. Nur zum Ausprobieren: unbekannte Schlüssel automatisch annehmen
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(adresse, **optionen)
stdin, stdout, stderr = client.exec_command("echo Hallo von $HOSTALIAS; echo Warnung >&2; exit 4")
ausgabe = stdout.read().decode()
fehler = stderr.read().decode()
exitcode = stdout.channel.recv_exit_status()
print("2. stdout:", repr(ausgabe), "| stderr:", repr(fehler), "| Exit-Code:", exitcode)
client.close()
umgebung.stoppe()
```

```text
1. Standardverhalten: SSHException - Server '[127.0.0.11]:PORT' not found in known_hosts
2. stdout: 'Hallo von web01\n' | stderr: 'Warnung\n' | Exit-Code: 4
```

Die erste Zeile zeigt die Fehlermeldung mit dem Typ `SSHException`; die Portnummer ist durch das Wort PORT ersetzt, damit die Ausgabe stabil bleibt. Die zweite Zeile zeigt das Muster zum Lesen von Ergebnissen. `exec_command()` liefert drei Dateiobjekte: `stdin` zum Schreiben an den Befehl, `stdout` und `stderr` zum Lesen. Der Befehl läuft auf dem Server, und das Lesen mit `.read()` wartet, bis er seine Ausgabe fertig geschrieben hat; die Fehlerausgabe wird getrennt gelesen. Den Exit-Code erhält man erst mit `stdout.channel.recv_exit_status()`, das blockiert, bis der Befehl beendet ist. Ein häufiger Fehler ist, diese Reihenfolge zu vergessen und den Exit-Code zu früh abzufragen oder die Ausgabe nicht zu lesen, wodurch der Befehl bei großen Ausgaben hängen bleiben kann (der Puffer läuft voll).

Aus diesen Bausteinen bauen wir uns eine kleine, wiederverwendbare Bibliothek: eine Klasse `Ergebnis`, die Exit-Code und Ausgaben bündelt, eine Funktion `ssh_ausfuehren(client, befehl, timeout)`, die den gesamten Ablauf kapselt, und eine Funktion `verbinde(...)`, die einen Client mit sinnvollen Zeitgrenzen aufbaut. Die Zeitgrenzen (`timeout`, `banner_timeout`, `auth_timeout`) sind wichtig: Ohne sie wartet ein Programm im ungünstigsten Fall sehr lange auf einen Server, der nicht antwortet, und blockiert damit alle weiteren (siehe Modul 10, 10.3). Die Optionen `allow_agent=False` und `look_for_keys=False` sorgen dafür, dass nur der ausdrücklich genannte Schlüssel verwendet wird und nicht zufällig auch andere, die auf dem Rechner herumliegen; das macht das Verhalten vorhersagbar.

```python
from dataclasses import dataclass

import paramiko


@dataclass
class Ergebnis:
    exitcode: int
    stdout: str
    stderr: str

    @property
    def ok(self):
        return self.exitcode == 0


def ssh_ausfuehren(client, befehl, timeout=10):
    """Führt einen Befehl aus, wartet auf das Ende und liefert Exit-Code und Ausgaben."""
    stdin, stdout, stderr = client.exec_command(befehl, timeout=timeout)
    stdin.close()
    ausgabe = stdout.read().decode("utf-8", errors="replace")
    fehler = stderr.read().decode("utf-8", errors="replace")
    return Ergebnis(stdout.channel.recv_exit_status(), ausgabe, fehler)


def verbinde(adresse, port, benutzer, schluessel, timeout=5, hostkeys=None):
    """Baut eine Verbindung auf. Ohne Datei 'hostkeys' werden unbekannte Server angenommen (nur für Tests!)."""
    client = paramiko.SSHClient()
    if hostkeys:
        client.load_host_keys(str(hostkeys))
        client.set_missing_host_key_policy(paramiko.RejectPolicy())
    else:
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(adresse, port=port, username=benutzer, key_filename=str(schluessel),
                   timeout=timeout, banner_timeout=timeout, auth_timeout=timeout,
                   allow_agent=False, look_for_keys=False)
    return client
```

Der Parameter `hostkeys` legt fest, ob unbekannte Server abgelehnt werden (mit Datei) oder angenommen (ohne Datei); der Kommentar im Docstring macht die Gefahr sichtbar, und in 11.4 setzen wir das strenge Verhalten als Regelfall ein. Zum Speichern unter dem Namen `sshlib.py`, damit die folgenden Beispiele es einbinden können. Der Einsatz sieht so aus:

```python
import testssh
from sshlib import ssh_ausfuehren, verbinde

umgebung = testssh.starte(["web01"])
adresse, port = umgebung.hosts["web01"]

client = verbinde(adresse, port, umgebung.benutzer, umgebung.schluessel)
try:
    for befehl in ("echo $HOSTALIAS", "cat /nicht/vorhanden", "exit 9", "printf 'Zeile 1\\nZeile 2\\n' | wc -l"):
        r = ssh_ausfuehren(client, befehl)
        print(f"{befehl!r:45} -> Exit {r.exitcode}, ok={r.ok}, stdout={r.stdout.strip()!r}, stderr={r.stderr.strip()[:30]!r}")
finally:
    client.close()
umgebung.stoppe()
```

```text
'echo $HOSTALIAS'                             -> Exit 0, ok=True, stdout='web01', stderr=''
'cat /nicht/vorhanden'                        -> Exit 1, ok=False, stdout='', stderr='cat: /nicht/vorhanden: No such'
'exit 9'                                      -> Exit 9, ok=False, stdout='', stderr=''
"printf 'Zeile 1\\nZeile 2\\n' | wc -l"       -> Exit 0, ok=True, stdout='2', stderr=''
```

Vier Befehle, vier Ergebnisse: Der erste zeigt den Servernamen, der zweite scheitert, weil die Datei fehlt, und zeigt, dass die Fehlermeldung auf `stderr` landet, der dritte liefert nur einen Exit-Code, und der vierte zeigt, dass auf dem Server auch Pipelines funktionieren, denn die Befehlszeile wird dort von einer Shell ausgeführt. Dasselbe heißt aber auch: Setzen Sie Befehlszeilen nie aus unvalidierten Eingaben zusammen, sonst gelingt einem Angreifer die Einschleusung eigener Befehle (*Command Injection*; siehe Modul 5 zu `shell=True`). Wer Parameter einfügen muss, nimmt `shlex.quote()`.

**Dateien übertragen mit SFTP.** Neben Befehlen braucht man oft Dateien: eine Konfiguration hochladen, ein Protokoll herunterholen. Dafür bringt SSH das Teilprotokoll SFTP mit, das `client.open_sftp()` bereitstellt. Mit `put(lokal, entfernt)` laden Sie eine Datei hoch, mit `get(entfernt, lokal)` laden Sie sie herunter; `stat()` liefert Größe und Zeitstempel, `listdir()` den Inhalt eines Ordners, daneben gibt es `remove()`, `mkdir()`, `rename()` und weitere. Wir übertragen eine Datei auf den „Server“ (der dasselbe Dateisystem hat wie wir, weshalb wir einen eigenen Arbeitsordner anlegen), prüfen sie und laden sie wieder herunter. Beachten Sie das Muster mit `try/finally`: Verbindungen sind Ressourcen, die man schließen muss, auch wenn dazwischen etwas schiefgeht.

```python
import tempfile
from pathlib import Path

import testssh
from sshlib import verbinde

umgebung = testssh.starte(["web01"])
adresse, port = umgebung.hosts["web01"]
arbeitsordner = Path(tempfile.mkdtemp())
(arbeitsordner / "bericht.txt").write_text("Bericht vom Montag\n", encoding="utf-8")

client = verbinde(adresse, port, umgebung.benutzer, umgebung.schluessel)
sftp = client.open_sftp()
try:
    sftp.put(str(arbeitsordner / "bericht.txt"), str(arbeitsordner / "kopie.txt"))
    info = sftp.stat(str(arbeitsordner / "kopie.txt"))
    print("Größe auf dem Server:", info.st_size, "Bytes")
    print("Ordnerinhalt         :", sorted(sftp.listdir(str(arbeitsordner))))
    sftp.get(str(arbeitsordner / "kopie.txt"), str(arbeitsordner / "zurueck.txt"))
    print("Inhalt identisch     :", (arbeitsordner / "zurueck.txt").read_text() == "Bericht vom Montag\n")
    try:
        sftp.stat(str(arbeitsordner / "gibt-es-nicht"))
    except FileNotFoundError as fehler:
        print("Fehlende Datei       :", type(fehler).__name__)
finally:
    sftp.close()
    client.close()
umgebung.stoppe()
```

```text
Größe auf dem Server: 19 Bytes
Ordnerinhalt         : ['bericht.txt', 'kopie.txt']
Inhalt identisch     : True
Fehlende Datei       : FileNotFoundError
```

Die Größe von 19 Bytes entspricht dem Text einschließlich Zeilenumbruch, der Ordnerinhalt zeigt beide Dateien, und die Rückübertragung liefert denselben Inhalt. Fehlende Dateien melden sich wie lokal mit `FileNotFoundError`. Wichtig: `put` überschreibt Zieldateien ohne Rückfrage, und bei einem Abbruch mittendrin bleibt unter Umständen eine halbe Datei zurück. Für Konfigurationsdateien lädt man deshalb zuerst unter einem Zwischennamen hoch und benennt dann um (`rename()` ist atomar, siehe Modul 9, 9.4).

**Fehlerarten.** Was kann beim Verbinden und Ausführen schiefgehen, und wie unterscheidet man es? Das ist eine der wichtigsten Fragen für jedes Überwachungsprogramm, denn „Server nicht erreichbar“, „Anmeldung abgelehnt“ und „Befehl hat Fehler gemeldet“ sind drei völlig verschiedene Situationen mit verschiedenen Ursachen und verschiedenen Reaktionen. Das folgende Beispiel erzeugt vier davon absichtlich: einen gesunden Server, einen Server, der unseren Schlüssel nicht kennt (`AuthenticationException`), einen Port, auf dem kein Dienst lauscht (`NoValidConnectionsError`), und einen Befehl, der länger dauert als erlaubt (Zeitüberschreitung). Die Funktion `versuche` ordnet jede Ausnahme einer verständlichen Meldung zu; die Reihenfolge der `except`-Zweige ist wichtig, weil die allgemeinere `SSHException` Elternklasse einiger der anderen ist und sonst alles abfangen würde.

```python
import socket

import paramiko

import testssh
from sshlib import ssh_ausfuehren, verbinde

umgebung = testssh.starte(["web01", "web02"])
umgebung.starte_host("db01", 3, autorisiert=False)       # dieser Server kennt unseren Schlüssel nicht


def versuche(beschreibung, aktion):
    try:
        ergebnis = aktion()
        text = f"ok ({ergebnis})"
    except paramiko.AuthenticationException:
        text = "Anmeldung abgelehnt (Schlüssel nicht erlaubt)"
    except paramiko.ssh_exception.NoValidConnectionsError:
        text = "nicht erreichbar (kein Dienst auf dem Port)"
    except (socket.timeout, TimeoutError):
        text = "Zeitüberschreitung"
    except paramiko.SSHException as fehler:
        text = f"SSH-Fehler: {type(fehler).__name__}"
    print(f"{beschreibung:28} -> {text}")


def an(name):
    adresse, port = umgebung.hosts[name]
    return verbinde(adresse, port, umgebung.benutzer, umgebung.schluessel)


versuche("gesunder Server", lambda: an("web01").close() or "verbunden")
versuche("Schlüssel nicht erlaubt", lambda: an("db01"))
versuche("Port ohne Dienst", lambda: verbinde("127.0.0.1", 1, umgebung.benutzer, umgebung.schluessel))

client = an("web01")
versuche("Befehl zu langsam", lambda: ssh_ausfuehren(client, "sleep 3; echo fertig", timeout=1))
client.close()
umgebung.stoppe()
```

```text
gesunder Server              -> ok (verbunden)
Schlüssel nicht erlaubt      -> Anmeldung abgelehnt (Schlüssel nicht erlaubt)
Port ohne Dienst             -> nicht erreichbar (kein Dienst auf dem Port)
Befehl zu langsam            -> Zeitüberschreitung
```

Die vier Zeilen lesen sich wie eine Checkliste für den Betrieb. Beachten Sie, dass die Zeitüberschreitung beim Warten auf die Ausgabe des Befehls entsteht, nicht beim Verbindungsaufbau; das `timeout` von `exec_command` gilt für das Lesen und Schreiben auf dem Kanal. Zu langsame Befehle sind in der Praxis häufiger als abgelehnte Anmeldungen.

**fabric: Befehle auf einem Server, bequem.** `fabric` nimmt Ihnen den Aufbau ab: Eine `Connection` enthält Host, Benutzer und Anmeldedaten (hier über `connect_kwargs`, die an paramiko durchgereicht werden), und `run("befehl")` führt den Befehl aus und liefert ein `Result`-Objekt mit `stdout`, `stderr`, `exited` (Exit-Code), `ok` und `failed`. Entscheidend ist ein Unterschied zu unserer eigenen Bibliothek: `fabric` wirft bei einem Exit-Code ungleich null standardmäßig die Ausnahme `UnexpectedExit`, es sei denn, man übergibt `warn=True`. Mit `hide=True` unterdrückt man die Ausgabe auf dem Bildschirm, die `fabric` sonst mitschreibt.

```python
from fabric import Connection
from invoke.exceptions import UnexpectedExit

import testssh

umgebung = testssh.starte(["web01"])
adresse, port = umgebung.hosts["web01"]
verbindung = Connection(adresse, user=umgebung.benutzer, port=port, connect_kwargs={
    "key_filename": str(umgebung.schluessel), "allow_agent": False, "look_for_keys": False, "timeout": 5})

r = verbindung.run("echo $HOSTALIAS", hide=True)
print("stdout :", repr(r.stdout), "| ok:", r.ok, "| exited:", r.exited)
print("Befehl :", r.command, "| Host:", r.connection.host)

try:
    verbindung.run("exit 7", hide=True)
except UnexpectedExit as fehler:
    print("Fehlschlag löst Ausnahme aus, Exit-Code:", fehler.result.exited)

r = verbindung.run("exit 7", hide=True, warn=True)
print("mit warn=True      :", r.exited, r.failed)
verbindung.close()
umgebung.stoppe()
```

```text
stdout : 'web01\n' | ok: True | exited: 0
Befehl : echo $HOSTALIAS | Host: 127.0.0.11
Fehlschlag löst Ausnahme aus, Exit-Code: 7
mit warn=True      : 7 True
```

Die erste Zeile zeigt das Ergebnisobjekt, die zweite den ausgeführten Befehl und den Host. Dann folgt der Vergleich: Ohne `warn=True` wird ein Fehlschlag zur Ausnahme, mit `warn=True` bekommen Sie ein normales Ergebnis, in dem `failed` wahr ist. Welche Variante richtig ist, hängt vom Zweck ab: Für ein Wartungsskript, das bei einem Fehler abbrechen soll, ist die Ausnahme genau richtig; für eine Abfrage, die den Exit-Code auswerten will, nimmt man `warn=True`. Bei der Anmeldung verwendet Fabric nach unserer Prüfung als Regel für unbekannte Hostschlüssel die `AutoAddPolicy` (es nimmt sie also an und merkt sie sich), was bequem ist, aber die Prüfung abschwächt; bei sicherheitsrelevanten Verbindungen sollte man die Schlüssel vorher bekannt machen (11.4).

**Typische Fehler und Stolperfallen.** Der erste Fehler ist, `AutoAddPolicy` dauerhaft im Produktivprogramm zu belassen, weil „es ja funktioniert“. Der zweite ist, den Exit-Code zu ignorieren: Ein Befehl, der mit Fehler endet, liefert Text, der unauffällig aussieht, und das Programm meldet Erfolg. Der dritte ist, keine Zeitgrenzen zu setzen. Der vierte ist, die Verbindung nicht zu schließen, sodass auf dem Server Sitzungen hängen bleiben. Das Gegenüber:

```python
import paramiko

# FALSCH: akzeptiert jeden Server, ohne Timeout, ohne Exit-Code, ohne Aufräumen
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect("server01", username="admin", password="geheim123")   # Passwort im Code!
stdin, stdout, stderr = client.exec_command("systemctl is-active webdienst")
print(stdout.read().decode())          # "inactive" wirkt wie eine normale Antwort

# RICHTIG: bekannte Hostschlüssel, Schlüsselanmeldung, Timeout, Exit-Code, Aufräumen
from sshlib import ssh_ausfuehren, verbinde
client = verbinde("server01", 22, "admin", "/home/admin/.ssh/id_ed25519", hostkeys="/home/admin/.ssh/known_hosts")
try:
    r = ssh_ausfuehren(client, "systemctl is-active webdienst", timeout=10)
    if not r.ok:
        print("Dienst läuft nicht:", r.stdout.strip())
finally:
    client.close()
```

Die falsche Variante hat mehrere Schwächen auf einmal: Der Server wird nicht geprüft, das Passwort steht im Quelltext (Modul 10, 10.4 zeigt, wohin so etwas gehört), und das Ergebnis von `systemctl is-active` (das bei einem gestoppten Dienst „inactive“ ausgibt und mit Code 3 endet) wird als harmlose Ausgabe behandelt. Die richtige Variante stützt sich auf Schlüssel und bekannte Hostschlüssel, wertet den Exit-Code aus und schließt die Verbindung auch im Fehlerfall.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentationsstand.** Zur Beschreibung der Klasse `SSHClient` wurde die Dokumentation von paramiko in Version 2.4 abgerufen (die neuere Seite war nicht erreichbar); die Prüfumgebung verwendet paramiko 5.1.0. Das Verhalten der Hostschlüssel-Regeln `RejectPolicy` (Vorgabe) und `AutoAddPolicy` und der Methoden `connect`, `exec_command` und `open_sftp` wurde mit der installierten Version selbst geprüft; die Beispiele laufen unverändert. Für Fabric wurden die Seite „Getting started“ (Version 3.2) und die API-Beschreibung der Gruppen abgerufen. Dass Fabric in der installierten Version unbekannte Schlüssel automatisch annimmt, wurde empirisch bestätigt. Die Testserver (`testssh`) wurden nur als Administrator mit einem lokalen `sshd` geprüft; ob `sshd` als normaler Benutzer ebenso startet, hängt von der Installation ab.

> **Merke**
>
> SSH verbindet verschlüsselt und prüft beide Seiten. paramiko: `SSHClient`, Regel für unbekannte Hostschlüssel (Vorgabe `RejectPolicy`), `connect` mit Schlüssel und Zeitgrenzen, `exec_command` liefert drei Ströme, Exit-Code über `recv_exit_status()`, SFTP für Dateien. Fehlerarten trennen: Anmeldung, Erreichbarkeit, Zeitüberschreitung, Befehlsfehler. fabric: `Connection.run` mit `hide` und `warn`, Ausnahme `UnexpectedExit` bei Fehlschlag. Verbindungen immer schließen, Befehlszeilen nie aus ungeprüften Eingaben bauen.

> **Praxisbeispiel**
>
> Wo ein Skript früher per Hand `ssh server "..."` tippte, steht heute ein paramiko- oder Fabric-Aufruf: Dienste neu starten, Protokolle holen, Konfigurationen verteilen, Werkzeuge wie `psutil` auf entfernten Servern ausführen. Eine Besonderheit bei Überwachungen: Das Skript sollte zwischen „der Server ist kaputt“ und „ich komme nicht an den Server heran“ unterscheiden, denn nur im ersten Fall ist der Dienst betroffen.

## 11.4 Mehrere Hosts, Schlüssel und known_hosts

Sobald mehr als ein Server im Spiel ist, wächst die Bedeutung zweier Fragen, die bei einem einzelnen Rechner noch nebensächlich wirken: Wie weise ich mich aus, ohne bei jedem Server ein Passwort einzutippen oder gar im Programm abzulegen? Und woher weiß ich, dass ich wirklich mit dem Server spreche, den ich meine? Beide Fragen beantwortet SSH mit Schlüsseln, jeweils in beide Richtungen: Der Benutzer beweist seine Identität mit einem Schlüsselpaar, der Server beweist seine mit einem Hostschlüssel. In dieser Lerneinheit lernen Sie beides kennen, dazu die Datei `known_hosts`, mit der sich Ihr Rechner die Server merkt, und die Datei `ssh_config`, mit der Sie viele Server bequem benennen.

**Schlüsselpaare: privat und öffentlich.** Ein SSH-Schlüssel ist ein Paar aus zwei mathematisch verbundenen Dateien. Der *private Schlüssel* bleibt geheim auf Ihrem Rechner und beweist Ihre Identität; der *öffentliche Schlüssel* darf jeder kennen und wird auf den Servern hinterlegt, auf die Sie zugreifen dürfen (in der Datei `\textasciitilde/.ssh/authorized_keys` des Zielbenutzers). Beim Anmelden stellt der Server eine Aufgabe, die nur der Besitzer des privaten Schlüssels lösen kann; der private Schlüssel selbst wird dabei nie übertragen. Das hat gegenüber Passwörtern große Vorteile: Es gibt nichts zu erraten, nichts abzuhören, und ein Server, der kompromittiert wird, verrät nur öffentliche Schlüssel, die für sich wertlos sind. Das Programm `ssh-keygen` erzeugt ein Schlüsselpaar. Als Schlüsselart ist `ed25519` heute die übliche Wahl: kurz, schnell und sicher; die Handbuchseite von `ssh-keygen` nennt Ed25519 als Standard der neueren Versionen (die Debian-Seite wurde abgerufen). Das folgende Beispiel erzeugt ein Schlüsselpaar in einem Wegwerfordner und untersucht das Ergebnis.

```python
import stat
import subprocess
import tempfile
from pathlib import Path

import paramiko

ordner = Path(tempfile.mkdtemp())
privat = ordner / "id_ed25519"

subprocess.run(["ssh-keygen", "-q", "-t", "ed25519", "-N", "", "-C", "lehrgang@beispiel", "-f", str(privat)], check=True)

print("Dateien              :", sorted(p.name for p in ordner.iterdir()))
print("Rechte privater Schl.:", stat.filemode(privat.stat().st_mode))
print("Rechte öffentlicher  :", stat.filemode(privat.with_suffix(".pub").stat().st_mode))

oeffentlich = privat.with_suffix(".pub").read_text().split()
print("Aufbau .pub          :", oeffentlich[0], oeffentlich[1][:12] + "...", oeffentlich[2])

ausgabe = subprocess.run(["ssh-keygen", "-l", "-f", str(privat.with_suffix(".pub"))], capture_output=True, text=True).stdout
print("Fingerabdruck beginnt mit SHA256:", ausgabe.split()[1].startswith("SHA256:"), "| Schlüsselart:", ausgabe.split()[-1])

schluessel = paramiko.Ed25519Key.from_private_key_file(str(privat))
print("Fingerabdruck (paramiko)  :", schluessel.fingerprint == ausgabe.split()[1])
print("Bits                      :", schluessel.get_bits())
```

```text
Dateien              : ['id_ed25519', 'id_ed25519.pub']
Rechte privater Schl.: -rw-------
Rechte öffentlicher  : -rw-r--r--
Aufbau .pub          : ssh-ed25519 AAAAC3NzaC1l... lehrgang@beispiel
Fingerabdruck beginnt mit SHA256: True | Schlüsselart: (ED25519)
Fingerabdruck (paramiko)  : True
Bits                      : 256
```

Es entstehen zwei Dateien. Der private Schlüssel hat die Rechte `-rw-------`: nur der Besitzer darf lesen und schreiben. Das ist keine Schönheit, sondern Pflicht, denn das Programm `ssh` weigert sich, einen Schlüssel zu benutzen, den andere Benutzer lesen können. Die öffentliche Datei besteht aus drei Teilen: der Schlüsselart, dem Schlüssel selbst (Base64-Text) und einem Kommentar, hier eine Bezeichnung, die später hilft zu erkennen, wem der Schlüssel gehört. Der *Fingerabdruck* (*fingerprint*) ist eine kurze Prüfsumme des Schlüssels; er beginnt bei neueren Versionen mit `SHA256:` und dient dazu, zwei Schlüssel bequem zu vergleichen, ohne den ganzen Text zu lesen. Paramiko berechnet denselben Fingerabdruck.

Beim Erzeugen haben wir die Passphrase mit `-N ""` leer gelassen, damit die Beispiele ohne Eingabe durchlaufen. In der Praxis sollte ein privater Schlüssel durch eine *Passphrase* geschützt sein, damit eine gestohlene Datei allein noch nicht reicht; damit man sie nicht ständig eintippen muss, gibt es den *SSH-Agenten* (`ssh-agent`), der den entsperrten Schlüssel für die Dauer der Sitzung im Speicher hält. Für unbeaufsichtigte Skripte (cron, systemd-Timer aus Modul 9) ist ein Schlüssel ohne Passphrase üblich, dann aber mit Einschränkungen auf dem Server, etwa mit der Option `command=` in `authorized_keys`, die diesem Schlüssel nur einen bestimmten Befehl erlaubt. Die genauen Optionen sind in der Handbuchseite von `sshd` beschrieben; sie wurde in der Prüfumgebung nicht abgerufen, dieser Hinweis ist also Fachwissen. Das Wichtigste: Ein privater Schlüssel gehört nie in ein Git-Repository, nie in ein Ticket, nie in eine Chatnachricht (Modul 10, 10.4).

**Hostschlüssel und known_hosts.** Die zweite Richtung der Prüfung: Jeder SSH-Server besitzt eigene Hostschlüssel, die bei der Installation erzeugt werden. Beim ersten Kontakt zeigt der Server dem Client seinen öffentlichen Hostschlüssel, und der Client merkt sich ihn in der Datei `\textasciitilde/.ssh/known_hosts` (je Zeile ein Server: Name oder Adresse, Schlüsselart, Schlüssel). Bei jedem weiteren Kontakt vergleicht der Client den gezeigten Schlüssel mit dem gemerkten. Stimmen sie überein, ist es derselbe Server wie damals; stimmen sie nicht überein, ist entweder der Server neu installiert worden oder jemand gibt sich als der Server aus (ein *Man-in-the-Middle-Angriff*), und SSH warnt unmissverständlich. Das schützt vor einer ganzen Angriffsklasse, aber nur, wenn man die Warnung nicht wegklickt. Die Schwachstelle liegt im ersten Kontakt: Beim allerersten Mal gibt es nichts zu vergleichen. Dieses Vorgehen heißt *Trust on first use* (Vertrauen beim ersten Mal). Professionell löst man das, indem man den Fingerabdruck des Servers aus einer zweiten Quelle bestätigt (Hosting-Oberfläche, Administrator) oder die Hostschlüssel vorab verteilt.

Die Datei lässt sich per Programm füllen. `ssh-keyscan` fragt einen Server nach seinen öffentlichen Hostschlüsseln und gibt Zeilen im richtigen Format aus. Zu beachten: Wenn ein Server auf einem anderen Port als 22 lauscht, schreibt `known_hosts` den Namen in der Form `[adresse]:port`. Das nächste Beispiel sammelt die Schlüssel zweier Server in einer eigenen Datei und liest sie mit `paramiko.HostKeys` wieder ein. Die Einträge zeigen Adresse und Schlüsselart; der Schlüssel selbst ist gekürzt. Beachten Sie, dass in der Praxis das Sammeln per `ssh-keyscan` dieselbe Schwäche hat wie das Vertrauen beim ersten Mal: Es holt, was der Server zeigt, ohne zu wissen, ob es der richtige ist. Es ersetzt die Bestätigung nicht, es automatisiert sie nur.

```python
import subprocess
import tempfile
from pathlib import Path

import paramiko

import testssh

umgebung = testssh.starte(["web01", "web02"])
known_hosts = Path(tempfile.mkdtemp()) / "known_hosts"

for name in ("web01", "web02"):
    adresse, port = umgebung.hosts[name]
    r = subprocess.run(["ssh-keyscan", "-p", str(port), "-t", "ed25519", adresse], capture_output=True, text=True)
    zeile = [z for z in r.stdout.splitlines() if not z.startswith("#")][0]
    with known_hosts.open("a") as datei:
        datei.write(zeile + "\n")

for zeile in known_hosts.read_text().splitlines():
    rechner, art, schluessel = zeile.split()
    print(rechner.rsplit(":", 1)[0], "|", art, "|", schluessel[:16] + "...")

hostkeys = paramiko.HostKeys(str(known_hosts))
adresse, port = umgebung.hosts["web01"]
eintrag = hostkeys.lookup(f"[{adresse}]:{port}")
print("Eintrag für web01 gefunden:", eintrag is not None, "| Schlüsselarten:", list(eintrag.keys()))
print("Für unbekannten Rechner   :", hostkeys.lookup("[127.0.0.99]:%d" % port))
umgebung.stoppe()
```

```text
[127.0.0.11] | ssh-ed25519 | AAAAC3NzaC1lZDI1...
[127.0.0.12] | ssh-ed25519 | AAAAC3NzaC1lZDI1...
Eintrag für web01 gefunden: True | Schlüsselarten: ['ssh-ed25519']
Für unbekannten Rechner   : None
```

Beide Server stehen mit ihrer Adresse in der Datei, der Schlüsseltext ist hier auf 16 Zeichen gekürzt; der Anfang ist bei allen Ed25519-Schlüsseln gleich, weil er die Schlüsselart kodiert, erst danach unterscheiden sich die Schlüssel. Die Suche nach einem Rechner, den es in der Datei nicht gibt, liefert `None`; damit kann ein Programm entscheiden, ob es einen Server bereits kennt. Unter Linux ist `known_hosts` oft mit Hashwerten statt lesbaren Namen angelegt (Option `HashKnownHosts`), damit eine gestohlene Datei nicht verrät, welche Server man besucht; nach der Handbuchseite von `ssh_config` (abgerufen) ist das keine Vorgabe der Software selbst, sondern hängt von der Konfiguration der Distribution ab.

**Streng prüfen im Programm.** Jetzt können wir das richtige Verhalten programmieren: Unbekannte Server werden *abgelehnt*; die Aufnahme eines Servers in `known_hosts` ist eine bewusste, getrennte Handlung; und wenn sich der Schlüssel eines bekannten Servers ändert, schlägt das Programm Alarm. Unsere Funktion `verbinde` aus 11.3 kann das schon: Sie lädt mit `load_host_keys` die Datei und setzt `RejectPolicy`. Das Beispiel spielt eine kleine Geschichte durch. Erst ist nichts bekannt, und die Verbindung scheitert; dann wird der Schlüssel von `web01` bewusst übernommen (die Funktion `merke_hostschluessel` entspricht dem Bestätigen beim ersten Kontakt) und die Verbindung gelingt; ein anderer Server bleibt weiter abgelehnt, und schließlich simulieren wir eine Neuinstallation von `web01` mit einem neuen Hostschlüssel, worauf die Abfrage warnt.

```python
import tempfile
from pathlib import Path

import paramiko

import testssh
from sshlib import ssh_ausfuehren, verbinde

umgebung = testssh.starte(["web01", "web02"])
known_hosts = Path(tempfile.mkdtemp()) / "known_hosts"


def merke_hostschluessel(name):
    """Holt den Hostschlüssel und trägt ihn in die Datei ein (die bewusste Erstvertrauens-Entscheidung)."""
    adresse, port = umgebung.hosts[name]
    transport = paramiko.Transport((adresse, port))
    transport.start_client(timeout=5)
    schluessel = transport.get_remote_server_key()
    transport.close()
    hostkeys = paramiko.HostKeys(str(known_hosts)) if known_hosts.exists() else paramiko.HostKeys()
    hostkeys.add(f"[{adresse}]:{port}", schluessel.get_name(), schluessel)
    hostkeys.save(str(known_hosts))
    return schluessel.fingerprint


def abfrage(name):
    adresse, port = umgebung.hosts[name]
    try:
        client = verbinde(adresse, port, umgebung.benutzer, umgebung.schluessel, hostkeys=known_hosts)
    except paramiko.BadHostKeyException:
        return "WARNUNG: Hostschlüssel hat sich geändert!"
    except paramiko.SSHException as fehler:
        return "abgelehnt: Hostschlüssel ist nicht in known_hosts"
    with client:
        return "ok: " + ssh_ausfuehren(client, "echo $HOSTALIAS").stdout.strip()


known_hosts.write_text("")
print("1. noch nichts bekannt  :", abfrage("web01"))
print("2. Fingerabdruck notiert:", merke_hostschluessel("web01").startswith("SHA256:"))
print("3. danach bekannt       :", abfrage("web01"))
print("4. anderer Server       :", abfrage("web02"))
umgebung.neuer_hostschluessel("web01")
print("5. nach Neuinstallation :", abfrage("web01"))
umgebung.stoppe()
```

```text
1. noch nichts bekannt  : abgelehnt: Hostschlüssel ist nicht in known_hosts
2. Fingerabdruck notiert: True
3. danach bekannt       : ok: web01
4. anderer Server       : abgelehnt: Hostschlüssel ist nicht in known_hosts
5. nach Neuinstallation : WARNUNG: Hostschlüssel hat sich geändert!
```

Besonders lehrreich ist Schritt 5. Der Server ist derselbe, hat dieselbe Adresse, nur der Schlüssel ist ein anderer. Ein Programm ohne geladene `known_hosts`-Datei, das `AutoAddPolicy` verwendet, hätte den neuen Schlüssel stillschweigend akzeptiert und wäre damit genau dem Angriff aufgesessen, den `known_hosts` verhindern soll (paramiko meldet einen geänderten Schlüssel nur, wenn es den alten kennt). Das Programm sollte in diesem Fall *nicht* automatisch weitermachen: Ein Mensch muss prüfen, ob eine Neuinstallation stattgefunden hat, und dann den alten Eintrag entfernen (`ssh-keygen -R adresse`) und den neuen bestätigen. Wo Sie viele Server verwalten, lohnt sich eine eigene `known_hosts`-Datei für das Projekt, die zusammen mit dem Skript (nicht aber mit den privaten Schlüsseln!) versioniert und geprüft wird.

**Die Datei ssh_config: Namen für Server.** Wer zehn Server verwaltet, will nicht jedes Mal Benutzer, Port, Schlüssel und Zwischenrechner angeben. Dafür gibt es die Datei `~/.ssh/config`, die dem Format `ssh_config` folgt. Jeder Abschnitt beginnt mit `Host` und einem oder mehreren Namensmustern, darunter stehen die Einstellungen, die für diese Namen gelten. Mit `HostName` legt man die echte Adresse fest, mit `User` den Benutzer, mit `Port` den Port, mit `IdentityFile` den Schlüssel, mit `ServerAliveInterval` den Abstand von Lebenszeichen, die verhindern, dass untätige Verbindungen von Firewalls getrennt werden, und mit `ProxyJump` einen Zwischenrechner (Sprungserver, *Bastion*), über den man ins interne Netz gelangt. Alle diese Schlüsselwörter sind in der Handbuchseite von `ssh_config(5)` beschrieben, die abgerufen wurde. Paramiko kann diese Datei lesen (`paramiko.SSHConfig`), sodass ein Programm dieselben Namen und Einstellungen verwendet wie das Terminal.

Die wichtigste Regel steht in dieser Handbuchseite und führt häufig zu Fehlern: Für jede Einstellung gilt der *erste* passende Wert. Spezielle Abschnitte müssen daher *vor* dem allgemeinen Abschnitt `Host *` stehen; wer es umgekehrt schreibt, überschreibt nichts, sondern bekommt stillschweigend die allgemeine Einstellung. Das Beispiel zeigt die richtige und die falsche Reihenfolge nebeneinander.

```python
import io
import os

import paramiko

RICHTIG = """
Host web01 web02
    HostName 192.0.2.11
    Port 2222

Host db01
    HostName db01.intern.example.org
    User dbadmin
    ProxyJump bastion

Host *
    User admin
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
"""

FALSCH = """
Host *
    User admin

Host db01
    User dbadmin
"""


def lies(text):
    konfig = paramiko.SSHConfig()
    konfig.parse(io.StringIO(text))
    return konfig


konfig = lies(RICHTIG)
for alias in ("web01", "db01", "unbekannt"):
    eintrag = konfig.lookup(alias)
    print(f"{alias:10} -> hostname={eintrag['hostname']}, port={eintrag.get('port', '22')}, user={eintrag['user']}, "
          f"proxyjump={eintrag.get('proxyjump')}")
print("Schlüsseldatei   :", [os.path.basename(p) for p in konfig.lookup("web01")["identityfile"]])
print("Alle Aliase      :", sorted(konfig.get_hostnames()))
print("Falsche Reihenfolge, db01 meldet sich als:", lies(FALSCH).lookup("db01")["user"])
```

```text
web01      -> hostname=192.0.2.11, port=2222, user=admin, proxyjump=None
db01       -> hostname=db01.intern.example.org, port=22, user=dbadmin, proxyjump=bastion
unbekannt  -> hostname=unbekannt, port=22, user=admin, proxyjump=None
Schlüsseldatei   : ['id_ed25519']
Alle Aliase      : ['*', 'db01', 'web01', 'web02']
Falsche Reihenfolge, db01 meldet sich als: admin
```

Bei `web01` gilt die Adresse und der Port aus dem speziellen Abschnitt, der Benutzer kommt aus dem Abschnitt `Host *`, weil der erste Abschnitt keinen nennt. `db01` hat einen eigenen Benutzer und den Sprungserver. `unbekannt` bekommt nur die allgemeinen Werte, und sein Host-Name bleibt der Aliasname. Die letzte Zeile zeigt den Stolperstein: In der falschen Reihenfolge steht `Host *` zuerst, sein Benutzer `admin` gewinnt, und `db01` meldet sich als `admin` statt als `dbadmin`.

**Typische Fehler und Stolperfallen.** Die Liste der Klassiker ist lang. Wer die Warnung bei geändertem Hostschlüssel mit `StrictHostKeyChecking=no` „löst“, schaltet den Schutz ab, statt die Ursache zu klären. Wer den privaten Schlüssel mit zu offenen Rechten ablegt (`chmod 644`), bekommt von `ssh` die Ablehnung *„UNPROTECTED PRIVATE KEY FILE“*. Wer den *öffentlichen* Schlüssel an den falschen Benutzer verteilt, sieht „Permission denied“, obwohl alles richtig aussieht. Wer `Host *` vor die Sonderfälle schreibt, wundert sich über falsche Benutzer. Und wer denselben Schlüssel für alles benutzt, vergrößert den Schaden, wenn dieser Schlüssel einmal in falsche Hände gerät. Das Gegenüber zu den beiden häufigsten Lösungen für den Hostschlüssel-Fehler:

```python
import paramiko

# FALSCH: "es geht ja nicht" -> Schutz abschalten
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect("server01", username="admin", key_filename="/home/admin/.ssh/id_ed25519")

# RICHTIG: Schlüssel bewusst bekannt machen, danach streng prüfen
#   einmalig im Terminal, Fingerabdruck mit dem Administrator abgleichen:
#       ssh-keyscan -t ed25519 server01 >> /home/admin/.ssh/known_hosts
#       ssh-keygen -lf /home/admin/.ssh/known_hosts
client = paramiko.SSHClient()
client.load_host_keys("/home/admin/.ssh/known_hosts")
client.set_missing_host_key_policy(paramiko.RejectPolicy())
client.connect("server01", username="admin", key_filename="/home/admin/.ssh/id_ed25519")
```

Die falsche Variante „funktioniert“, indem sie auf jede Prüfung verzichtet, und fällt auch nicht auf, wenn ein Angreifer den Server ersetzt. Die richtige hat einmalig mehr Aufwand: Der Schlüssel wird mit `ssh-keyscan` geholt, der Fingerabdruck wird über einen zweiten Weg mit dem Administrator verglichen, und danach lehnt das Programm alles ab, was nicht in der Datei steht.

> **Ergänzung aus der Nachrecherche**
>
> **Quellen und Grenzen.** Abgerufen wurden die Handbuchseiten `ssh-keygen(1)` (Debian trixie; Ed25519 als Standardschlüsselart) und `ssh_config(5)` (man7.org; `StrictHostKeyChecking`, `UserKnownHostsFile`, `HashKnownHosts` mit der Vorgabe „nein“, `IdentityFile`, `ServerAliveInterval`, `ProxyJump`). Das Format der Datei `known_hosts` wird in der Handbuchseite von `sshd(8)` erklärt, die nicht abrufbar war: Das hier gezeigte Format wurde mit den Beispielen selbst nachgeprüft (Name oder `[adresse]:port`, Schlüsselart, Schlüssel), die Angaben zu `authorized_keys`-Optionen sind Fachwissen. Die Aussage, dass Fabric unbekannte Hostschlüssel automatisch annimmt, wurde mit Fabric 3.2.3 selbst geprüft.

> **Merke**
>
> Schlüsselpaar: privat bleibt geheim (Rechte 600, Passphrase, Agent), öffentlich wandert nach `authorized_keys`. Hostschlüssel: Server beweist sich gegenüber dem Client, der Client merkt sich ihn in `known_hosts`; Änderung heißt Alarm, nicht Abschalten. Im Programm: Schlüssel bewusst einlesen, `RejectPolicy`. `ssh_config`: erster passender Wert gewinnt, Sonderfälle vor `Host *`. Eigene Schlüssel je Zweck, kein privater Schlüssel in Versionsverwaltung.

> **Praxisbeispiel**
>
> Ein neuer Server kommt ins Team: Sie erzeugen oder verteilen die öffentlichen Schlüssel, holen den Hostschlüssel, gleichen den Fingerabdruck mit dem Hoster ab und tragen den Server in die gemeinsame `known_hosts` und in die `ssh_config` ein. Ein Jahr später wird ein Server neu aufgesetzt, und Ihr Überwachungsprogramm meldet „Hostschlüssel geändert“: Das ist kein Fehler des Programms, sondern genau der Zweck der ganzen Einrichtung.

## 11.5 Mehrere Hosts abfragen und Quiz

Der Meilenstein dieses Moduls ist ein Programm, das mehrere Server gleichzeitig befragt und aus den Antworten einen Bericht macht. Es führt alles zusammen: die Messwerte aus 11.1 und 11.2 (hier lesen wir sie allerdings nicht mit `psutil`, sondern direkt aus `/proc` und mit `df`, weil `psutil` auf den Zielservern nicht installiert sein muss), die Verbindungen und Fehlerbehandlung aus 11.3, die Hostschlüssel-Strenge aus 11.4 und die Bausteine aus früheren Modulen: `argparse` für die Kommandozeile (Modul 5), `logging` und Exit-Codes (Modul 6), JSON für das Inventar (Modul 7) und die Struktur testbarer Programme (Modul 9, 9.5).

**Anforderungen.** Das Programm `hostcheck` liest ein Inventar im JSON-Format, das für jeden Server Adresse, Port und Benutzer nennt. Es fragt alle Server ab, und zwar *gleichzeitig*, denn eine Abfrage nach der anderen dauert bei zwanzig Servern mit je einer Sekunde Antwortzeit zwanzig Sekunden, bei gleichzeitigem Vorgehen nur wenig mehr als eine. Für jeden Server holt es drei Werte: die Last der letzten Minute, die Belegung des Wurzel-Dateisystems in Prozent und die Speicherbelegung in Prozent. Es gibt eine Tabelle oder mit `--json` JSON aus. Entscheidend ist die Fehlerbehandlung: Fällt ein Server aus, antwortet er nicht oder hat er einen geänderten Hostschlüssel, steht bei ihm eine verständliche Fehlermeldung, aber die anderen Server erscheinen weiterhin im Ergebnis. Der Exit-Code verrät den Gesamtzustand: 0 für alles in Ordnung, 2 für Warnungen (Schwellenwerte überschritten), 1 für Fehler bei mindestens einem Server.

**Der Entwurf.** Das Programm besteht aus kleinen, einzeln prüfbaren Funktionen. `parse_ausgabe` zerlegt den Text, den der Befehl auf dem Server liefert, in Zahlen. Es ist eine reine Funktion ohne Netzwerk und lässt sich deshalb mit festen Beispieltexten testen. `frage_host` baut eine Verbindung auf, führt den Befehl aus und übersetzt *jede* mögliche Ausnahme in ein `HostErgebnis` mit Fehlertext; sie wirft nie eine Ausnahme nach außen, denn das würde bei der gleichzeitigen Ausführung die anderen Server mitreißen. `frage_alle` verteilt die Server mit einem `ThreadPoolExecutor` (Modul 8) auf mehrere Arbeiter. Hier sind Threads richtig, weil die Arbeit fast nur aus Warten auf das Netzwerk besteht. Die Methode `pool.map` liefert die Ergebnisse in der Reihenfolge des Inventars, nicht in der Reihenfolge des Eintreffens, wodurch der Bericht stabil bleibt. `main` verbindet alles, nimmt die Argumente als Parameter (wie in 9.5) und liefert den Exit-Code als Rückgabewert.

Der Befehl, den wir auf den Servern ausführen, bündelt alle drei Abfragen in einem einzigen Aufruf: `cat /proc/loadavg` liefert die Last, `df -P /` die Belegung des Wurzel-Dateisystems (mit `-P` im festen, standardisierten Ausgabeformat), und `grep` holt zwei Zeilen aus `/proc/meminfo`. Ein einziger Befehl spart Verbindungen und Zeit.

```python
#!/usr/bin/env python3
"""hostcheck - fragt mehrere Server per SSH ab (Last, Platte, Speicher)."""
import argparse
import json
import logging
import socket
import sys
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from pathlib import Path

import paramiko

from sshlib import ssh_ausfuehren, verbinde

log = logging.getLogger("hostcheck")
EXIT_OK, EXIT_FEHLER, EXIT_WARNUNG = 0, 1, 2
BEFEHL = "cat /proc/loadavg; df -P / | tail -n 1; grep -E '^(MemTotal|MemAvailable):' /proc/meminfo"


@dataclass
class HostErgebnis:
    name: str
    fehler: str = ""
    werte: dict = field(default_factory=dict)

    @property
    def ok(self):
        return not self.fehler


def parse_ausgabe(text):
    """Zerlegt die Ausgabe von BEFEHL in Last, Plattenbelegung und RAM-Belegung (in Prozent)."""
    zeilen = text.strip().splitlines()
    last1 = float(zeilen[0].split()[0])
    platte = int(zeilen[1].split()[4].rstrip("%"))
    meminfo = {z.split(":")[0]: int(z.split(":")[1].split()[0]) for z in zeilen[2:]}
    ram = round(100 * (meminfo["MemTotal"] - meminfo["MemAvailable"]) / meminfo["MemTotal"])
    return {"last": last1, "platte": platte, "ram": ram}


def frage_host(name, eintrag, schluessel, known_hosts, timeout):
    try:
        client = verbinde(eintrag["adresse"], eintrag.get("port", 22), eintrag.get("benutzer", "admin"),
                          schluessel, timeout=timeout, hostkeys=known_hosts)
    except paramiko.BadHostKeyException:
        return HostErgebnis(name, "Hostschlüssel hat sich geändert")
    except paramiko.AuthenticationException:
        return HostErgebnis(name, "Anmeldung abgelehnt")
    except paramiko.ssh_exception.NoValidConnectionsError:
        return HostErgebnis(name, "nicht erreichbar")
    except (socket.timeout, TimeoutError):
        return HostErgebnis(name, "Zeitüberschreitung beim Verbinden")
    except paramiko.SSHException as fehler:
        return HostErgebnis(name, f"SSH-Fehler ({type(fehler).__name__})")
    with client:
        try:
            r = ssh_ausfuehren(client, BEFEHL, timeout=timeout)
        except (socket.timeout, TimeoutError):
            return HostErgebnis(name, "Befehl zu langsam")
        if not r.ok:
            return HostErgebnis(name, f"Befehl endete mit Code {r.exitcode}")
        try:
            return HostErgebnis(name, werte=parse_ausgabe(r.stdout))
        except (IndexError, KeyError, ValueError):
            return HostErgebnis(name, "Ausgabe nicht lesbar")


def frage_alle(inventar, schluessel, known_hosts, timeout=5, parallel=4):
    namen = list(inventar)
    with ThreadPoolExecutor(max_workers=parallel) as pool:
        ergebnisse = pool.map(lambda n: frage_host(n, inventar[n], schluessel, known_hosts, timeout), namen)
        return list(ergebnisse)


def main(argv=None):
    parser = argparse.ArgumentParser(description="Mehrere Hosts per SSH abfragen")
    parser.add_argument("inventar", type=Path, help="JSON-Datei mit den Hosts")
    parser.add_argument("--schluessel", type=Path, required=True)
    parser.add_argument("--known-hosts", type=Path, required=True)
    parser.add_argument("--timeout", type=float, default=5)
    parser.add_argument("--parallel", type=int, default=4)
    parser.add_argument("--platte-warnung", type=int, default=90)
    parser.add_argument("--ram-warnung", type=int, default=90)
    parser.add_argument("--json", action="store_true")
    args = parser.parse_args(argv)
    logging.basicConfig(level=logging.WARNING, format="%(levelname)s %(message)s")

    inventar = json.loads(args.inventar.read_text(encoding="utf-8"))["hosts"]
    ergebnisse = frage_alle(inventar, args.schluessel, args.known_hosts, args.timeout, args.parallel)

    warnungen = []
    for e in ergebnisse:
        if e.ok and e.werte["platte"] >= args.platte_warnung:
            warnungen.append(f"{e.name}: Platte zu {e.werte['platte']} % belegt")
        if e.ok and e.werte["ram"] >= args.ram_warnung:
            warnungen.append(f"{e.name}: RAM zu {e.werte['ram']} % belegt")

    if args.json:
        print(json.dumps([{"name": e.name, "ok": e.ok, "fehler": e.fehler, **e.werte} for e in ergebnisse], indent=2))
    else:
        for e in ergebnisse:
            if e.ok:
                v = e.werte
                print(f"{e.name:8} ok       Last {v['last']:.2f}, Platte {v['platte']} %, RAM {v['ram']} %")
            else:
                print(f"{e.name:8} FEHLER   {e.fehler}")
    for w in warnungen:
        log.warning(w)
    if any(not e.ok for e in ergebnisse):
        return EXIT_FEHLER
    return EXIT_WARNUNG if warnungen else EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Gehen wir die Teile durch. Die Konstanten für die Exit-Codes stehen am Anfang, damit man sie nicht in Zahlen versteckt. Das `HostErgebnis` hat entweder Werte oder einen Fehlertext, und die Eigenschaft `ok` fasst das zusammen. In `parse_ausgabe` ist die Plattenbelegung die fünfte Spalte der letzten Zeile von `df -P` (also Dateisystem, Größe, belegt, frei, Prozent, Einhängepunkt); die Speicherbelegung wird wie in 11.1 aus `MemTotal` und `MemAvailable` berechnet. In `frage_host` stehen die `except`-Zweige in der Reihenfolge von spezifisch nach allgemein: Der geänderte Hostschlüssel (`BadHostKeyException`) kommt zuerst, weil er die schwerwiegendste Meldung ist und zugleich eine Unterklasse von `SSHException`; die allgemeine `SSHException` steht zuletzt. Das `with client:` schließt die Verbindung in jedem Fall. Die Auswertung des Textes ist ebenfalls abgesichert: Liefert ein Server eine Ausgabe, die wir nicht lesen können (andere Betriebssystemvariante, abweichendes Format), wird das als Fehler gemeldet, nicht als Absturz. In `main` werden zuerst alle Server befragt, dann die Warnungen berechnet (nur für Server, die geantwortet haben, denn für die anderen gibt es nichts zu bewerten), dann ausgegeben. Die Warnungen gehen mit `logging` nach `stderr`, die Daten nach `stdout`, wie wir es seit Modul 5 halten.

Zuerst die reine Funktion. Das Beispiel prüft `parse_ausgabe` mit einem festen Beispieltext und mit drei kaputten Ausgaben und zeigt, welche Ausnahmen entstehen, denn `frage_host` muss genau diese drei Arten abfangen. Das ist der große Vorteil der Trennung von Netzwerk und Auswertung: Der Test läuft in Millisekunden und braucht keinen Server.

```python
from hostcheck import parse_ausgabe

beispiel = """0.52 0.40 0.35 2/412 9876
overlay 61255492 17647044 43592064 29% /
MemTotal:        8219876 kB
MemAvailable:    7570000 kB
"""
print(parse_ausgabe(beispiel))

for kaputt in ("", "0.52 0.40\n", "x y z\nA B C D E%\nMemTotal: 1 kB\nMemAvailable: 1 kB\n"):
    try:
        parse_ausgabe(kaputt)
    except (IndexError, KeyError, ValueError) as fehler:
        print(type(fehler).__name__, "bei", repr(kaputt[:20]))
```

```text
{'last': 0.52, 'platte': 29, 'ram': 8}
IndexError bei ''
IndexError bei '0.52 0.40\n'
ValueError bei 'x y z\nA B C D E%\nMem'
```

Die erste Zeile zeigt die Auswertung des gültigen Textes: eine Last von 0,52, eine Platte zu 29 Prozent und einen Speicher zu etwa 8 Prozent belegt. Die drei kaputten Eingaben erzeugen die drei erwarteten Ausnahmen: Ein leerer Text enthält nicht einmal eine erste Zeile (`IndexError`), ein zu kurzer Text lässt die zweite Zeile mit der Plattenbelegung vermissen (ebenfalls `IndexError`), und Zeichen statt Zahlen lassen die Umwandlung scheitern (`ValueError`).

Nun die gesamte Kette. Das Testprogramm startet vier lokale Server und trägt einen fünften ein, den es gar nicht gibt, und zeigt, dass das Programm mit allem zurechtkommt: `web01` und `web02` sind in Ordnung; `app01` wurde „neu installiert“ und hat einen anderen Hostschlüssel als in der Datei `known_hosts`; `web03` kennt unseren Schlüssel nicht; `db02` steht im Inventar, aber auf dem angegebenen Port lauscht niemand. Da alle Server in Wirklichkeit auf demselben Rechner laufen, sind die Messwerte überall gleich und von Lauf zu Lauf verschieden, weshalb das Beispiel die Zahlen vor der Ausgabe durch ein `N` ersetzt und die Struktur prüft.

```python
import io
import json
import re
import tempfile
from contextlib import redirect_stdout
from pathlib import Path

import paramiko

import testssh
from hostcheck import main

umgebung = testssh.starte(["web01", "web02", "app01"])
umgebung.starte_host("web03", 4, autorisiert=False)       # kennt unseren Schlüssel nicht
arbeit = Path(tempfile.mkdtemp())
known_hosts = arbeit / "known_hosts"

# Hostschlüssel der Server einmalig übernehmen (hier ohne Rückfrage, in der Praxis nach Prüfung des Fingerabdrucks)
hostkeys = paramiko.HostKeys()
for name, (adresse, port) in umgebung.hosts.items():
    transport = paramiko.Transport((adresse, port))
    transport.start_client(timeout=5)
    schluessel = transport.get_remote_server_key()
    transport.close()
    hostkeys.add(f"[{adresse}]:{port}", schluessel.get_name(), schluessel)
hostkeys.save(str(known_hosts))

inventar = {"hosts": {name: {"adresse": adresse, "port": port, "benutzer": umgebung.benutzer}
                      for name, (adresse, port) in umgebung.hosts.items()}}
inventar["hosts"]["db02"] = {"adresse": "127.0.0.1", "port": 1, "benutzer": umgebung.benutzer}   # niemand lauscht
umgebung.neuer_hostschluessel("app01")                    # app01 wurde neu installiert
datei = arbeit / "hosts.json"
datei.write_text(json.dumps(inventar), encoding="utf-8")

grund = [str(datei), "--schluessel", str(umgebung.schluessel), "--known-hosts", str(known_hosts), "--timeout", "3"]


def lauf(zusatz=()):
    puffer = io.StringIO()
    with redirect_stdout(puffer):
        code = main(grund + list(zusatz))
    return code, puffer.getvalue()


code, text = lauf()
print("Exit-Code:", code)
print(re.sub(r"(Last |Platte |RAM )\d+(\.\d+)?", r"\1N", text), end="")

code, text = lauf(["--json"])
daten = json.loads(text)
print("JSON, Hosts mit Fehler:", [d["name"] for d in daten if not d["ok"]])
print("web01 hat Last-Wert:", isinstance(daten[0]["last"], float))
umgebung.stoppe()
```

```text
Exit-Code: 1
web01    ok       Last N, Platte N %, RAM N %
web02    ok       Last N, Platte N %, RAM N %
app01    FEHLER   Hostschlüssel hat sich geändert
web03    FEHLER   Anmeldung abgelehnt
db02     FEHLER   nicht erreichbar
JSON, Hosts mit Fehler: ['app01', 'web03', 'db02']
web01 hat Last-Wert: True
```

Das Ergebnis zeigt genau die Struktur, auf die es ankommt: Alle fünf Server erscheinen, jeder mit seinem eigenen Zustand. Die beiden gesunden zeigen Werte; die anderen drei zeigen verständliche Fehler, und zwar einer pro Ursache: geänderter Hostschlüssel, abgelehnte Anmeldung, nicht erreichbar. Der Exit-Code 1 meldet, dass mindestens ein Server nicht abgefragt werden konnte. Die JSON-Variante enthält dieselben Informationen in maschinenlesbarer Form; sie ist die Schnittstelle für Modul 12, wo ein Alarm per Mail oder Webhook daraus entsteht. Bemerkenswert ist, was *nicht* passiert: Kein Server bringt die anderen zu Fall, und kein Fehler wird verschluckt.

**Grenzen dieser Lösung.** Das Programm ist ein Lehrbeispiel und bewusst einfach gehalten. Es nutzt eine feste Anzahl von vier Arbeitern (`--parallel`), was bei mehreren hundert Servern angepasst werden müsste. Es misst die Last als Momentaufnahme; für eine zuverlässige Überwachung benötigt man einen Verlauf (siehe das Merke-Kästchen in 11.1). Es erwartet Linux auf den Zielservern, weil es `/proc` liest. Und es fragt jeden Server einmal ohne Wiederholung: Ein einzelner Netzwerkaussetzer führt sofort zu einem Fehler. Für den Betrieb würde man bei Zeitüberschreitungen einen zweiten Versuch einbauen (Modul 10, 10.3). Alternativ lässt sich die Gruppenabfrage auch mit `fabric` lösen, was Aufgabe 8 der Übungen zeigt.

**Typische Fehler und Stolperfallen.** Ein häufiger Fehler bei der gleichzeitigen Abfrage ist, Ausnahmen aus den Arbeitern nicht abzufangen. Bei `pool.map` wird eine Ausnahme erst beim Einsammeln der Ergebnisse ausgelöst und beendet dann die ganze Schleife; die Ergebnisse der anderen Server sind verloren. Ein zweiter Fehler ist, die Reihenfolge der Ausgabe von der Antwortgeschwindigkeit abhängen zu lassen, sodass Berichte nicht vergleichbar sind. Ein dritter, die Zahl der Arbeiter unbegrenzt zu lassen: Hundert gleichzeitige SSH-Verbindungen können den Rechner oder das Netz überfordern. Das Gegenüber:

```python
from concurrent.futures import ThreadPoolExecutor

# FALSCH: eine Ausnahme in einem Arbeiter bricht alles ab, kein Timeout, keine Obergrenze
def frage(adresse):
    client = verbinde(adresse, 22, "admin", SCHLUESSEL)      # wirft bei Fehler
    return ssh_ausfuehren(client, "uptime").stdout

with ThreadPoolExecutor() as pool:
    ergebnisse = list(pool.map(frage, ADRESSEN))             # erster Fehler: Ausnahme, Rest verloren

# RICHTIG: jeder Arbeiter liefert immer ein Ergebnis, auch im Fehlerfall
def frage_sicher(adresse):
    try:
        client = verbinde(adresse, 22, "admin", SCHLUESSEL, timeout=5, hostkeys=KNOWN_HOSTS)
    except Exception as fehler:                              # hier bewusst breit: nichts darf entweichen
        return adresse, False, f"{type(fehler).__name__}: {fehler}"
    with client:
        return adresse, True, ssh_ausfuehren(client, "uptime", timeout=5).stdout.strip()

with ThreadPoolExecutor(max_workers=8) as pool:
    ergebnisse = list(pool.map(frage_sicher, ADRESSEN))
```

Die falsche Variante verliert beim ersten Fehler die Ergebnisse aller anderen Server; die richtige gibt jedem Server ein Ergebnis. Das `except Exception` ist hier ausnahmsweise berechtigt, weil dies die oberste Ebene eines Arbeiters ist, und der Fehler wird nicht verschluckt, sondern mit Typ und Meldung ins Ergebnis übernommen. Im fertigen Programm oben sind die Ausnahmen genauer aufgeteilt, damit die Meldungen verständlicher sind.

**Quiz zum Modul.** Zum Abschluss einige Fragen, die Sie im Kopf beantworten können; die Auflösungen stehen in den Lösungen am Modulende (Aufgaben 1 und 2). Erstens: Welche Zahl liefert `psutil.cpu_percent()` beim allerersten Aufruf, und warum? Zweitens: Warum darf ein Programm, das Speicherknappheit melden soll, nicht auf `free` schauen? Drittens: Was ist der Unterschied zwischen `NoSuchProcess` und `AccessDenied`? Viertens: Was geschieht bei `AutoAddPolicy`, wenn ein Server einen neuen Hostschlüssel hat? Fünftens: Welche Reihenfolge gilt in der Datei `ssh_config`? Sechstens: Warum meldet die gleichzeitige Abfrage keine Ausnahmen, sondern Ergebnisobjekte?

> **Ergänzung aus der Nachrecherche**
>
> **Was in dieser Lerneinheit nicht geprüft werden konnte.** Alle „Server“ der Beispiele laufen als lokale `sshd`-Prozesse auf einem Rechner; Netzwerkfehler wie verlorene Pakete, langsame Leitungen oder Firewalls, die Verbindungen stillschweigend verwerfen, wurden nicht nachgestellt. Die Zeitüberschreitung beim Verbinden (nicht erreichbarer Rechner) lässt sich lokal nur durch eine nicht erreichbare Adresse simulieren; die Beispiele nutzen stattdessen einen Port ohne Dienst. Wie sich die Software gegenüber verschiedenen `sshd`-Versionen und Konfigurationen verhält (etwa Server, die nur ältere Schlüsselarten erlauben), hängt vom Zielsystem ab.

> **Merke**
>
> Mehrere Hosts: Arbeit nebenläufig verteilen (Threads, weil Warten dominiert), *jedem Host ein Ergebnis* geben, auch im Fehlerfall, Reihenfolge stabil halten, Zahl der Arbeiter begrenzen, Zeitgrenzen setzen. Reine Auswertungsfunktionen vom Netzwerk trennen und mit festen Texten testen. Exit-Code und JSON sind die Schnittstelle zum nächsten Schritt (Alarmierung, Modul 12).

> **Praxisbeispiel**
>
> Genau so beginnt jede kleine Überwachung ohne teure Werkzeuge: eine Liste von Servern, ein Befehl, ein Bericht, ein Exit-Code. Größere Umgebungen setzen dafür Werkzeuge wie Monitoring-Systeme ein, doch das Muster bleibt: einsammeln, bewerten, melden. Auch für Wartungsarbeiten (auf allen Servern ein Update einspielen, einen Dienst neu starten) gilt dieselbe Regel, dass ein ausgefallener Server nicht das ganze Vorhaben verhindern darf.

## Übungsaufgaben zu Modul 11

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Für Aufgabe 5 und 6 brauchen Sie nur `psutil`; für Aufgabe 8 das Hilfsmodul `testssh` aus 11.3 und `fabric`.

**Aufgabe 1.** Beantworten Sie die Quizfragen zu `psutil`: Welche Zahl liefert `psutil.cpu_percent()` beim allerersten Aufruf ohne Intervall, und warum? Warum sollte ein Programm, das Speicherknappheit melden soll, `available` oder `percent` statt `free` auswerten? Worin unterscheiden sich `NoSuchProcess` und `AccessDenied`, und wie reagiert eine robuste Auswertung aller Prozesse darauf? Warum zeigt ein Prozess mit 200 % CPU-Nutzung kein Problem der Messung an?

**Aufgabe 2.** Beantworten Sie die Quizfragen zu SSH: Was geschieht bei `AutoAddPolicy`, wenn ein bekannter Server plötzlich einen anderen Hostschlüssel zeigt, und was bei `RejectPolicy` mit geladener `known_hosts`? Welche Reihenfolge gilt in der Datei `ssh_config`, wenn ein Abschnitt `Host *` und ein Abschnitt für einen einzelnen Server dieselbe Einstellung enthalten? Warum meldet die gleichzeitige Abfrage mehrerer Server Ergebnisobjekte und keine Ausnahmen? Worin unterscheiden sich ein privater und ein öffentlicher Schlüssel, und welcher gehört wohin?

**Aufgabe 3.** Finden Sie mindestens sechs Fehler oder Sicherheitsprobleme im folgenden Programm und erklären Sie jeden.

```python
import paramiko
import psutil

def pruefe(server):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(server, username="root", password="Sommer2026!")
    stdin, stdout, stderr = client.exec_command("df -h / | tail -1")
    zeile = stdout.read().decode()
    prozent = int(zeile.split()[4].strip("%"))
    if psutil.virtual_memory().free < 100000000:
        print("Speicher knapp auf", server)
    if psutil.cpu_percent() > 90:
        print("CPU ueberlastet")
    return prozent

for s in ["web01", "web02", "db01"]:
    print(pruefe(s))
```

**Aufgabe 4.** Ein Kollege berichtet: „Seit dem Wochenende meldet unser Überwachungsprogramm für `app01` eine Warnung wegen eines geänderten Hostschlüssels. Ich habe `StrictHostKeyChecking=no` gesetzt, jetzt geht es wieder.“ Beurteilen Sie das Vorgehen. Welche Ursachen kommen für die Warnung infrage, wie stellen Sie fest, welche zutrifft, und wie gehen Sie weiter vor? Beschreiben Sie außerdem, wie Sie für ein Skript, das nachts ohne Aufsicht läuft, einen eigenen Schlüssel einrichten und einschränken.

**Aufgabe 5.** **Code:** Schreiben Sie eine Funktion `top_prozesse(anzahl, nach)`, die die `anzahl` Prozesse mit dem größten Speicherverbrauch (`nach="speicher"`) oder der größten bisherigen Rechenzeit (`nach="cpu"`) als Liste von Wörterbüchern mit `pid`, `name`, `rss` und `cpu` liefert. Prozesse, die verschwinden oder gesperrt sind, werden übersprungen. Prüfen Sie die Funktion, indem Ihr Programm absichtlich 300 Mebibyte belegt und sich dann selbst in der Liste sucht.

**Aufgabe 6.** **Code:** Schreiben Sie eine Funktion `parse_df(text)`, die die Ausgabe von `df -P` in eine Liste von Wörterbüchern verwandelt (Dateisystem, Größe, belegt, frei, Prozent als Zahl, Einhängepunkt). Beachten Sie, dass ein Einhängepunkt Leerzeichen enthalten kann, und lehnen Sie unlesbare Zeilen mit einer verständlichen Fehlermeldung ab.

**Aufgabe 7.** **Code:** Schreiben Sie eine Funktion `ssh_optionen(alias, konfigtext)`, die mit `paramiko.SSHConfig` den Eintrag eines Servers liest und daraus die Argumente für `SSHClient.connect()` macht (Rechner, Port, Benutzer, Schlüsseldateien, Zeitlimit). Fehlende Angaben haben sinnvolle Vorgaben.

**Aufgabe 8.** **Code:** Schreiben Sie mit `fabric` eine Funktion `auf_allen(gruppe, befehl)`, die einen Befehl auf allen Servern einer Gruppe ausführt und je Server ein Paar aus Erfolg und Text liefert. Fehlgeschlagene Server dürfen die anderen nicht beeinflussen. Testen Sie sie mit einer `SerialGroup` und einer `ThreadingGroup` gegen vier lokale Testserver, von denen einer den Schlüssel nicht kennt.

## Lösungen zu Modul 11

**Aufgabe 1.** Der erste Aufruf von `cpu_percent()` ohne Intervall liefert `0.0`, weil `psutil` die Auslastung aus dem Vergleich zweier Messungen berechnet und beim ersten Aufruf keine frühere Messung existiert. Abhilfe: den ersten Wert wegwerfen oder mit `interval=1` messen. `free` ist der Speicher, der wirklich völlig unbenutzt ist; Linux nutzt freien Speicher aber als Zwischenspeicher für Dateien, so dass `free` auf einem gesunden Rechner klein ist. `available` berücksichtigt, was sich bei Bedarf sofort freigeben lässt, und ist die ehrliche Größe für „wie viel können neue Programme bekommen“; `percent` ist daraus abgeleitet. `NoSuchProcess` bedeutet, dass der Prozess nicht (mehr) existiert, `AccessDenied`, dass es ihn gibt, man ihn aber nicht lesen darf. Beide, und `ZombieProcess`, fängt eine robuste Auswertung ab und überspringt den Prozess, weil sie bei der Arbeit mit Prozessen normal sind, nicht Fehler des Programms. Die Prozentangabe eines Prozesses bezieht sich auf einen Kern; ein Prozess, der zwei Kerne voll nutzt, zeigt 200 %, das ist kein Messfehler, nur ein anderer Bezugswert als die Gesamtauslastung (die bei 100 % aller Kerne endet).

**Aufgabe 2.** Bei `AutoAddPolicy` wird der neue Schlüssel eines *unbekannten* Servers aufgenommen. Ist der Server aber bereits in der geladenen `known_hosts` verzeichnet und zeigt einen anderen Schlüssel, löst paramiko die Ausnahme `BadHostKeyException` aus, die Regel für unbekannte Server greift dann gar nicht; kennt der Client den Server hingegen nicht (etwa weil keine Datei geladen wurde), nimmt `AutoAddPolicy` jeden Schlüssel an und schützt vor nichts. Mit `RejectPolicy` und geladener Datei lehnt der Client alles ab, was nicht in der Datei steht, und meldet einen geänderten Schlüssel. In der `ssh_config` gilt für jede Einstellung der *erste* passende Wert beim Lesen von oben nach unten; spezielle Abschnitte gehören deshalb vor `Host *`. Die gleichzeitige Abfrage liefert Ergebnisobjekte, weil eine Ausnahme in einem Arbeiter beim Einsammeln die Schleife beendet und die Ergebnisse der anderen Server verloren gingen; jeder Server soll ein Ergebnis bekommen, im Fehlerfall mit Fehlertext. Der *private* Schlüssel beweist die Identität des Benutzers und bleibt geheim auf dem Rechner des Benutzers (Rechte 600, möglichst mit Passphrase); der *öffentliche* wird auf den Zielservern in `authorized_keys` des Zielbenutzers hinterlegt und darf bekannt sein.

**Aufgabe 3.** (1) `AutoAddPolicy` nimmt jeden Server an: Ein Angreifer kann sich als Server ausgeben; richtig ist `RejectPolicy` mit einer geladenen `known_hosts`. (2) Das Passwort steht im Quelltext, und es ist das Passwort von `root`; richtig ist Schlüsselanmeldung mit einem eingeschränkten Benutzer, Geheimnisse aus der Umgebung oder einer Datei (Modul 10, 10.4). (3) Die Anmeldung als `root` gibt dem Skript weit mehr Rechte, als es für das Lesen von `df` braucht. (4) Es gibt weder ein Zeitlimit bei `connect` noch bei `exec_command`: Ein hängender Server hält das ganze Programm fest. (5) Der Exit-Code wird nicht abgefragt, und `stderr` nicht gelesen; ein Fehlschlag des Befehls sieht aus wie leere Ausgabe und führt zu einem `IndexError`. (6) Die Verbindung wird nie geschlossen. (7) Die Auswertung der Ausgabe nimmt eine feste Form an (`split()[4]`), ohne `df -P`: Je nach Version bricht `df -h` bei langen Gerätenamen die Zeile um, und die Spalte stimmt nicht mehr. (8) Die `psutil`-Prüfungen untersuchen den *lokalen* Rechner, obwohl die Meldung `server` nennt: Sie sagen nichts über den entfernten Server. (9) `free` statt `available` und (10) der erste `cpu_percent()`-Aufruf ohne Intervall (liefert 0.0): beide Prüfungen sind wirkungslos beziehungsweise falsch. (11) Die Schleife bricht beim ersten Fehler eines Servers ab, die übrigen Server werden nicht geprüft.

**Aufgabe 4.** Das Vorgehen ist abzulehnen: `StrictHostKeyChecking=no` schaltet genau die Prüfung ab, die vor einem untergeschobenen Server schützen soll, und blendet damit eine mögliche Ursache aus, die ernst sein kann. Ursachen: (a) der Server wurde neu installiert oder der Hostschlüssel erneuert (harmlos, aber bestätigungspflichtig), (b) der Name oder die Adresse gehört jetzt einem anderen Rechner (Umzug, neue Adresszuteilung), (c) ein Angreifer schaltet sich dazwischen. Feststellen: Beim Administrator oder in der Verwaltungsoberfläche des Hosters nachfragen, ob `app01` neu aufgesetzt wurde; den Fingerabdruck des Servers lokal auf dem Server mit `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` ausgeben und mit dem Fingerabdruck vergleichen, den der Client sieht (`ssh-keyscan` und `ssh-keygen -lf`). Weitergehen: Erst wenn die Ursache geklärt und harmlos ist, entfernt man den alten Eintrag mit `ssh-keygen -R app01` und nimmt den neuen auf; die Sicherheitsprüfung bleibt eingeschaltet. Für das nächtliche Skript: ein eigenes Schlüsselpaar nur für dieses Skript mit `ssh-keygen -t ed25519` (ohne Passphrase, da unbeaufsichtigt), privater Schlüssel mit Rechten 600 im Verzeichnis des Skriptbenutzers, öffentlicher Schlüssel auf dem Zielserver in `authorized_keys` eines eigenen, schwach berechtigten Benutzers, bei Bedarf mit Einschränkungen wie `command="..."` und `restrict`, so dass der Schlüssel nur den einen Befehl ausführen darf.

**Aufgabe 5.** Die Funktion bildet mit `process_iter` und angeforderten Feldern ein Wörterbuch je Prozess. Prozesse, deren Felder `None` sind (gesperrt oder verschwunden), werden übersprungen. Die Sortierung nutzt `key=lambda ...` auf dem gewählten Schlüssel; die Übersetzung von `nach` in den Schlüssel erledigt ein kleines Wörterbuch, das bei einem unbekannten Wert einen `KeyError` auslöst. Als Rechenzeit gilt die Summe aus `user` und `system` aus `cpu_times()`.

```python
import os

import psutil


def top_prozesse(anzahl=5, nach="speicher"):
    """Liefert die Prozesse mit dem größten Speicher- oder CPU-Verbrauch als Liste von Wörterbüchern."""
    schluessel = {"speicher": "rss", "cpu": "cpu"}[nach]
    prozesse = []
    for proc in psutil.process_iter(["pid", "name", "memory_info", "cpu_times"]):
        info = proc.info
        if info["memory_info"] is None or info["cpu_times"] is None:     # gesperrt oder verschwunden
            continue
        prozesse.append({
            "pid": info["pid"],
            "name": info["name"],
            "rss": info["memory_info"].rss,
            "cpu": info["cpu_times"].user + info["cpu_times"].system,
        })
    prozesse.sort(key=lambda p: p[schluessel], reverse=True)
    return prozesse[:anzahl]


top = top_prozesse(3)
print("Anzahl:", len(top))
print("absteigend sortiert:", [p["rss"] for p in top] == sorted((p["rss"] for p in top), reverse=True))
block = b"x" * (300 * 1024 * 1024)               # mein Prozess wird groß
eigener = [p for p in top_prozesse(10_000) if p["pid"] == os.getpid()][0]
print("Mein Prozess belegt jetzt mehr als 300 MiB:", eigener["rss"] > 300 * 1024 * 1024)
del block
print("Sortierung nach CPU möglich:", len(top_prozesse(2, nach="cpu")) == 2)
```

```text
Anzahl: 3
absteigend sortiert: True
Mein Prozess belegt jetzt mehr als 300 MiB: True
Sortierung nach CPU möglich: True
```

Die Liste hat die verlangte Länge und ist nach `rss` absteigend geordnet. Das Testprogramm belegt 300 Mebibyte und sucht den eigenen Prozess in der vollständigen Liste (mit sehr großer `anzahl`); sein `rss` liegt über der Grenze. Nach dem Freigeben mit `del block` ist der Speicher wieder frei. Die Sortierung nach `cpu` funktioniert ebenfalls.

**Aufgabe 6.** Die Funktion überspringt die Kopfzeile und zerlegt jede Zeile mit `split(None, 5)` in höchstens sechs Teile: Das letzte Feld enthält den Rest der Zeile samt Leerzeichen, so dass Einhängepunkte wie `/mnt/Backup Platte` unversehrt bleiben. Hat eine Zeile weniger als sechs Teile, wird ein `ValueError` mit der Zeile ausgelöst. Das Prozentzeichen wird entfernt und der Rest in eine Zahl umgewandelt.

```python
def parse_df(text):
    """Liest die Ausgabe von 'df -P' und liefert eine Liste von Wörterbüchern."""
    zeilen = text.strip().splitlines()[1:]                    # Kopfzeile überspringen
    ergebnis = []
    for zeile in zeilen:
        teile = zeile.split(None, 5)                           # Einhängepunkt darf Leerzeichen enthalten
        if len(teile) != 6:
            raise ValueError(f"Unerwartete Zeile: {zeile!r}")
        dateisystem, groesse, belegt, frei, prozent, mount = teile
        ergebnis.append({
            "dateisystem": dateisystem,
            "groesse_kib": int(groesse),
            "belegt_kib": int(belegt),
            "frei_kib": int(frei),
            "prozent": int(prozent.rstrip("%")),
            "mount": mount,
        })
    return ergebnis


beispiel = """Filesystem     1024-blocks     Used Available Capacity Mounted on
/dev/sda1         61255492 17647044  43592064      29% /
tmpfs                65536        0     65536       0% /dev/shm
/dev/sdb1        103081248 98000000   5081248      95% /mnt/Backup Platte
"""
for eintrag in parse_df(beispiel):
    print(f"{eintrag['mount']:20} {eintrag['prozent']:3} %  frei: {eintrag['frei_kib'] // 1024} MiB")

voll = [e["mount"] for e in parse_df(beispiel) if e["prozent"] >= 90]
print("Kritisch:", voll)
try:
    parse_df("Kopf\nzu kurz 1 2\n")
except ValueError as fehler:
    print("Fehler:", fehler)
```

```text
/                     29 %  frei: 42570 MiB
/dev/shm               0 %  frei: 64 MiB
/mnt/Backup Platte    95 %  frei: 4962 MiB
Kritisch: ['/mnt/Backup Platte']
Fehler: Unerwartete Zeile: 'zu kurz 1 2'
```

Alle drei Einträge sind lesbar, auch der mit Leerzeichen im Einhängepunkt. Die Auswertung der kritischen Dateisysteme (90 Prozent oder mehr) findet nur die dritte Platte. Die kaputte Zeile erzeugt die Fehlermeldung mit dem Zeileninhalt. Eine mögliche Verbesserung wäre, `psutil.disk_usage` zu verwenden, wo es verfügbar ist, und diesen Weg nur für entfernte Server zu nehmen, auf denen kein `psutil` installiert ist.

**Aufgabe 7.** Die Funktion liest die Konfiguration mit `SSHConfig.parse` und fragt mit `lookup(alias)` den fertig zusammengeführten Eintrag ab (erster passender Wert gewinnt). `hostname` ist immer vorhanden (bei unbekannten Aliasen der Alias selbst). Port und Benutzer erhalten Vorgaben, Schlüsseldateien stehen als Liste unter `identityfile`, `~` wird mit `os.path.expanduser` aufgelöst. `ConnectTimeout` wird zum Zeitlimit.

```python
import io
import os

import paramiko


def ssh_optionen(alias, konfigtext, vorgabe_benutzer="root"):
    """Übersetzt einen Eintrag aus einer ssh_config in Argumente für SSHClient.connect()."""
    konfig = paramiko.SSHConfig()
    konfig.parse(io.StringIO(konfigtext))
    eintrag = konfig.lookup(alias)
    optionen = {
        "hostname": eintrag["hostname"],
        "port": int(eintrag.get("port", 22)),
        "username": eintrag.get("user", vorgabe_benutzer),
    }
    if "identityfile" in eintrag:
        optionen["key_filename"] = [os.path.expanduser(p) for p in eintrag["identityfile"]]
    if "connecttimeout" in eintrag:
        optionen["timeout"] = float(eintrag["connecttimeout"])
    return optionen


TEXT = """
Host web*
    User deploy
    Port 2200
    ConnectTimeout 4

Host *
    IdentityFile ~/.ssh/id_ed25519
"""
for alias in ("web01", "db01"):
    o = ssh_optionen(alias, TEXT)
    o["key_filename"] = [os.path.basename(p) for p in o["key_filename"]]
    print(alias, "->", o)
```

```text
web01 -> {'hostname': 'web01', 'port': 2200, 'username': 'deploy', 'key_filename': ['id_ed25519'], 'timeout': 4.0}
db01 -> {'hostname': 'db01', 'port': 22, 'username': 'root', 'key_filename': ['id_ed25519']}
```

`web01` passt zu `web*` und erhält den Benutzer `deploy`, Port 2200 und das Zeitlimit 4 sowie den Schlüssel aus `Host *`. `db01` passt nur zum allgemeinen Abschnitt und bekommt die Vorgaben (Port 22, Benutzer `root`). Zur Ausgabe wurden die Schlüsselpfade auf die Dateinamen gekürzt, damit die Ausgabe nicht vom Benutzerverzeichnis abhängt.

**Aufgabe 8.** Die Funktion führt `gruppe.run(befehl, hide=True, warn=True)` aus. Bei der `SerialGroup` und der `ThreadingGroup` sammelt Fabric Fehler einzelner Server und löst am Ende eine `GroupException` aus; deren Attribut `result` enthält die Ergebnisse *aller* Server, wobei fehlgeschlagene Server als Ausnahmeobjekt auftauchen. Die Funktion nimmt diese Ergebnisse und bildet für Ausnahmen ein Paar aus `False` und dem Typnamen, für Ergebnisse ein Paar aus `ok` und Text.

```python
from fabric import SerialGroup, ThreadingGroup
from fabric.exceptions import GroupException

import testssh


def auf_allen(gruppe, befehl):
    """Führt den Befehl auf allen Hosts der Gruppe aus: {adresse: (ok, text)}."""
    try:
        ergebnis = gruppe.run(befehl, hide=True, warn=True)
    except GroupException as fehler:
        ergebnis = fehler.result                                # enthält auch die Erfolge
    ausgabe = {}
    for verbindung, r in ergebnis.items():
        if isinstance(r, Exception):
            ausgabe[verbindung.host] = (False, type(r).__name__)
        else:
            ausgabe[verbindung.host] = (r.ok, r.stdout.strip() or r.stderr.strip())
    return ausgabe


umgebung = testssh.starte(["web01", "web02"])
umgebung.starte_host("web03", 3, autorisiert=False)
umgebung.starte_host("db01", 4)
verbinden = {"key_filename": str(umgebung.schluessel), "allow_agent": False, "look_for_keys": False, "timeout": 5}
hosts = [f"{umgebung.benutzer}@{a}:{p}" for a, p in umgebung.hosts.values()]

for klasse in (SerialGroup, ThreadingGroup):
    gruppe = klasse(*hosts, connect_kwargs=verbinden)
    ergebnis = auf_allen(gruppe, "echo $HOSTALIAS")
    print(klasse.__name__, "->", dict(sorted(ergebnis.items())))
print("Befehl mit Fehlercode:", dict(sorted(auf_allen(ThreadingGroup(*hosts[:2], connect_kwargs=verbinden), "ls /gibt/es/nicht").items())))
umgebung.stoppe()
```

```text
SerialGroup -> {'127.0.0.11': (True, 'web01'), '127.0.0.12': (True, 'web02'), '127.0.0.13': (False, 'AuthenticationException'), '127.0.0.14': (True, 'db01')}
ThreadingGroup -> {'127.0.0.11': (True, 'web01'), '127.0.0.12': (True, 'web02'), '127.0.0.13': (False, 'AuthenticationException'), '127.0.0.14': (True, 'db01')}
Befehl mit Fehlercode: {'127.0.0.11': (False, "ls: cannot access '/gibt/es/nicht': No such file or directory"), '127.0.0.12': (False, "ls: cannot access '/gibt/es/nicht': No such file or directory")}
```

Beide Gruppen liefern dasselbe Ergebnis; die `ThreadingGroup` arbeitet gleichzeitig und ist bei vielen Servern schneller, die `SerialGroup` geht nacheinander vor. Der Server, der unseren Schlüssel nicht kennt, taucht mit dem Namen der Ausnahme auf, die anderen mit ihrem Servernamen. Ein Befehl, der auf dem Server scheitert (Datei fehlt), wird dank `warn=True` als normales Ergebnis mit `ok=False` und der Fehlermeldung des Servers gemeldet.
