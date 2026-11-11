# Modul 13 - Git, Projektstruktur, pyproject.toml

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (13.1 bis 13.5), danach folgen „Übungsaufgaben zu Modul 13“ (8 Aufgaben) und „Lösungen zu Modul 13“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 08.02.2027 bis Fr 12.02.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 3 (Annahme: Der Lehrplan nennt keine Priorität; die Module 13 bis 15 bilden die Phase „Qualität und Abschluss“)

**Meilenstein:** Projekt im Git-Repo

**Lernziele.** Nach diesem Modul können Sie mit Git die Geschichte eines Projekts festhalten und nachvollziehen (Repository, Staging-Bereich, Commits, Branches, Merge und Konflikte), Änderungen rückgängig machen, ohne Arbeit zu verlieren, und mit einer `.gitignore`-Datei verhindern, dass Unerwünschtes oder Geheimes im Repository landet. Sie können aus einem einzelnen Skript ein ordentliches Python-Projekt mit `src`-Layout, Paket und Einstiegspunkt machen, es mit einer `pyproject.toml` beschreiben und installierbar machen, eine brauchbare `README.md` schreiben und das fertige Projekt in ein Repository bringen, das ein Kollege klonen und sofort benutzen kann. Das Modul bildet die Grundlage für die beiden folgenden: Tests und Qualitätswerkzeuge (Modul 14) sowie das Abschlussprojekt (Modul 15) setzen ein sauber versioniertes Projekt voraus.

| **Nr.** | **Lerneinheit**               | **Datum**     | **Dauer** | **Prio** |
|:--------|:------------------------------|:--------------|:----------|:---------|
| 13.1    | Git Grundlagen                | Mo 08.02.2027 | 2 h       | 3        |
| 13.2    | Branches, Commits, .gitignore | Di 09.02.2027 | 2 h       | 3        |
| 13.3    | Projektstruktur               | Mi 10.02.2027 | 2 h       | 3        |
| 13.4    | pyproject.toml & README       | Do 11.02.2027 | 2 h       | 3        |
| 13.5    | Projekt ins Git-Repo + Quiz   | Fr 12.02.2027 | 2 h       | 3        |
|         | **Summe**                     |               | **10 h**  |          |

> **Merke**
>
> Für dieses Modul brauchen Sie Git und Python 3.11 oder neuer unter Linux; `pip install` lädt bei der Einrichtung des Projekts ein Bauwerkzeug aus dem Internet. Getestet wurde mit Git 2.43.0, Python 3.13.18 und setuptools 84.2.1. Alle Mitschnitte in diesem Modul stammen aus echten Läufen in einem frischen Benutzerverzeichnis; die Ausgabe der Git-Befehle hängt von Ihrer Git-Version ab und kann in Einzelheiten abweichen. Die Kennungen der Commits (siebenstellige Hexadezimalzahlen wie `a1b2c3d`) sind bei Ihnen anders, weil sie von Inhalt, Zeitpunkt und Autor abhängen. Wo im Text ein Verzeichnis `/home/max` steht, ist Ihr eigenes Heimverzeichnis gemeint.

## 13.1 Git Grundlagen

Jeder, der länger an einem Skript arbeitet, kennt das Problem: Es gibt `backup.py`, `backup_neu.py`, `backup_neu2.py` und `backup_final_wirklich.py`, und niemand weiß mehr, welche Datei welchen Stand hat und warum eine Zeile geändert wurde. Sobald zwei Menschen an derselben Datei arbeiten, wird es schlimmer: Wer hat zuletzt gespeichert, und wessen Änderung ist verloren gegangen? Eine *Versionsverwaltung* (englisch *version control system*, kurz *VCS*) löst diese Probleme systematisch. Sie hält jeden Zustand eines Projekts fest, merkt sich, wer wann was und warum geändert hat, erlaubt es, zu jedem früheren Zustand zurückzukehren, und führt die Arbeit mehrerer Personen zusammen. Das Werkzeug, das sich dafür durchgesetzt hat, heißt *Git*. Es wurde 2005 von Linus Torvalds für die Entwicklung des Linux-Kernels geschrieben, und fast jedes Open-Source-Projekt und die meisten Firmen verwenden es heute.

**Wozu Git im Admin-Alltag?** Auch wenn Sie keine Software für Kunden schreiben, lohnt sich Git. Ihre Skripte, Konfigurationsdateien, Ansible-Playbooks und Dokumentation sind Dateien, die sich ändern, und bei jeder Störung lautet die erste Frage: *Was hat sich geändert?* Mit Git beantworten Sie sie mit einem Befehl. Außerdem ist Git die Voraussetzung für fast alles Weitere: Zusammenarbeit, Prüfung von Änderungen durch Kollegen, automatische Tests (Modul 14) und die Bereitstellung von Skripten auf Servern.

**Das Grundmodell.** Um Git zu verstehen, hilft ein Bild mit drei Bereichen. Das *Arbeitsverzeichnis* (*working tree*) ist der Ordner, in dem Sie ganz normal Dateien bearbeiten. Der *Staging-Bereich* (auch *Index*, englisch *staging area*) ist eine Art Ablage für die Änderungen, die in den *nächsten* Commit aufgenommen werden sollen. Das *Repository* schließlich ist die Datenbank im versteckten Verzeichnis `.git`, in der alle gespeicherten Zustände liegen. Ein *Commit* ist ein solcher gespeicherter Zustand: ein Schnappschuss aller Dateien des Projekts zusammen mit dem Autor, dem Zeitpunkt, einer Beschreibung und einem Verweis auf den vorherigen Commit. Dass es den Zwischenschritt „Staging“ gibt, wirkt anfangs umständlich, ist aber nützlich: Sie können in einer Arbeitssitzung mehrere Dinge ändern und daraus mehrere saubere, thematisch getrennte Commits bilden.

Wichtig für das Verständnis ist, dass Git keine Unterschiede („Deltas“) von Version zu Version speichert, sondern bei jedem Commit den kompletten Zustand der Dateien, wobei unveränderte Dateien nur einmal vorliegen und nur wiederverwendet werden. Jeder Commit bekommt eine *Kennung*, einen Hash, der aus dem Inhalt berechnet wird (40 Hexadezimalziffern; meist genügen die ersten sieben). Ändert sich auch nur ein Zeichen in der Geschichte, ändern sich die Kennungen aller späteren Commits. Das macht die Geschichte praktisch fälschungssicher und erklärt, warum man sie nicht unbedacht umschreibt. Zwei weitere Begriffe brauchen wir gleich: *HEAD* ist der Zeiger auf den Commit, auf dem Sie gerade stehen, und ein *Branch* (Zweig) ist ein benannter Zeiger auf einen Commit, der beim Weiterarbeiten mitwandert; das vertiefen wir in 13.2. Git ist außerdem *verteilt*: Jede Kopie eines Repositorys enthält die vollständige Geschichte, und man kann offline arbeiten; ein Server (GitHub, GitLab, Gitea oder ein eigener) dient nur als gemeinsamer Treffpunkt.

**Einmalige Einrichtung.** Bevor Sie den ersten Commit machen, muss Git wissen, wer Sie sind, denn Name und E-Mail-Adresse werden in jeden Commit geschrieben. Das tragen Sie einmal global ein. Wir legen außerdem fest, dass der erste Zweig eines neuen Repositorys `main` heißen soll. Der Befehl `git config --global --list` zeigt anschließend, was in Ihrer Benutzerkonfiguration (Datei `~/.gitconfig`) steht.

```pycon
~$ git config --global user.name "Max Muster"
~$ git config --global user.email "max.muster@beispiel.example"
~$ git config --global init.defaultBranch main
~$ git config --global --list
user.name=Max Muster
user.email=max.muster@beispiel.example
init.defaultbranch=main
```

Die Namen und die Adresse sind hier erfunden; tragen Sie Ihre eigenen ein. Wenn Sie später Ihre Arbeit auf einen öffentlichen Server stellen, wird diese Adresse sichtbar; bei GitHub und anderen Diensten kann man dafür eine anonymisierte Adresse verwenden. Die Einstellung `init.defaultBranch` ist wichtig, weil Git ohne sie je nach Version einen anderen Namen für den ersten Zweig wählt und dazu einen Hinweis ausgibt.

> **Ergänzung aus der Nachrecherche**
>
> **Standard-Zweigname.** Die Git-Dokumentation zu `git init` (abgerufen) beschreibt, dass ohne die Option `--initial-branch` und ohne die Einstellung `init.defaultBranch` derzeit der Name `master` verwendet wird und dass sich dieser Standard mit Git 3.0 auf `main` ändern soll. Wer die Einstellung wie oben setzt, ist davon unabhängig. Viele Dienste und Anleitungen sind inzwischen auf `main` umgestellt; ältere Repositorys heißen noch `master`. Die Befehle in diesem Modul funktionieren mit beiden Namen.

**Das erste Repository.** Jetzt legen wir ein Projekt an. Es begleitet uns durch die drei Module: ein kleines Programm namens *plattenwacht*, das meldet, wenn ein Dateisystem fast voll ist. Das Programm kennen Sie in Grundzügen aus den Modulen 5, 6 und 12; hier ist es bewusst klein, damit die Werkzeuge im Mittelpunkt stehen. Zuerst legen wir ein Verzeichnis an, wechseln hinein und machen es mit `git init` zum Repository. Danach schreiben wir die erste Version des Skripts.

```pycon
~$ mkdir plattenwacht
~$ cd plattenwacht
~/plattenwacht$ git init
Initialized empty Git repository in /home/max/plattenwacht/.git/
~/plattenwacht$ ls -a
.
..
.git
```

Der Befehl `git init` hat das Verzeichnis `.git` angelegt (`ls -a` zeigt auch versteckte Einträge). Dort lebt die gesamte Geschichte; löschen Sie es, ist das Projekt wieder ein gewöhnlicher Ordner. Fassen Sie den Inhalt von `.git` nie von Hand an. Nun die erste Version des Programms:

**Datei** `plattenwacht.py`**:**

```python
"""plattenwacht - meldet, wenn ein Dateisystem fast voll ist."""
import shutil
import sys

SCHWELLE_WARNUNG = 80
SCHWELLE_KRITISCH = 90


def belegung_prozent(pfad):
    nutzung = shutil.disk_usage(pfad)
    return nutzung.used / nutzung.total * 100


def bewerte(prozent):
    if prozent >= SCHWELLE_KRITISCH:
        return "KRITISCH"
    if prozent >= SCHWELLE_WARNUNG:
        return "WARNUNG"
    return "OK"


def main():
    pfad = sys.argv[1] if len(sys.argv) > 1 else "/"
    prozent = belegung_prozent(pfad)
    stufe = bewerte(prozent)
    print(f"{stufe}: {pfad} ist zu {prozent:.1f} % belegt")
    return {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}[stufe]


if __name__ == "__main__":
    sys.exit(main())
```

Das Skript misst mit `shutil.disk_usage` (Modul 4) die Belegung eines Pfades, bewertet sie gegen zwei Schwellen und gibt den Schweregrad als Exit-Code zurück (Modul 6): 0 für in Ordnung, 1 für Warnung, 2 für kritisch, wie es Überwachungswerkzeuge erwarten. Die Funktion `bewerte` steht absichtlich getrennt von der Messung; warum, sehen wir in Modul 14. Jetzt fragen wir Git, was es von der neuen Datei hält. Der Befehl `git status` ist der wichtigste Begleiter: Er zeigt, in welchem Zustand sich jede Datei befindet, und sagt oft sogar, welcher Befehl als Nächstes passt.

```pycon
~/plattenwacht$ git status
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	plattenwacht.py

nothing added to commit but untracked files present (use "git add" to track)
```

Git kennt die Datei noch nicht: Sie steht unter „Untracked files“, also unter den Dateien, die nicht versioniert werden. Wir nehmen sie in den Staging-Bereich auf (`git add`), prüfen den Zustand erneut und legen dann den Commit an (`git commit`). Mit der Option `-m` geben wir die Beschreibung direkt mit; ohne sie öffnet Git einen Texteditor.

```pycon
~/plattenwacht$ git add plattenwacht.py
~/plattenwacht$ git status
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   plattenwacht.py
~/plattenwacht$ git commit -m "Erste Version: Belegung prüfen und bewerten"
[main (root-commit) 36f2196] Erste Version: Belegung prüfen und bewerten
 1 file changed, 31 insertions(+)
 create mode 100644 plattenwacht.py
~/plattenwacht$ git log
commit 36f219611a78ddb165dc4735a2cab8d1c2090cc5
Author: Max Muster <max.muster@beispiel.example>
Date:   Mon Feb 8 19:36:00 2027 +0100

    Erste Version: Belegung prüfen und bewerten
```

Nach dem `git add` steht die Datei unter „Changes to be committed“; sie ist im Staging-Bereich, aber noch nicht gespeichert. Der Commit meldet in der ersten Zeile den Zweig (`main`), die Kennung und den Betreff. Beachten Sie den Zusatz `root-commit`: Es ist der erste Commit, er hat keinen Vorgänger. Das `git log` zeigt dann die Geschichte mit Kennung, Autor, Zeitpunkt und Beschreibung.

**Änderungen ansehen: diff.** Als Nächstes verbessern wir das Programm. Die Ausgabe soll immer den Wert mit einer Nachkommastelle zeigen, aber der Pfad soll in Anführungszeichen stehen, damit Leerzeichen im Pfad erkennbar sind. Wir ändern also die Zeile mit dem `print`. Damit die Änderung im Mitschnitt sichtbar bleibt, nehmen wir sie hier mit dem Werkzeug `sed` vor (Ersetzen von Text in einer Datei, Modul 8 hat Ähnliches mit regulären Ausdrücken gezeigt); in der Praxis ändern Sie die Zeile einfach in Ihrem Editor. Danach zeigt `git diff` genau, was sich gegenüber dem letzten Commit geändert hat.

```pycon
~/plattenwacht$ sed -i "s|{pfad} ist|'{pfad}' ist|" plattenwacht.py
~/plattenwacht$ git status --short
 M plattenwacht.py
~/plattenwacht$ git diff
diff --git a/plattenwacht.py b/plattenwacht.py
index d6c1954..6a75851 100644
--- a/plattenwacht.py
+++ b/plattenwacht.py
@@ -23,7 +23,7 @@ def main():
     pfad = sys.argv[1] if len(sys.argv) > 1 else "/"
     prozent = belegung_prozent(pfad)
     stufe = bewerte(prozent)
-    print(f"{stufe}: {pfad} ist zu {prozent:.1f} % belegt")
+    print(f"{stufe}: '{pfad}' ist zu {prozent:.1f} % belegt")
     return {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}[stufe]
```

Das Format von `git diff` nennt man *Unified Diff*: Die Kopfzeilen nennen die Datei, die Zeilen `@@ ... @@` geben die Stelle an, Zeilen mit `-` wurden entfernt, Zeilen mit `+` sind neu, und die Zeilen davor und danach (ohne Zeichen) sind Kontext. Dasselbe Format wird bei Code-Reviews und bei der Zusammenarbeit überall verwendet; es lohnt sich, es lesen zu lernen. Die Kurzform `git status --short` zeigt pro Datei zwei Spalten: links den Zustand im Staging-Bereich, rechts den im Arbeitsverzeichnis. Das `M` in der rechten Spalte heißt „geändert, aber nicht vorgemerkt“. Jetzt merken wir die Änderung vor und vergleichen noch einmal: `git diff` zeigt nur Änderungen, die noch *nicht* im Staging-Bereich sind, `git diff --staged` zeigt die vorgemerkten.

```pycon
~/plattenwacht$ git add plattenwacht.py
~/plattenwacht$ git diff
~/plattenwacht$ git diff --staged
diff --git a/plattenwacht.py b/plattenwacht.py
index d6c1954..6a75851 100644
--- a/plattenwacht.py
+++ b/plattenwacht.py
@@ -23,7 +23,7 @@ def main():
     pfad = sys.argv[1] if len(sys.argv) > 1 else "/"
     prozent = belegung_prozent(pfad)
     stufe = bewerte(prozent)
-    print(f"{stufe}: {pfad} ist zu {prozent:.1f} % belegt")
+    print(f"{stufe}: '{pfad}' ist zu {prozent:.1f} % belegt")
     return {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}[stufe]
 
 
~/plattenwacht$ git commit -m "Pfad in der Ausgabe in Anführungszeichen setzen"
[main 3581387] Pfad in der Ausgabe in Anführungszeichen setzen
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git log --oneline
3581387 Pfad in der Ausgabe in Anführungszeichen setzen
36f2196 Erste Version: Belegung prüfen und bewerten
```

Nach dem `git add` ist `git diff` leer, weil nichts Unvorgemerktes mehr da ist, und `git diff --staged` zeigt dieselbe Änderung wie vorher. Die Option `--oneline` verdichtet das Protokoll auf eine Zeile pro Commit. Mit `git show` sehen Sie einen einzelnen Commit samt seiner Änderung; ohne Argument ist es der letzte, und mit einer Kennung (`git show a1b2c3d`) ein beliebiger früherer.

```pycon
~/plattenwacht$ git show --stat HEAD
commit 3581387130f5fff465617a018df529c920d0de7f
Author: Max Muster <max.muster@beispiel.example>
Date:   Mon Feb 8 20:00:00 2027 +0100

    Pfad in der Ausgabe in Anführungszeichen setzen

 plattenwacht.py | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Die Option `--stat` fasst zusammen, welche Dateien wie stark verändert wurden. Das ist der schnellste Weg, um zu verstehen, was ein Commit getan hat.

**Der Lebenszyklus einer Datei.** Aus den Beispielen ergibt sich der Kreislauf, den Sie hundertfach wiederholen werden. Eine neue Datei ist zunächst *untracked*. Mit `git add` wird sie *staged*, also für den nächsten Commit vorgemerkt, und mit `git commit` ist sie *committed*, also gespeichert. Ändern Sie eine bereits gespeicherte Datei, ist sie *modified*, und der Kreislauf aus `git add` und `git commit` beginnt von vorn. Zwischendurch sind `git status` und `git diff` Ihre Augen. Eine gute Gewohnheit ist es, vor jedem Commit `git status` und `git diff --staged` anzusehen: Was Sie dort nicht erwartet haben, gehört meist nicht in den Commit.

**Gute Commits und gute Nachrichten.** Ein Repository ist nur so nützlich wie seine Geschichte. Zwei Regeln helfen. Erstens: *Ein Commit, ein Gedanke.* Wer in einem Commit einen Fehler behebt, eine Funktion umbenennt und ein neues Feature beginnt, macht es sich und anderen schwer, denn man kann nichts davon einzeln nachvollziehen oder rückgängig machen. Zweitens: Die Beschreibung erklärt das *Warum*, nicht nur das Was; das Was sieht man im `git diff`. Üblich ist eine kurze Betreffzeile von höchstens etwa 50 Zeichen, im Präsens oder Imperativ formuliert („Schwellen konfigurierbar machen“), und bei Bedarf nach einer Leerzeile ein ausführlicher Text. Nachrichten wie „Änderungen“, „fix“ oder „asdf“ sind wertlos, wenn man in einem halben Jahr den Grund für eine Zeile sucht. Eine verbreitete Konvention, die Commit-Typen wie `feat:` und `fix:` vorschreibt (*Conventional Commits*), ist in manchen Teams Pflicht; für den Anfang genügen verständliche Sätze.

**Typische Fehler und Stolperfallen.** Der häufigste Anfängerfehler ist, nach dem Ändern einer Datei direkt `git commit` aufzurufen und sich zu wundern, dass „nothing added to commit“ gemeldet wird: Ohne vorheriges `git add` ist nichts vorgemerkt. Wer Änderungen an bereits versionierten Dateien schnell speichern will, kann `git commit -am "Text"` verwenden; das nimmt alle geänderten, *bereits bekannten* Dateien mit, aber keine neuen. Der zweite Fehler ist ein zu großzügiges `git add .` oder `git add -A`, das alles im Verzeichnis vormerkt, auch Zwischendateien, Protokolle oder gar eine Datei mit Passwörtern. Der dritte ist ein Commit ohne `-m`, der in einem Editor (oft `vim`) landet, den man nicht zu bedienen weiß; wer darin hängt, verlässt ihn ohne Speichern mit `Esc` und danach `:q!`. Das Gegenüber:

```pycon
~/plattenwacht$ echo "# Notizen" > notizen.txt
~/plattenwacht$ git add .
~/plattenwacht$ git status --short
A  notizen.txt
~/plattenwacht$ git restore --staged notizen.txt
~/plattenwacht$ git add plattenwacht.py
~/plattenwacht$ git status --short
?? notizen.txt
~/plattenwacht$ rm notizen.txt
```

Im ersten Teil hat `git add .` die Notizdatei mit vorgemerkt (Spalte links: `A`, „added“); die Notizen wären im Repository gelandet. `git restore --staged` nimmt die Datei wieder aus dem Staging-Bereich, ohne sie zu löschen. Danach bleibt sie „untracked“ (`??`), bis wir sie entfernen. Die Regel lautet: Fügen Sie Dateien gezielt mit Namen hinzu, oder prüfen Sie nach einem `git add .` mit `git status`, was darin steckt. Gegen versehentlich eingefügte Dateien hilft außerdem die `.gitignore`-Datei, die wir in 13.2 behandeln.

> **Merke**
>
> Git speichert Schnappschüsse. Drei Bereiche: Arbeitsverzeichnis, Staging-Bereich, Repository (`.git`). Der Kreislauf lautet `git status` prüfen, `git add` vormerken, `git commit -m` speichern. `git diff` zeigt Unvorgemerktes, `git diff --staged` Vorgemerktes, `git log --oneline` die Geschichte. Name und Adresse einmalig mit `git config --global` eintragen. Ein Commit, ein Gedanke; die Nachricht erklärt das Warum. Dateien gezielt hinzufügen.

> **Praxisbeispiel**
>
> Wer Konfigurationsdateien von Servern in ein Git-Repository legt (zum Beispiel `/etc` mit dem Werkzeug `etckeeper` oder die eigenen Playbooks), hat bei jeder Störung eine Zeitmaschine: `git log -p datei` zeigt, wer wann welche Zeile geändert hat, und `git diff` gegen den gestrigen Stand zeigt, was sich zwischen *es lief* und *es läuft nicht* verändert hat. Auch für ein einzelnes Skript lohnt sich das Repository ab dem ersten Tag; es kostet zwei Befehle und bewahrt vor dem Chaos der Dateien mit `_final` im Namen.

## 13.2 Branches, Commits, .gitignore

Mit dem Kreislauf aus `git add` und `git commit` kommt man weit, solange man allein und geradlinig arbeitet. Schwierig wird es, sobald man etwas Neues ausprobieren will, ohne den funktionierenden Stand zu gefährden, oder sobald zwei Menschen gleichzeitig an verschiedenen Dingen arbeiten. Dafür gibt es *Branches*. Ein Branch (Zweig) ist, technisch gesehen, nichts weiter als ein benannter Zeiger auf einen Commit, der automatisch weiterwandert, wenn Sie auf diesem Branch einen neuen Commit machen. Weil das so leicht ist (eine Datei mit 41 Zeichen), legt man in Git Branches großzügig an: für jede Aufgabe einen, mit einem sprechenden Namen wie `feature/schwellen` oder `fix/tippfehler`. Der Zweig `main` bleibt dabei der Stand, der immer funktionieren soll.

**Einen Branch anlegen und wechseln.** Mit `git branch` listen Sie die vorhandenen Branches auf; der aktuelle ist mit einem Stern markiert. Mit `git switch -c name` legen Sie einen neuen Branch an und wechseln gleich darauf (das `-c` steht für *create*). Der Befehl `git switch` ist der neuere und übersichtlichere Weg, den Branch zu wechseln; die Git-Dokumentation (abgerufen) beschreibt ihn als „Wechseln auf einen bestimmten Branch“, wobei Arbeitsverzeichnis und Staging-Bereich an den Branch angepasst werden. Älteren Anleitungen begegnet man oft noch mit `git checkout`, das mehr kann und dadurch leichter verwirrt.

```pycon
~/plattenwacht$ git branch
* main
~/plattenwacht$ git switch -c feature/schwellen
Switched to a new branch 'feature/schwellen'
~/plattenwacht$ git branch
* feature/schwellen
  main
```

Wir sind jetzt auf dem neuen Branch `feature/schwellen`. Er zeigt auf denselben Commit wie `main`; erst der nächste Commit lässt sie auseinanderlaufen. Auf diesem Branch bauen wir ein, was in der ersten Version fehlte: Die Schwellen sollen sich auf der Kommandozeile einstellen lassen (`argparse`, Modul 5). Wir ersetzen dazu das Skript durch eine erweiterte Fassung.

**Datei** `plattenwacht.py`**:**

```python
"""plattenwacht - meldet, wenn ein Dateisystem fast voll ist."""
import argparse
import shutil
import sys

SCHWELLE_WARNUNG = 80
SCHWELLE_KRITISCH = 90


def belegung_prozent(pfad):
    nutzung = shutil.disk_usage(pfad)
    return nutzung.used / nutzung.total * 100


def bewerte(prozent, warnung=SCHWELLE_WARNUNG, kritisch=SCHWELLE_KRITISCH):
    if prozent >= kritisch:
        return "KRITISCH"
    if prozent >= warnung:
        return "WARNUNG"
    return "OK"


def main():
    parser = argparse.ArgumentParser(description="Meldet, wenn ein Dateisystem fast voll ist.")
    parser.add_argument("pfad", nargs="?", default="/", help="zu prüfender Pfad (Standard: /)")
    parser.add_argument("--warnung", type=float, default=SCHWELLE_WARNUNG, help="Schwelle für WARNUNG in Prozent")
    parser.add_argument("--kritisch", type=float, default=SCHWELLE_KRITISCH, help="Schwelle für KRITISCH in Prozent")
    args = parser.parse_args()
    prozent = belegung_prozent(args.pfad)
    stufe = bewerte(prozent, args.warnung, args.kritisch)
    print(f"{stufe}: '{args.pfad}' ist zu {prozent:.1f} % belegt")
    return {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}[stufe]


if __name__ == "__main__":
    sys.exit(main())
```

Die Funktion `bewerte` nimmt die Schwellen jetzt als Parameter mit den bisherigen Werten als Vorgabe, und `main` liest sie mit `argparse` aus den Optionen `--warnung` und `--kritisch`. Wir testen das Programm kurz und speichern die Arbeit als Commit auf dem Branch.

```pycon
~/plattenwacht$ python3 plattenwacht.py --help
usage: plattenwacht.py [-h] [--warnung WARNUNG] [--kritisch KRITISCH] [pfad]

Meldet, wenn ein Dateisystem fast voll ist.

positional arguments:
  pfad                 zu prüfender Pfad (Standard: /)

options:
  -h, --help           show this help message and exit
  --warnung WARNUNG    Schwelle für WARNUNG in Prozent
  --kritisch KRITISCH  Schwelle für KRITISCH in Prozent
~/plattenwacht$ python3 plattenwacht.py . --warnung 0 --kritisch 100; echo "Exit-Code: $?"
WARNUNG: '.' ist zu 4.8 % belegt
Exit-Code: 1
~/plattenwacht$ git commit -am "Schwellen per Kommandozeile einstellbar machen"
[feature/schwellen f3b5ef0] Schwellen per Kommandozeile einstellbar machen
 1 file changed, 12 insertions(+), 7 deletions(-)
~/plattenwacht$ git log --oneline --all --graph
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
```

Der Aufruf mit den Schwellen 0 und 100 ergibt immer eine Warnung (Exit-Code 1), egal wie voll der Datenträger ist; der genaue Prozentwert ist bei Ihnen ein anderer. Die Option `-a` von `git commit -am` nimmt alle geänderten, bereits bekannten Dateien mit. Die Grafik von `git log --oneline --all --graph` zeigt die Geschichte als Baum: das Sternchen markiert einen Commit, und die Namen in Klammern zeigen, wo die Branches und `HEAD` gerade stehen. Der neue Commit liegt nur auf `feature/schwellen`; `main` ist unverändert. Das prüfen wir, indem wir zurückwechseln.

```pycon
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ grep -c argparse plattenwacht.py || true
0
~/plattenwacht$ git switch feature/schwellen
Switched to branch 'feature/schwellen'
~/plattenwacht$ grep -c argparse plattenwacht.py
2
~/plattenwacht$ git switch main
Switched to branch 'main'
```

Beim Wechsel ändert Git die Dateien im Arbeitsverzeichnis passend zum Branch: Auf `main` kommt `argparse` nicht vor (Ergebnis 0), auf dem Feature-Branch zweimal. Genau das macht Branches so nützlich, denn ein Experiment kann nichts kaputt machen, was auf `main` liegt.

**Zusammenführen: Merge.** Ist das Feature fertig, führt man den Branch in `main` zusammen (*merge*). Man steht dazu auf dem Ziel-Branch (hier `main`) und nennt den Branch, dessen Arbeit hineinkommen soll. Weil `main` seit dem Abzweigen nicht verändert wurde, genügt es, den Zeiger einfach nach vorn zu schieben; Git nennt das *Fast-Forward* und legt keinen zusätzlichen Commit an.

```pycon
~/plattenwacht$ git merge feature/schwellen
Updating 3581387..f3b5ef0
Fast-forward
 plattenwacht.py | 19 ++++++++++++-------
 1 file changed, 12 insertions(+), 7 deletions(-)
~/plattenwacht$ git log --oneline --all --graph
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
~/plattenwacht$ git branch -d feature/schwellen
Deleted branch feature/schwellen (was f3b5ef0).
```

Danach steht `main` auf dem Commit des Features. Der Branch `feature/schwellen` ist überflüssig geworden und wird mit `git branch -d` gelöscht; die Commits bleiben erhalten, nur der Name verschwindet. Interessanter ist der Fall, in dem *beide* Seiten Neues haben. Wir bauen ihn nach: Von demselben Ausgangspunkt zweigen wir drei Branches ab. In `fix/warnschwelle-75` ändert eine Kollegin die Warnschwelle auf 75 Prozent, in `fix/warnschwelle-85` ändert ein Kollege sie auf 85 Prozent (und verbessert nebenbei die Beschreibung im Hilfetext), und in `docs/readme` entsteht eine Beschreibungsdatei. Jeder Branch erhält einen Commit.

```pycon
~/plattenwacht$ git switch -c fix/warnschwelle-75
Switched to a new branch 'fix/warnschwelle-75'
~/plattenwacht$ sed -i 's/^SCHWELLE_WARNUNG = 80/SCHWELLE_WARNUNG = 75/' plattenwacht.py
~/plattenwacht$ git commit -am "Warnschwelle auf 75 Prozent senken (Richtlinie)"
[fix/warnschwelle-75 97a458a] Warnschwelle auf 75 Prozent senken (Richtlinie)
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ git switch -c fix/warnschwelle-85
Switched to a new branch 'fix/warnschwelle-85'
~/plattenwacht$ sed -i 's/^SCHWELLE_WARNUNG = 80/SCHWELLE_WARNUNG = 85/' plattenwacht.py
~/plattenwacht$ sed -i 's/Meldet, wenn ein Dateisystem fast voll ist\./Meldet fast volle Dateisysteme (Exit-Codes 0, 1, 2)./' plattenwacht.py
~/plattenwacht$ git commit -am "Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)"
[fix/warnschwelle-85 7bd0f26] Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)
 1 file changed, 2 insertions(+), 2 deletions(-)
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ git switch -c docs/readme
Switched to a new branch 'docs/readme'
~/plattenwacht$ echo "# plattenwacht" > README.md
~/plattenwacht$ git add README.md
~/plattenwacht$ git commit -m "README anlegen"
[docs/readme 1b92d75] README anlegen
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ git log --oneline --all --graph
* 1b92d75 README anlegen
| * 7bd0f26 Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)
|/  
| * 97a458a Warnschwelle auf 75 Prozent senken (Richtlinie)
|/  
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
```

Die Grafik zeigt jetzt drei Äste, die vom gleichen Commit abgehen. Wir führen sie nacheinander in `main` zusammen. Der erste Merge (75 Prozent) ist wieder ein Fast-Forward. Der zweite (README) ist ein echter Merge: `main` hat sich inzwischen bewegt, und Git muss beide Seiten kombinieren. Weil die Branches unterschiedliche Dateien betreffen, gelingt das von selbst, und Git legt einen eigenen *Merge-Commit* mit zwei Vorgängern an. Die Option `--no-edit` übernimmt die vorgeschlagene Nachricht, ohne den Editor zu öffnen.

```pycon
~/plattenwacht$ git merge fix/warnschwelle-75
Updating f3b5ef0..97a458a
Fast-forward
 plattenwacht.py | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git merge --no-edit docs/readme
Merge made by the 'ort' strategy.
 README.md | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
~/plattenwacht$ git log --oneline --graph
*   644d6be Merge branch 'docs/readme'
|\  
| * 1b92d75 README anlegen
* | 97a458a Warnschwelle auf 75 Prozent senken (Richtlinie)
|/  
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
```

Jetzt der Ernstfall: Auf dem Branch `fix/warnschwelle-85` wurde dieselbe Zeile (`SCHWELLE_WARNUNG`) anders geändert als auf `main`. Hier kann Git nicht entscheiden, welche Fassung gilt, und bricht den Merge mit einem *Konflikt* ab, ohne etwas zu raten. Das ist keine Störung, sondern gewünschtes Verhalten: Eine Entscheidung, die fachlich ist, soll ein Mensch treffen. Die Änderung am Hilfetext in derselben Datei betrifft eine andere Stelle und wird trotzdem automatisch übernommen.

```pycon
~/plattenwacht$ git merge --no-edit fix/warnschwelle-85
Auto-merging plattenwacht.py
CONFLICT (content): Merge conflict in plattenwacht.py
Automatic merge failed; fix conflicts and then commit the result.
~/plattenwacht$ git status --short
UU plattenwacht.py
~/plattenwacht$ grep -n -A6 '^<<<<<<<' plattenwacht.py
6:<<<<<<< HEAD
7-SCHWELLE_WARNUNG = 75
8-=======
9-SCHWELLE_WARNUNG = 85
10->>>>>>> fix/warnschwelle-85
11-SCHWELLE_KRITISCH = 90
12-
```

Der Merge steht jetzt „mittendrin“ (`git status` zeigt `UU`, „unmerged“, für die betroffene Datei). In der Datei hat Git die Konfliktstelle mit Markierungen versehen: Zwischen `<<<<<<<` und `=======` steht die Fassung des aktuellen Branches (`HEAD`, also `main`), zwischen `=======` und `>>>>>>>` die Fassung des fremden Branches. Die Aufgabe lautet nun, die Datei so zu bearbeiten, dass sie am Ende genau das enthält, was gelten soll, und alle drei Markierungszeilen zu entfernen. Man öffnet dazu die Datei im Editor; viele Editoren (VS Code, PyCharm) zeigen Konflikte farbig und bieten Schaltflächen an. Wir einigen uns hier auf den Mittelwert 80 und ersetzen den ganzen Block durch eine einzige Zeile, was mit `sed` geht. Danach muss man Git mitteilen, dass der Konflikt gelöst ist: durch `git add` der Datei, und der Merge wird mit `git commit` abgeschlossen.

```pycon
~/plattenwacht$ sed -i '/^<<<<<<< /,/^>>>>>>> /c SCHWELLE_WARNUNG = 80' plattenwacht.py
~/plattenwacht$ grep -n 'SCHWELLE_WARNUNG =\|<<<<\|>>>>' plattenwacht.py
6:SCHWELLE_WARNUNG = 80
~/plattenwacht$ python3 plattenwacht.py --help | head -3
usage: plattenwacht.py [-h] [--warnung WARNUNG] [--kritisch KRITISCH] [pfad]

Meldet fast volle Dateisysteme (Exit-Codes 0, 1, 2).
~/plattenwacht$ git add plattenwacht.py
~/plattenwacht$ git commit --no-edit
[main f82da4b] Merge branch 'fix/warnschwelle-85'
~/plattenwacht$ git log --oneline --graph
*   f82da4b Merge branch 'fix/warnschwelle-85'
|\  
| * 7bd0f26 Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)
* |   644d6be Merge branch 'docs/readme'
|\ \  
| * | 1b92d75 README anlegen
| |/  
* / 97a458a Warnschwelle auf 75 Prozent senken (Richtlinie)
|/  
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
~/plattenwacht$ git branch -d fix/warnschwelle-75 fix/warnschwelle-85 docs/readme
Deleted branch fix/warnschwelle-75 (was 97a458a).
Deleted branch fix/warnschwelle-85 (was 7bd0f26).
Deleted branch docs/readme (was 1b92d75).
```

Prüfen Sie nach der Lösung eines Konflikts immer, ob die Datei noch funktioniert (hier genügte der Aufruf der Hilfe) und ob keine Markierungen übrig sind (`grep` nach den Zeichenfolgen). Wer mitten im Konflikt feststellt, dass er sich verlaufen hat, kann mit `git merge --abort` zum Zustand vor dem Merge zurückkehren. Die Grafik zeigt am Ende zwei Merge-Commits, in denen die Geschichte wieder zusammenläuft.

**Änderungen rückgängig machen.** Fehler passieren, und Git bietet für jede Lage das passende Werkzeug; entscheidend ist, die Lage zu unterscheiden. Hat man eine Datei im Arbeitsverzeichnis verändert und will die Änderung *wegwerfen* (noch nicht committet), hilft `git restore datei`. Das ist endgültig: Nicht Committetes kann Git nicht zurückholen.

```pycon
~/plattenwacht$ echo "kaputt" >> plattenwacht.py
~/plattenwacht$ git diff --stat
 plattenwacht.py | 1 +
 1 file changed, 1 insertion(+)
~/plattenwacht$ git restore plattenwacht.py
~/plattenwacht$ git status --short
```

Die Zeile `kaputt` ist weg, die Datei steht wieder so da wie im letzten Commit; die leere Ausgabe von `git status --short` bestätigt es. Ist die Änderung schon committet und liegt weiter zurück, sollte man sie nicht aus der Geschichte tilgen, sondern durch einen neuen Commit *aufheben*: `git revert` erzeugt einen Commit, der die Änderung eines früheren Commits umkehrt. Die Geschichte bleibt unverändert nachvollziehbar, und der Weg ist auch dann sicher, wenn der Commit schon auf einem gemeinsamen Server liegt.

```pycon
~/plattenwacht$ sed -i 's/^SCHWELLE_KRITISCH = 90/SCHWELLE_KRITISCH = 50/' plattenwacht.py
~/plattenwacht$ git commit -am "Kritische Schwelle auf 50 senken (Experiment)"
[main 2a047c9] Kritische Schwelle auf 50 senken (Experiment)
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git revert --no-edit HEAD
[main 32fd663] Revert "Kritische Schwelle auf 50 senken (Experiment)"
 Date: Mon Feb 8 22:57:00 2027 +0100
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git log --oneline -3
32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
2a047c9 Kritische Schwelle auf 50 senken (Experiment)
f82da4b Merge branch 'fix/warnschwelle-85'
~/plattenwacht$ grep -n '^SCHWELLE_KRITISCH' plattenwacht.py
7:SCHWELLE_KRITISCH = 90
```

Der Revert-Commit steht als eigener Eintrag in der Geschichte, und die Schwelle hat wieder den alten Wert. Ein Tippfehler in der *letzten* Commit-Nachricht lässt sich mit `git commit --amend -m "neuer Text"` korrigieren; das ersetzt den letzten Commit durch einen neuen. Das ist harmlos, solange der Commit nur bei Ihnen liegt; sobald andere ihn bereits haben, schreibt man die Geschichte nicht mehr um. Das mächtigste und gefährlichste Werkzeug ist `git reset --hard`, das den Branch auf einen früheren Commit zurücksetzt und dabei auch das Arbeitsverzeichnis überschreibt. Auch hier ist nicht alles verloren, denn Git führt ein Protokoll, wohin `HEAD` gezeigt hat: das *Reflog*.

```pycon
~/plattenwacht$ git reset --hard HEAD~1
HEAD is now at 2a047c9 Kritische Schwelle auf 50 senken (Experiment)
~/plattenwacht$ git log --oneline -2
2a047c9 Kritische Schwelle auf 50 senken (Experiment)
f82da4b Merge branch 'fix/warnschwelle-85'
~/plattenwacht$ git reflog -4
2a047c9 HEAD@{0}: reset: moving to HEAD~1
32fd663 HEAD@{1}: revert: Revert "Kritische Schwelle auf 50 senken (Experiment)"
2a047c9 HEAD@{2}: commit: Kritische Schwelle auf 50 senken (Experiment)
f82da4b HEAD@{3}: commit (merge): Merge branch 'fix/warnschwelle-85'
~/plattenwacht$ git reset --hard "HEAD@{1}"
HEAD is now at 32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
~/plattenwacht$ git log --oneline -2
32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
2a047c9 Kritische Schwelle auf 50 senken (Experiment)
```

Wir haben mit `HEAD~1` („ein Commit vor HEAD“) den Revert-Commit absichtlich weggeworfen. Das Reflog verrät die Kennungen der früheren Stände, und mit `git reset --hard "HEAD@{1}"` holen wir den Zustand von vorher zurück. Das Reflog ist ein lokales Sicherheitsnetz (es wird nach einiger Zeit aufgeräumt und existiert nicht auf dem Server), aber es rettet viele „Ich habe alles gelöscht“-Momente. Zwei weitere Hilfsmittel: `git switch --detach HEAD~2` lässt Sie einen früheren Stand nur *ansehen* (der Zustand heißt „detached HEAD“, weil kein Branch mehr mitwandert), und `git switch -` springt zum zuletzt benutzten Branch zurück. Und mit `git stash` legen Sie unfertige Änderungen kurz beiseite, etwa wenn Sie dringend den Branch wechseln müssen.

```pycon
~/plattenwacht$ echo "# in Arbeit" >> plattenwacht.py
~/plattenwacht$ git stash
Saved working directory and index state WIP on main: 32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
~/plattenwacht$ git status --short
~/plattenwacht$ git stash list
stash@{0}: WIP on main: 32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
~/plattenwacht$ git stash pop
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   plattenwacht.py

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (ba9f1e16eb68dad4107913f15ffe2461c3f16fae)
~/plattenwacht$ git restore plattenwacht.py
```

**Die .gitignore-Datei.** Neben den eigenen Dateien entstehen beim Arbeiten Dateien, die nicht ins Repository gehören: der Zwischenspeicher `__pycache__`, den Python anlegt, virtuelle Umgebungen (`.venv`, Modul 7), Protokolle, Editor-Dateien und vor allem Dateien mit Zugangsdaten. Sie in jedem `git status` als „untracked“ zu sehen, stört, und noch schlimmer ist es, sie mit einem unachtsamen `git add .` einzusammeln. Die Datei `.gitignore` im Hauptverzeichnis legt Muster fest, die Git ignoriert. Zuerst sehen wir das Problem: Ein Import des Programms erzeugt den Zwischenspeicher.

```pycon
~/plattenwacht$ python3 -c "import plattenwacht"
~/plattenwacht$ git status --short
?? __pycache__/
```

Die Struktur der Datei ist einfach. Nach der Git-Dokumentation (abgerufen) gilt: Leere Zeilen werden ignoriert, Zeilen mit `#` am Anfang sind Kommentare, ein Muster mit `/` am Ende trifft nur Verzeichnisse, ein `*` steht für beliebige Zeichen außer dem Schrägstrich, ein `**` auch über Verzeichnisgrenzen hinweg, ein `/` am Anfang bindet das Muster an das Verzeichnis der `.gitignore`-Datei, und ein `!` davor hebt ein früheres Muster auf. Wir schreiben eine Datei, die für Python-Projekte üblich ist.

**Datei** `.gitignore`**:**

```text
# Python
__pycache__/
*.pyc
*.egg-info/
build/
dist/

# Virtuelle Umgebungen und Werkzeug-Zwischenspeicher
.venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Geheimnisse und lokale Einstellungen: niemals committen
.env
*.env

# Protokolle
*.log
```

Die Datei gruppiert die Muster mit Kommentaren und nennt die Zwischenspeicher der Werkzeuge, die wir in Modul 14 verwenden werden (`.pytest_cache`, `.mypy_cache`, `.ruff_cache`). Wirkt sie? Mit `git status --short` prüfen wir, und `git check-ignore -v` nennt für eine Datei das Muster, das sie ausschließt, was beim Suchen von Fehlern hilft.

```pycon
~/plattenwacht$ git status --short
?? .gitignore
~/plattenwacht$ git check-ignore -v __pycache__/plattenwacht.cpython-313.pyc
.gitignore:2:__pycache__/	__pycache__/plattenwacht.cpython-313.pyc
~/plattenwacht$ git add .gitignore
~/plattenwacht$ git commit -m ".gitignore für Python-Projekte anlegen"
[main 87e60c0] .gitignore für Python-Projekte anlegen
 1 file changed, 19 insertions(+)
 create mode 100644 .gitignore
```

Der Zwischenspeicher taucht nicht mehr auf, nur die neue `.gitignore` selbst, die man bewusst mit versioniert, damit alle Beteiligten dieselben Regeln haben. Eine Falle betrifft Dateien, die *schon vor der Regel* versioniert wurden: Die `.gitignore` gilt nur für Dateien, die Git noch nicht kennt. Wir stellen den Fehler nach: Eine Protokolldatei wird versehentlich committet, danach erst kommt die Regel.

```pycon
~/plattenwacht$ echo "Lauf 1: ok" > debug.log
~/plattenwacht$ git add -f debug.log
~/plattenwacht$ git commit -m "Protokoll versehentlich mit eingecheckt"
[main d383641] Protokoll versehentlich mit eingecheckt
 1 file changed, 1 insertion(+)
 create mode 100644 debug.log
~/plattenwacht$ echo "Lauf 2: ok" >> debug.log
~/plattenwacht$ git status --short
 M debug.log
~/plattenwacht$ git rm --cached debug.log
rm 'debug.log'
~/plattenwacht$ git commit -m "Protokolldatei aus dem Repository nehmen"
[main b63aa85] Protokolldatei aus dem Repository nehmen
 1 file changed, 1 deletion(-)
 delete mode 100644 debug.log
~/plattenwacht$ git status --short
```

Obwohl `*.log` in der `.gitignore` steht, meldet Git die Änderung an `debug.log` (Spalte `M`), solange die Datei versioniert ist; wir mussten sie mit `-f` sogar erzwingen, um den Fall nachzustellen. Die Lösung ist `git rm --cached`: Es entfernt die Datei aus der Versionsverwaltung, *lässt sie aber auf der Festplatte* liegen, und von da an greift die Regel. Beachten Sie, dass die Datei in der älteren Geschichte weiter vorhanden ist. Bei einem Protokoll ist das harmlos, bei einem Passwort nicht, wie das nächste Beispiel zeigt.

**Geheimnisse gehören nicht ins Repository.** Wer ein Passwort, einen API-Schlüssel oder einen privaten SSH-Schlüssel committet, hat ein Problem, das sich nicht mit einem späteren Löschen beheben lässt, weil die Geschichte alles bewahrt. Wir spielen es mit einer erfundenen Datei durch: Eine Datei mit einem Passwort wird eingecheckt und im nächsten Commit gelöscht. Dann suchen wir mit der Pickaxe-Suche `git log -S`, welche Commits die Zeichenfolge je verändert haben.

```pycon
~/plattenwacht$ echo "PASSWORT=geheim123" > zugang.txt
~/plattenwacht$ git add -f zugang.txt
~/plattenwacht$ git commit -m "Zugangsdaten ergänzen"
[main 28297a1] Zugangsdaten ergänzen
 1 file changed, 1 insertion(+)
 create mode 100644 zugang.txt
~/plattenwacht$ git rm -q zugang.txt
~/plattenwacht$ git commit -m "Zugangsdaten wieder entfernen"
[main 54351aa] Zugangsdaten wieder entfernen
 1 file changed, 1 deletion(-)
 delete mode 100644 zugang.txt
~/plattenwacht$ git log --oneline -S"geheim123"
54351aa Zugangsdaten wieder entfernen
28297a1 Zugangsdaten ergänzen
~/plattenwacht$ git show HEAD~1:zugang.txt
PASSWORT=geheim123
```

Die Suche findet beide Commits (den, der das Passwort einführte, und den, der es entfernte), und `git show HEAD~1:zugang.txt` gibt es im Klartext aus. Wer das Repository später veröffentlicht, veröffentlicht das Passwort mit. Die Konsequenz ist eindeutig: Ein einmal committetes Geheimnis gilt als *kompromittiert* und muss geändert werden (Passwort neu setzen, Schlüssel widerrufen); erst danach kann man, wenn nötig, die Geschichte mit Spezialwerkzeugen bereinigen (zum Beispiel `git filter-repo`, hier nicht ausgeführt). Besser ist es, Geheimnisse gar nicht erst in die Nähe des Repositorys zu bringen: in Umgebungsvariablen (Modul 10), in Dateien außerhalb des Projekts oder in Dateien, die in der `.gitignore` stehen.

**Typische Fehler und Stolperfallen.** Am häufigsten ist die Meldung, dass lokale Änderungen beim Wechsel überschrieben würden. Git verweigert den Branch-Wechsel, wenn dabei nicht committete Arbeit verloren ginge. Wir stellen es nach.

```pycon
~/plattenwacht$ git switch -c versuch
Switched to a new branch 'versuch'
~/plattenwacht$ sed -i 's/ist zu/liegt bei/' plattenwacht.py
~/plattenwacht$ git commit -am "Ausgabetext ändern"
[versuch 0b229dd] Ausgabetext ändern
 1 file changed, 1 insertion(+), 1 deletion(-)
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ sed -i 's/ist zu/hat/' plattenwacht.py
~/plattenwacht$ git switch versuch
error: Your local changes to the following files would be overwritten by checkout:
	plattenwacht.py
Please commit your changes or stash them before you switch branches.
Aborting
```

Git bricht ab und nennt die betroffene Datei. Der saubere Ausweg ist, die Änderung entweder zu committen, mit `git stash` beiseitezulegen oder mit `git restore` zu verwerfen. Ein zweiter Klassiker ist die Arbeit auf dem falschen Branch: Prüfen Sie vor dem Commit mit `git branch` oder dem Statusbefehl, wo Sie stehen. Ein dritter ist das Erzwingen mit `git push --force`, das die Geschichte auf dem Server überschreibt und anderen die Arbeit zerstören kann; ersetzen Sie es nur bewusst und besser durch die vorsichtigere Variante `--force-with-lease`. Nachdem die Änderung verworfen ist, klappt der Wechsel ohne Murren, und den Versuchs-Branch räumen wir mit `-D` (großes D, löscht auch Ungemergtes) weg:

```pycon
~/plattenwacht$ git restore plattenwacht.py
~/plattenwacht$ git switch versuch
Switched to branch 'versuch'
~/plattenwacht$ git switch main
Switched to branch 'main'
~/plattenwacht$ git branch -D versuch
Deleted branch versuch (was 0b229dd).
```

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Beschreibung von `git switch` (Optionen `-c`, `-C`, `--detach` und die Kurzform `-` für den zuletzt benutzten Branch) und die Regeln für `.gitignore` (Muster, Vorrangfolge, Wirkung nur auf unversionierte Dateien, `git rm --cached`) stammen aus den abgerufenen Seiten der Git-Dokumentation. Alle Mitschnitte wurden mit Git 2.43.0 ausgeführt. Dass das Reflog nach einiger Zeit aufräumt, dass `git filter-repo` zum Bereinigen der Geschichte dient und dass die Pickaxe-Suche mit `-S` nach Änderungen einer Zeichenfolge sucht, ist Fachwissen; die Suche haben wir ausgeführt, die Bereinigung nicht.

> **Merke**
>
> Ein Branch ist ein beweglicher Zeiger auf einen Commit; `git switch -c name` legt ihn an, `git switch name` wechselt, `git merge name` führt zusammen (Fast-Forward oder Merge-Commit), `git branch -d` räumt auf. Konflikte löst ein Mensch: Markierungen entfernen, prüfen, `git add`, `git commit`; `git merge --abort` bricht ab. Rückgängig: `git restore` (nicht Committetes verwerfen), `git revert` (Commit durch Gegen-Commit aufheben), `git reset --hard` (gefährlich, Rettung über `git reflog`). `.gitignore` wirkt nur auf unversionierte Dateien; `git rm --cached` nimmt Versioniertes heraus. Geheimnisse nie committen; was committet wurde, gilt als kompromittiert.

> **Praxisbeispiel**
>
> In Teams läuft die Arbeit fast immer nach dem Muster „ein Branch pro Aufgabe“: Man zweigt von `main` ab, arbeitet in kleinen Commits, stellt den Branch auf den Server und bittet per *Pull Request* (bei GitLab *Merge Request*) um Durchsicht, bevor er in `main` einfließt. Viele Teams schützen `main` so, dass nur über diesen Weg etwas hineinkommt. Bei Skripten für die Serveradministration bewährt sich dieselbe Disziplin: Ein Experiment mit anderen Schwellen oder einem neuen Befehl landet auf einem Branch, und nur das Geprüfte kommt auf `main`, von wo die Server ihre Skripte beziehen. Eine `.gitignore` gehört in jedes neue Projekt, noch vor dem ersten Commit.

## 13.3 Projektstruktur

Eine einzelne Datei wie unser `plattenwacht.py` ist ein Skript. Das ist völlig in Ordnung, solange es klein bleibt und nur auf einem Rechner läuft. Wächst es, geschehen drei Dinge: Die Datei wird unübersichtlich, weil Logik, Kommandozeile und Konstanten durcheinanderliegen; man möchte Teile in anderen Programmen wiederverwenden; und man möchte das Programm auf anderen Rechnern installieren, ohne Dateien von Hand zu kopieren. Alle drei Probleme löst eine *Projektstruktur*: eine feste, allgemein bekannte Anordnung von Verzeichnissen und Dateien, die Python-Werkzeuge verstehen und die auch fremde Entwickler sofort lesen können.

Zunächst die Begriffe, mit denen man die Sache beschreibt. Ein *Modul* ist eine einzelne Python-Datei, die man mit `import` nutzen kann (Modul 7 hat das ausführlich behandelt). Ein *Paket* (englisch *package*) ist ein Verzeichnis, das mehrere Module zusammenfasst und eine Datei `__init__.py` enthält; über den Punkt wird auf Teile zugegriffen, etwa `plattenwacht.core`. Eine *Distribution* schließlich ist das, was man installiert und weitergibt: ein Paket samt Beschreibung (Name, Version, Abhängigkeiten), so wie es von `pip` aus einem Index oder aus einem lokalen Verzeichnis eingespielt wird. Im Alltag sagt man „Paket“ zu allen dreien; für die Fehlersuche lohnt es, sie zu unterscheiden.

**Das src-Layout.** Es gibt zwei übliche Anordnungen. Beim *flachen Layout* (flat layout) liegt das Paketverzeichnis direkt im Projektverzeichnis neben `README.md` und der Konfiguration. Beim *src-Layout* liegt es in einem Unterverzeichnis `src`. Der Python Packaging User Guide (abgerufen) nennt als Vorteil des src-Layouts, dass es verhindert, versehentlich den *ungebauten* Quelltext statt der installierten Fassung zu importieren: Beim flachen Layout liegt das Paket im Arbeitsverzeichnis, und Python findet es dort auch ohne Installation, sodass Tests grün werden können, obwohl die installierte Fassung kaputt wäre (zum Beispiel weil eine Datei in der Paketbeschreibung fehlt). Beim src-Layout muss man das Paket installieren, bevor man es nutzen kann; dadurch testet man genau das, was später ausgeliefert wird. Wir verwenden das src-Layout, weil es für Anwendungen und Skriptsammlungen heute der gängige Rat ist. Das Gerüst unseres Projekts sieht so aus:

```text
plattenwacht/
|-- .gitignore
|-- README.md
|-- pyproject.toml          (kommt in 13.4)
|-- src/
|   `-- plattenwacht/
|       |-- __init__.py
|       |-- __main__.py
|       |-- cli.py
|       `-- core.py
`-- tests/                  (Modul 14)
```

Bevor wir umbauen, überlegen wir, wie die Aufgaben verteilt werden. In `core.py` liegt die reine Logik: die Schwellenwerte, das Messen der Belegung und die Bewertung. Diese Funktionen kennen weder die Kommandozeile noch `print`; sie bekommen Werte und liefern Werte. Das macht sie später leicht testbar (Modul 14) und in anderen Programmen wiederverwendbar. In `cli.py` liegt alles, was mit dem Benutzer zu tun hat: das Auswerten der Optionen mit `argparse`, die Ausgabe und der Exit-Code. Die Datei `__init__.py` markiert das Verzeichnis als Paket und legt die Versionsnummer ab. Die Datei `__main__.py` macht das Paket mit `python -m plattenwacht` ausführbar, wie wir gleich sehen. Diese Trennung nach Verantwortung ist der eigentliche Gewinn der Struktur; die Verzeichnisse sind nur ihr Ausdruck.

**Umbau mit Git.** Wir verschieben die Datei mit `git mv`, damit Git die Verschiebung als solche erkennt und die Geschichte der Datei erhalten bleibt, und legen dann die neuen Dateien an. Zuerst das Verzeichnis und die Logik:

```pycon
~/plattenwacht$ mkdir -p src/plattenwacht tests
~/plattenwacht$ git mv plattenwacht.py src/plattenwacht/core.py
~/plattenwacht$ git status --short
R  plattenwacht.py -> src/plattenwacht/core.py
```

Git meldet eine Umbenennung (`R`, *renamed*). Weil wir die Datei gleich inhaltlich ändern, ist es sinnvoll, die Verschiebung und die Änderung getrennt zu committen; so bleibt die Zuordnung eindeutig. Wir committen die reine Verschiebung zuerst.

```pycon
~/plattenwacht$ git commit -m "Skript in src-Layout verschieben (noch unveraendert)"
[main 9f72b62] Skript in src-Layout verschieben (noch unveraendert)
 1 file changed, 0 insertions(+), 0 deletions(-)
 rename plattenwacht.py => src/plattenwacht/core.py (100%)
~/plattenwacht$ git log --oneline --follow -- src/plattenwacht/core.py
9f72b62 Skript in src-Layout verschieben (noch unveraendert)
32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
2a047c9 Kritische Schwelle auf 50 senken (Experiment)
7bd0f26 Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)
97a458a Warnschwelle auf 75 Prozent senken (Richtlinie)
f3b5ef0 Schwellen per Kommandozeile einstellbar machen
3581387 Pfad in der Ausgabe in Anführungszeichen setzen
36f2196 Erste Version: Belegung prüfen und bewerten
```

Die Option `--follow` lässt `git log` der Datei über die Umbenennung hinweg folgen; man sieht also weiter die früheren Commits. Jetzt ersetzen wir den Inhalt von `core.py` durch die reine Logik. Neu sind die Datei mit den Konstanten und der Funktion `pruefe`, die Messung und Bewertung zusammenfasst und ein Ergebnis zurückgibt, ohne etwas auszugeben.

**Datei** `src/plattenwacht/core.py`**:**

```python
"""Kernlogik von plattenwacht: messen und bewerten, ohne Ein- und Ausgabe."""
import shutil

SCHWELLE_WARNUNG = 80
SCHWELLE_KRITISCH = 90


def belegung_prozent(pfad):
    """Gibt zurueck, zu wie viel Prozent das Dateisystem von pfad belegt ist."""
    nutzung = shutil.disk_usage(pfad)
    return nutzung.used / nutzung.total * 100


def bewerte(prozent, warnung=SCHWELLE_WARNUNG, kritisch=SCHWELLE_KRITISCH):
    """Ordnet einen Prozentwert einer Stufe zu: OK, WARNUNG oder KRITISCH."""
    if prozent >= kritisch:
        return "KRITISCH"
    if prozent >= warnung:
        return "WARNUNG"
    return "OK"


def pruefe(pfad, warnung=SCHWELLE_WARNUNG, kritisch=SCHWELLE_KRITISCH):
    """Misst und bewertet in einem Schritt; gibt (stufe, prozent) zurueck."""
    prozent = belegung_prozent(pfad)
    return bewerte(prozent, warnung, kritisch), prozent
```

Jede Funktion trägt jetzt einen *Docstring*, eine Zeichenkette als erste Anweisung, die Werkzeuge wie `help()` und Editoren anzeigen. Das Modul selbst importiert nur `shutil` aus der Standardbibliothek und hat keine Nebenwirkungen beim Import, was eine wichtige Eigenschaft eines Bibliotheksmoduls ist. Die Kommandozeile übernimmt nun `cli.py`.

**Datei** `src/plattenwacht/cli.py`**:**

```python
"""Kommandozeile von plattenwacht."""
import argparse

from . import __version__
from .core import SCHWELLE_KRITISCH, SCHWELLE_WARNUNG, pruefe

EXIT_CODES = {"OK": 0, "WARNUNG": 1, "KRITISCH": 2}


def baue_parser():
    parser = argparse.ArgumentParser(
        prog="plattenwacht",
        description="Meldet, wenn ein Dateisystem fast voll ist.",
    )
    parser.add_argument("pfad", nargs="?", default="/", help="zu pruefender Pfad (Standard: /)")
    parser.add_argument("--warnung", type=float, default=SCHWELLE_WARNUNG, help="Schwelle fuer WARNUNG in Prozent")
    parser.add_argument("--kritisch", type=float, default=SCHWELLE_KRITISCH, help="Schwelle fuer KRITISCH in Prozent")
    parser.add_argument("--version", action="version", version=f"%(prog)s {__version__}")
    return parser


def main(argv=None):
    args = baue_parser().parse_args(argv)
    stufe, prozent = pruefe(args.pfad, args.warnung, args.kritisch)
    print(f"{stufe}: '{args.pfad}' ist zu {prozent:.1f} % belegt")
    return EXIT_CODES[stufe]
```

Drei Dinge sind neu. Erstens die Zeile `from . import __version__`: Der führende Punkt bedeutet „aus dem Paket, in dem ich selbst liege“; man nennt das einen *relativen Import*. Zweitens trennt `baue_parser()` den Aufbau des Parsers vom Ablauf, was später Tests erleichtert. Drittens nimmt `main(argv=None)` die Argumentliste als Parameter: Bei `None` liest `argparse` `sys.argv`, aber ein Test kann eine Liste übergeben, ohne das Programm als Prozess zu starten. Die Hilfsdateien folgen. Beachten Sie, dass `cli.py` *kein* `sys.exit` aufruft; das erledigt der Einstiegspunkt.

**Datei** `src/plattenwacht/__init__.py`**:**

```python
"""plattenwacht - meldet, wenn ein Dateisystem fast voll ist."""

__version__ = "0.1.0"
```

**Datei** `src/plattenwacht/__main__.py`**:**

```python
"""Ermoeglicht den Aufruf mit: python -m plattenwacht"""
import sys

from .cli import main

sys.exit(main())
```

Die Datei `__main__.py` wird ausgeführt, wenn jemand `python -m plattenwacht` eingibt. Sie reicht den Rückgabewert von `main()` als Exit-Code an das Betriebssystem weiter, genau wie wir es in Modul 5 und 6 für Skripte kennengelernt haben. Jetzt probieren wir das Ergebnis aus. Zuerst das, was *nicht* geht, denn daran zeigt sich der Sinn des src-Layouts: Im Projektverzeichnis ist das Paket nicht sichtbar, weil Python nur das aktuelle Verzeichnis und installierte Pakete durchsucht, nicht `src`.

```pycon
~/plattenwacht$ python3 -c "import plattenwacht"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    import plattenwacht
ModuleNotFoundError: No module named 'plattenwacht'
```

Die Meldung `ModuleNotFoundError` sagt, dass Python das Paket nicht findet. Das ist beabsichtigt: Das Paket gilt erst als verfügbar, wenn es installiert ist (13.4). Zum Ausprobieren kann man Python den Suchpfad vorübergehend mit der Umgebungsvariablen `PYTHONPATH` erweitern, die für genau diesen einen Aufruf gilt:

```pycon
~/plattenwacht$ PYTHONPATH=src python3 -m plattenwacht --version
plattenwacht 0.1.0
~/plattenwacht$ PYTHONPATH=src python3 -m plattenwacht . --warnung 0 --kritisch 100; echo "Exit-Code: $?"
WARNUNG: '.' ist zu 4.8 % belegt
Exit-Code: 1
```

Beide Aufrufe laufen: Die Version kommt aus `__init__.py`, und die Prüfung liefert den Exit-Code 1 (Warnung), weil wir die Schwellen so gesetzt haben. Der Prozentwert in Ihrer Ausgabe wird ein anderer sein. Wir sichern den Stand mit einem Commit.

```pycon
~/plattenwacht$ git add src
~/plattenwacht$ git commit -m "Paket plattenwacht mit core, cli und __main__ aufbauen"
[main af0e5ff] Paket plattenwacht mit core, cli und __main__ aufbauen
 4 files changed, 42 insertions(+), 17 deletions(-)
 create mode 100644 src/plattenwacht/__init__.py
 create mode 100644 src/plattenwacht/__main__.py
 create mode 100644 src/plattenwacht/cli.py
~/plattenwacht$ git status --short
~/plattenwacht$ git ls-files
.gitignore
README.md
src/plattenwacht/__init__.py
src/plattenwacht/__main__.py
src/plattenwacht/cli.py
src/plattenwacht/core.py
```

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Umstieg vom Skript zum Paket ist, eine Datei *direkt* auszuführen, die relative Importe enthält. Wir stellen es nach:

```pycon
~/plattenwacht$ PYTHONPATH=src python3 src/plattenwacht/cli.py --help
Traceback (most recent call last):
  File "/home/max/plattenwacht/src/plattenwacht/cli.py", line 4, in <module>
    from . import __version__
ImportError: attempted relative import with no known parent package
```

Die Fehlermeldung „attempted relative import with no known parent package“ sagt, dass Python die Datei als eigenständiges Skript gestartet hat und deshalb nicht weiß, zu welchem Paket sie gehört; der Punkt in `from . import ...` hat dann keinen Bezugspunkt. Die Regel lautet: Ein Paket startet man als Paket, also mit `python -m plattenwacht` oder, nach der Installation, über das Kommando aus 13.4, niemals über den Dateipfad eines Moduls im Paket. Ein zweiter Fehler ist, ein eigenes Modul so zu nennen wie ein Modul der Standardbibliothek: Eine Datei `json.py` oder `logging.py` im Projekt überdeckt das gleichnamige Standardmodul, und plötzlich schlägt ein `import json` in ganz anderen Bibliotheken mit rätselhaften Fehlern fehl. Wählen Sie also Namen, die es in der Standardbibliothek nicht gibt. Wir zeigen das gefahrlos in einem Wegwerfverzeichnis:

```pycon
~/plattenwacht$ mkdir -p /tmp/schatten && cd /tmp/schatten
/tmp/schatten$ printf 'def loads(s):\n    return "ich bin nicht das echte json"\n' > json.py
/tmp/schatten$ python3 -c "import json; print(json.dumps({'a': 1}))"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    import json; print(json.dumps({'a': 1}))
                       ^^^^^^^^^^
AttributeError: module 'json' has no attribute 'dumps' (consider renaming '/tmp/schatten/json.py' since it has the same name as the standard library module named 'json' and prevents importing that standard library module)
/tmp/schatten$ cd ~/plattenwacht
```

Python durchsucht zuerst das Verzeichnis des Skripts beziehungsweise das aktuelle Verzeichnis, und dort liegt unsere Datei, die das echte `json` verdeckt; der Aufruf von `json.dumps` scheitert, weil es die Funktion in unserer Version nicht gibt. Der dritte Klassiker sind *zirkuläre Importe*: Importiert Modul A das Modul B und B wiederum A, hängt das Ergebnis davon ab, was beim Import schon definiert ist, und endet oft in einem `ImportError`. Das Heilmittel ist fast immer eine bessere Aufteilung: Gemeinsam genutzte Dinge kommen in ein drittes Modul, von dem beide abhängen, so wie bei uns `core.py` von `cli.py` genutzt wird, aber nicht umgekehrt. Die Abhängigkeitsrichtung ist hier eindeutig, von der Oberfläche zur Logik.

**Was sonst noch ins Verzeichnis gehört.** Neben dem Paket gehört eine Reihe von Dateien ins Hauptverzeichnis: die `README.md` mit Zweck und Bedienung (13.4), die `.gitignore` (13.2), die `pyproject.toml` mit den Projektdaten (13.4), ein Verzeichnis `tests` (Modul 14) und bei Bedarf eine `LICENSE`. Skripte, die nicht zum Paket gehören, etwa ein Aufruf für `cron`, legt man häufig in ein Verzeichnis `scripts` oder `bin`. Konfigurationsbeispiele und Dokumentation haben ihre eigenen Verzeichnisse (`examples`, `docs`). Man muss das nicht von Anfang an anlegen; wichtig ist, dass jeder Dateityp einen festen Platz hat.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Gegenüberstellung von flachem und src-Layout und das Argument, das src-Layout verhindere ein versehentliches Importieren des ungebauten Quelltexts, stammen aus der abgerufenen Seite „src layout vs flat layout“ des Python Packaging User Guide. Der Guide betont dort auch, dass es sich um eine Empfehlung handelt und beide Anordnungen üblich sind. Der Ablauf des Umbaus und alle Fehlermeldungen in diesem Abschnitt stammen aus echten Läufen mit Python 3.13.18. Die Begriffe Modul, Paket und Distribution werden hier vereinfacht verwendet; die Packaging-Dokumentation grenzt sie genauer ab.

> **Merke**
>
> Ein Skript wird zum Projekt, wenn Logik und Oberfläche getrennt sind und alles seinen festen Platz hat. Im src-Layout liegt das Paket unter `src/`; dadurch ist es nur nach der Installation importierbar, und man testet, was man ausliefert. Relative Importe (`from .core import ...`) funktionieren nur, wenn man das Paket als Paket startet (`python -m paket`). Eigene Modulnamen dürfen keine Standardmodule überdecken. `git mv` erhält die Geschichte einer verschobenen Datei, und Verschieben und Ändern committet man getrennt.

> **Praxisbeispiel**
>
> Fast jedes Python-Projekt, das Sie bei der Arbeit finden, ist so oder ähnlich aufgebaut, und das Wissen um die Struktur hilft beim Einarbeiten in fremden Code: Der Einstieg liegt in `cli.py` oder `__main__.py`, die Logik im Rest. Für Administrationsskripte lohnt der Umbau, sobald ein Skript von mehreren Rechnern genutzt, von `cron` oder einem systemd-Timer (Modul 9) gestartet oder von einem Kollegen weiterentwickelt wird. Dann ist eine installierbare Distribution mit eigenem Kommando deutlich pflegeleichter als eine Datei, die per `scp` verteilt wird.

## 13.4 pyproject.toml und README

Unser Paket liegt jetzt ordentlich im Verzeichnis, aber Python weiß davon nur, wenn wir mit `PYTHONPATH` nachhelfen. Damit aus dem Verzeichnis eine installierbare Distribution wird, braucht es eine Beschreibung: Wie heißt das Projekt, welche Version hat es, was braucht es zum Laufen, und welches Kommando soll bei der Installation entstehen? Diese Angaben stehen seit einigen Jahren einheitlich in einer einzigen Datei namens `pyproject.toml` im Hauptverzeichnis. Früher verteilte man sie auf `setup.py`, `setup.cfg` und `requirements.txt`, und je nach Werkzeug galten unterschiedliche Regeln; heute gibt es dafür Standards, sogenannte PEPs (*Python Enhancement Proposals*): PEP 517 und 518 legen fest, wie ein Projekt gebaut wird, PEP 621 legt fest, wo die Projektdaten stehen.

**Das Dateiformat TOML.** Die Endung `.toml` steht für „Tom’s Obvious Minimal Language“, ein Format für Konfigurationsdateien, das Sie aus Modul 7 neben JSON, YAML und CSV kennen. Eine TOML-Datei besteht aus *Tabellen*, die in eckigen Klammern stehen (`[project]`), und darunter aus Zuweisungen der Form `schluessel = wert`. Texte stehen in doppelten Anführungszeichen, Listen in eckigen Klammern, Kommentare beginnen mit `#`. Eine Tabelle mit Unterstruktur wird durch einen Punkt im Namen ausgedrückt, etwa `[project.scripts]`. Wer Windows-INI-Dateien kennt, findet sich schnell zurecht; wichtig ist nur, dass Texte immer in Anführungszeichen stehen müssen, anders als bei INI.

**Die drei Abschnitte.** Die Datei `pyproject.toml` hat nach der Spezifikation des Python Packaging User Guide (abgerufen) bis zu drei Hauptabschnitte, von denen wir alle verwenden. Der Abschnitt `[build-system]` nennt, womit das Projekt gebaut wird: unter `requires` die Pakete, die der Bau braucht, und unter `build-backend` das Modul, das den Bau übernimmt. Wir nehmen *setuptools*, das am weitesten verbreitete Bauwerkzeug. Der Abschnitt `[project]` enthält die Projektdaten. Pflichtfelder sind der Name und, falls nicht als *dynamisch* gekennzeichnet, die Version. Der Abschnitt `[tool.xyz]` schließlich nimmt die Einstellungen einzelner Werkzeuge auf; in Modul 14 werden wir dort `pytest`, `ruff` und `mypy` einstellen. Im Folgenden die Datei für unser Projekt; die Erklärung der einzelnen Zeilen folgt darunter.

**Datei** `pyproject.toml`**:**

```text
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "plattenwacht"
dynamic = ["version"]
description = "Meldet, wenn ein Dateisystem fast voll ist."
readme = "README.md"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8", "ruff", "mypy"]

[project.scripts]
plattenwacht = "plattenwacht.cli:main"

[tool.setuptools.dynamic]
version = {attr = "plattenwacht.__version__"}
```

Die Zeile `requires = ["setuptools>=77"]` sagt, dass zum Bauen setuptools in mindestens Version 77 nötig ist; das Werkzeug `pip` richtet dafür selbstständig eine kurzlebige Bauumgebung ein. Der Wert `build-backend` nennt den Einstiegspunkt, den `pip` aufruft. Unter `[project]` ist `name` der Name, unter dem das Paket installiert wird. Er darf Bindestriche enthalten, während der Importname (das Verzeichnis unter `src`) keine enthalten darf; bei uns sind beide gleich. Mit `dynamic = ["version"]` sagen wir, dass die Version nicht hier steht, sondern ermittelt wird. Die Angabe unter `[tool.setuptools.dynamic]` holt sie aus dem Attribut `__version__` des Pakets. So steht die Versionsnummer an genau einer Stelle, der Datei `__init__.py`; die Alternative wäre eine feste Zeile `version = "0.1.0"` in `[project]`, die aber leicht von der Konstante im Quelltext abweicht, wenn jemand nur eine von beiden ändert. Das Feld `description` ist ein Satz für Verzeichnisse und Listen, `readme` verweist auf die ausführliche Beschreibung, und `requires-python = ">=3.11"` verhindert, dass das Paket auf einem zu alten Python installiert wird. Mit `dependencies = []` erklären wir, dass wir nur die Standardbibliothek brauchen; sobald das Projekt zum Beispiel `requests` nutzt, steht hier `["requests>=2.31"]`.

Der Abschnitt `[project.optional-dependencies]` bildet *Gruppen* von Zusatzpaketen. Die Gruppe `dev` sammelt die Entwicklungswerkzeuge, die für den Betrieb nicht nötig sind; man installiert sie nur bei Bedarf mit `pip install -e ".[dev]"`. Der Abschnitt `[project.scripts]` schließlich erzeugt bei der Installation ein ausführbares Kommando: Links steht der Name des Kommandos, rechts nach dem Muster `modul:funktion` die aufzurufende Funktion. Genau dafür haben wir `main` so geschrieben, dass sie einen Exit-Code zurückgibt; der erzeugte Startcode ruft `sys.exit(main())` für uns auf.

Die Datei `README.md` ist das Gesicht des Projekts. Sie ist die erste Datei, die jemand liest, und wird auf GitHub und GitLab automatisch angezeigt. Außerdem wird sie über das Feld `readme` als lange Beschreibung in die Distribution übernommen. Eine gute README beantwortet in dieser Reihenfolge: Was macht das Programm? Wie installiere ich es? Wie benutze ich es (mit einem Beispiel)? Was bedeuten die Exit-Codes? Wie entwickle ich daran weiter? Das Format Markdown (Endung `.md`) ist reiner Text mit einfachen Auszeichnungen: `#` für Überschriften, Einrückung oder dreifache Backticks für Code.

**Datei** `README.md`**:**

```text
# plattenwacht

Meldet, wenn ein Dateisystem fast voll ist. Gedacht für Cron-Jobs und
Monitoring-Skripte: Das Ergebnis steht im Exit-Code.

## Installation

    python3 -m venv .venv
    . .venv/bin/activate
    pip install .

## Benutzung

    plattenwacht                       # prueft "/"
    plattenwacht /var --warnung 70     # prueft /var, Warnung ab 70 Prozent

## Exit-Codes

| Code | Bedeutung                          |
|------|------------------------------------|
| 0    | OK                                 |
| 1    | WARNUNG (Schwelle erreicht)        |
| 2    | KRITISCH (kritische Schwelle)      |

## Entwicklung

    pip install -e ".[dev]"
```

Die README nutzt eingerückte Blöcke für Befehle, eine Tabelle für die Exit-Codes und nennt am Ende, wie man als Entwickler einsteigt. Jetzt installieren wir das Projekt. Wir legen dazu eine *virtuelle Umgebung* an (Modul 7), damit die Installation das System-Python nicht verändert, und installieren mit der Option `-e` im *editierbaren Modus*. Das bedeutet: Statt den Quelltext zu kopieren, richtet `pip` nur einen Verweis auf unser Verzeichnis ein. Änderungen am Code wirken deshalb sofort, ohne Neuinstallation. Das ist der übliche Modus während der Entwicklung.

```pycon
~/plattenwacht$ python3 -m venv .venv
~/plattenwacht$ . .venv/bin/activate
(.venv) ~/plattenwacht$ pip install -q --disable-pip-version-check -e .
(.venv) ~/plattenwacht$ pip show plattenwacht | grep -E '^(Name|Version|Summary)'
Name: plattenwacht
Version: 0.1.0
Summary: Meldet, wenn ein Dateisystem fast voll ist.
```

Die Ausgabe von `pip show` bestätigt Name, Version und Beschreibung; die Version `0.1.0` stammt dabei aus der Konstante in `__init__.py`. Da die Umgebung aktiviert ist, steht jetzt das Kommando `plattenwacht` zur Verfügung, ohne `PYTHONPATH` und ohne `python -m`. Auch der Import funktioniert nun von überall.

```pycon
(.venv) ~/plattenwacht$ plattenwacht --version
plattenwacht 0.1.0
(.venv) ~/plattenwacht$ plattenwacht . --warnung 0 --kritisch 100; echo "Exit-Code: $?"
WARNUNG: '.' ist zu 4.8 % belegt
Exit-Code: 1
(.venv) ~/plattenwacht$ cd /tmp && python -c "import plattenwacht; print(plattenwacht.__file__)"; cd ~/plattenwacht
/home/max/plattenwacht/src/plattenwacht/__init__.py
(.venv) ~/plattenwacht$ which plattenwacht
/home/max/plattenwacht/.venv/bin/plattenwacht
```

Der Befehl `which` zeigt, dass das Kommando in der virtuellen Umgebung liegt (`.venv/bin`). Die Pfadausgabe von `plattenwacht.__file__` verrät, dass Python den Quelltext in unserem Verzeichnis `src` nutzt, weil wir editierbar installiert haben. Wir vergleichen das mit einer Distribution, die man weitergibt. Das Werkzeug `build` erzeugt aus dem Projekt eine *Wheel*-Datei (`.whl`), das fertige Installationspaket; eine Wheel ist technisch ein ZIP-Archiv. Wir listen seinen Inhalt auf.

```pycon
(.venv) ~/plattenwacht$ pip install -q --disable-pip-version-check build
(.venv) ~/plattenwacht$ python -m build --wheel > /dev/null 2>&1
(.venv) ~/plattenwacht$ ls dist
plattenwacht-0.1.0-py3-none-any.whl
(.venv) ~/plattenwacht$ python -m zipfile -l dist/*.whl | awk '{print $1}'
File
plattenwacht/__init__.py
plattenwacht/__main__.py
plattenwacht/cli.py
plattenwacht/core.py
plattenwacht-0.1.0.dist-info/METADATA
plattenwacht-0.1.0.dist-info/WHEEL
plattenwacht-0.1.0.dist-info/entry_points.txt
plattenwacht-0.1.0.dist-info/top_level.txt
plattenwacht-0.1.0.dist-info/RECORD
```

Im Archiv liegen genau die Dateien des Pakets (ohne `tests`, `README.md` im Hauptverzeichnis oder `.gitignore`) sowie ein Verzeichnis `*.dist-info` mit den Metadaten. Dort steht auch die Datei `entry_points.txt`, die aus `[project.scripts]` entstanden ist. Den Dateinamen der Wheel kann man lesen: Name, Version, dann `py3-none-any` für „Python 3, keine Binärerweiterung, jede Plattform“. Eine solche Datei kann man per `scp` auf einen anderen Rechner kopieren und dort mit `pip install plattenwacht-0.1.0-py3-none-any.whl` installieren, ohne dass der Quelltext vorhanden sein muss. Die Ordner `dist`, `build` und `*.egg-info` sind Bauergebnisse; sie stehen deshalb schon in unserer `.gitignore`.

**Typische Fehler und Stolperfallen.** Ein klassischer Fehler ist ein Tippfehler im Einstiegspunkt. Das Kommando wird trotzdem erzeugt, denn `pip` prüft nicht, ob die Funktion existiert. Der Fehler zeigt sich erst beim Aufruf. Wir stellen es nach:

```pycon
(.venv) ~/plattenwacht$ sed -i 's/plattenwacht.cli:main/plattenwacht.cli:mian/' pyproject.toml
(.venv) ~/plattenwacht$ pip install -q --disable-pip-version-check -e .
(.venv) ~/plattenwacht$ plattenwacht --version 2>&1 | tail -1
ImportError: cannot import name 'mian' from 'plattenwacht.cli' (/home/max/plattenwacht/src/plattenwacht/cli.py)
```

Die Installation verlief ohne Meldung, doch der Aufruf endet mit einem `ImportError`: Das Modul `plattenwacht.cli` hat kein Attribut `mian`. Wir korrigieren und installieren erneut. Das ist auch die zweite typische Stolperfalle: Änderungen an `pyproject.toml` (Einstiegspunkte, Abhängigkeiten, Metadaten) werden auch im editierbaren Modus erst nach einer erneuten Installation wirksam; nur der Python-Quelltext wirkt sofort.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/plattenwacht.cli:mian/plattenwacht.cli:main/' pyproject.toml
(.venv) ~/plattenwacht$ pip install -q --disable-pip-version-check -e .
(.venv) ~/plattenwacht$ plattenwacht --version
plattenwacht 0.1.0
```

Eine dritte Falle ist die Umgebung selbst: Wer das Kommando in einer neuen Shell aufruft, ohne die Umgebung zu aktivieren, erhält `command not found`. Auf Servern, wo ein `cron`-Job das Kommando starten soll, ruft man deshalb den vollen Pfad auf (`/opt/plattenwacht/.venv/bin/plattenwacht`) oder nutzt `pipx`, das Anwendungen in eigene Umgebungen installiert und das Kommando global verfügbar macht. Ein vierter Fehler betrifft die Syntax: In TOML müssen Texte in Anführungszeichen stehen; `name = plattenwacht` ohne Anführungszeichen führt zu einer Fehlermeldung beim Bau, die auf die Zeile der `pyproject.toml` verweist. Und schließlich: Wer die Version in zwei Dateien pflegt, gerät irgendwann aus dem Takt, was wir mit der dynamischen Version vermieden haben.

**Versionsnummern.** Die Version besteht üblicherweise aus drei Zahlen, `MAJOR.MINOR.PATCH`, nach dem Schema der *Semantic Versioning* (SemVer). Die dritte Zahl erhöht man bei Fehlerkorrekturen, die zweite bei neuen, abwärtskompatiblen Funktionen, die erste bei Änderungen, die bestehende Aufrufe brechen. Beispielsweise wäre eine Änderung der Exit-Codes für Aufrufer von `plattenwacht` eine „große“ Änderung, ein neuer Schalter nur eine „kleine“. Solange die erste Zahl 0 ist, gilt das Projekt als noch nicht stabil. Diese Regeln stammen aus der Spezifikation auf semver.org, die wir nicht abgerufen haben; die Schreibweise von Versionen in Python-Paketen regelt zusätzlich PEP 440.

**Eine Lizenz festlegen.** Ein öffentliches Projekt ohne Lizenzdatei ist rechtlich gesehen nicht frei nutzbar: Standardmäßig behält der Urheber alle Rechte, und andere dürfen den Code weder kopieren noch ändern. Wer den Code teilen will, legt eine Datei `LICENSE` an und trägt die Lizenz in der `pyproject.toml` ein (bei setuptools ab Version 77 als SPDX-Ausdruck im Feld `license`, zum Beispiel `"MIT"`). Für interne Skripte einer Firma ist keine Lizenz nötig; man schreibt dann in die README, dass das Projekt intern ist. Die Auswahl einer passenden Lizenz ist eine rechtliche Entscheidung, die dieses Skript nicht abnehmen kann (keine Rechtsberatung); Hilfen bieten Auswahlseiten wie choosealicense.com, die wir nicht abgerufen haben.

Zum Abschluss committen wir das Ergebnis und prüfen, dass nur die Quelldateien in der Versionsverwaltung landen, die Bauergebnisse und die Umgebung aber nicht.

```pycon
(.venv) ~/plattenwacht$ git add pyproject.toml README.md
(.venv) ~/plattenwacht$ git commit -q -m "pyproject.toml und README anlegen"
(.venv) ~/plattenwacht$ git status --short
(.venv) ~/plattenwacht$ git ls-files
.gitignore
README.md
pyproject.toml
src/plattenwacht/__init__.py
src/plattenwacht/__main__.py
src/plattenwacht/cli.py
src/plattenwacht/core.py
```

Die Liste der versionierten Dateien enthält keine `.venv`, kein `dist` und keine `egg-info`-Verzeichnisse; die `.gitignore` aus Abschnitt 13.2 hat sich also ausgezahlt.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Die Struktur der `pyproject.toml` mit den Abschnitten `[build-system]`, `[project]` und `[tool]`, die Rolle von `requires` und `build-backend`, die Felder des Abschnitts `[project]` (Name, Version, `dynamic`, `requires-python`, `dependencies`, `optional-dependencies`, `scripts`) stammen aus der abgerufenen „pyproject.toml specification“ des Python Packaging User Guide. Dass setuptools ab Version 77 Lizenzangaben als SPDX-Ausdruck zulässt, folgt PEP 639, das hier nicht abgerufen wurde; gebaut wurde mit setuptools 84.2.1 und der Angabe `requires = ["setuptools>=77"]`. Alle Installations- und Bauvorgänge in diesem Abschnitt wurden tatsächlich ausgeführt (mit Netzzugang für `pip`). Die Hinweise zu SemVer, `pipx` und choosealicense.com sind Fachwissen und nicht abgerufen.

> **Merke**
>
> Die `pyproject.toml` enthält drei Abschnitte: `[build-system]` (womit gebaut wird), `[project]` (Name, Version, Abhängigkeiten, Kommandos) und `[tool.*]` (Einstellungen von Werkzeugen). `[project.scripts]` erzeugt bei der Installation ein Kommando der Form `name = "paket.modul:funktion"`. `pip install -e .` installiert editierbar (Quelltext wirkt sofort), Änderungen an der `pyproject.toml` brauchen eine Neuinstallation. Die Version pflegt man an genau einer Stelle. Eine README beantwortet: Was, Installation, Benutzung, Exit-Codes, Entwicklung. Ohne Lizenz ist öffentlicher Code nicht frei nutzbar.

> **Praxisbeispiel**
>
> Eine installierbare Distribution zahlt sich aus, sobald dasselbe Werkzeug auf mehreren Servern laufen soll. Statt Skripte zu kopieren, baut man eine Wheel (oder lädt das Projekt aus einem internen Paketindex) und installiert sie per `pip` in eine Umgebung unter `/opt`. Das Kommando kann dann von `cron` oder einem systemd-Timer (Modul 9) mit vollem Pfad aufgerufen werden. Auch in Firmen, die keine Pakete veröffentlichen, gilt die `pyproject.toml` inzwischen als Standard; Werkzeuge wie `pytest`, `ruff` und `mypy` lesen ihre Einstellungen genau dort.

## 13.5 Projekt ins Git-Repo und Selbsttest

Das Projekt hat jetzt eine Struktur, eine Beschreibung und eine Geschichte. Was noch fehlt, ist der Schritt, der aus einem privaten Verzeichnis eine gemeinsam nutzbare Sache macht: Es braucht einen Ort, an dem der Stand für andere abrufbar ist, und eine Möglichkeit, bestimmte Stände dauerhaft zu benennen. Beides bietet Git. Bisher haben wir ausschließlich mit einem *lokalen* Repository gearbeitet, das nur auf unserem Rechner liegt. Git ist aber ein *verteiltes* Versionsverwaltungssystem: Jeder Beteiligte hat eine vollständige Kopie der Geschichte, und die Kopien werden über sogenannte *Remotes* abgeglichen. Ein Remote ist ein anderes Repository, mit dem das eigene Daten austauscht, meist auf einem Server oder einer Plattform wie GitHub oder GitLab.

**Tags: Versionen benennen.** Zunächst die Namen für wichtige Stände. Ein *Tag* ist eine feste Marke an einem Commit, anders als ein Branch bewegt er sich nicht. Man verwendet Tags für Veröffentlichungen: `v0.1.0` heißt „der Stand, den wir als Version 0.1.0 ausgeliefert haben“. Es gibt einfache Tags (nur eine Marke) und *annotierte* Tags, die zusätzlich Autor, Datum und eine Nachricht speichern; für Releases nimmt man die annotierten, mit der Option `-a`. Der Tag passt zur Versionsnummer in `__init__.py`, und wer die Nummer erhöht, setzt danach den neuen Tag.

```pycon
(.venv) ~/plattenwacht$ git tag -a v0.1.0 -m "Erste benutzbare Version"
(.venv) ~/plattenwacht$ git tag -n
v0.1.0          Erste benutzbare Version
(.venv) ~/plattenwacht$ git log --oneline --decorate -3
426b88f (HEAD -> main, tag: v0.1.0) pyproject.toml und README anlegen
af0e5ff Paket plattenwacht mit core, cli und __main__ aufbauen
9f72b62 Skript in src-Layout verschieben (noch unveraendert)
```

Die Option `-n` von `git tag` zeigt zu jedem Tag die erste Zeile der Nachricht. In der Ausgabe von `git log --decorate` steht der Tag in Klammern neben dem Commit, auf den er zeigt, zusammen mit `HEAD` und dem Branch. Ein Tag lässt sich jederzeit wieder ansehen, etwa mit `git show v0.1.0` oder indem man ihn auscheckt (`git switch --detach v0.1.0`), und er bleibt bestehen, auch wenn `main` längst weiterwandert.

**Ein Remote einrichten.** Zum Üben brauchen wir keinen Server im Internet. Ein Remote kann auch ein Verzeichnis auf demselben Rechner sein, und für den Austausch von Daten ist es für Git gleich, ob die Gegenstelle über einen Pfad, per SSH oder per HTTPS erreichbar ist. Wir legen ein *Bare-Repository* an, also ein Repository ohne Arbeitsverzeichnis, nur mit der Geschichte. Genau so sind die Repositories auf einem Git-Server aufgebaut: Niemand arbeitet direkt darin, man schiebt Commits hinein und holt sie heraus. Danach tragen wir es als Remote namens `origin` ein. Der Name `origin` ist reine Konvention für „das Repository, von dem ich stamme beziehungsweise zu dem ich sende“.

```pycon
(.venv) ~/plattenwacht$ git init -q --bare -b main ~/server/plattenwacht.git
(.venv) ~/plattenwacht$ git remote add origin ~/server/plattenwacht.git
(.venv) ~/plattenwacht$ git remote -v
origin	/home/max/server/plattenwacht.git (fetch)
origin	/home/max/server/plattenwacht.git (push)
(.venv) ~/plattenwacht$ git push -q -u origin main
(.venv) ~/plattenwacht$ git push -q origin v0.1.0
(.venv) ~/plattenwacht$ git branch -vv
* main 426b88f [origin/main] pyproject.toml und README anlegen
```

Der Befehl `git push -u origin main` überträgt den Branch `main` zum Remote; die Option `-u` (*set upstream*) merkt sich, dass der lokale `main` dem `main` von `origin` entspricht, sodass später ein einfaches `git push` oder `git pull` genügt. Tags werden nicht automatisch mitgeschickt, deshalb schieben wir `v0.1.0` gesondert (alternativ schickt `git push --tags` alle). In der letzten Zeile zeigt `git branch -vv` den Zusammenhang: Hinter dem Branch steht in eckigen Klammern der Branch des Remotes. Auf GitHub oder GitLab legt man das Repository vorher im Browser an und erhält eine Adresse wie `git@github.com:name/plattenwacht.git` (SSH) oder eine `https://`-Adresse; ansonsten sind die Befehle dieselben. Wie die Anmeldung dort funktioniert, besprechen wir weiter unten. Beachten Sie nebenbei, was der Push mit übertragen hat: die *gesamte* Geschichte, auch die beiden Commits mit dem Passwort aus 13.2. Wäre das Passwort echt, läge es jetzt auf dem Server. Genau deshalb muss ein Geheimnis *vor* dem ersten Push aus der Geschichte verschwunden sein, nicht erst danach.

**Als zweite Person arbeiten.** Jetzt spielen wir eine Kollegin, die das Projekt übernimmt. Mit `git clone` holt man eine vollständige Kopie samt Geschichte; Git trägt dabei automatisch `origin` als Remote ein. Die Kollegin ändert die README und schiebt ihren Commit auf den Server.

```pycon
(.venv) ~/plattenwacht$ git clone -q ~/server/plattenwacht.git ~/kollegin/plattenwacht
(.venv) ~/plattenwacht$ cd ~/kollegin/plattenwacht
(.venv) ~/kollegin/plattenwacht$ git config user.name "Kollegin"
(.venv) ~/kollegin/plattenwacht$ git config user.email "kollegin@example.org"
(.venv) ~/kollegin/plattenwacht$ git log --oneline --decorate -2
426b88f (HEAD -> main, tag: v0.1.0, origin/main, origin/HEAD) pyproject.toml und README anlegen
af0e5ff Paket plattenwacht mit core, cli und __main__ aufbauen
(.venv) ~/kollegin/plattenwacht$ echo "" >> README.md
(.venv) ~/kollegin/plattenwacht$ echo "Lizenz: intern, nur fuer den Gebrauch in der Firma." >> README.md
(.venv) ~/kollegin/plattenwacht$ git commit -q -am "README: Lizenzhinweis ergaenzen"
(.venv) ~/kollegin/plattenwacht$ git push -q
(.venv) ~/kollegin/plattenwacht$ cd ~/plattenwacht
```

Das Repository der Kollegin enthält die gesamte Geschichte einschließlich des Tags. Mit `git config user.name` ohne `--global` stellen wir Name und Adresse nur für dieses Repository ein. Zurück in unserem eigenen Verzeichnis arbeiten wir unterdessen an etwas anderem, nämlich an einem Docstring in `core.py`. Wir committen lokal und wollen ebenfalls pushen.

```pycon
(.venv) ~/plattenwacht$ sed -i 's/Kernlogik von plattenwacht: messen und bewerten, ohne Ein- und Ausgabe./Kernlogik: messen und bewerten, ohne Ein- und Ausgabe (kein print, kein exit)./' src/plattenwacht/core.py
(.venv) ~/plattenwacht$ git commit -q -am "core: Modul-Docstring praezisieren"
(.venv) ~/plattenwacht$ git push
To /home/max/server/plattenwacht.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to '/home/max/server/plattenwacht.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

Git lehnt den Push ab: Auf dem Server liegt ein Commit, den wir nicht kennen, und das Hinzufügen unseres Commits würde ihn verdrängen. Die Meldung `rejected ... (fetch first)` oder „non-fast-forward“ ist ein Schutzmechanismus und kein Fehler. Der Weg heraus: erst die fremden Änderungen holen, mit den eigenen zusammenführen und dann pushen. Dafür dient `git pull`, das ein `git fetch` (Commits holen, ohne etwas zu verändern) mit einem Merge (oder Rebase) verbindet. Weil neuere Git-Versionen bei auseinandergelaufenen Branches nachfragen, wie zusammengeführt werden soll, geben wir die Strategie explizit an.

```pycon
(.venv) ~/plattenwacht$ git fetch
From /home/max/server/plattenwacht
   426b88f..8775c1e  main       -> origin/main
(.venv) ~/plattenwacht$ git log --oneline --graph --all --decorate
* 7014ae7 (HEAD -> main) core: Modul-Docstring praezisieren
| * 8775c1e (origin/main) README: Lizenzhinweis ergaenzen
|/  
* 426b88f (tag: v0.1.0) pyproject.toml und README anlegen
* af0e5ff Paket plattenwacht mit core, cli und __main__ aufbauen
* 9f72b62 Skript in src-Layout verschieben (noch unveraendert)
* 54351aa Zugangsdaten wieder entfernen
* 28297a1 Zugangsdaten ergänzen
* b63aa85 Protokolldatei aus dem Repository nehmen
* d383641 Protokoll versehentlich mit eingecheckt
* 87e60c0 .gitignore für Python-Projekte anlegen
* 32fd663 Revert "Kritische Schwelle auf 50 senken (Experiment)"
* 2a047c9 Kritische Schwelle auf 50 senken (Experiment)
*   f82da4b Merge branch 'fix/warnschwelle-85'
|\  
| * 7bd0f26 Warnschwelle auf 85 Prozent anheben (zu viele Fehlalarme)
* |   644d6be Merge branch 'docs/readme'
|\ \  
| * | 1b92d75 README anlegen
| |/  
* / 97a458a Warnschwelle auf 75 Prozent senken (Richtlinie)
|/  
* f3b5ef0 Schwellen per Kommandozeile einstellbar machen
* 3581387 Pfad in der Ausgabe in Anführungszeichen setzen
* 36f2196 Erste Version: Belegung prüfen und bewerten
(.venv) ~/plattenwacht$ git pull -q --no-rebase --no-edit
(.venv) ~/plattenwacht$ git push -q
(.venv) ~/plattenwacht$ git log --oneline --graph -4
*   fc26b73 Merge branch 'main' of /home/max/server/plattenwacht
|\  
| * 8775c1e README: Lizenzhinweis ergaenzen
* | 7014ae7 core: Modul-Docstring praezisieren
|/  
* 426b88f pyproject.toml und README anlegen
(.venv) ~/plattenwacht$ git status -sb
## main...origin/main
```

Mit `git fetch` holen wir die Commits der Kollegin, ohne unseren Arbeitsstand anzufassen. Die Grafik zeigt, dass der Branch von `origin` und unser `main` auseinandergelaufen sind. Das anschließende `git pull` führt sie in einem Merge-Commit zusammen; da die Commits verschiedene Dateien betreffen, gibt es keinen Konflikt. Danach klappt der Push. Die Kurzform `git status -sb` (`-b` zeigt den Branch) bestätigt, dass wir mit dem Remote gleichauf sind. Wer lieber eine geradlinige Geschichte ohne Merge-Commits hat, nutzt `git pull --rebase`, was die eigenen Commits auf die fremden aufsetzt; die Entscheidung zwischen beiden Strategien ist Teamsache. Wichtig ist nur, bei geteilten Branches die Geschichte nicht rückwirkend umzuschreiben.

**Zugang zu GitHub und GitLab.** Auf Plattformen im Netz muss man sich ausweisen. Üblich sind zwei Wege. Beim Weg über *SSH* erzeugt man ein Schlüsselpaar (`ssh-keygen -t ed25519`), hinterlegt den öffentlichen Schlüssel im Profil der Plattform und nutzt Adressen der Form `git@github.com:...`; das kennen Sie aus Modul 11. Der private Schlüssel verlässt den Rechner nie und gehört niemals ins Repository. Beim Weg über *HTTPS* verwendet man einen *Personal Access Token*, den man statt eines Passworts eingibt; Tokens lassen sich auf bestimmte Rechte beschränken und widerrufen. Beide Wege haben wir hier nicht vorgeführt, weil dafür ein Konto bei einer Plattform nötig ist; die Schritte unterscheiden sich im Detail je nach Anbieter und ändern sich gelegentlich, deshalb gilt die jeweilige Dokumentation von GitHub oder GitLab (nicht abgerufen). Als Faustregel schützen Sie den Zugang zusätzlich mit einer zweiten Stufe der Anmeldung und legen einen Token niemals in eine Datei im Repository.

**Typische Fehler und Stolperfallen.** Der meistgemachte Fehler ist der soeben gezeigte abgelehnte Push, und die meistgemachte falsche Antwort darauf lautet `git push --force`, was den Stand des Servers durch den eigenen ersetzt und die Arbeit der Kollegin aus dem Branch löscht. Der sichere Weg ist die bessere Antwort: Wer wirklich erzwingen muss, nimmt `--force-with-lease`, das nur überschreibt, wenn der Server noch so aussieht, wie man ihn zuletzt gesehen hat. Ein zweiter häufiger Fehler ist, Tags zu vergessen: Ohne `git push origin v0.1.0` (oder `--tags`) sehen andere den Tag nicht. Wir prüfen auf der Seite der Kollegin, was dort ankommt.

```pycon
(.venv) ~/plattenwacht$ cd ~/kollegin/plattenwacht
(.venv) ~/kollegin/plattenwacht$ git pull -q
(.venv) ~/kollegin/plattenwacht$ git tag -n
v0.1.0          Erste benutzbare Version
(.venv) ~/kollegin/plattenwacht$ git log --oneline -3
fc26b73 Merge branch 'main' of /home/max/server/plattenwacht
7014ae7 core: Modul-Docstring praezisieren
8775c1e README: Lizenzhinweis ergaenzen
(.venv) ~/kollegin/plattenwacht$ git ls-remote --tags origin
a617b7ede6db40c6273ac90e4b229228c1ca5494	refs/tags/v0.1.0
426b88fa9d5c1b723452014a1207ccfcc0ae4d84	refs/tags/v0.1.0^{}
(.venv) ~/kollegin/plattenwacht$ cd ~/plattenwacht
```

Der Tag ist beim Klon vorhanden, weil wir ihn gepusht hatten. Der Befehl `git ls-remote --tags origin` fragt den Server direkt und zeigt, welche Tags dort liegen. Ein dritter Fehler ist ein falsches oder gar kein Upstream: Meldet `git pull` „There is no tracking information for the current branch“, fehlt die Verknüpfung, die wir mit `-u` hergestellt haben; sie lässt sich mit `git branch --set-upstream-to=origin/main` nachholen. Und schließlich der Dauerbrenner: Geheimnisse im Repository. Wer ein Repository auf eine öffentliche Plattform stellt, sollte vorher mit `git log -p` und einer Suche prüfen, ob etwas auftaucht, das dort nicht hingehört (siehe 13.2).

**Gute Commit-Nachrichten.** Eine Commit-Nachricht beantwortet nicht, *was* geändert wurde (das zeigt `git diff`), sondern *warum*. Üblich ist eine kurze Betreffzeile von höchstens etwa 50 Zeichen im Imperativ oder in der Gegenwart („Schwellen per Kommandozeile einstellbar machen“), gegebenenfalls eine Leerzeile und eine ausführlichere Begründung. Viele Teams verwenden zusätzlich ein Präfix nach den *Conventional Commits* (`feat:`, `fix:`, `docs:`), aus dem sich Versionsnummern und Änderungsprotokolle automatisch ableiten lassen; die Spezifikation haben wir nicht abgerufen. Kleine, thematisch geschlossene Commits erleichtern die Suche nach der Ursache eines Fehlers erheblich, weil man dann einzelne Commits prüfen oder zurücknehmen kann.

**Der Meilenstein: das Projekt im Repository.** Mit den Mitteln dieses Moduls ist das Zwischenziel erreicht. Prüfen Sie Ihr eigenes Projekt anhand dieser Fragen: Hat das Verzeichnis ein src-Layout, und lässt sich das Paket mit `pip install -e .` installieren? Gibt `git status` ein sauberes Verzeichnis aus, und stehen `.venv`, `__pycache__` und Bauordner in der `.gitignore`? Zeigt `git log` eine nachvollziehbare Geschichte in kleinen Schritten? Hat die README einen Abschnitt zur Installation und zur Benutzung? Ist die Version als Tag gesetzt, und liegt das Ganze auf einem Remote? In Modul 14 sichern wir genau diesen Stand mit Tests ab, und in Modul 15 wird daraus das Abschlussprojekt.

```pycon
(.venv) ~/plattenwacht$ git status -sb
## main...origin/main
(.venv) ~/plattenwacht$ git log --oneline --graph --decorate -8
*   fc26b73 (HEAD -> main, origin/main) Merge branch 'main' of /home/max/server/plattenwacht
|\  
| * 8775c1e README: Lizenzhinweis ergaenzen
* | 7014ae7 core: Modul-Docstring praezisieren
|/  
* 426b88f (tag: v0.1.0) pyproject.toml und README anlegen
* af0e5ff Paket plattenwacht mit core, cli und __main__ aufbauen
* 9f72b62 Skript in src-Layout verschieben (noch unveraendert)
* 54351aa Zugangsdaten wieder entfernen
* 28297a1 Zugangsdaten ergänzen
(.venv) ~/plattenwacht$ git ls-files | wc -l
7
```

**Selbsttest.** Beantworten Sie die folgenden Fragen zunächst ohne Nachschlagen; die Antworten folgen im Anschluss. Erstens: Was ist der Unterschied zwischen `git add` und `git commit`? Zweitens: Welchen Zweck hat die Datei `.gitignore`, und warum schützt sie nicht vor einem bereits versionierten Passwort? Drittens: Weshalb liegt das Paket im src-Layout unter `src/`? Viertens: Was bewirkt der Abschnitt `[project.scripts]`? Fünftens: Welche zwei Möglichkeiten gibt es, einen Commit rückgängig zu machen, und wann nimmt man welche? Sechstens: Was bedeutet die Ablehnung eines Pushs mit „non-fast-forward“, und wie reagiert man richtig?

Zu den Antworten: Erstens legt `git add` fest, was in den nächsten Commit kommen soll (Staging-Bereich), während `git commit` diesen Stand dauerhaft mit Nachricht in die Geschichte schreibt. Zweitens listet die `.gitignore` Muster für Dateien, die Git nicht beobachten soll; sie wirkt aber nur auf unversionierte Dateien, und was einmal committet wurde, bleibt in der Geschichte und gilt als kompromittiert. Drittens erzwingt das src-Layout, dass man das Paket installiert, bevor man es nutzt, und verhindert damit, dass Tests den ungebauten Quelltext statt der auslieferbaren Fassung verwenden. Viertens erzeugt der Abschnitt bei der Installation ein Kommando, das die angegebene Funktion aufruft. Fünftens hilft `git restore` für nicht committete Änderungen und `git revert` für schon committete (neuer Gegen-Commit), während `git reset --hard` die Geschichte zurücksetzt und nur für lokale, ungeteilte Stände taugt. Sechstens heißt die Meldung, dass der Server Commits enthält, die man nicht hat; man holt sie mit `git pull`, löst gegebenenfalls Konflikte und pusht dann erneut, statt zu erzwingen.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Das Verhalten der Befehle `git tag`, `git remote`, `git push`, `git clone` und `git pull` wurde hier mit Git 2.43.0 an einem lokalen Bare-Repository ausgeführt und gezeigt. Die Dokumentationsseiten zu `git init` (Option `--bare`, Option `-b`) wurden abgerufen; die Hinweise zu GitHub und GitLab (Zugang per SSH oder Token, Pull- und Merge-Requests), zu Conventional Commits und zu `--force-with-lease` sind Fachwissen und wurden nicht abgerufen. Ein Test gegen eine echte Plattform war in diesem Arbeitsbereich nicht möglich; die Befehle für ein Remote im Netz sind dieselben, nur die Adresse und die Anmeldung unterscheiden sich.

> **Merke**
>
> Ein Remote ist ein anderes Repository, mit dem man Commits austauscht (`git remote add`, `git push`, `git fetch`, `git pull`, `git clone`). `push -u` verknüpft den lokalen Branch mit dem Remote-Branch. Tags (`git tag -a v0.1.0 -m ...`) benennen Versionen fest und werden gesondert gepusht. Ein abgelehnter Push heißt: erst holen und zusammenführen, nicht erzwingen. Zugangsdaten und Schlüssel gehören nie ins Repository.

> **Praxisbeispiel**
>
> Fast alle Teams hosten ihre Repositories auf GitHub, GitLab, Bitbucket oder einem eigenen Server. Für Administrationsskripte hat sich ein einfacher Ablauf bewährt: Entwickeln auf einem Branch, per Pull Request prüfen lassen, in `main` zusammenführen, mit einem Tag versehen und auf den Servern über `pip` oder `git pull` bei einem bestimmten Tag ausrollen. So ist jederzeit klar, welche Fassung wo läuft, und im Störfall lässt sich gezielt auf einen früheren Tag zurückgehen.

## Übungsaufgaben zu Modul 13

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen praktisches Arbeiten mit Git und Python. Arbeiten Sie bei den praktischen Aufgaben jeweils in einem neuen, leeren Verzeichnis (zum Beispiel `uebung5`), damit Ihr Projekt unberührt bleibt. Lösungen mit Erklärung finden Sie im Anschluss. Für die Aufgaben 7 und 8 brauchen Sie Internetzugang, weil `pip` bei der Installation das Bauwerkzeug lädt.

**Aufgabe 1.** Quiz zu den Grundlagen von Git: Welche drei Bereiche unterscheidet Git, und welcher Befehl bewegt Änderungen von einem in den nächsten? Was zeigt `git diff` im Vergleich zu `git diff --staged`? Weshalb kann ein Commit nicht nachträglich unbemerkt verändert werden (was bedeutet die Kennung des Commits)? Was geschieht bei `git add .`, und welche Gefahr birgt der Befehl? Wozu dienen `git status`, `git log --oneline` und `git show`?

**Aufgabe 2.** Quiz zu Branches und Rückgängigmachen: Was ist ein Branch technisch gesehen? Wodurch unterscheidet sich ein Fast-Forward-Merge von einem Merge mit eigenem Merge-Commit? Woran erkennt man eine Konfliktstelle in einer Datei, und welche Schritte führen zum Abschluss des Merges? Welchen Befehl wählen Sie, (a) um eine nicht committete Änderung zu verwerfen, (b) um einen bereits geteilten Commit aufzuheben, (c) um nach einem versehentlichen `git reset --hard` den Stand zurückzuholen?

**Aufgabe 3.** Quiz zur Projektstruktur: Weshalb legt man ein Paket in das Verzeichnis `src`? Was bedeutet der Punkt in `from .core import pruefe`, und warum scheitert der Aufruf `python src/plattenwacht/cli.py`? Welche drei Abschnitte hat eine `pyproject.toml`, und was steht in jedem? Was bewirkt `pip install -e .` im Unterschied zu `pip install .`, und wann muss man nach Änderungen neu installieren? Was ist ein Wheel?

**Aufgabe 4.** Finden Sie mindestens sechs Fehler oder Probleme in der folgenden `pyproject.toml` und der `.gitignore` und erklären Sie jeden. Das Paket liegt im src-Layout unter `src/mein_tool/` und hat die Funktion `main` in der Datei `cli.py`.

```text
# pyproject.toml
[build-system]
requires = setuptools
build-backend = "setuptools.build_meta"

[project]
name = mein tool
version = "1.0"
version_extra = "0.0.1"
requires-python = ">=3"

[project.scripts]
mein-tool = "mein_tool.cli.main"

# .gitignore
*.py
__pycache__
venv
.env
!.env
```

**Aufgabe 5.** **Praxis (Git):** Legen Sie ein neues Repository an. Es soll die Datei `gruss.txt` mit dem Inhalt `Hallo Welt` enthalten, aufgenommen in einem ersten Commit. Zweigen Sie zwei Branches `formell` und `locker` ab; in `formell` wird der Inhalt zu `Guten Tag, Welt`, in `locker` zu `Hi, Welt` (je ein Commit). Führen Sie beide in `main` zusammen, lösen Sie den auftretenden Konflikt, sodass am Ende `Guten Tag, Welt` in der Datei steht, und zeigen Sie die Geschichte als Grafik.

**Aufgabe 6.** **Praxis (.gitignore):** In einem Projekt liegen die Dateien `.env`, `config.example.env`, `notizen.bak`, `.venv/bin/python`, `src/paket/__pycache__/m.cpython-313.pyc`, `daten/export.csv`, `daten/README.md` und `src/paket/m.py`. Schreiben Sie eine `.gitignore`, sodass *nur* `config.example.env`, `daten/README.md` und `src/paket/m.py` versioniert werden, und belegen Sie es mit `git status --short` und `git check-ignore -v`.

**Aufgabe 7.** **Praxis (Paket):** Bauen Sie ein kleines Paket `zeilen` im src-Layout mit der Funktion `zaehle(pfad)`, die für eine Textdatei die Zahl der Zeilen, Wörter und Zeichen als Tupel liefert, und einem Kommando `zaehle-datei`, das die drei Zahlen für eine übergebene Datei ausgibt. Installieren Sie es editierbar in einer virtuellen Umgebung und probieren Sie das Kommando an einer Datei mit drei Zeilen aus.

**Aufgabe 8.** **Praxis (Release):** Gegeben ist ein kleines Paket, dessen Datei `__init__.py` die Version `0.1.0` enthält und das als `v0.1.0` getaggt ist (legen Sie es bei Bedarf an). Erhöhen Sie die Version auf `0.2.0`, committen Sie das, setzen Sie den annotierten Tag `v0.2.0`, schieben Sie Commit und Tag in ein lokales Bare-Repository, klonen Sie von dort in ein zweites Verzeichnis und zeigen Sie mit `git describe` und einem Blick in die Datei, dass der Klon genau die Version `0.2.0` enthält. Prüfen Sie außerdem, dass ein Checkout des Tags `v0.1.0` die alte Version zeigt.

## Lösungen zu Modul 13

**Aufgabe 1.** Git unterscheidet das *Arbeitsverzeichnis* (die Dateien, wie Sie sie bearbeiten), den *Staging-Bereich* (die Vormerkung für den nächsten Commit) und die *Geschichte* (die Commits im Verzeichnis `.git`). `git add` bewegt Änderungen vom Arbeitsverzeichnis in den Staging-Bereich, `git commit` vom Staging-Bereich in die Geschichte. `git diff` zeigt die Unterschiede zwischen Arbeitsverzeichnis und Staging-Bereich (also das, was noch *nicht* vorgemerkt ist), `git diff --staged` die zwischen Staging-Bereich und letztem Commit (also das, was der nächste Commit enthalten würde). Die Kennung eines Commits ist eine Prüfsumme über seinen Inhalt, seine Nachricht, seine Metadaten und die Kennung seines Vorgängers. Ändert man irgendetwas, ändert sich die Kennung, und damit auch die aller Nachfolger; eine nachträgliche Änderung fällt also auf. `git add .` merkt *alles* im aktuellen Verzeichnis vor, was neu oder geändert ist, einschließlich Dateien, die nicht ins Repository gehören (Protokolle, Zugangsdaten, Zwischenspeicher). Deshalb prüft man vor dem Commit mit `git status` und `git diff --staged`, was tatsächlich im Commit landet, und pflegt eine `.gitignore`. Der Status zeigt den Zustand der drei Bereiche, `git log --oneline` die Geschichte in einer Zeile pro Commit, `git show` den Inhalt eines einzelnen Commits.

**Aufgabe 2.** Ein Branch ist ein benannter, beweglicher Zeiger auf einen Commit. Beim Commit auf dem Branch wandert der Zeiger zum neuen Commit mit. Ein Fast-Forward-Merge ist möglich, wenn der Zielbranch seit der Abzweigung unverändert ist: Dann schiebt Git nur den Zeiger nach vorn, und es entsteht kein neuer Commit. Haben beide Seiten neue Commits, legt Git einen eigenen Merge-Commit mit zwei Vorgängern an. Eine Konfliktstelle erkennt man an den Zeilen `<<<<<<<`, `=======` und `>>>>>>>`: Dazwischen stehen die beiden Fassungen. Zum Abschluss bearbeitet man die Datei, sodass nur das Gewünschte bleibt und die Markierungen weg sind, prüft, ob das Programm noch läuft, merkt die Datei mit `git add` vor und schließt mit `git commit` ab; mit `git merge --abort` bricht man ab. Zu den drei Fällen: (a) `git restore datei` verwirft die nicht committete Änderung (endgültig). (b) Für einen Commit, den andere schon haben, nimmt man `git revert`, weil es einen neuen Gegen-Commit erzeugt und die Geschichte nicht verändert. (c) Nach einem versehentlichen `git reset --hard` sucht man im `git reflog` die Kennung des früheren Standes und setzt mit `git reset --hard` auf diese Kennung (oder `HEAD@{n}`) zurück. Das geht nur für Commits; nicht committete Änderungen sind nach einem harten Reset verloren.

**Aufgabe 3.** Im src-Layout ist das Paket nur nach der Installation importierbar; man testet daher genau das, was später ausgeliefert wird, und importiert nicht versehentlich den ungebauten Quelltext aus dem Arbeitsverzeichnis. Der Punkt bedeutet „aus dem Paket, in dem diese Datei liegt“ (relativer Import). Bei `python src/plattenwacht/cli.py` startet Python die Datei als eigenständiges Skript ohne Paketzugehörigkeit; der Punkt hat dann keinen Bezugspunkt, und es erscheint „attempted relative import with no known parent package“. Man startet Pakete mit `python -m paket` oder über das installierte Kommando. Die `pyproject.toml` hat `[build-system]` (Bauwerkzeuge und Backend), `[project]` (Name, Version, Beschreibung, Python-Version, Abhängigkeiten, Kommandos über `[project.scripts]`) und `[tool.*]` (Einstellungen von Werkzeugen). `pip install -e .` richtet einen Verweis auf das Verzeichnis ein, sodass Änderungen am Quelltext sofort gelten; `pip install .` kopiert den Stand zum Zeitpunkt der Installation in die Umgebung. Neu installieren muss man bei Änderungen an der `pyproject.toml` (neue Einstiegspunkte, Abhängigkeiten, Version), nicht bei Änderungen am Python-Code. Ein Wheel (`.whl`) ist das fertig gebaute Installationspaket, technisch ein ZIP-Archiv, das `pip` ohne Bauschritt einspielen kann.

**Aufgabe 4.** In der `pyproject.toml` gibt es folgende Probleme: (1) `requires = setuptools` ist keine gültige Liste; richtig ist `requires = ["setuptools>=77"]`, mit Anführungszeichen und eckigen Klammern. (2) `name = mein tool` hat weder Anführungszeichen noch ist ein Leerzeichen im Namen zulässig; richtig ist `name = "mein-tool"`. (3) `version = "1.0"` ist formal zulässig, aber unüblich; nach SemVer schreibt man dreiteilig `"1.0.0"`. (4) `version_extra` ist kein Feld der Spezifikation und führt zu einer Fehlermeldung beim Bau; in `[project]` sind nur die festgelegten Felder erlaubt. (5) `requires-python = ">=3"` ist so lax, dass das Paket auch auf Python 3.0 installierbar wäre, das gar nicht lauffähig ist; sinnvoll ist eine konkrete Untergrenze wie `">=3.11"`. (6) Der Einstiegspunkt `"mein_tool.cli.main"` verwendet einen Punkt statt des Doppelpunkts vor dem Funktionsnamen; richtig ist `"mein_tool.cli:main"`. In der `.gitignore`: (7) `*.py` würde sämtlichen Quelltext ignorieren; gemeint war `*.pyc`. (8) `__pycache__` ohne abschließenden Schrägstrich passt zwar, aber auch auf eine Datei dieses Namens; mit `__pycache__/` ist der Zweck eindeutig. (9) `venv` ignoriert nur einen Ordner dieses Namens, während der Standardname `.venv` lautet. (10) `.env` gefolgt von `!.env` hebt das Muster wieder auf, sodass Zugangsdaten im Repository landen; die Ausnahme muss weg (oder auf eine bewusste Beispieldatei beschränkt werden).

**Aufgabe 5.** Wir legen das Repository an, committen die Grundfassung und zweigen die beiden Branches ab. Da beide dieselbe Zeile ändern, wird der erste Merge ein Fast-Forward, der zweite scheitert mit einem Konflikt.

```pycon
(.venv) ~/plattenwacht$ mkdir -p ~/uebung5 && cd ~/uebung5
(.venv) ~/uebung5$ git init -q -b main
(.venv) ~/uebung5$ echo "Hallo Welt" > gruss.txt
(.venv) ~/uebung5$ git add gruss.txt
(.venv) ~/uebung5$ git commit -q -m "Grundfassung"
(.venv) ~/uebung5$ git switch -q -c formell
(.venv) ~/uebung5$ echo "Guten Tag, Welt" > gruss.txt
(.venv) ~/uebung5$ git commit -q -am "Formelle Anrede"
(.venv) ~/uebung5$ git switch -q main
(.venv) ~/uebung5$ git switch -q -c locker
(.venv) ~/uebung5$ echo "Hi, Welt" > gruss.txt
(.venv) ~/uebung5$ git commit -q -am "Lockere Anrede"
(.venv) ~/uebung5$ git switch -q main
(.venv) ~/uebung5$ git merge formell
Updating 1764da3..b166833
Fast-forward
 gruss.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Der erste Merge war ein Fast-Forward, weil `main` seit dem Abzweigen nicht verändert wurde. Beim zweiten Merge haben beide Seiten dieselbe Zeile unterschiedlich geändert, daher der Konflikt:

```pycon
(.venv) ~/uebung5$ git merge locker
Auto-merging gruss.txt
CONFLICT (content): Merge conflict in gruss.txt
Automatic merge failed; fix conflicts and then commit the result.
(.venv) ~/uebung5$ cat gruss.txt
<<<<<<< HEAD
Guten Tag, Welt
=======
Hi, Welt
>>>>>>> locker
```

In der Datei stehen jetzt beide Fassungen mit den Markierungen. Wir lösen den Konflikt, indem wir die Datei mit dem gewünschten Inhalt neu schreiben, merken sie vor und schließen den Merge ab.

```pycon
(.venv) ~/uebung5$ echo "Guten Tag, Welt" > gruss.txt
(.venv) ~/uebung5$ git add gruss.txt
(.venv) ~/uebung5$ git commit -q --no-edit
(.venv) ~/uebung5$ git log --oneline --graph --all
*   30e0166 Merge branch 'locker'
|\  
| * a90eb93 Lockere Anrede
* | b166833 Formelle Anrede
|/  
* 1764da3 Grundfassung
(.venv) ~/uebung5$ cat gruss.txt
Guten Tag, Welt
```

Die Grafik zeigt den Merge-Commit, in dem die Geschichte zusammenläuft. Beachten Sie, dass wir die Datei einfach neu geschrieben haben; im Alltag bearbeitet man die Markierungen im Editor und entscheidet Zeile für Zeile. Die Fehlerquelle ist das Vergessen der Markierungen: Wer mit `git add` abschließt, ohne sie zu entfernen, committet die Markierungszeilen in die Datei.

**Aufgabe 6.** Das Muster `*.env` passt auf `.env` und auf `config.example.env`; die Beispieldatei wollen wir aber behalten, daher folgt eine Ausnahme mit `!`. Die Ausnahme muss *nach* dem Muster stehen, weil spätere Zeilen die früheren überstimmen. Das Verzeichnis `daten` dürfen wir nicht als Ganzes ignorieren, denn aus einem ignorierten Verzeichnis lässt sich keine einzelne Datei zurückholen; stattdessen ignorieren wir nur `daten/*.csv`.

```pycon
(.venv) ~/uebung5$ mkdir -p ~/uebung6 && cd ~/uebung6
(.venv) ~/uebung6$ git init -q -b main
(.venv) ~/uebung6$ mkdir -p .venv/bin src/paket/__pycache__ daten
(.venv) ~/uebung6$ touch .env config.example.env notizen.bak .venv/bin/python src/paket/__pycache__/m.cpython-313.pyc daten/export.csv daten/README.md src/paket/m.py
(.venv) ~/uebung6$ git status --short
?? .env
?? .venv/
?? config.example.env
?? daten/
?? notizen.bak
?? src/
```

Ohne `.gitignore` erscheint alles als unversioniert. Jetzt die Datei:

**Datei** `.gitignore`**:**

```text
__pycache__/
.venv/
*.bak
*.env
!config.example.env
daten/*.csv
```

```pycon
(.venv) ~/uebung6$ git status --short -uall
?? .gitignore
?? config.example.env
?? daten/README.md
?? src/paket/m.py
(.venv) ~/uebung6$ git check-ignore -v .env notizen.bak daten/export.csv .venv/bin/python src/paket/__pycache__/m.cpython-313.pyc
.gitignore:4:*.env	.env
.gitignore:3:*.bak	notizen.bak
.gitignore:6:daten/*.csv	daten/export.csv
.gitignore:2:.venv/	.venv/bin/python
.gitignore:1:__pycache__/	src/paket/__pycache__/m.cpython-313.pyc
(.venv) ~/uebung6$ git check-ignore -v config.example.env; echo "Rueckgabe: $?"
.gitignore:5:!config.example.env	config.example.env
Rueckgabe: 0
```

Der Status zeigt nur noch die drei gewünschten Dateien und die `.gitignore` selbst. Die zweite Prüfung nennt für jede ignorierte Datei die Zeile der `.gitignore`, die sie ausschließt. Bei `config.example.env` zeigt `git check-ignore -v` sogar die Ausnahmezeile mit dem Ausrufezeichen an, denn es nennt jede Zeile, die über das Schicksal der Datei entschieden hat; dass die Datei *nicht* ignoriert wird, sehen Sie daran, dass sie im Status erscheint. Hätte man die Ausnahme *vor* `*.env` gestellt, wäre sie wirkungslos geblieben.

**Aufgabe 7.** Wir legen Projekt, Paket und `pyproject.toml` an. Die Funktion `zaehle` liest die Datei und liefert das Tupel; die Funktion `main` im Modul `cli` ruft sie auf und gibt die drei Zahlen aus.

```pycon
(.venv) ~/uebung6$ mkdir -p ~/uebung7/src/zeilen && cd ~/uebung7
```

**Datei** `src/zeilen/__init__.py`**:**

```python
"""zeilen - zaehlt Zeilen, Woerter und Zeichen einer Textdatei."""

__version__ = "0.1.0"
```

**Datei** `src/zeilen/core.py`**:**

```python
from pathlib import Path


def zaehle(pfad):
    """Gibt (zeilen, woerter, zeichen) der Textdatei pfad zurueck."""
    text = Path(pfad).read_text(encoding="utf-8")
    return len(text.splitlines()), len(text.split()), len(text)
```

**Datei** `src/zeilen/cli.py`**:**

```python
import argparse

from .core import zaehle


def main(argv=None):
    parser = argparse.ArgumentParser(prog="zaehle-datei", description="Zaehlt Zeilen, Woerter und Zeichen.")
    parser.add_argument("datei")
    args = parser.parse_args(argv)
    zeilen, woerter, zeichen = zaehle(args.datei)
    print(f"{zeilen} Zeilen, {woerter} Woerter, {zeichen} Zeichen")
    return 0
```

**Datei** `pyproject.toml`**:**

```text
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "zeilen"
dynamic = ["version"]
description = "Zaehlt Zeilen, Woerter und Zeichen einer Textdatei."
requires-python = ">=3.11"
dependencies = []

[project.scripts]
zaehle-datei = "zeilen.cli:main"

[tool.setuptools.dynamic]
version = {attr = "zeilen.__version__"}
```

Die Struktur entspricht der aus 13.3 und 13.4. Der Kommandoname `zaehle-datei` darf einen Bindestrich enthalten, der Modulname nicht. Jetzt richten wir die Umgebung ein, installieren und probieren aus.

```pycon
(.venv) ~/uebung7$ python3 -m venv .venv
(.venv) ~/uebung7$ . .venv/bin/activate
(.venv) ~/uebung7$ pip install -q --disable-pip-version-check -e .
(.venv) ~/uebung7$ printf 'eins zwei\ndrei\nvier fuenf sechs\n' > probe.txt
(.venv) ~/uebung7$ zaehle-datei probe.txt
3 Zeilen, 6 Woerter, 32 Zeichen
```

Die Datei hat drei Zeilen, sechs Wörter und 32 Zeichen (inklusive der Zeilenumbrüche). `splitlines()` zählt Zeilen, `split()` trennt an beliebigem Leerraum in Wörter, und `len(text)` zählt Zeichen. Eine häufige Fehlerquelle ist der Eintrag in `[project.scripts]`: Er muss den Doppelpunkt enthalten, und das Paket muss nach einer Änderung erneut installiert werden.

**Aufgabe 8.** Wir bauen zuerst das Projekt in Version 0.1.0 mit Tag und Remote nach, damit der Ausgangspunkt der Aufgabe gegeben ist. Anschließend folgt die eigentliche Lösung.

```pycon
(.venv) ~/uebung7$ mkdir -p ~/uebung8/proj && cd ~/uebung8/proj
(.venv) ~/uebung8/proj$ git init -q -b main
(.venv) ~/uebung8/proj$ mkdir -p src/tool
(.venv) ~/uebung8/proj$ printf '__version__ = "0.1.0"\n' > src/tool/__init__.py
(.venv) ~/uebung8/proj$ git add .
(.venv) ~/uebung8/proj$ git commit -q -m "Version 0.1.0"
(.venv) ~/uebung8/proj$ git tag -a v0.1.0 -m "Release 0.1.0"
(.venv) ~/uebung8/proj$ git init -q --bare -b main ~/uebung8/server.git
(.venv) ~/uebung8/proj$ git remote add origin ~/uebung8/server.git
(.venv) ~/uebung8/proj$ git push -q -u origin main
(.venv) ~/uebung8/proj$ git push -q origin v0.1.0
```

Nun erhöhen wir die Version, committen, setzen den Tag und schieben beides auf den Server.

```pycon
(.venv) ~/uebung8/proj$ sed -i 's/0.1.0/0.2.0/' src/tool/__init__.py
(.venv) ~/uebung8/proj$ git commit -q -am "Version auf 0.2.0 erhoehen"
(.venv) ~/uebung8/proj$ git tag -a v0.2.0 -m "Release 0.2.0"
(.venv) ~/uebung8/proj$ git push -q origin main v0.2.0
(.venv) ~/uebung8/proj$ git describe
v0.2.0
(.venv) ~/uebung8/proj$ git tag -n
v0.1.0          Release 0.1.0
v0.2.0          Release 0.2.0
```

Der Befehl `git describe` nennt den jüngsten erreichbaren Tag; liegt der aktuelle Commit genau auf dem Tag, ist die Ausgabe der Tag-Name selbst, sonst folgen die Zahl der Commits seither und die Kennung. Jetzt klonen wir ein zweites Mal und prüfen die Version, dann sehen wir uns den alten Stand an.

```pycon
(.venv) ~/uebung8/proj$ git clone -q ~/uebung8/server.git ~/uebung8/klon
(.venv) ~/uebung8/proj$ cd ~/uebung8/klon
(.venv) ~/uebung8/klon$ cat src/tool/__init__.py
__version__ = "0.2.0"
(.venv) ~/uebung8/klon$ git describe
v0.2.0
(.venv) ~/uebung8/klon$ git switch -q --detach v0.1.0
(.venv) ~/uebung8/klon$ cat src/tool/__init__.py
__version__ = "0.1.0"
(.venv) ~/uebung8/klon$ git switch -q main
(.venv) ~/uebung8/klon$ cat src/tool/__init__.py
__version__ = "0.2.0"
```

Der Klon enthält die Version 0.2.0, und der Checkout des Tags `v0.1.0` zeigt die alte Fassung, weil der Tag fest am damaligen Commit hängt. Mit `git switch -q main` kehren wir zum Branch zurück. Achten Sie darauf, dass Tags nur dann auf dem Server liegen, wenn sie ausdrücklich gepusht wurden; hier geschah das mit `git push origin main v0.2.0`.
