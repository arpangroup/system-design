# Real-Time Analytical System — Step-by-Step System Design Guide

## 1. System Design Interview Question

> **Design a Real-Time Analytical System**
>
> Design a distributed real-time analytical platform that ingests large volumes of events from applications, services, devices, and business systems and allows users to run dashboards, aggregations, filters, and analytical queries with low latency.
>
> The system should support:
>
> - Real-time event ingestion
> - Streaming and near-real-time processing
> - Aggregations such as count, sum, average, min/max, distinct count
> - Time-windowed analytics
> - Ad-hoc analytical queries
> - Dashboards and alerts
> - Horizontal scaling
> - Fault tolerance
> - Data replay and recovery
> - Historical analytics
> - Multi-tenant isolation
> - Configurable retention
>
> Design both the **High-Level Design (HLD)** and **Low-Level Design (LLD)**.
>
> The implementation should follow **SOLID principles**, clean architecture, and extensible design patterns.

---

# 2. Interview Follow-Ups

A strong system-design discussion can be driven through these follow-ups:

### Follow-up 1 — Requirements

- What are the functional requirements?
- What are the non-functional requirements?
- What does "real-time" mean?
- What latency is acceptable?

### Follow-up 2 — Scale

- How many events arrive per second?
- What is the average event size?
- How many concurrent users query the system?
- How much data is stored per day?

### Follow-up 3 — Ingestion

- How do producers send events?
- How do we handle bursts?
- What happens when consumers are slow?
- How do we guarantee ordering?

### Follow-up 4 — Processing

- Where should aggregation happen?
- How do tumbling, sliding, and session windows work?
- How do we handle late events?
- How do we recover streaming state?

### Follow-up 5 — Storage

- Should we use a relational database?
- Should we use a columnar analytical database?
- Why do we need hot and cold storage?
- How do we partition data?

### Follow-up 6 — Querying

- How do dashboards query the data?
- How do we prevent expensive queries from affecting ingestion?
- How do we cache repeated dashboard queries?

### Follow-up 7 — Reliability

- What happens if Kafka/broker/processor/storage fails?
- Can events be replayed?
- How do we achieve at-least-once or exactly-once semantics?

### Follow-up 8 — Extensibility

- How can we add a new aggregation?
- How can we add another storage engine?
- How can we add another event source?
- How do SOLID principles influence the design?

### Follow-up 9 — Advanced Scale

- What if traffic increases 10x?
- What if one tenant generates 50% of traffic?
- What if a single analytical query scans billions of records?

---

# 3. Clarify "Real-Time"

The first important interview question is:

> What does real-time mean?

Real-time can mean different things:

| Requirement | Typical Target |
|---|---:|
| Interactive dashboard | < 1–2 seconds |
| Streaming metric | < 5 seconds |
| Near-real-time analytics | < 30–60 seconds |
| Batch analytics | Minutes/hours |

For this design, assume:

```text
Event ingestion:
    < 1 second

Streaming aggregation:
    1–5 seconds

Dashboard query:
    p95 < 2 seconds

Alert evaluation:
    < 5 seconds

Historical analytical queries:
    < 10–30 seconds depending on scan size
```

These are design targets rather than hard guarantees.

---

# 4. Functional Requirements

## 4.1 Event Ingestion

The system should accept events from:

- REST APIs
- SDKs
- Backend services
- IoT devices
- Application logs
- Message brokers
- Database CDC pipelines

Example event:

```json
{
  "eventId": "evt-123",
  "tenantId": "tenant-1",
  "eventType": "ORDER_CREATED",
  "timestamp": "2026-09-26T10:10:00Z",
  "userId": "user-100",
  "properties": {
    "amount": 1200,
    "city": "Hyderabad",
    "paymentMethod": "UPI"
  }
}
```

---

# 5. Event Schema

A common envelope should be used.

```text
Event
 ├── eventId
 ├── tenantId
 ├── eventType
 ├── eventTime
 ├── ingestionTime
 ├── source
 ├── schemaVersion
 ├── partitionKey
 └── payload
```

### Event Time vs Ingestion Time

This distinction is extremely important.

```text
Device generated event
        |
        | eventTime = 10:00:00
        v
Network delay
        |
        | ingestionTime = 10:00:08
        v
Analytics system
```

Analytics should usually calculate time windows using **event time**, not ingestion time.

---

# 6. Functional Requirements

The platform should support:

1. Event ingestion
2. Event validation
3. Schema validation
4. Event partitioning
5. Stream processing
6. Windowed aggregation
7. Filtering
8. Group-by
9. Historical storage
10. Real-time dashboards
11. Alerts
12. Query APIs
13. Data retention
14. Replay
15. Dead-letter processing
16. Multi-tenancy
17. Authentication/authorization
18. Query management

---

# 7. Non-Functional Requirements

## Scalability

The system should horizontally scale:

```text
10K events/sec
        ↓
100K events/sec
        ↓
1M+ events/sec
```

without redesigning the entire architecture.

## Availability

Target:

```text
99.9%+
```

for ingestion and query services.

## Latency

Example:

```text
Ingestion p95       < 500 ms
Aggregation         < 5 sec
Dashboard query     < 2 sec
```

## Durability

Events should not be lost because a processing node crashes.

## Fault Tolerance

Components should survive:

- node failure
- network failure
- consumer restart
- storage failure
- broker failure

## Consistency

Different components can use different consistency guarantees.

For example:

```text
Raw events:
    durable

Streaming aggregates:
    eventually consistent

Dashboard:
    near-real-time
```

---

# 8. Capacity Estimation

Assume:

```text
Events/sec = 100,000
Average event size = 1 KB
```

Raw ingestion:

```text
100,000 × 1 KB
= 100 MB/sec
```

Per day:

```text
100 MB × 86,400
≈ 8.64 TB/day
```

For 30 days:

```text
8.64 × 30
≈ 259 TB
```

This is before compression, replication, indexes, and metadata.

Therefore a traditional single-node relational database is not a suitable primary raw-event store.

---

# 9. High-Level Architecture

```text
                         +----------------------+
                         |      Clients         |
                         | Dashboard / API      |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         |     Query API        |
                         +----------+-----------+
                                    |
                    +---------------+---------------+
                    |                               |
                    v                               v
             +-------------+                 +-------------+
             | Query Cache |                 | Query       |
             |             |                 | Planner     |
             +-------------+                 +------+------+
                                                   |
                                                   v
                                           +---------------+
                                           | Analytical DB |
                                           +---------------+

 Producers
    |
    v
+-------------------+
| Ingestion Gateway |
+---------+---------+
          |
          v
+-------------------+
| Message Broker    |
| Kafka/Pulsar      |
+---------+---------+
          |
          +-----------------------+
          |                       |
          v                       v
+-------------------+     +-------------------+
| Stream Processor  |     | Raw Event Sink    |
|                   |     |                   |
| Filter            |     | Object Storage    |
| Window            |     | Data Lake         |
| Aggregate         |     |                   |
+---------+---------+     +-------------------+
          |
          v
+-------------------+
| Hot Analytical DB |
|                   |
| ClickHouse-like   |
| columnar store    |
+---------+---------+
          |
          v
+-------------------+
| Dashboard/Alerts  |
+-------------------+
```

---

# 10. Core Components

## 10.1 Ingestion Gateway

Responsibilities:

- authentication
- tenant validation
- schema validation
- rate limiting
- event normalization
- partition-key generation
- publishing to broker

It should **not** perform expensive analytical processing.

---

# 11. Message Broker

A distributed log such as Kafka/Pulsar can provide:

```text
Producer
   |
   v
Topic
   |
   +---- Partition 0
   +---- Partition 1
   +---- Partition 2
   +---- Partition N
```

Partitioning enables parallel consumption.

Example:

```text
partitionKey = tenantId + ":" + userId
```

This can preserve ordering for events sharing the same key.

---

# 12. Why a Message Broker?

Without a broker:

```text
Producer ---> Processor ---> Database
```

A processor failure can cause backpressure or data loss.

With a broker:

```text
Producer
   |
   v
Broker
   |
   +---- Processor A
   +---- Processor B
   +---- Processor C
```

The broker provides:

- buffering
- decoupling
- replay
- partitioning
- consumer groups
- fault tolerance

---

# 13. Stream Processing Layer

The stream processor performs:

```text
Event
  |
  v
Deserialize
  |
  v
Validate
  |
  v
Filter
  |
  v
Enrich
  |
  v
Window
  |
  v
Aggregate
  |
  v
Persist
```

Possible technologies:

- Apache Flink
- Kafka Streams
- Spark Structured Streaming
- custom Java stream processor for learning

---

# 14. Windowing

Real-time analytics usually requires time windows.

## Tumbling Window

Non-overlapping windows:

```text
00:00 ---- 01:00
01:00 ---- 02:00
02:00 ---- 03:00
```

Example:

```text
Count orders per minute
```

---

# 15. Sliding Window

Overlapping windows:

```text
10:00 ───────── 10:05
      10:01 ───────── 10:06
            10:02 ───────── 10:07
```

Example:

> Number of orders in the last 5 minutes, calculated every minute.

---

# 16. Session Window

A session groups events separated by inactivity.

Example:

```text
User activity
10:00
10:01
10:03
10:04

----- inactivity -----

10:30
10:31
```

These become two sessions.

---

# 17. Late Events

Suppose:

```text
Event time = 10:01
Arrival time = 10:05
```

The event belongs to the old window.

A robust system uses:

```text
Watermark
+
Allowed lateness
+
State correction
```

Example:

```text
Window:
10:00–10:05

Allowed lateness:
2 minutes

Events accepted until:
10:07
```

---

# 18. Aggregation Model

Support:

```text
COUNT
SUM
AVG
MIN
MAX
DISTINCT_COUNT
PERCENTILE
```

Example:

```sql
SELECT
    city,
    COUNT(*) AS orders,
    SUM(amount) AS revenue
FROM orders
GROUP BY city;
```

---

# 19. Incremental Aggregation

Do not repeatedly scan raw events.

Instead:

```text
Events

Order 100
Order 101
Order 102
Order 103

        ↓

Window Aggregator

Hyderabad:
    count = 4
    revenue = 5200
```

This dramatically reduces query work.

---

# 20. Analytical Storage

A real-time analytical system usually separates:

## Hot Data

Recent data:

```text
Last few hours/days
```

Stored in a fast analytical database.

## Warm Data

Recent historical data:

```text
Days/months
```

## Cold Data

Long-term raw data:

```text
Object Storage
S3 / GCS / Azure Blob
```

Architecture:

```text
                 +----------------+
                 | Hot Analytics  |
                 +----------------+
                         |
                         v
                 +----------------+
                 | Warm Analytics |
                 +----------------+
                         |
                         v
                 +----------------+
                 | Object Storage |
                 +----------------+
```

---

# 21. Why Columnar Storage?

Analytical queries usually read a few columns from many rows.

Example:

```sql
SELECT
    city,
    SUM(amount)
FROM orders
WHERE event_time >= ...
GROUP BY city;
```

Only a few columns are required.

Columnar storage is therefore well suited for:

- aggregations
- scans
- compression
- analytical workloads

---

# 22. Data Partitioning

Partition primarily by time.

Example:

```text
events
 ├── 2026-09-24
 ├── 2026-09-25
 ├── 2026-09-26
 └── ...
```

Within partitions, additional ordering/indexing can use:

```text
tenantId
eventType
eventTime
```

Avoid creating too many tiny partitions.

---

# 23. Query Architecture

A query should pass through:

```text
Client
  |
  v
Query API
  |
  v
Authentication
  |
  v
Query Validator
  |
  v
Query Planner
  |
  +------> Cache
  |
  v
Analytical DB
  |
  v
Result Aggregator
  |
  v
Client
```

---

# 24. Query API

Example:

```http
POST /api/v1/query
```

Request:

```json
{
  "tenantId": "tenant-1",
  "metric": "revenue",
  "dimensions": ["city"],
  "filters": [
    {
      "field": "paymentMethod",
      "operator": "EQ",
      "value": "UPI"
    }
  ],
  "window": {
    "from": "2026-09-26T10:00:00Z",
    "to": "2026-09-26T11:00:00Z"
  },
  "aggregation": "SUM",
  "field": "amount"
}
```

---

# 25. Query Optimization

Potential optimizations:

### Predicate Pushdown

Apply filters as early as possible.

### Projection Pushdown

Read only required columns.

### Partition Pruning

Avoid scanning irrelevant time partitions.

### Pre-Aggregation

Use materialized aggregates.

### Result Cache

Cache frequently repeated dashboard queries.

---

# 26. Dashboard Architecture

```text
Browser
   |
   v
Dashboard API
   |
   +------> Query Cache
   |
   v
Query Service
   |
   v
Analytical DB
```

Example dashboard:

```text
Orders/minute
Revenue/hour
Active users
Conversion rate
Top cities
Top products
Error rate
```

---

# 27. Push vs Polling

Two approaches:

## Polling

```text
Browser ---> GET /metrics
Browser ---> GET /metrics
Browser ---> GET /metrics
```

Simple but inefficient.

## WebSocket/SSE

```text
Browser <==== persistent connection ====> Server
```

The server can push updated metrics.

For dashboards, SSE is often sufficient when communication is primarily server-to-client.

---

# 28. Alerting System

Example:

> Alert when error rate > 5% for 5 consecutive minutes.

Architecture:

```text
Stream Processor
      |
      v
Metric Store
      |
      v
Alert Evaluator
      |
      v
Notification Service
      |
      +---- Email
      +---- SMS
      +---- Push
      +---- Webhook
```

---

# 29. Alert Rule

```text
AlertRule
 ├── ruleId
 ├── tenantId
 ├── metric
 ├── operator
 ├── threshold
 ├── window
 ├── evaluationInterval
 ├── severity
 └── notificationChannels
```

Example:

```json
{
  "metric": "error_rate",
  "operator": "GT",
  "threshold": 5,
  "window": "5m"
}
```

---

# 30. Exactly-Once vs At-Least-Once

This is an important interview discussion.

## At-Least-Once

An event can be processed more than once.

```text
Event
  |
  v
Processor
  |
  X crash
  |
  v
Retry
```

The same event may be processed again.

Use:

```text
eventId
+
idempotent writes
```

to avoid incorrect results.

## At-Most-Once

An event is processed zero or one time.

Risk:

```text
Event loss
```

## Exactly-Once

The system attempts to ensure each event affects the result once.

This is more complex and usually requires coordination between:

- broker offsets
- processing state
- transactional sinks
- checkpointing

For many practical systems, **at-least-once + idempotency** is an effective engineering trade-off.

---

# 31. Idempotency

Suppose:

```text
eventId = EVT-100
```

The processor receives it twice.

Maintain:

```text
ProcessedEventStore

EVT-100 -> processed
```

Before applying a non-idempotent operation:

```text
if alreadyProcessed(eventId):
    ignore

else:
    process
    markProcessed(eventId)
```

At high scale, this can be implemented through transactional/idempotent sink semantics rather than a centralized lookup for every event.

---

# 32. Dead Letter Queue

Invalid events should not block the pipeline.

```text
Event
 |
 v
Validation
 |
 +---- valid ------> Kafka
 |
 +---- invalid ----> DLQ
```

DLQ record:

```text
eventId
originalPayload
errorCode
errorMessage
timestamp
source
schemaVersion
```

---

# 33. Replay

One of the biggest advantages of event-based architecture is replay.

```text
Object Storage / Kafka
        |
        v
Replay Service
        |
        v
Kafka
        |
        v
New Processor Version
```

Use cases:

- bug fix
- new aggregation
- rebuilding materialized views
- data recovery

---

# 34. Schema Evolution

Events evolve.

Version 1:

```json
{
  "amount": 100
}
```

Version 2:

```json
{
  "amount": 100,
  "currency": "INR"
}
```

Use:

```text
schemaVersion
+
Schema Registry
```

Compatibility policies:

```text
Backward compatible
Forward compatible
Full compatible
```

Avoid silently changing the meaning of an existing field.

---

# 35. Multi-Tenancy

Every event should contain:

```text
tenantId
```

Every query should be scoped by:

```text
tenantId
```

Enforce tenant isolation at multiple layers:

```text
API
 ↓
Authorization
 ↓
Query Planner
 ↓
Storage
```

Never trust a tenant ID supplied directly by a client without validating it against the authenticated identity.

---

# 36. Security

Security requirements:

- TLS
- OAuth/JWT
- API keys for ingestion
- RBAC
- tenant isolation
- encryption at rest
- encryption in transit
- audit logs
- rate limiting
- query authorization
- sensitive-field masking

---

# 37. Rate Limiting

Ingestion:

```text
Tenant A -> 10K events/sec
Tenant B -> 100 events/sec
```

Use tenant-level quotas.

Possible algorithm:

```text
Token Bucket
```

Configuration:

```text
capacity = 10,000
refillRate = 5,000/sec
```

---

# 38. Backpressure

Suppose:

```text
Producer = 100K events/sec
Processor = 60K events/sec
```

The broker accumulates:

```text
Backlog
100K
110K
120K
...
```

Solutions:

1. Increase consumer instances
2. Increase partitions
3. Optimize processing
4. Batch writes
5. Reduce expensive enrichment
6. Apply admission control
7. Scale automatically

---

# 39. Hot-Key Problem

Suppose:

```text
tenantId = BIG-CUSTOMER
```

receives 70% of all traffic.

A partition based only on tenant ID creates:

```text
Partition 0 -> 70%
Partition 1 -> 5%
Partition 2 -> 5%
...
```

This creates a hotspot.

Possible solution:

```text
partitionKey =
    tenantId + ":" + hash(entityId)
```

But this can affect ordering.

This is a classic trade-off:

```text
Ordering
    vs
Parallelism
```

---

# 40. High-Level Failure Scenarios

## Processor Failure

```text
Processor crashes
       |
       v
Consumer group rebalances
       |
       v
Another processor continues
```

## Database Failure

```text
Processor
   |
   v
Retry
   |
   v
Buffer in broker
```

## Broker Partition Failure

Use replicated partitions.

```text
Leader
  |
  +---- Replica 1
  |
  +---- Replica 2
```

---

# 41. Disaster Recovery

Use:

```text
Primary Region
       |
       v
Replicated Event/Data
       |
       v
Secondary Region
```

Define:

### RPO

Maximum acceptable data loss.

Example:

```text
RPO = < 1 minute
```

### RTO

Maximum acceptable recovery time.

Example:

```text
RTO = < 15 minutes
```

---

# 42. Observability

The platform itself must be observable.

Metrics:

```text
events_received_total
events_failed_total
consumer_lag
processing_latency
query_latency
query_failure_rate
storage_write_latency
cache_hit_rate
alert_evaluation_latency
```

Logs:

```text
structured JSON logs
```

Tracing:

```text
Producer
  ↓
Gateway
  ↓
Kafka
  ↓
Processor
  ↓
Analytical DB
  ↓
Query API
```

Use correlation IDs.

---

# 43. Low-Level Design

Now move from services to classes.

Recommended package structure:

```text
com.analytics
├── ingestion
│   ├── controller
│   ├── service
│   ├── validator
│   └── publisher
│
├── event
│   ├── model
│   ├── schema
│   └── serializer
│
├── processing
│   ├── pipeline
│   ├── filter
│   ├── window
│   ├── aggregation
│   └── state
│
├── query
│   ├── controller
│   ├── parser
│   ├── planner
│   └── executor
│
├── storage
│   ├── repository
│   ├── hot
│   ├── warm
│   └── cold
│
├── alert
│   ├── evaluator
│   ├── rule
│   └── notification
│
├── tenant
│   ├── authorization
│   └── quota
│
└── common
    ├── exception
    ├── metrics
    └── security
```

---

# 44. Core Domain Model

```text
Tenant
  |
  +---- Event
  |
  +---- Dashboard
  |
  +---- Query
  |
  +---- AlertRule
```

Event:

```text
Event
 ├── eventId
 ├── tenantId
 ├── eventType
 ├── eventTime
 ├── ingestionTime
 ├── payload
 └── metadata
```

---

# 45. Class-Level Design

```text
                    +----------------------+
                    | EventIngestionService|
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | EventValidator       |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | EventPublisher       |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | MessageBroker        |
                    +----------------------+

                    +----------------------+
                    | StreamProcessor       |
                    +----------+-----------+
                               |
                +--------------+--------------+
                |                             |
                v                             v
       +----------------+             +----------------+
       | EventFilter    |             | EventEnricher  |
       +----------------+             +----------------+
                |
                v
       +----------------+
       | WindowManager  |
       +-------+--------+
               |
               v
       +----------------+
       | Aggregator     |
       +-------+--------+
               |
               v
       +----------------+
       | MetricSink     |
       +----------------+
```

---

# 46. SOLID Principle — Single Responsibility

Bad:

```java
class AnalyticsService {
    validateEvent();
    publishKafka();
    aggregate();
    store();
    sendAlert();
    executeQuery();
}
```

This class has too many responsibilities.

Better:

```java
EventValidator
EventPublisher
StreamProcessor
Aggregator
MetricRepository
AlertEvaluator
QueryExecutor
```

Each class has one primary responsibility.

---

# 47. Open/Closed Principle

The aggregation engine should allow new aggregations without modifying existing logic.

Bad:

```java
if (type.equals("SUM")) {
    ...
} else if (type.equals("COUNT")) {
    ...
} else if (type.equals("AVG")) {
    ...
}
```

Better:

```java
interface Aggregator<T> {

    void add(T value);

    void merge(Aggregator<T> other);

    Object result();
}
```

Implementations:

```text
CountAggregator
SumAggregator
AverageAggregator
MinAggregator
MaxAggregator
DistinctCountAggregator
PercentileAggregator
```

Adding a new aggregator does not require changing the existing aggregation framework.

---

# 48. Liskov Substitution Principle

All aggregation implementations should satisfy the same contract.

```java
Aggregator<Double> aggregator;
```

It should be possible to substitute:

```java
SumAggregator
AverageAggregator
MinAggregator
MaxAggregator
```

without breaking the caller's assumptions.

---

# 49. Interface Segregation Principle

Avoid:

```java
interface Storage {
    save();
    read();
    delete();
    scan();
    stream();
    compact();
    backup();
}
```

A component may need only query capabilities.

Instead:

```java
interface EventWriter {
    void write(Event event);
}

interface EventReader {
    List<Event> read(Query query);
}

interface EventScanner {
    Stream<Event> scan(Query query);
}
```

---

# 50. Dependency Inversion Principle

Bad:

```java
class AnalyticsService {

    private final KafkaPublisher publisher =
        new KafkaPublisher();
}
```

Better:

```java
class AnalyticsService {

    private final EventPublisher publisher;

    AnalyticsService(EventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

Interface:

```java
interface EventPublisher {
    void publish(Event event);
}
```

Implementations:

```text
KafkaEventPublisher
PulsarEventPublisher
InMemoryEventPublisher
```

---

# 51. Aggregation Interface

```java
public interface Aggregator<T, R> {

    void add(T value);

    void merge(Aggregator<T, R> other);

    R result();

    void reset();
}
```

Example:

```java
public final class SumAggregator
        implements Aggregator<Double, Double> {

    private double sum;

    @Override
    public void add(Double value) {
        if (value != null) {
            sum += value;
        }
    }

    @Override
    public void merge(Aggregator<Double, Double> other) {
        sum += other.result();
    }

    @Override
    public Double result() {
        return sum;
    }

    @Override
    public void reset() {
        sum = 0;
    }
}
```

---

# 52. Aggregator Factory

```java
public interface AggregatorFactory {

    Aggregator<?, ?> create(AggregationType type);
}
```

Implementation:

```java
public final class DefaultAggregatorFactory
        implements AggregatorFactory {

    @Override
    public Aggregator<?, ?> create(AggregationType type) {

        return switch (type) {
            case SUM -> new SumAggregator();
            case COUNT -> new CountAggregator();
            case AVG -> new AverageAggregator();
            case MIN -> new MinAggregator();
            case MAX -> new MaxAggregator();
        };
    }
}
```

For a larger system, registry-based discovery can remove even the factory's growing switch.

---

# 53. Strategy Pattern — Windowing

```java
public interface WindowStrategy {

    WindowAssignment assign(Instant eventTime);
}
```

Implementations:

```text
TumblingWindowStrategy
SlidingWindowStrategy
SessionWindowStrategy
```

Processor:

```java
class StreamProcessor {

    private final WindowStrategy windowStrategy;

    void process(Event event) {
        WindowAssignment window =
            windowStrategy.assign(event.eventTime());

        // aggregate into window
    }
}
```

This makes windowing configurable.

---

# 54. Strategy Pattern — Partitioning

```java
public interface PartitionStrategy {

    String partitionKey(Event event);
}
```

Implementations:

```text
TenantPartitionStrategy
UserPartitionStrategy
HashPartitionStrategy
CompositePartitionStrategy
```

---

# 55. Chain of Responsibility — Processing Pipeline

A processing pipeline can use:

```text
Validate
   ↓
Filter
   ↓
Enrich
   ↓
Deduplicate
   ↓
Window
   ↓
Aggregate
```

Interface:

```java
public interface EventProcessor {

    ProcessingResult process(Event event);
}
```

Each processor can delegate to the next processor.

---

# 56. Query Abstraction

```java
public interface QueryExecutor {

    QueryResult execute(AnalyticsQuery query);
}
```

Implementations:

```text
HotStoreQueryExecutor
HistoricalQueryExecutor
DistributedQueryExecutor
CachedQueryExecutor
```

A query router can select the appropriate implementation.

---

# 57. Storage Abstraction

```java
public interface MetricRepository {

    void write(MetricRecord record);

    List<MetricRecord> query(AnalyticsQuery query);
}
```

Implementations:

```text
ClickHouseMetricRepository
DruidMetricRepository
PostgresMetricRepository
InMemoryMetricRepository
```

The domain does not depend on the concrete database.

---

# 58. Query Planner

```text
AnalyticsQuery
      |
      v
QueryValidator
      |
      v
QueryPlanner
      |
      +------ Cache hit ---> Result
      |
      v
Partition pruning
      |
      v
Aggregation selection
      |
      v
Storage selection
      |
      v
QueryExecutor
```

Planner responsibilities:

- validate query
- identify time range
- identify required dimensions
- identify aggregation
- choose data source
- optimize filters
- enforce tenant limits
- enforce maximum scan size

---

# 59. Alert Strategy

```java
public interface AlertCondition {

    boolean evaluate(MetricSnapshot snapshot);
}
```

Implementations:

```text
ThresholdCondition
PercentageCondition
RateCondition
AnomalyCondition
CompositeCondition
```

Example:

```java
class ThresholdCondition implements AlertCondition {

    private final double threshold;

    @Override
    public boolean evaluate(MetricSnapshot snapshot) {
        return snapshot.value() > threshold;
    }
}
```

---

# 60. Notification Abstraction

```java
public interface NotificationChannel {

    void send(Notification notification);
}
```

Implementations:

```text
EmailNotificationChannel
SmsNotificationChannel
PushNotificationChannel
WebhookNotificationChannel
SlackNotificationChannel
```

Alerting does not need to know how notifications are delivered.

---

# 61. Complete LLD Relationship

```text
EventIngestionService
        |
        +--> EventValidator
        |
        +--> EventPublisher
                    |
                    v
              MessageBroker
                    |
                    v
              StreamProcessor
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Filter   Enricher   Deduplicator
          |
          v
    WindowStrategy
          |
          v
    AggregatorFactory
          |
          v
      Aggregator
          |
          v
    MetricRepository
          |
          v
    Analytical Storage


QueryController
      |
      v
QueryService
      |
      v
QueryPlanner
      |
      +----> QueryCache
      |
      v
QueryExecutor
      |
      v
MetricRepository
```

---

# 62. Data Model

## Raw Event

```text
raw_events
-------------------------
event_id
tenant_id
event_type
event_time
ingestion_time
schema_version
source
payload
```

## Aggregated Metric

```text
metric_aggregates
-------------------------
tenant_id
metric_name
window_start
window_end
dimension
dimension_value
count
sum
min
max
avg
```

For a real analytical database, the physical schema should be designed around the actual query patterns rather than blindly copying an OLTP schema.

---

# 63. Materialized Views

Suppose users frequently query:

```text
revenue per city per minute
```

Instead of calculating it repeatedly:

```text
Raw Events
   |
   v
Aggregation
   |
   v
Materialized Metric
```

Example:

```text
2026-09-26 10:00
Hyderabad
orders = 1,200
revenue = 1,800,000
```

Dashboard queries become much cheaper.

---

# 64. Real-Time + Historical Query

A query may span:

```text
Last 1 hour
+
Last 2 years
```

Use a federation layer:

```text
                  Query
                    |
                    v
               Query Planner
                 /       \
                /         \
               v           v
          Hot Store     Data Lake
                \         /
                 \       /
                  v     v
                Merge
                  |
                  v
               Result
```

---

# 65. Cache Design

Cache key:

```text
hash(
    tenantId +
    query +
    timeRange +
    filters +
    dimensions
)
```

TTL:

```text
5–30 seconds
```

This works well for dashboards where users repeatedly request similar data.

Avoid caching queries with highly unique parameters when cache memory is limited.

---

# 66. Query Protection

A user could submit:

```text
SELECT *
FROM billions_of_events
GROUP BY 50 dimensions
```

Protect the system with:

```text
Maximum time range
Maximum dimensions
Maximum cardinality
Maximum result rows
Query timeout
Concurrent query limit
Tenant quota
```

For expensive reports, use asynchronous queries:

```text
POST /reports
       |
       v
Job Queue
       |
       v
Worker
       |
       v
Object Storage
       |
       v
Download result
```

---

# 67. Cardinality Problem

Some dimensions have huge cardinality:

```text
userId
requestId
deviceId
IP
```

Grouping by these can be extremely expensive.

Low-cardinality dimensions:

```text
country
city
status
eventType
```

should generally be preferred for interactive dashboards.

For high-cardinality analytics, specialized data structures such as approximate distinct-count algorithms may be used.

---

# 68. Approximate Analytics

Some metrics do not require exact results.

Examples:

```text
Unique users
Top-K products
Percentiles
```

Possible algorithms:

```text
HyperLogLog
Count-Min Sketch
T-Digest
Top-K sketches
```

This can reduce:

```text
Memory
CPU
Storage
Latency
```

at the cost of controlled approximation.

---

# 69. Backfill Architecture

Suppose a bug caused one day's aggregates to be incorrect.

Do not necessarily reprocess the entire system.

```text
Historical Raw Data
       |
       v
Backfill Job
       |
       v
Temporary Aggregates
       |
       v
Validation
       |
       v
Replace affected partitions
```

Keep raw immutable data whenever feasible.

---

# 70. Event Processing State

Streaming processors may maintain state:

```text
WindowState
 ├── windowId
 ├── aggregationState
 ├── eventCount
 ├── lastUpdated
 └── watermark
```

Example:

```text
Window 10:00–10:01

count = 10,000
sum = 4,500,000
```

State should be checkpointed.

---

# 71. Checkpointing

```text
Processor
    |
    +---- State
    |
    v
Checkpoint Store
```

On failure:

```text
New Processor
      |
      v
Load checkpoint
      |
      v
Resume from broker offset
```

---

# 72. Event Ordering

Global ordering across millions of events is expensive.

Usually guarantee ordering only within a key:

```text
userId = 100

Event A
Event B
Event C
```

Same partition:

```text
Partition 5
A -> B -> C
```

Events for another user can be processed in parallel.

---

# 73. Exactly-Once Design Discussion

For interview purposes, explain:

```text
Exactly-once is not simply:
"process every message once."

It means:
the observable effect of processing appears exactly once.
```

A practical design can combine:

```text
Stable event IDs
+
Deterministic aggregation
+
Checkpointing
+
Idempotent sink
+
Transactional/atomic commits where supported
```

---

# 74. API Design

## Ingestion

```http
POST /api/v1/events
```

## Batch Ingestion

```http
POST /api/v1/events/batch
```

## Query

```http
POST /api/v1/query
```

## Dashboard

```http
GET /api/v1/dashboards/{dashboardId}
```

## Alerts

```http
POST /api/v1/alerts
GET  /api/v1/alerts
DELETE /api/v1/alerts/{id}
```

## Query Job

```http
POST /api/v1/query-jobs
GET  /api/v1/query-jobs/{id}
```

---

# 75. Batch Ingestion

Instead of:

```text
1 HTTP request = 1 event
```

support:

```text
1 HTTP request = N events
```

Example:

```json
{
  "events": [
    {...},
    {...},
    {...}
  ]
}
```

This reduces:

```text
network overhead
connection overhead
broker requests
database writes
```

---

# 76. Batching Strategy

Processor should batch writes:

```text
Events
  |
  v
Buffer
  |
  +---- 10,000 events
  |
  v
Batch Write
```

Flush based on:

```text
batch size
OR
time threshold
```

Example:

```text
10,000 records
OR
100 ms
```

whichever comes first.

---

# 77. Compression

Use compression between components where appropriate.

Typical options:

```text
LZ4
Snappy
ZSTD
```

Benefits:

```text
Lower network usage
Lower storage usage
Higher throughput
```

Trade-off:

```text
CPU cost
```

---

# 78. Retention Policy

Example:

```text
Raw Kafka events:
    7 days

Hot analytical data:
    30 days

Warm data:
    1 year

Cold object storage:
    7 years
```

Actual values depend on business and compliance requirements.

---

# 79. Compaction

Older data can be compacted.

```text
Raw events
     |
     v
Daily aggregate
     |
     v
Delete/reduce old raw representation
```

But preserve raw data if replay/audit requirements require it.

---

# 80. API-Level SOLID Design

A clean dependency chain:

```text
Controller
    |
    v
Application Service
    |
    v
Domain Interfaces
    |
    v
Infrastructure Implementations
```

Example:

```text
QueryController
      |
      v
QueryApplicationService
      |
      v
QueryPlanner
      |
      v
QueryExecutor
      |
      v
MetricRepository
      |
      v
ClickHouseAdapter
```

The controller should not directly depend on ClickHouse/Kafka.

---

# 81. Clean Architecture

```text
             +-----------------------+
             |     Controllers       |
             +-----------+-----------+
                         |
                         v
             +-----------------------+
             | Application Services  |
             +-----------+-----------+
                         |
                         v
             +-----------------------+
             |      Domain           |
             | Entities + Interfaces |
             +-----------+-----------+
                         |
                         v
             +-----------------------+
             | Infrastructure        |
             | Kafka / DB / Redis     |
             +-----------------------+
```

Dependency direction:

```text
Infrastructure ---> Domain
Application    ---> Domain
Controllers    ---> Application
```

The domain should not depend on infrastructure details.

---

# 82. Concurrency

Important concurrency areas:

- multiple stream processors
- shared aggregation state
- query execution
- cache updates
- alert evaluation
- checkpointing

Prefer partition ownership over global locks.

Instead of:

```text
global lock
```

use:

```text
partition ownership
+
single-threaded state per partition
```

where the processing framework supports it.

This reduces contention.

---

# 83. Thread Safety

A dangerous design:

```java
class Aggregator {
    private double sum;

    void add(double value) {
        sum += value;
    }
}
```

If shared by multiple threads, updates can race.

Options:

```text
single owner
AtomicLong
LongAdder
locks
partition-local state
```

For high-throughput streaming, partition-local ownership is often preferable to a single shared lock.

---

# 84. Consistent Metric Updates

If an aggregate update contains:

```text
count++
sum += amount
```

the update should be atomic from the perspective of downstream readers.

Possible techniques:

```text
transaction
batch replacement
versioned aggregate
upsert
```

---

# 85. Versioned Aggregates

Example:

```text
metric = revenue
window = 10:00–10:01
version = 42
```

If a late event changes the result:

```text
version = 43
```

Consumers can detect updates.

---

# 86. Observability Architecture

```text
Services
   |
   +---- Metrics ----> Prometheus
   |
   +---- Logs -------> Log Platform
   |
   +---- Traces -----> OpenTelemetry
```

Dashboard:

```text
Ingestion Rate
Consumer Lag
Processing Latency
Error Rate
Query P95
Storage Latency
Cache Hit Ratio
```

---

# 87. Important SLOs

Example:

```text
Availability:
    99.9%

Ingestion:
    p95 < 500ms

Analytics freshness:
    < 5 seconds

Query:
    p95 < 2 seconds

Alert:
    < 10 seconds
```

Track SLO violations separately from raw infrastructure metrics.

---

# 88. Deployment Architecture

Example Kubernetes deployment:

```text
                    Load Balancer
                         |
                         v
                  Ingestion Gateway
                    /           \
                   v             v
              Gateway Pod    Gateway Pod
                   |
                   v
                 Kafka
          /        |        \
         v         v         v
   Processor   Processor   Processor
         \        |        /
                  v
          Analytical Store
             /        \
            v          v
         Replica      Replica
```

Query tier:

```text
Load Balancer
      |
      v
Query Service
   /   |   \
  v    v    v
 Q1   Q2   Q3
```

---

# 89. Autoscaling

Scale ingestion based on:

```text
CPU
request rate
queue depth
```

Scale processors based on:

```text
consumer lag
CPU
processing latency
```

For analytics:

```text
concurrent queries
CPU
memory
query latency
```

Consumer lag is often more useful than CPU alone for streaming workers.

---

# 90. What Happens During a Traffic Spike?

Normal:

```text
100K events/sec
```

Spike:

```text
1M events/sec
```

System:

```text
Producer
   |
   v
Gateway autoscaling
   |
   v
Kafka partitions absorb burst
   |
   v
Processor autoscaling
   |
   v
Batch writes
   |
   v
Analytical DB
```

If the downstream system cannot keep up, the broker provides temporary buffering while autoscaling catches up.

---

# 91. Multi-Region Architecture

```text
Region A
 Producers
    |
    v
 Kafka A
    |
    v
Processors A
    |
    v
Analytics A


Region B
 Producers
    |
    v
 Kafka B
    |
    v
Processors B
    |
    v
Analytics B
```

Cross-region replication can be added according to RPO/RTO requirements.

---

# 92. Global Query Routing

```text
User
 |
 v
Global Load Balancer
 |
 +----> Region A
 |
 +----> Region B
```

The query router can select:

```text
nearest region
or
region containing requested data
```

---

# 93. Cost Optimization

Major cost drivers:

```text
Data ingestion
Broker storage
Analytical storage
Object storage
Network transfer
Query CPU
Replication
```

Optimization:

1. Compression
2. Columnar storage
3. Partition pruning
4. Pre-aggregation
5. Tiered storage
6. Query cache
7. Retention policies
8. Sampling/approximation where acceptable
9. Autoscaling
10. Batch writes

---

# 94. Sampling

For extremely high-volume events:

```text
100M events
```

a dashboard may not require every event.

Example:

```text
1% sample
```

can be used for certain approximate metrics.

Do not use sampling when exact financial/audit metrics are required.

---

# 95. Security Threat Model

Potential threats:

```text
Fake events
Tenant data leakage
Query abuse
DDoS
Credential theft
Sensitive data exposure
Malicious dashboard queries
```

Controls:

```text
Authentication
Authorization
Rate limits
Input validation
Query limits
Encryption
Audit logs
Tenant filters
WAF/API gateway
```

---

# 96. Data Governance

Add metadata:

```text
owner
classification
retention
schema
PII flag
source
lineage
```

Sensitive fields should be:

```text
masked
hashed
tokenized
or excluded
```

depending on requirements.

---

# 97. Testing Strategy

## Unit Tests

Test:

```text
Aggregator
WindowStrategy
QueryPlanner
AlertCondition
PartitionStrategy
```

## Integration Tests

Test:

```text
Gateway -> Kafka
Kafka -> Processor
Processor -> Analytical DB
Query -> DB
```

## Contract Tests

Verify event schemas.

## Load Tests

Test:

```text
100K events/sec
500K events/sec
1M events/sec
```

## Failure Tests

Simulate:

```text
processor crash
broker failure
database timeout
network partition
duplicate events
late events
out-of-order events
```

---

# 98. Property-Based Testing

Aggregation logic is suitable for properties.

For sum:

```text
sum(A + B) = sum(A) + sum(B)
```

For count:

```text
count(A + B) = count(A) + count(B)
```

This is especially useful when implementing mergeable streaming aggregators.

---

# 99. Performance Test Matrix

Measure:

| Test | Metric |
|---|---|
| Ingestion | events/sec |
| Broker | throughput |
| Processor | events/sec |
| Aggregation | processing latency |
| DB write | rows/sec |
| Query | p50/p95/p99 |
| Cache | hit ratio |
| Alert | evaluation latency |
| Recovery | RTO |
| Replay | events/sec |

---

# 100. End-to-End Flow

Let's follow one event.

```text
1. Client creates ORDER_CREATED

2. Ingestion Gateway receives event

3. Authentication validates tenant

4. Schema validator validates payload

5. Gateway publishes event to Kafka

6. Kafka stores event

7. Stream processor consumes event

8. Event is deduplicated

9. Event is filtered

10. Event is enriched

11. Window strategy assigns event

12. Aggregator updates state

13. State is checkpointed

14. Aggregate is written to analytical DB

15. Alert evaluator checks metrics

16. Dashboard queries analytical store

17. Query result is returned
```

---

# 101. Detailed Example

Event:

```text
ORDER_CREATED
amount = ₹1,000
city = Hyderabad
eventTime = 10:00:20
```

Current window:

```text
10:00:00–10:01:00
```

Aggregator state:

```text
Hyderabad

count = 500
sum = ₹450,000
```

After event:

```text
count = 501
sum = ₹451,000
```

Dashboard:

```text
Hyderabad
Orders = 501
Revenue = ₹451,000
```

---

# 102. Interview Trade-Offs

A good system-design answer should not claim there is one universally correct architecture.

Discuss trade-offs.

## Kafka vs Direct API-to-DB

Kafka:

```text
+ buffering
+ replay
+ decoupling
+ scalable
- operational complexity
```

Direct DB:

```text
+ simple
- poor burst handling
- weak replay model
- tightly coupled
```

## Streaming vs Batch

Streaming:

```text
+ low latency
- operational complexity
```

Batch:

```text
+ simpler
+ cheaper
- higher latency
```

## Exact vs Approximate

Exact:

```text
+ accurate
- more expensive
```

Approximate:

```text
+ scalable
+ lower memory
- estimation error
```

---

# 103. Technology Mapping

A production implementation could look like:

| Layer | Example Technology |
|---|---|
| API | Spring Boot |
| Language | Java 21 |
| Broker | Kafka |
| Stream processing | Apache Flink / Kafka Streams |
| Hot analytics | ClickHouse |
| Cache | Redis |
| Cold storage | S3-compatible object storage |
| Metadata DB | PostgreSQL |
| Query API | Spring Boot |
| Dashboard | React |
| Metrics | Prometheus |
| Tracing | OpenTelemetry |
| Deployment | Kubernetes |
| Authentication | OAuth2/JWT |

These are examples; the interfaces should prevent the business/domain layer from being coupled to them.

---

# 104. Suggested Java Module Structure

```text
analytics-system/
│
├── analytics-domain/
│   ├── event/
│   ├── metric/
│   ├── aggregation/
│   ├── window/
│   ├── query/
│   └── alert/
│
├── analytics-application/
│   ├── ingestion/
│   ├── processing/
│   ├── query/
│   └── alert/
│
├── analytics-infrastructure/
│   ├── kafka/
│   ├── clickhouse/
│   ├── redis/
│   ├── s3/
│   └── postgres/
│
├── analytics-api/
│   ├── controller/
│   ├── dto/
│   └── security/
│
└── analytics-worker/
    ├── consumer/
    ├── processor/
    └── checkpoint/
```

---

# 105. Domain Interfaces

```java
public interface EventPublisher {
    PublishResult publish(Event event);
}
```

```java
public interface EventRepository {
    void save(Event event);
}
```

```java
public interface MetricRepository {
    void write(MetricRecord record);

    QueryResult query(AnalyticsQuery query);
}
```

```java
public interface QueryCache {
    Optional<QueryResult> get(QueryCacheKey key);

    void put(QueryCacheKey key, QueryResult result);
}
```

```java
public interface NotificationChannel {
    void send(Notification notification);
}
```

---

# 106. Application Service

```java
public final class EventIngestionApplicationService {

    private final EventValidator validator;
    private final EventPublisher publisher;

    public EventIngestionApplicationService(
            EventValidator validator,
            EventPublisher publisher) {

        this.validator = validator;
        this.publisher = publisher;
    }

    public PublishResult ingest(Event event) {

        validator.validate(event);

        return publisher.publish(event);
    }
}
```

The service coordinates use cases rather than implementing Kafka-specific behavior.

---

# 107. Event Processing Pipeline

```java
public interface EventStage {

    EventContext process(EventContext context);
}
```

Stages:

```text
ValidationStage
DeduplicationStage
FilteringStage
EnrichmentStage
WindowStage
AggregationStage
PersistenceStage
```

Pipeline:

```java
public final class EventPipeline {

    private final List<EventStage> stages;

    public EventPipeline(List<EventStage> stages) {
        this.stages = List.copyOf(stages);
    }

    public EventContext process(EventContext context) {

        EventContext current = context;

        for (EventStage stage : stages) {
            current = stage.process(current);
        }

        return current;
    }
}
```

This design makes the processing flow configurable and testable.

---

# 108. Query Object Model

```java
public record AnalyticsQuery(
        String tenantId,
        Instant from,
        Instant to,
        List<String> dimensions,
        List<Filter> filters,
        AggregationType aggregation,
        String metric
) {}
```

The query model should be independent of SQL.

Then:

```text
AnalyticsQuery
       |
       v
QueryPlanner
       |
       v
Storage-specific query
```

This prevents SQL from leaking into the domain.

---

# 109. Query Execution Flow

```text
POST /query
      |
      v
QueryController
      |
      v
QueryApplicationService
      |
      v
Authorization
      |
      v
QueryValidator
      |
      v
QueryPlanner
      |
      +---- Cache
      |
      v
QueryExecutor
      |
      v
MetricRepository
      |
      v
Result
```

---

# 110. Future Enhancements

## 110.1 Anomaly Detection

Detect:

```text
Revenue suddenly drops
Traffic suddenly increases
Error rate spikes
```

Pipeline:

```text
Metric
  |
  v
Feature Extraction
  |
  v
Anomaly Detector
  |
  v
Alert
```

---

# 111. Machine Learning

Future system:

```text
Streaming Events
       |
       v
Feature Pipeline
       |
       v
Feature Store
       |
       v
ML Model
       |
       v
Prediction
```

Use cases:

- fraud detection
- demand forecasting
- anomaly detection
- recommendation
- predictive maintenance

---

# 112. CEP — Complex Event Processing

Detect event sequences:

```text
LOGIN
  ↓
PASSWORD_CHANGE
  ↓
LARGE_PAYMENT
```

within:

```text
10 minutes
```

This can indicate a suspicious sequence.

Generalize the system with:

```text
EventPattern
PatternMatcher
SequenceWindow
```

---

# 113. User-Defined Analytics

Allow users to define:

```text
metric
dimensions
filters
windows
aggregation
alerts
```

Store these definitions as configuration rather than hard-coded Java.

---

# 114. Query-as-a-Service

Expose:

```text
Saved Queries
Saved Dashboards
Scheduled Reports
Exports
```

Architecture:

```text
Query Definition
      |
      v
Scheduler
      |
      v
Query Engine
      |
      v
Object Storage
      |
      v
Email/Webhook
```

---

# 115. Streaming SQL

Future capability:

```sql
SELECT
    city,
    COUNT(*)
FROM orders
WINDOW TUMBLING(1 MINUTE)
GROUP BY city;
```

The SQL parser can translate the query into the internal:

```text
AnalyticsQuery
+
WindowStrategy
+
Aggregator
```

This is a natural extension of the Strategy and Factory abstractions.

---

# 116. Self-Service Analytics

Users can build dashboards:

```text
Choose metric
      ↓
Choose dimensions
      ↓
Choose filters
      ↓
Choose time range
      ↓
Preview
      ↓
Save dashboard
```

The backend converts UI configuration into the same query model.

---

# 117. Governance Layer

Add:

```text
Data Catalog
Schema Registry
Lineage
PII classification
Retention policy
Data ownership
Access policy
```

This becomes important when the platform grows across many teams.

---

# 118. Advanced Architecture

A mature platform may look like:

```text
                           +------------------+
                           | Dashboard / BI   |
                           +--------+---------+
                                    |
                                    v
                           +------------------+
                           | Query Gateway    |
                           +--------+---------+
                                    |
                     +--------------+--------------+
                     |                             |
                     v                             v
                Query Cache                  Query Planner
                                                   |
                              +--------------------+-------------------+
                              |                    |                   |
                              v                    v                   v
                         Hot Store            Warm Store          Data Lake
                              |                    |                   |
                              +--------------------+-------------------+
                                                   |
                                                   v
                                              Query Result


Producers
    |
    v
Ingestion Gateway
    |
    v
Kafka / Pulsar
    |
    +-------------------+
    |                   |
    v                   v
Stream Processor     Raw Sink
    |
    +----------+
    |          |
    v          v
Metrics     Alerts
    |          |
    v          v
Hot Store   Notification
```

---

# 119. Interview Answer Sequence

When answering this question in an interview, follow this order.

## Step 1

Clarify:

```text
Users
Events/sec
Event size
Latency
Retention
Availability
```

## Step 2

State requirements.

## Step 3

Estimate scale.

## Step 4

Draw:

```text
Producer
→ Gateway
→ Kafka
→ Processor
→ Analytical DB
→ Query API
→ Dashboard
```

## Step 5

Explain partitioning.

## Step 6

Explain stream processing.

## Step 7

Explain windowing.

## Step 8

Explain analytical storage.

## Step 9

Explain query optimization.

## Step 10

Explain reliability.

## Step 11

Explain SOLID/LLD.

## Step 12

Discuss bottlenecks.

## Step 13

Discuss future enhancements.

---

# 120. Interview Follow-Up: "Why Not Just Use PostgreSQL?"

Answer:

A relational database can work for low-volume analytics, but at very high event rates and large historical data volumes, continuously scanning large datasets for aggregations creates contention with transactional workloads.

A dedicated analytical architecture provides:

```text
Columnar storage
Parallel scans
Compression
Partition pruning
Pre-aggregation
Horizontal scaling
Separate analytical workload
```

PostgreSQL can still be useful for metadata:

```text
tenants
users
dashboards
alert definitions
schemas
configuration
```

---

# 121. Interview Follow-Up: "Why Kafka?"

Answer:

Kafka decouples producers from consumers and provides:

```text
buffering
partitioning
consumer groups
replay
durability
horizontal scaling
```

This is especially useful when ingestion rate and processing rate temporarily differ.

---

# 122. Interview Follow-Up: "Why Not Store Everything in Redis?"

Redis is excellent for:

```text
cache
short-lived state
counters
real-time lookups
```

but using it as the complete long-term analytical store can become expensive and unsuitable for very large historical scans.

Use it selectively:

```text
Kafka -> Stream Processor -> Redis
                           |
                           v
                     dashboard cache
```

while durable analytical data lives elsewhere.

---

# 123. Interview Follow-Up: "What Is the Biggest Bottleneck?"

There may be several:

```text
1. Kafka partition throughput
2. Consumer lag
3. Aggregation state
4. Analytical DB writes
5. High-cardinality group-by
6. Expensive queries
7. Hot tenants
8. Network bandwidth
9. Object-storage scanning
```

The bottleneck should be identified from measured metrics rather than assumed.

---

# 124. Interview Follow-Up: "How Do You Scale?"

Scale independently:

```text
Ingestion:
    Gateway replicas

Messaging:
    More partitions/brokers

Processing:
    More consumers

Storage:
    Sharding/replicas

Query:
    More query workers

Cache:
    Redis cluster
```

This independent scaling is one of the key benefits of decoupled architecture.

---

# 125. Interview Follow-Up: "What If Kafka Is Down?"

The exact behavior depends on availability requirements.

Gateway can:

```text
reject requests
or
temporarily buffer
```

Do not acknowledge an event as durably accepted until the required durability point is reached.

Clients can retry using:

```text
eventId
```

to maintain idempotency.

---

# 126. Interview Follow-Up: "What If Analytical DB Is Down?"

The broker retains events.

```text
Kafka
  |
  v
Processor
  |
  X
Analytical DB unavailable
  |
  v
Retry / pause writes
```

Once storage recovers:

```text
Processor
   |
   v
Resume
   |
   v
Catch up backlog
```

The exact retention period must be large enough to cover the expected recovery window.

---

# 127. Interview Follow-Up: "How Do You Handle Duplicates?"

Use:

```text
eventId
```

and make downstream operations idempotent.

For aggregates:

```text
eventId
+
deduplication state
+
idempotent sink
```

Do not rely only on application-memory deduplication because it disappears after restart.

---

# 128. Interview Follow-Up: "How Do You Handle Late Events?"

Use:

```text
event time
+
watermarks
+
allowed lateness
+
state correction
```

Example:

```text
Window closes at 10:05

Watermark = 10:05
Allowed lateness = 2 minutes

Late events accepted until 10:07
```

---

# 129. Interview Follow-Up: "How Do You Handle a New Aggregation?"

Because of SOLID:

```text
Aggregator interface
        |
        +-- Sum
        +-- Count
        +-- Avg
        +-- Min
        +-- Max
        +-- NewAggregation
```

Add a new implementation and register it.

Existing processors should not require modification.

---

# 130. Interview Follow-Up: "How Do You Handle a New Database?"

Because of dependency inversion:

```text
MetricRepository
      |
      +---- ClickHouseAdapter
      +---- DruidAdapter
      +---- BigQueryAdapter
      +---- TestAdapter
```

The application depends on the interface, not the database.

---

# 131. Interview Follow-Up: "How Do You Handle a New Event Source?"

Use:

```text
EventSourceAdapter
```

Implement:

```text
KafkaSource
HttpSource
FileSource
DatabaseCDCSource
IoTSource
```

Normalize all of them into the common:

```text
Event
```

model.

---

# 132. Design Principles Summary

The final system should demonstrate:

### SOLID

```text
S — Single Responsibility
O — Open/Closed
L — Liskov Substitution
I — Interface Segregation
D — Dependency Inversion
```

### Other Principles

```text
Separation of concerns
Dependency injection
Programming to interfaces
Immutability where practical
Idempotency
Stateless services
Partition-local state
Configuration over hard-coding
```

---

# 133. Design Patterns Used

| Pattern | Usage |
|---|---|
| Strategy | Windowing, partitioning |
| Factory | Aggregator creation |
| Chain of Responsibility | Event pipeline |
| Adapter | Storage/event-source integrations |
| Repository | Persistence abstraction |
| Observer/Pub-Sub | Alerts/events |
| Builder | Complex query construction |
| Command | Async query/report jobs |
| Template Method | Common processing flow |
| Circuit Breaker | External service protection |

---

# 134. Final Architecture Checklist

Before finishing the interview, verify:

```text
[ ] Functional requirements
[ ] Non-functional requirements
[ ] Capacity estimation
[ ] API design
[ ] Event schema
[ ] Ingestion architecture
[ ] Message broker
[ ] Partition strategy
[ ] Stream processing
[ ] Windowing
[ ] Watermarks
[ ] Late events
[ ] Deduplication
[ ] Aggregation
[ ] Analytical storage
[ ] Hot/warm/cold storage
[ ] Query architecture
[ ] Query cache
[ ] Query optimization
[ ] Dashboard
[ ] Alerting
[ ] Security
[ ] Multi-tenancy
[ ] Rate limiting
[ ] Backpressure
[ ] Failure recovery
[ ] Replay
[ ] Checkpointing
[ ] Disaster recovery
[ ] Observability
[ ] HLD
[ ] LLD
[ ] Class design
[ ] SOLID
[ ] Design patterns
[ ] Testing
[ ] Cost optimization
[ ] Future enhancements
```

---

# 135. Final Interview Summary

A concise final answer can be:

> I would design the platform around an event-driven architecture. Producers send events through an authenticated ingestion gateway into a partitioned durable message broker. A horizontally scalable stream-processing layer consumes those events, performs validation, deduplication, enrichment, event-time windowing, and incremental aggregation. Recent aggregates are written to a columnar analytical store for low-latency dashboard queries, while immutable raw events are retained in object storage for replay, historical analytics, and backfills.
>
> A query service provides tenant-aware query validation, caching, partition pruning, pre-aggregation, and controlled access to the analytical stores. Alerts consume the same real-time metrics and trigger notification channels. The system uses at-least-once processing with idempotent operations where practical, checkpointed state, replayable events, dead-letter queues, and automated scaling.
>
> At the LLD level, the design uses interfaces for event publishers, repositories, aggregators, window strategies, query executors, and notification channels. Strategy, Factory, Adapter, Repository, and Chain of Responsibility patterns allow the system to evolve without modifying core business logic. Dependency inversion keeps the domain independent of Kafka, Redis, ClickHouse, object storage, or any specific infrastructure technology.

---

# 136. Recommended Implementation Roadmap

If implementing this system from scratch in Java, build it in stages.

## Phase 1 — Single-Process Prototype

Implement:

```text
Event
EventValidator
EventPipeline
Aggregator
WindowStrategy
InMemoryMetricRepository
QueryService
```

No Kafka initially.

---

## Phase 2 — Persistent Storage

Add:

```text
PostgreSQL
```

for metadata and a suitable analytical store for metrics.

---

## Phase 3 — Message Broker

Add:

```text
Kafka
```

and separate:

```text
Producer
Consumer
Processor
```

---

## Phase 4 — Streaming

Implement:

```text
TumblingWindow
SlidingWindow
Watermark
LateEventHandler
Checkpoint
```

---

## Phase 5 — Query Engine

Implement:

```text
QueryParser
QueryValidator
QueryPlanner
QueryExecutor
QueryCache
```

---

## Phase 6 — Dashboard

Build:

```text
React
   |
   v
Query API
   |
   v
Analytics Engine
```

Add:

```text
WebSocket/SSE
```

for real-time updates.

---

## Phase 7 — Alerts

Implement:

```text
AlertRule
AlertCondition
AlertEvaluator
NotificationChannel
```

---

## Phase 8 — Production Hardening

Add:

```text
Authentication
Authorization
Rate limiting
Multi-tenancy
Metrics
Tracing
Retries
Circuit breakers
DLQ
Replay
Backpressure
Autoscaling
Disaster recovery
```

---

# 137. The Most Important Concepts to Learn

For a system-design interview, focus particularly on:

```text
1. Event-driven architecture
2. Kafka partitioning
3. Consumer groups
4. Stream processing
5. Event time vs processing time
6. Watermarks
7. Tumbling/sliding/session windows
8. Late events
9. Stateful stream processing
10. Checkpointing
11. Exactly-once vs at-least-once
12. Idempotency
13. Columnar databases
14. Time-based partitioning
15. Materialized views
16. Query optimization
17. High-cardinality problems
18. Approximate algorithms
19. Hot/warm/cold storage
20. Backpressure
21. Replay/backfill
22. Multi-tenancy
23. SOLID
24. Distributed systems failure handling
25. Observability
```

---

# 138. One-Line Mental Model

Remember the architecture as:

```text
INGEST
   ↓
BUFFER
   ↓
PROCESS
   ↓
AGGREGATE
   ↓
STORE
   ↓
QUERY
   ↓
VISUALIZE
   ↓
ALERT
```

And the reliability model as:

```text
Durable Event
      +
Replay
      +
Idempotency
      +
Checkpoint
      +
Partitioning
      +
Horizontal Scaling
      =
Scalable Real-Time Analytics
```

---

# 139. Final System Design Diagram

```text
                         ┌─────────────────────┐
                         │   Web / Mobile / BI │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Query Gateway    │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                │
                    ▼                                ▼
             ┌─────────────┐                 ┌──────────────┐
             │ Query Cache │                 │ Query Planner│
             └─────────────┘                 └──────┬───────┘
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │ Analytical Store │
                                          └──────────────────┘


 Producers
    │
    ▼
┌───────────────────┐
│ Ingestion Gateway │
└─────────┬─────────┘
          │
          ▼
┌─────────────────────────────┐
│      Kafka / Event Log      │
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐│
│ │ P0 │ │ P1 │ │ P2 │ │ PN ││
│ └────┘ └────┘ └────┘ └────┘│
└──────────────┬──────────────┘
               │
               ▼
      ┌──────────────────┐
      │ Stream Processor │
      └────────┬─────────┘
               │
      ┌────────┼─────────┐
      │        │         │
      ▼        ▼         ▼
   Filter   Enrich   Deduplicate
      │        │         │
      └────────┼─────────┘
               ▼
       ┌───────────────┐
       │ Window Engine │
       └───────┬───────┘
               ▼
       ┌───────────────┐
       │ Aggregators   │
       └───────┬───────┘
               │
        ┌──────┴─────────┐
        ▼                ▼
┌───────────────┐  ┌──────────────┐
│ Hot Analytics │  │ Raw Data Lake│
└───────────────┘  └──────────────┘
        │
        ▼
┌─────────────────┐
│ Alert Evaluator │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ Email / SMS / Push / Webhook│
└─────────────────────────────┘
```

This architecture provides a complete path from raw event ingestion to real-time analytics while keeping the system horizontally scalable, replayable, fault tolerant, and extensible through SOLID-based low-level abstractions.
