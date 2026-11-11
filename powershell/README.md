# PowerShell – Lernskript

Lernskript zu PowerShell 7 in zwölf Modulen: von Grundlagen und Architektur über Cmdlets, Objekte und Pipeline, Skripting, Fehlerbehandlung und Systemadministration bis zu Remoting, Security, Monitoring, Cloud (Microsoft 365, Azure), Automation mit CI/CD und Infrastructure as Code sowie Migration, Kosten, Governance und BCDR. Jede Lerneinheit hat eine Erklärung, ein vollständiges Beispiel, typische Fehler mit Gegenbeispiel und einen Praxisbezug. Am Ende jedes Moduls stehen acht Übungsaufgaben mit Lösungen.

## Wichtig: Was dieses Repository (noch) ist

- Es enthält das **Skript als Markdown**, ein Kapitel pro Modul (`skript.md`), dazu [Einleitung](docs/skript-einleitung.md) und [Quellenverzeichnis](docs/skript-quellenverzeichnis.md) im Ordner `docs`.
- Eine eigenständige Übungsreihe mit Skripten, Testdateien und Lösungen je Modul ist **noch nicht** Teil des Repositories (die Übungsaufgaben am Kapitelende gehören zum Skript).
- **Die Beispiele wurden für dieses Repository nicht ausgeführt.** Alle 341 Codeblöcke sind zeichengleich aus der Quelle übernommen (339 als `powershell`, zwei als `text`). Laut Einleitung hat der Autor die Beispiele mit PowerShell 7.6.6 unter Linux ausgeführt und, wo nötig, korrigiert; Befehle, die nur unter Windows oder mit Zugriff auf die PowerShell Gallery funktionieren, sind im Text als „nur auf Syntax geprüft“ gekennzeichnet.
- Viele Beispiele legen Dateien an, löschen sie oder verändern Systemzustand (Dienste, Registry, Remoting, Berechtigungen, Cloud-Ressourcen). Nur in einem Testverzeichnis, einer VM oder einem Container ausprobieren, nie ungeprüft auf einem Produktivsystem.
- **Stundenangaben:** Die Summe der Lerneinheiten laut Detailblatt (112 Stunden) weicht bei 9 der zwölf Module von der Dauer laut Übersichtsblatt ab (zusammen 126 Stunden). Das Skript folgt laut Hinweisen in den Modulen dem Detailblatt; die Abweichungen sind im Kopf jedes Moduls angegeben.
- Das Quellenverzeichnis unterscheidet laut Skript zwischen tatsächlich abgerufenen und nicht abgerufenen Quellen. Maßgeblich ist immer die aktuelle Dokumentation von PowerShell und den eingesetzten Diensten. Hinweise zu Sicherheit und Governance sind keine Rechtsberatung.

## Module

| Modul | Thema | Dauer laut Übersichtsblatt | Summe Detailblatt | Prio | Stand |
|---|---|---|---|---|---|
| [01](modul-01-grundlagen-architektur/) | PowerShell Grundlagen und Architektur | 8 h | 10 h | 1 | Skript verfügbar |
| [02](modul-02-cmdlets-hilfe-module-provider/) | Cmdlets, Hilfe, Module und Provider | 10 h | 9 h | 1 | Skript verfügbar |
| [03](modul-03-objekte-pipeline-daten/) | Objekte, Pipeline und Datenverarbeitung | 10 h | 9 h | 1 | Skript verfügbar |
| [04](modul-04-skripting-variablen-kontrollstrukturen/) | Skripting, Variablen und Kontrollstrukturen | 10 h | 9 h | 1 | Skript verfügbar |
| [05](modul-05-fehlerbehandlung-debugging/) | Fehlerbehandlung, Debugging und robuste Skripte | 12 h | 9 h | 1 | Skript verfügbar |
| [06](modul-06-dateisystem-registry-systemadministration/) | Dateisystem, Registry und Systemadministration | 12 h | 9 h | 2 | Skript verfügbar |
| [07](modul-07-remoting-jobs-skalierung/) | Remoting, Jobs und Skalierung | 10 h | 10 h | 2 | Skript verfügbar |
| [08](modul-08-security-hardening-secrets/) | Security, Hardening und Secret Management | 12 h | 9 h | 1 | Skript verfügbar |
| [09](modul-09-monitoring-logging-troubleshooting/) | Monitoring, Logging und Troubleshooting | 10 h | 10 h | 2 | Skript verfügbar |
| [10](modul-10-cloud-m365-azure-automation/) | Cloud, Microsoft 365 und Azure Automation | 12 h | 9 h | 2 | Skript verfügbar |
| [11](modul-11-automation-cicd-iac/) | Automation, CI/CD und Infrastructure as Code | 10 h | 10 h | 2 | Skript verfügbar |
| [12](modul-12-migration-kosten-governance-bcdr/) | Migration, Kosten, Governance und BCDR | 10 h | 9 h | 3 | Skript verfügbar |

Priorität: 1 = höchste.

## Lizenz

- **Code** (Beispieldateien und Skripte, falls das Repository solche enthält): [MIT-Lizenz](LICENSE)
- **Texte** (die Modul-READMEs, die Kapitel `skript.md` und die Dateien unter `docs/`): [CC BY 4.0](LICENSE-DOCS.md)
- PowerShell, Windows, Microsoft 365, Azure, Linux, GitHub und weitere genannte Produkte sind Marken ihrer jeweiligen Inhaber; dieses Repository steht in keiner Verbindung zu ihnen.

Autor: AnsibleArtisan4711
