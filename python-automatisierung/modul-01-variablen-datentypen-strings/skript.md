# Modul 01 - Variablen, Datentypen, Strings

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (1.1 bis 1.5), danach folgen „Übungsaufgaben zu Modul 01“ (8 Aufgaben) und „Lösungen zu Modul 01“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 27.10.2026 bis Mo 02.11.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; die Grundlagen stehen am Anfang der Lernreihenfolge)

**Meilenstein:** Speicherplatz-Umrechner

**Lernziele.** Nach diesem Modul können Sie Python interaktiv und als Skript ausführen, Werte in Variablen speichern und die grundlegenden Datentypen (Ganzzahlen, Kommazahlen, Wahrheitswerte, Zeichenketten) sicher unterscheiden. Sie rechnen mit Zahlen, formatieren Ausgaben mit f-Strings, lesen Eingaben ein und wandeln sie in den passenden Typ um. Am Ende steht ein kleines, vollständiges Werkzeug aus dem Admin-Alltag: ein Umrechner für Speichergrößen.

| **Nr.** | **Lerneinheit**                | **Datum**     | **Dauer** | **Prio** |
|:--------|:-------------------------------|:--------------|:----------|:---------|
| 1.1     | REPL, erstes Skript, Variablen | Di 27.10.2026 | 2 h       | 1        |
| 1.2     | Datentypen & Rechnen           | Mi 28.10.2026 | 2 h       | 1        |
| 1.3     | Strings & f-Strings            | Do 29.10.2026 | 2 h       | 1        |
| 1.4     | input() & Typumwandlung        | Fr 30.10.2026 | 2 h       | 1        |
| 1.5     | Speicherplatz-Umrechner + Quiz | Mo 02.11.2026 | 2 h       | 1        |
|         | **Summe**                      |               | **10 h**  |          |

## 1.1 REPL, erstes Skript, Variablen

Python ist eine Programmiersprache, die auf Lesbarkeit und kurze Wege vom Gedanken zum laufenden Programm ausgelegt ist. Ein Programm besteht aus Textdateien, dem *Quelltext*. Damit der Computer ihn ausführen kann, braucht er einen *Interpreter*: ein Programm, das den Quelltext einliest, intern in einen Zwischencode (Bytecode) übersetzt und diesen Schritt für Schritt abarbeitet. Sie müssen Ihr Programm also nicht wie in C oder Go erst in eine eigene ausführbare Datei übersetzen, sondern rufen den Interpreter mit dem Namen Ihrer Datei auf. Für die Systemadministration unter Linux ist das ideal: Python ist auf nahezu jeder Distribution entweder vorinstalliert oder in einem Paket verfügbar, ein Skript lässt sich in Minuten schreiben und ändern, und die mitgelieferte Standardbibliothek deckt Dateien, Prozesse, Netzwerk, Zeit, JSON und vieles mehr ab, ohne dass Sie zusätzliche Software installieren müssen. Dieses Prinzip nennt man „batteries included“.

Wichtig ist die Versionsfrage. Es gibt nur noch Python in der Hauptversion 3; Python 2 wird seit 2020 nicht mehr gepflegt. Unter Linux heißt der Befehl deshalb meist `python3`, und auf manchen Systemen existiert der kurze Name `python` gar nicht. Welche Version bei Ihnen läuft, zeigt der Aufruf `python3 --version`. Dieses Skript verlangt Python 3.11 oder neuer, so wie es der Lehrplan vorsieht.

> **Ergänzung aus der Nachrecherche**
>
> **Aktueller Stand der Python-Versionen (abgerufen im Oktober 2026).** Nach der Übersicht der Python-Entwickler (Developer’s Guide, Seite „Status of Python versions“) und der Zusammenstellung auf endoflife.date (Stand 27.10.2026) ist Python 3.15 am 09.10.2026 erschienen (aktuell 3.15.2). Python 3.14 stammt vom 07.10.2025 (aktuell 3.14.10, Fehlerkorrekturen bis 01.10.2027, Sicherheitskorrekturen bis 31.10.2030). Für ältere Zweige endet der Sicherheitssupport am 31.10.2029 (3.13), 31.10.2028 (3.12) und 31.10.2027 (3.11). Python 3.10 ist seit dem 01.10.2026 am Ende seiner Unterstützung und sollte nicht mehr neu eingesetzt werden. Ubuntu 26.04 LTS liefert laut einer Installationsanleitung (linuxize.com) Python 3.14 als Standard-Interpreter aus; `pip` und `venv` sind dort separate Pakete (`python3-pip`, `python3-venv`). Die Beispiele in diesem Skript sind so geschrieben, dass sie unter Python 3.11 bis 3.15 laufen; ausgeführt und geprüft wurden sie mit Python 3.13.18. Versionsnummern ändern sich laufend, prüfen Sie den aktuellen Stand im Quellenverzeichnis.

Zum Ausprobieren starten Sie Python ohne Argument, indem Sie in der Shell `python3` eingeben. Es erscheint die Eingabeaufforderung `>>>`. Das ist die *REPL*, kurz für Read-Eval-Print-Loop: Python liest Ihre Eingabe, wertet sie aus, gibt das Ergebnis aus und wartet auf die nächste Eingabe. Die REPL ist Ihr Notizblock und Taschenrechner. Hier probieren Sie eine Funktion oder einen Ausdruck aus, bevor Sie ihn in ein Skript übernehmen. Beenden können Sie die REPL mit `exit()` oder, unter Linux, mit der Tastenkombination Strg+D. Ab Python 3.13 bietet die Standard-REPL mehrzeiliges Editieren und Farbausgabe; ab 3.14 wird zusätzlich die Syntax farbig hervorgehoben (sofern Ihr Terminal Farben unterstützt). Das ändert nichts am Verhalten, nur an der Lesbarkeit.

Der folgende Auszug zeigt eine kurze Sitzung. Alles hinter `>>>` tippen Sie selbst ein; die Zeilen ohne Prompt sind die Antworten von Python. Beachten Sie den Unterschied zwischen dem bloßen Namen `name`, der den Wert samt Anführungszeichen als Darstellung zeigt, und `print(name)`, das den reinen Text ausgibt.

```pycon
>>> 2 + 3
5
>>> name = "server01"
>>> name
'server01'
>>> print(name)
server01
>>> len(name)
8
```

In der REPL gilt eine Besonderheit: Der Wert des zuletzt ausgegebenen Ausdrucks steht in der Variable `_`, sodass Sie nach `7 * 6` mit `_ + 1` direkt weiterrechnen können. In Skripten gibt es dieses Verhalten nicht, dort erscheint nur, was Sie mit `print()` ausgeben.

Ein *Skript* ist eine Textdatei mit der Endung `.py`, die Sie mit einem beliebigen Editor schreiben (unter Linux etwa `nano`, `vim` oder Visual Studio Code) und mit `python3 dateiname.py` starten. Das erste Skript legt zwei Werte ab und gibt sie aus.

```python
# hallo.py - mein erstes Skript
server = "srv01"
anzahl_benutzer = 12

print("Server:", server)
print("Angemeldete Benutzer:", anzahl_benutzer)
```

```text
Server: srv01
Angemeldete Benutzer: 12
```

Die erste Zeile beginnt mit dem Zeichen `#` und ist ein *Kommentar*. Python ignoriert alles ab diesem Zeichen bis zum Zeilenende; Kommentare sind für Menschen gedacht. In der zweiten Zeile steht eine *Zuweisung*: Der Wert rechts vom Gleichheitszeichen wird berechnet und unter dem Namen links abgelegt. Die Funktion `print()` schreibt ihre Argumente auf den Bildschirm. Gibt man mehrere Argumente durch Kommas getrennt an, setzt `print()` zwischen sie je ein Leerzeichen und am Ende einen Zeilenumbruch. Das wird gleich wichtig: Die Zahl `12` und der Text `"Server:"` sind verschiedene Datentypen, und `print()` kommt mit beiden zurecht, weil es jedes Argument selbst in Text umwandelt.

Damit sind wir bei den *Variablen*. Eine Variable ist in Python ein Name, der auf einen Wert im Speicher zeigt. Man kann sich das wie ein Namensschild vorstellen, das man an einen Gegenstand klebt, nicht wie eine Schublade mit festem Inhalt. Dieselbe Variable kann später auf einen Wert eines ganz anderen Typs zeigen, denn Python legt den Typ am Wert fest, nicht am Namen. Man nennt das *dynamische Typisierung*. Welchen Typ ein Wert hat, erfahren Sie mit der Funktion `type()`.

Für Namen gelten wenige feste Regeln. Sie dürfen Buchstaben, Ziffern und den Unterstrich enthalten, dürfen aber nicht mit einer Ziffer beginnen, und Groß- und Kleinschreibung wird unterschieden: `Server` und `server` sind zwei verschiedene Namen. Die reservierten Wörter der Sprache wie `if`, `for`, `class` oder `import` sind als Namen verboten. Darüber hinaus gibt der Python-Styleguide PEP 8 Konventionen vor, an die sich praktisch alle Projekte halten: Variablen und Funktionen schreibt man klein mit Unterstrichen („snake_case“, etwa `anzahl_benutzer`), Konstanten komplett groß (`MAX_VERSUCHE`), Klassen in „CamelCase“. Einrückungen bestehen aus vier Leerzeichen. Aussagekräftige Namen sind wichtiger als kurze: `freier_platz_gb` erklärt sich selbst, `x` nicht.

```python
host = "srv01"
print(type(host))

port = 22
print(type(port))

port = "ssh"          # derselbe Name zeigt jetzt auf einen Text
print(type(port))

a, b = 3, 4           # mehrfache Zuweisung
a, b = b, a           # Werte tauschen, ohne Hilfsvariable
print(a, b)

x = y = 0             # beide Namen zeigen auf den Wert 0
print(x, y)

MAX_VERSUCHE = 3      # Konstante: nur eine Konvention
print(MAX_VERSUCHE)
```

```text
<class 'str'>
<class 'int'>
<class 'str'>
4 3
0 0
3
```

Das Beispiel zeigt mehrere Dinge zugleich. Zuerst entsteht `host` als Text (Typ `str`), danach `port` als Ganzzahl (`int`); anschließend wird `port` neu zugewiesen und zeigt auf einen Text, was Python ohne Beanstandung zulässt. Die mehrfache Zuweisung `a, b = 3, 4` legt rechts ein Paar von Werten an und verteilt es links auf die Namen; beim Tauschen `a, b = b, a` wird die rechte Seite vollständig berechnet, bevor zugewiesen wird, deshalb braucht es keinen Zwischenspeicher. Die Großschreibung bei `MAX_VERSUCHE` verhindert nichts, Python erlaubt weiterhin eine Neuzuweisung. Sie ist ein Signal an Leserinnen und Leser, dass der Wert sich nicht ändern soll.

Beim Einstieg passieren immer wieder dieselben Fehler. Der häufigste ist ein Tippfehler im Namen. Python meldet dann einen `NameError`, und seit Version 3.10 schlägt es bei ähnlich klingenden Namen gleich eine Korrektur vor.

```python
server = "srv01"
print(servr)
```

```text
NameError: name 'servr' is not defined. Did you mean: 'server'?
```

Lesen Sie die letzte Zeile einer Fehlermeldung immer zuerst: Sie nennt die Art des Fehlers und die Ursache. Eine Zeile darüber steht, in welcher Zeile Ihres Skripts er passiert ist. Ein zweiter klassischer Fehler ist das Überschreiben eingebauter Namen. Wer eine Variable `str`, `list`, `input` oder `type` nennt, verdeckt die gleichnamige Funktion, und spätere Aufrufe schlagen auf rätselhafte Weise fehl.

```python
# FALSCH: der Name str gehört Python und wird hier überschrieben
str = "abc"
print(str(42))
```

```text
TypeError: 'str' object is not callable
```

Die Lösung ist, sprechende eigene Namen zu wählen, hier zum Beispiel `text` oder `beschreibung`. Weitere Stolperfallen: Das einfache Gleichheitszeichen weist zu, der Vergleich zweier Werte wird mit `==` geschrieben (Modul 2). Und Python reagiert empfindlich auf Einrückung. Beginnt eine Zeile ohne Grund mit einem Leerzeichen, endet das mit einem `IndentationError: unexpected indent`, weil Einrückung in Python Blöcke bildet (ebenfalls Modul 2).

> **Merke**
>
> Eine Variable ist ein Name für einen Wert; der Typ gehört zum Wert, nicht zum Namen. Schreiben Sie Namen in `snake_case`, vermeiden Sie eingebaute Namen wie `str` oder `input`, und lesen Sie Fehlermeldungen von unten nach oben: Die letzte Zeile nennt die Ursache.

> **Praxisbeispiel**
>
> Ein Admin-Skript beginnt oft mit einer Bestandsaufnahme des Systems. Das folgende Programm benutzt das Standardmodul `platform`, um Rechnername, Betriebssystem und Python-Version auszugeben, und legt die Werte dazu in Variablen ab. Solche Zeilen stehen später am Anfang jeder Protokolldatei, damit man bei einem Fehlerbericht sofort weiß, auf welchem System das Skript lief.

```python
import platform

rechner = platform.node()
system = platform.system()
kernel = platform.release()
python_version = platform.python_version()

print("Rechner:", rechner)
print("System:", system, kernel)
print("Python:", python_version)
```

Die erste Zeile lädt mit `import` das Modul `platform` aus der Standardbibliothek; Module behandelt Modul 7 ausführlich. Danach rufen vier Funktionen des Moduls Informationen ab, die wir in Variablen ablegen und mit `print()` ausgeben. Auf dem Lehrrechner sieht das etwa wie folgt aus; Ihre Werte werden abweichen.

```text
Rechner: srv01
System: Linux 6.8.0-45-generic
Python: 3.13.18
```

## 1.2 Datentypen & Rechnen

Jeder Wert in Python hat einen Typ, und der Typ bestimmt, welche Operationen erlaubt sind: Zahlen kann man addieren, Texte aneinanderhängen, aber eine Zahl und einen Text kann man nicht addieren. Die fünf Grundtypen dieses Moduls sind `int` (ganze Zahlen), `float` (Kommazahlen), `bool` (Wahrheitswerte), `str` (Zeichenketten) und `NoneType` mit dem einzigen Wert `None`, der „kein Wert“ bedeutet. Mit `type()` fragen Sie den Typ ab, mit `isinstance()` prüfen Sie, ob ein Wert zu einem Typ gehört.

Ganze Zahlen (`int`) haben in Python keine feste Obergrenze. Wo andere Sprachen bei 32 oder 64 Bit überlaufen, rechnet Python einfach mit beliebig vielen Stellen weiter, bis der Speicher ausgeht. Das ist für Admin-Aufgaben angenehm, etwa wenn Sie Byte-Zahlen im Petabyte-Bereich oder große Zähler auswerten. Zur besseren Lesbarkeit dürfen Sie in Zahlenliteralen Unterstriche als Tausendertrenner verwenden, und Sie können Zahlen auch hexadezimal (`0xFF`), oktal (`0o755`) oder binär (`0b1010`) schreiben. Die oktale Schreibweise taucht bei Dateirechten auf: `0o755` entspricht den Rechten `rwxr-xr-x`.

Kommazahlen (`float`) schreibt man mit Dezimalpunkt, nicht mit Komma: `3.14`. Auch die Exponentialschreibweise ist möglich (`1.5e3` ist 1500.0). Intern speichert Python Kommazahlen im Binärformat nach dem IEEE-754-Standard mit doppelter Genauigkeit, und das hat eine Folge, die jeder kennen sollte: Viele Dezimalbrüche wie 0,1 lassen sich im Binärsystem nicht exakt darstellen, genauso wie ein Drittel im Dezimalsystem nicht endlich ist. Es entstehen winzige Rundungsfehler.

Der Typ `bool` kennt die beiden Werte `True` und `False` (mit großem Anfangsbuchstaben). Technisch ist `bool` ein Untertyp von `int`: `True` verhält sich wie 1, `False` wie 0. Das ist selten nützlich, erklärt aber manche Überraschung, etwa dass `True + True` den Wert 2 ergibt. Wahrheitswerte entstehen meist als Ergebnis von Vergleichen; ihnen widmet sich Modul 2.

Die folgende REPL-Sitzung probiert die Typen und die Grundrechenarten aus. Beachten Sie vor allem den Unterschied der drei Divisionsarten: `/` liefert immer einen `float`, `//` rundet das Ergebnis der Division nach unten ab (Ganzzahldivision), und `%` liefert den Rest.

```pycon
>>> type(42), type(3.14), type(True), type("text"), type(None)
(<class 'int'>, <class 'float'>, <class 'bool'>, <class 'str'>, <class 'NoneType'>)
>>> 17 + 5, 17 - 5, 17 * 5
(22, 12, 85)
>>> 17 / 5
3.4
>>> 17 // 5
3
>>> 17 % 5
2
>>> 2 ** 10
1024
>>> 8 / 2
4.0
>>> divmod(17, 5)
(3, 2)
>>> isinstance(3.0, float), isinstance(True, int)
(True, True)
```

Die Tabelle fasst die Operatoren zusammen. Python wertet nach den üblichen mathematischen Regeln aus: Zuerst kommt die Potenz, dann Multiplikation, Division und Rest, zuletzt Addition und Subtraktion. Klammern ändern die Reihenfolge. Das Minuszeichen vor einer Potenz ist eine Falle: `-3 ** 2` ergibt $-9$, weil zuerst potenziert und dann negiert wird; gemeint ist meist `(-3) ** 2`.

| **Operator**                 | **Bedeutung**                         | **Hinweis**                                 | **Beispiel** |
|:-----------------------------|:--------------------------------------|:--------------------------------------------|:-------------|
| `+`, `-`, `*` | Addition, Subtraktion, Multiplikation | Ergebnis `int`, wenn beide `int` sind | `6 * 7`     |
| `/`                     | Division                              | liefert immer `float`                      | `7 / 2`     |
| `//`                     | Ganzzahldivision                      | rundet nach unten ab                        | `7 // 2`     |
| `%`                     | Rest (Modulo)                         | Vorzeichen folgt dem Teiler                 | `7 % 2`     |
| `**`                     | Potenz                                | auch Wurzeln möglich: `x ** 0.5`              | `2 ** 8`     |
| `+=`, `-=`, …        | Kurzform der Zuweisung                | `n += 1` ist `n = n + 1`                       | `n = 5; n += 2`     |

Ein typischer Rechenfall aus dem Alltag ist die Auslastung eines Datenträgers. Das folgende Skript berechnet aus Gesamt- und belegtem Speicher den freien Platz und die Auslastung in Prozent und demonstriert dabei `round()`, die Rundungsfunktion. Die Zahlen sind frei erfunden, die Rechnung entspricht aber genau dem, was Monitoringwerkzeuge tun.

```python
gesamt_gb = 500
belegt_gb = 387.5

frei_gb = gesamt_gb - belegt_gb
auslastung = belegt_gb / gesamt_gb * 100

print("Frei:", frei_gb, "GB")
print("Auslastung:", auslastung, "%")
print("Gerundet:", round(auslastung, 1), "%")

# Wie viele Tage reicht der Platz, wenn täglich 2,5 GB dazukommen?
zuwachs_pro_tag = 2.5
tage = frei_gb / zuwachs_pro_tag
print("Reicht noch:", int(tage), "Tage")
```

```text
Frei: 112.5 GB
Auslastung: 77.5 %
Gerundet: 77.5 %
Reicht noch: 45 Tage
```

Die Subtraktion mischt eine `int` (500) mit einer `float` (387.5); Python wandelt in solchen Fällen die Ganzzahl automatisch in eine Kommazahl um, das Ergebnis ist daher ebenfalls ein `float`. Die Division liefert wie immer einen `float`. Die Funktion `round(zahl, stellen)` rundet auf die angegebene Zahl von Nachkommastellen. Die Funktion `int()` schneidet bei Kommazahlen die Nachkommastellen ab (sie rundet nicht), was hier gewollt ist: Wir wollen nur volle Tage ausgeben. Die Rechnung am Ende, wie lange der freie Platz bei gleichbleibendem Zuwachs reicht, ist die einfachste Form einer Kapazitätsplanung.

Die Rundungsfehler von Kommazahlen sollten Sie einmal mit eigenen Augen sehen. Das nächste Beispiel zeigt die Folgen und die üblichen Gegenmittel: Kommazahlen nie mit `==` vergleichen, sondern mit `math.isclose()`, und für Geldbeträge oder andere Fälle, in denen Dezimalstellen exakt sein müssen, den Typ `Decimal` aus dem Modul `decimal` verwenden.

```python
import math
from decimal import Decimal

summe = 0.1 + 0.2
print(summe)
print(summe == 0.3)
print(math.isclose(summe, 0.3))
print(Decimal("0.1") + Decimal("0.2"))

print(round(0.5), round(1.5), round(2.5))
print(-7 // 2, -7 % 2)
print(2 ** 100)
print(math.sqrt(144), math.floor(3.7), math.ceil(3.2))
```

```text
0.30000000000000004
False
True
0.3
0 2 2
-4 1
1267650600228229401496703205376
12.0 3 4
```

Die Summe aus 0,1 und 0,2 ist in Binärdarstellung nicht exakt 0,3, daher schlägt der Vergleich fehl, während `math.isclose()` mit einer kleinen Toleranz arbeitet. `Decimal` rechnet dagegen tatsächlich dezimal; man erzeugt es aus einem Text, nicht aus einer Kommazahl, da sonst der Rundungsfehler schon im Argument stünde. Zwei weitere Überraschungen: Die Funktion `round()` rundet exakte Hälften zur nächsten geraden Zahl („Bankers Rounding“), deshalb ergibt `round(0.5)` den Wert 0 und `round(2.5)` den Wert 2. Und die Ganzzahldivision rundet immer in Richtung minus Unendlich; `-7 // 2` ist daher $-4$, nicht $-3$, und der Rest `-7 % 2` ist 1, sodass die Beziehung `(a // b) * b + a % b == a` immer gilt. Die Zahl `2 ** 100` zeigt schließlich die unbegrenzte Genauigkeit der Ganzzahlen.

> **Merke**
>
> Vergleichen Sie Kommazahlen nie mit `==`. Verwenden Sie `math.isclose()` oder rechnen Sie mit Ganzzahlen (zum Beispiel in Byte statt in Gigabyte). Die Division `/` liefert immer einen `float`, die Ganzzahldivision `//` rundet nach unten, und `round()` rundet Hälften zur geraden Zahl.

Ein letzter Stolperstein sind Rechenfehler mit gemischten Typen. Wer eine Zahl und einen Text mit `+` verbindet, erhält einen `TypeError`; die Division durch null löst einen `ZeroDivisionError` aus. Beides sind keine Schönheitsfehler, sondern das Programm bricht an dieser Stelle ab. Wie man solche Fehler abfängt, lernen Sie in Modul 6; für den Moment gilt, dass man sie durch sorgfältiges Umwandeln der Typen (Lerneinheit 1.4) und durch Prüfen vor der Division (Modul 2) vermeidet.

```python
# FALSCH: Zahl und Text lassen sich nicht addieren
anzahl = 3
meldung = "Fehlgeschlagene Logins: " + anzahl
```

```text
TypeError: can only concatenate str (not "int") to str
```

```python
# RICHTIG: die Zahl in Text umwandeln oder f-String benutzen
anzahl = 3
meldung = "Fehlgeschlagene Logins: " + str(anzahl)
print(meldung)
print(f"Fehlgeschlagene Logins: {anzahl}")
```

```text
Fehlgeschlagene Logins: 3
Fehlgeschlagene Logins: 3
```

Die Umwandlung mit `str()` und die bequemere Variante mit dem f-String behandelt die nächste Lerneinheit.

> **Praxisbeispiel**
>
> Zahlen und Typen begegnen Ihnen im Betrieb ständig: Ein Monitoringwert wie „Auslastung 77,5 %“ ist ein `float`; ein Zähler für fehlgeschlagene Anmeldungen ein `int`; ein Flag „Dienst läuft“ ein `bool`; ein nicht vorhandener Messwert ein `None`. Ob ein Skript 10 Dateien oder 10 Millionen verarbeitet, ändert am Rechnen nichts, dank der unbegrenzten Ganzzahlen. Bei Geldbeträgen, Lizenzkosten oder Abrechnungen sollten Sie dagegen `Decimal` verwenden, weil Kommazahlen Rundungsabweichungen erzeugen.

## 1.3 Strings & f-Strings

Eine Zeichenkette (englisch *string*, Typ `str`) ist eine Folge von Zeichen, in Python 3 genauer von Unicode-Zeichen. Unicode ist der weltweite Standard, der jedem Zeichen, ob Buchstabe, Ziffer, Umlaut oder Emoji, eine eindeutige Nummer (einen *Codepunkt*) zuordnet. Python-Strings kommen deshalb mit Umlauten und fremden Schriften ohne Weiteres zurecht. Strings sind *unveränderlich* (englisch *immutable*): Man kann einen vorhandenen String nie ändern, sondern nur einen neuen daraus erzeugen. Diese Eigenschaft prägt den Umgang mit Strings, wie wir gleich sehen.

Geschrieben werden Strings in einfachen oder doppelten Anführungszeichen; beide sind gleichwertig. Das erlaubt, das jeweils andere Zeichen im Text zu verwenden: `"it's"` oder `'sagte "hallo"'`. Mehrzeilige Texte stehen in dreifachen Anführungszeichen. Sonderzeichen schreibt man mit dem Fluchtzeichen (Backslash): `\n` ist ein Zeilenumbruch, `\t` ein Tabulator, `\\` ein einzelner Backslash. Wer den Backslash wörtlich haben will, etwa in Windows-Pfaden oder regulären Ausdrücken, schreibt einen *Raw-String* mit vorangestelltem `r`. Das kleine Programm fasst die Schreibweisen zusammen.

```python
a = "Python"
b = 'auch "Python"'
c = "Zeile 1\nZeile 2"
d = r"C:\neu\tabelle"      # Raw-String: Backslashes bleiben erhalten
e = """Das ist
ein mehrzeiliger Text."""

print(a)
print(b)
print(c)
print(d)
print(e)
print(len(a), len(c))
```

```text
Python
auch "Python"
Zeile 1
Zeile 2
C:\neu\tabelle
Das ist
ein mehrzeiliger Text.
6 15
```

Das Beispiel zeigt, dass `\n` in `c` tatsächlich einen Zeilenumbruch erzeugt, während der Raw-String `d` den Text unverändert lässt. Die Funktion `len()` zählt die Zeichen; in `c` sind es 15, weil der Umbruch als ein Zeichen zählt.

Auf einzelne Zeichen greift man mit einem *Index* in eckigen Klammern zu. Gezählt wird ab 0; negative Indizes zählen von hinten, `-1` ist also das letzte Zeichen. Ein *Slice* `text[von:bis]` schneidet einen Abschnitt aus: Das Zeichen am Anfangsindex ist dabei enthalten, das am Endindex nicht. Dieses Halboffen-Prinzip hat den Vorteil, dass `text[:n] + text[n:]` immer wieder den ganzen Text ergibt und dass die Länge des Slice einfach die Differenz der Grenzen ist. Ein dritter Wert gibt die Schrittweite an; `[::-1]` kehrt den Text um.

```pycon
>>> datei = "backup-2026-10-28.tar.gz"
>>> datei[0], datei[-1]
('b', 'z')
>>> datei[7:17]
'2026-10-28'
>>> datei[:6]
'backup'
>>> datei[-6:]
'tar.gz'
>>> datei[::2]
'bcu-061-8trg'
>>> "ssh"[::-1]
'hss'
>>> "ssh" in "sshd"
True
>>> "ab" * 3 + "!"
'ababab!'
```

Für die tägliche Arbeit sind die String-Methoden wichtiger als der Slice. Eine *Methode* ist eine Funktion, die zu einem Wert gehört und mit einem Punkt aufgerufen wird: `text.upper()`. Da Strings unveränderlich sind, liefern alle diese Methoden einen *neuen* String und lassen das Original unberührt. Wer das Ergebnis nicht in einer Variablen auffängt, verwirft es. Die wichtigsten Methoden: `upper()` und `lower()` ändern die Schreibweise; `strip()` entfernt Leerraum (Leerzeichen, Tabulatoren, Zeilenumbrüche) an beiden Enden, `lstrip()` und `rstrip()` nur links beziehungsweise rechts; `replace(alt, neu)` ersetzt Teilstrings; `startswith()` und `endswith()` prüfen Anfang und Ende; `find()` liefert die Position eines Teilstrings oder `-1`, `index()` dagegen einen Fehler, wenn nichts gefunden wird; `count()` zählt Vorkommen; `split()` zerlegt einen Text in Teile, `join()` setzt Teile wieder zusammen; `partition(trenner)` teilt in genau drei Teile (vor, Trenner, nach); `zfill(n)` füllt eine Zahl links mit Nullen auf.

Die Methode `split()` verdient besondere Aufmerksamkeit, weil sie das wichtigste Werkzeug für das Zerlegen von Logzeilen ist. Ohne Argument trennt sie an beliebig langen Folgen von Leerraum und verwirft leere Teile; mit einem Argument trennt sie genau an diesem Text und erzeugt bei doppelten Trennern leere Teile. Das Ergebnis ist eine Liste (Modul 2), aus der Sie einzelne Teile wie bei einem String per Index entnehmen.

```python
zeile = "Oct 28 18:01:02 srv01 sshd[812]: Failed password for root from 10.0.0.5 port 22 ssh2"

teile = zeile.split()
print(teile[0], teile[1], teile[2])      # Monat, Tag, Uhrzeit
print("Host:", teile[3])
print("Anzahl Teile:", len(teile))

vorher, trenner, nachher = zeile.partition(" from ")
print("Quelle:", nachher.split()[0])

print("Failed" in zeile, zeile.count("s"))
print("a  b".split(), "a  b".split(" "))
print("  Leerzeichen  ".strip() + "|")
print("srv01.example.org".replace(".example.org", ""))
print("5".zfill(3), ",".join(["a", "b", "c"]))

name = "srv01"
name.upper()                              # Ergebnis wird verworfen!
print(name)
name = name.upper()
print(name)
```

```text
Oct 28 18:01:02
Host: srv01
Anzahl Teile: 14
Quelle: 10.0.0.5
True 7
['a', 'b'] ['a', '', 'b']
Leerzeichen|
srv01
005 a,b,c
srv01
SRV01
```

Die Zeile ist eine typische Meldung des SSH-Servers aus `/var/log/auth.log`. `zeile.split()` zerlegt sie in 14 Wörter; die ersten drei bilden das Datum, das vierte ist der Rechnername. Mit `partition` teilen wir am Text `" from "` und nehmen danach das erste Wort, das ist die IP-Adresse des Angreifers. Die letzten Zeilen zeigen die Unveränderlichkeit: Der Aufruf `name.upper()` berechnet `"SRV01"`, doch ohne Zuweisung verfällt das Ergebnis, `name` bleibt unverändert. Erst `name = name.upper()` ändert, wohin der Name zeigt. Ein direkter Schreibversuch wie `name[0] = "S"` ist verboten und löst einen `TypeError` aus.

Zum Einbauen von Werten in Text benutzt man heute *f-Strings*. Ein f-String ist ein String mit vorangestelltem `f`, in dem Ausdrücke in geschweiften Klammern berechnet und eingesetzt werden. Hinter einem Doppelpunkt folgt optional eine Formatangabe, die festlegt, wie der Wert dargestellt wird. Das Muster lautet `{ausdruck:format}`. Die wichtigsten Formatangaben sind `.2f` (Festkommazahl mit zwei Nachkommastellen), `,` oder `_` als Tausendertrenner, `%` für Prozentwerte, `>10` (rechtsbündig in Breite 10), `<10` (linksbündig), `^10` (zentriert) und `05d` (Ganzzahl mit führenden Nullen auf Breite 5). Zwei Besonderheiten: `{wert=}` gibt den Ausdruck samt Wert aus, das ist ein praktisches Hilfsmittel beim Fehlersuchen; `{wert!r}` gibt die Darstellung mit Anführungszeichen aus, wie sie die REPL zeigt. Wörtliche geschweifte Klammern schreibt man doppelt.

```python
host = "srv01"
frei = 112.5
gesamt = 500
bytes_frei = 120795955200
port = 22

print(f"Host {host} hat {frei} GB frei")
print(f"Das sind {frei / gesamt:.1%} von {gesamt} GB")
print(f"Das sind {bytes_frei:,} Byte")
print(f"Das sind {bytes_frei:_} Byte")
print(f"{host:>10}|{port:<6}|{frei:^9.2f}|")
print(f"Port {port:05d}")
print(f"{port=}, {host!r}")
print(f"Geschweifte Klammern: {{host}}")
print(f"Rechnung: {2 * 21} und {host.upper()}")
```

```text
Host srv01 hat 112.5 GB frei
Das sind 22.5% von 500 GB
Das sind 120,795,955,200 Byte
Das sind 120_795_955_200 Byte
     srv01|22    | 112.50  |
Port 00022
port=22, 'srv01'
Geschweifte Klammern: {host}
Rechnung: 42 und SRV01
```

Jede Zeile zeigt ein Prinzip. In den geschweiften Klammern steht ein beliebiger Python-Ausdruck, nicht nur ein Name: `frei / gesamt` rechnet, `host.upper()` ruft eine Methode auf. Die Angabe `.1%` multipliziert mit 100, setzt ein Prozentzeichen dahinter und rundet auf eine Nachkommastelle. Der Tausendertrenner `,` schreibt amerikanisch mit Komma, `_` neutral mit Unterstrich; die deutsche Schreibweise mit Punkt lässt sich über Formatangaben nicht direkt erzeugen, dafür gibt es eine Ersetzung oder das Modul `locale`. Die Angaben mit Größerzeichen, Kleinerzeichen und Dach richten Text in einer festen Breite aus. Diese Tabellenform ist das Fundament für übersichtliche Konsolenausgaben, die Sie in diesem und den folgenden Modulen oft brauchen.

> **Ergänzung aus der Nachrecherche**
>
> **Template-Strings (t-Strings) in Python 3.14.** Mit Python 3.14 (veröffentlicht am 07.10.2025) wurden durch PEP 750 „Template String Literals“ eingeführt. Sie sehen aus wie f-Strings, tragen aber das Präfix `t`. Anders als ein f-String liefert ein t-String keinen fertigen Text, sondern ein Objekt vom Typ `string.templatelib.Template`, das die festen Textteile und die eingesetzten Werte getrennt enthält (laut „What’s New in Python 3.14“). So kann eine Bibliothek die Werte vor dem Zusammensetzen prüfen oder maskieren, etwa bei SQL-Abfragen oder HTML. Für alltägliche Ausgaben bleibt der f-String die richtige Wahl; t-Strings sind ein Werkzeug für Bibliotheksautoren. Sie funktionieren erst ab Python 3.14 und werden in diesem Skript nicht verwendet.

Ältere Programme benutzen zwei andere Schreibweisen für denselben Zweck: die Methode `"{} hat {} GB".format(host, frei)` und den Prozent-Operator `"%s hat %.1f GB" % (host, frei)`. Beide funktionieren weiterhin, und Sie werden ihnen in vorhandenen Skripten und in Logging-Aufrufen (Modul 6) begegnen. In neuem Code sind f-Strings kürzer und lesbarer.

Abschließend ein Wort zur Textkodierung, die im Admin-Alltag oft Ärger macht. Ein Python-`str` besteht aus Unicode-Zeichen. Dateien und Netzwerkverbindungen enthalten jedoch Bytes, und erst eine *Kodierung* legt fest, welche Bytes welches Zeichen bedeuten. Heute ist `UTF-8` der Standard unter Linux. Der Umlaut ä ist ein Zeichen, benötigt in UTF-8 aber zwei Bytes. Mit `encode()` wird aus Text Bytes, mit `decode()` wieder Text.

```python
text = "Größe"
daten = text.encode("utf-8")
print(len(text), len(daten))
print(daten)
print(daten.decode("utf-8"))
```

```text
5 7
b'Gr\xc3\xb6\xc3\x9fe'
Größe
```

Der Text hat fünf Zeichen, die Bytefolge sieben Bytes, weil ö und ß je zwei belegen. Beim Lesen von Dateien (Modul 4) geben Sie die Kodierung deshalb ausdrücklich an. Typische Fehler im Umgang mit Strings sind schnell aufgezählt: Man vergisst das `f` vor dem String, sodass die Klammern wörtlich ausgegeben werden; man vergisst, das Ergebnis einer Methode einer Variablen zuzuweisen; man schreibt einen Windows-Pfad mit einfachem Backslash, sodass `"C:\neu"` plötzlich einen Zeilenumbruch enthält (dagegen hilft der Raw-String); oder man nimmt an, dass `split(" ")` dasselbe tut wie `split()`, obwohl es bei mehrfachen Leerzeichen leere Teile erzeugt.

> **Merke**
>
> Strings sind unveränderlich; jede Methode liefert einen neuen String, den Sie einer Variablen zuweisen müssen. Für Ausgaben benutzen Sie f-Strings mit Formatangaben wie `.2f`, `,` und `>10`. Für Logzeilen ist `split()` ohne Argument meist die richtige Wahl.

> **Praxisbeispiel**
>
> Fast jede Auswertung von Logdateien beginnt mit dem Zerlegen von Zeilen: Datum, Rechnername, Dienst, Meldung. Mit `split()`, `partition()`, Slices und `in` schaffen Sie bereits viele Aufgaben, bevor Sie in Modul 8 reguläre Ausdrücke kennenlernen. Die f-Strings brauchen Sie überall, wo ein Skript Ergebnisse, Berichte oder Fehlermeldungen ausgibt.

## 1.4 input() & Typumwandlung

Bisher standen alle Werte fest im Programm. Mit der Funktion `input()` fragt ein Skript den Benutzer: Es gibt einen Hinweistext aus, wartet, bis eine Zeile eingegeben und mit der Eingabetaste bestätigt wurde, und liefert den eingegebenen Text zurück. Wichtig ist, dass das Ergebnis *immer ein String* ist, auch wenn der Benutzer eine Zahl getippt hat. Wer damit rechnen möchte, muss den Text zuerst in eine Zahl umwandeln. Diese Typumwandlung (englisch *type conversion* oder *casting*) erledigen die Funktionen `int()`, `float()`, `str()` und `bool()`.

Die Umwandlung ist streng: `int("42")` ergibt 42, auch `int(" 42 ")` funktioniert, da Leerraum an den Enden ignoriert wird. Aber `int("3.5")` schlägt fehl, weil der Text keine ganze Zahl ist, und `int("1,5")` ebenfalls. Dafür muss man zuerst mit `float()` umwandeln, und zwar mit Dezimalpunkt; ein deutsches Dezimalkomma ersetzt man vorher mit `replace(",", ".")`. Das Abschneiden von Nachkommastellen erledigt `int()`, wenn es eine Kommazahl bekommt: `int(3.9)` ergibt 3, ohne zu runden. Die Funktion `int()` kennt außerdem ein zweites Argument für die Basis: `int("ff", 16)` ist 255, `int("755", 8)` ist 493.

Bei ungültigen Eingaben löst die Umwandlung einen `ValueError` aus. Das Beispiel prüft die Umwandlung für vier Eingaben mit einer `try`/`except`-Anweisung. Sie wird in Modul 6 ausführlich behandelt; für jetzt reicht die Lesart: „Versuche den Block unter `try`; wenn dabei ein `ValueError` auftritt, springe zu `except`.“ Die Schleife und die Liste, mit denen der Test vier Werte nacheinander durchgeht, erklärt Modul 2.

```python
for eingabe in ["42", " 42 ", "3.5", "1,5", "abc", ""]:
    try:
        zahl = int(eingabe)
        print(repr(eingabe), "->", zahl)
    except ValueError as fehler:
        print(repr(eingabe), "->", fehler)

print(float("3.5"), float("1,5".replace(",", ".")))
print(int(3.9), int("ff", 16), int("755", 8))
print(str(42) + "!", str(3.5), str(True))
print(bool(0), bool(""), bool("0"), bool("False"), bool(None))
```

```text
'42' -> 42
' 42 ' -> 42
'3.5' -> invalid literal for int() with base 10: '3.5'
'1,5' -> invalid literal for int() with base 10: '1,5'
'abc' -> invalid literal for int() with base 10: 'abc'
'' -> invalid literal for int() with base 10: ''
3.5 1.5
3 255 493
42! 3.5 True
False False True True False
```

Die Ausgabe der Umwandlung in Wahrheitswerte ist eine häufige Quelle von Irrtümern. Die Funktion `bool()` ist nur für leere Werte falsch: die Zahl 0, der leere String, `None` und leere Sammlungen. Jeder nichtleere String ist wahr, auch `"0"` und `"False"`. Wer also eine Benutzereingabe „ja/nein“ auswerten will, darf sie nicht mit `bool()` umwandeln, sondern muss den Text vergleichen, wie es Modul 2 zeigt.

Nun zur Eingabe selbst. Das folgende Skript fragt zwei Werte ab und rechnet damit. Bei einem Testlauf in der Shell tippt man die Antworten selbst ein; in diesem Skript sind die Eingaben unten abgedruckt.

```python
name = input("Wie heißt der Server-Admin? ")
groesse_text = input("Größe der Festplatte in GB: ")

groesse = int(groesse_text)
print(f"Hallo {name.strip().title()}!")
print(f"{groesse} GB sind {groesse * 1000:,} MB (dezimal).")
print(type(groesse_text), type(groesse))
```

```text
Wie heißt der Server-Admin? Größe der Festplatte in GB: Hallo Anna!
250 GB sind 250,000 MB (dezimal).
<class 'str'> <class 'int'>
```

Der Hinweistext von `input()` erscheint im echten Lauf direkt vor dem Cursor; weil in dieser Dokumentation die Eingaben nicht sichtbar mitgedruckt werden, stehen beide Hinweistexte hintereinander in der ersten Zeile. Wichtig ist die Zeile `groesse = int(groesse_text)`: Vor der Umwandlung ist `groesse_text` ein String und lässt sich nicht mit 1000 multiplizieren, jedenfalls nicht mit dem gewünschten Ergebnis. Denn Python erlaubt die Multiplikation eines Strings mit einer Zahl und würde `"250" * 1000` zu einem Text mit 3000 Zeichen machen. Auch der umgekehrte Fall führt in die Irre: Mit `"10" + "5"` entsteht der Text `"105"`, nicht die Zahl 15.

Das Gegenstück zu `input()` ist `print()`, das mehr kann, als wir bisher genutzt haben. Mit `sep` legt man das Trennzeichen zwischen mehreren Argumenten fest, mit `end` das Zeilenende (Standard ist der Zeilenumbruch), und mit `file` wählt man den Ausgabekanal. Skripte schreiben Fehlermeldungen üblicherweise auf den Fehlerkanal `sys.stderr`, damit sie sich von normalen Ausgaben trennen lassen (Modul 5).

```python
import sys

print("a", "b", "c", sep="-")
print("Lade", end="")
print("...", end="")
print(" fertig")
print("Das ist eine Fehlermeldung", file=sys.stderr)
```

```text
a-b-c
Lade... fertig
Das ist eine Fehlermeldung
```

Die ersten beiden Aufrufe zeigen `sep` und `end`, der letzte schreibt auf den Fehlerkanal; auf dem Bildschirm sieht man den Unterschied nicht, wohl aber bei einer Umleitung wie `python3 skript.py > ausgabe.txt`, wo nur der normale Kanal in der Datei landet.

Zu den typischen Fehlern gehört das Weglassen der Umwandlung (ein Vergleich wie `groesse_text > 100` schlägt mit einem `TypeError` fehl, da Text und Zahl nicht vergleichbar sind), die Umwandlung einer Kommazahl in einen `int` über den Umweg `int("3.5")` und das Vertrauen auf die Eingabe: Ein Benutzer tippt irgendetwas, und ein Skript, das ohne Prüfung umwandelt, bricht mit einem `ValueError` ab. Gefährlich ist besonders die Funktion `eval()`, die in alten Anleitungen als „bequeme“ Umwandlung gezeigt wird: Sie führt eingegebenen Text als Programmcode aus. Verwenden Sie sie niemals mit Eingaben von außen, sondern `int()` und `float()`.

> **Merke**
>
> `input()` liefert immer einen String. Wandeln Sie mit `int()`, `float()` oder `str()` um, rechnen Sie mit dem umgewandelten Wert, und rechnen Sie damit, dass die Umwandlung mit einem `ValueError` scheitern kann. Für Passwörter gibt es `getpass.getpass()`, das die Eingabe nicht anzeigt. Skripte für die Automatisierung lesen ihre Eingaben besser aus Argumenten (Modul 5), weil niemand an der Tastatur sitzt.

> **Praxisbeispiel**
>
> Interaktive Abfragen kommen in Hilfsskripten vor, die ein Mensch bedient, etwa in einem Assistenten, der einen neuen Benutzer anlegt. Skripte, die nachts per Cronjob (Modul 9) laufen, dürfen dagegen nie auf `input()` warten, denn niemand antwortet, und das Skript würde endlos hängen. Dort übergibt man Werte über Argumente, Umgebungsvariablen oder Konfigurationsdateien. Das Umwandeln und Prüfen von Texten bleibt in beiden Fällen dasselbe.

## 1.5 Speicherplatz-Umrechner + Quiz

Der heutige Tag bringt alles zusammen: Wir bauen den Speicherplatz-Umrechner, den Meilenstein des ersten Moduls. Das Programm liest eine Speichergröße samt Einheit ein, etwa „250 GiB“ oder „1.5 TB“, und gibt sie in allen gängigen Einheiten aus. Das klingt einfach, enthält aber eine fachliche Falle, die jedem Systemadministrator früher oder später begegnet: Was bedeutet „Kilo“?

> **Ergänzung aus der Nachrecherche**
>
> **Dezimal- und Binärpräfixe (abgerufen bei NIST, Oktober 2026).** Im Alltag der Informatik bedeutete „Kilo“ lange 1024, weil die Zweierpotenz $2^{10}$ der Zahl 1000 nahekommt. Für Fachleute war das praktisch, für die Öffentlichkeit missverständlich: Nach NIST nutzten Speicherhersteller „Megabyte“ teils als 1 048 576 Byte, teils als 1 000 000 Byte und bei manchen Disketten sogar als 1 024 000 Byte. Deshalb wurden im Dezember 1998 die binären Präfixe der IEC beschlossen: kibi (Ki, $2^{10}$=1 024), mebi (Mi, $2^{20}$=1 048 576), gibi (Gi, $2^{30}$=1 073 741 824), tebi (Ti, $2^{40}$), pebi (Pi, $2^{50}$) und exbi (Ei, $2^{60}$). Ein „Kilobyte“ (kB) sind seither 1000 Byte, ein „Kibibyte“ (KiB) 1024 Byte. Auch die Linux-Werkzeuge unterscheiden das: Laut Hilfetext von `df` gibt `-h` die Größen in Potenzen von 1024 an (zum Beispiel „1023M“), `-H` (auch `--si`) dagegen in Potenzen von 1000 (zum Beispiel „1,1G“). Ein Datenträger mit 500 GB (dezimal) hat daher nur etwa 465,7 GiB, und genau darauf beruht das Rätsel „Warum zeigt mein Betriebssystem weniger Platz an?“.

Wie geht man an ein solches Programm heran? Zuerst werden die Anforderungen in Worte gefasst. Das Programm soll erstens eine Eingabe der Form „Zahl Einheit“ lesen. Zweitens soll es erkennen, ob die Einheit dezimal (KB, MB, GB, TB, PB) oder binär (KiB, MiB, GiB, TiB, PiB) gemeint ist. Drittens soll es den Wert in Byte umrechnen, denn Byte ist die kleinste gemeinsame Einheit, über die sich alles andere ausdrücken lässt. Viertens soll es das Ergebnis in beiden Einheitensystemen als saubere Tabelle ausgeben. Dieses Vorgehen, zuerst in eine Basiseinheit zu normalisieren und dann in die Zielformate zu konvertieren, ist ein Muster, das Sie später bei Zeiten (Sekunden), Netzwerkraten (Bit pro Sekunde) und Prozentwerten ständig anwenden.

Die Einheit lässt sich mit den Mitteln dieses Moduls elegant bestimmen. Die Buchstaben `B`, `K`, `M`, `G`, `T`, `P` entsprechen den Exponenten 0 bis 5. Steht der Text `"BKMGTP"` in einer Variablen, liefert `"BKMGTP".index("G")` genau den Exponenten 3. Für die Basis genügt die Frage, ob in der Einheit ein kleines oder großes „i“ vorkommt; ein Bedingungsausdruck der Form `wert_a if bedingung else wert_b` wählt dann 1024 oder 1000. Bedingungen behandelt Modul 2 ausführlich; hier verwenden wir nur diese eine, sehr lesbare Kurzform. Die Umrechnung in Byte ist danach eine einzige Zeile: `zahl * basis ** exponent`.

Das fertige Programm sieht so aus:

```python
# speicher_umrechner.py - rechnet eine Speichergröße in alle Einheiten um
EINHEITEN = "BKMGTP"          # Position im Text = Exponent der Potenz

eingabe = input("Größe (z. B. 250 GiB oder 1.5 TB): ")
zahl_text, einheit = eingabe.split()
zahl = float(zahl_text.replace(",", "."))

exponent = EINHEITEN.index(einheit[0].upper())
basis = 1024 if "i" in einheit.lower() else 1000
byte = zahl * basis ** exponent

print()
print(f"{eingabe} entsprechen {byte:,.0f} Byte")
print()
print("Dezimal (SI)                Binär (IEC)")
print(f"{byte / 1000:>16,.2f} kB   {byte / 1024:>16,.2f} KiB")
print(f"{byte / 1000**2:>16,.2f} MB   {byte / 1024**2:>16,.2f} MiB")
print(f"{byte / 1000**3:>16,.2f} GB   {byte / 1024**3:>16,.2f} GiB")
print(f"{byte / 1000**4:>16,.2f} TB   {byte / 1024**4:>16,.2f} TiB")
```

```text
Größe (z. B. 250 GiB oder 1.5 TB):
250 GiB entsprechen 268,435,456,000 Byte

Dezimal (SI)                Binär (IEC)
  268,435,456.00 kB     262,144,000.00 KiB
      268,435.46 MB         256,000.00 MiB
          268.44 GB             250.00 GiB
            0.27 TB               0.24 TiB
```

Gehen wir den Quelltext Zeile für Zeile durch. Die Konstante `EINHEITEN` enthält die Einheitenbuchstaben; die Position eines Buchstabens im Text ist sein Exponent. Die Eingabe `"250 GiB"` wird mit `split()` an den Leerzeichen zerlegt, und die Mehrfachzuweisung verteilt die beiden Teile auf `zahl_text` und `einheit`. Dabei wird auch das deutsche Dezimalkomma durch einen Punkt ersetzt, damit `float()` die Zahl versteht. Der erste Buchstabe der Einheit (in Großbuchstaben) bestimmt mit `index()` den Exponenten, das „i“ die Basis. Die Zeile `byte = zahl * basis ** exponent` führt die Normalisierung in Byte aus; das ist der eigentliche Kern des Programms. Bei der Ausgabe sorgen die Formatangaben `>16,.2f` für rechtsbündige Zahlen in Breite 16 mit Tausendertrennern und zwei Nachkommastellen, sodass die Spalten sauber untereinander stehen. Der Ausdruck `1000**2` ist eine Million, `1024**3` ein Gibi. Die Tabelle zeigt es deutlich: 250 GiB sind 268,44 GB, die Binäreinheit ist also größer.

Das Programm funktioniert für richtige Eingaben, bricht aber bei falschen ab. Probieren Sie es aus: Gibt jemand nur „250“ ohne Einheit ein, scheitert das Auspacken mit `ValueError: not enough values to unpack`; gibt jemand „250 XB“ ein, meldet `index()` den Fehler `ValueError: substring not found`; und „abc GB“ scheitert bei `float()`. Alle drei Fälle sind `ValueError`, was für die Fehlerbehandlung günstig ist. In Modul 2 ergänzen Sie Bedingungen und Wiederholungen, in Modul 6 die saubere Fehlerbehandlung. Eine Zwischenlösung mit den heutigen Mitteln fängt die Fehler mit `try`/`except` ab und gibt eine freundliche Meldung aus.

```python
EINHEITEN = "BKMGTP"
eingabe = input("Größe (z. B. 250 GiB): ")

try:
    zahl_text, einheit = eingabe.split()
    zahl = float(zahl_text.replace(",", "."))
    exponent = EINHEITEN.index(einheit[0].upper())
except ValueError:
    print("Ungültige Eingabe. Erwartet wird: Zahl und Einheit, z. B. 250 GiB")
else:
    basis = 1024 if "i" in einheit.lower() else 1000
    print(f"{zahl * basis ** exponent:,.0f} Byte")
```

```text
Größe (z. B. 250 GiB): Ungültige Eingabe. Erwartet wird: Zahl und Einheit, z. B. 250 GiB
```

Alle Zeilen, die schiefgehen können, stehen im `try`-Block. Tritt dort ein `ValueError` auf, läuft der `except`-Block, andernfalls der `else`-Block. Das ist bereits ein robustes Programm, auch wenn die vollständige Fehlerlehre erst in Modul 6 folgt.

> **Praxisbeispiel**
>
> Genau diese Umrechnungen brauchen Sie ständig: Ein Kunde bestellt „2 TB“ Speicher, die Virtualisierung rechnet in GiB, und das Monitoring meldet in Prozent. Ein Skript, das Werte zuerst in Byte normalisiert, vermeidet die klassischen Verwechslungen zwischen GB und GiB. Wenn Sie später mit `shutil.disk_usage()` (Modul 4) die tatsächliche Belegung einer Partition abfragen, bekommen Sie das Ergebnis in Byte und können es mit diesem Umrechner in lesbare Einheiten verwandeln.

Zum Abschluss des Moduls folgt das Quiz. Die Übungsaufgaben 1 bis 4 sind die Quizfragen zum Wissen aus den Lerneinheiten 1.1 bis 1.4, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben, die den Umrechner wieder aufgreifen. Versuchen Sie, jede Aufgabe zuerst ohne Hilfsmittel zu lösen, und prüfen Sie dann Ihre Lösung im Anschluss mit der Erklärung.

## Übungsaufgaben zu Modul 01

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Welche der folgenden Namen sind in Python als Variablennamen gültig, und welche entsprechen der PEP-8-Konvention für Variablen: `server1`, `1server`, `MaxWert`, `max_wert`, `class`, `_temp`, `freier-platz`?

**Aufgabe 2.** Bestimmen Sie ohne Computer Wert und Typ der folgenden Ausdrücke: `7 / 2`, `7 // 2`, `-7 // 2`, `7 % 3`, `2 ** 3 ** 2`, `0.1 + 0.2 == 0.3`, `bool("0")`, `int(3.99)`, `round(2.5)`.

**Aufgabe 3.** Gegeben sei `datei = "messung-2026-10-28.csv"`. Geben Sie an, was die folgenden Ausdrücke liefern: `datei[:7]`, `datei[-3:]`, `datei[8:18]`, `datei.upper()`, `datei.split("-")`, `datei.replace(".csv", ".json")`. Was geschieht bei `datei[0] = "M"`?

**Aufgabe 4.** Der folgende Code soll den Rest der Platte ausgeben, enthält aber drei Fehler. Finden und beheben Sie sie, und erklären Sie jeden Fehler anhand der Fehlermeldung beziehungsweise der falschen Ausgabe.

```python
gesamt = input("Gesamt in GB: ")
belegt = int(input("Belegt in GB: "))
frei = gesamt - belegt
print("Frei: {frei} GB")
```

**Aufgabe 5.** Schreiben Sie ein Programm, das Gesamtgröße und belegten Platz einer Partition in GB einliest (Kommazahlen mit Dezimalpunkt oder -komma sind erlaubt) und den freien Platz sowie die Auslastung in Prozent mit einer Nachkommastelle ausgibt, zum Beispiel `Frei: 112.5 GB, Auslastung: 77.5 %`.

**Aufgabe 6.** Die Datei `backup_srv01_2026-10-28.tar.gz` besteht aus Präfix, Rechnername, Datum und Endung. Schreiben Sie ein Programm, das den Dateinamen einliest und Rechnername, Datum und Endung getrennt ausgibt. Verwenden Sie dazu `split()` und `partition()`.

**Aufgabe 7.** Ein Server meldet seine Laufzeit (Uptime) in Sekunden. Schreiben Sie ein Programm, das eine Sekundenzahl einliest und als `T Tage, HH:MM:SS` ausgibt, zum Beispiel 93784 Sekunden als `1 Tage, 02:03:04`. Nutzen Sie `divmod()` und die Formatangabe `02d`.

**Aufgabe 8.** Erweitern Sie den Umrechner um eine Übertragungszeit: Eingelesen werden eine Dateigröße (zum Beispiel `4.7 GiB`) und eine Übertragungsrate in Megabit pro Sekunde (zum Beispiel `100`). Das Programm soll die Dauer in Minuten und Sekunden ausgeben. Beachten Sie, dass ein Byte aus acht Bit besteht und die Rate dezimal gemeint ist (1 Mbit = 1 000 000 Bit).

## Lösungen zu Modul 01

**Aufgabe 1.** Gültig sind `server1`, `MaxWert`, `max_wert` und `_temp`. Ungültig sind `1server` (beginnt mit einer Ziffer), `class` (reserviertes Wort) und `freier-platz` (der Bindestrich wird als Minus gelesen). Der PEP-8-Konvention für Variablen (Kleinbuchstaben mit Unterstrichen) entsprechen `server1`, `max_wert` und mit Einschränkung `_temp`, das mit dem Unterstrich üblicherweise Namen für „intern“ oder „unwichtig“ kennzeichnet. `MaxWert` wäre eine Schreibweise für Klassennamen.

**Aufgabe 2.** Wir prüfen die Antworten im Programm nach.

```python
for ausdruck in ["7 / 2", "7 // 2", "-7 // 2", "7 % 3", "2 ** 3 ** 2",
                 "0.1 + 0.2 == 0.3", "bool('0')", "int(3.99)", "round(2.5)"]:
    wert = eval(ausdruck)
    print(f"{ausdruck:<18} -> {wert!r:<6} {type(wert).__name__}")
```

```text
7 / 2              -> 3.5    float
7 // 2             -> 3      int
-7 // 2            -> -4     int
7 % 3              -> 1      int
2 ** 3 ** 2        -> 512    int
0.1 + 0.2 == 0.3   -> False  bool
bool('0')          -> True   bool
int(3.99)          -> 3      int
round(2.5)         -> 2      int
```

Die Potenz bindet von rechts nach links: `2 ** 3 ** 2` ist $2^{(3^2)}=2^9=512$. Die Ganzzahldivision rundet nach unten, daher $-4$. Der Vergleich der Kommazahlen scheitert an Rundungsfehlern, `bool('0')` ist wahr, weil der String nicht leer ist, und `round(2.5)` ergibt 2 durch die Rundung zur geraden Zahl. (Hier haben wir `eval()` ausnahmsweise benutzt, weil die Ausdrücke von uns stammen; mit fremden Eingaben ist es gefährlich.)

**Aufgabe 3.**

```python
datei = "messung-2026-10-28.csv"
print(datei[:7])
print(datei[-3:])
print(datei[8:18])
print(datei.upper())
print(datei.split("-"))
print(datei.replace(".csv", ".json"))
```

```text
messung
csv
2026-10-28
MESSUNG-2026-10-28.CSV
['messung', '2026', '10', '28.csv']
messung-2026-10-28.json
```

Die Zuweisung `datei[0] = "M"` scheitert mit einem `TypeError`, denn Strings sind unveränderlich. Gewollt wäre `datei = "M" + datei[1:]`, was einen neuen String erzeugt.

**Aufgabe 4.** Die drei Fehler sind: (1) `gesamt` bleibt ein String, weil die Umwandlung fehlt; die Subtraktion `gesamt - belegt` führt zu einem `TypeError` (nicht unterstützter Operandentyp für `-`). (2) Beim Verwenden von `int()` sind Kommazahlen wie „387,5“ nicht erlaubt; sinnvoller ist `float()`, weil Speichergrößen Nachkommastellen haben können. (3) Dem String fehlt das `f`, sodass `{frei}` wörtlich ausgegeben würde.

```python
gesamt = float(input("Gesamt in GB: "))
belegt = float(input("Belegt in GB: "))
frei = gesamt - belegt
print(f"Frei: {frei} GB")
```

**Aufgabe 5.** Die Lösung wandelt beide Eingaben mit `float()` um und ersetzt vorher ein Dezimalkomma durch den Punkt.

```python
gesamt = float(input("Gesamt in GB: ").replace(",", "."))
belegt = float(input("Belegt in GB: ").replace(",", "."))

frei = gesamt - belegt
auslastung = belegt / gesamt * 100
print()
print(f"Frei: {frei} GB, Auslastung: {auslastung:.1f} %")
```

```text
Gesamt in GB: Belegt in GB:
Frei: 112.5 GB, Auslastung: 77.5 %
```

Die Formatangabe `.1f` rundet die Auslastung auf eine Nachkommastelle. Ein Gesamtwert von 0 würde zu einem `ZeroDivisionError` führen; diesen Fall fängt Modul 2 mit einer Bedingung ab.

**Aufgabe 6.** Nach dem Entfernen der Endung zerlegt man den Namen an den Unterstrichen. Für die Endung nutzt man `partition`, weil der erste Punkt die Endung `tar.gz` vom Datum trennt.

```python
dateiname = input("Dateiname: ").strip()

stamm, punkt, endung = dateiname.partition(".")
praefix, rechner, datum = stamm.split("_")

print()
print("Rechner:", rechner)
print("Datum:  ", datum)
print("Endung: ", endung)
```

```text
Dateiname:
Rechner: srv01
Datum:   2026-10-28
Endung:  tar.gz
```

Die Zeile mit `partition(".")` teilt am ersten Punkt in den Stamm `backup_srv01_2026-10-28`, den Trenner und die Endung `tar.gz`. Danach zerlegt `split("_")` den Stamm in drei Teile, die auf drei Variablen verteilt werden. Das Auspacken klappt nur, wenn der Name genau drei Teile hat; ansonsten gibt es einen `ValueError`.

**Aufgabe 7.** Zuerst teilt man die Sekunden durch 86400 (Sekunden pro Tag), dann den Rest durch 3600 und 60.

```python
sekunden = int(input("Uptime in Sekunden: "))

tage, rest = divmod(sekunden, 86400)
stunden, rest = divmod(rest, 3600)
minuten, sekunden = divmod(rest, 60)

print()
print(f"{tage} Tage, {stunden:02d}:{minuten:02d}:{sekunden:02d}")
```

```text
Uptime in Sekunden:
1 Tage, 02:03:04
```

`divmod(a, b)` liefert Ganzzahlquotient und Rest in einem Schritt. Die Formatangabe `02d` füllt auf zwei Stellen mit Nullen auf. Die Variable `sekunden` wird am Ende überschrieben, was hier unproblematisch ist, da der Anfangswert nicht mehr gebraucht wird.

**Aufgabe 8.** Die Dateigröße rechnet man in Byte um, multipliziert mit acht (Bit) und teilt durch die Rate in Bit pro Sekunde.

```python
EINHEITEN = "BKMGTP"

eingabe = input("Dateigröße (z. B. 4.7 GiB): ")
rate_mbit = float(input("Rate in Mbit/s: ").replace(",", "."))

zahl_text, einheit = eingabe.split()
zahl = float(zahl_text.replace(",", "."))
exponent = EINHEITEN.index(einheit[0].upper())
basis = 1024 if "i" in einheit.lower() else 1000

byte = zahl * basis ** exponent
bit = byte * 8
sekunden = bit / (rate_mbit * 1_000_000)

minuten, rest = divmod(sekunden, 60)
print()
print(f"Dauer: {int(minuten)} min {rest:.0f} s")
```

```text
Dateigröße (z. B. 4.7 GiB): Rate in Mbit/s:
Dauer: 6 min 44 s
```

4,7 GiB sind rund 5,05 Milliarden Byte oder 40,4 Milliarden Bit; bei 100 Mbit/s dauert das gut 400 Sekunden, also etwa 6 Minuten und 44 Sekunden. Typische Fehler sind, das Byte mit dem Bit zu verwechseln (Faktor 8) und die Rate binär statt dezimal zu rechnen. Netzwerkraten werden dezimal angegeben, Dateigrößen dagegen oft binär.
