# Git und Versionskontrolle: Lernskript

Lernskript zu Git in zwölf Modulen: von den Grundlagen der Versionskontrolle und dem Git-Datenmodell über Repositories, Commits, Branching, Merging und Konfliktlösung, Remote-Repositories, Workflows, Historie und Recovery sowie Tags, Releases und Semantic Versioning bis zu Security und Supply Chain, CI/CD, Betrieb und Troubleshooting, Enterprise-Plattformen und Governance sowie Migration, Kosten, Skalierung und BCDR. Jede Lerneinheit hat eine Erklärung, mindestens ein vollständiges Beispiel, typische Fehler mit Gegenbeispiel und einen Praxisbezug. Am Ende jedes Moduls stehen Übungsaufgaben mit Lösungen.

## Wichtig: Was dieses Repository (noch) ist

- Es enthält das **Skript als Markdown**, ein Kapitel pro Modul (`skript.md`), dazu [Einleitung](docs/skript-einleitung.md) und [Quellenverzeichnis](docs/skript-quellenverzeichnis.md) im Ordner `docs`.
- Eine eigenständige Übungsreihe mit Beispielrepositories, Testskripten und Lösungen je Modul ist **noch nicht** Teil des Repositories (die Übungsaufgaben am Kapitelende gehören zum Skript).
- **Prüfstand der Beispiele:** Alle 119 Codeblöcke sind zeichengleich aus der Quelle übernommen (109 als `bash`, dazu 1 groovy, 1 json, 5 text, 3 yaml). Die 109 Shell-Beispiele wurden mit `bash -n` (GNU bash 5.2) auf Syntax geprüft und jeweils in einem leeren Wegwerf-Verzeichnis mit Git 2.43.0 ausgeführt, mit eigenem Home-Verzeichnis und ohne Zugangsdaten für Plattformen. Dabei liefen 104 eigenständig mit Exit-Code 0. Fünf enden mit einem anderen Code: drei zeigen absichtlich die Ablehnung durch einen Hook beziehungsweise einen Prüf-Exit-Code 1, zwei sind Fortsetzungen eines vorherigen Beispiels (Modul 03, Konfliktlösung) und brauchen dessen Zustand. Die ausgegebenen Texte wurden **nicht** mit den im Skript gezeigten verglichen. Plattform-Konfigurationen (YAML, JSON, Jenkinsfile) wurden nicht ausgeführt; laut Einleitung hat der Autor die Beispiele mit Git 2.43 getestet.
- Viele Beispiele ändern Repositories, Konfiguration oder Verlauf (Rebase, Reset, Filter, Force-Push). Nur in einem leeren Übungsordner oder einem Wegwerf-Repository ausprobieren, nie in einem echten Projekt.
- **Stundenangaben:** Dauer laut Skript 128 Stunden insgesamt. Bei 7 der zwölf Module weicht die Modulübersicht im Lehrplan (zusammen 126 Stunden) von der Summe der Lerneinheiten (zusammen 114 Stunden) ab; das Skript folgt dem jeweils höheren Wert und nennt beide im Modulkopf.
- Das Quellenverzeichnis unterscheidet zwischen tatsächlich abgerufenen und nicht abgerufenen Quellen (Stand der Abrufe laut Skript: 28.10.2026). Maßgeblich ist immer die aktuelle Git-Dokumentation. Hinweise zu Lizenzen, Datenschutz und Compliance sind keine Rechtsberatung.

## Module

| Modul | Thema | Dauer laut Skript | Modulübersicht | Summe Lerneinheiten | Prio | Aufgaben | Stand |
|---|---|---|---|---|---|---|---|
| [01](modul-01-grundlagen-versionskontrolle/) | Git Grundlagen und Versionskontrollarchitektur | 10 h | 8 h | 10 h | 1 | 8 | Skript verfügbar |
| [02](modul-02-repositories-commits-arbeitsbereiche/) | Repositories, Commits und Arbeitsbereiche | 10 h | 10 h | 9 h | 1 | 8 | Skript verfügbar |
| [03](modul-03-branching-merging-konflikte/) | Branching, Merging und Konfliktlösung | 12 h | 12 h | 9 h | 1 | 9 | Skript verfügbar |
| [04](modul-04-remote-kollaboration/) | Remote Repositories und Kollaboration | 10 h | 10 h | 10 h | 1 | 8 | Skript verfügbar |
| [05](modul-05-workflows-repository-standards/) | Git Workflows und Repository-Standards | 12 h | 12 h | 9 h | 1 | 8 | Skript verfügbar |
| [06](modul-06-historie-rewriting-recovery/) | Historie, Rewriting und Recovery | 12 h | 12 h | 9 h | 1 | 8 | Skript verfügbar |
| [07](modul-07-tags-releases-semver/) | Tags, Releases und Semantic Versioning | 10 h | 10 h | 10 h | 2 | 8 | Skript verfügbar |
| [08](modul-08-security-secrets-supply-chain/) | Security, Secrets und Supply Chain | 12 h | 12 h | 9 h | 1 | 8 | Skript verfügbar |
| [09](modul-09-git-in-cicd-automation/) | Git in CI/CD und Automation | 10 h | 10 h | 10 h | 2 | 8 | Skript verfügbar |
| [10](modul-10-betrieb-monitoring-troubleshooting/) | Betrieb, Monitoring und Troubleshooting | 10 h | 10 h | 10 h | 2 | 8 | Skript verfügbar |
| [11](modul-11-enterprise-plattformen-governance/) | Enterprise Git, Plattformen und Governance | 10 h | 10 h | 10 h | 2 | 8 | Skript verfügbar |
| [12](modul-12-migration-kosten-skalierung-bcdr/) | Migration, Kosten, Skalierung und BCDR | 10 h | 10 h | 9 h | 3 | 8 | Skript verfügbar |

Priorität: 1 = höchste.

## Lizenz

- **Code** (Beispieldateien und Skripte, falls das Repository solche enthält): [MIT-Lizenz](LICENSE)
- **Texte** (die Modul-READMEs, die Kapitel `skript.md` und die Dateien unter `docs/`): [CC BY 4.0](LICENSE-DOCS.md)
- Git, GitHub, GitLab, Azure DevOps, Jenkins, Bitbucket, Linux und weitere genannte Produkte sind Marken ihrer jeweiligen Inhaber; dieses Repository steht in keiner Verbindung zu ihnen.

Autor: AnsibleArtisan4711
