# Modul 07 - Remoting, Jobs und Skalierung

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (7.1 bis 7.8; 7.9 Übungsaufgaben, 7.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 10 h (8 Lerneinheiten à 1,25 h)      |
| Priorität                             | 2 (einzelne Lerneinheiten: 2 bzw. 3) |

> **Merke**
>
> Übersichtsblatt und Detailblatt stimmen in diesem Modul überein: 8 Lerneinheiten à 1,25 h ergeben 10 h. Nach dem Richtwert von mindestens einer Seite je Unterrichtsstunde wären 10 Seiten nötig; dieses Skript ist deutlich umfangreicher, weil die Themen viele Stolperfallen haben.

### Lernziele.

Nach diesem Modul können Sie Befehle auf entfernten Rechnern ausführen und dabei WinRM, SSH und lokale Sitzungen unterscheiden, Argumente und Ergebnisse von entfernten Aufrufen korrekt behandeln (Stichwort Serialisierung), dauerhafte Sitzungen für mehrstufige Aufgaben verwalten, Authentifizierungsverfahren einordnen und das Double-Hop-Problem erkennen, Hintergrundaufträge starten und auswerten, mit `ForEach-Object -Parallel` parallelisieren, viele Rechner kontrolliert mit Begrenzung, Zeitlimit und Wiederholung bearbeiten und JEA als Werkzeug für delegierte Administration einordnen.

| **Nr.** | **Lerneinheit**                | **Dauer** | **Prio** |
|:--------|:-------------------------------|:----------|:---------|
| 7.1     | PowerShell Remoting Grundlagen | 1,25 h    | 2        |
| 7.2     | Invoke-Command                 | 1,25 h    | 2        |
| 7.3     | PSSessions                     | 1,25 h    | 2        |
| 7.4     | Authentication und Delegation  | 1,25 h    | 2        |
| 7.5     | Jobs und Background Execution  | 1,25 h    | 2        |
| 7.6     | ForEach-Object -Parallel       | 1,25 h    | 2        |
| 7.7     | Fan-out Administration         | 1,25 h    | 3        |
| 7.8     | JEA Grundlagen                 | 1,25 h    | 3        |
|         | **Summe**                      | **10 h**  |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde.** Alles, was lokal läuft, wurde mit PowerShell 7.6.6 unter Linux ausgeführt: Hintergrundaufträge (`Start-Job`, `Start-ThreadJob`), `ForEach-Object -Parallel`, Serialisierung, ein Fan-out-Muster mit Wiederholung und die Erzeugung von JEA-Konfigurationsdateien. Echte *entfernte* Verbindungen konnten im Arbeitsbereich nicht aufgebaut werden: `Invoke-Command -ComputerName localhost` scheiterte mit „This parameter set requires WSMan, and no supported WSMan client library was found. WSMan is either not installed or unavailable for this system.“, und der Versuch einer SSH-Sitzung (`New-PSSession -HostName`) scheiterte, weil kein SSH-Client vorhanden war. Außerdem fehlten dort `Enable-PSRemoting`, `Test-WSMan`, `Register-PSSessionConfiguration`, `Test-PSSessionConfigurationFile` und die Cmdlets zum Trennen und Wiederverbinden von Sitzungen. Alle Beispiele mit `-ComputerName`, `-HostName` und Sitzungen sind deshalb auf Syntax geprüft und mit der abgerufenen Microsoft-Dokumentation abgeglichen, aber nicht ausgeführt. Testen Sie sie in einer Laborumgebung mit zwei Rechnern, bevor Sie sie produktiv einsetzen.

## 7.1 PowerShell Remoting Grundlagen

Bisher lief jeder Befehl auf dem Rechner, an dem man saß. In der Praxis verwaltet man aber Dutzende oder Tausende Server, auf die man nicht jedes Mal per Fernwartung wechseln will. *PowerShell Remoting* löst das: Ein Befehl wird auf einem entfernten Rechner ausgeführt, das Ergebnis kommt zurück, und alles geschieht mit denselben Cmdlets und Skripten wie lokal. Das entfernte Ausführen ist keine Besonderheit einzelner Cmdlets, sondern ein eigener Mechanismus: Auf dem Zielrechner läuft ein PowerShell-Prozess, der die Befehle entgegennimmt, ausführt und die Ergebnisse als Daten zurückschickt.

Die technische Grundlage nennt sich *PowerShell Remoting Protocol* (PSRP). Es ist ein Protokoll für den Austausch von Befehlen und Objekten und braucht einen *Transport*, der die Bytes sicher zwischen den Rechnern bewegt. PowerShell kennt zwei verbreitete Transporte. Der klassische ist *WinRM* (Windows Remote Management), Microsofts Umsetzung des Standards WS-Management; er ist ein Dienst unter Windows und die Grundlage von Remoting in Windows PowerShell und PowerShell 7 unter Windows. Der zweite ist *SSH*; PowerShell 7 kann laut der abgerufenen Dokumentation Remoting über SSH nutzen und sich damit mit jedem Windows-, macOS- oder Linux-Rechner verbinden, auf dem SSH läuft. Beide Transporte unterscheiden sich in Port, Authentifizierung und Möglichkeiten; die Befehle zum Ausführen (`Invoke-Command`, `Enter-PSSession`, `New-PSSession`) sind dieselben, nur die Parameter für das Ziel unterscheiden sich: `-ComputerName` verwendet WinRM, `-HostName` verwendet SSH.

### Voraussetzungen bei WinRM.

Nach der abgerufenen Dokumentation (`about_Remote_Requirements`) braucht Remoting in Windows Windows PowerShell 3.0 oder höher, .NET Framework 4 oder höher und WinRM 3.0 oder höher. Server ab Windows Server 2012 sind standardmäßig für Remoting aktiviert; wurden die Einstellungen geändert, stellt `Enable-PSRemoting` den Standard wieder her. Auf Windows-Clients ist es standardmäßig aus; dort richtet man es mit `Enable-PSRemoting` in einer Konsole mit erhöhten Rechten ein. Dabei werden der WinRM-Dienst konfiguriert, ein Listener angelegt, Firewallregeln erstellt und Sitzungskonfigurationen (Endpunkte) registriert. Auf Clients mit öffentlichem Netzwerkprofil verweigert der Befehl standardmäßig die Arbeit, weil das ein Sicherheitsrisiko ist; der Parameter `-SkipNetworkProfileCheck` erzwingt die Einrichtung mit einer Firewallregel, die nur das lokale Subnetz zulässt.

Wer verbinden darf, regelt der *Endpunkt*, die sogenannte Sitzungskonfiguration. Die Dokumentation hält fest: Zum Erstellen von Sitzungen und Ausführen von Befehlen braucht man ein Konto, das Mitglied der Gruppe `Administratoren` oder `Remote Management Users` auf dem Zielrechner ist, und die Sicherheitsbeschreibungen der Standardkonfigurationen `Microsoft.PowerShell` und `Microsoft.PowerShell32` lassen nur Administratoren zu. Man kann neue Endpunkte mit eigenen Berechtigungen anlegen (Lerneinheit 7.8) und wählt sie mit `-ConfigurationName`. Eine weitere Besonderheit nennt die Dokumentation: Auch eine Verbindung zum *eigenen* Rechner (*Loopback*) braucht Administratorrechte, und man muss PowerShell dafür ausdrücklich „als Administrator“ starten, selbst wenn man Mitglied der Administratorgruppe ist.

> **Ergänzung aus der Nachrecherche**
>
> **Richtlinien gelten auf beiden Seiten (laut about_Remote_Requirements).** Bei einer Remoteverbindung arbeiten zwei PowerShell-Instanzen. Vor und während des Verbindungsaufbaus gelten die Richtlinien des lokalen Rechners, während der Nutzung der Verbindung die des entfernten Rechners. Eine Execution Policy (Modul 8.1) auf dem Ziel wirkt also auf Skriptdateien dort, nicht auf Befehle, die man als Skriptblock sendet. Außerdem hält die Dokumentation fest, dass einige Cmdlets einen Parameter `-ComputerName` haben, aber *nicht* PowerShell-Remoting nutzen, sondern RPC oder WMI (zum Beispiel `Get-WinEvent`, Lerneinheit 6.4); für sie gelten diese Voraussetzungen nicht.

### WinRM oder SSH?

Die Wahl ergibt sich aus der Umgebung. Zwischen Windows-Rechnern einer Domäne ist WinRM der Normalfall: Anmeldung per Kerberos ohne Kennworteingabe, Endpunkte mit JEA, reife Werkzeuge. SSH ist die Wahl für Linux und macOS und für gemischte Umgebungen; es nutzt Schlüsselpaare, ist in der Linux-Welt ohnehin vorhanden und braucht keine Domäne. Die abgerufene SSH-Dokumentation nennt die wichtigsten Unterschiede: PowerShell wird als *SSH-Subsystem* eingerichtet (in der `sshd_config` eine Zeile der Form `Subsystem powershell ... -sshs`), und SSH-Remoting unterstützt derzeit weder Endpunktkonfigurationen noch JEA. Wer JEA braucht, bleibt also bei WinRM. Außerdem fehlen bei SSH die Möglichkeiten, Sitzungen zu trennen und wiederzuverbinden, die WinRM bietet (im Test fehlten die entsprechenden Cmdlets unter Linux).

```powershell
# Lokale Voraussetzungen und Ziel pruefen (Windows, WinRM)
Test-WSMan -ComputerName srv01                       # antwortet der WinRM-Dienst?
Test-NetConnection -ComputerName srv01 -CommonTCPPort WINRM   # Port offen?

# Remoting auf einem Zielrechner aktivieren (Administratorkonsole)
Enable-PSRemoting -Force

# Einmal-Verbindung testen: interaktive Sitzung
Enter-PSSession -ComputerName srv01
hostname
Exit-PSSession

# SSH: Verbindung zu einem Linux-Rechner (PowerShell 7)
Enter-PSSession -HostName linux01 -UserName admin -KeyFilePath ~/.ssh/id_ed25519
```

Die ersten beiden Zeilen prüfen vom Client aus, ob der WinRM-Dienst auf dem Ziel antwortet und der Port erreichbar ist (`Test-NetConnection` mit `-CommonTCPPort WINRM` ist in der abgerufenen Dokumentation beschrieben). `Enable-PSRemoting -Force` richtet das Ziel ein und unterdrückt Rückfragen. `Enter-PSSession` öffnet eine *interaktive* Sitzung, in der der Prompt den Rechnernamen anzeigt und jeder Befehl auf dem Ziel läuft, bis `Exit-PSSession` sie beendet; sie ist für Fehlersuche gedacht, nicht für Skripte. Die letzte Zeile zeigt die SSH-Variante mit Benutzername und privatem Schlüssel. Auch diese Beispiele sind auf Syntax geprüft, aber nicht ausgeführt (siehe Hinweis zu Beginn).

### Vertrauensgrenzen.

Remoting ist mächtig und deshalb ein Angriffsweg. Wer sich verbinden darf, kann auf dem Ziel mit den Rechten seines Kontos alles tun, was dieses Konto darf. Daraus folgen Regeln: Nur Administratoren und dafür vorgesehene Konten bekommen Zugang. Verbindungen laufen verschlüsselt: WinRM verschlüsselt den Inhalt der PowerShell-Verbindungen auch über HTTP mit Kerberos, bei NTLM-Anmeldung ist HTTPS sicherer (allgemeines Fachwissen, nicht abgerufen). Außerhalb einer Domäne (Arbeitsgruppe) muss der Client dem Ziel ausdrücklich vertrauen (`TrustedHosts`), was die Identitätsprüfung schwächt; HTTPS mit Zertifikat ist die bessere Wahl. Und jeder Rechner, auf dem man sich anmeldet, *bekommt* unter Umständen Anmeldedaten (Lerneinheit 7.4), weshalb man sich nicht mit hochprivilegierten Konten auf wenig vertrauenswürdigen Rechnern anmeldet. Die Anwendung des Prinzips der geringsten Rechte (Modul 8.4) und JEA (Lerneinheit 7.8) begrenzen den Schaden.

> **Merke**
>
> Remoting = PSRP über einen Transport. WinRM (`-ComputerName`): Windows, Kerberos, JEA, Sitzungen trennbar. SSH (`-HostName`): plattformübergreifend, Schlüssel, kein JEA. Zugang: Administratoren oder `Remote Management Users`; Endpunkt (Sitzungskonfiguration) entscheidet. `Enable-PSRemoting`. Loopback braucht Administrator. Richtlinien: lokal beim Aufbau, entfernt bei der Nutzung. Nicht jedes Cmdlet mit `-ComputerName` nutzt Remoting.

```powershell
# FALSCH: TrustedHosts fuer alle freigeben, damit "es einfach geht"
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '*' -Force

# RICHTIG: nur bestimmte Ziele, besser HTTPS mit Zertifikat bzw. Domaene und Kerberos
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 'lab01.contoso.local' -Force
```

Im ersten Fall vertraut der Client jedem Rechner, einschließlich eines Angreifers, der sich als Zielrechner ausgibt, und schickt ihm Anmeldedaten. Im zweiten Fall ist die Ausnahme auf ein bekanntes Ziel begrenzt.

> **Praxisbeispiel**
>
> Ein Betriebsteam muss auf 40 Linux-Servern und 60 Windows-Servern dieselbe Konfigurationsprüfung durchführen. Für Windows nutzt es WinRM mit Kerberos aus der Domäne, für Linux SSH mit Schlüsseln aus einem zentralen Verwaltungsrechner; das Prüfskript ist dasselbe, nur die Verbindungsart pro Ziel unterscheidet sich.

## 7.2 Invoke-Command

`Invoke-Command` ist das Arbeitspferd des Remotings: Es führt einen Skriptblock oder ein Skript auf einem oder vielen entfernten Rechnern aus. Der Skriptblock wird als Text zum Ziel geschickt, dort in einer neuen Sitzung ausgeführt, und die Ausgabe kommt zurück. Mit `-ComputerName` (WinRM) oder `-HostName` (SSH) gibt man die Ziele an, mit `-Credential` das Konto, mit `-FilePath` ein lokales Skript, das auf dem Ziel läuft. Gibt man mehrere Ziele an, führt `Invoke-Command` sie parallel aus; die Begrenzung der gleichzeitigen Verbindungen steuert `-ThrottleLimit` (der Standard ist 32 laut allgemeinem Fachwissen, nicht abgerufen).

Das Wichtigste an `Invoke-Command` ist, was zwischen den Rechnern *nicht* passiert: Der Skriptblock läuft in einer fremden Sitzung, die nichts von Ihren lokalen Variablen, Funktionen und Modulen weiß. Wer im Skriptblock eine lokale Variable verwendet, bekommt einen leeren Wert (und im Fall von `Set-StrictMode` einen Fehler). Es gibt zwei Wege, Werte zu übergeben: den Parameter `-ArgumentList` zusammen mit einem `param`-Block im Skriptblock, und den Bereichsmodifizierer `$using:`, mit dem man lokale Variablen direkt in den Skriptblock einblendet.

Die Lokaltests zeigen, wie das aussieht. Ohne `-ComputerName` führt `Invoke-Command` den Skriptblock lokal aus; dann funktioniert `-ArgumentList`, `$using:` aber nicht: Im Test brach `Invoke-Command -ScriptBlock { $using:x }` mit „A Using variable cannot be retrieved. A Using variable can be used only with Invoke-Command, Start-Job, or InlineScript in the script workflow. When it is used with Invoke-Command, the Using variable is valid only if the script block is invoked on a remote computer.“ ab. Die Fehlermeldung ist genau die Regel: `$using:` ist nur gültig, wenn der Block auf einem entfernten Rechner (oder in einem Job) läuft.

```powershell
$schwelle = 80
$ziele = 'srv01', 'srv02'

# Weg 1: ArgumentList mit param-Block
Invoke-Command -ComputerName $ziele -ArgumentList $schwelle -ScriptBlock {
    param($prozent)
    Get-PSDrive -PSProvider FileSystem |
        Where-Object { $_.Used -and ($_.Used / ($_.Used + $_.Free) * 100) -gt $prozent } |
        Select-Object Name, @{ n = 'Voll%'; e = { [math]::Round($_.Used / ($_.Used + $_.Free) * 100) } }
}

# Weg 2: $using:
Invoke-Command -ComputerName $ziele -ScriptBlock {
    Get-PSDrive -PSProvider FileSystem |
        Where-Object { $_.Used -and ($_.Used / ($_.Used + $_.Free) * 100) -gt $using:schwelle }
}

# Lokales Skript auf dem Ziel ausfuehren, mit Parametern
Invoke-Command -ComputerName $ziele -FilePath .\Pruefe-Server.ps1 -ArgumentList 'Vollstaendig'
```

Das erste Beispiel übergibt die Schwelle über `-ArgumentList`; im Skriptblock nimmt der `param`-Block sie entgegen, genau wie bei einer Funktion. Das zweite blendet dieselbe Variable mit `$using:schwelle` ein, was kürzer ist, aber nur in einem entfernten Block oder Job funktioniert. Das dritte führt ein lokales Skript auf allen Zielen aus: Die Datei wird gelesen und *als Text* gesendet, sodass sie auf dem Ziel nicht vorhanden sein muss; dafür sieht das Ziel auch keine Dateien, die das Skript neben sich erwartet. Alle drei Aufrufe sind auf Syntax geprüft. Der lokale Aufruf mit `-ArgumentList` (`Invoke-Command -ScriptBlock { param($a) $a * 2 } -ArgumentList 5`) lieferte im Test 10.

### Ergebnisobjekte: Was zurückkommt.

Das zweite große Thema sind die Rückgabewerte. Objekte können nicht als lebende .NET-Objekte über das Netzwerk geschickt werden; sie werden *serialisiert* (in XML umgewandelt) und auf der anderen Seite *deserialisiert*, also als Nachbildung wiederhergestellt. Die Nachbildung hat die Eigenschaftswerte, aber *keine Methoden*, und ihr Typ heißt `Deserialized.`*Ursprungstyp*. Der Test zeigt es: Ein `Process`-Objekt, das aus einem Hintergrundauftrag zurückkam (derselbe Mechanismus), hatte den Typ `System.Management.Automation.PSObject` mit dem Typnamen `Deserialized.System.Diagnostics.Process`. Die Methode `Kill` war lokal vorhanden und im Ergebnis nicht. Die Eigenschaft `Id` blieb erhalten. Einfache Typen wie Zahlen, Zeichenketten, `DateTime` und Hashtabellen kommen unverändert zurück; im Test war ein serialisiertes und deserialisiertes `DateTime` wieder ein `DateTime`.

Aus dieser Eigenschaft folgt eine Regel für die Praxis: *Die Arbeit gehört auf das Ziel, auf dem sie sinnvoll ist, nicht auf den Client.* Wer ein entferntes `Process`-Objekt zurückholt und dann lokal `.Kill()` aufruft, bekommt einen Fehler oder, schlimmer, beendet einen gleichnamigen lokalen Prozess. Richtig ist, den Prozess *im Skriptblock* zu beenden (`Stop-Process` dort) und nur das Ergebnis zurückzugeben. Außerdem liefert `Invoke-Command` an jedes Ergebnisobjekt die Eigenschaft `PSComputerName`, die den Herkunftsrechner nennt, und diese Eigenschaft braucht man, um Ergebnisse vieler Rechner zuzuordnen.

```powershell
# Objekt serialisieren und zurueckholen (derselbe Mechanismus wie beim Remoting)
$p = Get-Process -Id $PID
$xml = [System.Management.Automation.PSSerializer]::Serialize($p, 1)
$zurueck = [System.Management.Automation.PSSerializer]::Deserialize($xml)

$p.GetType().Name                        # Process
$zurueck.GetType().Name                  # PSObject
[bool]($zurueck | Get-Member Kill)       # False: Methoden gehen verloren
$zurueck.Id -eq $p.Id                    # True: Eigenschaftswerte bleiben

# Besser: Arbeit auf dem Ziel erledigen, nur das Ergebnis zurueckgeben
$ergebnis = Invoke-Command -ComputerName srv01 -ScriptBlock {
    $dienste = Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }
    foreach ($d in $dienste) {
        [pscustomobject]@{ Name = $d.Name; Status = [string]$d.Status }
    }
}
$ergebnis | Select-Object PSComputerName, Name, Status
```

Der erste Block führt die Serialisierung von Hand vor: `PSSerializer.Serialize` wandelt das Objekt in XML um (der Parameter 1 ist die Tiefe), `Deserialize` stellt es wieder her. Die Zeilen mit den Ergebnissen als Kommentar entsprechen den Testläufen. Der zweite Block ist das empfohlene Muster: Der Skriptblock fragt Dienste auf dem Ziel ab und baut *einfache* Ergebnisobjekte (`pscustomobject` mit Zeichenketten und Zahlen); solche Objekte überstehen die Reise unbeschadet. Das Feld `Status` wird bewusst in eine Zeichenkette umgewandelt, weil ein Enum-Wert sonst als Deserialized-Objekt ankäme. Die Auswertung am Client greift auf `PSComputerName` zu.

### Fehler bei der Remote-Ausführung.

Fehler im Skriptblock kommen als Fehlerdatensätze zurück, nicht als abbrechende Ausnahmen. Im Test lieferte ein Fehler in einem Hintergrundauftrag (derselbe Mechanismus) einen Datensatz vom Typ `RemotingErrorRecord`. Mit `-ErrorAction Stop` auf `Invoke-Command` wird daraus ein abfangbarer Fehler (Modul 5). Wichtig ist außerdem: Kann eine Verbindung gar nicht aufgebaut werden, erzeugt `Invoke-Command` für dieses Ziel einen Fehler (mit der Kennung `PSSessionStateBroken` oder ähnlich, nicht getestet), verarbeitet aber die anderen Ziele weiter. Man muss die Fehler gezielt einsammeln, zum Beispiel mit `-ErrorVariable`, sonst gehen sie in der Ausgabe unter.

> **Ergänzung aus der Nachrecherche**
>
> **Bei Ausführung auf vielen Zielen.** `Invoke-Command -AsJob` startet die Ausführung als Hintergrundauftrag (Lerneinheit 7.5), sodass die Konsole frei bleibt; `-HideComputerName` blendet `PSComputerName` aus, was man selten will. Die Sitzungsoptionen (Zeitlimits, Kodierung, Proxy) setzt man mit `New-PSSessionOption` und übergibt sie mit `-SessionOption`. Dieses Cmdlet ist in PowerShell 7 plattformübergreifend vorhanden (im Test unter Linux ja).

> **Merke**
>
> `Invoke-Command`: Skriptblock oder Skript auf Zielen ausführen, mehrere Ziele parallel (`-ThrottleLimit`). Der Block kennt lokale Variablen nicht: `-ArgumentList` mit `param` oder `$using:` (nur entfernt oder im Job). Ergebnisse sind serialisiert: Eigenschaften ja, Methoden nein, Typ `Deserialized.*`. Arbeit auf dem Ziel erledigen, einfache Objekte zurückgeben. `PSComputerName` ordnet zu. Fehler gezielt einsammeln.

```powershell
# FALSCH: lokale Variable im Block, Methode auf Ergebnis
$name = 'Spooler'
$d = Invoke-Command -ComputerName srv01 -ScriptBlock { Get-Service $name }
$d.Stop()

# RICHTIG: Wert uebergeben, Aktion auf dem Ziel, Ergebnis pruefen
$name = 'Spooler'
Invoke-Command -ComputerName srv01 -ScriptBlock {
    Stop-Service -Name $using:name -ErrorAction Stop
    (Get-Service -Name $using:name).Status.ToString()
}
```

Im ersten Fall ist `$name` im Block leer, sodass `Get-Service` alle Dienste liefert oder fehlschlägt, und `$d.Stop()` existiert für das deserialisierte Objekt nicht. Im zweiten Fall kommt der Name mit `$using:` an, der Dienst wird auf dem Ziel beendet, und zurück kommt nur der Status als Zeichenkette.

> **Praxisbeispiel**
>
> Ein Skript fragt auf 80 Servern den freien Plattenplatz ab. Ein Kollege hatte es zuerst so geschrieben, dass es auf jedem Ziel `Get-PSDrive` ausführte und *alle* Laufwerksobjekte zurückholte, um sie lokal zu filtern; das dauerte lange und lieferte unbrauchbare Deserialized-Objekte. Nach dem Umbau filtert der Skriptblock auf dem Ziel und gibt nur Name, Größe und Füllgrad zurück.

## 7.3 PSSessions

Jeder `Invoke-Command`-Aufruf mit `-ComputerName` baut eine Sitzung auf, führt den Block aus und baut die Sitzung wieder ab. Das ist einfach, aber teuer: Der Verbindungsaufbau kostet bei jedem Aufruf Zeit, und vor allem geht der *Zustand* verloren. Variablen, die ein erster Aufruf setzt, sind im zweiten nicht mehr da; geladene Module muss man erneut laden. Eine *PSSession* ist eine dauerhafte Sitzung, die man einmal aufbaut, mehrfach benutzt und gezielt wieder schließt. Man nutzt sie für mehrstufige Aufgaben: Zuerst ein Modul laden, dann Daten holen, dann verarbeiten, dann aufräumen, alles im selben Zustand.

Die Cmdlets dazu: `New-PSSession` baut eine Sitzung auf und gibt ein Sitzungsobjekt zurück; `Get-PSSession` listet bestehende Sitzungen; `Invoke-Command -Session` und `Enter-PSSession -Session` nutzen eine Sitzung; `Remove-PSSession` schließt sie. Das Sitzungsobjekt hat Eigenschaften wie `Id`, `Name`, `ComputerName`, `State` (`Opened`, `Closed`, `Broken`, `Disconnected`) und `Availability`. Beim WinRM-Transport gibt es außerdem `Disconnect-PSSession`, `Connect-PSSession` und `Receive-PSSession`: Man kann eine Sitzung trennen, der Auftrag läuft auf dem Ziel weiter, und später (auch von einem anderen Rechner) wieder anschließen. Im Test unter Linux waren diese drei Cmdlets nicht vorhanden; sie sind WinRM-spezifisch.

```powershell
$ziele = 'srv01', 'srv02'
$sitzungen = New-PSSession -ComputerName $ziele -ErrorAction SilentlyContinue -ErrorVariable verbindungsfehler

try {
    # Schritt 1: Zustand aufbauen (bleibt in der Sitzung erhalten)
    Invoke-Command -Session $sitzungen -ScriptBlock {
        $script:startzeit = Get-Date
        $script:zaehler = 0
    }

    # Schritt 2: Zustand wiederverwenden
    $ergebnis = Invoke-Command -Session $sitzungen -ScriptBlock {
        $script:zaehler++
        [pscustomobject]@{
            Rechner  = $env:COMPUTERNAME
            Zaehler  = $script:zaehler
            Laufzeit = ((Get-Date) - $script:startzeit).TotalSeconds
        }
    }
    $ergebnis | Select-Object PSComputerName, Rechner, Zaehler, Laufzeit

    # Dateien uebertragen: ueber die Sitzung statt ueber Netzfreigaben
    Copy-Item -Path .\konfig.json -Destination 'C:\Temp\' -ToSession $sitzungen[0]
    Copy-Item -Path 'C:\Temp\ergebnis.csv' -Destination .\ -FromSession $sitzungen[0]
}
finally {
    $sitzungen | Remove-PSSession
}
if ($verbindungsfehler) { Write-Warning "Keine Verbindung: $($verbindungsfehler.TargetObject -join ', ')" }
```

Das Beispiel baut für zwei Ziele je eine Sitzung auf; `-ErrorAction SilentlyContinue` mit `-ErrorVariable` sammelt Verbindungsfehler ein, ohne dass der Lauf abbricht (die erreichbaren Ziele erhalten trotzdem eine Sitzung). Im ersten `Invoke-Command` wird in jeder Sitzung ein Zustand angelegt (Startzeit, Zähler); `$script:` sorgt dafür, dass die Variablen über den Aufruf hinaus in der Sitzung bleiben. Der zweite Aufruf liest und verändert diesen Zustand: Wäre es eine neue Sitzung, wäre der Zähler dort leer. `Copy-Item` mit `-ToSession` und `-FromSession` überträgt Dateien über die bestehende Verbindung, ohne dass eine Netzfreigabe nötig ist; das funktioniert in beide Richtungen. Der `finally`-Block schließt die Sitzungen in jedem Fall. Das Beispiel ist auf Syntax geprüft, aber nicht gegen echte Ziele ausgeführt.

### Lebenszyklus sauber verwalten.

Sitzungen kosten Ressourcen auf dem Ziel (einen Prozess und Speicher) und sind begrenzt: Ein Ziel akzeptiert nur eine bestimmte Zahl gleichzeitiger Sitzungen je Benutzer, und vergessene Sitzungen blockieren Plätze. Daraus folgen drei Regeln. Erstens: *Jede Sitzung wird geschlossen*, im `finally`-Block, damit auch ein Fehler sie nicht zurücklässt. Zweitens: Man prüft vor dem Benutzen den Zustand (`State -eq 'Opened'`) und baut eine kaputte Sitzung (`Broken`) neu auf, etwa nach einem Neustart des Ziels oder einem Netzwerkausfall. Drittens: Sitzungen für lange Aufgaben sind anfällig für Verbindungsabbrüche; dafür gibt es Optionen mit `New-PSSessionOption` (zum Beispiel Leerlaufzeit und Wiederholungsversuche bei Netzwerkproblemen), oder man startet die Arbeit als Auftrag, der auf dem Ziel überlebt (Lerneinheit 7.5).

```powershell
function Get-GesundeSitzung {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $ComputerName)

    $s = Get-PSSession -ComputerName $ComputerName -ErrorAction SilentlyContinue |
        Where-Object State -eq 'Opened' | Select-Object -First 1
    if (-not $s) {
        # kaputte oder geschlossene Sitzungen zu diesem Ziel aufraeumen
        Get-PSSession -ComputerName $ComputerName -ErrorAction SilentlyContinue |
            Remove-PSSession -ErrorAction SilentlyContinue
        $s = New-PSSession -ComputerName $ComputerName -ErrorAction Stop
    }
    $s
}
```

Die Funktion sucht für das Ziel zuerst eine offene Sitzung und verwendet sie. Gibt es keine, räumt sie vorhandene (kaputte oder geschlossene) Sitzungen zu diesem Ziel ab und baut eine neue auf; `-ErrorAction Stop` sorgt dafür, dass ein echter Verbindungsfehler als Ausnahme beim Aufrufer ankommt. Dieses Muster macht Skripte robust gegen Verbindungsabbrüche und verhindert, dass sich Sitzungen ansammeln. Auch hier gilt: auf Syntax geprüft, unter Windows zu testen.

> **Ergänzung aus der Nachrecherche**
>
> **Implizites Remoting.** Mit `Import-PSSession -Session $s -Module ActiveDirectory` bindet man Befehle eines entfernten Moduls lokal ein: Man ruft sie wie lokale Cmdlets auf, ausgeführt werden sie auf dem Ziel. Das ist nützlich, wenn ein Modul nur auf einem Server installiert ist; die Ergebnisse sind wieder serialisiert (Lerneinheit 7.2). Die Dokumentation zu diesem Cmdlet wurde nicht abgerufen; prüfen Sie die Hilfe (`Get-Help Import-PSSession -Full`).

> **Merke**
>
> PSSession = dauerhafte Sitzung mit Zustand: `New-PSSession`, `Invoke-Command -Session`, `Remove-PSSession`. Mehrstufige Aufgaben, Zustand bleibt (`$script:`), Dateien mit `Copy-Item -ToSession/-FromSession`. Immer im `finally` schließen, Zustand prüfen, kaputte Sitzungen neu aufbauen. Trennen und Wiederverbinden nur bei WinRM.

```powershell
# FALSCH: Sitzung aufgebaut, bei Fehler nie geschlossen
$s = New-PSSession -ComputerName srv01
Invoke-Command -Session $s -ScriptBlock { throw 'bumm' }
Remove-PSSession $s     # wird bei Fehler mit ErrorAction Stop nie erreicht

# RICHTIG: finally schliesst in jedem Fall
$s = New-PSSession -ComputerName srv01
try   { Invoke-Command -Session $s -ScriptBlock { throw 'bumm' } -ErrorAction Stop }
catch { Write-Warning "Fehler auf srv01: $($_.Exception.Message)" }
finally { Remove-PSSession $s }
```

Im ersten Fall bleibt die Sitzung auf dem Ziel offen, wenn der Fehler das Skript beendet. Nach einigen Läufen sind alle Plätze belegt, und neue Verbindungen scheitern. Im zweiten Fall räumt `finally` immer auf.

> **Praxisbeispiel**
>
> Ein Installationsskript muss auf jedem Server mehrere Schritte nacheinander ausführen: Dienst stoppen, Dateien kopieren, Konfiguration ändern, Dienst starten, Test. Mit einer Sitzung je Server geschieht das ohne wiederholten Verbindungsaufbau, die Dateien gehen über `Copy-Item -ToSession`, und der `finally`-Block schließt die Sitzungen auch dann, wenn ein Schritt scheitert.

## 7.4 Authentication und Delegation

Bevor ein entfernter Rechner einen Befehl ausführt, muss er wissen, wer fragt. Dieses Verfahren heißt *Authentifizierung*. Welches Verfahren zum Einsatz kommt, hängt von Transport und Umgebung ab, und es entscheidet, ob Anmeldedaten übertragen werden, ob der Client das Ziel prüfen kann und ob das Ziel mit den Rechten des Anmeldenden weiter auf andere Rechner zugreifen darf. Ein sicherer Umgang mit Remoting setzt voraus, diese Unterschiede zu kennen.

### Kerberos.

In einer Active-Directory-Domäne ist *Kerberos* das Standardverfahren für WinRM, wenn man das Ziel mit seinem Rechnernamen anspricht. Der Client fordert beim Domänencontroller ein Ticket für den Zieldienst an und legt es dem Ziel vor; das Kennwort wird dabei nie übertragen, und beide Seiten weisen sich gegenseitig aus (die *gegenseitige Authentifizierung* schützt vor einem untergeschobenen Ziel). Kerberos braucht Namen (nicht IP-Adressen), gleiche Uhrzeit auf allen Rechnern (die Toleranz ist klein) und die Domäne. Bei Zugriff über die IP-Adresse fällt Windows auf NTLM zurück oder verweigert die Verbindung.

### NTLM.

Das ältere Verfahren *NTLM* (Challenge-Response) funktioniert auch ohne Domäne, etwa zwischen Rechnern einer Arbeitsgruppe oder bei Anmeldung per IP-Adresse. Es prüft das Ziel nicht, ist anfälliger für Angriffe (zum Beispiel das Weiterleiten von Anmeldedaten) und wird deshalb zurückgedrängt. Bei Verwendung von NTLM über WinRM muss der Client dem Ziel ausdrücklich vertrauen (`TrustedHosts`, Lerneinheit 7.1), oder man nutzt HTTPS mit einem Zertifikat, damit das Ziel überprüfbar ist. Diese Aussagen beruhen auf allgemeinem Fachwissen und wurden nicht abgerufen.

### SSH-Schlüssel.

Bei SSH-Remoting weist man sich mit einem *Schlüsselpaar* aus: Der private Schlüssel bleibt auf dem Client, der öffentliche liegt auf dem Ziel (in der Datei `authorized_keys`). Das ist kennwortfrei, verlangt aber Sorgfalt: Der private Schlüssel gehört mit einer Passphrase und strengen Dateirechten geschützt, und der öffentliche wird nur für bestimmte Konten hinterlegt. Alternativ kann man Kennwortanmeldung erlauben, was für Automation ungeeignet ist.

### CredSSP und das Double-Hop-Problem.

Der wichtigste Stolperstein heißt *Double Hop* oder *Second Hop*. Die abgerufene Dokumentation beschreibt ihn so: Sie sind an Server A angemeldet und starten eine Remotesitzung zu Server B. Ein Befehl, den Sie auf B ausführen, greift auf eine Ressource auf Server C zu, zum Beispiel eine Netzfreigabe. Der Zugriff wird verweigert, weil die Anmeldedaten, mit denen die Sitzung zu B erstellt wurde, nicht von B an C weitergegeben werden. Das ist kein Fehler, sondern Absicht: B darf nicht ohne Weiteres in Ihrem Namen auf andere Rechner zugreifen, sonst könnte ein kompromittierter B alles tun, was Sie dürfen.

Es gibt mehrere Lösungen, die die Dokumentation in dieser Reihenfolge der Empfehlung nennt: *CredSSP* (ausgewogen zwischen Komfort und Sicherheit), *ressourcenbasierte Kerberos-Delegierung* (höhere Sicherheit, einfachere Konfiguration), *Kerberos-eingeschränkte Delegierung* (hohe Sicherheit, aber nur mit Domänenadministrator einzurichten), *uneingeschränkte Kerberos-Delegierung* (nicht empfohlen), *JEA* (kann die beste Sicherheit bieten, braucht aber aufwendigere Konfiguration), eine *Sitzungskonfiguration mit RunAs-Konto* und die *Übergabe von Anmeldedaten im Skriptblock* (am einfachsten, aber man muss Anmeldedaten bereithalten). Ihre Vor- und Nachteile hängen von der Umgebung ab, und die Dokumentation fasst sie so zusammen.

> **Ergänzung aus der Nachrecherche**
>
> **Die Verfahren im Einzelnen (laut „Making the second hop in PowerShell Remoting“).** *CredSSP* legt die Anmeldedaten auf dem Zwischenrechner (B) ab; wird B kompromittiert, hat der Angreifer die Daten des Benutzers. Es ist deshalb auf Client und Server standardmäßig deaktiviert und sollte nur in sehr vertrauenswürdigen Umgebungen aktiviert werden; es funktioniert nicht mit der Gruppe „Protected Users“. *Ressourcenbasierte Kerberos-Delegierung* (seit Windows Server 2012) konfiguriert man auf dem Zielrechner C: Er legt fest, von welchen Rechnern er delegierte Anmeldedaten akzeptiert (Attribut `msDS-AllowedToActOnBehalfOfOtherIdentity`, gesetzt mit `Set-ADComputer -PrincipalsAllowedToDelegateToAccount`). Es speichert keine Anmeldedaten, funktioniert über Domänen- und Gesamtstrukturgrenzen und braucht keinen Domänenadministrator; die Dokumentation vermerkt aber bei beiden Kerberos-Varianten (eingeschränkt und ressourcenbasiert), dass sie den zweiten Hop *für WinRM* nicht unterstützen. Konten, bei denen „Konto ist vertraulich und kann nicht delegiert werden“ gesetzt ist, lassen sich nicht delegieren. Ein negativer Kerberos-Cache von 15 Minuten kann eine frisch gesetzte Delegierung verzögern (`klist purge`). Bei der Methode „Anmeldedaten im Skriptblock“ reicht man das `PSCredential` mit `$using:cred` an einen verschachtelten `Invoke-Command` weiter.

```powershell
# Double Hop erkennen (Windows): von A ueber B nach C
$cred = Get-Credential
Invoke-Command -ComputerName srvB -Credential $cred -ScriptBlock {
    Test-Path "\\srvC\Freigabe"          # False oder Zugriff verweigert: zweiter Hop scheitert
}

# Loesung "Anmeldedaten im Skriptblock" (einfach, aber Anmeldedaten bereithalten)
Invoke-Command -ComputerName srvB -Credential $cred -ScriptBlock {
    Invoke-Command -ComputerName srvC -Credential $using:cred -ScriptBlock { hostname }
}

# Besser oft: Aufgabe direkt auf C ausfuehren, B nicht als Zwischenstation nutzen
Invoke-Command -ComputerName srvC -Credential $cred -ScriptBlock { Get-ChildItem 'D:\Freigabe' }
```

Im ersten Block scheitert der Zugriff auf die Freigabe von C, weil die Sitzung auf B nur lokale Rechte kennt. Der zweite Block ist die in der Dokumentation gezeigte Lösung: Die Anmeldedaten werden mit `$using:cred` in die Sitzung auf B gereicht und dort für die Verbindung zu C verwendet; das setzt voraus, dass das `PSCredential` auf B verfügbar ist, was die Anmeldedaten dort kurzzeitig angreifbar macht. Der dritte Block ist oft die einfachste und sicherste Lösung: Man vermeidet den zweiten Hop, indem man die Aufgabe auf dem Rechner ausführt, auf dem die Ressource liegt. Alle Beispiele sind auf Syntax geprüft, nicht ausgeführt.

### Umgang mit Anmeldedaten.

Aus alledem folgen praktische Regeln. Anmeldedaten kommen aus einem Tresor (Modul 8.5), nicht aus dem Skript. Man nutzt für Automation Konten mit geringsten Rechten (Modul 8.4) und, wo möglich, gruppenverwaltete Dienstkonten. Man wählt die schwächste Delegierung, die die Aufgabe erlaubt, und *keine* uneingeschränkte Delegierung. Hochprivilegierte Konten (Domänenadministratoren) melden sich nicht an Zwischenrechnern an, weil diese die Anmeldedaten im Speicher halten können.

> **Merke**
>
> Kerberos (Domäne, Namen, kein Kennwort im Netz, gegenseitige Prüfung), NTLM (Fallback, schwächer, `TrustedHosts` oder HTTPS), SSH-Schlüssel (kennwortfrei, Schlüssel schützen). Double Hop: Anmeldedaten gelten nicht für den zweiten Rechner. Lösungen nach Empfehlung: CredSSP, ressourcenbasierte Kerberos-Delegierung, eingeschränkte Delegierung, JEA, RunAs-Konfiguration, Anmeldedaten im Block. Am besten den zweiten Hop vermeiden.

```powershell
# FALSCH: CredSSP pauschal einschalten, "damit der zweite Hop geht"
Enable-WSManCredSSP -Role Client -DelegateComputer '*' -Force

# RICHTIG: zweiten Hop vermeiden oder gezielt loesen (ressourcenbasierte Delegierung)
Set-ADComputer -Identity $serverC -PrincipalsAllowedToDelegateToAccount $serverB
```

Im ersten Fall dürfen Anmeldedaten an jeden Rechner delegiert werden, und ein einziger kompromittierter Server genügt, um sie abzugreifen. Im zweiten Fall darf nur B an C delegieren, und die Anmeldedaten werden nicht gespeichert (die Variablen `$serverB` und `$serverC` sind AD-Computerobjekte, siehe Dokumentation).

> **Praxisbeispiel**
>
> Ein Wartungsskript soll von einem Verwaltungsserver aus auf einem Anwendungsserver eine Datei von einer Dateifreigabe kopieren. Es scheitert mit „Zugriff verweigert“. Die Ursache ist der zweite Hop. Statt CredSSP zu aktivieren, ändert das Team den Ablauf: Das Skript läuft direkt auf dem Dateiserver und schiebt die Datei zum Anwendungsserver; der zweite Hop entfällt.

## 7.5 Jobs und Background Execution

Eine Konsole ist blockiert, solange ein Befehl läuft. Für langlaufende Aufgaben (Kopieren großer Datenmengen, Installationen, Abfragen vieler Rechner) ist das unpraktisch. Ein *Job* ist ein Befehl, der im Hintergrund läuft, während die Konsole weiter benutzbar bleibt; man kann ihn später abfragen, auf ihn warten und sein Ergebnis einsammeln. Die Job-Cmdlets sind `Start-Job` (Auftrag starten), `Get-Job` (Aufträge auflisten), `Wait-Job` (warten), `Receive-Job` (Ergebnis abholen), `Stop-Job` (abbrechen) und `Remove-Job` (aufräumen). Außerdem können `Invoke-Command -AsJob` und `ForEach-Object -Parallel -AsJob` Aufträge liefern.

Ein klassischer `Start-Job` startet für jeden Auftrag einen *neuen PowerShell-Prozess*. Das hat Folgen: Der Auftrag ist von der Sitzung des Aufrufers getrennt, kennt deren Variablen und Funktionen nicht (Werte übergibt man mit `-ArgumentList` oder `$using:`), und der Start kostet Zeit (im Test spürbar). Dafür ist die Isolierung gut: Ein Absturz im Auftrag schadet dem Aufrufer nicht. Eine leichtere Variante sind *Thread-Jobs* (`Start-ThreadJob`, im Test unter PowerShell 7 vorhanden): Sie laufen als Thread im selben Prozess, starten sehr schnell, teilen sich aber den Prozess.

### Zustände und Ergebnisse.

Ein Auftrag durchläuft Zustände: `Running`, `Completed`, `Failed`, `Stopped`, `Blocked`. Die wichtigste Lehre aus den Tests: *Die Fehlerbehandlung unterscheidet sich vom normalen Code.* Wirft ein Auftrag eine terminierende Ausnahme (`throw`), endet er im Zustand `Failed`; der Grund steht in `$job.ChildJobs[0].JobStateInfo.Reason`, und `Receive-Job` wirft den Fehler beim Abholen erneut (im Test: „catch: bumm“). Schreibt ein Auftrag dagegen nur einen nicht terminierenden Fehler (`Write-Error`), bleibt er `Completed`, und der Fehler erscheint als `ErrorRecord` (im Test vom Typ `RemotingErrorRecord`) in der Ausgabe von `Receive-Job`. Das heißt: „Completed“ bedeutet *nicht* „fehlerfrei“. Ein Job-Ergebnis ist außerdem ein serialisiertes Objekt (Lerneinheit 7.2).

```powershell
# Mehrere Auftraege starten, mit Zeitlimit warten, alles auswerten
$jobs = 1..4 | ForEach-Object {
    Start-Job -Name "Auftrag$_" -ArgumentList $_ -ScriptBlock {
        param($n)
        if ($n -eq 3) { throw "Auftrag $n kaputt" }
        if ($n -eq 4) { Start-Sleep -Seconds 60 }
        "Ergebnis $n"
    }
}

$null = Wait-Job -Job $jobs -Timeout 10        # wartet bis alle fertig sind oder 10 s vorbei

$bericht = foreach ($j in $jobs) {
    $text = $null; $fehler = $null
    switch ($j.State) {
        'Completed' { $text = Receive-Job -Job $j }
        'Failed'    { $fehler = $j.ChildJobs[0].JobStateInfo.Reason.Message }
        default     { $fehler = "Zeitlimit (Status $($j.State))"; Stop-Job -Job $j }
    }
    [pscustomobject]@{ Name = $j.Name; Status = $j.State; Ergebnis = $text; Fehler = $fehler }
}
$jobs | Remove-Job -Force
$bericht | Format-Table -AutoSize
```

Das Skript startet vier Aufträge mit `-ArgumentList` und einem `param`-Block. Auftrag 3 wirft einen Fehler, Auftrag 4 schläft eine Minute. `Wait-Job -Timeout 10` wartet höchstens zehn Sekunden auf *alle* Aufträge; der Test zeigt, dass der Aufruf nach Ablauf des Zeitlimits einfach zurückkehrt, und die langsamen Aufträge laufen weiter (im Test: Auftrag in Zustand `Running`, `Wait-Job` lieferte nichts zurück). Die Auswertung geht jeden Auftrag durch: Fertige werden mit `Receive-Job` abgeholt, fehlgeschlagene liefern ihren Grund, und die übrigen (zu lange laufenden) werden mit `Stop-Job` abgebrochen, was ihren Zustand auf `Stopped` setzt. Der Lauf ergab: Aufträge 1 und 2 `Completed` mit ihren Ergebnissen, Auftrag 3 `Failed` mit „Auftrag 3 kaputt“, Auftrag 4 `Stopped` mit „Zeitlimit (Status Running)“. Zuletzt entfernt `Remove-Job -Force` alle Aufträge; ohne diesen Schritt sammeln sich Aufträge und ihre Ergebnisse im Speicher der Sitzung an.

```powershell
# Aufgabe auf vielen Zielen im Hintergrund starten (WinRM)
$job = Invoke-Command -ComputerName $ziele -AsJob -ThrottleLimit 16 -ScriptBlock {
    Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 1 HotFixID, InstalledOn
}
# Ein Hauptauftrag mit einem Kindauftrag je Ziel
$job.ChildJobs | Select-Object Id, Location, State

# Spaeter abholen: Ergebnisse, Fehler und Laufzeiten je Ziel
$null = Wait-Job $job -Timeout 120
foreach ($kind in $job.ChildJobs) {
    $laufzeit = if ($kind.PSEndTime) { ($kind.PSEndTime - $kind.PSBeginTime).TotalSeconds } else { $null }
    [pscustomobject]@{ Ziel = $kind.Location; Status = $kind.State; Sekunden = $laufzeit }
}
Receive-Job $job -Keep | Select-Object PSComputerName, HotFixID, InstalledOn
Remove-Job $job -Force
```

Mit `Invoke-Command -AsJob` entsteht ein Hauptauftrag, dessen `ChildJobs` je Ziel einen Kindauftrag enthalten; `Location` nennt das Ziel, `PSBeginTime` und `PSEndTime` liefern die Laufzeit. So sammelt man Laufzeiten und Fehler je Rechner ein. `Receive-Job -Keep` holt die Ergebnisse, ohne sie aus dem Auftrag zu löschen, was nützlich ist, um sie mehrfach auszuwerten. `Get-HotFix` gibt es nur unter Windows; das Beispiel ist auf Syntax geprüft. Auch der Hinweis auf `PSBeginTime` stammt aus dem Auftragsmodell, im Test wurde die Eigenschaft nicht ausgewertet.

> **Ergänzung aus der Nachrecherche**
>
> **Unterschiede der Auftragsarten (aus den Tests).** Ein `Start-Job` startet einen neuen Prozess (Start dauert spürbar), ein `Start-ThreadJob` einen Thread im selben Prozess. Im Test dauerten vier Thread-Jobs mit je einer Sekunde Schlaf und `-ThrottleLimit 2` zusammen 2,2 Sekunden, was zeigt, dass die Begrenzung der gleichzeitig laufenden Aufträge wirkt. Beide Arten liefern ihre Ergebnisse mit `Receive-Job`. Thread-Jobs sind schneller und sparsamer, teilen sich aber Prozessumgebung und Abstürze mit dem Aufrufer; Prozess-Jobs sind stärker getrennt.

> **Merke**
>
> Jobs: `Start-Job`, `Get-Job`, `Wait-Job -Timeout`, `Receive-Job`, `Stop-Job`, `Remove-Job`. Werte per `-ArgumentList` oder `$using:`. `Failed` bei `throw`, aber `Completed` bei `Write-Error`: Fehler in der Ausgabe auswerten. `Wait-Job -Timeout` bricht nichts ab; langsame Aufträge selbst mit `Stop-Job` beenden. Immer aufräumen. `Start-ThreadJob` ist leichter, `Invoke-Command -AsJob` liefert Kindaufträge je Ziel.

```powershell
# FALSCH: Auftrag starten, Ergebnis "irgendwann" lesen, nie aufraeumen
$j = Start-Job { Get-ChildItem C:\Windows -Recurse -ErrorAction SilentlyContinue | Measure-Object }
Receive-Job $j       # liefert nichts, Auftrag laeuft noch; Ergebnisse gehen verloren

# RICHTIG: warten (mit Zeitlimit), Zustand pruefen, abholen, aufraeumen
$j = Start-Job { Get-ChildItem C:\Windows -Recurse -ErrorAction SilentlyContinue | Measure-Object }
if (Wait-Job $j -Timeout 60) { Receive-Job $j } else { Stop-Job $j; Write-Warning 'Zeitlimit' }
Remove-Job $j -Force
```

Im ersten Fall liefert `Receive-Job` nur, was bis zu diesem Moment angefallen ist, also meist nichts, und der Auftrag bleibt im Speicher. Im zweiten Fall wartet das Skript mit Zeitlimit, behandelt den Zeitablauf und räumt auf.

> **Praxisbeispiel**
>
> Ein nächtlicher Lauf startet auf 120 Servern eine Inventur als `Invoke-Command -AsJob`. Am Morgen zeigt der Bericht je Server Status und Laufzeit: 117 fertig, 2 fehlgeschlagen (Zugriff verweigert), 1 durch Zeitlimit abgebrochen. Ohne Kindaufträge hätte man nur gewusst, dass „etwas“ nicht funktioniert hat.

## 7.6 ForEach-Object -Parallel

Ab PowerShell 7 kann `ForEach-Object` mit dem Parameter `-Parallel` die Elemente der Pipeline gleichzeitig verarbeiten. Anders als `Start-Job` startet es keine neuen Prozesse, sondern verwendet *Runspaces* (leichtgewichtige, voneinander getrennte PowerShell-Umgebungen) innerhalb desselben Prozesses; der Start ist dadurch viel schneller. Das lohnt sich bei Aufgaben, die hauptsächlich *warten*: Netzwerkabfragen, Webaufrufe, Dateizugriffe auf langsame Medien. Bei reiner Rechenarbeit auf einem einzigen Prozessorkern bringt es dagegen wenig, und bei sehr kurzen Aufgaben überwiegt der Aufwand der Parallelisierung.

Die Dokumentation (abgerufen) nennt die wichtigsten Parameter: `-ThrottleLimit` begrenzt die Zahl gleichzeitig laufender Skriptblöcke (Standard 5), `-TimeoutSeconds` legt fest, wie lange die gesamte Verarbeitung dauern darf, bevor sie abgebrochen wird, und `-AsJob` liefert einen Auftrag statt direkter Ausgabe. Die Messungen im Test machen die Wirkung greifbar.

| **Aufruf (je 0,5 s Arbeit pro Element)** | **Gemessene Zeit** |
|:-----------------------------------------|:-------------------|
| 6 Elemente seriell mit `ForEach-Object`         | 3,2 s              |
| 6 Elemente, `-Parallel -ThrottleLimit 6`                    | 0,6 s              |
| 6 Elemente, `-Parallel -ThrottleLimit 2`                    | 1,6 s              |
| 10 Elemente, `-Parallel` (Standard)        | 1,1 s              |

Die Zeiten passen zur Theorie: Seriell addieren sich sechs Mal 0,5 s. Mit sechs gleichzeitigen Ausführungen dauert alles eine halbe Sekunde plus Startaufwand. Mit zwei gleichzeitigen Ausführungen sind es drei Durchgänge (1,5 s). Beim Standardwert 5 laufen zehn Elemente in zwei Durchgängen (1,0 s). Die Begrenzung ist ein Schutz für Ressourcen: Ohne sie würden tausend gleichzeitige Netzwerkverbindungen den Rechner oder das Ziel überlasten.

### Variablen und Funktionen.

Jeder Skriptblock läuft in einem eigenen Runspace und sieht *keine* Variablen des Aufrufers. Wer im Block eine Variable des Aufrufers ohne `$using:` schreibt, bekommt `$null` und *keinen Fehler*: Im Test lieferte `1..2 | ForEach-Object -Parallel { $_ * $faktor }` die Werte 0 und 0, weil `$faktor` im Block leer war; mit `$using:faktor` kamen 10, 20 und 30 heraus (mit dem Faktor 10 für `1..3`). Dieser stille Fehler ist besonders tückisch. Auch Funktionen des Aufrufers fehlen: Im Test scheiterte der Aufruf einer lokalen Funktion mit „The term ’Mein-Tool’ is not recognized“. Man übergibt die Funktionsdefinition als Text und definiert sie im Block neu (`\${function:Mein-Tool} = \$using:def`); im Test funktionierte das. Innerhalb von `$using:` sind nur einfache Variablenzugriffe erlaubt: `$using:Aktion.ToString()` wurde schon beim Parsen mit „Expression is not allowed in a Using expression.“ abgelehnt; man berechnet den Wert vorher in eine eigene Variable.

### Fehler und Zeitlimit.

Hier steckt die zweite wichtige Einschränkung. Die gemeinsamen Parameter `-ErrorAction`, `-WarningAction`, `-InformationAction` und `-PipelineVariable` werden mit `-Parallel` *nicht unterstützt*; im Test brach der Aufruf mit „The following common parameters are not currently supported in the Parallel parameter set: ErrorAction, WarningAction, InformationAction, PipelineVariable“ ab. Man behandelt Fehler daher *im Skriptblock*: Mit `$ErrorActionPreference = 'Stop'` und `try/catch` im Block fängt man sie ab (im Test lieferte der Block „gefangen: ObjectNotFound“). Ein nicht abgefangener Fehler in einem Element lässt die anderen weiterlaufen; er landet im Fehlerstrom, und `-ErrorVariable` sammelte ihn (im Test: 1 Fehler, „Fehler bei 3“, Ergebnisse 1, 2, 4). Das Zeitlimit `-TimeoutSeconds 3` beendete im Test nach 3,0 Sekunden die Verarbeitung; ein Element war fertig (Ergebnis 1), für die anderen erschien „The pipeline has been stopped.“.

### Reihenfolge und Threadsicherheit.

Die Reihenfolge der Ausgabe entspricht der Reihenfolge der Fertigstellung, nicht der Eingabe: Im Test kamen die Werte 1 bis 8 in der Folge 2, 6, 4, 1, 5, 7, 3, 8. Wer die Eingabereihenfolge braucht, gibt das Eingabeelement mit zurück und sortiert danach. Alle Blöcke laufen *gleichzeitig im selben Prozess*. Greifen sie auf eine gemeinsame Sammlung zu, muss diese *threadsicher* sein: Eine normale `List` oder `Hashtable` kann bei gleichzeitigen Schreibzugriffen beschädigt werden; im Test ergab eine `List` zwar 2000 Einträge, aber das ist Zufall und kein Beweis (solche Fehler treten nur unregelmäßig auf). Sicher sind die Typen aus `System.Collections.Concurrent` (im Test: `ConcurrentBag` mit 50 Einträgen bei 8 Threads). Am besten gibt man Ergebnisse einfach aus der Pipeline zurück und sammelt sie am Ende, statt gemeinsame Sammlungen zu füllen.

```powershell
$urls = 'https://example.org/a', 'https://example.org/b', 'https://example.org/c'
$zeitlimit = 10

$ergebnis = $urls | ForEach-Object -ThrottleLimit 5 -Parallel {
    $url = $_
    $ErrorActionPreference = 'Stop'
    $watch = [System.Diagnostics.Stopwatch]::StartNew()
    try {
        $antwort = Invoke-WebRequest -Uri $url -TimeoutSec $using:zeitlimit -Method Head
        [pscustomobject]@{ Url = $url; Status = [int]$antwort.StatusCode; Sekunden = [math]::Round($watch.Elapsed.TotalSeconds, 2); Fehler = $null }
    }
    catch {
        [pscustomobject]@{ Url = $url; Status = $null; Sekunden = [math]::Round($watch.Elapsed.TotalSeconds, 2); Fehler = $_.Exception.Message }
    }
}
$ergebnis | Sort-Object Url | Format-Table -AutoSize
```

Das Beispiel prüft mehrere Webadressen parallel. Jedes Element läuft in einem Block mit eigenem Runspace; die Variable `$zeitlimit` wird mit `$using:` übergeben. `$ErrorActionPreference = 'Stop'` im Block macht aus Fehlern abfangbare Ausnahmen, und `try/catch` liefert in beiden Fällen ein einheitliches Ergebnisobjekt mit Status, Dauer und Fehlertext. Dadurch kann ein einzelner Fehler weder die anderen Elemente noch die Auswertung stören. Das Ergebnis wird am Ende aus der Pipeline in eine Variable gesammelt und nach der Adresse sortiert, da die Reihenfolge der Fertigstellung beliebig ist. Der Webzugriff wurde im Test nicht ausgeführt, weil der Arbeitsbereich keinen Zugriff auf beliebige Webseiten hat; das Muster entspricht dem getesteten Aufbau (Ergebnisobjekt aus Block, Fehler im Block).

> **Ergänzung aus der Nachrecherche**
>
> **Aus der Dokumentation (ForEach-Object).** Der Parameter `-Parallel` gibt es ab PowerShell 7.0. Die Dokumentation weist darauf hin, dass der parallele Skriptblock keinen direkten Zugriff auf Variablen des Aufrufers hat und mit `$using:` arbeitet, dass gemeinsam genutzte Objekte threadsicher sein müssen und dass `-ThrottleLimit` den Standard 5 hat. Die Parallelisierung hat einen Overhead; bei kurzen Aufgaben ist die serielle Verarbeitung oft schneller. Der Parameter `-UseNewRunspace` (hier nicht getestet) erzeugt für jedes Element einen neuen Runspace statt Runspaces wiederzuverwenden.

> **Merke**
>
> `ForEach-Object -Parallel`: Runspaces im selben Prozess, schnell, für wartende Aufgaben. `-ThrottleLimit` (Standard 5) schützt Ressourcen, `-TimeoutSeconds` begrenzt die Gesamtzeit, `-AsJob` liefert einen Auftrag. Variablen nur mit `$using:` (sonst stilles `$null`), Funktionen als Text übergeben. `-ErrorAction` & Co. nicht unterstützt: Fehler im Block behandeln. Ausgabe ungeordnet. Gemeinsame Sammlungen nur threadsicher.

```powershell
# FALSCH: Variable ohne using, gemeinsame List, Fehler nicht behandelt
$faktor = 10
$sammlung = [System.Collections.Generic.List[int]]::new()
1..100 | ForEach-Object -Parallel { $sammlung.Add($_ * $faktor) }   # $sammlung und $faktor sind $null

# RICHTIG: using, Ergebnis aus der Pipeline, Fehler im Block
$faktor = 10
$ergebnis = 1..100 | ForEach-Object -ThrottleLimit 8 -Parallel { $_ * $using:faktor }
$ergebnis.Count
```

Im ersten Fall sind beide Variablen im Block leer; der Aufruf von `.Add` auf `$null` erzeugt Fehler (jeweils für jedes Element), und `$faktor` wäre im Block ebenfalls leer. Im zweiten Fall übergibt `$using:` den Faktor, und die Ergebnisse kommen aus der Pipeline zurück, ohne dass gemeinsame Sammlungen nötig sind.

> **Praxisbeispiel**
>
> Ein Skript prüft täglich die Erreichbarkeit von 2000 Webadressen. Seriell dauert es über eine Stunde, weil die meisten Zeit auf Antworten warten. Mit `-Parallel` und einem `ThrottleLimit` von 20 sind es wenige Minuten; das Limit schützt den eigenen Proxy vor Überlastung, und das Zeitlimit je Anfrage verhindert, dass einzelne hängende Adressen die Gesamtlaufzeit bestimmen.

## 7.7 Fan-out Administration

*Fan-out* (Auffächern) heißt, dieselbe Aufgabe auf vielen Zielen auszuführen und die Ergebnisse wieder einzusammeln. Mit zehn Servern ist das trivial, mit tausend entscheidet die Planung: Wie viele Verbindungen gleichzeitig? Was geschieht mit Zielen, die nicht antworten? Wie lange darf eine Aufgabe dauern? Soll ein fehlgeschlagener Versuch wiederholt werden, und wie oft? Bei *ändernden* Aufgaben kommt hinzu: Was passiert, wenn mittendrin etwas schiefgeht? Ein Skript, das alle tausend Server gleichzeitig neu startet, ist kein Werkzeug, sondern ein Vorfall.

Die Bausteine dafür haben Sie bereits: Parallelität mit Begrenzung (Lerneinheit 7.6 oder `Invoke-Command -ThrottleLimit`), Zeitlimits, Fehlerbehandlung je Ziel und Ergebnisobjekte mit `PSComputerName`. Neu sind drei Strategien. *Batching*: Man teilt die Ziele in Gruppen (Batches) und arbeitet eine Gruppe nach der anderen ab, damit nicht alles gleichzeitig läuft und man nach jeder Gruppe prüfen und notfalls abbrechen kann. *Timeouts*: Jede Aktion bekommt ein Zeitlimit, damit ein hängendes Ziel nicht alles aufhält. *Retry*: Bei vorübergehenden Fehlern (Netzwerk, Last) versucht man es nach einer Wartezeit erneut, mit steigenden Abständen, aber nur bei Fehlern, die sich wiederholen lassen.

```powershell
function Invoke-FanOut {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string[]] $Ziele,
        [Parameter(Mandatory)] [scriptblock] $Aktion,
        [ValidateRange(1, 64)] [int] $ThrottleLimit = 8,
        [ValidateRange(1, 20)] [int] $MaxVersuche = 3,
        [ValidateRange(0, 60)] [int] $WartenSekunden = 1,
        [ValidateRange(1, 3600)] [int] $TimeoutSekunden = 30
    )
    $aktionText = $Aktion.ToString()
    $Ziele | ForEach-Object -ThrottleLimit $ThrottleLimit `
        -TimeoutSeconds ($TimeoutSekunden * $MaxVersuche) -Parallel {
        $ziel   = $_
        $aktion = [scriptblock]::Create($using:aktionText)
        $max    = $using:MaxVersuche
        $pause  = $using:WartenSekunden
        $versuch = 0; $fehler = $null; $wert = $null; $ok = $false
        while (-not $ok -and $versuch -lt $max) {
            $versuch++
            try   { $wert = & $aktion $ziel; $ok = $true; $fehler = $null }
            catch {
                $fehler = $_.Exception.Message
                if ($versuch -lt $max) { Start-Sleep -Seconds ($pause * $versuch) }
            }
        }
        [pscustomobject]@{ Ziel = $ziel; Erfolg = $ok; Versuche = $versuch; Wert = $wert; Fehler = $fehler }
    }
}

# Testaktion: ein Ziel ist tot, ein Ziel ist unzuverlaessig
$aktion = {
    param($z)
    if ($z -eq 'srv-tot') { throw 'Verbindung abgelehnt' }
    if ($z -eq 'srv-flaky' -and (Get-Random -Maximum 100) -lt 60) { throw 'Zeitueberschreitung' }
    Start-Sleep -Milliseconds 200
    "ok von $z"
}
$ziele = 'srv01', 'srv02', 'srv-tot', 'srv-flaky', 'srv03'
$r = Invoke-FanOut -Ziele $ziele -Aktion $aktion -ThrottleLimit 3 -MaxVersuche 3 -WartenSekunden 0
$r | Sort-Object Ziel | Format-Table Ziel, Erfolg, Versuche, Fehler, Wert -AutoSize
```

Die Funktion packt die Muster zusammen. Der Parameter `Aktion` ist ein Skriptblock, der pro Ziel aufgerufen wird; so lässt sich dasselbe Gerüst für Abfragen mit `Invoke-Command`, Dateikopien oder Webaufrufe verwenden. Weil Skriptblöcke nicht sicher zwischen Runspaces übergeben werden, wandelt die Funktion ihn vor dem Parallelblock in Text um (`$aktionText`) und baut ihn im Block neu auf; dass dieser Umweg nötig ist, zeigte der Test, in dem `$using:Aktion.ToString()` direkt abgelehnt wurde. Im Block läuft eine Wiederholungsschleife: Bei einem Fehler wartet sie `pause * versuch` Sekunden (linear steigende Pause) und versucht es erneut, höchstens `MaxVersuche`-mal. Das Ergebnis ist je Ziel ein Objekt mit Erfolg, Zahl der Versuche, Wert und letztem Fehler. Das Gesamtzeitlimit ist `TimeoutSekunden * MaxVersuche`, damit auch alle Versuche Platz haben. Der Testlauf (mit `ThrottleLimit 3`) ergab: `srv01`, `srv02` und `srv03` erfolgreich im ersten Versuch, `srv-tot` nach drei Versuchen mit „Verbindung abgelehnt“ fehlgeschlagen, `srv-flaky` (zufällig) nach mehreren Versuchen erfolgreich; im gezeigten Lauf waren es drei, das hängt vom Zufall ab.

### Batching.

Bei ändernden Aufgaben sollte man nicht alle Ziele auf einmal anfassen. Mit `[System.Linq.Enumerable]::Chunk()` (ab .NET 6, also ab PowerShell 7.2) teilt man eine Liste in gleich große Stücke. Zwischen den Gruppen kann man warten, den Erfolg auswerten und bei zu vielen Fehlern abbrechen (ein *Abbruchkriterium*, in der Praxis oft „wenn mehr als zehn Prozent einer Gruppe scheitern, stopp“).

```powershell
$alle = 1..23 | ForEach-Object { "srv$_" }
$batchGroesse = 10
$maxFehlerAnteil = 0.2
$gesamt = [System.Collections.Generic.List[object]]::new()

$nummer = 0
foreach ($batch in [System.Linq.Enumerable]::Chunk([string[]]$alle, $batchGroesse)) {
    $nummer++
    Write-Verbose "Batch $nummer mit $($batch.Count) Zielen ($($batch[0]) bis $($batch[-1]))" -Verbose
    $r = Invoke-FanOut -Ziele $batch -Aktion $aktion -ThrottleLimit 5 -MaxVersuche 2 -WartenSekunden 1
    $gesamt.AddRange(@($r))

    $fehler = @($r | Where-Object { -not $_.Erfolg }).Count
    if ($fehler / $batch.Count -gt $maxFehlerAnteil) {
        Write-Warning "Abbruch nach Batch ${nummer}: $fehler von $($batch.Count) fehlgeschlagen"
        break
    }
}
"{0} Ziele bearbeitet, {1} fehlgeschlagen" -f $gesamt.Count, @($gesamt | Where-Object { -not $_.Erfolg }).Count
```

Die Schleife teilt 23 Ziele in Gruppen zu je 10 (der Test lieferte drei Gruppen mit 10, 10 und 3 Zielen) und ruft für jede Gruppe das Fan-out auf. Nach jeder Gruppe wird der Anteil fehlgeschlagener Ziele berechnet; überschreitet er die Grenze von 20 Prozent, bricht die Schleife ab, bevor weitere Server betroffen sind. Im Testlauf mit vier ausgefallenen Zielen in der ersten Gruppe (4 von 10, also 40 Prozent) erschien „Abbruch nach Batch 1: 4 von 10 fehlgeschlagen“, und es wurden nur 10 der 23 Ziele angefasst. Ein Abbruchkriterium ist bei Änderungen die wichtigste Sicherung.

> **Ergänzung aus der Nachrecherche**
>
> **Eine Falle bei der Gruppierung mit Group-Object.** Wer Elemente mit einem Zähler gruppieren will (`Group-Object { [math]::Floor($i++ / 10) }`), stolpert über Gültigkeitsbereiche: Der Skriptblock von `Group-Object` läuft pro Element in einem eigenen Bereich, sodass `$i` jedes Mal von vorn beginnt. Im Test landeten alle 23 Elemente in einer einzigen Gruppe („Gruppe 0: 23“). `Chunk()` umgeht das Problem.

### Weitere Planungsfragen.

Ein Fan-out ist auch eine Frage der Verantwortung. Man begrenzt die Parallelität nach dem Ziel, nicht nur nach dem eigenen Rechner: 500 gleichzeitige Verbindungen zu einem zentralen Server (Datenbank, Proxy, Domänencontroller) sind ein Angriff auf diesen Server. Man trennt *lesende* von *ändernden* Aufgaben und testet die Änderung zuerst an einem Pilotziel. Man sorgt dafür, dass die Aktion *idempotent* ist (Modul 5): Läuft sie ein zweites Mal, entsteht kein Schaden, was Wiederholungen und Neustarts nach Abbrüchen erst sicher macht. Und man protokolliert pro Ziel Zeit, Ergebnis, Versuche und Fehler, damit sich nachträglich klären lässt, was wo geschah.

> **Merke**
>
> Fan-out: viele Ziele kontrolliert bearbeiten. Parallelität begrenzen (auch nach dem Ziel), Zeitlimit je Aktion, Retry nur bei wiederholbaren Fehlern mit steigender Pause, Batching mit `Chunk()`, Abbruchkriterium bei Änderungen, idempotente Aktionen, Protokoll je Ziel. Lesen vor Ändern, Pilotziel zuerst.

```powershell
# FALSCH: alle Server gleichzeitig, kein Limit, kein Abbruch, Fehler verschwinden
$server | ForEach-Object -Parallel { Restart-Computer -ComputerName $_ -Force } -ThrottleLimit 1000

# RICHTIG: kleine Gruppen, Pruefung nach jeder Gruppe, Abbruchkriterium, Protokoll
foreach ($batch in [System.Linq.Enumerable]::Chunk([string[]]$server, 5)) {
    $r = Invoke-FanOut -Ziele $batch -Aktion $neustartAktion -ThrottleLimit 5 -MaxVersuche 1
    if (@($r | Where-Object { -not $_.Erfolg }).Count -gt 0) { Write-Warning 'Fehler, Stopp'; break }
}
```

Im ersten Fall starten tausend Server gleichzeitig neu, und niemand sieht, welche scheiterten. Im zweiten Fall bleibt ein Fehler auf eine kleine Gruppe begrenzt.

> **Praxisbeispiel**
>
> Ein Patch-Skript soll 600 Server neu starten. Es arbeitet in Gruppen zu 20, wartet nach jeder Gruppe, bis alle Server wieder erreichbar sind (Portprüfung, Lerneinheit 6.7), und stoppt, wenn mehr als zwei Server einer Gruppe nicht zurückkommen. In einer Nacht fällt so ein fehlerhaftes Update nach der ersten Gruppe auf, statt 600 Server lahmzulegen.

## 7.8 JEA Grundlagen

Remoting hat ein Problem: Wer sich verbinden darf, hat meist Administratorrechte auf dem Ziel. Soll das Helpdesk nur einen Dienst neu starten dürfen, müsste man ihm dafür volle Administratorrechte geben, und jeder Fehler oder Missbrauch trifft dann den ganzen Server. *Just Enough Administration* (JEA) löst das Problem mit einem eigenen Endpunkt: Benutzer verbinden sich mit ihren normalen Konten, aber der Endpunkt erlaubt nur genau die Befehle, die ihre Rolle braucht, und führt sie unter einem anderen, privilegierten Konto aus. Die abgerufene Microsoft-Dokumentation beschreibt den Nutzen so: JEA verringert die Zahl der Administratoren, begrenzt, welche Cmdlets, Funktionen und externen Befehle Benutzer ausführen dürfen, und liefert über Transkripte und Protokolle, welche Befehle ein Benutzer tatsächlich ausgeführt hat.

### Bausteine.

JEA besteht aus zwei Dateitypen und einem Endpunkt. Eine *Rollenfähigkeitsdatei* (`.psrc`, Role Capability) beschreibt, *was* eine Rolle darf: sichtbare Cmdlets (mit Einschränkung der Parameter und erlaubter Werte), Funktionen, externe Programme und Provider. Eine *Sitzungskonfigurationsdatei* (`.pssc`, Session Configuration) beschreibt den *Endpunkt*: den Sitzungstyp, unter welchem Konto die Befehle laufen (zum Beispiel ein *virtuelles Konto*, das für die Sitzung temporär entsteht und nach ihrem Ende verschwindet), wo Transkripte landen und welche Gruppen welche Rollen bekommen (`RoleDefinitions`). Mit `Register-PSSessionConfiguration` wird der Endpunkt auf dem Server angelegt, und Benutzer verbinden sich mit `-ConfigurationName`. Die Sitzung läuft im `NoLanguage`-Modus (Modul 8.3): Man kann nur die freigegebenen Befehle aufrufen, aber keine eigene Skriptlogik schreiben.

Die Cmdlets zum Erzeugen der Dateien, `New-PSRoleCapabilityFile` und `New-PSSessionConfigurationFile`, gibt es auch unter Linux; mit ihnen ließen sich die Dateien erzeugen. Es fehlten dort dagegen `Register-PSSessionConfiguration` und `Test-PSSessionConfigurationFile`, denn JEA selbst ist ein Windows-Endpunkt auf WinRM-Basis. Die erzeugten Dateien sind gewöhnliche PowerShell-Datendateien, die man bearbeiten kann.

```powershell
# 1. Rollenfaehigkeit: Helpdesk darf Dienste anzeigen und nur den Spooler neu starten
$ordner = 'C:\JEA\Helpdesk'
New-Item -ItemType Directory -Path $ordner -Force | Out-Null
New-PSRoleCapabilityFile -Path "$ordner\SpoolerNeustart.psrc" `
    -VisibleCmdlets @(
        'Get-Service',
        @{ Name = 'Restart-Service'; Parameters = @{ Name = 'Name'; ValidateSet = 'Spooler' } }
    )

# 2. Sitzungskonfiguration: virtuelles Konto, Transkripte, Rollenzuordnung
New-PSSessionConfigurationFile -Path "$ordner\Helpdesk.pssc" `
    -SessionType RestrictedRemoteServer `
    -RunAsVirtualAccount `
    -TranscriptDirectory 'C:\JEA\Transkripte' `
    -RoleDefinitions @{ 'CONTOSO\Helpdesk' = @{ RoleCapabilities = 'SpoolerNeustart' } }

# 3. Endpunkt registrieren (Administratorkonsole, Windows)
Register-PSSessionConfiguration -Name 'HelpdeskJEA' -Path "$ordner\Helpdesk.pssc" -Force

# 4. Als Helpdesk-Mitarbeiter verbinden und testen
Enter-PSSession -ComputerName srv01 -ConfigurationName 'HelpdeskJEA'
Get-Command                       # zeigt nur die erlaubten Befehle
Restart-Service -Name Spooler     # erlaubt
Restart-Service -Name WinRM       # abgelehnt: Wert nicht im ValidateSet
```

Schritt 1 legt die Fähigkeit fest: `Get-Service` uneingeschränkt, `Restart-Service` nur mit dem Parameter `Name`, und der darf nur den Wert `Spooler` haben (`ValidateSet`). Die Erzeugung der Datei mit Parameterbeschränkung wurde im Test ausgeführt; die erzeugte Datei enthielt den erwarteten Eintrag `VisibleCmdlets` mit `Name = 'Restart-Service'` und `Parameters` mit `ValidateSet = 'Spooler'`. Schritt 2 legt den Endpunkt fest: Der Sitzungstyp `RestrictedRemoteServer` bietet einen minimalen Befehlssatz, `-RunAsVirtualAccount` lässt die Befehle unter einem temporären lokalen Administratorkonto laufen (ein virtuelles Konto, ohne Kennwortverwaltung), `-TranscriptDirectory` schreibt von jeder Sitzung ein Transkript, und `-RoleDefinitions` ordnet der Gruppe `CONTOSO\Helpdesk` die Rolle `SpoolerNeustart` zu. Auch diese Datei wurde im Test erzeugt und enthielt `SessionType = 'RestrictedRemoteServer'`, `RunAsVirtualAccount = $true` und die Rollenzuordnung. Schritt 3 und 4 sind nur unter Windows ausführbar und deshalb auf Syntax geprüft. In der Sitzung zeigt `Get-Command` nur die erlaubten Befehle, und der Versuch, einen anderen Dienst neu zu starten, wird vom `ValidateSet` abgelehnt.

### Einordnung der Grenzen.

JEA ersetzt nicht alles. Es ist ein *Delegationswerkzeug*, kein Allheilmittel: Eine zu großzügige Rollenfähigkeit hebelt den Schutz aus. Wer zum Beispiel `Start-Process` oder `Invoke-Expression` freigibt, erlaubt beliebige Befehle. Auch erlaubte Cmdlets können missbraucht werden, wenn sie Skriptblöcke oder Dateien entgegennehmen. Die Rollen sollten deshalb *möglichst eng* sein, mit eingeschränkten Parametern, und regelmäßig geprüft werden. Wichtig ist außerdem die Reichweite: JEA funktioniert über WinRM und nicht über SSH-Remoting (siehe Lerneinheit 7.1).

> **Ergänzung aus der Nachrecherche**
>
> **JEA und das Double-Hop-Problem.** Die abgerufene Dokumentation zum zweiten Hop nennt JEA ausdrücklich als mögliche Lösung („can provide the best security but requires more detailed configuration“): Weil die Befehle im JEA-Endpunkt unter einem virtuellen Konto oder einem gruppenverwalteten Dienstkonto laufen, braucht man keine Weitergabe der Benutzeranmeldedaten. Als Nachteile nennt sie, dass es WMF 5.0 oder höher braucht und die Konfiguration auf jedem Zwischenserver nötig ist. Ein virtuelles Konto hat auf dem Rechner lokale Administratorrechte, aber im Netzwerk nur die Identität des Rechnerkontos; für den Zugriff auf andere Rechner braucht man ein gruppenverwaltetes Dienstkonto (laut allgemeinem Fachwissen, nicht abgerufen).

> **Merke**
>
> JEA = delegierte Administration über einen eigenen Endpunkt: `.psrc` (was darf die Rolle), `.pssc` (Endpunkt, Konto, Transkripte, Rollenzuordnung), `Register-PSSessionConfiguration`, Verbindung mit `-ConfigurationName`. `NoLanguage`, virtuelles Konto, Transkripte. Rollen so eng wie möglich, Parameter mit `ValidateSet` einschränken, keine mächtigen Befehle freigeben. Nur WinRM, nicht SSH.

```powershell
# FALSCH: zu grosszuegige Rolle - erlaubt faktisch alles
New-PSRoleCapabilityFile -Path .\Zuviel.psrc -VisibleCmdlets 'Restart-Service', 'Start-Process', 'Invoke-Command' `
    -VisibleExternalCommands 'C:\Windows\System32\cmd.exe'

# RICHTIG: nur der benoetigte Befehl, Parameter und Werte eingeschraenkt
New-PSRoleCapabilityFile -Path .\Eng.psrc -VisibleCmdlets @(
    @{ Name = 'Restart-Service'; Parameters = @{ Name = 'Name'; ValidateSet = 'Spooler' } }
)
```

Die erste Rolle erlaubt über `cmd.exe` und `Start-Process` jeden Befehl und macht die Einschränkung wertlos, und der Dienstname ist nicht begrenzt. Die zweite erlaubt genau eine Aktion für genau einen Dienst.

> **Praxisbeispiel**
>
> Das Helpdesk soll auf Druckservern den Spooler neu starten und Dienststatus abfragen dürfen. Das Serverteam richtet einen JEA-Endpunkt mit der oben gezeigten Rolle ein und trägt die Helpdesk-Gruppe ein. Jeder Neustart steht im Transkript mit Benutzer, Zeit und Befehl; die Mitarbeiter haben keinen Administratorzugang zum Server, und eine Kompromittierung ihres Kontos erlaubt nur den Neustart eines Dienstes.

## 7.9 Übungsaufgaben zu Modul 07

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Aufgaben 1 bis 4 prüfen Verständnis, Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung folgen im Anschluss. Aufgaben mit echten Zielrechnern brauchen eine Laborumgebung; die Aufgaben 6 und 7 lassen sich lokal ausführen.

**Aufgabe 1.** Nennen Sie je zwei Vor- und Nachteile von WinRM-Remoting und SSH-Remoting. Wann wählen Sie welches, und warum ist JEA nur mit einem von beiden möglich?

**Aufgabe 2.** Ein Kollege holt mit `Invoke-Command` ein `Process`-Objekt von einem Server und ruft lokal `.Kill()` auf. Was geschieht, und wie lösen Sie die Aufgabe richtig?

**Aufgabe 3.** Erklären Sie das Double-Hop-Problem an einem Beispiel und nennen Sie drei Lösungen mit Vor- und Nachteilen. Welche würden Sie als Erstes prüfen?

**Aufgabe 4.** Warum ist `Completed` bei einem Hintergrundauftrag keine Garantie für Fehlerfreiheit? Wie erkennen Sie beide Fehlerarten (`throw` und `Write-Error`) und was geschieht bei `Wait-Job -Timeout`?

**Aufgabe 5.** Schreiben Sie eine Funktion `Get-ServerStatus`, die für eine Liste von Servern per `Invoke-Command` Rechnername, Startzeit und freien Speicher abfragt (einfache Objekte), nicht erreichbare Server einsammelt und am Ende einen Bericht mit `PSComputerName` und einer Liste der Fehlschläge liefert. (Syntax prüfen, ohne Zielrechner.)

**Aufgabe 6.** Schreiben Sie ein Skript mit `ForEach-Object -Parallel`, das eine Liste von Zahlen (1 bis 20) parallel verarbeitet (jede Verarbeitung braucht 0,3 s), den Faktor 3 über `$using:` erhält, Fehler für die Zahl 7 *im Block* abfängt und die Ergebnisse in der Eingabereihenfolge ausgibt. Messen Sie die Zeit für `ThrottleLimit` 1, 5 und 20.

**Aufgabe 7.** Erweitern Sie `Invoke-FanOut` aus Lerneinheit 7.7 so, dass es zusätzlich die Dauer je Ziel in Sekunden liefert und bei einem Fehler nur wiederholt, wenn die Fehlermeldung das Wort `Zeit` enthält (vorübergehender Fehler); andere Fehler brechen sofort ab.

**Aufgabe 8.** Entwerfen Sie ein JEA-Konzept für das Team „Berichte“, das auf Dateiservern Dienste „Berichtsdienst“ neu starten, Protokolle des Ordners `D:\Berichte\Logs` lesen (aber nicht ändern) und den freien Plattenplatz abfragen darf. Schreiben Sie die `.psrc`-Parameter und die `.pssc`-Parameter und begründen Sie Ihre Wahl der Einschränkungen.

## 7.10 Lösungen zu Modul 07

### Aufgabe 1.

WinRM: Vorteile sind Kerberos-Anmeldung ohne Kennwort in der Domäne, JEA-Endpunkte und trennbare Sitzungen; Nachteile sind die Beschränkung auf Windows als Ziel (für PowerShell 7 unter Linux gibt es SSH) und die Konfiguration von Listenern, `TrustedHosts` und Zertifikaten außerhalb von Domänen. SSH: Vorteile sind Plattformunabhängigkeit, Schlüsselanmeldung und keine Domäne; Nachteile sind das Fehlen von Endpunktkonfiguration und JEA, kein Trennen und Wiederverbinden von Sitzungen und die Pflege der Schlüssel. Wahl: reine Windows-Domäne, delegierte Administration oder JEA: WinRM; gemischte Umgebung oder Linux: SSH. JEA braucht Sitzungskonfigurationen (Endpunkte), die es bei SSH-Remoting laut Dokumentation nicht gibt.

### Aufgabe 2.

Das Ergebnis ist ein serialisiertes Objekt (Typ `Deserialized.System.Diagnostics.Process`); Eigenschaften wie `Id` sind vorhanden, Methoden wie `Kill` nicht (im Test bestätigt). Der Aufruf scheitert mit einem Fehler. Richtig: den Prozess auf dem Ziel beenden, zum Beispiel `Invoke-Command -ComputerName srv01 -ScriptBlock { Stop-Process -Id $using:id -ErrorAction Stop }`, und nur ein einfaches Ergebnis zurückgeben. Wer die `Id` lokal weiterverwendet, nutzt sie nicht auf dem lokalen Rechner (dort könnte sie einen ganz anderen Prozess bezeichnen).

### Aufgabe 3.

Beispiel: Auf Server A starten Sie eine Sitzung zu Server B, ein Befehl dort greift auf eine Freigabe von Server C zu und scheitert, weil die Anmeldedaten nicht von B an C weitergegeben werden. Lösungen aus der Dokumentation: (a) CredSSP: einfach, aber die Anmeldedaten liegen auf B, ein kompromittiertes B gibt sie preis; nur in sehr vertrauenswürdigen Umgebungen. (b) Ressourcenbasierte Kerberos-Delegierung: keine gespeicherten Anmeldedaten, per Cmdlet konfigurierbar, über Domänen hinweg, ohne Domänenadministrator, aber ab Windows Server 2012 und laut Dokumentation nicht für den zweiten Hop von WinRM. (c) Anmeldedaten im Skriptblock: ohne Serverkonfiguration, aber man muss Anmeldedaten bereithalten und sie gelangen auf B. (d) JEA: beste Sicherheit, aber aufwendige Konfiguration auf jedem Zwischenserver. Zuerst prüfe ich, ob sich der zweite Hop *vermeiden* lässt, indem die Aufgabe direkt auf dem Rechner mit der Ressource läuft; danach kommen die ressourcenbasierte Delegierung oder JEA in Betracht, CredSSP erst zuletzt.

### Aufgabe 4.

`Completed` heißt nur, dass der Auftrag ohne *terminierende* Ausnahme endete. Ein `Write-Error` im Auftrag erzeugt einen nicht terminierenden Fehler: Der Zustand bleibt `Completed`, und der Fehler erscheint als `ErrorRecord` (im Test `RemotingErrorRecord`) in der Ausgabe von `Receive-Job`. Ein `throw` beendet den Auftrag im Zustand `Failed`; der Grund steht in `$job.ChildJobs[0].JobStateInfo.Reason`, und `Receive-Job -ErrorAction Stop` wirft ihn beim Abholen. Wer beide Arten erkennen will, prüft den Zustand und wertet `Receive-Job -ErrorVariable` aus. `Wait-Job -Timeout` kehrt nach Ablauf des Zeitlimits einfach zurück (und liefert nichts zurück, wenn kein Auftrag fertig ist); die Aufträge laufen weiter (Zustand `Running`) und müssen mit `Stop-Job` beendet werden.

### Aufgabe 5.

```powershell
function Get-ServerStatus {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string[]] $Ziele,
          [pscredential] $Credential)

    $parameter = @{
        ComputerName  = $Ziele
        ErrorAction   = 'SilentlyContinue'
        ErrorVariable = 'verbindungsfehler'
        ScriptBlock   = {
            $os = Get-CimInstance -ClassName Win32_OperatingSystem -Property LastBootUpTime, FreePhysicalMemory
            [pscustomobject]@{
                Rechner      = $env:COMPUTERNAME
                Startzeit    = $os.LastBootUpTime.ToString('o')
                FreiMemoryMB = [math]::Round($os.FreePhysicalMemory / 1KB)
            }
        }
    }
    if ($Credential) { $parameter.Credential = $Credential }

    $ergebnisse = @(Invoke-Command @parameter)
    $fehlgeschlagen = @($verbindungsfehler | ForEach-Object {
        [pscustomobject]@{ Ziel = $_.TargetObject; Grund = $_.Exception.Message }
    })
    [pscustomobject]@{
        Ergebnisse     = $ergebnisse | Select-Object PSComputerName, Rechner, Startzeit, FreiMemoryMB
        Fehlgeschlagen = $fehlgeschlagen
    }
}
```

Die Parameter stehen in einer Hashtable und werden per Splatting übergeben, damit `-Credential` nur dann hinzukommt, wenn es angegeben wurde. Der Skriptblock baut ein *einfaches* Objekt: Die Startzeit wird als Zeichenkette zurückgegeben, der Speicher als Zahl, damit nichts deserialisiert ankommt. `-ErrorVariable` sammelt Verbindungsfehler der nicht erreichbaren Ziele, `TargetObject` nennt das Ziel (die genaue Eigenschaft kann je nach Fehlerart variieren; prüfen Sie sie in Ihrer Umgebung). Die Funktion liefert ein Bericht-Objekt mit Ergebnissen und Fehlschlägen. Nur Syntax geprüft, `Get-CimInstance` verlangt Windows.

### Aufgabe 6.

```powershell
Set-StrictMode -Version Latest
$faktor = 3
foreach ($limit in 1, 5, 20) {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $ergebnis = 1..20 | ForEach-Object -ThrottleLimit $limit -Parallel {
        $eingabe = $_
        $ErrorActionPreference = 'Stop'
        try {
            if ($eingabe -eq 7) { throw 'Sieben ist verboten' }
            Start-Sleep -Milliseconds 300
            [pscustomobject]@{ Eingabe = $eingabe; Ergebnis = $eingabe * $using:faktor; Fehler = $null }
        }
        catch {
            [pscustomobject]@{ Eingabe = $eingabe; Ergebnis = $null; Fehler = $_.Exception.Message }
        }
    } | Sort-Object Eingabe
    '{0,2} parallel: {1:n1} s, {2} Ergebnisse, {3} Fehler' -f $limit, $sw.Elapsed.TotalSeconds,
        $ergebnis.Count, @($ergebnis | Where-Object Fehler).Count
}
```

Jedes Element liefert in beiden Fällen ein Ergebnisobjekt mit der Eingabe; so lässt sich am Ende mit `Sort-Object Eingabe` die Eingabereihenfolge wiederherstellen, obwohl die Verarbeitung ungeordnet endet. Der Fehler für die Zahl 7 wird im Block abgefangen (`-ErrorAction` wäre mit `-Parallel` nicht erlaubt). Die Zeiten fallen mit steigendem Limit: Bei 1 ist es seriell (etwa 6 s für 19 Elemente mit je 0,3 s), bei 5 etwa 1,3 s und bei 20 etwa 0,7 s (0,3 s Arbeit plus Startaufwand der Runspaces). Im Testlauf waren es 5,9 s, 1,3 s und 0,7 s, jeweils mit 20 Ergebnissen und einem Fehler; die genauen Werte hängen von der Maschine ab.

### Aufgabe 7.

```powershell
function Invoke-FanOut {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string[]] $Ziele,
        [Parameter(Mandatory)] [scriptblock] $Aktion,
        [ValidateRange(1, 64)] [int] $ThrottleLimit = 8,
        [ValidateRange(1, 20)] [int] $MaxVersuche = 3,
        [ValidateRange(0, 60)] [int] $WartenSekunden = 1,
        [ValidateRange(1, 3600)] [int] $TimeoutSekunden = 30
    )
    $aktionText = $Aktion.ToString()
    $Ziele | ForEach-Object -ThrottleLimit $ThrottleLimit `
        -TimeoutSeconds ($TimeoutSekunden * $MaxVersuche) -Parallel {
        $ziel = $_
        $aktion = [scriptblock]::Create($using:aktionText)
        $max = $using:MaxVersuche; $pause = $using:WartenSekunden
        $uhr = [System.Diagnostics.Stopwatch]::StartNew()
        $versuch = 0; $fehler = $null; $wert = $null; $ok = $false
        while (-not $ok -and $versuch -lt $max) {
            $versuch++
            try { $wert = & $aktion $ziel; $ok = $true; $fehler = $null }
            catch {
                $fehler = $_.Exception.Message
                $voruebergehend = $fehler -match 'Zeit'
                if (-not $voruebergehend) { break }
                if ($versuch -lt $max) { Start-Sleep -Seconds ($pause * $versuch) }
            }
        }
        [pscustomobject]@{ Ziel = $ziel; Erfolg = $ok; Versuche = $versuch
                           Sekunden = [math]::Round($uhr.Elapsed.TotalSeconds, 2)
                           Wert = $wert; Fehler = $fehler }
    }
}
```

Die Stoppuhr beginnt je Ziel und liefert am Ende die Dauer. Bei einem Fehler prüft ein regulärer Ausdruck, ob die Meldung „Zeit“ enthält; nur dann geht es mit der Wartezeit weiter, sonst beendet `break` die Schleife sofort. Dadurch wird ein „Verbindung abgelehnt“ nicht dreimal wiederholt, wohl aber eine „Zeitueberschreitung“.

### Aufgabe 8.

Rollenfähigkeit (`.psrc`): `VisibleCmdlets` mit `Get-Service` (nur mit `Name` auf den Berichtsdienst eingeschränkt, zum Beispiel `ValidateSet = 'Berichtsdienst'`), `Restart-Service` mit `Parameters = @{ Name = 'Name'; ValidateSet = 'Berichtsdienst' }`, `Get-ChildItem` und `Get-Content` nur mit `-LiteralPath` und `ValidatePattern` auf den Ordner `D:\Berichte\Logs` (Muster: `'^D:\\Berichte\\Logs\\[^\\]+$'` oder eine kleine Funktion, die den Pfad prüft), dazu eine freigegebene Funktion `Get-FreierPlatz`, die nur den Füllgrad liefert (`VisibleFunctions`, Funktion über `FunctionDefinitions` definiert). Keine freigegebenen Befehle wie `Start-Process`, `Invoke-Expression` oder `cmd.exe`. Sitzungskonfiguration (`.pssc`): `SessionType = 'RestrictedRemoteServer'`, `RunAsVirtualAccount = $true` (die Befehle laufen unter einem temporären Konto, das Team braucht keine Administratorrechte), `TranscriptDirectory` auf einen Ordner, den das Team nicht ändern darf, `RoleDefinitions = @{ 'CONTOSO\Berichte' = @{ RoleCapabilities = 'BerichteTeam' } }`. Begründung: Jede Einschränkung begrenzt den Schaden bei einem übernommenen Konto; die Dienstnamen und der Pfad sind feste Werte statt freier Eingabe (Allowlist, Modul 8.7), das Transkript macht jede Aktion nachvollziehbar, und das virtuelle Konto vermeidet Kennwörter. Eine Funktion für den Pfad ist sicherer als ein reguläres Muster, weil sie auch `..`-Pfade prüfen kann (Modul 8.7).
