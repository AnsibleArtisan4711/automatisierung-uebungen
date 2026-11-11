# Modul 04 - Skripting, Variablen und Kontrollstrukturen

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (4.1 bis 4.9; 4.10 Übungsaufgaben, 4.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Übersichtsblatt (10 h) und Detailblatt (9 h) weichen um eine Stunde voneinander ab. Dieses Skript folgt dem Detailblatt, weil jede Lerneinheit dort einzeln aufgeführt ist.

### Lernziele.

Nach diesem Modul können Sie strukturierte Skripte mit Parametern, Logik und Funktionen erstellen. Sie bauen Skripte mit klarem Kopf und Rumpf, verwenden Variablen mit Typen und Gültigkeitsbereichen, modellieren Daten mit Arrays und Hashtables, steuern den Ablauf mit Verzweigungen und Schleifen, schreiben Advanced Functions mit validierten Parametern, übergeben Parameter lesbar per Splatting und trennen Konfiguration vom Code. Sicherheitsaspekte und Betriebspraxis begleiten die Beispiele, und das Ziel bleibt wiederholbare Automation für Windows, Cloud und CI/CD.

| **Nr.** | **Lerneinheit**                | **Dauer** | **Prio** |
|:--------|:-------------------------------|:----------|:---------|
| 4.1     | Skriptstruktur und Shebang     | 1 h       | 1        |
| 4.2     | Variablen und Typisierung      | 1 h       | 1        |
| 4.3     | Arrays und Hashtables          | 1 h       | 1        |
| 4.4     | If, Switch und Vergleichslogik | 1 h       | 1        |
| 4.5     | Schleifen                      | 1 h       | 1        |
| 4.6     | Funktionen                     | 1 h       | 1        |
| 4.7     | Parameter und Validation       | 1 h       | 2        |
| 4.8     | Splatting                      | 1 h       | 2        |
| 4.9     | Konfigurationsdateien          | 1 h       | 2        |
|         | **Summe**                      | **9 h**   |          |

## 4.1 Skriptstruktur und Shebang

Ein Skript ist eine Textdatei mit der Endung `.ps1`, die PowerShell-Befehle enthält und sie der Reihe nach ausführt. Das klingt einfach, doch der Unterschied zwischen einer Befehlsliste, die auf dem eigenen Rechner zufällig funktioniert, und einem Skript, das zuverlässig in einer Pipeline, einem geplanten Task oder auf dem Rechner eines Kollegen läuft, liegt fast ausschließlich in der *Struktur*. Ein gut aufgebautes Skript sagt am Anfang, was es braucht und was es tut, nimmt seine Eingaben über Parameter entgegen, bricht bei Fehlern kontrolliert ab und meldet sein Ergebnis über einen Exit Code. Diese Gliederung erwarten Kollegen und Prüfwerkzeuge gleichermaßen.

Ein bewährter Aufbau besteht aus fünf Teilen. Zuerst stehen die *Voraussetzungen*: Mit `#Requires`-Zeilen (Lerneinheit 2.8) legt das Skript fest, welche PowerShell-Version und welche Module es benötigt. Darauf folgt die *kommentarbasierte Hilfe* (Lerneinheit 2.9) mit Zweck, Parametern und Beispielen. Danach kommt der *Parameterblock*, der `param()` mit den Eingaben des Skripts. Es folgt der *Vorspann* mit den Einstellungen für das Laufzeitverhalten, vor allem `Set-StrictMode` und `$ErrorActionPreference`. Den Schluss bilden *Funktionen* und der *Hauptablauf*, der diese Funktionen aufruft. Wichtig ist die Reihenfolge: Funktionen müssen vor ihrem ersten Aufruf definiert sein, weil PowerShell Skripte von oben nach unten abarbeitet.

Das folgende vollständige Skript zeigt diese Struktur an einem kleinen Beispiel, das die Dateien eines Ordners in einen Sicherungsordner kopiert. Speichern Sie es als `Sichere-Kopie.ps1`.

```powershell
#!/usr/bin/env pwsh
#Requires -Version 7.0
<#
.SYNOPSIS
Kopiert Dateien eines Ordners in einen Sicherungsordner.
.PARAMETER Quelle
Quellordner.
.PARAMETER Ziel
Zielordner (wird angelegt).
.EXAMPLE
./Sichere-Kopie.ps1 -Quelle ./daten -Ziel ./sicherung
#>
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)][string] $Quelle,
    [Parameter(Mandatory)][string] $Ziel
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

function Copy-Sicherung {
    [CmdletBinding(SupportsShouldProcess)]
    param([string] $Von, [string] $Nach)
    if (-not (Test-Path -LiteralPath $Von -PathType Container)) {
        throw "Quellordner nicht gefunden: $Von"
    }
    if (-not (Test-Path -LiteralPath $Nach)) {
        New-Item -Path $Nach -ItemType Directory | Out-Null
    }
    foreach ($datei in Get-ChildItem -LiteralPath $Von -File) {
        if ($PSCmdlet.ShouldProcess($datei.Name, 'Kopieren')) {
            Copy-Item -LiteralPath $datei.FullName -Destination $Nach
            Write-Verbose "Kopiert: $($datei.Name)"
        }
    }
}

try {
    Copy-Sicherung -Von $Quelle -Nach $Ziel
    Write-Information "Fertig: $Quelle -> $Ziel" -InformationAction Continue
    exit 0
}
catch {
    Write-Error $_
    exit 1
}
```

Die Datei beginnt mit der *Shebang*-Zeile `#!/usr/bin/env pwsh`, die weiter unten erklärt wird. Es folgen die Anforderung an PowerShell 7 und der Hilfeblock. `[CmdletBinding(SupportsShouldProcess)]` macht das Skript zu einem Programm mit den gemeinsamen Parametern, darunter `-Verbose` und `-WhatIf`. Der Block `param()` deklariert zwei Pflichtparameter vom Typ Text; Typen und Validierung vertieft Lerneinheit 4.7. `Set-StrictMode -Version Latest` lässt PowerShell den Zugriff auf nicht definierte Variablen als Fehler melden, und `$ErrorActionPreference = 'Stop'` verwandelt Fehler, die sonst nur gemeldet würden, in abbrechende Fehler, die sich mit `try` und `catch` abfangen lassen (Modul 5). Die Funktion `Copy-Sicherung` prüft den Quellordner und wirft bei Bedarf einen Fehler, legt den Zielordner an und kopiert Datei für Datei. Dabei fragt `$PSCmdlet.ShouldProcess()` vor jeder Änderung, ob sie ausgeführt oder (bei `-WhatIf`) nur angezeigt werden soll. Der Hauptablauf am Ende ruft die Funktion auf, meldet den Erfolg mit `Write-Information` und beendet das Skript mit `exit 0`; bei einem Fehler schreibt der `catch`-Block die Fehlermeldung und beendet das Skript mit `exit 1`.

Im Test (PowerShell 7.6.6 unter Linux) lief das Skript wie erwartet. Mit `-Verbose` erschienen die beiden Zeilen „VERBOSE: Kopiert: a.txt“ und „VERBOSE: Kopiert: b.txt“, und der Exit Code war 0. Mit `-WhatIf` erschienen die Meldungen „What if: Performing the operation "Create Directory" on target ...“ und je eine für die beiden Kopiervorgänge, und es wurde nichts verändert. Beachten Sie dabei einen Effekt, der Einsteiger überrascht: `-WhatIf` wirkt auch auf die Befehle *innerhalb* der Funktion. Das `New-Item` wurde deshalb ebenfalls nur angezeigt, nicht ausgeführt. Mit einem nicht vorhandenen Quellordner brach das Skript mit der Meldung des `throw` ab, und der Exit Code war 1.

Ein weiterer Test betrifft fehlende Pflichtparameter. In einer interaktiven Sitzung fragt PowerShell danach; ohne Benutzer, etwa in einem geplanten Task, würde das Skript dabei hängen. Mit dem Schalter `-NonInteractive` bricht es stattdessen sofort mit einer klaren Meldung ab. Im Test lautete sie: „Cannot process command because of one or more missing mandatory parameters: Quelle Ziel.“ bei Exit Code 1. Automatisierte Aufrufe sollten deshalb immer mit `-NonInteractive` gestartet werden.

### Shebang und Aufruf.

Das Zeichen `#` leitet in PowerShell einen Kommentar ein, die erste Zeile `#!/usr/bin/env pwsh` ist für PowerShell also bedeutungslos. Unter Linux und macOS liest dagegen das Betriebssystem diese Zeile (die *Shebang*, von „hash“ und „bang“) und startet das Skript mit dem dort genannten Programm. Voraussetzung ist, dass die Datei ausführbar ist (`chmod +x Sichere-Kopie.ps1`) und Unix-Zeilenenden (LF) verwendet. Danach genügt `./Sichere-Kopie.ps1 -Quelle ./daten -Ziel ./sicherung`; im Test funktionierte das genau so. Die Form `/usr/bin/env pwsh` sucht `pwsh` im Suchpfad und ist damit unabhängig vom Installationsort. Unter Windows ignoriert PowerShell die Zeile; dort startet man Skripte über PowerShell selbst.

Es gibt mehrere Wege, ein Skript zu starten, die sich unterscheiden. `pwsh -NoProfile -File ./skript.ps1 -Parameter Wert` startet eine neue Sitzung, führt die Datei aus und liefert deren Exit Code; das ist der Normalfall für automatisierte Aufrufe. `pwsh -Command "..."` führt dagegen einen Befehlstext aus. Mit `pwsh -c 'exit 7'` ergab der Exit Code im Test 7. Innerhalb einer laufenden Sitzung startet `./skript.ps1` das Skript in einem eigenen Gültigkeitsbereich, sodass seine Variablen danach verschwunden sind. Mit dem Punkt-Operator, dem *Dot-Sourcing* (`. ./skript.ps1`), läuft es im aktuellen Gültigkeitsbereich, sodass Funktionen und Variablen danach zur Verfügung stehen; so lädt man Hilfsfunktionen aus einer Bibliotheksdatei. Unter Windows kann zusätzlich die *Execution Policy* den Start von Skripten einschränken (Modul 8).

> **Ergänzung aus der Nachrecherche**
>
> **Unterschied zwischen Windows PowerShell und PowerShell 7 beim Start (Fachwissen, nicht live abgerufen; siehe Quellen).** In `pwsh` ist der erste Positionsparameter `-File` statt `-Command`: `pwsh skript.ps1` führt die Datei aus, während Befehlstexte ausdrücklich mit `-c` oder `-Command` angegeben werden müssen. Außerdem versteht `-File` die Werte `$true` und `$false` für Parameter vom Typ Schalter und Boolean. Der Aufruf mit `-i` (interaktiv) steht für den Einsatz als Standardshell unter Unix zur Verfügung. Das Skript sollte seinen Exit Code ausdrücklich setzen; für Fehler in Pipelines gilt der Wert ungleich 0.

> **Merke**
>
> Struktur eines Skripts: `#Requires`, Hilfeblock, `param()`, `Set-StrictMode` und `$ErrorActionPreference`, Funktionen, Hauptablauf mit `try/catch` und Exit Code. Aufruf automatisiert mit `pwsh -NoProfile -NonInteractive -File`. Unter Unix: Shebang `#!/usr/bin/env pwsh`, ausführbar machen, LF-Zeilenenden.

Typische Fehler sind Funktionen, die erst unter dem Aufruf stehen, und fest verdrahtete Pfade zum Ort des Skripts. Wer ein Skript aus einem anderen Ordner startet, findet seine Begleitdateien nicht, wenn es sich auf das aktuelle Arbeitsverzeichnis verlässt. Die automatische Variable `$PSScriptRoot` enthält dagegen immer den Ordner, in dem das Skript liegt.

```powershell
# FALSCH: relativer Pfad, hängt vom aktuellen Ordner ab
$konfig = Get-Content -Path .\konfig.json

# RICHTIG: Pfad relativ zum Skript aufbauen
$konfig = Get-Content -LiteralPath (Join-Path -Path $PSScriptRoot -ChildPath 'konfig.json')
```

Im zweiten Fall funktioniert das Skript unabhängig davon, aus welchem Ordner es gestartet wird, und der Pfadaufbau mit `Join-Path` ist plattformunabhängig.

> **Praxisbeispiel**
>
> Ein Team legt eine Vorlagendatei für neue Skripte an, die genau diese Struktur enthält (Voraussetzungen, Hilfeblock, Parameter, Strict Mode, `try/catch`, Exit Code). Jedes neue Skript beginnt als Kopie der Vorlage. Das verhindert, dass Skripte ohne Fehlerbehandlung in Produktion landen, und macht Code-Reviews einfacher, weil alle Skripte gleich aufgebaut sind.

## 4.2 Variablen und Typisierung

Eine Variable ist ein benannter Speicherplatz für einen Wert. In PowerShell beginnt jede Variable mit dem Dollarzeichen und wird durch einfache Zuweisung angelegt: `$server = 'srv-web01'`. Eine Deklaration ist nicht nötig. Namen sind nicht groß-/kleinschreibungsabhängig, dürfen Buchstaben, Ziffern und Unterstriche enthalten und sollten sprechend sein. Voreingestellt ist PowerShell *dynamisch typisiert*: Die Variable nimmt den Typ des Werts an, den man ihr zuweist, und kann später einen anderen Typ aufnehmen. Das ist bequem, kann aber zu Fehlern führen, wenn PowerShell Werte *stillschweigend* umwandelt.

Mit einer *Typangabe* in eckigen Klammern vor dem Namen legt man den Typ dauerhaft fest: `[int]$anzahl = 5`. Danach wandelt PowerShell jede Zuweisung in diesen Typ um oder meldet einen Fehler, wenn das nicht geht. Die wichtigsten Typen sind `[string]` (Text), `[int]` und `[long]` (Ganzzahlen), `[double]` und `[decimal]` (Dezimalzahlen), `[bool]` (Wahrheitswert), `[datetime]` (Datum und Zeit), `[array]` und `[hashtable]` (Sammlungen) sowie `[pscustomobject]`.

```powershell
$a = '5'
$a + 3
3 + $a

[int]$n = '5'
try { $n = 'abc' } catch { $_.Exception.Message }

[int]3.5
[int]2.5
[int](5 / 2)
[int]'0x1F'
'5' * 3
```

Die ersten beiden Rechnungen zeigen die wichtigste Regel: *Der Typ des linken Operanden bestimmt das Ergebnis.* `'5' + 3` ist Textverkettung und ergibt `53`, während `3 + '5'` den Text in eine Zahl umwandelt und `8` ergibt. Als Nächstes legt `[int]$n` den Typ fest, sodass die Zuweisung von „abc“ scheitert, mit der Meldung „Cannot convert value "abc" to type "System.Int32"“. Die Rundungszeilen zeigen ein oft überraschendes Detail: `[int]3.5` ergibt 4, aber `[int]2.5` ergibt 2. PowerShell rundet beim Umwandeln auf die nächste *gerade* Zahl (Bankers Rounding), und `[int](5/2)` liefert deshalb 2, obwohl die Division 2,5 ergibt. Wer kaufmännisch runden will, verwendet `[math]::Round($x, [MidpointRounding]::AwayFromZero)`. Zuletzt zeigt `[int]'0x1F'` die Hexadezimalumwandlung (31) und `'5' * 3` die Textwiederholung (`555`).

### Gültigkeitsbereiche.

Jede Variable gehört zu einem *Scope*. Es gibt den globalen Scope der Sitzung, den Scope eines Skripts und den lokalen Scope einer Funktion oder eines Skriptblocks. Das Prinzip heißt *dynamischer Gültigkeitsbereich*: Beim *Lesen* sucht PowerShell die Variable zuerst im aktuellen Scope und dann nach außen; beim *Schreiben* entsteht dagegen immer eine neue lokale Variable, die die äußere nur überdeckt. Damit sind Funktionen vor ungewollten Nebenwirkungen geschützt, aber auch überraschend: Eine Funktion sieht die Variablen ihres *Aufrufers*, nicht nur die des Orts, an dem sie definiert wurde.

```powershell
$x = 10
function Test-Scope {
    $x = 20
    "innen: $x"
    $script:y = 'gesetzt im Skript-Scope'
    "global x: $global:x"
}
Test-Scope
"außen: $x"
$y

function Test-Lesen { "lesen: $x"; $x++; "nach x++: $x" }
Test-Lesen
"außen danach: $x"

function F1 { $x = 1; F2 }
function F2 { "F2 sieht x=$x" }
F1
```

`Test-Scope` setzt ein lokales `$x` auf 20; das äußere `$x` bleibt 10. Mit den Präfixen `$script:` und `$global:` spricht man gezielt einen anderen Scope an; die Variable `$y` bleibt danach sichtbar. `Test-Lesen` liest zunächst das äußere `$x` (10), aber `$x++` legt eine neue lokale Variable mit dem Wert 11 an; draußen bleibt der Wert bei 10. Die letzten Zeilen zeigen den dynamischen Scope: `F2` sieht das `$x` von `F1` („F2 sieht x=1“), obwohl es dort nicht definiert wurde. Globale und Skriptvariablen sind trotzdem ein Warnsignal: Funktionen sollten ihre Eingaben über Parameter erhalten und ihr Ergebnis zurückgeben, nicht über gemeinsame Variablen kommunizieren.

### Zeichenketten.

Einfache Anführungszeichen erzeugen wörtlichen Text: `'Hallo $name'` gibt genau das aus. Doppelte Anführungszeichen *expandieren* Variablen und Ausdrücke: `"Hallo $name"` setzt den Wert ein. Für Eigenschaften und Berechnungen braucht man den Teilausdruck `$()`: `"Länge: $($text.Length)"`. Ohne ihn zeigt `"$text.Length"` nur den Wert der Variable gefolgt vom Text „.Length“ (im Test: `abc.Length`). Mehrzeilige Texte schreibt man als *Here-String* mit `@" ... "@` (expandierend) oder `@' ... '@` (wörtlich); die schließende Marke muss am Zeilenanfang stehen.

```powershell
$name = 'Welt'
"Hallo $name"
'Hallo $name'
"Summe: $(1 + 2)"
$text = 'abc'
"$text.Length"
"$($text.Length)"

@"
Zeile 1 für $name
  Zeile 2
"@
```

### Konstanten und automatische Variablen.

Werte, die sich nicht ändern sollen, legt man mit `Set-Variable -Name PI2 -Value 6.28 -Option Constant` an; eine spätere Zuweisung scheitert mit „Cannot overwrite variable PI2 because it is read-only or constant.“ Daneben gibt es *automatische Variablen*, die PowerShell selbst verwaltet: `$_` und `$PSItem` (aktuelles Pipeline-Objekt), `$?` (Erfolg des letzten Befehls), `$LASTEXITCODE` (Exit Code des letzten externen Programms), `$args` (nicht benannte Argumente), `$PSScriptRoot` (Ordner des Skripts), `$PSBoundParameters` (tatsächlich übergebene Parameter), `$Error` (Liste bisheriger Fehler), `$true`, `$false`, `$null` sowie `$HOME`, `$PID`, `$IsWindows`, `$IsLinux` und `$IsMacOS`. Diese Namen dürfen Sie nicht für eigene Variablen verwenden.

Zahlen verdienen eine eigene Bemerkung. Die Typen `[double]` sind binäre Gleitkommazahlen und können Dezimalbrüche nicht exakt darstellen: `0.1 + 0.2 -eq 0.3` ergibt `False`. Für Geldbeträge verwendet man `[decimal]`, wo `[decimal]0.1 + [decimal]0.2 -eq [decimal]0.3` wahr ist (im Test bestätigt). PowerShell bietet die Zahlensuffixe `KB`, `MB`, `GB` (`1GB` ist 1 073 741 824) sowie bei der Ganzzahlarithmetik eine automatische Erweiterung des Typs: `[int]::MaxValue + 1` ergibt `2147483648` statt eines Überlaufs.

> **Merke**
>
> Der linke Operand bestimmt die Operation (`'5' + 3` ist `53`, `3 + '5'` ist `8`). Typangaben (`[int]$x`) legen den Typ fest. Zuweisung erzeugt eine lokale Variable, Lesen sucht nach außen. Einfache Anführungszeichen sind wörtlich, doppelte expandieren; Eigenschaften brauchen `$()`. Für Geld `[decimal]`.

```powershell
# FALSCH: untypisierte Eingabe, stillschweigende Umwandlung
$menge = Read-Host 'Menge'
$gesamt = $menge + 1            # Eingabe 5 ergibt "51"

# RICHTIG: Typ festlegen, Eingabe prüfen
[int]$menge = Read-Host 'Menge'
$gesamt = $menge + 1            # ergibt 6
```

`Read-Host` liefert immer Text. Im ersten Fall entsteht durch die Verkettung „51“, im zweiten wird die Eingabe schon bei der Zuweisung in eine Zahl umgewandelt, und eine ungültige Eingabe führt zu einem klaren Fehler statt einem falschen Ergebnis.

> **Praxisbeispiel**
>
> In Administrationsskripten stammen viele Werte aus Text: CSV-Dateien, Parameter, Umgebungsvariablen, Webantworten. Wer die Zielwerte konsequent typisiert (`[int]`, `[datetime]`, `[bool]`), findet Eingabefehler an der Stelle, wo sie entstehen, und nicht erst im Bericht, in dem eine Summe seltsam aussieht.

## 4.3 Arrays und Hashtables

Einzelne Werte reichen selten aus. Für Listen von Servern, Dateien oder Benutzern braucht man *Arrays*, für Zuordnungen wie „Servername zu Rolle“ *Hashtables*. Beide Strukturen begegnen Ihnen in jedem nicht trivialen Skript.

Ein *Array* ist eine geordnete Liste von Werten, die über eine Nummer (den *Index*) angesprochen werden, wobei die Zählung bei 0 beginnt. Man erzeugt es mit dem Komma-Operator oder mit `@()`: `$z = 1, 2, 3` oder `$z = 1..5` (Bereichsoperator). `$z[0]` liefert das erste Element, `$z[-1]` das letzte, `$z[1..3]` einen Ausschnitt, und `$z.Count` die Anzahl.

Eine wichtige Eigenschaft ist, dass ein Array eine *feste Größe* hat. Der Operator `+=` fügt daher nicht an, sondern erzeugt ein *neues, größeres Array*, kopiert alle bisherigen Elemente hinein und ersetzt die Variable. Bei wenigen Elementen ist das unproblematisch, bei Tausenden wird es spürbar langsam. Im Test dauerte das Anhängen von 20 000 Zahlen mit `+=` etwa 1,5 Sekunden, mit einer Liste (`[System.Collections.Generic.List[int]]`) etwa 0,04 Sekunden. Für wachsende Sammlungen ist die generische Liste die richtige Wahl.

```powershell
$z = 1..5
$z[0]
$z[-1]
$z[1..3] -join ','
$z.IsFixedSize
try { $z.Add(6) } catch { $_.Exception.Message }
$z += 6
$z.Count

$liste = [System.Collections.Generic.List[string]]::new()
$liste.Add('a')
$liste.Add('b')
$liste.Count
$liste -join ','

$a1 = @(1, 2)
$a2 = $a1
$a2 += 3
$a1.Count
```

Die Zeilen zeigen: `IsFixedSize` ist `True`, und `.Add()` scheitert mit „Collection was of a fixed size.“ Mit `+=` entsteht dagegen ein neues Array mit sechs Elementen. Die Liste `List[string]` hat dagegen eine veränderliche Größe und die Methode `.Add()`. Am Ende zeigt das Beispiel, dass `$a2 += 3` ein neues Array erzeugt: `$a1` behält seine zwei Elemente.

Ein häufiger Stolperstein ist das *Entpacken von Einzelwerten*. Liefert ein Befehl genau ein Objekt, gibt PowerShell dieses Objekt zurück und kein Array mit einem Element, und bei keinem Ergebnis `$null`. Das hat in Skripten, die `.Count` oder Indizes verwenden, schon viele Fehler verursacht. In neueren Versionen haben auch einzelne Objekte eine `.Count`-Eigenschaft (im Test 1 für eine einzelne Datei), und der Typ bleibt `FileInfo`. Mit `@()` erzwingt man ein Array: `@(Get-ChildItem ...)` liefert im Test den Typ `Object[]`, bei keinem Treffer ein leeres Array mit `Count` 0.

Eine *Hashtable* speichert Paare aus *Schlüssel* und *Wert*. Man erzeugt sie mit `@{ Schlüssel = Wert }`. Der Zugriff erfolgt über den Schlüssel in eckigen Klammern oder als Eigenschaft. Schlüssel sind bei Texten nicht groß-/kleinschreibungsabhängig; die Reihenfolge der Einträge ist nicht garantiert, es sei denn, man verwendet `[ordered]`. Hashtables sind ideal für Nachschlagetabellen, Konfigurationen und Parameter-Sammlungen (Splatting, Lerneinheit 4.8).

```powershell
$h = @{ Name = 'srv01'; Rolle = 'Web' }
$h['Name']
$h.Rolle
$h.ContainsKey('Port')
$h['Port'] = 443
$h.Count
$h['name']

$o = [ordered]@{ z = 1; a = 2 }
$o.Keys -join ','

foreach ($k in $h.Keys) { "$k=$($h[$k])" }
$h.GetEnumerator() | Sort-Object -Property Name | ForEach-Object { "$($_.Key)=$($_.Value)" }

$h3 = @{ a = 1 }
$h4 = $h3
$h4.b = 2
$h3.Count

@{ a = 1 } + @{ b = 2 }
```

Die ersten Zeilen lesen Werte, prüfen mit `ContainsKey`, ob ein Schlüssel existiert, und fügen `Port` hinzu. `$h['name']` findet den Wert trotz Kleinschreibung. Die `[ordered]`-Variante liefert die Schlüssel in der Reihenfolge `z,a`. Zur Auflistung verwendet man `.Keys` oder `.GetEnumerator()`; Letzteres liefert Schlüssel-Wert-Paare mit den Eigenschaften `Key` und `Value`. Wichtig ist das Verhalten bei Zuweisung: `$h4 = $h3` kopiert nicht, beide Variablen zeigen auf *dieselbe* Hashtable, deshalb zählt `$h3.Count` nach dem Ändern über `$h4` bereits 2 Einträge (im Test bestätigt). Eine echte Kopie liefert `.Clone()`, die allerdings nur flach kopiert (verschachtelte Hashtables bleiben geteilt). Der Operator `+` verbindet zwei Hashtables, scheitert aber bei doppelten Schlüsseln mit „Item has already been added“.

> **Ergänzung aus der Nachrecherche**
>
> **Hashtables in der Microsoft-Dokumentation.** Der Leitfaden zu Hashtables weist auf mehrere Fallen hin. Erstens „BadEnumeration“: Man darf eine Hashtable nicht verändern, während man über ihre Schlüssel läuft; die Meldung lautet „Collection was modified; enumeration operation may not execute.“ Im Test brach die Schleife `foreach ($k in $h.Keys) { $h[$k] = 'x' }` nach dem ersten Eintrag mit genau diesem Fehler ab; ein Eintrag war da schon verändert. Abhilfe schafft das Durchlaufen einer Kopie der Schlüssel (`@($h.Keys)` oder `$h.Keys.Clone()`). Zweitens die Prüfung mit `if ($h.Alter)`, die bei einem Wert 0 oder `$false` fälschlich „nicht vorhanden“ meldet: Für die Existenz eines Schlüssels nimmt man `ContainsKey`. Drittens die Besonderheit, dass eine Hashtable in einer Pipeline als *ein* Objekt gilt: `$h | Measure-Object` zählt 1; für die Einträge braucht man `.GetEnumerator()`. Viertens: Eine als JSON gespeicherte und wieder gelesene Hashtable ist ein `PSCustomObject`, außer man liest mit `ConvertFrom-Json -AsHashtable`.

> **Merke**
>
> Arrays: Index ab 0, feste Größe, `+=` kopiert; für wachsende Listen `List[T]`. `@()` erzwingt ein Array. Hashtables: Schlüssel-Wert-Paare, Zuweisung kopiert nicht (Referenz), `.Clone()` ist flach, `ContainsKey` prüft die Existenz, `[ordered]` bewahrt die Reihenfolge.

```powershell
# FALSCH: Array in Schleife mit += aufbauen und Hashtable-Existenz mit Wahrheitswert prüfen
$ergebnis = @()
foreach ($i in 1..20000) { $ergebnis += $i }
if ($konfig.Port) { 'Port gesetzt' }          # schlägt bei Port = 0 fehl

# RICHTIG: Liste verwenden, ContainsKey prüfen
$ergebnis = [System.Collections.Generic.List[int]]::new()
foreach ($i in 1..20000) { $ergebnis.Add($i) }
if ($konfig.ContainsKey('Port')) { 'Port gesetzt' }
```

> **Praxisbeispiel**
>
> Beim Ermitteln von Ergebnissen in einer Schleife über viele Rechner ist die Variante `$ergebnisse = foreach ($s in $server) { ... }` oft die eleganteste: Die Schleife selbst sammelt die Ausgaben in einem Array, ohne `+=` und ohne Liste. Hashtables dienen als Nachschlagetabellen, etwa um aus dem Servernamen die Rolle oder den Verantwortlichen zu bestimmen, ohne bei jedem Zugriff die ganze Liste zu durchsuchen.

## 4.4 If, Switch und Vergleichslogik

Skripte müssen Entscheidungen treffen: Ist der Dienst gestartet? Reicht der Speicherplatz? Welche Aktion gehört zu diesem Statuscode? PowerShell bietet dafür `if` mit `elseif` und `else` sowie die Mehrfachverzweigung `switch`. Die Bedingungen nutzen die Vergleichsoperatoren aus Lerneinheit 3.3 und die logischen Operatoren `-and`, `-or`, `-not` und `-xor`.

Eine `if`-Bedingung muss kein Vergleich sein. PowerShell wandelt jeden Wert in wahr oder falsch um, nach festen Regeln. Als *falsch* gelten `$false`, die Zahl 0, der leere Text, `$null`, ein leeres Array und ein Array mit genau einem Element, das selbst falsch ist. Alles andere ist *wahr*. Das hat Überraschungen zur Folge, die im Test geprüft wurden: Der Text `'0'` ist wahr (denn er ist nicht leer), ebenso `'false'` und ein einzelnes Leerzeichen. Eine leere Hashtable (`@{}`) ist *wahr*, ebenso ein leeres `PSCustomObject`. Ein Array mit dem Element 0 ist falsch, eines mit zwei Elementen ist wahr.

```powershell
foreach ($v in $false, 0, '', $null, @(), @(0), '0', 'false', @{}) {
    $anzeige = if ($null -eq $v) { '$null' } elseif ($v -is [array]) { "Array($($v.Count))" } else { "'$v'" }
    '{0,-12} -> {1}' -f $anzeige, [bool]$v
}

$p = 42
if ($p -gt 100) { 'sehr groß' }
elseif ($p -gt 10) { 'groß' }
else { 'klein' }

$ergebnis = if ($p -gt 10) { 'groß' } else { 'klein' }
$p -gt 10 ? 'groß' : 'klein'
$null ?? 'Vorgabe'
$w = $null
$w ??= 'gesetzt'
$w
```

Die Schleife am Anfang gibt für verschiedene Werte aus, wie PowerShell sie als Bedingung auswertet; die Ausgabe bestätigt die Regeln oben. Es folgt ein `if`-Block mit zwei Alternativen. Danach zeigt das Beispiel, dass `if` auch einen Wert *zurückgibt*, den man einer Variablen zuweisen kann. Die letzten Zeilen zeigen drei Kurzformen aus PowerShell 7: den *ternären Operator* `Bedingung ? A : B`, den *Null-Coalescing-Operator* `??`, der den linken Wert liefert, sofern er nicht `$null` ist, und den Zuweisungsoperator `??=`, der nur zuweist, wenn die Variable `$null` ist. In Windows PowerShell 5.1 gibt es diese drei Operatoren nicht.

Bei Vergleichen mit `$null` gehört `$null` *nach links*: `$null -eq $x`. Steht links ein Array, filtert der Operator, wie in Lerneinheit 3.3 gezeigt: `@() -eq $null` liefert nichts (also falsch) und nicht wahr. Auch `if (@(0) -eq 0)` ist deshalb falsch, weil das Ergebnis des Vergleichs das Element 0 ist, und die Zahl 0 als falsch gilt. Im Test lief dieses `if` nicht in den Block.

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Operatoren in PowerShell 7 (Fachwissen, nicht live abgerufen; siehe Quellen).** Neben dem Null-Coalescing-Operator gibt es die Null-bedingten Operatoren `?.` und `?[]` für Eigenschafts- und Elementzugriff, die bei `$null` das Ergebnis `$null` liefern, statt einen Fehler zu erzeugen. Weil das Fragezeichen in Variablennamen vorkommen darf, müssen die Namen dann in geschweifte Klammern gesetzt werden: `${a}?.Name`. Außerdem lässt sich eine Pipeline mit `&` am Ende als Hintergrundauftrag starten. Im Test lieferte `${obj}?.Name` mit `$obj = $null` ohne Fehler nichts zurück.

`switch` ersetzt lange Ketten aus `if` und `elseif`, wenn ein Wert mit mehreren Fällen verglichen wird. Es ist mächtiger, als es zunächst aussieht: Fälle können Zahlen, Texte, Platzhalter (`-Wildcard`), reguläre Ausdrücke (`-Regex`) oder beliebige Skriptblöcke sein; `-CaseSensitive` schaltet die Groß-/Kleinschreibung ein, und `default` fängt alles andere ab. Ein wesentlicher Unterschied zu vielen anderen Sprachen: *Alle zutreffenden Fälle werden ausgeführt*, nicht nur der erste. Nur ein `break` beendet das `switch`. Bekommt `switch` eine Liste, prüft es jedes Element einzeln; ein `break` beendet dann die gesamte Verarbeitung.

```powershell
switch (404) {
    200     { 'OK' }
    404     { 'Nicht gefunden' }
    500     { 'Serverfehler' }
    default { 'Unbekannt' }
}

switch -Wildcard ('srv-web01') {
    'srv-web*' { 'Web' }
    'srv-db*'  { 'Datenbank' }
}

switch -Regex ('ERROR 42') {
    '^ERROR' { 'Fehler' }
    '\d+'    { 'enthält eine Zahl' }
}

switch ('Datei.TXT') {
    { $_ -like '*.txt' } { 'Textdatei' }
    { $_ -like 'd*' }    { 'beginnt mit d' }
}

switch (1, 2, 3) { 1 { 'eins' } 2 { 'zwei'; break } 3 { 'drei' } }
```

Der erste `switch` bildet einen Statuscode auf eine Meldung ab („Nicht gefunden“). Der zweite verwendet Platzhalter; die Servernamen-Muster liefern „Web“. Der dritte zeigt *Fall-through*: Beide Muster treffen zu, es erscheinen deshalb „Fehler“ *und* „enthält eine Zahl“. Im vierten bestehen die Fälle aus Skriptblöcken mit Bedingungen; auch hier laufen beide zutreffenden Fälle („Textdatei“ und „beginnt mit d“). Der letzte `switch` bekommt drei Werte: Er gibt „eins“ und „zwei“ aus; durch `break` im zweiten Fall wird die Verarbeitung beendet, „drei“ erscheint nie.

> **Merke**
>
> Wahrheitswerte: falsch sind `$false`, 0, `''`, `$null`, leere Arrays und Einzelelement-Arrays mit falschem Element; `'0'`, `'false'` und leere Hashtables sind wahr. `$null` steht links. `switch` führt alle zutreffenden Fälle aus; `break` beendet ihn.

```powershell
# FALSCH: Annahme, switch nehme nur den ersten Treffer
switch -Wildcard ($name) {
    'srv-*'     { $rolle = 'Server' }
    'srv-web*'  { $rolle = 'Web' }       # überschreibt, obwohl zuerst 'srv-*' zutraf
}

# RICHTIG: spezifischen Fall zuerst, danach break
switch -Wildcard ($name) {
    'srv-web*' { $rolle = 'Web';    break }
    'srv-*'    { $rolle = 'Server'; break }
}
```

Im falschen Beispiel laufen beide Zweige, und der Zustand hängt von der Reihenfolge ab. Mit spezifischen Fällen zuerst und `break` entsteht eine eindeutige Entscheidung.

> **Praxisbeispiel**
>
> Eine Wartungsroutine bewertet den freien Speicherplatz einer Platte: unter 5 % „kritisch“, unter 15 % „Warnung“, sonst „in Ordnung“. Die Grenzen stehen am besten in einer Konfigurationsdatei (Lerneinheit 4.9), und die Entscheidung trifft ein `switch` mit Skriptblock-Bedingungen, der eine Statusangabe für die Überwachung liefert.

## 4.5 Schleifen

Schleifen wiederholen Anweisungen. PowerShell kennt vier Formen. Die `foreach`-Schleife durchläuft jedes Element einer Sammlung: `foreach ($s in $server) { ... }`. Die `for`-Schleife zählt mit einem Startwert, einer Bedingung und einem Schritt: `for ($i = 0; $i -lt 10; $i++) { ... }`. Die `while`-Schleife prüft die Bedingung *vor* jedem Durchlauf, die `do`-Schleife *nach* dem Durchlauf, entweder mit `while` (wiederholen, solange die Bedingung wahr ist) oder mit `until` (wiederholen, bis sie wahr wird). Eine `do`-Schleife läuft deshalb mindestens einmal.

```powershell
foreach ($i in 1..3) { "foreach $i" }
"nach der Schleife: $i"

for ($i = 0; $i -lt 3; $i++) { "for $i" }

$n = 0
while ($n -lt 3) { $n++ }
$n

$n = 10
do { $n++ } while ($n -lt 3)
$n

$n = 0
do { $n++ } until ($n -ge 3)
$n
```

Im Test ergaben die Schleifen die erwarteten Werte: `foreach` gab 1, 2 und 3 aus, und die Variable `$i` behielt den letzten Wert (3). Das ist wichtig: Die Schleifenvariable endet *nicht* mit der Schleife. Die `while`-Schleife zählte bis 3. Die `do-while`-Schleife lief trotz falscher Bedingung einmal (`$n` wurde 11), die `do-until`-Schleife bis 3.

Mit `break` verlässt man die innerste Schleife sofort, mit `continue` springt man zum nächsten Durchlauf. Bei verschachtelten Schleifen lassen sich die äußeren mit einer *Marke* ansteuern: `:aussen foreach (...)` und dann `continue aussen` oder `break aussen`.

```powershell
foreach ($i in 1..10) {
    if ($i % 2) { continue }      # ungerade überspringen
    if ($i -gt 6) { break }       # bei 8 abbrechen
    $i
}

:aussen foreach ($a in 1..3) {
    foreach ($b in 1..3) {
        if ($b -eq 2) { continue aussen }
        "$a-$b"
    }
}
```

Die erste Schleife gibt 2, 4 und 6 aus: Ungerade Zahlen werden übersprungen, bei 8 endet sie. Die zweite gibt `1-1`, `2-1` und `3-1` aus, weil `continue aussen` nach dem ersten Durchlauf der inneren Schleife direkt zum nächsten Wert der äußeren springt.

> **Ergänzung aus der Nachrecherche**
>
> **Falle: break und continue in ForEach-Object.** Der Pipeline-Befehl `ForEach-Object` ist keine Schleifenanweisung, sondern ein Cmdlet mit einem Skriptblock. Ein `break` darin beendet deshalb *nicht* nur die Pipeline, sondern die nächste umgebende Schleife oder, wenn es keine gibt, das ganze Skript. Im Test gab `1..3 | ForEach-Object { if ($_ -eq 2) { break }; $_ }` nur die 1 aus, und die Anweisung nach der Pipeline wurde nie erreicht. In einer umgebenden `foreach`-Schleife endete dagegen die äußere Schleife. Zum Überspringen eines Elements verwendet man in `ForEach-Object` `return` (im Test: 1, 3, 4), zum vorzeitigen Beenden einer Pipeline `Select-Object -First N`. Bei Bedarf nutzen Sie statt der Pipeline die `foreach`-Anweisung, in der `break` und `continue` wie erwartet funktionieren.

Schleifen sind der richtige Ort für *Wiederholversuche* (Retry). Bei Netzwerkoperationen und Cloud-Aufrufen scheitern einzelne Versuche aus vorübergehenden Gründen, und ein kontrollierter Wiederholungsmechanismus erhöht die Zuverlässigkeit erheblich. Das folgende Beispiel zeigt ein bewährtes Muster mit begrenzter Zahl von Versuchen und wachsender Wartezeit.

```powershell
function Invoke-MitWiederholung {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [scriptblock] $Aktion,
        [int] $MaxVersuche = 3,
        [int] $WartenMs = 100
    )
    for ($versuch = 1; $versuch -le $MaxVersuche; $versuch++) {
        try {
            return & $Aktion
        }
        catch {
            Write-Warning "Versuch $versuch von $MaxVersuche fehlgeschlagen: $($_.Exception.Message)"
            if ($versuch -eq $MaxVersuche) { throw }
            Start-Sleep -Milliseconds ($WartenMs * [math]::Pow(2, $versuch - 1))
        }
    }
}

$zaehler = 0
Invoke-MitWiederholung -Aktion {
    $script:zaehler++
    if ($script:zaehler -lt 3) { throw "noch nicht ($script:zaehler)" }
    "Erfolg nach $script:zaehler Versuchen"
}
```

Die Funktion führt den übergebenen Skriptblock aus. Gelingt er, endet sie mit `return` und gibt das Ergebnis zurück. Scheitert er, gibt der `catch`-Block eine Warnung aus; beim letzten Versuch wird der Fehler mit `throw` erneut ausgelöst, sonst wartet die Funktion und verdoppelt dabei die Wartezeit (`[math]::Pow(2, $versuch - 1)` liefert 1, 2, 4, ...). Im Test scheiterten die ersten beiden Versuche mit Warnungen, der dritte lieferte „Erfolg nach 3 Versuchen“. Ein Aktionsblock, der immer scheitert, löste nach den zulässigen Versuchen den Fehler aus. Das `$script:zaehler` im Aktionsblock ist nötig, weil der Block in einem eigenen Scope läuft.

Beim Durchlaufen von Sammlungen gilt außerdem: Die Sammlung wird zu Beginn der `foreach`-Schleife festgelegt. Im Test änderte die Zuweisung `$liste = @()` im Schleifenrumpf nichts mehr an den verbleibenden Durchläufen. Wer die Sammlung selbst verändert (Elemente entfernen oder hinzufügen), erhält bei Hashtables und Listen den Fehler „Collection was modified“; dann durchläuft man eine Kopie.

> **Merke**
>
> `foreach` für Sammlungen, `for` zum Zählen, `while` und `do` für Bedingungen, `do` läuft mindestens einmal. `break`/`continue` wirken auf die innerste Schleife, mit Marken auf äußere. In `ForEach-Object` ist `return` das `continue`, und `break` beendet die umgebende Schleife oder das Skript.

```powershell
# FALSCH: Endlosschleife, wenn die Bedingung nie wahr wird
while (-not (Test-Path -Path $datei)) { Start-Sleep -Seconds 5 }

# RICHTIG: Obergrenze für die Wartezeit
$ende = (Get-Date).AddMinutes(2)
while (-not (Test-Path -LiteralPath $datei) -and (Get-Date) -lt $ende) {
    Start-Sleep -Seconds 5
}
if (-not (Test-Path -LiteralPath $datei)) { throw "Datei $datei erschien nicht innerhalb von 2 Minuten." }
```

Im ersten Fall hängt ein Skript für immer, wenn die Datei nie erscheint, und blockiert womöglich eine Pipeline. Im zweiten gibt es eine Frist und eine klare Fehlermeldung.

> **Praxisbeispiel**
>
> Typische Schleifen in der Administration: Alle Server einer Liste nacheinander abfragen und Ergebnisse sammeln, auf das Ende eines Deployments warten (Polling mit Frist), fehlgeschlagene Operationen wiederholen. In allen Fällen gehören eine Obergrenze und eine Fehlermeldung dazu, damit die Automation nicht unbemerkt hängt.

## 4.6 Funktionen

Eine *Funktion* fasst Anweisungen unter einem Namen zusammen, damit man sie wiederverwenden kann, ohne Code zu kopieren. Das gilt schon für kleine Skripte: Eine Funktion mit einem sprechenden Namen macht den Hauptablauf lesbar, lässt sich einzeln testen und kann in mehreren Skripten genutzt werden. Funktionen folgen dem Namensschema Verb-Substantiv (Lerneinheit 1.4).

Die einfachste Form besteht aus dem Schlüsselwort `function`, dem Namen und einem Block. Parameter werden mit `param()` deklariert. Wichtig ist, wie eine Funktion Werte zurückgibt: PowerShell hat keine ausdrückliche Rückgabeanweisung als Pflicht. *Alles*, was während der Funktion in die Ausgabe gelangt, ist der Rückgabewert, also jeder Ausdruck, der nicht einer Variablen zugewiesen oder verworfen wird. Das Schlüsselwort `return` gibt einen Wert aus und beendet die Funktion, ist aber optional.

```powershell
function Add-Zahlen { param($A, $B) $A + $B }
Add-Zahlen 2 3

function Test-Rueckgabe { 'a'; return 'b'; 'c' }
Test-Rueckgabe

function Test-Leak {
    $liste = [System.Collections.ArrayList]::new()
    $liste.Add('x')
    'Ergebnis'
}
Test-Leak

function Test-KeinLeak {
    $liste = [System.Collections.ArrayList]::new()
    $null = $liste.Add('x')
    'Ergebnis'
}
Test-KeinLeak
```

`Add-Zahlen` gibt die Summe zurück, ohne dass `return` nötig wäre. `Test-Rueckgabe` liefert `a` und `b`; der Text `c` wird nicht mehr erreicht, weil `return` die Funktion beendet. `Test-Leak` zeigt den berüchtigten Fehler: Die Methode `Add` der `ArrayList` liefert den Index des neuen Elements zurück (im Test `0`), und dieser Wert gelangt in die Ausgabe, sodass die Funktion *zwei* Werte liefert. In `Test-KeinLeak` verwirft `$null = ...` die Ausgabe, und nur „Ergebnis“ bleibt. Alternativ eignen sich `| Out-Null` oder `[void](...)`.

Für professionelle Funktionen verwendet man das Attribut `[CmdletBinding()]`, das aus einer einfachen Funktion eine *Advanced Function* macht. Sie erhält dadurch alle gemeinsamen Parameter (`-Verbose`, `-Debug`, `-ErrorAction`, `-WhatIf`, `-Confirm` und weitere), prüft Parameter strenger und kann `$PSCmdlet` nutzen. Mit `begin`, `process` und `end` (Lerneinheit 1.6) verarbeitet sie Pipeline-Eingaben; `[OutputType()]` dokumentiert den Ausgabetyp.

```powershell
function Get-Gruss {
    [CmdletBinding()]
    param([string] $Name = 'Welt')
    Write-Verbose "Gruß für $Name"
    "Hallo $Name"
}
Get-Gruss
Get-Gruss -Name 'Anna' -Verbose

function Remove-Demo {
    [CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'High')]
    param([string] $Name)
    if ($PSCmdlet.ShouldProcess($Name, 'Löschen')) { "gelöscht: $Name" }
}
Remove-Demo -Name 'x' -WhatIf
Remove-Demo -Name 'x' -Confirm:$false

function Get-Quadrat {
    [CmdletBinding()]
    [OutputType([int])]
    param([Parameter(ValueFromPipeline)] [int] $Zahl)
    process { $Zahl * $Zahl }
}
1..4 | Get-Quadrat
```

`Get-Gruss` akzeptiert dank `[CmdletBinding()]` den Schalter `-Verbose`: Ohne ihn erscheint nur „Hallo Welt“, mit ihm zusätzlich die Zeile „VERBOSE: Gruß für Anna“. `Remove-Demo` zeigt das Muster für Befehle, die etwas verändern: `SupportsShouldProcess` schaltet `-WhatIf` und `-Confirm` frei, und `$PSCmdlet.ShouldProcess()` fragt, ob die Aktion ausgeführt werden soll. Mit `-WhatIf` erscheint nur „What if: Performing the operation "Löschen" on target "x".“; mit `-Confirm:$false` läuft die Aktion ohne Rückfrage. `ConfirmImpact = 'High'` sorgt dafür, dass ohne weitere Angabe nachgefragt wird. `Get-Quadrat` verarbeitet jedes Element der Pipeline im `process`-Block und liefert 1, 4, 9 und 16.

Zur Fehlersuche hilft `$PSBoundParameters`: Diese Variable enthält nur die Parameter, die der Aufrufer *tatsächlich übergeben hat*, nicht die mit Standardwerten. Im Test lieferte der Aufruf mit `-A 1` nur den Schlüssel `A`, obwohl `B` einen Standardwert hat.

> **Ergänzung aus der Nachrecherche**
>
> **Gemeinsame Parameter in PowerShell 7.6.** Im Test zeigte `(Get-Command Get-Gruss).Parameters.Keys` neben dem eigenen Parameter `Name` die gemeinsamen Parameter `Verbose`, `Debug`, `ErrorAction`, `WarningAction`, `InformationAction`, `ProgressAction`, `ErrorVariable`, `WarningVariable`, `InformationVariable`, `OutVariable`, `OutBuffer` und `PipelineVariable`. `ProgressAction` ist neuer als die übrigen und in Windows PowerShell 5.1 nicht vorhanden.

> **Merke**
>
> Funktionen: Verb-Substantiv, Eingaben über Parameter, Ergebnis über die Ausgabe. *Alles* Nicht-Verworfene ist Rückgabewert (`$null = ...` oder `| Out-Null` für Nebenergebnisse). `[CmdletBinding()]` macht eine Advanced Function mit `-Verbose`, `-WhatIf` und weiteren Parametern; `SupportsShouldProcess` für ändernde Funktionen.

```powershell
# FALSCH: Funktion verändert globale Variable und gibt Text aus
function Get-Anzahl {
    $global:anzahl = (Get-ChildItem).Count
    Write-Host "Anzahl: $global:anzahl"
}

# RICHTIG: Parameter rein, Wert raus
function Get-Anzahl {
    [CmdletBinding()]
    param([string] $Pfad = '.')
    (Get-ChildItem -LiteralPath $Pfad).Count
}
$n = Get-Anzahl -Pfad '.'
```

Die erste Fassung verändert eine globale Variable als Nebeneffekt und gibt ihr Ergebnis nur als Text auf den Bildschirm aus; nichts davon lässt sich weiterverarbeiten oder testen. Die zweite erhält ihre Eingabe über einen Parameter und gibt ein Ergebnis zurück.

> **Praxisbeispiel**
>
> Häufig verwendete Funktionen wandern in eine Bibliotheksdatei oder ein Modul und werden von mehreren Skripten genutzt. Fehlerkorrekturen müssen dann nur an einer Stelle erfolgen. Ein Pester-Test (Modul 5) prüft jede Funktion einzeln, was bei großen Skripten die Fehlersuche erheblich vereinfacht.

## 4.7 Parameter und Validation

Eine Funktion oder ein Skript ist nur so robust wie seine Eingaben. Wer Parameter nicht prüft, erlebt Fehler erst tief im Ablauf, oft mit Meldungen, die mit der eigentlichen Ursache nichts zu tun haben. Die *Parametervalidierung* verschiebt die Prüfung an den Eingang: PowerShell lehnt ungültige Werte ab, bevor der erste Befehl läuft, und meldet verständlich, was falsch ist. Dafür braucht man keinen eigenen Prüfcode, sondern nur Attribute am Parameter.

Das Attribut `[Parameter()]` steuert das Verhalten des Parameters. `Mandatory` macht ihn zur Pflicht, `Position` legt die Stelle für positionales Binden fest, `ValueFromPipeline` und `ValueFromPipelineByPropertyName` erlauben Pipeline-Eingaben (Lerneinheit 2.7), `ParameterSetName` ordnet ihn einem Parametersatz zu, und `HelpMessage` liefert einen Hilfetext. Mit `[Alias()]` erhält der Parameter weitere Namen. Ein Parameter vom Typ `[switch]` ist ein Schalter, der nur angegeben oder weggelassen wird.

Zur Validierung gibt es mehrere Attribute: `ValidateSet` erlaubt nur Werte aus einer Liste und ermöglicht die Tab-Vervollständigung, `ValidateRange` prüft Zahlenbereiche, `ValidatePattern` verlangt, dass ein Text zu einem regulären Ausdruck passt, `ValidateLength` begrenzt die Länge, `ValidateCount` die Anzahl der Elemente einer Liste, `ValidateNotNullOrEmpty` verbietet leere Werte, und `ValidateScript` erlaubt eine eigene Prüfung als Skriptblock, der `$true` oder `$false` liefern muss. Bei `ValidatePattern` und `ValidateScript` lässt sich mit `ErrorMessage` eine eigene Fehlermeldung hinterlegen, in der `{0}` für den eingegebenen Wert steht.

```powershell
function Test-Val {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidateSet('Dev', 'Test', 'Prod')]
        [string] $Umgebung,

        [ValidateRange(1, 65535)]
        [int] $Port = 443,

        [ValidatePattern('^srv-\w+\d{2}$', ErrorMessage = '"{0}" ist kein gültiger Servername (Beispiel: srv-web01).')]
        [string] $Server = 'srv-web01',

        [ValidateScript({ Test-Path -LiteralPath $_ -PathType Container },
                        ErrorMessage = 'Ordner "{0}" existiert nicht.')]
        [string] $Ordner = '.',

        [ValidateLength(2, 8)] [string] $Kurz = 'ab',
        [ValidateNotNullOrEmpty()] [string[]] $Liste = 'x',
        [ValidateCount(1, 3)] [string[]] $Max3 = 'a',
        [Alias('P')] [switch] $Plan
    )
    "$Umgebung $Port $Server $Plan"
}

Test-Val -Umgebung Dev -P
Test-Val -Umgebung Staging
Test-Val -Umgebung Dev -Port 70000
Test-Val -Umgebung Dev -Server 'web1'
```

Die Funktion zeigt die wichtigsten Attribute an einem Ort. Der erste Aufruf läuft durch und gibt `Dev 443 srv-web01 True` aus; der Alias `-P` schaltet den Schalter `-Plan` ein. Die folgenden Aufrufe scheitern *vor* der Ausführung, und die Fehlermeldungen sind aussagekräftig. Im Test lauteten sie unter anderem: „The argument "Staging" does not belong to the set "Dev,Test,Prod" specified by the ValidateSet attribute“, „The 70000 argument is greater than the maximum allowed range of 65535“ und, mit eigenem Text, „"web1" ist kein gültiger Servername (Beispiel: srv-web01).“ Die anderen Attribute verhalten sich analog: zu kurze Texte, leere Werte und zu viele Listenelemente führen zu Meldungen, die den Parameter und die verletzte Regel nennen. Beachten Sie, dass `ValidateSet` Groß-/Kleinschreibung ignoriert: Der Aufruf mit `'dev'` lief durch und gab den Wert unverändert (`dev`) weiter. Wer die Schreibweise normalisieren will, tut das im Funktionsrumpf.

Besonders nützlich sind *Parametersätze*. Sie beschreiben Gruppen von Parametern, die sich gegenseitig ausschließen, etwa „Name“ oder „Id“. PowerShell wählt anhand der übergebenen Parameter den passenden Satz und meldet einen Fehler, wenn die Kombination nicht eindeutig ist. In der Funktion steht der gewählte Satz in `$PSCmdlet.ParameterSetName`.

```powershell
function Test-Sets {
    [CmdletBinding(DefaultParameterSetName = 'ByName')]
    param(
        [Parameter(ParameterSetName = 'ByName', Mandatory)] [string] $Name,
        [Parameter(ParameterSetName = 'ById', Mandatory)]   [int] $Id
    )
    $PSCmdlet.ParameterSetName
}
Test-Sets -Name x
Test-Sets -Id 1
Test-Sets -Name x -Id 1
```

Die ersten beiden Aufrufe liefern `ByName` und `ById`. Der dritte kombiniert Parameter beider Sätze und scheitert mit „Parameter set cannot be resolved using the specified named parameters.“ (Lerneinheit 2.7).

> **Merke**
>
> Validierung gehört an den Eingang: `Mandatory`, `ValidateSet`, `ValidateRange`, `ValidatePattern`, `ValidateScript`, `ValidateNotNullOrEmpty`. Eigene Fehlermeldungen mit `ErrorMessage`. Parametersätze trennen sich ausschließende Eingaben. Ein Pflichtparameter ohne Wert blockiert unbeaufsichtigte Läufe, wenn nicht mit `-NonInteractive` gestartet wird.

```powershell
# FALSCH: Prüfung im Rumpf, spät und mit unklarer Meldung
param([string] $Umgebung)
if ($Umgebung -notin 'Dev', 'Test', 'Prod') { throw 'Fehler' }

# RICHTIG: Prüfung am Parameter, Tab-Vervollständigung inklusive
param(
    [Parameter(Mandatory)]
    [ValidateSet('Dev', 'Test', 'Prod')]
    [string] $Umgebung
)
```

Im ersten Fall ist die Fehlermeldung nichtssagend und die Prüfung läuft erst nach anderen Anweisungen. Im zweiten kennt PowerShell die erlaubten Werte, bietet sie per Tab-Taste an und meldet falsche Eingaben mit einer klaren Meldung. Wichtig ist auch die *Sicherheit*: Parameter, die in Dateipfade, Befehle oder Abfragen einfließen, brauchen strenge Prüfungen (Modul 8), weil sonst manipulierte Eingaben den Ablauf verändern können.

> **Praxisbeispiel**
>
> Ein Deployment-Skript akzeptiert die Umgebung nur aus einer festen Liste, den Servernamen nur nach einem Namensmuster und die Anzahl paralleler Läufe nur zwischen 1 und 10. Fehlerhafte Aufrufe in der Pipeline scheitern sofort mit einer verständlichen Meldung, statt erst nach Minuten bei einem Cloud-Aufruf mit einem kryptischen Fehler.

## 4.8 Splatting

Manche Befehle haben viele Parameter. Eine Zeile mit zehn Parametern passt nicht auf den Bildschirm, bricht an unvorhersehbaren Stellen um und lässt sich schlecht lesen. *Splatting* löst das Problem: Man sammelt die Parameter vorab in einer Hashtable (oder einem Array) und übergibt diese in einem Schritt. Dazu schreibt man statt des Dollarzeichens ein At-Zeichen vor den Variablennamen: `Get-ChildItem @p`. PowerShell entpackt die Hashtable und behandelt jeden Schlüssel als benannten Parameter mit dem zugehörigen Wert. Bei einem Array werden die Werte positional gebunden.

```powershell
$dir = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'splat-demo'
New-Item -Path (Join-Path $dir 'sub') -ItemType Directory -Force | Out-Null
'x' | Set-Content -LiteralPath (Join-Path $dir 'a.txt')
'x' | Set-Content -LiteralPath (Join-Path $dir 'b.md')
'x' | Set-Content -LiteralPath (Join-Path $dir 'sub/c.txt')

$p = @{ Path = $dir; File = $true; Recurse = $true }
(Get-ChildItem @p).Name -join ','

$filter = @{ Filter = '*.txt' }
(Get-ChildItem @p @filter).Name -join ','
(Get-ChildItem @p -Filter '*.md').Name -join ','

$ziel = Join-Path -Path $dir -ChildPath 'kopie.txt'
$pos = (Join-Path -Path $dir -ChildPath 'a.txt'), $ziel
Copy-Item @pos
Test-Path -LiteralPath $ziel

Remove-Item -LiteralPath $dir -Recurse
```

Die Hashtable `$p` enthält drei Parameter; `Get-ChildItem @p` findet im Test alle drei Dateien `a.txt`, `b.md` und `c.txt`. Mehrere Hashtables lassen sich im selben Aufruf kombinieren (`@p @filter`); der Test lieferte dann nur die `.txt`-Dateien `a.txt` und `c.txt`. Zusätzliche Parameter dürfen auch ausdrücklich neben dem Splat stehen (`-Filter '*.md'` lieferte `b.md`). Das Array `$pos` wird positional gebunden: Der erste Wert geht an den ersten Positionsparameter von `Copy-Item` (den Quellpfad), der zweite an den zweiten (das Ziel). Der Test bestätigte, dass die Kopie entstand.

> **Ergänzung aus der Nachrecherche**
>
> **Rangfolge bei doppelten Parametern (Fachwissen, nicht live abgerufen; siehe Quellen).** Wird ein Parameter sowohl im Splat als auch ausdrücklich genannt, hat in PowerShell 7 der ausdrücklich genannte Vorrang. Die Dokumentation beschreibt die zugrunde liegende Änderung so, dass Parameter aus dem Splatting hinter alle ausdrücklich genannten benannten Parameter verschoben werden. Im Test lieferte `Get-ChildItem @p -Path $dir`, obwohl `Path` auch im Splat steht, ohne Fehlermeldung das erwartete Ergebnis. Splatting funktioniert auch bei externen Programmen, die die Syntax `/Parameter:Wert` verwenden, etwa `$robo = @{R=1;W=1;MT=8}` und dann `robocopy quelle ziel @robo`.

Eine der wichtigsten Anwendungen ist der Umgang mit *optionalen Parametern*. Statt den Befehl in mehreren `if`-Zweigen mit verschiedenen Parameterkombinationen aufzurufen, baut man die Hashtable schrittweise auf und ruft den Befehl nur einmal auf. Das spart Code und vermeidet Fehler durch Kopieren.

```powershell
function Get-Dateien {
    [CmdletBinding()]
    param(
        [string] $Pfad = '.',
        [switch] $Rekursiv,
        [string] $Filter
    )
    $p = @{ Path = $Pfad; File = $true }
    if ($Rekursiv) { $p.Recurse = $true }
    if ($Filter)   { $p.Filter = $Filter }
    Get-ChildItem @p
}

function Get-Wrapper {
    [CmdletBinding()]
    param([string] $Path, [switch] $Recurse, [string] $Filter)
    Get-ChildItem @PSBoundParameters
}
```

`Get-Dateien` erzeugt die Parameter nach Bedarf: Die Pflichtangaben stehen von Anfang an in der Hashtable, optionale Einträge kommen nur dazu, wenn der Aufrufer sie angegeben hat. `Get-Wrapper` zeigt eine zweite, sehr elegante Technik: `$PSBoundParameters` (Lerneinheit 4.6) enthält genau die Parameter, die der Aufrufer übergeben hat, und lässt sich direkt an den inneren Befehl weiterreichen. Im Test lieferten `Get-Dateien -Pfad $dir` die Dateien der obersten Ebene und `Get-Wrapper -Path $dir -Recurse -Filter '*.txt'` alle `.txt`-Dateien einschließlich der Unterordner. Bei einem Schalter in der Hashtable gilt: `@{ Recurse = $true }` schaltet ein, `@{ Recurse = $false }` schaltet ausdrücklich aus.

> **Merke**
>
> Splatting: Parameter in einer Hashtable (benannt) oder einem Array (positional) sammeln und mit `@name` statt `$name` übergeben. Mehrere Splats lassen sich kombinieren. Optionale Parameter schrittweise in die Hashtable legen. `@PSBoundParameters` reicht die Aufruferparameter weiter.

```powershell
# FALSCH: Befehl mehrfach mit Parameterkombinationen
if ($Rekursiv -and $Filter) { Get-ChildItem -Path $Pfad -Recurse -Filter $Filter }
elseif ($Rekursiv)           { Get-ChildItem -Path $Pfad -Recurse }
elseif ($Filter)             { Get-ChildItem -Path $Pfad -Filter $Filter }
else                         { Get-ChildItem -Path $Pfad }

# RICHTIG: ein Aufruf, Parameter schrittweise gesammelt
$p = @{ Path = $Pfad }
if ($Rekursiv) { $p.Recurse = $true }
if ($Filter)   { $p.Filter = $Filter }
Get-ChildItem @p
```

Die erste Fassung wächst mit jedem weiteren Parameter exponentiell, die zweite linear.

> **Praxisbeispiel**
>
> Cloud-Cmdlets wie `New-AzVM` oder `New-MgUser` haben zehn oder mehr Parameter. In Skripten stehen sie fast immer als Splat-Hashtable, die sich zeilenweise lesen, kommentieren und in einer Code-Überprüfung nachvollziehen lässt. Für mehrere ähnliche Aufrufe teilt man eine gemeinsame Hashtable mit allgemeinen Werten (Ressourcengruppe, Standort, Tags) mit mehreren Befehlen.

## 4.9 Konfigurationsdateien

Ein Skript, das Servernamen, Pfade, Schwellwerte oder Adressen fest im Code trägt, muss bei jeder Änderung bearbeitet und neu geprüft werden und lässt sich nicht für zwei Umgebungen gleichzeitig nutzen. Die Lösung ist, *Konfiguration vom Code zu trennen*: Das Skript enthält die Logik, eine getrennte Datei enthält die Werte, die sich von Umgebung zu Umgebung unterscheiden (Entwicklung, Test, Produktion). Dasselbe Skript läuft dann überall, nur die Konfiguration wechselt. Das ist eine Grundregel der Automation und gilt für Skripte ebenso wie für größere Anwendungen.

Es gibt drei verbreitete Formate. *JSON* ist weit verbreitet, gut lesbar, kennt Zahlen, Wahrheitswerte, Listen und Verschachtelung und wird mit `ConvertFrom-Json` gelesen. *PSD1* (PowerShell Data File) ist eine Hashtable in PowerShell-Syntax, die mit `Import-PowerShellDataFile` sicher eingelesen wird; auch die Moduldeskriptoren (Lerneinheit 2.4) sind PSD1-Dateien. *ENV-Dateien* bestehen aus Zeilen `NAME=Wert` und sind vor allem in Container-Umgebungen üblich. Dazu kommen Umgebungsvariablen, die sich besonders gut für Werte eignen, die ein Orchestrierungssystem oder eine Pipeline setzt.

```powershell
$json = @'
{
  "Umgebung": "Dev",
  "Server": { "Name": "srv-web01", "Port": 8080 },
  "Features": ["Log", "Mail"],
  "AufbewahrungTage": 30
}
'@
$k = $json | ConvertFrom-Json
$k.Server.Port
$k.Features -join ','
$k.AufbewahrungTage.GetType().Name

$h = $json | ConvertFrom-Json -AsHashtable
$h.GetType().Name
$h['Server']['Name']

$schema = '{ "type": "object", "required": ["Umgebung", "Server"], "properties": { "Umgebung": { "enum": ["Dev", "Test", "Prod"] }, "AufbewahrungTage": { "type": "integer", "minimum": 1 } } }'
Test-Json -Json $json -Schema $schema
Test-Json -Json '{ "Umgebung": "Staging", "Server": {} }' -Schema $schema -ErrorAction SilentlyContinue
```

Die JSON-Konfiguration wird in ein Objekt umgewandelt, durch dessen Eigenschaften man mit Punkten navigiert (`$k.Server.Port` ergibt `8080`). Die Listen werden zu Arrays, die Zahlen zu `Int64`. Mit `-AsHashtable` entsteht eine (geordnete) Hashtable, im Test vom Typ `OrderedHashtable`, die sich flexibler zusammenführen lässt. Mit `Test-Json -Schema` prüft man, ob die Datei einem *JSON-Schema* entspricht, einer Beschreibung der erlaubten Struktur. Die gültige Konfiguration ergab `True`; die fehlerhafte (Umgebung „Staging“) `False`, mit der Meldung „The JSON is not valid with the schema: Value should match one of the values specified by the enum at ’/Umgebung’.“ Eine solche Prüfung direkt nach dem Einlesen verhindert, dass ein Tippfehler in der Konfiguration erst mitten im Lauf auffällt.

```powershell
$psd = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'konfig.psd1'
@'
@{
    Umgebung = 'Dev'
    Server   = @{ Name = 'srv-web01'; Port = 8080 }
    Features = @('Log', 'Mail')
}
'@ | Set-Content -Path $psd -Encoding utf8

$c = Import-PowerShellDataFile -Path $psd
$c.GetType().Name
$c.Server.Port

'@{ Zeit = Get-Date }' | Set-Content -Path $psd -Encoding utf8
try { Import-PowerShellDataFile -Path $psd } catch { $_.Exception.Message }
Remove-Item -Path $psd
```

`Import-PowerShellDataFile` liefert eine echte `Hashtable` (im Test `Hashtable`, Port 8080). Der entscheidende Vorteil ist die *Sicherheit*: Der Befehl führt keinen Code aus, sondern akzeptiert nur Daten. Enthält die Datei einen Befehlsaufruf oder eine Variable, bricht er ab; im Test mit „Cannot generate a PowerShell object for a ScriptBlock evaluating dynamic expressions. Dynamic expression: @{ Zeit = Get-Date }.“ (und ebenso für `$env:HOME`). Würde man dieselbe Datei mit `. ./konfig.psd1` oder `Invoke-Expression` laden, würde der Code ausgeführt, was bei Dateien aus fremder Quelle ein ernstes Risiko ist. Konfigurationsdateien sind Eingaben und müssen wie Eingaben behandelt werden (Modul 8).

Reale Skripte kombinieren mehrere Quellen. Eine bewährte Rangfolge lautet: *Standardwerte im Code* werden von der *Konfigurationsdatei* überschrieben, diese von *Umgebungsvariablen*, und diese von ausdrücklich übergebenen *Parametern*. Das folgende Beispiel implementiert die ersten drei Stufen mit einer rekursiven Zusammenführung zweier Hashtables, damit auch verschachtelte Werte einzeln überschrieben werden können.

```powershell
function Merge-Konfiguration {
    param([hashtable] $Basis, [hashtable] $Ueberschreibung)
    $ergebnis = $Basis.Clone()
    foreach ($schluessel in $Ueberschreibung.Keys) {
        if ($ergebnis[$schluessel] -is [hashtable] -and $Ueberschreibung[$schluessel] -is [hashtable]) {
            $ergebnis[$schluessel] = Merge-Konfiguration -Basis $ergebnis[$schluessel] -Ueberschreibung $Ueberschreibung[$schluessel]
        } else {
            $ergebnis[$schluessel] = $Ueberschreibung[$schluessel]
        }
    }
    $ergebnis
}

function Get-Konfiguration {
    [CmdletBinding()]
    param([string] $Pfad)
    $konfig = @{ Umgebung = 'Dev'; Server = @{ Name = 'localhost'; Port = 8080 }; AufbewahrungTage = 30 }
    if ($Pfad -and (Test-Path -LiteralPath $Pfad)) {
        $datei = Get-Content -LiteralPath $Pfad -Raw | ConvertFrom-Json -AsHashtable
        $konfig = Merge-Konfiguration -Basis $konfig -Ueberschreibung $datei
    }
    $umgebung = @{}
    if ($env:APP_UMGEBUNG) { $umgebung.Umgebung = $env:APP_UMGEBUNG }
    if ($env:APP_PORT)     { $umgebung.Server = @{ Port = [int]$env:APP_PORT } }
    Merge-Konfiguration -Basis $konfig -Ueberschreibung $umgebung
}

$jp = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'konfig.json'
'{ "Server": { "Name": "srv-web01" }, "AufbewahrungTage": 90 }' | Set-Content -Path $jp

$k1 = Get-Konfiguration -Pfad $jp
$k1.Server.Name; $k1.Server.Port; $k1.AufbewahrungTage; $k1.Umgebung

$env:APP_PORT = '9090'
$env:APP_UMGEBUNG = 'Prod'
$k2 = Get-Konfiguration -Pfad $jp
$k2.Server.Port; $k2.Server.Name; $k2.Umgebung

$env:APP_PORT = $null
$env:APP_UMGEBUNG = $null
Remove-Item -Path $jp
```

`Merge-Konfiguration` kopiert die Basis (flach, mit `.Clone()`) und überschreibt dann Schlüssel für Schlüssel; sind beide Werte selbst Hashtables, ruft sich die Funktion rekursiv auf, sodass nur der Teil überschrieben wird, der in der Überschreibung vorkommt. `Get-Konfiguration` beginnt mit den Standardwerten, mischt die Konfigurationsdatei ein, falls sie existiert, und überschreibt danach mit den Umgebungsvariablen. Im Test ergab der erste Lauf (nur Datei): Servername `srv-web01` aus der Datei, Port `8080` und Umgebung `Dev` aus den Standardwerten, Aufbewahrung 90 Tage aus der Datei. Der zweite Lauf mit gesetzten Umgebungsvariablen lieferte Port 9090 (als `Int32`), Umgebung `Prod` und den Servernamen unverändert aus der Datei. Dass die Funktion den Port mit `[int]` umwandelt, ist wichtig: Umgebungsvariablen sind immer Text. Ein ENV-Format lässt sich mit wenigen Zeilen einlesen, zum Beispiel ``$text -split "`n" | Where-Object { $_ -match '^\s*[^#\s][^=]*=' }`` und anschließend `$name, $wert = $_ -split '=', 2`.

> **Merke**
>
> Code und Konfiguration trennen. JSON für flexible Strukturen, PSD1 mit `Import-PowerShellDataFile` für sichere Hashtables, Umgebungsvariablen für Werte aus Pipeline und Container. Rangfolge: Standardwert, Datei, Umgebung, Parameter. Konfiguration validieren (`Test-Json -Schema`). *Keine Geheimnisse in Konfigurationsdateien.*

```powershell
# FALSCH: Konfigurationsdatei ausführen und Passwort in der Datei
. ./konfig.ps1                       # führt beliebigen Code aus
$passwort = 'GeheimesPasswort123'    # im Repository sichtbar

# RICHTIG: Daten laden, Geheimnisse aus einem Tresor beziehen
$konfig = Import-PowerShellDataFile -Path (Join-Path $PSScriptRoot 'konfig.psd1')
$passwort = Get-Secret -Name 'ServiceKonto'    # Modul SecretManagement (Modul 8)
```

Im ersten Fall führt das Laden beliebigen Code aus, und das Kennwort liegt im Klartext in der Versionsverwaltung, wo es dauerhaft bleibt, auch wenn man es später löscht. Im zweiten Fall enthält die Datei nur Daten, und das Geheimnis kommt aus einem dafür vorgesehenen Tresor (Modul 8).

> **Praxisbeispiel**
>
> Dasselbe Backup-Skript läuft in Entwicklung, Test und Produktion. Jede Umgebung hat ihre eigene Konfigurationsdatei im Repository (ohne Geheimnisse), die Pipeline setzt den Pfad per Umgebungsvariable, und ein Schema prüft die Datei vor jedem Lauf. Eine Änderung der Aufbewahrungsdauer ist eine Textänderung in einer Datei und braucht weder einen Eingriff in den Code noch eine neue Version des Skripts.

## 4.10 Übungsaufgaben zu Modul 04

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Was ergeben `'5' + 3`, `3 + '5'` und `[int](7 / 2)`? Begründen Sie jeweils das Ergebnis.

**Aufgabe 2.** Nennen Sie fünf Werte, die in einer `if`-Bedingung als falsch gelten, und zwei überraschende Werte, die als wahr gelten.

**Aufgabe 3.** Warum ist es keine gute Idee, ein Array in einer Schleife von 50 000 Durchläufen mit `+=` aufzubauen? Welche Alternative gibt es?

**Aufgabe 4.** Was passiert bei `1..3 | ForEach-Object { if ($_ -eq 2) { break } }` und was beim Ersatz von `break` durch `return`?

**Aufgabe 5.** Schreiben Sie eine Funktion `Get-Speicherstatus`, die aus einer Zahl für den freien Speicherplatz in Prozent (0 bis 100) mit `switch` die Meldung „kritisch“ (unter 5), „Warnung“ (unter 15) oder „in Ordnung“ liefert. Der Parameter muss ein Pflichtparameter sein und den Bereich 0 bis 100 validieren.

**Aufgabe 6.** Schreiben Sie eine Funktion `Get-Dateien`, die Dateien eines Ordners zurückgibt. Parameter: `Pfad` (Pflicht, muss ein existierender Ordner sein), `Filter` (optional) und der Schalter `Rekursiv`. Verwenden Sie Splatting, ohne `if`-Verzweigungen mit mehreren Aufrufen.

**Aufgabe 7.** Schreiben Sie eine Funktion `Invoke-MitWiederholung` mit Parametern `Aktion` (Skriptblock), `MaxVersuche` (1 bis 10, Standard 3) und `WartenMs` (Standard 200), die bei Fehlern wiederholt und die Wartezeit verdoppelt. Testen Sie sie mit einem Block, der erst beim dritten Aufruf gelingt.

**Aufgabe 8.** Schreiben Sie ein Skript `Pruefe-Server.ps1` mit Konfiguration aus einer JSON-Datei: Die Datei enthält eine Liste von Servern mit Namen und Port und einen Schwellwert für die Antwortzeit. Das Skript liest die Datei (mit `$PSScriptRoot`), validiert sie mit einem JSON-Schema, prüft für jeden Server mit `Test-Connection` oder `Test-NetConnection`, ob er antwortet, und gibt für jeden ein Objekt mit `Name`, `Port`, `Erreichbar` und `Status` aus. Das Skript endet mit Exit Code 1, wenn ein Server nicht erreichbar ist.

## 4.11 Lösungen zu Modul 04

### Aufgabe 1.

`'5' + 3` ergibt `53`, weil der linke Operand ein Text ist und `+` dann Text verkettet; die 3 wird in Text umgewandelt. `3 + '5'` ergibt `8`, weil der linke Operand eine Zahl ist und der Text in eine Zahl umgewandelt wird. `[int](7 / 2)` ergibt `4`: Die Division liefert 3,5, und die Umwandlung in `[int]` rundet auf die nächste gerade Zahl (Bankers Rounding), das ist 4. (Bei 2,5 wäre das Ergebnis 2.)

### Aufgabe 2.

Falsch: `$false`, `0`, der leere Text `''`, `$null`, ein leeres Array `@()` (auch `@(0)` und `0.0`). Überraschend wahr: der Text `'0'` (nicht leer), der Text `'false'`, ein einzelnes Leerzeichen, eine leere Hashtable `@{}`.

### Aufgabe 3.

Ein Array hat feste Größe; `+=` erzeugt bei jedem Durchlauf ein neues Array und kopiert alle bisherigen Elemente. Der Aufwand wächst mit dem Quadrat der Elementzahl (im Test: 20 000 Elemente in etwa 1,5 Sekunden gegenüber 0,04 Sekunden mit einer Liste). Alternativen: `[System.Collections.Generic.List[int]]` mit `.Add()`, oder die Ergebnisse der Schleife direkt zuweisen: `$x = foreach (...) { ... }`.

### Aufgabe 4.

Mit `break` beendet PowerShell nicht nur die Pipeline: `break` wirkt auf die nächste umgebende Schleife oder, wenn es keine gibt, auf das gesamte Skript. Im Test wurde nach dem Element 1 abgebrochen und nichts mehr ausgeführt. Mit `return` wird nur der aktuelle Durchlauf verlassen (wie `continue`); alle Elemente werden verarbeitet.

### Aufgabe 5.

```powershell
function Get-Speicherstatus {
    [CmdletBinding()]
    [OutputType([string])]
    param(
        [Parameter(Mandatory)]
        [ValidateRange(0, 100)]
        [double] $FreiProzent
    )
    switch ($FreiProzent) {
        { $_ -lt 5 }  { 'kritisch'; break }
        { $_ -lt 15 } { 'Warnung'; break }
        default       { 'in Ordnung' }
    }
}
Get-Speicherstatus -FreiProzent 3
Get-Speicherstatus -FreiProzent 12
Get-Speicherstatus -FreiProzent 60
```

Die Fälle prüfen die engste Bedingung zuerst, und `break` verhindert, dass bei einem Wert unter 5 auch der zweite Zweig läuft (sonst käme bei 3 auch „Warnung“). `ValidateRange` lehnt Werte außerhalb von 0 bis 100 vor der Ausführung ab. Erwartete Ausgabe: „kritisch“, „Warnung“, „in Ordnung“.

### Aufgabe 6.

```powershell
function Get-Dateien {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidateScript({ Test-Path -LiteralPath $_ -PathType Container },
                        ErrorMessage = 'Ordner "{0}" existiert nicht.')]
        [string] $Pfad,
        [string] $Filter,
        [switch] $Rekursiv
    )
    $p = @{ LiteralPath = $Pfad; File = $true }
    if ($Rekursiv) { $p.Recurse = $true }
    if ($Filter)   { $p.Filter = $Filter }
    Get-ChildItem @p
}
Get-Dateien -Pfad $HOME -Filter '*.txt'
```

Die Parameter werden schrittweise in die Hashtable gelegt, und `Get-ChildItem` wird nur einmal aufgerufen. `-LiteralPath` verhindert Probleme mit Sonderzeichen im Pfad. Bei einem nicht existierenden Ordner meldet `ValidateScript` die eigene Fehlermeldung, bevor etwas ausgeführt wird.

### Aufgabe 7.

```powershell
function Invoke-MitWiederholung {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [scriptblock] $Aktion,
        [ValidateRange(1, 10)] [int] $MaxVersuche = 3,
        [int] $WartenMs = 200
    )
    for ($versuch = 1; $versuch -le $MaxVersuche; $versuch++) {
        try {
            return & $Aktion
        }
        catch {
            Write-Warning "Versuch $versuch von $MaxVersuche fehlgeschlagen: $($_.Exception.Message)"
            if ($versuch -eq $MaxVersuche) { throw }
            Start-Sleep -Milliseconds ($WartenMs * [math]::Pow(2, $versuch - 1))
        }
    }
}

$zaehler = 0
Invoke-MitWiederholung -Aktion {
    $script:zaehler++
    if ($script:zaehler -lt 3) { throw "Fehler $($script:zaehler)" }
    "Erfolg im Versuch $($script:zaehler)"
}
```

Die ersten beiden Versuche erzeugen Warnungen, der dritte liefert „Erfolg im Versuch 3“. Das Warten von 200 ms und 400 ms entspricht dem verdoppelten Abstand. Beim letzten Versuch wird der Fehler mit `throw` weitergegeben, damit der Aufrufer ihn behandeln kann.

### Aufgabe 8.

```powershell
#Requires -Version 7.0
[CmdletBinding()]
param(
    [string] $KonfigPfad = (Join-Path -Path $PSScriptRoot -ChildPath 'server.json')
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$schema = @'
{
  "type": "object",
  "required": ["Server", "SchwellwertMs"],
  "properties": {
    "SchwellwertMs": { "type": "integer", "minimum": 1 },
    "Server": {
      "type": "array", "minItems": 1,
      "items": { "type": "object", "required": ["Name", "Port"],
                 "properties": { "Name": { "type": "string" }, "Port": { "type": "integer", "minimum": 1, "maximum": 65535 } } }
    }
  }
}
'@

try {
    $text = Get-Content -LiteralPath $KonfigPfad -Raw
    if (-not (Test-Json -Json $text -Schema $schema -ErrorAction SilentlyContinue)) {
        throw "Konfiguration ungültig: $KonfigPfad"
    }
    $konfig = $text | ConvertFrom-Json

    $ergebnisse = foreach ($s in $konfig.Server) {
        $ok = Test-Connection -TargetName $s.Name -TcpPort $s.Port -TimeoutSeconds 2 -Quiet
        [pscustomobject]@{
            Name      = $s.Name
            Port      = $s.Port
            Erreichbar = [bool]$ok
            Status    = if ($ok) { 'OK' } else { 'FEHLER' }
        }
    }
    $ergebnisse
    if ($ergebnisse | Where-Object { -not $_.Erreichbar }) { exit 1 }
    exit 0
}
catch {
    Write-Error $_
    exit 2
}
```

Zur Prüfung dient eine Datei `server.json` neben dem Skript, etwa mit `"SchwellwertMs": 500` und einer Liste `"Server": [ { "Name": "localhost", "Port": 22 } ]`. Das Skript validiert die Konfiguration gegen das Schema, bevor es etwas ausführt, und beendet sich mit Code 2 bei Konfigurations- und Laufzeitfehlern, mit Code 1 bei nicht erreichbaren Servern und mit Code 0 bei Erfolg. Der Parameter `-TcpPort` und `-TargetName` von `Test-Connection` stehen in PowerShell 7 zur Verfügung; unter Windows PowerShell 5.1 verwendet man dafür `Test-NetConnection`. Der Schwellwert für die Antwortzeit ist als Konfigurationswert vorgesehen; eine Messung der Antwortzeit lässt sich mit `Measure-Command` um den Verbindungstest ergänzen. Der Beispieltext enthält einen Rückwärtsschrägstrich nicht, daher funktionieren Pfade plattformübergreifend durch `Join-Path`.
