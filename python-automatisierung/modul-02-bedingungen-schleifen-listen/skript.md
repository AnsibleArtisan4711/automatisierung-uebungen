# Modul 02 - Bedingungen, Schleifen, Listen

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (2.1 bis 2.5), danach folgen „Übungsaufgaben zu Modul 02“ (8 Aufgaben) und „Lösungen zu Modul 02“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 03.11.2026 bis Mo 09.11.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; Kontrollfluss und Listen sind Grundlage für alles Weitere)

**Meilenstein:** Hostnamen/Ports-Skript

**Lernziele.** Nach diesem Modul können Sie Programme schreiben, die Entscheidungen treffen und Arbeitsschritte wiederholen. Sie vergleichen Werte, verknüpfen Bedingungen logisch, durchlaufen Sammlungen mit `for`, wiederholen mit `while` und steuern Schleifen mit `break` und `continue`. Sie verwalten Folgen von Werten in Listen und Tupeln. Am Ende prüft ein Skript eine ganze Liste von Hostnamen und Ports auf Gültigkeit.

| **Nr.** | **Lerneinheit**                         | **Datum**     | **Dauer** | **Prio** |
|:--------|:----------------------------------------|:--------------|:----------|:---------|
| 2.1     | Vergleiche & Bedingungen (if/elif/else) | Di 03.11.2026 | 2 h       | 1        |
| 2.2     | Boolesche Logik & for-Schleifen         | Mi 04.11.2026 | 2 h       | 1        |
| 2.3     | while, break & continue                 | Do 05.11.2026 | 2 h       | 1        |
| 2.4     | Listen & Tupel                          | Fr 06.11.2026 | 2 h       | 1        |
| 2.5     | Hostnamen/Ports-Skript + Quiz           | Mo 09.11.2026 | 2 h       | 1        |
|         | **Summe**                               |               | **10 h**  |          |

## 2.1 Vergleiche & Bedingungen (if/elif/else)

Bisher lief jedes Programm stur von oben nach unten ab. Echte Skripte müssen aber Entscheidungen treffen: Ist die Festplatte voll genug für eine Warnung? Existiert die Datei? Hat der Benutzer „ja“ eingegeben? Dafür gibt es *Bedingungen*. Eine Bedingung ist ein Ausdruck, dessen Ergebnis ein Wahrheitswert (`True` oder `False`) ist, und mit der `if`-Anweisung führt das Programm einen Codeblock nur dann aus, wenn die Bedingung wahr ist.

Wahrheitswerte entstehen vor allem durch *Vergleichsoperatoren*. Python kennt `==` (gleich), `!=` (ungleich), `<`, `<=`, `>` und `>=`. Man kann Vergleiche sogar verketten: `0 < x < 10` prüft, ob `x` zwischen 0 und 10 liegt, genau wie in der Mathematik. Für Zugehörigkeit gibt es die Operatoren `in` und `not in`, für die Identität (dasselbe Objekt, nicht nur gleicher Wert) `is` und `is not`. Letztere verwendet man in der Praxis fast nur für den Vergleich mit `None`: `wert is None`.

```pycon
>>> 5 == 5, 5 != 5, 5 < 7, 5 >= 7
(True, False, True, False)
>>> 0 < 7 < 10
True
>>> "ssh" == "SSH", "ssh" == "SSH".lower()
(False, True)
>>> "ssh" in "sshd", 22 in [22, 80, 443]
(True, True)
>>> 3 == 3.0
True
>>> "3" == 3
False
>>> wert = None
>>> wert is None
True
```

Die Sitzung zeigt einige Feinheiten. Strings werden Zeichen für Zeichen verglichen und unterscheiden Groß- und Kleinschreibung. Eine Ganzzahl und eine Kommazahl mit gleichem Wert sind gleich, ein String und eine Zahl dagegen nie, auch wenn der Text wie die Zahl aussieht. Das ist der Grund, warum man Eingaben aus `input()` zuerst umwandeln muss (Modul 1).

Die `if`-Anweisung hat eine feste Form. Nach dem Schlüsselwort steht die Bedingung, danach ein Doppelpunkt, und der Block, der ausgeführt werden soll, wird *eingerückt*. Die Einrückung ist in Python keine Schönheitsfrage, sondern Teil der Syntax: Alles, was gleich weit eingerückt ist, gehört zum selben Block. Üblich sind vier Leerzeichen. Mit `elif` (kurz für „else if“) hängt man weitere Bedingungen an, die nur geprüft werden, wenn alle vorherigen falsch waren, und `else` fängt alle übrigen Fälle auf. Das folgende Skript klassifiziert die Auslastung eines Datenträgers wie eine Verkehrsampel.

```python
auslastung = float(input("Auslastung in Prozent: "))

if auslastung >= 95:
    status = "KRITISCH"
elif auslastung >= 85:
    status = "WARNUNG"
elif auslastung >= 0:
    status = "OK"
else:
    status = "UNGÜLTIG"

print()
print(f"Status: {status} ({auslastung:.1f} %)")
```

```text
Auslastung in Prozent:
Status: WARNUNG (87.5 %)
```

Python prüft die Bedingungen von oben nach unten und führt genau einen Block aus, den ersten, dessen Bedingung zutrifft. Daraus folgt eine wichtige Regel für die Reihenfolge: Die strengsten Prüfungen stehen zuerst. Würde man die Prüfung `auslastung >= 0` an den Anfang setzen, wäre sie bei jedem gültigen Wert wahr, und die Warnstufen würden nie erreicht. Der `else`-Zweig am Ende fängt hier negative Werte ab, also ungültige Eingaben; solche Absicherungen unterscheiden ein Wegwerfskript von einem Werkzeug, auf das man sich verlassen kann.

Ein `if` kann auch alleine stehen, ohne `elif` und `else`, und Blöcke dürfen verschachtelt werden. Für einfache Wertzuweisungen gibt es außerdem den *bedingten Ausdruck* (Ternary), den Sie schon im Umrechner in Modul 1 gesehen haben: `wert_a if bedingung else wert_b`. Er eignet sich für kurze Fälle, bei längeren Fällen ist ein ausgeschriebenes `if` lesbarer.

```python
port = 443

if port < 1024:
    print("Privilegierter Port")
    if port == 443:
        print("Das ist HTTPS")

art = "Systemport" if port <= 1023 else "Benutzerport"
print(art)

if 0 < port < 65536:
    print("Gültiger Portbereich")
```

```text
Privilegierter Port
Das ist HTTPS
Systemport
Gültiger Portbereich
```

Der innere `if`-Block ist doppelt eingerückt (acht Leerzeichen), weil er zum äußeren Block gehört. Die verkettete Prüfung `0 < port < 65536` ist kürzer und klarer als die Kombination zweier Vergleiche mit `and`, die Sie in der nächsten Lerneinheit kennenlernen.

Zu den wichtigen Eigenschaften von Python gehört die *Wahrheitswertigkeit* beliebiger Werte (englisch *truthiness*). Eine Bedingung muss kein Ergebnis eines Vergleichs sein, jeder Wert kann dort stehen. Als falsch gelten `False`, `None`, die Zahl 0 (auch `0.0`), der leere String und leere Sammlungen wie die leere Liste; alles andere gilt als wahr. Das erlaubt kurze Schreibweisen wie `if name:` statt `if name != "":`. Man sollte sie aber mit Bedacht einsetzen: Die Zahl 0 ist ein sinnvoller Messwert und darf nicht mit „kein Wert“ verwechselt werden. Wer einen fehlenden Wert prüft, vergleicht ausdrücklich mit `None`.

```python
benutzername = ""
if benutzername:
    print("Hallo", benutzername)
else:
    print("Kein Name angegeben")

fehler_anzahl = 0
if fehler_anzahl:
    print("Es gab Fehler")
else:
    print("Keine Fehler")

messwert = None
if messwert is None:
    print("Kein Messwert vorhanden")
```

```text
Kein Name angegeben
Keine Fehler
Kein Messwert vorhanden
```

Seit Python 3.10 gibt es zusätzlich die Mehrfachverzweigung `match`/`case` („Structural Pattern Matching“). Sie wird in diesem Lehrgang nicht benötigt; für die meisten Verzweigungen genügt `if`/`elif`.

Die häufigsten Fehler bei Bedingungen sind schnell genannt. Wer statt `==` ein einzelnes `=` schreibt, erhält einen `SyntaxError`, weil die Zuweisung in einer Bedingung nicht erlaubt ist. Fehlt der Doppelpunkt, erscheint ebenfalls ein `SyntaxError`; ist die Einrückung uneinheitlich, ein `IndentationError`. Bei Kommazahlen sollte man nicht auf exakte Gleichheit prüfen (Modul 1). Und der Vergleich mit `True` ist überflüssig: `if aktiv == True:` schreibt man besser `if aktiv:`.

```text
# FALSCH: Zuweisung statt Vergleich
port = 22
if port = 22:
    print("SSH")
```

```text
SyntaxError: invalid syntax. Maybe you meant '==' or ':=' instead of '='?
```

Die Fehlermeldung nennt bereits die Korrektur. Richtig ist `if port == 22:`.

> **Merke**
>
> Bedingungen werden von oben nach unten geprüft, und es läuft nur der erste passende Block. Die Einrückung (vier Leerzeichen) ist Syntax. Vergleichen Sie mit `==`, fehlende Werte mit `is None`, und benutzen Sie verkettete Vergleiche wie `0 < x < 10`.

> **Praxisbeispiel**
>
> Jedes Monitoring-Skript besteht im Kern aus solchen Schwellenwerten: Unter 85 Prozent Plattenbelegung ist alles in Ordnung, ab 85 gibt es eine Warnung, ab 95 einen Alarm. Auch die Reaktion auf Benutzereingaben, die Wahl des Pfads in Abhängigkeit vom Betriebssystem oder die Entscheidung, ob ein Backup nötig ist, laufen über `if`/`elif`/`else`. Die Ampel aus diesem Abschnitt können Sie später fast unverändert in ein Alerting einbauen (Modul 12).

## 2.2 Boolesche Logik & for-Schleifen

Oft hängt eine Entscheidung von mehreren Bedingungen ab: „Der Dienst läuft nicht *und* es ist keine Wartung“, „der Port ist 80 *oder* 443“. Dafür dienen die *booleschen Operatoren* `and`, `or` und `not`. Der Ausdruck `a and b` ist nur wahr, wenn beide Teile wahr sind; `a or b` ist wahr, wenn mindestens einer wahr ist; `not a` kehrt den Wahrheitswert um. Bei der Auswertung bindet `not` am stärksten, dann `and`, zuletzt `or`; im Zweifel setzen Sie Klammern, was die Lesbarkeit verbessert.

Eine Besonderheit nennt sich *Kurzschlussauswertung*: Python wertet von links nach rechts aus und hört auf, sobald das Ergebnis feststeht. Bei `and` wird der rechte Teil nicht berechnet, wenn der linke falsch ist; bei `or`, wenn der linke wahr ist. Das ist nützlich, um Fehler zu vermeiden: Der Ausdruck `gesamt > 0 and belegt / gesamt > 0.9` teilt nie durch null, weil die erste Prüfung das schon ausschließt. Außerdem liefern `and` und `or` nicht strikt `True` oder `False`, sondern den zuletzt ausgewerteten Operanden. Daraus entsteht die verbreitete Schreibweise für Vorgabewerte: `name = eingabe or "unbekannt"`.

```python
dienst_laeuft = False
wartung = False
port = 8080

if not dienst_laeuft and not wartung:
    print("Alarm: Dienst ausgefallen")

if port == 80 or port == 443 or port == 8080:
    print("Webport")

if port in (80, 443, 8080):
    print("Webport (kürzer mit in)")

gesamt = 0
belegt = 10
if gesamt > 0 and belegt / gesamt > 0.9:
    print("Fast voll")
else:
    print("Kein Problem, und keine Division durch null")

eingabe = ""
name = eingabe or "unbekannt"
print(name)
print(any([False, True, False]), all([True, True, False]))
```

```text
Alarm: Dienst ausgefallen
Webport
Webport (kürzer mit in)
Kein Problem, und keine Division durch null
unbekannt
True False
```

Das Beispiel zeigt auch zwei praktische Funktionen: `any()` ist wahr, wenn mindestens ein Element wahr ist, `all()` wenn alle wahr sind. Und die Prüfung `port in (80, 443, 8080)` ersetzt mehrere `or`-Vergleiche durch eine einzige Zugehörigkeitsprüfung; sie ist kürzer und weniger fehleranfällig. Häufige Fehler bei booleschen Ausdrücken sind die Schreibweise `port == 80 or 443` (das ist immer wahr, weil die Zahl 443 als eigener Ausdruck wahr ist) und fehlende Klammern bei gemischtem `and`/`or`.

Nun zu den Wiederholungen. Eine *Schleife* führt einen Block mehrfach aus. Die `for`-Schleife ist in Python eine Schleife über die Elemente einer Sammlung: Sie nimmt der Reihe nach jedes Element und weist es einer Variablen zu, der *Schleifenvariablen*, und führt für jedes den Block aus. Durchlaufen lassen sich Listen (Lerneinheit 2.4), Strings (Zeichen für Zeichen), und mit `range()` auch Zahlenfolgen. Die Funktion `range(stop)` liefert die Zahlen von 0 bis `stop - 1`, `range(start, stop)` beginnt bei `start`, und `range(start, stop, schritt)` springt in Schritten. Wie bei Slices ist das Ende ausgeschlossen.

```python
for i in range(3):
    print("Durchlauf", i)

for port in range(20, 26):
    print(port, end=" ")
print()

for zahl in range(10, 0, -3):
    print(zahl, end=" ")
print()

for buchstabe in "ssh":
    print(buchstabe.upper(), end=".")
print()

hosts = ["srv01", "srv02", "srv03"]
for host in hosts:
    print(f"Prüfe {host} ...")

for nr, host in enumerate(hosts, start=1):
    print(nr, host)
```

```text
Durchlauf 0
Durchlauf 1
Durchlauf 2
20 21 22 23 24 25
10 7 4 1
S.S.H.
Prüfe srv01 ...
Prüfe srv02 ...
Prüfe srv03 ...
1 srv01
2 srv02
3 srv03
```

Die Ausgabe von `range(20, 26)` endet bei 25, nicht bei 26. Die Schleife über die Liste von Hosts ist die häufigste Form: „Tue etwas mit jedem Rechner“. Die Funktion `enumerate()` liefert zusätzlich eine laufende Nummer, sodass man keinen Zähler von Hand führen muss; mit `start=1` beginnt die Zählung bei 1. Das bekannte Muster aus anderen Sprachen, über `range(len(hosts))` zu laufen und per Index zuzugreifen, ist in Python selten nötig und weniger lesbar.

Ein Grundmuster, das fast jede Schleife begleitet, ist der *Akkumulator*: Man legt vor der Schleife eine Variable an (zum Beispiel eine Summe oder einen Zähler) und verändert sie in jedem Durchlauf. Das folgende Beispiel zählt die Treffer in einer Reihe von Logzeilen und addiert Größen.

```python
zeilen = [
    "Failed password for root from 10.0.0.5",
    "Accepted password for anna from 10.0.0.7",
    "Failed password for admin from 10.0.0.5",
    "Failed password for root from 10.0.0.9",
]

fehlversuche = 0
for zeile in zeilen:
    if "Failed" in zeile:
        fehlversuche += 1

print("Fehlversuche:", fehlversuche)

groessen_mb = [120, 45.5, 300, 8]
summe = 0
for g in groessen_mb:
    summe += g
print("Summe:", summe, "MB, Durchschnitt:", summe / len(groessen_mb))
print(sum(groessen_mb), max(groessen_mb), min(groessen_mb))

for zeile_nr in range(1, 4):
    for spalte in range(1, 4):
        print(zeile_nr * spalte, end="\t")
    print()
```

```text
Fehlversuche: 3
Summe: 473.5 MB, Durchschnitt: 118.375
473.5 300 8
1	2	3
2	4	6
3	6	9
```

Die Variable `fehlversuche` beginnt bei null und wird für jede passende Zeile um eins erhöht. Die Summe ließe sich auch mit der eingebauten Funktion `sum()` bilden; ebenso gibt es `max()` und `min()`. Am Ende steht eine verschachtelte Schleife: Für jede Zeile durchläuft die innere Schleife alle Spalten, was ein kleines Einmaleins ausgibt. Bei verschachtelten Schleifen multiplizieren sich die Durchläufe (hier 3 mal 3), was bei großen Datenmengen schnell zu langen Laufzeiten führt.

Die klassischen Fehler bei `for`-Schleifen sind der Off-by-one-Fehler (`range(1, 10)` enthält die 10 nicht), das Verändern einer Liste, während man sie durchläuft (Elemente werden dann übersprungen), und das Vergessen des Einrückens, wodurch Anweisungen nicht mehr in der Schleife liegen. Das folgende Beispiel zeigt den zweiten Fall und die Abhilfe.

```python
# FALSCH: Elemente entfernen, während die Liste durchlaufen wird
zahlen = [1, 2, 2, 3, 4]
for z in zahlen:
    if z == 2:
        zahlen.remove(z)
print(zahlen)

# RICHTIG: eine neue Liste aufbauen
zahlen = [1, 2, 2, 3, 4]
ohne_zwei = []
for z in zahlen:
    if z != 2:
        ohne_zwei.append(z)
print(ohne_zwei)
```

```text
[1, 2, 3, 4]
[1, 3, 4]
```

Im falschen Beispiel bleibt eine 2 übrig, denn nach dem Entfernen des ersten Elements rutscht das nächste nach vorne, und die Schleife überspringt es. Die richtige Variante sammelt die gewünschten Elemente in einer neuen Liste.

> **Merke**
>
> `and`, `or` und `not` verknüpfen Bedingungen; Python wertet von links nach rechts und bricht ab, sobald das Ergebnis feststeht. Eine `for`-Schleife läuft über die Elemente einer Sammlung; `range(a, b)` endet vor `b`. Verändern Sie eine Liste nicht, während Sie sie durchlaufen.

> **Praxisbeispiel**
>
> `for` ist das Arbeitspferd der Automatisierung: Für jeden Server, jede Datei, jede Zeile einer Logdatei, jeden Benutzer wird derselbe Ablauf ausgeführt. Ein Skript, das 200 Server auf freien Speicherplatz prüft, ist eine Schleife über die Serverliste mit einer Ampel (Lerneinheit 2.1) als Rumpf. Genau diese Struktur taucht in Modul 11 beim Abfragen mehrerer Hosts über SSH wieder auf.

## 2.3 while, break & continue

Die `for`-Schleife eignet sich, wenn man die Menge der Durchläufe kennt. Manchmal weiß man aber nicht im Voraus, wie oft etwas passieren muss: „Frage so lange nach, bis die Eingabe gültig ist“, „versuche die Verbindung, bis sie klappt oder fünf Versuche erreicht sind“. Dafür gibt es die `while`-Schleife. Sie prüft vor jedem Durchlauf eine Bedingung und führt den Block aus, solange diese wahr ist. Damit die Schleife irgendwann endet, muss der Block etwas verändern, das die Bedingung irgendwann falsch macht.

```python
zaehler = 3
while zaehler > 0:
    print("Noch", zaehler, "Sekunden")
    zaehler -= 1
print("Start!")

# Wie oft kann man halbieren, bis weniger als 1 GB übrig ist?
groesse_gb = 100
schritte = 0
while groesse_gb >= 1:
    groesse_gb = groesse_gb / 2
    schritte += 1
print(schritte, "Schritte, Rest:", round(groesse_gb, 3), "GB")
```

```text
Noch 3 Sekunden
Noch 2 Sekunden
Noch 1 Sekunden
Start!
7 Schritte, Rest: 0.781 GB
```

Im ersten Beispiel verringert `zaehler -= 1` den Wert in jedem Durchlauf, bis die Bedingung falsch wird. Im zweiten wird die Größe so lange halbiert, bis sie unter 1 fällt; man kann vorher nicht ohne Rechnung sagen, wie viele Durchläufe es braucht, und genau dafür ist `while` gedacht. Die größte Gefahr ist die *Endlosschleife*: Verändert der Block die Bedingung nie, läuft das Programm ewig. Unterbrechen Sie es in diesem Fall in der Shell mit Strg+C, was einen `KeyboardInterrupt` auslöst.

Zwei Schlüsselwörter steuern den Ablauf innerhalb einer Schleife. `break` beendet die Schleife sofort, und das Programm macht hinter ihr weiter. `continue` bricht nur den aktuellen Durchlauf ab und springt zum nächsten. Beide gelten für `for` und `while`. Mit `break` entsteht ein verbreitetes Muster: die absichtliche Endlosschleife `while True:`, die per `break` verlassen wird, sobald eine Bedingung im Inneren erfüllt ist. Das ist die natürliche Form für „frage, bis die Eingabe stimmt“.

Das folgende Programm greift den Umrechner aus Modul 1 auf und macht die Eingabe robust: Es fragt so lange, bis eine Größe samt gültiger Einheit eingegeben wurde, höchstens aber dreimal. Zur Wiederholung der Eingabe wird in den Beispielen aus diesem Skript die Eingabezeile des Benutzers vorgegeben.

```python
EINHEITEN = "BKMGTP"
MAX_VERSUCHE = 3

byte = None
versuch = 0
while versuch < MAX_VERSUCHE:
    versuch += 1
    eingabe = input(f"Größe mit Einheit ({versuch}/{MAX_VERSUCHE}): ")
    teile = eingabe.split()
    if len(teile) != 2:
        print("  Bitte Zahl und Einheit angeben, z. B. 250 GiB")
        continue
    try:
        zahl = float(teile[0].replace(",", "."))
        exponent = EINHEITEN.index(teile[1][0].upper())
    except ValueError:
        print("  Zahl oder Einheit nicht verständlich")
        continue
    basis = 1024 if "i" in teile[1].lower() else 1000
    byte = zahl * basis ** exponent
    break

if byte is None:
    print("Zu viele ungültige Eingaben, Abbruch.")
else:
    print(f"{byte:,.0f} Byte")
```

```text
Größe mit Einheit (1/3):   Bitte Zahl und Einheit angeben, z. B. 250 GiB
Größe mit Einheit (2/3):   Bitte Zahl und Einheit angeben, z. B. 250 GiB
Größe mit Einheit (3/3): 268,435,456,000 Byte
```

Gehen wir den Ablauf durch. Die Variable `byte` beginnt mit `None`, was „noch kein Ergebnis“ bedeutet. Die Schleife läuft höchstens dreimal. In jedem Durchlauf wird die Eingabe zerlegt. Stimmt die Anzahl der Teile nicht, schreibt das Programm einen Hinweis und springt mit `continue` sofort zum nächsten Versuch. Nur wenn die Eingabe verständlich ist, wird das Ergebnis berechnet und die Schleife mit `break` verlassen. Nach der Schleife zeigt die Prüfung auf `None`, ob je ein Ergebnis entstand. In der Ausgabe stehen die ersten beiden Aufforderungen und Hinweise hintereinander, weil die vorgegebenen Eingaben nicht mitgedruckt werden; im echten Lauf erscheint jede Eingabe hinter ihrer Aufforderung.

Python kennt noch eine Besonderheit, die viele Programmierer überrascht: Eine Schleife darf einen `else`-Zweig haben, der nur ausgeführt wird, wenn die Schleife *nicht* mit `break` verlassen wurde. Er eignet sich für Suchen: „Wenn die Schleife ohne Treffer durchlief, melde, dass nichts gefunden wurde.“

```python
ports = [22, 80, 443]
for port in ports:
    if port == 3306:
        print("Datenbankport gefunden")
        break
else:
    print("Kein Datenbankport in der Liste")

import time
versuch = 0
while versuch < 3:
    versuch += 1
    print("Versuch", versuch)
    time.sleep(0.01)    # Pause, in echten Skripten z. B. 5 Sekunden
```

```text
Kein Datenbankport in der Liste
Versuch 1
Versuch 2
Versuch 3
```

Der zweite Teil zeigt die typische Wiederholung mit Pause (englisch *retry*): Das Modul `time` liefert mit `time.sleep(sekunden)` eine Wartefunktion. In Modul 10 werden Sie solche Wiederholungen mit zunehmenden Pausen für Netzwerkaufrufe verwenden.

Zu den Stolperfallen gehört, den Zähler zu vergessen (Endlosschleife), den Zähler an der falschen Stelle zu verändern (ein `continue` vor dem `zaehler += 1` überspringt die Veränderung und blockiert die Schleife) und die Bedingung mit der falschen Grenze zu schreiben (`<` statt `<=`). Das folgende Beispiel zeigt den zweiten Fall und die korrekte Variante.

```python
# FALSCH: continue überspringt den Zähler, die Schleife hängt bei 3
i = 0
sicherung = 0
while i < 5 and sicherung < 20:
    sicherung += 1          # nur Schutz, damit dieses Beispiel endet
    if i == 3:
        continue
    i += 1
print("i =", i, "nach", sicherung, "Durchläufen")

# RICHTIG: Zähler vor dem continue erhöhen
i = 0
while i < 5:
    i += 1
    if i == 3:
        continue
    print("Verarbeite", i)
```

```text
i = 3 nach 20 Durchläufen
Verarbeite 1
Verarbeite 2
Verarbeite 4
Verarbeite 5
```

Die falsche Variante wäre ohne die künstliche Sicherung eine Endlosschleife: Sobald `i` den Wert 3 erreicht, springt `continue` jedes Mal zurück, ohne `i` zu erhöhen. In der richtigen Variante wird der Zähler zuerst erhöht.

> **Merke**
>
> `while` wiederholt, solange eine Bedingung gilt; der Block muss die Bedingung irgendwann falsch machen. `break` verlässt die Schleife, `continue` springt zum nächsten Durchlauf, und der `else`-Zweig einer Schleife läuft nur ohne `break`. Begrenzen Sie Wiederholungen immer, etwa mit einem Versuchszähler.

> **Praxisbeispiel**
>
> Wiederholungen mit Obergrenze sind im Betrieb allgegenwärtig: Ein Skript wartet, bis ein Dienst nach dem Neustart antwortet, versucht eine Datenbankverbindung mehrfach oder fragt eine Warteschlange ab, bis sie leer ist. Ohne Obergrenze blockiert ein Skript, das auf eine nie eintretende Bedingung wartet, im ungünstigsten Fall tagelang einen Cronjob und damit alle folgenden Läufe. Eine `while`-Schleife mit Zähler und Pause ist das Standardmittel dagegen.

## 2.4 Listen & Tupel

Eine *Liste* ist eine geordnete Folge von Werten, die man in eckigen Klammern schreibt: `hosts = ["srv01", "srv02"]`. Listen können Werte beliebiger Typen enthalten, auch gemischt, und sind *veränderlich*: Man kann Elemente hinzufügen, entfernen und ersetzen. Auf Elemente greift man wie bei Strings mit Index und Slice zu; der Index beginnt bei 0, negative Indizes zählen von hinten.

Die wichtigsten Listenmethoden sind `append(x)` (hängt ein Element hinten an), `extend(liste)` (hängt alle Elemente einer anderen Liste an), `insert(i, x)` (fügt an Position `i` ein), `remove(x)` (entfernt das erste Vorkommen), `pop()` (entfernt und liefert das letzte Element, oder das an einem Index), `index(x)` (Position), `count(x)` (Anzahl), `sort()` (sortiert die Liste an Ort und Stelle) und `reverse()`. Die eingebaute Funktion `sorted(liste)` liefert dagegen eine neue, sortierte Liste und lässt das Original unberührt. Mit `len()` erfährt man die Länge, mit `in` prüft man die Zugehörigkeit, `sum()`, `min()` und `max()` funktionieren wie bei Zahlenfolgen.

```pycon
>>> hosts = ["srv03", "srv01", "srv02"]
>>> hosts[0], hosts[-1], hosts[1:]
('srv03', 'srv02', ['srv01', 'srv02'])
>>> hosts.append("srv04")
>>> hosts
['srv03', 'srv01', 'srv02', 'srv04']
>>> hosts.insert(0, "gw01")
>>> hosts.remove("srv03")
>>> hosts
['gw01', 'srv01', 'srv02', 'srv04']
>>> sorted(hosts, reverse=True)
['srv04', 'srv02', 'srv01', 'gw01']
>>> hosts.pop()
'srv04'
>>> len(hosts), "srv01" in hosts, hosts.index("srv02")
(3, True, 2)
>>> hosts.sort()
>>> hosts
['gw01', 'srv01', 'srv02']
>>> hosts + ["db01"], hosts * 2
(['gw01', 'srv01', 'srv02', 'db01'], ['gw01', 'srv01', 'srv02', 'gw01', 'srv01', 'srv02'])
```

Die Sitzung zeigt, dass `append`, `insert`, `remove` und `sort` die Liste direkt ändern und nichts zurückgeben (die REPL gibt deshalb nichts aus), `pop` dagegen das entfernte Element liefert. Das ist eine häufige Falle: Wer `hosts = hosts.sort()` schreibt, überschreibt seine Liste mit `None`, denn `sort()` gibt nichts zurück. Die Operatoren `+` und `*` erzeugen neue Listen.

Eine Liste lässt sich auch nach eigenen Kriterien sortieren. Dafür übergibt man mit dem Parameter `key` eine Funktion, die für jedes Element den Sortierschlüssel liefert, zum Beispiel `len` für die Länge. Eine nützliche Kurzform zum Erzeugen von Listen ist die *List Comprehension*: `[ausdruck for element in sammlung if bedingung]`. Sie entspricht einer Schleife mit `append`, ist aber kompakter, und man sollte sie benutzen, solange sie lesbar bleibt.

```python
dateien = ["backup.tar.gz", "notizen.txt", "log.txt", "bild.png"]

nach_laenge = sorted(dateien, key=len)
print(nach_laenge)

txt_dateien = [d for d in dateien if d.endswith(".txt")]
print(txt_dateien)

grossbuchstaben = [d.upper() for d in dateien]
print(grossbuchstaben[:2])

quadrate = [n ** 2 for n in range(1, 6)]
print(quadrate)

# dasselbe als Schleife
quadrate2 = []
for n in range(1, 6):
    quadrate2.append(n ** 2)
print(quadrate == quadrate2)

matrix = [[1, 2, 3], [4, 5, 6]]
print(matrix[1][2], len(matrix))
```

```text
['log.txt', 'bild.png', 'notizen.txt', 'backup.tar.gz']
['notizen.txt', 'log.txt']
['BACKUP.TAR.GZ', 'NOTIZEN.TXT']
[1, 4, 9, 16, 25]
True
6 2
```

Die zweite List Comprehension liest sich fast wie ein Satz: „Nimm jedes `d` aus `dateien`, wenn `d` auf `.txt` endet.“ Die Schleife darunter erzeugt dasselbe Ergebnis, und der Vergleich der beiden Listen mit `==` bestätigt das. Das Beispiel mit `matrix` zeigt verschachtelte Listen: `matrix[1][2]` greift zuerst auf die zweite innere Liste und dann auf deren drittes Element zu.

Wichtig zu verstehen ist, dass Listen *Referenzen* sind. Eine Zuweisung `b = a` kopiert die Liste nicht, sondern lässt beide Namen auf dieselbe Liste zeigen. Verändert man sie über einen der Namen, ändert sie sich für beide. Das ist die vielleicht häufigste Überraschung beim Umgang mit Listen.

```python
a = ["srv01", "srv02"]
b = a                  # kein Kopieren, nur ein zweiter Name
b.append("srv03")
print(a)
print(a is b)

c = a.copy()           # echte (flache) Kopie
c.append("srv04")
print(a)
print(c)
print(a is c, a == c[:3])
```

```text
['srv01', 'srv02', 'srv03']
True
['srv01', 'srv02', 'srv03']
['srv01', 'srv02', 'srv03', 'srv04']
False True
```

Nachdem `b` an `a` gebunden wurde, taucht `srv03` auch in `a` auf, und der Operator `is` bestätigt, dass es dasselbe Objekt ist. Erst `a.copy()` (oder `a[:]` oder `list(a)`) legt eine eigene Liste an. Man nennt das eine *flache* Kopie, denn bei verschachtelten Listen werden nur die äußeren Referenzen kopiert; für tiefe Kopien dient `copy.deepcopy()`.

Das Gegenstück zur Liste ist das *Tupel* (englisch *tuple*). Es sieht aus wie eine Liste mit runden Klammern, ist aber *unveränderlich*: Nach dem Anlegen kann man weder Elemente hinzufügen noch ändern. Das ist eine Eigenschaft, keine Einschränkung: Ein Tupel signalisiert „diese Werte gehören fest zusammen“ und eignet sich für feste Datensätze wie eine Adresse aus Host und Port `("srv01", 22)` oder eine Koordinate. Außerdem können Tupel später als Schlüssel in Dictionaries dienen (Modul 3). Praktisch ist das *Entpacken*: Man verteilt die Elemente eines Tupels oder einer Liste in einem Schritt auf mehrere Variablen. Das haben Sie bereits bei `a, b = 3, 4` genutzt, denn die rechte Seite ist ein Tupel. Mit einem Stern sammelt man den Rest.

```python
adresse = ("srv01", 22)
host, port = adresse
print(host, port)

erste, *rest = [10, 20, 30, 40]
print(erste, rest)

ein_element = (5,)          # das Komma macht das Tupel
kein_tupel = (5)
print(type(ein_element), type(kein_tupel))

werte = (3, 1, 2)
print(sorted(werte), len(werte), werte[0])

try:
    werte[0] = 9
except TypeError as fehler:
    print(fehler)

for host, port in [("srv01", 22), ("web01", 443)]:
    print(f"{host}:{port}")
```

```text
srv01 22
10 [20, 30, 40]
<class 'tuple'> <class 'int'>
[1, 2, 3] 3 3
'tuple' object does not support item assignment
srv01:22
web01:443
```

Das Beispiel zeigt die drei wichtigen Eigenschaften des Tupels. Das Entpacken verteilt die Elemente auf Namen, das Komma (nicht die Klammer) macht ein einelementiges Tupel, und ein Schreibversuch scheitert mit einem `TypeError`. Lesende Operationen wie `sorted()`, `len()` oder der Zugriff per Index funktionieren dagegen wie bei Listen; `sorted()` liefert dabei immer eine Liste.

> **Merke**
>
> Listen sind veränderlich, Tupel nicht. Eine Zuweisung `b = a` kopiert eine Liste nicht; benutzen Sie `a.copy()`. Methoden wie `append`, `sort` und `remove` ändern die Liste direkt und geben `None` zurück. Für ein einelementiges Tupel braucht es das Komma: `(5,)`.

> **Praxisbeispiel**
>
> Fast jede Admin-Aufgabe beginnt mit einer Liste: Serverlisten, Dateilisten, Zeilen einer Logdatei (jede Zeile ein Element), die Ausgabe eines Befehls. Wer mit `split()` einen Text zerlegt, erhält eine Liste; wer Dateien eines Verzeichnisses sucht (Modul 4), ebenfalls. Tupel verwenden Sie für feste Paare wie `(host, port)`. Beim Sortieren von Messwerten oder Dateien hilft `sorted()` mit `key`.

## 2.5 Hostnamen/Ports-Skript + Quiz

Der Meilenstein dieses Moduls verbindet Bedingungen, Schleifen und Listen zu einem nützlichen Werkzeug: Das Skript prüft eine Liste von Einträgen der Form `host:port` auf formale Gültigkeit und gibt für jeden Eintrag eine Auswertung aus. Vorher klären wir, was „gültig“ eigentlich heißt.

> **Ergänzung aus der Nachrecherche**
>
> **Regeln für Hostnamen und Ports (abgerufen im Oktober 2026).** Zu Hostnamen heißt es in der Zusammenfassung der Wikipedia-Seite „Hostname“, die sich auf die Standards RFC 952 und RFC 1123 stützt: Ein Name besteht aus Labels, die durch Punkte getrennt sind. Jedes Label ist 1 bis 63 Zeichen lang und darf nur die ASCII-Buchstaben a bis z (ohne Unterscheidung der Schreibweise), die Ziffern 0 bis 9 und den Bindestrich enthalten; Unterstriche sind nicht erlaubt. Ein Label darf nicht mit einem Bindestrich beginnen oder enden. RFC 1123 hat die ältere Regel gelockert, dass ein Label mit einem Buchstaben beginnen muss, sodass nun auch Ziffern am Anfang zulässig sind. Der gesamte Name darf einschließlich der Punkte höchstens 253 Zeichen lang sein. Zu Ports nennt das IANA-Register die Bereiche System-Ports (0 bis 1023), User-Ports (1024 bis 49151) sowie dynamische oder private Ports (49152 bis 65535), die nicht vergeben werden. Häufig hört man die Bezeichnungen „Well-known“ für den ersten und „Registered“ für den zweiten Bereich; das IANA-Register verwendet auf der abgerufenen Seite die Namen System und User. Port 0 ist reserviert; gültige Zielports für Verbindungen liegen zwischen 1 und 65535, da der Wert in 16 Bit gespeichert wird.

Aus diesen Regeln leiten wir die Prüfungen ab. Ein Eintrag `host:port` wird am *letzten* Doppelpunkt getrennt (`rpartition`), damit später auch Formen wie IPv6-Adressen nicht zu Fehlern führen; für diese Übung genügt die einfache Form. Der Hostname wird in Labels zerlegt und jedes Label einzeln geprüft: Länge, erlaubte Zeichen, kein Bindestrich an Anfang oder Ende. Der Port muss eine ganze Zahl zwischen 1 und 65535 sein, und das Skript benennt außerdem seinen Bereich.

Gehen wir das Programm in drei Schritten an. Zuerst die Prüfung des Hostnamens als eigene Aufgabe: Sie gibt entweder `None` zurück (alles in Ordnung) oder einen Text, der den Fehler beschreibt. Die Schleife über die Labels zeigt, wie `for`, `if` und Stringmethoden zusammenspielen. Eigene Funktionen, die wir hier schon einsetzen, behandelt Modul 3 genauer; für den Moment genügt es zu wissen, dass `def` einem Block einen Namen gibt, unter dem man ihn mehrfach aufrufen kann, und dass `return` einen Wert zurückgibt.

```python
ERLAUBT = "abcdefghijklmnopqrstuvwxyz0123456789-"

def pruefe_hostname(host):
    """Gibt None zurück, wenn der Name gültig ist, sonst den Fehlertext."""
    if host == "":
        return "leerer Hostname"
    if len(host) > 253:
        return "Name länger als 253 Zeichen"
    for label in host.lower().split("."):
        if label == "":
            return "leeres Label (doppelter oder abschließender Punkt)"
        if len(label) > 63:
            return f"Label '{label[:10]}...' länger als 63 Zeichen"
        if label.startswith("-") or label.endswith("-"):
            return f"Label '{label}' beginnt oder endet mit Bindestrich"
        for zeichen in label:
            if zeichen not in ERLAUBT:
                return f"unerlaubtes Zeichen '{zeichen}'"
    return None

for name in ["srv01", "web-01.example.org", "-bad.example.org",
             "db_01", "a..b", "3com.example.com", ""]:
    print(f"{name!r:24} {pruefe_hostname(name)}")
```

```text
'srv01'                  None
'web-01.example.org'     None
'-bad.example.org'       Label '-bad' beginnt oder endet mit Bindestrich
'db_01'                  unerlaubtes Zeichen '_'
'a..b'                   leeres Label (doppelter oder abschließender Punkt)
'3com.example.com'       None
''                       leerer Hostname
```

Die Funktion `pruefe_hostname` arbeitet von der groben zur feinen Prüfung: erst leer und Gesamtlänge, dann je Label Länge und Bindestriche und zuletzt jedes Zeichen. Sobald ein Fehler gefunden ist, endet die Funktion mit `return` und dem Fehlertext. Der Name `3com.example.com` ist laut RFC 1123 gültig. Die Zeichen prüft die Operation `zeichen not in ERLAUBT`; weil vorher mit `lower()` in Kleinbuchstaben umgewandelt wurde, sind auch Großbuchstaben im Eingabenamen erlaubt.

Der zweite Schritt prüft den Port und benennt den Bereich. Die Aufteilung der Bereiche folgt dem IANA-Register.

```python
def pruefe_port(text):
    """Liefert (Port als Zahl oder None, Meldung)."""
    try:
        port = int(text)
    except ValueError:
        return None, f"'{text}' ist keine ganze Zahl"
    if not 1 <= port <= 65535:
        return None, f"{port} liegt außerhalb von 1 bis 65535"
    if port <= 1023:
        bereich = "System-Port"
    elif port <= 49151:
        bereich = "User-Port"
    else:
        bereich = "dynamischer Port"
    return port, bereich

for text in ["22", "8080", "50000", "0", "70000", "ssh", " 443 "]:
    print(f"{text!r:9}", pruefe_port(text))
```

```text
'22'      (22, 'System-Port')
'8080'    (8080, 'User-Port')
'50000'   (50000, 'dynamischer Port')
'0'       (None, '0 liegt außerhalb von 1 bis 65535')
'70000'   (None, '70000 liegt außerhalb von 1 bis 65535')
'ssh'     (None, "'ssh' ist keine ganze Zahl")
' 443 '   (443, 'System-Port')
```

Die Funktion gibt ein Tupel zurück, nämlich den Port (oder `None`) und eine Meldung; der Aufrufer entpackt es mit `port, meldung = pruefe_port(text)`. Die verkettete Prüfung `1 <= port <= 65535` nutzt, was Sie in Lerneinheit 2.1 gelernt haben. Beachten Sie auch, dass `int(" 443 ")` funktioniert, weil `int` Leerraum entfernt.

Im dritten Schritt setzen wir alles zu einem Programm zusammen, das eine Liste von Einträgen durchläuft, Gültige und Ungültige zählt und eine Tabelle ausgibt. Als Zugabe prüft das Skript, ob der Port tatsächlich erreichbar ist: Mit dem Modul `socket` öffnet `create_connection` eine TCP-Verbindung, die entweder gelingt oder mit einem `OSError` scheitert. Damit das Beispiel überall reproduzierbar läuft, startet das Programm selbst einen kleinen lokalen Server, der auf einem freien Port lauscht, und prüft zusätzlich einen Port, auf dem niemand lauscht.

```python
import socket

ERLAUBT = "abcdefghijklmnopqrstuvwxyz0123456789-"

def pruefe_hostname(host):
    if host == "":
        return "leerer Hostname"
    if len(host) > 253:
        return "Name länger als 253 Zeichen"
    for label in host.lower().split("."):
        if label == "" or len(label) > 63:
            return "Label leer oder länger als 63 Zeichen"
        if label.startswith("-") or label.endswith("-"):
            return "Label beginnt oder endet mit Bindestrich"
        for zeichen in label:
            if zeichen not in ERLAUBT:
                return f"unerlaubtes Zeichen '{zeichen}'"
    return None

def pruefe_port(text):
    try:
        port = int(text)
    except ValueError:
        return None, "keine ganze Zahl"
    if not 1 <= port <= 65535:
        return None, "außerhalb von 1 bis 65535"
    return port, "ok"

def erreichbar(host, port):
    try:
        with socket.create_connection((host, port), timeout=1):
            return True
    except OSError:
        return False

# lokaler Testserver auf einem freien Port
server = socket.socket()
server.bind(("127.0.0.1", 0))
server.listen()
offener_port = server.getsockname()[1]

# freien, aber geschlossenen Port ermitteln
probe = socket.socket()
probe.bind(("127.0.0.1", 0))
geschlossener_port = probe.getsockname()[1]
probe.close()

eintraege = [
    f"127.0.0.1:{offener_port}",
    f"127.0.0.1:{geschlossener_port}",
    "web_01.example.org:443",
    "srv01:99999",
    "db01:abc",
    "ohne-port",
]

gueltig = 0
for eintrag in eintraege:
    host, doppelpunkt, port_text = eintrag.rpartition(":")
    if doppelpunkt == "":
        print(f"{'(Eintrag)':10} UNGÜLTIG  Format host:port fehlt")
        continue
    fehler = pruefe_hostname(host)
    port, meldung = pruefe_port(port_text)
    if fehler or port is None:
        print(f"{host:10} UNGÜLTIG  {fehler or ''} {'' if port else meldung}".rstrip())
        continue
    gueltig += 1
    status = "erreichbar" if erreichbar(host, port) else "nicht erreichbar"
    print(f"{host:10} gültig    {status}")

print()
print(f"{gueltig} von {len(eintraege)} Einträgen gültig")
server.close()
```

```text
127.0.0.1  gültig    erreichbar
127.0.0.1  gültig    nicht erreichbar
web_01.example.org UNGÜLTIG  unerlaubtes Zeichen '_'
srv01      UNGÜLTIG   außerhalb von 1 bis 65535
db01       UNGÜLTIG   keine ganze Zahl
(Eintrag)  UNGÜLTIG  Format host:port fehlt

2 von 6 Einträgen gültig
```

Die Struktur ist die Standardform vieler Prüfskripte: eine Liste von Eingaben, eine Schleife, in der jeder Eintrag zerlegt, geprüft und bewertet wird, ein Zähler für die Statistik und eine Zusammenfassung. `continue` überspringt Einträge, die nicht weiter verarbeitet werden können, sodass der Rest des Rumpfs nur für gültige Einträge läuft. Die Verbindungsprüfung nutzt eine `with`-Anweisung, die dafür sorgt, dass die Verbindung nach Gebrauch geschlossen wird (Modul 4 erklärt das Prinzip bei Dateien). Der Aufruf scheitert mit einem `OSError`, wenn niemand lauscht; das wird in Modul 6 systematisch behandelt. Wichtig ist: Die Prüfung *gültig* sagt nur, dass der Eintrag formal in Ordnung ist. Ob ein Dienst läuft, zeigt erst die Verbindungsprobe.

Das Quiz zum Modul sind die Übungsaufgaben 1 bis 4; die Aufgaben 5 bis 8 vertiefen das Meilenstein-Skript und die Schleifen.

> **Praxisbeispiel**
>
> Prüfskripte dieser Art laufen in der Praxis vor jeder Änderung an einer Firewall oder an einem Load Balancer: Man erfasst die gewünschten Endpunkte als Liste, prüft sie formal, bevor sie ins System geschrieben werden, und testet danach die Erreichbarkeit. Fehlerhafte Einträge in Konfigurationsdateien, etwa ein Unterstrich im Hostnamen oder ein Port jenseits von 65535, führen sonst zu Dienstausfällen, die sich schwer finden lassen.

## Übungsaufgaben zu Modul 02

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Welche Ausgabe erzeugt der folgende Code? Begründen Sie die Reihenfolge der Prüfungen.

```python
wert = 75
if wert > 50:
    print("A")
elif wert > 70:
    print("B")
else:
    print("C")
```

**Aufgabe 2.** Welchen Wert haben die Ausdrücke `not False and True`, `True or False and False`, `0 or "x"`, `"" and 5`, `1 < 2 < 3`, `[] or [0]`?

**Aufgabe 3.** Was geben `list(range(2, 11, 3))`, `list(range(5, 0, -2))` und `len(range(0, 100, 10))` zurück? Wie oft läuft die Schleife `for i in range(1, 10)`, und welche Zahl ist die letzte?

**Aufgabe 4.** Gegeben ist `a = [1, 2, 3]`, `b = a`, `c = a.copy()`, danach `b.append(4)`. Welche Werte haben `a`, `b` und `c`? Was liefert `a.sort()` als Ergebnis, und was geschieht bei `t = (1, 2); t[0] = 5`?

**Aufgabe 5.** Schreiben Sie ein Programm, das eine Prozentzahl einliest und folgende Ausgabe erzeugt: unter 70 `OK`, ab 70 `BEOBACHTEN`, ab 85 `WARNUNG`, ab 95 `KRITISCH`, außerhalb von 0 bis 100 `UNGÜLTIG`.

**Aufgabe 6.** Gegeben ist die Liste der Ports `[22, 80, 443, 3306, 8080, 49200, 5432]`. Geben Sie alle System-Ports (bis 1023) aus, zählen Sie die User-Ports (1024 bis 49151) und ermitteln Sie den größten Port, ohne `max()` zu verwenden.

**Aufgabe 7.** Ein Skript soll ein Passwort abfragen. Es erlaubt höchstens drei Versuche. Bei richtigem Passwort (hier `geheim`) meldet es „Zugang gewährt“ und endet sofort, nach drei Fehlversuchen „Gesperrt“. Verwenden Sie `while` und `break`.

**Aufgabe 8.** Gegeben ist eine Liste von Hostnamen mit Dubletten und gemischter Schreibweise: `["SRV01", "web01", "srv01", "DB01", "web01", "db01"]`. Erzeugen Sie eine sortierte Liste ohne Dubletten in Kleinbuchstaben und geben Sie aus, wie oft jeder Name ursprünglich vorkam.

## Lösungen zu Modul 02

**Aufgabe 1.** Die Ausgabe ist `A`. Python prüft von oben nach unten und führt den ersten passenden Block aus. Weil 75 größer als 50 ist, trifft die erste Bedingung zu, und die Prüfung `wert > 70` wird nie erreicht. Wollte man drei Stufen unterscheiden, müsste die strengste Prüfung (`wert > 70`) an erster Stelle stehen.

**Aufgabe 2.** Wir prüfen die Antworten im Programm.

```python
print(not False and True)
print(True or False and False)
print(0 or "x")
print(repr("" and 5))
print(1 < 2 < 3)
print([] or [0])
```

```text
True
True
x
''
True
[0]
```

Das Ergebnis von `True or False and False` ist `True`, weil `and` stärker bindet als `or`: Zuerst wird `False and False` berechnet, dann `True or False`. Der Ausdruck `0 or "x"` liefert den ersten wahren Operanden, `"x"`; `"" and 5` liefert den ersten falschen Operanden, den leeren String. Das zeigt, dass `and` und `or` Operanden zurückgeben, nicht nur `True` oder `False`.

**Aufgabe 3.**

```python
print(list(range(2, 11, 3)))
print(list(range(5, 0, -2)))
print(len(range(0, 100, 10)))
```

```text
[2, 5, 8]
[5, 3, 1]
10
```

Die Schleife `for i in range(1, 10)` läuft neunmal; die letzte Zahl ist 9, denn das Ende ist ausgeschlossen.

**Aufgabe 4.**

```python
a = [1, 2, 3]
b = a
c = a.copy()
b.append(4)
print(a, b, c)
print(a.sort())
try:
    t = (1, 2)
    t[0] = 5
except TypeError as fehler:
    print(fehler)
```

```text
[1, 2, 3, 4] [1, 2, 3, 4] [1, 2, 3]
None
'tuple' object does not support item assignment
```

`b` ist nur ein zweiter Name für dieselbe Liste, daher sehen `a` und `b` die Ergänzung; `c` ist eine Kopie und bleibt unverändert. Die Methode `sort()` sortiert an Ort und Stelle und liefert `None`. Ein Tupel ist unveränderlich, die Zuweisung löst einen `TypeError` aus.

**Aufgabe 5.** Die Reihenfolge der Prüfungen läuft von streng nach locker, und ungültige Werte werden zuerst abgefangen.

```python
prozent = float(input("Prozent: "))

if prozent < 0 or prozent > 100:
    status = "UNGÜLTIG"
elif prozent >= 95:
    status = "KRITISCH"
elif prozent >= 85:
    status = "WARNUNG"
elif prozent >= 70:
    status = "BEOBACHTEN"
else:
    status = "OK"

print()
print(status)
```

```text
Prozent:
WARNUNG
```

Das Abfangen der ungültigen Werte an erster Stelle verhindert, dass etwa 120 als „KRITISCH“ durchgeht.

**Aufgabe 6.** Ein Akkumulator sammelt das Maximum, ein Zähler die User-Ports.

```python
ports = [22, 80, 443, 3306, 8080, 49200, 5432]

user_ports = 0
groesster = ports[0]
for port in ports:
    if port <= 1023:
        print("System-Port:", port)
    elif port <= 49151:
        user_ports += 1
    if port > groesster:
        groesster = port

print("User-Ports:", user_ports)
print("Größter Port:", groesster)
```

```text
System-Port: 22
System-Port: 80
System-Port: 443
User-Ports: 3
Größter Port: 49200
```

Der Startwert für das Maximum ist das erste Element, nicht 0; so funktioniert die Suche auch bei Listen mit lauter negativen Zahlen. Die Ports 3306, 8080 und 5432 liegen im User-Bereich, 49200 ist ein dynamischer Port und wird deshalb weder als System- noch als User-Port gezählt.

**Aufgabe 7.**

```python
PASSWORT = "geheim"
MAX_VERSUCHE = 3

versuch = 0
erfolg = False
while versuch < MAX_VERSUCHE:
    versuch += 1
    if input(f"Passwort ({versuch}/{MAX_VERSUCHE}): ") == PASSWORT:
        erfolg = True
        break

print()
print("Zugang gewährt" if erfolg else "Gesperrt")
```

```text
Passwort (1/3): Passwort (2/3): Passwort (3/3):
Zugang gewährt
```

Die Variable `erfolg` merkt sich, ob die Schleife durch `break` verlassen wurde. Alternativ eignet sich der `else`-Zweig der Schleife. In echten Programmen speichert man Passwörter nie im Klartext im Quelltext und liest sie mit `getpass.getpass()` ein, damit sie nicht am Bildschirm erscheinen.

**Aufgabe 8.** Wir wandeln alle Namen in Kleinbuchstaben, bauen eine Liste ohne Dubletten und zählen mit `count()`.

```python
namen = ["SRV01", "web01", "srv01", "DB01", "web01", "db01"]

klein = [n.lower() for n in namen]
eindeutig = []
for n in klein:
    if n not in eindeutig:
        eindeutig.append(n)
eindeutig.sort()

print(eindeutig)
for n in eindeutig:
    print(f"{n}: {klein.count(n)}x")
```

```text
['db01', 'srv01', 'web01']
db01: 2x
srv01: 2x
web01: 2x
```

Die Prüfung `n not in eindeutig` verhindert Dubletten. Bei sehr großen Listen ist dieses Verfahren langsam, weil jede Prüfung die ganze Liste durchsucht; dafür gibt es die Datenstruktur Set (Modul 3), mit der sich die Aufgabe in einer Zeile lösen lässt: `sorted(set(klein))`.
