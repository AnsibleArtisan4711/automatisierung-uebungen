# Python: Automatisierung unter Linux (Lernskript)

Lernskript in 15 Modulen zu 10 Stunden (75 Lerntage à 2 Stunden): Python für Administratoren, von Variablen und Schleifen über Dateien, Prozesse, Protokolle, Zeitsteuerung, Web-Schnittstellen, SSH und Alarmierung bis zu Git, Tests und einem Abschlussprojekt. Die Beispiele sind auf Linux ausgelegt, benötigt wird Python 3.11 oder neuer.

## Wichtig: Was dieses Repository (noch) ist

- Es enthält das **Skript als Markdown**, ein Kapitel pro Modul (`skript.md`), dazu [Einleitung](docs/skript-einleitung.md) und [Quellenverzeichnis](docs/skript-quellenverzeichnis.md) im Ordner `docs`.
- Eine eigenständige Übungsreihe mit Beispieldateien, Testskripten und Lösungen je Modul ist **noch nicht** Teil des Repositories (die Übungsaufgaben am Kapitelende gehören zum Skript).
- **Umwandlung:** Alle 948 Codeblöcke wurden maschinell mit der LaTeX-Quelle verglichen und sind zeichengleich (438 als `python`, 340 als `text`, 165 als `pycon`, 4 als `ini`, 1 als `bash`). Sprache der Blöcke: nach Inhalt bestimmt (Python-Syntax geprüft, sonst Konfiguration, Terminalsitzung oder Ausgabe), der Inhalt selbst ist unverändert.
- **Ausführung der Beispiele (von mir erneut geprüft, nur Module 01 bis 12):** Von den 313 abgedruckten Ausgaben der Module 01 bis 12 habe ich 305 durch Ausführen der Beispiele (Python 3.13) nachgestellt: 281 stimmen Zeichen für Zeichen überein (Leerzeichen am Zeilenende ignoriert), 10 Fehlerausgaben stimmen in der letzten Zeile überein, 14 weichen ab. Davon sind 12 in Modul 11 (dort fehlte in meiner Umgebung `ssh-keygen`/OpenSSH, die SSH-Beispiele ließen sich nicht ausführen) und 2 in Modul 06 (nur der Pfad in der Fehlermeldung: die Quelle zeigt `beispiel.py`). 8 Ausgaben sind in der Quelle als nicht vergleichbar gekennzeichnet (Eingaben, Messwerte); 39 Beispiele sind in der Quelle als nicht ausführbar markiert (zum Beispiel dauerhafte Dienste, Zugriff auf fremde Server).
- **Module 13 bis 15** (Git, Projektstruktur, Tests, Qualitätswerkzeuge, Abschlussprojekt) habe ich nicht neu ausgeführt: Die Terminalsitzungen mit `git`, `pytest`, `ruff` und `mypy` sind aus der Quelle übernommen. Die Einleitung nennt für sie Git 2.43.0, pytest 9.2.0, ruff 0.17.4, mypy 2.4.1 und setuptools 84.2.1.
- **Aussagen der Einleitung:** Sie gibt an, dass alle Beispiele ausgeführt und die Ausgaben verglichen wurden (Python 3.13.18; Pfade und Zeiten vereinheitlicht). Das ist die Angabe der Quelle; sie ist nur für die oben genannten Teile von mir nachgeprüft.
- **Zusatzpakete:** `paramiko` und `fabric` (Modul 11), `psutil` (Modul 11), `requests` (Modul 10), `PyYAML` (Modul 07), `aiosmtpd` (Modul 12, nur zum Testen), `pytest`, `ruff` und `mypy` (Modul 14). Die genauen Versionen stehen in den Kapiteln.
- **Priorität:** Der Lehrplan nennt keine Priorität; die Einleitung legt sie als Annahme fest (Blöcke 1 bis 6: 1, Blöcke 7 bis 12: 2, Blöcke 13 bis 15: 3).
- Das Quellenverzeichnis trennt abgerufene und weiterführende Quellen. Hinweise zu Datenschutz und Lizenzen sind keine Rechtsberatung.

## Module

| Modul | Thema | Dauer laut Skript | Prio | Codeblöcke |
|---|---|---|---|---|
| [01](modul-01-variablen-datentypen-strings/) | Variablen, Datentypen, Strings | 10 h | 1 | 53 |
| [02](modul-02-bedingungen-schleifen-listen/) | Bedingungen, Schleifen, Listen | 10 h | 1 | 53 |
| [03](modul-03-dictionaries-funktionen-tracebacks/) | Dictionaries, Funktionen, Tracebacks | 10 h | 1 | 52 |
| [04](modul-04-pathlib-dateien-os-shutil/) | pathlib, Dateien lesen-schreiben, os-shutil | 10 h | 1 | 41 |
| [05](modul-05-subprocess-argparse/) | subprocess, argparse | 10 h | 1 | 51 |
| [06](modul-06-exceptions-logging-exit-codes/) | Exceptions, logging, Exit-Codes | 10 h | 1 | 57 |
| [07](modul-07-module-venv-pip-json-yaml-csv/) | Module, venv-pip, JSON-YAML-CSV | 10 h | 2 | 63 |
| [08](modul-08-regex-datetime-textparsing/) | Regex, datetime, Textparsing | 10 h | 2 | 84 |
| [09](modul-09-produktiv-cron-systemd-timer-locking/) | Produktiv- cron, systemd-Timer, Locking | 10 h | 2 | 56 |
| [10](modul-10-requests-rest-apis-secrets/) | requests, REST-APIs, Secrets | 10 h | 2 | 62 |
| [11](modul-11-psutil-ssh-paramiko-fabric/) | psutil, SSH (paramiko-fabric) | 10 h | 2 | 67 |
| [12](modul-12-alerting-mail-webhook/) | Alerting- Mail & Webhook | 10 h | 2 | 53 |
| [13](modul-13-git-projektstruktur-pyproject/) | Git, Projektstruktur, pyproject.toml | 10 h | 3 | 72 |
| [14](modul-14-pytest-mocking-type-hints-ruff/) | pytest, Mocking, Type Hints, ruff | 10 h | 3 | 106 |
| [15](modul-15-abschlussprojekt-review-refactoring/) | Abschlussprojekt, Review, Refactoring | 10 h | 3 | 78 |

Priorität: 1 = höchste (Annahme der Einleitung).

## Lizenz

- **Code** (Beispieldateien und Skripte, falls das Repository solche enthält): [MIT-Lizenz](LICENSE)
- **Texte** (die Modul-READMEs, die Kapitel `skript.md` und die Dateien unter `docs/`): [CC BY 4.0](LICENSE-DOCS.md)
- Genannte Produkte (zum Beispiel Python, Git, pytest, ruff, mypy, paramiko, Fabric, psutil) sind Marken oder Werke ihrer jeweiligen Inhaber; dieses Repository steht in keiner Verbindung zu ihnen.

Autor: AnsibleArtisan4711
