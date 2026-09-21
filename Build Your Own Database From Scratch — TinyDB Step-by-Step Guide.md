# Build Your Own Database From Scratch — TinyDB Step-by-Step Guide

> **Goal:** Build a small but real database engine — **TinyDB** — from a raw on-disk log upward, in plain Java: a write-ahead log, crash recovery, indexing, a tiny query engine, a wire protocol, and a JDBC driver, so it can be pointed at directly from your own [MiniSpring](MiniSpring-Step-by-Step-Guide.md) `@Repository` beans through the connection pool built in the [MiniTomcat guide](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>). Along the way we implement real **ACID** transactions with **rollback**, **leader-follower replication**, per-key **TTL**, and a **DynamoDB-Streams-style change feed**.
>
> This guide assumes familiarity with the companion guides: [MiniSpring-Step-by-Step-Guide.md](MiniSpring-Step-by-Step-Guide.md) (the IoC container and `@Repository` beans TinyDB will plug into) and [Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) (the `MiniConnectionPool` pattern §32–§34 that pools TinyDB connections exactly like a JDBC pool pools any other database's connections).

---

# 1. What We Are Building

We are building **TinyDB** — a single-node, disk-backed, key/row-oriented database engine with a client-server protocol and a real JDBC driver. By the end of this guide you will have:

- A **write-ahead log (WAL)** and **crash recovery** that survives a process kill mid-write.
- An **on-disk storage engine** with a **hash index** (Bitcask-style) and, as an alternative, a simplified **B-Tree** and **LSM-Tree/SSTable** design, with the tradeoffs between them made explicit.
- A minimal **table/row model** and **query engine** (`GET`/`PUT`/`DELETE`/`SCAN`), reachable over a **wire protocol**, a **Java client driver**, and a real **`java.sql.Driver`** implementation.
- Real **ACID transactions**: atomicity via the WAL, isolation via both **row-level locking (2PL)** and **MVCC**, durability via `fsync` policy, and **rollback** via an **undo log**.
- **Leader-follower replication** by shipping the WAL, with follower bootstrap and a simplified failover story.
- Per-key **TTL** (lazy + active expiration), correctly interacting with the WAL and replication.
- A **change-event stream** — insert/modify/remove events with before/after images, a shard-iterator-style consumer API, and durable replay — modeled directly on DynamoDB Streams.
- A real, if small, **SQL layer** — both DML (`SELECT`/`INSERT`/`UPDATE`/`DELETE` with a `WHERE` clause) and DDL (`CREATE TABLE`/`DROP TABLE`) — parsed and translated into the same engine underneath, plus a **`mysql`-style terminal REPL** (`tinydb-cli`).
- Full wiring into MiniSpring: a `@Repository` bean using TinyDB through a pooled JDBC connection, and a `@Component` subscribed to TinyDB's change stream — **and** the same JDBC driver working unmodified from a terminal client and from a generic desktop SQL tool like **DBeaver**.

```text
Client / MiniSpring @Repository
        |
        v
TinyDB JDBC Driver  --(wire protocol)-->  TinyDB Server
                                                |
                                    +-----------+-----------+
                                    |                       |
                              Query Engine            Change Stream
                              (txn, locks/MVCC)        (publish events)
                                    |
                                    v
                              Write-Ahead Log  ---ships to--->  Follower(s)
                                    |
                                    v
                              Storage Engine (index + segment files on disk)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to answer, from first principles:

- Why does *every* durable database write to a log before touching its main data structures, and what specifically breaks if it doesn't?
- How does replaying a WAL after a crash reconstruct exactly the state the database had right before it died — no more, no less?
- What is the actual difference between a hash index, a B-Tree, and an LSM-Tree, and which write/read pattern favors each?
- What do atomicity, consistency, isolation, and durability each protect against, concretely, and which mechanism in TinyDB provides each one?
- Why does rollback need its own log (an *undo* log) that is conceptually the mirror image of the WAL (a *redo* log)?
- How does leader-follower replication turn "the leader's WAL" into "the follower's state," and what happens when a follower falls behind?
- Why is DynamoDB Streams (and CDC in general) just "expose the WAL to consumers," dressed up with a durable, resumable, ordered API?

---

# 3. Why Build a Database From Scratch? (Interview Motivation)

> **"Design and implement a minimal but durable key-value database that supports transactions with rollback, survives a crash without losing committed data, replicates to a standby node, expires keys after a TTL, and lets other services subscribe to a stream of changes. Explain the concrete mechanism behind each guarantee — don't just name it."**

This is one of the highest-signal **senior/staff backend and infrastructure interview questions** because a correct answer requires connecting several distinct areas that most engineers only use through an abstraction:

- **Durability engineering** — WAL, `fsync`, crash recovery — the same reasoning behind Postgres's WAL, Kafka's log segments, and Cassandra's commit log.
- **Data structures under I/O constraints** — why a B-Tree beats a hash index for range scans, why an LSM-Tree trades read amplification for write throughput.
- **Concurrency control** — locking vs MVCC, and the isolation-level tradeoffs every "why is my read seeing uncommitted data" bug traces back to.
- **Distributed systems fundamentals** — replication topologies, replica lag, and what "the leader crashed" actually requires to recover from safely.
- **API design for eventual consumers** — a change stream that must be *ordered*, *resumable after a crash*, and *at-least-once* — the same shape of problem as Kafka consumer offsets or DynamoDB Streams shard iterators.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | Matches the MiniSpring and MiniTomcat companion projects; `RandomAccessFile`/`FileChannel` give precise control over disk I/O. |
| Disk I/O | `java.io.RandomAccessFile`, `java.nio.channels.FileChannel`, `FileChannel.force(true)` for `fsync` | Direct control over exactly when bytes are guaranteed on disk — the crux of durability (§31). |
| Serialization | Hand-rolled binary format (length-prefixed records) | Same reasoning as the MiniSpring/MiniTomcat guides: seeing the bytes is the point. |
| Concurrency | `ReentrantReadWriteLock` per row/key, `ConcurrentHashMap` for in-memory structures | Enough to build real 2PL (§36) and MVCC (§38) without an external library. |
| Networking | The `MiniTomcat`-style accept loop + thread pool from the companion guide, reused for TinyDB's own server | Avoids re-deriving concurrency machinery already covered in depth there. |
| Client integration | A hand-written `java.sql.Driver` implementation | Lets MiniSpring's `@Repository` beans use TinyDB through the exact same `MiniConnectionPool` built in the MiniTomcat guide (§32–§34 there), unmodified. |
| Testing | JUnit 5, plus deliberate `kill -9`-style crash-recovery tests | A database's correctness claims are only as good as the crash tests that back them. |

---

# 5. Project Structure

```text
tinydb/
├── pom.xml
├── src/main/java/com/example/tinydb/
│   ├── storage/
│   │   ├── WriteAheadLog.java
│   │   ├── LogRecord.java
│   │   ├── SegmentFile.java
│   │   ├── HashIndex.java
│   │   ├── BTreeIndex.java
│   │   ├── SSTable.java
│   │   ├── Compactor.java
│   │   └── StorageEngine.java
│   ├── table/
│   │   ├── Row.java
│   │   ├── Schema.java
│   │   └── Table.java
│   ├── query/
│   │   ├── QueryEngine.java
│   │   └── Command.java              // GET / PUT / DELETE / SCAN / BEGIN / COMMIT / ROLLBACK
│   ├── txn/
│   │   ├── Transaction.java
│   │   ├── LockManager.java          // §35–§37
│   │   ├── UndoLog.java              // §33
│   │   ├── MvccStore.java            // §38
│   │   └── DeadlockDetector.java     // §37
│   ├── replication/
│   │   ├── ReplicationLeader.java
│   │   ├── ReplicationFollower.java
│   │   └── FollowerBootstrap.java
│   ├── ttl/
│   │   ├── ExpiryIndex.java
│   │   └── ExpiryReaper.java
│   ├── stream/
│   │   ├── ChangeEvent.java
│   │   ├── ChangeEventPublisher.java
│   │   ├── ShardIterator.java
│   │   └── StreamConsumer.java
│   ├── server/
│   │   └── TinyDbServer.java          // reuses MiniTomcat's accept loop + pool
│   ├── protocol/
│   │   ├── WireProtocol.java
│   │   └── TinyDbClient.java
│   └── jdbc/
│       ├── TinyDbDriver.java
│       ├── TinyDbConnection.java
│       ├── TinyDbStatement.java
│       └── TinyDbResultSet.java
└── src/test/java/com/example/tinydb/
    ├── WriteAheadLogTest.java
    ├── CrashRecoveryTest.java
    ├── TransactionRollbackTest.java
    ├── ReplicationTest.java
    ├── TtlExpiryTest.java
    └── ChangeStreamTest.java
```

---

# 6. Phase 1 — The Data Model: Key, Value, and a Table

TinyDB starts as a **key-value store** — the smallest useful data model, and the one every richer model (documents, rows, columns) is eventually built on top of.

```java
// table/Row.java — a value is just named columns; a pure KV store would use byte[] instead
public class Row {
    private final Map<String, Object> columns;
    public Row(Map<String, Object> columns) { this.columns = columns; }
    public Object get(String column) { return columns.get(column); }
    public Map<String, Object> asMap() { return columns; }
}
```

Every operation TinyDB supports reduces to one of four primitives on a `(table, key)` pair: `PUT(table, key, row)`, `GET(table, key)`, `DELETE(table, key)`, `SCAN(table, keyRange)`. Everything else in this guide — transactions, replication, TTL, streams — is built by wrapping or observing these four operations, never by bypassing them.

---

# 7. Phase 2 — A Naive In-Memory Store

```java
// storage/StorageEngine.java — version 1: correct, but forgets everything on restart
public class StorageEngine {
    private final Map<String, Map<String, Row>> tables = new ConcurrentHashMap<>();

    public void put(String table, String key, Row row) {
        tables.computeIfAbsent(table, t -> new ConcurrentHashMap<>()).put(key, row);
    }
    public Row get(String table, String key) {
        Map<String, Row> t = tables.get(table);
        return t == null ? null : t.get(key);
    }
    public void delete(String table, String key) {
        Map<String, Row> t = tables.get(table);
        if (t != null) t.remove(key);
    }
    public List<Row> scan(String table, String fromKey, String toKey) {
        Map<String, Row> t = tables.getOrDefault(table, Map.of());
        return t.entrySet().stream()
            .filter(e -> e.getKey().compareTo(fromKey) >= 0 && e.getKey().compareTo(toKey) < 0)
            .sorted(Map.Entry.comparingByKey())
            .map(Map.Entry::getValue)
            .toList();
    }
}
```

This is correct and fast — and completely useless as a *database*, because a JVM crash, a `kill -9`, or a power loss erases every row that was ever written. §8–§12 fix exactly that.

---

# 8. Why In-Memory Alone Is Not a Database

The defining property of a database — as opposed to a cache — is **durability**: once a write is acknowledged as committed, it must survive a crash. A `ConcurrentHashMap` gives you correctness under concurrent access, but nothing about it is ever guaranteed to reach a disk, and RAM contents are lost the instant the process dies.

The fix is not "write every change straight into on-disk data structures" either — updating a B-Tree or hash table *in place* on disk, for every single write, is slow (random I/O) and dangerous (a crash mid-update can leave the structure itself corrupted, not just incomplete). The standard answer, used by essentially every real database, is the **write-ahead log**.

---

# 9. Phase 3 — The Write-Ahead Log (WAL)

**The rule that defines a WAL:** before any change is applied to the "real" data structures (the in-memory map, the on-disk index), a record describing that change is first appended to a simple, sequential, append-only log file, and that append is confirmed durable (`fsync`, §31) *before* the write is acknowledged to the caller.

```text
PUT(users, "42", {name: "Alice"})
    |
    v
1. Append a LogRecord describing this write to the WAL file
2. fsync the WAL file (the bytes are now guaranteed to survive a crash)
3. ONLY NOW: apply the change to the in-memory map / index
4. ONLY NOW: acknowledge success to the caller
```

If the process crashes between steps 1 and 3, nothing is lost: step 5 (§12, crash recovery) simply replays the WAL from the beginning and re-derives the exact same in-memory state. If the process crashes *before* step 1, the write was never acknowledged, so its absence is correct, not a bug.

---

# 10. Anatomy of a WAL Record

Each record needs enough information to be replayed identically, plus a way to detect a **torn write** (a record partially written when the crash happened):

```text
+------------+------------+----------+-------+-------+-------+-----------+
| length (4B)| checksum(4B)| type(1B) | table | key   | value | (repeat)  |
+------------+------------+----------+-------+-------+-------+-----------+
```

```java
// storage/LogRecord.java
public class LogRecord {
    public enum Type { PUT, DELETE, TXN_BEGIN, TXN_COMMIT, TXN_ROLLBACK }

    private final Type type;
    private final long transactionId; // 0 for non-transactional writes
    private final String table;
    private final String key;
    private final byte[] value; // null for DELETE

    // constructor / getters omitted for brevity

    public byte[] serialize() {
        ByteArrayOutputStream body = new ByteArrayOutputStream();
        writeByte(body, type.ordinal());
        writeLong(body, transactionId);
        writeString(body, table);
        writeString(body, key);
        writeBytes(body, value == null ? new byte[0] : value);

        byte[] bodyBytes = body.toByteArray();
        int checksum = crc32(bodyBytes); // detects a torn/corrupted record on replay, §59

        ByteArrayOutputStream full = new ByteArrayOutputStream();
        writeInt(full, bodyBytes.length);
        writeInt(full, checksum);
        full.writeBytes(bodyBytes);
        return full.toByteArray();
    }
    // deserialize(...), writeByte/writeLong/writeString/writeBytes/crc32 omitted for brevity
}
```

The **length prefix** lets the reader know exactly how many bytes to read for this record even if a later record is corrupted; the **checksum** lets it detect that *this* record's bytes were only partially written before a crash — the single most important defense a WAL has against silently replaying garbage (§59 goes deeper on this).

---

# 11. Phase 4 — Writing to the WAL Before Applying a Change

```java
// storage/WriteAheadLog.java
public class WriteAheadLog {
    private final FileChannel channel;
    private final Object appendLock = new Object();

    public WriteAheadLog(Path path) throws IOException {
        this.channel = FileChannel.open(path, StandardOpenOption.CREATE, StandardOpenOption.WRITE, StandardOpenOption.READ);
        this.channel.position(channel.size()); // append at the end of any existing log
    }

    /** Appends and fsyncs one record; returns only once the bytes are durable on disk. */
    public long append(LogRecord record) throws IOException {
        byte[] bytes = record.serialize();
        synchronized (appendLock) {           // WAL appends must be strictly ordered on disk
            long offset = channel.position();
            channel.write(ByteBuffer.wrap(bytes));
            channel.force(true);              // fsync — see §31 for why this is non-negotiable, and its cost
            return offset;
        }
    }
}
```

```java
// storage/StorageEngine.java — version 2: durable
public Row put(String table, String key, Row row) throws IOException {
    wal.append(new LogRecord(LogRecord.Type.PUT, 0, table, key, serialize(row)));
    return applyPutInMemory(table, key, row); // only after the WAL append is durable
}
```

Notice the WAL append is a **single, strictly sequential, append-only write** — no seeking, no read-modify-write — which is exactly why it can afford to `fsync` on every write when a random-access update to a B-Tree page could not: sequential disk I/O is dramatically cheaper than random I/O, on both spinning disks and (to a lesser but still real degree) SSDs.

---

# 12. Phase 5 — Crash Recovery by Replaying the WAL

On startup, before serving any request, TinyDB reads the WAL from the beginning and re-applies every record to rebuild in-memory state exactly as it was the instant before the crash:

```java
// storage/StorageEngine.java
public void recover(Path walPath) throws IOException {
    try (WalReader reader = new WalReader(walPath)) {
        LogRecord record;
        while ((record = reader.readNext()) != null) { // stops cleanly at a truncated/corrupt tail, §59
            switch (record.getType()) {
                case PUT -> applyPutInMemory(record.getTable(), record.getKey(), deserialize(record.getValue()));
                case DELETE -> applyDeleteInMemory(record.getTable(), record.getKey());
                case TXN_BEGIN, TXN_COMMIT, TXN_ROLLBACK -> replayTransactionMarker(record); // §32–§33
            }
        }
    }
}
```

Because every acknowledged write was durably appended to the WAL *before* the caller was told it succeeded (§9), replaying the entire WAL from position zero reconstructs precisely the set of writes the database had promised to keep — no committed write is lost, and (crucially, once transactions are added in §32) no uncommitted write is resurrected either.

---

# 13. Phase 6 — Snapshots: Bounding WAL Replay Time

Replaying a WAL from the very beginning works, but its cost grows forever — after a year of writes, recovery would mean replaying a year of history. A **snapshot** is a periodic, complete dump of the current in-memory state to disk, paired with the WAL offset at the moment it was taken; recovery then only needs the snapshot plus whatever WAL records came *after* it.

```java
public void takeSnapshot() throws IOException {
    long walOffsetAtSnapshotTime = wal.currentPosition();
    Path snapshotFile = nextSnapshotPath();
    try (DataOutputStream out = new DataOutputStream(Files.newOutputStream(snapshotFile))) {
        out.writeLong(walOffsetAtSnapshotTime);
        for (var tableEntry : tables.entrySet()) {
            for (var rowEntry : tableEntry.getValue().entrySet()) {
                writeRow(out, tableEntry.getKey(), rowEntry.getKey(), rowEntry.getValue());
            }
        }
    }
    wal.truncateBefore(walOffsetAtSnapshotTime); // now-redundant WAL prefix can be safely discarded
}
```

```text
Recovery with a snapshot:
1. Load the newest snapshot file into memory directly (no replay needed for this part)
2. Replay ONLY the WAL records written after the snapshot's recorded offset
3. Done — the total recovery cost is now bounded by "time since the last snapshot," not "time since the database was created"
```

---

# 14. Phase 7 — Segment Files and Compaction

Rather than one WAL file growing forever, real log-structured engines (Bitcask, Cassandra's commit log, Kafka's segments) roll the log into fixed-size **segment files**, closing one and starting a new one once it hits a size threshold:

```java
public class SegmentFile {
    private final Path path;
    private final long maxSizeBytes;
    private FileChannel channel;
    private boolean sealed = false; // sealed segments are never appended to again, only read/compacted

    public boolean isFull() throws IOException {
        return channel.size() >= maxSizeBytes;
    }
}
```

Old, sealed segments accumulate multiple versions of the same key over time (every `PUT`/`DELETE` for `"42"` is a brand-new appended record, never an in-place edit). A background **compactor** periodically rewrites a set of old segments into a new one containing only the *latest* value per key (and dropping keys whose latest record is a tombstone `DELETE`), reclaiming the space consumed by superseded versions:

```java
public void compact(List<SegmentFile> oldSegments, SegmentFile newSegment) throws IOException {
    Map<String, LogRecord> latestPerKey = new LinkedHashMap<>();
    for (SegmentFile segment : oldSegments) {
        for (LogRecord record : segment.readAll()) {
            latestPerKey.put(record.getTable() + "/" + record.getKey(), record); // later records overwrite earlier ones
        }
    }
    for (LogRecord latest : latestPerKey.values()) {
        if (latest.getType() != LogRecord.Type.DELETE) newSegment.append(latest);
    }
    // old segments are only deleted once every reader has switched to the new segment + rebuilt index (§16)
}
```

This is the same fundamental idea as §13's snapshotting, applied per-segment instead of to the whole database at once — and it is the core mechanism that keeps an append-only log-structured store from growing without bound.

---

# 15. Why We Need an Index

Once data lives in segment files on disk rather than entirely in memory, `GET(table, key)` can no longer just be a `HashMap` lookup — naively, it would mean scanning every segment file from newest to oldest looking for the key, which is an `O(number of records ever written)` disk scan for every single read. An **index** is a separate, smaller structure that maps a key directly to *where its value lives* (a file + byte offset), turning that scan into a near-constant-time lookup.

---

# 16. Phase 8 — A Hash Index (Bitcask-Style)

The simplest useful index: an in-memory hash map from key to `(segment file, byte offset, record length)`, rebuilt at startup by scanning the segment files' record headers (not their full values):

```java
// storage/HashIndex.java
public class HashIndex {
    public record Location(SegmentFile segment, long offset, int length) { }

    private final Map<String, Location> index = new ConcurrentHashMap<>();

    public void put(String table, String key, Location location) {
        index.put(table + "/" + key, location);
    }
    public Location lookup(String table, String key) {
        return index.get(table + "/" + key);
    }
    public void remove(String table, String key) {
        index.remove(table + "/" + key);
    }

    /** Rebuilds the index by replaying every segment's records — the SAME traversal as WAL recovery (§12). */
    public static HashIndex rebuild(List<SegmentFile> segments) throws IOException {
        HashIndex index = new HashIndex();
        for (SegmentFile segment : segments) {
            for (var entry : segment.readAllWithOffsets()) { // (LogRecord, offset, length)
                if (entry.record().getType() == LogRecord.Type.DELETE) {
                    index.remove(entry.record().getTable(), entry.record().getKey());
                } else {
                    index.put(entry.record().getTable(), entry.record().getKey(),
                        new Location(segment, entry.offset(), entry.length()));
                }
            }
        }
        return index;
    }
}
```

A `GET` becomes: hash-index lookup (in memory, O(1)) → one seek + one read at the returned offset (one disk I/O, not a scan). This is exactly Bitcask's design (the storage engine behind Riak): trivially simple, extremely fast for point lookups, but it has a real limitation — a pure hash index cannot answer a **range query** (`SCAN` from key A to key B) without still touching every key, because hashing destroys key ordering.

---

# 17. Phase 9 — A B-Tree Index for Range Queries

A **B-Tree** keeps keys in sorted order across a tree of fixed-size disk pages, so both point lookups *and* range scans stay efficient — this is the index structure behind most traditional relational databases (Postgres, MySQL's InnoDB, SQLite).

```java
// storage/BTreeIndex.java — a deliberately simplified in-memory-node version (real B-Trees page to disk directly)
public class BTreeIndex {
    private static final int ORDER = 4; // max children per node — tiny on purpose, for a readable example

    static class Node {
        List<String> keys = new ArrayList<>();
        List<HashIndex.Location> values = new ArrayList<>(); // leaf-only in this simplified version
        List<Node> children = new ArrayList<>();
        boolean isLeaf = true;
    }

    private Node root = new Node();

    public HashIndex.Location lookup(String key) {
        Node node = root;
        while (!node.isLeaf) {
            int i = 0;
            while (i < node.keys.size() && key.compareTo(node.keys.get(i)) >= 0) i++;
            node = node.children.get(i);
        }
        int i = Collections.binarySearch(node.keys, key);
        return i >= 0 ? node.values.get(i) : null;
    }

    public List<HashIndex.Location> rangeScan(String fromKey, String toKey) {
        List<HashIndex.Location> results = new ArrayList<>();
        collectInRange(root, fromKey, toKey, results);
        return results; // walks leaves left-to-right, which are kept in sorted key order by insert()
    }

    public void insert(String key, HashIndex.Location location) {
        // Standard B-Tree insert: find the correct leaf, insert in sorted position,
        // split the node (and propagate a new separator key upward) if it exceeds ORDER - 1 keys.
        // Omitted here for brevity — the key property that matters for this guide is that
        // every leaf-level scan visits keys in sorted order, which HashIndex fundamentally cannot offer.
    }
    // collectInRange(...) omitted for brevity
}
```

The tradeoff versus §16's hash index: B-Tree lookups cost `O(log n)` page reads instead of `O(1)`, and **every write touches and potentially splits a page in place** — a random-I/O cost the pure-append hash index design never pays, but one that buys ordered range scans in return.

---

# 18. LSM-Trees vs B-Trees — Tradeoffs

A third design, the **Log-Structured Merge-Tree (LSM-Tree)**, is what TinyDB's segment-file-plus-compaction design (§14) is already most of the way toward — it's worth naming the tradeoff space explicitly, since "which index structure would you pick and why" is a near-guaranteed follow-up question:

| | Hash Index (§16) | B-Tree (§17) | LSM-Tree (§14, §19) |
|---|---|---|---|
| Writes | Append-only, O(1), no in-place page updates | In-place page update, may cascade into node splits | Append-only to an in-memory table, flushed sequentially — very fast |
| Point reads | O(1) via the hash map | O(log n) page reads | O(log n) *per level*, checked newest-to-oldest until found (§19) |
| Range scans | Not supported without a full scan | Efficient — keys are kept in sorted order on disk | Efficient per-level (sorted SSTables), but must merge across levels |
| Write amplification | Lowest — one sequential append per write | Moderate — page splits rewrite whole pages | Higher — the same key is rewritten again by every compaction pass it survives |
| Read amplification | Lowest for point reads | Low, single tree traversal | Higher — may need to check several levels before finding (or ruling out) a key |
| Real-world examples | Bitcask / Riak | Postgres, MySQL InnoDB, SQLite | Cassandra, RocksDB, LevelDB, HBase |

**The honest interview answer:** there is no universally "best" index — it's a write-throughput-vs-read-latency-vs-range-query tradeoff, and production systems increasingly let you choose per-workload (e.g. Postgres's B-Tree vs a purpose-built LSM engine like RocksDB embedded in a service that is write-heavy).

---

# 19. Phase 10 — SSTables and Leveled Compaction (Optional Deep Dive)

An **SSTable** ("Sorted String Table") is a segment file (§14) with one added constraint: its records are written in **sorted key order**, with a small in-memory sparse index of "key → offset" checkpoints every N records — the format that makes an LSM-Tree's per-level lookups efficient.

```text
Write path (LSM-Tree):
  PUT -> append to WAL (§9, unchanged) -> insert into an in-memory sorted table (a "memtable")
  When the memtable hits a size threshold -> flush it to disk as a new, immutable, sorted SSTable (a new "level 0" file)

Read path:
  GET(key) -> check the memtable first (newest data)
           -> then check level-0 SSTables, newest to oldest
           -> then level 1, level 2, ... until the key is found or every level is exhausted
           -> a Bloom filter (§64) in front of each SSTable lets most "not in this file" checks skip the disk read entirely
```

Compaction (§14's idea, generalized) periodically merges several sorted SSTables at one level into fewer, larger sorted SSTables at the next level — "leveled compaction" — which is exactly how RocksDB and LevelDB bound both space amplification and the number of levels a read must check. TinyDB's simpler single-directory compaction in §14 is this same idea with one flat level instead of a leveled hierarchy — a deliberate simplification, not a different mechanism.

---

# 20. Phase 11 — The Table Abstraction (Schema, Rows, Columns)

```java
// table/Schema.java
public class Schema {
    private final String tableName;
    private final String primaryKeyColumn;
    private final Map<String, Class<?>> columnTypes;

    public Schema(String tableName, String primaryKeyColumn, Map<String, Class<?>> columnTypes) {
        this.tableName = tableName;
        this.primaryKeyColumn = primaryKeyColumn;
        this.columnTypes = columnTypes;
    }

    public void validate(Row row) {
        for (var entry : row.asMap().entrySet()) {
            Class<?> expected = columnTypes.get(entry.getKey());
            if (expected == null) throw new IllegalArgumentException("Unknown column: " + entry.getKey());
            if (entry.getValue() != null && !expected.isInstance(entry.getValue())) {
                throw new IllegalArgumentException("Column " + entry.getKey() + " expects " + expected.getSimpleName());
            }
        }
        if (row.get(primaryKeyColumn) == null) {
            throw new IllegalArgumentException("Primary key " + primaryKeyColumn + " cannot be null");
        }
    }
}

// table/Table.java
public class Table {
    private final Schema schema;
    private final StorageEngine storage;

    public Table(Schema schema, StorageEngine storage) { this.schema = schema; this.storage = storage; }

    public void insert(Row row) throws IOException {
        schema.validate(row); // this is TinyDB's "C" in ACID — see §29
        String key = String.valueOf(row.get(schema.getPrimaryKeyColumn()));
        storage.put(schema.getTableName(), key, row);
    }
}
```

`Schema.validate(...)` is the first concrete appearance of **consistency** (§29) in this guide: a write that would violate a declared invariant (wrong type, missing primary key) is rejected *before* it ever reaches the WAL, rather than being durably recorded as a broken row.

---

# 21. Phase 12 — A Minimal Query Engine (GET / PUT / DELETE / SCAN)

```java
// query/Command.java
public sealed interface Command {
    record Put(String table, String key, Row row) implements Command { }
    record Get(String table, String key) implements Command { }
    record Delete(String table, String key) implements Command { }
    record Scan(String table, String fromKey, String toKey) implements Command { }
    record Begin() implements Command { }
    record Commit() implements Command { }
    record Rollback() implements Command { }
}

// query/QueryEngine.java
public class QueryEngine {
    private final StorageEngine storage;
    private final TransactionManager transactionManager; // §32

    public Object execute(Command command, Transaction currentTxn) throws IOException {
        return switch (command) {
            case Command.Put c -> { transactionManager.put(currentTxn, c.table(), c.key(), c.row()); yield null; }
            case Command.Get c -> transactionManager.get(currentTxn, c.table(), c.key());
            case Command.Delete c -> { transactionManager.delete(currentTxn, c.table(), c.key()); yield null; }
            case Command.Scan c -> storage.scan(c.table(), c.fromKey(), c.toKey());
            case Command.Begin c -> transactionManager.begin();
            case Command.Commit c -> { transactionManager.commit(currentTxn); yield null; }
            case Command.Rollback c -> { transactionManager.rollback(currentTxn); yield null; }
        };
    }
}
```

Every command TinyDB understands is represented as one of these small, closed set of types — a `sealed interface` with a Java `switch` pattern-match is a natural fit, and it makes adding a new command (say, a future `INCREMENT`) a compiler-checked exercise: the switch stops compiling until every command type is handled.

---

# 22. Phase 13 — A Tiny Wire Protocol

A simple length-prefixed binary protocol, deliberately similar in spirit to Redis's RESP — text-friendly enough to debug with `nc`, structured enough to parse unambiguously:

```text
Request:  <command-byte> <arg-count (1B)> [<arg-length (4B)><arg-bytes>]*
Response: <status-byte: 0=OK, 1=ERROR> <payload-length (4B)> <payload-bytes>
```

```java
// protocol/WireProtocol.java
public class WireProtocol {
    public Command decode(DataInputStream in) throws IOException {
        byte commandByte = in.readByte();
        int argCount = in.readUnsignedByte();
        String[] args = new String[argCount];
        for (int i = 0; i < argCount; i++) {
            int length = in.readInt();
            byte[] bytes = in.readNBytes(length);
            args[i] = new String(bytes, StandardCharsets.UTF_8);
        }
        return switch (commandByte) {
            case 1 -> new Command.Put(args[0], args[1], parseRow(args[2]));
            case 2 -> new Command.Get(args[0], args[1]);
            case 3 -> new Command.Delete(args[0], args[1]);
            case 4 -> new Command.Scan(args[0], args[1], args[2]);
            case 5 -> new Command.Begin();
            case 6 -> new Command.Commit();
            case 7 -> new Command.Rollback();
            default -> throw new IllegalArgumentException("Unknown command byte: " + commandByte);
        };
    }

    public void encodeResult(DataOutputStream out, Object result) throws IOException {
        byte[] payload = serializeResult(result); // e.g. JSON, reusing MiniSpring's JsonSerializer
        out.writeByte(0); // OK
        out.writeInt(payload.length);
        out.write(payload);
    }
    // encodeError(...), parseRow(...), serializeResult(...) omitted for brevity
}
```

---

# 23. Phase 14 — The TinyDB Server

TinyDB's server deliberately reuses the concurrency machinery already built in the companion guide instead of re-deriving it: an accept loop, a bounded thread pool, and per-connection handling, exactly as in [MiniTomcat §16–§21](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>).

```java
// server/TinyDbServer.java
public class TinyDbServer {
    private final ServerSocket serverSocket;
    private final ServerThreadPool pool;        // MiniTomcat §16
    private final WireProtocol protocol = new WireProtocol();
    private final QueryEngine queryEngine;

    public TinyDbServer(int port, QueryEngine queryEngine, int coreThreads, int maxThreads, int queueCapacity) throws IOException {
        this.serverSocket = new ServerSocket(port);
        this.pool = new ServerThreadPool(coreThreads, maxThreads, queueCapacity);
        this.queryEngine = queryEngine;
    }

    public void start() {
        new Thread(this::acceptLoop, "tinydb-acceptor").start();
    }

    private void acceptLoop() {
        while (true) {
            try {
                Socket client = serverSocket.accept();
                pool.submit(() -> handleConnection(client));
            } catch (IOException e) {
                System.err.println("Accept failed: " + e.getMessage());
            }
        }
    }

    private void handleConnection(Socket client) {
        Transaction connectionTxn = null; // one transaction at a time per connection, like a real JDBC Connection
        try (client;
             DataInputStream in = new DataInputStream(new BufferedInputStream(client.getInputStream()));
             DataOutputStream out = new DataOutputStream(new BufferedOutputStream(client.getOutputStream()))) {
            while (true) {
                Command command = protocol.decode(in);
                if (command instanceof Command.Begin) connectionTxn = (Transaction) queryEngine.execute(command, null);
                Object result = queryEngine.execute(command, connectionTxn);
                if (command instanceof Command.Commit || command instanceof Command.Rollback) connectionTxn = null;
                protocol.encodeResult(out, result);
                out.flush();
            }
        } catch (EOFException clientClosed) {
            // normal disconnect
        } catch (IOException e) {
            System.err.println("Connection error: " + e.getMessage());
        }
    }
}
```

One TCP connection maps to at most one in-flight transaction at a time — the same constraint a real `java.sql.Connection` has, and exactly what §26's `TinyDbConnection` relies on.

---

# 24. Phase 15 — A Java Client Driver

Before wrapping this in JDBC, a plain client makes the wire protocol concrete:

```java
// protocol/TinyDbClient.java
public class TinyDbClient implements AutoCloseable {
    private final Socket socket;
    private final DataInputStream in;
    private final DataOutputStream out;
    private final WireProtocol protocol = new WireProtocol();

    public TinyDbClient(String host, int port) throws IOException {
        this.socket = new Socket(host, port);
        this.in = new DataInputStream(new BufferedInputStream(socket.getInputStream()));
        this.out = new DataOutputStream(new BufferedOutputStream(socket.getOutputStream()));
    }

    public void put(String table, String key, Row row) throws IOException {
        sendCommand(new Command.Put(table, key, row));
        readResult();
    }

    public Row get(String table, String key) throws IOException {
        sendCommand(new Command.Get(table, key));
        return (Row) readResult();
    }

    public void beginTransaction() throws IOException { sendCommand(new Command.Begin()); readResult(); }
    public void commit() throws IOException { sendCommand(new Command.Commit()); readResult(); }
    public void rollback() throws IOException { sendCommand(new Command.Rollback()); readResult(); }

    @Override public void close() throws IOException { socket.close(); }
    // sendCommand(...), readResult(...) omitted for brevity — encode/decode via WireProtocol
}
```

---

# 25. Phase 16 — A JDBC Driver for TinyDB

Implementing (a useful subset of) `java.sql.Driver` is what lets TinyDB slot into **any** JDBC-based codebase — including MiniSpring's `@Repository` beans — without those beans knowing TinyDB isn't Postgres or MySQL underneath.

```java
// jdbc/TinyDbDriver.java
public class TinyDbDriver implements java.sql.Driver {
    static {
        try {
            java.sql.DriverManager.registerDriver(new TinyDbDriver());
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public boolean acceptsURL(String url) { return url.startsWith("jdbc:tinydb://"); }

    @Override
    public Connection connect(String url, Properties info) throws SQLException {
        if (!acceptsURL(url)) return null; // per the java.sql.Driver contract
        URI uri = URI.create(url.substring("jdbc:".length())); // "tinydb://host:port/dbname"
        try {
            TinyDbClient client = new TinyDbClient(uri.getHost(), uri.getPort());
            return new TinyDbConnection(client);
        } catch (IOException e) {
            throw new SQLException("Failed to connect to TinyDB at " + url, e);
        }
    }
    // getMajorVersion()/getMinorVersion()/jdbcCompliant()/etc. omitted — required by the interface, mostly boilerplate
}
```

```java
// jdbc/TinyDbConnection.java — a minimal but real java.sql.Connection
public class TinyDbConnection implements Connection {
    private final TinyDbClient client;
    private boolean autoCommit = true;

    public TinyDbConnection(TinyDbClient client) { this.client = client; }

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        this.autoCommit = autoCommit;
        try { if (!autoCommit) client.beginTransaction(); } catch (IOException e) { throw new SQLException(e); }
    }

    @Override
    public void commit() throws SQLException {
        try { client.commit(); } catch (IOException e) { throw new SQLException(e); }
    }

    @Override
    public void rollback() throws SQLException {
        try { client.rollback(); } catch (IOException e) { throw new SQLException(e); }
    }

    @Override
    public Statement createStatement() { return new TinyDbStatement(client); }

    @Override
    public void close() throws SQLException {
        try { client.close(); } catch (IOException e) { throw new SQLException(e); }
    }
    // The remaining ~40 java.sql.Connection methods are stubbed to throw
    // SQLFeatureNotSupportedException — a real driver implements only what it needs,
    // exactly as MiniSpring's own guide only implemented the ApplicationContext surface it needed.
}
```

`TinyDbStatement`/`TinyDbResultSet` follow the same pattern: translate `executeQuery("...")`-shaped calls into `Command.Get`/`Command.Scan` wire calls, and wrap the returned `Row`s in a minimal `ResultSet` that supports `next()`/`getString(...)`/`getObject(...)`.

---

# 26. Phase 17 — Wiring TinyDB Into MiniSpring via a Repository Bean

This is the payoff: because TinyDB now speaks real JDBC, it drops into the *exact* `MiniConnectionPool` built in the MiniTomcat companion guide ([§32–§34](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>)) completely unmodified — that pool was written against `java.sql.Connection`, never against any specific database.

```java
@Component
public class TinyDbConfig {
    @Bean
    public MiniConnectionPool tinyDbConnectionPool() throws SQLException {
        return new MiniConnectionPool("jdbc:tinydb://localhost:9090/pureeats", "app", "", /* size */ 10);
    }
}

@Repository
public class TinyDbUserRepository implements UserRepository {
    private final MiniConnectionPool pool;

    @Autowired
    public TinyDbUserRepository(MiniConnectionPool pool) { this.pool = pool; }

    @Override
    public User findById(long id) {
        try (Connection connection = pool.borrow();
             PreparedStatement ps = connection.prepareStatement("GET users " + id)) {
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? mapRow(rs) : null;
            }
        } catch (SQLException | InterruptedException e) {
            throw new RuntimeException("TinyDB query failed", e);
        }
    }

    @Override
    public void save(User user) {
        try (Connection connection = pool.borrow()) {
            connection.setAutoCommit(false);     // -> Command.Begin over the wire (§25)
            try (PreparedStatement ps = connection.prepareStatement("PUT users " + user.getId())) {
                ps.executeUpdate();
                connection.commit();             // -> Command.Commit (§25) -> WAL fsync (§9) -> durable
            } catch (SQLException e) {
                connection.rollback();           // -> Command.Rollback (§25) -> undo log replay (§33)
                throw e;
            }
        } catch (SQLException | InterruptedException e) {
            throw new RuntimeException("TinyDB write failed", e);
        }
    }
    // mapRow(...) omitted for brevity
}
```

Not one line of MiniSpring's `ApplicationContext`, `DefaultBeanFactory`, or `HandlerMapping` needed to change — TinyDB entered the system exactly the way a real production database would, through the JDBC boundary.

---

# 27. What ACID Actually Means

| Property | Guarantees | What breaks without it |
|---|---|---|
| **A**tomicity | A transaction's writes are all-or-nothing — never partially applied. | A crash mid-transaction leaves some writes committed and others missing, corrupting whatever invariant spans them (e.g. half of a money transfer). |
| **C**onsistency | Every transaction moves the database from one valid state to another, per its declared constraints. | A schema/type violation (§20) or a broken invariant gets durably persisted as if it were valid data. |
| **I**solation | Concurrent transactions don't observe each other's uncommitted (or, at higher levels, even committed-but-later) changes. | Two concurrent transactions can read each other's half-finished work, producing results neither transaction alone would ever have produced. |
| **D**urability | Once committed, a transaction's writes survive any subsequent crash. | An acknowledged write disappears on restart — the exact failure §9's WAL exists to prevent. |

The rest of this guide implements each letter with a specific, concrete mechanism: **A** via the WAL + undo log (§28, §33), **C** via schema validation (§20, §29), **I** via locking and/or MVCC (§35, §38), **D** via `fsync` (§9, §31) — "ACID" is not one feature, it's four separate engineering problems that happen to share an acronym.

---

# 28. Atomicity — All or Nothing

A transaction writing to three rows must never leave the database with only two of them changed. TinyDB achieves this by treating an entire transaction as **one deferred batch of WAL records**, all sharing a `transactionId`, bracketed by `TXN_BEGIN` and `TXN_COMMIT` markers (§10):

```text
TXN_BEGIN(txnId=7)
PUT(txnId=7, users, "1", {...})
PUT(txnId=7, users, "2", {...})
TXN_COMMIT(txnId=7)
```

The critical rule for **crash recovery** (§12) to respect atomicity: a record's changes are only replayed into the in-memory/index state if a matching `TXN_COMMIT(txnId)` record was *also* found in the WAL. A crash after the two `PUT`s but before `TXN_COMMIT` means recovery **skips both writes entirely** — exactly the "all or nothing" guarantee, reconstructed purely by re-reading the log.

```java
public void recover(Path walPath) throws IOException {
    Set<Long> committedTxnIds = scanForCommittedTransactionIds(walPath); // first pass: find every TXN_COMMIT
    try (WalReader reader = new WalReader(walPath)) {
        LogRecord record;
        while ((record = reader.readNext()) != null) {
            if (record.getTransactionId() != 0 && !committedTxnIds.contains(record.getTransactionId())) {
                continue; // belongs to a transaction that never committed — skip, per atomicity
            }
            applyRecord(record);
        }
    }
}
```

---

# 29. Consistency — Enforcing Invariants

Consistency is the property with the least dedicated machinery, because it is enforced *before* a write is even proposed to the WAL: `Schema.validate(row)` (§20) already rejects a type violation or a missing primary key before it becomes a durable fact. A richer TinyDB could add cross-row invariants (foreign-key-style references, uniqueness constraints) — each one is simply another check that must pass before `TXN_COMMIT` is allowed to be written, using exactly the same "reject before it's durable" principle.

---

# 30. Isolation Levels, From Weakest to Strongest

| Level | Prevents | Still allows |
|---|---|---|
| Read Uncommitted | Nothing | Dirty reads — seeing another transaction's uncommitted writes |
| Read Committed | Dirty reads | Non-repeatable reads — re-reading the same row twice in one transaction can see different (but each individually committed) values |
| Repeatable Read | Dirty reads, non-repeatable reads | Phantom reads — a range scan re-run in the same transaction can see newly committed rows that didn't exist on the first scan |
| Serializable | All of the above | Nothing — equivalent to transactions running one at a time, in some order |

TinyDB implements **Read Committed** with row-level locking (§35–§36) and **Snapshot Isolation** (very close to Repeatable Read, and what many real databases label "Serializable" even though it technically permits a narrow class of anomalies) with MVCC (§38) — giving a concrete choice between the two dominant real-world isolation mechanisms, not just a table of definitions.

---

# 31. Durability — fsync and the Group Commit Tradeoff

`FileChannel.force(true)` (§11) is what actually turns "written to the OS" into "guaranteed on physical disk" — without it, `channel.write(...)` may only place bytes in the OS page cache, which a kernel panic or power loss can lose just as completely as an unflushed JVM buffer. This is not optional for a durability claim to be true.

It is, however, expensive: `fsync` is one of the slowest common operations in a hot write path (often single-digit milliseconds). **Group commit** is the standard mitigation — batch several transactions' WAL appends together and issue one `fsync` for the whole batch, trading a small amount of added latency (waiting a few milliseconds to see if more commits arrive) for dramatically higher throughput:

```java
public class GroupCommitWriter {
    private final BlockingQueue<PendingWrite> pending = new LinkedBlockingQueue<>();

    record PendingWrite(byte[] bytes, CompletableFuture<Void> onDurable) { }

    // A single background thread drains the queue and batches fsyncs — see §62 for the full design
    private void writerLoop() throws IOException, InterruptedException {
        while (true) {
            List<PendingWrite> batch = new ArrayList<>();
            batch.add(pending.take());               // block for at least one write
            pending.drainTo(batch, 63);                // opportunistically grab more, up to a small cap
            for (PendingWrite w : batch) channel.write(ByteBuffer.wrap(w.bytes()));
            channel.force(true);                       // ONE fsync durs the WHOLE batch
            batch.forEach(w -> w.onDurable().complete(null));
        }
    }
}
```

This is the exact mechanism behind Postgres's and Kafka's own "group commit" / batched-fsync designs, and it is the first entry on §62's optimization list because it is usually the single biggest throughput lever in a WAL-based system.

---

# 32. Phase 18 — A Transaction API (BEGIN / COMMIT / ROLLBACK)

```java
// txn/Transaction.java
public class Transaction {
    private final long id;
    private final List<LogRecord> pendingWrites = new ArrayList<>(); // buffered until COMMIT, §28
    private final UndoLog undoLog = new UndoLog();                    // §33
    private final Set<String> heldLocks = new LinkedHashSet<>();      // §35–§36
    private TransactionState state = TransactionState.ACTIVE;

    public long getId() { return id; }
    // ... getters/setters for the fields above
}

// txn/TransactionManager.java
public class TransactionManager {
    private final WriteAheadLog wal;
    private final StorageEngine storage;
    private final LockManager lockManager;      // §35
    private final AtomicLong nextTxnId = new AtomicLong(1);

    public Transaction begin() throws IOException {
        Transaction txn = new Transaction(nextTxnId.getAndIncrement());
        wal.append(new LogRecord(LogRecord.Type.TXN_BEGIN, txn.getId(), null, null, null));
        return txn;
    }

    public void put(Transaction txn, String table, String key, Row newRow) throws IOException {
        lockManager.acquireExclusive(txn, table, key); // §35 — blocks if another txn holds this key
        Row previousRow = storage.get(table, key);
        txn.getUndoLog().recordBeforeImage(table, key, previousRow); // §33 — save the "before" for rollback
        txn.getPendingWrites().add(new LogRecord(LogRecord.Type.PUT, txn.getId(), table, key, serialize(newRow)));
    }

    public void commit(Transaction txn) throws IOException {
        for (LogRecord record : txn.getPendingWrites()) wal.append(record); // still redo-only until the marker below
        wal.append(new LogRecord(LogRecord.Type.TXN_COMMIT, txn.getId(), null, null, null)); // atomic "point of no return"
        for (LogRecord record : txn.getPendingWrites()) storage.applyInMemory(record); // now safe to apply
        lockManager.releaseAll(txn);
    }

    public void rollback(Transaction txn) throws IOException {
        txn.getUndoLog().undoAll(storage);          // §33 — restore every before-image, newest first
        wal.append(new LogRecord(LogRecord.Type.TXN_ROLLBACK, txn.getId(), null, null, null));
        lockManager.releaseAll(txn);
    }
}
```

Note the ordering in `commit`: **every** pending WAL record is appended, *then* the single `TXN_COMMIT` marker is appended and fsynced, and *only then* are the changes applied to in-memory state. If a crash happens at any point before that `TXN_COMMIT` record is durable, §28's recovery logic discards the whole transaction — atomicity holds even across a crash mid-commit.

---

# 33. Phase 19 — Implementing Rollback With an Undo Log

Rollback needs to answer one question: "what was every value *before* this transaction touched it?" — the **undo log** is exactly that answer, recorded eagerly as each write happens, not reconstructed after the fact:

```java
// txn/UndoLog.java
public class UndoLog {
    public record BeforeImage(String table, String key, Row previousValueOrNull) { }

    private final Deque<BeforeImage> images = new ArrayDeque<>(); // stack: undo in REVERSE order of writes

    public void recordBeforeImage(String table, String key, Row previousValueOrNull) {
        images.push(new BeforeImage(table, key, previousValueOrNull));
    }

    public void undoAll(StorageEngine storage) throws IOException {
        while (!images.isEmpty()) {
            BeforeImage image = images.pop(); // LIFO: undo the LAST write first
            if (image.previousValueOrNull() == null) {
                storage.applyDeleteInMemory(image.table(), image.key()); // it didn't exist before -> remove it
            } else {
                storage.applyPutInMemory(image.table(), image.key(), image.previousValueOrNull()); // restore it
            }
        }
    }
}
```

Two details make this correct rather than merely plausible:

- **LIFO undo order matters.** If a transaction writes key `"A"` twice, the undo log must restore the *original* value (from before the first write), not the intermediate one — popping the stack in reverse write order guarantees that, since the last-pushed image for `"A"` is whatever preceded the very first write to it within this transaction... actually the *first* push for a given key already holds the true original value, and later pushes for the same key would need to be skipped or the stack walked fully; a production implementation keys the undo log by `(table, key)` and only keeps the *earliest* before-image per key within a transaction to get this right cheaply. The ArrayDeque sketch above is correct as long as each key is written at most once per transaction — worth calling out explicitly as a gap to close if extending this further.
- **Rollback never touches the WAL's PUT/DELETE records that were never committed** — because `commit()` (§32) only appends pending writes right before the `TXN_COMMIT` marker, a rolled-back transaction's writes were only ever buffered in `Transaction.pendingWrites`, in memory, and are simply discarded; the only WAL record rollback itself appends is the `TXN_ROLLBACK` marker, for observability and for §53's change stream.

---

# 34. Redo Log vs Undo Log — Why Real Databases Use Both

| | Redo log (the WAL, §9) | Undo log (§33) |
|---|---|---|
| Records | "What should be (re)applied if this commit needs to be replayed" | "What to restore if this transaction is abandoned" |
| Used for | Crash recovery — reconstructing committed state (§12, §28) | Rollback — undoing an in-progress or explicitly aborted transaction (§33) |
| Direction | Forward: apply the new value | Backward: restore the previous value |
| Lifetime | Kept until a snapshot/compaction supersedes it (§13–§14) | Only needed for the lifetime of the transaction — discarded on commit |

Real databases (Postgres, MySQL/InnoDB, Oracle) universally implement both, for exactly the reasons split out here: the redo log answers "how do we survive a crash," the undo log answers "how do we support a transaction changing its mind" — two different failure modes, two different logs, even though both are conceptually "logs of changes."

---

# 35. Phase 20 — Row-Level Locking for Isolation

```java
// txn/LockManager.java
public class LockManager {
    private final Map<String, ReentrantReadWriteLock> locks = new ConcurrentHashMap<>();
    private final Map<String, Long> ownerByKey = new ConcurrentHashMap<>(); // for deadlock detection, §37

    public void acquireExclusive(Transaction txn, String table, String key) {
        String lockKey = table + "/" + key;
        ReentrantReadWriteLock lock = locks.computeIfAbsent(lockKey, k -> new ReentrantReadWriteLock());
        lock.writeLock().lock(); // blocks here if another transaction holds this row
        ownerByKey.put(lockKey, txn.getId());
        txn.getHeldLocks().add(lockKey);
    }

    public void releaseAll(Transaction txn) {
        for (String lockKey : txn.getHeldLocks()) {
            ownerByKey.remove(lockKey);
            locks.get(lockKey).writeLock().unlock();
        }
        txn.getHeldLocks().clear();
    }
}
```

Every `put`/`delete` inside a transaction (§32) acquires an exclusive lock on that specific row *before* reading or writing it, and holds it until commit or rollback — this is what prevents two concurrent transactions from interleaving writes to the same row into a corrupted result, i.e. it's the direct mechanism behind Isolation (§30) at the Read Committed level.

---

# 36. Two-Phase Locking (2PL)

**Two-Phase Locking** is the rule that makes row-locking actually serializable-safe rather than just "avoids simultaneous writes": a transaction is split into a **growing phase** (locks are only ever acquired, never released) and a **shrinking phase** (locks are only ever released, never re-acquired) — and crucially, all locks are released together, at commit or rollback (exactly what `releaseAll` in §32/§35 does), never one at a time mid-transaction.

```text
Growing phase:    acquire(A) -> acquire(B) -> acquire(C)   <- transaction's actual work happens interleaved here
Shrinking phase:  release(A), release(B), release(C)        <- all at once, at commit/rollback
```

Releasing a lock early (say, right after the last read of row `A`, instead of waiting for commit) seems like a throughput win, but it reopens the door to another transaction observing a state that could still be undone by this transaction's eventual rollback — violating isolation. Holding every lock until the transaction's own end is what makes 2PL's guarantee hold.

---

# 37. Deadlock Detection (Wait-For Graph)

Two-phase locking (§36) introduces a new failure mode: transaction A holds row `"1"` and waits for row `"2"`; transaction B holds row `"2"` and waits for row `"1"` — neither can ever proceed. A **wait-for graph** detects this by modeling "transaction X is waiting for a lock held by transaction Y" as a directed edge `X -> Y`; a **cycle** in that graph is a deadlock.

```java
// txn/DeadlockDetector.java
public class DeadlockDetector {
    private final Map<Long, Long> waitsFor = new ConcurrentHashMap<>(); // waiting txnId -> holding txnId

    public void recordWait(long waitingTxnId, long holdingTxnId) {
        waitsFor.put(waitingTxnId, holdingTxnId);
    }
    public void clearWait(long txnId) { waitsFor.remove(txnId); }

    /** Walks the wait-for chain from a transaction back to itself — a cycle means deadlock. */
    public boolean hasCycleStartingFrom(long txnId) {
        Set<Long> visited = new HashSet<>();
        Long current = txnId;
        while (current != null) {
            if (!visited.add(current)) return current.equals(txnId); // revisited a node -> cycle; check it includes us
            current = waitsFor.get(current);
        }
        return false;
    }
}
```

When `acquireExclusive` (§35) is about to block, it first records a wait-for edge and checks for a cycle; if one exists, TinyDB picks a victim (conventionally the youngest transaction, to bound wasted work) and forces it to roll back (§33) instead of letting both transactions wait forever — the same strategy real databases like Postgres and SQL Server use, just with far more sophisticated victim-selection heuristics.

---

# 38. Phase 21 — MVCC: Snapshot Isolation Without Blocking Readers

Locking (§35) has a real cost: readers block writers and vice versa, even when a reader would have been perfectly happy seeing a slightly older, still-consistent value. **Multi-Version Concurrency Control** avoids this by never overwriting a row in place — instead, every write creates a new *version* tagged with the transaction/commit id that created it, and a reader is given a **snapshot**: "the newest version of each row that was committed before my transaction began."

```java
// txn/MvccStore.java
public class MvccStore {
    public record VersionedRow(Row row, long createdByTxnId, long committedAtSeq, boolean deleted) { }

    private final Map<String, NavigableMap<Long, VersionedRow>> versionsByKey = new ConcurrentHashMap<>();
    private final AtomicLong commitSequence = new AtomicLong(0);

    public void write(String table, String key, Row row, long txnId) {
        // Tentatively tagged with the transaction id; made visible only once committed (assignCommitSequence below)
        versionsByKey.computeIfAbsent(table + "/" + key, k -> new ConcurrentSkipListMap<>())
            .put(Long.MAX_VALUE - txnId, new VersionedRow(row, txnId, -1, false)); // pending, not yet visible
    }

    /** Called at commit — this is the moment a version becomes visible to snapshots started AFTER it. */
    public void assignCommitSequence(long txnId, String table, String key) {
        long seq = commitSequence.incrementAndGet();
        // re-key the pending version's entry with the real commit sequence — omitted plumbing for brevity
    }

    /** A snapshot read never blocks on a lock — it just picks the newest version committed before mySnapshotSeq. */
    public Row readAsOf(String table, String key, long mySnapshotSeq) {
        NavigableMap<Long, VersionedRow> versions = versionsByKey.get(table + "/" + key);
        if (versions == null) return null;
        for (VersionedRow version : versions.values()) {
            if (version.committedAtSeq() != -1 && version.committedAtSeq() <= mySnapshotSeq) {
                return version.deleted() ? null : version.row();
            }
        }
        return null;
    }
}
```

Old versions are never needed once no active transaction's snapshot could still reference them — a background **garbage collector** (conceptually the same idea as §14's compaction, applied to row versions instead of segment files) reclaims them once the oldest active transaction's snapshot sequence has moved past their commit sequence.

---

# 39. Comparing Locking vs MVCC

| | Row-level locking + 2PL (§35–§37) | MVCC (§38) |
|---|---|---|
| Readers vs writers | Block each other | Never block each other — a reader sees an old-but-consistent snapshot instead of waiting |
| Writers vs writers | Block each other on the same row (needed either way) | Still need to detect write-write conflicts (two txns both writing the same key based on the same snapshot) |
| Memory/storage cost | None beyond the lock table | Must retain old row versions until no snapshot needs them |
| Deadlocks | Possible (§37) | Possible only among writers, not between readers and writers |
| Real-world examples | Traditional 2PL databases, most default MySQL/InnoDB configurations for writers | Postgres (MVCC is its default model), Oracle, MySQL/InnoDB's read view mechanism |

TinyDB deliberately implements **both** so the tradeoff is something you can point to in running code rather than only describe: use `LockManager` for workloads that need strict, blocking correctness with simple reasoning; use `MvccStore` for read-heavy workloads where blocking readers behind writers would be an unacceptable throughput cost.

---

# 40. Phase 22 — Putting It Together: A Transactional put/get/commit/rollback

```java
Transaction txn = transactionManager.begin();                 // §32 -> WAL: TXN_BEGIN
try {
    transactionManager.put(txn, "accounts", "alice", debited); // §32 -> lock acquired (§35), undo image saved (§33)
    transactionManager.put(txn, "accounts", "bob", credited);  // same again for the second row
    transactionManager.commit(txn);                            // §32 -> WAL: PUTs + TXN_COMMIT, THEN applied + fsynced
} catch (Exception e) {
    transactionManager.rollback(txn);                          // §33 -> undo log replay, locks released, no data changed
    throw e;
}
```

If the process crashes at any point before `TXN_COMMIT` is durable, recovery (§28) reconstructs a database where **neither** of Alice's or Bob's balances changed — atomicity held even though nobody was around to call `rollback()`. If it crashes right after, both changes are present — durability held. Every request that reached `commit()` successfully saw a consistent, isolated view the whole way through, thanks to §35/§38. This one code block is the entire ACID story, executed.

---

# 41. Why Replication

A single-node database has a single point of failure: if that node's disk dies, every committed transaction is gone regardless of how correctly §9–§40 were implemented — durability on one machine only protects against *process* crashes, not *hardware* loss. **Replication** copies committed data to one or more other nodes so the database survives losing an entire machine, and can additionally serve reads from more than one place.

---

# 42. Replication Topologies — Leader-Follower vs Multi-Leader vs Leaderless

| Topology | Writes accepted by | Strength | Weakness |
|---|---|---|---|
| Leader-follower (single-leader) | Only the leader | Simple to reason about — one order of writes, no write conflicts | The leader is a bottleneck and a single point of write availability |
| Multi-leader | Any of several leader nodes, often in different regions | Writes succeed even if one region is unreachable | Concurrent writes to the same key on different leaders can conflict, requiring conflict resolution |
| Leaderless (quorum-based) | Any replica, coordinated via read/write quorums | No single node is special; naturally tolerant of individual node failures | Requires careful quorum math (`W + R > N`) to guarantee consistency; conflict resolution still needed |

TinyDB implements **leader-follower** replication (§43–§46) — it is by far the most common starting point (Postgres streaming replication, MySQL binlog replication, and the "primary" side of most managed databases all work this way), and it reuses a component TinyDB already has: the WAL is, by construction, an ordered list of every change the leader has ever made — exactly what a follower needs to reconstruct the same state.

---

# 43. Phase 23 — Leader-Follower Replication by Shipping the WAL

**The core idea:** a follower doesn't need to be told "row X changed to Y" through some separate replication-specific mechanism — it just needs a copy of the leader's WAL records, applied in the same order, using the exact same `recover()`-style replay logic already built in §12.

```java
// replication/ReplicationLeader.java
public class ReplicationLeader {
    private final List<FollowerConnection> followers = new CopyOnWriteArrayList<>();

    /** Called by WriteAheadLog.append() (§11) right after a record is fsynced on the leader. */
    public void onRecordAppended(LogRecord record, long offset) {
        for (FollowerConnection follower : followers) {
            follower.sendAsync(record, offset); // fire-and-forget, or awaited for synchronous replication — see below
        }
    }
}

// replication/ReplicationFollower.java
public class ReplicationFollower {
    private final StorageEngine storage;
    private long lastAppliedOffset = -1;

    public void onRecordReceived(LogRecord record, long offset) throws IOException {
        if (offset != lastAppliedOffset + 1) {
            throw new IllegalStateException("Gap in replication stream — follower must resync from a snapshot, §44");
        }
        localWal.append(record);        // the follower keeps its OWN durable WAL too — it can itself crash and recover
        storage.applyRecord(record);    // then apply it, exactly like local crash recovery (§12) would
        lastAppliedOffset = offset;
    }
}
```

**Synchronous vs asynchronous replication** is a direct durability/latency tradeoff: if the leader waits for at least one follower to acknowledge the record before telling the *client* the commit succeeded, a leader crash immediately after can never lose that transaction (it's on the follower too) — at the cost of every commit now waiting on a network round trip. Asynchronous replication acknowledges the client as soon as the local WAL `fsync` completes (§9, §31), which is faster but means a leader crash *can* lose the most recent few transactions that hadn't yet reached a follower.

---

# 44. Phase 24 — Follower Bootstrap From a Snapshot

A brand-new follower (or one that's been offline so long its gap can't be filled by replaying missing WAL segments) can't start by asking for "every WAL record ever written" — that's exactly the unbounded-replay problem §13 already solved for local recovery, and the same fix applies:

```java
public class FollowerBootstrap {
    public void bootstrap(ReplicationLeader leader, StorageEngine localStorage) throws IOException {
        SnapshotTransfer snapshot = leader.requestLatestSnapshot(); // §13's snapshot file, sent over the wire
        localStorage.loadSnapshot(snapshot.data());
        long walOffsetAtSnapshot = snapshot.walOffset();

        // Now stream every WAL record the leader has written SINCE that snapshot's offset
        try (WalRecordStream stream = leader.streamWalFrom(walOffsetAtSnapshot)) {
            LogRecord record;
            while ((record = stream.next()) != null) {
                localStorage.applyRecord(record);
            }
        }
        // From here on, the follower switches to the live onRecordReceived(...) path from §43
    }
}
```

This is precisely the same "snapshot + replay the remaining log" pattern from §13, just with the snapshot and log transferred over a network connection instead of read from local disk — replication and crash recovery are, structurally, the same problem solved twice.

---

# 45. Replication Lag and Read-Your-Writes Consistency

With asynchronous replication (§43), a follower is always at least slightly behind the leader — **replication lag**. This becomes user-visible as a real correctness surprise: a client writes a row via the leader, then immediately reads it back via a follower (e.g. through a naive "route reads to followers for load-balancing" policy) and doesn't see its own write, because it hasn't replicated yet.

The standard mitigations, none of which are free:

- **Read-your-writes consistency:** route a client's reads to the leader (or to a follower known to have caught up past the offset of that client's own last write) for some window after it writes.
- **Monotonic reads:** always route a given client's reads to the *same* follower, so it never sees time appear to "go backward" by reading a more-caught-up follower and then a less-caught-up one.
- **Bounded staleness:** expose the follower's current lag (leader's latest offset minus the follower's `lastAppliedOffset` from §43) and let callers decide how much staleness they can tolerate.

---

# 46. Phase 25 — Failover and Leader Election (Simplified)

A full consensus protocol (Raft, Paxos) is out of scope for this guide, but the shape of the problem is worth building in simplified form, because "what happens when the leader dies" is a near-guaranteed follow-up question once replication is on the table:

```java
public class SimpleFailoverCoordinator {
    private final List<ReplicaHandle> replicas;
    private final Duration heartbeatTimeout = Duration.ofSeconds(5);

    public void monitorLeader(ReplicaHandle leader) {
        while (true) {
            if (!leader.respondedToHeartbeatWithin(heartbeatTimeout)) {
                promoteNewLeader();
                return;
            }
            sleep(Duration.ofSeconds(1));
        }
    }

    private void promoteNewLeader() {
        // Pick the follower with the HIGHEST lastAppliedOffset (§43) — it has the least data to reconcile,
        // minimizing (but, without a consensus protocol, not fully eliminating) the chance of losing
        // an acknowledged-but-not-yet-replicated write from the old leader.
        ReplicaHandle mostCaughtUp = replicas.stream()
            .max(Comparator.comparingLong(ReplicaHandle::getLastAppliedOffset))
            .orElseThrow();
        mostCaughtUp.promoteToLeader();
        replicas.stream().filter(r -> r != mostCaughtUp).forEach(r -> r.followNewLeader(mostCaughtUp));
    }
}
```

**Why this is a simplification, honestly labeled:** without a quorum-based consensus protocol, there's a real risk of a **split-brain** scenario — a network partition can make the old leader look "dead" to the monitor while it's still alive and accepting writes from clients on its side of the partition, leading to two nodes both believing they're the leader. Production systems solve this with Raft/Paxos-style majority voting specifically to make split-brain provably impossible; naming that gap explicitly is itself a legitimate and expected part of a strong interview answer.

---

# 47. Why TTL Matters

Many workloads write data that is only meaningful for a bounded time — session tokens, rate-limit counters, cache entries, idempotency keys. Without a **time-to-live**, every such row lives forever unless something explicitly deletes it, silently growing storage and — worse — letting stale data (an expired session, a used-up idempotency key) be read back as if it were still valid. DynamoDB's TTL feature and Redis's `EXPIRE` solve exactly this; TinyDB implements the same idea.

---

# 48. Phase 26 — Storing Expiry Metadata

An expiry time is just one more piece of metadata alongside a row — stored in the same WAL record (so it survives crash recovery and replication for free) and indexed separately so expired rows can be found efficiently without scanning every row in the database:

```java
// ttl/ExpiryIndex.java
public class ExpiryIndex {
    // Sorted by expiry time, so "give me everything expired before now" is a cheap prefix scan (headMap)
    private final NavigableMap<Long, Set<String>> keysByExpiryTime = new ConcurrentSkipListMap<>();

    public void trackExpiry(String table, String key, long expiresAtEpochMillis) {
        keysByExpiryTime.computeIfAbsent(expiresAtEpochMillis, t -> ConcurrentHashMap.newKeySet())
            .add(table + "/" + key);
    }

    public List<String> expiredAsOf(long nowEpochMillis) {
        List<String> expired = new ArrayList<>();
        keysByExpiryTime.headMap(nowEpochMillis, true).values().forEach(expired::addAll);
        return expired;
    }

    public void untrack(String table, String key, long previousExpiryTime) {
        var keys = keysByExpiryTime.get(previousExpiryTime);
        if (keys != null) keys.remove(table + "/" + key);
    }
}
```

`Command.Put` (§21) grows an optional `ttlSeconds` field, and `LogRecord` (§10) carries the resulting absolute `expiresAt` timestamp — recorded once, in absolute terms, rather than as a relative "seconds from now," specifically so that **replaying it later** (during crash recovery, §12, or on a replication follower, §43) expires the row at the *originally intended* wall-clock time, not "N seconds after this replay happens to run."

---

# 49. Phase 27 — Lazy Expiration on Read

The simplest correctness guarantee: never return an expired row, checked at the moment of read, regardless of whether anything has proactively cleaned it up yet:

```java
public Row get(String table, String key) {
    Row row = storage.getRaw(table, key);
    if (row == null) return null;
    Long expiresAt = row.getExpiryTimestamp();
    if (expiresAt != null && expiresAt <= System.currentTimeMillis()) {
        return null; // logically expired — even if the physical row hasn't been reaped from disk yet (§50)
    }
    return row;
}
```

This alone is enough to guarantee **correctness** (no caller ever observes an expired value) even with zero background work — but without §50, expired rows still physically occupy space and still slow down `SCAN` (§21), which must apply this same check to every row it walks past.

---

# 50. Phase 28 — Active Expiration (Background Reaper)

A background thread periodically finds and physically removes expired rows, turning "logically gone" into "actually reclaimed," using exactly the index built in §48:

```java
// ttl/ExpiryReaper.java
public class ExpiryReaper {
    private final ExpiryIndex expiryIndex;
    private final StorageEngine storage;
    private final ChangeEventPublisher changeEvents; // §54 — an expiry IS a change worth publishing

    public void runPeriodically(Duration interval) {
        Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(this::reapOnce,
            interval.toMillis(), interval.toMillis(), TimeUnit.MILLISECONDS);
    }

    private void reapOnce() {
        long now = System.currentTimeMillis();
        for (String compositeKey : expiryIndex.expiredAsOf(now)) {
            String[] parts = compositeKey.split("/", 2);
            try {
                Row expiredRow = storage.get(parts[0], parts[1]);
                storage.delete(parts[0], parts[1]); // goes through the normal WAL-logged delete path (§9)
                changeEvents.publishRemove(parts[0], parts[1], expiredRow, ChangeEvent.Cause.TTL_EXPIRY);
            } catch (IOException e) {
                System.err.println("Failed to reap expired key " + compositeKey + ": " + e.getMessage());
            }
        }
    }
}
```

Running this on a fixed interval, rather than exactly at each key's expiry instant, is a deliberate, DynamoDB-like tradeoff: DynamoDB's own documentation is explicit that TTL deletion "usually" happens within 48 hours, precisely because background, batched reaping is far cheaper at scale than a per-key timer — §49's lazy check is what makes that delay invisible to correctness.

---

# 51. TTL Interactions With Replication and the WAL

Two subtleties worth calling out explicitly, since they're exactly the kind of edge case an interviewer probes for:

- **The reaper's delete must itself be replicated.** Because `storage.delete(...)` in §50 goes through the normal write path, it produces a normal `DELETE` `LogRecord` that §43's `ReplicationLeader.onRecordAppended` ships to followers like any other write — a follower must never independently decide to expire a row on its own clock, or a leader and follower could disagree about which rows exist. The expiry decision is made **once**, on the leader, and replicated as an ordinary delete.
- **Absolute, not relative, timestamps in the WAL** (§48) matter doubly here: a follower replaying WAL records during bootstrap (§44) must compute "is this row expired" using the row's stored absolute `expiresAt`, not by re-deriving a TTL countdown from whenever the follower happens to process the record — otherwise a slow follower could expire rows "early" relative to what the leader promised.

---

# 52. What Change Data Capture Is (and How DynamoDB Streams Works)

**Change Data Capture (CDC)** is the general pattern of exposing every write a database makes as a durable, ordered, subscribable event feed — so other services can react to changes without polling the database or coupling to its internal schema. DynamoDB Streams is a well-known, purpose-built implementation of this pattern: every `INSERT`/`MODIFY`/`REMOVE` on a table is captured as an event with (optionally) the before and/or after image of the item, made available through a **shard iterator** API that lets consumers resume exactly where they left off, even across restarts.

The mechanically important insight — and the reason this section comes *after*, not before, the WAL sections — is that **a WAL already is a change stream**; TinyDB's implementation is "expose the WAL's records to external subscribers, in a consumer-friendly shape," not a separate subsystem built from scratch. This is also literally how Debezium (the most widely used open-source CDC tool) works against Postgres/MySQL: it reads their write-ahead/binary logs, not their tables.

---

# 53. Phase 29 — A Change Event Model

```java
// stream/ChangeEvent.java
public class ChangeEvent {
    public enum Type { INSERT, MODIFY, REMOVE }
    public enum Cause { USER_WRITE, TTL_EXPIRY, ROLLBACK_UNDONE }

    private final long sequenceNumber;   // strictly increasing — this IS the resumable position (§55)
    private final Type type;
    private final Cause cause;
    private final String table;
    private final String key;
    private final Row beforeImage;       // null for INSERT
    private final Row afterImage;        // null for REMOVE
    private final long timestamp;

    // constructor / getters omitted for brevity
}
```

Modeling `beforeImage`/`afterImage` explicitly — rather than just "here's the new value" — is what lets a consumer compute a diff, detect exactly which fields changed, or implement its own undo logic downstream, mirroring DynamoDB Streams' `NEW_AND_OLD_IMAGES` view type.

---

# 54. Phase 30 — Publishing Change Events From the Write Path

The publish point is exactly where a write becomes durable and visible — the same moment replication (§43) hooks in, and for the same reason: **only committed changes should ever be published**, otherwise a subscriber could react to a write that a subsequent rollback (§33) erases.

```java
// stream/ChangeEventPublisher.java
public class ChangeEventPublisher {
    private final AtomicLong sequenceGenerator = new AtomicLong(0);
    private final PersistentChangeLog changeLog; // §57 — durable, so subscribers can resume after a restart
    private final List<StreamConsumer> inProcessSubscribers = new CopyOnWriteArrayList<>();

    public void publish(ChangeEvent.Type type, ChangeEvent.Cause cause, String table, String key, Row before, Row after) {
        ChangeEvent event = new ChangeEvent(sequenceGenerator.incrementAndGet(), type, cause, table, key, before, after, System.currentTimeMillis());
        changeLog.append(event);                                    // durable first (§57) — same principle as the WAL
        inProcessSubscribers.forEach(subscriber -> subscriber.onEvent(event));
    }
}
```

```java
// Hooked into TransactionManager.commit() (§32) — AFTER the WAL's TXN_COMMIT is durable, never before
public void commit(Transaction txn) throws IOException {
    for (LogRecord record : txn.getPendingWrites()) wal.append(record);
    wal.append(new LogRecord(LogRecord.Type.TXN_COMMIT, txn.getId(), null, null, null));
    for (LogRecord record : txn.getPendingWrites()) {
        Row before = storage.get(record.getTable(), record.getKey());
        storage.applyInMemory(record);
        Row after = storage.get(record.getTable(), record.getKey());
        changeEventPublisher.publish(before == null ? ChangeEvent.Type.INSERT : ChangeEvent.Type.MODIFY,
            ChangeEvent.Cause.USER_WRITE, record.getTable(), record.getKey(), before, after);
    }
    lockManager.releaseAll(txn);
}
```

---

# 55. Phase 31 — A Stream Consumer API (Shard Iterator Style)

Mirroring DynamoDB Streams' actual API shape: a consumer doesn't get pushed events directly forever — it gets an **iterator** representing a position in the stream, polls for a batch, processes it, and explicitly advances:

```java
// stream/ShardIterator.java
public class ShardIterator {
    private final PersistentChangeLog changeLog;
    private long position; // the next sequence number to read — this IS the resumable checkpoint

    public static ShardIterator fromSequenceNumber(PersistentChangeLog changeLog, long afterSequenceNumber) {
        ShardIterator it = new ShardIterator(changeLog);
        it.position = afterSequenceNumber + 1;
        return it;
    }
    public static ShardIterator trimHorizon(PersistentChangeLog changeLog) {
        return fromSequenceNumber(changeLog, 0); // start from the very beginning of retained history
    }
    public static ShardIterator latest(PersistentChangeLog changeLog) {
        return fromSequenceNumber(changeLog, changeLog.currentMaxSequence());
    }

    public List<ChangeEvent> getRecords(int maxCount) {
        List<ChangeEvent> batch = changeLog.readFrom(position, maxCount);
        if (!batch.isEmpty()) position = batch.get(batch.size() - 1).getSequenceNumber() + 1;
        return batch;
    }
}
```

```java
// A consumer's poll loop — the same shape as a Kinesis/DynamoDB Streams client, or a Kafka consumer
ShardIterator iterator = ShardIterator.fromSequenceNumber(changeLog, lastCheckpointedSequence);
while (running) {
    List<ChangeEvent> batch = iterator.getRecords(100);
    for (ChangeEvent event : batch) {
        process(event);
        checkpoint(event.getSequenceNumber()); // persist progress SOMEWHERE the consumer owns — §56
    }
}
```

---

# 56. Ordering, At-Least-Once, and Exactly-Once Semantics

- **Ordering:** because `sequenceNumber` is assigned by a single `AtomicLong` at publish time (§54) and every consumer reads strictly by increasing sequence number (§55), all consumers see the same total order of events for a given TinyDB instance — a single-partition, single-writer log has no ordering ambiguity to resolve.
- **At-least-once is the default and the safe choice:** if a consumer crashes after calling `process(event)` but before `checkpoint(...)` persists, it will see that event again after restarting from its last saved checkpoint — meaning **consumers must make `process(event)` idempotent** (e.g. keyed by `sequenceNumber`, so reprocessing the same event twice is harmless) rather than TinyDB trying to guarantee delivery exactly once, which would require a distributed transaction spanning TinyDB and every consumer's own storage.
- **Exactly-once "effectively"** is achievable only by combining at-least-once delivery with an idempotent consumer — precisely the pattern DynamoDB Streams, Kinesis, and Kafka all document as the correct way to build a reliable consumer, rather than promising exactly-once delivery themselves.

---

# 57. Phase 32 — Persisting the Change Log for Replay After Restart

```java
// stream/PersistentChangeLog.java — a dedicated append-only file, structurally identical to the WAL (§9-§11)
public class PersistentChangeLog {
    private final FileChannel channel;

    public void append(ChangeEvent event) {
        byte[] bytes = serialize(event); // length-prefixed + checksummed, same format discipline as §10
        synchronized (this) {
            try {
                channel.write(ByteBuffer.wrap(bytes));
                channel.force(true); // a consumer must never be told about an event that could still be lost
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        }
    }

    public List<ChangeEvent> readFrom(long sequenceNumber, int maxCount) {
        // Scans forward from a sequence-number index (built the same way HashIndex is, §16) until maxCount is reached
        // or the end of the log — omitted here; structurally identical to WalReader from §12.
        return List.of();
    }

    public long currentMaxSequence() { /* tracked incrementally as events are appended */ return 0; }
}
```

Retention (how far back the change log keeps events before old segments are compacted away, §14-style) is a deliberate, configurable tradeoff: DynamoDB Streams retains 24 hours by default — long enough for a consumer to recover from a reasonable outage, short enough to bound storage. A consumer that falls behind retention entirely (its checkpoint points at a sequence number that's already been compacted away) has no choice but to fall back to a **full table scan** to rebuild its own state — the same "gap too large, must resync from a snapshot" situation replication followers hit in §44.

---

# 58. Phase 33 — Wiring a Stream Listener Into MiniSpring

```java
@Component
public class UserChangeAuditListener implements StreamConsumer {
    private final MiniConnectionPool auditDbPool; // could even be a second TinyDB instance, or any JDBC target

    @Autowired
    public UserChangeAuditListener(MiniConnectionPool auditDbPool) { this.auditDbPool = auditDbPool; }

    @PostConstruct // MiniSpring companion guide §34
    public void startConsuming(ChangeEventPublisher publisher) {
        Thread consumerThread = new Thread(() -> {
            ShardIterator iterator = ShardIterator.trimHorizon(publisher.getChangeLog());
            while (true) {
                for (ChangeEvent event : iterator.getRecords(100)) {
                    if ("users".equals(event.getTable())) onEvent(event); // idempotent — §56
                }
            }
        }, "user-change-audit-consumer");
        consumerThread.setDaemon(true);
        consumerThread.start();
    }

    @Override
    public void onEvent(ChangeEvent event) {
        // e.g. write an audit row keyed by event.getSequenceNumber() — safe to process the same event twice (§56)
    }
}
```

This is the same architectural shape as a `@KafkaListener` in real Spring Kafka, or a DynamoDB Streams Lambda trigger — a `@Component` bean that MiniSpring's `ApplicationContext` (companion guide §24) constructs and wires with `@Autowired` exactly like any other bean, which happens to spend its life consuming a stream instead of responding to HTTP requests.

---

# 59. Data Integrity: Checksums and Corruption Detection

Every append-only log in this guide (the WAL, §10; the change log, §57) shares one more responsibility beyond "record what happened": detecting when its own bytes have been corrupted, whether by a torn write during a crash or by bit rot on disk.

```java
private LogRecord readNextSafely(DataInputStream in) throws IOException {
    int length;
    try { length = in.readInt(); } catch (EOFException e) { return null; } // clean end of file — normal
    int storedChecksum = in.readInt();
    byte[] body = in.readNBytes(length);
    if (body.length < length) {
        // Torn write: the crash happened mid-record. This is EXPECTED after an unclean shutdown —
        // recovery must stop here, not throw, and treat everything from this point as if it were never written.
        return null;
    }
    int actualChecksum = crc32(body);
    if (actualChecksum != storedChecksum) {
        throw new CorruptLogException("Checksum mismatch at record — possible disk corruption, not a torn write");
    }
    return LogRecord.deserialize(body);
}
```

The distinction in that comment matters: a **truncated** record (fewer bytes than the length prefix promised) is the *expected* signature of a crash mid-append, and recovery treats it as "stop here, this record and anything after it was never durably committed." A **checksum mismatch on a full-length record** is a different, more serious signal — the bytes are all present but wrong, which points at disk-level corruption rather than an interrupted write, and is treated as a hard failure rather than a normal end-of-log condition.

---

# 60. Full End-to-End Example: Transaction + Replication + TTL + Stream in One Request

```java
@Service
public class SessionService {
    private final MiniConnectionPool pool; // TinyDB, via JDBC (§25-§26)

    @Autowired
    public SessionService(MiniConnectionPool pool) { this.pool = pool; }

    public void createSession(String sessionId, String userId) throws SQLException, InterruptedException {
        try (Connection connection = pool.borrow()) {
            connection.setAutoCommit(false);                                  // TXN_BEGIN (§32)
            try (PreparedStatement ps = connection.prepareStatement(
                    "PUT sessions " + sessionId + " TTL 1800")) {              // 30-minute TTL (§48)
                ps.setObject(1, Map.of("userId", userId, "createdAt", Instant.now()));
                ps.executeUpdate();
                connection.commit();                                          // WAL fsync (§9, §31) -> durable
            } catch (SQLException e) {
                connection.rollback();                                        // undo log replay (§33) on failure
                throw e;
            }
        }
    }
}
```

Tracing everything that happens on a successful call: the write is buffered under a transaction and locked (§32, §35); on `commit()`, it's appended to the WAL and fsynced (§9, §28); the leader ships that WAL record to every follower (§43); the row is indexed with its absolute expiry time (§48); a `ChangeEvent` is published and durably logged (§54, §57), which `UserChangeAuditListener` (§58) picks up on its own consumer thread; and thirty minutes later, `ExpiryReaper` (§50) deletes the row through the same WAL-logged, replicated, stream-published path as any other write. Every guarantee this guide built is exercised by one ordinary method call.

---

# 61. Final Architecture

```text
                        Client / MiniSpring @Repository, @Component
                                          |
                            TinyDbDriver (java.sql.Driver, §25)
                                          |
                              TinyDbConnection / MiniConnectionPool
                                          |
                                    Wire Protocol (§22)
                                          |
                                          v
                              +----------------------+
                              |     TinyDbServer       |
                              |  accept loop + pool    |
                              +-----------+------------+
                                          |
                                          v
                              +----------------------+
                              |     QueryEngine        |
                              +-----------+------------+
                                          |
              +---------------------------+---------------------------+
              |                           |                           |
              v                           v                           v
    +-------------------+     +-------------------+       +----------------------+
    | TransactionManager |     |   ExpiryReaper     |       | ChangeEventPublisher |
    | locks (2PL, §35-37)|     |   (§50, TTL)        |       | (§54, streams)       |
    | or MVCC (§38)       |     +-------------------+       +----------+-----------+
    | undo log (§33)      |                                            |
    +----------+----------+                                 PersistentChangeLog (§57)
               |                                                       |
               v                                                       v
    +----------------------+                                StreamConsumer(s) (§58)
    | Write-Ahead Log (§9)  |
    +----------+-----------+
               |  ships records
               v
    +----------------------+
    | ReplicationLeader (§43)|----> ReplicationFollower(s) (§43-44) ----> their own StorageEngine
    +----------+-----------+
               v
    +----------------------+
    |   StorageEngine       |
    |  HashIndex / BTree     |
    |  (§16-17) + Segments   |
    |  (§14) + Snapshots (§13)|
    +----------------------+
```

---

# 62. Optimization — Group Commit and Batched fsync

Already introduced in §31 as the mechanism, worth restating as the **first** optimization to reach for: if a benchmark (§69) shows write throughput capped well below disk bandwidth with high `fsync`-wait time in a profiler, batching concurrent commits into a single `fsync` call is almost always the highest-leverage fix — it directly attacks the single most expensive operation on the write path without weakening any durability guarantee (every batched write is still fsynced before being acknowledged).

---

# 63. Optimization — Page Cache, mmap, and Zero-Copy Reads

Reads from already-written segment files (§14) can bypass an extra JVM-heap copy the same way the MiniTomcat guide's `FileChannel.transferTo` (that guide's §60) avoids one for outgoing file serving: memory-mapping a read-only, sealed segment file with `FileChannel.map(MapMode.READ_ONLY, ...)` lets the OS's page cache serve repeated reads of hot data directly, without TinyDB re-reading bytes through `read()` calls it has already read once. Sealed (post-compaction) segments are ideal candidates specifically because they never change once written — there's no cache-invalidation problem to solve.

---

# 64. Optimization — Bloom Filters to Skip Unnecessary Disk Reads

Referenced in §19: an LSM-style engine checking multiple SSTable levels for a key it may not even have needs a cheap way to rule out "definitely not in this file" without an actual disk read. A **Bloom filter** — a compact, probabilistic bit array built when each SSTable is written — answers "might be present" or "definitely absent" in memory, in constant time:

```java
public class BloomFilter {
    private final BitSet bits;
    private final int size;
    private final int hashCount;

    public void add(String key) {
        for (int i = 0; i < hashCount; i++) bits.set(hash(key, i) % size);
    }
    public boolean mightContain(String key) {
        for (int i = 0; i < hashCount; i++) {
            if (!bits.get(hash(key, i) % size)) return false; // a single missing bit proves ABSENCE, for certain
        }
        return true; // all bits set — PROBABLY present; false positives are possible, false negatives are not
    }
}
```

Attaching one Bloom filter per SSTable means a `GET` for a key that exists in only one of several levels can skip the disk read for every other level entirely — the exact optimization LevelDB, RocksDB, and Cassandra all rely on to keep LSM-Tree read amplification (§18) manageable in practice.

---

# 65. Optimization — Compaction Scheduling

Running compaction (§14, §19) too aggressively steals disk I/O from live read/write traffic; running it too rarely lets superseded versions and tombstones accumulate, growing both disk usage and worst-case read latency (more segments/levels to check per `GET`). A workable default: trigger compaction when a segment/level's size passes a threshold *and* current write load is below a configured rate, and always cap compaction's own I/O throughput (a token-bucket rate limiter around its reads/writes) so it never starves the live request path — the same "background work must not win a resource fight against foreground latency" principle as the MiniTomcat guide's thread-pool sizing discipline.

---

# 66. Thread Safety Checklist

| Component | Shared state | Safety mechanism |
|---|---|---|
| `WriteAheadLog.append()` (§11) | The file's current write position | `synchronized (appendLock)` — WAL order must be strictly serialized, one writer at a time |
| `HashIndex` (§16) | The key→location map | `ConcurrentHashMap` — safe concurrent reads and writes |
| `LockManager` (§35) | Per-row locks and ownership | `ReentrantReadWriteLock` per key, `ConcurrentHashMap` for the lock table itself |
| `MvccStore` (§38) | Per-key version chains | `ConcurrentSkipListMap` per key — safe concurrent version inserts and snapshot reads without a global lock |
| `ExpiryIndex` (§48) | The time→keys map | `ConcurrentSkipListMap` — safe concurrent inserts alongside the reaper's periodic `headMap` scan |
| `ChangeEventPublisher` (§54) | The sequence counter, the subscriber list | `AtomicLong` for the counter, `CopyOnWriteArrayList` for subscribers (rarely mutated, frequently iterated) |
| `Transaction` state itself | Pending writes, held locks, undo log | Owned by exactly one connection/thread at a time (§23) — never shared across threads, so no locking is needed *within* it |

The last row is worth stating as a design principle, not just an implementation detail: a single `Transaction` object is intentionally never handed to more than one thread concurrently, which sidesteps an entire category of bugs rather than defending against it with locks.

---

# 67. Common Mistakes

- **Mistake 1 — Skipping `fsync` "for performance."** Silently turns every durability claim in §9/§28/§31 false; a crash can then lose acknowledged writes with no warning.
- **Mistake 2 — Applying a write to in-memory state before the WAL record is durable.** Reorders §9's rule and reopens the exact crash-loses-data window the WAL exists to close.
- **Mistake 3 — Forgetting the `TXN_COMMIT` gate on recovery replay (§28).** Without checking `committedTxnIds`, a crash mid-transaction resurrects a *partial* transaction on restart — a direct atomicity violation.
- **Mistake 4 — An unbounded WAL with no snapshotting (§13).** Works in every demo and makes crash recovery time grow forever in production.
- **Mistake 5 — Releasing locks before commit/rollback (§36).** Breaks two-phase locking's isolation guarantee even though "it still runs" in a quick manual test.
- **Mistake 6 — TTL expiry computed as a relative countdown instead of an absolute timestamp (§48, §51).** Produces different expiry behavior on a replication follower or during WAL replay than what the leader actually promised.
- **Mistake 7 — A non-idempotent stream consumer (§56).** At-least-once delivery is the honest default; a consumer that isn't safe to run twice on the same event will eventually double-process one.
- **Mistake 8 — Confusing "truncated record" with "corrupted record" during WAL replay (§59).** Throwing a hard error on every truncated tail turns an expected post-crash condition into a database that refuses to start after any unclean shutdown.
- **Mistake 9 — Promoting a replication follower without picking the most caught-up one (§46).** Silently loses more committed-but-unreplicated writes than necessary.
- **Mistake 10 — Compacting or expiring data a replication follower or CDC consumer might still need (§44, §57).** Retention windows must be sized against the *slowest* legitimate consumer/follower, not the average one.

---

# 68. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| WAL + recovery (§9-§12, §28) | A record survives a clean restart; a crash mid-append (simulated by cutting the file short) is detected as a torn write, not a corruption error | Write records, kill the process (or simulate by truncating the file), restart, assert exactly the committed state reappears |
| Transactions + rollback (§32-§34) | Rollback restores every touched row to its pre-transaction value; a crash before `TXN_COMMIT` loses the whole transaction | Begin a multi-row transaction, roll it back, assert original values; separately, crash before commit and assert recovery shows none of it |
| Locking + deadlock detection (§35-§37) | Two transactions taking locks in opposite order trigger detection and a forced rollback, not a permanent hang | Spawn two threads deliberately racing to deadlock; assert the test completes within a timeout, not that it hangs |
| MVCC (§38) | A reader's snapshot never observes a concurrent, uncommitted (or even a later-committed) write | Start a long-running read transaction, commit a conflicting write on another thread, assert the reader still sees its original snapshot |
| Replication (§43-§46) | A follower ends up byte-for-byte consistent with the leader after a burst of writes; bootstrap from a snapshot (§44) produces the same result as replaying every WAL record from the start | Run leader + follower in-process in a test, write a batch, assert `follower.get(...)` matches `leader.get(...)` for every key |
| TTL (§48-§51) | An expired row is invisible to `GET` immediately (lazy) and physically removed within one reaper interval (active) | Insert with a 1-second TTL, assert visible immediately, assert invisible after 1s even before the reaper runs, assert gone from storage after the reaper interval |
| Change stream (§53-§58) | Events are ordered, resumable after a simulated consumer restart, and only published for committed (never rolled-back) writes | Commit some writes, roll back others, assert the stream contains only the committed ones, in order |

The recovery, rollback, and replication tests are the ones that actually distinguish "a database" from "a fast in-memory map with a save button" — prioritize them over pure unit tests of individual data structures.

---

# 69. Benchmarking TinyDB

```bash
# A simple write-throughput benchmark: N clients, each doing M sequential PUTs
java -jar tinydb-bench.jar --mode write --clients 8 --ops-per-client 10000 --host localhost --port 9090
```

```bash
# Compare synchronous vs asynchronous replication's effect on commit latency (§43)
java -jar tinydb-bench.jar --mode write --replication sync   --clients 8 --ops-per-client 5000
java -jar tinydb-bench.jar --mode write --replication async  --clients 8 --ops-per-client 5000
```

What to look for, tying results back to specific sections:

| Observation | Likely explanation |
|---|---|
| Write throughput plateaus far below raw disk write bandwidth | `fsync` is the bottleneck — implement or tune group commit (§31, §62) |
| Latency rises sharply as concurrent transactions on overlapping keys increase | Lock contention (§35) — consider whether MVCC (§38) fits the workload better |
| Synchronous replication roughly doubles commit latency vs asynchronous | Expected — that round trip is the direct cost of the stronger durability guarantee (§43) |
| Read latency degrades over time under sustained writes with LSM-style storage | Compaction (§14, §19, §65) isn't keeping up with write volume — check its scheduling and I/O budget |
| Memory grows unbounded under MVCC with long-running read transactions | Old versions aren't being garbage-collected because a long-lived snapshot is pinning them (§38) — bound transaction lifetime or add monitoring for it |

---

# 70. Progressive Interview Question Set

**Level 1 — Durability fundamentals**
1. Why must a database append to a log before updating its main data structures, rather than the other way around?
2. What's the difference between a torn write and a corrupted record during WAL replay, and why does the recovery code need to treat them differently?

**Level 2 — Indexing**
3. Why can't a pure hash index answer a range query, and what does a B-Tree give up to be able to?
4. Walk through why an LSM-Tree can have higher read amplification than a B-Tree despite often having better write throughput.

**Level 3 — Transactions**
5. Trace exactly what's in the WAL, in order, for a two-row transaction that commits successfully — then for one that crashes right before its commit marker.
6. Why does rollback need its own log instead of just being "run the WAL records backward"?
7. Explain why releasing a lock before a transaction commits can violate isolation even if it looks safe in a quick test.

**Level 4 — Concurrency control**
8. Compare locking and MVCC for a workload that is 95% reads, 5% writes — which would you pick, and why?
9. Walk through how a wait-for graph detects a deadlock, and what the database should do once it finds one.

**Level 5 — Replication**
10. What's the actual latency/durability tradeoff between synchronous and asynchronous replication, in concrete terms?
11. Why is follower bootstrap "snapshot plus replay the remaining log" instead of just replaying the entire log from the beginning?
12. What specifically can go wrong with the simplified failover design in §46, and what would fix it?

**Level 6 — TTL and streams**
13. Why must TTL expiry be computed from an absolute timestamp rather than a relative countdown?
14. Why is "at-least-once plus an idempotent consumer" the realistic target for a change stream, rather than "exactly-once"?

**Final challenge:** A consumer of TinyDB's change stream (§55) needs to maintain a real-time materialized view of "total balance per user" from a stream of account-balance change events, must survive its own restarts without double-counting or missing an update, and must handle the underlying account rows occasionally being TTL-expired and later re-created. Design the consumer's checkpointing and idempotency strategy, and explain what part of TinyDB's guarantees it can lean on versus what it must handle itself.

---

# 71. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Real consensus-based replication (Raft) | Provably safe leader election, no split-brain window | Replaces §46's simplified heartbeat coordinator |
| Secondary indexes | Efficient lookups by a non-primary-key column | A second `HashIndex`/`BTreeIndex` per indexed column, updated in the same transaction as the primary write |
| A real query language (a SQL subset) | `WHERE`, `ORDER BY`, joins across tables — DML and DDL (`CREATE`/`DROP TABLE`) are now built (§73–§90); `ORDER BY`/joins/aggregates remain future work (see the row below) | A parser producing `Command`s (§21), sitting in front of the existing engine |
| Sharding / partitioning | Horizontal scale beyond one node's disk and CPU | Route `(table, key)` to a shard via consistent hashing; each shard is its own independent TinyDB with its own WAL/replication |
| Encryption at rest | Confidentiality for data on disk | Encrypt each segment file (§14) and the WAL (§9) with a per-database key, decrypt on read |
| Backup/restore tooling | Point-in-time recovery beyond crash recovery | Archive snapshots (§13) plus WAL segments to external storage on a schedule, exactly like Postgres's WAL archiving |
| Multi-region multi-leader replication | Lower write latency for geographically distributed clients | Requires conflict resolution (last-write-wins, CRDTs, or application-level merge) on top of §42's multi-leader topology |
| Richer isolation-level selection | Let callers choose Read Committed vs Snapshot Isolation per transaction | Expose it as a `Connection` property in the JDBC driver (§25), routed to `LockManager` or `MvccStore` accordingly |
| `ORDER BY`, aggregates (`COUNT`/`SUM`/`AVG`), `JOIN` | Real analytical queries, not just point lookups and filtered scans | Extends §75's translator with a sort/aggregate/join stage sitting between the storage scan and the result set (§76) |
| A cost-based query planner | Choosing a secondary index vs a full scan automatically, instead of always scanning (§79) | Sits in front of §75's translator once secondary indexes (row above) exist to choose between |
| CLI authentication (`\u user -p`) | Real credential checks instead of the open `-h`/`-P` connection §78 builds | A `USER`/`PASSWORD` handshake added to the wire protocol (§22) before the first command is accepted |

---

# 72. Why SQL and a Terminal Client Were Not Built Yet (Design Rationale)

Two things a real database is expected to have are conspicuously absent from everything built so far: a **query language** (`SELECT * FROM users WHERE id = '42'`) and a **terminal client** you can point at it the way `mysql -h host -P port` points at MySQL. Naming exactly why, before building them, is worth doing explicitly:

- **§21's `Command` and §22's wire protocol are a closed, hand-coded set of operations** — `GET`, `PUT`, `DELETE`, `SCAN`, `BEGIN`, `COMMIT`, `ROLLBACK` — dispatched by a single byte, not parsed from text. There is no grammar, no tokenizer, and no notion of a `WHERE` clause anywhere in §1–§71.
- **§25–§26's JDBC integration used SQL-*shaped* strings as a shortcut, not real SQL.** `connection.prepareStatement("GET users " + id)` sends a literal string the server never actually parses as a query — it's a placeholder that demonstrates "TinyDB speaks JDBC," not a working query language. That gap was real, and this section is where it gets closed.
- **The sequencing was deliberate, not accidental.** A query language is, mechanically, a thin *translation* layer that turns text into the exact same `Command` objects §21 already runs — it adds essentially no new correctness risk once the engine underneath (WAL, transactions, replication, TTL, streams) is trustworthy. Building it *before* that core would have meant polishing a front door on a building with no foundation. This is the same layering every real database uses: MySQL's SQL layer sits in front of pluggable storage engines (InnoDB, MyISAM) that know nothing about SQL syntax; SQLite's parser compiles to bytecode (VDBE) that runs against a B-Tree pager with its own, separate API.

§73–§79 build exactly this layer — a small SQL grammar, a parser, a translator into the existing `Command` API, a tabular wire format, and a REPL — without touching a single line of the storage engine, transaction manager, or replication code already built.

---

# 73. Phase 34 — A Minimal SQL Grammar

TinyDB's SQL subset is deliberately small — just enough to demonstrate the mechanism, not a general-purpose SQL implementation:

```text
SELECT ( * | column (',' column)* ) 'FROM' table [ 'WHERE' column '=' literal ]
INSERT 'INTO' table '(' column (',' column)* ')' 'VALUES' '(' literal (',' literal)* ')'
UPDATE table 'SET' column '=' literal [ 'WHERE' column '=' literal ]
DELETE 'FROM' table [ 'WHERE' column '=' literal ]
BEGIN | COMMIT | ROLLBACK

literal := '\'' ... '\''  |  digit+
```

No `JOIN`, no `ORDER BY`, no aggregate functions, and `WHERE` supports only a single `column = literal` equality — each of those is called out explicitly in §71's table as real future work, not something quietly skipped. What's here is enough to run `SELECT * FROM users WHERE id = '42'` end to end, which is the actual question this section exists to answer.

---

# 74. Phase 35 — Tokenizing and Parsing SQL

```java
// sql/SqlTokenizer.java
public class SqlTokenizer {
    public List<String> tokenize(String sql) {
        List<String> tokens = new ArrayList<>();
        StringBuilder current = new StringBuilder();
        boolean inQuotes = false;
        for (char c : sql.trim().toCharArray()) {
            if (c == '\'') {
                inQuotes = !inQuotes;
                current.append(c);
            } else if (!inQuotes && (Character.isWhitespace(c) || c == ',' || c == '(' || c == ')' || c == '=')) {
                if (!current.isEmpty()) { tokens.add(current.toString()); current.setLength(0); }
                if (!Character.isWhitespace(c)) tokens.add(String.valueOf(c));
            } else {
                current.append(c);
            }
        }
        if (!current.isEmpty()) tokens.add(current.toString());
        return tokens;
    }
}
```

```java
// sql/SqlStatement.java
public sealed interface SqlStatement {
    record Select(List<String> columns, String table, Predicate where) implements SqlStatement { }
    record Insert(String table, List<String> columns, List<String> values) implements SqlStatement { }
    record Update(String table, String setColumn, String setValue, Predicate where) implements SqlStatement { }
    record Delete(String table, Predicate where) implements SqlStatement { }
    record TxnBegin() implements SqlStatement { }
    record TxnCommit() implements SqlStatement { }
    record TxnRollback() implements SqlStatement { }

    record Predicate(String column, String value) { } // equality only, per §73's grammar
}

// sql/SqlParser.java
public class SqlParser {
    private final SqlTokenizer tokenizer = new SqlTokenizer();

    public SqlStatement parse(String sql) {
        List<String> tokens = tokenizer.tokenize(sql.trim().replaceAll(";\\s*$", ""));
        String keyword = tokens.get(0).toUpperCase();
        return switch (keyword) {
            case "SELECT" -> parseSelect(tokens);
            case "INSERT" -> parseInsert(tokens);
            case "UPDATE" -> parseUpdate(tokens);
            case "DELETE" -> parseDelete(tokens);
            case "BEGIN" -> new SqlStatement.TxnBegin();
            case "COMMIT" -> new SqlStatement.TxnCommit();
            case "ROLLBACK" -> new SqlStatement.TxnRollback();
            default -> throw new IllegalArgumentException("Unsupported SQL statement: " + sql);
        };
    }

    private SqlStatement.Select parseSelect(List<String> t) {
        int i = 1;
        List<String> columns = new ArrayList<>();
        while (!t.get(i).equalsIgnoreCase("FROM")) {
            if (!t.get(i).equals(",")) columns.add(t.get(i));
            i++;
        }
        String table = t.get(++i);
        SqlStatement.Predicate where = null;
        i++;
        if (i < t.size() && t.get(i).equalsIgnoreCase("WHERE")) {
            where = new SqlStatement.Predicate(t.get(i + 1), stripQuotes(t.get(i + 3))); // column '=' 'value'
        }
        return new SqlStatement.Select(columns, table, where);
    }
    // parseInsert/parseUpdate/parseDelete/stripQuotes omitted for brevity — same token-walking style
}
```

A hand-rolled recursive/positional parser like this is exactly proportionate to the grammar in §73 — a grammar this small doesn't earn a parser-generator (ANTLR, JavaCC); a real SQL engine supporting joins and subqueries would.

---

# 75. Phase 36 — Translating SQL Statements Into the Query Engine

This is the step that proves the design claim from §72: every `SqlStatement` becomes nothing more than one or more calls into the **exact same** `TransactionManager`/`StorageEngine` API §21–§40 already built — SQL never touches storage directly.

```java
// sql/SqlExecutor.java
public class SqlExecutor {
    private final TransactionManager transactionManager;
    private final StorageEngine storage;

    public TabularResult execute(SqlStatement statement, Transaction txn) throws IOException {
        return switch (statement) {
            case SqlStatement.Select s -> executeSelect(s);
            case SqlStatement.Insert s -> { doInsert(s, txn); yield TabularResult.affectedRows(1); }
            case SqlStatement.Update s -> { int n = doUpdate(s, txn); yield TabularResult.affectedRows(n); }
            case SqlStatement.Delete s -> { int n = doDelete(s, txn); yield TabularResult.affectedRows(n); }
            case SqlStatement.TxnBegin s -> TabularResult.ok(); // caller already called transactionManager.begin()
            case SqlStatement.TxnCommit s -> { transactionManager.commit(txn); yield TabularResult.ok(); }
            case SqlStatement.TxnRollback s -> { transactionManager.rollback(txn); yield TabularResult.ok(); }
        };
    }

    private TabularResult executeSelect(SqlStatement.Select select) throws IOException {
        List<Row> matches;
        if (select.where() != null && select.where().column().equals(primaryKeyOf(select.table()))) {
            // WHERE on the primary key -> a single indexed GET (§16/§17), not a scan
            Row row = storage.get(select.table(), select.where().value());
            matches = row == null ? List.of() : List.of(row);
        } else {
            // WHERE on any other column, or no WHERE at all -> a full table SCAN with a predicate applied in memory
            matches = storage.scan(select.table(), MIN_KEY, MAX_KEY).stream()
                .filter(row -> select.where() == null || matchesPredicate(row, select.where()))
                .toList();                                                          // see §79 for the cost of this path
        }
        List<String> columns = select.columns().equals(List.of("*")) ? allColumnsOf(select.table()) : select.columns();
        return TabularResult.of(columns, matches);
    }

    private boolean matchesPredicate(Row row, SqlStatement.Predicate where) {
        Object value = row.get(where.column());
        return value != null && value.toString().equals(where.value());
    }
    // doInsert/doUpdate/doDelete route to transactionManager.put/delete (§32) exactly like §26's JDBC path did
}
```

`SELECT * FROM users WHERE id = '42'` and `SELECT * FROM users WHERE name = 'Alice'` now both work — but they take **structurally different paths** through the engine, and that difference is the entire subject of §79.

---

# 76. Phase 37 — A Tabular Result Set Wire Format

§22's original wire protocol returned one opaque JSON-serialized value per response — fine for `GET`/`PUT`, not enough for a CLI that needs to know column names to draw a table:

```java
// sql/TabularResult.java
public class TabularResult {
    private final List<String> columnNames;
    private final List<List<String>> rows; // every value pre-stringified for display
    private final int affectedRows;        // for INSERT/UPDATE/DELETE, columnNames/rows are empty

    public static TabularResult of(List<String> columns, List<Row> matchedRows) {
        List<List<String>> rendered = matchedRows.stream()
            .map(row -> columns.stream().map(c -> String.valueOf(row.get(c))).toList())
            .toList();
        return new TabularResult(columns, rendered, -1);
    }
    public static TabularResult affectedRows(int n) { return new TabularResult(List.of(), List.of(), n); }
    public static TabularResult ok() { return affectedRows(0); }
    // getters omitted for brevity
}
```

```text
Wire encoding (extends §22's response format):
<status-byte> <column-count (1B)> [<column-name-length(2B)><column-name-bytes>]*
              <row-count (4B)> [<cell-count(1B)> [<cell-length(4B)><cell-bytes>]* ]*
              <affected-rows (4B, -1 if this was a SELECT)>
```

This is structurally the same idea as a real MySQL/Postgres wire protocol response — a column definition block followed by a stream of row data — just shrunk to what a terminal client (§78) needs to render a box-drawn table.

---

# 77. Phase 38 — Accepting Raw SQL Over the Wire

One new command byte is added to §22's protocol: `SQL_TEXT`, carrying exactly one argument — the raw SQL string — mirroring how MySQL's own wire protocol has a single `COM_QUERY` command carrying raw SQL text, distinct from its lower-level prepared-statement commands.

```java
// protocol/WireProtocol.java — extending §22's decode()
case 8 -> new Command.SqlText(args[0]); // NEW: raw SQL text, parsed server-side
```

```java
// server/TinyDbServer.java — extending §23's handleConnection()
private void handleConnection(Socket client) {
    Transaction connectionTxn = null;
    SqlParser sqlParser = new SqlParser();
    SqlExecutor sqlExecutor = new SqlExecutor(transactionManager, storage);
    try (client; /* streams as in §23 */) {
        while (true) {
            Command command = protocol.decode(in);
            if (command instanceof Command.SqlText sqlText) {
                SqlStatement statement = sqlParser.parse(sqlText.sql());                 // §74
                if (statement instanceof SqlStatement.TxnBegin) connectionTxn = transactionManager.begin();
                TabularResult result = sqlExecutor.execute(statement, connectionTxn);    // §75
                if (statement instanceof SqlStatement.TxnCommit || statement instanceof SqlStatement.TxnRollback) {
                    connectionTxn = null;
                }
                protocol.encodeTabularResult(out, result);                              // §76
            } else {
                // §21's original non-SQL Command path (still used directly by the JDBC driver, §25) is unchanged
                Object result = queryEngine.execute(command, connectionTxn);
                protocol.encodeResult(out, result);
            }
            out.flush();
        }
    } catch (EOFException | IOException e) { /* as in §23 */ }
}
```

Both paths — the original binary `Command` protocol (§21–§22) and the new `SQL_TEXT` path — end up calling into the *same* `TransactionManager`, so a transaction begun via raw SQL from the CLI (§78) and one begun via the JDBC driver (§25) are indistinguishable to the storage engine underneath.

---

# 78. Phase 39 — TinyDB CLI: A Terminal Client Like `mysql`

```java
// cli/TinyDbShell.java
public class TinyDbShell {
    public static void main(String[] args) throws IOException {
        CliOptions opts = CliOptions.parse(args); // -h host, -P port, -d database
        try (TinyDbClient client = new TinyDbClient(opts.host(), opts.port())) {
            System.out.println("Welcome to the TinyDB monitor.  Commands end with ;");
            BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in));
            StringBuilder buffer = new StringBuilder();
            while (true) {
                System.out.print(buffer.isEmpty() ? "tinydb> " : "      -> "); // multi-line continuation, like mysql
                String line = stdin.readLine();
                if (line == null || line.trim().equalsIgnoreCase("exit") || line.trim().equalsIgnoreCase("\\q")) {
                    System.out.println("Bye");
                    return;
                }
                buffer.append(line).append(' ');
                if (line.trim().endsWith(";")) {
                    runStatement(client, buffer.toString().trim());
                    buffer.setLength(0);
                }
            }
        }
    }

    private static void runStatement(TinyDbClient client, String sql) {
        long start = System.nanoTime();
        try {
            TabularResult result = client.sendSql(sql); // wraps Command.SqlText + protocol.encode, §77
            if (result.isSelect()) {
                printAsciiTable(result);
                System.out.printf("%d row%s in set (%.2f sec)%n%n",
                    result.rowCount(), result.rowCount() == 1 ? "" : "s", elapsedSeconds(start));
            } else {
                System.out.printf("Query OK, %d row%s affected (%.2f sec)%n%n",
                    result.affectedRows(), result.affectedRows() == 1 ? "" : "s", elapsedSeconds(start));
            }
        } catch (IOException e) {
            System.out.println("ERROR: " + e.getMessage());
        }
    }

    /** Box-drawn table output, deliberately matching the mysql CLI's own rendering style. */
    private static void printAsciiTable(TabularResult result) {
        int[] widths = computeColumnWidths(result);
        String separator = buildSeparatorLine(widths);
        System.out.println(separator);
        System.out.println(formatRow(result.columnNames(), widths));
        System.out.println(separator);
        result.rows().forEach(row -> System.out.println(formatRow(row, widths)));
        System.out.println(separator);
    }
    // computeColumnWidths/buildSeparatorLine/formatRow/elapsedSeconds omitted for brevity
}
```

Run it exactly the way you'd run the `mysql` client:

```bash
java -jar tinydb-cli.jar -h localhost -P 9090 -d pureeats
```

```text
Welcome to the TinyDB monitor.  Commands end with ;

tinydb> SELECT * FROM users WHERE id = '42';
+----+-------+
| id | name  |
+----+-------+
| 42 | Alice |
+----+-------+
1 row in set (0.00 sec)

tinydb> INSERT INTO users (id, name) VALUES ('43', 'Bob');
Query OK, 1 row affected (0.01 sec)

tinydb> BEGIN;
Query OK, 0 rows affected (0.00 sec)

tinydb> UPDATE users SET name = 'Robert' WHERE id = '43';
Query OK, 1 row affected (0.00 sec)

tinydb> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)

tinydb> exit
Bye
```

That `BEGIN` / `UPDATE` / `ROLLBACK` sequence is not cosmetic — it runs through §32's real `TransactionManager.rollback()`, replaying §33's undo log, from a terminal session exactly as it would from MiniSpring's `@Repository` code in §26.

---

# 79. Why `SELECT *` Without a WHERE on the Primary Key Is a Full Table Scan

§75's translator makes the cost of a query visible rather than hiding it: `WHERE id = '42'` where `id` is the table's primary key becomes a single `storage.get(...)` — an O(1) hash-index lookup (§16) or an O(log n) B-Tree lookup (§17). `WHERE name = 'Alice'`, or no `WHERE` at all, becomes `storage.scan(table, MIN_KEY, MAX_KEY)` over **every row in the table**, filtered in memory — because TinyDB, as built through §71, has no index on `name`.

```text
SELECT * FROM users WHERE id = '42';       -- O(1)/O(log n): index lookup (§16/§17)
SELECT * FROM users WHERE name = 'Alice';  -- O(n): full table scan + in-memory filter (§75)
SELECT * FROM users;                       -- O(n): full table scan, same cost as the line above with no filter at all
```

This is not a TinyDB-specific limitation — it's exactly what "sequential scan" (`Seq Scan` in a Postgres `EXPLAIN` plan) means in every relational database, and it's precisely why §71 lists **secondary indexes** as the very next V2 enhancement: adding a `HashIndex` or `BTreeIndex` (§16–§17) keyed by `name`, maintained on every write the same way the primary index already is, would turn that second query into an indexed lookup too. Building TinyDB yourself is what makes that connection — "a slow `WHERE` clause means add an index on that column" — something you've implemented, not just memorized as DBA advice.

---

# 80. Phase 40 — Implementing DatabaseMetaData for Tool Compatibility

MiniSpring's `@Repository` beans (§26) only ever call `Connection.prepareStatement(...)`/`executeQuery()` — a narrow slice of JDBC. A generic SQL client like **DBeaver**, IntelliJ's Database tool window, or `SquirrelSQL` is a completely different kind of caller: before it ever runs a query, it calls `connection.getMetaData()` and walks the schema — `getTables(...)`, `getColumns(...)`, `getPrimaryKeys(...)`, `getSchemas()` — to populate the tree view you browse in its UI. §25's `TinyDbConnection` never implemented `getMetaData()` at all; a generic client hitting that gap sees either an empty schema tree or an outright connection failure, not a working query editor.

```java
// jdbc/TinyDbDatabaseMetaData.java — a real, if partial, java.sql.DatabaseMetaData
public class TinyDbDatabaseMetaData implements DatabaseMetaData {
    private final TinyDbConnection connection;
    private final SchemaRegistry schemaRegistry; // the set of Schema objects from §20, one per table (SchemaRegistry itself is formalized in §87, once CREATE TABLE needs somewhere durable to register into)

    public TinyDbDatabaseMetaData(TinyDbConnection connection, SchemaRegistry schemaRegistry) {
        this.connection = connection;
        this.schemaRegistry = schemaRegistry;
    }

    @Override
    public String getDatabaseProductName() { return "TinyDB"; }
    @Override
    public String getDatabaseProductVersion() { return "1.0"; }
    @Override
    public String getDriverName() { return "TinyDB JDBC Driver"; }
    @Override
    public boolean supportsTransactions() { return true; } // §32 really does support them
    @Override
    public String getIdentifierQuoteString() { return "\""; }

    @Override
    public ResultSet getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types) {
        // Every DBeaver-style client calls this FIRST to populate its schema tree.
        List<Row> rows = schemaRegistry.allSchemas().stream()
            .filter(schema -> tableNamePattern == null || matchesSqlLikePattern(schema.getTableName(), tableNamePattern))
            .map(schema -> Row.of(Map.of(
                "TABLE_CAT", "tinydb", "TABLE_SCHEM", "public",
                "TABLE_NAME", schema.getTableName(), "TABLE_TYPE", "TABLE")))
            .toList();
        return new TinyDbMetaResultSet(rows); // a ResultSet backed by an in-memory list, not a live query
    }

    @Override
    public ResultSet getColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern) {
        Schema schema = schemaRegistry.get(tableNamePattern);
        List<Row> rows = schema.getColumnTypes().entrySet().stream()
            .map(entry -> Row.of(Map.of(
                "TABLE_NAME", schema.getTableName(), "COLUMN_NAME", entry.getKey(),
                "TYPE_NAME", entry.getValue().getSimpleName(), "NULLABLE", 1)))
            .toList();
        return new TinyDbMetaResultSet(rows);
    }

    @Override
    public ResultSet getPrimaryKeys(String catalog, String schema, String table) {
        Schema s = schemaRegistry.get(table);
        return new TinyDbMetaResultSet(List.of(Row.of(Map.of(
            "TABLE_NAME", table, "COLUMN_NAME", s.getPrimaryKeyColumn(), "KEY_SEQ", 1))));
    }

    @Override
    public ResultSet getSchemas() { return new TinyDbMetaResultSet(List.of(Row.of(Map.of("TABLE_SCHEM", "public")))); }

    // The remaining ~150 java.sql.DatabaseMetaData methods are stubbed to return an empty/false/0 default —
    // the SAME "implement only the surface a real caller needs" discipline as §25's TinyDbConnection, just
    // applied against a much larger interface because generic tools probe far more of it than MiniSpring does.
}
```

```java
// jdbc/TinyDbConnection.java — one addition to §25
@Override
public DatabaseMetaData getMetaData() { return new TinyDbDatabaseMetaData(this, schemaRegistry); }

@Override
public boolean isValid(int timeoutSeconds) {
    // DBeaver's "Test Connection" button, and its periodic connection-health check, both call this.
    try { client.ping(timeoutSeconds); return true; } catch (IOException e) { return false; } // §77's wire protocol, +1 command byte
}
```

`getTables`/`getColumns` reading from a `SchemaRegistry` — not from a live query against `information_schema` the way Postgres does it — is a deliberate simplification: TinyDB's `Schema` objects (§20) are already an in-memory, authoritative description of every table's shape, so metadata queries are answered directly from that registry rather than by inventing a second schema-storage mechanism.

---

# 81. Phase 41 — Connecting From DBeaver (or Any Generic SQL Client)

With §80's `DatabaseMetaData` in place, TinyDB is a normal-enough JDBC data source for a generic desktop client to drive. The steps are the same ones you'd follow for any unfamiliar JDBC database DBeaver doesn't ship a built-in driver definition for:

1. **Package the driver as a standalone jar** (`tinydb-jdbc-1.0.jar`) containing `TinyDbDriver`, `TinyDbConnection`, `TinyDbStatement`, `TinyDbResultSet`, `TinyDbDatabaseMetaData`, and their dependencies.
2. **Register it for JDBC 4 auto-discovery** by adding a service-loader file inside that jar:
   ```text
   src/main/resources/META-INF/services/java.sql.Driver
   ```
   containing exactly one line:
   ```text
   com.example.tinydb.jdbc.TinyDbDriver
   ```
   This is what lets `java.sql.DriverManager` (and tools built on it, including DBeaver) find `TinyDbDriver` automatically once the jar is on the classpath, without any code calling `Class.forName(...)` explicitly — the same mechanism real drivers like the Postgres JDBC jar or MySQL Connector/J use.
3. **In DBeaver: Database → Driver Manager → New Driver**, and fill in:

   | Field | Value |
   |---|---|
   | Driver Name | `TinyDB` |
   | Class Name | `com.example.tinydb.jdbc.TinyDbDriver` |
   | URL Template | `jdbc:tinydb://{host}:{port}/{database}` |
   | Libraries | Add File → point at `tinydb-jdbc-1.0.jar` |

4. **File → New → Database Connection → TinyDB**, enter the host/port TinyDB's `TinyDbServer` (§23) is listening on, and click **Test Connection** — this calls `connect(url, info)` (§25) and then `isValid(timeout)` (§80), exactly the two methods added/present so far.
5. Once connected, DBeaver's schema navigator calls `getTables`/`getColumns` (§80) to populate the tree, and its SQL editor sends whatever you type straight through as `Command.SqlText` (§77) — `SELECT * FROM users WHERE id = '42'` typed into DBeaver runs through the **exact same** `SqlParser` → `SqlExecutor` → `TransactionManager`/`StorageEngine` path (§74–§75) as the `tinydb-cli` REPL (§78) and MiniSpring's own `@Repository` beans (§26).

```text
                        One driver jar, three completely different callers:

  MiniSpring @Repository  --\
  tinydb-cli terminal REPL --+--> TinyDbDriver (§25) --> TinyDbConnection --> wire protocol (§22, §77) --> TinyDbServer
  DBeaver / any JDBC tool  --/
```

No TinyDB server-side code needed to change to support DBeaver specifically — the entire addition was **more of the standard JDBC interface implemented** (`DatabaseMetaData`, `isValid`), because a generic tool simply calls more of that interface than a purpose-built `@Repository` ever does. This is the practical version of §25's own point: JDBC is a contract, and how much of it you need to implement is determined by who's calling, not by what the database underneath can do.

---

# 82. Design Patterns and Architectural Improvements for a More Scalable TinyDB

Everything through §81 was built to be *correct* — the WAL, the transaction manager, the replication and streaming machinery all behave the way a database should. It was **not** built to be *easy to extend* — most of it hard-wires one concrete implementation where a growing system would want a swappable one. This section names the specific design-pattern refactors that close that gap, each tied to a concrete pain point already visible in the code built so far.

| Pattern | Applied to | What it improves | Ties back to |
|---|---|---|---|
| **Strategy** | Index structure, concurrency control, replication acknowledgement | Swap hash/B-Tree/LSM, locking/MVCC, or sync/async replication via configuration instead of editing `StorageEngine`/`TransactionManager` | §16–§19, §35 vs §38, §43 |
| **Decorator** | Cross-cutting concerns on `StorageEngine` (metrics, caching, encryption) | Add a concern as a new wrapping class, without touching the core engine | §62–§64, §71's encryption-at-rest item |
| **Chain of Responsibility** | The query execution pipeline (parse → validate → plan → execute → publish) | Insert a new stage (a query cache, an authorizer, a slow-query logger) without restructuring `SqlExecutor` | §75, mirrors `MiniFilter` in the [MiniTomcat guide](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) §46 |
| **CQRS** (Command Query Responsibility Segregation) | Splitting the write path (`TransactionManager` → WAL → `StorageEngine`) from a read path served by `MvccStore` snapshots or a replication follower | Read traffic scales independently of write traffic, with no change to write-path correctness | §38, §43 |
| **Event Sourcing** (already present, worth naming) | The WAL (§9) plus the change-event log (§57) | Recognizing that in-memory/indexed state is a *derived projection* of the log opens the door to multiple independent projections (a secondary index, an analytics rollup) replaying the same source of truth | §12, §54 |
| **Circuit Breaker + Bulkhead** | Calls from `ReplicationLeader` to each follower | One slow or unreachable follower can't hang every commit or starve the others | §43, §46 |
| **Facade** (already present, worth protecting) | `QueryEngine`/`SqlExecutor`/`TinyDbConnection` hiding `TransactionManager`, `LockManager`, `MvccStore`, `StorageEngine`, `ReplicationLeader` | Keeps the public surface small as internals grow — any new subsystem should be wired in *behind* this facade, not exposed as one more thing callers must know how to call | §21, §25 |

The two highest-leverage refactors — Strategy for the index, and Strategy for concurrency control — are worth seeing in code, because they're small changes with an outsized effect on Open/Closed compliance (§83):

```java
// storage/IndexStrategy.java — formalizes what §16's HashIndex and §17's BTreeIndex already do implicitly
public interface IndexStrategy {
    void insert(String table, String key, HashIndex.Location location);
    HashIndex.Location lookup(String table, String key);
    void remove(String table, String key);

    /** Not every index can do this — see §83's LSP/ISP discussion for why this isn't just a shared method. */
    default List<HashIndex.Location> rangeScan(String table, String fromKey, String toKey) {
        throw new UnsupportedOperationException(getClass().getSimpleName() + " does not support range scans");
    }
}

// StorageEngine now depends on the INTERFACE, chosen once at startup — this is the Dependency Inversion half of DIP
public class StorageEngine {
    private final IndexStrategy index; // was: a hard-coded `HashIndex` field

    public StorageEngine(IndexStrategy index, WriteAheadLog wal) {
        this.index = index; // HashIndex for point-lookup-heavy tables, BTreeIndex for range-scan-heavy ones — a config decision, not a code change
        this.wal = wal;
    }
}
```

```java
// txn/ConcurrencyControlStrategy.java — formalizes the §35 vs §38 choice §39 only discussed in prose
public interface ConcurrencyControlStrategy {
    void beforeWrite(Transaction txn, String table, String key);
    Row read(Transaction txn, String table, String key);
    void onCommit(Transaction txn);
    void onRollback(Transaction txn);
}

public class PessimisticLockingStrategy implements ConcurrencyControlStrategy { /* delegates to LockManager, §35 */ }
public class MvccStrategy implements ConcurrencyControlStrategy { /* delegates to MvccStore, §38 */ }

// TransactionManager (§32) takes one of these via its constructor instead of calling LockManager directly —
// a workload can now pick per-table or per-connection which strategy to use, without TransactionManager's
// own code ever branching on "which concurrency model are we using."
```

A `QueryPipeline` built as a Chain of Responsibility (the third row above) is the same shape already used for HTTP in the companion guide — worth building the exact same way rather than inventing a new mechanism:

```java
public interface QueryPipelineStage {
    Object handle(SqlStatement statement, Transaction txn, QueryPipelineChain next) throws Exception;
}

// Example additions that become trivial once the pipeline exists — none of this touches SqlExecutor's core logic:
public class QueryLoggingStage implements QueryPipelineStage { /* logs statement + latency, then calls next.handle(...) */ }
public class QueryCacheStage implements QueryPipelineStage { /* returns a cached result for a repeated read-only SELECT */ }
```

---

# 83. SOLID Principles Applied to TinyDB — What Holds and Where It Bends

| Principle | Where TinyDB already honors it | Where the design in §1–§81 bends it | The fix |
|---|---|---|---|
| **S**ingle Responsibility | `WriteAheadLog` only appends/reads log bytes (§9–§11); `LockManager` only manages locks (§35); `UndoLog` only records/replays before-images (§33) — each a narrow, named job | `SqlExecutor` (§75) both *translates* SQL into engine calls **and** *decides* whether a `WHERE` clause becomes an indexed lookup or a full scan — that second job is query planning, a distinct responsibility that will only grow (§71's cost-based planner) | Extract a `QueryPlanner` that `SqlExecutor` calls, so "translate" and "choose an access path" are two classes, not one |
| **O**pen/Closed | `Command`'s `sealed interface` + `switch` (§21) makes adding a new command a compiler-checked, additive change | `StorageEngine` hard-coding `HashIndex`, and `TransactionManager` hard-coding `LockManager`, both require *editing* existing classes to add a new index type or concurrency model | §82's `IndexStrategy` / `ConcurrencyControlStrategy` — new implementations plug in without modifying the classes that use them |
| **L**iskov Substitution | Every `MiniFilter`-style stage (§82's `QueryPipelineStage`) is fully substitutable — any implementation can stand in for any other without breaking the chain | A naive shared `IndexStrategy` interface where `HashIndex.rangeScan(...)` just throws is a classic near-violation: callers can't treat every `IndexStrategy` interchangeably if calling one method might unexpectedly blow up | Split the interface (below, under ISP) so a caller that needs range scans only ever holds a reference to a type that actually supports them — the throwing `default` method above is a pragmatic stopgap, not the end state |
| **I**nterface Segregation | `ObjectPool<T>` (§31) is a good example already in the codebase — three tiny methods, no bloat | `IndexStrategy` bundling `rangeScan` into an interface that a pure hash index can't honestly implement forces an awkward exception-throwing default; `java.sql.DatabaseMetaData` (§80) forces implementing/stubbing ~150 methods to satisfy a caller (DBeaver) that only needs five | Split into `KeyValueIndex` (insert/lookup/remove) and `RangeScannableIndex extends KeyValueIndex` (adds `rangeScan`); `StorageEngine`/`SqlExecutor` check `instanceof RangeScannableIndex` before attempting a range query. `DatabaseMetaData`'s bloat, by contrast, is an *external* JDBC contract TinyDB must conform to, not a TinyDB design choice — worth distinguishing "interfaces we designed" from "interfaces we're required to implement" |
| **D**ependency Inversion | `TinyDbConnection` (§25) depends on the `TinyDbClient` abstraction, not on raw socket code; `ChangeEventPublisher` (§54) is injected into `TransactionManager.commit()`'s call site rather than constructed inline | `TransactionManager` (§32) directly `new`s or field-references a concrete `LockManager`; `StorageEngine` directly references a concrete `HashIndex` | The two constructor-injection changes shown in §82 — both classes should depend on the interfaces (`ConcurrencyControlStrategy`, `IndexStrategy`), with the concrete choice made once, at startup, by whatever assembles the server |

That last row has a natural closing thought: **this is exactly the problem MiniSpring's `@Autowired` constructor injection (companion guide [§17–§21](MiniSpring-Step-by-Step-Guide.md)) already solves.** Once `StorageEngine`, `TransactionManager`, `IndexStrategy`, and `ConcurrencyControlStrategy` are expressed as constructor dependencies rather than hard-coded fields, there's nothing stopping TinyDB's own internals from being assembled as `@Component` beans inside a `MiniApplicationContext` — `@Component class BTreeIndexStrategy implements IndexStrategy`, wired into `StorageEngine` by `@Autowired`, chosen at startup the same way any other bean is. The dependency injection container built in guide #1 of this series and the database built in this guide aren't just deployed side by side through a JDBC connection (§26) — TinyDB's *own* internal wiring is a legitimate customer of exactly that container.

---

# 84. Why CREATE TABLE and Full DDL Support Were Still Missing

A gap worth naming directly: every `Schema`/`Table` object used throughout this guide (§20 onward) was constructed **in Java code**, by hand, before the server ever started. §73's SQL grammar covered `SELECT`/`INSERT`/`UPDATE`/`DELETE` — **DML** (Data Manipulation Language, statements that read/write *rows*) — but never `CREATE TABLE` or `DROP TABLE` — **DDL** (Data Definition Language, statements that define the *shape* rows must conform to). Typing `CREATE TABLE users (...)` into the `tinydb-cli` REPL (§78) or DBeaver's SQL editor (§81) would have failed with `SqlParser`'s "Unsupported SQL statement" error, because nothing in §74's tokenizer/parser recognized the keyword `CREATE` at all.

This wasn't an oversight so much as an intentionally deferred, structurally different problem: DML operates *within* a schema that already exists (§29's consistency checks in `Schema.validate(row)` assume a `Schema` object is already sitting in memory to validate against); DDL is what **creates** that `Schema` object in the first place, and — just like every other write in this guide — a table definition has to be **durable** (survive a crash, §9) and **replicated** (every follower must agree on what tables exist, §43), not just held in a local Java field. §85–§90 build exactly that, reusing every mechanism already in place rather than inventing a parallel one.

---

# 85. Phase 42 — Extending the SQL Grammar With DDL

```text
CREATE TABLE table '(' column type ('PRIMARY' 'KEY')? (',' column type ('PRIMARY' 'KEY')?)* ')'
DROP TABLE table

type := 'VARCHAR' | 'INT' | 'BOOLEAN' | 'TIMESTAMP'
```

Exactly one column may be marked `PRIMARY KEY` — TinyDB's storage engine has been keyed by a single primary key since §6, and nothing in §7–§83 assumes otherwise, so the grammar doesn't pretend to support composite keys.

```java
// sql/SqlStatement.java — two new variants added to §74's sealed interface
public sealed interface SqlStatement {
    // ... Select, Insert, Update, Delete, TxnBegin, TxnCommit, TxnRollback from §74, unchanged ...
    record ColumnDef(String name, String sqlType, boolean primaryKey) { }
    record CreateTable(String table, List<ColumnDef> columns) implements SqlStatement { }
    record DropTable(String table) implements SqlStatement { }
}
```

Because `SqlStatement` is a `sealed interface`, adding these two variants makes every existing `switch` over it (§75's `SqlExecutor.execute`, §89 below) **fail to compile** until each one adds a matching case — the compiler itself enforces that DDL can't be half-wired-in and silently ignored. That's not a coincidence; it's the exact Command-pattern payoff §83's Open/Closed row already named in the abstract, now paying off on a concrete change.

---

# 86. Phase 43 — Parsing CREATE TABLE / DROP TABLE

```java
// sql/SqlParser.java — extending §74's parse() dispatch
public SqlStatement parse(String sql) {
    List<String> tokens = tokenizer.tokenize(sql.trim().replaceAll(";\\s*$", ""));
    String keyword = tokens.get(0).toUpperCase();
    return switch (keyword) {
        case "SELECT" -> parseSelect(tokens);
        case "INSERT" -> parseInsert(tokens);
        case "UPDATE" -> parseUpdate(tokens);
        case "DELETE" -> parseDelete(tokens);
        case "CREATE" -> parseCreateTable(tokens);   // NEW
        case "DROP" -> parseDropTable(tokens);       // NEW
        case "BEGIN" -> new SqlStatement.TxnBegin();
        case "COMMIT" -> new SqlStatement.TxnCommit();
        case "ROLLBACK" -> new SqlStatement.TxnRollback();
        default -> throw new IllegalArgumentException("Unsupported SQL statement: " + sql);
    };
}

private SqlStatement.CreateTable parseCreateTable(List<String> t) {
    // t = ["CREATE", "TABLE", "users", "(", "id", "VARCHAR", "PRIMARY", "KEY", ",", "name", "VARCHAR", ")"]
    String table = t.get(2);
    List<SqlStatement.ColumnDef> columns = new ArrayList<>();
    int i = 4; // skip CREATE TABLE <name> (
    while (!t.get(i).equals(")")) {
        String columnName = t.get(i);
        String sqlType = t.get(i + 1);
        boolean isPrimaryKey = (i + 3 < t.size()) && t.get(i + 2).equalsIgnoreCase("PRIMARY") && t.get(i + 3).equalsIgnoreCase("KEY");
        columns.add(new SqlStatement.ColumnDef(columnName, sqlType, isPrimaryKey));
        i += isPrimaryKey ? 4 : 2;
        if (i < t.size() && t.get(i).equals(",")) i++;
    }
    return new SqlStatement.CreateTable(table, columns);
}

private SqlStatement.DropTable parseDropTable(List<String> t) {
    return new SqlStatement.DropTable(t.get(2)); // t = ["DROP", "TABLE", "users"]
}
```

Same token-walking style as §74's `parseSelect` — no new parsing *technique* was needed, only more cases, which is exactly the point: a well-scoped grammar (§73) and a straightforward recursive-descent parser (§74) scale to new statement types linearly, not by rewriting what's already there.

---

# 87. Phase 44 — The SchemaRegistry: TinyDB's Table Catalog

§80 already *used* a `SchemaRegistry` to answer DBeaver's `getTables()`/`getColumns()` calls, but this guide never actually defined one — that gap is closed here. Every real database has exactly this concept under a different name: Postgres's `pg_catalog`, MySQL's `information_schema`, SQLite's `sqlite_master` table. TinyDB's version is simpler — an in-memory map, not a table-of-tables — but it plays the identical role: **the single source of truth for "what tables exist and what do their rows look like."**

```java
// table/SchemaRegistry.java
public class SchemaRegistry {
    private final Map<String, Schema> schemasByTable = new ConcurrentHashMap<>();

    public void register(Schema schema) {
        if (schemasByTable.containsKey(schema.getTableName())) {
            throw new IllegalStateException("Table already exists: " + schema.getTableName());
        }
        schemasByTable.put(schema.getTableName(), schema);
    }

    public void unregister(String tableName) {
        if (!schemasByTable.containsKey(tableName)) {
            throw new IllegalStateException("No such table: " + tableName);
        }
        schemasByTable.remove(tableName);
    }

    public Schema get(String tableName) {
        Schema schema = schemasByTable.get(tableName);
        if (schema == null) throw new IllegalStateException("No such table: " + tableName);
        return schema;
    }

    public Collection<Schema> allSchemas() { return schemasByTable.values(); }
}
```

This class has exactly one job — hold the current set of table definitions — the same Single Responsibility discipline §83's SRP row already credited to `WriteAheadLog`, `LockManager`, and `UndoLog`. `Schema.validate(row)` (§20, §29) and `TinyDbDatabaseMetaData.getTables()`/`getColumns()` (§80) both now read from this **one** registry instead of each holding their own notion of "what tables exist."

---

# 88. Phase 45 — Making Schema Changes Durable and Replicated

A `CREATE TABLE` that only updates the in-memory `SchemaRegistry` would vanish on the next crash, and a follower would never learn about it — exactly the durability and replication problems §9 and §43 already solved for row data. The fix is to route DDL through the **same WAL**, not a separate mechanism:

```java
// storage/LogRecord.java — extending §10's Type enum
public enum Type { PUT, DELETE, TXN_BEGIN, TXN_COMMIT, TXN_ROLLBACK, CREATE_TABLE, DROP_TABLE }
```

```java
// sql/SqlExecutor.java — the DDL execution path
private TabularResult executeCreateTable(SqlStatement.CreateTable stmt) throws IOException {
    Schema schema = Schema.fromColumnDefs(stmt.table(), stmt.columns()); // builds the Schema object from §20
    byte[] serializedSchema = serialize(schema);
    wal.append(new LogRecord(LogRecord.Type.CREATE_TABLE, 0, stmt.table(), null, serializedSchema)); // durable FIRST (§9)
    schemaRegistry.register(schema);                                                                  // THEN visible
    return TabularResult.ok();
}

private TabularResult executeDropTable(SqlStatement.DropTable stmt) throws IOException {
    wal.append(new LogRecord(LogRecord.Type.DROP_TABLE, 0, stmt.table(), null, null));
    schemaRegistry.unregister(stmt.table());
    storage.scheduleTableForReclamation(stmt.table()); // rows aren't deleted synchronously — see the note below
    return TabularResult.ok();
}
```

Three consequences fall out of reusing the WAL, each one already built for a different purpose:

- **Crash recovery (§12) needs one more `case`.** `recover()`'s replay loop already switches on `record.getType()`; `CREATE_TABLE`/`DROP_TABLE` just add `schemaRegistry.register(...)`/`unregister(...)` alongside the existing `applyPutInMemory`/`applyDeleteInMemory` cases — replayed in the same strict log order as everything else, so a table that was created, written to, and later dropped replays in exactly that sequence, with no special-casing needed.
- **Replication (§43) needs zero changes.** `ReplicationLeader.onRecordAppended` ships *every* WAL record to followers already, regardless of type — a follower applying a `CREATE_TABLE` record calls the same `schemaRegistry.register(...)` the leader did, so "does this table exist" can never disagree between leader and follower.
- **Dropping a table doesn't synchronously delete its rows.** `scheduleTableForReclamation` just marks the table's segments as eligible for the *next* compaction pass (§14) to skip when rewriting — the same lazy-tombstone discipline §50's TTL reaper already uses for expired rows, rather than a second, bespoke "delete everything for this table right now" code path.

---

# 89. Phase 46 — Wiring DDL Into SqlExecutor

```java
// sql/SqlExecutor.java — extending §75's execute() switch
public TabularResult execute(SqlStatement statement, Transaction txn) throws IOException {
    return switch (statement) {
        case SqlStatement.Select s -> executeSelect(s);
        case SqlStatement.Insert s -> { doInsert(s, txn); yield TabularResult.affectedRows(1); }
        case SqlStatement.Update s -> { int n = doUpdate(s, txn); yield TabularResult.affectedRows(n); }
        case SqlStatement.Delete s -> { int n = doDelete(s, txn); yield TabularResult.affectedRows(n); }
        case SqlStatement.CreateTable s -> executeCreateTable(s); // NEW — §88
        case SqlStatement.DropTable s -> executeDropTable(s);     // NEW — §88
        case SqlStatement.TxnBegin s -> TabularResult.ok();
        case SqlStatement.TxnCommit s -> { transactionManager.commit(txn); yield TabularResult.ok(); }
        case SqlStatement.TxnRollback s -> { transactionManager.rollback(txn); yield TabularResult.ok(); }
    };
}
```

Notice what did **not** need to change: `TinyDbServer.handleConnection` (§77), the wire protocol (§22, §76), the `tinydb-cli` REPL (§78), and the JDBC driver (§25) are all completely unaware that a new statement type was added — they all already forward arbitrary SQL text to `SqlExecutor` and render whatever `TabularResult` comes back. Adding DDL support touched exactly three files: the grammar (§85), the parser (§86), and this switch — everything else in the request path was already generic enough not to care.

---

# 90. Full Worked Example: CREATE TABLE Through DROP TABLE, End to End

```text
$ java -jar tinydb-cli.jar -h localhost -P 9090
Welcome to the TinyDB monitor.  Commands end with ;

tinydb> CREATE TABLE users (id VARCHAR PRIMARY KEY, name VARCHAR);
Query OK, 0 rows affected (0.01 sec)

tinydb> INSERT INTO users (id, name) VALUES ('42', 'Alice');
Query OK, 1 row affected (0.00 sec)

tinydb> SELECT * FROM users;
+----+-------+
| id | name  |
+----+-------+
| 42 | Alice |
+----+-------+
1 row in set (0.00 sec)

tinydb> UPDATE users SET name = 'Alicia' WHERE id = '42';
Query OK, 1 row affected (0.00 sec)

tinydb> DELETE FROM users WHERE id = '42';
Query OK, 1 row affected (0.00 sec)

tinydb> DROP TABLE users;
Query OK, 0 rows affected (0.00 sec)

tinydb> exit
Bye
```

Tracing just the first line end to end, since it's the one this whole addition was for: `CREATE TABLE users (...)` is tokenized (§74) and parsed into a `SqlStatement.CreateTable` (§85–§86); `SqlExecutor` (§89) builds a `Schema` (§20), appends a `CREATE_TABLE` `LogRecord` to the WAL and `fsync`s it (§88, reusing §9's durability guarantee), registers it in `SchemaRegistry` (§87), and — because `TinyDbServer` (§23) is a leader — `ReplicationLeader` ships that same WAL record to every follower (§43) without a single line of replication code knowing DDL exists. The exact same statement, typed into DBeaver's SQL editor instead of the CLI (§81), runs through the identical path — `getTables()` on DBeaver's schema tree would show `users` immediately after, because it reads from the same `SchemaRegistry` this `CREATE TABLE` just populated (§80).

---

# 91. How §82's Design Patterns Made This Extension Nearly Free

This is the concrete answer to "how do design patterns help with code segregation and scalability" — not in the abstract, but pointing at what just happened in §85–§89:

- **Command pattern (§21, §74) + Open/Closed (§83):** `SqlStatement` being a `sealed interface` meant the compiler itself refused to let `CreateTable`/`DropTable` be added without a matching case in every `switch` — §85's two-line interface addition to `SqlExecutor.execute()` (§89) is what Open/Closed *actually* looks like: extending behavior by adding a case, not by rewriting existing ones.
- **Single Responsibility (§83) via `SchemaRegistry` (§87):** table definitions now live in exactly one place. Before this section, "what tables exist" was implicitly answered by whatever Java code happened to construct `Schema` objects at startup — an unwritten, easy-to-violate contract. Now `Schema.validate()` (§20), `TinyDbDatabaseMetaData.getTables()` (§80), and `SqlExecutor`'s DDL handlers (§88) all consult the **same** object, so there's exactly one place a bug in "does this table exist" could ever live.
- **Facade (§82) preserved under load:** `TinyDbServer`, the wire protocol, the CLI, and the JDBC driver needed **zero** changes to support DDL (§89's closing paragraph) — because they were already talking to `SqlExecutor` as a facade over the engine, not to `SchemaRegistry`/`WriteAheadLog`/`StorageEngine` directly, adding a whole new *category* of statement (DDL vs DML) never had to leak past that one seam.
- **Where Chain of Responsibility (§82) would pay off next:** `DROP TABLE` is a far more dangerous statement than `SELECT` — a natural next step is a `QueryPipelineStage` (§82) that requires an elevated permission specifically for `SqlStatement.DropTable`/`CreateTable`, inserted in front of `SqlExecutor` without changing a single line inside it. That this is now an *additive* change rather than a scattered set of `if (statement instanceof DropTable)` checks throughout the codebase is the entire scalability argument §82–§83 were making — and DDL support is the first real feature built *after* naming that argument, which is exactly why it went in this smoothly.

---

# 92. Final Takeaway

Every "database feature" in this guide reduces to the same handful of ideas, applied at different points: **log before you leap** (the WAL underpins durability, crash recovery, replication, and the change stream — four features built from one append-only file), **index separately from storage** (a hash index, B-Tree, or LSM-Tree is a lookup accelerator layered on top of the log, never a replacement for it), **make "undo" a first-class citizen** (the undo log is the mirror image of the redo log, not an afterthought), and **treat every guarantee as a specific, nameable mechanism** rather than a label — atomicity is a commit-marker check on replay, isolation is a lock table or a version chain, durability is an `fsync` call you can point to in the code.

Once TinyDB is wired into MiniSpring through a real JDBC driver (§25–§26), the entire stack this series of guides has built — [MiniTomcat](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) accepting the HTTP request, [MiniSpring](MiniSpring-Step-by-Step-Guide.md) routing it to a `@Controller` and injecting a `@Repository`, and TinyDB durably, transactionally, and observably persisting the result — is a complete, if miniature, version of a production web application's full request path, with no black boxes left between the browser and the disk.

And because that same driver speaks standard JDBC rather than a MiniSpring-specific API, TinyDB is reachable from every direction a real database would be: MiniSpring's own `@Repository` beans (§26), a `mysql`-style terminal session (§78), and a generic desktop tool like DBeaver browsing its schema and running ad-hoc SQL (§80–§81) — three completely different callers, one unmodified server underneath.

The one thing worth carrying forward past this guide is §82–§83's honest audit: correctness and extensibility are different achievements, and TinyDB earned the first without automatically earning the second. Turning `StorageEngine`'s and `TransactionManager`'s hard-coded concrete dependencies into injected `IndexStrategy`/`ConcurrencyControlStrategy` interfaces is a small, mechanical change — and it's the same change that would let TinyDB's own internals be wired up by the very `@Autowired` container this series started with, closing the loop between all three guides.

