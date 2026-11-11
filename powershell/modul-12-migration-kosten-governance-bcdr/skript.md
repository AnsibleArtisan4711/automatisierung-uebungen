# Modul 12 - Migration, Kosten, Governance und BCDR

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (12.1 bis 12.9; 12.10 Übungsaufgaben, 12.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                             |
|:--------------------------------------|:----------------------------|
| Dauer laut Übersichtsblatt            | 10 h                        |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h) |
| Priorität                             | 3                           |

> **Merke**
>
> **Stundenabweichung im Lehrplan.** Das Übersichtsblatt nennt 10 h, die neun Lerneinheiten des Detailblatts ergeben zusammen nur 9 h. Dieses Skript folgt dem Detailblatt, weil nur dort die einzelnen Lerneinheiten stehen; die fehlende Stunde lässt sich als Puffer für die Übungsaufgaben oder die Besprechung der Lösungen einplanen.

### Lernziele.

Dieses Modul schließt den Kurs ab und betrachtet PowerShell-Automatisierung als dauerhaften Betriebsgegenstand. Sie lernen, vorhandene Skripte systematisch zu inventarisieren, sie auf PowerShell 7 zu heben, Standards und Governance einzuführen, den Nutzen von Automatisierung nachvollziehbar zu rechnen, Abläufe zu skalieren, ohne Dienste zu überlasten, die Wiederherstellbarkeit der Automatisierung selbst abzusichern, Änderungen kontrolliert einzuführen, Skripte betriebsfähig zu übergeben und die Qualität kontinuierlich zu verbessern.

| **Nr.** | **Lerneinheit**                 | **Dauer** | **Prio** |
|:--------|:--------------------------------|:----------|:---------|
| 12.1    | Bestandsaufnahme                | 1 h       | 3        |
| 12.2    | Migration zu PowerShell 7       | 1 h       | 3        |
| 12.3    | Standardisierung und Governance | 1 h       | 3        |
| 12.4    | Kosten und Nutzen               | 1 h       | 3        |
| 12.5    | Skalierung der Automation       | 1 h       | 3        |
| 12.6    | BCDR für Automation             | 1 h       | 3        |
| 12.7    | Change Management               | 1 h       | 3        |
| 12.8    | Betriebsübergabe                | 1 h       | 3        |
| 12.9    | Kontinuierliche Verbesserung    | 1 h       | 3        |
|         | **Summe**                       | **9 h**   |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde.** Alle Skripte dieses Moduls, die ohne Cloud- oder Windows-Dienste auskommen, wurden mit PowerShell 7.6.6 unter Linux ausgeführt: das Skriptinventar auf Basis des Syntaxbaums, die Rentabilitätsrechnung, die Wiederholungslogik mit Wartezeiten nach `Retry-After`, die Prüfsummenliste für die Wiederherstellungsprobe, die Änderungsprüfung auf Version und Changelog, die Prüfung der Übergabe-Checkliste und die Auswertung von Protokolldateien nach Tagen. Die gezeigten Ausgaben stammen aus diesen Läufen. Nicht getestet wurden der Einsatz in einer Windows-Umgebung, Zugriffe auf Cloud-Dienste und die Dienste, die in den Beispielen als Ziel auftauchen (zum Beispiel eine echte Programmierschnittstelle, die Antwortcode 429 liefert); diese Teile sind mit einer simulierten Funktion nachgestellt. Die Hinweise zu Lebenszyklen und Fristen von Windows PowerShell 5.1 und PowerShell 7 stammen aus Fachwissen und wurden für dieses Modul nicht abgerufen; vor einer Entscheidung ist der aktuelle Stand in der Microsoft-Dokumentation zu prüfen.

## 12.1 Bestandsaufnahme

Bevor man Skripte verbessert, migriert oder übergibt, muss man wissen, was überhaupt da ist. In vielen Betrieben ist die Automatisierung über Jahre gewachsen: Skripte liegen auf Dateiservern, in Benutzerordnern, auf Servern im Aufgabenplaner, in alten Wikis und in den Köpfen einzelner Kollegen. Niemand kennt den Bestand genau, und genau das ist das Risiko. Die *Bestandsaufnahme* (englisch *Inventory*) ist deshalb der erste Schritt jedes Aufräumprojekts. Sie beantwortet drei Fragen: Was gibt es, was davon läuft tatsächlich und wer ist dafür verantwortlich?

Man unterscheidet zwei Quellen. Die *Dateisuche* findet Skriptdateien (`*.ps1`, `*.psm1`, `*.psd1`) in Ordnern und Repositories. Die *Laufzeitsuche* findet, was wirklich ausgeführt wird: geplante Aufgaben im Windows-Aufgabenplaner, Cron-Einträge, Pipeline-Definitionen, Runbooks in Azure Automation, Dienste, die PowerShell starten. Eine Datei, die niemand mehr aufruft, ist Altlast; ein Planeintrag, der auf eine nicht mehr vorhandene Datei zeigt, ist ein schleichender Ausfall. Erst der Vergleich beider Listen zeigt die Wahrheit.

Für die Dateisuche reicht `Get-ChildItem` allein nicht aus, weil man mehr als Namen braucht. Man möchte wissen, wie groß ein Skript ist, wie viele Funktionen es enthält, ob es Syntaxfehler hat, welche PowerShell-Version es verlangt, ob es veraltete Befehle benutzt und ob ein Kennwort im Klartext darin steht. Dazu liest man die Datei nicht als Text, sondern lässt PowerShell sie mit dem eigenen Parser in einen *abstrakten Syntaxbaum* (englisch *Abstract Syntax Tree*, AST) zerlegen. Der Syntaxbaum kennt die Struktur: Er unterscheidet einen Befehlsaufruf von einer Zeichenkette, die zufällig denselben Text enthält. Damit vermeidet man die typischen Fehlalarme einer reinen Textsuche, bei der ein Kommentar mit dem Wort „Get-WmiObject“ als Fund zählt.

```powershell
$script:Veraltet = @{
    'Get-WmiObject'    = 'Get-CimInstance'
    'Invoke-WmiMethod' = 'Invoke-CimMethod'
    'Get-EventLog'     = 'Get-WinEvent'
    'Write-EventLog'   = '.NET-Klasse System.Diagnostics.EventLog'
}

function Get-SkriptInventar {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad)

    Get-ChildItem -LiteralPath $Pfad -Filter *.ps1 -Recurse -File | ForEach-Object {
        $tokens = $null; $fehler = $null
        $ast = [System.Management.Automation.Language.Parser]::ParseFile(
                   $_.FullName, [ref]$tokens, [ref]$fehler)
        $aufrufe = $ast.FindAll({ param($n)
                       $n -is [System.Management.Automation.Language.CommandAst] }, $true) |
                   ForEach-Object { $_.GetCommandName() } |
                   Where-Object { $_ } | Sort-Object -Unique
        $funde  = @($aufrufe | Where-Object { $script:Veraltet.ContainsKey($_) })
        $inhalt = Get-Content -LiteralPath $_.FullName -Raw
        [pscustomobject]@{
            Datei            = $_.Name
            Zeilen           = ($inhalt -split "`n").Count
            Funktionen       = @($ast.FindAll({ param($n)
                                   $n -is [System.Management.Automation.Language.FunctionDefinitionAst] }, $true)).Count
            Syntaxfehler     = @($fehler).Count
            RequiresVersion  = if ($inhalt -match '(?im)^\s*#requires\s+-version\s+(\S+)') { $Matches[1] } else { '' }
            Veraltet         = ($funde | ForEach-Object { "$_ -> $($script:Veraltet[$_])" }) -join '; '
            Kennwortverdacht = [bool]($inhalt -match '(?i)(password|passwort|kennwort)\s*=\s*[''"][^''"]+[''"]')
            Aufrufe          = $aufrufe.Count
        }
    }
}

Get-SkriptInventar -Pfad ./skripte | Export-Csv inventar.csv -NoTypeInformation -Encoding utf8
```

Die Tabelle `$script:Veraltet` ordnet veralteten Befehlen den Nachfolger zu; sie lässt sich beliebig erweitern. Die Funktion durchsucht rekursiv alle `.ps1`-Dateien. Für jede Datei ruft sie `Parser::ParseFile` auf, das den Syntaxbaum und zwei Listen liefert: die Tokens und die Syntaxfehler. Mit `FindAll` sucht man im Baum alle Knoten eines Typs; hier sind es zuerst alle Befehlsaufrufe (`CommandAst`), deren Namen die Methode `GetCommandName()` liefert, und dann alle Funktionsdefinitionen. Der Aufruf `GetCommandName()` liefert für dynamische Aufrufe, etwa `& $programm`, keinen Namen; deshalb filtert `Where-Object { $_ }` leere Werte aus. Die Zeile mit `#requires` liest die verlangte Mindestversion aus. Der letzte reguläre Ausdruck sucht Zuweisungen der Form `password = 'text'`; er ist nur ein Hinweisgeber und ersetzt weder ein Werkzeug zur Geheimnissuche noch eine Durchsicht.

Der Testlauf mit drei Beispieldateien lieferte folgendes. Die erste Datei (mit `#requires -Version 5.1`, einem Aufruf von `Get-WmiObject` und `Get-EventLog`, einer Funktion und einer Kennwortzuweisung) ergab: 6 Zeilen, 1 Funktion, keine Syntaxfehler, `RequiresVersion` 5.1, `Veraltet` „Get-EventLog -\> Get-WinEvent; Get-WmiObject -\> Get-CimInstance“, `Kennwortverdacht` wahr. Die zweite Datei, mit absichtlichem Syntaxfehler, ergab drei Syntaxfehler und keine Aufrufe; die dritte, modern geschriebene Datei meldete weder veraltete Befehle noch einen Verdacht. Wichtig zur Einordnung: Die Datei mit dem Syntaxfehler zeigt, wie die Inventur auch beschädigte Skripte erkennt, ohne an ihnen zu scheitern, weil `ParseFile` die Fehler sammelt, statt eine Ausnahme zu werfen.

### Was gehört noch ins Inventar?

Zu jedem Skript gehören Angaben, die man nicht aus dem Code lesen kann: der Zweck in einem Satz, der Eigentümer (Person oder Team), der Ausführungsort (Server, Planer, Pipeline), das Dienstkonto, die Häufigkeit, die Abhängigkeiten (Module, Dienste, Freigaben, Zertifikate), die Kritikalität (was passiert bei Ausfall?) und das Datum des letzten erfolgreichen Laufs. Diese Felder pflegt man am besten in einer einfachen Tabelle oder besser als Datei im Repository neben dem Skript. Die technischen Spalten füllt die Funktion automatisch; die fachlichen ergänzen Menschen. Gerade die Kritikalität erlaubt später eine sinnvolle Reihenfolge: Zuerst werden die Skripte migriert, standardisiert und abgesichert, deren Ausfall wehtut.

> **Ergänzung aus der Nachrecherche**
>
> **Werkzeuge für die Bestandsaufnahme.** Neben dem selbst gebauten Inventar gibt es `Invoke-ScriptAnalyzer`, das Regelverstöße und Kompatibilitätsprobleme findet (siehe Modul 11, Lerneinheit 11.3), und die Auswertung des Aufgabenplaners mit `Get-ScheduledTask` unter Windows. Der Schwerpunkt dieser Lerneinheit liegt auf der Methode: Datei- und Laufzeitsicht verbinden, den Syntaxbaum statt Textsuche benutzen und fachliche Angaben ergänzen. Die genannten Cmdlets wurden hier nicht ausgeführt.

> **Merke**
>
> Die Bestandsaufnahme verbindet Dateisicht (was liegt herum?) und Laufzeitsicht (was läuft wirklich?). Technische Merkmale liest man mit dem Syntaxbaum aus (Größe, Funktionen, Syntaxfehler, veraltete Befehle), fachliche Merkmale (Zweck, Eigentümer, Kritikalität) müssen Menschen ergänzen. Das Ergebnis ist eine priorisierte Liste, nicht ein Berg Dateien.

```powershell
# FALSCH: Textsuche findet auch Kommentare und Zeichenketten
Select-String -Path *.ps1 -Pattern 'Get-WmiObject'

# RICHTIG: Syntaxbaum - nur echte Befehlsaufrufe zaehlen
$ast.FindAll({ param($n) $n -is [System.Management.Automation.Language.CommandAst] }, $true) |
    Where-Object { $_.GetCommandName() -eq 'Get-WmiObject' }
```

Die Textsuche meldet eine Datei auch dann, wenn der Befehl nur in einem Kommentar oder in einer Meldung steht. Der Syntaxbaum zählt ausschließlich tatsächliche Aufrufe.

> **Praxisbeispiel**
>
> Ein neues Team übernimmt eine Windows-Umgebung. In der ersten Woche entsteht ein Inventar mit 140 Skripten; 40 davon haben keinen Planeintrag mehr, 15 benutzen veraltete Befehle, 6 enthalten Kennwörter. Mit dieser Liste kann das Team Aufwand schätzen, Prioritäten setzen und den Kolleginnen und Kollegen konkret sagen, welche Skripte als Erstes angefasst werden.

## 12.2 Migration zu PowerShell 7

Auf vielen Windows-Servern ist *Windows PowerShell 5.1* vorinstalliert, das Programm `powershell.exe`. *PowerShell 7* (`pwsh.exe`) ist eine eigenständige Weiterentwicklung auf Basis von .NET, läuft auf Windows, Linux und macOS und wird parallel installiert. Beide Versionen existieren nebeneinander; eine Migration heißt deshalb nicht, etwas zu ersetzen, sondern Skripte so anzupassen, dass sie unter PowerShell 7 laufen, und die Aufrufer (Planer, Pipelines, Dienste) auf `pwsh` umzustellen. Gründe dafür sind plattformübergreifender Betrieb, bessere Leistung, neue Sprachmerkmale und die Tatsache, dass neue Funktionen nur noch in PowerShell 7 entstehen. Die Aussage, dass Windows PowerShell 5.1 nur noch Fehlerkorrekturen erhält, stammt aus Fachwissen und ist vor einer Entscheidung in der Microsoft-Dokumentation zu prüfen.

Die Migration verläuft am besten in vier Schritten. Zuerst steht die Bestandsaufnahme aus 12.1 mit der Liste veralteter Befehle. Dann folgt die *Kompatibilitätsprüfung*: Man lässt das Skript unter PowerShell 7 laufen und liest die Fehler. Dazu gibt es ein Regelwerk in `PSScriptAnalyzer` (Regeln `PSUseCompatibleCmdlets`, `PSUseCompatibleCommands` und `PSUseCompatibleSyntax`), die laut abgerufener Dokumentation zu den Kompatibilitätsregeln zählen; in dieser Umgebung war das Modul nicht installierbar, die Regeln wurden daher nicht ausgeführt. Drittens werden die Probleme behoben, und viertens läuft das Skript eine Zeit lang parallel (alt und neu) und wird verglichen, bevor man die alte Variante abschaltet.

Welche Unterschiede treten tatsächlich auf? Es gibt vier Gruppen. Erstens fehlende Befehle: Cmdlets wie `Get-WmiObject`, `Get-EventLog` und `Write-EventLog` gehören zu Windows PowerShell; im Linux-Test waren sie in PowerShell 7.6.6 nicht vorhanden, und für `Get-EventLog` und `Write-EventLog` bestätigt es die abgerufene Dokumentation (nur Windows PowerShell 5.1 und älter). Der Ersatz für die WMI-Cmdlets ist `Get-CimInstance`. Zweitens nicht kompatible Module: Manche Module, vor allem ältere für Windows-Verwaltung, laufen nur unter 5.1. Für sie gibt es die Kompatibilitätsschicht: `Import-Module Name -UseWindowsPowerShell` lädt das Modul in einem Hintergrundprozess von Windows PowerShell und stellt seine Befehle über eine Umleitung bereit. Der Parameter `UseWindowsPowerShell` ist im Test in PowerShell 7.6.6 vorhanden; die Funktionsweise unter Windows habe ich nicht ausgeführt, die Beschreibung stützt sich auf Fachwissen. Die über diese Schicht gelieferten Objekte sind *serialisiert*, also Abbilder ohne Methoden, was Folgeprobleme erzeugt.

Drittens geänderte Standardwerte. Ein wichtiger ist die Zeichenkodierung: In PowerShell 7 schreiben `Set-Content`, `Out-File` und andere Cmdlets standardmäßig UTF-8 *ohne* Byte-Order-Mark (BOM); Windows PowerShell 5.1 benutzte je nach Cmdlet andere Kodierungen (zum Beispiel UTF-16 oder die ANSI-Codepage). Im Test begann eine mit `Set-Content` geschriebene Datei mit den Bytes `C3 A4 C3 B6` (UTF-8 ohne BOM, Umlaute als je zwei Bytes), mit `-Encoding utf8BOM` dagegen mit `EF BB BF`. Das kann Empfänger überraschen, die ein BOM erwarten, etwa ältere Excel-Importe. Ein weiterer Standardwert ist die Standardtiefe von `ConvertTo-Json`, die in beiden Versionen 2 beträgt; im Test erschien beim Umwandeln eines vierfach verschachtelten Objekts die Warnung „Resulting JSON is truncated as serialization has exceeded the set depth of 2“, und die innerste Ebene kam als Text `"@{d=1}"` an. Mit `-Depth 5` wurde alles korrekt ausgegeben. Diese Falle ist keine Migrationsfolge, wird bei Migrationen aber oft entdeckt.

Viertens neue Möglichkeiten, die man nutzen, aber nicht voraussetzen sollte, solange noch 5.1 im Spiel ist: der Ternäroperator (`($a -gt 3) ? 'groß' : 'klein'`), der Null-Koaleszenz-Operator (`$x ?? 'Standard'`) und `ForEach-Object -Parallel`. Im Test lieferten diese Beispiele „Standard“, „gross“ und für `1..4 | ForEach-Object -Parallel { $_ * 2 } -ThrottleLimit 2` nach dem Sortieren 2, 4, 6, 8. Skripte, die solche Syntax benutzen, laufen nicht mehr unter 5.1; man macht das mit `#requires -Version 7.0` sichtbar, damit der Fehler früh und verständlich auftritt.

```powershell
#requires -Version 7.0
# Zeigt die drei Sprachmerkmale und die Kodierungs-Eigenheit
$x = $null
$x ?? 'Standard'                        # Null-Koaleszenz
$a = 5
($a -gt 3) ? 'gross' : 'klein'          # Ternaer
1..4 | ForEach-Object -Parallel { $_ * 2 } -ThrottleLimit 2 | Sort-Object

$f = Join-Path ([IO.Path]::GetTempPath()) 'enc.txt'
'äöü' | Set-Content $f                  # UTF-8 ohne BOM (PowerShell 7)
[IO.File]::ReadAllBytes($f)[0..3]       # 195 164 195 182
'äöü' | Set-Content $f -Encoding utf8BOM
[IO.File]::ReadAllBytes($f)[0..3]       # 239 187 191 195
```

Die Datei beginnt mit `#requires -Version 7.0`; unter 5.1 verweigert PowerShell den Start mit einer klaren Meldung, statt mitten im Skript an der unbekannten Syntax zu scheitern. Die Anweisungen zeigen die drei Neuerungen. Zum Schluss prüft das Beispiel die ersten Bytes der Datei: Die Zahlen 195 164 195 182 sind die Dezimalwerte von `C3 A4 C3 B6`, 239 187 191 sind `EF BB BF`, also das BOM.

> **Ergänzung aus der Nachrecherche**
>
> **Parallel nebeneinander statt Big Bang.** Praxiserprobt ist ein Stufenplan: Neue Skripte entstehen gleich für PowerShell 7. Bestehende werden nach Kritikalität migriert, mit einem Test auf einer Kopie der Umgebung. Planer und Pipelines rufen `pwsh` *explizit* auf, damit jederzeit klar ist, welche Version läuft. Schließlich bleibt Windows PowerShell 5.1 für die wenigen Skripte, die an inkompatible Module gebunden sind, bis diese ersetzt werden. Ein Skript, das in beiden Versionen laufen muss, nutzt den kleinsten gemeinsamen Nenner und wird in beiden getestet.

> **Merke**
>
> Windows PowerShell 5.1 und PowerShell 7 laufen nebeneinander. Typische Unterschiede: fehlende Windows-Cmdlets (`Get-WmiObject`, `Get-EventLog`), inkompatible Module (Ausweg: `-UseWindowsPowerShell`), Standardkodierung UTF-8 ohne BOM, neue Syntax (`?:`, `??`, `-Parallel`). `#requires -Version 7.0` schützt vor dem Start in der falschen Version.

```powershell
# FALSCH: Skript nutzt neue Syntax, ohne dies zu deklarieren, Planer ruft powershell.exe
$name = $eingabe ?? 'unbekannt'          # unter 5.1: Syntaxfehler mitten im Lauf

# RICHTIG: Voraussetzung deklarieren, Planer ruft pwsh explizit
#requires -Version 7.0
$name = $eingabe ?? 'unbekannt'
```

Ohne `#requires` scheitert das Skript unter 5.1 mit einer Syntaxmeldung, die auf die Zeile zeigt, aber nicht auf die eigentliche Ursache, nämlich die falsche Version.

> **Praxisbeispiel**
>
> Ein Unternehmen stellt 60 Aufgabenplan-Skripte auf PowerShell 7 um. Das Inventar aus 12.1 liefert die Liste der betroffenen Befehle, ein Testlauf unter `pwsh` deckt zwei Skripte mit alten Modulen und zehn mit Kodierungsproblemen in CSV-Dateien auf. Nach drei Wochen Parallelbetrieb mit Vergleich der Ergebnisse wird auf `pwsh` umgestellt.

## 12.3 Standardisierung und Governance

Wenn zehn Personen Skripte schreiben, entstehen zehn Stile: unterschiedliche Parameternamen, unterschiedliche Fehlerbehandlung, unterschiedliche Protokollformate. Jedes einzelne Skript mag in Ordnung sein, aber im Ganzen wird der Betrieb teuer, weil jede Übernahme eine neue Einarbeitung bedeutet. *Standardisierung* vereinheitlicht die Dinge, die sich lohnen zu vereinheitlichen. *Governance* (zu Deutsch etwa Steuerung oder Regelwerk) legt fest, wer was entscheidet, welche Regeln gelten und wie ihre Einhaltung geprüft wird.

Wichtig ist das richtige Maß. Ein Regelwerk mit 80 Seiten liest niemand, und Regeln, die in der Praxis nicht prüfbar sind, werden ignoriert. Bewährt hat sich ein kurzes Dokument mit wenigen Pflichtregeln (zum Beispiel: jedes Skript hat `[CmdletBinding()]`, einen Kommentarkopf mit Zweck und Eigentümer, nutzt `Set-StrictMode`, behandelt Fehler mit `try/catch`, schreibt Protokolle im Format aus Modul 9, enthält keine Geheimnisse im Code und liegt in der Versionsverwaltung) und einem Anhang mit Empfehlungen. Jede Pflichtregel sollte sich *automatisch* prüfen lassen, sonst bleibt sie ein Wunsch.

Die automatische Prüfung übernimmt `PSScriptAnalyzer`. Man legt eine Einstellungsdatei an, die festlegt, welche Regeln aktiv sind und wie schwer ihre Verstöße wiegen. Laut abgerufener Dokumentation wird eine Datei mit dem Namen `PSScriptAnalyzerSettings.psd1` im Projektstamm implizit gefunden; ansonsten übergibt man sie mit `-Settings`. Die Datei ist eine Hashtable mit Schlüsseln wie `IncludeRules`, `ExcludeRules`, `Severity` und `Rules`. Das Modul selbst war im Arbeitsbereich nicht installierbar; das folgende Beispiel wurde daher nur als Datenstruktur mit `Import-PowerShellDataFile` geprüft, nicht mit dem Analyzer ausgeführt.

```powershell
# PSScriptAnalyzerSettings.psd1
@{
    Severity     = @('Error', 'Warning')
    IncludeRules = @(
        'PSAvoidUsingCmdletAliases'
        'PSAvoidUsingPlainTextForPassword'
        'PSAvoidUsingConvertToSecureStringWithPlainText'
        'PSUseApprovedVerbs'
        'PSUseShouldProcessForStateChangingFunctions'
        'PSUseCompatibleSyntax'
    )
    ExcludeRules = @('PSAvoidUsingWriteHost')
    Rules        = @{
        PSUseCompatibleSyntax = @{
            Enable         = $true
            TargetVersions = @('7.0')
        }
    }
}
```

Die Datei aktiviert eine überschaubare Reihe von Regeln, schaltet die Regel gegen `Write-Host` ab (siehe Modul 9: Skripte für den Betrieb nutzen es bewusst nicht, interaktive Hilfswerkzeuge dürfen es), und konfiguriert die Syntaxkompatibilitätsregel für das Ziel PowerShell 7.0. Die Regelnamen stammen aus dem Fundus der Regeln des Analyzers; die Namen der Kompatibilitätsregeln bestätigt die abgerufene Dokumentation, die übrigen Namen stammen aus Fachwissen und sind vor der Verwendung mit `Get-ScriptAnalyzerRule` abzugleichen. Aufgerufen wird die Prüfung später in der Pipeline, siehe Modul 11, Lerneinheit 11.3.

Neben der Prüfung hilft eine *Vorlage*. Wer ein neues Skript beginnt, soll nicht bei null anfangen, sondern von einem Gerüst ausgehen, das die Standards schon enthält. Ein Generator erzeugt dieses Gerüst. So sind die Regeln der bequeme Weg und kein Zusatzaufwand.

```powershell
function New-SkriptGeruest {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [ValidatePattern('^[A-Z][a-z]+-[A-Za-z]+$')] [string] $Name,
        [Parameter(Mandatory)] [string] $Zweck,
        [Parameter(Mandatory)] [string] $Eigentuemer,
        [string] $Ordner = '.'
    )
    $datei = Join-Path $Ordner "$Name.ps1"
    if (Test-Path -LiteralPath $datei) { throw "Datei existiert bereits: $datei" }
    $inhalt = @"
<#
.SYNOPSIS
    $Zweck
.NOTES
    Eigentuemer: $Eigentuemer
    Erstellt:    $(Get-Date -Format 'yyyy-MM-dd')
#>
#requires -Version 7.0
[CmdletBinding(SupportsShouldProcess)]
param()

Set-StrictMode -Version Latest
`$ErrorActionPreference = 'Stop'
`$Codes = @{ Ok = 0; Warnung = 10; Fehler = 20; Unerwartet = 99 }

try {
    # TODO: Aufgabe hier umsetzen
    exit `$Codes.Ok
}
catch {
    Write-Error `$_
    exit `$Codes.Unerwartet
}
"@
    if ($PSCmdlet.ShouldProcess($datei, 'Skriptgeruest anlegen')) {
        Set-Content -LiteralPath $datei -Value $inhalt -Encoding utf8
        Get-Item -LiteralPath $datei
    }
}
```

Der Generator prüft zunächst den Namen mit `ValidatePattern`: Er muss der Form *Verb-Substantiv* entsprechen (großer Anfangsbuchstabe, Bindestrich), wie es PowerShell-Befehle tun. Der Gerüsttext steht in einem mehrzeiligen *Here-String* mit doppelten Anführungszeichen; darin werden Variablen ersetzt, deshalb müssen die Variablen, die im *erzeugten* Skript stehen sollen (`$ErrorActionPreference`, `$Codes`, `$_`), mit einem Gegenakzent (```$``) geschützt werden. Dagegen setzen `$Zweck`, `$Eigentuemer` und das Datum ihre Werte schon beim Erzeugen ein. Das erzeugte Skript enthält Kommentarkopf, Versionsanforderung, `Set-StrictMode`, abbrechende Fehlerbehandlung, die Code-Tabelle aus Modul 9 und ein `try/catch` mit Exit Codes. `SupportsShouldProcess` ermöglicht den Probelauf mit `-WhatIf`. Alle Skripte des Moduls wurden durch einen Syntaxtest mit dem PowerShell-Parser geprüft; der erzeugte Gerüsttext ist damit ebenfalls syntaktisch korrekt.

> **Ergänzung aus der Nachrecherche**
>
> **Governance ist mehr als Technik.** Zu einem tragfähigen Regelwerk gehören drei organisatorische Klärungen, die kein Werkzeug ersetzt: Wer darf Skripte in Produktion bringen (Vier-Augen-Prinzip über Pull Requests)? Wer ist Eigentümer eines Skripts und wird alarmiert? Wie lange werden Skripte, Protokolle und Geheimnisse aufbewahrt, und wer genehmigt Ausnahmen von den Regeln? Für personenbezogene Daten in Protokollen und Skriptausgaben gilt zusätzlich der Datenschutz (siehe Modul 9, Lerneinheit 9.1). Das ist keine Rechtsberatung; die Klärung gehört zu den Aufgaben von Datenschutz und Informationssicherheit im Unternehmen.

> **Merke**
>
> Standardisierung vereinheitlicht wenige, lohnende Dinge: Kopf, Parameter, Fehlerbehandlung, Protokoll, Exit Codes, Ablage. Governance legt Zuständigkeiten und Prüfwege fest. Jede Pflichtregel muss automatisch prüfbar sein (Analyzer mit Settings-Datei), und eine Vorlage macht den Standard zum bequemen Weg.

```powershell
# FALSCH: Regel nur im Wiki, keine Pruefung, keine Vorlage
# "Alle Skripte muessen Fehler behandeln."

# RICHTIG: Regel als pruefbare Einstellung plus Vorlage
# PSScriptAnalyzerSettings.psd1 im Repository-Stamm, Pruefung in der Pipeline,
# neue Skripte entstehen mit New-SkriptGeruest
```

Eine Regel, die nur auf einer Wiki-Seite steht, wird im Alltag vergessen. Eine automatisch geprüfte Regel schlägt im Pull Request Alarm, noch bevor das Skript in Produktion geht.

> **Praxisbeispiel**
>
> Ein Team führt in drei Schritten Standards ein: zuerst nur „Warnung“ in der Pipeline, damit alle die Regeln kennenlernen; nach zwei Monaten werden die Regeln zu Fehlern für neue Skripte; Altbestand bekommt eine Frist und wird nach Kritikalität nachgezogen. Das verhindert, dass die Einführung den Betrieb lahmlegt.

## 12.4 Kosten und Nutzen

Automatisierung kostet Zeit und Geld: Entwicklung, Betrieb, Wartung, gegebenenfalls Lizenzen und Cloud-Ressourcen. Sie spart ebenfalls Zeit und Geld: weniger Handarbeit, weniger Fehler, schnellere Abläufe. Wer Automatisierungsprojekte verantwortet, muss beides gegenüberstellen können, einerseits um Prioritäten zu setzen (welche Aufgabe automatisieren wir zuerst?), andererseits um Entscheidungen gegenüber der Leitung zu begründen. Ein bloßes „das lohnt sich bestimmt“ überzeugt selten; eine kurze Rechnung mit offenen Annahmen schon.

Die Grundrechnung ist die *Amortisationszeit* (englisch *Payback Period*): Wie viele Monate dauert es, bis die Einsparung den einmaligen Entwicklungsaufwand ausgeglichen hat? Dafür braucht man vier Zahlen. Die *Einsparung pro Lauf* (Minuten, die ein Mensch sonst bräuchte), die *Häufigkeit* (Läufe pro Monat), den *Stundensatz* (interne Kosten pro Arbeitsstunde) und den *Aufwand* (Entwicklungsstunden) sowie die laufenden Kosten (Betrieb, Lizenzen). Daraus folgen Einsparung pro Monat, Nettonutzen pro Monat (Einsparung minus laufende Kosten) und die Amortisationszeit als Quotient aus Aufwand und Nettonutzen.

```powershell
function Get-AutomationRoi {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [double] $MinutenProLauf,
        [Parameter(Mandatory)] [double] $LaeufeProMonat,
        [Parameter(Mandatory)] [double] $Stundensatz,
        [Parameter(Mandatory)] [double] $EntwicklungStunden,
        [double] $BetriebStundenProMonat = 0,
        [double] $FixkostenProMonat = 0
    )
    $ersparnisMonat = ($MinutenProLauf / 60) * $LaeufeProMonat * $Stundensatz
    $kostenMonat    = $BetriebStundenProMonat * $Stundensatz + $FixkostenProMonat
    $nettoMonat     = $ersparnisMonat - $kostenMonat
    $einmalig       = $EntwicklungStunden * $Stundensatz
    [pscustomobject]@{
        ErsparnisProMonat  = [math]::Round($ersparnisMonat, 2)
        KostenProMonat     = [math]::Round($kostenMonat, 2)
        NettoProMonat      = [math]::Round($nettoMonat, 2)
        Einmalaufwand      = [math]::Round($einmalig, 2)
        AmortisationMonate = if ($nettoMonat -gt 0) { [math]::Round($einmalig / $nettoMonat, 1) } else { $null }
    }
}

Get-AutomationRoi -MinutenProLauf 20 -LaeufeProMonat 60 -Stundensatz 60 `
    -EntwicklungStunden 16 -BetriebStundenProMonat 1 -FixkostenProMonat 5
Get-AutomationRoi -MinutenProLauf 5 -LaeufeProMonat 4 -Stundensatz 60 -EntwicklungStunden 40
```

Die Funktion rechnet die Einsparung aus: Minuten durch 60 ergibt Stunden pro Lauf, mal Läufe pro Monat und Stundensatz. Die laufenden Kosten sind Betriebsstunden mal Stundensatz plus Fixkosten. Die Amortisationszeit wird nur berechnet, wenn der Nettonutzen positiv ist; sonst bleibt sie leer, denn eine Automatisierung, die mehr kostet, als sie spart, amortisiert sich nie. Der erste Testaufruf (20 Minuten, 60 Läufe im Monat, 60 Euro Stundensatz, 16 Stunden Entwicklung, eine Betriebsstunde pro Monat und 5 Euro Fixkosten) ergab eine Einsparung von 1 200 pro Monat, Kosten von 65, einen Nettonutzen von 1 135, einen Einmalaufwand von 960 und eine Amortisation nach 0,8 Monaten. Der zweite Aufruf (5 Minuten, nur 4 Läufe im Monat, 40 Stunden Entwicklung) ergab eine Einsparung von nur 20 pro Monat bei 2 400 Einmalaufwand, also 120 Monate: Das ist ein Beispiel für eine Aufgabe, die man *nicht* automatisiert, jedenfalls nicht aus Kostengründen.

Die Rechnung hat Grenzen, die man offen benennen sollte. Sie erfasst nur Zeitersparnis, nicht den Wert vermiedener Fehler (ein falsch angelegtes Konto, das ein Sicherheitsrisiko ist), nicht Qualität und Reproduzierbarkeit (jede Ausführung ist gleich) und nicht die Entlastung für interessante Arbeit. Umgekehrt unterschätzt sie oft die Folgekosten: Skripte müssen gewartet, Module aktualisiert, Zertifikate erneuert und Kenntnisse weitergegeben werden. Ein verbreiteter Fehler ist, nur die Entwicklungszeit zu rechnen und den Betrieb zu vergessen. Ehrlicher ist es, für Wartung einen Prozentsatz des Entwicklungsaufwands pro Jahr anzusetzen, beispielsweise einen zweistelligen Prozentsatz; die konkrete Zahl hängt vom Umfeld ab und muss aus eigenen Erfahrungswerten kommen.

> **Ergänzung aus der Nachrecherche**
>
> **Cloud-Kosten der Automatisierung.** Läuft die Automatisierung in der Cloud (zum Beispiel in Azure Automation, siehe Modul 10), entstehen verbrauchsabhängige Kosten, etwa nach Laufzeitminuten oder nach Anzahl der Aufträge. Die Preise ändern sich und unterscheiden sich je Region und Vertrag; die Zahlen müssen deshalb immer aus der aktuellen Preisliste und dem Vertrag des Unternehmens stammen. In diesem Modul werden keine Preise genannt, weil sie hier nicht abgerufen wurden. Vorteilhaft ist, die Kosten in Berichten sichtbar zu machen (Modul 10, Lerneinheit 10.9), damit Entscheidungen auf Daten beruhen.

> **Merke**
>
> Amortisationszeit = Einmalaufwand durch Nettonutzen pro Monat; Nettonutzen = Zeitersparnis minus laufende Kosten. Die Rechnung braucht offene Annahmen (Minuten, Häufigkeit, Stundensatz, Betriebsaufwand) und ergänzt, was sie nicht erfasst: vermiedene Fehler, Qualität, Wartung. Selten genutzte, kurze Aufgaben lohnen sich meist nicht.

```powershell
# FALSCH: nur die Einsparung rechnen, Betrieb vergessen
$monate = $entwicklungsKosten / $ersparnisProMonat

# RICHTIG: laufende Kosten abziehen, Division durch null vermeiden
$netto  = $ersparnisProMonat - $kostenProMonat
$monate = if ($netto -gt 0) { $entwicklungsKosten / $netto } else { $null }
```

Die falsche Rechnung liefert einen zu kurzen Zeitraum, weil Betrieb und Wartung fehlen; bei einem Nettonutzen von null oder weniger entstünde zudem eine Division durch null beziehungsweise ein negativer Zeitraum. Die richtige Fassung fängt beides ab.

> **Praxisbeispiel**
>
> Die Leitung fragt, welche von fünf möglichen Automatisierungen zuerst umgesetzt werden soll. Das Team füllt die Funktion für jede Idee aus und sortiert nach Amortisationszeit; die Berichtserstellung (häufig, aufwändig) steht oben, die einmalige Datenbereinigung unten. Die Annahmen stehen in der Tabelle, sodass die Leitung sie hinterfragen kann.

## 12.5 Skalierung der Automation

Was für zehn Rechner funktioniert, versagt oft bei zehntausend. Skalieren heißt, dieselbe Automatisierung auf viel mehr Objekte, Läufe oder Nutzer auszudehnen, ohne dass sie langsam, unzuverlässig oder zur Belastung für die Zieldienste wird. Dabei gibt es zwei Seiten. Die eine ist die *Geschwindigkeit*: Eine Schleife, die nacheinander 5 000 Server anspricht und je Server eine Sekunde braucht, läuft fast anderthalb Stunden. Die andere ist die *Rücksicht*: Wer dieselben 5 000 Anfragen gleichzeitig an einen Dienst schickt, überlastet ihn und wird abgewiesen.

Für die Geschwindigkeit gibt es in PowerShell 7 den Parameter `-Parallel` von `ForEach-Object`, der Elemente in mehreren Threads gleichzeitig abarbeitet; `-ThrottleLimit` begrenzt die Zahl der gleichzeitigen Durchläufe (im Test lieferte `1..4 | ForEach-Object -Parallel { $_ * 2 } -ThrottleLimit 2` die Werte 2, 4, 6, 8). Eine Besonderheit: Der Skriptblock läuft in einem eigenen Gültigkeitsbereich und sieht Variablen des Aufrufers nicht von selbst; man reicht sie mit dem Präfix `$using:` hinein. Und Parallelität hat ihren Preis: Die Reihenfolge der Ergebnisse ist nicht garantiert, gemeinsame Zähler brauchen sichere Datentypen, und mehr Parallelität ist nicht automatisch schneller, weil das Ziel zum Engpass werden kann.

Für die Rücksicht gibt es zwei Techniken. Die *Drosselung* (Throttling) begrenzt die Rate der Anfragen. Die *Wiederholung mit Wartezeit* (Retry mit *Backoff*) behandelt vorübergehende Fehler: Statusmeldung 429 („zu viele Anfragen“) oder 503 („Dienst nicht verfügbar“) bedeuten, dass ein neuer Versuch später gelingen kann. Viele Dienste teilen in der Antwort mit, wie lange man warten soll, im HTTP-Kopf `Retry-After`. Diesen Wert muss man befolgen. Gibt es keinen, wartet man mit *exponentiellem Backoff*: erst eine Sekunde, dann zwei, dann vier und so weiter, bis zu einem Höchstwert, plus eine zufällige Streuung (*Jitter*), damit nicht alle Skripte zur selben Zeit wiederkommen. Fehler, die kein neuer Versuch heilt (404, 401, 403), dürfen nicht wiederholt werden.

```powershell
function Invoke-MitWiederholung {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [scriptblock] $Aktion,
        [int] $MaxVersuche = 5,
        [double] $BasisSekunden = 1,
        [double] $MaxSekunden = 60,
        [scriptblock] $Warte = { param($s) Start-Sleep -Milliseconds ([int]($s * 1000)) }
    )
    for ($v = 1; $v -le $MaxVersuche; $v++) {
        try { return & $Aktion }
        catch {
            $status       = $_.Exception.Data['Status']
            $retryAfter   = $_.Exception.Data['RetryAfter']
            $wiederholbar = $status -in 429, 500, 502, 503, 504
            if (-not $wiederholbar -or $v -eq $MaxVersuche) { throw }
            $pause = if ($retryAfter) { [double]$retryAfter }
                     else {
                         [math]::Min($MaxSekunden, $BasisSekunden * [math]::Pow(2, $v - 1)) *
                         (0.5 + 0.5 * (Get-Random -Minimum 0.0 -Maximum 1.0))
                     }
            Write-Verbose ("Versuch {0} fehlgeschlagen (Status {1}), warte {2:N1} s" -f $v, $status, $pause)
            & $Warte $pause
        }
    }
}
```

Die Funktion führt die übergebene Aktion bis zu fünfmal aus. Im Fehlerfall liest sie Status und `Retry-After` aus der Ausnahme; in echten Aufrufen von `Invoke-RestMethod` stehen diese Angaben je nach Version in der Antwort der Ausnahme, hier werden sie vereinfachend über `Exception.Data` übergeben. Ist der Status nicht wiederholbar oder war es der letzte Versuch, wirft die Funktion den Fehler unverändert weiter (`throw` ohne Argument). Andernfalls bestimmt sie die Pause: der Wert aus `Retry-After`, sonst die verdoppelte Basiszeit mit Obergrenze, multipliziert mit einem Zufallsfaktor zwischen 0,5 und 1. Die eigentliche Wartefunktion ist als Parameter austauschbar; das macht die Logik testbar, ohne dass der Test wirklich schläft.

Der Test simulierte eine Schnittstelle, die zweimal mit 429 und `Retry-After 3` antwortet, einmal mit 503 und beim vierten Mal erfolgreich ist. Die Pausen betrugen 3,00 und 3,00 Sekunden (befolgtes `Retry-After`) und 3,15 Sekunden für den dritten Versuch (exponentielle Pause von vier Sekunden, mal Zufallsfaktor; der Wert schwankt von Lauf zu Lauf zwischen 2 und 4); die Antwort kam nach dem vierten Versuch. Ein Fehler mit Status 404 wurde sofort weitergereicht und nicht wiederholt. Die simulierte Schnittstelle ersetzt hier einen echten Dienst; echte Antwortköpfe und Statuscodes wurden nicht abgerufen.

> **Ergänzung aus der Nachrecherche**
>
> **Drosselung in Cloud-Diensten.** Dienste wie Microsoft Graph oder Azure Resource Manager begrenzen die Zahl der Anfragen pro Zeitraum und melden Überschreitungen mit 429 und `Retry-After`. Die genauen Grenzen unterscheiden sich je Dienst und Mandant und ändern sich; sie stehen in der Dokumentation des jeweiligen Dienstes und wurden hier nicht abgerufen. Das Muster „Retry-After befolgen, sonst exponentiell warten“ gilt unabhängig davon. Ob ein SDK-Cmdlet selbst wiederholt, hängt von Modul und Version ab; vor dem Einbau einer eigenen Schleife lohnt der Blick in die Dokumentation, damit nicht doppelt gewartet wird.

### Weitere Skalierungshebel.

Neben Parallelität und Wiederholung gibt es Maßnahmen, die oft mehr bringen: *Filtern am Quellsystem* (der Server filtert, nicht das Skript: `-Filter` statt `Where-Object`), *Stapelverarbeitung* (eine Anfrage für 100 Objekte statt 100 Anfragen), *Zwischenspeicherung* (Ergebnisse, die sich nicht ändern, nur einmal holen), *Aufteilung in Abschnitte* mit Fortschrittsmarke, damit ein abgebrochener Lauf fortgesetzt werden kann, und *Idempotenz*, damit Wiederholungen ungefährlich sind. Zu skalieren bedeutet auch, die Protokollmenge zu beherrschen: 5 000 Zeilen pro Lauf sind in Ordnung, 5 Millionen nicht.

> **Merke**
>
> Skalieren heißt schneller und rücksichtsvoll zugleich. Parallelität mit `ForEach-Object -Parallel -ThrottleLimit` beschleunigt, Drosselung und Wiederholung mit `Retry-After`, exponentiellem Backoff und Jitter schützen das Ziel. Nur vorübergehende Fehler (429, 5xx) werden wiederholt, 4xx nicht. Filtern am Quellsystem und Stapelverarbeitung bringen oft mehr als jede Parallelität.

```powershell
# FALSCH: fester Takt, alle Fehler wiederholen, Retry-After ignorieren
for ($i = 0; $i -lt 10; $i++) { try { return Invoke-RestMethod $url } catch { Start-Sleep 1 } }

# RICHTIG: nur wiederholbare Fehler, Retry-After befolgen, sonst Backoff mit Jitter
Invoke-MitWiederholung -Aktion { Invoke-RestMethod $url -TimeoutSec 30 }
```

Die falsche Schleife wiederholt auch Fehler, die sich nie ändern (falsches Kennwort sperrt dabei im schlimmsten Fall ein Konto), und hämmert im Sekundentakt auf einen ohnehin überlasteten Dienst.

> **Praxisbeispiel**
>
> Ein Skript legt für 8 000 Mitarbeiter Postfachberechtigungen an. In der ersten Fassung läuft es sieben Stunden und wird ab Mittag gedrosselt. Nach der Überarbeitung (Stapelabfragen, 8 parallele Durchläufe, Wiederholung nach `Retry-After`, Fortschrittsmarke) braucht es 50 Minuten und kann nach einem Abbruch an der letzten Marke fortsetzen.

## 12.6 BCDR für Automation

*BCDR* steht für *Business Continuity and Disaster Recovery*, auf Deutsch Geschäftsfortführung und Wiederherstellung nach einer Katastrophe. Der Gedanke ist einfach: Man plant vorab, wie der Betrieb nach einem schweren Ausfall weitergeht und wie verlorene Systeme zurückkommen. Für die Automatisierung ist das eine zweischneidige Frage. Einerseits automatisiert man oft genau die Abläufe, die im Notfall gebraucht werden (Wiederherstellung, Umschaltung). Andererseits ist die Automatisierung selbst ein System, das ausfallen kann: der Server mit den Planeraufgaben, das Repository mit den Skripten, der Tresor mit den Zugangsdaten, das Dienstkonto, die Pipeline. Fällt sie aus, wenn sie am dringendsten gebraucht wird, steht man mit leeren Händen da.

Zwei Kennzahlen strukturieren jede Planung. Das *Recovery Time Objective* (RTO) gibt an, wie lange ein Dienst höchstens ausfallen darf, bevor er wieder laufen muss. Das *Recovery Point Objective* (RPO) gibt an, wie viel Datenverlust höchstens hinnehmbar ist, ausgedrückt als Zeitraum: Ein RPO von vier Stunden heißt, dass nach der Wiederherstellung höchstens die Daten der letzten vier Stunden fehlen dürfen, die Sicherung muss also mindestens alle vier Stunden laufen. Beide Werte legt nicht die IT allein fest, sondern der Fachbereich, der die Kosten eines Ausfalls kennt. Sie entscheiden darüber, wie aufwändig und teuer die Vorsorge ausfallen darf.

Was ist an der Automatisierung zu sichern? Erstens der *Quellcode*: Skripte, Module, Pipeline-Definitionen, Vorlagen und Dokumentation liegen in einem Repository, das selbst gesichert oder gespiegelt wird. Zweitens die *Konfiguration und Zustandsdaten*: Planeinträge, Parameterdateien, Zertifikate (ohne Geheimnisse im Klartext), Zuordnungstabellen. Drittens die *Geheimnisse*: Zugangsdaten gehören in einen Tresor mit eigenem Sicherungs- und Wiederherstellungsverfahren (siehe Modul 10, Lerneinheit 10.6), nicht in Skripte. Viertens die *Ausführungsumgebung*: die Server oder Dienste, ihre Einrichtung als Skript oder Vorlage, damit man sie neu aufbauen kann, statt sie von Hand zu rekonstruieren. Fünftens das *Wissen*: das Runbook aus Modul 9 und die Übergabedokumentation aus 12.8. Der wichtigste Satz dieser Lerneinheit lautet: *Eine Sicherung, deren Wiederherstellung nie geübt wurde, ist ein Wunsch, keine Sicherung.*

Die Wiederherstellungsprobe lässt sich zum Teil automatisieren. Eine einfache und sehr wirksame Methode ist die Prüfsummenliste: Zum Zeitpunkt der Sicherung wird für jede Datei eine Prüfsumme (hier SHA-256) in einer Liste festgehalten. Nach einer Wiederherstellung wird verglichen, ob jede Datei vorhanden ist und unverändert aussieht. Die folgenden zwei Funktionen erstellen und prüfen diese Liste.

```powershell
function New-HashManifest {
    param([Parameter(Mandatory)] [string] $Ordner, [Parameter(Mandatory)] [string] $Ziel)
    $basis = (Resolve-Path -LiteralPath $Ordner).Path
    Get-ChildItem -LiteralPath $basis -File -Recurse | ForEach-Object {
        [pscustomobject]@{
            Pfad   = [IO.Path]::GetRelativePath($basis, $_.FullName)
            SHA256 = (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash
            Bytes  = $_.Length
        }
    } | ConvertTo-Json | Set-Content -LiteralPath $Ziel -Encoding utf8
}

function Test-HashManifest {
    param([Parameter(Mandatory)] [string] $Ordner, [Parameter(Mandatory)] [string] $Manifest)
    $basis = (Resolve-Path -LiteralPath $Ordner).Path
    foreach ($e in (Get-Content -LiteralPath $Manifest -Raw | ConvertFrom-Json)) {
        $datei  = Join-Path $basis $e.Pfad
        $status = if (-not (Test-Path -LiteralPath $datei)) { 'FEHLT' }
                  elseif ((Get-FileHash -LiteralPath $datei -Algorithm SHA256).Hash -ne $e.SHA256) { 'ABWEICHEND' }
                  else { 'OK' }
        [pscustomobject]@{ Pfad = $e.Pfad; Status = $status }
    }
}
```

`New-HashManifest` löst den Ordner zu einem absoluten Pfad auf und berechnet für jede Datei den *relativen* Pfad (so bleibt die Liste gültig, wenn der Ordner nach der Wiederherstellung woanders liegt), die Prüfsumme und die Größe. Das Ergebnis geht als JSON in die Zieldatei. `Test-HashManifest` liest die Liste und prüft jede Datei: Fehlt sie, lautet der Status `FEHLT`, stimmt die Prüfsumme nicht, `ABWEICHEND`, sonst `OK`. Im Test wurde ein Ordner mit drei Dateien (`a.txt`, `c.txt` und `sub/b.txt`) erfasst, kopiert, in der Kopie `c.txt` gelöscht und `a.txt` verändert. Die Prüfung ergab: `a.txt` ABWEICHEND, `c.txt` FEHLT, `sub/b.txt` OK. Das Ergebnis eignet sich als Ausgabe für einen Bericht und als Grundlage für einen Exit Code.

> **Ergänzung aus der Nachrecherche**
>
> **Grenzen dieser Probe.** Prüfsummen belegen, dass Dateien *bytegleich* zurückgekommen sind, nicht dass die Anwendung damit läuft. Eine echte Wiederherstellungsübung prüft daher den *Betrieb*: Startet der Dienst, kann das Skript sich anmelden, läuft ein Testauftrag durch? Die Übung wird mit Datum, Dauer und Ergebnis protokolliert; die gemessene Dauer ist die Antwort auf die Frage, ob das RTO realistisch ist. Aussagen zu Sicherungsfunktionen bestimmter Produkte (Repositories, Tresore, Cloud-Dienste) hängen vom jeweiligen Anbieter ab und wurden hier nicht abgerufen.

> **Merke**
>
> BCDR plant Betrieb und Wiederherstellung nach schwerem Ausfall. RTO = tolerierte Ausfalldauer, RPO = tolerierter Datenverlust. Für die Automatisierung zu sichern sind Quellcode, Konfiguration, Geheimnisse, Ausführungsumgebung und Wissen. Nur eine geübte Wiederherstellung zählt; Prüfsummenlisten sind ein einfacher Baustein, ersetzen aber nicht den Betriebstest.

```powershell
# FALSCH: Sicherung nur anlegen, nie zurueckspielen
Copy-Item $quelle $sicherung -Recurse

# RICHTIG: Pruefsummen festhalten und Rueckspielung regelmaessig pruefen
New-HashManifest -Ordner $quelle -Ziel $manifest
# ... spaeter, nach der Wiederherstellung:
Test-HashManifest -Ordner $wiederhergestellt -Manifest $manifest | Where-Object Status -ne 'OK'
```

Die erste Variante gibt das trügerische Gefühl der Sicherheit; ob die Kopie vollständig und lesbar ist, merkt man erst im Ernstfall. Die zweite macht jede Abweichung sichtbar.

> **Praxisbeispiel**
>
> Das Team übt halbjährlich den Fall „Server mit allen Planeraufgaben ist verloren“. Aus Repository und Einrichtungsskript entsteht ein neuer Server in 90 Minuten; das RTO von vier Stunden ist damit eingehalten. Die Übung deckt auf, dass ein Zertifikat nur auf dem alten Server lag; es wird fortan im Tresor geführt.

## 12.7 Change Management

Ein *Change* ist jede Änderung an einem produktiven System: ein neues Skript, eine neue Version, ein geänderter Parameter, ein anderes Dienstkonto. *Change Management* ist der geordnete Umgang damit. Sein Zweck ist nicht Bürokratie, sondern Berechenbarkeit: Man soll wissen, was wann warum geändert wurde, wer es geprüft hat und wie man zurückkommt, wenn etwas schiefgeht. Gerade bei Automatisierung ist das wichtig, weil ein Fehler in einem Skript sich vervielfacht: Was ein Mensch einmal falsch macht, macht das Skript tausendmal.

Die Strenge richtet sich nach dem Risiko. Üblich sind drei Klassen. *Standard-Changes* sind vorab genehmigte, wiederkehrende, risikoarme Änderungen (zum Beispiel das Anlegen eines Benutzers mit dem geprüften Skript); sie brauchen keine Einzelgenehmigung. *Normale Changes* werden beantragt, bewertet und genehmigt, ehe sie umgesetzt werden. *Notfall-Changes* dürfen schnell erfolgen, werden aber nachträglich begründet und geprüft. Die Einordnung legt fest, wer entscheidet.

Wie sieht der technische Weg aus? Skripte liegen in einem Git-Repository. Änderungen entstehen auf einem Zweig und gelangen über einen *Pull Request* in den Hauptzweig; dort prüft ein Mensch (Vier-Augen-Prinzip) und eine Pipeline (Tests, Analyzer, siehe Modul 11). Jede Veröffentlichung bekommt eine Versionsnummer nach dem Schema der *semantischen Versionierung* (Major.Minor.Patch: Patch für Korrekturen, Minor für kompatible Erweiterungen, Major für inkompatible Änderungen) und einen Eintrag im *Changelog*. Die Pipeline kann prüfen, ob diese formalen Dinge stimmen: das ist das *Change-Gate*, eine automatische Schranke vor der Freigabe.

```powershell
function Test-Aenderung {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Projekt)
    $fehler = [System.Collections.Generic.List[string]]::new()
    $psd1 = Get-ChildItem -LiteralPath $Projekt -Filter *.psd1 -File | Select-Object -First 1
    if (-not $psd1) { throw "Keine Manifestdatei in $Projekt." }
    $daten   = Import-PowerShellDataFile -LiteralPath $psd1.FullName
    $version = [version]$daten.ModuleVersion
    $cl = Join-Path $Projekt 'CHANGELOG.md'
    if (-not (Test-Path -LiteralPath $cl)) { $fehler.Add('CHANGELOG.md fehlt.') }
    else {
        $kopf = Select-String -LiteralPath $cl -Pattern '^##\s+\[?(\d+\.\d+\.\d+)\]?' | Select-Object -First 1
        if (-not $kopf) { $fehler.Add('CHANGELOG.md enthält keinen Versionsabschnitt.') }
        elseif ([version]$kopf.Matches[0].Groups[1].Value -ne $version) {
            $fehler.Add("Version im Manifest ($version) stimmt nicht mit dem Changelog ($($kopf.Matches[0].Groups[1].Value)) überein.")
        }
    }
    if (-not $daten.Description) { $fehler.Add('Description im Manifest fehlt.') }
    [pscustomobject]@{ Version = $version; Ok = ($fehler.Count -eq 0); Probleme = $fehler.ToArray() }
}
```

Die Funktion sucht im Projektordner die Manifestdatei (`.psd1`) und liest sie mit `Import-PowerShellDataFile` ein, das Datendateien auswertet, *ohne* Code auszuführen, und deshalb für fremde Dateien sicherer ist als `Invoke-Expression`. Aus dem Manifest nimmt sie die Version, dann sucht sie im Changelog die erste Überschrift mit einer Versionsnummer und vergleicht beide. Außerdem verlangt sie eine Beschreibung. Die Probleme sammelt sie in einer Liste und gibt am Ende ein Objekt mit Version, Gesamtergebnis und Problemliste zurück. Im Test ergab ein Projekt mit Manifest-Version 1.2.0, leerer Beschreibung und Changelog-Eintrag 1.1.0 zwei Probleme (Versionsabweichung, fehlende Beschreibung); nach Korrektur beider Dateien lautete das Ergebnis `Ok = true` ohne Probleme. In einer Pipeline würde man bei `Ok = false` mit Exit Code 20 enden (siehe Modul 9, Lerneinheit 9.3).

> **Ergänzung aus der Nachrecherche**
>
> **Änderungen sichtbar machen und umkehren.** Gute Praxis sind kleine, häufige Änderungen (leichter zu prüfen und zurückzunehmen), das Prinzip „Rollback vor Rollout“ (der Weg zurück ist vor der Änderung geprüft) und eine *Änderungssperre* für kritische Zeiträume. Bei Modulen hilft die Veröffentlichung mit Vorabversionen (siehe Modul 11, Lerneinheit 11.4), damit Anwender neue Versionen erst freiwillig testen. Die Einordnung in Standard-, normale und Notfall-Changes folgt gängigen Rahmenwerken des IT-Servicemanagements; die konkrete Ausgestaltung legt jede Organisation selbst fest. Diese Rahmenwerke wurden hier nicht abgerufen.

> **Merke**
>
> Change Management macht Änderungen nachvollziehbar und umkehrbar. Die Strenge folgt dem Risiko (Standard, normal, Notfall). Technisch: Repository, Pull Request mit Vier-Augen-Prüfung, Pipeline, Versionsnummer nach Major.Minor.Patch, Changelog und ein Change-Gate, das diese Formalien automatisch prüft.

```powershell
# FALSCH: direkt auf dem Server editieren, keine Version, keine Spur
notepad C:\Skripte\Sicherung.ps1

# RICHTIG: Aenderung ueber Zweig und Pull Request, Version und Changelog angehoben
git switch -c fix/zielpfad
# ... aendern, Version in .psd1 anheben, CHANGELOG.md ergaenzen ...
git push -u origin fix/zielpfad     # Pull Request eroeffnen, Pipeline prueft
```

Das Editieren auf dem Server hinterlässt keine Spur; niemand weiß später, was sich wann geändert hat, und eine Rücknahme ist Handarbeit. Der Weg über den Pull Request liefert Prüfung, Verlauf und eine Version, auf die man zurückgehen kann. Die Befehle sind ein Muster und wurden nicht ausgeführt.

> **Praxisbeispiel**
>
> Nach einem Vorfall, der auf eine ungeprüfte Änderung zurückging, führt das Team Pull Requests mit automatischem Change-Gate ein. In den ersten Wochen bricht das Gate oft ab, weil Changelog-Einträge fehlen; nach einem Monat gehört das Anheben von Version und Changelog zur Routine, und jede Änderung ist im Verlauf nachvollziehbar.

## 12.8 Betriebsübergabe

Skripte werden früher oder später übergeben: an ein anderes Team, an einen Dienstleister, an einen Nachfolger. Eine Übergabe ist gelungen, wenn die übernehmende Seite das Skript *betreiben* kann, ohne den Autor anzurufen. Das ist deutlich mehr als eine Dateiablage. Typische Übergaben scheitern an Kleinigkeiten, die dem Autor selbstverständlich sind: Wo liegt das Dienstkonto-Kennwort? Welcher Planer startet das Skript? Wen ruft man, wenn es um 3 Uhr nachts scheitert? Was passiert, wenn man es zweimal startet?

Eine *Übergabe-Checkliste* stellt sicher, dass nichts vergessen wird. Sie enthält Pflichtfelder: Zweck, Eigentümer (Team, nicht nur Person), Ort des Quellcodes, Verweis auf das Runbook, Alarmierungsweg, Ort der Zugangsdaten, Datum des letzten erfolgreichen Tests und Ergebnis der letzten Wiederherstellungsübung. Dazu kommen Wissensfragen: Welche Abhängigkeiten gibt es, welche Eigenheiten („nicht vor 02:00 starten, weil dann die Datenbank gesichert wird“) und welche bekannten Probleme sind offen? Wie jede Checkliste lebt sie nur, wenn sie geprüft wird. Deshalb liegt sie als maschinenlesbare Datei im Repository, und ein kleines Skript prüft sie.

```powershell
# uebergabe.psd1
@{
    Skript         = 'Sicherung-Import'
    Verantwortlich = 'team-betrieb@example.org'
    Repository     = 'https://git.example.org/betrieb/sicherung'
    Runbook        = 'https://wiki.example.org/runbooks/sicherung'
    Alarmierung    = 'Rufbereitschaft Windows'
    Zugangsdaten   = 'Key Vault kv-betrieb, Secret sicherung-konto'
    LetzterTest    = '2026-10-10'
    Restore        = ''
}

function Test-Uebergabe {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad, [int] $MaxAlterTage = 90)
    $d = Import-PowerShellDataFile -LiteralPath $Pfad
    $pflicht = 'Skript', 'Verantwortlich', 'Repository', 'Runbook',
               'Alarmierung', 'Zugangsdaten', 'LetzterTest', 'Restore'
    foreach ($k in $pflicht) {
        $wert   = $d[$k]
        $status = if ([string]::IsNullOrWhiteSpace($wert)) { 'LEER' } else { 'OK' }
        if ($k -eq 'LetzterTest' -and $status -eq 'OK') {
            $alter = ((Get-Date) - [datetime]::ParseExact($wert, 'yyyy-MM-dd',
                       [Globalization.CultureInfo]::InvariantCulture)).TotalDays
            if ($alter -gt $MaxAlterTage) { $status = 'VERALTET' }
        }
        [pscustomobject]@{ Feld = $k; Status = $status }
    }
}
```

Die Checkliste ist eine `.psd1`-Datei, das gleiche Format wie ein Modulmanifest, das PowerShell sicher als Daten einliest. Die Prüffunktion geht jedes Pflichtfeld durch. Ein leeres Feld hat den Status `LEER`. Beim Feld `LetzterTest` wird zusätzlich das Datum mit fester Formatvorgabe und invarianter Kultur gelesen und mit dem Höchstalter von 90 Tagen verglichen; ein zu alter Test ist `VERALTET`. Im Test ergaben sich für die Beispieldatei sieben Felder mit `OK` (der letzte Test lag 15 Tage zurück) und das Feld `Restore` mit `LEER`: Es fehlte also der Nachweis der Wiederherstellungsübung. Genau so soll die Prüfung wirken: Sie lässt eine Übergabe nicht „fertig“ werden, solange offene Punkte bleiben.

### Mehr als ein Dokument.

Übergabe ist auch ein Gespräch. Bewährt hat sich der Ablauf „gemeinsam betreiben, dann übernehmen“: Die übernehmende Seite führt unter Aufsicht des Autors einen kompletten Lauf aus, bearbeitet einen simulierten Störfall mit dem Runbook und stellt dabei fest, was fehlt. Erst danach wechselt die Verantwortung formal. In den ersten Wochen bleibt der Autor erreichbar, aber nicht zuständig. Der Wissenstransfer ist am Ende der Prüfstein: Wenn die übernehmende Seite eine Frage stellt, die im Runbook nicht beantwortet wird, ist das ein Fehler im Runbook, nicht bei der Seite.

> **Ergänzung aus der Nachrecherche**
>
> **Eigentümer ist ein Team.** Ein Skript, dessen einzige Ansprechperson eine einzelne Kollegin ist, hat ein Ausfallrisiko (Urlaub, Krankheit, Wechsel). In der Checkliste steht daher eine Gruppenadresse, und es gibt mindestens eine Vertretung. Zugangsdaten gehören im Tresor einer Gruppe, nicht in ein persönliches Konto; wechselt eine Person, endet nicht gleichzeitig der Zugriff der Automatisierung. Das ist Organisationswissen und keine Aussage zu einem bestimmten Produkt.

> **Merke**
>
> Eine Übergabe ist gelungen, wenn der Betrieb ohne den Autor möglich ist. Die Checkliste (Zweck, Eigentümer-Team, Quellcode, Runbook, Alarmierung, Zugangsdaten, letzter Test, Wiederherstellungsübung) liegt maschinenlesbar im Repository und wird automatisch geprüft. Gemeinsam betreiben, dann übernehmen.

```powershell
# FALSCH: "Das Skript liegt auf dem Server, frag mich, wenn was ist."
# RICHTIG: Checkliste vollstaendig, Pruefung gruen, Probelauf durch die uebernehmende Seite
Test-Uebergabe -Pfad ./uebergabe.psd1 | Where-Object Status -ne 'OK'   # darf nichts liefern
```

Die falsche Übergabe verlagert das Risiko auf eine einzelne Person; die richtige macht offene Punkte sichtbar, bevor die Verantwortung wechselt.

> **Praxisbeispiel**
>
> Ein Dienstleister übergibt zwölf Skripte an das interne Team. Die Prüfung der Checklisten zeigt bei vier Skripten ein leeres Alarmierungsfeld und bei drei einen Nachweis, der älter als ein Jahr ist. Die Übergabe wird erst nach Behebung dieser Punkte abgenommen; der Vertrag nennt die grüne Prüfung als Abnahmekriterium.

## 12.9 Kontinuierliche Verbesserung

Automatisierung ist nie fertig. Systeme ändern sich, Anforderungen wandern, und Skripte altern: Module werden aktualisiert, Dienste ändern Schnittstellen, Kollegen finden neue Randfälle. *Kontinuierliche Verbesserung* (japanisch *Kaizen*, im Englischen *continuous improvement*) heißt, die Qualität der Automatisierung in kleinen, regelmäßigen Schritten zu erhöhen, statt sie in seltenen Großprojekten aufzufrischen. Die Grundlage dafür sind Zahlen: Wer nicht misst, kann nicht sagen, ob es besser wird.

Die Daten dafür hat man schon, wenn man Modul 9 umgesetzt hat. Die strukturierten Protokolle enthalten Läufe, Fehler, Dauer. Zusammengefasst nach Tagen oder Wochen ergeben sie Trends: Steigt die Fehlerzahl? Wird der Lauf langsamer? Wiederholt sich dasselbe Fehlerbild? Dazu kommen Kennzahlen aus dem Betrieb: Anzahl der Vorfälle, durchschnittliche Zeit bis zur Behebung, Anteil der Skripte mit Runbook, mit Tests, mit aktuellem Analyzer-Ergebnis. Eine kleine Zahl gut gewählter Kennzahlen genügt; wichtig ist, dass sie regelmäßig angesehen werden.

```powershell
function Get-LogTrend {
    param([Parameter(Mandatory)] [string] $Ordner)
    Get-ChildItem -LiteralPath $Ordner -Filter *.jsonl | Sort-Object Name | ForEach-Object {
        $e = Get-Content -LiteralPath $_.FullName | ConvertFrom-Json -DateKind String
        $dauern = @($e | Where-Object aktion -eq 'Ende' | ForEach-Object { $_.daten.dauerMs })
        [pscustomobject]@{
            Tag      = $_.BaseName
            Laeufe   = @($e.runId | Select-Object -Unique).Count
            Fehler   = @($e | Where-Object level -eq 'ERROR').Count
            MittelMs = [math]::Round(($dauern | Measure-Object -Average).Average)
        }
    }
}
Get-LogTrend -Ordner ./logs | Format-Table -AutoSize
```

Die Funktion erwartet einen Ordner mit einer Protokolldatei pro Tag (Dateiname gleich Datum). Für jede Datei liest sie die Einträge, zählt die verschiedenen Run-IDs als Läufe, die Einträge mit Level `ERROR` als Fehler und bildet den Mittelwert der Laufzeit aus den Einträgen mit der Aktion `Ende`, deren Zusatzdaten das Feld `dauerMs` enthalten. Im Test mit fünf erfundenen Tagen zu je drei Läufen zeigte die Auswertung zwei Dinge: Die mittlere Laufzeit stieg von 1 155 ms am ersten Tag auf 5 201 ms am fünften, und an den Tagen vier und fünf trat je ein Fehler auf. Die Daten sind künstlich erzeugt; die Auswertung zeigt das Prinzip, keine reale Messung. Ein solcher Anstieg wäre im Alltag das Signal, nachzusehen, bevor es zum Ausfall kommt.

Aus Trends entsteht ein regelmäßiger *Verbesserungskreislauf*: beobachten (Kennzahlen), bewerten (welches Problem kostet am meisten?), eine Maßnahme wählen, umsetzen, wirken lassen und wieder messen. Bewährte Maßnahmen sind: ein wiederkehrender Fehler wird ursächlich beseitigt statt jedes Mal neu behandelt, ein Handgriff im Runbook wird zum Skript, ein Skript bekommt Tests für den Fehler, der es getroffen hat, veraltete Skripte werden abgeschaltet statt gepflegt. Hilfreich ist ein fester Termin, etwa eine kurze monatliche Durchsicht der Kennzahlen und eine Quartalsrunde zum Aufräumen. Und Skripte, die niemand mehr braucht, werden gelöscht: Der billigste Wartungsaufwand ist der, der gar nicht erst entsteht.

> **Ergänzung aus der Nachrecherche**
>
> **Lernen aus Vorfällen.** Der wertvollste Eingang in den Verbesserungskreislauf sind Vorfälle. Die schuldfreie Nachbetrachtung aus Modul 9, Lerneinheit 9.8, liefert Maßnahmen mit Verantwortlichen und Terminen; diese gehören in dieselbe Liste wie alle anderen Verbesserungen und werden nachverfolgt. Wichtig ist, Maßnahmen *abzuschließen*: Ein Maßnahmenfriedhof, der wächst, ohne dass etwas erledigt wird, entwertet jede Nachbetrachtung.

> **Merke**
>
> Kontinuierliche Verbesserung braucht Daten (Protokolle, wenige Kennzahlen), einen festen Rhythmus und abgeschlossene Maßnahmen. Trends aus den strukturierten Protokollen zeigen Verschlechterungen früh. Wiederkehrende Fehler beseitigt man ursächlich, Handarbeit wird zum Skript, Altlasten werden abgeschaltet.

```powershell
# FALSCH: Fehler jedes Mal von Hand beheben, nichts festhalten
# "Ach, das Skript ist wieder haengengeblieben, ich starte es neu."

# RICHTIG: Muster erkennen, Ursache beseitigen, Test und Kennzahl ergaenzen
Get-LogTrend -Ordner ./logs | Where-Object Fehler -gt 0 | Sort-Object Tag -Descending
# -> Ursache suchen, Pester-Test fuer den Fall schreiben, Zeile im Runbook aktualisieren
```

Wer jeden Fehler nur neu startet, bezahlt ihn immer wieder. Wer das Muster sichtbar macht, kann die Ursache beseitigen und die Wiederholung künftig verhindern.

> **Praxisbeispiel**
>
> In der monatlichen Durchsicht fällt auf, dass die Laufzeit der Berichtserstellung seit Wochen steigt. Die Ursache ist eine wachsende Tabelle ohne Index. Das Team behebt sie, ergänzt einen Alarm für Laufzeiten über 20 Minuten und streicht zwei seit einem Jahr nicht mehr genutzte Skripte aus dem Inventar.

## 12.10 Übungsaufgaben zu Modul 12

Die Aufgaben steigen in der Schwierigkeit an. Die Aufgaben 3, 5 und 7 sind praktisch (Code schreiben). Die Lösungen mit Erklärung folgen im nächsten Abschnitt.

**Aufgabe 1.** Warum genügt für eine Bestandsaufnahme die Dateisuche allein nicht? Nennen Sie zwei Quellen für die Laufzeitsicht und erklären Sie, warum der Syntaxbaum einer Textsuche überlegen ist.

**Aufgabe 2.** Nennen Sie drei typische Unterschiede zwischen Windows PowerShell 5.1 und PowerShell 7, die ein Skript beim Umstieg zum Scheitern bringen können, und für jeden einen Lösungsweg.

**Aufgabe 3.** **Praxis:** Schreiben Sie eine Funktion `Find-Veraltet`, die eine `.ps1`-Datei mit dem Syntaxbaum untersucht und alle Aufrufe von `Get-WmiObject` und `Get-EventLog` samt Zeilennummer ausgibt.

**Aufgabe 4.** Welche vier Angaben braucht man für die Berechnung der Amortisationszeit? Berechnen Sie sie für eine Aufgabe, die 15 Minuten dauert, 40-mal im Monat anfällt, bei einem Stundensatz von 50 Euro, 12 Stunden Entwicklung und ohne laufende Kosten.

**Aufgabe 5.** **Praxis:** Schreiben Sie eine Funktion, die einen Skriptblock bis zu dreimal ausführt, wenn er mit einer Ausnahme endet, die in `Data['Status']` den Wert 503 trägt, und dazwischen jeweils 2 Sekunden wartet. Andere Fehler sollen sofort weitergereicht werden.

**Aufgabe 6.** Erklären Sie RTO und RPO an einem Beispiel: Eine Datenbank wird jede Nacht um 02:00 Uhr gesichert, ein Ausfall tritt um 16:00 Uhr auf und die Wiederherstellung dauert drei Stunden. Welche Werte hat man tatsächlich erreicht?

**Aufgabe 7.** **Praxis:** Schreiben Sie ein Skript, das in einem Ordner alle `.ps1`-Dateien auf Syntaxfehler prüft (mit dem Parser, ohne sie auszuführen) und mit Exit Code 20 endet, wenn mindestens eine Datei Fehler hat, sonst mit 0. Es soll pro fehlerhafter Datei den ersten Fehler mit Zeile ausgeben.

**Aufgabe 8.** Sie übernehmen ein Skript, dessen Checkliste „Restore“ leer lässt und dessen letzter Test 200 Tage alt ist. Welche Schritte unternehmen Sie, bevor Sie die Verantwortung übernehmen? Welche Kennzahlen würden Sie danach monatlich betrachten?

## 12.11 Lösungen zu Modul 12

### Aufgabe 1.

Die Dateisuche findet nur, was herumliegt, nicht was tatsächlich läuft. Eine Datei ohne Aufrufer ist Altlast, ein Planeintrag, der auf eine fehlende Datei zeigt, ein schleichender Ausfall; erst der Vergleich zeigt beides. Quellen der Laufzeitsicht sind geplante Aufgaben (Aufgabenplaner, Cron), Pipeline-Definitionen, Runbooks in Azure Automation und Dienste, die PowerShell starten. Der Syntaxbaum kennt die Struktur: Er zählt nur echte Befehlsaufrufe, eine Textsuche meldet auch Kommentare und Zeichenketten.

### Aufgabe 2.

Mögliche Antworten: (a) Fehlende Cmdlets wie `Get-WmiObject` oder `Get-EventLog`: Ersatz durch `Get-CimInstance` und `Get-WinEvent`. (b) Module, die nur unter Windows PowerShell 5.1 laufen: Laden mit `Import-Module Name -UseWindowsPowerShell` (die Objekte sind dann serialisiert) oder Ersatz des Moduls. (c) Geänderte Standardkodierung (UTF-8 ohne BOM): bei Bedarf ausdrücklich `-Encoding utf8BOM` angeben. (d) Neue Syntax unter 5.1 nicht lauffähig: `#requires -Version 7.0` deklarieren und Planer auf `pwsh` umstellen. Drei dieser vier genügen.

### Aufgabe 3.

```powershell
function Find-Veraltet {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Datei)
    $ziele = 'Get-WmiObject', 'Get-EventLog'
    $tokens = $null; $fehler = $null
    $ast = [System.Management.Automation.Language.Parser]::ParseFile(
               (Resolve-Path -LiteralPath $Datei).Path, [ref]$tokens, [ref]$fehler)
    $ast.FindAll({ param($n) $n -is [System.Management.Automation.Language.CommandAst] }, $true) |
        Where-Object { $_.GetCommandName() -in $ziele } |
        ForEach-Object {
            [pscustomobject]@{ Zeile = $_.Extent.StartLineNumber; Befehl = $_.GetCommandName() }
        }
}
```

Die Funktion zerlegt die Datei in den Syntaxbaum und sucht alle Befehlsaufrufe. Der Filter behält nur die gesuchten Namen; `Extent.StartLineNumber` liefert die Zeilennummer des Aufrufs. Im Test mit der Beispieldatei aus 12.1 wurden `Get-WmiObject` in Zeile 2 und `Get-EventLog` in Zeile 3 gefunden.

### Aufgabe 4.

Benötigt werden Einsparung pro Lauf, Häufigkeit, Stundensatz und Entwicklungsaufwand (dazu die laufenden Kosten). Rechnung: 15 Minuten sind 0,25 Stunden, mal 40 Läufe sind 10 Stunden, mal 50 Euro sind 500 Euro Einsparung pro Monat. Der Einmalaufwand beträgt 12 mal 50 gleich 600 Euro. Der Nettonutzen entspricht der Einsparung (keine laufenden Kosten), also 500 Euro pro Monat. Die Amortisationszeit ist 600 geteilt durch 500, also 1,2 Monate.

### Aufgabe 5.

```powershell
function Invoke-NachMaximalDreiVersuchen {
    param([Parameter(Mandatory)] [scriptblock] $Aktion,
          [scriptblock] $Warte = { Start-Sleep -Seconds 2 })
    for ($v = 1; $v -le 3; $v++) {
        try { return & $Aktion }
        catch {
            if ($_.Exception.Data['Status'] -ne 503 -or $v -eq 3) { throw }
            & $Warte
        }
    }
}
```

Die Schleife führt die Aktion aus und gibt das Ergebnis bei Erfolg sofort zurück. Im `catch`-Block prüft sie den Status: Ist er nicht 503 oder war es der dritte Versuch, wird der Fehler mit `throw` unverändert weitergereicht; sonst wartet sie und versucht es erneut. Die Wartefunktion ist als Parameter ausgelagert, damit man sie im Test durch einen leeren Block ersetzen kann. Im Test lieferte eine Aktion, die zweimal 503 meldet, „ok nach 3“; eine Aktion, die immer 503 meldet, scheiterte nach drei Versuchen; ein anderer Fehler wurde nach dem ersten Versuch weitergereicht.

### Aufgabe 6.

Das RPO ist der tolerierte Datenverlust, das RTO die tolerierte Ausfalldauer. Die Nachtsicherung um 02:00 und der Ausfall um 16:00 bedeuten, dass bis zu 14 Stunden Daten fehlen: Der tatsächlich erreichte Datenverlust liegt bei 14 Stunden. Die Wiederherstellung dauert drei Stunden, der tatsächlich erreichte Ausfall beträgt also drei Stunden (bis 19:00 Uhr). Ob das genügt, hängt von den Zielwerten ab: Wäre das RPO vier Stunden, müsste häufiger gesichert werden, etwa alle vier Stunden oder per Transaktionsprotokoll.

### Aufgabe 7.

```powershell
param([Parameter(Mandatory)] [string] $Ordner)
$anzahl = 0
foreach ($datei in Get-ChildItem -LiteralPath $Ordner -Filter *.ps1 -Recurse -File) {
    $tokens = $null; $fehler = $null
    [void][System.Management.Automation.Language.Parser]::ParseFile(
        $datei.FullName, [ref]$tokens, [ref]$fehler)
    if ($fehler.Count -gt 0) {
        $anzahl++
        Write-Output ("{0}: Zeile {1}: {2}" -f $datei.Name,
                      $fehler[0].Extent.StartLineNumber, $fehler[0].Message)
    }
}
if ($anzahl -gt 0) { exit 20 }
exit 0
```

Der Parser führt nichts aus, er sammelt nur Syntaxfehler. Für jede fehlerhafte Datei wird der erste Fehler mit Zeilennummer ausgegeben, und am Ende steht der Exit Code gemäß der Tabelle aus Modul 9. Im Test ergab ein Ordner mit einer defekten Datei die Ausgabe „kaputt.ps1: Zeile 1: Unexpected token ’{’ in expression or statement.“ und Exit Code 20; ein Ordner mit nur einer fehlerfreien Datei lieferte den Code 0.

### Aufgabe 8.

Schritte: Zuerst die offenen Punkte der Checkliste schließen. Das heißt, eine Wiederherstellungsübung durchführen und das Ergebnis eintragen, den Test erneuern (der letzte liegt über den im Beispiel angenommenen 90 Tagen) und das Runbook mit dem Autor durchgehen. Dann das Skript gemeinsam mit dem Autor einmal vollständig betreiben und einen simulierten Störfall mit dem Runbook bearbeiten. Erst danach wechselt die Verantwortung. Monatliche Kennzahlen: Anzahl Läufe, Fehler und Laufzeit aus den Protokollen (siehe 12.9), Anzahl Vorfälle und Zeit bis zur Behebung, Alter von letztem Test und letzter Wiederherstellungsübung.
