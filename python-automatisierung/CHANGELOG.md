# Changelog

Alle nennenswerten Änderungen an diesem Repository. Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).

## [Unveröffentlicht]

### Hinzugefügt

- Skript als Markdown: `skript.md` und `README.md` in jedem der 15 Modulordner, `docs/skript-einleitung.md` und `docs/skript-quellenverzeichnis.md` (aus der LaTeX-Quelle umgewandelt, 948 Codeblöcke zeichengleich geprüft)
- Lizenzen: MIT für Code (`LICENSE`), CC BY 4.0 für Texte (`LICENSE-DOCS.md`)

### Hinweise zur Umwandlung

- Lerneinheiten wie in der Quelle als Modul.Lerneinheit nummeriert (zum Beispiel 3.5); „Übungsaufgaben zu Modul NN“ (8 Aufgaben) und „Lösungen zu Modul NN“ als Abschnitte am Modulende
- Modulkopf (Dauer, Summe, Priorität, Meilenstein, Lernziele) als Absätze mit fetten Bezeichnungen, Hinweisboxen als Zitatblöcke mit fettem Titel
- Codeblöcke ohne Sprachangabe in der Quelle sind nach Inhalt als `python`, `ini`, `bash`, `pycon` oder `text` ausgezeichnet (Inhalt unverändert)
- Beispiele der Module 01 bis 12 erneut ausgeführt und mit den abgedruckten Ausgaben verglichen (Ergebnis und Abweichungen in der README); Module 13 bis 15 nicht
- Auffälligkeit der Quelle, unverändert übernommen: In Modul 11 steht an zwei Stellen im Text (Pfade zu `authorized_keys` und `known_hosts`) das LaTeX-Makro `\textasciitilde` buchstäblich statt der Tilde `~`
