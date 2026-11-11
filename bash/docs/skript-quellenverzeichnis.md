# Quellenverzeichnis

> **Hinweis:** Teil des Skripts „Bash – Vom Einstieg bis zur Automation“, aus der LaTeX-Quelle in Markdown umgewandelt. Die Kapitel zu den Modulen stehen als `skript.md` in den Modulordnern.

Das Verzeichnis trennt Quellen, die im Rahmen der Nachrecherche *tatsächlich abgerufen* wurden, von Material, das *weiterführend* empfohlen wird, aber nicht ausgewertet wurde. Rechtliche Hinweise (Modul 08, Datenschutz) sind allgemeine Information und **keine Rechtsberatung**.

## Tatsächlich abgerufen (Stand 30.10.2026)

- ShellCheck Wiki: SC2086 – Double quote to prevent globbing and word splitting. <https://shellcheck.net/wiki/SC2086>. Verwendet in Modul 01 (1.8) und Modul 03 (3.3).

- OMG! Ubuntu: Bash 5.3 neue Funktionen (Veröffentlichung am 7. Juli 2025). <https://www.omgubuntu.co.uk/2025/07/bash-5-3-new-features>. Verwendet in Modul 01 (1.4) und Modul 04 (4.3).

- jq Projekt: NEWS zu jq 1.8.x (Änderungen gegenüber 1.7). <https://codegraph.jelmer.uk/jq/1.8.2-1/NEWS.md>. Verwendet in Modul 05 (5.6).

- ripgrep: README des Projekts (BurntSushi/ripgrep). <https://github.com/BurntSushi/ripgrep>. Verwendet in Modul 05 (5.1).

- Greg’s Wiki: BashFAQ/105 – Why doesn’t set -e do what I expected? <https://mywiki.wooledge.org/BashFAQ/105>. Verwendet in Modul 06 (6.1).

- OWASP Cheat Sheet Series: OS Command Injection Defense Cheat Sheet. <https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html>. Verwendet in Modul 08 (8.3).

- Verordnung (EU) 2016/679 (DSGVO), Art. 32 Sicherheit der Verarbeitung, Textwiedergabe bei gesetze.legal. <https://gesetze.legal/eu/vo_eu_2016_679/32>. Verwendet in Modul 08 (8.9); keine Rechtsberatung.

- systemd.timer(5), Handbuchseite (Debian manpages). <https://manpages.debian.org/buster/systemd/systemd.timer.5.en.html>. Verwendet in Modul 10 (10.1). Es handelt sich um die Seite der Debian-Version *buster*; Optionen können in neueren systemd-Versionen abweichen.

- GitHub Docs: Setting default values for jobs (`defaults.run`). <https://docs.github.com/actions/using-jobs/setting-default-values-for-jobs>. Verwendet in Modul 10 (10.4).

- Robust Perception: Atomic writes and the textfile collector. <https://robustperception.io/atomic-writes-and-the-textfile-collector>. Verwendet in Modul 09 (9.3).

- Terraform CLI: `terraform plan` (Dokumentation, Option `-detailed-exitcode`), wiedergegeben bei Mintlify. <https://www.mintlify.com/hashicorp/terraform/cli/plan>. Verwendet in Modul 11 (11.2). Es ist eine Drittseite, nicht die offizielle HashiCorp-Dokumentation; maßgeblich ist die Dokumentation der eingesetzten Terraform-Version.

## Weiterführend (nicht abgerufen)

Die folgenden Quellen sind maßgeblich für die behandelten Themen. Ein Abruf war in der Arbeitsumgebung nicht möglich beziehungsweise wurde nicht ausgewertet; die Inhalte dieses Skripts stützen sich daher nicht auf diese Seiten, sondern auf das Fachwissen des Autors.

- GNU Bash Reference Manual. <https://www.gnu.org/software/bash/manual/bash.html> (Abruf blockiert).

- The Open Group: POSIX Shell Command Language. <https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html>.

- Filesystem Hierarchy Standard. <https://refspecs.linuxfoundation.org/fhs.shtml>.

- ShellCheck-Projektseite. <https://www.shellcheck.net/> (Abruf blockiert).

- LWN.net: Artikel zu Bash 5.3 (nur als Suchergebnis gesehen, nicht ausgewertet). <https://lwn.net/Articles/1029079>.

- DSGVO, Art. 5 (Grundsätze der Verarbeitung: Datenminimierung, Speicherbegrenzung); amtlicher Text auf EUR-Lex. Nicht abgerufen; keine Rechtsberatung.

- Dokumentation von curl, GnuPG und sudoers (`man curl`, `man gpg`, `man sudoers`). Nicht abgerufen.

- Werkzeug-Dokumentation zu Modul 11, jeweils nicht abgerufen: `aws`, `az`, `gcloud`, Docker, `kubectl`, Open Policy Agent/Conftest, Checkov, Trivy, GitHub-Dokumentation zu OIDC. Die Beispiele wurden gegen Attrappen getestet, nicht gegen echte Dienste.

- Git-Dokumentation zu Modul 12: `git bundle`, `git interpret-trailers`, `git log`. Nicht abgerufen; das beschriebene Verhalten von `git bundle verify` bei abgeschnittenen Dateien wurde im Test des Autors beobachtet, nicht einer Quelle entnommen.

- Handbuchseiten des Systems: `man bash`, `man hier`, `man find`, `man tar`, `man signal`.
