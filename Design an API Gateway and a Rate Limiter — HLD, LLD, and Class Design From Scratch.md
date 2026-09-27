# Design an API Gateway and a Rate Limiter — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design an API gateway that sits in front of a set of microservices. Cover routing, authentication, load balancing, and resilience. Then — this is usually where the interview gets serious — design the rate limiter the gateway uses to protect those services, in real depth: the algorithm, and how it stays correct across many gateway instances running at once."*
>
> This guide is structured exactly as that interview unfolds: a requirements-gathering phase, a high-level gateway architecture built up decision by decision, and then a deep low-level dive into rate limiting specifically — because that second half is almost always where a candidate's design either holds up under real pressure or quietly falls apart.

---

# 1. What We Are Building

We are building **MiniGate** — an API gateway and its rate limiter, covering:

- **Functional requirements**: request routing, a pluggable cross-cutting-concern pipeline, load balancing, service discovery integration, circuit breaking, edge authentication, and a correct, production-grade rate limiter.
- **High-level design**: why a gateway exists at all, the filter-chain architecture that makes it extensible, and how every cross-cutting concern (auth, rate limiting, logging) plugs into that one pipeline without touching the others.
- **Low-level design**: a genuinely pluggable filter chain (Chain of Responsibility), a Strategy-based load balancer, a State-pattern circuit breaker, and — the deep dive — five real rate-limiting algorithms building from a naive fixed window up to a distributed, atomically-correct token bucket that stays correct across dozens of gateway instances.
- **Scaling concerns**: the exact race condition that breaks a naive distributed rate limiter, atomicity via a Lua script, a local/distributed hybrid to cut latency, and sharding the shared counter store.

```text
                              Client
                                |
                          API Gateway (this guide)
                    +-----------+-----------+
                    |    Filter Chain       |
                    |  auth -> rate-limit -> |
                    |  routing -> logging    |
                    +-----------+-----------+
                                |
                    Load Balancer + Circuit Breaker
                                |
        +---------+---------+---------+---------+
        v         v         v         v         v
     Orders   Inventory  Payments   Users    Notifications
    Service    Service    Service   Service     Service
                                |
                    Shared Rate-Limit Store
                    (Redis-style, atomic Lua script)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Justify why an API gateway exists at all — not as a buzzword, but as a specific answer to specific cross-cutting-concern duplication across services.
- Design a genuinely pluggable filter pipeline (the Chain of Responsibility pattern) so adding a new cross-cutting concern never touches existing filters.
- Design load balancing and circuit breaking as pluggable, testable components (Strategy and State), not inline `if/else` logic bolted onto the routing code.
- Explain, precisely, why a single-node in-memory rate limiter is wrong the moment there is more than one gateway instance — and fix it correctly with a shared store and an atomic check-and-increment.
- Compare five rate-limiting algorithms (fixed window, sliding window log, sliding window counter, token bucket, leaky bucket) by their actual accuracy/memory/burst-tolerance tradeoffs, not by name recognition alone.
- Reason about what breaks first at millions of requests per second across thousands of tenants, and name the specific technique (sharding, a local/distributed hybrid) that addresses it.

---

# 3. Why This Matters (The Interview, Framed)

"Design an API gateway" is a common senior/staff systems-design question precisely because it has a shallow answer and a deep one, and the interview is designed to find out which one a candidate actually has. The shallow answer is a box labeled "gateway" with arrows to some services. The deep answer requires:

- **Architecture that's genuinely extensible** — a real gateway accumulates cross-cutting concerns over its lifetime (auth today, rate limiting next quarter, request signing after that); a design that requires editing a central class for each new one doesn't survive contact with a real roadmap.
- **Resilience reasoning** — a gateway sits in the request path of *everything*; a naive design turns "one backend service is unhealthy" into "the entire platform is unhealthy," which is precisely the failure mode a circuit breaker exists to prevent.
- **The rate limiter, in real depth** — this is where the interview usually escalates hardest, because "just count requests and reject over the limit" sounds trivial until the very next question is *"you have 50 gateway instances. Where does that count even live?"* — and the honest answer to that question is where most of the genuinely hard engineering in this entire guide lives.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's interfaces and pattern implementations (Chain of Responsibility, Strategy, State). |
| Shared rate-limit store | An in-memory key-value store with atomic scripting (Redis-style, `EVAL`/Lua) | A gateway cluster needs one shared, atomically-updated counter per rate-limit key — exactly what Redis's single-threaded execution plus Lua scripting guarantees (§51). |
| Service discovery | A registry with health checks (Consul/Eureka-style) | The gateway must route only to instances known to be alive right now (§23). |
| Load balancer transport | Plain HTTP/1.1 or HTTP/2 to backends | The gateway's own concern is *which* instance to call, not the wire protocol — this guide stays protocol-agnostic on purpose. |
| Observability | Structured logs + a correlation ID propagated end to end | Every filter in the chain (§15-§16) can attach context to one traceable request. |

---

# 5. Project Structure

```text
minigate/
├── src/main/java/com/example/minigate/
│   ├── filter/
│   │   ├── Filter.java, FilterChain.java                  // §16
│   │   ├── AuthenticationFilter.java                       // §31
│   │   ├── RateLimitFilter.java                             // §56
│   │   └── LoggingFilter.java
│   ├── routing/
│   │   ├── Route.java, RouteRegistry.java                  // §18-§19
│   │   └── RouteConfigWatcher.java                          // §21
│   ├── discovery/
│   │   └── ServiceDiscoveryClient.java, HealthChecker.java  // §23
│   ├── loadbalancer/
│   │   ├── LoadBalancer.java                                // §26
│   │   ├── RoundRobinLoadBalancer.java, WeightedLoadBalancer.java, LeastConnectionsLoadBalancer.java
│   ├── resilience/
│   │   └── CircuitBreaker.java, CircuitState.java           // §28-§29
│   ├── auth/
│   │   └── ApiKeyValidator.java, JwtValidator.java          // §31
│   └── ratelimit/
│       ├── RateLimitAlgorithm.java                          // §38
│       ├── FixedWindowCounter.java                          // §38
│       ├── SlidingWindowLog.java, SlidingWindowCounter.java // §40-§41
│       ├── TokenBucket.java, LeakyBucket.java                // §44-§45
│       ├── DistributedTokenBucket.java (Lua-script backed)  // §51
│       ├── HybridRateLimiter.java                            // §53
│       └── RateLimitKeyResolver.java                         // §55
└── src/test/java/com/example/minigate/
    ├── FilterChainOrderingTest.java
    ├── RateLimiterAccuracyTest.java
    └── DistributedRateLimiterRaceConditionTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"How many backend services and gateway instances are we talking about — tens, or thousands? Is rate limiting per API key, per user, per IP, or all three layered together? Do we need hard limits (reject over the limit) or soft/bursty limits (allow short bursts above the sustained rate)? Is exact accuracy required, or is a reasonable approximation acceptable if it buys much lower latency?"*

Exactly as [the Workflow Automation guide's opening](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) and [the JIRA-style guide's opening](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) both argue, narrowing an intentionally broad prompt before designing anything is the first move that separates a strong answer from a shallow one. For this guide, we settle on a concrete, realistic scope: **dozens of gateway instances** behind a load balancer, **hundreds of backend services**, rate limiting **layered by API key, user, and route simultaneously** (§55), **bursts allowed up to a bucket capacity** (favoring the token bucket, §43-§44), and **a reasonable, bounded approximation accepted in exchange for lower added latency** (§53's hybrid design) — the realistic, industry-standard tradeoff, not an unrealistically perfect one.

---

# 7. Functional Requirements

- **Route** incoming requests to the correct backend service, by path and/or host.
- Support a **pluggable pipeline** of cross-cutting concerns (authentication, rate limiting, logging, request/response transformation) applied to every request, in a configurable order.
- **Discover** healthy backend instances dynamically, and stop sending traffic to instances that fail health checks.
- **Load balance** across multiple healthy instances of the same service, with more than one selection strategy available.
- **Circuit break** on a backend that is failing repeatedly, to protect both that backend and the gateway's own resources.
- **Authenticate** requests at the edge (API key or JWT) before they ever reach a backend service.
- **Rate limit** requests, correctly, across every gateway instance in the cluster — not per-instance, but per logical client, cluster-wide.
- Update **routes** and **rate-limit configuration** without restarting the gateway.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Low added latency** | The gateway itself should add single-digit milliseconds, not become the platform's bottleneck | The local/distributed hybrid rate limiter (§53), non-blocking filter chain (§16) |
| **High availability** | No single gateway instance, and no single backend instance, is a single point of failure | Multiple gateway instances behind their own load balancer; the circuit breaker (§28-§29) |
| **Correctness under concurrency** | A rate limit must hold even when many gateway instances check it at the same moment | Atomic distributed rate limiting (§49-§51) |
| **Extensibility** | Adding a new cross-cutting concern must never require modifying existing filters | The Filter Chain (Chain of Responsibility, §15-§16) |
| **Fairness** | One noisy client must never exhaust capacity meant for others | Per-key rate limiting (§55), layered limits |

---

# 9. Follow-up Question 1 — "What Are the Core Responsibilities Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Before services and diagrams — what does 'the gateway' actually have to DO, as a list of responsibilities? Not components. Responsibilities."*

This is the same deliberate pivot [the Workflow Automation guide's own Follow-up 1](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) makes — naming responsibilities before naming classes keeps the design honest about what actually needs solving, rather than reaching for familiar component names first.

---

# 10. Identifying the Core Domain Entities and Responsibilities

| Entity / Responsibility | Represents | Key relationships |
|---|---|---|
| **Request** | One incoming HTTP call | Flows through the `FilterChain` (§16), matched against a `Route` |
| **Route** | A mapping from a path/host pattern to a backend service | Held in the `RouteRegistry` (§19), resolved once per request |
| **Filter** | One cross-cutting concern (auth, rate limit, logging) | Composed into an ordered `FilterChain` (§15-§16) |
| **ServiceInstance** | One running, discoverable instance of a backend service | Tracked by `ServiceDiscoveryClient` (§23), health-checked continuously |
| **LoadBalancer** | The strategy for picking one `ServiceInstance` among several healthy ones | Pluggable (§26), consulted once per request after routing |
| **CircuitBreaker** | Per-backend failure-tracking state machine | Wraps every call to a `ServiceInstance` (§28-§29) |
| **RateLimiter** | The thing that decides "allow" or "reject," per key, per request | The deep-dive subject of Part 2 (§34 onward) |
| **RateLimitKey** | What a limit is actually measured against (an API key, a user, a route, or a composite) | Resolved once per request (§55), before the limiter is ever consulted |

Every section from §11 onward either builds infrastructure **around** these entities or builds the entities **themselves** as real, working class designs — nothing introduced later is untraceable back to this table.

---

# 11. High-Level Architecture Overview

```text
                                    Client
                                       |
                          Load Balancer (in front of the gateway cluster itself)
                                       |
                    +------------------+------------------+
                    v                  v                  v
              Gateway Instance   Gateway Instance   Gateway Instance
                    |                  |                  |
                    +------------------+------------------+
                                       |
                          Shared Rate-Limit Store (§48, one per cluster, NOT per instance)
                                       |
                          Service Discovery + Health Checks (§23)
                                       |
                    +------------------+------------------+
                    v                  v                  v
               Orders Service    Payments Service    Users Service
              (N instances)       (N instances)       (N instances)
```

The single most important line in this diagram is the **shared rate-limit store**, sitting *outside* every individual gateway instance — every other piece of this architecture (routing, load balancing, circuit breaking) can be entirely per-instance and still be correct, but rate limiting specifically cannot, which §46-§51 exist to prove and fix.

---

# 12. Follow-up Question 2 — "Why Not Just Let Clients Call Services Directly? What Does a Gateway Actually Buy You?"

> **Interviewer:** *"You have five services. Why not just give clients five URLs? What does inserting a gateway in the middle actually solve?"*

The honest answer is **not** "because that's the standard architecture" — it's a specific, concrete cost that duplicating cross-cutting concerns across every service imposes. §13 names that cost precisely.

---

# 13. The Case for an API Gateway: Cross-Cutting Concerns Centralized

Without a gateway, every one of the five services in §12 needs its **own** authentication logic, its **own** rate limiting, its **own** request logging and correlation-ID handling — five copies of the same concern, each one an independent place for a bug or an inconsistency to creep in (service A validates a JWT slightly differently than service B; service C forgot to rate-limit an endpoint entirely). An API gateway centralizes exactly these **cross-cutting** concerns — the ones that apply uniformly across services and have nothing to do with any single service's actual business logic — into one place, built once, tested once, applied consistently everywhere. This is precisely the same "duplication across boundaries is the actual cost, not a stylistic preference" argument [the JIRA-style guide's Observer-pattern section](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) makes for notification side effects — just applied at the network-boundary level instead of the in-process event level.

---

# 14. Follow-up Question 3 — "Walk Me Through One Request, End to End, Through the Gateway."

> **Interviewer:** *"A request for `GET /orders/42` arrives. Trace it through your gateway, step by step, until a response goes back to the client."*

§15-§16 build the mechanism that makes this trace possible to describe precisely rather than hand-wave: a **filter chain**, each filter handling exactly one cross-cutting concern, composed in a fixed, explicit order.

---

# 15. The Filter Chain: Chain of Responsibility for Cross-Cutting Concerns

Every cross-cutting concern — authentication, rate limiting, logging, request transformation — is a `Filter`, and every request passes through an ordered chain of them before ever reaching routing and load balancing. This is the textbook **Chain of Responsibility** pattern: each filter decides whether to continue the chain, short-circuit it (reject the request), or wrap the eventual response on the way back out.

```text
Request
   |
   v
AuthenticationFilter --(unauthenticated)--> reject: 401
   |
   v
RateLimitFilter --(over limit)--> reject: 429
   |
   v
LoggingFilter (attaches a correlation ID, logs the request)
   |
   v
Routing + Load Balancing + Circuit Breaker (§17-§29)
   |
   v
Backend Service
   |
   v  (response flows back UP through the same chain, in reverse)
LoggingFilter (logs the response, timing)
   |
   v
Response to Client
```

---

# 16. Implementing the Filter Pipeline in Java

```java
public interface Filter {
    void doFilter(GatewayRequest request, GatewayResponse response, FilterChain chain) throws GatewayException;
}
```

```java
public final class FilterChain {

    private final List<Filter> filters;
    private final int index;
    private final BackendInvoker backendInvoker; // §17-§29 -- routing, load balancing, circuit breaking

    private FilterChain(List<Filter> filters, int index, BackendInvoker backendInvoker) {
        this.filters = filters;
        this.index = index;
        this.backendInvoker = backendInvoker;
    }

    public static FilterChain of(List<Filter> filters, BackendInvoker backendInvoker) {
        return new FilterChain(filters, 0, backendInvoker);
    }

    public void doFilter(GatewayRequest request, GatewayResponse response) throws GatewayException {
        if (index >= filters.size()) {
            backendInvoker.invoke(request, response); // end of the chain -- actually call the backend
            return;
        }
        Filter current = filters.get(index);
        FilterChain next = new FilterChain(filters, index + 1, backendInvoker);
        current.doFilter(request, response, next);
    }
}
```

Each filter never knows how many filters come after it, or what they do — it only ever calls `chain.doFilter(request, response)` to continue, or writes a rejection directly to `response` and simply **doesn't** call `chain.doFilter(...)` to short-circuit. Adding a new cross-cutting concern (request signing, say) is: implement `Filter`, insert it into the configured list at the right position — nothing about `FilterChain` itself, or any existing filter, ever changes. This is the Open/Closed Principle (§64) made concrete at the very center of the gateway's design.

---

# 17. Follow-up Question 4 — "How Does the Gateway Know Where to Route a Request?"

> **Interviewer:** *"`GET /orders/42` comes in. How does the gateway know that means 'call the Orders service'?"*

§18-§19 build a `Route` model and a registry that matches an incoming path/host against configured patterns — the mechanism the end of §15's filter chain hands off to.

---

# 18. The Route Registry: Path- and Host-Based Matching

```java
public record Route(String routeId, String pathPattern, String hostPattern /* nullable */, String serviceName,
                     List<String> filterOverrides /* nullable -- per-route filter tweaks */) { }
```

A `pathPattern` like `/orders/**` or `/users/{id}/profile` is matched against the incoming request's path; an optional `hostPattern` additionally requires a specific `Host` header, supporting the common case of one gateway fronting multiple domains, each with its own route table.

---

# 19. Implementing Route Matching and a Route Registry

```java
public final class RouteRegistry {

    private volatile List<Route> routes; // §21 -- volatile so a config reload is visible to every request thread immediately

    public RouteRegistry(List<Route> initialRoutes) {
        this.routes = List.copyOf(initialRoutes);
    }

    public Optional<Route> match(GatewayRequest request) {
        return routes.stream()
                .filter(route -> hostMatches(route, request) && pathMatches(route, request))
                .findFirst(); // first-match-wins -- route ORDER is a real, documented part of the configuration contract
    }

    private boolean hostMatches(Route route, GatewayRequest request) {
        return route.hostPattern() == null || route.hostPattern().equals(request.host());
    }

    private boolean pathMatches(Route route, GatewayRequest request) {
        return AntStylePathMatcher.matches(route.pathPattern(), request.path()); // `**`/`{var}` semantics, elided here
    }

    public void replaceAll(List<Route> newRoutes) {
        this.routes = List.copyOf(newRoutes); // §21 -- an atomic, single-write swap, never a partial in-place edit
    }
}
```

`replaceAll` swapping the entire list in one atomic reference write — rather than mutating the existing list in place — means a concurrent `match()` call from another request thread always sees either the *complete* old route table or the *complete* new one, never a route table half-updated mid-reload. This is the same "never let a reader observe a partially-applied change" discipline the [TinyDB Storage Engine guide's WAL-before-flush rule](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) enforces for a different kind of durable state.

---

# 20. Follow-up Question 5 — "Routes Change as Services Deploy. How Do You Update Routing Without Restarting the Gateway?"

> **Interviewer:** *"A new service ships, or an existing route's path changes. You can't restart every gateway instance for every deploy. How does route configuration actually get updated live?"*

§19's `replaceAll` already made the *application* of a new route table atomic and safe; §21 covers *where the new table comes from* and how it reaches every running instance.

---

# 21. Dynamic Route Configuration and Hot Reload

A `RouteConfigWatcher` subscribes to a configuration source (a config-management service, a versioned file in object storage, or a dedicated config database) and, on any change, fetches the full new route list and calls `RouteRegistry.replaceAll` (§19) — never an incremental patch, which would require correctly diffing additions, removals, and re-orderings and risks applying a diff out of order relative to another concurrent update. Every gateway instance runs its **own** `RouteConfigWatcher`, polling or subscribing independently — routing configuration is deliberately **not** something that needs the same cluster-wide atomicity guarantee the rate limiter does (§48), because a route table briefly disagreeing between two gateway instances for a few seconds during a rollout is a minor, self-correcting inconsistency, not a correctness violation the way a rate limit being briefly wrong would be.

---

# 22. Follow-up Question 6 — "How Does the Gateway Find a Healthy Instance of the Backend Service?"

> **Interviewer:** *"A route says 'send this to the Orders service.' The Orders service has 12 running instances, and 2 of them are currently unhealthy. How does the gateway know that?"*

§23 answers this with a discovery client and a continuous health-check loop, so "which instances exist" and "which instances are actually usable right now" are tracked as two separate, continuously-updated facts.

---

# 23. Service Discovery Integration and Health Checks

```java
public record ServiceInstance(String instanceId, String host, int port, boolean healthy) { }

public interface ServiceDiscoveryClient {
    List<ServiceInstance> instancesFor(String serviceName); // only instances currently believed healthy
}
```

```java
public final class HealthChecker {

    private final Map<String, ServiceInstance> instancesById = new ConcurrentHashMap<>();
    private final HttpClient httpClient;

    public HealthChecker(HttpClient httpClient) { this.httpClient = httpClient; }

    /** Called on a fixed schedule (e.g. every 5 seconds) by a background thread, per registered instance. */
    public void checkOne(ServiceInstance instance) {
        boolean healthy = pingHealthEndpoint(instance);
        instancesById.put(instance.instanceId(), new ServiceInstance(instance.instanceId(), instance.host(), instance.port(), healthy));
    }

    public List<ServiceInstance> healthyInstancesFor(String serviceName) {
        return instancesById.values().stream()
                .filter(ServiceInstance::healthy)
                .filter(instance -> belongsTo(instance, serviceName))
                .toList();
    }

    private boolean pingHealthEndpoint(ServiceInstance instance) {
        try {
            HttpResponse<Void> response = httpClient.send(
                    HttpRequest.newBuilder(URI.create("http://" + instance.host() + ":" + instance.port() + "/health")).build(),
                    HttpResponse.BodyHandlers.discarding());
            return response.statusCode() == 200;
        } catch (IOException | InterruptedException e) {
            return false; // unreachable is unhealthy -- fail closed, never assume health on a network error
        }
    }
}
```

An instance that fails its health check is never *removed* from `instancesById` — it's marked `healthy=false` and kept, so it can be automatically restored to rotation the moment a later check succeeds, without needing a separate re-registration flow. Failing **closed** on a network error (treating "I couldn't reach it" as "it's unhealthy," never as "it's probably fine") is the conservative, correct default — the alternative risks routing live traffic to an instance that's actually down.

---

# 24. Follow-up Question 7 — "Multiple Healthy Instances Exist. How Do You Pick One?"

> **Interviewer:** *"§23 gives you a list of healthy instances. You need exactly one to actually send this request to. What's the selection logic, and does it ever change?"*

It changes constantly, in practice — which is exactly why §25-§26 make instance selection a pluggable **Strategy**, not a single hard-coded algorithm.

---

# 25. Load Balancing Strategies: Round Robin, Weighted, and Least-Connections

- **Round robin**: cycle through healthy instances in order — simple, fair when every instance has equal capacity.
- **Weighted round robin**: some instances get proportionally more traffic (a bigger instance, or one still warming up after a deploy gets less).
- **Least connections**: send the next request to whichever healthy instance currently has the fewest in-flight requests — self-correcting under uneven request durations, where round robin can accidentally pile requests onto an already-slow instance.

---

# 26. Implementing a Pluggable LoadBalancer

```java
public interface LoadBalancer {
    ServiceInstance select(List<ServiceInstance> healthyInstances);
}
```

```java
public final class RoundRobinLoadBalancer implements LoadBalancer {
    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();

    @Override
    public ServiceInstance select(List<ServiceInstance> healthyInstances) {
        if (healthyInstances.isEmpty()) throw new NoHealthyInstanceException();
        AtomicInteger counter = counters.computeIfAbsent(healthyInstances.get(0).instanceId(), k -> new AtomicInteger(0));
        int index = Math.floorMod(counter.getAndIncrement(), healthyInstances.size());
        return healthyInstances.get(index);
    }
}
```

```java
public final class LeastConnectionsLoadBalancer implements LoadBalancer {
    private final Map<String, AtomicInteger> inFlightByInstanceId = new ConcurrentHashMap<>();

    @Override
    public ServiceInstance select(List<ServiceInstance> healthyInstances) {
        if (healthyInstances.isEmpty()) throw new NoHealthyInstanceException();
        return healthyInstances.stream()
                .min(Comparator.comparingInt(instance -> inFlightByInstanceId.getOrDefault(instance.instanceId(), new AtomicInteger(0)).get()))
                .orElseThrow();
    }

    public void onRequestStarted(String instanceId) { inFlightByInstanceId.computeIfAbsent(instanceId, k -> new AtomicInteger(0)).incrementAndGet(); }
    public void onRequestFinished(String instanceId) { inFlightByInstanceId.computeIfAbsent(instanceId, k -> new AtomicInteger(0)).decrementAndGet(); }
}
```

Swapping `RoundRobinLoadBalancer` for `LeastConnectionsLoadBalancer` — per route, even — is a one-line configuration change; nothing in the routing or filter-chain code needs to know which strategy is active, which is precisely the Strategy pattern doing its job (§63).

---

# 27. Follow-up Question 8 — "A Backend Instance Is Failing Repeatedly. How Do You Stop Hammering It?"

> **Interviewer:** *"An instance starts timing out on every call. Your load balancer keeps sending it traffic anyway, because health checks are still 5 seconds apart and it hasn't failed one yet. What's missing?"*

A **circuit breaker** — a mechanism that reacts to *live call failures*, not just periodic health checks, and stops sending traffic the moment a backend looks unhealthy from the gateway's own direct experience, faster than the next scheduled health check could ever catch it.

---

# 28. The Circuit Breaker: A State Machine for Failing Dependencies

```text
   CLOSED (normal operation, calls pass through)
      |
      | failure rate exceeds threshold
      v
   OPEN (calls fail IMMEDIATELY, without even attempting the backend -- protects both sides)
      |
      | after a cooldown period elapses
      v
   HALF_OPEN (a small number of trial calls are allowed through)
      |
      +-- trial calls succeed --> back to CLOSED
      |
      +-- trial calls fail --> back to OPEN (cooldown restarts)
```

This is the exact same **State pattern** discipline [the JIRA-style guide's issue-status workflow](<Design a Project Management Tool Like JIRA — HLD, LLD, and Class Design From Scratch.md>) and [the Workflow Automation guide's run/step lifecycle](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) both already apply to a different status field — here applied to a backend's *health*, as directly observed by the gateway's own calls, entirely independent of §23's periodic health checks.

---

# 29. Implementing the Circuit Breaker

```java
public final class CircuitBreaker {

    private final int failureThreshold;
    private final Duration cooldown;
    private final int halfOpenTrialCount;

    private volatile CircuitState state = CircuitState.CLOSED;
    private final AtomicInteger consecutiveFailures = new AtomicInteger(0);
    private volatile Instant openedAt;
    private final AtomicInteger halfOpenTrialsRemaining = new AtomicInteger(0);

    public CircuitBreaker(int failureThreshold, Duration cooldown, int halfOpenTrialCount) {
        this.failureThreshold = failureThreshold;
        this.cooldown = cooldown;
        this.halfOpenTrialCount = halfOpenTrialCount;
    }

    public boolean allowRequest() {
        if (state == CircuitState.OPEN) {
            if (Instant.now().isAfter(openedAt.plus(cooldown))) {
                state = CircuitState.HALF_OPEN;
                halfOpenTrialsRemaining.set(halfOpenTrialCount);
            } else {
                return false; // still cooling down -- fail fast, never even attempt the backend
            }
        }
        if (state == CircuitState.HALF_OPEN) {
            return halfOpenTrialsRemaining.getAndDecrement() > 0;
        }
        return true; // CLOSED
    }

    public void onSuccess() {
        consecutiveFailures.set(0);
        if (state == CircuitState.HALF_OPEN) state = CircuitState.CLOSED; // trial succeeded -- fully recovered
    }

    public void onFailure() {
        if (state == CircuitState.HALF_OPEN) {
            reopen(); // a single trial failure during HALF_OPEN is enough -- don't risk more real traffic on it
            return;
        }
        if (consecutiveFailures.incrementAndGet() >= failureThreshold) {
            reopen();
        }
    }

    private void reopen() {
        state = CircuitState.OPEN;
        openedAt = Instant.now();
        consecutiveFailures.set(0);
    }
}
```

Every call to a backend instance is wrapped: `if (!breaker.allowRequest()) throw new CircuitOpenException();` before the call, then `breaker.onSuccess()`/`breaker.onFailure()` after it returns or throws. Failing **immediately** while `OPEN` — never even attempting the network call — is the entire point: it protects the *already-struggling* backend from more load, and protects the gateway's own threads/connections from piling up waiting on a backend that's very unlikely to respond in time anyway.

---

# 30. Follow-up Question 9 — "How Does the Gateway Authenticate Requests at the Edge?"

> **Interviewer:** *"You don't want every backend service re-implementing authentication. How does the gateway validate a caller's identity once, centrally?"*

Two common mechanisms, both implemented as `Filter`s (§15-§16) that run **before** routing ever happens — rejecting an unauthenticated request before it costs the gateway a route lookup, a load-balancer decision, or a backend call.

---

# 31. Authentication and Authorization at the Edge: API Keys and JWT Validation

```java
public final class AuthenticationFilter implements Filter {

    private final ApiKeyValidator apiKeyValidator;
    private final JwtValidator jwtValidator;

    public AuthenticationFilter(ApiKeyValidator apiKeyValidator, JwtValidator jwtValidator) {
        this.apiKeyValidator = apiKeyValidator;
        this.jwtValidator = jwtValidator;
    }

    @Override
    public void doFilter(GatewayRequest request, GatewayResponse response, FilterChain chain) throws GatewayException {
        String apiKey = request.header("X-Api-Key");
        String bearerToken = request.bearerToken();

        AuthResult result = apiKey != null ? apiKeyValidator.validate(apiKey)
                           : bearerToken != null ? jwtValidator.validate(bearerToken)
                           : AuthResult.unauthenticated();

        if (!result.authenticated()) {
            response.reject(401, "Unauthorized");
            return; // deliberately never calls chain.doFilter -- short-circuits here, §15
        }
        request.setPrincipal(result.principal()); // later filters (§56's rate limiter) read this to resolve a key
        chain.doFilter(request, response);
    }
}
```

`request.setPrincipal(...)` is what makes rate limiting *by user* (§55), not just by raw IP, possible at all — every filter after this one in the chain can read the authenticated identity, without needing to re-validate a token or an API key itself.

---

# 32. High-Level Architecture Diagram, Assembled

```text
                                     Client
                                       |
                            Gateway Instance (any one of many)
                                       |
                    AuthenticationFilter (§31) --(fail)--> 401
                                       |
                    RateLimitFilter (§56) --(fail)--> 429  [reads the SHARED store, §48]
                                       |
                    LoggingFilter (correlation ID attached)
                                       |
                    RouteRegistry.match() (§19) --(no match)--> 404
                                       |
                    HealthChecker.healthyInstancesFor(...) (§23)
                                       |
                    LoadBalancer.select(...) (§26)
                                       |
                    CircuitBreaker.allowRequest() (§29) --(open)--> 503, fail fast
                                       |
                              Backend Service Instance
                                       |
                    (response flows back up through Logging + any response-transforming filters)
                                       |
                                    Client
```

---

# 33. Class Diagram: The Gateway Core

```text
Filter <<interface>>                          FilterChain
+ doFilter(request, response, chain)          + doFilter(request, response)
      ^                                              |
      | implements                                   | terminates into
  +---+------------+------------+                    v
Auth   RateLimit   Logging      BackendInvoker
Filter   Filter     Filter      + invoke(request, response)
                                        |
                    +-------------------+-------------------+
                    v                   v                   v
              RouteRegistry      LoadBalancer          CircuitBreaker
              + match(request)   <<interface>>          + allowRequest()
                    |             + select(instances)    + onSuccess()/onFailure()
                    v                   ^                       |
                 Route          +-------+-------+               v
                              RoundRobin   LeastConnections  CircuitState
                                                              (CLOSED/OPEN/HALF_OPEN)
```

Everything above this line is the **gateway core** — extensible, pluggable, and (deliberately) entirely per-instance-safe, meaning no two gateway instances need to coordinate with each other to run any of it correctly. Everything from §34 onward is the one component where that stops being true.

---

# 34. Follow-up Question 10 — "Now the Hard Part. Design the Rate Limiter."

> **Interviewer:** *"Everything so far has been solid, standard gateway design. Now I want real depth: design the rate limiter this gateway uses. Start from the requirements, and don't skip the part where it has to work correctly across every gateway instance at once."*

This is the pivot point the entire rest of this guide exists for. §35-§36 restate requirements specifically for the rate limiter; §37 onward builds five algorithms, in increasing sophistication, ending at the one that's actually correct under concurrency and at scale.

---

# 35. Why Rate Limit At All: Protecting Backends and Enforcing Fairness

Three genuinely distinct reasons, worth naming separately because they sometimes call for different limits: **protecting backend capacity** (a backend service can only handle so much load before its own latency degrades for everyone, regardless of who's calling); **enforcing fair usage and tiered plans** (a free-tier caller should not be able to consume the capacity a paying enterprise tier is guaranteed); and **abuse and denial-of-service resistance** (a misbehaving or malicious client hammering an endpoint should be capped, cheaply, at the edge, before it ever costs a backend service any real work).

---

# 36. Rate Limiting Requirements: What "Correct" Actually Means Here

- A client that has exceeded its limit is rejected with `429 Too Many Requests` (§57), not silently dropped or slowed.
- The limit is enforced **cluster-wide** — a client's true request rate across *all* gateway instances combined, never per-instance (§46-§48).
- Short bursts up to a configured capacity are allowed; the limit is on **sustained** rate, not a rigid "exactly N per second, no more, ever" (§43-§44's token bucket).
- The limiter must remain **correct under concurrency** — two gateway instances checking the same client's limit at the same instant must never both succeed when only one request's worth of quota remains (§49-§50).
- Checking a rate limit must add **minimal latency** to every single request — this rules out anything requiring multiple sequential round trips to a shared store per check (§51's single-atomic-operation requirement, §53's hybrid optimization).

---

# 37. Follow-up Question 11 — "Start Simple. What's the Naive Approach, and What's Wrong With It?"

> **Interviewer:** *"Forget distribution for a second — one machine, one client, one limit. What's the simplest thing that could possibly work, and where does it break?"*

§38 builds it — a fixed window counter — and shows precisely where "simplest" stops being "correct enough."

---

# 38. The Fixed Window Counter: Simple, and Its Boundary-Burst Flaw

Every algorithm from here through §53 implements one shared interface, so `RateLimitFilter` (§56) never needs to know or care which specific algorithm is actually configured:

```java
public interface RateLimitAlgorithm {
    boolean allow(String key); // true = allowed, false = reject with 429 (§57)
}
```

```java
public final class FixedWindowCounter implements RateLimitAlgorithm {

    private final int limit;
    private final Duration windowSize;
    private final Map<String, WindowState> stateByKey = new ConcurrentHashMap<>();

    private record WindowState(long windowStartEpochMillis, AtomicInteger count) { }

    public FixedWindowCounter(int limit, Duration windowSize) {
        this.limit = limit;
        this.windowSize = windowSize;
    }

    @Override
    public boolean allow(String key) {
        long now = System.currentTimeMillis();
        long windowStart = (now / windowSize.toMillis()) * windowSize.toMillis();

        WindowState state = stateByKey.compute(key, (k, existing) -> {
            if (existing == null || existing.windowStartEpochMillis() != windowStart) {
                return new WindowState(windowStart, new AtomicInteger(0)); // a NEW window -- resets the count to zero
            }
            return existing;
        });
        return state.count().incrementAndGet() <= limit;
    }
}
```

Simple, and genuinely wrong at the boundary: a limit of "100 requests per minute" allows 100 requests in the last millisecond of one window **and** another 100 in the first millisecond of the next — 200 requests in a two-millisecond span, twice the intended sustained rate, purely because the window edges reset the count to zero with no memory of what just happened in the *previous* window. This is not a rare edge case; a client with any reason to burst near a window boundary hits it reliably.

---

# 39. Follow-up Question 12 — "How Do You Fix the Boundary-Burst Problem?"

> **Interviewer:** *"You just showed me a real gap. Fix it — and show me the tradeoff, because I doubt the fix is free."*

Two standard fixes, with a genuine accuracy-vs-memory tradeoff between them: §40's sliding window log (perfectly accurate, memory scales with request volume) and §41's sliding window counter (an approximation, memory stays constant).

---

# 40. The Sliding Window Log: Accurate, But Memory-Heavy

```java
public final class SlidingWindowLog implements RateLimitAlgorithm {

    private final int limit;
    private final Duration windowSize;
    private final Map<String, Deque<Long>> timestampsByKey = new ConcurrentHashMap<>();

    public SlidingWindowLog(int limit, Duration windowSize) {
        this.limit = limit;
        this.windowSize = windowSize;
    }

    @Override
    public synchronized boolean allow(String key) { // synchronized -- the deque mutation below must be atomic per key
        long now = System.currentTimeMillis();
        long windowStartCutoff = now - windowSize.toMillis();

        Deque<Long> timestamps = timestampsByKey.computeIfAbsent(key, k -> new ArrayDeque<>());
        while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStartCutoff) {
            timestamps.pollFirst(); // evict every timestamp that has aged out of the CURRENT sliding window
        }
        if (timestamps.size() >= limit) return false;
        timestamps.addLast(now);
        return true;
    }
}
```

Every single request's exact timestamp is stored, and the window slides continuously — evaluated at the *instant* of each check, not snapped to a fixed boundary — which makes this perfectly accurate: there is no boundary anywhere for a client to exploit. The cost is exactly what it looks like: memory proportional to the limit itself, per key, forever (a limit of 10,000 requests/minute means up to 10,000 stored timestamps per client, at all times) — fine for a modest limit, genuinely expensive at a high one across millions of distinct keys.

---

# 41. The Sliding Window Counter: An Accuracy/Memory Compromise

```java
public final class SlidingWindowCounter implements RateLimitAlgorithm {

    private final int limit;
    private final Duration windowSize;
    private final Map<String, long[]> windowsByKey = new ConcurrentHashMap<>(); // [windowStart, previousCount, currentCount]

    public SlidingWindowCounter(int limit, Duration windowSize) {
        this.limit = limit;
        this.windowSize = windowSize;
    }

    @Override
    public synchronized boolean allow(String key) {
        long windowMillis = windowSize.toMillis();
        long now = System.currentTimeMillis();
        long currentWindowStart = (now / windowMillis) * windowMillis;

        long[] state = windowsByKey.computeIfAbsent(key, k -> new long[]{currentWindowStart, 0, 0});
        if (state[0] != currentWindowStart) {
            boolean adjacentWindow = (currentWindowStart - state[0]) == windowMillis;
            state[0] = currentWindowStart;
            state[1] = adjacentWindow ? state[2] : 0; // only carry the count forward from the IMMEDIATELY preceding window
            state[2] = 0;
        }

        double elapsedFractionOfWindow = (now - currentWindowStart) / (double) windowMillis;
        double weightedCount = state[1] * (1 - elapsedFractionOfWindow) + state[2];

        if (weightedCount >= limit) return false;
        state[2]++;
        return true;
    }
}
```

The insight: instead of storing every timestamp (§40), keep just **two fixed-window counts** — the previous window's total and the current window's running total — and estimate the true sliding-window count as a weighted blend of the two, based on how far into the current window "now" is. This is an **approximation**, not perfectly exact (it assumes requests were spread evenly across the previous window, which is usually close enough in practice), but its memory footprint per key is now a small, constant handful of numbers regardless of the limit's size — the standard, industry-common compromise between §38's flawed simplicity and §40's exact-but-expensive accuracy.

---

# 42. Follow-up Question 13 — "What About Allowing Controlled Bursts, Not Just a Flat Rate?"

> **Interviewer:** *"Every algorithm so far treats the limit as a hard, flat ceiling. What if I want to allow a short burst above the sustained rate — say, a client that's usually well under its limit should be allowed to briefly spike, as long as it settles back down?"*

Neither §38, §40, nor §41 was designed for this — all three enforce a ceiling *within* a window, with no concept of "saved-up" capacity from a quiet period. §43-§44 build the algorithm actually designed for exactly this case: the **token bucket**.

---

# 43. The Token Bucket Algorithm

A bucket holds up to `capacity` tokens, refilling at a steady `refillRate` (tokens per second); every request consumes one token, and is allowed only if a token is available. A client that's been quiet accumulates tokens up to the bucket's capacity, and can then spend them all in a genuine burst — exactly the "smooth sustained rate, but tolerate a burst" behavior §42 asked for. This is a genuinely different use of the same underlying bucket-and-refill mechanism the [Workflow Automation guide's retry policy](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) uses exponential backoff for: that guide smooths *outgoing* retries against a flaky dependency, while this section caps *incoming* request rate — the same "control a rate against a budget that refills over time" idea, applied in opposite directions.

---

# 44. Implementing a Real Token Bucket Rate Limiter

```java
public final class TokenBucket implements RateLimitAlgorithm {

    private final long capacity;
    private final double refillTokensPerMillisecond;
    private final Map<String, BucketState> stateByKey = new ConcurrentHashMap<>();

    private static final class BucketState {
        double tokens;
        long lastRefillTimestampMillis;
        BucketState(double tokens, long lastRefillTimestampMillis) {
            this.tokens = tokens;
            this.lastRefillTimestampMillis = lastRefillTimestampMillis;
        }
    }

    public TokenBucket(long capacity, double refillTokensPerSecond) {
        this.capacity = capacity;
        this.refillTokensPerMillisecond = refillTokensPerSecond / 1000.0;
    }

    @Override
    public boolean allow(String key) {
        BucketState state = stateByKey.computeIfAbsent(key, k -> new BucketState(capacity, System.currentTimeMillis()));
        synchronized (state) { // per-KEY lock, not global -- one noisy key never blocks checks for every other key
            long now = System.currentTimeMillis();
            long elapsedMillis = now - state.lastRefillTimestampMillis;
            state.tokens = Math.min(capacity, state.tokens + elapsedMillis * refillTokensPerMillisecond);
            state.lastRefillTimestampMillis = now;

            if (state.tokens >= 1.0) {
                state.tokens -= 1.0;
                return true;
            }
            return false;
        }
    }
}
```

Refilling **lazily**, computed from elapsed time at the moment of each check, rather than running a background timer per key that ticks tokens in continuously, is the standard, efficient implementation — it means a key nobody has called in an hour costs zero background work, and still refills to full capacity correctly (capped at `capacity`) the instant it's checked again. Locking **per key** (`synchronized (state)`), never with one global lock across every key, is exactly the same "isolate contention to the smallest necessary scope" discipline the [ConcurrentHashMap guide's bucket-level locking](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) builds from scratch for a general-purpose map.

---

# 45. The Leaky Bucket Algorithm, and How It Differs From Token Bucket

A **leaky bucket** models the same "bucket" intuition inverted: incoming requests fill the bucket (up to its capacity, beyond which new requests are rejected), and the bucket **leaks** — processes queued requests — at a constant, fixed rate, regardless of how full it is. The practical difference from a token bucket matters: a token bucket allows a burst to pass through the gateway **immediately**, back-to-back, as long as tokens are available; a leaky bucket **smooths** a burst out, queuing excess requests and releasing them at the steady leak rate rather than letting them all through at once. A token bucket is the right choice when the goal is "cap the sustained rate but don't punish a legitimate short burst" (§42's requirement); a leaky bucket is the right choice when the goal is "guarantee a perfectly smooth, constant outgoing rate regardless of how bursty the incoming traffic is" — for MiniGate's stated requirements (§36), the token bucket is the better fit, and is what §51 onward builds out to a full distributed implementation.

---

# 46. Follow-up Question 14 — "This All Assumed One Machine. You Have 50 Gateway Instances Behind a Load Balancer. Now What?"

> **Interviewer:** *"Every algorithm you've shown me lives in one process's memory. A client's requests get spread across 50 different gateway instances by your own load balancer. What happens to their rate limit?"*

This is the question every prior section was building toward, and it's where a design that looked complete quietly falls apart. §47 states exactly why; §48 fixes it.

---

# 47. Why a Local, In-Memory Counter Breaks Under Multiple Gateway Instances

Every algorithm built in §38-§45 stores its state in a `Map` living inside **one JVM process**. With 50 gateway instances, a client whose requests happen to be spread evenly across all 50 gets a *true* effective limit 50 times higher than configured — each instance independently thinks the client has used 0 out of, say, 100 requests, allows all 100, and has no idea the other 49 instances just did the exact same thing. The limit was never actually violated **from any one instance's point of view** — it was violated in aggregate, which is the only view that was ever supposed to matter. A rate limiter whose correctness depends on which specific instance happens to receive a given request is not a correct rate limiter at all.

---

# 48. Distributed Rate Limiting With a Shared Store

The fix is unsurprising once §47 is stated plainly: **the counter cannot live in any one gateway instance's memory — it must live in one shared store every instance reads and writes.** A Redis-style in-memory key-value store, reachable by every gateway instance over the network, replaces the `Map<String, ...>` at the center of every algorithm in §38-§45 — the *algorithm* (fixed window, sliding window, token bucket) stays conceptually identical; only *where its state lives* changes. This alone is not sufficient, though — §49 shows exactly what still goes wrong even with a shared store, if the store is used carelessly.

---

# 49. Follow-up Question 15 — "Two Gateway Instances Check-Then-Increment at the Same Time. What Goes Wrong?"

> **Interviewer:** *"You've moved the counter to Redis. Instance A reads the count, sees 99 out of 100 remaining, and prepares to increment. At the exact same moment, Instance B does the same read, also sees 99. What happens next?"*

§50 names this precisely: it's a classic check-then-act race condition, and it lets exactly one extra request through every time it happens — which, at real traffic volume, is not a rare accident, it's routine.

---

# 50. The Race Condition, and Why Atomicity Is Non-Negotiable

```text
Instance A: GET count for key "user:42"     -> 99
Instance B: GET count for key "user:42"     -> 99   (before A's increment lands)
Instance A: count < limit (100) -> ALLOW, then SET count = 100
Instance B: count < limit (100) -> ALLOW, then SET count = 100   (should have been REJECTED -- limit already reached)
```

Two separate network round trips — a `GET` to check, then a `SET`/`INCR` to update — is exactly the same "check-then-act must be one atomic step, not two" hazard the [ConcurrentHashMap guide](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) and the [Optimistic vs Pessimistic Locking guide](<Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation.md>) both build entire mechanisms to solve for a single in-process map or a single database row — here the same hazard shows up across a network, between two entirely separate processes, which makes an in-process lock useless (Instance A's lock protects nothing Instance B can see). The fix has to be atomicity **inside the shared store itself**, not in either gateway instance.

---

# 51. An Atomic Token Bucket Using a Redis-Style Lua Script

Redis executes a submitted Lua script as a single, indivisible operation — no other client's command can interleave in the middle of it, even though many different gateway instances are calling it concurrently. This turns "read the bucket state, compute, write it back" from three separate round trips (each individually race-able, as §50 showed) into **one** atomic round trip:

```lua
-- KEYS[1] = the bucket's key, e.g. "ratelimit:user:42"
-- ARGV[1] = capacity, ARGV[2] = refillTokensPerMillisecond, ARGV[3] = now (epoch millis)

local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call("HMGET", KEYS[1], "tokens", "lastRefill")
local tokens = tonumber(bucket[1])
local lastRefill = tonumber(bucket[2])

if tokens == nil then
    tokens = capacity
    lastRefill = now
end

local elapsed = now - lastRefill
tokens = math.min(capacity, tokens + elapsed * refillRate)

local allowed = 0
if tokens >= 1 then
    tokens = tokens - 1
    allowed = 1
end

redis.call("HMSET", KEYS[1], "tokens", tokens, "lastRefill", now)
redis.call("EXPIRE", KEYS[1], 3600)  -- bound memory: a key nobody touches for an hour is reclaimed automatically

return allowed
```

```java
public final class DistributedTokenBucket implements RateLimitAlgorithm {

    private final RedisScriptClient redis; // EVAL-capable client
    private final String scriptSha;         // the §51 Lua script above, loaded once at startup via SCRIPT LOAD
    private final long capacity;
    private final double refillTokensPerMillisecond;

    public DistributedTokenBucket(RedisScriptClient redis, String scriptSha, long capacity, double refillTokensPerSecond) {
        this.redis = redis;
        this.scriptSha = scriptSha;
        this.capacity = capacity;
        this.refillTokensPerMillisecond = refillTokensPerSecond / 1000.0;
    }

    @Override
    public boolean allow(String key) {
        long allowed = redis.evalSha(scriptSha,
                List.of("ratelimit:" + key),
                List.of(String.valueOf(capacity), String.valueOf(refillTokensPerMillisecond), String.valueOf(System.currentTimeMillis())));
        return allowed == 1L;
    }
}
```

Every gateway instance calls the exact same script, against the exact same key, and Redis's single-threaded command execution guarantees that two concurrent `EVALSHA` calls for the same key are processed **one at a time**, never interleaved — the read-compute-write sequence inside the script is exactly as atomic as a single `INCR` would be, just with real logic (refill math, a capacity cap) attached to it. This is the one design decision that actually makes distributed rate limiting *correct*, not merely "probably fine most of the time" — everything in §38-§45 was correct algorithm design; §51 is what makes it correct **across machines**.

---

# 52. Follow-up Question 16 — "A Round Trip to the Shared Store on Every Single Request Adds Latency. How Do You Reduce That?"

> **Interviewer:** *"Every request now pays a network round trip to Redis before it can even be routed. At your target scale, that's real, measurable added latency on the hot path. What do you do about it?"*

§53 answers with a hybrid: keep §48's shared store as the source of truth, but stop consulting it on **every single** request.

---

# 53. Hybrid Rate Limiting: Local Approximation Plus Periodic Sync

```java
public final class HybridRateLimiter implements RateLimitAlgorithm {

    private final DistributedTokenBucket sharedTruth;      // §51 -- consulted periodically, not per-request
    private final Map<String, LocalAllowance> localState = new ConcurrentHashMap<>();
    private final int localBudgetPerSyncInterval;           // this instance's own slice of the global capacity

    private record LocalAllowance(AtomicInteger remaining, volatile long fetchedAtMillis) { }

    public HybridRateLimiter(DistributedTokenBucket sharedTruth, int localBudgetPerSyncInterval) {
        this.sharedTruth = sharedTruth;
        this.localBudgetPerSyncInterval = localBudgetPerSyncInterval;
    }

    @Override
    public boolean allow(String key) {
        LocalAllowance allowance = localState.computeIfAbsent(key, k -> fetchNewAllowance());
        if (allowance.remaining().get() > 0 && !expired(allowance)) {
            return allowance.remaining().getAndDecrement() > 0;
        }
        // Local budget exhausted or stale -- fall back to a real, atomic check against the shared truth (§51).
        boolean allowedByShared = sharedTruth.allow(key);
        localState.put(key, fetchNewAllowance()); // refresh -- give this instance a fresh local slice either way
        return allowedByShared;
    }

    private LocalAllowance fetchNewAllowance() {
        return new LocalAllowance(new AtomicInteger(localBudgetPerSyncInterval), System.currentTimeMillis());
    }

    private boolean expired(LocalAllowance allowance) {
        return System.currentTimeMillis() - allowance.fetchedAtMillis() > 1000; // re-sync at least once a second
    }
}
```

The tradeoff, stated honestly rather than hidden: this is **not** perfectly precise the way §51 alone is — an instance can allow slightly more than its exact fair share within one sync interval before it re-checks, meaning the true cluster-wide limit can be modestly over-shot for a brief window. In exchange, the overwhelming majority of requests are decided **entirely locally, in memory, with zero network round trip** — only the (comparatively rare) local-budget-exhausted case pays §51's network cost. This is the exact same "accept a small, bounded, honestly-stated inaccuracy in exchange for dramatically better latency" tradeoff §41's sliding window counter already made for a single-node algorithm, now applied to the distributed case.

---

# 54. Follow-up Question 17 — "How Do You Rate-Limit by API Key, by User, AND by Route, All at Once?"

> **Interviewer:** *"Real limits are layered — a per-route cap so no single endpoint gets hammered, a per-user cap so no single caller monopolizes the platform, maybe a per-API-key tier cap on top of that. How does your design support all three simultaneously, cleanly?"*

None of §38-§53's algorithm implementations care what a "key" *means* — every one of them takes an opaque `String key` and tracks a bucket/window/counter against it. §55 exploits that directly: layered limits are just **multiple calls** to the same algorithm, with **different key strings**, not a different algorithm.

---

# 55. Rate Limit Key Design and Layered/Composite Limits

```java
public final class RateLimitKeyResolver {

    public List<String> keysFor(GatewayRequest request) {
        List<String> keys = new ArrayList<>();
        keys.add("route:" + request.matchedRoute().routeId());                       // protects one endpoint from overload
        if (request.principal() != null) {
            keys.add("user:" + request.principal().userId());                          // protects fairness across users
            keys.add("apikey-tier:" + request.principal().apiKeyTier());               // enforces the caller's plan
        } else {
            keys.add("ip:" + request.remoteIp());                                      // the only identity an unauthenticated caller has
        }
        return keys;
    }
}
```

```java
public boolean allowAllLayers(GatewayRequest request, RateLimitAlgorithm limiter, RateLimitKeyResolver keyResolver) {
    for (String key : keyResolver.keysFor(request)) {
        if (!limiter.allow(key)) return false; // ANY layer rejecting rejects the whole request -- the strictest limit wins
    }
    return true;
}
```

Each layer typically has its **own** configured capacity and refill rate (a route-level limit is usually much higher than any single user's share of it) — in practice this means each key string above maps to a *differently-configured* `RateLimitAlgorithm` instance, not one shared configuration reused across layers, even though they're all the same algorithm implementation.

---

# 56. The Rate Limiter as a Gateway Filter: Wiring It All Together

```java
public final class RateLimitFilter implements Filter {

    private final RateLimitKeyResolver keyResolver; // §55
    private final RateLimitAlgorithm limiter;         // §53's HybridRateLimiter, in production

    public RateLimitFilter(RateLimitKeyResolver keyResolver, RateLimitAlgorithm limiter) {
        this.keyResolver = keyResolver;
        this.limiter = limiter;
    }

    @Override
    public void doFilter(GatewayRequest request, GatewayResponse response, FilterChain chain) throws GatewayException {
        for (String key : keyResolver.keysFor(request)) {
            if (!limiter.allow(key)) {
                response.reject(429, "Too Many Requests"); // §57 -- with the standard headers attached
                return; // short-circuits, §15 -- never even reaches routing
            }
        }
        chain.doFilter(request, response);
    }
}
```

Placing `RateLimitFilter` **after** `AuthenticationFilter` (§31) in the chain (§15, §32's diagram) is deliberate: `request.principal()` must already be populated for §55's `user:`/`apikey-tier:` keys to resolve to anything meaningful — an unauthenticated request only ever gets the `ip:` fallback key, which is exactly the right, conservative behavior for a caller the gateway can't otherwise identify.

---

# 57. Rate Limit Response Contract: 429, Retry-After, and the X-RateLimit-* Headers

A rejected request is never just a bare `429` — a well-designed rate limiter tells the caller enough to behave correctly on its own:

```text
HTTP/1.1 429 Too Many Requests
Retry-After: 4
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735689600
```

`Retry-After` (seconds until it's worth trying again) lets a well-behaved client back off correctly without guessing or polling aggressively; `X-RateLimit-Limit`/`Remaining`/`Reset` let a client proactively throttle *itself* before ever getting rejected, by watching how much headroom it has left. These headers cost nothing extra to compute — §44's `TokenBucket`/§51's Lua script already know exactly how many tokens remain and when the bucket will next have capacity; §56's filter just needs to surface those numbers on every response, allowed or rejected, not only on a rejection.

---

# 58. Follow-up Question 18 — "At Millions of Requests Per Second Across Thousands of Tenants, What Breaks First?"

> **Interviewer:** *"Single Redis instance backing every gateway instance's rate limiter. What's the first thing that falls over as traffic grows, and what's the standard fix?"*

The shared store itself, precisely because §48 deliberately centralized everything onto it — §59 shards it the same way [the Workflow Automation guide's execution-engine partitioning](<Design a Workflow Automation System — HLD, LLD, and Class Design From Scratch.md>) and [the File Storage guide's consistent-hashing object stores](<Design Your Own File Storage System — Block, File, Object Storage, and RAID From Scratch.md>) both shard their own respective bottlenecks.

---

# 59. Sharding the Rate Limiter's Shared Store

```java
public final class ShardedRedisClient {

    private final List<RedisScriptClient> shards;

    public ShardedRedisClient(List<RedisScriptClient> shards) { this.shards = shards; }

    public RedisScriptClient shardFor(String rateLimitKey) {
        int shardIndex = Math.floorMod(rateLimitKey.hashCode(), shards.size()); // consistent hashing across shards
        return shards.get(shardIndex);
    }
}
```

Every rate-limit key is independent of every other one — nothing about §51's Lua script, or §55's layered keys, ever needs to read or write *two different* keys atomically together — which is exactly what makes this sharding trivially safe: a given key always hashes to the same shard, so every check for that key is still atomic (still one Redis instance, still one Lua script execution), and overall throughput now scales by adding shards, the same horizontal-scaling story every consistently-hashed system in this guide series tells.

---

# 60. Class Diagram: The Rate Limiter

```text
RateLimitAlgorithm <<interface>>                    RateLimitKeyResolver
+ allow(key): boolean                                + keysFor(request): List<String>
      ^                                                        |
      | implements                                             | resolves, per request
  +---+--------+--------+--------+--------+                    v
Fixed    Sliding   Sliding   Token    Leaky        RateLimitFilter  <<Filter>>
Window   Window    Window    Bucket   Bucket       + doFilter(request, response, chain)
Counter  Log       Counter   (§44)    (§45)              |  calls .allow(key) for EVERY resolved key (§55)
(§38)    (§40)     (§41)                                 |  ANY rejection -> 429 (§57)
                                                           v
                                                    HybridRateLimiter  (implements RateLimitAlgorithm too)
                                                    + allow(key): boolean
                                                           |
                                          +----------------+----------------+
                                          v                                 v
                                 LocalAllowance                    DistributedTokenBucket
                                 (per-instance, in-memory,          + allow(key): boolean
                                  §53 -- no network round trip)            |
                                                                           v
                                                                 ShardedRedisClient (§59)
                                                                 + shardFor(key): RedisScriptClient
                                                                           |
                                                                           v
                                                              Redis Shard 0..N, each running
                                                              the SAME atomic Lua script (§51)
```

Two things this diagram makes visible that prose alone can blur: first, `HybridRateLimiter` **itself** implements `RateLimitAlgorithm` (§53) — it is not a special case `RateLimitFilter` treats differently, it's simply the algorithm configured in production, indistinguishable from the call site's point of view from swapping in a bare `TokenBucket` for a test. Second, every box below `HybridRateLimiter` exists purely to make the *single* interface above it fast in the common case and correct in every case — `RateLimitFilter` (§56), and every filter around it, never needs to know a shared store, a shard, or a Lua script exists at all.

---

# 61. Full Worked Example: One Request, Rate-Limited, End to End

`GET /orders/42`, with a valid JWT, arriving at Gateway Instance 7 of 50, against a Redis cluster of 4 shards.

```text
1.  AuthenticationFilter (§31): JwtValidator validates the bearer token -> principal set: userId=u_88, apiKeyTier=pro
2.  RateLimitKeyResolver.keysFor() (§55) -> ["route:orders-get", "user:u_88", "apikey-tier:pro"]
3.  RateLimitFilter (§56) checks each key against the HybridRateLimiter (§53):
      "route:orders-get"   -> local budget has capacity -- ALLOWED, zero network round trip
      "user:u_88"          -> local budget exhausted for this instance's slice -- falls back to §51
                              ShardedRedisClient.shardFor("user:u_88") -> shard 2
                              EVALSHA against shard 2 -- atomic read-refill-check-decrement -- ALLOWED
      "apikey-tier:pro"    -> local budget has capacity -- ALLOWED
4.  Response headers attached (§57): X-RateLimit-Remaining reflects the "user:u_88" bucket's post-decrement count
5.  LoggingFilter attaches a correlation ID, logs the request
6.  RouteRegistry.match() (§19) -> Route(routeId="orders-get", serviceName="orders-service")
7.  HealthChecker.healthyInstancesFor("orders-service") (§23) -> 11 of 12 instances healthy
8.  LoadBalancer.select(...) (§26) -> instance "orders-7"
9.  CircuitBreaker for "orders-7" (§29): state=CLOSED -> allowRequest() returns true
10. Backend call succeeds -> CircuitBreaker.onSuccess(); response flows back up through the chain to the client
```

Every numbered line traces to a section this guide built real code for — including step 3's mixed outcome (two layers decided locally, one fell through to the atomic distributed check), which is exactly the everyday case §53's hybrid design was built for, not a rare exception.

---

# 62. Final Architecture Diagram

```text
                                          Client
                                             |
                              Load Balancer (in front of the gateway cluster)
                                             |
                    +------------------------+------------------------+
                    v                        v                        v
             Gateway Instance          Gateway Instance          Gateway Instance
             AuthN (§31) -> RateLimit (§56, hybrid §53) -> Logging -> Route (§19)
                    |                        |                        |
                    +------------------------+------------------------+
                                             |
                    LoadBalancer (§26) + CircuitBreaker (§29) per backend
                                             |
                    +------------------------+------------------------+
                    v                        v                        v
               Orders Service          Payments Service          Users Service
              (health-checked, §23)   (health-checked, §23)    (health-checked, §23)

                              Sharded Rate-Limit Store (§59)
                    Shard 0        Shard 1        Shard 2        Shard 3
              each an atomic, Lua-scripted token bucket (§51), consulted only on a
              local-budget miss (§53) -- the source of truth every instance defers to
```

---

# 63. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Chain of Responsibility** | `Filter`/`FilterChain` (§15-§16) | Every cross-cutting concern is an independent link; adding one never touches another. |
| **Strategy** | `LoadBalancer` (§26), `RateLimitAlgorithm` (§38-§53) | Both the instance-selection policy and the rate-limiting algorithm are swappable behind one interface each. |
| **State** | `CircuitBreaker`/`CircuitState` (§28-§29) | CLOSED/OPEN/HALF_OPEN transitions live in one place, never a scattered `if/else` on a raw flag. |
| **Facade** | The gateway's own request-handling entry point | Auth, rate limiting, routing, load balancing, and circuit breaking are all invisible to the client behind one HTTP endpoint. |
| **Proxy** | The gateway itself, relative to every backend service | Clients never talk to a backend directly; every call is transparently intercepted and mediated. |
| **Observer (conceptually)** | `HealthChecker` updating instance health (§23) | Load balancing and circuit breaking both react to health state changing, without polling for it themselves. |

---

# 64. SOLID Principles Applied

- **Single Responsibility**: `RouteRegistry` only matches routes; `HealthChecker` only tracks liveness; `RateLimitAlgorithm` implementations only decide allow/reject. None of them know how to route a request end to end.
- **Open/Closed**: adding a new `Filter` (§16), a new `LoadBalancer` strategy (§26), or a new `RateLimitAlgorithm` (§38-§53) never requires modifying `FilterChain`, the backend invoker, or the rate-limit filter — all three are consumed purely through their interfaces.
- **Liskov Substitution**: any `RateLimitAlgorithm` — a `FixedWindowCounter`, a `TokenBucket`, a `HybridRateLimiter` — is fully substitutable wherever the interface type is used (§56's `RateLimitFilter` never changes based on which one is configured).
- **Interface Segregation**: `Filter` exposes exactly one method; `LoadBalancer` exposes exactly one method (`select`) — neither interface forces an implementation to support behavior it doesn't need.
- **Dependency Inversion**: `RateLimitFilter` (§56) depends on `RateLimitAlgorithm` and `RateLimitKeyResolver` — both interfaces, injected through its constructor — never on `DistributedTokenBucket` or `HybridRateLimiter` directly.

---

# 65. Common Mistakes When Building This Yourself

- **Rate limiting per gateway instance, in-memory, and calling it done** (§47) — the single most common mistake in this entire domain; it works perfectly in a single-instance test and silently multiplies the true limit by the instance count in production.
- **Using separate `GET`-then-`SET` calls against the shared store instead of one atomic script** (§49-§50) — looks correct under light load, and lets extra requests through under real concurrent traffic, exactly when the limit matters most.
- **A fixed window counter presented as "good enough" without naming its boundary-burst flaw** (§38) — it may genuinely be good enough for a given use case, but that has to be a stated, deliberate tradeoff, not an unnoticed gap.
- **A single global lock across every rate-limit key** (§44) — serializes checks for every client through one lock, when a per-key lock (or, better, a lock-free per-key atomic structure) costs nothing extra and removes the contention entirely.
- **No `Retry-After`/`X-RateLimit-*` headers on a rejection** (§57) — technically correct, needlessly unhelpful; a well-behaved client is left guessing when it's safe to retry.
- **Failing open when the shared rate-limit store is unreachable** — silently disabling rate limiting during exactly the kind of infrastructure incident (Redis under load, a network partition) when abusive or runaway traffic is most likely to be happening; failing closed (reject, or fall back to a conservative local-only limit) is the safer default, and which one to choose is worth stating explicitly as a decision, not defaulting to accidentally.

---

# 66. Testing Strategy

- **`FixedWindowCounter`/`SlidingWindowLog`/`SlidingWindowCounter`/`TokenBucket`** (§38-§44): a burst test confirms each algorithm's actual boundary behavior matches its documented tradeoff (the fixed window's boundary flaw should be *reproducible* in a test, not just described in prose); a sustained-rate test over many windows confirms the long-run average matches the configured limit.
- **`DistributedTokenBucket`** (§51): a concurrency test firing many simultaneous `allow()` calls for the *same* key from multiple threads (simulating multiple gateway instances) confirms the total allowed count never exceeds the configured capacity plus refill — this is the test that would have caught §50's race condition had a naive `GET`-then-`SET` implementation been used instead.
- **`HybridRateLimiter`** (§53): confirms the bounded-overshoot property is genuinely bounded (never exceeds the documented worst case), and that a local-budget-exhausted key correctly falls through to the shared store rather than silently allowing or rejecting.
- **`CircuitBreaker`** (§29): a sequence of failures reaching the threshold transitions to `OPEN`; `allowRequest()` returns `false` throughout the cooldown; after the cooldown, exactly `halfOpenTrialCount` calls are allowed through in `HALF_OPEN`, and either outcome (success or failure) transitions correctly.
- **`RouteRegistry`** (§19): a config reload mid-traffic never causes an in-flight request to see a partially-applied route table (§19's atomic swap).

---

# 67. Suggested Future Enhancements

- **Adaptive rate limits** that tighten automatically when a backend's circuit breaker (§29) trips — the gateway's own resilience signal feeding directly back into how aggressively it protects that backend, rather than the two mechanisms operating in complete isolation.
- **A request-cost model**, where different endpoints consume a different number of tokens per call (a heavy search query costs more than a cheap health check), rather than every request costing exactly one token uniformly.
- **GeoDNS-aware routing**, sending a client to the nearest regional gateway cluster, each with its own regional rate-limit shard set (§59), reducing both latency and the blast radius of any one region's traffic spike.
- **A canary/traffic-shifting load balancer strategy** (§26), gradually shifting a small, increasing percentage of traffic to a newly-deployed backend version before fully cutting over.
- **Per-tenant dashboards** surfacing real-time rate-limit consumption, using the exact same `X-RateLimit-*` values (§57) already computed for every request, aggregated rather than recomputed.

---

# 68. Progressive Interview Question Set

1. Why does the Chain of Responsibility pattern make adding a new cross-cutting concern safe, concretely — what specifically would have to change in `FilterChain` if it didn't use this pattern?
2. Walk through exactly why a fixed window counter allows more than the configured limit near a window boundary, with concrete numbers.
3. What's the actual memory/accuracy tradeoff between a sliding window log and a sliding window counter, and when would each be the right choice?
4. Why does a token bucket allow bursts while a leaky bucket smooths them out — what's the mechanical difference in how each one processes a batch of requests that all arrive at once?
5. A single gateway instance's in-memory rate limiter is 100% correct by itself. Why is a cluster of them running the identical code still wrong?
6. Walk through the exact race condition two gateway instances hit against a naive `GET`-then-`SET` distributed counter, and explain precisely why a Lua script fixes it.
7. Why is a per-key lock the right granularity for a token bucket implementation, rather than one global lock?
8. What's the actual cost the hybrid local/distributed rate limiter pays in exchange for lower latency, and is that cost acceptable for a payments API versus an analytics-tracking endpoint?
9. Why must a route-config reload swap the entire route table atomically, rather than patching it incrementally in place?
10. If asked to add a "burst allowance that resets daily, separate from the per-minute sustained limit," how would you implement it using pieces this guide already built, without inventing a sixth algorithm?

---

# 69. Final Takeaway

An API gateway's hard problem is not routing — routing is a lookup. The hard problem is centralizing cross-cutting concerns without turning that centralization into a single point of failure or a wall of hard-coded `if/else`, which is exactly what the Chain of Responsibility, Strategy, and State patterns solve, each for a different concern. The rate limiter's hard problem is not the algorithm — token bucket versus sliding window is a well-understood, well-documented choice. The hard problem is that the moment a rate limiter has to run correctly across more than one process, "count requests and compare to a limit" stops being a data-structure question and becomes a **distributed-systems atomicity** question — and the entire back half of this guide exists because that transition is where a design that sounds complete on a whiteboard quietly stops being correct in production.
