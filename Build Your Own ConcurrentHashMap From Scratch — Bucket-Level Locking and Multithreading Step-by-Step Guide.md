# Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide

> **Goal:** Build a thread-safe hash map from scratch, in plain Java, that ends up architecturally close to the real `java.util.concurrent.ConcurrentHashMap` — starting from a single-threaded `HashMap`, through the *wrong but instructive* full-lock approach (`Hashtable`'s design), through **lock striping** (Java 7's `Segment`-based `ConcurrentHashMap`), and finally to **per-bucket locking with CAS for the uncontended path** (the real Java 8+ redesign) — deriving *why* each step exists from a concrete concurrency bug the previous step has.
>
> This guide assumes working knowledge of Java generics and basic multithreading (`Thread`, `synchronized`) but explains every concurrency primitive (CAS, `volatile`, the Java Memory Model) from first principles — you should not need outside references to follow the derivations.

---

# 1. What We Are Building

We are building **MiniConcurrentMap** — a hash map that is safe to read and write from many threads at once, without serializing every operation behind one global lock. By the end of this guide you will have:

- A correct **single-threaded** hash map (bucket array, separate chaining, resize) — the baseline everything else modifies.
- A working demonstration of **exactly how** a plain `HashMap` breaks under concurrent access, including the infamous Java 7 resize infinite-loop bug.
- A **lock-striped** map (Java 7 `ConcurrentHashMap`'s `Segment` design) — one lock per *segment* of the table, giving genuine concurrent writes.
- A **per-bucket-locked, CAS-accelerated** map (the real Java 8+ redesign) — no lock at all for the common case (inserting into an empty bucket), a `synchronized` block scoped to a *single bucket* otherwise.
- **Treeification** — converting a badly-collided bucket's linked list into a red-black tree, exactly as real `ConcurrentHashMap` does, to bound worst-case lookup time.
- **Concurrent, cooperative resizing** — multiple threads splitting the table together, safely, without a stop-the-world pause.
- Correct **size counting** under contention (why a single shared counter doesn't scale, and the striped-counter fix), and correct **atomic compound operations** (`computeIfAbsent`, `merge`) that a naive "get then put" implementation gets wrong.

```text
Single-threaded HashMap (§6-§12)
        |
        v
"Just synchronize everything" — Hashtable-style (§16-§17)  <- correct, but serializes ALL access
        |
        v
Lock striping — Segment[] , one lock per segment (§25-§30)  <- N-way concurrency, N fixed at construction
        |
        v
Per-bucket locking + CAS — Java 8+ design (§31-§47)  <- concurrency scales with TABLE SIZE, not a fixed N
        |
        v
+ Treeification (§38-§41) + Striped counters (§48-§50) + Weakly consistent iteration (§53-§55)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to answer, from first principles:

- Exactly what goes wrong, mechanically, when two threads call `put()` on a plain `HashMap` at the same time — not just "it's not thread-safe," but the actual corrupted state that results.
- Why `Hashtable`'s "synchronize every method" approach is *correct* but throughput-limited, and what specifically it serializes that doesn't need to be.
- What "lock striping" means, why it bounds concurrency at a fixed number of segments, and why Java 8 replaced it.
- What a CAS (compare-and-swap) instruction actually does at the hardware level, and why it lets `ConcurrentHashMap` insert into an empty bucket with no lock at all.
- Why `volatile` is necessary but not sufficient for the table array and node fields, and what specific bug appears if you leave it out.
- How resizing can happen *while other threads are reading and writing the map*, without either corrupting data or blocking every other thread.
- Why `ConcurrentHashMap`'s iterators never throw `ConcurrentModificationException`, and what "weakly consistent" actually promises (and doesn't promise) as a result.

---

# 3. Why Build This? (Interview Motivation)

> **"Implement a thread-safe hash map. Explain the tradeoffs between locking the entire map, locking segments of it, and locking individual buckets. Then explain how you would resize the table without blocking every other thread, and how `size()` can be computed correctly and efficiently under concurrent modification."**

This is a **staple senior/staff Java interview question** because it forces you to reason about several distinct concurrency ideas at once, each individually common in interviews but rarely combined:

- **Lock granularity** — the direct tradeoff between "easy to prove correct" (one big lock) and "scales with contention" (many small locks), made concrete rather than abstract.
- **Lock-free programming** — CAS as an alternative to locking for the specific case where it's sufficient (an empty slot), and knowing precisely which case that is.
- **The Java Memory Model** — why `volatile` exists, what "happens-before" means, and why getting this wrong produces bugs that only appear under real concurrent load, never in a single-threaded test.
- **Amortized/cooperative algorithms** — resizing an enormous table without a stop-the-world pause is the same shape of problem as concurrent garbage collection or online index rebuilding.
- **Correct concurrent counting** — why `size()` on a highly concurrent structure is a genuinely hard problem, not a one-line `AtomicInteger` fix.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | `VarHandle` gives modern, explicit access to volatile reads/CAS (§21); `synchronized`/`Thread` cover everything else. |
| Concurrency primitives | `java.lang.invoke.VarHandle` (CAS, volatile get/set), `synchronized` | The same primitives (in spirit — real `ConcurrentHashMap` predates `VarHandle` and uses `sun.misc.Unsafe`) that back the real implementation. |
| No external concurrency libraries | — | The entire point is to build the primitives real `java.util.concurrent` classes are built from, not to call them. |
| Testing | JUnit 5 + a hand-rolled multi-threaded stress harness | Concurrency bugs are probabilistic — a single-threaded unit test cannot catch a race condition; §59 builds a harness that actually can, most of the time. |

---

# 5. Project Structure

```text
miniconcurrentmap/
├── src/main/java/com/example/miniconcurrentmap/
│   ├── singlethreaded/
│   │   ├── Node.java
│   │   └── SimpleHashMap.java
│   ├── segmented/
│   │   ├── Segment.java
│   │   └── SegmentedConcurrentMap.java
│   ├── bucketlocked/
│   │   ├── Node.java                  // volatile val/next, hash field
│   │   ├── TreeNode.java              // red-black tree node, §40
│   │   ├── TreeBin.java               // reader-writer lock wrapper around a tree, §41
│   │   ├── ForwardingNode.java        // resize marker, §43
│   │   ├── CounterCell.java           // striped counter, §49
│   │   └── MiniConcurrentMap.java     // the main class
│   └── bench/
│       └── ThroughputBenchmark.java
└── src/test/java/com/example/miniconcurrentmap/
    ├── SimpleHashMapTest.java
    ├── ConcurrentStressTest.java
    ├── ResizeUnderLoadTest.java
    └── WeaklyConsistentIteratorTest.java
```

---

# 6. Phase 1 — The Bucket Array and the Node

Every hash map, concurrent or not, starts from the same idea: an array of **buckets**, where a key's hash code decides which bucket it lives in, and each bucket holds a small collection (here, a linked list) of the entries that landed there.

```java
// singlethreaded/Node.java
public class Node<K, V> {
    final int hash;
    final K key;
    V val;
    Node<K, V> next; // separate chaining — colliding keys form a linked list within one bucket

    Node(int hash, K key, V val, Node<K, V> next) {
        this.hash = hash; this.key = key; this.val = val; this.next = next;
    }
}
```

```java
// singlethreaded/SimpleHashMap.java
public class SimpleHashMap<K, V> {
    private Node<K, V>[] table;
    private int size;
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;

    @SuppressWarnings("unchecked")
    public SimpleHashMap() {
        table = (Node<K, V>[]) new Node[DEFAULT_CAPACITY];
    }
}
```

---

# 7. Hashing: From hashCode() to a Bucket Index

A key's `hashCode()` is an arbitrary 32-bit integer — it can be negative, and its range has no relationship to the table's current size. Turning it into a valid array index needs two steps: make it non-negative, then fold it into range.

```java
private int bucketIndex(int hash, int tableLength) {
    return (hash & (tableLength - 1)); // tableLength is ALWAYS a power of two — see why below
}
```

`hash & (tableLength - 1)` is a bitwise trick that's exactly equivalent to `hash % tableLength`, but only when `tableLength` is a power of two — because a power of two's binary representation is a single `1` bit followed by zeros, so `tableLength - 1` is a solid run of `1` bits (a "mask") that keeps exactly the low bits of `hash` and zeroes the rest, which is precisely what modulo-by-a-power-of-two does. A bitwise AND is markedly cheaper than a modulo (`%`) operation, which is why every real hash map implementation — this one included — constrains its table size to powers of two specifically to unlock this optimization.

---

# 8. Why We Spread (Re-Hash) the Hash Code

Using `key.hashCode()` directly has a subtle problem: `bucketIndex` only looks at the **low bits** of the hash (per §7's masking trick) — if many keys' hash codes differ only in their *high* bits (a realistic scenario for some hash function implementations, notably `Integer.hashCode()` for values that are multiples of the table size), they'd all collide into the same bucket despite having "different" hash codes overall.

```java
private int spread(int hashCode) {
    return hashCode ^ (hashCode >>> 16); // mix the high 16 bits into the low 16 bits via XOR
}
```

XOR-ing the hash code with its own upper half folds high-bit entropy down into the low bits `bucketIndex` actually consults — a cheap way to substantially reduce collisions caused by poor-quality or adversarially-crafted hash codes, and exactly the technique real `HashMap`/`ConcurrentHashMap` use (their actual `spread()` method is this exact line).

---

# 9. Phase 2 — get(), put(), and remove() on a Single-Threaded Map

```java
public V get(K key) {
    int idx = bucketIndex(spread(key.hashCode()), table.length);
    for (Node<K, V> node = table[idx]; node != null; node = node.next) {
        if (node.key.equals(key)) return node.val;
    }
    return null;
}

public V put(K key, V value) {
    int hash = spread(key.hashCode());
    int idx = bucketIndex(hash, table.length);
    for (Node<K, V> node = table[idx]; node != null; node = node.next) {
        if (node.key.equals(key)) {
            V old = node.val;
            node.val = value;
            return old;
        }
    }
    table[idx] = new Node<>(hash, key, value, table[idx]); // prepend — O(1), no need to walk to the tail
    size++;
    if (size > table.length * LOAD_FACTOR) resize(); // §11-§12
    return null;
}

public V remove(K key) {
    int idx = bucketIndex(spread(key.hashCode()), table.length);
    Node<K, V> prev = null;
    for (Node<K, V> node = table[idx]; node != null; prev = node, node = node.next) {
        if (node.key.equals(key)) {
            if (prev == null) table[idx] = node.next; else prev.next = node.next;
            size--;
            return node.val;
        }
    }
    return null;
}
```

Every one of these methods reads and mutates shared state (`table`, a bucket's linked list, `size`) with **zero synchronization** — completely correct for one thread, and, as §13 makes concrete, actively dangerous the moment a second thread calls any of these concurrently.

---

# 10. Collision Resolution: Separate Chaining

The linked list at each bucket (§6's `Node.next`) is called **separate chaining** — as opposed to *open addressing* (probing for a different empty slot in the array itself when a collision occurs). Chaining is what real `HashMap`/`ConcurrentHashMap` use, for a reason that becomes directly relevant to concurrency: a chain is a small, **independent** data structure per bucket, which is exactly the unit of granularity §34's bucket-level locking will lock — open addressing, where one key's probe sequence can wander through many other keys' "home" slots, has no such clean per-bucket boundary to lock.

---

# 11. Load Factor and Resizing: When and Why

As more entries are added to a fixed-size table, chains get longer, and `get`/`put`'s "walk the chain" cost degrades from O(1) toward O(n). The **load factor** (default `0.75`) is the threshold — `size / capacity` — past which the table doubles in size and every existing entry is redistributed into the new, larger table, shortening chains back down.

---

# 12. Phase 3 — Implementing resize()

```java
@SuppressWarnings("unchecked")
private void resize() {
    Node<K, V>[] oldTable = table;
    Node<K, V>[] newTable = (Node<K, V>[]) new Node[oldTable.length * 2];
    for (Node<K, V> head : oldTable) {
        Node<K, V> node = head;
        while (node != null) {
            Node<K, V> next = node.next;                                  // save BEFORE we mutate node.next below
            int newIdx = bucketIndex(node.hash, newTable.length);
            node.next = newTable[newIdx];                                  // re-link into the new table's bucket
            newTable[newIdx] = node;
            node = next;
        }
    }
    table = newTable;
}
```

Saving `next` **before** overwriting `node.next` is not optional — reusing the existing `Node` objects (rather than allocating fresh copies) is a nice efficiency win, but it means the very field you're about to overwrite is also the field you need to know where to walk next. §14 shows exactly what happens when two threads run a version of this loop concurrently without protection.

---

# 13. Race Conditions on a Shared Bucket Array

Two threads calling `put()` on the *same bucket* at the *same time* (§9) can both read `table[idx]` before either writes back their new node — both then compute `new Node<>(hash, key, value, table[idx])` from the *same* stale head, and both write `table[idx] = ...`. Whichever write happens last **wins**, silently discarding the other thread's insert — a **lost update**, with no exception, no warning, and no way to detect after the fact that it happened. This is the single most common concrete failure mode of using a plain `HashMap` from multiple threads, and it can go unnoticed for a long time precisely because it requires an unlucky interleaving to manifest.

---

# 14. The Classic Java 7 HashMap Resize Infinite-Loop Bug

A more dramatic failure, famous enough to be a standing interview question on its own: Java 7's `HashMap.resize()` rebuilt each bucket's chain by **prepending** nodes to the new bucket in the *same order* it walked the old one — which, done carelessly, **reverses** the chain's order on every resize. If two threads both call `resize()` concurrently on the same map, one thread's half-completed pointer rewiring can be interleaved with the other's, producing a **circular linked list** — `A.next = B` and `B.next = A`, with no `null` terminator at all. A subsequent `get()` call that walks into that cycle **loops forever**, pegging a CPU core at 100% with no exception and no crash — often the very first symptom that alerts a team to the bug, in production, under load. Java 8 changed the resize algorithm's node-splitting order specifically to no longer reverse chains (a partial mitigation), but the deeper lesson stands: **no version of unsynchronized resize is safe under concurrent modification** — the fix is never a cleverer single-threaded algorithm, it's genuine concurrency control, which is what §25 onward builds.

---

# 15. Lost Updates: Two Threads, One put(), One Missing Entry

A minimal reproduction of §13's lost-update bug, useful as a first stress test once §25's `SegmentedConcurrentMap` exists to compare against:

```java
SimpleHashMap<Integer, Integer> map = new SimpleHashMap<>();
int threadCount = 8, insertsPerThread = 10_000;
ExecutorService pool = Executors.newFixedThreadPool(threadCount);
CountDownLatch done = new CountDownLatch(threadCount);
for (int t = 0; t < threadCount; t++) {
    final int base = t * insertsPerThread;
    pool.submit(() -> {
        for (int i = 0; i < insertsPerThread; i++) map.put(base + i, base + i);
        done.countDown();
    });
}
done.await();
// Expected: map.size() == threadCount * insertsPerThread
// Actual, run repeatedly: a smaller number, different each run — lost updates, and/or a hung thread from §14's cycle
```

Every key in this test is **unique** across threads — there's no logical reason for any insert to be lost. The failures that appear anyway are the concurrency bugs in §13–§14 made visible, not a flaw in the test.

---

# 16. Why "Just Add synchronized to Everything" Is Correct But Slow (Hashtable's Approach)

The simplest fix: wrap every public method's body in `synchronized`, exactly as `java.util.Hashtable` (predating `ConcurrentHashMap` by years) does:

```java
public synchronized V get(K key) { /* same body as §9 */ }
public synchronized V put(K key, V value) { /* same body as §9 */ }
public synchronized V remove(K key) { /* same body as §9 */ }
```

This is **genuinely correct** — a `synchronized` method acquires the map's intrinsic lock for its entire duration, so no two threads can ever be inside `get`/`put`/`remove` at the same time, which eliminates every race condition in §13–§15 by construction. The problem isn't correctness — it's that this lock is **one single lock for the entire map**: a `put()` to bucket 3 and a completely unrelated `get()` on bucket 47 are forced to run one after another, even though they touch entirely disjoint memory and have no actual reason to conflict.

---

# 17. Coarse-Grained Locking and the Throughput Ceiling

Under this design, **throughput cannot exceed what one thread, running alone, could achieve** — adding more CPU cores or more concurrent threads makes zero difference to peak throughput once the lock is the bottleneck, because every operation is fully serialized regardless of how many threads are waiting. This is the concrete cost §16 is naming, and the entire motivation for everything from §25 onward: **the correctness §16 provides is not in question — only its granularity is.** Every subsequent design in this guide keeps §16's core guarantee (no two conflicting operations run concurrently) while shrinking *what counts as conflicting* from "the entire map" down to progressively smaller, more precise units.

---

# 18. The Java Memory Model in Three Rules

Before building anything finer-grained than §16's single lock, three facts about how Java actually executes concurrent code are load-bearing for everything that follows:

1. **Each CPU core may cache a variable's value locally**, and a write from one thread is not guaranteed to be visible to another thread just because it "already happened" in wall-clock time — without explicit synchronization, a reading thread may see a stale, cached value indefinitely.
2. **The compiler and CPU are allowed to reorder instructions** that don't affect single-threaded correctness — code that looks sequential in source order may not execute in that order from another thread's point of view, unless a memory barrier prevents it.
3. **`synchronized`, `volatile`, and `java.util.concurrent.atomic`/`VarHandle` operations each establish a "happens-before" relationship** — a guarantee that a write before the synchronization point is visible to a read after it, on another thread. Without one of these, there is **no** visibility guarantee at all, regardless of how "obviously" a write should be seen.

Every design decision from §19 onward — marking a field `volatile`, choosing exactly what a `synchronized` block spans — is a decision about **which** happens-before relationship to establish, and where.

---

# 19. The volatile Keyword: Visibility Without Atomicity

```java
private volatile Node<K, V>[] table;
```

`volatile` guarantees two things, and **only** two things: every read of the field sees the most recent write from *any* thread (visibility, per §18's rule 1), and reads/writes of that field cannot be reordered relative to surrounding code (per rule 2). It does **not** make a compound operation atomic — `volatileCounter++` is still a read, an increment, and a write as three separate steps, and two threads can still interleave those three steps and lose an update, exactly as in §13, even though the field itself is `volatile`. This distinction — visibility versus atomicity — is the single most common point of confusion about `volatile`, and getting it wrong is precisely how a "volatile size counter" (§48) turns out not to solve the concurrent-counting problem at all.

---

# 20. Why a Bucket Array Reference Must Be volatile

When §12's `resize()` runs, it builds an entirely new array and reassigns the `table` field to point at it. Without `volatile` on `table`, another thread's `get()` — reading `table` with no synchronization of its own — has **no guarantee** of ever observing that reassignment; it could keep reading the old, stale array reference forever (per §18's rule 1), silently missing every entry that only exists in the new table. Marking `table` `volatile` guarantees that once `resize()` publishes the new array reference, every subsequent read by any thread sees it — the foundational visibility guarantee every design from §25 onward depends on.

---

# 21. Compare-And-Swap (CAS): The Hardware Primitive

**CAS** is a single, hardware-supported atomic instruction (`cmpxchg` on x86) with the semantics: *"if the memory location currently holds the expected value, atomically replace it with a new value and report success; otherwise, change nothing and report failure."* Critically, this check-and-set happens as **one indivisible hardware operation** — no other thread can observe or interleave with it partway through, which is exactly what makes it usable as a building block for lock-free algorithms.

```java
// java.lang.invoke.VarHandle — a modern, explicit way to issue a CAS
private static final VarHandle TABLE_SLOT; // set up once via VarHandle.compareAndSet-capable array access

boolean casTabAt(Node<K, V>[] table, int index, Node<K, V> expected, Node<K, V> newValue) {
    return TABLE_SLOT.compareAndSet(table, index, expected, newValue);
}
```

The classic use: "insert into this array slot **only if** it's still `null`" — read the current value, compute the new node, then CAS it in with `expected = null`. If another thread beat you to it, the CAS fails (the slot is no longer `null`), and you find out **immediately**, without ever having taken a lock at all.

---

# 22. Using CAS Instead of a Lock for the Uncontended Case

CAS's real payoff is specific and narrow: it's cheap and effective **exactly when contention is low** — the common case of "insert a brand-new key into an empty bucket" almost never has two threads racing for the *same* bucket at the *same* instant, so a CAS almost always succeeds on the first try, with none of a lock's overhead (no OS-level thread parking, no context switch if uncontended). Under **high** contention, repeated CAS failures and retries can actually cost *more* than a lock would have — which is precisely why §34's design uses CAS **only** for the empty-bucket case and falls back to a real `synchronized` block the moment a bucket already has an entry in it (i.e., the moment genuine contention on that specific bucket becomes plausible).

---

# 23. synchronized Blocks: What They Actually Guarantee

`synchronized(lockObject) { ... }` guarantees **mutual exclusion** (only one thread executes the block, for a given `lockObject`, at a time) **and** a happens-before relationship (§18's rule 3) — everything a thread wrote before releasing the lock is visible to the next thread that acquires it. The granularity is entirely up to *what object you synchronize on* — `synchronized` on `this` (§16) locks out the whole object; `synchronized` on a single bucket's first node (§34) locks out only operations on that one bucket. The keyword's mechanics never change; only the scope of what's being protected does.

---

# 24. Deadlock, Livelock, and Lock Ordering

Locking more than one thing at once introduces a new failure mode this guide must design around from §25 onward: **deadlock**, where thread A holds lock 1 and waits for lock 2, while thread B holds lock 2 and waits for lock 1 — neither can ever proceed. The standard defenses, both used implicitly in the designs that follow:

- **Never hold more than one bucket/segment lock at a time** — §29's `SegmentedConcurrentMap` and §34's per-bucket locking both acquire exactly one lock per operation, never nested, which makes deadlock structurally impossible regardless of ordering.
- **When an operation genuinely must touch two locks** (§46's resize, which must coordinate an old bucket and its destination in the new table), always acquire them in a **fixed, consistent order** across every thread — the classic fix for the "two locks, opposite order" deadlock shape, and the reason §46's transfer logic processes buckets by increasing index rather than in an arbitrary order.

---

# 25. The Segment: A Mini Hash Table With Its Own Lock

**Lock striping** (Java 7's `ConcurrentHashMap` design) generalizes §16's single lock into **N independent locks**, by splitting the table into N **segments** — each segment is a small, self-contained hash table (its own bucket array) guarded by its own lock. A key is routed to exactly one segment (deterministically, by another slice of its hash code), so two keys in *different* segments can be written to concurrently, genuinely in parallel, with zero coordination between them.

```java
// segmented/Segment.java
public class Segment<K, V> {
    private final ReentrantLock lock = new ReentrantLock();
    private volatile Node<K, V>[] table; // volatile — §20's reasoning applies per-segment now, not globally

    @SuppressWarnings("unchecked")
    Segment(int initialCapacity) { table = (Node<K, V>[]) new Node[initialCapacity]; }

    V put(int hash, K key, V value) {
        lock.lock();                          // ONLY this segment's operations block each other
        try {
            int idx = hash & (table.length - 1);
            for (Node<K, V> node = table[idx]; node != null; node = node.next) {
                if (node.key.equals(key)) { V old = node.val; node.val = value; return old; }
            }
            table[idx] = new Node<>(hash, key, value, table[idx]);
            return null;
        } finally {
            lock.unlock();                    // ALWAYS in a finally — an exception must never leave the lock held
        }
    }
}
```

Each `Segment` is, on its own, exactly a smaller version of §16's fully-synchronized map — the innovation is entirely in *how many* of these independent, separately-locked units exist side by side.

---

# 26. Phase 4 — Implementing a Segment

A complete `Segment` needs `get`/`remove` alongside §25's `put`, following the identical pattern — lock, operate, unlock in a `finally`:

```java
V get(int hash, K key) {
    // Deliberately NO lock here — see §29's note on why a lock-free read is still safe against a
    // volatile table and volatile Node fields, the same reasoning §36 develops in full for the bucket-locked design.
    Node<K, V>[] tab = table;
    int idx = hash & (tab.length - 1);
    for (Node<K, V> node = tab[idx]; node != null; node = node.next) {
        if (node.key.equals(key)) return node.val;
    }
    return null;
}

V remove(int hash, K key) {
    lock.lock();
    try {
        int idx = hash & (table.length - 1);
        Node<K, V> prev = null;
        for (Node<K, V> node = table[idx]; node != null; prev = node, node = node.next) {
            if (node.key.equals(key)) {
                if (prev == null) table[idx] = node.next; else prev.next = node.next;
                return node.val;
            }
        }
        return null;
    } finally {
        lock.unlock();
    }
}
```

---

# 27. Routing a Key to Its Segment

A key's hash needs to be split into two independent pieces: **which segment** it belongs to, and **which bucket within that segment**. Using different bits of the hash for each avoids the two decisions from being correlated (which would defeat the point of spreading load across segments):

```java
// segmented/SegmentedConcurrentMap.java
private final Segment<K, V>[] segments;
private final int segmentMask; // segments.length - 1, segments.length is a power of two, same reasoning as §7

private Segment<K, V> segmentFor(int hash) {
    int segmentHash = hash >>> (32 - segmentShift); // use the HIGH bits for segment selection
    return segments[segmentHash & segmentMask];
}
```

Using the **high** bits for segment selection and letting each segment's own `table.length - 1` mask (§7) consume the **low** bits for its internal bucket index means the two decisions draw from non-overlapping parts of the hash — two keys landing in the same segment are still well-distributed across that segment's own buckets.

---

# 28. Why Splitting Into N Segments Gives (Up To) N-Way Write Concurrency

If keys are reasonably well distributed across segments (which §27's hash-splitting aims for), N threads writing to N *different* segments run **fully in parallel** — no shared lock, no waiting. The realistic ceiling is exactly `min(threadCount, segmentCount)`: more threads than segments means some threads *must* contend for the same segment's lock (two keys hashing to the same segment, regardless of how many threads exist), and more segments than active threads provides no additional benefit, since there's no contention left to relieve. This is why Java 7's `ConcurrentHashMap` exposed a `concurrencyLevel` constructor parameter — a hint for "how many threads do you expect to write concurrently," used to size the segment array.

---

# 29. Phase 5 — Assembling Segments Into a ConcurrentHashMap

```java
// segmented/SegmentedConcurrentMap.java
public class SegmentedConcurrentMap<K, V> {
    private final Segment<K, V>[] segments;
    private final int segmentMask, segmentShift;

    @SuppressWarnings("unchecked")
    public SegmentedConcurrentMap(int concurrencyLevel) {
        int segmentCount = tableSizeFor(concurrencyLevel); // rounds up to the next power of two — §7's constraint, again
        this.segmentMask = segmentCount - 1;
        this.segmentShift = 32 - Integer.numberOfTrailingZeros(segmentCount);
        this.segments = new Segment[segmentCount];
        for (int i = 0; i < segmentCount; i++) segments[i] = new Segment<>(16);
    }

    public V put(K key, V value) {
        int hash = spread(key.hashCode()); // §8
        return segmentFor(hash).put(hash, key, value);
    }

    public V get(K key) {
        int hash = spread(key.hashCode());
        return segmentFor(hash).get(hash, key);
    }
    // remove(...), segmentFor(...) — §27
}
```

Every public method is now a two-step dispatch: find the right `Segment` (a pure hash computation, no locking), then delegate to that segment's own, independently-locked implementation. This is the entire lock-striping design — deliberately no more complex than that, and precisely why it was the production `ConcurrentHashMap` design for over a decade (Java 5 through 7) before Java 8's redesign.

---

# 30. The Cost of Lock Striping: Fixed Concurrency Level, Per-Segment Resize

Two real limitations, both of which motivate §31's redesign directly:

- **The number of segments is fixed at construction** (`concurrencyLevel`, §29). Guess too low, and you've recreated §17's bottleneck at a smaller scale once real concurrent load exceeds your guess; guess too high, and you've allocated more table arrays and lock objects than the workload will ever use.
- **Each segment resizes independently** (§12's algorithm, run per-segment) — which avoids a single enormous stop-the-world resize, but means segments can end up wildly different sizes over time if keys don't distribute perfectly evenly, and a hot segment's resize still fully blocks *that* segment's traffic while it runs, even though every other segment stays available.

Both limitations trace back to the same root cause: **the unit of locking (a whole segment, containing many buckets) is coarser than it needs to be.** §31 removes segments entirely and locks individual buckets directly — the concurrency granularity becomes "however many buckets the table currently has," which grows automatically with the table itself, with no `concurrencyLevel` guess required.

---

# 31. Why Java 8 Abandoned Segments for Per-Bin Locking

Java 8's `ConcurrentHashMap` rewrite eliminates `Segment` entirely: there is **one** table, and the lock for a given operation is **the first `Node` in the bucket that operation touches** — no separate lock object per bucket even needs to be allocated, since every bucket already has a natural, unique object (its head node) to synchronize on. This gives concurrency granularity equal to the **number of buckets** (which is always at least as large as, and typically far larger than, any reasonable `concurrencyLevel` would have been) — and it grows automatically every time the table resizes, with no separate "segment resize" concept needed at all. The remaining sections in this part (§32–§37) build this design piece by piece.

---

# 32. Phase 6 — The Node and the volatile Fields That Make It Safe

```java
// bucketlocked/Node.java
public class Node<K, V> {
    final int hash;
    final K key;
    volatile V val;          // volatile: a get() on another thread must see a just-completed put()'s value
    volatile Node<K, V> next; // volatile: a get() walking the chain must see a just-linked node

    Node(int hash, K key, V val, Node<K, V> next) {
        this.hash = hash; this.key = key; this.val = val; this.next = next;
    }
}
```

Compare this to §6's single-threaded `Node` — `val` and `next` are now `volatile`. This single change is what makes §36's **lock-free `get()`** correct: without it, a `get()` reading `val`/`next` with no synchronization of its own would have no visibility guarantee (§18's rule 1) into a concurrent `put()`'s write, `volatile` or not on the surrounding array.

---

# 33. Phase 7 — A CAS-Based Insert Into an Empty Bucket

```java
// bucketlocked/MiniConcurrentMap.java
public V put(K key, V value) {
    int hash = spread(key.hashCode());
    Node<K, V>[] tab = table;
    int idx = (tab.length - 1) & hash;

    Node<K, V> first = tabAt(tab, idx); // volatile read — §21's VarHandle-based accessor
    if (first == null) {
        Node<K, V> newNode = new Node<>(hash, key, value, null);
        if (casTabAt(tab, idx, null, newNode)) {  // §21 — CAS: only succeeds if the bucket is STILL empty
            addCount(1);                          // §50 — striped counter, not a plain increment
            return null;
        }
        // CAS failed: another thread inserted first. Fall through to §34's synchronized path below.
    }
    return putIntoExistingBucket(tab, idx, hash, key, value); // §34
}
```

For the extremely common case of inserting a brand-new key whose bucket happens to be empty, this path **never takes a lock at all** — exactly §22's "CAS wins when contention is low" argument, applied to the specific operation where it pays off most: the first insert into any given bucket.

---

# 34. Phase 8 — synchronized on the Bucket's First Node for a Non-Empty Bucket

```java
private V putIntoExistingBucket(Node<K, V>[] tab, int idx, int hash, K key, V value) {
    Node<K, V> first = tabAt(tab, idx);
    synchronized (first) {                        // lock scoped to THIS bucket's head node — nothing else
        if (tabAt(tab, idx) != first) return putIntoExistingBucket(table, idx, hash, key, value); // re-check, §35
        for (Node<K, V> node = first; ; node = node.next) {
            if (node.hash == hash && node.key.equals(key)) {
                V old = node.val;
                node.val = value;                  // volatile write — visible to a concurrent get(), §36
                return old;
            }
            if (node.next == null) {
                node.next = new Node<>(hash, key, value, null); // volatile write publishes the new node
                addCount(1);
                return null;
            }
        }
    }
}
```

`synchronized (first)` — locking on the bucket's *first node object itself*, not on some separate, purpose-built lock — is the detail that makes per-bucket locking essentially free to set up: no extra `Lock` object needs to be allocated or tracked per bucket, because the bucket's own head node already is a unique, stable-while-locked object to synchronize on.

---

# 35. Why Locking the First Node Is Enough to Protect the Whole Bucket

The re-check `if (tabAt(tab, idx) != first)` immediately inside the `synchronized` block in §34 is not defensive paranoia — it closes a real race: between reading `first` (before the `synchronized` block) and actually acquiring the lock on it, **another thread could have already changed the bucket's head** (e.g. a resize in progress, §43, replaces the head with a `ForwardingNode`, or another `put` on an empty-*again* bucket raced ahead). Locking an object that is no longer actually the bucket's current head protects nothing — the re-check detects that stale situation and retries from scratch (`tabAt` again, fresh), rather than proceeding under a false assumption about what's actually being protected.

---

# 36. Phase 9 — get() Without Any Locking At All

```java
public V get(Object key) {
    int hash = spread(key.hashCode());
    Node<K, V>[] tab = table;                  // volatile read of the field — §20
    Node<K, V> node = tabAt(tab, (tab.length - 1) & hash); // volatile read of the array slot — §21's accessor
    for (; node != null; node = node.next) {   // volatile read of `next` each step — §32
        if (node.hash == hash && node.key.equals(key)) return node.val; // volatile read of `val` — §32
    }
    return null;
}
```

Not one `synchronized` block, not one CAS, not one `Lock` — and yet this is **fully correct** under arbitrary concurrent `put`/`remove`/resize activity from other threads. §37 explains precisely why that's true rather than a lucky accident.

---

# 37. Why get() Can Be Lock-Free: Reasoning About Visibility

`get()`'s correctness rests entirely on **every field it reads being `volatile`** (`table` itself, §20; each array slot via the CAS-capable accessor, §21; `Node.val`/`Node.next`, §32) — each read is guaranteed, by the Java Memory Model (§18's rule 3, applied to volatile reads/writes specifically), to observe the most recently *completed* write to that same field from any thread, no matter which thread wrote it or when. `get()` never needs to *prevent* a concurrent write from happening (which is what a lock is for) — it only needs to see a **consistent, fully-formed** version of whatever it reads, and `volatile` alone guarantees exactly that for individual field reads. What `get()` explicitly does **not** get is a guarantee of seeing the *very latest* write if one is racing concurrently — it might see the bucket's state from just before or just after a concurrent `put()` completes, but never a **torn**, half-written, or otherwise corrupted node. That specific, narrower guarantee — "always consistent, not necessarily the newest" — is precisely what §53's "weakly consistent" iteration terminology also describes, and it's sufficient for a hash map's `get()` to be correct without ever blocking.

---

# 38. Why a Long Linked-Chain Bucket Is a Denial-of-Service Risk

Under a poor-quality (or **adversarially crafted** — a real, documented attack against naive hash maps accepting untrusted keys, such as HTTP request parameters) hash function, many keys can collide into the *same* bucket, degrading that bucket's `get`/`put` from O(1) toward O(n) as its chain grows. Worse, under §34's per-bucket locking, a very long chain means the `synchronized` block held while traversing it is held for **longer**, directly increasing contention on that one bucket specifically — a pathological bucket doesn't just slow down its own lookups, it becomes a lock-contention hotspot for everything hashing there.

---

# 39. Treeification Threshold and Untreeification Threshold

Real `ConcurrentHashMap` (and this guide's version) converts a bucket's linked list into a **red-black tree** once it exceeds `TREEIFY_THRESHOLD` (8) entries **and** the overall table has at least `MIN_TREEIFY_CAPACITY` (64) buckets — bounding worst-case lookup within that one bucket to O(log n) instead of O(n). The second condition matters: if the *table itself* is still small, a bucket with 8+ entries more likely indicates the table simply hasn't grown enough yet (§11) rather than a genuinely pathological hash distribution — in that case, **resizing the whole table** (which naturally redistributes entries into more buckets) is preferred over treeifying one bucket. Symmetrically, `UNTREEIFY_THRESHOLD` (6) converts a tree bin back into a plain list if enough entries are removed — a tree's fixed overhead per node isn't worth paying once the collision problem it was solving no longer exists.

---

# 40. Phase 10 — Converting a Bin to a Red-Black Tree

```java
// bucketlocked/TreeNode.java — extends the linked-list Node with tree structure
public class TreeNode<K, V> extends Node<K, V> {
    TreeNode<K, V> parent, left, right;
    boolean red;
    TreeNode(int hash, K key, V val, Node<K, V> next) { super(hash, key, val, next); }
}
```

```java
// Triggered from putIntoExistingBucket (§34) once a bucket's chain length crosses TREEIFY_THRESHOLD
private void treeifyBin(Node<K, V>[] tab, int idx) {
    if (tab.length < MIN_TREEIFY_CAPACITY) { resize(); return; } // §39's second condition — grow instead, if the table itself is still small
    Node<K, V> head = tabAt(tab, idx);
    synchronized (head) {                                        // same locking discipline as §34 — one bucket at a time
        TreeBin<K, V> treeBin = TreeBin.buildFrom(head);         // walks the linked list, inserts each node into a new red-black tree
        setTabAt(tab, idx, treeBin);                              // the bucket's head is now a TreeBin, not a plain Node
    }
}
```

The tree is ordered by hash code (with a tie-breaking rule for equal hashes, since a red-black tree needs a total order, and raw hash equality doesn't guarantee key equality) — `get()`/`put()` on a treeified bucket walk down the tree by comparing hash values instead of scanning a list linearly.

---

# 41. TreeNode Locking: A Reader-Writer Lock Per Tree Bin

A single `synchronized` per operation (§34's model) is a poor fit for a tree bin specifically: **tree rebalancing** (rotations after an insert/delete, standard red-black tree maintenance) can temporarily leave the tree in a state that's unsafe for a concurrent reader to traverse, but the *vast majority* of tree-bin operations are reads that don't need to block each other at all. `TreeBin` therefore wraps its tree in a lightweight **reader-writer lock built from a few `volatile` fields and CAS**, not `java.util.concurrent.locks.ReentrantReadWriteLock` (too heavyweight for this purpose): readers proceed lock-free unless a writer is actively rebalancing, in which case they fall back to a plain linked-list-style traversal of the tree bin's nodes (still correct, just temporarily not benefiting from the tree's O(log n) shape) rather than blocking outright. This is a genuinely advanced piece of the real implementation — worth knowing this refinement exists and *why* (reads should almost never block, even during a rare rebalance), without needing to reproduce its full bit-flag-based lock state machine to understand the core lesson: **the locking strategy for a data structure should match how that specific structure is actually used**, exactly as §34's per-node lock and §41's reader-biased lock are two different answers to that same question, applied to two different bucket shapes.

---

# 42. Why Resizing Is the Hardest Part of a Concurrent Hash Map

§12's single-threaded `resize()` assumes nothing else touches the table while it runs. Under real concurrent load, a resize must tolerate — correctly — **other threads calling `get`, `put`, and `remove` on the table while the resize is still in progress**, and ideally should let **multiple threads help perform the resize simultaneously**, rather than one thread doing all the work while every other thread simply blocks and waits.

---

# 43. The ForwardingNode: A Marker for "This Bucket Has Moved"

The mechanism that makes concurrent resizing possible: once a bucket's contents have been split and copied into the new (larger) table, its slot in the **old** table is replaced with a special sentinel node — a `ForwardingNode` — whose only job is to redirect any thread that still finds it to the new table instead.

```java
// bucketlocked/ForwardingNode.java
public class ForwardingNode<K, V> extends Node<K, V> {
    final Node<K, V>[] newTable;
    ForwardingNode(Node<K, V>[] newTable) {
        super(MOVED, null, null, null); // MOVED is a reserved, impossible-for-a-real-key hash value, e.g. -1
        this.newTable = newTable;
    }
}
```

```java
// get() (§36), extended to follow a ForwardingNode transparently
public V get(Object key) {
    int hash = spread(key.hashCode());
    Node<K, V>[] tab = table;
    Node<K, V> node = tabAt(tab, (tab.length - 1) & hash);
    while (node != null) {
        if (node instanceof ForwardingNode<K, V> fwd) {
            tab = fwd.newTable;                             // follow the redirect — transparent to the caller
            node = tabAt(tab, (tab.length - 1) & hash);
            continue;
        }
        if (node.hash == hash && node.key.equals(key)) return node.val;
        node = node.next;
    }
    return null;
}
```

A `get()` that lands on a `ForwardingNode` simply follows it to the new table and keeps looking — the resize is **entirely invisible** to a reader beyond this one redirect check, which is why §36's lock-free read path survives a concurrent resize without any special-casing beyond recognizing this one sentinel type.

---

# 44. Phase 11 — Splitting a Bucket During Resize

Each old bucket's entries split into exactly **two** destination buckets in the new (double-sized) table — determined by a single additional bit of the hash that the old table's smaller mask didn't consider:

```java
// One old bucket's entries always split into newTab[idx] and newTab[idx + oldCapacity] — never anywhere else
private void splitBucket(Node<K, V>[] oldTab, Node<K, V>[] newTab, int idx) {
    Node<K, V> head = tabAt(oldTab, idx);
    synchronized (head) {                                    // same per-bucket lock discipline as every other mutation
        Node<K, V> lowHead = null, lowTail = null;            // entries staying at the SAME index in the new table
        Node<K, V> highHead = null, highTail = null;          // entries moving to index + oldCapacity
        for (Node<K, V> node = head; node != null; node = node.next) {
            if ((node.hash & oldTab.length) == 0) {           // the new, higher bit of the hash is 0 -> stays "low"
                if (lowTail == null) lowHead = node; else lowTail.next = node;
                lowTail = node;
            } else {                                          // the new bit is 1 -> moves "high"
                if (highTail == null) highHead = node; else highTail.next = node;
                highTail = node;
            }
        }
        if (lowTail != null) lowTail.next = null;
        if (highTail != null) highTail.next = null;
        setTabAt(newTab, idx, lowHead);
        setTabAt(newTab, idx + oldTab.length, highHead);
        setTabAt(oldTab, idx, new ForwardingNode<>(newTab));   // §43 — old slot now redirects here
    }
}
```

Splitting into exactly two destinations (never scattering entries across many new buckets) is why resizing can be done **bucket by bucket, independently** — each old bucket's split is a self-contained unit of work, needing no coordination with any other old bucket's split, which is exactly what makes §45's cooperative multi-threaded resize possible.

---

# 45. Cooperative Resizing: Helping Threads Assist the Transfer

Rather than one thread performing every bucket's split while every other thread blocks, real `ConcurrentHashMap` lets **any thread that calls `put`/`get` during a resize notice one is in progress and help**, by claiming a range of not-yet-transferred old buckets and running §44's `splitBucket` on them. A shared, atomically-updated "next bucket index to claim" counter is what lets multiple threads divide the transfer work without duplicating effort or missing a bucket.

---

# 46. Phase 12 — Implementing transfer()

```java
private final AtomicInteger transferIndex = new AtomicInteger();

private void transfer(Node<K, V>[] oldTab, Node<K, V>[] newTab) {
    int stride = 16; // each "claim" covers a batch of buckets, not just one — reduces contention on transferIndex itself
    int bound;
    int start;
    while ((start = transferIndex.getAndAdd(-stride)) > 0) { // claim a range by DECREMENTING a shared counter atomically
        bound = Math.max(0, start - stride);
        for (int i = start - 1; i >= bound; i--) {
            if (i < oldTab.length && !(tabAt(oldTab, i) instanceof ForwardingNode)) {
                splitBucket(oldTab, newTab, i); // §44 — this thread does the actual work for these buckets
            }
        }
    }
    // Once transferIndex reaches (or passes) zero, every bucket has been claimed by SOME thread — resize is complete
    // for the buckets this thread was responsible for; a final CAS on a shared "size control" field (omitted here for
    // brevity) determines which single thread is responsible for publishing the new table as the map's `table` field.
}
```

Batching claims into a `stride` of 16 buckets at a time, rather than CAS-ing the counter once per bucket, is a direct application of §22's lesson: reduce how often multiple threads contend for the *same* piece of shared, frequently-updated state (`transferIndex` here), even though each individual claim is itself still perfectly correct at a stride of 1 — batching is a throughput optimization layered on top of correctness, not a correctness requirement itself.

---

# 47. What Happens to a get()/put() That Arrives Mid-Resize

- **A `get()`** either finds the key in the old table's not-yet-transferred bucket (fine — nothing has changed yet), or finds a `ForwardingNode` and follows it into the new table (§43 — transparent), or finds the key already migrated into the new table directly if the caller's own `table` field read happened to observe the swap. Every one of these outcomes is correct; none of them require the `get()` to know a resize is happening at all.
- **A `put()`** that lands on an untransferred bucket proceeds exactly as §33–§34 describe, against the *old* table — perfectly safe, since that bucket hasn't been touched by the resize yet. A `put()` that lands on a bucket already replaced by a `ForwardingNode` recognizes it (the same check `get()` does) and **helps transfer** that portion of the resize (§45) before retrying its own operation against the new table — turning a would-be blocked thread into a useful contributor to finishing the resize faster.

---

# 48. Why a Single volatile Counter Doesn't Scale

The obvious approach to tracking size — `private volatile long count; count++` on every insert — fails for exactly the reason §19 already named: `count++` is not one operation, it's read-increment-write, and `volatile` guarantees *visibility* of each individual read/write, not *atomicity* of the whole sequence. Two threads incrementing concurrently can both read the same value before either writes back, losing an increment — the counter itself becomes a second, independent source of the exact §13 bug this entire guide has been eliminating from the map's actual data.

A `synchronized` increment or an `AtomicLong.incrementAndGet()` *would* be correct — but at high thread counts, they become a **new, single bottleneck** of their own: every one of potentially thousands of concurrent `put`/`remove` calls across every bucket in the whole table ends up serialized on this one shared counter, even though the actual data mutations they're counting are, per §31–§37, wonderfully parallel.

---

# 49. Striped Counters: The CounterCell Technique (LongAdder's Idea)

The fix — the same idea behind `java.util.concurrent.atomic.LongAdder` — is to stop pretending the count needs to live in one place at all. Instead, maintain an **array of counter cells**, and let each thread update whichever cell it can claim with the *least* contention (typically, one derived from a per-thread hash, so different threads tend to land on different cells); the true count is only ever computed on demand, as the **sum across all cells**, which is needed far less often than individual increments happen.

```java
// bucketlocked/CounterCell.java — padded to avoid false sharing between adjacent cells on the same cache line
public class CounterCell {
    volatile long value;
    CounterCell(long initial) { this.value = initial; }
}
```

```java
private volatile long baseCount;              // the fast path: used when there's no contention at all
private volatile CounterCell[] counterCells;   // allocated lazily, only once contention on baseCount is actually detected
```

---

# 50. Phase 13 — Implementing size() via Striped Counters

```java
private void addCount(long delta) {
    // Fast path: try to CAS the shared base counter directly — cheap, and correct when uncontended
    if (BASECOUNT.compareAndSet(this, baseCount, baseCount + delta)) return;

    // Fallback: contention detected on baseCount -> use (or lazily create) a per-thread-ish counter cell instead
    CounterCell[] cells = counterCells;
    if (cells == null) cells = initializeCounterCells();
    int cellIndex = ThreadLocalRandom.current().nextInt() & (cells.length - 1); // spread threads across cells
    CounterCell cell = cells[cellIndex];
    long current = cell.value;
    if (!CELL_VALUE.compareAndSet(cell, current, current + delta)) {
        addCount(delta); // this specific cell was contended too — retry (a different random cell, likely, next time)
    }
}

public long mappingCount() {
    long sum = baseCount;
    CounterCell[] cells = counterCells;
    if (cells != null) {
        for (CounterCell cell : cells) sum += cell.value; // only summed on demand — size() is the RARE operation here
    }
    return sum; // may be very slightly stale under concurrent modification — see §53's "weakly consistent" discussion
}
```

The core trade-off, stated plainly: **writes become cheap and scalable** (each thread usually only contends with a small fraction of other threads, on whichever cell it happens to land on) **at the cost of reads becoming more expensive** (summing potentially many cells instead of reading one field) — exactly the right trade for a counter that's incremented on every single write but read comparatively rarely, which is the actual access pattern a hash map's size counter has.

---

# 51. compute(), computeIfAbsent(), merge(): Atomicity Without a Global Lock

A naive "check then act" implementation of `computeIfAbsent` — `if (map.get(key) == null) map.put(key, compute(key))` — has a **race condition** even on a fully thread-safe map: two threads can both call `get`, both see `null`, and both proceed to compute and insert, with one insert silently overwriting the other (or, worse, the "compute" function running twice when the contract implies it should run at most once per key). A **correct** `computeIfAbsent` must perform the check-and-insert as a single atomic step relative to other operations on *that specific key* — which, conveniently, is exactly what §34's per-bucket `synchronized` block already provides, if the computation is done *inside* that same locked region rather than before it.

---

# 52. Phase 14 — Implementing computeIfAbsent() Correctly

```java
public V computeIfAbsent(K key, Function<K, V> mappingFunction) {
    int hash = spread(key.hashCode());
    Node<K, V>[] tab = table;
    int idx = (tab.length - 1) & hash;

    Node<K, V> first = tabAt(tab, idx);
    if (first == null) {
        // Even the "empty bucket" fast path (§33) can't just CAS blindly here — the mapping function
        // must run at most once, so a temporary placeholder-and-lock approach (or, as real ConcurrentHashMap
        // does, a reservation node) is needed even in the "no contention yet" case. Simplified here to a lock:
        synchronized (tab) { /* re-check under a coarser lock scoped to this rare path only, then compute + CAS in */ }
    }
    return computeIfAbsentInExistingBucket(tab, idx, hash, key, mappingFunction); // synchronized(first), same as §34
}

private V computeIfAbsentInExistingBucket(Node<K, V>[] tab, int idx, int hash, K key, Function<K, V> mappingFunction) {
    Node<K, V> first = tabAt(tab, idx);
    synchronized (first) {
        for (Node<K, V> node = first; node != null; node = node.next) {
            if (node.hash == hash && node.key.equals(key)) return node.val; // already present — mappingFunction NEVER called
        }
        V computed = mappingFunction.apply(key);       // computed WHILE HOLDING the bucket's lock
        if (computed != null) {
            appendNode(tab, idx, new Node<>(hash, key, computed, null));
            addCount(1);
        }
        return computed;
    }
}
```

Calling `mappingFunction.apply(key)` **while still holding the bucket's lock** is the detail that makes this correct rather than merely convenient — it guarantees no other thread can concurrently insert a value for the same key between "checked, not present" and "inserted the computed value," which is exactly the race a naive get-then-put implementation cannot close. The real cost of this correctness: `mappingFunction` runs while holding a lock, so a slow or blocking mapping function directly extends how long *that one bucket* is unavailable to other threads — a documented, deliberate constraint on what kind of function is safe to pass to `computeIfAbsent` in production code.

---

# 53. Weakly Consistent Iterators: What Guarantee You Actually Get

An iterator over `MiniConcurrentMap` makes a specific, narrower promise than a single-threaded map's iterator: it will **never** throw an exception due to concurrent modification, it is **guaranteed** to reflect the state of the map at some valid point in time for each element it returns (never a torn/corrupted node, per §37's reasoning), but it offers **no guarantee** about whether a concurrent insert or removal that happens *during* the iteration will or won't be reflected in what the iterator subsequently returns — it might see it, might not, and either outcome is considered correct.

---

# 54. Why ConcurrentHashMap Never Throws ConcurrentModificationException

A single-threaded `HashMap`'s iterator uses a `modCount` field, incremented on every structural change, checked on every `next()` call — if it doesn't match what the iterator expects, `ConcurrentModificationException` is thrown (a **fail-fast** design, appropriate when concurrent modification during iteration always indicates a bug). A concurrent map rejects that design entirely: concurrent modification during iteration is an **expected, routine** occurrence, not a bug — throwing an exception for it would make the map effectively unusable under the exact concurrent access pattern it exists to support. §53's weaker guarantee is the deliberate trade that makes iteration usable at all on a structure other threads are actively mutating.

---

# 55. Phase 15 — Implementing a Weakly Consistent Iterator

```java
public class WeaklyConsistentIterator implements Iterator<Map.Entry<K, V>> {
    private Node<K, V>[] currentTable;
    private int bucketIndex;
    private Node<K, V> nextNode;

    WeaklyConsistentIterator() {
        this.currentTable = table; // a single volatile read at construction time — §20 — this snapshot may go stale, and that's fine (§53)
        advanceToNextNonEmptyBucket();
    }

    @Override
    public boolean hasNext() { return nextNode != null; }

    @Override
    public Map.Entry<K, V> next() {
        Node<K, V> node = nextNode;
        if (node instanceof ForwardingNode) { // the table this iterator started with was mid-resize — follow it, §43
            currentTable = ((ForwardingNode<K, V>) node).newTable;
            advanceToNextNonEmptyBucket();
            return next(); // retry against the new table
        }
        Map.Entry<K, V> entry = Map.entry(node.key, node.val); // a SNAPSHOT of val — a later concurrent update won't retroactively change it
        nextNode = node.next;
        if (nextNode == null) { bucketIndex++; advanceToNextNonEmptyBucket(); }
        return entry;
    }

    private void advanceToNextNonEmptyBucket() {
        while (nextNode == null && bucketIndex < currentTable.length) {
            nextNode = tabAt(currentTable, bucketIndex);
            if (nextNode == null) bucketIndex++;
        }
    }
}
```

No `modCount`, no exception path at all — every read (`tabAt`, `node.next`, `node.val`) is a plain volatile read exactly like `get()`'s (§36–§37), which is precisely why this iterator needs no special coordination with concurrent writers: it's built from the same lock-free-but-consistent building blocks the rest of the read path already relies on.

---

# 56. Full Worked Example: Concurrent Put/Get/Resize Traced Step by Step

Tracing two threads, `T1` calling `put("a", 1)` and `T2` calling `put("b", 2)`, where both keys happen to hash into the *same* bucket, while the table happens to be resizing concurrently:

```text
T1: put("a", 1)                              T2: put("b", 2)                            Resize thread: transfer()
  hash("a") -> bucket 5                        hash("b") -> bucket 5 (collision with "a")
  tabAt(tab, 5) -> ForwardingNode!              tabAt(tab, 5) -> ForwardingNode!            splitBucket(oldTab, newTab, 5)
  follow to newTab (§43)                        follow to newTab (§43)                        synchronized(oldHead) { ... }
  tabAt(newTab, 5') -> null                                                                    setTabAt(oldTab, 5, FWD)
  CAS insert "a" into newTab[5'] (§33)                                                          <- races with T1/T2 above;
    SUCCEEDS (uncontended CAS)                  tabAt(newTab, 5') -> Node("a") now!              whichever happens first,
                                                 CAS insert "b" fails (bucket no longer empty)    T1/T2 correctly either
                                                 falls back to synchronized(first) (§34)          see the FWD or the
                                                 appends "b" after "a" in newTab[5']              already-split new bucket
  addCount(1) — striped counter (§50)           addCount(1) — likely a DIFFERENT cell (§49)
```

Every one of the mechanisms built in this guide appears in this one trace: hash routing (§7–§8), the `ForwardingNode` redirect (§43), CAS for the uncontended case (§33), `synchronized`-on-first-node for the contended case (§34), and striped counting (§49–§50) — and at no point do T1 and T2 ever block each other on a lock that spans more than the one bucket they both happen to collide on.

---

# 57. Final Architecture

```text
                         MiniConcurrentMap
                                 |
              +------------------+------------------+
              v                                     v
    volatile Node<K,V>[] table (§20)        AtomicInteger transferIndex (§46)
              |                                     |
    +---------+---------+                  cooperative resize (§42-§47)
    v                   v                           |
Empty bucket:      Non-empty bucket:                v
CAS insert (§33)   synchronized(first) (§34)   ForwardingNode (§43)
    |                   |                      splitBucket (§44)
    v                   v
  (bucket grows into a red-black tree past TREEIFY_THRESHOLD, §38-§41)
              |
              v
    get() / iterator — all lock-free, volatile-read-based (§36-§37, §53-§55)
              |
              v
    size tracking via striped CounterCells (§48-§50)
    compute/computeIfAbsent — atomic via the SAME per-bucket lock (§51-§52)
```

---

# 58. Common Mistakes

- **Mistake 1 — Assuming a "concurrent" collection makes every compound operation atomic.** `if (!map.containsKey(k)) map.put(k, v)` is still a race on `MiniConcurrentMap`/`ConcurrentHashMap` alike — only `computeIfAbsent` (§51–§52), `putIfAbsent`, and similar single-call atomic methods actually close that gap.
- **Mistake 2 — Forgetting `volatile` on `Node.val`/`Node.next` (§32).** Produces a map that "usually" works in testing (single-threaded or lightly-loaded runs rarely expose it) and intermittently returns stale or missing values under real concurrent load — one of the hardest classes of bug to reproduce on demand.
- **Mistake 3 — Calling a slow or blocking function inside `computeIfAbsent` (§52).** Directly extends how long that bucket's lock is held, potentially stalling every other thread that happens to hash to the same bucket.
- **Mistake 4 — Locking more than one bucket at a time.** Reopens the deadlock risk §24 warns about — every design in this guide (§25's segments, §34's per-bucket lock, §44's per-bucket split) deliberately acquires exactly one lock per operation.
- **Mistake 5 — A single shared counter for `size()` (§48).** Correct, but turns a highly concurrent map's every single write into a contention point on one field — exactly the bottleneck §49's striped counters exist to remove.
- **Mistake 6 — Treating `mappingCount()`/`size()` as exact under concurrent modification.** It's a best-effort snapshot (§50) — correct code should not assume `size()` called twice in a row, with concurrent writers active, returns consistent or monotonically increasing values.
- **Mistake 7 — Skipping the `first != tabAt(tab, idx)` re-check after acquiring a bucket lock (§35).** Locks an object that may no longer be the bucket's actual current head, protecting nothing.

---

# 59. Testing Strategy for Concurrent Code

Concurrency bugs are inherently probabilistic — a test that passes ten times in a row has not proven correctness, only failed to get unlucky. A meaningfully useful test suite leans on this asymmetry rather than fighting it:

| Test | What it checks | How |
|---|---|---|
| High-thread-count insert stress test (§15's pattern) | No lost updates: final size equals total unique keys inserted | Many threads inserting disjoint key ranges concurrently, assert `size()` afterward, repeat many times in CI to raise the odds of catching a rare interleaving |
| Concurrent resize under load | Every key inserted before, during, and after a forced resize is retrievable afterward | Insert enough keys to trigger several resizes while concurrently reading/writing from other threads throughout |
| `computeIfAbsent` single-invocation guarantee | The mapping function runs **exactly once** per key, even under concurrent calls for the same key from many threads | Many threads call `computeIfAbsent` for the *same* key simultaneously; assert the mapping function's invocation counter is exactly 1 |
| Deadlock absence | The test suite itself completes within a timeout, never hangs | Wrap concurrent stress tests in a bounded `awaitTermination`; a hang is a `Deadlock` finding, not a slow pass |
| Iterator weak consistency (§53) | Iteration completes without exception despite concurrent modification; every element returned was genuinely present at some point | Iterate while another thread concurrently inserts/removes; assert no exception, and that returned entries are a subset of "keys present at some point during the run" |
| `jcstress`-style low-level checks (advanced) | Actual memory-visibility bugs the JIT/CPU reordering could otherwise expose only rarely | The OpenJDK `jcstress` harness is purpose-built for exactly this — a worthwhile mention for taking `volatile`/CAS correctness claims beyond "ran fine in my stress test" |

---

# 60. Benchmarking: Throughput Under Contention

```java
// bench/ThroughputBenchmark.java — comparing all four designs built in this guide under identical load
public static void main(String[] args) throws InterruptedException {
    for (var mapSupplier : List.of(
            (Supplier<Object>) SimpleHashMap::new,             // §6-§12, wrapped in one external lock for a fair baseline
            (Supplier<Object>) () -> new SegmentedConcurrentMap<>(16),
            (Supplier<Object>) MiniConcurrentMap::new)) {
        // ... run N threads, each doing a mix of get/put/remove for a fixed duration, measure total ops/sec ...
    }
}
```

What the results should show, and why: the fully-locked baseline's throughput **plateaus** almost immediately as thread count increases past 1–2 (§17's ceiling); the segmented map's throughput scales **up to roughly its segment count**, then plateaus (§28); the per-bucket-locked map continues scaling substantially further, because its effective concurrency unit (a bucket) vastly outnumbers any reasonable segment count and grows with the table itself. Seeing this as an actual measured curve, not just an assertion, is what makes §30 and §31's design arguments concrete rather than theoretical.

---

# 61. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Strategy** (implicit) | Choosing CAS vs `synchronized` per operation, based on whether the target bucket is empty | §33-§34 |
| **Sentinel/Null Object** | `ForwardingNode` — a special `Node` subtype that redirects rather than storing a real entry | §43 |
| **Composite** | `TreeBin` wrapping a red-black tree behind the same `Node`-shaped interface a plain bucket head presents | §40-§41 |
| **Striping** | Splitting one hot piece of shared state (a lock in §25, a counter in §49) into many independently-contended pieces | §25-§30, §48-§50 |
| **Template Method** (implicit) | Every mutating operation follows the same shape: hash, locate bucket, try CAS, fall back to lock-and-traverse | §33-§34, §52 |

Striping (segments *and* counters) is the single idea doing the most conceptual work in this entire guide — reducing contention by increasing the number of independently-lockable/independently-updatable units is the mechanism behind both the map's core write path and its size-tracking, applied to two different kinds of shared state.

---

# 62. Progressive Interview Question Set

**Level 1 — The core problem**
1. Trace exactly what corrupted state results from two threads calling `put()` on a plain `HashMap` at the same time, for the same bucket.
2. Why is `Hashtable`'s "synchronize everything" approach correct but throughput-limited?

**Level 2 — Locking granularity**
3. Explain lock striping, and why the number of segments is a fixed ceiling on write concurrency.
4. Why does Java 8's `ConcurrentHashMap` lock individual buckets instead of segments?

**Level 3 — Memory visibility**
5. What does `volatile` guarantee, and — precisely — what does it *not* guarantee?
6. Why can `get()` be implemented with zero locking, and what specifically makes that safe?

**Level 4 — CAS and lock-free programming**
7. What does a CAS instruction do, atomically, at the hardware level?
8. Why does `ConcurrentHashMap` use CAS only for empty-bucket insertion and fall back to `synchronized` otherwise, rather than using CAS everywhere?

**Level 5 — Resizing**
9. Walk through what a `ForwardingNode` is for, and what a `get()` does when it encounters one.
10. Why can a bucket's contents always be split into exactly two destination buckets during a resize, never more?
11. What does it mean for a resize to be "cooperative," and why is that better than one thread doing all the work?

**Level 6 — Correctness at the edges**
12. Why is `if (!map.containsKey(k)) map.put(k, v)` still a race condition on a "thread-safe" map, and how does `computeIfAbsent` close it?
13. What does "weakly consistent" mean for an iterator, and why doesn't `ConcurrentHashMap` throw `ConcurrentModificationException`?
14. Why doesn't a single `volatile` counter correctly and efficiently track `size()` under high write concurrency?

**Final challenge:** A workload's keys are highly skewed — 90% of all `put`/`get` calls target the same 10 keys out of millions. Explain exactly which part of this guide's design (bucket locking? treeification? striped counters?) helps least under this specific access pattern, and sketch what additional mechanism you would add to address the actual bottleneck this workload creates.

---

# 63. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| `putIfAbsent`/`remove(key, value)`/`replace(key, oldValue, newValue)` | The full atomic compound-operation surface real `ConcurrentHashMap` exposes | Each follows §52's "compute inside the bucket lock" pattern with a different check/action |
| Bulk operations (`forEach`, `search`, `reduce` with parallelism thresholds) | Data-parallel operations across the whole map, splitting work by table region | Built on top of §55's iteration machinery, parallelized across bucket ranges the way §46's resize already splits work |
| `ConcurrentHashMap.KeySetView`-style live views | A `Set<K>` view backed by the same map, without copying | A thin adapter over the existing iterator (§55) and `remove`/`put` paths |
| NUMA-aware striping | Reducing cross-socket cache-line traffic on very large multi-socket machines | Extending §49's striping to prefer counter cells local to the calling thread's NUMA node |
| A lock-free skip-list variant (`ConcurrentSkipListMap`-style) | Ordered iteration and range queries, which a hash-based structure cannot offer | An entirely different structure, not an extension of this one — worth knowing it exists as the answer to "I need concurrency *and* sorted order" |
| Off-heap / memory-mapped storage | Handling a map larger than JVM heap comfortably allows | Replacing `Node` references with offsets into an off-heap buffer, with CAS operating on that buffer directly |

---

# 64. Final Takeaway

Every step in this guide is the same move, applied at a different scale: **take a piece of shared, mutable state that's currently protected by one lock (or one plain field), and ask whether it can instead be split into many independently-protected pieces** — a table into segments (§25), a segment into individual buckets (§31), a size counter into striped cells (§49). The correctness argument at each step never changes (mutual exclusion for writers on colliding operations, visibility for readers via `volatile`) — only the **granularity** at which that argument is applied gets progressively finer, buying more real concurrency at each step, right up to the point (a single bucket, a single counter cell) where finer granularity stops paying for itself. Real `java.util.concurrent.ConcurrentHashMap` is, underneath its considerable production-hardened detail, exactly this same sequence of decisions — and having derived each one yourself is what turns "it's a concurrent hash map, it's just thread-safe" into an answer you can actually defend, line by line, in an interview or a design review.

