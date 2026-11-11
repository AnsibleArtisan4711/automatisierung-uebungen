# Modul 12 - Alerting- Mail & Webhook

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (12.1 bis 12.5), danach folgen „Übungsaufgaben zu Modul 12“ (8 Aufgaben) und „Lösungen zu Modul 12“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 01.02.2027 bis Fr 05.02.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul schließt den Admin-Teil des Kurses ab, denn erst eine Meldung an einen Menschen macht aus einer Messung eine Überwachung)

**Meilenstein:** Health-Check mit Alerting

**Lernziele.** Nach diesem Modul können Sie E-Mails mit `smtplib` und dem Paket `email` korrekt bauen und über einen Mailserver mit Verschlüsselung und Anmeldung versenden, Webhooks für Chat-Werkzeuge wie Slack und Microsoft Teams ansprechen und dabei Fehler, Wiederholungen und Geheimnisse richtig behandeln, einen Health-Check schreiben, der Dienste und Ressourcen prüft, und die Meldungen so steuern, dass Menschen weder zu spät noch zu oft gestört werden (Zustandsspeicher, Schwellen, Erinnerungen, Entwarnung). Am Ende steht ein vollständiges, getestetes Programm, das Sie aus Modul 9 heraus zeitgesteuert starten können.

| **Nr.** | **Lerneinheit**                                              | **Datum**     | **Dauer** | **Prio** |
|:--------|:-------------------------------------------------------------|:--------------|:----------|:---------|
| 12.1    | Mail-Versand mit smtplib                                     | Mo 01.02.2027 | 2 h       | 2        |
| 12.2    | Webhooks                                                     | Di 02.02.2027 | 2 h       | 2        |
| 12.3    | Health-Check Teil 1                                          | Mi 03.02.2027 | 2 h       | 2        |
| 12.4    | Health-Check Teil 2 (Alerting)                               | Do 04.02.2027 | 2 h       | 2        |
| 12.5    | Health-Check abschließen, Themenwahl Abschlussprojekt + Quiz | Fr 05.02.2027 | 2 h       | 2        |
|         | **Summe**                                                    |               | **10 h**  |          |

> **Merke**
>
> Für dieses Modul brauchen Sie `requests` (Modul 10) und `psutil` (Modul 11); für die lokalen Testserver zusätzlich `aiosmtpd` (`pip install aiosmtpd`) und das Programm `openssl`, mit dem das Hilfsmodul `testsmtp` ein kurzlebiges Testzertifikat erzeugt. Getestet wurde mit Python 3.13, aiosmtpd 1.4.6 und requests 2.34.2 unter Linux. Alle Beispiele laufen vollständig auf Ihrem eigenen Rechner: Die Mails landen in einem Testserver im Arbeitsspeicher, und die Webhooks gehen an einen kleinen lokalen Empfänger. Es werden weder echte Mails noch echte Chat-Nachrichten verschickt, und es kommen keine echten Zugangsdaten vor. Die Hilfsmodule bauen aufeinander auf; speichern Sie jedes Modul unter dem in der Überschrift genannten Namen im selben Ordner.

## 12.1 Mail-Versand mit smtplib

Eine Überwachung, die niemand bemerkt, hilft nicht. Der Health-Check, den wir in diesem Modul bauen, soll Menschen benachrichtigen, wenn etwas schiefgeht, und der älteste, am weitesten verbreitete Weg dafür ist die E-Mail. Jeder Server, jedes Heimnetz und jede Firma hat einen Mailzugang, Mail braucht keine zusätzliche Software auf dem Empfängerrechner, und Mails bleiben im Postfach, auch wenn der Empfänger gerade nicht online ist. Die Standardbibliothek von Python liefert alles Nötige mit: das Paket `email` baut Nachrichten, das Modul `smtplib` spricht mit dem Mailserver.

**Wie Mail funktioniert.** Zunächst ein Blick auf das Protokoll, denn viele Fehler entstehen aus einem falschen Bild davon. *SMTP* (*Simple Mail Transfer Protocol*) ist das Protokoll, mit dem Mail *eingeliefert* und von Server zu Server *weitergereicht* wird. Der Ablauf gleicht dem Einwerfen eines Briefes: Ihr Programm (der *Mail User Agent*, *MUA*) verbindet sich mit einem *Mail-Einlieferungsserver* (*Submission*), meldet sich bei Bedarf an und übergibt die Nachricht; dieser Server leitet sie an den Server des Empfängers weiter, und der legt sie im Postfach ab. Das Lesen aus dem Postfach geschieht mit anderen Protokollen (*IMAP*, *POP3*) und ist hier nicht Thema. Wichtig ist: Ihr Programm spricht nur mit *einem* Server, nämlich dem, bei dem es sich anmelden darf, meist dem Mailserver der eigenen Firma oder des eigenen Anbieters.

Eine Besonderheit führt zu vielen Missverständnissen: SMTP unterscheidet den *Umschlag* (*envelope*) von der *Nachricht* selbst. Der Umschlag besteht aus der Absender- und den Empfängeradressen, die dem Server mit den Befehlen `MAIL FROM` und `RCPT TO` genannt werden; nur danach richtet sich die Zustellung. Die Nachricht, die danach mit `DATA` folgt, hat ihre eigenen Kopfzeilen (`From`, `To`, `Cc`, `Subject`, …), die das Mailprogramm des Empfängers anzeigt. Beide sind unabhängig: Man kann an jemanden senden, der in den Kopfzeilen gar nicht vorkommt. Genau so funktioniert die *Blindkopie* (`Bcc`): Der Empfänger steht im Umschlag, aber nicht in den Kopfzeilen. Die Dokumentation von `smtplib` (Python 3.7, abgerufen) bestätigt, dass `send_message` die Adressen aus den Kopfzeilen `To`, `Cc` und `Bcc` für den Umschlag nimmt und eine `Bcc`-Kopfzeile nicht mitschickt, und dass das ältere `sendmail` die Kopfzeilen überhaupt nicht verändert, sie also vom Programmierer vollständig in den Nachrichtentext gehören.

**Ein Testserver für die Beispiele.** Zum Üben brauchen wir einen Mailserver, bei dem nichts passieren kann, wenn wir einen Fehler machen. Das Hilfsmodul `testsmtp` startet mit der Bibliothek `aiosmtpd` einen kleinen Server auf `127.0.0.1` (nur der eigene Rechner kann ihn erreichen), der alle Mails nur in einer Liste im Speicher sammelt. Er kann in drei Betriebsarten laufen: im Klartext, mit *STARTTLS* und mit implizitem TLS (beides erklären wir gleich), und er kann Anmeldung verlangen. Für die verschlüsselten Arten erzeugt er mit `openssl` ein Zertifikat, das einen Tag gültig ist. Empfänger der Domäne `gesperrt.example` lehnt er ab, damit wir Fehlerfälle üben können. Sie müssen den Code nicht im Einzelnen verstehen: Alles, was Sie für die Praxis brauchen, steht in den folgenden Beispielen, das Hilfsmodul sorgt nur dafür, dass es sie ohne fremden Mailserver gibt.

```python
"""testsmtp - kleiner lokaler Mailserver fuer die Beispiele von Modul 12 (nur auf 127.0.0.1)."""
import logging
import socket
import ssl
import subprocess
import tempfile
from pathlib import Path

from aiosmtpd.controller import Controller
from aiosmtpd.smtp import AuthResult

logging.getLogger("mail.log").setLevel(logging.CRITICAL)      # keine Server-Meldungen in den Beispielen


class Sammler:
    """Nimmt Mails an und merkt sie sich; Empfaenger @gesperrt.example werden abgelehnt."""

    def __init__(self):
        self.nachrichten = []

    async def handle_RCPT(self, server, session, envelope, address, rcpt_options):
        if address.endswith("@gesperrt.example"):
            return "550 5.1.1 Empfaenger unbekannt"
        envelope.rcpt_tos.append(address)
        return "250 OK"

    async def handle_DATA(self, server, session, envelope):
        self.nachrichten.append({"absender": envelope.mail_from, "empfaenger": list(envelope.rcpt_tos),
                                 "inhalt": envelope.content})
        return "250 Nachricht angenommen"


class Server:
    def __init__(self, modus="klartext", benutzer=None, passwort=None):
        """modus: 'klartext', 'starttls' oder 'ssl' (implizites TLS)."""
        self.sammler = Sammler()
        self.modus = modus
        self.ordner = Path(tempfile.mkdtemp(prefix="testsmtp_"))
        self.zertifikat = self.ordner / "server.pem"
        with socket.socket() as s:
            s.bind(("127.0.0.1", 0))
            self.port = s.getsockname()[1]
        optionen = {}
        if modus != "klartext":
            self._zertifikat_erzeugen()
            kontext = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
            kontext.load_cert_chain(self.zertifikat, self.ordner / "server.key")
            optionen = {"ssl_context": kontext} if modus == "ssl" else {"tls_context": kontext, "require_starttls": True}
        if benutzer:
            def pruefe(server, session, envelope, mechanism, auth_data):
                ok = (auth_data.login == benutzer.encode() and auth_data.password == passwort.encode())
                return AuthResult(success=ok, handled=False)
            optionen.update(authenticator=pruefe, auth_required=True, auth_require_tls=(modus == "starttls"))
        self._controller = Controller(self.sammler, hostname="127.0.0.1", port=self.port, **optionen)
        self._controller.start()

    def _zertifikat_erzeugen(self):
        subprocess.run(["openssl", "req", "-x509", "-newkey", "rsa:2048", "-nodes", "-days", "1",
                        "-subj", "/CN=localhost", "-addext", "subjectAltName=DNS:localhost,IP:127.0.0.1",
                        "-keyout", str(self.ordner / "server.key"), "-out", str(self.zertifikat)],
                       check=True, capture_output=True)

    def stoppe(self):
        self._controller.stop()
```

Die Klasse `Server` nimmt als Argumente die Betriebsart (`"klartext"`, `"starttls"` oder `"ssl"`) und optional Benutzer und Passwort; `port` verrät, wo er lauscht, `sammler.nachrichten` enthält die angenommenen Mails (Umschlagabsender, Umschlagempfänger und den rohen Inhalt), `zertifikat` den Pfad des Testzertifikats, und `stoppe()` beendet ihn. Beachten Sie, dass der Testserver absichtlich ein *selbst ausgestelltes* Zertifikat verwendet: Das gibt uns die Gelegenheit zu zeigen, was die Prüfung von Zertifikaten leistet.

**Die erste Mail.** Eine Nachricht baut man mit der Klasse `email.message.EmailMessage`. Man setzt die Kopfzeilen wie bei einem Wörterbuch (`nachricht["Subject"] = ...`) und den Inhalt mit `set_content()`. Das Paket kümmert sich um die vielen Feinheiten, die früher von Hand zu erledigen waren: die richtige Kodierung von Umlauten, das Zeilenende `\r\n`, die Erzeugung der MIME-Struktur. Versendet wird mit `smtplib.SMTP(host, port)` und `send_message()`; der Aufruf mit `with` sorgt dafür, dass die Verbindung ordentlich beendet wird (Befehl `QUIT`), auch wenn dazwischen ein Fehler auftritt. Das folgende Beispiel verschickt eine Mail mit `To`, `Cc` und `Bcc` und vergleicht anschließend, was der Server als *Umschlag* bekam und was in den *Kopfzeilen* steht. Die empfangene Nachricht wird mit `email.message_from_bytes` wieder zerlegt.

```python
import smtplib
from email import message_from_bytes, policy
from email.message import EmailMessage

import testsmtp

server = testsmtp.Server()                       # lokaler Mailserver auf 127.0.0.1

nachricht = EmailMessage()
nachricht["From"] = "monitoring@beispiel.example"
nachricht["To"] = "admin@beispiel.example"
nachricht["Cc"] = "chef@beispiel.example"
nachricht["Bcc"] = "archiv@beispiel.example"
nachricht["Subject"] = "Testmail vom Monitoring"
nachricht.set_content("Hallo,\n\ndies ist eine Testmail.\n")

with smtplib.SMTP("127.0.0.1", server.port, timeout=5) as smtp:
    smtp.send_message(nachricht)

empfangen = server.sammler.nachrichten[0]
print("Umschlag, Absender   :", empfangen["absender"])
print("Umschlag, Empfänger  :", empfangen["empfaenger"])

gelesen = message_from_bytes(empfangen["inhalt"], policy=policy.default)
print("Kopf, Subject        :", gelesen["Subject"])
print("Kopf, To und Cc      :", gelesen["To"], "|", gelesen["Cc"])
print("Kopf enthält Bcc     :", "Bcc" in gelesen)
print("Text                 :", gelesen.get_content().strip().splitlines()[0])
server.stoppe()
```

```text
Umschlag, Absender   : monitoring@beispiel.example
Umschlag, Empfänger  : ['admin@beispiel.example', 'archiv@beispiel.example', 'chef@beispiel.example']
Kopf, Subject        : Testmail vom Monitoring
Kopf, To und Cc      : admin@beispiel.example | chef@beispiel.example
Kopf enthält Bcc     : False
Text                 : Hallo,
```

Die Ausgabe zeigt den Unterschied genau: Im Umschlag stehen drei Empfänger (der Server liefert an alle drei), in den Kopfzeilen nur `To` und `Cc`; die Blindkopie ist unsichtbar. Die Mail selbst wurde mit dem Betreff und dem Text unverändert übertragen.

**Der Aufbau einer Nachricht.** Eine E-Mail ist nur Text, genauer: eine Folge von Kopfzeilen, eine Leerzeile und der Inhalt, wie es der Standard *RFC 5322* beschreibt. Sobald mehr als einfacher Text hineinsoll, kommt *MIME* (*Multipurpose Internet Mail Extensions*) ins Spiel: Eine Nachricht kann aus mehreren *Teilen* bestehen, jedes mit eigenem Inhaltstyp (`text/plain`, `text/html`, `application/pdf`, …). Die Struktur ist ein Baum: `multipart/mixed` enthält Text und Anhänge, `multipart/alternative` enthält dieselbe Nachricht in mehreren Fassungen (Text und HTML), aus denen das Mailprogramm die beste auswählt. `EmailMessage` baut diesen Baum für Sie, wenn Sie `add_alternative()` und `add_attachment()` aufrufen. Das nächste Beispiel erzeugt eine Mail mit Umlauten, einer HTML-Fassung und einem Anhang und zeigt die Struktur, ohne sie zu versenden.

```python
import tempfile
from pathlib import Path

from email.message import EmailMessage

nachricht = EmailMessage()
nachricht["From"] = "monitoring@beispiel.example"
nachricht["To"] = "Jörg Müller <admin@beispiel.example>"
nachricht["Subject"] = "Platte voll auf Server Köln"
nachricht.set_content("Die Platte ist zu 93 % belegt.\n")
nachricht.add_alternative("<p>Die Platte ist <b>zu 93 %</b> belegt.</p>", subtype="html")

log = Path(tempfile.mkdtemp()) / "bericht.log"
log.write_text("Zeile 1\nZeile 2\n", encoding="utf-8")
nachricht.add_attachment(log.read_bytes(), maintype="text", subtype="plain", filename=log.name)

print("Hauptart          :", nachricht.get_content_type())
print("Teile             :", [teil.get_content_type() for teil in nachricht.walk()])
print("Anhänge           :", [teil.get_filename() for teil in nachricht.iter_attachments()])
kopf = nachricht.as_string().split("\n\n", 1)[0].splitlines()
print("Betreff kodiert   :", [z for z in kopf if z.startswith("Subject")][0])

try:
    nachricht["X-Test"] = "harmlos\nBcc: fremder@boese.example"
except ValueError as fehler:
    print("Zeilenumbruch im Kopf abgelehnt:", type(fehler).__name__)
```

```text
Hauptart          : multipart/mixed
Teile             : ['multipart/mixed', 'multipart/alternative', 'text/plain', 'text/html', 'text/plain']
Anhänge           : ['bericht.log']
Betreff kodiert   : Subject: Platte voll auf Server =?utf-8?q?K=C3=B6ln?=
Zeilenumbruch im Kopf abgelehnt: ValueError
```

Man sieht den Baum: Die Hauptnachricht ist `multipart/mixed`, darin liegt ein `multipart/alternative` mit Text- und HTML-Fassung und danach der Anhang als `text/plain`. Der Betreff mit dem Umlaut ist nach den Regeln für Kopfzeilen kodiert: `=?utf-8?q?K=C3=B6ln?=` bedeutet „Zeichensatz UTF-8, quoted-printable, Inhalt K=C3=B6ln“; das Mailprogramm des Empfängers dekodiert es wieder zu „Köln“. Kopfzeilen dürfen im Original nur ASCII-Zeichen enthalten, deshalb ist diese Kodierung nötig; wer `EmailMessage` benutzt, muss nichts davon selbst tun, wer Mails mit Zeichenkettenverkettung baut, schon.

Die letzte Zeile des Beispiels zeigt eine Sicherheitsfunktion: Ein Zeilenumbruch in einer Kopfzeile wird abgelehnt. Das ist der Schutz gegen die *Kopfzeilen-Einschleusung* (*header injection*). Der Hintergrund: Trennt man Kopfzeilen mit Zeilenumbrüchen, so kann ein Angreifer, dessen Eingabe in eine Kopfzeile gelangt (etwa als Betreff), mit einem Zeilenumbruch eigene Kopfzeilen anhängen, zum Beispiel eine `Bcc`-Zeile, und so Ihren Server zum Spamversand missbrauchen. Auf CRLF-Einschleusung als allgemeine Schwachstellenklasse (CWE-93) weist auch die abgerufene Seite von shoulder.dev hin, die für Python empfiehlt, Zeilenumbrüche aus Kopfzeilenwerten zu entfernen. Die moderne Klasse `EmailMessage` schützt Sie davor; ältere Wege (`MIMEText` in alten Versionen, selbst gebaute Zeichenketten) taten das nicht. In unserem späteren `baue_nachricht` wird der Betreff zusätzlich bereinigt, damit ein Zeilenumbruch im Betreff nicht zum Fehler führt.

**Verschlüsselung: STARTTLS und implizites TLS.** Ohne Verschlüsselung läuft eine Mail *im Klartext* über das Netz: Jeder, der mitlesen kann, sieht den Inhalt und, falls angemeldet wird, auch Benutzername und Passwort. Deshalb gehört bei jedem Server außerhalb des eigenen Rechners TLS (*Transport Layer Security*) dazu. Es gibt zwei Wege, und beide kennt `smtplib`. Beim *STARTTLS* verbindet man sich zunächst unverschlüsselt (üblicherweise auf Port 587, dem Einlieferungsport) und *befördert* die Verbindung mit dem Befehl `STARTTLS` zu einer verschlüsselten. Beim *impliziten TLS* (früher „SMTPS“, Port 465) ist die Verbindung von der ersten Sekunde an verschlüsselt. Nach der Beschreibung auf Wikipedia (abgerufen) wurde Port 465 1997 für SMTPS vergeben, 1998 wieder zurückgezogen, weil STARTTLS standardisiert wurde, und durch RFC 8314 für die Einlieferung per implizitem TLS neu eingetragen; dieser RFC empfiehlt implizites TLS ausdrücklich gegenüber STARTTLS. Für Sie in der Praxis heißt das: Beide Wege sind sicher, wenn man das *Zertifikat prüft*; welchen Port Ihr Mailserver anbietet, steht in seiner Dokumentation, und meist sind es 587 mit STARTTLS oder 465 mit implizitem TLS. Der alte Port 25 dient der Weiterleitung zwischen Servern und ist für die Einlieferung von Clients bei vielen Anbietern gesperrt.

Der entscheidende Punkt ist die Prüfung. Eine verschlüsselte Verbindung nützt nur, wenn Sie sicher sind, *mit wem* Sie verschlüsselt sprechen; sonst verschlüsseln Sie womöglich genau für den Angreifer. Dafür dient das *Zertifikat* des Servers: Eine Zertifizierungsstelle bestätigt damit, dass der Schlüssel zu diesem Rechnernamen gehört. In Python übernimmt `ssl.create_default_context()` die Prüfung: Er lädt die vertrauenswürdigen Stellen des Systems und verlangt, dass das Zertifikat gültig ist und zum Rechnernamen passt. Die Dokumentation von `smtplib` zeigt ihn in ihrem Beispiel und erwähnt, dass seit Python 3.4 die Prüfung des Rechnernamens unterstützt wird. Dieser Kontext wird an `starttls(context=...)` oder `SMTP_SSL(..., context=...)` übergeben. Nach `starttls()` ist `ehlo()` erneut nötig, was `smtplib` bei `login` und beim Senden automatisch erledigt. Das folgende Beispiel spielt beide Wege gegen unseren Testserver durch, der Anmeldung verlangt, und zeigt vier typische Misserfolge: unbekanntes Zertifikat, falsches Passwort und das Senden ohne Verschlüsselung.

```python
import smtplib
import ssl
from email.message import EmailMessage

import testsmtp


def mail():
    nachricht = EmailMessage()
    nachricht["From"] = "monitoring@beispiel.example"
    nachricht["To"] = "admin@beispiel.example"
    nachricht["Subject"] = "TLS-Test"
    nachricht.set_content("Hallo\n")
    return nachricht


# Variante 1: STARTTLS (Port 587 im echten Leben). Der Server verlangt Verschlüsselung und Anmeldung.
server = testsmtp.Server("starttls", benutzer="monitor", passwort="geheim")
vertrauen = ssl.create_default_context(cafile=str(server.zertifikat))    # kennt unser Testzertifikat

with smtplib.SMTP("127.0.0.1", server.port, timeout=5) as smtp:
    smtp.starttls(context=vertrauen)
    smtp.login("monitor", "geheim")
    smtp.send_message(mail())
print("1. STARTTLS mit Prüfung des Zertifikats  : angenommen =", len(server.sammler.nachrichten))

try:
    with smtplib.SMTP("127.0.0.1", server.port, timeout=5) as smtp:
        smtp.starttls(context=ssl.create_default_context())              # kennt das Testzertifikat nicht
except ssl.SSLCertVerificationError as fehler:
    print("2. Unbekanntes Zertifikat                :", type(fehler).__name__)

try:
    with smtplib.SMTP("127.0.0.1", server.port, timeout=5) as smtp:
        smtp.starttls(context=vertrauen)
        smtp.login("monitor", "falsch")
except smtplib.SMTPAuthenticationError as fehler:
    print("3. Falsches Passwort                     :", fehler.smtp_code)

try:
    with smtplib.SMTP("127.0.0.1", server.port, timeout=5) as smtp:
        smtp.send_message(mail())                                        # ohne starttls
except smtplib.SMTPSenderRefused as fehler:
    print("4. Ohne Verschlüsselung                  :", fehler.smtp_code, fehler.smtp_error.decode())
server.stoppe()

# Variante 2: implizites TLS (Port 465 im echten Leben): Verschlüsselung von der ersten Sekunde an
server = testsmtp.Server("ssl", benutzer="monitor", passwort="geheim")
vertrauen = ssl.create_default_context(cafile=str(server.zertifikat))
with smtplib.SMTP_SSL("127.0.0.1", server.port, context=vertrauen, timeout=5) as smtp:
    smtp.login("monitor", "geheim")
    smtp.send_message(mail())
print("5. Implizites TLS (SMTP_SSL)             : angenommen =", len(server.sammler.nachrichten))
server.stoppe()
```

```text
1. STARTTLS mit Prüfung des Zertifikats  : angenommen = 1
2. Unbekanntes Zertifikat                : SSLCertVerificationError
3. Falsches Passwort                     : 535
4. Ohne Verschlüsselung                  : 530 Must issue a STARTTLS command first
5. Implizites TLS (SMTP_SSL)             : angenommen = 1
```

Gehen wir die fünf Zeilen durch. In 1. kennt unser Kontext das Testzertifikat (wir haben es mit `cafile=` als vertrauenswürdig eingetragen), STARTTLS gelingt, die Anmeldung auch, die Mail wird angenommen. In 2. benutzt dasselbe Programm den Standardkontext, der unser selbst ausgestelltes Zertifikat zu Recht nicht kennt: Die Verbindung wird mit `SSLCertVerificationError` abgebrochen, ehe Passwort oder Mail übertragen werden. Das ist das gewünschte Verhalten, kein Fehler. In 3. ist die Verschlüsselung in Ordnung, aber das Passwort falsch: Der Server antwortet mit dem Code 535, `smtplib` meldet das als `SMTPAuthenticationError`. In 4. versucht das Programm, ohne STARTTLS zu senden, der Server besteht auf Verschlüsselung (Code 530), und das Senden schlägt als `SMTPSenderRefused` fehl. In 5. läuft der Weg über implizites TLS mit `SMTP_SSL`, bei dem es keinen unverschlüsselten Anfang gibt. Die `SMTP_SSL`-Dokumentation beschreibt die Klasse als die richtige Wahl, wenn TLS von Anfang an verlangt wird und `starttls()` nicht angemessen ist.

**Fehlerarten.** Wie in Modul 11 beim SSH gilt auch hier: Ein Programm, das Mails versendet, muss die verschiedenen Fehlerarten unterscheiden können, denn sie verlangen verschiedene Reaktionen. Ist der Empfänger unbekannt, bringt Wiederholen nichts. Ist der Server nicht erreichbar, hilft später vielleicht ein zweiter Versuch. Stimmt das Passwort nicht, muss ein Mensch etwas ändern. Die Dokumentation nennt die Ausnahmen: `SMTPException` ist die gemeinsame Oberklasse aller Fehler des Moduls (und seit Python 3.4 eine Unterklasse von `OSError`); `SMTPRecipientsRefused` meldet, dass *alle* Empfänger abgelehnt wurden, mit Einzelheiten im Attribut `recipients`; `SMTPSenderRefused` betrifft den Absender, `SMTPAuthenticationError` die Anmeldung, `SMTPDataError` den Inhalt, `SMTPServerDisconnected` den unerwartet geschlossenen Server, `SMTPConnectError` den fehlgeschlagenen Verbindungsaufbau und `SMTPNotSupportedError` fehlende Funktionen des Servers. Hinzu kommen die Fehler des Netzwerks selbst. Das nächste Beispiel erzeugt fünf davon und ordnet jeden einer Meldung zu.

```python
import smtplib
import socket
from email.message import EmailMessage

import testsmtp


def mail(an):
    nachricht = EmailMessage()
    nachricht["From"] = "monitoring@beispiel.example"
    nachricht["To"] = an
    nachricht["Subject"] = "Fehlerarten"
    nachricht.set_content("Hallo\n")
    return nachricht


def versuche(beschreibung, aktion):
    try:
        aktion()
        text = "gesendet"
    except smtplib.SMTPRecipientsRefused as fehler:
        text = "Empfänger abgelehnt: " + ", ".join(f"{a} ({c})" for a, (c, _) in fehler.recipients.items())
    except smtplib.SMTPAuthenticationError:
        text = "Anmeldung abgelehnt"
    except smtplib.SMTPServerDisconnected:
        text = "keine (rechtzeitige) Antwort vom Server"
    except smtplib.SMTPException as fehler:
        text = "SMTP-Fehler: " + type(fehler).__name__
    except (TimeoutError, socket.timeout):
        text = "Zeitüberschreitung"
    except ConnectionRefusedError:
        text = "Verbindung abgelehnt (kein Mailserver auf dem Port)"
    except OSError as fehler:
        text = "Netzwerkfehler: " + type(fehler).__name__
    print(f"{beschreibung:34} -> {text}")


def senden(port, an, host="127.0.0.1", timeout=1):
    with smtplib.SMTP(host, port, timeout=timeout) as smtp:
        smtp.send_message(mail(an))


server = testsmtp.Server()
versuche("gesunder Server", lambda: senden(server.port, "admin@beispiel.example"))
versuche("Empfänger gesperrt", lambda: senden(server.port, "x@gesperrt.example"))
versuche("Port ohne Dienst", lambda: senden(1, "admin@beispiel.example"))

stumm = socket.socket()                    # nimmt Verbindungen an, sagt aber nie etwas
stumm.bind(("127.0.0.1", 0))
stumm.listen()
versuche("Server schweigt", lambda: senden(stumm.getsockname()[1], "admin@beispiel.example"))
stumm.close()
versuche("Name existiert nicht", lambda: senden(25, "admin@beispiel.example", host="kein-server.invalid"))
server.stoppe()
```

```text
gesunder Server                    -> gesendet
Empfänger gesperrt                 -> Empfänger abgelehnt: x@gesperrt.example (550)
Port ohne Dienst                   -> Verbindung abgelehnt (kein Mailserver auf dem Port)
Server schweigt                    -> keine (rechtzeitige) Antwort vom Server
Name existiert nicht               -> Netzwerkfehler: gaierror
```

Beachten Sie zwei Dinge. Erstens: Der „schweigende“ Server, bei dem die Verbindung zwar zustande kommt, der aber nie seine Begrüßung sendet, führt nicht zu einem `TimeoutError`, sondern zu `SMTPServerDisconnected`: `smtplib` wandelt die Zeitüberschreitung beim Lesen der Begrüßung in diese Ausnahme um (in der Meldung steht der Grund). Deshalb steht im Beispiel dafür kein eigener Zweig; für andere Stellen des Ablaufs kann trotzdem ein `TimeoutError` auftreten, weshalb er nicht fehlen sollte. Zweitens: Die Reihenfolge der `except`-Zweige folgt wieder dem Grundsatz von speziell zu allgemein, denn `SMTPException` ist ein `OSError`, und ein früherer `OSError`-Zweig würde sie abfangen.

**Ein wiederverwendbares Modul.** Aus diesen Bausteinen bauen wir ein kleines Modul `mailer.py`, das wir im Rest des Moduls einsetzen. Es besteht aus drei Teilen. Die Klasse `MailKonfig` bündelt Server, Port, Absender, Betriebsart, Zugangsdaten, Zeitlimit und eine optionale eigene Zertifizierungsstelle (für interne Server mit selbst ausgestellten Zertifikaten). Das Passwort-Feld hat `repr=False`; das hat denselben Grund wie in Modul 10, 10.4: Stünde das Passwort in der automatisch erzeugten Textdarstellung, würde es in jedem Protokolleintrag und jedem Traceback auftauchen, in dem die Konfiguration abgebildet wird. Die Funktion `baue_nachricht` erzeugt eine Mail und bereinigt den Betreff. Und `sende` führt den ganzen Ablauf aus (Verbindung, TLS, Anmeldung, Senden) und übersetzt *alle* Fehler in eine einzige Ausnahme `MailFehler` mit einer verständlichen Meldung, wobei die ursprüngliche Ausnahme mit `raise ... from ...` als Ursache erhalten bleibt. Das Aufrufprogramm muss dann nur *eine* Ausnahme kennen.

```python
import smtplib
import socket
import ssl
from dataclasses import dataclass, field
from email.message import EmailMessage
from pathlib import Path


class MailFehler(Exception):
    """Alle Fehler beim Mailversand, mit einer für Menschen lesbaren Meldung."""


@dataclass
class MailKonfig:
    host: str
    port: int
    absender: str
    modus: str = "starttls"                      # "starttls", "ssl" oder "klartext"
    benutzer: str = ""
    passwort: str = field(default="", repr=False)    # erscheint nicht in Protokollen und Tracebacks
    cafile: str = ""                             # eigene Zertifizierungsstelle (nur für interne Server)
    timeout: float = 10


def baue_nachricht(absender, empfaenger, betreff, text, anhaenge=()):
    """Baut eine Textmail. 'empfaenger' ist eine Liste von Adressen, 'anhaenge' eine Liste von Pfaden."""
    nachricht = EmailMessage()
    nachricht["From"] = absender
    nachricht["To"] = ", ".join(empfaenger)
    nachricht["Subject"] = " ".join(betreff.split())            # Zeilenumbrüche im Betreff vermeiden
    nachricht.set_content(text)
    for pfad in map(Path, anhaenge):
        nachricht.add_attachment(pfad.read_bytes(), maintype="application", subtype="octet-stream",
                                 filename=pfad.name)
    return nachricht


def sende(konfig, nachricht):
    """Sendet die Nachricht. Wirft MailFehler, wenn etwas schiefgeht."""
    kontext = ssl.create_default_context(cafile=konfig.cafile or None)
    try:
        if konfig.modus == "ssl":
            verbindung = smtplib.SMTP_SSL(konfig.host, konfig.port, context=kontext, timeout=konfig.timeout)
        else:
            verbindung = smtplib.SMTP(konfig.host, konfig.port, timeout=konfig.timeout)
        with verbindung as smtp:
            if konfig.modus == "starttls":
                smtp.starttls(context=kontext)
            if konfig.benutzer:
                smtp.login(konfig.benutzer, konfig.passwort)
            smtp.send_message(nachricht)
    except smtplib.SMTPRecipientsRefused as fehler:
        raise MailFehler("Empfänger abgelehnt: " + ", ".join(fehler.recipients)) from fehler
    except smtplib.SMTPAuthenticationError as fehler:
        raise MailFehler("Anmeldung am Mailserver abgelehnt") from fehler
    except ssl.SSLError as fehler:
        raise MailFehler(f"TLS-Fehler ({type(fehler).__name__})") from fehler
    except smtplib.SMTPException as fehler:
        raise MailFehler(f"SMTP-Fehler ({type(fehler).__name__})") from fehler
    except (TimeoutError, socket.timeout):
        raise MailFehler("Zeitüberschreitung beim Mailserver") from None
    except OSError as fehler:
        raise MailFehler(f"Mailserver nicht erreichbar ({type(fehler).__name__})") from fehler
```

Zum Entwurf: In `sende` wird der Kontext mit `ssl.create_default_context(cafile=konfig.cafile or None)` erzeugt, das heißt, ohne eigene Stelle gilt der Standard des Systems, mit eigener Stelle wird *zusätzlich* deren Zertifikat vertraut; auf keinen Fall wird die Prüfung abgeschaltet. Die Verbindung wird in einer Variablen gehalten und mit `with` verwendet, damit sie in jedem Fall geschlossen wird. Die `except`-Zweige stehen von speziell zu allgemein; `from None` bei der Zeitüberschreitung unterdrückt das Anhängen der Ursache, weil sie keine zusätzliche Information trägt. Der Betreff wird mit `" ".join(betreff.split())` bereinigt: `split()` zerlegt an jedem Leerraum (auch Zeilenumbrüchen), `join` setzt mit einzelnen Leerzeichen wieder zusammen. Speichern Sie das Modul als `mailer.py`.

```python
import tempfile
from email import message_from_bytes, policy
from pathlib import Path

import testsmtp
from mailer import MailFehler, MailKonfig, baue_nachricht, sende

server = testsmtp.Server("starttls", benutzer="monitor", passwort="geheim")
konfig = MailKonfig("127.0.0.1", server.port, "monitoring@beispiel.example", benutzer="monitor",
                    passwort="geheim", cafile=str(server.zertifikat), timeout=5)
print("repr der Konfiguration enthält das Passwort nicht:", "geheim" not in repr(konfig))

bericht = Path(tempfile.mkdtemp()) / "bericht.txt"
bericht.write_text("Platte /: 93 %\n", encoding="utf-8")
nachricht = baue_nachricht(konfig.absender, ["admin@beispiel.example", "bereitschaft@beispiel.example"],
                           "Platte voll auf\nweb01", "Die Platte ist zu 93 % belegt.\n", [bericht])
sende(konfig, nachricht)

gelesen = message_from_bytes(server.sammler.nachrichten[0]["inhalt"], policy=policy.default)
print("Betreff        :", gelesen["Subject"])
print("An             :", gelesen["To"])
print("Anhang         :", [teil.get_filename() for teil in gelesen.iter_attachments()])
print("Text           :", gelesen.get_body().get_content().strip())

for beschreibung, aenderung in (("falsches Passwort", {"passwort": "falsch"}),
                                ("Server vertraut niemand", {"cafile": ""}),
                                ("Port ohne Dienst", {"port": 1})):
    schlecht = MailKonfig(**{**konfig.__dict__, **aenderung})
    try:
        sende(schlecht, nachricht)
    except MailFehler as fehler:
        print(f"{beschreibung:24} ->", fehler)
server.stoppe()
```

```text
repr der Konfiguration enthält das Passwort nicht: True
Betreff        : Platte voll auf web01
An             : admin@beispiel.example, bereitschaft@beispiel.example
Anhang         : ['bericht.txt']
Text           : Die Platte ist zu 93 % belegt.
falsches Passwort        -> Anmeldung am Mailserver abgelehnt
Server vertraut niemand  -> TLS-Fehler (SSLCertVerificationError)
Port ohne Dienst         -> Mailserver nicht erreichbar (ConnectionRefusedError)
```

Die Ausgabe zeigt, dass der Zeilenumbruch im Betreff verschwunden ist („Platte voll auf web01“), beide Empfänger in der Kopfzeile stehen, der Anhang mitgeschickt wurde und der Text ankam. Die drei Fehlerfälle liefern jeweils verständliche Meldungen, die ein Betreiber ohne Kenntnis der Bibliothek lesen kann. Die erste Zeile prüft, dass das Passwort nicht in der Textdarstellung der Konfiguration erscheint.

**Zugangsdaten und Geheimnisse.** Mailserver verlangen fast immer Anmeldung, und das Passwort gehört nicht in den Quelltext. Es gelten die Regeln aus Modul 10, 10.4: Das Programm liest das Passwort aus einer Umgebungsvariablen oder aus einer Datei mit eingeschränkten Rechten, und es protokolliert es nie. Viele Anbieter bieten für automatisierte Programme eigene *App-Passwörter* oder Zugangsdaten mit eingeschränkten Rechten an (nur Senden, keine Mailbox); verwenden Sie diese statt des Hauptpassworts eines Menschen. Manche Anbieter lassen die Anmeldung per Passwort für neue Programme nicht mehr zu und verlangen ein anderes Verfahren (OAuth 2.0); das ist Sache des Anbieters und der Einrichtung und wird im Modul nicht behandelt, weil es sich je Anbieter unterscheidet.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, das Zertifikat „zur Not“ nicht zu prüfen, wenn die Verbindung wegen eines unbekannten Zertifikats scheitert: Wer `context.check_hostname = False` und `context.verify_mode = ssl.CERT_NONE` setzt, hat zwar Verschlüsselung, aber keinen Schutz davor, mit dem Falschen zu sprechen. Richtig ist, die eigene Zertifizierungsstelle bekannt zu machen (`cafile`). Der zweite ist, Mails aus Zeichenketten zu bauen, was bei Umlauten und Zeilenumbrüchen schiefgeht. Der dritte ist, den Betreff aus unbereinigten Daten zu bilden. Der vierte, kein Zeitlimit zu setzen: `smtplib.SMTP` wartet dann, bis das Betriebssystem aufgibt, und ein Überwachungsprogramm, das beim Melden hängt, meldet nichts mehr. Das Gegenüber:

```python
import smtplib
import ssl

# FALSCH: Mail von Hand zusammengesetzt, Zertifikat ignoriert, Passwort im Code, kein Timeout
text = "From: monitor@example.org\nTo: admin@example.org\nSubject: " + betreff + "\n\n" + inhalt
kontext = ssl._create_unverified_context()                  # schaltet die Prüfung ab!
with smtplib.SMTP("mail.example.org", 587) as smtp:
    smtp.starttls(context=kontext)
    smtp.login("monitor", "Sommer2026!")
    smtp.sendmail("monitor@example.org", ["admin@example.org"], text)

# RICHTIG: EmailMessage, geprüftes Zertifikat, Passwort aus der Umgebung, Zeitlimit
import os
from email.message import EmailMessage

nachricht = EmailMessage()
nachricht["From"] = "monitor@example.org"
nachricht["To"] = "admin@example.org"
nachricht["Subject"] = " ".join(betreff.split())
nachricht.set_content(inhalt)
with smtplib.SMTP("mail.example.org", 587, timeout=10) as smtp:
    smtp.starttls(context=ssl.create_default_context())
    smtp.login("monitor", os.environ["MAIL_PASSWORT"])
    smtp.send_message(nachricht)
```

Die falsche Variante hat vier Schwächen: Ein Zeilenumbruch in `betreff` schleust Kopfzeilen ein, Umlaute im Betreff sind nicht korrekt kodiert, die ungeprüfte Verbindung lässt sich abhören und umleiten, und das Passwort steht im Quelltext (und damit schnell in der Versionsverwaltung). Die richtige Variante behebt alle vier, wobei `os.environ["MAIL_PASSWORT"]` ausdrücklich einen `KeyError` auslöst, wenn die Variable fehlt; das ist besser als ein stilles Weitermachen ohne Passwort.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage.** Abgerufen wurden die Dokumentation von `smtplib` in den Fassungen für Python 3.4 und 3.7 (die aktuelle Seite war nicht erreichbar; die beschriebenen Klassen, Methoden und Ausnahmen haben sich im Kern nicht geändert, aber Einzelheiten wie veraltete Parameter können abweichen), die Beispielseite des Pakets `email` (Version in spanischer Übersetzung der Python-Dokumentation, mit den Beispielen für `set_content`, `add_attachment` und `send_message`), der Wikipedia-Artikel zu SMTPS und STARTTLS (Zusammenfassung von RFC 8314; der RFC selbst wurde nicht abgerufen) und die Seite zu CWE-93 (Schutz vor Zeilenumbruch-Einschleusung). Alle Beispiele dieser Lerneinheit wurden mit Python 3.13 gegen den Testserver ausgeführt. Dass sich die Fehlerausnahmen so verhalten wie beschrieben (insbesondere das Umwandeln der Zeitüberschreitung bei der Begrüßung in `SMTPServerDisconnected`), wurde empirisch bestätigt. Aussagen über Anbieter (App-Passwörter, gesperrter Port 25, OAuth) sind Fachwissen und können sich ändern; prüfen Sie die Angaben Ihres Mailanbieters.

> **Merke**
>
> Mail besteht aus *Umschlag* (Absender, Empfänger für die Zustellung) und *Nachricht* (Kopfzeilen und Inhalt); `Bcc` steht nur im Umschlag. Nachrichten mit `EmailMessage` bauen, nie aus Zeichenketten. Versand mit `smtplib.SMTP` (STARTTLS, Port 587) oder `SMTP_SSL` (implizites TLS, Port 465), immer mit `ssl.create_default_context()` und mit Zeitlimit. Fehler in einer eigenen Ausnahme bündeln und die Ursachen unterscheiden (Empfänger, Anmeldung, Zertifikat, Netzwerk). Passwörter aus der Umgebung, nie protokollieren.

> **Praxisbeispiel**
>
> Fast jedes Skript, das nachts läuft (Backup, Bericht, Überwachung), verschickt am Ende eine Mail: „Backup erfolgreich, 12 GB“ oder „FEHLER: Platte voll“. In einer Firma läuft das meist über ein internes Mail-Relais, das ohne Anmeldung nur aus dem eigenen Netz Mails annimmt; dann entfallen Anmeldung und oft auch TLS, und die Konfiguration schrumpft auf Server und Absender. Wichtig ist, was in die Mail gehört: ein aussagekräftiger Betreff (der Betreff ist oft alles, was auf dem Telefon sichtbar ist), die betroffene Maschine, die Uhrzeit und das, was zu tun ist. Dazu in 12.4 mehr.

## 12.2 Webhooks

Mail ist zuverlässig, aber träge: Ein Postfach wird nicht ständig angesehen, und ein Alarm zwischen hundert anderen Mails geht leicht unter. Teams arbeiten deshalb zunehmend mit Chat-Werkzeugen, in denen es einen gemeinsamen Kanal für Störungen gibt, den alle im Blick haben. Der Weg dorthin heißt *Webhook*. Ein Webhook ist nichts anderes als eine Internetadresse (URL), an die ein Programm eine HTTP-Anfrage schickt, um dem Empfänger etwas mitzuteilen; der Name beschreibt die Richtung: Normalerweise *fragen* Sie einen Dienst nach Daten (wie in Modul 10 bei der REST-API), bei einem Webhook *meldet* Ihr Programm von sich aus ein Ereignis, es „hängt sich an“ den Dienst. Alles Nötige haben Sie schon gelernt: `requests` (Modul 10), JSON (Modul 7), Zeitlimits, Wiederholungen und Geheimnisse.

**Wie ein Webhook funktioniert.** Das Muster ist überall gleich. Man legt im Chat-Dienst einen *eingehenden Webhook* an und erhält eine URL. Diese URL ist die einzige Zugangsberechtigung: Wer sie kennt, kann Nachrichten in den Kanal schicken. Das Programm sendet an diese URL eine `POST`-Anfrage mit einem JSON-Dokument im Körper; der Dienst antwortet mit einem Statuscode und nimmt die Nachricht an. Nach der Dokumentation von Slack (abgerufen) bekommt man mit dem Webhook „eine eindeutige URL, an die Sie eine JSON-Nutzlast mit dem Nachrichtentext und einigen Optionen senden“; Sie senden die Anfrage mit der Kopfzeile `Content-type: application/json` und einem Körper wie `{"text": "Hello, world."}`, und im Normalfall antwortet der Dienst mit `HTTP 200` und dem Text `ok`. Fehler meldet Slack mit aussagekräftigen Texten (die Seite nennt unter anderem `invalid_payload` für einen fehlerhaften Inhalt, `no_text` für ein fehlendes Textfeld, `no_service` für einen abgeschalteten oder ungültigen Webhook und `channel_is_archived` für einen archivierten Kanal) mit den Statuscodes 400, 403 oder 404.

Für Microsoft Teams hat sich die Lage in den letzten Jahren geändert. Früher gab es dort *Office-365-Connectors* mit eingehenden Webhooks. Laut der abgerufenen Ankündigung im Microsoft-365-Nachrichtenzentrum (Meldung MC808160, in der abgerufenen Fassung) wurde das Anlegen neuer Connectors ab dem 15. August 2024 eingestellt, bestehende Webhook-URLs sollten bis zum 31. Januar 2025 aktualisiert werden, und der gesamte Connector-Dienst sollte Ende 2025 abgeschaltet werden; empfohlen wird der Wechsel zur *Workflows*-App in Teams. Die aktuelle Microsoft-Seite zu eingehenden Webhooks (abgerufen) beschreibt den heutigen Weg: Man legt einen Workflow aus der Vorlage „Send webhook alerts to a channel“ oder mit dem Auslöser „When a Teams webhook request is received“ an, bekommt eine URL, und eine JSON-Nutzlast, die an diese URL gesendet wird, erscheint als Nachricht im gewählten Kanal. Die Seite zeigt eine einfache Nutzlast mit einem Feld `text` und eine *Adaptive Card* im Feld `attachments`, und sie nennt Grenzen: eine Nachrichtengröße von höchstens 28 KB und eine Drosselung bei mehr als vier Anfragen pro Sekunde, wofür sie Wiederholungen mit wachsender Wartezeit empfiehlt. Da sich diese Dienste ändern, gilt für den Betrieb: Die Dokumentation des jeweiligen Dienstes ist die maßgebliche Quelle, nicht dieses Skript.

**Die erste Nachricht.** Das folgende Beispiel verwendet unseren Hilfsmodul `testhook`, einen kleinen Empfänger auf `127.0.0.1`, der sich wie ein Chat-Dienst verhält: Er nimmt `POST`-Anfragen mit JSON an, merkt sie sich, antwortet mit `ok` und lehnt Anfragen mit ungültigem Inhalt mit `400` ab. Er kann zum Üben von Fehlern bestimmte Antworten der Reihe nach liefern (Parameter `antworten`), eine Pause vor der Antwort einlegen (`pause`) und Signaturen verlangen (`geheimnis`, dazu später mehr). Auch hier gilt: Es kommt auf die Beispiele an, nicht auf den Aufbau des Hilfsmoduls.

```python
"""testhook - lokaler Webhook-Empfaenger fuer die Beispiele von Modul 12 (nur auf 127.0.0.1)."""
import hashlib
import hmac
import json
import threading
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer


class Empfaenger:
    """Nimmt POST-Anfragen an und merkt sie sich.

    antworten:  Liste von (Statuscode, Text) oder (Statuscode, Text, Kopfzeilen), die der Reihe nach
                statt der normalen Antwort 200/ok verwendet werden (zum Ueben von Fehlern).
    geheimnis:  Wenn gesetzt, muss die Kopfzeile X-Signatur den HMAC-SHA256 des Koerpers enthalten.
    pause:      Sekunden, die der Server vor jeder Antwort wartet (zum Ueben von Zeitlimits).
    """

    def __init__(self, antworten=None, geheimnis=None, pause=0):
        self.empfangen = []
        self.antworten = list(antworten or [])
        self.geheimnis = geheimnis.encode() if geheimnis else None
        self.pause = pause
        self.anfragen = 0
        empfaenger = self

        class Handler(BaseHTTPRequestHandler):
            def handle(self):
                try:
                    super().handle()
                except (BrokenPipeError, ConnectionResetError):
                    pass

            def log_message(self, *args):
                pass

            def do_POST(self):
                empfaenger.anfragen += 1
                roh = self.rfile.read(int(self.headers.get("Content-Length", 0)))
                if empfaenger.pause:
                    time.sleep(empfaenger.pause)
                if empfaenger.antworten:
                    eintrag = empfaenger.antworten.pop(0)
                    return self.antwort(eintrag[0], eintrag[1], eintrag[2] if len(eintrag) > 2 else {})
                if empfaenger.geheimnis is not None:
                    erwartet = "sha256=" + hmac.new(empfaenger.geheimnis, roh, hashlib.sha256).hexdigest()
                    if not hmac.compare_digest(erwartet, self.headers.get("X-Signatur", "")):
                        return self.antwort(401, "invalid_signature")
                try:
                    nutzlast = json.loads(roh)
                except ValueError:
                    return self.antwort(400, "invalid_payload")
                empfaenger.empfangen.append({"pfad": self.path, "nutzlast": nutzlast,
                                             "typ": self.headers.get("Content-Type")})
                self.antwort(200, "ok")

            def antwort(self, status, text, kopfzeilen=None):
                daten = text.encode()
                self.send_response(status)
                for name, wert in (kopfzeilen or {}).items():
                    self.send_header(name, wert)
                self.send_header("Content-Type", "text/plain")
                self.send_header("Content-Length", str(len(daten)))
                self.end_headers()
                self.wfile.write(daten)

        self._server = ThreadingHTTPServer(("127.0.0.1", 0), Handler)
        self._server.daemon_threads = True
        self.port = self._server.server_address[1]
        self.url = f"http://127.0.0.1:{self.port}/hooks/T000/B000/geheimertoken"
        threading.Thread(target=self._server.serve_forever, daemon=True).start()

    def stoppe(self):
        self._server.shutdown()
        self._server.server_close()
```

Mit `requests.post(url, json=...)` wird das Python-Wörterbuch in JSON umgewandelt und die Kopfzeile `Content-Type: application/json` automatisch gesetzt. Der Unterschied zu `data=` ist wichtig und ein häufiger Fehler: Mit `data=` und einem Wörterbuch sendet `requests` ein *Formular* (`application/x-www-form-urlencoded`), das ein JSON-Webhook als ungültigen Inhalt zurückweist. Das Beispiel zeigt beides.

```python
import requests

import testhook

empfaenger = testhook.Empfaenger()

antwort = requests.post(empfaenger.url, json={"text": "Platte voll auf web01"}, timeout=5)
print("Status und Antworttext:", antwort.status_code, antwort.text)

eintrag = empfaenger.empfangen[0]
print("Server sah Nutzlast   :", eintrag["nutzlast"])
print("Inhaltstyp            :", eintrag["typ"])

antwort = requests.post(empfaenger.url, data={"text": "als Formular"}, timeout=5)     # data= ist KEIN JSON
print("Mit data= statt json= :", antwort.status_code, antwort.text)
empfaenger.stoppe()
```

```text
Status und Antworttext: 200 ok
Server sah Nutzlast   : {'text': 'Platte voll auf web01'}
Inhaltstyp            : application/json
Mit data= statt json= : 400 invalid_payload
```

Die erste Anfrage kommt an: Der Server antwortet mit `200` und `ok`, hat das Wörterbuch als JSON-Objekt gesehen, und der Inhaltstyp war `application/json`. Die zweite Anfrage mit `data=` wird mit `400 invalid_payload` abgelehnt, weil der Inhalt kein JSON war. Ein Tipp für die Fehlersuche: Antworttext und Statuscode des Dienstes sind die besten Hinweise; wer nur `ok` oder `failed` protokolliert, wirft die Erklärung des Dienstes weg.

**Nutzlasten für Slack und Teams.** Je nach Dienst sieht die JSON-Nutzlast anders aus. Bei Slack genügt im einfachsten Fall ein Textfeld; die Dokumentation erwähnt zusätzliche Blöcke und Formatierungen im selben Dokument. Bei Teams ist für Workflows eine *Adaptive Card* üblich: ein Kartenformat, das aus Elementen wie `TextBlock` besteht. Nach dem Beispiel der abgerufenen Microsoft-Seite hat die Nachricht den Typ `message` und ein Feld `attachments` mit einer Karte, deren `contentType` `application/vnd.microsoft.card.adaptive` lautet. Der Aufbau der Karte (Schema, Version, Elemente) stammt aus dem Standard der Adaptive Cards; welche Versionen ein Workflow akzeptiert, hängt von der Teams-Version ab, und die Microsoft-Seite dokumentiert das Schema nicht vollständig. Wir bauen deshalb zwei kleine Funktionen für unsere Bibliothek, die man leicht ändern kann, wenn der Dienst etwas anderes verlangt. Dazu kommen die Hilfen, die jede Webhook-Bibliothek braucht: das *Maskieren* der URL für Protokolle (die URL ist ein Geheimnis, siehe unten) und die *Signatur* von Nachrichten, die wir nach dem Senden besprechen. Die Funktion `sende_webhook` ist das Herz der Bibliothek, und sie verdient eine genaue Erklärung.

```python
import hashlib
import hmac
import time
from urllib.parse import urlsplit

import requests


class WebhookFehler(Exception):
    """Der Webhook konnte nicht zugestellt werden (mit lesbarer Meldung)."""


def slack_nachricht(text):
    """Einfachste Nutzlast für einen Slack-Webhook: ein Feld 'text'."""
    return {"text": text}


def teams_nachricht(titel, text):
    """Nutzlast für einen Teams-Workflow-Webhook: eine Adaptive Card in 'attachments'."""
    return {
        "type": "message",
        "attachments": [{
            "contentType": "application/vnd.microsoft.card.adaptive",
            "content": {
                "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
                "type": "AdaptiveCard",
                "version": "1.4",
                "body": [
                    {"type": "TextBlock", "text": titel, "weight": "Bolder", "size": "Medium"},
                    {"type": "TextBlock", "text": text, "wrap": True},
                ],
            },
        }],
    }


def maskiere(url):
    """Zeigt nur Rechner und den ersten Pfadteil; der Rest der URL ist das Geheimnis."""
    teile = urlsplit(url)
    erster = teile.path.strip("/").split("/")[0]
    return f"{teile.scheme}://{teile.hostname}/{erster}/***"


def signiere(geheimnis, koerper):
    """Berechnet die Signatur, die der Empfänger prüfen kann (HMAC-SHA256 über den Körper)."""
    return "sha256=" + hmac.new(geheimnis.encode(), koerper, hashlib.sha256).hexdigest()


def pruefe_signatur(geheimnis, koerper, kopfzeile):
    """Prüft die Signatur im Zeitkonstanz-Vergleich (verhindert Timing-Angriffe)."""
    return hmac.compare_digest(signiere(geheimnis, koerper), kopfzeile or "")


def sende_webhook(url, nutzlast, timeout=(3.05, 10), versuche=3, pause=1.0, schlafen=time.sleep):
    """Sendet die Nutzlast als JSON. Wiederholt bei 429, 5xx und Verbindungsfehlern, nicht bei 4xx."""
    letzter = "unbekannt"
    for versuch in range(1, versuche + 1):
        try:
            antwort = requests.post(url, json=nutzlast, timeout=timeout)
        except (requests.ConnectionError, requests.Timeout) as fehler:
            letzter = type(fehler).__name__
            wartezeit = pause * 2 ** (versuch - 1)
        else:
            if antwort.ok:
                return antwort.text
            letzter = f"HTTP {antwort.status_code}"
            if antwort.status_code != 429 and antwort.status_code < 500:
                raise WebhookFehler(f"{maskiere(url)} lehnt ab: {letzter} {antwort.text[:40]!r}")
            wartezeit = pause * 2 ** (versuch - 1)
            if antwort.status_code == 429 and antwort.headers.get("Retry-After", "").isdigit():
                wartezeit = max(wartezeit, int(antwort.headers["Retry-After"]))
        if versuch < versuche:
            schlafen(wartezeit)
    raise WebhookFehler(f"{maskiere(url)} nach {versuche} Versuchen nicht erreichbar ({letzter})")
```

`sende_webhook` wiederholt nach einem klaren Regelwerk, das auf den Erfahrungen aus Modul 10 (10.3) beruht. Verbindungsfehler und Zeitüberschreitungen werden wiederholt, ebenso die Statuscodes `429` (zu viele Anfragen) und alles ab `500` (Fehler beim Dienst); bei `429` beachtet die Funktion die Kopfzeile `Retry-After`, falls der Dienst eine Wartezeit nennt, und wartet mindestens so lange. Alle anderen Fehler der Klasse `4xx` (falsche URL, ungültiger Inhalt, fehlende Berechtigung) werden *nicht* wiederholt, denn der zweite Versuch scheitert genauso, und die Funktion meldet sofort den Grund. Die Wartezeit verdoppelt sich bei jedem Versuch (*exponentielles Zurückweichen*, englisch *exponential backoff*), damit ein überlasteter Dienst nicht von allen Wiederholungen gleichzeitig getroffen wird. Der Parameter `schlafen` nimmt die Funktion, die wartet; im Normalfall `time.sleep`, im Test eine Funktion, die sich die Pause nur merkt, damit die Tests nicht Minuten brauchen. Das ist eine Technik, die man *Einschleusen von Abhängigkeiten* nennt (*dependency injection*) und die in Modul 14 beim Testen wieder auftaucht. Das Ergebnis bei Erfolg ist der Antworttext, bei endgültigem Scheitern wird `WebhookFehler` mit der *maskierten* URL in der Meldung ausgelöst.

Bevor wir die Wiederholungen ansehen, prüfen wir die beiden Nutzlasten und das Maskieren. Die Teams-Karte wird mit `json.dumps` nicht abgedruckt, damit die Ausgabe kurz bleibt; wir prüfen die wichtigsten Felder.

```python
import json

from hooks import maskiere, slack_nachricht, teams_nachricht

print(json.dumps(slack_nachricht("Platte voll auf web01"), ensure_ascii=False))

karte = teams_nachricht("Alarm: web01", "Platte zu 93 % belegt")
anhang = karte["attachments"][0]
print("Teams, Typ der Nachricht :", karte["type"])
print("Teams, Inhaltstyp        :", anhang["contentType"])
print("Teams, Textblöcke        :", [b["text"] for b in anhang["content"]["body"]])

url = "https://hooks.slack.example/services/T000/B000/xyzGEHEIMxyz"
print("Maskierte URL fürs Log   :", maskiere(url))
print("Geheimer Teil sichtbar   :", "GEHEIM" in maskiere(url))
```

```text
{"text": "Platte voll auf web01"}
Teams, Typ der Nachricht : message
Teams, Inhaltstyp        : application/vnd.microsoft.card.adaptive
Teams, Textblöcke        : ['Alarm: web01', 'Platte zu 93 % belegt']
Maskierte URL fürs Log   : https://hooks.slack.example/services/***
Geheimer Teil sichtbar   : False
```

Die Slack-Nutzlast ist ein einzelnes Textfeld. Von der Teams-Nachricht prüfen wir Typ, Inhaltstyp der Karte und die beiden Textblöcke. Das Maskieren zeigt nur Schema, Rechner und den ersten Pfadteil; der Rest, der bei Slack die geheimen Kennungen enthält, bleibt verborgen. Das ist wichtig, weil URLs gern in Fehlermeldungen und Protokollen landen.

Nun das Verhalten bei Fehlern, das wir mit vier Fällen prüfen. Im ersten liefert der Testempfänger erst einen Fehler 503, dann 429 mit dem Wunsch „7 Sekunden warten“, danach Erfolg. Im zweiten ist die URL ungültig (404), im dritten antwortet der Dienst dauerhaft mit 500, und im vierten ist er langsamer als unser Zeitlimit. Statt wirklich zu warten, merkt sich das Programm die verlangten Pausen.

```python
import testhook
from hooks import WebhookFehler, sende_webhook

wartezeiten = []                                          # statt wirklich zu schlafen, merken wir uns die Pausen


def merke(sekunden):
    wartezeiten.append(sekunden)


# 1. Erst Überlastung (503), dann Ratenbegrenzung mit Wunsch nach 7 Sekunden Pause, dann Erfolg
server = testhook.Empfaenger(antworten=[(503, "busy"), (429, "slow down", {"Retry-After": "7"})])
ergebnis = sende_webhook(server.url, {"text": "hallo"}, versuche=3, pause=1, schlafen=merke)
print("1. Ergebnis:", ergebnis, "| Anfragen:", server.anfragen, "| Pausen:", wartezeiten)
server.stoppe()

# 2. Client-Fehler (404): sofort aufgeben, nicht wiederholen
wartezeiten.clear()
server = testhook.Empfaenger(antworten=[(404, "no_service")])
try:
    sende_webhook(server.url, {"text": "hallo"}, schlafen=merke)
except WebhookFehler as fehler:
    print("2. Meldung :", fehler, "| Anfragen:", server.anfragen, "| Pausen:", wartezeiten)
server.stoppe()

# 3. Dauerhaft 500: nach allen Versuchen aufgeben, Pausen wachsen verdoppelnd
wartezeiten.clear()
server = testhook.Empfaenger(antworten=[(500, "kaputt")] * 5)
try:
    sende_webhook(server.url, {"text": "hallo"}, versuche=4, pause=2, schlafen=merke)
except WebhookFehler as fehler:
    print("3. Meldung :", fehler, "| Anfragen:", server.anfragen, "| Pausen:", wartezeiten)
server.stoppe()

# 4. Server zu langsam: Zeitlimit von 0,3 s bei einer Antwortzeit von 1 s
wartezeiten.clear()
server = testhook.Empfaenger(pause=1)
try:
    sende_webhook(server.url, {"text": "hallo"}, timeout=(1, 0.3), versuche=2, pause=0.5, schlafen=merke)
except WebhookFehler as fehler:
    print("4. Meldung :", str(fehler).split(" nach ")[1], "| Pausen:", wartezeiten)
server.stoppe()
```

```text
1. Ergebnis: ok | Anfragen: 3 | Pausen: [1, 7]
2. Meldung : http://127.0.0.1/hooks/*** lehnt ab: HTTP 404 'no_service' | Anfragen: 1 | Pausen: []
3. Meldung : http://127.0.0.1/hooks/*** nach 4 Versuchen nicht erreichbar (HTTP 500) | Anfragen: 4 | Pausen: [2, 4, 8]
4. Meldung : 2 Versuchen nicht erreichbar (ReadTimeout) | Pausen: [0.5]
```

Im ersten Fall sind drei Anfragen nötig und die Pausen betragen 1 Sekunde (nach 503, der erste Schritt der Verdopplung) und 7 Sekunden (nach 429, weil der Dienst mit `Retry-After` mehr verlangt als die eigene Regel). Im zweiten Fall gibt es genau eine Anfrage und keine Pause; die Meldung nennt den Fehler des Dienstes (`no_service`) und die maskierte URL, nicht die vollständige. Im dritten Fall wird nach vier Versuchen aufgegeben; die Pausen 2, 4 und 8 Sekunden zeigen die Verdopplung. Im vierten Fall scheitert jeder Versuch an der Zeitüberschreitung (`ReadTimeout`); `timeout=(1, 0.3)` bedeutet 1 Sekunde für den Verbindungsaufbau und 0,3 Sekunden Warten auf die Antwort (siehe Modul 10, 10.3). Zwischen den beiden Versuchen liegt eine Pause von 0,5 Sekunden.

**Wer darf Nachrichten senden? Geheimnisse und Signaturen.** Bei einem eingehenden Webhook ist die URL das Geheimnis. Die Slack-Dokumentation sagt es unmissverständlich: Die URL enthält ein Geheimnis, man solle sie nicht online teilen, auch nicht in öffentlichen Versionsverwaltungen, und Slack suche aktiv nach öffentlich gewordenen Geheimnissen und widerrufe sie. Daraus folgen dieselben Regeln wie für Passwörter (Modul 10, 10.4): Die URL gehört nicht in den Quelltext, sondern in die Umgebung oder eine geschützte Datei; sie wird nicht protokolliert (deshalb `maskiere`); und bei Verdacht auf Offenlegung wird der Webhook gelöscht und neu erzeugt.

Dieselbe Frage stellt sich in der Gegenrichtung, wenn *Ihr* Programm einen Webhook *empfängt*, etwa von einem Monitoring-System, das Ihnen Ereignisse sendet. Dann kann jeder, der Ihre Adresse kennt, Anfragen senden, und das Programm muss sicherstellen, dass die Nachricht wirklich vom erwarteten Absender stammt und unterwegs nicht verändert wurde. Verbreitet ist dafür die *Signatur*: Absender und Empfänger teilen ein geheimes Passwort, der Absender berechnet über den Körper der Nachricht einen *HMAC* (einen mit dem Geheimnis geschlüsselten Prüfwert, hier mit SHA-256) und schickt ihn in einer Kopfzeile mit; der Empfänger berechnet den Wert selbst und vergleicht. Stimmt er überein, kennt der Absender das Geheimnis, und der Inhalt wurde nicht verändert. Zwei Einzelheiten sind wichtig: Der Vergleich muss mit `hmac.compare_digest` erfolgen, einem Vergleich, dessen Dauer nicht davon abhängt, wie viele Zeichen übereinstimmen (ein normaler Vergleich mit `==` bricht beim ersten Unterschied ab, und Angreifer können aus der Zeit auf den richtigen Wert schließen), und die Signatur muss über die *rohen Bytes* des Körpers berechnet werden, nicht über ein bereits eingelesenes und wieder ausgegebenes JSON, weil dabei Leerzeichen und Reihenfolge der Schlüssel verloren gehen könnten. Die Namen der Kopfzeile und des Verfahrens legt jeder Dienst selbst fest; unsere `X-Signatur` ist ein Beispiel, kein Standard.

```python
import json

import requests

import testhook
from hooks import pruefe_signatur, signiere

GEHEIMNIS = "ein-langes-zufaelliges-geheimnis"
koerper = json.dumps({"ereignis": "ALARM", "host": "web01"}).encode()

signatur = signiere(GEHEIMNIS, koerper)
print("Signatur beginnt mit sha256=:", signatur.startswith("sha256="), "| Länge:", len(signatur))
print("Echte Nachricht prüft       :", pruefe_signatur(GEHEIMNIS, koerper, signatur))
print("Veränderter Körper          :", pruefe_signatur(GEHEIMNIS, koerper.replace(b"web01", b"web02"), signatur))
print("Falsches Geheimnis          :", pruefe_signatur("anderes", koerper, signatur))
print("Signatur fehlt              :", pruefe_signatur(GEHEIMNIS, koerper, None))

# Gegen einen Empfänger, der Signaturen verlangt
empfaenger = testhook.Empfaenger(geheimnis=GEHEIMNIS)
kopf = {"Content-Type": "application/json"}
ohne = requests.post(empfaenger.url, data=koerper, headers=kopf, timeout=5)
mit = requests.post(empfaenger.url, data=koerper, headers={**kopf, "X-Signatur": signatur}, timeout=5)
print("Ohne Signatur gesendet      :", ohne.status_code, ohne.text)
print("Mit Signatur gesendet       :", mit.status_code, mit.text)
empfaenger.stoppe()
```

```text
Signatur beginnt mit sha256=: True | Länge: 71
Echte Nachricht prüft       : True
Veränderter Körper          : False
Falsches Geheimnis          : False
Signatur fehlt              : False
Ohne Signatur gesendet      : 401 invalid_signature
Mit Signatur gesendet       : 200 ok
```

Die Signatur beginnt mit dem Kennwort des Verfahrens und hat 71 Zeichen (`sha256=` plus 64 Hexziffern). Die echte Nachricht wird anerkannt; ein veränderter Körper, ein falsches Geheimnis und eine fehlende Signatur werden abgelehnt. Gegen den Empfänger, der Signaturen verlangt, scheitert die ungesignierte Anfrage mit 401, die signierte kommt durch. Ein weiterer Schutz, den die meisten Dienste zusätzlich einsetzen, ist ein Zeitstempel im Körper oder in einer Kopfzeile, den der Empfänger auf Aktualität prüft, damit eine abgefangene Nachricht nicht Tage später erneut eingespielt werden kann (*Replay-Angriff*). Er ist im Beispiel nicht enthalten.

**Grenzen der Kanäle.** Webhooks sind einfach, haben aber Grenzen, die man kennen muss. Chat-Dienste begrenzen Größe und Häufigkeit (Teams laut der abgerufenen Seite 28 KB je Nachricht und höchstens vier Anfragen pro Sekunde); wer bei einem Großausfall hundert Einzelmeldungen in einer Sekunde sendet, wird gedrosselt oder blockiert. Deshalb fasst man Meldungen zusammen (das üben wir in Aufgabe 6). Webhooks sind *einseitig*: Eine Zustellung wird nur bestätigt, dass der Dienst sie *angenommen* hat, nicht, dass ein Mensch sie gelesen hat. Und der Dienst kann ausfallen oder die Adresse ändern; ein Alarmweg, auf den man sich verlässt, braucht einen zweiten Weg (in 12.4 versenden wir auf allen Kanälen, und ein ausgefallener Kanal verhindert die anderen nicht).

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Senden ist, die Antwort nicht zu prüfen: `requests.post(url, json=daten)` ohne Blick auf den Statuscode meldet „gesendet“, auch wenn der Dienst 404 geantwortet hat. Der zweite ist `data=` statt `json=`. Der dritte, die geheime URL im Quelltext oder im Protokoll. Der vierte, kein Zeitlimit zu setzen. Der fünfte, 4xx-Fehler endlos zu wiederholen. Das Gegenüber:

```python
import requests

URL = "https://hooks.slack.com/services/T000/B000/geheimer-teil"      # im Quelltext!

# FALSCH: keine Antwortprüfung, kein Timeout, data= statt json=, URL im Protokoll
def melde(text):
    requests.post(URL, data={"text": text})
    print("Gesendet an", URL)

# RICHTIG: Antwort prüfen, Zeitlimit, json=, URL maskiert, Fehler laut melden
import os
from hooks import maskiere, sende_webhook, slack_nachricht

def melde(text):
    url = os.environ["ALARM_WEBHOOK"]                       # aus der Umgebung
    sende_webhook(url, slack_nachricht(text), timeout=(3.05, 10))
    print("Gesendet an", maskiere(url))
```

Die falsche Variante meldet auch dann „Gesendet“, wenn die Nachricht nie angekommen ist, schickt ein Formular statt JSON, kann beliebig lange hängen und schreibt die geheime URL in jedes Protokoll. Die richtige löst eine Ausnahme aus, wenn die Zustellung scheitert, wiederholt bei vorübergehenden Fehlern, setzt Zeitlimits und zeigt in der Ausgabe nur die maskierte URL.

> **Ergänzung aus der Nachrecherche**
>
> **Quellenlage und Grenzen.** Abgerufen wurden die Dokumentation von Slack zu eingehenden Webhooks (URL und JSON-Nutzlast, Antworttext `ok`, Fehlertexte, Hinweis zur Geheimhaltung; *nicht* auf der Seite stand die oft genannte Drosselung auf eine Nachricht pro Sekunde, sie wird hier deshalb nicht behauptet), die Seite von Microsoft zu eingehenden Webhooks in Teams (Workflows, Nutzlast mit `text` und mit Adaptive Card, 28-KB-Grenze, Drosselung bei mehr als vier Anfragen pro Sekunde) und die Ankündigung zur Einstellung der Office-365-Connectors (Meldung MC808160). Das genaue Schema, das ein Teams-Workflow für die Karte erwartet, ist dort nicht vollständig beschrieben; die gezeigte Nutzlast folgt dem Beispiel der Microsoft-Seite und dem Standard der Adaptive Cards und wurde *nicht* gegen einen echten Teams-Workflow getestet, da dafür ein Microsoft-365-Zugang nötig wäre. Die Beispiele laufen gegen einen lokalen Testempfänger. Die Beschreibung der Signatur mit HMAC und `hmac.compare_digest` ist Fachwissen und wurde mit den Beispielen geprüft; die konkreten Kopfzeilen echter Dienste unterscheiden sich.

> **Merke**
>
> Ein Webhook ist eine URL, die man mit `POST` und JSON anspricht (`json=`, nicht `data=`). Die URL ist ein Geheimnis: nicht im Quelltext, nicht im Protokoll (maskieren), bei Offenlegung erneuern. Beim Senden: Antwort prüfen, Zeitlimits, bei 429 und 5xx mit wachsender Pause wiederholen (`Retry-After` beachten), 4xx nicht wiederholen. Beim Empfangen: Signatur (HMAC) über die rohen Bytes prüfen, mit `hmac.compare_digest` vergleichen. Dienste haben Größen- und Häufigkeitsgrenzen; Meldungen zusammenfassen.

> **Praxisbeispiel**
>
> Ein typischer Betrieb: Ein Kanal „#alarme“ im Chat, in den Überwachung, Backup-Skripte und Bereitstellung melden. Wer Bereitschaft hat, sieht dort, was passiert ist, und kann Rückfragen stellen; die Mail dient als Archiv und als Ausweichweg. Der Webhook wird in der Konfiguration des Skripts hinterlegt, die URL kommt aus einer geschützten Datei oder dem Geheimnisspeicher des Servers. Und weil Dienste ihre Webhook-Verfahren ändern, wie bei Teams geschehen, gehört ein regelmäßiger Testalarm zum Betrieb: Ein Alarmweg, den niemand testet, funktioniert im Ernstfall oft nicht.

## 12.3 Health-Check, Teil 1: Prüfungen schreiben

Wir haben jetzt zwei Wege, Menschen zu erreichen, und in Modul 11 gelernt, wie man Messwerte holt. Es fehlt das Bindeglied: ein Programm, das *prüft*, ob alles in Ordnung ist. Ein solcher *Health-Check* (zu Deutsch etwa „Gesundheitsprüfung“) gehört zu den häufigsten Werkzeugen in der Verwaltung von Servern und Diensten. In dieser Lerneinheit bauen wir die Prüfungen selbst: eine Sammlung kleiner Funktionen, die je eine Frage beantworten („antwortet die Website?“, „ist die Platte voll?“, „läuft der Prozess?“) und das Ergebnis in einer einheitlichen Form liefern. Die Verbindung mit den Meldungen folgt in 12.4, das fertige Programm in 12.5.

**Was eine gute Prüfung ausmacht.** Bevor wir Code schreiben, lohnt ein Blick auf den Entwurf, denn hier entscheidet sich, ob die Überwachung später nützt oder nervt. Eine Prüfung soll erstens etwas messen, das *für den Benutzer des Dienstes zählt*. Dass ein Prozess läuft, heißt noch nicht, dass die Website funktioniert; deshalb prüft man besser, ob eine Seite die erwartete Antwort liefert (*Prüfung von außen*), als nur, ob ein Programm existiert (*Prüfung von innen*). Beides hat seinen Platz, aber die Prüfung von außen ist die ehrlichere. Zweitens soll eine Prüfung *schnell und mit Zeitlimit* antworten: Eine Prüfung, die selbst hängen bleibt, ist ein Ausfall der Überwachung. Drittens soll sie *unterscheiden*, was los ist: Sie liefert nicht nur „ja“ oder „nein“, sondern eine Stufe und eine Meldung, die dem Leser sagt, was zu tun ist. Viertens soll sie *nie abstürzen*: Wenn die Prüfung selbst scheitert (ein Programmfehler, ein unerwartetes Format), ist das ein eigener Zustand („unbekannt“) und kein Grund, die ganze Überwachung abzubrechen.

Die Stufen sind eine Konvention, die auf die Überwachungssoftware Nagios zurückgeht und von vielen Systemen übernommen wurde: `OK` (alles in Ordnung), `WARNUNG` (auffällig, aber noch nicht dringend), `KRITISCH` (Eingriff nötig) und `UNBEKANNT` (die Prüfung konnte kein Urteil fällen). Die zugehörigen Exit-Codes 0, 1, 2 und 3 stammen ebenfalls aus dieser Konvention; wir verwenden sie in 12.5 für unser Programm (das ist Fachwissen und wurde nicht in der Dokumentation von Nagios nachgelesen). Der Zustand `UNBEKANNT` ist wichtig und wird häufig vergessen: Wenn eine Prüfung wegen eines eigenen Fehlers kein Ergebnis hat, darf man weder `OK` (das wäre eine falsche Entwarnung) noch `KRITISCH` (das wäre ein möglicherweise falscher Alarm) melden.

**Ergebnis und Bewertung.** Jede Prüfung liefert ein Objekt derselben Form: einen Namen, eine Stufe, einen Text für Menschen und optional den gemessenen Zahlenwert. Dafür eignet sich eine `dataclass` (Modul 3 und 5). Zahlenwerte werden mit einer gemeinsamen Funktion `bewerte` in Stufen übersetzt: ab der Warnschwelle `WARNUNG`, ab der kritischen Schwelle `KRITISCH`. Die Schwellen sind Parameter mit Vorgaben, damit man sie je Prüfung ändern kann, ohne den Code anzufassen (in 12.5 kommen sie aus der Konfigurationsdatei). Wichtig ist die Grenze: Der Wert *genau* auf der Schwelle gehört schon zur höheren Stufe (`>=`), und die Beispiele prüfen die Grenzfälle.

Damit wir Prüfungen gegen ein Ziel testen können, das wir nach Belieben ein- und ausschalten und verändern können, benutzen wir das Hilfsmodul `testweb`: ein kleiner Webserver auf `127.0.0.1`, dessen Statuscode und Text wir während des Programmablaufs ändern können (`web.status = 500`) und der unter dem Pfad `/langsam` zwei Sekunden wartet.

```python
"""testweb - kleiner lokaler Webserver, dessen Zustand wir per Programm ändern können (Modul 12)."""
import threading
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer


class Web:
    """Antwortet auf GET mit dem eingestellten Statuscode; Pfad /langsam wartet zwei Sekunden."""

    def __init__(self, status=200, text="Willkommen"):
        self.status = status
        self.text = text
        web = self

        class Handler(BaseHTTPRequestHandler):
            def handle(self):
                try:
                    super().handle()
                except (BrokenPipeError, ConnectionResetError):
                    pass

            def log_message(self, *args):
                pass

            def do_GET(self):
                if self.path == "/langsam":
                    time.sleep(2)
                daten = web.text.encode()
                self.send_response(web.status)
                self.send_header("Content-Type", "text/plain; charset=utf-8")
                self.send_header("Content-Length", str(len(daten)))
                self.end_headers()
                self.wfile.write(daten)

        self._server = ThreadingHTTPServer(("127.0.0.1", 0), Handler)
        self._server.daemon_threads = True
        self.port = self._server.server_address[1]
        self.url = f"http://127.0.0.1:{self.port}"
        threading.Thread(target=self._server.serve_forever, daemon=True).start()

    def stoppe(self):
        self._server.shutdown()
        self._server.server_close()
```

Jetzt die Prüfungen. Die Datei `checks.py` enthält sechs Prüfarten, eine Tabelle `ARTEN`, die den Namen jeder Art der Funktion zuordnet, und die Funktion `fuehre_aus`, die eine Prüfung aus einem Wörterbuch heraus ausführt. Dieses Wörterbuch entspricht später einem Eintrag der Konfigurationsdatei, zum Beispiel `{"art": "http", "name": "web", "url": "http://web01/", "timeout": 2}`; `art` und `name` werden herausgenommen, alle übrigen Einträge werden als Argumente an die Prüffunktion weitergereicht (`**definition`). So lassen sich neue Prüfarten ergänzen, ohne dass sich das Programm ändert: eine Funktion schreiben, in `ARTEN` eintragen, fertig. Diese Technik nennt man eine *Registratur* (*registry*).

```python
import os
import socket
import time
from dataclasses import dataclass

import psutil
import requests

OK, WARNUNG, KRITISCH, UNBEKANNT = "OK", "WARNUNG", "KRITISCH", "UNBEKANNT"
SCHWERE = {OK: 0, WARNUNG: 1, KRITISCH: 2, UNBEKANNT: 3}


@dataclass
class Ergebnis:
    name: str
    status: str
    meldung: str
    wert: float | None = None


def bewerte(wert, warnung, kritisch):
    """Ordnet einen Messwert einer Stufe zu: ab 'warnung' WARNUNG, ab 'kritisch' KRITISCH."""
    if wert >= kritisch:
        return KRITISCH
    if wert >= warnung:
        return WARNUNG
    return OK


def check_http(name, url, erwartet=200, enthaelt=None, timeout=5, langsam=2.0):
    start = time.monotonic()
    try:
        antwort = requests.get(url, timeout=timeout)
    except requests.Timeout:
        return Ergebnis(name, KRITISCH, f"{url}: keine Antwort in {timeout} s")
    except requests.ConnectionError:
        return Ergebnis(name, KRITISCH, f"{url}: nicht erreichbar")
    dauer = time.monotonic() - start
    if antwort.status_code != erwartet:
        return Ergebnis(name, KRITISCH, f"{url}: Status {antwort.status_code} statt {erwartet}", dauer)
    if enthaelt and enthaelt not in antwort.text:
        return Ergebnis(name, KRITISCH, f"{url}: erwarteter Text fehlt", dauer)
    if dauer > langsam:
        return Ergebnis(name, WARNUNG, f"{url}: antwortet langsam ({dauer:.1f} s)", dauer)
    return Ergebnis(name, OK, f"{url}: Status {antwort.status_code}", dauer)


def check_tcp(name, host, port, timeout=3):
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return Ergebnis(name, OK, f"{host}:{port} nimmt Verbindungen an")
    except OSError as fehler:
        return Ergebnis(name, KRITISCH, f"{host}:{port}: {type(fehler).__name__}")


def check_platte(name, pfad="/", warnung=80, kritisch=90):
    prozent = psutil.disk_usage(pfad).percent
    return Ergebnis(name, bewerte(prozent, warnung, kritisch), f"{pfad} zu {prozent:.0f} % belegt", prozent)


def check_speicher(name, warnung=80, kritisch=90):
    prozent = psutil.virtual_memory().percent
    return Ergebnis(name, bewerte(prozent, warnung, kritisch), f"Speicher zu {prozent:.0f} % belegt", prozent)


def check_last(name, warnung=1.0, kritisch=2.0):
    """Last der letzten Minute je Kern (1,0 heißt: alle Kerne gerade ausgelastet)."""
    je_kern = os.getloadavg()[0] / (os.cpu_count() or 1)
    return Ergebnis(name, bewerte(je_kern, warnung, kritisch), f"Last je Kern {je_kern:.2f}", je_kern)


def check_prozess(name, teil):
    """OK, wenn mindestens ein Prozess läuft, in dessen Befehlszeile 'teil' vorkommt."""
    eigene = os.getpid()
    for proc in psutil.process_iter(["pid", "cmdline"]):
        try:
            if proc.info["pid"] != eigene and teil in " ".join(proc.info["cmdline"] or []):
                return Ergebnis(name, OK, f"Prozess mit '{teil}' läuft (PID {proc.info['pid']})")
        except psutil.Error:
            continue
    return Ergebnis(name, KRITISCH, f"kein Prozess mit '{teil}' gefunden")


ARTEN = {"http": check_http, "tcp": check_tcp, "platte": check_platte, "speicher": check_speicher,
         "last": check_last, "prozess": check_prozess}


def fuehre_aus(definition):
    """Führt einen Check aus der Konfiguration aus; unerwartete Fehler werden zu UNBEKANNT, nie zum Absturz."""
    definition = dict(definition)
    art, name = definition.pop("art"), definition.pop("name")
    try:
        return ARTEN[art](name, **definition)
    except KeyError:
        return Ergebnis(str(name), UNBEKANNT, f"unbekannte Art {art!r}")
    except Exception as fehler:
        return Ergebnis(name, UNBEKANNT, f"Check selbst fehlgeschlagen: {type(fehler).__name__}: {fehler}")
```

Gehen wir die Funktionen der Reihe nach durch. `check_http` fragt eine Adresse ab und prüft mehrere Dinge: Gibt es überhaupt eine Antwort (`Timeout` und `ConnectionError` werden unterschieden und enden in `KRITISCH`), hat sie den erwarteten Statuscode, kommt ein erwarteter Text vor (denn manche Fehlerseiten liefern trotzdem 200) und war die Antwort schnell genug? Die gemessene Dauer wird mit `time.monotonic()` bestimmt, und eine Antwort, die länger als `langsam` Sekunden braucht, ist eine `WARNUNG`: Der Dienst funktioniert, aber er leidet. `check_tcp` prüft, ob ein Port Verbindungen annimmt, ohne ein Protokoll zu sprechen; das ist die einfachste Form der Erreichbarkeitsprüfung und passt für Datenbanken und andere Dienste, die kein HTTP sprechen. `check_platte`, `check_speicher` und `check_last` verwenden `psutil` und `os` aus Modul 11; bei der Last wird durch die Zahl der Kerne geteilt, damit dieselben Schwellen auf kleinen und großen Servern gelten (1,0 heißt, alle Kerne sind gerade voll beschäftigt). `check_prozess` sucht in den Befehlszeilen der laufenden Prozesse nach einem Textstück; es überspringt den eigenen Prozess und fängt die Fehler von `psutil` ab (Modul 11, 11.2).

Am wichtigsten ist `fuehre_aus`: Sie umschließt jeden Aufruf mit einem `try`-Block und macht aus *jeder* unerwarteten Ausnahme ein Ergebnis mit der Stufe `UNBEKANNT`. Das breite `except Exception` ist hier ausnahmsweise richtig und gewollt: Es steht an der obersten Ebene der Prüfung, der Fehler geht nicht verloren (Typ und Meldung stehen im Ergebnis), und der Rest der Überwachung läuft weiter. An jeder anderen Stelle wäre es ein Fehler (Modul 6). Die Stufe `UNBEKANNT` wird außerdem für unbekannte Prüfarten verwendet, damit sich ein Tippfehler in der Konfiguration (`"htp"` statt `"http"`) nicht als stille Lücke in der Überwachung versteckt.

Das Beispiel zeigt alle Prüfarten im Einsatz. Unser `testweb`-Server wird gesund gestartet, dann auf Statuscode 500 gestellt, und der Pfad `/langsam` prüft das Zeitlimit und die Warnung bei langsamer Antwort. Beim TCP-Test wird ein lauschender Socket geöffnet und wieder geschlossen. Die Prozessprüfung startet zum Üben einen Kindprozess mit einem eindeutigen Stichwort in der Befehlszeile und beendet ihn danach wieder.

```python
import socket
import subprocess
import sys
import time

import testweb
from checks import check_http, check_prozess, check_tcp, bewerte, fuehre_aus

web = testweb.Web()
print("OK        :", check_http("web", web.url + "/", enthaelt="Willkommen").status)
print("Text fehlt:", check_http("web", web.url + "/", enthaelt="Impressum").status)
web.status = 500
r = check_http("web", web.url + "/")
print("Status 500:", r.status, "|", r.meldung.split(": ", 1)[1])
web.status = 200
r = check_http("web", web.url + "/langsam", timeout=1)
print("Zu langsam:", r.status, "|", r.meldung.split(": ", 1)[1])
r = check_http("web", web.url + "/langsam", timeout=5, langsam=1.0)
print("Langsam   :", r.status, "|", r.meldung.split("(")[0].split(": ", 1)[1].strip())
web.stoppe()

server = socket.socket()
server.bind(("127.0.0.1", 0))
server.listen()
print("TCP offen :", check_tcp("tcp", "127.0.0.1", server.getsockname()[1]).status)
server.close()
print("TCP zu    :", check_tcp("tcp", "127.0.0.1", 1).status)

print("Schwellen :", [bewerte(w, 80, 90) for w in (10, 79.9, 80, 89.9, 90, 100)])
kind = subprocess.Popen([sys.executable, "-c", "import time; time.sleep(30)", "dienst-4711"])
time.sleep(0.3)
print("Prozess läuft  :", check_prozess("dienst", "dienst-4711").status)
kind.terminate()
kind.wait()
print("Prozess beendet:", check_prozess("dienst", "dienst-4711").status)
print("Fehler im Check wird UNBEKANNT:", fuehre_aus({"art": "tcp", "name": "x", "host": "127.0.0.1"}).status)
print("Unbekannte Art                :", fuehre_aus({"art": "wetter", "name": "x"}).status)
```

```text
OK        : OK
Text fehlt: KRITISCH
Status 500: KRITISCH | Status 500 statt 200
Zu langsam: KRITISCH | keine Antwort in 1 s
Langsam   : WARNUNG | antwortet langsam
TCP offen : OK
TCP zu    : KRITISCH
Schwellen : ['OK', 'OK', 'WARNUNG', 'WARNUNG', 'KRITISCH', 'KRITISCH']
Prozess läuft  : OK
Prozess beendet: KRITISCH
Fehler im Check wird UNBEKANNT: UNBEKANNT
Unbekannte Art                : UNBEKANNT
```

Die erste Zeile zeigt eine erfolgreiche Prüfung mit erwartetem Text, die zweite die Prüfung auf einen Text, der nicht vorkommt: `KRITISCH`, obwohl der Statuscode 200 war. Mit Statuscode 500 meldet die Prüfung genau, was nicht stimmt („Status 500 statt 200“). Der Pfad `/langsam` braucht zwei Sekunden: Bei einem Zeitlimit von einer Sekunde ist es eine Zeitüberschreitung (`KRITISCH`), bei fünf Sekunden Limit und einer Warngrenze von einer Sekunde nur eine `WARNUNG`. Der TCP-Test unterscheidet offen und zu. Die Schwellen liefern an den Grenzen die richtigen Stufen (bei 80 und 90 beginnt jeweils die höhere). Die Prozessprüfung findet den Kindprozess, solange er läuft, und nach seinem Ende nicht mehr. Die beiden letzten Zeilen zeigen, dass eine unvollständige Definition (der Eintrag `port` fehlt) und eine unbekannte Art keine Ausnahme nach außen geben, sondern `UNBEKANNT`.

**Messwerte richtig deuten.** Einige Dinge verdienen besondere Aufmerksamkeit, weil sie in der Praxis zu falschen Alarmen oder zu verpassten Störungen führen. Bei der *Last* ist ein einzelner Wert wenig aussagekräftig; die Last der letzten Minute schwankt stark, deshalb sollte nur ein *anhaltender* Wert eine Meldung auslösen. Das erreichen wir nicht in der Prüfung selbst, sondern im Zustandsspeicher von 12.4, der erst nach mehreren Auffälligkeiten in Folge meldet. Beim *Speicher* gilt, was in Modul 11 stand (`percent` statt `free`). Bei der *Platte* reicht der Prozentwert allein nicht: Eine große Platte mit 90 Prozent hat noch viel Platz, eine kleine nicht; wer es genau nimmt, prüft zusätzlich den absoluten freien Platz. Und bei *HTTP-Prüfungen* muss man entscheiden, was als gesund gilt: Eine Weiterleitung (301 oder 302) ist nicht dasselbe wie eine Antwort, und `requests` folgt Weiterleitungen standardmäßig; wer genau die Antwort einer bestimmten Adresse prüfen will, setzt `allow_redirects=False` und erwartet den entsprechenden Code.

**Prüfungen für TLS-Zertifikate.** Eine Prüfung, die in der Praxis oft fehlt und dann einen peinlichen Ausfall verursacht, ist die Gültigkeitsdauer von *Zertifikaten*. Läuft das Zertifikat einer Website ab, zeigen Browser Warnseiten, und automatische Abfragen wie die unseren scheitern mit einem Zertifikatsfehler. Deshalb prüft man die *Restlaufzeit* und warnt rechtzeitig. Wir bauen diese Prüfung in Aufgabe 7 selbst.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, nur zu prüfen, ob ein *Prozess* läuft, statt ob der *Dienst antwortet*: Ein hängender Webserver ist ein laufender Prozess und trotzdem ein Ausfall. Der zweite ist eine Prüfung ohne Zeitlimit, die bei einem hängenden Ziel selbst hängt. Der dritte ist ein `except:` ohne Typ oder ein stilles `pass`, das Fehler der Prüfung als „in Ordnung“ behandelt. Der vierte ist, die Prozessprüfung zu locker zu schreiben: Wer nach „python“ sucht, findet jedes Python-Programm, und wer das Stichwort in der eigenen Shell eintippt, findet womöglich die Shell selbst, deren Befehlszeile das Stichwort enthält (beim Entwickeln dieses Moduls ist das tatsächlich passiert; deshalb schließt `check_prozess` den eigenen Prozess aus und die Demonstration benutzt ein eindeutiges Stichwort). Das Gegenüber:

```python
import requests

# FALSCH: kein Zeitlimit, Fehler werden zu "alles gut", nur das Ergebnis true/false
def web_ok(url):
    try:
        return requests.get(url).status_code == 200
    except:
        return True                       # "wird schon gehen"

# RICHTIG: Zeitlimit, Fehlerarten unterscheiden, Stufe und Meldung zurückgeben
from checks import check_http

ergebnis = check_http("web", "https://example.org/", erwartet=200, timeout=5)
print(ergebnis.status, ergebnis.meldung)  # z. B. KRITISCH https://example.org/: nicht erreichbar
```

Die falsche Variante hängt bei einem nicht antwortenden Server unbegrenzt, und wenn das Netz ausfällt, meldet sie ausgerechnet „in Ordnung“; sie ist damit schlimmer als gar keine Prüfung, weil sie falsche Sicherheit vermittelt. Die richtige unterscheidet die Fehlerarten, setzt ein Zeitlimit und liefert Stufe und Meldung.

> **Ergänzung aus der Nachrecherche**
>
> **Konventionen und Quellenlage.** Die Stufen `OK`, `WARNUNG`, `KRITISCH`, `UNBEKANNT` und die Exit-Codes 0 bis 3 folgen der in der Überwachungswelt üblichen Konvention (Nagios und kompatible Systeme); sie ist hier Fachwissen und wurde in dieser Fassung nicht durch eine abgerufene Quelle belegt. Die Dokumentation von `psutil` war nicht abrufbar (siehe Modul 11); die verwendeten Funktionen (`disk_usage`, `virtual_memory`, `process_iter`) wurden mit psutil 7.2.2 selbst geprüft. Zum Verhalten von `requests` bei Weiterleitungen und Zeitlimits siehe Modul 10; die Dokumentation von `requests` war dort nicht abrufbar. Alle Beispiele dieser Lerneinheit wurden ausgeführt; die Messwerte für Platte, Speicher und Last hängen vom Rechner ab, deshalb prüfen die Beispiele die Logik an Grenzfällen statt echte Werte auszugeben.

> **Merke**
>
> Ein Health-Check besteht aus kleinen Prüfungen mit einheitlichem Ergebnis (Name, Stufe, Meldung, Wert). Stufen: OK, WARNUNG, KRITISCH, UNBEKANNT. Prüfen, was der Benutzer erlebt (Antwort statt nur Prozess), immer mit Zeitlimit, Fehlerarten unterscheiden. Die Schwelle gehört schon zur höheren Stufe. Unerwartete Fehler der Prüfung selbst werden zu UNBEKANNT, nie zu OK und nie zum Absturz. Prüfarten über eine Registratur an die Konfiguration binden.

> **Praxisbeispiel**
>
> Professionelle Überwachungssysteme (Nagios, Icinga, Prometheus mit Alertmanager, Zabbix und andere) arbeiten nach genau diesem Muster: viele kleine Prüfungen, einheitliche Stufen, getrennte Bewertung und Alarmierung. Ihr eigenes Skript ersetzt sie nicht, aber es ist ein guter Weg, das Prinzip zu verstehen, und für kleine Umgebungen (ein paar Server, eine Webanwendung) oft ausreichend. Außerdem schreibt man auch dort eigene Prüfungen für Dinge, die kein Standardwerkzeug kennt: Ist das Backup von heute Nacht da? Ist der Export angekommen? Hat das Skript von gestern gelaufen?

## 12.4 Health-Check, Teil 2: Alerting

Eine Prüfung, die bei jedem Lauf eine Mail schickt, solange etwas nicht stimmt, ist eine Plage. Läuft der Health-Check alle fünf Minuten und fällt der Webserver für eine Stunde aus, kommen zwölf Mails mit dem gleichen Inhalt; fällt eine Prüfung bei jedem dritten Lauf kurz aus, kommen ständig Alarm und Entwarnung im Wechsel. Menschen reagieren darauf, indem sie die Meldungen ignorieren oder abbestellen, und dann geht auch der eine Alarm unter, der wichtig war. Das Problem heißt *Alarmmüdigkeit* (*alert fatigue*) und ist der häufigste Grund, warum Überwachung in der Praxis nicht funktioniert. In dieser Lerneinheit lösen wir es mit drei Bausteinen: einem *Zustandsspeicher*, der sich merkt, was beim letzten Lauf los war, einer *Entscheidungsfunktion*, die nur bei *Veränderungen* meldet, und *Benachrichtigern*, die eine Meldung über mehrere Kanäle senden, ohne dass ein kaputter Kanal die anderen mitreißt.

**Warum man sich den Zustand merken muss.** Ein Programm, das alle fünf Minuten startet (cron oder systemd-Timer, Modul 9), hat von selbst kein Gedächtnis: Jeder Lauf beginnt bei null. Um „es ist *noch* kaputt“ von „es ist *neu* kaputt“ zu unterscheiden, muss das Programm zwischen den Läufen etwas speichern. Der einfachste Weg ist eine kleine Datei im JSON-Format (Modul 7), die je Prüfung den letzten Status, die Zahl der Fehlversuche in Folge und den Zeitpunkt der letzten Meldung enthält. Zwei Regeln für diese Datei kennen Sie aus Modul 9 (9.4): Sie muss *atomar* geschrieben werden, also zuerst in eine Zwischendatei im selben Ordner und dann umbenannt, damit ein Absturz mittendrin keine halbe Datei hinterlässt, und ein defekter oder fehlender Zustand darf das Programm nicht stoppen, sondern führt zu einem frischen Anfang. Dass ein frischer Anfang eine Meldung verpassen oder doppelt senden kann, ist der Preis der Einfachheit und im Allgemeinen vertretbar.

**Die Entscheidungsregeln.** Die Funktion `entscheide` ist der fachliche Kern des Alertings. Sie bekommt den gemerkten Zustand einer Prüfung, das neue Ergebnis und die aktuelle Zeit und liefert den neuen Zustand sowie ein *Ereignis*. Die Regeln, die wir umsetzen, sind in der Praxis bewährt. Erstens: Ein Alarm kommt erst, wenn die Prüfung *mehrmals hintereinander* nicht in Ordnung war (der Parameter `schwelle`, Vorgabe 2). Das unterdrückt kurze Ausreißer, wie sie bei Netzwerken und überlasteten Servern ständig vorkommen. Je höher die Schwelle, desto weniger falsche Alarme, aber desto später die Meldung echter Störungen; bei einem Prüfintervall von fünf Minuten und einer Schwelle von zwei vergeht bis zum Alarm eine Spanne von bis zu zehn Minuten, und diese Verzögerung ist der Preis. Zweitens: Ist ein Alarm einmal gemeldet, wird *nicht jedes Mal erneut gemeldet*. Drittens: Wird die Lage *schlimmer* (von `WARNUNG` zu `KRITISCH`), gibt es das Ereignis `VERSCHLECHTERT`. Viertens: Bleibt die Störung lange bestehen, kommt nach der Frist `erinnerung` (Vorgabe eine Stunde) eine `ERINNERUNG`, damit sie nicht vergessen wird. Fünftens: Wird die Prüfung wieder `OK`, und es war ein Alarm gemeldet, kommt die *Entwarnung* (`ERHOLT`); war nie ein Alarm gemeldet, bleibt es still.

Diese Entscheidungsfunktion ist eine *reine Funktion*: Sie liest keine Uhr, keine Datei und kein Netzwerk, die Zeit kommt als Parameter (`jetzt`). Das macht sie mit einfachen Mitteln vollständig testbar, und das tun wir als Nächstes. Zuvor der Code des ganzen Moduls `alarm.py`, den wir danach abschnittsweise besprechen.

```python
import json
import os
import tempfile
from pathlib import Path

from checks import KRITISCH, OK, SCHWERE
from hooks import WebhookFehler, sende_webhook, slack_nachricht, teams_nachricht
from mailer import MailFehler, baue_nachricht, sende


def lade_zustand(pfad):
    """Liest den gemerkten Zustand; fehlt die Datei oder ist sie kaputt, beginnt man ohne Vorgeschichte."""
    try:
        return json.loads(Path(pfad).read_text(encoding="utf-8"))
    except (OSError, ValueError):
        return {}


def speichere_zustand(pfad, zustand):
    """Schreibt atomar: erst in eine Zwischendatei im selben Ordner, dann umbenennen (siehe Modul 9, 9.4)."""
    pfad = Path(pfad)
    pfad.parent.mkdir(parents=True, exist_ok=True)
    fd, zwischen = tempfile.mkstemp(dir=pfad.parent, prefix=pfad.name, suffix=".tmp")
    with os.fdopen(fd, "w", encoding="utf-8") as datei:
        json.dump(zustand, datei, indent=2, ensure_ascii=False)
    os.replace(zwischen, pfad)


def entscheide(alt, ergebnis, jetzt, schwelle=2, erinnerung=3600):
    """Vergleicht den gemerkten Zustand mit dem neuen Ergebnis.

    Liefert (neuer Zustand, Ereignis). Ereignis ist None, "ALARM", "VERSCHLECHTERT", "ERINNERUNG" oder "ERHOLT".
    Ein Alarm kommt erst, wenn der Check 'schwelle' Mal hintereinander nicht OK war.
    """
    neu = dict(alt or {"status": OK, "fehlversuche": 0, "gemeldet": None, "letzte_meldung": 0})
    gemeldet = neu["gemeldet"]
    neu["status"] = ergebnis.status
    if ergebnis.status == OK:
        neu.update(fehlversuche=0, gemeldet=None)
        return neu, ("ERHOLT" if gemeldet else None)
    neu["fehlversuche"] += 1
    if neu["fehlversuche"] < schwelle:
        return neu, None
    if not gemeldet:
        neu.update(gemeldet=ergebnis.status, letzte_meldung=jetzt)
        return neu, "ALARM"
    if SCHWERE[ergebnis.status] > SCHWERE[gemeldet]:
        neu.update(gemeldet=ergebnis.status, letzte_meldung=jetzt)
        return neu, "VERSCHLECHTERT"
    if jetzt - neu["letzte_meldung"] >= erinnerung:
        neu["letzte_meldung"] = jetzt
        return neu, "ERINNERUNG"
    return neu, None


def formuliere(ereignis, ergebnis, host):
    """Betreff und Text einer Meldung. Der Betreff enthält alles Wichtige für die Vorschau im Postfach."""
    betreff = f"[{ereignis}] {ergebnis.name} auf {host}: {ergebnis.status}"
    text = f"{ergebnis.meldung}\n\nEreignis: {ereignis}\nHost: {host}\nCheck: {ergebnis.name}\n"
    return betreff, text


class LogBenachrichtiger:
    def __init__(self, ausgabe=print):
        self.ausgabe = ausgabe

    def sende(self, betreff, text):
        self.ausgabe(betreff)


class MailBenachrichtiger:
    def __init__(self, konfig, empfaenger):
        self.konfig, self.empfaenger = konfig, empfaenger

    def sende(self, betreff, text):
        sende(self.konfig, baue_nachricht(self.konfig.absender, self.empfaenger, betreff, text))


class WebhookBenachrichtiger:
    def __init__(self, url, art="slack", **optionen):
        self.url, self.art, self.optionen = url, art, optionen

    def sende(self, betreff, text):
        nutzlast = teams_nachricht(betreff, text) if self.art == "teams" else slack_nachricht(f"{betreff}\n{text}")
        sende_webhook(self.url, nutzlast, **self.optionen)


def melde(benachrichtiger, ereignis, ergebnis, host):
    """Schickt die Meldung über alle Kanäle. Ein kaputter Kanal verhindert die anderen nicht.

    Liefert die Liste der Kanäle, die nicht zugestellt haben, als Texte.
    """
    betreff, text = formuliere(ereignis, ergebnis, host)
    probleme = []
    for kanal in benachrichtiger:
        try:
            kanal.sende(betreff, text)
        except (MailFehler, WebhookFehler) as fehler:
            probleme.append(f"{type(kanal).__name__}: {fehler}")
        except Exception as fehler:                                   # letzte Sicherung: nichts darf entweichen
            probleme.append(f"{type(kanal).__name__}: unerwartet {type(fehler).__name__}")
    return probleme
```

Der erste Teil sind die beiden Funktionen für den Zustandsspeicher: `lade_zustand` fängt `OSError` und `ValueError` (fehlende oder kaputte Datei) und liefert dann ein leeres Wörterbuch; `speichere_zustand` legt mit `tempfile.mkstemp(dir=...)` eine Zwischendatei im selben Ordner an (nur im selben Dateisystem ist das Umbenennen atomar), schreibt sie und ersetzt dann mit `os.replace` die Zieldatei. Der zweite Teil ist `entscheide` mit den fünf Regeln. Der dritte Teil, `formuliere`, macht aus Ereignis und Ergebnis einen Betreff und einen Text; im Betreff steht alles Wichtige (Ereignis, Prüfung, Rechner, Stufe), weil auf dem Telefon oft nur er sichtbar ist. Der vierte Teil sind die drei Benachrichtiger, die dieselbe Schnittstelle haben (`sende(betreff, text)`): einer, der in das Protokoll schreibt, einer für Mail (mit `mailer.py` aus 12.1) und einer für Webhooks (mit `hooks.py` aus 12.2). Ein neuer Kanal (SMS, Telefon, ein anderes Chat-System) ist dann eine weitere Klasse mit einer Methode, ohne dass sich der Rest ändert. Der fünfte Teil, `melde`, schickt die Meldung an alle Kanäle und sammelt die Fehler der einzelnen Kanäle in einer Liste, statt bei dem ersten abzubrechen.

Prüfen wir zunächst die Entscheidungsregeln, indem wir Folgen von Ergebnissen durchspielen. Das Beispiel gibt je Lauf den Status und das Ereignis aus; „-“ bedeutet, dass nichts gemeldet wird. Die Zeit läuft in Sekunden, die Schwelle ist zwei, die Erinnerung nach einer Stunde.

```python
from checks import Ergebnis, KRITISCH, OK, WARNUNG
from alarm import entscheide


def lauf(folge, **optionen):
    """Spielt eine Folge von (Sekunden, Status) durch und gibt die Ereignisse als Liste aus."""
    zustand, ereignisse = None, []
    for sekunden, status in folge:
        zustand, ereignis = entscheide(zustand, Ergebnis("web", status, "..."), sekunden, **optionen)
        ereignisse.append(ereignis or "-")
    return ereignisse


# Ein Ausfall, der erst nach zwei Fehlversuchen gemeldet wird, später eine Erinnerung und die Entwarnung
folge = [(0, OK), (60, KRITISCH), (120, KRITISCH), (180, KRITISCH), (3800, KRITISCH), (3860, OK), (3920, OK)]
print("Status   :", [s for _, s in folge])
print("Ereignis :", lauf(folge, schwelle=2, erinnerung=3600))

# Flackern: nie zwei Fehler hintereinander, also kein Alarm
flackern = [(0, KRITISCH), (60, OK), (120, KRITISCH), (180, OK), (240, KRITISCH), (300, OK)]
print("Flackern :", lauf(flackern, schwelle=2))
print("Flackern, Schwelle 1:", lauf(flackern, schwelle=1))

# Verschlechterung: erst Warnung, dann kritisch
print("Warnung -> kritisch :", lauf([(0, WARNUNG), (60, WARNUNG), (120, KRITISCH)], schwelle=2))
print("Sofort wieder OK    :", lauf([(0, WARNUNG), (60, WARNUNG), (120, OK)], schwelle=2))
```

```text
Status   : ['OK', 'KRITISCH', 'KRITISCH', 'KRITISCH', 'KRITISCH', 'OK', 'OK']
Ereignis : ['-', '-', 'ALARM', '-', 'ERINNERUNG', 'ERHOLT', '-']
Flackern : ['-', '-', '-', '-', '-', '-']
Flackern, Schwelle 1: ['ALARM', 'ERHOLT', 'ALARM', 'ERHOLT', 'ALARM', 'ERHOLT']
Warnung -> kritisch : ['-', 'ALARM', 'VERSCHLECHTERT']
Sofort wieder OK    : ['-', 'ALARM', 'ERHOLT']
```

Die erste Folge zeigt den ganzen Lebenslauf einer Störung: Der erste Fehler (Zeit 60) bleibt still, der zweite (Zeit 120) löst den `ALARM` aus, der dritte (180) ist keine Meldung wert, weil schon gemeldet wurde, der vierte, mehr als eine Stunde nach der letzten Meldung (3800), löst die `ERINNERUNG` aus, und die Rückkehr zu `OK` bringt die Entwarnung (`ERHOLT`). Danach bleibt es still. Die zweite Folge zeigt das Flackern: Weil nie zwei Fehler hintereinander auftreten, gibt es keinen einzigen Alarm. Mit der Schwelle eins dagegen wären es drei Alarme und drei Entwarnungen in sechs Minuten, genau die Plage, die wir vermeiden wollten. Die dritte Folge zeigt die Verschlechterung, die vierte, dass eine Warnung, die sich von selbst erledigt, mit Alarm und Entwarnung endet (hier ist die Schwelle erreicht worden).

Nun der Zustandsspeicher auf der Platte: fehlende Datei, Speichern in einen noch nicht existierenden Ordner, Wiedereinlesen, eine kaputte Datei und zwei aufeinanderfolgende Läufe ohne übrig gebliebene Zwischendateien.

```python
import json
import tempfile
from pathlib import Path

from alarm import lade_zustand, speichere_zustand

ordner = Path(tempfile.mkdtemp())
pfad = ordner / "zustand" / "healthcheck.json"          # der Unterordner existiert noch nicht

print("Datei fehlt        ->", lade_zustand(pfad))
speichere_zustand(pfad, {"web": {"status": "KRITISCH", "fehlversuche": 3, "gemeldet": "KRITISCH", "letzte_meldung": 1234}})
print("Gespeichert        ->", sorted(p.name for p in pfad.parent.iterdir()))
print("Wieder eingelesen  ->", lade_zustand(pfad)["web"]["fehlversuche"])

pfad.write_text('{"web": {"status": "KRIT', encoding="utf-8")     # Absturz mitten im Schreiben (so passiert es bei einfachem write_text)
print("Kaputte Datei      ->", lade_zustand(pfad))

speichere_zustand(pfad, {"a": 1})
speichere_zustand(pfad, {"a": 2})
print("Nach zwei Läufen   ->", json.loads(pfad.read_text())["a"], "| Zwischendateien übrig:",
      [p.name for p in pfad.parent.iterdir() if p.name.endswith(".tmp")])
```

```text
Datei fehlt        -> {}
Gespeichert        -> ['healthcheck.json']
Wieder eingelesen  -> 3
Kaputte Datei      -> {}
Nach zwei Läufen   -> 2 | Zwischendateien übrig: []
```

Die fehlende Datei führt zu einem leeren Zustand, das Speichern legt den Ordner an und erzeugt genau eine Datei. Die absichtlich abgeschnittene Datei (so entsteht sie bei einem Absturz mitten im Schreiben mit `write_text`) liest das Programm als leeren Zustand; sie führt nicht zum Absturz. Nach zwei Läufen steht der neue Wert in der Datei, und es sind keine Zwischendateien übrig.

Zuletzt die Zustellung. Das Beispiel richtet drei Kanäle ein (Protokoll, Mail an den Testserver, Webhook an den Testempfänger) und sendet einen Alarm. Danach schaltet es den Webhook-Empfänger ab, sendet eine Entwarnung und zeigt, dass die Mail trotzdem ankommt und der Fehler des Webhooks als Text gemeldet wird.

```python
from email import message_from_bytes, policy

import testhook
import testsmtp
from alarm import LogBenachrichtiger, MailBenachrichtiger, WebhookBenachrichtiger, melde
from checks import Ergebnis, KRITISCH
from mailer import MailKonfig

mailserver = testsmtp.Server()
hook = testhook.Empfaenger()
protokoll = []
konfig = MailKonfig("127.0.0.1", mailserver.port, "monitoring@beispiel.example", modus="klartext", timeout=3)

kanaele = [
    LogBenachrichtiger(ausgabe=protokoll.append),
    MailBenachrichtiger(konfig, ["admin@beispiel.example"]),
    WebhookBenachrichtiger(hook.url, art="slack", versuche=1),
]
ergebnis = Ergebnis("web", KRITISCH, "http://web01/: Status 500 statt 200")

probleme = melde(kanaele, "ALARM", ergebnis, "web01")
print("Probleme              :", probleme)
print("Log                   :", protokoll)
mail = message_from_bytes(mailserver.sammler.nachrichten[0]["inhalt"], policy=policy.default)
print("Mail-Betreff          :", mail["Subject"])
print("Webhook, erste Zeile  :", hook.empfangen[0]["nutzlast"]["text"].splitlines()[0])

# Jetzt ist der Webhook kaputt (Empfänger abgeschaltet), und die Mail darf trotzdem ankommen
hook.stoppe()
probleme = melde(kanaele, "ERHOLT", Ergebnis("web", "OK", "wieder erreichbar"), "web01")
print("Probleme nach Ausfall :", probleme)
print("Mails angekommen      :", len(mailserver.sammler.nachrichten))
mailserver.stoppe()
```

```text
Probleme              : []
Log                   : ['[ALARM] web auf web01: KRITISCH']
Mail-Betreff          : [ALARM] web auf web01: KRITISCH
Webhook, erste Zeile  : [ALARM] web auf web01: KRITISCH
Probleme nach Ausfall : ['WebhookBenachrichtiger: http://127.0.0.1/hooks/*** nach 1 Versuchen nicht erreichbar (ConnectionError)']
Mails angekommen      : 2
```

Im ersten Fall kommen alle drei Meldungen an, im Betreff steht `[ALARM] web auf web01: KRITISCH`. Im zweiten Fall ist der Webhook nicht erreichbar: `melde` liefert einen Eintrag mit dem Kanal und der Ursache (die URL erscheint nur maskiert), und die Mail kam trotzdem an (zwei Mails insgesamt). Der Protokollkanal gehört zur Sicherheit immer dazu: Er schreibt, wenn alle anderen versagen.

**Was in eine gute Meldung gehört.** Der Text einer Meldung entscheidet, ob der Empfänger sofort weiß, was zu tun ist. Eine gute Meldung nennt: *was* betroffen ist (Prüfung, Dienst), *wo* (Rechner), *was* nicht stimmt (der Messwert und die Schwelle, nicht nur „Fehler“), *seit wann* und, wenn bekannt, *was als Nächstes zu tun ist* (ein Link zur Anleitung für diesen Alarm, ein Hinweis auf den Verantwortlichen). Der Betreff beginnt mit dem Ereignis in eckigen Klammern, weil man danach im Postfach filtern kann. Meldungen sollen keine Geheimnisse und möglichst keine personenbezogenen Daten enthalten (siehe die Ergänzung unten). In Aufgabe 8 erweitern wir das Alerting um Wartungsfenster.

**Ausfall der Überwachung selbst.** Eine häufig übersehene Frage lautet: Wer überwacht den Überwacher? Fällt der Server aus, auf dem der Health-Check läuft, kommt keine Meldung, und die Stille wird als „alles in Ordnung“ gedeutet. Professionelle Systeme lösen das mit einem *Totmannschalter* (*dead man’s switch*): Der Health-Check meldet bei jedem erfolgreichen Lauf an einen zweiten Dienst, und dieser alarmiert, wenn die Meldungen *ausbleiben*. Auch eine einfache Variante hilft: ein zweiter Rechner, der prüft, ob der erste noch lebt. Im Rahmen des Moduls reicht es, die Frage zu kennen; im Abschlussprojekt kann sie ein Thema sein.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist, bei jedem Lauf zu melden, solange die Störung besteht („Alarm-Dauerfeuer“). Der zweite ist das Gegenstück: eine Schwelle, die so hoch ist, dass die Meldung erst nach einer Stunde kommt, oder gar keine Entwarnung, sodass niemand weiß, ob die Störung vorbei ist. Der dritte ist, den Zustand nicht atomar oder gar nicht zu speichern. Der vierte ist, die Zustellung ohne Fehlerbehandlung zu schreiben: Wenn der Mailserver ausfällt, bricht das Programm ab, und der Webhook-Kanal bekommt die Meldung nicht mehr. Der fünfte, die Meldung als *Einzelsendung ohne Ausweichweg* zu bauen. Das Gegenüber zu den ersten beiden Fehlern:

```python
# FALSCH: bei jedem Lauf melden, solange es kaputt ist; Fehler der Zustellung beenden das Programm
for ergebnis in ergebnisse:
    if ergebnis.status != "OK":
        mail_senden(ergebnis)                 # Mailserver down -> Ausnahme -> kein Webhook, keine weiteren Checks
        webhook_senden(ergebnis)

# RICHTIG: nur bei Veränderung melden, jeden Kanal einzeln absichern
from alarm import entscheide, melde

for ergebnis in ergebnisse:
    zustand[ergebnis.name], ereignis = entscheide(zustand.get(ergebnis.name), ergebnis, jetzt)
    if ereignis:
        for problem in melde(kanaele, ereignis, ergebnis, "web01"):
            log.error("Zustellung fehlgeschlagen: %s", problem)
```

Die falsche Variante schickt bei einer stundenlangen Störung zwölf gleiche Mails und kommt bei einem Ausfall des Mailservers nicht mehr zum Webhook; die richtige meldet nur beim Wechsel, merkt sich den Zustand, sendet über alle Kanäle und protokolliert die Probleme.

> **Ergänzung aus der Nachrecherche**
>
> **Datenschutz und Recht (keine Rechtsberatung).** Meldungen und Zustellwege können personenbezogene Daten berühren, etwa wenn Benutzernamen, Mailadressen oder IP-Adressen in Meldungen stehen oder wenn Meldungen über einen Chat-Dienst eines Anbieters laufen. Zu prüfen sind in der Regel: ob die Meldungen auf das Nötige beschränkt sind (Datensparsamkeit), ob mit dem Anbieter eines Dienstes ein Vertrag zur Auftragsverarbeitung nach Artikel 28 der DSGVO besteht und ob bei Anbietern außerhalb der EU die Regeln zur Übermittlung in Drittländer (Kapitel V der DSGVO) eingehalten werden. Diese Hinweise sind allgemeines Fachwissen und wurden in diesem Skript *nicht* aus einer abgerufenen Quelle belegt; sie sind *keine Rechtsberatung*. Fragen Sie in der Praxis Ihren Datenschutzbeauftragten oder Ihre Rechtsabteilung. Technisch bewährt: Meldungen ohne personenbezogene Daten und ohne Geheimnisse formulieren und Details über einen Link auf ein zugangsgeschütztes System bereitstellen.
>
> **Zur Methode.** Die Regeln des Alertings (Schwelle, Erinnerung, Entwarnung, Totmannschalter) beschreiben bewährte Praxis der Betriebsüberwachung; sie sind hier Fachwissen und wurden nicht aus einer abgerufenen Quelle belegt. Alle Beispiele wurden ausgeführt, die Entscheidungsregeln durch die Ablaufprotokolle belegt.

> **Merke**
>
> Alerting meldet *Veränderungen*, nicht Zustände: erst nach `schwelle` Fehlversuchen in Folge, danach höchstens bei Verschlechterung, Erinnerung nach Frist und Entwarnung bei Rückkehr zu OK. Dafür braucht man einen Zustandsspeicher (JSON-Datei, atomar geschrieben, bei Defekt neu beginnen). Die Entscheidung ist eine reine Funktion mit der Zeit als Parameter und deshalb testbar. Mehrere Kanäle, jeder einzeln abgesichert, und immer ein Protokollkanal. Gute Meldungen: Betreff mit Ereignis, Prüfung und Rechner, Text mit Messwert, Schwelle und nächstem Schritt, keine Geheimnisse.

> **Praxisbeispiel**
>
> Ob eine Überwachung angenommen wird, entscheidet sich an der Qualität der Meldungen. Teams starten oft mit zu vielen Alarmen und stumpfen ab; die bewährte Regel lautet, dass jeder Alarm *eine Handlung* auslösen muss. Alarme, auf die niemand reagiert, werden entweder abgeschafft oder zu Warnungen herabgestuft, die nur gesammelt gemeldet werden (zum Beispiel als Tagesbericht, Aufgabe 5 in diesem Modul). Außerdem lohnt es sich, Alarme zu *üben*: einen Testalarm auszulösen und zu sehen, wer ihn bekommt und wie lange es dauert.

## 12.5 Health-Check abschließen, Themenwahl Abschlussprojekt und Quiz

Wir haben jetzt alle Bausteine beisammen: die Mail (12.1), den Webhook (12.2), die Prüfungen (12.3) und das Alerting mit Zustandsspeicher (12.4). Der Meilenstein des Moduls ist das fertige Programm, das sie verbindet: ein Kommandozeilenwerkzeug, das aus einer Konfigurationsdatei liest, was zu prüfen ist und wohin gemeldet wird, die Prüfungen ausführt, Veränderungen meldet, den Zustand speichert und mit einem Exit-Code endet, den ein Zeitplaner auswerten kann. Es verwendet dabei alles Bisherige: `argparse` (Modul 5), `logging` und Exit-Codes (Modul 6), JSON-Konfiguration (Modul 7), Geheimnisse aus der Umgebung (Modul 10) und die Struktur testbarer Programme, bei denen die Hauptfunktion ihre Argumente, die Umgebung und die Uhr als Parameter bekommt (Modul 9, 9.5).

**Anforderungen.** Das Programm `healthcheck` liest eine JSON-Datei mit dem Namen des Rechners, dem Ort der Zustandsdatei, den Schwellen für Alarm und Erinnerung, der Liste der Prüfungen und den Einstellungen für Mail und Webhook. Zugangsdaten stehen *nicht* in der Datei: Die Konfiguration enthält nur die *Namen* der Umgebungsvariablen, aus denen das Passwort des Mailservers und die Webhook-URL gelesen werden. Das Programm führt alle Prüfungen aus, gibt eine Zeile je Prüfung aus (Stufe, Name, Meldung), meldet Veränderungen auf allen eingerichteten Kanälen und speichert den Zustand. Mit `--trockenlauf` sendet es nichts und speichert nichts, zeigt aber, was es melden *würde*; so lässt sich eine neue Konfiguration gefahrlos ausprobieren. Der Exit-Code ist die schlimmste gefundene Stufe (0 = alles in Ordnung, 1 = Warnung, 2 = kritisch, 3 = unbekannt) oder 4 bei einem Fehler der Konfiguration. Ein unzustellbarer Kanal führt zu einer Fehlermeldung im Protokoll, aber nicht zum Abbruch.

**Der Entwurf.** Das Hauptprogramm ist bewusst schmal, weil die Fachlogik in den Modulen steckt, die wir gebaut haben. `lade_konfig` liest und prüft die Konfigurationsdatei und löst bei Problemen eine `KonfigFehler` mit lesbarer Meldung aus. `baue_kanaele` erzeugt die Benachrichtiger aus der Konfiguration und holt die Geheimnisse aus der Umgebung; fehlt eine verlangte Variable, ist das ein Konfigurationsfehler, und zwar *vor* der ersten Prüfung, nicht erst, wenn der erste Alarm gesendet werden soll („früh scheitern“). Im Trockenlauf gibt es nur den Protokollkanal. `main` ist die Hauptfunktion; sie nimmt die Argumente, die Umgebung und die Uhr als Parameter, damit ein Test sie mit eigenen Werten aufrufen kann, ohne echte Umgebungsvariablen zu verändern und ohne zu warten.

```python
#!/usr/bin/env python3
"""healthcheck - prüft Dienste und Ressourcen und meldet Veränderungen per Mail und Webhook."""
import argparse
import json
import logging
import os
import sys
import time
from pathlib import Path

from alarm import (LogBenachrichtiger, MailBenachrichtiger, WebhookBenachrichtiger, entscheide, lade_zustand,
                   melde, speichere_zustand)
from checks import SCHWERE, fuehre_aus
from mailer import MailKonfig

log = logging.getLogger("healthcheck")
EXIT_KONFIG = 4                                   # 0 OK, 1 WARNUNG, 2 KRITISCH, 3 UNBEKANNT (wie bei Nagios)


class KonfigFehler(Exception):
    pass


def lade_konfig(pfad):
    try:
        konfig = json.loads(Path(pfad).read_text(encoding="utf-8"))
    except (OSError, ValueError) as fehler:
        raise KonfigFehler(f"Konfiguration {pfad} nicht lesbar: {fehler}") from fehler
    for pflicht in ("host", "zustand", "checks"):
        if pflicht not in konfig:
            raise KonfigFehler(f"Konfiguration: Eintrag '{pflicht}' fehlt")
    return konfig


def baue_kanaele(konfig, umgebung, trockenlauf):
    """Erzeugt die Benachrichtiger; Geheimnisse kommen aus der Umgebung, nicht aus der Datei."""
    kanaele = [LogBenachrichtiger(ausgabe=lambda text: print("MELDUNG:", text))]
    if trockenlauf:
        return kanaele
    if "mail" in konfig:
        m = konfig["mail"]
        passwort = umgebung.get(m.get("passwort_env", ""), "")
        if m.get("benutzer") and not passwort:
            raise KonfigFehler(f"Umgebungsvariable {m['passwort_env']} fehlt (Passwort für den Mailserver)")
        mailkonfig = MailKonfig(m["host"], m["port"], m["absender"], modus=m.get("modus", "starttls"),
                                benutzer=m.get("benutzer", ""), passwort=passwort, cafile=m.get("cafile", ""))
        kanaele.append(MailBenachrichtiger(mailkonfig, m["empfaenger"]))
    if "webhook" in konfig:
        w = konfig["webhook"]
        url = umgebung.get(w["url_env"], "")
        if not url:
            raise KonfigFehler(f"Umgebungsvariable {w['url_env']} fehlt (Webhook-URL)")
        kanaele.append(WebhookBenachrichtiger(url, art=w.get("art", "slack"), versuche=2, pause=0.5))
    return kanaele


def main(argv=None, umgebung=None, jetzt=time.time):
    umgebung = os.environ if umgebung is None else umgebung
    parser = argparse.ArgumentParser(description="Dienste prüfen und Veränderungen melden")
    parser.add_argument("--konfig", type=Path, required=True)
    parser.add_argument("--trockenlauf", action="store_true", help="nichts senden, Zustand nicht speichern")
    parser.add_argument("-v", "--verbose", action="store_true")
    args = parser.parse_args(argv)
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO, format="%(levelname)s %(message)s")

    try:
        konfig = lade_konfig(args.konfig)
        kanaele = baue_kanaele(konfig, umgebung, args.trockenlauf)
    except KonfigFehler as fehler:
        log.error("%s", fehler)
        return EXIT_KONFIG

    zustand = lade_zustand(konfig["zustand"])
    ergebnisse = [fuehre_aus(definition) for definition in konfig["checks"]]
    schlimmster = 0
    for ergebnis in ergebnisse:
        print(f"{ergebnis.status:10} {ergebnis.name:10} {ergebnis.meldung}")
        schlimmster = max(schlimmster, SCHWERE[ergebnis.status])
        zustand[ergebnis.name], ereignis = entscheide(
            zustand.get(ergebnis.name), ergebnis, jetzt(),
            schwelle=konfig.get("schwelle", 2), erinnerung=konfig.get("erinnerung", 3600))
        if ereignis:
            for problem in melde(kanaele, ereignis, ergebnis, konfig["host"]):
                log.error("Zustellung fehlgeschlagen: %s", problem)
    if not args.trockenlauf:
        speichere_zustand(konfig["zustand"], zustand)
    return schlimmster


if __name__ == "__main__":
    sys.exit(main())
```

Gehen wir `main` durch. Zuerst werden die Argumente gelesen; `--konfig` ist Pflicht, deshalb meldet `argparse` ohne sie einen Fehler und beendet das Programm mit dem Code 2 (so läuft auch der automatische Test dieses Listings ab). Danach wird die Konfiguration geladen und die Kanäle gebaut; bei einem `KonfigFehler` gibt es eine Meldung im Protokoll und den Exit-Code 4. Dann wird der Zustand geladen, jede Prüfung ausgeführt (`fuehre_aus` wirft nie), das Ergebnis ausgegeben und der schlimmste Wert gemerkt. Für jede Prüfung wird `entscheide` aufgerufen, der neue Zustand notiert und bei einem Ereignis `melde` aufgerufen; Probleme bei der Zustellung werden als Fehler protokolliert. Am Ende wird der Zustand gespeichert (außer im Trockenlauf). Die Ausgabe der Prüfungen geht nach `stdout`, die Fehlermeldungen des Programms über `logging` nach `stderr`: Wer das Programm in ein anderes Programm leitet, erhält nur die Daten (Modul 5).

Zwei Entwurfsentscheidungen sind erwähnenswert. Die Meldungen im Protokollkanal erscheinen mit dem Wort `MELDUNG:` auf `stdout`; das ist hier so gewählt, damit man sie im Beispiel sieht, und im Betrieb würde man eher das Protokoll nutzen. Und der Zustand wird auch dann gespeichert, wenn die Zustellung eines Alarms scheiterte; damit gilt der Alarm als „gemeldet“, obwohl keiner ankam. Eine Alternative wäre, den Zustand dann nicht zu übernehmen, damit beim nächsten Lauf erneut versucht wird; das erhöht die Zuverlässigkeit und das Risiko doppelter Meldungen. Ein Betreiber muss sich für eine der beiden Seiten entscheiden.

**Eine Konfigurationsdatei.** So sieht eine Konfiguration für einen Webserver aus. Die Schwellen und die Prüfungen sind Daten, keine Programmteile; ein Administrator kann eine Prüfung ergänzen, ohne Python zu können.

```python
{
  "host": "web01",
  "zustand": "/var/lib/healthcheck/zustand.json",
  "schwelle": 2,
  "erinnerung": 3600,
  "checks": [
    {"art": "http", "name": "shop", "url": "https://shop.example.org/", "enthaelt": "Warenkorb", "timeout": 5},
    {"art": "tcp", "name": "datenbank", "host": "10.0.0.5", "port": 5432},
    {"art": "platte", "name": "platte", "pfad": "/", "warnung": 80, "kritisch": 90},
    {"art": "speicher", "name": "ram", "warnung": 85, "kritisch": 95},
    {"art": "prozess", "name": "worker", "teil": "worker.py"}
  ],
  "mail": {"host": "mail.example.org", "port": 587, "modus": "starttls",
           "absender": "healthcheck@example.org", "empfaenger": ["bereitschaft@example.org"],
           "benutzer": "healthcheck", "passwort_env": "HEALTH_MAILPASS"},
  "webhook": {"url_env": "HEALTH_WEBHOOK", "art": "slack"}
}
```

Die Datei zeigt, wie sich die Idee der Registratur aus 12.3 auszahlt: Jeder Eintrag in `checks` nennt die Art und die Parameter dieser Art; die Namen `passwort_env` und `url_env` verweisen auf Umgebungsvariablen. Die Datei selbst enthält kein Geheimnis und darf deshalb in die Versionsverwaltung. Die Beispiele und Adressen sind frei erfunden (die Domäne `example.org` ist für Beispiele reserviert).

**Der Gesamttest.** Das Testprogramm startet den Webserver, einen Mailserver mit STARTTLS und Anmeldung und den Webhook-Empfänger, schreibt eine Konfiguration und ruft `main` mehrfach auf. Die Uhr läuft pro Aufruf eine Minute weiter. Das Programm spielt eine kleine Geschichte durch: Der Webserver ist gesund, fällt aus, kommt wieder und fällt ein zweites Mal aus, und zwischendurch läuft ein Trockenlauf und ein Lauf mit fehlendem Passwort. Nach jedem Lauf zeigt die Ausgabe Exit-Code, Zahl der empfangenen Mails und Webhooks und die Ausgabezeilen; die Ports sind durch das Wort PORT ersetzt, weil sie bei jedem Lauf anders sind.

```python
import io
import json
import re
import tempfile
from contextlib import redirect_stdout
from pathlib import Path

import testhook
import testsmtp
import testweb
from healthcheck import main

web = testweb.Web()
mailserver = testsmtp.Server("starttls", benutzer="monitor", passwort="geheim")
hook = testhook.Empfaenger()
arbeit = Path(tempfile.mkdtemp())

konfig = {
    "host": "web01",
    "zustand": str(arbeit / "zustand.json"),
    "schwelle": 2,
    "erinnerung": 3600,
    "checks": [
        {"art": "http", "name": "web", "url": web.url + "/", "timeout": 2},
        {"art": "tcp", "name": "webport", "host": "127.0.0.1", "port": web.port},
    ],
    "mail": {"host": "127.0.0.1", "port": mailserver.port, "modus": "starttls", "absender": "monitoring@beispiel.example",
             "empfaenger": ["admin@beispiel.example"], "benutzer": "monitor", "passwort_env": "HEALTH_MAILPASS",
             "cafile": str(mailserver.zertifikat)},
    "webhook": {"url_env": "HEALTH_WEBHOOK", "art": "slack"},
}
datei = arbeit / "konfig.json"
datei.write_text(json.dumps(konfig), encoding="utf-8")
umgebung = {"HEALTH_MAILPASS": "geheim", "HEALTH_WEBHOOK": hook.url}
uhr = iter(range(1000, 100000, 60))                       # jeder Lauf eine Minute später


def lauf(*zusatz, umgebung=umgebung):
    puffer = io.StringIO()
    with redirect_stdout(puffer):
        code = main(["--konfig", str(datei), *zusatz], umgebung=umgebung, jetzt=lambda: next(uhr))
    zeilen = [re.sub(r"127\.0\.0\.1:\d+", "127.0.0.1:PORT", z) for z in puffer.getvalue().splitlines()]
    return code, zeilen


def bericht(titel, code, zeilen):
    print(f"--- {titel}: Exit-Code {code}, Mails {len(mailserver.sammler.nachrichten)}, Webhooks {len(hook.empfangen)}")
    for zeile in zeilen:
        print("   ", zeile)


bericht("1. Lauf, alles gesund", *lauf())
web.status = 500
bericht("2. Lauf, erster Fehler", *lauf())
bericht("3. Lauf, zweiter Fehler", *lauf())
bericht("4. Lauf, immer noch kaputt", *lauf())
web.status = 200
bericht("5. Lauf, wieder gesund", *lauf())
web.status = 500
bericht("6. Lauf, erster Fehler", *lauf())
bericht("7. Trockenlauf, zweiter Fehler", *lauf("--trockenlauf"))
bericht("8. Lauf, zweiter Fehler", *lauf())
bericht("9. Passwort fehlt in der Umgebung", *lauf(umgebung={"HEALTH_WEBHOOK": hook.url}))

web.stoppe()
mailserver.stoppe()
hook.stoppe()
```

```text
--- 1. Lauf, alles gesund: Exit-Code 0, Mails 0, Webhooks 0
    OK         web        http://127.0.0.1:PORT/: Status 200
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 2. Lauf, erster Fehler: Exit-Code 2, Mails 0, Webhooks 0
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 3. Lauf, zweiter Fehler: Exit-Code 2, Mails 1, Webhooks 1
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    MELDUNG: [ALARM] web auf web01: KRITISCH
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 4. Lauf, immer noch kaputt: Exit-Code 2, Mails 1, Webhooks 1
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 5. Lauf, wieder gesund: Exit-Code 0, Mails 2, Webhooks 2
    OK         web        http://127.0.0.1:PORT/: Status 200
    MELDUNG: [ERHOLT] web auf web01: OK
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 6. Lauf, erster Fehler: Exit-Code 2, Mails 2, Webhooks 2
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 7. Trockenlauf, zweiter Fehler: Exit-Code 2, Mails 2, Webhooks 2
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    MELDUNG: [ALARM] web auf web01: KRITISCH
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 8. Lauf, zweiter Fehler: Exit-Code 2, Mails 3, Webhooks 3
    KRITISCH   web        http://127.0.0.1:PORT/: Status 500 statt 200
    MELDUNG: [ALARM] web auf web01: KRITISCH
    OK         webport    127.0.0.1:PORT nimmt Verbindungen an
--- 9. Passwort fehlt in der Umgebung: Exit-Code 4, Mails 3, Webhooks 3
```

Gehen wir die Geschichte durch. Im ersten Lauf ist alles gesund (Exit-Code 0), nichts wird gesendet. Im zweiten fällt der Webserver aus: Die Prüfung `web` ist `KRITISCH`, der Exit-Code ist 2, aber es ist erst der erste Fehler, daher keine Meldung (die Schwelle ist zwei). Die Prüfung `webport` bleibt `OK`, weil der Server noch Verbindungen annimmt, nur mit dem Statuscode 500 antwortet; hier sieht man den Wert der Prüfung von außen. Im dritten Lauf ist es der zweite Fehler: Der Alarm geht als Mail und als Webhook hinaus, die Zählung springt auf eine Mail und einen Webhook. Der vierte Lauf zeigt, dass die Störung nicht erneut gemeldet wird. Im fünften ist der Webserver wieder gesund: Die Entwarnung kommt auf beiden Kanälen, und der Exit-Code ist 0. Der sechste Lauf ist wieder ein erster Fehler (still), im siebten, einem Trockenlauf, erscheint die Meldung „\[ALARM\]“ nur auf dem Bildschirm, und die Zähler bleiben gleich, ohne dass der Zustand gespeichert wird. Deshalb kommt der Alarm im achten, echten Lauf tatsächlich (dritte Mail, dritter Webhook). Im neunten Lauf fehlt die Umgebungsvariable mit dem Passwort: Das Programm endet mit dem Exit-Code 4 und einer Fehlermeldung im Protokoll, ohne dass irgendetwas geprüft oder gesendet wird.

**In den Betrieb nehmen.** Zeitgesteuert starten Sie das Programm wie in Modul 9 gelernt. Mit systemd besteht der Betrieb aus zwei Dateien, einer *Service*-Unit, die das Programm einmal ausführt, und einer *Timer*-Unit, die den Takt vorgibt. Die Geheimnisse stehen in einer Datei, die nur der Dienstbenutzer lesen darf (zum Beispiel Rechte `0600`) und die der Service als `EnvironmentFile` lädt; so landen sie weder im Quelltext noch in der Konfiguration, noch in der Befehlszeile.

```ini
# /etc/systemd/system/healthcheck.service
[Unit]
Description=Health-Check der Webanwendung

[Service]
Type=oneshot
User=monitor
EnvironmentFile=/etc/healthcheck/geheimnisse.env
ExecStart=/usr/bin/python3 /opt/healthcheck/healthcheck.py --konfig /etc/healthcheck/konfig.json

# /etc/systemd/system/healthcheck.timer
[Unit]
Description=Health-Check alle fünf Minuten

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
AccuracySec=30s

[Install]
WantedBy=timers.target
```

Die beiden Dateien wurden mit `systemd-analyze verify` auf Syntaxfehler geprüft (dabei wird nicht geprüft, ob der Benutzer `monitor` und die genannten Dateien auf Ihrem System vorhanden sind). Mit `systemctl enable --now healthcheck.timer` wird der Takt aktiviert; `systemctl list-timers` zeigt den nächsten Lauf, und `journalctl -u healthcheck` zeigt die Ausgabe der Läufe. Ein Lauf, der mit einem Exit-Code ungleich null endet, färbt die Unit in `systemctl status` rot; das kann man zusätzlich zu den Meldungen als Hinweis nutzen. Mit cron wäre die Zeile `*/5 * * * *` und der Aufruf des Programms mit dem Pfad zur Konfiguration das Gegenstück (siehe Modul 9, 9.2 und die Hinweise zur Umgebung von cron, die dort besprochen wurden: Variablen, die in Ihrer Shell gesetzt sind, gibt es im cron-Lauf nicht).

**Typische Fehler und Stolperfallen.** Der häufigste Fehler beim Einrichten ist, den Alarm *nie auszuprobieren*: Man baut die Überwachung, sie läuft, und erst bei der ersten echten Störung stellt sich heraus, dass die Mail im Spamordner landet oder der Webhook ungültig ist. Der zweite ist, Geheimnisse in der Konfigurationsdatei abzulegen „nur für den Test“ und zu vergessen. Der dritte ist ein Prüfintervall, das kürzer als die Laufzeit des Programms ist (Läufe überschneiden sich; siehe Modul 9, 9.4 zur Sperrung). Der vierte, die Zustandsdatei in ein Verzeichnis zu legen, in das der Dienstbenutzer nicht schreiben darf. Das Gegenüber:

```text
# FALSCH: Geheimnis in der Konfiguration, Alarm nie getestet, Zustand in /etc
{
  "zustand": "/etc/healthcheck/zustand.json",
  "mail": {"benutzer": "healthcheck", "passwort": "Sommer2026!", ...}
}

# RICHTIG: Name der Umgebungsvariablen in der Konfiguration, Geheimnis im EnvironmentFile (0600),
#          Zustand im Datenverzeichnis des Dienstes, Testalarm gleich bei der Einrichtung
{
  "zustand": "/var/lib/healthcheck/zustand.json",
  "mail": {"benutzer": "healthcheck", "passwort_env": "HEALTH_MAILPASS", ...}
}
#   Testalarm: Prüfung auf einen absichtlich falschen Port konfigurieren,
#   zweimal laufen lassen, Mail und Webhook kontrollieren, danach zurückstellen.
```

Dass die Konfigurationsdatei hier als Listing erscheint, ist eine Darstellung; sie enthält Auslassungspunkte und ist kein gültiges JSON. Entscheidend ist der Unterschied: Das Passwort steht in der falschen Variante in der Datei, in der richtigen steht nur der Name der Variablen, und der Testalarm gehört zur Einrichtung.

**Themenwahl für das Abschlussprojekt.** Mit diesem Modul endet der inhaltliche Teil des Kurses. In Modul 13 bis 15 folgen Git, Projektstruktur, Tests und zum Schluss ein Abschlussprojekt aus Ihrem Alltag, das Sie selbst wählen. Heute ist der Tag, das Thema zu wählen, denn Planung und Umsetzung beginnen in Modul 15 und sollen mit klaren Zielen starten. Ein gutes Abschlussprojekt erfüllt drei Bedingungen: Es löst ein *echtes Problem*, das Sie kennen (dann wissen Sie, wann es fertig ist und woran man es testet), es lässt sich in *fünf Lerntagen* fertigstellen (das heißt: klein anfangen), und es benutzt *mehrere Bausteine* des Kurses (Dateien, Fehlerbehandlung, Konfiguration, Protokollierung, Tests).

Zur Orientierung einige Vorschläge, die an die Module dieses Kurses anschließen. Ein *Backup-Werkzeug* mit Rotation, Prüfsummen und Bericht per Mail (Module 6, 9, 12). Ein *Log-Auswerter*, der Anmeldeversuche, Fehlermeldungen oder Zugriffe zählt und einen Bericht schreibt (Modul 8). Ein *Inventar-Werkzeug*, das per SSH Angaben von mehreren Servern sammelt und eine Tabelle oder Webseite erzeugt (Modul 11). Ein *Aufräum-Werkzeug*, das alte Dateien nach Regeln findet, meldet und mit Bestätigung löscht (Modul 5, 6). Eine *Erweiterung des Health-Checks* um weitere Prüfarten, einen Tagesbericht oder eine Statusseite. Ein *Bereitstellungsskript*, das Dateien auf Server verteilt und Dienste neu startet (Modul 11). Oder ein *Berichtswerkzeug*, das Daten aus einer API holt und aufbereitet (Modul 10). Alles, was in Ihrem Berufsalltag regelmäßig von Hand geschieht, ist ein guter Kandidat.

**Vom Thema zum Plan.** Aus der Idee wird ein Plan, indem man das Ziel in drei Stufen aufteilt: was das Projekt *mindestens* können muss (der kleinste nützliche Kern, in der Fachsprache *Minimum Viable Product*), was es *möglichst* können soll und was es *vielleicht* können kann, wenn Zeit bleibt. Fertig sein bedeutet, die erste Stufe sauber geliefert zu haben, mit Tests, Protokollierung, Fehlerbehandlung, einer Konfigurationsdatei und einer kurzen Beschreibung. Das ist mehr wert als eine halbe Umsetzung aller drei Stufen. Eine einfache Vorlage für den Steckbrief, den Sie zu Beginn von Modul 15 brauchen:

```text
Projektsteckbrief
=================
Titel:          Backup-Werkzeug mit Mailbericht
Problem:        Ich sichere jeden Freitag Ordner von Hand und vergesse es manchmal.
Muss (Stufe 1): Konfigurierte Ordner als tar.gz sichern, alte Sicherungen löschen (letzte 7 behalten),
                Exit-Code 0/1, Protokolldatei.
Soll (Stufe 2): Prüfsumme je Sicherung, Mailbericht mit Größe und Dauer, Trockenlauf.
Kann (Stufe 3): Sicherung per SSH auf einen zweiten Rechner kopieren.
Bausteine:      pathlib, tarfile, logging, argparse, JSON-Konfiguration, smtplib, pytest.
Test:           Sicherung eines Beispielordners, Rotation mit künstlichen Zeitstempeln,
                Fehlerfall "Zielordner nicht beschreibbar".
Risiken:        Löschen darf nie die einzige Kopie treffen (Trockenlauf und Bestätigung).
```

Der Steckbrief ist Text und kein Programm. Er zwingt dazu, *vor* dem ersten Code zu entscheiden, was fertig heißt und woran man es prüft. Gut ist auch, das *Risiko* zu benennen: Bei Werkzeugen, die Dateien löschen oder Server verändern, ist das die Frage, was im schlimmsten Fall passiert, und die Antwort lautet fast immer: Es muss einen Trockenlauf geben.

**Quiz.** Das Quiz zum Modul steht in den Übungsaufgaben 1 bis 4; die Fragen, die Sie ohne Computer beantworten können, finden Sie dort mit Lösungen. Prüfen Sie sich zunächst selbst: Was unterscheidet Umschlag und Kopfzeilen einer Mail? Warum verlangt man bei STARTTLS ein geprüftes Zertifikat? Welche Statuscodes lohnen eine Wiederholung beim Webhook und welche nicht? Warum meldet ein guter Health-Check nicht bei jedem Lauf? Wozu dient ein Trockenlauf?

> **Ergänzung aus der Nachrecherche**
>
> **Was nicht geprüft werden konnte.** Alle Beispiele laufen gegen lokale Testserver. Nicht getestet wurden echte Mailserver (Zustellung, Spamprüfung, Anmeldeverfahren der Anbieter), echte Chat-Dienste (Slack, Teams) und der Betrieb als echter systemd-Dienst oder cron-Auftrag über längere Zeit; die Unit-Dateien wurden nur mit `systemd-analyze verify` auf ihre Syntax geprüft. Der Ablauf bei einem Ausfall des Rechners, auf dem der Health-Check selbst läuft, ist nicht Teil des Programms (siehe 12.4, Totmannschalter). Die Vorschläge zum Abschlussprojekt und die Planungsvorlage sind didaktische Empfehlungen, keine geprüften Normen.

> **Merke**
>
> Das fertige Programm liest Konfiguration (Daten statt Code), holt Geheimnisse aus der Umgebung, prüft, entscheidet, meldet auf allen Kanälen, speichert den Zustand atomar und endet mit einem auswertbaren Exit-Code. Früh scheitern bei Konfigurationsfehlern, Trockenlauf zum gefahrlosen Ausprobieren, Alarm bei der Einrichtung testen. Hauptfunktion mit Argumenten, Umgebung und Uhr als Parametern ist testbar. Abschlussprojekt: echtes Problem, klein anfangen (Muss, Soll, Kann), Steckbrief mit Test und Risiko.

> **Praxisbeispiel**
>
> Programme dieser Art laufen in vielen kleinen Firmen jahrelang unauffällig, und gerade deshalb muss man sie pflegen: Ändern sich Adressen oder Passwörter, muss die Konfiguration mit, und ein Testalarm nach jeder Änderung ist Pflicht. Wer das Programm an Kollegen übergibt, gibt außer dem Code die Konfigurationsvorlage, die Beschreibung der Umgebungsvariablen und eine Anleitung für den Alarmfall mit.

## Übungsaufgaben zu Modul 12

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Für die Aufgaben 6 bis 8 brauchen Sie die Hilfsmodule `testhook` (12.2), `testsmtp` (12.1), `hooks` (12.2) und `checks` (12.3) im selben Ordner.

**Aufgabe 1.** Quiz zu E-Mail: Was ist der Unterschied zwischen dem *Umschlag* und den *Kopfzeilen* einer Mail, und wo steht ein `Bcc`-Empfänger? Worin unterscheiden sich STARTTLS und implizites TLS, und welche Klasse von `smtplib` gehört zu welchem Weg? Warum nützt eine verschlüsselte Verbindung ohne Prüfung des Zertifikats wenig? Was ist eine Kopfzeilen-Einschleusung, und wie schützt `EmailMessage` davor?

**Aufgabe 2.** Quiz zu Webhooks und Alerting: Welche der Statuscodes 400, 404, 429 und 503 sollte ein Programm beim Senden eines Webhooks wiederholen, und warum? Warum ist die URL eines eingehenden Webhooks ein Geheimnis? Wie funktioniert eine HMAC-Signatur, und warum vergleicht man mit `hmac.compare_digest` statt mit `==`? Was bedeutet „Alarmmüdigkeit“, und welche drei Maßnahmen aus diesem Modul wirken dagegen?

**Aufgabe 3.** Finden Sie mindestens sieben Fehler oder Probleme im folgenden Programm und erklären Sie jeden.

```python
import smtplib
import requests
import psutil

WEBHOOK = "https://hooks.slack.com/services/T000/B000/abcdef"
PASSWORT = "Sommer2026!"


def alarm(host, text):
    nachricht = "From: monitor@example.org\nTo: admin@example.org\nSubject: Alarm " + host + "\n\n" + text
    smtp = smtplib.SMTP("mail.example.org", 587)
    smtp.login("monitor", PASSWORT)
    smtp.sendmail("monitor@example.org", ["admin@example.org"], nachricht)
    requests.post(WEBHOOK, data={"text": text})
    print("Alarm gesendet an", WEBHOOK)


while True:
    if psutil.disk_usage("/").percent > 90:
        alarm("web01", "Platte voll")
```

**Aufgabe 4.** Ein Health-Check läuft alle 5 Minuten. Beantworten Sie: (a) Wie lange dauert es bei der Schwelle 3 höchstens, bis ein Dauerausfall gemeldet wird, und wie lange mindestens? (b) Eine Prüfung ist nachts jede Nacht für drei Minuten gestört (Datensicherung). Wie vermeiden Sie den Alarm, ohne echte Ausfälle zu verpassen? (c) Ein Mitarbeiter schlägt vor, den Alarm bei jedem Lauf zu wiederholen, „damit niemand ihn vergisst“. Was halten Sie davon, und was ist die bessere Lösung? (d) Wer merkt, wenn der Server, auf dem der Health-Check läuft, ausfällt?

**Aufgabe 5.** **Code:** Schreiben Sie eine Funktion `tagesbericht_mail(absender, empfaenger, zeilen, anhang)`, die eine Mail mit Text- und HTML-Fassung (Aufzählung) und einem Textanhang baut. Der Betreff nennt die Zahl der Zeilen und enthält einen Umlaut. Prüfen Sie die Struktur der Nachricht, ohne sie zu senden.

**Aufgabe 6.** **Code:** Slack und Teams begrenzen die Größe von Nachrichten. Schreiben Sie eine Funktion `sende_gesammelt(url, meldungen, max_zeichen)`, die viele Meldungen zu möglichst wenigen Nachrichten zusammenfasst, keine Nachricht über die Grenze wachsen lässt, keine Meldung verliert und die Zahl der Nachrichten liefert. Testen Sie gegen den Testempfänger.

**Aufgabe 7.** **Code:** Schreiben Sie eine Prüfung `check_zertifikat(name, host, port, cafile, warnung_tage, kritisch_tage)`, die die Restlaufzeit des TLS-Zertifikats eines Servers in Tagen bestimmt und in eine Stufe übersetzt. Ein ungültiges Zertifikat ist `KRITISCH`. Testen Sie die Funktion gegen den Mailserver im Modus `"ssl"` (sein Zertifikat gilt einen Tag).

**Aufgabe 8.** **Code:** Alarme sollen in Wartungsfenstern (zum Beispiel samstags von 02:00 bis 04:00 Uhr) unterdrückt werden, Entwarnungen nie. Schreiben Sie dafür `in_wartung(jetzt, fenster)` und `filtere_ereignis(ereignis, jetzt, fenster)`. Prüfen Sie die Grenzen der Fenster (Beginn eingeschlossen, Ende ausgeschlossen) und einen Wochentag, an dem kein Fenster liegt.

## Lösungen zu Modul 12

**Aufgabe 1.** Der *Umschlag* (Absender und Empfänger, die dem Server mit `MAIL FROM` und `RCPT TO` genannt werden) bestimmt die Zustellung; die *Kopfzeilen* (`From`, `To`, `Cc`, `Subject`) gehören zur Nachricht und werden vom Mailprogramm des Empfängers angezeigt. Beide sind unabhängig voneinander. Ein `Bcc`-Empfänger steht im Umschlag, aber nicht in den Kopfzeilen der gelieferten Nachricht; `send_message` übernimmt ihn daher in den Umschlag und lässt die `Bcc`-Zeile weg (so zeigte es 12.1). Bei STARTTLS verbindet man sich zunächst unverschlüsselt (Port 587) und stellt die Verbindung mit dem Befehl `STARTTLS` auf TLS um (`smtplib.SMTP` mit `starttls()`); beim impliziten TLS (Port 465) ist die Verbindung von Anfang an verschlüsselt (`smtplib.SMTP_SSL`). Eine Verschlüsselung ohne Zertifikatsprüfung schützt nur vor passivem Mithören, nicht davor, dass sich ein Angreifer als Server ausgibt und verschlüsselt mit Ihnen spricht (und dabei Passwort und Mail erhält); erst die Prüfung stellt sicher, dass der Schlüssel zum erwarteten Rechnernamen gehört. Bei einer Kopfzeilen-Einschleusung nutzt ein Angreifer Zeilenumbrüche in einem Wert, der in eine Kopfzeile gelangt (etwa im Betreff), um eigene Kopfzeilen oder sogar Inhalt anzuhängen, etwa heimliche Empfänger. `EmailMessage` lehnt Zeilenumbrüche in Kopfzeilenwerten mit einem `ValueError` ab, wie das Beispiel in 12.1 zeigte.

**Aufgabe 2.** Wiederholt werden sollten `429` (zu viele Anfragen; nach einer Pause, am besten der von `Retry-After` genannten) und `503` (der Dienst ist vorübergehend überlastet oder in Wartung): Das sind vorübergehende Zustände. Nicht wiederholt werden `400` (der Inhalt ist fehlerhaft, ein zweiter Versuch ist es auch) und `404` (die URL stimmt nicht, oder der Webhook wurde entfernt); hier muss ein Mensch etwas ändern. Die URL eines eingehenden Webhooks ist die einzige Zugangsberechtigung: Wer sie kennt, kann Nachrichten in den Kanal senden (und etwa falsche Alarme oder Betrugsnachrichten einschleusen); deshalb gehört sie nicht in den Quelltext oder in Protokolle, und bei Offenlegung wird der Webhook erneuert. Eine HMAC-Signatur ist ein mit einem gemeinsamen Geheimnis berechneter Prüfwert über den Körper der Nachricht; der Empfänger berechnet ihn selbst und vergleicht, und bei Übereinstimmung stammt die Nachricht vom Besitzer des Geheimnisses und wurde nicht verändert. `hmac.compare_digest` vergleicht in einer Zeit, die nicht davon abhängt, wo sich die Werte unterscheiden; ein Vergleich mit `==` bricht beim ersten Unterschied ab, und ein Angreifer kann aus der Antwortzeit auf richtige Anfangsstücke schließen. Alarmmüdigkeit entsteht, wenn so viele Meldungen kommen, dass Menschen sie ignorieren und die wichtige übersehen. Dagegen helfen die Schwelle (erst nach mehreren Fehlschlägen in Folge melden), das Melden von Veränderungen statt von Zuständen (kein Dauerfeuer, Entwarnung) und eine Erinnerungsfrist; ergänzend Wartungsfenster und das Zusammenfassen von Meldungen.

**Aufgabe 3.** (1) Der Webhook und das Passwort stehen im Quelltext (Geheimnisse gehören in die Umgebung oder in eine geschützte Datei). (2) Die Mail wird aus einer Zeichenkette gebaut: `host` und `text` können Zeilenumbrüche enthalten und Kopfzeilen einschleusen, Umlaute im Betreff sind nicht korrekt kodiert; richtig ist `EmailMessage`. (3) Es gibt kein `starttls()` und keinen Kontext: Das Passwort geht im Klartext über das Netz, und ohne geprüftes Zertifikat ist die Verbindung nicht abgesichert. (4) `SMTP(...)` hat kein Zeitlimit und wird nicht mit `with` geöffnet oder mit `quit()` beendet. (5) Es gibt keine Fehlerbehandlung: Fällt der Mailserver aus, bricht das Programm ab, und der Webhook kommt nicht mehr dran. (6) `requests.post` verwendet `data=`, das ein Formular sendet; ein JSON-Webhook braucht `json=`. (7) Es gibt kein Zeitlimit und keine Prüfung des Statuscodes: Eine Ablehnung (404) bleibt unbemerkt. (8) Die geheime URL wird ausgegeben. (9) Die Endlosschleife ohne Pause fragt die Platte ununterbrochen ab und feuert bei einer vollen Platte *unaufhörlich* Alarme: Dauerfeuer ohne Zustand, ohne Schwelle, ohne Entwarnung und ohne Erinnerungsfrist. (10) Das Programm gehört zeitgesteuert gestartet (cron oder systemd-Timer, Modul 9), nicht als Dauerläufer ohne Ausweg. (11) Schwelle und Rechnername sind fest eingebaut statt in einer Konfiguration.

**Aufgabe 4.** (a) Bei einem Dauerausfall und der Schwelle drei sind drei Fehlläufe in Folge nötig. Fällt der Dienst direkt vor einem Lauf aus, wird dieser Lauf der erste Fehlschlag; der Alarm kommt mit dem dritten Fehlschlag, also nach 10 Minuten (zwei weitere Läufe im Abstand von 5 Minuten). Fällt der Dienst kurz nach einem Lauf aus, vergehen bis zum ersten Fehlschlag knapp 5 Minuten, und der Alarm kommt nach knapp 15 Minuten. Mindestens also etwa 10, höchstens knapp 15 Minuten nach Beginn der Störung (plus die Laufzeit der Prüfung und der Zustellung). (b) Entweder ein *Wartungsfenster*, in dem Alarme unterdrückt werden (Aufgabe 8), oder eine Schwelle, die über der Dauer der bekannten Störung liegt (bei einer Störung von drei Minuten und einem Intervall von fünf Minuten genügt die Schwelle zwei, denn die Störung zeigt sich höchstens in einem Lauf). Das Fenster ist die ehrlichere Lösung, weil es die Absicht ausdrückt; die Schwelle verzögert alle Alarme dieser Prüfung. Entwarnungen und Alarme *außerhalb* des Fensters bleiben erhalten. (c) Das ist Dauerfeuer, das zu Alarmmüdigkeit führt; wichtig ist dennoch, dass eine lange bestehende Störung nicht vergessen wird. Die bessere Lösung ist eine *Erinnerung nach einer Frist* (zum Beispiel stündlich), gegebenenfalls mit höherer Dringlichkeit oder einem weiteren Empfänger (Eskalation). (d) Niemand, wenn keine zweite Stelle prüft: Ein ausgefallener Health-Check sendet keine Alarme, und die Stille wird als „alles in Ordnung“ gedeutet. Abhilfe ist ein Totmannschalter (ein zweiter Dienst, der alarmiert, wenn die regelmäßigen Lebenszeichen ausbleiben) oder ein zweiter Rechner, der den ersten prüft.

**Aufgabe 5.** Die Funktion legt mit `set_content` die Textfassung an, ergänzt mit `add_alternative` die HTML-Fassung und mit `add_attachment` den Anhang. Die Nachricht ist danach `multipart/mixed`, mit einem `multipart/alternative` aus Text und HTML sowie dem Anhang. Der Betreff mit dem Umlaut wird von `EmailMessage` automatisch korrekt kodiert. Die Textfassung findet man mit `get_body(preferencelist=("plain",))`.

```python
import tempfile
from email.message import EmailMessage
from pathlib import Path


def tagesbericht_mail(absender, empfaenger, zeilen, anhang):
    """Baut eine Mail mit Text- und HTML-Fassung und einem Anhang; Umlaute im Betreff sind erlaubt."""
    nachricht = EmailMessage()
    nachricht["From"] = absender
    nachricht["To"] = ", ".join(empfaenger)
    nachricht["Subject"] = f"Tagesbericht: {len(zeilen)} Prüfungen"
    nachricht.set_content("\n".join(zeilen) + "\n")
    punkte = "".join(f"<li>{zeile}</li>" for zeile in zeilen)
    nachricht.add_alternative(f"<h1>Tagesbericht</h1><ul>{punkte}</ul>", subtype="html")
    pfad = Path(anhang)
    nachricht.add_attachment(pfad.read_bytes(), maintype="text", subtype="plain", filename=pfad.name)
    return nachricht


protokoll = Path(tempfile.mkdtemp()) / "protokoll.txt"
protokoll.write_text("Lauf 1: ok\nLauf 2: ok\n", encoding="utf-8")
mail = tagesbericht_mail("monitoring@beispiel.example", ["a@beispiel.example", "b@beispiel.example"],
                         ["web: OK", "db: WARNUNG"], protokoll)

print("Betreff     :", mail["Subject"])
print("Empfänger   :", mail["To"])
print("Teile       :", [teil.get_content_type() for teil in mail.walk()])
print("Anhang      :", [teil.get_filename() for teil in mail.iter_attachments()])
print("Textfassung :", mail.get_body(preferencelist=("plain",)).get_content().splitlines())
```

```text
Betreff     : Tagesbericht: 2 Prüfungen
Empfänger   : a@beispiel.example, b@beispiel.example
Teile       : ['multipart/mixed', 'multipart/alternative', 'text/plain', 'text/html', 'text/plain']
Anhang      : ['protokoll.txt']
Textfassung : ['web: OK', 'db: WARNUNG']
```

Das Ergebnis hat zwei Empfänger in einer Kopfzeile (mit Komma getrennt), die Teile in der erwarteten Reihenfolge, einen Anhang namens `protokoll.txt` und die Textfassung als Zeilenliste. Ein häufiger Fehler bei dieser Aufgabe ist die Reihenfolge: `add_alternative` setzt voraus, dass schon ein Inhalt mit `set_content` vorhanden ist, und `add_attachment` muss danach stehen, damit der Anhang als eigener Teil neben der Alternative landet.

**Aufgabe 6.** Die Funktion sammelt die Zeilen in einer Zeichenkette, solange die Grenze nicht überschritten wird, beginnt dann ein neues Paket und sendet am Ende alle Pakete. Eine einzelne Meldung, die für sich schon zu lang wäre, wird gekürzt, damit die Grenze unter allen Umständen hält. Die Zahl der Pakete wird zurückgegeben; eine leere Liste ergibt null Pakete.

```python
import testhook
from hooks import sende_webhook, slack_nachricht


def sende_gesammelt(url, meldungen, max_zeichen=100):
    """Fasst viele Meldungen zu möglichst wenigen Nachrichten zusammen; liefert die Zahl der Nachrichten."""
    pakete, aktuell = [], ""
    for meldung in meldungen:
        meldung = meldung[:max_zeichen]                              # eine einzelne Zeile darf nie zu lang sein
        if aktuell and len(aktuell) + 1 + len(meldung) > max_zeichen:
            pakete.append(aktuell)
            aktuell = ""
        aktuell = f"{aktuell}\n{meldung}" if aktuell else meldung
    if aktuell:
        pakete.append(aktuell)
    for paket in pakete:
        sende_webhook(url, slack_nachricht(paket), versuche=1)
    return len(pakete)


server = testhook.Empfaenger()
meldungen = [f"[ALARM] check{nummer:02d} auf web01: KRITISCH" for nummer in range(1, 8)]
anzahl = sende_gesammelt(server.url, meldungen)
print("Nachrichten gesendet:", anzahl, "| beim Server angekommen:", len(server.empfangen))
print("Keine Nachricht zu lang:", all(len(e["nutzlast"]["text"]) <= 100 for e in server.empfangen))
alle = "\n".join(e["nutzlast"]["text"] for e in server.empfangen).splitlines()
print("Keine Meldung verloren:", alle == meldungen)
print("Leere Liste sendet nichts:", sende_gesammelt(server.url, []) == 0)
server.stoppe()
```

```text
Nachrichten gesendet: 4 | beim Server angekommen: 4
Keine Nachricht zu lang: True
Keine Meldung verloren: True
Leere Liste sendet nichts: True
```

Die sieben Meldungen mit je 37 Zeichen ergeben bei der Grenze von 100 Zeichen vier Nachrichten (zwei, zwei, zwei, eins); die Probe zeigt, dass keine zu lang ist, nichts verloren geht und in der richtigen Reihenfolge ankommt. Bei einer Grenze unterhalb von 75 Zeichen würde jede Meldung allein gesendet. Eine Verbesserung wäre, die Meldungen bei Kürzung mit einem Hinweis zu kennzeichnen („…“) und bei einer sehr großen Zahl nur noch eine Zusammenfassung zu senden, etwa „43 weitere Meldungen“.

**Aufgabe 7.** Die Funktion baut eine TLS-Verbindung mit einem Kontext auf, der die eigene Zertifizierungsstelle kennt (`cafile`), liest mit `getpeercert()` das Ablaufdatum aus (`notAfter`), wandelt es mit `ssl.cert_time_to_seconds` in Sekunden um und rechnet die Differenz zur aktuellen Zeit in Tage. Ein Zertifikat, das nicht geprüft werden kann (hier ein selbst ausgestelltes, ohne `cafile`), löst `SSLCertVerificationError` aus, und das Programm meldet `KRITISCH` mit der Begründung aus `verify_message`. Für einen nicht erreichbaren Dienst gilt dasselbe, mit einem anderen Text. Wichtig: Der Servername wird beim Aufbau mit `server_hostname` übergeben, damit die Prüfung des Rechnernamens funktioniert.

```python
import socket
import ssl
import time

import testsmtp
from checks import KRITISCH, OK, WARNUNG, Ergebnis


def check_zertifikat(name, host, port, cafile=None, warnung_tage=21, kritisch_tage=7, timeout=5):
    """Prüft, wie viele Tage ein TLS-Zertifikat noch gültig ist."""
    kontext = ssl.create_default_context(cafile=cafile)
    try:
        with socket.create_connection((host, port), timeout=timeout) as roh:
            with kontext.wrap_socket(roh, server_hostname=host) as tls:
                ablauf = ssl.cert_time_to_seconds(tls.getpeercert()["notAfter"])
    except ssl.SSLCertVerificationError as fehler:
        return Ergebnis(name, KRITISCH, f"Zertifikat ungültig: {fehler.verify_message}")
    except OSError as fehler:
        return Ergebnis(name, KRITISCH, f"{host}:{port} nicht erreichbar ({type(fehler).__name__})")
    tage = (ablauf - time.time()) / 86400
    status = KRITISCH if tage < kritisch_tage else WARNUNG if tage < warnung_tage else OK
    return Ergebnis(name, status, f"Zertifikat noch {tage:.0f} Tage gültig", tage)


# Der Testmailserver im Modus "ssl" benutzt ein Zertifikat, das einen Tag gültig ist
server = testsmtp.Server("ssl")
cafile = str(server.zertifikat)
print("Strenge Schwellen    :", check_zertifikat("tls", "127.0.0.1", server.port, cafile, 21, 7).status)
print("Lockere Schwellen    :", check_zertifikat("tls", "127.0.0.1", server.port, cafile, 0, 0).status)
r = check_zertifikat("tls", "127.0.0.1", server.port, cafile, 0, 0)
print("Restlaufzeit in Tagen:", 0 < r.wert <= 1)
r = check_zertifikat("tls", "127.0.0.1", server.port)               # Zertifizierungsstelle unbekannt
print("Unbekannte Stelle    :", r.status, "|", r.meldung)
print("Kein Dienst          :", check_zertifikat("tls", "127.0.0.1", 1, cafile).status)
server.stoppe()
```

```text
Strenge Schwellen    : KRITISCH
Lockere Schwellen    : OK
Restlaufzeit in Tagen: True
Unbekannte Stelle    : KRITISCH | Zertifikat ungültig: self-signed certificate
Kein Dienst          : KRITISCH
```

Das Zertifikat des Testservers gilt einen Tag. Mit den Schwellen 21 und 7 Tage ist das `KRITISCH`; mit den Schwellen 0 und 0 ist es `OK`. Die Restlaufzeit liegt zwischen null und einem Tag. Ohne `cafile` (also mit den Standardstellen des Systems) kennt der Kontext das Testzertifikat nicht, und die Prüfung meldet `KRITISCH` mit der Begründung „self-signed certificate“; der letzte Fall (Port ohne Dienst) endet ebenfalls mit `KRITISCH`. In der Praxis würde man die Prüfung in `ARTEN` eintragen und mit Schwellen von etwa 21 und 7 Tagen betreiben.

**Aufgabe 8.** Die Funktion ermittelt den Wochentag mit `jetzt.weekday()` (Montag ist 0) und die Uhrzeit als Zeichenkette im Format `HH:MM`; weil sie mit führenden Nullen immer gleich lang ist, lassen sich Uhrzeiten als Zeichenketten direkt vergleichen. `von <= uhrzeit < bis` schließt den Beginn ein und das Ende aus. `filtere_ereignis` unterdrückt nur die Ereignisse, die Menschen stören (`ALARM`, `VERSCHLECHTERT`, `ERINNERUNG`), und lässt die Entwarnung durch, damit niemand auf einen ausgefallenen Dienst wartet, der längst wieder läuft.

```python
from datetime import datetime

TAGE = ["Mo", "Di", "Mi", "Do", "Fr", "Sa", "So"]


def in_wartung(jetzt, fenster):
    """Prüft, ob 'jetzt' (datetime) in einem Wartungsfenster liegt.

    fenster ist eine Liste von (Wochentag, "HH:MM", "HH:MM"), zum Beispiel ("Sa", "02:00", "04:00").
    """
    tag = TAGE[jetzt.weekday()]
    uhrzeit = jetzt.strftime("%H:%M")                     # Zeichenketten im Format HH:MM lassen sich direkt vergleichen
    return any(t == tag and von <= uhrzeit < bis for t, von, bis in fenster)


def filtere_ereignis(ereignis, jetzt, fenster):
    """In der Wartung werden Alarme unterdrückt, die Entwarnung aber nie."""
    if ereignis in ("ALARM", "VERSCHLECHTERT", "ERINNERUNG") and in_wartung(jetzt, fenster):
        return None
    return ereignis


fenster = [("Sa", "02:00", "04:00"), ("Mi", "22:00", "23:30")]
samstag = datetime(2027, 2, 6, 3, 15)                    # Samstag, 6. Februar 2027
print("Wochentag von", samstag.date(), ":", TAGE[samstag.weekday()])
for zeit in (datetime(2027, 2, 6, 3, 15), datetime(2027, 2, 6, 4, 0), datetime(2027, 2, 6, 1, 59),
             datetime(2027, 2, 3, 22, 30), datetime(2027, 2, 4, 3, 0)):
    print(TAGE[zeit.weekday()], zeit.strftime("%d.%m. %H:%M"), "->", in_wartung(zeit, fenster))
print("Alarm in der Wartung :", filtere_ereignis("ALARM", samstag, fenster))
print("Entwarnung           :", filtere_ereignis("ERHOLT", samstag, fenster))
print("Alarm außerhalb      :", filtere_ereignis("ALARM", datetime(2027, 2, 8, 12, 0), fenster))
```

```text
Wochentag von 2027-02-06 : Sa
Sa 06.02. 03:15 -> True
Sa 06.02. 04:00 -> False
Sa 06.02. 01:59 -> False
Mi 03.02. 22:30 -> True
Do 04.02. 03:00 -> False
Alarm in der Wartung : None
Entwarnung           : ERHOLT
Alarm außerhalb      : ALARM
```

Die Ausgabe zeigt, dass der Samstag, 6. Februar 2027, tatsächlich ein Samstag ist, dass Beginn und Ende des Fensters richtig behandelt werden (04:00 liegt schon außerhalb, 01:59 noch davor), dass das Mittwochsfenster greift und ein Donnerstag nicht. Der Alarm im Fenster wird unterdrückt (`None`), die Entwarnung bleibt, und außerhalb des Fensters kommt der Alarm durch. Eine Frage, die in der Praxis wichtig ist: Zeitzonen (Modul 8, 8.3). Ein `datetime` ohne Zeitzone gilt als Ortszeit des Rechners; läuft der Rechner in UTC, werden die Fenster in UTC gelesen. Das muss man bewusst festlegen und dokumentieren. Bei einer Umsetzung im Alerting müsste man außerdem entscheiden, ob ein unterdrückter Alarm nach dem Ende des Fensters nachgeholt wird, wenn die Störung dann noch besteht (in unserer `entscheide`-Logik würde er nicht erneut ausgelöst, sondern erst als Erinnerung).
