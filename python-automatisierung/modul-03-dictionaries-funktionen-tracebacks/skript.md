# Modul 03 - Dictionaries, Funktionen, Tracebacks

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (3.1 bis 3.5), danach folgen „Übungsaufgaben zu Modul 03“ (8 Aufgaben) und „Lösungen zu Modul 03“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Di 10.11.2026 bis Mo 16.11.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 1 (Annahme: Der Lehrplan nennt keine Priorität; Dictionaries und Funktionen sind Kernbausteine jedes Skripts)

**Meilenstein:** Log-Analyzer v1

**Lernziele.** Nach diesem Modul können Sie Daten in Dictionaries und Sets strukturieren und die passende Datenstruktur für eine Aufgabe auswählen. Sie schreiben eigene Funktionen mit Parametern und Rückgabewerten, verstehen, wo Variablen gelten (Gültigkeitsbereich), und lesen Fehlermeldungen (Tracebacks) systematisch von unten nach oben. Mit einfachen Debugging-Techniken finden Sie Fehler selbst. Der Meilenstein ist ein erster Log-Analyzer, der Anmeldeversuche aus Protokollzeilen auswertet.

| **Nr.** | **Lerneinheit**                       | **Datum**     | **Dauer** | **Prio** |
|:--------|:--------------------------------------|:--------------|:----------|:---------|
| 3.1     | Dictionaries                          | Di 10.11.2026 | 2 h       | 1        |
| 3.2     | Sets & Datenstrukturen im Vergleich   | Mi 11.11.2026 | 2 h       | 1        |
| 3.3     | Funktionen (Parameter, Rückgabewerte) | Do 12.11.2026 | 2 h       | 1        |
| 3.4     | Scope, Tracebacks lesen, Debugging    | Fr 13.11.2026 | 2 h       | 1        |
| 3.5     | Log-Analyzer v1 + Quiz                | Mo 16.11.2026 | 2 h       | 1        |
|         | **Summe**                             |               | **10 h**  |          |

## 3.1 Dictionaries

Listen sprechen Elemente über ihre Position an: „das dritte Element“. Oft will man aber über einen *Namen* auf Daten zugreifen: „der Port von srv01“, „die Zahl der Fehlversuche für diese IP-Adresse“. Dafür gibt es das *Dictionary* (kurz *Dict*, auf Deutsch etwa „Wörterbuch“ oder „Zuordnungstabelle“). Ein Dictionary speichert *Schlüssel-Wert-Paare*: Jeder Schlüssel (englisch *key*) ist eindeutig und führt zu genau einem Wert (*value*). Man schreibt es in geschweiften Klammern mit Doppelpunkt zwischen Schlüssel und Wert. Der Zugriff über einen Schlüssel geht sehr schnell, auch bei Millionen Einträgen, weil Python den Schlüssel in eine Zahl umrechnet (*Hashing*) und den Wert direkt ansteuert, statt alle Einträge durchzugehen. Seit Python 3.7 bleibt außerdem die Einfügereihenfolge der Einträge erhalten.

Schlüssel müssen *hashbar* sein. Das bedeutet praktisch: unveränderlich. Strings, Zahlen und Tupel (mit hashbaren Inhalten) sind als Schlüssel erlaubt, Listen und andere Dictionaries nicht. Als Werte sind beliebige Objekte erlaubt, auch Listen und weitere Dictionaries. Die REPL-Sitzung zeigt die Grundoperationen.

```pycon
>>> server = {"name": "srv01", "port": 22, "aktiv": True}
>>> server["name"]
'srv01'
>>> server["port"] = 2222
>>> server["os"] = "Ubuntu"
>>> server
{'name': 'srv01', 'port': 2222, 'aktiv': True, 'os': 'Ubuntu'}
>>> "port" in server, "cpu" in server
(True, False)
>>> server.get("cpu")
>>> server.get("cpu", "unbekannt")
'unbekannt'
>>> len(server)
4
>>> del server["os"]
>>> server.pop("aktiv")
True
>>> server
{'name': 'srv01', 'port': 2222}
>>> list(server.keys()), list(server.values())
(['name', 'port'], ['srv01', 2222])
```

Mit `server["name"]` liest man einen Wert, mit `server["os"] = "Ubuntu"` fügt man einen Eintrag hinzu oder ändert einen vorhandenen. Der Operator `in` prüft, ob ein *Schlüssel* vorhanden ist (nicht ein Wert). Sehr wichtig ist der Unterschied zwischen dem Zugriff mit eckigen Klammern und der Methode `get()`: Fehlt der Schlüssel, löst `server["cpu"]` einen `KeyError` aus, während `server.get("cpu")` den Wert `None` liefert (die REPL zeigt `None` nicht an) oder, mit zweitem Argument, einen Vorgabewert. Die Methode `pop(schluessel)` entfernt einen Eintrag und liefert seinen Wert, `del` entfernt ihn ohne Rückgabe.

Dictionaries durchläuft man meistens mit `items()`, das Schlüssel und Wert gleichzeitig liefert. Eine `for`-Schleife direkt über das Dictionary liefert nur die Schlüssel. Das folgende Beispiel zeigt Iteration, Verschachtelung und die Schreibweise als *Dict Comprehension*, die analog zur List Comprehension aus Modul 2 funktioniert.

```python
dienste = {22: "ssh", 80: "http", 443: "https", 3306: "mysql"}

for port, name in dienste.items():
    print(f"Port {port:>5} -> {name}")

print(dienste.get(25, "kein Eintrag"))

# nur Systemports (bis 1023), als neues Dictionary
system = {p: n for p, n in dienste.items() if p <= 1023}
print(system)

# umgekehrt: Name -> Port
umgekehrt = {name: port for port, name in dienste.items()}
print(umgekehrt["https"])

inventar = {
    "srv01": {"ip": "10.0.0.11", "rolle": "web", "ports": [22, 80, 443]},
    "db01": {"ip": "10.0.0.21", "rolle": "datenbank", "ports": [22, 3306]},
}
print(inventar["db01"]["rolle"], inventar["srv01"]["ports"][-1])

for name, daten in inventar.items():
    print(f"{name}: {daten['ip']} ({len(daten['ports'])} Ports)")

zusammen = dienste | {5432: "postgresql"}
print(len(zusammen))
```

```text
Port    22 -> ssh
Port    80 -> http
Port   443 -> https
Port  3306 -> mysql
kein Eintrag
{22: 'ssh', 80: 'http', 443: 'https'}
443
datenbank 443
srv01: 10.0.0.11 (3 Ports)
db01: 10.0.0.21 (2 Ports)
5
```

Das Dictionary `dienste` bildet Portnummern auf Dienstnamen ab, die Zahlen sind hier Schlüssel. Die Comprehension `{p: n for p, n in dienste.items() if p <= 1023}` filtert die Einträge, die zweite kehrt die Zuordnung um. Das verschachtelte Dictionary `inventar` ist die typische Form kleiner Datenbestände: Jeder Server hat wieder ein Dictionary mit eigenen Eigenschaften. Mit `inventar["db01"]["rolle"]` greift man kettenweise zu. Der Operator `|` fügt zwei Dictionaries zusammen (ab Python 3.9); bei doppelten Schlüsseln gewinnt der rechte.

Häufig muss man zählen oder Werte gruppieren: Wie oft kam jede IP-Adresse vor? Mit einem Dictionary lautet das Muster: Wenn der Schlüssel schon vorhanden ist, erhöhe den Wert, sonst lege ihn mit 1 an. Kürzer geht es mit `get` und einem Vorgabewert. Die Standardbibliothek bietet für diesen Fall zwei Werkzeuge im Modul `collections`: `Counter`, ein Dictionary zum Zählen, und `defaultdict`, das für fehlende Schlüssel automatisch einen Startwert anlegt.

```python
from collections import Counter, defaultdict

ips = ["10.0.0.5", "10.0.0.7", "10.0.0.5", "10.0.0.9", "10.0.0.5", "10.0.0.7"]

# 1. Zählen von Hand mit get
zaehler = {}
for ip in ips:
    zaehler[ip] = zaehler.get(ip, 0) + 1
print(zaehler)

# 2. Counter erledigt das in einer Zeile
c = Counter(ips)
print(c)
print(c.most_common(2))

# 3. Gruppieren: pro Rechner eine Liste von Ports
gruppen = defaultdict(list)
for host, port in [("srv01", 22), ("srv02", 22), ("srv01", 80)]:
    gruppen[host].append(port)
print(dict(gruppen))
```

```text
{'10.0.0.5': 3, '10.0.0.7': 2, '10.0.0.9': 1}
Counter({'10.0.0.5': 3, '10.0.0.7': 2, '10.0.0.9': 1})
[('10.0.0.5', 3), ('10.0.0.7', 2)]
{'srv01': [22, 80], 'srv02': [22]}
```

Der Ausdruck `zaehler.get(ip, 0) + 1` liest den bisherigen Zählerstand (oder 0, falls die IP neu ist), addiert eins und schreibt den Wert zurück. `Counter` macht dasselbe und bietet zusätzlich `most_common(n)`, das die häufigsten Einträge als Liste von Tupeln liefert. `defaultdict(list)` legt für jeden neuen Schlüssel automatisch eine leere Liste an, sodass man direkt `append` aufrufen kann.

Typische Fehler: Der Zugriff auf einen nicht vorhandenen Schlüssel mit eckigen Klammern führt zu einem `KeyError`; dagegen hilft `get()` oder vorher `in`. Eine Liste als Schlüssel ist nicht erlaubt, hier hilft ein Tupel. Und der Versuch, ein Dictionary während der Iteration zu verändern, löst einen `RuntimeError` aus.

```python
# FALSCH: Schlüssel, der nicht existiert
server = {"name": "srv01", "port": 22}
print(server["os"])
```

```text
KeyError: 'os'
```

```python
# RICHTIG: mit get und Vorgabewert
server = {"name": "srv01", "port": 22}
print(server.get("os", "unbekannt"))

# Liste als Schlüssel ist verboten, ein Tupel ist erlaubt
position = {("rack1", 3): "srv01"}
print(position[("rack1", 3)])
```

```text
unbekannt
srv01
```

> **Ergänzung aus der Nachrecherche**
>
> **Verständlichere Fehlermeldungen in Python 3.14 (abgerufen aus „What’s New in Python 3.14“).** Python 3.14 verbessert mehrere Fehlermeldungen. Als Beispiel nennt die Dokumentation die Meldung bei einer Liste als Dictionary-Schlüssel: `TypeError: cannot use 'list' as a dict key (unhashable type: 'list')`. Ältere Versionen melden nur `unhashable type: 'list'`, was auch die Beispiele mit Python 3.13 in Lerneinheit 3.2 zeigen. Das Verhalten der Sprache ist gleich; verändert hat sich nur der Hinweistext.

> **Merke**
>
> Ein Dictionary ordnet eindeutige, hashbare Schlüssel Werten zu. Lesen mit `d[k]` scheitert bei fehlendem Schlüssel mit `KeyError`; `d.get(k, vorgabe)` nicht. Durchlaufen Sie Dictionaries mit `items()`, und nutzen Sie `Counter` und `defaultdict` zum Zählen und Gruppieren.

> **Praxisbeispiel**
>
> Dictionaries sind die Standarddatenstruktur der Automatisierung: Ein Server-Inventar, eine Konfiguration, die Antwort einer REST-API (JSON, Modul 7 und 10) oder das Ergebnis eines Monitoringchecks liegt als Dictionary vor. Wer eine JSON-Datei einliest, erhält verschachtelte Dictionaries und Listen, und die Fähigkeit, in solchen Strukturen sicher zu navigieren, ist eine der wichtigsten Alltagsfähigkeiten.

## 3.2 Sets & Datenstrukturen im Vergleich

Ein *Set* (deutsch *Menge*) ist eine ungeordnete Sammlung *eindeutiger* Elemente. Jedes Element kommt höchstens einmal vor; fügt man ein vorhandenes Element erneut hinzu, passiert nichts. Wie bei Dictionary-Schlüsseln müssen die Elemente hashbar sein. Das hat zwei große Vorteile: Die Prüfung `x in menge` ist extrem schnell, und man kann Mengenoperationen wie in der Mathematik ausführen. Ein Set schreibt man mit geschweiften Klammern ohne Doppelpunkt; ein *leeres* Set erzeugt man dagegen mit `set()`, denn `{}` ist ein leeres Dictionary.

```pycon
>>> ports = {22, 80, 443, 80, 22}
>>> ports == {22, 80, 443}
True
>>> len(ports)
3
>>> ports.add(8080)
>>> ports.discard(22)
>>> sorted(ports)
[80, 443, 8080]
>>> type({}), type(set())
(<class 'dict'>, <class 'set'>)
>>> set("hallo") == {"h", "a", "l", "o"}
True
>>> sorted(set([3, 1, 3, 2, 1]))
[1, 2, 3]
```

Aus der Liste `[3, 1, 3, 2, 1]` wird durch `set()` die Menge der eindeutigen Elemente; `sorted()` macht daraus wieder eine geordnete Liste. Das ist der kürzeste Weg, Dubletten zu entfernen. Beachten Sie, dass die Reihenfolge der Elemente in einem Set nicht festgelegt ist, man darf sich also nie auf sie verlassen. Zum Ausgeben nimmt man `sorted()`.

Die eigentliche Stärke von Sets sind die Mengenoperationen. Mit ihnen beantwortet man Fragen wie „Welche Ports sind offen, aber nicht erlaubt?“ in einer Zeile. Die vier Grundoperationen sind die Vereinigung (`|`, alle Elemente beider Mengen), der Schnitt (`&`, nur gemeinsame Elemente), die Differenz (`-`, Elemente der ersten, die nicht in der zweiten sind) und die symmetrische Differenz (`\^`, Elemente, die in genau einer der beiden Mengen vorkommen).

```python
erlaubt = {22, 80, 443}
offen = {22, 80, 3306, 8080}

print("Vereinigung:", sorted(erlaubt | offen))
print("Schnitt:", sorted(erlaubt & offen))
print("Unerlaubt offen:", sorted(offen - erlaubt))
print("Erlaubt, aber geschlossen:", sorted(erlaubt - offen))
print("In genau einer Menge:", sorted(erlaubt ^ offen))

print({22} <= erlaubt, erlaubt.issubset(offen), erlaubt.isdisjoint({25}))

server_a = {"srv01", "srv02", "srv03"}
server_b = {"srv02", "srv03", "srv04"}
print("Nur in A:", sorted(server_a - server_b))
print("Nur in B:", sorted(server_b - server_a))
```

```text
Vereinigung: [22, 80, 443, 3306, 8080]
Schnitt: [22, 80]
Unerlaubt offen: [3306, 8080]
Erlaubt, aber geschlossen: [443]
In genau einer Menge: [443, 3306, 8080]
True False True
Nur in A: ['srv01']
Nur in B: ['srv04']
```

Das Beispiel ist eine kleine Firewall-Prüfung. `offen - erlaubt` liefert genau die Ports, die offen sind, aber nicht offen sein dürften (3306 und 8080), `erlaubt - offen` die Ports, die laut Plan offen sein sollten, aber nicht erreichbar sind (443). Die Vergleiche `<=` und `issubset` prüfen, ob eine Menge in einer anderen enthalten ist; `isdisjoint` prüft, ob zwei Mengen keine gemeinsamen Elemente haben. Auf dieselbe Weise vergleicht man die Server in zwei Inventarlisten oder die Benutzer in zwei Systemen.

Mit Listen, Tupeln, Dictionaries und Sets kennen Sie jetzt die vier eingebauten Sammlungstypen. Die Wahl ist eine Entwurfsentscheidung, und die folgenden Fragen helfen: Ist die Reihenfolge wichtig? Müssen Elemente veränderlich sein? Dürfen Dubletten vorkommen? Will ich über einen Namen zugreifen oder prüfen, ob etwas enthalten ist?

| **Typ**  | **Geordnet**            | **Veränderlich** | **Dubletten**       | **Typischer Einsatz**                         |
|:---------|:------------------------|:-----------------|:--------------------|:----------------------------------------------|
| `list` | ja                      | ja               | erlaubt             | Folgen von Werten: Zeilen, Dateien, Hosts     |
| `tuple` | ja                      | nein             | erlaubt             | feste Datensätze: `(host, port)`                    |
| `dict` | ja (Einfügereihenfolge) | ja               | Schlüssel eindeutig | Zuordnungen: Konfiguration, Inventar, Zähler  |
| `set` | nein                    | ja               | nicht möglich       | Eindeutigkeit, Zugehörigkeit, Mengenvergleich |

Der Unterschied bei der Suche ist beträchtlich. In einer Liste muss Python im schlechtesten Fall jedes Element ansehen, bis es das gesuchte gefunden hat; die Laufzeit wächst mit der Länge der Liste. In einem Set oder Dictionary dagegen führt der Hash direkt zum Ziel, die Laufzeit ist praktisch konstant. Bei kleinen Datenmengen spielt das keine Rolle, bei einer Liste von einer Million IP-Adressen, in der man hunderttausendmal sucht, macht es den Unterschied zwischen Sekunden und Stunden. Das folgende Beispiel misst den Unterschied. Die Zeiten hängen vom Rechner ab; entscheidend ist das Verhältnis.

```python
import time

zahlen_liste = list(range(20000))
zahlen_set = set(zahlen_liste)
gesucht = list(range(19000, 20000))

start = time.perf_counter()
treffer = sum(1 for z in gesucht if z in zahlen_liste)
dauer_liste = time.perf_counter() - start

start = time.perf_counter()
treffer = sum(1 for z in gesucht if z in zahlen_set)
dauer_set = time.perf_counter() - start

print(f"Liste: {dauer_liste:.4f} s, Set: {dauer_set:.6f} s")
print(f"Das Set war etwa {dauer_liste / dauer_set:,.0f}-mal schneller")
```

Auf dem Testrechner (Python 3.13) lag der Unterschied bei etwa dem Zweitausendfachen. Die Zahlen schwanken, doch die Aussage bleibt: Prüfungen auf Zugehörigkeit gehören in ein Set oder ein Dictionary, nicht in eine Liste.

Zu den Stolperfallen zählt die Verwechslung von `{}` und `set()`, wie oben erwähnt, und das Verlassen auf eine Reihenfolge im Set. Zudem können Sets nur hashbare Elemente aufnehmen; eine Liste von Listen lässt sich nicht in ein Set verwandeln, wohl aber eine Liste von Tupeln. Für ein unveränderliches Set gibt es `frozenset`, das auch als Dictionary-Schlüssel erlaubt ist.

```python
print(type({}), type(set()))

try:
    s = {[1, 2], [3, 4]}
except TypeError as fehler:
    print("Fehler:", fehler)

paare = {(1, 2), (3, 4), (1, 2)}
print(len(paare))
fs = frozenset([1, 2, 3])
print(fs, 2 in fs)
```

```text
<class 'dict'> <class 'set'>
Fehler: unhashable type: 'list'
2
frozenset({1, 2, 3}) True
```

> **Merke**
>
> Ein Set speichert eindeutige, hashbare Elemente ungeordnet; es ist ideal zum Entfernen von Dubletten, für schnelle Zugehörigkeitsprüfungen und für Mengenoperationen (`|`, `&`, `-`, `\^`). Ein leeres Set erzeugt man mit `set()`. Für Suchen in großen Datenmengen sind Set und Dictionary der Liste weit überlegen.

> **Praxisbeispiel**
>
> Mengenoperationen lösen viele Vergleichsaufgaben aus dem Alltag: Welche Benutzer stehen im Active Directory, aber nicht im Linux-System? Welche Pakete sind auf Server A installiert und fehlen auf Server B? Welche Ports sind offen, obwohl die Firewall-Richtlinie sie nicht vorsieht? Ein Audit-Skript, das zwei Listen in Sets verwandelt und die Differenzen ausgibt, ist in wenigen Zeilen geschrieben.

## 3.3 Funktionen (Parameter, Rückgabewerte)

Sobald Skripte länger werden, tauchen dieselben Codestücke mehrfach auf. Eine *Funktion* fasst einen Block unter einem Namen zusammen, sodass man ihn beliebig oft aufrufen kann. Das hat drei Vorteile: Man schreibt Code nur einmal (das Prinzip *DRY*, „Don’t repeat yourself“), man gibt einer Idee einen sprechenden Namen, und man kann jede Funktion für sich testen. Mit Funktionen wie `print()` und `len()` arbeiten Sie längst; jetzt schreiben Sie eigene.

Eine Funktion wird mit `def` definiert, gefolgt von Name, Parametern in Klammern und einem Doppelpunkt; der Rumpf ist eingerückt. Mit `return` gibt die Funktion einen Wert an den Aufrufer zurück und endet. Fehlt `return`, gibt sie automatisch `None` zurück. Die Namen in der Klammer der Definition heißen *Parameter*, die konkreten Werte beim Aufruf *Argumente*. Direkt unter der Zeile mit `def` steht idealerweise ein *Docstring*, ein Text in dreifachen Anführungszeichen, der die Funktion beschreibt; Werkzeuge und die Funktion `help()` zeigen ihn an.

```python
def begruessung(name):
    """Gibt eine Begrüßung für name zurück."""
    return f"Hallo {name}!"

def rechteck_flaeche(breite, hoehe):
    """Berechnet die Fläche eines Rechtecks."""
    return breite * hoehe

def melde(text):
    print("INFO:", text)       # kein return -> gibt None zurück

nachricht = begruessung("Anna")
print(nachricht)
print(rechteck_flaeche(3, 4))

ergebnis = melde("Skript gestartet")
print(ergebnis)

help(rechteck_flaeche)
```

```text
Hallo Anna!
12
INFO: Skript gestartet
None
Help on function rechteck_flaeche in module __main__:

rechteck_flaeche(breite, hoehe)
    Berechnet die Fläche eines Rechtecks.

```

Die Funktion `begruessung` nimmt ein Argument entgegen und gibt einen Text zurück. `melde` gibt nichts zurück, sondern macht eine Ausgabe, ihr Rückgabewert ist `None`, wie die vorletzte Zeile zeigt. Beachten Sie den Unterschied zwischen *ausgeben* und *zurückgeben*: `print` zeigt etwas am Bildschirm; `return` übergibt einen Wert an den Code, der die Funktion aufgerufen hat, damit er damit weiterarbeiten kann. Eine Funktion, die rechnet, sollte das Ergebnis zurückgeben und nicht selbst ausgeben, weil man es sonst nirgends weiterverwenden kann.

Beim Aufruf kann man Argumente *positionell* (in der Reihenfolge der Parameter) oder *benannt* (als `name=wert`) übergeben. Parameter können *Vorgabewerte* haben; dann darf man das Argument weglassen. Mit `*args` nimmt eine Funktion beliebig viele positionelle, mit `**kwargs` beliebig viele benannte Argumente entgegen, die als Tupel beziehungsweise Dictionary ankommen. Eine Funktion kann mehrere Werte zurückgeben, indem sie ein Tupel zurückgibt, das der Aufrufer entpackt.

Wir verwandeln den Kern des Umrechners aus Modul 1 in eine wiederverwendbare Funktion, die Byte-Werte lesbar formatiert:

```python
def format_bytes(anzahl, binaer=True, stellen=1):
    """Formatiert eine Byte-Anzahl als lesbaren Text, z. B. '1.5 GiB'."""
    basis = 1024 if binaer else 1000
    einheiten = ["B", "KiB", "MiB", "GiB", "TiB", "PiB"] if binaer \
        else ["B", "kB", "MB", "GB", "TB", "PB"]
    wert = float(anzahl)
    for einheit in einheiten:
        if abs(wert) < basis or einheit == einheiten[-1]:
            break
        wert /= basis
    if einheit == "B":
        return f"{int(wert)} {einheit}"
    return f"{wert:.{stellen}f} {einheit}"

print(format_bytes(512))
print(format_bytes(1536))
print(format_bytes(5 * 1024 ** 3))
print(format_bytes(5 * 1024 ** 3, binaer=False))
print(format_bytes(123456789, stellen=2))
print(format_bytes(0))
```

```text
512 B
1.5 KiB
5.0 GiB
5.4 GB
117.74 MiB
0 B
```

Die Funktion hat einen Pflichtparameter (`anzahl`) und zwei Parameter mit Vorgabewert. Der Aufruf `format_bytes(5 * 1024 ** 3, binaer=False)` übergibt das zweite Argument benannt. In der Schleife wird der Wert so lange durch die Basis geteilt, bis er kleiner als die Basis ist; der `break` verlässt die Schleife, und die Variable `einheit` enthält dann die passende Einheit. Die Anweisung `f"{wert:.{stellen}f}"` zeigt, dass auch die Formatangabe selbst Platzhalter enthalten darf. Das Zeilenumbruchzeichen am Zeilenende (der Backslash) setzt die Anweisung in der nächsten Zeile fort. Solche Hilfsfunktionen sammeln sich bei Admin-Skripten schnell an.

Eine der berüchtigtsten Fallen bei Funktionen sind *veränderliche Vorgabewerte*. Ein Vorgabewert wird einmal beim Definieren der Funktion erzeugt, nicht bei jedem Aufruf. Verwendet man dort eine Liste, teilen sich alle Aufrufe dieselbe Liste.

```python
# FALSCH: die Vorgabeliste wird bei allen Aufrufen wiederverwendet
def sammle_falsch(eintrag, liste=[]):
    liste.append(eintrag)
    return liste

print(sammle_falsch("a"))
print(sammle_falsch("b"))

# RICHTIG: None als Vorgabe, neue Liste im Rumpf
def sammle(eintrag, liste=None):
    if liste is None:
        liste = []
    liste.append(eintrag)
    return liste

print(sammle("a"))
print(sammle("b"))
```

```text
['a']
['a', 'b']
['a']
['b']
```

Im falschen Beispiel enthält die zweite Antwort noch das Element aus dem ersten Aufruf. Die Lösung ist, `None` als Vorgabewert zu verwenden und die Liste im Rumpf neu anzulegen. Weitere typische Fehler sind das Vergessen von `return` (die Funktion liefert dann `None`, und späteres Rechnen scheitert mit einem `TypeError`), die falsche Anzahl von Argumenten (`TypeError: ... missing 1 required positional argument`) und das Einrücken des `return` an der falschen Stelle, zum Beispiel innerhalb der Schleife, wodurch die Funktion nach dem ersten Durchlauf endet.

Ein letztes Werkzeug sind *Lambda-Ausdrücke*: kleine anonyme Funktionen in einer Zeile, nützlich vor allem als Sortierschlüssel. `lambda x: x[1]` ist eine Funktion, die ihr Argument entgegennimmt und dessen zweites Element zurückgibt. Und Skripte schließen üblicherweise mit einem festen Muster ab, das Sie in fast jedem Python-Programm finden: Die Hauptlogik steht in einer Funktion `main()`, und die Zeile `if __name__ == "__main__":` ruft sie nur auf, wenn die Datei direkt gestartet wird, nicht wenn sie von einem anderen Skript importiert wird (Modul 7).

```python
def main():
    messungen = [("srv01", 77.5), ("srv02", 91.2), ("srv03", 45.0)]
    nach_last = sorted(messungen, key=lambda eintrag: eintrag[1], reverse=True)
    for host, last in nach_last:
        print(f"{host}: {last:.1f} %")

if __name__ == "__main__":
    main()
```

```text
srv02: 91.2 %
srv01: 77.5 %
srv03: 45.0 %
```

Die Variable `__name__` enthält beim direkten Start den Text `"__main__"`. Das `main()`-Muster erleichtert später das Testen (Modul 14), weil man die Funktionen einzeln importieren kann, ohne dass das Skript sofort loslegt.

> **Merke**
>
> Eine Funktion kapselt einen Block unter einem Namen. `return` gibt einen Wert zurück (ohne `return` ist es `None`); `print` gibt nur aus. Verwenden Sie keine veränderlichen Vorgabewerte (`[]`, `{}`), sondern `None`. Schreiben Sie einen Docstring und fassen Sie die Hauptlogik in `main()` zusammen.

> **Praxisbeispiel**
>
> In echten Skripten bestehen 90 Prozent des Codes aus kleinen Funktionen wie `pruefe_hostname()`, `lese_konfiguration()` oder `sende_alarm()`, die man einzeln testen und in anderen Skripten wiederverwenden kann. Wer lange Skripte ohne Funktionen schreibt, kopiert irgendwann Codeblöcke und pflegt Fehlerkorrekturen an mehreren Stellen. Eine Funktion `format_bytes()` wie die aus diesem Abschnitt landet in der Praxis oft in einer Bibliothek, die von mehreren Skripten gemeinsam genutzt wird.

## 3.4 Scope, Tracebacks lesen, Debugging

Wo ist eine Variable gültig? Diese Frage beantwortet der *Gültigkeitsbereich* (englisch *scope*). Eine Variable, die innerhalb einer Funktion zugewiesen wird, ist *lokal*: Sie existiert nur, solange die Funktion läuft, und ist von außen nicht sichtbar. Variablen auf der obersten Ebene einer Datei sind *global* und von überall lesbar. Wenn Python einen Namen auflöst, sucht es der Reihe nach im lokalen Bereich der Funktion, in umgebenden Funktionen, im globalen Bereich der Datei und zuletzt bei den eingebauten Namen (die Regel heißt LEGB: Local, Enclosing, Global, Built-in).

```python
standard_port = 22          # global

def zeige():
    print("Standardport:", standard_port)   # liest die globale Variable

def aendere_lokal():
    standard_port = 2222    # neue LOKALE Variable, global bleibt unberührt
    print("In der Funktion:", standard_port)

def aendere_global():
    global standard_port    # erlaubt das Ändern der globalen Variable
    standard_port = 8022

zeige()
aendere_lokal()
print("Danach:", standard_port)
aendere_global()
print("Nach global:", standard_port)

def zaehle():
    summe = 0
    for i in range(3):
        schleifenwert = i * 10
        summe += schleifenwert
    return summe

print(zaehle())
try:
    print(summe)
except NameError as fehler:
    print("Fehler:", fehler)
```

```text
Standardport: 22
In der Funktion: 2222
Danach: 22
Nach global: 8022
30
Fehler: name 'summe' is not defined
```

Die Funktion `zeige` liest die globale Variable, ohne sie zu verändern, das ist erlaubt. In `aendere_lokal` legt die Zuweisung eine neue lokale Variable gleichen Namens an, die die globale nur verdeckt; deshalb bleibt der globale Wert unverändert. Erst die Anweisung `global` ändert die globale Variable. In der Praxis vermeidet man `global`, weil Funktionen, die versteckt auf globale Zustände zugreifen, schwer zu testen und zu verstehen sind. Besser gibt man Werte als Parameter hinein und als Rückgabewert heraus. Die Variable `summe` aus `zaehle` existiert außerhalb der Funktion nicht, der Zugriff endet mit einem `NameError`.

Wenn ein Programm abstürzt, gibt Python einen *Traceback* aus, das Protokoll des Aufrufwegs, der zum Fehler führte. Ein Traceback ist die wichtigste Informationsquelle beim Fehlersuchen, und man liest ihn von unten nach oben. Betrachten Sie dieses kleine Programm: Es liest Einträge der Form `host:port`, und der zweite Eintrag enthält statt der Portnummer einen Namen.

```python
def lese_port(eintrag):
    host, port = eintrag.split(":")
    return int(port)

def pruefe(liste):
    for eintrag in liste:
        print(lese_port(eintrag))

pruefe(["srv01:22", "srv02:ssh"])
```

```text
22
Traceback (most recent call last):
  File "beispiel.py", line 9, in <module>
    pruefe(["srv01:22", "srv02:ssh"])
    ~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "beispiel.py", line 7, in pruefe
    print(lese_port(eintrag))
          ~~~~~~~~~^^^^^^^^^
  File "beispiel.py", line 3, in lese_port
    return int(port)
ValueError: invalid literal for int() with base 10: 'ssh'
```

Die erste Ausgabezeile `22` stammt vom erfolgreichen ersten Eintrag. Dann folgt der Traceback. Lesen Sie ihn so: Die *letzte Zeile* nennt die Fehlerart (`ValueError`) und die Ursache: Der Text `'ssh'` lässt sich nicht in eine Ganzzahl umwandeln. Darüber stehen die *Stationen* des Aufrufwegs, und zwar von der ältesten (oben) bis zur jüngsten (unten, daher „most recent call last“). Jede Station nennt Datei, Zeilennummer, Funktionsname und die betroffene Codezeile. Die unterste Station, `return int(port)` in Zeile 3, ist die Stelle, an der der Fehler tatsächlich auftrat; die darüberliegenden Stationen zeigen, *wer* diese Funktion aufgerufen hat. Die Zeilen mit Tilde und Zirkumflex (`~~^^`), die in neueren Python-Versionen die Stelle in der Zeile markieren, hängen von der Version ab und können bei Ihnen etwas anders aussehen.

Die Fehlerursache liegt hier allerdings nicht in `lese_port`, sondern in den Daten, die die Funktion bekam. Ein Traceback zeigt, wo der Fehler *sichtbar* wurde; die tatsächliche Ursache kann weiter oben im Aufrufweg oder in den Eingabedaten liegen. Deshalb fragt man sich beim Lesen immer: Welcher Wert war an dieser Stelle falsch, und woher kam er?

Manchmal erscheinen zwei Tracebacks nacheinander. Das passiert, wenn ein Fehler abgefangen und durch einen anderen ersetzt wird; Python zeigt dann beide, verbunden durch den Satz „The above exception was the direct cause of the following exception“ (der oben genannte Fehler war die direkte Ursache des folgenden Fehlers). Auch hier gilt: Zuerst die letzte Zeile lesen, dann die erste Ursache am Anfang der Kette beachten.

```python
try:
    int("x")
except ValueError as e:
    raise RuntimeError("Konfiguration ungueltig") from e
```

```text
Traceback (most recent call last):
  File "beispiel.py", line 2, in <module>
    int("x")
    ~~~^^^^^
ValueError: invalid literal for int() with base 10: 'x'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "beispiel.py", line 4, in <module>
    raise RuntimeError("Konfiguration ungueltig") from e
RuntimeError: Konfiguration ungueltig
```

Die folgenden Fehlerarten begegnen Ihnen am häufigsten. Jede nennt sich nach dem, was schiefging, und die Meldung verrät die Ursache.

| **Fehler** | **Typische Ursache**                                  | **Abhilfe**                          |
|:-----------|:------------------------------------------------------|:-------------------------------------|
| `SyntaxError`   | fehlender Doppelpunkt, Klammer oder Anführungszeichen | Zeile und Zeile davor prüfen         |
| `IndentationError`   | uneinheitliche oder fehlende Einrückung               | vier Leerzeichen, keine Tabs mischen |
| `NameError`   | Tippfehler, Variable noch nicht definiert             | Schreibweise und Reihenfolge prüfen  |
| `TypeError`   | falscher Typ oder falsche Argumentzahl                | Typen mit `type()` prüfen            |
| `ValueError`  | richtiger Typ, ungeeigneter Wert                      | Eingabe prüfen, umwandeln            |
| `KeyError`  | Schlüssel nicht im Dictionary                         | `get()` oder vorher `in`      |
| `IndexError`  | Index außerhalb der Liste                             | `len()` prüfen                     |
| `AttributeError`  | Methode oder Attribut existiert nicht                 | Typ des Objekts prüfen, `dir()`    |
| `ZeroDivisionError`  | Division durch null                                   | Nenner vorher prüfen                 |
| `FileNotFoundError`  | Datei oder Verzeichnis fehlt                          | Pfad prüfen (Modul 4)                |

Beim Fehlersuchen (*Debugging*) gibt es eine Reihenfolge, die sich bewährt hat. Erstens: den Fehler reproduzieren und die letzte Zeile des Tracebacks lesen. Zweitens: die genannte Zeile ansehen und prüfen, welche Werte dort beteiligt sind. Das einfachste Hilfsmittel ist eine Ausgabe mit `print()`; besonders praktisch ist die Schreibweise `print(f"{wert=}")`, die Namen und Wert zugleich zeigt. Drittens: wenn das nicht reicht, setzt man einen *Haltepunkt* mit der eingebauten Funktion `breakpoint()`. Sie startet den eingebauten Debugger `pdb`, in dem man das Programm Zeile für Zeile ausführen und Variablen untersuchen kann. Die wichtigsten Befehle sind `p ausdruck` (Wert anzeigen), `n` (nächste Zeile), `s` (in einen Funktionsaufruf hineingehen), `c` (weiterlaufen), `l` (Quelltext anzeigen), `q` (Beenden) und `h` (Hilfe).

```python
def mittelwert(zahlen):
    summe = 0
    for z in zahlen:
        summe += z
    ergebnis = summe / len(zahlen)
    return ergebnis

werte = [4, 8, 15]
breakpoint()
print(mittelwert(werte))
```

Beim Start hält das Programm an der Zeile mit `breakpoint()` und zeigt die Eingabeaufforderung `(Pdb)`. Die folgende Sitzung (die Eingaben stehen hinter `(Pdb)`, die Pfade sind gekürzt) lässt sich so nachvollziehen:

```text
> /home/anna/beispiel.py(9)<module>()
-> breakpoint()
(Pdb) p werte
[4, 8, 15]
(Pdb) n
> /home/anna/beispiel.py(10)<module>()
-> print(mittelwert(werte))
(Pdb) p summe
*** NameError: name 'summe' is not defined
(Pdb) c
9.0
```

Der Befehl `p werte` zeigt die Liste, `n` führt die Zeile mit dem Haltepunkt aus und hält an der nächsten Zeile, und `p summe` liefert einen `NameError`, denn `summe` ist lokal in der Funktion `mittelwert` und auf dieser Ebene nicht sichtbar. Das illustriert noch einmal den Gültigkeitsbereich. Mit `c` läuft das Programm weiter, und die Ausgabe `9.0` erscheint. In Visual Studio Code und anderen Editoren steht dieselbe Funktion mit grafischer Oberfläche zur Verfügung. Mit `PYTHONBREAKPOINT=0` lassen sich alle `breakpoint()`-Aufrufe abschalten, ohne den Quelltext zu ändern.

> **Ergänzung aus der Nachrecherche**
>
> **Debugging und Fehlermeldungen in Python 3.14 (abgerufen aus der Python-Dokumentation).** Die Release-Seite von Python 3.14.0 nennt unter den wichtigsten Neuerungen auch *PEP 768*, eine „Zero-Overhead External Debugger Interface“: Debugger können sich damit an einen laufenden Python-Prozess anhängen, ohne dass dieser dafür langsamer läuft. „What’s New in Python 3.14“ beschreibt außerdem verbesserte Fehlermeldungen, zum Beispiel Vorschläge bei Tippfehlern in Schlüsselwörtern (`Did you mean 'while'?`) und eine eigene Meldung für ein `elif` nach einem `else`-Block. Für den Alltag ändert sich Ihr Vorgehen nicht: Sie lesen weiterhin die letzte Zeile zuerst.

> **Merke**
>
> Lokale Variablen gelten nur in ihrer Funktion; globale lesen Funktionen, ändern sie aber nur mit `global`, was man vermeidet. Einen Traceback liest man von unten: erst Fehlerart und Ursache, dann die Stationen von unten nach oben. Zum Suchen dienen `print(f"{x=}")`, `breakpoint()` und der Debugger `pdb`.

> **Praxisbeispiel**
>
> Im Betrieb sehen Sie Tracebacks vor allem in Logdateien von Cronjobs und Diensten, oft ohne Möglichkeit, das Programm interaktiv anzuhalten. Die Fähigkeit, einen Traceback in dreißig Sekunden zu lesen und die Stelle zu finden, ist deshalb wertvoller als jedes Debugging-Werkzeug. Dabei hilft, in Skripten sprechende Fehlermeldungen zu erzeugen und, wenn ein Fehler abgefangen wird, den Traceback ins Log zu schreiben (Modul 6).

## 3.5 Log-Analyzer v1 + Quiz

Der Meilenstein dieses Moduls ist der erste Log-Analyzer. Er wertet Meldungen des SSH-Dienstes aus, wie sie unter Linux in `/var/log/auth.log` (Debian, Ubuntu) oder `/var/log/secure` (Red Hat) stehen, und beantwortet typische Fragen: Wie viele Anmeldungen schlugen fehl? Von welchen IP-Adressen kamen sie? Welche Benutzernamen wurden ausprobiert? Welche Adresse war am hartnäckigsten? Dateien lesen wir erst im nächsten Modul; die Beispieldaten stehen deshalb als mehrzeiliger Text im Skript.

Eine Zeile sieht so aus (Monat, Tag, Uhrzeit, Rechnername, Dienst mit Prozessnummer, Meldung):

```text
Oct 28 18:01:02 srv01 sshd[812]: Failed password for root from 10.0.0.5 port 22 ssh2
Oct 28 18:03:40 srv01 sshd[830]: Accepted password for anna from 10.0.0.7 port 51514 ssh2
Oct 28 18:05:11 srv01 sshd[842]: Failed password for invalid user admin from 10.0.0.9 port 40022 ssh2
```

Es gibt zwei Formen von Fehlversuchen: `Failed password for root from ...` für einen existierenden Benutzer und `Failed password for invalid user admin from ...` für einen, den es nicht gibt. Der Analyzer soll beide erkennen. Dazu entwerfen wir zuerst eine Funktion, die eine Zeile in ein Dictionary zerlegt. Sie gibt `None` zurück, wenn die Zeile nicht zu den erwarteten Mustern gehört. So bleibt der Rest des Programms frei von Zeilenformat-Einzelheiten, was eine sinnvolle Trennung der Aufgaben ist.

```python
def parse_zeile(zeile):
    """Zerlegt eine sshd-Zeile. Gibt ein Dictionary oder None zurück."""
    teile = zeile.split()
    if len(teile) < 6 or not teile[4].startswith("sshd"):
        return None
    ereignis = None
    if "Failed password" in zeile:
        ereignis = "fehlgeschlagen"
    elif "Accepted password" in zeile or "Accepted publickey" in zeile:
        ereignis = "erfolgreich"
    if ereignis is None or " from " not in zeile:
        return None

    vor_from, _, nach_from = zeile.partition(" from ")
    ip = nach_from.split()[0]
    benutzer = vor_from.split(" for ")[-1].replace("invalid user ", "")
    return {
        "zeit": teile[2],
        "stunde": teile[2][:2],
        "host": teile[3],
        "ereignis": ereignis,
        "benutzer": benutzer,
        "ip": ip,
    }

beispiele = [
    "Oct 28 18:01:02 srv01 sshd[812]: Failed password for root from 10.0.0.5 port 22 ssh2",
    "Oct 28 18:05:11 srv01 sshd[842]: Failed password for invalid user admin from 10.0.0.9 port 40022 ssh2",
    "Oct 28 18:06:00 srv01 CRON[900]: session opened for user root",
]
for z in beispiele:
    print(parse_zeile(z))
```

```text
{'zeit': '18:01:02', 'stunde': '18', 'host': 'srv01', 'ereignis': 'fehlgeschlagen', 'benutzer': 'root', 'ip': '10.0.0.5'}
{'zeit': '18:05:11', 'stunde': '18', 'host': 'srv01', 'ereignis': 'fehlgeschlagen', 'benutzer': 'admin', 'ip': '10.0.0.9'}
None
```

Gehen wir die Funktion durch. Die erste Prüfung verwirft Zeilen, die zu kurz sind oder nicht vom SSH-Dienst stammen (das fünfte Wort beginnt mit `sshd`). Danach wird das Ereignis anhand von Textbausteinen erkannt. Mit `partition(" from ")` wird die Zeile in den Teil davor und den Teil danach geteilt; das erste Wort danach ist die IP-Adresse. Der Benutzername steht nach dem letzten `" for "` im vorderen Teil, und der Zusatz `invalid user` wird entfernt, damit er einheitlich ist. Der Unterstrich `_` ist eine Konvention für einen Wert, den man nicht braucht. Das Ergebnis ist ein Dictionary, mit dem man bequem weiterarbeiten kann. Die dritte Beispielzeile stammt von `CRON` und wird zu `None`. Wie Sie solche Muster später mit regulären Ausdrücken noch robuster erfassen, zeigt Modul 8.

Der zweite Teil des Programms durchläuft alle Zeilen, sammelt die Ergebnisse und erstellt die Auswertung. Hier kommen die Datenstrukturen dieses Moduls zusammen: ein `Counter` für die Häufigkeiten, ein Set für die eindeutigen Adressen, ein Dictionary für die Stunden und die Sortierung mit `most_common`.

```python
from collections import Counter

LOG = """\
Oct 28 18:01:02 srv01 sshd[812]: Failed password for root from 10.0.0.5 port 22 ssh2
Oct 28 18:01:09 srv01 sshd[812]: Failed password for root from 10.0.0.5 port 22 ssh2
Oct 28 18:03:40 srv01 sshd[830]: Accepted password for anna from 10.0.0.7 port 51514 ssh2
Oct 28 18:05:11 srv01 sshd[842]: Failed password for invalid user admin from 10.0.0.9 port 40022 ssh2
Oct 28 18:05:15 srv01 sshd[842]: Failed password for invalid user test from 10.0.0.9 port 40024 ssh2
Oct 28 18:06:00 srv01 CRON[900]: session opened for user root
Oct 28 19:10:21 srv01 sshd[901]: Failed password for root from 10.0.0.5 port 22 ssh2
Oct 28 19:12:00 srv01 sshd[905]: Accepted publickey for bernd from 10.0.0.8 port 51600 ssh2
Oct 28 19:15:44 srv01 sshd[911]: Failed password for anna from 10.0.0.7 port 51520 ssh2
Oct 28 19:20:01 srv01 sshd[920]: Failed password for root from 10.0.0.5 port 22 ssh2
"""

def parse_zeile(zeile):
    teile = zeile.split()
    if len(teile) < 6 or not teile[4].startswith("sshd"):
        return None
    if "Failed password" in zeile:
        ereignis = "fehlgeschlagen"
    elif "Accepted" in zeile:
        ereignis = "erfolgreich"
    else:
        return None
    vor_from, _, nach_from = zeile.partition(" from ")
    return {
        "stunde": teile[2][:2],
        "ereignis": ereignis,
        "benutzer": vor_from.split(" for ")[-1].replace("invalid user ", ""),
        "ip": nach_from.split()[0],
    }

def analysiere(text):
    fehl_ip = Counter()
    fehl_benutzer = Counter()
    pro_stunde = Counter()
    erfolgreich = 0
    ignoriert = 0
    for zeile in text.splitlines():
        eintrag = parse_zeile(zeile)
        if eintrag is None:
            ignoriert += 1
        elif eintrag["ereignis"] == "erfolgreich":
            erfolgreich += 1
        else:
            fehl_ip[eintrag["ip"]] += 1
            fehl_benutzer[eintrag["benutzer"]] += 1
            pro_stunde[eintrag["stunde"]] += 1
    return fehl_ip, fehl_benutzer, pro_stunde, erfolgreich, ignoriert

def main():
    fehl_ip, fehl_benutzer, pro_stunde, erfolgreich, ignoriert = analysiere(LOG)
    gesamt_fehl = sum(fehl_ip.values())

    print("=== Log-Analyzer v1 ===")
    print(f"Fehlgeschlagene Anmeldungen: {gesamt_fehl}")
    print(f"Erfolgreiche Anmeldungen:    {erfolgreich}")
    print(f"Ignorierte Zeilen:           {ignoriert}")
    print(f"Verschiedene Angreifer-IPs:  {len(fehl_ip)}")

    print()
    print("Top IP-Adressen:")
    for ip, anzahl in fehl_ip.most_common(3):
        print(f"  {ip:<12} {anzahl:>3}x")

    print()
    print("Probierte Benutzernamen:")
    for name, anzahl in sorted(fehl_benutzer.items(), key=lambda t: (-t[1], t[0])):
        print(f"  {name:<8} {anzahl}x")

    print()
    print("Fehlversuche pro Stunde:")
    for stunde in sorted(pro_stunde):
        print(f"  {stunde}:00  {'#' * pro_stunde[stunde]} ({pro_stunde[stunde]})")

    verdaechtig = [ip for ip, n in fehl_ip.items() if n >= 3]
    print()
    print("Verdächtig (>= 3 Fehlversuche):", ", ".join(sorted(verdaechtig)) or "keine")

if __name__ == "__main__":
    main()
```

```text
=== Log-Analyzer v1 ===
Fehlgeschlagene Anmeldungen: 7
Erfolgreiche Anmeldungen:    2
Ignorierte Zeilen:           1
Verschiedene Angreifer-IPs:  3

Top IP-Adressen:
  10.0.0.5       4x
  10.0.0.9       2x
  10.0.0.7       1x

Probierte Benutzernamen:
  root     4x
  admin    1x
  anna     1x
  test     1x

Fehlversuche pro Stunde:
  18:00  #### (4)
  19:00  ### (3)

Verdächtig (>= 3 Fehlversuche): 10.0.0.5
```

Die Struktur folgt dem bewährten Muster aus diesem Modul: kleine Funktionen mit einer Aufgabe (`parse_zeile` zerlegt, `analysiere` zählt, `main` gibt aus) und die Absicherung mit `if __name__ == "__main__":`. In `analysiere` stehen drei `Counter`, die jeweils anders gruppieren, und die Funktion gibt fünf Werte als Tupel zurück, die `main` entpackt. Die Sortierung der Benutzernamen verwendet als Schlüssel ein Tupel, `(-anzahl, name)`: Zuerst wird absteigend nach der Anzahl sortiert (durch das Minuszeichen), bei gleicher Anzahl alphabetisch. Die Auswertung nach Stunden bildet ein Balkendiagramm aus Rautenzeichen, ein praktisches Mittel für Konsolenberichte. Die verdächtigen IP-Adressen entstehen mit einer List Comprehension und einem Schwellenwert, den man als Konstante an den Anfang des Skripts stellen würde.

Zu den Grenzen dieser Version: Sie unterscheidet nicht nach Datum (die Stundenwerte vermischen verschiedene Tage), sie liest keine Dateien und sie erkennt nur zwei Meldungsformen. Die Version 2 liest echte Dateien (Modul 4), wertet Zeitfenster aus (Modul 8) und kann mit Fehlern in einzelnen Zeilen umgehen (Modul 6).

> **Praxisbeispiel**
>
> Eine Auswertung dieser Art ist der Einstieg in die Sicherheitsüberwachung: Wer sieht, dass eine einzelne Adresse in einer Stunde Hunderte Anmeldungen erfolglos versucht, kann sie sperren; Werkzeuge wie fail2ban automatisieren genau das. Auch die Zahl der erfolgreichen Anmeldungen von ungewöhnlichen Adressen oder zu ungewöhnlichen Zeiten ist ein wertvoller Hinweis. Ihr Skript zeigt in kleinem Maßstab, wie solche Werkzeuge denken.

Das Quiz zum Modul sind die Übungsaufgaben 1 bis 4, die Aufgaben 5 bis 8 sind Programmieraufgaben rund um Dictionaries, Mengen, Funktionen und den Log-Analyzer.

## Übungsaufgaben zu Modul 03

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Gegeben ist `d = {"a": 1, "b": 2}`. Was liefern `d["a"]`, `d.get("c")`, `d.get("c", 0)`, `"b" in d`, `2 in d` und `len(d)`? Was passiert bei `d["c"]`? Warum ist `{[1, 2]: "x"}` ein Fehler?

**Aufgabe 2.** Gegeben sind `a = {1, 2, 3, 4}` und `b = {3, 4, 5}`. Berechnen Sie `a ` b\|, `a & b`, `a - b`, `b - a` und `a ^ b`. Welche Datenstruktur würden Sie wählen, um (a) eine Reihenfolge von Logzeilen, (b) die eindeutigen IP-Adressen, (c) die Zuordnung Benutzername zu UID und (d) ein festes Paar aus Host und Port zu speichern?

**Aufgabe 3.** Was gibt der folgende Code aus? Begründen Sie jede Zeile mit Hilfe der Begriffe Gültigkeitsbereich, Rückgabewert und veränderlicher Vorgabewert.

```python
zaehler = 0

def erhoehe():
    zaehler = zaehler + 1
    return zaehler

def fuege_hinzu(x, ziel=[]):
    ziel.append(x)
    return ziel

print(fuege_hinzu(1))
print(fuege_hinzu(2))
print(erhoehe())
```

**Aufgabe 4.** Lesen Sie diesen Traceback: In welcher Datei und Zeile ist der Fehler entstanden, welche Funktion hat die fehlerhafte Funktion aufgerufen, welche Fehlerart liegt vor, und was ist vermutlich die Ursache?

```text
Traceback (most recent call last):
  File "audit.py", line 14, in <module>
    bericht(daten)
  File "audit.py", line 9, in bericht
    print(f"{eintrag['host']}: {eintrag['port']}")
KeyError: 'port'
```

**Aufgabe 5.** Schreiben Sie ein Programm mit einem Dictionary, das die Dienste `ssh`, `http`, `https`, `dns` und `smtp` den Standardports 22, 80, 443, 53 und 25 zuordnet. Es soll einen Dienstnamen einlesen und den Port ausgeben, bei unbekanntem Namen die Meldung `Unbekannter Dienst`. Geben Sie am Ende die Dienste aufsteigend nach Port sortiert aus.

**Aufgabe 6.** Zwei Server haben die installierten Pakete `["nginx", "curl", "vim", "git"]` und `["curl", "vim", "postgresql", "git", "htop"]`. Geben Sie mit Mengenoperationen aus: die gemeinsamen Pakete, die nur auf dem ersten und die nur auf dem zweiten Server installierten, und wie viele verschiedene Pakete es insgesamt gibt.

**Aufgabe 7.** Schreiben Sie eine Funktion `auslastung_status(prozent, warn=85, krit=95)`, die für eine Prozentzahl die Texte `OK`, `WARNUNG` oder `KRITISCH` zurückgibt und für Werte außerhalb von 0 bis 100 `None`. Testen Sie sie mit mehreren Werten, auch mit eigenen Schwellen.

**Aufgabe 8.** Erweitern Sie den Log-Analyzer um eine Funktion `top_n(zaehler, n)`, die die häufigsten Einträge eines Counters als formatierte Liste zurückgibt, und um die Auswertung, wie viel Prozent der Fehlversuche von der häufigsten IP-Adresse stammen. Verwenden Sie dazu die Beispieldaten aus Lerneinheit 3.5.

## Lösungen zu Modul 03

**Aufgabe 1.**

```python
d = {"a": 1, "b": 2}
print(d["a"], d.get("c"), d.get("c", 0))
print("b" in d, 2 in d, len(d))
try:
    d["c"]
except KeyError as fehler:
    print("KeyError:", fehler)
```

```text
1 None 0
True False 2
KeyError: 'c'
```

Die Prüfung `2 in d` ist falsch, weil `in` bei Dictionaries nur die Schlüssel betrachtet; 2 ist ein Wert. Der Ausdruck `{[1, 2]: "x"}` ist ein Fehler, weil eine Liste veränderlich und damit nicht hashbar ist; als Schlüssel taugen nur unveränderliche Objekte, etwa ein Tupel `(1, 2)`.

**Aufgabe 2.**

```python
a = {1, 2, 3, 4}
b = {3, 4, 5}
print(sorted(a | b), sorted(a & b), sorted(a - b), sorted(b - a), sorted(a ^ b))
```

```text
[1, 2, 3, 4, 5] [3, 4] [1, 2] [5] [1, 2, 5]
```

Als Datenstrukturen eignen sich: (a) eine Liste, weil die Reihenfolge wichtig ist und Dubletten erlaubt sind; (b) ein Set für eindeutige IP-Adressen; (c) ein Dictionary für die Zuordnung Name zu UID; (d) ein Tupel für das feste Paar `(host, port)`, das sich nicht ändern soll.

**Aufgabe 3.** Der Code gibt `[1]` und `[1, 2]` aus, und der dritte Aufruf löst einen Fehler aus. Die Funktion `fuege_hinzu` verwendet einen veränderlichen Vorgabewert: Die Liste `ziel` wird einmal bei der Definition erzeugt und von beiden Aufrufen weiterverwendet, deshalb enthält die zweite Antwort noch das Element 1. Die Funktion `erhoehe` scheitert mit `UnboundLocalError`: Weil in der Funktion `zaehler` zugewiesen wird, gilt der Name als lokal, und die Lesung `zaehler + 1` greift auf eine noch nicht zugewiesene lokale Variable zu. Abhilfe: den Wert als Parameter übergeben (`def erhoehe(zaehler): return zaehler + 1`).

```python
zaehler = 0

def erhoehe():
    zaehler = zaehler + 1
    return zaehler

erhoehe()
```

```text
UnboundLocalError: cannot access local variable 'zaehler' where it is not associated with a value
```

**Aufgabe 4.** Der Fehler wurde in der Datei `audit.py`, Zeile 9, in der Funktion `bericht` sichtbar; sie wurde in Zeile 14 auf oberster Ebene (`<module>`) mit der Variable `daten` aufgerufen. Die Fehlerart ist ein `KeyError` für den Schlüssel `'port'`: Mindestens ein Eintrag in `daten` hat keinen Schlüssel `port`. Die Ursache liegt also vermutlich in den Daten (ein unvollständiger Eintrag), nicht in Zeile 9; der Zugriff dort sollte entweder `eintrag.get('port', '?')` verwenden oder die Daten vorher prüfen.

**Aufgabe 5.**

```python
ports = {"ssh": 22, "http": 80, "https": 443, "dns": 53, "smtp": 25}

name = input("Dienst: ").strip().lower()
print()
if name in ports:
    print(f"{name} nutzt Port {ports[name]}")
else:
    print("Unbekannter Dienst")

print()
for dienst, port in sorted(ports.items(), key=lambda paar: paar[1]):
    print(f"{port:>4} {dienst}")
```

```text
Dienst:
https nutzt Port 443

  22 ssh
  25 smtp
  53 dns
  80 http
 443 https
```

Die Sortierung nach Port gelingt mit `sorted(ports.items(), key=...)`: Das Dictionary selbst bleibt unverändert, `sorted` liefert eine Liste von Paaren, und der Schlüssel `lambda paar: paar[1]` wählt den Port als Sortierkriterium. Die Alternative `ports.get(name, "Unbekannter Dienst")` ersetzt die `if`-Abfrage durch eine Zeile.

**Aufgabe 6.**

```python
server1 = ["nginx", "curl", "vim", "git"]
server2 = ["curl", "vim", "postgresql", "git", "htop"]
a, b = set(server1), set(server2)

print("Gemeinsam:", sorted(a & b))
print("Nur Server 1:", sorted(a - b))
print("Nur Server 2:", sorted(b - a))
print("Verschiedene Pakete:", len(a | b))
```

```text
Gemeinsam: ['curl', 'git', 'vim']
Nur Server 1: ['nginx']
Nur Server 2: ['htop', 'postgresql']
Verschiedene Pakete: 6
```

**Aufgabe 7.**

```python
def auslastung_status(prozent, warn=85, krit=95):
    """Gibt OK, WARNUNG oder KRITISCH zurück, bei ungültigem Wert None."""
    if prozent < 0 or prozent > 100:
        return None
    if prozent >= krit:
        return "KRITISCH"
    if prozent >= warn:
        return "WARNUNG"
    return "OK"

for wert in [50, 85, 94.9, 95, 100, 101, -1]:
    print(wert, auslastung_status(wert))
print(auslastung_status(75, warn=70, krit=90))
```

```text
50 OK
85 WARNUNG
94.9 WARNUNG
95 KRITISCH
100 KRITISCH
101 None
-1 None
WARNUNG
```

Die Funktion gibt ihr Ergebnis zurück und gibt nichts aus; der Aufrufer entscheidet, was damit geschieht. Das `elif` wird hier durch mehrere `if` mit `return` ersetzt, was gleichwertig ist, weil `return` die Funktion ohnehin beendet. Die Grenzwerte stehen als Parameter mit Vorgabewerten, sodass sich die Funktion für verschiedene Systeme anpassen lässt.

**Aufgabe 8.**

```python
from collections import Counter

fehl_ip = Counter({"10.0.0.5": 4, "10.0.0.9": 2, "10.0.0.7": 1})

def top_n(zaehler, n):
    """Liefert die n häufigsten Einträge als Textzeilen."""
    return [f"{name:<12} {anzahl:>3}x" for name, anzahl in zaehler.most_common(n)]

for zeile in top_n(fehl_ip, 2):
    print(zeile)

gesamt = sum(fehl_ip.values())
ip, anzahl = fehl_ip.most_common(1)[0]
print(f"{ip} verursacht {anzahl / gesamt:.0%} aller Fehlversuche")
```

```text
10.0.0.5       4x
10.0.0.9       2x
10.0.0.5 verursacht 57% aller Fehlversuche
```

Die Funktion `top_n` nutzt eine List Comprehension und gibt Textzeilen zurück statt sie auszugeben. Die Prozentangabe verwendet die Formatangabe `.0%`, die mit 100 multipliziert, ohne Nachkommastellen rundet und ein Prozentzeichen anhängt. Mit den Zahlen aus dem Meilenstein-Skript (4 von 7 Fehlversuchen) ergibt sich 57 Prozent.
