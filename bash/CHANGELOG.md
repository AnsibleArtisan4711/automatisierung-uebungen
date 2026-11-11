# Changelog

Alle nennenswerten Änderungen an diesem Repository. Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).

## [Unveröffentlicht]

### Hinzugefügt

- Skript als Markdown: `skript.md` und `README.md` in jedem der 12 Modulordner, `docs/skript-einleitung.md` und `docs/skript-quellenverzeichnis.md` (aus der LaTeX-Quelle umgewandelt, 245 Codebeispiele zeichengleich geprüft, 4.609 Inline-Codes vollständig übernommen)
- Lizenzen: MIT für Code (`LICENSE`), CC BY 4.0 für Texte (`LICENSE-DOCS.md`)

### Hinweise zur Umwandlung

- Überschriften wie in der Quelle als Modul.Lerneinheit nummeriert (zum Beispiel 5.3); Übungsaufgaben und Lösungen als eigene Abschnitte am Modulende (je Modul 8 Aufgaben und 8 Lösungen)
- Stundenangaben: Dauer laut Modulblatt (124 h) und Summe der Lerneinheiten laut Detailblatt (113 h) weichen bei acht Modulen ab; unverändert aus der Quelle übernommen
- Codeblöcke: 242 als `bash`, die Gegenüberstellung „falsch/richtig“ (Modul 03) und die CSV-Daten (Modul 05) als `text`, die Runbook-Vorlage (Modul 09) als `markdown`; die Bash-Blöcke wurden mit `bash -n` auf Syntax geprüft, nicht ausgeführt
- Mathematische Zeichen in der Einleitung (zum Beispiel +2, −1) und in Modul 03, 04 und 12 als Text umgesetzt
