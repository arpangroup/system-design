# Build Your Own Java Web Server from Scratch
## From `ServerSocket` → HTTP Server → Servlet Container → Thread Pool → ThreadLocal → Connection Pool → MiniSpring Integration

> A step-by-step engineering guide for building a small Java web server similar in architectural concepts to Tomcat/Servlet containers, and integrating it with your own **MiniSpring** dependency-injection framework.

---

# Table of Contents

1. [What We Are Building](#1-what-we-are-building)
2. [Learning Objectives](#2-learning-objectives)
3. [Final Architecture](#3-final-architecture)
4. [Project Structure](#4-project-structure)
5. [Step 1 — Create a Raw TCP Server](#5-step-1--create-a-raw-tcp-server)
6. [Step 2 — Understand the Request Lifecycle](#6-step-2--understand-the-request-lifecycle)
7. [Step 3 — Parse HTTP Requests](#7-step-3--parse-http-requests)
8. [Step 4 — Create HttpRequest](#8-step-4--create-httprequest)
9. [Step 5 — Create HttpResponse](#9-step-5--create-httpresponse)
10. [Step 6 — Build a Basic HTTP Server](#10-step-6--build-a-basic-http-server)
11. [Step 7 — Add Routing](#11-step-7--add-routing)
12. [Step 8 — Introduce Handlers](#12-step-8--introduce-handlers)
13. [Step 9 — Understand Multithreading](#13-step-9--understand-multithreading)
14. [Step 10 — One Thread Per Request](#14-step-10--one-thread-per-request)
15. [Step 11 — Introduce ThreadPool](#15-step-11--introduce-threadpool)
16. [Step 12 — ThreadPool Tuning](#16-step-12--threadpool-tuning)
17. [Step 13 — Understand ThreadLocal](#17-step-13--understand-threadlocal)
18. [Step 14 — RequestContext](#18-step-14--requestcontext)
19. [Step 15 — Filters / Middleware](#19-step-15--filters--middleware)
20. [Step 16 — Servlet-like Abstraction](#20-step-16--servlet-like-abstraction)
21. [Step 17 — Servlet Container](#21-step-17--servlet-container)
22. [Step 18 — ApplicationContext](#22-step-18--applicationcontext)
23. [Step 19 — Integrate MiniSpring](#23-step-19--integrate-minispring)
24. [Step 20 — Dependency Injection into Controllers](#24-step-20--dependency-injection-into-controllers)
25. [Step 21 — Connection Pool](#25-step-21--connection-pool)
26. [Step 22 — Database Access](#26-step-22--database-access)
27. [Step 23 — Request → Controller → Service → Repository](#27-step-23--request--controller--service--repository)
28. [Step 24 — Exception Handling](#28-step-24--exception-handling)
29. [Step 25 — Server Lifecycle](#29-step-25--server-lifecycle)
30. [Step 26 — Graceful Shutdown](#30-step-26--graceful-shutdown)
31. [Step 27 — Keep-Alive](#31-step-27--keep-alive)
32. [Step 28 — HTTP Methods](#32-step-28--http-methods)
33. [Step 29 — Query Parameters](#33-step-29--query-parameters)
34. [Step 30 — Headers](#34-step-30--headers)
35. [Step 31 — JSON](#35-step-31--json)
36. [Step 32 — Authentication](#36-step-32--authentication)
37. [Step 33 — Request ID and Logging](#37-step-33--request-id-and-logging)
38. [Step 34 — Async Processing](#38-step-34--async-processing)
39. [Step 35 — Backpressure](#39-step-35--backpressure)
40. [Step 36 — Optimization](#40-step-36--optimization)
41. [Step 37 — Security](#41-step-37--security)
42. [Step 38 — MiniSpring Integration Architecture](#42-step-38--minispring-integration-architecture)
43. [Step 39 — Use the Server with Spring Boot](#43-step-39--use-the-server-with-spring-boot)
44. [Step 40 — Replace Embedded Tomcat](#44-step-40--replace-embedded-tomcat)
45. [Step 41 — Advanced Improvements](#45-step-41--advanced-improvements)
46. [Step 42 — Testing Strategy](#46-step-42--testing-strategy)
47. [Step 43 — Observability](#47-step-43--observability)
48. [Step 44 — Interview Questions](#48-step-44--interview-questions)
49. [Step 45 — System Design Questions](#49-step-45--system-design-questions)
50. [Final Architecture](#50-final-architecture)
51. [Suggested Learning Roadmap](#51-suggested-learning-roadmap)

---

# 1. What We Are Building

We will progressively build a Java web server.

The first version will be extremely small:

```text
Browser
   |
   | TCP
   v
ServerSocket
   |
   v
Socket
   |
   v
Read HTTP Request
   |
   v
Write HTTP Response
```

Eventually it becomes:

```text
                    ┌───────────────────────────────┐
                    │         MiniWebServer         │
                    │                               │
Client ────────────►│ ServerSocket                  │
                    │       │                       │
                    │       ▼                       │
                    │ ConnectionAcceptor            │
                    │       │                       │
                    │       ▼                       │
                    │ ThreadPoolExecutor            │
                    │       │                       │
                    │       ▼                       │
                    │ HttpRequestParser             │
                    │       │                       │
                    │       ▼                       │
                    │ FilterChain                   │
                    │       │                       │
                    │       ▼                       │
                    │ Router                        │
                    │       │                       │
                    │       ▼                       │
                    │ Dispatcher                    │
                    │       │                       │
                    │       ▼                       │
                    │ Controller / Servlet           │
                    │       │                       │
                    │       ▼                       │
                    │ Service                       │
                    │       │                       │
                    │       ▼                       │
                    │ Repository                    │
                    │       │                       │
                    │       ▼                       │
                    │ ConnectionPool                │
                    └───────────────────────────────┘
```

Along the way we will understand:

- TCP
- HTTP
- sockets
- blocking I/O
- multithreading
- thread pools
- thread lifecycle
- `ThreadLocal`
- request context
- filters
- routing
- servlet concepts
- dependency injection
- application context
- connection pools
- database connections
- graceful shutdown
- backpressure
- keep-alive
- authentication
- logging
- observability
- performance optimization

---

# 2. Learning Objectives

By completing this project you should understand why frameworks such as:

```text
Tomcat
Jetty
Undertow
Spring MVC
Spring Boot
```

have the architectural layers they do.

You should also understand the relationship:

```text
Operating System
       ↓
TCP Socket
       ↓
HTTP
       ↓
Web Server
       ↓
Servlet Container
       ↓
Spring MVC
       ↓
Spring ApplicationContext
       ↓
Application
```

---

# 3. Final Architecture

The final project will look approximately like this:

```text
                       Client
                         |
                         v
                  ┌──────────────┐
                  │ TCP Listener │
                  └──────┬───────┘
                         |
                         v
                  ┌──────────────┐
                  │ Accept Loop  │
                  └──────┬───────┘
                         |
                         v
                  ┌──────────────┐
                  │ Thread Pool   │
                  └──────┬───────┘
                         |
                         v
                ┌──────────────────┐
                │ Request Context  │
                │   ThreadLocal    │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ HTTP Parser      │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Filter Chain     │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Router           │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Dispatcher       │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Controller       │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Service          │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Repository       │
                └────────┬─────────┘
                         |
                         v
                ┌──────────────────┐
                │ Connection Pool  │
                └────────┬─────────┘
                         |
                         v
                     Database
```

---

# 4. Project Structure

Start with a Maven project.

```text
mini-web-server/
│
├── pom.xml
│
└── src/
    ├── main/
    │   └── java/
    │       └── com/
    │           └── example/
    │               └── webserver/
    │
    │                   ├── server/
    │                   │   ├── WebServer.java
    │                   │   ├── ConnectionHandler.java
    │                   │   └── ServerConfig.java
    │                   │
    │                   ├── http/
    │                   │   ├── HttpRequest.java
    │                   │   ├── HttpResponse.java
    │                   │   ├── HttpMethod.java
    │                   │   └── HttpParser.java
    │                   │
    │                   ├── routing/
    │                   │   ├── Router.java
    │                   │   └── Route.java
    │                   │
    │                   ├── servlet/
    │                   │   ├── MiniServlet.java
    │                   │   ├── MiniServletRequest.java
    │                   │   └── MiniServletResponse.java
    │                   │
    │                   ├── filter/
    │                   │   ├── Filter.java
    │                   │   └── FilterChain.java
    │                   │
    │                   ├── context/
    │                   │   ├── RequestContext.java
    │                   │   └── ApplicationContext.java
    │                   │
    │                   ├── db/
    │                   │   ├── ConnectionPool.java
    │                   │   └── PooledConnection.java
    │                   │
    │                   └── Main.java
    │
    └── test/
        └── java/
```

---

# 5. Step 1 — Create a Raw TCP Server

Before HTTP, understand TCP.

Java provides:

```java
ServerSocket
```

for accepting TCP connections.

Create:

```java
public class WebServer {

    public static void main(String[] args) throws Exception {

        ServerSocket serverSocket = new ServerSocket(8080);

        System.out.println("Server started on port 8080");

        while (true) {

            Socket socket = serverSocket.accept();

            System.out.println(
                    "Client connected: "
                            + socket.getRemoteSocketAddress()
            );

            socket.close();
        }
    }
}
```

Run:

```bash
curl http://localhost:8080
```

The connection will reach your server.

---

# 6. Step 2 — Understand the Request Lifecycle

When the browser sends:

```http
GET /hello HTTP/1.1
Host: localhost:8080
Connection: keep-alive
```

your server receives bytes.

Conceptually:

```text
Browser
   |
   | bytes
   v
TCP
   |
   v
Socket InputStream
   |
   v
HTTP Parser
   |
   v
HttpRequest
```

Your application should not deal with raw bytes after parsing.

Therefore:

```java
Socket
   ↓
HttpParser
   ↓
HttpRequest
```

---

# 7. Step 3 — Parse HTTP Requests

First implementation:

```java
public class HttpParser {

    public HttpRequest parse(InputStream inputStream)
            throws IOException {

        BufferedReader reader =
                new BufferedReader(
                        new InputStreamReader(inputStream)
                );

        String requestLine = reader.readLine();

        if (requestLine == null) {
            return null;
        }

        String[] parts = requestLine.split(" ");

        String method = parts[0];
        String path = parts[1];
        String version = parts[2];

        Map<String, String> headers = new HashMap<>();

        String line;

        while ((line = reader.readLine()) != null
                && !line.isEmpty()) {

            int separator = line.indexOf(':');

            if (separator > 0) {

                String key =
                        line.substring(0, separator).trim();

                String value =
                        line.substring(separator + 1).trim();

                headers.put(key, value);
            }
        }

        return new HttpRequest(
                method,
                path,
                version,
                headers
        );
    }
}
```

---

# 8. Step 4 — Create HttpRequest

```java
public class HttpRequest {

    private final String method;
    private final String path;
    private final String version;

    private final Map<String, String> headers;

    public HttpRequest(
            String method,
            String path,
            String version,
            Map<String, String> headers) {

        this.method = method;
        this.path = path;
        this.version = version;
        this.headers = headers;
    }

    public String getMethod() {
        return method;
    }

    public String getPath() {
        return path;
    }

    public String getVersion() {
        return version;
    }

    public String getHeader(String name) {
        return headers.get(name);
    }
}
```

Later we will add:

```text
query parameters
cookies
body
content type
remote address
request ID
authenticated user
```

---

# 9. Step 5 — Create HttpResponse

```java
public class HttpResponse {

    private int statusCode = 200;

    private String contentType =
            "text/plain";

    private byte[] body =
            new byte[0];

    public void status(int statusCode) {
        this.statusCode = statusCode;
    }

    public void contentType(String contentType) {
        this.contentType = contentType;
    }

    public void body(String body) {

        this.body =
                body.getBytes(StandardCharsets.UTF_8);
    }

    public void write(OutputStream outputStream)
            throws IOException {

        String response =
                "HTTP/1.1 " +
                statusCode +
                " OK\r\n" +

                "Content-Type: " +
                contentType +
                "\r\n" +

                "Content-Length: " +
                body.length +
                "\r\n" +

                "Connection: close\r\n" +

                "\r\n";

        outputStream.write(
                response.getBytes(StandardCharsets.UTF_8)
        );

        outputStream.write(body);

        outputStream.flush();
    }
}
```

---

# 10. Step 6 — Build a Basic HTTP Server

Now combine everything.

```java
public class WebServer {

    public static void main(String[] args)
            throws IOException {

        ServerSocket serverSocket =
                new ServerSocket(8080);

        HttpParser parser =
                new HttpParser();

        while (true) {

            Socket socket =
                    serverSocket.accept();

            try (socket) {

                HttpRequest request =
                        parser.parse(
                                socket.getInputStream()
                        );

                HttpResponse response =
                        new HttpResponse();

                response.body(
                        "Hello from Mini Web Server!"
                );

                response.write(
                        socket.getOutputStream()
                );
            }
        }
    }
}
```

Test:

```bash
curl http://localhost:8080
```

You have created a basic HTTP server.

---

# 11. Step 7 — Add Routing

We don't want:

```java
if (path.equals("/hello")) {
}
```

everywhere.

Create:

```java
public interface Handler {

    void handle(
            HttpRequest request,
            HttpResponse response
    );
}
```

Router:

```java
public class Router {

    private final Map<String, Handler> routes =
            new HashMap<>();

    public void get(
            String path,
            Handler handler) {

        routes.put("GET " + path, handler);
    }

    public Handler find(
            String method,
            String path) {

        return routes.get(
                method + " " + path
        );
    }
}
```

Usage:

```java
Router router = new Router();

router.get(
        "/hello",
        (request, response) ->
                response.body("Hello World")
);
```

---

# 12. Step 8 — Introduce Handlers

Now:

```text
HTTP Request
      |
      v
Router
      |
      v
Handler
```

Example:

```java
router.get(
        "/hello",
        (request, response) -> {

            response.contentType(
                    "text/plain"
            );

            response.body(
                    "Hello from /hello"
            );
        }
);
```

Another:

```java
router.get(
        "/users",
        (request, response) -> {

            response.contentType(
                    "application/json"
            );

            response.body(
                    "[{\"id\":1,\"name\":\"Arpan\"}]"
            );
        }
);
```

---

# 13. Step 9 — Understand Multithreading

Our current server has a major problem.

```java
while (true) {

    Socket socket =
            serverSocket.accept();

    handle(socket);
}
```

If one request takes:

```text
10 seconds
```

the next request waits.

Example:

```text
Request A
   |
   | 10 seconds
   v
Response A

Request B
   |
   | waiting
   |
   v
Request C
   |
   | waiting
```

This is unacceptable.

We need concurrency.

---

# 14. Step 10 — One Thread Per Request

Basic solution:

```java
while (true) {

    Socket socket =
            serverSocket.accept();

    Thread thread =
            new Thread(
                    () -> handle(socket)
            );

    thread.start();
}
```

Now:

```text
Request A → Thread A
Request B → Thread B
Request C → Thread C
Request D → Thread D
```

This works but creates another problem.

Suppose:

```text
10,000 requests
```

Then potentially:

```text
10,000 threads
```

This causes:

- memory pressure
- context switching
- CPU overhead
- scheduling overhead
- instability

Therefore we need a thread pool.

---

# 15. Step 11 — Introduce ThreadPool

Java provides:

```java
ExecutorService
```

Use:

```java
ExecutorService executor =
        Executors.newFixedThreadPool(100);
```

Server:

```java
while (true) {

    Socket socket =
            serverSocket.accept();

    executor.submit(
            () -> handle(socket)
    );
}
```

Now:

```text
              ThreadPool
          ┌─────────────────┐
Request A ┤                 │
Request B ┤                 │
Request C ┤  Worker Thread  │
Request D ┤                 │
Request E ┤                 │
          └─────────────────┘
```

Only a controlled number of requests execute concurrently.

---

# 16. Step 12 — ThreadPool Tuning

Do not blindly use:

```java
Executors.newFixedThreadPool(100);
```

Instead:

```java
ThreadPoolExecutor executor =
        new ThreadPoolExecutor(
                10,
                100,
                60,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1000),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
```

Understand the parameters:

```text
corePoolSize
maximumPoolSize
keepAliveTime
workQueue
rejectionPolicy
```

Architecture:

```text
Incoming Requests
       |
       v
+----------------+
| BlockingQueue  |
+----------------+
       |
       v
+----------------+
| Worker Threads |
+----------------+
```

---

# 17. Step 13 — Understand ThreadLocal

Suppose we want a request ID.

```text
Request A → requestId=A123
Request B → requestId=B456
```

We don't want to pass:

```java
requestId
```

through every method:

```java
controller(requestId)
service(requestId)
repository(requestId)
```

We can use:

```java
ThreadLocal
```

Example:

```java
public class RequestContextHolder {

    private static final ThreadLocal<RequestContext>
            CONTEXT =
            new ThreadLocal<>();

    public static void set(
            RequestContext context) {

        CONTEXT.set(context);
    }

    public static RequestContext get() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

---

# 18. Step 14 — RequestContext

```java
public class RequestContext {

    private final String requestId;

    private final Instant startTime;

    public RequestContext(
            String requestId) {

        this.requestId = requestId;

        this.startTime =
                Instant.now();
    }

    public String getRequestId() {
        return requestId;
    }

    public Instant getStartTime() {
        return startTime;
    }
}
```

Worker:

```java
executor.submit(() -> {

    try {

        RequestContext context =
                new RequestContext(
                        UUID.randomUUID().toString()
                );

        RequestContextHolder.set(context);

        handle(socket);

    } finally {

        RequestContextHolder.clear();

        socket.close();
    }
});
```

---

# Important ThreadLocal Rule

Always do:

```java
try {

    context.set(...);

}
finally {

    context.remove();

}
```

Why?

Because thread pools reuse threads.

Consider:

```text
Thread-1

Request A
   |
ThreadLocal = User A
   |
Request finished
   |
Thread reused
   |
Request B
```

If we don't clear:

```text
Request B
   |
ThreadLocal
   |
User A
```

This can cause:

- security bugs
- incorrect logging
- user-data leakage
- memory leaks

---

# 19. Step 15 — Filters / Middleware

We now need a mechanism similar to:

```text
Servlet Filter
Spring Interceptor
Middleware
```

Create:

```java
public interface Filter {

    void doFilter(
            HttpRequest request,
            HttpResponse response,
            FilterChain chain
    );
}
```

Filter chain:

```java
public class FilterChain {

    private final List<Filter> filters;

    private final Handler handler;

    private int index = 0;

    public FilterChain(
            List<Filter> filters,
            Handler handler) {

        this.filters = filters;
        this.handler = handler;
    }

    public void doFilter(
            HttpRequest request,
            HttpResponse response) {

        if (index < filters.size()) {

            Filter filter =
                    filters.get(index++);

            filter.doFilter(
                    request,
                    response,
                    this
            );

        } else {

            handler.handle(
                    request,
                    response
            );
        }
    }
}
```

---

# 20. Example Logging Filter

```java
public class LoggingFilter
        implements Filter {

    @Override
    public void doFilter(
            HttpRequest request,
            HttpResponse response,
            FilterChain chain) {

        long start =
                System.currentTimeMillis();

        try {

            chain.doFilter(
                    request,
                    response
            );

        } finally {

            long duration =
                    System.currentTimeMillis()
                    - start;

            System.out.println(
                    request.getMethod()
                    + " "
                    + request.getPath()
                    + " "
                    + duration
                    + "ms"
            );
        }
    }
}
```

Architecture:

```text
Request
  |
  v
LoggingFilter
  |
  v
AuthenticationFilter
  |
  v
AuthorizationFilter
  |
  v
Controller
```

---

# 21. Step 16 — Servlet-like Abstraction

Now introduce a Servlet-like abstraction.

```java
public interface MiniServlet {

    void service(
            MiniServletRequest request,
            MiniServletResponse response
    ) throws Exception;
}
```

Request:

```java
public class MiniServletRequest {

    private final HttpRequest request;

    public MiniServletRequest(
            HttpRequest request) {

        this.request = request;
    }

    public String getMethod() {
        return request.getMethod();
    }

    public String getPath() {
        return request.getPath();
    }

    public String getHeader(String name) {
        return request.getHeader(name);
    }
}
```

Response:

```java
public class MiniServletResponse {

    private final HttpResponse response;

    public MiniServletResponse(
            HttpResponse response) {

        this.response = response;
    }

    public void write(String body) {

        response.body(body);
    }

    public void status(int status) {

        response.status(status);
    }
}
```

---

# 22. Step 17 — Servlet Container

Now create:

```java
public class ServletContainer {

    private final Map<String, MiniServlet>
            servletMappings =
            new HashMap<>();

    public void register(
            String path,
            MiniServlet servlet) {

        servletMappings.put(
                path,
                servlet
        );
    }

    public MiniServlet find(
            String path) {

        return servletMappings.get(path);
    }
}
```

Register:

```java
container.register(
        "/hello",
        (request, response) ->
                response.write(
                        "Hello Servlet"
                )
);
```

Now:

```text
HTTP
 ↓
WebServer
 ↓
ServletContainer
 ↓
Servlet
```

You are beginning to recreate the fundamental servlet-container idea.

---

# 23. Step 18 — ApplicationContext

Now we introduce dependency injection.

Create:

```java
public class ApplicationContext {

    private final Map<Class<?>, Object>
            beans =
            new ConcurrentHashMap<>();

    public <T> void register(
            Class<T> type,
            T instance) {

        beans.put(type, instance);
    }

    public <T> T getBean(
            Class<T> type) {

        Object bean =
                beans.get(type);

        if (bean == null) {

            throw new RuntimeException(
                    "Bean not found: "
                    + type.getName()
            );
        }

        return type.cast(bean);
    }
}
```

Example:

```java
ApplicationContext context =
        new ApplicationContext();

UserService userService =
        new UserService();

context.register(
        UserService.class,
        userService
);
```

Retrieve:

```java
UserService service =
        context.getBean(
                UserService.class
        );
```

---

# 24. Step 19 — Integrate MiniSpring

This is where your own MiniSpring becomes extremely useful.

The responsibilities should be separated.

## MiniWebServer

Responsible for:

```text
TCP
HTTP
Threads
Request
Response
Routing
Filters
Servlet lifecycle
```

## MiniSpring

Responsible for:

```text
Bean creation
Dependency Injection
Component scanning
ApplicationContext
Bean lifecycle
Configuration
```

Do NOT make the web server responsible for dependency injection.

The architecture should be:

```text
              MiniWebServer
                    |
                    v
             DispatcherServlet
                    |
                    v
             MiniSpring
                    |
          +---------+---------+
          |         |         |
          v         v         v
      Controller  Service  Repository
```

---

# 25. Step 20 — Dependency Injection into Controllers

Suppose MiniSpring supports:

```java
@Component
public class UserService {

    public String getUser() {
        return "User 1";
    }
}
```

Controller:

```java
@Controller
public class UserController {

    private final UserService userService;

    public UserController(
            UserService userService) {

        this.userService = userService;
    }

    @GetMapping("/users")
    public String users() {

        return userService.getUser();
    }
}
```

MiniSpring should create:

```text
UserService
     |
     v
UserController
```

Then the web server invokes the controller.

---

# 26. Controller Registry

MiniSpring can expose controller metadata:

```java
public class ControllerRegistry {

    private final Map<String, ControllerMethod>
            mappings =
            new ConcurrentHashMap<>();

    public void register(
            String method,
            String path,
            ControllerMethod controller) {

        mappings.put(
                method + " " + path,
                controller
        );
    }

    public ControllerMethod find(
            String method,
            String path) {

        return mappings.get(
                method + " " + path
        );
    }
}
```

---

# 27. DispatcherServlet

Create:

```java
public class DispatcherServlet {

    private final ApplicationContext
            applicationContext;

    private final ControllerRegistry
            controllerRegistry;

    public DispatcherServlet(
            ApplicationContext applicationContext,
            ControllerRegistry controllerRegistry) {

        this.applicationContext =
                applicationContext;

        this.controllerRegistry =
                controllerRegistry;
    }

    public void dispatch(
            HttpRequest request,
            HttpResponse response)
            throws Exception {

        ControllerMethod method =
                controllerRegistry.find(
                        request.getMethod(),
                        request.getPath()
                );

        if (method == null) {

            response.status(404);

            response.body(
                    "Not Found"
            );

            return;
        }

        method.invoke(
                request,
                response
        );
    }
}
```

Now the architecture looks very similar conceptually to:

```text
HTTP Request
     |
     v
Servlet Container
     |
     v
DispatcherServlet
     |
     v
Handler Mapping
     |
     v
Controller
```

---

# 28. Controller Method Abstraction

```java
public interface ControllerMethod {

    void invoke(
            HttpRequest request,
            HttpResponse response)
            throws Exception;
}
```

Later MiniSpring can create this dynamically using:

```java
Method
```

reflection.

Example:

```java
Method method =
        controllerClass.getMethod(
                "users"
        );
```

Then:

```java
method.invoke(
        controllerBean
);
```

---

# 29. Annotation-Based Mapping

MiniSpring can support:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Controller {
}
```

and:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface GetMapping {

    String value();
}
```

Controller:

```java
@Controller
public class UserController {

    @GetMapping("/users")
    public String users() {

        return "users";
    }
}
```

MiniSpring startup:

```text
Scan packages
      |
      v
Find @Controller
      |
      v
Create bean
      |
      v
Find @GetMapping
      |
      v
Register route
```

---

# 30. The Critical Integration Point

Create an interface in the web server:

```java
public interface RequestDispatcher {

    void dispatch(
            HttpRequest request,
            HttpResponse response
    ) throws Exception;
}
```

The web server knows only this interface.

It does NOT know MiniSpring.

This is very important.

```text
                    ┌─────────────────┐
                    │  MiniWebServer  │
                    └────────┬────────┘
                             |
                             | RequestDispatcher
                             |
                    ┌────────▼────────┐
                    │    MiniSpring   │
                    └─────────────────┘
```

This follows Dependency Inversion.

---

# 31. MiniSpring Adapter

Create:

```java
public class MiniSpringRequestDispatcher
        implements RequestDispatcher {

    private final DispatcherServlet
            dispatcherServlet;

    public MiniSpringRequestDispatcher(
            DispatcherServlet dispatcherServlet) {

        this.dispatcherServlet =
                dispatcherServlet;
    }

    @Override
    public void dispatch(
            HttpRequest request,
            HttpResponse response)
            throws Exception {

        dispatcherServlet.dispatch(
                request,
                response
        );
    }
}
```

Now:

```text
MiniWebServer
      |
      v
RequestDispatcher
      |
      v
MiniSpringRequestDispatcher
      |
      v
MiniSpring DispatcherServlet
```

This is the cleanest integration boundary.

---

# 32. Step 21 — Connection Pool

Now we introduce database connections.

Never do:

```java
Connection connection =
        DriverManager.getConnection(...);
```

for every request.

Instead:

```text
Request
   |
   v
ConnectionPool
   |
   +---- Connection 1
   |
   +---- Connection 2
   |
   +---- Connection 3
   |
   +---- Connection 4
```

---

# 33. Basic ConnectionPool

```java
public class ConnectionPool {

    private final BlockingQueue<Connection>
            pool;

    public ConnectionPool(
            int size,
            String url,
            String username,
            String password)
            throws SQLException {

        pool =
                new ArrayBlockingQueue<>(size);

        for (int i = 0; i < size; i++) {

            Connection connection =
                    DriverManager.getConnection(
                            url,
                            username,
                            password
                    );

            pool.add(connection);
        }
    }

    public Connection borrow()
            throws InterruptedException {

        return pool.take();
    }

    public void release(
            Connection connection) {

        pool.offer(connection);
    }
}
```

---

# 34. Connection Pool Usage

```java
Connection connection =
        connectionPool.borrow();

try {

    // database operation

} finally {

    connectionPool.release(
            connection
    );
}
```

This is extremely important.

Never forget:

```java
finally
```

Otherwise the pool eventually becomes empty.

---

# 35. Thread Pool vs Connection Pool

These solve different problems.

## Thread Pool

Controls:

```text
How many requests execute concurrently?
```

Example:

```text
100 worker threads
```

## Connection Pool

Controls:

```text
How many database connections exist?
```

Example:

```text
20 DB connections
```

They are independent resources.

Example:

```text
100 worker threads
        |
        v
20 database connections
```

Potentially:

```text
20 requests → database
80 requests → waiting
```

This is an important system-design concept.

---

# 36. Database Architecture

Final application:

```text
Request
   |
   v
ThreadPool
   |
   v
Controller
   |
   v
Service
   |
   v
Repository
   |
   v
ConnectionPool
   |
   v
Database
```

---

# 37. Step 22 — Database Access

Repository:

```java
@Repository
public class UserRepository {

    private final ConnectionPool connectionPool;

    public UserRepository(
            ConnectionPool connectionPool) {

        this.connectionPool =
                connectionPool;
    }

    public User findById(
            long id)
            throws Exception {

        Connection connection =
                connectionPool.borrow();

        try {

            PreparedStatement statement =
                    connection.prepareStatement(
                            """
                            SELECT id, name
                            FROM users
                            WHERE id = ?
                            """
                    );

            statement.setLong(1, id);

            ResultSet resultSet =
                    statement.executeQuery();

            if (resultSet.next()) {

                return new User(
                        resultSet.getLong("id"),
                        resultSet.getString("name")
                );
            }

            return null;

        } finally {

            connectionPool.release(
                    connection
            );
        }
    }
}
```

---

# 38. Step 23 — Request → Controller → Service → Repository

This is the architecture we ultimately want:

```text
GET /users/10
       |
       v
MiniWebServer
       |
       v
ThreadPool
       |
       v
RequestContext
       |
       v
FilterChain
       |
       v
DispatcherServlet
       |
       v
UserController
       |
       v
UserService
       |
       v
UserRepository
       |
       v
ConnectionPool
       |
       v
PostgreSQL
```

---

# 39. Step 24 — Exception Handling

Never allow exceptions to kill the worker thread silently.

Create:

```java
public class ExceptionHandler {

    public void handle(
            Exception exception,
            HttpResponse response) {

        exception.printStackTrace();

        response.status(500);

        response.body(
                """
                {
                  "error": "Internal Server Error"
                }
                """
        );
    }
}
```

Worker:

```java
try {

    dispatcher.dispatch(
            request,
            response
    );

} catch (Exception e) {

    exceptionHandler.handle(
            e,
            response
    );
}
```

---

# 40. Step 25 — Server Lifecycle

Our server should not just have:

```java
while (true)
```

Create:

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}
```

Server:

```java
public class MiniWebServer
        implements Lifecycle {

    private volatile boolean running;

    @Override
    public void start() {

        running = true;

        // start accept loop
    }

    @Override
    public void stop() {

        running = false;

        // cleanup resources
    }

    @Override
    public boolean isRunning() {
        return running;
    }
}
```

---

# 41. Step 26 — Graceful Shutdown

Graceful shutdown means:

```text
Stop accepting new requests
          |
          v
Allow existing requests to finish
          |
          v
Shutdown ThreadPool
          |
          v
Close connection pool
          |
          v
Close ServerSocket
```

Example:

```java
public void shutdown() {

    running = false;

    executor.shutdown();

    try {

        if (!executor.awaitTermination(
                30,
                TimeUnit.SECONDS)) {

            executor.shutdownNow();
        }

    } catch (InterruptedException e) {

        Thread.currentThread().interrupt();

        executor.shutdownNow();
    }

    connectionPool.close();

    try {

        serverSocket.close();

    } catch (IOException ignored) {
    }
}
```

---

# 42. Step 27 — Keep-Alive

Current implementation:

```http
Connection: close
```

means:

```text
Request
  ↓
Response
  ↓
Socket closes
```

HTTP keep-alive allows:

```text
Socket
  |
  +-- Request 1
  |
  +-- Response 1
  |
  +-- Request 2
  |
  +-- Response 2
  |
  +-- Request 3
```

This reduces TCP connection overhead.

---

# 43. Keep-Alive Loop

Conceptually:

```java
while (keepAlive) {

    HttpRequest request =
            parser.parse(inputStream);

    if (request == null) {
        break;
    }

    HttpResponse response =
            new HttpResponse();

    dispatcher.dispatch(
            request,
            response
    );

    response.write(
            outputStream
    );
}
```

Now your connection handler manages multiple HTTP requests.

---

# 44. Step 28 — HTTP Methods

Create:

```java
public enum HttpMethod {

    GET,
    POST,
    PUT,
    PATCH,
    DELETE,
    HEAD,
    OPTIONS
}
```

Router should support:

```java
router.get(
        "/users",
        handler
);

router.post(
        "/users",
        handler
);

router.put(
        "/users/{id}",
        handler
);

router.delete(
        "/users/{id}",
        handler
);
```

---

# 45. Step 29 — Query Parameters

Request:

```http
GET /users?page=2&size=20
```

Represent:

```java
Map<String, String> queryParameters;
```

Then:

```java
request.getQueryParam("page");
```

returns:

```text
2
```

---

# 46. Step 30 — Headers

Support:

```java
request.getHeader(
        "Authorization"
);
```

Examples:

```text
Authorization
Content-Type
Accept
User-Agent
Host
Cookie
X-Request-ID
```

Use case:

```java
String authorization =
        request.getHeader(
                "Authorization"
        );
```

---

# 47. Step 31 — JSON

Don't manually construct JSON everywhere.

Introduce:

```java
JsonSerializer
JsonDeserializer
```

Architecture:

```text
JSON
 ↓
Object
 ↓
Controller
 ↓
Object
 ↓
JSON
```

Example:

```java
public class JsonResponseWriter {

    public void write(
            Object object,
            HttpResponse response) {

        String json =
                objectMapper.writeValueAsString(
                        object
                );

        response.contentType(
                "application/json"
        );

        response.body(json);
    }
}
```

---

# 48. Step 32 — Authentication

Add:

```text
AuthenticationFilter
```

Flow:

```text
Request
   |
   v
AuthenticationFilter
   |
   +---- Invalid → 401
   |
   v
AuthorizationFilter
   |
   +---- Forbidden → 403
   |
   v
Controller
```

Authentication information can be stored in:

```java
RequestContext
```

Example:

```java
requestContext.setUser(
        authenticatedUser
);
```

Then:

```java
RequestContextHolder
        .get()
        .getUser();
```

---

# 49. Step 33 — Request ID and Logging

Generate:

```java
String requestId =
        UUID.randomUUID().toString();
```

Store:

```java
RequestContext
```

Then logs can contain:

```text
requestId=abc123
method=GET
path=/users
duration=15ms
status=200
```

This is the foundation of distributed tracing.

---

# 50. Step 34 — Async Processing

Suppose:

```text
HTTP request
     |
     v
Long-running operation
     |
     | 30 seconds
     v
Response
```

This occupies a worker thread.

Instead:

```text
HTTP request
     |
     v
Submit job
     |
     v
Return job ID
```

Example:

```json
{
  "jobId": "12345",
  "status": "PROCESSING"
}
```

Worker:

```text
HTTP ThreadPool
       |
       v
Application Task Executor
       |
       v
Background Task
```

---

# 51. Step 35 — Backpressure

Suppose:

```text
100,000 requests
```

but:

```text
100 threads
```

and queue:

```text
1000
```

What happens to request 1001?

We need a rejection strategy.

```java
new ThreadPoolExecutor(
        20,
        100,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

Other policies:

```text
AbortPolicy
CallerRunsPolicy
DiscardPolicy
DiscardOldestPolicy
```

This is a very important production concept.

---

# 52. Step 36 — Optimization

After functionality works, optimize.

## 52.1 Avoid unnecessary object creation

Bad:

```java
new ObjectMapper()
```

per request.

Better:

```java
ObjectMapper
```

as a shared singleton.

---

## 52.2 Reuse threads

Use:

```java
ThreadPoolExecutor
```

instead of:

```java
new Thread()
```

per request.

---

## 52.3 Reuse database connections

Use:

```text
ConnectionPool
```

instead of opening a connection per request.

---

## 52.4 Buffer I/O

Use:

```java
BufferedInputStream
BufferedOutputStream
```

where appropriate.

---

## 52.5 Avoid unnecessary synchronization

Bad:

```java
synchronized
```

around every request.

Use concurrent structures:

```java
ConcurrentHashMap
BlockingQueue
AtomicInteger
LongAdder
```

when appropriate.

---

## 52.6 Avoid ThreadLocal where unnecessary

ThreadLocal is useful for:

```text
RequestContext
logging context
security context
```

but it should not become a global dumping ground.

---

## 52.7 Reduce lock contention

Instead of:

```java
synchronized(sharedMap)
```

consider:

```java
ConcurrentHashMap
```

---

## 52.8 Use immutable request objects

Prefer:

```java
HttpRequest
```

as immutable.

This makes concurrent processing safer.

---

# 53. Blocking vs Non-Blocking Architecture

Our server is currently:

```text
Blocking I/O
+
Thread Pool
```

Architecture:

```text
1000 connections
       |
       v
100 worker threads
```

A more advanced architecture uses:

```text
Java NIO
Selector
Channel
ByteBuffer
```

Architecture:

```text
10,000 connections
       |
       v
Selector
       |
       +---- Event 1
       +---- Event 2
       +---- Event 3
```

Then:

```text
Event Loop
```

can handle many connections.

This leads toward the architectural concepts used by:

```text
Netty
Reactor
Undertow
```

---

# 54. Step 37 — Security

A production web server needs protection against:

```text
Request smuggling
Header injection
Malformed HTTP
Oversized headers
Oversized body
Slow clients
Connection exhaustion
Thread exhaustion
Database connection exhaustion
Path traversal
CRLF injection
```

Always configure limits:

```text
maxHeaderSize
maxBodySize
maxRequestLineSize
maxConnections
requestTimeout
idleTimeout
```

Example:

```java
public class ServerConfig {

    private int port = 8080;

    private int maxThreads = 100;

    private int queueCapacity = 1000;

    private int maxHeaderSize = 8192;

    private long maxBodySize = 10 * 1024 * 1024;

    private Duration requestTimeout =
            Duration.ofSeconds(30);
}
```

---

# 55. Step 38 — MiniSpring Integration Architecture

Your MiniSpring project should become responsible for:

```text
MiniSpring
│
├── BeanFactory
├── ApplicationContext
├── ComponentScanner
├── BeanDefinition
├── BeanPostProcessor
├── DependencyResolver
├── Configuration
├── ControllerRegistry
└── DispatcherServlet
```

Web server:

```text
MiniWebServer
│
├── ServerSocket
├── ConnectionAcceptor
├── ThreadPool
├── HttpParser
├── HttpRequest
├── HttpResponse
├── FilterChain
└── RequestDispatcher
```

Database:

```text
MiniData
│
├── ConnectionPool
├── TransactionManager
├── JdbcTemplate
└── Repository
```

---

# 56. Recommended Module Architecture

Eventually split the project:

```text
mini-framework/
│
├── mini-core/
│
├── mini-context/
│
├── mini-web/
│
├── mini-jdbc/
│
└── mini-boot/
```

Conceptually:

```text
mini-core
    |
    v
mini-context
    |
    +----------------+
    |                |
    v                v
mini-web         mini-jdbc
    |
    v
mini-boot
```

---

# 57. Step 39 — Use the Server with Spring Boot

There are two different meanings of this question.

## Option A — Run your server inside Spring Boot

This is easy.

Spring Boot starts:

```text
Spring ApplicationContext
```

and you create your server as a bean.

Example:

```java
@Configuration
public class WebServerConfiguration {

    @Bean
    public MiniWebServer miniWebServer(
            RequestDispatcher dispatcher) {

        return new MiniWebServer(
                8080,
                dispatcher
        );
    }
}
```

Start it:

```java
@Component
public class WebServerStarter
        implements ApplicationRunner {

    private final MiniWebServer server;

    public WebServerStarter(
            MiniWebServer server) {

        this.server = server;
    }

    @Override
    public void run(
            ApplicationArguments args) {

        server.start();
    }
}
```

Architecture:

```text
Spring Boot
    |
    +--------------------+
    |                    |
    v                    v
Spring MVC           MiniWebServer
                         |
                         v
                   Your Dispatcher
```

Both can coexist, although they should normally use different ports.

Example:

```text
Spring Boot
localhost:8080

MiniWebServer
localhost:9090
```

---

# 58. Option B — Use Your Server as Spring Boot's Web Server

This is much more advanced.

Spring Boot normally uses an embedded web server such as:

```text
Tomcat
Jetty
Undertow
```

To replace it with your own implementation, you would need to integrate with Spring Boot's embedded-web-server abstraction and provide the expected server/container lifecycle and Servlet infrastructure.

Your server would need to behave sufficiently like a Servlet-capable embedded container.

Conceptually:

```text
Spring Boot
     |
     v
ServletWebServerFactory
     |
     v
YourMiniWebServer
     |
     v
Servlet Container
```

This is considerably more work than simply embedding your server as another component.

---

# 59. Option C — Don't Implement Servlet API

You can instead create:

```text
Spring Boot
      |
      v
Your Web Server
      |
      v
Your RequestDispatcher
      |
      v
Spring ApplicationContext
```

But this is no longer standard Spring MVC integration.

You would essentially build:

```text
Your own Spring-like web framework
```

which is actually a very good learning exercise.

---

# 60. Recommended Integration Strategy

For your MiniSpring project, I recommend doing this in stages.

## Stage 1

```text
MiniWebServer
     |
     v
Mini Handler
```

## Stage 2

```text
MiniWebServer
     |
     v
RequestDispatcher
     |
     v
MiniSpring
```

## Stage 3

```text
@Controller
@GetMapping
@PostMapping
```

## Stage 4

```text
@Controller
    |
    v
Service
    |
    v
Repository
```

## Stage 5

```text
ConnectionPool
```

## Stage 6

```text
Transactions
```

## Stage 7

```text
Async
```

## Stage 8

```text
NIO
```

This keeps each architectural concept understandable.

---

# 61. Step 40 — Replace Embedded Tomcat

If you eventually want:

```text
Spring Boot
     |
     v
Your Web Server
```

you need to understand Spring Boot's embedded web-server mechanism.

At a high level:

```text
SpringApplication
       |
       v
WebApplicationContext
       |
       v
ServletWebServerFactory
       |
       v
WebServer
```

Tomcat integration implements these abstractions.

Your implementation would provide an equivalent adapter.

---

# 62. Why Not Directly Replace Tomcat First?

Because you would simultaneously need to solve:

```text
HTTP
Servlet lifecycle
ServletContext
ServletConfig
ServletRequest
ServletResponse
Filters
Listeners
Sessions
Error handling
Async Servlet
Multipart
WebSocket
HTTP/2
TLS
Security
```

That makes learning harder.

First build:

```text
MiniWebServer
```

Then:

```text
MiniServletContainer
```

Then:

```text
MiniSpringWeb
```

Only afterwards study:

```text
Spring Boot Embedded Server integration
```

---

# 63. Step 41 — Advanced Improvements

After the basic version works, add the following.

## 63.1 Path variables

Support:

```text
/users/{id}
```

Request:

```text
/users/123
```

Controller:

```java
@GetMapping("/users/{id}")
```

Extract:

```text
id = 123
```

---

# 64. Request Body

Support:

```http
POST /users
Content-Type: application/json
```

Body:

```json
{
  "name": "Arpan"
}
```

Then:

```java
request.getBody();
```

---

# 65. Content Negotiation

Support:

```http
Accept: application/json
```

versus:

```http
Accept: text/html
```

Response can be selected based on:

```text
Accept
Content-Type
```

---

# 66. Cookie Support

Parse:

```http
Cookie: SESSION_ID=abc123
```

and provide:

```java
request.getCookie(
        "SESSION_ID"
);
```

---

# 67. Session Management

Create:

```java
SessionManager
```

Architecture:

```text
Cookie
  |
  v
SESSION_ID
  |
  v
SessionManager
  |
  v
Session
```

Example:

```java
session.set(
        "userId",
        123L
);
```

---

# 68. Static File Server

Support:

```text
GET /index.html
GET /css/app.css
GET /js/app.js
```

Architecture:

```text
Request
   |
   +---- /api/* → Controller
   |
   +---- /static/* → StaticFileHandler
```

Be extremely careful about path traversal.

---

# 69. Compression

Support:

```http
Accept-Encoding: gzip
```

Then:

```text
Response
   |
   v
GZIP
   |
   v
Compressed Response
```

---

# 70. HTTPS

Eventually support:

```text
SSLServerSocket
```

or:

```text
SSLEngine
```

Architecture:

```text
Client
  |
 HTTPS
  |
  v
TLS
  |
  v
HTTP
```

---

# 71. HTTP/2

After HTTP/1.1 is understood, study:

```text
HTTP/2
```

Concepts:

```text
Streams
Frames
Multiplexing
HPACK
Flow control
```

---

# 72. WebSocket

Add:

```text
HTTP Upgrade
     |
     v
WebSocket
```

Architecture:

```text
Client
   |
HTTP handshake
   |
   v
WebSocket Session
   |
   +---- Message
   +---- Message
   +---- Message
```

---

# 73. Virtual Threads

Modern Java provides:

```java
Thread.ofVirtual()
```

and:

```java
Executors.newVirtualThreadPerTaskExecutor()
```

You can experiment with:

```java
try (ExecutorService executor =
        Executors.newVirtualThreadPerTaskExecutor()) {

    executor.submit(
            () -> handle(socket)
    );
}
```

This gives an excellent opportunity to compare:

```text
Platform Threads
vs
Virtual Threads
```

Study:

```text
CPU-bound workloads
I/O-bound workloads
ThreadLocal behavior
Synchronization
Connection pools
Pinning
```

---

# 74. Virtual Threads and Connection Pools

A common mistake is thinking:

```text
Virtual threads
=
Unlimited database connections
```

Not true.

You might have:

```text
10,000 virtual threads
```

but:

```text
50 database connections
```

The database remains a limited resource.

Therefore:

```text
Virtual Thread Pool
        |
        v
Connection Pool
        |
        v
Database
```

still requires resource management.

---

# 75. Step 42 — Testing Strategy

Create unit tests for:

```text
HttpParser
Router
FilterChain
RequestContext
ConnectionPool
ControllerRegistry
DispatcherServlet
```

Example:

```java
@Test
void shouldRouteGetRequest() {

    Router router =
            new Router();

    router.get(
            "/hello",
            (req, res) ->
                    res.body("hello")
    );

    Handler handler =
            router.find(
                    "GET",
                    "/hello"
            );

    assertNotNull(handler);
}
```

---

# 76. Integration Tests

Start the real server:

```java
server.start();
```

Then:

```java
HttpClient
```

call:

```text
http://localhost:8080/hello
```

Verify:

```text
HTTP 200
body = hello
```

---

# 77. Concurrency Testing

Send:

```text
100
1000
10000
```

requests concurrently.

Measure:

```text
throughput
latency
CPU
memory
thread count
queue size
database connections
```

---

# 78. Load Testing

Use tools such as:

```text
Apache JMeter
wrk
k6
Gatling
```

Example conceptual command:

```bash
wrk -t4 -c100 -d30s http://localhost:8080/hello
```

Study:

```text
Requests/sec
Latency
Errors
CPU utilization
```

---

# 79. Step 43 — Observability

Expose:

```text
/health
/metrics
```

Example metrics:

```text
server.requests.total
server.requests.active
server.requests.failed
server.request.duration
server.threadpool.active
server.threadpool.queue
server.connectionpool.active
server.connectionpool.idle
```

Architecture:

```text
Application
     |
     +---- Logs
     |
     +---- Metrics
     |
     +---- Traces
```

---

# 80. Request Timing

Use:

```java
long start =
        System.nanoTime();

try {

    dispatch();

} finally {

    long duration =
            System.nanoTime() - start;
}
```

Convert:

```java
TimeUnit.NANOSECONDS
        .toMillis(duration);
```

---

# 81. Trace Context

Eventually your:

```java
RequestContext
```

can contain:

```text
requestId
traceId
spanId
user
tenant
startTime
```

Example:

```java
public class RequestContext {

    private String requestId;

    private String traceId;

    private String spanId;

    private Object authenticatedUser;

    private Instant startTime;
}
```

This creates the foundation for distributed tracing.

---

# 82. Step 44 — Interview Questions

After implementing each stage, ask yourself these questions.

## Basic

### Question 1

What is the difference between:

```text
Socket
ServerSocket
```

### Question 2

What happens internally when:

```java
serverSocket.accept();
```

is called?

### Question 3

Why does:

```java
InputStream.read()
```

block?

---

# 83. Multithreading Questions

### Question 4

Why is one-thread-per-request problematic?

### Question 5

Why do we need a ThreadPool?

### Question 6

What happens when the thread pool queue becomes full?

### Question 7

What is the difference between:

```text
corePoolSize
maximumPoolSize
queue capacity
```

### Question 8

What happens if a request takes 60 seconds?

---

# 84. ThreadLocal Questions

### Question 9

Why use ThreadLocal?

### Question 10

Why must ThreadLocal be cleared?

### Question 11

Why is ThreadLocal dangerous with thread pools?

### Question 12

Can ThreadLocal context automatically propagate to another thread?

Study:

```text
ThreadLocal
InheritableThreadLocal
ScopedValue
```

---

# 85. Connection Pool Questions

### Question 13

Why use a connection pool?

### Question 14

What happens when all connections are busy?

### Question 15

What should happen when:

```text
ThreadPool = 100
ConnectionPool = 10
```

?

### Question 16

Can increasing database connections always improve performance?

No.

Eventually the database itself becomes the bottleneck.

---

# 86. Servlet Questions

### Question 17

What is a Servlet?

### Question 18

What does a Servlet Container do?

### Question 19

What is the Servlet lifecycle?

Conceptually:

```text
load
  ↓
init
  ↓
service
  ↓
destroy
```

### Question 20

What is the relationship between:

```text
Tomcat
Servlet
Spring MVC
DispatcherServlet
```

?

---

# 87. Spring Questions

### Question 21

Why should MiniWebServer not create controllers?

Because that belongs to the application/container layer.

### Question 22

Why should MiniSpring not manage TCP sockets?

Because that belongs to the web infrastructure layer.

### Question 23

Where should dependency inversion happen?

At the boundary:

```java
RequestDispatcher
```

---

# 88. Architecture Questions

### Question 24

What happens if:

```text
10,000 requests
100 threads
1000 queue
```

arrive?

### Question 25

What happens if:

```text
100 threads
10 DB connections
```

are used?

### Question 26

What happens if:

```text
DB response time = 5 seconds
```

?

### Question 27

How would you prevent one slow endpoint from exhausting all worker threads?

Possible solutions:

```text
separate executor
timeouts
bulkheads
rate limits
circuit breakers
```

---

# 89. Step 45 — System Design Questions

Once your server works, try designing the following.

## Level 1

Design:

```text
HTTP server
```

## Level 2

Design:

```text
Servlet container
```

## Level 3

Design:

```text
Thread pool
```

## Level 4

Design:

```text
Connection pool
```

## Level 5

Design:

```text
Request tracing system
```

## Level 6

Design:

```text
Rate limiter
```

## Level 7

Design:

```text
API Gateway
```

## Level 8

Design:

```text
Reverse Proxy
```

## Level 9

Design:

```text
Load Balancer
```

---

# 90. Suggested Enhancement — Rate Limiter

Add:

```java
RateLimiter
```

Architecture:

```text
Request
   |
   v
RateLimiter
   |
   +---- rejected
   |
   v
ThreadPool
```

Possible algorithms:

```text
Fixed Window
Sliding Window
Token Bucket
Leaky Bucket
```

Implement each one.

---

# 91. Suggested Enhancement — Circuit Breaker

Suppose database is failing.

Without protection:

```text
Request
   |
   v
DB
   |
 timeout
   |
Thread remains busy
```

Eventually:

```text
All threads busy
```

Circuit breaker:

```text
CLOSED
   |
 failures
   v
OPEN
   |
 timeout
   v
HALF_OPEN
```

This is a great MiniSpring enhancement.

---

# 92. Suggested Enhancement — Bulkhead

Separate thread pools:

```text
                    Application
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
       UserPool      PaymentPool   ReportPool
```

If reports become slow:

```text
ReportPool exhausted
```

but:

```text
UserPool
PaymentPool
```

continue working.

---

# 93. Suggested Enhancement — Async Context

When a task moves from:

```text
Thread A
```

to:

```text
Thread B
```

you need to decide what happens to:

```text
requestId
traceId
security context
tenant
```

Design:

```java
ContextSnapshot snapshot =
        ContextSnapshot.capture();

executor.submit(() -> {

    snapshot.restore();

    try {

        task.run();

    } finally {

        snapshot.clear();
    }
});
```

This is an excellent advanced exercise.

---

# 94. Suggested Enhancement — Configuration

Create:

```yaml
server:
  port: 8080
  workerThreads: 100
  queueCapacity: 1000
  maxHeaderSize: 8192
  maxBodySize: 10485760
  keepAlive: true
  requestTimeout: 30s

database:
  poolSize: 20
  connectionTimeout: 5s
```

Then:

```java
ServerConfig
DatabaseConfig
```

are created by MiniSpring.

---

# 95. Suggested Enhancement — Auto Configuration

This is where your MiniSpring can start looking like Spring Boot.

For example:

```java
@AutoConfiguration
public class WebServerAutoConfiguration {

    @Bean
    public MiniWebServer miniWebServer(
            ServerConfig config,
            RequestDispatcher dispatcher) {

        return new MiniWebServer(
                config,
                dispatcher
        );
    }
}
```

Then application code only needs:

```java
@SpringBootApplication
public class Application {
}
```

Your MiniSpring framework can automatically configure:

```text
WebServer
ThreadPool
Router
Dispatcher
ConnectionPool
```

---

# 96. Suggested Enhancement — Embedded Server Starter

Create:

```text
mini-web-starter
```

Application:

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>mini-web-starter</artifactId>
</dependency>
```

Then:

```java
@MiniSpringApplication
public class Application {
}
```

Automatically starts:

```text
MiniSpring
   |
   +---- ApplicationContext
   |
   +---- ComponentScanner
   |
   +---- MiniWebServer
   |
   +---- DispatcherServlet
```

This is essentially the beginning of your own:

```text
Spring Boot
```

---

# 97. Final MiniSpring + MiniWebServer Architecture

The final architecture should look like:

```text
                       Application
                            |
                            v
                  ┌───────────────────┐
                  │    MiniSpring     │
                  │                   │
                  │ ApplicationContext│
                  │ BeanFactory       │
                  │ ComponentScanner  │
                  │ DI Container      │
                  └─────────┬─────────┘
                            |
                            v
                  ┌───────────────────┐
                  │ DispatcherServlet │
                  └─────────┬─────────┘
                            |
                            v
                    ControllerRegistry
                            |
                            v
                      Controllers
                            |
                            v
                        Services
                            |
                            v
                      Repositories
                            |
                            v
                    ConnectionPool
                            |
                            v
                         Database


       ┌────────────────────────────────────────┐
       │             MiniWebServer              │
       │                                        │
       │ ServerSocket                           │
       │     ↓                                  │
       │ ConnectionAcceptor                     │
       │     ↓                                  │
       │ ThreadPool                             │
       │     ↓                                  │
       │ RequestContext                         │
       │     ↓                                  │
       │ HttpParser                             │
       │     ↓                                  │
       │ FilterChain                            │
       │     ↓                                  │
       │ RequestDispatcher ─────────────────────┘
       │                                        │
       └────────────────────────────────────────┘
```

---

# 98. Complete Request Lifecycle

A request should eventually flow like this:

```text
1. Client
      |
      v

2. TCP connection
      |
      v

3. ServerSocket.accept()
      |
      v

4. Worker ThreadPool
      |
      v

5. Create RequestContext
      |
      v

6. ThreadLocal.set(context)
      |
      v

7. Parse HTTP
      |
      v

8. Create HttpRequest
      |
      v

9. FilterChain
      |
      +---- Logging
      |
      +---- Authentication
      |
      +---- Authorization
      |
      +---- Rate Limiting
      |
      v

10. RequestDispatcher
      |
      v

11. MiniSpring DispatcherServlet
      |
      v

12. Controller
      |
      v

13. Service
      |
      v

14. Repository
      |
      v

15. ConnectionPool.borrow()
      |
      v

16. Database
      |
      v

17. ConnectionPool.release()
      |
      v

18. Controller Response
      |
      v

19. HttpResponse
      |
      v

20. Socket OutputStream
      |
      v

21. Client

22. ThreadLocal.remove()

23. Thread returns to ThreadPool
```

---

# 99. The Most Important Concepts to Understand

Don't just implement the code.

For every component ask:

## ServerSocket

```text
Who accepts connections?
```

## Socket

```text
How does TCP communication happen?
```

## Thread

```text
Who executes a request?
```

## ThreadPool

```text
How many requests can execute concurrently?
```

## ThreadLocal

```text
How do we associate context with the current execution thread?
```

## RequestContext

```text
What information belongs to a single request?
```

## Filter

```text
What cross-cutting behavior should happen around requests?
```

## Router

```text
Which application handler should execute?
```

## Dispatcher

```text
How do we invoke application code?
```

## ApplicationContext

```text
How are application objects created and connected?
```

## ConnectionPool

```text
How do we control access to the database?
```

## Graceful Shutdown

```text
How do we stop the application without corrupting work?
```

---

# 100. Recommended Implementation Order

Do NOT implement everything at once.

Follow this exact progression.

```text
Phase 1
-------
ServerSocket
Socket
InputStream
OutputStream


Phase 2
-------
HTTP Parser
HttpRequest
HttpResponse


Phase 3
-------
Router
Handler


Phase 4
-------
One Thread Per Request


Phase 5
-------
ThreadPoolExecutor


Phase 6
-------
RequestContext
ThreadLocal


Phase 7
-------
FilterChain


Phase 8
-------
MiniServlet


Phase 9
-------
ServletContainer


Phase 10
--------
ApplicationContext


Phase 11
--------
Controller
Service
Repository


Phase 12
--------
ConnectionPool


Phase 13
--------
Exception Handling


Phase 14
--------
Graceful Shutdown


Phase 15
--------
Keep-Alive


Phase 16
--------
JSON


Phase 17
--------
Authentication


Phase 18
--------
Metrics / Logging


Phase 19
--------
Rate Limiter


Phase 20
--------
Circuit Breaker


Phase 21
--------
Virtual Threads


Phase 22
--------
Java NIO


Phase 23
--------
HTTPS


Phase 24
--------
HTTP/2


Phase 25
--------
Spring Boot Integration
```

---

# 101. Final Challenge

After completing all stages, try to answer this without looking at the code:

> A client sends `GET /users/123` to your server. Explain every important step from the operating system receiving the TCP packet until the response reaches the client.

Your answer should contain:

```text
TCP
 ↓
ServerSocket
 ↓
Socket
 ↓
ThreadPool
 ↓
RequestContext
 ↓
ThreadLocal
 ↓
HTTP Parser
 ↓
HttpRequest
 ↓
FilterChain
 ↓
Router
 ↓
DispatcherServlet
 ↓
Controller
 ↓
Service
 ↓
Repository
 ↓
ConnectionPool
 ↓
Database
 ↓
Repository
 ↓
Service
 ↓
Controller
 ↓
HttpResponse
 ↓
Socket
 ↓
Client
```

If you can explain every arrow, you have understood the architecture rather than merely memorized the code.

---

# 102. Final Architecture Comparison

## Your Mini Web Server

```text
ServerSocket
    +
ThreadPool
    +
HTTP Parser
    +
Router
    +
Servlet
    +
Filter
```

## Tomcat-like Architecture

```text
Network Layer
    +
Connector
    +
Protocol Handler
    +
Thread Pool
    +
Servlet Container
    +
Servlet
    +
Filter
    +
Lifecycle
```

## Spring MVC

```text
Servlet Container
       +
DispatcherServlet
       +
HandlerMapping
       +
Controller
       +
Service
       +
Repository
```

## Spring Boot

```text
SpringApplication
       +
ApplicationContext
       +
AutoConfiguration
       +
Embedded Web Server
       +
Spring MVC
```

## Your Final MiniSpring

```text
MiniSpringApplication
       +
MiniApplicationContext
       +
AutoConfiguration
       +
MiniWebServer
       +
DispatcherServlet
       +
Controller
       +
Service
       +
Repository
       +
ConnectionPool
```

---

# 103. What You Should Implement in Your MiniSpring

The most valuable next enhancements are:

### 1. `@RestController`

```java
@RestController
public class UserController {
}
```

### 2. `@GetMapping`

```java
@GetMapping("/users")
```

### 3. `@PostMapping`

```java
@PostMapping("/users")
```

### 4. `@PathVariable`

```java
@GetMapping("/users/{id}")
```

### 5. `@RequestParam`

```java
@GetMapping("/users")
public User get(
        @RequestParam("id") Long id
)
```

### 6. `@RequestBody`

```java
@PostMapping("/users")
public User create(
        @RequestBody CreateUserRequest request
)
```

### 7. Exception Handler

```java
@ExceptionHandler
```

### 8. Interceptor

```java
HandlerInterceptor
```

### 9. Authentication

```text
SecurityFilter
```

### 10. Transaction Management

```java
@Transactional
```

### 11. Connection Pool

```text
DataSource
```

### 12. Configuration

```java
@Value
@ConfigurationProperties
```

### 13. Profiles

```text
dev
test
prod
```

### 14. Auto Configuration

```text
@EnableAutoConfiguration
```

### 15. Starter Modules

```text
mini-web-starter
mini-jdbc-starter
mini-security-starter
```

---

# 104. The Bigger Learning Goal

The ultimate goal isn't really:

> "Build a web server."

The bigger goal is to understand how the following pieces fit together:

```text
Operating System
       ↓
TCP/IP
       ↓
Socket
       ↓
HTTP
       ↓
Web Server
       ↓
Concurrency
       ↓
Thread Pool
       ↓
Request Context
       ↓
Servlet
       ↓
Application Context
       ↓
Dependency Injection
       ↓
MVC
       ↓
Database
       ↓
Connection Pool
       ↓
Transactions
       ↓
Security
       ↓
Observability
       ↓
Distributed Systems
```

Once you understand this chain, frameworks such as Spring Boot become much easier to reason about.

---

# 105. Recommended Final Project

Create one repository:

```text
mini-platform/
│
├── mini-core/
│
├── mini-context/
│
├── mini-web/
│
├── mini-jdbc/
│
├── mini-security/
│
├── mini-boot/
│
├── sample-application/
│
└── README.md
```

Sample application:

```text
sample-application
        |
        v
UserController
        |
        v
UserService
        |
        v
UserRepository
        |
        v
MiniJdbcTemplate
        |
        v
ConnectionPool
        |
        v
PostgreSQL
```

Web layer:

```text
Client
  |
  v
MiniWebServer
  |
  v
ThreadPool
  |
  v
RequestContext
  |
  v
FilterChain
  |
  v
DispatcherServlet
  |
  v
Controller
```

Framework layer:

```text
MiniSpring
  |
  +-- ComponentScanner
  +-- BeanFactory
  +-- ApplicationContext
  +-- DependencyResolver
  +-- ControllerRegistry
  +-- Configuration
  +-- Lifecycle
```

---

# 106. Final Milestone Checklist

Use this checklist while implementing.

## Networking

- [ ] ServerSocket
- [ ] Socket
- [ ] InputStream
- [ ] OutputStream
- [ ] TCP connection lifecycle

## HTTP

- [ ] HTTP request parser
- [ ] HTTP response writer
- [ ] HTTP methods
- [ ] Headers
- [ ] Query parameters
- [ ] Request body
- [ ] Status codes
- [ ] Keep-alive

## Concurrency

- [ ] Thread per request
- [ ] ExecutorService
- [ ] ThreadPoolExecutor
- [ ] BlockingQueue
- [ ] Rejection policy
- [ ] ThreadLocal
- [ ] RequestContext
- [ ] Virtual threads

## Web Framework

- [ ] Router
- [ ] Handler
- [ ] Filter
- [ ] FilterChain
- [ ] Servlet
- [ ] ServletContainer
- [ ] DispatcherServlet
- [ ] Controller registry

## MiniSpring

- [ ] ApplicationContext
- [ ] Component scanning
- [ ] Bean creation
- [ ] Dependency injection
- [ ] Constructor injection
- [ ] Controller discovery
- [ ] Request mapping
- [ ] Lifecycle management

## Database

- [ ] ConnectionPool
- [ ] Borrow connection
- [ ] Release connection
- [ ] Repository
- [ ] Transaction
- [ ] Connection timeout
- [ ] Pool exhaustion handling

## Production

- [ ] Graceful shutdown
- [ ] Request timeout
- [ ] Connection timeout
- [ ] Maximum body size
- [ ] Maximum header size
- [ ] Authentication
- [ ] Authorization
- [ ] Rate limiting
- [ ] Circuit breaker
- [ ] Bulkhead
- [ ] Metrics
- [ ] Logging
- [ ] Request ID
- [ ] Trace ID

## Advanced

- [ ] NIO
- [ ] Selector
- [ ] Non-blocking I/O
- [ ] HTTPS
- [ ] HTTP/2
- [ ] WebSocket
- [ ] Compression
- [ ] Static files
- [ ] Async processing
- [ ] Spring Boot integration

---

# 107. Final Takeaway

Build the project incrementally:

```text
ServerSocket
     ↓
HTTP Server
     ↓
Concurrent HTTP Server
     ↓
ThreadPool HTTP Server
     ↓
ThreadLocal Request Context
     ↓
Filter Chain
     ↓
Servlet Container
     ↓
DispatcherServlet
     ↓
MiniSpring
     ↓
Connection Pool
     ↓
Database
     ↓
Security
     ↓
Observability
     ↓
Virtual Threads
     ↓
NIO
     ↓
Spring Boot Integration
```

The most important architectural principle is:

```text
                 ┌───────────────────┐
                 │   MiniWebServer   │
                 │                   │
                 │ Network + HTTP    │
                 └─────────┬─────────┘
                           │
                           │ interface
                           ▼
                 ┌───────────────────┐
                 │RequestDispatcher  │
                 └─────────┬─────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │    MiniSpring     │
                 │                   │
                 │ DI + MVC + Beans  │
                 └─────────┬─────────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Controller      Service      Repository
                                          |
                                          ▼
                                   ConnectionPool
                                          |
                                          ▼
                                      Database
```

**Keep the web server independent from MiniSpring.**

That single decision gives you a clean architecture, follows SOLID/Dependency Inversion, and makes it possible to use the same server with:

```text
Plain Java
   +
MiniSpring
   +
Potentially Spring Boot
   +
Other frameworks
```

without coupling the networking layer to the dependency-injection framework.

---
## End of Guide