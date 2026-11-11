# Modul 06 - Dateisystem, Registry und Systemadministration

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (6.1 bis 6.9; 6.10 Übungsaufgaben, 6.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 12 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 2 (einzelne Lerneinheiten: 2 bzw. 3) |

> **Merke**
>
> Übersichtsblatt (12 h) und Detailblatt (9 h) weichen um drei Stunden voneinander ab. Dieses Skript folgt dem Detailblatt. Die Zusatzzeit eignet sich für die Übungen am Modulende, die in einer Windows-Testumgebung am meisten bringen.

### Lernziele.

Nach diesem Modul können Sie Windows- und plattformübergreifende Systeme automatisiert verwalten: Dateien, Ordner und Zugriffsrechte steuern und dabei gefährliche Massenoperationen absichern, die Registrierung lesen, ändern und vorher sichern, Dienste und Prozesse kontrolliert starten und beenden, Ereignisprotokolle gezielt abfragen, lokale Benutzer und Gruppen mit geringsten Rechten verwalten, Software nicht-interaktiv installieren, Netzwerkprobleme systematisch eingrenzen, Systeminformationen über CIM abfragen und Kopien mit Prüfsummen so absichern, dass sich wirklich etwas wiederherstellen lässt.

| **Nr.** | **Lerneinheit**            | **Dauer** | **Prio** |
|:--------|:---------------------------|:----------|:---------|
| 6.1     | Dateisystem-Automation     | 1 h       | 2        |
| 6.2     | Registry-Verwaltung        | 1 h       | 2        |
| 6.3     | Services und Prozesse      | 1 h       | 2        |
| 6.4     | Event Logs                 | 1 h       | 2        |
| 6.5     | Benutzer und Gruppen lokal | 1 h       | 2        |
| 6.6     | Pakete und Software        | 1 h       | 2        |
| 6.7     | Netzwerkdiagnose           | 1 h       | 3        |
| 6.8     | WMI und CIM                | 1 h       | 3        |
| 6.9     | Backup-nahe Aufgaben       | 1 h       | 3        |
|         | **Summe**                  | **9 h**   |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde, und warum viele Beispiele Windows brauchen.** Alles, was plattformübergreifend läuft (Dateioperationen, Archive, Prüfsummen, Prozesse, `Start-Process`, `-WhatIf`, eigene Funktionen), wurde mit PowerShell 7.6.6 unter Linux ausgeführt; die angegebenen Ergebnisse stammen aus diesen Läufen. Die Registrierung, Windows-Dienste, Ereignisprotokolle, lokale Benutzer, `winget` und der Befehl `Test-NetConnection` gibt es unter Linux nicht. Im Test fehlten dort ausdrücklich `Get-Service`, `Get-Acl`, `Set-Acl`, `Get-CimInstance`, `Get-WinEvent`, `Get-LocalUser`, `Resolve-DnsName` und `Test-NetConnection`, und das Laufwerk `HKLM:` existierte nicht („Cannot find a provider with the name ’Registry’.“). Diese Beispiele sind auf Syntax geprüft und mit der Microsoft-Dokumentation abgeglichen, wo sie abgerufen wurde (siehe Quellen); führen Sie sie vor dem Einsatz in einer Windows-Testumgebung aus. Aussagen, die auf allgemeinem Fachwissen beruhen und nicht abgerufen wurden, sind als solche gekennzeichnet.

## 6.1 Dateisystem-Automation

Kaum eine Administrationsaufgabe kommt ohne Dateien aus: Protokolle aufräumen, Berichte einsammeln, Ordner anlegen, Rechte setzen, Daten verschieben. Das Dateisystem ist in PowerShell ein *Provider* (Modul 2): Es erscheint als Laufwerk, und die gleichen Cmdlets (`Get-ChildItem`, `Copy-Item`, `Move-Item`, `Remove-Item`, `New-Item`, `Rename-Item`, `Get-Content`, `Set-Content`) arbeiten dort wie in anderen Providern. Der Vorteil gegenüber den alten Konsolenbefehlen liegt in den Objekten: `Get-ChildItem` liefert keine Textzeilen, sondern `FileInfo`- und `DirectoryInfo`-Objekte mit Eigenschaften wie `Length`, `LastWriteTime`, `Extension` und `Attributes`, die man filtern, sortieren und gruppieren kann.

Der gefährliche Teil der Dateiautomation sind *Massenoperationen*. Ein Befehl wie `Get-ChildItem -Recurse | Remove-Item` ist eine Zeile lang und kann in Sekunden Tausende Dateien vernichten. Fehlerquellen sind ein falsch aufgelöster Pfad (leere Variable, die zu `C:\` oder dem aktuellen Ordner wird), ein zu weit gefasster Filter, ein vertauschtes Kleiner- und Größerzeichen beim Alter und die Annahme, dass der Befehl nur wenige Treffer liefert. Die Abwehr besteht aus mehreren Schichten: Pfad prüfen, Treffermenge begrenzen, `-WhatIf` vorab, `ShouldProcess` in eigenen Funktionen und ein Protokoll.

### Die Grundbefehle im Überblick.

`Get-ChildItem -Path ... -Filter *.log -File -Recurse` listet Dateien rekursiv; `-Filter` wird vom Dateisystemprovider ausgewertet und ist dadurch viel schneller als `-Include`, das erst von PowerShell nachgelagert filtert. `-File` und `-Directory` unterscheiden Dateien und Ordner. `New-Item -ItemType Directory -Force` legt einen Ordner samt fehlender Elternordner an und ist ein sicherer Weg, damit ein Ordner existiert. `Copy-Item -Recurse` kopiert Ordner, `Move-Item` verschiebt, `Rename-Item` benennt um. Mit `Test-Path -LiteralPath` prüft man die Existenz; `-LiteralPath` verhindert, dass Zeichen wie `[` oder `*` als Muster gelten (Modul 8.7). Im Test lieferte `Copy-Item $d $d-kopie -Recurse` eine vollständige Kopie mit fünf Dateien.

Das folgende Beispiel zeigt eine Funktion, die alte Dateien löscht und dabei mehrere Sicherungen eingebaut hat. Es ist plattformübergreifend und wurde unter Linux ausgeführt.

```powershell
function Remove-AlteDateien {
    [CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'High')]
    param(
        [Parameter(Mandatory)] [string] $Ordner,
        [ValidateRange(1, 3650)] [int] $AlterTage = 30,
        [string] $Filter = '*.log',
        [ValidateRange(1, 1000)] [int] $MaxAnzahl = 20
    )
    $voll = (Resolve-Path -LiteralPath $Ordner).ProviderPath
    if ($voll.Length -lt 4 -or $voll -eq [IO.Path]::GetPathRoot($voll)) {
        throw "Ordner zu allgemein: $voll"
    }
    $grenze = (Get-Date).AddDays(-$AlterTage)
    $kandidaten = @(Get-ChildItem -LiteralPath $voll -Filter $Filter -File |
        Where-Object LastWriteTime -lt $grenze)
    if ($kandidaten.Count -gt $MaxAnzahl) {
        throw "Sicherung: $($kandidaten.Count) Treffer, erlaubt sind $MaxAnzahl"
    }
    foreach ($k in $kandidaten) {
        if ($PSCmdlet.ShouldProcess($k.FullName, 'Loeschen')) {
            Remove-Item -LiteralPath $k.FullName
        }
    }
    '{0} Datei(en) bearbeitet' -f $kandidaten.Count
}

Remove-AlteDateien -Ordner $pfad -AlterTage 25 -WhatIf
Remove-AlteDateien -Ordner $pfad -AlterTage 25 -Confirm:$false
```

Die Funktion deklariert mit `SupportsShouldProcess`, dass sie `-WhatIf` und `-Confirm` versteht, und setzt `ConfirmImpact = 'High'`, damit PowerShell standardmäßig zurückfragt. `Resolve-Path -LiteralPath` löst den Ordner zu einem vollständigen Pfad auf und bricht ab, wenn er nicht existiert; `.ProviderPath` liefert den reinen Dateisystempfad. Die folgende Prüfung lehnt Wurzelverzeichnisse und sehr kurze Pfade ab, damit ein leerer oder falscher Parameter nicht zu `C:\` oder `/` wird. `Get-ChildItem -LiteralPath ... -Filter ... -File` sammelt die Kandidaten; der Ausdruck `@(...)` sorgt dafür, dass `.Count` auch bei null oder einem Treffer funktioniert. Die Obergrenze `MaxAnzahl` ist die wichtigste Sicherung: Wenn plötzlich 5000 statt 5 Dateien passen, stimmt etwas nicht, und die Funktion bricht ab, statt zu löschen. Erst in der Schleife entscheidet `ShouldProcess`, ob wirklich gelöscht wird. Im Test mit sechs Dateien im Alter von 10 bis 60 Tagen listete der `-WhatIf`-Lauf vier Dateien auf („What if: Performing the operation “Loeschen“ on target ...”), der echte Lauf löschte sie, und es blieben zwei übrig. Mit `-MaxAnzahl 1` brach die Funktion mit „Sicherung: 2 Treffer, erlaubt sind 1“ ab, mit dem Ordner `/` mit „Ordner zu allgemein: /“.

### Zugriffsrechte (ACLs).

Unter Windows steuern *Access Control Lists* (ACL), wer eine Datei lesen, ändern oder löschen darf. Jeder Eintrag (*Access Control Entry*) verbindet eine Identität (Benutzer oder Gruppe) mit einem Recht (zum Beispiel `Read`, `Modify`, `FullControl`) und einer Wirkung (`Allow` oder `Deny`); Einträge können an Unterordner und Dateien vererbt werden. In PowerShell liest man sie mit `Get-Acl` und schreibt sie mit `Set-Acl`. Beide Cmdlets stammen aus dem Modul Microsoft.PowerShell.Security und sind unter Linux nicht vorhanden (im Test: „The term ’Get-Acl’ is not recognized“); dort verwendet man `chmod` und `chown` über native Aufrufe. Die folgenden Beispiele sind für Windows geschrieben und nur auf Syntax geprüft.

```powershell
# Windows: Rechte lesen
$acl = Get-Acl -LiteralPath 'D:\Daten\Berichte'
$acl.Access | Select-Object IdentityReference, FileSystemRights, AccessControlType, IsInherited

# Einem Benutzer Aendern-Rechte geben (mit Vererbung auf Unterordner und Dateien)
$regel = [System.Security.AccessControl.FileSystemAccessRule]::new(
    'CONTOSO\berichte-schreiber',
    [System.Security.AccessControl.FileSystemRights]::Modify,
    [System.Security.AccessControl.InheritanceFlags]'ContainerInherit, ObjectInherit',
    [System.Security.AccessControl.PropagationFlags]::None,
    [System.Security.AccessControl.AccessControlType]::Allow)
$acl.AddAccessRule($regel)
Set-Acl -LiteralPath 'D:\Daten\Berichte' -AclObject $acl
```

Der erste Block liest das Sicherheitsobjekt und zeigt seine Einträge; `IsInherited` verrät, ob ein Eintrag von einem übergeordneten Ordner stammt. Der zweite Block baut eine Regel (`FileSystemAccessRule`) aus Identität, Recht, Vererbungsflags, Weitergabeflags und Wirkung, fügt sie dem Objekt hinzu und schreibt das Objekt zurück. Wichtig ist die Reihenfolge: `Get-Acl` liefert eine *Kopie*; geändert wird erst durch `Set-Acl`. Rechte vergibt man grundsätzlich an Gruppen statt an einzelne Benutzer und nach dem Prinzip der geringsten Rechte (Modul 8.4), und man dokumentiert den Zustand vor der Änderung, etwa mit `(Get-Acl $pfad).Sddl`, um ihn bei Bedarf wiederherstellen zu können.

> **Ergänzung aus der Nachrecherche**
>
> **Plattformunterschiede (laut Tests und Dokumentation).** Unter Linux und macOS gibt es keine NTFS-ACLs in PowerShell, und Pfade verwenden `/`, sind meist case-sensitiv und kennen keine Laufwerksbuchstaben. `Start-Process -Verb` und `-WindowStyle` gelten laut Dokumentation nicht auf Nicht-Windows-Systemen. Wer plattformübergreifende Skripte schreibt, baut Pfade mit `Join-Path` und `[IO.Path]::Combine()`, nicht durch Anhängen von Backslashes, und prüft mit `$IsWindows`, `$IsLinux` und `$IsMacOS`, auf welchem System er läuft.

> **Merke**
>
> Dateien sind Objekte. Massenoperationen absichern: Pfad auflösen und prüfen, Treffer begrenzen, `-WhatIf`, `ShouldProcess`, `-LiteralPath`. `-Filter` ist schneller als `-Include`. ACLs: `Get-Acl` liefert eine Kopie, `Set-Acl` schreibt zurück; Rechte an Gruppen, Zustand vorher dokumentieren (nur Windows).

```powershell
# FALSCH: Variable leer -> Pfad wird zum aktuellen Ordner, nichts wird geprueft
$ordner = $env:BERICHTE_PFAD     # nicht gesetzt
Get-ChildItem "$ordner\*" -Recurse | Remove-Item -Force

# RICHTIG: Eingabe pruefen, begrenzen, erst simulieren
if ([string]::IsNullOrWhiteSpace($ordner)) { throw 'BERICHTE_PFAD nicht gesetzt' }
Remove-AlteDateien -Ordner $ordner -AlterTage 90 -WhatIf
```

Im ersten Fall wird aus einer nicht gesetzten Umgebungsvariable der Pfad `\*`, und das Skript löscht, was im aktuellen Ordner oder am Laufwerkswurzelpfad liegt. Im zweiten Fall bricht das Skript ab, bevor etwas geschieht, und simuliert zuerst.

> **Praxisbeispiel**
>
> Eine Aufräumaufgabe löscht jede Nacht Protokolle älter als 30 Tage auf Dateiservern. Nach einer Umstellung zeigt die Funktion plötzlich „Sicherung: 48000 Treffer, erlaubt sind 500“: Ein Fehler in der Anwendung hatte den Zeitstempel aller Dateien verändert. Ohne die Obergrenze wären alle Protokolle gelöscht worden. Die Funktion wirft, der Planer meldet den Fehlercode, ein Mensch prüft.

## 6.2 Registry-Verwaltung

Die *Registrierung* (Registry) ist die hierarchische Konfigurationsdatenbank von Windows. Sie speichert Einstellungen des Betriebssystems, von Diensten, Treibern und Anwendungen. Die Struktur gleicht einem Dateisystem: *Schlüssel* (Keys) entsprechen Ordnern, *Werte* (Values) entsprechen Dateien mit einem Namen, einem Typ und einem Inhalt. Die wichtigsten Wurzeln sind `HKEY_LOCAL_MACHINE` (`HKLM`, gilt für den Rechner) und `HKEY_CURRENT_USER` (`HKCU`, gilt für den angemeldeten Benutzer). Die Registrierung gibt es nur unter Windows; unter Linux und macOS fehlt der Provider.

PowerShell macht die Registrierung über den Registry-Provider als Laufwerke `HKLM:` und `HKCU:` zugänglich. Dadurch gelten die vertrauten Cmdlets: `Get-ChildItem` listet Unterschlüssel, `New-Item` legt einen Schlüssel an, `Remove-Item` löscht einen Schlüssel, `Test-Path` prüft die Existenz. Werte sind dagegen *Eigenschaften* der Schlüssel (Properties) und werden mit eigenen Cmdlets bearbeitet: `Get-ItemProperty` liest, `Set-ItemProperty` setzt, `New-ItemProperty` legt mit Typ an, `Remove-ItemProperty` entfernt, `Get-ItemPropertyValue` liefert nur den Wert. Diese Unterscheidung (Schlüssel als Item, Wert als Property) ist der häufigste Stolperstein.

Die Wertetypen bestimmen, was gespeichert werden kann: `String` (Text), `ExpandString` (Text mit Umgebungsvariablen, die beim Lesen aufgelöst werden), `DWord` (32-Bit-Zahl), `QWord` (64-Bit-Zahl), `Binary` (Bytefolge) und `MultiString` (Textliste). Wer den falschen Typ anlegt, erzeugt Einstellungen, die das Programm ignoriert oder falsch liest, etwa einen `String` `'1'` statt eines `DWord` 1.

```powershell
# Windows: Schluessel und Werte lesen
Get-ChildItem 'HKCU:\Software' | Select-Object -First 5 Name
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName, CurrentBuild, DisplayVersion
Get-ItemPropertyValue -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' -Name ProductName

# Eigenen Schluessel und Wert anlegen (im Benutzerbereich, ohne Administratorrechte)
$pfad = 'HKCU:\Software\Contoso\Werkzeug'
if (-not (Test-Path -LiteralPath $pfad)) { New-Item -Path $pfad -Force | Out-Null }
New-ItemProperty -LiteralPath $pfad -Name 'LogLevel' -Value 2 -PropertyType DWord -Force | Out-Null
Set-ItemProperty -LiteralPath $pfad -Name 'LogLevel' -Value 3
(Get-ItemProperty -LiteralPath $pfad).LogLevel

# Wert pruefen, ohne Fehler bei Nichtexistenz
$aktuell = (Get-ItemProperty -LiteralPath $pfad -Name 'Fehlt' -ErrorAction SilentlyContinue).Fehlt
if ($null -eq $aktuell) { 'Wert nicht vorhanden' }
```

Der erste Block liest Schlüssel und Werte: `Get-ItemProperty` gibt ein Objekt zurück, dessen Eigenschaften die Wertnamen sind, `Get-ItemPropertyValue` liefert nur den Inhalt. Der zweite Block legt einen eigenen Schlüssel im Benutzerbereich an (`HKCU` braucht keine Administratorrechte, `HKLM` schon). `New-ItemProperty` legt den Wert mit ausdrücklichem Typ `DWord` an; `Set-ItemProperty` ändert ihn später. `-LiteralPath` verhindert Musterauswertung, was bei Schlüsselnamen mit eckigen Klammern oder Sternchen relevant ist. Der dritte Block zeigt, wie man einen fehlenden Wert abfängt: Mit `-ErrorAction SilentlyContinue` entsteht kein Fehler, und der Zugriff auf die Eigenschaft liefert `$null`. Das ist wichtig, weil `Get-ItemProperty` bei einem nicht vorhandenen Wert einen Fehler auslöst. Unter Linux brachen die Aufrufe ab mit „Cannot find drive. A drive with the name ’HKLM’ does not exist.“

### Änderungen dokumentieren und sichern.

Registrierungsänderungen sind schwer rückgängig zu machen, weil es keinen Papierkorb gibt. Deshalb gehört vor jede Änderung eine Sicherung des betroffenen Schlüssels. Das Programm `reg.exe` exportiert Schlüssel in Dateien (`reg export "HKCU\Software\Contoso" sicherung.reg /y`) und stellt sie mit `reg import` wieder her. Alternativ liest man die Werte mit `Get-ItemProperty` und speichert sie als JSON oder CSV. Ein gutes Skript schreibt außerdem ein Protokoll: welcher Schlüssel, welcher Wert, alter Wert, neuer Wert, Zeitpunkt, ausführender Benutzer. So lässt sich später nachvollziehen, was sich geändert hat, und im Zweifel zurückgehen.

```powershell
function Set-RegistryWertGesichert {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Pfad,
        [Parameter(Mandatory)] [string] $Name,
        [Parameter(Mandatory)] $Wert,
        [ValidateSet('String', 'ExpandString', 'DWord', 'QWord', 'Binary', 'MultiString')]
        [string] $Typ = 'String',
        [string] $Protokoll = (Join-Path $env:TEMP 'registry-aenderungen.csv')
    )
    $alt = (Get-ItemProperty -LiteralPath $Pfad -Name $Name -ErrorAction SilentlyContinue).$Name
    if ($PSCmdlet.ShouldProcess("$Pfad\$Name", "setzen auf '$Wert' (alt: '$alt')")) {
        New-ItemProperty -LiteralPath $Pfad -Name $Name -Value $Wert -PropertyType $Typ -Force | Out-Null
        [pscustomobject]@{
            Zeit = (Get-Date).ToString('o'); Benutzer = [Environment]::UserName
            Pfad = $Pfad; Name = $Name; Alt = $alt; Neu = $Wert
        } | Export-Csv -LiteralPath $Protokoll -Append -NoTypeInformation -Encoding utf8
    }
}
```

Die Funktion liest zuerst den alten Wert (und toleriert, dass es ihn nicht gibt), fragt über `ShouldProcess`, setzt dann den neuen Wert mit dem gewünschten Typ und hängt eine Zeile an eine CSV-Datei an. `New-ItemProperty -Force` überschreibt einen vorhandenen Wert, ändert dabei aber auch den Typ, was `Set-ItemProperty` nicht täte; deshalb ist es hier die bessere Wahl. Die Funktion ist auf Syntax geprüft; die Zeilen mit dem Protokoll (`Export-Csv -Append`) entsprechen dem im Test unter Linux ausgeführten Muster aus Lerneinheit 6.9.

> **Ergänzung aus der Nachrecherche**
>
> **Reichweite und 64-Bit-Besonderheiten (allgemeines Fachwissen, nicht abgerufen).** Unter 64-Bit-Windows gibt es zwei Sichten auf Teile der Registrierung (`WOW6432Node` für 32-Bit-Programme). 32-Bit-PowerShell sieht dadurch andere Werte als 64-Bit-PowerShell unter `HKLM:\SOFTWARE`. Einstellungen für andere Benutzer liegen in deren Profil-Hive (`NTUSER.DAT`), den man mit `reg load` einhängen kann; für geplante Änderungen für alle Benutzer sind Gruppenrichtlinien meist der bessere Weg.

> **Merke**
>
> Schlüssel sind Items, Werte sind Properties. Lesen mit `Get-ItemProperty` oder `Get-ItemPropertyValue`, anlegen mit `New-ItemProperty -PropertyType`, ändern mit `Set-ItemProperty`. Typ beachten (`DWord` nicht `String`). Vor Änderungen sichern (`reg export`) und protokollieren. `HKCU` ohne, `HKLM` mit Administratorrechten. Nur Windows.

```powershell
# FALSCH: falscher Typ, keine Sicherung, kein Protokoll
Set-ItemProperty 'HKLM:\SOFTWARE\Contoso' -Name Aktiv -Value '1'

# RICHTIG: Typ ausdruecklich, gesichert und protokolliert
reg export 'HKLM\SOFTWARE\Contoso' "$env:TEMP\contoso-vorher.reg" /y
Set-RegistryWertGesichert -Pfad 'HKLM:\SOFTWARE\Contoso' -Name Aktiv -Wert 1 -Typ DWord
```

Im ersten Fall entsteht ein `String`-Wert (oder es wird ein bestehender Typ unbemerkt beibehalten), und niemand weiß später, wie es vorher war. Im zweiten Fall liegt eine Sicherung vor, der Typ ist richtig und die Änderung steht im Protokoll.

> **Praxisbeispiel**
>
> Ein Sicherheitsskript setzt auf allen Servern einen Registrierungswert, der ein altes Protokoll abschaltet. Weil jede Änderung mit altem und neuem Wert protokolliert wird, kann das Team, als eine Fachanwendung nach der Änderung nicht mehr verbindet, in Minuten nachvollziehen, welcher Server welchen Wert hatte, und die Änderung gezielt zurücknehmen.

## 6.3 Services und Prozesse

Ein *Prozess* ist ein laufendes Programm mit eigenem Speicher. Ein *Dienst* (Service) ist ein Programm, das im Hintergrund ohne angemeldeten Benutzer läuft und vom Betriebssystem verwaltet wird: Es wird beim Start automatisch gestartet, kann bei einem Fehler neu gestartet werden und läuft unter einem bestimmten Konto. Jeder Dienst läuft in einem Prozess, aber nicht jeder Prozess ist ein Dienst. Die Cmdlets für Dienste (`Get-Service`, `Start-Service`, `Stop-Service`, `Restart-Service`, `Set-Service`, `New-Service`) gibt es nur unter Windows; im Test unter Linux war `Get-Service` nicht vorhanden. Die Prozess-Cmdlets (`Get-Process`, `Stop-Process`, `Start-Process`, `Wait-Process`) laufen auf allen Plattformen.

### Prozesse abfragen und beenden.

`Get-Process` liefert `Process`-Objekte mit Eigenschaften wie `Id`, `ProcessName`, `CPU` (Prozessorzeit in Sekunden, keine Prozentzahl), `WorkingSet` (belegter Arbeitsspeicher in Byte), `StartTime` und `Responding`. Man filtert und sortiert wie bei allen Objekten. Ein nicht vorhandener Prozessname erzeugt einen Fehler mit der Kennung `NoProcessFoundForGivenName`; im Test war `$_.FullyQualifiedErrorId` dort `NoProcessFoundForGivenName,Microsoft.PowerShell.Commands.GetProcessCommand`. Beenden geschieht mit `Stop-Process -Id ...` oder `-Name ...`; das ist hart (der Prozess bekommt keine Gelegenheit zum Aufräumen), weshalb man bei Anwendungen zuerst `CloseMainWindow()` versucht. `Stop-Process` unterstützt `-WhatIf`; der Test zeigte „What if: Performing the operation “Stop-Process“ on target „pwsh (2984)“.”

```powershell
# Die zwei speicherhungrigsten Prozesse
Get-Process | Sort-Object WorkingSet -Descending |
    Select-Object -First 2 Name, Id, @{ n = 'MB'; e = { [math]::Round($_.WorkingSet / 1MB) } }

# Prozess mit Zeitlimit starten und notfalls beenden
$p = Start-Process -FilePath 'sleep' -ArgumentList '5' -PassThru
if (-not $p.WaitForExit(500)) {
    'Zeitlimit erreicht, beende Prozess'
    $p.Kill()
}

# Auf Prozess warten und Exit-Code lesen
$q = Start-Process -FilePath '/bin/sh' -ArgumentList '-c', '"exit 3"' -PassThru -Wait
$q.ExitCode

# Nicht vorhandenen Prozess sauber abfangen
try { Get-Process -Name 'gibtsnicht' -ErrorAction Stop }
catch { "Nicht gefunden: $($_.FullyQualifiedErrorId.Split(',')[0])" }
```

Der erste Block sortiert nach `WorkingSet` und rechnet mit einer berechneten Eigenschaft in Megabyte um. Der zweite startet ein Programm mit `-PassThru` (sonst liefert `Start-Process` kein Objekt) und wartet höchstens 500 Millisekunden; `WaitForExit` gibt `$false` zurück, wenn die Zeit abläuft, und `Kill()` beendet den Prozess. Im Test erschien „Zeitlimit erreicht“, und der Prozess wurde beendet. Der dritte Block startet einen Prozess synchron und liest den Exit-Code (im Test: 3). Der vierte fängt den Fehler mit `-ErrorAction Stop` ab, damit er im `catch` landet (Modul 5).

### Eine Stolperfalle bei Start-Process.

Beim dritten Block steckt ein Detail, das im Test aufgefallen ist und sehr häufig Fehler verursacht: `Start-Process` verbindet die Elemente von `-ArgumentList` laut Dokumentation zu einer einzigen Zeichenfolge, getrennt durch Leerzeichen, und reicht Anführungszeichen nicht von selbst weiter. Im ersten Versuch stand dort `-ArgumentList '-c', 'exit 3'`; das Programm erhielt `-c exit 3`, die Shell führte nur `exit` aus (und nahm die 3 als Namen des Programms), und der Exit-Code war *0*, obwohl 3 erwartet war. Erst mit den eingebetteten Anführungszeichen (`'"exit 3"'`) kam 3 heraus. Die Dokumentation empfiehlt deshalb, für beste Ergebnisse eine einzige Zeichenfolge mit allen Argumenten und nötigen Anführungszeichen zu übergeben. Wer nur ein Programm ausführen und seine Ausgabe verwenden will, nimmt statt `Start-Process` den Aufrufoperator (`& /bin/sh -c 'exit 3'`), bei dem PowerShell die Argumente einzeln übergibt und der Exit-Code in `$LASTEXITCODE` steht; im Test lieferte dieser Weg ebenfalls 3.

> **Ergänzung aus der Nachrecherche**
>
> **Aus der Dokumentation zu Start-Process (PowerShell 7.5).** `Start-Process` startet standardmäßig asynchron. Auf Windows ist das gestartete Programm unabhängig vom aufrufenden; auf Nicht-Windows-Systemen hängt es an der startenden Shell und endet mit ihr (Abhilfe: `nohup`). In einer Remote-Sitzung endet ein gestarteter Prozess mit der Sitzung, weshalb man dort `-Wait` verwenden muss. `-Wait` wartet auf den ganzen Prozessbaum (Prozess und Nachkommen), `Wait-Process` nur auf die angegebenen Prozesse. Der Parameter `-Environment` (ab PowerShell 7.4) überschreibt Umgebungsvariablen für den neuen Prozess; ein Wert `$null` entfernt sie. Die Dokumentation warnt außerdem, `-FilePath` und `-WorkingDirectory` nie mit nicht vertrauenswürdigen Daten zu verwenden (Modul 8.7).

### Dienste verwalten (Windows).

Ein Dienst hat einen *Status* (`Running`, `Stopped`, `Paused`, `StartPending`, `StopPending`), einen *Starttyp* (`Automatic`, `Manual`, `Disabled`, bei Bedarf auch `AutomaticDelayedStart`) und *Abhängigkeiten*: Dienste, die er braucht (`RequiredServices`), und Dienste, die ihn brauchen (`DependentServices`). Beim Beenden eines Dienstes muss man diese Abhängigkeiten beachten, denn `Stop-Service` verweigert das Beenden, solange laufende abhängige Dienste existieren, außer mit `-Force`, das die abhängigen mitbeendet. Ein sicherer Neustart prüft deshalb den Zustand vorher und nachher und wartet, bis der gewünschte Zustand wirklich erreicht ist.

```powershell
# Windows: Dienst kontrolliert neu starten und Zustand pruefen
function Restart-DienstSicher {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Name,
        [ValidateRange(1, 300)] [int] $TimeoutSekunden = 30
    )
    $dienst = Get-Service -Name $Name -ErrorAction Stop
    if ($dienst.StartType -eq 'Disabled') { throw "Dienst $Name ist deaktiviert" }
    $abhaengige = $dienst.DependentServices | Where-Object Status -eq 'Running'
    if ($abhaengige) { Write-Warning "Abhaengige Dienste laufen: $($abhaengige.Name -join ', ')" }

    if ($PSCmdlet.ShouldProcess($Name, 'Neustart')) {
        Restart-Service -Name $Name -Force -ErrorAction Stop
        $dienst.WaitForStatus('Running', [TimeSpan]::FromSeconds($TimeoutSekunden))
        $dienst.Refresh()
        if ($dienst.Status -ne 'Running') { throw "Dienst $Name laeuft nach $TimeoutSekunden s nicht" }
        "Dienst $Name laeuft"
    }
}
```

Die Funktion holt das Dienstobjekt (und bricht ab, wenn es den Dienst nicht gibt), lehnt deaktivierte Dienste ab und warnt vor laufenden abhängigen Diensten, weil ein Neustart mit `-Force` diese mit beendet. Nach dem Neustart wartet `WaitForStatus` bis zum Zeitlimit auf den Zustand `Running`; `Refresh()` aktualisiert das zwischengespeicherte Objekt, und die Prüfung stellt sicher, dass der Dienst wirklich läuft, bevor sie Erfolg meldet. Ohne diese Prüfung meldet `Restart-Service` Erfolg, auch wenn der Dienst kurz danach wieder abstürzt. Die Funktion ist auf Syntax geprüft und nur unter Windows ausführbar.

> **Merke**
>
> Prozesse (plattformübergreifend): `Get-Process`, `Stop-Process`, `Start-Process -PassThru`. `Start-Process` verbindet `-ArgumentList` ohne Anführungszeichen zu einer Zeichenfolge, Exit-Codes prüfen. Dienste (Windows): Status, Starttyp, Abhängigkeiten beachten, nach Änderung den Zielzustand abwarten und prüfen. `-WhatIf` nutzen.

```powershell
# FALSCH: Argumente als Array ohne Quotes, Exit-Code wird nie geprueft
Start-Process /bin/sh -ArgumentList '-c', 'exit 3' -Wait
'Fertig'

# RICHTIG: Argumente korrekt quoten, Exit-Code auswerten
$p = Start-Process /bin/sh -ArgumentList '-c', '"exit 3"' -Wait -PassThru
if ($p.ExitCode -ne 0) { throw "Programm fehlgeschlagen: Exit-Code $($p.ExitCode)" }
```

Im ersten Fall läuft der Befehl scheinbar erfolgreich und meldet „Fertig“, obwohl das gewünschte Programm nie das tat, was gedacht war. Im zweiten Fall kommt der Exit-Code an und wird geprüft.

> **Praxisbeispiel**
>
> Ein Wartungsskript startet nachts einen Installer mit mehreren Argumenten. Weil ein Pfad ein Leerzeichen enthält und nicht gequotet war, bekam der Installer einen abgeschnittenen Pfad und beendete sich mit Exit-Code 1603; das Skript meldete trotzdem „erfolgreich“. Erst die Auswertung von `ExitCode` und das Quoten der Argumente machten das sichtbar.

## 6.4 Event Logs

Windows schreibt alles Wichtige in *Ereignisprotokolle* (Event Logs): Dienststarts und Abstürze, Anmeldungen, Treiberfehler, Updates, Anwendungsfehler. Wenn ein System sich seltsam verhält, steht die Ursache oft schon im Protokoll, und das Ziel dieser Lerneinheit ist, sie gezielt zu finden, statt in der Ereignisanzeige zu blättern. Die klassischen Protokolle heißen `System`, `Application` und `Security`; dazu kommen viele anwendungsspezifische (zum Beispiel `Microsoft-Windows-PowerShell/Operational`). Ein Ereignis besteht aus Protokoll, Quelle (Provider), Ereignis-ID, Stufe (Kritisch, Fehler, Warnung, Information, Ausführlich), Zeitpunkt, Rechner, Benutzer und Meldungstext. Die Ereignis-ID ist nur zusammen mit der Quelle eindeutig.

Das Cmdlet der Wahl ist `Get-WinEvent`. Laut Microsoft-Dokumentation (abgerufen) holt es Ereignisse aus Ereignisprotokollen und aus Ablaufverfolgungsdateien (ETW) auf lokalen und entfernten Rechnern, liefert standardmäßig die neuesten zuerst und ist „only available on the Windows platform“. Es ersetzt `Get-EventLog`, das nur klassische Protokolle liest und nur noch aus Gründen der Abwärtskompatibilität existiert; in PowerShell 7 ist `Get-EventLog` nicht mehr enthalten. Ohne Administratorrechte kann man manche Protokolle (vor allem `Security`) nicht lesen.

Der entscheidende Punkt ist das *Filtern*. Ein Server kann Millionen Ereignisse enthalten. `Get-WinEvent -LogName System | Where-Object Id -eq 7036` holt *alle* Ereignisse und filtert erst danach in PowerShell, was minutenlang dauern kann. Die Dokumentation sagt es ausdrücklich: Die Filtermethoden des Cmdlets sind effizienter als `Where-Object`, weil sie beim Abrufen angewendet werden. Es gibt drei: `-FilterHashtable` (einfach, für die meisten Fälle), `-FilterXPath` (XPath-Ausdruck) und `-FilterXml` (strukturierte Abfrage, wie sie die Ereignisanzeige als benutzerdefinierte Ansicht erzeugt).

```powershell
# Windows: Fehler und Kritisches der letzten 24 Stunden aus System und Application
$seit = (Get-Date).AddHours(-24)
$ereignisse = Get-WinEvent -FilterHashtable @{
    LogName   = 'System', 'Application'
    Level     = 1, 2            # 1 = Kritisch, 2 = Fehler
    StartTime = $seit
} -ErrorAction SilentlyContinue

# Auswertung: nach Quelle und ID gruppieren
$ereignisse |
    Group-Object ProviderName, Id |
    Sort-Object Count -Descending |
    Select-Object -First 10 Count, Name

# Einzelne Ereignisse im Detail, mit den Daten des Ereignisses
$ereignisse | Select-Object -First 3 TimeCreated, Id, ProviderName, LevelDisplayName, Message

# Dienstwechsel (Provider Service Control Manager, ID 7036) fuer einen Dienst
Get-WinEvent -FilterHashtable @{
    LogName = 'System'; ProviderName = 'Service Control Manager'; Id = 7036; StartTime = $seit
} | Where-Object Message -like '*Spooler*' |
    Select-Object TimeCreated, Message
```

Der erste Block bildet die Abfrage als Hashtable: Protokolle als Liste, `Level` 1 und 2 für Kritisch und Fehler, `StartTime` als Untergrenze. Laut Dokumentation sind Schlüssel und Werte nicht case-sensitiv und Platzhalter nur bei `LogName` und `ProviderName` erlaubt; gültige Schlüssel sind unter anderem `LogName`, `ProviderName`, `Path`, `Keywords`, `ID`, `Level`, `StartTime`, `EndTime`, `UserID`, `Data` und `SuppressHashFilter`. `-ErrorAction SilentlyContinue` unterdrückt die Meldung, die `Get-WinEvent` ausgibt, wenn keine Ereignisse passen („Es wurden keine Ereignisse gefunden“), und ist hier bewusst gewählt, weil „keine Fehler“ ein gutes Ergebnis ist. Der zweite Block fasst nach Quelle und ID zusammen und zeigt die häufigsten Fehlerbilder, was die schnellste Methode ist, ein Problem zu erkennen, das sich oft wiederholt. Der dritte Block zeigt einzelne Ereignisse mit Meldungstext. Im vierten Block wird zuerst serverseitig nach Protokoll, Quelle, ID und Zeit gefiltert und erst die Feinheit (Dienstname im Text) mit `Where-Object` nachgezogen; diese Reihenfolge ist die richtige Arbeitsteilung.

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Hinweise aus der Dokumentation zu Get-WinEvent.** Mit `-ListLog *` listet das Cmdlet alle Protokolle samt Größe und Zahl der Einträge, mit `-ListProvider *` alle Quellen. Mit `-ComputerName` fragt man entfernte Rechner ab; das Cmdlet nutzt dabei *nicht* PowerShell-Remoting, sondern den Ereignisdienst, dessen Firewallregel auf dem Zielrechner freigegeben sein muss. Der Parameter akzeptiert nur einen Namen, mehrere Rechner fragt man mit `foreach` ab. Die Windows-Schnittstelle begrenzt `-LogName` auf 256 Protokolle pro Abfrage. Gespeicherte Protokolldateien (`.evtx`, `.evt`, `.etl`) liest man mit `-Path`; für `.etl`-Dateien ist `-Oldest` nötig. Mit `SuppressHashFilter` lassen sich Ereignisse ausschließen, zum Beispiel alle der Stufe 4 (Information).

### Fehlerbilder ableiten.

Ein einzelnes Ereignis erklärt selten ein Problem; das Muster tut es. Fragen, die helfen: Wann trat der erste Fehler auf, und was geschah kurz davor (Update, Neustart, Konfigurationsänderung)? Wiederholt sich dieselbe Quelle und ID? Treten Fehler mehrerer Quellen zeitgleich auf (ein Netzwerkproblem erzeugt Folgefehler in vielen Diensten)? Ein bewährtes Vorgehen ist, die Ereignisse eines Zeitfensters um den Störungszeitpunkt aus *allen* relevanten Protokollen zu holen, nach Zeit zu sortieren und von oben nach unten zu lesen. Mit `$e.Properties[0].Value` greift man auf die Rohdaten eines Ereignisses zu; `$e.ToXml()` liefert die vollständige XML-Darstellung für die Weiterverarbeitung.

> **Merke**
>
> `Get-WinEvent` (nur Windows) statt `Get-EventLog`. Immer *beim Abruf* filtern (`-FilterHashtable`, `-FilterXPath`, `-FilterXml`), nicht danach mit `Where-Object`. Zeitfenster und Stufe angeben, nach Quelle und ID gruppieren, Muster statt Einzelereignisse lesen. Remote braucht keine Remoting-Konfiguration, aber eine Firewallfreigabe.

```powershell
# FALSCH: alles holen, danach filtern (kann bei grossen Protokollen minutenlang dauern)
Get-WinEvent -LogName System | Where-Object { $_.Id -eq 7036 -and $_.TimeCreated -gt (Get-Date).AddDays(-1) }

# RICHTIG: Filter wird beim Abruf angewendet
Get-WinEvent -FilterHashtable @{ LogName = 'System'; Id = 7036; StartTime = (Get-Date).AddDays(-1) }
```

Im ersten Fall muss PowerShell jedes Ereignis des Protokolls laden und als Objekt aufbauen, bevor es verworfen wird. Im zweiten Fall liefert der Ereignisdienst nur die passenden.

> **Praxisbeispiel**
>
> Ein Server startet nachts unerwartet neu. Ein Skript holt für das Zeitfenster aus `System` und `Application` alle Ereignisse der Stufen Kritisch bis Warnung, sortiert sie nach Zeit und zeigt zehn Minuten davor und danach. Das Ergebnis: Ein Update-Dienst hat einen geplanten Neustart ausgelöst, kurz darauf hat eine Anwendung den Dienst nicht mehr erreicht. Das Skript läuft nun als Standardwerkzeug für alle Serverstörungen.

## 6.5 Benutzer und Gruppen lokal

Jeder Windows-Rechner hat eine eigene Benutzerdatenbank für lokale Konten und Gruppen, neben der Domäne. Lokale Konten braucht man für Dienste, Notfallzugänge (zum Beispiel das integrierte Administratorkonto), Arbeitsgruppenrechner und Testumgebungen. In Domänen verwaltet man Benutzer und Gruppen dagegen im Active Directory (Modul 10); diese Lerneinheit betrifft nur die lokale Seite. Das Modul *Microsoft.PowerShell.LocalAccounts* enthält laut Microsoft-Dokumentation (abgerufen, Windows PowerShell 5.1) die Cmdlets `Get-LocalUser`, `New-LocalUser`, `Set-LocalUser`, `Enable-LocalUser`, `Disable-LocalUser`, `Rename-LocalUser`, `Remove-LocalUser` sowie `Get-LocalGroup`, `New-LocalGroup`, `Set-LocalGroup`, `Rename-LocalGroup`, `Remove-LocalGroup`, `Get-LocalGroupMember`, `Add-LocalGroupMember` und `Remove-LocalGroupMember`. Die Seite vermerkt außerdem, dass das Modul in einer 32-Bit-PowerShell auf einem 64-Bit-System nicht verfügbar ist. Unter Linux fehlen die Cmdlets (im Test: `Get-LocalUser` nicht vorhanden); dort nutzt man `useradd`, `usermod` und `groupadd` über native Aufrufe. Ob die Cmdlets in der aktuellen PowerShell-7-Version unter Windows ohne Zusatzschritt zur Verfügung stehen, wurde nicht geprüft; in Windows PowerShell 5.1 sind sie vorhanden.

Das *Prinzip der geringsten Rechte* (Modul 8.4) gilt hier doppelt: Ein Konto bekommt nur die Gruppenmitgliedschaften, die es braucht, und Mitgliedschaften in der Gruppe `Administratoren` sind die Ausnahme, nicht die Regel. Typische Aufgaben sind das Anlegen eines Dienstkontos ohne Anmeldeberechtigung am Desktop, das Hinzufügen und Entfernen von Mitgliedern, das Deaktivieren von Konten ehemaliger Mitarbeiter und die Prüfung, wer lokaler Administrator ist.

```powershell
# Windows (erhoehte Rechte): lokale Administratoren auflisten
Get-LocalGroupMember -Group 'Administrators' |
    Select-Object Name, ObjectClass, PrincipalSource

# Benutzer anlegen: Kennwort als SecureString, laeuft nie ab ist hier bewusst NICHT gesetzt
$kennwort = Read-Host -AsSecureString 'Kennwort fuer das neue Konto'
New-LocalUser -Name 'svc-berichte' -Password $kennwort `
    -Description 'Dienstkonto fuer Berichtsjob' -PasswordNeverExpires:$false `
    -UserMayNotChangePassword:$false -AccountNeverExpires

# Zur Gruppe fuer Batchanmeldungen... oder zu einer eigenen Fachgruppe hinzufuegen
New-LocalGroup -Name 'Berichte-Leser' -Description 'Lesezugriff auf Berichte'
Add-LocalGroupMember -Group 'Berichte-Leser' -Member 'svc-berichte'

# Konto deaktivieren statt loeschen (Nachvollziehbarkeit)
Disable-LocalUser -Name 'svc-berichte'
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordLastSet
```

Der erste Block listet die Mitglieder der lokalen Administratorgruppe (unter deutschen Windows-Systemen heißt die Gruppe „Administratoren“; das Cmdlet akzeptiert den lokalisierten Namen, deshalb sind Skripte, die für mehrere Sprachen laufen sollen, besser mit der festen Kennung (SID) `S-1-5-32-544` aufgebaut). Der zweite Block legt ein Konto an: Das Kennwort kommt als `SecureString` über `Read-Host -AsSecureString`, also nicht im Skript (Modul 8.6). Der dritte Block legt eine eigene Gruppe an und fügt das Konto hinzu; eine Fachgruppe mit klarem Zweck ist besser als die Mitgliedschaft in einer allgemeinen Gruppe. Der vierte Block deaktiviert das Konto statt es zu löschen, weil ein gelöschtes Konto auch seine SID verliert, mit der Dateirechte und Verweise verbunden sind; ein deaktiviertes Konto lässt sich nachvollziehen und bei Bedarf wieder aktivieren. Die Beispiele sind auf Syntax geprüft; Parameternamen entsprechen der Cmdlet-Liste der Dokumentation, die einzelnen Cmdlet-Seiten wurden nicht abgerufen.

> **Ergänzung aus der Nachrecherche**
>
> **Hinweis zu Kennwörtern und Dienstkonten.** Ein eigenes lokales Konto mit festem Kennwort ist für Dienste die schwächere Lösung. In Active-Directory-Umgebungen sind gruppenverwaltete Dienstkonten (gMSA) besser, weil das Betriebssystem das Kennwort regelmäßig selbst wechselt (Modul 8.4); in Azure ersetzen verwaltete Identitäten Kennwörter (Modul 10). Lokale Konten bleiben für isolierte Systeme und Notfallzugänge.

> **Merke**
>
> Lokale Konten mit `Get-LocalUser`, `New-LocalUser`, `Get-LocalGroupMember` und Verwandten (nur Windows, Modul LocalAccounts). Kennwörter als `SecureString`, nie im Skript. Geringste Rechte: Fachgruppen statt Administratoren. Deaktivieren statt löschen. Mehrsprachige Skripte nutzen die SID statt des Gruppennamens. In Domänen: AD, gMSA.

```powershell
# FALSCH: Kennwort im Klartext im Skript, Konto gleich zum Administrator gemacht
$pw = ConvertTo-SecureString 'Sommer2026!' -AsPlainText -Force
New-LocalUser -Name 'bericht' -Password $pw
Add-LocalGroupMember -Group 'Administrators' -Member 'bericht'

# RICHTIG: Kennwort aus Tresor/Abfrage, Fachgruppe mit minimalen Rechten
$pw = Get-Secret -Name 'BerichtKonto'
New-LocalUser -Name 'bericht' -Password $pw -AccountNeverExpires
Add-LocalGroupMember -Group 'Berichte-Leser' -Member 'bericht'
```

Im ersten Fall steht das Kennwort im Skript, im Verlauf und im Script-Block-Protokoll, und das Konto hat volle Rechte. Im zweiten Fall kommt es aus dem Tresor, und das Konto kann nur, was die Fachgruppe erlaubt.

> **Praxisbeispiel**
>
> Bei einer Prüfung stellt sich heraus, dass auf mehreren Servern ein ehemaliger Dienstleister weiterhin als lokaler Administrator eingetragen ist. Ein Skript listet auf allen Servern die Mitglieder der Administratorgruppe auf, vergleicht sie mit einer Sollliste und meldet Abweichungen. Die Auswertung läuft nun monatlich.

## 6.6 Pakete und Software

Software von Hand auf vielen Rechnern zu installieren, ist langsam und fehleranfällig; deshalb gibt es *Paketmanager*, die Programme aus einem Katalog installieren, aktualisieren und entfernen. Für Windows sind drei Systeme verbreitet. *WinGet* (Windows Package Manager) ist Microsofts eigenes Kommandozeilenprogramm. Laut Dokumentation (abgerufen) ist es verfügbar auf Windows 10 ab Version 1809, Windows 11 und Windows Server 2025 als Teil des „App Installer“, und es ist ein Programm für die Kommandozeile, kein PowerShell-Modul; für PowerShell gibt es zusätzlich das Modul `Microsoft.WinGet.Client` aus der PowerShell Gallery. *Chocolatey* ist ein verbreiteter Paketmanager eines Drittanbieters (nicht abgerufen; Aussage aus allgemeinem Fachwissen). *PackageManagement* (das Modul mit `Get-Package` und `Install-Package`) ist die in PowerShell eingebaute Schnittstelle, die verschiedene Paketquellen über Anbieter anspricht; im Test unter Linux waren `Get-Package` und `Install-Package` vorhanden, `winget` dagegen nicht. Für PowerShell-Module selbst nimmt man *PSResourceGet* (Modul 2 und 8.9).

Für Automation zählt, dass die Installation *nicht-interaktiv* abläuft: ohne Rückfragen, ohne Fenster, mit prüfbarem Ergebnis. WinGet bietet dafür Schalter, die in der abgerufenen Dokumentation genannt sind: `--disable-interactivity` schaltet Rückfragen ab, `--verbose` schreibt ausführliche Protokolle, `winget export` und `winget import` schreiben und installieren Paketlisten, `winget pin` hält Pakete auf einer Version, und `winget list` zeigt installierte Pakete. Weitere Schalter wie `--silent`, `--accept-package-agreements` und `--accept-source-agreements` sind bei `winget install` üblich, die zugehörige Befehlsseite wurde aber nicht abgerufen; prüfen Sie sie mit `winget install --help`.

```powershell
# Windows: nicht-interaktive Installation mit Pruefung des Ergebnisses
function Install-WinGetPaket {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Id,
        [string] $Version
    )
    if (-not (Get-Command winget -ErrorAction SilentlyContinue)) { throw 'winget nicht gefunden' }
    $argumente = @('install', '--id', $Id, '--exact', '--silent',
                   '--accept-package-agreements', '--accept-source-agreements',
                   '--disable-interactivity')
    if ($Version) { $argumente += @('--version', $Version) }

    if ($PSCmdlet.ShouldProcess($Id, 'Installieren')) {
        & winget @argumente
        if ($LASTEXITCODE -ne 0) { throw "winget endete mit Exit-Code $LASTEXITCODE (Paket $Id)" }
    }
}

# Paketliste exportieren und auf anderem Rechner wieder einspielen
winget export -o "$env:TEMP\pakete.json"
winget import -i "$env:TEMP\pakete.json" --disable-interactivity
```

Die Funktion prüft zuerst, ob `winget` vorhanden ist, und baut die Argumente als Array auf; der Aufruf `& winget @argumente` übergibt sie einzeln und vermeidet damit das Quoting-Problem von `Start-Process` (Lerneinheit 6.3). Die Installation wird über `ShouldProcess` abgesichert. Entscheidend ist die Auswertung von `$LASTEXITCODE`: Ein natives Programm löst keinen PowerShell-Fehler aus, wenn es scheitert (Modul 5.1); nur der Exit-Code verrät den Misserfolg. Mit `--exact` wird die Kennung genau verglichen, damit nicht ein ähnlich benanntes Paket installiert wird. Der zweite Block zeigt Export und Import einer Paketliste, mit der man einen Standardsatz an Software auf neue Rechner bringt. Das Beispiel ist auf Syntax geprüft und konnte unter Linux nicht ausgeführt werden.

### Sicherheit und Reproduzierbarkeit.

Pakete aus öffentlichen Katalogen sind Fremdcode (Modul 8.9). Für Automation in Unternehmen heißt das: Quellen einschränken, Versionen festlegen (`--version` oder `winget pin`), Installer vor der Freigabe in einer Testumgebung prüfen und nach Möglichkeit eine interne Quelle verwenden. Die Dokumentation weist zudem auf Administratorfragen hin: Ohne Administratorrechte können manche Installer eine Erhöhung (UAC-Abfrage) verlangen, und wer sie ablehnt, bricht die Installation ab; in einer Administratorkonsole entfallen die Abfragen, weshalb man dort nur vertrauenswürdige Software installieren sollte.

> **Ergänzung aus der Nachrecherche**
>
> **Hinweise zu WinGet aus der Dokumentation.** WinGet steht erst zur Verfügung, nachdem sich ein Benutzer erstmals angemeldet hat, weil der Microsoft Store die Registrierung asynchron vornimmt; für frische Systeme nennt die Seite einen Registrierungsbefehl (`Add-AppxPackage -RegisterByFamilyName ...`). In der Windows Sandbox fehlt WinGet; dort lässt es sich mit `Repair-WinGetPackageManager` aus dem Modul `Microsoft.WinGet.Client` einrichten. Unterstützte Installationsformate sind unter anderem EXE (mit Silent-Schaltern), MSI, MSIX, APPX, ZIP und PORTABLE. Diese Details können sich ändern; prüfen Sie den aktuellen Stand in der Dokumentation.

> **Merke**
>
> Paketmanager: WinGet (Kommandozeile, Windows), Chocolatey (Drittanbieter), PackageManagement (`Get-Package`), PSResourceGet (PowerShell-Module). Nicht-interaktiv installieren (`--silent`, `--disable-interactivity`), Argumente als Array, `$LASTEXITCODE` prüfen, Versionen festlegen, Quellen einschränken.

```powershell
# FALSCH: Rueckfrage moeglich, neueste Version, Ergebnis nie geprueft
winget install Git
'fertig'

# RICHTIG: genau dieses Paket, feste Version, Ergebnis geprueft
Install-WinGetPaket -Id 'Git.Git' -Version '2.47.0'
```

Im ersten Fall wartet der Befehl in einer unbeaufsichtigten Sitzung eventuell auf eine Eingabe, installiert jeweils die neueste Version und meldet „fertig“ auch bei einem Fehler. Im zweiten Fall ist alles festgelegt und geprüft (die Versionsnummer ist ein Beispiel).

> **Praxisbeispiel**
>
> Für neue Entwicklerrechner pflegt ein Team eine JSON-Liste mit Paketkennungen und Versionen. Ein Skript installiert sie nicht-interaktiv, protokolliert jedes Paket mit Exit-Code und meldet am Ende, was gefehlt hat. Ein neuer Rechner ist so in einer Viertelstunde einsatzbereit, und alle Rechner haben dieselben Versionen.

## 6.7 Netzwerkdiagnose

„Das Netzwerk geht nicht“ ist eine Meldung, aber keine Diagnose. Ein systematisches Vorgehen teilt die Strecke in Schichten und prüft sie von unten nach oben: Hat der Rechner eine gültige IP-Konfiguration (Adresse, Netzmaske, Gateway, DNS-Server)? Ist das Gateway erreichbar? Wird der Zielname per DNS aufgelöst? Ist der Zielrechner per Ping erreichbar? Ist der Zielport offen? Antwortet der Dienst? Wo die erste Prüfung scheitert, liegt das Problem, und man spart sich die Suche in den oberen Schichten.

Unter Windows bietet das Modul NetTCPIP dafür Cmdlets. `Get-NetIPConfiguration` zeigt Adressen, Gateway und DNS-Server je Schnittstelle, `Resolve-DnsName` löst Namen auf und zeigt die Antwort samt Eintragstyp (A, AAAA, CNAME, MX, TXT), und `Test-NetConnection` prüft Verbindungen. Laut Dokumentation (abgerufen) zeigt `Test-NetConnection` Diagnoseinformationen zu einer Verbindung und unterstützt Ping-Tests, TCP-Tests, Routenverfolgung und Routendiagnose; je nach Parametern enthält die Ausgabe das Ergebnis der DNS-Auflösung, die IP-Schnittstellen und die Bestätigung des Verbindungsaufbaus. Der Parameter `-Port` testet einen TCP-Port, `-CommonTCPPort` akzeptiert die Namen `HTTP`, `RDP`, `SMB` und `WINRM`, `-TraceRoute` verfolgt die Route, und `-InformationLevel Quiet` liefert nur einen Wahrheitswert, was sich für Skripte eignet. Die Ausgabe enthält unter anderem `PingSucceeded` und `TcpTestSucceeded`. Unter Linux gibt es `Test-NetConnection` und `Resolve-DnsName` nicht (im Test nicht vorhanden); stattdessen existiert in PowerShell 7 ein plattformübergreifendes `Test-Connection` mit TCP-Test.

```powershell
# Windows: Schichtweise Diagnose
Get-NetIPConfiguration | Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DnsServer
Resolve-DnsName -Name 'www.contoso.com' -Type A
Test-NetConnection -ComputerName 'srv01' -Port 443 -InformationLevel Detailed

# Als Funktion: Ergebnis als Objekt, damit viele Ziele auswertbar sind
function Test-Ziel {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)] [string] $Name,
        [Parameter(Mandatory)] [ValidateRange(1, 65535)] [int] $Port,
        [ValidateRange(1, 30)] [int] $TimeoutSekunden = 3
    )
    process {
        $dns = $null; $tcp = $false; $fehler = $null
        try {
            $dns = [System.Net.Dns]::GetHostAddresses($Name) | Select-Object -First 1
            $client = [System.Net.Sockets.TcpClient]::new()
            try {
                $aufgabe = $client.ConnectAsync($dns, $Port)
                $tcp = $aufgabe.Wait([TimeSpan]::FromSeconds($TimeoutSekunden)) -and $client.Connected
                if (-not $tcp) { $fehler = 'Zeitlimit oder abgelehnt' }
            } finally { $client.Dispose() }
        } catch { $fehler = $_.Exception.InnerException.Message ?? $_.Exception.Message }
        [pscustomobject]@{ Name = $Name; Port = $Port; Adresse = $dns; Offen = $tcp; Fehler = $fehler }
    }
}
'localhost', 'gibt-es-nicht.invalid' | Test-Ziel -Port 22
```

Der erste Block zeigt die Standardabfolge unter Windows: Konfiguration, Namensauflösung, Portprobe mit Details. Der zweite Block ist eine plattformübergreifende Funktion, die diese Prüfung als Objekt liefert und für viele Ziele per Pipeline funktioniert. Sie löst den Namen mit `[System.Net.Dns]::GetHostAddresses` auf (die .NET-Methode funktioniert überall), öffnet mit `TcpClient.ConnectAsync` eine Verbindung und wartet mit `Wait` höchstens die angegebene Zeit. Das Zeitlimit ist entscheidend: Ohne eines wartet ein Verbindungsversuch auf einen blockierten Port (den eine Firewall stillschweigend verwirft) mehrere Sekunden bis Minuten. Der `finally`-Block gibt den Client in jedem Fall frei, und die Fehlermeldung wird aus der inneren Ausnahme gelesen, weil die .NET-Fehler gekapselt sind. Der Operator `??` (ab PowerShell 7) liefert den ersten Wert, der nicht `$null` ist. Im Test wurde `[System.Net.Dns]::GetHostAddresses('localhost')` zu `127.0.0.1` aufgelöst. Der Testlauf unter Linux mit `'localhost', 'gibt-es-nicht.invalid' | Test-Ziel -Port 22` lieferte für `localhost` die Adresse `127.0.0.1`, `Offen = False` und die Meldung „One or more errors occurred. (Connection refused)“ (der Rechner ist erreichbar, aber am Port lauscht niemand), für den zweiten Namen `Offen = False` und „Name or service not known“ (die Namensauflösung scheitert). Die Meldung der ersten Zeile zeigt, dass die innere Ausnahme hier ein `AggregateException` ist; wer nur den Kern der Meldung will, wertet `$_.Exception.GetBaseException().Message` aus.

### Probleme eingrenzen.

Aus den Ergebnissen folgt die Ursache fast mechanisch. Schlägt die Namensauflösung fehl, aber ein Ping auf die IP-Adresse klappt, ist DNS schuld (falscher DNS-Server, fehlender Eintrag, Suchsuffix). Klappt der Ping, aber der Port ist zu, läuft der Dienst nicht, oder eine Firewall blockiert ihn. Ist *nur* der Ping tot, heißt das noch nichts, denn viele Firewalls verwerfen ICMP absichtlich; der TCP-Test auf den echten Port ist die verlässlichere Aussage. Bei langen Zeitüberschreitungen deutet es auf eine stille Firewall oder fehlende Route, bei einer sofortigen Ablehnung (*Connection refused*) darauf, dass der Rechner erreichbar ist, aber kein Dienst am Port lauscht. Mit `-TraceRoute` sieht man, an welchem Router die Strecke endet.

> **Ergänzung aus der Nachrecherche**
>
> **Test-Connection in PowerShell 7 (laut eigenem Test).** Das Cmdlet existiert in PowerShell 7 plattformübergreifend und bietet laut Parameterliste einen TCP-Test (`-TcpPort`) und ein Zeitlimit (`-TimeoutSeconds`); im Testlauf unter Linux lieferte `Test-Connection -TargetName localhost -TcpPort 1 -Count 1 -TimeoutSeconds 1` bei einem geschlossenen Port `False`. Der Parameter heißt in PowerShell 7 `-TargetName`, in Windows PowerShell 5.1 `-ComputerName`; Skripte für beide Versionen müssen das berücksichtigen.

> **Merke**
>
> Schichtweise prüfen: IP-Konfiguration, Gateway, DNS, Ping, Port, Dienst. Windows: `Get-NetIPConfiguration`, `Resolve-DnsName`, `Test-NetConnection -Port`. Plattformübergreifend: `Test-Connection -TcpPort`, .NET-Klassen. Immer ein Zeitlimit setzen. Ping allein beweist nichts, der TCP-Test auf den echten Port schon.

```powershell
# FALSCH: nur Ping, kein Zeitlimit, nur Wahrheitswert ohne Ursache
if (Test-Connection srv01 -Count 1 -Quiet) { 'Server ok' } else { 'Server down' }

# RICHTIG: Port testen, Namensaufloesung getrennt betrachten, Ursache mitliefern
$e = 'srv01' | Test-Ziel -Port 443 -TimeoutSekunden 3
if ($e.Offen) { 'Dienst erreichbar' } else { "Dienst nicht erreichbar: $($e.Fehler)" }
```

Im ersten Fall meldet „Server down“ auch einen laufenden Server, dessen Firewall ICMP verwirft, und liefert keinen Hinweis auf die Ursache. Im zweiten wird der echte Dienst getestet, und die Meldung nennt den Grund.

> **Praxisbeispiel**
>
> Eine Anwendung meldet sporadisch „Datenbank nicht erreichbar“. Ein Skript prüft alle fünf Minuten Namensauflösung und Port der Datenbank und schreibt ein Protokoll. Nach einem Tag zeigt es, dass die Auflösung zeitweise einen alten Eintrag liefert; ein DNS-Server hatte einen veralteten Datensatz. Ohne die getrennte Prüfung wäre das als „Netzwerk instabil“ abgetan worden.

## 6.8 WMI und CIM

*WMI* (Windows Management Instrumentation) ist die Verwaltungsschnittstelle von Windows: Eine große Datenbank von Klassen, die Hardware, Betriebssystem, Dienste, Netzwerk und vieles mehr beschreiben, abfragbar und teilweise änderbar. Beispiele sind `Win32_OperatingSystem`, `Win32_BIOS`, `Win32_LogicalDisk` und `Win32_Service`. *CIM* (Common Information Model) ist der herstellerneutrale Standard dahinter. Laut der abgerufenen Microsoft-Dokumentation gilt: Die alten WMI-Cmdlets (`Get-WmiObject` und Verwandte) sind veraltet und ab PowerShell 6 nicht mehr verfügbar; die CIM-Cmdlets wurden mit PowerShell 3.0 eingeführt, stehen im Modul `CimCmdlets`, und „for new development, use the CIM cmdlets“. Der Name führt zu Missverständnissen: Die CIM-Cmdlets *greifen auf WMI zu*; wer sagt „ich frage WMI mit den CIM-Cmdlets ab“, meint dasselbe. Unter Linux gibt es die Cmdlets nicht (im Test: `Get-CimInstance` nicht vorhanden).

Das Modul enthält laut Dokumentation unter anderem `Get-CimInstance`, `Get-CimClass`, `Get-CimAssociatedInstance`, `New-CimInstance`, `Set-CimInstance`, `Remove-CimInstance`, `Invoke-CimMethod`, `New-CimSession`, `Get-CimSession`, `Remove-CimSession`, `New-CimSessionOption` und `Register-CimIndicationEvent`.

```powershell
# Windows: Systeminformationen abfragen
Get-CimInstance -ClassName Win32_OperatingSystem |
    Select-Object Caption, Version, LastBootUpTime, FreePhysicalMemory, TotalVisibleMemorySize

# Nur die benoetigten Eigenschaften holen (spart bei Remote-Abfragen Zeit und Netzlast)
Get-CimInstance -ClassName Win32_BIOS -Property SerialNumber |
    Select-Object -ExpandProperty SerialNumber

# Mit Filter: Laufwerke mit wenig freiem Platz
Get-CimInstance -ClassName Win32_LogicalDisk -Filter 'DriveType = 3' |
    Where-Object { $_.FreeSpace / $_.Size -lt 0.1 } |
    Select-Object DeviceID, @{ n = 'FreiGB'; e = { [math]::Round($_.FreeSpace / 1GB, 1) } }

# Mit WQL-Abfrage
Get-CimInstance -Query 'SELECT Name, State FROM Win32_Service WHERE StartMode = "Auto" AND State <> "Running"'
```

Der erste Block holt das Betriebssystem und zeigt Name, Version, Startzeitpunkt und Speicher; `FreePhysicalMemory` ist in Kilobyte angegeben. Der zweite begrenzt mit `-Property` die abgefragten Eigenschaften; die Dokumentation merkt an, dass standardmäßig viele nie benutzte Eigenschaften mit abgerufen werden, was bei Remote-Abfragen unnötige Zeit und Netzlast kostet. Der dritte Block filtert serverseitig mit `-Filter` (WQL-Syntax, `=` und `<>` statt `-eq`); `DriveType = 3` sind lokale Festplatten. Der vierte Block zeigt die Abfragesprache WQL direkt, die man aus alten VBScripts übernehmen kann: Die Dokumentation zeigt, dass sich solche Abfragen ohne Änderung mit `Get-CimInstance -Query` verwenden lassen. Hier sucht sie automatisch startende Dienste, die nicht laufen, ein klassisches Gesundheitsmerkmal. Alle Beispiele sind auf Syntax geprüft und nur unter Windows ausführbar.

### Remote-Abfragen und Sitzungen.

Mit `-ComputerName` fragt `Get-CimInstance` einen entfernten Rechner ab. Dabei treten zwei Hürden auf. Erstens: Die Berechtigung. Laut der abgerufenen Dokumentation bekommt man „Access is denied“, wenn das ausführende Konto keine Rechte für WMI auf dem Zielrechner hat. Da `Get-CimInstance` keinen `-Credential`-Parameter besitzt, erzeugt man eine *CIM-Sitzung* mit `New-CimSession -ComputerName ... -Credential ...` und übergibt sie mit `-CimSession`. Das entspricht dem Prinzip der geringsten Rechte (Modul 8.4): Nur dieser Befehl läuft mit den erhöhten Rechten, nicht die ganze Konsole. Zweitens: Das Protokoll. `Get-CimInstance` verwendet standardmäßig WSMan (dasselbe Protokoll wie PowerShell-Remoting, Modul 7); der Zielrechner braucht Stack-Version 3.0, die `Test-WSMan` anzeigt. Für ältere Systeme kann eine Sitzung mit `New-CimSessionOption -Protocol Dcom` das ältere DCOM-Protokoll verwenden, das neuere Windows-Versionen aber oft in der Firewall blockieren. Sitzungen sind effizienter als Computernamen, weil der Verbindungsaufbau nur einmal erfolgt, und man räumt sie mit `Get-CimSession | Remove-CimSession` auf.

```powershell
# Windows: viele Rechner effizient mit einer Sitzung je Rechner abfragen
$cred = Get-Credential -Message 'Konto mit WMI-Leserechten'
$rechner = 'srv01', 'srv02', 'srv03'
$sitzungen = New-CimSession -ComputerName $rechner -Credential $cred -ErrorAction SilentlyContinue

try {
    Get-CimInstance -CimSession $sitzungen -ClassName Win32_OperatingSystem -Property Caption, LastBootUpTime |
        Select-Object PSComputerName, Caption, LastBootUpTime
    $fehlende = $rechner | Where-Object { $_ -notin $sitzungen.ComputerName }
    if ($fehlende) { Write-Warning "Keine Sitzung zu: $($fehlende -join ', ')" }
}
finally {
    $sitzungen | Remove-CimSession
}
```

Das Skript fragt die Zugangsdaten einmal ab und erzeugt für alle Rechner Sitzungen; `-ErrorAction SilentlyContinue` unterdrückt Fehler für nicht erreichbare Rechner, die man danach gezielt auswertet. Eine einzige `Get-CimInstance`-Abfrage fragt alle Sitzungen ab, und die Eigenschaft `PSComputerName` zeigt, woher die Zeile stammt. Der `finally`-Block räumt die Sitzungen in jedem Fall auf. Die Voraussetzungen müssen stimmen: Das Konto braucht WMI-Rechte auf den Zielen, und die Firewall muss das Protokoll erlauben (bei WSMan der WinRM-Port 5985 oder 5986; die Portnummern sind allgemeines Fachwissen und wurden hier nicht abgerufen).

> **Ergänzung aus der Nachrecherche**
>
> **Hinweis für plattformübergreifende Skripte.** Die CIM-Cmdlets gibt es unter Linux und macOS nicht. Wer Systeminformationen plattformübergreifend braucht, kombiniert .NET (`[System.Environment]`, `[System.IO.DriveInfo]::GetDrives()`), `Get-Process` und native Programme (`df`, `uname`). Im Test lieferten `$IsLinux` und `[System.Environment]::OSVersion` die nötigen Angaben, um zwischen Zweigen zu wählen.

> **Merke**
>
> CIM-Cmdlets (`Get-CimInstance`) statt der alten WMI-Cmdlets (ab PowerShell 6 entfernt). Klassen wie `Win32_OperatingSystem`. `-Property` und `-Filter` begrenzen Daten, WQL über `-Query`. Remote: `New-CimSession` mit `-Credential`, Standard WSMan, DCOM nur für Altsysteme; Sitzungen aufräumen. Nur Windows.

```powershell
# FALSCH: alte, entfernte Cmdlets; alles holen, dann filtern; ganze Konsole als Admin
Get-WmiObject Win32_Service | Where-Object { $_.State -ne 'Running' }

# RICHTIG: CIM, serverseitig filtern, nur noetige Eigenschaften
Get-CimInstance -ClassName Win32_Service -Filter 'State <> "Running"' -Property Name, State, StartMode
```

Im ersten Fall läuft der Befehl in PowerShell 7 gar nicht, weil `Get-WmiObject` dort fehlt, und holt in Windows PowerShell alle Dienste vollständig. Im zweiten filtert WMI selbst und liefert nur die benötigten Felder.

> **Praxisbeispiel**
>
> Für ein Inventar fragt ein Skript nachts alle Server per CIM-Sitzung nach Betriebssystemversion, letztem Start, Speicher und freiem Plattenplatz ab und schreibt die Ergebnisse in eine CSV-Datei. Server mit Startzeitpunkt älter als 60 Tage werden für Updates markiert, Server ohne Sitzung gelten als Fehlerfall und tauchen im Bericht auf.

## 6.9 Backup-nahe Aufgaben

Ein Backup, das nie wiederhergestellt wurde, ist eine Hoffnung, kein Backup. Diese Lerneinheit behandelt keine vollständigen Sicherungsprodukte, sondern die Bausteine, die in Skripten oft gebraucht werden: Dateien kopieren, Exporte erzeugen, Archive packen und, vor allem, *prüfen*, dass die Kopie vollständig und unverändert ist und dass man sie wirklich wieder einspielen kann. Die Reihenfolge der Fragen lautet: Was wird gesichert? Wohin, und liegt die Kopie auf einem anderen Medium oder Standort als das Original? Wie lange wird sie aufbewahrt? Wie prüfe ich, dass sie korrekt ist? Und wie spiele ich sie zurück?

Für das Kopieren genügt `Copy-Item -Recurse` bei kleinen Aufgaben. Unter Windows ist `robocopy` das Werkzeug für große Datenmengen: Es setzt unterbrochene Kopien fort, wiederholt bei Fehlern, behält Zeitstempel und Rechte und kann spiegeln. Es ist ein natives Programm und meldet seinen Erfolg über Exit-Codes, bei denen Werte unter 8 Erfolg bedeuten (allgemeines Fachwissen, nicht abgerufen; unter Linux war `robocopy` nicht vorhanden). Das gilt als besondere Falle, weil `$LASTEXITCODE -ne 0` bei robocopy falsch wäre. Für Archive bietet PowerShell `Compress-Archive` und `Expand-Archive`; im Test packte `Compress-Archive` fünf Dateien in eine ZIP-Datei, und `Expand-Archive` stellte sie wieder her (fünf Dateien).

### Integrität prüfen.

Dass eine Kopie existiert, beweist nicht, dass sie stimmt. Eine *Prüfsumme* (Hash) ist ein Fingerabdruck fester Länge, der sich bei der kleinsten Änderung des Inhalts völlig ändert. `Get-FileHash` berechnet sie (Standard SHA-256; im Test meldete `.Algorithm` `SHA256`). Das Muster: Beim Sichern ein *Manifest* mit Pfad, Hash und Größe jeder Datei schreiben; beim Prüfen und Wiederherstellen die Hashes der Kopie mit dem Manifest vergleichen. So erkennt man fehlende, zusätzliche und veränderte Dateien.

```powershell
function New-HashManifest {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Quelle,
        [Parameter(Mandatory)] [string] $Ziel
    )
    $q = (Resolve-Path -LiteralPath $Quelle).ProviderPath
    Get-ChildItem -LiteralPath $q -File -Recurse | ForEach-Object {
        [pscustomobject]@{
            Pfad   = [IO.Path]::GetRelativePath($q, $_.FullName)
            Sha256 = (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash
            Bytes  = $_.Length
        }
    } | Export-Csv -LiteralPath $Ziel -NoTypeInformation -Encoding utf8
}

function Test-HashManifest {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Ordner,
        [Parameter(Mandatory)] [string] $Manifest
    )
    $o = (Resolve-Path -LiteralPath $Ordner).ProviderPath
    foreach ($e in Import-Csv -LiteralPath $Manifest) {
        $p = Join-Path $o $e.Pfad
        $status = if (-not (Test-Path -LiteralPath $p)) { 'FEHLT' }
                  elseif ((Get-FileHash -LiteralPath $p -Algorithm SHA256).Hash -ne $e.Sha256) { 'ABWEICHEND' }
                  else { 'OK' }
        [pscustomobject]@{ Pfad = $e.Pfad; Status = $status }
    }
}

New-HashManifest -Quelle $quelle -Ziel $manifest
Copy-Item -LiteralPath $quelle -Destination $kopie -Recurse
Test-HashManifest -Ordner $kopie -Manifest $manifest | Where-Object Status -ne 'OK'
```

`New-HashManifest` geht rekursiv durch die Quelle und schreibt für jede Datei den relativen Pfad, den SHA-256-Wert und die Größe in eine CSV-Datei. Der relative Pfad ist wichtig, damit das Manifest auch nach dem Verschieben der Kopie auf ein anderes Laufwerk gültig bleibt. `Test-HashManifest` liest das Manifest, berechnet für jede Datei der Kopie den Hash neu und vergleicht; es meldet `FEHLT`, `ABWEICHEND` oder `OK`. Im Test wurde in der Kopie eine Datei verändert und eine gelöscht; die Prüfung meldete für die erste `ABWEICHEND`, für die zweite `FEHLT`. Die Auswertung blieb dabei zunächst leer, weil die Tabellenausgabe ohne Terminal keine Breite hat; das war kein Fehler der Funktion (mit `Out-String -Width 120` oder `ConvertTo-Json` erschien die Ausgabe). Das Beispiel hat eine Lücke: Zusätzliche Dateien in der Kopie, die nicht im Manifest stehen, werden nicht gemeldet; für ein vollständiges Prüfwerkzeug sollten Sie diesen Fall in der Übung ergänzen.

### Restore-Fähigkeit.

Ein Skript, das sichert, sollte drei Dinge bedenken, damit die Wiederherstellung funktioniert. Erstens: Das Ziel der Sicherung muss unabhängig vom Original sein (anderer Datenträger, anderer Rechner), sonst vernichtet ein Ausfall beides. Zweitens: Die Sicherung muss alles enthalten, was zur Wiederherstellung nötig ist: nicht nur die Daten, sondern auch Konfiguration, Rechte (zum Beispiel als exportierte ACL oder SDDL) und das Manifest. Drittens: Die Wiederherstellung muss *geübt* sein. Das einfachste Verfahren ist ein automatischer Test: Das Skript stellt regelmäßig eine zufällige Stichprobe an einen Testort wieder her und prüft die Hashes. Die Aufbewahrung regelt man ebenfalls im Skript, etwa durch Ordner mit Datumsnamen und Löschen alter Sicherungen mit der gesicherten Funktion aus Lerneinheit 6.1.

```powershell
# Sicherung mit Datumsordner, Manifest, Pruefung und Rueckmeldung per Exit-Code
function Invoke-Sicherung {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $Quelle,
        [Parameter(Mandatory)] [string] $ZielBasis
    )
    $stempel = Get-Date -Format 'yyyyMMdd-HHmmss'
    $ziel = Join-Path $ZielBasis $stempel
    if (-not $PSCmdlet.ShouldProcess($ziel, "Sichern von $Quelle")) { return }

    New-Item -ItemType Directory -Path $ziel -Force | Out-Null
    $manifest = Join-Path $ziel 'manifest.csv'
    $daten = Join-Path $ziel 'daten'
    New-HashManifest -Quelle $Quelle -Ziel $manifest
    Copy-Item -LiteralPath $Quelle -Destination $daten -Recurse

    $probleme = @(Test-HashManifest -Ordner $daten -Manifest $manifest | Where-Object Status -ne 'OK')
    if ($probleme.Count -gt 0) {
        throw "Sicherung fehlerhaft: $($probleme.Count) Abweichung(en), z. B. $($probleme[0].Pfad)"
    }
    "Sicherung ok: $ziel"
}
```

Die Funktion legt einen Ordner mit Zeitstempel an, schreibt das Manifest *vor* dem Kopieren (damit es den Zustand der Quelle zum Sicherungszeitpunkt widerspiegelt), kopiert und prüft die Kopie sofort gegen das Manifest. Bei Abweichungen wirft sie eine Ausnahme mit einem Beispiel, sodass ein Planer oder eine Pipeline den Lauf als fehlgeschlagen erkennt. Der Erfolgsfall meldet nur den Zielordner. Die Funktion nutzt die zuvor gezeigten Funktionen und wurde in ihren Bausteinen unter Linux geprüft.

> **Merke**
>
> Sichern heißt: kopieren, prüfen, wiederherstellen können. Prüfsummen-Manifest (`Get-FileHash`) mit relativen Pfaden, Kopie sofort gegen das Manifest prüfen, Ziel unabhängig vom Original, Rechte und Konfiguration mitsichern, Wiederherstellung regelmäßig testen. `robocopy`-Exit-Codes unter 8 bedeuten Erfolg. Fehler als Ausnahme melden.

```powershell
# FALSCH: kopieren und Erfolg annehmen
Copy-Item -Path $quelle -Destination $ziel -Recurse
'Backup fertig'

# RICHTIG: Manifest schreiben, kopieren, pruefen
Invoke-Sicherung -Quelle $quelle -ZielBasis $zielBasis
```

Im ersten Fall bleibt eine unvollständige Kopie (gesperrte Datei, voller Datenträger) unbemerkt, und „Backup fertig“ ist gelogen. Im zweiten Fall würde jede Abweichung als Fehler gemeldet.

> **Praxisbeispiel**
>
> Nach einem Ransomware-Vorfall zeigt sich, dass die nächtliche Kopie seit Wochen nur noch leere Dateien enthielt, weil eine Anwendung die Dateien sperrte. Die Kopie „lief erfolgreich“, weil niemand sie prüfte. Seitdem schreibt das Skript ein Manifest, vergleicht die Hashes, stellt wöchentlich eine Stichprobe testweise wieder her und meldet Abweichungen an das Überwachungssystem.

## 6.10 Übungsaufgaben zu Modul 06

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Aufgaben 1 bis 4 prüfen Verständnis, Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung folgen im Anschluss. Aufgaben mit Windows-Bezug brauchen eine Testumgebung.

**Aufgabe 1.** Ein Kollege löscht mit `Get-ChildItem -Recurse -Filter *.tmp | Remove-Item`. Nennen Sie drei Risiken dieser Zeile und drei Sicherungen, die Sie einbauen würden.

**Aufgabe 2.** Erklären Sie den Unterschied zwischen Schlüssel und Wert in der Registrierung und nennen Sie jeweils die passenden Cmdlets. Warum ist der Typ `DWord` statt `String` wichtig?

**Aufgabe 3.** Ein Skript startet ein Programm mit `Start-Process -ArgumentList '-c','exit 3' -Wait` und meldet dennoch Erfolg. Was ist schiefgelaufen, und wie lässt sich der Fehler vermeiden?

**Aufgabe 4.** Warum ist `Get-WinEvent -LogName System | Where-Object Id -eq 7036` schlechter als `-FilterHashtable`? Nennen Sie die drei Filtermöglichkeiten von `Get-WinEvent`.

**Aufgabe 5.** Schreiben Sie (plattformübergreifend) eine Funktion `Get-GrosseDateien`, die in einem Ordner die `N` größten Dateien rekursiv liefert (Pfad, Größe in MB, letzte Änderung), mit Prüfung des Ordners und Obergrenze für `N`.

**Aufgabe 6.** Schreiben Sie ein Skript (Windows), das alle automatisch startenden Dienste findet, die nicht laufen, und sie nach Bestätigung startet. Es soll `-WhatIf` unterstützen, nach dem Start den Zustand prüfen und eine Zusammenfassung mit Erfolg und Misserfolg je Dienst ausgeben.

**Aufgabe 7.** Erweitern Sie `Test-HashManifest` (Lerneinheit 6.9) so, dass es auch Dateien meldet, die in der Kopie vorhanden sind, aber nicht im Manifest stehen (Status `ZUSAETZLICH`).

**Aufgabe 8.** Entwerfen Sie ein Wartungsskript für einen Dateiserver, das jeden Sonntag (a) Protokolle älter als 60 Tage löscht, (b) ein Hash-Manifest und eine Kopie der Berichte auf ein zweites Laufwerk schreibt, (c) die Kopie prüft und (d) bei Problemen mit Exit-Code 1 endet. Beschreiben Sie Aufbau und Absicherungen, und schreiben Sie das Gerüst.

## 6.11 Lösungen zu Modul 06

### Aufgabe 1.

Risiken: Erstens trifft `-Recurse` alle Unterordner, auch solche, die man nicht im Blick hat (andere Anwendungen, Systemordner). Zweitens gibt es keinen Pfad: Ohne `-Path` gilt der aktuelle Ordner, der von woanders her gesetzt sein kann. Drittens wird nichts begrenzt oder bestätigt, ein falscher Filter löscht beliebig viel. Sicherungen: den Pfad ausdrücklich angeben, auflösen und gegen Wurzelverzeichnisse prüfen; die Trefferzahl begrenzen (`MaxAnzahl`); zuerst `-WhatIf` laufen lassen; `-LiteralPath` und `-File` verwenden; Alter als Bedingung; das Löschen protokollieren. Die Funktion `Remove-AlteDateien` aus 6.1 vereint das.

### Aufgabe 2.

Ein Schlüssel ist wie ein Ordner, ein Wert wie eine Datei mit Namen, Typ und Inhalt in diesem Ordner. Schlüssel bearbeitet man als Items (`Get-ChildItem`, `New-Item`, `Remove-Item`, `Test-Path`), Werte als Properties (`Get-ItemProperty`, `Get-ItemPropertyValue`, `New-ItemProperty`, `Set-ItemProperty`, `Remove-ItemProperty`). Der Typ ist wichtig, weil Programme einen bestimmten Typ erwarten: Ein `String` `'1'` ist kein `DWord` 1, und viele Programme ignorieren einen falschen Typ oder lesen ihn falsch.

### Aufgabe 3.

`Start-Process` verbindet die Elemente von `-ArgumentList` mit Leerzeichen zu einer Zeichenfolge und lässt Anführungszeichen nicht hinzukommen. Das Programm erhielt `-c exit 3`, die Shell führte `exit` ohne Zahl aus, und der Exit-Code war 0 (im Test bestätigt). Vermeidung: die Argumente mit eingebetteten Anführungszeichen übergeben (`'"exit 3"'`) oder eine einzige Zeichenfolge, mit `-PassThru -Wait` den Exit-Code lesen und prüfen, oder den Aufrufoperator `&` nutzen und `$LASTEXITCODE` auswerten.

### Aufgabe 4.

Im ersten Fall holt PowerShell alle Ereignisse des Protokolls und filtert erst danach; bei großen Protokollen dauert das sehr lange. Die Filterparameter werden beim Abruf angewendet und sind laut Dokumentation effizienter. Die drei Möglichkeiten: `-FilterHashtable`, `-FilterXPath` und `-FilterXml`.

### Aufgabe 5.

```powershell
function Get-GrosseDateien {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Ordner,
        [ValidateRange(1, 1000)] [int] $Anzahl = 10
    )
    if (-not (Test-Path -LiteralPath $Ordner -PathType Container)) {
        throw "Ordner nicht gefunden: $Ordner"
    }
    Get-ChildItem -LiteralPath $Ordner -File -Recurse -ErrorAction SilentlyContinue |
        Sort-Object Length -Descending |
        Select-Object -First $Anzahl -Property @(
            @{ n = 'Pfad';      e = { $_.FullName } },
            @{ n = 'MB';        e = { [math]::Round($_.Length / 1MB, 2) } },
            @{ n = 'Geaendert'; e = { $_.LastWriteTime } }
        )
}
```

Die Funktion prüft den Ordner, begrenzt `Anzahl` per `ValidateRange`, sammelt Dateien rekursiv (nicht lesbare Unterordner werden still übergangen, was man je nach Zweck anders halten kann), sortiert absteigend nach Größe und formt die Ausgabe mit berechneten Eigenschaften. `-First` nach `Sort-Object` nimmt die größten.

### Aufgabe 6.

```powershell
[CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'Medium')]
param([ValidateRange(1, 120)] [int] $TimeoutSekunden = 30)
Set-StrictMode -Version Latest

$kandidaten = Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }
$ergebnis = foreach ($d in $kandidaten) {
    $status = 'Uebersprungen'
    if ($PSCmdlet.ShouldProcess($d.Name, 'Starten')) {
        try {
            Start-Service -Name $d.Name -ErrorAction Stop
            $d.WaitForStatus('Running', [TimeSpan]::FromSeconds($TimeoutSekunden))
            $d.Refresh()
            $status = if ($d.Status -eq 'Running') { 'Erfolg' } else { "Misserfolg ($($d.Status))" }
        }
        catch { $status = "Misserfolg ($($_.Exception.Message))" }
    }
    [pscustomobject]@{ Dienst = $d.Name; Ergebnis = $status }
}
$ergebnis | Group-Object Ergebnis -NoElement | Format-Table Count, Name -AutoSize
$ergebnis | Where-Object Ergebnis -like 'Misserfolg*'
```

Das Skript sammelt Kandidaten, startet jeden hinter `ShouldProcess` (daher `-WhatIf` und `-Confirm`), wartet auf den Zielzustand, prüft ihn und fängt Fehler je Dienst ab, damit ein Fehler die Schleife nicht beendet. Am Ende gibt es eine Zusammenfassung und eine Fehlerliste. Dienste mit dem Starttyp `Automatic`, die nach dem Start regulär beendet sind (Trigger-Dienste), können zu Fehlalarmen führen; für den Einsatz sollte man eine Ausschlussliste ergänzen. Nur Windows, nur Syntax geprüft.

### Aufgabe 7.

```powershell
function Test-HashManifest {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Ordner,
        [Parameter(Mandatory)] [string] $Manifest
    )
    $o = (Resolve-Path -LiteralPath $Ordner).ProviderPath
    $bekannt = [System.Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase)
    foreach ($e in Import-Csv -LiteralPath $Manifest) {
        [void]$bekannt.Add($e.Pfad)
        $p = Join-Path $o $e.Pfad
        $status = if (-not (Test-Path -LiteralPath $p)) { 'FEHLT' }
                  elseif ((Get-FileHash -LiteralPath $p -Algorithm SHA256).Hash -ne $e.Sha256) { 'ABWEICHEND' }
                  else { 'OK' }
        [pscustomobject]@{ Pfad = $e.Pfad; Status = $status }
    }
    foreach ($f in Get-ChildItem -LiteralPath $o -File -Recurse) {
        $rel = [IO.Path]::GetRelativePath($o, $f.FullName)
        if (-not $bekannt.Contains($rel)) {
            [pscustomobject]@{ Pfad = $rel; Status = 'ZUSAETZLICH' }
        }
    }
}
```

Während das Manifest gelesen wird, merkt sich die Funktion alle bekannten Pfade in einer Menge (`HashSet`, schnell beim Nachschlagen). Danach durchläuft sie die Dateien der Kopie und meldet alle, die nicht in der Menge stehen. Der Vergleich ohne Beachtung der Groß-/Kleinschreibung entspricht Windows; auf Linux wäre `Ordinal` genauer. Das Manifest selbst sollte nicht im geprüften Ordner liegen, sonst meldet die Funktion es als zusätzlich.

### Aufgabe 8.

Aufbau: ein Skript mit Parametern (Protokollordner, Berichtsordner, Zielbasis, Aufbewahrungstage, Obergrenze), `Set-StrictMode`, `$ErrorActionPreference = 'Stop'`, ein Protokoll (Lerneinheit 5.6) und eine Struktur mit `try/catch/finally`. Absicherungen: Pfade auflösen und gegen Wurzelverzeichnisse prüfen, Obergrenze für das Löschen, `-WhatIf`-Fähigkeit, Ziel auf einem anderen Laufwerk (Prüfung, dass Quelle und Ziel nicht auf demselben Datenträger liegen), Prüfsummen, Fehlerzusammenfassung und `exit 1` bei Problemen.

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)] [string] $ProtokollOrdner,
    [Parameter(Mandatory)] [string] $BerichtsOrdner,
    [Parameter(Mandatory)] [string] $ZielBasis,
    [int] $AufbewahrenTage = 60
)
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
$probleme = [System.Collections.Generic.List[string]]::new()
try {
    $r = Remove-AlteDateien -Ordner $ProtokollOrdner -AlterTage $AufbewahrenTage -Confirm:$false -WhatIf:$WhatIfPreference
    Write-Verbose $r
}
catch { $probleme.Add("Aufraeumen: $($_.Exception.Message)") }
try {
    $s = Invoke-Sicherung -Quelle $BerichtsOrdner -ZielBasis $ZielBasis -WhatIf:$WhatIfPreference
    Write-Verbose $s
}
catch { $probleme.Add("Sicherung: $($_.Exception.Message)") }
if ($probleme.Count -gt 0) {
    $probleme | ForEach-Object { Write-Error $_ -ErrorAction Continue }
    exit 1
}
```

Das Gerüst ruft die gesicherten Funktionen aus 6.1 und 6.9 auf, sammelt Fehler getrennt je Schritt (ein Fehler beim Aufräumen verhindert die Sicherung nicht) und beendet sich mit Exit-Code 1, damit der Aufgabenplaner und die Überwachung es merken. `-WhatIf:$WhatIfPreference` reicht die Simulation an die Funktionen weiter. Zu ergänzen wären Protokolldatei, Prüfung der Datenträger und die Aufbewahrung alter Sicherungen.
