# Modul 10 - Cloud, Microsoft 365 und Azure Automation

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „PowerShell – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze bzw. Überschriften). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (10.1 bis 10.9; 10.10 Übungsaufgaben, 10.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt:** Laut Einleitung hat der Autor sie mit PowerShell 7.6.6 unter Linux ausgeführt; was nur unter Windows oder mit Zugriff auf die PowerShell Gallery läuft, ist im Text als „nur auf Syntax geprüft“ gekennzeichnet. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand (Dienste, Registry, Remoting, Cloud-Ressourcen): nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

|                                       |                                      |
|:--------------------------------------|:-------------------------------------|
| Dauer laut Übersichtsblatt            | 12 h                                 |
| Summe der Lerneinheiten (Detailblatt) | 9 h (9 Lerneinheiten à 1 h)          |
| Priorität                             | 2 (einzelne Lerneinheiten: 2 bzw. 3) |

> **Merke**
>
> **Stundenabweichung im Lehrplan.** Das Übersichtsblatt nennt 12 h, die neun Lerneinheiten des Detailblatts ergeben zusammen nur 9 h. Dieses Skript folgt dem Detailblatt, weil nur dort die einzelnen Lerneinheiten stehen. Die Differenz von 3 h bietet sich für praktische Übungen in einem Test-Mandanten an, denn die Cloud-Themen dieses Moduls lassen sich nur mit eigenem Zugang wirklich ausprobieren.

### Lernziele.

Nach diesem Modul können Sie sich sicher an Azure und Microsoft 365 anmelden, die wichtigsten Modulfamilien (Az, Microsoft Graph, Exchange Online, Teams, SharePoint) einordnen und gezielt einsetzen, Skripte als Runbooks in Azure Automation betreiben, Zugangsdaten über Managed Identities und Key Vault bereitstellen, statt sie im Skript zu speichern, die Cloud mit Tags, Richtlinien und Rollenprüfungen steuern, eigene Schnittstellenaufrufe mit Seitenabruf und Authentifizierung bauen und aus Kostendaten verlässliche Berichte erzeugen.

| **Nr.** | **Lerneinheit**                | **Dauer** | **Prio** |
|:--------|:-------------------------------|:----------|:---------|
| 10.1    | Azure PowerShell Grundlagen    | 1 h       | 2        |
| 10.2    | Microsoft Graph PowerShell     | 1 h       | 2        |
| 10.3    | Exchange Online PowerShell     | 1 h       | 2        |
| 10.4    | Teams und SharePoint Online    | 1 h       | 2        |
| 10.5    | Azure Automation               | 1 h       | 2        |
| 10.6    | Managed Identity und Key Vault | 1 h       | 2        |
| 10.7    | Cloud Governance               | 1 h       | 3        |
| 10.8    | API-Integration                | 1 h       | 3        |
| 10.9    | Kosten- und Nutzungsreports    | 1 h       | 3        |
|         | **Summe**                      | **9 h**   |          |

> **Ergänzung aus der Nachrecherche**
>
> **Wie dieses Modul getestet wurde, und was nicht.** Die Cloud-Module (Az, Microsoft Graph, Exchange Online, Teams, PnP) sowie Azure Automation und Key Vault waren im Arbeitsbereich *nicht* nutzbar: Es gab weder einen Mandanten noch einen Zugang, und der Katalog der PowerShell Gallery war nicht erreichbar. Befehle mit Cloud-Bezug sind deshalb nur auf Syntax geprüft und mit „nur auf Syntax geprüft“ gekennzeichnet. Ihre Parameter und Abläufe stützen sich, soweit angegeben, auf die für dieses Modul abgerufenen Seiten der Microsoft-Dokumentation (Stand Oktober 2026); alles andere stammt aus Fachwissen und ist als solches gekennzeichnet. Getestet mit PowerShell 7.6.6 unter Linux wurden dagegen die Teile, die reine Programmlogik sind: der Schutz vor der falschen Subscription (mit simuliertem Kontext), der Seitenabruf entlang von `@odata.nextLink` (mit simulierter Schnittstelle), der Aufbau der Token-Anfrage (ohne sie zu senden), die Umwandlung von Größenangaben, die Namensprüfung mit Probelauf, der Umgang mit Geheimnissen als `SecureString`, die Tag-Prüfung und die Kostenauswertung auf Beispieldaten.

## 10.1 Azure PowerShell Grundlagen

*Microsoft Azure* ist die Cloud-Plattform von Microsoft: virtuelle Maschinen, Speicher, Datenbanken, Netzwerke und viele weitere Dienste. Alles, was man im Azure-Portal anklicken kann, lässt sich auch per Skript steuern, und genau das ist der Sinn von Automatisierung in der Cloud: Ressourcen reproduzierbar anlegen, ändern, prüfen und löschen. Das PowerShell-Werkzeug dafür ist das Modul *Az* (Azure PowerShell). Es besteht aus vielen Untermodulen (`Az.Accounts`, `Az.Resources`, `Az.Compute`, `Az.Storage`, `Az.KeyVault` und weiteren), die man gemeinsam oder einzeln installiert. Die Befehle folgen dem bekannten Schema *Verb-Substantiv* mit dem Präfix `Az` im Substantiv, etwa `Get-AzResourceGroup` oder `New-AzVM`. Das ältere Modul AzureRM ist veraltet und wird nicht mehr verwendet; auch dieser Hinweis stammt aus Fachwissen.

Zum Aufbau von Azure gehören einige Begriffe, die man kennen muss, weil jede Anmeldung und jeder Befehl sich darauf bezieht. Ein *Mandant* (englisch *Tenant*) ist die Organisation, die das Verzeichnis (Microsoft Entra ID, früher Azure Active Directory) besitzt, in dem Benutzer, Gruppen und Anwendungen verwaltet werden. Eine *Subscription* (Abonnement) ist der Abrechnungs- und Verwaltungsrahmen für Ressourcen; ein Mandant kann viele Subscriptions haben. Eine *Ressourcengruppe* ist ein Container, in dem zusammengehörige Ressourcen liegen und gemeinsam verwaltet oder gelöscht werden. Eine *Ressource* ist das einzelne Ding: eine virtuelle Maschine, ein Speicherkonto, ein Tresor. Jede Ressource gehört zu genau einer Ressourcengruppe in genau einer Subscription.

Die Anmeldung geschieht mit `Connect-AzAccount`. Laut der abgerufenen Dokumentation öffnet der Befehl bei interaktiver Verwendung den Anmeldedialog; in Az ab Version 12 nutzt er unter Windows den Windows-Kontenvermittler (WAM), unter Linux und macOS die Anmeldung im Browser. Wichtig für die Automatisierung ist ein Hinweis in derselben Dokumentation: Ab September 2025 verlangt Azure PowerShell für *Benutzeridentitäten* die Mehrfaktor-Authentifizierung (MFA). Das betrifft Skripte, die sich mit Benutzername und Kennwort anmelden; sie werden nicht mehr funktionieren. Für unbeaufsichtigte Abläufe sind stattdessen *Dienstidentitäten* vorgesehen: ein Dienstprinzipal (mit Zertifikat oder föderiertem Token) oder eine Managed Identity (siehe 10.6). Die Dokumentation von `Connect-AzAccount` beschreibt außerdem die Anmeldung als Dienstprinzipal mit Zertifikat.

```powershell
# Nur auf Syntax geprueft - benoetigt einen Azure-Zugang
Install-Module Az -Scope CurrentUser          # oder: Install-PSResource Az
Connect-AzAccount                             # interaktiv (MFA)
Get-AzSubscription | Select-Object Name, Id, TenantId
Set-AzContext -Subscription 'Entwicklung'     # Arbeitskontext festlegen
Get-AzContext | Select-Object Name, Account, Subscription, Tenant

Get-AzResourceGroup | Select-Object ResourceGroupName, Location
Get-AzResource -ResourceGroupName 'rg-demo' |
    Select-Object Name, ResourceType, Location, Tags
```

Das Beispiel installiert das Modul für den aktuellen Benutzer (ohne Administratorrechte), meldet sich an, listet die verfügbaren Subscriptions und wählt eine als *Kontext* aus. Der Kontext (`Get-AzContext`) ist der wichtigste Begriff für die Sicherheit: Er legt fest, in welcher Subscription und welchem Mandanten alle folgenden Befehle wirken. Danach folgen lesende Abfragen nach Ressourcengruppen und Ressourcen. Es sind nur lesende Befehle, ein sinnvoller Start in jeder Umgebung.

Weil der Kontext so entscheidend ist, lohnt sich eine Sicherung gegen den Klassiker aller Cloud-Unfälle: den richtigen Befehl in der falschen Subscription ausführen. Eine kleine Schutzfunktion prüft vor jeder verändernden Aktion, ob man wirklich in der erwarteten Umgebung steht.

```powershell
function Assert-AzKontext {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $ErwarteteSubscriptionId,
        [string[]] $ErlaubteMandanten = @(),
        [scriptblock] $KontextQuelle = { Get-AzContext }
    )
    $k = & $KontextQuelle
    if (-not $k) { throw 'Keine Azure-Anmeldung vorhanden. Zuerst Connect-AzAccount ausführen.' }
    if ($k.Subscription.Id -ne $ErwarteteSubscriptionId) {
        throw "Falsche Subscription: '$($k.Subscription.Name)' ($($k.Subscription.Id)), erwartet $ErwarteteSubscriptionId."
    }
    if ($ErlaubteMandanten.Count -gt 0 -and $k.Tenant.Id -notin $ErlaubteMandanten) {
        throw "Mandant $($k.Tenant.Id) ist nicht freigegeben."
    }
    $k
}

# Aufruf am Skriptanfang:
$null = Assert-AzKontext -ErwarteteSubscriptionId $env:ZIEL_SUBSCRIPTION
```

Die Funktion holt den aktuellen Kontext über einen austauschbaren Skriptblock (Standard: `Get-AzContext`); das macht sie testbar, ohne dass eine Anmeldung nötig ist. Fehlt der Kontext, bricht sie mit einer klaren Anleitung ab. Stimmt die Subscription-Kennung nicht, nennt die Fehlermeldung beide Seiten (gefundene und erwartete), damit man sofort sieht, was los ist. Optional prüft sie zusätzlich den Mandanten. Getestet wurde die Logik mit simulierten Kontexten: Der passende Kontext lieferte „Produktion“ zurück; ein Kontext mit anderer Subscription erzeugte die Meldung „Falsche Subscription: ’Test’ (sub-test), erwartet sub-prod.“; ein fehlender Kontext die Anmeldeaufforderung; ein nicht freigegebener Mandant die Meldung „Mandant t1 ist nicht freigegeben.“. Der echte Aufruf mit `Get-AzContext` wurde nicht getestet.

Ein weiterer Grundsatz für jeden verändernden Befehl: zuerst mit `-WhatIf` anzeigen lassen, was passieren würde. Az-Befehle, die etwas anlegen, ändern oder löschen, unterstützen in der Regel die gemeinsamen Parameter `-WhatIf` und `-Confirm` (aus Modul 5 bekannt), wobei die Unterstützung je Cmdlet unterschiedlich ist; ob ein bestimmtes Cmdlet `-WhatIf` kann, zeigt `Get-Help` oder die Dokumentation. Wer Ressourcen löscht, prüft vorher mit `Get-AzResource`, was der Filter tatsächlich trifft.

> **Ergänzung aus der Nachrecherche**
>
> **Kontext pro Prozess und Automatisierung.** Az speichert den Anmeldekontext standardmäßig zwischen Sitzungen. In unbeaufsichtigten Abläufen (Runbooks, Pipelines) ist das unerwünscht, weil ein Kontext eines früheren Laufs „überleben“ kann. Die Dokumentation zu Azure Automation zeigt deshalb in ihren Runbook-Beispielen den Aufruf `Disable-AzContextAutosave -Scope Process` vor der Anmeldung (siehe 10.5). Auch das Setzen des Kontexts ist dort ausdrücklich Teil des Musters: Nach `Connect-AzAccount` folgt `Set-AzContext`, damit der Lauf in einer bekannten Subscription arbeitet.

> **Merke**
>
> Az besteht aus Untermodulen; Befehle heißen `Verb-Az...`. Mandant, Subscription und Ressourcengruppe bilden die Hierarchie, der *Kontext* legt fest, wo ein Befehl wirkt. Interaktiv meldet man sich mit `Connect-AzAccount` an, unbeaufsichtigt mit Dienstidentität (ab September 2025 verlangt Azure PowerShell MFA für Benutzeridentitäten). Vor verändernden Befehlen den Kontext prüfen und `-WhatIf` nutzen.

```powershell
# FALSCH: veraendernder Befehl ohne Kontextpruefung, Filter nicht kontrolliert
Get-AzResource -Name '*test*' | Remove-AzResource -Force

# RICHTIG: Kontext pruefen, erst anzeigen, dann mit WhatIf, dann gezielt loeschen
$null = Assert-AzKontext -ErwarteteSubscriptionId $env:ZIEL_SUBSCRIPTION
$ziele = Get-AzResource -ResourceGroupName 'rg-demo' -Name 'vm-test-*'
$ziele | Select-Object Name, ResourceType
$ziele | Remove-AzResource -WhatIf
```

Die falsche Variante löscht alles, dessen Name „test“ enthält, in der gerade aktiven Subscription, womöglich der falschen, und ohne Rückfrage. Die richtige Variante begrenzt den Bereich auf eine Ressourcengruppe, zeigt die Treffer an und lässt den Löschbefehl nur simulieren. Erst wenn die Liste stimmt, entfernt man `-WhatIf`. Die Befehle sind nur auf Syntax geprüft.

> **Praxisbeispiel**
>
> Ein Administrator hat zwei Browserfenster offen, Test und Produktion, und meldet sich im Terminal versehentlich in der falschen Subscription an. Weil jedes Skript des Teams mit `Assert-AzKontext` beginnt, bricht der Löschbefehl mit „Falsche Subscription“ ab, bevor etwas passiert. Die Schutzfunktion kostet drei Zeilen und verhindert einen Ausfall.

## 10.2 Microsoft Graph PowerShell

*Microsoft Graph* ist die einheitliche Programmierschnittstelle für Microsoft 365: Benutzer, Gruppen, Postfächer, Kalender, Teams, SharePoint, Geräte und vieles mehr erreicht man über dieselbe Adresse (`https://graph.microsoft.com`) und mit demselben Anmeldeverfahren. Das *Microsoft Graph PowerShell SDK* verpackt diese Schnittstelle in Cmdlets, deren Namen das Präfix `Mg` tragen (`Get-MgUser`, `Get-MgGroup`, `New-MgUser`). Es löst die älteren Module *AzureAD* und *MSOnline* ab, die veraltet sind; auch das ist Fachwissen und nicht in den für dieses Modul abgerufenen Seiten belegt. Wer heute neue Skripte für Entra-ID und Microsoft 365 schreibt, beginnt mit Graph.

Der wichtigste Punkt beim Arbeiten mit Graph ist das Berechtigungsmodell. Anders als bei einem Administratorkonto, das „alles darf“, muss man für jede Art von Zugriff eine ausdrückliche Berechtigung (englisch *Scope* bzw. *Permission*) erteilen, etwa `User.Read.All` zum Lesen aller Benutzer oder `User.ReadWrite.All` zum Ändern. Es gibt zwei Arten der Anmeldung mit unterschiedlichen Berechtigungen. Bei der *delegierten* Anmeldung handelt das Skript im Namen eines angemeldeten Benutzers; es darf höchstens, was dieser Benutzer darf *und* was die gewährte Berechtigung erlaubt. Bei der *reinen Anwendungsanmeldung* (*app-only*) handelt das Skript als eigene Anwendung ohne Benutzer; sie ist für unbeaufsichtigte Abläufe gedacht, und die Berechtigungen wirken dann tenantweit, weshalb man sie besonders sparsam vergibt.

Die abgerufene Dokumentation zeigt den delegierten Einstieg mit `Connect-MgGraph -Scopes "User.Read.All"`; für Rechner ohne Browser gibt es `-UseDeviceAuthentication` (Anmeldung per Code auf einem anderen Gerät), und die app-only-Anmeldung wird ebenfalls unterstützt. Welche Berechtigung ein Cmdlet braucht, findet man mit `Find-MgGraphCommand`, das laut Dokumentation zu einem Cmdlet oder einer Adresse die benötigten Berechtigungen nennt. Eine Folge des Prinzips „so wenig wie nötig“: Man beginnt mit lesenden Berechtigungen und erweitert nur, wenn ein Befehl mit einer Berechtigungsmeldung scheitert.

```powershell
# Nur auf Syntax geprueft - benoetigt einen Microsoft-365-Mandanten
Install-Module Microsoft.Graph.Users -Scope CurrentUser
Connect-MgGraph -Scopes 'User.Read.All'              # delegiert, interaktiv
# Connect-MgGraph -Scopes 'User.Read.All' -UseDeviceAuthentication   # ohne Browser

Get-MgContext | Select-Object Account, TenantId, Scopes

Get-MgUser -Top 5 -Property DisplayName, UserPrincipalName, AccountEnabled |
    Select-Object DisplayName, UserPrincipalName, AccountEnabled

Get-MgUser -All -Property DisplayName, UserPrincipalName |
    Where-Object { $_.UserPrincipalName -like 'a*' } | Measure-Object

# Erweiterte Abfrage (Filter auf bestimmte Eigenschaften): zusaetzliche Parameter noetig
Get-MgUser -Filter "endswith(userPrincipalName,'@example.org')" `
    -ConsistencyLevel eventual -Count userCount
"Treffer: $userCount"

Find-MgGraphCommand -Command Get-MgUser | Select-Object -First 1 -ExpandProperty Permissions
Disconnect-MgGraph
```

Das Beispiel installiert nur das Untermodul für Benutzer, statt des riesigen Gesamtpakets, und meldet sich delegiert mit der Berechtigung zum Lesen an. `Get-MgContext` zeigt, wer angemeldet ist und welche Berechtigungen gelten. `Get-MgUser -Top 5` holt nur fünf Benutzer, eine gute Probe. `-Property` begrenzt die abgefragten Felder, was die Antwort kleiner und schneller macht. `-All` sorgt dafür, dass das Cmdlet alle Seiten abruft; ohne `-All` liefert Graph standardmäßig nur eine Seite (typischerweise 100 Einträge). Der nächste Block zeigt die Besonderheit: Für „erweiterte Abfragen“, bei denen Graph spezielle Indizes verwenden muss (zum Beispiel Filter mit `endswith`), verlangt die Schnittstelle laut abgerufener Dokumentation den Parameter `-ConsistencyLevel eventual`, bei Zählungen zusätzlich `-Count` mit dem Namen einer Variablen, die das Ergebnis aufnimmt (die Dokumentation nennt `-Count userCount`; der Name wird ohne Dollarzeichen angegeben). Ob ein bestimmter Filter als erweiterte Abfrage gilt, hängt vom Feld und von der Modulversion ab und ist in der Dokumentation zu prüfen. Zuletzt zeigt `Find-MgGraphCommand` die nötigen Berechtigungen, und `Disconnect-MgGraph` beendet die Verbindung.

Was geschieht im Hintergrund, wenn „alle Seiten“ abgerufen werden? Graph liefert große Ergebnisse seitenweise. Jede Antwort enthält das Feld `value` mit den Einträgen der Seite und, falls es weitere gibt, das Feld `@odata.nextLink` mit der Adresse der nächsten Seite. Die SDK-Cmdlets folgen diesem Link bei `-All` selbst. Wer die Schnittstelle direkt mit `Invoke-MgGraphRequest` anspricht, muss das selbst tun. Dafür gibt es eine kleine, wiederverwendbare Funktion, die man testen kann, ohne einen Mandanten zu haben, indem man den Abruf austauschbar macht.

```powershell
function Get-GraphSeiten {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Uri,
        [scriptblock] $Abruf = { param($u) Invoke-MgGraphRequest -Method GET -Uri $u -OutputType PSObject },
        [int] $MaxSeiten = 1000
    )
    $seite = 0
    while ($Uri -and $seite -lt $MaxSeiten) {
        $seite++
        $antwort = & $Abruf $Uri
        $antwort.value
        $Uri = $antwort.'@odata.nextLink'
    }
    if ($Uri) { Write-Warning "Abbruch nach $MaxSeiten Seiten, es gibt weitere Daten." }
}

# Test mit simulierter Schnittstelle (drei Seiten: 3 + 3 + 2 Benutzer)
$aufrufe = [System.Collections.Generic.List[string]]::new()
$fake = { param($u) $aufrufe.Add($u); $seiten[$u] }
$alle = @(Get-GraphSeiten -Uri 'https://graph.example/users' -Abruf $fake)
"Benutzer: $($alle.Count), Aufrufe: $($aufrufe.Count)"
```

Die Funktion läuft in einer Schleife: Sie ruft die Adresse ab, gibt die Einträge der Seite aus (`$antwort.value`) und liest dann den Link zur nächsten Seite. Gibt es keinen, ist `$Uri` leer, und die Schleife endet. Ein Höchstwert für die Seitenzahl schützt vor einer Endlosschleife durch einen fehlerhaften Server; wird er erreicht, gibt die Funktion eine Warnung aus. Der Abruf ist als Skriptblock austauschbar (Standard: `Invoke-MgGraphRequest`). Im Test lieferte die simulierte Schnittstelle drei Seiten mit zusammen acht Benutzern; die Funktion rief die Schnittstelle dreimal auf und gab acht Benutzer zurück (die beiden letzten hießen „Benutzer 7“ und „Benutzer 8“). Mit `-MaxSeiten 2` endete sie nach zwei Aufrufen mit der Warnung „Abbruch nach 2 Seiten, es gibt weitere Daten.“. Die Hilfsvariable `$seiten` mit den simulierten Seiten ist im Beispiel ausgelassen; sie enthält je Adresse ein Objekt mit `value` und gegebenenfalls `@odata.nextLink`.

> **Ergänzung aus der Nachrecherche**
>
> **Berechtigungen und Einwilligung.** Bei der ersten Anmeldung mit neuen Berechtigungen verlangt Microsoft eine *Einwilligung* (Consent), bei weitreichenden Berechtigungen durch einen Administrator. Für app-only-Zugriffe sind Anwendungsberechtigungen nötig, die nur ein Administrator freigeben kann. Welche Berechtigung für welchen Befehl nötig ist, steht in der Dokumentation des jeweiligen Cmdlets und in `Find-MgGraphCommand`. Beachten Sie auch, dass Berechtigungsnamen und die Zuordnung zu Befehlen sich mit der Zeit ändern; prüfen Sie im Zweifel die aktuelle Dokumentation.

> **Merke**
>
> Graph ist die einheitliche Schnittstelle für Microsoft 365, das SDK heißt `Microsoft.Graph.*` mit Präfix `Mg`. Berechtigungen werden ausdrücklich erteilt (`-Scopes`). Delegiert heißt im Namen eines Benutzers, app-only als eigene Anwendung ohne Benutzer. `-All` ruft alle Seiten ab; für erweiterte Abfragen braucht Graph `-ConsistencyLevel eventual`. `Find-MgGraphCommand` nennt die nötigen Berechtigungen.

```powershell
# FALSCH: weitreichende Berechtigung "auf Verdacht" und alles abrufen
Connect-MgGraph -Scopes 'Directory.ReadWrite.All'
$u = Get-MgUser -All            # alle Felder, obwohl nur der Name gebraucht wird

# RICHTIG: kleinste noetige Berechtigung, nur benoetigte Felder
Connect-MgGraph -Scopes 'User.Read.All'
$u = Get-MgUser -All -Property DisplayName, UserPrincipalName |
         Select-Object DisplayName, UserPrincipalName
```

Die falsche Variante holt sich Schreibrechte für das gesamte Verzeichnis, obwohl nur gelesen wird; ein Fehler im Skript oder ein gestohlenes Token hätte weitreichende Folgen. Die richtige Variante beschränkt Rechte und Datenmenge.

> **Praxisbeispiel**
>
> Das Team erzeugt jeden Montag einen Bericht über deaktivierte Benutzerkonten mit lange nicht erfolgter Anmeldung. Das Skript braucht nur Leserechte; es läuft unbeaufsichtigt als Anwendung mit der Berechtigung zum Lesen von Benutzern. Ein späteres Skript, das Konten deaktiviert, bekommt eine *eigene* Anwendung mit Schreibrecht und strengerer Freigabe, damit ein Fehler im Berichtsskript nie Konten verändern kann.

## 10.3 Exchange Online PowerShell

*Exchange Online* ist der Postfachdienst von Microsoft 365. Für Verwaltungsaufgaben, die sich im Administrationsportal nur mühsam oder gar nicht erledigen lassen (Massenänderungen, Berichte, Weiterleitungsregeln, Berechtigungen auf Postfächer), gibt es das Modul *ExchangeOnlineManagement*. Es hat eine Besonderheit, die Neueinsteiger verwirrt: Die meisten Befehle werden nicht lokal aus dem Modul geladen, sondern stehen erst *nach der Verbindung* zur Verfügung, weil das Modul sie zur Laufzeit vom Dienst bezieht (moderne Versionen verwenden dafür eine REST-basierte Schnittstelle). Ohne `Connect-ExchangeOnline` gibt es deshalb kein `Get-Mailbox`; dass es das Modul und seine Befehle gibt, aber „der Befehl nicht gefunden wird“, ist ein häufiger Anfängerfehler. Diese Funktionsweise ist Fachwissen und nicht in den abgerufenen Seiten beschrieben.

Die interaktive Anmeldung ist einfach: `Connect-ExchangeOnline -UserPrincipalName admin@firma.example`. Für unbeaufsichtigte Skripte reicht das nicht, und die abgerufene Dokumentation beschreibt die beiden passenden Verfahren. Beim *zertifikatsbasierten app-only-Zugriff* meldet sich das Skript als Anwendung an. Dazu gehören laut Dokumentation: Eine Anwendung im Entra-Verzeichnis, ein Zertifikat, die Anwendungsberechtigung `Exchange.ManageAsApp` mit Administratoreinwilligung und eine Rolle (Entra- oder Exchange-Rolle), die der Anwendung zugewiesen wird. Der Aufruf lautet dann `Connect-ExchangeOnline -CertificateThumbPrint ... -AppID ... -Organization contoso.onmicrosoft.com`; die Optionen `-Certificate` (Objekt) und `-CertificateFilePath` (Datei mit Kennwort) sind Alternativen. Die Dokumentation weist auf zwei Stolperfallen hin: Die Variante mit dem *Fingerabdruck* (Thumbprint) funktioniert nur unter Windows, und das Zertifikat muss mit einem sogenannten CSP-Anbieter erzeugt sein, nicht mit CNG. Beim zweiten Verfahren, der *Managed Identity*, entfällt das Zertifikat: In Azure Automation oder auf einer Azure-Ressource lautet der Aufruf `Connect-ExchangeOnline -ManagedIdentity -Organization contoso.onmicrosoft.com`; auch hier gehören die Berechtigung `Exchange.ManageAsApp` und eine Rolle dazu.

```powershell
# Nur auf Syntax geprueft - benoetigt einen Exchange-Online-Mandanten
Install-Module ExchangeOnlineManagement -Scope CurrentUser

# a) interaktiv (Administrator)
Connect-ExchangeOnline -UserPrincipalName 'admin@contoso.onmicrosoft.com'

# b) unbeaufsichtigt, Anwendung mit Zertifikat (Fingerabdruck nur unter Windows)
Connect-ExchangeOnline -CertificateThumbPrint 'AB12...EF' `
    -AppID '00000000-0000-0000-0000-000000000000' `
    -Organization 'contoso.onmicrosoft.com'

# c) unbeaufsichtigt in Azure Automation, Managed Identity
Connect-ExchangeOnline -ManagedIdentity -Organization 'contoso.onmicrosoft.com'

try {
    Get-Mailbox -ResultSize 5 |
        Select-Object DisplayName, PrimarySmtpAddress, RecipientTypeDetails
    Get-MailboxStatistics -Identity 'anna@contoso.onmicrosoft.com' |
        Select-Object DisplayName, ItemCount, TotalItemSize
}
finally {
    Disconnect-ExchangeOnline -Confirm:$false
}
```

Das Beispiel zeigt die drei Anmeldearten nebeneinander; in der Praxis nutzt man genau eine. Der Platzhalter für den Fingerabdruck und die Kennung der Anwendung sind erfunden. Der `try/finally`-Block stellt sicher, dass die Verbindung auch bei einem Fehler wieder getrennt wird; das ist wichtig, weil Verbindungen einen begrenzten Vorrat an Sitzungen verbrauchen und bei Skripten, die oft laufen, sonst zum Engpass werden. `Get-Mailbox -ResultSize 5` begrenzt die Ausgabe, eine gute Probe, bevor man alle Postfächer abfragt. `Get-MailboxStatistics` liefert unter anderem die Zahl der Elemente und die belegte Größe.

Ein Detail aus dem Alltag: Die Größe eines Postfachs kommt als Zeichenkette zurück und nicht als Zahl, etwa in der Form `49.5 GB (53,150,220,288 bytes)`. In dieser Form (Anzeigetext mit den Bytes in Klammern) kenne ich sie aus der Praxis; sie ist nicht in den abgerufenen Seiten belegt, und die genaue Darstellung kann je nach Umgebung variieren. Damit sich rechnen lässt, wandelt man den Text in eine Zahl um. Die folgende Funktion tut das und wurde mit beiden Formen (mit und ohne Klammerangabe) getestet.

```powershell
function ConvertTo-Bytes {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)] [string] $Text)
    process {
        if ($Text -match '\(([\d,\.]+)\s*bytes\)') {
            [long]($Matches[1] -replace '[,\.]', '')
        }
        elseif ($Text -match '^\s*([\d\.,]+)\s*(B|KB|MB|GB|TB)\s*$') {
            $zahl   = [double]::Parse(($Matches[1] -replace ',', ''), [Globalization.CultureInfo]::InvariantCulture)
            $faktor = @{ B = 1; KB = 1KB; MB = 1MB; GB = 1GB; TB = 1TB }[$Matches[2]]
            [long]($zahl * $faktor)
        }
        else { throw "Unbekanntes Format: '$Text'" }
    }
}

'49.5 GB (53,150,220,288 bytes)' | ConvertTo-Bytes     # 53150220288
'1.5 GB' | ConvertTo-Bytes                              # 1610612736
```

Die Funktion probiert zuerst, die genaue Byte-Angabe in Klammern zu lesen; Kommas und Punkte als Tausendertrenner werden entfernt. Fehlt die Klammer, liest sie Zahl und Einheit und rechnet mit den PowerShell-Konstanten `1KB`, `1MB` und so weiter (Faktor 1 024). Die Zahl wird dabei bewusst mit der invarianten Kultur gelesen, damit die Funktion unter jeder Spracheinstellung gleich arbeitet. Ein unbekanntes Format führt zu einem klaren Fehler statt zu einer falschen Zahl. Test: Die erste Eingabe ergab 53 150 220 288, die zweite 1 610 612 736, die dritte (`100 MB (104,857,600 bytes)`) 104 857 600; der Text „viel“ erzeugte „Unbekanntes Format: ’viel“’. Mit zwei Beispielpostfächern ergab sich: Ein Postfach mit 49,5 von 50 GB war zu 99,0 Prozent belegt und löste die Warnung (über 90 Prozent) aus, ein anderes mit 2,1 GB war bei 4,2 Prozent.

> **Ergänzung aus der Nachrecherche**
>
> **Sitzungen sparen und zeitgemäß anmelden.** Pro Verbindung entsteht eine Sitzung; wer in einer Schleife für jedes Postfach neu verbindet, stößt schnell an Grenzen. Besser ist: einmal verbinden, viele Befehle ausführen, am Ende trennen. Außerdem gilt: Die Basisauthentifizierung (Benutzername und Kennwort ohne moderne Anmeldung) ist für Exchange Online abgeschafft; Skripte müssen moderne Verfahren verwenden (diese Aussage stammt aus allgemeinem Fachwissen, nicht aus den für dieses Modul abgerufenen Seiten). Welche Rolle der Anwendung zuzuweisen ist, richtet sich nach dem Prinzip der geringsten Rechte.

> **Merke**
>
> Exchange-Befehle stehen erst nach `Connect-ExchangeOnline` zur Verfügung. Unbeaufsichtigt: Anwendung mit Zertifikat (`-AppID`, `-Organization`; Fingerabdruck nur unter Windows, Zertifikat mit CSP) oder Managed Identity (`-ManagedIdentity`); beides braucht `Exchange.ManageAsApp`, Administratoreinwilligung und eine Rolle. Immer in `try/finally` mit `Disconnect-ExchangeOnline` arbeiten.

```powershell
# FALSCH: pro Postfach neu verbinden, Verbindung bleibt bei Fehler offen
foreach ($m in $adressen) {
    Connect-ExchangeOnline -ManagedIdentity -Organization $org
    Get-MailboxStatistics $m
}

# RICHTIG: einmal verbinden, im finally trennen
Connect-ExchangeOnline -ManagedIdentity -Organization $org
try     { $adressen | ForEach-Object { Get-MailboxStatistics -Identity $_ } }
finally { Disconnect-ExchangeOnline -Confirm:$false }
```

Die falsche Variante öffnet für jede Adresse eine neue Verbindung und räumt bei einem Fehler nicht auf; bei tausend Postfächern erschöpft das die Sitzungen. Die richtige Variante verbindet einmal.

> **Praxisbeispiel**
>
> Eine monatliche Auswertung meldet Postfächer über 90 Prozent Belegung an den Support. Das Skript läuft in Azure Automation mit Managed Identity, wandelt die Größenangaben mit der Funktion aus diesem Abschnitt in Zahlen, schreibt eine CSV-Datei und schickt eine Zusammenfassung ans Ticketsystem. Es braucht kein einziges gespeichertes Kennwort.

## 10.4 Teams und SharePoint Online

*Microsoft Teams* und *SharePoint Online* sind die Zusammenarbeitsdienste von Microsoft 365. Hinter jedem Team steckt eine Microsoft-365-Gruppe mit einer SharePoint-Website für die Dateien; Kanäle, Mitglieder, Besitzer und Einstellungen lassen sich per Skript verwalten. Typische Aufgaben für die Automatisierung sind das *Bereitstellen* (Provisionierung) neuer Teams nach einer Vorlage, das *Aufräumen* verwaister Teams, das Setzen einheitlicher Einstellungen und Berichte über Mitglieder und Gäste.

Für diese Dienste gibt es drei verbreitete Werkzeuge. Das Modul *MicrosoftTeams* bietet Befehle wie `Connect-MicrosoftTeams`, `Get-Team`, `New-Team`, `Add-TeamUser` und `Get-TeamChannel`. Die *SharePoint Online Management Shell* (Modul `Microsoft.Online.SharePoint.PowerShell`) bietet Verwaltungsbefehle auf Mandantenebene (`Connect-SPOService`, `Get-SPOSite`). Und *PnP PowerShell* (Modul `PnP.PowerShell`, eine Gemeinschaftsentwicklung) deckt Inhalte und Einstellungen innerhalb einer Website ab (`Connect-PnPOnline`, `Get-PnPList`, `Add-PnPListItem`) und gilt als besonders ausdrucksstark. Zusätzlich lässt sich vieles direkt über Microsoft Graph erledigen (siehe 10.2), und genau dorthin bewegt sich Microsoft: Neue Funktionen erscheinen zuerst in Graph. Diese ganze Einordnung stammt aus Fachwissen; die Dokumentation zu Teams, SharePoint und PnP wurde für dieses Modul nicht abgerufen, und die Befehle sind nur auf Syntax geprüft. Besonders bei PnP PowerShell ändern sich die Anmeldevorgaben (zum Beispiel die Notwendigkeit einer eigenen Anwendungsregistrierung) von Version zu Version; vor dem Einsatz ist die aktuelle Dokumentation des Projekts zu lesen.

Das Wichtigste an solchen Skripten ist nicht der einzelne Befehl, sondern die Disziplin drumherum: Eingaben prüfen, Namensregeln durchsetzen, Probelauf anbieten, Fehler einer Zeile von den anderen trennen. Ein typisches Beispiel ist die Massenanlage von Teams aus einer CSV-Datei. Hier sind drei Dinge zu beachten. Die Namen müssen einer Konvention folgen (zum Beispiel Präfix „PRJ-“, keine Sonderzeichen, begrenzte Länge), damit man später aufräumen kann. Jede Zeile wird einzeln geprüft, damit eine fehlerhafte Zeile nicht alle anderen aufhält. Und der Lauf muss mit `-WhatIf` simulierbar sein, damit man das Ergebnis sieht, bevor etwas entsteht.

```powershell
function Test-Bezeichnung {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)] [string] $Name, [string] $Praefix = 'PRJ-')
    process {
        $probleme = @()
        if (-not $Name.StartsWith($Praefix)) { $probleme += "Praefix $Praefix fehlt" }
        if ($Name.Length -gt 50) { $probleme += 'laenger als 50 Zeichen' }
        if ($Name -match '[^\p{L}\p{Nd}\- ]') { $probleme += 'unerlaubte Zeichen' }
        [pscustomobject]@{ Name = $Name; Gueltig = ($probleme.Count -eq 0); Probleme = $probleme -join '; ' }
    }
}

function New-TeamsAusCsv {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $CsvPfad,
        [scriptblock] $Erzeuger = {
            param($name, $besitzer)
            # nur auf Syntax geprueft: echte Anlage mit dem Teams-Modul
            New-Team -DisplayName $name -Owner $besitzer -Visibility Private
        }
    )
    foreach ($z in Import-Csv -LiteralPath $CsvPfad -Delimiter ';') {
        $pruefung = $z.Name | Test-Bezeichnung
        if (-not $pruefung.Gueltig) {
            Write-Warning "Zeile '$($z.Name)' uebersprungen: $($pruefung.Probleme)"
            continue
        }
        if ($PSCmdlet.ShouldProcess($z.Name, 'Team anlegen')) { & $Erzeuger $z.Name $z.Besitzer }
    }
}

# Aufruf: erst simulieren, dann ausfuehren
New-TeamsAusCsv -CsvPfad .\teams.csv -WhatIf
New-TeamsAusCsv -CsvPfad .\teams.csv
```

`Test-Bezeichnung` prüft drei Regeln. Der Name muss mit dem Präfix beginnen, darf höchstens 50 Zeichen haben und nur Buchstaben (`\p{L}`), Ziffern (`\p{Nd}`), Bindestrich und Leerzeichen enthalten; die Klasse mit `\p` erkennt Buchstaben aller Sprachen einschließlich Umlauten. Die Funktion gibt je Name ein Objekt mit dem Ergebnis und den Problemen zurück. `New-TeamsAusCsv` liest die CSV-Datei (Trennzeichen Semikolon, wie von deutschsprachigem Excel erzeugt), prüft jede Zeile und überspringt ungültige mit einer Warnung (`continue`), statt abzubrechen. `SupportsShouldProcess` schaltet die Parameter `-WhatIf` und `-Confirm` frei; `ShouldProcess` entscheidet pro Zeile, ob wirklich etwas getan wird. Der Erzeuger ist als Skriptblock austauschbar, sodass man die Logik ohne Mandanten testen kann.

Test mit einer CSV-Datei mit zwei Zeilen (`PRJ-Alpha` und `Beta`) und einem Erzeuger, der nur Text ausgibt: Der normale Lauf meldete „Team ’PRJ-Alpha’ mit Besitzer anna@example.org angelegt“ und die Warnung „Zeile ’Beta’ uebersprungen: Praefix PRJ- fehlt“. Mit `-WhatIf` erschien statt der Anlage die Meldung „What if: Performing the operation "Team anlegen" on target "PRJ-Alpha".“ (ohne den Erzeuger aufzurufen), und die Warnung für die ungültige Zeile blieb erhalten. Die Namensprüfung selbst ergab: „PRJ-Webportal“ gültig; „Webportal“ wegen fehlendem Präfix ungültig; „PRJ-Gross&Klein“ wegen unerlaubter Zeichen ungültig; ein Name mit 60 Zeichen wegen der Länge ungültig. Die echte Anlage mit `New-Team` wurde nicht ausgeführt.

> **Ergänzung aus der Nachrecherche**
>
> **Reihenfolge und Wartezeiten.** Cloud-Dienste arbeiten häufig *asynchron*: Ein neu angelegtes Team ist nicht sofort vollständig bereit, die zugehörige SharePoint-Website kann Minuten brauchen, bis sie erreichbar ist. Skripte, die nach dem Anlegen sofort Dateien hochladen oder Berechtigungen setzen, scheitern deshalb gelegentlich. Abhilfe ist eine Wiederholung mit Wartezeit (siehe Modul 12, Lerneinheit 12.5) statt eines festen `Start-Sleep`. Auch dieser Hinweis stammt aus der Praxis, nicht aus einer abgerufenen Quelle.

> **Merke**
>
> Teams und SharePoint haben je eigene Werkzeuge (MicrosoftTeams, SharePoint Online Management Shell, PnP PowerShell) und lassen sich teils über Graph steuern. Wichtiger als der einzelne Befehl ist die Disziplin: Namenskonvention prüfen, jede Zeile einzeln behandeln, `-WhatIf` anbieten, auf asynchrone Bereitstellung gefasst sein. Anmeldeverfahren der Werkzeuge ändern sich; die aktuelle Dokumentation ist maßgeblich.

```powershell
# FALSCH: ein Fehler bricht alles ab, keine Namenspruefung, kein Probelauf
Import-Csv teams.csv | ForEach-Object { New-Team -DisplayName $_.Name -Owner $_.Besitzer }

# RICHTIG: jede Zeile geprueft, Fehler einzeln behandelt, mit WhatIf simulierbar
New-TeamsAusCsv -CsvPfad .\teams.csv -WhatIf
```

Die falsche Variante legt auch Teams mit unbrauchbaren Namen an, die man später mühsam wieder entfernen muss. Bei der richtigen sieht man vorher, was geschehen würde, und fehlerhafte Zeilen werden gemeldet und übersprungen.

> **Praxisbeispiel**
>
> Ein Schulungsanbieter legt für jeden Kurs ein Team nach Vorlage an. Die Kursliste kommt als CSV aus dem Buchungssystem; das Skript prüft Namen, legt Teams und Standardkanäle an, setzt den Trainer als Besitzer und schreibt je Zeile ein Ergebnis in ein Protokoll nach dem Muster aus Modul 9. Nach Kursende archiviert ein zweites Skript Teams, die 90 Tage inaktiv sind.

## 10.5 Azure Automation

*Azure Automation* ist der Dienst, in dem man PowerShell-Skripte als *Runbooks* in der Cloud ausführen kann: zeitgesteuert, auf Anforderung (zum Beispiel über Webhook) oder als Reaktion auf Ereignisse. Der Vorteil gegenüber einem Server mit Aufgabenplaner liegt im Betrieb: Es gibt keine Maschine zu pflegen, die Ausführung wird protokolliert, Zugangsdaten und Konfigurationswerte sind zentral verwaltbar, und das Runbook läuft mit einer eigenen Identität. Die Gegenstücke dazu sind die Windows-Aufgabenplanung und die Pipelines aus Modul 11.

Ein *Automation-Konto* ist der Container für alles. Darin liegen die *Runbooks* (die Skripte, mit den Typen PowerShell, Python und anderen), *Zeitpläne* (Schedules), *Variablen*, *Anmeldeinformationen* und *Zertifikate* (zusammen *Assets* genannt) sowie die verfügbaren *Module*. Ein Lauf eines Runbooks heißt *Job*; er hat eine Ausgabe, getrennt nach den PowerShell-Strömen (Modul 9, Lerneinheit 9.2), und einen Status. Für Ziele, die die Cloud nicht erreicht (lokale Server ohne Verbindung), gibt es den *Hybrid Runbook Worker*, einen eigenen Rechner, auf dem das Runbook läuft, aber vom Dienst gesteuert wird.

Ein Thema, an dem viele Anleitungen veraltet sind, ist die Anmeldung. Früher benutzten Runbooks ein sogenanntes *Run-As-Konto* (ein Dienstprinzipal mit Zertifikat). Laut der abgerufenen Dokumentation wurden Run-As-Konten am 30. September 2023 eingestellt; sie sind durch *Managed Identities* ersetzt. Die Dokumentation beschreibt dazu: Ein Automation-Konto hat eine systemzugewiesene Identität (sie ist laut der abgerufenen Seite standardmäßig vorhanden), man kann zusätzlich benutzerzugewiesene Identitäten hinzufügen, und benutzerzugewiesene Identitäten werden nur für Cloud-Aufträge unterstützt, nicht auf Hybrid Workern. Ein Runbook, das noch ein Run-As-Konto erwartet, muss also migriert werden; die Dokumentation enthält dafür eine eigene Anleitung zur Migration.

Das empfohlene Muster am Anfang eines Runbooks, wie es die Dokumentation in ihrem Lernprogramm zeigt, besteht aus drei Schritten. Erstens `Disable-AzContextAutosave -Scope Process`: Es verhindert, dass der Lauf einen Kontext aus einem früheren Lauf übernimmt. Zweitens die Anmeldung mit der Identität des Kontos: `(Connect-AzAccount -Identity).context`. Drittens `Set-AzContext`, um die Subscription festzulegen. Für eine benutzerzugewiesene Identität wird die Anmeldung um `-AccountId` mit der Client-ID der Identität ergänzt. Das Lernprogramm in der Dokumentation verwendet die Runtime-Umgebung mit PowerShell 7.4; die verfügbaren Laufzeitversionen ändern sich, daher gilt auch hier: aktuelle Dokumentation prüfen.

```powershell
# Runbook (PowerShell). Nur auf Syntax geprueft - laeuft in Azure Automation.
param(
    [Parameter(Mandatory)] [string] $ResourceGroupName,
    [string] $SubscriptionId = (Get-AutomationVariable -Name 'SubscriptionId')
)

$ErrorActionPreference = 'Stop'

# 1) Kein Kontext aus frueheren Laeufen uebernehmen
$null = Disable-AzContextAutosave -Scope Process

# 2) Mit der Managed Identity des Automation-Kontos anmelden
$kontext = (Connect-AzAccount -Identity).Context
# Fuer eine benutzerzugewiesene Identitaet: Connect-AzAccount -Identity -AccountId '<Client-ID>'

# 3) Subscription festlegen
$kontext = Set-AzContext -Subscription $SubscriptionId -DefaultProfile $kontext
Write-Output "Angemeldet in Subscription: $($kontext.Subscription.Name)"

# Eigentliche Aufgabe: VMs der Ressourcengruppe auflisten, die nicht laufen
Get-AzVM -ResourceGroupName $ResourceGroupName -Status |
    Where-Object { $_.PowerState -ne 'VM running' } |
    Select-Object Name, PowerState
```

Das Runbook erwartet die Ressourcengruppe als Parameter. Die Subscription holt es standardmäßig aus einer *Automation-Variablen* über `Get-AutomationVariable`, einem Befehl, der nur innerhalb von Automation zur Verfügung steht. Mit `$ErrorActionPreference = 'Stop'` bricht der Lauf bei jedem Fehler ab und endet im Status „Fehlgeschlagen“, was für die Überwachung wichtig ist; ohne diese Zeile würde ein nicht abbrechender Fehler einen scheinbar erfolgreichen Lauf hinterlassen. Danach folgen die drei Anmeldeschritte. Die eigentliche Aufgabe ist hier lesend: Sie listet VMs auf, die nicht laufen. `Write-Output` im Runbook schreibt in die Job-Ausgabe, die man im Portal sieht.

Damit das Runbook überhaupt etwas darf, braucht die Identität *Rollen*. Die Dokumentation zeigt dafür `New-AzRoleAssignment` mit der Objekt-ID der Identität (`-ObjectId`), dem Geltungsbereich (`-Scope` oder `-ResourceGroupName`) und der Rolle (`-RoleDefinitionName`). Dem Grundsatz der geringsten Rechte folgend wählt man die engste passende Rolle und den kleinsten Bereich: „Virtual Machine Contributor“ nur auf der Ressourcengruppe des Runbooks statt „Owner“ auf der Subscription. Eine häufige Fehlermeldung beschreibt die Dokumentation ebenfalls: „this.Client.SubscriptionId cannot be null“ bedeutet, dass die Identität in der Subscription keine Rolle hat; sie kann sich zwar anmelden, aber keine Subscription sehen.

```powershell
# Nur auf Syntax geprueft - benoetigt Azure-Zugang und Rechte zur Rollenzuweisung
$identitaet = (Get-AzAutomationAccount -ResourceGroupName 'rg-auto' -Name 'aa-betrieb').Identity.PrincipalId

New-AzRoleAssignment -ObjectId $identitaet `
    -RoleDefinitionName 'Reader' `
    -Scope '/subscriptions/00000000-0000-0000-0000-000000000000'

New-AzRoleAssignment -ObjectId $identitaet `
    -RoleDefinitionName 'Virtual Machine Contributor' `
    -ResourceGroupName 'rg-demo'
```

Das Beispiel gibt der Identität zuerst die Rolle „Reader“ auf der Subscription (Voraussetzung, um sie überhaupt zu sehen und damit den Fehler mit der leeren Subscription-Kennung zu vermeiden) und dann die Rolle „Virtual Machine Contributor“ nur auf einer Ressourcengruppe. Das Auslesen der Identität über `Get-AzAutomationAccount` und die Eigenschaft `Identity.PrincipalId` stammt aus Fachwissen und ist nicht in den abgerufenen Seiten belegt; man kann die Objekt-ID auch im Portal ablesen. Die Kennung in der Adresse der Subscription ist ein Platzhalter.

> **Ergänzung aus der Nachrecherche**
>
> **Praktische Betriebsfragen.** Ein Runbook ist ein Skript mit Betriebsanforderungen: Es sollte idempotent sein (mehrfaches Ausführen schadet nicht), seine Ausgabe sparsam halten (große Ausgaben verlangsamen Aufträge), Fehler laut melden und Zeitpläne in UTC planen oder die Zeitzone ausdrücklich festlegen. Module werden im Automation-Konto verwaltet; wer ein Modul in einer neuen Version braucht, muss es dort aktualisieren, andernfalls läuft das Runbook mit der alten. Für lang laufende Aufgaben gibt es Grenzen der Laufzeit pro Auftrag, die sich ändern können und in der Dokumentation nachzulesen sind. Diese Betriebshinweise stammen aus Fachwissen.

> **Merke**
>
> Azure Automation führt Runbooks zeitgesteuert oder auf Anforderung aus. Run-As-Konten sind seit dem 30. September 2023 eingestellt; es gilt die Managed Identity. Muster: `Disable-AzContextAutosave -Scope Process`, `Connect-AzAccount -Identity`, `Set-AzContext`. Die Identität braucht Rollen (engste Rolle, kleinster Bereich). Benutzerzugewiesene Identitäten gibt es nur für Cloud-Aufträge, nicht auf Hybrid Workern.

```powershell
# FALSCH: Zugangsdaten im Runbook, kein Abbruch bei Fehlern
$kennwort = ConvertTo-SecureString 'Geheim123' -AsPlainText -Force
Connect-AzAccount -Credential (New-Object pscredential 'admin@firma.example', $kennwort)

# RICHTIG: Managed Identity, Abbruch bei Fehlern, Kontext gesetzt
$ErrorActionPreference = 'Stop'
$null = Disable-AzContextAutosave -Scope Process
$kontext = (Connect-AzAccount -Identity).Context
```

Die falsche Variante hat ein Kennwort im Klartext im Skript, funktioniert wegen der Mehrfaktor-Pflicht für Benutzeridentitäten nicht mehr zuverlässig und ist ein Sicherheitsrisiko. Die richtige braucht kein Geheimnis.

> **Praxisbeispiel**
>
> Ein Runbook schaltet jeden Abend um 19:00 Uhr alle Entwicklungs-VMs ab und morgens um 06:30 wieder ein. Es meldet sich mit der Managed Identity an, hat nur die Rolle „Virtual Machine Contributor“ auf der Ressourcengruppe der Entwicklung und schreibt je VM eine Zeile ins Protokoll. Die Einsparung ist messbar (siehe 10.9), und kein Kennwort ist gespeichert.

## 10.6 Managed Identity und Key Vault

Jedes Skript, das sich irgendwo anmeldet, braucht ein Geheimnis: ein Kennwort, einen Schlüssel, ein Zertifikat, ein Token. Wo dieses Geheimnis liegt, entscheidet über die Sicherheit der ganzen Automatisierung. Die schlechteste Lösung ist, es im Skript zu speichern; es landet im Repository, in Sicherungen und in Protokollen. Die beste Lösung ist, gar kein Geheimnis zu *besitzen*. Genau das ermöglicht eine *Managed Identity* (verwaltete Identität): Azure vergibt einer Ressource (zum Beispiel einem Automation-Konto, einer VM oder einer Funktion) eine Identität im Entra-Verzeichnis und kümmert sich um die Anmeldedaten, die das Skript nie zu Gesicht bekommt. Das Skript sagt nur „melde mich mit der Identität dieser Ressource an“ (`Connect-AzAccount -Identity`), und Azure liefert das Token.

Es gibt zwei Arten. Die *systemzugewiesene* Identität gehört fest zu einer Ressource und wird mit ihr gelöscht. Die *benutzerzugewiesene* Identität ist eine eigene Ressource, die man mehreren Ressourcen zuweisen kann und die unabhängig von ihnen besteht; das ist praktisch, wenn mehrere Runbooks dieselben Rechte brauchen oder wenn man Rechte vorab vergeben will. Wie in 10.5 gesagt, gibt es in Azure Automation Einschränkungen für benutzerzugewiesene Identitäten (nur für Cloud-Aufträge). Wichtig ist ein Gedanke, den beide teilen: Die Identität hat *keine* Rechte, bis man sie ihr gibt. Rechte vergibt man über *rollenbasierte Zugriffssteuerung* (RBAC) mit Rollenzuweisungen, wie in 10.5 mit `New-AzRoleAssignment` gezeigt.

Nicht jedes Geheimnis lässt sich durch eine Identität ersetzen. Manchmal muss ein Skript ein Kennwort für ein Drittsystem (eine Datenbank, eine externe Schnittstelle) verwenden. Für diese Fälle gibt es *Azure Key Vault*, einen Tresor für Geheimnisse, Schlüssel und Zertifikate mit Zugriffsprotokoll. Das Skript meldet sich mit seiner Managed Identity am Tresor an, liest das Geheimnis zur Laufzeit und verwendet es, ohne dass es je in Quellcode oder Konfiguration stand. Die abgerufene Dokumentation zeigt dafür den Befehl `Get-AzKeyVaultSecret -VaultName -Name` und den Schalter `-AsPlainText`, der den Wert als Text statt als `SecureString` liefert; beim Auflisten mehrerer Geheimnisse (ohne `-Name`) wird der Schalter laut Dokumentation ignoriert. Dem Skript muss die Rolle zum Lesen von Geheimnissen auf dem Tresor zugewiesen sein (zum Beispiel „Key Vault Secrets User“ bei RBAC-gesteuerten Tresoren; der Rollenname stammt aus Fachwissen, prüfen Sie ihn in der Dokumentation).

```powershell
# Nur auf Syntax geprueft - benoetigt Azure-Zugang und einen Key Vault
$null = Connect-AzAccount -Identity

# Variante 1: SecureString (Standard) - bevorzugt, wenn der Empfaenger SecureString akzeptiert
$geheimnis = Get-AzKeyVaultSecret -VaultName 'kv-betrieb' -Name 'db-kennwort'
$kennwort  = $geheimnis.SecretValue                 # [securestring]
$cred      = [pscredential]::new('svc-db', $kennwort)

# Variante 2: Klartext - nur wenn es nicht anders geht, und nur kurz im Speicher halten
$text = Get-AzKeyVaultSecret -VaultName 'kv-betrieb' -Name 'api-schluessel' -AsPlainText
try     { Invoke-RestMethod -Uri $url -Headers @{ 'x-api-key' = $text } }
finally { $text = $null }
```

Variante 1 behält das Geheimnis als `SecureString`; daraus baut das Beispiel ein `PSCredential`-Objekt, das viele Befehle direkt akzeptieren. Variante 2 benutzt `-AsPlainText`, wenn eine Schnittstelle nur Text annimmt, und setzt die Variable im `finally` wieder auf `$null`; das ist keine echte Löschung des Speichers, begrenzt aber zumindest die Lebensdauer in der Sitzung. Beide Varianten haben gemeinsam, dass das Geheimnis nie im Skript steht, nie in Protokollen auftauchen darf und nie mit `Write-Output` oder `Write-Host` ausgegeben wird.

Was passiert bei der Weiterverarbeitung eines Geheimnisses im Skript? Das Muster aus Modul 9 (Maskierung im Protokoll) ist ein Sicherheitsnetz, aber nicht die erste Verteidigungslinie. Die erste Verteidigungslinie ist, Geheimnisse als `SecureString` zu führen, der bei der Ausgabe nicht den Inhalt zeigt. Im Test mit PowerShell 7.6.6 lieferte `$s | Out-String` für einen `SecureString` lediglich den Typnamen `System.Security.SecureString`, und die Ausgabe eines `PSCredential`-Objekts enthielt das Kennwort nicht (die Prüfung, ob der Text des Kennworts darin vorkam, ergab `False`). Eine kleine Hilfsfunktion stellt sicher, dass man unabhängig von der Quelle immer einen `SecureString` erhält.

```powershell
function Get-GeheimnisAlsSecureString {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Name,
        [scriptblock] $Quelle = { param($n) throw 'keine Quelle konfiguriert' }
    )
    $roh = & $Quelle $Name
    if ($roh -is [securestring]) { return $roh }
    ConvertTo-SecureString -String ([string]$roh) -AsPlainText -Force
}

# Test mit simulierter Quelle:
$s = Get-GeheimnisAlsSecureString -Name 'db' -Quelle { param($n) "kennwort-fuer-$n" }
$s.GetType().Name                      # SecureString
([pscredential]::new('svc', $s) | Out-String).Contains('kennwort')   # False
```

Die Funktion holt das Geheimnis über eine austauschbare Quelle (im Betrieb `Get-AzKeyVaultSecret`, im Test ein Skriptblock) und wandelt es bei Bedarf in einen `SecureString`. Das Umwandeln mit `-AsPlainText -Force` ist hier unvermeidlich, weil die Quelle Text liefern kann; deshalb ist es wichtig, die Quelle möglichst gleich einen `SecureString` liefern zu lassen. Im Test lieferte der Aufruf den Typ `SecureString`, und die Ausgabe des `PSCredential`-Objekts enthielt den Kennworttext nicht. Eine Warnung gehört dazu: `SecureString` ist *kein* Verschlüsselungsschutz gegen jemanden, der auf dem Rechner Code ausführen darf; auf Linux und macOS ist der Schutz zudem schwächer als unter Windows (dieser Hinweis stammt aus Fachwissen). Er verhindert vor allem versehentliche Ausgabe.

> **Ergänzung aus der Nachrecherche**
>
> **Weitere Werkzeuge und Rotation.** Neben `Az.KeyVault` gibt es die Module *SecretManagement* und *SecretStore* für ein einheitliches Vorgehen mit verschiedenen Tresoren (Fachwissen, nicht abgerufen). Ein weiterer Punkt ist die *Rotation*: Geheimnisse sollten regelmäßig wechseln, und das Skript darf nicht an einer bestimmten Version hängen. In Key Vault liest `Get-AzKeyVaultSecret` ohne Versionsangabe stets die aktuelle Version, ein Wechsel im Tresor wirkt also ohne Änderung am Skript. Und: Das Zugriffsprotokoll des Tresors ist eine wertvolle Quelle, wenn man ungewöhnliche Zugriffe finden will.

> **Merke**
>
> Beste Lösung: kein Geheimnis besitzen, sondern eine Managed Identity benutzen (system- oder benutzerzugewiesen), die nur die nötigen Rollen hat. Geheimnisse, die man nicht vermeiden kann, liegen in Key Vault (`Get-AzKeyVaultSecret`); im Skript als `SecureString` führen, `-AsPlainText` nur im Notfall und kurz. Nie im Skript speichern, nie protokollieren, regelmäßig rotieren.

```powershell
# FALSCH: Geheimnis im Skript und im Protokoll
$kennwort = 'Geheim123'
Write-Log "Verbinde mit Kennwort $kennwort"

# RICHTIG: Geheimnis aus dem Tresor, nur als SecureString, nichts im Protokoll
$kennwort = (Get-AzKeyVaultSecret -VaultName 'kv-betrieb' -Name 'db-kennwort').SecretValue
Write-Log 'Verbinde mit Datenbank' -Aktion 'Verbinden'
```

Das Kennwort im Quelltext wandert in die Versionsverwaltung und bleibt dort für immer, auch wenn man es später löscht. Die richtige Variante liest es zur Laufzeit; im Protokoll steht nur, was geschah.

> **Praxisbeispiel**
>
> Nach einem Sicherheitsvorfall prüft das Team alle Skripte mit dem Inventar aus Modul 12 auf Kennwortzuweisungen. Zwölf Skripte enthalten Geheimnisse; sie werden auf Managed Identity oder Key Vault umgestellt, die alten Kennwörter rotiert. Seitdem gehört die Regel „keine Geheimnisse im Code“ zur automatischen Prüfung in der Pipeline.

## 10.7 Cloud Governance

In der Cloud kann jeder mit den richtigen Rechten in Minuten Ressourcen anlegen, und genau darin liegt die Gefahr: Ohne Regeln wachsen Umgebungen ungeordnet, Kosten laufen aus dem Ruder, niemand weiß, wem eine Ressource gehört, und Sicherheitsvorgaben werden unterlaufen. *Cloud Governance* bezeichnet die Regeln und Kontrollen, mit denen eine Organisation das verhindert, ohne die Teams unnötig auszubremsen. In Azure stehen dafür vier Bausteine im Mittelpunkt, die man alle per Skript prüfen und teilweise durchsetzen kann: *Namenskonventionen*, *Tags*, *Richtlinien* (Azure Policy) und *rollenbasierte Zugriffssteuerung*.

Ein *Tag* ist ein Schlüssel-Wert-Paar, das man an eine Ressource oder Ressourcengruppe hängt, zum Beispiel `Owner=team-web` oder `CostCenter=4711`. Tags sind das Rückgrat von Kostenzuordnung, Verantwortlichkeit und Aufräumaktionen: Mit `CostCenter` kann man Kosten verteilen, mit `Owner` weiß man, wen man fragen muss, mit `Environment` (Prod, Test, Dev) unterscheidet man Schutzbedarf. Wichtig ist, dass die Organisation eine kleine Menge von *Pflicht-Tags* festlegt und deren erlaubte Werte definiert. Sonst entstehen „Owner“, „owner“ und „Besitzer“ nebeneinander, und keine Auswertung stimmt.

Ob Ressourcen den Regeln entsprechen, prüft ein Skript am einfachsten auf einer Liste von Ressourcenobjekten, etwa dem Ergebnis von `Get-AzResource`. Die folgende Funktion prüft Pflicht-Tags und erlaubte Werte; sie ist unabhängig von Azure und deshalb mit Beispielobjekten testbar.

```powershell
function Test-TagKonformitaet {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)] [psobject] $Ressource,
        [string[]] $Pflichttags = @('Owner', 'CostCenter', 'Environment'),
        [string[]] $ErlaubteUmgebungen = @('Prod', 'Test', 'Dev')
    )
    process {
        $tags = if ($Ressource.Tags) { $Ressource.Tags } else { @{} }
        $probleme = [System.Collections.Generic.List[string]]::new()
        foreach ($t in $Pflichttags) {
            if (-not $tags.ContainsKey($t) -or [string]::IsNullOrWhiteSpace($tags[$t])) {
                $probleme.Add("Tag '$t' fehlt")
            }
        }
        if ($tags.ContainsKey('Environment') -and $tags['Environment'] -notin $ErlaubteUmgebungen) {
            $probleme.Add("Environment '$($tags['Environment'])' nicht erlaubt")
        }
        [pscustomobject]@{
            Name     = $Ressource.Name
            Typ      = $Ressource.ResourceType
            Konform  = ($probleme.Count -eq 0)
            Probleme = ($probleme -join '; ')
        }
    }
}

# Aufruf mit echten Daten (nur auf Syntax geprueft):
Get-AzResource | Test-TagKonformitaet | Where-Object { -not $_.Konform } |
    Export-Csv tag-verstoesse.csv -NoTypeInformation -Encoding utf8
```

Die Funktion nimmt Ressourcen aus der Pipeline entgegen und prüft jede einzeln im `process`-Block. Fehlt eine Ressource komplett an Tags (`$null`), wird sie wie eine leere Hashtable behandelt. Für jedes Pflicht-Tag prüft sie, ob es vorhanden und nicht leer ist, und sie vergleicht den Wert des Tags `Environment` mit der Liste der erlaubten Werte. Das Ergebnis je Ressource ist ein Objekt mit dem Befund. Hinweis: In Az sind die Tags einer Ressource je nach Cmdlet und Version eine Hashtable oder ein ähnliches Wörterbuch; ob `ContainsKey` in jedem Fall funktioniert, wurde nicht getestet, weil die Befehle nicht verfügbar waren. Man prüft das einmal mit einem echten `Get-AzResource`-Ergebnis.

Im Test mit drei Beispielressourcen ergab sich: Die VM `vm-web01` mit allen drei Tags und dem Wert „Prod“ war konform. Das Speicherkonto `st-daten` mit `Owner` und `Environment=Produktion` verletzte zwei Regeln: Es fehlte `CostCenter`, und „Produktion“ ist kein erlaubter Wert (erlaubt sind Prod, Test, Dev). Der Tresor `kv-alt` ganz ohne Tags meldete alle drei Pflicht-Tags als fehlend. Zusammengefasst: 1 von 3 Ressourcen war konform.

*Azure Policy* geht einen Schritt weiter: Statt nur zu *prüfen*, kann sie Regeln *durchsetzen*, etwa „Ressourcen ohne Tag CostCenter dürfen nicht angelegt werden“ (Wirkung *Deny*) oder „fehlende Tags automatisch ergänzen“ (Wirkung *Modify*). Man weist eine Richtlinie einem Bereich zu (Verwaltungsgruppe, Subscription oder Ressourcengruppe) und kann den Konformitätszustand abfragen. Die technische Beschreibung einer Richtlinie ist ein JSON-Dokument mit Bedingung und Wirkung. Dass es diesen Mechanismus gibt und wie er grob funktioniert, ist Fachwissen; die Dokumentation zu Azure Policy wurde für dieses Modul nicht abgerufen, und konkrete Befehle sind nur als Muster zu verstehen. Mehr zum Thema „Richtlinien als Code“ folgt in Modul 11, Lerneinheit 11.8.

```powershell
# Nur auf Syntax geprueft - Muster, Parameter vor Einsatz in der Dokumentation pruefen
$definition = Get-AzPolicyDefinition | Where-Object { $_.DisplayName -like '*Require a tag*' } |
    Select-Object -First 1

New-AzPolicyAssignment -Name 'tag-costcenter-pflicht' `
    -DisplayName 'CostCenter-Tag ist Pflicht' `
    -PolicyDefinition $definition `
    -PolicyParameterObject @{ tagName = 'CostCenter' } `
    -Scope '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-demo'

# Rollenzuweisungen pruefen: wer ist Owner, und ist es ein Mensch statt einer Gruppe?
Get-AzRoleAssignment | Where-Object { $_.RoleDefinitionName -eq 'Owner' } |
    Select-Object DisplayName, ObjectType, Scope
```

Das Beispiel zeigt zwei Dinge. Erstens die Zuweisung einer eingebauten Richtlinie (Name der Definition und Parameter sind hier Platzhalter, die man in der Dokumentation und im Portal nachschlägt) auf eine Ressourcengruppe. Zweitens eine Prüfung der Rollenzuweisungen: `Get-AzRoleAssignment` liefert, wer welche Rolle in welchem Bereich hat. Eine einfache und wirkungsvolle Regelprüfung lautet: „Owner-Rechte nur für Gruppen, nicht für einzelne Benutzer, und nur dort, wo nötig.“ Genau solche Abfragen gehören in regelmäßige Berichte.

> **Ergänzung aus der Nachrecherche**
>
> **Prüfen, durchsetzen, nachweisen.** Eine gute Governance hat drei Stufen. Zuerst wird *geprüft* (Berichte wie oben), dann *durchgesetzt* (Policy mit Deny oder Modify), schließlich *nachgewiesen* (Berichte als Beleg für Revisoren). Am Anfang ist Durchsetzung riskant, weil sie bestehende Arbeitsabläufe bricht; bewährt hat sich, mit Prüfen zu beginnen, die Teams Verstöße beheben zu lassen und erst danach zu sperren. Auch Ressourcensperren (Locks gegen versehentliches Löschen) und Budgets mit Benachrichtigung zählen zu den Bausteinen; sie wurden hier nicht abgerufen und sind als Hinweis zu verstehen.

> **Merke**
>
> Governance = Namenskonvention, Pflicht-Tags mit erlaubten Werten, Richtlinien (Azure Policy: prüfen oder durchsetzen) und RBAC mit geringsten Rechten. Skripte prüfen den Ist-Zustand (Tag-Prüfung, Rollenberichte); Richtlinien setzen durch. Erst prüfen und beheben lassen, dann sperren.

```powershell
# FALSCH: Tags frei erfunden, Auswertung unmoeglich
# Owner=Anna   owner=anna   Besitzer=A.M.   Env=produktion   Environment=PROD

# RICHTIG: feste Schluessel, feste Werteliste, automatisch geprueft
Get-AzResource | Test-TagKonformitaet -ErlaubteUmgebungen 'Prod','Test','Dev' |
    Where-Object { -not $_.Konform }
```

Ohne feste Schlüssel und Werte lässt sich nichts zuverlässig gruppieren; mit der automatischen Prüfung werden Abweichungen sofort sichtbar.

> **Praxisbeispiel**
>
> Ein Unternehmen führt Pflicht-Tags ein. Zuerst läuft vier Wochen lang nur der Bericht aus diesem Abschnitt und geht an die Teamleiter; 40 Prozent der Ressourcen sind anfangs nicht konform. Nach Behebung wird die Richtlinie „CostCenter-Tag ist Pflicht“ mit der Wirkung Deny aktiviert, damit künftig keine untaggten Ressourcen mehr entstehen.

## 10.8 API-Integration

Nicht für jeden Dienst gibt es ein PowerShell-Modul, und manchmal ist das Modul langsamer oder unvollständiger als die dahinterliegende Schnittstelle. Dann spricht man die *REST-Schnittstelle* (englisch *REST API*) direkt an. REST bedeutet in der Praxis: Man schickt eine HTTP-Anfrage (GET zum Lesen, POST zum Anlegen, PUT oder PATCH zum Ändern, DELETE zum Löschen) an eine Adresse und bekommt eine Antwort mit Statuscode und meist JSON im Inhalt. Das PowerShell-Werkzeug dafür ist `Invoke-RestMethod`; es wandelt JSON-Antworten automatisch in Objekte um. Für die Antwort mit allen Details (Kopfzeilen, Rohinhalt) gibt es `Invoke-WebRequest`.

Vier Aufgaben tauchen bei jeder Integration auf: Authentifizierung, Seitenabruf, Fehler- und Drosselungsbehandlung und Zeitlimits. Zur *Authentifizierung*: Die meisten modernen Schnittstellen erwarten ein *Bearer-Token* im Kopf `Authorization`. Das Token holt man bei einem Anmeldedienst. Für unbeaufsichtigte Abläufe heißt das übliche Verfahren *Client Credentials*: Die Anwendung weist sich mit Kennung und Geheimnis aus und erhält ein Token mit begrenzter Lebensdauer. Im Fall von Microsoft Entra ID wird dafür ein POST an die Token-Adresse des Mandanten geschickt, mit dem Inhaltstyp `application/x-www-form-urlencoded` und den Feldern `grant_type`, `client_id`, `client_secret` und `scope`. Dieses Protokoll ist Fachwissen und nicht aus den abgerufenen Seiten belegt; die genauen Adressen und Felder sind in der Dokumentation des Anmeldedienstes nachzuschlagen, und die Anfrage wurde nicht an Microsoft gesendet.

```powershell
function New-TokenAnfrage {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $TenantId,
        [Parameter(Mandatory)] [string] $ClientId,
        [Parameter(Mandatory)] [securestring] $ClientSecret,
        [string] $Scope = 'https://graph.microsoft.com/.default'
    )
    [pscustomobject]@{
        Method      = 'POST'
        Uri         = "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token"
        ContentType = 'application/x-www-form-urlencoded'
        Body        = @{
            grant_type    = 'client_credentials'
            client_id     = $ClientId
            client_secret = [System.Net.NetworkCredential]::new('', $ClientSecret).Password
            scope         = $Scope
        }
    }
}

# Nur auf Syntax geprueft (die Anfrage wird hier nicht gesendet):
$a = New-TokenAnfrage -TenantId 'contoso.onmicrosoft.com' -ClientId $id -ClientSecret $geheimnis
$token = (Invoke-RestMethod -Method $a.Method -Uri $a.Uri -ContentType $a.ContentType -Body $a.Body).access_token
```

Die Funktion baut nur die Anfrage zusammen, sie sendet sie nicht; das trennt Aufbau und Versand und macht den Aufbau testbar. Das Geheimnis kommt als `SecureString` herein (Quelle: Key Vault, siehe 10.6) und wird erst im allerletzten Moment, im Anfragekörper, in Text umgewandelt. Im Test lautete die Adresse `https://login.microsoftonline.com/contoso.onmicrosoft.com/oauth2/v2.0/token`, und der Körper enthielt die vier Felder `client_id`, `client_secret`, `grant_type` und `scope`. Wie PowerShell eine Hashtable als Formularkörper kodiert, wurde gegen einen lokalen Testserver geprüft: Aus `@{ grant_type = 'client_credentials'; scope = 'a b' }` entstand der Text `grant_type=client_credentials&scope=a+b` mit dem Inhaltstyp `application/x-www-form-urlencoded`; Sonderzeichen werden also automatisch kodiert.

Das Token übergibt man danach im Kopf. PowerShell 7 bietet dafür die Parameter `-Authentication Bearer -Token $tokenAlsSecureString`. Der lokale Test ergab dabei eine nützliche Sicherheitsfunktion: Über eine unverschlüsselte Verbindung (`http://`) verweigert das Cmdlet das Senden und meldet, dass es Geheimnisse im Klartext nicht über unverschlüsselte Netzwerke senden will; nur mit dem ausdrücklichen Schalter `-AllowUnencryptedAuthentication` geht es, und der gehört ausschließlich in lokale Tests. Mit diesem Schalter erreichte der Testserver den Kopf `Bearer abc123`; gleichwertig ist ein selbst gesetzter Kopf `-Headers @{ Authorization = "Bearer ..." }` (im Test: `Bearer xyz`). Für echte Schnittstellen gilt: immer `https://`.

Zum *Seitenabruf*: Dasselbe Muster wie bei Graph in 10.2 gilt für fast jede Schnittstelle, nur das Feld für den nächsten Link heißt je nach Dienst anders (`@odata.nextLink`, `next`, `Link`-Kopf oder ein Fortsetzungs-Token). Im Test mit einem lokalen Server, der drei Seiten à zwei Einträgen lieferte, sammelte eine Schleife nach demselben Muster sechs Einträge ein; der letzte hieß „Eintrag 6“.

Zu *Fehlern und Drosselung*: Antwortet der Server mit einem Fehlercode, wirft `Invoke-RestMethod` eine Ausnahme; den Fehlertext des Servers findet man in `$_.ErrorDetails.Message`, den Statuscode in `$_.Exception.Response.StatusCode` (im Test: 404 und der Text `{ "error": "nicht gefunden" }`). Wer den Fehlercode ohne Ausnahme auswerten will, nutzt `-SkipHttpErrorCheck` zusammen mit `-StatusCodeVariable`; im Test ergab das den Wert 404 und den Fehlertext im Ergebnisobjekt. Die ausführliche Wiederholungslogik mit exponentiellem Backoff steht in Modul 12, Lerneinheit 12.5; hier der Teil, den PowerShell 7 selbst beherrscht: `Invoke-RestMethod` hat die Parameter `-MaximumRetryCount` und `-RetryIntervalSec`. Im Test lieferte ein lokaler Server zweimal die Antwort 429 mit `Retry-After: 2` und danach Erfolg. Mit `-MaximumRetryCount 3 -RetryIntervalSec 1` kam beim dritten Versuch die Erfolgsantwort, und der Aufruf dauerte vier Sekunden: Das Cmdlet hatte also zweimal die vom Server genannten zwei Sekunden gewartet und nicht die eine Sekunde aus `-RetryIntervalSec`. Es befolgt `Retry-After`. Ein Fehler 404 wurde im selben Test nicht wiederholt und sofort gemeldet.

```powershell
# Lokal getestet gegen einen Testserver (127.0.0.1); echte Dienste: immer https
$antwort = Invoke-RestMethod -Uri $url `
    -Headers @{ Accept = 'application/json' } `
    -TimeoutSec 30 `
    -MaximumRetryCount 3 -RetryIntervalSec 2 `
    -ErrorAction Stop

# Statuscode ohne Ausnahme auswerten
$r = Invoke-RestMethod -Uri $url -SkipHttpErrorCheck -StatusCodeVariable status
if ($status -ge 400) { Write-Warning "Server meldet $status" }

# Fehlertext des Servers bei einer Ausnahme
try { Invoke-RestMethod -Uri $url -ErrorAction Stop }
catch {
    $code = [int]$_.Exception.Response.StatusCode
    $text = $_.ErrorDetails.Message
    Write-Error "Aufruf fehlgeschlagen ($code): $text"
}
```

Der erste Aufruf legt Zeitlimit (`-TimeoutSec`, Alias von `-OperationTimeoutSeconds`) und Wiederholungen fest. Der zweite wertet den Statuscode selbst aus; die Variable wird mit dem Namen ohne Dollarzeichen übergeben. Der dritte Block liest im Fehlerfall Statuscode und Fehlertext aus. Das Zeitlimit ist unverzichtbar: Ohne eines kann ein hängender Server ein Skript ewig blockieren (siehe Modul 9, Lerneinheit 9.7).

Schließlich die *Idempotenz*: Eine Operation heißt idempotent, wenn man sie mehrfach ausführen kann, ohne dass sich das Ergebnis über den ersten Aufruf hinaus ändert. GET, PUT und DELETE sind es in der Regel, POST („lege an“) meist nicht. Wenn ein POST wegen eines Zeitlimits unklar ausgeht (angekommen oder nicht?), kann eine blinde Wiederholung einen Eintrag doppelt anlegen. Gute Schnittstellen bieten einen *Idempotenz-Schlüssel* an (ein Kopf, den man mit einer eindeutigen Kennung füllt); bei fehlender Unterstützung prüft man vor dem Wiederholen mit einer Abfrage, ob der Eintrag schon existiert.

> **Ergänzung aus der Nachrecherche**
>
> **Eigene Aufrufe oder SDK-Cmdlets?** SDK-Cmdlets (Graph, Az) nehmen einem viel ab: Anmeldung, Seitenabruf, Wiederholung, Datentypen. Direkte Aufrufe lohnen sich, wenn es kein Cmdlet gibt, wenn Cmdlets zu langsam sind oder neue Schnittstellenfunktionen noch nicht abgedeckt sind. Für Azure gibt es `Invoke-AzRestMethod`, das den Anmeldekontext von Az verwendet, und für Graph `Invoke-MgGraphRequest`, das den Graph-Kontext verwendet; damit entfällt die eigene Token-Beschaffung (beide Befehle nur als Hinweis aus Fachwissen, nicht getestet). Das manuelle Token-Verfahren oben ist für Dienste gedacht, die keinen solchen Kontext anbieten.

> **Merke**
>
> REST-Aufrufe mit `Invoke-RestMethod`: Authentifizierung (Bearer-Token, Client Credentials, Geheimnis aus dem Tresor), Seitenabruf (nächster Link), Fehler (`ErrorDetails.Message`, `-SkipHttpErrorCheck`), Drosselung (`-MaximumRetryCount` befolgt `Retry-After`), Zeitlimit (`-TimeoutSec`) und Idempotenz. Nie über `http://` mit Geheimnissen; `-AllowUnencryptedAuthentication` nur für lokale Tests.

```powershell
# FALSCH: kein Zeitlimit, Token als Klartext in der URL, Fehler ignoriert
Invoke-RestMethod "http://api.example/items?token=$token" -ErrorAction SilentlyContinue

# RICHTIG: https, Token im Kopf, Zeitlimit, Fehler sichtbar
Invoke-RestMethod 'https://api.example/items' -Authentication Bearer -Token $tokenSecure `
    -TimeoutSec 30 -ErrorAction Stop
```

In der falschen Variante steht das Token in der Adresse (und damit in Protokollen von Servern und Proxys), es gibt kein Zeitlimit, und Fehler verschwinden. Die richtige Variante nutzt eine verschlüsselte Verbindung, den vorgesehenen Kopf und macht Fehler sichtbar.

> **Praxisbeispiel**
>
> Ein Inventarisierungsdienst im Haus hat nur eine REST-Schnittstelle. Das Skript holt ein Token mit Client Credentials (Geheimnis aus Key Vault), liest alle Geräte seitenweise, wiederholt bei 429 nach `Retry-After`, bricht bei 4xx mit einer klaren Meldung ab und schreibt jede Seite mit Run-ID ins Protokoll. Ein Zeitlimit von 30 Sekunden pro Aufruf verhindert Hänger.

## 10.9 Kosten- und Nutzungsreports

Cloud-Kosten sind verbrauchsabhängig und deshalb schwer zu überblicken. Wer sie nicht regelmäßig auswertet, merkt erst auf der Rechnung, dass eine vergessene Testumgebung das Budget sprengt. *Kostenberichte* beantworten wiederkehrende Fragen: Was kostet uns welcher Dienst? Welches Team verursacht wie viel? Was hat sich gegenüber dem Vormonat geändert? Und: Welche Kosten sind keinem Team zugeordnet? Die letzte Frage hängt direkt mit den Tags aus 10.7 zusammen: Ohne `CostCenter`-Tag gibt es keine saubere Verteilung.

Woher kommen die Daten? Azure Cost Management bietet Auswertungen im Portal, Exporte als CSV-Dateien in ein Speicherkonto und eine Abfrageschnittstelle. Die Existenz dieser Möglichkeiten ist Fachwissen; weder ihre genauen Befehle noch Spaltennamen wurden für dieses Modul abgerufen, und die Spaltennamen der Exporte unterscheiden sich je nach Vertragsart und Version. Die Beispiele arbeiten deshalb mit einer *angenommenen* CSV-Datei mit den Spalten `Datum`, `Ressourcengruppe`, `Dienst`, `Kosten` und `CostCenter`. Die Kernlogik (Gruppieren, Summieren, Anteil berechnen, nicht zugeordnete Kosten finden) ist unabhängig von der Quelle und wurde mit Beispieldaten getestet.

```powershell
$csv = @'
Datum,Ressourcengruppe,Dienst,Kosten,CostCenter
2026-10-01,rg-web,Virtual Machines,120.50,4711
2026-10-01,rg-web,Storage,20.10,4711
2026-10-02,rg-data,Storage,80.00,4712
2026-10-02,rg-data,SQL,210.25,4712
2026-10-03,rg-web,Virtual Machines,118.00,4711
2026-10-03,rg-test,Virtual Machines,35.00,
'@
$daten = $csv | ConvertFrom-Csv
$inv = [Globalization.CultureInfo]::InvariantCulture
$daten | ForEach-Object { $_.Kosten = [double]::Parse($_.Kosten, $inv) }

$gesamt = ($daten | Measure-Object Kosten -Sum).Sum
$proDienst = $daten | Group-Object Dienst | ForEach-Object {
    $summe = ($_.Group | Measure-Object Kosten -Sum).Sum
    [pscustomobject]@{
        Dienst = $_.Name
        Summe  = [math]::Round($summe, 2)
        Anteil = [math]::Round(100 * $summe / $gesamt, 1)
    }
} | Sort-Object Summe -Descending

$proDienst | Format-Table -AutoSize
"Gesamt: {0:N2}" -f $gesamt
$ohne = @($daten | Where-Object { [string]::IsNullOrWhiteSpace($_.CostCenter) })
"Ohne CostCenter: $($ohne.Count) Zeilen, Summe $(($ohne | Measure-Object Kosten -Sum).Sum)"
```

Das Beispiel liest die Daten und wandelt die Spalte `Kosten` von Text in Zahlen, ausdrücklich mit der invarianten Kultur (die Quelle verwendet den Punkt als Dezimaltrenner; ohne diese Angabe würde ein deutschsprachiges System „120.50“ falsch lesen). Dann berechnet es die Gesamtsumme, gruppiert nach Dienst, summiert je Gruppe, berechnet den Anteil in Prozent und sortiert absteigend. Zuletzt sucht es die Zeilen ohne Kostenstelle. Das Ergebnis im Test: Virtual Machines 273,50 (46,8 Prozent), SQL 210,25 (36,0 Prozent), Storage 100,10 (17,1 Prozent), Gesamtsumme 583,85, und eine Zeile ohne `CostCenter` mit 35. Als JSON ausgegeben (`ConvertTo-Json -Compress`) kam eine Liste mit den Feldern `Dienst`, `Summe` und `Anteil` heraus, die sich an ein Dashboard liefern lässt.

Ein Bericht, der für Menschen gedacht ist, braucht die richtige Darstellung der Zahlen. Wer eine CSV-Datei für ein deutschsprachiges Excel erzeugt, braucht das Semikolon als Trennzeichen und das Komma als Dezimaltrenner; `Export-Csv -Delimiter ';'` liefert das Trennzeichen, die Zahlen müssen aber selbst mit der passenden Kultur formatiert werden. Im Test erzeugten `.ToString('N2', de-AT)` die Zeilen `Virtual Machines;273,50;46,8`, `SQL;210,25;36,0` und `Storage;100,10;17,1`. Für Maschinen (JSON, Prometheus-Dateien) bleibt dagegen die invariante Kultur mit Punkt richtig; das Prinzip „intern invariant, nur für die Anzeige lokal“ aus Modul 9 gilt hier unverändert.

```powershell
$de = [Globalization.CultureInfo]::GetCultureInfo('de-AT')
$proDienst | ForEach-Object {
    [pscustomobject]@{
        Dienst = $_.Dienst
        Summe  = $_.Summe.ToString('N2', $de)
        Anteil = $_.Anteil.ToString('N1', $de)
    }
} | Export-Csv kosten-bericht.csv -Delimiter ';' -NoTypeInformation -Encoding utf8BOM
```

Die Schleife formatiert die Zahlen als Text für die österreichische Kultur und schreibt eine CSV-Datei mit Semikolon. Die Kodierung `utf8BOM` ist gewählt, weil ältere Excel-Versionen Umlaute sonst falsch anzeigen (siehe Modul 12, Lerneinheit 12.2 zum Thema Byte-Order-Mark). Der Aufruf von `GetCultureInfo('de-AT')` lief im Test; auf Systemen ohne Kulturdaten (zum Beispiel minimale Container) kann er fehlschlagen.

Aus Kostenberichten entstehen *Warnungen*: Ein Skript vergleicht die Kosten der letzten Tage mit dem Durchschnitt und meldet Ausreißer, etwa wenn ein Dienst um mehr als 50 Prozent über dem Mittel liegt. Ebenso nützlich sind *Nutzungsberichte* (nicht nur Kosten): ungenutzte Ressourcen wie nicht angehängte Datenträger, ausgeschaltete, aber weiter abgerechnete VMs oder Testumgebungen ohne Tag `Owner`. Genau hier bringt das Governance-Wissen aus 10.7 den größten Hebel, denn eine Ressource mit Besitzer-Tag lässt sich ansprechen, eine ohne nicht.

> **Ergänzung aus der Nachrecherche**
>
> **Preise und Quellen.** Dieses Modul nennt bewusst keine Azure-Preise: Sie ändern sich, hängen von Region und Vertrag ab und wurden hier nicht abgerufen. Für Prognosen und Budgets sind die aktuelle Preisliste und die tatsächlichen Verträge der Organisation maßgeblich. Wichtig ist außerdem, dass Exportdaten verzögert eintreffen können und Preise nachträglich korrigiert werden; ein Bericht über den laufenden Monat ist daher eine Schätzung und kein Abschluss.

> **Merke**
>
> Kostenberichte beantworten: Was kostet welcher Dienst, welches Team, was ist neu, was ist niemandem zugeordnet? Kernlogik: einlesen (Zahlen mit invarianter Kultur), gruppieren, summieren, Anteile, Zeilen ohne Kostenstelle. Für Maschinen invariant (Punkt), für Excel lokal (Semikolon, Komma). Tags sind die Grundlage jeder Verteilung.

```powershell
# FALSCH: Kosten als Text sortieren und summieren
$daten | Sort-Object Kosten | Select-Object -Last 3        # Textsortierung: "9" > "100"

# RICHTIG: erst in Zahlen umwandeln, dann rechnen
$daten | ForEach-Object { $_.Kosten = [double]::Parse($_.Kosten, [Globalization.CultureInfo]::InvariantCulture) }
$daten | Sort-Object Kosten -Descending | Select-Object -First 3
```

Aus einer CSV-Datei kommt jeder Wert als Text; ohne Umwandlung sortiert PowerShell alphabetisch, sodass „9.00“ vor „100.00“ steht. Erst nach der Umwandlung stimmen Sortierung und Summen.

> **Praxisbeispiel**
>
> Jeden Montag erzeugt ein Runbook den Bericht der Vorwoche: die fünf teuersten Dienste, Kosten je Kostenstelle und die Summe ohne Kostenstelle. Die Ausreißerwarnung meldet, dass eine Testumgebung seit Freitag 400 Euro pro Tag kostet; sie wird am Montagmorgen abgeschaltet, bevor der Monat teuer wird.

## 10.10 Übungsaufgaben zu Modul 10

Die Aufgaben steigen in der Schwierigkeit an. Die Aufgaben 3, 4, 6 und 7 sind praktisch (Code schreiben). Die Aufgaben mit Cloud-Befehlen lassen sich nur mit eigenem Zugang ausführen; die übrigen laufen ohne Mandanten. Die Lösungen mit Erklärung folgen im nächsten Abschnitt.

**Aufgabe 1.** Erklären Sie den Unterschied zwischen delegierter und reiner Anwendungsanmeldung (app-only) bei Microsoft Graph. Welche Art würden Sie für ein nächtliches Berichtsskript wählen und warum? Wie finden Sie heraus, welche Berechtigung ein Cmdlet braucht?

**Aufgabe 2.** Warum wurden Run-As-Konten in Azure Automation durch Managed Identities ersetzt, und was ist der Unterschied zwischen systemzugewiesener und benutzerzugewiesener Identität? Welche Einschränkung gilt für benutzerzugewiesene Identitäten?

**Aufgabe 3.** **Praxis:** Schreiben Sie eine Funktion `Get-BelegungProzent`, die zwei Größenangaben als Text erhält (belegt und Quota, etwa `'49.5 GB (53,150,220,288 bytes)'` und `'50 GB (53,687,091,200 bytes)'`) und den belegten Anteil in Prozent (eine Nachkommastelle) liefert. Nutzen Sie die Funktion `ConvertTo-Bytes` aus 10.3 und brechen Sie bei einer Quota von 0 mit einer klaren Meldung ab.

**Aufgabe 4.** **Praxis:** Schreiben Sie eine Funktion `Group-RessourcenNachTyp`, die Ressourcenobjekte (mit `Name`, `ResourceType`, `Location`) aus der Pipeline entgegennimmt und je Typ die Anzahl und die Liste der Standorte ausgibt, absteigend nach Anzahl.

**Aufgabe 5.** Ein Runbook soll unbeaufsichtigt Postfachstatistiken aus Exchange Online lesen. Welche Anmeldeart wählen Sie, was muss vorab eingerichtet sein, und welche zwei Stolperfallen nennt die Dokumentation zur zertifikatsbasierten Variante?

**Aufgabe 6.** **Praxis:** Erweitern Sie die Tag-Prüfung aus 10.7 um die Regel, dass `CostCenter` aus genau vier Ziffern bestehen muss. Fehlt das Tag, soll die bisherige Meldung erscheinen; ist es vorhanden, aber falsch, soll „CostCenter ’x’ ist nicht vierstellig“ gemeldet werden.

**Aufgabe 7.** **Praxis:** Schreiben Sie eine Funktion `Get-KostenBericht`, die eine CSV-Datei mit den Spalten aus 10.9 einliest und ein Objekt mit Gesamtsumme, den `N` teuersten Diensten (Summe und Anteil in Prozent) sowie Anzahl und Summe der Zeilen ohne Kostenstelle liefert.

**Aufgabe 8.** Ein Runbook bricht mit der Meldung „this.Client.SubscriptionId cannot be null“ ab, obwohl die Anmeldung mit `Connect-AzAccount -Identity` erfolgreich war. Was ist die Ursache, wie beheben Sie sie nach dem Prinzip der geringsten Rechte, und welche Fehler sollten Sie in der Praxis vermeiden?

## 10.11 Lösungen zu Modul 10

### Aufgabe 1.

Bei der delegierten Anmeldung handelt das Skript im Namen eines angemeldeten Benutzers; es darf höchstens, was der Benutzer darf und was die erteilte Berechtigung erlaubt. Bei app-only handelt es als eigene Anwendung ohne Benutzer, die Berechtigungen wirken mandantenweit und müssen von einem Administrator freigegeben werden. Für ein nächtliches Skript ohne Benutzer eignet sich app-only, mit einer rein lesenden Berechtigung (zum Beispiel `User.Read.All`) und bei Möglichkeit mit einer Managed Identity oder einem Zertifikat statt eines Geheimnisses. Die benötigte Berechtigung findet man mit `Find-MgGraphCommand` und in der Dokumentation des Cmdlets.

### Aufgabe 2.

Laut abgerufener Dokumentation wurden Run-As-Konten am 30. September 2023 eingestellt. Managed Identities brauchen kein Zertifikat, das abläuft und erneuert werden muss; Azure verwaltet die Anmeldedaten. Die systemzugewiesene Identität gehört fest zum Automation-Konto und verschwindet mit ihm; die benutzerzugewiesene ist eine eigene Ressource, die mehreren Ressourcen zugewiesen werden kann. Benutzerzugewiesene Identitäten werden in Azure Automation nur für Cloud-Aufträge unterstützt, nicht auf Hybrid Runbook Workern.

### Aufgabe 3.

```powershell
function Get-BelegungProzent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Belegt,
        [Parameter(Mandatory)] [string] $Quota
    )
    $b = $Belegt | ConvertTo-Bytes
    $q = $Quota  | ConvertTo-Bytes
    if ($q -le 0) { throw 'Quota muss größer als 0 sein.' }
    [math]::Round(100 * $b / $q, 1)
}
```

Beide Texte werden mit der Funktion aus 10.3 in Bytes umgewandelt; die Division durch null wird vorher ausgeschlossen. Im Test ergab `'49.5 GB (53,150,220,288 bytes)'` bei `'50 GB (53,687,091,200 bytes)'` den Wert 99, und `'2.1 GB'` bei `'50 GB'` den Wert 4,2.

### Aufgabe 4.

```powershell
function Group-RessourcenNachTyp {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)] [psobject] $Ressource)
    begin   { $liste = [System.Collections.Generic.List[object]]::new() }
    process { $liste.Add($Ressource) }
    end {
        $liste | Group-Object ResourceType | Sort-Object Count -Descending | ForEach-Object {
            [pscustomobject]@{
                Typ       = $_.Name
                Anzahl    = $_.Count
                Standorte = (($_.Group.Location | Sort-Object -Unique) -join ', ')
            }
        }
    }
}
```

Der `process`-Block sammelt die Objekte, und der `end`-Block gruppiert sie erst, wenn alle da sind: Gruppieren braucht die vollständige Liste. Im Test mit drei VMs (zwei in westeurope, eine in northeurope) und einem Speicherkonto kam: Virtual Machines, Anzahl 3, Standorte „northeurope, westeurope“; Storage Accounts, Anzahl 1, „westeurope“.

### Aufgabe 5.

Passend ist die Managed Identity des Automation-Kontos: `Connect-ExchangeOnline -ManagedIdentity` mit dem Parameter `-Organization` und der Domäne des Mandanten (`...onmicrosoft.com`). Vorab einzurichten sind die Anwendungsberechtigung `Exchange.ManageAsApp` mit Administratoreinwilligung und eine Rolle (Entra- oder Exchange-Rolle) für die Identität, nach dem Prinzip der geringsten Rechte. Bei der zertifikatsbasierten Variante nennt die Dokumentation: Die Anmeldung mit Fingerabdruck funktioniert nur unter Windows, und das Zertifikat muss mit einem CSP-Anbieter erzeugt sein, nicht mit CNG. Außerdem gehört `Disconnect-ExchangeOnline` in einen `finally`-Block.

### Aufgabe 6.

```powershell
function Test-TagKonformitaet2 {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)] [psobject] $Ressource)
    process {
        $tags = if ($Ressource.Tags) { $Ressource.Tags } else { @{} }
        $p = [System.Collections.Generic.List[string]]::new()
        foreach ($t in 'Owner', 'CostCenter', 'Environment') {
            if (-not $tags.ContainsKey($t) -or [string]::IsNullOrWhiteSpace($tags[$t])) {
                $p.Add("Tag '$t' fehlt")
            }
        }
        if ($tags.ContainsKey('CostCenter') -and $tags['CostCenter'] -and
            $tags['CostCenter'] -notmatch '^\d{4}$') {
            $p.Add("CostCenter '$($tags['CostCenter'])' ist nicht vierstellig")
        }
        [pscustomobject]@{ Name = $Ressource.Name; Konform = ($p.Count -eq 0); Probleme = $p -join '; ' }
    }
}
```

Die zusätzliche Regel greift nur, wenn das Tag vorhanden und nicht leer ist; sonst gilt die Meldung „fehlt“. Der reguläre Ausdruck `^\d{4}$` verlangt genau vier Ziffern von Anfang bis Ende. Test: Eine Ressource mit `CostCenter=4711` war konform, mit `47` kam „CostCenter ’47’ ist nicht vierstellig“, ohne das Tag „Tag ’CostCenter’ fehlt“.

### Aufgabe 7.

```powershell
function Get-KostenBericht {
    [CmdletBinding()]
    param([Parameter(Mandatory)] [string] $CsvPfad, [int] $Top = 2)
    $inv   = [Globalization.CultureInfo]::InvariantCulture
    $daten = Import-Csv -LiteralPath $CsvPfad | ForEach-Object {
        $_.Kosten = [double]::Parse($_.Kosten, $inv); $_
    }
    $gesamt  = ($daten | Measure-Object Kosten -Sum).Sum
    $spitze  = $daten | Group-Object Dienst | ForEach-Object {
        $s = ($_.Group | Measure-Object Kosten -Sum).Sum
        [pscustomobject]@{
            Dienst = $_.Name
            Summe  = [math]::Round($s, 2)
            AnteilProzent = [math]::Round(100 * $s / $gesamt, 1)
        }
    } | Sort-Object Summe -Descending | Select-Object -First $Top
    $ohne = @($daten | Where-Object { [string]::IsNullOrWhiteSpace($_.CostCenter) })
    [pscustomobject]@{
        Gesamt               = [math]::Round($gesamt, 2)
        Top                  = $spitze
        OhneCostCenterZeilen = $ohne.Count
        OhneCostCenterSumme  = [math]::Round(($ohne | Measure-Object Kosten -Sum).Sum, 2)
    }
}
```

Die Funktion wandelt die Kosten in Zahlen, rechnet Summen und Anteile je Dienst und gibt die ersten `N` zurück. Eine Stolperfalle aus dem Test: Die Zwischenvariable darf nicht `$top` heißen, denn PowerShell unterscheidet Groß- und Kleinschreibung bei Variablen nicht, und `$top` ist dieselbe Variable wie der Parameter `$Top`. Mein erster Versuch scheiterte deshalb mit der Meldung „Cannot convert the "System.Object\[\]" value of type "System.Object\[\]" to type "System.Int32"“, weil dem ganzzahligen Parameter plötzlich eine Liste zugewiesen wurde. Mit der Umbenennung in `$spitze` lief die Funktion; mit `-Top 2` und den Beispieldaten aus 10.9 ergab sich: Gesamt 583,85, Virtual Machines 273,50 (46,8 Prozent), SQL 210,25 (36,0 Prozent), eine Zeile ohne Kostenstelle mit 35.

### Aufgabe 8.

Die Identität kann sich anmelden, hat aber in der Subscription keine Rolle und „sieht“ sie deshalb nicht; die Dokumentation nennt genau diese Ursache für die Meldung. Behebung: Der Identität mit `New-AzRoleAssignment -ObjectId <PrincipalId> -RoleDefinitionName 'Reader' -Scope '/subscriptions/<Id>'` die Leserolle geben und für die eigentliche Aufgabe eine möglichst enge Rolle auf der kleinsten nötigen Ebene, etwa „Virtual Machine Contributor“ nur auf der Ressourcengruppe. Zu vermeiden sind: „Owner“ oder „Contributor“ auf der ganzen Subscription „damit es läuft“, die Rolle direkt einem Benutzer statt der Identität zu geben und den Fehler zu umgehen, indem man ein Kennwort ins Runbook schreibt. Die Befehle dieser Lösung wurden nicht ausgeführt.
