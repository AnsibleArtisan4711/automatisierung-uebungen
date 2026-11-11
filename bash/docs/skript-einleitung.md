# Einleitung

> **Hinweis:** Teil des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt. Die Kapitel zu den Modulen stehen als `skript.md` in den Modulordnern.

Dieses Skript begleitet den Lehrplan „Lehrplan - Bash“ und soll es ermöglichen, die Shell Bash von Grund auf zu verstehen, ohne weitere Quellen zu benötigen. Es folgt der Gliederung des Lehrplans: Jedes Modul entspricht einem Modul des Lehrplans, jede Lerneinheit aus dem Blatt „Lehrplan in Details“ bildet einen eigenen Abschnitt. Da viele Einträge des Lehrplans nur allgemeine Formulierungen enthalten, wurden die Inhalte aus dem Titel der Lerneinheit fachlich erschlossen und ergänzt.

## Grundlage und Quellenlage

Im Projektwissen und in den angebundenen Ablagen lagen keine eigenen Kursunterlagen (.tex-Dateien) vor. Der Inhalt basiert deshalb auf dem Lehrplan, auf dem Fachwissen des Autors und auf einer begrenzten Nachrecherche im Web. Ergänzungen aus der Nachrecherche sind in grünen Boxen gekennzeichnet; welche Quellen tatsächlich abgerufen wurden, ist im Quellenverzeichnis getrennt von weiterführender Literatur aufgeführt. Alle Beispielskripte wurden mit einer aktuellen Bash-Version unter Linux ausgeführt, soweit sie ohne Spezialumgebung (zum Beispiel Netzwerk, systemd) laufen.

## Aufbau jedes Abschnitts

Jede Lerneinheit besteht aus erklärender Prosa, mindestens einem vollständigen, lauffähigen Beispiel mit Erläuterung, typischen Fehlern mit Gegenbeispiel und einem Praxisbezug. Drei Hinweisboxen ergänzen den Text: *Merke* (blau) fasst das Wichtigste zusammen, *Ergänzung aus der Nachrecherche* (grün) kennzeichnet recherchierte Zusatzinformationen, *Praxisbeispiel* (orange) zeigt den Einsatz im Projektalltag. Am Ende jedes Moduls stehen Übungsaufgaben mit steigender Schwierigkeit; die Lösungen mit Erklärung folgen in einem eigenen Abschnitt.

## Stundenangaben im Lehrplan

Die beiden Blätter des Lehrplans nennen nicht für alle Module dieselbe Dauer. Das Blatt „Lehrplan in Modulen“ gibt die Gesamtdauer je Modul an, das Blatt „Lehrplan in Details“ die Dauer je Lerneinheit. Die Summen der Einzeleinheiten weichen bei acht der zwölf Module von der Modulangabe ab. Als Umfangsrichtwert (mindestens eine Seite je Stunde) gilt in diesem Skript jeweils der höhere Wert; die Abweichungen sind im Kopf jedes Moduls vermerkt.

| Nr. | Modul                                                   | Modulblatt (h) | Details (h) | Differenz |
|:----|:--------------------------------------------------------|---------------:|------------:|----------:|
| 01  | Bash Grundlagen und Shell-Architektur                   |              8 |          10 |        +2 |
| 02  | Dateisystem, Navigation und Textwerkzeuge               |             10 |           9 |        −1 |
| 03  | Bash-Syntax, Variablen und Kontrollstrukturen           |             10 |           9 |        −1 |
| 04  | Pipelines, Redirection und Prozesssteuerung             |             10 |          10 |         0 |
| 05  | Textverarbeitung mit grep, sed, awk und jq              |             12 |           9 |        −3 |
| 06  | Robuste Skripte, Fehlerbehandlung und Debugging         |             12 |           9 |        −3 |
| 07  | Systemadministration mit Bash                           |             12 |           9 |        −3 |
| 08  | Security, Hardening und sichere Skriptpraxis            |             10 |           9 |        −1 |
| 09  | Monitoring, Logging und Troubleshooting                 |             10 |          10 |         0 |
| 10  | Automation, Scheduling und CI/CD                        |             10 |          10 |         0 |
| 11  | Infrastructure as Code, Cloud und Container-Integration |             10 |          10 |         0 |
| 12  | Migration, Kosten, Skalierung und BCDR                  |             10 |           9 |        −1 |
|     | **Summe**                                               |        **124** |     **113** |       −11 |

## Stand der Ausarbeitung

Dieser Stand enthält alle zwölf Module des Lehrplans (Priorität 1, 2 und 3), ausgearbeitet in vier Etappen nach Priorität. Inhalt basiert auf dem Lehrplan, dem Fachwissen des Autors und der im Quellenverzeichnis genannten Nachrecherche; Kursunterlagen lagen nicht vor.

## Arbeitsumgebung

Für alle Beispiele genügt eine Linux-Umgebung mit Bash ab Version 4.3 (VM, WSL oder Container). Einzelne Beispiele nennen höhere Anforderungen ausdrücklich. Empfohlen werden ShellCheck, Git und `jq`. Führen Sie Beispiele, die Dateien anlegen oder löschen, stets in einem Testverzeichnis aus; die meisten Skripte erzeugen dafür selbst ein temporäres Verzeichnis mit `mktemp -d`.
