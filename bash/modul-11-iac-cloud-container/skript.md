# Modul 11 - Infrastructure as Code, Cloud und Container-Integration

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (11.1 bis 11.8; 11.9 Übungsaufgaben, 11.10 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 10 h laut Blatt „Lehrplan in Modulen“, 10 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (8 Einheiten à 1,25 h). Beide Angaben stimmen überein.  
**Priorität:** 2 (mittel); die Einheiten 11.7 und 11.8 sind im Detailblatt mit Priorität 3 geführt.

| Nr.  | Lerneinheit                  | Dauer (h) | Prio |
|:-----|:-----------------------------|----------:|:----:|
| 11.1 | Cloud-CLI Integration        |      1,25 |  2   |
| 11.2 | IaC Glue-Code                |      1,25 |  2   |
| 11.3 | Container-Basics             |      1,25 |  2   |
| 11.4 | Kubernetes CLI               |      1,25 |  2   |
| 11.5 | Secrets in Cloud und CI      |      1,25 |  2   |
| 11.6 | Umgebungs-Bootstrap          |      1,25 |  2   |
| 11.7 | Policy und Compliance Checks |      1,25 |  3   |
| 11.8 | Portabilität und Runner      |      1,25 |  3   |
|      | **Summe**                    | **10,00** |      |

Moderne Infrastruktur wird nicht mehr von Hand eingerichtet, sondern *beschrieben* und von Werkzeugen hergestellt: Cloud-Ressourcen über deren Kommandozeilen und APIs, Server und Netze über Infrastructure as Code (IaC), Anwendungen in Containern, die ein Orchestrator wie Kubernetes betreibt. Jedes dieser Werkzeuge bringt eine eigene Kommandozeile mit, und Bash ist der *Klebstoff* (Glue-Code) dazwischen: Es ruft die Werkzeuge in der richtigen Reihenfolge auf, wertet ihre Ausgaben aus, prüft Voraussetzungen, reicht Geheimnisse sicher durch und übersetzt Ergebnisse in Exit-Codes für die Pipeline. Dieses Modul zeigt, wie das zuverlässig gelingt. Es setzt die Bausteine der Module 6 bis 10 voraus, besonders JSON-Verarbeitung mit `jq`, Exit-Codes, Logging und die Sicherheitsregeln aus Modul 8.

Eine ehrliche Bemerkung zur Übungsumgebung: Die Beispiele dieses Moduls wurden in einer Umgebung getestet, in der weder ein Cloud-Konto noch ein Docker-Dienst noch ein Kubernetes-Cluster zur Verfügung stand. Die Skripte arbeiten deshalb gegen kleine *Attrappen* (Stubs), die sich wie `aws`, `terraform`, `docker` oder `kubectl` verhalten und typische Ausgaben und Fehler nachbilden. Das ist kein Mangel des Konzepts, sondern ein nützliches Testverfahren: Wer Skripte gegen Attrappen prüft, kann Fehlerpfade (Drosselung, Timeouts, Änderungen) gefahrlos auslösen. Die Aufrufe der echten Werkzeuge sind im Text jeweils genannt; ihr Verhalten gegen ein echtes Konto wurde hier nicht geprüft.

## 11.1 Cloud-CLI Integration

Jeder große Cloud-Anbieter liefert eine Kommandozeile: `aws` (Amazon Web Services), `az` (Microsoft Azure) und `gcloud` (Google Cloud). Alle drei übersetzen Kommandos in Aufrufe der Web-API des Anbieters. Eine *API* (Application Programming Interface) ist die Schnittstelle, über die Programme Dienste steuern; die CLI ist ein bequemer Aufsatz darauf. Ein Kommando besteht aus Dienst, Aktion und Optionen, zum Beispiel `aws ec2 describe-instances`, `az vm list` oder `gcloud compute instances list`. Zum *skriptbaren* Einsatz gehören drei Dinge: eine maschinenlesbare Ausgabe, stabile Exit-Codes und nicht interaktive Anmeldung.

Die *Ausgabe* steuert man mit einer Formatoption: `--output json` bei `aws`, `-o json` bei `az` und `--format=json` bei `gcloud`. JSON ist die Wahl für Skripte, weil es strukturiert ist und sich mit `jq` (Modul 6) zuverlässig auswerten lässt; Tabellenformate sind für Menschen gedacht und ändern sich. Alle drei Werkzeuge haben außerdem eine eingebaute Filtersprache (`-{}-query` mit JMESPath bei `aws` und `az`, `-{}-filter` und `-{}-format` bei `gcloud`), die die Antwort schon vor der Ausgabe eingrenzt. Das spart Datenmenge, ersetzt aber nicht `jq`, wenn man weiterverarbeiten, gruppieren oder verknüpfen will. Eine nützliche Faustregel: serverseitig oder per `-{}-query` grob filtern, mit `jq` fein auswerten.

Die *Anmeldung* in Skripten darf nie interaktiv sein. Pipelines verwenden *Dienstidentitäten* (Service Principals, IAM-Rollen, Workload Identity), idealerweise ohne langlebige Schlüssel: die CI-Plattform bestätigt per kurzlebigem Token, wer sie ist, und der Anbieter gibt befristete Zugangsdaten aus (Modul 11.5). Das Skript prüft nur, *ob* eine Anmeldung besteht (`aws sts get-caller-identity`, `az account show`, `gcloud auth list`), und bricht mit klarer Meldung ab, wenn nicht. Der Exit-Code der CLIs ist bei Erfolg 0 und sonst ungleich 0, aber die genauen Werte unterscheiden sich je Werkzeug und Fehlerart; man verlässt sich daher auf *null oder nicht null* und wertet bei Bedarf die Fehlermeldung (stderr) aus.

Zwei Eigenheiten der Cloud-APIs prägen gute Skripte. Erstens *Drosselung und Zeitüberschreitungen*: Ein Aufruf kann vorübergehend scheitern (Rate Limit, Netzfehler, überlasteter Dienst). Solche Fehler wiederholt man mit wachsendem Abstand (*Exponential Backoff*, ein paar Versuche, dann aufgeben); die CLIs wiederholen intern schon manches, aber nicht alles. Zweitens *Paginierung*: Große Listen kommen seitenweise. Die CLIs blättern meist automatisch (`aws` hat dafür `-{}-no-paginate` und `-{}-max-items`); wer die Antworten selbst abruft, muss das Blättern beachten, sonst sieht er nur die erste Seite. Zusätzlich sollten Skripte *idempotent* sein (Modul 7.7): “Lege den Bucket an, falls er fehlt” statt “lege den Bucket an”, damit ein zweiter Lauf keinen Fehler erzeugt.

Das folgende Beispiel verwendet eine Attrappe, die sich wie ein Cloud-CLI verhält: Sie liefert JSON, schlägt beim ersten Aufruf mit einer Drosselung fehl (Exit-Code 254) und zeigt so die Wiederholungslogik. Das Skript erzeugt die Attrappe selbst in einem temporären Verzeichnis und stellt sie vor den echten `PATH`. Mit einem echten Werkzeug entfiele der Block `bauePseudoCLI`, und der restliche Code bliebe unverändert.

```bash
#!/usr/bin/env bash
# cloud-demo.sh - Cloud-CLI skriptbar nutzen: JSON, jq, Wiederholung, Idempotenz
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/bin"

# --- Attrappe fuer ein Cloud-CLI "cloudctl" (ersetzt hier aws/az/gcloud)
cat > "$t/bin/cloudctl" <<'ENDE'
#!/usr/bin/env bash
zaehler="$CLOUD_STATE/aufrufe"; n=$(( $(cat "$zaehler" 2>/dev/null || echo 0) + 1 )); echo "$n" > "$zaehler"
case "$1 $2" in
  "vm list")
     if [[ ${CLOUD_DROSSELN:-0} == 1 && $n -le 2 ]]; then echo "ThrottlingException: Rate exceeded" >&2; exit 254; fi
     cat <<'JSON'
[{"name":"web-1","state":"running","size":"small","tags":{"env":"prod"}},
 {"name":"web-2","state":"stopped","size":"small","tags":{"env":"prod"}},
 {"name":"db-1","state":"running","size":"large","tags":{"env":"prod","owner":"team-b"}},
 {"name":"test-1","state":"running","size":"small","tags":{"env":"test","owner":"team-a"}}]
JSON
     ;;
  "bucket exists") [[ -e "$CLOUD_STATE/bucket-$3" ]] ;;
  "bucket create") touch "$CLOUD_STATE/bucket-$3"; echo "{\"bucket\":\"$3\",\"created\":true}" ;;
  *) echo "unbekanntes Kommando: $*" >&2; exit 2 ;;
esac
ENDE
chmod +x "$t/bin/cloudctl"
export PATH="$t/bin:$PATH" CLOUD_STATE="$t"

# --- 1. Aufruf mit Wiederholung und Exponential Backoff
cloud_retry() {                              # cloud_retry VERSUCHE BEFEHL...
    local max="$1" versuch=1 pause=1 rc; shift
    while true; do
        rc=0; "$@" && return 0 || rc=$?
        (( versuch >= max )) && { echo "aufgegeben nach $versuch Versuchen (rc=$rc)" >&2; return "$rc"; }
        echo "Versuch $versuch fehlgeschlagen (rc=$rc), warte ${pause}s" >&2
        sleep "$pause"; pause=$((pause * 2)); versuch=$((versuch + 1))
    done
}

# --- 2. Auswerten mit jq: laufende Maschinen ohne owner-Tag
echo "== Laufende VMs ohne owner-Tag"
CLOUD_DROSSELN=1 cloud_retry 4 cloudctl vm list -o json > "$t/vms.json"
jq -r '.[] | select(.state == "running" and (.tags.owner // "") == "") | .name' "$t/vms.json"

echo "== Anzahl je Umgebung"
jq -r 'group_by(.tags.env) | map({env: .[0].tags.env, anzahl: length}) | .[] | "\(.env): \(.anzahl)"' "$t/vms.json"

echo "== Unbeabsichtigte Kosten: gestoppte VMs"
jq -r --arg status stopped '[.[] | select(.state == $status) | .name] | join(", ")' "$t/vms.json"

# --- 3. Idempotent anlegen
sichere_bucket() {
    local name="$1"
    if cloudctl bucket exists "$name"; then echo "Bucket $name existiert bereits"
    else cloudctl bucket create "$name" | jq -r '"angelegt: " + .bucket'; fi
}
echo "== Bucket zweimal sichern"
sichere_bucket daten-backup
sichere_bucket daten-backup
echo "Aufrufe insgesamt: $(cat "$t/aufrufe")"
```

Der erste Abschnitt erzeugt die Attrappe `cloudctl`. Sie zählt ihre Aufrufe in einer Zustandsdatei, damit sie beim ersten und zweiten Aufruf von `vm list` eine Drosselung (Exit-Code 254) melden kann, wenn die Umgebungsvariable `CLOUD_DROSSELN` gesetzt ist. Die Funktion `cloud_retry` ist der wichtigste wiederverwendbare Baustein: Sie führt einen beliebigen Befehl aus, wiederholt ihn bei Fehlern bis zu `max`-mal, verdoppelt die Wartezeit nach jedem Versuch und gibt am Ende den letzten Exit-Code zurück. Das Muster `rc=0; "$@" && return 0 || rc=$?` ist dasselbe wie in Modul 10: Es erfasst den Exit-Code, ohne dass `set -e` das Skript abbricht. Im zweiten Abschnitt landet die JSON-Antwort in einer Datei und wird dreimal mit `jq` ausgewertet: zuerst gefiltert (laufende Maschinen ohne Besitzer-Tag), dann gruppiert (Anzahl je Umgebung) und schließlich zu einer Zeile zusammengefasst. Der Operator `// ""` liefert einen Ersatzwert, wenn das Tag fehlt, und `--arg` übergibt Werte sicher an `jq`, statt sie in den Filtertext zu kleben (Modul 8.2). Der dritte Abschnitt zeigt die Idempotenz: Erst wird geprüft, ob der Bucket existiert, und nur bei Bedarf angelegt, sodass der zweite Aufruf nichts ändert.

Typische Fehler beginnen mit dem Auswerten von Tabellenausgaben (`awk '{print $3}'` auf `--output table`), das bei der nächsten Version des Werkzeugs bricht. Zweitens verlässt man sich auf den ersten Treffer einer Liste (`.[0]`), obwohl die Reihenfolge nicht garantiert ist, oder übersieht die Paginierung. Drittens meldet man Erfolg, obwohl die Anmeldung fehlte oder ein falsches Konto aktiv war: Ein Skript, das gegen die Produktions-Subscription statt gegen die Testumgebung läuft, richtet echten Schaden an. Deshalb gibt man Konto, Region und Projekt *ausdrücklich* an oder prüft sie zu Beginn und bricht bei Abweichung ab. Viertens fehlen Wiederholungen bei Drosselung, und eine Pipeline scheitert zufällig. Fünftens enthalten Namen und Tags Sonderzeichen oder Leerzeichen und werden ungequotet weitergegeben (Modul 3.4). Sechstens schreibt man Secrets in Befehlszeilen, sodass sie in der Prozessliste und im Log erscheinen (Modul 11.5).

```bash
# falsch: Tabellenausgabe parsen, ohne Konto zu pruefen, Namen ungequotet
aws ec2 describe-instances --output table | awk '/running/ {print $4}' | while read id; do aws ec2 stop-instances --instance-ids $id; done

# richtig: JSON + jq, Konto pruefen, quoten, Fehler beachten
[[ "$(aws sts get-caller-identity --query Account --output text)" == "$ERWARTETES_KONTO" ]] || { echo "falsches Konto" >&2; exit 1; }
aws ec2 describe-instances --filters Name=instance-state-name,Values=running --output json |
    jq -r '.Reservations[].Instances[].InstanceId' |
    while IFS= read -r id; do aws ec2 stop-instances --instance-ids "$id" >/dev/null || echo "Fehler bei $id" >&2; done
```

> **Merke**
>
> Cloud-CLIs skriptbar nutzen heißt: JSON-Ausgabe, Auswertung mit `jq` (Werte per `-{}-arg`), Exit-Code beachten, nicht interaktiv anmelden, Konto und Region prüfen, bei Drosselung mit Backoff wiederholen, Paginierung beachten, Anlegen idempotent gestalten.

> **Praxisbeispiel**
>
> Ein nächtliches Skript meldet alle laufenden Maschinen ohne Besitzer-Tag an die Kostenverantwortlichen. Es nutzt dieselbe Struktur wie oben: Konto prüfen, Liste als JSON holen, mit `jq` filtern, das Ergebnis als Tabelle in eine Mail schreiben. Bei einer vorübergehenden Drosselung wiederholt der Wrapper den Aufruf, bevor er Alarm schlägt.

## 11.2 IaC Glue-Code

*Infrastructure as Code* (IaC) beschreibt Infrastruktur in Textdateien, die man versioniert, prüft und automatisch anwenden kann. Zwei Familien von Werkzeugen sind verbreitet: *Terraform* (und das Open-Source-Derivat OpenTofu) mit eigener Beschreibungssprache für viele Anbieter, und *Bicep* (sowie ARM-Vorlagen) für Azure. Beide arbeiten *deklarativ*: Man beschreibt den gewünschten Zustand, und das Werkzeug berechnet, was zu tun ist, um dorthin zu kommen. Terraform vergleicht dazu den *Zustand* (State), die Beschreibung und die tatsächlichen Ressourcen und zeigt die Unterschiede als *Plan*. Erst `apply` führt Änderungen aus.

Bash orchestriert diesen Ablauf, denn die Werkzeuge selbst beantworten nicht, *wann* welcher Schritt läuft. Der Standardablauf ist: `terraform init` (Anbieter-Plugins und Backend vorbereiten), `terraform validate` (Syntax und interne Konsistenz), `terraform fmt -check` (Formatierung), `terraform plan -out=plan.bin` (Änderungen berechnen und speichern), Prüfung und Freigabe, `terraform apply plan.bin` (genau den gespeicherten Plan ausführen). Das *Speichern* des Plans ist wichtig: Wer nach der Freigabe ein neues `apply` ohne Plan startet, kann etwas anderes ausführen, als geprüft wurde, weil sich die Umgebung zwischenzeitlich geändert haben kann. In Pipelines gibt man `-input=false` an, damit das Werkzeug nie auf eine Eingabe wartet, und `-no-color`, damit Logs lesbar bleiben.

Besonders für Automation wertvoll ist ein bestimmter Schalter: `terraform plan -detailed-exitcode`. Normalerweise endet `plan` mit 0, wenn er erfolgreich war, egal ob Änderungen anstehen. Mit dem Schalter gilt: Exit-Code 0 heißt „erfolgreich, keine Änderungen“, 1 heißt „Fehler“ und 2 heißt „erfolgreich, Änderungen vorhanden“. Damit kann ein Skript Abweichungen erkennen, ohne die Textausgabe zu parsen.

> **Ergänzung aus der Nachrecherche**
>
> **Ergänzung aus der Nachrecherche:** Die Dokumentation zu `terraform plan` (abgerufen, siehe Quellenverzeichnis) beschreibt für die Option `-detailed-exitcode` die Bedeutung der Exit-Codes so: 0 für einen erfolgreichen Lauf ohne Änderungen, 1 für einen Fehler und 2 für einen erfolgreichen Lauf mit vorhandenen Änderungen. Genau diese Unterscheidung nutzt die Drift-Prüfung unten. Das Verhalten wurde hier mit einer Attrappe nachgestellt, nicht mit dem echten Terraform. Prüfen Sie die Dokumentation Ihrer eingesetzten Version, da Optionen sich ändern können.

Die *Drift-Prüfung* fragt: Entspricht die reale Infrastruktur noch dem Code? Abweichungen entstehen, wenn jemand Ressourcen von Hand ändert (Konsolen-Klick, Notfalleingriff). Eine regelmäßige Pipeline führt dazu `plan -detailed-exitcode` aus und meldet bei Exit-Code 2 eine Abweichung, ohne etwas zu ändern. Das Skript muss die drei Fälle sauber unterscheiden, denn „Fehler“ (1) und „Abweichung“ (2) verlangen unterschiedliche Reaktionen. Außerdem sollte der `apply`-Schritt nur laufen, wenn ein Mensch (oder eine Regel) den Plan freigegeben hat, und der *State* (Zustandsdatei) gehört in ein gesichertes, gesperrtes Backend, nicht in das Repository: Er enthält oft Geheimnisse.

Auch hier arbeitet das Beispiel mit einer Attrappe `terraform`, die über eine Datei steuerbar ist (keine Änderungen, Änderungen, Fehler). Es zeigt den Ablauf und die Auswertung der Exit-Codes, nicht das Verhalten eines echten Anbieters.

```bash
#!/usr/bin/env bash
# iac-ablauf.sh - Plan, Drift-Pruefung und kontrolliertes Apply (mit Attrappe "terraform")
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/bin" "$t/arbeit"

cat > "$t/bin/terraform" <<'ENDE'
#!/usr/bin/env bash
# Attrappe: Verhalten steuert die Datei $TF_SZENARIO (keine|aenderung|fehler)
cmd="$1"; shift
szenario="$(cat "$TF_SZENARIO")"
case "$cmd" in
  init|validate|fmt) echo "terraform $cmd: ok" ;;
  plan)
     detailed=0; out=""
     for a in "$@"; do [[ $a == -detailed-exitcode ]] && detailed=1; [[ $a == -out=* ]] && out="${a#-out=}"; done
     case "$szenario" in
        fehler)    echo "Error: Invalid provider configuration" >&2; exit 1 ;;
        aenderung) echo "Plan: 1 to add, 0 to change, 0 to destroy."; [[ -n $out ]] && echo "PLAN-INHALT" > "$out"; (( detailed )) && exit 2 || exit 0 ;;
        *)         echo "No changes. Infrastructure matches the configuration."; [[ -n $out ]] && echo "LEERER-PLAN" > "$out"; exit 0 ;;
     esac ;;
  apply) planfile="${!#}"; echo "Apply complete! Verwendeter Plan: $(cat "$planfile")" ;;
esac
ENDE
chmod +x "$t/bin/terraform"
export PATH="$t/bin:$PATH" TF_SZENARIO="$t/szenario"

# --- Plan mit ausgewerteten Exit-Codes
plane() {                                    # plane VERZ  -> setzt PLAN_RC
    local verz="$1"
    ( cd "$verz" && terraform init -input=false >/dev/null && terraform validate >/dev/null \
        && terraform fmt -check >/dev/null ) || { echo "Vorpruefung fehlgeschlagen" >&2; return 1; }
    PLAN_RC=0
    ( cd "$verz" && terraform plan -input=false -no-color -detailed-exitcode -out=plan.bin ) || PLAN_RC=$?
    return 0
}

drift_pruefung() {
    plane "$t/arbeit"
    case $PLAN_RC in
        0) echo "[OK] kein Drift"; return 0 ;;
        2) echo "[WARNUNG] Drift: reale Infrastruktur weicht vom Code ab" >&2; return 2 ;;
        *) echo "[FEHLER] Plan fehlgeschlagen (rc=$PLAN_RC)" >&2; return 1 ;;
    esac
}

freigabe() { [[ "${FREIGABE:-nein}" == ja ]]; }     # in der Praxis: Pipeline-Freigabe oder Review

anwenden() {
    drift_pruefung && { echo "nichts zu tun"; return 0; }
    local rc=$?
    (( rc == 2 )) || return "$rc"
    freigabe || { echo "Apply nicht freigegeben; Plan bleibt gespeichert" >&2; return 3; }
    ( cd "$t/arbeit" && terraform apply -input=false -no-color plan.bin )
}

for s in keine aenderung fehler; do
    echo "=== Szenario: $s"; echo "$s" > "$t/szenario"
    rc=0; anwenden || rc=$?; echo "Ergebnis-Exit-Code: $rc"
done
echo "=== Szenario aenderung mit Freigabe"; echo aenderung > "$t/szenario"
FREIGABE=ja anwenden
```

Die Attrappe unterscheidet die drei Szenarien über eine Datei, sodass das Skript jeden Weg durchlaufen kann. Die Funktion `plane` ist der Kern: Sie prüft zuerst `init`, `validate` und `fmt -check` und führt dann den Plan mit `-detailed-exitcode` und `-out` aus. Dass der Plan-Aufruf mit `|| PLAN_RC=$?` abgefangen wird, ist entscheidend, weil Exit-Code 2 hier ein *normales Ergebnis* und kein Fehler ist; ohne das Abfangen würde `set -e` das Skript abbrechen. `drift_pruefung` übersetzt die Codes in Meldungen und eigene Rückgabewerte. `anwenden` führt `apply` nur aus, wenn Änderungen anstehen *und* eine Freigabe vorliegt, und verwendet dann genau die gespeicherte Datei `plan.bin`. Der letzte Test zeigt, dass mit gesetzter Freigabe die Attrappe den gespeicherten Plan anwendet.

Typische Fehler: Erstens läuft `terraform apply` ohne gespeicherten Plan und ohne Freigabe automatisch auf Produktion. Zweitens wertet man den Exit-Code 2 als Fehler und bricht ab, obwohl nur Änderungen anstehen, oder umgekehrt ignoriert man Code 1. Drittens fehlt `-input=false`, und die Pipeline hängt an einer Rückfrage. Viertens liegt der State lokal oder im Repository, und zwei Läufe überschreiben einander; ein Remote-Backend mit Sperre löst das. Fünftens landen Secrets in Variablendateien oder im Plan (der Plan enthält oft Klartext-Werte und gehört daher nicht in öffentliche Artefakte). Sechstens läuft die Drift-Prüfung mit denselben weitreichenden Rechten wie `apply`, obwohl lesende Rechte genügen würden (Modul 8.5). Siebtens pinnt man Werkzeug- und Anbieterversionen nicht und erhält bei jedem Lauf anderes Verhalten.

```bash
# falsch: Code 2 als Fehler behandeln (set -e), apply ohne Plan
set -e
terraform plan -detailed-exitcode        # Exit 2 bricht hier das Skript ab
terraform apply -auto-approve

# richtig: Codes auswerten, gespeicherten Plan anwenden
rc=0; terraform plan -input=false -detailed-exitcode -out=plan.bin || rc=$?
case $rc in 0) echo "keine Aenderung" ;; 2) terraform apply -input=false plan.bin ;; *) echo "Plan-Fehler" >&2; exit 1 ;; esac
```

> **Merke**
>
> IaC-Ablauf: `init`, `validate`, `fmt -check`, `plan -out`, Freigabe, `apply` mit dem gespeicherten Plan. `-detailed-exitcode`: 0 keine Änderung, 1 Fehler, 2 Änderungen. Drift regelmäßig prüfen, State im gesperrten Remote-Backend, `-input=false`, Versionen pinnen, Pläne wie Geheimnisse behandeln.

> **Praxisbeispiel**
>
> Jede Nacht läuft eine Pipeline mit lesender Rolle: `plan -detailed-exitcode` je Umgebung. Bei Exit-Code 2 erscheint ein Ticket „Drift in prod-netz“ mit dem Planauszug; der Eingriff von Hand wird entweder in den Code übernommen oder zurückgesetzt. Änderungen an der Infrastruktur selbst laufen nur über Pull Requests mit freigegebenem Plan.

## 11.3 Container-Basics

Ein *Container* ist ein isolierter Prozess mit eigenem Dateisystem, eigenem Netz und eigenen Ressourcengrenzen, der sich den Kernel mit dem Rechner teilt. Er ist leichter als eine virtuelle Maschine, weil er kein eigenes Betriebssystem startet. Seine Stärke liegt in der *Reproduzierbarkeit*: Ein *Image* enthält Programm, Bibliotheken und Konfiguration in genau einer bestimmten Version, und derselbe Container läuft auf dem Laptop wie auf dem Server. Für Bash ist das doppelt nützlich. Man kann Skripte *in* einem Container laufen lassen, um eine definierte Werkzeugumgebung zu haben (zum Beispiel genau die Version von `jq` und `shellcheck`, die das Team testet), und man steuert Container *aus* Skripten heraus.

Die wichtigsten Befehle sind `docker build` (aus einer *Dockerfile* ein Image bauen), `docker run` (aus einem Image einen Container starten), `docker logs` (die Ausgabe eines Containers lesen), `docker ps` (laufende Container), `docker exec` (einen Befehl in einem laufenden Container ausführen) und `docker rm` (Container löschen). Die Dockerfile besteht aus Anweisungen: `FROM` (Basis-Image), `RUN` (Befehl beim Bauen), `COPY` (Dateien hineinkopieren), `WORKDIR`, `USER` (unter welchem Benutzer der Prozess läuft) und `ENTRYPOINT`/`CMD` (was beim Start passiert). Jede Anweisung erzeugt eine Schicht, die zwischengespeichert wird; deshalb stehen selten ändernde Anweisungen vorn.

Wichtige Optionen für Skripte sind `--rm` (Container nach dem Ende automatisch löschen, sonst sammeln sie sich an), `-v quelle:ziel:ro` (Verzeichnis als Volume einbinden, hier nur lesbar), `-e NAME` (Umgebungsvariable weiterreichen, ohne den Wert im Befehl zu nennen), `--user "$(id -u):$(id -g)"` (nicht als root laufen und erzeugte Dateien dem eigenen Benutzer gehören lassen), `--read-only` (Dateisystem schreibgeschützt), `--network none` (kein Netz, wo keines nötig ist) und `--memory`/`--cpus` (Grenzen). Der Exit-Code von `docker run` ist der des Prozesses im Container, sodass er sich wie jeder Befehl in Skripten auswerten lässt. Reproduzierbarkeit verlangt außerdem, Images nicht über `:latest` zu beziehen, sondern über ein festes Tag oder einen Digest (`image@sha256:...`).

Weil hier kein Docker-Dienst lief, zeigt das Beispiel einen Ablauf gegen eine Attrappe `docker`. Er enthält die echte Dockerfile als Text und ein Skript, das ein Werkzeug-Image baut, ein Prüfskript im Container ausführt und den Exit-Code zurückgibt. Die Attrappe gibt den Aufruf aus, den sie bekommen hat, damit man die Optionen im Test sieht.

```bash
#!/usr/bin/env bash
# container-demo.sh - Skript in reproduzierbarer Umgebung ausfuehren (Attrappe "docker")
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/bin" "$t/projekt"

cat > "$t/bin/docker" <<'ENDE'
#!/usr/bin/env bash
# Attrappe: zeigt Aufruf, simuliert Build und Run
case "$1" in
  build) echo "[docker-Attrappe] build $*" >&2; echo "sha256:abc123" ;;
  run)   echo "[docker-Attrappe] run $*" >&2
         [[ "$*" == *pruefe-skript* ]] && { echo "pruefe-skript: 3 Dateien geprueft, 0 Fehler"; exit 0; }
         echo "fehler im Container" >&2; exit 3 ;;
  logs)  echo "Zeile aus Container-Log" ;;
  *)     exit 2 ;;
esac
ENDE
chmod +x "$t/bin/docker"
export PATH="$t/bin:$PATH"

# --- Dockerfile (Text; mit echtem Docker wuerde "docker build" sie verwenden)
cat > "$t/projekt/Dockerfile" <<'ENDE'
FROM debian:12-slim
RUN apt-get update && apt-get install -y --no-install-recommends jq shellcheck \
    && rm -rf /var/lib/apt/lists/*
RUN useradd --create-home --uid 10001 werkzeug
USER werkzeug
WORKDIR /arbeit
ENTRYPOINT ["/bin/bash"]
ENDE

IMAGE="skript-werkzeuge:1.0"

baue_image() {
    docker build --pull --tag "$IMAGE" "$t/projekt" > "$t/build.id"
    echo "Image gebaut: $(cat "$t/build.id")"
}

fuehre_im_container_aus() {                  # fuehre_im_container_aus VERZ SKRIPT
    local verz="$1" skript="$2"
    docker run --rm \
        --user "$(id -u):$(id -g)" \
        --read-only --network none --memory 256m --cpus 1 \
        -v "$verz:/arbeit:ro" \
        "$IMAGE" "/arbeit/$skript"
}

baue_image
echo "== Lauf 1 (Skript vorhanden)"
fuehre_im_container_aus "$t/projekt" pruefe-skript
rc=0; echo "== Lauf 2 (Skript fehlt)"
fuehre_im_container_aus "$t/projekt" fehlt.sh || rc=$?
echo "Exit-Code des Containers: $rc"
```

Die Dockerfile oben im Heredoc zeigt die übliche Struktur: ein schlankes Basis-Image mit fester Hauptversion, ein einziges `RUN` für Installation und Aufräumen (damit die Zwischendateien nicht in der Schicht bleiben), ein unprivilegierter Benutzer und ein `ENTRYPOINT`. Die Funktion `fuehre_im_container_aus` vereint die Sicherheitsoptionen: Der Container läuft unter der eigenen Benutzerkennung, ohne Schreibrecht auf das Dateisystem, ohne Netz, mit Speicher- und CPU-Grenze, und das Projekt wird nur lesbar eingebunden. Das ist eine gute Voreinstellung für Prüfskripte, weil ein fehlerhaftes oder bösartiges Skript so kaum Schaden anrichten kann. Der Exit-Code wird mit `|| rc=$?` erfasst, damit man das Ergebnis des Containers auswerten kann. Die Attrappe antwortet im ersten Lauf mit Erfolg und im zweiten mit dem Code 3.

Typische Fehler: Erstens läuft alles als `root` im Container, und gemountete Dateien gehören danach `root`. Zweitens wird “`:latest`” verwendet, und der Lauf ist nicht reproduzierbar. Drittens bleiben Container ohne `-{}-rm` zurück und füllen die Platte. Viertens werden Geheimnisse mit `-e TOKEN=wert` oder in `ENV`/`ARG` der Dockerfile eingebaut, sodass sie in der Prozessliste, im Verlauf des Images und im Log landen (Modul 11.5). Fünftens bindet man das Docker-Socket (`/var/run/docker.sock`) oder `/` in den Container ein, was dem Container praktisch Root-Rechte auf dem Host gibt. Sechstens wartet man nicht auf Bereitschaft: Ein Container ist „gestartet“, aber der Dienst darin noch nicht erreichbar; man prüft mit einer Wiederholungsschleife (Modul 11.1) oder einem Health-Check. Siebtens nimmt man an, ein Skript verhalte sich im Container wie auf dem Host, obwohl dort andere Werkzeugversionen, Zeitzone und Sprache gelten (Modul 11.8).

```bash
# falsch: latest, root, Secret in der Befehlszeile, Socket durchgereicht
docker run -it -e DB_PASSWORT=geheim123 -v /var/run/docker.sock:/var/run/docker.sock tools:latest bash

# richtig: festes Tag, eigener Benutzer, Secret nur per Name, nichts durchreichen
docker run --rm --user "$(id -u):$(id -g)" -e DB_PASSWORT tools:1.4.2 /arbeit/migriere.sh
```

> **Merke**
>
> Container machen Umgebungen reproduzierbar. Skriptfreundlich: `-{}-rm`, festes Image-Tag oder Digest, eigener Benutzer, Volumes nur lesbar, kein Netz wo möglich, Grenzen für Speicher und CPU. Exit-Code von `docker run` auswerten, Geheimnisse nie in Befehlszeile oder Image, nie das Docker-Socket durchreichen.

> **Praxisbeispiel**
>
> Das Team führt ShellCheck und die Skript-Tests nicht auf dem Laptop, sondern im selben Werkzeug-Image wie die CI aus. Dadurch gibt es kein „bei mir geht es“: Dieselben Versionen von Bash, `jq` und `shellcheck` prüfen lokal und in der Pipeline.

## 11.4 Kubernetes CLI

*Kubernetes* ist ein Orchestrator, der Container über viele Rechner verteilt betreibt, neu startet, skaliert und aktualisiert. Man beschreibt, was laufen soll (Anwendung, Anzahl der Kopien, Ressourcen), und Kubernetes sorgt dafür, dass der tatsächliche Zustand dem gewünschten entspricht. Die Kommandozeile heißt `kubectl`. Sie spricht mit der API des Clusters und kennt drei Ordnungsbegriffe, die für sicheres Skripten entscheidend sind. Der *Context* bündelt Cluster, Benutzer und Standard-Namespace und steht in der Datei `~/.kube/config`. Der *Namespace* ist eine logische Abtrennung innerhalb eines Clusters (zum Beispiel `dev`, `test`, `prod`). Das *Objekt* ist die Ressource selbst: `pod` (kleinste Laufeinheit mit einem oder mehreren Containern), `deployment` (verwaltet Kopien eines Pods und Aktualisierungen), `service` (stabile Adresse), `configmap` und `secret` (Konfiguration und Geheimnisse).

Die Befehle, die man in Skripten täglich nutzt: `kubectl get` (Objekte anzeigen), `kubectl describe` (Details und Ereignisse), `kubectl logs` (Container-Ausgabe), `kubectl apply -f` (gewünschten Zustand aus einer Datei herstellen), `kubectl rollout status` (auf das Ende einer Aktualisierung warten), `kubectl rollout undo` (zur vorigen Version zurück), `kubectl wait` (auf eine Bedingung warten) und `kubectl exec` (Befehl in einem Container). Für Skripte ist `-o json` wichtig, mit `jq` ausgewertet, oder `-o jsonpath=...` und `-o name`. Wie bei den Cloud-CLIs gilt: Tabellenausgaben sind für Menschen gedacht und nicht stabil.

Der wichtigste Sicherheitsgrundsatz lautet: *Nie auf dem impliziten Context arbeiten.* Wer in einem Terminal zuletzt Produktion ausgewählt hat und dann ein Skript startet, löscht womöglich die falschen Ressourcen. Skripte geben deshalb Context und Namespace *in jedem Aufruf* an (`--context`, `--namespace`), prüfen den Context gegen eine Allowlist und lassen Produktionskontexte nur bei ausdrücklicher Bestätigung zu. Außerdem prüft man vor Änderungen mit `kubectl diff -f datei.yaml` (zeigt Unterschiede ohne Änderung) und `--dry-run=server`. Rollouts wartet man mit `kubectl rollout status deployment/NAME --timeout=120s` ab; der Befehl endet mit Exit-Code 0, wenn die Aktualisierung gelungen ist, und mit einem Fehlercode bei Zeitüberschreitung. Daraus ergibt sich das Muster: anwenden, warten, bei Fehler zurückrollen.

Auch dieses Beispiel arbeitet mit einer Attrappe `kubectl`, damit es ohne Cluster läuft. Die Attrappe kennt zwei Contexts (`dev` und `prod`) und simuliert einen gelungenen und einen fehlgeschlagenen Rollout. Das Skript zeigt Context-Sicherheit, die Auswertung von JSON und das Rollback.

```bash
#!/usr/bin/env bash
# k8s-deploy.sh CONTEXT NAMESPACE DEPLOYMENT IMAGE - sicherer Rollout mit Rollback (Attrappe "kubectl")
set -euo pipefail
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/bin"

cat > "$t/bin/kubectl" <<'ENDE'
#!/usr/bin/env bash
# Attrappe: protokolliert Aufrufe; "kaputt" im Image-Namen laesst den Rollout scheitern
echo "kubectl $*" >> "$K8S_LOG"
args=" $* "
case "$args" in
  *" config get-contexts "*) printf 'dev\nprod\n' ;;
  *" get deployment "*-o\ json*)
      echo '{"metadata":{"name":"web"},"spec":{"replicas":3},"status":{"readyReplicas":3,"updatedReplicas":3}}' ;;
  *" set image "*)   : ;;
  *" rollout status "*) [[ -e $K8S_KAPUTT ]] && { echo "error: timed out waiting for the condition" >&2; exit 1; }; echo "deployment successfully rolled out" ;;
  *" rollout undo "*) echo "deployment rolled back" ;;
  *) : ;;
esac
ENDE
chmod +x "$t/bin/kubectl"
export PATH="$t/bin:$PATH" K8S_LOG="$t/log" K8S_KAPUTT="$t/kaputt"

ERLAUBTE_CONTEXTS=(dev prod)
PRODUKTIV=(prod)

enthalten() { local x="$1"; shift; local y; for y in "$@"; do [[ $x == "$y" ]] && return 0; done; return 1; }

rollout() {
    local ctx="$1" ns="$2" dep="$3" image="$4"
    enthalten "$ctx" "${ERLAUBTE_CONTEXTS[@]}" || { echo "Context nicht erlaubt: $ctx" >&2; return 2; }
    if enthalten "$ctx" "${PRODUKTIV[@]}" && [[ "${BESTAETIGT:-}" != "$ctx" ]]; then
        echo "Produktiver Context '$ctx': BESTAETIGT=$ctx setzen" >&2; return 3
    fi
    [[ $ns =~ ^[a-z0-9]([-a-z0-9]*[a-z0-9])?$ ]] || { echo "ungueltiger Namespace" >&2; return 2; }
    local k=(kubectl --context "$ctx" --namespace "$ns")            # immer explizit
    "${k[@]}" set image "deployment/$dep" "$dep=$image"
    if "${k[@]}" rollout status "deployment/$dep" --timeout=120s; then
        "${k[@]}" get deployment "$dep" -o json |
            jq -r '"bereit: \(.status.readyReplicas)/\(.spec.replicas), aktualisiert: \(.status.updatedReplicas)"'
    else
        echo "Rollout fehlgeschlagen, Rollback" >&2
        "${k[@]}" rollout undo "deployment/$dep"
        return 1
    fi
}

echo "== 1) Erfolgreicher Rollout in dev";       rollout dev web web registry.example/web:1.5
echo "== 2) prod ohne Bestaetigung";             rc=0; rollout prod web web registry.example/web:1.5 || rc=$?; echo "rc=$rc"
echo "== 3) unbekannter Context";                rc=0; rollout test web web x || rc=$?; echo "rc=$rc"
echo "== 4) Fehlschlagender Rollout mit Rollback"; touch "$t/kaputt"
rc=0; BESTAETIGT=prod rollout prod web web registry.example/web:kaputt || rc=$?; echo "rc=$rc"
echo "== Protokoll der kubectl-Aufrufe"; cut -c1-110 "$t/log"
```

Das Skript beantwortet die entscheidende Frage “*wo* wirkt mein Befehl” mit Programmcode. Die Hilfsfunktion `enthalten` prüft, ob ein Wert in einer Liste steht. In `rollout` werden zuerst Context und Namespace validiert; produktive Contexts verlangen eine ausdrückliche Bestätigung über eine Variable, die den Namen des Contexts nennen muss (eine simple, aber wirksame Sicherung gegen versehentliches Starten). Das Feld `k` ist ein *Array* mit dem festen Präfix `kubectl --context ... --namespace ...`; jeder spätere Aufruf expandiert es als `"${k[@]}"`, wodurch Context und Namespace nie vergessen werden können (Modul 3.7). Nach `set image` wird der Rollout mit `rollout status` abgewartet. Gelingt er, liest `jq` den Zustand aus der JSON-Antwort, andernfalls ruft das Skript `rollout undo` auf und liefert Exit-Code 1. Das Protokoll am Ende zeigt, dass jeder Aufruf Context und Namespace trug.

Typische Fehler: Erstens läuft das Skript gegen den zuletzt gewählten Context, weil man ihn nicht angegeben hat. Zweitens wird nach dem Anwenden nicht gewartet, und die Pipeline meldet Erfolg, obwohl die neuen Pods nie bereit wurden. Drittens fehlt das Zeitlimit bei `rollout status` oder `wait`, und die Pipeline hängt. Viertens verwendet man Tabellenausgabe statt JSON. Fünftens wird mit `kubectl delete` oder `apply -{}-prune` gearbeitet, ohne vorher mit `diff` oder `-{}-dry-run` zu prüfen. Sechstens liest `kubectl get secret -o yaml` Geheimnisse ins Log; sie sind nur Base64-kodiert, nicht verschlüsselt (Modul 11.5). Siebtens werden Pods per Namen angesprochen, obwohl deren Namen sich bei jedem Neustart ändern; man wählt sie über Labels (`-l app=web`) oder spricht das Deployment an.

```bash
# falsch: impliziter Context, kein Warten, Tabelle parsen
kubectl set image deployment/web web=reg/web:1.5
kubectl get pods | grep web | awk '{print $2}'

# richtig: explizit, warten, strukturiert pruefen, bei Fehler zurueck
k=(kubectl --context "$CTX" --namespace "$NS")
"${k[@]}" set image deployment/web web=reg/web:1.5
"${k[@]}" rollout status deployment/web --timeout=120s || { "${k[@]}" rollout undo deployment/web; exit 1; }
"${k[@]}" get pods -l app=web -o json | jq -r '.items[] | "\(.metadata.name) \(.status.phase)"'
```

> **Merke**
>
> Context und Namespace in jedem Aufruf angeben (Array `k=(kubectl -{}-context ... -{}-namespace ...)`), Contexts per Allowlist prüfen, Produktion bestätigen lassen. Vor Änderungen `diff`/`dry-run`, nach Änderungen `rollout status` mit Zeitlimit, bei Fehler `rollout undo`. Ausgaben als JSON; Pods über Labels, nicht über Namen.

> **Praxisbeispiel**
>
> Ein Release-Skript nimmt “Umgebung” als Argument, übersetzt sie in Context und Namespace aus einer festen Tabelle und verweigert alles andere. Für Produktion verlangt es ein zweites Argument mit der Ticketnummer, die es in das Protokoll schreibt. Dadurch ist jeder Rollout einem Vorgang zugeordnet.

## 11.5 Secrets in Cloud und CI

Ein *Secret* ist ein Geheimnis: Passwort, API-Schlüssel, Token, privater Schlüssel, Verbindungszeichenfolge. In Pipelines und Cloud-Umgebungen entstehen Lecks selten durch geniale Angriffe, sondern durch Alltagsfehler: ein Wert im Repository, im Log, in der Befehlszeile, im Container-Image oder im Plan eines IaC-Werkzeugs. Die Regeln aus Modul 8.4 (Geheimnisse nie im Code, nie in Argumenten, nie in Logs) gelten hier unverändert; diese Einheit zeigt, wie man sie mit den Mechanismen von Cloud-Anbietern und CI-Plattformen umsetzt.

Es gibt zwei Quellen für Secrets in Automationen. Die erste sind *Secret Stores* der Cloud: AWS Secrets Manager, Azure Key Vault, Google Secret Manager oder ein eigenständiger Tresor wie HashiCorp Vault. Das Skript holt ein Geheimnis zur Laufzeit mit seiner eigenen Identität (Rolle, Managed Identity, Workload Identity) und nutzt es nur im Speicher; die Rechte sind auf genau dieses Geheimnis beschränkt, und jeder Zugriff wird protokolliert. Die zweite Quelle sind *Pipeline-Secrets* der CI-Plattform (zum Beispiel GitHub Actions Secrets): Die Plattform stellt den Wert als maskierte Umgebungsvariable bereit. Besser als beide statischen Varianten ist es, gar keine langlebigen Schlüssel zu besitzen: Die Pipeline meldet sich per *OIDC* (OpenID Connect) mit einem kurzlebigen, plattformsignierten Token beim Anbieter an und erhält befristete Zugangsdaten (Workload Identity Federation). Dann gibt es nichts, was gestohlen werden könnte.

Die Weitergabe an Programme ist der heikle Teil. Befehlszeilenargumente sind für alle Benutzer des Systems in der Prozessliste sichtbar (`ps`); Umgebungsvariablen sind im Prozess lesbar und werden an Kindprozesse vererbt. Sicherer sind Standardeingabe (`--password-stdin` bei `docker login`), eine Datei mit Rechten 600 in einem flüchtigen Verzeichnis oder ein Dateideskriptor. Zweite Gefahr sind *Logs*: Plattformen maskieren Pipeline-Secrets, aber nur den exakten Wert; ein Base64-kodierter, abgeschnittener oder in JSON eingebetteter Wert wird nicht erkannt. Deshalb schaltet man `set -x` in Abschnitten mit Geheimnissen ab, gibt Befehle nie samt Secrets aus und maskiert abgeleitete Werte ausdrücklich (bei GitHub Actions mit `echo "::add-mask::WERT"`).

Das Beispiel zeigt die wiederverwendbaren Bausteine: Ein Geheimnis wird aus einem Store (Attrappe) geholt, nur an einen Kindprozess über die Umgebung weitergereicht, aus Fehlermeldungen herausgefiltert, und am Ende wird das Log auf Leckagen geprüft. Die Funktion `maskiere` ersetzt bekannte Geheimnisse in Texten, die man ausgibt.

```bash
#!/usr/bin/env bash
# secrets-demo.sh - Secrets holen, sparsam weitergeben, Log auf Leckagen pruefen (Attrappe "secretctl")
set -euo pipefail
umask 077
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/bin"

cat > "$t/bin/secretctl" <<'ENDE'
#!/usr/bin/env bash
# Attrappe eines Secret-Stores: secretctl get NAME
[[ $1 == get ]] || exit 2
case "$2" in
  db/passwort) printf 'S3cret-Passw0rt!' ;;
  *) echo "Secret nicht gefunden: $2" >&2; exit 1 ;;
esac
ENDE
chmod +x "$t/bin/secretctl"
export PATH="$t/bin:$PATH"

GEHEIMNISSE=()                                   # Liste der bekannten Geheimnisse fuer die Maskierung
hole_secret() {                                  # hole_secret NAME -> gibt Wert auf stdout, merkt ihn zum Maskieren
    local wert
    wert="$(secretctl get "$1")" || { echo "Secret $1 nicht lesbar" >&2; return 1; }
    GEHEIMNISSE+=("$wert")
    printf '%s' "$wert"
}
maskiere() {                                     # filtert bekannte Geheimnisse aus stdin
    local text; text="$(cat)"
    local g
    for g in "${GEHEIMNISSE[@]}"; do text="${text//"$g"/***}"; done
    printf '%s\n' "$text"
}

# --- Programm, das ein Passwort aus der Umgebung braucht (statt aus der Befehlszeile)
db_client() { echo "verbinde mit Passwort der Laenge ${#DB_PASSWORT}"; [[ $DB_PASSWORT == S3cret* ]] || return 1; }

echo "== 1) Secret nur im Kindprozess verfuegbar"
( DB_PASSWORT="$(hole_secret db/passwort)"; export DB_PASSWORT; db_client )
echo "Im Elternprozess gesetzt? ${DB_PASSWORT:-nein}"

echo "== 2) Falsch: Secret im Befehl (Prozessliste) - nur zur Demonstration der Maskierung"
hole_secret db/passwort >/dev/null     # direkt aufrufen: in \$(...) ginge die Eintragung ins Array verloren
fehlertext="Fehler beim Login mit Passwort S3cret-Passw0rt! an db.test"
printf '%s\n' "$fehlertext" | maskiere

echo "== 3) Secret ueber stdin an ein Programm (kein Argument)"
printf '%s' "$(secretctl get db/passwort)" | { IFS= read -r pw || [[ -n $pw ]]; echo "gelesen: ${#pw} Zeichen"; }

echo "== 4) Log auf Leckagen pruefen"
{ echo "Start"; echo "Verbindung mit S3cret-Passw0rt! hergestellt"; echo "Ende"; } > "$t/pipeline.log"
if grep -qF -- "$(secretctl get db/passwort)" "$t/pipeline.log"; then
    echo "LECK: Geheimnis im Log gefunden" >&2
else echo "Log sauber"; fi

echo "== 5) set -x ausschalten, wenn Geheimnisse im Spiel sind"
( set -x; x=1; { set +x; } 2>/dev/null; echo "Ausgabe ohne Ablaufprotokoll" )
```

Der erste Abschnitt zeigt das Grundmuster: Das Geheimnis wird in einer Subshell gelesen und exportiert, sodass es nur dort und in deren Kindprozessen existiert; im Elternprozess ist die Variable nicht gesetzt (die Ausgabe belegt das). Die Funktion `hole_secret` merkt sich jeden Wert in einem Array, damit `maskiere` ihn später aus Texten entfernt, bevor man sie anzeigt oder speichert. Das ist eine zusätzliche Sicherung, kein Ersatz für Sorgfalt. Eine Eigenheit ist zu beachten: Läuft `hole_secret` innerhalb von `$(...)`, geschieht das in einer Subshell, und der Eintrag ins Array geht verloren; im zweiten Abschnitt wird die Funktion deshalb direkt aufgerufen. Abschnitt 3 zeigt die Weitergabe über die Standardeingabe, die in der Prozessliste nicht sichtbar ist. Abschnitt 4 prüft ein fertiges Log mit `grep -qF` (festes Muster, kein regulärer Ausdruck) auf das Geheimnis und schlägt Alarm, wenn es darin steht. Der letzte Abschnitt demonstriert den Trick, den Ablaufprotokoll-Modus mit `{ set +x; } 2>/dev/null` auszuschalten, ohne dass der Befehl `set +x` selbst im Log erscheint.

Typische Fehler: Erstens werden Geheimnisse mit `-e`, `-p` oder als Argument übergeben und tauchen in `ps` und in Shell-Verläufen auf. Zweitens läuft ein Skript mit `set -x` über einen Abschnitt mit Secrets, und der Wert steht im Log. Drittens wird davon ausgegangen, die Plattform maskiere *alles*; abgeleitete Werte (Base64, URL-kodiert, Teilstrings) bleiben sichtbar. Viertens liegen Secrets in der Dockerfile (`ENV`, `ARG`), in der Image-Historie oder in Terraform-State und -Plan. Fünftens existieren langlebige Cloud-Schlüssel als Pipeline-Secrets, obwohl OIDC-Anmeldung möglich wäre. Sechstens haben Pipeline-Schritte aus fremden Pull Requests Zugriff auf Secrets (bei offenen Repositories ein klassisches Einfallstor). Siebtens bleiben kompromittierte Secrets im Umlauf, weil niemand *rotiert* (tauscht): Jedes Secret braucht einen Verantwortlichen und einen Rotationsplan, und nach einem Leck gilt es als verbrannt, auch wenn das Log gelöscht wurde.

```bash
# falsch: Secret im Argument und im Ablaufprotokoll
set -x
mysql -h db -u app -p"$DB_PASSWORT" -e 'SELECT 1'

# richtig: aus Datei mit Rechten 600 lesen, Ablaufprotokoll aus, kein Secret in der Prozessliste
umask 077; cfg="$(mktemp)"; trap 'rm -f -- "$cfg"' EXIT
{ set +x; } 2>/dev/null
printf '[client]\npassword=%s\n' "$DB_PASSWORT" > "$cfg"
mysql --defaults-extra-file="$cfg" -h db -u app -e 'SELECT 1'
```

> **Merke**
>
> Keine langlebigen Schlüssel: OIDC/Workload Identity bevorzugen. Secrets aus Store oder Pipeline-Variable zur Laufzeit holen, nur an den Kindprozess weitergeben, per Stdin oder Datei 600 statt Argument. Kein `set -x` um Geheimnisse, Logs auf Leckagen prüfen, abgeleitete Werte maskieren, regelmäßig rotieren; nach Leck sofort tauschen. Siehe auch Modul 8.4.

> **Praxisbeispiel**
>
> Eine Pipeline meldet sich per OIDC bei der Cloud an und liest das Datenbankpasswort aus dem Secret Store. Im letzten Schritt durchsucht ein Skript die Log-Datei nach Mustern bekannter Schlüssel (zum Beispiel `AKIA...`-Präfixe oder `-----BEGIN PRIVATE KEY-----`) und lässt den Lauf scheitern, wenn es etwas findet. Der Lauf endet dann bewusst rot, und das Secret wird rotiert.

## 11.6 Umgebungs-Bootstrap

*Bootstrap* bedeutet, eine leere oder neue Umgebung in einen arbeitsfähigen Zustand zu bringen: die Entwicklungsumgebung auf einem neuen Laptop, ein frisches Testsystem, eine neue Produktionsregion. Ein guter Bootstrap-Ablauf ist ein einziges Skript (oder ein Aufruf), das aus „nichts“ eine definierte Umgebung herstellt, bei jeder Wiederholung zum selben Ergebnis führt (idempotent, Modul 7.7) und verständlich meldet, was fehlt. Wer so etwas nicht hat, kennt den Zustand aus Wikiseiten mit zwanzig Schritten, die niemand vollständig befolgt.

Der Ablauf besteht aus vier Phasen. In der ersten, den *Prerequisite Checks* (Voraussetzungsprüfungen), prüft das Skript, ob alles Nötige vorhanden ist, bevor es etwas verändert: erforderliche Programme (`command -v`) in ausreichenden Versionen, Berechtigungen, freier Speicherplatz, Netzzugang zu den benötigten Zielen, gesetzte Umgebungsvariablen. Alle Befunde werden *gesammelt* und gemeinsam ausgegeben, nicht beim ersten Fehler abgebrochen, damit man nicht nach jeder Korrektur den nächsten Fehler entdeckt. In der zweiten Phase wählt das Skript die *Umgebung* (dev, test, prod) aus einem Parameter und lädt die zugehörige Konfiguration (Modul 10.7). In der dritten Phase *richtet es ein*: Verzeichnisse, Konfigurationsdateien, Benutzer, Abhängigkeiten, Datenbankschemata. Jeder Schritt prüft zuerst, ob er schon erledigt ist. In der vierten Phase *verifiziert* es das Ergebnis (Smoke Test) und berichtet.

Wichtig ist die Unterscheidung der Umgebungen. Für Entwicklung darf das Skript freizügig sein (Beispieldaten, Debug-Optionen). Für Produktion verlangt es zusätzliche Bestätigungen, verbietet destruktive Schritte (“Datenbank neu anlegen”) und läuft nur mit passender Identität. Die Umgebungsunterschiede stehen in Konfigurationsdateien, nicht im Code (Modul 10.7), damit dasselbe Skript überall läuft.

Das Beispiel implementiert einen Bootstrap mit gesammelten Voraussetzungsprüfungen, Versionsvergleich, idempotenten Schritten und Verifikation. Es arbeitet in einem Testverzeichnis und installiert nichts im System.

```bash
#!/usr/bin/env bash
# bootstrap.sh UMGEBUNG ZIELVERZ  - Voraussetzungen pruefen, einrichten, verifizieren (idempotent)
set -euo pipefail
umgebung="${1:?Aufruf: $0 dev|test|prod ZIELVERZ}"; ziel="${2:?ZIELVERZ fehlt}"
case $umgebung in dev|test|prod) ;; *) echo "unbekannte Umgebung: $umgebung" >&2; exit 2 ;; esac

fehler=()
# --- Phase 1: Voraussetzungen sammeln (nichts veraendern)
brauche_befehl() { command -v "$1" >/dev/null 2>&1 || fehler+=("Programm fehlt: $1"); }
mindestversion() {                                # mindestversion NAME IST MIN  (z.B. bash 5.1 4.4)
    local name="$1" ist="$2" min="$3"
    [[ "$(printf '%s\n%s\n' "$min" "$ist" | sort -V | head -n 1)" == "$min" ]] || fehler+=("$name $ist ist aelter als $min")
}
brauche_befehl jq
brauche_befehl curl
brauche_befehl nichtvorhanden-xyz                 # absichtlich: zeigt die Sammlung von Befunden
mindestversion bash "${BASH_VERSINFO[0]}.${BASH_VERSINFO[1]}" 4.4
[[ -d $(dirname "$ziel") && -w $(dirname "$ziel") ]] || fehler+=("kein Schreibrecht in $(dirname "$ziel")")
frei_mb=$(df -Pm -- "$(dirname "$ziel")" | awk 'NR==2 {print $4}')
(( frei_mb >= 50 )) || fehler+=("zu wenig Speicher: ${frei_mb} MB frei, 50 MB noetig")
[[ $umgebung != prod || "${BESTAETIGT:-}" == prod ]] || fehler+=("prod verlangt BESTAETIGT=prod")

if (( ${#fehler[@]} > 0 )); then
    echo "Voraussetzungen nicht erfuellt (${#fehler[@]}):" >&2
    printf '  - %s\n' "${fehler[@]}" >&2
    if [[ ${NUR_PRUEFEN:-0} != 1 ]]; then exit 1; fi
fi

# --- Phase 2/3: einrichten, jeder Schritt idempotent
schritt() { echo "[$umgebung] $*"; }
schritt "Verzeichnisse"
mkdir -p -- "$ziel"/{config,daten,log}
schritt "Konfiguration schreiben (nur wenn noetig)"
cfg="$ziel/config/app.env"
inhalt="APP_ENV=$umgebung
LOGLEVEL=$([[ $umgebung == dev ]] && echo debug || echo info)
"
if [[ ! -f $cfg ]] || [[ "$(cat "$cfg")" != "${inhalt%$'\n'}" ]]; then
    printf '%s' "$inhalt" > "$cfg.neu" && mv -- "$cfg.neu" "$cfg"; schritt "  geschrieben"
else schritt "  unveraendert"; fi
if [[ $umgebung == dev && ! -f $ziel/daten/beispiel.json ]]; then
    echo '[{"id":1,"name":"Beispiel"}]' > "$ziel/daten/beispiel.json"; schritt "Beispieldaten angelegt (nur dev)"
fi

# --- Phase 4: verifizieren
jq -e . "$ziel/daten/beispiel.json" >/dev/null 2>&1 || [[ $umgebung != dev ]]
grep -q "^APP_ENV=$umgebung\$" "$cfg" && schritt "Verifikation bestanden"
```

Die Prüfphase sammelt alle Befunde in einem Array und gibt sie auf einmal aus; mit `NUR_PRUEFEN=1` fährt das Skript trotzdem fort, was für den Test der folgenden Phasen nützlich ist. Die Versionsprüfung nutzt `sort -V` (Versionssortierung): Steht die Mindestversion nach der Sortierung vorn, ist die vorhandene mindestens so neu. Die Einrichtung ist idempotent, weil jede Aktion zuerst prüft, ob sie nötig ist: `mkdir -p` ist es von Natur aus, die Konfiguration wird nur geschrieben, wenn sie fehlt oder abweicht, und zwar atomar über eine temporäre Datei und `mv`. Die Umgebung beeinflusst Inhalt (Loglevel) und Zusatzschritte (Beispieldaten nur in dev). Zum Schluss prüft die Verifikation, dass die Konfiguration zur Umgebung passt. Ein zweiter Aufruf mit denselben Parametern zeigt “unverändert” und ändert nichts.

Typische Fehler: Erstens prüft das Skript Voraussetzungen erst, wenn sie fehlen, und lässt eine halb eingerichtete Umgebung zurück. Zweitens bricht es beim ersten Befund ab, sodass man fünfmal neu starten muss. Drittens ist der Ablauf nicht wiederholbar: Der zweite Lauf scheitert, weil ein Verzeichnis existiert oder ein Benutzer schon angelegt ist. Viertens behandelt man Produktion wie Entwicklung. Fünftens sind Versionsvergleiche Zeichenfolgenvergleiche (`"5.10" < "5.9"` ist wahr); `sort -V` oder numerischer Vergleich der Teile löst das. Sechstens installiert man Abhängigkeiten ohne feste Versionen. Siebtens fehlt die Verifikation, und „fertig“ bedeutet nur „kein Fehler aufgetreten“.

```bash
# falsch: Zeichenvergleich, harter Abbruch beim ersten Fehler, nicht wiederholbar
[[ $(bash --version | head -1 | awk '{print $4}') > "4.4" ]] || exit 1
mkdir /srv/app
useradd app

# richtig: Versionssortierung, idempotent
[[ "$(printf '%s\n%s\n' 4.4 "${BASH_VERSINFO[0]}.${BASH_VERSINFO[1]}" | sort -V | head -n 1)" == 4.4 ]] || echo "Bash zu alt"
mkdir -p /srv/app
id app >/dev/null 2>&1 || useradd --system app
```

> **Merke**
>
> Bootstrap in vier Phasen: Voraussetzungen sammeln, Umgebung wählen, idempotent einrichten, verifizieren. Alle Befunde auf einmal melden, Versionen mit `sort -V` vergleichen, Produktion besonders absichern, Unterschiede in Konfigurationsdateien statt Code.

> **Praxisbeispiel**
>
> Neue Teammitglieder führen `./bootstrap.sh dev ~/arbeit` aus. Fehlt `jq` oder ist die Bash-Version zu alt, erscheint eine vollständige Liste mit Installationshinweisen; sonst steht nach einer Minute eine laufende Entwicklungsumgebung bereit. Derselbe Ablauf mit `test` baut die Testumgebung in der CI auf.

## 11.7 Policy und Compliance Checks

Eine *Policy* ist eine Regel, die eine Organisation für ihre Systeme festlegt: “Container laufen nicht als root”, “Images tragen nie das Tag `latest`”, “Speicher-Buckets sind nicht öffentlich”, “jede Ressource hat einen Besitzer-Tag”, “Dateien mit Zugangsdaten sind nur für den Besitzer lesbar”. *Compliance* bedeutet, dass die Systeme diese Regeln einhalten, und zwar nachweisbar. Je früher eine Abweichung auffällt, desto billiger ist die Korrektur: beim Schreiben (Pre-Commit, Modul 10.3) kostet sie Sekunden, in der Pipeline Minuten, in Produktion nach einem Vorfall Tage. Man spricht vom “Shift Left”: Prüfungen wandern an den Anfang des Prozesses.

Für Standardprüfungen genügen Shell-Werkzeuge erstaunlich oft: `grep` und `jq` für Konfigurationsdateien, `stat` und `find` für Dateirechte, die Ausgaben von `terraform show -json`, `kubectl get -o json` oder den Cloud-CLIs für Zustände. Daneben gibt es spezialisierte Policy-Werkzeuge (zum Beispiel Open Policy Agent mit Conftest, Checkov oder tfsec für IaC, Trivy für Images, ShellCheck für Skripte), die umfangreiche Regelwerke mitbringen. Bash ist der Rahmen: Es ruft sie auf, sammelt die Ergebnisse in einem einheitlichen Bericht und übersetzt sie in einen Exit-Code. Wer nur drei oder vier Regeln braucht, schreibt sie selbst; wer ein ganzes Regelwerk braucht, setzt ein Werkzeug ein und klebt es an.

Eine gute Prüfung folgt festen Grundsätzen. Jede Regel hat eine *Kennung* (“POL-003”), eine *Schwere* (hoch, mittel, niedrig), einen *klaren Text* und einen *Hinweis zur Behebung*. Das Skript meldet *alle* Verstöße in einem Lauf, nicht nur den ersten. Es unterscheidet zwischen blockierenden Verstößen (Exit-Code ungleich 0) und Warnungen. Ausnahmen sind erlaubt, aber *dokumentiert* und befristet (ein Vermerk mit Begründung und Ablaufdatum im Code), sonst werden Regeln durch Gewohnheit untergraben. Und die Prüfung muss selbst getestet sein: Für jede Regel gibt es ein Beispiel, das sie verletzt, und eins, das sie erfüllt.

Das Beispiel prüft ein Verzeichnis mit drei Dateiarten, einem Kubernetes-ähnlichen Manifest, einer Storage-Konfiguration als JSON und einer Datei mit Zugangsdaten. Es zeigt Regeln mit Kennung, Schwere und Behebungshinweis, eine dokumentierte Ausnahme und einen Bericht, dessen Exit-Code nur bei hohen Verstößen auf 1 springt.

```bash
#!/usr/bin/env bash
# policy-check.sh VERZ  - Standardpruefungen mit Kennung, Schwere, Behebung und Ausnahmen
set -uo pipefail
verz="${1:?Aufruf: $0 VERZ}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }
hoch=0; mittel=0
melde() {                                         # melde KENNUNG SCHWERE DATEI TEXT HINWEIS
    local id="$1" sw="$2" datei="$3" text="$4" hinweis="$5"
    # dokumentierte Ausnahme: Zeile "# policy-ausnahme: ID bis JJJJ-MM-TT Begruendung" in der Datei
    local ausn
    ausn="$(grep -E "^# policy-ausnahme: $id bis [0-9]{4}-[0-9]{2}-[0-9]{2} " "$datei" 2>/dev/null | head -n 1 || true)"
    if [[ -n $ausn ]]; then
        local bis; bis="$(awk '{print $5}' <<< "$ausn")"
        if [[ "$bis" > "$(date +%F)" || "$bis" == "$(date +%F)" ]]; then
            printf '[AUSNAHME ] %s %s: %s (bis %s)\n' "$id" "${datei#"$verz"/}" "$text" "$bis"; return
        fi
        text="$text (Ausnahme abgelaufen am $bis)"
    fi
    printf '[%-9s] %s %s: %s\n           Behebung: %s\n' "${sw^^}" "$id" "${datei#"$verz"/}" "$text" "$hinweis"
    case $sw in hoch) hoch=$((hoch + 1)) ;; mittel) mittel=$((mittel + 1)) ;; esac
}

while IFS= read -r -d '' f; do
    case $f in
    *.yaml|*.yml)
        # POL-001: kein :latest-Tag
        grep -Eq '^\s*(-\s+)?image:\s*\S+:latest\s*$' "$f" && melde POL-001 mittel "$f" "Image mit Tag :latest" "festes Tag oder Digest verwenden"
        # POL-002: kein Container als root
        grep -Eq '^\s*runAsNonRoot:\s*true' "$f" || melde POL-002 hoch "$f" "runAsNonRoot nicht gesetzt" "securityContext.runAsNonRoot: true ergaenzen"
        # POL-003: Ressourcengrenzen
        grep -Eq '^\s*limits:' "$f" || melde POL-003 mittel "$f" "keine Ressourcenlimits" "resources.limits fuer cpu und memory setzen"
        ;;
    *.json)
        # POL-004: Speicher darf nicht oeffentlich sein
        jq -e '.. | objects | select(.publicAccess? == true)' "$f" >/dev/null 2>&1 && melde POL-004 hoch "$f" "Speicher oeffentlich zugaenglich" "publicAccess auf false setzen"
        # POL-005: Besitzer-Tag
        jq -e '.tags.owner? // empty' "$f" >/dev/null 2>&1 || melde POL-005 mittel "$f" "Tag owner fehlt" "Tag owner mit Team oder Person setzen"
        ;;
    *.env|*.secret)
        # POL-006: Dateirechte fuer Zugangsdaten
        modus="$(stat -c '%a' "$f")"
        [[ $modus =~ ^[0-7]00$ ]] || melde POL-006 hoch "$f" "Rechte $modus statt 600 oder 400" "chmod 600 DATEI"
        ;;
    esac
done < <(find "$verz" -type f -print0 | sort -z)

echo "Zusammenfassung: $hoch hoch, $mittel mittel"
(( hoch == 0 ))
```

Die Funktion `melde` bündelt alles, was für eine einheitliche Meldung nötig ist: Kennung, Schwere, Datei, Text und Behebung. Sie sucht in der betroffenen Datei nach einem Ausnahmevermerk und vergleicht das Ablaufdatum als Zeichenfolge, was bei ISO-Datumsangaben (`JJJJ-MM-TT`) korrekt funktioniert, weil die alphabetische Reihenfolge der zeitlichen entspricht. Eine abgelaufene Ausnahme gilt nicht mehr und wird als Verstoß mit Zusatz gemeldet. Die Hauptschleife liest Dateien nullbyte-getrennt aus `find` und wählt die Regeln nach der Dateiendung. Man sieht außerdem, wie verschiedene Werkzeuge zum Einsatz kommen: `grep` für textuelle Muster, `jq` für Strukturen in JSON (der rekursive Operator `..` findet das Feld in beliebiger Tiefe) und `stat` für Dateirechte. Am Ende steht der Exit-Code 0, wenn es keine hohen Verstöße gab, sonst 1. Mittlere Verstöße werden gemeldet, blockieren aber nicht. Zum Ausprobieren legen Sie ein Verzeichnis mit einer Manifestdatei ohne `runAsNonRoot`, einer JSON-Datei mit `"publicAccess": true` und einer `.env`-Datei mit Rechten 644 an.

Typische Fehler: Erstens prüft man nur textuell und übersieht Varianten (`image: "x:latest"` mit Anführungszeichen, anderes Einrückungsformat); strukturierte Prüfungen mit `yq` oder `jq` auf geparsten Daten sind robuster, und Textprüfungen sind eine pragmatische Annäherung, deren Grenzen man kennen muss. Zweitens bricht das Skript beim ersten Verstoß ab, und man arbeitet Fehler einzeln ab. Drittens gibt es Ausnahmen ohne Ablaufdatum, die für immer gelten. Viertens sind Regeln nicht getestet und liefern zu viele falsche Alarme, bis das Team sie ignoriert. Fünftens prüft die Pipeline nur nach dem Zusammenführen statt davor. Sechstens vermischt man Warnung und Blockade und stoppt Releases wegen Kleinigkeiten oder, schlimmer, lässt echte Verstöße durch, weil alles nur eine Warnung ist. Siebtens vergisst man, dass eine bestandene Prüfung keine Sicherheit *beweist*; sie belegt nur, dass die geprüften Regeln erfüllt sind.

```bash
# falsch: erster Fehler stoppt, Ausnahme ohne Frist, nur Text im Fehlerfall
grep -q 'latest' deployment.yaml && { echo "Fehler"; exit 1; }
# policy-ausnahme: latest erlaubt

# richtig: alle Verstoesse sammeln, ID + Behebung, befristete Ausnahme
# policy-ausnahme: POL-001 bis 2027-03-31 Legacy-Dienst, Migration geplant (Ticket OPS-412)
melde POL-001 mittel deployment.yaml "Image mit Tag :latest" "festes Tag oder Digest verwenden"
```

> **Merke**
>
> Policy-Prüfungen: Regeln mit Kennung, Schwere und Behebungstext; alle Verstöße in einem Lauf melden; blockierend nur, was blockieren soll. Ausnahmen dokumentiert und befristet. Strukturierte Daten mit `jq` prüfen, Textmuster nur als Annäherung. Regeln selbst testen. Spezialwerkzeuge einbinden statt neu erfinden.

> **Praxisbeispiel**
>
> Der Pull Request für Infrastruktur-Änderungen startet `policy-check.sh`. Ein Verstoß “Speicher öffentlich” blockiert das Zusammenführen mit einer klaren Meldung und Korrekturhinweis; fehlende Tags erscheinen als Warnung im Kommentar. Einmal im Quartal prüft das Team die offenen Ausnahmen auf Ablauf.

## 11.8 Portabilität und Runner

Ein Skript, das auf dem eigenen Laptop läuft, läuft nicht automatisch auf dem CI-Runner, dem Server oder dem Rechner der Kollegin. Gründe sind verschiedene *Linux-Distributionen* (Debian, Ubuntu, Red Hat, Alpine), verschiedene *Shell-Versionen* (Bash 3.2 auf macOS, 5.x auf aktuellen Linux-Systemen; auf Alpine oft nur BusyBox `ash`) und verschiedene *Werkzeugvarianten* (GNU- gegen BSD-Utilities). Portabilität heißt, solche Unterschiede bewusst zu kennen, zu prüfen und zu kapseln. Ein vollständig portables Skript gibt es selten; sinnvoll ist, die unterstützten Umgebungen festzulegen und genau diese zu testen.

Die häufigsten Stolpersteine lassen sich benennen. Bash 3.2 (macOS) kennt keine assoziativen Arrays (Bash 4), kein `mapfile`/`readarray` (4.0), keine Kleinschreibungs-Umwandlung `${var,,}` (4.0) und kein `wait -n` (4.3). Bei den Werkzeugen unterscheiden sich unter anderem `sed -i` (GNU: ohne Argument, BSD: verlangt eine Endung), `date -d` (GNU; BSD nutzt `-j -f`), `readlink -f` (GNU; BSD erst seit neueren Versionen), `grep -P` (nicht überall), `stat -c` (GNU) gegen `stat -f` (BSD), `xargs -r` und `sort -V`. Auf BusyBox fehlen einige Optionen und Programme ganz. Bei Skripten für `#!/bin/sh` gilt zusätzlich: nur POSIX-Syntax, keine `[[ ]]`, keine Arrays, kein `local` (auch wenn die meisten Shells es anbieten).

Es gibt drei Strategien. Erstens *Abgrenzen*: Man dokumentiert und prüft die Mindestversionen (Bash ab 4.4, GNU coreutils) und bricht früh mit klarer Meldung ab, wenn sie fehlen (wie im Bootstrap, Einheit 11.6). Zweitens *Kapseln*: Plattformabhängiges steckt in kleinen Funktionen mit einheitlicher Schnittstelle (`sed_inplace`, `datei_alter`), sodass der Rest des Skripts davon nichts weiß. Drittens *Isolieren*: Man führt das Skript in einem Container mit festgelegten Werkzeugversionen aus (Einheit 11.3), dann ist die Umgebung identisch.

Für *CI-Runner* kommt Reproduzierbarkeit hinzu. Runner-Images ändern sich: Ein Label wie `ubuntu-latest` verweist irgendwann auf eine neue Hauptversion, vorinstallierte Werkzeuge werden aktualisiert, und eine Pipeline, die gestern grün war, wird ohne Codeänderung rot. Reproduzierbar wird sie durch feste Runner-Versionen statt `latest`, ein dokumentiertes und von der Pipeline selbst installiertes Werkzeugset mit festen Versionen (zum Beispiel eine Datei `tools.lock` oder ein eigenes Container-Image), eine Eingangsprüfung, die Ist- und Soll-Versionen vergleicht, und einen Zwischenspeicher (Cache), dessen Schlüssel von den Versionen abhängt. Ein Prüfskript am Anfang jedes Laufs, das die Umgebung ausgibt und mit der Vorgabe vergleicht, macht Abweichungen sofort sichtbar.

```bash
#!/usr/bin/env bash
# portabel.sh - Umgebung erkennen, Mindestversionen pruefen, Plattformunterschiede kapseln
set -euo pipefail

# --- 1. Umgebung erfassen (fuer jedes CI-Log sinnvoll)
os="$(uname -s)"
distro="unbekannt"
if [[ -r /etc/os-release ]]; then distro="$(. /etc/os-release; echo "${PRETTY_NAME:-$ID}")"; fi
echo "System: $os / $distro / $(uname -m)"
echo "Bash:   $BASH_VERSION"

# --- 2. Gewuenschte Werkzeuge und Mindestversionen (wie eine kleine tools.lock)
tools_soll=$(cat <<'ENDE'
bash 4.4
jq 1.6
curl 7.0
ENDE
)
version_von() {                                   # version_von WERKZEUG -> erste Versionsnummer der Ausgabe
    case $1 in
        bash) printf '%s.%s\n' "${BASH_VERSINFO[0]}" "${BASH_VERSINFO[1]}" ;;
        jq)   jq --version 2>/dev/null | sed -E 's/^jq-//' ;;
        *)    "$1" --version 2>/dev/null | head -n 1 | grep -Eo '[0-9]+(\.[0-9]+)+' | head -n 1 ;;
    esac
}
mind() { [[ "$(printf '%s\n%s\n' "$2" "$1" | sort -V | head -n 1)" == "$2" ]]; }   # mind IST SOLL

abweichungen=0
while read -r name soll; do
    ist="$(version_von "$name" || true)"
    if [[ -z $ist ]]; then echo "FEHLT    $name (soll >= $soll)"; abweichungen=$((abweichungen + 1))
    elif mind "$ist" "$soll"; then echo "ok       $name $ist (soll >= $soll)"
    else echo "ZU ALT   $name $ist (soll >= $soll)"; abweichungen=$((abweichungen + 1)); fi
done <<< "$tools_soll"

# --- 3. Plattformunterschiede kapseln
sed_inplace() {                                   # sed_inplace AUSDRUCK DATEI  (GNU und BSD)
    if sed --version >/dev/null 2>&1; then sed -i -e "$1" "$2"; else sed -i '' -e "$1" "$2"; fi
}
datei_alter_s() {                                 # Alter einer Datei in Sekunden (GNU stat oder BSD stat)
    local m
    if m="$(stat -c %Y -- "$1" 2>/dev/null)"; then :; else m="$(stat -f %m -- "$1")"; fi
    echo $(( $(date +%s) - m ))
}
kleinbuchstaben() {                               # ${var,,} gibt es erst ab Bash 4; tr geht ueberall
    printf '%s' "$1" | tr '[:upper:]' '[:lower:]'
}

t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
echo "Wert=ALT" > "$t/datei"
sed_inplace 's/ALT/NEU/' "$t/datei"
echo "sed_inplace: $(cat "$t/datei")"
echo "Alter der Datei: $(datei_alter_s "$t/datei") s"
echo "Kleinschreibung: $(kleinbuchstaben 'GROSS-und-klein')"

(( abweichungen == 0 )) || { echo "$abweichungen Abweichung(en) von der Werkzeugvorgabe" >&2; exit 1; }
echo "Umgebung entspricht der Vorgabe"
```

Der erste Abschnitt druckt die Umgebung aus, denn die wichtigste Hilfe bei “bei mir geht es, auf dem Runner nicht” ist ein Log, das System, Distribution und Bash-Version nennt. Der zweite Abschnitt vergleicht Ist- und Sollversionen. Die Sollwerte stehen als Text im Skript (in der Praxis in einer eigenen Datei im Repository), und die Funktion `version_von` liest die Version je Werkzeug; Bash verrät sie über `BASH_VERSINFO`, `jq` mit führendem `jq-`, andere Programme meist als erste Zahl ihrer `-{}-version`-Ausgabe. `mind` nutzt `sort -V` für den Vergleich. Der dritte Abschnitt kapselt Unterschiede: `sed_inplace` erkennt GNU-`sed` an der Option `-{}-version` (BSD-`sed` kennt sie nicht), `datei_alter_s` probiert erst GNU-, dann BSD-`stat`, und `kleinbuchstaben` nutzt `tr`, das überall gleich funktioniert. Wichtig ist, dass die Rückgabewerte nur dort geprüft werden, wo ein Fehlschlag erwartet ist (die erste `stat`-Variante), und dass Abweichungen gesammelt, nicht beim ersten Fehler abgebrochen, gemeldet werden. Hinweis zur Prüfung: Dieser Ablauf wurde hier nur unter Linux (GNU-Werkzeuge) ausgeführt; die BSD-Zweige sind nicht getestet.

Typische Fehler: Erstens nimmt man an, die eigene Bash-Version sei überall vorhanden, und nutzt `mapfile` oder assoziative Arrays in Skripten, die auf macOS laufen sollen. Zweitens verwendet man `sed -i` ohne Rücksicht auf die BSD-Variante. Drittens schreibt man `#!/bin/sh`, benutzt aber Bash-Funktionen; auf Debian ist `sh` `dash`, und es kommt zu Syntaxfehlern. Viertens fehlt eine Prüfung, sodass das Skript mit rätselhaften Meldungen irgendwo in der Mitte scheitert. Fünftens nutzt man `ubuntu-latest` oder `:latest`-Images und wundert sich über plötzliche Ausfälle. Sechstens installiert die Pipeline Werkzeuge ohne feste Version, sodass jeder Lauf andere Versionen holt. Siebtens unterscheiden sich Zeitzone, Sprache und Zeichencodierung (`LC_ALL`) zwischen Laptop und Runner; Sortierreihenfolgen und Zahlenformate ändern sich dadurch. Sie sollten in Skripten, bei denen es darauf ankommt, ausdrücklich gesetzt werden (`LC_ALL=C`), wie in Modul 9.6 gezeigt.

```bash
# falsch: GNU-spezifisch ohne Pruefung, floating Runner und Versionen
runs-on: ubuntu-latest
steps:
  - run: sudo apt-get install -y jq && ./skript.sh      # 'sed -i' im Skript bricht auf macOS

# richtig: fester Runner, feste Versionen, Eingangspruefung, Plattformfunktionen
runs-on: ubuntu-24.04
steps:
  - run: ./ci/pruefe-umgebung.sh            # vergleicht Ist/Soll aus tools.lock und bricht bei Abweichung ab
  - run: ./skript.sh                         # nutzt sed_inplace() statt sed -i
```

> **Merke**
>
> Portabel heißt: unterstützte Umgebungen festlegen, Mindestversionen prüfen, Plattformunterschiede (GNU/BSD, Bash 3 gegen 5, `sh` gegen `bash`) in Funktionen kapseln. Runner reproduzierbar machen: feste Runner-Versionen, festes Werkzeugset (Lockdatei oder Container-Image), Umgebungsausgabe im Log, Eingangsprüfung.

> **Praxisbeispiel**
>
> Die Pipeline beginnt mit `./ci/pruefe-umgebung.sh`: Es druckt Betriebssystem, Bash- und Werkzeugversionen und bricht ab, wenn sie von der `tools.lock` abweichen. Als ein Runner-Update plötzlich eine neuere `jq`-Version brachte, zeigte der erste Schritt die Abweichung sofort, anstatt dass irgendwo ein Filter unerklärlich anders arbeitete.

## 11.9 Übungsaufgaben zu Modul 11

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit. Für die praktischen Aufgaben genügen kleine Attrappen der Werkzeuge (Skripte, die feste Ausgaben liefern); ein Cloud-Konto oder Cluster ist nicht nötig.

**Aufgabe 1.** Warum sind `-o json` und `jq` für Skripte besser als Tabellenausgabe? Nennen Sie drei Gründe, warum ein Cloud-Skript Konto und Region zu Beginn prüfen sollte, und beschreiben Sie, was Exponential Backoff bedeutet.

**Aufgabe 2.** Was bedeuten die Exit-Codes 0, 1 und 2 bei `terraform plan` mit der Option `-detailed-exitcode`, und warum ist es wichtig, den Plan zu speichern und genau diesen anzuwenden? Was ist Drift?

**Aufgabe 3.** Nennen Sie fünf Optionen von `docker run`, die ein Prüfskript sicherer machen, und erklären Sie jeweils in einem Satz, wovor sie schützen. Warum darf man das Docker-Socket nicht in einen Container einbinden?

**Aufgabe 4.** Warum sind Secrets in Befehlszeilenargumenten problematisch, und welche drei sichereren Wege kennen Sie? Warum reicht die Maskierung durch die CI-Plattform allein nicht?

**Aufgabe 5.** Schreiben Sie `cloud-inventar.sh DATEI`, das eine JSON-Datei mit VMs (Felder `name`, `state`, `tags`) einliest und ausgibt: (a) Anzahl je Zustand, (b) Namen aller laufenden VMs ohne Tag `owner`, (c) mit Exit-Code 1, wenn es solche VMs gibt. Verwenden Sie `jq` und prüfen Sie die Eingabedatei auf gültiges JSON.

**Aufgabe 6.** Schreiben Sie `drift.sh`, das einen Befehl aus der Variable `PLAN_BEFEHL` (Attrappe: Skript, das mit 0, 1 oder 2 endet) ausführt, den Exit-Code auswertet und mit einer Meldung sowie eigenem Code antwortet (0 = “kein Drift”, 10 = “Drift”, 20 = “Fehler”). Das Skript darf nicht durch `set -e` abbrechen.

**Aufgabe 7.** Schreiben Sie `sicher-kubectl.sh CONTEXT NAMESPACE ARGS...`, das (a) den Context gegen die Liste `dev,test` prüft und `prod` nur mit gesetztem `BESTAETIGT=prod` zulässt, (b) den Namespace per regulärem Ausdruck validiert und (c) `kubectl` immer mit `-{}-context` und `-{}-namespace` aufruft. Testen Sie mit einer Attrappe, die ihre Argumente ausgibt.

**Aufgabe 8.** Schreiben Sie `log-scan.sh LOGDATEI`, das ein Pipeline-Log nach Geheimnismustern durchsucht (`AKIA` gefolgt von 16 Großbuchstaben oder Ziffern, `-----BEGIN [A-Z ]*PRIVATE KEY-----`, `password=` mit nichtleerem Wert), jeden Treffer mit Zeilennummer meldet, aber den gefundenen Wert nur gekürzt (erste 4 Zeichen und `***`) ausgibt und mit Exit-Code 1 endet, wenn es Treffer gibt.

## 11.10 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** JSON ist strukturiert und stabil, Tabellen sind für Menschen formatiert und ändern sich mit Version und Terminalbreite; `jq` wertet Felder gezielt aus, ohne Spalten zu zählen. Konto und Region prüft man, weil (1) sonst gegen die falsche Umgebung, womöglich Produktion, gearbeitet wird, (2) Ressourcen in unerwarteten Regionen Kosten und Rechtsprobleme verursachen, (3) ein abgelaufener oder fehlender Login sonst erst mitten im Lauf scheitert. Exponential Backoff: Nach einem vorübergehenden Fehler wartet man 1, 2, 4, 8 Sekunden vor jedem weiteren Versuch, begrenzt die Zahl der Versuche und gibt dem überlasteten Dienst Zeit zur Erholung.

**Lösung zu Aufgabe 2.** 0: erfolgreich, keine Änderungen; 1: Fehler; 2: erfolgreich, Änderungen anstehen. Der gespeicherte Plan stellt sicher, dass genau das angewendet wird, was geprüft und freigegeben wurde; ein neuer Plan beim `apply` könnte inzwischen andere Änderungen enthalten. Drift ist die Abweichung zwischen realer Infrastruktur und Code, etwa durch manuelle Änderungen.

**Lösung zu Aufgabe 3.** `-{}-rm` (Container wird gelöscht, keine Altlasten), `-{}-user` (kein root, Dateirechte stimmen), `-{}-read-only` (Dateisystem unveränderbar), `-{}-network none` (kein Netzzugang für Abfluss oder Nachladen), `-{}-memory`/`-{}-cpus` (begrenzt Ressourcen), außerdem `-v ...:ro` (nur lesender Zugriff). Das Docker-Socket erlaubt, beliebige Container zu starten, auch privilegierte mit Zugriff auf den Host; wer es hat, hat praktisch Root auf dem Rechner.

**Lösung zu Aufgabe 4.** Argumente sind für alle lokalen Benutzer in der Prozessliste sichtbar und landen in Shell-Verlauf und Logs. Sichere Wege: Standardeingabe (`-{}-password-stdin`), Datei mit Rechten 600 in einem flüchtigen Verzeichnis, Umgebungsvariable nur für den Kindprozess; besser noch kurzlebige Zugangsdaten per OIDC. Die Plattform maskiert nur exakte Werte; abgeleitete (Base64, URL-kodiert, Teilstücke, in JSON eingebettete Werte) bleiben sichtbar, und `set -x` oder Fehlermeldungen können Werte offenlegen.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# cloud-inventar.sh DATEI
set -euo pipefail
datei="${1:?Aufruf: $0 DATEI}"
jq -e 'type == "array"' "$datei" >/dev/null 2>&1 || { echo "keine gueltige JSON-Liste: $datei" >&2; exit 2; }

echo "== Anzahl je Zustand"
jq -r 'group_by(.state) | .[] | "\(.[0].state): \(length)"' "$datei"

echo "== Laufende VMs ohne Tag owner"
ohne="$(jq -r '.[] | select(.state == "running" and ((.tags.owner // "") == "")) | .name' "$datei")"
if [[ -n $ohne ]]; then
    printf '%s\n' "$ohne"
    exit 1
fi
echo "(keine)"
```

Die Prüfung `jq -e 'type == "array"'` validiert zugleich die Syntax und die Struktur. `group_by` sortiert und gruppiert; der Ausdruck `(.tags.owner // "")` liefert einen leeren Text, wenn das Tag oder gar `tags` fehlt. Das Ergebnis wird in einer Variablen gesammelt, damit der Exit-Code vom Inhalt abhängt.

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# drift.sh - PLAN_BEFEHL ausfuehren und Exit-Code auswerten
set -uo pipefail
befehl="${PLAN_BEFEHL:?PLAN_BEFEHL setzen}"
rc=0
"$befehl" || rc=$?
case $rc in
    0) echo "kein Drift"; exit 0 ;;
    2) echo "Drift: Infrastruktur weicht vom Code ab" >&2; exit 10 ;;
    *) echo "Fehler beim Plan (rc=$rc)" >&2; exit 20 ;;
esac
```

Wichtig ist die Zeile `"$befehl" || rc=$?`: Sie erfasst den Code, auch wenn er ungleich 0 ist. Alternativ lässt man `set -e` weg, wie hier (es steht nur `-uo pipefail`). Test: `printf '#!/bin/sh\nexit 2\n' > plan.sh; chmod +x plan.sh; PLAN_BEFEHL=./plan.sh ./drift.sh; echo $?` ergibt 10.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# sicher-kubectl.sh CONTEXT NAMESPACE KUBECTL-ARGUMENTE...
set -euo pipefail
ctx="${1:?Aufruf: $0 CONTEXT NAMESPACE ARGS...}"; ns="${2:?NAMESPACE fehlt}"; shift 2
(( $# >= 1 )) || { echo "kubectl-Argumente fehlen" >&2; exit 2; }

case $ctx in
    dev|test) ;;
    prod) [[ "${BESTAETIGT:-}" == prod ]] || { echo "prod verlangt BESTAETIGT=prod" >&2; exit 3; } ;;
    *) echo "Context nicht erlaubt: $ctx" >&2; exit 2 ;;
esac
[[ $ns =~ ^[a-z0-9]([-a-z0-9]*[a-z0-9])?$ ]] || { echo "ungueltiger Namespace: $ns" >&2; exit 2; }

exec kubectl --context "$ctx" --namespace "$ns" "$@"
```

Die Allowlist per `case` lässt nur bekannte Contexts durch; der Namespace wird gegen das Namensformat von Kubernetes geprüft (Kleinbuchstaben, Ziffern, Bindestriche), was auch Injektion über Sonderzeichen ausschließt. `exec` ersetzt die Shell durch `kubectl`, sodass dessen Exit-Code unverändert zurückkommt. Test mit einer Attrappe `kubectl` im `PATH`, die `echo "kubectl $@"` ausgibt.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# log-scan.sh LOGDATEI
set -uo pipefail
log="${1:?Aufruf: $0 LOGDATEI}"
[[ -r $log ]] || { echo "nicht lesbar: $log" >&2; exit 2; }
treffer=0
muster=(
    'AKIA[A-Z0-9]{16}'
    '-----BEGIN [A-Z ]*PRIVATE KEY-----'
    'password=[^[:space:]]+'
)
for m in "${muster[@]}"; do
    while IFS=: read -r nr rest; do
        wert="$(grep -Eo -- "$m" <<< "$rest" | head -n 1)"
        printf 'Zeile %s: moegliches Geheimnis (%s***)\n' "$nr" "${wert:0:4}"
        treffer=$((treffer + 1))
    done < <(grep -nE -- "$m" "$log" || true)
done
(( treffer == 0 )) && echo "keine Geheimnismuster gefunden"
(( treffer == 0 ))
```

Das Skript sucht jedes Muster mit `grep -nE`, das Zeilennummer und Zeile liefert; die Zeile wird beim ersten Doppelpunkt getrennt. Der gefundene Wert wird auf vier Zeichen gekürzt und mit `***` ergänzt, damit der Bericht selbst kein Geheimnis ausplaudert. Die Endzeile liefert den Exit-Code 1 bei Treffern. Zeilen mit Doppelpunkten im Inhalt bleiben dank `read -r nr rest` vollständig in `rest`. Test: Eine Logdatei mit der Zeile `Verbinde mit password=geheim123` ergibt einen Treffer mit `pass***`.
