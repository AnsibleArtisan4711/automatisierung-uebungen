# Modul 07 - Systemadministration mit Bash

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Listings unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Überschriften sind wie in der Quelle nach Modul und Lerneinheit nummeriert (7.1 bis 7.9; 7.10 Übungsaufgaben, 7.11 Lösungen). **Die Beispiele wurden für dieses Repository nicht ausgeführt**, nur auf Bash-Syntax geprüft (`bash -n`, GNU bash 5.2); laut Einleitung hat der Autor sie ausgeführt, soweit sie ohne Spezialumgebung laufen. Viele Beispiele legen Dateien an, löschen oder verändern Systemzustand: nur in einem Testverzeichnis oder einer Testumgebung ausprobieren. Übersicht zum Modul: [README](README.md).

**Modulübersicht**

**Dauer:** 12 h laut Blatt „Lehrplan in Modulen“, 9 h laut Summe der Einzeleinheiten im Blatt „Lehrplan in Details“ (9 Einheiten à 1 h). Die Angaben weichen um drei Stunden ab; als Maßstab für den Umfang dient der höhere Wert (12 h).  
**Priorität:** 2 (mittel)

| Nr. | Lerneinheit                    | Dauer (h) | Prio |
|:----|:-------------------------------|----------:|:----:|
| 7.1 | Benutzer und Gruppen           |         1 |  2   |
| 7.2 | Pakete und Repositories        |         1 |  2   |
| 7.3 | Services mit systemd           |         1 |  2   |
| 7.4 | Dateisystem-Wartung            |         1 |  2   |
| 7.5 | Netzwerkdiagnose               |         1 |  2   |
| 7.6 | Backup-Skripte                 |         1 |  2   |
| 7.7 | Konfigurationsmanagement light |         1 |  3   |
| 7.8 | Remote Administration          |         1 |  3   |
| 7.9 | Patch- und Wartungsfenster     |         1 |  3   |
|     | **Summe (Details)**            |     **9** |      |

Systemadministration ist das klassische Einsatzgebiet der Shell: Konten anlegen, Software installieren, Dienste steuern, Platten aufräumen, Netzwerkprobleme eingrenzen, Daten sichern. Fast jede dieser Aufgaben lässt sich von Hand erledigen, aber erst als Skript wird sie *wiederholbar*, *prüfbar* und *dokumentiert*. Die Beispiele dieses Moduls folgen einem gemeinsamen Muster, das Sie aus den Modulen 6 und 8 kennen: Eingaben prüfen, den Ist-Zustand feststellen, nur bei Abweichung handeln (Idempotenz), Änderungen melden und im Zweifel zuerst einen Trockenlauf anbieten. Die Beispiele laufen in Testverzeichnissen oder simulieren gefährliche Schritte; wo ein Beispiel echte Systemänderungen vornimmt (Benutzer anlegen), räumt es hinterher auf. Das Modul geht von Debian/Ubuntu-nahen Systemen aus und nennt die Entsprechungen für Red Hat (dnf) und SUSE (zypper), wo sie sich unterscheiden.

## 7.1 Benutzer und Gruppen

Jeder Prozess und jede Datei gehört auf einem Linux-System einem *Benutzer* (User) und einer *Gruppe*. Die Konten stehen in den Dateien `/etc/passwd` (Name, numerische UID, Home-Verzeichnis, Login-Shell), `/etc/shadow` (Passwort-Hashes, nur für root lesbar) und `/etc/group` (Gruppen und ihre Mitglieder). Man bearbeitet sie nicht von Hand, sondern mit den Werkzeugen `useradd`, `usermod`, `userdel` sowie `groupadd`, `groupmod`, `groupdel`, die Sperren beachten und die Dateien konsistent halten. Zum *Abfragen* nimmt man nicht `grep` auf diese Dateien, sondern `id` und `getent`: `getent passwd anna` fragt die gesamte Namensauflösung ab, also auch LDAP oder SSSD, wenn das System daran angebunden ist.

Das *Least-Privilege-Prinzip* (Modul 8.5) prägt die Kontenverwaltung: Dienstkonten bekommen keine Login-Shell (`/usr/sbin/nologin`) und kein Passwort; normale Benutzer bekommen nur die Gruppen, die sie brauchen. Wichtige Optionen: `-m` legt das Home-Verzeichnis an, `-s` setzt die Shell, `-G` nennt zusätzliche (supplementary) Gruppen, `-r` (`-{}-system`) erzeugt ein Systemkonto. Die berüchtigtste Falle steckt in `usermod`: `usermod -G admins anna` *ersetzt* alle bisherigen Zusatzgruppen, nur `usermod -aG admins anna` (`-a` für append) fügt hinzu. Passwörter setzt man nie über die Befehlszeile (sichtbar in `ps`); `chpasswd` liest `name:passwort` von der Standardeingabe, `passwd -l` sperrt ein Konto, `chage` steuert Ablaufdaten. In produktiven Umgebungen kommen Passwörter ohnehin selten vor; man verteilt SSH-Schlüssel (Modul 7.8).

Ein Skript für die Kontenverwaltung ist ein Paradebeispiel für Idempotenz: Es liest eine *Sollliste* und stellt den Zustand her. Es verifiziert zuerst die Namen (Modul 8.2: Allowlist `\^{}[a-z_][a-z0-9_-]{0,31}$`), prüft, ob Gruppe und Benutzer existieren, legt nur Fehlendes an und ergänzt Gruppen. Standardmäßig läuft es im Trockenlauf und führt Änderungen nur mit `-{}-apply` aus.

```bash
#!/usr/bin/env bash
# benutzer-sync.sh [--apply] SOLLDATEI   (Zeile: name:shell:gruppe1,gruppe2)
set -euo pipefail
apply=0
[[ ${1:-} == --apply ]] && { apply=1; shift; }
soll="${1:?Aufruf: $0 [--apply] SOLLDATEI}"
[[ $EUID -eq 0 || $apply -eq 0 ]] || { echo "--apply braucht root" >&2; exit 1; }

aenderungen=0
tue() {                              # tue BEFEHL...  : im Trockenlauf nur anzeigen
    aenderungen=$((aenderungen + 1))
    if (( apply )); then echo "+ $*"; "$@"; else echo "(Trockenlauf) $*"; fi
}
name_ok() { [[ $1 =~ ^[a-z_][a-z0-9_-]{0,31}$ ]]; }

while IFS=: read -r name shell gruppen; do
    [[ -z $name || $name == \#* ]] && continue
    name_ok "$name" || { echo "ungueltiger Name: $name" >&2; continue; }
    shell="${shell:-/bin/bash}"

    # Gruppen sicherstellen
    IFS=, read -ra gl <<< "${gruppen:-}"
    for g in "${gl[@]}"; do
        getent group "$g" >/dev/null || tue groupadd -- "$g"
    done

    if id -u -- "$name" >/dev/null 2>&1; then
        # existiert: Shell und Gruppen angleichen, ohne andere Gruppen zu entfernen
        ist_shell="$(getent passwd "$name" | cut -d: -f7)"
        [[ $ist_shell == "$shell" ]] || tue usermod -s "$shell" -- "$name"
        ist_gruppen=" $(id -nG -- "$name") "
        for g in "${gl[@]}"; do
            [[ $ist_gruppen == *" $g "* ]] || tue usermod -aG "$g" -- "$name"
        done
    else
        tue useradd -m -s "$shell" ${gruppen:+-G "$gruppen"} -- "$name"
    fi
done < "$soll"
echo "Aenderungen: $aenderungen"
```

Die Hilfsfunktion `tue` kapselt das Trockenlauf-Prinzip: Sie zählt jede geplante Änderung und führt den Befehl nur mit `-{}-apply` aus, sodass derselbe Codepfad für Anzeige und Ausführung gilt. Die Schleife liest die Sollliste mit `IFS=: read -r`, überspringt Kommentare und leere Zeilen und validiert den Namen, bevor er irgendwohin gelangt. Für jede Gruppe aus der Liste prüft `getent group`, ob sie existiert. Existiert der Benutzer, werden Shell und Zusatzgruppen *nur bei Abweichung* angepasst, und zwar mit `usermod -aG`, also ohne bestehende Gruppen zu entfernen. Fehlt der Benutzer, legt `useradd -m` ihn an; die Konstruktion `${gruppen:+-G "$gruppen"}` fügt die Option nur hinzu, wenn Gruppen angegeben sind. Ein zweiter Lauf nach `-{}-apply` meldet 0 Änderungen, ein Hinweis auf Idempotenz. Das Skript legt bewusst weder Passwörter noch Schlüssel an.

Typische Fehler: Erstens `usermod -G` ohne `-a`, wodurch ein Benutzer plötzlich aus allen anderen Gruppen verschwindet (und `sudo` verliert). Zweitens `useradd` ohne `-m`: Das Home-Verzeichnis fehlt, und der erste Login scheitert. Drittens werden Konten mit `userdel` ohne `-r` gelöscht, sodass Home und Mail-Spool zurückbleiben und die UID später einem anderen Benutzer zugeteilt wird, der dann fremde Dateien erbt. Viertens ist `userdel` für Konten gedacht, deren Prozesse nicht mehr laufen; sonst bricht es ab. Fünftens werden Passwörter mit `echo "name:pw" | chpasswd` in Skripten hartcodiert (Modul 8.4). Sechstens prüft man mit `grep anna /etc/passwd` und trifft auch `hanna`; `getent passwd anna` und `id -u anna` sind exakt.

```bash
# falsch: ersetzt alle Zusatzgruppen; Teiltreffer bei der Pruefung
grep -q "$name" /etc/passwd || useradd "$name"
usermod -G wiki "$name"

# richtig: exakte Pruefung, Gruppe hinzufuegen
id -u -- "$name" >/dev/null 2>&1 || useradd -m -- "$name"
usermod -aG wiki -- "$name"
```

> **Merke**
>
> Konten mit `useradd`/`usermod` verwalten, nie die Dateien von Hand. Abfragen mit `id` und `getent`. `usermod -aG` (nicht `-G`). Dienstkonten ohne Shell und Passwort. Sollzustand lesen, Idempotenz, Trockenlauf als Standard.

> **Praxisbeispiel**
>
> Beim Onboarding einer Gruppe von Praktikanten legt ein Skript aus einer CSV-Liste Konten mit den passenden Gruppen an; beim Offboarding sperrt es die Konten (`usermod -L -e 1`), archiviert das Home-Verzeichnis und löscht es nach der Aufbewahrungsfrist. Der Trockenlauf wird vor jedem echten Lauf im Ticket dokumentiert.

## 7.2 Pakete und Repositories

Software kommt unter Linux in *Paketen* aus *Repositories*, die die Distribution oder Dritte betreiben. Der *Paketmanager* löst Abhängigkeiten auf, prüft Signaturen und führt Buch über installierte Dateien. Unter Debian und Ubuntu arbeiten `apt-get` (Skriptschnittstelle) und `dpkg` (Einzelpakete); bei Red Hat, Fedora und Alma `dnf` und `rpm`; bei SUSE `zypper`. Für Skripte ist `apt-get` gedacht, nicht das benutzerfreundliche `apt`, dessen Ausgabeformat sich ändern darf („WARNING: apt does not have a stable CLI interface“).

Eine *nicht-interaktive* Installation darf nie auf Eingaben warten, weder auf „Weiter? \[J/n\]“ noch auf Konfigurationsdialoge. Dafür setzt man `-y` (`dnf -y`, `zypper -{}-non-interactive`) und unter Debian die Umgebungsvariable `DEBIAN_FRONTEND=noninteractive`; Fragen zu geänderten Konfigurationsdateien beantwortet man mit `-o Dpkg::Options::=--force-confold`. Weitere Absicherungen: `--no-install-recommends` hält die Installation klein, `-o DPkg::Lock::Timeout=120` wartet, wenn ein anderer Paketprozess die Sperre hält (zum Beispiel automatische Updates), statt sofort zu scheitern, und `-s` simuliert. Pakete vor der Installation zu prüfen ist ein Muster der Idempotenz: `dpkg -s paket` beziehungsweise `rpm -q paket` liefert Exit-Code 0, wenn es installiert ist.

Repositories sind eine Vertrauensfrage (Modul 8.6). Jedes zusätzliche Repository darf Pakete mit Root-Rechten einspielen. Moderne Debian-Systeme verwalten Schlüssel pro Repository in `/etc/apt/keyrings/` und verknüpfen sie über die Option `signed-by` in der Quellenliste; das alte, globale `apt-key add` ist abgekündigt. Skripte, die Repositories hinzufügen, prüfen den Fingerabdruck des Schlüssels gegen einen bekannten Wert, bevor sie ihm vertrauen. Versionen *pinnt* man, wenn Reproduzierbarkeit zählt (`apt-get install paket=1.2.3-1`, `apt-mark hold paket`).

```bash
#!/usr/bin/env bash
# paket-sicher.sh PAKET...  - installiert fehlende Pakete nicht-interaktiv (Simulation mit SIMULATE=1)
set -euo pipefail

erkenne_pm() {
    if   command -v apt-get >/dev/null; then echo apt
    elif command -v dnf     >/dev/null; then echo dnf
    elif command -v zypper  >/dev/null; then echo zypper
    else echo "kein bekannter Paketmanager" >&2; return 1; fi
}

ist_installiert() {                         # ist_installiert PM PAKET
    case $1 in
        apt)        dpkg -s -- "$2" >/dev/null 2>&1 ;;
        dnf|zypper) rpm -q -- "$2"  >/dev/null 2>&1 ;;
    esac
}

installiere() {                             # installiere PM PAKET...
    local pm="$1"; shift
    local sim=()
    case $pm in
        apt)
            [[ ${SIMULATE:-0} == 1 ]] && sim=(-s)
            DEBIAN_FRONTEND=noninteractive apt-get install -y "${sim[@]}" \
                --no-install-recommends \
                -o DPkg::Lock::Timeout=120 \
                -o Dpkg::Options::=--force-confold -- "$@" ;;
        dnf)    dnf -y install -- "$@" ;;
        zypper) zypper --non-interactive install -- "$@" ;;
    esac
}

pm="$(erkenne_pm)"
fehlend=()
for p in "$@"; do
    [[ $p =~ ^[a-z0-9][a-z0-9+.:_-]*$ ]] || { echo "ungueltiger Paketname: $p" >&2; exit 2; }
    ist_installiert "$pm" "$p" && echo "ok:      $p ist installiert" || fehlend+=("$p")
done

if (( ${#fehlend[@]} )); then
    echo "installiere: ${fehlend[*]}  (Paketmanager: $pm)"
    installiere "$pm" "${fehlend[@]}" | tail -n 3
else
    echo "nichts zu tun"
fi
```

Das Skript trennt Erkennung, Zustandsprüfung und Installation. `erkenne_pm` wählt den Paketmanager; `ist_installiert` fragt je nach System `dpkg` oder `rpm`, ohne Ausgabe, nur über den Exit-Code. Nur fehlende Pakete werden gesammelt (Array `fehlend`) und in *einem* Aufruf installiert, was schneller ist und Abhängigkeiten gemeinsam auflöst. Die Namensprüfung verhindert, dass ein Paketname als Option gelesen wird (`-{}-` und die Allowlist). Der Block mit `sim=()` zeigt, wie man eine Option nur bedingt hinzufügt (leeres Array bleibt unsichtbar; mit `set -u` ist `"${sim[@]}"` in Bash 4.4 und neuer bei leeren Arrays erlaubt, in älteren Versionen ein Fehler, siehe Modul 3.6). Der Aufruf mit `SIMULATE=1` und einem noch nicht installierten Paket zeigt die geplante Installation, ohne etwas zu ändern.

Typische Fehler: Erstens `apt install` statt `apt-get install` in Skripten. Zweitens fehlt `-y` oder `DEBIAN_FRONTEND`, und das Skript hängt in CI und Cron ohne Terminal für immer. Drittens führt man `apt-get update` bei *jedem* Lauf aus (langsam, belastet Spiegel) oder gar nicht (veraltete Paketlisten, 404-Fehler). Viertens wird jedes Paket einzeln installiert. Fünftens laufen zwei Paketprozesse gleichzeitig (etwa `unattended-upgrades`) und das Skript scheitert an der Sperre; `DPkg::Lock::Timeout` hilft. Sechstens installiert man Pakete aus unbekannten Quellen mit `curl ... | bash` oder `dpkg -i` ohne Signaturprüfung. Siebtens setzt man `apt-get remove -y` ohne Nachdenken ein: Autoremove und Abhängigkeiten können mehr entfernen als erwartet; zuerst mit `-s` simulieren.

```bash
# falsch: interaktiv, ohne Pruefung, unsignierte Quelle
apt install nginx
curl https://example.org/repo.sh | sudo bash

# richtig: skriptfaehig, mit Sperr-Timeout, Zustand geprueft
dpkg -s nginx >/dev/null 2>&1 || \
  DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
     -o DPkg::Lock::Timeout=120 nginx
```

> **Merke**
>
> In Skripten `apt-get`, `dnf`, `zypper`; nicht-interaktiv mit `-y` und `DEBIAN_FRONTEND=noninteractive`. Erst prüfen, dann nur Fehlendes in einem Aufruf installieren. Sperren abwarten. Repositories nur mit geprüftem Schlüssel; Versionen bei Bedarf pinnen; vor dem Entfernen simulieren (`-s`).

> **Praxisbeispiel**
>
> Ein Provisionierungsskript richtet frische Server ein: Pakete laufen über genau eine Funktion, die Liste der Pflichtpakete steht in einer Datei im Repository, und im CI läuft das Skript als Simulation gegen ein Basis-Image, sodass fehlende Paketnamen schon im Pull Request auffallen.

## 7.3 Services mit systemd

Auf den meisten Distributionen startet und überwacht *systemd* die Dienste. Ein Dienst ist eine *Unit* (Dateien mit Endung `.service`, `.timer`, `.socket` und anderen). Zentrale Werkzeuge sind `systemctl` zum Steuern und Abfragen und `journalctl` zum Lesen des Journals (des strukturierten Systemlogs). Die wichtigsten Unterbefehle für Skripte: `systemctl is-active dienst` (Exit-Code 0, wenn aktiv; Ausgabe `active`, `inactive`, `failed`), `systemctl is-enabled dienst` (startet beim Boot?), `systemctl is-failed`, `systemctl restart`, `reload`, `enable --now`, `daemon-reload` (nach Änderung an Unit-Dateien) und `systemctl show -p Eigenschaft --value dienst` (maschinenlesbare Werte, etwa `MainPID` oder `ActiveEnterTimestamp`).

Für Skripte gilt: *Neustart ist nicht gleich Erfolg*. `systemctl restart` kehrt oft zurück, sobald der Startversuch eingeleitet wurde; bei `Type=simple`-Diensten sieht man einen Absturz kurz danach nicht. Eine robuste Routine wartet deshalb mit einer Zeitgrenze, bis `is-active` stabil `active` meldet, und zeigt bei Misserfolg die letzten Journal-Zeilen. `journalctl -u dienst --since "10 min ago" -p warning -o cat --no-pager` filtert nach Dienst, Zeit und Schwere und gibt reinen Text ohne Zeitstempel aus. Wichtig ist `--no-pager`, damit in Skripten kein Pager wartet.

Eigene Skripte werden zu Diensten über eine Unit-Datei in `/etc/systemd/system/`. Sie beschreibt `ExecStart`, den Benutzer (`User=`), das Verhalten bei Absturz (`Restart=on-failure`) und Härtungsoptionen wie `NoNewPrivileges=true`, `ProtectSystem=strict` und `PrivateTmp=true`, die Least Privilege (Modul 8.5) praktisch umsetzen. Mit `systemd-analyze verify datei.service` prüft man die Datei vor dem Einspielen. Nach dem Kopieren folgen `systemctl daemon-reload` und `systemctl enable --now`.

```bash
#!/usr/bin/env bash
# dienst-tools.sh - Neustart mit Wartezeit und Fehlerdiagnose
# SYSTEMCTL kann ersetzt werden (Tests); Standard ist das echte systemctl.
set -euo pipefail
SC="${SYSTEMCTL:-systemctl}"

warte_aktiv() {                       # warte_aktiv DIENST SEKUNDEN
    local dienst="$1" max="$2" i
    for (( i = 0; i < max; i++ )); do
        if "$SC" is-active --quiet -- "$dienst"; then return 0; fi
        sleep 1
    done
    return 1
}

stabil_aktiv() {                      # nach Start noch N Sekunden pruefen, ob er aktiv bleibt
    local dienst="$1" stabil="$2" i
    for (( i = 0; i < stabil; i++ )); do
        "$SC" is-active --quiet -- "$dienst" || return 1
        sleep 1
    done
}

neustart_mit_check() {                # neustart_mit_check DIENST
    local d="$1"
    [[ $d =~ ^[A-Za-z0-9@:._-]+$ ]] || { echo "ungueltiger Dienstname" >&2; return 2; }
    echo "Neustart von $d"
    "$SC" restart -- "$d"
    if warte_aktiv "$d" 10 && stabil_aktiv "$d" 3; then
        echo "OK: $d laeuft stabil"
    else
        echo "FEHLER: $d ist nicht stabil aktiv. Letzte Journalzeilen:" >&2
        journalctl -u "$d" -n 5 --no-pager -o cat 2>/dev/null || true
        return 1
    fi
}

zeige_status() {
    local d="$1" aktiv aktiviert
    aktiv="$("$SC" is-active -- "$d" 2>/dev/null || true)"
    aktiviert="$("$SC" is-enabled -- "$d" 2>/dev/null || true)"
    printf '%-12s aktiv=%-9s beim_boot=%s\n' "$d" "${aktiv:-unbekannt}" "${aktiviert:-unbekannt}"
}

# --- Test mit einem Stub, der systemctl nachahmt
if [[ ${1:-} == --selbsttest ]]; then
    stub="$(mktemp)"; trap 'rm -f -- "$stub"' EXIT
    cat > "$stub" <<'ENDE'
#!/usr/bin/env bash
case $1 in
    restart)   exit 0 ;;
    is-active) [[ ${*: -1} == gesund ]] && { [[ $2 == --quiet ]] || echo active; exit 0; }
               [[ $2 == --quiet ]] || echo failed; exit 3 ;;
    is-enabled) echo enabled ;;
esac
ENDE
    chmod +x "$stub"
    SC="$stub"
    zeige_status gesund
    neustart_mit_check gesund
    neustart_mit_check kaputt || echo "(Fehlerfall erkannt, Status $?)"
fi
```

Der Kern ist das Wechselspiel von `warte_aktiv` (Warten bis aktiv, mit Obergrenze, also nie eine Endlosschleife) und `stabil_aktiv` (bleibt der Dienst in den Sekunden danach aktiv, oder ist er nach dem Start direkt wieder abgestürzt?). Fehler führen zu einer Diagnose mit den letzten Journal-Zeilen statt zu einem stummen „fehlgeschlagen“. Der Name wird per Allowlist geprüft, denn er stammt oft aus Eingaben. Die Variable `SC` erlaubt es, das echte `systemctl` im Test zu ersetzen: Der Selbsttest baut einen Stub, der sich wie ein gesunder und ein kaputter Dienst verhält, so lässt sich die Logik ohne laufendes systemd prüfen. Diese Technik („Abhängigkeit austauschbar machen“) ist in der Betriebsautomation allgemein nützlich (Modul 12).

Typische Fehler: Erstens wird nach `restart` nichts geprüft, und das Skript meldet „fertig“, während der Dienst schon wieder abgestürzt ist. Zweitens wertet man die Ausgabe von `systemctl status` aus (für Menschen gedacht, mit Farben und Pager) statt `is-active` und `show -p`. Drittens fehlt `--no-pager`, und das Skript hängt in CI. Viertens `systemctl stop` statt `reload`, wo ein Reload genügt und Verbindungen nicht abbrechen soll. Fünftens vergisst man `daemon-reload` nach einer geänderten Unit-Datei, und systemd verwendet die alte Fassung (es warnt dann). Sechstens steht in Unit-Dateien `ExecStart=` mit Shell-Syntax (Pipes, `$VAR`), die systemd nicht interpretiert; dafür ruft man ausdrücklich `/bin/bash -c '...'` auf oder besser ein Skript. Siebtens gibt es Container und Systeme ohne systemd (Alpine, WSL ohne systemd, Docker); das Skript soll das erkennen (`[[ -d /run/systemd/system ]]`).

```bash
# falsch: kein Warten, Text-Ausgabe fuer Menschen, Pager
systemctl restart app
systemctl status app | grep -q running && echo ok
journalctl -u app

# richtig: Exit-Code nutzen, Zeitgrenze, kein Pager
systemctl restart -- app
systemctl is-active --quiet -- app && echo ok
journalctl -u app --since "10 min ago" -p warning --no-pager -o cat
```

> **Merke**
>
> `is-active`, `is-enabled` und `show -p` statt `status` auswerten. Nach `restart` mit Zeitgrenze auf stabilen Zustand warten und bei Fehlern das Journal zeigen. `--no-pager` in Skripten. Unit-Dateien prüfen (`systemd-analyze verify`), danach `daemon-reload`. Auf Systeme ohne systemd achten.

> **Praxisbeispiel**
>
> Ein Deployment-Skript startet einen Dienst neu und wartet 30 Sekunden auf einen stabilen Zustand, danach folgt ein Smoke Test. Bei Misserfolg gibt es automatisch die letzten Journal-Zeilen in die CI-Ausgabe und einen Exit-Code, der die Pipeline stoppt (Modul 10.5).

## 7.4 Dateisystem-Wartung

Volle Dateisysteme sind eine der häufigsten Ursachen für nächtliche Störungen: Anwendungen können nicht mehr schreiben, Datenbanken brechen ab, Logs gehen verloren. Dateisystem-Wartung heißt deshalb, den Füllstand *zu beobachten*, *Ursachen zu finden* und *Abgelaufenes automatisch aufzuräumen*. Zum Messen dienen `df` (Belegung je Dateisystem) und `du` (Belegung je Verzeichnis). In Skripten verwendet man `df -P` (POSIX-Format mit festen Spalten, unabhängig von Zeilenumbrüchen bei langen Gerätenamen) und `df -h` nur für Menschen; für Zahlen nimmt man `df --output=pcent,avail,target` (GNU) oder `-P` mit `awk`. Auch *Inodes* können ausgehen (`df -i`): Millionen kleiner Dateien füllen die Inode-Tabelle, obwohl noch Platz frei ist.

Um den Verursacher zu finden, hilft `du -xh --max-depth=1 /var | sort -rh | head`: `-x` bleibt auf einem Dateisystem, `sort -h` sortiert menschenlesbare Größen. Ein klassisches Rätsel ist: `df` zeigt 100 Prozent, `du` findet aber nur die Hälfte. Der Grund sind *gelöschte, aber noch geöffnete Dateien* (ein Dienst hält ein gelöschtes Log offen); `lsof +L1` zeigt sie, ein Neustart des Dienstes oder `truncate` über `/proc/<pid>/fd` gibt den Platz frei.

Beim Aufräumen gelten die Sicherheitsregeln aus Modul 8: eng begrenzte Muster, Trockenlauf, keine leeren Variablen. `find /var/log/app -type f -name '*.log' -mtime +30 -delete` entfernt alte Logs; `-xdev` verhindert das Überschreiten von Dateisystemgrenzen; `-type f` lässt Verzeichnisse stehen. Logrotation übernimmt besser `logrotate`, das Komprimieren, Rotieren und Signalisieren erledigt. Das Journal begrenzt man mit `journalctl --vacuum-size=500M` oder `--vacuum-time=30d`. Temporäre Verzeichnisse wie `/tmp` werden von `systemd-tmpfiles` verwaltet; eigene Aufräumjobs beschränken sich auf eigene Verzeichnisse.

```bash
#!/usr/bin/env bash
# speicher-check.sh - Schwellwert pruefen, Verursacher zeigen, alte Dateien bereinigen
set -euo pipefail
SCHWELLE="${SCHWELLE:-80}"                    # Prozent

pruefe_belegung() {                           # pruefe_belegung PFAD -> Exit 1 bei Ueberschreitung
    local pfad="$1" prozent
    prozent="$(df -P -- "$pfad" | awk 'NR==2 {gsub("%","",$5); print $5}')"
    printf 'Belegung %s: %s%% (Schwelle %s%%)\n' "$pfad" "$prozent" "$SCHWELLE"
    (( prozent < SCHWELLE ))
}

groesste_verzeichnisse() {                    # groesste_verzeichnisse PFAD [ANZAHL]
    du -xk --max-depth=1 -- "$1" 2>/dev/null | sort -rn | sed -n "2,$((${2:-5} + 1))p" |
        awk '{ printf "%8.1f MB  %s\n", $1/1024, $2 }'
}

bereinige() {                                 # bereinige VERZ TAGE MUSTER [--force]
    local verz="${1:?}" tage="${2:?}" muster="${3:?}" modus="${4:-trocken}"
    [[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; return 1; }
    [[ $tage =~ ^[0-9]+$ ]] || { echo "TAGE muss Zahl sein" >&2; return 1; }
    local n
    if [[ $modus == --force ]]; then
        n="$(find "$verz" -xdev -type f -name "$muster" -mtime "+$tage" -print -delete | wc -l)"
        echo "geloescht: $n Datei(en) in $verz"
    else
        n="$(find "$verz" -xdev -type f -name "$muster" -mtime "+$tage" -print | wc -l)"
        echo "Trockenlauf: $n Datei(en) wuerden geloescht in $verz"
    fi
}

# --- Vorfuehrung in einem Testverzeichnis
t="$(mktemp -d)"; trap 'rm -rf -- "$t"' EXIT
mkdir -p "$t/logs" "$t/cache"
head -c 3000000 /dev/zero > "$t/logs/gross.log"
head -c  500000 /dev/zero > "$t/cache/klein.dat"
touch -d '45 days ago' "$t/logs/alt1.log" "$t/logs/alt2.log"
touch "$t/logs/neu.log"

pruefe_belegung "$t" && echo "unter der Schwelle" || echo "ACHTUNG: ueber der Schwelle"
echo "--- groesste Verzeichnisse:"; groesste_verzeichnisse "$t" 3
bereinige "$t/logs" 30 '*.log'
bereinige "$t/logs" 30 '*.log' --force
ls "$t/logs"
```

Die Funktion `pruefe_belegung` liest die fünfte Spalte der `df -P`-Ausgabe mit `awk`, entfernt das Prozentzeichen und liefert über die Arithmetikprüfung einen Exit-Code: 0, wenn die Schwelle unterschritten wird. So lässt sie sich direkt in `if` oder Healthchecks (Modul 9.2) verwenden. `groesste_verzeichnisse` sortiert die Ausgabe von `du -xk` numerisch absteigend, überspringt die erste Zeile (die Summe) und formatiert die Größen in Megabyte. `bereinige` hat die bekannte Struktur aus Modul 8.1: Eingaben prüfen, standardmäßig Trockenlauf, echtes Löschen nur mit `-{}-force`; die Muster werden in Anführungszeichen übergeben, damit `find` und nicht die Shell sie auswertet. Im Test sind zwei Dateien 45 Tage alt: Der Trockenlauf meldet sie nur, erst `-{}-force` löscht sie, und die große sowie die neue Logdatei bleiben unberührt.

Typische Fehler: Erstens wertet man `df -h` mit `awk` aus, wobei lange Gerätenamen die Spalten verschieben; `-P` verwenden. Zweitens löscht man Logdateien, die ein Dienst noch geöffnet hält: Der Platz bleibt belegt, und der Dienst schreibt ins Leere. Für aktive Logs ist `logrotate` mit `copytruncate` oder ein Reload-Signal der richtige Weg. Drittens sind Muster nicht eng genug (`find / -name '*.log' -delete`). Viertens überschreitet `find` Dateisystemgrenzen (NFS, andere Mounts); `-xdev` verhindert das. Fünftens misst man nur Platz und nicht Inodes. Sechstens wird `-mtime` falsch verstanden: `+30` bedeutet „älter als 30 volle Tage“, gerundet nach unten. Siebtens löscht man, ohne zu protokollieren, was und wie viel entfernt wurde (Modul 8.8).

```bash
# falsch: breites Muster, kein Trockenlauf, Mount-Grenzen ignoriert
find / -name "*.log" -mtime +7 -delete
df -h | awk '{print $5}'

# richtig: enger Pfad, -xdev, -type f, vorher anzeigen
find /var/log/app -xdev -type f -name '*.log' -mtime +30 -print
df -P /var | awk 'NR==2 {print $5}'
```

> **Merke**
>
> `df -P` für Zahlen, `du -x | sort -h` für Verursacher, auch Inodes beachten. Aufräumen mit engem Pfad, `-xdev`, `-type f`, Trockenlauf und Protokoll. Aktive Logs nicht einfach löschen: `logrotate`. Differenz zwischen `df` und `du` = gelöscht, aber geöffnet.

> **Praxisbeispiel**
>
> Ein Cron-Job prüft jede Stunde die Belegung von `/var` und `/srv`. Über 80 Prozent gibt es eine Warnung mit den fünf größten Verzeichnissen, über 90 Prozent ein Alarm, und nachts löscht ein Bereinigungsskript Zwischendateien, die älter als eine Woche sind.

## 7.5 Netzwerkdiagnose

Wenn „das Netz kaputt ist“, hilft kein Raten, sondern ein Vorgehen entlang der Schichten: Gibt es eine *Adresse und eine Route*? Funktioniert die *Namensauflösung* (DNS)? Ist der *Port erreichbar* (TCP-Verbindung)? Antwortet die *Anwendung* korrekt (HTTP-Status, Antwortzeit)? Für jede Schicht gibt es ein Werkzeug. `ip addr` und `ip route` zeigen Adressen und Routen (`ip route get 8.8.8.8` nennt den gewählten Weg), `ss -tlnp` listet lauschende TCP-Sockets samt Prozessen und löst `netstat` ab, `dig +short name` und `getent hosts name` lösen Namen auf (`getent` nutzt die Systemkonfiguration wie Anwendungen, `dig` fragt direkt einen DNS-Server und kann mit `@server` gezielt abgefragt werden), `curl` prüft HTTP.

Für Skripte sind die Exit-Codes und maschinenlesbaren Ausgaben entscheidend. `curl -sS -o /dev/null -w '%{http_code} %{time_total}' --max-time 5 URL` liefert Statuscode und Dauer; `--fail` macht aus HTTP-Fehlern einen Exit-Code. Ein TCP-Test ohne zusätzliche Werkzeuge gelingt mit Bash selbst: `timeout 3 bash -c ': < /dev/tcp/host/port'` öffnet eine Verbindung und scheitert, wenn niemand lauscht (die Pseudodatei `/dev/tcp` ist eine Bash-Funktion, kein echtes Gerät). Typische Befunde lassen sich so unterscheiden: „Name nicht auflösbar“ (DNS, Resolver, `/etc/resolv.conf`), „Connection refused“ (Host erreichbar, aber niemand lauscht), „Timeout“ (Paket kommt nirgends an: Firewall, Routing, Host aus), „Verbindung ok, Fehler 502/503“ (Anwendungsproblem).

```bash
#!/usr/bin/env bash
# netzdiag.sh HOST PORT [URLPFAD]  - Diagnose von DNS ueber TCP bis HTTP
set -uo pipefail
host="${1:?Aufruf: $0 HOST PORT [PFAD]}"; port="${2:?PORT fehlt}"; pfad="${3:-/}"
[[ $host =~ ^[A-Za-z0-9._-]+$ ]]  || { echo "ungueltiger Host" >&2; exit 2; }
[[ $port =~ ^[0-9]{1,5}$ ]] && (( port >= 1 && port <= 65535 )) || { echo "ungueltiger Port" >&2; exit 2; }

ok()   { printf '[ OK ] %s\n' "$*"; }
fail() { printf '[FAIL] %s\n' "$*"; }
status=0

# 1. Namensaufloesung wie Anwendungen sie sehen
adresse="$(getent hosts -- "$host" | awk 'NR==1 {print $1}')"
if [[ -n $adresse ]]; then ok "DNS: $host -> $adresse"
else fail "DNS: $host nicht aufloesbar (Resolver pruefen: /etc/resolv.conf, dig @server)"; exit 10; fi

# 2. Route zum Ziel (nur wenn das Werkzeug da ist)
if command -v ip >/dev/null; then
    route="$(ip route get "$adresse" 2>/dev/null | head -n 1 | cut -c1-70)"
    [[ -n $route ]] && ok "Route: $route" || { fail "keine Route zu $adresse"; status=11; }
fi

# 3. TCP-Verbindung mit Zeitgrenze; Refused und Timeout unterscheiden
start=$SECONDS
if timeout 3 bash -c ": < /dev/tcp/$adresse/$port" 2>/dev/null; then
    ok "TCP: $adresse:$port erreichbar ($((SECONDS - start)) s)"
else
    rc=$?
    if (( rc == 124 )); then fail "TCP: Timeout (Firewall, Routing oder Host aus)"; status=12
    else fail "TCP: Verbindung abgelehnt (Host erreichbar, aber kein Dienst auf $port)"; status=13; fi
    exit "$status"
fi

# 4. HTTP-Antwort und Dauer
if command -v curl >/dev/null; then
    out="$(curl -sS -o /dev/null --max-time 5 -w '%{http_code} %{time_total}' "http://$host:$port$pfad" 2>&1)" || true
    code="${out%% *}"
    case $code in
        2??|3??) ok "HTTP: Status $code, Dauer ${out#* } s" ;;
        000)     fail "HTTP: keine Antwort ($out)"; status=14 ;;
        *)       fail "HTTP: Status $code"; status=15 ;;
    esac
fi
exit "$status"
```

Das Skript prüft von unten nach oben und bricht ab, sobald eine Schicht ausfällt: ohne Namensauflösung ist alles Weitere sinnlos. Die Exit-Codes (10 bis 15) unterscheiden die Fehlerarten, sodass ein Scheduler oder ein Alarmsystem sie auswerten kann (Modul 9.2). Die TCP-Prüfung nutzt `timeout` und unterscheidet über den Exit-Code 124 den Timeout von einer abgelehnten Verbindung, eine Unterscheidung, die in der Praxis oft die halbe Diagnose ist: Ein Timeout deutet auf Firewall oder Routing, ein „Refused“ darauf, dass der Rechner antwortet, aber kein Dienst lauscht. Host und Port werden validiert, bevor sie in `bash -c` oder in eine URL eingebaut werden (Modul 8.3). Zum Test startet man lokal einen Webserver (`python3 -m http.server 8099 --bind 127.0.0.1 &`) und ruft das Skript mit `localhost 8099` auf; mit einem ungenutzten Port zeigt es die „Refused“-Variante.

Typische Fehler: Erstens wird `ping` als Beweis für „Netz ok“ gewertet; ICMP wird oft gefiltert, und ein Dienst kann trotz Ping unerreichbar sein. Zweitens wird nur mit `dig` geprüft, obwohl Anwendungen über `/etc/nsswitch.conf`, `/etc/hosts` und lokale Resolver auflösen; `getent hosts` zeigt, was Programme sehen. Drittens fehlen Zeitgrenzen (`-{}-max-time`, `timeout`), und Diagnoseskripte hängen minutenlang. Viertens verwechselt man IPv4 und IPv6 (ein `AAAA`-Eintrag führt zu einer Adresse, die nicht erreichbar ist; `curl -4` und `-6` trennen das). Fünftens werden `curl`-Fehler verschluckt, weil `-s` ohne `-S` auch die Fehlermeldung unterdrückt. Sechstens wird der Proxy vergessen (`http_proxy`, `no_proxy`) und gewundert, warum Skript und Browser Unterschiedliches sehen. Siebtens verlässt man sich auf `netstat`, das nicht mehr installiert ist; `ss` ist der Nachfolger.

```bash
# falsch: Ping als Beweis, kein Zeitlimit, Fehlermeldung unterdrueckt
ping -c1 dienst.intern && echo "Dienst ok"
curl -s http://dienst.intern/health

# richtig: Schicht fuer Schicht, mit Zeitgrenzen und Fehlertext
getent hosts dienst.intern
timeout 3 bash -c ': < /dev/tcp/dienst.intern/8080' && echo "Port offen"
curl -sS --fail --max-time 5 -o /dev/null -w '%{http_code}\n' http://dienst.intern:8080/health
```

> **Merke**
>
> Schichtweise vorgehen: Adresse/Route (`ip`), DNS (`getent`, `dig`), Port (`ss`, `/dev/tcp`), Anwendung (`curl`). Immer Zeitgrenzen setzen. Timeout, Refused und Namensfehler sind verschiedene Befunde. Exit-Codes je Fehlerart vergeben.

> **Praxisbeispiel**
>
> Ein Healthcheck-Skript prüft für jede Abhängigkeit eines Dienstes DNS, Port und Health-Endpunkt und liefert getrennte Exit-Codes. Der Bereitschaftsdienst sieht bei einem Alarm sofort, ob es ein Namens-, Netz- oder Anwendungsproblem ist, und spart die erste halbe Stunde Fehlersuche.

## 7.6 Backup-Skripte

Ein Backup ist erst dann eines, wenn man daraus *wiederherstellen* kann; der häufigste Fehler ist ein Backup, das nie getestet wurde. Ein gutes Backup-Skript beantwortet vier Fragen: Was wird gesichert (Umfang), wohin (Ziel, getrennt vom Original), wie oft und wie lange aufbewahrt (Rotation) und wie wird die Wiederherstellung geprüft (Restore-Test). Für Dateien eignen sich zwei Werkzeuge. `tar` packt Verzeichnisse in ein Archiv (`tar -czf ziel.tar.gz -C quelle .`), das sich leicht verschieben und prüfen lässt (`tar -tzf` listet den Inhalt, `gzip -t` testet die Kompression). `rsync` gleicht Verzeichnisse *inkrementell* ab: Es überträgt nur Geändertes; mit `--link-dest=VORHERIGES` entstehen platzsparende Snapshots, bei denen unveränderte Dateien per *Hardlink* auf den vorigen Stand zeigen und doch jeder Snapshot wie ein vollständiges Backup aussieht. Wichtige `rsync`-Optionen: `-a` (Archivmodus: Rechte, Zeiten, Symlinks, Rekursion), `-{}-delete` (Gelöschtes auch im Ziel löschen; *gefährlich* bei falschem Ziel), `-n` (Trockenlauf), `-{}-exclude`, `-{}-numeric-ids` und `-H -A -X` für Hardlinks, ACLs und erweiterte Attribute. Auch das Quell-Argument hat eine Eigenheit: `quelle/` (mit Schrägstrich) synchronisiert den *Inhalt*, `quelle` (ohne) das Verzeichnis selbst.

Die *Rotation* begrenzt den Platzbedarf: Man behält die letzten N Sicherungen (oder gestaffelt: täglich 7, wöchentlich 4, monatlich 6) und löscht ältere. Am sichersten ist es, Backup-Namen mit Zeitstempel zu versehen (`20261030-231500`), weil sie sich lexikografisch sortieren lassen: Das Löschen der ältesten wird zu `ls | sort | head`. Ein Backup-Lauf sollte *atomar* sein: Er schreibt in ein Verzeichnis mit Namen `.unvollstaendig` und benennt es erst nach Erfolg um, damit nie ein halbes Backup wie ein gutes aussieht. Zum Nachweis gehören eine *Prüfsummendatei* und ein *Restore-Test*: regelmäßig in ein Testverzeichnis zurückspielen und vergleichen. Schließlich gelten alle Regeln aus Modul 8: Backups enthalten Geheimnisse und personenbezogene Daten und gehören deshalb mit restriktiven Rechten abgelegt, gegebenenfalls verschlüsselt, und mit Aufbewahrungsfristen versehen (Modul 8.9).

```bash
#!/usr/bin/env bash
# backup.sh QUELLE ZIELBASIS [BEHALTEN]  - rsync-Snapshots mit Hardlinks, Rotation, Restore-Test
set -euo pipefail
umask 077
quelle="${1:?Aufruf: $0 QUELLE ZIELBASIS [BEHALTEN]}"; basis="${2:?}"; behalten="${3:-3}"
[[ -d $quelle ]] || { echo "Quelle fehlt: $quelle" >&2; exit 2; }
[[ $behalten =~ ^[1-9][0-9]*$ ]] || { echo "BEHALTEN muss >= 1 sein" >&2; exit 2; }
mkdir -p -- "$basis"
basis="$(cd -- "$basis" && pwd)"                  # absoluter Pfad (wichtig nach cd in Subshells)

jetzt="${BACKUP_ZEIT:-$(date +%Y%m%d-%H%M%S)}"        # BACKUP_ZEIT nur fuer Tests
ziel="$basis/$jetzt"; roh="$basis/.unvollstaendig-$jetzt"
letztes="$(find "$basis" -mindepth 1 -maxdepth 1 -type d -name '2*' | sort | tail -n 1)"

# 1. Snapshot nach .unvollstaendig schreiben, unveraenderte Dateien per Hardlink uebernehmen
opts=(-a --numeric-ids)
[[ -n $letztes ]] && opts+=(--link-dest="$letztes")
rsync "${opts[@]}" -- "$quelle"/ "$roh"/

# 2. Pruefsummen erzeugen, dann atomar freigeben
( cd "$roh" && find . -type f -print0 | sort -z | xargs -0 sha256sum > "$basis/.sums-$jetzt" )
mv -- "$basis/.sums-$jetzt" "$roh/.SHA256SUMS"
mv -- "$roh" "$ziel"
echo "Backup angelegt: $ziel"

# 3. Rotation: nur die letzten N behalten
mapfile -t alle < <(find "$basis" -mindepth 1 -maxdepth 1 -type d -name '2*' | sort)
anzahl=${#alle[@]}
if (( anzahl > behalten )); then
    for alt in "${alle[@]:0:anzahl-behalten}"; do
        echo "Rotation: loesche $alt"; rm -rf -- "${alt:?}"
    done
fi

# 4. Restore-Test: in Testverzeichnis zurueckspielen und Pruefsummen vergleichen
test_ziel="$(mktemp -d)"; trap 'rm -rf -- "$test_ziel"' EXIT
rsync -a -- "$ziel"/ "$test_ziel"/
( cd "$test_ziel" && grep -v '\.SHA256SUMS' .SHA256SUMS | sha256sum -c --quiet - ) \
    && echo "Restore-Test OK ($(find "$test_ziel" -type f ! -name .SHA256SUMS | wc -l) Dateien)" \
    || { echo "Restore-Test FEHLGESCHLAGEN" >&2; exit 5; }
```

Das Skript sichert in Snapshot-Verzeichnisse mit Zeitstempel. Der Hardlink-Trick: `-{}-link-dest` verweist auf den vorigen Snapshot; `rsync` legt für unveränderte Dateien nur einen Hardlink an (kein zusätzlicher Platz), kopiert aber neue und geänderte. Alles landet zunächst in einem Verzeichnis mit Präfix `.unvollstaendig-`, das erst nach den Prüfsummen mit `mv` umbenannt wird; bricht der Lauf ab, bleibt ein erkennbar unfertiger Rest statt eines scheinbar fertigen Backups (er kann mit Rotation oder `trap` entfernt werden). Die Rotation sortiert die Namen, die mit `2` beginnen (Jahreszahl), und löscht alle bis auf die letzten N über eine Array-Teilausschnitt-Syntax. Der Restore-Test spielt den Snapshot in ein Testverzeichnis zurück und verifiziert die Prüfsummen; schlägt er fehl, endet das Skript mit Exit-Code 5. Das Skript sichert auf demselben Rechner; in der Praxis liegt das Ziel auf einem anderen Datenträger oder Host (Modul 7.8).

Typische Fehler: Erstens liegt das Backup auf derselben Platte wie das Original und verschwindet bei einem Defekt mit. Zweitens wird nie ein Restore getestet. Drittens `rsync --delete` mit vertauschten Pfaden oder leerer Variable: Das Ziel wird zum Spiegel eines leeren Verzeichnisses; deshalb Variablen mit `${var:?}` absichern und neue Aufrufe erst mit `-n` testen. Viertens wird der Schrägstrich am Quellpfad vergessen. Fünftens sichert man laufende Datenbanken per Dateikopie, was inkonsistente Zustände ergibt; dafür gibt es `pg_dump`, `mysqldump` oder Snapshots. Sechstens ist die Rotation fehlerhaft, und alle Backups werden gelöscht (Mindestanzahl erzwingen). Siebtens fehlen Rechte: Backups mit `644` verraten Geheimnisse.

```bash
# falsch: Ziel auf gleicher Platte, --delete ohne Schutz, kein Test
rsync -a --delete /srv/daten/ $ZIEL/

# richtig: Variablen abgesichert, erst Trockenlauf, dann echt, danach Restore-Test
rsync -a --delete -n -- "${QUELLE:?}"/ "${ZIEL:?}"/
rsync -a --delete    -- "${QUELLE:?}"/ "${ZIEL:?}"/
```

> **Merke**
>
> 3-2-1: drei Kopien, zwei Medien, eine außer Haus. Inkrementell mit `rsync -a -{}-link-dest`, Rotation mit Mindestanzahl, atomar freigeben, Prüfsummen anlegen, *Restore regelmäßig testen*. `-{}-delete` nur mit gesicherten Variablen und nach Trockenlauf. Backups schützen wie Originale.

> **Praxisbeispiel**
>
> Ein nächtliches Backup-Skript sichert Konfiguration und Anwendungsdaten per Snapshot, behält 7 tägliche und 4 wöchentliche Stände und führt jeden Sonntag einen Restore-Test auf einem Testserver aus. Bei einem Defekt des Hauptsystems war die Wiederherstellung binnen einer Stunde möglich, weil der Ablauf erprobt war.

## 7.7 Konfigurationsmanagement light

Große Werkzeuge wie Ansible, Puppet oder Chef verwalten Konfigurationen deklarativ. Für kleine Umgebungen genügt oft ein leichtgewichtiger Ansatz mit Bash: *Vorlagen* (Templates) mit Platzhaltern, *Variablen* je Umgebung, ein Skript, das die fertigen Dateien *erzeugt und nur bei Änderung ersetzt*, und *Git* als Gedächtnis, das jede Änderung nachvollziehbar macht. Das Grundprinzip lautet: Die Wahrheit liegt im Repository (Vorlagen und Variablen), nicht auf dem Server; ein Server wird aus dem Repository *erzeugt*, nicht von Hand editiert.

Zum Ersetzen von Platzhaltern eignet sich `envsubst` (Paket `gettext-base`). Es ersetzt `$VAR` und `${VAR}` durch Umgebungswerte. Wichtig ist die Variante `envsubst '$PORT $HOST' < vorlage`: Die Argumentliste *beschränkt* die Ersetzung auf genannte Variablen, alles andere (zum Beispiel nginx-Variablen wie `$host`) bleibt unberührt. Wer ganz ohne Zusatzprogramm auskommen will, nutzt Bash selbst, etwa mit `${zeile//@@PORT@@/$PORT}`. Die Variablen je Umgebung liegen in einer Datei, die man mit `source` einliest; da sie Code ausführt, muss sie aus einer vertrauenswürdigen Quelle stammen und darf keine Geheimnisse im Repository enthalten (Modul 8.4).

Der Ablauf eines sauberen Konfigurationslaufs: Vorlage rendern in eine temporäre Datei, mit der installierten Fassung *vergleichen* (`cmp -s`, `diff -u`), nur bei Unterschied *atomar* ersetzen (Modul 6.6), optional vorher die Konfiguration validieren (`nginx -t`, `sshd -t`) und erst danach den Dienst neu laden. Das spart unnötige Neustarts und macht das Skript idempotent: Läuft es zweimal, passiert beim zweiten Mal nichts. Für die Nachvollziehbarkeit hält man das Zielverzeichnis oder die Vorlagen in Git: `git add` und `git commit` mit sprechender Meldung nach jeder Änderung, `git log -p datei` zeigt, wer wann was geändert hat, `git diff` und `git revert` ermöglichen den Rollback.

```bash
#!/usr/bin/env bash
# konfig-render.sh UMGEBUNG  - Vorlage + Variablen -> Konfig, nur bei Aenderung ersetzen, in Git festhalten
set -euo pipefail
umgebung="${1:?Aufruf: $0 UMGEBUNG (dev|prod)}"
[[ $umgebung =~ ^[a-z]{2,10}$ ]] || { echo "ungueltige Umgebung" >&2; exit 2; }

repo="$(mktemp -d)"; trap 'rm -rf -- "$repo"' EXIT
mkdir -p "$repo"/{vorlagen,variablen,ziel}
cd "$repo"
git init -q; git config user.name "Admin"; git config user.email "admin@example.org"

cat > vorlagen/app.conf.tpl <<'ENDE'
# Verwaltet per Skript - nicht von Hand aendern
listen   = 0.0.0.0:${PORT}
umgebung = ${UMGEBUNG}
loglevel = ${LOGLEVEL}
pfad     = $request_uri
ENDE
printf 'PORT=8080\nLOGLEVEL=debug\n'  > variablen/dev.env
printf 'PORT=80\nLOGLEVEL=warn\n'     > variablen/prod.env

rendere() {                        # rendert Vorlage; gibt Ergebnis auf stdout aus
    ( set -a; source "variablen/$umgebung.env"; UMGEBUNG="$umgebung"; set +a
      envsubst '${PORT} ${UMGEBUNG} ${LOGLEVEL}' < vorlagen/app.conf.tpl )
}

installiere_wenn_geaendert() {     # installiere_wenn_geaendert ZIELDATEI
    local ziel="$1" neu
    neu="$(mktemp "$(dirname "$ziel")/.neu.XXXXXX")"
    rendere > "$neu"
    if [[ -f $ziel ]] && cmp -s -- "$neu" "$ziel"; then
        rm -f -- "$neu"; echo "unveraendert: $ziel"; return 0
    fi
    [[ -f $ziel ]] && diff -u -- "$ziel" "$neu" | tail -n +3 || true
    chmod 644 "$neu"; mv -f -- "$neu" "$ziel"; echo "aktualisiert: $ziel"
    git add -- "$ziel"
    git commit -q -m "Konfig fuer $umgebung neu gerendert: $(basename "$ziel")"
}

installiere_wenn_geaendert "ziel/app.conf"          # erster Lauf: neu
installiere_wenn_geaendert "ziel/app.conf"          # zweiter Lauf: nichts zu tun
echo 'timeout  = 30' >> vorlagen/app.conf.tpl                  # Vorlage wird erweitert
installiere_wenn_geaendert "ziel/app.conf"
echo "--- Git-Verlauf:"; git log --oneline
echo "--- Ergebnis:"; cat ziel/app.conf
```

Das Skript baut sich ein kleines Repository mit Vorlage und Variablendateien. `rendere` lädt die Variablen in einer Subshell (damit nichts in die Hauptshell leckt), exportiert sie mit `set -a` und lässt `envsubst` *nur* die drei genannten Variablen ersetzen: Die Zeile mit `$request_uri` bleibt wörtlich erhalten, wie es bei nginx-Konfigurationen nötig ist. `installiere_wenn_geaendert` schreibt das Ergebnis in eine temporäre Datei im Zielverzeichnis, vergleicht mit `cmp -s` und tauscht nur bei Unterschied per `mv` aus; beim zweiten Aufruf meldet es `unveraendert`. Jede echte Änderung wird in Git committet; `git log` zeigt anschließend den Verlauf. Die Rückgabewerte sind bewusst einfach, denn der Dienst-Reload (hier nicht gezeigt) folgt nur, wenn das Skript „aktualisiert“ gemeldet hat; ein sauberer Weg ist ein Exit-Code oder eine Marker-Datei für „geändert“.

Typische Fehler: Erstens wird `envsubst` ohne Variablenliste benutzt und ersetzt dadurch Teile, die kein Platzhalter sein sollten (Shell- oder nginx-Variablen werden leer). Zweitens fehlt eine Variable und das Ergebnis enthält stillschweigend leere Werte; `set -u` im Render-Schritt oder eine Prüfung auf Pflichtvariablen beugt vor. Drittens werden Dateien direkt am Ziel überschrieben statt atomar ersetzt; ein halb geschriebenes Konfigfile legt Dienste lahm. Viertens editiert jemand die erzeugte Datei auf dem Server von Hand, und beim nächsten Lauf geht die Änderung verloren; der Kopfkommentar „nicht von Hand ändern“ und ein Vergleich beim Lauf (*Drift*) machen das sichtbar. Fünftens landen Passwörter in Variablendateien im Git. Sechstens wird ohne Validierung neu geladen. Siebtens werden Commits ohne Aussagekraft gemacht („update“).

```bash
# falsch: ersetzt ALLES, schreibt direkt ins Ziel, kein Vergleich
envsubst < app.conf.tpl > /etc/app/app.conf
systemctl restart app

# richtig: beschraenkte Variablen, Vergleich, atomar, validieren, dann reload
envsubst '${PORT} ${LOGLEVEL}' < app.conf.tpl > "$neu"
cmp -s "$neu" /etc/app/app.conf || { app --check-config "$neu" && mv "$neu" /etc/app/app.conf && systemctl reload app; }
```

> **Merke**
>
> Vorlagen und Variablen im Git-Repository, Server werden daraus erzeugt. `envsubst` immer mit Variablenliste. Rendern, vergleichen, nur bei Änderung atomar ersetzen, validieren, dann reload. Pflichtvariablen prüfen; keine Geheimnisse im Repository; Änderungen mit aussagekräftigen Commits.

> **Praxisbeispiel**
>
> Ein Team betreibt zehn Webserver mit leicht unterschiedlichen Konfigurationen. Eine Vorlage plus eine Variablendatei je Server ersetzt zehn handgepflegte Dateien; ein Pull Request mit Diff und Review ist der einzige Weg zur Änderung, und das Skript lädt Dienste nur bei tatsächlich geänderten Dateien neu.

## 7.8 Remote Administration

Die Fernadministration läuft über *SSH* (Secure Shell). `ssh host befehl` führt einen Befehl auf einem anderen Rechner aus, `scp` und `rsync -e ssh` kopieren Dateien, alles über eine verschlüsselte, authentifizierte Verbindung. Für die Automatisierung kommt es auf drei Dinge an: Die Anmeldung muss *ohne Rückfragen* funktionieren, die Gegenstelle muss *sicher identifiziert* werden, und der Befehl muss *korrekt übergeben* werden.

Die Anmeldung erfolgt mit *Schlüsselpaaren*: `ssh-keygen -t ed25519` erzeugt einen privaten und einen öffentlichen Schlüssel. Der öffentliche steht in `\~{}/.ssh/authorized_keys` auf dem Zielsystem, der private bleibt geheim (Rechte `600`, ideal mit Passphrase und `ssh-agent`; für Automatisierung ein eigener, eingeschränkter Schlüssel pro Zweck). In `authorized_keys` kann man einen Schlüssel *einschränken*: `restrict,command="/usr/local/bin/backup-empfang"` erlaubt nur genau diesen Befehl, ohne Shell, Forwarding oder TTY. Die Identität der Gegenstelle klärt die Datei `known_hosts`: Beim ersten Kontakt wird der Host-Schlüssel gespeichert, danach warnt SSH bei jeder Abweichung (Schutz vor Man-in-the-Middle). Skripte schalten diese Prüfung *nie* mit `StrictHostKeyChecking=no` ab; stattdessen trägt man Host-Schlüssel vorab mit `ssh-keyscan` ein *und prüft den Fingerabdruck* (`ssh-keygen -lf`) über einen unabhängigen Weg, oder verteilt sie zentral.

Für unbeaufsichtigte Läufe setzt man Optionen: `BatchMode=yes` (nie nach Passwort fragen, sondern scheitern), `ConnectTimeout=10`, `ServerAliveInterval=15` und `ServerAliveCountMax=3` (tote Verbindungen erkennen), `StrictHostKeyChecking=yes`. Beim Befehl selbst gilt: Die Gegenstelle *wertet den übergebenen Text mit ihrer Shell erneut aus*. Variablen und Sonderzeichen werden daher zweimal interpretiert (lokal und entfernt). Sicher übergibt man Werte mit `printf %q` oder, besser, als Argumente eines festen Skripts auf der Gegenseite. Beim Durchlaufen einer Hostliste mit `while read` *verschluckt* `ssh` die Standardeingabe der Schleife; Abhilfe ist `ssh -n` oder `< /dev/null`.

```bash
#!/usr/bin/env bash
# fernwartung.sh - sichere SSH-Optionen, Hostschluessel pruefen, Befehle korrekt uebergeben
set -euo pipefail
arbeit="$(mktemp -d)"; trap 'rm -rf -- "$arbeit"' EXIT
export HOME="$arbeit"                         # Test: eigene .ssh-Umgebung, nichts beruehren
mkdir -m 700 "$HOME/.ssh"

# --- Optionen fuer unbeaufsichtigte Laeufe, als Array (Modul 3.6)
ssh_opts=(
    -o BatchMode=yes
    -o ConnectTimeout=10
    -o ServerAliveInterval=15 -o ServerAliveCountMax=3
    -o StrictHostKeyChecking=yes
    -o UserKnownHostsFile="$HOME/.ssh/known_hosts"
)

# --- 1. Eigenen, eingeschraenkten Schluessel erzeugen (ohne Passphrase nur fuer Automatisierung!)
ssh-keygen -q -t ed25519 -N '' -C 'backup-automation' -f "$HOME/.ssh/id_backup"
ssh-keygen -lf "$HOME/.ssh/id_backup.pub" | awk '{print "Schluessel-Fingerabdruck:", $2}'
stat -c 'Rechte privater Schluessel: %a' "$HOME/.ssh/id_backup"

# --- 2. Beispielzeile fuer authorized_keys auf dem Zielsystem (nur ausgeben)
echo "restrict,command=\"/usr/local/bin/backup-empfang\" $(cut -d' ' -f1,2 "$HOME/.ssh/id_backup.pub") backup-automation"

# --- 3. Wirksame Konfiguration anzeigen, ohne zu verbinden: ssh -G
echo "--- Effektive Optionen fuer backup1:"
ssh -G -T "${ssh_opts[@]}" -i "$HOME/.ssh/id_backup" backup1 |
    grep -E '^(batchmode|connecttimeout|stricthostkeychecking|serveraliveinterval) '

# --- 4. Befehl sicher uebergeben: Werte quoten, damit die ENTFERNTE Shell sie nicht zerlegt
datei="mein bericht; echo PWNED.txt"           # boesartiger Name
lokal_zeile="ls -l -- $(printf '%q' "$datei")" # %q maskiert fuer die Gegenseite
echo "Gesendeter Befehlstext: $lokal_zeile"

# --- 5. Schleife ueber Hosts: ssh darf stdin der Schleife nicht verbrauchen (-n)
printf 'web1\nweb2\n' > "$arbeit/hosts.txt"
while IFS= read -r h; do
    echo "wuerde ausfuehren: ssh -n ${ssh_opts[*]} $h uptime"
done < "$arbeit/hosts.txt"
```

Das Skript arbeitet in einem eigenen Home-Verzeichnis, sodass es echte Schlüssel und `known_hosts` nicht berührt. Die Optionen stehen in einem Array, das man bei allen `ssh`-, `scp`- und `rsync -e`-Aufrufen wiederverwendet. Schritt 1 erzeugt ein Schlüsselpaar mit Typ `ed25519` und prüft die Rechte; Schritt 2 gibt aus, wie die *eingeschränkte* `authorized_keys`-Zeile aussieht (nur ein erlaubter Befehl, kein weiteres Forwarding). Mit `ssh -G` (hier mit `-T`, damit keine Terminal-Warnung erscheint) zeigt Schritt 3 die *wirksame Konfiguration* an, ohne eine Verbindung aufzubauen, ein nützliches Mittel, um Optionen und Config-Dateien zu prüfen. Schritt 4 zeigt das Quoting-Problem: Der Befehl wird als Text an die Gegenseite gesendet und dort erneut von der Shell zerlegt; `printf '%q'` maskiert den bösartigen Dateinamen so, dass er dort ein einziges Argument bleibt. Schritt 5 erinnert an `ssh -n`. Eine echte Verbindung findet nicht statt, damit das Beispiel überall lauffähig bleibt.

Typische Fehler: Erstens `StrictHostKeyChecking=no` oder `UserKnownHostsFile=/dev/null`, womit jede Man-in-the-Middle-Attacke unbemerkt durchgeht. Zweitens ein einziger Schlüssel ohne Einschränkung für alles, womöglich ohne Passphrase und mit `644`-Rechten (SSH weigert sich dann). Drittens: `ssh` in einer `while read`-Schleife ohne `-n`; nach dem ersten Host ist die Eingabeliste verbraucht. Viertens ungeprüftes Einsetzen von Variablen in Remote-Befehle (Command Injection auf der Gegenseite, Modul 8.3). Fünftens fehlende Zeitgrenzen, wodurch Skripte bei einem toten Host minutenlang hängen. Sechstens liegen private Schlüssel in Repositories oder CI-Logs (Modul 8.4). Siebtens wird Agent-Forwarding (`-A`) auf nicht vertrauenswürdige Hosts aktiviert, die dann den Agenten missbrauchen können. Achtens: `scp` ist veraltet und verhält sich je nach Version unterschiedlich (neuere Versionen nutzen das SFTP-Protokoll); `rsync -e ssh` oder `sftp` sind die robusteren Alternativen.

```bash
# falsch: Hostpruefung aus, Variable ungequotet in Remote-Befehl, stdin verbraucht
while read h; do
    ssh -o StrictHostKeyChecking=no "$h" rm -rf /srv/cache/$name
done < hosts.txt

# richtig: Pruefung an, Zeitgrenzen, Wert maskiert, -n gegen stdin-Verbrauch
while IFS= read -r h; do
    ssh -n -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=yes -- "$h" \
        "rm -rf -- /srv/cache/$(printf '%q' "$name")"
done < hosts.txt
```

> **Merke**
>
> Schlüssel statt Passwörter, pro Zweck einen eingeschränkten Schlüssel (`restrict,command=`). Host-Schlüssel vorab verteilen und prüfen, `StrictHostKeyChecking` nie abschalten. Unbeaufsichtigt: `BatchMode`, Timeouts, `ssh -n` in Schleifen. Remote-Befehle werden doppelt interpretiert, daher `printf %q` oder feste Skripte mit Argumenten.

> **Praxisbeispiel**
>
> Ein zentrales Wartungsskript verteilt nachts Konfigurationen an 40 Server. Es nutzt einen eigenen, auf einen Empfangsbefehl beschränkten Schlüssel, verteilt `known_hosts` aus dem Repository und bricht pro Host nach zehn Sekunden Verbindungsaufbau ab, sodass ein toter Server nicht den Lauf blockiert und im Bericht als Fehler erscheint.

## 7.9 Patch- und Wartungsfenster

Ein *Wartungsfenster* ist ein vereinbarter Zeitraum, in dem Änderungen am Produktivsystem erlaubt sind: Patches einspielen, Kernel aktualisieren, Dienste neu starten. Die Arbeit lässt sich in drei Phasen gliedern: *Vorabchecks* (darf es überhaupt losgehen?), *Durchführung* und *Nachprüfungen* (ist alles wieder gesund?), dazu ein *Rollback-Pfad* für den Misserfolg. Bash ist dafür das ideale Bindeglied, denn es ruft vorhandene Werkzeuge in fester Reihenfolge auf und bricht beim ersten Problem ab.

Zu den Vorabchecks gehören ausreichend freier Platz (Modul 7.4), ein aktuelles, geprüftes Backup (Modul 7.6), ein gesunder Ausgangszustand (alle erwarteten Dienste aktiv, Healthchecks grün, Modul 9.2), keine laufenden Paketprozesse und ein erfasster *Ist-Stand*: Man speichert die Liste der installierten Paketversionen (`dpkg-query -W` beziehungsweise `rpm -qa`) und die Dienststände in Dateien. Dieser Ist-Stand ist die Grundlage für die *Differenz* nach dem Patchen (was wurde tatsächlich geändert?) und für den Rollback (`apt-get install paket=alteversion`). Bei der Durchführung wird der Dienst vorher aus dem Lastverteiler genommen (*Draining*), gepatcht, bei Bedarf neu gestartet und erst nach bestandener Nachprüfung wieder aufgenommen. Wenn ein Neustart des Systems nötig ist, zeigt eine Datei wie `/var/run/reboot-required` (Debian) oder `needs-restarting -r` (Red Hat) das an.

Nachprüfungen sollen *objektiv* sein: Dienste aktiv und stabil, Endpunkte antworten (Smoke Test), keine neuen Fehler im Journal, Versionen wie erwartet. Der Rollback-Pfad wird *vor* dem Fenster dokumentiert und möglichst erprobt: Welche Versionen werden zurückgesetzt? Wie stellt man das Backup wieder her? Wer entscheidet? Zeitlimits (*Go/No-Go* nach 30 Minuten) verhindern, dass man sich ohne Entscheidung in einen halbfertigen Zustand verrennt. Alles wird protokolliert (Modul 8.8): Wer hat wann was ausgeführt, mit welchem Ergebnis?

```bash
#!/usr/bin/env bash
# wartungsfenster.sh - Vorabchecks, Ist-Stand, Durchfuehrung (simuliert), Nachpruefung, Differenz
set -euo pipefail
lauf="$(mktemp -d)"; trap 'rm -rf -- "$lauf"' EXIT
log() { printf '%s %s\n' "$(date -u +%H:%M:%S)" "$*"; }
fehler=0
FREI_MIN_PROZENT="${FREI_MIN_PROZENT:-20}"

# --- Phase 1: Vorabchecks (jeder Check meldet OK/FAIL, am Ende entscheidet die Summe)
check() {                                 # check BESCHREIBUNG BEFEHL...
    local text="$1"; shift
    if "$@" >/dev/null 2>&1; then log "[ OK ] $text"; else log "[FAIL] $text"; fehler=$((fehler + 1)); fi
}
genug_platz() {
    local frei; frei="$(df -P / | awk 'NR==2 {gsub("%","",$5); print 100 - $5}')"
    (( frei >= FREI_MIN_PROZENT ))
}
kein_paketprozess() { ! pgrep -x -f 'apt-get|dpkg|dnf|zypper' >/dev/null; }
backup_aktuell()    { [[ -n $(find "${BACKUP_DIR:-/nonexistent}" -maxdepth 1 -mmin -1440 2>/dev/null | head -n 1) ]]; }

mkdir -p "$lauf/backups"; touch "$lauf/backups/20261030-0200"    # simuliertes frisches Backup
BACKUP_DIR="$lauf/backups"
log "== Vorabchecks"
check "mind. ${FREI_MIN_PROZENT}% Platz auf /"  genug_platz
check "kein laufender Paketprozess"             kein_paketprozess
check "Backup juenger als 24 h"                 backup_aktuell
check "bash vorhanden"                          command -v bash
(( fehler == 0 )) || { log "NO-GO: $fehler Check(s) fehlgeschlagen"; exit 1; }
log "GO: alle Vorabchecks bestanden"

# --- Ist-Stand festhalten
stand_vorher="$lauf/pakete-vorher.txt"
dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | sort > "$stand_vorher" || true
log "Ist-Stand: $(wc -l < "$stand_vorher") Pakete gesichert"

# --- Phase 2: Durchfuehrung (hier simuliert: ein Paket "wird aktualisiert")
log "== Durchfuehrung (simuliert)"
stand_nachher="$lauf/pakete-nachher.txt"
sed 's/^bash .*/bash 99.0-patched/' "$stand_vorher" > "$stand_nachher"

# --- Phase 3: Nachpruefung und Differenz
log "== Nachpruefung"
check "Healthcheck-Endpunkt (simuliert)"  true
log "Geaenderte Pakete:"
diff <(cat "$stand_vorher") <(cat "$stand_nachher") | grep '^[<>]' | sed 's/^/    /' || true

# --- Rollback-Hinweis aus dem Ist-Stand erzeugen
log "Rollback-Befehle (nur Anzeige):"
join -j1 <(cut -d' ' -f1,2 "$stand_vorher") <(cut -d' ' -f1,2 "$stand_nachher") |
    awk '$2 != $3 {print "    apt-get install -y " $1 "=" $2}'
[[ -e /var/run/reboot-required ]] && log "HINWEIS: Neustart erforderlich" || log "kein Neustart noetig"
```

Die Phase der Vorabchecks ist als Liste kleiner Prüffunktionen gebaut, die `check` ausführt und zählt; wichtig ist die *Entscheidung am Ende*: Bei einem einzigen Fehlschlag wird das Fenster abgebrochen („No-Go“). Alle Checks laufen durch, damit der Bericht *alle* Probleme zeigt, nicht nur das erste. Vor der Änderung wird der Ist-Stand der Paketversionen in eine Datei geschrieben. Die Durchführung ist hier nur simuliert (ein Paket erhält eine neue Version), damit das Beispiel gefahrlos laufen kann. Nach der Nachprüfung vergleicht `diff` den Stand vorher und nachher und zeigt geänderte Pakete; aus der Differenz erzeugt `join` in Verbindung mit `awk` die konkreten Rollback-Befehle mit den alten Versionen. Der Hinweis auf die Neustart-Datei rundet den Bericht ab.

Typische Fehler: Erstens beginnt man ohne frisches Backup oder ohne Test, ob es lesbar ist. Zweitens fehlt ein Ist-Stand, und nach dem Patchen weiß niemand, was sich geändert hat. Drittens wird `apt-get upgrade` ohne Simulation und Eingrenzung ausgeführt, und ein ungeplantes Major-Upgrade ändert Dienste. Viertens wartet man nach dem Neustart nicht auf stabile Dienste (Modul 7.3). Fünftens werden Cluster-Knoten alle gleichzeitig gepatcht, statt einzeln nacheinander mit Prüfung (*Rolling Update*). Sechstens fehlt ein dokumentierter Rollback oder er wurde nie erprobt. Siebtens überlebt das Skript keine Unterbrechung, weil es nicht wiederanlauffähig ist (Idempotenz, Modul 6.7), oder es läuft aus der SSH-Sitzung heraus und stirbt bei einem Verbindungsabbruch (`tmux`, `screen` oder `systemd-run` nutzen).

```bash
# falsch: blind patchen, keine Pruefung, kein Rollback
ssh server 'apt-get -y dist-upgrade && reboot'

# richtig: Vorabcheck, Ist-Stand, kontrollierte Durchfuehrung, Nachpruefung
./vorabchecks.sh        || exit 1
dpkg-query -W > "vorher-$(date +%F).txt"
apt-get -s upgrade | tee plan.txt            # erst simulieren
apt-get -y upgrade && ./nachpruefung.sh || echo "Rollback mit vorher-*.txt"
```

> **Merke**
>
> Drei Phasen: Vorabchecks (Go/No-Go), Durchführung, Nachprüfung. Ist-Stand vorher sichern, Differenz nachher zeigen, daraus Rollback ableiten. Backup und Rollback vorher prüfen. Nacheinander statt alles gleichzeitig. Alles protokollieren, Zeitlimit für Entscheidungen.

> **Praxisbeispiel**
>
> Ein monatliches Wartungsfenster für 20 Server läuft nach Skript: Vorabchecks je Server, Patchen in Gruppen zu je zwei, Nachprüfung, dann die nächste Gruppe. Schlägt eine Prüfung fehl, stoppt der gesamte Lauf, und der Bericht enthält die Versionen vorher und nachher als Beleg für die Change-Dokumentation.

## 7.10 Übungsaufgaben zu Modul 07

Die Aufgaben 1 bis 4 prüfen das Verständnis, die Aufgaben 5 bis 8 sind praktische Programmieraufgaben mit steigender Schwierigkeit.

**Aufgabe 1.** Was unterscheidet `usermod -G wiki anna` von `usermod -aG wiki anna`? Wie prüfen Sie exakt, ob ein Benutzer existiert, und warum ist `grep anna /etc/passwd` dafür ungeeignet?

**Aufgabe 2.** Nennen Sie vier Maßnahmen, die eine Paketinstallation *nicht-interaktiv und sicher* machen, und erklären Sie, warum in Skripten `apt-get` statt `apt` verwendet wird.

**Aufgabe 3.** Warum genügt es nicht, nach `systemctl restart dienst` auf den Exit-Code 0 zu vertrauen? Beschreiben Sie, wie Sie stabilen Betrieb prüfen und welche Journal-Optionen in Skripten nötig sind.

**Aufgabe 4.** Erklären Sie, warum ein Backup ohne Restore-Test wenig wert ist, und nennen Sie drei Gefahren von `rsync --delete`. Welche Rolle spielt der Schrägstrich am Ende eines rsync-Quellpfads?

**Aufgabe 5.** Schreiben Sie `speicher-warnung.sh PFAD [SCHWELLE]`, das die Belegung des Dateisystems von PFAD prüft, bei Überschreiten die fünf größten Unterverzeichnisse anzeigt und mit Exit-Code 0 (ok), 1 (Warnung ab SCHWELLE, Standard 80) oder 2 (kritisch ab 95) endet. Eingaben werden validiert.

**Aufgabe 6.** Schreiben Sie `rotiere.sh VERZ MUSTER BEHALTEN`, das in VERZ die Dateien nach MUSTER (zum Beispiel `backup-*.tar.gz`) nach Namen sortiert und alle bis auf die letzten BEHALTEN löscht. Es hat einen Trockenlauf als Standard (`-{}-force` löscht), verweigert BEHALTEN kleiner als 1 und geht mit Dateinamen mit Leerzeichen korrekt um.

**Aufgabe 7.** Schreiben Sie `dienste-pruefen.sh LISTE`, das aus einer Textdatei (ein Dienst pro Zeile) für jeden Dienst `systemctl is-active` abfragt (über die Variable `SYSTEMCTL` ersetzbar), eine Tabelle ausgibt und mit Exit-Code 1 endet, wenn mindestens ein Dienst nicht aktiv ist. Testen Sie es mit einem Stub-Skript, das `systemctl` nachahmt.

**Aufgabe 8.** Schreiben Sie `sync-konfig.sh QUELLE ZIEL`, das alle `*.conf`-Dateien aus QUELLE nach ZIEL kopiert, aber nur Dateien ersetzt, deren Inhalt sich geändert hat (Vergleich mit `cmp`), jede ersetzte Datei atomar austauscht, die Anzahl geänderter Dateien ausgibt und mit Exit-Code 10 endet, wenn etwas geändert wurde (damit ein aufrufendes Skript einen Reload auslösen kann), sonst mit 0.

## 7.11 Lösungen zu den Übungsaufgaben

**Lösung zu Aufgabe 1.** `-G` ersetzt die gesamte Liste der Zusatzgruppen durch die angegebenen; alle bisherigen Mitgliedschaften gehen verloren. `-aG` (append) fügt die Gruppe zu den bestehenden hinzu. Die exakte Existenzprüfung ist `id -u -- anna >/dev/null 2>&1` (Exit-Code 0 bei Existenz) oder `getent passwd anna`. `grep anna /etc/passwd` findet auch `hanna` oder Kommentarzeilen, kennt keine Konten aus LDAP/SSSD und prüft nicht den Namen, sondern irgendeinen Treffer in der Zeile.

**Lösung zu Aufgabe 2.** (1) `-y` beziehungsweise `-{}-non-interactive`; (2) `DEBIAN_FRONTEND=noninteractive` gegen Konfigurationsdialoge, plus `-o Dpkg::Options::=--force-confold` für geänderte Konfigurationsdateien; (3) `-o DPkg::Lock::Timeout=120`, damit das Skript auf Sperren anderer Paketprozesse wartet statt zu scheitern; (4) vorher mit `dpkg -s` prüfen und nur Fehlendes in *einem* Aufruf installieren, gegebenenfalls zuerst mit `-s` simulieren; zusätzlich Paketnamen validieren und `-{}-` setzen. `apt` ist für Menschen gedacht und ändert Ausgabe und Verhalten ohne Vorankündigung (die Handbuchseite warnt vor einem instabilen CLI); `apt-get` hat eine stabile Schnittstelle.

**Lösung zu Aufgabe 3.** `restart` kehrt häufig zurück, sobald der Startvorgang eingeleitet ist; ein Dienst, der direkt danach abstürzt, wird nicht erkannt. Man wartet mit Zeitgrenze, bis `systemctl is-active --quiet dienst` erfolgreich ist, und prüft dann über einige Sekunden, ob er aktiv *bleibt*; ergänzend ein Smoke Test (Endpunkt antwortet). Bei Misserfolg zeigt `journalctl -u dienst -n 20 --no-pager -o cat` die letzten Meldungen; `-{}-no-pager` verhindert, dass in Skripten ein Pager wartet.

**Lösung zu Aufgabe 4.** Ein Backup, das nie zurückgespielt wurde, kann unvollständig, beschädigt, verschlüsselt ohne verfügbaren Schlüssel oder schlicht in einem unbekannten Format sein; man erfährt es erst im Ernstfall. Gefahren von `-{}-delete`: (1) Ziel und Quelle vertauscht, dann wird das Original geleert; (2) leere oder falsche Variable macht das Ziel zum Spiegel eines leeren Verzeichnisses; (3) ein defekter oder nicht eingehängter Quellpfad ist leer, und das Backup wird mit gelöscht. Abhilfe: `-n` testen, Variablen mit `${var:?}` sichern, Existenz und Plausibilität prüfen. `quelle/` synchronisiert den Inhalt des Verzeichnisses, `quelle` (ohne Schrägstrich) legt im Ziel ein Unterverzeichnis `quelle` an.

**Lösung zu Aufgabe 5.**

```bash
#!/usr/bin/env bash
# speicher-warnung.sh PFAD [SCHWELLE]
set -euo pipefail
pfad="${1:?Aufruf: $0 PFAD [SCHWELLE]}"; schwelle="${2:-80}"
[[ -d $pfad ]] || { echo "kein Verzeichnis: $pfad" >&2; exit 3; }
[[ $schwelle =~ ^[0-9]{1,3}$ ]] && (( schwelle >= 1 && schwelle <= 100 )) \
    || { echo "SCHWELLE muss 1..100 sein" >&2; exit 3; }

prozent="$(df -P -- "$pfad" | awk 'NR==2 {gsub("%","",$5); print $5}')"
echo "Belegung von $pfad: ${prozent}%"

if (( prozent >= 95 )); then status=2
elif (( prozent >= schwelle )); then status=1
else status=0; fi

if (( status > 0 )); then
    echo "Groesste Unterverzeichnisse:"
    du -xk --max-depth=1 -- "$pfad" 2>/dev/null | sort -rn | sed -n '2,6p' |
        awk '{ printf "  %8.1f MB  %s\n", $1/1024, $2 }'
fi
exit "$status"
```

Das Skript leitet seinen Exit-Code aus dem Messwert ab: 0, 1 oder 2, sodass Scheduler und Monitoring die Schweregrade auswerten können. Die Verzeichnisanalyse läuft nur im Fehlerfall, das spart Zeit im Normalfall. Die Schwelle wird als Zahl im Bereich 1 bis 100 validiert; ein Fehler bei der Benutzung hat den eigenen Exit-Code 3, damit er nicht mit „Warnung“ verwechselt wird. Testen lässt sich das Skript mit `SCHWELLE=1` (löst Warnung aus).

**Lösung zu Aufgabe 6.**

```bash
#!/usr/bin/env bash
# rotiere.sh VERZ MUSTER BEHALTEN [--force]
set -euo pipefail
verz="${1:?Aufruf: $0 VERZ MUSTER BEHALTEN [--force]}"; muster="${2:?}"; behalten="${3:?}"
force="${4:-}"
[[ -d $verz ]] || { echo "kein Verzeichnis: $verz" >&2; exit 2; }
[[ $behalten =~ ^[0-9]+$ ]] && (( behalten >= 1 )) || { echo "BEHALTEN muss >= 1 sein" >&2; exit 2; }

mapfile -d '' -t dateien < <(find "$verz" -maxdepth 1 -type f -name "$muster" -print0 | sort -z)
anzahl=${#dateien[@]}
echo "gefunden: $anzahl, behalten: $behalten"
(( anzahl > behalten )) || { echo "nichts zu loeschen"; exit 0; }

for datei in "${dateien[@]:0:anzahl-behalten}"; do
    if [[ $force == --force ]]; then rm -f -- "$datei"; echo "geloescht:   $datei"
    else echo "Trockenlauf: $datei"; fi
done
```

Dateinamen werden mit `-print0` und `mapfile -d ''` über Nullbytes übergeben, damit Leerzeichen und sogar Zeilenumbrüche im Namen keine Probleme machen. Die Sortierung nach Name funktioniert bei Zeitstempeln im Namen als Zeitreihenfolge. Die Array-Schnittstelle `${dateien[@]:0:anzahl-behalten}` liefert alle *bis auf die letzten* BEHALTEN. Der Trockenlauf ist Standard, die Mindestanzahl 1 schützt vor dem Löschen aller Sicherungen.

**Lösung zu Aufgabe 7.**

```bash
#!/usr/bin/env bash
# dienste-pruefen.sh LISTE
set -uo pipefail
liste="${1:?Aufruf: $0 LISTE}"
SC="${SYSTEMCTL:-systemctl}"
[[ -f $liste ]] || { echo "Liste fehlt: $liste" >&2; exit 2; }

nicht_aktiv=0
printf '%-20s %s\n' DIENST STATUS
while IFS= read -r dienst || [[ -n $dienst ]]; do
    [[ -z $dienst || $dienst == \#* ]] && continue
    [[ $dienst =~ ^[A-Za-z0-9@:._-]+$ ]] || { echo "ungueltig: $dienst" >&2; nicht_aktiv=$((nicht_aktiv+1)); continue; }
    status="$("$SC" is-active -- "$dienst" 2>/dev/null)" || true
    printf '%-20s %s\n' "$dienst" "${status:-unbekannt}"
    [[ $status == active ]] || nicht_aktiv=$((nicht_aktiv + 1))
done < "$liste"
(( nicht_aktiv == 0 )) || { echo "$nicht_aktiv Dienst(e) nicht aktiv" >&2; exit 1; }
```

Der Stub für den Test: `printf '#!/bin/bash\n[[ $2 == -- && $3 == gut ]] && echo active || { echo inactive; exit 3; }\n' > stub; chmod +x stub; SYSTEMCTL=./stub ./dienste-pruefen.sh liste.txt`. Die Variable `nicht_aktiv` steht in der Hauptshell, weil die Schleife per Umleitung (nicht per Pipe) gespeist wird; bei einer Pipe ginge der Zähler in der Subshell verloren (Modul 4.6). Das Muster `read -r dienst || [[ -n $dienst ]]` behandelt auch eine letzte Zeile ohne Zeilenumbruch.

**Lösung zu Aufgabe 8.**

```bash
#!/usr/bin/env bash
# sync-konfig.sh QUELLE ZIEL
set -euo pipefail
quelle="${1:?Aufruf: $0 QUELLE ZIEL}"; ziel="${2:?}"
[[ -d $quelle && -d $ziel ]] || { echo "Verzeichnis fehlt" >&2; exit 2; }

geaendert=0
shopt -s nullglob
for f in "$quelle"/*.conf; do
    name="${f##*/}"
    if [[ -f $ziel/$name ]] && cmp -s -- "$f" "$ziel/$name"; then
        continue
    fi
    tmp="$(mktemp "$ziel/.$name.XXXXXX")"
    cp -- "$f" "$tmp"
    chmod 644 "$tmp"
    mv -f -- "$tmp" "$ziel/$name"            # atomar
    echo "aktualisiert: $name"
    geaendert=$((geaendert + 1))
done
echo "geaenderte Dateien: $geaendert"
(( geaendert == 0 )) || exit 10
```

Das Skript vergleicht Inhalt statt Zeitstempel (`cmp -s`), sodass unveränderte Dateien nicht angefasst werden und ein Reload nur bei echten Änderungen nötig ist. Die temporäre Datei liegt im Zielverzeichnis, damit `mv` ein atomares Umbenennen ist; leere Treffer werden durch `shopt -s nullglob` korrekt als „keine Dateien“ behandelt, nicht als wörtliches Muster. Der Exit-Code 10 ist ein vereinbartes Signal („geändert“) und kein Fehler: Ein aufrufendes Skript prüft ihn gezielt (`./sync-konfig.sh a b; rc=$?; (( rc == 10 )) && systemctl reload app`). Mit `set -e` muss man den Aufruf deshalb in einer `if`-Bedingung oder mit `|| rc=$?` schreiben.
