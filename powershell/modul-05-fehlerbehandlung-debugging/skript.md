# Modul 05 - Fehlerbehandlung, Debugging und robuste Skripte

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (5.1 bis 5.9; 5.10 Übungsaufgaben, 5.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 12 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Übersichtsblatt (12 h) und Detailblatt (9 h) weichen um drei Stunden voneinander ab. Dieses Skript folgt dem Detailblatt, weil jede Lerneinheit dort einzeln aufgeführt ist. Die Differenz lässt sich mit den Übungen am Modulende füllen.

### Lernziele.

Nach diesem Modul können Sie Fehler in PowerShell präzise einordnen (nicht abbrechend, anweisungsabbrechend, skriptabbrechend, native Programme), mit `try`, `catch` und `finally` kontrolliert behandeln und Ressourcen sicher freigeben. Sie setzen den Strict Mode ein, um Tippfehler früh zu finden, debuggen Skripte mit Haltepunkten, trennen Nutzdaten von Diagnoseausgaben, schreiben strukturierte Logs mit Run-ID, bauen idempotente Skripte, testen Funktionen mit Pester und prüfen Code mit PSScriptAnalyzer. Dahinter steht ein Ziel: stabile, nachvollziehbare und wartbare Automation für Windows, Cloud und CI/CD.

| **Nr.** | **Lerneinheit**                | **Dauer** | **Prio** |
|:--------|:-------------------------------|:----------|:---------|
| 5.1     | Error Streams und ErrorAction  | 1 h       | 1        |
| 5.2     | try, catch, finally            | 1 h       | 1        |
| 5.3     | Strict Mode                    | 1 h       | 1        |
| 5.4     | Debugging in VS Code           | 1 h       | 1        |
| 5.5     | Verbose, Debug und Information | 1 h       | 1        |
| 5.6     | Logging-Patterns               | 1 h       | 1        |
| 5.7     | Idempotenz                     | 1 h       | 2        |
| 5.8     | Pester Grundlagen              | 1 h       | 2        |
| 5.9     | Codequalität                   | 1 h       | 2        |
|         | **Summe**                      | **9 h**   |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde.** Alle Beispiele zu Fehlern, Strict Mode, Streams, Logging, Idempotenz und Breakpoints wurden mit PowerShell 7.6.6 unter Linux ausgeführt; die genannten Meldungen stammen aus diesen Läufen (englischsprachige Systemmeldungen, wie sie PowerShell dort ausgibt). Pester und PSScriptAnalyzer ließen sich im Arbeitsbereich nicht installieren, weil der Zugriff auf die PowerShell Gallery gesperrt ist. Die Beispiele dazu (Lerneinheiten 5.8 und 5.9) wurden deshalb nur auf korrekte PowerShell-Syntax geprüft, nicht ausgeführt. Die Aussagen zu Fehlerarten, `try/catch`, `ErrorAction` und `$PSNativeCommandUseErrorActionPreference` wurden mit den Microsoft-Seiten „about_Error_Handling“, „about_Try_Catch_Finally“ und „Everything you wanted to know about exceptions“ abgeglichen (siehe Quellen).

## 5.1 Error Streams und ErrorAction

Fehler gehören zum Alltag jeder Automation: Eine Datei fehlt, ein Server antwortet nicht, ein Benutzer hat keine Berechtigung. Ob ein Skript daran scheitert oder sauber damit umgeht, entscheidet sich an einem Detail, das Einsteiger verblüfft: PowerShell kennt nicht *einen* Fehlerbegriff, sondern drei, die sich in der Wirkung stark unterscheiden. Wer sie verwechselt, schreibt entweder Skripte, die nach einem Fehler einfach weiterlaufen und falsche Ergebnisse liefern, oder Skripte, die bei jeder Kleinigkeit abbrechen.

Ein *nicht abbrechender Fehler* (non-terminating error) meldet ein Problem, stoppt aber weder den Befehl noch das Skript. Er entsteht zum Beispiel, wenn `Get-Content` unter mehreren Dateien eine nicht findet: Die übrigen werden trotzdem verarbeitet. Ein *anweisungsabbrechender Fehler* (statement-terminating error) beendet die aktuelle Anweisung, das Skript läuft aber an der nächsten Anweisung weiter. Typische Quellen sind ein nicht vorhandener Befehl, ein ungültiger Parameterwert oder eine .NET-Methode, die eine Ausnahme wirft, etwa `[int]::Parse('abc')`. Ein *skriptabbrechender Fehler* (script-terminating error) wickelt den gesamten Aufrufstapel ab und beendet das Skript, sofern ihn niemand abfängt. Das erzeugt vor allem das Schlüsselwort `throw`. Dazu kommt eine vierte Kategorie, die außerhalb dieses Systems liegt: Native Programme wie `git` oder `robocopy` melden Misserfolg nur über einen Exit Code, den PowerShell in `$LASTEXITCODE` ablegt.

Das erste Listing zeigt die drei Arten nebeneinander. Es erzeugt einen nicht abbrechenden Fehler mit `Write-Error`, einen anweisungsabbrechenden Fehler mit einem fehlerhaften .NET-Aufruf in einer Funktion und demonstriert, dass `catch` nur bei abbrechenden Fehlern anspringt.

```powershell
# 1. Nicht abbrechend: catch wird NICHT ausgefuehrt
try {
    Write-Error 'nicht abbrechend'
    'nach Write-Error'
} catch { 'nie erreicht' }

# 2. Anweisungsabbrechend: Funktion wird abgebrochen, Skript laeuft weiter
function Test-Anweisung {
    [int]::Parse('abc')
    'weiter in der Funktion'
}
Test-Anweisung
'nach Test-Anweisung'

# 3. Mit -ErrorAction Stop wird der Fehler abbrechend und fangbar
try {
    Write-Error 'jetzt abbrechend' -ErrorAction Stop
    'nie erreicht'
} catch {
    'gefangen: ' + $_.Exception.GetType().Name
}
```

Im ersten Block wird der Fehlertext angezeigt, aber `'nach Write-Error'` erscheint ebenfalls, und das `catch` bleibt unberührt, weil der Fehler nicht abbrechend ist. Im zweiten Block meldet PowerShell die Ausnahme „Exception calling "Parse" with "1" argument(s)“; die Funktion bricht an dieser Stelle ab, die Zeile `'weiter in der Funktion'` wird im Test jedoch trotzdem ausgeführt, denn ohne `try` wirkt der Fehler hier wie ein gemeldeter Fehler, dem das Skript folgt. Das ist einer der Gründe, weshalb Skripte ohne Fehlerbehandlung so unvorhersehbar sind. Im dritten Block macht `-ErrorAction Stop` aus dem Fehler eine Ausnahme vom Typ `WriteErrorException`, die das `catch` fängt.

Hinter diesem Verhalten steht der *Error Stream*. PowerShell kennt mehrere Ausgabeströme: Stream 1 (Success, die normale Ausgabe), 2 (Error), 3 (Warning), 4 (Verbose), 5 (Debug) und 6 (Information); Lerneinheit 5.5 vertieft die Ströme. Fehler wandern in Stream 2 und werden zusätzlich in die automatische Liste `$Error` eingetragen, in der der jüngste Fehler an Index 0 steht. Jeder Eintrag ist ein `ErrorRecord`-Objekt mit den Eigenschaften `Exception` (die eigentliche Ausnahme), `CategoryInfo` (Fehlerkategorie), `FullyQualifiedErrorId` (Kennung, nützlich zum gezielten Abfangen), `TargetObject` (das Objekt, bei dem der Fehler auftrat) und `ScriptStackTrace`.

### Das Verhalten steuern: `-ErrorAction` und `$ErrorActionPreference`.

Jeder Befehl mit `[CmdletBinding()]` versteht den allgemeinen Parameter `-ErrorAction`. Er legt für *diesen einen Befehl* fest, wie nicht abbrechende Fehler behandelt werden. `Continue` ist der Standard (anzeigen und weitermachen), `SilentlyContinue` unterdrückt die Anzeige, trägt den Fehler aber in `$Error` ein, `Ignore` unterdrückt beides, `Stop` verwandelt den Fehler in eine abbrechende Ausnahme, `Inquire` fragt nach, und `Break` öffnet den Debugger. Dieselben Werte nimmt die Präferenzvariable `$ErrorActionPreference` an, die für alle Befehle im aktuellen Gültigkeitsbereich gilt. Ein ausdrücklich angegebenes `-ErrorAction` hat Vorrang vor der Variable.

```powershell
# SilentlyContinue: kein Text, aber Eintrag in $Error und $? ist False
$Error.Clear()
Get-Item /nicht-da -ErrorAction SilentlyContinue
"? = $?, Fehler in Liste: $($Error.Count)"

# Ignore: auch kein Eintrag in $Error
$Error.Clear()
Get-Item /nicht-da -ErrorAction Ignore
"Fehler in Liste: $($Error.Count)"

# ErrorVariable sammelt Fehler getrennt von $Error
Get-Item /nicht-da -ErrorVariable ev -ErrorAction SilentlyContinue
$ev[0].FullyQualifiedErrorId
$ev[0].CategoryInfo.Category

# Gezielt auf den Fehlertyp reagieren
try {
    Get-Item /nicht-da -ErrorAction Stop
} catch [System.Management.Automation.ItemNotFoundException] {
    "Nicht gefunden: $($_.TargetObject)"
}
```

Die Ausgabe des Tests lautete: `? = False, Fehler in Liste: 1` bei `SilentlyContinue`, danach `Fehler in Liste: 0` bei `Ignore`. Die Kennung des Fehlers war `PathNotFound,Microsoft.PowerShell.Commands.GetItemCommand`, die Kategorie `ObjectNotFound`. Das letzte `catch` greift, weil `-ErrorAction Stop` den Fehler in eine Ausnahme vom Typ `ItemNotFoundException` verwandelt und das Objekt `$_.TargetObject` den Pfad nennt. `-ErrorVariable` sammelt Fehler getrennt von der globalen Liste; mit einem vorangestellten Pluszeichen (`-ErrorVariable +ev`) hängt PowerShell neue Fehler an, statt die Variable zu überschreiben.

### Native Programme.

Programme außerhalb von PowerShell sprechen nicht die Sprache der ErrorRecords. Eine Anweisung wie `/bin/sh -c 'exit 3'` (unter Windows etwa `cmd /c exit 3`) setzt lediglich `$LASTEXITCODE` auf 3 und `$?` auf `False`. Es entsteht kein Eintrag in `$Error`, und ein `catch` greift nicht; im Test lief die Zeile `'kein catch'` nach dem fehlgeschlagenen Programm unbeeindruckt weiter. Seit PowerShell 7.4 (zuvor als experimentelles Feature in 7.3) gibt es die Präferenzvariable `$PSNativeCommandUseErrorActionPreference`. Setzt man sie auf `$true`, erzeugt jeder Exit Code ungleich 0 einen Fehler vom Typ `NativeCommandExitException`, der den Regeln von `$ErrorActionPreference` folgt.

```powershell
/bin/sh -c 'exit 3'
"LASTEXITCODE=$LASTEXITCODE  ?=$?"

$PSNativeCommandUseErrorActionPreference = $true
$ErrorActionPreference = 'Stop'
try {
    /bin/sh -c 'exit 3'
    'nie erreicht'
} catch {
    $_.Exception.GetType().Name
    $_.Exception.Message
}
```

Der Test ergab `LASTEXITCODE=3  ?=False` und danach im `catch` die Ausgabe `NativeCommandExitException` und die Meldung „Program "sh" ended with non-zero exit code: 3 (0x03).“ Mit `$ErrorActionPreference = 'Continue'` (Standard) wird derselbe Fehler nur angezeigt, und das Skript läuft weiter, auch das wurde im Test bestätigt. Nicht jedes Programm benutzt Exit Codes im Sinn von „0 ist Erfolg“: `robocopy` meldet mit Werten unter 8 Erfolg in verschiedenen Varianten. Für solche Programme prüft man `$LASTEXITCODE` ausdrücklich.

> **Ergänzung aus der Nachrecherche**
>
> **ErrorAction ist nicht gleich ErrorActionPreference (laut about_Error_Handling).** Das Handbuch weist auf eine Asymmetrie hin, die selbst erfahrene Anwender überrascht. Der Parameter `-ErrorAction` wirkt nur auf *nicht abbrechende* Fehler; ruft eine Funktion `$PSCmdlet.ThrowTerminatingError()` auf, kann der Aufrufer den Fehler mit `-ErrorAction SilentlyContinue` nicht unterdrücken. Die Variable `$ErrorActionPreference` wirkt dagegen auch auf anweisungsabbrechende Fehler und kann sie mit `SilentlyContinue` oder `Ignore` sogar unterdrücken. Ebenfalls laut Handbuch bleibt ein mit `Stop` eskalierter Fehler in *erweiterten* Funktionen (mit `[CmdletBinding()]`) anweisungsabbrechend, in einfachen Skriptblöcken wird er skriptabbrechend. Im Test endete ein einfacher Skriptblock mit `$ErrorActionPreference = 'Stop'` und Division durch null den ganzen Aufruf mit Exit Code 1, während derselbe Block mit `[CmdletBinding()]` anschließend weiterlief (die Zeile nach dem Block wurde ausgegeben).

Eine Falle, die im Test sichtbar wurde: `$?` beschreibt nur den *letzten Befehl*. Lässt man `Get-Content` in einem `ForEach-Object`-Block laufen und eine der Dateien fehlt, ist `$?` nach der Pipeline trotzdem `True`, denn `ForEach-Object` selbst war erfolgreich. Wer Fehler sicher erkennen will, braucht `-ErrorAction Stop` mit `try/catch` oder `-ErrorVariable`.

> **Merke**
>
> Drei Fehlerarten: nicht abbrechend (Skript läuft weiter, `catch` greift nicht), anweisungsabbrechend (aktuelle Anweisung endet) und skriptabbrechend (`throw`). `-ErrorAction Stop` macht einen Fehler fangbar. `Ignore` unterdrückt auch den Eintrag in `$Error`. Native Programme melden nur über `$LASTEXITCODE`; mit `$PSNativeCommandUseErrorActionPreference` (ab 7.4) werden daraus Fehler.

```powershell
# FALSCH: catch erwartet einen abbrechenden Fehler, bekommt aber keinen
try {
    Get-Content -LiteralPath 'C:\daten\fehlt.txt'
    'Datei gelesen'
} catch {
    'Fehler behandelt'
}

# RICHTIG: Fehler ausdruecklich abbrechend machen
try {
    $text = Get-Content -LiteralPath 'C:\daten\fehlt.txt' -ErrorAction Stop
} catch {
    Write-Warning "Datei nicht lesbar: $($_.Exception.Message)"
}
```

Im falschen Beispiel zeigt PowerShell den Fehler an, das `catch` bleibt unberührt und die Meldung `Datei gelesen` erscheint sogar, obwohl nichts gelesen wurde. Im richtigen Beispiel zwingt `-ErrorAction Stop` den Fehler in das `catch`, und die Variable `$text` wird nur bei Erfolg gesetzt.

> **Praxisbeispiel**
>
> Ein Skript soll 200 Benutzerkonten aus einer Liste bearbeiten. Ein Fehler bei einem einzelnen Konto darf nicht die übrigen 199 blockieren. Deshalb arbeiten Administratoren hier mit `-ErrorAction Stop` pro Konto in einem `try/catch` innerhalb der Schleife, sammeln die Fehler in einer Liste und melden am Ende eine Zusammenfassung. Nur bei Fehlern, die den ganzen Lauf sinnlos machen (zum Beispiel keine Verbindung zum Verzeichnisdienst), bricht das Skript mit `throw` insgesamt ab.

## 5.2 try, catch, finally

Wer weiß, dass etwas schiefgehen kann, behandelt den Fehler an der Stelle, an der er auftritt. PowerShell bietet dafür die Anweisung `try` mit den Blöcken `catch` und `finally`. Der `try`-Block enthält den Code, der überwacht werden soll. Tritt darin ein abbrechender Fehler auf, springt die Ausführung in den passenden `catch`-Block. Der `finally`-Block läuft immer, egal ob ein Fehler auftrat, ob er gefangen wurde oder ob das Skript mit `exit` oder Strg+C beendet wird. Eine `try`-Anweisung braucht mindestens einen `catch`- oder einen `finally`-Block.

Im `catch`-Block steht der aktuelle Fehler in der automatischen Variable `$_` (gleichbedeutend `$PSItem`) als `ErrorRecord`. Die Meldung erhält man mit `$_.Exception.Message`, die Position im Skript mit `$_.InvocationInfo.PositionMessage`, die Aufrufkette mit `$_.ScriptStackTrace`. Wird `$_` in einen Text eingesetzt, erscheint die kürzeste lesbare Fassung des Fehlers.

```powershell
function Test-Fin {
    try {
        'try'
        throw 'boom'
    } catch {
        'catch: ' + $_.Exception.Message
        return 'Rueckgabewert'
    } finally {
        'finally laeuft auch nach return'
    }
}
Test-Fin
```

Die Ausgabe zeigt die Reihenfolge: `try`, dann `catch: boom`, dann `Rueckgabewert` und schließlich `finally laeuft auch nach return`. Der `finally`-Block wird also selbst dann ausgeführt, wenn der `catch`-Block die Funktion mit `return` verlässt. Dasselbe gilt für `exit`: Im Test beendete `exit 5` im `catch` den Prozess mit dem Exit Code 5, und die Ausgabe von `finally` erschien trotzdem. Das macht `finally` zum richtigen Ort für Aufräumarbeiten.

### Mehrere `catch`-Blöcke und Fehlertypen.

Ein `catch`-Block kann auf einen oder mehrere .NET-Ausnahmetypen beschränkt werden: `catch [System.IO.FileNotFoundException]`. Mehrere Typen trennt man mit Kommas. PowerShell prüft die Blöcke von oben nach unten, und es wird höchstens einer ausgeführt. Ein `catch` ohne Typ fängt alles, was übrig bleibt. Da Ausnahmetypen voneinander erben (`FileNotFoundException` erbt von `IOException`), gehört der spezifischere Typ vor den allgemeineren. Interessant ist: PowerShell erkennt die falsche Reihenfolge schon beim Einlesen des Skripts. Im Test lehnte der Parser ein Skript mit `catch [IOException]` vor `catch [FileNotFoundException]` mit der Meldung „Exception type System.IO.FileNotFoundException is already handled by a previous handler.“ ab, noch bevor die erste Zeile lief.

```powershell
try {
    throw [System.IO.FileNotFoundException]::new('Datei fehlt', 'x.txt')
}
catch [System.IO.DirectoryNotFoundException], [System.IO.FileNotFoundException] {
    'Pfad oder Datei fehlt: ' + $_.Exception.FileName
}
catch [System.IO.IOException] {
    'Anderer E/A-Fehler'
}
catch {
    'Unbekannter Fehler: ' + $_.Exception.Message
}
```

Hier trifft der erste Block zu, und das Beispiel gibt `Pfad oder Datei fehlt: x.txt` aus. Die Eigenschaft `FileName` existiert, weil die Ausnahme ein Objekt mit eigenen Eigenschaften ist; die .NET-Dokumentation nennt für jeden Typ die verfügbaren Eigenschaften und Konstruktoren.

### Fehler weitergeben.

Wer einen Fehler nur protokollieren, aber nicht selbst behandeln kann, gibt ihn mit dem alleinstehenden Schlüsselwort `throw` weiter. Dadurch bleiben Skriptname, Zeile und Aufrufkette des Ursprungs erhalten. Auch `throw $_` behielt im Test die ursprüngliche Position, doch ein leeres `throw` im `catch` drückt die Absicht („unverändert weitergeben“) am klarsten aus. Wirft man dagegen einen neuen Text, ist die Ursprungsposition verloren. Wer stattdessen eine neue Ausnahme werfen will, packt die ursprüngliche als innere Ausnahme hinein (`[System.InvalidOperationException]::new('Konfiguration fehlerhaft', $_.Exception)`), damit sie über `$_.Exception.InnerException` erreichbar bleibt.

```powershell
function Inner { throw 'innen' }
function Outer {
    try { Inner }
    catch {
        'Log: ' + $_
        throw
    }
}
try { Outer }
catch {
    'Aussen: ' + $_.Exception.Message
    ($_.ScriptStackTrace -split "`n")[0]
}
```

Im Test gab die Funktion zuerst `Log: innen` aus und das äußere `catch` dann `Aussen: innen`; die erste Zeile des Stacks zeigte `at Inner` mit der Zeilennummer, wo `throw` ursprünglich stand. Ohne Weitergabe oder mit einem neuen `throw` mit anderem Text ginge diese Information verloren.

### Fehler für Aufrufer: `ThrowTerminatingError`.

Ein `throw` in einer Funktion verweist in der Fehlermeldung auf die Zeile des `throw`, also auf das Innere der Funktion. Für Benutzer, die die Funktion nur aufrufen, ist die Zeile des *Aufrufs* aussagekräftiger. Dafür gibt es in erweiterten Funktionen `$PSCmdlet.ThrowTerminatingError()` mit einem selbst gebauten `ErrorRecord`. Die Meldung zeigt dann auf den Aufruf, und die Kennung (hier `Res.NotFound,Get-Res`) enthält den Funktionsnamen.

```powershell
function Get-Res {
    [CmdletBinding()]
    param([string] $Path)
    $fehler = [System.Management.Automation.ErrorRecord]::new(
        [System.IO.FileNotFoundException]::new("Nicht gefunden: $Path"),
        'Res.NotFound',
        [System.Management.Automation.ErrorCategory]::ObjectNotFound,
        $Path)
    $PSCmdlet.ThrowTerminatingError($fehler)
}

Get-Res -Path 'x'
'weiter nach dem Aufruf'
try { Get-Res -Path 'x' } catch { $_.FullyQualifiedErrorId }
```

Beim Test blieb der Fehler beim ersten Aufruf ein *anweisungsabbrechender* Fehler: Er wurde mit Verweis auf die Aufrufzeile angezeigt, und `'weiter nach dem Aufruf'` erschien trotzdem. Im `try`-Block dagegen fing das `catch` den Fehler und gab `Res.NotFound,Get-Res` aus. Das Beispiel zeigt, warum man öffentliche Funktionen mit `try/catch` und `ThrowTerminatingError` absichert: Der Aufrufer entscheidet über `-ErrorAction`, wie hart der Fehler wirkt.

### Ressourcen freigeben.

Dateien, Datenbankverbindungen, Netzwerksockets und temporäre Ordner müssen auch im Fehlerfall freigegeben werden. Das ist die klassische Aufgabe von `finally`. Das folgende Beispiel öffnet einen Schreibstrom in eine temporäre Datei, löst absichtlich einen Fehler aus und stellt sicher, dass der Strom geschlossen und die Datei gelöscht wird.

```powershell
$tmp = New-TemporaryFile
$writer = $null
try {
    $writer = [System.IO.StreamWriter]::new($tmp.FullName)
    $writer.WriteLine('Zwischenergebnis')
    throw 'Fehler mitten in der Verarbeitung'
}
catch {
    Write-Warning "Abbruch: $_"
}
finally {
    if ($writer) { $writer.Dispose() }
    Remove-Item -LiteralPath $tmp.FullName -ErrorAction SilentlyContinue
    'aufgeraeumt: ' + (-not (Test-Path -LiteralPath $tmp.FullName))
}
```

Die Variable `$writer` wird vor dem `try` auf `$null` gesetzt, damit `finally` auch dann nicht scheitert, wenn schon das Öffnen misslang. Im Test erschien die Warnung „Abbruch: Fehler mitten in der Verarbeitung“, danach `aufgeraeumt: True`. Beachten Sie die Reihenfolge: zuerst Strom schließen, dann Datei löschen, denn unter Windows lässt sich eine geöffnete Datei nicht löschen.

Zum Schluss sei das ältere `trap` erwähnt. Es fängt abbrechende Fehler im ganzen Gültigkeitsbereich und kann mit `continue` den Lauf fortsetzen. Im Test gab `trap` mit `continue` die Meldung aus und das Skript lief mit der nächsten Anweisung weiter. In neuen Skripten ist `try/catch` lesbarer, weil der geschützte Bereich sichtbar umrahmt ist; `trap` begegnet Ihnen in älteren Skripten.

> **Ergänzung aus der Nachrecherche**
>
> **Empfehlung aus der Microsoft-Dokumentation zum Melden von Fehlern.** Das Handbuch „about_Error_Handling“ empfiehlt je nach Schwere den passenden Mechanismus. Wenn eine Funktion mehrere Eingaben verarbeitet und einzelne scheitern dürfen, verwendet man `Write-Error` (in erweiterten Funktionen besser `$PSCmdlet.WriteError()`, damit `$?` im Aufrufer korrekt auf `$false` steht). Kann die Funktion gar nicht weiterarbeiten, der Aufrufer soll aber entscheiden, nimmt man `$PSCmdlet.ThrowTerminatingError()`. Muss das gesamte Skript enden, nimmt man `throw`. Ein `catch` soll außerdem nur Fehler abfangen, die man auch wirklich behandelt; das bloße Verschlucken („Swallowing“) erschwert die Fehlersuche erheblich.

> **Merke**
>
> `try` überwacht, `catch` behandelt, `finally` räumt immer auf (auch bei `return`, `exit` und Strg+C). Spezifische Typen zuerst. Ein leeres `throw` im `catch` gibt den Fehler mit Ursprungsinformation weiter. Für Aufrufer: `ThrowTerminatingError`. Nur fangen, was man behandelt.

```powershell
# FALSCH: Fehler verschlucken
try {
    Remove-Item -LiteralPath $ordner -Recurse -ErrorAction Stop
} catch { }
'Ordner geloescht'

# RICHTIG: Fehler melden und Folge ziehen
try {
    Remove-Item -LiteralPath $ordner -Recurse -ErrorAction Stop
} catch {
    Write-Error "Ordner konnte nicht geloescht werden: $($_.Exception.Message)"
    exit 1
}
'Ordner geloescht'
```

Im falschen Beispiel meldet das Skript Erfolg, obwohl der Ordner vielleicht noch existiert, und niemand erfährt den Grund. Im richtigen wird der Fehler gemeldet, und der Exit Code zeigt dem Scheduler oder der Pipeline den Misserfolg an.

> **Praxisbeispiel**
>
> Ein nächtliches Sicherungsskript öffnet eine Netzwerkfreigabe, kopiert Dateien und trennt die Verbindung wieder. Fällt das Kopieren aus, muss die Verbindung trotzdem getrennt werden, sonst blockiert sie am nächsten Tag das Laufwerk. Das Trennen steht deshalb im `finally`; im `catch` wird der Fehler protokolliert und der Exit Code auf 1 gesetzt, damit das Monitoring Alarm schlägt.

## 5.3 Strict Mode

PowerShell verzeiht vieles. Greift ein Skript auf eine nie gesetzte Variable zu, liefert PowerShell ohne Murren `$null`; fragt man eine nicht vorhandene Eigenschaft ab, kommt ebenfalls `$null`. Das ist beim interaktiven Arbeiten bequem, in Skripten aber tückisch, denn ein einziger Tippfehler im Variablennamen kann ein Skript dazu bringen, mit leeren Werten weiterzuarbeiten, zum Beispiel einen Löschbefehl mit einem leeren Pfad auszuführen. Der *Strict Mode* schaltet diese Nachsicht ab: PowerShell meldet dann solche Zugriffe als Fehler.

Man aktiviert ihn mit `Set-StrictMode -Version Latest` am Anfang des Skripts (oder mit einer festen Versionsnummer) und deaktiviert ihn mit `Set-StrictMode -Off`. Der Modus gilt für den aktuellen Gültigkeitsbereich und die darunterliegenden. Welche Prüfungen aktiv sind, hängt von der Version ab: Version 1.0 verbietet den Zugriff auf nicht initialisierte Variablen *außer innerhalb von Zeichenketten*. Version 2.0 verbietet zusätzlich den Zugriff auf nicht vorhandene Eigenschaften eines Objekts, den Aufruf einer Funktion mit Methodensyntax (`F(1,2)`) und verlangt, dass Variablen in Zeichenketten ebenfalls initialisiert sind. Die neueren Versionen verschärfen weiter: Der Test zeigte, dass unter `Latest` auch ein Zugriff auf einen Index außerhalb eines Arrays (`$arr[10]`) und auf `$null.Length` oder `'text'.Count` einen Fehler auslöst.

```powershell
$obj = [pscustomobject]@{ A = 1 }
$arr = 1, 2, 3

Set-StrictMode -Off
"ohne Strict Mode: [$undefiniert] [$($obj.B)] [$($arr[10])]"

Set-StrictMode -Version 1.0
"Version 1.0: [$undefiniert2]"          # in Zeichenketten erlaubt

Set-StrictMode -Version Latest
try { $x = $nichtGesetzt }
catch { $_.Exception.Message }
try { $obj.B }
catch { $_.Exception.Message }
try { $arr[10] }
catch { $_.Exception.Message }
try { $null.Length }
catch { $_.Exception.Message }
Set-StrictMode -Off
```

Im Test gab die Zeile ohne Strict Mode `[] [] []` aus, ebenso die Zeile mit Version 1.0, weil in Zeichenketten Variablen weiter leer bleiben dürfen. Mit `Latest` erschienen die Meldungen „The variable ’\$nichtGesetzt’ cannot be retrieved because it has not been set.“, „The property ’B’ cannot be found on this object. Verify that the property exists.“, „Index was outside the bounds of the array.“ und zum Schluss für `$null.Length` wieder „The property ’Length’ cannot be found on this object.“ Die letzte Zeile ist eine Folge, die man kennen muss: Auch der ansonsten bequeme Zugriff auf `.Count` oder `.Length` eines `$null`-Werts oder einer Zeichenkette, die man als Array betrachtet, scheitert unter Strict Mode. Hash-Tabellen sind besonders betroffen: `if ($h.Port)` ist für eine Hashtable ohne Schlüssel `Port` im Test ebenfalls ein Fehler; die Alternative ist `$h.ContainsKey('Port')`.

Ein typischer Nutzen zeigt sich bei Tippfehlern. Wer `$serverName` setzt, aber später `$servername` schreibt, hat in PowerShell kein Problem, denn Variablennamen unterscheiden Groß-/Kleinschreibung nicht. Schreibt er dagegen `$sevrerName`, bekommt er ohne Strict Mode einen leeren Wert, mit Strict Mode eine klare Fehlermeldung mit Zeilennummer.

### Kompatibilitätsfolgen.

Der Strict Mode ist ein Gewinn, aber keine kostenlose Einstellung. Skripte, die mit lockeren Zugriffen geschrieben sind, brechen plötzlich ab; vor allem Code, der absichtlich prüft, ob eine Eigenschaft existiert (`if ($obj.Eigenschaft)`), muss auf `$obj.PSObject.Properties['Eigenschaft']` oder `ContainsKey` umgestellt werden. Die Einstellung `Latest` bedeutet außerdem, dass neuere PowerShell-Versionen strengere Prüfungen mitbringen können. Wer ein Skript für Wartungsjahre schreibt, legt sich besser auf eine feste Version wie `3.0` fest und hebt sie bewusst an. Der Modus wirkt zudem nur im Gültigkeitsbereich, in dem er gesetzt wurde: Ein Skript, das `Set-StrictMode` aufruft, beeinflusst die aufrufende Sitzung nicht, ein Modul, das ihn setzt, nur sein eigenes Innenleben.

```powershell
Set-StrictMode -Version 3.0

function Get-Port {
    param([hashtable] $Konfig)
    if ($Konfig.ContainsKey('Port')) { $Konfig['Port'] } else { 443 }
}

function Test-Eigenschaft {
    param($Objekt, [string] $Name)
    $null -ne $Objekt.PSObject.Properties[$Name]
}

Get-Port -Konfig @{ Name = 'srv01' }
Test-Eigenschaft -Objekt ([pscustomobject]@{ A = 1 }) -Name 'B'
Test-Eigenschaft -Objekt ([pscustomobject]@{ A = 1 }) -Name 'A'
```

Beide Hilfsfunktionen sind Strict-Mode-fest. `Get-Port` prüft mit `ContainsKey` die Existenz des Schlüssels, statt ihn einfach abzufragen. `Test-Eigenschaft` nutzt die Sammlung `PSObject.Properties`, die für nicht vorhandene Namen `$null` zurückgibt, ohne einen Fehler auszulösen. Die Ausgabe lautet `443`, `False`, `True`.

> **Merke**
>
> `Set-StrictMode -Version Latest` (oder eine feste Version) am Skriptanfang macht Zugriffe auf unbekannte Variablen, Eigenschaften und Indizes zu Fehlern. Version 1.0 prüft Variablen in Zeichenketten nicht. Existenzprüfungen schreibt man mit `ContainsKey` oder `PSObject.Properties`. Der Modus gilt nur im eigenen Gültigkeitsbereich.

```powershell
# FALSCH: Tippfehler bleibt unbemerkt, leerer Pfad wird zum Loeschbefehl
$ziel = 'C:\Temp\Export'
Remove-Item -Path "$zeil\*" -Recurse -WhatIf

# RICHTIG: Strict Mode meldet den Tippfehler sofort
Set-StrictMode -Version Latest
Remove-Item -Path "$zeil\*" -Recurse -WhatIf
```

Im ersten Fall wird der Pfad zu `\*` verkürzt, also dem Wurzelverzeichnis des aktuellen Laufwerks. Das Beispiel nutzt `-WhatIf`, damit nichts passiert; ohne diese Absicherung wäre der Schaden erheblich. Der Strict Mode im zweiten Fall verbietet Variablen in Zeichenketten, die nie gesetzt wurden, erst ab Version 2.0 (auch `Latest` umfasst das) und meldet den Fehler, bevor der Befehl ausgeführt wird.

> **Praxisbeispiel**
>
> Teams schreiben `Set-StrictMode -Version Latest` und `$ErrorActionPreference = 'Stop'` in ihre Skriptvorlage (Modul 4). Dadurch fallen Tippfehler und stillschweigend leere Werte bereits im Test auf und nicht erst im Betrieb, wo sie zum Beispiel ein Backup mit leerem Zielpfad erzeugen würden.

## 5.4 Debugging in VS Code

Fehlermeldungen sagen, *dass* etwas schiefging; ein Debugger zeigt, *warum*. Mit ihm hält man ein Skript an einer Stelle an, betrachtet Variablen, geht Zeile für Zeile weiter und findet so Fehler, die man sonst durch viele `Write-Host`-Zeilen einkreisen müsste. In der Praxis arbeitet man mit Visual Studio Code und der Erweiterung „PowerShell“ von Microsoft, die den Debugger der Shell an die grafische Oberfläche anbindet.

Das Grundprinzip beruht auf *Haltepunkten* (Breakpoints). Klickt man in VS Code in den Rand neben einer Zeilennummer, erscheint ein roter Punkt. Startet man die Datei mit F5 („Start Debugging“), hält die Ausführung an dieser Zeile an. Dann stehen mehrere Steuerbefehle zur Verfügung: „Continue“ (F5) läuft bis zum nächsten Haltepunkt, „Step Over“ (F10) führt die aktuelle Zeile aus, ohne in aufgerufene Funktionen hineinzugehen, „Step Into“ (F11) springt in die aufgerufene Funktion, „Step Out“ (Umschalt+F11) läuft bis zum Ende der aktuellen Funktion, und „Stop“ (Umschalt+F5) bricht ab. Im Bereich „Variables“ sieht man die Werte der lokalen Variablen, im Bereich „Watch“ beobachtet man selbst gewählte Ausdrücke, im Bereich „Call Stack“ die Kette der Aufrufe. Im Debug-Konsolenfenster lassen sich beliebige PowerShell-Ausdrücke auswerten, etwa `$summe` oder `$z.GetType()`.

Neben einfachen Haltepunkten bietet VS Code *bedingte Haltepunkte*: Mit einem Rechtsklick auf den Punkt legt man eine Bedingung wie `$z -eq 3` fest, sodass der Debugger nur dann anhält. Ein *Logpoint* gibt eine Meldung aus, ohne anzuhalten, und ein *Trefferzähler* lässt den Punkt erst beim n-ten Durchlauf auslösen. Das ist unschätzbar in Schleifen mit tausenden Durchläufen, in denen nur ein Element Probleme macht.

Auch ohne grafische Oberfläche gibt es einen vollwertigen Debugger in der Shell, den die Erweiterung nur bedient. Die Cmdlets `Set-PSBreakpoint`, `Get-PSBreakpoint`, `Remove-PSBreakpoint`, `Enable-PSBreakpoint` und `Disable-PSBreakpoint` setzen und verwalten Haltepunkte nach Zeile, Befehl oder Variable. Der Befehl `Wait-Debugger` hält ein Skript an der Stelle an, an der er steht; das ist hilfreich, wenn man ein Skript nicht aus VS Code startet, sondern in einer Pipeline oder remote debuggen will.

Das folgende Beispiel verwendet ein kleines Skript `berechne.ps1`, das die Summe einer Zahlenliste bildet. Wir hängen mit `Set-PSBreakpoint` einen Haltepunkt mit *Aktion* an Zeile 5. Eine Aktion ist ein Skriptblock, der beim Erreichen des Punktes ausgeführt wird, ohne dass das Skript anhält, vergleichbar mit einem Logpoint.

```powershell
# Inhalt von berechne.ps1
function Get-Summe {
    param([int[]] $Zahlen)
    $summe = 0
    foreach ($z in $Zahlen) {
        $summe += $z
    }
    $summe
}
Get-Summe -Zahlen 1, 2, 3
```

```powershell
$bp = Set-PSBreakpoint -Script ./berechne.ps1 -Line 5 -Action {
    Write-Host "BP: z=$z summe=$summe"
}
. ./berechne.ps1
$bp.HitCount
Get-PSBreakpoint | Remove-PSBreakpoint
```

Beim Test erschienen drei Zeilen `BP: z=1 summe=0`, `BP: z=2 summe=1` und `BP: z=3 summe=3`, danach das Ergebnis `6`. `HitCount` stand nach dem Lauf auf 3, es wurde also bei jedem Schleifendurchlauf einmal ausgelöst. Beachten Sie, dass die Aktion die Ausgabe mit `Write-Host` schreibt: Normale Rückgabewerte im Aktionsblock erschienen im Test nicht auf der Konsole. Zum Schluss entfernt `Remove-PSBreakpoint` alle Punkte, denn Haltepunkte bleiben bis zum Ende der Sitzung aktiv.

Für das Debugging in VS Code legt man eine Startkonfiguration an (Datei `.vscode/launch.json`). Die Erweiterung erzeugt sie bei Bedarf; das folgende Beispiel startet die aktuell geöffnete Datei mit Argumenten.

```powershell
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "PowerShell: aktuelle Datei mit Argumenten",
      "type": "PowerShell",
      "request": "launch",
      "script": "${file}",
      "args": ["-Quelle", "./daten", "-Ziel", "./sicherung"],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

Die Einträge sind: `type` wählt den Debugger der PowerShell-Erweiterung, `request` legt fest, dass ein Skript gestartet (und nicht an einen laufenden Prozess angehängt) wird, `script` nennt die Datei (`\${file}` steht für die aktuell geöffnete), `args` sind die Parameter für das Skript, `cwd` das Arbeitsverzeichnis. Die Konfiguration `"request": "attach"` verbindet dagegen mit einem bereits laufenden PowerShell-Prozess, den man vorher mit `Enter-PSHostProcess` betritt. Diese Zeilen wurden nicht in einer grafischen Oberfläche ausgeführt, sondern aus der Dokumentation der Erweiterung bekannt; sie gelten als Fachwissen und sollten mit der installierten Version der Erweiterung abgeglichen werden.

### Fehler reproduzierbar eingrenzen.

Ein Debugger hilft nur, wenn man den Fehler wiederholen kann. Dazu gehört ein Dreischritt: erstens den Fehler mit festen Eingabedaten auslösen, nicht mit Livedaten; zweitens das Skript so verkleinern, bis nur noch der fehlerhafte Teil übrig ist (ein *Minimalbeispiel*); drittens vermuten, prüfen, widerlegen. Wer den Fehler eingrenzt, indem er die Hälfte des Codes auskommentiert und nachsieht, ob er noch auftritt (Halbierungsverfahren), findet ihn auch in langen Skripten in wenigen Schritten. Hilfreich sind außerdem `Set-PSDebug -Trace 1`, das jede ausgeführte Zeile anzeigt (im Test erschienen Zeilen wie `DEBUG: 13+ >>>> $a = 1 + 1`), und `Get-PSCallStack`, das die Aufrufkette ausgibt (im Test `B`, `A`, `<ScriptBlock>`).

> **Merke**
>
> Haltepunkte (Klick am Zeilenrand), F5/F10/F11 zum Steuern, Variablen- und Watch-Bereich zum Beobachten. Bedingte Haltepunkte und Logpoints für Schleifen. In der Shell: `Set-PSBreakpoint`, `Wait-Debugger`, `Set-PSDebug -Trace 1`, `Get-PSCallStack`. Fehler zuerst reproduzierbar machen und verkleinern, dann debuggen.

```powershell
# FALSCH: Debug-Ausgaben ueberall verstreuen und im Skript vergessen
Write-Host "HIER 1"
$daten = Get-Daten
Write-Host "HIER 2: $($daten.Count)"

# RICHTIG: Bedingten Haltepunkt setzen oder Write-Debug nutzen
$daten = Get-Daten
Write-Debug "Anzahl Datensaetze: $($daten.Count)"
```

Die `Write-Host`-Zeilen landen in der Ausgabe, stören Folgeprozesse und werden gern im produktiven Code vergessen. `Write-Debug` erscheint nur, wenn man `-Debug` angibt oder `$DebugPreference` setzt (Lerneinheit 5.5), und gehört dauerhaft in den Code.

> **Praxisbeispiel**
>
> Ein Skript verarbeitet 5 000 Benutzer, und für genau einen schlägt die Verarbeitung fehl. Statt 5 000 Mal anzuhalten, setzt man einen bedingten Haltepunkt mit `$benutzer.Name -eq 'mmustermann'` und untersucht in aller Ruhe, welche Eigenschaft bei diesem einen Konto abweicht.

## 5.5 Verbose, Debug und Information

Ein Skript hat zwei Arten von Ausgaben, die man sauber trennen muss: die *Nutzdaten*, die ein anderer Befehl weiterverarbeiten soll, und die *Diagnoseausgaben*, die Menschen helfen, den Ablauf zu verstehen. Vermischt man sie, steht ein Fortschrittstext plötzlich in einer CSV-Datei oder in der Eingabe des nächsten Befehls. PowerShell löst das mit getrennten *Streams*, von denen jeder eigene Befehle, eigene Steuerung und eigene Umleitungsnummern hat.

Stream 1 ist der Success Stream, die normale Ausgabe. Stream 2 ist der Error Stream. Stream 3 ist der Warning Stream (`Write-Warning`), Stream 4 der Verbose Stream (`Write-Verbose`), Stream 5 der Debug Stream (`Write-Debug`) und Stream 6 der Information Stream (`Write-Information`). Nur Stream 1 wird von der Pipeline an den nächsten Befehl weitergegeben. Die anderen erscheinen auf der Konsole, je nach Voreinstellung der zugehörigen Präferenzvariablen (`$WarningPreference`, `$VerbosePreference`, `$DebugPreference`, `$InformationPreference`). Standardmäßig sind Verbose, Debug und Information *aus*, Warnungen und Fehler *an*.

Das folgende Beispiel schreibt in alle Ströme und leitet einzelne um. Die Umleitung `3>$null` verwirft den Warning Stream, `4>&1` leitet den Verbose Stream in den Success Stream um.

```powershell
function Test-Streams {
    [CmdletBinding()]
    param()
    'Ausgabe'
    Write-Verbose 'Verbose'
    Write-Debug 'Debug'
    Write-Warning 'Warnung'
    Write-Information 'Information'
    Write-Host 'Host'
    Write-Error 'Fehler'
}

# Nur der Success Stream: alle anderen verwerfen
$nutzdaten = Test-Streams 2>$null 3>$null 4>$null 5>$null 6>$null
"nur Success: [$nutzdaten]"

# Streams als Objekte einsammeln
Test-Streams -Verbose -InformationAction Continue 4>&1 6>&1 2>$null 3>$null |
    ForEach-Object { $_.GetType().Name + ': ' + $_ }
```

Im Test ergab die erste Zeile `nur Success: [Ausgabe]`. Die zweite Aufrufvariante lieferte vier Objekte: `String: Ausgabe`, `VerboseRecord: Verbose`, `InformationRecord: Information` und `InformationRecord: Host`. Das letzte zeigt eine Besonderheit: `Write-Host` schreibt seit PowerShell 5 in den Information Stream (mit der Markierung `PSHost`), und kann daher umgeleitet werden. Früher ging die Ausgabe direkt an die Konsole und war nicht abgreifbar. Trotzdem gilt weiter: `Write-Host` ist für Anzeigen gedacht (Menüs, Farben), nicht für Daten. Wer Nutzdaten ausgibt, schreibt sie deshalb mit `Write-Output` oder einfach durch Nennen des Wertes.

### Die Streams im Einzelnen.

`Write-Verbose` dient für Fortschrittsmeldungen für Menschen: „Verbinde mit srv01“, „12 Dateien gefunden“. Es erscheint nur bei `-Verbose`. In einer Funktion mit `[CmdletBinding()]` erhält man diesen Schalter gratis, und er wirkt auch auf alle aufgerufenen erweiterten Funktionen. `Write-Debug` ist für Entwicklerdetails gedacht; mit `-Debug` zeigt PowerShell 7 die Meldungen an (im Test erschien `DEBUG: dbg` ohne Rückfrage); Windows PowerShell 5.1 fragt dagegen bei jeder Meldung nach, wie es weitergehen soll (nicht getestet). Dauerhaft schaltet man die Anzeige mit `$DebugPreference = 'Continue'` ein. `Write-Information` ist der neueste Strom. Er überträgt nicht nur Text, sondern ein Objekt (`InformationRecord`) mit Zeitstempel, Quelle und *Tags*. Man kann mit `-Tags 'Audit','Schritt1'` Kategorien vergeben und später danach filtern (im Test lieferte `$_.Tags` die Werte `Audit,Schritt1`). Mit `-InformationAction Continue` wird die Meldung angezeigt, mit `SilentlyContinue` (Standard) bleibt sie unsichtbar, aber über `-InformationVariable` und die Umleitung `6>` abgreifbar.

### Warum die Trennung wichtig ist.

Eine Funktion, die `Write-Host` nutzt, um ihr Ergebnis auszugeben, liefert nichts zurück: Die Variable bleibt leer. Eine Funktion, die Fortschrittsmeldungen mit `Write-Output` schreibt, verschmutzt ihr Ergebnis. Der Aufrufer bekommt dann ein Array aus Meldungen *und* Daten. Klare Regel: In den Success Stream gehört nur das, was ein Aufrufer als Ergebnis erwartet.

```powershell
function Get-Dateien {
    [CmdletBinding()]
    param([string] $Pfad = '.')
    Write-Verbose "Durchsuche $Pfad"
    $liste = Get-ChildItem -LiteralPath $Pfad -File
    Write-Information "Gefunden: $($liste.Count)" -Tags 'Statistik'
    $liste
}

Get-Dateien -Pfad /tmp | Select-Object -First 1 -ExpandProperty Name
Get-Dateien -Pfad /tmp -Verbose -InformationAction Continue | Out-Null
```

Die erste Zeile liefert nur Daten, die zweite zeigt zusätzlich die Meldungen `VERBOSE: Durchsuche /tmp` und `Gefunden: ...`. Weil die Funktion Daten und Diagnose trennt, lässt sie sich in einer Pipeline bedenkenlos verwenden.

### Transkript.

Mit `Start-Transcript -Path datei.txt` protokolliert PowerShell alles, was in der Sitzung auf der Konsole erscheint, in eine Textdatei, bis `Stop-Transcript` aufgerufen wird. Das ist für Skripte auf Servern praktisch, wenn man im Nachhinein sehen will, was ein nächtlicher Lauf getan hat. Im Test enthielt die Datei die normalen Ausgaben, den `Write-Host`-Text und die Warnung (`WARNING: warn`). Es protokolliert, was angezeigt wird; was Verbose-Präferenzen unterdrücken, steht auch nicht im Transkript. Außerdem können Transkripte vertrauliche Daten enthalten (Lerneinheit 8.8).

> **Ergänzung aus der Nachrecherche**
>
> **Ausgabe in der Pipeline und Exit Codes.** In Pipelines von CI/CD-Systemen (Modul 11) wird die Standardausgabe oft als Ergebnis gelesen, der Fehlerstrom als Fehlerausgabe. Aus diesem Grund sollte ein Skript, das in einer Pipeline läuft, Diagnosen nicht mit `Write-Output` schreiben. Einige Systeme erwarten zudem spezielle Textformate für Warnungen (zum Beispiel Befehle in der Form `##vso[...]` bei Azure Pipelines); diese Formate sind plattformspezifisch und wurden nicht abgerufen. Prüfen Sie sie in der Dokumentation Ihres Systems.

> **Merke**
>
> Sechs Streams: 1 Success, 2 Error, 3 Warning, 4 Verbose, 5 Debug, 6 Information. Nur Stream 1 ist Pipeline-Nutzdaten. `Write-Verbose`/`Write-Debug`/`Write-Information` sind abschaltbare Diagnose, `Write-Host` ist für Anzeigen. Umleiten mit `N>`, `N>&1`. `Start-Transcript` protokolliert die Konsole.

```powershell
# FALSCH: Diagnose im Success Stream
function Get-Anzahl {
    Write-Output 'Zaehle Dateien...'
    (Get-ChildItem -File).Count
}
$n = Get-Anzahl
$n + 1                       # Fehler: Text + Zahl, $n ist ein Array

# RICHTIG: Diagnose in den Verbose Stream
function Get-Anzahl {
    [CmdletBinding()]
    param()
    Write-Verbose 'Zaehle Dateien...'
    (Get-ChildItem -File).Count
}
$n = Get-Anzahl
$n + 1
```

Im ersten Fall ist `$n` ein Array aus dem Text und der Zahl; die Addition endet mit einem Typfehler. Im zweiten ist `$n` eine Zahl, und die Meldung erscheint nur bei `-Verbose`.

> **Praxisbeispiel**
>
> Ein Skript, das von einem Planer täglich gestartet wird, schreibt Ergebnisse (eine CSV-Datei) über den Success Stream, Verlaufsmeldungen über `Write-Verbose` und Fehler über den Error Stream. Der Planer startet es mit `-Verbose` und leitet alle Ströme getrennt in Dateien um, sodass man bei einem Problem sowohl das Ergebnis als auch den Ablauf nachlesen kann.

## 5.6 Logging-Patterns

Meldungen auf der Konsole verschwinden, sobald das Fenster geschlossen oder der Lauf vom Planer beendet ist. Für Skripte, die unbeaufsichtigt arbeiten, braucht man *Logs*: dauerhafte, durchsuchbare Aufzeichnungen. Ein gutes Log beantwortet vier Fragen: Wann ist etwas passiert? Welcher Lauf war es? Wie schwerwiegend ist es? Was genau ist passiert? Alles Weitere (Rechner, Benutzer, betroffene Objekte) ist willkommen, solange es keine Geheimnisse enthält.

Die Grundlage ist eine einheitliche Struktur. Statt freier Textzeilen schreibt man *strukturierte Logs*: eine Zeile pro Ereignis, jede Zeile ein JSON-Objekt (das Format heißt JSON Lines). So lassen sich Logs später mit `ConvertFrom-Json` auswerten und von Log-Werkzeugen einlesen. Die wichtigsten Felder sind: ein Zeitstempel (am besten in UTC im ISO-8601-Format), eine *Run-ID* (eine kurze eindeutige Kennung pro Skriptlauf), der Schweregrad (`Debug`, `Info`, `Warn`, `Error`) und die Meldung. Mit der Run-ID kann man bei mehreren Läufen einer Nacht alle Zeilen eines Laufs herausfiltern.

```powershell
$script:RunId    = [guid]::NewGuid().ToString('N').Substring(0, 8)
$script:LogDatei = Join-Path ([System.IO.Path]::GetTempPath()) "demo-$($script:RunId).jsonl"
$script:LogLevel = 'Info'

function Write-Log {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, Position = 0)] [string] $Message,
        [ValidateSet('Debug', 'Info', 'Warn', 'Error')] [string] $Level = 'Info',
        [hashtable] $Data = @{}
    )
    $rang = @{ Debug = 0; Info = 1; Warn = 2; Error = 3 }
    if ($rang[$Level] -lt $rang[$script:LogLevel]) { return }

    $eintrag = [ordered]@{
        time  = [datetime]::UtcNow.ToString('o')
        runId = $script:RunId
        level = $Level
        host  = [System.Net.Dns]::GetHostName()
        user  = [System.Environment]::UserName
        msg   = $Message
    }
    foreach ($k in $Data.Keys) { $eintrag[$k] = $Data[$k] }
    ($eintrag | ConvertTo-Json -Compress -Depth 4) |
        Add-Content -LiteralPath $script:LogDatei -Encoding utf8

    switch ($Level) {
        'Debug' { Write-Debug $Message }
        'Info'  { Write-Information $Message -Tags 'Log' }
        'Warn'  { Write-Warning $Message }
        'Error' { Write-Error $Message }
    }
}

Write-Log 'Start' -Data @{ quelle = '/daten' }
Write-Log 'Detail' -Level Debug
Write-Log 'Achtung' -Level Warn 3>$null
Write-Log 'Kopiert' -Data @{ dateien = 12; dauerMs = 340 }
Get-Content $script:LogDatei
```

Die Funktion `Write-Log` legt zuerst die Rangfolge der Stufen fest und verwirft Meldungen unter dem eingestellten Mindestlevel, im Beispiel `Info`. Deshalb fehlt die Debug-Zeile im Log. Danach baut sie einen geordneten Eintrag mit Zeit (`'o'` liefert das ISO-8601-Format mit Millisekunden und dem Zeichen Z für UTC), Run-ID, Stufe, Rechner, Benutzer und Meldung, hängt zusätzliche Felder aus der Hashtable `-Data` an und schreibt die Zeile als kompaktes JSON an die Datei. Zusätzlich gibt sie die Meldung über den passenden Stream aus, damit der Konsolenbetrieb weiter funktioniert. Im Test begann eine Zeile mit den Feldern `time`, `runId`, `level`, `host`, `user` und `msg`, gefolgt von den Zusatzfeldern, zum Beispiel `"quelle":"/daten"`. Die Auswertung ist einfach: `Get-Content $datei | ConvertFrom-Json | Where-Object level -eq 'Warn'` lieferte im Test genau die Warnung „Achtung“. Der Zeitstempel kam dabei als `DateTime` zurück (`ConvertFrom-Json` wandelt ISO-Zeitstempel selbstständig um, in die lokale Zeitzone).

### Zielorte und Standardisierung.

Mehrere Skripte sollten dieselbe Funktion und dasselbe Format verwenden; üblich ist eine Funktion in einem gemeinsamen Modul (Modul 11). Zielorte sind je nach Umgebung: eine Datei pro Lauf oder pro Tag (mit Aufbewahrungsregel, damit der Speicher nicht überläuft), das Windows-Ereignisprotokoll (Modul 9), ein zentraler Log-Dienst oder die Konsole eines Pipeline-Systems. Der Ort der Logdatei gehört in die Konfiguration (Lerneinheit 4.9), nicht in den Code. Beim Schreiben sind drei Regeln wichtig: Keine Geheimnisse und möglichst keine personenbezogenen Daten ins Log (Passwörter, Tokens, Mailadressen); Zeitstempel in UTC, damit Läufe auf Servern in verschiedenen Zeitzonen vergleichbar bleiben; und Schreibfehler des Logs dürfen die eigentliche Arbeit nicht zum Absturz bringen, wenn das Log nicht kritisch ist.

> **Ergänzung aus der Nachrecherche**
>
> **Datenschutz im Log.** Logs sind Datensammlungen. Sobald sie Benutzernamen, Rechnernamen mit Personenbezug oder Inhalte aus Nutzerdaten enthalten, unterliegen sie in der EU der Datenschutz-Grundverordnung (DSGVO): Zweckbindung, Datensparsamkeit und Löschfristen sind dann zu beachten. Diese Hinweise sind allgemeiner Art, wurden in diesem Modul nicht aus Rechtsquellen abgerufen und sind keine Rechtsberatung. Bei konkreten Fragen wenden Sie sich an die Datenschutzbeauftragte oder den Datenschutzbeauftragten Ihrer Organisation.

> **Merke**
>
> Strukturierte Logs: eine JSON-Zeile pro Ereignis mit Zeit (UTC), Run-ID, Level, Meldung und Zusatzfeldern. Level-Filter, einheitliche Funktion, Ort aus der Konfiguration. Keine Geheimnisse ins Log. Ausgewertet wird mit `ConvertFrom-Json`.

```powershell
# FALSCH: Freitext ohne Zeit, ohne Kennung, mit Geheimnis
Add-Content log.txt "Verbinde mit $server als $benutzer mit Passwort $passwort"

# RICHTIG: strukturiert, ohne Geheimnis
Write-Log 'Verbindung aufgebaut' -Data @{ server = $server; benutzer = $benutzer }
```

Der erste Eintrag hat keinen Zeitstempel, ist nicht maschinenlesbar und gibt das Kennwort an jeden weiter, der die Datei lesen darf. Der zweite enthält nur das, was für die Fehlersuche nötig ist.

> **Praxisbeispiel**
>
> Ein Team betreibt zwölf geplante Skripte. Alle verwenden dieselbe `Write-Log`-Funktion und schreiben JSON Lines in einen gemeinsamen Ordner. Eine kurze Abfrage mit `ConvertFrom-Json` liefert jeden Morgen eine Liste aller Läufe mit Fehlern der letzten 24 Stunden, mit Run-ID, Rechner und Meldung.

## 5.7 Idempotenz

Eine Operation ist *idempotent*, wenn mehrfaches Ausführen dasselbe Ergebnis hat wie einmaliges. Eine Lichtschalterfunktion „einschalten“ ist idempotent (das Licht ist danach an, egal wie oft man sie ruft), „umschalten“ ist es nicht. Für Automation ist das eine der wichtigsten Eigenschaften, denn Skripte laufen mehrfach: nach einem Abbruch, bei einem Wiederanlauf durch den Planer, auf mehreren Servern, nach einer Änderung der Konfiguration. Ein nicht idempotentes Skript hängt bei jedem Lauf eine weitere Zeile in die Datei, legt einen weiteren Benutzer an oder scheitert, weil es existiert.

Das Entwurfsmuster dahinter ist *Zielzustand statt Einzelschritt*: Das Skript beschreibt, wie es hinterher aussehen soll, prüft den Ist-Zustand und tut nur das, was fehlt. Die Prüfung folgt immer demselben Dreischritt: Ist-Zustand lesen, mit dem Soll vergleichen, nur bei Abweichung ändern, und melden, ob etwas geändert wurde. Moderne Konfigurationswerkzeuge (Desired State Configuration, Modul 11) arbeiten nach diesem Prinzip.

```powershell
function Set-ZeileInDatei {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Pfad,
        [Parameter(Mandatory)] [string] $Zeile
    )
    $inhalt = if (Test-Path -LiteralPath $Pfad) { Get-Content -LiteralPath $Pfad } else { @() }
    if ($inhalt -contains $Zeile) {
        Write-Verbose "Unveraendert: $Zeile"
        return $false
    }
    if ($PSCmdlet.ShouldProcess($Pfad, "Zeile hinzufuegen: $Zeile")) {
        Add-Content -LiteralPath $Pfad -Value $Zeile
        return $true
    }
}

$datei = Join-Path ([System.IO.Path]::GetTempPath()) 'hosts-demo.txt'
Set-ZeileInDatei -Pfad $datei -Zeile '10.0.0.5 srv01'
Set-ZeileInDatei -Pfad $datei -Zeile '10.0.0.5 srv01' -Verbose
Set-ZeileInDatei -Pfad $datei -Zeile '10.0.0.6 srv02' -WhatIf
(Get-Content -LiteralPath $datei).Count
```

Die Funktion liest zuerst den Inhalt der Datei. Ist die Zeile bereits vorhanden, meldet sie über `-Verbose`, dass nichts zu tun ist, und liefert `$false` („nichts geändert“). Andernfalls fragt sie mit `ShouldProcess`, ob die Änderung ausgeführt werden soll (damit `-WhatIf` funktioniert), und hängt die Zeile an. Im Test lieferte der erste Aufruf `True`, der zweite `False` samt der Verbose-Meldung, und der dritte mit `-WhatIf` zeigte „What if: Performing the operation "Zeile hinzufuegen: 10.0.0.6 srv02"...“ ohne Änderung; die Datei hatte danach eine Zeile. Der Rückgabewert, der angibt, ob etwas verändert wurde, ist nützlich, weil ein Aufrufer daraus zum Beispiel ableitet, ob ein Dienst neu gestartet werden muss.

Für Verzeichnisse und ähnliche Objekte verwendet man dasselbe Muster mit einem Ergebnisobjekt. `New-Item -ItemType Directory` scheitert, wenn das Verzeichnis existiert (im Test setzte der zweite Aufruf `$?` auf `False`); mit `-Force` wird er zwar nicht scheitern, aber man erfährt nicht, ob etwas getan wurde. Besser ist die ausdrückliche Prüfung.

```powershell
function Set-Ordner {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad)
    if (Test-Path -LiteralPath $Pfad -PathType Container) {
        return [pscustomobject]@{ Pfad = $Pfad; Geaendert = $false }
    }
    New-Item -ItemType Directory -Path $Pfad | Out-Null
    [pscustomobject]@{ Pfad = $Pfad; Geaendert = $true }
}

$p = Join-Path ([System.IO.Path]::GetTempPath()) 'idem-demo'
(Set-Ordner -Pfad $p).Geaendert
(Set-Ordner -Pfad $p).Geaendert
Remove-Item -LiteralPath $p
```

Die Ausgabe war `True` und dann `False`. Die Kontrolle „existiert es schon“ ist auch für die Wiederaufnahme nach Abbrüchen wertvoll: Ein Skript, das 1 000 Dateien bearbeiten soll und nach 400 abbricht, kann beim nächsten Lauf die ersten 400 überspringen.

### Rücknehmbarkeit.

Zum idempotenten Design gehört der Rückweg. Wer eine Datei ändert, sichert vorher das Original; scheitert die Änderung, wird die Sicherung zurückkopiert (Rollback). Das folgende Beispiel zeigt das Muster in Kurzform: Sicherung anlegen, ändern, bei Fehler zurückspielen.

```powershell
$datei = Join-Path ([System.IO.Path]::GetTempPath()) 'konfig-demo.txt'
Set-Content -LiteralPath $datei -Value 'alt'
$sicherung = "$datei.bak"
Copy-Item -LiteralPath $datei -Destination $sicherung -Force
try {
    Add-Content -LiteralPath $datei -Value 'neu'
    throw 'Fehler nach der Aenderung'
}
catch {
    Copy-Item -LiteralPath $sicherung -Destination $datei -Force
    Write-Warning "Zurueckgerollt: $($_.Exception.Message)"
}
Get-Content -LiteralPath $datei
Remove-Item -LiteralPath $datei, $sicherung
```

Nach dem simulierten Fehler steht in der Datei wieder nur `alt`; das `catch` hat die Sicherung zurückgespielt. Dieses Muster gilt bei jeder Änderung, die sich nicht von selbst umkehrt: Registry-Werte, Konfigurationsdateien, Berechtigungen.

> **Merke**
>
> Idempotent heißt: mehrfaches Ausführen ergibt dasselbe wie einmaliges. Muster: Ist lesen, mit Soll vergleichen, nur bei Abweichung ändern, Änderung melden. `ShouldProcess` für `-WhatIf`. Vor Änderungen sichern, bei Fehlern zurückrollen.

```powershell
# FALSCH: haengt bei jedem Lauf eine weitere Zeile an
Add-Content -Path $hosts -Value '10.0.0.5 srv01'

# RICHTIG: nur anhaengen, wenn nicht vorhanden
if ((Get-Content -Path $hosts) -notcontains '10.0.0.5 srv01') {
    Add-Content -Path $hosts -Value '10.0.0.5 srv01'
}
```

Im ersten Fall steht die Zeile nach drei Läufen dreifach in der Datei (im Test erschien sie bei zwei Aufrufen zweimal). Im zweiten steht sie genau einmal, egal wie oft das Skript läuft.

> **Praxisbeispiel**
>
> Ein Einrichtungsskript für neue Server legt Ordner an, setzt Berechtigungen, installiert Software und trägt Einstellungen ein. Weil es idempotent ist, kann man es nach einer Änderung der Vorgaben auf allen hundert Servern erneut laufen lassen, ohne Schäden zu riskieren: Wo schon alles stimmt, tut es nichts, und im Bericht stehen nur die Server, die verändert wurden.

## 5.8 Pester Grundlagen

Wer Funktionen ändert, will wissen, ob sie noch das tun, was sie sollen. Manuelles Ausprobieren stößt schnell an Grenzen: Es ist langweilig, unvollständig und wird irgendwann vergessen. *Automatisierte Tests* prüfen das in Sekunden und bei jeder Änderung. Das Standardwerkzeug dafür in PowerShell ist *Pester*. Es ist ein Modul, das Testskripte (Dateien mit der Endung `.Tests.ps1`) ausführt und über jeden Test Erfolg oder Misserfolg berichtet. Pester 5 ist die aktuelle Hauptversion; Windows PowerShell 5.1 bringt eine alte Version 3.4 mit, die man mit `Install-Module Pester -Force -SkipPublisherCheck` (oder `Install-PSResource`) auf Version 5 hebt. Die folgenden Beispiele verwenden die Syntax von Pester 5.

> **Ergänzung aus der Nachrecherche**
>
> **Hinweis zur Prüfung dieses Abschnitts.** Pester ließ sich im Arbeitsbereich nicht installieren, weil die PowerShell Gallery gesperrt war. Die Pester-Beispiele wurden nur auf PowerShell-Syntax geprüft, nicht ausgeführt, und die hier genannte Pester-Syntax entspricht dem Fachwissen zur Version 5. Prüfen Sie die Beispiele in Ihrer Umgebung und lesen Sie bei Abweichungen die aktuelle Pester-Dokumentation (siehe Quellen, weiterführend).

Ein Pester-Test folgt dem Muster *Arrange-Act-Assert*. In der Phase *Arrange* bereitet man die Bedingungen vor (Testdaten, Dateien, Attrappen). In *Act* führt man genau die Funktion aus, die man testen will. In *Assert* prüft man das Ergebnis. Die Struktur bilden die Schlüsselwörter `Describe` (oder `Context`) zur Gruppierung, `It` für einen einzelnen Test und `Should` für die Prüfung. Der Operator `-Be` vergleicht auf Gleichheit, `-BeTrue`, `-BeNullOrEmpty`, `-Contain`, `-Match`, `-Throw` und weitere stehen zur Verfügung. Zusätzlich bieten `BeforeAll`, `BeforeEach` und `AfterEach` Blöcke für vorbereitende und aufräumende Schritte.

Das Beispiel testet die Funktion `Get-Speicherstatus` aus Modul 4. Man legt die Funktion in einer Datei `Get-Speicherstatus.ps1` ab und daneben die Testdatei `Get-Speicherstatus.Tests.ps1`.

```powershell
# Get-Speicherstatus.ps1
function Get-Speicherstatus {
    [CmdletBinding()]
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
```

```powershell
# Get-Speicherstatus.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Get-Speicherstatus.ps1
}

Describe 'Get-Speicherstatus' {
    It 'meldet kritisch unter 5 Prozent' {
        Get-Speicherstatus -FreiProzent 3 | Should -Be 'kritisch'
    }
    It 'meldet Warnung zwischen 5 und 15 Prozent' {
        Get-Speicherstatus -FreiProzent 12 | Should -Be 'Warnung'
    }
    It 'meldet in Ordnung ab 15 Prozent' {
        Get-Speicherstatus -FreiProzent 15 | Should -Be 'in Ordnung'
    }
    It 'lehnt Werte ueber 100 ab' {
        { Get-Speicherstatus -FreiProzent 101 } | Should -Throw
    }
    It 'prueft die Grenzen genau' -ForEach @(
        @{ Wert = 4.99; Erwartet = 'kritisch' }
        @{ Wert = 5;    Erwartet = 'Warnung' }
        @{ Wert = 14.99; Erwartet = 'Warnung' }
    ) {
        Get-Speicherstatus -FreiProzent $Wert | Should -Be $Erwartet
    }
}
```

Die Testdatei lädt zuerst im `BeforeAll`-Block die zu testende Datei mit Dot-Sourcing, denn in Pester 5 muss Code, der von den Tests benötigt wird, in `BeforeAll` stehen und nicht auf oberster Ebene der Datei. `Describe` fasst alle Tests der Funktion zusammen. Jeder `It`-Block ruft die Funktion auf und schickt das Ergebnis an `Should`. Für Fehlerfälle verpackt man den Aufruf in einen Skriptblock und prüft mit `Should -Throw`, weil der Fehler sonst den Test selbst zum Absturz bringen würde. Der letzte Test ist ein *datengetriebener Test*: `-ForEach` wiederholt ihn für jede Hashtable, und die Schlüssel stehen im Test als Variablen zur Verfügung. Die Grenzwerttests (4,99; 5; 14,99) sind besonders wertvoll, denn Fehler stecken fast immer an den Grenzen. Man startet die Tests mit `Invoke-Pester ./Get-Speicherstatus.Tests.ps1 -Output Detailed`.

### Mocking und Isolation.

Admin-Code ruft oft Befehle auf, die man im Test nicht ausführen will: Löschen von Dateien, Neustart von Diensten, Aufrufe in der Cloud. Pester ersetzt solche Befehle durch Attrappen (`Mock`) und prüft mit `Should -Invoke`, ob und wie oft sie aufgerufen wurden. Mit `TestDrive:` stellt Pester außerdem ein temporäres Laufwerk bereit, das nach dem Test automatisch verschwindet.

```powershell
# Remove-AlteDateien.ps1
function Remove-AlteDateien {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Pfad,
        [int] $AlterTage = 30
    )
    $grenze = (Get-Date).AddDays(-$AlterTage)
    Get-ChildItem -LiteralPath $Pfad -File |
        Where-Object LastWriteTime -lt $grenze |
        ForEach-Object {
            if ($PSCmdlet.ShouldProcess($_.FullName, 'Loeschen')) {
                Remove-Item -LiteralPath $_.FullName
            }
        }
}
```

```powershell
# Remove-AlteDateien.Tests.ps1
BeforeAll { . $PSScriptRoot/Remove-AlteDateien.ps1 }

Describe 'Remove-AlteDateien' {
    BeforeEach {
        $alt = New-Item -Path 'TestDrive:/alt.txt' -ItemType File
        $alt.LastWriteTime = (Get-Date).AddDays(-60)
        $neu = New-Item -Path 'TestDrive:/neu.txt' -ItemType File
    }
    It 'loescht nur alte Dateien' {
        Remove-AlteDateien -Pfad 'TestDrive:/' -AlterTage 30
        Test-Path 'TestDrive:/alt.txt' | Should -BeFalse
        Test-Path 'TestDrive:/neu.txt' | Should -BeTrue
    }
    It 'loescht mit -WhatIf nichts' {
        Remove-AlteDateien -Pfad 'TestDrive:/' -AlterTage 30 -WhatIf
        Test-Path 'TestDrive:/alt.txt' | Should -BeTrue
    }
    It 'ruft Remove-Item nur einmal auf' {
        Mock Remove-Item { }
        Remove-AlteDateien -Pfad 'TestDrive:/' -AlterTage 30
        Should -Invoke Remove-Item -Exactly 1
    }
}
```

Der `BeforeEach`-Block legt vor jedem Test eine alte und eine neue Datei an, indem er die Änderungszeit der ersten zurückdatiert. Der erste Test prüft das Hauptverhalten, der zweite die `-WhatIf`-Sicherheit und der dritte, dass der gefährliche Befehl nur einmal aufgerufen wird, ohne dass tatsächlich etwas gelöscht wird, weil `Mock` ihn ersetzt. Dass sich Tests in dieser Form für Admin-Code lohnen, liegt auf der Hand: Sie dokumentieren das erwartete Verhalten, fangen Regressionen ab und ermöglichen, Änderungen ohne Angst zu machen.

### Was man testen sollte.

Nicht jede Zeile braucht einen Test. Lohnend sind Logik mit Verzweigungen, Grenzwerte, Fehlerfälle und alles, was Schaden anrichten kann. Nicht lohnend ist das Testen fremder Befehle (ob `Get-ChildItem` funktioniert, weiß Microsoft schon). Ein Test sollte schnell sein, unabhängig von anderen Tests und ohne Nebenwirkungen auf das System, denn sonst wird er nicht ausgeführt. Dafür gibt es Attrappen und `TestDrive`. In Modul 11 laufen dieselben Tests automatisch in der Pipeline.

> **Merke**
>
> Pester: `Describe`/`Context`/`It`/`Should`. Arrange-Act-Assert. Code in `BeforeAll` laden, `Should -Throw` mit Skriptblock, `-ForEach` für Datentabellen, `Mock` und `Should -Invoke` gegen Nebenwirkungen, `TestDrive:` für Dateien. Start: `Invoke-Pester`.

```powershell
# FALSCH: Test ruft den gefaehrlichen Befehl wirklich auf
It 'raeumt auf' {
    Remove-AlteDateien -Pfad 'C:\Daten'     # loescht echte Dateien!
}

# RICHTIG: Isoliert in TestDrive, Attrappe fuer Folgen
It 'raeumt auf' {
    Remove-AlteDateien -Pfad 'TestDrive:/' -AlterTage 30
    Test-Path 'TestDrive:/alt.txt' | Should -BeFalse
}
```

Der erste Test fasst echte Daten an und hängt von der Umgebung ab; er kann bei einem Fehler Schäden anrichten. Der zweite arbeitet in einem temporären Laufwerk, das Pester danach löscht, und prüft das Ergebnis.

> **Praxisbeispiel**
>
> Das Team ändert die Schwellwerte für die Speicherüberwachung. Die Tests zeigen binnen zwei Sekunden, dass ein Grenzwert (genau 15 Prozent) jetzt anders eingestuft wird als vorher. Ohne Test wäre das erst im Betrieb aufgefallen, wenn ein Alarm ausblieb.

## 5.9 Codequalität

Tests prüfen, ob der Code das Richtige tut; die *statische Codeanalyse* prüft, ob er auf eine Weise geschrieben ist, die Fehler und Missverständnisse vermeidet, ohne den Code auszuführen. Für PowerShell heißt das Werkzeug *PSScriptAnalyzer*. Es ist ein Modul mit dem Befehl `Invoke-ScriptAnalyzer`, der Skripte und Module gegen eine Sammlung von Regeln prüft und Befunde mit Schweregrad (`Error`, `Warning`, `Information`) meldet. Man installiert es aus der PowerShell Gallery (`Install-PSResource PSScriptAnalyzer`) und ruft es auf einer Datei oder einem Ordner auf.

> **Ergänzung aus der Nachrecherche**
>
> **Hinweis zur Prüfung dieses Abschnitts.** PSScriptAnalyzer ließ sich im Arbeitsbereich nicht installieren (PowerShell Gallery gesperrt). Die genannten Regelnamen entsprechen dem Fachwissen und sollten in der Dokumentation des Projekts gegengeprüft werden; die Befehlsbeispiele wurden nur auf Syntax geprüft.

Die Regeln erkennen typische Probleme. `PSAvoidUsingCmdletAliases` meldet Aliase in Skripten (Lerneinheit 2.6), `PSAvoidUsingWriteHost` die Nutzung von `Write-Host` für Daten, `PSAvoidUsingPlainTextForPassword` und `PSAvoidUsingConvertToSecureStringWithPlainText` Kennwörter im Klartext, `PSUseApprovedVerbs` nicht genehmigte Verben in Funktionsnamen, `PSUseShouldProcessForStateChangingFunctions` Funktionen, die etwas ändern, aber `-WhatIf` nicht unterstützen, `PSAvoidUsingInvokeExpression` die gefährliche Nutzung von `Invoke-Expression`, `PSUseDeclaredVarsMoreThanAssignments` nie benutzte Variablen und `PSAvoidGlobalVars` globale Variablen. Gerade die Sicherheitsregeln fangen Fehler, die in Code-Reviews leicht übersehen werden.

Das folgende Beispiel zeigt einen absichtlich schlechten Code und den Aufruf des Analyzers. Dazu legt man zusätzlich eine Konfigurationsdatei an, die festlegt, welche Regeln gelten und welche Schweregrade den Lauf scheitern lassen.

```powershell
# schlecht.ps1
function Do-Thing {
    param($Pfad, $Passwort)
    $unbenutzt = 1
    gci $Pfad | % { Write-Host $_.Name }
    Remove-Item $Pfad -Recurse
}
```

```powershell
Invoke-ScriptAnalyzer -Path ./schlecht.ps1 |
    Select-Object RuleName, Severity, Line, Message

# Konfiguration: PSScriptAnalyzerSettings.psd1
@{
    Severity     = @('Error', 'Warning')
    ExcludeRules = @('PSAvoidUsingWriteHost')
    Rules        = @{
        PSUseCompatibleSyntax = @{
            Enable         = $true
            TargetVersions = @('7.2', '7.4')
        }
    }
}

Invoke-ScriptAnalyzer -Path ./src -Recurse -Settings ./PSScriptAnalyzerSettings.psd1
```

Für `schlecht.ps1` würde der Analyzer unter anderem die Aliase `gci` und `%`, `Write-Host`, das Klartext-Kennwort im Parameter `$Passwort`, den nicht genehmigten Verbnamen `Do-Thing`, die nicht benutzte Variable und die fehlende `ShouldProcess`-Unterstützung des `Remove-Item`-Aufrufs melden; die genaue Liste hängt von der installierten Version ab. Im zweiten Block liest `Invoke-ScriptAnalyzer` eine `.psd1`-Datei (Datenformat aus Lerneinheit 4.9), die gewünschte Schweregrade, ausgeschlossene Regeln und Regelparameter festlegt. Das Beispiel verwendet `PSUseCompatibleSyntax` mit den Zielversionen; so lässt sich prüfen, ob ein Skript auf den Zielplattformen läuft.

### Styleguides und Reviews.

Werkzeuge können Formatierung und viele Fehler erkennen, aber sie ersetzen kein menschliches Auge. Ein Team braucht daher einen kurzen, schriftlichen Styleguide mit Regeln, die der Analyzer nicht kennt: Namenskonventionen für Funktionen und Parameter (Verb-Substantiv in Einzahl, PascalCase), Aufbau der Skripte (Lerneinheit 4.1), Pflichtinhalte wie kommentarbasierte Hilfe (Lerneinheit 2.9), Umgang mit Fehlern und Logging (diese Lerneinheiten) und Vorgaben zu Sicherheit (Modul 8). Der Analyzer setzt einen Teil davon durch; für den Rest gibt es *Code-Reviews*: Jede Änderung wird vor dem Zusammenführen von einer zweiten Person gelesen. Hilfreich ist eine kurze Prüfliste: Gibt es Hilfe und Parameterprüfung? Gibt es Fehlerbehandlung mit Exit Codes? Sind Geheimnisse sicher abgelegt? Ist die Änderung testbar und getestet? Läuft der Analyzer ohne Befunde?

Die Formatierung selbst lässt sich automatisieren: `Invoke-Formatter` (ebenfalls Teil des Moduls) wendet einheitliche Einrückung und Leerzeichen an. In VS Code läuft PSScriptAnalyzer zudem im Hintergrund und unterstreicht Befunde schon beim Tippen, sodass man Probleme sofort sieht. Ausnahmen für einzelne Stellen (wenn ein Befund begründet unzutreffend ist) markiert man mit dem Attribut `[Diagnostics.CodeAnalysis.SuppressMessageAttribute()]` und einer Begründung, statt die Regel für das ganze Projekt abzuschalten.

```powershell
function Show-Hinweis {
    [Diagnostics.CodeAnalysis.SuppressMessageAttribute(
        'PSAvoidUsingWriteHost', '',
        Justification = 'Interaktive Anzeige, keine Nutzdaten')]
    param([string] $Text)
    Write-Host $Text -ForegroundColor Yellow
}
```

Das Attribut unterdrückt die Regel nur für diese Funktion und dokumentiert die Begründung im Code, was bei Reviews nachvollziehbar bleibt. In Modul 11 läuft derselbe Analyzer automatisch in der Pipeline und blockiert Änderungen mit Fehlern, bevor sie in das Hauptprojekt gelangen.

> **Merke**
>
> PSScriptAnalyzer: `Invoke-ScriptAnalyzer` prüft Skripte statisch (Aliase, `Write-Host`, Klartext-Kennwörter, Verben, ShouldProcess). Einstellungen in einer `.psd1`-Datei, Ausnahmen mit `SuppressMessageAttribute` und Begründung. Dazu ein schriftlicher Styleguide und Code-Reviews mit Prüfliste.

```powershell
# FALSCH: Regel global abschalten, weil ein Befund laestig ist
Invoke-ScriptAnalyzer -Path ./src -ExcludeRule PSAvoidUsingPlainTextForPassword

# RICHTIG: Befund beheben (SecureString oder PSCredential nutzen)
param([Parameter(Mandatory)] [pscredential] $Credential)
```

Wer die Regel abschaltet, entfernt das Sicherheitsnetz für das ganze Projekt. Wer den Befund behebt, löst das Problem (Lerneinheit 8.6) und die Regel bleibt für künftigen Code aktiv.

> **Praxisbeispiel**
>
> Ein Repository mit Admin-Skripten hat eine Einstellungsdatei für PSScriptAnalyzer und eine Pester-Testsuite. Beim Zusammenführen einer Änderung werden beide automatisch ausgeführt; Befunde mit Schweregrad `Error` und fehlgeschlagene Tests blockieren die Änderung. Dadurch bleibt die Qualität stabil, auch wenn viele Personen daran arbeiten.

## 5.10 Übungsaufgaben zu Modul 05

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss.

**Aufgabe 1.** Nennen Sie die drei Fehlerarten in PowerShell, je ein Beispiel für ihre Entstehung und die Aussage, ob ein `try/catch` sie ohne weitere Maßnahmen fängt.

**Aufgabe 2.** Was ist der Unterschied zwischen `-ErrorAction SilentlyContinue` und `-ErrorAction Ignore`? Wie erkennt man beim Aufruf eines externen Programms, dass es gescheitert ist?

**Aufgabe 3.** Warum ist unter `Set-StrictMode -Version Latest` der Ausdruck `if ($h.Port) { ... }` bei einer Hashtable ohne Schlüssel `Port` ein Problem, und was ist die richtige Schreibweise?

**Aufgabe 4.** Erklären Sie, warum ein Skript Diagnoseausgaben nicht mit `Write-Output` schreiben soll. Welche Befehle sind stattdessen geeignet?

**Aufgabe 5.** Schreiben Sie eine Funktion `Read-JsonDatei`, die eine JSON-Datei einliest und das Objekt zurückgibt. Fehlt die Datei oder ist der Inhalt kein gültiges JSON, soll die Funktion mit `ThrowTerminatingError` und aussagekräftiger Meldung scheitern. Der Aufrufer soll `-ErrorAction` und `try/catch` verwenden können.

**Aufgabe 6.** Schreiben Sie eine Funktion `Copy-MitProtokoll`, die alle Dateien eines Ordners in einen Zielordner kopiert. Ein Fehler bei einer Datei darf die übrigen nicht blockieren. Am Ende soll die Funktion eine Zusammenfassung (Anzahl kopiert, Anzahl fehlgeschlagen) als Objekt zurückgeben und jeden Fehler mit `Write-Log` aus Lerneinheit 5.6 protokollieren. Die Funktion muss idempotent sein: Dateien, die schon identisch vorhanden sind (gleiche Größe und Änderungszeit), werden übersprungen.

**Aufgabe 7.** Schreiben Sie Pester-Tests für die Funktion `Set-ZeileInDatei` aus Lerneinheit 5.7: Zeile wird bei leerer Datei hinzugefügt, zweiter Aufruf ändert nichts und liefert `$false`, `-WhatIf` ändert die Datei nicht. Verwenden Sie `TestDrive:`.

**Aufgabe 8.** Entwerfen Sie ein kleines Qualitätsprojekt: eine Datei `PSScriptAnalyzerSettings.psd1`, die Fehler und Warnungen meldet und die Regel `PSAvoidUsingWriteHost` ausschließt, sowie ein Skript `Test-Qualitaet.ps1`, das den Analyzer über einen Ordner laufen lässt und bei Befunden mit Schweregrad `Error` mit Exit Code 1 endet. Erklären Sie, wie Sie das in eine Pipeline einbauen würden.

## 5.11 Lösungen zu Modul 05

### Aufgabe 1.

Nicht abbrechend: `Write-Error` oder ein Cmdlet, das bei einem Objekt von vielen scheitert (`Get-Content` mit fehlender Datei); das `catch` fängt ihn *nicht*, außer man verwendet `-ErrorAction Stop`. Anweisungsabbrechend: nicht vorhandener Befehl, ungültiger Parameterwert, `[int]::Parse('abc')`, `ThrowTerminatingError`; das `catch` fängt ihn. Skriptabbrechend: `throw`, Parserfehler, mit `Stop` eskalierte Fehler in einfachen Skriptblöcken; das `catch` fängt ihn, ohne Fangmaßnahme endet das Skript.

### Aufgabe 2.

`SilentlyContinue` unterdrückt die Anzeige, trägt den Fehler aber in `$Error` ein; `Ignore` unterdrückt auch den Eintrag (im Test: Anzahl 1 gegenüber 0). Ein externes Programm meldet Misserfolg nur über `$LASTEXITCODE` (ungleich 0) und `$?` gleich `$false`; es erzeugt weder einen Fehlereintrag noch ein `catch`, außer `$PSNativeCommandUseErrorActionPreference` ist `$true` (ab PowerShell 7.4).

### Aufgabe 3.

Unter Strict Mode ist der Zugriff auf eine nicht vorhandene Eigenschaft ein Fehler; im Test lautete die Meldung „The property ’Port’ cannot be found on this object. Verify that the property exists.“ Richtig ist `if ($h.ContainsKey('Port')) { ... }`, für Objekte `$o.PSObject.Properties['Port']`. Außerdem ist der Wert 0 oder `$false` sonst fälschlich „nicht vorhanden“.

### Aufgabe 4.

Der Success Stream gehört den Nutzdaten. Schreibt eine Funktion Diagnose dorthin, enthält ihr Rückgabewert Meldungen und Daten gemischt, und der Aufrufer muss sie trennen. Geeignet sind `Write-Verbose` (Ablauf), `Write-Debug` (Entwicklerdetails), `Write-Information` (strukturierte Hinweise mit Tags) und `Write-Warning` oder `Write-Error` für Probleme. Alle lassen sich einzeln ein- und ausschalten oder umleiten.

### Aufgabe 5.

```powershell
function Read-JsonDatei {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad)

    if (-not (Test-Path -LiteralPath $Pfad -PathType Leaf)) {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.IO.FileNotFoundException]::new("Datei nicht gefunden: $Pfad", $Pfad),
            'Json.DateiFehlt',
            [System.Management.Automation.ErrorCategory]::ObjectNotFound,
            $Pfad)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
    try {
        Get-Content -LiteralPath $Pfad -Raw -ErrorAction Stop |
            ConvertFrom-Json -ErrorAction Stop
    }
    catch {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.InvalidOperationException]::new("Ungueltiges JSON in $Pfad", $_.Exception),
            'Json.Ungueltig',
            [System.Management.Automation.ErrorCategory]::InvalidData,
            $Pfad)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
}

try { Read-JsonDatei -Pfad './fehlt.json' }
catch { $_.FullyQualifiedErrorId }
```

Die Funktion prüft die Existenz und wirft einen typisierten Fehler mit Kennung und Kategorie. Beim Einlesen schützt `try/catch` vor kaputtem JSON, die ursprüngliche Ausnahme bleibt als innere Ausnahme erhalten (`$_.Exception.InnerException`). Der Aufrufer sieht die Aufrufzeile statt der inneren Funktionszeile und kann mit `try/catch` (Ausgabe: `Json.DateiFehlt,Read-JsonDatei`) reagieren. Im Test unterdrückte `-ErrorAction SilentlyContinue` den Fehler nicht, die Meldung erschien trotzdem und das Skript lief an der nächsten Anweisung weiter, genau wie die Asymmetrie aus Lerneinheit 5.1 es beschreibt.

### Aufgabe 6.

```powershell
function Copy-MitProtokoll {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Quelle,
        [Parameter(Mandatory)] [string] $Ziel
    )
    if (-not (Test-Path -LiteralPath $Ziel)) {
        New-Item -ItemType Directory -Path $Ziel | Out-Null
    }
    $kopiert = 0; $uebersprungen = 0; $fehler = 0
    foreach ($datei in Get-ChildItem -LiteralPath $Quelle -File) {
        $zielPfad = Join-Path $Ziel $datei.Name
        try {
            if (Test-Path -LiteralPath $zielPfad) {
                $vorhanden = Get-Item -LiteralPath $zielPfad
                if ($vorhanden.Length -eq $datei.Length -and
                    $vorhanden.LastWriteTimeUtc -eq $datei.LastWriteTimeUtc) {
                    $uebersprungen++
                    Write-Log 'uebersprungen' -Level Debug -Data @{ datei = $datei.Name }
                    continue
                }
            }
            Copy-Item -LiteralPath $datei.FullName -Destination $zielPfad -Force -ErrorAction Stop
            $kopiert++
            Write-Log 'kopiert' -Data @{ datei = $datei.Name }
        }
        catch {
            $fehler++
            Write-Log 'Kopieren fehlgeschlagen' -Level Warn -Data @{ datei = $datei.Name; grund = $_.Exception.Message }
        }
    }
    [pscustomobject]@{ Kopiert = $kopiert; Uebersprungen = $uebersprungen; Fehlgeschlagen = $fehler }
}
```

Jede Datei hat ihr eigenes `try/catch`, deshalb blockiert ein Fehler die übrigen nicht. Die Prüfung von Größe und Änderungszeit (Copy-Item behält die Zeit der Quelle bei, daher ist der Vergleich nach dem Kopieren stabil) macht die Funktion idempotent: Der zweite Lauf meldet `Kopiert = 0` und überspringt alles. Als Fehlerstufe verwendet das Beispiel `Warn`, weil `Error` in `Write-Log` über `Write-Error` einen weiteren Fehler erzeugen würde.

### Aufgabe 7.

```powershell
BeforeAll { . $PSScriptRoot/Set-ZeileInDatei.ps1 }

Describe 'Set-ZeileInDatei' {
    BeforeEach {
        $pfad = Join-Path 'TestDrive:' 'test.txt'
        if (Test-Path $pfad) { Remove-Item $pfad }
    }
    It 'fuegt die Zeile bei fehlender Datei hinzu' {
        Set-ZeileInDatei -Pfad $pfad -Zeile 'a' | Should -BeTrue
        Get-Content $pfad | Should -Be 'a'
    }
    It 'aendert bei zweitem Aufruf nichts' {
        Set-ZeileInDatei -Pfad $pfad -Zeile 'a' | Out-Null
        Set-ZeileInDatei -Pfad $pfad -Zeile 'a' | Should -BeFalse
        (Get-Content $pfad).Count | Should -Be 1
    }
    It 'veraendert mit -WhatIf die Datei nicht' {
        Set-ZeileInDatei -Pfad $pfad -Zeile 'a' -WhatIf
        Test-Path $pfad | Should -BeFalse
    }
}
```

Jeder Test beginnt mit einem sauberen Zustand (`BeforeEach`). Der erste prüft den Normalfall, der zweite die Idempotenz (Rückgabewert und Zeilenanzahl), der dritte die `-WhatIf`-Sicherheit. Die Tests wurden nur auf Syntax geprüft, nicht mit Pester ausgeführt.

### Aufgabe 8.

```powershell
# PSScriptAnalyzerSettings.psd1
@{
    Severity     = @('Error', 'Warning')
    ExcludeRules = @('PSAvoidUsingWriteHost')
}
```

```powershell
# Test-Qualitaet.ps1
[CmdletBinding()]
param([string] $Pfad = (Join-Path $PSScriptRoot 'src'))

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$einstellungen = Join-Path $PSScriptRoot 'PSScriptAnalyzerSettings.psd1'
$befunde = Invoke-ScriptAnalyzer -Path $Pfad -Recurse -Settings $einstellungen

$befunde | Select-Object RuleName, Severity, ScriptName, Line, Message | Format-Table -AutoSize

$fehler = @($befunde | Where-Object Severity -eq 'Error')
if ($fehler.Count -gt 0) {
    Write-Error "$($fehler.Count) Befund(e) mit Schweregrad Error."
    exit 1
}
exit 0
```

Das Skript liest die Einstellungen, analysiert den Ordner rekursiv und gibt die Befunde aus. Nur Befunde mit Schweregrad `Error` führen zum Exit Code 1; `@(...)` erzwingt ein Array, damit `.Count` unter Strict Mode auch bei null oder einem Treffer funktioniert. In einer Pipeline ruft man es als Schritt auf (`pwsh -NoProfile -File ./Test-Qualitaet.ps1`); ein Exit Code ungleich 0 lässt den Schritt scheitern und blockiert die Änderung (Modul 11). Auch dieses Beispiel wurde nur auf Syntax geprüft.
