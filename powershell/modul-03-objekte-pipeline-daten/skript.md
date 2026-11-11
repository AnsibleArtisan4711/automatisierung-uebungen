# Modul 03 - Objekte, Pipeline und Datenverarbeitung

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (3.1 bis 3.9; 3.10 Übungsaufgaben, 3.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Übersichtsblatt (10 h) und Detailblatt (9 h) weichen um eine Stunde voneinander ab. Dieses Skript folgt dem Detailblatt, weil jede Lerneinheit dort einzeln aufgeführt ist.

### Lernziele.

Nach diesem Modul können Sie Objekte filtern, sortieren, gruppieren und exportieren. Sie lesen Objekte strukturiert, wählen Eigenschaften gezielt aus, filtern mit Bedingungen, fassen Daten zusammen, transformieren Pipeline-Elemente, speichern und laden Daten in passenden Formaten, trennen Daten von ihrer Darstellung und lesen Text mit regulären Ausdrücken. Dabei beachten Sie Zeichenkodierung und Kultureinstellungen, damit Berichte auf jedem System gleich aussehen. Sicherheitsaspekte und Betriebspraxis begleiten die Beispiele, und das Ziel bleibt wiederholbare Automation für Windows, Cloud und CI/CD.

| **Nr.** | **Lerneinheit**              | **Dauer** | **Prio** |
|:--------|:-----------------------------|:----------|:---------|
| 3.1     | Objekte und Properties       | 1 h       | 1        |
| 3.2     | Select-Object                | 1 h       | 1        |
| 3.3     | Where-Object                 | 1 h       | 1        |
| 3.4     | Sort-Object und Group-Object | 1 h       | 1        |
| 3.5     | ForEach-Object               | 1 h       | 1        |
| 3.6     | Import und Export            | 1 h       | 1        |
| 3.7     | Formatierung vs Daten        | 1 h       | 2        |
| 3.8     | Reguläre Ausdrücke           | 1 h       | 2        |
| 3.9     | Encoding und Kultur          | 1 h       | 2        |
|         | **Summe**                    | **9 h**   |          |

### Beispieldaten.

Die meisten Beispiele dieses Moduls arbeiten mit einem kleinen Serverinventar aus sechs Einträgen. Es wird in jedem Listing, das es braucht, mit denselben drei Zeilen neu erzeugt, damit jedes Beispiel für sich lauffähig ist. Jede Zeichenkette enthält Name, Rolle, Arbeitsspeicher in Gigabyte und Standort; `ForEach-Object` zerlegt sie und baut daraus Objekte mit einem echten Zahlentyp für den Speicher.

## 3.1 Objekte und Properties

Alles, womit PowerShell arbeitet, ist ein Objekt: eine Datei, ein Prozess, eine Zeichenkette, eine Zahl, ein Datum, ein Fehler. Ein Objekt bündelt Daten und Verhalten. Die Daten stecken in *Eigenschaften* (Properties), die jeweils einen Namen, einen Wert und einen *Typ* haben. Das Verhalten steckt in *Methoden*, die man aufrufen kann. Zwei Objekte desselben Typs haben dieselben Eigenschaftsnamen, aber meist verschiedene Werte, ähnlich wie die Zeilen einer Tabelle dieselben Spalten haben. Diese Analogie ist hilfreich: Eine Pipeline, die Objekte weiterreicht, entspricht einer Tabelle, in der jede Zeile ein Objekt und jede Spalte eine Eigenschaft ist.

Eigenschaften lesen Sie mit dem Punkt-Operator: `$objekt.Name`. Der Typ eines Werts entscheidet darüber, wie er sich verhält. Eine Zahl wird nach Größe verglichen, ein Text alphabetisch, ein Datum chronologisch. Deshalb ist es so wichtig, Daten im *richtigen Typ* vorliegen zu haben; auf diese Frage kommen wir bei Sortierung und Dateiimport zurück. Aus einer Sammlung von Objekten lässt sich eine Eigenschaft für alle Elemente gleichzeitig abrufen: `$inventar.Name` liefert die Namen aller Einträge als Liste. Diese Kurzschreibweise heißt *Member Enumeration*.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$inventar[0].Name
($inventar.Name) -join ', '
$inventar.Count

$inventar | Add-Member -MemberType NoteProperty -Name Online -Value $true
$inventar[2].Online = $false
$inventar | Select-Object -Property Name, Online

$inventar[0].psobject.Properties |
    ForEach-Object { "$($_.Name) = $($_.Value) ($($_.TypeNameOfValue))" }

$inventar.RamGB | Measure-Object -Sum -Average -Maximum -Minimum
```

Der Block beginnt mit den Beispieldaten. Die sechs Zeichenketten werden an `ForEach-Object` übergeben; im Skriptblock zerlegt `-split ','` jede Zeichenkette in vier Teile, und `[pscustomobject]@{ ... }` erzeugt daraus ein Objekt mit benannten Eigenschaften. Beachten Sie `[int]$t[2]`: Der Text „16“ wird bewusst in eine Ganzzahl umgewandelt. Die nächsten Zeilen greifen auf die Eigenschaft `Name` des ersten Elements zu (`srv-web01`), listen mit Member Enumeration alle Namen auf und lesen die Anzahl der Elemente (`6`). `Add-Member` hängt allen Objekten eine neue Eigenschaft `Online` an, die auf `True` gesetzt wird; danach wird bei einem einzelnen Objekt der Wert verändert. Das ist möglich, weil Eigenschaften von selbst erzeugten Objekten beschreibbar sind. Die Auflistung über `psobject.Properties` zeigt Name, Wert und Typ jeder Eigenschaft des ersten Objekts, im Test etwa `RamGB = 16 (System.Int32)` und `Online = True (System.Boolean)`. Zuletzt berechnet `Measure-Object` Summe, Durchschnitt, Maximum und Minimum; im Test waren das 200, 33,33, 64 und 8 bei sechs Elementen.

Es gibt mehrere Wege, eigene Objekte zu erzeugen. Am kürzesten und lesbarsten ist `[pscustomobject]@{ ... }` mit einer Hashtable. Wenn die Reihenfolge der Eigenschaften wichtig ist, verwendet man `[ordered]@{ ... }` und wandelt das Ergebnis dann um. Ältere Skripte arbeiten mit `New-Object -TypeName PSObject -Property $hash`, das gleichwertig, aber umständlicher ist. Für größere Projekte definiert man eigene Klassen (Modul 4), die Typsicherheit bieten.

> **Ergänzung aus der Nachrecherche**
>
> **Typsicherheit mit dem Strict Mode.** Greifen Sie auf eine Eigenschaft zu, die nicht existiert, ergibt das normalerweise stillschweigend `$null`, ohne jede Fehlermeldung. Wenn Sie dagegen `Set-StrictMode -Version Latest` aktivieren, meldet PowerShell den Fehler „The property ’Gibtsnicht’ cannot be found on this object. Verify that the property exists.“ (so im Test). Tippfehler in Eigenschaftsnamen fallen so sofort auf; der Strict Mode wird in Modul 5 behandelt.

> **Merke**
>
> Eigenschaften werden mit `.Name` gelesen, Methoden mit `.Name()` aufgerufen. Member Enumeration (`$sammlung.Eigenschaft`) liefert die Eigenschaft aller Elemente. Der Typ des Werts bestimmt Vergleich und Sortierung. Eigene Objekte erzeugen Sie am einfachsten mit `[pscustomobject]@{ ... }`.

Ein häufiger Fehler ist, Zahlen als Text zu behandeln, weil sie aus einer Textquelle stammen. Der Wert „16“ aus einer CSV-Datei ist eine Zeichenkette, keine Zahl, und verhält sich bei Vergleich und Sortierung anders (Lerneinheiten 3.4 und 3.6). Ein zweiter Fehler betrifft verschachtelte Objekte: Eine Eigenschaft kann selbst ein Objekt mit eigenen Eigenschaften sein, etwa `$prozess.StartTime.Year`. Wer die Struktur nicht kennt, sieht in der Tabelle nur Typnamen statt Werten und sollte mit `Format-List *` oder `Get-Member` (Lerneinheit 2.3) nachsehen.

```powershell
# FALSCH: Zahl als Text speichern, später falsch sortieren oder vergleichen
$a = [pscustomobject]@{ Name = 'srv-web01'; RamGB = '16' }

# RICHTIG: Typ beim Erzeugen festlegen
$a = [pscustomobject]@{ Name = 'srv-web01'; RamGB = [int]16 }
$a.RamGB.GetType().Name
```

> **Praxisbeispiel**
>
> Wenn Sie in einem Inventarisierungsskript für jeden Server ein Objekt mit Name, Betriebssystem, Speicher und Online-Status erzeugen, können dieselben Daten später in einen Bericht, eine Datenbank oder ein Dashboard wandern, ohne dass Sie Code ändern müssen. Die Disziplin, *Objekte* statt formatierter Texte zu erzeugen, ist die Grundlage jeder wiederverwendbaren Automation.

## 3.2 Select-Object

`Select-Object` wählt Eigenschaften oder Objekte aus. Das Cmdlet hat zwei Aufgaben, die man gedanklich trennen sollte. Die erste ist das *Auswählen von Spalten*: Aus Objekten mit vielen Eigenschaften werden Objekte mit nur den gewünschten. Die zweite ist das *Auswählen von Zeilen*: Aus einer Menge von Objekten bleiben die ersten, letzten oder eindeutigen übrig. Außerdem kann es Eigenschaften *berechnen* und einzelne Werte *auspacken*.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$inventar | Select-Object -Property Name, RamGB -First 2
$inventar | Select-Object -Property Name, @{ Name = 'RamMB'; Expression = { $_.RamGB * 1024 } } -First 2
$inventar | Select-Object -Property Rolle -Unique
($inventar | Select-Object -ExpandProperty Name) -join ', '
$inventar | Select-Object -Skip 4
$inventar | Select-Object -Last 1
$inventar | Select-Object -Property * -ExcludeProperty Standort -First 1
```

Die erste Zeile behält von den ersten zwei Objekten nur `Name` und `RamGB`. Die zweite fügt eine *berechnete Eigenschaft* hinzu: Eine Hashtable mit den Schlüsseln `Name` (Spaltenname) und `Expression` (Skriptblock, der den Wert berechnet; `$_` ist das aktuelle Objekt). Statt `Name` und `Expression` dürfen die Kurzformen `n` und `e` stehen, was in älteren Skripten häufig ist. Das Ergebnis enthält `RamMB` mit dem Wert 16384. Die dritte Zeile liefert mit `-Unique` jede Rolle nur einmal: Web, Datenbank, Anwendung, Dateien. `-ExpandProperty Name` gibt keine Objekte mit einer Eigenschaft zurück, sondern die *Werte selbst*, im Beispiel also reine Zeichenketten. Mit `-Skip` überspringen Sie Elemente am Anfang (hier die ersten vier, es bleiben `srv-app01` und `srv-file01`), mit `-Last` nehmen Sie die letzten, mit `-First` die ersten. Die letzte Zeile wählt mit dem Platzhalter `*` alle Eigenschaften und schließt eine davon mit `-ExcludeProperty` aus.

Der Unterschied zwischen `-Property` und `-ExpandProperty` sorgt oft für Verwirrung. `-Property Name` liefert Objekte, die eine Eigenschaft `Name` besitzen; `-ExpandProperty Name` liefert den Wert dieser Eigenschaft, also etwa Text. Wollen Sie das Ergebnis an einen Befehl übergeben, der Text erwartet (zum Beispiel einen Pfad oder einen Namen), brauchen Sie `-ExpandProperty` oder die Kurzform `ForEach-Object Name`.

Wichtig ist außerdem, was `Select-Object` mit dem Objekt macht. Das Ergebnis ist ein *neues* Objekt, das nur die ausgewählten Eigenschaften enthält. Sein Typ ist `Selected.` gefolgt vom ursprünglichen Typnamen; im Test lautete er `Selected.System.Management.Automation.PSCustomObject`. Bei Objekten aus dem .NET-Framework gehen dabei die *Methoden* des Originals verloren, denn das neue Objekt ist nur ein Behälter für die kopierten Werte.

```powershell
$jetzt = Get-Date
$jetzt.AddDays(1).ToString('yyyy-MM-dd')

$nurJahr = Get-Date | Select-Object -Property Year, Month
$nurJahr | Get-Member -MemberType Method
```

Das ursprüngliche Datumsobjekt hat Methoden wie `AddDays`. Das reduzierte Objekt `$nurJahr` hat nur noch die Eigenschaften `Year` und `Month` und keine eigenen Methoden mehr (der Aufruf von `Get-Member` listet nur die Methoden, die jedes Objekt besitzt, wie `Equals` und `ToString`). Wer später ein Original-Objekt braucht, etwa zum Löschen einer Datei mit `.Delete()`, darf `Select-Object` nicht davorschalten.

> **Merke**
>
> `Select-Object` wählt Spalten (`-Property`, `-ExcludeProperty`), Zeilen (`-First`, `-Last`, `-Skip`, `-Unique`) und erzeugt berechnete Eigenschaften. `-ExpandProperty` liefert Werte statt Objekte. Das Ergebnis ist ein neues Objekt ohne die Methoden des Originals.

```powershell
# FALSCH: Objekte statt Text weitergeben, wo Text erwartet wird
$name = $inventar | Select-Object -Property Name -First 1
"Server: $name"          # zeigt @{Name=srv-web01}

# RICHTIG: Wert auspacken
$name = $inventar | Select-Object -ExpandProperty Name -First 1
"Server: $name"          # zeigt srv-web01
```

Im ersten Fall ist `$name` ein Objekt, und die Textumwandlung ergibt die sperrige Darstellung `@{Name=srv-web01}`. Dieser Fehler taucht regelmäßig auf, wenn Namen in Meldungen, Dateipfade oder Parameter eingesetzt werden.

> **Praxisbeispiel**
>
> Für eine Management-Übersicht aus `Get-Process` oder einem Cloud-Cmdlet reduzieren Sie die Objekte mit `Select-Object` auf wenige Spalten, benennen sie mit berechneten Eigenschaften sprechend („SpeicherMB“ statt `WorkingSet64`) und exportieren das Ergebnis. Ein Bericht mit sechs Spalten ist für Leser nützlicher als ein Dump mit sechzig.

## 3.3 Where-Object

`Where-Object` filtert Objekte nach einer Bedingung. Für jedes durch die Pipeline kommende Objekt wird die Bedingung ausgewertet, und nur Objekte, für die sie wahr ist, laufen weiter. Es gibt zwei Schreibweisen. Die *Skriptblock-Syntax* `Where-Object { $_.RamGB -ge 32 }` erlaubt beliebig komplexe Bedingungen. Die *vereinfachte Syntax* `Where-Object RamGB -ge 32` ist für einfache Vergleiche kürzer, kommt ohne Klammern und `$_` aus und erlaubt nur eine Bedingung.

Vergleiche werden mit *Operatoren* geschrieben, die mit einem Bindestrich beginnen. Die wichtigsten sind `-eq` (gleich), `-ne` (ungleich), `-gt` (größer), `-ge` (größer oder gleich), `-lt` (kleiner), `-le` (kleiner oder gleich), `-like` und `-notlike` (Vergleich mit Platzhaltern wie `*` und `?`), `-match` und `-notmatch` (reguläre Ausdrücke, Lerneinheit 3.8), `-in` und `-notin` (Wert ist in einer Liste) sowie `-contains` und `-notcontains` (Liste enthält Wert). Bedingungen werden mit `-and`, `-or`, `-not` (kurz: `!`) und `-xor` verknüpft. Textvergleiche sind standardmäßig *nicht* groß-/kleinschreibungsabhängig; wer das braucht, setzt ein `c` vor den Operator (`-ceq`, `-clike`, `-cmatch`).

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$inventar | Where-Object { $_.RamGB -ge 32 -and $_.Standort -eq 'Wien' }
$inventar | Where-Object -Property Rolle -EQ -Value 'Web'
$inventar | Where-Object Name -Like 'srv-db*'
$inventar | Where-Object Rolle -In 'Web', 'Dateien'
$inventar.Where({ $_.RamGB -lt 16 })
$inventar | Where-Object { $_.Name -match '^srv-(web|app)\d+$' }
```

Die erste Abfrage kombiniert zwei Bedingungen und liefert nur `srv-db01` (64 GB, Wien). Die zweite verwendet die vereinfachte Syntax mit ausgeschriebenen Parameternamen und liefert die beiden Webserver. Die dritte filtert mit Platzhalter alle Namen, die mit `srv-db` beginnen. Die vierte prüft mit `-In`, ob die Rolle in der Liste vorkommt, und liefert die Webserver sowie `srv-file01`. Die fünfte Zeile zeigt die *Methodensyntax* `.Where()`, die auf Sammlungen im Speicher arbeitet, etwas schneller als die Pipeline ist und hier `srv-file01` liefert. Die sechste filtert mit einem regulären Ausdruck alle Server, deren Name aus „srv-“, „web“ oder „app“ und einer Zahl besteht.

Vorsicht ist bei der Reihenfolge der Operanden geboten. Steht links eine *Liste*, liefern Vergleichsoperatoren nicht wahr oder falsch, sondern die passenden *Elemente*: `1,2,3,2 -eq 2` ergibt die zwei Werte `2 2`. In einer `if`-Bedingung reicht das als Wahrheitswert, in einer Zuweisung führt es zu Überraschungen. Auch der Typ des *linken* Operanden entscheidet: `'10' -gt '9'` ist falsch, weil Texte alphabetisch verglichen werden („1“ kommt vor „9“), `10 -gt '9'` ist dagegen wahr, weil PowerShell den rechten Operanden in eine Zahl umwandelt. Im Test bestätigten sich beide Ergebnisse.

> **Merke**
>
> Vergleichsoperatoren beginnen mit einem Bindestrich (`-eq`, `-gt`, `-like`, `-match`, `-in`). Das Zeichen `>` ist *kein* Vergleich, sondern eine Ausgabeumleitung in eine Datei. Der Typ des linken Operanden bestimmt, wie verglichen wird.

Der berüchtigtste Fehler ist die Verwendung von `>` oder `<` als Vergleich, wie es aus anderen Sprachen bekannt ist. In PowerShell bedeutet `$_ > 2` die Umleitung der Ausgabe in eine Datei mit dem Namen `2`. Das Ergebnis ist eine leere Ausgabe und eine unerwartete Datei im aktuellen Ordner; im Test entstand genau eine solche Datei.

```powershell
# FALSCH: Umleitung statt Vergleich, erzeugt eine Datei namens 2
1..3 | Where-Object { $_ > 2 }

# RICHTIG
1..3 | Where-Object { $_ -gt 2 }
```

Ein zweiter wichtiger Grundsatz lautet: *Filtern Sie so früh und so nah an der Quelle wie möglich.* Viele Cmdlets haben eigene Filterparameter, die direkt in der Datenquelle arbeiten und deutlich effizienter sind als `Where-Object` in der Pipeline.

```powershell
# Langsam: erst alles holen, dann filtern
Get-ChildItem -Path $HOME -Recurse -File | Where-Object { $_.Extension -eq '.log' }

# Besser: Filter direkt an das Cmdlet übergeben
Get-ChildItem -Path $HOME -Recurse -File -Filter '*.log'
```

Im ersten Fall erzeugt PowerShell für jede Datei ein Objekt und wirft die meisten wieder weg, im zweiten Fall liefert bereits das Dateisystem nur passende Dateien. Bei Active Directory, Exchange oder Graph ist der Unterschied noch gravierender, weil sonst Tausende Objekte über das Netzwerk übertragen werden, die niemand braucht.

> **Ergänzung aus der Nachrecherche**
>
> In der Microsoft-Dokumentation zu den Änderungen in PowerShell 7 ist vermerkt, dass `Where-Object` einen Parameter `-Not` erhielt: `Get-Service | Where-Object -Not DependentServices` liefert Dienste ohne abhängige Dienste, `Where-Object -Not Online` filtert Objekte, bei denen eine Eigenschaft nicht existiert oder leer beziehungsweise `False` ist. Im Test lieferte `$inventar | Where-Object -Not Online` (nach Setzen von `Online = $false` für `srv-db01`) genau diesen Eintrag.

> **Praxisbeispiel**
>
> Ein typischer Wartungsbericht fragt: Welche Zertifikate laufen in weniger als 30 Tagen ab? Welche Konten haben sich seit 90 Tagen nicht angemeldet? Welche virtuellen Maschinen haben mehr als 8 GB Speicher, aber weniger als 10 % Auslastung? In allen Fällen steht am Anfang ein `Where-Object` mit zwei oder drei Bedingungen, und die Qualität der Auswertung hängt davon ab, dass Typen (Datum, Zahl) richtig und Operatoren korrekt gewählt sind.

## 3.4 Sort-Object und Group-Object

Berichte brauchen Ordnung und Zusammenfassung. `Sort-Object` bringt Objekte in eine Reihenfolge, `Group-Object` fasst Objekte mit gleichem Wert einer Eigenschaft zusammen, und `Measure-Object` berechnet Kennzahlen wie Anzahl, Summe und Durchschnitt. Zusammen ermöglichen sie Auswertungen, die man sonst in einer Tabellenkalkulation macht: Top-Listen, Bestandsübersichten, Häufigkeitsverteilungen.

`Sort-Object -Property` sortiert aufsteigend nach einer oder mehreren Eigenschaften; `-Descending` kehrt die Reihenfolge um. Sollen verschiedene Eigenschaften unterschiedlich sortiert werden, übergibt man statt eines Namens eine Hashtable mit den Schlüsseln `Expression` und `Descending`.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$inventar | Sort-Object -Property RamGB -Descending
$inventar | Sort-Object -Property Standort, @{ Expression = 'RamGB'; Descending = $true }
$inventar | Sort-Object -Property RamGB -Descending | Select-Object -First 3 -Property Name, RamGB

$inventar | Group-Object -Property Standort | Select-Object -Property Name, Count
$inventar | Group-Object -Property Standort | ForEach-Object {
    [pscustomobject]@{
        Standort = $_.Name
        Anzahl   = $_.Count
        RamSumme = [int]($_.Group | Measure-Object -Property RamGB -Sum).Sum
    }
}
```

Die erste Zeile sortiert nach Speicher absteigend; die beiden 64-GB-Server stehen oben. Die zweite sortiert zuerst aufsteigend nach Standort (Graz, Linz, Wien) und innerhalb eines Standorts absteigend nach Speicher. Die dritte Zeile ist das klassische Muster einer *Top-N-Liste*: sortieren, dann mit `Select-Object -First` die ersten drei behalten (`srv-db01`, `srv-db02`, `srv-app01`). `Group-Object` bildet je Standort eine Gruppe. Jede Gruppe ist ein Objekt mit den Eigenschaften `Name` (der gemeinsame Wert), `Count` (Anzahl der Elemente) und `Group` (die Elemente selbst). Die erste Gruppierung zeigt Graz mit 2, Linz mit 1 und Wien mit 3 Servern. Die zweite berechnet je Gruppe die Summe des Speichers: Graz 48, Linz 64, Wien 88 GB. Das Ergebnis der Summe wird mit `[int]` in eine Ganzzahl umgewandelt, damit die Tabelle keine Nachkommastellen zeigt, und dass `Measure-Object` die Summe liefert, verdankt man dem Schalter `-Sum`.

> **Ergänzung aus der Nachrecherche**
>
> Laut der Microsoft-Dokumentation zu den Unterschieden zwischen Windows PowerShell und PowerShell 7 liefert `Group-Object` in PowerShell 7 die Gruppen *sortiert* zurück, weil das Cmdlet aus Leistungsgründen überarbeitet wurde. Man soll sich nicht auf die Reihenfolge verlassen, aber Skripte, die in Windows PowerShell den ersten aufgetretenen Wert als erste Gruppe erwarteten, können dadurch anders reagieren. Im Test erschienen die Standorte tatsächlich alphabetisch (Graz, Linz, Wien), nicht in der Reihenfolge des ersten Auftretens (Wien, Graz, Linz). Außerdem erlaubt `Group-Object -AsHashtable -AsString` das Erzeugen eines Nachschlagewerks, mit dem man später direkt über den Schlüssel zugreift.

Eine besonders nützliche Variante ist die Gruppierung als Hashtable. Dabei wird die Gruppenbezeichnung zum Schlüssel und die Menge der Gruppenmitglieder zum Wert. Damit lassen sich spätere Zugriffe ohne erneutes Filtern erledigen.

```powershell
$nachRolle = $inventar | Group-Object -Property Rolle -AsHashtable -AsString
$nachRolle['Web'].Name -join ', '
$nachRolle.GetType().Name
$inventar | Group-Object -Property Standort -NoElement
```

Die Hashtable `$nachRolle` liefert für den Schlüssel `Web` die zugehörigen Server (im Test `srv-web01, srv-web02`); der Typname lautet `Hashtable`. Mit `-NoElement` werden nur Namen und Anzahl ausgegeben, was bei großen Datenmengen Speicher spart.

Der wohl wichtigste Stolperstein beim Sortieren ist der *Typ* der Werte. Zeichenketten werden alphabetisch sortiert, auch wenn sie Zahlen enthalten: Die Liste `'10', '9', '2', '100'` wird zu `10, 100, 2, 9`. Erst nach der Umwandlung in Zahlen ergibt sich die erwartete Reihenfolge.

```powershell
# FALSCH: Zahlen als Text sortiert (alphabetisch)
'10', '9', '2', '100' | Sort-Object
# Ergebnis: 10, 100, 2, 9

# RICHTIG: Sortierschlüssel in eine Zahl umwandeln
'10', '9', '2', '100' | Sort-Object -Property { [int]$_ }
# Ergebnis: 2, 9, 10, 100
```

Dieses Problem begegnet Ihnen vor allem bei Daten aus CSV-Dateien, aus Webdiensten oder aus Text-Parsing, wo alles zunächst Text ist (Lerneinheit 3.6). Das Umwandeln in `[int]`, `[double]` oder `[datetime]` ist ein fester Schritt jeder Datenaufbereitung.

> **Merke**
>
> Sortieren: `Sort-Object -Property X [-Descending]`. Zusammenfassen: `Group-Object` (Gruppen mit `Name`, `Count`, `Group`) und `Measure-Object -Sum -Average -Maximum -Minimum`. Top-N: sortieren, dann `Select-Object -First N`. Sortierschlüssel müssen den richtigen Typ haben.

> **Praxisbeispiel**
>
> Ein Dateiserver-Bericht fasst zusammen, welche Dateitypen den meisten Speicher belegen: Alle Dateien einlesen, nach `Extension` gruppieren, je Gruppe die Summe der `Length`-Werte berechnen, absteigend sortieren und die ersten zehn ausgeben. Mit vier Befehlen in einer Pipeline ersetzt das eine Auswertung, die manuell Stunden dauern würde.

## 3.5 ForEach-Object

`ForEach-Object` führt für jedes Objekt der Pipeline einen Skriptblock aus. Damit lassen sich Objekte *transformieren* (ein neues Objekt oder einen Wert daraus berechnen), *Aktionen ausführen* (zum Beispiel pro Element einen Befehl aufrufen) und Zwischenwerte *sammeln*. Im Skriptblock steht `$_` (alternativ `$PSItem`) für das aktuelle Objekt. Was der Skriptblock ausgibt, wird Teil des Ergebnisses der Pipeline; gibt er nichts aus, entstehen auch keine Ergebnisse.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

1..5 | ForEach-Object { $_ * 2 }
$inventar | ForEach-Object { '{0} hat {1} GB RAM' -f $_.Name, $_.RamGB }
$inventar | ForEach-Object -Begin { $summe = 0 } -Process { $summe += $_.RamGB } -End { "Gesamt: $summe GB" }
($inventar | ForEach-Object Name) -join ', '
($inventar.ForEach({ $_.Name.ToUpper() })) -join ', '
1..6 | ForEach-Object { if ($_ % 2) { return }; $_ }
```

Die erste Zeile verdoppelt die Zahlen von 1 bis 5. Die zweite formatiert mit dem *Formatoperator* `-f` eine Zeichenkette pro Server: `{0}` und `{1}` sind Platzhalter für die folgenden Werte (zum Beispiel „srv-web01 hat 16 GB RAM“). Die dritte Zeile zeigt die drei Phasen aus Lerneinheit 1.6: `-Begin` läuft einmal vorab und setzt den Zähler zurück, `-Process` läuft pro Objekt und addiert, `-End` läuft einmal am Schluss und gibt die Summe aus (im Test „Gesamt: 200 GB“). Die vierte Zeile ist eine Kurzform für den Zugriff auf eine Eigenschaft. Die fünfte nutzt die Methodensyntax `.ForEach()`, die auf einer Sammlung im Speicher arbeitet. Die sechste Zeile zeigt, dass `return` im Skriptblock nicht die Pipeline verlässt, sondern nur die Verarbeitung des aktuellen Elements beendet, also wie `continue` in einer Schleife wirkt: Es erscheinen nur 2, 4 und 6.

Man muss zwischen der *Pipeline-Variante* `$sammlung | ForEach-Object { ... }` und der *Anweisung* `foreach ($x in $sammlung) { ... }` unterscheiden, die Modul 4 genauer behandelt. Die Anweisung lädt zuerst alle Elemente in den Speicher und läuft dann in einer schnellen Schleife. Die Pipeline verarbeitet Element für Element im Streaming-Betrieb, was bei sehr großen oder langsam eintreffenden Datenmengen Speicher spart und frühe Ergebnisse liefert, aber mehr Verwaltungsaufwand pro Element bedeutet. Im Test dauerte das Verarbeiten von 200 000 Zahlen mit `ForEach-Object` etwa 1,1 Sekunden, mit der `foreach`-Anweisung etwa 0,14 Sekunden. Solche Werte hängen stark von Rechner und Version ab; entscheidend ist der Faktor von rund acht, der sich bei kleinen Datenmengen nicht bemerkbar macht.

> **Ergänzung aus der Nachrecherche**
>
> **Parallele Verarbeitung.** Seit PowerShell 7.0 besitzt `ForEach-Object` den Parameter `-Parallel`, mit dem die Elemente gleichzeitig in mehreren Threads verarbeitet werden, begrenzt durch `-ThrottleLimit` (Standard 5). Variablen aus dem umgebenden Skript sind im Skriptblock über den Gültigkeitsbereich `$using:` erreichbar. Im Test lief `1..3 | ForEach-Object -Parallel { ... } -ThrottleLimit 3` auf drei verschiedenen Threads. Details, Risiken und Alternativen behandelt Modul 7.

> **Merke**
>
> `ForEach-Object` wandelt Pipeline-Elemente um; `$_` ist das aktuelle Element; `-Begin` und `-End` laufen einmal, `-Process` je Element. `return` beendet nur den aktuellen Durchlauf. Für Daten, die bereits im Speicher liegen, ist die `foreach`-Anweisung deutlich schneller.

Ein klassischer Fehler betrifft die Verschachtelung. Wer `ForEach-Object` in `ForEach-Object` schachtelt, verliert den äußeren `$_`, weil der innere Skriptblock seinen eigenen Wert hat. Abhilfe schafft eine Zwischenvariable.

```powershell
# FALSCH: Der innere Block überdeckt $_ des äußeren
$inventar | ForEach-Object {
    1..2 | ForEach-Object { "$($_.Name) Durchlauf $_" }   # $_.Name ist leer
}

# RICHTIG: äußeres Objekt in eigener Variable merken
$inventar | ForEach-Object {
    $server = $_
    1..2 | ForEach-Object { "$($server.Name) Durchlauf $_" }
}
```

Im falschen Beispiel bezeichnet `$_` im inneren Block die Zahl, nicht den Server; ihr `Name` ist leer, und die Ausgabe bleibt unvollständig, ohne dass ein Fehler gemeldet würde.

> **Praxisbeispiel**
>
> Ein typischer Einsatz ist das Aufbereiten von Eingabedaten für einen Massenlauf: Aus einer CSV-Liste mit Benutzernamen erzeugt `ForEach-Object` pro Zeile ein Objekt mit berechneten Feldern (Anzeigename, Benutzername aus Vor- und Nachnamen, Passwortrichtlinie), das anschließend an das Cmdlet zum Anlegen der Konten geht. Die Transformation ist von der Aktion sauber getrennt und lässt sich vorab prüfen.

## 3.6 Import und Export

Daten, die nur im Speicher liegen, verschwinden mit der Sitzung. Um Ergebnisse zu speichern, weiterzugeben oder später wieder einzulesen, wandelt PowerShell Objekte in Textformate um und umgekehrt. Vier Formate sind im Alltag wichtig, und jedes hat ein Paar von Cmdlets: *CSV* (Tabellen mit Trennzeichen), *JSON* (verschachtelte Daten, Standard bei Webschnittstellen), *XML* (strukturierte Dokumente und Konfigurationsdateien) und *CLIXML* (PowerShell-eigenes XML-Format, das Objekte mit ihren Typen speichert).

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$ordner = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'export-demo'
New-Item -Path $ordner -ItemType Directory -Force | Out-Null

# CSV
$csvPfad = Join-Path -Path $ordner -ChildPath 'inventar.csv'
$inventar | Export-Csv -Path $csvPfad -Delimiter ';' -Encoding utf8
$csv = Import-Csv -Path $csvPfad -Delimiter ';'
$csv[0].RamGB.GetType().Name
$csv | Sort-Object -Property { [int]$_.RamGB } | Select-Object -ExpandProperty RamGB

# JSON
$jsonPfad = Join-Path -Path $ordner -ChildPath 'inventar.json'
$inventar | ConvertTo-Json | Set-Content -Path $jsonPfad -Encoding utf8
(Get-Content -Path $jsonPfad -Raw | ConvertFrom-Json)[0].RamGB.GetType().Name

# CLIXML
$xmlPfad = Join-Path -Path $ordner -ChildPath 'inventar.clixml'
$inventar | Export-Clixml -Path $xmlPfad
$wieder = Import-Clixml -Path $xmlPfad
$wieder[0].RamGB.GetType().Name

Remove-Item -LiteralPath $ordner -Recurse
```

Der CSV-Teil schreibt das Inventar mit Semikolon als Trennzeichen (in deutschsprachigen Excel-Installationen der übliche Wert) und UTF-8-Kodierung. Beim Einlesen mit `Import-Csv` hat jede Spalte den Typ *Text*: Der Typ der Eigenschaft `RamGB` ist `String`. Die Sortierung ohne Umwandlung ergäbe die Reihenfolge 16, 16, 32, 64, 64, 8; mit dem Sortierschlüssel `[int]$_.RamGB` stimmt sie (8, 16, 16, 32, 64, 64). Der JSON-Teil wandelt die Objekte in einen JSON-Text um und liest sie zurück; JSON kennt Zahlen, deshalb ist `RamGB` danach wieder eine Zahl, allerdings vom Typ `Int64`. Im CLIXML-Teil bleibt der Typ `Int32` unverändert erhalten.

Jedes Format hat Vor- und Nachteile. CSV ist universell lesbar (Excel, Datenbanken), kennt aber nur flache Tabellen und nur Text. JSON bildet verschachtelte Strukturen ab, ist der Standard für Webschnittstellen und gut lesbar. XML eignet sich für dokumentartige Daten und Konfigurationen und lässt sich mit `[xml]` und `Select-Xml` durchsuchen. CLIXML bewahrt PowerShell-Objekte am genauesten, ist aber nur für PowerShell lesbar. Die beim Import entstehenden Objekte sind *deserialisiert*: Der Typname beginnt mit `Deserialized.`, und Methoden des Originals sind nicht vorhanden. Der richtige Einsatz richtet sich nach dem Zweck: Austausch mit anderen Systemen, Konfiguration oder Zwischenspeichern.

> **Ergänzung aus der Nachrecherche**
>
> **Standardverhalten in PowerShell 7 (laut Microsoft-Dokumentation und Test).** `Export-Csv` schreibt seit PowerShell 6 keine Typinformationszeile mehr (`-NoTypeInformation` ist Standard), `-IncludeTypeInformation` stellt das alte Verhalten wieder her. Im Test waren alle Felder in der CSV-Datei in Anführungszeichen gesetzt. `Import-Csv` kann außerdem Protokolldateien im W3C-Format lesen. `ConvertFrom-Json` hat den Schalter `-AsHashtable`, der Hashtables statt Objekten liefert. `ConvertTo-Json` wertet verschachtelte Objekte standardmäßig nur bis zur Tiefe 2 aus und gibt bei Überschreitung die Warnung „Resulting JSON is truncated as serialization has exceeded the set depth of 2.“ aus; im Test wurde die tiefste Ebene durch den Text `System.Collections.Hashtable` ersetzt, bis `-Depth 5` gesetzt wurde. Mit `Test-Json` prüfen Sie, ob ein Text gültiges JSON ist (im Test: gültig `True`, abgeschnittener Text `False`).

Das folgende Beispiel zeigt den Umgang mit der Tiefe von JSON und mit XML.

```powershell
$verschachtelt = @{ a = @{ b = @{ c = @{ d = 'e' } } } }
$verschachtelt | ConvertTo-Json -Compress
$verschachtelt | ConvertTo-Json -Depth 5 -Compress

[xml]$xml = '<inventar><server name="srv-web01" ram="16"/><server name="srv-db01" ram="64"/></inventar>'
$xml.inventar.server | ForEach-Object { "$($_.name): $($_.ram)" }
(Select-Xml -Xml $xml -XPath '//server[@ram>20]').Node.name

'{"a": 1, "b": [1,2]}' | Test-Json
```

Die erste Ausgabe ist wegen der Standardtiefe unvollständig: Auf der dritten Ebene steht statt der Daten der Text `System.Collections.Hashtable`. Die zweite zeigt die vollständige Struktur mit dem Wert `e`. Beim XML-Teil wandelt die Typangabe `[xml]` den Text in ein Dokumentobjekt um, durch das man mit Punkten navigiert: `$xml.inventar.server` liefert die beiden Server-Elemente, deren Attribute wie Eigenschaften lesbar sind. `Select-Xml` sucht mit einem XPath-Ausdruck; hier findet es den Server mit mehr als 20 GB (`srv-db01`).

> **Merke**
>
> CSV: flach, nur Text, universell. JSON: verschachtelt, Standard im Web, auf `-Depth` achten. XML: dokumentartig, mit `[xml]` und `Select-Xml`. CLIXML: erhält PowerShell-Typen, nur in PowerShell lesbar. Importierte Werte sind nach dem Import oft Text und müssen umgewandelt werden.

```powershell
# FALSCH: tief verschachtelte Daten mit Standardtiefe speichern
$konfig | ConvertTo-Json | Set-Content -Path konfig.json

# RICHTIG: Tiefe ausreichend wählen und das Ergebnis testen
$konfig | ConvertTo-Json -Depth 10 | Set-Content -Path konfig.json -Encoding utf8
Get-Content -Path konfig.json -Raw | Test-Json
```

Im falschen Fall gehen tiefere Ebenen verloren, und die Datei sieht trotzdem gültig aus; nur die Warnung weist auf das Problem hin, die in Skripten leicht übersehen wird. Ebenso heimtückisch ist die Annahme, dass ein Export und Import die Daten vollständig bewahren: Bei `Export-Clixml` mit Anmeldedaten ist der Schutz plattformabhängig (Modul 8), und Passwörter gehören nicht in Dateien.

> **Praxisbeispiel**
>
> Der gängige Weg zwischen Systemen: Ein Skript fragt die Cloud nach Benutzern (Objekte), reduziert sie auf relevante Spalten und exportiert eine CSV-Datei für die Personalabteilung. Ein zweites Skript liest eine JSON-Konfiguration mit Schwellwerten und prüft die Werte vor der Nutzung. Ein drittes speichert Zwischenstände mit `Export-Clixml`, um einen Lauf nach einem Abbruch fortzusetzen.

## 3.7 Formatierung vs Daten

Dieses Thema klingt harmlos und ist doch die Ursache vieler verwirrender Fehler. PowerShell trennt strikt zwischen *Daten* (Objekte mit Eigenschaften) und *Darstellung* (wie diese Objekte am Bildschirm aussehen). Solange Sie nichts anderes angeben, sorgt am Ende jeder Pipeline ein unsichtbares Cmdlet (`Out-Default`) dafür, dass Objekte lesbar angezeigt werden. Dabei entscheidet PowerShell anhand von Formatdefinitionen, welche Eigenschaften in welchen Spalten erscheinen. Wer die Darstellung selbst steuern möchte, verwendet die `Format-*`-Cmdlets: `Format-Table` (Tabelle), `Format-List` (Liste mit einer Zeile pro Eigenschaft), `Format-Wide` (mehrspaltig, nur ein Wert) und `Format-Custom` (eigene Ansichten).

Entscheidend ist, was diese Cmdlets zurückgeben: *keine Daten*, sondern Formatierungsobjekte, die nur eine Anweisung „zeige das so an“ enthalten. Das lässt sich nachweisen.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$ft = $inventar | Format-Table
$ft[0].GetType().FullName
($ft | Get-Member).TypeName | Select-Object -Unique
$ft.Count

$inventar | Format-Table -GroupBy Standort -Property Name, RamGB
$inventar | Select-Object -First 2 | Format-List -Property *
```

Die Variable `$ft` enthält nicht sechs Server, sondern zehn Objekte von Typen wie `FormatStartData`, `GroupStartData`, `FormatEntryData`, `GroupEndData` und `FormatEndData` aus dem Namensraum `Microsoft.PowerShell.Commands.Internal.Format`. Das sind die Bausteine der Darstellung: ein Kopf, je Zeile ein Eintrag und ein Abschluss. Daten, die man weiterverarbeiten könnte, stecken darin nicht mehr. Die beiden letzten Zeilen zeigen praktische Darstellungsoptionen: Eine Tabelle mit Zwischenüberschriften je Standort (`-GroupBy` gruppiert aufeinanderfolgende Einträge, sortieren Sie also vorher) und eine Liste aller Eigenschaften zweier Server.

Bei der Weiterverarbeitung ist das fatal. Wer `Format-Table` vor `Export-Csv` schaltet, erhält eine Datei mit Spalten wie `ClassId2e4f51ef21dd47e99d3c952918aff9cd`, `pageHeaderEntry` und `shapeInfo`: die internen Eigenschaften der Formatierungsobjekte. Im Test sah die erste Zeile genau so aus. Die Regel lautet daher: *Formatieren ist der letzte Schritt vor der Anzeige am Bildschirm.*

Wenn Sie die Darstellung als Text brauchen (für eine Protokolldatei oder eine E-Mail), wandeln Sie sie mit `Out-String` in eine Zeichenkette um. Der Parameter `-Width` bestimmt die Zeilenbreite (sonst gilt die Breite des Fensters, die in einer Pipeline unbekannt ist und Spalten abschneiden kann), `-Stream` liefert eine Zeichenkette pro Zeile.

```powershell
$text = $inventar | Format-Table -AutoSize | Out-String -Width 120
$text.GetType().Name
$text.Split("`n").Count
$zeilen = $inventar | Format-Table -AutoSize | Out-String -Stream
$zeilen.Count

$inventar | Select-Object -Property Name, RamGB | ConvertTo-Html -Fragment
```

Im Test war `$text` eine einzige `String`-Instanz mit 11 Zeilen; mit `-Stream` entstand eine Liste mit 10 Zeichenketten (ohne die abschließende Leerzeile). Für Berichte ist `ConvertTo-Html` oft besser als formatierter Text: Es erzeugt aus Objekten eine HTML-Tabelle, die sich per E-Mail verschicken lässt, und trennt Daten von Darstellung sauber.

Ein weiterer Effekt der Darstellungsregeln: Die *Spalten werden vom ersten Objekt bestimmt*. Haben spätere Objekte andere Eigenschaften, erscheinen diese nicht. Im Test erzeugte die Ausgabe von `[pscustomobject]@{A=1}` gefolgt von `[pscustomobject]@{B=2}` eine Tabelle mit der Spalte `A`; für das zweite Objekt blieb nur eine leere Zeile. Gemischte Objekttypen in einer Ausgabe sind deshalb zu vermeiden. Zusätzlich gilt: Objekte mit mehr als vier Eigenschaften zeigt PowerShell ohne eigene Formatdefinition als Liste, mit höchstens vier als Tabelle. Im Test wurde ein Objekt mit fünf Eigenschaften als Liste, eines mit vier als Tabelle angezeigt.

> **Merke**
>
> Daten und Darstellung trennen: Objekte erzeugen, filtern, sortieren, exportieren; `Format-*` nur ganz am Ende für die Anzeige. `Format-*` liefert keine Daten, sondern Formatierungsobjekte. Für Text als Ausgabe `Out-String`, für Berichte `ConvertTo-Html` oder `Export-Csv` mit echten Objekten.

```powershell
# FALSCH: formatieren, dann weiterverarbeiten
Get-Process | Format-Table Name, Id | Export-Csv -Path prozesse.csv

# RICHTIG: Objekte reduzieren, dann exportieren
Get-Process | Select-Object -Property Name, Id | Export-Csv -Path prozesse.csv
```

Dieselbe Falle gilt für `Write-Host`: Der Befehl schreibt Text direkt in die Konsole, *nicht* in die Pipeline, und liefert deshalb keine Daten für nachfolgende Befehle (Lerneinheit 1.3). Für die Anzeige von Zwischenständen eignet sich `Write-Verbose` oder `Write-Information`, für Ergebnisse die normale Ausgabe.

> **Praxisbeispiel**
>
> In Pipelines und geplanten Tasks gibt es keine Konsole mit definierter Breite. Ein Skript, das lokal eine schön formatierte Tabelle zeigt, kann im Task abgeschnittene Spalten oder zerrissene Zeilen erzeugen. Die robuste Lösung ist, Objekte zu erzeugen und das Format erst beim Schreiben der Zieldatei festzulegen (`Export-Csv`, `ConvertTo-Json`, `ConvertTo-Html`).

## 3.8 Reguläre Ausdrücke

Nicht jede Information liegt als Objekt vor. Protokolldateien, Konfigurationstexte, E-Mails und Ausgaben externer Programme bestehen aus Text, aus dem man Datum, Servername, Fehlercode oder Pfad herausholen muss. Dafür gibt es *reguläre Ausdrücke* (regular expressions, kurz Regex): eine kompakte Schreibweise für Textmuster, die in praktisch allen Programmiersprachen existiert. Ein Muster beschreibt, wie ein gesuchter Text aussieht, nicht, was er enthält: „vier Ziffern, ein Bindestrich, zwei Ziffern“ passt auf jedes Datum.

Die wichtigsten Bausteine: Ein normales Zeichen steht für sich selbst. Der Punkt steht für ein beliebiges Zeichen, `\d` für eine Ziffer, `\w` für ein Buchstaben-, Ziffern- oder Unterstrichzeichen, `\s` für Leerraum, `\S` für alles andere als Leerraum. Mengenangaben legen die Wiederholung fest: `*` (null oder mehr), `+` (eins oder mehr), `?` (null oder eins), `{4}` (genau vier), `{2,4}` (zwei bis vier). Das Zeichen `^` markiert den Anfang, `$` das Ende des Textes. Runde Klammern bilden eine *Gruppe*, deren Treffer sich einzeln abrufen lässt; mit `(?<name>...)` bekommt die Gruppe einen Namen. Eckige Klammern definieren eine Zeichenklasse (`[A-Z]` ein Großbuchstabe), der senkrechte Strich steht für „oder“. Zeichen mit besonderer Bedeutung müssen mit einem Rückwärtsschrägstrich maskiert werden, wenn sie wörtlich gemeint sind.

PowerShell bietet mehrere Wege, Muster anzuwenden. Der Operator `-match` prüft, ob ein Text zum Muster passt, und füllt bei einem Treffer die automatische Variable `$Matches` mit dem Gesamttreffer (`$Matches[0]`) und den Gruppen (nummeriert oder, wenn benannt, über den Namen). `-replace` ersetzt Treffer, `-split` zerlegt Text an Treffern. `Select-String` durchsucht Dateien oder Texte zeilenweise, ähnlich wie `grep`. Für alle Treffer eines Textes dient die .NET-Methode `[regex]::Matches()`.

```powershell
$zeilen = @(
    '2026-10-24 08:15:01 ERROR srv-db01 Verbindung abgelehnt (Code 1042)'
    '2026-10-24 08:15:07 INFO srv-web01 Dienst gestartet'
    '2026-10-24 08:16:30 WARN srv-app01 Antwortzeit 2300 ms'
    '2026-10-24 08:17:12 ERROR srv-web02 Zeitüberschreitung (Code 408)'
)
$muster = '^(?<Datum>\d{4}-\d{2}-\d{2}) (?<Zeit>\d{2}:\d{2}:\d{2}) (?<Stufe>[A-Z]+) (?<Host>\S+) (?<Text>.*)$'

$ereignisse = foreach ($zeile in $zeilen) {
    if ($zeile -match $muster) {
        [pscustomobject]@{
            Zeit  = [datetime]"$($Matches.Datum) $($Matches.Zeit)"
            Stufe = $Matches.Stufe
            Host  = $Matches.Host
            Text  = $Matches.Text
        }
    }
}
$ereignisse | Format-Table -AutoSize
($ereignisse | Where-Object Stufe -eq 'ERROR' | Measure-Object).Count

[regex]::Matches('Code 1042 und Code 408', 'Code (\d+)') | ForEach-Object { $_.Groups[1].Value }
```

Das Beispiel verwandelt Textzeilen in Objekte. Das Muster beschreibt eine Zeile: Datum aus vier, zwei und zwei Ziffern, Uhrzeit, Stufe aus Großbuchstaben, ein Rechnername ohne Leerraum und der restliche Text. Jede Gruppe hat einen Namen (`Datum`, `Zeit`, `Stufe`, `Host`, `Text`). Passt eine Zeile, stehen die Gruppen in `$Matches`; aus Datum und Zeit entsteht per Typumwandlung ein echtes Datumsobjekt, und es wird ein Objekt mit vier Eigenschaften erzeugt. Alle vier Zeilen passen; die Auswertung zählt zwei Fehlermeldungen und der Typ von `Zeit` ist `DateTime`. Ab hier lässt sich normal weiterarbeiten: filtern, sortieren, gruppieren, exportieren. Die letzte Zeile findet mit `[regex]::Matches` alle Treffer in einem Text und liest die erste Gruppe jedes Treffers aus (`1042` und `408`); `-match` findet dagegen nur den *ersten* Treffer.

```powershell
'srv-web01' -replace '^srv-(\w+?)(\d+)$', '$2-$1'
'a1b22c333' -split '\d+'
'abc', 'a1', 'b2' -match '\d'
'<b>fett</b> und <i>kursiv</i>' -replace '<.+?>', ''
Select-String -Path $logdatei -Pattern 'ERROR' | ForEach-Object LineNumber
```

Die erste Zeile vertauscht mit `$1` und `$2` die Gruppen im Ersatztext: „srv-web01“ wird zu `01-web`. Die zweite zerlegt an Ziffernfolgen und liefert die drei Buchstaben a, b und c. Die dritte zeigt, dass `-match` auf einer *Liste* die passenden Elemente zurückgibt (`a1` und `b2`) und `$Matches` nicht füllt. Die vierte entfernt HTML-Marken; das Fragezeichen nach dem Plus macht die Suche *genügsam* (lazy): ohne es würde `<.+>` gierig den Text vom ersten bis zum letzten Zeichen `>` entfernen und im Test eine leere Zeichenkette liefern, mit dem Fragezeichen bleibt „fett und kursiv“. Die letzte Zeile gibt die Nummern der Zeilen einer Datei aus, die „ERROR“ enthalten (im Test: 1 und 4); sie setzt eine Variable `$logdatei` voraus.

> **Ergänzung aus der Nachrecherche**
>
> Laut Microsoft-Dokumentation zu Hashtables und `$Matches` lassen sich benannte Untergruppen direkt über `$Matches.Name` lesen, zum Beispiel `$Matches.SSN`. Das ist deutlich lesbarer als nummerierte Gruppen und überlebt Änderungen am Muster. Ein häufiger Irrtum betrifft doppelte Anführungszeichen: In einem Ersatztext in Anführungszeichen wertet PowerShell `$1` als *Variable* aus, bevor der Ersatz stattfindet. Im Test lieferte `"$2-$1"` deshalb nur den Bindestrich; mit einfachen Anführungszeichen funktionierte die Ersetzung.

> **Merke**
>
> `-match` liefert wahr/falsch und füllt `$Matches` (nur erster Treffer; auf Listen filtert er). `-replace` und `-split` arbeiten mit Mustern, `[regex]::Matches` und `Select-String -AllMatches` finden alle Treffer. Muster und Ersatztexte gehören in einfache Anführungszeichen. Sonderzeichen vorher mit `[regex]::Escape()` maskieren.

Typische Fehler betreffen Maskierung und Anführungszeichen. Der Rückwärtsschrägstrich ist in einem Windows-Pfad das Trennzeichen und im Muster das Maskierungszeichen. Wer einen Pfad unbearbeitet als Muster verwendet, erhält Fehler wie „Unrecognized escape sequence \T“.

```powershell
# FALSCH: Pfad direkt als Muster
'C:\Temp\datei.txt' -match 'C:\Temp'        # Fehler: ungültiges Muster

# RICHTIG: wörtlichen Text maskieren
$pfad = 'C:\Temp'
'C:\Temp\datei.txt' -match [regex]::Escape($pfad)
'C:\Temp\datei.txt' -replace '\\', '/'
```

Bei der Maskierung erzeugt `[regex]::Escape()` ein Muster, das jedes Zeichen wörtlich nimmt. Besonders wichtig ist das, wenn Benutzereingaben in ein Muster einfließen: Ungeprüft können sie Sonderzeichen enthalten, die das Muster verändern; das ist ein Sicherheits- und Zuverlässigkeitsthema (Modul 8). Auch der Punkt wird häufig missverstanden: `'a.b'` als Muster passt auch auf „axb“ (im Test: `True`), während die maskierte Fassung nur den wörtlichen Punkt findet.

> **Praxisbeispiel**
>
> Aus einem Exchange- oder Webserver-Protokoll sollen alle Fehlerzeilen eines Tages mit Client-Adresse und Statuscode ausgewertet werden. Ein Muster mit benannten Gruppen verwandelt jede Zeile in ein Objekt, danach genügen `Group-Object` und `Sort-Object`, um die zehn häufigsten Fehler zu erhalten. Dieselbe Technik hilft bei der Auswertung der Ausgaben externer Befehlszeilenprogramme, die kein Objekt liefern.

## 3.9 Encoding und Kultur

Zwei unsichtbare Einstellungen entscheiden darüber, ob Ihr Skript auf jedem Rechner dasselbe Ergebnis liefert: die *Zeichenkodierung* (wie Zeichen in Bytes übersetzt werden) und die *Kultur* (wie Zahlen, Datumsangaben und Sortierungen dargestellt und gelesen werden). Beide sind klassische Quellen für Fehler, die nur auf manchen Rechnern auftreten und deshalb schwer zu finden sind.

### Zeichenkodierung.

Eine Datei besteht aus Bytes; die Kodierung legt fest, welche Bytes welche Zeichen darstellen. *ASCII* kennt nur 128 Zeichen ohne Umlaute. *Windows-1252* (oft als „ANSI“ bezeichnet) und *ISO-8859-1* (Latin1) belegen ein Byte pro Zeichen und kennen westeuropäische Umlaute. *UTF-8* ist die heute übliche Kodierung für alle Zeichen der Welt: ASCII-Zeichen belegen ein Byte, Umlaute zwei, andere Zeichen bis zu vier. *UTF-16* (in PowerShell `unicode`) belegt zwei oder vier Byte. Ein *Byte-Order-Mark* (BOM) ist eine Bytefolge am Dateianfang, die die Kodierung anzeigt; bei UTF-8 sind es die drei Bytes `EF BB BF`.

Im Test wurde der Text „Grüße“ mit verschiedenen Werten für `-Encoding` gespeichert und die Bytes angezeigt. Die Kodierung `utf8NoBOM` lieferte `47 72 C3 BC C3 9F 65`: Umlaut und ß belegen je zwei Byte. `utf8BOM` stellte `EF BB BF` voran. `unicode` (UTF-16) lieferte `FF FE 47 00 72 00 FC 00 DF 00 65 00`, also zwei Bytes pro Zeichen mit BOM. `ascii` lieferte `47 72 3F 3F 65`: Aus ü und ß wurde je ein Fragezeichen (Byte `3F`), die Information ist unwiederbringlich verloren. `ansi` und `latin1` lieferten `47 72 FC DF 65`, jeweils ein Byte pro Sonderzeichen.

> **Ergänzung aus der Nachrecherche**
>
> **Standardwerte in PowerShell 7 (laut Microsoft-Dokumentation).** Windows PowerShell hat je nach Cmdlet unterschiedliche Standardkodierungen (zum Beispiel ANSI oder UTF-16 mit BOM bei `Out-File` und Umleitung), PowerShell 7 schreibt dagegen konsequent *UTF-8 ohne BOM*. Die Variable `$OutputEncoding` (Kodierung für die Übergabe an externe Programme) steht in PowerShell 7 auf UTF-8 ohne BOM statt ASCII. Beim Parameter `-Encoding` wurde der Wert `Byte` entfernt; stattdessen gibt es `-AsByteStream`. Neu sind die Werte `utf8BOM` und `utf8NoBOM`; `-Encoding utf8` bedeutet in PowerShell 7 ohne BOM, in Windows PowerShell 5.1 mit BOM. Wer Dateien zwischen beiden Editionen austauscht, gibt die Kodierung deshalb besser ausdrücklich an.

Das folgende Beispiel zeigt, wie Sie die Kodierung einer Datei prüfen und was bei falscher Lesart geschieht.

```powershell
$f = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'enc.txt'

Set-Content -Path $f -Value 'Grüße' -Encoding utf8BOM
(Get-Content -Path $f -AsByteStream -TotalCount 3 | ForEach-Object { $_.ToString('X2') }) -join ' '

Set-Content -Path $f -Value 'Grüße' -Encoding utf8NoBOM
Get-Content -Path $f -Encoding utf8
Get-Content -Path $f -Encoding latin1

Remove-Item -Path $f
```

Zuerst wird die Datei mit BOM geschrieben und die ersten drei Bytes werden als Hexadezimalzahlen ausgegeben (`EF BB BF`); so erkennt man eine UTF-8-Datei mit BOM. Dann wird sie ohne BOM geschrieben und zweimal gelesen: Mit der richtigen Kodierung erscheint „Grüße“, mit der falschen (`latin1`) die typische *Zeichenverwirrung* (Mojibake): „Grüße“ wird als „GrÃ¼Ãe“ gelesen. Taucht so etwas in einem Bericht auf, ist praktisch immer eine Datei in einer anderen Kodierung gelesen worden, als sie geschrieben wurde. Der Parameter `-AsByteStream` ist in PowerShell 7 verfügbar; in Windows PowerShell heißt die Entsprechung `-Encoding Byte`.

### Kultur.

Die Kultur bestimmt, wie Zahlen und Datumsangaben geschrieben werden. In Österreich ist das Dezimalzeichen das Komma (`1234,5`), in den USA der Punkt (`1234.5`); das Datum `03/04/2026` bedeutet in den USA den 4. März, in Europa den 3. April. Eine Sitzung hat zwei Kulturen: `Get-Culture` (Formate für Zahlen und Daten) und `Get-UICulture` (Sprache von Oberfläche und Meldungen); die Variable `$PSCulture` liefert den Namen der ersten.

PowerShell geht mit Kulturen auf eine besondere, oft überraschende Art um. Alles, was Sie im *Quelltext* schreiben (Zahlenliterale, Typumwandlungen wie `[double]'1.5'`), interpretiert PowerShell *immer mit der invarianten Kultur*, also mit Punkt als Dezimalzeichen und Datum im US-Format. Die Umwandlung einer Zahl in Text mit `"$zahl"` verwendet ebenfalls die invariante Kultur. Methoden von .NET und der Formatoperator `-f` verwenden dagegen die *aktuelle Kultur*. Das Ziel dieser Regel ist, dass Skripte nicht je nach Spracheinstellung anders laufen; die Folge ist, dass man beide Fälle kennen muss.

```powershell
[cultureinfo]::CurrentCulture = [cultureinfo]'de-AT'

$zahl = 1234.5
"$zahl"
$zahl.ToString()
'{0:N2}' -f $zahl
(Get-Date '2026-10-25').ToString('D')

[double]'1,5'
[double]::Parse('1234,5', [cultureinfo]'de-AT')
[datetime]'03/04/2026' | ForEach-Object { $_.ToString('yyyy-MM-dd') }
[datetime]::ParseExact('04.03.2026', 'dd.MM.yyyy', [cultureinfo]::InvariantCulture).ToString('yyyy-MM-dd')

[cultureinfo]::CurrentCulture = [cultureinfo]::InvariantCulture
```

Die erste Zeile setzt die Kultur der Sitzung auf Deutsch (Österreich) und die letzte stellt sie zurück. Im Test ergab `"$zahl"` `1234.5` (invariant), `$zahl.ToString()` dagegen `1234,5`, und `'{0:N2}' -f $zahl` `1 234,50`, mit einem geschützten Leerzeichen als Tausendertrenner, wie es in Österreich üblich ist. Das lange Datumsformat lieferte „Sonntag, 25. Oktober 2026“. Besonders gefährlich ist `[double]'1,5'`: Die invariante Kultur kennt das Komma als Tausendertrenner, und das Ergebnis ist die Zahl *fünfzehn*, nicht 1,5; es gibt keinen Fehler. Wer deutsche Eingaben lesen will, verwendet `[double]::Parse()` mit der passenden Kultur und erhält 1234,5. Das Datum `'03/04/2026'` wird als 4. März 2026 gelesen (US-Format); für europäische Datumsangaben gibt man das Format an: `ParseExact` mit `'dd.MM.yyyy'` liefert für `04.03.2026` den 4. März. Beachten Sie, dass die österreichische Kultur ein geschütztes Leerzeichen als Tausendertrenner verwendet: Der Text `'1.234,5'` mit Punkt wird mit `de-AT` nicht gelesen (im Test: Formatfehler), mit `de-DE` dagegen schon.

> **Merke**
>
> Quelltext und Umwandlungen sind invariant (Punkt, US-Datum), `.ToString()` und `-f` folgen der aktuellen Kultur. Dateien und Austauschformate: Kodierung ausdrücklich angeben (UTF-8) und Datum im ISO-Format (`yyyy-MM-dd`) schreiben. Eingaben aus anderen Kulturen mit `Parse` oder `ParseExact` und expliziter Kultur lesen.

```powershell
# FALSCH: deutsche Kommazahl und Landesdatum per Typumwandlung lesen
[double]'1234,5'            # ergibt 12345, kein Fehler
[datetime]'04.03.2026'      # liest den 3. April 2026 statt des 4. März

# RICHTIG: Kultur und Format ausdrücklich angeben
[double]::Parse('1234,5', [cultureinfo]'de-AT')
[datetime]::ParseExact('04.03.2026', 'dd.MM.yyyy', [cultureinfo]::InvariantCulture)
```

Im falschen Fall erhält man ohne jede Warnung falsche Werte, die erst in der Auswertung auffallen. Wer Berichte für Excel schreibt, nutzt zusätzlich `Export-Csv -UseCulture`: Der Parameter verwendet das Listentrennzeichen der aktuellen Kultur (in Österreich das Semikolon). Dieser Wert hängt aber vom Rechner ab; für wiederholbare Automation ist ein ausdrücklich angegebenes Trennzeichen wie `-Delimiter ';'` verlässlicher (im Test hatte die Standardkultur der Testumgebung das Komma).

> **Praxisbeispiel**
>
> Ein Skript läuft auf dem englischsprachigen Build-Server und auf dem deutschsprachigen Administratorrechner und soll überall dieselbe CSV-Datei liefern. Es gibt UTF-8, das Trennzeichen Semikolon und Datumswerte im ISO-Format ausdrücklich an und liest Fremddaten mit expliziter Kultur. Ohne diese Disziplin erhält man Umlaute als Fragezeichen, Zahlen mit falschem Dezimalzeichen und vertauschte Monate und Tage.

## 3.10 Übungsaufgaben zu Modul 03

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Verwenden Sie für die Aufgaben 5 und 6 die Beispieldaten aus diesem Modul. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Erklären Sie den Unterschied zwischen `Select-Object -Property Name` und `Select-Object -ExpandProperty Name`. Wann brauchen Sie welche Variante?

**Aufgabe 2.** Was geschieht bei `1..5 | Where-Object { $_ > 3 }`? Wie lautet die korrekte Schreibweise, und welchen Nebeneffekt hat die falsche?

**Aufgabe 3.** Warum darf `Format-Table` nicht vor `Export-Csv` stehen? Woran erkennen Sie den Fehler in der erzeugten Datei?

**Aufgabe 4.** Warum liefert `[double]'1,5'` die Zahl 15, und wie lesen Sie die Eingabe „1,5“ korrekt als 1,5?

**Aufgabe 5.** Filtern Sie aus den Beispieldaten alle Server mit mindestens 16 GB Speicher am Standort Wien oder Graz. Sortieren Sie nach Name und geben Sie nur `Name` und eine berechnete Eigenschaft `RamMB` aus.

**Aufgabe 6.** Fassen Sie die Beispieldaten nach `Rolle` zusammen: Anzahl der Server und Gesamtspeicher je Rolle. Sortieren Sie absteigend nach Gesamtspeicher und exportieren Sie das Ergebnis als CSV (Semikolon, UTF-8) in das temporäre Verzeichnis. Lesen Sie die Datei wieder ein und prüfen Sie, dass die Spalte `Gesamt` nach Umwandlung in eine Zahl korrekt sortiert werden kann.

**Aufgabe 7.** Die Zeilen `2026-10-25 09:00:01 WARN srv-a Speicher 91 %`, `2026-10-25 09:00:09 ERROR srv-b Dienst gestoppt`, `2026-10-25 09:01:15 WARN srv-b Speicher 87 %` und `2026-10-25 09:02:40 INFO srv-a Neustart` sollen mit einem regulären Ausdruck mit benannten Gruppen in Objekte (`Zeit`, `Stufe`, `Host`, `Text`) umgewandelt werden. Zählen Sie die Meldungen je Stufe und schreiben Sie die Objekte als JSON (ausreichende Tiefe) in eine Datei, die Sie mit `Test-Json` prüfen.

**Aufgabe 8.** Lesen Sie die folgenden Verkaufsdaten in deutscher Schreibweise ein (Semikolon, Dezimalkomma, Datum `dd.MM.yyyy`): `Datum;Betrag`, `03.10.2026;1.250,50`, `17.10.2026;980,00` und `04.11.2026;2.100,25`. Wandeln Sie Datum und Betrag in echte Typen um (Kultur `de-DE` für den Betrag) und berechnen Sie die Summe je Monat. Geben Sie das Ergebnis mit Monat im Format `yyyy-MM` aus.

## 3.11 Lösungen zu Modul 03

### Aufgabe 1.

`-Property Name` liefert *Objekte*, die nur die Eigenschaft `Name` besitzen; `-ExpandProperty Name` liefert den *Wert* der Eigenschaft, also im Beispiel reinen Text. Für weitere Verarbeitung mit Cmdlets, die Werte erwarten (Pfade, Namen, Zahlen), oder zum Einsetzen in Zeichenketten braucht man `-ExpandProperty`. Für Berichte und Tabellen mit mehreren Spalten braucht man `-Property`.

### Aufgabe 2.

`>` ist keine Vergleichsoperation, sondern leitet die Ausgabe in eine Datei um; hier würde der Skriptblock `$_` in eine Datei namens `3` schreiben und liefert keine Ausgabe. Als Nebeneffekt entsteht eine Datei `3` im aktuellen Ordner. Korrekt ist `1..5 | Where-Object { $_ -gt 3 }` mit dem Ergebnis 4 und 5.

### Aufgabe 3.

`Format-Table` erzeugt Formatierungsobjekte (Typen aus `Microsoft.PowerShell.Commands.Internal.Format`), keine Daten. `Export-Csv` schreibt dann deren interne Eigenschaften. Erkennbar ist der Fehler an Spaltennamen wie `ClassId...`, `pageHeaderEntry` oder `shapeInfo`. Richtig ist, Spalten mit `Select-Object` zu wählen und Formatierung nur zur Anzeige zu nutzen.

### Aufgabe 4.

Typumwandlungen im Quelltext verwenden die invariante Kultur, in der der Punkt das Dezimalzeichen und das Komma ein Tausendertrenner ist. „1,5“ wird deshalb als „15“ gelesen. Richtig: `[double]::Parse('1,5', [cultureinfo]'de-AT')` oder `'de-DE'` liefert 1,5.

### Aufgabe 5.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$inventar |
    Where-Object { $_.RamGB -ge 16 -and $_.Standort -in 'Wien', 'Graz' } |
    Sort-Object -Property Name |
    Select-Object -Property Name, @{ Name = 'RamMB'; Expression = { $_.RamGB * 1024 } }
```

Die Bedingung verknüpft den Größenvergleich mit `-in`. Erwartetes Ergebnis: `srv-app01` (32768), `srv-db01` (65536), `srv-web01` (16384) und `srv-web02` (16384), alphabetisch sortiert. Sortiert wird *vor* `Select-Object`, solange die Eigenschaft `Name` noch vorhanden ist (sie bliebe auch danach vorhanden, aber die Reihenfolge „filtern, sortieren, formen“ ist die robuste Faustregel).

### Aufgabe 6.

```powershell
$inventar = 'srv-web01,Web,16,Wien', 'srv-web02,Web,16,Graz', 'srv-db01,Datenbank,64,Wien',
    'srv-db02,Datenbank,64,Linz', 'srv-app01,Anwendung,32,Graz', 'srv-file01,Dateien,8,Wien' |
    ForEach-Object { $t = $_ -split ','; [pscustomobject]@{ Name = $t[0]; Rolle = $t[1]; RamGB = [int]$t[2]; Standort = $t[3] } }

$bericht = $inventar | Group-Object -Property Rolle | ForEach-Object {
    [pscustomobject]@{
        Rolle  = $_.Name
        Anzahl = $_.Count
        Gesamt = [int]($_.Group | Measure-Object -Property RamGB -Sum).Sum
    }
} | Sort-Object -Property Gesamt -Descending

$pfad = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'rollen.csv'
$bericht | Export-Csv -Path $pfad -Delimiter ';' -Encoding utf8
$gelesen = Import-Csv -Path $pfad -Delimiter ';'
$gelesen | Sort-Object -Property { [int]$_.Gesamt } -Descending | Format-Table -AutoSize
$gelesen[0].Gesamt.GetType().Name
Remove-Item -Path $pfad
```

Erwartet wird „Datenbank“ mit 128 GB (2 Server) an erster Stelle, danach „Web“ (2 Server) und „Anwendung“ (1 Server) mit je 32 GB, deren Reihenfolge bei Gleichstand nicht festgelegt ist, und zuletzt „Dateien“ mit 8 GB. Nach dem Einlesen ist `Gesamt` Text (`String`); erst der Sortierschlüssel `[int]$_.Gesamt` sortiert numerisch korrekt. Ohne Umwandlung würde bei absteigender Sortierung „8“ an erster Stelle und „128“ an letzter stehen, weil Text alphabetisch verglichen wird.

### Aufgabe 7.

```powershell
$zeilen = @(
    '2026-10-25 09:00:01 WARN srv-a Speicher 91 %'
    '2026-10-25 09:00:09 ERROR srv-b Dienst gestoppt'
    '2026-10-25 09:01:15 WARN srv-b Speicher 87 %'
    '2026-10-25 09:02:40 INFO srv-a Neustart'
)
$muster = '^(?<Datum>\d{4}-\d{2}-\d{2}) (?<Zeit>\d{2}:\d{2}:\d{2}) (?<Stufe>[A-Z]+) (?<Host>\S+) (?<Text>.*)$'

$objekte = foreach ($zeile in $zeilen) {
    if ($zeile -match $muster) {
        [pscustomobject]@{
            Zeit  = [datetime]"$($Matches.Datum) $($Matches.Zeit)"
            Stufe = $Matches.Stufe
            Host  = $Matches.Host
            Text  = $Matches.Text
        }
    }
}

$objekte | Group-Object -Property Stufe | Select-Object -Property Name, Count

$json = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'ereignisse.json'
$objekte | ConvertTo-Json -Depth 3 | Set-Content -Path $json -Encoding utf8
Get-Content -Path $json -Raw | Test-Json
Remove-Item -Path $json
```

Die Gruppen sind benannt, deshalb ist `$Matches.Stufe` lesbar. Ergebnis der Zählung: ERROR 1, INFO 1, WARN 2 (alphabetisch sortiert). `Test-Json` liefert `True`. Datumswerte werden in JSON als Text im ISO-Format abgelegt.

### Aufgabe 8.

```powershell
$daten = @'
Datum;Betrag
03.10.2026;1.250,50
17.10.2026;980,00
04.11.2026;2.100,25
'@
$de = [cultureinfo]'de-DE'
$invariant = [cultureinfo]::InvariantCulture

$verkaeufe = $daten | ConvertFrom-Csv -Delimiter ';' | ForEach-Object {
    [pscustomobject]@{
        Datum  = [datetime]::ParseExact($_.Datum, 'dd.MM.yyyy', $invariant)
        Betrag = [decimal]::Parse($_.Betrag, $de)
    }
}

$verkaeufe |
    Group-Object -Property { $_.Datum.ToString('yyyy-MM') } |
    ForEach-Object {
        [pscustomobject]@{
            Monat = $_.Name
            Summe = ($_.Group | Measure-Object -Property Betrag -Sum).Sum
        }
    }
```

Das Datum wird mit `ParseExact` und dem Format `dd.MM.yyyy` gelesen, der Betrag mit der Kultur `de-DE`, die den Punkt als Tausendertrenner und das Komma als Dezimalzeichen kennt (die Kultur `de-AT` würde hier ein Leerzeichen als Tausendertrenner erwarten). `decimal` eignet sich für Geldbeträge besser als `double`, weil es Dezimalbrüche exakt speichert. Ergebnis: 2026-10 mit 2230,50 und 2026-11 mit 2100,25. Die Ausgabe der Zahlen folgt dabei der aktuellen Kultur der Sitzung.
