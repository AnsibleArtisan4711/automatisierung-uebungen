# Modul 02 - Cmdlets, Hilfe, Module und Provider

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (2.1 bis 2.9; 2.10 Übungsaufgaben, 2.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Übersichtsblatt (10 h) und Detailblatt (9 h) weichen um eine Stunde voneinander ab. Dieses Skript folgt dem Detailblatt, weil jede Lerneinheit dort einzeln aufgeführt ist.

### Lernziele.

Nach diesem Modul können Sie PowerShell-Werkzeuge sicher finden, laden und anwenden: Sie lesen und aktualisieren die Hilfe, entdecken Befehle systematisch, untersuchen Objekte mit `Get-Member`, verwalten Module aus der PowerShell Gallery, navigieren mit Providern durch Datenspeicher, setzen Aliase bewusst ein, verstehen die Parameterbindung, gehen kontrolliert mit Profilen und der Umgebung um und dokumentieren eigene Befehle nach Standard. Sie ordnen dabei Sicherheitsaspekte und Betriebspraxis ein und bereiten wiederholbare Automation für Windows, Cloud und CI/CD vor.

| **Nr.** | **Lerneinheit**                   | **Dauer** | **Prio** |
|:--------|:----------------------------------|:----------|:---------|
| 2.1     | Get-Help und Update-Help          | 1 h       | 1        |
| 2.2     | Get-Command und Command Discovery | 1 h       | 1        |
| 2.3     | Get-Member                        | 1 h       | 1        |
| 2.4     | Module Management                 | 1 h       | 1        |
| 2.5     | Provider und PSDrives             | 1 h       | 1        |
| 2.6     | Aliase und Lesbarkeit             | 1 h       | 1        |
| 2.7     | Parameter Binding                 | 1 h       | 2        |
| 2.8     | Profiles und Umgebung             | 1 h       | 2        |
| 2.9     | Dokumentationsstandards           | 1 h       | 2        |
|         | **Summe**                         | **9 h**   |          |

## 2.1 Get-Help und Update-Help

PowerShell besitzt mehr als tausend Befehle, und niemand kennt sie auswendig. Das ist kein Problem, weil die Shell ein eingebautes Nachschlagewerk mitbringt: das *Hilfesystem*. Es beantwortet drei Fragen, die im Alltag ständig auftauchen. Was macht dieser Befehl? Welche Parameter akzeptiert er, und welche sind Pflicht? Wie sieht ein funktionierender Aufruf aus? Wer das Hilfesystem sicher bedient, ist bei neuen Befehlen und Modulen nie hilflos und muss nicht ständig im Browser suchen.

Der zentrale Befehl heißt `Get-Help`. Ohne weitere Angaben liefert er zu einem Befehl Name, Kurzbeschreibung, Syntax und Hinweise. Mit Schaltern steuern Sie die Ausführlichkeit: `-Examples` zeigt nur Beispiele, `-Detailed` ergänzt Parameterbeschreibungen und Beispiele, `-Full` zeigt alles einschließlich Eingabe- und Ausgabetypen, und `-Parameter` gibt die Beschreibung eines einzelnen Parameters aus. Mit `-Online` öffnet sich die Webversion im Browser, und `-ShowWindow` zeigt die Hilfe in einem eigenen Fenster (nur auf Windows). Neben den Hilfetexten einzelner Befehle gibt es konzeptionelle Hilfethemen, die mit `about_` beginnen, etwa `about_Pipelines` oder `about_Operators`. Auch Platzhalter funktionieren: `Get-Help *hash*` listet alle Hilfethemen, deren Name „hash“ enthält.

```powershell
Get-Help -Name Get-ChildItem -Parameter Path
Get-Help -Name Get-ChildItem -Parameter * |
    Select-Object -Property Name, Required, Position, PipelineInput |
    Format-Table -AutoSize
(Get-Help -Name Get-Process).Syntax.syntaxItem.Count
```

Die erste Zeile zeigt die Beschreibung des Parameters `Path` von `Get-ChildItem`. Die Ausgabe nennt, ob der Parameter Pflicht ist (`Required`), an welcher Stelle er ohne Namen angegeben werden darf (`Position`), ob er Werte aus der Pipeline entgegennimmt (`PipelineInput`) und ob Platzhalter erlaubt sind (`Globbing`). Im Test lieferte die Ausgabe für `Path`: nicht Pflicht, Position 0, Pipelineeingabe „true (ByValue, ByPropertyName)“. Die zweite Anweisung listet diese Eigenschaften für *alle* Parameter in einer Tabelle, was oft schneller ist als das Lesen des Hilfetexts. Dabei zeigt sich auch, dass Parameter ohne Positionsangabe (`Named`) immer mit ihrem Namen angegeben werden müssen. Die dritte Zeile liest die Hilfe wie ein gewöhnliches Objekt: `Get-Process` hat sechs Syntaxvarianten (Parametersätze, Lerneinheit 2.7).

Die Syntaxzeile eines Befehls folgt einer festen Notation, die man einmal lernen sollte. Eckige Klammern um einen Parameter bedeuten, dass er optional ist. Eckige Klammern um den *Namen* (`[-Name]`) bedeuten, dass der Name weggelassen werden darf, der Wert dann aber an der richtigen Position stehen muss. Ein Typ in spitzen Klammern (`<string[]>`) nennt, welche Art von Wert erwartet wird; die doppelten eckigen Klammern des Typs bedeuten eine Liste, es dürfen also mehrere Werte durch Komma getrennt übergeben werden. Parameter ohne Typ sind Schalter und werden nur angegeben oder weggelassen. Mehrere Syntaxzeilen unter einem Befehl sind verschiedene *Parametersätze*: Je Aufruf darf nur Parameter aus einem Satz verwendet werden.

> **Ergänzung aus der Nachrecherche**
>
> **Hilfe ist nicht automatisch vollständig.** Auf einem frisch installierten PowerShell 7 liefert `Get-Help` nur eine automatisch erzeugte Kurzfassung. Im Test (PowerShell 7.6.6 unter Linux) erschien bei `Get-Help Get-Process` dieser Hinweis: „Get-Help cannot find the Help files for this cmdlet on this computer. It is displaying only partial help.“ mit der Aufforderung, `Update-Help` auszuführen. Beispiele fehlen in dieser Kurzfassung ganz. Laut der Microsoft-Dokumentation benötigt `Update-Help` in PowerShell 7 keine Administratorrechte mehr, weil die Hilfe standardmäßig benutzerbezogen gespeichert wird; in Windows PowerShell sind sie erforderlich.

Warum liefert PowerShell die Hilfe nicht gleich mit? Die Hilfedateien (XML im sogenannten MAML-Format) sind groß und werden unabhängig von den Programmen aktualisiert, damit Korrekturen und neue Beispiele auch ohne neue PowerShell-Version ankommen. `Update-Help` lädt sie aus dem Internet und legt sie lokal ab. Der Befehl ist standardmäßig auf einen Lauf in 24 Stunden begrenzt (`-Force` hebt das auf) und kennt Optionen wie `-Module` für einzelne Module und `-UICulture` für die gewünschte Sprache; für die meisten Befehle gibt es nur englische Hilfe, deutsche Übersetzungen sind selten.

Für Server ohne Internetzugang gibt es einen zweistufigen Weg: Auf einem Rechner mit Internet lädt `Save-Help` die Hilfedateien in einen Ordner oder auf eine Netzfreigabe, auf den isolierten Servern liest `Update-Help -SourcePath` sie von dort ein.

```powershell
# Auf dem Rechner mit Internetzugang
$freigabe = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'hilfe-offline'
New-Item -Path $freigabe -ItemType Directory -Force | Out-Null
Save-Help -DestinationPath $freigabe -Module Microsoft.PowerShell.Management -UICulture en-US

# Auf dem isolierten Server (Ordner vorher dorthin kopieren)
Update-Help -SourcePath $freigabe -Module Microsoft.PowerShell.Management -UICulture en-US
```

Der erste Block legt einen Ordner an und speichert dort die Hilfe für das Modul `Microsoft.PowerShell.Management`, in dem Befehle wie `Get-ChildItem` und `Get-Process` liegen. Im zweiten Block liest der isolierte Server die Dateien ein. In der Praxis liegt der Ordner auf einer Netzfreigabe, auf die alle Server zugreifen. Wichtig ist, dass die Module auf beiden Seiten übereinstimmen; `Save-Help` speichert nur die Hilfe für die Module, die auf dem Quellrechner installiert sind, oder für die ausdrücklich genannten.

> **Merke**
>
> Zuerst `Update-Help`, dann `Get-Help -Examples`. Beispiele sind der schnellste Weg, einen unbekannten Befehl zu verstehen. Bei Syntaxzeilen gilt: eckig heißt optional, doppelte eckige Klammern am Typ heißen Liste, mehrere Zeilen heißen mehrere Parametersätze.

Typische Fehler im Umgang mit der Hilfe sind schnell aufgezählt. Viele lesen nur die erste Zeile (die Kurzbeschreibung) und übersehen die Beispiele, die darunter stehen. Andere vergessen das Aktualisieren und wundern sich über dürftige Hilfetexte. Ein weiterer Fehler ist, den Hilfetext als Ersatz für das Experimentieren zu sehen: Hilfe beschreibt den Normalfall, bei Sonderfällen klärt nur ein Test mit `-WhatIf` oder in einer Testumgebung, was wirklich passiert.

```powershell
# FALSCH: im Internet nach Parametern suchen und raten
Get-ChildItem -Rekursiv

# RICHTIG: Parameter in der Hilfe nachsehen
Get-Help Get-ChildItem -Parameter Recurse
Get-ChildItem -Recurse
```

> **Praxisbeispiel**
>
> Bei Audits und Übernahmen fremder Skripte begegnen Ihnen ständig Befehle, die Sie nie zuvor gesehen haben. Wer den Reflex entwickelt, zuerst `Get-Help -Examples` auszuführen, bearbeitet solche Skripte deutlich schneller und sicherer. In abgeschotteten Rechenzentren ist `Save-Help` mit einer Freigabe der Standardweg, die Hilfe zentral zu pflegen.

## 2.2 Get-Command und Command Discovery

Die Hilfe beantwortet Fragen zu einem *bekannten* Befehl. Bleibt die Frage, wie man den *passenden* Befehl überhaupt findet. Dafür gibt es `Get-Command`. Das Cmdlet durchsucht alles, was PowerShell aufrufen kann: Cmdlets, Funktionen, Aliase, Skripte im Suchpfad und externe Programme. Das Ergebnis sind Objekte, deren Typ verrät, um welche Art Befehl es sich handelt (`CommandType`) und aus welchem Modul oder welchem Pfad er stammt (`Source`).

Weil Befehle dem Verb-Substantiv-Schema folgen (Lerneinheit 1.4), lassen sich Suchen sehr gezielt formulieren. Mit `-Verb` und `-Noun` grenzen Sie nach Aktion und Objekt ein, mit `-Module` nach Herkunft, mit `-Name` suchen Sie anhand des Namens, und Platzhalter wie das Sternchen sind überall erlaubt. Mit `-ParameterName` finden Sie Befehle, die einen bestimmten Parameter anbieten, und `-Syntax` zeigt direkt die Syntaxzeilen, ohne die Hilfe zu öffnen.

```powershell
Get-Command -Name '*-Item' -CommandType Cmdlet | Select-Object -ExpandProperty Name | Sort-Object
Get-Command -Verb Stop | Where-Object -Property CommandType -EQ 'Cmdlet' |
    Group-Object -Property ModuleName | Select-Object -Property Name, Count
Get-Command -Name pwsh | Select-Object -Property CommandType, Name, Source
Get-Command -Name Get-Process -Syntax
Get-Command -ParameterName ErrorVariable -CommandType Cmdlet | Measure-Object
```

Die erste Zeile findet alle Cmdlets, die auf `-Item` enden. Im Test lieferte sie neun Treffer: `Clear-Item`, `Copy-Item`, `Get-Item`, `Invoke-Item`, `Move-Item`, `New-Item`, `Remove-Item`, `Rename-Item` und `Set-Item`. Daran sieht man sehr schön die Familie der Befehle für Elemente. Die zweite Anweisung gruppiert alle „Stop“-Befehle nach Modul. Dass hier `Where-Object` statt des Parameters `-CommandType` verwendet wird, ist Absicht: `Get-Command` lässt `-Verb` und `-CommandType` *nicht* in einem Aufruf zu und meldet dann „Parameter set cannot be resolved using the specified named parameters“ (siehe Lerneinheit 2.7). Die dritte Zeile findet das externe Programm `pwsh` und zeigt seinen Pfad (`Source`); so klärt man, *welche* Programmkopie aufgerufen wird, wenn mehrere installiert sind. Die vierte zeigt die Syntaxzeilen von `Get-Process`, die fünfte zählt, wie viele Cmdlets den gemeinsamen Parameter `-ErrorVariable` haben; im Test waren es 209.

Wichtig für das Verständnis ist das *automatische Laden von Modulen*. Seit PowerShell 3.0 durchsucht `Get-Command` bei Bedarf die Module im Modulpfad und lädt ein Modul, sobald Sie einen seiner Befehle verwenden. Deshalb können Sie `Get-Command` auch für Befehle nutzen, deren Modul noch nicht importiert wurde, solange es installiert ist. Mit `-ListImported` beschränken Sie die Suche auf bereits geladene Module. Die Reihenfolge, nach der PowerShell bei mehreren Treffern entscheidet, kennen Sie aus Lerneinheit 1.6: Alias, Funktion, Cmdlet, externes Programm. Mit `-All` sehen Sie alle Treffer statt nur den, der gewinnt.

Das Ergebnis ist ein Objekt, das sich mit `Get-Member` untersuchen lässt (nächste Lerneinheit). Es hat Eigenschaften wie `Verb`, `Noun`, `ModuleName`, `Source`, `Version`, `Definition` (die Syntax) und `Parameters`, ein Wörterbuch aller Parameter. Man kann es sogar nutzen, um Werkzeuge zu bauen, etwa eine Liste aller Befehle, die einen bestimmten Parameter anbieten.

> **Merke**
>
> Befehle finden: `Get-Command -Verb ...`, `-Noun ...`, `-Module ...` oder Platzhalter im Namen. Welcher Befehl gewinnt, zeigt `-All`. Wo ein externes Programm liegt, zeigt die Eigenschaft `Source`.

Der typische Fehler ist, sich auf Gewohnheiten aus anderen Shells zu verlassen: Wer `where` oder `which` eintippt, erhält in PowerShell unter Windows einen Alias beziehungsweise ein anderes Verhalten. Ein weiterer Fehler ist, Pfade von Programmen fest in Skripte zu schreiben, statt sie über `Get-Command` aufzulösen.

```powershell
# FALSCH: fester Pfad, auf anderem System ungültig
& 'C:\Program Files\Git\bin\git.exe' status

# RICHTIG: Programm im Suchpfad finden und Fehler abfangen
$git = Get-Command -Name git -CommandType Application -ErrorAction SilentlyContinue
if ($git) { & $git.Source status } else { Write-Warning 'git nicht gefunden.' }
```

Der zweite Block sucht das Programm im Pfad und führt es nur aus, wenn es gefunden wurde. Der Aufrufoperator `&` startet das Programm, dessen Pfad in einer Variablen steht.

> **Praxisbeispiel**
>
> Im Azure- oder Microsoft-365-Umfeld installieren Sie große Module mit hunderten Befehlen. Mit `Get-Command -Module <Modulname> -Verb New` verschaffen Sie sich in Sekunden einen Überblick, was sich damit alles anlegen lässt. Das ist meist schneller als jede Webrecherche.

## 2.3 Get-Member

Weil PowerShell Objekte weitergibt, muss man wissen, was in einem Objekt steckt. `Get-Member` ist das Werkzeug dafür. Es zeigt zu jedem Objekt, das es aus der Pipeline erhält, den *Typnamen* und eine Liste aller *Member*: Eigenschaften (Properties), Methoden (Methods), Ereignisse (Events) und weitere Arten. Der Typname verrät, womit man es zu tun hat, die Eigenschaften zeigen die Daten, die Methoden die möglichen Aktionen. Wer ein unbekanntes Objekt vor sich hat, etwa das Ergebnis einer Web-Abfrage, beginnt fast immer mit `| Get-Member`.

Die häufigsten Member-Arten sind `Property` (Wert, der vom .NET-Objekt kommt), `Method` (aufrufbare Funktion), `NoteProperty` (nachträglich angehängter Wert, wie er bei selbst erzeugten Objekten vorkommt), `ScriptProperty` (Wert, der per Skript berechnet wird) und `AliasProperty` (alternativer Name für eine andere Eigenschaft). Die letzten drei stammen vom *erweiterten Typsystem* (Extended Type System) von PowerShell, das .NET-Objekte um Komfortmember ergänzt. So hat jedes Prozessobjekt die kurzen Namen `WS` und `VM` als Alias für `WorkingSet64` und `VirtualMemorySize64`.

```powershell
Get-Process -Id $PID | Get-Member -MemberType Property |
    Select-Object -First 6 -Property Name, MemberType

'PowerShell'.Length
'PowerShell'.ToUpper()
'PowerShell' | Get-Member -Name To*Case

[string] | Get-Member -Static -Name Join
[string]::Join(', ', 'a', 'b', 'c')
```

Der erste Befehl zeigt sechs Eigenschaften des Prozessobjekts der aktuellen Sitzung (`$PID` enthält die Prozess-ID), darunter `BasePriority`, `ExitCode` und `Handle`. Im zweiten Abschnitt wird ein Text als Objekt behandelt: Die Eigenschaft `Length` liefert die Länge, die Methode `ToUpper()` wandelt in Großbuchstaben, und `Get-Member -Name To*Case` zeigt die Verwandten dieser Methode. Der dritte Abschnitt führt *statische* Member vor: Sie gehören nicht zu einem einzelnen Objekt, sondern zum Typ selbst und werden mit dem Typ in eckigen Klammern und zwei Doppelpunkten aufgerufen (`[string]::Join()`). Der Schalter `-Static` listet sie auf.

Ein oft übersehenes Verhalten betrifft Sammlungen. Wenn Sie ein Feld (Array) durch die Pipeline an `Get-Member` schicken, entpackt PowerShell es und zeigt den Typ der *Elemente*, nicht den des Feldes. Soll das Feld selbst untersucht werden, übergeben Sie es mit `-InputObject`.

```powershell
$zahlen = 1, 2, 3
($zahlen | Get-Member).TypeName | Select-Object -Unique
(Get-Member -InputObject $zahlen).TypeName | Select-Object -Unique
$zahlen.GetType().FullName
```

Die erste Zeile liefert `System.Int32`, weil jedes Element einzeln ankommt. Die zweite liefert `System.Object[]`, den Typ des Feldes selbst. Die dritte zeigt dasselbe über die .NET-Methode `GetType()`, die jedes Objekt besitzt.

> **Merke**
>
> Eigenschaften lesen Sie ohne Klammern, Methoden rufen Sie mit Klammern auf. Statische Member erreichen Sie über den Typ (`[Typ]::Member`). `Get-Member` entpackt Sammlungen aus der Pipeline; für das Feld selbst nehmen Sie `-InputObject`.

Zwei weitere Hilfsmittel ergänzen `Get-Member`. Mit `Select-Object -Property *` oder `Format-List -Property *` sehen Sie nicht nur die Namen der Eigenschaften, sondern auch deren *Werte*, was beim Erkunden oft wichtiger ist. Und über die verdeckte Eigenschaft `psobject` erreichen Sie die Metadaten jedes Objekts, etwa mit `$obj.psobject.Properties.Name` eine Liste der Eigenschaftsnamen, die sich in Skripten weiterverwenden lässt.

```powershell
(Get-Item -Path $HOME).psobject.Properties.Name | Select-Object -First 4
Get-Process -Id $PID | Select-Object -Property * | Format-List -Property Name, Id, ProcessName
```

Typische Fehler beim Arbeiten mit Objekten betreffen die Annahme, ein Wert sei ein Text. Wer ein Datum mit `Get-Date` erhält und es mit einem Text vergleicht, bekommt unerwartete Ergebnisse, weil ein Datumsobjekt beim Vergleich unterschiedlich umgewandelt wird. Wer die Eigenschaft `Length` einer Zeichenkette mit der `Length` einer Datei verwechselt, versteht den Unterschied sofort, wenn er den Typnamen im `Get-Member`-Kopf liest.

```powershell
# FALSCH: raten, welche Eigenschaft es gibt
Get-Process | Where-Object { $_.Speicher -gt 100MB }

# RICHTIG: erst erkunden, dann filtern
Get-Process | Get-Member -MemberType Property | Where-Object Name -Like '*Memory*'
Get-Process | Where-Object { $_.WorkingSet64 -gt 100MB }
```

Im falschen Beispiel gibt es die Eigenschaft `Speicher` nicht; der Vergleich liefert still immer „falsch“, weil nicht vorhandene Eigenschaften `$null` ergeben und kein Fehler auftritt. Das ist eine häufige Quelle unerklärlich leerer Ergebnisse.

> **Praxisbeispiel**
>
> Bei einer neuen REST-Schnittstelle liefert `Invoke-RestMethod` ein Objekt mit verschachtelten Eigenschaften. Mit `| Get-Member` und `| Format-List *` verschaffen Sie sich in zwei Minuten einen Überblick über die Struktur, bevor Sie eine einzige Zeile Auswertungscode schreiben.

## 2.4 Module Management

Ein *Modul* ist ein Paket, das zusammengehörige Befehle, Funktionen, Variablen, Aliase und gegebenenfalls Hilfedateien bündelt. Erst durch Module wird PowerShell erweiterbar: Die Befehle für Azure, Microsoft 365, Active Directory oder Tests stecken alle in Modulen. Je nach Aufbau unterscheidet man *Skriptmodule* (eine `.psm1`-Datei mit PowerShell-Code), *Binärmodule* (eine kompilierte .NET-Bibliothek, oft von Herstellern) und *Manifestmodule*. Das *Manifest* ist eine `.psd1`-Datei, die Metadaten wie Name, Version, Autor, Voraussetzungen und die Liste der exportierten Befehle enthält. Technisch ist ein Manifest nur eine Hashtable, die als Datei gespeichert wurde.

PowerShell sucht Module in den Ordnern der Umgebungsvariablen `PSModulePath`. Im Test unter Linux enthielt sie benutzerbezogene und systemweite Ordner sowie den Installationsordner von PowerShell. Mit `Get-Module -ListAvailable` sehen Sie alle verfügbaren Module, mit `Get-Module` nur die in der aktuellen Sitzung geladenen. `Import-Module` lädt ein Modul ausdrücklich, `Remove-Module` entlädt es wieder. Wegen des automatischen Ladens (Lerneinheit 2.2) ist der ausdrückliche Import oft nicht nötig, in Skripten aber guter Stil, weil er Abhängigkeiten sichtbar macht und früh fehlschlägt, wenn ein Modul fehlt.

```powershell
Get-Module -ListAvailable | Select-Object -First 3 -Property Name, Version, ModuleType
Get-Module | Select-Object -ExpandProperty Name
$env:PSModulePath -split [System.IO.Path]::PathSeparator
```

Die erste Zeile zeigt, dass auch die mitgelieferten Module (zum Beispiel `Microsoft.PowerShell.Archive`, `Microsoft.PowerShell.Management`) Manifestmodule mit Versionen sind. Die zweite zeigt die geladenen Module, die dritte zerlegt den Modulpfad plattformneutral in seine Ordner.

Neue Module stammen meist aus der *PowerShell Gallery*, dem zentralen Verzeichnis für Module und Skripte. Zum Suchen, Installieren, Aktualisieren und Veröffentlichen von Paketen gibt es zwei Generationen von Werkzeugen.

> **Ergänzung aus der Nachrecherche**
>
> **Paketverwaltung: PowerShellGet und PSResourceGet (abgerufen aus der Microsoft-Dokumentation, Oktober 2026).** Microsoft stellt drei Werkzeuge bereit: das Modul `Microsoft.PowerShell.PSResourceGet`, die älteren Module `PowerShellGet` und `PackageManagement` sowie das NuGet-Modul für Visual Studio. PSResourceGet wird seit PowerShell 7.4.0 mitgeliefert, ist ein eigenständiges Modul ohne Abhängigkeit zu PowerShellGet und PackageManagement und *ersetzt* diese beiden. Die aktuelle Version war laut Dokumentation 1.2.0, eine Vorschau 1.3.0-preview1 macht die Microsoft Artifact Registry zu einem Standard-Repository neben der PowerShell Gallery. Die in Windows PowerShell 5.1 mitgelieferte Version 1.0.0.1 von PowerShellGet wird nicht mehr unterstützt und muss aktualisiert werden. Im Test (PowerShell 7.6.6 unter Linux) war `Install-PSResource` in Version 1.2.0 vorhanden, `Install-Module` dagegen nicht mehr. Merken Sie sich daher beide Namensfamilien: `Find-/Install-/Update-/Uninstall-PSResource` (neu) und `Find-/Install-/Update-/Uninstall-Module` (alt, vor allem für Windows PowerShell 5.1).

Die Namen der neuen Befehle folgen der gewohnten Logik. Ein typischer Ablauf besteht aus vier Schritten, die im Folgenden gezeigt werden. Beachten Sie, dass dieses Beispiel auf einem Rechner mit Zugang zur PowerShell Gallery ausgeführt werden muss; im abgeschotteten Testsystem dieses Skripts war der Zugriff gesperrt, die Syntax wurde mit `Get-Command -Syntax` gegen die installierte Version geprüft.

```powershell
# 1. Suchen
Find-PSResource -Name PSScriptAnalyzer -Repository PSGallery

# 2. Installieren (nur für den aktuellen Benutzer, ohne Adminrechte)
Install-PSResource -Name PSScriptAnalyzer -Scope CurrentUser -TrustRepository

# 3. Prüfen und laden
Get-InstalledPSResource -Name PSScriptAnalyzer
Import-Module -Name PSScriptAnalyzer
Get-Command -Module PSScriptAnalyzer

# 4. Auf eine feste Version beschränken, später aktualisieren oder entfernen
Install-PSResource -Name Pester -Version '[5.0.0,6.0.0)' -Scope CurrentUser
Update-PSResource -Name Pester
Uninstall-PSResource -Name PSScriptAnalyzer
```

Schritt 1 sucht das Paket; im Ergebnis stehen Name, Version und Herkunft. Schritt 2 installiert es mit `-Scope CurrentUser` in den Benutzerordner, was keine Administratorrechte erfordert. `-TrustRepository` unterdrückt die Rückfrage, ob dem Repository vertraut wird. Die PowerShell Gallery ist standardmäßig als nicht vertrauenswürdig markiert (im Test: `Trusted = False`); das ist ein Sicherheitsmerkmal, kein Fehler. Schritt 3 prüft die Installation, lädt das Modul und listet seine Befehle. Schritt 4 zeigt die Versionsschreibweise: `'[5.0.0,6.0.0)'` bedeutet „mindestens 5.0.0, aber kleiner als 6.0.0“; die eckige Klammer schließt die Grenze ein, die runde schließt sie aus. Damit sind unbeabsichtigte Sprünge auf neue Hauptversionen ausgeschlossen. Für Windows PowerShell 5.1 gelten die entsprechenden `-Module`-Befehle `Find-Module`, `Install-Module` und `Update-Module`.

Unternehmen betreiben oft eigene Repositories, etwa auf einer Dateifreigabe oder in einem Paketdienst, und geben nur geprüfte Pakete frei. Ein solches Repository registrieren Sie mit `Register-PSResourceRepository -Name Intern -Uri <Pfad oder URL> -Trusted`, bestehende Einstellungen ändern Sie mit `Set-PSResourceRepository`. Den Zusammenhang von Quellenprüfung, Versionsfixierung und Lieferkettensicherheit behandelt Modul 8.

> **Merke**
>
> Module: `Get-Module -ListAvailable` (installiert), `Get-Module` (geladen), `Import-Module` (laden). Installation aus der Gallery mit `Install-PSResource` (neu) oder `Install-Module` (alt), bevorzugt `-Scope CurrentUser`. Versionen in Skripten eingrenzen.

```powershell
# FALSCH: neueste Version ungeprüft aus dem Netz holen und Rückfragen wegklicken
Install-PSResource -Name EinModul -Scope AllUsers -TrustRepository

# RICHTIG: Quelle prüfen, Version festlegen, Wirkungsbereich begrenzen
Find-PSResource -Name EinModul | Format-List Name, Version, Author, Repository
Install-PSResource -Name EinModul -Version '[2.1.0,3.0.0)' -Scope CurrentUser
```

Im ersten Fall installieren Sie ein unbekanntes Paket für alle Benutzer des Systems, ohne etwas über Herkunft oder Autor zu wissen. Im zweiten Fall prüfen Sie zuerst die Metadaten und begrenzen Version und Wirkung.

> **Praxisbeispiel**
>
> Ein Build-Server soll Skripte mit `PSScriptAnalyzer` prüfen. Statt jedes Mal die neueste Version zu holen, legen Sie eine feste Versionsspanne fest und speichern sie im Repository. So liefert die Prüfung heute und in sechs Monaten dasselbe Ergebnis, und ein Update des Werkzeugs wird bewusst und nachvollziehbar vorgenommen.

## 2.5 Provider und PSDrives

Eine der eleganten Ideen von PowerShell ist, dass sehr unterschiedliche Datenspeicher einheitlich bedient werden können. Das Dateisystem, die Registrierung, Umgebungsvariablen, Zertifikate oder die Menge der definierten Funktionen: Alle sind hierarchisch aufgebaut, und deshalb lassen sie sich wie Ordner und Dateien durchblättern. Möglich machen das *Provider*. Ein Provider ist ein Adapter, der einen Datenspeicher so darstellt, dass die immer gleichen Befehle (`Get-ChildItem`, `Get-Item`, `Set-Location`, `New-Item`, `Remove-Item` und weitere) darauf funktionieren. Auf den Datenspeicher greift man über ein *PSDrive* zu, ein Laufwerk, dem PowerShell einen Namen gibt, etwa `C:`, `Env:` oder `HKLM:`.

Welche Provider es gibt, hängt vom Betriebssystem ab. Im Test unter Linux standen `Alias`, `Environment`, `FileSystem`, `Function` und `Variable` zur Verfügung. Unter Windows kommen zusätzlich `Registry` (Laufwerke `HKLM:` und `HKCU:`), `Certificate` (Laufwerk `Cert:`) und `WSMan` hinzu. PowerShell 7 bringt außerdem ein Laufwerk `Temp:` für das temporäre Verzeichnis mit; im Test war es ebenfalls vorhanden.

```powershell
Get-PSProvider | Select-Object -Property Name,
    @{ Name = 'Laufwerke'; Expression = { $_.Drives.Name -join ', ' } }

$env:MEINE_VARIABLE = 'Hallo'
Get-ChildItem -Path Env:MEINE*

Get-ChildItem -Path Function: | Select-Object -First 3 -ExpandProperty Name
Get-Item -Path Variable:PSVersionTable | Select-Object -Property Name
Test-Path -Path Env:HOME
```

Die erste Anweisung listet die Provider samt Laufwerken auf. Die Schreibweise `$env:MEINE_VARIABLE` setzt eine Umgebungsvariable; sie greift auf das Laufwerk `Env:` zu, ohne dass man es ausdrücklich nennt. `Get-ChildItem -Path Env:MEINE*` listet daraufhin alle Umgebungsvariablen, die mit „MEINE“ beginnen. Mit den Laufwerken `Function:` und `Variable:` lassen sich definierte Funktionen und Variablen wie Dateien auflisten oder abfragen, `Test-Path` prüft die Existenz eines Eintrags. Das Prinzip ist immer dasselbe, nur der Datenspeicher wechselt.

> **Ergänzung aus der Nachrecherche**
>
> **Umgebungsvariablen sind prozessbezogen.** Eine Zuweisung wie `$env:MEINE_VARIABLE = 'Hallo'` gilt nur für den laufenden PowerShell-Prozess und die von ihm gestarteten Programme. Sie überlebt das Beenden der Sitzung nicht. Dauerhaft setzen Sie Variablen über die .NET-Methode `[Environment]::SetEnvironmentVariable(Name, Wert, 'User')` (unter Windows; unter Linux und macOS über Konfigurationsdateien der Shell). Eine solche Änderung wirkt erst in neu gestarteten Prozessen.

Eigene Laufwerke legen Sie mit `New-PSDrive` an. Das ist nützlich, um lange Pfade abzukürzen oder Projektordner einheitlich zu erreichen. Ein so erzeugtes Laufwerk existiert nur in der laufenden Sitzung (außer bei Netzlaufwerken unter Windows mit `-Persist`).

```powershell
$dir = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'psdrive-demo'
New-Item -Path $dir -ItemType Directory -Force | Out-Null
$datei = Join-Path -Path $dir -ChildPath 'a[1].txt'
Set-Content -LiteralPath $datei -Value 'x'

New-PSDrive -Name Demo -PSProvider FileSystem -Root $dir | Out-Null
Set-Location -Path Demo:
(Get-ChildItem).Name
Set-Location -Path $HOME
Remove-PSDrive -Name Demo
```

Der Ordner enthält eine Datei, deren Name eckige Klammern hat. Das Laufwerk `Demo:` zeigt direkt auf den Ordner; nach dem Wechsel mit `Set-Location` listet `Get-ChildItem` dessen Inhalt. Am Ende wechselt man zurück und entfernt das Laufwerk wieder. Wichtig ist hier die Dateierstellung mit `-LiteralPath`: Der Name enthält Zeichen, die in PowerShell als Platzhalter gelten.

Damit ist ein häufiger Stolperstein angesprochen. Die meisten Befehle kennen zwei Parameter für Pfade. `-Path` interpretiert Platzhalter wie `*`, `?` und `[a-c]`, `-LiteralPath` nimmt den Pfad wörtlich. Bei Dateinamen mit eckigen Klammern liefert `-Path` deshalb keine Treffer oder sogar Fehler. Im Test erzeugte `Set-Content -Path` für den Namen `a[1].txt` keine Datei und warf einen Fehler („An object at the specified path ... does not exist“), und `Get-ChildItem -Path` fand die vorhandene Datei nicht. Mit `-LiteralPath` funktionierte beides. PowerShell 7 hat das Verhalten dahin korrigiert, dass `-LiteralPath` Platzhalter konsequent wörtlich nimmt.

> **Merke**
>
> Provider machen unterschiedliche Datenspeicher mit denselben Befehlen bedienbar; Zugriff über PSDrives (`C:`, `Env:`, `HKLM:`, `Function:`, `Variable:`, `Alias:`, `Cert:`). Bei Namen mit Sonderzeichen immer `-LiteralPath` verwenden. Welche Provider es gibt, hängt vom Betriebssystem ab.

```powershell
# FALSCH: Annahme, die Variable bleibe erhalten, und Windows-Registry auf Linux
$env:API_URL = 'https://beispiel.at/api'
Get-ItemProperty -Path HKCU:\Software\Meine-App

# RICHTIG: Plattform prüfen, Konfiguration aus Datei oder Parameter beziehen
if ($IsWindows -or $PSVersionTable.PSEdition -eq 'Desktop') {
    Get-ItemProperty -Path 'HKCU:\Software\Meine-App' -ErrorAction SilentlyContinue
}
```

> **Praxisbeispiel**
>
> Zertifikatsprüfungen sind ein Paradebeispiel für den Nutzen von Providern: Mit `Get-ChildItem Cert:\LocalMachine\My` (unter Windows) listen Sie alle Zertifikate des Computers und filtern nach Ablaufdatum, sodass ein kurzes Skript bald ablaufende Zertifikate meldet. Dieselbe Technik funktioniert für Registrierungseinträge: Sie lesen und ändern sie mit denselben Befehlen wie Dateien.

## 2.6 Aliase und Lesbarkeit

Ein *Alias* ist ein alternativer, meist kürzerer Name für einen Befehl. PowerShell bringt eine Menge davon mit; im Test unter Linux waren es 120, unter Windows sind es mehr. Bekannte Beispiele sind `gci` und `dir` für `Get-ChildItem`, `%` für `ForEach-Object`, `?` für `Where-Object`, `select` für `Select-Object` oder `sort` für `Sort-Object`. Aliase sind für die interaktive Arbeit gedacht: Wer eine Kommandozeile tippt, spart Anschläge. Auch viele Alias-Namen aus der Unix- und DOS-Welt (`cd`, `cp`, `rm`) erleichtern den Umstieg.

In Skripten sind Aliase dagegen problematisch. Erstens sind sie *weniger verständlich*: Wer nicht jeden Alias kennt, rät, was `?` oder `%` bedeutet. Zweitens sind sie *weniger zuverlässig*: Aliase sind veränderbar, jeder kann eigene definieren oder vorhandene überschreiben, und sie unterscheiden sich zwischen den Plattformen. Im Test war `ls` unter Linux *kein* Alias (damit das native Programm aufgerufen wird), unter Windows ist es einer. In Windows PowerShell 5.1 zeigen `curl` und `wget` auf `Invoke-WebRequest`, was bei Skripten, die das echte `curl` erwarten, zu Verwirrung führt. Drittens sind Aliase *schwerer zu prüfen*: Statische Analyse und Code-Reviews arbeiten mit ausgeschriebenen Namen deutlich besser. Das Werkzeug PSScriptAnalyzer enthält dafür die Regel `PSAvoidUsingCmdletAliases` (Modul 5, Modul 11).

```powershell
Get-Alias -Definition Get-ChildItem | Select-Object -ExpandProperty Name
Get-Alias -Name gci, select, sort | Format-Table -Property Name, Definition -AutoSize
(Get-Alias | Measure-Object).Count

Set-Alias -Name ll -Value Get-ChildItem
Get-Alias -Name ll
Remove-Item -Path Alias:ll
```

Die erste Zeile fragt rückwärts: Welche Aliase gibt es für `Get-ChildItem`? Im Test waren es `dir` und `gci`. Die zweite Zeile schlägt Aliase nach, die dritte zählt sie. Mit `Set-Alias` (oder `New-Alias`) legen Sie eigene an; sie gelten nur in der aktuellen Sitzung, wenn man sie nicht in einem Profil definiert oder mit `Export-Alias` speichert. Entfernen lassen sie sich über das Laufwerk `Alias:`, wie in Lerneinheit 1.6 gezeigt.

Ein Alias kann kein Argument verpacken: `Set-Alias -Name ll -Value 'Get-ChildItem -Force'` funktioniert nicht, weil der Wert ein Befehlsname sein muss. Wer einen Befehl mit festen Parametern abkürzen möchte, schreibt stattdessen eine kleine Funktion.

Wie stößt man in geerbten Skripten auf Aliase? Manuell ist das mühsam. Eleganter ist ein Blick auf die Struktur des Skripts, die PowerShell selbst über seinen Parser zur Verfügung stellt. Das folgende Beispiel zerlegt einen Beispielbefehl, findet alle aufgerufenen Befehle und prüft, ob es sich um Aliase handelt.

```powershell
$code = 'gci | ? { $_.Length -gt 1KB } | select Name'
$tokens = $null; $errors = $null
$ast = [System.Management.Automation.Language.Parser]::ParseInput($code, [ref]$tokens, [ref]$errors)

$tabelle = @{}
Get-Alias | ForEach-Object { $tabelle[$_.Name] = $_.Definition }

$ast.FindAll({ param($n) $n -is [System.Management.Automation.Language.CommandAst] }, $true) |
    ForEach-Object {
        $name = $_.GetCommandName()
        [pscustomobject]@{ Befehl = $name; IstAlias = $tabelle.ContainsKey($name); Ziel = $tabelle[$name] }
    }
```

Der Parser liefert einen *abstrakten Syntaxbaum* (AST) des Textes. `FindAll` sucht darin alle Knoten vom Typ `CommandAst`, also alle Befehlsaufrufe, und `GetCommandName()` gibt deren Namen zurück. Die Hashtable `$tabelle` bildet jeden Alias auf sein Ziel ab, was schneller und eindeutiger ist als `Get-Alias -Name ?`, denn ein Fragezeichen wäre dort ein Platzhalter. Das Ergebnis zeigt für `gci`, `?` und `select` jeweils `IstAlias = True` mit den Zielen `Get-ChildItem`, `Where-Object` und `Select-Object`. Dieselbe Technik bildet die Grundlage von Werkzeugen, die Skripte automatisch ausschreiben.

> **Merke**
>
> Aliase interaktiv ja, in Skripten nein. In Skripten gehören vollständige Cmdlet-Namen und ausgeschriebene Parameternamen, damit der Code verständlich, prüfbar und plattformunabhängig bleibt.

```powershell
# FALSCH (für ein Skript): Kurzformen
gci -r *.log | ? Length -gt 1MB | sort Length -desc | select -f 5 Name

# RICHTIG: ausgeschrieben, lesbar, prüfbar
Get-ChildItem -Path . -Recurse -Filter '*.log' |
    Where-Object -Property Length -GT -Value 1MB |
    Sort-Object -Property Length -Descending |
    Select-Object -First 5 -Property Name
```

Beide Fassungen leisten dasselbe; die zweite ist für Kollegen und Prüfwerkzeuge ohne Vorwissen lesbar und bricht nicht, wenn jemand einen Alias umdefiniert. Dasselbe gilt für Parameternamen: Abgekürzte Parameter (`-r` statt `-Recurse`) funktionieren nur, solange sie eindeutig sind, und können bei einer neuen Version des Befehls plötzlich mehrdeutig werden.

> **Praxisbeispiel**
>
> In Code-Reviews hat sich die Regel bewährt, dass jedes Skript mit PSScriptAnalyzer geprüft wird und Verstöße gegen die Alias-Regel nicht in die Hauptzweige gelangen. Das erspart die Diskussion über Geschmack und führt zu Skripten, die auch Jahre später noch jeder lesen kann.

## 2.7 Parameter Binding

Wenn Sie einen Befehl aufrufen, muss PowerShell die übergebenen Werte den Parametern des Befehls zuordnen. Dieser Vorgang heißt *Parameter Binding*. Meistens läuft er unbemerkt ab; wer ihn versteht, kann aber Fehlermeldungen lesen, die sonst rätselhaft wirken, und Pipelines gezielt bauen. PowerShell kennt dafür drei Wege.

Beim *benannten* Binden nennen Sie den Parameter ausdrücklich (`-Path C:\Temp`). Beim *positionalen* Binden lassen Sie den Namen weg und verlassen sich auf die Reihenfolge: `Get-ChildItem $d '*.txt'` bindet den ersten Wert an den Parameter an Position 0 (`Path`) und den zweiten an den Parameter an Position 1 (`Filter`). Welche Parameter eine Position haben, steht in der Hilfe und in der Tabelle aus Lerneinheit 2.1. Beim Binden *aus der Pipeline* schließlich ordnet PowerShell jedes ankommende Objekt einem Parameter zu, der Pipelineeingabe erlaubt, und das auf zwei Arten: *ByValue* bindet das Objekt, wenn sein Typ zum Typ des Parameters passt, *ByPropertyName* bindet, wenn das Objekt eine Eigenschaft hat, die genauso heißt wie der Parameter.

```powershell
Get-Process -Id $PID | Stop-Process -WhatIf

[pscustomobject]@{ Id = $PID } | Get-Process | Select-Object -ExpandProperty Id

$d = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'bind-demo'
New-Item -Path $d -ItemType Directory -Force | Out-Null
'a' | Set-Content -LiteralPath (Join-Path -Path $d -ChildPath 'a.txt')

$d | Get-ChildItem | Select-Object -ExpandProperty Name
[pscustomobject]@{ Path = $d } | Get-ChildItem | Select-Object -ExpandProperty Name
Get-ChildItem $d '*.txt' | Select-Object -ExpandProperty Name
```

Die erste Zeile zeigt *ByValue*: `Stop-Process` nimmt in seinem Parameter `InputObject` Prozessobjekte entgegen, und das aus `Get-Process` ankommende Objekt hat genau diesen Typ. Der Zusatz `-WhatIf` verhindert, dass Ihre eigene Sitzung beendet wird; die Ausgabe lautet im Test: „What if: Performing the operation "Stop-Process" on target "pwsh (1356)".“ Die zweite Zeile zeigt *ByPropertyName*: Das selbst erzeugte Objekt ist kein Prozess, hat aber eine Eigenschaft `Id`, und `Get-Process` hat einen Parameter `-Id`, der dieses Binden erlaubt. Die drei folgenden Zeilen zeigen, dass `Get-ChildItem` den Pfad auf beide Arten annimmt: als Text (ByValue) oder als Objekt mit der Eigenschaft `Path` (ByPropertyName); die letzte Zeile bindet positional. In allen drei Fällen lautet das Ergebnis `a.txt`.

Wenn PowerShell nichts findet, was passt, bricht es mit einer Fehlermeldung ab. Diese Meldungen sind lesbar, wenn man ihre Bedeutung kennt. Falsch geschriebene Parameter führen zu „A parameter cannot be found that matches parameter name ’Pfad’.“ Eine Kombination von Parametern aus verschiedenen Parametersätzen führt zu „Parameter set cannot be resolved using the specified named parameters.“ Und ein Objekt aus der Pipeline, das zu keinem Parameter passt, führt zu „The input object cannot be bound to any parameters for the command either because the command does not take pipeline input or the input and its properties do not match any of the parameters that take pipeline input.“

```powershell
try { Get-Process -Name pwsh -Id $PID -ErrorAction Stop } catch { $_.Exception.Message }
try { Get-ChildItem -Pfad $d -ErrorAction Stop } catch { $_.Exception.Message }
try { [pscustomobject]@{ Name = 'x' } | Get-Date -ErrorAction Stop } catch { $_.Exception.Message }
```

Die drei Aufrufe lösen genau die drei Meldungen aus und fangen sie mit `try` und `catch` ab (ausführlich in Modul 5). Der erste kombiniert `-Name` und `-Id`, die zu verschiedenen Parametersätzen von `Get-Process` gehören. Der zweite enthält einen Tippfehler im Parameternamen. Der dritte schickt ein Objekt an `Get-Date`, das keine passende Eigenschaft hat.

Zur Fehlersuche gibt es ein mächtiges Werkzeug: `Trace-Command` (mit `-Name ParameterBinding -PSHost`) dem Parameter `-Expression` und einem Skriptblock protokolliert Schritt für Schritt, wie PowerShell Werte zuordnet und umwandelt. Im Test erschienen dabei Zeilen wie „BIND arg \[/tmp/bind-demo\] to parameter \[Path\]“ und „CONVERT arg type to param type using LanguagePrimitives.ConvertTo“. Die Ausgabe ist ausführlich, deshalb sollte man sie nur bei hartnäckigen Problemen einsetzen.

> **Merke**
>
> Drei Wege der Bindung: benannt, positional, aus der Pipeline (ByValue, ByPropertyName). Die Hilfe (`Get-Help -Parameter *`) zeigt Position und Pipelineeingabe jedes Parameters. Mehrere Syntaxzeilen sind Parametersätze, die nicht gemischt werden dürfen.

```powershell
# FALSCH: Parameter aus verschiedenen Parametersätzen mischen
Get-Process -Name pwsh -Id 1234

# RICHTIG: einen Parametersatz wählen
Get-Process -Name pwsh
Get-Process -Id 1234
```

> **Praxisbeispiel**
>
> In der Praxis zahlt sich Parameter Binding beim Verketten von Befehlen aus: `Import-Csv benutzer.csv | New-ADUser` legt (unter Windows mit dem Modul `ActiveDirectory`) Konten an, wenn die Spaltennamen der CSV-Datei den Parameternamen entsprechen. Eine gut gewählte Spaltenüberschrift ersetzt dann eine ganze Schleife.

## 2.8 Profiles und Umgebung

Lerneinheit 1.5 hat Profile als Skripte eingeführt, die beim Start einer interaktiven Sitzung laufen. Hier geht es um den kontrollierten Einsatz und die Frage, wie man verhindert, dass Skripte unbemerkt von der Umgebung ihres Entwicklers abhängig werden. Es gibt vier Profile; sie werden in dieser Reihenfolge geladen: `AllUsersAllHosts`, `AllUsersCurrentHost`, `CurrentUserAllHosts`, `CurrentUserCurrentHost`. Spätere Profile können Einstellungen früherer überschreiben. Die Pfade liefert `$PROFILE`, wobei der Wert der Variablen selbst das persönliche Profil des aktuellen Hosts bezeichnet.

Was gehört in ein Profil? Gut aufgehoben sind persönliche Bequemlichkeiten, die nur Ihre interaktive Arbeit betreffen: die Gestaltung des Prompts, Einstellungen der Befehlszeile, Importe häufig genutzter Module, eigene Aliase und kurze Hilfsfunktionen. Nicht hineingehören Dinge, von denen Skripte abhängen sollen, also Konfiguration, Zugangsdaten oder Funktionen, die ein Skript ohne eigene Definition aufruft.

Besonders einflussreich ist die Variable `$PSDefaultParameterValues`, eine Hashtable, mit der Sie Standardwerte für Parameter beliebiger Befehle festlegen, ohne die Befehle zu ändern. Der Eintrag `$PSDefaultParameterValues['Get-ChildItem:Force'] = $true` lässt jedes `Get-ChildItem` in der Sitzung so arbeiten, als stünde `-Force` dabei. Für den interaktiven Gebrauch ist das praktisch, für Skripte gefährlich, denn das Verhalten eines Befehls hängt dann von unsichtbaren Voreinstellungen ab.

```powershell
# Beispiel für einen Profileintrag (nur interaktiv sinnvoll)
$PSDefaultParameterValues['Get-ChildItem:Force'] = $true

function prompt {
    "PS [$((Get-Location).Path | Split-Path -Leaf)] > "
}
```

Der erste Eintrag setzt einen Standardwert, die Funktion `prompt` bestimmt, wie die Eingabeaufforderung aussieht; hier erscheint nur der Name des aktuellen Ordners. Beides wirkt nur in Sitzungen, die dieses Profil laden. Eine Folge: Ein Skript, das auf dem eigenen Laptop mit `Get-ChildItem` auch versteckte Dateien findet, tut das auf dem Server nicht, wenn es sich unbewusst auf den Standardwert verlassen hat.

Es gibt mehrere Schutzmaßnahmen. Erstens starten Sie automatisierte Aufrufe mit `pwsh -NoProfile`, damit kein Profil geladen wird. Zweitens deklarieren Sie die Anforderungen eines Skripts ausdrücklich mit `#Requires`-Zeilen am Anfang, damit es sofort mit einer klaren Meldung abbricht, wenn die Umgebung nicht passt. Drittens geben Sie im Skript alle Parameter an, auf deren Wirkung es ankommt, statt sich auf Standardwerte zu verlassen.

```powershell
#Requires -Version 7.0
#Requires -Modules @{ ModuleName = 'Microsoft.PowerShell.PSResourceGet'; ModuleVersion = '1.0.0' }

[CmdletBinding()]
param()

# Skript startet mit definiertem Zustand, unabhängig vom Profil des Aufrufers
$PSDefaultParameterValues = @{}
Set-StrictMode -Version Latest

Get-ChildItem -Path $PSScriptRoot -Force -File | Select-Object -ExpandProperty Name
```

Die ersten beiden Zeilen sind Anweisungen an PowerShell und beginnen mit einem Doppelkreuz, sehen aber wie Kommentare aus. `#Requires -Version 7.0` bricht ab, wenn das Skript in einer älteren PowerShell gestartet wird; die zweite Zeile fordert ein Modul in Mindestversion. Die Zuweisung `\$PSDefaultParameterValues = @{}` legt im Skript eine *lokale* leere Hashtable an und überlagert damit die Einträge des Aufrufers für alle Befehle, die das Skript ausführt (Variablen werden von innen nach außen gesucht). `Set-StrictMode -Version Latest` meldet Zugriffe auf nicht definierte Variablen als Fehler (Modul 5). Schließlich nennt der Aufruf von `Get-ChildItem` `-Force` und `-File` ausdrücklich.

Die Dauer des Starts ist ein weiterer Aspekt. Ein Profil, das viele Module importiert oder Netzwerkzugriffe macht, verlangsamt jede neue Sitzung. Messen lässt sich das mit `Measure-Command { pwsh -Command exit }` gegenüber `pwsh -NoProfile -Command exit`. Und wer Profile über mehrere Rechner teilt, legt sie in ein Git-Repository, damit Änderungen nachvollziehbar bleiben.

> **Merke**
>
> Profile sind für Bequemlichkeit, nicht für Funktion. Automatisierte Aufrufe mit `-NoProfile`, Anforderungen mit `#Requires`, wirksame Parameter im Skript ausdrücklich angeben.

```powershell
# FALSCH: Skript verlässt sich auf Profileinstellungen
$PSDefaultParameterValues['Export-Csv:Delimiter'] = ';'   # steht im Profil
Get-Process | Export-Csv -Path prozesse.csv

# RICHTIG: Trennzeichen im Skript festlegen
Get-Process | Export-Csv -Path prozesse.csv -Delimiter ';'
```

> **Praxisbeispiel**
>
> Typisches Szenario: Ein Skript läuft auf Ihrem Rechner perfekt, im nächtlichen geplanten Task liefert es leere Berichte. Die Ursache steckt oft im Profil: ein Alias, ein Standardparameter oder ein bereits importiertes Modul, das im Task fehlt. Der Test mit `pwsh -NoProfile -File skript.ps1` deckt solche Abhängigkeiten sofort auf.

## 2.9 Dokumentationsstandards

Dokumentation klingt nach Pflicht, ist aber ein Werkzeug in eigener Sache: Wer in einem halben Jahr ein eigenes Skript wieder öffnet, versteht es nur, wenn Zweck, Parameter und Beispiele festgehalten sind. PowerShell bietet dafür *kommentarbasierte Hilfe* (comment-based help). Sie schreiben die Dokumentation als besonders gestalteten Kommentar direkt in den Code, und `Get-Help` liest sie genauso aus wie die Hilfe der eingebauten Befehle. Dokumentation und Code bleiben so an einem Ort und veralten seltener.

Der Kommentarblock steht in einem Blockkommentar (`<# ... #>`) und enthält Schlüsselwörter mit vorangestelltem Punkt. Die wichtigsten sind `.SYNOPSIS` (Kurzbeschreibung in einem Satz), `.DESCRIPTION` (ausführliche Beschreibung), `.PARAMETER Name` (Beschreibung eines Parameters, je Parameter ein Eintrag), `.EXAMPLE` (ein Beispiel samt Erklärung, möglichst mehrere), `.INPUTS` und `.OUTPUTS` (Typen der Ein- und Ausgabe), `.NOTES` (Autor, Version, Hinweise) und `.LINK` (Verweise). Der Block darf an drei Stellen stehen: unmittelbar vor der Funktion (mit höchstens einer Leerzeile dazwischen), am Anfang des Funktionsrumpfs oder an dessen Ende. Bei Skriptdateien steht er am Dateianfang. Falsch platzierte Blöcke werden von PowerShell stillschweigend ignoriert, was die häufigste Fehlerquelle ist.

```powershell
function Get-OrdnerGroesse {
<#
.SYNOPSIS
Berechnet die Gesamtgröße von Ordnern.

.DESCRIPTION
Summiert die Größe aller Dateien eines oder mehrerer Ordner rekursiv und
gibt je Ordner ein Objekt mit Pfad und Größe in Megabyte zurück.

.PARAMETER Path
Pfad eines oder mehrerer Ordner. Die Pfade können auch über die Pipeline kommen.

.EXAMPLE
Get-OrdnerGroesse -Path $HOME
Berechnet die Größe des Home-Verzeichnisses.

.EXAMPLE
'/tmp', '/var' | Get-OrdnerGroesse
Verarbeitet mehrere Ordner über die Pipeline.

.OUTPUTS
System.Management.Automation.PSCustomObject
#>
    [CmdletBinding()]
    [OutputType([pscustomobject])]
    param(
        [Parameter(Mandatory, ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [ValidateScript({ Test-Path -Path $_ -PathType Container })]
        [string[]] $Path
    )
    process {
        foreach ($p in $Path) {
            $summe = (Get-ChildItem -LiteralPath $p -Recurse -File -ErrorAction SilentlyContinue |
                Measure-Object -Property Length -Sum).Sum
            if (-not $summe) { $summe = 0 }
            [pscustomobject]@{ Pfad = $p; GroesseMB = [math]::Round($summe / 1MB, 2) }
        }
    }
}

Get-Help -Name Get-OrdnerGroesse -Examples
(Get-Help -Name Get-OrdnerGroesse).Synopsis
```

Der Block mit `.SYNOPSIS` bis `.OUTPUTS` steht am Anfang des Funktionsrumpfs. Darunter folgt die eigentliche Funktion: `[CmdletBinding()]` macht sie zur Advanced Function, `[OutputType()]` deklariert den Ausgabetyp, und der Parameter `Path` ist Pflicht, nimmt Werte aus der Pipeline entgegen und wird durch `ValidateScript` geprüft, ob jeder Wert ein existierender Ordner ist. Diese Attribute werden in Modul 4 im Detail behandelt. Im `process`-Block wird je Ordner die Summe der Dateigrößen berechnet; `-LiteralPath` verhindert Probleme mit Sonderzeichen, und der Fall eines leeren Ordners (Summe `$null`) wird auf null gesetzt. Im Test lieferten `Get-Help ... -Synopsis` die Kurzbeschreibung und `-Examples` die Beispiele mit der Überschrift „EXAMPLE 1“; der Parameter `Path` wurde als Pflichtparameter („required: true“) erkannt.

Die kommentarbasierte Hilfe genügt für Skripte und kleine Module. Für große, veröffentlichte Module erzeugt man dagegen externe Hilfedateien im MAML-Format, meist aus Markdown-Dokumenten mit dem Werkzeug *PlatyPS*; die Hilfe lässt sich dann auch mehrsprachig pflegen und über `Update-Help` verteilen. Daneben bewährt sich eine Dokumentation außerhalb des Codes: eine `README`-Datei mit Zweck, Voraussetzungen und Aufrufbeispielen, ein `CHANGELOG` mit Änderungen je Version und, wo sinnvoll, konzeptionelle `about_`-Themen.

Für die Qualität der Texte gelten einfache Regeln. Die Kurzbeschreibung bestimmt in einem Satz, was der Befehl tut, und beginnt mit einem Verb („Berechnet“, „Liest“). Jeder Parameter wird beschrieben, auch wenn der Name selbsterklärend scheint, mit Angaben zu Format, Einheit und erlaubten Werten. Beispiele müssen lauffähig sein und zeigen den Normalfall zuerst, dann Sonderfälle wie die Verwendung per Pipeline.

> **Merke**
>
> Kommentarbasierte Hilfe: Block `<# ... #>` mit `.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`. An einer der erlaubten Stellen platzieren; sonst ignoriert PowerShell sie. Prüfen mit `Get-Help -Name <Funktion> -Full`.

```powershell
# FALSCH: Leerzeilen zwischen Hilfeblock und Funktion
<#
.SYNOPSIS
Tut etwas.
#>



function Get-Etwas { }

# RICHTIG: Block direkt vor der Funktion oder im Rumpf
<#
.SYNOPSIS
Tut etwas.
#>
function Get-Etwas { }
```

Im falschen Fall liefert `Get-Help Get-Etwas` nur die automatisch erzeugte Syntaxzeile, weil der Block nicht als zur Funktion gehörig erkannt wird.

> **Praxisbeispiel**
>
> Ein Team führt als Qualitätsregel ein, dass jede Funktion der gemeinsamen Bibliothek eine `.SYNOPSIS`, alle Parameter und mindestens ein Beispiel dokumentiert hat. Ein Pester-Test (Modul 5) prüft das automatisch mit `Get-Help`: fehlende Beschreibungen lassen die Pipeline fehlschlagen. So wird Dokumentation vom guten Vorsatz zur überprüfbaren Eigenschaft.

## 2.10 Übungsaufgaben zu Modul 02

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Erklären Sie die Bedeutung der Notation in der Syntaxzeile `Get-Process [[-Name] <string[]>] [-Module]`. Welche Angaben sind optional, wie wird `Name` übergeben, und was bedeutet `<string[]>`?

**Aufgabe 2.** Warum liefert `Get-Help` nach einer frischen Installation oft nur wenig Information, und wie lösen Sie das Problem auf einem Server ohne Internetzugang?

**Aufgabe 3.** Was ist der Unterschied zwischen `-Path` und `-LiteralPath`? Nennen Sie ein Beispiel, in dem nur `-LiteralPath` funktioniert.

**Aufgabe 4.** Nennen Sie vier Gründe, warum Aliase in Skripten vermieden werden sollten, und ein Werkzeug, das Verstöße erkennt.

**Aufgabe 5.** Finden Sie mit `Get-Command` alle Cmdlets mit dem Verb `ConvertTo` und geben Sie nur die Namen sortiert aus. Gruppieren Sie danach alle `Convert*`-Cmdlets nach Verb und zählen Sie sie.

**Aufgabe 6.** Untersuchen Sie das Objekt, das `Get-Date` liefert. Geben Sie mit `Get-Member` die Methoden aus, deren Name mit `Add` beginnt, und berechnen Sie damit das Datum in 100 Tagen im Format `yyyy-MM-dd`.

**Aufgabe 7.** Erstellen Sie ein eigenes PSDrive `Proj:`, das auf einen neuen Ordner im temporären Verzeichnis zeigt. Legen Sie über `Proj:` drei Dateien an (eine davon mit eckigen Klammern im Namen), listen Sie sie auf und löschen Sie danach Dateien, Ordner und PSDrive.

**Aufgabe 8.** Schreiben Sie eine Funktion `Get-AliasZiel`, die für einen übergebenen Befehlsnamen angibt, ob es ein Alias ist und welcher Befehl dahintersteckt. Dokumentieren Sie die Funktion mit kommentarbasierter Hilfe (Synopsis, Parameter, zwei Beispiele) und prüfen Sie mit `Get-Help`, ob die Hilfe angezeigt wird.

## 2.11 Lösungen zu Modul 02

### Aufgabe 1.

Eckige Klammern um einen Parameter (`[-Module]`) bedeuten: optional. `[-Name]` in eckigen Klammern bedeutet: der Parametername darf weggelassen werden, der Wert muss dann an erster Stelle stehen. Das äußere Klammerpaar um `[[-Name] <string[]>]` macht den ganzen Parameter optional. `<string[]>` nennt den erwarteten Typ: eine Liste von Zeichenketten, es dürfen also mehrere Namen durch Komma getrennt übergeben werden. `-Module` ohne Typ ist ein Schalter.

### Aufgabe 2.

Die Hilfedateien werden getrennt vom Programm geliefert und müssen mit `Update-Help` nachgeladen werden; ohne sie zeigt `Get-Help` nur eine automatisch erzeugte Kurzfassung. Ohne Internet: Auf einem Rechner mit Zugang `Save-Help -DestinationPath <Ordner>` ausführen, den Ordner auf eine Freigabe legen und auf dem Server `Update-Help -SourcePath <Ordner>` verwenden.

### Aufgabe 3.

`-Path` wertet Platzhalter (`*`, `?`, `[]`) aus, `-LiteralPath` nimmt den Pfad wörtlich. Beispiel: Eine Datei `a[1].txt` wird mit `Get-ChildItem -Path` nicht gefunden (die Klammern werden als Zeichenklasse gedeutet), mit `-LiteralPath` schon. Auch zum Anlegen der Datei mit `Set-Content` ist `-LiteralPath` nötig.

### Aufgabe 4.

Mögliche Gründe: schlechtere Lesbarkeit, Aliase sind veränderbar, Aliase unterscheiden sich zwischen Plattformen und Versionen (unter Linux fehlen etwa `ls`-Aliase, in Windows PowerShell 5.1 zeigt `curl` auf `Invoke-WebRequest`), schlechtere Prüfbarkeit durch Werkzeuge. Erkennung: PSScriptAnalyzer mit der Regel `PSAvoidUsingCmdletAliases`.

### Aufgabe 5.

```powershell
Get-Command -Verb ConvertTo | Select-Object -ExpandProperty Name | Sort-Object

Get-Command -Name 'Convert*' |
    Group-Object -Property Verb |
    Select-Object -Property Name, Count
```

`Get-Command -Verb ConvertTo` liefert Befehle wie `ConvertTo-Json` und `ConvertTo-Csv`. Für die Gruppierung wird der Platzhalter `Convert*` gegen den Namen verwendet, sodass auch `ConvertFrom-*`, `Convert-Path` und weitere erfasst werden; die Eigenschaft `Verb` der Ergebnisobjekte dient als Gruppierung. Welche Befehle genau erscheinen, hängt von den installierten Modulen ab.

### Aufgabe 6.

```powershell
Get-Date | Get-Member -MemberType Method -Name Add* | Select-Object -ExpandProperty Name
(Get-Date).AddDays(100).ToString('yyyy-MM-dd')
```

Die Methoden `AddDays`, `AddHours` und weitere liefern ein neues Datumsobjekt; das ursprüngliche bleibt unverändert, weil `DateTime`-Objekte unveränderlich sind. `ToString('yyyy-MM-dd')` formatiert das Ergebnis.

### Aufgabe 7.

```powershell
$wurzel = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath 'proj-uebung'
New-Item -Path $wurzel -ItemType Directory -Force | Out-Null
New-PSDrive -Name Proj -PSProvider FileSystem -Root $wurzel | Out-Null

'a', 'b' | ForEach-Object { Set-Content -Path "Proj:\$_.txt" -Value $_ }
Set-Content -LiteralPath 'Proj:\c[1].txt' -Value 'c'

Get-ChildItem -Path Proj: | Select-Object -ExpandProperty Name

Set-Location -Path $HOME
Remove-Item -Path 'Proj:\*' -Force -Recurse
Remove-PSDrive -Name Proj
Remove-Item -LiteralPath $wurzel -Recurse
```

Die Dateien `a.txt` und `b.txt` lassen sich mit `-Path` anlegen; für `c[1].txt` ist `-LiteralPath` nötig. Vor dem Entfernen des PSDrive muss man den aktuellen Ort verlassen, falls er darauf zeigt. Unter Windows verwenden Sie statt `Proj:\` gleichwertig `Proj:/`; PowerShell akzeptiert beide Trenner.

### Aufgabe 8.

```powershell
function Get-AliasZiel {
<#
.SYNOPSIS
Gibt an, ob ein Befehlsname ein Alias ist, und nennt das Ziel.
.PARAMETER Name
Name des Befehls, der geprüft werden soll.
.EXAMPLE
Get-AliasZiel -Name gci
Zeigt, dass gci ein Alias für Get-ChildItem ist.
.EXAMPLE
'select', 'Get-Date' | Get-AliasZiel
Prüft mehrere Namen über die Pipeline.
#>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string[]] $Name
    )
    begin {
        $tabelle = @{}
        Get-Alias | ForEach-Object { $tabelle[$_.Name] = $_.Definition }
    }
    process {
        foreach ($n in $Name) {
            [pscustomobject]@{
                Name     = $n
                IstAlias = $tabelle.ContainsKey($n)
                Ziel     = $tabelle[$n]
            }
        }
    }
}

Get-AliasZiel -Name gci, Get-Date
Get-Help -Name Get-AliasZiel -Examples
```

Die Tabelle wird einmal im `begin`-Block aufgebaut und im `process`-Block für jeden Namen genutzt, was bei vielen Eingaben effizienter ist als wiederholtes `Get-Alias`. Der Hilfeblock steht am Anfang des Funktionsrumpfs und wird deshalb von `Get-Help` erkannt.
