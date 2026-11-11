# Modul 11 - Automation, CI-CD und Infrastructure as Code

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (11.1 bis 11.8; 11.9 Übungsaufgaben, 11.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 10 h (8 Lerneinheiten à 1,25 h)      |
| Priorität                             | 2 (einzelne Lerneinheiten: 2 bzw. 3) |

> **Merke**
>
> Übersichtsblatt und Detailblatt stimmen in diesem Modul überein: 8 Lerneinheiten à 1,25 h ergeben 10 h. Das Modul führt die Bausteine der vorherigen Module (Fehlerbehandlung, Exit Codes, Protokolle, Module) zu einem automatisierten Lieferweg zusammen: vom Commit bis zur Veröffentlichung.

### Lernziele.

Nach diesem Modul können Sie einen automatisierten Prüf- und Lieferweg für PowerShell-Code aufbauen: ein Build-Skript, das lokal und in jeder Pipeline gleich läuft, Tests und statische Analyse als Qualitätsschranke, Module mit Versionsnummern in ein Repository veröffentlichen, gewünschte Zustände idempotent herstellen (Desired State), Infrastrukturwerkzeuge wie Terraform oder Bicep aus PowerShell steuern, Änderungen in Ringen ausrollen und Regeln als prüfbaren Code ausdrücken.

| **Nr.** | **Lerneinheit**        | **Dauer** | **Prio** |
|:--------|:-----------------------|:----------|:---------|
| 11.1    | CI/CD mit PowerShell   | 1,25 h    | 2        |
| 11.2    | Pester in Pipelines    | 1,25 h    | 2        |
| 11.3    | PSScriptAnalyzer in CI | 1,25 h    | 2        |
| 11.4    | Modul-Publishing       | 1,25 h    | 2        |
| 11.5    | DSC und Desired State  | 1,25 h    | 2        |
| 11.6    | IaC-Orchestrierung     | 1,25 h    | 2        |
| 11.7    | Release-Strategien     | 1,25 h    | 3        |
| 11.8    | Policy as Code         | 1,25 h    | 3        |
|         | **Summe**              | **10 h**  |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde, und was nicht.** Mit PowerShell 7.6.6 unter Linux wurden ausgeführt: das Build-Skript samt Exit Codes, ein selbst geschriebener Mini-Analysator auf Basis des Syntaxbaums, die Erstellung und Prüfung eines Modulmanifests, die Versionsanhebung, die *tatsächliche Veröffentlichung* eines Moduls in ein lokales Dateirepository mit dem im Arbeitsbereich vorhandenen Modul `Microsoft.PowerShell.PSResourceGet` (Version 1.2.0) einschließlich Vorabversion, ein idempotentes Zustandsmodell nach dem Muster von Get, Test und Set, ein Wrapper für ein externes Infrastrukturwerkzeug (mit einem Ersatzprogramm, das *kein* echtes Terraform ist), eine Ring-Ausrollung mit simulierten Gesundheitsprüfungen und eine Richtlinienprüfung auf JSON-Daten. *Nicht* ausgeführt werden konnten: Pester und PSScriptAnalyzer (die PowerShell Gallery war nicht erreichbar, beide Module waren nicht installiert), Pipelines in Azure DevOps, GitHub Actions oder GitLab, die DSC-Befehlszeile der Version 3, Terraform und Bicep sowie Azure Policy. Diese Beispiele sind mit „nur auf Syntax geprüft“ gekennzeichnet. Aussagen zu Azure Pipelines, PSScriptAnalyzer, Veröffentlichung von Modulen und DSC stützen sich auf die für dieses Modul abgerufenen Seiten der Microsoft-Dokumentation (Stand Oktober 2026). Aussagen zu Pester, GitHub Actions, GitLab, Terraform, Bicep und Open Policy Agent stammen aus Fachwissen und wurden *nicht* abgerufen; sie sind entsprechend markiert.

## 11.1 CI/CD mit PowerShell

*Continuous Integration* (CI, kontinuierliche Integration) heißt: Jede Änderung am Code löst automatisch eine Prüfung aus, noch bevor sie mit der Arbeit der anderen vermischt wird. Ein Server zieht den Stand aus dem Repository, führt Syntaxprüfung, Analyse und Tests aus und meldet das Ergebnis zurück. *Continuous Delivery* (CD) erweitert das um die Auslieferung: Eine geprüfte Änderung wird automatisch zu einem lieferbaren Paket und, je nach Freigabe, in die Zielumgebung gebracht. Beides zusammen nennt man *CI/CD-Pipeline*. Der Nutzen ist handfest: Fehler werden Minuten nach dem Entstehen gefunden statt Wochen später, jede Lieferung läuft nach demselben Rezept ab, und niemand muss sich merken, welche Schritte vor einer Veröffentlichung nötig sind.

Der wichtigste Gestaltungsgrundsatz lautet: *Die Pipeline ruft ein Skript auf, sie enthält nicht die Logik.* Alle Schritte stehen in einem Build-Skript im Repository, das man lokal genauso ausführen kann wie auf dem Server. Die Pipeline-Definition (eine YAML-Datei) ist dann nur noch eine dünne Hülle: Code holen, PowerShell starten, das Build-Skript aufrufen. Das hat drei Vorteile. Man kann Fehler am eigenen Rechner reproduzieren, statt hundertmal zu pushen und zu warten. Das Build-Skript ist unabhängig vom Anbieter: Wechselt das Team von einem Dienst zum anderen, ändert sich nur die kleine Hülle. Und es wird mit denselben Methoden getestet und geprüft wie jeder andere Code.

Das Build-Skript folgt den Regeln aus Modul 9: klare Exit Codes und eine knappe Ausgabe. Das folgende Beispiel prüft Syntax und Modulmanifest eines Projekts und liefert den Exit Code 0 bei Erfolg, 20 bei fehlgeschlagenen Prüfungen und 99 bei einem unerwarteten Fehler.

```powershell
# build.ps1
[CmdletBinding()]
param(
    [ValidateSet('Syntax', 'Manifest', 'Alle')] [string] $Aufgabe = 'Alle',
    [string] $Projekt = (Join-Path $PSScriptRoot 'src/Demo')
)
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest
$Codes = @{ Ok = 0; Fehler = 20; Unerwartet = 99 }
$ergebnisse = [System.Collections.Generic.List[object]]::new()

function Add-Ergebnis([string] $Name, [bool] $Ok, [string] $Detail = '') {
    $ergebnisse.Add([pscustomobject]@{ Aufgabe = $Name; Ok = $Ok; Detail = $Detail })
    Write-Host ("[{0}] {1} {2}" -f ($(if ($Ok) { 'OK ' } else { 'FEHLER' })), $Name, $Detail)
}

function Invoke-SyntaxPruefung {
    $fehlerAnzahl = 0
    foreach ($f in Get-ChildItem -LiteralPath $Projekt -Include *.ps1, *.psm1, *.psd1 -Recurse -File) {
        $t = $null; $e = $null
        [void][System.Management.Automation.Language.Parser]::ParseFile($f.FullName, [ref]$t, [ref]$e)
        if ($e.Count -gt 0) { $fehlerAnzahl++; Write-Warning "$($f.Name): $($e[0].Message)" }
    }
    Add-Ergebnis 'Syntax' ($fehlerAnzahl -eq 0) "$fehlerAnzahl Datei(en) mit Fehlern"
}

function Invoke-ManifestPruefung {
    $psd1 = Get-ChildItem -LiteralPath $Projekt -Filter *.psd1 -File | Select-Object -First 1
    try {
        $m = Test-ModuleManifest -Path $psd1.FullName -ErrorAction Stop
        Add-Ergebnis 'Manifest' $true "Version $($m.Version)"
    }
    catch { Add-Ergebnis 'Manifest' $false $_.Exception.Message }
}

try {
    if ($Aufgabe -in 'Syntax', 'Alle')   { Invoke-SyntaxPruefung }
    if ($Aufgabe -in 'Manifest', 'Alle') { Invoke-ManifestPruefung }
    $ergebnisse | ConvertTo-Json | Set-Content (Join-Path $PSScriptRoot 'ergebnis.json') -Encoding utf8
    if ($ergebnisse.Where({ -not $_.Ok }).Count -gt 0) { exit $Codes.Fehler }
    exit $Codes.Ok
}
catch {
    Write-Error $_ -ErrorAction Continue
    exit $Codes.Unerwartet
}
```

Das Skript beginnt mit einem Parameter `-Aufgabe`, der einzelne Schritte wählbar macht: Wer lokal nur die Syntax prüfen will, startet nur diesen Schritt. `Set-StrictMode` und `$ErrorActionPreference = 'Stop'` sorgen dafür, dass Tippfehler und Fehler sofort auffallen. Die Hilfsfunktion `Add-Ergebnis` sammelt Ergebnisse und schreibt eine lesbare Zeile. Die Syntaxprüfung benutzt den Parser, der Dateien nur liest und nie ausführt; ihre Fehler gelten als Befund, nicht als Absturz. Die Manifestprüfung nutzt `Test-ModuleManifest`. Am Ende steht die Zuordnung zu den Exit Codes. Das Ergebnis wird zusätzlich als JSON-Datei abgelegt, damit eine Pipeline es als Artefakt aufbewahren kann.

Beim Test zeigten sich zwei Dinge. Das Skript lieferte bei einem fehlerfreien Projekt den Exit Code 0 mit den Zeilen „\[OK \] Syntax 0 Datei(en) mit Fehlern“ und „\[OK \] Manifest Version 1.1.0“. Mit einer absichtlich beschädigten Datei (`if ($x {`) lieferte es die Warnung mit der Parser-Meldung und den Exit Code 20. Und der zweite Befund war eine Überraschung: Bei einem nicht vorhandenen Projektordner endete das Skript zunächst mit Exit Code 1 statt mit 99. Die Ursache: Wegen `$ErrorActionPreference = 'Stop'` löst auch der `Write-Error`-Aufruf *im* `catch`-Block selbst eine Ausnahme aus, die niemand mehr abfängt; das Skript stirbt, bevor es das `exit` erreicht. Erst mit `-ErrorAction Continue` an diesem einen Aufruf kam der erwartete Code 99. Das gezeigte Listing enthält diese Korrektur. Es ist eine typische Falle: Die Fehlerbehandlung selbst darf nicht an der strengen Einstellung scheitern, die sie schützen soll.

Die Pipeline-Definition ist jetzt klein. Für Azure Pipelines beschreibt die abgerufene Dokumentation den Schritt `pwsh` mit mehreren Eigenschaften, die für Skripte wichtig sind. `errorActionPreference` steht standardmäßig auf `stop`, wodurch die Pipeline vor das Skript die Zeile `$ErrorActionPreference = 'stop'` setzt. `failOnStderr` lässt den Schritt scheitern, sobald etwas in den Fehlerstrom geschrieben wird. `ignoreLASTEXITCODE` steuert, ob der zuletzt gesetzte Exit Code externer Programme den Schritt scheitern lässt; die Dokumentation sagt, dass er standardmäßig weitergereicht wird. Außerdem gibt es `workingDirectory`, `retryCountOnTaskFailure` (Wiederholung bei Fehlern) und `timeoutInMinutes`.

```powershell
# azure-pipelines.yml (nur auf Syntax geprueft; Eigenschaften laut Dokumentation des pwsh-Schritts)
trigger:
  branches:
    include: [ main ]
pool:
  vmImage: ubuntu-latest
steps:
  - pwsh: ./build.ps1 -Aufgabe Alle
    displayName: Build und Pruefung
    errorActionPreference: stop
    workingDirectory: $(Build.SourcesDirectory)
    timeoutInMinutes: 15

# GitHub Actions (Fachwissen, nicht abgerufen): .github/workflows/ci.yml
name: ci
on: [ push, pull_request ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build und Pruefung
        shell: pwsh
        run: ./build.ps1 -Aufgabe Alle

# GitLab CI (Fachwissen, nicht abgerufen): .gitlab-ci.yml
build:
  image: mcr.microsoft.com/powershell:latest
  script:
    - pwsh -File ./build.ps1 -Aufgabe Alle
```

Die drei Beispiele zeigen dasselbe Prinzip in drei Systemen: eine Auslöseregel, eine Umgebung (hier Linux), das Auschecken des Codes und der Aufruf des Build-Skripts. Das erste Beispiel stützt sich auf die abgerufene Dokumentation zu Azure Pipelines; das zweite und dritte stammen aus Fachwissen und sind nicht abgerufen. Die Versionsangabe `actions/checkout@v4` und der Name des Container-Abbilds sind Beispielwerte und vor dem Einsatz zu prüfen. Wichtig ist der Aufruf mit `-File`-Semantik: Dort kommt der Exit Code des Skripts an (siehe Modul 9, Lerneinheit 9.3). Ein Aufruf von `pwsh -Command ./build.ps1` dagegen würde nur 0 oder 1 liefern.

> **Ergänzung aus der Nachrecherche**
>
> **Geheimnisse und Rechte in Pipelines.** Pipelines brauchen oft Zugangsdaten (zum Veröffentlichen von Modulen, zum Anmelden in Azure). Diese gehören nicht in die YAML-Datei und nicht ins Repository, sondern in den Geheimnisspeicher des Anbieters oder, besser, in eine Identität ohne Geheimnis (föderierte Anmeldung). Laut der abgerufenen Dokumentation zu `Connect-AzAccount` werden auch Dienstprinzipale mit Zertifikat oder föderiertem Token unterstützt. Pipelines laufen mit den Rechten, die man ihnen gibt: Ein Prüfschritt für Pull Requests braucht keine Schreibrechte auf die Produktion. Ob und wie diese Mechanismen bei GitHub Actions und GitLab heißen, ist Fachwissen und anbieterspezifisch.

> **Merke**
>
> CI prüft jede Änderung automatisch, CD liefert geprüfte Änderungen aus. Die Pipeline ruft ein Build-Skript auf; die Logik steht im Repository und läuft lokal wie auf dem Server. Exit Codes entscheiden über Erfolg: `-File` liefert den Code des Skripts. Die Fehlerbehandlung darf nicht an `$ErrorActionPreference = 'Stop'` scheitern (`Write-Error ... -ErrorAction Continue` im `catch`).

```powershell
# FALSCH: Logik in der YAML-Datei verteilt, nur auf dem Server reproduzierbar
steps:
  - pwsh: |
      Install-Module X -Force
      Test-Foo; Test-Bar; Publish-Baz

# RICHTIG: dünne Pipeline, ein Skript, lokal ausfuehrbar
steps:
  - pwsh: ./build.ps1 -Aufgabe Alle
```

Wenn die Befehle in der Pipeline-Datei stehen, kann sie nur der Server ausprobieren; jede Änderung heißt Commit, Push und Warten. Das Skript lässt sich dagegen am eigenen Rechner so oft wiederholen, wie man will.

> **Praxisbeispiel**
>
> Ein Team ersetzt eine handgeschriebene Checkliste („vor dem Release: Syntax prüfen, Version erhöhen, Changelog ergänzen“) durch `build.ps1`. Das Skript läuft lokal vor dem Commit und in der Pipeline bei jedem Pull Request. Die Zahl der fehlerhaften Releases sinkt, weil kein Schritt mehr vergessen werden kann.

## 11.2 Pester in Pipelines

Ein *Test* ist ein kleines Programm, das prüft, ob anderer Code sich wie erwartet verhält. *Pester* ist das verbreitete Testframework für PowerShell. Tests sind das Sicherheitsnetz, das Änderungen überhaupt erst gefahrlos macht: Wer eine Funktion umbaut, sieht sofort, ob sie weiterhin dasselbe liefert. In einer Pipeline laufen die Tests bei jeder Änderung automatisch, und ein fehlgeschlagener Test verhindert die Auslieferung.

Wichtiger Hinweis zur Quellenlage: Pester war im Arbeitsbereich nicht installiert, die PowerShell Gallery war nicht erreichbar, und die Beispiele dieser Lerneinheit konnten nicht ausgeführt werden. Auch zu Pester liegt kein abgerufener Dokumentationsabschnitt vor. Alles Folgende beruht auf Fachwissen (Pester 5) und ist *nicht live verifiziert*; die Beispiele sind nur auf Syntax geprüft. Vor dem Einsatz sind Version und Parameter in der Pester-Dokumentation nachzuschlagen.

Ein Pester-Test hat eine feste Struktur. `Describe` fasst Tests zu einer Einheit zusammen (meist eine Funktion), `Context` gliedert weiter, `It` ist ein einzelner Testfall. Innerhalb von `It` steht eine Behauptung der Form `Wert | Should -Be Erwartung`. `BeforeAll` und `BeforeEach` bereiten Dinge vor (Funktionen laden, Testdaten anlegen). Damit der Test etwas zu prüfen hat, hier die Funktion, die getestet werden soll; sie wurde ausgeführt und lieferte „Hallo Welt“, „Hallo Anna“ und bei leerem Namen die Fehlerkennung `ParameterArgumentValidationError,Get-Gruss`.

```powershell
# src/Get-Gruss.ps1
function Get-Gruss {
    [CmdletBinding()]
    param([ValidateNotNullOrEmpty()] [string] $Name = 'Welt')
    "Hallo $Name"
}

# tests/Get-Gruss.Tests.ps1  (Pester 5, nur auf Syntax geprueft)
BeforeAll {
    . $PSScriptRoot/../src/Get-Gruss.ps1
}

Describe 'Get-Gruss' {
    It 'liefert den Standardgruss' {
        Get-Gruss | Should -Be 'Hallo Welt'
    }
    It 'nutzt den uebergebenen Namen' {
        Get-Gruss -Name 'Anna' | Should -Be 'Hallo Anna'
    }
    It 'lehnt einen leeren Namen ab' {
        { Get-Gruss -Name '' } | Should -Throw
    }
}
```

Die Funktion verlangt per `ValidateNotNullOrEmpty` einen nicht leeren Namen. Die Testdatei lädt sie in `BeforeAll`, indem sie die Quelldatei mit vorangestelltem Punkt einliest (*dot-sourcing*, siehe Modul 5); `$PSScriptRoot` macht den Pfad unabhängig vom aktuellen Verzeichnis. Die ersten beiden Testfälle vergleichen die Rückgabe mit dem erwarteten Text. Der dritte prüft, dass ein leerer Name einen Fehler auslöst: Der zu prüfende Code steht in einem Skriptblock in geschweiften Klammern, den `Should -Throw` ausführt und auf eine Ausnahme untersucht.

Eine bekannte Falle bei Pester 5 betrifft die zwei Phasen eines Testlaufs. Zuerst läuft die *Entdeckung* (Discovery), in der Pester die Struktur der Testdatei einliest, um zu wissen, welche Testfälle es gibt; danach folgt der *Lauf* (Run), in dem die `It`-Blöcke ausgeführt werden. Code, der außerhalb von `BeforeAll`, `BeforeEach` und `It` steht, läuft nur in der Entdeckung, nicht beim Lauf, und ist deshalb in den Testfällen nicht verfügbar. Wer Funktionen oder Variablen für Tests am Dateianfang definiert, statt in `BeforeAll`, erlebt, dass sie „plötzlich nicht gefunden“ werden. Das ist eine der häufigsten Anfängerfehler bei Pester 5; diese Beschreibung stammt aus Fachwissen.

Ein weiteres Kernmittel sind *Mocks*: Ersatzimplementierungen von Befehlen, die Tests von der Außenwelt trennen. Ein Test für eine Funktion, die `Get-AzResource` aufruft, soll nicht wirklich nach Azure telefonieren, sondern mit einer Attrappe arbeiten, die feste Beispieldaten liefert. Mit `Mock Get-AzResource { ... }` ersetzt Pester den Befehl innerhalb des Tests, und `Should -Invoke` prüft, wie oft er aufgerufen wurde. Gute testbare Funktionen sind deshalb so geschrieben wie in den Beispielen dieses Kurses: Die Logik arbeitet auf Objekten und holt Daten über einen austauschbaren Aufruf (so wie `Get-GraphSeiten` in Modul 10 den Abruf als Skriptblock bekam).

```powershell
# tests/Get-Gruss.Tests.ps1 (Fortsetzung) - nur auf Syntax geprueft
Describe 'Aufruf mit Mock' {
    BeforeAll {
        function Get-Daten { throw 'echter Aufruf waere hier' }
        function Show-Anzahl { (Get-Daten).Count }
        Mock Get-Daten { 1..3 }
    }
    It 'zaehlt die Eintraege der Attrappe' {
        Show-Anzahl | Should -Be 3
        Should -Invoke Get-Daten -Times 1 -Exactly
    }
}
```

Die Testgruppe definiert in `BeforeAll` eine Funktion, die in der Wirklichkeit eine Ausnahme wirft (als Stellvertreter für einen teuren oder gefährlichen Aufruf), und eine zweite, die sie benutzt. `Mock Get-Daten` ersetzt sie durch einen Skriptblock mit festen Daten. Der Test prüft die Anzahl und dass die Attrappe genau einmal aufgerufen wurde.

Wie läuft das in der Pipeline? Man ruft `Invoke-Pester` mit einer *Konfiguration* auf. In Pester 5 erzeugt man sie mit `New-PesterConfiguration`, setzt Eigenschaften wie den Testordner (`Run.Path`), das Ergebnisformat (`TestResult.OutputFormat`, zum Beispiel `NUnitXml`, das viele Pipeline-Systeme als Testbericht anzeigen) und den Pfad der Ergebnisdatei. Die Eigenschaft `Run.Exit = $true` lässt den Prozess mit einem Exit Code ungleich 0 enden, wenn Tests fehlschlagen: Genau das braucht die Pipeline. Alle diese Namen stammen aus Fachwissen und sind vor dem Einsatz zu verifizieren.

```powershell
# Nur auf Syntax geprueft (Pester 5, nicht ausgefuehrt)
$cfg = New-PesterConfiguration
$cfg.Run.Path                = './tests'
$cfg.Run.Exit                = $true            # Exit Code <> 0 bei Fehlern
$cfg.Output.Verbosity        = 'Detailed'
$cfg.TestResult.Enabled      = $true
$cfg.TestResult.OutputFormat = 'NUnitXml'
$cfg.TestResult.OutputPath   = './testResults.xml'
Invoke-Pester -Configuration $cfg
```

Die Konfiguration verweist auf den Ordner `tests`, schaltet den Exit Code ein, wählt eine ausführliche Textausgabe und lässt einen Bericht im NUnit-Format schreiben. Ruft das Build-Skript aus 11.1 diesen Block als weiteren Schritt auf, schlägt die Pipeline bei einem fehlgeschlagenen Test fehl, ohne dass jemand die Ausgabe lesen muss.

> **Ergänzung aus der Nachrecherche**
>
> **Was testet man und was nicht?** Gute Kandidaten sind reine Funktionen (Eingabe gleich Ausgabe), Entscheidungslogik (Tag-Prüfung, Versionsanhebung), Parameterprüfung und Fehlerfälle. Weniger lohnend sind Tests, die nur wiederholen, was der Code ohnehin tut, und Tests, die echte Cloud-Dienste brauchen (langsam, teuer, wackelig); dafür nimmt man Mocks oder trennt eine kleine Integrationsprüfung ab. Die Faustregel: Testen Sie die Stellen, deren Ausfall wehtut, und jeden Fehler, der einmal passiert ist, damit er nicht wiederkommt (siehe Modul 12, Lerneinheit 12.9).

> **Merke**
>
> Pester-Tests bestehen aus `Describe`/`Context`/`It` mit `Should`-Behauptungen. In Pester 5 gehört Vorbereitung in `BeforeAll`, nicht an den Dateianfang (Discovery gegen Run). Mocks trennen von der Außenwelt. In der Pipeline: `Run.Exit`, ein Ergebnisbericht und Aufruf aus dem Build-Skript. Diese Lerneinheit beruht auf Fachwissen und wurde nicht ausgeführt.

```powershell
# FALSCH (Pester 5): Hilfsfunktion ausserhalb von BeforeAll - im It-Block nicht verfuegbar
function Get-Wert { 42 }
Describe 'X' { It 'nutzt Get-Wert' { Get-Wert | Should -Be 42 } }

# RICHTIG: Vorbereitung in BeforeAll
Describe 'X' {
    BeforeAll { function Get-Wert { 42 } }
    It 'nutzt Get-Wert' { Get-Wert | Should -Be 42 }
}
```

Im falschen Beispiel steht die Funktion außerhalb der Phasen, die zur Laufzeit ausgeführt werden; unter Pester 5 führt das zu der Meldung, der Befehl sei nicht gefunden. Das richtige Beispiel hat die Vorbereitung dort, wo sie wirkt. Auch hier gilt: nicht lokal ausgeführt.

> **Praxisbeispiel**
>
> Ein Modul zur Benutzeranlage hat 40 Pester-Tests, die in zwei Sekunden laufen und bei jedem Pull Request ausgeführt werden. Als jemand die Namensregel ändert, schlägt ein Test fehl und zeigt, dass Umlaute in Benutzernamen plötzlich anders behandelt werden; der Fehler wird gefunden, bevor ein einziges Konto falsch angelegt ist.

## 11.3 PSScriptAnalyzer in CI

Tests prüfen, ob Code das Richtige tut. Die *statische Analyse* prüft, ob Code *gut geschrieben* ist, ohne ihn auszuführen: Nutzt er veraltete Befehle, speichert er Kennwörter im Klartext, verwendet er Aliase, die in Skripten schwer lesbar sind, oder fehlt bei einer verändernden Funktion die Unterstützung für `-WhatIf`? Das Werkzeug dafür heißt *PSScriptAnalyzer*. Es liefert Diagnoseeinträge mit Regelname, Schweregrad (Information, Warnung, Fehler), Datei und Zeilennummer. Es ist der Wächter für die Regeln aus Modul 12, Lerneinheit 12.3, und wird in der Pipeline als Qualitätsschranke eingesetzt.

Im Arbeitsbereich war PSScriptAnalyzer nicht installiert und nicht installierbar. Alles zu seiner Bedienung stützt sich daher auf die abgerufenen Seiten der Microsoft-Dokumentation, und die Aufrufe sind nur auf Syntax geprüft. Laut der Dokumentation zu `Invoke-ScriptAnalyzer` sind die wichtigsten Parameter: `-Path` (Datei oder Ordner) mit `-Recurse`, `-Settings` (eine Einstellungsdatei, eine Hashtable oder eine Voreinstellung), `-Severity` (nur bestimmte Schweregrade), `-ExcludeRule` (Regeln auslassen), `-ReportSummary` (Zusammenfassung), `-Fix` (einfache Befunde automatisch beheben) und `-EnableExit`. Mit `-EnableExit` beendet sich die PowerShell-Sitzung nach der Analyse mit einem Exit Code, der laut Dokumentation der Anzahl der Fehlerdatensätze entspricht; damit lässt sich die Pipeline direkt steuern. Eine Datei `PSScriptAnalyzerSettings.psd1` im Projektstamm wird laut Dokumentation implizit gefunden. Es gibt mitgelieferte Voreinstellungen (zum Beispiel `PSGallery`, `ScriptSecurity`, `CmdletDesign`, `ScriptFunctions`), und für Kompatibilitätsprüfungen die Regeln `PSUseCompatibleCmdlets`, `PSUseCompatibleCommands`, `PSUseCompatibleSyntax` und `PSUseCompatibleTypes`.

```powershell
# Nur auf Syntax geprueft - benoetigt das Modul PSScriptAnalyzer
Install-PSResource PSScriptAnalyzer -TrustRepository

# Lokal: alle Befunde anzeigen
Invoke-ScriptAnalyzer -Path ./src -Recurse -Settings ./PSScriptAnalyzerSettings.psd1 |
    Select-Object RuleName, Severity, ScriptName, Line, Message

# In der Pipeline: Exit Code = Anzahl der Fehlerdatensaetze
Invoke-ScriptAnalyzer -Path ./src -Recurse -Settings ./PSScriptAnalyzerSettings.psd1 `
    -Severity Error -EnableExit

# Einzelne Ausnahme im Code, mit Begruendung:
[System.Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSAvoidUsingWriteHost', '', Justification = 'Interaktives Hilfswerkzeug')]
param()
```

Das Beispiel zeigt die drei typischen Aufrufe. Lokal gibt man die Befunde als Tabelle aus, um sie zu beheben. In der Pipeline begrenzt man auf Fehler (`-Severity Error`) und lässt mit `-EnableExit` den Exit Code vom Ergebnis bestimmen; Warnungen blockieren dann den Lauf nicht. Der letzte Block zeigt, wie man eine Regel für eine bestimmte Stelle ausschaltet: mit dem Attribut `SuppressMessage`, Regelname und *Begründung*. Die Begründung ist wichtig, weil sie die Ausnahme später nachvollziehbar macht; eine Ausnahme ohne Grund ist eine versteckte Altlast. Die Dokumentation zeigt dieses Attribut mit dem Parameter `Justification`.

Damit man versteht, *was* eine solche Regel im Kern tut, lohnt ein Blick unter die Haube. Eine Regel ist nichts anderes als eine Abfrage auf dem Syntaxbaum, den Sie aus Modul 12, Lerneinheit 12.1, kennen. Das folgende Beispiel baut einen kleinen Analysator mit drei Regeln nach, der tatsächlich lief. Es ersetzt den echten PSScriptAnalyzer *nicht*, zeigt aber das Prinzip: Befund suchen, mit Datei, Zeile und Schweregrad melden und daraus einen Exit Code ableiten.

```powershell
function Invoke-MiniAnalyse {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad)
    $regeln = @(
        @{ Name = 'KeinInvokeExpression'; Schwere = 'Error'; Meldung = 'Invoke-Expression ist unsicher.'
           Test = { param($ast) $ast.FindAll({ param($n)
                      $n -is [System.Management.Automation.Language.CommandAst] -and
                      $n.GetCommandName() -in 'Invoke-Expression', 'iex' }, $true) } },
        @{ Name = 'KeinKlartextKennwort'; Schwere = 'Error'; Meldung = 'ConvertTo-SecureString -AsPlainText im Skript.'
           Test = { param($ast) $ast.FindAll({ param($n)
                      $n -is [System.Management.Automation.Language.CommandAst] -and
                      $n.GetCommandName() -eq 'ConvertTo-SecureString' -and
                      ($n.CommandElements.Extent.Text -contains '-AsPlainText') }, $true) } },
        @{ Name = 'KeinWriteHost'; Schwere = 'Warning'; Meldung = 'Write-Host schreibt nicht in den Erfolgsstrom.'
           Test = { param($ast) $ast.FindAll({ param($n)
                      $n -is [System.Management.Automation.Language.CommandAst] -and
                      $n.GetCommandName() -eq 'Write-Host' }, $true) } }
    )
    foreach ($datei in Get-ChildItem -LiteralPath $Pfad -Filter *.ps1 -Recurse -File) {
        $t = $null; $e = $null
        $ast = [System.Management.Automation.Language.Parser]::ParseFile($datei.FullName, [ref]$t, [ref]$e)
        foreach ($r in $regeln) {
            foreach ($treffer in & $r.Test $ast) {
                [pscustomobject]@{
                    Datei = $datei.Name; Zeile = $treffer.Extent.StartLineNumber
                    Regel = $r.Name; Schwere = $r.Schwere; Meldung = $r.Meldung
                }
            }
        }
    }
}

$befunde = @(Invoke-MiniAnalyse -Pfad ./src)
$befunde | Format-Table -AutoSize
$fehler = @($befunde | Where-Object Schwere -eq 'Error').Count
exit $(if ($fehler) { 20 } else { 0 })
```

Jede Regel ist eine Hashtable mit Name, Schweregrad, Meldung und einem Skriptblock, der im Syntaxbaum nach passenden Knoten sucht. `FindAll` liefert alle Knoten, für die die Bedingung gilt; hier sind es Befehlsaufrufe mit bestimmtem Namen. Die zweite Regel prüft zusätzlich, ob unter den Bestandteilen des Aufrufs der Schalter `-AsPlainText` vorkommt. Die Schleife wendet jede Regel auf jede Datei an und gibt für jeden Treffer ein Objekt mit Zeile aus. Am Ende zählt das Skript die Fehler und steuert den Exit Code: 20 bei mindestens einem Fehler, sonst 0; Warnungen verändern ihn nicht.

Im Test lag in einem Ordner eine Datei mit fünf Zeilen: ein Kommentar, der die Wörter „Invoke-Expression“ und „Write-Host“ enthält, ein `ConvertTo-SecureString` mit `-AsPlainText` in Zeile 2, ein `Write-Host` in Zeile 3, ein `Invoke-Expression` in Zeile 4 und eine Zeichenkette, die „Write-Host“ nur als Text enthält. Der Analysator meldete genau drei Befunde: `KeinInvokeExpression` in Zeile 4, `KeinKlartextKennwort` in Zeile 2 und `KeinWriteHost` (Warnung) in Zeile 3; der Kommentar und die Zeichenkette lösten *nichts* aus, weil der Syntaxbaum sie nicht als Befehlsaufruf kennt. Eine zweite, saubere Datei lieferte keinen Befund. Bei zwei Fehlern wäre der Exit Code 20 gewesen. Das ist der entscheidende Vorteil gegenüber einer Textsuche mit `Select-String`.

> **Ergänzung aus der Nachrecherche**
>
> **Mit Warnungen beginnen, Altlasten einplanen.** Führt man den Analyzer in einem bestehenden Projekt ein, melden sich oft hunderte Befunde. Wer sofort alle als Fehler wertet, blockiert die Arbeit und erntet Widerstand. Bewährt ist ein stufenweises Vorgehen: zuerst nur anzeigen, dann Fehler blockieren, die echte Risiken sind (Sicherheitsregeln), später weitere Regeln aufnehmen. Altbestand kann über eine Ausnahmeliste mit Frist geführt werden. Die abgerufene Dokumentation beschreibt die Kompatibilitätsregeln, mit denen sich außerdem prüfen lässt, ob Skripte unter einer bestimmten PowerShell-Version laufen würden; das ist hilfreich bei der Migration aus Modul 12, Lerneinheit 12.2.

> **Merke**
>
> Statische Analyse prüft Stil und Risiko ohne Ausführung. `Invoke-ScriptAnalyzer` mit `-Settings`, `-Severity` und `-EnableExit` (Exit Code = Anzahl Fehler) steuert Pipelines; Ausnahmen mit `SuppressMessage` und Begründung. Eine Regel ist eine Abfrage auf dem Syntaxbaum; deshalb ignoriert die Analyse Kommentare und Zeichenketten, die eine Textsuche fälschlich melden würde. Schrittweise einführen.

```powershell
# FALSCH: Ausnahme ohne Begruendung, gleich fuer die ganze Datei
[Diagnostics.CodeAnalysis.SuppressMessageAttribute('PSAvoidUsingWriteHost', '')]
param()

# RICHTIG: Ausnahme mit Begruendung, so eng wie moeglich
[System.Diagnostics.CodeAnalysis.SuppressMessageAttribute(
    'PSAvoidUsingWriteHost', '', Justification = 'Interaktives Hilfswerkzeug, kein Betriebsskript')]
param()
```

Eine Ausnahme ohne Begründung erklärt nicht, warum die Regel hier nicht gilt; in einem Jahr weiß es niemand mehr. Mit Begründung bleibt die Entscheidung prüfbar.

> **Praxisbeispiel**
>
> Ein Team schaltet den Analyzer in der Pipeline zuerst nur als Hinweis ein. Nach vier Wochen blockieren Sicherheitsregeln (Klartextkennwörter, `Invoke-Expression`) den Merge; ein Entwickler bemerkt so, dass eine für „schnell“ geschriebene Funktion ein Kennwort im Skript hatte, bevor sie in die Versionsverwaltung gelangt.

## 11.4 Modul-Publishing

Ein PowerShell-*Modul* bündelt Funktionen mit einem Manifest und einer Versionsnummer; *Publishing* heißt, es in ein *Repository* zu veröffentlichen, aus dem andere es mit einem einzigen Befehl installieren. Das öffentliche Repository ist die *PowerShell Gallery*. Firmen betreiben meist ein *privates* Repository (zum Beispiel in Azure Artifacts oder als Dateifreigabe), damit interne Module nicht öffentlich werden. Der Nutzen: Skripte hängen nicht mehr an Kopien auf Dateiservern, sondern an *versionierten* Paketen. Ein Rechner kann `Install-PSResource Firma.Werkzeuge -Version 2.1.0` sagen und bekommt immer denselben Stand.

Das Werkzeug für Veröffentlichung und Installation ist *PSResourceGet*, die moderne Nachfolgerin von PowerShellGet. Im Arbeitsbereich war `Microsoft.PowerShell.PSResourceGet` in Version 1.2.0 vorhanden. Wegen der gesperrten PowerShell Gallery konnte nicht dorthin veröffentlicht werden, wohl aber in ein *lokales Dateirepository* (ein Ordner), das PSResourceGet wie ein echtes Repository behandelt. So wurde der komplette Weg tatsächlich ausgeführt, nur ohne die Gallery.

Am Anfang steht das *Modulmanifest*, eine `.psd1`-Datei mit den Metadaten: Version, Autor, Beschreibung, exportierte Funktionen, Mindestversion von PowerShell, Lizenz und Projektadresse. `New-ModuleManifest` erzeugt sie. Laut der abgerufenen Dokumentation schreibt der Befehl die Datei als UTF-8 ohne BOM; im Test begannen die Bytes der Datei mit `23 0A 23`, also dem Zeichen `#` und einem Zeilenumbruch, ohne `EF BB BF`. `Test-ModuleManifest` prüft die Datei und lieferte im Test Name, Version und die exportierten Funktionen. Die Dokumentation weist außerdem darauf hin, dass die PowerShell Gallery bestimmte Metadaten verlangt; welche, ist in der Dokumentation des Katalogs nachzulesen.

```powershell
# Modul anlegen und veroeffentlichen (lokal ausgefuehrt gegen ein Dateirepository)
$modul = './proj/Demo'
New-Item -ItemType Directory $modul -Force | Out-Null
Set-Content "$modul/Demo.psm1" -Value @'
function Get-Gruss { [CmdletBinding()] param([string]$Name = 'Welt') "Hallo $Name" }
Export-ModuleMember -Function Get-Gruss
'@

New-ModuleManifest -Path "$modul/Demo.psd1" -RootModule 'Demo.psm1' -ModuleVersion '1.0.0' `
    -Author 'Team Betrieb' -Description 'Demomodul fuer den Kurs' `
    -FunctionsToExport 'Get-Gruss' -PowerShellVersion '7.0' -CompatiblePSEditions Core `
    -Tags 'Demo', 'Kurs' -LicenseUri 'https://example.org/lizenz' -ProjectUri 'https://example.org/demo'
Test-ModuleManifest "$modul/Demo.psd1"

# Lokales Repository registrieren und veroeffentlichen
Register-PSResourceRepository -Name Lokal -Uri (Join-Path $PWD 'repo') -Trusted
Publish-PSResource -Path $modul -Repository Lokal
Find-PSResource -Name Demo -Repository Lokal
```

Das Beispiel legt ein winziges Modul mit einer Funktion an, erzeugt das Manifest mit allen Angaben und prüft es. Dann registriert es einen Ordner als Repository mit dem Namen „Lokal“ (`-Trusted`, damit keine Rückfrage zur Vertrauenswürdigkeit kommt) und veröffentlicht das Modul dorthin. Im Test entstand im Ordner die Datei `Demo.1.0.0.nupkg` (ein Paket im NuGet-Format, das PSResourceGet erzeugt), und `Find-PSResource` fand das Modul in der Version 1.0.0. Für die echte PowerShell Gallery wäre statt des lokalen Repositorys der Name „PSGallery“ und ein API-Schlüssel (`-ApiKey`) nötig; die abgerufene Dokumentation nennt für `Publish-PSResource` außerdem die Parameter `-Repository`, `-Credential`, `-NupkgPath` (ein fertiges Paket veröffentlichen) und `-SkipDependenciesCheck`. Diese Wege wurden nicht ausgeführt.

Jede Veröffentlichung braucht eine *neue* Versionsnummer. Die PowerShell Gallery nimmt dieselbe Version kein zweites Mal an (Fachwissen, nicht abgerufen); das lokale Dateirepository im Test dagegen akzeptierte eine zweite Veröffentlichung derselben Version ohne Meldung, sodass ein Fehler dort unbemerkt bleibt. Beides spricht dafür, die Anhebung zu automatisieren und nicht von der Gnade des Repositorys abhängig zu machen. Die Nummer folgt dem Schema der semantischen Versionierung (Major.Minor.Patch). Ein häufiger Fehler ist, die Anhebung zu vergessen. Das Anheben lässt sich automatisieren; die folgende Funktion hebt Major, Minor oder Patch an und kann eine *Vorabversion* (Prerelease) setzen.

```powershell
function Step-ModulVersion {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $ManifestPfad,
        [Parameter(Mandatory)] [ValidateSet('Major', 'Minor', 'Patch')] [string] $Stufe,
        [string] $Vorabversion
    )
    $m = Import-PowerShellDataFile -LiteralPath $ManifestPfad
    $v = [version]$m.ModuleVersion
    $neu = switch ($Stufe) {
        'Major' { [version]::new($v.Major + 1, 0, 0) }
        'Minor' { [version]::new($v.Major, $v.Minor + 1, 0) }
        'Patch' { [version]::new($v.Major, $v.Minor, [math]::Max($v.Build, 0) + 1) }
    }
    if ($PSCmdlet.ShouldProcess($ManifestPfad, "Version $v -> $neu")) {
        Update-PSModuleManifest -Path $ManifestPfad -ModuleVersion $neu
        if ($Vorabversion) { Update-PSModuleManifest -Path $ManifestPfad -Prerelease $Vorabversion }
    }
    $neu.ToString() + $(if ($Vorabversion) { "-$Vorabversion" })
}

Step-ModulVersion './proj/Demo/Demo.psd1' Minor -WhatIf                       # zeigt 1.1.0, aendert nichts
Step-ModulVersion './proj/Demo/Demo.psd1' Minor -Vorabversion 'alpha1'        # 1.1.0-alpha1
Publish-PSResource -Path './proj/Demo' -Repository Lokal
```

Die Funktion liest die aktuelle Version aus dem Manifest, berechnet die neue nach der gewählten Stufe und schreibt sie mit `Update-PSModuleManifest` zurück; die Dokumentation nennt diesen Befehl samt seinem Parameter `-Prerelease`. Mit `-WhatIf` zeigt sie den geplanten Schritt, ohne die Datei zu ändern. Im Test zeigte der Probelauf „What if: ... Version 1.0.0 -\> 1.1.0“ und ließ die Datei bei 1.0.0. Der echte Aufruf mit `-Vorabversion alpha1` lieferte „1.1.0-alpha1“ und schrieb in das Manifest `ModuleVersion = 1.1.0` und `Prerelease = alpha1` (im Abschnitt `PrivateData.PSData`).

Zu Vorabversionen nennt die abgerufene Dokumentation klare Regeln: Die `ModuleVersion` muss aus *drei* Teilen bestehen, die Vorabbezeichnung steht in `PrivateData.PSData.Prerelease`, darf nur Buchstaben und Ziffern enthalten (das Format entspricht SemVer 1.0.0, also ohne Punkt und ohne Plus), und beim Suchen und Installieren braucht man den Schalter für Vorabversionen (für das ältere PowerShellGet `-AllowPrerelease`, bei PSResourceGet `-Prerelease`). Im Test bestätigte sich das Verhalten: Nach dem Veröffentlichen lagen im Repository `Demo.1.0.0.nupkg` und `Demo.1.1.0-alpha1.nupkg`. Eine normale Suche (`Find-PSResource Demo`) lieferte nur 1.0.0, die Suche mit `-Prerelease` zeigte 1.1.0 mit der Vorabbezeichnung alpha1. Anwender bekommen eine Vorabversion also nur, wenn sie ausdrücklich danach fragen, genau das Ziel dieser Einrichtung.

Das Installieren schließlich: `Install-PSResource` hat im Test die Parameter `-Scope` (CurrentUser oder AllUsers) und `-TrustRepository`, aber *keinen* Parameter `-Path`; wer in einen bestimmten Ordner herunterladen will, nimmt `Save-PSResource -Path`. Der Versuch mit `Install-PSResource -Path` scheiterte mit „A parameter cannot be found that matches parameter name ’Path“’. Mit `Save-PSResource Demo -Repository Lokal -Prerelease -Path ./inst -TrustRepository` entstand der Ordner `inst/Demo/1.1.0`, und nach `Import-Module ./inst/Demo` lieferte `Get-Gruss -Name 'Kurs'` die Antwort „Hallo Kurs“. Damit war der vollständige Kreis (anlegen, prüfen, veröffentlichen, finden, laden, benutzen) einmal real durchlaufen.

> **Ergänzung aus der Nachrecherche**
>
> **Veröffentlichen ist ein Vertrag.** Wer ein Modul veröffentlicht, verspricht, dass Nutzer sich auf die Versionsnummer verlassen können: Ein Patch ändert kein Verhalten, ein Minor fügt nur hinzu, ein Major darf Verträge brechen. Zur Disziplin gehören ein Changelog mit Einträgen pro Version, ein Test des Pakets vor dem Veröffentlichen (Modul laden, Funktionen aufrufen) und geschützte Zugangsdaten: Der API-Schlüssel für die PowerShell Gallery gehört in den Geheimnisspeicher der Pipeline, nie ins Repository. Eine einmal veröffentlichte Version lässt sich in der Gallery nur noch verbergen, nicht überschreiben; diese Aussage stammt aus Fachwissen und wurde nicht abgerufen. Das Change-Gate aus Modul 12, Lerneinheit 12.7, prüft Manifest-Version und Changelog automatisch vor der Freigabe.

> **Merke**
>
> Module werden mit Manifest (`New-ModuleManifest`, `Test-ModuleManifest`), dreiteiliger Version und mit `Publish-PSResource` in ein Repository veröffentlicht. Jede Veröffentlichung braucht eine neue Version. Vorabversionen stehen in `PrivateData.PSData.Prerelease` und werden nur mit `-Prerelease` gefunden. `Save-PSResource -Path` lädt in einen Ordner, `Install-PSResource` kennt kein `-Path`. Pipeline-Geheimnisse (API-Schlüssel) gehören in den Geheimnisspeicher.

```powershell
# FALSCH: gleiche Version nochmal veroeffentlichen, API-Schluessel im Skript
Publish-PSResource -Path ./proj/Demo -Repository PSGallery -ApiKey 'oy2abc123...'

# RICHTIG: Version anheben, Schluessel aus dem Geheimnisspeicher der Pipeline
Step-ModulVersion ./proj/Demo/Demo.psd1 Patch
Publish-PSResource -Path ./proj/Demo -Repository PSGallery -ApiKey $env:GALLERY_API_KEY
```

Die PowerShell Gallery lehnt eine bereits vorhandene Version ab (andere Repositories je nach Konfiguration nicht), und ein im Skript stehender Schlüssel landet in der Versionsverwaltung. Die richtige Variante hebt die Version an und liest den Schlüssel aus einer geschützten Umgebungsvariable der Pipeline (Befehle nur auf Syntax geprüft).

> **Praxisbeispiel**
>
> Das Team pflegt ein internes Modul `Firma.Werkzeuge` mit 30 Funktionen. Jede Änderung durchläuft die Pipeline (Analyse, Tests, Manifestprüfung); beim Merge in den Hauptzweig hebt ein Schritt die Patch-Version an und veröffentlicht in das private Repository. Skripte auf den Servern installieren feste Versionen, und ein Update ist eine bewusste Entscheidung statt eines Zufalls.

## 11.5 DSC und Desired State

Bisher haben die meisten Skripte *Schritte* beschrieben: „Lege den Ordner an, kopiere die Datei, starte den Dienst“. Ein anderer Ansatz beschreibt den *Zielzustand*: „Der Ordner existiert, die Datei hat diesen Inhalt, der Dienst läuft“. Das nennt man *deklarative* Konfiguration oder *Desired State*. Wie man dorthin kommt, entscheidet das Werkzeug: Ist der Zustand schon erreicht, tut es nichts; weicht er ab, bringt es ihn zurück. Der große Vorteil ist die *Idempotenz*, die Sie aus Modul 5 kennen: Man kann die Konfiguration beliebig oft anwenden, und das Ergebnis ist immer derselbe Zustand. Das ist die Grundlage für verlässliche, wiederholbare Infrastruktur.

*PowerShell Desired State Configuration* (DSC) ist die Umsetzung dieses Gedankens von Microsoft. DSC kennt *Ressourcen*, die je einen Aspekt eines Systems beschreiben (Datei, Dienst, Registrywert, Paket). Jede Ressource hat drei Operationen: *Get* liest den aktuellen Zustand, *Test* vergleicht ihn mit dem gewünschten Zustand und liefert Ja oder Nein, *Set* stellt den gewünschten Zustand her. Dieses Dreigespann ist das Herz von DSC.

Die abgerufene Dokumentation unterscheidet zwei Generationen. In der älteren Form (DSC 1.1 und 2.0) arbeitet man mit `Invoke-DscResource` und dem Parameter `-Method` mit den Werten `Get`, `Test` oder `Set`; eine Konfiguration wird in eine MOF-Datei übersetzt und vom *Local Configuration Manager* (LCM) auf dem Zielrechner angewendet. Der Aufruf von `Test` liefert laut Dokumentation ein Objekt mit der Eigenschaft `InDesiredState`, das anzeigt, ob der Ist- dem Sollzustand entspricht. In der neuen Form (DSC 3.0) ist DSC ein eigenständiges Befehlszeilenprogramm namens `dsc`, das nicht mehr an PowerShell gebunden ist: Man ruft etwa `dsc resource get`, `dsc resource test` oder `dsc resource set` mit dem Typ der Ressource (`--resource`) und der Eingabe als JSON (`--input`) auf, und `dsc config set` wendet eine ganze Konfiguration an. Die Dokumentation merkt an, dass `test` und `set` auf einen sogenannten synthetischen Test zurückfallen können (DSC vergleicht dann selbst Get-Ergebnis und Sollwert), und dass `dsc resource set` die Operation Set immer aufruft, auch wenn der Zustand schon stimmt; empfohlen wird deshalb, vorher einen Test auszuführen. Als Beispielressource nennt die Dokumentation `Microsoft.WinGet.DSC/WinGetPackage`. Das Befehlszeilenprogramm `dsc` war im Arbeitsbereich nicht vorhanden; die Aufrufe sind nicht ausgeführt.

```powershell
# DSC 3.0 Befehlszeile - nur auf Syntax geprueft, dsc war nicht installiert
dsc resource get  --resource Microsoft.WinGet.DSC/WinGetPackage --input '{"id":"Git.Git"}'
dsc resource test --resource Microsoft.WinGet.DSC/WinGetPackage --input '{"id":"Git.Git"}'
dsc resource set  --resource Microsoft.WinGet.DSC/WinGetPackage --input '{"id":"Git.Git"}'

# Aeltere Form (DSC 1.1 / 2.0) in PowerShell - nur auf Syntax geprueft
$eigenschaften = @{ DestinationPath = 'C:\Temp\demo.txt'; Contents = 'Hallo'; Ensure = 'Present' }
$test = Invoke-DscResource -Name File -ModuleName PSDesiredStateConfiguration `
            -Method Test -Property $eigenschaften
if (-not $test.InDesiredState) {
    Invoke-DscResource -Name File -ModuleName PSDesiredStateConfiguration `
        -Method Set -Property $eigenschaften
}
```

Der obere Block zeigt die drei Operationen der neuen Befehlszeile für ein Paket. Der untere zeigt das ältere Muster „erst testen, dann nur bei Abweichung setzen“ mit der Ressource `File`; die Parameter `-Name`, `-ModuleName`, `-Method` und `-Property` sind die von `Invoke-DscResource`, die Eigenschaftsnamen der Ressource `File` (`DestinationPath`, `Contents`, `Ensure`) stammen aus Fachwissen. Dass das Muster „erst Test, dann Set“ dem Kern von DSC entspricht, bestätigt die Dokumentation, die das Testen vor dem Setzen ausdrücklich empfiehlt.

Man muss nicht DSC installieren, um das Prinzip zu verstehen und selbst anzuwenden. Das folgende Beispiel baut den Gedanken in reinem PowerShell nach: eine Ressource „Zeile steht in Datei“ mit `Get`, `Test` und `Set`, die idempotent arbeitet und eine kleine Konfiguration anwendet. Sie lief tatsächlich.

```powershell
function Get-ZeileZustand {
    param([Parameter(Mandatory)] [string] $Pfad, [Parameter(Mandatory)] [string] $Zeile)
    $vorhanden = (Test-Path -LiteralPath $Pfad) -and (@(Get-Content -LiteralPath $Pfad) -contains $Zeile)
    [pscustomobject]@{ Pfad = $Pfad; Zeile = $Zeile; Ensure = if ($vorhanden) { 'Present' } else { 'Absent' } }
}

function Test-ZeileZustand {
    param([Parameter(Mandatory)] [string] $Pfad, [Parameter(Mandatory)] [string] $Zeile,
          [ValidateSet('Present', 'Absent')] [string] $Ensure = 'Present')
    (Get-ZeileZustand -Pfad $Pfad -Zeile $Zeile).Ensure -eq $Ensure
}

function Set-ZeileZustand {
    [CmdletBinding(SupportsShouldProcess)]
    param([Parameter(Mandatory)] [string] $Pfad, [Parameter(Mandatory)] [string] $Zeile,
          [ValidateSet('Present', 'Absent')] [string] $Ensure = 'Present')
    if (Test-ZeileZustand -Pfad $Pfad -Zeile $Zeile -Ensure $Ensure) { return }   # schon im Sollzustand
    if (-not $PSCmdlet.ShouldProcess($Pfad, "Zeile '$Zeile' $Ensure herstellen")) { return }
    if ($Ensure -eq 'Present') { Add-Content -LiteralPath $Pfad -Value $Zeile }
    else { Set-Content -LiteralPath $Pfad -Value (Get-Content -LiteralPath $Pfad | Where-Object { $_ -ne $Zeile }) }
}

function Invoke-Konfiguration {
    param([Parameter(Mandatory)] [hashtable[]] $Eintraege)
    foreach ($e in $Eintraege) {
        $vorher = Test-ZeileZustand @e
        if (-not $vorher) { Set-ZeileZustand @e }
        [pscustomobject]@{ Zeile = $e.Zeile; Ensure = $e.Ensure; VorherImSollzustand = $vorher; Geaendert = (-not $vorher) }
    }
}

$f = Join-Path ([IO.Path]::GetTempPath()) 'dsc-demo.conf'
$soll = @(
    @{ Pfad = $f; Zeile = 'PermitRootLogin no'; Ensure = 'Present' },
    @{ Pfad = $f; Zeile = 'X11Forwarding yes';  Ensure = 'Absent'  }
)
Invoke-Konfiguration $soll | Format-Table -AutoSize     # Lauf 1
Invoke-Konfiguration $soll | Format-Table -AutoSize     # Lauf 2: nichts mehr zu tun
```

Die drei Funktionen bilden die Operationen nach. `Get-ZeileZustand` stellt fest, ob die Zeile in der Datei vorkommt (und ob die Datei existiert), und liefert „Present“ oder „Absent“. `Test-ZeileZustand` vergleicht diesen Wert mit dem gewünschten. `Set-ZeileZustand` prüft zuerst mit dem Test, ob überhaupt etwas zu tun ist; nur dann fügt sie die Zeile an oder entfernt sie, und auch das nur, wenn `ShouldProcess` es erlaubt (Probelauf mit `-WhatIf`). `Invoke-Konfiguration` wendet eine Liste von Sollzuständen an und meldet je Eintrag, ob eine Änderung nötig war. Der Operator `@e` (Splatting) übergibt die Schlüssel der Hashtable als benannte Parameter.

Das Ergebnis im Test zeigt die Idempotenz: Im ersten Lauf wurde die Zeile `PermitRootLogin no` hinzugefügt (`Geaendert = True`); die Zeile `X11Forwarding yes` sollte fehlen und fehlte schon (`Geaendert = False`). Im zweiten Lauf war alles im Sollzustand, es änderte sich nichts. Dann wurde die unerwünschte Zeile `X11Forwarding yes` von Hand in die Datei geschrieben, um eine *Abweichung* (Drift) zu erzeugen: Der dritte Lauf erkannte sie (`VorherImSollzustand = False`), entfernte sie, und die Datei enthielt wieder nur `PermitRootLogin no`. Genau dieses „Zurückholen bei Abweichung“ ist der praktische Wert von Desired State.

> **Ergänzung aus der Nachrecherche**
>
> **Wann DSC, wann eigene Skripte?** DSC lohnt sich, wenn viele Rechner gleich konfiguriert werden sollen und der Zustand dauerhaft überwacht werden soll. Für einmalige Aufgaben, komplexe Abläufe mit Verzweigungen und Aufgaben, die über ein einzelnes System hinausgehen, sind Skripte oder die Orchestrierung mit Infrastrukturwerkzeugen (siehe 11.6) flexibler. Wichtig ist, den *Denkansatz* mitzunehmen: Auch ein gewöhnliches Skript wird robuster, wenn es zuerst prüft, ob etwas zu tun ist (Test), und erst dann ändert (Set). Welche DSC-Generation in einem Unternehmen im Einsatz ist, hängt von Zielplattform und Version ab; die genannten Aufrufe sind vor der Verwendung mit der Dokumentation der eingesetzten Version abzugleichen.

> **Merke**
>
> Desired State beschreibt den Zielzustand statt der Schritte. Eine Ressource hat Get (lesen), Test (vergleichen) und Set (herstellen). Idempotenz: mehrfach anwenden, gleiches Ergebnis. DSC 1.1/2.0 nutzt `Invoke-DscResource -Method` und LCM/MOF, DSC 3.0 die Befehlszeile `dsc resource get|test|set`; `dsc resource set` ruft Set immer auf, deshalb vorher testen. Das Muster „Test, dann nur bei Abweichung Set“ verbessert jedes Skript.

```powershell
# FALSCH: Zeile blind anhaengen - bei jedem Lauf kommt sie nochmal dazu
Add-Content $datei 'PermitRootLogin no'

# RICHTIG: erst pruefen, nur bei Abweichung aendern
if (-not (Test-ZeileZustand -Pfad $datei -Zeile 'PermitRootLogin no')) {
    Set-ZeileZustand -Pfad $datei -Zeile 'PermitRootLogin no'
}
```

Die falsche Variante füllt die Datei bei jedem Lauf mit weiteren Duplikaten; bei der richtigen ist das Ergebnis nach dem hundertsten Lauf dasselbe wie nach dem ersten.

> **Praxisbeispiel**
>
> Ein Team sorgt dafür, dass auf allen Servern bestimmte Sicherheitseinstellungen gelten. Statt eines Skripts mit Schritten beschreibt es die Sollwerte; ein täglicher Lauf prüft und korrigiert Abweichungen und meldet die korrigierten Fälle ins Protokoll. Ändert jemand von Hand eine Einstellung, fällt es am nächsten Tag auf und wird zurückgesetzt.

## 11.6 IaC-Orchestrierung

*Infrastructure as Code* (IaC) bedeutet, Infrastruktur (Netzwerke, Server, Datenbanken, Berechtigungen) nicht von Hand zu erstellen, sondern in Dateien zu beschreiben, die in der Versionsverwaltung liegen und automatisiert angewendet werden. Das Ergebnis sind reproduzierbare Umgebungen: Test und Produktion entstehen aus derselben Beschreibung, jede Änderung ist nachvollziehbar, und eine verlorene Umgebung lässt sich neu aufbauen. Es ist die logische Fortsetzung der Gedanken aus 11.5 (Desired State) und aus Modul 12, Lerneinheit 12.6 (Wiederherstellbarkeit).

Für IaC gibt es spezialisierte, deklarative Werkzeuge. *Terraform* (anbieterübergreifend) und *Bicep* (für Azure) sind die verbreiteten Beispiele. Man beschreibt den gewünschten Zustand, das Werkzeug berechnet einen *Plan* (was würde sich ändern?) und wendet ihn nach Freigabe an (*Apply*). Terraform führt dazu einen *Zustand* (State), der festhält, welche Ressourcen es verwaltet. Diese Einordnung stammt aus Fachwissen; Terraform und Bicep waren im Arbeitsbereich nicht installiert, ihre Dokumentation wurde nicht abgerufen, und konkrete Befehle sind als Muster zu verstehen.

Wozu braucht man dann PowerShell? Als *Klebstoff* (englisch *glue*) und *Orchestrierung*. Die deklarativen Werkzeuge erledigen die eigentliche Infrastruktur; PowerShell übernimmt, was drumherum anfällt: die Werkzeuge in der richtigen Reihenfolge aufrufen, Eingaben aus Key Vault oder Umgebungsvariablen beschaffen, die Ausgaben auswerten, vor dem Anwenden eine Freigabe einholen, nach dem Anwenden Dinge konfigurieren, die das Werkzeug nicht kann (Daten einspielen, Benutzer anlegen), und alles protokollieren. Dazu muss PowerShell externe Programme sicher aufrufen, ihren Exit Code auswerten und Ausgabe und Fehlerausgabe getrennt erfassen. Das haben Sie in Modul 9 mit `$LASTEXITCODE` angefangen; für Werkzeuge wie Terraform braucht man etwas Robusteres.

Für Terraform gilt laut allgemeinem Fachwissen eine nützliche Konvention: Der Befehl `terraform plan -detailed-exitcode` liefert 0, wenn keine Änderung nötig ist, 2, wenn Änderungen anstehen, und 1 bei einem Fehler. Daraus lässt sich ein Ablauf bauen: Plan ausführen; bei 0 ist nichts zu tun; bei 2 kommt die Freigabe und dann der Apply; bei 1 bricht der Ablauf ab. Weil das echte Terraform nicht verfügbar war, wurde der Ablauf mit einem *Ersatzprogramm* getestet, einem kleinen Shell-Skript, das genau diese Exit Codes liefert. Es ist kein Terraform, es prüft nur die PowerShell-Seite der Orchestrierung.

```powershell
function Invoke-Werkzeug {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Programm,
        [string[]] $Argumente = @(),
        [int[]] $ErlaubteCodes = @(0),
        [string] $Arbeitsverzeichnis = (Get-Location).Path
    )
    $psi = [System.Diagnostics.ProcessStartInfo]::new($Programm)
    foreach ($a in $Argumente) { $psi.ArgumentList.Add($a) }
    $psi.WorkingDirectory       = $Arbeitsverzeichnis
    $psi.RedirectStandardOutput = $true
    $psi.RedirectStandardError  = $true
    $p   = [System.Diagnostics.Process]::Start($psi)
    $out = $p.StandardOutput.ReadToEndAsync()
    $err = $p.StandardError.ReadToEndAsync()
    $p.WaitForExit()
    [pscustomobject]@{
        Code    = $p.ExitCode
        Ok      = ($p.ExitCode -in $ErlaubteCodes)
        Ausgabe = $out.Result.TrimEnd()
        Fehler  = $err.Result.TrimEnd()
    }
}

function Invoke-PlanUndAnwenden {
    [CmdletBinding(SupportsShouldProcess)]
    param([Parameter(Mandatory)] [string] $Werkzeug, [string] $Ordner = '.')
    # Konvention: plan -detailed-exitcode -> 0 keine Aenderung, 2 Aenderungen, 1 Fehler
    $plan = Invoke-Werkzeug -Programm $Werkzeug -Argumente 'plan', '-detailed-exitcode' `
                -ErlaubteCodes 0, 2 -Arbeitsverzeichnis $Ordner
    if (-not $plan.Ok) { throw "Plan fehlgeschlagen (Code $($plan.Code)): $($plan.Fehler)" }
    if ($plan.Code -eq 0) { return [pscustomobject]@{ Status = 'KeineAenderung'; Plan = $plan.Ausgabe } }
    if ($PSCmdlet.ShouldProcess($Ordner, 'apply')) {
        $apply = Invoke-Werkzeug -Programm $Werkzeug -Argumente 'apply', '-auto-approve' -Arbeitsverzeichnis $Ordner
        if (-not $apply.Ok) { throw "Apply fehlgeschlagen (Code $($apply.Code)): $($apply.Fehler)" }
        return [pscustomobject]@{ Status = 'Angewendet'; Plan = $plan.Ausgabe }
    }
    [pscustomobject]@{ Status = 'AenderungenOffen'; Plan = $plan.Ausgabe }
}
```

`Invoke-Werkzeug` startet ein Programm *ohne* Umweg über die Shell. Die Argumente werden einzeln in die `ArgumentList` eingetragen, sodass Leerzeichen und Sonderzeichen keine Rolle spielen (anders als beim Zusammenbauen einer Befehlszeile als Text, wo Eingaben zum Befehl werden könnten). Ausgabe und Fehlerausgabe werden *getrennt und asynchron* gelesen (`ReadToEndAsync`); das vermeidet eine bekannte Falle: Liest man zuerst vollständig die eine Ausgabe und das Programm füllt währenddessen den Puffer der anderen, blockieren beide Seiten gegenseitig (ein Deadlock). Am Ende wird der Exit Code mit der Liste erlaubter Codes verglichen und ein Ergebnisobjekt zurückgegeben. `Invoke-PlanUndAnwenden` setzt darauf den Ablauf auf: Plan mit erlaubten Codes 0 und 2, bei einem anderen Code Abbruch mit Meldung, bei 0 nichts tun, bei 2 `ShouldProcess` (damit `-WhatIf` und `-Confirm` gelten) und dann Apply.

Im Test lieferte das Ersatzprogramm je nach Modus drei Fälle. Bei „keine Änderung“ (Code 0) meldete die Funktion `KeineAenderung` mit dem Text „No changes.“. Bei „Änderung“ (Code 2) führte sie den Apply aus und meldete `Angewendet` mit dem Plantext „Plan: 1 to add“. Beim Fehlerfall (Code 1, Meldung „Error: boom“ auf der Fehlerausgabe) warf sie die Ausnahme „Plan fehlgeschlagen (Code 1): Error: boom“ und rührte den Apply nicht an. Mit `-WhatIf` erschien statt des Apply die Zeile „What if: Performing the operation "apply" on target ...“, der Status war `AenderungenOffen`, und das Ersatzprogramm wurde für den Apply nicht aufgerufen. Das ist genau das gewünschte Verhalten: Ein Fehler im Plan verhindert die Änderung, und der Probelauf ändert nichts.

Weitere Aspekte der Orchestrierung: Der *Zustand* von Terraform gehört in einen gemeinsamen, gesperrten Speicher (nicht auf den Laptop einer einzelnen Person), und Skripte dürfen nicht gleichzeitig am selben Zustand arbeiten; das ist Fachwissen. *Geheimnisse* übergibt man dem Werkzeug über Umgebungsvariablen oder Variablendateien, die nicht ins Repository gehören, und liest sie vorher aus dem Tresor (siehe Modul 10, Lerneinheit 10.6). Ausgaben des Werkzeugs, die als JSON vorliegen (bei Terraform nach Fachwissen `terraform output -json`), wandelt man mit `ConvertFrom-Json` in Objekte und verwendet sie in den Folgeschritten. Und der Plan gehört als *Artefakt* in die Pipeline, damit ein Mensch genau den geprüften Plan freigibt und nicht einen später berechneten.

> **Ergänzung aus der Nachrecherche**
>
> **Bicep und Azure PowerShell.** Für Azure bietet Az PowerShell Befehle zum Bereitstellen von Vorlagen (nach Fachwissen `New-AzResourceGroupDeployment`, das `-WhatIf` unterstützt und so eine Vorschau liefert) und ein Bicep-Werkzeug, das Bicep in Vorlagen übersetzt. Die genauen Parameter sind vor der Verwendung zu prüfen; diese Befehle wurden hier nicht ausgeführt und nicht aus der Dokumentation abgerufen. Das Muster bleibt gleich: erst Vorschau, dann Freigabe, dann Anwenden, am Ende Prüfung und Protokoll.

> **Merke**
>
> IaC beschreibt Infrastruktur als Code; Terraform und Bicep sind die deklarativen Werkzeuge, PowerShell orchestriert: Reihenfolge, Eingaben, Freigabe, Auswertung, Protokoll. Externe Programme startet man mit getrennter, asynchroner Ausgabeerfassung und prüft den Exit Code gegen eine Liste erlaubter Werte. Muster: Plan, bei Fehler abbrechen, bei Änderung Freigabe, dann Apply; `-WhatIf` ändert nichts.

```powershell
# FALSCH: Befehlszeile als Text zusammenbauen, Exit Code ignorieren, direkt anwenden
Invoke-Expression "terraform apply -auto-approve -var name=$eingabe"

# RICHTIG: Argumente einzeln, Exit Code pruefen, vorher planen
Invoke-PlanUndAnwenden -Werkzeug 'terraform' -Ordner ./infra -WhatIf
```

Die falsche Zeile erlaubt es, über die Eingabe Befehle einzuschleusen (zum Beispiel ein Semikolon mit weiterem Befehl), wendet ohne Plan an und prüft nichts. Die richtige trennt Planung, Freigabe und Anwendung (nur als Muster, nicht mit echtem Terraform ausgeführt).

> **Praxisbeispiel**
>
> Ein Team verwaltet seine Netzwerke mit Terraform. Eine Pipeline führt bei jedem Pull Request den Plan aus und hängt ihn als Kommentar an; nach dem Merge wendet sie genau diesen Plan an. Ein PowerShell-Skript davor holt die Zugangsdaten, danach spielt es Konfigurationsdaten ein und schreibt das Ergebnis ins Protokoll mit Run-ID.

## 11.7 Release-Strategien

Eine neue Version an alle Systeme gleichzeitig zu verteilen, ist die riskanteste Art, Software auszuliefern: Steckt ein Fehler darin, ist er überall. *Release-Strategien* verringern dieses Risiko, indem sie neue Versionen *stufenweise* ausrollen und nach jeder Stufe prüfen, ob alles gesund ist. Die wichtigsten Muster: Beim *Canary-Release* (Kanarienvogel) wird die neue Version zuerst an einem kleinen Teil der Ziele ausgerollt; zeigt er Probleme, bleibt der Schaden klein. Beim *Ring-Rollout* wird in mehreren Ringen ausgerollt: erst ein Testring, dann ein früher Ring, dann die Breite. Beim *Blue/Green-Verfahren* läuft die neue Version neben der alten, und man schaltet um, sobald sie sich bewährt hat; der Weg zurück ist ein Umschalten. *Feature-Schalter* (Feature Flags) liefern Code aus, aber aktivieren die neue Funktion erst später und gezielt.

Zu jeder Strategie gehören drei Dinge. Erstens eine *Gesundheitsprüfung*: ein objektives, automatisch auswertbares Kriterium (läuft der Dienst, sind Fehlerzahlen normal?), damit nicht nach Gefühl entschieden wird. Zweitens eine *Abbruchregel*: Ab welcher Fehlerquote wird gestoppt? Drittens ein *Rückweg*: Wie macht man die Änderung rückgängig, und wurde das ausprobiert? Das Grundprinzip der Ringe lässt sich in PowerShell kompakt ausdrücken. Das folgende Beispiel rollt durch eine Liste von Ringen, prüft nach jedem Ring und stoppt bei zu hoher Fehlerquote; Bereitstellung und Prüfung sind austauschbare Skriptblöcke.

```powershell
function Invoke-RingRollout {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [hashtable[]] $Ringe,                 # @{ Name='Canary'; Ziele=@('srv01') }
        [Parameter(Mandatory)] [scriptblock] $Bereitstellen,         # param($ziel)
        [Parameter(Mandatory)] [scriptblock] $Gesundheitspruefung,   # param($ziel) -> $true / $false
        [double] $MaxFehlerquote = 0.0,
        [scriptblock] $Warte = { Start-Sleep -Seconds 60 }
    )
    foreach ($ring in $Ringe) {
        $fehler = 0
        foreach ($ziel in $ring.Ziele) {
            if ($PSCmdlet.ShouldProcess($ziel, "Bereitstellen (Ring $($ring.Name))")) { & $Bereitstellen $ziel }
        }
        & $Warte
        foreach ($ziel in $ring.Ziele) { if (-not (& $Gesundheitspruefung $ziel)) { $fehler++ } }
        $quote = $fehler / [math]::Max(1, @($ring.Ziele).Count)
        [pscustomobject]@{
            Ring = $ring.Name; Ziele = @($ring.Ziele).Count; Fehler = $fehler
            Quote = $quote; Weiter = ($quote -le $MaxFehlerquote)
        }
        if ($quote -gt $MaxFehlerquote) {
            Write-Warning "Ring '$($ring.Name)' ueberschreitet die Fehlerquote - Rollout gestoppt."
            return
        }
    }
}
```

Die Funktion nimmt die Ringe als Liste von Hashtables mit Name und Zielen. Für jeden Ring stellt sie zuerst alle Ziele bereit (mit `ShouldProcess` für den Probelauf), wartet eine einstellbare Zeit, damit sich Probleme zeigen können, prüft dann jedes Ziel und berechnet die Fehlerquote. Sie gibt je Ring ein Ergebnisobjekt aus. Überschreitet die Quote die erlaubte Grenze, warnt sie, beendet die Funktion mit `return` und rollt die weiteren Ringe *nicht* aus. Die Wartezeit ist als Skriptblock austauschbar, damit der Test nicht wirklich wartet.

Der Test mit drei Ringen (Canary mit einem Server, früher Ring mit vier, Breite mit drei) und simulierter Bereitstellung ergab drei Fälle. Fall A (alles gesund): Alle drei Ringe liefen durch (Fehlerquote 0, `Weiter = True`), acht Ziele wurden bereitgestellt. Fall B (Server srv04 im frühen Ring krank, erlaubte Fehlerquote 0): Der Canary-Ring passierte, im frühen Ring betrug die Quote 0,25, die Funktion warnte „Ring ’Frueh’ ueberschreitet die Fehlerquote - Rollout gestoppt.“, und der breite Ring wurde nicht angefasst; insgesamt waren fünf Ziele bereitgestellt (srv01 bis srv05). Fall C (Canary krank): Schon im ersten Ring Quote 1, Abbruch, nur ein Ziel betroffen. Die Fälle zeigen den Zweck: Der Schaden bleibt auf den Ring begrenzt, in dem er auftritt.

Dazu gehört die Frage, *was* ausgeliefert wird: Eine Version bekommt eine eindeutige Nummer (semantische Versionierung, siehe 11.4), und jede Auslieferung referenziert genau diese Nummer, nie „die neueste“. Das macht Rückwege einfach: Rückgängig machen heißt, die vorherige Nummer erneut bereitzustellen. Bei Datenbanken und Zuständen ist der Rückweg schwieriger; Änderungen, die Daten verwandeln, brauchen ein getrenntes Vorgehen (erst erweitern, später aufräumen), das hier nicht vertieft wird.

> **Ergänzung aus der Nachrecherche**
>
> **Rückweg üben, Ringe sinnvoll schneiden.** Ein Rückweg, der nie ausprobiert wurde, ist ein Wunsch. Er gehört in dieselbe Übung wie die Wiederherstellung (Modul 12, Lerneinheit 12.6). Ringe schneidet man nach Risiko und Repräsentativität: Der Canary-Ring sollte unkritische, aber typische Systeme enthalten, der frühe Ring genug Vielfalt, um Probleme zu zeigen. Die Wartezeit zwischen den Ringen richtet sich danach, wie schnell Probleme sichtbar werden: Ein Fehler, der erst nach einem Tag auftritt, braucht eine Wartezeit von mehr als einem Tag. Dieses Vorgehen ist branchenübliches Betriebswissen und wurde hier nicht aus einer Quelle abgerufen.

> **Merke**
>
> Stufenweise ausrollen (Canary, Ringe, Blue/Green, Feature-Schalter) begrenzt den Schaden. Jede Stufe braucht eine automatische Gesundheitsprüfung, eine Abbruchregel und einen geübten Rückweg. Auslieferungen referenzieren feste Versionsnummern, nie „neueste“. Die Wartezeit zwischen den Ringen richtet sich danach, wie schnell Probleme sichtbar werden.

```powershell
# FALSCH: alles auf einmal, Pruefung nach Gefuehl
$alleServer | ForEach-Object { Install-Version -Ziel $_ -Version latest }

# RICHTIG: in Ringen, mit Pruefung, feste Version
Invoke-RingRollout -Ringe $ringe -MaxFehlerquote 0 `
    -Bereitstellen { param($z) Install-Version -Ziel $z -Version '2.1.0' } `
    -Gesundheitspruefung { param($z) Test-Dienst -Ziel $z }
```

Die falsche Variante rollt die „neueste“ Version überall aus, ohne Pause und Prüfung; ein Fehler trifft alle. Die richtige begrenzt ihn auf den ersten Ring (`Install-Version` und `Test-Dienst` sind hier Platzhalter für eigene Funktionen).

> **Praxisbeispiel**
>
> Ein Konfigurationsskript für 400 Server wird in drei Ringen ausgerollt: 5 Testserver, 40 frühe, 355 breite, jeweils mit einer Stunde Wartezeit und einer Prüfung der Dienste. Beim zweiten Ring bricht der Rollout ab, weil ein Dienst bei zwei Servern nicht startet; 45 statt 400 Server sind betroffen, und der Fehler ist am gleichen Tag behoben.

## 11.8 Policy as Code

Regeln, die nur im Handbuch stehen, werden vergessen oder verschieden ausgelegt. *Policy as Code* drückt Regeln als *prüfbaren Code* aus: Sie liegen im Repository, haben Versionen und Tests und werden automatisch auf Konfigurationen angewendet, bevor diese in Betrieb gehen. Beispiele: „Speicherkonten dürfen nur über HTTPS erreichbar sein“, „Ressourcen brauchen den Tag Owner“, „Server nur in genehmigten Regionen“, „keine öffentlichen Freigaben“. Statt dass jemand im Nachhinein feststellt, dass die Regel verletzt wurde, scheitert die Pipeline schon beim Pull Request. Das ist die konsequente Fortsetzung der Governance-Gedanken aus Modul 10 (Lerneinheit 10.7) und Modul 12 (Lerneinheit 12.3).

Es gibt Werkzeuge in mehreren Schichten. Dienste wie *Azure Policy* setzen Regeln in der Cloud selbst durch (siehe Modul 10), und ihre Definitionen und Zuweisungen lassen sich als Code im Repository führen. Allgemeine Regelmaschinen wie der *Open Policy Agent* (OPA) mit der Sprache Rego werten Regeln auf strukturierten Daten aus. Und man kann Regeln in PowerShell selbst schreiben: Eine Konfiguration als JSON einlesen und mit Regeln prüfen. Dass es Azure Policy und OPA gibt und wofür sie dienen, ist Fachwissen; beide wurden für dieses Modul nicht abgerufen und nicht ausgeführt. Das folgende Beispiel zeigt das Prinzip mit reinem PowerShell und lief tatsächlich.

```powershell
function Test-Richtlinie {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $KonfigPfad,
        [Parameter(Mandatory)] [hashtable[]] $Regeln
    )
    $objekte = Get-Content -LiteralPath $KonfigPfad -Raw | ConvertFrom-Json
    foreach ($o in $objekte) {
        foreach ($r in $Regeln) {
            if ($o.typ -ne $r.Typ) { continue }
            $ok = [bool](& $r.Pruefung $o)
            [pscustomobject]@{ Objekt = $o.name; Regel = $r.Name; Ok = $ok; Schwere = $r.Schwere }
        }
    }
}

$regeln = @(
    @{ Name = 'Speicher: nur HTTPS'; Typ = 'storage'; Schwere = 'Fehler'
       Pruefung = { param($o) $o.httpsOnly -eq $true } }
    @{ Name = 'Speicher: kein oeffentlicher Zugriff'; Typ = 'storage'; Schwere = 'Fehler'
       Pruefung = { param($o) -not $o.publicAccess } }
    @{ Name = 'VM: Tag Owner vorhanden'; Typ = 'vm'; Schwere = 'Warnung'
       Pruefung = { param($o) [bool]$o.tags.Owner } }
    @{ Name = 'VM: erlaubte Region'; Typ = 'vm'; Schwere = 'Fehler'
       Pruefung = { param($o) $o.region -in 'westeurope', 'northeurope' } }
)

$ergebnis = Test-Richtlinie -KonfigPfad ./konfig.json -Regeln $regeln
$ergebnis | Format-Table -AutoSize
$fehler = @($ergebnis | Where-Object { -not $_.Ok -and $_.Schwere -eq 'Fehler' })
exit $(if ($fehler.Count) { 20 } else { 0 })
```

Die Funktion liest eine JSON-Datei mit Konfigurationsobjekten (jedes hat einen Typ und einen Namen) und wendet auf jedes Objekt die Regeln an, die zu seinem Typ passen. Eine Regel ist eine Hashtable mit Name, Typ, Schweregrad und einem Skriptblock, der für ein Objekt Wahr oder Falsch liefert. Der Typ-Filter mit `continue` überspringt Regeln, die für den Typ nicht gelten. Die Umwandlung mit `[bool]` stellt sicher, dass immer ein Wahrheitswert herauskommt (ein fehlender Tag ergibt `$null`, das zu Falsch wird). Am Ende zählt das Skript nur Verstöße mit Schweregrad „Fehler“; Warnungen blockieren nicht. Das Ergebnis steuert den Exit Code für die Pipeline.

Im Test enthielt die Konfiguration zwei Speicherkonten und zwei VMs. Das Konto `st1` (nur HTTPS, kein öffentlicher Zugriff) erfüllte beide Regeln. Das Konto `st2` (kein HTTPS-Zwang, öffentlicher Zugriff) verletzte beide. Die VM `vm1` (Region westeurope, Tag Owner gesetzt) erfüllte beide Regeln; `vm2` (Region eastus, keine Tags) verletzte die Tag-Regel (Warnung) und die Regionsregel (Fehler). Gezählt wurden drei Verstöße der Schwere „Fehler“ (zwei bei `st2`, einer bei `vm2`), der Exit Code wäre 20 gewesen. Der Vorteil dieser Form: Jede Regel ist eine kleine, benannte und einzeln testbare Funktion, und eine neue Regel ist ein Eintrag mehr in der Liste.

Auch die Regeln selbst brauchen Tests. Eine Richtlinie, die fälschlich alles durchlässt, ist schlimmer als keine, weil sie Sicherheit vortäuscht. Man schreibt daher für jede Regel mindestens einen Fall, der bestehen, und einen, der durchfallen muss (mit Pester, siehe 11.2). Und die Regeln werden wie jeder Code per Pull Request geändert, mit Begründung; eine Lockerung einer Sicherheitsregel ist eine Entscheidung, die dokumentiert gehört.

> **Ergänzung aus der Nachrecherche**
>
> **Prüfen vor dem Anwenden und nach dem Anwenden.** Richtlinien wirken an zwei Stellen. *Vor* dem Anwenden prüft man die geplante Konfiguration (zum Beispiel den Plan aus 11.6 oder die JSON-Beschreibung einer Vorlage) und verhindert schon den Fehler. *Nach* dem Anwenden prüft man den tatsächlichen Zustand in der Umgebung, weil dort auch Dinge existieren, die nicht über die Pipeline entstanden sind (Tag-Prüfung aus Modul 10, Lerneinheit 10.7). Beides zusammen schließt die Lücken; die Vorabprüfung ist billiger, die Nachprüfung ehrlicher.

> **Merke**
>
> Policy as Code drückt Regeln als versionierten, testbaren Code aus und prüft Konfigurationen automatisch vor der Inbetriebnahme. Jede Regel hat Name, Typ, Schweregrad und eine Prüfung; Fehler blockieren, Warnungen nicht. Regeln brauchen eigene Tests (einer besteht, einer fällt durch) und werden per Pull Request geändert. Vor dem Anwenden (Plan) und danach (Ist-Zustand) prüfen.

```powershell
# FALSCH: Regel nur als Satz im Handbuch
# "Speicherkonten muessen HTTPS erzwingen."

# RICHTIG: Regel als Code, mit Test, in der Pipeline
Test-Richtlinie -KonfigPfad ./konfig.json -Regeln $regeln |
    Where-Object { -not $_.Ok -and $_.Schwere -eq 'Fehler' }   # darf nichts liefern
```

Der Satz im Handbuch verhindert nichts; die Regel als Code lässt die Pipeline scheitern, sobald jemand ein Konto ohne HTTPS-Zwang beschreibt.

> **Praxisbeispiel**
>
> Das Cloud-Team hat 25 Regeln als Code. Jeder Pull Request an der Infrastruktur wird dagegen geprüft; ein Entwickler, der ein Speicherkonto mit öffentlichem Zugriff beschreibt, sieht die Meldung schon im Pull Request und korrigiert, statt dass der Sicherheitsbeauftragte es Wochen später findet.

## 11.9 Übungsaufgaben zu Modul 11

Die Aufgaben steigen in der Schwierigkeit an. Die Aufgaben 3, 5, 6 und 7 sind praktisch (Code schreiben). Aufgaben, die Pester oder PSScriptAnalyzer betreffen, sind theoretisch, weil beide im Arbeitsbereich nicht verfügbar waren. Die Lösungen mit Erklärung folgen im nächsten Abschnitt.

**Aufgabe 1.** Erklären Sie die Begriffe Continuous Integration und Continuous Delivery und nennen Sie drei Vorteile davon, dass die Pipeline nur ein Build-Skript aufruft, statt die Logik in der YAML-Datei zu halten.

**Aufgabe 2.** Ihr Build-Skript endet mit `exit 20`, die Pipeline meldet aber „Erfolg“. Nennen Sie zwei mögliche Ursachen und beschreiben Sie, wie Sie beide beheben. Warum liefert ein `Write-Error` im `catch`-Block unter `$ErrorActionPreference = 'Stop'` nicht den erwarteten Exit Code?

**Aufgabe 3.** **Praxis:** Erweitern Sie den Mini-Analysator aus 11.3 um die Regel `KeineAliasse` (Schweregrad Warnung): Sie soll Aufrufe der Aliase `gci`, `ls`, `%`, `?`, `gc` und `iex` melden und den vollständigen Befehl nennen. Aliase in Kommentaren und Zeichenketten dürfen nicht gemeldet werden.

**Aufgabe 4.** Was ist an folgendem Pester-5-Test falsch, und wie lautet die Korrektur? Die Funktion `Get-Wert` wird am Dateianfang definiert, der `It`-Block ruft sie auf und meldet, der Befehl sei nicht gefunden. Erklären Sie die Ursache mit den Begriffen Discovery und Run.

**Aufgabe 5.** **Praxis:** Schreiben Sie eine Funktion `Get-NaechsteVersion`, die eine Versionszeichenkette der Form `1.4.2` (optional mit Vorabbezeichnung wie `-alpha1`) und eine Stufe (`Major`, `Minor`, `Patch`) erhält und die nächste Version als Text liefert. Eine optionale Vorabbezeichnung darf nur Buchstaben und Ziffern enthalten. Ungültige Eingaben sollen mit einer klaren Meldung scheitern.

**Aufgabe 6.** **Praxis:** Schreiben Sie nach dem Muster aus 11.5 eine idempotente Ressource für „Verzeichnis existiert“ (`Present`) beziehungsweise „Verzeichnis existiert nicht“ (`Absent`) mit `Test-VerzeichnisZustand` und `Set-VerzeichnisZustand`. `Set-` soll `-WhatIf` unterstützen und `$true` zurückgeben, wenn etwas geändert wurde, sonst `$false`.

**Aufgabe 7.** **Praxis:** Ergänzen Sie die Regelliste aus 11.8 um eine Regel für Speicherkonten: Die Mindest-TLS-Version (Feld `minTls`) muss `TLS1_2` oder `TLS1_3` sein; fehlt das Feld, gilt die Regel als verletzt. Beschreiben Sie, wie die Regel getestet werden sollte.

**Aufgabe 8.** Entwerfen Sie ein Ring-Rollout für ein Konfigurationsskript auf 200 Servern: Wie viele Ringe, welche Gesundheitsprüfung, welche Abbruchregel, wie lange Wartezeit und welcher Rückweg? Begründen Sie Ihre Entscheidungen.

## 11.10 Lösungen zu Modul 11

### Aufgabe 1.

Continuous Integration prüft jede Änderung automatisch (Syntax, Analyse, Tests), bevor sie mit der Arbeit anderer vermischt wird; Continuous Delivery liefert geprüfte Änderungen automatisch als Paket oder in die Zielumgebung aus. Vorteile des dünnen Pipeline-Aufrufs: (1) Fehler lassen sich lokal reproduzieren, ohne zu pushen und zu warten. (2) Das Skript ist unabhängig vom Anbieter, bei einem Wechsel ändert sich nur die kleine Hülle. (3) Das Skript wird wie jeder andere Code versioniert, geprüft und getestet, und die Logik ist nicht auf mehrere YAML-Dateien verteilt.

### Aufgabe 2.

Mögliche Ursachen: (a) Das Skript wird mit `pwsh -Command` gestartet, sodass nur 0 oder 1 ankommt (siehe Modul 9, Lerneinheit 9.3); Lösung: `pwsh -File` oder ein ausdrückliches `exit $LASTEXITCODE`. (b) Die Pipeline wertet den Exit Code nicht aus, etwa weil ein Eintrag wie `ignoreLASTEXITCODE` gesetzt ist; Lösung: Einstellung prüfen und entfernen. Eine weitere Möglichkeit ist ein nachgelagerter Schritt, der den Fehler „überschreibt“. Zum `Write-Error`: Mit `$ErrorActionPreference = 'Stop'` macht `Write-Error` den Fehler zu einer abbrechenden Ausnahme; steht der Aufruf im `catch`-Block, wird diese Ausnahme nicht mehr abgefangen, und das Skript endet mit Exit Code 1, ohne das `exit` zu erreichen. Im Test des Build-Skripts aus 11.1 war das der Fall; mit `Write-Error $_ -ErrorAction Continue` kam der erwartete Code 99.

### Aufgabe 3.

```powershell
$aliasse = @{ gci = 'Get-ChildItem'; ls = 'Get-ChildItem'; '%' = 'ForEach-Object'
              '?' = 'Where-Object'; gc = 'Get-Content'; iex = 'Invoke-Expression' }

$regel = @{
    Name = 'KeineAliasse'; Schwere = 'Warning'
    Test = { param($ast) $ast.FindAll({ param($n)
               $n -is [System.Management.Automation.Language.CommandAst] -and
               $aliasse.ContainsKey($n.GetCommandName()) }, $true) }
}

foreach ($treffer in & $regel.Test $ast) {
    "Zeile {0}: '{1}' -> {2}" -f $treffer.Extent.StartLineNumber,
        $treffer.GetCommandName(), $aliasse[$treffer.GetCommandName()]
}
```

Die Regel schlägt den Namen jedes Befehlsaufrufs in der Alias-Tabelle nach. Weil sie nur Befehlsknoten des Syntaxbaums betrachtet, ignoriert sie Kommentare und Zeichenketten. Im Test mit einer Datei, die in Zeile 1 einen Kommentar mit „gci“ und in Zeile 2 `gci . | ? { ... } | % { ... }` enthielt, meldete die Regel drei Treffer in Zeile 2 (`gci`, `?` und `%` mit ihren vollständigen Befehlen) und keinen für den Kommentar oder für Zeile 3 mit den ausgeschriebenen Befehlen. In den Mini-Analysator aus 11.3 fügt man die Regel als weiteren Eintrag der Regelliste ein.

### Aufgabe 4.

Pester 5 arbeitet in zwei Phasen: In der *Discovery* liest es die Testdatei ein, um die Struktur zu erfassen, in der Phase *Run* führt es die `It`-Blöcke aus. Code außerhalb von `BeforeAll`, `BeforeEach` und `It` läuft nur in der Discovery; die Funktion existiert daher beim Lauf nicht. Korrektur: Die Definition in `BeforeAll` verschieben (am Anfang des `Describe`-Blocks oder der Datei). Diese Aufgabe beruht auf Fachwissen zu Pester 5; Pester war im Arbeitsbereich nicht verfügbar.

### Aufgabe 5.

```powershell
function Get-NaechsteVersion {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Aktuell,
        [Parameter(Mandatory)] [ValidateSet('Major', 'Minor', 'Patch')] [string] $Stufe,
        [ValidatePattern('^[0-9A-Za-z]+$')] [string] $Vorabversion
    )
    if ($Aktuell -notmatch '^(\d+)\.(\d+)\.(\d+)(?:-([0-9A-Za-z]+))?$') {
        throw "Keine gültige Version: '$Aktuell'."
    }
    $ma = [int]$Matches[1]; $mi = [int]$Matches[2]; $pa = [int]$Matches[3]
    switch ($Stufe) {
        'Major' { $ma++; $mi = 0; $pa = 0 }
        'Minor' { $mi++; $pa = 0 }
        'Patch' { $pa++ }
    }
    $v = "$ma.$mi.$pa"
    if ($Vorabversion) { $v += "-$Vorabversion" }
    $v
}
```

Der reguläre Ausdruck verlangt genau drei Zahlen, optional gefolgt von Bindestrich und Vorabbezeichnung. Die Zahlen werden mit `$Matches` gelesen; je nach Stufe wird die passende erhöht und alle niedrigeren Stellen auf null gesetzt. Test: `1.4.2` ergab bei Patch `1.4.3`, bei Minor `1.5.0`, bei Major `2.0.0`; `1.4.2-alpha1` mit Minor und `-Vorabversion beta1` ergab `1.5.0-beta1`. Die Eingabe `1.4` scheiterte mit „Keine gültige Version: ’1.4’.“, und eine Vorabbezeichnung mit Punkt (`a.b`) wurde von `ValidatePattern` abgelehnt.

### Aufgabe 6.

```powershell
function Test-VerzeichnisZustand {
    param([Parameter(Mandatory)] [string] $Pfad,
          [ValidateSet('Present', 'Absent')] [string] $Ensure = 'Present')
    (Test-Path -LiteralPath $Pfad -PathType Container) -eq ($Ensure -eq 'Present')
}

function Set-VerzeichnisZustand {
    [CmdletBinding(SupportsShouldProcess)]
    param([Parameter(Mandatory)] [string] $Pfad,
          [ValidateSet('Present', 'Absent')] [string] $Ensure = 'Present')
    if (Test-VerzeichnisZustand -Pfad $Pfad -Ensure $Ensure) { return $false }
    if ($PSCmdlet.ShouldProcess($Pfad, "Verzeichnis $Ensure herstellen")) {
        if ($Ensure -eq 'Present') { $null = New-Item -ItemType Directory -Path $Pfad }
        else { Remove-Item -LiteralPath $Pfad -Recurse }
        return $true
    }
    $false
}
```

Der Test vergleicht, ob die tatsächliche Existenz mit dem gewünschten Zustand übereinstimmt. `Set-` tut nichts, wenn der Sollzustand schon gilt (Idempotenz), und ändert nur nach Erlaubnis von `ShouldProcess`. Test: Der erste Aufruf mit `Present` lieferte `True` (angelegt), der zweite `False`; das Entfernen mit `Absent` lieferte `True`, ein zweites Entfernen `False`. Mit `-WhatIf` erschien die Zeile „What if: Performing the operation "Verzeichnis Present herstellen" ...“, die Funktion gab `False` zurück, und das Verzeichnis existierte weiterhin nicht.

### Aufgabe 7.

```powershell
@{ Name = 'Speicher: Mindest-TLS 1.2'; Typ = 'storage'; Schwere = 'Fehler'
   Pruefung = { param($o)
       ($o.PSObject.Properties.Name -contains 'minTls') -and ($o.minTls -in 'TLS1_2', 'TLS1_3') } }
```

Die Regel prüft zuerst, ob das Feld überhaupt existiert (`PSObject.Properties.Name`); das ist wichtig, weil unter `Set-StrictMode` der Zugriff auf ein nicht vorhandenes Feld einen Fehler auslösen würde. Dann vergleicht sie den Wert mit den erlaubten. Im Test mit drei Konten ergab sich: `TLS1_2` gültig, `TLS1_0` ungültig, fehlendes Feld ungültig. Eine solche Regel testet man mit mindestens drei Fällen (gültiger Wert, ungültiger Wert, fehlendes Feld) und fügt sie als Pester-Test in das Repository ein, damit eine spätere Änderung der Regel nicht unbemerkt alles durchlässt.

### Aufgabe 8.

Ein möglicher Entwurf: drei Ringe, nämlich 2 Testserver (1 Prozent), 20 frühe Server (10 Prozent, unterschiedliche Standorte und Rollen) und der Rest von 178. Gesundheitsprüfung: Der Zieldienst läuft, das Konfigurationsskript meldete Exit Code 0, es gibt keine neuen Fehler im Protokoll (Auswertung nach Run-ID, siehe Modul 9). Abbruchregel: Fehlerquote größer als 0 im Test- und im frühen Ring (strenge Grenze, weil Konfigurationsfehler selten zufällig sind); im breiten Ring gegebenenfalls in Teilschritten von 50 Servern. Wartezeit: mindestens ein vollständiger Betriebszyklus, in dem Probleme sichtbar werden (etwa eine Stunde für sofort auftretende Fehler, bis zum nächsten Tag, wenn nachts laufende Aufgaben betroffen sind). Rückweg: Die vorherige Version (feste Versionsnummer) wird erneut bereitgestellt, und dieser Weg wurde vorher am Testring geübt. Die Begründung steht in der Abwägung zwischen Geschwindigkeit und Risiko: Je kritischer die Systeme, desto kleiner die Ringe und desto länger die Wartezeiten.
