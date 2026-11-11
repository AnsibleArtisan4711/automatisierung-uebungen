# Modul 01 - PowerShell Grundlagen und Architektur

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (1.1 bis 1.8; 1.9 Übungsaufgaben, 1.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 8 h                                  |
| Summe der Lerneinheiten (Detailblatt) | 10 h (8 Lerneinheiten à 1,25 h)      |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Im Lehrplan weichen die Stunden des Übersichtsblatts (8 h) und die Summe der Lerneinheiten im Detailblatt (10 h) voneinander ab. Dieses Skript folgt dem Detailblatt, weil dort jede Lerneinheit einzeln ausgewiesen ist. Der Umfang richtet sich daher nach den 10 Stunden.

### Lernziele.

Nach diesem Modul können Sie die Shell, die Objektpipeline, die Editionen und die typischen Einsatzszenarien von PowerShell einordnen, die zugehörigen Werkzeuge samt Sicherheitsaspekten und Betriebspraxis beurteilen und verstehen, warum wiederholbare Automation für Windows, Cloud und CI/CD das eigentliche Ziel ist.

| **Nr.** | **Lerneinheit**                     | **Dauer** | **Prio** |
|:--------|:------------------------------------|:----------|:---------|
| 1.1     | PowerShell Zweck und Einsatzfelder  | 1,25 h    | 1        |
| 1.2     | Windows PowerShell vs PowerShell 7+ | 1,25 h    | 1        |
| 1.3     | Objektpipeline-Grundidee            | 1,25 h    | 1        |
| 1.4     | Cmdlet-Namensschema                 | 1,25 h    | 1        |
| 1.5     | Host, Console und Terminal          | 1,25 h    | 1        |
| 1.6     | Execution Model                     | 1,25 h    | 1        |
| 1.7     | Cross-Plattform-Grundlagen          | 1,25 h    | 2        |
| 1.8     | Praxisumgebung                      | 1,25 h    | 2        |
|         | **Summe**                           | **10 h**  |          |

## 1.1 PowerShell Zweck und Einsatzfelder

PowerShell ist gleichzeitig drei Dinge, und genau diese Verbindung macht es so nützlich. Erstens ist es eine *Shell*, also ein Programm, in das Sie Befehle eintippen und sofort eine Antwort erhalten. Fachleute sprechen von einer REPL (Read-Eval-Print-Loop): Die Eingabe wird gelesen, ausgewertet, das Ergebnis wird ausgegeben, und dann wartet die Shell auf die nächste Eingabe. Zweitens ist PowerShell eine *Skriptsprache*: Dieselben Befehle lassen sich in eine Datei mit der Endung `.ps1` schreiben, mit Variablen, Bedingungen, Schleifen und Funktionen kombinieren und beliebig oft ausführen. Drittens ist PowerShell eine *Automationsplattform*, weil rund um die Sprache ein Ökosystem aus Modulen, Remoting, Zeitplanung, Konfigurationsverwaltung und Integrationen in Cloud-Dienste entstanden ist. Wer PowerShell lernt, lernt also nicht nur eine Syntax, sondern eine Arbeitsweise: wiederkehrende Aufgaben nicht mehr per Mausklick zu erledigen, sondern als nachvollziehbaren, wiederholbaren Code festzuhalten.

Die Wurzeln liegen in der Windows-Administration. Die klassische Eingabeaufforderung (`cmd.exe`) konnte Text verarbeiten, aber kaum auf die Verwaltungsschnittstellen von Windows zugreifen. Unter Unix lösten Administratoren dieses Problem mit vielen kleinen Programmen, die Text durch Pipes aneinanderreihen. PowerShell, das ursprünglich unter dem Codenamen *Monad* entwickelt wurde und 2006 erschien, wählte einen anderen Weg: Es baut auf dem *.NET*-Framework auf, einer Laufzeitumgebung mit einer riesigen Klassenbibliothek, und reicht zwischen den Befehlen keine Textzeilen, sondern *Objekte* weiter (Lerneinheit 1.3). Seit PowerShell 6 ist das Projekt quelloffen und plattformübergreifend; es läuft auf Windows, Linux und macOS.

Die Einsatzfelder lassen sich grob in drei Szenarien gliedern, die sich in der Praxis überlappen. In der *klassischen Administration* verwalten Sie Server und Arbeitsplätze: Dienste starten und stoppen, Ereignisprotokolle auswerten, Benutzerkonten anlegen, Dateiberechtigungen prüfen, Software verteilen. In *DevOps*-Umgebungen ist PowerShell das Bindeglied in Build- und Release-Pipelines: Es ruft Compiler und Testwerkzeuge auf, erzeugt Artefakte, steuert Infrastruktur-Werkzeuge und wertet deren Ausgaben aus. In *Cloud-Szenarien* verwalten Sie Ressourcen in Azure oder Microsoft 365 über dafür bereitgestellte Module; ein Skript kann dann etwa hundert virtuelle Maschinen taggen oder die Lizenzzuweisung eines ganzen Mandanten prüfen. In allen drei Fällen zählt derselbe Gedanke: Was ein Skript einmal richtig macht, macht es tausendmal gleich richtig.

Ein erstes lauffähiges Beispiel zeigt, wie Shell, Skriptsprache und Objektdenken zusammenspielen. Das folgende Skript ermittelt die fünf Prozesse mit dem größten Arbeitsspeicherverbrauch, rechnet den Wert in Megabyte um, zeigt ihn als Tabelle an und speichert ihn zusätzlich als CSV-Datei im temporären Verzeichnis des Systems.

```powershell
$top = Get-Process |
    Sort-Object -Property WorkingSet64 -Descending |
    Select-Object -First 5 -Property Name, Id,
        @{ Name = 'SpeicherMB'; Expression = { [math]::Round($_.WorkingSet64 / 1MB, 1) } }

$top | Format-Table -AutoSize

$ziel = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'top5.csv'
$top | Export-Csv -Path $ziel -NoTypeInformation
"Bericht gespeichert unter: $ziel"
```

Die erste Anweisung besteht aus einer Pipeline mit drei Gliedern, die durch das Zeichen `|` getrennt sind. `Get-Process` liefert für jeden laufenden Prozess ein Objekt. `Sort-Object` sortiert diese Objekte nach der Eigenschaft `WorkingSet64`, das ist der belegte Arbeitsspeicher in Byte, absteigend. `Select-Object` behält nur die ersten fünf Objekte und von diesen nur die Eigenschaften `Name` und `Id`. Die geschweifte Struktur mit `Name` und `Expression` ist eine sogenannte *berechnete Eigenschaft*: Sie erzeugt eine neue Spalte, deren Wert der Skriptblock in `Expression` berechnet; `$_` steht dabei für das aktuelle Objekt, und `1MB` ist eine eingebaute Zahlenkonstante für 1 048 576. Dass die Zeile über mehrere Zeilen verteilt ist, liegt nur an der Lesbarkeit: Nach einem Pipe-Zeichen am Zeilenende erwartet PowerShell automatisch die Fortsetzung. `Format-Table` dient ausschließlich der Darstellung am Bildschirm. `Join-Path` setzt den Dateipfad plattformunabhängig zusammen, und `Export-Csv` schreibt die Objekte als Tabelle in eine Datei. Der Parameter `-NoTypeInformation` verhindert in Windows PowerShell 5.1 eine zusätzliche Kommentarzeile mit dem Typnamen; in PowerShell 7 ist dieses Verhalten bereits Standard, der Parameter schadet dort aber nicht, sodass das Skript in beiden Editionen identisch läuft.

> **Merke**
>
> PowerShell verarbeitet Objekte mit Eigenschaften und Methoden, keinen Text. Wer das Konzept der Objektpipeline verstanden hat, braucht kaum jemals Textwerkzeuge wie `grep` oder `awk`, um Informationen aus Befehlsausgaben herauszuschneiden.

Typische Fehler von Einsteigern betreffen weniger die Syntax als die Denkweise. Der häufigste ist, PowerShell wie eine Textshell zu benutzen und Ausgaben mit Zeichenketten-Operationen zu zerlegen, obwohl die gewünschte Information längst als Eigenschaft vorliegt. Ein zweiter Fehler ist, interaktive Kurzformen in Skripte zu übernehmen. Das folgende Gegenbeispiel zeigt beides.

```powershell
# FALSCH: Text zerlegen und Kurzformen in einem Skript
$zeilen = (Get-Process | Out-String) -split "`n"
$zeilen | select -first 5

# RICHTIG: Eigenschaften nutzen und Cmdlets ausschreiben
Get-Process | Select-Object -First 5 -Property Name, Id
```

Im falschen Beispiel wird die Tabellendarstellung in eine einzige Zeichenkette verwandelt und mühsam wieder in Zeilen zerlegt; dabei geht jede Typinformation verloren, und das Ergebnis hängt von Spaltenbreiten und Sprache ab. Zusätzlich verbirgt sich hinter `select` ein Alias, der Skripte schwerer lesbar und weniger robust macht (Lerneinheit 2.6).

> **Praxisbeispiel**
>
> Ein typischer Einstieg in der Praxis ist die Onboarding-Automatisierung: Wenn eine neue Mitarbeiterin anfängt, legt ein Skript das Benutzerkonto an, weist Gruppen und Lizenzen zu, erstellt das Home-Verzeichnis und sendet eine Bestätigung an die Personalabteilung. Dieselbe Folge von Schritten, die früher eine halbe Stunde Klickarbeit bedeutete, läuft in Sekunden und liefert jedes Mal dasselbe Ergebnis. Genau diese Wiederholbarkeit begegnet Ihnen in Administration, DevOps und Cloud-Betrieb immer wieder.

## 1.2 Windows PowerShell vs PowerShell 7+

Hinter dem Namen *PowerShell* verbergen sich zwei Produktlinien, die sich im Alltag zwar sehr ähnlich anfühlen, technisch aber verschieden sind. *Windows PowerShell* (aktuell Version 5.1, 2016 veröffentlicht) ist Bestandteil des Betriebssystems Windows und baut auf dem klassischen *.NET Framework* auf. Es wird nicht mehr um neue Funktionen erweitert, sondern bekommt nur noch Fehlerkorrekturen im Rahmen des Windows-Supports. *PowerShell 7* (ab Version 6 hieß die Linie zunächst PowerShell Core) ist dagegen ein eigenständiges, quelloffenes Produkt auf Basis des modernen, plattformübergreifenden *.NET*. Es wird parallel zu Windows PowerShell installiert, ersetzt dieses also nicht, und wird kontinuierlich weiterentwickelt. Für neue Skripte ist PowerShell 7 die richtige Wahl; Windows PowerShell 5.1 bleibt wichtig, weil auf vielen Servern Module existieren, die nur dort laufen, und weil sich Bestandsskripte nicht über Nacht migrieren lassen.

Sie erkennen die beiden Programme zunächst am Namen der ausführbaren Datei: Windows PowerShell startet mit `powershell.exe`, PowerShell 7 mit `pwsh.exe` (unter Linux und macOS `pwsh`). Diese Umbenennung ist bewusst erfolgt, damit beide Versionen nebeneinander betrieben werden können und ein Aufruf eindeutig ist. Innerhalb einer laufenden Sitzung geben die Variable `$PSVersionTable` und besonders deren Eigenschaft `PSEdition` Auskunft: Windows PowerShell meldet `Desktop`, PowerShell 7 meldet `Core`.

```powershell
$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
[System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription

if ($PSVersionTable.PSEdition -eq 'Core') {
    "Läuft unter PowerShell 7+ (Edition Core)"
} else {
    "Läuft unter Windows PowerShell (Edition Desktop)"
}
```

Die ersten beiden Zeilen lesen Version und Edition aus der automatischen Variablen `$PSVersionTable`. Die dritte Zeile ruft eine .NET-Klasse direkt auf und gibt aus, auf welcher .NET-Version die laufende PowerShell basiert, also beispielsweise „.NET Framework 4.x“ bei Windows PowerShell oder „.NET 10“ bei PowerShell 7.6. Der `if`-Block zeigt, wie ein Skript die Edition abfragt, um bei Bedarf unterschiedlich zu reagieren; Bedingungen behandelt Modul 4 ausführlich.

> **Ergänzung aus der Nachrecherche**
>
> **Aktueller Stand der Versionen (abgerufen aus der Microsoft-Dokumentation, Oktober 2026).** PowerShell 7 erscheint als *Long Term Servicing*-Version (LTS), die auf einer LTS-Version von .NET basiert und nur noch Sicherheits- und Servicing-Korrekturen erhält, sowie als *Stable*-Version dazwischen. Die aktuelle LTS-Version ist PowerShell 7.6 (veröffentlicht am 18. 03. 2026, unterstützt bis 14. 11. 2028, basierend auf .NET 10). PowerShell 7.4 (LTS) und PowerShell 7.5 (Stable) laufen am 10. 11. 2026 aus, also in wenigen Tagen; Systeme mit diesen Versionen sollten jetzt auf 7.6 aktualisiert werden. PowerShell 7.7 liegt als Vorschauversion vor und gehört nicht in Produktionsumgebungen. Das Supportende richtet sich nach dem .NET-Supportzeitraum der jeweiligen Version. Windows PowerShell 5.1 ist Teil von Windows und folgt dem Windows-Lebenszyklus. Konkrete Versionsnummern ändern sich laufend, prüfen Sie daher vor einer Entscheidung die Seite „PowerShell Support Lifecycle“ (siehe Quellenverzeichnis).

Beim Wechsel von 5.1 auf 7 sind die Unterschiede überschaubar, aber folgenreich. Die Sprache selbst ist nahezu identisch; PowerShell 7 ergänzt sie um praktische Operatoren, etwa den Null-Coalescing-Operator `??`, der den linken Wert liefert, sofern er nicht `$null` ist, und sonst den rechten, sowie den ternären Operator `Bedingung ? Wert1 : Wert2` und die Pipeline-Verkettung mit `&&` und `||`. Neue Cmdlets und Parameter erleichtern den Alltag, etwa `ForEach-Object -Parallel` für parallele Verarbeitung (Modul 7) oder `Get-Error` für vollständige Fehlerdetails. Die Standardansicht von Fehlermeldungen ist in PowerShell 7 die kompaktere *ConciseView*. Auf der anderen Seite wurden Funktionen entfernt, die auf Windows-spezifischen Technologien beruhen: PowerShell Workflow, Snap-Ins und die alten WMI-Cmdlets wie `Get-WmiObject` gibt es nicht mehr; stattdessen verwenden Sie die CIM-Cmdlets wie `Get-CimInstance`. Auch die Cmdlets `Get-EventLog` und `Write-EventLog` fehlen; Ereignisprotokolle lesen Sie mit `Get-WinEvent`.

Für Module, die nur in Windows PowerShell funktionieren, bietet PowerShell 7 unter Windows die *Windows PowerShell Compatibility*: Mit dem Schalter `-UseWindowsPowerShell` erzeugt `Import-Module` ein Stellvertretermodul, das die Befehle im Hintergrund in einem Windows-PowerShell-Prozess ausführt. Die Ergebnisse kommen dabei serialisiert zurück, es handelt sich also um Kopien ohne Methoden. Das genügt häufig, ist aber nicht in jedem Fall identisch mit der nativen Ausführung.

```powershell
# Modul, das nur unter Windows PowerShell läuft, aus PowerShell 7 heraus nutzen
Import-Module -Name ServerManager -UseWindowsPowerShell
Get-Module -Name ServerManager | Select-Object Name, ModuleType, PSEdition
```

Der erste Befehl lädt das (Windows-spezifische) Modul über die Kompatibilitätsschicht; der zweite zeigt, dass es als Stellvertreter geladen wurde. Dieses Beispiel funktioniert nur auf Windows-Servern, auf denen das Modul vorhanden ist, und eignet sich als Erkundungsübung dort.

Besonders heimtückisch sind Unterschiede im Verhalten von .NET-Methoden, weil PowerShell darauf direkt zugreift. Ein Beispiel aus der Dokumentation: In Windows PowerShell 5.1 konnte man der Methode `Split()` eine Zeichenkette übergeben, die als Menge einzelner Trennzeichen behandelt wurde. In PowerShell 7 gibt es zusätzliche Überladungen, die die Zeichenkette als *einen* Trenntext interpretieren.

```powershell
# In PowerShell 7 zerlegt dies NICHT an 'p' oder 'q':
"1111p2222q3333".Split('pq')

# RICHTIG: Zeichenfeld erzwingen, dann trennt PowerShell 7 wie 5.1
"1111p2222q3333".Split([char[]]'pq')
```

Das erste Beispiel liefert in PowerShell 7 die unveränderte Zeichenkette zurück, weil „pq“ als Ganzes nicht vorkommt. Durch das Umwandeln in ein Zeichenfeld (`[char[]]`) wählt PowerShell die Überladung, die jedes Zeichen einzeln als Trenner nutzt.

> **Merke**
>
> Beim Migrieren prüfen Sie drei Dinge: Welche Module werden gebraucht und sind sie mit Edition `Core` kompatibel? Welche entfernten Cmdlets (WMI v1, `Get-EventLog`, Workflow) kommen vor? Rufen Skripte .NET-Methoden direkt auf, deren Überladungen sich geändert haben könnten?

> **Praxisbeispiel**
>
> Auf einem Windows-Server startet ein geplanter Task nachts ein Backup-Skript mit `powershell.exe`. Wenn Sie das Skript für PowerShell 7 modernisieren, müssen Sie auch den Task-Aufruf auf `pwsh.exe` umstellen; sonst läuft weiterhin die alte Version. Üblich ist, das Skript zunächst in beiden Versionen zu testen und erst dann umzustellen. Die Planung dieser Migration vertieft Modul 12.

## 1.3 Objektpipeline-Grundidee

Die Objektpipeline ist der Kern von PowerShell und der wichtigste Unterschied zu klassischen Shells. In einer Unix-Shell verbindet das Pipe-Zeichen zwei Programme, indem es den Textstrom des einen an das nächste weitergibt. Jedes Programm muss die Struktur des Textes kennen, die eigentlich nirgends festgehalten ist: Wo endet eine Spalte, welche Zeile ist die Kopfzeile, wie sieht ein Datum aus? Ändert ein Programm sein Ausgabeformat, bricht die Auswertung. PowerShell löst das Problem, indem es zwischen den Befehlen *Objekte* weiterreicht. Ein Objekt ist eine strukturierte Dateneinheit eines bestimmten *Typs*. Es besitzt benannte *Eigenschaften* (Properties), also Werte wie Name, Größe oder Änderungsdatum, und *Methoden*, also Aktionen, die man mit dem Objekt ausführen kann, etwa Kopieren oder Löschen. Wenn `Get-ChildItem` Dateien auflistet, liefert es deshalb keine Textzeilen, sondern Objekte vom Typ `FileInfo` und `DirectoryInfo`.

Wie die Pipeline arbeitet, lässt sich an einem Beispiel gut beobachten. Das folgende Skript legt in einem temporären Ordner fünf Dateien unterschiedlicher Größe an, sortiert sie nach Größe und löscht sie wieder.

```powershell
$ordner = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'pipeline-demo'
New-Item -Path $ordner -ItemType Directory -Force | Out-Null

1..5 | ForEach-Object {
    $pfad = Join-Path -Path $ordner -ChildPath "datei$_.txt"
    Set-Content -Path $pfad -Value ('x' * ($_ * 100))
}

Get-ChildItem -Path $ordner |
    Sort-Object -Property Length -Descending |
    Select-Object -Property Name, Length, LastWriteTime

Remove-Item -Path $ordner -Recurse
```

Zunächst wird der Ordnerpfad zusammengesetzt und der Ordner angelegt; `Out-Null` verwirft die Ausgabe des Anlege-Befehls, damit sie das Ergebnis nicht stört. Der Ausdruck `1..5` erzeugt die Zahlen von eins bis fünf, die einzeln durch die Pipeline an `ForEach-Object` laufen. In dessen Skriptblock steht `$_` für die aktuelle Zahl. Innerhalb der Zeichenkette `"datei$_.txt"` wird die Variable ersetzt, sodass die Dateien `datei1.txt` bis `datei5.txt` entstehen; die Zeichenkette `'x' * ($_ * 100)` wiederholt das Zeichen entsprechend oft. Der zweite Teil ist der eigentliche Pipeline-Teil: `Get-ChildItem` liefert die Dateiobjekte, `Sort-Object` sortiert nach der Eigenschaft `Length` absteigend, und `Select-Object` wählt drei Eigenschaften aus. Nirgends wird ein Text zerlegt, denn jede Eigenschaft ist direkt ansprechbar.

Wichtig ist außerdem, *wann* Objekte fließen. Die Pipeline arbeitet im Streaming-Betrieb: Sobald das erste Objekt vom ersten Befehl erzeugt wird, kann der zweite es schon verarbeiten, ohne dass alle Objekte vorliegen müssen. Das spart Speicher bei großen Datenmengen. Sortieren ist die Ausnahme, denn `Sort-Object` muss alle Objekte kennen, bevor es das erste ausgeben kann. Wie ein nachfolgender Befehl ein ankommendes Objekt seinen Parametern zuordnet (*Parameter Binding*), erklärt Lerneinheit 2.7.

Um zu sehen, welche Eigenschaften und Methoden ein Objekt besitzt, verwenden Sie `Get-Member`. Das ist das wichtigste Erkundungswerkzeug überhaupt, dem Lerneinheit 2.3 einen eigenen Abschnitt widmet.

```powershell
Get-Item -Path $HOME | Get-Member -MemberType Property | Select-Object -First 8
(Get-Date).DayOfWeek
(Get-Date).AddDays(30).ToString('yyyy-MM-dd')
```

Die erste Zeile zeigt die ersten acht Eigenschaften des Verzeichnisobjekts. Die zweite greift auf die Eigenschaft `DayOfWeek` des aktuellen Datumsobjekts zu, die dritte ruft die Methode `AddDays` auf und formatiert das Ergebnis; Eigenschaften werden ohne, Methoden mit Klammern angesprochen.

> **Merke**
>
> Pipeline-Daten sind Objekte. Formatierende Befehle (`Format-Table`, `Format-List`) wandeln Objekte in reine Darstellungsobjekte um und gehören deshalb ans *Ende* einer Pipeline, nie in die Mitte.

Der häufigste Fehler im Zusammenhang mit der Pipeline besteht darin, nach `Format-Table` weiterverarbeiten zu wollen. Das Ergebnis wirkt zunächst unverständlich, weil danach nur noch Formatierungsobjekte ankommen.

```powershell
# FALSCH: Format-Table mitten in der Pipeline
Get-Process | Format-Table Name, Id | Export-Csv -Path prozesse.csv

# RICHTIG: Erst Daten auswählen, am Ende darstellen oder exportieren
Get-Process | Select-Object Name, Id | Export-Csv -Path prozesse.csv
```

Die CSV-Datei des ersten Beispiels enthält statt der erwarteten Spalten interne Eigenschaften der Formatierungsobjekte. Ein zweiter Stolperstein ist `Write-Host`: Dieser Befehl schreibt Text direkt auf den Bildschirm und gibt *keine* Objekte in die Pipeline. Daten, die später weiterverarbeitet werden sollen, gehören daher an die Ausgabe (`Write-Output` oder einfach der Ausdruck selbst).

> **Praxisbeispiel**
>
> In einem Inventarisierungsskript für einhundert Server liefern Sie pro Server ein Objekt mit den Eigenschaften Name, Betriebssystem, freier Speicher und letzter Neustart. Weil es echte Objekte sind, lassen sie sich ohne Zusatzaufwand sortieren, filtern und als CSV, JSON oder HTML-Bericht ausgeben. Der Bericht an die Geschäftsleitung ist dann nur noch ein weiterer Befehl am Ende der Pipeline.

## 1.4 Cmdlet-Namensschema

Die Befehle von PowerShell, die *Cmdlets* (gesprochen „Commandlets“), folgen einer strengen Namenskonvention: Ein *Verb* und ein *Substantiv*, durch einen Bindestrich getrennt, zum Beispiel `Get-Process`, `Stop-Service` oder `New-Item`. Das Verb beschreibt die Aktion, das Substantiv das Objekt, auf dem sie ausgeführt wird. Diese Regel hat einen großen praktischen Nutzen. Wer weiß, dass `Get-Service` Dienste abruft, kann mit sehr hoher Wahrscheinlichkeit erraten, dass es `Stop-Service`, `Start-Service` und `Restart-Service` ebenfalls gibt. Die Namen werden zu einer Art Sprache, die man sprechen kann, ohne jeden Befehl auswendig zu kennen.

Microsoft pflegt eine Liste *genehmigter Verben* („approved verbs“), die sich mit dem Cmdlet `Get-Verb` anzeigen lässt. Die Verben sind in Gruppen eingeteilt, etwa *Common* (allgemeine Aktionen wie `Get`, `Set`, `New`, `Remove`, `Add`, `Clear`), *Data* (`Import`, `Export`, `Convert`), *Lifecycle* (`Start`, `Stop`, `Enable`, `Disable`, `Install`), *Diagnostic* (`Test`, `Measure`, `Trace`), *Communications* (`Connect`, `Send`, `Receive`) und *Security* (`Grant`, `Block`, `Protect`). Die Bedeutung ist festgelegt: `Get` liest nur und verändert nichts, `Set` ändert einen vorhandenen Zustand, `New` legt etwas Neues an, `Remove` löscht, `Test` liefert in der Regel einen Wahrheitswert.

```powershell
Get-Verb | Group-Object -Property Group | Sort-Object -Property Count -Descending |
    Select-Object -Property Name, Count

Get-Verb | Where-Object -Property Verb -In 'Get', 'Create', 'Remove', 'Delete'
```

Die erste Pipeline gruppiert die genehmigten Verben nach ihrer Gruppe und zeigt, wie viele Verben jede Gruppe enthält. Die zweite fragt gezielt ab, ob vier Verben genehmigt sind. Das Ergebnis enthält nur `Get` und `Remove`: `Create` und `Delete` sind *nicht* genehmigt. Für „anlegen“ verwendet man `New`, für „löschen“ `Remove`.

Die Konvention gilt auch für Ihre eigenen Funktionen. Wenn Sie ein Modul mit nicht genehmigten Verben in den Namen von Befehlen importieren, gibt PowerShell eine Warnung aus; die Befehle funktionieren, wirken aber unprofessionell und erschweren das Auffinden. Das Substantiv soll im Singular stehen und möglichst spezifisch sein, bei eigenen Befehlen mit einem Präfix, das Kollisionen vermeidet.

```powershell
function Get-DiagnoseBericht {
    [CmdletBinding()]
    param()
    [pscustomobject]@{
        Computer = [System.Environment]::MachineName
        Zeit     = Get-Date
        Edition  = $PSVersionTable.PSEdition
    }
}
Get-DiagnoseBericht
```

Die Funktion folgt dem Schema: Das Verb `Get` ist genehmigt, das Substantiv `DiagnoseBericht` steht im Singular und ist beschreibend. Sie liefert ein selbst erzeugtes Objekt (`[pscustomobject]`) und liest nur, wie es sich für ein `Get`-Verb gehört. Die Attribute `[CmdletBinding()]` und `param()` machen aus der Funktion eine sogenannte *Advanced Function*, die sich wie ein Cmdlet verhält (Modul 4).

Zur Konvention gehört auch die Bedienung. Alle Cmdlets teilen sich *gemeinsame Parameter* wie `-Verbose`, `-ErrorAction` oder `-WhatIf`. Der Schalter `-WhatIf` ist besonders wertvoll: Er zeigt bei Befehlen, die etwas ändern, nur an, was passieren *würde*, ohne es auszuführen. Der Schalter `-Confirm` fragt vor der Ausführung nach.

```powershell
$datei = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'wichtig.txt'
Set-Content -Path $datei -Value 'Testinhalt'
Remove-Item -Path $datei -WhatIf
Remove-Item -Path $datei
```

Der dritte Befehl gibt eine Meldung aus, dass die Datei gelöscht *würde*, tut es aber nicht. Erst der vierte Befehl löscht tatsächlich. Dieses Muster, zuerst mit `-WhatIf` zu prüfen, sollten Sie sich bei allen Massenänderungen angewöhnen.

> **Merke**
>
> Verb-Substantiv, Verb aus `Get-Verb`, Substantiv im Singular. Die Namen verraten das Verhalten: `Get-` und `Test-` sind in der Regel unschädlich, `Set-`, `New-`, `Remove-` verändern den Zustand und unterstützen meist `-WhatIf`.

```powershell
# FALSCH: nicht genehmigtes Verb, Plural
function Delete-Users { }

# RICHTIG
function Remove-User { }
```

Im Gegenbeispiel würde `Import-Module` die Warnung über nicht genehmigte Verben ausgeben, und Kollegen würden vergeblich nach dem Gegenstück zu `New-User` suchen.

> **Praxisbeispiel**
>
> In Teams mit gemeinsamer Skriptbibliothek ist das Namensschema die einfachste Form der Qualitätssicherung. Bei einem Code-Review lässt sich „Dieses Verb ist nicht genehmigt“ ohne Diskussion prüfen, und Werkzeuge wie PSScriptAnalyzer (Modul 5) warnen automatisch vor Verstößen.

## 1.5 Host, Console und Terminal

Drei Begriffe werden im Alltag oft vermengt, bezeichnen aber verschiedene Schichten. Die *PowerShell-Engine* ist das Programmteil, das Ihre Befehle versteht und ausführt. Ein *Host* ist die Anwendung, die die Engine einbettet und ihr Ein- und Ausgabe bereitstellt. Die *Konsole* ist die textbasierte Oberfläche, in der Eingaben erfolgen und Ausgaben erscheinen. Ein *Terminal* (oder Terminalemulator) ist ein Programm, das eine solche Konsole darstellt und Funktionen wie Tabs, Farben und Schriftarten bietet. Beim Start von `pwsh` in einem Fenster arbeiten alle Schichten zusammen: Das Terminal zeichnet das Fenster, im Fenster läuft der Konsolen-Host, und im Host arbeitet die Engine.

Wichtig ist diese Unterscheidung, weil sich Verhalten je nach Host unterscheidet. `Read-Host` oder `Out-GridView` setzen einen interaktiven Host voraus; in einer Pipeline eines Build-Servers oder in Azure Automation gibt es keinen Benutzer, der Eingaben tippt. Auch Farben und Fortschrittsanzeigen hängen vom Host ab. Den Namen des Hosts erfahren Sie über die Variable `$Host`.

```powershell
$Host.Name
$Host.Version
$Host.UI.RawUI.WindowSize
```

Im normalen Konsolenfenster liefert `$Host.Name` den Wert `ConsoleHost`, in der Erweiterung für Visual Studio Code erscheint ein anderer Hostname. Die dritte Zeile zeigt die Fenstergröße, die es nur in Hosts mit echter Konsole gibt; in einer Pipeline ohne Konsole würde sie einen Fehler auslösen. Skripte, die in verschiedenen Hosts laufen sollen, dürfen deshalb nicht von Fenstergröße oder interaktiven Eingaben abhängen.

Unter Windows haben Sie mehrere Möglichkeiten für die Arbeitsumgebung. Das klassische Konsolenfenster ist funktional, aber schlicht. *Windows Terminal* ist ein moderner Terminalemulator mit Tabs, Profilen für verschiedene Shells (PowerShell 7, Windows PowerShell, Eingabeaufforderung, WSL) und guter Unicode-Unterstützung. Die *Windows PowerShell ISE* (Integrated Scripting Environment) ist eine ältere Entwicklungsumgebung, die nur mit Windows PowerShell 5.1 funktioniert und nicht mehr weiterentwickelt wird. Als Standardwerkzeug zum Skripten empfiehlt sich heute *Visual Studio Code* mit der *PowerShell-Erweiterung*: Sie bietet Syntaxhervorhebung, Code-Vervollständigung, einen Debugger (Modul 5) und ein integriertes Terminal; sie läuft auf Windows, Linux und macOS und unterstützt beide Editionen.

Zur Arbeitsumgebung gehört auch das *Profil*, ein Skript, das beim Start einer interaktiven Sitzung automatisch ausgeführt wird. Dort stellen Sie Voreinstellungen ein, etwa die Vorhersagefunktion der Befehlszeile oder eigene Hilfsfunktionen. Es gibt vier Profile, die sich nach Gültigkeit für Benutzer oder alle Benutzer und für einen oder alle Hosts unterscheiden. Ihre Pfade liefert die Variable `$PROFILE`.

```powershell
$PROFILE | Select-Object AllUsersAllHosts, AllUsersCurrentHost,
    CurrentUserAllHosts, CurrentUserCurrentHost | Format-List

$meinProfil = $PROFILE.CurrentUserCurrentHost
if (-not (Test-Path -Path $meinProfil)) {
    New-Item -Path $meinProfil -ItemType File -Force | Out-Null
}
Add-Content -Path $meinProfil -Value 'Set-PSReadLineOption -PredictionSource History'
```

Die Variable `$PROFILE` ist eine Zeichenkette für das Profil des aktuellen Benutzers und aktuellen Hosts, trägt aber zusätzliche Eigenschaften für die übrigen drei Profile. Das Beispiel gibt alle vier Pfade aus, legt die Profildatei samt fehlender Ordner an, falls sie nicht existiert (`-Force` erzeugt Zwischenordner), und ergänzt eine Zeile, die vorgeschlagene Eingaben aus dem Befehlsverlauf anzeigt. Beachten Sie, dass dieses Beispiel Ihr persönliches Profil ändert; probieren Sie es bewusst aus.

> **Merke**
>
> Ein Skript darf nie davon ausgehen, dass ein bestimmtes Profil geladen ist. Profile gelten nur für interaktive Sitzungen des jeweiligen Benutzers. Automatisierte Aufrufe sollten mit `pwsh -NoProfile` gestartet werden, damit sie überall gleich laufen.

```powershell
# FALSCH: Skript verlässt sich auf eine im Profil definierte Funktion
Get-MeineHilfsfunktion -Name 'Server01'

# RICHTIG: Abhängigkeit im Skript selbst herstellen
. "$PSScriptRoot\Hilfsfunktionen.ps1"
Get-MeineHilfsfunktion -Name 'Server01'
```

Im ersten Fall funktioniert das Skript nur auf dem Rechner, auf dem das Profil die Funktion definiert, und schlägt auf dem Build-Server fehl. Im zweiten Fall lädt das Skript seine Abhängigkeit selbst; der Punkt am Anfang (*Dot-Sourcing*) führt die Datei im aktuellen Gültigkeitsbereich aus, und `$PSScriptRoot` ist der Ordner des laufenden Skripts. Das Beispiel nutzt einen Windows-Pfadtrenner; plattformübergreifend wäre `Join-Path` vorzuziehen (Lerneinheit 1.7).

> **Praxisbeispiel**
>
> Wenn ein Skript auf Ihrem Laptop funktioniert, im geplanten Task aber nicht, liegt es überraschend oft an Profil, Host oder fehlender Umgebung. Wer Skripte von Anfang an für einen Host ohne Benutzer entwirft, erspart sich viel Fehlersuche.

## 1.6 Execution Model

Um zu verstehen, was PowerShell tut, wenn Sie die Eingabetaste drücken, hilft ein einfaches Bild der Verarbeitung. Zuerst *parst* (analysiert) PowerShell den Text: Es zerlegt ihn in Bestandteile wie Befehlsnamen, Parameter, Operatoren und Zeichenketten und prüft die Syntax. Dabei unterscheidet es zwei Modi. Im *Argumentmodus*, der mit einem Befehlsnamen beginnt (`Get-Process notepad`), werden die Wörter als Zeichenketten behandelt, sofern sie nicht mit `$` oder Klammern beginnen. Im *Ausdrucksmodus*, der bei Zahlen, Variablen oder Klammern beginnt (`2 + 3`), gelten die Regeln einer klassischen Programmiersprache. Das erklärt, warum `Write-Output 2+3` den Text „2+3“ ausgibt, `Write-Output (2+3)` aber die Zahl fünf.

Danach wird der Befehlsname aufgelöst (*Command Discovery*). PowerShell sucht in einer festen Reihenfolge: zuerst nach einem *Alias*, dann nach einer *Funktion*, danach nach einem *Cmdlet* und zuletzt nach einer externen Anwendung oder einem Skript im Suchpfad. Der erste Treffer gewinnt. Damit lässt sich auch erklären, warum ein eigenes Alias oder eine eigene Funktion den Namen eines Cmdlets überlagern kann, eine häufige Fehlerquelle bei geerbten Skripten.

```powershell
function Get-Uhrzeit { 'Ich bin die Funktion' }
Set-Alias -Name Get-Uhrzeit -Value Get-Date

Get-Command -Name Get-Uhrzeit -All | Select-Object CommandType, Name
Get-Uhrzeit

Remove-Item -Path Alias:Get-Uhrzeit
Get-Uhrzeit
```

Zuerst entsteht eine Funktion, dann ein gleichnamiges Alias, das auf `Get-Date` zeigt. `Get-Command -All` listet beide Treffer, und der Aufruf liefert das Datum, weil das Alias Vorrang hat. Nach dem Entfernen des Alias (über das Laufwerk `Alias:`, das Modul 2 als Provider erklärt) greift die Funktion.

Anschließend erfolgt das *Parameter Binding*: PowerShell ordnet Argumente den Parametern des Befehls zu. Danach wird der Befehl in drei Phasen ausgeführt, die in Funktionen als Blöcke `begin`, `process` und `end` sichtbar werden: Einmal zu Beginn, einmal pro Pipeline-Objekt und einmal am Ende. Aus diesem Grund kann ein Befehl mitten in einer Pipeline schon Ergebnisse weitergeben, während der Vorgänger noch arbeitet.

```powershell
function Show-Phasen {
    [CmdletBinding()]
    param([Parameter(ValueFromPipeline)] $Wert)
    begin   { 'begin' }
    process { "process: $Wert" }
    end     { 'end' }
}
1..3 | Show-Phasen
```

Die Ausgabe lautet: `begin`, dreimal `process` mit den Werten 1, 2, 3 und schließlich `end`. Der Parameter nimmt Werte aus der Pipeline entgegen (`ValueFromPipeline`); Details folgen in Modul 4.

PowerShell kennt nicht nur einen Ausgabekanal, sondern mehrere *Streams*, die durch Nummern angesprochen werden: 1 (Success, die eigentlichen Ergebnisse), 2 (Error), 3 (Warning), 4 (Verbose), 5 (Debug) und 6 (Information). Die Trennung ist entscheidend, weil Ergebnisse weiterverarbeitet werden, Meldungen aber nicht in die Daten geraten dürfen. Mit Umleitungsoperatoren wie `2>$null` lässt sich jeder Strom einzeln unterdrücken oder in eine Datei leiten.

```powershell
function Test-Streams {
    Write-Output 'Erfolg'
    Write-Warning 'Warnung'
    Write-Verbose 'Details' -Verbose
    Write-Information 'Info' -InformationAction Continue
    Write-Error 'Fehler'
}
$nurErfolg = Test-Streams 2>$null 3>$null 4>$null 6>$null
$nurErfolg
```

Die Funktion schreibt in fünf verschiedene Ströme. Beim Aufruf werden Fehler-, Warn-, Verbose- und Informationsstrom verworfen, in `$nurErfolg` landet nur der Text „Erfolg“. Zu dieser Trennung kehrt Modul 9 zurück.

Zuletzt entsteht der Rückgabewert. Bei PowerShell-Befehlen gibt es einen booleschen Erfolgsindikator in `$?`, bei externen Programmen den *Exit Code* in `$LASTEXITCODE`. Ein Exit Code von 0 bedeutet nach Konvention Erfolg, jeder andere Wert einen Fehler. Dass ein Programm etwas auf den Fehlerstrom schreibt, sagt dabei nichts über den Erfolg aus: In PowerShell 7 wird `$?` nur dann auf `$false` gesetzt, wenn ein externes Programm einen von Null verschiedenen Exit Code liefert.

```powershell
pwsh -NoProfile -Command 'exit 3'
$?
$LASTEXITCODE
```

Hier startet PowerShell eine zweite Sitzung, die mit dem Code 3 endet; `$?` liefert direkt danach `False`, und `$LASTEXITCODE` liefert 3. Die Reihenfolge ist wichtig, denn jede ausgeführte Anweisung setzt `$?` neu: Würde `$LASTEXITCODE` zuerst abgefragt, zeigte `$?` anschließend `True`, weil diese Abfrage selbst erfolgreich war. `$LASTEXITCODE` behält seinen Wert dagegen, bis das nächste externe Programm läuft. Unter Windows PowerShell ersetzen Sie `pwsh` durch `powershell`.

> **Merke**
>
> Reihenfolge: Parsen, Befehl auflösen (Alias, Funktion, Cmdlet, externe Anwendung), Parameter binden, Pipeline verarbeiten, Ergebnis und Status zurückgeben. Ergebnisse gehören in den Success-Stream, Meldungen in die übrigen Ströme.

```powershell
# FALSCH: Meldung als Ausgabe, landet in den Daten
function Get-Zahl { 'Berechne ...'; 42 }
$x = Get-Zahl      # $x enthält zwei Werte!

# RICHTIG: Meldung in den Verbose-Stream
function Get-Zahl { Write-Verbose 'Berechne ...'; 42 }
$x = Get-Zahl      # $x ist 42
```

Dieser Fehler gehört zu den häufigsten überhaupt: Jede nicht abgefangene Ausgabe in einer Funktion wird Teil des Rückgabewerts. Wer das nicht weiß, sucht lange nach Werten, die „zu viel“ sind.

> **Praxisbeispiel**
>
> In einer CI/CD-Pipeline entscheidet der Exit Code des Skripts, ob der Build als erfolgreich oder fehlgeschlagen gilt. Ein Skript, das Fehler nur ausgibt, aber mit Code 0 endet, lässt die Pipeline fälschlich grün erscheinen. Die systematische Behandlung folgt in Modul 9.

## 1.7 Cross-Plattform-Grundlagen

Seit PowerShell 6 läuft dieselbe Shell auf Windows, Linux und macOS. Für Administratoren eröffnet das die Möglichkeit, eine einzige Sprache für gemischte Umgebungen zu verwenden; für Skriptautoren bedeutet es, dass sie plattformspezifische Annahmen sichtbar machen müssen. Die Sprache und die meisten Kerncmdlets verhalten sich überall gleich. Unterschiede entstehen vor allem durch das Betriebssystem: Dateisysteme, Umgebungsvariablen, Zeichenkodierung, verfügbare Module und Programme.

Zur Erkennung der Plattform stellt PowerShell 7 die automatischen Variablen `$IsWindows`, `$IsLinux` und `$IsMacOS` bereit. In Windows PowerShell 5.1 existieren sie nicht; dort ergibt der Zugriff den Wert `$null` und damit „falsch“, sofern nicht der *Strict Mode* aktiv ist, der den Zugriff auf nicht definierte Variablen als Fehler meldet (Modul 5).

```powershell
$os = if ($IsLinux) { 'Linux' }
      elseif ($IsMacOS) { 'macOS' }
      else { 'Windows' }
"Betriebssystem: $os auf $([System.Environment]::MachineName)"

$konfig = Join-Path -Path $HOME -ChildPath 'meine-app' |
    Join-Path -ChildPath 'config.json'
$konfig
[System.IO.Path]::DirectorySeparatorChar
```

Die Plattformprüfung ist so formuliert, dass sie auch unter 5.1 zu „Windows“ führt: Weder `$IsLinux` noch `$IsMacOS` sind dort gesetzt, daher greift der `else`-Zweig. Der Pfad wird mit `Join-Path` zusammengesetzt, sodass je nach Plattform der richtige Trenner (Rückwärtsschrägstrich unter Windows, Schrägstrich unter Linux und macOS) verwendet wird. Die Verkettung über die Pipeline ist absichtlich gewählt: Die Variante, mehrere Teilpfade in einem Aufruf anzugeben, gibt es nur in PowerShell 7. Die Variable `$HOME` funktioniert überall, während `$env:USERPROFILE` nur unter Windows gesetzt ist und `$env:HOME` unter Unix.

Das Dateisystem ist die nächste Stolperfalle. Unter Linux unterscheiden die üblichen Dateisysteme Groß- und Kleinschreibung: `Daten.txt` und `daten.txt` sind zwei verschiedene Dateien. Unter Windows und meist auch unter macOS sind sie dieselbe. PowerShell selbst ist bei Cmdlet- und Variablennamen unempfindlich gegenüber der Schreibweise, bei Pfaden hängt das Verhalten vom Betriebssystem ab. Pfadangaben mit festem Laufwerksbuchstaben (`C:`) gehören nicht in plattformübergreifende Skripte. Auch Zeilenenden unterscheiden sich (Windows: CR+LF, Unix: LF); wer Skripte mit Git zwischen Plattformen tauscht, sollte das über Git-Attribute vereinheitlichen.

Die Zeichenkodierung ist ein weiteres bekanntes Problem. Windows PowerShell schreibt Dateien je nach Cmdlet in verschiedenen Kodierungen, die für Umlaute und Sonderzeichen zu Fehlern führen können. PowerShell 7 nutzt als Standard UTF-8 ohne Byte-Order-Mark (BOM) und ändert in diesem Zug auch den Standard von `$OutputEncoding`. Wer sicher sein will, gibt die Kodierung explizit an.

```powershell
$datei = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'umlaute.txt'
Set-Content -Path $datei -Value 'Grüße aus Österreich' -Encoding utf8
Get-Content -Path $datei -Encoding utf8
```

Mit der expliziten Angabe `-Encoding utf8` verhalten sich beide Editionen vergleichbar; in 5.1 wird allerdings eine BOM geschrieben, in 7 nicht. Tools, die keine BOM erwarten, sind das in der Regel zufrieden; solche, die eine erwarten, nicht.

> **Ergänzung aus der Nachrecherche**
>
> Laut Microsoft-Dokumentation unterstützen PowerShell 7 offiziell Windows, bestimmte Versionen von Debian, Ubuntu, Red Hat Enterprise Linux und Alpine Linux sowie macOS; Zwischenversionen von Linux-Distributionen ohne Langzeitunterstützung (etwa Ubuntu-Zwischenreleases) testet Microsoft nicht. Die genaue Liste ändert sich mit jedem Release, prüfen Sie sie vor einer Einführung auf der Seite „PowerShell Support Lifecycle“.

Weitere Unterschiede betreffen die Verfügbarkeit. Viele Cmdlets, die auf Windows-Techniken wie Dienste, Registrierung, WMI oder Ereignisprotokolle aufsetzen, gibt es nur unter Windows. Module wie `ActiveDirectory` sind Windows-spezifisch. Auf Linux und macOS sind außerdem einige Aliase nicht definiert, die unter Windows existieren (etwa für `ls`, `cat` oder `ps`), damit die gleichnamigen nativen Programme aufgerufen werden. Die Remoting-Technik WinRM steht nur unter Windows zur Verfügung; plattformübergreifend wird SSH verwendet (Modul 7).

> **Merke**
>
> Plattformübergreifende Skripte vermeiden harte Pfade und Trenner (`Join-Path`), prüfen die Plattform mit `$IsWindows` usw., geben Kodierungen explizit an und kapseln Windows-spezifische Befehle in Abfragen.

```powershell
# FALSCH: harter Windows-Pfad und Windows-only-Cmdlet ohne Prüfung
Get-Content 'C:\Temp\daten.txt'
Get-Service -Name ssh

# RICHTIG: Plattform prüfen, Pfad zusammensetzen
$pfad = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'daten.txt'
if ($IsWindows -or $PSVersionTable.PSEdition -eq 'Desktop') {
    Get-Service -Name 'Dhcp' -ErrorAction SilentlyContinue
}
```

> **Praxisbeispiel**
>
> Ein Team nutzt Windows-Laptops zur Entwicklung, aber Linux-Container als Build-Agenten. Ein Skript, das auf dem Laptop lief, schlägt im Container fehl, weil es mit `C:\` arbeitet oder Dienste abfragt. Wer von Anfang an plattformneutral schreibt und die Skripte im Container testet, findet solche Probleme früh.

## 1.8 Praxisumgebung

Zum Abschluss des Moduls richten Sie eine Arbeitsumgebung ein, mit der Sie alle folgenden Module bearbeiten können. Sie besteht aus drei Bausteinen: PowerShell 7, Visual Studio Code mit der PowerShell-Erweiterung und Git. Die Installation hängt vom Betriebssystem ab. Unter Windows genügt in der Regel der Paketmanager *winget*; unter macOS kann Homebrew verwendet werden; unter Linux stellt Microsoft Paketquellen für gängige Distributionen bereit. Die aktuellen Anleitungen stehen in der PowerShell-Dokumentation.

```text
# Windows
winget install --id Microsoft.PowerShell
winget install --id Microsoft.VisualStudioCode
winget install --id Git.Git

# macOS (Homebrew)
brew install --cask powershell
```

Die Befehle installieren PowerShell 7, Visual Studio Code und Git. Nach der Installation von VS Code öffnen Sie das Erweiterungsfenster und installieren die Erweiterung *PowerShell* von Microsoft. Beim Öffnen einer `.ps1`-Datei startet sie ein integriertes Terminal mit PowerShell 7. Prüfen Sie nach dem Start immer, welche Version tatsächlich läuft: Wer `powershell` statt `pwsh` eingibt, erhält unter Windows die alte Version.

Git ist ein Versionsverwaltungssystem, das jede Änderung an Ihren Skripten nachvollziehbar speichert. Es ist heute auch für Administratoren unverzichtbar: Ohne Versionsverwaltung ist unklar, wer wann welche Änderung am Skript vorgenommen hat und wie man zum letzten funktionierenden Stand zurückkehrt. Die ersten Schritte bestehen aus Name und E-Mail-Adresse, einem Repository und einer Ausschlussliste für Dateien, die nicht ins Repository gehören.

```text
git config --global user.name "Ihr Name"
git config --global user.email "sie@beispiel.at"
git init powershell-kurs
cd powershell-kurs
"*.log" | Set-Content .gitignore
git add .
git commit -m "Erste Version"
```

Die Befehle konfigurieren die Identität, legen den Ordner `powershell-kurs` als Repository an, erzeugen eine `.gitignore`-Datei, die Protokolldateien ausschließt, und speichern den ersten Stand. Dass der Befehl `cd` auch in PowerShell funktioniert, liegt an einem Alias für `Set-Location`. Niemals gehören Kennwörter oder Zugriffsschlüssel in ein Repository (Modul 8).

Anschließend aktualisieren Sie das Hilfesystem. PowerShell liefert nur eine Grundhilfe mit; die ausführlichen Hilfetexte lädt `Update-Help` aus dem Internet. In PowerShell 7 benötigt dies keine Administratorrechte, weil die Hilfe standardmäßig benutzerbezogen gespeichert wird; in Windows PowerShell sind Administratorrechte erforderlich. Der Befehl ist standardmäßig auf einen Lauf pro 24 Stunden begrenzt. Danach führen Sie einige Diagnosebefehle aus.

```powershell
Update-Help -ErrorAction SilentlyContinue

[pscustomobject]@{
    Version        = $PSVersionTable.PSVersion.ToString()
    Edition        = $PSVersionTable.PSEdition
    Host           = $Host.Name
    Plattform      = [System.Runtime.InteropServices.RuntimeInformation]::OSDescription
    Profil         = $PROFILE.CurrentUserCurrentHost
    ModulPfade     = ($env:PSModulePath -split [System.IO.Path]::PathSeparator) -join '; '
    Module         = (Get-Module -ListAvailable | Measure-Object).Count
}
```

`Update-Help` lädt die Hilfedateien; mit `-ErrorAction SilentlyContinue` werden Fehler bei Modulen ohne Hilfe unterdrückt, was für den ersten Lauf bequem ist. Das erzeugte Objekt fasst die wichtigsten Umgebungsdaten zusammen. Der Ausdruck mit `-split` zerlegt die Variable `PSModulePath`, eine Liste von Ordnern, am plattformabhängigen Trennzeichen (Semikolon unter Windows, Doppelpunkt unter Linux und macOS); `-join` setzt sie lesbar wieder zusammen. `Get-Module -ListAvailable` zählt alle installierten Module. Speichern Sie dieses Skript als `Get-Umgebung.ps1`; es ist ein guter erster Eintrag in Ihr Repository und ein praktisches Hilfsmittel, wenn jemand Sie später fragt, „wie es bei Ihnen läuft“.

> **Merke**
>
> Eine gute Arbeitsumgebung besteht aus aktueller PowerShell 7, einem Editor mit Debugger, Versionsverwaltung und aktualisierter Hilfe. Prüfen Sie am Anfang jeder Sitzung, welche Version und welcher Host aktiv sind.

```powershell
# FALSCH: Skript in Windows PowerShell testen, in PowerShell 7 produktiv einsetzen
powershell -File .\meinSkript.ps1

# RICHTIG: Test in der Zielumgebung (und bei Bedarf in beiden Editionen)
pwsh -NoProfile -File ./meinSkript.ps1
```

Das falsche Vorgehen führt zu Überraschungen, weil sich Verhalten zwischen den Editionen unterscheidet (Lerneinheit 1.2). Testen Sie, wo das Skript später läuft.

> **Praxisbeispiel**
>
> Viele Teams beschreiben die Umgebung ihrer Skripte in einem Container-Image oder einem Entwicklungscontainer, damit jedes Teammitglied und der Build-Server exakt dieselben Versionen verwenden. Die manuelle Einrichtung aus dieser Lerneinheit ist der erste Schritt dorthin; die Automatisierung folgt in Modul 11.

## 1.9 Übungsaufgaben zu Modul 01

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Nennen Sie die drei Rollen, die PowerShell gleichzeitig einnimmt, und ordnen Sie je ein Beispiel aus der Praxis zu.

**Aufgabe 2.** Welche Datei startet Windows PowerShell, welche PowerShell 7? Wie stellen Sie innerhalb einer laufenden Sitzung fest, welche Edition aktiv ist?

**Aufgabe 3.** Erklären Sie in zwei bis drei Sätzen, warum die Objektpipeline robuster ist als eine Textpipeline, und nennen Sie ein Beispiel für einen typischen Fehler beim Einsatz von `Format-Table` in einer Pipeline.

**Aufgabe 4.** In welcher Reihenfolge sucht PowerShell nach einem Befehlsnamen? Was passiert, wenn ein Alias und eine Funktion denselben Namen tragen?

**Aufgabe 5.** Schreiben Sie eine Pipeline, die die drei größten Dateien im temporären Verzeichnis (nicht rekursiv) mit Name und Größe in Kilobyte (eine Nachkommastelle) ausgibt.

**Aufgabe 6.** Prüfen Sie mit `Get-Verb`, ob die Verben `Fetch`, `Find`, `Create` und `Update` genehmigt sind. Schreiben Sie das Ergebnis als Liste mit je einem Eintrag `Verb` und `Genehmigt` (`True`/`False`).

**Aufgabe 7.** Schreiben Sie eine Funktion `Get-PlattformInfo`, die ein Objekt mit den Eigenschaften `Betriebssystem` (Windows, Linux oder macOS), `Edition`, `Version` und `Temp` (temporäres Verzeichnis) liefert und unter 5.1 und 7 läuft.

**Aufgabe 8.** Schreiben Sie ein Skript, das einen Ordner `uebung01` im temporären Verzeichnis anlegt, dort drei Textdateien mit UTF-8-Kodierung und deutschen Umlauten erzeugt, die Dateien mit Name und Länge als CSV in eine vierte Datei exportiert und die CSV-Datei danach einliest und anzeigt. Nutzen Sie `-WhatIf`, bevor Sie den Ordner abschließend löschen.

## 1.10 Lösungen zu Modul 01

### Aufgabe 1.

PowerShell ist Shell (interaktive Eingabe, z. B. schnelle Prüfung des Dienststatus), Skriptsprache (wiederholbare `.ps1`-Dateien, z. B. Onboarding eines Benutzers) und Automationsplattform (Module, Remoting, Cloud-Anbindung, z. B. Verwaltung von Azure-Ressourcen). Entscheidend ist, dass alle drei Rollen dieselben Befehle verwenden: Was Sie interaktiv ausprobieren, übernehmen Sie unverändert in ein Skript.

### Aufgabe 2.

Windows PowerShell startet mit `powershell.exe`, PowerShell 7 mit `pwsh.exe` (`pwsh` unter Linux und macOS). Die Edition liefert `$PSVersionTable.PSEdition`: `Desktop` für Windows PowerShell, `Core` für PowerShell 7. Die Versionsnummer steht in `$PSVersionTable.PSVersion`.

### Aufgabe 3.

Textpipelines sind abhängig von Ausgabeformat, Spaltenbreite und Sprache, denn die Struktur ist nur implizit vorhanden. Objekte tragen ihre Struktur (Eigenschaften, Typen) selbst mit sich und lassen sich direkt ansprechen. Fehler: `Format-Table` wandelt die Objekte in Darstellungsobjekte um; ein anschließendes `Export-Csv` würde nicht die gewünschten Spalten schreiben. Formatierung gehört ans Ende.

### Aufgabe 4.

Reihenfolge: Alias, Funktion, Cmdlet, externe Anwendung beziehungsweise Skript. Bei gleichem Namen gewinnt das Alias. Mit `Get-Command -Name <Name> -All` sehen Sie alle Kandidaten und ihre Reihenfolge.

### Aufgabe 5.

```powershell
Get-ChildItem -Path ([System.IO.Path]::GetTempPath()) -File |
    Sort-Object -Property Length -Descending |
    Select-Object -First 3 -Property Name,
        @{ Name = 'GroesseKB'; Expression = { [math]::Round($_.Length / 1KB, 1) } }
```

`-File` beschränkt auf Dateien, `Sort-Object` sortiert absteigend nach `Length`, `Select-Object` behält die ersten drei und ergänzt die berechnete Eigenschaft. `1KB` ist 1024. Gibt es im Verzeichnis weniger als drei Dateien, werden entsprechend weniger angezeigt.

### Aufgabe 6.

```powershell
$verben = 'Fetch', 'Find', 'Create', 'Update'
$genehmigt = (Get-Verb).Verb
$verben | ForEach-Object {
    [pscustomobject]@{
        Verb       = $_
        Genehmigt  = $genehmigt -contains $_
    }
}
```

`(Get-Verb).Verb` liefert die Namen aller genehmigten Verben. `-contains` prüft, ob die Liste das Verb enthält. Erwartetes Ergebnis: `Find` und `Update` sind genehmigt, `Fetch` und `Create` nicht. Der Vergleich ist unabhängig von der Groß- und Kleinschreibung.

### Aufgabe 7.

```powershell
function Get-PlattformInfo {
    [CmdletBinding()]
    param()

    $os = if ($IsLinux) { 'Linux' }
          elseif ($IsMacOS) { 'macOS' }
          else { 'Windows' }

    [pscustomobject]@{
        Betriebssystem = $os
        Edition        = $PSVersionTable.PSEdition
        Version        = $PSVersionTable.PSVersion.ToString()
        Temp           = [System.IO.Path]::GetTempPath()
    }
}
Get-PlattformInfo
```

Unter 5.1 sind `$IsLinux` und `$IsMacOS` nicht definiert, daher greift der `else`-Zweig und liefert Windows. Das Verb `Get` ist genehmigt, die Funktion liest nur und liefert ein Objekt statt Text.

### Aufgabe 8.

```powershell
$basis = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'uebung01'
New-Item -Path $basis -ItemType Directory -Force | Out-Null

'Grüße', 'Äpfel und Öl', 'Straße' | ForEach-Object -Begin { $i = 0 } -Process {
    $i++
    Set-Content -Path (Join-Path $basis "text$i.txt") -Value $_ -Encoding utf8
}

$bericht = Join-Path -Path $basis -ChildPath 'bericht.csv'
Get-ChildItem -Path $basis -Filter 'text*.txt' |
    Select-Object -Property Name, Length |
    Export-Csv -Path $bericht -Encoding utf8 -NoTypeInformation

Import-Csv -Path $bericht | Format-Table -AutoSize

Remove-Item -Path $basis -Recurse -WhatIf
Remove-Item -Path $basis -Recurse
```

Die Schleife nutzt `-Begin`, um den Zähler einmal zu initialisieren, und `-Process` für jedes Element (Phasen aus Lerneinheit 1.6). Der Export enthält nur Name und Länge, und `Import-Csv` liest die Datei wieder als Objekte ein; `Format-Table` steht am Ende. `-WhatIf` zeigt die Löschung nur an, erst der zweite Aufruf löscht. Die Länge einer Datei in Byte hängt von der Kodierung ab: Umlaute belegen in UTF-8 zwei Byte.
