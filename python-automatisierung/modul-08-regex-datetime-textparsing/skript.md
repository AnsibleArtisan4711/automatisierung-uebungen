# Modul 08 - Regex, datetime, Textparsing

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (8.1 bis 8.5), danach folgen „Übungsaufgaben zu Modul 08“ (8 Aufgaben) und „Lösungen zu Modul 08“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Do 17.12.2026 bis Mi 23.12.2026)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul liefert das Handwerkszeug, um Logdateien auszuwerten)

**Meilenstein:** Logparser mit Zeitfenstern

**Lernziele.** Nach diesem Modul können Sie Textmuster mit regulären Ausdrücken beschreiben und mit dem Modul `re` suchen, zerlegen und ersetzen, Datum und Uhrzeit mit `datetime` korrekt lesen, vergleichen und ausgeben (einschließlich Zeitzonen), und beides zu einem Logparser verbinden, der zum Beispiel die fehlgeschlagenen SSH-Anmeldungen pro Stunde zählt. Sie wissen, wo reguläre Ausdrücke angebracht sind und wo einfache String-Methoden genügen.

| **Nr.** | **Lerneinheit**                         | **Datum**     | **Dauer** | **Prio** |
|:--------|:----------------------------------------|:--------------|:----------|:---------|
| 8.1     | Reguläre Ausdrücke (re) Grundlagen      | Do 17.12.2026 | 2 h       | 2        |
| 8.2     | Regex Gruppen & Textparsing             | Fr 18.12.2026 | 2 h       | 2        |
| 8.3     | datetime & Zeitfenster                  | Mo 21.12.2026 | 2 h       | 2        |
| 8.4     | Logparser (SSH-Fehlversuche pro Stunde) | Di 22.12.2026 | 2 h       | 2        |
| 8.5     | Logparser abschließen + Quiz            | Mi 23.12.2026 | 2 h       | 2        |
|         | **Summe**                               |               | **10 h**  |          |

> **Merke**
>
> Die Beispiele dieses Moduls arbeiten mit erfundenen Logzeilen. Die IP-Adressen stammen aus den Bereichen, die für Dokumentation reserviert sind (`192.0.2.0/24`, `198.51.100.0/24` und `203.0.113.0/24`), die Benutzernamen sind ausgedacht. Das Format der Logzeilen entspricht dem, was OpenSSH üblicherweise in `/var/log/auth.log` (Debian, Ubuntu) beziehungsweise im Journal schreibt; die genauen Texte können je nach Version und Distribution leicht abweichen, deshalb prüfen Sie Ihr Muster immer an einer echten Datei Ihres Systems.

## 8.1 Reguläre Ausdrücke (re) Grundlagen

Wer Logdateien auswertet, sucht fast immer nach *Mustern* statt nach festen Texten: eine IP-Adresse, eine Uhrzeit, eine Zahl hinter dem Wort „port“, eine Zeile, die mit dem Datum beginnt. Mit den String-Methoden, die Sie aus Modul 1 kennen, kommt man weit, solange der gesuchte Text feststeht: `"Failed password" in zeile` prüft, ob eine feste Zeichenfolge vorkommt, `zeile.startswith("Dec")` prüft den Anfang, `zeile.split()` zerlegt an Leerzeichen. Sobald aber das Gesuchte nur eine *Form* hat (irgendeine Folge von Ziffern, dann ein Punkt, dann wieder Ziffern), brauchen Sie eine Sprache, um solche Formen zu beschreiben. Diese Sprache sind die *regulären Ausdrücke* (englisch *regular expressions*, kurz *Regex* oder *RE*). Ein regulärer Ausdruck ist ein Text, der ein Muster beschreibt; eine Regex-Engine prüft, ob und wo ein anderer Text zu diesem Muster passt. In Python steckt diese Engine im Modul `re` der Standardbibliothek.

Eine Warnung vorab, die die Python-Dokumentation selbst ausspricht: Nicht jede Textverarbeitung sollte mit regulären Ausdrücken erledigt werden. Wenn Sie eine feste Zeichenfolge suchen oder ersetzen und keine besonderen Fähigkeiten der Regex brauchen, sind die String-Methoden einfacher und, so die Dokumentation, in der Regel schneller. Und es gibt Aufgaben, die man zwar mit einem regulären Ausdruck lösen *kann*, deren Ausdruck aber so kompliziert wird, dass normaler Python-Code besser lesbar ist. Als Grenzfall nennt die Dokumentation ausdrücklich das Auswerten von HTML oder XML: Dafür gibt es Parser-Module, und Regex ist dort „schmerzhaft“. Behalten Sie diese Faustregel im Kopf: erst `in`, `startswith`, `split`, `replace`, und erst wenn das nicht mehr reicht, Regex.

Zum Einstieg dasselbe Problem mit beiden Werkzeugen. Wir wollen aus einer Logzeile die Portnummer holen, die hinter dem Wort `port` steht.

```python
import re

zeile = "Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2"

# Mit String-Methoden: umständlich und anfällig
teile = zeile.split()
position = teile.index("port")
print("split:", teile[position + 1])

# Mit einem regulären Ausdruck
treffer = re.search(r"port (\d+)", zeile)
print("re   :", treffer.group(1))
print(treffer)
```

```text
split: 51234
re   : 51234
<re.Match object; span=(73, 83), match='port 51234'>
```

Die String-Variante funktioniert nur, solange das Wort `port` genau einmal und als eigenes Wort vorkommt; steht es woanders, liefert `index` die erste Fundstelle, und bei einer Zeile ohne `port` stürzt das Programm mit einem `ValueError` ab. Der reguläre Ausdruck `r"port (\d+)"` beschreibt die Form dagegen direkt: das Wort `port`, ein Leerzeichen, und dann *eine oder mehrere Ziffern*, die mit den Klammern als „Gruppe“ gemerkt werden. Die Funktion `re.search` liefert ein *Match-Objekt*, wenn das Muster irgendwo im Text passt, und `None`, wenn nicht. Das Match-Objekt weiß, *was* gepasst hat (`group`), und *wo* (`span`); beim Ausdrucken sehen Sie den Bereich der Fundstelle und den passenden Text. Dass wir `treffer.group(1)` ohne Prüfung aufrufen, ist in diesem Beispiel nur zulässig, weil wir wissen, dass die Zeile passt; im echten Programm prüft man immer erst, ob `treffer` nicht `None` ist (siehe die Fehlerliste weiter unten).

**Der Raw-String.** In diesem Beispiel steht vor dem Muster ein `r`: `r"port (\d+)"`. Das ist ein *Raw-String* (englisch „roher“ Text). In normalen Python-Zeichenketten hat der Rückwärtsschrägstrich eine eigene Bedeutung (`\n` ist ein Zeilenumbruch, `\t` ein Tabulator), aber in regulären Ausdrücken hat er ebenfalls eine Bedeutung (`\d` steht für eine Ziffer, `\b` für eine Wortgrenze). Beide Bedeutungen kollidieren. Die Dokumentation nennt das die „Backslash-Plage“ und empfiehlt als Lösung die Raw-String-Schreibweise: Im Raw-String bleibt jeder Rückwärtsschrägstrich unverändert erhalten, und die Regex-Engine bekommt genau das, was Sie geschrieben haben. Schreiben Sie reguläre Ausdrücke *immer* als Raw-Strings. Wie leicht es sonst schiefgeht, zeigt das nächste Beispiel mit der Wortgrenze `\b`: In einer normalen Zeichenkette ist `\b` das Steuerzeichen „Rückschritt“ (Backspace), nicht die Wortgrenze der Regex.

```python
import re

text = "Das Wort class steht hier, nicht subclass."

print("normal:", re.findall("\bclass\b", text))      # \b ist hier ein Backspace-Zeichen!
print("raw   :", re.findall(r"\bclass\b", text))     # \b ist die Wortgrenze
print(len("\n"), len(r"\n"))
```

```text
normal: []
raw   : ['class']
1 2
```

Der normale String findet nichts (die Liste ist leer), weil er nach dem Zeichen Backspace sucht; der Raw-String findet das eigenständige Wort `class`, aber nicht den Teil von `subclass`. Die letzte Zeile zeigt den Unterschied an der Wurzel: `"\n"` ist ein einziges Zeichen, `r"\n"` sind zwei (Schrägstrich und n). Neuere Python-Versionen warnen außerdem bei unbekannten Escape-Folgen wie `"\d"` in normalen Zeichenketten; die Dokumentation kündigt an, dass daraus irgendwann ein Syntaxfehler wird.

**Die Bausteine eines Musters.** Ein regulärer Ausdruck besteht aus gewöhnlichen Zeichen, die für sich selbst stehen (`port` passt auf `port`), und aus *Metazeichen*, die eine besondere Bedeutung haben. Die wichtigsten Metazeichen fasst die folgende Tabelle zusammen; auf jede Gruppe gehen wir gleich mit Beispielen ein.

| **Baustein**                 | **Bedeutung**                                                                                          |
|:-----------------------------|:-------------------------------------------------------------------------------------------------------|
| `.`                     | ein beliebiges Zeichen außer dem Zeilenumbruch                                                         |
| `[abc]`, `[a-z]`, `[^0-9]` | ein Zeichen aus der Auswahl, aus einem Bereich, beziehungsweise ein Zeichen, das *nicht* dazugehört    |
| `\d`, `\w`, `\s` | Ziffer; Wortzeichen (Buchstabe, Ziffer, Unterstrich); Leerraum (Leerzeichen, Tabulator, Zeilenumbruch) |
| `\D`, `\W`, `\S` | das jeweilige Gegenteil                                                                                |
| `^`, `$`           | Anfang beziehungsweise Ende des Textes (mit dem Flag `MULTILINE`: jeder Zeile)                            |
| `\b`                     | Wortgrenze (die Stelle zwischen einem Wortzeichen und einem Nicht-Wortzeichen)                         |
| `*`, `+`, `?` | null oder mehr, eins oder mehr, null oder eins Wiederholungen des davor Stehenden                      |
| `{3}`, `{2,4}`           | genau drei, zwei bis vier Wiederholungen                                                               |
| `a\|b`                     | Alternative: `a` oder `b`                                                                    |
| `\.`                     | ein echter Punkt (der Rückwärtsschrägstrich nimmt einem Metazeichen die Sonderbedeutung)               |
| `( )`                     | Gruppe (merken, zusammenfassen; siehe 8.2)                                                             |

Die folgenden Beispiele probieren diese Bausteine an kurzen Texten aus. Lesen Sie jede Zeile als Satz: „eine oder mehrere Ziffern“, „ein Zeichen, das keine Ziffer ist“, und prüfen Sie Ihr Verständnis an der Ausgabe.

```python
import re

text = "Server web01 und web02 laufen, db7 nicht; Ports 22, 443 und 8080."

print(re.findall(r"\d+", text))                 # alle Zifferngruppen
print(re.findall(r"web\d\d", text))             # web gefolgt von genau zwei Ziffern
print(re.findall(r"\b\d{2,3}\b", text))         # eigenständige Zahlen mit 2 bis 3 Stellen
print(re.findall(r"[a-z]+\d+", text))           # Kleinbuchstaben, dann Ziffern
print(re.findall(r"(?:web|db)\d+", text))       # Alternative: web oder db
print(re.findall(r"[^a-z ]+", "ab1 cd22"))      # alles außer Kleinbuchstaben und Leerzeichen
```

```text
['01', '02', '7', '22', '443', '8080']
['web01', 'web02']
['22', '443']
['web01', 'web02', 'db7']
['web01', 'web02', 'db7']
['1', '22']
```

Die zweite Zeile liefert beide Servernamen, aber nicht `db7`, weil dort nur eine Ziffer folgt. Die dritte zeigt den Nutzen von `\b`: Ohne die Wortgrenzen würde auch aus `8080` der Teil `808` gefunden; mit den Grenzen passen nur `22` und `443`, während `8080` vier Stellen hat und damit aus der Auswahl fällt (und `01`, `02` gehören zu den Wörtern `web01` und `web02`, die keine Wortgrenze mitten im Wort haben). In der fünften Zeile steht vor der Klammer `?:`; das macht die Gruppe zu einer reinen Zusammenfassung, die ihren Inhalt nicht merkt. Warum das für `findall` wichtig ist, sehen wir in der nächsten Lerneinheit. Die letzte Zeile zeigt eine *negierte* Klasse: `[^a-z ]+` passt auf alles, was weder Kleinbuchstabe noch Leerzeichen ist, hier also auf die beiden Ziffernfolgen.

**Gierig und genügsam.** Ein Quantor wie `*` oder `+` nimmt standardmäßig so viel Text wie möglich; man nennt ihn *gierig* (englisch *greedy*). Das führt zu einer der häufigsten Überraschungen. Soll in `<b>fett</b> und <i>kursiv</i>` das erste Tag gefunden werden, passt `<.*>` auf den *gesamten* Text von der ersten spitzen Klammer bis zur letzten, denn `.*` verbraucht, was es kriegen kann. Mit einem Fragezeichen dahinter (`*?`, `+?`, `??`) wird der Quantor *genügsam* (*non-greedy*) und nimmt so wenig wie möglich.

```python
import re

html = "<b>fett</b> und <i>kursiv</i>"

print("gierig   :", re.search(r"<.*>", html).group())
print("genügsam :", re.search(r"<.*?>", html).group())
print("präzise  :", re.findall(r"<[^>]*>", html))
```

```text
gierig   : <b>fett</b> und <i>kursiv</i>
genügsam : <b>
präzise  : ['<b>', '</b>', '<i>', '</i>']
```

Die dritte Variante ist oft die beste Lösung: Statt mit `.*?` „irgendwas, aber wenig“ zu beschreiben, sagt `[^>]*` genau, was erlaubt ist, nämlich beliebig viele Zeichen, die keine schließende spitze Klammer sind. Solche negierten Klassen sind schneller und eindeutiger als genügsame Quantoren. (Das Beispiel dient nur der Demonstration der Quantoren; wie oben gesagt, wertet man echtes HTML mit einem HTML-Parser aus.)

**Die wichtigsten Funktionen.** Das Modul `re` bietet mehrere Funktionen, die sich vor allem darin unterscheiden, *was* sie mit dem Muster tun. `re.search(muster, text)` sucht die erste Stelle irgendwo im Text. `re.match(muster, text)` prüft dagegen nur den *Anfang* des Textes; die Dokumentation zeigt das mit `re.match("super", "insuperable")`, das `None` liefert, während `re.search` den Treffer an Position 2 findet. `re.fullmatch(muster, text)` verlangt, dass der *gesamte* Text zum Muster passt, und eignet sich deshalb zum Prüfen von Eingaben. `re.findall` liefert alle (nicht überlappenden) Treffer als Liste von Texten, `re.finditer` liefert sie als Iterator von Match-Objekten (mit Positionen, und ohne dass die ganze Liste im Speicher stehen muss). `re.sub(muster, ersatz, text)` ersetzt Treffer, und `re.split(muster, text)` zerlegt den Text an den Treffern.

```python
import re

print(re.match("super", "insuperable"))
print(re.search("super", "insuperable").span())
print(re.fullmatch(r"\d+", "8080"), re.fullmatch(r"\d+", "8080 "))

zeile = "user=anna id=17 user=bernd id=42"
print(re.findall(r"user=\w+", zeile))
for m in re.finditer(r"id=(\d+)", zeile):
    print(m.group(1), "an Position", m.start())

print(re.sub(r"\d+", "#", zeile))
print(re.split(r"\s*[,;]\s*", "a , b;c,   d"))
```

```text
None
(2, 7)
<re.Match object; span=(0, 4), match='8080'> None
['user=anna', 'user=bernd']
17 an Position 10
42 an Position 27
user=anna id=# user=bernd id=#
['a', 'b', 'c', 'd']
```

Beachten Sie den Unterschied in den ersten drei Zeilen: `match` meldet `None`, `search` findet den Treffer, und `fullmatch` lehnt `"8080 "` wegen des Leerzeichens am Ende ab. Das letzte Beispiel zeigt, dass `re.split` Muster als Trenner erlaubt, hier ein Komma oder Semikolon mit beliebig viel Leerraum drumherum, was `str.split` nicht kann.

**Muster vorübersetzen.** Wer dasselbe Muster viele tausend Mal anwendet, zum Beispiel auf jede Zeile einer Logdatei, kompiliert es einmal mit `re.compile` und behält das Ergebnis, ein *Pattern-Objekt*, das dieselben Methoden bietet (`search`, `match`, `findall`, `sub` und so weiter). Das spart Schreibarbeit, gibt dem Muster einen sprechenden Namen und vermeidet, dass das Muster bei jedem Aufruf erneut ausgewertet wird (das Modul `re` hält zwar selbst einen kleinen Zwischenspeicher, aber ein benannter Ausdruck ist auch für den Leser besser). Beim Kompilieren gibt man auch die *Flags* an. `re.IGNORECASE` (kurz `re.I`) ignoriert die Groß- und Kleinschreibung. `re.MULTILINE` (`re.M`) lässt `^` und `$` an jedem Zeilenanfang und -ende passen statt nur am Anfang und Ende des gesamten Textes. `re.DOTALL` (`re.S`) lässt den Punkt auch den Zeilenumbruch treffen. Mehrere Flags kombiniert man mit `|`.

```python
import re

log = """Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2
Dec 17 03:12:51 srv sshd[813]: Accepted publickey for anna from 198.51.100.7 port 40022 ssh2
Dec 17 03:13:02 srv sshd[814]: FAILED password for invalid user test from 203.0.113.5 port 51240 ssh2
"""

muster = re.compile(r"failed password", re.IGNORECASE)
print(len(muster.findall(log)), "Fehlversuche (ohne Rücksicht auf Groß-/Kleinschreibung)")
print(len(re.findall(r"failed password", log)), "ohne IGNORECASE")

print(re.findall(r"^Dec \d+", log))
print(re.findall(r"^Dec \d+", log, re.MULTILINE))
print(re.findall(r"ssh2$", log, re.MULTILINE))
```

```text
2 Fehlversuche (ohne Rücksicht auf Groß-/Kleinschreibung)
0 ohne IGNORECASE
['Dec 17']
['Dec 17', 'Dec 17', 'Dec 17']
['ssh2', 'ssh2', 'ssh2']
```

Ohne `MULTILINE` passt `^` nur am Anfang des gesamten Textes; der Ausdruck findet dann genau einen Treffer statt drei. Dieses Flag benötigen Sie immer dann, wenn Sie einen mehrzeiligen Text (etwa eine ganze Datei, die Sie mit `read_text()` geladen haben) auf einmal durchsuchen. Meist ist es einfacher, eine Datei Zeile für Zeile zu lesen und das Muster auf jede einzelne Zeile anzuwenden; dann braucht man das Flag nicht.

**Ersetzen und Sonderzeichen.** `re.sub` ersetzt nicht nur durch feste Texte. Im Ersatz kann man auf Gruppen verweisen, und man kann statt eines Textes sogar eine Funktion übergeben, die für jeden Treffer den Ersatz berechnet (dazu mehr in 8.2). Ein häufiger Anwendungsfall in der Administration ist das Unkenntlichmachen sensibler Angaben, bevor man Logzeilen weitergibt oder im Ticket zeigt. IP-Adressen können als personenbezogene Daten gelten; ob und wann das im Einzelfall zutrifft, ist eine rechtliche Frage, die hier nicht beantwortet wird (keine Rechtsberatung). Technisch genügt dafür ein einfacher Ausdruck.

Muss ein Text, den ein Benutzer eingibt, *wörtlich* im Muster vorkommen, dann setzt man ihn nicht ungeprüft ein, denn er könnte Metazeichen enthalten (der Punkt in `web01.example.org` würde auf jedes Zeichen passen). Dafür gibt es `re.escape`, das alle Metazeichen entschärft.

```python
import re

zeile = "Failed password for root from 203.0.113.5 port 51234 ssh2"

print(re.sub(r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}", "x.x.x.x", zeile))

gesucht = "a.b"
print(re.findall(gesucht, "a.b axb a-b"))              # Punkt passt auf jedes Zeichen
print(re.findall(re.escape(gesucht), "a.b axb a-b"))   # nur der echte Punkt
print(re.escape("web01.example.org (alt)"))
```

```text
Failed password for root from x.x.x.x port 51234 ssh2
['a.b', 'axb', 'a-b']
['a.b']
web01\.example\.org\ \(alt\)
```

Im zweiten Aufruf werden alle drei Wörter gefunden, weil der Punkt in `a.b` als „beliebiges Zeichen“ gilt; mit `re.escape` bleibt nur das echte `a.b` übrig. Die Ausgabe der letzten Zeile zeigt, welche Zeichen entschärft werden. Genaue Details, etwa ob Leerzeichen und Minuszeichen mit einem Rückwärtsschrägstrich versehen werden, hängen von der Python-Version ab; deshalb sollten Sie sich nicht auf das Aussehen des Ergebnisses verlassen, sondern es nur als Muster weiterverwenden.

**Muster prüfen, nicht raten.** Die Versuchung, ein Muster so allgemein zu formulieren, dass es „irgendwie“ passt, ist groß. Das Muster `\d+\.\d+\.\d+\.\d+` findet zwar IP-Adressen, aber auch `999.999.999.999` und die Versionsnummer `1.2.3.4`; es *beschreibt eine Form, nicht eine Bedeutung*. Wenn die Bedeutung zählt, prüft man das Ergebnis mit einem passenden Werkzeug nach. Für IP-Adressen bietet die Standardbibliothek das Modul `ipaddress`, das ungültige Adressen mit einem `ValueError` ablehnt. Die Arbeitsteilung ist sinnvoll: Die Regex findet *Kandidaten*, `ipaddress` entscheidet, ob es eine Adresse ist.

```python
import ipaddress
import re

text = "Gut: 203.0.113.5, Unsinn: 999.999.999.999, Version: 1.2.3.4, IPv6 ist ::1"

kandidaten = re.findall(r"\b\d{1,3}(?:\.\d{1,3}){3}\b", text)
print("Kandidaten:", kandidaten)

gueltige = []
for k in kandidaten:
    try:
        gueltige.append(str(ipaddress.ip_address(k)))
    except ValueError:
        print("verworfen:", k)
print("gültig:", gueltige)
```

```text
Kandidaten: ['203.0.113.5', '999.999.999.999', '1.2.3.4']
verworfen: 999.999.999.999
gültig: ['203.0.113.5', '1.2.3.4']
```

Die Versionsnummer `1.2.3.4` ist eine gültige IPv4-Adresse und kommt deshalb durch; nur der Kontext kann sagen, dass es keine ist. Auch das ist eine Lehre: Eine Regex ist so gut wie der Kontext, den man ihr gibt. In einer Logzeile wie `from 203.0.113.5 port` ist der Kontext (`from` davor, `port` danach) eindeutig, und man würde ihn ins Muster aufnehmen.

**Typische Fehler und Stolperfallen.** Die folgende Übersicht fasst die häufigsten Fehler im Zusammenhang mit regulären Ausdrücken zusammen; jeden haben Sie in diesem Abschnitt bereits kennengelernt.

```python
import re

zeile = "Zeile ohne Portangabe"

# FALSCH: Ergebnis von search() nicht geprüft
try:
    print(re.search(r"port (\d+)", zeile).group(1))
except AttributeError as fehler:
    print("FALSCH :", fehler)

# RICHTIG: erst prüfen
treffer = re.search(r"port (\d+)", zeile)
print("RICHTIG:", treffer.group(1) if treffer else "kein Port gefunden")

# FALSCH: match() sucht nur am Anfang
print("FALSCH :", re.match(r"\d+", "Port 22"))
# RICHTIG: search() sucht überall
print("RICHTIG:", re.search(r"\d+", "Port 22").group())

# FALSCH: der Punkt in der IP-Adresse ist ein Platzhalter
print("FALSCH :", bool(re.fullmatch(r"10.0.0.1", "10x0y0z1")))
# RICHTIG: Punkt entschärfen
print("RICHTIG:", bool(re.fullmatch(r"10\.0\.0\.1", "10x0y0z1")))
```

```text
FALSCH : 'NoneType' object has no attribute 'group'
RICHTIG: kein Port gefunden
FALSCH : None
RICHTIG: 22
FALSCH : True
RICHTIG: False
```

Die Fehlermeldung des ersten Falls lautet in Python 3.13 `'NoneType' object has no attribute 'group'`; sie entsteht, weil `re.search` bei einem Misserfolg `None` liefert, nicht ein leeres Match-Objekt. Daraus folgt eine Gewohnheit: Das Ergebnis von `search`, `match` und `fullmatch` prüfen Sie *immer*, bevor Sie darauf `.group()` aufrufen. Weitere typische Fehler sind das Vergessen des Raw-Strings (siehe oben), der gierige Quantor, der zu viel verschluckt, und der Versuch, mit einer einzigen riesigen Regex alles auf einmal zu lösen. Aus allgemeinem Fachwissen, das hier nicht in einer Quelle nachgeschlagen wurde, sei noch ein Risiko genannt: Verschachtelte Quantoren wie `(a+)+$` können bei ungünstigen Eingaben sehr lange rechnen, weil die Engine unzählige Wege durchprobiert (*katastrophales Backtracking*). Wer Muster auf Eingaben von Fremden anwendet (Webformular, öffentlicher Dienst), sollte einfache Muster mit klaren Grenzen bevorzugen und Eingaben auf eine vernünftige Länge beschränken.

> **Ergänzung aus der Nachrecherche**
>
> **Python-Dokumentation: Regular Expression HOWTO (abgerufen).** Die Seite hält fest: `re.match()` prüft nur den Anfang des Textes, `re.search()` durchsucht den ganzen Text (Beispiel `"insuperable"`); Muster sollen als Raw-Strings geschrieben werden, weil sonst Rückwärtsschrägstriche doppelt maskiert werden müssten (`"\\\\"` für einen einzelnen Schrägstrich), und unbekannte Escape-Folgen in normalen Zeichenketten lösen eine `DeprecationWarning` aus, die künftig ein `SyntaxError` werden soll; Quantoren sind standardmäßig gierig und werden durch ein angehängtes Fragezeichen genügsam; `re.compile` mit dem Flag `re.VERBOSE` erlaubt Leerraum und Kommentare im Muster; String-Methoden sind bei festen Texten einfacher und in der Regel schneller; HTML und XML wertet man mit einem Parser aus, nicht mit Regex. Die Seite behandelt `fullmatch()` nicht; diese Funktion wurde mit Python 3.13 ausprobiert. Quelle: <https://docs.python.org/3/howto/regex.html> (abgerufen wurde die Quelldatei `_sources/howto/regex.rst.txt`).

> **Merke**
>
> Erst String-Methoden, dann Regex. Muster immer als Raw-String schreiben (`r"..."`). `search` sucht überall, `match` nur am Anfang, `fullmatch` verlangt den ganzen Text. Das Ergebnis auf `None` prüfen. Quantoren sind gierig; `[^>]*` ist oft besser als `.*?`. Punkte und andere Metazeichen entschärfen (`\.`, `re.escape`). Eine Regex beschreibt eine *Form*; ob der Inhalt sinnvoll ist (IP-Adresse, Datum), prüft man danach mit einem passenden Werkzeug.

> **Praxisbeispiel**
>
> Reguläre Ausdrücke begegnen Ihnen in der Systemadministration ständig, nicht nur in Python: `grep -E`, `sed`, Filterregeln von Fail2ban und Logrotate, Suchfelder in Log-Werkzeugen, Konfigurationen von Webservern. Wer das Prinzip einmal verstanden hat, kann alle diese Muster lesen, auch wenn sich Einzelheiten der Schreibweise unterscheiden (Python, `grep -E` und `sed` verstehen nicht alle Zeichen gleich; der Unterschied zu `grep -P` (Perl-Syntax) ist ein klassischer Stolperstein, hier nicht nachgeschlagen). In der täglichen Arbeit nutzen Sie `re` vor allem, um Zeilen aus Logdateien zu filtern, Zahlen und Adressen herauszuziehen und Texte zu bereinigen.

## 8.2 Regex Gruppen & Textparsing

In 8.1 haben wir Muster gefunden. Beim *Parsen* (deutsch etwa „Zerlegen“) reicht das nicht: Wir wollen eine Zeile Text in ihre Bestandteile zerlegen und diese Bestandteile einzeln weiterverwenden: aus einer Logzeile den Zeitstempel, den Rechnernamen, das Programm und die Meldung, aus der Meldung den Benutzernamen und die IP-Adresse. Reguläre Ausdrücke erreichen das mit *Gruppen*. Eine Gruppe ist ein Teil des Musters in runden Klammern; die Engine merkt sich, welcher Text zu dieser Klammer gepasst hat, und liefert ihn im Match-Objekt zurück. Gruppen sind der Schlüssel, um aus einem Muster, das nur „passt oder passt nicht“ sagt, ein Werkzeug zu machen, das Daten herauszieht.

**Nummerierte Gruppen.** Die Gruppen werden von links nach rechts nach ihrer öffnenden Klammer durchnummeriert, beginnend bei 1. Die Nummer 0 steht für den gesamten Treffer. Das Match-Objekt bietet `group(n)` für einzelne Gruppen, `groups()` für alle Gruppen als Tupel und `span(n)` für die Position. Das folgende Beispiel zerlegt die Zeitangabe einer Logzeile und die Fehlermeldung eines SSH-Dienstes.

```python
import re

zeile = "Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2"

m = re.search(r"(\d\d):(\d\d):(\d\d)", zeile)
print(m.group(0), "|", m.group(1), m.group(2), m.group(3))
print(m.groups(), m.span(2))

m = re.search(r"for (\w+) from (\S+) port (\d+)", zeile)
benutzer, adresse, port = m.groups()
print(benutzer, adresse, int(port) + 1)
```

```text
03:12:45 | 03 12 45
('03', '12', '45') (10, 12)
root 203.0.113.5 51235
```

Die erste Gruppe fängt die Stunde, die zweite die Minute, die dritte die Sekunde; `group(0)` ist der vollständige Treffer `03:12:45`. Mit `span(2)` erfährt man, an welcher Stelle der Zeile die Minuten stehen. Im zweiten Muster liefert `groups()` ein Tupel aus drei Texten, das man direkt in drei Variablen entpacken kann. Beachten Sie: Alle Gruppen enthalten *Text*; die Portnummer muss mit `int()` in eine Zahl verwandelt werden, bevor man rechnet. Das Muster `\S+` steht für „eine oder mehrere Zeichen, die kein Leerraum sind“ und ist die Standardlösung für „ein Wort ohne Leerzeichen“, etwa eine Adresse.

**Benannte Gruppen.** Nummern werden unübersichtlich, sobald ein Muster mehr als drei oder vier Gruppen enthält; ändert man später das Muster, verschieben sich die Nummern. Deshalb gibt man den Gruppen Namen. Die Schreibweise ist `(?P<name>...)`. Die Dokumentation beschreibt: Benannte Gruppen verhalten sich wie normale Gruppen und tragen zusätzlich einen Namen. Man erreicht sie dann über `m.group("name")`, über `m["name"]` oder alle zusammen über `m.groupdict()`, das ein Dictionary liefert. Das Dictionary ist genau das, was man zur Weiterverarbeitung braucht (siehe Modul 3).

```python
import re

MUSTER = re.compile(r"for (?P<benutzer>\S+) from (?P<adresse>\S+) port (?P<port>\d+)")

zeile = "Failed password for root from 203.0.113.5 port 51234 ssh2"
m = MUSTER.search(zeile)

print(m.group("benutzer"), m["adresse"], m.groupdict())
daten = m.groupdict()
daten["port"] = int(daten["port"])
print(daten)
```

```text
root 203.0.113.5 {'benutzer': 'root', 'adresse': '203.0.113.5', 'port': '51234'}
{'benutzer': 'root', 'adresse': '203.0.113.5', 'port': 51234}
```

Das Ergebnis ist ein Dictionary mit sprechenden Schlüsseln, das man direkt in einen Report, eine JSON-Datei oder eine Datenbank schreiben kann. Die Umwandlung der Portnummer in eine Zahl ist hier ausdrücklich Ihre Aufgabe: Wie bei CSV (Modul 7) wandelt die Regex nichts um.

**Nicht merkende Gruppen und die Falle mit findall.** Manchmal braucht man Klammern nur, um etwas zusammenzufassen, etwa für eine Alternative oder eine Wiederholung, ohne dass der Inhalt gemerkt werden soll. Dafür gibt es die *nicht merkende* Gruppe `(?:...)`. Das ist mehr als Schönheit, denn Gruppen verändern, was `re.findall` zurückgibt. Enthält das Muster *keine* Gruppe, liefert `findall` die vollständigen Treffer. Enthält es genau eine Gruppe, liefert es nur den Inhalt dieser Gruppe. Enthält es mehrere, liefert es eine Liste von Tupeln. Wer das nicht weiß, wundert sich, warum das Muster plötzlich „die Hälfte“ des Textes zurückgibt.

```python
import re

text = "web01.example.org und db02.example.org"

print(re.findall(r"\w+\.example\.org", text))              # keine Gruppe: ganze Treffer
print(re.findall(r"(\w+)\.example\.org", text))            # eine Gruppe: nur der Inhalt
print(re.findall(r"(\w+)\.(example)\.(org)", text))        # mehrere Gruppen: Tupel
print(re.findall(r"\w+(?:\.\w+)+", text))                  # nicht merkend: ganze Treffer
```

```text
['web01.example.org', 'db02.example.org']
['web01', 'db02']
[('web01', 'example', 'org'), ('db02', 'example', 'org')]
['web01.example.org', 'db02.example.org']
```

Im vierten Aufruf erlaubt die nicht merkende Gruppe, die Wiederholung `(?:\.\w+)+` zu schreiben (ein Punkt mit Wort, mindestens einmal), ohne dass `findall` plötzlich nur das letzte Wort liefert. Die Regel für die Praxis: Sobald Sie mit `findall` arbeiten und Klammern nur zum Gruppieren benötigen, schreiben Sie `(?:...)`; wollen Sie dagegen mehrere Teile pro Treffer haben, nutzen Sie `finditer` und Match-Objekte (mit benannten Gruppen), weil diese weniger fehleranfällig sind.

**Optionale Gruppen und `None`.** Ein Teil eines Musters kann optional sein (`?` oder `*`). Passt dieser Teil nicht, hat die zugehörige Gruppe den Wert `None`, nicht einen leeren Text. Ein Beispiel aus dem Alltag: Die Prozess-ID in eckigen Klammern hinter dem Programmnamen einer Syslog-Zeile fehlt bei manchen Programmen (`sshd[812]:` gegenüber `CRON:`). Wer das übersieht, ruft auf `None` eine Methode oder die Funktion `int` auf und erhält einen Fehler.

```python
import re

muster = re.compile(r"(?P<programm>[\w./-]+)(?:\[(?P<pid>\d+)\])?:")

for text in ("sshd[812]: Verbindung", "CRON: erledigt"):
    m = muster.match(text)
    pid = m["pid"]
    print(m["programm"], repr(pid), int(pid) if pid else "keine PID")

print(muster.match("CRON: x").groupdict(default="-"))
```

```text
sshd '812' 812
CRON None keine PID
{'programm': 'CRON', 'pid': '-'}
```

Die Prüfung `if pid` fängt `None` ab. Die Methode `groupdict(default=...)` ersetzt nicht beteiligte Gruppen durch einen Standardwert. Das Muster selbst enthält eine nützliche Konstruktion: Die Gruppe `(?:\[(?P<pid>\d+)\])?` fasst die eckigen Klammern samt Zahl zusammen (die Klammern sind mit `\[` und `\]` entschärft, weil sie sonst eine Zeichenklasse beginnen würden) und macht das Ganze mit `?` optional.

**Gruppen wiederverwenden: Rückverweise.** Ein *Rückverweis* (englisch *backreference*) lässt ein Muster auf den Text zurückgreifen, den eine frühere Gruppe gefunden hat: `\1` steht für „genau das, was Gruppe 1 gefunden hat“, bei benannten Gruppen `(?P=name)`. So findet man Wiederholungen, etwa doppelte Wörter oder öffnende und schließende Tags mit demselben Namen. Im Ersatztext von `re.sub` verweist man mit `\1` oder `\g<name>` auf Gruppen; die Schreibweise `\g<name>` ist eindeutiger, wenn direkt eine Ziffer folgt.

```python
import re

print(re.findall(r"\b(\w+) \1\b", "Das ist ein ein Test und noch noch ein Test"))
print(re.sub(r"(?P<tag>\w+)=(?P<wert>\S+)", r"\g<wert> <- \g<tag>", "host=web01 port=22"))
print(re.sub(r"(\d{2}):(\d{2}):(\d{2})", r"\1h\2m\3s", "um 03:12:45 Uhr"))
```

```text
['ein', 'noch']
web01 <- host 22 <- port
um 03h12m45s Uhr
```

Die erste Zeile findet die doppelten Wörter `ein ein` und `noch noch`, weil `\1` dasselbe Wort verlangt, das die erste Gruppe gefunden hat. Die zweite dreht Schlüssel und Wert um. Die dritte baut aus den drei Zeitgruppen einen neuen Text.

**Ersetzen mit einer Funktion.** Wenn eine feste Vorlage nicht genügt, übergibt man `re.sub` als zweites Argument eine *Funktion*. Sie wird für jeden Treffer mit dem Match-Objekt aufgerufen, und ihr Rückgabewert ersetzt den Treffer. Damit lassen sich Berechnungen einbauen: Einheiten umrechnen, Adressen maskieren, Zahlen formatieren.

```python
import re

def maskiere(m):
    a, b, c, d = m.group(1, 2, 3, 4)
    return f"{a}.{b}.{c}.xxx"

zeile = "from 203.0.113.5 and 198.51.100.77"
print(re.sub(r"(\d+)\.(\d+)\.(\d+)\.(\d+)", maskiere, zeile))

def in_mb(m):
    return f"{int(m.group(1)) / 1024 / 1024:.1f} MB"

print(re.sub(r"(\d+) Bytes", in_mb, "Datei: 5242880 Bytes, Rest: 1048576 Bytes"))
print(re.subn(r"\d+", "#", "a1 b22 c333"))
```

```text
from 203.0.113.xxx and 198.51.100.xxx
Datei: 5.0 MB, Rest: 1.0 MB
('a# b# c#', 3)
```

Die Funktion `maskiere` bekommt für jede Adresse ein eigenes Match-Objekt; `m.group(1, 2, 3, 4)` liefert mehrere Gruppen auf einmal. `re.subn` liefert zusätzlich die Anzahl der Ersetzungen, was für Berichte nützlich ist („17 Adressen maskiert“).

**Lookahead und Lookbehind.** Manchmal soll ein Muster nur dann passen, wenn davor oder danach etwas Bestimmtes steht, ohne dass dieser Kontext mit im Treffer erscheint. Dafür gibt es die *Vorausschau* (englisch *lookahead*) `(?=...)` und die *Rückschau* (*lookbehind*) `(?<=...)`; die Verneinungen heißen `(?!...)` und `(?<!...)`. Diese Konstruktionen „schauen“ nur und verbrauchen keinen Text. Man braucht sie seltener als Gruppen, aber sie lösen manche Aufgabe elegant, etwa „die Zahl vor dem Wort ms“.

```python
import re

text = "Antwort in 35 ms, Timeout nach 3000 ms, Port 8080, 12 Treffer"

print(re.findall(r"\d+(?= ms)", text))         # Zahlen, auf die " ms" folgt
print(re.findall(r"(?<=Port )\d+", text))      # Zahl, vor der "Port " steht
print(re.findall(r"\b\d+\b(?! ms)", text))     # Zahlen, auf die " ms" NICHT folgt
```

```text
['35', '3000']
['8080']
['8080', '12']
```

In allen drei Fällen enthält das Ergebnis nur die Zahl, nicht den Kontext, der sie umgibt. (Die Dokumentation von Python beschreibt diese Konstruktionen ausführlich; hier zählt, dass Sie ihre Existenz kennen und sie nachschlagen können.)

**Lesbare Muster mit `re.VERBOSE`.** Ein Muster, das eine ganze Logzeile zerlegt, wird schnell unlesbar. Mit dem Flag `re.VERBOSE` (kurz `re.X`) darf man das Muster über mehrere Zeilen schreiben und kommentieren. Die Dokumentation beschreibt die Regeln: Leerraum im Muster wird ignoriert, außer in einer Zeichenklasse oder wenn er mit einem Rückwärtsschrägstrich versehen ist, und ein `#` beginnt einen Kommentar, außer in einer Zeichenklasse oder mit Rückwärtsschrägstrich. Das ist die empfohlene Form für alles, was länger als eine Zeile ist. Das folgende Muster zerlegt eine klassische Syslog-Zeile, wie sie in `/var/log/syslog` oder `auth.log` steht: Monat, Tag, Uhrzeit, Rechnername, Programm mit optionaler Prozess-ID und die eigentliche Meldung.

```python
import re

SYSLOG = re.compile(r"""
    ^(?P<monat>[A-Z][a-z]{2})\s+        # Dec
    (?P<tag>\d{1,2})\s+                 # 17 (einstellig mit Leerzeichen aufgefüllt)
    (?P<zeit>\d\d:\d\d:\d\d)\s+         # 03:12:45
    (?P<host>\S+)\s+                    # srv
    (?P<programm>[\w./-]+)              # sshd
    (?:\[(?P<pid>\d+)\])?:\s+           # [812]: (PID optional)
    (?P<meldung>.*)$                    # Rest der Zeile
""", re.VERBOSE)

zeilen = [
    "Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2",
    "Dec  5 14:00:01 srv CRON[1203]: (root) CMD (run-parts /etc/cron.hourly)",
    "Dec  5 14:00:09 srv kernel: [12.345] eth0: link up",
    "diese Zeile hat ein ganz anderes Format",
]
for zeile in zeilen:
    m = SYSLOG.match(zeile)
    if m is None:
        print("NICHT ERKANNT:", zeile)
        continue
    d = m.groupdict()
    print(d["monat"], d["tag"].rjust(2), d["zeit"], d["host"], d["programm"], d["pid"], "|", d["meldung"][:30])
```

```text
Dec 17 03:12:45 srv sshd 812 | Failed password for root from
Dec  5 14:00:01 srv CRON 1203 | (root) CMD (run-parts /etc/cro
Dec  5 14:00:09 srv kernel None | [12.345] eth0: link up
NICHT ERKANNT: diese Zeile hat ein ganz anderes Format
```

Die vierte Zeile wird sauber als „nicht erkannt“ behandelt, statt dass das Programm mit einem Fehler abbricht; das ist die Grundhaltung beim Parsen von Logdateien: Die Daten sind nicht unter Ihrer Kontrolle, also muss jede Zeile eine Chance bekommen, nicht zu passen. Bei der zweiten Zeile sehen Sie, wozu `\s+` zwischen Monat, Tag und Uhrzeit dient: Syslog füllt einstellige Tage mit einem zusätzlichen Leerzeichen auf (`Dec  5`), und ein einzelnes Leerzeichen im Muster würde diese Zeilen nicht erkennen. Bei der dritten Zeile steht keine Prozess-ID hinter dem Programmnamen, also bleibt `pid` leer (`None`). Wie die Meldung im Kopf und die Uhrzeit in einen echten Zeitstempel verwandelt werden, behandeln wir in 8.3.

**Muster aus Teilen bauen.** Große Muster lassen sich in benannte Bausteine zerlegen, die als Python-Variablen definiert und mit f-Strings zusammengesetzt werden. Das ist in Logparsern üblich, wenn mehrere Formate fast gleich aufgebaut sind: Der Kopf der Zeile ist gleich, nur die Meldung unterscheidet sich. Bei `re.VERBOSE` und f-Strings ist allerdings Vorsicht geboten, weil in f-Strings geschweifte Klammern doppelt geschrieben werden müssen (`{{3}}`), wenn sie zu einem Quantor gehören; deshalb halten wir die Bausteine als einfache Strings und setzen sie mit `+` zusammen.

```python
import re

ZEIT = r"(?P<zeit>\d\d:\d\d:\d\d)"
HOST = r"(?P<host>\S+)"
IP = r"(?P<ip>\d{1,3}(?:\.\d{1,3}){3})"

formate = {
    "fehlversuch": re.compile(ZEIT + r" " + HOST + r" .*Failed password for (?:invalid user )?(?P<benutzer>\S+) from " + IP),
    "erfolg": re.compile(ZEIT + r" " + HOST + r" .*Accepted \w+ for (?P<benutzer>\S+) from " + IP),
}

zeilen = [
    "03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2",
    "03:12:50 srv sshd[813]: Failed password for invalid user test from 203.0.113.5 port 51240 ssh2",
    "03:13:02 srv sshd[814]: Accepted publickey for anna from 198.51.100.7 port 40022 ssh2",
]
for zeile in zeilen:
    for art, muster in formate.items():
        m = muster.search(zeile)
        if m:
            print(art, m["benutzer"], m["ip"])
```

```text
fehlversuch root 203.0.113.5
fehlversuch test 203.0.113.5
erfolg anna 198.51.100.7
```

Der Teil `(?:invalid user )?` macht den Zusatz `invalid user` optional, so dass dasselbe Muster beide Varianten der Fehlermeldung erkennt. Der Benutzername `test` wird in beiden Fällen als `benutzer` geliefert. Bausteine wie `IP` lassen sich in mehreren Mustern wiederverwenden, und ändert sich die Form einer Adresse, ändert man sie an einer einzigen Stelle.

**Schlüssel-Wert-Texte und Zugriffsprotokolle.** Zwei Formate begegnen Ihnen beim Parsen besonders oft. Das erste sind Zeilen aus *Schlüssel*`=`*Wert*-Paaren, wie sie viele Dienste und Firewalls schreiben (auch Werte in Anführungszeichen, die Leerzeichen enthalten dürfen). Das zweite sind Zugriffsprotokolle von Webservern im sogenannten *Combined Log Format*, bei dem jedes Feld an seiner Stelle steht: Adresse, zwei Platzhalter, Zeit in eckigen Klammern, die Anfrage in Anführungszeichen, Statuscode, Größe, Referrer und Browserkennung. Die Details dieses Formats sind je nach Server einstellbar (nginx und Apache erlauben eigene Formate); das folgende Muster passt auf die verbreitete Standardform, und Sie sollten es immer an Ihrer eigenen Konfiguration prüfen.

```python
import re

# 1. Schluessel=Wert, Werte mit Leerzeichen in Anführungszeichen
kv = 'action=drop src=203.0.113.5 dst=192.0.2.10 proto=tcp msg="Verbindung abgelehnt" len=60'
paare = re.findall(r'(\w+)=("[^"]*"|\S+)', kv)
daten = {k: v.strip('"') for k, v in paare}
print(daten)

# 2. Combined Log Format (Standardform von nginx und Apache)
ZUGRIFF = re.compile(r'''
    ^(?P<ip>\S+)\s+\S+\s+(?P<benutzer>\S+)\s+
    \[(?P<zeit>[^\]]+)\]\s+
    "(?P<methode>[A-Z]+)\s+(?P<pfad>\S+)\s+(?P<protokoll>[^"]+)"\s+
    (?P<status>\d{3})\s+(?P<groesse>\d+|-)\s+
    "(?P<referrer>[^"]*)"\s+"(?P<agent>[^"]*)"$
''', re.VERBOSE)

zeile = '203.0.113.5 - - [17/Dec/2026:03:12:45 +0100] "GET /index.html HTTP/1.1" 200 5120 "-" "curl/8.5.0"'
m = ZUGRIFF.match(zeile)
d = m.groupdict()
d["status"] = int(d["status"])
d["groesse"] = 0 if d["groesse"] == "-" else int(d["groesse"])
print(d["ip"], d["zeit"], d["methode"], d["pfad"], d["status"], d["groesse"], d["agent"])
```

```text
{'action': 'drop', 'src': '203.0.113.5', 'dst': '192.0.2.10', 'proto': 'tcp', 'msg': 'Verbindung abgelehnt', 'len': '60'}
203.0.113.5 17/Dec/2026:03:12:45 +0100 GET /index.html 200 5120 curl/8.5.0
```

Das Muster für die Schlüssel-Wert-Paare unterscheidet zwei Möglichkeiten für den Wert: entweder ein Text in Anführungszeichen (`"[^"]*"`, beliebige Zeichen außer dem schließenden Anführungszeichen) oder ein Wort ohne Leerraum (`\S+`). Die Reihenfolge der Alternativen ist wichtig, weil die Engine die erste passende nimmt; stünde `\S+` vorn, würde bei `msg="Verbindung abgelehnt"` nur `"Verbindung` gefunden. Beim Zugriffsprotokoll werden die Felder mit Mustern abgegrenzt, die genau beschreiben, was erlaubt ist: `[^\]]+` (alles außer einer schließenden eckigen Klammer) für die Zeit, `[^"]+` für das Protokoll. Die Größe kann `-` sein, wenn keine Daten übertragen wurden, deshalb die Alternative `\d+`-\|; die Umwandlung in eine Zahl berücksichtigt das. Auch hier gilt: Aus Text wird erst durch `int()` eine Zahl.

**Eine ganze Datei parsen.** Zum Abschluss fassen wir die Technik zu einem kleinen, robusten Parser zusammen: Er liest Zeile für Zeile (ohne die ganze Datei in den Speicher zu laden), wendet das Muster an, übergibt gültige Zeilen als Dictionaries und zählt die Zeilen, die nicht passen, damit man erkennt, wenn sich das Format ändert. Die Funktion ist ein *Generator* (sie benutzt `yield`): Sie liefert ein Ergebnis nach dem anderen und rechnet erst weiter, wenn das nächste verlangt wird. Das ist die Standardform für das Verarbeiten großer Dateien.

```python
import io
import re

ZEILE = re.compile(r"^(?P<monat>\w{3})\s+(?P<tag>\d+)\s+(?P<zeit>\S+)\s+(?P<host>\S+)\s+(?P<programm>[^\s:\[]+)(?:\[\d+\])?:\s+(?P<meldung>.*)$")

def parse(datei, unbekannt):
    """Liefert Dictionaries; zaehlt nicht erkannte Zeilen in der Liste 'unbekannt'."""
    for nummer, zeile in enumerate(datei, start=1):
        zeile = zeile.rstrip("\n")
        if not zeile.strip():
            continue                      # Leerzeilen sind kein Fehler
        m = ZEILE.match(zeile)
        if m is None:
            unbekannt.append(nummer)
            continue
        yield m.groupdict()

beispiel = io.StringIO(
    "Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2\n"
    "\n"
    "kaputte Zeile\n"
    "Dec 17 03:13:02 srv sshd[814]: Accepted publickey for anna from 198.51.100.7 port 40022 ssh2\n"
    "Dec 17 03:14:00 srv CRON[99]: pam_unix(cron:session): session opened\n"
)

unbekannt = []
for eintrag in parse(beispiel, unbekannt):
    print(eintrag["zeit"], eintrag["programm"], "-", eintrag["meldung"][:28])
print("Nicht erkannte Zeilen:", unbekannt)
```

```text
03:12:45 sshd - Failed password for root fro
03:13:02 sshd - Accepted publickey for anna
03:14:00 CRON - pam_unix(cron:session): sess
Nicht erkannte Zeilen: [3]
```

Das Programm überspringt Leerzeilen, meldet die Nummer der kaputten Zeile und verarbeitet alles andere. Dass die Übergabe der Liste `unbekannt` an einen Generator ungewöhnlich aussieht, hat einen Grund: Ein Generator kann nur ein Ergebnis auf einmal zurückgeben, und wir wollen zusätzlich eine Statistik herausbekommen. Alternativ könnte man eine kleine Klasse oder ein Zählerobjekt verwenden. Im Logparser von 8.4 und 8.5 zählen wir die nicht erkannten Zeilen ebenso und geben sie am Ende aus, damit eine geänderte Logzeile auffällt und nicht stillschweigend zu falschen Zahlen führt.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler beim Parsen mit Gruppen haben Sie bereits gesehen: die Tupel-Überraschung bei `findall`, der Zugriff auf eine Gruppe, die `None` ist, und das Vergessen von `int()`. Hinzu kommen zwei Fehler, die man erst beim Arbeiten mit echten Logs bemerkt. Der erste: Das Muster ist *zu streng*. Es passt auf die drei Beispielzeilen, die der Entwickler gesehen hat, aber nicht auf die Variante, die der Dienst bei einem anderen Fehler schreibt. Beispielsweise gibt es bei SSH sowohl `Failed password for root` als auch `Failed password for invalid user test`; ein Muster ohne den optionalen Zusatz überspringt die zweite Variante, und die Statistik ist zu niedrig, ohne dass ein Fehler auftritt. Die Gegenmaßnahme ist das Zählen der nicht erkannten Zeilen und der Vergleich mit einer unabhängigen Zählung, zum Beispiel mit `grep -c "Failed password" auth.log` auf der Kommandozeile.

Der zweite Fehler: Das Muster ist *zu locker*. Ein `.*` irgendwo in der Mitte verschluckt bei unerwarteten Zeilen mehr als beabsichtigt, und ein fehlender Anker (`^` am Anfang) lässt das Muster mitten in der Zeile anspringen. Besonders tückisch ist, dass Teile einer Logzeile von außen beeinflusst werden können: Der Benutzername, den ein Angreifer bei einer SSH-Anmeldung angibt, landet im Log. Ein Muster, das den Benutzernamen mit `.*` statt mit `\S+` liest, kann dadurch zu falschen Ergebnissen gebracht werden. Dieses Risiko ist aus allgemeinem Fachwissen bekannt und wurde hier nicht in einer Quelle nachgeschlagen; die Konsequenz ist, Felder so eng wie möglich zu beschreiben und Protokolldaten nie als vertrauenswürdig zu behandeln.

```python
import re

# FALSCH: findall mit Gruppen liefert nicht, was man erwartet
print("FALSCH :", re.findall(r"(\d+)\.(\d+)", "v1.2 und v3.4"))
# RICHTIG: benannte Gruppen und finditer, dann weiss man, was was ist
print("RICHTIG:", [m.groupdict() for m in re.finditer(r"(?P<major>\d+)\.(?P<minor>\d+)", "v1.2 und v3.4")])

# FALSCH: optionaler Teil nicht beachtet
m = re.search(r"(\w+)(?:\[(\d+)\])?:", "CRON: erledigt")
try:
    print("FALSCH :", int(m.group(2)))
except TypeError as fehler:
    print("FALSCH :", fehler)
# RICHTIG
print("RICHTIG:", int(m.group(2)) if m.group(2) else None)

# FALSCH: zu lockeres Muster verschluckt den Rest der Zeile
zeile = "Failed password for bob from 203.0.113.5 port 22 ssh2 from 192.0.2.99 port 1"
print("FALSCH :", re.search(r"from (.*) port", zeile).group(1))
print("RICHTIG:", re.search(r"from (\S+) port", zeile).group(1))
```

```text
FALSCH : [('1', '2'), ('3', '4')]
RICHTIG: [{'major': '1', 'minor': '2'}, {'major': '3', 'minor': '4'}]
FALSCH : int() argument must be a string, a bytes-like object or a real number, not 'NoneType'
RICHTIG: None
FALSCH : 203.0.113.5 port 22 ssh2 from 192.0.2.99
RICHTIG: 203.0.113.5
```

Im dritten Fall liefert das lockere Muster einen Text mit mehreren Adressen, weil `.*` gierig ist und bis zum *letzten* `port` läuft; das enge Muster `\S+` liefert genau eine Adresse. Gerade solche Fehler fallen nicht durch Programmabstürze auf, sondern durch Zahlen, die niemand nachrechnet. Deshalb gehört zu jedem Parser ein Test mit Beispielzeilen, auch Grenzfällen und kaputten Zeilen, und das Ergebnis wird mit der Erwartung verglichen (siehe Modul 14, pytest).

> **Ergänzung aus der Nachrecherche**
>
> **Python-Dokumentation: Regular Expression HOWTO (abgerufen), Teil Gruppen.** Benannte Gruppen haben die Syntax `(?P<name>...)` und verhalten sich wie gewöhnliche Gruppen, die zusätzlich einen Namen tragen; `m.group('name')` und `m.group(1)` liefern denselben Wert, `m.groupdict()` liefert ein Dictionary der benannten Gruppen; ein benannter Rückverweis hat die Form `(?P=name)`, und im Ersatztext von `re.sub` verweist `\g<name>` auf eine benannte Gruppe; `re.VERBOSE` ignoriert Leerraum im Muster (außer in Zeichenklassen oder nach einem Rückwärtsschrägstrich) und erlaubt Kommentare mit `#`, was die Seite als „weit besser lesbar“ bezeichnet; `findall()` liefert eine Liste von Texten, `finditer()` einen Iterator von Match-Objekten. Dass `findall` bei mehreren Gruppen Tupel liefert und bei genau einer Gruppe nur deren Inhalt, steht auf der abgerufenen Seite nicht; das Verhalten wurde mit Python 3.13 ausprobiert. Quelle: <https://docs.python.org/3/howto/regex.html> (abgerufen wurde die Quelldatei `_sources/howto/regex.rst.txt`). Die Aussagen zu Lookahead und Lookbehind sowie zum Format des Combined Log Format beruhen auf allgemeinem Fachwissen und wurden nicht nachgeschlagen.

> **Merke**
>
> Gruppen holen Teile eines Treffers heraus; benannte Gruppen (`(?P<name>...)`) mit `groupdict()` machen daraus ein Dictionary. Klammern, die nur gruppieren, schreibt man `(?:...)`. Optionale Gruppen können `None` sein; alle Gruppen enthalten Text. `findall` liefert bei Gruppen etwas anderes als ohne Gruppen, daher lieber `finditer`. Lange Muster mit `re.VERBOSE` lesbar halten. Parser müssen mit Zeilen rechnen, die nicht passen, und diese zählen.

> **Praxisbeispiel**
>
> Der Parser für Syslog-Zeilen und das Combined Log Format ist in der Administration das tägliche Werkzeug: Sie ziehen aus `auth.log` die fehlgeschlagenen Anmeldungen, aus dem Webserverprotokoll die Antworten mit Statuscode 500, aus den Firewall-Logs die geblockten Quelladressen. Statt `grep` und `awk` in langen Pipelines verwenden Sie ein kleines Python-Programm, das lesbar, testbar und wiederverwendbar ist und dessen Ergebnis Sie direkt als JSON oder CSV (Modul 7) weitergeben können. Bevor Sie Logdaten an Dritte weitergeben (Support, Ticketsystem), prüfen Sie, ob sie personenbezogene Daten wie Benutzernamen oder IP-Adressen enthalten; ob und wie diese weitergegeben werden dürfen, ist eine Frage des Datenschutzes und nicht dieses Skripts (keine Rechtsberatung).

## 8.3 datetime & Zeitfenster

Ein Logparser, der nur Text zerlegt, weiß nicht, wie spät es war. Erst wenn aus der Zeichenkette `Dec 17 03:12:45` ein echter Zeitpunkt wird, kann man fragen: Wie viele Fehlversuche gab es in der letzten Stunde? Wann war der erste Eintrag? Wie lange lief der Job? Dafür gibt es das Modul `datetime` der Standardbibliothek. Es ist unscheinbar, aber ein Bereich voller Fallstricke, denn Zeit ist kompliziert: Zeitzonen, Sommerzeit, Schaltjahre, unterschiedliche Schreibweisen. In dieser Lerneinheit lernen Sie die Bausteine kennen, und vor allem, an welchen Stellen es regelmäßig schiefgeht.

**Die vier Typen.** Das Modul bietet vier Typen, die man auseinanderhalten sollte. Ein `date` ist ein Kalendertag ohne Uhrzeit (`2026-12-17`). Ein `time` ist eine Uhrzeit ohne Datum (`03:12:45`). Ein `datetime` ist beides zusammen, ein *Zeitpunkt*. Ein `timedelta` schließlich ist eine *Zeitspanne*, zum Beispiel „zwei Stunden“ oder „drei Tage und vier Minuten“. Die Unterscheidung zwischen Zeitpunkt und Zeitspanne ist die wichtigste Idee des Moduls: Man kann zwei Zeitpunkte voneinander abziehen und erhält eine Spanne; man kann zu einem Zeitpunkt eine Spanne addieren und erhält einen neuen Zeitpunkt; man kann aber nicht zwei Zeitpunkte addieren, denn „17. Dezember plus 18. Dezember“ hat keinen Sinn.

```python
from datetime import date, datetime, timedelta

start = datetime(2026, 12, 17, 3, 12, 45)
ende = datetime(2026, 12, 17, 5, 40, 0)

dauer = ende - start
print(dauer, "|", type(dauer).__name__)
print(dauer.total_seconds(), "Sekunden")

print(start + timedelta(hours=2, minutes=30))
print(start - timedelta(days=7))
print(start < ende, ende - start > timedelta(hours=2))

tag = date(2026, 12, 17)
print(tag, tag.weekday(), (date(2026, 12, 31) - tag).days, "Tage bis Silvester")
```

```text
2:27:15 | timedelta
8835.0 Sekunden
2026-12-17 05:42:45
2026-12-10 03:12:45
True True
2026-12-17 3 14 Tage bis Silvester
```

Die Differenz zweier `datetime`-Werte ist ein `timedelta`, und dieses gibt sich als „Stunden:Minuten:Sekunden“ aus. Mit `total_seconds()` bekommen Sie die gesamte Spanne in Sekunden als Zahl, mit der man rechnen kann. Das Ergebnis von `date.weekday()` ist eine Zahl, wobei Montag 0 ist; der 17. Dezember 2026 ist ein Donnerstag, also 3. Zeitpunkte lassen sich mit `<` und `>` vergleichen, und Spannen ebenfalls.

**Zeitspannen richtig lesen.** Ein `timedelta` speichert intern drei Zahlen: Tage, Sekunden und Mikrosekunden. Der Zugriff auf `.seconds` liefert daher nur den Sekundenanteil, der *nach den vollen Tagen* übrig bleibt, nicht die gesamte Spanne. Wer die Dauer in Sekunden braucht, verwendet `total_seconds()`. Das ist ein klassischer Fehler: Eine Spanne von 26 Stunden hat `.days` gleich 1 und `.seconds` gleich 7200 (zwei Stunden), aber `total_seconds()` gleich 93600.

```python
from datetime import timedelta

spanne = timedelta(hours=26)
print("days   :", spanne.days)
print("seconds:", spanne.seconds)               # FALSCH, wenn man die Gesamtdauer will
print("total  :", spanne.total_seconds())       # RICHTIG
print("Stunden:", spanne.total_seconds() / 3600)
print(spanne)
```

```text
days   : 1
seconds: 7200
total  : 93600.0
Stunden: 26.0
1 day, 2:00:00
```

Die Textform `1 day, 2:00:00` ist für Menschen lesbar, und das ist der Grund, warum die Ausgabe von `print(spanne)` nicht zum Rechnen taugt: Wandeln Sie nie diesen Text zurück, sondern rechnen Sie mit `total_seconds()`.

**Zeitpunkte ausgeben und einlesen.** Für die Ein- und Ausgabe von Zeitpunkten gibt es zwei Wege. Der erste ist das genormte Textformat *ISO 8601*, das Jahr, Monat, Tag, Uhrzeit und optional die Zeitzone von der größten zur kleinsten Einheit schreibt (`2026-12-17T03:12:45+01:00`); es ist eindeutig, sortiert sich als Text in der richtigen Reihenfolge und wird von nahezu allen Systemen verstanden. Mit `isoformat()` schreibt man es, mit `datetime.fromisoformat()` liest man es. Der zweite Weg sind selbst definierte Formate: `strftime(format)` gibt einen Zeitpunkt nach einem Formatmuster aus (englisch *string format time*), `strptime(text, format)` liest einen Text nach einem Formatmuster ein (*string parse time*). Das Formatmuster besteht aus Platzhaltern mit Prozentzeichen. Die wichtigsten zeigt die Tabelle.

| **Code**                        | **Bedeutung**                     | **Beispiel** |
|:--------------------------------|:----------------------------------|:-------------|
| `%Y`                       | Jahr vierstellig                  | 2026         |
| `%m`, `%d`            | Monat, Tag zweistellig            | 12, 17       |
| `%H`, `%M`, `%S` | Stunde (24 h), Minute, Sekunde    | 03, 12, 45   |
| `%f`                       | Mikrosekunden                     | 123456       |
| `%b`, `%a`            | Monats-, Wochentagsname abgekürzt | Dec, Thu     |
| `%z`                       | Abstand zu UTC                    | +0100        |
| `%j`                       | Tag im Jahr                       | 351          |

```python
from datetime import datetime

# Ausgabe
zeitpunkt = datetime(2026, 12, 17, 3, 12, 45)
print(zeitpunkt.isoformat())
print(zeitpunkt.strftime("%d.%m.%Y %H:%M"))
print(zeitpunkt.strftime("%A, %b %d"))

# Einlesen: verschiedene Quellen, ein Ergebnis
print(datetime.fromisoformat("2026-12-17T03:12:45"))
print(datetime.strptime("17/12/2026 03:12", "%d/%m/%Y %H:%M"))
print(datetime.strptime("17/Dec/2026:03:12:45 +0100", "%d/%b/%Y:%H:%M:%S %z"))

# Das Muster muss genau passen
try:
    datetime.strptime("2026-12-17", "%d.%m.%Y")
except ValueError as fehler:
    print("Fehler:", fehler)
```

```text
2026-12-17T03:12:45
17.12.2026 03:12
Thursday, Dec 17
2026-12-17 03:12:45
2026-12-17 03:12:00
2026-12-17 03:12:45+01:00
Fehler: time data '2026-12-17' does not match format '%d.%m.%Y'
```

Das dritte `strptime` liest die Zeitangabe eines Webserver-Zugriffsprotokolls aus 8.2 samt dem Abstand zu UTC. Das Ergebnis trägt den Zusatz `+01:00`; was das bedeutet, erklärt der nächste Abschnitt. Der letzte Fall zeigt die Fehlermeldung bei einem Muster, das nicht zum Text passt: ein `ValueError` mit einer Meldung, die Text und Format nennt. Beim Verarbeiten von Logdateien ist dieser Fehler Alltag, und man fängt ihn je nach Zweck ab und zählt die Zeile als nicht erkannt.

Zwei Eigenheiten von `%b` und `%a` sollten Sie kennen: Die Namen von Monaten und Wochentagen hängen von der *Spracheinstellung* (Locale) des Programms ab. In der Standardeinstellung (die in Skripten, Cron-Jobs und Containern oft `C` oder `POSIX` ist) sind es englische Namen wie `Dec`, bei gesetzter deutscher Locale könnten es `Dez` sein. Logdateien von Linux-Diensten sind in der Regel englisch (`Dec`); wenn ein Programm `strptime` dennoch in einer deutschen Umgebung ausführt, kann das Lesen von `Dec` fehlschlagen. Das ist allgemeines Fachwissen und wurde hier nicht in einer Quelle nachgeschlagen; wer sicher gehen will, schreibt in kritischen Programmen eine eigene Zuordnung von Monatsnamen zu Zahlen und unabhängig von der Locale (siehe Übungsaufgabe 6).

**Naiv oder zeitzonenbewusst.** Ein `datetime` kann *naiv* sein (ohne Angabe einer Zeitzone) oder *zeitzonenbewusst* (englisch *aware*). Ein naiver Zeitpunkt wie `2026-12-17 03:12:45` sagt nicht, ob er in Wien, in New York oder in UTC gemeint ist. Das ist bequem, solange alles auf demselben Rechner im selben Bereich stattfindet, und gefährlich, sobald Daten von mehreren Rechnern oder aus verschiedenen Zeiträumen kommen. Ein zeitzonenbewusster Zeitpunkt trägt zusätzlich einen Abstand zur Weltzeit UTC (*Coordinated Universal Time*) in seinem Attribut `tzinfo`. Python lässt Vergleiche und Differenzen zwischen naiven und bewussten Werten *absichtlich nicht* zu: Es löst einen `TypeError` aus, statt zu raten.

```python
from datetime import datetime, timedelta, timezone

naiv = datetime(2026, 12, 17, 3, 0)
bewusst = datetime(2026, 12, 17, 3, 0, tzinfo=timezone.utc)

print(naiv.tzinfo, "|", bewusst.tzinfo)
try:
    print(naiv < bewusst)
except TypeError as fehler:
    print("TypeError:", fehler)

# Zwei bewusste Zeitpunkte lassen sich vergleichen, auch in verschiedenen Zeitzonen
wien_winter = timezone(timedelta(hours=1))
a = datetime(2026, 12, 17, 4, 0, tzinfo=wien_winter)     # 04:00 in UTC+1
b = datetime(2026, 12, 17, 3, 0, tzinfo=timezone.utc)    # 03:00 in UTC
print(a == b, a - b)
```

```text
None | UTC
TypeError: can't compare offset-naive and offset-aware datetimes
True 0:00:00
```

Die beiden letzten Zeitpunkte sind *dieselbe Weltzeit*, nur anders beschriftet: 04:00 Uhr in der Zeitzone UTC+1 ist 03:00 Uhr UTC. Python rechnet beim Vergleich bewusster Zeitpunkte auf eine gemeinsame Grundlage um, deshalb ist das Ergebnis wahr und die Differenz null.

Daraus folgt eine Empfehlung, die in der Praxis viele Fehler verhindert: *Rechnen Sie intern mit UTC* und wandeln Sie nur bei der Ein- und Ausgabe in die Ortszeit um. Den aktuellen Zeitpunkt in UTC erhalten Sie mit `datetime.now(timezone.utc)`. Die ältere Funktion `datetime.utcnow()` liefert dagegen einen *naiven* Wert, der trotz des Namens keine Zeitzone trägt; deshalb ist sie seit Python 3.12 als veraltet gekennzeichnet. Der Autor Miguel Grinberg fasst das so zusammen, dass solche naiven Werte bei Methoden wie `timestamp()` als lokale Zeit des Rechners behandelt werden und dadurch falsche Ergebnisse entstehen können; als Ersatz nennt er `datetime.now(timezone.utc)` und für Zeitstempel `datetime.fromtimestamp(ts, timezone.utc)`. Wenn Sie in älteren Beispielen `utcnow()` sehen, ersetzen Sie es entsprechend.

```python
from datetime import datetime, timezone

# Epoch-Sekunden (Unix-Zeit): Sekunden seit dem 1. Januar 1970, 00:00 UTC
ts = 1797477165
zeitpunkt = datetime.fromtimestamp(ts, timezone.utc)
print(zeitpunkt, "|", zeitpunkt.tzinfo)
print(zeitpunkt.timestamp(), zeitpunkt.isoformat())

jetzt = datetime.now(timezone.utc)
print(jetzt.tzinfo, jetzt.utcoffset())
```

```text
2026-12-17 03:12:45+00:00 | UTC
1797477165.0 2026-12-17T03:12:45+00:00
UTC 0:00:00
```

Die *Unix-Zeit* (englisch *epoch time*) zählt die Sekunden seit dem 1. 1. 1970 um 00:00 Uhr UTC; sie ist ein praktisches Format, um Zeitpunkte als einzelne Zahl zu speichern (in Dateisystemen, Datenbanken und vielen APIs). Mit `fromtimestamp(ts, timezone.utc)` erhalten Sie daraus einen bewussten Zeitpunkt, und `timestamp()` geht den Weg zurück. Die Zeile zum aktuellen Zeitpunkt gibt bewusst nicht die Uhrzeit aus, weil sie bei jedem Lauf anders wäre; sie zeigt nur, dass `tzinfo` gesetzt ist und der Abstand zu UTC null beträgt.

**Echte Zeitzonen mit `zoneinfo`.** Eine Zeitzone wie `timezone(timedelta(hours=1))` ist ein fester Abstand zu UTC. Die Ortszeit in Wien ist aber im Winter UTC+1 und im Sommer UTC+2, und der Wechsel findet an Tagen statt, die politisch festgelegt sind und sich ändern können. Für solche Regeln gibt es seit Python 3.9 das Modul `zoneinfo` mit der *IANA-Zeitzonendatenbank*: `ZoneInfo("Europe/Vienna")` liefert eine Zeitzone, die die Sommerzeit kennt. Laut der Python-Dokumentation liest `ZoneInfo` die Daten aus der Zeitzonendatenbank des Betriebssystems, und wenn es dort keine gibt, aus dem Paket `tzdata` von PyPI. Das ist vor allem unter Windows wichtig, wo es im Allgemeinen keine solche Datenbank gibt: Projekte, die auch dort laufen sollen, tragen das Paket `tzdata` in ihre Abhängigkeiten ein (Modul 7, `requirements.txt`), sonst löst jeder Aufruf von `ZoneInfo` den Fehler `ZoneInfoNotFoundError` aus. Unter Linux sind die Daten in der Regel vorhanden, sodass die Skripte aus diesem Modul dort ohne Zusatz laufen.

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

wien = ZoneInfo("Europe/Vienna")

winter = datetime(2026, 12, 17, 3, 12, 45, tzinfo=wien)
sommer = datetime(2026, 7, 17, 3, 12, 45, tzinfo=wien)
print(winter.isoformat(), winter.tzname())
print(sommer.isoformat(), sommer.tzname())

# Von UTC in die Ortszeit umrechnen
utc = datetime(2026, 12, 17, 3, 12, 45, tzinfo=timezone.utc)
print(utc.astimezone(wien).isoformat())

# Falle: Ein "Tag später" ist in der Ortszeit nicht immer 24 Stunden später
vor = datetime(2026, 3, 28, 12, 0, tzinfo=wien)
nach = vor + timedelta(days=1)                      # Sommerzeit beginnt am 29. März 2026
print(nach.isoformat())
print("Ortszeit-Differenz:", nach - vor)
print("Echte Differenz   :", nach.astimezone(timezone.utc) - vor.astimezone(timezone.utc))
```

```text
2026-12-17T03:12:45+01:00 CET
2026-07-17T03:12:45+02:00 CEST
2026-12-17T04:12:45+01:00
2026-03-29T12:00:00+02:00
Ortszeit-Differenz: 1 day, 0:00:00
Echte Differenz   : 23:00:00
```

In der Zeile für den Winter steht `+01:00` (`CET`, Mitteleuropäische Zeit), im Sommer `+02:00` (`CEST`), obwohl wir beide Male dieselbe Zeitzone `Europe/Vienna` verwendet haben. Die Umrechnung von UTC nach Wien addiert im Dezember eine Stunde. Die letzte Gruppe zeigt die berühmteste Falle der Zeitrechnung: Beim Rechnen innerhalb derselben Zeitzone behandelt Python eine Spanne von einem Tag als „gleiche Uhrzeit am nächsten Kalendertag“. Die Differenz der Ortszeiten ist deshalb genau ein Tag, die echte verstrichene Zeit beträgt aber nur 23 Stunden, weil in der Nacht vom 28. auf den 29. März 2026 die Uhr eine Stunde vorgestellt wurde. Wenn es um *verstrichene Zeit* geht, zum Beispiel für Zeitfenster („die letzten 24 Stunden“), rechnen Sie deshalb in UTC.

Zwei weitere Sonderfälle ergeben sich aus der Zeitumstellung: Es gibt Uhrzeiten, die *nicht existieren* (am 29. März 2026 springt die Uhr von 02:00 auf 03:00 Uhr), und Uhrzeiten, die *zweimal vorkommen* (am 25. Oktober 2026 läuft die Uhr von 03:00 auf 02:00 zurück, sodass 02:30 Uhr zweimal erscheint). Für letztere kennt `datetime` das Attribut `fold`: `fold=0` bezeichnet das erste, `fold=1` das zweite Auftreten. Ohne Angabe gilt `fold=0`. Die Daten, die Sie aus Logdateien lesen, lassen sich dann nicht eindeutig zuordnen, wenn die Logdatei die Ortszeit ohne Abstand zu UTC enthält; Linux-Dienste schreiben deshalb in neueren Konfigurationen zunehmend ISO-8601-Zeitstempel mit Abstand (siehe 8.4).

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

wien = ZoneInfo("Europe/Vienna")

doppelt = datetime(2026, 10, 25, 2, 30, tzinfo=wien)
print("fold=0:", doppelt.utcoffset(), "| fold=1:", doppelt.replace(fold=1).utcoffset())

fehlt = datetime(2026, 3, 29, 2, 30, tzinfo=wien)      # diese Uhrzeit gibt es nicht
print(fehlt.isoformat(), "->", fehlt.astimezone(timezone.utc).isoformat())
```

```text
fold=0: 2:00:00 | fold=1: 1:00:00
2026-03-29T02:30:00+01:00 -> 2026-03-29T01:30:00+00:00
```

Bei der doppelten Uhrzeit unterscheiden sich die beiden Zeitpunkte um eine Stunde im Abstand zu UTC (zwei Stunden in der Sommerzeit, eine im Winter). Bei der nicht existierenden Uhrzeit nimmt Python stillschweigend den Abstand vor der Umstellung an, sodass `02:30` in UTC den Zeitpunkt `01:30` ergibt, der in Wien als `03:30` Sommerzeit erscheint, nicht als `02:30`. Eine Fehlermeldung erhalten Sie nicht; solche Zeitpunkte muss man selbst vermeiden oder bei der Prüfung von Eingaben erkennen. Für Skripte, die Logs lesen, ist das Wissen wichtig, dass zweimal im Jahr eine Stunde entsteht beziehungsweise verschwindet, und dass UTC diese Probleme nicht kennt.

**Zeitfenster berechnen.** Ein *Zeitfenster* ist ein Zeitraum, der durch Anfang und Ende festgelegt ist: „die letzten 24 Stunden“, „gestern“, „von 00:00 bis 06:00“. Mit dem, was Sie bisher gelernt haben, lässt sich jedes Fenster berechnen: aktuelle Zeit minus Spanne ergibt den Anfang; ein Eintrag liegt im Fenster, wenn `anfang <= zeitpunkt < ende` gilt. Die Schreibweise mit `<=` am Anfang und `<` am Ende (ein *halboffenes* Intervall) hat einen Grund: Reiht man mehrere Fenster aneinander (jeweils eine Stunde), gehört jeder Zeitpunkt zu genau einem von ihnen. Würde man beide Enden einschließen, läge ein Eintrag um genau 04:00:00 Uhr in zwei Fenstern.

Eine zweite Standardaufgabe ist das *Abrunden* auf eine Einheit, um Einträge zu Gruppen zusammenzufassen, etwa „alle Einträge dieser Stunde“. Dafür setzt man mit `replace()` die kleineren Einheiten auf null: `zeitpunkt.replace(minute=0, second=0, microsecond=0)`. Der abgerundete Zeitpunkt dient dann als Schlüssel in einem Dictionary oder Zähler.

```python
from collections import Counter
from datetime import datetime, timedelta, timezone

jetzt = datetime(2026, 12, 17, 6, 30, tzinfo=timezone.utc)     # fest, damit das Beispiel reproduzierbar ist
fenster_start = jetzt - timedelta(hours=3)
print("Fenster:", fenster_start.isoformat(), "bis", jetzt.isoformat())

eintraege = [
    datetime(2026, 12, 17, 2, 59, 59, tzinfo=timezone.utc),
    datetime(2026, 12, 17, 3, 30, 0, tzinfo=timezone.utc),
    datetime(2026, 12, 17, 3, 45, 10, tzinfo=timezone.utc),
    datetime(2026, 12, 17, 5, 5, 0, tzinfo=timezone.utc),
    datetime(2026, 12, 17, 6, 31, 0, tzinfo=timezone.utc),
]

im_fenster = [e for e in eintraege if fenster_start <= e < jetzt]
print(len(im_fenster), "Einträge im Fenster")

pro_stunde = Counter(e.replace(minute=0, second=0, microsecond=0) for e in im_fenster)
for stunde in sorted(pro_stunde):
    print(stunde.strftime("%H:%M"), "->", pro_stunde[stunde])
```

```text
Fenster: 2026-12-17T03:30:00+00:00 bis 2026-12-17T06:30:00+00:00
3 Einträge im Fenster
03:00 -> 2
05:00 -> 1
```

Der Eintrag um 02:59:59 liegt vor dem Fensteranfang (03:30:00) und fällt heraus; der Eintrag um genau 03:30:00 gehört dank `<=` noch dazu; der Eintrag um 06:31 liegt nach `jetzt` und ebenfalls nicht im Fenster. Die beiden Einträge der Stunde 03:00 werden zusammengezählt. Das `Counter`-Objekt aus dem Modul `collections` ist ein Dictionary, das Elemente zählt; ein Eintrag, der noch nicht vorkam, hat den Wert null, sodass man nicht vorher prüfen muss, ob der Schlüssel existiert. Den Umgang mit `Counter` vertiefen wir im Logparser von 8.4.

**Syslog-Zeiten ohne Jahr.** Die klassische Syslog-Zeitangabe `Dec 17 03:12:45` enthält weder Jahr noch Zeitzone. Beides muss das Programm ergänzen. Das Jahr ist meist das laufende; aber in den ersten Tagen im Januar können in einer Logdatei noch Einträge vom Dezember stehen, die dann zum *Vorjahr* gehören. Eine übliche Regel: Liegt der so entstandene Zeitpunkt in der Zukunft (nach dem aktuellen Zeitpunkt), gehört er zum Vorjahr. Ein weiterer Stolperstein ist der 29. Februar: Gibt man beim `strptime` kein Jahr an, rechnet Python mit dem Jahr 1900, das kein Schaltjahr ist, und weist `Feb 29` mit `ValueError: day is out of range for month` zurück (Python 3.13 gibt dazu zusätzlich eine `DeprecationWarning` aus, die ankündigt, dass sich dieses Verhalten ändern soll). Die Lösung ist, das Jahr in den zu lesenden Text und in das Format aufzunehmen. Die folgende Funktion fasst beides zusammen und lässt außerdem die Zeitzone des Servers als Parameter zu.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

def syslog_zeit(text, jetzt, zone):
    """Wandelt 'Dec 17 03:12:45' in einen bewussten Zeitpunkt (UTC) um.

    text:  Syslog-Zeit ohne Jahr (der Tag darf einstellig mit Leerzeichen aufgefüllt sein)
    jetzt: aktueller Zeitpunkt (bewusst), um das Jahr zu bestimmen
    zone:  Zeitzone, in der der Server die Zeit schreibt
    """
    text = " ".join(text.split())                      # "Dec  5" -> "Dec 5"
    ortszeit_jetzt = jetzt.astimezone(zone)
    for jahr in (ortszeit_jetzt.year, ortszeit_jetzt.year - 1):
        try:
            zeitpunkt = datetime.strptime(f"{jahr} {text}", "%Y %b %d %H:%M:%S").replace(tzinfo=zone)
        except ValueError:
            continue                                   # z. B. 29. Februar im falschen Jahr
        if zeitpunkt <= jetzt:
            return zeitpunkt.astimezone(timezone.utc)
    raise ValueError(f"Keine gültige Syslog-Zeit: {text!r}")

wien = ZoneInfo("Europe/Vienna")
jetzt = datetime(2027, 1, 3, 9, 0, tzinfo=timezone.utc)

print(syslog_zeit("Dec 31 23:59:59", jetzt, wien))      # gehört zum Vorjahr
print(syslog_zeit("Jan  2 08:15:00", jetzt, wien))      # gehört zum laufenden Jahr
print(syslog_zeit("Feb 29 12:00:00", datetime(2028, 3, 1, tzinfo=timezone.utc), wien))
try:
    syslog_zeit("Feb 29 12:00:00", datetime(2027, 3, 1, tzinfo=timezone.utc), wien)
except ValueError as fehler:
    print("ValueError:", fehler)
```

```text
2026-12-31 22:59:59+00:00
2027-01-02 07:15:00+00:00
2028-02-29 11:00:00+00:00
ValueError: Keine gültige Syslog-Zeit: 'Feb 29 12:00:00'
```

Die Funktion probiert nacheinander das laufende Jahr und das Vorjahr aus und nimmt das erste, in dem der Zeitpunkt gültig ist und nicht in der Zukunft liegt. Der 31. Dezember wird deshalb im Januar 2027 dem Jahr 2026 zugeordnet, und der 29. Februar findet in 2028 sein Schaltjahr; im Jahr 2027 gibt es ihn nicht, und die Funktion meldet das, statt etwas zu erfinden. Beachten Sie die Reihenfolge der Schritte: Erst wird die Ortszeit des Servers als `tzinfo` *angehängt* (`replace(tzinfo=zone)`; die Uhrzeit bleibt unverändert, sie wird dem Zeitpunkt nur als Zone zugeordnet), dann wird mit `astimezone(timezone.utc)` in Weltzeit *umgerechnet* (die Uhrzeit ändert sich). Die beiden Methoden werden oft verwechselt: `replace(tzinfo=...)` sagt, *welche* Zeitzone ein bisher unbekannt beschrifteter Wert hat, `astimezone(...)` rechnet einen bekannten Wert um.

**Typische Fehler und Stolperfallen.** Die folgende Zusammenstellung zeigt die häufigsten Fehler im Umgang mit `datetime` mit je einer falschen und einer richtigen Variante.

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

# 1. .seconds statt total_seconds()
spanne = timedelta(days=2, hours=1)
print("FALSCH :", spanne.seconds, "| RICHTIG:", spanne.total_seconds())

# 2. naive und bewusste Werte mischen
try:
    datetime(2026, 1, 1) - datetime(2026, 1, 1, tzinfo=timezone.utc)
except TypeError:
    print("FALSCH : naiv minus bewusst -> TypeError")
print("RICHTIG:", datetime(2026, 1, 1, tzinfo=timezone.utc) - datetime(2026, 1, 1, tzinfo=timezone.utc))

# 3. replace(tzinfo) und astimezone verwechseln
utc = datetime(2026, 12, 17, 12, 0, tzinfo=timezone.utc)
wien = ZoneInfo("Europe/Vienna")
print("FALSCH :", utc.replace(tzinfo=wien).isoformat())    # gleiche Uhrzeit, anderer Zeitpunkt!
print("RICHTIG:", utc.astimezone(wien).isoformat())        # gleicher Zeitpunkt, andere Uhrzeit

# 4. Monate als Zeitspanne
print("FALSCH :", [a for a in dir(timedelta(1)) if "month" in a], "(timedelta kennt keine Monate)")
```

```text
FALSCH : 3600 | RICHTIG: 176400.0
FALSCH : naiv minus bewusst -> TypeError
RICHTIG: 0:00:00
FALSCH : 2026-12-17T12:00:00+01:00
RICHTIG: 2026-12-17T13:00:00+01:00
FALSCH : [] (timedelta kennt keine Monate)
```

Im dritten Fall bewirkt `replace(tzinfo=wien)` auf einem UTC-Wert, dass 12:00 UTC plötzlich als 12:00 Wiener Zeit gilt, also ein anderer Zeitpunkt wird: Der Wert springt um eine Stunde. Im vierten Fall ist die Liste leer: Ein `timedelta` kennt Tage, Sekunden und Mikrosekunden, aber keine Monate oder Jahre, weil diese unterschiedlich lang sind. Soll „einen Monat zurück“ gerechnet werden, braucht man die Kalenderrechnung selbst oder eine Zusatzbibliothek; für Logfenster genügt fast immer eine feste Spanne in Tagen oder Stunden. Weitere häufige Fehler: das Verwenden von `datetime.utcnow()`, die Annahme, dass jeder Tag 24 Stunden hat (siehe Sommerzeit), und das Vergessen, dass Logzeiten ohne Zeitzone in der Zeitzone des *Servers* geschrieben wurden, nicht in der des Rechners, auf dem man sie auswertet.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentation und Quellen zu `datetime` (abgerufen).** *Zur veralteten Funktion `utcnow()`* (Blogbeitrag von Miguel Grinberg, abgerufen): Seit Python 3.12 sind `datetime.utcnow()` und `utcfromtimestamp()` als veraltet gekennzeichnet, weil sie naive Zeitpunkte liefern; Methoden wie `timestamp()` behandeln solche Werte als lokale Zeit des Rechners, was zu falschen Ergebnissen führen kann; als Ersatz nennt der Beitrag `datetime.now(timezone.utc)` (ab Python 3.11 auch `datetime.now(datetime.UTC)`) und für Zeitstempel `datetime.fromtimestamp(ts, timezone.utc)`. Quelle: <https://blog.miguelgrinberg.com/post/it-s-time-for-a-change-datetime-utcnow-is-now-deprecated/page/0>. *Zu `zoneinfo`* (Python-Dokumentation, abgerufen über eine Spiegelung des Quelltextes der Seite): `ZoneInfo` liest die Daten aus der Zeitzonendatenbank des Systems und greift sonst auf das Paket `tzdata` zurück; die Dokumentation weist darauf hin, dass manche Systeme, besonders Windows, keine solche Datenbank haben, und rät plattformübergreifenden Projekten, `tzdata` als Abhängigkeit anzugeben; fehlen beide Quellen, löst jeder Aufruf `ZoneInfoNotFoundError` aus (eine Unterklasse von `KeyError`); zeitzonenbewusste Zeitpunkte erzeugt man, indem man ein `ZoneInfo`-Objekt als `tzinfo` übergibt oder mit `replace()` beziehungsweise `astimezone()` anfügt. Quelle: <https://github.com/python/cpython/blob/bfc57d43d8766120ba0c8f3f6d7b2ac681a81d8a/Doc/library/zoneinfo.rst>. *Selbst ausgeführt* (Python 3.13.18, Zeitzone `Europe/Vienna`): das Verhalten bei der Zeitumstellung am 29. März und 25. Oktober 2026, das Verhalten von `strptime` beim 29. Februar ohne Jahr sowie die Meldungen der Fehlerbeispiele. Die Daten der Zeitumstellung (letzter Sonntag im März und im Oktober) entsprechen der Regel in der Europäischen Union und wurden aus der Zeitzonendatenbank im Sandkasten übernommen, nicht in einer amtlichen Quelle nachgeschlagen.

> **Merke**
>
> Zeitpunkt (`datetime`) minus Zeitpunkt ergibt eine Spanne (`timedelta`); `total_seconds()` statt `.seconds`. Intern mit zeitzonenbewussten UTC-Werten rechnen (`datetime.now(timezone.utc)`, nie `utcnow()`), erst zur Anzeige in die Ortszeit umrechnen (`astimezone`). `replace(tzinfo=...)` beschriftet, `astimezone` rechnet um. Echte Zeitzonen mit `zoneinfo` (unter Windows das Paket `tzdata`). Fenster als halboffenes Intervall `anfang <= t < ende`; auf Stunden abrunden mit `replace(minute=0, second=0, microsecond=0)`. Zeiten ohne Jahr (Syslog) brauchen eine Regel für das Jahr.

> **Praxisbeispiel**
>
> Zeitrechnung ist in der Administration allgegenwärtig: „Zeige alle Fehler der letzten Stunde“, „lösche Backups, die älter als 14 Tage sind“ (Modul 4 und 6), „melde, wenn der letzte erfolgreiche Lauf länger als 25 Stunden zurückliegt“, „zähle Anmeldeversuche pro Stunde“. Fast alle Zeitfehler in der Praxis stammen aus vergessenen Zeitzonen und der Sommerzeit: ein Report, der zweimal im Jahr falsche Zahlen liefert, oder ein Cron-Job, der in der Nacht der Zeitumstellung doppelt oder gar nicht läuft (Modul 9). Wer von Anfang an in UTC rechnet und Zeitstempel im ISO-Format mit Zeitzone speichert, erspart sich die meisten dieser Probleme.

## 8.4 Logparser (SSH-Fehlversuche pro Stunde)

Jetzt setzen wir die beiden Bausteine zusammen. Aufgabe: Aus der Protokolldatei des SSH-Dienstes sollen die fehlgeschlagenen Anmeldeversuche herausgezogen und pro Stunde gezählt werden, dazu die häufigsten Quelladressen und Benutzernamen. Wer einen Server mit offenem SSH-Zugang aus dem Internet betreibt, kennt das Bild: Schon wenige Minuten nach der Inbetriebnahme probieren automatische Programme Standardnamen wie `root`, `admin` oder `test` durch. Eine solche Auswertung zeigt, wie stark der Server beschossen wird, ob ein einzelner Angreifer dahintersteckt (eine Adresse mit vielen Versuchen) oder viele (viele Adressen mit wenigen Versuchen), und ob sich das Muster im Tagesverlauf ändert. Sie ist die Grundlage für Gegenmaßnahmen wie Fail2ban, eine Firewall-Regel oder das Abschalten der Passwort-Anmeldung zugunsten von Schlüsseln.

Bevor wir Code schreiben, klären wir, was genau wir zählen und aus welchen Daten. Der OpenSSH-Dienst schreibt seine Meldungen über das Syslog-System. Auf Debian und Ubuntu landen sie traditionell in `/var/log/auth.log`, auf Systemen mit `systemd` außerdem im Journal, das man mit `journalctl -u ssh` (Debian, Ubuntu) oder `journalctl -u sshd` (andere Distributionen) liest; auf Red-Hat-artigen Systemen heißt die Datei `/var/log/secure`. Die genauen Namen und die Form der Zeitstempel hängen von Distribution, Version und Konfiguration des Logdienstes ab (der klassische Syslog-Zeitstempel `Dec 17 03:12:45` ohne Jahr und Zeitzone ist verbreitet, neuere Konfigurationen schreiben ISO-8601-Zeitstempel mit Abstand zu UTC). Diese Angaben beruhen auf allgemeinem Fachwissen und wurden nicht in einer Quelle nachgeschlagen; entscheidend ist, dass Ihr Parser *beide* Zeitformate verstehen soll und Sie ihn an einer echten Datei Ihres Systems prüfen.

Auch die Meldungen selbst sind nicht festgeschrieben. Bei einem falschen Passwort für einen vorhandenen Benutzer schreibt OpenSSH typischerweise `Failed password for root from 203.0.113.5 port 51234 ssh2`; bei einem Benutzer, den es nicht gibt, `Failed password for invalid user admin from ...`; und oft zusätzlich eine eigene Zeile `Invalid user admin from ...`, bevor überhaupt ein Passwort versucht wird. Damit ein Versuch nicht doppelt zählt, entscheiden wir uns: Gezählt werden nur die Zeilen, die mit `Failed` beginnen, also tatsächlich fehlgeschlagene Anmeldungen mit einer Methode (`password`, `publickey` und so weiter). Das ist eine *Entwurfsentscheidung*, die man dokumentieren sollte, denn eine andere Zählweise liefert andere Zahlen. Dass alle diese Texte den Beobachtungen auf gängigen Debian- und Ubuntu-Systemen entsprechen, ist ebenfalls Fachwissen und nicht nachgeschlagen; Sie sollten Ihr Muster deshalb an den Zeilen Ihres eigenen Servers prüfen.

**Der Aufbau des Programms.** Wir gliedern das Programm in kleine Funktionen, die jede eine Aufgabe haben und einzeln testbar sind. Das Modul heißt `sshlog.py`. Zuerst kommen zwei Muster: `KOPF` zerlegt den Anfang jeder `sshd`-Zeile in Zeit, Rechner und Meldung und kennt beide Zeitformate als Alternativen (der senkrechte Strich im Muster trennt die Möglichkeiten); `FEHLVERSUCH` wird nur auf die Meldung angewendet und holt Methode, Benutzer, Adresse und Port. Die Funktion `parse_zeit` liest je nach Aussehen des Textes das eine oder das andere Format und liefert immer einen *bewussten UTC-Zeitpunkt*; so sind alle weiteren Berechnungen einheitlich. `lies_fehlversuche` ist ein Generator, der die Datei Zeile für Zeile verarbeitet und dabei in einer Statistik mitzählt, was mit den Zeilen geschah. `pro_stunde` und `haeufigste` schließlich werten die Einträge aus.

```python
"""sshlog - fehlgeschlagene SSH-Anmeldungen aus auth.log-Zeilen auswerten."""
import re
from collections import Counter
from datetime import datetime, timedelta, timezone

# Kopf einer sshd-Zeile: klassischer Syslog-Zeitstempel oder ISO 8601
KOPF = re.compile(r"""
    ^(?P<zeit>
        [A-Z][a-z]{2}\s+\d{1,2}\s+\d\d:\d\d:\d\d                          # Dec 17 03:12:45
      | \d{4}-\d\d-\d\dT\d\d:\d\d:\d\d(?:\.\d+)?(?:Z|[+-]\d\d:\d\d)?      # 2026-12-17T03:12:45+01:00
    )\s+(?P<host>\S+)\s+sshd(?:\[\d+\])?:\s+(?P<meldung>.*)$
""", re.VERBOSE)

# Meldung eines fehlgeschlagenen Anmeldeversuchs
FEHLVERSUCH = re.compile(
    r"^Failed (?P<methode>\S+) for (?:invalid user )?(?P<benutzer>\S+)"
    r" from (?P<ip>\S+) port (?P<port>\d+)"
)


def syslog_zeit(text, jetzt, zone):
    """Wandelt 'Dec 17 03:12:45' (ohne Jahr) in einen bewussten UTC-Zeitpunkt."""
    text = " ".join(text.split())
    ortszeit_jetzt = jetzt.astimezone(zone)
    for jahr in (ortszeit_jetzt.year, ortszeit_jetzt.year - 1):
        try:
            zeitpunkt = datetime.strptime(f"{jahr} {text}", "%Y %b %d %H:%M:%S").replace(tzinfo=zone)
        except ValueError:
            continue
        if zeitpunkt <= jetzt:
            return zeitpunkt.astimezone(timezone.utc)
    raise ValueError(f"Keine gültige Syslog-Zeit: {text!r}")


def parse_zeit(text, jetzt, zone):
    """Liest beide Zeitformate und liefert immer einen bewussten UTC-Zeitpunkt."""
    if text[0].isdigit():
        zeitpunkt = datetime.fromisoformat(text)
        if zeitpunkt.tzinfo is None:
            zeitpunkt = zeitpunkt.replace(tzinfo=zone)
        return zeitpunkt.astimezone(timezone.utc)
    return syslog_zeit(text, jetzt, zone)


def lies_fehlversuche(zeilen, jetzt, zone, statistik):
    """Liefert je Fehlversuch ein Dictionary; zählt alles Übrige in 'statistik'."""
    for zeile in zeilen:
        statistik["zeilen"] += 1
        kopf = KOPF.match(zeile.rstrip("\n"))
        if kopf is None:
            statistik["fremd"] += 1              # andere Programme (sudo, CRON ...) oder Leerzeilen
            continue
        treffer = FEHLVERSUCH.match(kopf["meldung"])
        if treffer is None:
            statistik["sshd_sonst"] += 1         # sshd-Zeile, aber kein Fehlversuch
            continue
        try:
            zeit = parse_zeit(kopf["zeit"], jetzt, zone)
        except ValueError:
            statistik["zeit_ungueltig"] += 1
            continue
        statistik["fehlversuche"] += 1
        yield {"zeit": zeit, "host": kopf["host"], **treffer.groupdict()}


def stunde_von(zeitpunkt):
    return zeitpunkt.replace(minute=0, second=0, microsecond=0)


def pro_stunde(eintraege, von, bis):
    """Zählt Einträge je voller Stunde; Stunden ohne Treffer erscheinen mit 0."""
    zaehler = Counter(stunde_von(e["zeit"]) for e in eintraege if von <= e["zeit"] < bis)
    ergebnis = {}
    stunde = stunde_von(von)
    while stunde < bis:
        ergebnis[stunde] = zaehler.get(stunde, 0)
        stunde += timedelta(hours=1)
    return ergebnis


def haeufigste(eintraege, schluessel, anzahl, von, bis):
    zaehler = Counter(e[schluessel] for e in eintraege if von <= e["zeit"] < bis)
    return zaehler.most_common(anzahl)
```

Gehen wir die wichtigsten Stellen durch. Das Muster `KOPF` verwendet `re.VERBOSE`, damit Kommentare die beiden Alternativen erklären können. Ein Zeitstempel ist *entweder* die klassische Form (drei Buchstaben für den Monat, Tag, Uhrzeit) *oder* die ISO-Form (Datum, `T`, Uhrzeit, optional Sekundenbruchteile und optional ein Abstand zu UTC oder `Z`). Im Muster steht der Rest der Zeile als Meldung; dort suchen wir weiter. Dass `KOPF` nur Zeilen akzeptiert, in denen `sshd` als Programm steht, ist eine wichtige Eigenschaft: In `auth.log` stehen auch Meldungen von `sudo`, `cron`, `su` und anderen, und die sollen nicht versehentlich mitgezählt werden.

In `parse_zeit` bestimmt das erste Zeichen, ob es sich um ein Jahr (eine Ziffer) oder einen Monatsnamen (ein Buchstabe) handelt. Ist es ISO, macht `datetime.fromisoformat` daraus einen Zeitpunkt; enthält der Text keinen Abstand zu UTC, wird die Zeitzone des Servers angenommen (`replace(tzinfo=zone)`); anschließend rechnet `astimezone(timezone.utc)` auf Weltzeit um. Ist es das klassische Format, übernimmt die Funktion `syslog_zeit` aus 8.3 die Arbeit, mit der Regel für das Jahr. Beachten Sie, dass `datetime.fromisoformat` seit Python 3.11 die meisten ISO-8601-Schreibweisen versteht, darunter auch ein angehängtes `Z`; in älteren Python-Versionen war es strenger. Das ist die Python-Version, mit der wir die Beispiele geprüft haben (3.13).

Die Funktion `lies_fehlversuche` ist der Kern. Für jede Zeile erhöht sie zuerst den Zähler `zeilen`. Passt `KOPF` nicht, ist die Zeile *fremd*: eine andere Anwendung, eine Leerzeile oder ein ganz anderes Format; sie wird gezählt und übersprungen. Passt `KOPF`, aber `FEHLVERSUCH` nicht, ist es eine `sshd`-Zeile ohne Fehlversuch (`Accepted publickey`, `Connection closed`); auch sie wird gezählt. Nur wenn beide Muster passen, wird der Zeitstempel gelesen; schlägt das fehl (der 30. Februar gibt es nicht), zählt die Zeile unter `zeit_ungueltig`. Erst danach liefert der Generator ein Dictionary. Die Schreibweise `{"zeit": zeit, "host": kopf["host"], **treffer.groupdict()}` baut ein Dictionary aus zwei festen Einträgen und allen Einträgen aus `groupdict()`; das `**` „entpackt“ ein Dictionary in ein anderes (siehe Modul 3). Am Ende der Verarbeitung ergibt sich aus der Statistik eine Bilanz, die immer aufgehen sollte: Jede Zeile ist entweder fremd, ein anderer sshd-Eintrag, ein Fehlversuch mit ungültiger Zeit oder ein gezählter Fehlversuch.

Zuletzt die Auswertung. `stunde_von` rundet einen Zeitpunkt auf die volle Stunde ab (siehe 8.3). `pro_stunde` zählt die Einträge im Fenster `von` bis `bis` je Stunde mit einem `Counter` und gibt dann ein Dictionary zurück, das *jede* Stunde des Fensters enthält, auch die ohne Treffer, mit dem Wert 0. Dieser Schritt ist wichtig: Ein Diagramm oder eine Tabelle, in der leere Stunden einfach fehlen, verzerrt den Eindruck (eine ruhige Nacht sähe wie ein durchgehender Beschuss aus). `haeufigste` nutzt `Counter.most_common`, das die gesuchte Rangliste liefert. Eine Einschränkung: Die Stunden sind *UTC-Stunden*. In Zeitzonen mit einem Abstand von einer halben oder dreiviertel Stunde (zum Beispiel Indien) stimmen sie nicht mit den lokalen Stunden überein; für Mitteleuropa mit seinen vollen Stunden ist das unkritisch.

Nun probieren wir es aus. Das folgende Programm enthält eine kleine, erfundene Logdatei mit zwölf Zeilen: darunter ein Fehlversuch mit nicht vorhandenem Benutzer, eine erfolgreiche Anmeldung, eine Zeile von `sudo`, eine Leerzeile, eine Zeile mit ungültigem Datum und Zeilen in der ISO-Schreibweise. Das Zeitfenster reicht von 03:00 bis 07:00 Uhr Ortszeit in Wien.

```python
from collections import Counter
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

import sshlog

LOG = """\
Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2
Dec 17 03:12:51 srv sshd[813]: Failed password for invalid user admin from 203.0.113.5 port 51240 ssh2
Dec 17 03:13:02 srv sshd[814]: Accepted publickey for anna from 198.51.100.7 port 40022 ssh2
Dec 17 03:14:00 srv sudo:     anna : TTY=pts/0 ; PWD=/home/anna ; USER=root ; COMMAND=/bin/ls

Dec 17 04:01:10 srv sshd[820]: Failed password for root from 203.0.113.5 port 51300 ssh2
Dec 17 04:01:15 srv sshd[821]: Failed password for root from 192.0.2.44 port 40100 ssh2
Dec 17 04:02:00 srv sshd[822]: Connection closed by 198.51.100.7 port 40022
Feb 30 10:00:00 srv sshd[823]: Failed password for root from 192.0.2.1 port 22 ssh2
2026-12-17T06:05:00+01:00 srv sshd[900]: Failed password for test from 192.0.2.44 port 40101 ssh2
2026-12-17T06:06:30+01:00 srv sshd[901]: Failed password for test from 192.0.2.44 port 40102 ssh2
2026-12-17T06:07:10+01:00 srv sshd[902]: Failed password for oracle from 203.0.113.5 port 40103 ssh2
"""

wien = ZoneInfo("Europe/Vienna")
jetzt = datetime(2026, 12, 17, 7, 0, tzinfo=wien)
bis = jetzt.astimezone(timezone.utc)
von = datetime(2026, 12, 17, 3, 0, tzinfo=wien).astimezone(timezone.utc)

statistik = Counter()
eintraege = list(sshlog.lies_fehlversuche(LOG.splitlines(), bis, wien, statistik))

print("Statistik:", dict(statistik))
print("Zeitfenster:", von.astimezone(wien).strftime("%H:%M"), "bis", bis.astimezone(wien).strftime("%H:%M"))
for stunde, anzahl in sshlog.pro_stunde(eintraege, von, bis).items():
    print(stunde.astimezone(wien).strftime("%H:%M"), f"{anzahl:2}", "#" * anzahl)
print("Top-Adressen :", sshlog.haeufigste(eintraege, "ip", 3, von, bis))
print("Top-Benutzer :", sshlog.haeufigste(eintraege, "benutzer", 3, von, bis))
```

```text
Statistik: {'zeilen': 12, 'fehlversuche': 7, 'sshd_sonst': 2, 'fremd': 2, 'zeit_ungueltig': 1}
Zeitfenster: 03:00 bis 07:00
03:00  2 ##
04:00  2 ##
05:00  0
06:00  3 ###
Top-Adressen : [('203.0.113.5', 4), ('192.0.2.44', 3)]
Top-Benutzer : [('root', 3), ('test', 2), ('admin', 1)]
```

Die Statistik geht auf: zwölf Zeilen, davon sieben gezählte Fehlversuche, zwei weitere `sshd`-Zeilen (`Accepted` und `Connection closed`), zwei fremde Zeilen (die `sudo`-Zeile und die Leerzeile) und eine Zeile mit ungültiger Zeit (der 30. Februar); `7 + 2 + 2 + 1 = 12`. Die Tabelle zeigt vier Stunden, auch die Stunde 05:00 ohne Treffer. Die zwei Fehlversuche um 03:00 Uhr kommen aus den klassischen Zeilen, die drei um 06:00 Uhr aus den ISO-Zeilen; beide Formate landen im selben Zähler, weil beide zuvor in UTC umgerechnet wurden. Die Adresse `203.0.113.5` ist mit vier Versuchen die häufigste; als Benutzername wird `root` am häufigsten versucht. Dass eine einzelne Adresse zwischen 03:12 und 06:07 Uhr immer wieder auftaucht, ist genau das Muster, das auf einen automatisierten Angriff hindeutet.

**Prüfen, ob die Zahlen stimmen.** Ein Parser, der Zahlen liefert, ist nur dann nützlich, wenn man ihnen vertrauen kann. Die wichtigste Kontrolle ist der Vergleich mit einer unabhängigen Zählung. Auf der Kommandozeile zählt `grep sshd /var/log/auth.log | grep -c "Failed "` die `sshd`-Zeilen mit einem fehlgeschlagenen Versuch; die Zahl muss mit der Summe der gezählten Fehlversuche und der Fehlversuche mit ungültiger Zeit übereinstimmen. Weichen die Zahlen ab, liegt es meist an einer Meldungsvariante, die das Muster nicht kennt. Die zweite Kontrolle ist die Statistik selbst: Ein Parser, der 90 Prozent der `sshd`-Zeilen als „sonst“ einstuft, hat vermutlich ein falsches Muster.

Ein weiterer Test ist eine bewusst gebaute Probedatei, deren Ergebnis man von Hand ausrechnen kann, so wie in unserem Beispiel. Solche Tests sollte jeder Parser besitzen, und in Modul 14 schreiben wir sie als automatisierte Tests mit `pytest`. Hier reicht es, zu wissen, dass *jede* Zeile entweder verarbeitet oder in einem benannten Zähler gelandet ist.

**Große Dateien und rotierte Logs.** Zwei praktische Einwände: Logdateien können sehr groß sein, und sie werden regelmäßig *rotiert*. Zum Ersten: Unser Programm liest die Datei Zeile für Zeile durch den Generator, es braucht also nicht die ganze Datei im Speicher. Nur die gefundenen Fehlversuche werden in einer Liste gesammelt (die Demo ruft `list(...)` auf), und diese Liste wächst mit der Zahl der Fehlversuche, nicht mit der Größe der Datei. Wäre selbst das zu viel, könnte man direkt in den Zählern sammeln. Zum Zweiten: Die Dienstprogramme `logrotate` benennen alte Logdateien um (`auth.log.1`) und komprimieren ältere (`auth.log.2.gz`). Ein Zeitfenster, das weiter zurückreicht als die aktuelle Datei, muss mehrere Dateien lesen. Die komprimierten liest man mit dem Modul `gzip` der Standardbibliothek: `gzip.open(pfad, "rt", encoding="utf-8")` öffnet sie wie eine Textdatei. Diese Angaben zur Rotation beruhen auf allgemeinem Fachwissen. Ihr tatsächliches Verhalten legt die Konfiguration von `logrotate` auf Ihrem System fest.

```python
import gzip
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    gz = Path(tmp) / "auth.log.2.gz"
    with gzip.open(gz, "wt", encoding="utf-8") as f:
        f.write("Dec 16 23:59:59 srv sshd[1]: Failed password for root from 192.0.2.1 port 22 ssh2\n")

    zeilen = []
    for pfad in (gz,):                                   # in der Praxis: mehrere Dateien
        offen = gzip.open if pfad.suffix == ".gz" else open
        with offen(pfad, "rt", encoding="utf-8") as f:
            zeilen.extend(f)
    print(len(zeilen), "Zeile(n) gelesen:", zeilen[0].split()[:3])
```

```text
1 Zeile(n) gelesen: ['Dec', '16', '23:59:59']
```

Das Beispiel schreibt zuerst eine komprimierte Datei, liest sie dann mit `gzip.open(..., "rt")` und wählt `gzip.open` oder `open` je nach Endung. Das ist die übliche Weise, mit rotierten Logs umzugehen; ein Zeilen-Generator über mehrere Dateien lässt sich daraus in drei Zeilen bauen. In 8.5 setzen wir das in ein vollständiges Kommandozeilenprogramm um.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler bei Logparsern kommen nicht aus der Programmierung, sondern aus falschen Annahmen über die Daten. Eine Auswahl mit Gegenmaßnahmen: *Doppelt gezählte Versuche*, weil man sowohl `Invalid user` als auch `Failed password` zählt; die Gegenmaßnahme ist die dokumentierte Zählweise. *Falsche Stunde*, weil man die Zeitzone des Servers vergessen hat; die Gegenmaßnahme ist `zone` als Parameter und das Umrechnen auf UTC am Eingang. *Falsches Jahr*, weil im Januar Einträge vom Dezember stehen; die Gegenmaßnahme ist die Regel „nicht in der Zukunft“. *Fehlende Stunden* in der Tabelle, weil nur Stunden mit Treffern auftauchen; die Gegenmaßnahme ist das Auffüllen mit null. *Stillschweigend ignorierte Zeilen*, weil das Muster zu streng ist; die Gegenmaßnahme ist die Statistik. Das folgende Beispiel zeigt die beiden letzten an einer kleinen Datei.

```python
from collections import Counter
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

import sshlog

LOG = [
    "Dec 17 03:12:45 srv sshd[1]: Failed password for root from 203.0.113.5 port 1 ssh2",
    "Dec 17 03:12:46 srv sshd[1]: Failed publickey for root from 203.0.113.5 port 1 ssh2",
    "Dec 17 05:30:00 srv sshd[1]: Failed password for root from 203.0.113.5 port 2 ssh2",
]
utc = ZoneInfo("UTC")
von = datetime(2026, 12, 17, 3, 0, tzinfo=timezone.utc)
bis = datetime(2026, 12, 17, 7, 0, tzinfo=timezone.utc)

statistik = Counter()
eintraege = list(sshlog.lies_fehlversuche(LOG, bis, utc, statistik))
print(dict(statistik))
print("mit Lücken      :", {s.strftime("%H"): n for s, n in Counter(e["zeit"].replace(minute=0, second=0) for e in eintraege).items()})
print("aufgefüllt      :", {s.strftime("%H"): n for s, n in sshlog.pro_stunde(eintraege, von, bis).items()})
print("Methoden        :", Counter(e["methode"] for e in eintraege))
```

```text
{'zeilen': 3, 'fehlversuche': 3}
mit Lücken      : {'03': 2, '05': 1}
aufgefüllt      : {'03': 2, '04': 0, '05': 1, '06': 0}
Methoden        : Counter({'password': 2, 'publickey': 1})
```

Die Variante „mit Lücken“ kennt die Stunden 04 und 06 überhaupt nicht, die aufgefüllte Variante zeigt sie mit 0. Dass das Programm auch `Failed publickey` erkennt, kommt vom Muster `Failed (?P<methode>\S+)`; es zählt jede Methode, und die letzte Zeile zeigt die Verteilung. Wollte man nur Passwort-Versuche zählen, würde man das Muster auf `Failed password` einschränken oder nach der Methode filtern.

> **Ergänzung aus der Nachrecherche**
>
> **Was in diesem Abschnitt nicht aus Quellen stammt.** Die Formate der OpenSSH-Meldungen (`Failed password for`, `Invalid user`, `Accepted publickey`), die Dateinamen der Protokolle (`auth.log`, `secure`), die Befehle zum Lesen des Journals sowie das Verhalten von `logrotate` wurden *nicht* in der Dokumentation nachgeschlagen; sie beruhen auf allgemeinem Fachwissen. Die Beispielzeilen sind erfunden und an das übliche Aussehen angelehnt. Eine Prüfung gegen die Dokumentation von OpenSSH (`sshd(8)`, Abschnitt zur Protokollierung), der Syslog-Konfiguration Ihrer Distribution und, falls Sie das Journal nutzen, `journalctl(1)` steht noch aus und gehört in die Pflege dieses Skripts. Abgerufen und für dieses Modul verwendet wurden die Python-Dokumentation zu regulären Ausdrücken (siehe 8.1) sowie die Hinweise zu `zoneinfo` und `utcnow()` (siehe 8.3). Die Ausführung der Beispiele erfolgte mit Python 3.13.18.

> **Merke**
>
> Ein Logparser besteht aus kleinen Funktionen: ein Muster für den Kopf, eins für die Meldung, eine Zeitfunktion, die immer UTC liefert, ein Generator, der jede Zeile zählt (fremd, sonstige, Fehlversuch, ungültig), und Auswertungen mit `Counter`. Alle Stunden des Fensters ausgeben, auch die leeren. Die Zählweise (was ist ein Fehlversuch?) dokumentieren und die Summe gegen eine unabhängige Zählung (`grep -c`) prüfen. Rotierte Logs mit `gzip.open` lesen.

> **Praxisbeispiel**
>
> Auf jedem Server, der aus dem Internet per SSH erreichbar ist, gehört eine solche Auswertung zur Routine. Die Ergebnisse führen zu konkreten Entscheidungen: Passwort-Anmeldung abschalten und nur Schlüssel erlauben, den Port ändern oder den Zugang auf bestimmte Adressen beschränken, ein Werkzeug wie Fail2ban einrichten, das Adressen nach zu vielen Fehlversuchen sperrt. Der Parser dieser Lerneinheit liefert die Zahlen dafür und lässt sich leicht auf andere Dienste übertragen: Mailserver, Webserver, VPN. Beachten Sie bei der Weitergabe der Ergebnisse, dass IP-Adressen und Benutzernamen personenbezogene Daten sein können (keine Rechtsberatung); im Report für das Team genügen oft die Zahlen und die häufigsten Adressen.

## 8.5 Logparser abschließen + Quiz

Das Modul `sshlog` aus 8.4 kann Zeilen lesen, Zeiten umrechnen und zählen. Für die tägliche Arbeit fehlt noch das, was aus einer Funktionssammlung ein Werkzeug macht: eine Kommandozeile, mit der man Dateien und Zeitfenster angibt, eine Rückmeldung für Skripte und Überwachung (Exit-Code), eine maschinenlesbare Ausgabe und eine ordentliche Fehlerbehandlung. Das ist der Inhalt dieser Lerneinheit: das Programm `sshfails.py`, das auf `sshlog` aufbaut und alle Bausteine der Module 4 bis 8 zusammenführt: Dateien und `gzip` (Modul 4), `argparse` und Exit-Codes (Modul 5 und 6), JSON (Modul 7) sowie reguläre Ausdrücke und `datetime` (dieses Modul).

**Anforderungen.** Bevor man ein Werkzeug schreibt, legt man fest, was es können soll. Unser Programm bekommt eine oder mehrere Logdateien (auch komprimierte mit der Endung `.gz`, und `-` für die Standardeingabe, damit man Ausgaben anderer Programme per Pipe hineinleiten kann). Mit `--seit` wählt man das Zeitfenster rückwärts ab jetzt in der Schreibweise `30m`, `24h` oder `7d`. Mit `--zone` sagt man, in welcher Zeitzone der Server seine klassischen Zeitstempel schreibt. `--top` legt die Länge der Ranglisten fest, `--format` wählt Text oder JSON, und `--schwelle` macht aus dem Bericht eine Überwachung: Hat eine einzelne Adresse im Fenster mindestens so viele Fehlversuche, endet das Programm mit dem Exit-Code 1 und einer Warnung. Eine versteckte Option `--jetzt` erlaubt es, den „aktuellen“ Zeitpunkt vorzugeben; das ist für Tests unentbehrlich, weil sonst jedes Ergebnis von der Uhrzeit des Laufs abhinge. Die Exit-Codes: 0 für „alles in Ordnung“, 1 für „Schwelle überschritten“, 2 für falsche Optionen (das erledigt `argparse` selbst) und 10 für eine fehlerhafte Eingabe (unlesbare Datei, ungültige Dauer oder Zeitzone), nach dem Schema aus Modul 6.

```python
#!/usr/bin/env python3
"""sshfails - fehlgeschlagene SSH-Anmeldungen pro Stunde auswerten."""
import argparse
import gzip
import json
import re
import sys
from collections import Counter
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo, ZoneInfoNotFoundError

import sshlog

DAUER = re.compile(r"^(?P<zahl>\d+)(?P<einheit>[mhd])$")
EINHEITEN = {"m": "minutes", "h": "hours", "d": "days"}


class EingabeFehler(Exception):
    """Falsche Angabe des Anwenders oder nicht lesbare Datei (Exit-Code 10)."""


def parse_dauer(text):
    treffer = DAUER.match(text)
    if treffer is None or int(treffer["zahl"]) == 0:
        raise EingabeFehler(f"Ungültige Dauer {text!r} (erlaubt: 30m, 24h, 7d)")
    return timedelta(**{EINHEITEN[treffer["einheit"]]: int(treffer["zahl"])})


def lade_zone(name):
    try:
        return ZoneInfo(name)
    except (ZoneInfoNotFoundError, ValueError) as fehler:
        raise EingabeFehler(f"Unbekannte Zeitzone {name!r}") from fehler


def alle_zeilen(pfade, stdin):
    for pfad in pfade:
        if pfad == "-":
            yield from stdin
            continue
        oeffnen = gzip.open if pfad.endswith(".gz") else open
        try:
            with oeffnen(pfad, "rt", encoding="utf-8", errors="replace") as datei:
                yield from datei
        except OSError as fehler:
            raise EingabeFehler(f"{pfad}: {fehler.strerror or fehler}") from fehler


def bericht_text(fenster, zone, statistik, stunden, adressen, benutzer, warnung):
    von, bis = fenster

    def zeige(zeitpunkt):
        return zeitpunkt.astimezone(zone).strftime("%Y-%m-%d %H:%M")

    zeilen = [f"Zeitfenster: {zeige(von)} bis {zeige(bis)} ({zone.key})"]
    zeilen.append("Zeilen: {zeilen}, Fehlversuche: {fehlversuche}, fremd: {fremd}, "
                  "sonstige sshd: {sshd_sonst}, ungültige Zeit: {zeit_ungueltig}".format_map(statistik))
    zeilen.append("")
    for stunde, anzahl in stunden.items():
        zeilen.append(f"{stunde.astimezone(zone).strftime('%d.%m. %H:00')} {anzahl:4} {'#' * min(anzahl, 40)}")
    zeilen.append("")
    zeilen.append("Häufigste Adressen: " + (", ".join(f"{k} ({n})" for k, n in adressen) or "-"))
    zeilen.append("Häufigste Benutzer: " + (", ".join(f"{k} ({n})" for k, n in benutzer) or "-"))
    if warnung:
        zeilen.append(warnung)
    return "\n".join(zeilen)


def main(argv=None, stdin=None, jetzt=None):
    parser = argparse.ArgumentParser(description="Fehlgeschlagene SSH-Anmeldungen pro Stunde zählen.")
    parser.add_argument("dateien", nargs="+", help="Logdateien (.gz erlaubt), '-' für die Standardeingabe")
    parser.add_argument("--seit", default="24h", help="Zeitfenster rückwärts ab jetzt: 30m, 24h, 7d (Standard 24h)")
    parser.add_argument("--zone", default="UTC", help="Zeitzone, in der der Server schreibt (Standard UTC)")
    parser.add_argument("--top", type=int, default=5, help="Länge der Ranglisten (Standard 5)")
    parser.add_argument("--schwelle", type=int, help="Exit-Code 1, wenn eine Adresse mindestens so oft scheiterte")
    parser.add_argument("--format", choices=("text", "json"), default="text")
    parser.add_argument("--jetzt", help="ISO-Zeitpunkt mit Zeitzone statt der aktuellen Zeit (für Tests)")
    args = parser.parse_args(argv)

    try:
        zone = lade_zone(args.zone)
        dauer = parse_dauer(args.seit)
        if args.jetzt:
            bis = datetime.fromisoformat(args.jetzt)
            if bis.tzinfo is None:
                raise EingabeFehler("--jetzt braucht eine Zeitzone, z. B. 2026-12-17T07:00:00+01:00")
        else:
            bis = jetzt or datetime.now(timezone.utc)
        bis = bis.astimezone(timezone.utc)
        von = bis - dauer

        statistik = Counter(zeilen=0, fehlversuche=0, fremd=0, sshd_sonst=0, zeit_ungueltig=0)
        zeilen = alle_zeilen(args.dateien, stdin if stdin is not None else sys.stdin)
        eintraege = list(sshlog.lies_fehlversuche(zeilen, bis, zone, statistik))
    except (EingabeFehler, ValueError) as fehler:
        print(f"Fehler: {fehler}", file=sys.stderr)
        return 10

    stunden = sshlog.pro_stunde(eintraege, von, bis)
    adressen = sshlog.haeufigste(eintraege, "ip", args.top, von, bis)
    benutzer = sshlog.haeufigste(eintraege, "benutzer", args.top, von, bis)

    warnung = None
    if args.schwelle is not None and adressen and adressen[0][1] >= args.schwelle:
        warnung = f"WARNUNG: {adressen[0][0]} hatte {adressen[0][1]} Fehlversuche (Schwelle {args.schwelle})"

    if args.format == "json":
        print(json.dumps({
            "von": von.isoformat(), "bis": bis.isoformat(),
            "statistik": dict(statistik),
            "pro_stunde": {s.isoformat(): n for s, n in stunden.items()},
            "adressen": adressen, "benutzer": benutzer, "warnung": warnung,
        }, indent=2, ensure_ascii=False))
    else:
        print(bericht_text((von, bis), zone, statistik, stunden, adressen, benutzer, warnung))
    return 1 if warnung else 0


if __name__ == "__main__":
    sys.exit(main())
```

Der Aufbau folgt der Idee, die Sie in 7.5 kennengelernt haben: Hilfsfunktionen lösen Ausnahmen aus, und nur `main` wandelt sie in Meldungen und Exit-Codes um. Die Funktion `parse_dauer` ist ein kleines Beispiel für einen sinnvollen Einsatz von Regex außerhalb des Log-Parsens: Das Muster `^(?P<zahl>\d+)(?P<einheit>[mhd])$` beschreibt genau die erlaubten Eingaben, und ein Wörterbuch ordnet dem Buchstaben die Einheit zu, die `timedelta` als Parameter erwartet; `timedelta(**{"hours": 24})` ist dasselbe wie `timedelta(hours=24)`. Eine Dauer von null wird abgelehnt, weil ein leeres Fenster keinen Sinn hat. `lade_zone` fängt die beiden Fehler ab, die `ZoneInfo` auslösen kann: `ZoneInfoNotFoundError`, wenn der Name unbekannt ist (oder auf einem System ohne Zeitzonendaten, siehe 8.3), und `ValueError`, wenn der Name nicht wie ein gültiger Schlüssel aussieht.

Die Funktion `alle_zeilen` ist ein Generator über mehrere Quellen. Für jede Datei wählt sie anhand der Endung `gzip.open` oder `open` und öffnet sie im Textmodus (`"rt"`) mit der Kodierung UTF-8. Das Argument `errors="replace"` ist hier eine bewusste Entscheidung: Logdateien enthalten Daten, die von außen kommen (Benutzernamen, Rechnernamen), und diese können Bytes enthalten, die kein gültiges UTF-8 sind. Mit der Voreinstellung würde das Programm mit `UnicodeDecodeError` mitten in der Datei abbrechen; mit `"replace"` werden die ungültigen Bytes durch ein Ersatzzeichen ersetzt, und die Zeile bleibt lesbar. Dass die Fehlerbehandlung *innerhalb* des Generators um die `with`-Anweisung liegt, hat Folgen: Die Ausnahme tritt erst beim Durchlaufen auf und nicht beim Erzeugen des Generators, deshalb sitzt der Aufruf in `main` innerhalb des `try`-Blocks, und zwar dort, wo die Liste gebildet wird. Ein `OSError` wird in unsere `EingabeFehler` umgewandelt, mit dem Dateinamen und der Meldung des Betriebssystems.

In `main` sind zwei Parameter bemerkenswert, die ein Kommandozeilenprogramm normalerweise nicht hat: `stdin` und `jetzt`. Sie gehören zu den Techniken, die ein Programm *testbar* machen: Statt fest `sys.stdin` und die Systemuhr zu verwenden, kann ein Test eigene Werte übergeben. Beim normalen Start bleiben beide `None`, und das Programm nimmt die echten Werte (`sys.stdin`, `datetime.now(timezone.utc)`). Der Bericht entsteht in `bericht_text`; die Methode `format\_map` füllt die Platzhalter der Zeichenkette aus dem Dictionary der Statistik, ohne dass man jeden Schlüssel einzeln angeben muss. Die Anzeige der Balken ist auf 40 Zeichen begrenzt, damit ein Ausreißer nicht die Zeile sprengt.

Prüfen wir das Programm wie ein Anwender, in fünf Aufrufen. Die Demo legt in einem temporären Verzeichnis eine Logdatei und eine komprimierte Datei vom Vortag an und ruft `main` auf, als wären es Kommandozeilenaufrufe; der Vorgabewert für `jetzt` ist fest (7 Uhr Ortszeit in Wien), damit das Ergebnis reproduzierbar ist. Beachten Sie die fünf Fälle: ein Bericht über zwei Dateien, ein Bericht mit Schwelle, die Ausgabe als JSON mit Standardeingabe, eine ungültige Dauer und eine fehlende Datei.

```python
import contextlib
import gzip
import io
import os
import tempfile
from datetime import datetime
from pathlib import Path

import sshfails

heute = """\
Dec 17 03:12:45 srv sshd[812]: Failed password for root from 203.0.113.5 port 51234 ssh2
Dec 17 03:12:51 srv sshd[813]: Failed password for invalid user admin from 203.0.113.5 port 51240 ssh2
Dec 17 03:14:00 srv sudo:     anna : TTY=pts/0 ; PWD=/home/anna ; USER=root ; COMMAND=/bin/ls
Dec 17 04:01:15 srv sshd[821]: Failed password for root from 192.0.2.44 port 40100 ssh2
Dec 17 06:07:10 srv sshd[902]: Failed password for oracle from 203.0.113.5 port 40103 ssh2
"""
gestern = "Dec 16 23:30:00 srv sshd[700]: Failed password for root from 203.0.113.5 port 5 ssh2\n"

def lauf(*argv, stdin=None):
    puffer = io.StringIO()
    with contextlib.redirect_stdout(puffer), contextlib.redirect_stderr(puffer):
        code = sshfails.main(list(argv), stdin=stdin)
    print("$ sshfails " + " ".join(argv))
    print(puffer.getvalue().rstrip("\n"))
    print(f"[Exit-Code {code}]\n")

with tempfile.TemporaryDirectory() as tmp:
    os.chdir(tmp)
    Path("auth.log").write_text(heute, encoding="utf-8")
    with gzip.open("auth.log.1.gz", "wt", encoding="utf-8") as f:
        f.write(gestern)

    grund = ["--zone", "Europe/Vienna", "--jetzt", "2026-12-17T07:00:00+01:00"]
    lauf("auth.log", "auth.log.1.gz", "--seit", "8h", *grund)
    lauf("auth.log", "--seit", "4h", "--top", "1", "--schwelle", "3", *grund)
    lauf("-", "--seit", "4h", "--format", "json", "--top", "1", *grund, stdin=io.StringIO(heute))
    lauf("auth.log", "--seit", "zwei Stunden", *grund)
    lauf("fehlt.log", *grund)
```

```text
$ sshfails auth.log auth.log.1.gz --seit 8h --zone Europe/Vienna --jetzt 2026-12-17T07:00:00+01:00
Zeitfenster: 2026-12-16 23:00 bis 2026-12-17 07:00 (Europe/Vienna)
Zeilen: 6, Fehlversuche: 5, fremd: 1, sonstige sshd: 0, ungültige Zeit: 0

16.12. 23:00    1 #
17.12. 00:00    0
17.12. 01:00    0
17.12. 02:00    0
17.12. 03:00    2 ##
17.12. 04:00    1 #
17.12. 05:00    0
17.12. 06:00    1 #

Häufigste Adressen: 203.0.113.5 (4), 192.0.2.44 (1)
Häufigste Benutzer: root (3), admin (1), oracle (1)
[Exit-Code 0]

$ sshfails auth.log --seit 4h --top 1 --schwelle 3 --zone Europe/Vienna --jetzt 2026-12-17T07:00:00+01:00
Zeitfenster: 2026-12-17 03:00 bis 2026-12-17 07:00 (Europe/Vienna)
Zeilen: 5, Fehlversuche: 4, fremd: 1, sonstige sshd: 0, ungültige Zeit: 0

17.12. 03:00    2 ##
17.12. 04:00    1 #
17.12. 05:00    0
17.12. 06:00    1 #

Häufigste Adressen: 203.0.113.5 (3)
Häufigste Benutzer: root (2)
WARNUNG: 203.0.113.5 hatte 3 Fehlversuche (Schwelle 3)
[Exit-Code 1]

$ sshfails - --seit 4h --format json --top 1 --zone Europe/Vienna --jetzt 2026-12-17T07:00:00+01:00
{
  "von": "2026-12-17T02:00:00+00:00",
  "bis": "2026-12-17T06:00:00+00:00",
  "statistik": {
    "zeilen": 5,
    "fehlversuche": 4,
    "fremd": 1,
    "sshd_sonst": 0,
    "zeit_ungueltig": 0
  },
  "pro_stunde": {
    "2026-12-17T02:00:00+00:00": 2,
    "2026-12-17T03:00:00+00:00": 1,
    "2026-12-17T04:00:00+00:00": 0,
    "2026-12-17T05:00:00+00:00": 1
  },
  "adressen": [
    [
      "203.0.113.5",
      3
    ]
  ],
  "benutzer": [
    [
      "root",
      2
    ]
  ],
  "warnung": null
}
[Exit-Code 0]

$ sshfails auth.log --seit zwei Stunden --zone Europe/Vienna --jetzt 2026-12-17T07:00:00+01:00
Fehler: Ungültige Dauer 'zwei Stunden' (erlaubt: 30m, 24h, 7d)
[Exit-Code 10]

$ sshfails fehlt.log --zone Europe/Vienna --jetzt 2026-12-17T07:00:00+01:00
Fehler: fehlt.log: No such file or directory
[Exit-Code 10]
```

Der erste Aufruf liest beide Dateien (die komprimierte Datei wird erkannt und entpackt) und zeigt acht Stunden mit den Leerstellen. Der Fehlversuch um 23:30 Uhr am Vortag steht in der ersten Zeile, weil das Fenster von 23:00 Uhr an reicht. Der zweite Aufruf verkürzt das Fenster auf vier Stunden und setzt die Schwelle auf 3: Die Adresse `203.0.113.5` erreicht genau 3 Fehlversuche, deshalb erscheint die Warnung und der Exit-Code ist 1; die Zahl der Ränge ist auf 1 begrenzt. Im dritten Aufruf liefert die Standardeingabe die Daten, und das Ergebnis erscheint als JSON, das man direkt an ein anderes Programm oder eine Datei weitergeben kann; die Zeiten stehen darin in UTC. Die beiden letzten Aufrufe zeigen die Fehlerfälle: Beide enden mit Exit-Code 10 und einer verständlichen Meldung, nicht mit einem Traceback. Das ist das Ziel: Ein Anwender, der sich vertippt, bekommt eine Erklärung, keinen Stapel von Funktionsnamen.

**Im Alltag aufrufen.** Auf einem echten Server rufen Sie das Programm in der Shell auf. Die folgenden Zeilen zeigen typische Aufrufe; sie werden hier nicht ausgeführt, weil sie ein echtes System voraussetzen, und müssen an Ihre Dateinamen und Ihre Distribution angepasst werden. Beachten Sie die letzten beiden Beispiele: Die Pipe mit `zcat` liest mehrere komprimierte Dateien auf einmal, und der Exit-Code in Verbindung mit `||` oder `if` macht aus dem Bericht eine Überwachung.

```text
$ python3 sshfails.py /var/log/auth.log --seit 24h --zone Europe/Vienna
$ python3 sshfails.py /var/log/auth.log /var/log/auth.log.1 --seit 7d --top 10
$ zcat /var/log/auth.log.*.gz | python3 sshfails.py - --seit 30d --format json > fehlversuche.json
$ python3 sshfails.py /var/log/auth.log --seit 1h --schwelle 20 || echo "Auffällig!"
$ python3 sshfails.py /var/log/auth.log --seit 1h --schwelle 20; echo "Exit-Code: $?"
```

Der Aufruf mit `||` führt den `echo`-Befehl nur aus, wenn das Programm mit einem Exit-Code ungleich 0 endet. In einem Cron-Job oder einem systemd-Timer (Modul 9) kann dieselbe Auswertung zeitgesteuert laufen, und die Meldung geht dann per Mail oder Webhook an das Team (Modul 12). Wichtig: Cron verwendet eine minimale Umgebung; verwenden Sie dort vollständige Pfade zum Programm und zum Interpreter. Das Lesen von `/var/log/auth.log` erfordert auf den meisten Systemen erhöhte Rechte (Mitglied einer Gruppe wie `adm` oder Root); das gehört zum Betriebssystem und nicht zum Programm. Diese Angaben beruhen auf allgemeinem Fachwissen und wurden nicht nachgeschlagen; ob `journalctl` Ihrer Version Ausgaben mit ISO-Zeitstempeln über die Option `-o short-iso` liefert, prüfen Sie bitte mit `man journalctl`.

**Was noch fehlt.** Das Programm ist ein solides Gerüst, aber noch kein fertiges Produkt. Es kennt nur Fehlversuche per `Failed`; Anmeldungen als `root`, Angriffe über andere Wege oder gesperrte Adressen (Fail2ban schreibt eigene Logs) werden nicht gezählt. Es gibt keine Konfigurationsdatei (siehe 7.5: Schwelle, Zone und Dateien könnten dort stehen), kein Logging der eigenen Arbeit (Modul 6) und keine automatisierten Tests (Modul 14). Auch die Stunden sind auf UTC-Stunden gerundet. Es ist eine gute Übung, diese Erweiterungen selbst vorzunehmen; einige sind Aufgaben am Ende dieses Moduls.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler beim Abschluss eines solchen Werkzeugs sind unspektakulär. Der erste ist, `datetime.now()` im Programm fest einzubauen, sodass sich nichts reproduzierbar testen lässt. Der zweite sind Fehlermeldungen als Traceback statt als lesbarer Text; er entsteht, wenn man die erwartbaren Ausnahmen (fehlende Datei, falsche Eingabe) nicht abfängt. Der dritte ist die Verwechslung von Exit-Codes: Wer einen Fehler mit 1 meldet und eine Schwellwertwarnung ebenfalls mit 1, kann beides nicht unterscheiden; deshalb haben wir 10 für Eingabefehler vorgesehen. Der vierte ist die Annahme, dass ein Programm, das gestern lief, auch auf dem Server von morgen läuft: andere Python-Version, andere Zeitzonendaten, andere Meldungsformate. Das folgende Beispiel zeigt zwei dieser Fälle (falsche Eingaben und ein Format, das nicht erkannt wird), damit Sie sehen, dass das Programm sie anständig meldet.

```python
import contextlib
import io

import sshfails

def lauf(*argv, stdin=None):
    puffer = io.StringIO()
    with contextlib.redirect_stdout(puffer), contextlib.redirect_stderr(puffer):
        code = sshfails.main(list(argv), stdin=stdin)
    return code, puffer.getvalue().splitlines()

grund = ["--jetzt", "2026-12-17T07:00:00+01:00"]

code, ausgabe = lauf("-", "--zone", "Mars/Olympus", *grund, stdin=io.StringIO(""))
print(code, ausgabe)

code, ausgabe = lauf("-", "--seit", "0h", *grund, stdin=io.StringIO(""))
print(code, ausgabe)

code, ausgabe = lauf("-", "--jetzt", "2026-12-17T07:00:00", stdin=io.StringIO(""))
print(code, ausgabe)

# Ein Log in einem unbekannten Format: das Programm meldet 0 Fehlversuche, aber die Statistik verrät es
fremdes_format = io.StringIO("17.12.2026 03:12:45 sshd: Anmeldung für root von 203.0.113.5 fehlgeschlagen\n")
code, ausgabe = lauf("-", "--seit", "4h", *grund, stdin=fremdes_format)
print(code, ausgabe[1])
```

```text
10 ["Fehler: Unbekannte Zeitzone 'Mars/Olympus'"]
10 ["Fehler: Ungültige Dauer '0h' (erlaubt: 30m, 24h, 7d)"]
10 ['Fehler: --jetzt braucht eine Zeitzone, z. B. 2026-12-17T07:00:00+01:00']
0 Zeilen: 1, Fehlversuche: 0, fremd: 1, sonstige sshd: 0, ungültige Zeit: 0
```

Die ersten drei Aufrufe enden mit dem Exit-Code 10 und einer verständlichen Fehlermeldung. Der vierte zeigt die gefährlichste Fehlerart: Das Programm läuft ohne Fehler, liefert Exit-Code 0 und meldet *keine Fehlversuche*, weil das Format der Zeile nicht erkannt wurde; die Statistik zeigt aber, dass die eine Zeile als `fremd` gezählt wurde. Wer die Statistik ignoriert, glaubt, der Server sei sicher. Für den Dauerbetrieb lohnt sich deshalb eine zusätzliche Regel: Ist der Anteil der fremden Zeilen unerwartet hoch, soll das Programm ebenfalls warnen.

**Selbsttest.** Das Quiz zu diesem Modul besteht aus den Aufgaben 1 bis 4 im folgenden Abschnitt (Fragen zu Regex, Gruppen, `datetime` und Logparsern) und den vier Programmieraufgaben 5 bis 8. Lösen Sie die Quizfragen zunächst ohne Nachschlagen; wenn Sie bei einer Frage unsicher sind, lesen Sie die zugehörige Lerneinheit noch einmal. Die Programmieraufgaben können Sie in einer eigenen virtuellen Umgebung mit `sshlog.py` und `sshfails.py` im selben Verzeichnis nachbauen.

> **Ergänzung aus der Nachrecherche**
>
> **Verwendete Module der Standardbibliothek.** Dieses Programm verwendet ausschließlich die Standardbibliothek von Python: `argparse`, `gzip`, `json`, `re`, `collections`, `datetime` und `zoneinfo`. Unter Windows benötigt `zoneinfo` das Paket `tzdata` (siehe 8.3; dort aus der Python-Dokumentation abgerufen). Die Exit-Code-Konvention (0, 1, 10) ist eine Entscheidung dieses Skripts; in der Überwachungswelt ist die Konvention 0 (OK), 1 (Warnung), 2 (kritisch), 3 (unbekannt) verbreitet, die wir in Modul 5 kennengelernt haben (aus Fachwissen, nicht nachgeschlagen). Das Skript meldet die Schwelle mit 1 und Eingabefehler mit 10, um beides unterscheiden zu können; wer es in ein Überwachungssystem einbindet, passt die Codes an dessen Erwartung an.

> **Merke**
>
> Ein Werkzeug entsteht aus Bausteinen plus Kommandozeile, Exit-Codes, Fehlermeldungen und Testbarkeit (`stdin` und `jetzt` als Parameter). Erwartbare Fehler in lesbare Meldungen mit Exit-Code 10 übersetzen. Die Statistik der nicht erkannten Zeilen ist das Frühwarnsystem gegen veränderte Logformate. Logdateien können ungültige Zeichen enthalten (`errors="replace"`). Den Exit-Code der Schwelle von Fehlern unterscheidbar halten.

> **Praxisbeispiel**
>
> Genau diese Form hat das typische Administrationsskript: Es liest Dateien oder die Standardeingabe, wertet ein Zeitfenster aus, schreibt einen Bericht für Menschen und eine Ausgabe für Maschinen und meldet über den Exit-Code, ob etwas auffällig ist. Als Cron-Job oder Timer läuft es täglich, schickt bei Überschreiten der Schwelle eine Nachricht (Modul 12) und wird im Repository mit Tests versioniert (Modul 13 und 14). Schon der Parser allein spart Zeit: Statt eine Datei von Hand durchzusehen, liefert er in einer Sekunde die Zahl, die in der nächsten Besprechung gebraucht wird.

## Übungsaufgaben zu Modul 08

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis (das Quiz), die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Für die Aufgabe 8 benötigen Sie das Modul `sshlog` aus 8.4.

**Aufgabe 1.** Worin unterscheiden sich `re.search`, `re.match` und `re.fullmatch`? Warum schreibt man Muster als Raw-String, und was passiert bei `re.findall("\bport\b", text)` ohne `r`? Was bedeutet „gierig“, und wie macht man einen Quantor genügsam? Warum passt das Muster `10.0.0.1` auch auf `10x0y0z1`, und wie behebt man das? Wann sind String-Methoden besser als reguläre Ausdrücke?

**Aufgabe 2.** Was liefert `re.findall` ohne Gruppe, mit einer Gruppe und mit mehreren Gruppen im Muster? Wozu dient `(?:...)`? Was ist der Unterschied zwischen `(\w+)` und `(?P<name>\w+)`, und wie liest man beide aus? Was liefert eine optionale Gruppe, die nicht teilnimmt? Wie arbeitet `re.sub` mit einer Funktion als Ersatz? Wozu dient `re.VERBOSE`?

**Aufgabe 3.** Was ist der Unterschied zwischen einem naiven und einem zeitzonenbewussten `datetime`, und warum erlaubt Python den Vergleich beider nicht? Warum gilt `datetime.utcnow()` als veraltet, und was verwendet man stattdessen? Was ist der Unterschied zwischen `.seconds` und `.total_seconds()`? Was ist der Unterschied zwischen `replace(tzinfo=...)` und `astimezone(...)`? Warum ist ein „Tag später“ nicht immer 24 Stunden später, und wann braucht man unter Windows das Paket `tzdata`?

**Aufgabe 4.** Finden Sie mindestens sechs Fehler im folgenden Programm und erklären Sie jeden.

```python
import re
from datetime import datetime

muster = "Failed password for (\w+) from (\d+.\d+.\d+.\d+)"
zaehler = {}

for zeile in open("auth.log"):
    m = re.match(muster, zeile)
    benutzer = m.group(1)
    zeit = datetime.strptime(zeile[:15], "%b %d %H:%M:%S")
    zaehler[zeit.hour] += 1

print(zaehler)
print("Dauer:", datetime.utcnow() - zeit.seconds)
```

**Aufgabe 5.** Schreiben Sie eine Funktion `parse_zugriff(zeile)`, die eine Zeile im Combined Log Format zerlegt (Muster aus 8.2) und ein Dictionary mit `ip`, `zeit` (zeitzonenbewusster `datetime`), `methode`, `pfad`, `status` (Zahl) und `groesse` (Zahl, bei `-` gleich 0) liefert, oder `None`, wenn die Zeile nicht passt. Werten Sie damit eine kleine Liste von Zeilen aus: Zählen Sie die Antworten mit Statuscodes ab 500 pro Stunde (in UTC) und geben Sie die Zahl der nicht erkannten Zeilen aus.

**Aufgabe 6.** Schreiben Sie eine Funktion `lies_syslog_zeit(text, jahr)`, die Syslog-Zeiten der Form `Dec 17 03:12:45` (auch mit einstelligem Tag, `Mar  5 07:00:01`) ohne die Hilfe von `%b` liest, also unabhängig von der Spracheinstellung des Rechners. Verwenden Sie dazu ein Dictionary für die englischen Monatsnamen und einen regulären Ausdruck. Ungültige Monate, Tage und Zeiten müssen mit einem `ValueError` abgelehnt werden.

**Aufgabe 7.** Ein Sicherungsprogramm schreibt bei jedem Lauf eine Zeile mit einem ISO-Zeitstempel in ein Protokoll. Schreiben Sie eine Funktion `luecken(zeitpunkte, pause)`, die aus einer Liste von Zeitpunkten (zeitzonenbewusst, unsortiert) alle Zeiträume liefert, in denen zwischen zwei aufeinanderfolgenden Läufen *mehr* als `pause` vergangen ist (Anfang und Ende der Lücke). Testen Sie mit täglichen Läufen, von denen zwei fehlen, und rechnen Sie in UTC, auch wenn die Eingaben in verschiedenen Zeitzonen vorliegen.

**Aufgabe 8.** Erweitern Sie den Logparser aus 8.4 um eine sicherheitsrelevante Auswertung: Liefern Sie alle Adressen, von denen im Log zuerst mindestens `schwelle` Fehlversuche kamen und anschließend eine *erfolgreiche* Anmeldung (`Accepted ...`), mit Zeitpunkt der Anmeldung und dem Benutzernamen. Verwenden Sie `sshlog.KOPF` und `sshlog.parse_zeit` und achten Sie auf die zeitliche Reihenfolge. Testen Sie mit einer kleinen erfundenen Logdatei.

## Lösungen zu Modul 08

**Aufgabe 1.** `re.search` sucht das Muster irgendwo im Text, `re.match` nur am Anfang, `re.fullmatch` verlangt, dass der ganze Text zum Muster passt. Muster schreibt man als Raw-String (`r"..."`), weil der Rückwärtsschrägstrich in normalen Zeichenketten eine eigene Bedeutung hat: In `"\bport\b"` ist `\b` das Backspace-Zeichen und nicht die Wortgrenze, sodass `findall` nichts findet. Ein gieriger Quantor (`*`, `+`, `?`, `{m,n}`) nimmt so viel Text wie möglich; ein angehängtes `?` (`*?`) macht ihn genügsam, und oft ist eine negierte Klasse (`[^>]*`) noch besser. Das Muster `10.0.0.1` passt auf `10x0y0z1`, weil der Punkt ein Platzhalter für ein beliebiges Zeichen ist; man entschärft ihn mit `\.` oder bei fremden Texten mit `re.escape`. String-Methoden (`in`, `startswith`, `split`, `replace`) sind besser, wenn man einen festen Text sucht; sie sind einfacher und in der Regel schneller. Außerdem sollte man HTML und XML nicht mit Regex auswerten, sondern mit einem Parser.

**Aufgabe 2.** Ohne Gruppe liefert `findall` die ganzen Treffer; mit genau einer Gruppe nur deren Inhalt; mit mehreren Gruppen eine Liste von Tupeln. `(?:...)` ist eine nicht merkende Gruppe: Sie fasst nur zusammen (für eine Alternative oder Wiederholung), ohne den Inhalt zu speichern und ohne das Ergebnis von `findall` zu verändern. Die Gruppe `(\w+)` erreicht man über die Nummer (`m.group(1)`), `(?P<name>\w+)` zusätzlich über den Namen (`m.group("name")`, `m["name"]`, `m.groupdict()`). Eine optionale Gruppe, die nicht teilnimmt, hat den Wert `None`, nicht den leeren Text; darauf muss man vor `int()` prüfen. Als Ersatz von `re.sub` darf man eine Funktion übergeben, die für jeden Treffer das Match-Objekt erhält und den Ersatztext zurückgibt. `re.VERBOSE` erlaubt Leerraum und Kommentare im Muster, und damit lange Muster lesbar zu halten.

**Aufgabe 3.** Ein naiver `datetime` trägt keine Zeitzone, ein bewusster hat ein `tzinfo` mit dem Abstand zu UTC. Python vergleicht beide nicht und löst einen `TypeError` aus, weil nicht klar ist, in welcher Zeitzone der naive Wert gemeint ist; Raten würde stille Fehler erzeugen. `datetime.utcnow()` liefert trotz des Namens einen naiven Wert; seit Python 3.12 ist die Funktion als veraltet gekennzeichnet; man verwendet `datetime.now(timezone.utc)`. `.seconds` ist nur der Sekundenanteil nach den vollen Tagen eines `timedelta`; die gesamte Dauer liefert `.total_seconds()`. `replace(tzinfo=z)` *beschriftet* einen Wert mit einer Zeitzone, ohne die Uhrzeit zu ändern; es erzeugt also einen anderen Zeitpunkt, wenn der Wert vorher eine andere Zone hatte. `astimezone(z)` *rechnet* einen bewussten Zeitpunkt in die andere Zone um und lässt den Zeitpunkt unverändert. Innerhalb einer Zeitzone rechnet Python mit „gleicher Uhrzeit am nächsten Kalendertag“; wegen der Sommerzeit können das 23 oder 25 Stunden sein, deshalb rechnet man verstrichene Zeit in UTC. Das Paket `tzdata` braucht man auf Systemen ohne eigene Zeitzonendatenbank, vor allem unter Windows, sonst löst `ZoneInfo` den Fehler `ZoneInfoNotFoundError` aus.

**Aufgabe 4.** (1) Das Muster ist kein Raw-String (`\w` und `\d` sind in normalen Zeichenketten unbekannte Escape-Folgen und lösen eine Warnung aus). (2) Die Punkte in der Adresse sind unentschärft (Platzhalter für beliebige Zeichen). (3) `re.match` prüft nur den Anfang der Zeile, aber die Zeile beginnt mit der Zeit, nicht mit `Failed`; richtig ist `re.search`. (4) Das Ergebnis `m` kann `None` sein; `m.group(1)` stürzt dann mit `AttributeError` ab. (5) `zeile[:15]` schneidet die Zeit ab, aber `strptime` ohne Jahr nimmt das Jahr 1900 an (und lehnt den 29. Februar ab); man muss das Jahr ergänzen. (6) `zaehler[zeit.hour] += 1` löst einen `KeyError` aus, weil der Schlüssel noch nicht existiert; man nutzt `collections.Counter` oder `dict.get`. (7) Die Datei wird mit `open` nie geschlossen und ohne `encoding` geöffnet; besser mit `with`. (8) Die Uhrzeit ist naiv, ihre Zeitzone ist nicht angegeben, und die Stunde ist daher nicht eindeutig. (9) `datetime.utcnow() - zeit.seconds` ist mehrfach falsch: `utcnow()` ist veraltet und liefert einen naiven Wert; `.seconds` ist ein Attribut eines `timedelta`, nicht eines `datetime`, und selbst als Zahl ließe sich eine Zahl nicht von einem `datetime` abziehen (`TypeError`). Gemeint war vermutlich die Differenz zweier Zeitpunkte, also ein `timedelta`. (10) Die Variable `benutzer` wird nie verwendet.

**Aufgabe 5.**

Die Funktion wendet das Muster aus 8.2 an, wandelt Status und Größe in Zahlen um und liest die Zeit mit `strptime` samt `%z`, sodass der Zeitpunkt bewusst ist. Zum Zählen wird der Zeitpunkt in UTC umgerechnet und auf die Stunde abgerundet.

```python
import re
from collections import Counter
from datetime import datetime, timezone

ZUGRIFF = re.compile(r'''
    ^(?P<ip>\S+)\s+\S+\s+\S+\s+
    \[(?P<zeit>[^\]]+)\]\s+
    "(?P<methode>[A-Z]+)\s+(?P<pfad>\S+)\s+[^"]+"\s+
    (?P<status>\d{3})\s+(?P<groesse>\d+|-)
''', re.VERBOSE)

def parse_zugriff(zeile):
    m = ZUGRIFF.match(zeile)
    if m is None:
        return None
    d = m.groupdict()
    try:
        d["zeit"] = datetime.strptime(d["zeit"], "%d/%b/%Y:%H:%M:%S %z")
    except ValueError:
        return None
    d["status"] = int(d["status"])
    d["groesse"] = 0 if d["groesse"] == "-" else int(d["groesse"])
    return d

zeilen = [
    '203.0.113.5 - - [17/Dec/2026:03:12:45 +0100] "GET /index.html HTTP/1.1" 200 5120 "-" "curl/8.5.0"',
    '203.0.113.5 - - [17/Dec/2026:03:40:01 +0100] "POST /api HTTP/1.1" 500 0 "-" "curl/8.5.0"',
    '198.51.100.7 - - [17/Dec/2026:03:59:59 +0100] "GET /x HTTP/1.1" 502 - "-" "Mozilla/5.0"',
    '198.51.100.7 - - [17/Dec/2026:04:00:00 +0100] "GET /y HTTP/1.1" 503 17 "-" "Mozilla/5.0"',
    'das ist kein Zugriffsprotokoll',
]

fehler_pro_stunde = Counter()
nicht_erkannt = 0
for zeile in zeilen:
    eintrag = parse_zugriff(zeile)
    if eintrag is None:
        nicht_erkannt += 1
    elif eintrag["status"] >= 500:
        stunde = eintrag["zeit"].astimezone(timezone.utc).replace(minute=0, second=0, microsecond=0)
        fehler_pro_stunde[stunde] += 1

for stunde, anzahl in sorted(fehler_pro_stunde.items()):
    print(stunde.strftime("%Y-%m-%d %H:00 UTC"), anzahl)
print("Nicht erkannte Zeilen:", nicht_erkannt)
```

```text
2026-12-17 02:00 UTC 2
2026-12-17 03:00 UTC 1
Nicht erkannte Zeilen: 1
```

Die Zeilen mit `03:40` und `03:59:59` (Ortszeit mit `+0100`) liegen in UTC um 02 Uhr, die Zeile mit `04:00:00` um 03 Uhr; deshalb stehen in der Stunde 02 zwei Fehler und in der Stunde 03 einer. Der Zugriff mit Status 200 wird nicht gezählt, die letzte Zeile erscheint in der Zahl der nicht erkannten Zeilen. Das Umrechnen *vor* dem Abrunden ist wichtig: Wer die Stunde der Ortszeit abrundet, vermischt Zeitzonen, sobald Zeilen mit unterschiedlichem Abstand im Log stehen.

**Aufgabe 6.**

Statt `%b` verwendet die Lösung ein Dictionary, das die englischen Monatsnamen auf Zahlen abbildet. Ob ein Tag oder eine Uhrzeit gültig ist, prüft der Konstruktor `datetime(...)`, der bei ungültigen Werten (30. Februar, 25 Uhr) einen `ValueError` auslöst.

```python
import re
from datetime import datetime

MONATE = {name: nummer for nummer, name in enumerate(
    ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"], start=1)}
SYSLOG_ZEIT = re.compile(r"^([A-Z][a-z]{2})\s+(\d{1,2})\s+(\d\d):(\d\d):(\d\d)$")

def lies_syslog_zeit(text, jahr):
    m = SYSLOG_ZEIT.match(text.strip())
    if m is None:
        raise ValueError(f"Kein Syslog-Zeitstempel: {text!r}")
    monat = MONATE.get(m[1])
    if monat is None:
        raise ValueError(f"Unbekannter Monat: {m[1]!r}")
    tag, stunde, minute, sekunde = (int(x) for x in m.groups()[1:])
    return datetime(jahr, monat, tag, stunde, minute, sekunde)

for text in ("Dec 17 03:12:45", "Mar  5 07:00:01", "Feb 29 12:00:00", "Dez 17 03:12:45", "Dec 17 25:00:00", "17.12.2026"):
    try:
        print(f"{text!r:20}", lies_syslog_zeit(text, 2028))
    except ValueError as fehler:
        print(f"{text!r:20}", "ValueError:", fehler)
```

```text
'Dec 17 03:12:45'    2028-12-17 03:12:45
'Mar  5 07:00:01'    2028-03-05 07:00:01
'Feb 29 12:00:00'    2028-02-29 12:00:00
'Dez 17 03:12:45'    ValueError: Unbekannter Monat: 'Dez'
'Dec 17 25:00:00'    ValueError: hour must be in 0..23
'17.12.2026'         ValueError: Kein Syslog-Zeitstempel: '17.12.2026'
```

Das Jahr 2028 ist ein Schaltjahr, daher ist der 29. Februar gültig. Der deutsche Monatsname `Dez` wird abgelehnt, weil er nicht im Wörterbuch steht, `25:00:00` vom Konstruktor von `datetime`, und das Datum im deutschen Format passt gar nicht zum Muster. Die Funktion hängt nicht von der Locale ab und liefert auf jedem Rechner dasselbe Ergebnis. Ein Nebenergebnis ist ein einfaches Muster für den Umgang mit Tupeln: `m.groups()[1:]` lässt die erste Gruppe (den Monatsnamen) weg.

**Aufgabe 7.**

Die Funktion sortiert die Zeitpunkte nach ihrer Umrechnung in UTC und vergleicht jeweils zwei benachbarte. Ist der Abstand größer als die erlaubte Pause, wird die Lücke als Paar (Anfang, Ende) gemerkt. Die Rechnung in UTC macht die Lösung unabhängig davon, ob die Eingaben in der Ortszeit oder in anderen Zeitzonen stehen.

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

def luecken(zeitpunkte, pause):
    utc = sorted(z.astimezone(timezone.utc) for z in zeitpunkte)
    ergebnis = []
    for vorher, nachher in zip(utc, utc[1:]):
        if nachher - vorher > pause:
            ergebnis.append((vorher, nachher))
    return ergebnis

wien = ZoneInfo("Europe/Vienna")
laeufe = [
    datetime(2026, 12, 14, 2, 0, tzinfo=wien),
    datetime(2026, 12, 15, 2, 0, tzinfo=wien),
    datetime(2026, 12, 18, 1, 0, tzinfo=timezone.utc),     # 02:00 Ortszeit, aber in UTC angegeben
    datetime(2026, 12, 19, 2, 0, tzinfo=wien),
    datetime(2026, 12, 17, 2, 0, tzinfo=wien),
]

for anfang, ende in luecken(laeufe, timedelta(hours=25)):
    print("Lücke:", anfang.astimezone(wien).strftime("%d.%m. %H:%M"), "bis", ende.astimezone(wien).strftime("%d.%m. %H:%M"),
          "=", (ende - anfang).total_seconds() / 3600, "Stunden")
print("Ohne Lücken:", luecken(laeufe[:2], timedelta(hours=25)))
```

```text
Lücke: 15.12. 02:00 bis 17.12. 02:00 = 48.0 Stunden
Ohne Lücken: []
```

Die Läufe sind täglich um 02:00 Uhr Ortszeit angesetzt; der 16. Dezember fehlt, und die Zeit am 18. Dezember ist in UTC angegeben (01:00 UTC ist 02:00 in Wien). Die Eingabe ist nicht sortiert; die Funktion sortiert selbst. Die Lücke vom 15. auf den 17. Dezember dauert 48 Stunden und ist größer als die Pause von 25 Stunden (eine Stunde Toleranz zu den 24 Stunden eines Tages, damit leichte Verschiebungen und die Zeitumstellung keinen Fehlalarm auslösen). Alle anderen Abstände betragen genau 24 Stunden. Das Beispiel nutzt `zip(utc, utc[1:])`, um benachbarte Paare zu bilden; man liest es als „jedes Element zusammen mit seinem Nachfolger“.

**Aufgabe 8.**

Die Lösung liest die Zeilen in der Reihenfolge, in der sie in der Datei stehen (ein Log ist zeitlich geordnet), und führt je Adresse einen Zähler für Fehlversuche. Kommt eine `Accepted`-Zeile, und hat die Adresse bis dahin mindestens `schwelle` Fehlversuche, wird sie mit dem Zeitpunkt und dem Benutzernamen gemeldet. Wichtig ist, dass Fehlversuche *nach* der erfolgreichen Anmeldung nicht mitzählen; das ergibt sich aus der Reihenfolge der Verarbeitung.

```python
import re
from collections import Counter
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

import sshlog

ERFOLG = re.compile(r"^Accepted (?P<methode>\S+) for (?P<benutzer>\S+) from (?P<ip>\S+) port (?P<port>\d+)")

def erfolg_nach_angriff(zeilen, schwelle, jetzt, zone):
    fehler = Counter()
    treffer = []
    for zeile in zeilen:
        kopf = sshlog.KOPF.match(zeile.rstrip("\n"))
        if kopf is None:
            continue
        try:
            zeit = sshlog.parse_zeit(kopf["zeit"], jetzt, zone)
        except ValueError:
            continue
        versuch = sshlog.FEHLVERSUCH.match(kopf["meldung"])
        if versuch:
            fehler[versuch["ip"]] += 1
            continue
        erfolg = ERFOLG.match(kopf["meldung"])
        if erfolg and fehler[erfolg["ip"]] >= schwelle:
            treffer.append((erfolg["ip"], erfolg["benutzer"], zeit, fehler[erfolg["ip"]]))
    return treffer

LOG = """\
Dec 17 03:00:01 srv sshd[1]: Failed password for root from 203.0.113.5 port 1 ssh2
Dec 17 03:00:05 srv sshd[1]: Failed password for root from 203.0.113.5 port 2 ssh2
Dec 17 03:00:09 srv sshd[1]: Failed password for invalid user admin from 203.0.113.5 port 3 ssh2
Dec 17 03:00:12 srv sshd[1]: Accepted password for admin from 203.0.113.5 port 4 ssh2
Dec 17 03:05:00 srv sshd[2]: Accepted publickey for anna from 198.51.100.7 port 5 ssh2
Dec 17 03:06:00 srv sshd[3]: Failed password for bob from 192.0.2.44 port 6 ssh2
Dec 17 03:06:30 srv sshd[3]: Accepted password for bob from 192.0.2.44 port 7 ssh2
Dec 17 03:07:00 srv sudo: anna : COMMAND=/bin/ls
"""

wien = ZoneInfo("Europe/Vienna")
jetzt = datetime(2026, 12, 17, 12, 0, tzinfo=timezone.utc)
for ip, benutzer, zeit, anzahl in erfolg_nach_angriff(LOG.splitlines(), 3, jetzt, wien):
    print(f"{ip}: {anzahl} Fehlversuche, dann Anmeldung als {benutzer} um {zeit.astimezone(wien).strftime('%H:%M:%S')}")
```

```text
203.0.113.5: 3 Fehlversuche, dann Anmeldung als admin um 03:00:12
```

Nur die erste Adresse hat mindestens drei Fehlversuche und danach eine erfolgreiche Anmeldung; bei `192.0.2.44` war es nur einer, bei `198.51.100.7` gar keiner. Dieses Muster („viele Fehlversuche, dann Erfolg“) ist ein typisches Anzeichen dafür, dass ein Angreifer ein Passwort erraten hat, und gehört in der Praxis zu den Warnungen, die man nicht übersehen will. Dass der Benutzername `admin` im Fehlversuch als *ungültiger* Benutzer erschien und danach erfolgreich angemeldet wurde, wäre in einem echten Log ein zusätzliches Warnsignal (ein Benutzer, den es erst gab oder der zwischenzeitlich angelegt wurde); das Programm beobachtet es nicht, aber Sie könnten es ergänzen. Die Lösung benutzt die Muster und die Zeitfunktion aus `sshlog` wieder: Das ist der Nutzen von Funktionen und Modulen aus Modul 7.
