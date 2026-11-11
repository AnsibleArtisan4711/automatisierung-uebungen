# Modul 12 - Migration, Kosten, Skalierung und BCDR

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (12.1 bis 12.9; 12.10 Übungsaufgaben, 12.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, aber nur 9 h laut Summe der neun Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Die Abweichung von einer Stunde ist im Lehrplan nicht erklärt. Als Richtwert für die Seitenzahl wird die höhere Angabe (10 h) verwendet.  
**Priorität:** 3 (niedrig; alle Einheiten im Detailblatt mit Priorität 3)

| Nr.  | Lerneinheit                   | Dauer (h) | Prio |
|:-----|:------------------------------|----------:|:----:|
| 12.1 | Skript-Inventarisierung       |      1,00 |  3   |
| 12.2 | Migration von Legacy-Skripten |      1,00 |  3   |
| 12.3 | Kosten und Betriebsaufwand    |      1,00 |  3   |
| 12.4 | Standardisierung              |      1,00 |  3   |
| 12.5 | Skalierung von Automation     |      1,00 |  3   |
| 12.6 | BCDR für Skripte              |      1,00 |  3   |
| 12.7 | Change Management             |      1,00 |  3   |
| 12.8 | Betriebsübergabe              |      1,00 |  3   |
| 12.9 | Kontinuierliche Verbesserung  |      1,00 |  3   |
|      | **Summe (Detailblatt)**       |  **9,00** |      |

Die bisherigen Module haben gezeigt, wie man einzelne Skripte gut schreibt, absichert und betreibt. Dieses letzte Modul betrachtet die *Gesamtheit*: Was passiert, wenn aus fünf Skripten fünfhundert geworden sind, die über Jahre von wechselnden Personen geschrieben wurden? Wie findet man heraus, was überhaupt existiert, wer dafür zuständig ist und was davon wichtig ist? Wie ersetzt man alte, unsichere Skripte, ohne den Betrieb zu gefährden? Was kostet Automation, und was spart sie? Wie bleibt sie auch dann verfügbar, wenn ein Server, ein Repository oder ein Runner ausfällt (*BCDR*: Business Continuity und Disaster Recovery)? Und wie wird sie vom Einzelwerk zur verlässlichen, übergebbaren Dienstleistung, die sich laufend verbessert?

Die Themen sind weniger technisch als in früheren Modulen, aber nicht weniger wichtig, denn die meisten Störungen mit Skripten entstehen nicht durch einen Syntaxfehler, sondern durch fehlende Zuständigkeit, vergessene Abhängigkeiten, ungetestete Wiederherstellung und unkontrollierte Änderungen. Auch hier gilt der Anspruch dieses Skripts: Jede Einheit enthält ein lauffähiges Beispiel, mit dem sich die Idee praktisch nachvollziehen lässt. Die Beispiele sind bewusst klein; im Betrieb wachsen sie mit den Anforderungen. Zahlen in den Rechenbeispielen sind *Annahmen zur Veranschaulichung*, keine Erhebungen.

## 12.1 Skript-Inventarisierung

Die erste Frage jeder Verbesserung lautet: *Was haben wir?* Skripte entstehen unauffällig: ein Einzeiler im Crontab, ein Hilfsskript im Home-Verzeichnis, eine Pipeline-Stufe, ein Wartungsskript auf einem Server, das “schon immer da war”. Ohne Überblick lassen sich weder Risiken bewerten noch Prioritäten setzen. Eine *Inventarisierung* erfasst Skripte systematisch: wo sie liegen, was sie tun, wer sie betreut, wie kritisch sie sind und in welchem Zustand sie sich befinden.

Man unterscheidet zwei Arten von Angaben. *Technische Merkmale* lassen sich automatisch ermitteln: Pfad, Größe, Zeilenzahl, Shebang (welcher Interpreter), Zeitpunkt der letzten Änderung, Ausführrecht, ob `set -euo pipefail` verwendet wird, ob riskante Muster auftauchen (`eval`, `curl ... | bash`, `chmod 777`, Passwörter im Klartext), ob ShellCheck Warnungen meldet. *Fachliche Merkmale* kann nur ein Mensch beisteuern: Zweck, Verantwortliche (Owner), Kritikalität (was geschieht, wenn das Skript ausfällt?), Abhängigkeiten und Aufrufer (Cron, Pipeline, manuell). Damit beides zusammenkommt, führt man *Metadaten im Skriptkopf* ein, zum Beispiel Kommentarzeilen `# owner: team-betrieb` und `# kritikalitaet: hoch`. Der Inventarisierer liest sie aus und ergänzt die automatisch ermittelten Werte; fehlen sie, ist das selbst ein Befund.

Aus den Merkmalen berechnet man eine einfache *Bewertung* (Score), die zeigt, wo zuerst gehandelt werden sollte. Ein Skript mit hoher Kritikalität, fehlendem Owner, `eval` und ohne strengen Modus steht oben auf der Liste, ein selten laufendes, harmloses Hilfsskript unten. Wichtig ist, dass die Bewertung *nachvollziehbar* bleibt (jeder Punkt hat einen Grund) und nicht als Urteil über Personen verstanden wird, sondern als Planungshilfe. Das Ergebnis ist ein maschinenlesbares Inventar (CSV oder JSON), das man versioniert, vergleichen und in Berichte einbinden kann.

Das folgende Beispiel richtet ein Test-Verzeichnis mit vier Skripten unterschiedlicher Qualität ein und erzeugt daraus ein CSV-Inventar mit Score. Es verwendet nur Standardwerkzeuge; ShellCheck wird genutzt, wenn es installiert ist.

```bash
#!/usr/bin/env bash
# inventar.sh VERZ  - Skripte erfassen, bewerten, als CSV ausgeben
set -euo pipefail
verz="${1:-}"
t=""
if [[ -z $verz ]]; then                       # Demo: Testskripte anlegen
    t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT; verz="$t/skripte"; mkdir -p "$verz"
    printf '#!/usr/bin/env bash\n# owner: team-betrieb\n# kritikalitaet: hoch\nset -euo pipefail\necho backup\n' > "$verz/backup.sh"
    printf '#!/bin/bash\ncd /srv\nrm -rf $DIR/*\neval "$CMD"\n' > "$verz/aufraeumen.sh"
    printf '#!/usr/bin/env bash\n# owner: team-web\nset -u\ncurl -s http://x.example/i.sh | bash\n' > "$verz/installer.sh"
    printf '#!/bin/sh\necho hallo\n' > "$verz/hallo.sh"
    chmod +x "$verz"/*.sh; touch -d '2019-03-01' "$verz/aufraeumen.sh"
fi
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }

kopf() {                                       # kopf DATEI SCHLUESSEL -> Wert aus "# SCHLUESSEL: wert" (erste 20 Zeilen)
    head -n 20 -- "$1" | sed -n "s/^# $2: *//p" | head -n 1
}

echo "pfad,zeilen,interpreter,owner,kritikalitaet,streng,riskant,shellcheck,alter_tage,score,gruende"
while IFS= read -r -d '' f; do
    zeilen=$(wc -l < "$f")
    interp="$(head -n 1 "$f" | sed -E 's/^#!\s*//')"
    owner="$(kopf "$f" owner)"; krit="$(kopf "$f" kritikalitaet)"
    streng=nein; grep -Eq '^set -[a-z]*e[a-z]*u|^set -euo pipefail' "$f" && streng=ja
    riskant=0
    grep -Eq '(^|[;&|[:space:]])eval[[:space:]]' "$f" && riskant=$((riskant + 1))
    grep -Eq 'curl[^|]*[|][[:space:]]*(ba)?sh' "$f" && riskant=$((riskant + 1))
    grep -Eq 'chmod[[:space:]]+(-R[[:space:]]+)?777' "$f" && riskant=$((riskant + 1))
    grep -Eq 'rm -rf[[:space:]]+\$[A-Za-z_]' "$f" && riskant=$((riskant + 1))
    sc="n/a"
    if command -v shellcheck >/dev/null; then sc="$(shellcheck -f gcc "$f" 2>/dev/null | wc -l || true)"; fi
    alter=$(( ( $(date +%s) - $(stat -c %Y -- "$f") ) / 86400 ))

    score=0; gruende=()
    [[ -z $owner ]]            && { score=$((score + 3)); gruende+=("kein-owner"); }
    [[ $krit == hoch ]]        && { score=$((score + 2)); gruende+=("kritisch"); }
    [[ $streng == nein ]]      && { score=$((score + 2)); gruende+=("kein-strenger-modus"); }
    (( riskant > 0 ))          && { score=$((score + 3 * riskant)); gruende+=("riskant:$riskant"); }
    [[ $sc != n/a && $sc -gt 0 ]] && { score=$((score + 1)); gruende+=("shellcheck:$sc"); }
    (( alter > 365 ))          && { score=$((score + 1)); gruende+=("lange-unveraendert"); }
    printf '%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s\n' "${f#"$verz"/}" "$zeilen" "$interp" "${owner:--}" "${krit:--}" \
        "$streng" "$riskant" "$sc" "$alter" "$score" "$(IFS=';'; echo "${gruende[*]:-}")"
done < <(find "$verz" -type f -name '*.sh' -print0 | sort -z) | { IFS= read -r kopfzeile; echo "$kopfzeile"; sort -t, -k10,10nr; }
```

Die Funktion `kopf` liest einen Metadatenwert aus den ersten zwanzig Zeilen eines Skripts mit einem `sed`-Befehl, der nur passende Zeilen ausgibt (`-n` mit dem Flag `p`). Die Hauptschleife sammelt für jede Datei die Merkmale. Die Mustersuchen mit `grep -Eq` liefern nur einen Wahrheitswert, und jede erfüllte Bedingung erhöht den Zähler der riskanten Muster. Die Bewertung addiert gewichtete Punkte und merkt sich die Gründe in einem Array; so sieht man im Ergebnis, *warum* ein Skript weit oben steht. Das Verfahren schreibt die Zeilen als CSV und sortiert sie am Ende absteigend nach der Punktzahl (Spalte 10), wobei die Kopfzeile mit `read` abgetrennt und unverändert vorangestellt wird. In der Demo erscheint `aufraeumen.sh` mit `eval`, ungeschütztem `rm -rf $DIR` und fehlendem Owner ganz oben; `backup.sh` mit Owner, strengem Modus und ohne riskante Muster steht unten.

Typische Fehler beim Inventarisieren: Erstens erfasst man nur das Repository und übersieht Skripte auf Servern, in Crontabs, Timer-Units, Pipeline-Definitionen und Wiki-Seiten; ein vollständiges Inventar sammelt auch dort (`crontab -l`, `systemctl list-timers`, Workflow-Dateien). Zweitens verwechselt man Dateiendung und Typ: Skripte ohne `.sh` werden nicht gefunden; zuverlässiger ist die Prüfung des Shebangs. Drittens wird das Inventar einmal erstellt und nie gepflegt; es gehört in einen wiederkehrenden Lauf. Viertens gibt man Punkte, aber keine Gründe aus, und niemand versteht die Bewertung. Fünftens erfasst man Zuständigkeiten nur als Name einer Einzelperson, die das Team verlässt; besser sind Teams oder Funktionspostfächer. Sechstens kopiert man das Inventar mit Pfaden und Hostnamen in öffentliche Ablagen, obwohl es Angriffsflächen verrät.

```bash
# falsch: nur *.sh im Repo zaehlen, Ergebnis ohne Gruende
find . -name '*.sh' | wc -l

# richtig: Shebang auswerten, alle Quellen, mit Begruendung
grep -rIl --exclude-dir=.git -m1 '^#!.*\(ba\|da\)\?sh' . | sort
crontab -l 2>/dev/null; systemctl list-timers --all --no-pager 2>/dev/null
```

> **Merke**
>
> Inventar = technische Merkmale (automatisch) plus fachliche Merkmale (Owner, Zweck, Kritikalität aus dem Skriptkopf). Score mit nachvollziehbaren Gründen, nach Risiko sortiert. Alle Fundorte erfassen (Repo, Cron, Timer, Pipelines, Server), regelmäßig neu erzeugen, Ergebnis als CSV oder JSON versionieren.

> **Praxisbeispiel**
>
> Ein Team übernimmt den Betrieb einer gewachsenen Umgebung und lässt das Inventar über Repository, Crontabs und Server laufen. Das Ergebnis zeigt zwölf kritische Skripte ohne Owner; sie werden zuerst Verantwortlichen zugewiesen, bevor irgendetwas migriert wird. Das Inventar dient danach als Backlog für die Einheiten 12.2 und 12.4.

## 12.2 Migration von Legacy-Skripten

*Legacy-Skripte* sind Altbestand: wichtig, weil sie laufen, riskant, weil niemand sie wirklich versteht. Typische Merkmale sind fehlendes Quoting, `eval`, Backticks, `cd` ohne Fehlerprüfung, fest eingetragene Pfade und Passwörter, keine Tests. Der naheliegende Wunsch, alles neu zu schreiben, scheitert oft, weil das Skript heimliches Wissen enthält (Sonderfälle, Reihenfolgen, Seiteneffekte), das niemand aufgeschrieben hat. Bewährt ist die *schrittweise Migration* in kleinen, jeweils überprüfbaren Schritten, bei der das Verhalten gleich bleibt, solange man nicht ausdrücklich etwas ändern will.

Der Schlüssel ist ein *Charakterisierungstest* (auch “Golden Master”): Bevor man etwas ändert, lässt man das Skript in einer kontrollierten Testumgebung mit festen Eingaben laufen und speichert Ausgabe, Exit-Code und Dateiveränderungen als Referenz. Nach jedem Umbauschritt führt man denselben Test aus und vergleicht. Weicht das Ergebnis ab, war der Schritt entweder fehlerhaft oder hat versteckt vorhandenes Verhalten verändert, und man entscheidet bewusst. So macht man Altbestand sicher änderbar, ohne ihn erst vollständig zu verstehen. Danach folgen die Umbauschritte in sinnvoller Reihenfolge: zuerst *mechanisch Sicheres* (Shebang, strenger Modus nur dort, wo er das Verhalten nicht verändert, Quoting, `$(...)` statt Backticks), dann *Struktur* (Funktionen, `main`, Argumentprüfung, Logging), zuletzt *Sicherheit* (Secrets herausziehen, `eval` ersetzen, Rechte). ShellCheck leitet die Reihenfolge an.

Die zweite Frage lautet: *Soll es überhaupt Bash bleiben?* Bash eignet sich für das Verketten von Programmen, Dateioperationen und kleine Automationen, deren Kern in Kommandos besteht. Sobald ein Skript komplexe Datenstrukturen, umfangreiche Textverarbeitung, Netzwerk-APIs mit viel JSON, Nebenläufigkeit oder Tests in größerem Umfang braucht, ist eine Sprache wie Python meist besser wartbar. *Ansible* (oder ein anderes Konfigurationsmanagement) ist die bessere Wahl, wenn der Zielzustand vieler Server *deklarativ* und *idempotent* hergestellt werden soll: Pakete, Dateien, Dienste, Benutzer. Eine grobe Faustregel: Wenn das Skript im Kern “stelle Zustand X her” ausdrückt, gehört es in Ansible oder IaC; wenn es “rufe diese Programme der Reihe nach auf und werte ihre Ausgabe aus” ausdrückt, bleibt es Bash; wenn es “verarbeite strukturierte Daten mit Logik” ausdrückt, wird es Python. Die Entscheidung dokumentiert man pro Skript.

Das Beispiel zeigt die Methode an einem kleinen Altskript: ein Test erfasst das Ist-Verhalten, dann wird das Skript Schritt für Schritt modernisiert, und der Test bestätigt nach jedem Schritt, dass sich nichts verändert hat. Das Altskript hat bewusst typische Mängel (kein Quoting, Backticks, ungeprüftes `cd`).

```bash
#!/usr/bin/env bash
# migration-demo.sh - Golden-Master-Test und schrittweise Modernisierung eines Altskripts
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
cd "$t"

# --- Altskript (Legacy): zaehlt Zeilen je .log-Datei und schreibt einen Bericht
cat > alt.sh <<'ENDE'
#!/bin/bash
cd $1
for f in `ls *.log`
do
  n=`wc -l < $f`
  echo $f $n
done
ENDE

# --- Testdaten, auch mit Leerzeichen im Dateinamen (typischer Sonderfall)
mkdir daten; printf 'a\nb\nc\n' > daten/eins.log; printf 'x\n' > daten/zwei.log

# --- 1. Golden Master festhalten (Referenz des Ist-Verhaltens)
bash alt.sh daten > master.txt
echo "Referenz erzeugt:"; sed 's/^/   /' master.txt

vergleiche() {                                 # vergleiche SKRIPT BEZEICHNUNG
    if diff -u master.txt <(bash "$1" daten) >/dev/null; then echo "[OK ] $2: Verhalten unveraendert"
    else echo "[ABW] $2: Verhalten weicht ab"; diff -u master.txt <(bash "$1" daten) | tail -n +3 | sed 's/^/     /' || true; fi
}

# --- Schritt A (mechanisch): Backticks -> $(...), Quoting
cat > a.sh <<'ENDE'
#!/bin/bash
cd "$1"
for f in $(ls *.log)
do
  n=$(wc -l < "$f")
  echo "$f" "$n"
done
ENDE
vergleiche a.sh "Schritt A (Backticks, Quoting)"

# --- Schritt B (Struktur): Fehlerpruefung, kein ls, Glob statt Parsing, main
cat > b.sh <<'ENDE'
#!/usr/bin/env bash
set -euo pipefail
main() {
    local verz="${1:?Aufruf: $0 VERZ}" f n
    cd -- "$verz"
    shopt -s nullglob
    for f in *.log; do
        n=$(wc -l < "$f")
        echo "$f" "$n"
    done
}
main "$@"
ENDE
vergleiche b.sh "Schritt B (strenger Modus, Glob, main)"

# --- Schritt C: bewusste Verhaltensaenderung (Ausgabeformat) wird vom Test sichtbar gemacht
sed 's/echo "\$f" "\$n"/printf "%s: %s\\n" "$f" "$n"/' b.sh > c.sh
vergleiche c.sh "Schritt C (neues Ausgabeformat)"
```

Der Ablauf folgt der oben beschriebenen Methode. Zuerst entsteht die Referenz durch einen Lauf des Altskripts mit festen Testdaten. Die Funktion `vergleiche` führt ein Skript erneut aus und vergleicht mit `diff`; nur wenn keine Abweichung besteht, meldet sie `OK`. Schritt A modernisiert rein mechanisch und ändert das Verhalten nicht, Schritt B verbessert die Struktur: Er ersetzt das Parsen der `ls`-Ausgabe (fehleranfällig bei Leerzeichen und Sonderzeichen) durch ein Glob-Muster, prüft das `cd`, aktiviert den strengen Modus und stellt eine `main`-Funktion davor. Dass auch dieser Schritt dieselbe Ausgabe liefert, bestätigt der Test. Schritt C ändert das Ausgabeformat absichtlich und zeigt, wie der Test eine Verhaltensänderung sofort sichtbar macht; man entscheidet dann, ob sie gewollt ist, und aktualisiert gegebenenfalls die Referenz mit einem Vermerk im Änderungsprotokoll.

Typische Fehler: Erstens beginnt man mit dem Umbau ohne Referenz und weiß hinterher nicht, was sich geändert hat. Zweitens ändert man in einem Schritt alles gleichzeitig (Struktur, Verhalten, Sprache), und bei Fehlern ist die Ursache unauffindbar. Drittens schaltet man `set -e` in ein Altskript, das bisher Fehler stillschweigend überging; plötzlich bricht es an Stellen ab, die früher “egal” waren, weil ein Befehl regulär einen Exit-Code ungleich null liefert (Modul 6.1). Viertens ersetzt man ein bewährtes Skript durch Python, ohne den Mehrwert zu benennen. Fünftens vergisst man Sonderfälle, die nur in Produktion auftreten (Dateinamen mit Leerzeichen, leere Verzeichnisse, andere Locale), und nimmt sie nicht in die Testdaten auf. Sechstens betreibt man alt und neu nicht parallel, sodass es keinen Rückweg gibt; besser läuft die neue Version zunächst im “Schattenbetrieb” und wird verglichen, bevor sie die alte ablöst.

```bash
# falsch: alles auf einmal umbauen, ls parsen, ohne Test
for f in `ls $DIR`; do cp $f /backup; done

# richtig: Referenz sichern, kleine Schritte, Glob statt ls, Fehler pruefen
shopt -s nullglob
for f in "$DIR"/*; do cp -- "$f" /backup/ || { echo "Kopieren fehlgeschlagen: $f" >&2; exit 1; }; done
```

> **Merke**
>
> Erst Verhalten festhalten (Golden Master), dann in kleinen Schritten umbauen und nach jedem Schritt vergleichen. Reihenfolge: mechanisch Sicheres, Struktur, Sicherheit. Bash für Programmverkettung, Ansible/IaC für Zielzustände, Python für strukturierte Datenlogik. Schattenbetrieb vor der Ablösung; Entscheidung dokumentieren.

> **Praxisbeispiel**
>
> Ein zehn Jahre altes Aufräumskript soll abgelöst werden. Das Team erstellt zuerst Testdaten und Referenz, bringt es mit ShellCheck-Befunden schrittweise auf Stand und lässt die neue Version vier Wochen parallel im Trockenlauf (nur protokollieren) mitlaufen. Erst nachdem beide Versionen dieselben Dateien auswählen, wird das alte Skript abgeschaltet und archiviert.

## 12.3 Kosten und Betriebsaufwand

Automation kostet Zeit (Entwicklung, Prüfung, Pflege) und spart Zeit (weniger manuelle Arbeit, weniger Fehler, schnellere Abläufe). Ob sich ein Skript lohnt, ist keine Gefühlsfrage, sondern lässt sich überschlägig rechnen. Das ist besonders dann wichtig, wenn Budget und Prioritäten verhandelt werden: “Das Skript ist doch nur ein paar Zeilen” ist leichter zu entkräften mit “die manuelle Variante kostet uns jährlich 120 Arbeitsstunden”. Ziel dieser Einheit ist, manuelle Arbeit und Fehlerkosten sichtbar zu machen und den Nutzen von Automation messbar zu halten.

Die Grundrechnung hat vier Größen. Die *manuelle Dauer* eines Vorgangs, seine *Häufigkeit*, die *Entwicklungs- und Pflegekosten* der Automation und die *Fehlerkosten*, also Wahrscheinlichkeit mal Schaden eines Fehlers. Die jährliche Ersparnis ist (manuelle Dauer minus verbleibender Aufwand) mal Häufigkeit mal Stundensatz, zuzüglich vermiedener Fehlerkosten. Der *Break-even-Punkt* gibt an, nach wie vielen Monaten die Entwicklungskosten wieder eingespielt sind. Zwei Fallen sind bekannt: Man vergisst den *laufenden* Aufwand (Anpassungen, Fehlersuche, Wartung des Skripts), und man vernachlässigt *nichtmonetäre* Wirkungen (Reproduzierbarkeit, Nachvollziehbarkeit, weniger Wissen in einzelnen Köpfen). Der erste Punkt lässt sich beziffern, der zweite verbal begründen.

Die Messbarkeit entsteht durch *Kennzahlen*, die das Skript selbst liefert: Zahl der Läufe, Laufzeit, Erfolgsquote, Zahl der betroffenen Objekte. Erinnern Sie sich an die Logs aus Modul 9: Wenn jeder Lauf eine Zeile mit Dauer und Ergebnis schreibt, lässt sich der Nutzen später aus den Daten belegen, statt ihn zu schätzen. Nützlich ist außerdem ein Vorher-Nachher-Vergleich: Man misst vor der Einführung kurz die manuelle Dauer (Stoppuhr, Ticketzeiten) und nach der Einführung den Aufwand für Aufsicht und Korrektur.

Das Beispiel rechnet einen Vorgang durch und gibt eine Tabelle mit Break-even aus. Die Zahlenwerte sind Beispielannahmen, nur zur Veranschaulichung.

```bash
#!/usr/bin/env bash
# nutzen.sh - Wirtschaftlichkeit einer Automation ueberschlaegig berechnen (Werte sind Annahmen)
set -euo pipefail
# Parameter per Umgebung aenderbar; Standardwerte sind reine Beispielannahmen
MANUELL_MIN="${MANUELL_MIN:-45}"             # Minuten je manuellem Vorgang
PRO_JAHR="${PRO_JAHR:-104}"                  # Haeufigkeit (z.B. 2 x pro Woche)
STUNDENSATZ="${STUNDENSATZ:-60}"             # Kosten je Stunde (Annahme)
ENTWICKLUNG_H="${ENTWICKLUNG_H:-24}"         # Aufwand fuer Entwicklung und Test
PFLEGE_H_JAHR="${PFLEGE_H_JAHR:-6}"          # laufende Pflege pro Jahr
AUFSICHT_MIN="${AUFSICHT_MIN:-5}"            # Aufsicht je automatischem Lauf
FEHLER_QUOTE="${FEHLER_QUOTE:-0.04}"         # Fehlerwahrscheinlichkeit je manuellem Vorgang
FEHLER_KOSTEN="${FEHLER_KOSTEN:-400}"        # Schaden je Fehler (Annahme)

for v in MANUELL_MIN PRO_JAHR STUNDENSATZ ENTWICKLUNG_H PFLEGE_H_JAHR AUFSICHT_MIN FEHLER_QUOTE FEHLER_KOSTEN; do
    [[ ${!v} =~ ^[0-9]+([.][0-9]+)?$ ]] || { echo "Parameter $v ungueltig: ${!v}" >&2; exit 2; }
done

awk -v m="$MANUELL_MIN" -v n="$PRO_JAHR" -v s="$STUNDENSATZ" -v e="$ENTWICKLUNG_H" \
    -v p="$PFLEGE_H_JAHR" -v a="$AUFSICHT_MIN" -v q="$FEHLER_QUOTE" -v k="$FEHLER_KOSTEN" '
BEGIN {
    manuell_h   = m * n / 60                          # Stunden pro Jahr, manuell
    automat_h   = a * n / 60 + p                      # Aufsicht plus Pflege pro Jahr
    ersparnis_h = manuell_h - automat_h
    fehler_manu = n * q * k                           # erwartete Fehlerkosten manuell
    nutzen      = ersparnis_h * s + fehler_manu       # Annahme: automatisch fehlerfrei genug -> Fehlerkosten entfallen
    einmalig    = e * s
    printf "Manuell:      %7.1f h/Jahr\n", manuell_h
    printf "Automatisch:  %7.1f h/Jahr (Aufsicht und Pflege)\n", automat_h
    printf "Ersparnis:    %7.1f h/Jahr = %8.0f Euro\n", ersparnis_h, ersparnis_h * s
    printf "Fehlerkosten: %7.0f Euro/Jahr vermieden (Annahme)\n", fehler_manu
    printf "Einmalig:     %7.0f Euro Entwicklung\n", einmalig
    if (nutzen <= 0) { print "Break-even: nie (Nutzen <= 0)"; exit 1 }
    printf "Nutzen/Jahr:  %7.0f Euro\n", nutzen
    printf "Break-even:   nach %.1f Monaten\n", einmalig / nutzen * 12
}'
```

Das Skript prüft zuerst, dass alle Parameter Zahlen sind (ein typischer Eingabefehler, der `awk` sonst stillschweigend als 0 rechnen ließe). Die Berechnung selbst läuft in `awk`, weil die Shell keine Gleitkommarechnung beherrscht (Modul 3.5). Die Variablen werden mit `-v` übergeben und nicht in den Programmtext eingesetzt, was auch hier Sicherheit und Lesbarkeit verbessert. Die Ausgabe trennt wiederkehrende Größen (Stunden und Kosten pro Jahr) von der einmaligen Investition und endet im Break-even in Monaten. Mit den Standardannahmen sparen 104 Vorgänge à 45 Minuten rund 78 Stunden im Jahr, von denen etwa 8,7 Stunden Aufsicht und 6 Stunden Pflege abgehen. Sie können Werte per Umgebungsvariable ändern (`PRO_JAHR=12 ./nutzen.sh`) und sehen, dass sich seltene Vorgänge kaum automatisieren lassen: Bei zwölf Läufen im Jahr bleibt der Nutzen klein oder negativ.

Typische Fehler: Erstens vergleicht man nur die Entwicklungszeit mit einem einzelnen manuellen Lauf und übersieht die Häufigkeit. Zweitens rechnet man Pflege und Aufsicht nicht ein. Drittens verwendet man frei erfundene Zahlen als Tatsachen; Annahmen müssen als solche gekennzeichnet und, wo möglich, aus Daten (Tickets, Zeiterfassung, Logs) gewonnen werden. Viertens misst man nach der Einführung nichts mehr, sodass ein Skript, das nie läuft oder dauernd scheitert, unbemerkt Kosten statt Nutzen erzeugt. Fünftens entscheidet man ausschließlich nach Geld und ignoriert Risiko und Compliance: Ein selten ausgeführter Notfallvorgang lohnt sich als Automation, weil Fehler unter Druck besonders teuer sind. Sechstens behandelt man Automation als einmaliges Projekt statt als Betriebsaufwand; ein Skript ohne Pflege veraltet.

```bash
# falsch: nur die Entwicklungszeit betrachten, Annahmen als Fakten
echo "Automation kostet 24 h, also teuer, lassen wir es"

# richtig: Haeufigkeit, Pflege und Aufsicht einrechnen, Zahlen aus Daten
awk -F, '$3 == "ok" {n++; s += $2} END {printf "Laeufe: %d, mittlere Dauer: %.1f s\n", n, s/n}' laeufe.csv
PRO_JAHR=104 AUFSICHT_MIN=5 ./nutzen.sh
```

> **Merke**
>
> Nutzen = (manuelle Zeit $-$ Aufsicht $-$ Pflege) × Häufigkeit × Stundensatz plus vermiedene Fehlerkosten; Break-even = Einmalaufwand durch Jahresnutzen. Annahmen kennzeichnen, aus echten Daten ersetzen, Kennzahlen im Betrieb mitschreiben. Seltene, aber riskante Vorgänge können sich auch ohne hohen Zeitgewinn lohnen.

> **Praxisbeispiel**
>
> Bevor ein Team ein Wartungsskript für einen vierteljährlichen Vorgang baut, rechnet es durch: Der Zeitgewinn ist gering, aber der Vorgang war schon zweimal durch Tippfehler fehlgeschlagen. Mit den Fehlerkosten ergibt sich ein positiver Break-even innerhalb eines Jahres; das Skript wird gebaut und schreibt Laufzeit und Ergebnis in das Betriebsprotokoll, damit der Nutzen später belegbar ist.

## 12.4 Standardisierung

Wenn jede Person Skripte nach eigenem Geschmack schreibt, gleicht kein Skript dem anderen: andere Argumentbehandlung, anderes Logging, andere Fehlerbehandlung. Das kostet Einarbeitung, erschwert Reviews und lässt jedes Skript dieselben Fehler neu machen. *Standardisierung* bündelt bewährte Lösungen an einer Stelle und macht sie für alle verfügbar. Die Einheit 10.8 hat die Standards im Überblick beschrieben (Namen, Gerüst, Stil, Prüfung, Review); hier geht es um die Umsetzung als *Bausteine*: Vorlagen, wiederverwendbare Bibliotheken und Werkzeuge, die den Standard leicht befolgbar machen. Die wichtigste Regel dabei: Was einfacher ist als der Eigenbau, wird tatsächlich verwendet.

Drei Bausteine haben sich bewährt. Erstens eine *Vorlage* (Template) für neue Skripte mit Shebang, Kopfkommentar samt Metadaten (Owner, Zweck, Kritikalität, siehe 12.1), strengem Modus, `usage`, Argumentverarbeitung und `main`. Ein kleines Erzeugerskript legt aus der Vorlage ein neues Skript an, sodass niemand bei null beginnt. Zweitens eine *Bibliothek* gemeinsamer Funktionen (Logging, Fehlermeldung, Pflichtprüfungen, Wiederholung mit Backoff, temporäre Verzeichnisse mit Aufräumen). Sie wird mit `source` eingebunden. Wichtig sind ein Schutz gegen doppeltes Laden, ein Präfix für alle Namen (`lib_log`, `lib_die`), damit sie nicht mit Skriptfunktionen kollidieren, und eine *Version*, damit Skripte feststellen können, ob die Bibliothek zu ihnen passt. Drittens ein *Styleguide* mit wenigen, knappen, begründeten Regeln und ein *Prüfwerkzeug*, das sie automatisch kontrolliert (10.8).

Die Bibliothek ist auch ein Risiko: Ein Fehler in ihr wirkt auf alle Skripte, und `source` führt den Inhalt mit den Rechten des Skripts aus (Modul 8.3). Deshalb liegt sie unter Versionskontrolle in einem Verzeichnis mit eingeschränkten Schreibrechten, wird gepinnt (eine feste Version, kein “das Neueste vom Hauptzweig”) und mit Tests abgesichert. Änderungen an der Bibliothek folgen den Regeln des Change Managements (12.7), denn ihre Wirkung ist breit. Gegen Verwechslung dokumentiert man die *Schnittstelle*: welche Funktionen existieren, welche Variablen sie setzen, welche Exit-Codes sie liefern.

Das Beispiel besteht aus drei Teilen: einer Bibliothek, einem Erzeugerskript, das aus einer Vorlage ein neues Skript anlegt, und einem Beispielaufruf. Alles läuft in einem Testverzeichnis.

```bash
#!/usr/bin/env bash
# standard-demo.sh - Bibliothek, Vorlage und Generator im Zusammenspiel
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/lib" "$t/bin"

# ---- 1. Bibliothek lib/standard.sh (mit Ladeschutz, Praefix und Version)
cat > "$t/lib/standard.sh" <<'ENDE'
# standard.sh - gemeinsame Funktionen (nur mit source einbinden)
[[ -n ${LIB_STANDARD_GELADEN:-} ]] && return 0
LIB_STANDARD_GELADEN=1
LIB_STANDARD_VERSION="1.2.0"

lib_log()  { printf '%s [%s] %s\n' "$(date -u +%FT%TZ)" "${SKRIPT_NAME:-?}" "$*" >&2; }
lib_fehler() { lib_log "FEHLER: $*"; }
lib_die()  { local rc="$1"; shift; lib_fehler "$*"; exit "$rc"; }          # lib_die CODE MELDUNG
lib_brauche() { local b; for b in "$@"; do command -v "$b" >/dev/null 2>&1 || lib_die 2 "Programm fehlt: $b"; done; }
lib_version_mind() {                                                         # lib_version_mind 1.1.0
    [[ "$(printf '%s\n%s\n' "$1" "$LIB_STANDARD_VERSION" | sort -V | head -n 1)" == "$1" ]] \
        || lib_die 2 "standard.sh $LIB_STANDARD_VERSION ist aelter als geforderte $1"
}
lib_tmp() {                                                                  # legt tmp-Verz. an, raeumt beim Ende auf
    LIB_TMP="$(mktemp -d)"; trap 'rm -rf -- "$LIB_TMP"' EXIT
}
ENDE

# ---- 2. Vorlage templates/skript.sh.vorlage mit Platzhaltern @NAME@, @ZWECK@, @OWNER@
mkdir -p "$t/templates"
cat > "$t/templates/skript.sh.vorlage" <<'ENDE'
#!/usr/bin/env bash
# @NAME@ - @ZWECK@
# owner: @OWNER@
# kritikalitaet: niedrig
set -euo pipefail
SKRIPT_NAME="${0##*/}"
LIB_VERZ="${LIB_VERZ:-$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")/../lib" && pwd)}"
# shellcheck source=/dev/null
source "$LIB_VERZ/standard.sh"
lib_version_mind 1.1.0

usage() { echo "Aufruf: $SKRIPT_NAME [-h] EINGABE"; }

main() {
    [[ ${1:-} == -h ]] && { usage; return 0; }
    local eingabe="${1:?$(usage)}"
    lib_log "Start mit Eingabe: $eingabe"
    # TODO: hier die eigentliche Arbeit
    lib_log "Ende"
}
main "$@"
ENDE

# ---- 3. Generator: neues Skript aus der Vorlage (mit Pruefung der Namen)
neu_skript() {                                   # neu_skript NAME "ZWECK" OWNER
    local name="$1" zweck="$2" owner="$3" ziel="$t/bin/$1.sh"
    [[ $name =~ ^[a-z]+(-[a-z0-9]+)+$ ]] || { echo "Name muss verb-objekt sein (klein, mit Bindestrich): $name" >&2; return 2; }
    [[ -e $ziel ]] && { echo "existiert bereits: $ziel" >&2; return 1; }
    sed -e "s|@NAME@|$name.sh|g" -e "s|@ZWECK@|$zweck|g" -e "s|@OWNER@|$owner|g" \
        "$t/templates/skript.sh.vorlage" > "$ziel"
    chmod +x "$ziel"; echo "angelegt: bin/${ziel##*/}"
}
neu_skript pruefe-konfig "Prueft Konfigurationsdateien" team-betrieb
neu_skript Kaputt "falscher Name" team-x || echo "(erwartet abgelehnt, rc=$?)"

echo "--- Ausfuehren"
"$t/bin/pruefe-konfig.sh" app.conf
echo "--- ohne Argument"
"$t/bin/pruefe-konfig.sh" 2>&1 | head -n 2 || true
```

Die Bibliothek beginnt mit dem Ladeschutz: Wird sie ein zweites Mal eingebunden, kehrt sie sofort zurück und definiert nichts doppelt. Alle Funktionen tragen das Präfix `lib_`, damit Kollisionen mit Funktionen des Skripts ausgeschlossen sind. `lib_die` nimmt den Exit-Code als erstes Argument und vereinheitlicht damit Fehlerabbrüche. `lib_version_mind` verhindert, dass ein Skript mit einer zu alten Bibliothek läuft. Die Vorlage nutzt Platzhalter in der Form `@NAME@`, die der Generator mit `sed` ersetzt; als Trennzeichen dient hier der senkrechte Strich, weil Texte Schrägstriche enthalten können. Das Erzeugerskript prüft den Namen gegen das Muster `verb-objekt` und weigert sich, vorhandene Dateien zu überschreiben. Im erzeugten Skript sehen Sie die Standardbestandteile: Metadaten im Kopf, strenger Modus, `main`-Funktion, Hilfetext und das Laden der Bibliothek über einen Pfad relativ zum Skript, der durch eine Umgebungsvariable überschrieben werden kann (nützlich für Tests).

Typische Fehler: Erstens kopiert jeder die Bibliothekfunktionen in das eigene Skript (Wildwuchs), statt sie einzubinden; Fehlerkorrekturen erreichen dann nur einen Teil. Zweitens erfindet man eine Bibliothek, die alles kann, und niemand überblickt sie; kleine, fokussierte Bibliotheken sind besser. Drittens fehlen Ladeschutz, Präfix und Version, und zwei Bibliotheken überschreiben gegenseitig ihre Funktionen. Viertens lädt man die Bibliothek aus einem beschreibbaren oder unsicheren Pfad (zum Beispiel aus dem aktuellen Verzeichnis), sodass jemand Code einschleusen kann. Fünftens ändert man die Bibliothek ohne Tests und ohne Ankündigung, und abhängige Skripte brechen. Sechstens ist der Styleguide ein langes Dokument mit hundert Regeln; kurze, begründete und automatisch geprüfte Regeln werden befolgt. Siebtens wird die Vorlage nie nachgezogen und enthält veraltete Muster.

```bash
# falsch: Bibliothek aus dem aktuellen Verzeichnis, ohne Ladeschutz und Version
source ./lib.sh

# richtig: fester, nicht schreibbarer Pfad relativ zum Skript, Version geprueft
LIB_VERZ="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")/../lib" && pwd)"
source "$LIB_VERZ/standard.sh"
lib_version_mind 1.1.0
```

> **Merke**
>
> Standards als Bausteine: Vorlage plus Generator, kleine Bibliothek (Ladeschutz, Präfix, Version, dokumentierte Schnittstelle), kurzer Styleguide mit automatischer Prüfung. Bibliothek aus festem, geschütztem Pfad einbinden, Version pinnen, Änderungen testen und per Change Management einführen.

> **Praxisbeispiel**
>
> Ein Team führt `neu-skript.sh` ein: Wer ein Skript braucht, startet den Generator und bekommt ein lauffähiges Gerüst samt Hilfetext, Logging und Metadaten. Die Zahl der Skripte ohne Owner sinkt, und Reviews drehen sich um die Fachlogik, weil Form und Fehlerbehandlung schon im Gerüst stimmen.

## 12.5 Skalierung von Automation

Automation, die für zehn Server funktioniert, scheitert bei tausend an anderen Dingen als am Code: Die Laufzeit wird zu lang, Ziele und Dienste werden überlastet (Rate Limits), die Ausführung auf einzelnen Rechnern lässt sich nicht mehr überblicken, und Fehler treten nicht mehr einzeln, sondern als Muster auf. Skalierung heißt hier nicht, ein Skript zu beschleunigen, sondern die Ausführung zu *planen*: wie viel gleichzeitig, auf welcher Plattform, mit welchen Grenzen, und wie man Teilergebnisse einsammelt.

Eine erste Stellgröße ist die *Parallelität*, die in Einheit 10.6 mit `xargs -P` und Job-Pools eingeführt wurde. Hier kommt die Frage nach der *richtigen Grenze* hinzu. Sie hängt vom schwächsten Glied ab: dem Jump-Host, der Netzbandbreite, einer API mit Anfragelimit, einer Datenbank mit begrenzter Zahl von Verbindungen. Die Grenze wird in einer Konfiguration festgelegt, nicht im Code, und anhand von Messungen angepasst. Bei APIs mit *Rate Limits* reicht eine feste Zahl gleichzeitiger Prozesse nicht, man braucht eine *Rate*: höchstens N Anfragen pro Sekunde. Dafür dient eine einfache Drosselung (Mindestabstand zwischen Anfragen) oder ein Token-Bucket. Reagiert die API mit Fehlern wie “zu viele Anfragen” (HTTP 429), wartet man mit Backoff (11.1) und respektiert Angaben des Servers zur Wartezeit.

Eine zweite Stellgröße ist das *Aufteilen* (Sharding) einer großen Menge von Zielen in Gruppen, die auf mehreren Läufern oder zeitlich versetzt verarbeitet werden. Ein stabiles Verfahren ist, jedem Ziel anhand seines Namens (Hash modulo Anzahl der Gruppen) eine Gruppe zuzuordnen: Dasselbe Ziel landet immer in derselben Gruppe, und die Last verteilt sich ungefähr gleichmäßig. Dazu gehören *gestaffelte Rollouts* (erst 5 %, dann 25 %, dann alle) mit Prüfung zwischen den Stufen, damit ein Fehler früh auffällt und nicht alles gleichzeitig trifft.

Die dritte Entscheidung betrifft den *Ort der Ausführung*: zentral (ein Orchestrator oder eine Pipeline führt alles aus, zum Beispiel über SSH oder Agenten) oder lokal (jeder Server führt sein Skript selbst aus, etwa per Cron oder Timer, und meldet das Ergebnis). Zentrale Ausführung bietet einen Überblick, einheitliche Protokollierung, einfache Freigaben und Abbruch; sie erzeugt aber einen zentralen Engpass und Ausfallpunkt und braucht weitreichende Zugriffsrechte auf viele Systeme. Lokale Ausführung skaliert von selbst und funktioniert auch bei Netzstörungen, ist aber schwerer zu überblicken, zu aktualisieren und zu überwachen. Häufig wählt man einen Mittelweg: Die Skripte laufen lokal, werden aber zentral verteilt, versioniert und überwacht (zum Beispiel über Konfigurationsmanagement und ein Meldeverfahren mit Heartbeat, siehe 9.4).

Das Beispiel zeigt zwei Bausteine: ein stabiles Sharding nach Hostnamen und eine Ratenbegrenzung, die höchstens eine bestimmte Zahl von Aufrufen pro Sekunde zulässt.

```bash
#!/usr/bin/env bash
# skalierung-demo.sh - Sharding nach Hostnamen und Ratenbegrenzung
set -euo pipefail

# ---- 1. Stabiles Sharding: Host -> Gruppe 0..N-1 (cksum ist ueberall verfuegbar)
shard_von() {                                   # shard_von HOST ANZAHL
    local host="$1" anzahl="$2" summe
    summe="$(printf '%s' "$host" | cksum | awk '{print $1}')"
    echo $(( summe % anzahl ))
}

hosts=(web-01 web-02 web-03 web-04 web-05 web-06 db-01 db-02 cache-01 cache-02 mail-01 batch-01)
ANZAHL=3
echo "== Aufteilung auf $ANZAHL Gruppen"
declare -A gruppen=()
for h in "${hosts[@]}"; do
    s="$(shard_von "$h" "$ANZAHL")"
    gruppen[$s]+="$h "
done
for s in $(seq 0 $((ANZAHL - 1))); do printf 'Gruppe %s: %s\n' "$s" "${gruppen[$s]:-}"; done
echo "Wiederholung ergibt dasselbe: web-03 -> $(shard_von web-03 "$ANZAHL"), $(shard_von web-03 "$ANZAHL")"

# ---- 2. Ratenbegrenzung: hoechstens RATE Aufrufe pro Sekunde (Mindestabstand)
RATE="${RATE:-5}"
[[ $RATE =~ ^[1-9][0-9]*$ ]] || { echo "RATE ungueltig" >&2; exit 2; }
abstand_ms=$(( 1000 / RATE ))
letzter_ns=0
drosseln() {                                    # wartet, bis der Mindestabstand seit dem letzten Aufruf vergangen ist
    local jetzt_ns warte_ms
    jetzt_ns=$(date +%s%N)
    if (( letzter_ns > 0 )); then
        warte_ms=$(( abstand_ms - (jetzt_ns - letzter_ns) / 1000000 ))
        (( warte_ms > 0 )) && sleep "$(awk -v m="$warte_ms" 'BEGIN{printf "%.3f", m/1000}')"
    fi
    letzter_ns=$(date +%s%N)
}

echo "== 10 Aufrufe mit RATE=$RATE pro Sekunde"
start=$(date +%s%N)
for i in $(seq 1 10); do drosseln; done
dauer_ms=$(( ( $(date +%s%N) - start ) / 1000000 ))
echo "Dauer: ${dauer_ms} ms (mindestens $(( 9 * abstand_ms )) ms erwartet)"
(( dauer_ms >= 9 * abstand_ms - 20 )) && echo "Ratenbegrenzung wirksam" || { echo "zu schnell!" >&2; exit 1; }
```

Das Sharding nutzt die Prüfsumme des Hostnamens (`cksum` gibt eine Zahl aus, die für gleiche Eingaben gleich bleibt) und bildet sie mit dem Modulo-Operator auf die Gruppennummer ab. Das ist deterministisch und verteilt kleine Mengen nicht perfekt gleichmäßig, bei großen Mengen aber ausreichend. Die Gruppen sammelt das Skript in einem assoziativen Array. Die Drosselung merkt sich den Zeitpunkt des letzten Aufrufs und wartet, bis der Mindestabstand (1000 Millisekunden geteilt durch die Rate) vergangen ist. Die Messung am Ende bestätigt, dass zehn Aufrufe bei fünf pro Sekunde mindestens neun Abstände von 200 ms brauchen. Die Rate ist per Variable änderbar. Beim Einsatz mit paralleler Ausführung muss jeder Prozess den Mindestabstand *gemeinsam* beachten, etwa über einen zentralen Taktgeber oder durch Aufteilen der Rate auf die Prozesse; sonst multipliziert sich die Rate mit der Zahl der Prozesse.

Typische Fehler: Erstens startet man bei Wachstum einfach mehr parallele Prozesse und überlastet das Ziel, ohne dass der eigene Rechner an die Grenze kommt. Zweitens ignoriert man *Rate-Limit-Antworten* (429, Retry-After) und läuft in Sperren. Drittens führt ein zentraler Orchestrator alles mit einem einzigen, weitreichenden Schlüssel aus, dessen Verlust das ganze Netz gefährdet (Modul 8.5). Viertens fehlt bei großen Rollouts eine Stufung, und ein Fehler trifft alle Systeme gleichzeitig. Fünftens verteilt man Ziele zufällig auf Gruppen, sodass ein Wiederholungslauf andere Gruppen bildet und sich Zustand und Verantwortung nicht zuordnen lassen. Sechstens ist die Rate-Begrenzung pro Prozess gebaut, obwohl das Limit für das Konto gilt. Siebtens kennt niemand die tatsächliche Laufzeit; Zeitmessungen gehören ins Log (9.1).

```bash
# falsch: alle 2000 Hosts gleichzeitig, kein Stufen, Rate-Limit ignoriert
for h in $(cat hosts.txt); do ssh "$h" ./update.sh & done; wait

# richtig: Stufen, begrenzte Parallelitaet, Pruefung zwischen den Stufen
for anteil in 5 25 100; do
    ./waehle-hosts.sh "$anteil" | xargs -P 10 -n 1 ./update-host.sh || { echo "Stopp bei ${anteil}%" >&2; exit 1; }
    ./gesundheitspruefung.sh || { echo "Gesundheitspruefung nach ${anteil}% fehlgeschlagen" >&2; exit 1; }
done
```

> **Merke**
>
> Skalierung planen: Parallelität am schwächsten Glied ausrichten, Rate (pro Sekunde) statt nur Prozesszahl begrenzen, deterministisch aufteilen (Hash modulo N), gestaffelt ausrollen mit Prüfung zwischen den Stufen. Zentral gegen lokal abwägen: Überblick und Kontrolle gegen Engpass und Rechteballung; oft ist verteilte Ausführung mit zentraler Steuerung der Mittelweg.

> **Praxisbeispiel**
>
> Ein Rollout auf 800 Server läuft in drei Wellen: zehn Server als Probe, dann zehn Prozent, dann der Rest. Nach jeder Welle prüft ein Gesundheitsskript Fehlerquote und Antwortzeiten; erst dann geht es weiter. Die API des Inventarsystems erlaubt zehn Anfragen pro Sekunde, daher fragt ein einziges Skript ab und verteilt die Ergebnisse an die Läufer.

## 12.6 BCDR für Skripte

*BCDR* steht für Business Continuity (den Betrieb aufrechterhalten) und Disaster Recovery (nach einem Ausfall wiederherstellen). Für Skripte klingt das übertrieben, bis man sich fragt, was geschieht, wenn das Git-Repository verloren geht, der einzige Rechner mit den Wartungsskripten ausfällt, der CI-Dienst tagelang nicht erreichbar ist oder die Zugangsdaten verschwinden. Eine Organisation, deren Betrieb von Skripten abhängt (Backups, Deployments, Wartung), muss diese Skripte und alles, was sie zum Laufen brauchen, wiederherstellen können. Zwei Kennzahlen strukturieren die Planung: das *Recovery Time Objective* (RTO, wie lange darf die Wiederherstellung dauern) und das *Recovery Point Objective* (RPO, wie viel Datenstand darf verloren gehen).

Es gibt vier Bestandteile, die wiederherstellbar sein müssen. Erstens die *Repositories*: Der Quelltext liegt im Idealfall an mindestens zwei voneinander unabhängigen Orten (zum Beispiel Hosting-Dienst und regelmäßiges Abbild im eigenen Speicher). Git unterstützt das mit `git clone --mirror` (vollständige Kopie aller Referenzen) und `git bundle` (eine einzelne Datei, die ein ganzes Repository enthält und sich prüfen lässt). Zweitens die *Artefakte*: Gebaute Pakete, Images und Release-Dateien liegen in einer Ablage mit Aufbewahrungsfrist und Prüfsummen; ohne sie lässt sich eine alte Version nicht ausrollen. Drittens die *Secrets*: Sie gehören nicht ins Repository, aber ihre *Wiederbeschaffung* muss beschrieben sein. Man dokumentiert, *welche* Geheimnisse es gibt, wer sie ausstellen kann und wie sie neu erzeugt oder rotiert werden (Modul 11.5) - ohne die Werte selbst aufzuführen. Viertens das *Wissen*: Runbooks (Modul 9.8) für Ausfall und Wiederherstellung, die auch dann zugänglich sind, wenn das Wiki ausgefallen ist (ausgedruckt oder in einer unabhängigen Ablage).

Der wichtigste Satz zu Sicherungen lautet: *Ein Backup, dessen Wiederherstellung nie geübt wurde, ist nur eine Hoffnung.* Deshalb prüft man regelmäßig, ob die Sicherung vollständig und lesbar ist, und übt die Wiederherstellung in einer getrennten Umgebung. Das Beispiel zeigt die Mechanik für ein Repository: Es erzeugt ein Bündel, prüft es, stellt es in einem frischen Verzeichnis wieder her, vergleicht den Stand und schreibt ein Manifest mit Prüfsumme.

```bash
#!/usr/bin/env bash
# bcdr-repo.sh - Repository sichern (git bundle), pruefen, Wiederherstellung testen, Manifest schreiben
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
export GIT_AUTHOR_NAME=Test GIT_AUTHOR_EMAIL=t@example.org GIT_COMMITTER_NAME=Test GIT_COMMITTER_EMAIL=t@example.org
export GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null

# ---- Testrepository mit etwas Historie
git init -q -b main "$t/original"
( cd "$t/original"
  echo "v1" > backup.sh; git add . ; git commit -qm "Version 1"
  echo "v2" > backup.sh; git commit -qam "Version 2"; git tag -a v2 -m "Release 2" )

sichere_repo() {                                  # sichere_repo QUELLE ZIELVERZ
    local quelle="$1" ziel="$2" stempel bundle
    mkdir -p -- "$ziel"
    stempel="$(date -u +%Y%m%dT%H%M%SZ)"; bundle="$ziel/repo-$stempel.bundle"
    git -C "$quelle" bundle create "$bundle" --all >/dev/null 2>&1
    git -C "$quelle" bundle verify "$bundle" >/dev/null 2>&1 || { echo "Bundle ungueltig: $bundle" >&2; return 1; }
    ( cd "$ziel" && sha256sum "${bundle##*/}" >> MANIFEST.sha256 )
    git -C "$quelle" rev-parse HEAD > "$bundle.head"
    echo "$bundle"
}

teste_restore() {                                 # teste_restore BUNDLE : stellt wieder her und vergleicht den HEAD
    local bundle="$1" ziel
    ziel="$(mktemp -d "$t/restore.XXXXXX")"
    git clone -q "$bundle" "$ziel/r" 2>/dev/null
    [[ "$(git -C "$ziel/r" rev-parse HEAD)" == "$(cat "$bundle.head")" ]] \
        && echo "Restore-Test OK: HEAD stimmt (${bundle##*/})" \
        || { echo "Restore-Test FEHLGESCHLAGEN" >&2; return 1; }
    git -C "$ziel/r" tag --list | sed 's/^/   Tag im Restore: /'
}

bundle="$(sichere_repo "$t/original" "$t/sicherung")"
echo "Gesichert: ${bundle##*/}"
echo "--- Manifest"; cat "$t/sicherung/MANIFEST.sha256" | sed 's/^\([0-9a-f]\{12\}\)[0-9a-f]*/\1.../'
echo "--- Integritaet der Sicherungsdatei pruefen"
( cd "$t/sicherung" && sha256sum -c --quiet MANIFEST.sha256 && echo "Pruefsumme stimmt" )
teste_restore "$bundle"
echo "--- Beschaedigte Sicherung wird erkannt"
head -c 200 "$bundle" > "$t/kaputt.bundle"
erwartet="$(awk '{print $1}' "$t/sicherung/MANIFEST.sha256")"
ist="$(sha256sum "$t/kaputt.bundle" | awk '{print $1}')"
[[ $ist == "$erwartet" ]] && echo "nicht erkannt!" || echo "Pruefsumme weicht ab: Beschaedigung erkannt"
git clone -q "$t/kaputt.bundle" "$t/kaputt-restore" >/dev/null 2>&1 && echo "Klon trotzdem moeglich?!" || echo "Restore der beschaedigten Datei scheitert"
```

Die Funktion `sichere_repo` erzeugt mit `git bundle create --all` eine einzelne Datei mit allen Branches und Tags und prüft sie sofort mit `git bundle verify` (der Befehl muss in einem Repository laufen, deshalb mit `git -C`). Das Manifest ist eine Liste von Prüfsummen, die mit `sha256sum -c` später die Unversehrtheit belegt; so erkennt man schleichende Beschädigung der Sicherung (Bit-Fäule, abgebrochene Übertragung). Neben dem Bündel liegt eine kleine Datei mit dem erwarteten HEAD-Commit, gegen die der Restore-Test vergleicht. `teste_restore` klont das Bündel in ein frisches Verzeichnis und vergleicht den Stand, sodass nicht nur “die Datei existiert”, sondern “der Inhalt ist wiederherstellbar” belegt wird. Am Ende wird absichtlich eine abgeschnittene Kopie erzeugt. Dabei zeigt sich eine wichtige Eigenheit: `git bundle verify` prüft vor allem den Kopf der Datei und die Voraussetzungen, nicht die gesamte Datei, und erkennt eine abgeschnittene Datei daher nicht zuverlässig (im Test des Autors wurde sie nicht erkannt). Verlässlich sind die Prüfsumme aus dem Manifest und der echte Restore-Test; beides schlägt hier an. Deshalb gehören beide Maßnahmen zusammen. Die Umgebungsvariablen am Anfang machen das Beispiel unabhängig von der Git-Konfiguration des Benutzers.

Typische Fehler: Erstens liegt die einzige Kopie bei einem einzigen Anbieter oder auf einem einzigen Rechner. Zweitens sichert man, ohne je eine Wiederherstellung zu testen; erst im Ernstfall zeigt sich, dass die Sicherung leer, beschädigt oder ohne Schlüssel unbrauchbar ist. Drittens speichert man Secrets im Repository und damit in allen Kopien, oder umgekehrt hat man keine Anleitung, wie man sie neu beschafft. Viertens liegen Backups im selben Konto und mit denselben Rechten wie die Produktion, sodass ein Angreifer oder ein Versehen beides löscht; Sicherungen brauchen getrennte, möglichst unveränderbare Ablagen. Fünftens fehlen RTO und RPO, und niemand weiß, ob die Wiederherstellung in Stunden oder Tagen gelingen muss. Sechstens liegen die Wiederherstellungsanleitungen im Wiki, das genau dann nicht erreichbar ist, wenn man sie braucht. Siebtens hängen Skripte von einem einzigen Rechner oder Runner ab, dessen Konfiguration nirgends beschrieben ist (Modul 11.6, 11.8).

```bash
# falsch: Sicherung ohne Pruefung, ohne Manifest, Restore nie getestet
tar czf /mnt/backup/skripte.tgz /srv/skripte

# richtig: pruefbar, mit Pruefsumme und regelmaessigem Restore-Test
git bundle create "$ziel/repo.bundle" --all && git bundle verify "$ziel/repo.bundle"   # im Repository ausfuehren
( cd "$ziel" && sha256sum repo.bundle >> MANIFEST.sha256 && sha256sum -c MANIFEST.sha256 )
git clone "$ziel/repo.bundle" "$(mktemp -d)/restore" && echo "Restore ok"
```

> **Merke**
>
> BCDR für Skripte: Repository (Mirror oder Bundle an zweitem Ort), Artefakte (mit Prüfsumme und Aufbewahrung), Secrets (Beschaffungswege dokumentiert, Werte nie im Repo) und Wissen (Runbooks unabhängig vom Wiki) wiederherstellbar halten. RTO und RPO festlegen, Sicherungen prüfen und die Wiederherstellung regelmäßig üben. Backups getrennt von der Produktion ablegen.

> **Praxisbeispiel**
>
> Ein nächtlicher Job erzeugt ein Git-Bundle aller Betriebs-Repositories, prüft es, legt es in eine unveränderbare Ablage und führt einmal im Monat einen Restore-Test in einer sauberen Umgebung durch. Als der Hosting-Dienst eine Stunde nicht erreichbar war, lief der Notfallbetrieb mit dem letzten Bündel weiter, und der Runbook-Schritt “Repository aus Bundle bereitstellen” war geübt.

## 12.7 Change Management

Jede Änderung an einem Skript, das produktiv läuft, ist ein Risiko: Ein Tippfehler im Aufräumskript kann Daten löschen, eine neue Option das Verhalten bestehender Aufrufer brechen. *Change Management* ist der geordnete Umgang mit Änderungen: Sie werden *beantragt oder begründet*, *geprüft*, *freigegeben*, *kontrolliert eingeführt* und sind *nachvollziehbar*, sodass man später klären kann, wer was wann und warum geändert hat, und sie im Zweifel zurücknehmen kann. Der Aufwand soll dem Risiko entsprechen: Eine Textkorrektur in einem Hilfsskript braucht weniger Zeremonie als eine Änderung am Backup-Skript der Produktion.

Die technische Grundlage ist die *Versionskontrolle*. Jede Änderung ist ein *Commit* mit einer verständlichen Meldung, der Zweck steht im Text (*warum*, nicht nur *was*). Änderungen laufen über *Branches* und *Pull Requests*, in denen mindestens eine zweite Person liest (Review-Prinzip der vier Augen). Automatische Prüfungen (ShellCheck, Tests, Policy-Checks, Einheit 11.7) laufen vor der Freigabe. Auf dem Hauptzweig sind direkte Änderungen gesperrt. Freigegebene Stände erhalten ein *Versions-Tag* (zum Beispiel `v1.4.0`) nach einem festen Schema, ein *Änderungsprotokoll* (Changelog) fasst für Anwender zusammen, was sich geändert hat, und Deployments verwenden *genau* diese Tags (Einheit 10.5), nie den beweglichen Hauptzweig.

Für die Nachvollziehbarkeit eignen sich Git-*Trailer*: genormte Zeilen am Ende der Commit-Meldung, etwa `Reviewed-by: Name` oder `Change-Ticket: OPS-412`. Sie lassen sich maschinell auslesen (`git interpret-trailers`, `git log --format`), sodass eine Prüfung vor dem Release feststellen kann, ob jede Änderung seit dem letzten Tag geprüft und einem Vorgang zugeordnet wurde. Der *Rollback* ist Teil der Planung, nicht der Notlösung: Man hält das vorige Release bereit (Einheit 10.5), kennt den Rückweg (`git checkout v1.3.0` oder Wechsel des Symlinks) und weiß, ob Daten-Änderungen den Rückschritt erschweren. Bei kritischen Änderungen legt man ein *Änderungsfenster* fest, informiert Betroffene und benennt, wer im Fehlerfall entscheidet.

Das Beispiel prüft, ob jede Änderung zwischen zwei Tags die Pflichtangaben trägt, und erzeugt aus den Commits einen Changelog-Eintrag. Es arbeitet in einem frischen Test-Repository.

```bash
#!/usr/bin/env bash
# freigabe-pruefen.sh - Release-Pruefung: Trailer vorhanden? Changelog erzeugen. (Demo im Testrepository)
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
export GIT_AUTHOR_NAME=Dev GIT_AUTHOR_EMAIL=dev@example.org GIT_COMMITTER_NAME=Dev GIT_COMMITTER_EMAIL=dev@example.org
export GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null

git init -q -b main "$t/repo"; cd "$t/repo"
echo 1 > skript.sh; git add .; git commit -qm "Erste Version"; git tag -a v1.0.0 -m "Release 1.0.0"
echo 2 > skript.sh; git commit -qam "Rotation der Logs korrigieren

Change-Ticket: OPS-101
Reviewed-by: Anna"
echo 3 > skript.sh; git commit -qam "Neue Option --trocken

Change-Ticket: OPS-102"
echo 4 > skript.sh; git commit -qam "Schnell-Fix ohne Pruefung"

pruefe_release() {                              # pruefe_release VORHERIGES_TAG
    local seit="$1" fehler=0 h betreff ticket review
    while read -r h; do
        betreff="$(git log -1 --format=%s "$h")"
        ticket="$(git log -1 --format=%B "$h" | git interpret-trailers --parse | sed -n 's/^Change-Ticket: *//p')"
        review="$(git log -1 --format=%B "$h" | git interpret-trailers --parse | sed -n 's/^Reviewed-by: *//p')"
        [[ -n $ticket ]] || { echo "[FEHLT] ${h:0:7} '$betreff': Change-Ticket"; fehler=$((fehler + 1)); }
        [[ -n $review ]] || { echo "[FEHLT] ${h:0:7} '$betreff': Reviewed-by"; fehler=$((fehler + 1)); }
    done < <(git rev-list --reverse "$seit"..HEAD)
    (( fehler == 0 ))
}

changelog() {                                   # changelog VORHERIGES_TAG NEUE_VERSION
    printf '## %s - %s\n' "$2" "$(date +%F)"
    git log --reverse --format='- %s' "$1"..HEAD
}

echo "== Pruefung seit v1.0.0"
if pruefe_release v1.0.0; then echo "bereit fuer Release"; else echo "Release blockiert: Angaben fehlen" >&2; fi
echo "== Changelog-Entwurf"
changelog v1.0.0 v1.1.0
echo "== Rollback-Weg"
echo "Vorheriger Stand: $(git describe --tags --abbrev=0) -> git checkout v1.0.0 (oder Symlink auf altes Release setzen)"
```

Das Test-Repository enthält drei Änderungen nach dem Tag `v1.0.0`: eine vollständige, eine ohne Review und eine ohne Ticket und Review. Die Funktion `pruefe_release` geht alle Commits seit dem Tag durch (`git rev-list --reverse`) und liest die Trailer mit `git interpret-trailers --parse` aus, einem Befehl, der genau die Trailer-Zeilen am Ende einer Meldung erkennt. Fehlende Angaben werden gezählt und gemeldet; die Funktion liefert nur dann Erfolg, wenn keine fehlt, und eignet sich daher als Sperre in der Release-Pipeline. `changelog` erzeugt aus den Betreffzeilen einen Eintrag für die neue Version. Die letzte Zeile zeigt den Rückweg zum vorigen Tag. In einem echten Ablauf erzwingt die Plattform das Review über Schutzregeln für den Hauptzweig; der Trailer `Reviewed-by` ist eine einfache, zusätzlich prüfbare Spur, aber allein kein Beweis, da man ihn schreiben kann, ohne dass jemand geprüft hat. Verbindlich ist die Freigabe in der Plattform.

Typische Fehler: Erstens wird direkt auf dem Hauptzweig der Produktion gearbeitet (“nur kurz gefixt”). Zweitens bestehen Commit-Meldungen aus “fix” und “update”; später weiß niemand, warum. Drittens deployt man den beweglichen Hauptzweig statt eines Tags, und zwei Server haben unterschiedliche Stände. Viertens fehlt ein Rollback-Plan, oder er wurde nie erprobt. Fünftens verlässt man sich auf Review ohne automatische Prüfung und lässt Menschen Formfehler suchen. Sechstens gibt es Notfalländerungen ohne Nachbereitung; sie werden nach dem Eingriff nicht dokumentiert und nachgeprüft. Siebtens überschreibt man veröffentlichte Historie (`push -{}-force` auf freigegebene Zweige), sodass die Nachvollziehbarkeit verloren geht.

```bash
# falsch: direkt auf main, unklare Meldung, beweglicher Stand wird ausgerollt
git commit -am "fix" && git push origin main
ssh prod 'cd /srv/skripte && git pull'

# richtig: Branch, Pull Request mit Review, Tag, Rollout genau dieses Tags
git switch -c fix/log-rotation
git commit -am "Rotation der Logs korrigieren" -m "Change-Ticket: OPS-101"
git tag -a v1.1.0 -m "Release 1.1.0" && git push origin v1.1.0
./deploy.sh /srv/app build/ v1.1.0
```

> **Merke**
>
> Änderungen begründen, prüfen (vier Augen plus Automatik), freigeben, als Tag ausrollen, dokumentieren (Commit-Meldung mit *Warum*, Trailer, Changelog) und Rollback vorab planen. Aufwand nach Risiko staffeln. Notfalländerungen nachträglich dokumentieren. Verbindlichkeit liefert die Plattform (Schutzregeln), nicht der Hinweis in einer Zeile.

> **Praxisbeispiel**
>
> Änderungen am Backup-Skript laufen über einen Pull Request mit Pflicht-Review und Ticketnummer; vor dem Release prüft die Pipeline die Trailer und erzeugt den Changelog. Als eine Änderung die Aufbewahrungsfrist falsch interpretierte, setzte das Team in Minuten das vorige Tag ein und konnte anhand des Tickets nachvollziehen, wer die Änderung beantragt und geprüft hatte.

## 12.8 Betriebsübergabe

Ein Skript ist erst dann Teil des Betriebs, wenn jemand anderes als sein Autor es betreiben, verstehen und reparieren kann. Die *Betriebsübergabe* (Handover) ist der Übergang von Entwicklung zu Betrieb, aber auch von einem Team zum nächsten oder von einer Person zur Vertretung. Ohne sie bleibt Wissen in einem Kopf; fällt dieser aus, wird das Skript zum Risiko. Ziel ist *Supportfähigkeit*: Eine Person, die das Skript nicht geschrieben hat, kann innerhalb vertretbarer Zeit Störungen einordnen, beheben oder eskalieren.

Eine tragfähige Übergabe klärt vier Dinge. Erstens *Dokumentation*: ein README mit Zweck, Voraussetzungen, Aufruf und Beispielen, Konfiguration (alle Parameter und Standardwerte), Abhängigkeiten, Ein- und Ausgaben, Exit-Codes, Logorte, bekannte Einschränkungen und ein Runbook für die häufigsten Störungen (Modul 9.8). Zweitens *Verantwortlichkeiten*: Wer ist Owner, wer vertritt, wer bekommt Alarme, wer entscheidet über Änderungen, wohin eskaliert man? Drittens *Schulung*: Wer den Betrieb übernimmt, führt das Skript einmal unter Anleitung aus, löst eine typische Störung im Testsystem und kennt die Rückwege. Viertens *Betriebsbereitschaft*: Monitoring und Alarmierung sind eingerichtet (9.4), Tests laufen, Wiederherstellung ist beschrieben (12.6), Zugriff und Rechte sind für die Betreuenden vorhanden, und Geheimnisse sind verwaltet.

Man kann einen Teil dieser Prüfung automatisieren: eine *Übergabe-Checkliste* als Skript, das für ein Skript beziehungsweise ein Verzeichnis feststellt, ob die formalen Voraussetzungen erfüllt sind. Sie ersetzt nicht das Gespräch, sorgt aber dafür, dass nichts Offensichtliches fehlt, und macht den Stand objektiv sichtbar. Wichtig ist, nach der Übergabe eine *Schonfrist* mit Rückfragemöglichkeit zu vereinbaren und die Dokumentation anhand der Fragen zu verbessern.

```bash
#!/usr/bin/env bash
# uebergabe-check.sh VERZ  - formale Voraussetzungen fuer die Betriebsuebergabe pruefen
set -uo pipefail
verz="${1:-}"
if [[ -z $verz ]]; then                          # Demo-Projekt anlegen
    t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT; verz="$t/projekt"; mkdir -p "$verz/bin" "$verz/tests" "$verz/docs"
    printf '# Backup-Werkzeug\n\n## Zweck\nSichert Datenbanken.\n\n## Aufruf\nbackup.sh ZIEL\n\n## Konfiguration\nAPP_RETENTION\n' > "$verz/README.md"
    printf '#!/usr/bin/env bash\n# owner: team-betrieb\n# vertretung: team-plattform\nset -euo pipefail\n[[ ${1:-} == -h ]] && { echo "Aufruf: backup.sh ZIEL"; exit 0; }\n' > "$verz/bin/backup.sh"
    chmod +x "$verz/bin/backup.sh"
    printf '# Runbook\n\n## Backup schlaegt fehl\nLog pruefen, Platz pruefen.\n' > "$verz/docs/runbook.md"
fi
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }

ok=0; fehlt=0
pruefe() {                                       # pruefe BESCHREIBUNG BEDINGUNGSBEFEHL...
    local text="$1"; shift
    if "$@" >/dev/null 2>&1; then printf '[ok   ] %s\n' "$text"; ok=$((ok + 1))
    else printf '[FEHLT] %s\n' "$text"; fehlt=$((fehlt + 1)); fi
}
hat_abschnitt() { grep -Eqi "^#+[[:space:]]+$2" "$1"; }          # hat_abschnitt DATEI NAME

pruefe "README.md vorhanden"                         test -s "$verz/README.md"
pruefe "README: Abschnitt Zweck"                     hat_abschnitt "$verz/README.md" "Zweck"
pruefe "README: Abschnitt Aufruf"                    hat_abschnitt "$verz/README.md" "Aufruf"
pruefe "README: Abschnitt Konfiguration"             hat_abschnitt "$verz/README.md" "Konfiguration"
pruefe "README: Abschnitt Exit-Codes"                hat_abschnitt "$verz/README.md" "Exit-Codes"
pruefe "Runbook vorhanden (docs/runbook.md)"         test -s "$verz/docs/runbook.md"
pruefe "Tests vorhanden (tests/ nicht leer)"         bash -c '[[ -n "$(ls -A "$1/tests" 2>/dev/null)" ]]' _ "$verz"
for f in "$verz"/bin/*.sh; do
    [[ -e $f ]] || continue
    n="${f##*/}"
    pruefe "$n: Owner im Kopf"                       grep -Eq '^# owner: *[^ ]' "$f"
    pruefe "$n: Vertretung im Kopf"                  grep -Eq '^# vertretung: *[^ ]' "$f"
    pruefe "$n: strenger Modus"                      grep -Eq '^set -[a-z]*e[a-z]*u|^set -euo pipefail' "$f"
    pruefe "$n: Hilfetext mit -h"                    bash -c 'bash "$1" -h 2>&1 | grep -qi aufruf' _ "$f"
    pruefe "$n: ausfuehrbar"                         test -x "$f"
done
echo "Ergebnis: $ok erfuellt, $fehlt fehlen"
(( fehlt == 0 ))
```

Die Funktion `pruefe` nimmt eine Beschreibung und einen Befehl, der Erfolg oder Misserfolg liefert, und druckt für jede Voraussetzung eine Zeile. Damit lassen sich beliebige Prüfungen einheitlich aufnehmen: Dateien vorhanden, Abschnitte im README (mit `grep -Ei` auf Überschriften), Metadaten im Skriptkopf, strenger Modus, Hilfetext (das Skript wird mit `-h` aufgerufen und seine Ausgabe geprüft) und Ausführrecht. Die Schleife wiederholt die skriptbezogenen Prüfungen für jedes Skript in `bin/`. In der Demo fehlen bewusst der Abschnitt zu Exit-Codes und die Tests; das Skript meldet sie und endet mit Exit-Code 1, sodass die Übergabe als nicht vollständig gilt. Beachten Sie, dass die Prüfung den *Inhalt* nicht beurteilt: Ein README mit leeren Überschriften besteht den formalen Test. Gute Übergaben ergänzen ihn deshalb um ein Gespräch und eine Probe (die übernehmende Person führt eine Störungsbehebung nach Runbook aus).

Typische Fehler: Erstens wird dokumentiert, *was* das Skript tut (das sieht man im Code), aber nicht, *warum* und *was im Fehlerfall zu tun ist*. Zweitens ist ein einzelner Name der Owner, ohne Vertretung. Drittens liegt die Dokumentation außerhalb des Repositories und veraltet unbemerkt; sie gehört neben den Code und wird mit ihm geändert. Viertens sind Zugänge der Betreuenden nicht eingerichtet, und die erste Störung beginnt mit einer Berechtigungsanfrage. Fünftens gibt es keine Schulung, sondern eine Wiki-Seite. Sechstens endet die Verantwortung der Entwickelnden mit dem Tag der Übergabe, und Rückfragen laufen ins Leere. Siebtens fehlen Alarme oder gehen an Personen, die nichts davon wissen.

```bash
# falsch: Uebergabe per E-Mail "laeuft schon, frag mich wenn was ist"
echo "Das Skript liegt auf srv01, startet nachts." | mail team@example.org

# richtig: Pruefliste als Gate, Probelauf einer Stoerung nach Runbook, Schonfrist vereinbart
./uebergabe-check.sh projekt/ && echo "formal uebergabefaehig"
# anschliessend: Stoerung im Testsystem nach docs/runbook.md loesen lassen und Rueckfragezeit festhalten
```

> **Merke**
>
> Übergabe = Dokumentation (README mit Zweck, Aufruf, Konfiguration, Exit-Codes, Runbook), klare Verantwortung (Owner, Vertretung, Eskalation), Schulung mit Probestörung und Betriebsbereitschaft (Monitoring, Tests, Zugriff, Restore). Eine automatische Prüfliste sichert das Formale, ersetzt aber nicht Gespräch und Probe. Dokumentation liegt beim Code.

> **Praxisbeispiel**
>
> Vor der Übergabe eines Wartungsskripts an den Betrieb läuft die Prüfliste; sie zeigt, dass der Abschnitt zu Exit-Codes fehlt. Nach der Ergänzung löst eine Kollegin im Testsystem die zwei häufigsten Störungen anhand des Runbooks. Die Entwicklerin bleibt für vier Wochen als Ansprechpartnerin eingetragen; die Fragen dieser Zeit fließen in das README ein.

## 12.9 Kontinuierliche Verbesserung

Automation ist kein Zustand, der erreicht und dann abgehakt wird. Systeme ändern sich, Anforderungen wachsen, Fehler zeigen Schwächen. *Kontinuierliche Verbesserung* bedeutet, regelmäßig zu messen, was passiert, die Ursachen von Problemen zu beseitigen und die Automation in sinnvollen Schritten reifer zu machen. Zwei Quellen liefern die Grundlage: *Fehlerdaten* (welche Skripte scheitern wie oft und warum) und *Laufzeitdaten* (wie lange laufen sie, wird es schlechter, wo gibt es Ausreißer). Beide stehen in den Logs und Metriken aus Modul 9, sofern jeder Lauf Ergebnis und Dauer mit einer Kennung festhält.

Aus den Rohdaten bildet man wenige *Kennzahlen*. Die *Erfolgsquote* je Skript (Anteil erfolgreicher Läufe), die *typische Laufzeit* (Median) und die *Ausreißer-Laufzeit* (zum Beispiel das 95. Perzentil: 95 % aller Läufe sind schneller), die *Häufigkeit von Wiederholungen und manuellen Eingriffen* sowie bei Störungen die Zeit bis zur Behebung. Wichtig ist der *Trend*: Eine Erfolgsquote von 97 % ist unbedenklich, wenn sie stabil ist, und ein Warnzeichen, wenn sie vor vier Wochen noch 100 % war. Perzentile sind aussagekräftiger als Mittelwerte, weil wenige sehr lange Läufe den Mittelwert verzerren und das Problem verdecken.

Aus den Kennzahlen entsteht eine *Roadmap*, die sich an einem einfachen Reifegradmodell orientieren kann: Stufe 1 “Einzelskripte” (keine Standards, kein Inventar), Stufe 2 “Gemeinsame Regeln” (Vorlage, Styleguide, Prüfungen), Stufe 3 “Betriebsfähig” (Monitoring, Runbooks, Restore getestet, Owner und Vertretung), Stufe 4 “Gesteuert” (Kennzahlen, Change Management, Skalierungsregeln) und Stufe 5 “Optimierend” (regelmäßige Auswertung, Ursachenbeseitigung, Wegwerfen unnötiger Automation). Die Stufen sind ein Denkgerüst, keine Norm; maßgeblich ist, welche Lücke für die Organisation das größte Risiko oder den größten Nutzen bedeutet. Verbesserungen werden klein gehalten, einzeln geplant und anhand der Kennzahlen überprüft, ob sie wirken. Nach größeren Störungen führt man eine *Nachbetrachtung* (Post-Mortem) ohne Schuldzuweisung durch: Was ist passiert, warum konnte es passieren, was verhindert die Wiederholung? Maßnahmen bekommen Verantwortliche und Termine.

Das Beispiel wertet ein Lauf-Log im CSV-Format aus und berechnet je Skript Läufe, Erfolgsquote, Median und 95. Perzentil der Laufzeit sowie die Zahl der Fehler im letzten Zeitraum im Vergleich zum davor. Die Daten sind für die Demo erzeugt.

```bash
#!/usr/bin/env bash
# kvp-auswertung.sh [LOGDATEI]  - Erfolgsquote, Median und P95 der Laufzeit, Trend je Skript
set -euo pipefail
log="${1:-}"
if [[ -z $log ]]; then                           # Demo-Daten: datum,skript,ergebnis,dauer_s
    t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT; log="$t/laeufe.csv"
    {
      for tag in $(seq 1 20); do
          d="$(date -d "2026-10-$((tag + 9))" +%F)"
          echo "$d,backup,ok,$((300 + tag * 2))"
          if (( tag % 5 == 0 )); then echo "$d,aufraeumen,fehler,12"; else echo "$d,aufraeumen,ok,$((10 + tag % 4))"; fi
          if (( tag >= 15 )); then echo "$d,sync,fehler,$((60 + tag))"; else echo "$d,sync,ok,$((40 + tag % 3))"; fi
      done
      echo "2026-10-30,backup,ok,1800"           # ein Ausreisser
    } > "$log"
fi
[[ -r $log ]] || { echo "Log nicht lesbar: $log" >&2; exit 2; }

sort -t, -k2,2 -k4,4n "$log" | awk -F, '
function perz(a, n, p,   i) { i = int(n * p + 0.9999); if (i < 1) i = 1; return a[i] }
{
    s = $2
    if (s != vorher && vorher != "") { ausgabe() }
    if (s != vorher) { n = 0; ok = 0; vorher = s; delete d }
    n++; d[n] = $4 + 0
    if ($3 == "ok") ok++
}
function ausgabe(   i, med) {
    med = (n % 2) ? d[(n + 1) / 2] : (d[n / 2] + d[n / 2 + 1]) / 2
    printf "%-10s Laeufe:%3d  Erfolg:%6.1f%%  Median:%7.1fs  P95:%5ds  Max:%5ds\n", vorher, n, 100 * ok / n, med, perz(d, n, 0.95), d[n]
}
END { if (vorher != "") ausgabe() }'

echo "--- Fehler je Skript: erste gegen zweite Haelfte des Zeitraums"
sort -t, -k1,1 "$log" | awk -F, '
{ tage[$1] = 1 }
{ zeile[NR] = $0 }
END {
    m = 0; for (t in tage) m++
    # Mitte des Zeitraums nach sortierten Datumswerten bestimmen
    n = asorti(tage, ds); mitte = ds[int(n / 2)]
    for (i = 1; i <= NR; i++) {
        split(zeile[i], f, ",")
        if (f[3] == "fehler") { if (f[1] <= mitte) alt[f[2]]++; else neu[f[2]]++; sk[f[2]] = 1 }
    }
    for (s in sk) printf "%-10s fruehere Haelfte: %d  spaetere Haelfte: %d%s\n", s, alt[s] + 0, neu[s] + 0, (neu[s] > alt[s]) ? "   <- Verschlechterung" : ""
}' 2>/dev/null || echo "(Trendteil benoetigt gawk)"
```

Das erste `awk`-Programm verarbeitet die nach Skriptname und Laufzeit sortierte Datei gruppenweise: Beim Wechsel des Skriptnamens gibt es die Kennzahlen der vorigen Gruppe aus und beginnt neu zu zählen. Die Laufzeiten stehen dank der Sortierung bereits aufsteigend in einem Feld, sodass das Maximum der letzte Wert ist und der Median (mittlerer Wert, bei gerader Anzahl Mittelwert der beiden mittleren) und das 95. Perzentil (Wert an Position 0,95 mal Anzahl, aufgerundet) direkt abgelesen werden. Man sieht in der Ausgabe, dass der einzelne Ausreißer von 1800 Sekunden bei `backup` weder den Median noch das 95. Perzentil bewegt (beide sind robust gegen einzelne Ausreißer), im Maximum aber auftaucht. Deshalb wertet man Median, Perzentil *und* Maximum gemeinsam aus: Das Perzentil beschreibt das typische Verhalten, das Maximum zeigt, ob es Ausreißer gibt, die eine Untersuchung verdienen. Der zweite Teil vergleicht die Zahl der Fehler in der ersten und zweiten Zeitraumhälfte und kennzeichnet Verschlechterungen; er verwendet `asorti`, eine Erweiterung von GNU `awk`, und meldet ersatzweise einen Hinweis, wenn nur ein anderes `awk` vorhanden ist (siehe 11.8).

Typische Fehler: Erstens misst man nichts und verbessert nach Gefühl. Zweitens wertet man nur Mittelwerte aus; sie werden von wenigen sehr langen Läufen verzerrt, und man erkennt nicht, was typisch ist und was Ausreißer sind. Drittens betrachtet man nur den aktuellen Stand statt den Trend. Viertens sammelt man Daten, die niemand liest; Auswertung braucht einen Termin und eine verantwortliche Person. Fünftens bearbeitet man Symptome (Neustart, manuelles Eingreifen) und nie die Ursache; die Fehlerdaten zeigen, welche Fehler sich wiederholen. Sechstens führt man nach Störungen Schuldzuweisungen statt Ursachenanalyse durch, sodass beim nächsten Mal niemand ehrlich berichtet. Siebtens baut man immer mehr Automation, ohne unnötige abzuschalten; Skripte ohne Nutzer, ohne Läufe oder ohne Owner gehören aus dem Betrieb genommen (siehe Inventar, 12.1).

```bash
# falsch: nur der Mittelwert, nur heute
awk -F, '{s += $4} END {print "Durchschnitt:", s/NR}' laeufe.csv

# richtig: Median und P95 je Skript, plus Trend gegen die Vorperiode
sort -t, -k2,2 -k4,4n laeufe.csv | awk -F, '$3 == "fehler" {f[$2]++} END {for (s in f) print s, f[s], "Fehler"}'
```

> **Merke**
>
> Messen, auswerten, verbessern: Erfolgsquote, Median und P95 der Laufzeit, Wiederholungen, Behebungszeit - jeweils mit Trend. Regelmäßiger Termin mit Verantwortlichen, kleine Verbesserungsschritte, Nachbetrachtung ohne Schuldzuweisung, Roadmap nach Reifegrad (Einzelskripte, gemeinsame Regeln, betriebsfähig, gesteuert, optimierend). Unnötige Automation abschalten.

> **Praxisbeispiel**
>
> Einmal im Monat wertet das Team die Lauf-Logs aus. Die Auswertung zeigt, dass ein Synchronisationsskript seit zwei Wochen häufiger scheitert; die Ursache ist ein Rate Limit, das nach einer Erweiterung der Zielmenge erreicht wird (siehe 12.5). Die Maßnahme (Drosselung und Backoff) wird in die Roadmap aufgenommen, und der nächste Monatsbericht zeigt, ob sich die Quote erholt hat.

## 12.10 Übungsaufgaben zu Modul 12

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Nennen Sie drei fachliche und drei technische Merkmale, die ein Skript-Inventar erfassen sollte. Warum reicht es nicht, nur das Repository zu durchsuchen?

**Aufgabe 2.** Erklären Sie das Vorgehen “Golden Master” bei der Migration eines Altskripts und nennen Sie je ein Kriterium, das für Bash, für Python und für Ansible als Zielplattform spricht.

**Aufgabe 3.** Erklären Sie RTO und RPO und beschreiben Sie, warum ein nie getesteter Restore keine Sicherung ist. Welche vier Bestandteile müssen für Skripte wiederherstellbar sein?

**Aufgabe 4.** Vergleichen Sie zentrale und lokale Ausführung von Automation: Nennen Sie je zwei Vor- und Nachteile. Was ist ein gestaffelter Rollout, und weshalb prüft man zwischen den Stufen?

**Aufgabe 5.** Schreiben Sie `skript-liste.sh VERZ`, das alle Dateien mit Bash- oder sh-Shebang (unabhängig von der Endung) in einem Verzeichnisbaum findet und je Datei Pfad, Zeilenzahl und Owner (aus `# owner:`) als CSV ausgibt; Dateien ohne Owner werden mit `-` gekennzeichnet. Dateien mit Leerzeichen im Namen müssen funktionieren.

**Aufgabe 6.** Schreiben Sie `break-even.sh MANUELL_MIN PRO_JAHR ENTWICKLUNG_H STUNDENSATZ`, das aus vier Argumenten den Break-even in Monaten berechnet, alle Argumente als positive Zahlen prüft und bei fehlendem Nutzen (Entwicklungsaufwand wird nie eingespielt) mit Exit-Code 1 und klarer Meldung endet. Aufsicht und Pflege sollen vereinfachend mit 10 % der manuellen Zeit angesetzt werden.

**Aufgabe 7.** Schreiben Sie `migrations-test.sh ALT NEU TESTVERZ`, das ein altes und ein neues Skript auf demselben Testverzeichnis ausführt und Ausgabe sowie Exit-Code vergleicht. Bei Gleichheit endet es mit 0 und der Meldung “identisch”, bei Abweichung mit 1 und einem Diff. Das Testverzeichnis wird nicht verändert (Prüfsumme vorher und nachher).

**Aufgabe 8.** Schreiben Sie `sicherung-pruefen.sh SICHERUNGSVERZ`, das für ein Verzeichnis mit `.bundle`-Dateien und einer Datei `MANIFEST.sha256` prüft: (a) Manifest vorhanden, (b) alle Prüfsummen stimmen, (c) die neueste Sicherung ist jünger als eine konfigurierbare Zahl Stunden (Standard 26), (d) jede Datei steht im Manifest. Jedes Ergebnis erscheint als Zeile `OK` oder `FEHLER`, der Exit-Code ist 1 bei mindestens einem Fehler.

## 12.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** Fachlich: Zweck, Owner (mit Vertretung), Kritikalität; ebenso Aufrufer und Abhängigkeiten. Technisch: Pfad, Interpreter (Shebang), Zeilenzahl, Alter beziehungsweise letzte Änderung, strenger Modus, riskante Muster, ShellCheck-Befunde. Das Repository genügt nicht, weil Skripte auch in Crontabs, Timer-Units, Pipeline-Definitionen, auf einzelnen Servern, in Home-Verzeichnissen und in Wikis liegen; was nicht gefunden wird, wird nicht bewertet und nicht gepflegt.

**Lösung zu Aufgabe 2.** Vor jeder Änderung hält man das Ist-Verhalten (Ausgabe, Exit-Code, Dateiänderungen) bei festen Testeingaben als Referenz fest und vergleicht nach jedem kleinen Umbauschritt; Abweichungen sind entweder Fehler oder bewusste Änderungen, die man dokumentiert. Bash spricht für Programmverkettung und Dateiarbeit mit wenig Logik; Python für strukturierte Daten, Tests und komplexe Logik; Ansible für das Herstellen von Zielzuständen auf vielen Systemen (idempotent, deklarativ).

**Lösung zu Aufgabe 3.** RTO: wie lange die Wiederherstellung höchstens dauern darf; RPO: wie viel Datenstand höchstens verloren gehen darf (Abstand zwischen Sicherungen). Ein ungetesteter Restore ist eine Hoffnung, weil Beschädigung, fehlende Schlüssel oder Lücken erst bei der Wiederherstellung auffallen. Bestandteile: Repositories, Artefakte, Secrets (Beschaffungsweg, nicht Werte), Wissen (Runbooks, unabhängig vom Wiki erreichbar).

**Lösung zu Aufgabe 4.** Zentral: Überblick, einheitliche Protokolle und Freigaben (Vorteile); Engpass und Ausfallpunkt sowie Ballung weitreichender Rechte (Nachteile). Lokal: skaliert von selbst und arbeitet auch bei Netzstörungen (Vorteile); schwer zu überblicken, zu aktualisieren und zu überwachen (Nachteile). Ein gestaffelter Rollout führt die Änderung erst bei wenigen Zielen ein und weitet sie bei Erfolg aus; Prüfung zwischen den Stufen begrenzt den Schaden, falls ein Fehler auftritt.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# skript-liste.sh VERZ
set -euo pipefail
verz="${1:?Aufruf: $0 VERZ}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }
echo "pfad,zeilen,owner"
while IFS= read -r -d '' f; do
    erste="$(head -n 1 -- "$f" 2>/dev/null || true)"
    [[ $erste =~ ^#!.*(bash|/sh|[[:space:]]sh)([[:space:]]|$) ]] || continue
    zeilen=$(wc -l < "$f")
    owner="$(head -n 20 -- "$f" | sed -n 's/^# owner: *//p' | head -n 1)"
    printf '"%s",%s,%s\n' "${f//\"/\"\"}" "$zeilen" "${owner:--}"
done < <(find "$verz" -type f -not -path '*/.git/*' -print0 | sort -z)
```

Das Skript liest Dateinamen nullbyte-getrennt, sodass Leerzeichen und Zeilenumbrüche im Namen keine Rolle spielen. Es erkennt Skripte am Shebang statt an der Endung und gibt den Pfad in Anführungszeichen aus (mit verdoppelten Anführungszeichen im Namen, wie es CSV verlangt). Die `|| true` beim Lesen der ersten Zeile verhindert, dass `set -e` bei unlesbaren Dateien abbricht.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# break-even.sh MANUELL_MIN PRO_JAHR ENTWICKLUNG_H STUNDENSATZ
set -euo pipefail
(( $# == 4 )) || { echo "Aufruf: $0 MANUELL_MIN PRO_JAHR ENTWICKLUNG_H STUNDENSATZ" >&2; exit 2; }
for a in "$@"; do
    [[ $a =~ ^[0-9]+([.][0-9]+)?$ ]] && awk -v x="$a" 'BEGIN{exit !(x > 0)}' || { echo "positive Zahl erwartet: $a" >&2; exit 2; }
done
awk -v m="$1" -v n="$2" -v e="$3" -v s="$4" 'BEGIN {
    manuell = m * n / 60                 # Stunden pro Jahr
    nutzen_h = manuell * 0.90            # 10 Prozent fuer Aufsicht und Pflege
    nutzen = nutzen_h * s                # Euro pro Jahr
    einmalig = e * s
    if (nutzen <= 0) { print "Kein Nutzen"; exit 1 }
    printf "Nutzen: %.0f Euro/Jahr, Einmalaufwand: %.0f Euro, Break-even: %.1f Monate\n", nutzen, einmalig, einmalig / nutzen * 12
}'
```

Die Zahlenprüfung kombiniert einen regulären Ausdruck mit einem `awk`-Vergleich auf größer null. Die Rechnung findet in `awk` statt, weil die Shell nur ganze Zahlen rechnet. Beispiel: `./break-even.sh 30 52 16 60` ergibt einen Nutzen von etwa 1 404 Euro pro Jahr bei 960 Euro Einmalaufwand und einen Break-even von rund 8,2 Monaten. Die Zahlen sind Rechenbeispiele.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# migrations-test.sh ALT NEU TESTVERZ
set -uo pipefail
alt="${1:?Aufruf: $0 ALT NEU TESTVERZ}"; neu="${2:?}"; testverz="${3:?}"
[[ -r $alt && -r $neu && -d $testverz ]] || { echo "Eingaben pruefen" >&2; exit 2; }
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT

stand() { ( cd "$1" && find . -type f -print0 | sort -z | xargs -0 -r sha256sum | sha256sum | awk '{print $1}' ); }
vorher="$(stand "$testverz")"

rc_alt=0; bash "$alt" "$testverz" > "$t/alt.out" 2>&1 || rc_alt=$?
rc_neu=0; bash "$neu" "$testverz" > "$t/neu.out" 2>&1 || rc_neu=$?
nachher="$(stand "$testverz")"
[[ $vorher == "$nachher" ]] || echo "WARNUNG: Testverzeichnis wurde veraendert" >&2

if (( rc_alt == rc_neu )) && cmp -s "$t/alt.out" "$t/neu.out"; then
    echo "identisch (Exit-Code $rc_alt, $(wc -l < "$t/alt.out") Zeilen Ausgabe)"
else
    echo "Abweichung: Exit-Code alt=$rc_alt neu=$rc_neu" >&2
    diff -u "$t/alt.out" "$t/neu.out" >&2 || true
    exit 1
fi
```

Der Zustand des Testverzeichnisses wird als Prüfsumme über die Prüfsummen aller Dateien festgehalten, damit ein Skript, das unerwartet Dateien verändert, auffällt. Beide Skripte laufen mit `||`-Erfassung des Exit-Codes, und die Ausgaben werden mit `cmp` und, bei Abweichung, mit `diff` verglichen. Ein Test mit den Skripten `a.sh` und `c.sh` aus Einheit 12.2 liefert eine Abweichung, mit `a.sh` und `b.sh` Gleichheit.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# sicherung-pruefen.sh SICHERUNGSVERZ
set -uo pipefail
verz="${1:?Aufruf: $0 SICHERUNGSVERZ}"; max_h="${MAX_ALTER_H:-26}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }
[[ $max_h =~ ^[0-9]+$ ]] || { echo "MAX_ALTER_H ungueltig" >&2; exit 2; }
fehler=0
ergebnis() { if [[ $1 == ok ]]; then echo "OK      $2"; else echo "FEHLER  $2"; fehler=$((fehler + 1)); fi; }

manifest="$verz/MANIFEST.sha256"
if [[ -s $manifest ]]; then ergebnis ok "Manifest vorhanden"; else ergebnis fehler "Manifest fehlt oder leer"; fi

if [[ -s $manifest ]] && ( cd "$verz" && sha256sum -c --quiet MANIFEST.sha256 >/dev/null 2>&1 ); then
    ergebnis ok "alle Pruefsummen stimmen"
else ergebnis fehler "Pruefsummen stimmen nicht (oder Manifest fehlt)"; fi

neueste="$(find "$verz" -maxdepth 1 -name '*.bundle' -printf '%T@ %p\n' 2>/dev/null | sort -nr | head -n 1 || true)"
if [[ -z $neueste ]]; then ergebnis fehler "keine .bundle-Datei gefunden"
else
    alter_h=$(( ( $(date +%s) - ${neueste%%.*} ) / 3600 ))
    if (( alter_h <= max_h )); then ergebnis ok "neueste Sicherung ist ${alter_h} h alt (Grenze ${max_h} h)"
    else ergebnis fehler "neueste Sicherung ist ${alter_h} h alt (Grenze ${max_h} h)"; fi
fi

fehlend=0
while IFS= read -r -d '' b; do
    grep -qF -- "${b##*/}" "$manifest" 2>/dev/null || { echo "         nicht im Manifest: ${b##*/}"; fehlend=$((fehlend + 1)); }
done < <(find "$verz" -maxdepth 1 -name '*.bundle' -print0)
if (( fehlend == 0 )); then ergebnis ok "alle Bundles stehen im Manifest"; else ergebnis fehler "$fehlend Bundle(s) fehlen im Manifest"; fi
(( fehler == 0 ))
```

Die Funktion `ergebnis` vereinheitlicht die Ausgabe und zählt Fehler. Die Prüfsummen werden im Sicherungsverzeichnis mit `sha256sum -c` geprüft, damit die relativen Dateinamen im Manifest stimmen. Das Alter der neuesten Datei kommt aus `find -printf '%T@'` (GNU-Erweiterung, Sekunden seit 1970); der Dezimalteil wird mit `${neueste%%.*}` abgeschnitten. Die Vollständigkeitsprüfung sucht jeden Dateinamen im Manifest und zeigt fehlende. Test: Legen Sie mit dem Skript aus Einheit 12.6 eine Sicherung an, rufen Sie dieses Skript auf, löschen Sie dann eine Zeile im Manifest oder verändern Sie die Datei mit `head -c 200` und beobachten Sie die Fehlermeldungen.
