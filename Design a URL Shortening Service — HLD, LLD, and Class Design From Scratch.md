# Design a URL Shortening Service — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a URL shortening service like bit.ly or TinyURL. Cover both high-level design (scale, availability, storage) and low-level design (class diagrams, the actual code-generation algorithm). Follow SOLID design principles, and be ready to justify every decision when I push back — starting with why this 'trivial CRUD app' is actually a real distributed-systems problem."*
>
> This guide is structured exactly as that interview unfolds: a requirements-gathering phase, a high-level architecture built up decision by decision, a low-level class design deep dive, and a sequence of escalating **follow-up questions** — each answered with real reasoning and real code, ending where this question always ends up: generating billions of globally-unique short codes without a single point of failure, and keeping the redirect path — the one operation that runs on every single click — fast forever.

---

# 1. What We Are Building

We are building **MiniShort** — a URL shortening service covering:

- **Functional requirements**: shorten a long URL into a short code, redirect a short code to its original URL, support user-chosen custom aliases, link expiration, and click analytics.
- **High-level design**: why this looks like a trivial CRUD app and isn't, the massive read:write ratio that shapes every downstream decision, a caching and sharding strategy for the redirect path, and asynchronous analytics that never sits on that path.
- **Low-level design**: a real Base62 encoder, a distributed, collision-free unique ID generator (no coordination on the hot path), a Strategy-pattern code-generation layer supporting both generated and custom codes, and a sharded, cached storage layer.
- **Scaling concerns**: what happens at billions of stored URLs and tens of thousands of redirects per second, and the specific things that break first.

```text
   POST /shorten {longUrl}                    GET /{shortCode}
          |                                          |
   Rate Limiter (§37-§38)                    Cache (hot codes, §31-§33)
          |                                          | miss
   ShortCodeGenerator (§17-§25)               Sharded Store (§34-§36)
          |                                          |
   Sharded Store (write)                      301/302 redirect (§29-§30)
                                                      |
                                          Async analytics event (§39-§41, NEVER on this path)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain precisely why a URL shortener is a real systems-design problem, not a CRUD exercise — and name the one ratio (read:write) that shapes almost every other decision in the design.
- Compare three approaches to generating a short code (random-with-collision-check, hash-and-truncate, counter-plus-Base62) and explain exactly why the industry-standard answer is the third one.
- Implement a real Base62 encoder/decoder, and a distributed unique-ID generator that never makes two application servers contend on a single shared counter for every request.
- Explain the real tradeoff between a 301 and a 302 redirect for this specific use case, and why the "obviously better" choice isn't actually free.
- Design a cached, sharded storage layer for the redirect path, and an asynchronous analytics pipeline that never adds latency to it.
- Reason about what breaks first at billions of stored URLs and real production traffic, and name the specific technique that addresses each bottleneck.

---

# 3. Why This Matters (The Interview, Framed)

"Design a URL shortener" is one of the most commonly asked systems-design questions precisely because it *looks* trivial — a `Map<String, String>` and an HTTP redirect — and that appearance is exactly the trap:

- **Requirements-driven scoping** — custom aliases, expiration, and analytics each change the design meaningfully; a strong candidate confirms scope before designing, the same discipline every guide in this series insists on.
- **The read:write ratio is the entire ballgame** — a URL is shortened once and redirected from potentially millions of times; every high-level decision (caching, sharding, even which redirect status code to use) traces back to optimizing the read path specifically, at the acceptable cost of the write path.
- **The code-generation algorithm is where candidates actually get filtered** — "just hash it" sounds reasonable and is subtly wrong (hash collisions after truncation, no uniqueness guarantee); the correct answer (a distributed counter, Base62-encoded) requires understanding *why* a counter has no collisions at all, and how to make counter allocation itself not become a bottleneck or single point of failure.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and pattern implementations (Strategy, Facade). |
| Code generation | A distributed counter, Base62-encoded | Guarantees uniqueness with zero collision checks — the industry-standard approach, built in full in §17-§21. |
| Storage | A sharded key-value store (short code -> long URL) | The access pattern is a pure key lookup — no relational joins are ever needed on the redirect path (§34-§36). |
| Cache | An in-memory LRU cache in front of storage | The read:write ratio means a small set of popular links accounts for a disproportionate share of all redirects — caching them is the single highest-leverage optimization available (§31-§33). |
| Rate limiting | A token-bucket limiter on the shorten endpoint | Prevents abuse of the write path without touching the (far more latency-sensitive) redirect path at all (§37-§38). |
| Analytics | An asynchronous event queue, consumed off the critical path | A click must redirect immediately; analytics can lag by seconds without anyone noticing (§39-§41). |

---

# 5. Project Structure

```text
minishort/
├── src/main/java/com/example/minishort/
│   ├── domain/
│   │   └── ShortUrl.java                                          // §10
│   ├── encoding/
│   │   └── Base62Encoder.java                                     // §18
│   ├── idgen/
│   │   ├── RangeAllocator.java, IdRange.java                       // §21
│   │   └── DistributedIdGenerator.java                              // §21
│   ├── generation/
│   │   ├── ShortCodeGenerator.java <<interface>>                    // §24
│   │   ├── CounterBasedGenerator.java, CustomAliasGenerator.java     // §25
│   │   └── AliasAlreadyTakenException.java
│   ├── storage/
│   │   ├── UrlStore.java <<interface>>, ShardedUrlStore.java         // §35-§36
│   │   └── ShardRouter.java
│   ├── cache/
│   │   └── RedirectCache.java (LRU)                                  // §33
│   ├── ratelimit/
│   │   └── ShortenRateLimiter.java                                    // §38
│   ├── analytics/
│   │   └── ClickEvent.java, ClickEventPublisher.java                  // §40-§41
│   └── safety/
│       └── MaliciousUrlChecker.java                                    // §46
└── src/test/java/com/example/minishort/
    ├── Base62RoundTripTest.java
    ├── RangeAllocatorNoDuplicateIdsTest.java
    └── ShardRoutingConsistencyTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"Do we need user-chosen custom aliases, or only system-generated codes? Should links expire, or live forever by default? Do we need click analytics, and if so, in real time or is a delay acceptable? What's the expected scale — millions or billions of stored URLs, and what's the expected read:write ratio?"*

Exactly as every prior guide in this series argues, narrowing an intentionally broad prompt before designing anything is the first move that separates a strong answer from a shallow one. For this guide, we settle on a concrete, realistic scope: **both generated and custom aliases** supported, **optional expiration** (a link lives forever unless a TTL was specified), **analytics required but explicitly asynchronous** (never blocking a redirect), targeting **billions of stored URLs** with a **read:write ratio on the order of 100:1 or higher** — the single number that shapes nearly everything from here on.

---

# 7. Functional Requirements

- Given a long URL, **generate a unique short code** and return a short URL.
- Given a short code, **redirect** the caller to the original long URL.
- Support **user-chosen custom aliases**, rejected if already taken.
- Support an optional **expiration time** on a shortened link.
- Track **click analytics** per short URL (count, timestamp, referrer) without adding latency to the redirect itself.
- Prevent shortening a URL that points at a **known-malicious** destination.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Extremely low redirect latency** | A redirect is on the critical path of every single click — it must be fast, always | Caching (§31-§33), sharded storage keyed for direct lookup (§34-§36) |
| **High availability** | A broken redirect makes every previously-shared link dead; this is the one thing that must never be down | Sharding with replication, no single point of failure anywhere on the read path |
| **Massive read:write skew** | Reads (redirects) vastly outnumber writes (shortens) — every optimization should target reads first | The entire architecture, starting from §13 |
| **Global uniqueness with no collisions** | Two different long URLs must never accidentally resolve from the same generated short code | Counter-based generation, not random or hash-based (§15-§17) |
| **No single point of contention for ID generation** | Generating IDs must scale horizontally, not bottleneck on one shared counter | Distributed range-based allocation (§19-§21) |
| **Extensibility** | Adding a new code-generation strategy must not require rewriting the shortening service | The Strategy pattern (§24-§25) |

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Before architecture — what actually exists in this system, and how does it relate?"*

This is the same deliberate pivot every prior guide in this series makes — naming the domain model before naming components keeps the design honest about what actually needs solving.

---

# 10. Identifying the Core Domain Entities

| Entity | Represents | Key relationships |
|---|---|---|
| **ShortUrl** | One mapping from a short code to a long URL | Has an optional expiration, a creation timestamp, an optional owning user |
| **ShortCodeGenerator** | The strategy that produces a short code for a new `ShortUrl` | Pluggable — a counter-based generator or a custom-alias generator (§24) |
| **IdRange** | A block of pre-allocated, contention-free unique IDs handed to one server | Consumed locally, without contacting a central allocator per request (§20-§21) |
| **UrlStore** | The sharded, durable mapping from short code to `ShortUrl` | The system of record every redirect ultimately reads from (§34-§36) |
| **RedirectCache** | An in-memory, LRU-evicted cache of hot short codes | Consulted before `UrlStore` on every redirect (§31-§33) |
| **ClickEvent** | One recorded fact: "this short code was redirected, at this time, from this referrer" | Published asynchronously, never blocking the redirect itself (§39-§41) |

Every section from §11 onward either builds infrastructure **around** these entities or builds the entities **themselves** as real, working class designs — nothing introduced later is untraceable back to this table.

---

# 11. High-Level Architecture Overview

```text
                             Client
                                |
                    +-----------+-----------+
                    v                       v
            POST /shorten             GET /{shortCode}
            (rate-limited, §37-§38)   (the critical path -- optimize THIS)
                    |                       |
            ShortCodeGenerator (§17-§25)    RedirectCache (§31-§33) --miss--> Sharded UrlStore (§34-§36)
                    |                       |
            Sharded UrlStore (write)        301/302 redirect (§27-§30)
                                             |
                                    ClickEventPublisher (async, §39-§41)
```

Two structurally different paths, deliberately treated asymmetrically: the **write path** (shortening) can afford a little latency and a little coordination overhead, because it happens once per URL; the **read path** (redirecting) cannot afford either, because it happens potentially millions of times for the same URL — every architectural decision from here on is, in one way or another, in service of keeping that second path fast.

---

# 12. Follow-up Question 2 — "This Looks Like a Trivial CRUD App. What Actually Makes It Hard?"

> **Interviewer:** *"A table mapping short codes to long URLs, and a redirect. What's actually hard about this?"*

§13 names the one number that makes it hard, precisely.

---

# 13. The Real Difficulty: Read:Write Ratio and the Redirect Latency Budget

A URL is shortened **once**. It can then be clicked — redirected — anywhere from zero to millions of times, indefinitely, for as long as it exists. This produces a read:write ratio that easily reaches 100:1, 1000:1, or higher for popular links, and it means: **every millisecond spent on the write path is paid once; every millisecond spent on the read path is paid on every single click, forever.** That asymmetry is the entire reason this "trivial CRUD app" needs caching (§31-§33), sharding tuned for point lookups (§34-§36), and a code-generation scheme that adds zero extra reads or writes at redirect time (§17 onward) — none of it is premature optimization; all of it is a direct, proportionate response to the one ratio that defines this domain.

---

# 14. Follow-up Question 3 — "How Do You Actually Generate a Short Code? Walk Through a Few Approaches and Their Flaws."

> **Interviewer:** *"Don't just tell me the answer you memorized. Walk me through the naive approaches first, and tell me specifically why each one is wrong."*

§15-§16 name two approaches that look reasonable and aren't; §17 builds the one that's actually correct.

---

# 15. Approach 1: Random String + Collision Check, and Why It Doesn't Scale

Generate a random 7-character string, check if it's already taken, retry on collision. This is **correct** — it eventually produces a unique code — but its performance **degrades as the keyspace fills up**: with billions of codes already stored, an increasing fraction of random guesses collide, and the service pays for an extra read (the collision check) on every single attempt, with the retry count itself becoming unpredictable and, at high load, a real source of tail latency. Worse, at high write concurrency, two requests can both pass the collision check for the same code before either has written it — a genuine race condition needing a real uniqueness constraint at the storage layer to catch, turning a probabilistic problem into a real, occasional failure mode.

---

# 16. Approach 2: Hash-and-Truncate, and Why It Still Collides

Hash the long URL (MD5/SHA-256) and take the first 6-7 characters of the hash's Base62 representation. This looks deterministic and collision-resistant, but truncating **any** hash to a short prefix reintroduces exactly the birthday-paradox collision risk a full-length hash was designed to avoid — a 6-character Base62 prefix has only `62^6` (~57 billion) possible values, and at billions of stored URLs, collisions become a real, regular occurrence requiring the *identical* collision-check-and-retry machinery §15 already needed, for no actual benefit over generating a random string in the first place.

---

# 17. Approach 3: A Global Counter + Base62 Encoding — the Standard Solution

The fix that eliminates collisions **entirely**, rather than merely making them rare: maintain a single, monotonically-increasing integer counter, and **Base62-encode** each new counter value into a short code. Because the counter never repeats a value, and Base62 encoding is a deterministic, reversible bijection, **two different URLs can never receive the same short code** — not "extremely unlikely," but structurally impossible, with zero collision checks needed anywhere in the write path.

```text
counter=0          -> "0"
counter=61         -> "Z"
counter=62         -> "10"
counter=125        -> "21"
counter=3521614606208 -> "3D2VvUE"  (a 7-character code, comfortably covering tens of trillions of URLs)
```

§18 implements the encoding itself; §19-§21 fix the one remaining problem this approach has — a single shared counter is a single point of contention.

---

# 18. Implementing Base62 Encoding and Decoding

```java
public final class Base62Encoder {

    private static final String ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int BASE = ALPHABET.length(); // 62

    public String encode(long value) {
        if (value == 0) return String.valueOf(ALPHABET.charAt(0));
        StringBuilder result = new StringBuilder();
        long remaining = value;
        while (remaining > 0) {
            int digit = (int) (remaining % BASE);
            result.append(ALPHABET.charAt(digit)); // built in REVERSE order -- least significant digit first
            remaining /= BASE;
        }
        return result.reverse().toString();
    }

    public long decode(String shortCode) {
        long value = 0;
        for (char c : shortCode.toCharArray()) {
            int digit = ALPHABET.indexOf(c);
            if (digit == -1) throw new IllegalArgumentException("Invalid character in short code: " + c);
            value = value * BASE + digit;
        }
        return value;
    }
}
```

This is precisely positional-notation encoding, exactly like decimal-to-binary conversion, just with a 62-symbol alphabet instead of 2 or 10 — `encode` repeatedly extracts the least-significant "digit" in base 62 via `% BASE` and `/ BASE`, and `decode` reverses that with Horner's method (`value = value * BASE + digit`, processing most-significant digit first). Using digits, then uppercase, then lowercase letters as the alphabet (`0-9A-Za-z`) is the conventional ordering, chosen for URL-safety (every character is a valid, unreserved URL path segment character) — never `+`/`/` the way Base64 would use, which would require URL-encoding and defeat the entire point of a "short" code.

---

# 19. Follow-up Question 4 — "A Single Global Counter Is a Single Point of Failure and a Bottleneck. Fix It."

> **Interviewer:** *"Every application server incrementing the same shared counter for every single shorten request is a contention point, and a single point of failure. How do you fix that without reintroducing collisions?"*

§20 names the standard fix; §21 implements it in full.

---

# 20. Distributed Unique ID Generation: Range-Based Allocation

Instead of every application server contacting a central counter for **every** ID, have each server request a **block** of IDs at once (say, 1,000 at a time) from a lightweight, central range allocator, then hand out IDs from that block **locally**, in memory, with zero coordination, until the block is exhausted — at which point it requests the next block. The central allocator's own job shrinks dramatically: instead of handling one request per *shortened URL*, it handles one request per *thousand* shortened URLs, and its own internal state (the next range to hand out) is a single, trivially-persisted counter, never contended at the same rate the application tier is.

```text
App Server A requests a block -> allocator hands out [0, 999]      -> A serves IDs 0,1,2,... locally, no coordination
App Server B requests a block -> allocator hands out [1000, 1999]  -> B serves IDs 1000,1001,... locally
App Server A exhausts its block -> requests another -> allocator hands out [2000, 2999]
```

No two servers are ever handed overlapping ranges, which is the entire uniqueness guarantee — each server can generate IDs from its own current range as fast as it wants, in memory, with no network round trip and no lock contention with any other server, for as long as that range lasts.

---

# 21. Implementing a Range-Based ID Allocator

```java
public record IdRange(long startInclusive, long endInclusive) {
    public boolean isExhausted(long nextValue) { return nextValue > endInclusive; }
}
```

```java
/** The CENTRAL allocator -- one instance, backed by a single persisted counter. Called rarely
 *  (once per BLOCK, not once per ID), which is exactly what keeps it from being a bottleneck. */
public final class RangeAllocator {

    private static final long BLOCK_SIZE = 1000;
    private final AtomicLong persistedNextRangeStart; // durably stored -- e.g. one row in a small, dedicated table

    public RangeAllocator(long initialValue) { this.persistedNextRangeStart = new AtomicLong(initialValue); }

    public synchronized IdRange allocateNextRange() {
        long start = persistedNextRangeStart.getAndAdd(BLOCK_SIZE);
        long end = start + BLOCK_SIZE - 1;
        // In a real system: persist `persistedNextRangeStart`'s NEW value durably here, BEFORE returning --
        // exactly the "durable before handed out" discipline this project's own storage-engine guide
        // applies to a WAL record, applied here to an ID range instead of a page mutation.
        return new IdRange(start, end);
    }
}
```

```java
/** Runs INSIDE each application server -- the class every shorten request actually calls. */
public final class DistributedIdGenerator {

    private final RangeAllocator allocator;
    private IdRange currentRange;
    private long nextValue;

    public DistributedIdGenerator(RangeAllocator allocator) { this.allocator = allocator; }

    public synchronized long nextId() {
        if (currentRange == null || currentRange.isExhausted(nextValue)) {
            currentRange = allocator.allocateNextRange(); // a network/DB round trip -- but only once per BLOCK_SIZE ids
            nextValue = currentRange.startInclusive();
        }
        return nextValue++;
    }
}
```

`DistributedIdGenerator.nextId()` only ever calls the (comparatively expensive, coordinated) `allocateNextRange()` once every `BLOCK_SIZE` calls — the other 999 calls out of every 1,000 are a plain, uncontended, in-memory increment. This is the exact mechanism that turns "one contention point for every write" into "one contention point every thousand writes," without weakening the uniqueness guarantee at all: every ID that's ever handed out, by any server, still comes from a disjoint range no other server was ever given.

---

# 22. Follow-up Question 5 — "How Do Custom Aliases Fit Into a Counter-Based Scheme?"

> **Interviewer:** *"A user wants `bit.ly/my-startup` instead of a generated code. That's not a Base62-encoded number at all. How does that fit into what you just built?"*

It doesn't try to fit into the counter at all — §23 explains why a custom alias is a structurally different case, not a variant of the same one.

---

# 23. Custom Aliases: A Separate Namespace, Not a Special Case of the Counter

A generated code's uniqueness is guaranteed *structurally* by the counter (§17) — no check is ever needed. A custom alias is **user-chosen text**, with no relationship to the counter at all, so its uniqueness has to be verified the old-fashioned way: **check if it's already taken, reject if so.** Both kinds of code end up in the exact same `UrlStore` (§34), keyed identically — a redirect for `/my-startup` and a redirect for `/3D2VvUE` go through the identical lookup path — but *creating* one is a fundamentally different operation from creating the other, which is exactly why §24-§25 model them as two implementations of one interface, not one method with a branch inside it.

---

# 24. The ShortCodeGenerator Strategy Interface

```java
public interface ShortCodeGenerator {
    String generate(String longUrl, String requestedAlias /* nullable */) throws AliasAlreadyTakenException;
}
```

One interface, one method, called identically by the shortening service regardless of which concrete strategy is configured — the service itself never needs an `if (customAliasRequested)` branch anywhere in its own logic.

---

# 25. Implementing CounterBasedGenerator and CustomAliasGenerator

```java
public final class CounterBasedGenerator implements ShortCodeGenerator {

    private final DistributedIdGenerator idGenerator; // §21
    private final Base62Encoder encoder;                // §18

    public CounterBasedGenerator(DistributedIdGenerator idGenerator, Base62Encoder encoder) {
        this.idGenerator = idGenerator;
        this.encoder = encoder;
    }

    @Override
    public String generate(String longUrl, String requestedAlias) {
        return encoder.encode(idGenerator.nextId()); // requestedAlias is IGNORED here -- §26's dispatcher decides which
                                                        // generator to call based on whether one was actually provided
    }
}
```

```java
public final class CustomAliasGenerator implements ShortCodeGenerator {

    private final UrlStore urlStore; // §34-§36

    public CustomAliasGenerator(UrlStore urlStore) { this.urlStore = urlStore; }

    @Override
    public String generate(String longUrl, String requestedAlias) throws AliasAlreadyTakenException {
        if (requestedAlias == null || requestedAlias.isBlank()) {
            throw new IllegalArgumentException("CustomAliasGenerator requires a non-blank alias");
        }
        if (urlStore.exists(requestedAlias)) { // the ONE collision check this entire design ever needs --
            throw new AliasAlreadyTakenException(requestedAlias); // confined to exactly the one case (custom
        }                                                          // text) where uniqueness genuinely isn't structural
        return requestedAlias;
    }
}
```

```java
/** The one place that decides WHICH strategy to use -- never inside either generator itself. */
public final class ShortCodeGeneratorDispatcher {

    private final CounterBasedGenerator counterBasedGenerator;
    private final CustomAliasGenerator customAliasGenerator;

    public ShortCodeGeneratorDispatcher(CounterBasedGenerator counterBasedGenerator, CustomAliasGenerator customAliasGenerator) {
        this.counterBasedGenerator = counterBasedGenerator;
        this.customAliasGenerator = customAliasGenerator;
    }

    public String generate(String longUrl, String requestedAlias) throws AliasAlreadyTakenException {
        ShortCodeGenerator generator = (requestedAlias != null && !requestedAlias.isBlank())
                ? customAliasGenerator
                : counterBasedGenerator;
        return generator.generate(longUrl, requestedAlias);
    }
}
```

The collision check `urlStore.exists(requestedAlias)` is confined to exactly `CustomAliasGenerator` — `CounterBasedGenerator` never calls it, never needs to, and never could accidentally be made to skip it either, since the check simply doesn't exist in that class at all. This is the Strategy pattern doing real work, not decoration: the two generation strategies don't just *look* different, they have genuinely different correctness properties (structural uniqueness vs. checked uniqueness), and keeping them as separate classes is what makes that difference impossible to blur accidentally.

---

# 26. Class Diagram: Domain Model and Code Generation

```text
ShortCodeGenerator <<interface>>                    ShortUrl
+ generate(longUrl, requestedAlias): String         + shortCode, longUrl
      ^                                              + createdAt, expiresAt (nullable)
      | implements                                          ^
  +---+----------------+                                     | produces
CounterBasedGenerator  CustomAliasGenerator                  |
      |                       |                     ShortCodeGeneratorDispatcher
      v                       v                     + generate(longUrl, requestedAlias)
DistributedIdGenerator   UrlStore.exists(alias)              |
      |                                              consulted by the shortening service,
      v                                              which then writes the resulting ShortUrl
RangeAllocator (§21)                                 into UrlStore (§34-§36)
      |
      v
Base62Encoder (§18)
```

Every arrow into `ShortCodeGeneratorDispatcher` reflects the real dependency direction: the shortening service depends on the dispatcher and nothing else, the dispatcher depends on both concrete generators, and each generator depends only on whatever it specifically needs (an ID generator plus an encoder for one; the store itself for the other) — never on each other.

---

# 27. Follow-up Question 6 — "Walk Me Through the Redirect Path End to End. What's on the Critical Path?"

> **Interviewer:** *"A user clicks a shortened link. Trace every step until their browser lands on the real destination."*

§28 traces it precisely, and names what belongs on that path and — just as importantly — what must never be allowed onto it.

---

# 28. The Redirect Path and Why Every Millisecond There Matters

```text
GET /{shortCode}
      |
RedirectCache.get(shortCode)  --hit-->  return cached longUrl  --> HTTP 301/302 (§29-§30)
      |  miss
UrlStore.lookup(shortCode) (§34-§36)  --> populate cache (§31-§33)  --> HTTP 301/302
      |
ClickEventPublisher.publish(...)  -- fired AFTER the redirect response is already sent, NEVER awaited (§39-§41)
```

Exactly two things are allowed on this path: a cache or store lookup, and issuing the redirect itself. Everything else this guide builds — analytics recording, expiration bookkeeping, malicious-URL re-verification — happens **after** the response is already on its way back to the client, specifically because none of it is something a user waits on, and all of it can tolerate being slightly stale or slightly delayed in a way that a redirect itself cannot.

---

# 29. Follow-up Question 7 — "301 or 302 Redirect? Does It Matter?"

> **Interviewer:** *"HTTP gives you two flavors of redirect. Which one do you use here, and does the choice actually matter, or is it a coin flip?"*

It matters, and in a direction that's genuinely counter-intuitive the first time through it: the "obviously better-performing" choice is actually the wrong default for this specific service, for a reason tied directly back to §39's analytics requirement.

---

# 30. Permanent vs. Temporary Redirects: The Real Tradeoff

A **301 (Moved Permanently)** tells the browser this mapping will never change — and browsers **cache that fact locally**, meaning every *subsequent* click on the same link, from the same browser, never even reaches the shortening service's servers again; the browser redirects on its own, straight from its local cache. A **302 (Found / Temporary)** tells the browser to check back with the server every single time. The 301's browser-side caching sounds like a pure win — less server load — but it directly **defeats click analytics** (§39-§41): a click the server never sees is a click that can never be counted, referrer-tracked, or geo-tagged. The standard, correct answer for a service whose functional requirements (§7) explicitly include analytics is **302**, deliberately trading some avoidable server load for the ability to observe every single click — a real tradeoff, decided by an actual requirement, not a default picked without reasoning.

---

# 31. Follow-up Question 8 — "How Do You Make Redirects Fast at Scale, Given the Read:Write Ratio?"

> **Interviewer:** *"§13 established reads dominate massively. What specifically do you do about it, beyond 'use a fast database'?"*

§32 names the standard fix; §33 implements it.

---

# 32. Caching Hot Short Codes

Real-world click distribution on shortened links is heavily skewed — a small fraction of links (a viral tweet, a marketing campaign) accounts for a disproportionate share of all redirects, the same "hot key" pattern this project's own guides have already met in a CDN context and a rate-limiter context. An in-memory LRU cache sitting in front of the sharded store (§34-§36), on every application server, turns the overwhelming majority of redirects for popular links into a pure in-memory lookup — no network round trip to storage at all.

---

# 33. Implementing a Redirect Cache

```java
public final class RedirectCache {

    private final int capacity;
    private final LinkedHashMap<String, ShortUrl> cache;

    public RedirectCache(int capacity) {
        this.capacity = capacity;
        // accessOrder=true is the ENTIRE LRU mechanism -- every get() moves an entry to the "most recently
        // used" end internally; removeEldestEntry below evicts from the OTHER end once over capacity.
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, ShortUrl> eldest) {
                return size() > RedirectCache.this.capacity;
            }
        };
    }

    public synchronized Optional<ShortUrl> get(String shortCode) {
        return Optional.ofNullable(cache.get(shortCode));
    }

    public synchronized void put(String shortCode, ShortUrl shortUrl) {
        cache.put(shortCode, shortUrl);
    }
}
```

`LinkedHashMap`'s own `accessOrder=true` constructor flag, plus overriding `removeEldestEntry`, is the entire LRU cache mechanism — no hand-rolled doubly-linked list needed, the same zero-extra-code LRU trick this project's own database-internals guides already reach for. Every cache hit here is exactly what the redirect path (§28) needs: one in-memory map lookup, no network call, no coordination — the single highest-leverage optimization this design makes, precisely because it targets the read path §13 identified as the one that matters most.

---

# 34. Follow-up Question 9 — "The Mapping Table Has Billions of Rows. How Do You Scale Storage?"

> **Interviewer:** *"A cache miss still has to hit real storage. At billions of stored URLs, what does that storage layer actually look like?"*

§35 names the fix; §36 implements the routing it requires.

---

# 35. Sharding the URL Mapping Store

No single database instance holds billions of rows and serves the required read volume comfortably — the store is **sharded**, splitting the keyspace across many physical database instances, each holding a manageable fraction of the total. Because every lookup is a **pure key lookup** (a short code, nothing else — never a range scan or a join), sharding by a hash of the short code itself is both natural and sufficient: this is the identical consistent-hashing idea this project's own dedicated guide builds out in full depth, applied here to route a lookup to the one shard that could possibly hold it, in `O(1)`, with no need to fan a query out to every shard and merge results.

---

# 36. Implementing Shard Routing

```java
public interface UrlStore {
    void save(ShortUrl shortUrl);
    boolean exists(String shortCode);
    Optional<ShortUrl> lookup(String shortCode);
}
```

```java
public final class ShardRouter {

    private final List<UrlStore> shards;

    public ShardRouter(List<UrlStore> shards) { this.shards = shards; }

    public UrlStore shardFor(String shortCode) {
        int shardIndex = Math.floorMod(shortCode.hashCode(), shards.size()); // a simple modulo hash over shard COUNT
        return shards.get(shardIndex);
    }
}
```

```java
public final class ShardedUrlStore implements UrlStore {

    private final ShardRouter router;

    public ShardedUrlStore(ShardRouter router) { this.router = router; }

    @Override public void save(ShortUrl shortUrl) { router.shardFor(shortUrl.shortCode()).save(shortUrl); }
    @Override public boolean exists(String shortCode) { return router.shardFor(shortCode).exists(shortCode); }
    @Override public Optional<ShortUrl> lookup(String shortCode) { return router.shardFor(shortCode).lookup(shortCode); }
}
```

A plain modulo hash over the shard **count** is deliberately named as the simple, honest starting point here, not the final word — adding or removing a shard with this scheme remaps the majority of keys, exactly the failure mode this project's own consistent-hashing guide builds an entire ring-based structure specifically to avoid; a production system re-shards using that structure instead of this section's simplified router, named explicitly as future work (§55) rather than silently glossed over.

---

# 37. Follow-up Question 10 — "How Do You Stop Someone From Writing a Script That Shortens a Million URLs a Minute?"

> **Interviewer:** *"Nothing so far stops abuse of the shorten endpoint specifically. What do you add?"*

Rate limiting — deliberately scoped to the **write** path only, never touching the redirect path's latency budget at all.

---

# 38. Rate Limiting the Shorten Endpoint

```java
public final class ShortenRateLimiter {

    private final Map<String, TokenBucket> bucketsByApiKey = new ConcurrentHashMap<>();
    private final long capacity;
    private final double refillTokensPerSecond;

    public ShortenRateLimiter(long capacity, double refillTokensPerSecond) {
        this.capacity = capacity;
        this.refillTokensPerSecond = refillTokensPerSecond;
    }

    public boolean allow(String apiKey) {
        TokenBucket bucket = bucketsByApiKey.computeIfAbsent(apiKey, k -> new TokenBucket(capacity, refillTokensPerSecond));
        return bucket.tryConsume();
    }
}
```

This is deliberately the same token-bucket shape this project's own dedicated API-gateway-and-rate-limiter guide builds out in full — capacity, refill rate, per-key isolation — reused here at exactly the scope it's actually needed: only in front of `POST /shorten`, never in front of `GET /{shortCode}`, because rate-limiting the redirect path would directly contradict §13's entire premise that the read path must stay as fast and unencumbered as possible.

---

# 39. Follow-up Question 11 — "How Do You Track Click Analytics Without Slowing Down the Redirect Itself?"

> **Interviewer:** *"Analytics is a stated requirement (§7). The redirect can't wait on writing an analytics record. How do you reconcile that?"*

§40-§41 answer with an asynchronous event pipeline — the redirect fires an event and returns immediately, never waiting for that event to actually be processed anywhere.

---

# 40. Asynchronous Click Analytics

```text
Redirect handler:
   1. Look up shortCode (cache or store)
   2. Send the 302 response back to the client                <-- client is DONE waiting, right here
   3. Publish a ClickEvent to a queue (fire-and-forget)         <-- happens AFTER step 2, never blocking it
                                                                      |
                                                                      v
                                                         Analytics consumer (separate process/service)
                                                         aggregates counts, updates dashboards, etc.
```

Step 3 happening **after** the response is already sent — not merely asynchronously dispatched but genuinely sequenced after the client-facing work is done — is what guarantees analytics can never add latency to a redirect, even in the degenerate case where the event queue itself is temporarily slow or unavailable.

---

# 41. Implementing Async Analytics Event Publishing

```java
public record ClickEvent(String shortCode, Instant occurredAt, String referrer, String userAgent) { }
```

```java
public final class ClickEventPublisher {

    private final ExecutorService analyticsExecutor; // a small, BOUNDED pool, isolated from the redirect path's own threads

    public ClickEventPublisher(ExecutorService analyticsExecutor) { this.analyticsExecutor = analyticsExecutor; }

    /** Called from the redirect handler, AFTER the 302 has already been written to the response. */
    public void publish(ClickEvent event) {
        analyticsExecutor.submit(() -> {
            try {
                sendToQueue(event); // a real message queue client call, elided here
            } catch (Exception e) {
                // A dropped analytics event is a missed data point, never a user-facing failure -- log and move on,
                // deliberately never propagating this failure back to anything the client could observe.
                logDropped(event, e);
            }
        });
    }

    private void sendToQueue(ClickEvent event) { /* elided -- a real queue client (Kafka-style) publish call */ }
    private void logDropped(ClickEvent event, Exception e) { /* elided -- structured logging, not a rethrow */ }
}
```

Running `publish` on its **own**, separately-bounded thread pool — never the same pool serving redirect requests — means a slow or backed-up analytics pipeline can never starve the redirect path of threads, the identical pool-isolation argument this project's own executor-framework and API-gateway guides already make for isolating a slow concern from a fast one.

---

# 42. Follow-up Question 12 — "Links Can Expire. How Do You Reclaim That Storage Safely?"

> **Interviewer:** *"A link created with a 30-day TTL is now 60 days old. It's still sitting in storage, occupying space and still technically redirectable. How do you actually reclaim it, correctly?"*

§43 states the two-part answer: an expired link must stop being **usable** immediately, and its storage must eventually be **reclaimed**, and these are two different guarantees on two different timelines.

---

# 43. Expiration and Garbage Collection

An expired `ShortUrl` must **stop redirecting** the moment it expires — checked at lookup time, on every redirect, regardless of whether cleanup has run yet — but its row does **not** need to be physically deleted at that same instant; deletion can happen later, in a batched background pass, exactly the same "correctness first, physical reclamation on its own schedule" split this project's own storage-engine and version-control guides already make for a tombstoned delete and a garbage-collection pass respectively.

---

# 44. Implementing Expiry Enforcement and Cleanup

```java
public final class RedirectService {

    private final RedirectCache cache;   // §33
    private final UrlStore store;         // §34-§36

    public RedirectService(RedirectCache cache, UrlStore store) { this.cache = cache; this.store = store; }

    public Optional<String> resolve(String shortCode) {
        Optional<ShortUrl> shortUrl = cache.get(shortCode).or(() -> fetchAndCache(shortCode));
        return shortUrl
                .filter(url -> url.expiresAt() == null || Instant.now().isBefore(url.expiresAt())) // the expiry CHECK,
                .map(ShortUrl::longUrl);                                                              // enforced on EVERY lookup
    }

    private Optional<ShortUrl> fetchAndCache(String shortCode) {
        Optional<ShortUrl> fromStore = store.lookup(shortCode);
        fromStore.ifPresent(url -> cache.put(shortCode, url));
        return fromStore;
    }
}
```

```java
/** Runs periodically, off any request path entirely -- a background batch job, not a per-request check. */
public final class ExpiredLinkCleanupJob {

    private final UrlStore store;

    public ExpiredLinkCleanupJob(UrlStore store) { this.store = store; }

    public void runOnce() {
        store.deleteWhereExpiredBefore(Instant.now()); // a single, bounded batch DELETE -- elided in full here
    }
}
```

`RedirectService.resolve`'s `.filter(...)` check is what makes expiry **immediately correct** — an expired link stops redirecting the instant its `expiresAt` passes, entirely independent of whether `ExpiredLinkCleanupJob` has run yet — while the cleanup job itself only ever needs to run occasionally, reclaiming storage as a housekeeping concern completely decoupled from correctness, which is exactly what makes it safe to run rarely, in large batches, without any risk of a stale link staying redirectable in the meantime.

---

# 45. Follow-up Question 13 — "How Do You Stop Someone From Shortening a URL to a Phishing or Malware Site?"

> **Interviewer:** *"A URL shortener is an attractive tool for hiding a malicious destination behind an innocuous-looking link. What do you actually do about that?"*

§46 answers with a check on the write path — deliberately never on the read path, for the identical reason rate limiting (§37-§38) lives only on the write path.

---

# 46. Malicious URL Protection

```java
public interface MaliciousUrlChecker {
    boolean isMalicious(String longUrl);
}
```

```java
public final class SafeBrowsingBackedChecker implements MaliciousUrlChecker {

    private final SafeBrowsingClient safeBrowsingClient; // an integration with a real, industry-standard threat-intel service

    public SafeBrowsingBackedChecker(SafeBrowsingClient safeBrowsingClient) { this.safeBrowsingClient = safeBrowsingClient; }

    @Override
    public boolean isMalicious(String longUrl) {
        return safeBrowsingClient.check(longUrl); // delegates to a real, continuously-updated threat database
    }
}
```

Checking against a real, industry-standard threat-intelligence service (Google Safe Browsing or an equivalent) rather than hand-rolling a blacklist is the identical engineering judgment this project's own driver and database-server guides already apply to password hashing and TLS: a maliciousness determination needs a continuously-updated, professionally-maintained data source behind it, not a homegrown list that's stale the day it's written — and this check belongs squarely on the **shorten** path, checked once at creation time, never repeated on every redirect, which would reintroduce exactly the redirect-latency cost §13 spends this entire design avoiding.

---

# 47. Follow-up Question 14 — "At Google/bit.ly Scale, What Breaks First?"

> **Interviewer:** *"Billions of stored URLs, tens of thousands of redirects per second, globally distributed users. What's the next bottleneck once everything in this guide is already built?"*

Physical distance itself — a request from Tokyo to a data center in Virginia pays real speed-of-light latency no amount of caching or sharding *within* that data center can remove.

---

# 48. Scaling the Read Path: CDN and Edge Redirect Caching

The fix mirrors the CDN architecture this project's own video-streaming and file-storage guides already build for a different kind of content: push the redirect decision itself out to **edge locations** geographically close to users, each edge node holding its own cache of hot short-code-to-long-URL mappings (§32-§33's identical LRU cache, just running at the edge instead of only in an origin data center), falling back to the origin service only on a genuine edge-cache miss. A redirect served entirely from a nearby edge node never pays the cross-continent round trip at all — the same multiplicative load-reduction argument a tiered CDN makes for video segments applies identically here to a much smaller, much more cacheable payload (a single short code -> long URL mapping is a handful of bytes, trivially replicable to every edge location that wants it).

---

# 49. Full Worked Example: Shortening and Redirecting a URL, End to End

```text
1.  POST /shorten {longUrl: "https://example.com/a/very/long/path", alias: null}
2.  ShortenRateLimiter.allow(apiKey) -> true                                              (§38)
3.  MaliciousUrlChecker.isMalicious(longUrl) -> false                                       (§46)
4.  ShortCodeGeneratorDispatcher.generate(longUrl, null) -> CounterBasedGenerator          (§25)
       -> DistributedIdGenerator.nextId() -> 3521614606208 (served from this server's LOCAL range, no round trip)  (§21)
       -> Base62Encoder.encode(3521614606208) -> "3D2VvUE"                                  (§18)
5.  ShardRouter.shardFor("3D2VvUE") -> shard 4 -> UrlStore.save(...)                        (§36)
6.  Response: { shortUrl: "https://mini.sh/3D2VvUE" }

7.  GET /3D2VvUE  (a later click, from a different user)
8.  RedirectCache.get("3D2VvUE") -> miss (first click on this code)                          (§33)
9.  ShardRouter.shardFor("3D2VvUE") -> shard 4 -> UrlStore.lookup(...) -> found              (§36)
10. RedirectCache.put("3D2VvUE", shortUrl)                                                   (§33)
11. expiresAt check passes (no expiry set)                                                   (§44)
12. HTTP 302 Found -> Location: https://example.com/a/very/long/path                        (§30)
13. ClickEventPublisher.publish(...) -- AFTER the response above was already sent            (§40-§41)

14. GET /3D2VvUE  (a SECOND click, moments later)
15. RedirectCache.get("3D2VvUE") -> HIT -- no store lookup, no shard routing, pure in-memory  (§33)
16. HTTP 302 Found -- served in a fraction of the time step 12 took
```

Step 16's speed difference from step 12 is the entire payoff of §32-§33's caching layer, made concrete: the *first* click on a brand-new code pays a real shard lookup; every click after that, for as long as the cache entry survives, pays almost nothing.

---

# 50. Final Architecture Diagram

```text
                          Client
                             |
                  +----------+----------+
                  v                     v
          POST /shorten           GET /{shortCode}
                  |                     |
      ShortenRateLimiter (§38)    RedirectCache (§33) --miss--> ShardRouter -> UrlStore shard (§35-§36)
                  |                     |
      MaliciousUrlChecker (§46)   expiresAt check (§44)
                  |                     |
      ShortCodeGeneratorDispatcher    HTTP 301/302 (§30)
      (§24-§25)                          |
                  |                ClickEventPublisher (async, §40-§41, on its OWN thread pool)
      ShardRouter -> UrlStore shard (write)
                  |
      DistributedIdGenerator -> RangeAllocator (§20-§21)
      Base62Encoder (§18)
```

---

# 51. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `ShortCodeGenerator` (§24-§25) | Counter-based and custom-alias generation are two interchangeable strategies behind one interface, dispatched by a single deciding class. |
| **Facade** | `RedirectService` (§44) | Cache lookup, store fallback, and expiry checking are all invisible to the caller behind one `resolve` call. |
| **Bijective Encoding** | `Base62Encoder` (§18) | A reversible, collision-free mapping between an integer counter and a short string — the algorithmic core this entire design depends on. |
| **Range/Lease Allocation** | `RangeAllocator`/`DistributedIdGenerator` (§20-§21) | Turns a single contention point into one contended call per thousand uncontended ones — the same amortized-coordination shape a distributed lock lease or a connection-pool checkout uses elsewhere. |
| **LRU Cache** | `RedirectCache` (§33) | The zero-extra-code `LinkedHashMap(accessOrder=true)` trick this project's own database-internals guides already use for the identical purpose. |

---

# 52. SOLID Principles Applied

- **Single Responsibility**: `Base62Encoder` only encodes/decodes; `RangeAllocator` only hands out ID ranges; `ShortenRateLimiter` only rate-limits. None of them know how to look up a URL or check for malicious content.
- **Open/Closed**: adding a new `ShortCodeGenerator` implementation (a vanity-domain generator, say) requires zero changes to `ShortCodeGeneratorDispatcher`'s callers — only a new `case` in the dispatch decision itself.
- **Liskov Substitution**: `CounterBasedGenerator` and `CustomAliasGenerator` are fully interchangeable wherever `ShortCodeGenerator` is used (§24) — the shortening service never branches on which one it's holding.
- **Interface Segregation**: `UrlStore` exposes exactly three methods (`save`/`exists`/`lookup`) — no sharding detail, no caching detail, no analytics detail leaks into its surface.
- **Dependency Inversion**: `RedirectService` (§44) depends on `RedirectCache` and `UrlStore` — both concrete collaborators, but each consumed only through the narrow, stable methods it actually needs — with `ShardedUrlStore` itself depending on the `UrlStore` interface for every individual shard, never a specific storage engine directly.

---

# 53. Common Mistakes When Building This Yourself

- **Random-string-plus-collision-check as the "final" design, not a named-and-rejected first draft** (§15) — degrades unpredictably as the keyspace fills, and hides a real race condition under concurrent writes.
- **A single, unpartitioned global counter** (§17, before §19's fix) — a real single point of failure and contention bottleneck, exactly the gap a real interview always asks about next.
- **Rate limiting or malicious-URL checks on the redirect path instead of the shorten path** (§38, §46) — directly contradicts the entire premise (§13) that the read path must stay minimal and fast.
- **A 301 redirect chosen by default "because it's faster"** (§30) — silently defeats click analytics, a stated functional requirement, by letting browsers bypass the server entirely on repeat clicks.
- **Physically deleting an expired link immediately instead of checking expiry at lookup time** (§43-§44) — couples correctness to a background job's schedule, when the two should be entirely decoupled.
- **Treating custom aliases and generated codes as the same code path with an `if` branch** (§23) — blurs two genuinely different correctness properties (structural uniqueness vs. checked uniqueness) into one class where a bug in one case can silently affect the other.

---

# 54. Testing Strategy

- **`Base62Encoder`** (§18): `decode(encode(n)) == n` for a wide range of values including `0`, a small value, and a very large `long`; `decode` throws on an invalid character.
- **`RangeAllocator`/`DistributedIdGenerator`** (§20-§21): two generators pulling from the same allocator concurrently never produce overlapping ranges, and therefore never produce the same ID — the single test that would catch a broken uniqueness guarantee directly.
- **`ShortCodeGeneratorDispatcher`** (§25): a request with a blank/null alias routes to `CounterBasedGenerator`; a request with a non-blank alias routes to `CustomAliasGenerator`; an already-taken alias throws `AliasAlreadyTakenException` rather than silently overwriting.
- **`RedirectCache`** (§33): a cache at capacity evicts the least-recently-used entry, not an arbitrary one; a `get` on a present key moves it to "most recently used," verified by then adding new entries and confirming that key survives eviction longer than it otherwise would.
- **`RedirectService`** (§44): a link past its `expiresAt` resolves to empty even though its row still physically exists in `UrlStore` — the test that proves expiry correctness never depends on the cleanup job having run.
- **`ShardRouter`** (§36): the same short code always routes to the same shard, repeatedly, across many calls — routing determinism is the one property everything else in the storage layer depends on.

---

# 55. Suggested Future Enhancements

- **A real consistent-hashing ring for shard routing** (§36) — replacing the plain modulo-over-shard-count router with the ring-based structure this project's own dedicated consistent-hashing guide builds in full, so adding or removing a shard remaps a small, bounded fraction of keys instead of the majority of them.
- **Edge/CDN-cached redirects** (§48) — pushing the hot-path cache lookup itself out to geographically distributed edge nodes, not just an origin-local in-memory cache.
- **QR code generation** for a shortened URL, a common, low-cost feature addition once the core shortening/redirect pipeline is solid.
- **Per-link, real-time analytics dashboards** — the `ClickEvent` stream (§40-§41) already carries everything needed; this is purely a consumption/aggregation layer on top of data already being captured.
- **Bulk shortening via an API/CSV upload** — a genuinely different write-path shape (many URLs in one request) that would need its own rate-limiting and batching consideration, distinct from the single-URL case this guide builds.
- **Link ownership and management** — letting a registered user list, edit the expiry of, or delete their own previously-shortened links, which the domain model (§10) already has the `owningUser` field to support but this guide doesn't build a UI/API surface for.

---

# 56. Progressive Interview Question Set

1. Why is a URL shortener a real distributed-systems problem, not a CRUD exercise — name the one ratio that drives almost every design decision.
2. Walk through why hash-and-truncate still produces real collisions at scale, even though a full-length cryptographic hash is collision-resistant.
3. Explain exactly why a counter-plus-Base62 scheme has zero collisions, structurally — not "very few," zero.
4. Why does handing out ID ranges in blocks reduce contention by roughly the block size, and what's the actual failure mode if two servers were ever handed overlapping ranges?
5. Why do custom aliases need a genuinely different generation strategy from generated codes, rather than a branch inside the same method?
6. Walk through the real tradeoff between a 301 and a 302 redirect for this specific service, and explain why the "faster" choice isn't the correct default here.
7. Why does rate limiting and malicious-URL checking belong exclusively on the write path, never the read path?
8. Why can expiry enforcement be checked at lookup time while physical deletion happens on a completely independent schedule, without any risk of an expired link staying accessible?
9. If asked to add "temporary custom aliases that expire and free up the alias for someone else to claim," how would you extend this design using pieces already built here?
10. What's the very next bottleneck once caching, sharding, rate limiting, and async analytics are all already in place — and why does none of this guide's existing machinery solve it?

---

# 57. Final Takeaway

Nearly every decision in this design is a direct, traceable consequence of one number: a URL is written once and read potentially millions of times. Once that ratio is taken seriously, caching, sharded point lookups, and a code-generation scheme that adds zero collision checks on the write path all stop looking like premature optimization and start looking like the only reasonable response to the actual shape of the workload. The single algorithmic insight that separates a shallow answer from a strong one — a monotonically increasing counter, Base62-encoded, has no collisions at all, and can be made contention-free by handing out ranges instead of individual values — is a small piece of math doing an enormous amount of architectural work, which is exactly why this question keeps showing up in interviews: it rewards a candidate who actually reasons about the problem, and quietly exposes one who's only memorized the word "hash."
