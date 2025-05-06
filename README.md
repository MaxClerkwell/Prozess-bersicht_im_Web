# Grundlagen der Webkommunikation

## Inhaltsverzeichnis

* [1. Betriebssystem-Netzwerk-Stack: Paket empfangen auf einem Port](#1-betriebssystem-netzwerk-stack-paket-empfangen-auf-einem-port)
* [2. Kommunikation via Named Pipe (FIFO) zwischen zwei C-Programmen](#2-kommunikation-via-named-pipe-fifo-zwischen-zwei-c-programmen)
* [3. HTTP GET Request zwischen lokalen Prozessen](#3-http-get-request-zwischen-lokalen-prozessen)
* [4. Apache und PHP (CGI)](#4-apache-und-php-cgi)
* [5. Nginx und Python (WSGI)](#5-nginx-und-python-wsgi)
* [6. Uvicorn und ASGI](#6-uvicorn-und-asgi)

---

## 1. Betriebssystem-Netzwerk-Stack: Paket empfangen auf einem Port

* **PCIe & IRQ**

  * IRQ (Interrupt Request): eindeutige Nummer für Hardware-Interruptquelle
  * PCIe-Gerät liest IRQ aus PCI-Konfigurationsraum via `pci_irq_vector(pdev, 0)`
  * Kernel-API `request_irq(irq, handler, flags, name, dev)` registriert Interrupt-Handler

    * Flags z.B. `IRQF_SHARED` für geteilte Leitungen
* **Hardware-Signalübertragung**

  * **Legacy Interrupts**: physische Leitung `INTA#`–`INTD#` auf PCIe-Steckverbinder

    * führt zur IO-APIC (I/O Advanced Programmable Interrupt Controller) im Chipsatz
  * **MSI (Message Signaled Interrupts)**: Interrupt als MMIO-Schreibzugriff

    * keine physische Pin-Leitung notwendig
    * konfiguriert per PCI-API (`pci_enable_msi`)
* **Interrupt-Handler (`netdev_irq_handler`)**

  * Bestätigung am NIC via `hw_ack_interrupt(dev->hw)`
  * Wechsel in Polling-Modus mit `napi_schedule(&dev->napi)`
* **NAPI (New API) Polling-Modus**

  * Ziel: Reduktion von Interrupt-Overhead bei hohem Paketaufkommen
  * Poll-Funktion: `napi_poll(struct napi_struct *napi, int budget)`

    * wiederholt: bis `budget` Pakete ziehen
    * `napi_recv(napi)` liefert `struct sk_buff *` (Kernel-Netzwerkpuffer)
    * `netif_receive_skb(skb)` übergibt Paket dem Netzwerkstack
    * wenn weniger Pakete als `budget`, `napi_complete(napi)` reaktiviert IRQ
* **Protokoll-Demultiplexing**

  * Unterscheidung zwischen IPv4 und IPv6 im Kernel
  * **UDP**: `udp4_lib_lookup(net, skb)` weist dem richtigen Socket zu
  * **TCP**: `tcp_v4_rcv(skb)` verwaltet Reihenfolge, Retransmits und Connection-State
* **Übergabe an User-Space**

  * System-Aufruf `recvfrom(fd, buf, size, flags, &addr, &addrlen)`

    * kopiert Daten aus Kernel-Puffer in User-Buffer
    * unterstützt blockierendes und nicht-blockierendes I/O (`O_NONBLOCK`)
* **Wichtige Abkürzungen**

  * **NIC**: Network Interface Controller (Netzwerkkarte)
  * **IRQ**: Interrupt Request
  * **IO-APIC**: I/O Advanced Programmable Interrupt Controller
  * **MMIO**: Memory-Mapped I/O
  * **NAPI**: New API (Kernel-Framework für poll-basiertes Netzwerkprocessing)

**Beispiel-Code:**

```c
irqreturn_t netdev_irq_handler(int irq, void *dev_id) {
    struct net_device *dev = dev_id;
    hw_ack_interrupt(dev->hw);
    napi_schedule(&dev->napi);
    return IRQ_HANDLED;
}

int napi_poll(struct napi_struct *napi, int budget) {
    int work = 0;
    while (work < budget && napi->poll) {
        struct sk_buff *skb = napi_recv(napi);
        netif_receive_skb(skb);
        work++;
    }
    if (work < budget) napi_complete(napi);
    return work;
}

ssize_t n = recvfrom(fd, buf, len, flags, (struct sockaddr *)&addr, &addrlen);
```

---

## 2. Kommunikation via Named Pipe (FIFO) zwischen zwei C-Programmen

### 2.1 Was ist eine Named Pipe (FIFO)?

Eine **Named Pipe** (auch FIFO – *First In, First Out*) ist eine spezielle Datei im Dateisystem, über die getrennte Prozesse (nicht-verwandt) Daten in Form eines Datenstroms austauschen können.

* **Unterschied zu anonymen Pipes**: Anonyme Pipes (erzeugt etwa mit `pipe()`) existieren nur im Speicher und können nur zwischen verwandten Prozessen (z. B. Eltern-Kind) geteilt werden. Eine Named Pipe hingegen liegt als Datei vor und kann von beliebigen Prozessen geöffnet werden.
* **Datenstrom**: Daten werden genau in der Reihenfolge gelesen, in der sie geschrieben wurden.

### 2.2 Erzeugung & Berechtigungen

```c
#include <sys/stat.h>   // für mkfifo()
#include <errno.h>

// FIFO erstellen:
if (mkfifo("/tmp/my_fifo", 0666) < 0) {
    if (errno != EEXIST) {
        perror("mkfifo"); // Behandeln von Fehlern
        exit(EXIT_FAILURE);
    }
}
```

* **`mkfifo(path, mode)`**
  Legt die FIFO-Datei am angegebenen `path` an.
* **`mode` (z. B. `0666`)**
  Steuert die Zugriffsrechte:

  * `0`–`7` je drei Bits für Owner/Group/Other
  * `4` = Lesen, `2` = Schreiben, `1` = Ausführen
  * `0666` erlaubt allen Nutzern Lesen und Schreiben.
* **Fehlerbehandlung**
  `mkfifo` schlägt fehl, wenn die Datei schon existiert oder das Verzeichnis nicht beschreibbar ist.

### 2.3 Öffnen der Pipe

```c
#include <fcntl.h>      // für open()
#include <unistd.h>     // für close()

// Lesender Prozess:
int fd_r = open("/tmp/my_fifo", O_RDONLY);
// Schreibender Prozess:
int fd_w = open("/tmp/my_fifo", O_WRONLY);
```

* **`O_RDONLY` / `O_WRONLY`**

  * `open(..., O_RDONLY)` blockiert, bis ein anderer Prozess die FIFO zum Schreiben öffnet.
  * `open(..., O_WRONLY)` blockiert, bis ein Lese-Ende geöffnet ist.
* **Nicht-blockierender Modus**
  Mit `O_NONBLOCK` kann man verhindern, dass `open` blockiert; in diesem Fall liefert `open` sofort `-1` mit `errno = ENXIO` (keine Gegenstelle).
* **Mehrere Leser/Schreiber**
  Du kannst die FIFO von mehreren Prozessen gleichzeitig öffnen – der Kernel kümmert sich um die Verteilung der Daten.

### 2.4 Datenübertragung

```c
ssize_t written = write(fd_w, data, length);
if (written < 0) {
    perror("write");
}

char buffer[128];
ssize_t read_bytes = read(fd_r, buffer, sizeof(buffer));
if (read_bytes < 0) {
    perror("read");
}
```

* **`write(fd, buf, count)`**
  Schreibt bis zu `count` Bytes aus `buf` in den Kernel-Puffer der FIFO.

  * Atomares Schreiben bis zur definierten Grenze `PIPE_BUF` (mindestens 512 Bytes, oft 4 KiB).
  * Größere Blöcke können in mehreren `write`-Aufrufen aufgeteilt werden.
* **`read(fd, buf, size)`**
  Liest maximal `size` Bytes in `buf` und gibt die tatsächlich gelesene Anzahl zurück.

  * 0 zurückgegeben heißt „EOF“ – alle Schreiber haben geschlossen.
* **Pufferung**
  Der Kernel puffert die Daten; solange der Puffer nicht voll ist, blockiert `write` nicht. Ist er voll, blockiert `write` bis der Leser Daten entnimmt.

### 2.5 Synchronisation & Lebensdauer

* **Implizite Synchronisation**
  Durch blockierende `open`, `read` und `write` ist kein zusätzliches Locking nötig:

  * Ein `read` wartet solange, bis Daten verfügbar sind.
  * Ein `write` wartet, bis Pufferplatz frei wird.
* **Lebensdauer der FIFO**

  * Die Datei bleibt im Dateisystem, bis sie gelöscht wird (z. B. `unlink("/tmp/my_fifo")`).
  * Prozesse können jederzeit nachträglich öffnen (solange sie Zugriffsrechte haben).
* **Mehrere Leser/Schreiber**
  Falls mehrere Leser und/oder Schreiber aktiv sind, mischt der Kernel die Datenströme im Puffer; um kontrollierte Kommunikation sicherzustellen, kann man zusätzliche Mechanismen (z. B. Semaphoren) verwenden.

---

### 2.6 Beispiel: Einfacher Reader und Writer

#### writer.c

```c
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    const char *fifo = "/tmp/my_fifo";
    mkfifo(fifo, 0666);

    int fd = open(fifo, O_WRONLY);
    if (fd < 0) { perror("open writer"); exit(1); }

    const char *msg = "Hallo von Writer\n";
    write(fd, msg, strlen(msg));
    close(fd);
    return 0;
}
```

#### reader.c

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
    const char *fifo = "/tmp/my_fifo";

    int fd = open(fifo, O_RDONLY);
    if (fd < 0) { perror("open reader"); exit(1); }

    char buf[128];
    ssize_t r;
    while ((r = read(fd, buf, sizeof(buf)-1)) > 0) {
        buf[r] = '\0';
        printf("Empfangen: %s", buf);
    }
    close(fd);
    return 0;
}
```

**Ablauf**

1. Terminal A: `gcc writer.c -o writer && ./writer`
2. Terminal B: `gcc reader.c -o reader && ./reader`
3. Der Reader wartet auf Daten, sobald der Writer schreibt, werden sie sofort angezeigt.

---

### 2.7 Wichtige Tipps und Fallstricke

* **Rechte prüfen**: Stelle sicher, dass beide Prozesse Leseberechtigung (`r`) bzw. Schreibberechtigung (`w`) auf die FIFO haben.
* **Blockierung vermeiden**: Für nicht-blockierendes Verhalten `open(..., O_NONBLOCK)` verwenden und `EAGAIN` abfangen.
* **Puffergröße beachten**: Bis `PIPE_BUF`–Bytes ist Schreiben atomar; größere Blöcke aufteilen.
* **Aufräumen**: Nach Gebrauch die FIFO per `unlink(path)` löschen, um das Dateisystem sauber zu halten.
* **Mehrere Prozesse**: Bei Bedarf explizite Synchronisation (z. B. POSIX-Semaphoren) einsetzen, um Nachrichten-Reihenfolgen oder Zugriffskonflikte zu steuern.


---

### 3. HTTP-GET-Request zwischen lokalen Prozessen

#### 3.1 Was ist ein HTTP-GET-Request?

Ein **HTTP-GET-Request** ist eine Standardanforderung im HTTP-Protokoll, mit dem ein Client (z. B. dein eigenes Programm) Daten von einem Server anfordert. Dabei sendet der Client Informationen über die angeforderte Ressource in der Request-Line und in Header-Feldern.

* **Request-Line**: Methode (`GET`), Pfad inklusive Query-Parameter (`/api?msg=foo`) und HTTP-Version (`HTTP/1.1`).
* **Header-Felder**: Zusätzliche Metadaten wie `Host`, `User-Agent`, `Accept` usw., die dem Server helfen, die Anfrage korrekt zu bearbeiten.

#### 3.2 Socket-Setup

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd < 0) {
    perror("socket");
    exit(EXIT_FAILURE);
}
```

* **`AF_INET`**: IPv4-Adressfamilie.
* **`SOCK_STREAM`**: TCP-Stream-Socket, gewährleistet zuverlässige, verbindungsorientierte Datenübertragung.
* **Rückgabewert**: Dateideskriptor (`fd`) für das Socket, oder `-1` bei Fehler.

#### 3.3 Verbindungsaufbau (Three-Way-Handshake)

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8000);
inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

if (connect(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
    perror("connect");
    close(fd);
    exit(EXIT_FAILURE);
}
```

* **`connect()`** initiiert den TCP-Handshake:

  1. Client sendet `SYN`
  2. Server antwortet mit `SYN-ACK`
  3. Client bestätigt mit `ACK`
* Nach Abschluss steht eine vollwertige TCP-Verbindung zwischen Client und Server.

#### 3.4 HTTP-Request senden

```c
const char *request =
    "GET /api?msg=foo HTTP/1.1\r\n"
    "Host: localhost:8000\r\n"
    "Connection: close\r\n"
    "\r\n";

tssize_t sent = send(fd, request, strlen(request), 0);
if (sent < 0) {
    perror("send");
}
```

* **`GET /api?msg=foo HTTP/1.1`**: Anforderungsmethode, Pfad mit Parameter und Protokollversion.
* **`Host`-Header**: Pflichtfeld in HTTP/1.1, besonders wichtig bei Virtual Hosting.
* **`Connection: close`**: Signalisiert dem Server, die Verbindung nach der Antwort zu schließen.
* **CRLF (`\r\n`)** trennt Zeilen, `\r\n\r\n` signalisiert das Ende der Header.

#### 3.5 Datenempfang und Teil-Reads

```c
char buffer[4096];
ssize_t total = 0;
while (1) {
    ssize_t n = recv(fd, buffer + total, sizeof(buffer) - total, 0);
    if (n < 0) {
        perror("recv");
        break;
    } else if (n == 0) {
        // Verbindung vom Server geschlossen
        break;
    }
    total += n;
}
```

* **`recv()`** holt TCP-Segmente aus dem Kernel-Puffer und schreibt sie in `buffer`.
* **Teil-Reads**: TCP liefert Daten in Chunks; eine einzelne `recv`-Aufruf kann weniger Bytes liefern als vom Server gesendet. Daher in einer Schleife lesen, bis `recv` 0 (EOF) zurückgibt.
* **Buffer-Größe**: Muss groß genug sein, um Header und Body aufzunehmen, oder man verwaltet dynamisch wachsende Puffer.

#### 3.6 Parsing der HTTP-Antwort

```c
// 1. Trennen von Header und Body:
char *sep = strstr(buffer, "\r\n\r\n");
if (!sep) {
    fprintf(stderr, "Ungültige Antwort: Kein Header-Body-Separator\n");
    exit(EXIT_FAILURE);
}
*sep = '\0'; // Ende der Header
char *body = sep + 4;

// 2. Aufteilen der Header-Zeilen:
char *line = strtok(buffer, "\r\n");
// Erste Zeile ist die Status-Zeile
char *status_line = line;
// Weitere Header:
while ((line = strtok(NULL, "\r\n"))) {
    // header: value
    char *colon = strchr(line, ':');
    *colon = '\0';
    char *name = line;
    char *value = colon + 2; // skip ": "
    printf("Header: %s => %s\n", name, value);
}

printf("Body:\n%s\n", body);
```

* **Separator `\r\n\r\n`** markiert das Ende der Header.
* **`strtok`** oder manuelle Suche per `strchr` / `memcmp` möglich.
* **Status-Zeile** enthält Protokollversion, Status-Code (z. B. `200`) und Status-Text.
* **Header-Felder** im Format `Name: Wert`.
* **Body**: Alles nach dem Separator; bei `Content-Length` oder Chunked-Encoding entsprechend behandeln.

---

### 3.7 Tipps und Fallstricke

* **Blocking vs. Non-Blocking**: Standardmäßig blockiert `recv`, bis Daten vorliegen. Mit `fcntl(fd, F_SETFL, O_NONBLOCK)` erhält man stattdessen `EAGAIN` bei keinem Datenangebot.
* **Timeouts**: Mit `setsockopt` SO\_RCVTIMEO/SO\_SNDTIMEO passende Timeouts setzen.
* **Buffer-Überlauf vermeiden**: Vor jedem `recv` prüfen, wieviel Platz im Puffer noch bleibt.
* **Chunked Encoding**: Komplexeres Parsing nötig, wenn der Server `Transfer-Encoding: chunked` nutzt.
* **Mehrfach-Requests**: HTTP/1.1 ermöglicht Pipelining (mehrere Requests ohne erneuten Handshake); braucht aber sorgfältiges Management der Antwortgrenzen.

---

## 4. Apache und PHP (CGI)

### 4.1 Request-Empfang durch Apache

1. **Eintrittspunkt**
   Jeder HTTP-Request trifft zunächst im Apache-Kernel-Modul ein. Moderne Apache-Installationen nutzen dafür typischerweise das "worker"- oder "event"-MPM (Multi-Processing Module).
2. **`ap_handle_request(r)`**
   Diese zentrale Funktion analysiert die eingehende URI, bestimmt den passenden Virtual Host und übersetzt die URL in einen konkreten Dateisystem-Pfad (`r->filename`).

```c
// Stark vereinfachtes Beispiel
request_rec *r = ap_make_request(...);
int status = ap_handle_request(r);
// Ab hier ist r->filename gesetzt und bereit für die Verarbeitung
```

### 4.2 Entscheidung: statisch vs. dynamisch

* **Statische Dateien**
  Liegt unter `r->filename` eine Datei wie `.html`, `.css` oder `.jpg`, liefert Apache sie direkt aus dem Dateisystem aus.
* **Dynamische Inhalte (.php)**
  Endet die Datei auf `.php`, wird die Anfrage an einen externen PHP-CGI-Prozess weitergegeben.

### 4.3 CGI-Invocation: Prozess-Erzeugung und -Start

1. **`fork()`**
   Apache erzeugt einen neuen Kindprozess.
2. **`execve("php-cgi", args, envp)`**
   Im Kindprozess wird das PHP-CGI-Programm gestartet.
3. **Umgebungsvariablen**
   Apache setzt für das PHP-Skript wichtige Variablen, z. B.:

   * `REQUEST_METHOD` (z. B. GET, POST)
   * `QUERY_STRING` (alles nach `?` in der URI)
   * `CONTENT_TYPE`, `CONTENT_LENGTH`
   * `SCRIPT_FILENAME` (vollständiger Pfad zur PHP-Datei)

```c
// Pseudocode im Kindprozess:
char *args[] = {"php-cgi", NULL};
char *envp[] = {
    "REQUEST_METHOD=GET",
    "QUERY_STRING=foo=bar",
    "SCRIPT_FILENAME=/var/www/html/index.php",
    NULL
};
execve("/usr/bin/php-cgi", args, envp);
perror("execve"); // nur bei Fehler
exit(1);
```

### 4.4 Ausgabe-Pooling: PHP → Apache

* **PHP schreibt auf stdout**
  Alles, was das PHP-Skript ausgibt (inklusive HTTP-Headern und Body), wird auf den Standardausgang geschrieben.
* **Apache liest aus der Pipe**
  Apache richtet vor dem `execve` eine Pipe ein. Der PHP-Prozess schreibt in das Schreibende, Apache liest aus dem Lesende.
* **`apr_bucket`-Strukturen**
  Eingehende Ausgabedaten sammelt Apache in Buckets, kleinen Datencontainern, die später zusammenhängend versendet werden.

### 4.5 Response-Lieferung und Aufräumen

1. **`ap_finalize_request_protocol(r)`**
   Nachdem alle Buckets gefüllt sind, verschickt Apache die **Bucket-Brigade** im richtigen HTTP-Format an den Client.
2. **Ressourcen freigeben**

   * Apache schließt die Pipe und wartet auf die Terminierung des PHP-Prozesses.
   * Der Kindprozess beendet sich automatisch nach Ausgabeende.
   * Temporäre Ressourcen oder Dateien werden automatisch freigegeben.

### 4.6 Pseudocode-Zusammenfassung

```c
if (ends_with(r->filename, ".php")) {
    pid_t pid = fork();
    if (pid == 0) {
        // Kindprozess: PHP-CGI starten
        execve("php-cgi", args, envp);
        exit(1);
    }
    // Elternprozess: Ausgabe der CGI-Anwendung einlesen
    char *output = read_from_pipe();
    build_response(r, output);
    free(output);
} else {
    // Statische Datei direkt ausliefern
    serve_static_file(r->filename);
}
```

### 4.7 Tipps

* **Performance**: Da jeder PHP-CGI-Aufruf einen neuen Prozess startet, sind PHP-FPM oder mod\_php in produktiven Umgebungen effizienter.
* **Fehlerdiagnose**: PHP- und Apache-Fehler findest du in `/var/log/apache2/error.log`.
* **Sicherheit**: Achte auf minimale Umgebungsvariablen und korrekte Datei- sowie Pipe-Berechtigungen.
* **Debugging**: Bei Bedarf in Apache `LogLevel debug` und in PHP `display_errors = On`, `error_reporting = E_ALL` setzen.


---

## 5. Nginx und Python (WSGI)

In diesem Abschnitt erfährst du, wie Nginx als Webserver Anfragen entgegennimmt und sie an eine Python-Anwendung über das WSGI-Interface weiterleitet. Wir betrachten das event-basierte I/O-Modell, das Parsen der Konfiguration, die Proxy-Pufferung, den WSGI-Contract sowie das Streaming der Antwort.

### 5.1 Event-basierte I/O

* **Epoll (Linux) / Kqueue (BSD/macOS)**

  * Nginx nutzt ein nicht-blockierendes I/O-Modell, basierend auf `epoll` oder `kqueue`. Das bedeutet, der Server kann tausende Verbindungen verwalten, ohne für jede einen eigenen Thread oder Prozess zu blockieren.
  * **Funktionsweise**: Der Kernel benachrichtigt Nginx, sobald ein Socket lesbar oder schreibbar ist (`EPOLLIN` / `EPOLLOUT`). Nginx iteriert in einer Ereignisschleife über diese Events.

```c
// Pseudocode Nginx Event-Loop
while (1) {
    events = epoll_wait(epfd, max_events, timeout);
    for (e in events) {
        if (e.events & EPOLLIN) handle_read(e.data.fd);
        if (e.events & EPOLLOUT) handle_write(e.data.fd);
    }
}
```

### 5.2 Konfigurations-Parsing

* **`nginx.conf` und `server {}`-Blöcke**

  * Eine globale Datei `nginx.conf` enthält einen oder mehrere `http {}`-Abschnitte. Darin definierst du mit `server {}`-Blöcken Hosts, Ports und SSL-Parameter.
* **`location`-Direktiven**

  * Unter `server {}` legst du mit `location /pfad/ {}` fest, welche URIs wie behandelt werden. Zum Beispiel mit `proxy_pass` an einen WSGI-Backend-Server weiterleiten.

```nginx
http {
    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://127.0.0.1:8000;
            include proxy_params;
        }
    }
}
```

### 5.3 Proxy-Pufferung

* **`proxy_buffer_size`** und **`proxy_buffers`**

  * `proxy_buffer_size 16k;` legt die Größe eines Puffers fest.
  * `proxy_buffers 4 32k;` definiert Anzahl und Größe weiterer Puffer, in denen Nginx die Antwort zwischenlegt.
* **Timeouts und Größenlimits**

  * `proxy_read_timeout`, `proxy_connect_timeout` und `proxy_send_timeout` verhindern, dass hängen bleibende Backends Nginx blockieren.
  * `client_max_body_size` begrenzt die Größe der Anfrage (z. B. für Datei-Uploads).

### 5.4 WSGI-Contract

* **`environ`-Dict**

  * Vor jedem Request baut Nginx (über einen WSGI-Server wie uWSGI oder Gunicorn) ein Dictionary namens `environ` zusammen.
  * Felder:

    * `REQUEST_METHOD`: HTTP-Methode (`GET`, `POST`, ...)
    * `PATH_INFO`: URI-Pfad ohne Query-String
    * `QUERY_STRING`: Alles hinter `?`
    * `CONTENT_TYPE`, `CONTENT_LENGTH`
    * `wsgi.input`: Dateiähnliches Objekt mit Body-Daten
* **`start_response(status, headers)`**

  * Callback-Funktion, die von der WSGI-App aufgerufen wird, um HTTP-Status und Header zu setzen.

```python
# Minimale WSGI-App
def app(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'text/plain')]
    start_response(status, headers)
    return [b"Hello from WSGI"]
```

### 5.5 Response-Streaming

* **Iterator von Byte-Strings**

  * Deine WSGI-App gibt einen Iterator (z. B. Liste oder Generator) von `bytes`-Objekten zurück.
  * Nginx bzw. der WSGI-Server liest Stück für Stück (`yield`) und sendet sie an den Client.
* **Chunked Transfer-Encoding**

  * Wenn Länge im Voraus nicht bekannt ist, nutzt Nginx automatisch `Transfer-Encoding: chunked`.

```python
def streaming_app(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    def generate():
        for i in range(5):
            yield f"Zeile {i}\n".encode()
    return generate()
```

### 5.6 Pseudocode-Zusammenfassung

```pseudocode
on_request(r):
    upstream = select_backend(r.host)
    # WSGI-Interface
    response_iter = wsgi_app(environ, start_response)
    for chunk in response_iter:
        send_chunk(r, chunk)
```

### 5.7 Tipps und Fallstricke

* **Maximale Anzahl offener Verbindungen**

  * Konfiguriere `worker_connections` und `worker_processes` passend zur Server-Hardware.
* **Keep-Alive**

  * Mit `keepalive_timeout` reduzierst du Kosten für wiederholte Handshakes.
* **Fehlerseiten**

  * Individuelle Fehlerseiten via `error_page 404 /404.html;` definieren.
* **Sicherheit**

  * Achte auf Header wie `X-Frame-Options`, `Content-Security-Policy` und SSL-Konfiguration.
* **Logging**

  * Trenne Zugriffs- (`access_log`) und Fehler-Logs (`error_log`) für bessere Analyse.


---

## 6. Uvicorn und ASGI

### 6.1 Überblick: AsyncIO und ASGI

Uvicorn ist ein schneller ASGI-Server, der auf Python’s AsyncIO-Event-Loop aufbaut. ASGI (Asynchronous Server Gateway Interface) ist der Nachfolger von WSGI und ermöglicht bidirektionale, asynchrone Kommunikation zwischen Server und Python-Anwendung.

* **AsyncIO-Event-Loop**
  AsyncIO verwaltet alle Netzwerk-Ereignisse in einer zentralen Ereignisschleife. Dadurch kann Uvicorn tausende gleichzeitiger Verbindungen mit minimalen Threads realisieren.
* **ASGI-Contract**
  Eine ASGI-App ist genau eine asynchrone Python-Funktion:

  ```python
  async def app(scope, receive, send): ...
  ```

  Dabei behandelt der `scope` eingehende Verbindungsmetadaten, `receive` liefert Events (z. B. HTTP-Request), und `send` verschickt Events (z. B. HTTP-Response).

### 6.2 Das ASGI-Scope

Der **`scope`** ist ein unveränderliches Dictionary, das beim ersten Aufruf in deine App übergeben wird. Es enthält:

* **`type`**: Art der Verbindung, z. B. `"http"`, `"websocket"`, `"lifespan"`.
* **`method`, `path`, `raw_path`**: HTTP-Methode (`GET`, `POST`), Pfad-Strings.
* **`query_string`**: URL-Parameter als `bytes`.
* **`headers`**: Liste aus `(name: bytes, value: bytes)` Tupeln.
* **`client`**, **`server`**: Tuple mit Client- bzw. Server-IP und Port.

```python
# Beispiel für scope bei GET /items?foo=bar
{
  'type': 'http',
  'method': 'GET',
  'path': '/items',
  'query_string': b'foo=bar',
  'headers': [(b'host', b'localhost:8000'), ...]
}
```

### 6.3 Events: receive() und send()

Im Inneren der App wirst du in der Regel zwei Asynchrone Funktionen nutzen:

1. **`event = await receive()`**
   Holt das nächste Input-Event, z. B. `http.request` oder `websocket.receive`.
2. **`await send(message)`**
   Schickt ein Message-Event zurück:

   * `{'type': 'http.response.start', 'status': 200, 'headers': [...]}`
   * `{'type': 'http.response.body', 'body': b'data', 'more_body': False}`

```python
# Minimaler HTTP-Handler
async def app(scope, receive, send):
    assert scope['type'] == 'http'
    # 1. Request starten
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [(b'content-type', b'text/plain')]
    })
    # 2. Body liefern
    await send({ 'type': 'http.response.body', 'body': b'Hello, world!' })
```

### 6.4 Concurrency in ASGI

Weil Uvicorn auf AsyncIO basiert, können mehrere Requests gleichzeitig bearbeitet werden, ohne dass für jede ein neuer Thread nötig ist. Deine App-Funktion darf daher `await` nutzen, um auf I/O (Datenbank, HTTP-Clients) zu warten.

```python
async def app(scope, receive, send):
    data = await fetch_from_db()  # nicht blockierend
    await send({ 'type': 'http.response.start', 'status': 200, 'headers': [] })
    await send({ 'type': 'http.response.body', 'body': data })
```

### 6.5 Framework-Integration

Viele moderne Python-Frameworks nutzen ASGI und bauen auf Uvicorn auf:

* **FastAPI**: Automatische OpenAPI-Dokumentation, Pydantic-Modelle.
* **Starlette**: Leichtgewichtiges ASGI-Toolkit.
* **Django Channels**: Erweiterung von Django für WebSockets.

```bash
# FastAPI mit Uvicorn starten
uvicorn myapp:app --host 0.0.0.0 --port 8000 --reload
```

### 6.6 Response-Streaming und WebSockets

* **Streaming**: Du kannst große Antworten stückweise senden:

  ```python
  async def stream(scope, receive, send):
      await send({ 'type': 'http.response.start', 'status': 200, 'headers': [] })
      for i in range(10):
          await send({ 'type': 'http.response.body', 'body': f'Chunk {i}\n'.encode(), 'more_body': True })
      await send({ 'type': 'http.response.body', 'body': b'', 'more_body': False })
  ```
* **WebSocket**:

  ```python
  async def websocket_app(scope, receive, send):
      assert scope['type'] == 'websocket'
      await send({'type': 'websocket.accept'})
      while True:
          event = await receive()
          if event['type'] == 'websocket.receive':
              await send({'type': 'websocket.send', 'text': event.get('text', '')})
          elif event['type'] == 'websocket.disconnect':
              break
  ```

### 6.7 Pseudocode-Zusammenfassung

```pseudocode
uvicorn.run(app, host="0.0.0.0", port=8000)

async def app(scope, receive, send):
    if scope['type'] == 'http':
        # Request verarbeiten wie oben
    elif scope['type'] == 'websocket':
        # WebSocket-Handshake und Echo
```

---
