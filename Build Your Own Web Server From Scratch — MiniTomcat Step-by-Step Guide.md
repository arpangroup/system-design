# Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide

> **Goal:** Build a small but real HTTP server — a "MiniTomcat" — from raw TCP sockets, in plain Java, that teaches you exactly what Tomcat/Jetty/Undertow do under the hood: accepting connections, parsing HTTP, running handlers on a thread pool, managing per-request state with `ThreadLocal`, pooling expensive resources, and finally exposing a Servlet-style API you can plug your own [MiniSpring](MiniSpring-Step-by-Step-Guide.md) `ApplicationContext` into — the same way real Tomcat hosts a real Spring application.
>
> This guide assumes you already have (or have read) the companion [MiniSpring-Step-by-Step-Guide.md](MiniSpring-Step-by-Step-Guide.md), which built a mini IoC container with `@Component`/`@Autowired`/`@Controller` and a tiny MVC layer running on the JDK's built-in `com.sun.net.httpserver.HttpServer`. This guide replaces that "toy" HTTP transport with a hand-built one, and explains, piece by piece, the concurrency machinery a real container needs.

---

# 1. What We Are Building

We are building **MiniTomcat** — a servlet-container-shaped HTTP server built from `java.net.ServerSocket` upward. By the end of this guide you will have:

- A raw **TCP accept loop** that speaks enough of HTTP/1.1 to be useful (request line, headers, body, keep-alive).
- A **thread pool** (`ThreadPoolExecutor`) that turns "one thread per connection" from a liability into a bounded, tunable resource.
- A **`ThreadLocal`-based request context** (`RequestContextHolder`), the same mechanism Spring's `RequestContextHolder` and Hibernate's `ThreadLocal` session both use.
- Two flavors of **connection pool**: reusable server-side sockets (HTTP keep-alive) and a client-side **JDBC connection pool** for the database your controllers talk to.
- An optional **NIO reactor** (`Selector`) and a look at **virtual threads** as the modern answer to the same scaling problem.
- A minimal **Servlet-style API** (`MiniServlet`, `MiniHttpServletRequest/Response`, filters, listeners) — the same shape as `jakarta.servlet`.
- A **`MiniDispatcherServlet`** that bridges MiniTomcat to MiniSpring's `ApplicationContext` and `HandlerMapping`, so `@GetMapping`-annotated controllers run on top of a server you wrote yourself.
- A serious answer to *"could I plug this into Spring Boot?"* — including what SPI you'd have to implement and why you probably wouldn't ship it.

```text
TCP accept loop  ---->  Thread Pool  ---->  HTTP Parser  ---->  MiniServletContainer
(ServerSocket)        (bounded workers)    (request/response)   (path -> Servlet)
                                                                        |
                                                                        v
                                                          MiniDispatcherServlet
                                                                        |
                                                                        v
                                                    MiniSpring ApplicationContext
                                                    (HandlerMapping -> @Controller)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to answer, from first principles, not from memory:

- Why does a naive "one thread per connection" server fall over at a few thousand concurrent clients, and what specifically breaks first (stack memory? context switching? file descriptors?)?
- What does `ThreadPoolExecutor` actually do when its queue is full and all workers are busy?
- Why is `ThreadLocal` the tool of choice for per-request state in a thread-pooled server, and why is it also a classic memory-leak source in exactly that environment?
- What is the difference between a "connection pool" on the server side (reusing an accepted socket across keep-alive requests) and a "connection pool" on the client side (reusing JDBC connections to a database)?
- Why does NIO's `Selector` let one thread manage thousands of idle connections, and what do you give up to get that?
- What is a Servlet, really — not "an annotated class," but the actual `service(request, response)` contract that Tomcat calls?
- What would it take, mechanically, to make Spring Boot start *your* server instead of Tomcat?

---

# 3. Why Build a Web Server From Scratch? (Interview Motivation)

> **"Design and implement a minimal HTTP server capable of handling concurrent connections efficiently. Explain how you would size a thread pool for it, how you'd avoid resource exhaustion, how you'd store per-request state safely, and how an existing MVC framework (like Spring) would run on top of it."**

This is a recurring **senior backend / infrastructure interview question** because, unlike CRUD-app questions, it forces you to reason about:

- **Concurrency primitives** (`Thread`, `Runnable`, `ExecutorService`, `synchronized`, `volatile`) instead of just calling libraries that hide them.
- **Resource bounding** — sockets, threads, and DB connections are all finite; an interviewer wants to see you reach for a *pool* and a *queue*, not `new Thread()` in a loop.
- **I/O models** — blocking vs non-blocking, and *why* a framework picks one over the other for a given workload.
- **Protocol literacy** — HTTP is "just text over a socket," and being able to parse it by hand proves you understand what `HttpServletRequest` is abstracting away.
- **Layered API design** — separating "accept bytes" from "route a request" from "run business logic," which is exactly the Tomcat → Servlet → Spring layering in a production stack.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 (LTS) | Gives us virtual threads (`Thread.ofVirtual()`) for §39–§40 alongside classic `ThreadPoolExecutor`. |
| Sockets | `java.net.ServerSocket` / `Socket` (blocking I/O) | Simplest mental model; matches how Tomcat's default (BIO/NIO) connector historically worked. |
| Non-blocking option | `java.nio.channels.Selector` | For the optional reactor phase (§37). |
| Build tool | Maven | Matches the MiniSpring companion project so both can live in one multi-module build. |
| Testing | JUnit 5 + a raw `Socket` client in tests | We are testing wire protocol behavior, not just object graphs. |
| No external HTTP/servlet libraries | — | The entire point is to see what they do for you. |

---

# 5. Project Structure

```text
minitomcat/
├── pom.xml
├── src/main/java/com/example/minitomcat/
│   ├── server/
│   │   ├── MiniTomcat.java              // accept loop + lifecycle
│   │   ├── ServerConfig.java            // port, pool size, timeouts
│   │   └── ConnectionWorker.java        // Runnable: handles one socket
│   ├── http/
│   │   ├── HttpRequest.java
│   │   ├── HttpRequestParser.java
│   │   ├── HttpResponse.java
│   │   ├── HttpMethod.java
│   │   └── HttpStatus.java
│   ├── concurrent/
│   │   ├── ServerThreadPool.java        // wraps ThreadPoolExecutor
│   │   └── RequestContextHolder.java    // ThreadLocal request context
│   ├── pool/
│   │   ├── ObjectPool.java              // generic pool interface
│   │   ├── BufferPool.java              // pooled byte[]/ByteBuffer
│   │   └── jdbc/
│   │       ├── MiniConnectionPool.java
│   │       └── PooledConnection.java
│   ├── servlet/
│   │   ├── MiniServlet.java
│   │   ├── MiniServletConfig.java
│   │   ├── MiniServletContext.java
│   │   ├── MiniHttpServletRequest.java
│   │   ├── MiniHttpServletResponse.java
│   │   ├── MiniFilter.java
│   │   └── MiniServletContainer.java    // path -> servlet routing
│   ├── bridge/
│   │   └── MiniDispatcherServlet.java   // MiniTomcat <-> MiniSpring bridge
│   └── nio/
│       └── ReactorServer.java           // optional Selector-based server
└── src/test/java/com/example/minitomcat/
    ├── HttpRequestParserTest.java
    ├── ServerThreadPoolTest.java
    └── EndToEndSocketTest.java
```

---

# 6. Phase 1 — TCP Sockets Refresher

Everything a web server does sits on top of two classes: `ServerSocket` (a listening endpoint bound to a port) and `Socket` (one accepted TCP connection). `ServerSocket.accept()` **blocks** until a client connects, then returns a `Socket` you can read/write like any stream.

```java
// The absolute minimum: bind, accept once, read nothing, write nothing, close.
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    System.out.println("Listening on :8080");
    while (true) {
        Socket client = serverSocket.accept(); // blocks here
        client.close(); // we'll actually talk to it in the next phase
    }
}
```

Three facts drive the rest of this guide:

1. `accept()` hands you a *new* `Socket` per connection — the `ServerSocket` never talks to a client directly.
2. Reading/writing a `Socket`'s streams blocks the calling thread until bytes arrive or the buffer drains.
3. Nothing stops you from calling `accept()` again immediately and handling the previous socket on another thread — that single decision is the entire subject of §13–§21.

---

# 7. Phase 2 — The Simplest Possible Server

Let's actually respond to a request, still fully single-threaded, still zero HTTP parsing — just enough to prove the socket plumbing works end to end.

```java
public class EchoServer {
    public static void main(String[] args) throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            while (true) {
                try (Socket client = serverSocket.accept();
                     BufferedReader in = new BufferedReader(
                         new InputStreamReader(client.getInputStream()));
                     OutputStream out = client.getOutputStream()) {

                    String requestLine = in.readLine(); // e.g. "GET / HTTP/1.1"
                    System.out.println("Received: " + requestLine);

                    String body = "Hello from EchoServer";
                    String response =
                        "HTTP/1.1 200 OK\r\n" +
                        "Content-Type: text/plain\r\n" +
                        "Content-Length: " + body.length() + "\r\n" +
                        "Connection: close\r\n" +
                        "\r\n" +
                        body;
                    out.write(response.getBytes(StandardCharsets.US_ASCII));
                    out.flush();
                }
            }
        }
    }
}
```

Point this at a browser (`http://localhost:8080`) and it works — because HTTP/1.1 really is just a text protocol over a socket. The `Connection: close` header matters: without it, some clients wait for more bytes because they still believe the connection is being kept alive (see §27).

---

# 8. Anatomy of an HTTP/1.1 Request

```text
GET /users/42?verbose=true HTTP/1.1\r\n        <- request line
Host: localhost:8080\r\n                        <- headers...
User-Agent: curl/8.4.0\r\n
Accept: */*\r\n
Content-Length: 0\r\n
\r\n                                            <- blank line ends headers
<optional body bytes, exactly Content-Length long>
```

| Part | Meaning |
|---|---|
| Request line | `METHOD SPACE PATH-AND-QUERY SPACE HTTP-VERSION`, terminated by `\r\n`. |
| Headers | `Name: value\r\n`, one per line, case-insensitive names, ends at an empty line. |
| Body | Only present if `Content-Length` (fixed size) or `Transfer-Encoding: chunked` (streamed) says so. |

Everything downstream — routing, `@PathVariable`, `@RequestParam` in MiniSpring — is built by parsing exactly this text.

---

# 9. Anatomy of an HTTP/1.1 Response

```text
HTTP/1.1 200 OK\r\n                             <- status line
Content-Type: application/json\r\n              <- headers
Content-Length: 27\r\n
Connection: keep-alive\r\n
\r\n
{"id":42,"name":"Alice"}                        <- body, exactly Content-Length bytes
```

The two headers that matter most for *this* guide are `Content-Length` (so the client knows exactly when the body ends without closing the socket) and `Connection` (whether the socket may be reused — see §27–§28).

---

# 10. Phase 3 — Parsing the Request Line and Headers

```java
// http/HttpRequest.java
public class HttpRequest {
    private final String method;
    private final String path;
    private final String queryString;
    private final String httpVersion;
    private final Map<String, String> headers;
    private final byte[] body;

    public HttpRequest(String method, String path, String queryString,
                        String httpVersion, Map<String, String> headers, byte[] body) {
        this.method = method;
        this.path = path;
        this.queryString = queryString;
        this.httpVersion = httpVersion;
        this.headers = headers;
        this.body = body;
    }

    public String getMethod() { return method; }
    public String getPath() { return path; }
    public String getQueryString() { return queryString; }
    public String getHeader(String name) { return headers.get(name.toLowerCase()); }
    public byte[] getBody() { return body; }
}
```

```java
// http/HttpRequestParser.java
public class HttpRequestParser {

    public HttpRequest parse(InputStream rawIn) throws IOException {
        BufferedInputStream in = new BufferedInputStream(rawIn);

        String requestLine = readLine(in);
        if (requestLine == null || requestLine.isBlank()) {
            throw new MalformedRequestException("Empty request line");
        }
        String[] parts = requestLine.split(" ");
        if (parts.length != 3) {
            throw new MalformedRequestException("Bad request line: " + requestLine);
        }
        String method = parts[0];
        String fullPath = parts[1];
        String httpVersion = parts[2];

        String path = fullPath;
        String query = "";
        int qIndex = fullPath.indexOf('?');
        if (qIndex >= 0) {
            path = fullPath.substring(0, qIndex);
            query = fullPath.substring(qIndex + 1);
        }

        Map<String, String> headers = new LinkedHashMap<>();
        String line;
        while ((line = readLine(in)) != null && !line.isEmpty()) {
            int colon = line.indexOf(':');
            if (colon < 0) continue; // tolerate malformed header line rather than fail the request
            String name = line.substring(0, colon).trim().toLowerCase();
            String value = line.substring(colon + 1).trim();
            headers.put(name, value);
        }

        byte[] body = readBody(in, headers);
        return new HttpRequest(method, path, query, httpVersion, headers, body);
    }

    /** Reads one CRLF-terminated line without over-reading into the next line — critical for keep-alive. */
    private String readLine(InputStream in) throws IOException {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        int prev = -1, curr;
        while ((curr = in.read()) != -1) {
            if (prev == '\r' && curr == '\n') {
                byte[] bytes = buffer.toByteArray();
                return new String(bytes, 0, bytes.length - 1, StandardCharsets.US_ASCII);
            }
            buffer.write(curr);
            prev = curr;
        }
        return buffer.size() == 0 ? null : buffer.toString(StandardCharsets.US_ASCII);
    }

    private byte[] readBody(InputStream in, Map<String, String> headers) throws IOException {
        String transferEncoding = headers.get("transfer-encoding");
        if ("chunked".equalsIgnoreCase(transferEncoding)) {
            return readChunkedBody(in); // see §11
        }
        String contentLengthHeader = headers.get("content-length");
        if (contentLengthHeader == null) return new byte[0];

        int contentLength = Integer.parseInt(contentLengthHeader.trim());
        byte[] body = new byte[contentLength];
        int read = 0;
        while (read < contentLength) {
            int n = in.read(body, read, contentLength - read);
            if (n == -1) throw new MalformedRequestException("Body shorter than Content-Length");
            read += n;
        }
        return body;
    }

    private byte[] readChunkedBody(InputStream in) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        while (true) {
            String sizeLine = readLine(in);
            int chunkSize = Integer.parseInt(sizeLine.trim(), 16); // chunk size is hex
            if (chunkSize == 0) {
                readLine(in); // consume the trailing empty line after the terminating 0-chunk
                break;
            }
            byte[] chunk = new byte[chunkSize];
            int read = 0;
            while (read < chunkSize) {
                int n = in.read(chunk, read, chunkSize - read);
                if (n == -1) throw new MalformedRequestException("Truncated chunk body");
                read += n;
            }
            out.write(chunk);
            readLine(in); // consume the CRLF that follows every chunk's data
        }
        return out.toByteArray();
    }
}
```

**Why hand-roll `readLine` instead of `BufferedReader.readLine()`?** A `BufferedReader` wraps an `InputStreamReader` that pulls ahead and buffers *decoded characters*, which can silently consume bytes that belong to the body (especially a binary or chunked body) before you ever ask for them. Parsing HTTP requires precise control over exactly how many raw bytes have been consumed at every step — this is the same reason real servers implement their own buffered line-reading instead of trusting `BufferedReader` past the request line.

---

# 11. Handling the Request Body: Content-Length vs Chunked

| Style | How the receiver knows where the body ends | When it's used |
|---|---|---|
| `Content-Length: N` | Read exactly N bytes. | Sender knows the full size upfront (e.g. a JSON payload already in memory). |
| `Transfer-Encoding: chunked` | Read `size\r\ndata\r\n` blocks until a `0\r\n\r\n` terminator. | Sender is streaming and doesn't know the total size yet (e.g. server-generated output). |

A request must not specify both; if it does, most servers (and we do too, by construction above) prefer `Transfer-Encoding` since it is a hop-by-hop instruction about how to *read* the wire, while `Content-Length` is only a hint about total size.

---

# 12. Phase 4 — A Minimal Working HTTP Server

Wiring the parser into the accept loop, still single-threaded:

```java
public class MiniTomcatV1 {
    public static void main(String[] args) throws IOException {
        HttpRequestParser parser = new HttpRequestParser();
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            while (true) {
                try (Socket client = serverSocket.accept()) {
                    handle(client, parser);
                } catch (IOException e) {
                    System.err.println("Connection error: " + e.getMessage());
                }
            }
        }
    }

    private static void handle(Socket client, HttpRequestParser parser) throws IOException {
        HttpRequest request = parser.parse(client.getInputStream());
        String body = "You requested " + request.getMethod() + " " + request.getPath();
        String response =
            "HTTP/1.1 200 OK\r\n" +
            "Content-Type: text/plain\r\n" +
            "Content-Length: " + body.getBytes(StandardCharsets.UTF_8).length + "\r\n" +
            "Connection: close\r\n\r\n" + body;
        client.getOutputStream().write(response.getBytes(StandardCharsets.UTF_8));
    }
}
```

Try it with `curl -v http://localhost:8080/hello` — you now have a hand-rolled HTTP/1.1 server. It is, however, completely serial: while one request is being handled, every other client sits in the OS's TCP accept backlog waiting. That's the problem the rest of the guide solves.

---

# 13. Why Single-Threaded Fails Under Load

`ServerSocket.accept()` returns one connection at a time, and if the code handling that connection is slow (a database call, a slow client uploading a large body, network jitter), the accept loop can't call `accept()` again until it's done. Two clients become impossible to serve concurrently — the second waits behind the first even though the CPU is mostly idle.

```text
Client A connects --> handle(A) takes 200ms (waiting on DB) --> accept() can't run --> Client B queues in the OS backlog
```

The backlog isn't infinite either — `new ServerSocket(port, backlog)` bounds how many pending connections the OS will hold before refusing new ones outright with a connection-reset error.

---

# 14. Phase 5 — Thread-Per-Connection Model

The obvious fix: hand each accepted socket to its own thread so `accept()` can loop immediately.

```java
public class MiniTomcatThreaded {
    public static void main(String[] args) throws IOException {
        HttpRequestParser parser = new HttpRequestParser();
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            while (true) {
                Socket client = serverSocket.accept();
                Thread worker = new Thread(() -> handleSafely(client, parser));
                worker.start(); // fire-and-forget — this is the part we fix next
            }
        }
    }

    private static void handleSafely(Socket client, HttpRequestParser parser) {
        try (client) {
            HttpRequest request = parser.parse(client.getInputStream());
            // ... build and write a response, same as §12 ...
        } catch (IOException e) {
            System.err.println("Worker failed: " + e.getMessage());
        }
    }
}
```

This genuinely fixes concurrency — many clients can now be served in parallel, bounded only by CPU cores and blocking I/O wait time. It also introduces a new failure mode: **nothing bounds how many threads get created.**

---

# 15. The Cost of Unbounded Threads

Every `new Thread()` costs real, non-trivial resources:

| Resource | Typical cost per thread | What happens at scale |
|---|---|---|
| Stack memory | ~512 KB–1 MB (JVM default `-Xss`) | 10,000 threads ≈ 5–10 GB of stack space reserved, before any request logic runs. |
| OS thread / kernel scheduling | 1 native OS thread per Java platform thread | Context-switch overhead grows superlinearly as the scheduler juggles more runnable threads than cores. |
| File descriptors | 1 per open socket | The OS has a hard `ulimit -n`; exceeding it makes `accept()` itself start throwing. |
| GC / safepoint pauses | Each live thread must reach a safepoint | More threads can lengthen stop-the-world pause coordination. |

A burst of slow or malicious clients (e.g. a client that connects and never sends a request line) can drive thread count to the thousands and take the whole JVM down with `OutOfMemoryError: unable to create new native thread` — long before CPU is actually the bottleneck. This is precisely why every real container (Tomcat, Jetty, Netty) puts a **bounded thread pool**, not raw `new Thread()`, between "socket accepted" and "handler runs."

---

# 16. Phase 6 — Introducing a Thread Pool

```java
// concurrent/ServerThreadPool.java
public class ServerThreadPool {
    private final ThreadPoolExecutor executor;

    public ServerThreadPool(int coreSize, int maxSize, int queueCapacity) {
        this.executor = new ThreadPoolExecutor(
            coreSize,
            maxSize,
            60L, TimeUnit.SECONDS,                       // idle timeout for threads above coreSize
            new ArrayBlockingQueue<>(queueCapacity),      // bounded queue — see §19
            new ThreadFactoryBuilder("mini-tomcat-worker"),
            new ThreadPoolExecutor.CallerRunsPolicy()     // rejection policy — see §19
        );
    }

    public void submit(Runnable task) {
        executor.execute(task);
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

```java
// A minimal named-thread factory so worker threads are identifiable in a thread dump.
class ThreadFactoryBuilder implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;

    ThreadFactoryBuilder(String prefix) { this.prefix = prefix; }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setDaemon(false); // non-daemon: let in-flight requests finish before JVM exit
        return t;
    }
}
```

The accept loop no longer spawns threads itself — it just hands work to the pool:

```java
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    ServerThreadPool pool = new ServerThreadPool(20, 200, 1000);
    while (true) {
        Socket client = serverSocket.accept();
        pool.submit(() -> handleSafely(client, parser));
    }
}
```

Now the *number of concurrently running handlers* is capped, no matter how many clients connect — excess work waits in the pool's queue instead of spawning unbounded OS threads.

---

# 17. ThreadPoolExecutor Internals

`ThreadPoolExecutor` (what `Executors.newFixedThreadPool()` etc. return under the hood) makes this decision on every `execute(task)` call:

```text
1. If fewer than corePoolSize threads exist -> start a new thread for this task, even if others are idle.
2. Else if the work queue has room -> enqueue the task; an existing thread will pick it up when free.
3. Else if fewer than maximumPoolSize threads exist -> start an extra ("non-core") thread for this task.
4. Else -> the pool is fully saturated: hand the task to the RejectedExecutionHandler (see §19).
```

This order surprises people: **the pool does not grow past `corePoolSize` just because the queue has items** — it only grows once the *queue is also full*. A bounded queue plus this ordering is what gives you predictable backpressure instead of silently unbounded thread growth.

---

# 18. Sizing the Pool: Core, Max, Queue, Keep-Alive

There is no universal formula, but a defensible starting point distinguishes CPU-bound from I/O-bound work:

```text
threads = number_of_cores * (1 + wait_time / compute_time)
```

| Workload | Rule of thumb | Reasoning |
|---|---|---|
| CPU-bound (hashing, serialization, template rendering) | `corePoolSize ≈ CPU cores` | More threads than cores just adds context-switch overhead with no extra throughput. |
| I/O-bound (DB calls, downstream HTTP, disk) | `corePoolSize ≈ cores * (1 + wait/compute)` | Threads spend most of their time blocked, not computing — more threads can be *in flight* concurrently. |
| `maximumPoolSize` | A hard ceiling, sized against available memory (`maxSize * stackSize`) and downstream capacity (e.g. DB pool size — no point running more request threads than your DB pool can serve). | Prevents an I/O-bound spike from still causing thread explosion. |
| `queueCapacity` | Bounded, sized to absorb short bursts (a few seconds of typical traffic) without either rejecting immediately or growing unboundedly. | An *unbounded* queue (`LinkedBlockingQueue` with no capacity) defeats `maximumPoolSize` entirely — the pool never grows past `corePoolSize` because step 3 above is never reached. |
| `keepAliveTime` | Seconds to a couple of minutes | How long a non-core thread sits idle before being reclaimed, trading thread-creation cost against idle memory. |

> **Interview trap:** "Just increase `maximumPoolSize`" is not free — it moves the bottleneck downstream (to the database, to a rate-limited API) and reintroduces the exact thread-explosion risk from §15 if the queue is also unbounded. Always size the pool *and* the queue *and* whatever the pool calls into, together.

---

# 19. Bounded Queues and Rejection Policies

When the queue is full and the pool is already at `maximumPoolSize`, `ThreadPoolExecutor` must decide what happens to the *next* submitted task — this is backpressure made concrete:

| `RejectedExecutionHandler` | Behavior | When to use it for an HTTP server |
|---|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` immediately. | You want the accept loop to know *now* that the server is saturated, e.g. to immediately return `503 Service Unavailable`. |
| `CallerRunsPolicy` | Runs the task on the *calling* thread (here, the accept loop itself). | A simple, effective self-throttle: while the accept thread is busy running an overflow request, it can't call `accept()` again, naturally slowing new connection intake. |
| `DiscardPolicy` | Silently drops the task. | Almost never appropriate for a server — a dropped HTTP request looks like a hung connection to the client. |
| `DiscardOldestPolicy` | Drops the *oldest* queued task, then retries submission. | Rarely appropriate for requests (fairness violation — a request that's been waiting longest gets punished), more defensible for a metrics/logging queue. |

For MiniTomcat, `CallerRunsPolicy` (used in §16) combined with a fast, explicit `503` fallback in the accept loop is the most instructive combination — it demonstrates real backpressure instead of hiding saturation behind an ever-growing queue.

---

# 20. Phase 7 — Wiring the Thread Pool Into the Server

```java
public class MiniTomcat {
    private final ServerSocket serverSocket;
    private final ServerThreadPool pool;
    private final HttpRequestParser parser = new HttpRequestParser();
    private volatile boolean running = true;

    public MiniTomcat(int port, int coreThreads, int maxThreads, int queueCapacity) throws IOException {
        this.serverSocket = new ServerSocket(port);
        this.pool = new ServerThreadPool(coreThreads, maxThreads, queueCapacity);
    }

    public void start() {
        Thread acceptThread = new Thread(this::acceptLoop, "mini-tomcat-acceptor");
        acceptThread.start();
    }

    private void acceptLoop() {
        while (running) {
            try {
                Socket client = serverSocket.accept();
                pool.submit(new ConnectionWorker(client, parser));
            } catch (RejectedExecutionException saturated) {
                // Pool + queue both full and CallerRunsPolicy itself blocked too long — fail fast.
                writeServiceUnavailableAndClose(/* client */ null);
            } catch (IOException e) {
                if (running) System.err.println("Accept failed: " + e.getMessage());
            }
        }
    }

    public void stop() throws IOException {
        running = false;
        serverSocket.close(); // unblocks the pending accept()
        pool.shutdown();
    }
}
```

```java
// server/ConnectionWorker.java
public class ConnectionWorker implements Runnable {
    private final Socket client;
    private final HttpRequestParser parser;

    public ConnectionWorker(Socket client, HttpRequestParser parser) {
        this.client = client;
        this.parser = parser;
    }

    @Override
    public void run() {
        try (client) {
            HttpRequest request = parser.parse(client.getInputStream());
            HttpResponse response = MiniServletContainer.getInstance().dispatch(request); // see §45
            client.getOutputStream().write(response.toBytes());
        } catch (IOException e) {
            System.err.println("Request failed: " + e.getMessage());
        }
    }
}
```

---

# 21. Graceful Shutdown

A production-grade `stop()` should not just call `executor.shutdown()` and exit — in-flight requests deserve a chance to finish, and hung ones need a deadline:

```java
public void stop(Duration gracePeriod) throws IOException, InterruptedException {
    running = false;
    serverSocket.close();               // 1. stop accepting new connections
    pool.getExecutor().shutdown();      // 2. reject new submissions, let queued/running tasks finish
    boolean finished = pool.getExecutor()
        .awaitTermination(gracePeriod.toMillis(), TimeUnit.MILLISECONDS);
    if (!finished) {
        pool.getExecutor().shutdownNow(); // 3. interrupt anything still running past the deadline
    }
}
```

This three-step shutdown — *stop intake, drain, then force* — is the same pattern Kubernetes expects from a container that receives `SIGTERM`: stop accepting new work immediately, but give existing requests a bounded window to complete before the process is killed.

---

# 22. Why We Need Per-Request Context

Once requests run on pooled worker threads, a new problem appears: code deep inside a controller or repository (a logging statement, a security check, a `@Transactional`-style helper) often needs request-scoped data — the current user, a trace/correlation ID, the raw `HttpRequest` — without every method signature threading it through as an extra parameter.

```java
// Without request context — every layer must pass traceId explicitly, forever
public User findUser(long id, String traceId) {
    log.info("[{}] looking up user {}", traceId, id);
    return repository.findById(id, traceId);
}
```

Since exactly one thread handles exactly one request at a time in the thread-per-request model (§14–§16), **the thread itself** is a natural place to stash that data — as long as two different requests running on the *same reused pooled thread*, one after another, never see each other's leftovers.

---

# 23. Introducing ThreadLocal

`ThreadLocal<T>` gives each thread its own independent copy of a variable — reads and writes from thread A never affect thread B's copy, even though they share the same `ThreadLocal` instance (the same static field).

```java
ThreadLocal<String> currentUser = new ThreadLocal<>();

// Thread A:
currentUser.set("alice");
System.out.println(currentUser.get()); // "alice"

// Thread B (concurrently):
currentUser.set("bob");
System.out.println(currentUser.get()); // "bob" — completely independent of thread A's value
```

Internally, every `Thread` object carries a private `ThreadLocalMap` keyed by the `ThreadLocal` instance itself; `get()`/`set()` on a `ThreadLocal` are really "look up my map entry inside *this* thread's map." No locking is needed, because no two threads ever touch the same map.

---

# 24. Phase 8 — RequestContextHolder

```java
// concurrent/RequestContextHolder.java
public final class RequestContextHolder {
    private static final ThreadLocal<RequestContext> CONTEXT = new ThreadLocal<>();

    private RequestContextHolder() { }

    public static void set(RequestContext context) {
        CONTEXT.set(context);
    }

    public static RequestContext get() {
        RequestContext context = CONTEXT.get();
        if (context == null) {
            throw new IllegalStateException("No request context bound to this thread");
        }
        return context;
    }

    public static void clear() {
        CONTEXT.remove(); // see §25 for why this call is not optional
    }
}

public class RequestContext {
    private final String traceId;
    private final HttpRequest request;
    private String currentUser; // set later, e.g. by an auth filter

    public RequestContext(String traceId, HttpRequest request) {
        this.traceId = traceId;
        this.request = request;
    }
    public String getTraceId() { return traceId; }
    public HttpRequest getRequest() { return request; }
    public void setCurrentUser(String user) { this.currentUser = user; }
    public String getCurrentUser() { return currentUser; }
}
```

`ConnectionWorker.run()` (§20) now binds and unbinds the context around the actual work:

```java
@Override
public void run() {
    try (client) {
        HttpRequest request = parser.parse(client.getInputStream());
        RequestContextHolder.set(new RequestContext(UUID.randomUUID().toString(), request));
        HttpResponse response = MiniServletContainer.getInstance().dispatch(request);
        client.getOutputStream().write(response.toBytes());
    } catch (IOException e) {
        System.err.println("Request failed: " + e.getMessage());
    } finally {
        RequestContextHolder.clear(); // ALWAYS runs, even on exception — see §25–§26
    }
}
```

Now any code running on this thread during this request — no matter how deeply nested — can call `RequestContextHolder.get().getTraceId()` without a single extra method parameter. This is exactly how Spring's real `RequestContextHolder` (`org.springframework.web.context.request.RequestContextHolder`) lets `@Autowired HttpServletRequest` work inside a singleton-scoped bean.

---

# 25. ThreadLocal and Thread Pools: The Leak Trap

A `ThreadLocal` value's lifetime is tied to the **thread**, not the **request**. In the thread-per-connection model of §14 that was harmless (the thread died with the connection). Once we introduced a *pool* in §16–§20, worker threads are **reused across many requests** — and that changes everything:

```text
Request 1 runs on worker-thread-7 -> sets RequestContext(traceId="abc") -> forgets to clear()
Worker-thread-7 returns to the pool, waits idle
Request 2 is submitted, happens to run on worker-thread-7 -> calls RequestContextHolder.get()
  -> gets Request 1's stale traceId "abc"!  (a correctness bug: cross-request data leakage)
```

Worse, if the `RequestContext` (or anything it references — a large cached object, a security principal) is never cleared, it stays reachable for as long as the pooled thread lives, i.e. potentially the lifetime of the whole server process — a genuine **memory leak**, because the GC can never collect an object a live thread still references through its `ThreadLocalMap`.

This exact failure mode is a classic in application servers that reuse thread pools (Tomcat, Spring, Hibernate's `ThreadLocal` session) — it's precisely why frameworks are religious about clearing `ThreadLocal`s at the end of every request.

---

# 26. Cleaning Up: try/finally and remove()

The fix is the one already shown in §24: **every** code path that sets a `ThreadLocal` for a request must clear it in a `finally` block, so it runs whether the request succeeded, threw an exception, or was cut off by a client disconnect.

```java
RequestContextHolder.set(context);
try {
    handleRequest(context);
} finally {
    RequestContextHolder.clear(); // guaranteed to run
}
```

A few supporting rules worth internalizing:

- **Never** rely on the *next* request to overwrite a stale value with `set()` — an exception thrown before that `set()` call (e.g. in an auth filter) leaves the stale value in place for a request that never expected it.
- Prefer `threadLocal.remove()` over `threadLocal.set(null)` — `remove()` deletes the map entry entirely; `set(null)` still leaves an entry (with a `null` value) referencing the `ThreadLocal` key, which is a smaller but real difference when many distinct `ThreadLocal`s are in play.
- If you build a **filter chain** (§46) around the dispatch call, put the `clear()` in the outermost filter's `finally`, not inside the innermost handler — that way it runs exactly once per request no matter how many filters short-circuit or throw.
- Tools like static analyzers and even `ThreadMXBean`-based thread dumps can help catch a `ThreadLocal` that was set but never cleared, by inspecting a suspiciously large `ThreadLocalMap` on a long-lived pool thread.

---

# 27. HTTP Keep-Alive Semantics

Every server so far has closed the socket after one response (`Connection: close`). Opening a fresh TCP connection per request — the three-way handshake, TLS negotiation if applicable, slow-start — is expensive relative to a tiny request/response pair. HTTP/1.1 makes **persistent connections the default**: unless a `Connection: close` header says otherwise, both sides keep the socket open and send another request/response pair over it.

```text
Connection stays open:
  Client -> GET /a  -> Server responds -> (socket stays open)
  Client -> GET /b  -> Server responds -> (socket stays open)
  Client -> GET /c  -> Server responds -> ... until either side sends Connection: close, or an idle timeout fires
```

This changes our threading model's job: a worker thread handling a keep-alive connection isn't done after one response — it must loop, parsing another request from the *same socket*, until the client disconnects or times out.

---

# 28. Phase 9 — Implementing Keep-Alive

```java
@Override
public void run() {
    try (client) {
        client.setSoTimeout(idleTimeoutMillis); // see §29
        while (!client.isClosed()) {
            HttpRequest request;
            try {
                request = parser.parse(client.getInputStream());
            } catch (SocketTimeoutException idle) {
                break; // no request arrived within the idle window — close politely
            } catch (EOFException clientClosed) {
                break; // client closed its write side — nothing more to read
            }

            RequestContextHolder.set(new RequestContext(UUID.randomUUID().toString(), request));
            boolean keepAlive;
            try {
                HttpResponse response = MiniServletContainer.getInstance().dispatch(request);
                keepAlive = shouldKeepAlive(request, response);
                response.setHeader("Connection", keepAlive ? "keep-alive" : "close");
                client.getOutputStream().write(response.toBytes());
                client.getOutputStream().flush();
            } finally {
                RequestContextHolder.clear();
            }
            if (!keepAlive) break;
        }
    } catch (IOException e) {
        System.err.println("Connection error: " + e.getMessage());
    }
}

private boolean shouldKeepAlive(HttpRequest request, HttpResponse response) {
    String clientHeader = request.getHeader("connection");
    if ("close".equalsIgnoreCase(clientHeader)) return false;
    if ("HTTP/1.0".equals(request.getHttpVersion())) {
        // HTTP/1.0 defaults to close unless the client explicitly opts in
        return "keep-alive".equalsIgnoreCase(clientHeader);
    }
    return true; // HTTP/1.1 defaults to keep-alive
}
```

Notice the worker thread is now occupied for the **entire lifetime of the connection**, not just one request — which is exactly why keep-alive and thread-pool sizing (§18) are coupled: a busy keep-alive client holds a pool thread hostage even while idle between requests, unless you either give it a short idle timeout (§29) or move to a model where one thread can serve many idle connections (§35–§38).

---

# 29. Idle Connection Timeouts and Reaping

`Socket.setSoTimeout(millis)` makes a blocking read throw `SocketTimeoutException` after that many milliseconds of no data — without it, a client that opens a connection and never sends anything (deliberately or by a network glitch) parks a worker thread **forever**, one of the simplest denial-of-service vectors against a thread-per-connection server.

```java
client.setSoTimeout(30_000); // no data for 30s on an idle keep-alive connection -> give up gracefully
```

Two timeouts matter in practice, and real servers set them differently:

| Timeout | Applies to | Typical value |
|---|---|---|
| Idle timeout (between requests on a keep-alive connection) | Time waiting for the *next* request line | Tens of seconds (Tomcat's `connectionTimeout` / `keepAliveTimeout` defaults are in this range). |
| Request timeout (mid-request) | Time to fully receive one request or send one response | Should generally be looser than the idle timeout, or handled separately, so a slow-but-legitimate upload isn't killed by the same clock as an idle keep-alive gap. |

---

# 30. Two Meanings of "Connection Pool"

The phrase "connection pool" means two genuinely different things depending on which side of the request you're standing on — mixing them up is a common interview stumble:

| | Server-side "connection pool" | Client-side connection pool (e.g. JDBC) |
|---|---|---|
| What's being reused | An **accepted `Socket`** from a browser/client, kept alive across several HTTP requests (§27–§29). | An **outbound `Connection`** your server opens *to* something else (a database, another service), reused across several of *your own* outgoing calls. |
| Who initiates reuse | The remote client, via `Connection: keep-alive`. | Your own application code, via a pool object it calls `borrow()`/`return()` on. |
| What it saves | Repeated TCP handshakes with browsers. | Repeated TCP handshakes **and** expensive DB session/auth setup with the database. |
| Where it lives in MiniTomcat | `ConnectionWorker` + keep-alive loop (§28). | `MiniConnectionPool` (§32–§33), used by MiniSpring `@Repository` beans. |

Both are the same underlying idea — *"opening this kind of connection is expensive, so don't throw it away after one use"* — applied at two different layers of the same request's journey.

---

# 31. Phase 10 — A Generic Object Pool

Before building the JDBC-specific pool, it's worth building the reusable shape once, since MiniTomcat needs pooling in more than one place (buffers, parser scratch objects, DB connections):

```java
// pool/ObjectPool.java
public interface ObjectPool<T> {
    T borrow() throws InterruptedException;
    void release(T item);
    void close();
}

// pool/BlockingObjectPool.java — a bounded pool backed by a BlockingQueue
public class BlockingObjectPool<T> implements ObjectPool<T> {
    private final BlockingQueue<T> available;
    private final Supplier<T> factory;

    public BlockingObjectPool(int size, Supplier<T> factory) {
        this.factory = factory;
        this.available = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            available.offer(factory.get()); // pre-warm: create every instance up front
        }
    }

    @Override
    public T borrow() throws InterruptedException {
        return available.take(); // blocks here if the pool is fully checked out
    }

    @Override
    public void release(T item) {
        available.offer(item); // hand it back for the next borrower
    }

    @Override
    public void close() {
        available.clear();
    }
}
```

A `BufferPool` (pooled reusable `byte[]` buffers for reading request bodies) is then just `new BlockingObjectPool<>(64, () -> new byte[8192])` — avoiding a fresh 8 KB allocation (and the GC churn that goes with it) on every single request. The exact same shape, with a different factory, becomes the JDBC connection pool next.

---

# 32. Connection Pooling for the Database (JDBC)

Opening a JDBC `Connection` is expensive: TCP handshake to the database, authentication, session setup on the DB server — often tens of milliseconds, dwarfing the actual query. A web server handling many short-lived requests **must not** open a fresh `Connection` per request; instead it keeps a small pool of already-open, already-authenticated connections and hands them out on demand.

```text
Without pooling: every request pays connection setup cost
  Request -> new Connection() [~20ms] -> query [~2ms] -> close()   <- 90% of the time is setup!

With pooling: setup cost is paid once, amortized across many requests
  Request -> pool.borrow() [~0ms, already open] -> query [~2ms] -> pool.release()
```

---

# 33. Phase 11 — A Minimal JDBC Connection Pool

```java
// pool/jdbc/MiniConnectionPool.java
public class MiniConnectionPool implements ObjectPool<Connection> {
    private final BlockingQueue<Connection> available;
    private final Set<Connection> allConnections = ConcurrentHashMap.newKeySet();
    private final String url, user, password;

    public MiniConnectionPool(String url, String user, String password, int size) throws SQLException {
        this.url = url; this.user = user; this.password = password;
        this.available = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            Connection connection = createRawConnection();
            allConnections.add(connection);
            available.offer(connection);
        }
    }

    private Connection createRawConnection() throws SQLException {
        return DriverManager.getConnection(url, user, password);
    }

    @Override
    public Connection borrow() throws InterruptedException {
        Connection connection = available.poll(); // non-blocking first, then wait
        if (connection == null) {
            connection = available.take(); // block until one is released — bounded, deliberate backpressure
        }
        return wrapAsPooled(connection);
    }

    /** Wraps close() so callers can use try-with-resources naturally without truly closing the socket. */
    private Connection wrapAsPooled(Connection real) {
        return (Connection) Proxy.newProxyInstance(
            getClass().getClassLoader(),
            new Class<?>[]{Connection.class},
            (proxy, method, args) -> {
                if ("close".equals(method.getName())) {
                    release(real); // return to the pool instead of actually closing the socket
                    return null;
                }
                return method.invoke(real, args);
            });
    }

    @Override
    public void release(Connection connection) {
        available.offer(connection);
    }

    @Override
    public void close() {
        allConnections.forEach(c -> {
            try { c.close(); } catch (SQLException ignored) { }
        });
    }
}
```

**Why the `Proxy`-based `close()` trick?** It lets calling code keep writing idiomatic `try (Connection c = pool.borrow()) { ... }` — the try-with-resources block calls `close()` exactly as it would on a real connection, but that call is intercepted to mean "return to the pool" instead of "tear down the socket." This is the same illusion real pools like HikariCP maintain: application code never has to know it's holding a pooled wrapper.

---

# 34. Wiring the JDBC Pool Into a MiniSpring Repository Bean

Because MiniSpring's `DefaultBeanFactory` (see the companion guide, [§17](MiniSpring-Step-by-Step-Guide.md)) resolves constructor dependencies by type, the pool slots in as an ordinary `@Component` — no special-casing needed anywhere else in the container:

```java
@Component
public class DataSourceConfig {
    @Bean // if you added §88's @Bean support; otherwise construct it directly in a @Component's constructor
    public MiniConnectionPool connectionPool() throws SQLException {
        return new MiniConnectionPool(
            "jdbc:postgresql://localhost:5432/pureeats", "app", "secret", /* size */ 10);
    }
}

@Repository
public class JdbcUserRepository implements UserRepository {
    private final MiniConnectionPool pool;

    @Autowired
    public JdbcUserRepository(MiniConnectionPool pool) {
        this.pool = pool;
    }

    @Override
    public User findById(long id) {
        try (Connection connection = pool.borrow();
             PreparedStatement ps = connection.prepareStatement("SELECT * FROM users WHERE id = ?")) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? mapRow(rs) : null;
            }
        } catch (SQLException | InterruptedException e) {
            throw new RuntimeException("Query failed", e);
        }
    }
    // mapRow(...) omitted for brevity
}
```

Two pools are now cooperating in the same request: MiniTomcat's server-side socket pool keeps the *browser* connection alive across requests (§27–§29), while `MiniConnectionPool` keeps a small set of *database* connections alive across all requests, for the whole lifetime of the process.

---

# 35. The C10K Problem

Thread-per-connection plus keep-alive (§28) has a specific weakness: a keep-alive connection that is **idle between requests** still occupies a full pool thread, blocked inside `client.getInputStream().read()`, doing nothing. Ten thousand slow-polling or idle-but-open clients means ten thousand blocked threads — the exact stack-memory and context-switch costs from §15, just triggered by *idleness* instead of load.

This is the classic **C10K problem** (serving ten thousand concurrent connections on one machine): with a blocking-I/O, thread-per-connection model, you eventually run out of threads long before you run out of CPU or bandwidth, because most of those threads are simply parked waiting for the next byte.

---

# 36. Introducing NIO: Selector, Channel, Buffer

Java's NIO package answers this with **non-blocking** I/O and a **readiness-based** event loop:

| Blocking I/O (`java.io`) | Non-blocking I/O (`java.nio`) |
|---|---|
| `socket.getInputStream().read()` blocks the calling thread until data arrives. | `channel.read(buffer)` returns immediately — with 0 bytes if nothing is available yet. |
| Needs one thread per connection to stay responsive. | One thread can ask a `Selector` *"which of these 10,000 channels actually have data ready right now?"* and only touch those. |

```java
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();                                   // blocks only until SOMETHING is ready
    Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
    while (keys.hasNext()) {
        SelectionKey key = keys.next();
        keys.remove();
        if (key.isAcceptable()) acceptNewConnection(key, selector);
        else if (key.isReadable()) readFromConnection(key);
    }
}
```

One thread, driven by `selector.select()`, services **every** ready channel in a tight loop — the "reactor pattern" that Netty, Node.js, and NGINX all build on.

---

# 37. Phase 12 (Optional) — A Selector-Based Reactor Loop

A sketch of the read side, enough to see the shape without building a full production reactor:

```java
private void acceptNewConnection(SelectionKey key, Selector selector) throws IOException {
    ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
    SocketChannel client = serverChannel.accept();
    client.configureBlocking(false);
    client.register(selector, SelectionKey.OP_READ, new RequestBuffer()); // per-connection parse state
}

private void readFromConnection(SelectionKey key) throws IOException {
    SocketChannel client = (SocketChannel) key.channel();
    RequestBuffer state = (RequestBuffer) key.attachment();

    ByteBuffer buffer = ByteBuffer.allocate(4096);
    int bytesRead = client.read(buffer);
    if (bytesRead == -1) {
        client.close();
        return;
    }
    buffer.flip();
    state.append(buffer); // accumulate bytes across possibly many partial reads

    if (state.isCompleteRequest()) {
        HttpRequest request = state.toHttpRequest();
        pool.submit(() -> {                          // hand CPU/blocking work back to a worker pool!
            HttpResponse response = MiniServletContainer.getInstance().dispatch(request);
            writeResponseNonBlocking(client, response);
        });
        state.reset();
    }
}
```

The crucial design point: **the reactor thread itself must never block.** A partially-received request is *state* (`RequestBuffer`) attached to the `SelectionKey`, not a paused call stack — because with non-blocking I/O there is no thread sitting inside `read()` waiting for the rest of the bytes to arrive on a slow connection.

---

# 38. Thread-Per-Connection vs Reactor — Tradeoffs

| | Thread-per-connection (§14–§29) | Reactor / Selector (§36–§37) |
|---|---|---|
| Mental model | One thread = one connection's whole lifecycle; code reads top-to-bottom. | One thread drives many connections; state must be explicit (no call stack to "come back to"). |
| Idle-connection cost | One parked thread per idle keep-alive client. | Near-zero — an idle channel is just an unselected entry in the `Selector`. |
| Code complexity | Simple, linear, easy to debug with a thread dump. | Harder: partial reads, explicit per-connection state machines, careful handoff of CPU work to a separate pool. |
| CPU-bound work | Runs inline on the connection's own thread — fine, since that thread isn't needed elsewhere. | Must be handed off to a worker pool (as in §37) — running it on the reactor thread would stall every other connection. |
| Best fit | Moderate connection counts, request/response-shaped traffic (typical REST APIs). | Very high connection counts, especially many long-lived/idle connections (chat, long-polling, proxies). |

Real servers often **combine** both: a small reactor tier accepts connections and does non-blocking I/O, while actual request handling still runs on a bounded worker pool — exactly the handoff shown at the `pool.submit(...)` line in §37.

---

# 39. Virtual Threads as a Modern Alternative

Java 21's **virtual threads** (`Thread.ofVirtual()`, JEP 444) offer a third option: keep the simple thread-per-connection *programming model* of §14–§29, but let the JVM itself multiplex many virtual threads onto a small number of OS ("carrier") threads, parking a virtual thread cheaply (not an OS-level block) whenever it hits blocking I/O.

```text
Platform thread-per-connection:  10,000 connections -> 10,000 OS threads -> exhausted stacks/schedulers (§15)
Virtual thread-per-connection:   10,000 connections -> 10,000 CHEAP virtual threads -> a handful of OS carrier threads
```

This directly attacks the C10K problem (§35) **without** requiring the non-blocking, state-machine style of §36–§38 — ordinary blocking `InputStream.read()` calls are fine, because blocking a virtual thread doesn't block its carrier OS thread.

---

# 40. Phase 12b — Swapping to Virtual Threads

The change to MiniTomcat is almost embarrassingly small — replace the `ThreadPoolExecutor` from §16 with a virtual-thread-per-task executor:

```java
public class ServerThreadPool {
    private final ExecutorService executor;

    public ServerThreadPool(Mode mode, int coreSize, int maxSize, int queueCapacity) {
        this.executor = switch (mode) {
            case PLATFORM_POOL -> new ThreadPoolExecutor(coreSize, maxSize, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(queueCapacity));
            case VIRTUAL_THREADS -> Executors.newVirtualThreadPerTaskExecutor();
        };
    }

    public void submit(Runnable task) { executor.execute(task); }
    public void shutdown() { executor.shutdown(); }
}
```

**What you keep:** the entire blocking-I/O `ConnectionWorker` from §20/§28 works completely unchanged — no `Selector`, no partial-read state machine.

**What you give up / must watch for:**

- Virtual threads still need a **real, bounded resource somewhere else** in the chain — if every virtual thread ends up contending for the *same* fixed-size JDBC pool (§33) or the *same* small set of CPU cores for actual computation, you've just moved the bottleneck, not removed it.
- **`synchronized` blocks pin the virtual thread to its carrier** in older JDKs (this was fixed in JDK 24 but is a real caveat on JDK 21/22/23) — a long-held `synchronized` lock around blocking I/O can quietly reintroduce platform-thread-level contention. Prefer `java.util.concurrent.locks.ReentrantLock` in code virtual threads will run.
- `ThreadLocal` (§22–§26) still works exactly the same way — each virtual thread gets its own `ThreadLocalMap` — but with potentially *millions* of short-lived virtual threads created over a server's lifetime, a `ThreadLocal` leak (§25) becomes a leak per-request instead of a leak per pooled platform thread, which can make it both less severe per-instance and easier to miss until it's pervasive.

---

# 41. What Tomcat Actually Is

It's worth being precise about the term: **Tomcat is a Servlet container**, not a web framework. It owns everything this guide has built so far — the socket accept loop, the thread pool, HTTP parsing, keep-alive — and its *only* contract with application code is the **Servlet API** (`jakarta.servlet.Servlet`): "give me an object with a `service(request, response)` method, and I will call it whenever a matching request arrives." Spring MVC's `DispatcherServlet` is, from Tomcat's point of view, just one more `Servlet` — a single one that happens to internally do its own routing to `@Controller` methods.

```text
Tomcat (owns sockets, threads, HTTP parsing)
   |
   v
Servlet API boundary  <-- the ONLY thing app code implements
   |
   v
DispatcherServlet (Spring's one big Servlet)
   |
   v
HandlerMapping -> @Controller method
```

Everything MiniTomcat has built up to this point (§1–§40) is the top box. This section starts building the boundary layer that makes MiniSpring's `@Controller` beans (or, in principle, real Spring's) pluggable underneath it.

---

# 42. The Servlet API in a Nutshell

The real `jakarta.servlet.Servlet` interface is small on purpose:

```java
public interface Servlet {
    void init(ServletConfig config) throws ServletException;
    void service(ServletRequest request, ServletResponse response) throws ServletException, IOException;
    void destroy();
    ServletConfig getServletConfig();
    String getServletInfo();
}
```

| Method | Called when | Purpose |
|---|---|---|
| `init(config)` | Once, when the container first loads the servlet | One-time setup (read init params, acquire resources). |
| `service(req, res)` | Once per matching HTTP request | The actual per-request work — this is what `HttpServlet.doGet/doPost` delegate to. |
| `destroy()` | Once, when the container shuts the servlet down | Release resources acquired in `init`. |

`ServletConfig` carries per-servlet init parameters and a reference to the shared `ServletContext` — the container-wide object every servlet in the same deployment can reach (attributes, resource lookup, and, in a Spring app, the root `ApplicationContext` itself, stashed as a `ServletContext` attribute).

---

# 43. Phase 13 — Designing MiniServlet

```java
// servlet/MiniServlet.java
public interface MiniServlet {
    void init(MiniServletConfig config) throws Exception;
    void service(MiniHttpServletRequest request, MiniHttpServletResponse response) throws Exception;
    void destroy();
}

// servlet/MiniServletConfig.java
public class MiniServletConfig {
    private final String servletName;
    private final MiniServletContext servletContext;
    private final Map<String, String> initParams;

    public MiniServletConfig(String servletName, MiniServletContext servletContext, Map<String, String> initParams) {
        this.servletName = servletName;
        this.servletContext = servletContext;
        this.initParams = initParams;
    }
    public String getServletName() { return servletName; }
    public MiniServletContext getServletContext() { return servletContext; }
    public String getInitParameter(String name) { return initParams.get(name); }
}

// servlet/MiniServletContext.java — one instance shared by every servlet in the process
public class MiniServletContext {
    private final Map<String, Object> attributes = new ConcurrentHashMap<>();

    public void setAttribute(String name, Object value) { attributes.put(name, value); }
    public Object getAttribute(String name) { return attributes.get(name); }
}
```

Note the direct parallel with real Tomcat: `MiniServletContext.setAttribute(...)` is exactly the mechanism used in §51 to publish the MiniSpring `ApplicationContext` somewhere every servlet (specifically, `MiniDispatcherServlet`) can retrieve it — mirroring how real Spring's `ContextLoaderListener` stores the root `WebApplicationContext` as a `ServletContext` attribute for `DispatcherServlet` to pick up.

---

# 44. MiniHttpServletRequest / MiniHttpServletResponse

These wrap the wire-level `HttpRequest`/`HttpResponse` from §10/§ (the ones `HttpRequestParser` produces) behind a friendlier, Servlet-shaped API:

```java
// servlet/MiniHttpServletRequest.java
public class MiniHttpServletRequest {
    private final HttpRequest raw;
    private final Map<String, String> pathVariables = new HashMap<>(); // filled in by routing, §45/§50

    public MiniHttpServletRequest(HttpRequest raw) { this.raw = raw; }

    public String getMethod() { return raw.getMethod(); }
    public String getPathInfo() { return raw.getPath(); }
    public String getHeader(String name) { return raw.getHeader(name); }
    public String getParameter(String name) { return parseQueryParam(raw.getQueryString(), name); }
    public String getPathVariable(String name) { return pathVariables.get(name); }
    void setPathVariable(String name, String value) { pathVariables.put(name, value); }
    public byte[] getBody() { return raw.getBody(); }
    // parseQueryParam(...) omitted for brevity — splits on '&' and '='
}

// servlet/MiniHttpServletResponse.java
public class MiniHttpServletResponse {
    private int status = 200;
    private final Map<String, String> headers = new LinkedHashMap<>();
    private byte[] body = new byte[0];

    public void setStatus(int status) { this.status = status; }
    public void setHeader(String name, String value) { headers.put(name, value); }
    public void write(byte[] bytes) { this.body = bytes; }
    public void writeJson(String json) {
        setHeader("Content-Type", "application/json");
        write(json.getBytes(StandardCharsets.UTF_8));
    }

    HttpResponse toWireResponse() {
        HttpResponse response = new HttpResponse(status, body);
        headers.forEach(response::setHeader);
        response.setHeader("Content-Length", String.valueOf(body.length));
        return response;
    }
}
```

---

# 45. Phase 14 — MiniServletContainer (Path Routing)

```java
// servlet/MiniServletContainer.java
public class MiniServletContainer {
    private static final MiniServletContainer INSTANCE = new MiniServletContainer();
    public static MiniServletContainer getInstance() { return INSTANCE; }

    private final Map<String, MiniServlet> pathToServlet = new LinkedHashMap<>(); // insertion order = priority
    private final MiniServletContext servletContext = new MiniServletContext();
    private final List<MiniFilter> filters = new ArrayList<>();

    public void register(String pathPattern, MiniServlet servlet, Map<String, String> initParams) {
        try {
            servlet.init(new MiniServletConfig(servlet.getClass().getSimpleName(), servletContext, initParams));
        } catch (Exception e) {
            throw new RuntimeException("Servlet init failed for " + pathPattern, e);
        }
        pathToServlet.put(pathPattern, servlet);
    }

    public void addFilter(MiniFilter filter) { filters.add(filter); }

    public MiniServletContext getServletContext() { return servletContext; }

    public HttpResponse dispatch(HttpRequest rawRequest) {
        MiniHttpServletRequest request = new MiniHttpServletRequest(rawRequest);
        MiniHttpServletResponse response = new MiniHttpServletResponse();

        MiniServlet servlet = resolve(rawRequest.getPath());
        if (servlet == null) {
            response.setStatus(404);
            response.write("Not Found".getBytes(StandardCharsets.UTF_8));
            return response.toWireResponse();
        }

        try {
            runFilterChainThenServlet(0, request, response, servlet);
        } catch (Exception e) {
            response.setStatus(500);
            response.write(("Internal Server Error: " + e.getMessage()).getBytes(StandardCharsets.UTF_8));
        }
        return response.toWireResponse();
    }

    private void runFilterChainThenServlet(int index, MiniHttpServletRequest req,
                                            MiniHttpServletResponse res, MiniServlet servlet) throws Exception {
        if (index < filters.size()) {
            filters.get(index).doFilter(req, res, () -> runFilterChainThenServlet(index + 1, req, res, servlet));
        } else {
            servlet.service(req, res);
        }
    }

    /** Longest-prefix match — the same basic strategy Tomcat uses for path-mapped servlets. */
    private MiniServlet resolve(String path) {
        return pathToServlet.entrySet().stream()
            .filter(e -> path.startsWith(e.getKey()))
            .max(Comparator.comparingInt(e -> e.getKey().length()))
            .map(Map.Entry::getValue)
            .orElse(null);
    }
}
```

This is the container's routing table — mechanically the same idea as `web.xml`'s `<servlet-mapping>`, just built in code. In §50, exactly **one** entry gets registered here: `/` mapped to `MiniDispatcherServlet`, which then does its *own*, finer-grained routing via MiniSpring's `HandlerMapping`.

---

# 46. Filters: Chain of Responsibility

```java
// servlet/MiniFilter.java
@FunctionalInterface
public interface MiniFilterChain {
    void doFilter() throws Exception;
}

public interface MiniFilter {
    void doFilter(MiniHttpServletRequest request, MiniHttpServletResponse response, MiniFilterChain chain) throws Exception;
}
```

```java
// A logging filter — runs before AND after the rest of the chain
public class LoggingFilter implements MiniFilter {
    @Override
    public void doFilter(MiniHttpServletRequest req, MiniHttpServletResponse res, MiniFilterChain chain) throws Exception {
        long start = System.nanoTime();
        try {
            chain.doFilter(); // runs the next filter, or finally the servlet
        } finally {
            long tookMs = (System.nanoTime() - start) / 1_000_000;
            System.out.printf("%s %s -> %d (%dms)%n", req.getMethod(), req.getPathInfo(), res.getStatusForLogging(), tookMs);
        }
    }
}
```

This `doFilter(request, response, chain)` shape *is* the Chain of Responsibility pattern: each filter decides whether to call `chain.doFilter()` (continue) or not (short-circuit — e.g. an auth filter rejecting an unauthenticated request without ever reaching the servlet). It is also exactly where §26's `RequestContextHolder.clear()` belongs — in the outermost filter's `finally`, so it always runs regardless of which inner filter or servlet threw.

---

# 47. Listeners: Lifecycle Hooks

Real Servlet containers also support `ServletContextListener` (container startup/shutdown) and `HttpSessionListener` (session created/destroyed) — hooks that run independent of any single request. A minimal equivalent:

```java
public interface MiniServletContextListener {
    void contextInitialized(MiniServletContext context);
    void contextDestroyed(MiniServletContext context);
}
```

MiniTomcat calls every registered listener's `contextInitialized` once, right after building the `MiniServletContext` and before accepting the first connection — this is precisely the hook real Spring uses (`ContextLoaderListener`) to build the root `ApplicationContext` *before* any servlet's `init()` runs, guaranteeing beans exist by the time a request arrives. §50 reuses this same ordering for MiniSpring.

---

# 48. Recap: What MiniSpring Already Gives Us

The companion guide ([MiniSpring-Step-by-Step-Guide.md](MiniSpring-Step-by-Step-Guide.md)) already built everything *above* the transport layer:

| Already built in MiniSpring | Lives in |
|---|---|
| `MiniApplicationContext` — scans `@Component`/`@Service`/`@Repository`/`@Controller`, wires `@Autowired` dependencies | MiniSpring §15, §24 |
| `HandlerMapping` — a table of `(HTTP method, URL pattern) -> (controller bean, method)` | MiniSpring §27–§28 |
| `@GetMapping`/`@PostMapping`/`@PathVariable`/`@RequestParam` handling, argument resolution, JSON serialization | MiniSpring §12–§13, §63–§65 |
| A `FrontController` that previously ran on the JDK's `com.sun.net.httpserver.HttpServer` | MiniSpring §29–§31 |

The **only** piece MiniSpring's guide left as a "toy" was the transport: the JDK's built-in `HttpServer` has no real thread-pool tuning story, no keep-alive control worth mentioning, and no path to plugging into anything else. Everything in *this* guide up through §47 is a drop-in replacement for that one box.

---

# 49. Where the Web Server Fits in the MiniSpring Architecture

```text
                    MiniTomcat (this guide, §1–§47)
                    accept loop -> thread pool -> HTTP parse -> MiniServletContainer
                                                                        |
                                                     ONE servlet registered at "/"
                                                                        v
                                                        MiniDispatcherServlet (§50)
                                                                        |
                                        looks up ApplicationContext from MiniServletContext (§43, §47)
                                                                        |
                                                                        v
                                          MiniSpring's HandlerMapping.resolve(method, path)
                                                                        |
                                                                        v
                                                    Invoke the matched @Controller method
                                                    (constructor/field-injected beans, resolved
                                                     ONCE at ApplicationContext startup — MiniSpring §24)
```

Nothing in MiniSpring's `ApplicationContext`, `HandlerMapping`, or `@Controller` classes needs to change at all — they were already written against a plain `(HttpRequest) -> HttpResponse`-shaped abstraction. Only the *transport underneath* changes.

---

# 50. Phase 15 — MiniDispatcherServlet: Bridging the Two

```java
// bridge/MiniDispatcherServlet.java
public class MiniDispatcherServlet implements MiniServlet {
    public static final String APPLICATION_CONTEXT_ATTRIBUTE = "miniSpring.applicationContext";

    private ApplicationContext applicationContext; // MiniSpring's context interface
    private HandlerMapping handlerMapping;          // MiniSpring's route table

    @Override
    public void init(MiniServletConfig config) {
        // Retrieve the ApplicationContext a listener already built and published (§51) —
        // the servlet itself never calls new MiniApplicationContext(...) directly, mirroring
        // how real Spring's DispatcherServlet finds a context a ContextLoaderListener already built.
        this.applicationContext = (ApplicationContext)
            config.getServletContext().getAttribute(APPLICATION_CONTEXT_ATTRIBUTE);
        this.handlerMapping = new HandlerMapping();
        this.handlerMapping.registerControllers(applicationContext,
            ((MiniApplicationContext) applicationContext).getBeanFactory());
    }

    @Override
    public void service(MiniHttpServletRequest request, MiniHttpServletResponse response) throws Exception {
        HandlerMethod handler = handlerMapping.find(request.getMethod(), request.getPathInfo());
        if (handler == null) {
            response.setStatus(404);
            response.writeJson("{\"error\":\"No handler for " + request.getMethod() + " " + request.getPathInfo() + "\"}");
            return;
        }

        extractPathVariables(handler, request); // populate request.pathVariables from the matched pattern

        Object result = invokeHandler(handler, request);
        writeResult(response, result);
    }

    private Object invokeHandler(HandlerMethod handler, MiniHttpServletRequest request) throws Exception {
        Method method = handler.getMethod();
        Object[] args = resolveArguments(method, request); // @PathVariable / @RequestParam, MiniSpring §63
        return method.invoke(handler.getControllerBean(), args);
    }

    private void writeResult(MiniHttpServletResponse response, Object result) {
        if (result instanceof ResponseEntity<?> entity) {
            response.setStatus(entity.getStatus());
            response.writeJson(JsonSerializer.toJson(entity.getBody()));
        } else {
            response.setStatus(200);
            response.writeJson(JsonSerializer.toJson(result));
        }
    }

    @Override
    public void destroy() { /* no-op: the ApplicationContext outlives this servlet */ }

    // extractPathVariables(...) / resolveArguments(...) reuse the exact logic already built in
    // MiniSpring's own FrontController (companion guide §31) — only the request/response TYPES differ,
    // since they now come from MiniHttpServletRequest/Response instead of com.sun.net.httpserver's classes.
}
```

The entire point of this class is that it is **thin** — it does no business logic and no bean wiring itself. It only translates between two vocabularies: "MiniTomcat's Servlet API" on one side, "MiniSpring's `HandlerMapping`/`ApplicationContext`" on the other. That thinness is intentional and mirrors real Spring's `DispatcherServlet`, which is famously also "just" a translation and delegation layer.

---

# 51. Replacing the JDK HttpServer With MiniTomcat

Two small pieces of wiring finish the integration: a listener that builds the MiniSpring context once at startup, and a `main` that registers `MiniDispatcherServlet` at `/` before starting the accept loop.

```java
public class MiniSpringContextListener implements MiniServletContextListener {
    private final Class<?> appConfigClass;
    public MiniSpringContextListener(Class<?> appConfigClass) { this.appConfigClass = appConfigClass; }

    @Override
    public void contextInitialized(MiniServletContext servletContext) {
        ApplicationContext context = new MiniApplicationContext(appConfigClass); // MiniSpring §24–§25
        servletContext.setAttribute(MiniDispatcherServlet.APPLICATION_CONTEXT_ATTRIBUTE, context);
    }

    @Override
    public void contextDestroyed(MiniServletContext servletContext) { /* no-op */ }
}
```

```java
public class Application {
    public static void main(String[] args) throws IOException {
        MiniServletContainer container = MiniServletContainer.getInstance();

        new MiniSpringContextListener(AppConfig.class).contextInitialized(container.getServletContext());
        container.addFilter(new LoggingFilter());                                  // §46
        container.register("/", new MiniDispatcherServlet(), Map.of());            // §50

        MiniTomcat server = new MiniTomcat(8080, /* coreThreads */ 20, /* maxThreads */ 200, /* queue */ 1000);
        server.start(); // §20
        System.out.println("MiniTomcat + MiniSpring listening on :8080");
    }
}
```

This is the direct analogue of `MiniApplication.run(AppConfig.class, args)` from the companion guide's §33/§70 — except the HTTP transport underneath is now the server built in *this* guide, with real thread-pool tuning, keep-alive, and connection pooling, instead of the JDK's built-in one.

---

# 52. Full Sample: MiniTomcat + MiniSpring End-to-End

Putting every piece together with a concrete controller, exactly like the companion guide's demo app (§72), now running on MiniTomcat:

```java
@Repository
public class UserRepository {
    private final MiniConnectionPool pool; // §33
    @Autowired
    public UserRepository(MiniConnectionPool pool) { this.pool = pool; }

    public User findById(long id) { /* ... JDBC via pool.borrow(), §34 ... */ return new User(id, "Alice"); }
}

@Service
public class UserService {
    private final UserRepository repository;
    @Autowired
    public UserService(UserRepository repository) { this.repository = repository; }
    public User getUser(long id) { return repository.findById(id); }
}

@Controller
@RequestMapping("/users")
public class UserController {
    private final UserService userService;
    @Autowired
    public UserController(UserService userService) { this.userService = userService; }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable("id") long id) {
        User user = userService.getUser(id);
        return user == null ? ResponseEntity.status(404).build() : ResponseEntity.ok(user);
    }
}
```

```text
curl http://localhost:8080/users/42
```

```text
Request flow for the curl above:
  MiniTomcat accept loop (§20) -> ConnectionWorker on pool thread (§16)
    -> RequestContextHolder.set(...) (§24)
    -> MiniServletContainer.dispatch() (§45) -> LoggingFilter (§46) -> MiniDispatcherServlet.service() (§50)
       -> HandlerMapping.find("GET", "/users/42") -> UserController.getUser(42)
          -> UserService.getUser(42) -> UserRepository.findById(42) -> pool.borrow() (§33) -> SQL query
       <- ResponseEntity<User> <- JSON serialized <- MiniHttpServletResponse
    -> RequestContextHolder.clear() (§26, always runs)
  <- HTTP/1.1 200 OK { "id": 42, "name": "Alice" } written back over the (possibly kept-alive) socket
```

Every concept from §1–§51 appears somewhere on this one request's path — that end-to-end trace is the single best artifact to walk an interviewer through.

---

# 53. Startup Sequence Diagram

```text
Application.main()
  |
  1. MiniSpringContextListener.contextInitialized()
  |     -> new MiniApplicationContext(AppConfig.class)     [MiniSpring §24]
  |          -> ComponentScanner.scan(basePackage)          [MiniSpring §15]
  |          -> DefaultBeanFactory eagerly instantiates ALL beans, wiring @Autowired  [MiniSpring §24]
  |     -> servletContext.setAttribute("miniSpring.applicationContext", context)
  |
  2. container.register("/", new MiniDispatcherServlet(), ...)
  |     -> MiniDispatcherServlet.init(config)
  |          -> reads the ApplicationContext back out of servletContext
  |          -> builds HandlerMapping by scanning @Controller beans   [MiniSpring §27]
  |
  3. new MiniTomcat(port, coreThreads, maxThreads, queueCapacity)
  |     -> new ServerSocket(port).bind(...)
  |     -> new ServerThreadPool(...)                        [§16]
  |
  4. server.start() -> acceptLoop() runs on its own thread, forever, until stop()
```

Notice step 1 fully completes — **every** bean is constructed and wired — before step 4 ever calls `accept()`. This mirrors MiniSpring's own "eager instantiation at `refresh()`" design (companion guide §24): a broken `@Autowired` wiring fails the process at startup, never partway through serving live traffic.

---

# 54. How Spring Boot Starts an Embedded Container

When a real Spring Boot app calls `SpringApplication.run(...)`, one of the things that happens (for a web application) is that a `ServletWebServerApplicationContext` asks a **`ServletWebServerFactory`** bean to produce a **`WebServer`**, then calls `webServer.start()`. Spring Boot ships three implementations out of the box:

| Factory bean | Produces | Backing library |
|---|---|---|
| `TomcatServletWebServerFactory` | A `WebServer` wrapping an embedded Tomcat `Connector` | Apache Tomcat (`tomcat-embed-core`) |
| `JettyServletWebServerFactory` | A `WebServer` wrapping embedded Jetty | Eclipse Jetty |
| `UndertowServletWebServerFactory` | A `WebServer` wrapping embedded Undertow | Undertow |

Whichever one is on the classpath (and not excluded) gets auto-configured. Critically, Spring Boot itself doesn't care *how* the server accepts sockets or schedules threads — it only cares that the resulting object satisfies two small interfaces.

---

# 55. The ServletWebServerFactory / WebServer SPI

```java
// org.springframework.boot.web.server (real Spring Boot interfaces, shown for reference)
public interface WebServer {
    void start() throws WebServerException;
    void stop() throws WebServerException;
    int getPort();
}

public interface ServletWebServerFactory {
    WebServer getWebServer(ServletContextInitializer... initializers);
}
```

`ServletContextInitializer` is the hook Spring Boot uses to register its *own* servlets, filters, and listeners (including the one that ultimately registers `DispatcherServlet`) onto whatever `ServletContext` the container provides — the factory's job is to call each initializer with a real `ServletContext` before starting the server, so Spring's machinery gets wired into it exactly the way it would be wired into Tomcat.

---

# 56. What It Would Take to Plug MiniTomcat Into Spring Boot

Conceptually, three things:

1. **A real `jakarta.servlet.ServletContext` implementation**, not `MiniServletContext` (§43) — Spring's `ServletContextInitializer`s expect the *actual* servlet API type, with methods like `addServlet(...)`, `addFilter(...)`, `getAttribute(...)`, `getInitParameter(...)`, and dozens more.
2. **A real `HttpServletRequest`/`HttpServletResponse` implementation** on top of MiniTomcat's parsed `HttpRequest`/`HttpResponse` (§10, §44) — not `MiniHttpServletRequest`/`Response`, but the full `jakarta.servlet.http` interfaces, including things this guide never implemented: multipart file upload parsing, `HttpSession` support, async request handling (`startAsync()`), cookies, locale negotiation, and more.
3. **A `ServletWebServerFactory` + `WebServer`** that:
   - builds your real `ServletContext`,
   - calls every `ServletContextInitializer` Spring Boot hands it (this is how `DispatcherServlet` itself gets registered — Spring Boot never talks to your accept loop directly),
   - starts MiniTomcat's accept loop (§20) as `start()`,
   - calls the three-step graceful shutdown (§21) as `stop()`.

```java
// Sketch only — a REAL implementation needs the full jakarta.servlet surface, not MiniServlet's subset
public class MiniTomcatWebServerFactory implements ServletWebServerFactory {
    private int port = 8080;

    @Override
    public WebServer getWebServer(ServletContextInitializer... initializers) {
        RealServletContext servletContext = new RealServletContext(); // jakarta.servlet.ServletContext impl
        for (ServletContextInitializer initializer : initializers) {
            try {
                initializer.onStartup(servletContext); // this is where DispatcherServlet gets addServlet()'d
            } catch (ServletException e) {
                throw new WebServerException("Startup initializer failed", e);
            }
        }
        MiniTomcat server = new MiniTomcat(port, /* ... */);
        return new MiniTomcatWebServer(server, servletContext);
    }

    public void setPort(int port) { this.port = port; }
}

public class MiniTomcatWebServer implements WebServer {
    private final MiniTomcat server;
    public MiniTomcatWebServer(MiniTomcat server, RealServletContext ctx) { this.server = server; /* wire ctx into dispatch */ }
    @Override public void start() { server.start(); }
    @Override public void stop() { /* graceful shutdown, §21 */ }
    @Override public int getPort() { return server.getPort(); }
}
```

Registering it is the last, small piece — a `@Bean` that shadows Boot's auto-configured Tomcat factory:

```java
@Configuration
public class MiniTomcatConfig {
    @Bean
    public ServletWebServerFactory servletWebServerFactory() {
        MiniTomcatWebServerFactory factory = new MiniTomcatWebServerFactory();
        factory.setPort(8080);
        return factory;
    }
}
```

With that bean present (and the Tomcat starter excluded from the classpath so there's no ambiguity), `SpringApplication.run(MyApp.class, args)` would start **your** accept loop instead of embedded Tomcat's, and real `DispatcherServlet` would run on top of it.

---

# 57. Sample: MiniTomcatWebServerFactory (Wiring It Into a Spring Boot App)

```xml
<!-- pom.xml — depend on spring-boot-starter-web but EXCLUDE embedded Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args); // now boots MiniTomcatWebServerFactory's WebServer
    }
}
```

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() { return "Hello from real Spring Boot, running on MiniTomcat!"; }
}
```

If steps 1–3 in §56 are implemented faithfully enough, this ordinary-looking `@RestController` runs unmodified — proof that the Servlet API boundary really is the *entire* contract between a framework and its container.

---

# 58. Limitations vs Real Tomcat/Jetty/Undertow

Being honest about the gap is the difference between a great learning exercise and a false sense of "production-ready":

| Concern | Real Tomcat | MiniTomcat as built in this guide |
|---|---|---|
| HTTP/1.1 correctness (edge cases: malformed headers, pipelining, `100-continue`, header size limits) | Battle-tested against two decades of real-world clients | Handles the common cases shown here; many edge cases unhandled |
| HTTPS/TLS | Full `SSLContext`/ALPN support | Not built in this guide (see §71) |
| HTTP/2, WebSockets | Supported | Not built in this guide (see §71) |
| Security hardening | Slow-loris mitigation, request smuggling defenses, header injection protections, years of CVE fixes | None of this — this server should never face untrusted traffic directly |
| Full Servlet spec (async, multipart, sessions, `web.xml`, JSP) | Complete | Only the minimal subset in §41–§47 |
| Observability | JMX metrics, access logs, connector-level stats | None built here |
| Battle-testing | Millions of production deployments | A teaching project |

**The honest verdict:** building this teaches you *what* Tomcat does and *why* each piece exists — invaluable for interviews and for reading Tomcat/Spring source code with real comprehension. It should not replace Tomcat/Jetty/Undertow behind any traffic you don't fully control.

---

# 59. Verdict: Toy vs Production-Ready

A simple checklist for judging any "build your own X" project, applied here:

- ✅ Correct for the happy path and the failure modes explicitly discussed in this guide (timeouts, backpressure, malformed request lines).
- ✅ Demonstrates the real architectural boundary (Servlet API) that lets frameworks and containers evolve independently.
- ❌ Not hardened against adversarial input (a fuzzed request line, a client that sends gigabytes without a `Content-Length`, slow-loris-style partial headers).
- ❌ Missing huge swaths of the Servlet spec real frameworks quietly depend on (sessions, async, multipart).
- ❌ No security review, no fuzzing, no years of CVEs fixed by a community.

Use it exactly as intended: as a teaching vehicle and a portfolio/interview artifact, deployed at most behind a trusted internal boundary or in a sandbox — not as `webServerFactory` in anything customer-facing.

---

# 60. Optimization — Zero-Copy File Serving

If MiniTomcat ever serves static files, the naive path — `FileInputStream` into a `byte[]`, then `OutputStream.write(byte[])` — copies the file's bytes from kernel space to a JVM buffer, then back to kernel space for the socket write: two copies that never needed to touch application memory at all.

```java
// Naive: two extra copies through JVM heap memory
byte[] data = Files.readAllBytes(path);
socketOutputStream.write(data);
```

```java
// Zero-copy: the kernel transfers bytes directly from the file to the socket
try (FileChannel fileChannel = FileChannel.open(path, StandardOpenOption.READ)) {
    SocketChannel socketChannel = client.getChannel(); // requires NIO-based sockets, §36
    long position = 0, size = fileChannel.size();
    while (position < size) {
        position += fileChannel.transferTo(position, size - position, socketChannel);
    }
}
```

`FileChannel.transferTo` maps to the OS's `sendfile()` syscall where available — the data path becomes "disk → kernel buffer → socket buffer," skipping the JVM heap entirely. This is exactly how Tomcat's NIO connector serves static resources efficiently.

---

# 61. Optimization — Output Buffering

Writing a response in many small `OutputStream.write()` calls (one per header line, for instance) issues one system call per write, each with real overhead. Wrapping the socket's output stream in a `BufferedOutputStream` — and, more importantly, building the full response into one contiguous `byte[]` before a single `write()` call — collapses that to one syscall per response:

```java
public byte[] toBytes() {
    ByteArrayOutputStream buffer = new ByteArrayOutputStream(256 + body.length);
    buffer.writeBytes(statusLine().getBytes(StandardCharsets.US_ASCII));
    headers.forEach((name, value) ->
        buffer.writeBytes((name + ": " + value + "\r\n").getBytes(StandardCharsets.US_ASCII)));
    buffer.writeBytes("\r\n".getBytes(StandardCharsets.US_ASCII));
    buffer.writeBytes(body);
    return buffer.toByteArray(); // ONE array, written with ONE client.getOutputStream().write(...) call
}
```

The same principle applies on the read side: `BufferedInputStream` around the raw socket stream (already used in §10's parser) batches many small `read()` syscalls from the OS into fewer, larger ones.

---

# 62. Optimization — Reducing GC Pressure

A server handling thousands of requests per second allocates constantly — every one of those allocations is eventually garbage-collected, and GC pauses directly hurt tail latency. Three concrete levers, all already seeded elsewhere in this guide:

- **Pool reusable buffers** (§31's `BufferPool`) instead of `new byte[8192]` per request — the same allocation-avoidance idea as connection pooling, applied to memory instead of sockets/connections.
- **Avoid intermediate `String` churn** in the hot parsing path — `HttpRequestParser` (§10) already reads directly from the byte stream rather than, say, reading the entire request into one giant `String` and calling `.split(...)` on it repeatedly, which would allocate many short-lived array and string objects per request.
- **Reuse `HttpRequest`/response builder objects** across keep-alive iterations on the same connection where safe, instead of constructing a fresh object graph per request — with care not to reintroduce the `ThreadLocal`-style stale-data bug from §25 by forgetting to reset every field.

---

# 63. Optimization — Caching Routes and Reflection Metadata

MiniSpring's own companion guide already identifies this exact optimization for the DI container (§80–§81 there: caching reflection metadata instead of re-walking annotations on every `getBean()` call). The same principle applies at this layer:

- `MiniServletContainer.resolve(path)` (§45) does a linear scan with a `startsWith` check on every request — fine for a handful of servlets, but worth replacing with a **trie or sorted-prefix structure** once you have dozens of path patterns, so lookup is `O(log n)` or better instead of `O(n)`.
- `MiniDispatcherServlet`'s `HandlerMapping.find(method, path)` (§50) is exactly the same shape of problem as MiniSpring's `HandlerMapping` (companion guide §27–§28) — cache the compiled path-pattern-to-regex conversion once at startup (already implicit in that guide's design) rather than re-deriving it per request.
- Constructor/field injection metadata (which fields are `@Autowired`, which constructor to call) is already resolved once per bean at `ApplicationContext` startup in MiniSpring (companion guide §24), not per request — this web server layer should hold itself to the same standard: anything derivable purely from a `Class` object belongs in a cache built once, not recomputed on the request path.

---

# 64. Thread Pool Tuning Checklist

A concrete checklist to work through before calling any thread-pool configuration "done," tying together §16–§21 and §39–§40:

1. Classify the workload: CPU-bound, I/O-bound, or mixed (§18's core formula needs this answer first).
2. Set `corePoolSize`/`maximumPoolSize` from that classification, then load-test and adjust — formulas are a starting point, not a guarantee.
3. Bound the queue (§19) — an unbounded queue silently defeats `maximumPoolSize` and hides overload until an `OutOfMemoryError`.
4. Pick a rejection policy deliberately (§19) — know what a client experiences when the server is saturated (a fast `503`? A `CallerRunsPolicy`-induced slowdown?).
5. Set socket-level timeouts (§29) so a slow/idle/malicious client can't hold a thread hostage forever.
6. Confirm every downstream resource pool (the JDBC pool, §33) is sized *with* the thread pool in mind — more request threads than your DB pool can serve just moves queueing from the thread pool to the database driver.
7. Verify graceful shutdown (§21) actually drains in-flight work within your deployment platform's shutdown grace period (e.g. Kubernetes' `terminationGracePeriodSeconds`).
8. If idle-connection count, not CPU, is the actual bottleneck, reconsider whether §36's reactor model or §39's virtual threads is a better fit than tuning a platform-thread pool further.

---

# 65. Benchmarking Your Server

Configuration changes are only meaningful if you can measure their effect. Two widely-used HTTP load generators work well against MiniTomcat:

```bash
# wrk: high-throughput HTTP benchmarking tool
wrk -t4 -c200 -d30s http://localhost:8080/users/42
```

```bash
# ApacheBench: simpler, good for a quick sanity check
ab -n 10000 -c 100 http://localhost:8080/users/42
```

What to read from the output, and what it tells you about the sections above:

| Metric | What it reveals |
|---|---|
| Requests/sec (throughput) | Whether the thread pool (§18) and downstream JDBC pool (§33) are sized adequately for the offered load. |
| p50/p99/p999 latency | Tail latency spikes often point to GC pauses (§62), queueing at a saturated thread pool (§17, §19), or idle-timeout misconfiguration (§29). |
| Connection errors / resets | Backlog exhaustion (§13) or a rejection policy (§19) kicking in — expected under deliberate overload testing, alarming otherwise. |
| Behavior as concurrency (`-c`) scales past thread pool size | This is where you directly observe the backpressure behavior chosen in §19 — watch latency climb smoothly (`CallerRunsPolicy`) vs errors appear abruptly (`AbortPolicy`). |

Always benchmark keep-alive (`wrk`'s default) and non-keep-alive (`-H "Connection: close"`) separately — they exercise completely different parts of this guide (§27–§29 vs a fresh accept+handshake per request) and can produce very different throughput numbers for the same server.

---

# 66. Thread Safety Checklist for the Whole Stack

Every shared, mutable piece of state introduced in this guide needs an explicit answer to "how is this safe under concurrent worker threads?" — walking through them:

| Component | Shared state | Safety mechanism |
|---|---|---|
| `ServerThreadPool` (§16) | The executor's internal queue and worker set | `ThreadPoolExecutor` is internally thread-safe — no extra locking needed by us. |
| `RequestContextHolder` (§24) | The `ThreadLocal` map | Safe by construction — each thread only ever sees its own entry. |
| `MiniConnectionPool` (§33) | The `available` queue of connections | `ArrayBlockingQueue` is thread-safe; `Proxy`-wrapped `close()` always returns to the same safe queue. |
| `MiniServletContainer.pathToServlet` (§45) | The route table | Populated once at startup (§53) before `accept()` ever runs, then only **read** concurrently — no writes during traffic means no locking needed. |
| `MiniServletContext` attributes (§43) | `ConcurrentHashMap` | Explicitly chosen for safe concurrent `get`/`set` from any request thread. |
| A `@Controller`/`@Service`/`@Repository` bean itself | Any mutable instance field it declares | **Not automatically safe** — beans are singletons (MiniSpring companion guide §23) shared by every concurrent request; mutable instance state on a bean is a bug unless deliberately synchronized. |

That last row is the one developers most often get wrong when moving from a single-threaded toy server to a real concurrent one: a singleton `@Service` with a plain (non-thread-safe) instance field silently corrupts data under concurrent load, with no exception to warn you.

---

# 67. Common Mistakes

- **Mistake 1 — Unbounded thread creation.** `new Thread(...)` per connection with no pool (§14) works in a demo and falls over under real concurrency (§15).
- **Mistake 2 — Unbounded queues.** A `LinkedBlockingQueue` with no capacity behind a `ThreadPoolExecutor` defeats `maximumPoolSize` (§17) and turns overload into a slow-motion `OutOfMemoryError` instead of visible backpressure.
- **Mistake 3 — Forgetting `ThreadLocal.remove()`.** The single most common source of cross-request data leakage and memory growth in a pooled-thread server (§25–§26).
- **Mistake 4 — No socket timeout.** A client that opens a connection and sends nothing parks a worker thread forever (§29) — a trivial denial-of-service vector.
- **Mistake 5 — Mixing up the two connection pools.** Sizing the JDBC pool (§33) without regard to the HTTP thread pool's `maximumPoolSize` (§18), or vice versa, just moves the bottleneck between them instead of removing it.
- **Mistake 6 — Blocking the reactor thread.** In a `Selector`-based server (§36–§37), running a slow database call directly on the event-loop thread stalls *every* connection it manages, not just the current one.
- **Mistake 7 — Mutable state on a singleton bean.** Treating a `@Service`/`@Controller` like a per-request object when it is actually shared by every concurrent request (§66).
- **Mistake 8 — Reading the body with a `BufferedReader`/`InputStreamReader` past the headers.** Character-decoding buffering can silently consume raw bytes that belong to a binary or chunked body (§10).
- **Mistake 9 — No graceful shutdown.** Killing the process with active requests in flight instead of the three-step drain in §21, causing client-visible errors during every deploy.
- **Mistake 10 — Assuming `synchronized` is free on virtual threads.** A `synchronized` block that wraps blocking I/O can pin a virtual thread to its carrier on pre-JDK-24 runtimes, quietly reintroducing platform-thread-level contention (§40).

---

# 68. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| `HttpRequestParser` (§10–§11) | Well-formed requests, missing `Content-Length`, chunked bodies, malformed request lines, header edge cases (folded headers, duplicate headers) | Pure unit tests — feed a hand-built `InputStream` (e.g. `ByteArrayInputStream`) of raw bytes, assert the parsed `HttpRequest`. |
| `ServerThreadPool` sizing/rejection (§16–§19) | Behavior when the queue fills: does the configured `RejectedExecutionHandler` actually run? | Unit test with a tiny pool (`core=1, max=1, queue=1`) and submit blocking tasks to force saturation deterministically. |
| `RequestContextHolder` (§24–§26) | No leakage across two "requests" run sequentially on the same thread | Unit test: set a context, clear it, assert `get()` throws `IllegalStateException` afterward. |
| `MiniConnectionPool` (§33) | Borrowed connections are distinct; `close()` returns to the pool instead of really closing; pool blocks correctly when exhausted | Unit test against an embedded/in-memory database (e.g. H2) so no real network dependency is needed. |
| Keep-alive loop (§28–§29) | Multiple requests over one socket; idle timeout actually fires and closes cleanly | Integration test: open a raw `Socket` in the test, write two requests back-to-back, assert two responses come back over the same connection. |
| End-to-end (§52) | The full MiniTomcat + MiniSpring path for a real `@Controller` | Integration test: start `Application.main`-equivalent on a random port, hit it with a real HTTP client, assert the JSON body. |

The keep-alive and end-to-end tests are the ones most guides skip, and the ones most likely to catch the interaction bugs this guide is really about — a unit test on the parser in isolation cannot catch a `ThreadLocal` leaking across two requests sharing a pooled thread.

---

# 69. Progressive Interview Question Set

**Level 1 — Sockets & HTTP**
1. Walk through what happens, byte by byte, from `ServerSocket.accept()` returning to the first line of the HTTP request being available to your code.
2. Why can't you safely use `BufferedReader.readLine()` for the entire request, headers and body included?

**Level 2 — Concurrency**
3. What breaks first in a thread-per-connection server under heavy load — and at roughly what scale, on typical hardware?
4. Trace exactly what `ThreadPoolExecutor.execute()` does when `corePoolSize` is full, the queue has room, and later, when the queue is *also* full.
5. Why is an unbounded queue a footgun even though it looks like it "just works" under a load test that doesn't push it hard enough to notice?

**Level 3 — ThreadLocal**
6. Why does `ThreadLocal` become dangerous specifically once a thread *pool* is introduced, when it was harmless in the pure thread-per-connection model?
7. What's the difference between `threadLocal.set(null)` and `threadLocal.remove()`, and why does it matter at scale?

**Level 4 — Pooling**
8. Explain the difference between the server's HTTP keep-alive "connection pool" and a JDBC connection pool — what is each one saving you, mechanically?
9. Why does a JDBC connection pool's `close()` method not actually close the underlying socket?

**Level 5 — I/O models**
10. Why does NIO's `Selector` let one thread handle thousands of idle connections where a thread-per-connection model can't?
11. If virtual threads solve the same scaling problem as NIO without a state-machine rewrite, why would you ever still choose a `Selector`-based reactor?

**Level 6 — Servlet API & Integration**
12. What, precisely, is the contract between Tomcat and a `Servlet`? Where does that contract end and framework-specific behavior (like Spring MVC's `@Controller` routing) begin?
13. What would you have to implement to make Spring Boot start your own server instead of embedded Tomcat, and why is that a nontrivial amount of work even though the *idea* is simple?

**Final challenge:** Design a rate limiter that sits in front of `MiniServletContainer.dispatch()` (§45) as a `MiniFilter` (§46), bounded by both requests-per-second *and* total in-flight requests, that must not itself become a source of `ThreadLocal` leaks or thread-pool starvation. Explain your data structure choice and where each lock (if any) lives.

---

# 70. Final Architecture

```text
                              +----------------------------+
                              |         Application         |
                              |  .main() / Application.run  |
                              +---------------+--------------+
                                              |
                    starts                    | registers
        +--------------------------+          v
        |  MiniSpringContextListener|  +----------------------------+
        |  builds ApplicationContext|->|     MiniServletContext      |
        +--------------------------+  |  (shared attributes, §43)   |
                                       +---------------+--------------+
                                                       |
                                                       v
                                       +----------------------------+
                                       |    MiniServletContainer     |
                                       |  path -> Servlet routing    |
                                       |  filter chain (§46)         |
                                       +---------------+--------------+
                                                       |  "/"
                                                       v
                                       +----------------------------+
                                       |    MiniDispatcherServlet    |
                                       |  bridges to HandlerMapping  |
                                       +---------------+--------------+
                                                       |
                                                       v
                                       +----------------------------+
                                       | MiniSpring HandlerMapping   |
                                       | -> @Controller method       |
                                       +---------------+--------------+
                                                       |
                             +-------------------------+-------------------------+
                             |                                                   |
                     @Service / business logic                        @Repository -> MiniConnectionPool (§33)
                                                                                   -> Database

              Underneath all of the above, for every request:
    ServerSocket.accept() -> ServerThreadPool (§16) -> ConnectionWorker (§20)
        -> RequestContextHolder.set/clear (§24, §26) wraps the entire dispatch
        -> HttpRequestParser (§10) / HttpResponse.toBytes() (§61) at the wire boundary
```

---

# 71. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| TLS/HTTPS | Wrap the accepted `Socket` in an `SSLSocket` (`SSLServerSocketFactory`), negotiate the handshake before handing off to `HttpRequestParser` | `MiniTomcat`'s accept loop, §20 |
| HTTP/2 | Binary framing, multiplexed streams over one connection, HPACK header compression | Would replace §8–§11's text-based parser entirely for `h2` connections |
| WebSockets | An `Upgrade: websocket` handshake detected in `HttpRequestParser`, then handing the raw socket off to a persistent, bidirectional frame reader/writer instead of the request/response loop | A new servlet type alongside `MiniServlet` (§43), registered like any other path |
| Response compression | Detect `Accept-Encoding: gzip`, wrap the outgoing body in a `GZIPOutputStream` above a size threshold | `MiniHttpServletResponse.toWireResponse()`, §44 |
| Real Servlet spec coverage | `HttpSession`, multipart uploads, async `startAsync()` | Needed for the Spring Boot integration path in §56 to be genuinely complete |
| Metrics/observability | Request counters, latency histograms, thread-pool queue depth exposed via JMX or a `/metrics` endpoint | A `MiniFilter` (§46) plus periodic polling of `ThreadPoolExecutor.getActiveCount()`/`getQueue().size()` |
| Access logging | Structured request logs (method, path, status, latency, remote address) | Extend `LoggingFilter` (§46) |
| Config externalization | Port, pool sizes, timeouts read from a properties file instead of hardcoded constructor arguments | Mirrors MiniSpring companion guide §44's `@ConfigurationProperties` |

---

# 72. Final Takeaway

A production container like Tomcat is not one clever trick — it is the disciplined layering of a handful of ideas this guide built one at a time: **bound every resource** (threads in §16–§19, connections in §29 and §33, queues in §19), **isolate per-request state correctly** (`ThreadLocal`, §22–§26), **separate the transport from the framework** with a narrow contract (the Servlet API, §41–§47), and **fail predictably** under overload rather than silently (backpressure and rejection policies, §19; timeouts, §29; graceful shutdown, §21).

Once you've built MiniTomcat and wired MiniSpring on top of it (§48–§53), you have, in miniature, the exact same three-layer stack every real Spring Boot application runs on: **container → Servlet API → framework**. Understanding *why* each boundary is drawn where it is — not just that `@GetMapping` works — is the actual interview signal this whole exercise is built to produce.

