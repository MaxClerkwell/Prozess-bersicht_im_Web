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

* Interrupt-Konfiguration

  * PCIe-Gerät liest IRQ aus Konfigurationsraum
  * Kernel-API `request_irq` bindet Handler an IRQ-Nummer
* Hardware-Signal

  * Legacy: physischer `INTx#`-Pin auf PCIe-Steckverbinder
  * MSI: MMIO-Schreibzugriff löst IRQ aus
* Handler-Ablauf

  * `netdev_irq_handler` bestätigt und deaktiviert IRQ am NIC
  * `napi_schedule` wechselt in Polling-Modus
* NAPI-Polling

  * `napi_poll(napi, budget)` zieht bis zu `budget` Pakete
  * Paketabruf via `napi_recv` → `netif_receive_skb`
  * `napi_complete` reaktiviert IRQ nach Ende
* Protokoll-Demultiplex

  * IPv4 vs. IPv6 Unterscheidung im Kernel
  * UDP: `udp4_lib_lookup` findet Socket
  * TCP: `tcp_v4_rcv` verwaltet Paket-Reihenfolge & State
* User-Space-Übergabe

  * `recvfrom(fd, buf, len, flags, &addr, &addrlen)` kopiert Daten in User-Buffer
  * Blockierendes vs. nicht-blockierendes Socket

```c
irqreturn_t netdev_irq_handler(int irq, void *dev_id) { /* ... */ }
int napi_poll(struct napi_struct *napi, int budget) { /* ... */ }
ssize_t n = recvfrom(fd, buf, len, flags, (struct sockaddr *)&addr, &addrlen);
```

---

## 2. Kommunikation via Named Pipe (FIFO) zwischen zwei C-Programmen

* Erzeugung & Berechtigungen

  * `mkfifo(path, mode)` legt FIFO im Dateisystem an
  * Modus regelt Lese-/Schreibrechte
* Öffnen der Pipe

  * `open(path, O_RDONLY)` blockiert, wenn kein Writer
  * `open(path, O_WRONLY)` blockiert, wenn kein Reader
* Datenübertragung

  * `write(fd_w, data, len)` schreibt Bytes in Kernel-Puffer
  * `read(fd_r, buf, size)` liest Bytes, EOF bei Close
* Synchronisation

  * Implizit via blockierende I/O
  * Kein zusätzliches Locking nötig

```c
mkfifo("/tmp/my_fifo", 0666);
int fd_r = open("/tmp/my_fifo", O_RDONLY);
int fd_w = open("/tmp/my_fifo", O_WRONLY);
written = write(fd_w, data, length);
read_bytes = read(fd_r, buffer, sizeof(buffer));
close(fd_w);
close(fd_r);
```

---

## 3. HTTP GET Request zwischen lokalen Prozessen

* Socket-Setup

  * `socket(AF_INET, SOCK_STREAM, 0)` erzeugt TCP-Socket
* Verbindungsaufbau

  * Drei-Wege-Handshake: `SYN` → `SYN-ACK` → `ACK`
* HTTP-Request

  * Format: `GET /path?query HTTP/1.1\r\nHost: localhost\r\n\r\n`
  * `Host`-Header für Virtual Hosting
* Datenempfang

  * `recv(fd, buf, size)` sammelt TCP-Segmente in Buffer
  * Handhabung von Teil-Reads
* Parsing

  * Kopf und Body trennen mit `"\r\n\r\n"`
  * Header-Zeilen splitten mit `"\r\n"`
  * Request-Line splitten an Leerzeichen (Methode, Pfad, Version)
  * Status-Code & Header-Felder extrahieren

```pseudocode
fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, ("127.0.0.1", 8000));
send(fd, "GET /api?msg=foo HTTP/1.1\r\nHost: localhost\r\n\r\n");
buffer = recv(fd, 8192);
close(fd);
response = HTTPParser.parse(buffer);
```

---

## 4. Apache und PHP (CGI)

* Request-Empfang

  * `ap_handle_request(r)` evaluiert URI & vhost
* Handler-Entscheidung

  * statisch vs. dynamisch (`.php` → CGI)
* CGI-Invocation

  * `fork()` + `execve("php-cgi", args, envp)`
  * Übergabe von `REQUEST_METHOD`, `QUERY_STRING`, etc.
* Ausgabe-Pooling

  * PHP schreibt auf `stdout`, Apache liest Pipe
  * In `apr_bucket`-Strukturen abgelegt
* Response-Lieferung

  * Bucket-Brigade über `ap_finalize_request_protocol(r)` versendet

```pseudocode
if ends_with(r->filename, ".php") {
  pid = fork_and_exec("php-cgi", envp);
  output = read_pipe(pid);
  build_response(output);
}
```

---

## 5. Nginx und Python (WSGI)

* Event-basierte I/O

  * `epoll` (Linux) oder `kqueue` (BSD/macOS)
* Konfigurations-Parsing

  * `server {}`-Blöcke, `location`-Direktiven
* Proxy-Pufferung

  * `proxy_buffer_size`, `proxy_buffers`
  * Timeout- und Größenlimits
* WSGI-Contract

  * `environ`-Dict mit `REQUEST_METHOD`, `PATH_INFO`, etc.
  * `start_response(status, headers)` Callback
* Response-Streaming

  * Iterator (`yield`) von Byte-Strings
  * Unterstützt `Transfer-Encoding: chunked`

```pseudocode
on_request(r):
  upstream = select_backend(r.host);
  response_iter = wsgi_app(environ, start_response);
  for chunk in response_iter: send_chunk(r, chunk);
```

---

## 6. Uvicorn und ASGI

* AsyncIO Event-Loop
* ASGI-Scope

  * `{'type': 'http', 'method': ..., 'path': ..., 'headers': ...}`
* Receive/Send

  * `event = await receive()`
  * `await send({'type': 'http.response.start', 'status': 200})`
  * `await send({'type': 'http.response.body', 'body': data})`
* Concurrency

  * `async def app(scope, receive, send)`
* Framework-Integration

  * FastAPI, Starlette, Django Channels

```pseudocode
uvicorn.run(app, host="0.0.0.0", port=8000)
```

---
