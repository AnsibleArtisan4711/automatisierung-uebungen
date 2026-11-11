# Bash – Lernskript

Lernskript zur Shell Bash in zwölf Modulen: von Grundlagen und Shell-Architektur über Dateisystem, Syntax, Pipelines und Textverarbeitung (grep, sed, awk, jq) bis zu robusten Skripten, Systemadministration, Security, Monitoring, Automation und CI/CD, Infrastructure as Code und Betrieb im größeren Maßstab. Jede Lerneinheit hat eine Erklärung, ein vollständiges Beispiel, typische Fehler mit Gegenbeispiel und einen Praxisbezug. Am Ende jedes Moduls stehen acht Übungsaufgaben mit Lösungen.

## Wichtig: Was dieses Repository (noch) ist

- Es enthält das **Skript als Markdown**, ein Kapitel pro Modul (`skript.md`), dazu [Einleitung](docs/skript-einleitung.md) und [Quellenverzeichnis](docs/skript-quellenverzeichnis.md) im Ordner `docs`.
- Eine eigenständige Übungsreihe mit Skripten, Testdateien und Lösungen je Modul ist **noch nicht** Teil des Repositories (die Übungsaufgaben am Kapitelende gehören zum Skript).
- **Die Beispiele wurden für dieses Repository nicht ausgeführt.** Alle 245 Codeblöcke sind zeichengleich aus der Quelle übernommen; die 242 als `bash` ausgezeichneten wurden mit `bash -n` auf Syntaxfehler geprüft (GNU bash 5.2), die drei übrigen sind keine Skripte (Gegenüberstellung, CSV-Daten, Markdown-Vorlage). Laut Einleitung hat der Autor die Beispiele ausgeführt, soweit sie ohne Spezialumgebung (Netzwerk, systemd) laufen.
- Viele Beispiele legen Dateien an, löschen sie oder verändern Systemzustand (Benutzer, Dienste, Pakete, Cron, Firewall). Nur in einem Testverzeichnis, einer VM, WSL oder einem Container ausprobieren, nie ungeprüft auf einem Produktivsystem.
- **Stundenangaben:** Die Summe der Lerneinheiten laut Detailblatt (113 Stunden) weicht bei acht der zwölf Module von der Dauer laut Modulblatt ab (zusammen 124 Stunden). Das Skript richtet sich nach dem jeweils höheren Wert; die Abweichungen sind in der Einleitung und im Kopf jedes Moduls vermerkt.
- Die Nachrecherche beruhte laut Quellenverzeichnis auf einer begrenzten Zahl abgerufener Seiten; weitere Quellen sind als „nicht abgerufen“ gekennzeichnet. Maßgeblich ist immer die aktuelle Dokumentation der eingesetzten Werkzeuge. Hinweise zu Datenschutz (Modul 08) sind keine Rechtsberatung.

## Module

| Modul | Thema | Dauer laut Modulblatt | Summe Detailblatt | Prio | Stand |
|---|---|---|---|---|---|
| [01](modul-01-grundlagen-shell-architektur/) | Grundlagen und Shell-Architektur | 8 h | 10 h | 1 | Skript verfügbar |
| [02](modul-02-dateisystem-navigation-textwerkzeuge/) | Dateisystem, Navigation und Textwerkzeuge | 10 h | 9 h | 1 | Skript verfügbar |
| [03](modul-03-syntax-variablen-kontrollstrukturen/) | Syntax, Variablen und Kontrollstrukturen | 10 h | 9 h | 1 | Skript verfügbar |
| [04](modul-04-pipelines-redirection-prozesse/) | Pipelines, Redirection und Prozesssteuerung | 10 h | 10 h | 1 | Skript verfügbar |
| [05](modul-05-textverarbeitung-grep-sed-awk-jq/) | Textverarbeitung mit grep, sed, awk und jq | 12 h | 9 h | 1 | Skript verfügbar |
| [06](modul-06-robuste-skripte-debugging/) | Robuste Skripte, Fehlerbehandlung, Debugging | 12 h | 9 h | 1 | Skript verfügbar |
| [07](modul-07-systemadministration/) | Systemadministration mit Bash | 12 h | 9 h | 2 | Skript verfügbar |
| [08](modul-08-security-hardening/) | Security, Hardening, sichere Skriptpraxis | 10 h | 9 h | 1 | Skript verfügbar |
| [09](modul-09-monitoring-logging-troubleshooting/) | Monitoring, Logging, Troubleshooting | 10 h | 10 h | 2 | Skript verfügbar |
| [10](modul-10-automation-scheduling-cicd/) | Automation, Scheduling, CI/CD | 10 h | 10 h | 2 | Skript verfügbar |
| [11](modul-11-iac-cloud-container/) | Infrastructure as Code, Cloud, Container | 10 h | 10 h | 2 | Skript verfügbar |
| [12](modul-12-migration-kosten-skalierung-bcdr/) | Migration, Kosten, Skalierung, BCDR | 10 h | 9 h | 3 | Skript verfügbar |

Priorität: 1 = höchste.

## Lizenz

- **Code** (Beispieldateien und Skripte, falls das Repository solche enthält): [MIT-Lizenz](LICENSE)
- **Texte** (die Modul-READMEs, die Kapitel `skript.md` und die Dateien unter `docs/`): [CC BY 4.0](LICENSE-DOCS.md)
- Linux, Git, GitHub, Docker, Kubernetes, Terraform, AWS, Azure und Google Cloud sind Marken ihrer jeweiligen Inhaber; dieses Repository steht in keiner Verbindung zu ihnen.

Autor: AnsibleArtisan4711
