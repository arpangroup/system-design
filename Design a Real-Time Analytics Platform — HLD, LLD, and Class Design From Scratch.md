# Design a Real-Time Analytics Platform — HLD, LLD, and Class Design From Scratch

# 1. What We Are Building

```text
Client SDKs / Apps
   |  (events: clicks, page views, purchases, custom metrics)
   v
[ Ingestion API ]  ->  [ Durable Log (Kafka-style) ]  ->  [ Stream Processing Layer ]
                                                                |            |
                                                     (windowed aggregates)  (raw events)
                                                                v            v
                                                          [ Hot Store ]  [ Cold Store ]
                                                                \            /
                                                                 v          v
                                                            [ Query Serving Layer ]
                                                                     |
                                                          [ Dashboards / Alerting ]
```

A real-time analytics platform ingests a continuous, high-volume stream of events (page views, clicks, purchases, custom application metrics) from potentially millions of sources, computes aggregate metrics (counts, sums, unique visitor counts, latency percentiles) over sliding or fixed time windows, and serves those aggregates to dashboards and alerting systems within seconds of the underlying events occurring — not hours later via a nightly batch job. This guide builds one from scratch: the ingestion path, the windowed-aggregation engine, approximate algorithms for cardinality and percentiles at scale, fault tolerance, multi-tenancy, and the query-serving layer that stitches real-time and historical data together.

---

# 2. Learning Objectives

By the end of this guide, you will be able to:

- Explain why a naive "write every event to a database, then `GROUP BY` on read" architecture cannot serve real-time dashboards at scale.
- Design a windowed stream-aggregation pipeline (tumbling, sliding, session windows) with correct handling of out-of-order and late-arriving events via watermarks.
- Implement approximate algorithms (HyperLogLog for cardinality, t-digest for percentiles) and explain precisely why exact computation doesn't scale for these two specific metric types.
- Design for fault tolerance (checkpointing, replay) and exactly-once aggregation semantics on top of an at-least-once delivery log.
- Diagnose and fix hot-key skew, where one tenant or metric key receives disproportionate traffic.
- Design a query-serving layer that transparently merges fresh, pre-aggregated "hot" results with historical "cold" results.
- Apply SOLID principles and recognizable design patterns (Strategy, Observer, Template Method, Builder) to keep the aggregation engine extensible without modifying already-tested code.

---

# 3. Why This Matters (The Interview, Framed)

"Design a real-time analytics platform" is a favorite senior/staff-level system design question precisely because it cannot be answered correctly with a single database and a cron job — it forces a candidate to reason about unbounded streams, time (event-time vs. processing-time, out-of-order arrival), approximation algorithms (when and why exact answers are the wrong engineering tradeoff), and failure recovery in a system that is, by definition, always running. This guide frames the design as a live interview: each major decision is preceded by the clarifying question that should have prompted it, and each design step is immediately followed by the hardest question a good interviewer asks next.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Records for immutable events/aggregates, virtual threads for high-concurrency ingestion |
| Durable ingestion log | Kafka-style partitioned log | Decouples ingestion spikes from processing; replayable for fault recovery |
| Stream processing | Custom windowed engine (built in this guide) | Teaches the actual mechanics; production systems use Flink/Kafka Streams implementing these same ideas |
| Cardinality estimation | HyperLogLog (HLL) | O(1) memory approximate distinct-count, independent of true cardinality |
| Percentile estimation | t-digest | Mergeable, bounded-memory approximate percentiles, accurate at the tails |
| Hot store | In-memory time-series store (custom, this guide) | Sub-second read latency for the last N hours of pre-aggregated data |
| Cold store | Columnar data warehouse (e.g., ClickHouse/BigQuery-style) | Cheap, scalable storage for ad-hoc historical queries over raw events |
| Checkpointing | Periodic state snapshot to durable storage | Enables replay-from-checkpoint instead of replay-from-the-beginning-of-time |

---

# 5. Project Structure

```text
realtime-analytics/
├── src/main/java/com/example/analytics/
│   ├── ingestion/
│   │   └── IngestionController.java, EventValidator.java          // §11-13
│   ├── log/
│   │   └── EventLogPartitioner.java                                // §15
│   ├── windowing/
│   │   ├── Window.java, WindowAssigner.java                         // §17-18
│   │   └── Watermark.java, WatermarkGenerator.java                  // §20-21
│   ├── aggregation/
│   │   ├── Aggregator.java (Strategy interface)                     // §26
│   │   ├── CountAggregator.java, SumAggregator.java
│   │   ├── HyperLogLogAggregator.java                                // §23
│   │   └── TDigestAggregator.java                                    // §25
│   ├── faulttolerance/
│   │   └── CheckpointedStateStore.java, EventDeduplicator.java       // §29, §32
│   ├── query/
│   │   ├── HotStore.java, ColdStore.java                             // §37, §39
│   │   └── QueryService.java                                         // §40
│   ├── alerting/
│   │   └── AlertRule.java, AlertingEngine.java (Observer)            // §43
│   └── tenancy/
│       └── TenantPartitioner.java, TenantQuota.java                  // §45
└── src/test/java/com/example/analytics/
    ├── WindowingCorrectnessTest.java
    ├── ExactlyOnceReplayTest.java
    └── HotKeySkewTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Interviewer:** *"Design a real-time analytics platform."*

Before drawing a single box, the correct first move is to narrow an intentionally vague prompt: What kind of events (page views? IoT sensor readings? financial trades?) At what volume? How "real-time" does "real-time" actually need to be — sub-second, or "within a minute or two" is fine? Do we need exact counts, or are approximate answers acceptable for some metrics? Is this single-tenant (one company's own data) or multi-tenant (a SaaS analytics product serving many customers)? The answers shape every subsequent decision, and asking them signals the difference between a candidate who designs *a* system and one who designs *this* system.

---

# 7. Functional Requirements

- **Ingest events** from client SDKs/applications at high volume (page views, clicks, custom application-defined metrics), each event carrying a timestamp, a tenant/source identifier, a metric name, and a payload.
- **Compute windowed aggregates** in near-real-time: counts, sums, averages, approximate unique counts (e.g., unique visitors), and approximate percentiles (e.g., p95 request latency) over tumbling and sliding time windows.
- **Serve low-latency queries** against these aggregates, powering live dashboards that refresh every few seconds.
- **Support ad-hoc queries** against raw historical event data for questions the pre-aggregation pipeline didn't anticipate in advance.
- **Support alerting**: notify when a computed metric crosses a configured threshold (e.g., error rate exceeds 5% over a 5-minute window).
- **Support multi-tenancy**: many independent customers' data and queries, isolated from one another, sharing the same underlying infrastructure.

---

# 8. Non-Functional Requirements

- **High ingestion throughput**: sustain millions of events per second across the platform without data loss.
- **Low end-to-end latency**: an event should be reflected in dashboard-facing aggregates within single-digit seconds of occurring, not minutes.
- **Durability**: no event is silently dropped between ingestion and aggregation, even across process crashes.
- **Horizontal scalability**: adding more ingestion/processing capacity should scale throughput roughly linearly, with no single coordination bottleneck.
- **Fault tolerance**: a crashed processing node must not lose in-flight aggregation state, and recovery must not require reprocessing the entire event history from the beginning.
- **Correctness under at-least-once delivery**: the underlying log guarantees at-least-once delivery (duplicates on redelivery are possible); the aggregation layer must still produce exactly-once-equivalent aggregate results.
- **Bounded resource usage**: memory used for aggregation state must not grow unboundedly with the number of distinct values seen (this is precisely why approximate algorithms are required for unique counts and percentiles, not merely a nice-to-have).

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Before you draw architecture boxes, what are the core domain concepts this system revolves around?"*

Naming the nouns precisely, before any architecture, prevents an entire class of later confusion (this same discipline paid off throughout every other guide in this series):

- **Event** — a single, immutable fact that occurred at a specific point in time (`tenantId`, `metricName`, `eventTime`, `payload`).
- **Window** — a bounded span of time (`[start, end)`) over which events are grouped for aggregation.
- **Aggregate** — the computed result of applying an aggregation function (count, sum, HLL, t-digest) to all events assigned to one window.
- **Watermark** — the stream processor's own estimate of "we have now seen all events up to event-time T," used to decide when a window can be safely finalized.
- **Metric Definition** — tenant-configured metadata describing what aggregation function applies to a given metric name.
- **Tenant** — an isolated customer/namespace whose events, metrics, and queries are logically (and partially physically) separated from every other tenant's.

---

# 10. Identifying the Core Domain Entities

```java
public record Event(
    String tenantId,
    String metricName,
    long eventTimeMillis,   // when it actually happened (event time)
    long ingestTimeMillis,  // when the platform received it (processing time)
    String eventId,         // unique, client-generated -- used for deduplication, §32
    double value            // the numeric payload this event contributes (e.g., 1 for a count, latency_ms for a timer)
) { }

public record Window(long startMillis, long endMillis) {
    public boolean contains(long eventTimeMillis) {
        return eventTimeMillis >= startMillis && eventTimeMillis < endMillis;
    }
}

public record Aggregate(String tenantId, String metricName, Window window, Object result) { }

public record MetricDefinition(String metricName, AggregationType type, Duration windowSize) { }

public enum AggregationType { COUNT, SUM, UNIQUE_COUNT, PERCENTILE }
```

Note the deliberate separation of `eventTimeMillis` from `ingestTimeMillis` on `Event` — this distinction is the single most important modeling decision in the entire system, and §16-21 build an entire mechanism (watermarks) around exactly this gap between "when it happened" and "when we found out."

---

# 11. High-Level Architecture Overview

```text
                 +----------------+     +------------------+     +-----------------------+
Client SDKs ---> | Ingestion API  | --> | Durable Log      | --> | Stream Processing      |
(millions/sec)   | (validate,     |     | (partitioned by  |     | (windowed aggregation, |
                 |  enrich)       |     |  tenantId+metric)|     |  Strategy-based)       |
                 +----------------+     +------------------+     +-----------------------+
                                                                      |            |
                                                          pre-aggregated        raw events
                                                          results                also archived
                                                                v                    v
                                                        +---------------+   +----------------+
                                                        |   Hot Store   |   |   Cold Store   |
                                                        | (in-memory,   |   | (columnar,     |
                                                        |  last few hrs)|   |  full history) |
                                                        +---------------+   +----------------+
                                                                \                  /
                                                                 v                v
                                                            +------------------------+
                                                            |   Query Serving Layer  |
                                                            +------------------------+
                                                                 |             |
                                                          Dashboards      Alerting Engine
```

Five layers, each independently scalable: **ingestion** (stateless, horizontally scaled), the **durable log** (the shock absorber between bursty ingestion and steady-state processing), **stream processing** (the stateful, windowed-aggregation core — the hardest part of this system, and the focus of §16-36), **storage** (hot for recent/low-latency, cold for historical/ad-hoc), and **query serving** (merges both stores into one coherent answer).

---

# 12. Follow-up Question 2 — "Why Not Just Write Every Event Straight to a Database and Query with GROUP BY?"

> **Interviewer:** *"This sounds like overkill. Why can't we just insert every event into Postgres and run `SELECT COUNT(*) ... GROUP BY` when a dashboard needs a number?"*

Because that architecture's cost scales with the **product** of ingestion volume and query frequency, not with either alone — every dashboard refresh re-scans (or re-aggregates from an index over) however many raw rows fall in the requested window, and at millions of events per second, a window spanning even a few minutes contains far more rows than any interactive query can afford to touch at read time. Real-time analytics platforms exist specifically to move the aggregation cost from **query time** (expensive, repeated on every dashboard refresh) to **ingestion time** (paid once, incrementally, per event, as it arrives) — the aggregate is already computed and sitting in the hot store by the time a dashboard asks for it.

---

# 13. Why a Naive "Write-Then-Query" Architecture Doesn't Scale

```text
Naive:  event -> INSERT INTO raw_events -> (on every dashboard refresh) SELECT COUNT(*) FROM raw_events
                                              WHERE tenant=X AND metric=Y AND ts BETWEEN a AND b
                                              -- cost per query: O(rows in window) -- RE-PAID every refresh

This design:  event -> incrementally update an in-memory aggregate for its window -> dashboard reads
                                              the ALREADY-COMPUTED aggregate directly
                                              -- cost per query: O(1) -- the expensive work happened ONCE, at ingest
```

The incremental approach also has a second, less obvious advantage: an aggregate like `COUNT` or `SUM` can be updated with a single arithmetic operation per incoming event (`count += 1`), entirely independent of how many events have already been folded into that aggregate — the per-event ingestion cost stays constant no matter how large the window's total event count grows, which is precisely the property a `GROUP BY`-on-read architecture cannot offer.

---

# 14. Follow-up Question 3 — "How Do You Decouple Ingestion Spikes from Steady Processing?"

> **Interviewer:** *"Traffic to a real system is bursty — a marketing campaign can 10x your event volume in minutes. How does the pipeline avoid falling over?"*

By inserting a **durable, partitioned log** (Kafka-style) between ingestion and processing: the ingestion layer's only job becomes validating an event and appending it to the log, an operation whose cost is nearly constant regardless of downstream processing speed — a burst of incoming events queues up in the log rather than overwhelming the stream processors directly, and the log's own durability guarantees mean a temporarily-slow (or briefly-crashed) processing layer never loses data, it simply catches up once capacity returns.

---

# 15. The Durable Ingestion Log as a Shock Absorber

```java
public class EventLogPartitioner {

    private final int partitionCount;

    public EventLogPartitioner(int partitionCount) {
        this.partitionCount = partitionCount;
    }

    public int partitionFor(Event event) {
        // partition by (tenantId, metricName) so that ALL events for one metric land on
        // the SAME partition -- required so a single stream-processing worker can own,
        // and correctly aggregate, the complete window for that metric without cross-partition coordination
        String partitionKey = event.tenantId() + ":" + event.metricName();
        return Math.floorMod(partitionKey.hashCode(), partitionCount);
    }
}
```

The partitioning key choice here is load-bearing: partitioning by `(tenantId, metricName)` rather than, say, a round-robin or per-event-ID scheme is what guarantees every event contributing to a given window ends up processed by the same worker, in the same partition, without needing any cross-worker coordination to compute that window's aggregate — this is revisited under load in §34-35 when a single hot key overwhelms its one partition.

---

# 16. Follow-up Question 4 — "How Do You Aggregate a Continuous, Unbounded Stream? There's No 'End' to Run COUNT(*) On."

> **Interviewer:** *"A stream never ends. `SUM()` needs a bounded set of rows. How do you reconcile that?"*

By imposing **windows** — artificial, explicit boundaries in time that carve the unbounded stream into a sequence of bounded, aggregatable chunks. A window is "closed" (its aggregate finalized and emitted) once the processor is confident no more events belonging to it will arrive; until then, the window's aggregate lives as mutable, in-progress state, updated incrementally as each new event for that window arrives.

---

# 17. Windowing: Tumbling, Sliding, and Session Windows

```text
Tumbling windows (fixed, non-overlapping):
  |--- window 1 ---|--- window 2 ---|--- window 3 ---|
  0               60s              120s             180s
  Each event belongs to EXACTLY ONE window.

Sliding windows (fixed size, overlapping, advance by a smaller step):
  |------- window A (60s) -------|
        |------- window B (60s) -------|
              |------- window C (60s) -------|
  0     10s   20s                            An event can belong to MULTIPLE overlapping windows.

Session windows (dynamic size, bounded by a gap of inactivity):
  |--session 1--|      gap > 30min      |--session 2--|
  Boundaries are determined by the DATA (a period of inactivity), not a fixed clock.
```

**Tumbling** windows are the simplest and most common for dashboard metrics ("events per minute"); **sliding** windows suit metrics that need a smoothly-updating rolling view (a "last 5 minutes" rate that updates every second, not jumping in 5-minute steps); **session** windows suit per-user-activity metrics (a user's browsing "session," ended by a gap of inactivity, not a fixed clock boundary) — the choice is a metric-definition concern, not a platform-wide constant, which is exactly why it belongs behind a pluggable `WindowAssigner` interface rather than a single hardcoded windowing scheme.

---

# 18. Designing the Window Assignment Strategy

```java
public interface WindowAssigner {
    List<Window> assignWindows(long eventTimeMillis);
}

public class TumblingWindowAssigner implements WindowAssigner {
    private final long windowSizeMillis;

    public TumblingWindowAssigner(Duration windowSize) {
        this.windowSizeMillis = windowSize.toMillis();
    }

    @Override
    public List<Window> assignWindows(long eventTimeMillis) {
        long windowStart = (eventTimeMillis / windowSizeMillis) * windowSizeMillis;
        return List.of(new Window(windowStart, windowStart + windowSizeMillis));
    }
}

public class SlidingWindowAssigner implements WindowAssigner {
    private final long windowSizeMillis;
    private final long slideMillis;

    public SlidingWindowAssigner(Duration windowSize, Duration slide) {
        this.windowSizeMillis = windowSize.toMillis();
        this.slideMillis = slide.toMillis();
    }

    @Override
    public List<Window> assignWindows(long eventTimeMillis) {
        // an event can belong to MULTIPLE sliding windows -- enumerate every window whose
        // [start, end) range contains this event's timestamp
        List<Window> windows = new ArrayList<>();
        long lastStart = (eventTimeMillis / slideMillis) * slideMillis;
        for (long start = lastStart; start > eventTimeMillis - windowSizeMillis; start -= slideMillis) {
            windows.add(new Window(start, start + windowSizeMillis));
        }
        return windows;
    }
}
```

`WindowAssigner` is a textbook **Strategy** pattern application: the stream processor's core loop never needs to know whether tumbling or sliding semantics are in effect — it simply calls `assignWindows(event.eventTimeMillis())` and updates every returned window's aggregate, making it trivial to add a `SessionWindowAssigner` later (§54) without touching the processing loop itself.

---

# 19. Follow-up Question 5 — "Events Arrive Out of Order and Late. When Is It Actually Safe to Finalize a Window?"

> **Interviewer:** *"A mobile client buffers events offline and uploads them 10 minutes later. If you close a window the instant its time range passes, you'll silently drop that data. But you also can't wait forever. How do you resolve this?"*

By tracking, not the wall-clock time, but an explicit, conservative **estimate of event-time progress** called a **watermark**: "we believe we have now seen all events with event-time up to T." A window is only finalized once the watermark passes its end boundary — this deliberately trades a small, bounded amount of latency (the window waits a configured grace period before closing) for materially better completeness, which is almost always the correct tradeoff for dashboard-facing metrics.

---

# 20. Watermarks and Allowed Lateness

```text
Event-time axis:  ---------------------------------------------------------------->
Events arrive:     e1(t=10)  e2(t=25)  e3(t=15, LATE!)  e4(t=40)  watermark=30
                                                                        ^
                                            window [0,30) can now be finalized --
                                            e3, though it arrived AFTER e2, has
                                            event-time 15 which is < 30, so it's
                                            still correctly folded into that window
                                            BEFORE the watermark passes 30.

If e3 had arrived AFTER the watermark already passed 30, it would be "too late" --
handled per the configured allowed-lateness policy (§21).
```

The watermark is deliberately an **estimate**, not a guarantee — a sufficiently delayed event can still arrive after its window has already been finalized; the system's job is to make that estimate good enough, via a tunable **allowed lateness** grace period, that this happens rarely enough to be an acceptable tradeoff, not to make it impossible.

---

# 21. Implementing the Watermark Generator

```java
public class WatermarkGenerator {
    private final Duration maxOutOfOrderness; // how late we tolerate events being, before calling them "too late"
    private volatile long currentWatermarkMillis = Long.MIN_VALUE;

    public WatermarkGenerator(Duration maxOutOfOrderness) {
        this.maxOutOfOrderness = maxOutOfOrderness;
    }

    public void onEvent(Event event) {
        long candidateWatermark = event.eventTimeMillis() - maxOutOfOrderness.toMillis();
        currentWatermarkMillis = Math.max(currentWatermarkMillis, candidateWatermark); // watermark NEVER moves backward
    }

    public long currentWatermark() {
        return currentWatermarkMillis;
    }

    public boolean isTooLate(Event event, Duration allowedLateness) {
        // an event is "too late" only if its window has ALREADY been finalized, i.e. the
        // watermark passed its window's end PLUS the extra grace period this policy allows
        return event.eventTimeMillis() + allowedLateness.toMillis() < currentWatermarkMillis;
    }
}
```

The watermark is derived from the single highest event-time seen so far, minus a configured slack (`maxOutOfOrderness`) — and it is monotonically non-decreasing by construction (`Math.max`), which is exactly the property that lets every downstream window-finalization check be a simple, cheap comparison rather than a re-scan of buffered events.

---

# 22. Follow-up Question 6 — "How Do You Compute Unique Visitor Counts Without Storing Every Visitor ID in Memory?"

> **Interviewer:** *"'Unique visitors per minute' needs a distinct count. Storing a `HashSet<UserId>` per window works for small volumes — what breaks at scale, and what do you do instead?"*

An exact distinct count requires memory proportional to the **cardinality itself** — a window with ten million distinct visitors needs a set holding ten million entries, and this cost is paid **per window, per tenant, per metric**, multiplying quickly across a real deployment. The practical fix is to trade a small, bounded, *known* error rate for **constant memory regardless of cardinality**, using a probabilistic data structure — **HyperLogLog** — that estimates distinct counts within roughly 1-2% error using only a few kilobytes, whether the true count is a thousand or a billion.

---

# 23. Approximate Cardinality Estimation with HyperLogLog

```text
HyperLogLog's core insight: hash each element, look at the position of the first 1-bit in
the hash's binary representation. The LONGEST run of leading zeros observed across all
hashed elements is a probabilistic signal of how many DISTINCT elements were hashed --
a longer max run strongly suggests a larger set was hashed, because seeing a rare pattern
(long run of zeros) becomes more likely the more independent hashes you've tried.

HLL splits the hash space into many "buckets" (registers) and tracks the max run-length
PER bucket, then harmonically averages across all buckets -- this dramatically reduces the
variance a single run-length estimate alone would have, at the cost of a small, fixed,
well-understood memory footprint (a few KB per HLL, independent of the true cardinality).
```

```java
public class HyperLogLog {
    private final int[] registers;      // fixed-size, e.g. 16384 registers -- NEVER grows with cardinality
    private final int registerCountLog2; // number of bits used to select a register

    public HyperLogLog(int registerCountLog2) {
        this.registerCountLog2 = registerCountLog2;
        this.registers = new int[1 << registerCountLog2];
    }

    public void add(String element) {
        long hash = hash64(element);
        int registerIndex = (int) (hash >>> (64 - registerCountLog2));
        int leadingZeros = Long.numberOfLeadingZeros(hash << registerCountLog2) + 1;
        registers[registerIndex] = Math.max(registers[registerIndex], leadingZeros);
    }

    public long estimateCardinality() {
        double harmonicMeanSum = 0;
        for (int register : registers) {
            harmonicMeanSum += Math.pow(2, -register);
        }
        double alpha = 0.7213 / (1 + 1.079 / registers.length); // bias-correction constant
        return (long) (alpha * registers.length * registers.length / harmonicMeanSum);
    }

    private long hash64(String element) { /* a well-distributed 64-bit hash, e.g. MurmurHash3 */ return 0L; }
}
```

The property that makes HLL uniquely suited to a *streaming, windowed* system — beyond its fixed memory — is that **two HLLs can be merged** (a simple element-wise max of their registers) to produce the exact same estimate as if every element from both had been fed into one HLL from the start, which is precisely what's needed when merging per-partition partial aggregates into a single window-level aggregate (§34).

---

# 24. Follow-up Question 7 — "How Do You Compute p95/p99 Latency at Scale Without Sorting Every Value?"

> **Interviewer:** *"Percentiles need sorted data. Sorting millions of latency values per window, per minute, clearly doesn't scale. What's the alternative?"*

The same tradeoff as HyperLogLog, applied to a different problem: give up exactness for a **mergeable, bounded-memory approximate structure**, in this case **t-digest** — a data structure that clusters nearby values into a small number of centroids, with the property that it allocates *more, smaller* centroids near the distribution's tails (where percentile accuracy matters most for latency SLOs, e.g. p99) and *fewer, larger* centroids in the dense middle (where slight inaccuracy matters far less).

---

# 25. Approximate Percentiles with t-digest

```text
Raw latency values:  [12, 15, 14, 13, 890, 16, 14, 15, 13, 920, 14, 15, ...]
                                          ^                    ^
                                    rare, high-latency outliers -- t-digest gives these
                                    THEIR OWN small, precise centroids, because the tail
                                    is exactly where p95/p99 accuracy is actually needed

t-digest centroids:  [(mean=14.1, count=9), (mean=890, count=1), (mean=920, count=1), ...]
                        ^ one centroid absorbs many near-identical typical values
```

```java
public class TDigestAggregator implements Aggregator<Double> {
    private final TDigest digest = TDigest.createDigest(100); // "compression" parameter -- accuracy/memory tradeoff

    @Override
    public void add(double value) {
        digest.add(value);
    }

    @Override
    public TDigest merge(TDigest other) {
        digest.add(other); // t-digest centroids are mergeable, same property HLL registers have
        return digest;
    }

    public double quantile(double q) { // e.g. quantile(0.95) for p95
        return digest.quantile(q);
    }
}
```

Both HyperLogLog and t-digest share the exact same architectural justification: a metric type whose exact computation would require memory proportional to the data volume gets replaced with a structure whose memory is bounded and configurable, at the cost of a small, well-quantified error — this is not a shortcut, it is the correct engineering answer once "compute this over millions of events per second, per window, per tenant" is the actual constraint.

---

# 26. Implementing the Aggregator Strategy Interface

```java
public interface Aggregator<T> {
    void add(double value);
    T result();
    Aggregator<T> merge(Aggregator<T> other); // required for combining per-partition partial aggregates, §34
}

public class CountAggregator implements Aggregator<Long> {
    private long count = 0;
    @Override public void add(double value) { count++; }
    @Override public Long result() { return count; }
    @Override public Aggregator<Long> merge(Aggregator<Long> other) {
        CountAggregator merged = new CountAggregator();
        merged.count = this.count + other.result();
        return merged;
    }
}

public class SumAggregator implements Aggregator<Double> {
    private double sum = 0;
    @Override public void add(double value) { sum += value; }
    @Override public Double result() { return sum; }
    @Override public Aggregator<Double> merge(Aggregator<Double> other) {
        SumAggregator merged = new SumAggregator();
        merged.sum = this.sum + other.result();
        return merged;
    }
}

public class AggregatorFactory {
    public static Aggregator<?> create(AggregationType type) {
        return switch (type) {
            case COUNT -> new CountAggregator();
            case SUM -> new SumAggregator();
            case UNIQUE_COUNT -> new HyperLogLogAggregator();
            case PERCENTILE -> new TDigestAggregator();
        };
    }
}
```

Every aggregation function — exact ones like `COUNT`/`SUM` and approximate ones like HLL/t-digest — implements the exact same `Aggregator<T>` interface, which is what lets the stream processing core loop (§27-32) update *any* window's aggregate with a single, uniform call, entirely ignorant of which specific aggregation function is actually plugged in for a given metric.

---

# 27. Follow-up Question 8 — "A Processing Node Crashes Mid-Window. Is the Partially-Computed Aggregate Lost?"

> **Interviewer:** *"You've been incrementally updating an in-memory aggregate for the last 40 seconds of a 60-second window. The process crashes. What happens to those 40 seconds of work?"*

Without a recovery mechanism, that in-memory state is simply gone, and worse — the log has already advanced past those events (or a naive re-consumption would double-count them). The fix has two parts: **checkpointing** (periodically persisting the in-progress aggregation state, along with the log offset it corresponds to, to durable storage) so a restarted worker can resume from a recent snapshot rather than from the dawn of time, and **exactly-once-equivalent replay** (§30-32) so that re-processing the small window of events between the last checkpoint and the crash doesn't double-count anything.

---

# 28. Fault Tolerance via Checkpointing and Replay

```text
Timeline:  [checkpoint @ offset 1000] --- process events 1000-1400 --- [CRASH @ offset 1400]
                                                                              |
                                                                     restart, load checkpoint @ 1000
                                                                              |
                                                                    re-consume log from offset 1000
                                                                    (events 1000-1400 are RE-PROCESSED)
                                                                              |
                                                          without dedup, events 1000-1400 would be
                                                          COUNTED TWICE -- this is exactly what §31-32 fixes
```

Checkpointing bounds the "how much work do we redo on recovery" cost to "however often we checkpoint," entirely independent of how long the system has been running in total — a checkpoint interval of, say, 30 seconds means recovery only ever needs to replay at most 30 seconds' worth of events, regardless of whether the pipeline has been running for an hour or a year.

---

# 29. Implementing Checkpointed State Store

```java
public class CheckpointedStateStore {
    private final Map<Window, Aggregator<?>> windowState = new ConcurrentHashMap<>();
    private final DurableStorage durableStorage; // e.g. blob storage, replicated filesystem
    private volatile long lastCheckpointedOffset = -1;

    public void update(Window window, double value, AggregationType type) {
        windowState.computeIfAbsent(window, w -> AggregatorFactory.create(type)).add(value);
    }

    public void checkpoint(long currentLogOffset) {
        // atomically persist BOTH the aggregation state AND the log offset it corresponds to --
        // these must be written together, or a crash between the two writes reintroduces
        // exactly the inconsistency checkpointing exists to prevent
        durableStorage.writeAtomic("checkpoint", new CheckpointSnapshot(windowState, currentLogOffset));
        lastCheckpointedOffset = currentLogOffset;
    }

    public long restoreFromCheckpoint() {
        CheckpointSnapshot snapshot = durableStorage.readLatest("checkpoint");
        if (snapshot == null) return 0; // no checkpoint yet -- start from the beginning
        windowState.putAll(snapshot.windowState());
        return snapshot.logOffset(); // the caller resumes log consumption from EXACTLY this offset
    }
}
```

The atomicity of writing the aggregation state and the log offset **together** is the load-bearing detail — if the offset were persisted first and the process crashed before the state finished writing, recovery would resume the log past events whose contribution to the aggregate was never actually saved, silently losing data in exactly the way checkpointing exists to prevent.

---

# 30. Follow-up Question 9 — "The Log Guarantees At-Least-Once Delivery. How Do You Avoid Double-Counting on Replay?"

> **Interviewer:** *"Replay after a crash necessarily re-delivers some events the aggregator already processed before the crash. A `COUNT` aggregator has no way to know 'I already saw this one.' How do you fix that?"*

By making the aggregation step **idempotent with respect to a given event's unique ID** — tracking which specific `eventId`s have already been folded into which window's aggregate, and skipping (not re-applying) any event whose ID has already been recorded, even if the log redelivers it. This converts an at-least-once delivery guarantee from the log into an effectively exactly-once *aggregation* result, without requiring the log itself to guarantee exactly-once delivery (a much harder, more expensive property to provide at the transport layer).

---

# 31. Exactly-Once Semantics via Idempotent Aggregation

```text
At-least-once delivery:  event e42 delivered once, normally -- COUNTED
                          [CRASH before checkpoint]
                          event e42 REDELIVERED on replay -- must be SKIPPED, not counted again

Idempotent aggregation:  before applying e42 to any window's aggregate, check: "have I already
                          recorded e42's ID as applied to THIS window?" -- if yes, skip;
                          if no, apply AND record the ID atomically alongside the aggregate update
```

The deduplication check must be scoped **per window**, not globally — the same `eventId` legitimately contributes to exactly one window's aggregate (or several, for overlapping sliding windows, §17), and a global "have I ever seen this ID" check would incorrectly suppress an event's valid re-application to a *different* window it also belongs to.

---

# 32. Implementing Idempotent Event Deduplication

```java
public class EventDeduplicator {
    // per-window set of already-applied event IDs -- bounded by the number of DISTINCT windows
    // currently open (a small, bounded number), not by total event volume
    private final Map<Window, Set<String>> appliedEventIds = new ConcurrentHashMap<>();

    public boolean tryApply(Window window, Event event, Runnable applyToAggregate) {
        Set<String> seenIds = appliedEventIds.computeIfAbsent(window, w -> ConcurrentHashMap.newKeySet());
        boolean isNew = seenIds.add(event.eventId()); // atomic check-and-add
        if (isNew) {
            applyToAggregate.run(); // only apply the event's contribution if it's genuinely new to this window
        }
        return isNew;
    }

    public void onWindowFinalized(Window window) {
        appliedEventIds.remove(window); // once a window closes for good, its dedup set can be freed
    }
}
```

`onWindowFinalized` is what keeps this dictionary's memory bounded — a window's deduplication set is only needed while that window is still open (i.e., still within its allowed-lateness grace period, §20-21); once finalized, there is no legitimate future event that could still arrive for it, so its dedup set is safely discarded rather than retained forever.

---

# 33. Class Diagram: The Stream Processing Core

```text
+------------------------+       +---------------------+       +------------------------+
|     WindowAssigner     |<------|  StreamProcessor     |------>|   WatermarkGenerator   |
|  <<interface>>         |       |  (orchestrates the    |       +------------------------+
|  + assignWindows(t)    |       |   per-event pipeline) |
+------------------------+       +----------+-----------+
       ^         ^                          |
       |         |                          v
+-------------+ +----------------+  +------------------------+
| Tumbling    | | Sliding        |  |  EventDeduplicator     |
| WindowAssig.| | WindowAssigner |  +------------------------+
+-------------+ +----------------+              |
                                                  v
                                   +------------------------+       +------------------------+
                                   | CheckpointedStateStore |<----->|   Aggregator<T>        |
                                   |  Map<Window,           |       |  <<interface>>          |
                                   |      Aggregator<?>>    |       |  + add(v), result()    |
                                   +------------------------+       +-----------+------------+
                                                                                 ^
                                                     +---------------------------+---------------------------+
                                                     |               |                |                      |
                                              CountAggregator  SumAggregator  HyperLogLogAggregator   TDigestAggregator
```

`StreamProcessor` is the orchestrator: for each incoming event, it asks the plugged-in `WindowAssigner` which window(s) the event belongs to, checks `EventDeduplicator` to guard against replay double-counting, and — only if genuinely new — updates the corresponding `Aggregator` inside `CheckpointedStateStore`, while feeding the event's timestamp into `WatermarkGenerator` to eventually decide when each window can be finalized and emitted downstream.

---

# 34. Follow-up Question 10 — "One Tenant's Metric Gets 100x More Traffic Than Others. What Breaks?"

> **Interviewer:** *"You partitioned by `(tenantId, metricName)` back in §15 so one worker owns a whole window. A single huge customer's `page_view` metric now generates more traffic than every other tenant combined. What happens to that one partition, and how do you fix it?"*

Since §15's partitioning scheme routes *all* events for one `(tenantId, metricName)` pair to a single partition (a deliberate choice, made so one worker could own a complete window without cross-worker coordination), a single disproportionately hot key now overwhelms the one worker assigned to that partition, while every other worker sits comfortably underloaded — a classic **hot-key skew** problem, and the direct, unavoidable cost of the very design choice that made simple, coordination-free aggregation possible in the first place.

---

# 35. Handling Hot Keys and Partition Skew

```text
Fix: TWO-STAGE aggregation for hot keys specifically.

Stage 1 (many workers, in parallel):  split the hot key's traffic further, by an ADDITIONAL
  synthetic sub-key (e.g. a random salt 0-15), so 16 workers each aggregate a SLICE of the
  hot key's events into a PARTIAL aggregate for the same window.

Stage 2 (one worker, cheap):  merge the 16 partial aggregates into the final aggregate for
  that window, using each Aggregator's own merge() method (§26) -- CountAggregator sums
  16 partial counts, HyperLogLogAggregator merges 16 partial HLL register sets, etc.
```

This is precisely why every `Aggregator<T>` implementation from §26 was required to expose a `merge()` method from the start — two-stage aggregation for hot keys is not a special case bolted on afterward, it is a direct, mechanical consequence of every aggregation function already being mergeable by design; the same `merge()` used here to combine sub-key partials is reused unchanged in §29's checkpoint-recovery path and in §40's hot/cold result merging.

---

# 36. Follow-up Question 11 — "Dashboards Need Sub-Second Query Latency. Where Does the Aggregate Actually Live for Querying?"

> **Interviewer:** *"A finalized window's aggregate needs to be readable by a dashboard within a second of being computed. Where does it actually get stored to make that possible?"*

In a **hot store** — a storage layer explicitly optimized for exactly this access pattern: point lookups and small range scans over recently-finalized aggregates, kept entirely (or mostly) in memory, covering only a modest recent time range (hours, not months) since dashboards overwhelmingly query recent data. Older aggregates age out of the hot store and either get discarded (if only recent data matters for that metric) or get rolled into the cold store's historical record.

---

# 37. The Hot Store: Serving Pre-Aggregated Results at Low Latency

```java
public class HotStore {
    // key: (tenantId, metricName, window) -- exactly the granularity aggregates are finalized at
    private final Map<HotStoreKey, Aggregate> aggregatesByKey = new ConcurrentHashMap<>();
    private final Duration retentionWindow; // e.g. 6 hours -- older entries are evicted, not queried here

    public void store(Aggregate aggregate) {
        aggregatesByKey.put(HotStoreKey.from(aggregate), aggregate);
    }

    public List<Aggregate> query(String tenantId, String metricName, long fromMillis, long toMillis) {
        return aggregatesByKey.values().stream()
            .filter(a -> a.tenantId().equals(tenantId) && a.metricName().equals(metricName))
            .filter(a -> a.window().startMillis() >= fromMillis && a.window().endMillis() <= toMillis)
            .toList(); // in production, an actual time-indexed structure replaces this linear scan
    }

    public void evictOlderThan(long cutoffMillis) {
        aggregatesByKey.values().removeIf(a -> a.window().endMillis() < cutoffMillis);
    }
}
```

The hot store deliberately holds only pre-aggregated `Aggregate` objects, never raw events — this is what keeps its memory footprint small and its query latency low, at the explicit cost of being unable to answer any question the pre-aggregation pipeline didn't anticipate in advance, which is exactly the gap §38-39 addresses.

---

# 38. Follow-up Question 12 — "What About an Ad-Hoc Query the Pre-Aggregation Didn't Anticipate?"

> **Interviewer:** *"An analyst wants to know 'unique visitors from mobile Safari in Germany last Tuesday' — a slice nobody pre-configured a metric for. The hot store only has what was pre-aggregated. Now what?"*

The hot store's entire value proposition — sub-second latency — comes from pre-computing exactly the aggregates a metric definition anticipated; it structurally cannot answer a question outside that predefined shape. The fix is to **also** retain raw (or lightly-processed) events in a separate, cheaper, higher-latency **cold store** built for exactly this: flexible, ad-hoc analytical queries over historical raw data, accepting query latencies of seconds-to-minutes in exchange for not needing to have anticipated the query shape in advance.

---

# 39. The Cold Store: Raw Event Storage for Ad-Hoc Queries

A columnar, analytical data warehouse (conceptually like ClickHouse or BigQuery) receives a copy of every raw event (§11's "raw events" branch out of stream processing), stored compressed and partitioned by time and tenant. Its columnar layout means a query touching only a few columns (e.g., `browser`, `country`, `userId` for the example above) reads only those columns' compressed data, not entire rows — this is what makes ad-hoc analytical queries over billions of historical rows tractable at all, in exchange for genuinely higher latency than the hot store's in-memory point lookups.

---

# 40. Merging Hot and Cold Results: The Query Serving Layer

```java
public class QueryService {
    private final HotStore hotStore;
    private final ColdStore coldStore;
    private final Duration hotStoreRetention;

    public QueryResult query(QueryRequest request) {
        long hotCutoff = System.currentTimeMillis() - hotStoreRetention.toMillis();

        List<Aggregate> hotResults = request.toMillis() >= hotCutoff
            ? hotStore.query(request.tenantId(), request.metricName(), Math.max(request.fromMillis(), hotCutoff), request.toMillis())
            : List.of();

        List<Aggregate> coldResults = request.fromMillis() < hotCutoff
            ? coldStore.query(request.tenantId(), request.metricName(), request.fromMillis(), Math.min(request.toMillis(), hotCutoff))
            : List.of();

        return QueryResult.merge(hotResults, coldResults); // stitched by window boundary, no overlap by construction
    }
}
```

The split point (`hotCutoff`) is chosen so hot and cold results never overlap for the same window — a query spanning both recent and historical time simply routes each portion of the requested range to whichever store actually owns it, then concatenates the results, entirely hiding from the caller which store served which part of the answer.

---

# 41. Class Diagram: The Query Serving Layer

```text
+------------------------+
|      QueryService       |
|  (Facade)                |
|  + query(QueryRequest)   |
+-----------+-------------+
            |
     +------+------+
     |             |
     v             v
+-----------+  +-----------+
| HotStore  |  | ColdStore |
| (in-mem,  |  | (columnar,|
|  recent)  |  |  full hist)|
+-----------+  +-----------+
     |             |
     +------+------+
            v
   +------------------------+
   |     QueryResult         |
   |  (merged, time-ordered) |
   +------------------------+
```

`QueryService` is a **Facade**: dashboards and the alerting engine (§43) depend only on this single, simple interface, never on `HotStore` or `ColdStore` directly — which means the hot-store retention window, the cold-store's underlying storage engine, or even the split-point logic can all change independently, as long as `QueryService`'s own contract stays stable.

---

# 42. Follow-up Question 13 — "How Do You Support Alerting on Top of This, e.g. 'Notify When Error Rate Exceeds 5%'?"

> **Interviewer:** *"Dashboards are pull-based — someone looks at them. An alert needs to push a notification the moment a threshold is crossed. How does that fit into a pipeline built around windows being finalized?"*

Every window finalization is already a discrete, well-defined event in this pipeline (§20-21 decide precisely when it happens) — alerting simply **observes** that same event stream of finalized aggregates and evaluates each tenant's configured threshold rules against it, firing a notification the instant a rule's condition is met, without needing any separate polling loop or its own copy of the aggregation logic.

---

# 43. Implementing the Alerting Engine as an Observer

```java
public interface AggregateListener {
    void onWindowFinalized(Aggregate aggregate);
}

public class AlertingEngine implements AggregateListener {
    private final Map<String, List<AlertRule>> rulesByMetric; // tenantId+metricName -> configured rules
    private final NotificationSender notificationSender;

    @Override
    public void onWindowFinalized(Aggregate aggregate) {
        String key = aggregate.tenantId() + ":" + aggregate.metricName();
        for (AlertRule rule : rulesByMetric.getOrDefault(key, List.of())) {
            if (rule.isTriggered(aggregate.result())) {
                notificationSender.send(rule.buildNotification(aggregate));
            }
        }
    }
}

public record AlertRule(String metricName, Threshold threshold, ComparisonOperator operator) {
    public boolean isTriggered(Object aggregateResult) {
        double value = ((Number) aggregateResult).doubleValue();
        return operator.compare(value, threshold.value());
    }
    public Notification buildNotification(Aggregate aggregate) {
        return new Notification(aggregate.tenantId(), metricName, aggregate.result(), threshold.value());
    }
}
```

`StreamProcessor` (§33) publishes every finalized `Aggregate` to a small set of registered `AggregateListener`s — this is the **Observer** pattern applied directly: `AlertingEngine` is just one subscriber among potentially several (a metrics-export listener, an audit-log listener), and none of them require any change to the core windowing/aggregation logic to be added or removed.

---

# 44. Follow-up Question 14 — "How Is This Multi-Tenant Without One Tenant's Load Affecting Another's?"

> **Interviewer:** *"This is a SaaS analytics product — many customers share the platform. What stops one noisy tenant from starving another's query latency or ingestion throughput?"*

Two complementary mechanisms, applied at different layers: **partition-level isolation** (§15's partitioning already scopes each tenant+metric to specific log partitions and processing workers, so one tenant's aggregation load is physically confined rather than sharing a single global worker pool) and **quota enforcement** at the ingestion boundary (rejecting or throttling a tenant's events once they exceed their provisioned rate, rather than letting unbounded traffic from one tenant degrade the durable log or processing layer for everyone else).

---

# 45. Multi-Tenancy: Isolation via Partitioning and Quotas

```java
public class TenantQuota {
    private final Map<String, RateLimiter> perTenantLimiters; // one bucket per tenant, §-referenced Rate Limiter guide

    public boolean allowIngest(String tenantId) {
        return perTenantLimiters
            .computeIfAbsent(tenantId, id -> RateLimiter.create(quotaForTenant(id)))
            .tryAcquire();
    }

    private double quotaForTenant(String tenantId) {
        return tenantConfigService.getProvisionedEventsPerSecond(tenantId);
    }
}
```

Enforcing the quota at the **ingestion API**, before an event is even appended to the durable log, is deliberate — rejecting an over-quota event as early as possible means the cost of one tenant's excess traffic is confined to that tenant's own rejected requests, never propagating into log write pressure, processing load, or storage cost that every other tenant would otherwise indirectly share.

---

# 46. Capacity Estimation: Ingestion Throughput and Storage

```text
Assume: 50 million active users, each generating ~20 events/day on average
Total events/day  = 50,000,000 * 20                     = 1,000,000,000 events/day
Average events/sec = 1,000,000,000 / 86,400              ≈ 11,600 events/sec (sustained average)
Peak events/sec     (assume 5x average at peak traffic)  ≈ 58,000 events/sec

Raw event size (tenantId, metricName, timestamps, eventId, value, small payload) ≈ 200 bytes
Raw storage/day = 1,000,000,000 events * 200 bytes       = 200 GB/day (before compression)
With ~5x columnar compression in the cold store          ≈ 40 GB/day effective storage growth
```

At this scale, a single-digit number of ingestion API instances behind a load balancer, and a durable log with a few dozen partitions, comfortably absorbs both the sustained average and the 5x peak burst — the numbers only become genuinely challenging (requiring the hot-key mitigation of §35, and careful partition-count planning) once a single platform serves traffic several orders of magnitude larger, which is exactly the regime real large-scale analytics platforms operate in.

---

# 47. Capacity Estimation: Aggregation State Size and Memory

```text
Assume: 10,000 distinct (tenantId, metricName) pairs, each with a 1-minute tumbling window,
        keeping the last 60 minutes of finalized windows "warm" in the hot store for query.

Per-window aggregate size (COUNT/SUM: ~16 bytes; HyperLogLog: ~12 KB fixed; t-digest: ~2-4 KB) 
Worst case (all metrics use HLL): 10,000 keys * 60 windows * 12 KB  ≈ 7.2 GB total hot-store memory

This is why choosing the CHEAPEST aggregation type that satisfies the actual requirement matters:
a COUNT-only metric costs ~1000x less memory than the same metric mistakenly configured as
UNIQUE_COUNT -- the choice of AggregationType per metric (§10's MetricDefinition) is a real
capacity-planning decision, not just a semantic one.
```

This is the concrete payoff of §22-25's decision to use approximate, bounded-memory structures at all — even in this worst-case estimate, total hot-store memory for ten thousand actively-tracked metrics stays in the single-digit gigabytes, comfortably fitting on a handful of commodity servers, specifically because HLL and t-digest's memory is a small constant per aggregate rather than proportional to the (potentially enormous) underlying cardinality.

---

# 48. Full Worked Example: One Event's Journey, Traced End to End

```text
1. A mobile client sends: {tenantId: "acme", metricName: "page_view", eventTime: T, eventId: "e-991", value: 1}
2. Ingestion API validates the event's shape, checks TenantQuota.allowIngest("acme") -> allowed (§45)
3. EventLogPartitioner routes it to partition hash("acme:page_view") % N (§15)
4. StreamProcessor (owning that partition) receives it:
     a. TumblingWindowAssigner.assignWindows(T) -> Window[start=T0, end=T0+60000] (§18)
     b. WatermarkGenerator.onEvent(event) updates the watermark estimate (§21)
     c. EventDeduplicator.tryApply(window, event, ...) -> true (first time seeing "e-991" for this window) (§32)
     d. CheckpointedStateStore updates the CountAggregator for that window: count += 1 (§29)
5. ~30 seconds later, a periodic checkpoint persists (windowState, logOffset) atomically (§29)
6. Once the watermark passes T0+60000 + allowedLateness, the window is finalized:
     a. Final Aggregate{tenantId="acme", metricName="page_view", window, result=<final count>} is emitted
     b. HotStore.store(aggregate) makes it queryable within milliseconds (§37)
     c. AlertingEngine.onWindowFinalized(aggregate) evaluates any configured threshold rules (§43)
     d. The raw event is also durably archived into the ColdStore for future ad-hoc queries (§39)
7. A dashboard querying "page_view count, last hour" calls QueryService.query(...), which merges
   this window's now-finalized hot-store aggregate with any older windows already past hot-store
   retention, served instead from the cold store (§40)
```

Every single mechanism introduced by a follow-up question in this guide appears somewhere in this one event's journey — which is exactly the point: none of §15-45 are independent, optional add-ons, they are the actual, load-bearing steps a real event passes through in this design.

---

# 49. Final Architecture Diagram

```text
                        +-------------------+
Client SDKs  --events-->|  Ingestion API    |--(quota check, §45)-->  REJECTED if over quota
                        +---------+---------+
                                  |
                                  v
                     +------------------------+
                     | Durable Partitioned Log|  (partition by tenantId+metricName, §15)
                     +------------+-----------+
                                  |
                 +----------------+-----------------+
                 v                                   v
      +--------------------------+          +--------------------+
      |     StreamProcessor       |          |   (raw event copy)  |
      |  WindowAssigner            |          +----------+----------+
      |  WatermarkGenerator        |                     |
      |  EventDeduplicator         |                     v
      |  CheckpointedStateStore    |             +----------------+
      |     -> Aggregator<T>       |             |   Cold Store   |
      +-------------+--------------+             +--------+-------+
                     | (finalized Aggregate)               |
        +------------+-------------+                       |
        v                          v                        |
 +--------------+          +-----------------+                |
 |  Hot Store   |          | AlertingEngine  |                |
 +------+-------+          | (Observer)      |                |
        |                  +-----------------+                |
        +------------------------+---------------------------+
                                  v
                       +------------------------+
                       |     QueryService        |
                       |     (Facade)            |
                       +-----------+-------------+
                                   |
                          Dashboards / Ad-hoc Queries
```

---

# 50. Design Patterns Used Throughout This Guide

- **Strategy** — `WindowAssigner` (§18: tumbling vs. sliding vs. future session windows) and `Aggregator<T>` (§26: count/sum/HLL/t-digest) are both swappable behind stable interfaces, chosen per metric definition.
- **Observer** — `AggregateListener` (§43): `AlertingEngine` and any future subscriber (metrics export, audit logging) react to finalized windows without the core pipeline knowing or caring who's listening.
- **Facade** — `QueryService` (§41): hides the hot-store/cold-store split and the merge logic behind one simple query interface.
- **Template Method** — the per-event processing sequence in `StreamProcessor` (assign windows → check watermark → dedup → update aggregate) is a fixed skeleton; only the plugged-in `WindowAssigner`/`Aggregator` strategies vary the specifics.
- **Factory** — `AggregatorFactory` (§26): centralizes the mapping from a metric's configured `AggregationType` to a concrete `Aggregator` implementation, so adding a new aggregation type touches one place.

---

# 51. SOLID Principles Applied

- **Single Responsibility** — `WatermarkGenerator` only tracks event-time progress; `EventDeduplicator` only guards against replay double-counting; `CheckpointedStateStore` only persists/restores aggregation state — none of the three knows how to do the others' job, even though all three cooperate on every single event.
- **Open/Closed** — adding a new `AggregationType` (e.g., a future `MIN`/`MAX`) means adding a new `Aggregator<T>` implementation and one `AggregatorFactory` case, never modifying `StreamProcessor`'s core loop.
- **Liskov Substitution** — every `Aggregator<T>` implementation must honestly support `add()`/`result()`/`merge()` with the same contract, so `CheckpointedStateStore` can treat all of them uniformly regardless of which is actually plugged in for a given metric.
- **Interface Segregation** — `AggregateListener` exposes exactly one method (`onWindowFinalized`), so a simple metrics-export subscriber isn't forced to implement irrelevant alerting-specific behavior.
- **Dependency Inversion** — `StreamProcessor` depends on the `WindowAssigner` and `Aggregator<T>` abstractions, never on a concrete tumbling-window or count-aggregator implementation, so either can be swapped per metric without touching the processor itself.

---

# 52. Common Mistakes When Building This Yourself

```text
MISTAKE                                                CORRECT APPROACH (this guide's section)
Using processing-time instead of event-time for windows Watermarks track EVENT time explicitly (§19-21)
Storing every distinct value in a HashSet for uniques    HyperLogLog: bounded, mergeable memory (§22-23)
Sorting all values to compute percentiles per window     t-digest: bounded, mergeable, tail-accurate (§24-25)
Checkpointing state without the corresponding log offset  Atomic (state, offset) checkpoint write (§28-29)
Assuming the log's at-least-once delivery is enough       Idempotent, per-window event deduplication (§30-32)
One partition per tenant regardless of that tenant's load Two-stage aggregation for hot keys (§34-35)
Querying only the hot store and calling it "complete"     Hot+cold merge in the query serving layer (§38-40)
No per-tenant rate limiting on a shared platform          Quota enforcement at the ingestion boundary (§44-45)
```

---

# 53. Testing Strategy

- **Unit tests** — each `Aggregator<T>` implementation in isolation (does `merge()` of two partial `CountAggregator`s produce the correct sum? does `HyperLogLogAggregator`'s estimate stay within its documented error bound against a known true cardinality?).
- **Windowing correctness tests** — feed a carefully-ordered (and deliberately out-of-order) sequence of timestamped events through `TumblingWindowAssigner`/`SlidingWindowAssigner` and assert each event lands in exactly the expected window(s).
- **Exactly-once replay tests** — simulate a crash mid-window (stop after a partial checkpoint), restart from that checkpoint, replay the log from the checkpointed offset, and assert the final aggregate matches what a non-crashing run would have produced (this is precisely what `EventDeduplicator` exists to guarantee).
- **Hot-key skew tests** — synthetically generate one wildly disproportionate key's traffic and confirm two-stage aggregation (§35) keeps its owning worker's throughput within acceptable bounds compared to a non-skewed baseline.
- **Query-serving integration tests** — issue a query spanning both hot and cold store time ranges and assert the merged result is correctly ordered with no duplicate or missing windows at the split boundary.

---

# 54. Suggested Future Enhancements

- **Session windows** (§17's third windowing type, not yet implemented in this guide) for per-user-activity metrics bounded by inactivity gaps rather than fixed clock boundaries.
- **Adaptive checkpoint intervals** — shortening the checkpoint interval automatically for metrics/partitions experiencing unusually high write rates, bounding worst-case replay cost more tightly than a single global interval can.
- **Tiered cold storage** — moving very old, rarely-queried historical data to cheaper, higher-latency archival storage (e.g., object storage with retrieval delay) beneath the existing cold store, further reducing cost for data queried only rarely.
- **Cross-metric derived aggregates** — supporting metrics defined as a function of other metrics' aggregates (e.g., "error rate" derived from `error_count / total_count` over the same window), rather than only aggregating raw event values directly.
- **Dynamic, self-tuning HLL/t-digest precision** — automatically increasing a specific metric's register count or t-digest compression when its estimate's error is observed (via periodic exact-count sampling) to exceed the tenant's configured tolerance.

---

# 55. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. Why can't a plain database with `GROUP BY` on read serve this system's latency requirements? (§12-13)
2. Design the windowing scheme: tumbling vs. sliding vs. session, and when each is the right choice. (§17)
3. Events arrive late and out of order. Design a mechanism to decide when a window is safe to finalize. (§19-21)
4. Why can't you use an exact `HashSet` for unique visitor counts at scale — and what's the alternative, precisely? (§22-23)
5. A processing node crashes mid-window. Walk through recovery, including how you avoid double-counting on replay. (§27-32)
6. One tenant's traffic is 100x every other tenant's. Diagnose the failure mode and fix it. (§34-35)
7. Design the query-serving layer so a single query can transparently span both very recent and very old data. (§36-40)
8. How would you add alerting on top of this pipeline without duplicating the aggregation logic? (§42-43)

---

# 56. Final Takeaway

Every hard decision in this guide traces back to one recurring tension: **an unbounded, continuous stream must still produce bounded, timely, correct answers** — windows bound the *stream* (§16-18), watermarks bound the *wait* for late data (§19-21), HyperLogLog and t-digest bound the *memory* per aggregate (§22-25), checkpointing bounds the *recovery* cost (§27-29), and two-stage aggregation bounds the *load* any single worker must absorb (§34-35). None of these are independent tricks — they are the same engineering instinct, applied consistently wherever "unbounded" would otherwise creep into a system that has to keep running, correctly, forever.

---
