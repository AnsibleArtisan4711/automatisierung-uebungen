# Modul 09 - Monitoring, Logging und Troubleshooting

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (9.1 bis 9.8; 9.9 Übungsaufgaben, 9.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 10 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 10 h (8 Lerneinheiten à 1,25 h)      |
| Priorität                             | 2 (einzelne Lerneinheiten: 2 bzw. 3) |

> **Merke**
>
> Übersichtsblatt und Detailblatt stimmen in diesem Modul überein: 8 Lerneinheiten à 1,25 h ergeben 10 h. Nach dem Richtwert von mindestens einer Seite je Unterrichtsstunde wären 10 Seiten nötig; dieses Skript ist deutlich umfangreicher, weil Betriebsthemen viele Stolperfallen haben.

### Lernziele.

Nach diesem Modul können Sie Skripte so schreiben, dass man im Betrieb jederzeit erkennt, was sie getan haben und warum sie scheiterten. Sie schreiben strukturierte Protokolle mit Lauf-Kennung, trennen die sechs PowerShell-Ströme sauber, setzen Exit Codes verlässlich, melden Betriebsereignisse an das Ereignisprotokoll des Betriebssystems, messen Laufzeit und Mengen und stellen Messwerte für Überwachungssysteme bereit. Außerdem gehen Sie bei Fehlern methodisch vor, erkennen die häufigsten Fehlerbilder an ihren Merkmalen und dokumentieren Wiederanlauf und Eskalation in Runbooks.

| **Nr.** | **Lerneinheit**                | **Dauer** | **Prio** |
|:--------|:-------------------------------|:----------|:---------|
| 9.1     | Strukturiertes Skript-Logging  | 1,25 h    | 2        |
| 9.2     | PowerShell Streams             | 1,25 h    | 2        |
| 9.3     | Exit Codes und Status          | 1,25 h    | 2        |
| 9.4     | Eventlog-Integration           | 1,25 h    | 2        |
| 9.5     | Metriken aus Skripten          | 1,25 h    | 2        |
| 9.6     | Troubleshooting Methodik       | 1,25 h    | 2        |
| 9.7     | Typische Fehlerbilder          | 1,25 h    | 3        |
| 9.8     | Runbooks und Incident Response | 1,25 h    | 3        |
|         | **Summe**                      | **10 h**  |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde.** Alle Beispiele zu Protokollierung, Strömen, Exit Codes, Messwerten, Diagnose und Incident-Bericht wurden mit PowerShell 7.6.6 unter Linux ausgeführt; die gezeigten Ausgaben stammen aus diesen Läufen. Nicht ausführbar waren die Windows-Teile: das Schreiben in das Windows-Ereignisprotokoll (Lerneinheit 9.4), die Prüfung auf Administratorrechte unter Windows und die Fehlermeldung „Zugriff verweigert“, weil der Arbeitsbereich mit Administratorrechten (root) lief. Der Linux-Zweig des Ereignis-Beispiels (Aufruf von `logger`) lief ohne Fehler; einen Syslog-Dienst, in dem sich der Eintrag hätte nachlesen lassen, gab es im Arbeitsbereich nicht. Diese Teile sind als „nur auf Syntax geprüft“ gekennzeichnet. Die Aussagen zu den Ereignisprotokoll-Cmdlets stützen sich auf die abgerufenen Seiten `about_Logging_Windows` und `about_Logging` der Microsoft-Dokumentation.

## 9.1 Strukturiertes Skript-Logging

Ein Skript, das nachts um drei Uhr auf einem Server läuft, hat keinen Zuschauer. Wenn am Morgen jemand fragt, ob die Sicherung durchgelaufen ist, gibt es genau eine Antwort: das, was das Skript aufgeschrieben hat. Protokollieren heißt deshalb nicht, ab und zu etwas auf den Bildschirm zu schreiben, sondern dem späteren Leser, meist dem eigenen Ich in einer Notlage, genug Beweise zu hinterlassen, um den Ablauf zu rekonstruieren. In Modul 5 (Lerneinheit 5.6) haben Sie die Grundmuster kennengelernt: Zeitstempel, Log-Level, Zielorte. Dieses Modul behandelt die Betriebsseite, also was ein Protokoll enthalten muss, damit andere Menschen und Werkzeuge es auswerten können.

Der entscheidende Schritt ist der Übergang von Freitext zu *strukturierten* Einträgen. Eine Zeile wie „12:03 Kopiere Datei A.txt, 3 Sekunden, fertig“ ist für Menschen lesbar, aber ein Programm kann daraus nur mit fehleranfälligen Textmustern etwas gewinnen. Ein strukturierter Eintrag dagegen besteht aus benannten Feldern, heute meist als JSON-Objekt, eines pro Zeile (das Format heißt *JSON Lines* oder *NDJSON*). Jedes Feld hat einen festen Namen, sodass eine spätere Auswertung wie „alle Fehler von Rechner srv01 gestern“ ein einfacher Filter ist. Dieselbe Datei bleibt trotzdem mit `Get-Content` lesbar.

Welche Felder gehören hinein? Bewährt hat sich ein kleiner, stabiler Kern. Ein *Zeitstempel* in UTC im ISO-8601-Format vermeidet Verwirrung durch Zeitzonen und Sommerzeit. Das *Log-Level* (DEBUG, INFO, WARN, ERROR) erlaubt, Wichtiges von Gesprächigem zu trennen. Die *Run-ID* ist eine kurze zufällige Kennung, die beim Start erzeugt wird und in jedem Eintrag steht; sie bindet alle Zeilen eines Laufs zusammen, auch wenn mehrere Läufe gleichzeitig in dieselbe Datei schreiben. Hinzu kommen *Rechnername* und *Benutzer*, damit klar ist, wo und mit welchen Rechten etwas geschah, die *Aktion* als kurzes, stabiles Stichwort und der *Text* als Beschreibung. Schließlich ein Feld für beliebige *Zusatzdaten* (Anzahl, Dauer, Zielname), in dem sich Zahlen als Zahlen und nicht als Teil eines Satzes speichern lassen.

```powershell
function New-RunId { [guid]::NewGuid().ToString('N').Substring(0,8) }

$script:Redact = @(
    @{ Muster = '(?i)(password|passwort|token|secret|apikey)\s*[=:]\s*\S+'; Ersatz = '$1=***' },
    @{ Muster = '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}';        Ersatz = '<mail>' }
)

function Write-Log {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, Position = 0)] [string] $Meldung,
        [ValidateSet('DEBUG','INFO','WARN','ERROR')] [string] $Level = 'INFO',
        [string] $Aktion = '',
        [hashtable] $Daten = @{},
        [string] $Pfad = $script:LogPfad
    )
    foreach ($r in $script:Redact) { $Meldung = $Meldung -replace $r.Muster, $r.Ersatz }
    $eintrag = [ordered]@{
        zeit   = (Get-Date).ToUniversalTime().ToString('o')
        level  = $Level
        runId  = $script:RunId
        host   = [System.Environment]::MachineName
        user   = [System.Environment]::UserName
        aktion = $Aktion
        text   = $Meldung
        daten  = $Daten
    }
    $zeile = $eintrag | ConvertTo-Json -Compress -Depth 4
    if ($Pfad) { Add-Content -LiteralPath $Pfad -Value $zeile -Encoding utf8 } else { $zeile }
}

$script:RunId   = New-RunId
$script:LogPfad = Join-Path ([IO.Path]::GetTempPath()) "demo-$($script:RunId).jsonl"
Write-Log 'Start des Laufs' -Aktion 'Start'
Write-Log 'Verbinde mit password=Geheim123 als anna@example.org' -Level WARN -Aktion 'Verbinden'
Write-Log 'Dateien verarbeitet' -Aktion 'Kopieren' -Daten @{ anzahl = 42; dauerMs = 1830 }
Get-Content $script:LogPfad
```

Die Funktion `Write-Log` fasst die Idee in ein Werkzeug. Zuerst legt `New-RunId` die Lauf-Kennung fest, hier acht Zeichen aus einer GUID. Der Block `$script:Redact` ist eine Liste von Ersetzungsregeln; `Write-Log` wendet jede davon mit `-replace` auf den Text an, bevor irgendetwas geschrieben wird. Die erste Regel findet Wörter wie „password“ oder „token“, gefolgt von `=` oder `:` und einem Wert, und ersetzt den Wert durch drei Sterne (`$1` steht dabei für das erste Klammerpaar, also das Schlüsselwort); die zweite maskiert E-Mail-Adressen. Der Eintrag selbst ist ein geordnetes Objekt, das `ConvertTo-Json -Compress` in genau eine Zeile verwandelt; `-Depth 4` sorgt dafür, dass auch verschachtelte Zusatzdaten vollständig gespeichert werden. `Add-Content` hängt die Zeile an die Datei an, sodass frühere Einträge erhalten bleiben. Ist kein Pfad gesetzt, gibt die Funktion die Zeile stattdessen aus. Der Testlauf lieferte drei Zeilen; die zweite lautete im Kern `"text":"Verbinde mit password=*** als <mail>"`: Das Kennwort und die Adresse stehen nicht im Protokoll.

```powershell
# Auswertung: alle Warnungen und Fehler dieses Laufs
Get-Content $script:LogPfad |
    ConvertFrom-Json |
    Where-Object level -ne 'INFO' |
    Select-Object zeit, level, aktion, text
```

Die Auswertung zeigt den Gewinn: Jede Zeile wird mit `ConvertFrom-Json` wieder zu einem Objekt, auf das die gewohnten Cmdlets (`Where-Object`, `Group-Object`, `Sort-Object`) anwendbar sind. Eine Stolperfalle bemerkt man dabei sofort: `ConvertFrom-Json` wandelt ISO-Zeitstempel standardmäßig in `DateTime`-Werte um, die dann in der Kultur des Rechners angezeigt werden. Im Test erschien aus dem Zeitstempel eine Darstellung im Format „10/25/2026 08:00:00“. Wer den Text unverändert braucht, ruft `ConvertFrom-Json -DateKind String` auf; der Parameter `DateKind` ist in PowerShell 7.6 vorhanden (im Test bestätigt) und lieferte dann ein `String`-Objekt.

> **Ergänzung aus der Nachrecherche**
>
> **Datenschutz im Protokoll (keine Rechtsberatung).** Protokolle enthalten schnell personenbezogene Daten: Benutzernamen, E-Mail-Adressen, Dateinamen mit Personennamen, IP-Adressen. Nach der Datenschutz-Grundverordnung (DSGVO) gelten Grundsätze wie Datenminimierung und Speicherbegrenzung, die auch für technische Protokolle von Bedeutung sind. Praktisch heißt das: Nur das protokollieren, was für Betrieb und Fehlersuche nötig ist, Geheimnisse (Kennwörter, Token, Schlüssel) *nie* schreiben, personenbezogene Felder wo möglich kürzen oder durch Kennungen ersetzen, eine Aufbewahrungsdauer festlegen und mit dem Datenschutzverantwortlichen abstimmen. Diese Hinweise ersetzen keine rechtliche Beratung; die Maskierungsregeln im Beispiel sind eine technische Hilfe und keine Garantie, weil Muster nie alle Fälle erfassen.

### Wohin schreiben?

Für ein einzelnes Skript genügt eine Datei pro Lauf oder pro Tag. Sobald viele Rechner im Spiel sind, will man die Protokolle zentral sammeln; dann schreibt das Skript weiterhin eine lokale Datei, und ein Agent oder ein Sammelprozess überträgt die Zeilen. Genau dafür ist ein zeilenweises Format ideal: Eine neue Zeile ist ein neues Ereignis, und ein Sammler muss nur das Dateiende beobachten. Man gibt Dateien einen sprechenden Namen mit Datum, rotiert oder löscht alte Dateien nach einer festen Frist und stellt sicher, dass das ausführende Konto nur in dieses eine Verzeichnis schreiben darf. Wichtig ist außerdem, dass das Protokollieren selbst nicht zum Fehler wird: Schlägt das Schreiben fehl (Platte voll, Verzeichnis gesperrt), sollte das Skript das bemerken, aber nicht mit einer unklaren Folgemeldung abstürzen.

> **Merke**
>
> Strukturierte Protokolle bestehen aus einem Eintrag pro Zeile mit festen Feldern: Zeitstempel in UTC, Level, Run-ID, Rechner, Benutzer, Aktion, Text und Zusatzdaten. Die Run-ID verbindet alle Zeilen eines Laufs. Kennwörter, Token und unnötige personenbezogene Daten gehören nicht ins Protokoll; Maskierung geschieht *vor* dem Schreiben.

```powershell
# FALSCH: Freitext, lokale Zeit, Geheimnis im Klartext, kein Bezug zum Lauf
Add-Content log.txt "$(Get-Date) Verbinde mit Kennwort $kennwort als $user ..."

# RICHTIG: strukturiert, UTC, maskiert, mit Run-ID und Aktion
Write-Log "Verbinde als $user" -Aktion 'Verbinden' -Daten @{ ziel = $server }
```

Die falsche Variante schreibt ein Geheimnis dauerhaft auf die Platte, benutzt die Ortszeit (bei der Umstellung auf Winterzeit gibt es eine Stunde doppelt) und lässt sich nicht maschinell filtern. Die richtige Variante speichert nur, was gebraucht wird, und macht jedes Feld einzeln auswertbar.

> **Praxisbeispiel**
>
> Nach einem Ausfall fragt die Leitung, ob der nächtliche Import nur einmal oder wiederholt fehlschlug. Mit Run-IDs lautet die Antwort in einer Zeile: Alle Einträge der letzten drei Nächte nach `runId` gruppieren, je Lauf die höchste Stufe bestimmen. Ohne Struktur müsste jemand die Textdateien Zeile für Zeile lesen.

## 9.2 PowerShell Streams

PowerShell kennt nicht einen Ausgabekanal, sondern mehrere. Diese *Ströme* (englisch *Streams*) trennen Nutzdaten von Meldungen, und sie sind der Grund, warum man ein Skript in der Pipeline weiterverarbeiten kann, ohne dass Statusmeldungen die Daten verschmutzen. Es gibt sechs sichtbare Ströme plus einen für Fortschrittsanzeigen: den Erfolgsstrom (Stream 1, *Output*), den Fehlerstrom (2, *Error*), den Warnungsstrom (3, *Warning*), den ausführlichen Strom (4, *Verbose*), den Debug-Strom (5, *Debug*) und den Informationsstrom (6, *Information*). Aus Modul 5 kennen Sie die Cmdlets dazu (`Write-Verbose`, `Write-Warning` und so weiter); hier geht es um den Betrieb, also darum, wie Planer, Pipelines und Überwachung die Ströme aufnehmen.

Zuerst die wichtigste Regel: Nur der Erfolgsstrom enthält das Ergebnis. Alles, was ein Skript oder eine Funktion ohne Umleitung ausgibt, landet dort und wird an den Aufrufer zurückgegeben. Wer mitten in einer Funktion eine Statusmeldung mit `Write-Output` oder einfach als Zeichenkette ausgibt, mischt sie unter die Ergebnisse. Für Meldungen sind die anderen Ströme da. `Write-Host` schreibt seit PowerShell 5.0 in den Informationsstrom (Stream 6); das ist der Grund, warum seine Ausgabe sich umleiten und abfangen lässt. Im Test lieferte die Umleitung `6>&1` für eine Zeile `Write-Host 'nur Host'` ein Objekt vom Typ `InformationRecord`.

```powershell
function Test-Streams {
    [CmdletBinding()] param()
    'Ergebnis (Output)'
    Write-Warning 'Warnung (3)'
    Write-Verbose 'Ausführlich (4)'
    Write-Debug 'Debug (5)'
    Write-Information 'Information (6)' -InformationAction Continue
    Write-Host 'Host (6, Information)'
    Write-Error 'Fehler (2)' -ErrorAction Continue
}

# Alle Ströme in den Erfolgsstrom umleiten und nach Typ zählen
$alle = Test-Streams -Verbose -Debug 4>&1 5>&1 3>&1 2>&1 6>&1
$alle | Group-Object { $_.GetType().Name } | Sort-Object Name

# Ströme in getrennte Dateien schreiben
$d = Join-Path ([IO.Path]::GetTempPath()) 'streams-demo'
New-Item -ItemType Directory $d -Force | Out-Null
$null = Test-Streams -Verbose 2>"$d/err.txt" 3>"$d/warn.txt" 4>"$d/verbose.txt" 6>"$d/info.txt"
```

Die Funktion erzeugt absichtlich jede Meldungsart einmal. Die Umleitungsoperatoren bestehen aus der Stream-Nummer und `>`: `2>&1` schickt den Fehlerstrom dorthin, wo der Erfolgsstrom hingeht, und mit `*>&1` (nicht im Beispiel) leitet man alle Ströme auf einmal um. Im ersten Aufruf fließen sämtliche Ströme in die Variable; das Zählen nach Typ zeigte im Test je einen `DebugRecord`, `ErrorRecord`, `VerboseRecord` und `WarningRecord`, zwei `InformationRecord`-Objekte (`Write-Information` und `Write-Host`) und einen `String`, das eigentliche Ergebnis. Man sieht daran, dass jede Meldungsart ein eigener, auswertbarer Objekttyp ist. Der zweite Aufruf schreibt die Ströme in vier Dateien; das Ergebnis geht mit `$null =` verloren, weil es hier nicht interessiert. Die Datei `verbose.txt` enthielt „Ausführlich (4)“, `warn.txt` die Warnung und `info.txt` beide Informationszeilen.

Warum ist das für den Betrieb wichtig? Weil Planer und Pipeline-Systeme in der Regel nur zwei Dinge sehen: die Standardausgabe und den Fehlerausgabekanal des Prozesses (stdout und stderr) sowie den Exit Code. Von den Ausgaben eines PowerShell-Prozesses gelangt der Erfolgsstrom nach stdout; der Fehlerstrom landet im Normalfall in stderr. Verbose, Debug und Warning erscheinen nur, wenn sie eingeschaltet sind, und werden dann ebenfalls auf die Konsole geschrieben. Ein Skript, das zusätzlich seine Nutzdaten mit Statustext vermischt, ist für den nachfolgenden Schritt unlesbar. Deshalb gilt: Ergebnisse als Objekte oder als eine klar definierte Textform in den Erfolgsstrom, alles Erzählende in die anderen Ströme oder in die Protokolldatei.

Die Ströme lassen sich auch ohne Dateien abfangen. Jedes Cmdlet und jede erweiterte Funktion kennt gemeinsame Parameter wie `-ErrorVariable`, `-WarningVariable`, `-InformationVariable` und `-OutVariable`. Sie sammeln die jeweiligen Meldungen in einer Variablen, ohne dass der Aufrufer die Umleitung kennen muss. Im Test lieferte ein Aufruf mit `-InformationVariable iv -ErrorVariable ev -WarningVariable wv` danach in `iv` die Information, und `ev` und `wv` enthielten je einen Eintrag. Man übergibt den Namen *ohne* Dollarzeichen, und mit einem vorangestellten Pluszeichen (`-ErrorVariable +ev`) hängt PowerShell neue Meldungen an, statt die Variable zu überschreiben.

> **Ergänzung aus der Nachrecherche**
>
> **Voreinstellungen steuern, was überhaupt entsteht.** Ob ein Strom Ausgabe erzeugt, regeln Einstellungsvariablen: `$VerbosePreference`, `$DebugPreference`, `$InformationPreference`, `$WarningPreference` und `$ErrorActionPreference`. Standardmäßig sind Verbose und Debug aus; `Write-Verbose` tut dann nichts. In Azure Pipelines kann der PowerShell-Schritt diese Voreinstellungen über Eingaben (zum Beispiel `debugPreference`) vorab setzen, und der Parameter `errorActionPreference` steht dort laut abgerufener Dokumentation standardmäßig auf `stop`, sodass ein Fehler den Schritt scheitern lässt. Der Eintrag `failOnStderr` lässt einen Schritt außerdem scheitern, sobald irgendetwas in den Fehlerstrom geschrieben wird.

> **Merke**
>
> Sechs Ströme, ein Ergebnis: Nur der Erfolgsstrom (1) trägt die Nutzdaten. Fehler (2), Warnung (3), Verbose (4), Debug (5) und Information (6) sind Meldungen. `Write-Host` schreibt in Strom 6. Umleiten mit `n>&1` oder `n>Datei`, abfangen mit den gemeinsamen Parametern `-ErrorVariable`, `-WarningVariable`, `-InformationVariable`.

```powershell
# FALSCH: Statusmeldung im Erfolgsstrom verfaelscht das Ergebnis
function Get-Anzahl { 'Zaehle ...'; 42 }
$n = Get-Anzahl        # $n ist ein Array: 'Zaehle ...' und 42

# RICHTIG: Meldung in den passenden Strom
function Get-Anzahl { Write-Verbose 'Zaehle ...'; 42 }
$n = Get-Anzahl -Verbose   # $n ist 42, die Meldung erscheint getrennt
```

Im falschen Beispiel besteht `$n` aus zwei Elementen, und ein späterer Vergleich wie `$n -gt 10` liefert kein Wahr oder Falsch, sondern eine gefilterte Liste. Das ist einer der häufigsten Fehler in selbst geschriebenen Funktionen.

> **Praxisbeispiel**
>
> Ein Skript für den Windows-Aufgabenplaner schreibt seine Ergebnisse als JSON in den Erfolgsstrom, den ein nachgelagerter Schritt einliest. Fortschritt und Warnungen gehen über `Write-Verbose` und `Write-Warning` in die Protokolldatei. Als ein Entwickler ein `Write-Host 'Fertig'` in den Erfolgsstrom-Pfad einbaut und der Einleseschritt mit einem Parse-Fehler scheitert, zeigt sich der Wert dieser Trennung.

## 9.3 Exit Codes und Status

Ein Protokoll beantwortet die Frage „Was ist passiert?“ für Menschen. Für Maschinen gibt es eine viel kürzere Antwort: den *Exit Code* (auch *Rückgabecode* oder *Beendigungscode*). Das ist eine ganze Zahl, die jeder Prozess beim Beenden an seinen Aufrufer zurückgibt. Nach einer langen Konvention bedeutet 0 „alles in Ordnung“ und jede andere Zahl „es gab ein Problem“. Der Windows-Aufgabenplaner, Cron, Jenkins, Azure Pipelines, GitHub Actions und jede Shell entscheiden anhand dieser Zahl, ob der nächste Schritt laufen darf, ob ein Alarm ausgelöst wird oder ob der Lauf als gescheitert gilt. Ein Skript, das bei einem Fehler 0 zurückgibt, belügt also alle Systeme um sich herum.

In PowerShell gibt es dafür mehrere Mechanismen, die man auseinanderhalten muss. Die automatische Variable `$?` enthält Wahr oder Falsch für den *letzten Befehl* innerhalb der Sitzung. Die Variable `$LASTEXITCODE` enthält die Zahl, die das zuletzt ausgeführte *externe Programm* (zum Beispiel `robocopy`, `git`, `terraform`) zurückgegeben hat. Und die Anweisung `exit N` beendet das Skript bzw. die Sitzung mit dem Exit Code N. Cmdlets setzen `$LASTEXITCODE` nicht.

Für eigene Skripte lohnt sich eine kleine, dokumentierte Tabelle von Codes. Im Beispiel gilt: 0 = in Ordnung, 10 = Warnung (Lauf erfolgreich, aber mit Auffälligkeiten), 20 = Fehler (Aufgabe nicht erfüllt), 99 = unerwarteter Fehler (Ausnahme, die niemand vorhergesehen hat). Wichtig ist, dass ein *Warnungs*-Code existiert; viele Systeme, etwa Nagios-artige Überwachungen, unterscheiden zwischen „kritisch“ und „Warnung“, und damit ist Ihr Skript direkt anschlussfähig.

```powershell
# ex.ps1
param([ValidateSet('ok','warn','fehler','boom')] [string] $Fall = 'ok')

$Codes = @{ Ok = 0; Warnung = 10; Fehler = 20; Unerwartet = 99 }

try {
    switch ($Fall) {
        'ok'     { Write-Output 'Alles gut' ; $rc = $Codes.Ok }
        'warn'   { Write-Warning 'Teilweise erfolgreich' ; $rc = $Codes.Warnung }
        'fehler' { Write-Error 'Aufgabe nicht erfüllt' -ErrorAction Continue ; $rc = $Codes.Fehler }
        'boom'   { throw 'Unvorhergesehen' }
    }
}
catch {
    Write-Error $_
    $rc = $Codes.Unerwartet
}
exit $rc
```

Das Skript nimmt über `ValidateSet` nur vier Eingaben an. Die Tabelle `$Codes` gibt den Zahlen Namen, damit im Code nie nackte Zahlen stehen. Der `try`-Block führt den Fall aus und setzt in jedem Zweig `$rc`; im Fall `boom` wird absichtlich eine Ausnahme ausgelöst, die der `catch`-Block abfängt, meldet und mit 99 quittiert. Die letzte Zeile, `exit $rc`, ist entscheidend: Erst sie macht aus dem berechneten Wert den Exit Code des Prozesses.

Im Test mit PowerShell 7.6.6 ergaben die Aufrufe `pwsh -File ./ex.ps1 -Fall ok`, `warn`, `fehler` und `boom` nacheinander die Exit Codes 0, 10, 20 und 99. Die Auswertung geschieht in der aufrufenden Shell: unter Linux mit `echo $?`, in der Windows-Eingabeaufforderung mit `echo %ERRORLEVEL%`, in PowerShell mit `$LASTEXITCODE`.

Nun zu den Fallen, die im Test deutlich wurden. Erstens: Der Weg des Aufrufs bestimmt, was ankommt. Mit `-File` liefert ein `exit 20` tatsächlich 20. Wird dasselbe Skript dagegen mit `pwsh -Command "& ./ex.ps1 -Fall fehler"` gestartet, kam im Test als Exit Code nur 1 an; mit `-Command` unterscheidet PowerShell lediglich zwischen 0 (letzter Befehl erfolgreich) und 1 (letzter Befehl fehlgeschlagen). Wer mit `-Command` den genauen Code weiterreichen will, muss ihn ausdrücklich durchreichen: `pwsh -Command "./ex.ps1 -Fall warn; exit $LASTEXITCODE"` lieferte 10. Ohne diesen Zusatz endete ein Aufruf wie `-Command "./ex.ps1 -Fall warn; Write-Host after"` mit 0, weil das letzte Statement (`Write-Host`) erfolgreich war. Zweitens: Eine unbehandelte `throw`-Ausnahme ergab im Test sowohl mit `-Command` als auch mit `-File` den Exit Code 1. Drittens: Ein bloßes `Write-Error` am Ende eines `-File`-Skripts ergab Exit Code 0, weil der Fehler nicht abbrach; er erscheint nur im Fehlerstrom. Viertens: Wenn das letzte Statement eines `-File`-Skripts ein externes Programm war, das mit 5 endete (im Test `sh -c 'exit 5'`), blieb der Skript-Exit-Code 0, weil der Exit Code des Skripts nicht automatisch `$LASTEXITCODE` ist.

> **Merke**
>
> Ein Exit Code entsteht in PowerShell nur durch `exit N` (oder durch eine unbehandelte Ausnahme, die 1 ergibt). `Write-Error` allein ändert ihn nicht. `$LASTEXITCODE` wird von externen Programmen gesetzt, nicht von Cmdlets. Startet man ein Skript mit `-Command`, kommt ohne `exit $LASTEXITCODE` nur 0 oder 1 an; mit `-File` kommt der Wert von `exit` an.

### Externe Programme richtig auswerten.

Ruft ein Skript Werkzeuge wie `robocopy`, `git` oder `terraform` auf, wirft PowerShell bei einem Fehlercode keine Ausnahme; auch `$ErrorActionPreference = 'Stop'` greift hier nicht. Man muss selbst prüfen. Ein Spezialfall ist `robocopy`: Seine Codes 0 bis 7 bedeuten Erfolg in verschiedenen Abstufungen (zum Beispiel „Dateien kopiert“ oder „nichts zu tun“), erst 8 und höher stehen für Fehler. Eine pauschale Prüfung „ungleich 0 ist Fehler“ wäre hier falsch.

```powershell
function Invoke-Extern {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Programm,
        [string[]] $Argumente = @(),
        [int[]] $ErlaubteCodes = @(0)
    )
    $ausgabe = & $Programm @Argumente 2>&1
    $rc = $LASTEXITCODE
    if ($rc -notin $ErlaubteCodes) {
        throw "Programm '$Programm' endete mit Code $rc. Ausgabe: $($ausgabe -join ' | ')"
    }
    $ausgabe
}

Invoke-Extern -Programm 'sh' -Argumente '-c','echo hallo; exit 0'
try { Invoke-Extern -Programm 'sh' -Argumente '-c','echo schlecht; exit 5' }
catch { "Fehler gefangen: $($_.Exception.Message)" }
Invoke-Extern -Programm 'sh' -Argumente '-c','exit 3' -ErlaubteCodes 0,3
```

Die Hilfsfunktion kapselt die Prüfung. Das Programm wird mit `&` aufgerufen und die Argumente über *Splatting* (`@Argumente`) als Feld übergeben, sodass Leerzeichen nicht zu Problemen führen. Die Umleitung `2>&1` sammelt die Fehlerausgabe mit ein. Direkt danach merkt sich die Funktion `$LASTEXITCODE`, bevor ein anderer Befehl ihn überschreiben kann. Ist der Code nicht in der Liste der erlaubten Werte, löst die Funktion mit `throw` eine Ausnahme aus, die man mit `try/catch` behandeln kann. Das Beispiel verwendet `sh`, weil der Testlauf unter Linux stattfand; unter Windows ersetzt man das Programm, etwa durch `robocopy` mit `-ErlaubteCodes 0,1,2,3,4,5,6,7`. Dieses Robocopy-Beispiel wurde nicht ausgeführt.

> **Ergänzung aus der Nachrecherche**
>
> **Native Befehle und Fehlerverhalten in PowerShell 7.** Die Voreinstellungsvariable `$PSNativeCommandUseErrorActionPreference` ist ein experimentelles Feature in PowerShell 7.3; sie lässt Fehlercodes externer Programme wie Fehler behandeln. Ob und in welcher Form dieses Verhalten in Ihrer Version fest eingebaut ist, prüfen Sie in der Dokumentation `about_Preference_Variables` Ihrer PowerShell-Version. Diese Seite wurde für dieses Modul nicht abgerufen; die Angabe stammt aus allgemeinem Fachwissen und ist nicht live verifiziert.

### Exit Codes in Pipelines.

Laut abgerufener Dokumentation zum PowerShell-Schritt in Azure Pipelines wird der zuletzt gesetzte `$LASTEXITCODE` an den Agenten weitergereicht, sofern die Option `ignoreLASTEXITCODE` nicht gesetzt ist; ein Wert ungleich 0 lässt den Schritt scheitern. Das ist praktisch, bedeutet aber auch: Ein früher Aufruf eines externen Programms, der mit einem Wert ungleich 0 endete und dessen Code nie zurückgesetzt wurde, kann einen sonst erfolgreichen Schritt am Ende scheitern lassen. Wer das nicht will, setzt `$global:LASTEXITCODE = 0`, nachdem der Code bewusst behandelt wurde.

```powershell
# FALSCH: Fehler wird gemeldet, aber der Prozess endet mit 0
try { Copy-Item 'quelle.txt' 'ziel.txt' -ErrorAction Stop }
catch { Write-Error "Kopieren fehlgeschlagen: $_" }
# (Skriptende ohne exit -> Planer sieht Erfolg)

# RICHTIG: Fehler in Exit Code uebersetzen
try { Copy-Item 'quelle.txt' 'ziel.txt' -ErrorAction Stop }
catch { Write-Error "Kopieren fehlgeschlagen: $_" ; exit 20 }
```

Der falsche Ansatz ist der Klassiker: Der Fehler steht im Protokoll, aber niemand wird alarmiert, weil der Planer nur den Exit Code liest. Im richtigen Beispiel steht nach der Meldung `exit 20`.

> **Praxisbeispiel**
>
> In einer Pipeline soll eine Datenbankmigration nur laufen, wenn der Test-Schritt davor erfolgreich war. Das Test-Skript beendet sich bei Auffälligkeiten mit 10 und bei Fehlern mit 20; die Pipeline wertet 10 als „gelb“ (weiterlaufen, aber melden) und 20 als „rot“ (abbrechen). Ohne definierte Codes wäre diese Unterscheidung nur über Textsuche in der Ausgabe möglich.

## 9.4 Eventlog-Integration

Eine eigene Protokolldatei ist gut, aber Betriebsteams haben oft schon ein zentrales System, in dem alle Ereignisse der Rechner zusammenlaufen. Unter Windows ist das das *Ereignisprotokoll* (englisch *Event Log*), das die Ereignisanzeige darstellt und das Werkzeuge wie Windows Event Forwarding, Überwachungsagenten und SIEM-Systeme einsammeln. Wenn ein Skript seine wichtigen Ereignisse dort ablegt (Start, Ende, Fehler), erscheinen sie in denselben Auswertungen und Alarmen wie die des Betriebssystems selbst. Unter Linux übernimmt diese Rolle *Syslog* bzw. das *Journal* von systemd.

Beim Schreiben ins Ereignisprotokoll braucht jeder Absender eine *Quelle* (englisch *Source*), also einen Namen, unter dem seine Einträge erscheinen, und eine *Ereignis-ID*, eine Zahl, die die Art des Ereignisses kennzeichnet. Eine Quelle muss einmalig registriert werden, was Administratorrechte erfordert; danach darf auch ein normales Konto Einträge schreiben. Das ist die häufigste Falle: Das Skript läuft im Test als Administrator durch und scheitert in der Produktion mit „Quelle nicht vorhanden“ oder „Zugriff verweigert“, weil die Registrierung nie stattgefunden hat.

Ein wichtiger Befund zur Version: Die Cmdlets `Get-EventLog`, `Write-EventLog` und `New-EventLog` gehören zu Windows PowerShell 5.1 und sind in PowerShell 7 nicht enthalten. Das entspricht der abgerufenen Seite `about_Logging` der Dokumentation, die EventLog-Cmdlets nur für Windows PowerShell 5.1 und älter nennt, und wurde im Test bestätigt: In PowerShell 7.6.6 waren `Write-EventLog` und `New-WinEvent` nicht vorhanden (der Test lief unter Linux). Unter Windows bleibt in PowerShell 7 der Weg über die .NET-Klasse `System.Diagnostics.EventLog`, die in diesem Beispiel verwendet wird; `Get-WinEvent` ist für das *Lesen* vorgesehen.

```powershell
function Write-BetriebsEvent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Meldung,
        [ValidateSet('Information','Warning','Error')] [string] $Typ = 'Information',
        [int] $EreignisId = 1000,
        [string] $Quelle = 'MeinSkript',
        [string] $Log = 'Application'
    )
    if ($IsWindows -or $PSVersionTable.PSEdition -eq 'Desktop') {
        if (-not [System.Diagnostics.EventLog]::SourceExists($Quelle)) {
            # Registrierung braucht Administratorrechte, einmalig bei der Installation
            [System.Diagnostics.EventLog]::CreateEventSource($Quelle, $Log)
        }
        $art = [System.Diagnostics.EventLogEntryType]::$Typ
        [System.Diagnostics.EventLog]::WriteEntry($Quelle, $Meldung, $art, $EreignisId)
    }
    else {
        $prio = switch ($Typ) { 'Error' { 'user.err' } 'Warning' { 'user.warning' } default { 'user.info' } }
        & logger -t $Quelle -p $prio "[$EreignisId] $Meldung"
    }
}

Write-BetriebsEvent -Meldung 'Sicherung abgeschlossen' -EreignisId 1001
Write-BetriebsEvent -Meldung 'Zielplatte fast voll' -Typ Warning -EreignisId 2001
```

Die Funktion hat zwei Zweige. Unter Windows (erkennbar an `$IsWindows` oder an der Edition „Desktop“ für Windows PowerShell 5.1) prüft sie, ob die Quelle existiert, und legt sie bei Bedarf an; das darf in der Praxis nicht im Skript selbst geschehen, sondern gehört in ein einmaliges Installationsskript mit erhöhten Rechten. Dann wandelt sie den Text des Typs in den passenden Wert der Aufzählung `EventLogEntryType` um und ruft `WriteEntry` mit Quelle, Text, Art und ID auf. Unter Linux ruft sie das Programm `logger` auf, das die Meldung an Syslog weitergibt; das Format „\[ID\] Text“ ersetzt die dort nicht vorhandene Ereignis-ID.

Der Linux-Zweig lief im Test ohne Fehler; einen Syslog-Dienst, aus dem sich die Zeilen hätten zurücklesen lassen, gab es im Arbeitsbereich nicht. Der Windows-Zweig konnte nicht ausgeführt werden und wurde nur auf Syntax geprüft.

Wie liest man die Einträge wieder aus? Mit `Get-WinEvent` und einem Filter. Der Hashtable-Filter ist deutlich schneller als das Laden aller Einträge und anschließendes `Where-Object`, weil die Filterung im Ereignisdienst stattfindet.

```powershell
# Nur auf Syntax geprueft (Windows):
Get-WinEvent -FilterHashtable @{
    LogName      = 'Application'
    ProviderName = 'MeinSkript'
    Level        = 2,3          # 2 = Fehler, 3 = Warnung
    StartTime    = (Get-Date).AddDays(-1)
} | Select-Object TimeCreated, Id, LevelDisplayName, Message
```

Das Beispiel sucht im Anwendungsprotokoll alle Einträge der Quelle „MeinSkript“ der letzten 24 Stunden mit Level Fehler oder Warnung. Die Schlüssel des Filters (`LogName`, `ProviderName`, `Level`, `StartTime`) sind die in der Dokumentation von `Get-WinEvent` beschriebenen Filterfelder; die Seite selbst wurde für dieses Modul nicht abgerufen.

> **Ergänzung aus der Nachrecherche**
>
> **Was PowerShell selbst protokolliert.** Die abgerufene Seite `about_Logging_Windows` beschreibt die Protokollierung der Engine selbst: Das *Script Block Logging* schreibt den Inhalt ausgeführter Skriptblöcke in das Ereignisprotokoll (Ereignis-ID 4104), das *Module Logging* und die *Transcription* (Mitschnitt der Sitzung in eine Textdatei) ergänzen das. Für Betriebs- und Sicherheitsteams ist das ein Fundus, weil sich daran nachvollziehen lässt, was tatsächlich ausgeführt wurde. Zusätzlich gibt es „geschütztes Ereignisprotokoll“ (Protected Event Logging), bei dem Einträge mit einem öffentlichen Zertifikat verschlüsselt werden, damit Geheimnisse in Skriptblöcken nicht im Klartext im Protokoll stehen. Dieser Teil betrifft Windows; unter Linux gibt es keine Entsprechung im Ereignisprotokoll.

> **Merke**
>
> Windows: Quelle einmalig mit Administratorrechten registrieren, danach mit `WriteEntry` und Ereignis-ID schreiben, mit `Get-WinEvent -FilterHashtable` lesen. In PowerShell 7 fehlen `Write-EventLog` und `Get-EventLog`. Linux: `logger`, Auswertung über Syslog bzw. Journal. Ereignis-IDs sind ein Vertrag mit der Überwachung und werden dokumentiert.

```powershell
# FALSCH: jedes Skript erfindet IDs, Quelle wird bei jedem Lauf angelegt
Write-EventLog -LogName Application -Source 'x' -EventId 1 -Message $text   # fehlt in PS 7

# RICHTIG: dokumentierte IDs, Quelle bei der Installation, .NET-Weg
# 1000-1999 Information, 2000-2999 Warnung, 3000-3999 Fehler
Write-BetriebsEvent -Meldung $text -Typ Error -EreignisId 3001
```

Die falsche Variante funktioniert nur in Windows PowerShell 5.1, benutzt die ID 1 für alles und macht eine Auswertung unmöglich. Die richtige Variante legt Zahlenbereiche fest: Die Überwachung kann dann auf „alle IDs ab 3000“ alarmieren, ohne jede Meldung einzeln zu kennen.

> **Praxisbeispiel**
>
> Das Team betreibt einen Überwachungsagenten, der im Anwendungsprotokoll nach Einträgen mit Level „Fehler“ sucht und sie als Ticket weiterleitet. Das Sicherungsskript schreibt Fehler mit ID 3001 (Ziel nicht erreichbar) und 3002 (zu wenig Platz); die Ticketregeln unterscheiden daran die Zuständigkeit: Netzwerkteam oder Speicherteam.

## 9.5 Metriken aus Skripten

Protokolle erzählen einzelne Ereignisse; *Metriken* sind Zahlen, die sich über die Zeit vergleichen lassen: Wie viele Dateien hat der Import heute verarbeitet? Wie lange dauerte der Lauf? Wann war der letzte erfolgreiche Lauf? Mit solchen Zahlen sieht man Trends (der Lauf wird jede Woche langsamer) und kann Schwellenwerte alarmieren (länger als 30 Minuten). Ein Skript, das nur „Fertig“ ins Protokoll schreibt, kann das nicht leisten.

Als Erstes misst man die Laufzeit. `Measure-Command` führt einen Block aus und liefert ein `TimeSpan`-Objekt; für Messungen über mehrere Abschnitte eignet sich die Klasse `System.Diagnostics.Stopwatch`, die man startet und stoppt. Beide Wege wurden im Test benutzt und lieferten Millisekundenwerte. Wichtig: `Measure-Command` verwirft die Ausgabe des Blocks, der Ergebniswert geht also verloren; wer ihn braucht, verwendet die Stopwatch.

Wie kommen die Zahlen zum Überwachungssystem? Es gibt zwei Grundmuster. Beim *Push*-Muster sendet das Skript seine Werte aktiv an einen Dienst. Beim *Pull*-Muster legt das Skript die Werte in eine Datei, und ein Agent liest sie. Für kurzlebige Skripte ist Pull über eine Textdatei besonders einfach. Das *Prometheus*-Textformat ist dafür weit verbreitet: Jede Zeile ist `name{label="wert"} zahl`, und der „Node Exporter“ kann solche Dateien über seinen Textfile-Collector einlesen. Dieses Format ist hier als Beispiel gewählt; die Prometheus-Dokumentation wurde für dieses Modul nicht abgerufen, die Angabe beruht auf Fachwissen.

```powershell
function Export-Metriken {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Job,
        [Parameter(Mandatory)] [hashtable] $Werte,
        [Parameter(Mandatory)] [string] $Pfad
    )
    $inv = [System.Globalization.CultureInfo]::InvariantCulture
    $zeilen = foreach ($name in $Werte.Keys | Sort-Object) {
        $zahl = ([double]$Werte[$name]).ToString($inv)
        'skript_{0}{{job="{1}"}} {2}' -f $name, $Job, $zahl
    }
    $tmp = "$Pfad.tmp"
    Set-Content -LiteralPath $tmp -Value $zeilen -Encoding utf8
    Move-Item -LiteralPath $tmp -Destination $Pfad -Force   # atomar ersetzen
}

$sw = [System.Diagnostics.Stopwatch]::StartNew()
Start-Sleep -Milliseconds 300
$sw.Stop()
Export-Metriken -Job 'import' -Pfad (Join-Path ([IO.Path]::GetTempPath()) 'import.prom') -Werte @{
    laufzeit_sekunden   = $sw.Elapsed.TotalSeconds
    dateien_verarbeitet = 42
    fehler_anzahl       = 0
    letzter_erfolg_unix = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
}
```

Die Funktion macht drei Dinge bewusst. Sie formatiert Zahlen mit der invarianten Kultur: Unter einer deutschsprachigen Einstellung erzeugt `(1.5).ToString()` den Text „1,5“ (im Test mit de-AT bestätigt), während die invariante Kultur „1.5“ liefert; das Prometheus-Format erwartet den Punkt. Sie schreibt die Datei zuerst unter einem temporären Namen und ersetzt dann die endgültige Datei mit `Move-Item`, damit der einlesende Agent nie eine halb geschriebene Datei sieht; eine solche Ersetzung gilt als atomar. Und sie schreibt eine Metrik „letzter Erfolg“ als Unix-Zeitstempel. Dieses Muster ist wertvoll, weil die Überwachung dann auch merkt, wenn das Skript gar nicht mehr läuft: Die Regel lautet „Alarm, wenn jetzt minus letzter Erfolg größer als 26 Stunden“. Ein Skript, das nie startet, kann keinen Fehler melden. Der Testlauf schrieb eine Datei mit Zeilen wie `skript_dateien_verarbeitet{job="import"} 42`.

> **Ergänzung aus der Nachrecherche**
>
> **Temporäres Verzeichnis plattformübergreifend.** Das Beispiel ermittelt den Ordner für Temporärdateien mit `[IO.Path]::GetTempPath()`. Die Umgebungsvariable `$env:TEMP` ist unter Windows gesetzt, unter Linux dagegen in der Regel nicht; ein Pfad wie `"$env:TEMP/import.prom"` würde dort ohne Verzeichnis entstehen. Bei Skripten für mehrere Betriebssysteme prüft man Umgebungsvariablen daher vorher oder nutzt die .NET-Methode.

### Welche Werte lohnen sich?

Nützlich sind wenige, stabil benannte Zahlen: Laufzeit, Menge der verarbeiteten Elemente, Menge der Fehler, Zeitpunkt des letzten Erfolgs und gegebenenfalls die Größe von Warteschlangen oder Ausgabedateien. Zu viele Metriken verwässern die Aufmerksamkeit; Namen sollten Einheiten tragen (`_sekunden`, `_bytes`), und die Bezeichner dürfen sich nie ändern, weil Auswertungen daran hängen. Einzelne Werte wie Benutzernamen sollten nicht als Beschriftung vorkommen, weil jede neue Kombination eine neue Zeitreihe erzeugt.

```powershell
# FALSCH: Zahl als Text in lokaler Kultur, direkt in die Zieldatei, nur Erfolgsmeldung
"Dauer: $($sw.Elapsed.TotalSeconds)" | Out-File metrik.prom

# RICHTIG: stabiler Name, invariante Kultur, atomar, mit "letzter Erfolg"
Export-Metriken -Job 'import' -Pfad $pfad -Werte @{ laufzeit_sekunden = $sw.Elapsed.TotalSeconds }
```

Die falsche Zeile erzeugt eine Datei, die ein Agent nicht parsen kann, und schreibt unter deutscher Kultur Kommazahlen.

> **Merke**
>
> Metriken sind wenige, stabil benannte Zahlen mit Einheit: Laufzeit, Mengen, Fehler und vor allem der Zeitpunkt des letzten Erfolgs. Zahlen immer mit invarianter Kultur formatieren, Dateien atomar ersetzen. Der Zeitstempel des letzten Erfolgs erkennt auch Skripte, die gar nicht mehr laufen.

> **Praxisbeispiel**
>
> Ein nächtlicher Abgleich schreibt nach jedem Lauf eine Metrikdatei. Das Dashboard zeigt die Laufzeit der letzten 30 Tage; an einem Anstieg von 4 auf 25 Minuten erkennt das Team einen fehlenden Datenbankindex, bevor der Lauf in das Zeitfenster der Morgenverarbeitung hineinragt.

## 9.6 Troubleshooting Methodik

Wenn ein Skript nicht tut, was es soll, ist die Versuchung groß, wild zu ändern und neu zu starten, bis es zufällig klappt. Das kostet Zeit und führt oft zu zusätzlichen Fehlern. Erfahrene Administratoren gehen stattdessen nach einer festen Reihenfolge vor, die sich in vier Schritte fassen lässt: das Problem *beschreiben*, die Umgebung *erfassen*, die Ursache durch *kleine Experimente* eingrenzen und erst dann *ändern*, jeweils nur eine Sache auf einmal.

Der erste Schritt klingt banal, entscheidet aber über den Aufwand: Was genau ist die erwartete und was die tatsächliche Beobachtung? „Das Skript geht nicht“ ist keine Beschreibung. „Der Lauf endet um 01:07 mit Exit Code 1, im Fehlerstrom steht die Meldung X, gestern lief er noch“ ist eine. Gerade die letzte Frage, *was hat sich seit dem letzten Erfolg geändert?*, löst erstaunlich viele Fälle: ein neues Modulupdate, ein abgelaufenes Kennwort oder Zertifikat, eine geänderte Firewall-Regel, ein anderes Konto, eine neue PowerShell-Version.

Der zweite Schritt erfasst die Umgebung, in der das Skript wirklich lief. Ein klassischer Grund für „bei mir geht es, im Planer nicht“ sind Unterschiede: andere PowerShell-Edition (Windows PowerShell 5.1 gegen PowerShell 7), anderes Benutzerkonto mit anderen Rechten, anderes Arbeitsverzeichnis, andere Ausführungsrichtlinie, andere Kultureinstellung, andere Modulpfade. Statt das zu raten, lässt man das Skript im Fehlerfall einen kurzen *Diagnosebericht* schreiben.

```powershell
function Get-Diagnose {
    [CmdletBinding()]
    param([string[]] $Befehl = @(), [string[]] $Modul = @())
    $ist = if ($IsWindows -or $PSVersionTable.PSEdition -eq 'Desktop') {
        ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
            [Security.Principal.WindowsBuiltInRole]::Administrator)
    } else { (& id -u) -eq '0' }
    [pscustomobject][ordered]@{
        PSVersion       = $PSVersionTable.PSVersion.ToString()
        Edition         = $PSVersionTable.PSEdition
        OS              = $PSVersionTable.OS
        Benutzer        = [Environment]::UserName
        IstAdmin        = $ist
        Verzeichnis     = (Get-Location).Path
        Kultur          = (Get-Culture).Name
        ExecutionPolicy = if (Get-Command Get-ExecutionPolicy -ErrorAction SilentlyContinue) { try { (Get-ExecutionPolicy).ToString() } catch { 'n/a' } } else { 'n/a' }
        ErrorAction     = $ErrorActionPreference
        ModulPfade      = ($env:PSModulePath -split [IO.Path]::PathSeparator)
        Befehle         = foreach ($b in $Befehl) {
            $c = Get-Command $b -All -ErrorAction SilentlyContinue
            [pscustomobject]@{ Name = $b; Treffer = @($c).Count; Quelle = (@($c | ForEach-Object { $_.Source }) -join '; ') }
        }
        Module          = foreach ($m in $Modul) {
            $x = Get-Module -ListAvailable -Name $m | Sort-Object Version -Descending | Select-Object -First 1
            [pscustomobject]@{ Name = $m; Version = $x.Version; Pfad = $x.ModuleBase }
        }
    }
}

$bericht = Get-Diagnose -Befehl Get-Process, gibtesnicht -Modul Microsoft.PowerShell.Utility, NixDa
$bericht | Format-List
```

Die Funktion sammelt die Fakten in einem einzigen Objekt. Die Version und Edition zeigen, ob man in Windows PowerShell oder PowerShell 7 ist. Die Prüfung auf Administratorrechte läuft je nach Betriebssystem verschieden: Unter Windows fragt sie die Rolle des aktuellen Benutzers ab (dieser Zweig wurde nicht ausgeführt), unter Linux prüft sie, ob die Benutzernummer 0 ist (im Test: `True`, weil der Arbeitsbereich als root läuft). `Get-Command -All` liefert alle Befehle mit einem Namen, nicht nur den ersten; so erkennt man, wenn ein Alias, eine Funktion und ein Programm denselben Namen tragen und nicht der erwartete aufgerufen wird. Der Test lieferte für `Get-Process` einen Treffer aus dem Modul `Microsoft.PowerShell.Management`, für den erfundenen Namen null Treffer, für das vorhandene Modul seine Version und seinen Pfad und für das fehlende Modul leere Felder. Die Kultur war im Arbeitsbereich leer (invariant), auf einem deutschen Rechner steht dort etwa `de-AT`.

Der dritte Schritt ist das Eingrenzen. Man halbiert die Fehlersuche wie bei einem Telefonbuch: Läuft der erste Teil des Skripts fehlerfrei, liegt der Fehler im zweiten, dort wieder der Mitte und so weiter. Praktische Hilfen dafür sind `Set-PSDebug -Trace 1` (zeigt jede ausgeführte Zeile), `Set-StrictMode -Version Latest` (macht den Zugriff auf nicht gesetzte Variablen zum Fehler), Haltepunkte in Visual Studio Code und der gezielte Blick auf das letzte Fehlerobjekt. Dieses steht in `$Error[0]` und enthält viel mehr als die kurze Meldung.

```powershell
$Error.Clear()
try { Get-Item /nix/da -ErrorAction Stop } catch { }
$e = $Error[0]
$e.Exception.GetType().Name     # ItemNotFoundException
$e.CategoryInfo.Category        # ObjectNotFound
$e.InvocationInfo.Line.Trim()   # die Zeile, in der es passierte
$e.ScriptStackTrace             # Aufrufkette mit Zeilennummern
$e.FullyQualifiedErrorId        # maschinenlesbare Kennung
```

Das Fehlerobjekt (`ErrorRecord`) ist die wichtigste Quelle. Im Test ergab der Zugriff auf einen nicht existierenden Pfad den Ausnahmetyp `ItemNotFoundException` und die Kategorie `ObjectNotFound`; die Eigenschaft `ScriptStackTrace` nannte die Skriptdatei und die Zeilennummer. Die Kennung `FullyQualifiedErrorId` ist besonders nützlich, weil man darauf in einem `catch`-Block gezielt reagieren kann, ohne Meldungstexte zu vergleichen (die sich ändern und lokalisiert sind).

Ein besonderes Kapitel sind Fehler beim *Parameterbinden*, also beim Zuordnen der übergebenen Werte zu den Parametern einer Funktion. Sie entstehen, bevor die erste Zeile der Funktion läuft, und sehen deshalb oft rätselhaft aus. Im Test lieferten die typischen Fälle diese Kennungen: ein Text für einen ganzzahligen Parameter `ParameterArgumentTransformationError`, ein unbekannter Parametername `NamedParameterNotFound`, ein Text, der aus der Pipeline in einen ganzzahligen Parameter fließt, `InputObjectNotBound`, und ein unbekannter Befehl eine `CommandNotFoundException`.

```powershell
function Get-Demo {
    [CmdletBinding()]
    param([Parameter(ValueFromPipeline)] [int] $Anzahl)
    process { "Anzahl=$Anzahl" }
}
try { Get-Demo -Anzahl 'abc' -ErrorAction Stop } catch { $_.FullyQualifiedErrorId }
try { Get-Demo -Gibtsnicht 1 -ErrorAction Stop }  catch { $_.FullyQualifiedErrorId }
try { 'abc' | Get-Demo -ErrorAction Stop }        catch { $_.FullyQualifiedErrorId }

# Wie bindet PowerShell? Ablauf sichtbar machen:
Trace-Command -Name ParameterBinding -PSHost -Expression { 5 | Get-Demo }
```

Der Befehl `Trace-Command -Name ParameterBinding` schreibt die inneren Schritte der Parameterbindung auf den Bildschirm. Im Test erschienen unter anderem die Stationen `BIND NAMED`, `BIND POSITIONAL`, `COERCE arg to [System.Int32]` und `BIND PIPELINE object to parameters`, am Ende folgte „BIND arg \[5\] to param \[Anzahl\] SUCCESSFUL“. Mit dieser Spur sieht man, ob PowerShell einen Wert umwandeln musste (*Coercion*), warum ein Pipeline-Objekt nicht gebunden wurde und welche Parameter „Pflicht“ sind. Die Ausgabe ist ausführlich und eignet sich als letzte Waffe, nicht als Dauerzustand.

> **Ergänzung aus der Nachrecherche**
>
> **Zugriff verweigert nicht getestet.** Die Fehlermeldung „Zugriff verweigert“ ist im Alltag der häufigste Fehler überhaupt. Sie lässt sich mit der Ausnahme `UnauthorizedAccessException` gezielt abfangen: `catch [System.UnauthorizedAccessException] { ... }`. Im Arbeitsbereich, in dem dieses Skript entstand, lief alles mit Administratorrechten (root); die Meldung konnte deshalb nicht erzeugt werden, und das Beispiel ist nicht getestet. Bei der Fehlersuche vergleicht man in so einem Fall mit `whoami` (Windows) oder `id` (Linux), unter welchem Konto das Skript tatsächlich lief, und prüft die Rechte des Ziels.

Der vierte Schritt ist das Ändern, und zwar eine Sache pro Versuch. Wer drei Dinge gleichzeitig ändert und der Fehler verschwindet, weiß nicht, welche Änderung geholfen hat, und weiß auch nicht, ob die anderen beiden Schäden anrichten. Man notiert, was man geändert hat und mit welchem Ergebnis; diese Notizen sind später die Grundlage für einen Eintrag in der Wissensdatenbank (siehe 9.8).

> **Merke**
>
> Vier Schritte: beschreiben (erwartet gegen tatsächlich, was hat sich geändert?), Umgebung erfassen (Edition, Konto, Rechte, Verzeichnis, Kultur, Module), eingrenzen (halbieren, `$Error[0]`, `Trace-Command`), eine Änderung pro Versuch. Meldungstexte sind schlechte Kriterien; `FullyQualifiedErrorId` und Ausnahmetyp sind stabil.

```powershell
# FALSCH: auf den Meldungstext pruefen (aendert sich mit Sprache und Version)
catch { if ($_.Exception.Message -like '*Zugriff*') { ... } }

# RICHTIG: auf den Ausnahmetyp pruefen
catch [System.UnauthorizedAccessException] { ... }
catch [System.IO.FileNotFoundException]    { ... }
```

Die falsche Variante funktioniert nur auf einem deutschsprachigen System und bricht, sobald dieselbe Software auf einem englischen Server läuft. Typbasierte `catch`-Blöcke sind unabhängig davon.

> **Praxisbeispiel**
>
> Ein Skript lief monatelang als Aufgabe unter einem Dienstkonto und scheitert plötzlich. Der Diagnosebericht zeigt, dass der Dienst-Host auf PowerShell 7 umgestellt wurde: Ein Modul, das nur unter Windows PowerShell 5.1 verfügbar ist, wird nicht mehr gefunden (`Module`-Eintrag mit leerer Version). Ohne Diagnosebericht hätte man das Skript selbst durchsucht.

## 9.7 Typische Fehlerbilder

Viele Störungen wiederholen sich. Wer die häufigsten Fehlerbilder samt Ursache kennt, spart sich die Methodik aus 9.6 und kommt in Minuten statt Stunden ans Ziel. Dieses Kapitel ist ein Katalog der Bilder, die in der Praxis immer wieder auftreten. Jedes besteht aus Symptom, wahrscheinlicher Ursache und erster Prüfung; die Reihenfolge entspricht ungefähr der Häufigkeit.

### Symptom: Es funktioniert interaktiv, aber nicht im Planer.

Ursachen sind meist ein anderes Konto (kein Zugriff auf Netzfreigaben, die interaktiv per Anmeldung verbunden waren), ein anderes Arbeitsverzeichnis (relative Pfade zeigen ins Nichts), ein nicht geladenes Profil (Aliase und Funktionen aus `$PROFILE` fehlen) oder eine Umgebungsvariable, die nur in der interaktiven Sitzung gesetzt ist. Erste Prüfung: Diagnosebericht aus 9.6 im Planerlauf schreiben und mit dem interaktiven vergleichen. Im Skript gehören Pfade deshalb absolut oder relativ zu `$PSScriptRoot`.

### Symptom: Das Skript läuft, tut aber nichts.

Ein häufiger Grund ist ein Filter, der alles ausschließt, oder eine Schleife über eine leere Liste. Ein Beispiel: `Get-ChildItem -Filter *.csv` findet nichts, weil die Dateien Großbuchstaben haben und das Dateisystem (unter Linux) Groß- und Kleinschreibung unterscheidet. Ich habe die Linux-Variante dieser Falle für dieses Modul nicht nachgestellt; auf Windows ist der Dateisystemvergleich in der Regel unabhängig von der Schreibweise. Prüfen Sie bei Skripten, die auf beiden Systemen laufen, Pfade und Dateinamen deshalb ausdrücklich.

### Symptom: Fehler werden verschluckt.

Ohne `-ErrorAction Stop` sind viele Fehler in Cmdlets *nicht abbrechend*: Der Fehler erscheint im Fehlerstrom, das Skript läuft aber weiter, und ein `try/catch` springt nicht an. Diese Eigenheit aus Modul 5 ist eine der häufigsten Ursachen für „das Skript meldete Erfolg, aber die Datei fehlt“. Abhilfe: `-ErrorAction Stop` an den kritischen Aufrufen oder `$ErrorActionPreference = 'Stop'` zu Beginn.

```powershell
# FALSCH: Fehler ist nicht abbrechend, catch wird nie erreicht
try { Get-Item /nix/da } catch { "gefangen" }
"weiter" 

# RICHTIG: abbrechend machen
try { Get-Item /nix/da -ErrorAction Stop } catch { "gefangen" }
```

Im ersten Fall schreibt PowerShell eine Fehlermeldung und führt das Skript fort; im zweiten springt der Ablauf in den `catch`-Block. Ein häufiger Nebeneffekt der falschen Form: Das Skript arbeitet mit einem leeren Ergebnis weiter und richtet Folgeschäden an.

### Symptom: Ausgabe ist abgeschnitten oder komisch formatiert.

Wer Objekte mit `Format-Table` oder `Format-List` formatiert und das Ergebnis weiter in die Pipeline gibt, erhält Formatierungsobjekte statt der Daten. Die Formatbefehle gehören ans Ende der Pipeline. Zusätzlich kürzt die Konsole breite Tabellen auf die Fensterbreite; für Dateiausgabe nutzt man `Out-String -Width` oder besser strukturierte Formate wie CSV und JSON.

### Symptom: Zahlen und Datumsangaben sind falsch.

Kulturabhängige Umwandlung ist ein Dauerbrenner. Der Text „1,5“ wird in einer deutschen Kultur anders gelesen als in einer englischen; Datumsangaben wie „03.04.2026“ können als 3. April oder 4. März verstanden werden. Im Test erschien für eine Zeitangabe unter der Standardumwandlung von `ConvertFrom-Json` das englische Format `10/25/2026 08:00:00`. Schutz: Datum und Zahlen intern invariant behandeln (ISO 8601, Punkt als Dezimaltrenner), nur für die Anzeige in die Landeskultur umwandeln.

### Symptom: Modul oder Befehl nicht gefunden.

Ursachen sind ein Modul, das nicht installiert ist, ein falscher Modulpfad für das Dienstkonto, ein Modul, das nur in Windows PowerShell 5.1 läuft, oder ein Tippfehler. Der Test lieferte für einen unbekannten Befehl die Kennung `CommandNotFoundException`. Erste Prüfung: `Get-Module -ListAvailable Name` und der Modulpfad aus dem Diagnosebericht.

### Symptom: Das Skript hängt.

Meist wartet es auf eine Eingabe, die niemand geben kann (eine Bestätigungsabfrage, ein `Read-Host`, eine Kennwortabfrage), oder auf eine Netzwerkverbindung ohne Zeitlimit. Lösung: Im automatisierten Betrieb nie interaktive Abfragen zulassen, `-Confirm:$false` gezielt setzen, bei Netzwerkaufrufen immer Zeitlimits (`-TimeoutSec`) angeben und für den Gesamtlauf eine Überwachung des Planers einrichten, die den Prozess nach einer Höchstdauer beendet.

```powershell
# FALSCH: wartet im Planer ewig auf eine Antwort
$antwort = Read-Host 'Fortfahren? (j/n)'

# RICHTIG: Parameter statt Abfrage, Zeitlimit beim Webaufruf
param([switch] $Bestaetigt)
if (-not $Bestaetigt) { throw 'Ohne -Bestaetigt wird nichts geaendert.' }
Invoke-RestMethod -Uri $url -TimeoutSec 30
```

Die falsche Zeile blockiert einen unbeaufsichtigten Lauf auf Dauer. Die richtige Fassung verlagert die Entscheidung in einen Schalter und lässt das Skript sofort mit einer verständlichen Meldung scheitern, wenn er fehlt.

### Symptom: Nach der Umstellung auf PowerShell 7 geht etwas nicht.

Ursachen sind Cmdlets, die es nur in Windows PowerShell gibt (zum Beispiel `Get-EventLog`, `Get-WmiObject`), Module ohne Unterstützung, geänderte Standardkodierung (PowerShell 7 schreibt Dateien standardmäßig als UTF-8 ohne BOM, Windows PowerShell 5.1 nutzt je nach Cmdlet andere Kodierungen) und geänderte Fehlerausgabe. Vertiefung: Modul 12, Lerneinheit 12.2.

> **Ergänzung aus der Nachrecherche**
>
> **Fehler in der Pipeline von Azure DevOps.** Laut der abgerufenen Dokumentation zum PowerShell-Schritt setzt die Pipeline vor dem Skript standardmäßig `$ErrorActionPreference = 'stop'`; dadurch bricht ein nicht abbrechender Fehler, der lokal nur eine rote Zeile erzeugt hätte, den Schritt ab. Ein Skript, das lokal „funktioniert“, kann in der Pipeline also scheitern. Mit der Eingabe `errorActionPreference: continue` verhält sich der Schritt wie lokal; empfehlenswert ist das nicht, besser ist es, das Skript so zu schreiben, dass es mit `Stop` sauber läuft.

> **Merke**
>
> Wiederkehrende Bilder: anderes Konto oder Verzeichnis im Planer, nicht abbrechende Fehler, Kultur bei Zahlen und Datum, fehlendes Modul oder falsche Edition, hängende Abfragen, Unterschiede zwischen Windows PowerShell und PowerShell 7. Für jedes gibt es eine schnelle erste Prüfung; der Diagnosebericht beantwortet die meisten.

> **Praxisbeispiel**
>
> Ein Admin-Team führt eine Liste „Top 10 Störungen“ mit Symptom, Ursache und erster Prüfung. Neue Kolleginnen und Kollegen lösen damit die Hälfte aller Tickets ohne Rückfrage; die Liste wird nach jedem Vorfall um das neue Bild ergänzt.

## 9.8 Runbooks und Incident Response

Ein *Runbook* ist eine Betriebsanleitung für eine wiederkehrende Aufgabe oder Störung: Was ist zu tun, in welcher Reihenfolge, von wem, und woran erkennt man, dass es funktioniert hat? Der Begriff stammt aus dem Rechenzentrum und bezeichnet heute sowohl Textdokumente als auch automatisierte Abläufe (zum Beispiel die Runbooks in Azure Automation, siehe Modul 10). Ein gutes Runbook ist so geschrieben, dass es jemand um drei Uhr nachts ohne Vorwissen abarbeiten kann.

Ein *Incident* ist eine ungeplante Störung oder Beeinträchtigung eines Dienstes. *Incident Response* bezeichnet den geordneten Umgang damit. In jedem Betriebsmodell gleichen sich die Grundphasen: Die Störung wird *erkannt* (Alarm, Meldung), *eingeordnet* (wie schwer, wer ist betroffen), *eingedämmt* (der Schaden soll nicht größer werden), *behoben*, *geprüft* (läuft alles wieder?) und *aufgearbeitet* (was lernen wir?). Skripte spielen dabei an drei Stellen eine Rolle: Sie liefern die Beweise (Protokolle, Diagnosebericht), sie setzen die Standardmaßnahmen um (Neustart eines Dienstes, Wiederholung eines Laufs) und sie erzeugen den Bericht.

Was gehört in ein Runbook? Der Zweck und der Auslöser („Alarm: Sicherung seit mehr als 26 Stunden nicht erfolgreich“). Die Voraussetzungen (Konto, Rechte, Zugang). Die Schritte mit genauen Befehlen und der erwarteten Ausgabe. Ein Prüfschritt, der das Ergebnis bestätigt. Der Weg zurück (Rollback), falls der Eingriff die Lage verschlimmert. Die Eskalation (wen rufe ich an, wenn Schritt 4 scheitert, und ab welcher Dauer). Und der Verantwortliche samt Datum der letzten Prüfung, denn veraltete Runbooks sind gefährlicher als keine.

Eng verwandt ist die Idee der *idempotenten* Abläufe, die Sie aus Modul 5 kennen: Ein Runbook-Skript sollte sich gefahrlos wiederholen lassen. Wenn der Operator unsicher ist, ob Schritt 3 schon gelaufen ist, startet er das Skript einfach noch einmal, und es stellt nur her, was fehlt. Das senkt die Fehlerquote in Stresssituationen erheblich.

Aus den strukturierten Protokollen aus 9.1 lässt sich ein Vorfallbericht fast automatisch erzeugen. Die folgende Funktion liest die Datei, filtert die Zeilen eines Laufs und erstellt einen Bericht im Markdown-Format mit den Fakten und einer Vorlage für die Teile, die ein Mensch ausfüllen muss (Auswirkung, Ursache, Maßnahmen).

```powershell
function New-IncidentBericht {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $LogPfad, [Parameter(Mandatory)] [string] $RunId)
    $e = Get-Content -LiteralPath $LogPfad | ConvertFrom-Json -DateKind String | Where-Object runId -eq $RunId
    if (-not $e) { throw "Keine Einträge für Lauf $RunId gefunden." }
    $z = $e.zeit | ForEach-Object { [datetime]::Parse($_, [Globalization.CultureInfo]::InvariantCulture, 'RoundtripKind') }
    $start = ($z | Measure-Object -Minimum).Minimum
    $ende  = ($z | Measure-Object -Maximum).Maximum
    $fehler = @($e | Where-Object level -eq 'ERROR')
    $warn   = @($e | Where-Object level -eq 'WARN')
    $zeilen = @(
        "# Incident-Bericht Lauf $RunId", '',
        "- Rechner: $($e[0].host)", "- Benutzer: $($e[0].user)",
        "- Beginn (UTC): $($start.ToString('u'))",
        "- Dauer: $([math]::Round(($ende - $start).TotalSeconds, 1)) s",
        "- Einträge: $(@($e).Count), Warnungen: $($warn.Count), Fehler: $($fehler.Count)", '',
        '## Fehler'
    )
    $zeilen += if ($fehler) { $fehler | ForEach-Object { "- $($_.zeit) [$($_.aktion)] $($_.text)" } } else { '- keine' }
    $zeilen += '', '## Auswirkung', '(auszufüllen)', '', '## Ursache', '(auszufüllen)', '', '## Maßnahmen', '(auszufüllen)'
    $zeilen -join "`n"
}
```

Die Funktion liest die Datei mit `ConvertFrom-Json -DateKind String`, damit die Zeitstempel als Text erhalten bleiben und nicht unbeabsichtigt in die Landeszeit umgewandelt werden. Sie filtert nach der Run-ID und bricht mit einer klaren Meldung ab, wenn kein Eintrag passt (im Test: „Keine Einträge für Lauf nix gefunden.“). Dann wandelt sie die Zeitstempel mit der invarianten Kultur und der Option `RoundtripKind` in `DateTime`-Werte um, bestimmt kleinsten und größten Wert und berechnet daraus die Dauer. Anschließend zählt sie Warnungen und Fehler, baut die Berichtszeilen auf und fügt die Fehlerliste ein. Der Testlauf mit einem Beispielprotokoll von drei Zeilen lieferte als Kopf: Rechner srv01, Benutzer svc, Beginn 2026-10-25 01:00:00Z, Dauer 9 s, 3 Einträge, eine Warnung und ein Fehler; darunter standen der Fehlertext „Ziel nicht erreichbar“ und die drei leeren Abschnitte zum Ausfüllen.

> **Ergänzung aus der Nachrecherche**
>
> **Blameless Post-Mortem.** In der Betriebspraxis hat sich die „schuldfreie Nachbetrachtung“ durchgesetzt: Nach einem Vorfall wird nicht gefragt, wer schuld ist, sondern was im System den Fehler möglich oder wahrscheinlich gemacht hat und was geändert wird. Das Ergebnis sind konkrete Maßnahmen mit Verantwortlichen und Terminen: ein neuer Alarm, ein ergänztes Runbook, ein automatischer Test. Diese Praxis stammt aus dem Betriebswissen der Branche (zum Beispiel aus Veröffentlichungen zu Site Reliability Engineering) und ist hier ohne Quellenabruf wiedergegeben.

```powershell
# FALSCH: Runbook als Prosa ohne Pruefschritt und Rueckweg
# "Dienst neu starten, danach sollte es gehen."

# RICHTIG: Schritt, Pruefung, Rueckweg, Eskalation
# 1. Dienst-Status:      Get-Service Spooler
# 2. Neustart:           Restart-Service Spooler
# 3. Pruefung:           (Get-Service Spooler).Status -eq 'Running'
# 4. Wenn nicht Running: Eskalation an Rufbereitschaft Windows (Tel. im Dienstplan)
```

Das falsche Beispiel sagt nicht, woran man Erfolg erkennt und was man tut, wenn der Neustart nicht hilft. Das richtige ist kurz, aber vollständig; es lässt sich zudem fast unverändert in ein Skript übernehmen. Die Befehle dieses Beispiels sind ein Windows-Beispiel und wurden nicht ausgeführt.

> **Merke**
>
> Ein Runbook nennt Auslöser, Voraussetzungen, Schritte mit erwarteter Ausgabe, einen Prüfschritt, den Rückweg und die Eskalation, dazu Verantwortlichen und Prüfdatum. Incident Response folgt den Phasen erkennen, einordnen, eindämmen, beheben, prüfen, aufarbeiten. Skripte liefern Beweise, setzen Standardmaßnahmen um und erzeugen den Bericht.

> **Praxisbeispiel**
>
> Das Team verknüpft jeden Alarm mit dem passenden Runbook: Der Alarmtext enthält den Link. Der Bereitschaftsdienst öffnet bei einem Alarm um 03:12 Uhr das Runbook, führt vier Schritte aus, prüft das Ergebnis und startet am Morgen den Incident-Bericht-Skript, dessen Gerüst er nur noch mit Ursache und Maßnahmen ergänzt.

## 9.9 Übungsaufgaben zu Modul 09

Die Aufgaben steigen in der Schwierigkeit an. Die Aufgaben 4, 5 und 7 sind praktisch (Code schreiben). Die Lösungen mit Erklärung folgen im nächsten Abschnitt.

**Aufgabe 1.** Nennen Sie vier Felder, die ein strukturierter Protokolleintrag mindestens enthalten sollte, und begründen Sie, warum der Zeitstempel in UTC und nicht in Ortszeit geschrieben wird. Wozu dient die Run-ID?

**Aufgabe 2.** Eine Funktion enthält die Zeilen `'Zähle ...'` und `42` (beide landen unverändert in der Ausgabe). Was enthält `$n` nach `$n = Get-Anzahl`? Wie muss die Meldung geschrieben werden, damit `$n` nur 42 enthält? In welchen Strom schreibt `Write-Host`?

**Aufgabe 3.** Ein Skript endet mit `exit 20`. Welcher Exit Code kommt beim Aufruf mit `pwsh -File skript.ps1` an, welcher bei `pwsh -Command "& ./skript.ps1"`, und wie reicht man den genauen Wert mit `-Command` durch?

**Aufgabe 4.** **Praxis:** Schreiben Sie eine Funktion `Get-LogZusammenfassung`, die eine JSON-Lines-Protokolldatei (Felder wie in 9.1) liest und je `runId` eine Zeile ausgibt: Anzahl der Einträge, Anzahl Warnungen, Anzahl Fehler und einen Status (`OK`, `WARNUNG` oder `FEHLER`).

**Aufgabe 5.** **Praxis:** Schreiben Sie ein Skript, das einen Ordner als Parameter erhält. Es soll mit Exit Code 20 enden, wenn der Ordner nicht existiert, mit 10, wenn er keine Dateien enthält, und sonst mit 0 und der Ausgabe der Dateianzahl.

**Aufgabe 6.** Warum werden Zahlen für Metrikdateien mit der invarianten Kultur formatiert, und warum schreibt man die Datei zuerst unter einem temporären Namen? Welchen Zweck hat die Metrik „letzter Erfolg“?

**Aufgabe 7.** **Praxis:** Schreiben Sie ein Prüfskript für die Metrikdatei aus 9.5: Es liest den Wert `skript_letzter_erfolg_unix`, berechnet das Alter in Stunden und beendet sich mit 0 (jünger als 26 h), 10 (älter als 26 h) oder 20 (älter als 48 h bzw. Datei oder Metrik fehlt).

**Aufgabe 8.** Ein Skript läuft interaktiv fehlerfrei, im Windows-Aufgabenplaner scheitert es. Nennen Sie vier mögliche Ursachen und beschreiben Sie, wie Sie mit der Methode aus 9.6 vorgehen. Wie fangen Sie im Skript eine Ausnahme „Zugriff verweigert“ unabhängig von der Sprache der Fehlermeldung ab?

## 9.10 Lösungen zu Modul 09

### Aufgabe 1.

Sinnvolle Felder sind Zeitstempel, Level, Run-ID, Rechnername, Benutzer, Aktion, Text und Zusatzdaten (vier davon genügen). UTC vermeidet Mehrdeutigkeiten bei der Zeitumstellung (eine Stunde kommt im Herbst doppelt vor) und macht Protokolle von Rechnern in verschiedenen Zeitzonen vergleichbar. Die Run-ID ist eine kurze Kennung pro Lauf, die in jedem Eintrag steht: Man kann alle Zeilen eines Laufs herausfiltern, auch wenn mehrere Läufe gleichzeitig in dieselbe Datei schreiben.

### Aufgabe 2.

`$n` ist ein Feld mit zwei Elementen (`'Zähle ...'` und 42), denn beide Werte gelangen in den Erfolgsstrom. Mit `Write-Verbose 'Zähle ...'` bleibt nur 42 übrig; die Meldung erscheint dann nur bei `-Verbose`. `Write-Host` schreibt seit PowerShell 5.0 in den Informationsstrom (Strom 6); im Test erschien es bei der Umleitung `6>&1` als `InformationRecord`.

### Aufgabe 3.

Mit `-File` kommt 20 an. Mit `-Command "& ./skript.ps1"` kam im Test nur 1 an, weil bei `-Command` nur zwischen erfolgreich (0) und fehlgeschlagen (1) unterschieden wird. Den genauen Wert reicht man mit `pwsh -Command "./skript.ps1; exit $LASTEXITCODE"` durch (im Test 10 für ein Skript mit `exit 10`).

### Aufgabe 4.

```powershell
function Get-LogZusammenfassung {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $Pfad)
    Get-Content -LiteralPath $Pfad |
        ConvertFrom-Json -DateKind String |
        Group-Object runId |
        ForEach-Object {
            $stufen = $_.Group.level
            [pscustomobject]@{
                RunId     = $_.Name
                Eintraege = $_.Count
                Warnungen = @($stufen | Where-Object { $_ -eq 'WARN' }).Count
                Fehler    = @($stufen | Where-Object { $_ -eq 'ERROR' }).Count
                Status    = if ($stufen -contains 'ERROR') { 'FEHLER' }
                            elseif ($stufen -contains 'WARN') { 'WARNUNG' }
                            else { 'OK' }
            }
        }
}
```

Jede Zeile wird mit `ConvertFrom-Json` zu einem Objekt, `Group-Object runId` bündelt die Zeilen eines Laufs, und je Gruppe entsteht ein Ergebnisobjekt. Die Klammern `@(...)` sorgen dafür, dass `.Count` auch bei null oder einem Treffer funktioniert. Der Status richtet sich nach der höchsten Stufe: Fehler vor Warnung vor OK. Im Test mit einem Beispielprotokoll ergab sich für einen Lauf mit einer Warnung und einem Fehler der Status `FEHLER`, für einen Lauf nur mit INFO-Zeilen `OK`.

### Aufgabe 5.

```powershell
param([Parameter(Mandatory)] [string] $Ordner)
$Codes = @{ Ok = 0; Warnung = 10; Fehler = 20 }
if (-not (Test-Path -LiteralPath $Ordner -PathType Container)) {
    Write-Error "Ordner '$Ordner' existiert nicht."
    exit $Codes.Fehler
}
$anzahl = @(Get-ChildItem -LiteralPath $Ordner -File).Count
if ($anzahl -eq 0) {
    Write-Warning "Ordner '$Ordner' enthält keine Dateien."
    exit $Codes.Warnung
}
Write-Output "$anzahl Dateien gefunden."
exit $Codes.Ok
```

`Test-Path -PathType Container` prüft, dass der Pfad ein Ordner ist, nicht eine Datei. `@(...)` macht das Ergebnis zu einem Feld, damit `.Count` auch bei einer einzelnen Datei stimmt. Im Test lieferten ein Ordner mit zwei Dateien den Code 0 und die Ausgabe „2 Dateien gefunden.“, ein leerer Ordner den Code 10 und ein nicht vorhandener Ordner den Code 20. Aufgerufen wurde mit `pwsh -File`.

### Aufgabe 6.

Ohne die invariante Kultur schreibt `ToString()` unter einer deutschen Einstellung „1,5“, das Prometheus-Textformat verlangt aber den Punkt. Die temporäre Datei mit anschließendem `Move-Item` verhindert, dass ein einlesender Agent eine halb geschriebene Datei sieht. Die Metrik „letzter Erfolg“ ermöglicht einen Alarm, wenn das Skript gar nicht mehr startet: Ein Skript, das nicht läuft, kann auch keinen Fehler melden, aber das Alter des letzten Erfolgs wächst.

### Aufgabe 7.

```powershell
param([Parameter(Mandatory)] [string] $MetrikDatei, [int] $WarnStunden = 26, [int] $FehlerStunden = 48)
if (-not (Test-Path -LiteralPath $MetrikDatei)) { Write-Error 'Metrikdatei fehlt.'; exit 20 }
$zeile = Select-String -LiteralPath $MetrikDatei -Pattern '^skript_letzter_erfolg_unix\{[^}]*\}\s+(\d+)' | Select-Object -First 1
if (-not $zeile) { Write-Error 'Metrik letzter_erfolg_unix nicht gefunden.'; exit 20 }
$unix  = [long]$zeile.Matches[0].Groups[1].Value
$alter = [DateTimeOffset]::UtcNow - [DateTimeOffset]::FromUnixTimeSeconds($unix)
Write-Output ('Letzter Erfolg vor {0:N1} Stunden' -f $alter.TotalHours)
if ($alter.TotalHours -gt $FehlerStunden) { exit 20 }
if ($alter.TotalHours -gt $WarnStunden)   { exit 10 }
exit 0
```

Der reguläre Ausdruck sucht die Zeile mit dem Metriknamen, überspringt die Beschriftung in geschweiften Klammern und fängt die Zahl ein. `FromUnixTimeSeconds` wandelt sie in einen Zeitpunkt, die Differenz zu jetzt ist das Alter. Die Prüfung der größeren Schwelle steht zuerst, sonst würde 60 Stunden schon bei der Warnung enden. Im Test ergaben Dateien mit einem Alter von 1, 30 und 60 Stunden die Codes 0, 10 und 20.

### Aufgabe 8.

Mögliche Ursachen: anderes Benutzerkonto (fehlende Rechte oder Zugriff auf Netzfreigaben), anderes Arbeitsverzeichnis (relative Pfade), nicht geladenes Profil (fehlende Aliase und Funktionen), andere PowerShell-Edition oder -Version, andere Umgebungsvariablen oder Modulpfade, interaktive Abfragen. Vorgehen: Problem beschreiben (erwartet gegen tatsächlich, was hat sich geändert?), im Planerlauf den Diagnosebericht aus 9.6 schreiben und mit dem interaktiven Ergebnis vergleichen, durch Halbieren eingrenzen, eine Änderung pro Versuch. Abfangen: `catch [System.UnauthorizedAccessException] { ... }` prüft den Ausnahmetyp statt des Meldungstexts. Dieser Teil wurde nicht ausgeführt, weil der Arbeitsbereich mit Administratorrechten lief.
