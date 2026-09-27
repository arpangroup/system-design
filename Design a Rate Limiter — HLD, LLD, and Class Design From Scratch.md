# Design a Rate Limiter — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Client --request--> [ Rate Limiter Middleware ] --allowed?--> [ Backend Service ]
                            |
                     [ Limit Store ]
                     (in-memory, or a shared
                      Redis-backed store for
                      a multi-server deployment)
                            |
                     429 + Retry-After  (if rejected)
```

A rate limiter decides, for every incoming request, whether the client (identified by user ID, API key, or IP) has exceeded a configured quota — and if so, rejects the request cheaply, before it ever reaches the backend service it would otherwise burden. This guide builds one from scratch: the classic algorithm family (fixed window, sliding window log, sliding window counter, token bucket, leaky bucket) with their precise tradeoffs, distributed coordination across many servers via a shared store, multi-tier limits, pluggable partitioning keys, and the client-facing rejection contract (status codes, headers, backoff).

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Explain the fixed-window boundary burst problem precisely, and describe two different fixes with different cost/accuracy tradeoffs.
- Implement token bucket and leaky bucket algorithms, and articulate the actual behavioral difference between them despite their near-identical formulas.
- Design a distributed rate limiter backed by a shared store (Redis), including why the check-and-decrement operation must be atomic and how to make it so.
- Design for the failure of the shared store itself, without either fully disabling rate limiting or fully blocking all traffic.
- Design multi-tier rate limits (e.g., simultaneous per-second and per-hour quotas) and pluggable partitioning keys (per-user, per-IP, per-API-key) without duplicating logic per combination.
- Design the correct client-facing rejection contract: status code, headers, and backoff guidance that prevents a rejected client from immediately retrying into a "thundering herd."
- Apply SOLID principles and recognizable design patterns (Strategy, Composite, Facade) to keep the limiter extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design a rate limiter" is one of the most commonly asked system design questions precisely because it looks deceptively simple (it's "just a counter with a threshold") while actually requiring careful reasoning about algorithmic precision, distributed atomicity, and graceful degradation — three concerns that separately show up in almost every other distributed systems question too. This guide frames the design as a live interview: each major algorithm or mechanism is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Records for immutable rate-limit decisions; virtual threads for high-concurrency request handling |
| Single-node algorithm implementations | Custom (built in this guide) | Teaches the actual mechanics behind libraries like Guava's `RateLimiter` and Resilience4j |
| Distributed shared store | Redis | Sub-millisecond round-trip, native atomic operations, and Lua scripting for compound check-and-update |
| Atomicity across check + update | Redis Lua scripting (`EVAL`) | Executes as a single atomic step server-side, closing the race condition a separate GET-then-SET would have |
| Client rejection contract | HTTP 429 + `Retry-After` + `X-RateLimit-*` headers | Industry-standard, machine-readable signal a well-behaved client can act on |

---

# 5. Project Structure

```text
rate-limiter/
├── src/main/java/com/example/ratelimit/
│   ├── core/
│   │   └── RateLimitDecision.java, RateLimitRule.java                 // §10
│   ├── algorithm/
│   │   ├── RateLimiter.java (Strategy interface)                      // §31
│   │   ├── FixedWindowCounter.java                                    // §18
│   │   ├── SlidingWindowLog.java                                      // §21
│   │   ├── SlidingWindowCounter.java                                  // §24
│   │   ├── TokenBucket.java                                           // §27
│   │   └── LeakyBucket.java                                           // §30
│   ├── distributed/
│   │   ├── RedisTokenBucket.java, token_bucket.lua                    // §35
│   │   └── LocalFallbackLimiter.java                                  // §39
│   ├── composite/
│   │   ├── MultiTierRateLimiter.java                                  // §42
│   │   └── KeyResolver.java (Strategy)                                // §44
│   └── http/
│       └── RateLimitResponseWriter.java (429, headers)                // §47
└── src/test/java/com/example/ratelimit/
    ├── FixedWindowBurstTest.java
    ├── TokenBucketBurstAllowanceTest.java
    └── RedisAtomicityRaceTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design a rate limiter."*

Before drawing any boxes, the intentionally vague prompt needs narrowing: rate-limit by what key — per user, per IP, per API key, or some combination? A single limit, or multiple simultaneous tiers (e.g., a burst limit and a sustained limit)? Deployed on a single server, or must the limit hold consistently across a fleet of many servers sharing the same clients? Is strict precision required, or is a well-understood approximation (with a documented error bound) acceptable in exchange for lower memory and latency cost? The answers to these determine which algorithm is even in consideration, and asking them signals the difference between a candidate reciting "token bucket" from memory and one who actually designed *this* system.

---

# 7. Functional Requirements

- **Limit the request rate** for a given key (user, IP, API key) to a configured threshold over a configured time window.
- **Reject requests** that exceed the threshold, returning a clear, machine-readable rejection response rather than silently dropping or slowing them.
- **Support multiple simultaneous limit tiers** for the same key (e.g., 100 requests/second AND 5,000 requests/hour, both enforced at once).
- **Support pluggable partitioning keys** so the same limiting logic can apply per-user, per-IP, or per-API-key without duplicated implementations.
- **Work correctly across a fleet of many servers** handling the same client's traffic, not just on a single process.
- **Communicate the limiter's state to the client** (remaining quota, reset time) so well-behaved clients can self-throttle before being rejected.

---

# 8. Non-Functional Requirements

- **Low latency overhead**: the rate-limit check itself must add negligible latency to the request path (sub-millisecond for a local check; low-single-digit milliseconds for a distributed check).
- **High availability**: the rate limiter itself must never become a single point of failure that takes down the entire service if it (or its backing store) becomes unavailable.
- **Bounded memory**: per-key limiter state must not grow unboundedly with the number of distinct keys or the length of the configured window.
- **Correctness under concurrency**: many concurrent requests for the same key must never race into an inconsistent decision (e.g., all being allowed simultaneously when only one should be).
- **Horizontal scalability**: adding more application servers must not require re-architecting the rate-limiting mechanism, nor should it let the effective aggregate limit silently balloon (each server enforcing the full limit independently).

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Name the core domain concepts before you draw any architecture."*

- **Rate Limit Rule** — a configured threshold (`maxRequests`) over a configured window (`windowDuration`), attached to a specific key type (per-user, per-IP, ...).
- **Key** — the specific identity being limited (a concrete user ID, IP address, or API key value) — the same rule can apply to millions of distinct keys, each with independent state.
- **Limiter State** — the actual mutable data a specific algorithm needs to track for one key (a counter and window boundary; a token count and last-refill time; a log of recent timestamps — the shape varies per algorithm, §15).
- **Decision** — the result of evaluating one request against a rule: allowed or rejected, plus enough metadata (remaining quota, reset time) for the client-facing contract (§46-47).

---

# 10. Identifying the Core Domain Entities

```java
public record RateLimitRule(String name, long maxRequests, Duration window) { }

public record RateLimitDecision(boolean allowed, long remaining, Instant resetAt, Duration retryAfter) {
    public static RateLimitDecision allow(long remaining, Instant resetAt) {
        return new RateLimitDecision(true, remaining, resetAt, Duration.ZERO);
    }
    public static RateLimitDecision reject(Instant resetAt, Duration retryAfter) {
        return new RateLimitDecision(false, 0, resetAt, retryAfter);
    }
}

public interface RateLimiter {
    RateLimitDecision tryAcquire(String key, RateLimitRule rule);
}
```

Notice `RateLimitDecision` carries not just a boolean but `remaining`, `resetAt`, and `retryAfter` — this is deliberate, and directly required by §46-47's client-facing contract: a rate limiter that only answers "yes or no" forces every rejected client to guess when it's safe to retry, which is precisely the design gap that produces uncoordinated retry storms.

---

# 11. High-Level Architecture Overview

```text
                    +------------------------+
Client requests --->|  Rate Limiter Middleware|
                    +-----------+------------+
                                |
                    +-----------v------------+
                    |   RateLimiter (Strategy)|
                    |   tryAcquire(key, rule) |
                    +-----------+------------+
                                |
                +---------------+----------------+
                v                                 v
      +------------------+             +------------------------+
      | Local in-memory   |             |  Shared store (Redis)  |
      | state (single      |             |  -- required once      |
      |  server only)      |             |  MULTIPLE servers share|
      +------------------+             |  the same client's traffic|
                                        +------------------------+
                                                |
                                    (rejected) v
                                    429 + Retry-After + X-RateLimit-* headers
```

Two deployment shapes share the exact same `RateLimiter` interface: a single-server deployment can use purely local, in-memory state (§17-30 build several algorithms this way), while a multi-server deployment needs that same decision logic backed by a shared store so every server sees a consistent view of one client's consumed quota (§32-39).

---

# 12. Follow-up Question 2 — "Why Not Just Count Requests in a Database Row and Check Before Every Request?"

> **Interviewer:** *"Simplest possible design: a database row per key holding a request count, incremented on every request, checked against the limit. Why doesn't that work?"*

Because a rate limiter sits directly in the request's hot path, and this naive design pays a full database round-trip (with its own locking/contention under concurrent writes to the same row) on **every single request**, for every client — at any meaningful traffic volume, the rate limiter itself would become slower and less reliable than the backend service it's meant to protect, and a spike in traffic (the exact scenario rate limiting exists to survive) is precisely when this design's single hot row becomes a serialization bottleneck.

---

# 13. Why a Naive DB-Counter-Per-Request Doesn't Scale

```text
Naive:  request -> BEGIN TRANSACTION -> SELECT count FROM limits WHERE key=X FOR UPDATE
                 -> if count < threshold: UPDATE count = count+1; COMMIT; ALLOW
                 -> else: COMMIT; REJECT
        -- EVERY request pays a full DB round-trip AND a row-level lock, contended by
           every OTHER concurrent request for the SAME key (exactly the popular/hot keys
           a rate limiter most needs to handle gracefully)

This guide's approach:  keep limiter state in FAST, low-latency storage purpose-built for
        this access pattern (in-memory for single-server, Redis for distributed) --
        the same principle the Real-Time Analytics Platform guide's §12-13 used to justify
        moving aggregation off a query-time database scan entirely.
```

The fix is architecturally identical to a decision this series has made before: move frequently-updated, latency-sensitive state out of a general-purpose relational database and into storage purpose-built for the specific access pattern (fast key-based reads and atomic increments), reserving the database for what it's actually good at.

---

# 14. Follow-up Question 3 — "What Are the Actual Algorithm Options, and Their Tradeoffs?"

> **Interviewer:** *"There isn't just one way to implement 'count requests per window.' Walk me through the real algorithm choices."*

Five algorithms cover essentially every real rate-limiting need, each trading off memory, precision, and burst-handling behavior differently — and the right choice genuinely depends on the specific requirement (is a burst at a window boundary acceptable? is exact precision worth the memory cost? does the traffic pattern need smoothing, not just capping?).

---

# 15. Algorithm Survey: Fixed Window, Sliding Window, Token Bucket, Leaky Bucket

```text
Fixed Window Counter     -- simplest: a counter per fixed time bucket, reset at each boundary.
                            CHEAP (one counter per key) but allows a 2x burst at window edges (§16-17).

Sliding Window Log       -- exact: stores every request's timestamp, counts how many fall within
                            the trailing window. PRECISE but memory scales with request volume (§19-21).

Sliding Window Counter   -- approximate: weights the previous window's count by how much of it
                            still overlaps the current sliding window. CHEAP AND reasonably
                            accurate -- the practical middle ground (§22-24).

Token Bucket             -- a bucket refills at a steady rate, requests consume tokens; ALLOWS
                            CONTROLLED BURSTS up to the bucket's capacity (§25-27).

Leaky Bucket             -- requests queue and are processed (leak out) at a steady, FIXED rate,
                            smoothing bursty input into steady output (§28-30).
```

Fixed window and sliding window variants all answer "how many requests occurred in this time span," a **counting** question; token bucket and leaky bucket instead model a **rate**, with genuinely different burst-handling behavior — this distinction is why interviewers press specifically on token-vs-leaky-bucket (§28), since the two are often conflated despite behaving differently under bursty load.

---

# 16. Follow-up Question 4 — "Fixed Window Has a Boundary Burst Problem — Explain and Fix It"

> **Interviewer:** *"A fixed window of '100 requests per minute' resets its counter every minute. Where's the bug, precisely?"*

A client can send 100 requests in the last millisecond of one window, and then another 100 requests in the first millisecond of the very next window — both individually within the 100/minute limit, but **200 requests landed within a single, effectively 2-millisecond span**, a burst the limit was specifically meant to prevent. The bug isn't in the counting logic itself, it's in treating "requests per window" as equivalent to "requests per any rolling time span," when a fixed window only actually bounds the former.

---

# 17. The Fixed-Window Boundary Burst Problem

```text
Window 1: [00:00 - 00:01)         Window 2: [00:01 - 00:02)
  ...................|9999999999][9999999999|...................
                       ^ 100 requests in the last 1ms of window 1
                                  ^ 100 requests in the first 1ms of window 2
  Both windows report "100/100, within limit" independently -- but a 2ms SLIDING
  span centered on the boundary actually saw 200 requests, double the intended limit.
```

This diagram is the concrete justification for every algorithm from §19 onward: each one is, in some form, an answer to "how do we bound requests over a genuinely *rolling* window, not just a fixed one that resets on a clock tick."

---

# 18. Implementing Fixed Window Counter

```java
public class FixedWindowCounter implements RateLimiter {
    private final Map<String, WindowState> stateByKey = new ConcurrentHashMap<>();

    private static class WindowState {
        volatile long windowStartMillis;
        final AtomicLong count = new AtomicLong(0);
    }

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        long now = System.currentTimeMillis();
        long windowSizeMillis = rule.window().toMillis();
        WindowState state = stateByKey.computeIfAbsent(key, k -> new WindowState());

        synchronized (state) {
            long currentWindowStart = (now / windowSizeMillis) * windowSizeMillis;
            if (state.windowStartMillis != currentWindowStart) {
                state.windowStartMillis = currentWindowStart; // new window -- reset the counter
                state.count.set(0);
            }
            long newCount = state.count.incrementAndGet();
            Instant resetAt = Instant.ofEpochMilli(currentWindowStart + windowSizeMillis);
            if (newCount > rule.maxRequests()) {
                return RateLimitDecision.reject(resetAt, Duration.between(Instant.now(), resetAt));
            }
            return RateLimitDecision.allow(rule.maxRequests() - newCount, resetAt);
        }
    }
}
```

Cheap and simple — exactly one counter and one timestamp per key, regardless of request volume — which is precisely why it remains a reasonable choice whenever the boundary-burst behavior from §16-17 is acceptable for the specific use case (e.g., a generous, coarse quota where a brief 2x burst at a boundary genuinely doesn't matter).

---

# 19. Follow-up Question 5 — "How Does Sliding Window Log Fix This, and What Does It Cost?"

> **Interviewer:** *"Design a version that genuinely enforces 'at most N requests in ANY trailing window,' with no boundary loophole."*

By storing the **actual timestamp of every request** for a given key, and, on each new request, counting how many stored timestamps fall within the trailing window ending *now* — discarding anything older. This is exact by construction (it directly answers "how many requests occurred in the last N seconds, as of this exact instant," with zero approximation), but its memory cost scales with **request volume within the window**, not with a fixed per-key constant, which is a real cost for a high-traffic key.

---

# 20. Sliding Window Log: Exact but Memory-Heavy

```text
Key "user-42"'s log:  [t-58s, t-45s, t-30s, t-12s, t-3s, t-1s]   (window = 60s, now = t)
On a new request at time t:
  1. Discard entries older than (t - 60s)  ->  removes t-58s if it's now outside the window
  2. Count remaining entries               ->  5 remain
  3. If count < limit: append t, ALLOW.  Else: REJECT.

Memory cost: O(requests within the window) PER KEY -- a key legitimately sending 10,000
requests within its window needs a log of 10,000 timestamps, not a small fixed structure.
```

The exactness here is genuinely valuable for use cases where even a small approximation error is unacceptable (e.g., a strict billing-relevant quota) — but for the overwhelming majority of rate-limiting use cases (protecting a backend from overload), this guide's next algorithm gets nearly the same precision at a small, fixed memory cost per key, which is almost always the better tradeoff.

---

# 21. Implementing Sliding Window Log

```java
public class SlidingWindowLog implements RateLimiter {
    private final Map<String, Deque<Long>> logsByKey = new ConcurrentHashMap<>();

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        long now = System.currentTimeMillis();
        long windowStart = now - rule.window().toMillis();
        Deque<Long> log = logsByKey.computeIfAbsent(key, k -> new ConcurrentLinkedDeque<>());

        synchronized (log) {
            while (!log.isEmpty() && log.peekFirst() < windowStart) {
                log.pollFirst(); // discard entries that have aged out of the trailing window
            }
            if (log.size() >= rule.maxRequests()) {
                long oldestRelevant = log.peekFirst();
                Instant resetAt = Instant.ofEpochMilli(oldestRelevant + rule.window().toMillis());
                return RateLimitDecision.reject(resetAt, Duration.between(Instant.now(), resetAt));
            }
            log.addLast(now);
            return RateLimitDecision.allow(rule.maxRequests() - log.size(), Instant.ofEpochMilli(now + rule.window().toMillis()));
        }
    }
}
```

The `while` loop discarding aged-out entries runs on every single request, which means the log's size stays bounded by *actual recent traffic volume*, never growing without bound over the application's lifetime — but it does mean a key sustaining traffic right at its limit permanently carries a log sized to that limit, which is exactly the memory cost §22 is designed to avoid.

---

# 22. Follow-up Question 6 — "How Do You Get Sliding-Window Accuracy Without Storing Every Timestamp?"

> **Interviewer:** *"Can you get most of sliding window log's precision with fixed-window counter's memory cost?"*

Yes, via a **weighted interpolation approximation**: keep two fixed-window counters (the current window and the immediately preceding one), and estimate the trailing window's true count as a weighted blend of the two — weighting the previous window's count by *how much of the current sliding window still overlaps it*. This trades a small, well-understood approximation error for collapsing memory back down to two counters per key, regardless of request volume — the same "bounded approximation instead of unbounded exactness" tradeoff that motivated HyperLogLog and t-digest in the Real-Time Analytics Platform guide.

---

# 23. Sliding Window Counter: Weighted Interpolation Approximation

```text
Previous window [00:00-00:01): 80 requests        Current window [00:01-00:02): 20 requests so far
Now = 00:01:15  (25% into the current window, so 75% of the PREVIOUS window still "overlaps"
                 the trailing 60-second span ending now)

Estimated count in the trailing 60s = (previous_count * overlap_fraction) + current_count
                                     = (80 * 0.75) + 20
                                     = 60 + 20 = 80

This estimate assumes requests were EVENLY distributed within the previous window -- a real,
documented approximation, not an exact count -- but it's close enough for nearly every real
rate-limiting use case, at a FIXED memory cost of two counters, independent of traffic volume.
```

The evenly-distributed-traffic assumption is the one honest caveat of this algorithm — a client that concentrates all of the previous window's 80 requests into its very last millisecond (rather than spreading them evenly) would make this estimate slightly optimistic — but this failure mode is a bounded, rare edge case, not the routine boundary-doubling failure fixed-window counter has by design (§16-17).

---

# 24. Implementing Sliding Window Counter

```java
public class SlidingWindowCounter implements RateLimiter {
    private final Map<String, TwoWindowState> stateByKey = new ConcurrentHashMap<>();

    private static class TwoWindowState {
        volatile long currentWindowStart;
        final AtomicLong currentCount = new AtomicLong(0);
        volatile long previousCount = 0;
    }

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        long now = System.currentTimeMillis();
        long windowSizeMillis = rule.window().toMillis();
        TwoWindowState state = stateByKey.computeIfAbsent(key, k -> new TwoWindowState());

        synchronized (state) {
            long currentWindowStart = (now / windowSizeMillis) * windowSizeMillis;
            if (state.currentWindowStart != currentWindowStart) {
                boolean isConsecutiveWindow = currentWindowStart - state.currentWindowStart == windowSizeMillis;
                state.previousCount = isConsecutiveWindow ? state.currentCount.get() : 0; // gap -> previous data is stale, discard
                state.currentWindowStart = currentWindowStart;
                state.currentCount.set(0);
            }

            double elapsedFraction = (now - currentWindowStart) / (double) windowSizeMillis;
            double overlapFraction = 1.0 - elapsedFraction;
            double estimatedCount = (state.previousCount * overlapFraction) + state.currentCount.get();

            if (estimatedCount >= rule.maxRequests()) {
                Instant resetAt = Instant.ofEpochMilli(currentWindowStart + windowSizeMillis);
                return RateLimitDecision.reject(resetAt, Duration.between(Instant.now(), resetAt));
            }
            long newCount = state.currentCount.incrementAndGet();
            Instant resetAt = Instant.ofEpochMilli(currentWindowStart + windowSizeMillis);
            return RateLimitDecision.allow((long) (rule.maxRequests() - estimatedCount - 1), resetAt);
        }
    }
}
```

The `isConsecutiveWindow` check handles a subtlety easy to miss: if a key has been idle long enough that the "previous" window isn't actually the window immediately before the current one (a gap of true inactivity), its stale count must be discarded as zero rather than incorrectly blended in as if it were recent traffic.

---

# 25. Follow-up Question 7 — "What Is Token Bucket, and Why Do Production Systems Prefer It?"

> **Interviewer:** *"AWS, Stripe, and most real API gateways default to token bucket rather than any of the counting algorithms you've described. Why?"*

Because every algorithm so far answers a **counting** question ("how many requests occurred"), while real traffic often has a legitimate need for **controlled burstiness** — a client that's been idle and then needs to send a quick burst of requests (e.g., loading a dashboard's ten widgets at once) shouldn't necessarily be capped as tightly as one sustaining continuous traffic. **Token bucket** models this directly: a bucket holds up to a fixed capacity of tokens, refilled at a steady rate; a request consumes one token if available. A client that's been idle has a full bucket and can burst up to that capacity instantly, while sustained traffic is still capped at the steady refill rate — this is a genuinely different, often more desirable, behavior than any of §17-24's algorithms provide.

---

# 26. Token Bucket: Allowing Controlled Bursts

```text
Bucket capacity: 10 tokens.  Refill rate: 1 token/second.

t=0:   bucket has 10 tokens (been idle, fully refilled)
t=0:   client sends 10 requests instantly -- ALL ALLOWED (burst consumes all 10 tokens at once)
t=0:   bucket now has 0 tokens
t=1:   1 token has refilled -- 1 more request can be allowed
t=1.5: client sends a request -- ALLOWED (consumes the 1 refilled token)
t=1.6: client sends another request -- REJECTED (bucket is empty again, refill hasn't caught up)
```

The capacity parameter and the refill-rate parameter are two genuinely independent knobs: capacity controls **how large a burst** is tolerated, refill rate controls the **long-run sustained** throughput — a system can tune these separately to match its actual tolerance for burstiness versus its actual backend capacity, which is exactly the flexibility fixed/sliding window counting doesn't offer.

---

# 27. Implementing Token Bucket

```java
public class TokenBucket implements RateLimiter {
    private final Map<String, BucketState> stateByKey = new ConcurrentHashMap<>();

    private static class BucketState {
        double tokens;
        long lastRefillMillis;
    }

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        long now = System.currentTimeMillis();
        double capacity = rule.maxRequests();
        double refillPerMilli = capacity / rule.window().toMillis(); // tokens/ms, derived from the rule

        BucketState state = stateByKey.computeIfAbsent(key, k -> {
            BucketState s = new BucketState();
            s.tokens = capacity; // a brand-new key starts with a FULL bucket
            s.lastRefillMillis = now;
            return s;
        });

        synchronized (state) {
            long elapsed = now - state.lastRefillMillis;
            state.tokens = Math.min(capacity, state.tokens + elapsed * refillPerMilli); // refill, capped at capacity
            state.lastRefillMillis = now;

            if (state.tokens >= 1.0) {
                state.tokens -= 1.0;
                return RateLimitDecision.allow((long) state.tokens, Instant.ofEpochMilli(now));
            }
            long millisUntilNextToken = (long) ((1.0 - state.tokens) / refillPerMilli);
            return RateLimitDecision.reject(Instant.ofEpochMilli(now + millisUntilNextToken), Duration.ofMillis(millisUntilNextToken));
        }
    }
}
```

Refilling **lazily** (computing how many tokens should have accumulated based on elapsed time, only when a request actually arrives) rather than via a separate periodically-running timer is the standard, more efficient implementation — it avoids the cost of a background thread waking up for every key on every tick, entirely regardless of whether that key has any traffic at all.

---

# 28. Follow-up Question 8 — "What's Leaky Bucket, and How Is It Actually Different from Token Bucket?"

> **Interviewer:** *"Leaky bucket and token bucket have suspiciously similar-sounding descriptions. What's the actual behavioral difference?"*

The names describe two genuinely different mental models, despite the similarity: **token bucket** controls the rate of **admission** — it decides whether to let a request *in*, and once admitted, the request proceeds immediately (bursts pass straight through, up to bucket capacity). **Leaky bucket** instead controls the rate of **processing/output** — incoming requests queue up (up to a bounded queue size), and are processed (leak out) at a strictly constant rate, regardless of how bursty their arrival was. Token bucket answers "how many requests can I admit right now"; leaky bucket answers "at what steady rate do admitted requests get processed" — the former can produce bursty *downstream* traffic, the latter, by construction, cannot.

---

# 29. Leaky Bucket: Smoothing Output Rate

```text
Token Bucket (admission control):     10 requests arrive instantly -> ALL 10 processed IMMEDIATELY
                                       (downstream sees a burst of 10 at once)

Leaky Bucket (output smoothing):      10 requests arrive instantly -> queued, then processed
                                       ONE AT A TIME at the fixed leak rate (e.g., 1/second)
                                       (downstream sees a smooth, steady trickle of 1/second,
                                        even though they all arrived in a burst)
```

This is precisely why leaky bucket is the better fit when the actual goal is protecting a downstream system that genuinely cannot handle bursty load at all (even a brief one) — e.g., smoothing requests before they reach a rate-sensitive legacy system — while token bucket is the better fit when the goal is simply capping total volume while still tolerating bursts the downstream system can actually absorb.

---

# 30. Implementing Leaky Bucket

```java
public class LeakyBucket implements RateLimiter {
    private final Map<String, LeakyBucketState> stateByKey = new ConcurrentHashMap<>();

    private static class LeakyBucketState {
        double queueLevel;      // how much "water" (queued requests) is currently in the bucket
        long lastLeakMillis;
    }

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        long now = System.currentTimeMillis();
        double capacity = rule.maxRequests();          // max QUEUE depth, not a burst allowance
        double leakPerMilli = capacity / rule.window().toMillis();

        LeakyBucketState state = stateByKey.computeIfAbsent(key, k -> {
            LeakyBucketState s = new LeakyBucketState();
            s.lastLeakMillis = now;
            return s;
        });

        synchronized (state) {
            long elapsed = now - state.lastLeakMillis;
            state.queueLevel = Math.max(0, state.queueLevel - elapsed * leakPerMilli); // leak out at the fixed rate
            state.lastLeakMillis = now;

            if (state.queueLevel + 1.0 <= capacity) {
                state.queueLevel += 1.0; // this request is added to the queue, to be processed at the steady leak rate
                return RateLimitDecision.allow((long) (capacity - state.queueLevel), Instant.ofEpochMilli(now));
            }
            return RateLimitDecision.reject(Instant.ofEpochMilli(now), rule.window()); // queue is full -- reject outright
        }
    }
}
```

Note the structural similarity to `TokenBucket` (§27) — both lazily compute an elapsed-time-based adjustment on each call — but observe the inverted semantics: token bucket's `tokens` *decreases* toward zero as requests are admitted and *increases* via refill, while leaky bucket's `queueLevel` *increases* as requests queue up and *decreases* via leaking — the same lazy-computation implementation technique, applied to two conceptually opposite quantities.

---

# 31. Class Diagram: The Rate Limiting Algorithm Core

```text
+------------------------+
|      RateLimiter         |
|    <<interface>>         |
|  + tryAcquire(key, rule) |
+-----------+--------------+
            ^
   +--------+--------+--------+--------+--------+
   |        |        |        |        |
+--------+ +--------+ +----------+ +--------+ +--------+
| Fixed  | | Sliding| | Sliding  | | Token  | | Leaky  |
| Window | | Window | | Window   | | Bucket | | Bucket |
| Counter| | Log    | | Counter  | |        | |        |
+--------+ +--------+ +----------+ +--------+ +--------+

                    +------------------------+
                    |     RateLimitRule        |
                    |  maxRequests, window     |
                    +------------------------+
                    +------------------------+
                    |     RateLimitDecision    |
                    |  allowed, remaining,     |
                    |  resetAt, retryAfter     |
                    +------------------------+
```

All five algorithms implement the identical `RateLimiter` interface, which is the textbook **Strategy** pattern applied directly: the middleware calling `tryAcquire(key, rule)` never needs to know or care which specific algorithm is plugged in for a given rule — swapping fixed window for token bucket in a configuration file requires zero changes to any calling code.

---

# 32. Follow-up Question 9 — "This Works on One Server. How Do You Rate-Limit Consistently Across Many Servers Sharing the Same Client?"

> **Interviewer:** *"Your service runs behind a load balancer, across 10 servers. A single client's requests get distributed across all 10. Every implementation so far keeps state in that one process's memory. What breaks?"*

Each of the 10 servers independently tracks its own local view of the client's request count — meaning a client limited to "100 requests/second" could actually send up to **1,000 requests/second** (100 against each of the 10 servers) before any single server's local limiter would reject anything, since none of them can see the other nine's counts. Enforcing one true, shared limit across a fleet requires moving the limiter's *state* (not its algorithm) into a **shared store** every server reads from and writes to.

---

# 33. The Distributed Coordination Problem

```text
WITHOUT shared state:
  Client -> Load Balancer -> distributes requests round-robin across 10 servers
  Server 1's local TokenBucket: 100/sec limit, sees 1/10th of the client's actual traffic
  Server 2's local TokenBucket: 100/sec limit, sees another 1/10th
  ... each of the 10 servers independently allows up to 100/sec ->  EFFECTIVE limit = 1000/sec

WITH shared state (Redis):
  All 10 servers read/write the SAME bucket state for this client, stored in Redis --
  the effective limit is genuinely 100/sec, TOTAL, across the entire fleet, because there
  is only ONE bucket, not ten independent ones.
```

This is a direct instance of the exact same principle the Real-Time Analytics Platform guide's partitioning (§15 there) and the Logging guide's `ThreadLocal` MDC (§29-31 there) both had to grapple with: state that needs to be consistent across multiple independent execution contexts (multiple servers, here) cannot live purely locally in any one of them — it has to live somewhere all of them can see.

---

# 34. Centralized Store via Redis: Atomic Check-and-Decrement

Redis is the standard choice for this shared state: it offers sub-millisecond round-trip latency (critical since this check now sits directly in every request's hot path across the fleet), and — critically — it supports **atomic, server-side scripted operations** via `EVAL` (Lua scripting), which is the mechanism that closes a race condition a naive "read the count, check it, write the updated count" sequence would otherwise have under concurrent requests from multiple servers hitting the same key simultaneously.

---

# 35. Implementing a Redis-Backed Token Bucket with a Lua Script

```text
-- token_bucket.lua (executed ATOMICALLY, server-side, by Redis -- no other command can
-- interleave between this script's reads and writes, closing the race condition entirely)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillPerMilli = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call("HMGET", key, "tokens", "lastRefill")
local tokens = tonumber(bucket[1]) or capacity
local lastRefill = tonumber(bucket[2]) or now

local elapsed = now - lastRefill
tokens = math.min(capacity, tokens + elapsed * refillPerMilli)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call("HMSET", key, "tokens", tokens, "lastRefill", now)
    redis.call("EXPIRE", key, 3600)  -- bound memory: idle keys eventually expire from Redis entirely
    return 1  -- ALLOWED
else
    redis.call("HMSET", key, "tokens", tokens, "lastRefill", now)
    return 0  -- REJECTED
end
```

```java
public class RedisTokenBucket implements RateLimiter {
    private final RedisClient redisClient;
    private final String luaScriptSha; // pre-loaded once via SCRIPT LOAD, invoked by SHA thereafter (EVALSHA)

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        double refillPerMilli = rule.maxRequests() / (double) rule.window().toMillis();
        long result = redisClient.evalSha(luaScriptSha,
            List.of(key), List.of(String.valueOf(rule.maxRequests()), String.valueOf(refillPerMilli),
                String.valueOf(System.currentTimeMillis())));
        return result == 1
            ? RateLimitDecision.allow(-1, Instant.now())   // exact remaining count omitted here for brevity
            : RateLimitDecision.reject(Instant.now(), rule.window());
    }
}
```

The entire read-modify-write sequence (fetch current state, compute refill, decide, write updated state) executes as **one atomic Redis operation** — no other server's concurrent request against the same key can interleave partway through, which is exactly the property a separate `GET`-then-`SET` from application code could never guarantee under concurrent access from multiple servers.

---

# 36. Follow-up Question 10 — "A Single Redis Call Per Request Adds Latency. How Do You Reduce That?"

> **Interviewer:** *"Even at sub-millisecond Redis latency, a network round-trip on every single request adds up across millions of requests. Can you avoid making one every time?"*

By having each server **locally cache a small allotment of tokens**, refilling that local cache from Redis only periodically (e.g., every 100ms, or once the local allotment is exhausted) rather than on every single request — trading a small amount of precision (a server could, briefly, allow slightly more than its exact fair share if it just refilled its local cache right before traffic dropped) for a large reduction in Redis round-trips, since most requests are now served from a fast, purely local decrement.

---

# 37. Client-Side Batching to Reduce Redis Round-Trips

```text
Instead of: EVERY request -> one Redis call

Each server periodically "leases" a batch from Redis, e.g.:
  Server 1: EVALSHA ... -> "you're granted 20 tokens, valid for the next 100ms"
  Server 1 then serves up to 20 LOCAL requests from this leased batch, ZERO Redis calls needed
  After 100ms (or the batch is exhausted), Server 1 requests a fresh batch from Redis

Tradeoff: the GLOBAL limit is now enforced with a small, bounded slop (up to one batch's
worth of over-allowance, fleet-wide, in the worst case) -- in exchange for cutting Redis
round-trips by roughly the batch size (a 20x reduction, in this example).
```

This is precisely the same amortization principle the Unique ID Generator guide's §37-38 used (batch-leasing machine-ID sequence ranges to avoid a network round-trip per generated ID) — applied here to rate-limit quota instead of ID ranges, because both problems share the same shape: a scarce, globally-coordinated resource, and a latency-sensitive per-request path that shouldn't have to pay a network round-trip every single time.

---

# 38. Follow-up Question 11 — "What Happens If Redis Itself Becomes a Bottleneck or Goes Down?"

> **Interviewer:** *"The whole distributed design now depends on Redis being available and fast. What's the failure mode if it isn't?"*

Two distinct concerns, requiring two distinct answers: if Redis is merely **slow** (not down), the batching from §37 already limits how often it's even consulted, bounding the blast radius of added latency. If Redis is genuinely **unavailable**, the correct behavior is an explicit, deliberate policy decision — either **fail open** (allow all requests through, accepting a temporary loss of rate limiting, prioritizing availability) or **fail closed** (reject all requests, prioritizing strict quota enforcement over availability) — and most production systems choose a middle ground: fail open, but fall back to a conservative **local-only** limiter during the outage, so the backend isn't left completely unprotected even without global coordination.

---

# 39. Fault Tolerance: Local Fallback Limiter and Redis Sharding

```java
public class LocalFallbackLimiter implements RateLimiter {
    private final RateLimiter distributedLimiter;   // RedisTokenBucket, the normal path
    private final RateLimiter localFallbackLimiter; // e.g. a plain TokenBucket (§27), per-server only
    private final CircuitBreaker redisCircuitBreaker;

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule rule) {
        if (redisCircuitBreaker.isOpen()) {
            // Redis is currently considered unhealthy -- fall back to a conservative, PER-SERVER
            // limit rather than either fully disabling limiting or fully blocking all traffic
            return localFallbackLimiter.tryAcquire(key, rule.scaledDownFor(estimatedFleetSize()));
        }
        try {
            return distributedLimiter.tryAcquire(key, rule);
        } catch (RedisConnectionException e) {
            redisCircuitBreaker.recordFailure();
            return localFallbackLimiter.tryAcquire(key, rule.scaledDownFor(estimatedFleetSize()));
        }
    }
}
```

Scaling the fallback rule down by the estimated fleet size (e.g., a 100/sec global limit becomes a 10/sec local limit across 10 servers) is the detail that keeps the fallback's *aggregate* behavior close to the intended global limit, rather than accidentally re-introducing §33's original "each server independently allows the full limit" problem during exactly the outage window when disciplined behavior matters most.

---

# 40. Follow-up Question 12 — "How Do You Support Multiple Simultaneous Limits, Like 100/Second AND 1,000/Hour?"

> **Interviewer:** *"A real API often needs a tight burst limit AND a looser sustained limit, enforced together — e.g., 100 requests/second, but also no more than 1,000/hour even if spread out. How do you avoid duplicating the whole limiter for every combination of tiers?"*

By treating "a rate limit" as potentially **several independent rules evaluated together**, all of which must pass for a request to be allowed — a request is only admitted if it satisfies *every* configured tier simultaneously, and rejected (with the *most restrictive* tier's retry guidance) the moment any single tier's `tryAcquire` returns a rejection. Each tier is just an ordinary `RateLimitRule` evaluated against an ordinary `RateLimiter`, so no new algorithm is needed — only a thin composition layer over the existing ones.

---

# 41. Multi-Tier / Hierarchical Rate Limits

```text
Rule Set for "premium-tier-user":
  Tier 1: 100 requests / 1 second   (burst limit -- uses Token Bucket for burst tolerance)
  Tier 2: 1,000 requests / 1 hour   (sustained limit -- uses Sliding Window Counter)

A request is ALLOWED only if BOTH tiers independently allow it.
A request is REJECTED if EITHER tier rejects it -- using THAT tier's specific retryAfter,
since it's the binding constraint the client actually needs to respect.
```

Each tier can legitimately use a *different* underlying algorithm — a tight, short-window burst limit is a natural fit for token bucket (§25-27), while a loose, long-window sustained limit is a natural fit for sliding window counter's low memory cost (§22-24) — which is exactly why §31's `RateLimiter` Strategy interface matters here too: tiers are composed *across* algorithms, not locked into using the same one.

---

# 42. Implementing Composite Rate Limiting

```java
public class MultiTierRateLimiter implements RateLimiter {
    private final List<TieredRule> tiers; // each pairs a RateLimitRule with its OWN RateLimiter instance

    public record TieredRule(RateLimitRule rule, RateLimiter limiter) { }

    @Override
    public RateLimitDecision tryAcquire(String key, RateLimitRule ignoredTopLevelRule) {
        RateLimitDecision mostRestrictiveRejection = null;

        for (TieredRule tier : tiers) {
            RateLimitDecision decision = tier.limiter().tryAcquire(key, tier.rule());
            if (!decision.allowed()) {
                // a rejection from ANY tier rejects the whole request -- keep the one with
                // the LONGEST retryAfter, since that's the binding constraint the client must respect
                if (mostRestrictiveRejection == null || decision.retryAfter().compareTo(mostRestrictiveRejection.retryAfter()) > 0) {
                    mostRestrictiveRejection = decision;
                }
            }
        }
        return mostRestrictiveRejection != null ? mostRestrictiveRejection : RateLimitDecision.allow(-1, Instant.now());
    }
}
```

`MultiTierRateLimiter` is itself just another implementation of the `RateLimiter` interface — a **Composite** wrapping several other `RateLimiter` instances — which means it can be nested or swapped in anywhere a single limiter was expected, with zero change to any calling code, exactly the same payoff the Composite pattern already delivered for `CompositeAppender` in the Logging guide.

---

# 43. Follow-up Question 13 — "How Do You Rate-Limit by Different Keys — Per-User, Per-IP, Per-API-Key — Without Duplicating Logic?"

> **Interviewer:** *"The same tiered-limit logic needs to apply per-user for logged-in traffic, but per-IP for anonymous traffic. How do you avoid writing that logic twice?"*

By separating **what key to limit by** from **how to enforce the limit against that key** — the former becomes a small, pluggable `KeyResolver` (extract a user ID from the auth context, or fall back to the request's IP address if unauthenticated), while the latter stays exactly the `RateLimiter`/`MultiTierRateLimiter` machinery already built. The middleware calls `keyResolver.resolve(request)` once, then passes whatever key comes back into the same, unmodified rate-limiting pipeline.

---

# 44. Key Resolution Strategy: Pluggable Partitioning Keys

```java
public interface KeyResolver {
    String resolve(HttpRequest request);
}

public class UserOrIpKeyResolver implements KeyResolver {
    @Override
    public String resolve(HttpRequest request) {
        return request.authenticatedUserId()
            .map(userId -> "user:" + userId)
            .orElseGet(() -> "ip:" + request.remoteAddress());
    }
}

public class ApiKeyResolver implements KeyResolver {
    @Override
    public String resolve(HttpRequest request) {
        return "apikey:" + request.header("X-API-Key")
            .orElseThrow(() -> new MissingApiKeyException("Rate-limited endpoint requires an API key"));
    }
}
```

Prefixing each resolved key with its category (`"user:"`, `"ip:"`, `"apikey:"`) is a small but important detail — without it, a numeric user ID and a numerically-similar API key hash could theoretically collide into the same underlying limiter-state entry, silently sharing quota between two entirely unrelated identities.

---

# 45. Class Diagram: The Distributed Rate Limiter

```text
+------------------------+
|      KeyResolver         |
|    <<interface>>         |
|  + resolve(request)      |
+-----------+--------------+
            |
            v
+------------------------+        +------------------------+
| MultiTierRateLimiter     |------>|      TieredRule          |
| (Composite)               |       |  RateLimitRule +         |
| tryAcquire(key, rule)     |       |  RateLimiter (per tier)  |
+-----------+--------------+        +-----------+--------------+
                                                  |
                                                  v
                                    +------------------------+
                                    | LocalFallbackLimiter     |
                                    |  (Decorator over the      |
                                    |   distributed limiter,   |
                                    |   circuit-breaker gated) |
                                    +-----------+--------------+
                                                  |
                                    +-------------+-------------+
                                    v                             v
                        +------------------+          +------------------------+
                        | RedisTokenBucket  |          | Local RateLimiter        |
                        | (distributed)     |          | (fallback, any §31       |
                        +------------------+          |  algorithm)              |
                                                       +------------------------+
```

`LocalFallbackLimiter` is best understood as a **Decorator** over `RedisTokenBucket`: it implements the exact same `RateLimiter` interface, adds circuit-breaker-gated fallback behavior around it, and is otherwise indistinguishable from the limiter it wraps to anything calling it — precisely the same relationship `CompositeAppender` (Logging guide) has to the individual appenders it fans out to.

---

# 46. Follow-up Question 14 — "What Should the Response Look Like When a Client Is Rate-Limited?"

> **Interviewer:** *"A request gets rejected. What status code, and what does the response body/headers actually communicate to the client?"*

**`429 Too Many Requests`**, always accompanied by a **`Retry-After`** header (seconds until it's safe to retry) — without it, a poorly-behaved client will likely retry immediately, worsening exactly the overload condition the rejection exists to prevent. Well-designed rate limiters additionally expose `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` on **every** response, not just rejections, so a well-behaved client can self-throttle *before* ever being rejected at all — the same header convention this series' REST API guide already established in its own §47.

---

# 47. The Rejection Contract: 429, Retry-After, and Rate-Limit Headers

```java
public class RateLimitResponseWriter {
    public void writeDecision(HttpResponse response, RateLimitDecision decision, RateLimitRule rule) {
        response.setHeader("X-RateLimit-Limit", String.valueOf(rule.maxRequests()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(Math.max(0, decision.remaining())));
        response.setHeader("X-RateLimit-Reset", String.valueOf(decision.resetAt().getEpochSecond()));

        if (!decision.allowed()) {
            response.setStatus(429);
            response.setHeader("Retry-After", String.valueOf(decision.retryAfter().toSeconds()));
            response.setBody(new ApiError("RATE_LIMITED", "Too many requests. Retry after "
                + decision.retryAfter().toSeconds() + " seconds."));
        }
    }
}
```

Writing the `X-RateLimit-*` headers **before** checking whether the request was actually allowed is deliberate — every response, allowed or rejected, carries the same quota-visibility headers, giving a well-behaved client the information it needs to self-regulate on every single call, not merely learn about the limit retroactively once it's already been rejected.

---

# 48. Follow-up Question 15 — "How Do You Avoid a Thundering Herd of Retries All Hitting at Once After Being Rejected?"

> **Interviewer:** *"Retry-After tells one client when to retry. But if a thousand clients were all rejected at the same instant with the same Retry-After value, don't they all just retry simultaneously and immediately get rejected again?"*

Exactly — a single, precise `Retry-After` value shared identically across every rejected client just relocates the burst to a later, equally synchronized instant. The fix is **jitter**: each client adds a small, randomized delay on top of the server-provided `Retry-After` guidance, spreading what would otherwise be one synchronized retry spike into a smoothed distribution over a short window — this is the exact same insight behind exponential-backoff-with-jitter in any distributed retry strategy, applied here specifically to rate-limit rejections.

---

# 49. Jittered Backoff and the Client-Side Contract

```text
Naive client retry:  wait EXACTLY retryAfter seconds, then retry
                      -- every rejected client retries at the SAME instant -> thundering herd

Jittered client retry:  wait retryAfter + random(0, retryAfter * 0.5) seconds, then retry
                      -- rejected clients' retries are now SPREAD across a window,
                         not synchronized to a single instant
```

This is ultimately a **client-side** responsibility — the server can only provide the base `Retry-After` guidance; the server-side design's real contribution is *documenting this expectation clearly* (in the API's own docs, or via a client SDK the server team also owns) so that "add jitter before retrying" isn't left as an unstated assumption most client implementations will get wrong by default.

---

# 50. Capacity Estimation: Memory and Redis Throughput

```text
Assume: 5 million distinct active keys (users), each with a 2-tier limit (burst + sustained)

Local in-memory algorithm state (Token Bucket: ~40 bytes/key; Sliding Window Counter: ~32 bytes/key)
  Memory per server = 5,000,000 keys * 2 tiers * ~40 bytes  ≈ 400 MB -- comfortably fits in memory

Redis-backed distributed state (same key count, HASH per key: ~80 bytes overhead + fields)
  Redis memory ≈ 5,000,000 * 2 tiers * ~100 bytes ≈ 1 GB -- well within a single Redis instance's
  capacity; EXPIRE on idle keys (§35) keeps this from growing unboundedly as users churn

Redis round-trip budget: with client-side batching (§37, batch size 20), effective Redis QPS is
  (actual request QPS) / 20 -- a fleet handling 200,000 req/sec generates only ~10,000 Redis
  ops/sec, comfortably within a single well-provisioned Redis instance's real-world throughput.
```

The batching-driven QPS reduction from §37 is what keeps this design's Redis load tractable at real production scale — without it, a 200,000 req/sec fleet would need Redis to sustain 200,000 ops/sec purely for rate-limit checks, competing directly with whatever else Redis is used for in the same deployment.

---

# 51. Full Worked Example: One Request's Journey Through the Limiter, Traced

```text
1. Request arrives at Server 3 (one of 10 servers behind the load balancer), authenticated as user-42
2. UserOrIpKeyResolver.resolve(request) -> "user:42" (§44)
3. MultiTierRateLimiter.tryAcquire("user:42", ...) evaluates BOTH configured tiers (§41-42):
     a. Tier 1 (100/sec, Token Bucket, Redis-backed): LocalFallbackLimiter checks its circuit
        breaker -- Redis is healthy, so it delegates to RedisTokenBucket (§39)
     b. RedisTokenBucket calls EVALSHA with the pre-loaded Lua script (§35) -- atomically checks
        and decrements the shared bucket for "user:42" -- tokens remaining: ALLOWED
     c. Tier 2 (1,000/hour, Sliding Window Counter, Redis-backed): also ALLOWED
4. Since BOTH tiers allowed the request, MultiTierRateLimiter returns an overall ALLOW decision
5. RateLimitResponseWriter attaches X-RateLimit-Remaining/Reset headers to the eventual response,
   even though the request was allowed (§47)
6. The request proceeds to the backend service

--- A few seconds later, the SAME user's Tier 1 bucket is empty ---

7. A new request for "user:42" arrives at Server 7 (a DIFFERENT server) -- because the bucket
   state lives in shared Redis, not local memory, Server 7 sees the SAME depleted bucket
   Server 3's earlier requests drew down (§33-34) -- correctly REJECTED
8. RateLimitResponseWriter sets 429, Retry-After, and the rejection ApiError body (§47)
9. A well-behaved client waits retryAfter + jitter before its next attempt (§48-49)
```

Every mechanism introduced by a follow-up question in this guide appears somewhere in this one request's journey (and its later, cross-server rejection) — demonstrating concretely why the shared-store design from §32-39 was necessary: a purely local limiter on Server 7 would have no way to know Server 3 had already drawn the same user's bucket down to empty.

---

# 52. Final Architecture Diagram

```text
Client requests --> Load Balancer --> [Server 1 .. Server N]
                                            |
                                  KeyResolver.resolve(request)
                                            |
                                  MultiTierRateLimiter
                                    /                  \
                          Tier 1 (burst)          Tier 2 (sustained)
                                |                          |
                      LocalFallbackLimiter         LocalFallbackLimiter
                        /              \             /              \
              RedisTokenBucket   Local TokenBucket   ...            ...
              (normal path)      (circuit-open fallback)
                     |
              +--------------+
              |    Redis      |  <-- shared, atomic Lua-scripted state, EXPIRE-bounded memory
              +--------------+
                     |
              RateLimitResponseWriter --> 429 + Retry-After + X-RateLimit-* headers (rejected)
                                      --> pass-through + X-RateLimit-* headers (allowed)
```

---

# 53. Design Patterns Used Throughout This Guide

- **Strategy** — `RateLimiter` (§31: fixed window, sliding window log/counter, token bucket, leaky bucket, all interchangeable) and `KeyResolver` (§44: per-user, per-IP, per-API-key) are both swappable behaviors selected independently of the code that uses them.
- **Composite** — `MultiTierRateLimiter` (§42) treats a single rule and a fleet of simultaneously-enforced tiers identically, since both are just "something implementing `RateLimiter`."
- **Decorator** — `LocalFallbackLimiter` (§39, §45) wraps `RedisTokenBucket` with circuit-breaker-gated fallback behavior, without changing the interface either exposes to callers.
- **Facade** — the rate-limiter middleware itself hides key resolution, tiered evaluation, distributed coordination, and header-writing behind a single request-handling entry point application code never needs to look inside.
- **Circuit Breaker** — gating whether `LocalFallbackLimiter` attempts the distributed path or falls back locally, preventing repeated attempts against an already-known-unhealthy Redis from adding latency to every request during an outage.

---

# 54. SOLID Principles Applied

- **Single Responsibility** — `KeyResolver` only extracts a key; `RateLimiter` implementations only decide allow/reject for a key+rule; `RateLimitResponseWriter` only translates a decision into HTTP — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new algorithm (e.g., a future GCRA-based limiter) means implementing one interface, never modifying `MultiTierRateLimiter`'s composition logic or the middleware that calls it.
- **Liskov Substitution** — every `RateLimiter` implementation must honestly support `tryAcquire(key, rule)` with the same contract (a `RateLimitDecision`, never a thrown exception for an ordinary rejection), so `MultiTierRateLimiter` can compose any mix of them uniformly.
- **Interface Segregation** — `KeyResolver` exposes exactly one method, so a simple per-IP resolver isn't forced to implement irrelevant authentication-lookup logic a per-user resolver needs.
- **Dependency Inversion** — the middleware depends on the `RateLimiter` and `KeyResolver` abstractions, never on `RedisTokenBucket` or `UserOrIpKeyResolver` directly, so either can be swapped per deployment without touching the middleware itself.

---

# 55. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Fixed window counter, unaware of the boundary burst      Sliding window log/counter, or accept the
  problem for a use case that actually needs precision     tradeoff explicitly (§16-24)
Confusing token bucket and leaky bucket as interchangeable Understand admission control vs. output
                                                             smoothing are genuinely different (§28-29)
Rate limiting each server independently in a fleet        Shared store (Redis) for one true global
                                                             limit across all servers (§32-34)
A separate GET-then-SET against Redis for check+update    Atomic Lua-scripted check-and-decrement (§35)
One Redis round-trip per request at high fleet QPS         Client-side batch leasing (§37)
No plan for Redis becoming unavailable                     Circuit-breaker-gated local fallback (§38-39)
A single Retry-After value causing synchronized retries    Client-side jittered backoff (§48-49)
Duplicating limiter logic per partitioning key type        Pluggable KeyResolver strategy (§43-44)
```

---

# 56. Testing Strategy

- **Algorithm correctness tests** — for each `RateLimiter` implementation: assert the exact allow/reject boundary at the configured threshold, and specifically assert `FixedWindowCounter`'s known boundary-burst behavior (§16-17) is reproduced, not accidentally "fixed" by an implementation bug that would silently change its documented behavior.
- **Token bucket burst-allowance tests** — verify a fully-idle bucket allows exactly `capacity` requests instantly, then correctly throttles to the steady refill rate afterward.
- **Redis atomicity race tests** — fire many concurrent `tryAcquire` calls for the same key from multiple simulated clients against a real (or embedded-test) Redis instance, and assert the total allowed count never exceeds the configured limit, verifying the Lua script's atomicity actually holds under real concurrency.
- **Fallback behavior tests** — simulate a Redis outage (via the circuit breaker) and assert the system falls back to the scaled-down local limiter rather than either fully disabling limiting or fully rejecting all traffic.
- **Multi-tier composition tests** — configure two tiers with different limits and assert a request is rejected the instant *either* tier's threshold is exceeded, with the response carrying the more restrictive tier's `retryAfter`.

---

# 57. Suggested Future Enhancements

- **Generic Cell Rate Algorithm (GCRA)** — an alternative to token bucket that achieves equivalent behavior using a single timestamp per key instead of a token count, at a small further memory reduction.
- **Adaptive rate limits** — automatically tightening a client's limit temporarily following repeated rejections (suggesting abusive or misbehaving traffic) rather than a single statically-configured threshold for all traffic.
- **Cost-weighted requests** — allowing different endpoints to consume different amounts of quota per call (an expensive bulk-export endpoint costing more "tokens" than a cheap read), rather than treating every request as equally costly.
- **Per-tenant configurable limits** — exposing rate-limit configuration itself as tenant-configurable data (mirroring the Real-Time Analytics Platform guide's per-tenant quota design, §44-45 there), rather than a single global configuration for every client.
- **Geo-distributed Redis** — for a globally-distributed fleet, replacing a single Redis instance with a geo-replicated store, trading a small amount of cross-region consistency lag for lower latency in each region.

---

# 58. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Explain the fixed-window boundary burst problem precisely, with a concrete example. (§16-17)
2. Design an algorithm that fixes it, and explain the memory-vs-precision tradeoff of your fix compared to an exact solution. (§19-24)
3. Token bucket and leaky bucket sound similar. What's the actual behavioral difference, and when would you choose one over the other? (§28-29)
4. Your rate limiter runs on 10 servers behind a load balancer. Diagnose why a per-server limiter fails, and design the fix. (§32-35)
5. The shared store your fix depends on becomes unavailable. Design the failure-handling behavior. (§38-39)
6. Design support for two simultaneous limits (a burst tier and a sustained tier) without duplicating the underlying algorithm implementations. (§40-42)
7. What should a rejected client actually receive in the response, and why does a single `Retry-After` value alone remain insufficient? (§46-49)
8. How would you reduce Redis round-trip volume for a very high-QPS fleet without giving up global coordination entirely? (§36-37)

---

# 59. Final Takeaway

Every hard decision in this guide reduces to the same underlying tension, applied at a different layer each time: **a limit enforced independently, in isolation, is not the same limit enforced globally** — a fixed window enforces its limit only *within* a window, not across the boundary between windows (§16-17); a per-server limiter enforces its limit only *on that server*, not across the fleet (§32-33); a single, unjittered `Retry-After` enforces a retry delay only *in expectation*, not in practice, once many clients share it (§48-49). Recognizing which "boundary" a given design is only *locally* correct across — a time window, a server, a client population — and then deliberately closing that gap, is the transferable skill this guide is really teaching.

---
