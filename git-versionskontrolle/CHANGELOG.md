# Changelog

Alle nennenswerten Änderungen an diesem Repository. Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).

## [Unveröffentlicht]

### Hinzugefügt

- Skript als Markdown: `skript.md` und `README.md` in jedem der 12 Modulordner, `docs/skript-einleitung.md` und `docs/skript-quellenverzeichnis.md` (aus der LaTeX-Quelle umgewandelt, 119 Codebeispiele zeichengleich geprüft, Inline-Codes vollständig übernommen)
- Lizenzen: MIT für Code (`LICENSE`), CC BY 4.0 für Texte (`LICENSE-DOCS.md`)

### Hinweise zur Umwandlung

- Lerneinheiten wie in der Quelle als Modul.Lerneinheit nummeriert (zum Beispiel 5.3); Übungen (8 Aufgaben je Modul, in Modul 03 neun) und Lösungen als eigene Abschnitte „Übungen zu Modul NN“ und „Lösungen zu Modul NN“ am Modulende
- Stundenangaben: Dauer laut Skript (128 h), Modulübersicht (126 h) und Summe der Lerneinheiten (114 h) weichen bei 7 Modulen ab; unverändert aus der Quelle übernommen (Modul 12 nennt im Hinweis „10 Seiten“ statt Stunden, so in der Quelle)
- Codeblöcke: 109 als `bash` (in der Quelle ohne Sprachangabe, Standardsprache `gitsh`), die übrigen Blöcke mit der Sprache `language={}` in der Quelle sind hier nach ihrer Überschrift als `yaml`, `json`, `groovy` oder `text` ausgezeichnet; Überschriften von Listings (caption) stehen als Zeile „Listing: …“ vor dem Block
- Shell-Beispiele mit `bash -n` und in Wegwerf-Verzeichnissen mit Git 2.43.0 ausgeführt (Einzelheiten in der README); Ausgaben nicht verglichen
- Mathematische Ausdrücke in Modul 03 und 06 als Text umgesetzt (zum Beispiel ·, ⌈ ⌉)
