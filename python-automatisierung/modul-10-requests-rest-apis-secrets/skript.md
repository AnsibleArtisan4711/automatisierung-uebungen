# Modul 10 - requests, REST-APIs, Secrets

> **Hinweis zu diesem Kapitel:** Das ist das Kapitel des Skripts „Python – Automatisierung unter Linux“, aus der LaTeX-Quelle in Markdown umgewandelt (Text und Codeblöcke unverändert, farbige Boxen als Zitatblöcke mit fettem Titel, Tabellen als Markdown-Tabellen, Übungsaufgaben und Lösungen als Absätze). Die Lerneinheiten sind wie in der Quelle nach Modul nummeriert (10.1 bis 10.5), danach folgen „Übungsaufgaben zu Modul 10“ (8 Aufgaben) und „Lösungen zu Modul 10“. Welche Beispiele ich selbst ausgeführt habe, steht in der [README des Repositories](../README.md). Übersicht zum Modul: [README](README.md).

**Dauer:** 10 h (5 Lerntage à 2 h, Mo 18.01.2027 bis Fr 22.01.2027)

**Summe der Lerneinheiten:** 10 h (5 Lerneinheiten à 2 h)

**Priorität:** 2 (Annahme: Der Lehrplan nennt keine Priorität; das Modul verbindet Ihre Skripte mit Webdiensten und zeigt, wie man dabei sicher mit Zugangsdaten umgeht)

**Meilenstein:** API-Abfrage

**Lernziele.** Nach diesem Modul können Sie mit der Bibliothek `requests` Webdienste und REST-Schnittstellen abfragen, JSON-Antworten sicher auswerten und durch mehrere Seiten blättern, Zeitüberschreitungen, Wiederholungen und Fehler sauber behandeln, Zugangsdaten und andere Geheimnisse außerhalb des Quelltexts halten und nicht versehentlich ausgeben, und all das in einem kleinen Kommandozeilenprogramm zusammenführen, das einen Monitoring-Dienst abfragt.

| **Nr.** | **Lerneinheit**                     | **Datum**     | **Dauer** | **Prio** |
|:--------|:------------------------------------|:--------------|:----------|:---------|
| 10.1    | HTTP & requests Grundlagen          | Mo 18.01.2027 | 2 h       | 2        |
| 10.2    | JSON-APIs auswerten                 | Di 19.01.2027 | 2 h       | 2        |
| 10.3    | Timeouts, Retries, Fehlerbehandlung | Mi 20.01.2027 | 2 h       | 2        |
| 10.4    | Secrets & Umgebungsvariablen        | Do 21.01.2027 | 2 h       | 2        |
| 10.5    | API-Abfrage-Projekt + Quiz          | Fr 22.01.2027 | 2 h       | 2        |
|         | **Summe**                           |               | **10 h**  |          |

> **Merke**
>
> Alle Beispiele dieses Moduls sprechen mit einem kleinen *lokalen Übungsserver* (Modul `testapi`, vollständig in 10.1 abgedruckt), nicht mit dem Internet. So sind die Ergebnisse reproduzierbar, Fehlerfälle (Überlastung, Zeitüberschreitung, falsches Token) lassen sich gezielt herbeiführen, und niemand wird mit Testanfragen belastet. Voraussetzung ist die Bibliothek `requests` (`pip install requests`, siehe Modul 7); getestet wurde mit requests 2.34.2 und urllib3 2.8.0 unter Python 3.13. Die Übungsserver-Beispiele bauen aufeinander auf: Die Module `hostsapi`, `apiclient`, `konfig` und `hostabfrage` entstehen in 10.2 bis 10.5 und werden jeweils als Datei gleichen Namens neben das nächste Beispiel gelegt.

## 10.1 HTTP und requests: die Grundlagen

Fast alles, womit Programme heute über das Netz miteinander sprechen, benutzt *HTTP*, das Protokoll des Webs. Der Browser, mit dem Sie eine Seite öffnen, nutzt es; ebenso die Monitoring-Oberfläche, die Ihnen den Zustand der Server zeigt, der Cloud-Anbieter, bei dem Sie eine virtuelle Maschine anlegen, und das Ticketsystem, das Alarme entgegennimmt. Wer Administrationsaufgaben automatisieren will, muss diese Dienste aus Skripten heraus ansprechen können, und dafür ist HTTP die Sprache. Zum Glück ist es ein erstaunlich einfaches Protokoll, und die Python-Bibliothek `requests` verbirgt fast alle Einzelheiten.

**Anfrage und Antwort.** HTTP arbeitet nach dem Muster *Frage und Antwort*. Der *Client* (hier: Ihr Skript) schickt eine *Anfrage* (englisch *request*) an einen *Server*, und der Server schickt genau eine *Antwort* (*response*) zurück. Dann ist der Austausch beendet; der Server behält von sich aus nichts über den Client im Kopf (man sagt, HTTP sei *zustandslos*). Eine Anfrage besteht aus vier Teilen: der *Methode*, die sagt, was getan werden soll, der *URL*, die sagt, worauf sich das bezieht, den *Kopfzeilen* (*Headers*), die Zusatzinformationen als Paare aus Name und Wert enthalten, und optional einem *Körper* (*Body*) mit Nutzdaten. Die Antwort hat dieselbe Gestalt, nur dass statt Methode und URL ein dreistelliger *Statuscode* steht, der das Ergebnis zusammenfasst.

Die wichtigsten Methoden sind `GET` (eine Ressource lesen), `POST` (etwas Neues anlegen oder eine Aktion auslösen), `PUT` (eine Ressource vollständig ersetzen), `PATCH` (teilweise ändern) und `DELETE` (löschen). Zwei Eigenschaften dieser Methoden werden uns in 10.3 beschäftigen. Eine Methode heißt *sicher* (*safe*), wenn sie nichts am Server verändert; das gilt für `GET` und `HEAD`. Sie heißt *idempotent*, wenn mehrfaches Senden dasselbe bewirkt wie einmaliges (vergleiche 9.4); das gilt außer für die sicheren Methoden auch für `PUT` und `DELETE`, nicht aber für `POST`. Die Dokumentation der Bibliothek urllib3 (siehe 10.3) bezieht diese Einteilung ausdrücklich in ihre Voreinstellungen ein: Standardmäßig werden nur die Methoden `DELETE`, `GET`, `HEAD`, `OPTIONS`, `PUT` und `TRACE` automatisch wiederholt.

Der Statuscode ist in fünf Klassen eingeteilt, die man an der ersten Ziffer erkennt. `1xx` sind Zwischenmeldungen, die man selten sieht. `2xx` bedeutet Erfolg: `200 OK` (alles gut), `201 Created` (etwas wurde angelegt), `204 No Content` (Erfolg ohne Antwortkörper). `3xx` sind Umleitungen: `301 Moved Permanently` und `302 Found` schicken den Client mit einer `Location`-Kopfzeile zu einer anderen Adresse. `4xx` sind Fehler, die auf der Seite des Clients liegen: `400 Bad Request` (die Anfrage ist fehlerhaft), `401 Unauthorized` (Anmeldung fehlt oder ist falsch; trotz des Namens geht es um die *Authentifizierung*), `403 Forbidden` (angemeldet, aber nicht berechtigt), `404 Not Found` (gibt es nicht), `429 Too Many Requests` (zu viele Anfragen in zu kurzer Zeit). `5xx` sind Fehler auf der Serverseite: `500 Internal Server Error` (der Server ist abgestürzt oder hat einen Fehler), `502 Bad Gateway` und `504 Gateway Timeout` (ein zwischengeschalteter Server bekam keine brauchbare Antwort von dahinter), `503 Service Unavailable` (der Dienst ist vorübergehend nicht verfügbar, etwa überlastet oder in Wartung). Diese Einteilung ist für Automatisierung entscheidend: Ein `4xx` sagt „Ihr Skript hat etwas falsch gemacht, nochmaliges Versuchen hilft nicht“, ein `5xx` oder `429` dagegen oft „versuchen Sie es später noch einmal“.

> **Ergänzung aus der Nachrecherche**
>
> **Statuscodes und Retry-After (abgerufen).** Laut der MDN-Seite zu `429` ist dies die Antwort darauf, dass „der Client zu viele Anfragen in einer bestimmten Zeit gesendet hat“, der Mechanismus heißt *Rate Limiting*; eine `Retry-After`-Kopfzeile *kann* beigefügt sein, um anzugeben, wie lange der Client warten soll. Die Benennungen der Statuscodes (`200 OK`, `404 Not Found`, `503 Service Unavailable` usw.) entsprechen dem Inhaltsverzeichnis von RFC 9110 (HTTP Semantics); von dieser Norm war in der Prüfumgebung nur die Gliederung lesbar, die Details zu Methoden und Kopfzeilen sind daher Fachwissen. Quellen im Quellenverzeichnis.

**URLs und Parameter.** Eine URL wie `https://api.example.org/v1/hosts?status=down&page=2` besteht aus dem *Schema* (`https`), dem *Rechner* samt optionalem Port (`api.example.org`), dem *Pfad* (`/v1/hosts`) und der *Query* nach dem Fragezeichen, einer Reihe von `name=wert`-Paaren, getrennt durch `&`. Sonderzeichen müssen in der Query codiert werden (ein Leerzeichen wird zu `+` oder `%20`, ein `&` im Wert zu `%26`), sonst verwechselt der Server Werte mit Trennzeichen. Diese Arbeit nimmt `requests` Ihnen ab, wenn Sie die Parameter als Dictionary übergeben, und das sollten Sie immer tun: `requests.get(url, params={"status": "down"})` statt eines von Hand zusammengebauten f-Strings. Das Schema `https` bedeutet, dass die Verbindung mit TLS verschlüsselt und der Server anhand seines Zertifikats geprüft wird; `requests` prüft das Zertifikat standardmäßig und bricht ab, wenn es nicht stimmt.

**Der Übungsserver.** Um die Beispiele dieses Moduls ohne Internet und ohne fremde Server auszuführen, verwenden wir ein kleines Programm, das mit der Standardbibliothek einen HTTP-Server auf dem eigenen Rechner startet und eine Handvoll Pfade anbietet: eine Liste von Hosts mit Seitenaufteilung, einen einzelnen Host, einen langsamen Pfad, einen launischen Pfad, der anfangs mit `503` antwortet, einen Pfad, der ein Token verlangt, und einige mehr. Der Server hört nur auf `127.0.0.1` (dem eigenen Rechner) und einem vom System frei gewählten Port, ist von außen also nicht erreichbar. Sie müssen den Code nicht verstehen, um das Modul durchzuarbeiten; er steht hier vollständig, weil jede Behauptung in den folgenden Lerneinheiten nachprüfbar sein soll. Speichern Sie ihn als `testapi.py` in Ihrem Übungsordner, dann können Sie jedes Beispiel nachvollziehen. Wer neugierig ist, sieht Folgendes: Eine Klasse `Handler` erbt von `BaseHTTPRequestHandler` (Standardbibliothek, Modul `http.server`), und ihre Methoden `do_GET` und `do_POST` werden für die jeweilige Anfrage aufgerufen; `self.path` enthält den angefragten Pfad, `self.headers` die Kopfzeilen, und `senden` schreibt Statuszeile, Kopfzeilen und Körper. Die Funktion `starte` startet den Server in einem Hintergrund-Thread und gibt die Basisadresse zurück.

```python
"""testapi - kleiner lokaler Uebungsserver fuer die Module 10 und 12."""
import json
import threading
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import parse_qs, urlparse

HOSTS = [
    {"name": "web01", "ip": "192.0.2.11", "status": "up", "cpu": 23.5},
    {"name": "web02", "ip": "192.0.2.12", "status": "up", "cpu": 71.0},
    {"name": "db01", "ip": "192.0.2.21", "status": "up", "cpu": 12.8},
    {"name": "db02", "ip": "192.0.2.22", "status": "down", "cpu": 0.0},
    {"name": "cache01", "ip": "192.0.2.31", "status": "up", "cpu": 5.2},
    {"name": "backup01", "ip": "192.0.2.41", "status": "degraded", "cpu": 88.4},
    {"name": "mail01", "ip": "192.0.2.51", "status": "up", "cpu": 9.9},
]
TOKEN = "geheim123"
TOKEN_PFLICHT = False  # True: alle Pfade unter /api/ verlangen den Bearer-Token
ZAEHLER = {}          # Anzahl der Aufrufe je Schluessel (fuer /flaky)
EMPFANGEN = []        # Nutzlasten, die per POST an /webhook kamen


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def log_message(self, *args):
        pass                                   # keine Zugriffsmeldungen auf der Konsole

    def handle(self):
        try:
            super().handle()
        except (BrokenPipeError, ConnectionResetError):
            pass                               # Client hat vorzeitig aufgelegt (z. B. Timeout)

    def senden(self, code, daten=None, typ="application/json", extra=None):
        if daten is None:
            daten = {"status": code}
        koerper = daten if isinstance(daten, bytes) else (
            daten.encode() if isinstance(daten, str) else json.dumps(daten).encode())
        self.send_response(code)
        self.send_header("Content-Type", typ)
        self.send_header("Content-Length", str(len(koerper)))
        for name, wert in (extra or {}).items():
            self.send_header(name, wert)
        self.end_headers()
        self.wfile.write(koerper)

    def do_GET(self):
        url = urlparse(self.path)
        q = {k: v[0] for k, v in parse_qs(url.query).items()}
        pfad = url.path
        if TOKEN_PFLICHT and pfad.startswith("/api/") and \
                self.headers.get("Authorization") != f"Bearer {TOKEN}":
            self.senden(401, {"error": "Token fehlt oder falsch"}, extra={"WWW-Authenticate": "Bearer"})
        elif pfad == "/api/hosts":
            seite, pro_seite = int(q.get("page", 1)), int(q.get("per_page", 3))
            teil = HOSTS[(seite - 1) * pro_seite: seite * pro_seite]
            seiten = -(-len(HOSTS) // pro_seite)
            self.senden(200, {"page": seite, "pages": seiten, "items": teil,
                              "next": seite + 1 if seite < seiten else None})
        elif pfad.startswith("/api/hosts/"):
            name = pfad.rsplit("/", 1)[1]
            treffer = [h for h in HOSTS if h["name"] == name]
            if treffer:
                self.senden(200, treffer[0])
            else:
                self.senden(404, {"error": f"unbekannt: {name}"})
        elif pfad.startswith("/status/"):
            self.senden(int(pfad.rsplit("/", 1)[1]))
        elif pfad == "/slow":
            time.sleep(float(q.get("s", 1)))
            self.senden(200, {"ok": True})
        elif pfad == "/flaky":
            schluessel = q.get("id", "standard")
            ZAEHLER[schluessel] = ZAEHLER.get(schluessel, 0) + 1
            if ZAEHLER[schluessel] <= int(q.get("fails", 2)):
                extra = {"Retry-After": q["retry_after"]} if "retry_after" in q else None
                self.senden(503, {"error": "ueberlastet"}, extra=extra)
            else:
                self.senden(200, {"ok": True, "versuch": ZAEHLER[schluessel]})
        elif pfad == "/secure":
            if self.headers.get("Authorization") == f"Bearer {TOKEN}":
                self.senden(200, {"user": "demo", "rolle": "lesen"})
            else:
                self.senden(401, {"error": "Token fehlt oder falsch"},
                            extra={"WWW-Authenticate": "Bearer"})
        elif pfad == "/headers":
            self.senden(200, {k.lower(): v for k, v in self.headers.items()})
        elif pfad == "/redirect":
            self.senden(302, {"weiter": "/api/hosts"}, extra={"Location": "/api/hosts?per_page=2"})
        elif pfad == "/kaputt":
            self.senden(200, "<html>Wartungsarbeiten</html>", typ="text/html")
        else:
            self.senden(404, {"error": "Pfad unbekannt"})

    def do_POST(self):
        laenge = int(self.headers.get("Content-Length", 0))
        roh = self.rfile.read(laenge)
        typ = self.headers.get("Content-Type", "")
        if self.path == "/api/echo":
            self.senden(200, {"content_type": typ, "laenge": laenge, "text": roh.decode()})
        elif self.path.startswith("/flaky"):
            ZAEHLER["post"] = ZAEHLER.get("post", 0) + 1
            self.senden(503, {"error": "ueberlastet", "aufrufe": ZAEHLER["post"]})
        elif self.path == "/webhook":
            try:
                EMPFANGEN.append(json.loads(roh))
            except json.JSONDecodeError:
                self.senden(400, {"error": "kein JSON"})
                return
            self.senden(200, {"empfangen": len(EMPFANGEN)})
        else:
            self.senden(404, {"error": "Pfad unbekannt"})


def starte():
    """Startet den Server auf einem freien Port und gibt (server, basis_url) zurueck."""
    server = ThreadingHTTPServer(("127.0.0.1", 0), Handler)
    threading.Thread(target=server.serve_forever, daemon=True).start()
    return server, f"http://127.0.0.1:{server.server_address[1]}"
```

**Die erste Anfrage.** Nun können wir `requests` ausprobieren. Die Funktion `requests.get` schickt eine GET-Anfrage und liefert ein `Response`-Objekt. Es hat eine Reihe von Attributen, die man sich merken sollte: `status_code` (die Zahl), `reason` (der Text dazu), `ok` (`True` für alle Statuscodes unter 400), `headers` (ein Dictionary, bei dem die Groß- und Kleinschreibung der Namen egal ist), `text` (der Körper als Zeichenkette, die Zeichencodierung rät `requests` aus den Kopfzeilen), `content` (der Körper als rohe Bytes) und `json()` (der Körper, als JSON gelesen und in Python-Objekte umgewandelt, siehe 10.2). Das Programm fragt die zweite Seite der Hostliste mit zwei Einträgen pro Seite ab; die Parameter gehen als Dictionary in `params`. Der Aufruf enthält außerdem `timeout=5`, was wir in 10.3 erklären; bis dahin nehmen Sie es als Gewohnheit hin.

```python
import requests

import testapi

server, basis = testapi.starte()

antwort = requests.get(basis + "/api/hosts", params={"per_page": 2, "page": 2}, timeout=5)

print("URL         :", antwort.url.replace(basis, ""))
print("Statuscode  :", antwort.status_code, antwort.reason)
print("erfolgreich :", antwort.ok)
print("Content-Type:", antwort.headers["Content-Type"])
print("Typ des Textes:", type(antwort.text).__name__, "| Typ der Rohdaten:", type(antwort.content).__name__)
print("Rohdaten    :", antwort.content[:60])
daten = antwort.json()
print("JSON-Schlüssel:", sorted(daten))
print("erster Host :", daten["items"][0])
```

```text
URL         : /api/hosts?per_page=2&page=2
Statuscode  : 200 OK
erfolgreich : True
Content-Type: application/json
Typ des Textes: str | Typ der Rohdaten: bytes
Rohdaten    : b'{"page": 2, "pages": 4, "items": [{"name": "db01", "ip": "19'
JSON-Schlüssel: ['items', 'next', 'page', 'pages']
erster Host : {'name': 'db01', 'ip': '192.0.2.21', 'status': 'up', 'cpu': 12.8}
```

Die Ausgabe zeigt zuerst die URL, die `requests` aus der Basisadresse und den Parametern gebaut hat; die Reihenfolge der Parameter folgt dem Dictionary. Der Statuscode ist 200, und das Attribut `ok` bestätigt den Erfolg. Der `Content-Type` verrät, dass der Körper JSON ist. Der Unterschied zwischen `text` (eine Zeichenkette) und `content` (Bytes) ist wichtig: Bytes sind das, was tatsächlich übertragen wurde, die Zeichenkette ist das Ergebnis der Decodierung. Zum Schluss liefert `json()` ein Dictionary, aus dem wir die Schlüssel und den ersten Eintrag der Liste lesen. Die Aufteilung der API in Seiten (hier `page`, `pages`, `items`, `next`) ist ein Thema von 10.2.

**Daten senden: POST, Formulare und JSON.** Bei `POST` (und `PUT`, `PATCH`) schickt man Daten im Körper der Anfrage. `requests` kennt dafür zwei Parameter, die sich in der Art der Verpackung unterscheiden. Mit `data={...}` wird der Körper als *Formular* codiert (Content-Type `application/x-www-form-urlencoded`), wie es ein HTML-Formular im Browser täte. Mit `json={...}` wird das Dictionary in JSON umgewandelt, und `requests` setzt die Kopfzeile `Content-Type: application/json` selbst. Moderne APIs erwarten fast immer JSON. Die Kopfzeilen kann man mit dem Parameter `headers` ergänzen; `requests` setzt von selbst sinnvolle Vorgaben (unter anderem einen `User-Agent`, der Ihr Programm als `python-requests/...` ausweist). Es ist guter Stil, einen eigenen `User-Agent` zu setzen, der Ihr Skript benennt, damit der Betreiber des Dienstes im Protokoll sieht, wer anfragt.

```python
import requests

import testapi

server, basis = testapi.starte()

mit_json = requests.post(basis + "/api/echo", json={"name": "web03", "tags": ["prod", "öffentlich"]}, timeout=5)
print("json=  :", mit_json.json()["content_type"], "|", mit_json.json()["text"])

mit_formular = requests.post(basis + "/api/echo", data={"name": "web03", "kommentar": "zwei Wörter"}, timeout=5)
print("data=  :", mit_formular.json()["content_type"], "|", mit_formular.json()["text"])

antwort = requests.get(basis + "/headers", headers={"User-Agent": "hostabfrage/1.0", "Accept": "application/json"}, timeout=5)
gesendet = antwort.json()
print("Server sah:", gesendet["user-agent"], "|", gesendet["accept"])
```

```text
json=  : application/json | {"name": "web03", "tags": ["prod", "\u00f6ffentlich"]}
data=  : application/x-www-form-urlencoded | name=web03&kommentar=zwei+W%C3%B6rter
Server sah: hostabfrage/1.0 | application/json
```

Der Echo-Pfad des Übungsservers spiegelt zurück, was er erhalten hat. Man sieht, dass `json=` die Zeichenfolge als JSON ausgibt (mit `\u00f6` für das *ö*, denn `json.dumps` verwendet standardmäßig ASCII), während `data=` die Felder als `name=wert&...` mit Prozent-Codierung schickt. Beide Wege führen zum Ziel, aber der Server muss den richtigen erwarten: Schickt man Formulardaten an eine API, die JSON erwartet, antwortet sie meist mit `400` oder `415`.

**Umleitungen und Fehlerstatus.** Zwei Verhalten von `requests` überraschen Anfänger. Erstens folgt die Bibliothek bei `GET` automatisch Umleitungen (`3xx` mit `Location`); die Liste `response.history` zeigt die übersprungenen Stationen, und mit `allow_redirects=False` schaltet man das ab. Zweitens, und das ist wichtiger: *Ein Fehlerstatus wie 404 oder 500 löst keine Ausnahme aus.* Die Anfrage selbst war ja erfolgreich, der Server hat geantwortet, nur eben mit einer schlechten Nachricht. Wer das nicht weiß, arbeitet mit der Fehlerseite weiter, als wäre sie das gewünschte Ergebnis. Die Methode `raise_for_status()` wandelt Fehlerstatus (4xx und 5xx) in eine Ausnahme vom Typ `requests.HTTPError` um; sie ist die wichtigste Zeile nach jeder Anfrage.

```python
import requests

import testapi

server, basis = testapi.starte()

folgt = requests.get(basis + "/redirect", timeout=5)
print("mit Umleitung :", folgt.status_code, folgt.url.replace(basis, ""), "| Stationen:", [r.status_code for r in folgt.history])

folgt_nicht = requests.get(basis + "/redirect", allow_redirects=False, timeout=5)
print("ohne Umleitung:", folgt_nicht.status_code, "->", folgt_nicht.headers["Location"])

nicht_da = requests.get(basis + "/api/hosts/gibtsnicht", timeout=5)
print("404 ist kein Python-Fehler:", nicht_da.status_code, nicht_da.json())
try:
    nicht_da.raise_for_status()
except requests.HTTPError as fehler:
    print("raise_for_status:", type(fehler).__name__, "-", str(fehler).replace(basis, ""))
```

```text
mit Umleitung : 200 /api/hosts?per_page=2 | Stationen: [302]
ohne Umleitung: 302 -> /api/hosts?per_page=2
404 ist kein Python-Fehler: 404 {'error': 'unbekannt: gibtsnicht'}
raise_for_status: HTTPError - 404 Client Error: Not Found for url: /api/hosts/gibtsnicht
```

Die erste Anfrage wurde einmal umgeleitet (Station mit Status 302) und landete bei der Hostliste mit Status 200; mit `allow_redirects=False` sieht man die `302` samt Zielangabe selbst. Der Abruf eines unbekannten Hosts liefert `404` und einen JSON-Körper, aber keine Python-Ausnahme; erst `raise_for_status()` macht daraus einen `HTTPError`, dessen Text Statuscode, Grund und URL nennt. Die Ausnahme trägt die Antwort in `fehler.response` mit sich, sodass man Details (Statuscode, Körper) auslesen kann. Ein Muster, das Sie sich angewöhnen sollten, sieht so aus: Anfrage mit `timeout`, danach `raise_for_status()`, danach `json()`.

**Sitzungen.** Möchte man mehrere Anfragen an denselben Server schicken, erzeugt man mit `requests.Session()` ein Sitzungsobjekt und ruft `get` und `post` auf diesem auf. Das hat zwei Vorteile. Die Sitzung hält die Netzwerkverbindung offen und verwendet sie für die nächste Anfrage wieder, was Zeit spart (besonders bei verschlüsselten Verbindungen, deren Aufbau teuer ist). Und sie merkt sich Einstellungen, die für alle Anfragen gelten sollen: Kopfzeilen (`sitzung.headers.update({...})`), Anmeldedaten und Cookies. Wir verwenden in 10.2 bis 10.5 durchgehend Sitzungen, am besten als Kontextmanager (`with requests.Session() as sitzung:`), damit die Verbindungen am Ende sauber geschlossen werden.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler am Anfang lassen sich an einem einzigen Programm zeigen, das Sie vielleicht in ähnlicher Form schon gesehen haben. Es baut die URL von Hand, vergisst die Zeitbegrenzung, prüft den Status nicht und verwendet die Antwort einfach weiter.

```python
import requests

# FALSCH
name = "web 01 & co"
antwort = requests.get("http://monitoring.example.org/api/hosts?name=" + name)
print(antwort.json()["status"])

# RICHTIG
antwort = requests.get("http://monitoring.example.org/api/hosts",
                       params={"name": name}, timeout=5)
antwort.raise_for_status()
print(antwort.json()["status"])
```

Im falschen Teil enthält die URL ein Leerzeichen und ein `&`, die der Server als Trennzeichen missversteht; die Anfrage ändert ihre Bedeutung oder schlägt fehl. Es gibt kein `timeout`, sodass das Programm bei einem hängenden Server theoretisch ewig wartet (siehe 10.3). Wenn der Server mit `404` oder `500` antwortet, bricht erst das `.json()` mit einer Ausnahme ab, die nichts mit der eigentlichen Ursache zu tun hat, oder, noch schlimmer, es klappt und liefert die Daten einer Fehlerseite. Der richtige Teil verwendet `params`, setzt ein Zeitlimit und lässt `raise_for_status()` ein Fehlerereignis auslösen, wo es hingehört. Als zweiten häufigen Fehler sehen wir in Skripten `http://` statt `https://` für Dienste, die Zugangsdaten verlangen: Das Token wandert dann unverschlüsselt durch das Netz. Und `verify=False` (schaltet die Zertifikatsprüfung ab, damit eine Fehlermeldung verschwindet) ist fast nie die richtige Lösung; der Fehler beim Zertifikat sagt Ihnen etwas Wichtiges, und Abschalten öffnet Angreifern die Tür.

> **Ergänzung aus der Nachrecherche**
>
> **Was in der Prüfumgebung beobachtet wurde.** Alle Beispiele dieser Lerneinheit laufen gegen den lokalen Übungsserver mit requests 2.34.2. Die Dokumentation von `requests` (requests.readthedocs.io) war für den Abruf nicht freigegeben; die hier beschriebenen Eigenschaften (Attribute von `Response`, `raise_for_status`, automatische Umleitungen, Verhalten von `json=` und `data=`) wurden deshalb ausschließlich durch Ausprobieren und durch Lesen der installierten Bibliothek belegt. Verweise auf die offizielle Dokumentation stehen im Quellenverzeichnis unter *weiterführend*.

> **Merke**
>
> HTTP besteht aus Anfrage (Methode, URL, Kopfzeilen, Körper) und Antwort (Statuscode, Kopfzeilen, Körper). `2xx` Erfolg, `3xx` Umleitung, `4xx` Fehler des Clients, `5xx` Fehler des Servers. Parameter immer als `params=` übergeben, JSON-Daten als `json=`. Ein Fehlerstatus ist *keine* Ausnahme; `raise_for_status()` macht eine daraus. Immer ein `timeout` angeben (siehe 10.3), Sitzungen für mehrere Anfragen verwenden, Zertifikate nicht abschalten.

> **Praxisbeispiel**
>
> Fast jede moderne Verwaltungsoberfläche (Monitoring, Ticketsystem, Cloud-Konsole, Verzeichnisdienst, Firewall) bietet neben der Weboberfläche eine HTTP-Schnittstelle an, über die man dasselbe aus einem Skript erledigen kann. Vor dem Programmieren lohnt der Probelauf mit dem Kommandozeilenwerkzeug `curl` (zum Beispiel `curl -i https://api.example.org/v1/hosts`); die Option `-i` zeigt Statuszeile und Kopfzeilen mit an. Was `curl` liefert, muss auch Ihr Python-Programm liefern, und wenn nicht, liegt der Unterschied fast immer in den Kopfzeilen oder in der Anmeldung.

## 10.2 JSON-APIs auswerten

Wenn von einer „REST-API“ oder „Web-API“ die Rede ist, meint man fast immer einen Dienst, der über HTTP erreichbar ist und seine Daten als JSON liefert. Das *REST*-Prinzip (Representational State Transfer) ist weniger eine Vorschrift als eine Gestaltungsregel: Alles, was der Dienst verwaltet, ist eine *Ressource* mit eigener Adresse (`/api/hosts` ist die Sammlung aller Hosts, `/api/hosts/web01` ein einzelner), und die HTTP-Methoden bestimmen die Operation (`GET` liest, `POST` legt an, `PUT` und `PATCH` ändern, `DELETE` löscht). Wenn Sie diese Schreibweise einmal verstanden haben, können Sie sich in fast jeder Schnittstelle zurechtfinden, auch wenn die Details der einzelnen Anbieter voneinander abweichen: Dort entscheidet immer die Dokumentation der Schnittstelle über Pfade, Parameter und Anmeldung.

**JSON in Python.** JSON (JavaScript Object Notation) kennen Sie aus Modul 7. Zur Wiederholung: Die Werte eines JSON-Textes sind Objekte (`{...}`), Listen (`[...]`), Zeichenketten, Zahlen, die Wahrheitswerte `true` und `false` sowie `null`. Python wandelt sie in Dictionaries, Listen, `str`, `int` oder `float`, `True`, `False` und `None` um; `response.json()` ist nichts anderes als `json.loads(response.text)` mit besserer Fehlermeldung. Es gibt zwei Dinge, die man beim Lesen fremder JSON-Antworten nie vergessen darf. Erstens ist das Ergebnis nur so zuverlässig wie der Server: Felder können fehlen, `null` sein oder einen anderen Typ haben als dokumentiert, und neue Versionen der API können Felder hinzufügen oder entfernen. Zweitens kennt JSON keine Datums- oder Zeitwerte; sie kommen als Zeichenketten an, meist im ISO-Format (`2027-01-18T09:30:00Z`), und müssen mit den Werkzeugen aus Modul 8 umgewandelt werden.

**Durch Seiten blättern (Pagination).** Liefert eine Sammlung Tausende Einträge, schickt der Server sie nicht auf einmal, sondern in *Seiten*. Das schont Server und Netz. Wie das genau funktioniert, ist von API zu API verschieden. Die häufigsten Varianten sind: ein Parameter `page` (und `per_page` oder `limit`) mit einem Hinweis in der Antwort, ob es eine weitere Seite gibt; ein `offset`; ein *Cursor* (ein vom Server vergebener Marker, der in der nächsten Anfrage mitgeschickt wird); oder ein `Link`-Header mit der Adresse der nächsten Seite. Unser Übungsserver wählt die erste Variante: Die Antwort enthält `page`, `pages`, `items` und `next` (die Nummer der nächsten Seite oder `null` bei der letzten). Eine Schleife, die so lange weiterblättert, wie es eine nächste Seite gibt, ist ein klassischer Fall für einen *Generator* (Modul 3 und 4): Die Funktion liefert mit `yield from` die Einträge eines nach dem anderen und holt die nächste Seite erst, wenn der Aufrufer danach verlangt. Das ist speichersparend, und man kann nach den ersten Treffern abbrechen, ohne alle Seiten geladen zu haben.

```python
def alle_hosts(sitzung, basis, pro_seite=3):
    """Liefert nacheinander alle Hosts und blättert dabei durch die Seiten der API."""
    seite = 1
    while seite is not None:
        antwort = sitzung.get(f"{basis}/api/hosts", params={"page": seite, "per_page": pro_seite}, timeout=5)
        antwort.raise_for_status()
        daten = antwort.json()
        yield from daten["items"]
        seite = daten["next"]
```

Die Funktion erhält eine Sitzung, die Basisadresse und die Seitengröße. Die Schleife läuft, solange `seite` nicht `None` ist: Sie holt die Seite, prüft den Status mit `raise_for_status()`, liest das JSON, gibt die Einträge der Seite mit `yield from` weiter und setzt `seite` auf den Wert von `next` aus der Antwort. Bei der letzten Seite steht dort `null`, also `None`, und die Schleife endet. Speichern Sie diese Funktion als Datei `hostsapi.py`; die nächsten Beispiele importieren sie. Der Aufrufer kann alle Hosts mit `list(...)` zusammenholen oder mit `next(...)` nur den ersten.

```python
import requests

import testapi
from hostsapi import alle_hosts

server, basis = testapi.starte()
with requests.Session() as sitzung:
    hosts = list(alle_hosts(sitzung, basis))

print(len(hosts), "Hosts:", ", ".join(h["name"] for h in hosts))

with requests.Session() as sitzung:
    erste = next(alle_hosts(sitzung, basis, pro_seite=2))
print("Nur den ersten holen:", erste["name"])
```

```text
7 Hosts: web01, web02, db01, db02, cache01, backup01, mail01
Nur den ersten holen: web01
```

Der erste Aufruf holt alle sieben Hosts in drei Anfragen (bei drei Einträgen pro Seite). Der zweite zeigt die Stärke des Generators: Mit `next` wird nur der erste Eintrag verlangt, es wird also nur die erste Seite geladen. Bei einer API mit Hunderttausenden Einträgen ist das der Unterschied zwischen Sekunden und Stunden. Wichtig: Eine Schleife über *alle* Seiten ist nur dann ungefährlich, wenn die Datenmenge überschaubar ist; sonst belastet man den Server (ein freundlicher Client setzt bei großen Datenmengen eine kleine Pause zwischen die Anfragen oder nutzt Filter, die der Server anbietet).

**Auswerten mit den Werkzeugen aus den Modulen 2 und 3.** Sind die Daten erst einmal als Liste von Dictionaries im Speicher, ist die Arbeit mit der API vorbei und die mit Python beginnt. Alles, was Sie über Listenverarbeitung gelernt haben, gilt jetzt: Listen filtern, mit `Counter` zählen, mit `sorted` und einer Schlüsselfunktion ordnen, mit Dictionaries nach einem Feld indizieren. Das nächste Programm wertet die Hostliste aus: Es zählt die Hosts je Status, nennt alle, die nicht laufen, findet die ausgelasteten und berechnet den Durchschnitt der Prozessorlast über die laufenden Hosts.

```python
from collections import Counter

import requests

import testapi
from hostsapi import alle_hosts


server, basis = testapi.starte()
with requests.Session() as sitzung:
    hosts = list(alle_hosts(sitzung, basis))

nach_status = Counter(h["status"] for h in hosts)
print("Verteilung:", dict(nach_status))

problem = [h for h in hosts if h["status"] != "up"]
for h in sorted(problem, key=lambda h: h["name"]):
    print(f"  {h['name']:10} {h['ip']:13} {h['status']}")

heiss = [h["name"] for h in hosts if h["cpu"] >= 70]
print("CPU >= 70 %:", heiss)

durchschnitt = sum(h["cpu"] for h in hosts if h["status"] == "up") / nach_status["up"]
print(f"Mittlere CPU der laufenden Hosts: {durchschnitt:.1f} %")

nach_name = {h["name"]: h for h in hosts}
print("db02 hat die IP", nach_name["db02"]["ip"])
```

```text
Verteilung: {'up': 5, 'down': 1, 'degraded': 1}
  backup01   192.0.2.41    degraded
  db02       192.0.2.22    down
CPU >= 70 %: ['web02', 'backup01']
Mittlere CPU der laufenden Hosts: 24.5 %
db02 hat die IP 192.0.2.22
```

Das Beispiel enthält mehrere Muster, die Sie in Auswertungen immer wieder brauchen werden. Der `Counter` zählt Vorkommen in einem Durchgang und lässt sich mit `dict(...)` lesbar ausgeben. Die Listenkomprehension mit Bedingung (`[h for h in hosts if ...]`) filtert. Die Sortierung nach Name verwendet eine Schlüsselfunktion; im f-String steuern Formatangaben wie `:10` und `:13` die Spaltenbreite. Beim Durchschnitt teilt der Code durch die Anzahl der laufenden Hosts, die er aus dem Zähler liest; wäre kein Host `up`, gäbe es einen `ZeroDivisionError`, den ein sorgfältiges Programm abfängt oder vorher prüft. Das letzte Muster, ein Dictionary `nach_name`, ermöglicht den schnellen Zugriff auf einen Host über seinen Namen, ohne jedes Mal die Liste zu durchsuchen.

**Defensiv lesen.** Man kann bei einer fremden Schnittstelle nicht davon ausgehen, dass jedes Feld da ist. Es gibt zwei Haltungen. Die *tolerante* Haltung verwendet `dict.get(feld, vorgabe)` und liefert Platzhalter, wenn Daten fehlen; sie eignet sich für Anzeigen, bei denen ein fehlendes Feld nicht schlimm ist. Die *strenge* Haltung greift mit `eintrag["feld"]` zu und lässt den `KeyError` zu, wenn ein Pflichtfeld fehlt; sie eignet sich, wenn man mit falschen Daten keinesfalls weiterarbeiten will. Welche richtig ist, hängt vom Zweck ab: Ein Monitoring, das einen Host ohne Lastwert als „n/a“ anzeigt, ist nützlich; ein Skript, das anhand der IP-Adresse eine Firewall-Regel anlegt, darf bei fehlender IP nicht raten. Das folgende Beispiel zeigt beide Haltungen auf vier Antworten mit unterschiedlichen Lücken. Beachten Sie, dass der Wert `null` zu `None` wird: Das Feld ist dann *vorhanden*, aber leer, und `get` liefert deshalb `None` und nicht die Vorgabe.

```python
import json


def host_zeile(eintrag):
    """Liest einen Host-Eintrag tolerant: fehlende Felder ergeben Platzhalter statt KeyError."""
    name = eintrag.get("name", "?")
    status = eintrag.get("status", "unbekannt")
    cpu = eintrag.get("cpu")
    cpu_text = f"{cpu:.1f} %" if isinstance(cpu, (int, float)) else "n/a"
    return f"{name:8} {status:10} {cpu_text}"


antworten = [
    '{"name": "web01", "status": "up", "cpu": 23.5}',
    '{"name": "web02", "status": "up"}',
    '{"name": "db01", "status": "up", "cpu": null}',
    '{"name": "db02", "cpu": "hoch"}',
]
for text in antworten:
    print(host_zeile(json.loads(text)))

# Streng: ein fehlendes Pflichtfeld soll auffallen, nicht verschwinden
daten = json.loads('{"name": "web02"}')
try:
    ip = daten["ip"]
except KeyError as fehler:
    print("Pflichtfeld fehlt:", fehler)
```

```text
web01    up         23.5 %
web02    up         n/a
db01     up         n/a
db02     unbekannt  n/a
Pflichtfeld fehlt: 'ip'
```

Die Funktion `host_zeile` meldet bei fehlendem oder leerem Lastwert „n/a“, weil sie mit `isinstance` prüft, ob überhaupt eine Zahl vorliegt. Das schützt auch vor dem Fall, dass der Server die Last als Text liefert (`"hoch"` im vierten Beispiel), was bei der Formatierung `:.1f` sonst einen `ValueError` auslösen würde. Der letzte Teil zeigt die strenge Variante: Das fehlende Pflichtfeld `ip` führt zu einem `KeyError` mit dem Namen des Feldes, der in der Praxis in eine verständliche Fehlermeldung übersetzt wird.

**Antworten prüfen.** Wer eine API in ein Programm einbindet, das unbeaufsichtigt läuft, prüft die Antworten, bevor er sie verarbeitet. Dafür gibt es Werkzeuge (JSON Schema mit der Bibliothek `jsonschema`, Datenmodelle mit `pydantic`), die über den Rahmen dieses Moduls hinausgehen; mit den Mitteln, die Sie haben, lässt sich aber eine einfache Prüfung selbst schreiben. Eine Tabelle beschreibt die erwarteten Felder und Typen, eine Funktion gibt die Liste der Probleme zurück. Zwei Details: Python behandelt `True` als Zahl (`bool` ist eine Unterklasse von `int`), weshalb der Code ausdrücklich `bool` ausschließt; und die Funktion sammelt *alle* Probleme, statt beim ersten aufzuhören, weil das die Fehlersuche erleichtert.

```python
import json

PFLICHT = {"name": str, "ip": str, "status": str, "cpu": (int, float)}
ERLAUBT = {"up", "down", "degraded"}


def pruefe_host(eintrag):
    """Gibt eine Liste von Problemen zurück (leer = in Ordnung)."""
    if not isinstance(eintrag, dict):
        return [f"Objekt erwartet, {type(eintrag).__name__} erhalten"]
    probleme = []
    for feld, typ in PFLICHT.items():
        if feld not in eintrag:
            probleme.append(f"Feld '{feld}' fehlt")
        elif isinstance(eintrag[feld], bool) or not isinstance(eintrag[feld], typ):
            probleme.append(f"Feld '{feld}' hat den Typ {type(eintrag[feld]).__name__}")
    if eintrag.get("status") not in ERLAUBT and "status" in eintrag:
        probleme.append(f"unbekannter Status {eintrag['status']!r}")
    return probleme


proben = [
    {"name": "web01", "ip": "192.0.2.11", "status": "up", "cpu": 23.5},
    {"name": "web02", "ip": "192.0.2.12", "status": "up"},
    {"name": "db01", "ip": 21, "status": "kaputt", "cpu": True},
    ["web03"],
]
for probe in proben:
    print(json.dumps(probe)[:45].ljust(45), "->", pruefe_host(probe) or "ok")
```

```text
{"name": "web01", "ip": "192.0.2.11", "status -> ok
{"name": "web02", "ip": "192.0.2.12", "status -> ["Feld 'cpu' fehlt"]
{"name": "db01", "ip": 21, "status": "kaputt" -> ["Feld 'ip' hat den Typ int", "Feld 'cpu' hat den Typ bool", "unbekannter Status 'kaputt'"]
["web03"]                                     -> ['Objekt erwartet, list erhalten']
```

Die erste Probe ist in Ordnung. Die zweite zeigt, dass das Feld `cpu` fehlt. Die dritte hat gleich drei Probleme: eine Zahl als IP-Adresse, einen unbekannten Status und einen Wahrheitswert als Last (der nicht als Zahl durchgeht). Die vierte ist gar kein Objekt, sondern eine Liste. Solche Prüfungen bewahren davor, dass eine fehlerhafte Antwort erst viel später und an unerwarteter Stelle einen Fehler auslöst.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler beim Auswerten von APIs sind die folgenden. Wer `response.json()` aufruft, ohne vorher zu prüfen, ob die Antwort tatsächlich JSON ist, bekommt bei einer HTML-Fehlerseite (etwa einer Wartungsmeldung des Anbieters oder einer Seite eines Proxys) eine `JSONDecodeError`, die erst in 10.3 behandelt wird. Wer sich auf die erste Seite beschränkt, ohne es zu merken, bekommt unvollständige Ergebnisse, und zwar ohne Fehlermeldung: In einer Statistik über „alle“ Hosts fehlen dann die Hosts ab Seite 2. Wer Felder durch Verschachtelung liest (`daten["a"]["b"]["c"]`), fängt sich einen `KeyError` oder `TypeError` ein, wenn eine Zwischenstufe fehlt oder `None` ist. Und wer Zahlen mit Text vergleicht (`"70" > 50`) bekommt einen `TypeError`: JSON liefert Zahlen als Zahlen, manche APIs aber als Text, was man vor der Verarbeitung mit `float()` umwandeln muss.

> **Ergänzung aus der Nachrecherche**
>
> **Wie APIs Seiten umsetzen.** Die Aufteilung in Seiten ist nicht standardisiert; die oben genannten Varianten (Seitennummer, Offset, Cursor, `Link`-Header) sind Fachwissen aus der Praxis und wurden in der Prüfumgebung nur für die erste Variante mit dem Übungsserver ausprobiert. Lesen Sie bei jeder realen API zuerst im Abschnitt zur Seitenaufteilung nach, wie sie das löst, und achten Sie auf Obergrenzen (maximale Seitengröße) und auf Ratenbegrenzungen (siehe 10.3).

> **Merke**
>
> Eine REST-API stellt Ressourcen unter Adressen bereit; die HTTP-Methode bestimmt die Operation. Antworten sind JSON: `response.json()` liefert Dictionaries und Listen. Sammlungen kommen seitenweise, ein Generator mit `yield from` blättert bequem. Fremde Daten defensiv lesen (`get`, `isinstance`) oder streng prüfen (Pflichtfelder), je nach Zweck. Nie annehmen, dass die erste Seite alles enthält.

> **Praxisbeispiel**
>
> Ob Sie die Server einer Cloud auflisten, offene Tickets eines Dienstes zählen oder die Mitglieder einer Gruppe abfragen: Der Ablauf bleibt derselbe. Sitzung öffnen, Seiten abholen, Daten in Python filtern und zusammenfassen, Ergebnis ausgeben. Die Auswertung mit den Mitteln der Standardbibliothek (`Counter`, Listenkomprehensionen, `sorted`) reicht für fast alle Verwaltungsaufgaben aus; erst bei großen Datenmengen lohnen sich Werkzeuge wie `pandas`.

## 10.3 Timeouts, Retries und Fehlerbehandlung

Im Netz geht ständig etwas schief. Server sind überlastet, Verbindungen reißen ab, ein Router zwischen Ihnen und dem Ziel verliert Pakete, ein Zertifikat läuft ab, ein Dienst wird gerade neu gestartet, ein Anbieter begrenzt die Anfragen pro Minute. Ein Skript, das nur im Idealfall funktioniert, ist für den unbeaufsichtigten Betrieb unbrauchbar: Es bleibt nachts hängen, bricht beim ersten Schluckauf ab oder, schlimmer, hämmert mit Anfragen auf einen schon kranken Server ein. Diese Lerneinheit behandelt die drei Werkzeuge, mit denen man das in den Griff bekommt: Zeitlimits, Wiederholungen mit Wartezeit und eine bewusste Behandlung der verschiedenen Fehlerarten.

**Zeitlimits (Timeouts).** Die wichtigste Tatsache zuerst: *Ohne Angabe wartet requests unbegrenzt.* Der Parameter `timeout` hat den Vorgabewert `None`, was „nie aufgeben“ bedeutet; das lässt sich an der Signatur von `Session.request` in der installierten Bibliothek ablesen. Reagiert der Server nicht (er hängt, das Netz verschluckt Pakete), bleibt Ihr Programm für immer an dieser Zeile stehen. Ein Skript, das ein cron-Job startet, würde dann nie fertig, und mit der Zeit liefen immer mehr hängende Kopien (siehe 9.4). Deshalb gilt die Regel: *Jede Anfrage bekommt ein Zeitlimit.*

Das Limit lässt sich auf zwei Arten angeben. Eine einzelne Zahl (`timeout=5`) gilt sowohl für den *Verbindungsaufbau* als auch für das *Lesen*. Ein Tupel (`timeout=(3.05, 10)`) legt beides getrennt fest: die erste Zahl für den Aufbau der Verbindung (wie lange darf es dauern, bis der Server den Anruf annimmt), die zweite für das Warten auf Daten. Wichtig ist, was das Lesen-Limit *nicht* ist: Es ist keine Obergrenze für die Gesamtdauer der Anfrage, sondern die Zeit, die man auf das *nächste Datenpaket* wartet. Ein Server, der alle paar Sekunden ein Byte schickt, umgeht das Limit beliebig lange. Wer eine harte Obergrenze braucht, muss sie selbst durchsetzen (etwa mit einer Frist, die man zwischen Teilen der Arbeit prüft, oder bei größeren Programmen mit eigenen Mechanismen). Das folgende Programm ruft einen Pfad auf, der genau eine Sekunde braucht, und zeigt das Verhalten bei verschiedenen Einstellungen. Der Fehlerfall wird mit `requests.exceptions.RequestException` abgefangen, der gemeinsamen Oberklasse aller Fehler der Bibliothek.

```python
import time

import requests

import testapi

server, basis = testapi.starte()


def versuche(beschreibung, **optionen):
    start = time.monotonic()
    try:
        antwort = requests.get(basis + "/slow", params={"s": 1}, **optionen)
        ergebnis = f"Status {antwort.status_code}"
    except requests.exceptions.RequestException as fehler:
        ergebnis = type(fehler).__name__
    print(f"{beschreibung:28} -> {ergebnis:12} nach {time.monotonic() - start:.1f} s")


versuche("ohne timeout", )
versuche("timeout=0.3", timeout=0.3)
versuche("timeout=(1, 0.3)", timeout=(1, 0.3))
versuche("timeout=(1, 3)", timeout=(1, 3))
```

```text
ohne timeout                 -> Status 200   nach 1.0 s
timeout=0.3                  -> ReadTimeout  nach 0.3 s
timeout=(1, 0.3)             -> ReadTimeout  nach 0.3 s
timeout=(1, 3)               -> Status 200   nach 1.0 s
```

Ohne Limit wartet das Programm die volle Sekunde und bekommt seine Antwort. Mit `timeout=0.3` gibt es nach 0,3 Sekunden auf und löst eine `ReadTimeout`-Ausnahme aus. Das Tupel `(1, 0.3)` verhält sich ebenso, weil die Verbindung sofort steht und nur das Lesen zu lange dauert; erst `(1, 3)` lässt die Antwort zu. Die Verbindungsgrenze greift in der Praxis, wenn ein Rechner gar nicht antwortet (etwa weil ein Firewall-Eintrag die Pakete wortlos verwirft); dann zeigt sich der Fehler als `ConnectTimeout`. Als Faustregel für Verwaltungsskripte sind wenige Sekunden für den Aufbau und zehn bis dreißig Sekunden für das Lesen vernünftig; wer große Datenmengen abholt, braucht mehr. Für den Verbindungsaufbau wird häufig ein Wert knapp über drei Sekunden empfohlen, weil TCP einen verlorenen ersten Verbindungswunsch standardmäßig nach etwa drei Sekunden wiederholt; diese Empfehlung ist Fachwissen und hier nicht anhand der Dokumentation geprüft.

**Fehlerarten unterscheiden.** Was kann bei einer Anfrage schiefgehen? Die Bibliothek `requests` fasst die Fehler in einer Klassenhierarchie zusammen, deren Wurzel `requests.exceptions.RequestException` ist. Darunter liegen unter anderem `ConnectionError` (kein Verbindungsaufbau: Rechner nicht erreichbar, Port geschlossen, Namensauflösung fehlgeschlagen), `Timeout` mit den Unterarten `ConnectTimeout` und `ReadTimeout`, `HTTPError` (von `raise_for_status()`, der Server antwortete mit Fehlerstatus), `TooManyRedirects`, `InvalidURL` und `MissingSchema` (die Adresse ist unbrauchbar) sowie `JSONDecodeError` (die Antwort war kein JSON). Die Reihenfolge der `except`-Zweige ist wichtig: Python nimmt den ersten passenden, und eine Unterklasse muss vor ihrer Oberklasse stehen. `ConnectTimeout` ist zugleich ein `ConnectionError` und ein `Timeout`; wer `ConnectionError` zuerst fängt, behandelt Verbindungs-Zeitüberschreitungen als Verbindungsfehler. Auch `ReadTimeout` gehört nur zu `Timeout`. Das nächste Programm klassifiziert sieben Anfragen, die jeweils anders scheitern, und gibt eine verständliche Meldung aus.

```python
import requests

import testapi

server, basis = testapi.starte()


def klassifiziere(url, **optionen):
    try:
        antwort = requests.get(url, timeout=0.5, **optionen)
        antwort.raise_for_status()
        antwort.json()
        return "ok"
    except requests.exceptions.ConnectTimeout:
        return "Verbindungs-Timeout"
    except requests.exceptions.ReadTimeout:
        return "Lese-Timeout (Server antwortet nicht rechtzeitig)"
    except requests.exceptions.ConnectionError:
        return "Verbindungsfehler (Server nicht erreichbar)"
    except requests.exceptions.HTTPError as fehler:
        return f"HTTP-Fehler {fehler.response.status_code}"
    except requests.exceptions.JSONDecodeError:
        return "Antwort ist kein JSON"
    except requests.exceptions.RequestException as fehler:
        return f"anderer requests-Fehler: {type(fehler).__name__}"


for beschreibung, url in [
    ("alles in Ordnung", basis + "/api/hosts"),
    ("Seite nicht gefunden", basis + "/api/hosts/xyz"),
    ("Server überlastet", basis + "/status/503"),
    ("langsamer Server", basis + "/slow?s=2"),
    ("HTML statt JSON", basis + "/kaputt"),
    ("niemand lauscht", "http://127.0.0.1:1/"),
    ("ungültige URL", "ohne-schema/api"),
]:
    print(f"{beschreibung:22} -> {klassifiziere(url)}")
```

```text
alles in Ordnung       -> ok
Seite nicht gefunden   -> HTTP-Fehler 404
Server überlastet      -> HTTP-Fehler 503
langsamer Server       -> Lese-Timeout (Server antwortet nicht rechtzeitig)
HTML statt JSON        -> Antwort ist kein JSON
niemand lauscht        -> Verbindungsfehler (Server nicht erreichbar)
ungültige URL          -> anderer requests-Fehler: MissingSchema
```

Jeder der Fälle landet in einem eigenen Zweig. Die HTTP-Fehler (404 und 503) kommen erst durch `raise_for_status()` zustande; ein überlasteter Server liefert `503`, aber das ist *kein* Verbindungsproblem. Der langsame Pfad löst `ReadTimeout` aus, weil das Limit mit 0,5 Sekunden kürzer ist als die zwei Sekunden Antwortzeit. Die HTML-Seite besteht die Statusprüfung (200), scheitert aber an `json()`. Auf Port 1 lauscht niemand, was ein `ConnectionError` ist, und die ungültige Adresse bleibt als `MissingSchema` im letzten Zweig hängen, weil sie keine der vorher genannten Ausnahmen auslöst. Solche Unterscheidungen sind nicht nur Kosmetik: Sie bestimmen, was das Programm als Nächstes tut, wie die nächsten Abschnitte zeigen.

**Wann wiederholen, wann nicht.** Nicht jeder Fehler verdient einen zweiten Versuch. Fehler, die von selbst verschwinden können, heißen *vorübergehend* (transient): ein überlasteter Server (`503`), ein fehlerhaftes Gateway (`502`, `504`), zu viele Anfragen (`429`), ein abgebrochener Verbindungsaufbau, eine Zeitüberschreitung. Fehler, die bei jedem Versuch wiederkehren, nennt man *dauerhaft*: ein falsches Passwort (`401`), eine nicht vorhandene Ressource (`404`), eine fehlerhafte Anfrage (`400`), ein nicht lesbarer Zertifikatsfehler. Wer ein falsches Passwort fünfmal wiederholt, erreicht nichts und sperrt womöglich noch das Konto. Eine zweite Frage betrifft die Anfrage selbst: Ist sie *idempotent*, also gefahrlos wiederholbar (siehe 9.4 und 10.1)? Ein wiederholtes `GET` schadet nicht; ein wiederholtes `POST`, das eine Bestellung oder eine Überweisung anlegt, kann zwei Bestellungen auslösen, wenn die erste Anfrage zwar ankam, die Antwort aber verloren ging. Daher sind Wiederholungen nur für idempotente Methoden die Voreinstellung.

**Wartezeiten: Backoff.** Wer nach einem Fehler sofort wieder anfragt, verschärft die Lage: Ein überlasteter Server wird mit Wiederholungen von allen Clients noch stärker belastet. Üblich ist ein *exponentieller Backoff*: Die Wartezeit verdoppelt sich nach jedem Fehlschlag (zum Beispiel 0,5 s, 1 s, 2 s, 4 s). Das gibt dem Server Luft, und bei kurzzeitigen Störungen erreicht man schnell Erfolg, während man bei längeren Störungen nicht pausenlos Anfragen schickt. Viele Clients mischen noch einen Zufallsanteil (*Jitter*) in die Wartezeit, damit nicht hundert Clients, die gleichzeitig gescheitert sind, auch gleichzeitig wieder anfragen. Und wenn der Server selbst sagt, wie lange man warten soll, nämlich mit der Kopfzeile `Retry-After` (Anzahl der Sekunden oder ein Zeitpunkt), hat diese Angabe Vorrang.

Das folgende Programm baut diese Überlegungen von Hand nach. Es versucht eine Anfrage bis zu viermal, wiederholt bei Verbindungsfehlern, Zeitüberschreitungen und den Statuscodes 429, 502, 503 und 504, wartet exponentiell mit Jitter oder, falls vorhanden, so lange wie `Retry-After` verlangt, und gibt nach dem letzten Versuch mit einer verständlichen Meldung auf. Die Wartefunktion ist als Parameter `schlafen` austauschbar; im Test sammelt sie die Pausen in einer Liste, statt wirklich zu schlafen, sodass das Beispiel sofort läuft und sich die Pausen anzeigen lassen. Der Zufallsgenerator ist mit `random.seed(1)` festgelegt, damit die Ausgabe reproduzierbar ist.

```python
import random
import time

import requests

WIEDERHOLBAR = {429, 502, 503, 504}


def hole_mit_wiederholung(url, versuche=4, basis_pause=0.5, schlafen=time.sleep, **optionen):
    """GET mit exponentiellem Backoff und Jitter. Nur für idempotente Anfragen verwenden."""
    for versuch in range(1, versuche + 1):
        try:
            antwort = requests.get(url, timeout=5, **optionen)
            if antwort.status_code not in WIEDERHOLBAR:
                return antwort
            grund = f"Status {antwort.status_code}"
            pause_vom_server = antwort.headers.get("Retry-After")
        except (requests.exceptions.ConnectionError, requests.exceptions.Timeout) as fehler:
            grund = type(fehler).__name__
            pause_vom_server = None
        if versuch == versuche:
            raise RuntimeError(f"Aufgegeben nach {versuche} Versuchen ({grund})")
        if pause_vom_server and pause_vom_server.isdigit():
            pause = int(pause_vom_server)
        else:
            pause = basis_pause * 2 ** (versuch - 1) * random.uniform(0.5, 1.5)
        print(f"  Versuch {versuch} fehlgeschlagen ({grund}), warte {pause:.2f} s")
        schlafen(pause)


if __name__ == "__main__":
    import testapi
    server, basis = testapi.starte()
    random.seed(1)
    pausen = []
    antwort = hole_mit_wiederholung(basis + "/flaky?id=x&fails=2", schlafen=pausen.append)
    print("Ergebnis:", antwort.json(), "| Pausen:", [round(p, 2) for p in pausen])
    try:
        hole_mit_wiederholung(basis + "/flaky?id=y&fails=9", versuche=3, schlafen=lambda s: None)
    except RuntimeError as fehler:
        print("Fehler:", fehler)
```

```text
  Versuch 1 fehlgeschlagen (Status 503), warte 0.32 s
  Versuch 2 fehlgeschlagen (Status 503), warte 1.35 s
Ergebnis: {'ok': True, 'versuch': 3} | Pausen: [0.32, 1.35]
  Versuch 1 fehlgeschlagen (Status 503), warte 0.63 s
  Versuch 2 fehlgeschlagen (Status 503), warte 0.76 s
Fehler: Aufgegeben nach 3 Versuchen (Status 503)
```

Bei der ersten Anfrage fallen zwei Versuche mit `503` durch, danach klappt es; die Pausen von 0,32 und 1,35 Sekunden entsprechen 0,5 s und 1,0 s mal einem Zufallsfaktor zwischen 0,5 und 1,5. Der zweite Fall scheitert dauerhaft und endet nach drei Versuchen mit der Meldung. Beachten Sie die Struktur: Die Schleife läuft von 1 bis `versuche`; beim letzten Versuch wird nicht mehr gewartet, sondern aufgegeben; und die Ausnahme wird bewusst als `RuntimeError` mit klarer Meldung ausgelöst, nicht als der ursprüngliche Netzwerkfehler (in einem richtigen Programm würde man eine eigene Ausnahmeklasse verwenden, wie wir gleich sehen).

**Wiederholungen mit urllib3.** Die Bibliothek `requests` baut auf `urllib3` auf, das fertige Wiederholungslogik mitbringt, die Klasse `urllib3.util.Retry`. Man hängt sie über einen `HTTPAdapter` an eine Sitzung, und alle Anfragen dieser Sitzung profitieren davon. Die Dokumentation von `Retry` (abgerufen) beschreibt die wichtigsten Parameter. `total` begrenzt die Gesamtzahl der Wiederholungen und hat Vorrang vor den anderen Zählern. `connect` und `read` begrenzen die Wiederholungen nach Verbindungs- beziehungsweise Lesefehlern; Lesefehler treten auf, *nachdem* die Anfrage beim Server angekommen ist, daher kann die Anfrage Nebenwirkungen gehabt haben. `status` begrenzt die Wiederholungen nach Antworten, deren Statuscode in `status_forcelist` steht; diese Liste ist standardmäßig leer, man muss sie also angeben. `backoff_factor` steuert die Wartezeit nach der dokumentierten Formel *Faktor mal 2 hoch Anzahl der bisherigen Wiederholungen*; der Vorgabewert ist 0 (keine Wartezeit), und die Obergrenze liegt standardmäßig bei 120 Sekunden. `allowed_methods` legt fest, für welche Methoden wiederholt wird; Vorgabe sind die idempotenten Methoden `DELETE`, `GET`, `HEAD`, `OPTIONS`, `PUT` und `TRACE`. `respect_retry_after_header` schließlich sorgt dafür, dass die `Retry-After`-Kopfzeile bei den Statuscodes 413, 429 und 503 beachtet wird. Mit `raise_on_status` legt man fest, ob nach erschöpften Versuchen eine Ausnahme ausgelöst (Vorgabe) oder die letzte Antwort zurückgegeben wird.

```python
import time

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

import testapi

server, basis = testapi.starte()


def sitzung_mit_retries(**retry_optionen):
    sitzung = requests.Session()
    sitzung.mount("http://", HTTPAdapter(max_retries=Retry(**retry_optionen)))
    sitzung.mount("https://", HTTPAdapter(max_retries=Retry(**retry_optionen)))
    return sitzung


s = sitzung_mit_retries(total=3, backoff_factor=0.5, status_forcelist=[502, 503, 504])
start = time.monotonic()
antwort = s.get(basis + "/flaky", params={"id": "a", "fails": 3}, timeout=5)
print("1. drei Fehlschläge, dann Erfolg:", antwort.status_code, antwort.json(), f"nach {time.monotonic() - start:.0f} s")

s = sitzung_mit_retries(total=1, status_forcelist=[503])
try:
    s.get(basis + "/flaky", params={"id": "b", "fails": 5}, timeout=5)
except requests.exceptions.RetryError as fehler:
    print("2. zu wenige Versuche:", type(fehler).__name__)

s = sitzung_mit_retries(total=1, status_forcelist=[503], raise_on_status=False)
antwort = s.get(basis + "/flaky", params={"id": "c", "fails": 5}, timeout=5)
print("3. raise_on_status=False:", antwort.status_code, "(Antwort wird zurückgegeben)")

start = time.monotonic()
s = sitzung_mit_retries(total=3, status_forcelist=[503])
antwort = s.get(basis + "/flaky", params={"id": "d", "fails": 2, "retry_after": 1}, timeout=5)
print("4. Retry-After: 1 wird beachtet:", antwort.status_code, f"nach {time.monotonic() - start:.0f} s")
```

```text
1. drei Fehlschläge, dann Erfolg: 200 {'ok': True, 'versuch': 4} nach 3 s
2. zu wenige Versuche: RetryError
3. raise_on_status=False: 503 (Antwort wird zurückgegeben)
4. Retry-After: 1 wird beachtet: 200 nach 2 s
```

Der erste Fall braucht vier Anfragen (drei Fehlschläge, ein Erfolg) und dauert drei Sekunden: Bei `backoff_factor=0.5` wartet die verwendete Version von urllib3 vor der ersten Wiederholung nicht, danach 1 und 2 Sekunden. Dass die erste Wiederholung sofort erfolgt, ist ein Verhalten der gemessenen Version (urllib3 2.8.0); die dokumentierte Formel allein würde es nicht erwarten lassen, prüfen Sie es bei Ihrer Version nach. Im zweiten Fall reichen die Versuche nicht, und die Sitzung löst `RetryError` aus. Der dritte Fall zeigt `raise_on_status=False`: Nach dem letzten Versuch erhält das Programm die Fehlerantwort (503) und kann selbst entscheiden. Im vierten Fall schickt der Server die Kopfzeile `Retry-After: 1`; die Sitzung wartet bei beiden Wiederholungen jeweils eine Sekunde und braucht deshalb zwei Sekunden, obwohl der `backoff_factor` auf null steht. Der Server bestimmt das Tempo.

Dass die Voreinstellung nur idempotente Methoden wiederholt, kann man gut sichtbar machen. Das nächste Programm schickt ein `POST` an einen Pfad, der immer `503` liefert, und zählt auf Serverseite, wie oft er aufgerufen wurde.

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

import testapi

server, basis = testapi.starte()

for erlaubt in (None, frozenset({"GET", "POST"})):
    testapi.ZAEHLER.clear()
    optionen = {"total": 3, "status_forcelist": [503], "raise_on_status": False}
    if erlaubt:
        optionen["allowed_methods"] = erlaubt
    sitzung = requests.Session()
    sitzung.mount("http://", HTTPAdapter(max_retries=Retry(**optionen)))
    antwort = sitzung.post(basis + "/flaky", json={"auftrag": 1}, timeout=5)
    print("allowed_methods:", sorted(erlaubt) if erlaubt else "Standard", "-> Server sah", testapi.ZAEHLER["post"], "Aufruf(e)")
```

```text
allowed_methods: Standard -> Server sah 1 Aufruf(e)
allowed_methods: ['GET', 'POST'] -> Server sah 4 Aufruf(e)
```

Mit der Voreinstellung bleibt es bei einem einzigen Aufruf: `POST` wird nicht wiederholt, denn der Server hätte die Anfrage womöglich schon verarbeitet. Erst wenn man `POST` ausdrücklich in `allowed_methods` aufnimmt, schickt die Sitzung vier Anfragen. Das ist nur zu empfehlen, wenn die Schnittstelle Wiederholungen sicher verarbeitet, etwa über einen mitgeschickten Idempotenzschlüssel (eine eindeutige Kennung je Auftrag, anhand derer der Server Doppelte erkennt); ob eine API das anbietet, steht in ihrer Dokumentation.

**Ein belastbarer Client.** Aus alldem lässt sich ein kleiner, wiederverwendbarer Baustein zusammensetzen. Er besteht aus einer eigenen Ausnahmeklasse `ApiFehler` mit Meldung und optionalem Statuscode, einer Funktion `neue_sitzung`, die Sitzung, Wiederholungen, `User-Agent` und optional ein Token vorbereitet, und einer Funktion `hole_json`, die eine Anfrage mit Zeitlimit stellt und alle Fehlerarten in verständliche `ApiFehler` übersetzt. Der Aufrufer muss dann nur noch eine einzige Ausnahmeart behandeln und bekommt Meldungen, die er dem Benutzer zeigen kann. Das Tupel `(3.05, 10)` als Zeitlimit folgt der oben genannten Faustregel: knapp über drei Sekunden für die Verbindung, zehn für das Lesen. Speichern Sie den Code als `apiclient.py`.

```python
from urllib.parse import urlsplit

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry


class ApiFehler(Exception):
    """Fehler beim Abruf einer API, mit verständlicher Meldung und optionalem Statuscode."""

    def __init__(self, text, status=None):
        super().__init__(text)
        self.status = status


def neue_sitzung(token=None, user_agent="apiclient/1.0", wiederholungen=3):
    sitzung = requests.Session()
    retry = Retry(total=wiederholungen, backoff_factor=0.5, status_forcelist=(429, 502, 503, 504))
    adapter = HTTPAdapter(max_retries=retry)
    sitzung.mount("http://", adapter)
    sitzung.mount("https://", adapter)
    sitzung.headers["User-Agent"] = user_agent
    if token:
        sitzung.headers["Authorization"] = f"Bearer {token}"
    return sitzung


def hole_json(sitzung, url, **parameter):
    """GET-Anfrage, die JSON liefert oder einen ApiFehler auslöst."""
    try:
        antwort = sitzung.get(url, params=parameter or None, timeout=(3.05, 10))
        antwort.raise_for_status()
        return antwort.json()
    except requests.exceptions.HTTPError as fehler:
        status = fehler.response.status_code
        if status == 401:
            raise ApiFehler("Anmeldung abgelehnt (401): Token prüfen", status) from None
        if status == 404:
            raise ApiFehler(f"Nicht gefunden (404): {urlsplit(url).path}", status) from None
        raise ApiFehler(f"Server meldet Status {status}", status) from None
    except requests.exceptions.RetryError:
        raise ApiFehler("Server dauerhaft überlastet (Wiederholungen aufgebraucht)") from None
    except requests.exceptions.Timeout:
        raise ApiFehler("Zeitüberschreitung: Server antwortet nicht rechtzeitig") from None
    except requests.exceptions.ConnectionError:
        raise ApiFehler("Server nicht erreichbar") from None
    except requests.exceptions.JSONDecodeError:
        raise ApiFehler("Antwort ist kein gültiges JSON") from None
```

Beachten Sie die Details. Die `from None`-Anweisung unterdrückt die Anzeige der ursprünglichen Ausnahme; das macht die Meldung für den Benutzer kürzer, kostet aber Diagnoseinformation: Wenn Sie eine Fehlersuche vorbereiten wollen, lassen Sie `from None` weg oder protokollieren Sie die Ursache, bevor Sie sie ersetzen. Der Zweig für `HTTPError` unterscheidet `401` und `404` von anderen Statuscodes, weil sie unterschiedliche Maßnahmen verlangen (Token prüfen oder Adresse korrigieren). Der Zweig `RetryError` steht *vor* `Timeout` und `ConnectionError`; `RetryError` entsteht, wenn die von urllib3 angebotenen Versuche für Statuscodes erschöpft sind. Die Prüfung verwendet `urlsplit(url).path`, damit in Fehlermeldungen nur der Pfad steht und keine Adresse mit Zugangsdaten oder Token in der URL (siehe 10.4). Das Demoprogramm stellt fünf typische Fälle nach.

```python
import testapi
from apiclient import ApiFehler, hole_json, neue_sitzung

server, basis = testapi.starte()
testapi.TOKEN_PFLICHT = True

faelle = [
    ("mit gültigem Token", "geheim123", "/api/hosts/web01", {}),
    ("falsches Token", "falsch", "/api/hosts/web01", {}),
    ("unbekannter Host", "geheim123", "/api/hosts/nix", {}),
    ("HTML statt JSON", None, "/kaputt", {}),
    ("dauernd 503", None, "/flaky", {"id": "dauer", "fails": 99}),
]
for beschreibung, token, pfad, parameter in faelle:
    sitzung = neue_sitzung(token=token, wiederholungen=1)
    try:
        daten = hole_json(sitzung, basis + pfad, **parameter)
        print(f"{beschreibung:20} -> ok: {daten['name']} ({daten['status']})")
    except ApiFehler as fehler:
        print(f"{beschreibung:20} -> ApiFehler: {fehler} [Status: {fehler.status}]")
```

```text
mit gültigem Token   -> ok: web01 (up)
falsches Token       -> ApiFehler: Anmeldung abgelehnt (401): Token prüfen [Status: 401]
unbekannter Host     -> ApiFehler: Nicht gefunden (404): /api/hosts/nix [Status: 404]
HTML statt JSON      -> ApiFehler: Antwort ist kein gültiges JSON [Status: None]
dauernd 503          -> ApiFehler: Server dauerhaft überlastet (Wiederholungen aufgebraucht) [Status: None]
```

In allen Fällen erhält das aufrufende Programm eine einzige, verständliche Ausnahme. Im Fall `dauernd 503` versucht die Sitzung die eine zugelassene Wiederholung und gibt dann auf. Bei einer echten Anwendung würden Sie die Anzahl der Wiederholungen erhöhen, aber nicht ins Unendliche: Nach mehreren Minuten vergeblicher Versuche ist es besser, das Problem zu melden (Exit-Code, Alarm, siehe Modul 12), als weiterzumachen.

**Typische Fehler und Stolperfallen.** Der häufigste Fehler ist das Fehlen jeden Zeitlimits. Der zweite ist ein zu breites `except Exception:` um die Anfrage, das jeden Fehler gleich behandelt, auch Programmierfehler, und die Ursache hinter einer Standardmeldung verbirgt. Der dritte sind Wiederholungen ohne Wartezeit oder ohne Obergrenze: Das Programm schickt in Sekunden Hunderte Anfragen an einen Server, der gerade Probleme hat, und wird womöglich ausgesperrt. Der vierte betrifft Fehlerseiten: Wer Wiederholungen nur bei Ausnahmen, nicht bei Statuscodes einrichtet, wiederholt nie bei `503`, weil die Anfrage technisch erfolgreich war. Und der fünfte ist die Wiederholung nicht-idempotenter Anfragen, die zu doppelten Bestellungen, Tickets oder Löschungen führt. Das folgende Gegenüber zeigt den Unterschied bei der Fehlerbehandlung.

```python
# FALSCH: alles wird gleich behandelt, Ursache geht verloren, keine Zeitgrenze
try:
    daten = requests.get(url).json()
except Exception:
    daten = {}

# RICHTIG: Zeitgrenze, gezielte Fehler, verständliche Meldung
try:
    antwort = requests.get(url, timeout=(3.05, 10))
    antwort.raise_for_status()
    daten = antwort.json()
except requests.exceptions.Timeout:
    raise SystemExit("Server antwortet nicht rechtzeitig")
except requests.exceptions.HTTPError as fehler:
    raise SystemExit(f"Server meldet {fehler.response.status_code}")
```

Die falsche Variante liefert bei jedem Fehler, auch bei einem Tippfehler im Variablennamen, still ein leeres Dictionary, und das Programm läuft mit falschen Daten weiter. Die richtige Variante unterscheidet die erwarteten Fehler, gibt eine verständliche Meldung aus und beendet das Programm mit einem Fehlercode, den Zeitplaner und Überwachung erkennen können (Modul 6 und 9). Programmierfehler, mit denen man nicht rechnet, bleiben mit vollem Traceback sichtbar.

> **Ergänzung aus der Nachrecherche**
>
> **Dokumentation (abgerufen) und Messungen.** Die Beschreibung der Parameter von `urllib3.util.Retry` (`total`, `connect`, `read`, `status`, `backoff_factor` mit der Formel und der Obergrenze von 120 Sekunden, `status_forcelist`, `allowed_methods`, `respect_retry_after_header` mit den Statuscodes 413, 429 und 503, `raise_on_status`) stammt aus der Dokumentation von urllib3. Die Antwortzeiten (drei Sekunden bei Faktor 0,5 und drei Wiederholungen; zwei Sekunden bei `Retry-After: 1` und zwei Wiederholungen) und das Verhalten für `POST` wurden mit requests 2.34.2 und urllib3 2.8.0 selbst gemessen. Dass `requests` ohne Angabe kein Zeitlimit setzt, wurde an der Signatur der installierten Bibliothek abgelesen; die offizielle Dokumentation von `requests` war nicht abrufbar. Die Hinweise zum Idempotenzschlüssel und zum Verhalten bei Gesamtdauer-Limits sind Fachwissen.

> **Merke**
>
> Jede Anfrage braucht ein `timeout`; ohne Angabe wartet requests unbegrenzt. Das Tupel `(verbinden, lesen)` trennt beide Phasen; das Lesen-Limit ist keine Gesamtdauer. Fehlerarten unterscheiden: `ConnectionError`, `Timeout`, `HTTPError`, `JSONDecodeError`, Unterklassen vor Oberklassen fangen. Wiederholen nur bei vorübergehenden Fehlern (429, 5xx, Zeitüberschreitung) und nur idempotente Anfragen, mit exponentieller Wartezeit und Obergrenze; `Retry-After` beachten. Am Ende gibt das Programm verständlich auf, statt unendlich weiterzumachen.

> **Praxisbeispiel**
>
> In der Praxis scheitern Skripte selten am Idealfall, sondern an Nebenwirkungen: Der nächtliche Export läuft genau zu dem Zeitpunkt, an dem der Dienst neu startet; der Anbieter begrenzt auf 60 Anfragen pro Minute und antwortet beim 61. mit `429`; ein Gateway meldet kurz `502`. Mit einer Sitzung, die vorübergehende Fehler selbstständig wiederholt, und einem Client, der alle anderen Fehler verständlich meldet, überstehen Ihre Skripte diese Störungen ohne Eingriff und melden sich nur, wenn wirklich etwas kaputt ist.

## 10.4 Secrets und Umgebungsvariablen

Fast jede Schnittstelle, die etwas Nützliches tut, verlangt eine Anmeldung: ein Passwort, einen API-Schlüssel, ein Token, ein Zertifikat samt privatem Schlüssel. Diese Dinge nennt man *Geheimnisse* (englisch *secrets*). Sie haben eine unangenehme Eigenschaft: Wer sie kennt, *ist* für den Dienst der Besitzer, und dem Dienst ist egal, ob er sie von Ihnen oder von einem Angreifer erhält. Ein ins Netz gestellter Schlüssel wird innerhalb von Minuten gefunden und missbraucht; automatische Suchprogramme durchkämmen öffentliche Code-Verzeichnisse laufend nach genau solchen Zeichenfolgen. Die Regeln dieser Lerneinheit sind darum keine Schönheitsfragen, sondern gehören zu den wenigen Dingen, die in der Verwaltung von Systemen wirklich unverzeihlich sind, wenn man sie missachtet.

**Die Grundregel: Geheimnisse gehören nicht in den Quelltext.** Der verführerische Anfang sieht so aus: `TOKEN = "abc123..."` ganz oben im Skript. Es läuft, es ist bequem, und es ist falsch. Der Quelltext wandert in ein Versionsverwaltungssystem (Modul 13), in Sicherungen, in Tickets, in Chats; jeder, der den Code sieht, kennt dann das Token, und aus der Versionsgeschichte lässt es sich selbst nach dem Löschen wieder herauslesen. Die Webseite *The Twelve-Factor App* formuliert das als Prüfstein (abgerufen): Konfiguration, die sich zwischen Installationen unterscheidet, wozu ausdrücklich Zugangsdaten für externe Dienste zählen, muss *strikt vom Code getrennt* sein; ein Maßstab dafür ist, ob die Codebasis *jederzeit als Open Source veröffentlicht werden könnte, ohne dass Zugangsdaten preisgegeben würden*. Das OWASP Secrets Management Cheat Sheet (abgerufen) beschreibt als Ausgangsproblem, dass viele Organisationen Geheimnisse *im Klartext im Quelltext* ablegen. Wo aber sollen die Geheimnisse dann stehen? Es gibt mehrere Möglichkeiten mit unterschiedlichen Vor- und Nachteilen.

**Umgebungsvariablen.** Eine *Umgebungsvariable* ist ein Name-Wert-Paar, das ein Prozess von seinem Elternprozess erbt. Sie kennen sie aus der Shell (`PATH`, `HOME`) und aus 9.2 (die kleine Umgebung eines cron-Jobs). In Python liegen sie im Dictionary-ähnlichen Objekt `os.environ`. Der Zugriff `os.environ["NAME"]` löst einen `KeyError` aus, wenn die Variable fehlt; `os.environ.get("NAME")` liefert dann `None`, `os.environ.get("NAME", "vorgabe")` einen Standardwert. Der Wert ist *immer eine Zeichenkette*. Das führt zu einer klassischen Falle: Aus `API_VERIFY_TLS=false` wird der Text `"false"`, und `bool("false")` ist `True`, weil jede nichtleere Zeichenkette wahr ist. Zahlen und Wahrheitswerte muss man selbst umwandeln, am besten an einer einzigen Stelle des Programms. Das folgende Beispiel setzt drei Variablen selbst (im Betrieb kämen sie von außen) und zeigt die Zugriffsarten und die Falle.

```python
import os

# Zur Vorführung setzen wir zwei Variablen selbst; im Betrieb kommen sie von außen.
os.environ["API_URL"] = "https://monitoring.example.org/api"
os.environ["API_TIMEOUT"] = "7.5"
os.environ["API_VERIFY_TLS"] = "false"

print("direkt        :", os.environ["API_URL"])
print("mit Vorgabe   :", os.environ.get("API_REGION", "eu-west"))
print("fehlt (None)  :", os.environ.get("API_TOKEN"))
print("getenv        :", os.getenv("API_TIMEOUT"), "(immer ein Text)")

zeitlimit = float(os.environ.get("API_TIMEOUT", "10"))
print("als Zahl      :", zeitlimit + 0.5)

roh = os.environ.get("API_VERIFY_TLS", "true")
print("Text 'false' ist wahr:", bool(roh), "| richtig gelesen:", roh.strip().lower() in {"1", "true", "yes", "ja"})

try:
    os.environ["API_TOKEN"]
except KeyError as fehler:
    print("os.environ[...] ohne Variable:", type(fehler).__name__, fehler)
```

```text
direkt        : https://monitoring.example.org/api
mit Vorgabe   : eu-west
fehlt (None)  : None
getenv        : 7.5 (immer ein Text)
als Zahl      : 8.0
Text 'false' ist wahr: True | richtig gelesen: False
os.environ[...] ohne Variable: KeyError 'API_TOKEN'
```

Die Zeile mit `os.getenv` zeigt eine Kurzform, die `os.environ.get` entspricht. Die Umwandlung `float(...)` macht aus dem Text eine Zahl. Beim Wahrheitswert sieht man das Problem deutlich: `bool(roh)` ist wahr, obwohl in der Variable `false` steht, und erst der Vergleich mit einer Menge erlaubter Schreibweisen liefert das erwartete Ergebnis. Der letzte Zugriff löst einen `KeyError` aus, dessen Meldung (nur der Name der Variable) zu dürftig ist, um einem Benutzer zu erklären, was fehlt. Besser ist, Konfiguration *gebündelt und früh* zu lesen und Fehler verständlich zu melden.

**Warum Umgebungsvariablen, und wo ihre Grenzen liegen.** Die Twelve-Factor-Seite empfiehlt Umgebungsvariablen aus mehreren Gründen (abgerufen): Sie lassen sich zwischen Installationen ändern, ohne Code anzufassen; die Gefahr, dass sie versehentlich in die Versionsverwaltung geraten, ist kleiner als bei Konfigurationsdateien; und sie sind ein sprach- und systemunabhängiger Standard. Das OWASP-Merkblatt sieht dagegen Nachteile und rät in seiner Voreinstellung von Umgebungsvariablen *ab*, soweit andere Wege möglich sind (abgerufen): Sie seien für alle Prozesse zugänglich und könnten in Protokollen oder Speicherabzügen landen. Beide Aussagen sind richtig, sie beziehen sich auf unterschiedliche Bedrohungen, und für Verwaltungsskripte lautet das praktische Ergebnis: Umgebungsvariablen sind eine gute Übergabeschnittstelle, sind aber kein sicherer Speicher. Wo die Geheimnisse *herkommen*, ist eine eigene Frage: aus einer Datei mit strengen Rechten, aus dem Dienstmanager (siehe unten) oder aus einem Geheimnisverwaltungsprogramm.

Wie sichtbar die beiden naheliegenden Übergabewege sind, zeigt ein Versuch. Das folgende Programm startet zwei Kindprozesse, die nur schlafen. Dem ersten gibt es das Geheimnis als *Befehlszeilenargument*, dem zweiten als *Umgebungsvariable*, und liest dann aus dem Verzeichnis `/proc` (dem vom Linux-Kern bereitgestellten Abbild laufender Prozesse), was von außen zu sehen ist. Das Beispiel funktioniert nur unter Linux.

```python
import subprocess
import sys
import time
from pathlib import Path

WARTEN = "import time; time.sleep(5)"

# Variante A: Geheimnis als Befehlszeilenargument
kind_a = subprocess.Popen([sys.executable, "-c", WARTEN, "--token=s3cr3t-abc"])
# Variante B: Geheimnis in der Umgebung des Kindprozesses
kind_b = subprocess.Popen([sys.executable, "-c", WARTEN], env={"API_TOKEN": "s3cr3t-abc"})
time.sleep(0.5)

try:
    argumente = Path(f"/proc/{kind_a.pid}/cmdline").read_bytes().split(b"\0")
    print("Befehlszeile (für jeden Benutzer lesbar):", [a.decode() for a in argumente if a.startswith(b"--token")])
    umgebung = Path(f"/proc/{kind_b.pid}/environ").read_bytes().split(b"\0")
    print("Umgebung (nur Besitzer und root):", [e.decode() for e in umgebung if e.startswith(b"API_TOKEN")])
finally:
    kind_a.kill()
    kind_b.kill()
```

```text
Befehlszeile (für jeden Benutzer lesbar): ['--token=s3cr3t-abc']
Umgebung (nur Besitzer und root): ['API_TOKEN=s3cr3t-abc']
```

Die Befehlszeile eines Prozesses ist für *jeden Benutzer des Rechners* lesbar (auch `ps aux` zeigt sie an); ein Geheimnis als Argument zu übergeben (`--token=...`) ist deshalb nie ratsam. Die Umgebung liest der Kern nur dem Besitzer des Prozesses und dem Administrator vor: sicherer als die Befehlszeile, aber für root und für jeden Prozess unter demselben Benutzerkonto offen. Außerdem erben alle *Kindprozesse* die Umgebung des Elternprozesses: Startet Ihr Skript mit `subprocess` ein anderes Programm, weiß dieses ebenfalls alles, was in der Umgebung steht, es sei denn, Sie übergeben mit `env=` eine kleinere. Das ist ein ernst zu nehmender Hinweis für Skripte, die Fremdprogramme starten.

**Konfiguration bündeln und früh prüfen.** Wenn ein Skript an zehn Stellen `os.environ` liest, findet man Fehler spät und zufällig (erst wenn der Programmteil läuft). Bewährt hat sich, die Konfiguration *einmal am Programmstart* in ein Objekt zu lesen und dabei alles zu prüfen: Fehlen Pflichtwerte, bricht das Programm sofort mit einer verständlichen Meldung ab (man nennt das *fail fast*); alle fehlenden Angaben werden *zusammen* gemeldet, nicht eine nach der anderen. Zahlen und Wahrheitswerte werden hier umgewandelt. Hinzu kommt ein zweiter Weg für das Token: Statt als Umgebungsvariable kann es in einer *Datei* stehen, auf die eine zweite Variable (`API_TOKEN_FILE`) zeigt. Solche Dateien sind die Grundlage vieler Verwaltungswerkzeuge: Das Token steht nicht in der Umgebung, die Datei gehört dem Benutzer, der das Skript betreibt, und hat die Rechte `0600` (siehe 9.1); das Programm *weigert sich*, eine Datei zu lesen, die für andere zugänglich ist. Das hat zusätzlich den Vorteil, dass Dienstmanager und Container-Plattformen Geheimnisse häufig genau so bereitstellen. Die Datenklasse `Konfig` trägt `field(repr=False)` am Token, sodass es auch dann nicht auftaucht, wenn jemand das Objekt versehentlich mit `print` ausgibt oder in einem Traceback zeigt. Speichern Sie das Beispiel als `konfig.py`.

```python
import os
import stat
from dataclasses import dataclass, field
from pathlib import Path


class KonfigFehler(Exception):
    """Die Konfiguration ist unvollständig oder unsicher."""


@dataclass(frozen=True)
class Konfig:
    url: str
    token: str = field(repr=False)          # erscheint nie in print() oder in Tracebacks
    timeout: float = 10.0
    tls_pruefen: bool = True


def _token_aus_datei(pfad):
    pfad = Path(pfad)
    modus = stat.S_IMODE(pfad.stat().st_mode)
    if modus & 0o077:
        raise KonfigFehler(f"{pfad} ist für andere lesbar (Modus {modus:04o}); erwartet 0600")
    return pfad.read_text(encoding="utf-8").strip()


def lade_konfig(umgebung=os.environ):
    fehlend = [name for name in ("API_URL",) if not umgebung.get(name)]
    token = umgebung.get("API_TOKEN")
    if not token and umgebung.get("API_TOKEN_FILE"):
        token = _token_aus_datei(umgebung["API_TOKEN_FILE"])
    if not token:
        fehlend.append("API_TOKEN (oder API_TOKEN_FILE)")
    if fehlend:
        raise KonfigFehler("Fehlende Einstellungen: " + ", ".join(fehlend))
    try:
        timeout = float(umgebung.get("API_TIMEOUT", "10"))
    except ValueError:
        raise KonfigFehler(f"API_TIMEOUT ist keine Zahl: {umgebung['API_TIMEOUT']!r}") from None
    tls = umgebung.get("API_VERIFY_TLS", "true").strip().lower() not in {"0", "false", "no", "nein"}
    return Konfig(url=umgebung["API_URL"].rstrip("/"), token=token, timeout=timeout, tls_pruefen=tls)
```

Die Funktion `lade_konfig` nimmt die Umgebung als Parameter (Vorgabe: `os.environ`), wodurch sie sich mit einem einfachen Dictionary testen lässt, ohne die echte Umgebung zu verändern. Sie sammelt fehlende Pflichtangaben in der Liste `fehlend`, liest das Token direkt oder über die Datei, wandelt `API_TIMEOUT` in eine Zahl um (und meldet Fehler als `KonfigFehler`) und liest den Schalter für die Zertifikatsprüfung. `_token_aus_datei` prüft mit `stat.S_IMODE` die Rechte: `modus \& 0o077` ist ungleich null, sobald Gruppe oder andere irgendein Recht haben. Probieren wir die Fälle aus.

```python
import os
import tempfile
from pathlib import Path

from konfig import KonfigFehler, lade_konfig

print(lade_konfig({"API_URL": "https://api.example.org/", "API_TOKEN": "s3cr3t-abc", "API_TIMEOUT": "5"}))

for beschreibung, umgebung in [
    ("nichts gesetzt", {}),
    ("Timeout kaputt", {"API_URL": "https://x", "API_TOKEN": "t", "API_TIMEOUT": "schnell"}),
]:
    try:
        lade_konfig(umgebung)
    except KonfigFehler as fehler:
        print(f"{beschreibung:15} -> {fehler}")

ordner = Path(tempfile.mkdtemp())
tokendatei = ordner / "token"
tokendatei.write_text("aus-der-datei\n", encoding="utf-8")
umgebung = {"API_URL": "https://x", "API_TOKEN_FILE": str(tokendatei)}

tokendatei.chmod(0o644)
try:
    lade_konfig(umgebung)
except KonfigFehler as fehler:
    print("Datei 0644      ->", str(fehler).replace(str(ordner), "..."))

tokendatei.chmod(0o600)
print("Datei 0600      ->", lade_konfig(umgebung).token == "aus-der-datei")
```

```text
Konfig(url='https://api.example.org', timeout=5.0, tls_pruefen=True)
nichts gesetzt  -> Fehlende Einstellungen: API_URL, API_TOKEN (oder API_TOKEN_FILE)
Timeout kaputt  -> API_TIMEOUT ist keine Zahl: 'schnell'
Datei 0644      -> .../token ist für andere lesbar (Modus 0644); erwartet 0600
Datei 0600      -> True
```

Die erste Ausgabe zeigt das Konfigurationsobjekt: Das Token fehlt in der Darstellung, weil `repr=False` gesetzt ist, und der abschließende Schrägstrich der Adresse ist entfernt. Der zweite Fall meldet *beide* fehlenden Angaben auf einmal. Im dritten Fall ist `API_TIMEOUT` keine Zahl. Bei der Tokendatei lehnt das Programm die Datei mit `0644` ab und akzeptiert sie mit `0600`. In einem Programm wie `hostabfrage` in 10.5 ist `KonfigFehler` ein Grund für einen eigenen Exit-Code.

**Dateien mit Konfiguration (.env).** Bei der Entwicklung ist es lästig, jedes Mal ein halbes Dutzend Variablen zu exportieren. Eine verbreitete Gewohnheit ist darum eine Datei mit dem Namen `.env` im Projektverzeichnis, in der Zeilen der Form `NAME=wert` stehen; ein kleines Hilfsprogramm liest sie beim Start und setzt die Variablen. Die Bibliothek `python-dotenv` übernimmt das (sie ist *weiterführend* genannt und hier nicht ausgeführt). Für das Verständnis genügt ein eigener Leser mit einigen Zeilen Code. Wichtig ist die Regel, dass *bereits gesetzte Variablen Vorrang haben*: Die Datei liefert nur Standardwerte für die lokale Entwicklung, die echte Umgebung gewinnt. Die Datei selbst gehört niemals in die Versionsverwaltung (Eintrag in `.gitignore`, siehe Modul 13) und hat die Rechte `0600`.

```python
import os
from pathlib import Path


def lade_dotenv(pfad, umgebung=os.environ):
    """Liest KEY=VALUE-Zeilen. Bereits gesetzte Variablen werden nicht überschrieben."""
    gelesen = {}
    for nummer, zeile in enumerate(Path(pfad).read_text(encoding="utf-8").splitlines(), start=1):
        zeile = zeile.strip()
        if not zeile or zeile.startswith("#"):
            continue
        if "=" not in zeile:
            raise ValueError(f"{pfad}, Zeile {nummer}: '=' fehlt")
        name, _, wert = zeile.partition("=")
        wert = wert.strip()
        if len(wert) >= 2 and wert[0] == wert[-1] and wert[0] in "\"'":
            wert = wert[1:-1]
        gelesen[name.strip()] = wert
    for name, wert in gelesen.items():
        umgebung.setdefault(name, wert)
    return gelesen
```

```python
import tempfile
from pathlib import Path

from dotenv_mini import lade_dotenv

datei = Path(tempfile.mkdtemp()) / ".env"
datei.write_text(
    "# Zugang zur Monitoring-API\n"
    "API_URL=https://monitoring.example.org\n"
    'API_TOKEN="abc 123"\n'
    "\n"
    "API_TIMEOUT=5\n",
    encoding="utf-8",
)
umgebung = {"API_TIMEOUT": "30"}            # kommt von außen und hat Vorrang
lade_dotenv(datei, umgebung)
for name in sorted(umgebung):
    print(f"{name:12} = {umgebung[name]}")
```

```text
API_TIMEOUT  = 30
API_TOKEN    = abc 123
API_URL      = https://monitoring.example.org
```

Die Datei enthält einen Kommentar, eine Leerzeile, einen Wert mit Anführungszeichen (die entfernt werden) und die Angabe `API_TIMEOUT=5`. Weil die Umgebung dieses Versuchs schon `API_TIMEOUT=30` enthält, bleibt es bei 30; die anderen Werte kommen aus der Datei. Der Aufruf `umgebung.setdefault(name, wert)` ist hier das ganze Geheimnis.

**Geheimnisse erzeugen und vergleichen.** Zwei Details gehören in jeden Werkzeugkasten. Wer selbst ein Token, ein Passwort oder einen Schlüssel erzeugt, verwendet nie das Modul `random` (dessen Zahlen sind vorhersagbar), sondern das Modul `secrets`, das den kryptografisch sicheren Zufallsgenerator des Betriebssystems nutzt: `secrets.token_urlsafe(32)` liefert 32 Zufallsbytes als URL-taugliche Zeichenkette. Und wer ein eingegebenes Geheimnis mit einem erwarteten vergleicht, verwendet `hmac.compare_digest` statt `==`: Der normale Vergleich hört beim ersten abweichenden Zeichen auf, und aus der Antwortzeit lässt sich bei sehr vielen Versuchen herauslesen, wie viele Zeichen stimmten (ein sogenannter *Timing-Angriff*); `compare_digest` braucht immer gleich lang.

```python
import hmac
import secrets

token = secrets.token_urlsafe(32)
print("Länge des Tokens:", len(token), "Zeichen | nur URL-sichere Zeichen:", all(c.isalnum() or c in "-_" for c in token))
print("Zwei Tokens gleich?", token == secrets.token_urlsafe(32))

erwartet = "geheim123"
for eingabe in ("geheim123", "geheim124"):
    print(eingabe, "->", hmac.compare_digest(eingabe.encode(), erwartet.encode()))
```

```text
Länge des Tokens: 43 Zeichen | nur URL-sichere Zeichen: True
Zwei Tokens gleich? False
geheim123 -> True
geheim124 -> False
```

Das erzeugte Token ist 43 Zeichen lang (32 Bytes, in Base64 ohne Füllzeichen codiert), besteht nur aus URL-tauglichen Zeichen und ist bei jedem Aufruf ein anderes. Der Vergleich liefert ein Ergebnis wie `==`, nur eben in konstanter Zeit. Beide Bytes-Objekte werden mit `.encode()` aus den Zeichenketten gemacht, weil `compare_digest` nur Bytes oder reine ASCII-Zeichenketten vergleicht.

**Geheimnisse nicht ausgeben.** Ein oft übersehener Verlustweg sind *Protokolle*: Eine Zeile wie `log.info("Anfrage mit %s", kopfzeilen)` schreibt die Kopfzeile `Authorization: Bearer ...` samt Token in die Protokolldatei, die oft für viele lesbar ist und in Sicherungen und Auswertungssystemen landet. Die OWASP-Empfehlung ist unmissverständlich (abgerufen): Geheimnisse dürfen *nie protokolliert* werden, und wenn doch eines im Protokoll landet, braucht es ein Verfahren, es dort wieder zu entfernen. Die erste Schutzmaßnahme ist Disziplin: keine Kopfzeilen, Adressen mit Token oder ganze Konfigurationen ausgeben. Die zweite ist ein technischer Sicherheitsgurt, den Sie bei `logging` (Modul 6) einhängen können: ein *Filter*, der bekannte Geheimnisse in jeder Zeile durch `***` ersetzt.

```python
import logging
import sys

GEHEIM = {"s3cr3t-abc"}


class GeheimnisFilter(logging.Filter):
    """Ersetzt bekannte Geheimnisse in jeder Logzeile durch ***."""

    def filter(self, record):
        text = record.getMessage()
        for geheim in GEHEIM:
            text = text.replace(geheim, "***")
        record.msg, record.args = text, None
        return True


log = logging.getLogger("demo")
handler = logging.StreamHandler(sys.stdout)
handler.addFilter(GeheimnisFilter())
log.addHandler(handler)
log.setLevel(logging.INFO)

kopfzeilen = {"Authorization": "Bearer s3cr3t-abc", "Accept": "application/json"}
log.info("Sende Anfrage mit %s", kopfzeilen)
log.info("Token ist %s", "s3cr3t-abc")


def maskiert(kopfzeilen):
    return {k: ("***" if k.lower() in {"authorization", "cookie"} else v) for k, v in kopfzeilen.items()}


print("Besser gleich maskieren:", maskiert(kopfzeilen))
```

```text
Sende Anfrage mit {'Authorization': 'Bearer ***', 'Accept': 'application/json'}
Token ist ***
Besser gleich maskieren: {'Authorization': '***', 'Accept': 'application/json'}
```

Der Filter `GeheimnisFilter` bekommt jeden Protokolleintrag, bevor er ausgegeben wird, setzt die fertige Nachricht zusammen (`record.getMessage()`), ersetzt jedes bekannte Geheimnis und schreibt die bereinigte Nachricht zurück. Die beiden Protokollzeilen zeigen, dass das Token in beiden Fällen verschwindet; das Wörterbuch der Kopfzeilen wird mit seiner Darstellung in den Text eingesetzt und dabei bereinigt. Der Filter ist nur ein Netz unter dem Seil: Er kennt *nur* die Geheimnisse, die man ihm nennt, und kann beispielsweise eine abgeleitete Form (ein Hash, eine Teilfolge) nicht erkennen. Die bessere Lösung ist deshalb, Geheimnisse gar nicht erst in Protokollzeilen zu geben, wie die letzte Zeile mit der Funktion `maskiert` zeigt. Hier werden die Kopfzeilen vor der Ausgabe gefiltert und nur harmlose Einträge durchgelassen.

**Geheimnisse und die Versionsverwaltung.** Was passiert, wenn trotzdem ein Geheimnis in einem Repository landet? Das Löschen der Zeile im nächsten Commit genügt *nicht*, denn die Geschichte enthält es weiter (siehe Modul 13). Die erste und wichtigste Reaktion ist, das Geheimnis als *kompromittiert* zu behandeln: im Dienst widerrufen und ein neues ausstellen (*rotieren*). Erst danach säubert man gegebenenfalls die Geschichte. Besser ist Vorbeugung: ein Eintrag in `.gitignore` für alle Dateien mit Geheimnissen (`.env`, Tokendateien), und Werkzeuge, die vor dem Einchecken nach Geheimnissen suchen. Das OWASP-Merkblatt empfiehlt (abgerufen), *Erkennung von Geheimnissen schon auf der Ebene der Entwickler zu aktivieren*, mehr als ein Werkzeug zu verwenden und das Projekt `detect-secrets` von Yelp ausdrücklich nennt. Das folgende Programm zeigt das Prinzip in einer sehr einfachen Form: Es sucht mit regulären Ausdrücken (Modul 8) nach typischen Mustern in Quelltext. Echte Werkzeuge sind gründlicher und bringen Hunderte Muster mit; die Idee ist dieselbe.

```python
import re

MUSTER = {
    "Passwort im Klartext": re.compile(r"""(?i)\b(password|passwort|passwd|pwd)\s*[:=]\s*['"][^'"]{4,}['"]"""),
    "Token/API-Key": re.compile(r"""(?i)\b(api[_-]?key|token|secret)\s*[:=]\s*['"][A-Za-z0-9_\-]{12,}['"]"""),
    "privater Schlüssel": re.compile(r"-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----"),
}


def suche_geheimnisse(text, name="<text>"):
    for nummer, zeile in enumerate(text.splitlines(), start=1):
        for art, muster in MUSTER.items():
            if muster.search(zeile):
                yield f"{name}:{nummer}: {art}"


beispiel = '''import requests
URL = "https://api.example.org"
password = "hunter2hunter2"
API_TOKEN = os.environ["API_TOKEN"]
api_key = 'abcDEF123456xyz789'
-----BEGIN OPENSSH PRIVATE KEY-----
'''
for fund in suche_geheimnisse(beispiel, "skript.py"):
    print(fund)
```

```text
skript.py:3: Passwort im Klartext
skript.py:5: Token/API-Key
skript.py:6: privater Schlüssel
```

Die Suche findet das Passwort im Klartext (Zeile 3), den Schlüssel in einer Zuweisung (Zeile 5) und den privaten Schlüssel (Zeile 6). Die Zeile 4, die das Token aus der Umgebung liest, wird *nicht* gemeldet, denn dort steht kein Wert, sondern nur ein Zugriff. Ein solches einfaches Verfahren erzeugt auch Fehlalarme und übersieht Geheimnisse in ungewöhnlichen Formen; es ersetzt keine Sorgfalt, ist aber ein billiges Sicherheitsnetz.

**Weitere Aufbewahrungsorte.** Neben Umgebung und Datei gibt es weitere Orte, die je nach Umgebung sinnvoll sind. Dienstmanager wie systemd können Geheimnisse bereitstellen: Mit `EnvironmentFile=` liest eine Unit Variablen aus einer Datei, und `LoadCredential=` reicht Geheimnisse als Dateien an einen Dienst durch (Fachwissen, hier nicht geprüft; siehe die Dokumentation von `systemd.exec`). Die Bibliothek `keyring` spricht den Schlüsselbund des Betriebssystems an und eignet sich für Programme, die ein Mensch an seinem Arbeitsplatz startet. Zentrale Geheimnisverwaltungen (Vault, die Dienste der Cloud-Anbieter) bieten Rotation, Zugriffsrechte und Protokollierung; das OWASP-Merkblatt beschreibt solche zentralen Lösungen als Kern einer geordneten Geheimnisverwaltung (abgerufen), mit kurzlebigen Geheimnissen, regelmäßiger Erneuerung und dem Prinzip, dass niemand Zugriff auf mehr Geheimnisse haben soll, als er braucht. Für den Anfang genügt es, die Grundregeln zu befolgen: nicht im Code, nicht in der Befehlszeile, nicht im Protokoll, nicht in der Versionsverwaltung, Datei mit `0600`, kurze Lebensdauer, Zugriff nur für das, was es braucht.

**Typische Fehler und Stolperfallen.** Die häufigsten Fehler fassen sich in wenigen Sätzen zusammen. Das Geheimnis steht im Quelltext, auch wenn nur „vorübergehend“; es steht in der Befehlszeile (`--password=...`), wo `ps` es zeigt; es steht in einer Protokollzeile, einer Fehlermeldung oder einem Traceback; die `.env`-Datei steht nicht in `.gitignore`; die Tokendatei ist für alle lesbar; dasselbe Token wird für alles verwendet, statt für jeden Zweck ein eigenes mit den kleinstmöglichen Rechten zu haben; ein bekannt gewordenes Geheimnis wird gelöscht, aber nicht widerrufen. Auch Bequemlichkeit führt in die Falle: das Token ins Chatfenster kopieren, um Kollegen zu helfen, oder in eine Fehlermeldung an einen Dienstleister, die ein Ticketsystem für alle lesbar speichert. Das Gegenüber zeigt die Grundregel in zwei Zeilen.

```python
# FALSCH: Token im Quelltext, Ausgabe samt Token
TOKEN = "ghp_aBcD1234EfGh5678IjKl9012MnOp"
print("Sende Anfrage mit", {"Authorization": f"Bearer {TOKEN}"})

# RICHTIG: Token von außen, nie ausgegeben
import os
token = os.environ.get("API_TOKEN")
if not token:
    raise SystemExit("API_TOKEN ist nicht gesetzt")
print("Sende Anfrage (Token gesetzt: ja)")
```

Die falsche Variante legt das Token jedem offen, der den Quelltext oder die Ausgabe sieht. Die richtige Variante liest es aus der Umgebung, beendet das Programm mit einer klaren Meldung, falls es fehlt, und gibt nur aus, *dass* ein Token vorhanden ist, nicht seinen Wert.

> **Ergänzung aus der Nachrecherche**
>
> **Nachrecherche (abgerufen).** Die Aussagen zu Konfiguration und Umgebungsvariablen (strikte Trennung von Code, Prüfstein „jederzeit als Open Source veröffentlichbar“, Vorteile von Umgebungsvariablen) stammen von der Seite *III. Config* der Twelve-Factor App. Die Aussagen zu Klartext-Geheimnissen im Quelltext, zur Zentralisierung, Rotation und zum geringsten Zugriffsrecht, zur Empfehlung gegen Umgebungsvariablen als Speicher, zum Verbot der Protokollierung und zur Erkennung von Geheimnissen im Quelltext stammen aus dem OWASP Secrets Management Cheat Sheet. Die Hinweise zu systemd (`EnvironmentFile`, `LoadCredential`), zu `keyring` und `python-dotenv` sind Fachwissen und wurden nicht ausgeführt oder gelesen. Dies ist *keine Rechtsberatung*; für den Umgang mit personenbezogenen Zugangsdaten und Meldepflichten nach einem Datenabfluss gelten rechtliche Anforderungen (zum Beispiel die DSGVO), die hier nicht behandelt werden.

> **Merke**
>
> Geheimnisse gehören nicht in Quelltext, Befehlszeile, Protokoll oder Versionsverwaltung. Übergabe über Umgebungsvariablen oder, besser, über eine Datei mit Rechten `0600`; Konfiguration einmal am Start lesen und prüfen (fail fast). Umgebungsvariablen sind immer Text; Zahlen und Wahrheitswerte bewusst umwandeln. Zufallswerte aus `secrets`, Vergleiche mit `hmac.compare_digest`. Ein bekannt gewordenes Geheimnis wird widerrufen und ersetzt, nicht nur gelöscht.

> **Praxisbeispiel**
>
> Kaum ein Vorfall ist so häufig und so vermeidbar wie das Token im öffentlichen Repository. Im Alltag heißt das: ein Konfigurationsmodul wie `konfig.py` für jedes Skript, die Zugangsdaten in einer Datei mit `0600` unter dem Dienstkonto oder über den Dienstmanager, ein `.gitignore` von Anfang an und, wo möglich, ein Suchwerkzeug vor jedem Commit. Und wenn doch etwas durchrutscht: sofort das Geheimnis beim Dienst widerrufen, dann erst aufräumen.

## 10.5 Projekt: API-Abfrage

Der Meilenstein dieses Moduls ist ein kleines, vollständiges Kommandozeilenprogramm, das einen Monitoring-Dienst abfragt, die Hosts nach Status und Last filtert, das Ergebnis als Tabelle oder JSON ausgibt und bei Auffälligkeiten einen Exit-Code liefert, den ein Zeitplaner oder eine Überwachung auswerten kann. Es verwendet alles, was Sie in diesem Modul gelernt haben, und die Bausteine der früheren Module: Die Argumente verarbeitet `argparse` (Modul 5), die Meldungen laufen über `logging` und Exit-Codes (Modul 6), die Konfiguration kommt aus `konfig.py` (10.4), die Anfragen stellt `apiclient.py` (10.3) mit Zeitlimits und Wiederholungen. Weil diese Bausteine fertig sind, schrumpft das eigentliche Programm auf die Fachlogik.

**Anforderungen.** Das Programm `hostabfrage` soll Folgendes leisten. Es liest Adresse und Token der API aus der Umgebung oder einer Tokendatei und bricht verständlich ab, wenn sie fehlen (Exit-Code 3). Es holt *alle* Hosts, indem es durch die Seiten blättert. Es filtert optional nach Status (`--status`) und nach Mindestlast (`--cpu-ab`). Es gibt eine Tabelle oder, mit `--json`, JSON aus; die Daten gehen nach `stdout`, Meldungen nach `stderr`, damit man die Ausgabe in andere Programme weiterleiten kann (Modul 5). Es meldet Hosts, die nicht `up` sind, als Warnung und liefert mit `--streng` den Exit-Code 2. Fehler der Abfrage (falsches Token, Server nicht erreichbar) führen zu Exit-Code 1. Und das Token erscheint *nirgends* in Ausgaben oder Protokollen.

**Der Entwurf.** Das Programm gliedert sich in kleine Funktionen mit je einer Aufgabe, was es testbar macht. `alle_hosts` blättert durch die Seiten und verwendet dabei `hole_json` aus `apiclient`; alle Fehler der Bibliothek erreichen sie als `ApiFehler`. `filtere` ist ein Generator, der die Hosts nach den Wünschen des Benutzers durchlässt. `tabelle` formatiert die Ausgabe. `main` verbindet alles, erhält die Argumente als Parameter (testbar, siehe 9.5) und liefert den Exit-Code als Rückgabewert; die Umgebung ist ebenfalls ein Parameter, damit sich die Tests keine echten Variablen leihen müssen. Die Exit-Codes stehen als benannte Konstanten am Anfang.

```python
#!/usr/bin/env python3
"""hostabfrage - fragt die Host-API ab und meldet auffaellige Hosts."""
import argparse
import json
import logging
import os
import sys

from apiclient import ApiFehler, hole_json, neue_sitzung
from konfig import KonfigFehler, lade_konfig

log = logging.getLogger("hostabfrage")
EXIT_OK, EXIT_FEHLER, EXIT_WARNUNG, EXIT_KONFIG = 0, 1, 2, 3


def alle_hosts(sitzung, basis, pro_seite=3):
    seite = 1
    while seite is not None:
        daten = hole_json(sitzung, f"{basis}/api/hosts", page=seite, per_page=pro_seite)
        yield from daten["items"]
        seite = daten["next"]


def filtere(hosts, status=None, cpu_ab=None):
    for host in hosts:
        if status and host["status"] != status:
            continue
        if cpu_ab is not None and host["cpu"] < cpu_ab:
            continue
        yield host


def tabelle(hosts):
    zeilen = [f"{'Name':10} {'IP':13} {'Status':9} {'CPU %':>6}"]
    for h in hosts:
        zeilen.append(f"{h['name']:10} {h['ip']:13} {h['status']:9} {h['cpu']:6.1f}")
    return "\n".join(zeilen)


def main(argv=None, umgebung=os.environ):
    parser = argparse.ArgumentParser(description="Hosts über die Monitoring-API abfragen")
    parser.add_argument("--status", choices=["up", "down", "degraded"], help="nur Hosts mit diesem Status")
    parser.add_argument("--cpu-ab", type=float, metavar="PROZENT", help="nur Hosts mit mindestens dieser CPU-Last")
    parser.add_argument("--json", action="store_true", help="Ausgabe als JSON statt als Tabelle")
    parser.add_argument("--streng", action="store_true", help="Exit-Code 2, wenn ein Host nicht 'up' ist")
    parser.add_argument("-v", "--verbose", action="store_true")
    args = parser.parse_args(argv)
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.WARNING, format="%(levelname)s %(message)s")

    try:
        konfig = lade_konfig(umgebung)
    except KonfigFehler as fehler:
        log.error("%s", fehler)
        return EXIT_KONFIG
    log.debug("Abfrage von %s (Timeout %.1f s)", konfig.url, konfig.timeout)

    try:
        sitzung = neue_sitzung(token=konfig.token)
        alle = list(alle_hosts(sitzung, konfig.url))
    except ApiFehler as fehler:
        log.error("Abfrage fehlgeschlagen: %s", fehler)
        return EXIT_FEHLER

    treffer = list(filtere(alle, args.status, args.cpu_ab))
    print(json.dumps(treffer, indent=2) if args.json else tabelle(treffer))
    auffaellig = [h["name"] for h in alle if h["status"] != "up"]
    if auffaellig:
        log.warning("Nicht 'up': %s", ", ".join(auffaellig))
    return EXIT_WARNUNG if (auffaellig and args.streng) else EXIT_OK


if __name__ == "__main__":
    sys.exit(main())
```

Gehen wir den Ablauf von `main` durch. Zuerst definiert `argparse` die Optionen; `choices=[...]` lässt für `--status` nur gültige Werte zu und erzeugt bei einem Tippfehler eine verständliche Fehlermeldung; `type=float` wandelt die Last in eine Zahl um. Danach wird die Protokollierung eingerichtet (mit `-v` auch Debug-Meldungen). Die Konfiguration wird *vor* der ersten Anfrage geladen, und ein `KonfigFehler` endet mit dem Exit-Code 3, ohne dass überhaupt eine Verbindung aufgebaut wird. Die Debug-Zeile nennt Adresse und Zeitlimit, aber nicht das Token. Dann folgt der Abruf: Weil `alle_hosts` ein Generator ist, löst erst `list(...)` die Anfragen aus, und ein `ApiFehler` aus irgendeiner Seite wird im `except` gefangen. Die gefilterte Ausgabe geht nach `stdout`, die Warnung über die auffälligen Hosts nach `stderr` (über `log.warning`); sie bezieht sich auf *alle* Hosts, nicht nur auf die gefilterten, weil die Frage „ist etwas kaputt?“ unabhängig vom Anzeigefilter beantwortet werden soll. Die letzte Zeile entscheidet über den Exit-Code.

Das Programm testet man, wie in 9.5, indem man `main` direkt aufruft. Das Testprogramm startet den Übungsserver mit Tokenpflicht, bereitet eine Umgebung mit dem richtigen Token vor und spielt mehrere Fälle durch. Um die Tabellen von den Meldungen zu trennen, wird `stdout` in eine Zeichenkette umgeleitet; die Meldungen von `logging` gehen nach `stderr` und erscheinen in der Ausgabe nicht.

```python
import io
import json
from contextlib import redirect_stdout

import testapi
from hostabfrage import main

server, basis = testapi.starte()
testapi.TOKEN_PFLICHT = True
umgebung = {"API_URL": basis, "API_TOKEN": "geheim123"}


def lauf(beschreibung, argumente, env=umgebung):
    puffer = io.StringIO()
    with redirect_stdout(puffer):
        code = main(argumente, env)
    print(f"--- {beschreibung}: Exit-Code {code}")
    return puffer.getvalue()


print(lauf("Alle Hosts", []))
print(lauf("Nur degraded", ["--status", "degraded"]))
print(lauf("CPU ab 70 %, streng", ["--cpu-ab", "70", "--streng"]))
daten = json.loads(lauf("JSON-Ausgabe", ["--status", "down", "--json"]))
print("JSON enthält:", [h["name"] for h in daten])
lauf("falsches Token", [], {**umgebung, "API_TOKEN": "falsch"})
lauf("kein Token", [], {"API_URL": basis})
```

```text
--- Alle Hosts: Exit-Code 0
Name       IP            Status     CPU %
web01      192.0.2.11    up          23.5
web02      192.0.2.12    up          71.0
db01       192.0.2.21    up          12.8
db02       192.0.2.22    down         0.0
cache01    192.0.2.31    up           5.2
backup01   192.0.2.41    degraded    88.4
mail01     192.0.2.51    up           9.9

--- Nur degraded: Exit-Code 0
Name       IP            Status     CPU %
backup01   192.0.2.41    degraded    88.4

--- CPU ab 70 %, streng: Exit-Code 2
Name       IP            Status     CPU %
web02      192.0.2.12    up          71.0
backup01   192.0.2.41    degraded    88.4

--- JSON-Ausgabe: Exit-Code 0
JSON enthält: ['db02']
--- falsches Token: Exit-Code 1
--- kein Token: Exit-Code 3
```

Der erste Lauf zeigt alle sieben Hosts, liefert den Exit-Code 0 und gibt (auf `stderr`, hier nicht abgedruckt) die Warnung über `db02` und `backup01` aus. Der zweite filtert nach dem Status `degraded`. Der dritte kombiniert die Last-Schwelle mit `--streng`: Es erscheinen die beiden Hosts mit mindestens 70 Prozent, und weil es nicht laufende Hosts gibt, ist der Exit-Code 2. Die JSON-Ausgabe lässt sich wieder einlesen; der Test prüft, dass genau `db02` im Ergebnis steht. Bei einem falschen Token kommt die Meldung „Anmeldung abgelehnt (401): Token prüfen“ mit Exit-Code 1, bei fehlendem Token die Meldung der Konfiguration mit Exit-Code 3. In keiner dieser Ausgaben steht das Token.

**In den Betrieb nehmen.** Als Skript auf dem Server liegt das Programm an einem festen Ort (zum Beispiel `/opt/hostabfrage`) zusammen mit den Modulen `apiclient.py` und `konfig.py`, mit der Shebang-Zeile und dem Ausführungsrecht (9.1). Die Zugangsdaten kommen aus einer Datei mit den Rechten `0600`; der Aufruf könnte so aussehen (die beiden Variablen gelten nur für diesen einen Aufruf):

```text
API_URL=https://monitoring.example.org API_TOKEN_FILE=/etc/hostabfrage/token \
    ./hostabfrage.py --streng
```

Läuft er täglich als Zeitplan, trägt man ihn wie in 9.5 als systemd-Timer oder cron-Zeile ein; ein Exit-Code 2 färbt dann die Unit rot, und die Meldung zu den auffälligen Hosts steht im Journal. In Modul 12 verwenden wir genau dieses Muster weiter: Der Exit-Code und die Meldungen sind die Grundlage für das Alarmieren per Mail oder Webhook.

**Weiterentwicklung.** Die Aufgabenstellung lässt viel Spielraum, den Sie in den Übungen nutzen können. Mögliche Erweiterungen sind eine Sortierung (`--sortiere cpu`), eine Ausgabe als CSV (Modul 7), ein Speichern der Ergebnisse mit Zeitstempel (Modul 8) für Verläufe, die Abfrage *mehrerer* Dienste mit demselben Gerüst, ein Zwischenspeicher, der die Antwort für kurze Zeit behält, und eine Beschränkung der Abfragefrequenz. Das Gerüst aus Konfiguration, Client, Fachlogik und Ausgabe bleibt dabei unverändert; nur die Fachlogik wächst.

**Typische Fehler und Stolperfallen.** Aus der Erfahrung scheitern solche Programme an wenigen Stellen. Die Fachlogik wird mit der Netzwerkarbeit verwoben, sodass man nichts mehr testen kann, ohne einen Server zu haben; hier hilft die Trennung in kleine Funktionen, bei der `filtere` und `tabelle` reine Datenfunktionen sind. Das Programm liest bei jedem Lauf nur die erste Seite und merkt es nicht; Pagination gehört in einen Test. Meldungen und Daten gehen in denselben Kanal, sodass `hostabfrage --json | jq` bei der ersten Warnung zerbricht: Daten nach `stdout`, Meldungen nach `stderr`. Das Token wird in einer Debug-Zeile mit ausgegeben. Und Exit-Codes sind nicht dokumentiert, sodass niemand weiß, was 2 bedeutet; eine Zeile im Hilfetext oder im Kopf der Datei genügt. Im folgenden Gegenüber sehen Sie die Trennung der Kanäle.

```python
# FALSCH: Meldung und Daten im selben Kanal
print("Warnung: db02 ist down")
print(json.dumps(treffer))

# RICHTIG: Daten nach stdout, Meldungen ueber logging nach stderr
log.warning("db02 ist down")
print(json.dumps(treffer))
```

Im falschen Beispiel beginnt die Ausgabe mit einer Zeile, die kein JSON ist, und ein nachgeschaltetes Programm, das die Ausgabe einliest, scheitert. Im richtigen Beispiel bleibt `stdout` reine Nutzlast; wer die Meldungen sehen will, sieht sie im Terminal oder in der Protokolldatei.

> **Ergänzung aus der Nachrecherche**
>
> **Was ausgeführt wurde.** Das Programm `hostabfrage` wurde gegen den lokalen Übungsserver mit Tokenpflicht getestet (Exit-Codes 0, 2, 1 und 3 wie gezeigt). Nicht getestet wurde die Abfrage gegen einen echten Monitoring-Dienst, die Einbindung in cron oder systemd und die Verwendung von `https` mit echtem Zertifikat; der Übungsserver spricht nur unverschlüsseltes HTTP auf dem eigenen Rechner. Bei einem echten Dienst prüfen Sie Adresse, Anmeldung, Seitenaufteilung und Ratenbegrenzung anhand der Dokumentation des Anbieters.

> **Merke**
>
> Ein Programm, das eine API abfragt, trennt Konfiguration (`konfig`), Netzwerk (`apiclient`), Fachlogik (Filter, Auswertung) und Ausgabe. Alle Seiten abholen, alle Fehler verständlich melden, eigene Exit-Codes für Konfigurationsfehler, Abfragefehler und Warnungen, Daten nach `stdout`, Meldungen nach `stderr`, nie das Token ausgeben. `main` nimmt Argumente und Umgebung als Parameter und lässt sich so ohne Prozessstart testen.

> **Praxisbeispiel**
>
> Dieses Gerüst begegnet Ihnen in der Verwaltung ständig: Ein Skript fragt Zustand ab (Monitoring, Inventar, Zertifikatsablauf, Lizenzen, Tickets), vergleicht ihn mit einer Erwartung und meldet die Abweichung über Exit-Code und Protokoll. Dieselbe Struktur verwenden Sie in Modul 11 für Abfragen per SSH und in Modul 12 für den Health-Check mit Alarmierung.

## Übungsaufgaben zu Modul 10

Die Aufgaben sind nach steigender Schwierigkeit geordnet. Die Aufgaben 1 bis 4 prüfen Wissen und Verständnis, die Aufgaben 5 bis 8 verlangen eigenen Code. Lösungen mit Erklärung finden Sie im Anschluss. Für die Aufgaben 5 und 8 benötigen Sie den Übungsserver `testapi` aus 10.1.

**Aufgabe 1.** Ordnen Sie die Statuscodes `200`, `201`, `301`, `400`, `401`, `403`, `404`, `429`, `500` und `503` ihrer Klasse zu und beschreiben Sie in je einem Satz, was sie bedeuten. Bei welchen lohnt sich ein zweiter Versuch, bei welchen nicht, und warum? Welche der Methoden `GET`, `POST`, `PUT` und `DELETE` sind idempotent, und was bedeutet das für automatische Wiederholungen?

**Aufgabe 2.** Was geschieht, wenn Sie bei `requests.get` kein `timeout` angeben? Worin unterscheiden sich `timeout=5` und `timeout=(3, 10)`, und was ist das Lesen-Limit *nicht*? Warum reicht `response.ok` nicht aus, um einen Abruf als gelungen zu betrachten, und welche Rolle hat `raise_for_status()`? In welcher Reihenfolge fängt man `ConnectTimeout`, `ConnectionError` und `RequestException`, und warum?

**Aufgabe 3.** Finden Sie mindestens sieben Fehler oder Sicherheitsprobleme im folgenden Programm und erklären Sie jeden.

```python
import logging

import requests

TOKEN = "ghp_aBcD1234EfGh5678IjKl9012MnOp"


def hole_ip(host):
    url = "https://api.example.org/hosts?name=" + host
    r = requests.get(url, headers={"Authorization": "Bearer " + TOKEN}, verify=False)
    logging.info("Antwort für %s, Anfrage-Kopfzeilen: %s", host, r.request.headers)
    return r.json()["items"][0]["ip"]


for host in ["web 01", "db&02"]:
    try:
        print(hole_ip(host))
    except:
        pass
```

**Aufgabe 4.** Ein Kollege hat versehentlich ein API-Token in ein Repository eingecheckt, das auf einem firmeninternen Server liegt, und es zwei Commits später wieder gelöscht. Was ist zu tun, in welcher Reihenfolge, und warum genügt das Löschen nicht? Nennen Sie außerdem vier weitere Orte, an denen ein Geheimnis ungewollt sichtbar werden kann, und je eine Gegenmaßnahme.

**Aufgabe 5.** Schreiben Sie eine Funktion `alle_eintraege(sitzung, url, ...)`, die durch eine seitenweise API blättert (Antwort mit `items` und `next`) und die Einträge als Generator liefert. Zeigen Sie, dass man nach drei Einträgen abbrechen kann.

**Aufgabe 6.** Schreiben Sie eine Funktion `backoff_plan(wiederholungen, faktor, maximum)`, die die Wartezeiten vor jeder Wiederholung nach der Formel *Faktor mal 2 hoch (n minus 1)* berechnet, auf das Maximum begrenzt und wahlweise die erste Wiederholung ohne Wartezeit lässt. Prüfen Sie Ihre Ergebnisse gegen den Plan von urllib3 aus 10.3 (Faktor 0,5 und drei Wiederholungen ergaben insgesamt drei Sekunden).

**Aufgabe 7.** Schreiben Sie eine Funktion `maskiere_url(url)`, die Passwörter im Benutzerteil einer URL und Werte von Abfrageparametern wie `token`, `key` oder `password` durch `***` ersetzt, damit URLs gefahrlos protokolliert werden können.

**Aufgabe 8.** Schreiben Sie eine Funktion `pruefe_urls(urls, timeout)`, die jede Adresse mit einem Zeitlimit abruft und je Adresse Status (`OK` oder `FEHLER`), einen Kurztext und die Dauer liefert. Ein Fehler bei einer Adresse darf die Prüfung der anderen nicht abbrechen. Testen Sie mit dem Übungsserver (guter Pfad, `503`, langsamer Pfad, niemand lauscht, ungültige Adresse).

## Lösungen zu Modul 10

**Aufgabe 1.** `200 OK` ist Erfolg, `201 Created` Erfolg mit Anlegen einer Ressource (beides Klasse 2xx). `301 Moved Permanently` ist eine dauerhafte Umleitung (Klasse 3xx). `400 Bad Request` (fehlerhafte Anfrage), `401 Unauthorized` (Anmeldung fehlt oder falsch), `403 Forbidden` (angemeldet, aber nicht berechtigt), `404 Not Found` (Ressource gibt es nicht) und `429 Too Many Requests` (zu viele Anfragen) gehören zur Klasse 4xx, Fehler auf Seiten des Clients. `500 Internal Server Error` und `503 Service Unavailable` sind Serverfehler (5xx). Ein zweiter Versuch lohnt sich bei den *vorübergehenden* Zuständen: `429` (nach einer Pause, am besten der von `Retry-After` genannten) und `503` (der Dienst ist überlastet oder in Wartung); bei `500` nur eingeschränkt, denn manche 500er sind dauerhaft (ein Programmfehler beim Server). Bei `400`, `401`, `403` und `404` hilft ein Wiederholen nicht, die Anfrage oder die Anmeldung muss korrigiert werden; wiederholte Anmeldeversuche können sogar zur Sperrung führen. Idempotent sind `GET`, `PUT` und `DELETE` (mehrfaches Senden hat dieselbe Wirkung wie einmaliges); `POST` ist es nicht (jeder Aufruf kann etwas Neues anlegen). Folglich wiederholen Clients wie urllib3 nur die idempotenten Methoden automatisch; ein `POST` wird nur wiederholt, wenn man das ausdrücklich zulässt und die Schnittstelle Doppelte erkennt (zum Beispiel durch einen Idempotenzschlüssel).

**Aufgabe 2.** Ohne Angabe ist das Zeitlimit `None`: `requests` wartet unbegrenzt, ein hängender Server hält das Programm für immer fest. `timeout=5` gilt für Verbindungsaufbau und Lesen; `timeout=(3, 10)` trennt beides (3 Sekunden für den Aufbau, 10 für das Warten auf Daten). Das Lesen-Limit ist *keine Gesamtdauer*: Es bezieht sich auf die Pause zwischen zwei Datenpaketen; ein Server, der tröpfchenweise sendet, kann die Anfrage deutlich länger dehnen. `response.ok` ist nur ein Hinweis auf den Statuscode (unter 400); es sagt nichts darüber, ob der Körper brauchbares JSON enthält, und einen Fehlerstatus lässt `requests` stillschweigend durch. `raise_for_status()` wandelt Fehlerstatus in einen `HTTPError` um, sodass Fehler nicht unbemerkt bleiben. Bei den Ausnahmen müssen Unterklassen vor ihren Oberklassen stehen: `ConnectTimeout` ist zugleich `ConnectionError` und `Timeout`; fängt man `ConnectionError` zuerst, erreicht eine Verbindungs-Zeitüberschreitung den Zweig für `ConnectTimeout` nie. Zuletzt steht `RequestException`, die Oberklasse von allen, als Auffangzweig für alles Übrige.

**Aufgabe 3.** Das Programm hat mehr Probleme, als man auf Anhieb sieht. (1) Das Token steht im Quelltext; es gehört in die Umgebung oder eine Datei mit `0600` (10.4). (2) Die URL wird per Zeichenkettenverkettung gebaut: Das Leerzeichen in `web 01` und das `&` in `db&02` verändern die Anfrage, es gehört `params={"name": host}` hin. (3) Es gibt kein `timeout`; ein hängender Server hängt das Programm. (4) `verify=False` schaltet die Zertifikatsprüfung ab, sodass jeder im Netz den Server spielen und das Token mitlesen kann; der Zertifikatsfehler ist zu beheben, nicht abzuschalten. (5) Es fehlt `raise_for_status()`, ein Fehlerstatus würde als Erfolg behandelt, und `.json()` scheitert womöglich an einer Fehlerseite. (6) Die Protokollzeile gibt die Kopfzeilen der Anfrage aus, also auch `Authorization: Bearer ...`; das Token landet im Protokoll. (7) `r.json()["items"][0]["ip"]` nimmt an, dass es einen Eintrag gibt und das Feld vorhanden ist; bei einer leeren Liste gibt es einen `IndexError`, bei fehlendem Feld einen `KeyError`. (8) Das nackte `except: pass` verschluckt *alle* Fehler, sogar `KeyboardInterrupt`; kein Mensch erfährt, was schiefging, und das Programm scheint zu laufen. (9) Es gibt keine Wiederholung bei vorübergehenden Fehlern und keine Sitzung, die Verbindungen wiederverwendet. Ein verbesserter Kern: `requests.get(url, params={"name": host}, headers=..., timeout=(3.05, 10))`, danach `raise_for_status()`, ein Zugriff mit Prüfung auf leere Listen, gezielte Ausnahmen mit Meldungen und das Token aus der Umgebung.

**Aufgabe 4.** Zuerst muss das Token als *kompromittiert* gelten und im Dienst *widerrufen* werden, anschließend wird ein neues ausgestellt (rotieren) und an den Stellen eingetragen, an denen es gebraucht wird. Erst danach kann man die Geschichte des Repositorys bereinigen (mit Werkzeugen, die Commits umschreiben, wobei alle Kopien des Repositorys betroffen sind) und prüfen, ob das Token benutzt wurde (Protokolle des Dienstes). Das Löschen im Folgecommit genügt nicht, weil die Versionsgeschichte jeden früheren Stand bewahrt; jeder, der das Repository geklont hat, besitzt das Token bis heute, und Sicherungen und Spiegel ebenfalls. Weitere Orte, an denen Geheimnisse sichtbar werden: die Befehlszeile (sichtbar über `ps`; Gegenmaßnahme: Übergabe über Umgebung oder Datei), die Shell-Historie (Gegenmaßnahme: Eingabe ohne Echo, Token aus Datei lesen), Protokolldateien und Fehlermeldungen (Gegenmaßnahme: nicht protokollieren, maskieren), Tracebacks und Debug-Ausgaben mit Konfigurationsobjekten (Gegenmaßnahme: `field(repr=False)`), die Umgebung geerbter Kindprozesse (Gegenmaßnahme: `env=` beim Start schmal halten), Chat und Ticketsysteme (Gegenmaßnahme: nie im Klartext einfügen, Austausch über eine Geheimnisverwaltung).

**Aufgabe 5.** Die Funktion blättert mit einer Schleife, die mit `seite = None` endet, sobald die API `next` als `null` meldet. Die Parameter `seitenschluessel`, `naechste` und `eintraege` machen sie für APIs mit anderen Feldnamen tauglich; weitere Parameter (hier `per_page`) werden durchgereicht.

```python
import requests

import testapi


def alle_eintraege(sitzung, url, seitenschluessel="page", naechste="next", eintraege="items", **parameter):
    """Blättert durch eine API, deren Antwort einen Verweis auf die nächste Seite enthält."""
    seite = 1
    while seite is not None:
        antwort = sitzung.get(url, params={**parameter, seitenschluessel: seite}, timeout=5)
        antwort.raise_for_status()
        daten = antwort.json()
        yield from daten[eintraege]
        seite = daten[naechste]


server, basis = testapi.starte()
with requests.Session() as sitzung:
    namen = [h["name"] for h in alle_eintraege(sitzung, basis + "/api/hosts", per_page=2)]
print(namen)
with requests.Session() as sitzung:
    erste_drei = []
    for host in alle_eintraege(sitzung, basis + "/api/hosts", per_page=2):
        erste_drei.append(host["name"])
        if len(erste_drei) == 3:
            break
print(erste_drei)
```

```text
['web01', 'web02', 'db01', 'db02', 'cache01', 'backup01', 'mail01']
['web01', 'web02', 'db01']
```

Der erste Aufruf liefert alle sieben Namen, indem er vier Seiten zu je zwei Einträgen holt. Der zweite bricht mit `break` nach dem dritten Eintrag ab; weil die Funktion ein Generator ist, wurden nur zwei Seiten geladen, nicht alle vier. Der `with`-Block schließt die Sitzung, auch bei einem Abbruch.

**Aufgabe 6.** Die Formel ergibt für die $n$-te Wiederholung Faktor mal 2 hoch (n minus 1); die erste Wiederholung kann ohne Wartezeit erfolgen, wie bei der gemessenen Version von urllib3.

```python
def backoff_plan(wiederholungen, faktor, maximum=120.0, erste_sofort=True):
    """Wartezeiten vor jeder Wiederholung nach der Formel faktor * 2**(n-1), gedeckelt auf maximum."""
    plan = []
    for n in range(1, wiederholungen + 1):
        if n == 1 and erste_sofort:
            plan.append(0.0)
        else:
            plan.append(min(maximum, faktor * 2 ** (n - 1)))
    return plan


print(backoff_plan(5, 0.5))
print(backoff_plan(5, 0.5, erste_sofort=False))
print(backoff_plan(9, 1.0))
print(sum(backoff_plan(5, 0.5)), "Sekunden Wartezeit insgesamt bei 5 Wiederholungen")
```

```text
[0.0, 1.0, 2.0, 4.0, 8.0]
[0.5, 1.0, 2.0, 4.0, 8.0]
[0.0, 2.0, 4.0, 8.0, 16.0, 32.0, 64.0, 120.0, 120.0]
15.0 Sekunden Wartezeit insgesamt bei 5 Wiederholungen
```

Bei Faktor 0,5 und fünf Wiederholungen sind es 0, 1, 2, 4 und 8 Sekunden; mit drei Wiederholungen (0, 1, 2) ergibt die Summe die gemessenen drei Sekunden aus 10.3. Der dritte Plan zeigt die Obergrenze: Ab der achten Wiederholung wartet man nie länger als 120 Sekunden. Die Summe der Wartezeiten ist wichtig für die Planung: Fünf Wiederholungen ergeben mit Faktor 0,5 schon 15 Sekunden Wartezeit zusätzlich zur Dauer der Anfragen selbst.

**Aufgabe 7.** Die Funktion zerlegt die URL mit `urlsplit`, ersetzt ein Passwort im Netzort (`benutzer:passwort@rechner`) und die Werte geheimer Parameter, und setzt die URL wieder zusammen. `parse_qsl` liest die Parameter als Liste von Paaren; mit `keep_blank_values=True` gehen auch leere Werte nicht verloren; `urlencode(..., safe="*")` lässt das Sternchen unverändert, sodass `***` lesbar bleibt.

```python
from urllib.parse import parse_qsl, urlencode, urlsplit, urlunsplit

GEHEIM = {"token", "key", "apikey", "api_key", "password", "passwort", "secret", "access_token"}


def maskiere_url(url):
    teile = urlsplit(url)
    netzort = teile.netloc
    if "@" in netzort:
        zugang, _, rechner = netzort.rpartition("@")
        benutzer = zugang.partition(":")[0]
        netzort = f"{benutzer}:***@{rechner}" if ":" in zugang else netzort
    parameter = [(k, "***" if k.lower() in GEHEIM else v) for k, v in parse_qsl(teile.query, keep_blank_values=True)]
    return urlunsplit((teile.scheme, netzort, teile.path, urlencode(parameter, safe="*"), teile.fragment))


print(maskiere_url("https://admin:hunter2@api.example.org/v1/hosts?page=2&token=abc123&x=1"))
print(maskiere_url("https://api.example.org/v1/hosts"))
print(maskiere_url("https://api.example.org/login?password=pw&user=anna"))
print(maskiere_url("https://nur-benutzer@host.example.org/pfad"))
```

```text
https://admin:***@api.example.org/v1/hosts?page=2&token=***&x=1
https://api.example.org/v1/hosts
https://api.example.org/login?password=***&user=anna
https://nur-benutzer@host.example.org/pfad
```

Die erste URL verliert das Passwort im Netzort und den Wert von `token`, die Parameter `page` und `x` bleiben. Die zweite hat nichts Geheimes und bleibt unverändert. Bei der dritten wird `password=pw` maskiert, `user=anna` bleibt. Die vierte hat nur einen Benutzernamen ohne Passwort und bleibt ebenfalls unverändert. Ein Schönheitsfehler: Die Reihenfolge der Parameter bleibt erhalten, aber die Schreibweise der Codierung kann sich geringfügig ändern.

**Aufgabe 8.** Die Funktion verwendet eine Sitzung und fängt je Adresse die möglichen Fehler getrennt ab: `Timeout`, `ConnectionError`, zuletzt `RequestException` für alles Übrige (etwa ungültige Adressen). Die Dauer misst sie mit `time.monotonic()`, das nicht von der Systemzeit abhängt. Ein Statuscode ab 400 gilt als Fehler, aber es wird nicht abgebrochen.

```python
import time

import requests

import testapi


def pruefe_urls(urls, timeout=1.0):
    """Gibt je URL (Status, Kurztext, Sekunden) zurück. Fehler brechen die Prüfung nicht ab."""
    ergebnis = {}
    with requests.Session() as sitzung:
        for url in urls:
            start = time.monotonic()
            try:
                antwort = sitzung.get(url, timeout=timeout)
                status = "OK" if antwort.ok else "FEHLER"
                text = f"HTTP {antwort.status_code}"
            except requests.exceptions.Timeout:
                status, text = "FEHLER", "Zeitüberschreitung"
            except requests.exceptions.ConnectionError:
                status, text = "FEHLER", "nicht erreichbar"
            except requests.exceptions.RequestException as fehler:
                status, text = "FEHLER", type(fehler).__name__
            ergebnis[url] = (status, text, time.monotonic() - start)
    return ergebnis


server, basis = testapi.starte()
urls = [basis + "/api/hosts", basis + "/status/503", basis + "/slow?s=2", "http://127.0.0.1:1/", "kaputt"]
for url, (status, text, dauer) in pruefe_urls(urls, timeout=0.5).items():
    kurz = url.replace(basis, "") or "/"
    print(f"{kurz:20} {status:7} {text:20} {'schnell' if dauer < 0.4 else 'langsam'}")
```

```text
/api/hosts           OK      HTTP 200             schnell
/status/503          FEHLER  HTTP 503             schnell
/slow?s=2            FEHLER  Zeitüberschreitung   langsam
http://127.0.0.1:1/  FEHLER  nicht erreichbar     schnell
kaputt               FEHLER  MissingSchema        schnell
```

Der gute Pfad liefert `OK`, der Pfad mit Status 503 `FEHLER` mit Angabe des Codes. Der langsame Pfad (zwei Sekunden bei einem Limit von 0,5) endet mit einer Zeitüberschreitung nach rund einer halben Sekunde, was als „langsam“ gilt. Port 1 ist nicht erreichbar, und die letzte Adresse hat kein Schema und löst `MissingSchema` aus, die von `RequestException` aufgefangen wird. Kein Fehler hat die anderen Prüfungen beeinträchtigt, das war die Anforderung. Für viele Adressen würde man die Prüfungen parallel ausführen (zum Beispiel mit `concurrent.futures`); das geht über dieses Modul hinaus.
