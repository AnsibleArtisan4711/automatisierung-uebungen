# Modul 08 - Security, Hardening und Secret Management

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (8.1 bis 8.9; 8.10 Übungsaufgaben, 8.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 12 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 1 (einzelne Lerneinheiten: 1 bzw. 2) |

> **Merke**
>
> Übersichtsblatt (12 h) und Detailblatt (9 h) weichen um drei Stunden voneinander ab. Dieses Skript folgt dem Detailblatt. Die Zusatzzeit eignet sich für die praktischen Übungen am Modulende, vor allem in einer Windows-Testumgebung.

### Lernziele.

Nach diesem Modul können Sie PowerShell sicher betreiben und Missbrauchsrisiken verringern. Sie ordnen die Execution Policy richtig ein, signieren Skripte und prüfen Signaturen, kennen Constrained Language Mode und Application Control, setzen das Prinzip der geringsten Rechte um, verwalten Geheimnisse und Anmeldedaten ohne Klartext, prüfen Eingaben gegen Einschleusungen, verstehen die Protokollierungsfunktionen von PowerShell samt Datenschutzfolgen und beurteilen die Herkunft von Modulen. Das Ziel bleibt wiederholbare Automation für Windows, Cloud und CI/CD, die auch einem Angreifer standhält.

| **Nr.** | **Lerneinheit**               | **Dauer** | **Prio** |
|:--------|:------------------------------|:----------|:---------|
| 8.1     | Execution Policy und Realität | 1 h       | 1        |
| 8.2     | Script Signing                | 1 h       | 1        |
| 8.3     | Constrained Language Mode     | 1 h       | 1        |
| 8.4     | Least Privilege               | 1 h       | 1        |
| 8.5     | Secrets Management            | 1 h       | 1        |
| 8.6     | Credential Handling           | 1 h       | 1        |
| 8.7     | Input Validation Security     | 1 h       | 2        |
| 8.8     | PowerShell Logging Security   | 1 h       | 2        |
| 8.9     | Supply Chain Security         | 1 h       | 2        |
|         | **Summe**                     | **9 h**   |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde.** Was sich plattformübergreifend prüfen ließ (Sprachmodi, SecureString, Anmeldeobjekte, Eingabeprüfung, Pfadprüfung, Versionsbindung, Hashwerte), wurde mit PowerShell 7.6.6 unter Linux ausgeführt; die genannten Meldungen stammen aus diesen Läufen. Windows-spezifische Funktionen (Execution Policy mit Durchsetzung, Authenticode-Signierung, Zertifikatspeicher, AppLocker und App Control, Ereignisprotokolle, Gruppenrichtlinien, `#Requires -RunAsAdministrator`, JEA) und Funktionen, die Zugriff auf die PowerShell Gallery brauchen (SecretManagement, SecretStore), konnten im Arbeitsbereich nicht ausgeführt werden. Diese Beispiele sind auf Syntax geprüft und mit der Microsoft-Dokumentation abgeglichen (siehe Quellen); führen Sie sie vor dem Einsatz in einer Testumgebung aus. Die Hinweise zu Datenschutz sind allgemeiner Art und keine Rechtsberatung.

## 8.1 Execution Policy und Realität

Fast jede Einführung in PowerShell stößt irgendwann auf die Fehlermeldung „Die Datei kann nicht geladen werden, da die Ausführung von Skripts auf diesem System deaktiviert ist“. Die Ursache ist die *Execution Policy*, eine Einstellung unter Windows, die festlegt, unter welchen Bedingungen PowerShell Skripte, Moduldateien und Konfigurationsdateien lädt. Wegen dieser Fehlermeldung glauben viele, die Execution Policy sei ein Sicherheitsmechanismus, der Angreifer aufhält. Das ist ein verbreitetes Missverständnis, und genau das sollen Sie in dieser Lerneinheit richtig einordnen.

Die Microsoft-Dokumentation sagt es ausdrücklich: Die Execution Policy ist *keine Sicherheitsgrenze*, sondern eine Maßnahme zur gestaffelten Verteidigung (defense in depth). Sie soll Benutzer daran hindern, versehentlich Skripte auszuführen, die sie nicht ausführen wollten. Sie verhindert nicht, dass ein entschlossener Benutzer oder ein Angreifer den Inhalt eines Skripts einfach in die Konsole tippt, per `Invoke-Expression` ausführt oder mit `pwsh -ExecutionPolicy Bypass` einen neuen Prozess mit anderer Einstellung startet. Als sicherheitsrelevante Merkmale nennt Microsoft dagegen die Kombination aus Application Control und Constrained Language Mode (Lerneinheit 8.3).

Ein weiterer wichtiger Punkt ist die Plattform: Die Durchsetzung findet *nur unter Windows* statt. Auf Linux und macOS steht die Richtlinie fest auf `Unrestricted` und lässt sich nicht ändern; `Set-ExecutionPolicy` existiert, meldet aber, dass der Vorgang nicht unterstützt wird. Im Test unter Linux gab `Get-ExecutionPolicy` den Wert `Unrestricted` zurück, `Get-ExecutionPolicy -List` zeigte in allen fünf Bereichen `Unrestricted`, und `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` scheiterte mit „Operation is not supported on this platform.“ Die Dokumentation ergänzt, dass `Unrestricted` dort tatsächlich wie `Bypass` wirkt, weil die Windows-Sicherheitszonen fehlen.

### Die Richtlinien.

Unter Windows kennt PowerShell folgende Werte. `Restricted` erlaubt einzelne Befehle, aber keine Skripte; sie ist der Standard auf Windows-Clients, wenn keine andere Richtlinie gesetzt ist. `RemoteSigned` ist der Standard auf Windows-Servern: Lokal erstellte Skripte laufen ohne Signatur, aus dem Internet heruntergeladene Skripte brauchen die Signatur eines vertrauenswürdigen Herausgebers. `AllSigned` verlangt für alle Skripte eine Signatur, auch für selbst geschriebene. `Unrestricted` erlaubt alle Skripte und warnt bei Skripten, die nicht aus dem lokalen Intranet stammen. `Bypass` blockiert nichts und warnt nicht; es ist für Fälle gedacht, in denen PowerShell in eine größere Anwendung mit eigenem Sicherheitsmodell eingebettet ist. `Undefined` bedeutet, dass in diesem Bereich nichts gesetzt ist. Sind alle Bereiche undefiniert, gilt `Restricted` (Windows-Client) beziehungsweise `RemoteSigned` (Windows Server).

### Bereiche und Rangfolge.

Eine Richtlinie lässt sich für fünf Bereiche (*Scopes*) setzen. `MachinePolicy` und `UserPolicy` kommen aus Gruppenrichtlinien und haben die höchste Priorität; sie lassen sich in PowerShell nicht ändern. Danach folgen `Process` (nur für den laufenden Prozess, gespeichert in der Umgebungsvariable `$Env:PSExecutionPolicyPreference`), `CurrentUser` (gespeichert in der benutzerbezogenen Datei `powershell.config.json`) und `LocalMachine` (für alle Benutzer, gespeichert in `$PSHOME/powershell.config.json`). Die wirksame Richtlinie ergibt sich aus dem ersten gesetzten Bereich in dieser Reihenfolge. Ein erfolgreich ausgeführtes `Set-ExecutionPolicy` kann deshalb wirkungslos bleiben, wenn ein Bereich mit höherer Priorität etwas anderes festlegt.

Das folgende Beispiel zeigt die wichtigsten Befehle. Es ist für Windows geschrieben und wurde unter Linux nur teilweise getestet (siehe oben).

```powershell
# Wirksame Richtlinie und alle Bereiche in Rangfolge anzeigen
Get-ExecutionPolicy
Get-ExecutionPolicy -List

# Fuer den eigenen Benutzer setzen (kein Administrator noetig)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Nur fuer den aktuellen Prozess (verschwindet mit dem Prozess)
Set-ExecutionPolicy -ExecutionPolicy AllSigned -Scope Process

# Fuer einen neuen Prozess beim Start festlegen
pwsh.exe -NoProfile -ExecutionPolicy AllSigned -File .\skript.ps1

# Einen Bereich wieder entfernen
Set-ExecutionPolicy -ExecutionPolicy Undefined -Scope CurrentUser

# Heruntergeladene Datei freigeben, nachdem man sie geprueft hat
Get-Item .\skript.ps1 -Stream Zone.Identifier
Unblock-File -Path .\skript.ps1
```

Die ersten beiden Befehle zeigen den Ist-Zustand; `-List` gibt die Bereiche in der Reihenfolge ihrer Priorität aus, sodass man erkennt, welcher Bereich die wirksame Einstellung liefert. Das Setzen für `CurrentUser` ist der übliche Weg auf Entwicklerrechnern, weil es keine Administratorrechte braucht; `LocalMachine` braucht sie. Die Einstellung für `Process` ist der saubere Weg für Skripte, die nur temporär andere Regeln benötigen. Beim Start mit `-ExecutionPolicy` gilt dasselbe: Der Wert landet nur im Prozess, nicht in der Konfigurationsdatei. `Get-Item -Stream Zone.Identifier` zeigt den alternativen Datenstrom, den Windows an aus dem Internet heruntergeladene Dateien hängt (*Mark of the Web*); `Unblock-File` entfernt ihn, nachdem man den Code gelesen hat. Beachten Sie, dass nicht jeder Download diese Markierung setzt: Laut Dokumentation tun das `curl.exe`, `Invoke-WebRequest` und `Invoke-RestMethod` nicht.

> **Ergänzung aus der Nachrecherche**
>
> **Gruppenrichtlinie und Server Core (laut about_Execution_Policies).** Die Einstellung „Turn on Script Execution“ in den Gruppenrichtlinien überschreibt die in PowerShell gesetzten Richtlinien in allen Bereichen. Ihre drei Optionen entsprechen `Unrestricted` („Allow all scripts“), `RemoteSigned` („Allow local scripts and remote signed scripts“) und `AllSigned` („Allow only signed scripts“); „deaktiviert“ entspricht `Restricted`. Auf Windows Server Core und Nano Server kann die Zonenprüfung für `RemoteSigned` fehlschlagen, weil dort die Windows-Shell fehlt; die Meldung lautet „AuthorizationManager check failed“. `Bypass` und `AllSigned` brauchen keine Zonenprüfung und vermeiden das Problem.

Warum bleibt die Execution Policy trotzdem sinnvoll? Sie verhindert Unfälle: den Doppelklick auf ein heruntergeladenes Skript, den automatischen Start eines Skripts aus einem Mailanhang, die versehentliche Ausführung von Skripten aus unbekannter Quelle. In Verbindung mit Signaturen (Lerneinheit 8.2) lässt sich außerdem der Schutz vor nachträglichen Änderungen aufbauen. Richtig eingesetzt ist sie ein Bestandteil einer Verteidigung aus mehreren Schichten, aber nie die einzige Schicht.

> **Merke**
>
> Die Execution Policy ist keine Sicherheitsgrenze, sondern Schutz vor Versehen. Durchgesetzt wird sie nur unter Windows; Linux und macOS laufen immer wie `Bypass`. Rangfolge: Gruppenrichtlinie, `Process`, `CurrentUser`, `LocalMachine`. Echten Schutz bieten Application Control und Constrained Language Mode.

```powershell
# FALSCH: Richtlinie dauerhaft auf Bypass stellen, damit "alles laeuft"
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force

# RICHTIG: Richtlinie belassen, Skript pruefen und gezielt freigeben
Get-Content .\skript.ps1          # lesen und verstehen
Unblock-File -Path .\skript.ps1   # erst danach freigeben
```

Im falschen Beispiel entfällt auch der Schutz vor Versehen für alle Benutzer des Rechners, und das Problem (ein nicht geprüftes Skript) bleibt bestehen. Im richtigen Beispiel wird nur diese eine Datei freigegeben, nachdem ein Mensch sie angesehen hat.

> **Praxisbeispiel**
>
> Ein Sicherheitsteam prüft, warum auf einem Server ein unbekanntes Skript lief, obwohl die Richtlinie `AllSigned` gesetzt war. Es stellt fest, dass der Angreifer den Code einfach über die Pipeline an `Invoke-Expression` übergeben hat; die Execution Policy greift bei Skriptdateien, nicht bei Text, der zur Laufzeit ausgeführt wird. Daraufhin stellt das Team auf Application Control mit Constrained Language Mode um und aktiviert Script Block Logging (Lerneinheit 8.8).

## 8.2 Script Signing

Eine *digitale Signatur* beantwortet zwei Fragen: Von wem stammt dieses Skript, und ist es seit der Signierung unverändert? Technisch berechnet der Signierer einen Hashwert über den Inhalt und verschlüsselt ihn mit seinem privaten Schlüssel. Der Empfänger prüft mit dem öffentlichen Schlüssel aus dem Zertifikat des Signierers, ob der Hash zum Inhalt passt, und prüft, ob dem Zertifikat vertraut wird. PowerShell verwendet für Skripte den Standard *Authenticode*. Die Signatur steht als Kommentarblock am Ende der Datei, der mit `# SIG # Begin signature block` beginnt und mit `# SIG # End signature block` endet.

PowerShell prüft Signaturen für `.ps1`, `.psm1`, `.psd1`, `.ps1xml`, `.cdxml` und `.xaml` (laut about_Signing), allerdings nur unter Windows und nur, wenn die Execution Policy es verlangt (`AllSigned`) oder die Datei aus dem Internet stammt (`RemoteSigned`). Zum Signieren braucht man ein *Codesignatur-Zertifikat*. Es gibt zwei Quellen: Zertifikate einer Zertifizierungsstelle, die gegen Gebühr die Identität prüft und deren Zertifikate auf anderen Rechnern anerkannt werden, und selbst erstellte Zertifikate, die nur auf dem eigenen Rechner (oder nach Verteilung des Zertifikats in die Vertrauensstellungen) funktionieren. Die Dokumentation sagt deutlich, dass selbst signierte Zertifikate nur für Tests taugen. In Organisationen betreibt man meist eine interne Zertifizierungsstelle (PKI) und verteilt deren Stammzertifikat per Gruppenrichtlinie.

Das folgende Beispiel zeigt den gesamten Ablauf für eine Testumgebung unter Windows: Zertifikat erstellen, Skript signieren, Signatur prüfen, Manipulation erkennen.

```powershell
# 1. Selbst signiertes Testzertifikat im Benutzerspeicher anlegen
$params = @{
    Subject           = 'CN=PowerShell Code Signing Test'
    Type              = 'CodeSigning'
    CertStoreLocation = 'Cert:\CurrentUser\My'
    HashAlgorithm     = 'sha256'
}
$cert = New-SelfSignedCertificate @params

# 2. Skript signieren (mit Zeitstempel, falls eine Zeitstempel-URL vorhanden ist)
Set-AuthenticodeSignature -FilePath .\Sichere-Kopie.ps1 -Certificate $cert

# 3. Signatur pruefen
$sig = Get-AuthenticodeSignature -FilePath .\Sichere-Kopie.ps1
$sig.Status
$sig.SignerCertificate.Subject

# 4. Manipulation erkennen
Add-Content -Path .\Sichere-Kopie.ps1 -Value '# geaendert'
(Get-AuthenticodeSignature -FilePath .\Sichere-Kopie.ps1).Status
```

Schritt 1 erzeugt mit `New-SelfSignedCertificate` (Modul PKI, nur Windows) ein Zertifikat für den Zweck `CodeSigning` im persönlichen Zertifikatspeicher; die Variable `$cert` enthält es. Schritt 2 hängt die Signatur an die Datei; `Set-AuthenticodeSignature` gibt ein Signaturobjekt zurück. Schritt 3 liest die Signatur aus der Datei: Die Eigenschaft `Status` meldet `Valid`, wenn alles stimmt. Bei einem selbst signierten Zertifikat, das nicht in den vertrauenswürdigen Stammzertifikaten steht, meldet sie dagegen `UnknownError` oder `NotTrusted`; die Signatur ist mathematisch in Ordnung, aber der Herausgeber unbekannt. Schritt 4 verändert die Datei und prüft erneut: Der Status lautet dann `HashMismatch`. Eine Datei ohne Signatur meldet `NotSigned`. Diese vier Statuswerte sollte man auswendig kennen, denn sie stehen in Überwachungsskripten, die Signaturen kontrollieren.

Im Betrieb stellt man mit `AllSigned` sicher, dass nur signierte Skripte laufen. Der Änderungsschutz ist der eigentliche Gewinn: Wer ein signiertes Skript auf einem Server ändert, bricht die Signatur, und PowerShell lehnt die Ausführung ab. Wichtig ist, dass dazu das Signaturzertifikat auf den Zielrechnern als vertrauenswürdig gelten muss (im Speicher der vertrauenswürdigen Herausgeber) und dass der private Schlüssel gut geschützt bleibt. Wer ihn stehlen kann, kann beliebigen Code signieren. Die Dokumentation rät deshalb zu einem Schutz des privaten Schlüssels mit Kennwort beim Export als `.pfx`-Datei. In Organisationen liegen die Schlüssel besser in einem Hardware-Sicherheitsmodul oder einem Cloud-Dienst für Schlüssel, und das Signieren erfolgt in der Build-Pipeline (Modul 11), nicht auf Entwicklerrechnern.

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Hinweise aus about_Signing.** Vor PowerShell 7.2 musste ein zu signierendes Skript als ASCII oder UTF-8 ohne BOM gespeichert sein; ab 7.2 funktionieren signierte Skripte mit jeder Zeichenkodierung. Ein Zeitstempel (Parameter `-TimestampServer` bei `Set-AuthenticodeSignature`) sorgt dafür, dass die Signatur auch nach Ablauf des Zertifikats gültig bleibt, solange sie bei gültigem Zertifikat angebracht wurde. Das ist wichtig, weil Codesignatur-Zertifikate oft nur ein Jahr gültig sind. Das Beispiel oben enthält keinen Zeitstempel-Server, weil dessen Adresse von Ihrer Zertifizierungsstelle abhängt. Auf Linux und macOS gibt es keine `*-AuthenticodeSignature`-Cmdlets; im Test fanden sich dort nur `Get-PfxCertificate` und `Unblock-File`.

> **Merke**
>
> Signatur beweist Herkunft und Unverändertheit. Ablauf: Zertifikat, `Set-AuthenticodeSignature`, `Get-AuthenticodeSignature`. Status: `Valid`, `NotSigned`, `HashMismatch`, `NotTrusted` bzw. `UnknownError`. Mit `AllSigned` wirkt sie als Änderungsschutz. Privaten Schlüssel schützen, produktiv in der Pipeline signieren, Zeitstempel verwenden.

```powershell
# FALSCH: Signatur nur zum Schein, Status wird nie geprueft
Set-AuthenticodeSignature -FilePath .\deploy.ps1 -Certificate $cert
.\deploy.ps1

# RICHTIG: Vor der Ausfuehrung ausdruecklich pruefen
$sig = Get-AuthenticodeSignature -FilePath .\deploy.ps1
if ($sig.Status -ne 'Valid' -or $sig.SignerCertificate.Thumbprint -ne $erwarteterThumbprint) {
    throw "Signatur ungueltig oder fremd: $($sig.Status)"
}
.\deploy.ps1
```

Im ersten Fall würde auch ein verändertes Skript ausgeführt, wenn die Execution Policy es erlaubt. Im zweiten prüft das Skript, ob die Signatur gültig ist *und* ob sie vom erwarteten Zertifikat stammt; die zweite Bedingung ist wichtig, denn Signaturen von irgendeinem fremden, aber gültigen Zertifikat nützen nichts.

> **Praxisbeispiel**
>
> Ein Unternehmen signiert alle freigegebenen Administrationsskripte in der Build-Pipeline mit einem Zertifikat der internen Zertifizierungsstelle und setzt auf den Servern `AllSigned`. Ein Mitarbeiter, der ein Skript „schnell“ auf einem Server korrigiert, stellt fest, dass es danach nicht mehr startet. Die Änderung muss über das Repository, die Prüfung und die Pipeline laufen, und genau das war beabsichtigt.

## 8.3 Constrained Language Mode

Die Sprachmodi gehören zu den wenigen Merkmalen von PowerShell, die Microsoft als echte Sicherheitsfunktion bezeichnet, allerdings nur in Verbindung mit Application Control. Der *Sprachmodus* einer Sitzung bestimmt, welche Elemente der PowerShell-Sprache in ihr zulässig sind. Es gibt vier Modi: `FullLanguage` (alles erlaubt, der Standard), `ConstrainedLanguage` (eingeschränkt), `RestrictedLanguage` (Befehle ja, Skriptblöcke nein; nur für interne Zwecke wie das Lesen von Modulmanifesten) und `NoLanguage` (nur Befehle, keine Sprache; Just Enough Administration nutzt ihn).

Der *Constrained Language Mode* (CLM) erlaubt die Grundelemente der Sprache, also Schleifen, Bedingungen, Zeichenketten und Eigenschaftszugriffe, sowie alle Cmdlets. Er verbietet jedoch Operationen, die ein Angreifer missbrauchen könnte: das Definieren neuer Typen, den Zugriff auf beliebige .NET-Typen und -Methoden, beliebige COM-Objekte und das Laden von C#-Code. Erlaubt sind nur Typen aus einer festen Liste (etwa `[string]`, `[int]`, `[datetime]`, `[hashtable]`, `[pscustomobject]`, `[regex]`, `[pscredential]`, `[securestring]` und `[xml]`). Dadurch verlieren viele Angriffstechniken, die auf .NET-Aufrufen oder dem Einbinden von Win32-Funktionen beruhen, ihre Grundlage.

Das folgende Beispiel schaltet die laufende Sitzung in CLM und zeigt, welche Dinge dann noch funktionieren. Das geht zum Ausprobieren ohne Administratorrechte, gilt aber nicht als Schutz (siehe unten).

```powershell
$ExecutionContext.SessionState.LanguageMode
$ExecutionContext.SessionState.LanguageMode = 'ConstrainedLanguage'

[System.IO.File]::ReadAllText('/etc/hostname')          # .NET-Methode
Add-Type -TypeDefinition 'public class X1 { }'          # neuen Typ definieren
New-Object -TypeName System.Net.WebClient               # nicht erlaubter Typ
[Environment]::GetEnvironmentVariable('HOME')           # statische Methode
[scriptblock]::Create('1+1')                            # Code zur Laufzeit bauen

$x = [int]'5'                                           # erlaubter Typ
'abc'.ToUpper()                                         # Methode erlaubter Typen
$h = @{ a = 1 }; $h.a
Get-ChildItem /tmp | Select-Object -First 1             # Cmdlets
Invoke-Expression '1+2'

$ExecutionContext.SessionState.LanguageMode = 'FullLanguage'
```

Im Test lieferten die ersten fünf Aufrufe nach dem Wechsel Fehlermeldungen: „Cannot invoke method. Method invocation is supported only on core types in this language mode.“ für die .NET-Methoden und `[scriptblock]::Create`, „Cannot add type. Definition of new types is not supported in this language mode.“ für `Add-Type` und „Cannot create type. Only core types are supported in this language mode.“ für `New-Object`. Dagegen funktionierten die Umwandlung in `[int]`, `'abc'.ToUpper()`, die Hashtable, das Cmdlet `Get-ChildItem` und `Invoke-Expression '1+2'` (Ergebnis 3) weiter. Der letzte Versuch, auf `FullLanguage` zurückzuschalten, scheiterte mit „Cannot set property. Property setting is supported only on core types in this language mode.“ Die Sitzung war also in CLM gefangen. Das zeigt zweierlei: CLM ist eine Einbahnstraße innerhalb einer Sitzung, und das Setzen des Modus von Hand ist nur zum Experimentieren sinnvoll. Ein Angreifer startet einfach einen neuen Prozess, der wieder im Standardmodus beginnt. Genau das stellt die Dokumentation klar: CLM ist nur dann eine Sicherheitsgrenze, wenn es durch Application Control erzwungen wird; als Einstellung in einer Sitzungskonfiguration oder von Hand gilt es nur als gestaffelte Verteidigung.

### Application Control als Grundlage.

Unter Windows erkennt PowerShell, wenn eine systemweite Application-Control-Richtlinie aktiv ist, und läuft dann automatisch im CLM. Die Richtlinie kommt von *App Control for Business* (früher Windows Defender Application Control, WDAC) oder von AppLocker. Beide bestimmen, welche Programme und Skripte laufen dürfen. Laut Dokumentation ist App Control for Business das bevorzugte System und gilt als Sicherheitsfunktion im Sinne der Servicing Criteria des Microsoft Security Response Center; AppLocker gilt als älteres System, das noch unterstützt wird, aber nicht als Sicherheitsfunktion in diesem Sinn. Im sogenannten *System Lockdown*-Modus bestimmt die Richtlinie den Sprachmodus für jede Sitzung. Skripte und Module, die die Richtlinie ausdrücklich zulässt (zum Beispiel weil sie signiert sind), laufen mit `FullLanguage`; alle anderen Skripte und Eingaben in der interaktiven Konsole laufen im CLM. So bleibt die Shell für Administratoren nutzbar, aber nicht-genehmigter Code kann keine gefährlichen .NET-Aufrufe ausführen. Ab PowerShell 7.4 zeigt der Startbanner an, dass PowerShell im eingeschränkten Modus läuft.

> **Ergänzung aus der Nachrecherche**
>
> **Wichtige Aussagen zur Einordnung (laut about_Language_Modes und „PowerShell security features“).** Erstens: Ohne aktiven Application-Control-Richtlinie breitet sich der Sprachmodus nicht zwischen Runspaces aus; neue Runspaces beginnen mit `FullLanguage`. Zweitens: Die Dokumentation nennt als Sicherheitsfunktionen ausdrücklich „System Lockdown with App Control for Business“ und „Constrained language mode with App Control for Business“; als gestaffelte Verteidigung gelten „Constrained language mode with AppLocker or configured through session configuration or by manually setting `$ExecutionContext.SessionState.LanguageMode`“, „System Lockdown with AppLocker“ und die Execution Policy. Drittens: Ab PowerShell 7.2 ist `New-Object` im `RestrictedLanguage`- und `NoLanguage`-Modus deaktiviert, wenn System Lockdown konfiguriert ist. Viertens: Ab PowerShell 7.3 kann `pwsh` mit `-ConfigurationFile` mit einer bestimmten Sitzungskonfiguration gestartet werden. Außerdem sendet PowerShell 7.3 mehr Daten an AMSI (die Windows-Schnittstelle für Virenscanner); es umfasst nun alle .NET-Methodenaufrufe.

### Grenzen und Folgen für die Praxis.

CLM verändert, wie man Skripte schreibt. Code, der .NET-Methoden aufruft, `Add-Type` verwendet oder mit COM-Objekten arbeitet, bricht im CLM ab, wenn er nicht von der Richtlinie zugelassen ist. Skripte, die für den Betrieb unter CLM gedacht sind, sollten möglichst mit Cmdlets auskommen, die erlaubten Typen verwenden und signiert sein. Man testet sie deshalb vor der Einführung in einer Sitzung mit eingeschaltetem CLM, wie im Beispiel oben (mit der Einschränkung, dass der Test nicht alle Verhaltensunterschiede im System-Lockdown abbildet). Außerdem gilt: CLM schützt vor missbräuchlichen Aufrufen *innerhalb* von PowerShell, nicht vor anderen Programmen. Es ist ein Teil eines Konzepts, das Application Control, Signaturen, Protokollierung und geringste Rechte verbindet.

> **Merke**
>
> Sprachmodi: `FullLanguage`, `ConstrainedLanguage`, `RestrictedLanguage`, `NoLanguage`. CLM verbietet beliebige .NET-Aufrufe, `Add-Type` und beliebige Typen, erlaubt Cmdlets und Grundsprache. Als Sicherheitsgrenze gilt CLM nur mit App Control for Business (System Lockdown). Von Hand gesetzt ist es eine Einbahnstraße für die Sitzung und kein Schutz.

```powershell
# FALSCH: Sicherheit durch manuelles Setzen des Sprachmodus
$ExecutionContext.SessionState.LanguageMode = 'ConstrainedLanguage'
# ... Angreifer startet einfach pwsh neu und hat wieder FullLanguage

# RICHTIG: Modus pruefen, Skript fuer CLM tauglich schreiben
if ($ExecutionContext.SessionState.LanguageMode -ne 'FullLanguage') {
    Write-Verbose "Eingeschraenkter Modus: $($ExecutionContext.SessionState.LanguageMode)"
}
# nur Cmdlets und erlaubte Typen verwenden
$datum = [datetime]'2026-10-25'
Get-ChildItem -LiteralPath $pfad | Sort-Object LastWriteTime
```

Im ersten Fall entsteht nur eine trügerische Sicherheit, denn die Einstellung gilt für diese eine Sitzung und kann von einem neuen Prozess umgangen werden. Im zweiten Fall passt sich das Skript an den Modus an und verwendet nur Elemente, die in beiden Modi funktionieren.

> **Praxisbeispiel**
>
> Auf Administrationsservern erzwingt App Control for Business, dass nur signierte Skripte und Module der IT-Abteilung voll laufen. Ein Administrator, der interaktiv in der Konsole arbeitet, bekommt CLM: Er kann Dienste verwalten, Dateien kopieren und Berichte erzeugen, aber kein `Add-Type` mit Schadcode einschleusen. Ein Angreifer, der Zugriff auf eine Sitzung erlangt, hat dadurch deutlich weniger Möglichkeiten.

## 8.4 Least Privilege

Das *Prinzip der geringsten Rechte* (Least Privilege) besagt: Jedes Konto, jeder Prozess und jedes Skript bekommt genau die Rechte, die es für seine Aufgabe braucht, und nicht mehr, und nur so lange, wie es sie braucht. Der Grund liegt in der Schadensbegrenzung. Läuft ein Skript mit Domänenadministratorrechten und enthält einen Fehler oder wird es manipuliert, kann es die gesamte Umgebung beschädigen. Läuft es dagegen mit einem Konto, das nur eine bestimmte Freigabe lesen darf, bleibt der Schaden auf diese Freigabe begrenzt. Die Microsoft-Dokumentation zu Just Enough Administration beschreibt das Problem am Beispiel von DNS-Administratoren, die auf einem Domänencontroller lokale Administratorrechte brauchen und dadurch faktisch Domänenadministratoren werden.

Für Skripte lässt sich das Prinzip in vier Maßnahmen übersetzen. Erstens: *Rechte minimieren*. Skripte laufen unter eigenen Dienstkonten mit gezielten Rechten, nicht unter persönlichen Administratorkonten. Zweitens: *Rollen trennen*. Wer Skripte entwickelt, ist nicht dieselbe Person, die sie freigibt, und das Konto, das liest, ist nicht dasselbe, das schreibt. Drittens: *Privilegierte Teile kapseln*. Nur der kleine Teil eines Skripts, der erhöhte Rechte braucht, läuft damit, der Rest nicht. Viertens: *Rechte zeitlich begrenzen*, zum Beispiel mit temporärer Rechteerhöhung (Just-in-Time) statt dauerhafter Administratormitgliedschaft.

### Erkennen, mit welchen Rechten ein Skript läuft.

Ein Skript sollte wissen, ob es mit erhöhten Rechten läuft, und entsprechend reagieren: Braucht es sie und hat sie nicht, bricht es mit klarer Meldung ab; hat es sie, aber braucht sie nicht, warnt es. Unter Windows prüft man die Mitgliedschaft in der Administratorgruppe mit `WindowsPrincipal`, unter Linux und macOS die Benutzerkennung 0 (root). Die Anweisung `#Requires -RunAsAdministrator` verlangt unter Windows erhöhte Rechte schon beim Start und bricht sonst mit einer Meldung ab.

```powershell
function Test-Erhoeht {
    [CmdletBinding()]
    [OutputType([bool])]
    param()
    if ($IsWindows) {
        $identitaet = [Security.Principal.WindowsIdentity]::GetCurrent()
        $prinzipal  = [Security.Principal.WindowsPrincipal]::new($identitaet)
        return $prinzipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    }
    return ((& id -u) -eq '0')
}

if (Test-Erhoeht) {
    Write-Warning 'Das Skript laeuft mit erhoehten Rechten, braucht sie aber nicht.'
}
```

Die Funktion unterscheidet nach Plattform. Unter Windows erzeugt sie ein `WindowsPrincipal` für den aktuellen Benutzer und fragt, ob er in der Rolle „Administrator“ ist. Die Variable `$IsWindows` gibt es seit PowerShell 6; unter Windows PowerShell 5.1 fehlt sie (dort läuft immer Windows). Unter Linux und macOS ruft sie das Programm `id -u` auf und vergleicht mit `0`. Im Test unter Linux lieferte `id -u` den Wert 0, die Funktion also `True`, weil der Testlauf als root lief; für den Windows-Zweig gilt, dass er nur auf Syntax geprüft wurde.

### Konten und Dienste.

Geplante Aufgaben und Dienste laufen am besten unter *gruppenverwalteten Dienstkonten* (gMSA) in Active-Directory-Umgebungen. Deren Kennwort verwaltet das Betriebssystem, und es ist nie in einem Skript zu sehen. In Azure ersetzen *Managed Identities* die Kennwörter vollständig (Modul 10). Für lokale Aufgaben bietet sich ein eigenes Konto pro Aufgabe an, mit Rechten nur auf die benötigten Ordner. Ein geplanter Task, der im Namen von `SYSTEM` läuft, hat weitreichende Rechte; ein eigenes Konto mit minimalen Rechten ist fast immer die bessere Wahl.

### Delegation mit JEA.

*Just Enough Administration* (JEA) ist die in PowerShell eingebaute Technik für delegierte Administration. Ein Administrator konfiguriert einen Endpunkt, der nur genau die Befehle erlaubt, die eine Rolle braucht, etwa das Neustarten eines bestimmten Dienstes. Die Benutzer verbinden sich mit ihren normalen Konten, und der Endpunkt führt die Befehle unter einem temporären, privilegierten virtuellen Konto aus. Alle Aktionen lassen sich per Transkript nachvollziehen. JEA läuft im `NoLanguage`-Modus und funktioniert über WinRM-basierte Remoting-Endpunkte; SSH-Remoting unterstützt laut Dokumentation JEA nicht. Modul 7 behandelt die Grundlagen von JEA; hier genügt die Einordnung als Werkzeug für Least Privilege.

> **Ergänzung aus der Nachrecherche**
>
> **Aussagen aus der Microsoft-Dokumentation zu JEA.** Laut „Just Enough Administration“ reduziert JEA die Zahl der Administratoren auf Rechnern, indem virtuelle Konten oder gruppenverwaltete Dienstkonten im Auftrag normaler Benutzer privilegierte Aktionen ausführen; es begrenzt, welche Cmdlets, Funktionen und externen Befehle Benutzer ausführen dürfen, und liefert über Transkripte und Protokolle, welche Befehle ein Benutzer in seiner Sitzung tatsächlich ausgeführt hat. Die Seite „PowerShell remoting over SSH“ hält fest, dass SSH-Remoting derzeit keine Endpunktkonfiguration und kein JEA unterstützt.

> **Merke**
>
> Geringste Rechte: eigenes Konto pro Aufgabe, Rollen trennen, privilegierte Teile kapseln, Rechte zeitlich begrenzen. Rechte im Skript prüfen (`WindowsPrincipal`, `id -u`, `#Requires -RunAsAdministrator`). gMSA und Managed Identities statt Kennwörtern. JEA für delegierte Aufgaben.

```powershell
# FALSCH: Ganzes Skript als Administrator, nur ein Schritt braucht es
#Requires -RunAsAdministrator
$daten = Get-Content .\liste.txt
foreach ($d in $daten) { Write-Output $d }
Restart-Service -Name Spooler

# RICHTIG: Privilegierten Schritt kapseln, Rest ohne erhoehte Rechte
$daten = Get-Content .\liste.txt
foreach ($d in $daten) { Write-Output $d }
if (Test-Erhoeht) { Restart-Service -Name Spooler }
else { Write-Warning 'Dienstneustart uebersprungen: erhoehte Rechte noetig.' }
```

Im ersten Fall läuft jede Zeile mit Administratorrechten, auch das Lesen und Ausgeben, das sie nicht braucht. Im zweiten Fall ist der privilegierte Teil klein, sichtbar und überspringbar.

> **Praxisbeispiel**
>
> Das Helpdesk-Team soll den Druckspooler auf Druckservern neu starten dürfen, ohne Administrator zu sein. Statt ihnen Administratorrechte zu geben, richtet das Serverteam einen JEA-Endpunkt ein, der nur `Restart-Service` für den Dienst `Spooler` erlaubt. Das Helpdesk verbindet sich mit dem eigenen Konto, und jeder Neustart steht im Transkript.

## 8.5 Secrets Management

Ein *Geheimnis* (Secret) ist jeder Wert, mit dem man sich ausweisen oder Zugang verschaffen kann: Kennwörter, API-Schlüssel, Tokens, Zertifikatsschlüssel, Verbindungszeichenfolgen. Skripte brauchen sie ständig, und die Versuchung ist groß, sie dorthin zu schreiben, wo es bequem ist: in das Skript, in eine Konfigurationsdatei, in Umgebungsvariablen eines Projekts, in den Verlauf der Konsole. Genau dort werden sie gefunden. Ein im Skript abgelegtes Kennwort landet in der Versionsverwaltung und bleibt dort in der Historie, auch wenn man es später löscht, wird mit dem Skript kopiert, per Mail verschickt und in Protokollen gespeichert.

Die Grundregel lautet: *Geheimnisse gehören nicht in Code, Logs oder Verlauf*, sondern in einen dafür gebauten Tresor, und ein Skript holt sich den Wert erst zur Laufzeit. Am besten gibt es das Geheimnis gar nicht: Wenn ein Skript sich mit einer Identität anmelden kann (Managed Identity, Zertifikat, Windows-Authentifizierung), braucht es kein Kennwort. Die Dokumentation von Microsoft nennt passwortlose Verfahren wie Passkeys, Single Sign-On und föderierte Identitätssysteme ausdrücklich als Zukunft.

PowerShell bietet für Tresore das Modul *Microsoft.PowerShell.SecretManagement*. Es ist eine einheitliche Schnittstelle (eine Abstraktionsschicht) mit wenigen Cmdlets zum Speichern und Lesen von Geheimnissen, die verschiedene Tresore dahinter ansprechen kann. Die konkrete Ablage erledigen *Erweiterungs-Tresore*, die man separat installiert und registriert. Ein Beispiel ist *Microsoft.PowerShell.SecretStore*: ein lokaler, verschlüsselter Tresor für den aktuellen Benutzer, der auf allen Plattformen mit PowerShell 7 läuft. Weitere Erweiterungen binden Azure Key Vault, KeePass, HashiCorp Vault, LastPass oder den Schlüsselbund von macOS an. Der Vorteil der Schnittstelle: Das Skript ruft nur `Get-Secret -Name ...` auf, und welcher Tresor dahintersteht, kann sich ändern, ohne dass das Skript angepasst werden muss. So lassen sich Skripte weitergeben, ohne zu wissen, welche Tresore die Benutzer haben, und dasselbe Skript läuft lokal, im Test und in der Produktion, indem man nur den Namen des Tresors ändert.

> **Ergänzung aus der Nachrecherche**
>
> **Stand von SecretManagement (laut Microsoft Learn).** Die Dokumentation vermerkt, dass das PowerShell-Team die Secret-Module als funktional abgeschlossen betrachtet: Sie werden nicht mehr aktiv weiterentwickelt, aber weiter für Sicherheits- und kritische Fehlerkorrekturen unterstützt; das Code-Repository ist archiviert. Als jüngste veröffentlichte Versionen nennt die Seite SecretManagement 1.1.2 und SecretStore 1.0.6. Als Begründung heißt es, die Natur von Geheimnissen habe sich seit dem Entwurf grundlegend geändert, und passwortlose Verfahren seien die Zukunft. Prüfen Sie vor dem Einsatz in einem neuen Projekt, ob Managed Identities oder Zertifikate das Geheimnis überflüssig machen.

Das folgende Beispiel zeigt die Einrichtung eines lokalen Tresors und den Zugriff. Es konnte im Arbeitsbereich nicht ausgeführt werden (kein Zugriff auf die PowerShell Gallery); die Befehle sind syntaxgeprüft und folgen der Modulbeschreibung.

```powershell
# Einmalig: Module installieren und einen lokalen Tresor registrieren
Install-PSResource -Name Microsoft.PowerShell.SecretManagement -Scope CurrentUser
Install-PSResource -Name Microsoft.PowerShell.SecretStore -Scope CurrentUser
Register-SecretVault -Name LokalerTresor -ModuleName Microsoft.PowerShell.SecretStore -DefaultVault

# Geheimnis speichern (Wert wird nie im Skript hinterlegt)
Set-Secret -Name 'ApiToken' -Secret (Read-Host -AsSecureString 'Token')
Set-Secret -Name 'Dienstkonto' -Secret ([pscredential]::new('svc-backup', (Read-Host -AsSecureString)))

# Abfragen: Namen auflisten, Wert holen
Get-SecretInfo | Select-Object Name, Type, VaultName
$token = Get-Secret -Name 'ApiToken'                  # SecureString
$cred  = Get-Secret -Name 'Dienstkonto'               # PSCredential
$text  = Get-Secret -Name 'ApiToken' -AsPlainText     # nur wenn zwingend noetig

# In einer Pipeline ohne Benutzer: Tresor mit Kennwort entsperren
Unlock-SecretStore -Password $tresorKennwort
```

Der erste Block installiert die beiden Module für den aktuellen Benutzer und registriert den SecretStore als Standardtresor. Der zweite speichert Geheimnisse: Der Wert wird zur Laufzeit mit `Read-Host -AsSecureString` eingegeben, steht also nie im Skript oder im Verlauf. Der dritte Block liest Geheimnisse: `Get-SecretInfo` zeigt nur Namen und Typ, nicht den Wert; `Get-Secret` liefert je nach gespeichertem Typ ein `SecureString` oder ein `PSCredential`. Den Klartext (`-AsPlainText`) holt man nur, wenn eine Schnittstelle zwingend Text verlangt, und gibt ihn nie aus. Der SecretStore verlangt standardmäßig ein Kennwort für den Zugriff. In unbeaufsichtigten Umgebungen kann man das mit `Set-SecretStoreConfiguration -Authentication None -Interaction None` abschalten, was den Schutz reduziert (der Tresor ist dann nur noch durch das Benutzerkonto geschützt), oder man entsperrt ihn mit `Unlock-SecretStore`, wobei das Entsperrkennwort dann selbst ein Geheimnis ist. Dieses „Henne-Ei-Problem“ (Wo liegt das Kennwort für den Tresor?) löst man am besten mit einer Identität, etwa Managed Identity und Key Vault (Modul 10).

### Wo Geheimnisse trotzdem auftauchen.

Auch mit Tresor gibt es Lecks. Der Verlauf der Konsole (die Erweiterung PSReadLine speichert eingegebene Zeilen in einer Datei) enthält jedes Kennwort, das man als Argument eintippt. Skriptausgaben und Transkripte enthalten Werte, die man ausgibt. Fehlermeldungen können Verbindungszeichenfolgen mit Kennwort zeigen. Umgebungsvariablen sind für alle Kindprozesse sichtbar. Die Gegenmaßnahmen: Geheimnisse nie als Argument oder Text im Code, nie in eine Ausgabe, nur als `SecureString` oder `PSCredential` durch das Skript führen, und Logs vor dem Schreiben bereinigen (Lerneinheit 5.6).

> **Merke**
>
> Geheimnisse nie in Code, Konfiguration, Log oder Verlauf. Besser: gar kein Geheimnis (Managed Identity, Zertifikat). Sonst Tresor über SecretManagement (`Register-SecretVault`, `Set-Secret`, `Get-Secret`) mit SecretStore, Key Vault oder anderem Erweiterungs-Tresor. SecretManagement ist funktional abgeschlossen. Klartext nur wenn zwingend.

```powershell
# FALSCH: Token im Skript und im Verlauf
$token = 'ghp_ABCDEFG1234567'
Invoke-RestMethod -Uri $url -Headers @{ Authorization = "Bearer $token" }

# RICHTIG: Token aus dem Tresor, erst zur Laufzeit
$token = Get-Secret -Name 'ApiToken' -AsPlainText
Invoke-RestMethod -Uri $url -Headers @{ Authorization = "Bearer $token" }
```

Im ersten Fall steht das Token in der Datei, im Repository und in jedem Backup. Im zweiten kennt das Skript nur den Namen; der Wert kommt aus dem Tresor und taucht nur im Arbeitsspeicher auf (und der Klartextumweg ist in diesem Fall nötig, weil der HTTP-Kopf Text verlangt).

> **Praxisbeispiel**
>
> Ein Repository enthält Skripte für die Cloud-Verwaltung. Ein Prüfwerkzeug durchsucht den Verlauf nach eingetragenen Schlüsseln (Secret Scanning) und findet einen Zugangsschlüssel aus einem alten Commit. Obwohl der Schlüssel im neuesten Stand gelöscht ist, gilt er als offengelegt und muss gesperrt und ersetzt werden; Löschen aus der Historie reicht nicht. Das Team stellt auf Managed Identity um, sodass kein Schlüssel mehr existiert.

## 8.6 Credential Handling

Auch wenn man Geheimnisse nicht im Code ablegt, muss man in PowerShell oft Anmeldedaten übergeben: an `New-PSSession`, an `Invoke-Command`, an Cmdlets für Datenbanken oder Verzeichnisse. PowerShell kennt dafür zwei Typen. Ein `SecureString` ist eine Zeichenkette, die so gespeichert ist, dass sie nicht versehentlich ausgegeben wird. Ein `PSCredential` bündelt einen Benutzernamen (als Text) mit einem Kennwort (als `SecureString`). Cmdlets, die sich anmelden, erwarten fast immer einen Parameter `-Credential` vom Typ `PSCredential`.

```powershell
$pw = ConvertTo-SecureString 'Geheim123!' -AsPlainText -Force
$pw
"$pw"
$cred = [pscredential]::new('svc-backup', $pw)
$cred | Format-List

[System.Net.NetworkCredential]::new('', $cred.Password).Password
$cred.GetNetworkCredential().Password
ConvertFrom-SecureString -SecureString $pw -AsPlainText
```

Die erste Zeile erzeugt aus einem Klartext ein `SecureString`; `-AsPlainText -Force` bestätigt ausdrücklich, dass man den unsicheren Weg geht, weil der Klartext hier bereits im Code steht. Die Ausgabe von `$pw` und `"$pw"` zeigt nur den Typnamen `System.Security.SecureString`, nicht den Inhalt; ebenso zeigt `$cred | Format-List` bei `Password` nur den Typnamen. Das ist der Zweck: kein versehentliches Ausgeben in Konsole oder Protokoll. Die letzten drei Zeilen zeigen aber die Wahrheit über die Schutzwirkung: Den Klartext erhält jeder, der ein `SecureString` oder `PSCredential` in der Hand hat, mit einem Einzeiler zurück. Im Test lieferten alle drei Varianten `Geheim123!`. Ein `SecureString` ist also ein Schutz gegen *versehentliche* Preisgabe, kein Schutz gegen jemanden, der im selben Prozess Code ausführen kann.

> **Ergänzung aus der Nachrecherche**
>
> **SecureString ist für neue Entwicklung nicht empfohlen (laut „PowerShell security features“).** Microsoft empfiehlt, `SecureString` nicht für neue Entwicklungen zu verwenden. PowerShell unterstützt den Typ aus Gründen der Abwärtskompatibilität weiter, und er ist sicherer als ein Klartext, weil er verhindert, dass der Inhalt versehentlich auf der Konsole oder in Protokollen erscheint. Die Seite empfiehlt, Kennwörter zu vermeiden und andere Verfahren zu verwenden, etwa Zertifikate oder die Windows-Authentifizierung. Der Typ lässt sich leicht in Klartext zurückverwandeln und muss mit Vorsicht gebraucht werden.

### Warum die Plattform entscheidend ist.

Nun zu einer Eigenschaft, die in vielen Skripten zu einem Sicherheitsloch führt. Man will Anmeldedaten für einen unbeaufsichtigten Lauf speichern, zum Beispiel mit `Export-Clixml` oder `ConvertFrom-SecureString`. Unter Windows verschlüsseln beide den Wert mit der Windows Data Protection API (DPAPI): Die Datei lässt sich nur mit demselben Benutzerkonto auf demselben Rechner wieder einlesen. Auf Linux und macOS gibt es DPAPI nicht. Die Dokumentation zu `ConvertFrom-SecureString` sagt ausdrücklich: „The contents of a SecureString aren’t encrypted on non-Windows systems.“ Der Test macht das sichtbar.

```powershell
$pw   = ConvertTo-SecureString 'Geheim123!' -AsPlainText -Force
$cred = [pscredential]::new('svc-backup', $pw)
$datei = Join-Path ([System.IO.Path]::GetTempPath()) 'cred.xml'

$cred | Export-Clixml -Path $datei
Select-String -Path $datei -Pattern 'Password'

$pw | ConvertFrom-SecureString
Import-Clixml -Path $datei | ForEach-Object { $_.GetNetworkCredential().Password }
Remove-Item -LiteralPath $datei
```

Unter Linux enthielt die Datei die Zeile `<SS N="Password">470065006800650069006d003100320033002100</SS>`, und `ConvertFrom-SecureString` lieferte dieselbe Zeichenfolge. Das ist keine Verschlüsselung, sondern lediglich die hexadezimal geschriebene UTF-16-Darstellung des Kennworts: `47 00 65 00 68 00 ...` sind die Zeichen G, e, h, ... mit je einem Nullbyte. Jeder, der die Datei lesen kann, liest das Kennwort. Wer Skripte schreibt, die auf mehreren Plattformen laufen, darf sich auf diese Dateien daher nicht verlassen. Auch unter Windows gilt: Die DPAPI-Verschlüsselung schützt nur vor anderen Benutzern und anderen Rechnern, nicht vor Prozessen desselben Benutzers.

Eine Alternative für den plattformübergreifenden Einsatz ist die Verschlüsselung mit einem AES-Schlüssel (`-Key` oder `-SecureKey`). Das verschiebt das Problem lediglich: Wo liegt der Schlüssel? Liegt er neben der Datei, schützt er nichts. Im Test ergab die Umwandlung mit einem 32-Byte-Schlüssel und zurück wieder das Ausgangskennwort; als Mechanismus taugt das nur, wenn der Schlüssel aus einer anderen, geschützten Quelle stammt.

### Anmeldedaten sicher beschaffen.

Interaktiv fragt `Get-Credential` Benutzernamen und Kennwort ab und liefert ein `PSCredential`; `Read-Host -AsSecureString` fragt ein einzelnes Kennwort ab, ohne es anzuzeigen. Im Test (ohne Terminal) erschienen die Abfragen „PowerShell credential request“ und „Password for user u:“, das Skript wartete also auf Eingabe. Für unbeaufsichtigte Läufe sind Tresor (Lerneinheit 8.5), Managed Identity oder Zertifikat die richtigen Quellen. Skripte sollten Anmeldedaten als Parameter vom Typ `[pscredential]` entgegennehmen, statt Benutzername und Kennwort als zwei Texte; so gelangt das Kennwort nie als Klartext durch die Schnittstelle, und Werkzeuge wie PSScriptAnalyzer (Lerneinheit 5.9) melden Klartextparameter.

```powershell
function Connect-Dienst {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Server,
        [Parameter(Mandatory)] [pscredential] $Credential
    )
    # Anmeldedaten nur durchreichen, nie ausgeben oder protokollieren
    Write-Verbose "Verbinde mit $Server als $($Credential.UserName)"
    # New-PSSession -ComputerName $Server -Credential $Credential
}

$cred = Get-Credential -Message 'Dienstkonto'
Connect-Dienst -Server 'srv01' -Credential $cred -Verbose
```

Die Funktion nimmt ein `PSCredential` entgegen und gibt in der Meldung nur den Benutzernamen aus, nie das Kennwort. Das Durchreichen an `-Credential` anderer Cmdlets (hier als Kommentar) ist der richtige Weg. Der Aufrufer erhält die Daten per `Get-Credential` oder aus dem Tresor.

> **Merke**
>
> `SecureString` und `PSCredential` verhindern versehentliche Ausgabe, nicht gezielte Rückgewinnung. Auf Nicht-Windows-Systemen sind `Export-Clixml` und `ConvertFrom-SecureString` *nicht verschlüsselt*. Parameter vom Typ `[pscredential]` nutzen, Klartext vermeiden, nie ausgeben oder loggen. Für neue Entwicklung besser Zertifikate oder Managed Identities.

```powershell
# FALSCH: Klartextpasswort als Parameter und Kennwort in Datei
param([string] $Benutzer, [string] $Passwort)
$cred = [pscredential]::new($Benutzer, (ConvertTo-SecureString $Passwort -AsPlainText -Force))
$Passwort | Out-File .\kennwort.txt

# RICHTIG: PSCredential als Parameter, keine Datei mit Kennwort
param([Parameter(Mandatory)] [pscredential] $Credential)
```

Im ersten Fall steht das Kennwort in der Prozessliste und im Verlauf des Aufrufs und danach auch in einer Datei. Im zweiten wird es als `SecureString` durchgereicht und nirgends gespeichert.

> **Praxisbeispiel**
>
> Ein Team stellt fest, dass auf einem Linux-Server eine Datei `cred.xml` liegt, die ein Skript für nächtliche Läufe liest. Es war überzeugt, sie sei verschlüsselt, weil das Skript unter Windows entwickelt wurde. Der Test zeigt, dass sie auf Linux lesbar ist. Das Team ersetzt sie durch ein Zertifikat für die Anmeldung und prüft zugleich die anderen Skripte, die auf dieselbe Weise gespeichert haben.

## 8.7 Input Validation Security

Jede Eingabe, die ein Skript nicht selbst erzeugt, ist *nicht vertrauenswürdig*: Parameter, Dateiinhalte, Webantworten, Umgebungsvariablen, Namen aus einem Verzeichnisdienst. Wird eine solche Eingabe in einen Befehl, einen Pfad oder eine Abfrage eingebaut, kann sie das Verhalten des Skripts ändern. Diese Angriffsklasse heißt *Injection*. Sie ist in PowerShell besonders relevant, weil Skripte oft mit erhöhten Rechten laufen und Eingaben aus Tickets, Formularen oder Dateien verarbeiten.

Die klassische Gefahr ist die Befehlseinschleusung über `Invoke-Expression`. Der Befehl führt einen Text als Code aus. Baut man diesen Text mit einer Eingabe zusammen, entscheidet der Angreifer, was ausgeführt wird. Das folgende Beispiel zeigt das mit einer harmlosen Nutzlast.

```powershell
$name = "x'; Write-Output INJIZIERT; '"

# FALSCH: Eingabe wird Teil des Codes
Invoke-Expression "Write-Output '$name'"

# RICHTIG: Eingabe bleibt Daten
Write-Output $name
```

Beim Test gab der erste Aufruf zwei Zeilen aus, `x` und `INJIZIERT`: Die Eingabe hat das Anführungszeichen geschlossen und einen eigenen Befehl angehängt. Der zweite Aufruf gab die Eingabe unverändert als Text aus. Die Regel lautet daher: `Invoke-Expression` nie mit Eingaben füttern; fast immer gibt es eine Alternative mit Parametern (Aufrufoperator `&`, Splatting).

### Pfade.

Die zweite Gefahr betrifft Dateipfade. Setzt ein Skript einen Benutzerwert an einen Basispfad, kann der Angreifer mit `../` aus dem erlaubten Ordner ausbrechen (*Path Traversal*). Im Test las `Get-Content (Join-Path $basis '../geheim.txt')` die Datei außerhalb des Basisordners. Die Abwehr besteht darin, den vollständigen Pfad aufzulösen und zu prüfen, dass er noch unterhalb des erlaubten Ordners liegt. Wichtig: Der Vergleich muss mit einem abschließenden Trennzeichen erfolgen, sonst gilt der Ordner `daten-evil` als unterhalb von `daten`.

```powershell
function Resolve-SichererPfad {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Basis,
        [Parameter(Mandatory)] [string] $Relativ
    )
    $trenner   = [System.IO.Path]::DirectorySeparatorChar
    $basisVoll = [System.IO.Path]::GetFullPath($Basis).TrimEnd($trenner) + $trenner
    $ziel      = [System.IO.Path]::GetFullPath((Join-Path $Basis $Relativ))
    if (-not $ziel.StartsWith($basisVoll, [System.StringComparison]::OrdinalIgnoreCase)) {
        throw "Pfad ausserhalb des erlaubten Ordners: $Relativ"
    }
    $ziel
}

Resolve-SichererPfad -Basis '/tmp/daten' -Relativ 'ok.txt'
Resolve-SichererPfad -Basis '/tmp/daten' -Relativ '../geheim.txt'
Resolve-SichererPfad -Basis '/tmp/daten' -Relativ '../daten-evil/x'
```

Die Funktion berechnet den vollständigen Basispfad mit abschließendem Trennzeichen, löst den Zielpfad mit `GetFullPath` auf (das `../` wird dabei aufgelöst) und vergleicht den Anfang. Im Test lieferte der erste Aufruf den Pfad, die beiden anderen die Meldung „Pfad ausserhalb des erlaubten Ordners“. Der Vergleich ohne Beachtung der Groß-/Kleinschreibung entspricht Windows; unter Linux wäre ein genauer Vergleich (`Ordinal`) strenger und besser. Symbolische Verknüpfungen löst `GetFullPath` nicht auf; wer sie ausschließen muss, prüft zusätzlich das Attribut `LinkType`.

### Platzhalter.

PowerShell-Pfadparameter (`-Path`) interpretieren Zeichen wie `*`, `?` und eckige Klammern als Muster. Ein Dateiname mit Klammern kann so ein anderes Ziel treffen als gewollt. Im Test lagen die Dateien `[abc].txt` und `a.txt` im selben Ordner; `Get-Content -Path '[abc].txt'` las `a.txt` (Inhalt `datei-a`), `-LiteralPath` dagegen die Datei mit dem Namen `[abc].txt`. Für Eingaben aus unsicherer Quelle verwendet man deshalb immer `-LiteralPath`. Das gilt besonders für Löschbefehle, denn `Remove-Item -Path '*'` löscht alles.

### Native Programme und Argumente.

Beim Aufruf externer Programme übergibt PowerShell Argumente als getrennte Werte. Im Test gab `/bin/echo $arg` mit `$arg = 'a b; echo PWNED'` den Text unverändert aus, während `/bin/sh -c "echo $arg"` das Semikolon als Befehlstrenner auswertete und „PWNED“ als eigene Zeile ausgab. Die Lehre: Argumente niemals in einen Befehlstext einbauen, der von einer Shell interpretiert wird; sie einzeln übergeben. Gleiches gilt für Datenbankabfragen: Werte gehören in Parameter der Abfrage, nicht in den Abfragetext; die Zeichenfolgenverkettung `"SELECT ... WHERE n = '$user'"` ergab im Test mit `x' OR '1'='1` eine Abfrage, die jede Zeile trifft.

### Allowlists und Typen.

Die wirksamste Verteidigung ist, nur Bekanntes zu erlauben (*Allowlist*), statt Böses zu verbieten. PowerShell bietet dafür Parameterattribute (Lerneinheit 4.7): `ValidateSet` für feste Werte, `ValidatePattern` für Muster, `ValidateRange` für Zahlen und Typen wie `[guid]`, `[ipaddress]`, `[uri]`, `[int]`, die falsche Werte beim Binden ablehnen. Im Test lehnte `ValidatePattern('^[a-zA-Z0-9-]{1,63}$')` den Namen `srv; calc` ab, `ValidateSet('nginx','sshd','cron')` den Wert `nginx; rm -rf /`, `[guid]` den Wert `nope` („Unrecognized Guid format.“) und `[ipaddress]` den Wert `999.1.1.1` („An invalid IP address was specified.“). Typisierte Parameter wehren so ganze Klassen von Eingaben ab, noch bevor der erste Befehl läuft.

```powershell
function Restart-AppDienst {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)]
        [ValidateSet('nginx', 'sshd', 'cron')]
        [string] $Name,

        [ValidatePattern('^[a-zA-Z0-9-]{1,63}$')]
        [string] $Server = 'localhost',

        [ValidateRange(1, 65535)]
        [int] $Port = 22
    )
    if ($PSCmdlet.ShouldProcess("$Server`:$Port", "Dienst $Name neu starten")) {
        # Restart-Service -Name $Name -ComputerName $Server  (Beispiel)
        "Neustart: $Name auf $Server"
    }
}

Restart-AppDienst -Name nginx -WhatIf
try { Restart-AppDienst -Name 'nginx; calc' } catch { $_.Exception.Message }
```

Die Funktion kombiniert Allowlist (`ValidateSet`), Muster (`ValidatePattern`) und Zahlenbereich (`ValidateRange`); jede ungültige Eingabe scheitert, bevor der Funktionsrumpf läuft. `ShouldProcess` ergänzt eine Sicherung gegen Versehen. Der zweite Aufruf endet mit der Meldung des `ValidateSet`-Attributs.

> **Ergänzung aus der Nachrecherche**
>
> **Tiefe und Größe bei JSON und Dateien.** Auch Datenformate sind Eingaben. `ConvertFrom-Json` begrenzt die Verschachtelungstiefe über `-Depth`; im Test brach `ConvertFrom-Json -Depth 1` bei einem dreifach verschachtelten Objekt mit „The reader’s MaxDepth of 1 has been exceeded“ ab. Dateien aus unsicherer Quelle sollte man vor dem Einlesen auf Größe und Typ prüfen. Für Konfigurationen lädt man Daten mit `Import-PowerShellDataFile` statt Skripte auszuführen (Lerneinheit 4.9).

> **Merke**
>
> Eingaben sind nicht vertrauenswürdig. Nie in `Invoke-Expression` oder Shell-Befehlstexte einbauen. Pfade auflösen und gegen den Basisordner prüfen, `-LiteralPath` verwenden. Argumente einzeln übergeben, Abfragen parametrisieren. Allowlists (`ValidateSet`, `ValidatePattern`, `ValidateRange`) und typisierte Parameter.

```powershell
# FALSCH: Benutzereingabe im Dateipfad und im Befehl
$datei = Read-Host 'Datei'
Get-Content -Path "C:\Berichte\$datei"
cmd /c "type C:\Berichte\$datei"

# RICHTIG: Pfad aufloesen und pruefen, Argument einzeln uebergeben
$voll = Resolve-SichererPfad -Basis 'C:\Berichte' -Relativ $datei
Get-Content -LiteralPath $voll
```

Im ersten Fall kann die Eingabe `..\..\Windows\win.ini` den erlaubten Ordner verlassen, und `cmd /c` wertet Zeichen wie `&` als Befehlstrenner aus. Im zweiten bleibt der Zugriff im Ordner, und es wird kein Befehlstext gebaut.

> **Praxisbeispiel**
>
> Ein Self-Service-Skript legt auf Wunsch aus einem Ticket Benutzerordner an; der Ordnername stammt aus dem Ticketfeld. Durch einen Namen wie `..\..\Windows` könnte ein Ordner an unerwarteter Stelle entstehen. Mit einer `ValidatePattern`-Allowlist für den Namen und der Pfadprüfung gegen den Basisordner ist das ausgeschlossen, und ein Eintrag im Log hält jeden abgewiesenen Versuch fest.

## 8.8 PowerShell Logging Security

Verhindern allein genügt nicht; man muss auch erkennen, wenn etwas schiefgeht oder missbraucht wird. PowerShell bietet dafür drei eingebaute Protokollierungsfunktionen, die über Gruppenrichtlinien, Registrierung oder eine Konfigurationsdatei aktiviert werden: *Module Logging*, *Script Block Logging* und *Transcription*. Sie sind kein Ersatz für das Logging eigener Skripte (Lerneinheit 5.6), sondern ein Werkzeug der Sicherheitsüberwachung: Sie protokollieren, was ausgeführt wurde, auch wenn der Ausführende kein Logging eingebaut hat.

*Module Logging* zeichnet die Pipeline-Ausführung für die ausgewählten Module auf. Es muss laut Dokumentation sowohl für die Sitzung als auch für die einzelnen Module aktiviert sein; einzelne Module lassen sich in einer Sitzung über die Eigenschaft `LogPipelineExecutionDetails` des Moduls ein- oder ausschalten. *Script Block Logging* zeichnet die Verarbeitung aller Befehle, Skriptblöcke, Funktionen und Skripte auf, ob interaktiv oder per Automation aufgerufen. Wichtig für die Erkennung von Angriffen: Auch Code, der zur Laufzeit zusammengebaut oder verschleiert wurde, erscheint im Protokoll im Klartext, denn protokolliert wird der Skriptblock, bevor er ausgeführt wird. *Transcription* schreibt die Konsolenausgabe von Sitzungen in Textdateien (Lerneinheit 5.5) und lässt sich über eine Gruppenrichtlinie automatisch für alle Sitzungen einschalten.

> **Ergänzung aus der Nachrecherche**
>
> **Details laut about_Logging_Windows (PowerShell 7).** Unter Windows muss der PowerShell-Ereignisanbieter vor der Nutzung registriert werden, und zwar mit `$PSHOME\RegisterManifest.ps1` in einer Eingabeaufforderung mit erhöhten Rechten; zum Aktualisieren der Bibliothek ist er mit `-Unregister` abzumelden. Die Protokolle erscheinen in der Ereignisanzeige unter „Anwendungs- und Dienstprotokolle“ im Protokoll `PowerShellCore`; die zugehörige ETW-Anbieter-GUID lautet `{f90714a8-5509-434a-bf6d-b1624c8a19a2}`. Script Block Logging schreibt die Ereignisse mit der Kennung 4104 in das Protokoll `PowerShellCore/Operational`. Aktivieren lässt es sich über die Gruppenrichtlinie „Turn on PowerShell Script Block Logging“ (Administrative Templates, PowerShell Core), über den Registrierungswert `EnableScriptBlockLogging` unter `HKLM:\Software\Policies\Microsoft\PowerShellCore\ScriptBlockLogging` oder über die Datei `powershell.config.json`. Für Windows PowerShell 5.1 nennt die Seite „PowerShell security features“ dagegen das Protokoll `Microsoft-Windows-PowerShell/Operational`; achten Sie bei der Auswertung also darauf, welche PowerShell-Edition das Ereignis erzeugt hat.

Das folgende Beispiel für Windows zeigt, wie man Script Block Logging über die Registrierung aktiviert (das Muster stammt aus der Microsoft-Dokumentation) und die Ereignisse mit der Kennung 4104 liest.

```powershell
# Script Block Logging fuer PowerShell 7 per Registrierung aktivieren (erhoehte Rechte)
function Enable-PSScriptBlockLogging {
    $basis = 'HKLM:\Software\Policies\Microsoft\PowerShellCore\ScriptBlockLogging'
    if (-not (Test-Path $basis)) { $null = New-Item $basis -Force }
    Set-ItemProperty $basis -Name EnableScriptBlockLogging -Value '1'
}

# Ereignisse lesen (PowerShell 7, Windows)
Get-WinEvent -LogName 'PowerShellCore/Operational' -MaxEvents 20 |
    Where-Object Id -eq 4104 |
    Select-Object TimeCreated, Id, @{ n = 'Text'; e = { $_.Message.Substring(0, [Math]::Min(80, $_.Message.Length)) } }
```

Die Funktion legt, falls nötig, den Registrierungsschlüssel an und setzt den Wert `EnableScriptBlockLogging` auf 1. Neue PowerShell-Sitzungen protokollieren danach die Skriptblöcke. Die Abfrage liest die jüngsten Ereignisse aus dem Protokoll, filtert auf die Kennung 4104 und zeigt Zeitpunkt und die ersten Zeichen des Skripttexts. Beide Teile konnten unter Linux nicht ausgeführt werden und sind auf Syntax geprüft. Auf Linux und macOS schreibt PowerShell die Protokolle in das System-Log (laut Handbuch „about_Logging_Non-Windows“; diese Seite wurde nicht abgerufen).

### Datenschutz und Sicherheit der Protokolle selbst.

Hier liegt der eigentliche Zielkonflikt der Lerneinheit. Je mehr protokolliert wird, desto größer ist die Gefahr, dass die Protokolle sensible Daten enthalten. Script Block Logging schreibt den Skripttext, und damit alles, was im Skript steht oder berechnet wird, etwa Kennwörter, die ein Skript als Text zusammenbaut. Die Dokumentation warnt, dass bei aktivierter Skriptprotokollierung Anmeldedaten oder andere vertrauliche Daten, die ein Skript verwendet, in das Ereignisprotokoll gelangen können, und dass diese Informationen einem Angreifer helfen, wenn der Rechner kompromittiert wird. Sie empfiehlt daher, *Protected Event Logging* einzuschalten, wenn Script Block Logging für mehr als Diagnosezwecke genutzt wird.

Protected Event Logging verschlüsselt sensible Ereignisdaten mit einem öffentlichen Schlüssel (Standard Cryptographic Message Syntax, CMS). Der Schlüssel zum Entschlüsseln liegt nicht auf dem protokollierenden Rechner, sondern an einem sicheren zentralen Ort, etwa einem zentralen Sammelserver oder einem SIEM. Das Zertifikat muss laut Dokumentation die erweiterte Schlüsselverwendung „Document Encryption“ (`1.3.6.1.4.1.311.80.1`) und entweder „Data Encipherment“ oder „Key Encipherment“ tragen; der private Schlüssel darf nicht auf den protokollierenden Rechnern liegen. Die Entschlüsselung geschieht mit dem Cmdlet `Unprotect-CmsMessage`, wie in der Dokumentation: `Get-WinEvent Microsoft-Windows-PowerShell/Operational | Where-Object Id -EQ 4104 | Unprotect-CmsMessage`.

> **Ergänzung aus der Nachrecherche**
>
> **Datenschutz und Mitbestimmung.** Protokolle, die Benutzernamen, Rechnernamen und Skriptinhalte enthalten, sind in der EU personenbezogene Daten, sobald sie einer Person zugeordnet werden können. Es gelten die Grundsätze der Datenschutz-Grundverordnung (DSGVO), vor allem Zweckbindung, Datensparsamkeit, Speicherbegrenzung und Transparenz; in vielen Organisationen ist zusätzlich die Arbeitnehmervertretung einzubinden, weil Protokolle auch das Verhalten von Beschäftigten zeigen. Diese Hinweise sind allgemein gehalten, wurden nicht aus Rechtstexten abgerufen und sind *keine Rechtsberatung*. Klären Sie Umfang, Zweck und Aufbewahrungsdauer der Protokollierung mit der oder dem Datenschutzbeauftragten und der Rechtsabteilung.

Aus der Sicht der Sicherheitsüberwachung sind folgende Muster in den Protokollen verdächtig: sehr lange, kodierte Zeichenketten (Base64) in Skriptblöcken, Aufrufe von `Invoke-Expression` oder `DownloadString` mit Webadressen, Aufrufe in der Reihenfolge eines Werkzeugs für Angriffe, das Umgehen von Schutzfunktionen und plötzlich auftretende Skriptblöcke auf Servern, auf denen sonst niemand interaktiv arbeitet. Die Abwägung zwischen Erkennung und Datenschutz besteht darin, zentral zu sammeln, mit klaren Zugriffsrechten und Aufbewahrungsfristen, Geheimnisse gar nicht erst als Text im Skript zu führen und sensible Ereignisse zu verschlüsseln.

> **Merke**
>
> Module Logging, Script Block Logging (Ereignis 4104), Transcription. Script Block Logging zeigt auch verschleierten Code im Klartext. Protokolle können Geheimnisse und personenbezogene Daten enthalten: Protected Event Logging, zentrale Auswertung mit Zugriffsbeschränkung, Aufbewahrungsfristen, Abstimmung mit Datenschutz. Keine Rechtsberatung.

```powershell
# FALSCH: Kennwort im Skript zusammenbauen (landet im Script Block Log)
$kennwort = 'Geheim' + '123!'
Invoke-Dienst -Passwort $kennwort

# RICHTIG: Geheimnis aus dem Tresor als SecureString, nie als Text im Skript
$kennwort = Get-Secret -Name 'Dienst'
Invoke-Dienst -Credential (New-Object pscredential 'svc', $kennwort)
```

Der erste Skriptblock steht mit seinem Kennwort im Klartext im Ereignisprotokoll, weil Script Block Logging den Text speichert. Der zweite enthält nur den Namen des Geheimnisses und führt den Wert als `SecureString`.

> **Praxisbeispiel**
>
> Ein Sicherheitsteam wertet täglich die Skriptblock-Ereignisse aller Server in einem zentralen System aus und sucht nach kodierten Befehlen. Dabei fällt ein Skriptblock auf, der eine lange Base64-Zeichenkette dekodiert und ausführt. Das Protokoll zeigt den entschlüsselten Klartext; so erkennt das Team die Absicht, auch wenn der Angreifer den Code verschleiert hatte. Gleichzeitig sind die Ereignisse verschlüsselt, sodass nur das Sicherheitsteam sie lesen kann.

## 8.9 Supply Chain Security

Kein Skript steht allein. Es lädt Module aus Katalogen, ruft Skripte aus Repositories ab, nutzt Pakete und Container und läuft in Pipelines, die selbst Software verwenden. Jede dieser Abhängigkeiten ist ein möglicher Angriffspunkt: Ein manipuliertes Modul, ein Paket mit einem Namen, der dem beliebten Modul ähnelt (Typosquatting), ein übernommenes Konto eines Autors oder ein kompromittierter Katalog. Man nennt das *Lieferkettenangriff* (Supply Chain Attack). Weil ein importiertes Modul mit den Rechten des Skripts läuft, kann es alles tun, was das Skript darf.

Die Abwehr beruht auf vier Bausteinen. Erstens: *Herkunft prüfen*. Man installiert nur Module aus bekannten Quellen und prüft Herausgeber, Veröffentlichungsdatum, Downloadzahlen, Projektseite und die Signatur. Zweitens: *Versionen festlegen*. Statt immer die neueste Version zu laden, bindet man ein Skript an eine geprüfte Version, sodass sich nichts unbemerkt ändert. Drittens: *Interne Repositories*. Eine Organisation betreibt ein eigenes Repository, in das nur geprüfte Module gelangen, und lässt Produktionssysteme nur dort einkaufen. Viertens: *Prüfsummen und Signaturen* stellen sicher, dass die geladene Datei die erwartete ist.

### Quellen und Vertrauen.

Der erste Aufruf von `Install-PSResource` aus der PowerShell Gallery fragt standardmäßig nach, weil die Gallery nicht als vertrauenswürdig markiert ist. Im Test zeigte `Get-PSResourceRepository` den Eintrag `PSGallery` mit `Trusted = False` und Priorität 50. Dieses Verhalten ist erwünscht: Wer die Rückfrage mit `-TrustRepository` oder `Set-PSResourceRepository -Trusted` dauerhaft abschaltet, verzichtet auf eine Sicherung. Der Befehl `Install-PSResource` bietet laut Parameterliste unter anderem `-Version`, `-Prerelease`, `-Scope`, `-TrustRepository`, `-Reinstall`, `-Quiet`, `-AcceptLicense` und `-AuthenticodeCheck`; Letzteres prüft die Authenticode-Signatur der installierten Dateien (unter Windows). Mit `Register-PSResourceRepository` richtet man ein internes Repository ein, mit `-Priority` legt man fest, welche Quelle zuerst befragt wird; ein niedriger Wert hat Vorrang. Wichtig ist, dass bei mehreren Quellen für denselben Namen die Reihenfolge entscheidet, und dass ein öffentlicher Katalog mit gleichem Modulnamen ein internes Modul nicht überlagern darf (Angriff über Namensverwechslung, *Dependency Confusion*).

```powershell
# Repositories und Vertrauensstatus anzeigen
Get-PSResourceRepository | Select-Object Name, Uri, Trusted, Priority

# Internes Repository registrieren, Prioritaet vor der Gallery
Register-PSResourceRepository -Name Intern -Uri 'https://nuget.example.org/v3/index.json' `
    -Trusted -Priority 10

# Eine bestimmte Version aus einer bestimmten Quelle installieren
Install-PSResource -Name Contoso.Tools -Version '2.1.0' -Repository Intern -Scope CurrentUser

# Installierte Resourcen auflisten
Get-InstalledPSResource | Select-Object Name, Version, Repository
```

Der erste Befehl zeigt die Quellen. Der zweite registriert ein internes Repository (die Adresse ist ein erfundenes Beispiel) als vertrauenswürdig mit hoher Priorität. Der dritte installiert eine feste Version aus genau dieser Quelle; `-Repository` verhindert, dass ein gleichnamiges Modul aus der Gallery gezogen wird. Der vierte zeigt, was installiert ist und woher es kam. Die Registrierung und die Installation aus dem Katalog konnten im Arbeitsbereich nicht ausgeführt werden; die Befehle sind syntaxgeprüft.

### Versionen festlegen.

Auch unabhängig vom Katalog lässt sich festlegen, welche Version ein Skript nutzt. Mit `Import-Module -RequiredVersion` oder der Anweisung `#Requires -Modules` bindet man ein Skript an eine Version. Im Test lagen zwei Versionen eines Testmoduls (`1.0.0` und `2.0.0`) im Modulpfad. `Import-Module Demo` lud die neueste (2.0.0), `-RequiredVersion 1.0.0` lud die ältere, und `-FullyQualifiedName @{ModuleName='Demo';ModuleVersion='1.5.0'}` verlangte mindestens 1.5.0 und lud 2.0.0. Eine nicht vorhandene Version (3.0.0) scheiterte mit „...was not loaded because no valid module file was found in any module directory.“ Ein Skript mit `#Requires -Modules @{ModuleName='Demo';RequiredVersion='1.0.0'}` lief, ein Skript mit `RequiredVersion='9.9.9'` brach mit „...the following modules that are specified by the "#requires" statements of the script are missing: The module ’Demo’ cannot be found with RequiredVersion ’9.9.9’.“ ab, bevor eine Zeile lief.

```powershell
#Requires -Version 7.2
#Requires -Modules @{ ModuleName = 'Demo'; RequiredVersion = '1.0.0' }

Import-Module -Name Demo -RequiredVersion 1.0.0 -ErrorAction Stop
Get-Ver

# Pruefsumme einer heruntergeladenen Datei gegen den veroeffentlichten Wert pruefen
$erwartet = 'DEIN-GEPRUEFTER-SHA256-WERT'
$ist = (Get-FileHash -LiteralPath .\paket.zip -Algorithm SHA256).Hash
if ($ist -ne $erwartet) { throw 'Pruefsumme stimmt nicht: Datei nicht verwenden.' }
```

Die `#Requires`-Zeilen stoppen das Skript, wenn PowerShell oder das Modul nicht passen. `Import-Module -RequiredVersion` wiederholt die Festlegung beim Laden. Der zweite Block zeigt die Prüfsumme: `Get-FileHash` berechnet den SHA-256-Wert; im Test stimmte er nach dem Schreiben überein („Hash passt“) und wich nach dem Anhängen eines einzigen Zeichens ab („Hash weicht ab“). Der erwartete Wert muss aus einer *anderen* Quelle stammen als die Datei, zum Beispiel von der Projektseite über eine gesicherte Verbindung; ein Wert, der neben der Datei liegt, beweist nichts.

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Maßnahmen und Hinweise.** PowerShell-Installationspakete enthalten ab Version 7.2 eine Software-Stückliste (SBOM, Software Bill of Materials); laut „PowerShell security features“ liegt sie unter `$PSHOME/_manifest/spdx_2.2/manifest.spdx.json`, bei Modulen im Ordner `_manifest/spdx_2.2/manifest.spdx.json` des Moduls. Solche Stücklisten erleichtern die Frage, welche Komponenten in welcher Version in einer Umgebung stecken. Weitere übliche Maßnahmen, die hier aus allgemeinem Fachwissen stammen und nicht abgerufen wurden: die Abhängigkeiten eines Repositories regelmäßig auf bekannte Schwachstellen prüfen, Pipelines mit minimalen Rechten und kurzlebigen Anmeldedaten ausführen, Module vor der Freigabe in einer Testumgebung prüfen und Änderungen am internen Repository nur über Reviews zulassen.

> **Merke**
>
> Lieferkette: Herkunft prüfen, Versionen festlegen (`-RequiredVersion`, `#Requires`), interne Repositories mit Priorität, `PSGallery` nicht pauschal als vertrauenswürdig markieren, Signatur- und Prüfsummenkontrolle (`Get-FileHash`), Stücklisten (SBOM). Dependency Confusion durch Quellenangabe und Priorität verhindern.

```powershell
# FALSCH: Immer die neueste Version, Gallery pauschal vertraut
Set-PSResourceRepository -Name PSGallery -Trusted
Install-PSResource -Name BeliebigesModul -Reinstall

# RICHTIG: Feste, geprueft freigegebene Version aus dem internen Repository
Install-PSResource -Name BeliebigesModul -Version '3.2.1' -Repository Intern -Scope CurrentUser
```

Im ersten Fall zieht jeder Lauf unbesehen die jeweils aktuelle Version aus einer Quelle, auf die jeder Autor Pakete hochladen kann. Im zweiten Fall ändert sich nichts ohne Freigabe.

> **Praxisbeispiel**
>
> Eine Pipeline installiert vor dem Testlauf drei Module. Bisher lud sie ohne Versionsangabe die jeweils neueste aus der Gallery. Nach einem Vorfall bei einem Drittanbieter stellt das Team auf ein internes Repository um, das nur nach Prüfung befüllt wird, und legt die Versionen in einer Datei im Repository fest. Eine Aktualisierung ist jetzt eine bewusste, nachvollziehbare Änderung mit Review.

## 8.10 Übungsaufgaben zu Modul 08

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Windows-spezifische Aufgaben sollten in einer Testumgebung ausgeführt werden.

**Aufgabe 1.** Ein Kollege sagt: „Wir haben `AllSigned` gesetzt, also kann kein fremder Code laufen.“ Nennen Sie zwei Wege, auf denen Code trotzdem ausgeführt werden kann, und erklären Sie, was die Execution Policy tatsächlich leistet.

**Aufgabe 2.** Wie ändert sich die Aussage von `Get-ExecutionPolicy` auf Linux, und warum? Welche Rangfolge gilt unter Windows für die Bereiche?

**Aufgabe 3.** Erklären Sie den Unterschied zwischen CLM als Sicherheitsgrenze und CLM als gestaffelter Verteidigung. Warum konnte man im Test die Sitzung nicht mehr auf `FullLanguage` zurückstellen, und warum hilft das einem Angreifer trotzdem nicht weiter, wenn er die Einstellung nur selbst setzt?

**Aufgabe 4.** Ein Skript speichert ein Dienstkonto mit `Export-Clixml` und liest es später wieder ein. Es läuft auf Windows und Linux. Beurteilen Sie die Sicherheit auf beiden Plattformen und nennen Sie zwei bessere Alternativen.

**Aufgabe 5.** Schreiben Sie eine Funktion `Test-SkriptSignatur`, die für eine Datei die Signatur prüft und `$true` nur zurückgibt, wenn der Status `Valid` ist und der Fingerabdruck des Signierers in einer übergebenen Liste erlaubter Fingerabdrücke steht. Sonst soll sie einen aussagekräftigen Fehler auslösen. (Windows, nur Syntax prüfbar.)

**Aufgabe 6.** Schreiben Sie eine Funktion `Get-BerichtDatei`, die aus einem Basisordner eine Datei per relativem Namen liest. Die Funktion muss Path Traversal verhindern, `-LiteralPath` verwenden, den Dateinamen mit einer Allowlist (Buchstaben, Ziffern, Bindestrich, Unterstrich, Punkt, Endung `.txt` oder `.csv`) prüfen und Fehler mit `ThrowTerminatingError` melden.

**Aufgabe 7.** Schreiben Sie ein Skript, das einen Satz von Modulen und Versionen aus einer JSON-Datei liest (Name, Version, Repository), prüft, dass jede Version fest angegeben ist (kein Platzhalter, kein leerer Wert) und dass die Quelle in einer Liste erlaubter Repositories steht, und die Installationsbefehle zunächst nur mit `-WhatIf`-ähnlicher Ausgabe anzeigt (ohne Installation).

**Aufgabe 8.** Entwerfen Sie ein Konzept für die Sicherung eines Skripts, das täglich um 02:00 Uhr ein Backup auf einem Dateiserver ausführt: Welches Konto, welche Rechte, wo liegt das Geheimnis, wie wird das Skript vor Änderung geschützt, was wird protokolliert, was darf nicht im Protokoll stehen? Begründen Sie jede Entscheidung in ein bis zwei Sätzen.

## 8.11 Lösungen zu Modul 08

### Aufgabe 1.

Erstens kann ein Angreifer den Code als Text ausführen (`Invoke-Expression`, Eingabe in die Konsole, `[scriptblock]::Create`); die Policy betrifft nur Skriptdateien. Zweitens kann er einen neuen Prozess mit anderer Einstellung starten (`pwsh -ExecutionPolicy Bypass` oder `Set-ExecutionPolicy -Scope Process`), solange keine Gruppenrichtlinie das verhindert. Die Execution Policy schützt vor *Versehen*: Sie verhindert, dass Benutzer unbeabsichtigt Skripte aus unbekannter Quelle ausführen. Sie ist laut Microsoft gestaffelte Verteidigung und keine Sicherheitsgrenze.

### Aufgabe 2.

Auf Linux und macOS liefert `Get-ExecutionPolicy` immer `Unrestricted`, das sich nicht ändern lässt (im Test: `Set-ExecutionPolicy` scheitert mit „Operation is not supported on this platform.“); das Verhalten entspricht `Bypass`, weil die Windows-Sicherheitszonen fehlen. Unter Windows gilt die Rangfolge `MachinePolicy`, `UserPolicy`, `Process`, `CurrentUser`, `LocalMachine`; die Gruppenrichtlinien stehen vorn und lassen sich nicht überschreiben.

### Aufgabe 3.

Als Sicherheitsgrenze gilt CLM nur zusammen mit App Control for Business im System-Lockdown: Dann bestimmt die Richtlinie des Systems den Modus jeder Sitzung, und der Benutzer kann ihn nicht ändern. Als gestaffelte Verteidigung gelten CLM mit AppLocker, über eine Sitzungskonfiguration oder durch manuelles Setzen. Im Test blieb die Sitzung gefangen, weil die Eigenschaft des Sitzungszustands im CLM nur auf erlaubten Typen gesetzt werden darf und der Sprachmodus selbst dazu gehört („Cannot set property...“). Das hilft nicht, weil ein Angreifer mit eigenen Rechten einfach einen neuen `pwsh`-Prozess startet, der wieder mit `FullLanguage` beginnt, solange keine Application Control das erzwingt.

### Aufgabe 4.

Unter Windows verschlüsselt DPAPI die Datei; sie ist nur für dasselbe Konto auf demselben Rechner lesbar, schützt aber nicht vor Prozessen desselben Benutzers. Unter Linux ist der Inhalt *nicht verschlüsselt*: Im Test stand das Kennwort als Hexfolge der UTF-16-Zeichen in der Datei, und jeder Leser des Dateisystems kann es lesen. Beides ist für sensible Konten unzureichend. Besser: ein Tresor (SecretManagement mit Key Vault oder SecretStore) oder eine Identität ohne Kennwort (Managed Identity, Zertifikatsanmeldung).

### Aufgabe 5.

```powershell
function Test-SkriptSignatur {
    [CmdletBinding()]
    [OutputType([bool])]
    param(
        [Parameter(Mandatory)] [string] $Pfad,
        [Parameter(Mandatory)] [string[]] $ErlaubteFingerabdruecke
    )
    $sig = Get-AuthenticodeSignature -LiteralPath $Pfad
    if ($sig.Status -ne 'Valid') {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.Security.SecurityException]::new("Signatur nicht gueltig ($($sig.Status)): $Pfad"),
            'Signatur.Ungueltig',
            [System.Management.Automation.ErrorCategory]::SecurityError,
            $Pfad)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
    $fp = $sig.SignerCertificate.Thumbprint
    if ($ErlaubteFingerabdruecke -notcontains $fp) {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.Security.SecurityException]::new("Fremder Signierer ($fp): $Pfad"),
            'Signatur.Fremd',
            [System.Management.Automation.ErrorCategory]::SecurityError,
            $Pfad)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
    $true
}
```

Die Funktion prüft zuerst den Status (nur `Valid` zählt, nicht `UnknownError`, `NotSigned` oder `HashMismatch`) und dann, ob der Fingerabdruck des Signierers in der Liste erlaubter Zertifikate steht. Beide Fälle liefern einen typisierten Fehler mit Kennung. Der Aufrufer kann `try/catch` nutzen und über `FullyQualifiedErrorId` unterscheiden. Syntaxprüfung erfolgte, ausgeführt werden kann die Funktion nur unter Windows.

### Aufgabe 6.

```powershell
function Get-BerichtDatei {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Basis,
        [Parameter(Mandatory)]
        [ValidatePattern('^[A-Za-z0-9_.-]{1,100}\.(txt|csv)$')]
        [string] $Name
    )
    $trenner   = [System.IO.Path]::DirectorySeparatorChar
    $basisVoll = [System.IO.Path]::GetFullPath($Basis).TrimEnd($trenner) + $trenner
    $ziel      = [System.IO.Path]::GetFullPath((Join-Path $Basis $Name))

    if (-not $ziel.StartsWith($basisVoll, [System.StringComparison]::Ordinal)) {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.UnauthorizedAccessException]::new("Pfad ausserhalb von $Basis"),
            'Bericht.Pfad', [System.Management.Automation.ErrorCategory]::SecurityError, $Name)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
    if (-not (Test-Path -LiteralPath $ziel -PathType Leaf)) {
        $fehler = [System.Management.Automation.ErrorRecord]::new(
            [System.IO.FileNotFoundException]::new("Datei fehlt: $Name"),
            'Bericht.Fehlt', [System.Management.Automation.ErrorCategory]::ObjectNotFound, $Name)
        $PSCmdlet.ThrowTerminatingError($fehler)
    }
    Get-Content -LiteralPath $ziel -Raw
}
```

Das Muster in `ValidatePattern` erlaubt nur harmlose Zeichen und die Endungen `.txt` oder `.csv`; Schrägstriche und Doppelpunkte sind nicht enthalten, sodass `../x.txt` schon an der Parameterbindung scheitert. Die Pfadprüfung bleibt als zweite Schicht bestehen (gestaffelte Verteidigung), weil ein späterer Änderer das Muster lockern könnte. `-LiteralPath` verhindert Platzhalter, und die Fehler werden typisiert gemeldet.

### Aufgabe 7.

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory)] [string] $Konfig,
    [string[]] $ErlaubteRepositories = @('Intern')
)
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$eintraege = Get-Content -LiteralPath $Konfig -Raw | ConvertFrom-Json
$probleme = [System.Collections.Generic.List[string]]::new()

foreach ($e in $eintraege) {
    if ([string]::IsNullOrWhiteSpace($e.Version) -or $e.Version -notmatch '^\d+\.\d+\.\d+(-[0-9A-Za-z.-]+)?$') {
        $probleme.Add("$($e.Name): Version nicht fest angegeben ('$($e.Version)')")
    }
    if ($e.Repository -notin $ErlaubteRepositories) {
        $probleme.Add("$($e.Name): Repository nicht erlaubt ('$($e.Repository)')")
    }
}
if ($probleme.Count -gt 0) {
    $probleme | ForEach-Object { Write-Error $_ -ErrorAction Continue }
    exit 1
}
foreach ($e in $eintraege) {
    "Wuerde installieren: Install-PSResource -Name $($e.Name) -Version $($e.Version) -Repository $($e.Repository) -Scope CurrentUser"
}
```

Das Skript sammelt alle Probleme, bevor es abbricht, damit der Benutzer sie in einem Lauf sieht. Die Regex lässt nur feste Versionen im Format `Major.Minor.Patch` (optional mit Vorabversion) zu; Platzhalter wie `*` oder leere Werte fallen durch. Erst wenn alles gültig ist, gibt es die Befehle aus, ohne sie auszuführen. Eine Beispieldatei: `[ { "Name": "Contoso.Tools", "Version": "2.1.0", "Repository": "Intern" } ]`.

### Aufgabe 8.

Konto: ein eigenes gruppenverwaltetes Dienstkonto (gMSA) oder ein eigenes lokales Dienstkonto nur für diese Aufgabe, weil sein Kennwort nicht in einem Skript steht und es nur diese Aufgabe kann. Rechte: Leserecht auf die Quelle, Schreibrecht auf das Ziel, „Anmelden als Stapelverarbeitungsauftrag“, keine Administratorrechte; so bleibt der Schaden bei einem Fehler auf diese Ordner begrenzt. Geheimnis: wenn möglich keines (Windows-Authentifizierung des Dienstkontos); sonst Tresor über SecretManagement oder Key Vault, damit nichts im Skript, Repository oder Verlauf steht. Schutz vor Änderung: Skript im Repository mit Review, in der Pipeline signiert, auf dem Server `AllSigned` und Schreibrechte für das Skriptverzeichnis nur für die Pipeline; so fällt eine Änderung auf dem Server sofort auf. Protokoll: strukturiertes Log mit Run-ID, Quelle, Ziel, Dateianzahl, Dauer und Fehlern (Lerneinheit 5.6), Exit Code für den Planer, Script Block Logging auf dem Server mit zentraler Auswertung. Nicht ins Protokoll: Kennwörter, Tokens, Dateiinhalte und nach Möglichkeit keine personenbezogenen Daten in Dateinamen; Aufbewahrungsdauer festlegen und mit dem Datenschutz abstimmen.
