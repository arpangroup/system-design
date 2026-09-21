# Optimistic vs Pessimistic Locking — A Practical, Step-by-Step Guide With a Custom Java Implementation

> **Goal:** Understand *why* concurrent database updates need locking at all — not as an abstract rule, but by watching a real bug happen — then fix it two different ways (pessimistic and optimistic locking), understand the real tradeoffs between them using an actual production-shaped use case (a flash-sale inventory system), and finally **build a working optimistic lock completely from scratch in Java**, without relying on JPA/Hibernate's `@Version` to do it for you.
>
> This guide is deliberately use-case-first: every concept is introduced because a concrete bug demands it, not as a definition to memorize before you've seen the problem it solves.

---

# 1. What We Are Building

Using one running example — **QuickCart**, an e-commerce flash-sale flow where a limited quantity of a product goes on sale and thousands of users try to buy it in the same few seconds — we will:

- Reproduce a real, concrete **lost-update bug**: overselling stock because two purchases race each other.
- Fix it with **pessimistic locking** (`SELECT ... FOR UPDATE`) — understand exactly what it locks, why that's correct, and what it costs in throughput.
- Fix the *same* bug a second way with **optimistic locking** (a version column) — understand why it's correct without ever blocking a reader, and what it costs when conflicts are frequent.
- **Build our own optimistic lock from scratch** — a reusable `OptimisticLockTemplate` in plain Java/JDBC, with a proper retry-with-backoff loop, with no framework annotation doing the work invisibly.
- Walk through **real future optimizations** a team would reach for once either approach hits its own limits at scale: reducing lock scope, sharding a hot row, a single-writer queue, caching, advisory locks, and distributed locking.

```text
Two users, same product, last unit in stock, same second:
  User A: GET stock -> 1        User B: GET stock -> 1
  User A: stock - 1 = 0         User B: stock - 1 = 0
  User A: SAVE stock = 0        User B: SAVE stock = 0     <- BOTH think they succeeded. Stock is now -1 sold, 0 left.
                                                                Overselling. This guide exists to stop this, twice over.
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Reproduce, explain, and fix a lost-update race condition on a shared database row — not just recite "use locking."
- Explain precisely what a `SELECT ... FOR UPDATE` row lock does, what it blocks, and for how long.
- Explain precisely how a version column detects a conflict *after the fact*, without ever taking a database lock.
- Choose correctly between pessimistic and optimistic locking for a given workload's contention level and read/write ratio — with a concrete argument, not a rule of thumb.
- Implement a working, reusable optimistic-locking retry loop in plain Java, understanding exactly why each part of it exists.
- Name the next three or four things a team would actually do once either approach becomes the bottleneck at real scale.

---

# 3. Why This Matters (Interview Motivation)

> **"A flash sale has 500 units of a product and 50,000 users trying to buy it in the same 10 seconds. Design the purchase flow so stock is never oversold. Compare a pessimistic-locking and an optimistic-locking implementation, explain the tradeoffs, and explain what you'd do differently if contention got 100x worse."**

This is one of the most common **e-commerce/backend system design interview questions** because it's a compact way to test several things that are individually common but rarely combined into one clean scenario:

- Whether you actually understand *why* a race condition happens at the row level, not just that "concurrency is hard."
- Whether you know the real mechanics of database locking (`FOR UPDATE`, isolation levels) rather than only the word "lock."
- Whether you can reason about a genuine tradeoff (blocking vs. retrying) instead of treating one approach as universally "the modern one."
- Whether you can talk credibly about what happens *after* the interview's toy scale — sharding, queues, distributed locks — which is exactly what separates "I can implement a pattern" from "I understand when a pattern stops working."

---

# 4. The Real Practical Use Case: An E-Commerce Flash-Sale Inventory System

**QuickCart** sells a limited-edition product: 500 units, on sale at exactly 12:00:00. Every purchase must:

1. Check that stock remains (`stock > 0`).
2. Decrement stock by the quantity purchased.
3. Record the order.

The correctness requirement is absolute: **the number of successful purchases must never exceed 500**, regardless of how many requests arrive in the same instant. This is the concrete, non-negotiable constraint every section of this guide is in service of — not an abstract "make it thread-safe," but "never let the 501st purchase succeed."

---

# 5. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | Matches this guide's plain-JDBC and `java.util.concurrent` code. |
| Persistence | Plain JDBC first (§16, §24, §31–§32), then the JPA/Hibernate equivalents shown for context (§18, §26) | Seeing the raw SQL a lock strategy issues is the point — an ORM annotation should be understood as a shorthand for exactly this SQL, not a separate mechanism. |
| Database | Any standard relational database supporting row-level locking and `SELECT ... FOR UPDATE` (Postgres, MySQL/InnoDB, or [TinyDB](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) from the companion guide, which already implements row-level locking, §35 there) | The mechanics in this guide are standard SQL, not vendor-specific. |
| Testing | JUnit 5 + a concurrent stress harness (the same shape as the [ConcurrentHashMap guide's](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) §59) | A locking bug is probabilistic — only a genuinely concurrent test can catch it. |

---

# 6. Project Structure

```text
quickcart-locking/
├── src/main/java/com/example/quickcart/
│   ├── model/
│   │   └── Product.java
│   ├── naive/
│   │   └── NaivePurchaseService.java          // §8 — deliberately broken, to reproduce the bug
│   ├── pessimistic/
│   │   ├── PessimisticPurchaseService.java     // §16
│   │   └── PessimisticJpaPurchaseService.java  // §18
│   ├── optimistic/
│   │   ├── OptimisticPurchaseService.java      // §24
│   │   ├── OptimisticJpaPurchaseService.java   // §26
│   │   └── OptimisticLockTemplate.java         // §33 — the reusable, hand-built retry loop
│   ├── lock/
│   │   ├── VersionedEntity.java                // §31
│   │   ├── OptimisticLockException.java
│   │   └── InMemoryOptimisticLock.java         // §34
│   └── QuickCartApplication.java
└── src/test/java/com/example/quickcart/
    ├── NaivePurchaseRaceConditionTest.java      // §48 — proves the bug exists
    ├── PessimisticPurchaseConcurrencyTest.java
    └── OptimisticPurchaseConcurrencyTest.java
```

---

# 7. Phase 1 — Modeling the Product and Stock

```java
// model/Product.java
public class Product {
    private final String id;
    private String name;
    private int stock;
    private long version; // unused until §23 — present now so later sections don't need to change this class

    // constructor / getters / setters omitted for brevity
}
```

```sql
CREATE TABLE product (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    stock INT,
    version BIGINT DEFAULT 0
);
INSERT INTO product (id, name, stock) VALUES ('sku-42', 'Limited Sneaker', 500);
```

---

# 8. The Naive "Read, Check, Write" Purchase Flow

The obviously-correct-looking, single-threaded-correct implementation:

```java
// naive/NaivePurchaseService.java
public class NaivePurchaseService {
    private final DataSource dataSource;

    public boolean purchase(String productId, int quantity) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            int currentStock = readStock(conn, productId);              // 1. READ
            if (currentStock < quantity) return false;                  // 2. CHECK
            writeStock(conn, productId, currentStock - quantity);       // 3. WRITE
            recordOrder(conn, productId, quantity);
            return true;
        }
    }

    private int readStock(Connection conn, String productId) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement("SELECT stock FROM product WHERE id = ?")) {
            ps.setString(1, productId);
            try (ResultSet rs = ps.executeQuery()) { rs.next(); return rs.getInt("stock"); }
        }
    }

    private void writeStock(Connection conn, String productId, int newStock) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement("UPDATE product SET stock = ? WHERE id = ?")) {
            ps.setInt(1, newStock);
            ps.setString(2, productId);
            ps.executeUpdate();
        }
    }
    // recordOrder(...) omitted for brevity
}
```

Read this code in isolation, on one thread, and it is completely correct. The bug it contains only exists in the gap **between** the read and the write — invisible to any single-threaded test, and invisible to code review unless you're specifically looking for it.

---

# 9. Tracing Two Concurrent Purchases With No Locking at All

Two users, User A and User B, both attempt to buy the **last unit** of a product (`stock = 1`) in the same instant:

```text
Time  User A (buying 1 unit)              User B (buying 1 unit)
----  -----------------------------       -----------------------------
t0    readStock() -> 1
t1                                        readStock() -> 1        <- both read the SAME stale value: 1
t2    check: 1 >= 1, proceed
t3                                        check: 1 >= 1, proceed  <- both pass the check
t4    writeStock(1 - 1 = 0)
t5                                        writeStock(1 - 1 = 0)   <- both write 0, both think they succeeded
t6    recordOrder() -- SUCCESS
t7                                        recordOrder() -- SUCCESS
```

Both purchases report success. Both orders are recorded. **The database ends the sequence with `stock = 0`** — correct-looking on its own — but **two units were sold from a stock of one.** Nothing in this trace threw an exception, logged an error, or behaved in any way that looks obviously wrong from either request's own point of view — that's precisely what makes this class of bug dangerous: it fails silently, at the business level, not at the code level.

---

# 10. The Lost Update Anomaly, Made Concrete

What just happened has a name — a **lost update**: two transactions each read the same value, each compute a new value based on that read, and the second write silently overwrites the first's *intent* even though it doesn't literally overwrite the same bytes with the same value here (both happened to compute the same new stock, `0`, which is what makes this variant especially sneaky — even inspecting the final row shows nothing anomalous). The actual lost information isn't a database value at all — it's the fact that **two decrements should have happened, and the database's final state is only consistent with one.**

---

# 11. Why This Isn't a Rare Edge Case: Real-World Contention (Flash Sales, Ticket Drops)

This isn't a contrived timing coincidence — it's the *expected*, common case under real contention:

- A flash sale drives thousands of requests at the **exact same product row** within a few hundred milliseconds of each other by design (that's what "flash sale" means).
- Concert ticket drops, limited sneaker releases, and cryptocurrency/NFT mints all share this exact shape: many concurrent writers, one contended row, a hard business constraint on the total.
- Under real load, §9's race isn't a once-in-a-million interleaving — with a popular flash sale and a database read/write round trip taking even a few milliseconds, **dozens or hundreds** of requests can land inside that same read-to-write gap simultaneously.

---

# 12. What a Database Lock Actually Is (Row Lock vs Table Lock)

A **lock**, at the database level, is a mechanism that makes a second transaction **wait** (or, for optimistic locking, **fail after the fact**) before it can read or write something a first transaction is already working with — closing exactly the gap §9 traced. Two granularities matter here:

| | What it blocks | Cost |
|---|---|---|
| **Table lock** | Every row in the table, for every other transaction | Simple, but serializes unrelated purchases of *different* products — vastly more blocking than the problem requires |
| **Row lock** | Only the specific row(s) a transaction has touched | Two purchases of *different* products never block each other at all — exactly the granularity §13 onward uses |

Every design in this guide operates at **row-level** granularity — locking (or version-checking) only the single `product` row being purchased, never the whole table — for the same reason the [ConcurrentHashMap guide](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) moved from table-wide locking to per-bucket locking: the smallest correct unit of contention is the one worth paying for.

---

# 13. Two Fundamentally Different Strategies: Pessimistic vs Optimistic

| | Pessimistic locking | Optimistic locking |
|---|---|---|
| Core assumption | Conflicts are common enough that preventing them upfront is worth the cost | Conflicts are rare enough that it's cheaper to detect them after the fact than to prevent them |
| Mechanism | Take a real database lock **before** reading, so no other transaction can read-for-write the same row until you're done | Read freely, no lock taken; **on write**, verify nothing else changed the row since you read it (§23's version check) |
| What a conflicting second transaction experiences | **Blocks** (waits) until the first transaction finishes | **Fails** with a detectable conflict, immediately, and must retry |
| Best suited for | High contention, or operations expensive enough that wasted retry work would be worse than waiting | Low-to-moderate contention, where blocking would waste more time than an occasional retry costs |

Both strategies fix §9's exact bug — they simply move the fix to a different point in the timeline: pessimistic locking prevents the second read from ever seeing stale data; optimistic locking lets the second read happen, but refuses to let its stale-based write actually commit.

---

# 14. The Core Idea: Lock First, Then Read

Pessimistic locking's entire idea in one sentence: **acquire an exclusive lock on the row as part of the read itself**, so that no other transaction can even *read* that row (for the purpose of writing it) until the lock is released. "Pessimistic" names the assumption driving this: you're pessimistic that a conflict is likely enough that it's worth paying an upfront cost (making everyone else wait) to prevent it outright, rather than gambling on a rare conflict and cleaning up after the fact.

---

# 15. SELECT ... FOR UPDATE, Mechanically

```sql
SELECT stock FROM product WHERE id = 'sku-42' FOR UPDATE;
```

This single clause changes everything about what happens next: the database takes an **exclusive row lock** on the matching row(s) for the remainder of the current transaction. Any *other* transaction that also tries to `SELECT ... FOR UPDATE` (or, on most databases, to `UPDATE`) that same row **blocks** — it doesn't error, it doesn't see stale data, it simply **waits** until the first transaction commits or rolls back and releases the lock. A plain `SELECT` (without `FOR UPDATE`) from another transaction is typically still allowed to proceed unblocked (it just sees a pre-commit or post-commit snapshot depending on isolation level) — it's specifically the "I intend to write based on this read" declaration that `FOR UPDATE` makes explicit, and that's exactly what earns the lock.

---

# 16. Phase 2 — Implementing Pessimistic Locking With Plain JDBC

```java
// pessimistic/PessimisticPurchaseService.java
public class PessimisticPurchaseService {
    private final DataSource dataSource;

    public boolean purchase(String productId, int quantity) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try {
                int currentStock = readStockForUpdate(conn, productId); // acquires the row lock HERE
                if (currentStock < quantity) {
                    conn.rollback();
                    return false;
                }
                writeStock(conn, productId, currentStock - quantity);
                recordOrder(conn, productId, quantity);
                conn.commit();                                           // releases the row lock
                return true;
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            }
        }
    }

    private int readStockForUpdate(Connection conn, String productId) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement("SELECT stock FROM product WHERE id = ? FOR UPDATE")) {
            ps.setString(1, productId);
            try (ResultSet rs = ps.executeQuery()) { rs.next(); return rs.getInt("stock"); }
        }
    }
    // writeStock(...) / recordOrder(...) — same as §8, now running INSIDE the locked transaction
}
```

The only structural change from §8's broken version: the read is now `FOR UPDATE`, and the whole read-check-write sequence runs inside one transaction that holds the lock until `commit()`/`rollback()`. That's the entire fix — no retry logic, no version field, because the lock makes the race in §9 **impossible to enter in the first place**, not merely detectable after the fact.

---

# 17. Tracing the Fixed Flow With Pessimistic Locking

```text
Time  User A (buying 1 unit)                    User B (buying 1 unit)
----  -----------------------------------       -----------------------------------
t0    SELECT ... FOR UPDATE -> 1, LOCK ACQUIRED
t1                                               SELECT ... FOR UPDATE -> BLOCKS (waits for A's lock)
t2    check: 1 >= 1, proceed
t3    UPDATE stock = 0
t4    COMMIT -- lock released
t5                                               (unblocks) SELECT ... FOR UPDATE -> 0
t6                                               check: 0 >= 1 -> FALSE
t7                                               ROLLBACK, return "out of stock"
```

User B's request isn't wrong or racy anymore — it's simply **delayed** until it can see the true, post-A stock value, at which point the check correctly and honestly rejects it. Exactly one purchase succeeds, exactly as the business rule in §4 requires.

---

# 18. Pessimistic Locking in JPA/Hibernate (@Lock(PESSIMISTIC_WRITE))

```java
// pessimistic/PessimisticJpaPurchaseService.java
@Service
public class PessimisticJpaPurchaseService {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public boolean purchase(String productId, int quantity) {
        Product product = em.find(Product.class, productId, LockModeType.PESSIMISTIC_WRITE); // issues SELECT ... FOR UPDATE
        if (product.getStock() < quantity) return false;
        product.setStock(product.getStock() - quantity);
        // no explicit save() needed — JPA's dirty-checking flushes the UPDATE at transaction commit
        return true;
    }
}
```

`LockModeType.PESSIMISTIC_WRITE` is Hibernate's name for exactly §15's `FOR UPDATE` clause — this annotation-driven version and §16's raw-JDBC version issue the **same SQL** and provide the **same guarantee**; the ORM is a convenience layer over the mechanism, not a different mechanism.

---

# 19. The Cost of Pessimistic Locking: Blocking and Throughput

§17's trace shows the correctness win directly, and its cost just as directly: **User B's request takes measurably longer** than it would have without locking — it spent real wall-clock time blocked, waiting for User A's entire transaction (read, check, write, commit) to finish. Under the flash-sale scenario from §4/§11, with potentially hundreds of concurrent requests for the same row, this queues **every** request behind whichever one currently holds the lock — throughput on that specific row is capped at "how fast one transaction at a time can complete," no matter how many CPU cores or database connections are available. This is the exact same throughput ceiling the [ConcurrentHashMap guide's §17](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) describes for a single coarse-grained lock — here it's just scoped to one contended row instead of an entire in-memory structure.

---

# 20. Deadlocks Under Pessimistic Locking and How to Avoid Them

If a single purchase ever needs to lock **more than one row** (e.g., decrementing stock for two different products in one order), a new failure mode appears: transaction A locks product 1 then waits for product 2, while transaction B locks product 2 then waits for product 1 — **deadlock**, exactly the shape [the ConcurrentHashMap guide's §24](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) describes for in-memory locks, now at the database level. Real databases detect this automatically and abort one transaction with a deadlock error rather than hanging forever — but the *application* fix is the same one that guide already established: **always acquire multi-row locks in a fixed, consistent order** (e.g., always lock the lower product ID first) across every code path that might lock more than one row, so the "opposite order" deadlock shape can never occur in the first place.

---

# 21. Lock Timeouts: Failing Fast Instead of Waiting Forever

Left unbounded, a blocked transaction under heavy contention (§19) can wait an unacceptably long time — or, if the lock-holding transaction itself hangs (a slow downstream call, a stuck connection), indefinitely. A **lock timeout** — `SET LOCAL lock_timeout = '3s'` on Postgres, or an equivalent setting per database — bounds how long a transaction will wait for a row lock before giving up and raising an error, which the application then surfaces as a clear, fast "try again" response instead of leaving a user's request hanging with no feedback. Choosing this timeout is a genuine tradeoff: too short, and legitimate requests fail under ordinary contention; too long, and a stuck request can tie up a database connection (and everything queued behind it) for an unacceptable duration.

---

# 22. The Core Idea: Assume No Conflict, Verify Before Committing

Optimistic locking flips §14's assumption entirely: **read without taking any lock at all**, do the work, and only at the moment of writing, check whether anything else changed the row since it was read. "Optimistic" names this assumption directly — you're betting that, most of the time, nothing else touched this row in the meantime, so paying an upfront locking cost on every read would be wasted effort; you only pay a cost (a retry) on the comparatively rare occasions the bet is wrong.

---

# 23. The Version Column: How Optimistic Locking Detects a Conflict

The mechanism needs some way to answer "has this row changed since I read it?" without a lock — a **version column**, incremented on every successful update, is the standard answer:

```sql
ALTER TABLE product ADD COLUMN version BIGINT DEFAULT 0; -- already present in §7's schema
```

The write is then made **conditional on the version being unchanged**:

```sql
UPDATE product SET stock = ?, version = version + 1 WHERE id = ? AND version = ?;
```

If another transaction updated the row (and bumped its version) between this transaction's read and its write, this `UPDATE` matches **zero rows** — the `version = ?` predicate no longer matches anything — and the driver reports "0 rows affected." That single fact, checked by the application immediately after the update, is the entire conflict-detection mechanism: no lock, no blocking, just a conditional write that silently fails to match when the assumption behind it turns out to be wrong.

---

# 24. Phase 3 — Implementing Optimistic Locking With Plain JDBC

```java
// optimistic/OptimisticPurchaseService.java
public class OptimisticPurchaseService {
    private final DataSource dataSource;

    /** Returns true on success, false if stock was insufficient, throws OptimisticLockException on a detected conflict. */
    public boolean purchase(String productId, int quantity) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            ProductSnapshot snapshot = readProduct(conn, productId); // plain SELECT — no lock, no FOR UPDATE
            if (snapshot.stock() < quantity) return false;

            int rowsUpdated = tryUpdate(conn, productId, snapshot.stock() - quantity, snapshot.version());
            if (rowsUpdated == 0) {
                throw new OptimisticLockException(productId); // §27 — the caller decides whether/how to retry
            }
            recordOrder(conn, productId, quantity);
            return true;
        }
    }

    private ProductSnapshot readProduct(Connection conn, String productId) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement("SELECT stock, version FROM product WHERE id = ?")) {
            ps.setString(1, productId);
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return new ProductSnapshot(rs.getInt("stock"), rs.getLong("version"));
            }
        }
    }

    private int tryUpdate(Connection conn, String productId, int newStock, long expectedVersion) throws SQLException {
        String sql = "UPDATE product SET stock = ?, version = version + 1 WHERE id = ? AND version = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, newStock);
            ps.setString(2, productId);
            ps.setLong(3, expectedVersion);
            return ps.executeUpdate(); // returns the number of rows matched/updated — THE conflict signal
        }
    }
    // record ProductSnapshot(int stock, long version) {}
}
```

`tryUpdate`'s return value — the JDBC-reported row count — is doing all the work §15's `FOR UPDATE` clause did, just at a completely different point in the sequence: **after** the write attempt, not **before** the read.

---

# 25. Tracing the Fixed Flow With Optimistic Locking

```text
Time  User A (buying 1 unit)                          User B (buying 1 unit)
----  ---------------------------------------          ---------------------------------------
t0    SELECT stock, version -> (1, v5)      <- no lock
t1                                                       SELECT stock, version -> (1, v5)  <- same snapshot, still no lock
t2    check: 1 >= 1, proceed
t3                                                       check: 1 >= 1, proceed
t4    UPDATE ... WHERE version = 5 -> 1 row updated (v6)
t5                                                       UPDATE ... WHERE version = 5 -> 0 ROWS updated (version is now 6, not 5!)
t6    recordOrder() -- SUCCESS
t7                                                       throw OptimisticLockException -- caller retries (§27-§28)
```

Both users read the identical stale snapshot, exactly as in §9's original bug — the difference is entirely in what happens next: User B's write is **conditionally rejected by the database itself**, not silently accepted, because the version it's conditioned on no longer matches reality. No blocking occurred anywhere in this trace — User B's request ran at full speed right up until the exact instant its assumption was proven wrong.

---

# 26. Optimistic Locking in JPA/Hibernate (@Version)

```java
// model/Product.java (JPA-annotated variant)
@Entity
public class Product {
    @Id private String id;
    private int stock;

    @Version                    // Hibernate automatically includes this in every UPDATE's WHERE clause, and increments it
    private long version;
}

@Service
public class OptimisticJpaPurchaseService {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public boolean purchase(String productId, int quantity) {
        Product product = em.find(Product.class, productId); // a PLAIN find — no lock hint at all
        if (product.getStock() < quantity) return false;
        product.setStock(product.getStock() - quantity);
        // at flush/commit time, Hibernate issues exactly §23's conditional UPDATE, and throws
        // jakarta.persistence.OptimisticLockException if it matches zero rows
        return true;
    }
}
```

`@Version` is Hibernate doing, automatically, exactly what §24 wrote by hand: reading the version alongside the entity, and including `AND version = ?` (plus `version = version + 1`) on every generated `UPDATE` for that entity — the annotation is a code-generation convenience over the identical SQL-level mechanism.

---

# 27. Handling a Detected Conflict: Retry, Not Just Fail

Throwing `OptimisticLockException` (§24, §26) the moment a conflict is detected is **correct but incomplete** — from the *business* perspective, User B might well still have succeeded if their purchase had simply re-read the current stock and tried again (the item might not have sold out at all; they just happened to compute their update against a now-stale snapshot). Treating every optimistic-lock conflict as a hard failure throws away legitimate business outcomes that a **retry** would have correctly captured — which is exactly why real optimistic-locking code is almost never "catch the exception, fail the request" in isolation; it's "catch the exception, retry the whole read-check-write sequence against the now-current data."

---

# 28. Phase 4 — A Retry Loop With Exponential Backoff

```java
public boolean purchaseWithRetry(String productId, int quantity, int maxAttempts) throws SQLException {
    long backoffMillis = 10;
    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            return purchase(productId, quantity); // §24 — may throw OptimisticLockException
        } catch (OptimisticLockException conflict) {
            if (attempt == maxAttempts) throw conflict; // give up after enough genuine attempts — see §29
            sleepWithJitter(backoffMillis);
            backoffMillis = Math.min(backoffMillis * 2, 500); // exponential backoff, capped
        }
    }
    throw new IllegalStateException("unreachable");
}

private void sleepWithJitter(long baseMillis) {
    long jitter = ThreadLocalRandom.current().nextLong(baseMillis / 2);
    try { Thread.sleep(baseMillis + jitter); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}
```

**Exponential backoff** (doubling the wait between attempts, up to a cap) exists to prevent a specific failure mode: if many concurrent requests all conflict and all retry *immediately and simultaneously*, they simply collide again on the next attempt, and the next — a self-inflicted contention storm. Backing off, with **jitter** (a small random component) added specifically so that requests which started retrying at the same moment don't stay synchronized retry-for-retry, spreads retries out over time and gives the contention a real chance to actually dissipate between attempts.

---

# 29. The Cost of Optimistic Locking: Wasted Work Under High Contention

§25's trace shows the win: User B never blocked. But User B's entire read-check-compute sequence, up to the failed `UPDATE`, was **wasted work** — CPU cycles and a database round trip spent computing a result the database then discarded. Under *low* contention, this cost is negligible (conflicts, and therefore wasted attempts, are rare by definition). Under **very high** contention — exactly the flash-sale scenario from §4, where hundreds of requests target the identical row in the identical instant — optimistic locking can, counterintuitively, perform **worse** than pessimistic locking: many requests repeatedly read, compute, fail, retry, read again, and fail again, burning far more total database work than a queue of waiting-but-eventually-succeeding pessimistic transactions would have. This exact tension — "optimistic locking assumes low contention, but a flash sale is the highest-contention scenario imaginable" — is precisely what §37–§39 confront directly, and part of why the single-writer-queue optimization in Part 6 exists at all.

---

# 30. Designing a Reusable OptimisticLock Abstraction

§24 and §28 hard-code the retry loop and the version-check logic specifically to `Product`/stock. A **reusable** optimistic-locking mechanism — the actual goal of this part of the guide — needs to separate three concerns that are currently tangled together: **what a versioned entity looks like** (§31), **how to attempt a conditional update generically, for any such entity** (§32), and **how to retry on conflict, generically, regardless of what business operation is being retried** (§33).

---

# 31. Phase 5 — The Versioned Entity Base Class

```java
// lock/VersionedEntity.java
public abstract class VersionedEntity {
    private final String id;
    private long version;

    protected VersionedEntity(String id, long version) {
        this.id = id;
        this.version = version;
    }

    public String getId() { return id; }
    public long getVersion() { return version; }
    void setVersion(long version) { this.version = version; } // package-private — only OptimisticLockTemplate advances this
}
```

```java
// model/Product.java, updated to extend it
public class Product extends VersionedEntity {
    private int stock;
    public Product(String id, int stock, long version) { super(id, version); this.stock = stock; }
    public int getStock() { return stock; }
    public void setStock(int stock) { this.stock = stock; }
}
```

Any entity that needs optimistic locking — not just `Product` — extends `VersionedEntity` and immediately gets access to the generic machinery §32–§33 build against this one shared shape, instead of every entity type needing its own hand-written version-check `UPDATE` statement.

---

# 32. Phase 6 — A Generic compareAndUpdate() Using Raw JDBC

```java
// lock/OptimisticLockDao.java
public class OptimisticLockDao<T extends VersionedEntity> {
    private final DataSource dataSource;
    private final String tableName;
    private final RowMapper<T> rowMapper;               // reads a ResultSet row into a T
    private final BiConsumer<PreparedStatement, T> columnBinder; // binds T's mutable fields, in column order, for the UPDATE

    public T load(String id) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement("SELECT * FROM " + tableName + " WHERE id = ?")) {
            ps.setString(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (!rs.next()) return null;
                return rowMapper.map(rs);
            }
        }
    }

    /** The one generic method every entity's optimistic update goes through — mirrors §23's UPDATE ... WHERE version = ? */
    public boolean compareAndUpdate(T entity, String updateColumnsSql) throws SQLException {
        String sql = "UPDATE " + tableName + " SET " + updateColumnsSql + ", version = version + 1 "
                   + "WHERE id = ? AND version = ?";
        try (Connection conn = dataSource.getConnection(); PreparedStatement ps = conn.prepareStatement(sql)) {
            columnBinder.accept(ps, entity);              // binds the business columns (e.g. stock = ?)
            int paramCount = countPlaceholders(updateColumnsSql);
            ps.setString(paramCount + 1, entity.getId());
            ps.setLong(paramCount + 2, entity.getVersion());
            int updated = ps.executeUpdate();
            if (updated == 1) entity.setVersion(entity.getVersion() + 1); // keep the in-memory object consistent, too
            return updated == 1;
        }
    }
    // countPlaceholders(...) omitted for brevity — counts '?' occurrences in the caller-supplied SQL fragment
}
```

This is the direct Java analogue of `AtomicReference.compareAndSet(expected, newValue)` (or the CAS primitive from [the ConcurrentHashMap guide's §21](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>)), operating against a database row instead of an in-memory field: "update this row **only if** its version still matches what I last read," reporting success or failure rather than blocking either way.

---

# 33. Phase 7 — Wrapping It in a Reusable OptimisticLockTemplate (The Retry Loop, Generalized)

```java
// lock/OptimisticLockTemplate.java
public class OptimisticLockTemplate {
    private final int maxAttempts;
    private final long initialBackoffMillis;
    private final long maxBackoffMillis;

    public OptimisticLockTemplate(int maxAttempts, long initialBackoffMillis, long maxBackoffMillis) {
        this.maxAttempts = maxAttempts;
        this.initialBackoffMillis = initialBackoffMillis;
        this.maxBackoffMillis = maxBackoffMillis;
    }

    /**
     * Runs `operation` up to maxAttempts times. `operation` should load the current entity itself EACH TIME
     * it's called (never reuse a stale read across attempts) and return true if its compareAndUpdate succeeded.
     */
    public boolean executeWithRetry(BooleanSupplier operation) throws InterruptedException {
        long backoff = initialBackoffMillis;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            if (operation.getAsBoolean()) return true;               // the business operation's own compareAndUpdate succeeded
            if (attempt == maxAttempts) return false;                // exhausted retries — caller decides what "false" means
            long jitter = ThreadLocalRandom.current().nextLong(backoff / 2 + 1);
            Thread.sleep(backoff + jitter);
            backoff = Math.min(backoff * 2, maxBackoffMillis);
        }
        return false;
    }
}
```

```java
// Using the template for the flash-sale purchase, with NO purchase-specific retry logic written by hand
OptimisticLockTemplate template = new OptimisticLockTemplate(5, 10, 500);
boolean success = template.executeWithRetry(() -> {
    try {
        Product product = dao.load(productId);                        // fresh read EVERY attempt — critical, see below
        if (product.getStock() < quantity) return true;                // "succeeded" at correctly rejecting — not a conflict
        product.setStock(product.getStock() - quantity);
        return dao.compareAndUpdate(product, "stock = ?");
    } catch (SQLException e) { throw new UncheckedIOException(new IOException(e)); }
});
```

The comment "fresh read EVERY attempt" is load-bearing, not a style note: retrying with the **same** stale `Product` object read before the first attempt would just fail identically forever — a retry is only meaningful if it re-reads current data before recomputing, exactly mirroring why the DDA-style "re-check under lock" pattern in earlier guides re-reads rather than trusting a value captured before a wait.

---

# 34. Phase 8 — An In-Memory Optimistic Lock (No Database At All)

The same idea, entirely in memory, useful for protecting a cached or in-process shared object without a database round trip at all:

```java
// lock/InMemoryOptimisticLock.java
public class InMemoryOptimisticLock<T> {
    private final AtomicReference<Versioned<T>> ref;

    public record Versioned<T>(T value, long version) { }

    public InMemoryOptimisticLock(T initial) { this.ref = new AtomicReference<>(new Versioned<>(initial, 0)); }

    public Versioned<T> read() { return ref.get(); }

    /** Returns true if the update was applied; false if `expected`'s version was stale (someone else updated first). */
    public boolean compareAndUpdate(Versioned<T> expected, T newValue) {
        Versioned<T> updated = new Versioned<>(newValue, expected.version() + 1);
        return ref.compareAndSet(expected, updated); // an object-identity CAS on the WRAPPER, not the raw value
    }
}
```

```java
InMemoryOptimisticLock<Integer> stockLock = new InMemoryOptimisticLock<>(500);
boolean success = template.executeWithRetry(() -> {
    var current = stockLock.read();
    if (current.value() < quantity) return true;
    return stockLock.compareAndUpdate(current, current.value() - quantity);
});
```

---

# 35. Why the In-Memory Version Is Structurally the Same as AtomicStampedReference

§34's `InMemoryOptimisticLock` is, functionally, a hand-rolled version of `java.util.concurrent.atomic.AtomicStampedReference` — a reference plus a version "stamp," updated together atomically via CAS, specifically to solve the **ABA problem** a plain `AtomicReference<T>` can't: if a value changes from A to B and back to A between a thread's read and its CAS attempt, a bare reference-based CAS would incorrectly succeed (the reference looks unchanged), while a stamped/versioned CAS correctly fails, because the version number kept climbing even though the value returned to its starting point. This is the exact same guarantee §23's database version column provides over a hypothetical "just compare the stock value" check — a value coincidentally returning to what you last saw is not the same as nothing having happened in between, and only a monotonically advancing version can tell the two apart.

---

# 36. Phase 9 — Putting It Together: The Flash-Sale Purchase Service

```java
@Service
public class FlashSalePurchaseService {
    private final OptimisticLockDao<Product> productDao;
    private final OptimisticLockTemplate lockTemplate;

    public PurchaseResult purchase(String productId, int quantity) {
        try {
            boolean succeeded = lockTemplate.executeWithRetry(() -> {
                try {
                    Product product = productDao.load(productId);
                    if (product == null || product.getStock() < quantity) return true; // not a conflict — a real "no stock" outcome
                    product.setStock(product.getStock() - quantity);
                    return productDao.compareAndUpdate(product, "stock = ?");
                } catch (SQLException e) { throw new RuntimeException(e); }
            });
            if (!succeeded) return PurchaseResult.CONFLICT_EXHAUSTED_RETRIES; // §29's worst case, surfaced honestly
            Product finalState = productDao.load(productId);
            return finalState.getStock() >= 0 ? PurchaseResult.SUCCESS : PurchaseResult.OUT_OF_STOCK;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return PurchaseResult.INTERRUPTED;
        }
    }
}
```

Every piece built in §30–§35 appears here: `VersionedEntity` (§31) gives `Product` its version field for free, `OptimisticLockDao` (§32) issues the version-conditioned `UPDATE`, and `OptimisticLockTemplate` (§33) supplies the retry-with-backoff loop — none of it purchase-specific, all of it reusable for the next entity that needs the same guarantee.

---

# 37. Pessimistic vs Optimistic — A Direct Comparison Table

| | Pessimistic locking | Optimistic locking |
|---|---|---|
| When a conflict is detected | Never happens — the second transaction simply waits its turn | After the fact, at write time — the conditional `UPDATE` matches zero rows |
| Cost under low contention | Pays lock-acquisition overhead even when no conflict would have occurred | Near-zero overhead — no lock ever taken |
| Cost under high contention | Requests queue and wait, but each one that gets through succeeds on its first real attempt | Many requests read, compute, and fail repeatedly — real, measurable wasted work (§29) |
| Failure mode under extreme load | Long queues, possible timeouts (§21), possible deadlocks if multiple rows are involved (§20) | Retry storms unless backoff/jitter (§28) is implemented correctly |
| Implementation complexity | Simpler application code — no retry loop needed, the database does the waiting | Requires a retry loop, and a decision about what to do when retries are exhausted (§36) |
| Read-heavy workloads | Reads that don't use `FOR UPDATE` are unaffected — pessimistic locking only impacts the write path | No impact on reads either way — an even better fit here, since reads never risk a false conflict |

---

# 38. A Decision Framework: Contention Level, Read/Write Ratio, Criticality

Three questions, asked in this order, cover the large majority of real cases:

1. **How contended is this specific row/resource, realistically?** A user's own profile row is touched by essentially one writer (that user) — vanishingly low contention, and optimistic locking's near-zero overhead wins easily. A flash-sale product row during the sale's opening seconds is about as contended as a single row can get — pessimistic locking's "queue and succeed" behavior may genuinely outperform optimistic locking's "fail and retry, repeatedly" behavior here, exactly per §29.
2. **How expensive is the work between read and write?** If the computation between reading and writing is slow (an external API call, a complex calculation), pessimistic locking holds the lock for that entire duration, blocking everyone else for longer — optimistic locking's "only pay a cost on an actual conflict" model tolerates a slow read-compute-write path far better, since nothing else is forced to wait on it.
3. **What does a failure actually cost the business?** An optimistic-locking conflict that exhausts its retries (§29, §36) has to resolve to *something* user-visible — a failure, a queue position, a "please try again." If that outcome is unacceptable for the operation (a bank transfer that must eventually succeed, not just retry-then-give-up), pessimistic locking's "everyone eventually gets their turn" guarantee is often the safer default, even at some throughput cost.

---

# 39. Hybrid Strategies: Optimistic First, Pessimistic Fallback

A workload can't always be cleanly classified as "always high contention" or "always low contention" — a product's contention level is near-zero for 99% of its life and spikes enormously for the ten minutes it's on flash sale. One practical hybrid: use optimistic locking by default (§24, §33), but **detect** sustained high conflict rates for a specific row (e.g., via the retry count metrics a well-instrumented `OptimisticLockTemplate` would emit) and **switch that specific row** to pessimistic locking for the duration of the spike — getting optimistic locking's low overhead during normal operation and pessimistic locking's queue-and-succeed behavior exactly when contention would otherwise cause the retry-storm cost §29 describes. This is a genuinely more complex system to build and operate than either pure strategy, and is worth reaching for only once measurement (§49) shows it's actually needed — not as a default starting design.

---

# 40. Optimization 1 — Reducing Lock Scope (Row-Level Instead of Table-Level)

Already the default throughout this guide (§12) — worth restating as the first thing to verify before reaching for anything more exotic: confirm that whatever ORM or raw SQL is in use is genuinely taking a **row**-level lock (or a row-level version check) and not, through a misconfigured query or a missing index on the `WHERE` clause's column, silently escalating to a broader lock. A `SELECT ... FOR UPDATE` without an index on the filtered column can force a full table scan under a lock in some databases — the single most common accidental cause of "pessimistic locking is blocking way more than I expected."

---

# 41. Optimization 2 — Reducing Contention by Sharding the Hot Row

If one product row is fundamentally too hot for either strategy to handle well at some scale (the flash sale's true worst case), the next lever is to stop treating it as **one** row at all: split the 500-unit stock count across, say, 10 **shard rows** of 50 units each, route each incoming purchase to a shard (randomly, or round-robin), and only fall back to checking a different shard if the chosen one is exhausted. This turns one heavily-contended row into ten moderately-contended ones — directly analogous to [lock striping in the ConcurrentHashMap guide's §25](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>), applied to database rows instead of in-memory segments, at the cost of slightly more complex logic to reconcile "which shard has stock left."

---

# 42. Optimization 3 — A Single-Writer Queue Per Resource (Serialization Without Locks)

An entirely different strategy sidesteps database-level locking altogether: route every write for a given hot resource through a **single, dedicated in-process queue/worker** (or a Kafka partition keyed by product ID, processed by one consumer), so that writes for that resource are **naturally serialized** by construction — never actually concurrent in the first place, so there's no race to detect or prevent. This trades "handle concurrency at write time" for "eliminate concurrency by design," at the cost of the queue itself becoming the new bottleneck and a more complex operational model (monitoring queue depth/lag becomes as important as monitoring lock wait times used to be) — a legitimate and common answer for the highest-contention resources in a system, and the shape of solution real large-scale flash-sale systems (ticketing platforms, sneaker-drop platforms) often reach for once neither locking strategy alone is enough.

---

# 43. Optimization 4 — Caching Reads to Reduce Load on the Locked Path

Neither locking strategy helps with a different, related cost: if every purchase attempt first checks "is this even still in stock" via a full database read, that read traffic alone can overwhelm the database during a spike, independent of the locking strategy protecting the actual write. Caching a **coarse, slightly-stale** "is stock likely available" signal (an in-memory or Redis-backed approximate counter, refreshed frequently but not on every single request) lets the vast majority of "sorry, sold out" responses be served without ever touching the row that pessimistic/optimistic locking is protecting — reserving the actual locked/versioned path only for requests that pass this cheap, approximate pre-check.

---

# 44. Optimization 5 — Database-Specific Advisory Locks

Postgres's `pg_advisory_lock(key)` (and equivalents on other databases) provides an **application-defined** lock, keyed by an arbitrary integer/string, that has no connection to any actual table row at all. This is useful precisely when the thing needing serialization **isn't** a single database row — e.g., "only one instance of this scheduled job should run at a time across a fleet of servers," a coordination problem locking a specific table row doesn't naturally model, but which an advisory lock keyed by `"nightly-report-job"` handles directly, with the same acquire/block/release semantics as §15's row lock, applied to an arbitrary logical resource instead.

---

# 45. Optimization 6 — Distributed Locking Beyond One Database (Redis/Zookeeper)

Once a system scales beyond a single database instance being the sole source of truth for a resource (a multi-region deployment, a resource coordinated across several independent services), locking mechanisms tied to one database's row-lock implementation stop being sufficient — a **distributed lock**, implemented against a shared coordination service (Redis with the Redlock algorithm, or ZooKeeper/etcd's native lock primitives), extends the same "acquire, hold, release" idea from §14–§21 across process and machine boundaries. This is a substantially harder problem than single-database locking — network partitions mean a lock holder can be presumed dead when it isn't, or vice versa — and is worth reaching for only once the architecture has genuinely outgrown "one database can serialize this," not as a default replacement for §16's `FOR UPDATE`.

---

# 46. Optimization 7 — Batching Updates to Reduce Lock Acquisitions

If many small updates to the same row arrive close together (many 1-unit purchases, rather than one large one), a further optimization is to **batch** them: accumulate a short window of pending decrements in memory (or in a queue, per §42), and apply them to the database as one combined `UPDATE ... SET stock = stock - ?` per batch instead of one lock-acquire-and-release cycle per individual purchase. This trades a small amount of added latency (waiting briefly to accumulate a batch) for a large reduction in the *number* of lock acquisitions or version-check round trips against the contended row — the same "batch to reduce contention on shared state" idea as [group commit in the TinyDB guide's §31](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>), applied to application-level writes instead of WAL `fsync` calls.

---

# 47. Common Mistakes

- **Mistake 1 — "Thread-safe" application code with no database-level protection.** A `synchronized` block or an in-memory lock around the *application's* call to the database does nothing to prevent two different application server instances (or two separate connections from the same instance) from racing at the database level — the lock has to exist where the contention actually is.
- **Mistake 2 — Retrying with a stale, previously-read object (§33's warning).** A retry that recomputes against the same data it already failed with will simply fail identically forever; every retry attempt must re-read current state first.
- **Mistake 3 — No backoff/jitter on optimistic retries (§28).** Synchronized, immediate retries under contention just recreate the same collision on the next attempt, for every conflicting request, simultaneously.
- **Mistake 4 — Holding a pessimistic lock across a slow external call.** Calling a third-party payment API *while* holding `SELECT ... FOR UPDATE`'s row lock blocks every other purchase attempt for the entire duration of that external call — a classic, severe throughput bug; validate/reserve first, call external services outside the lock's scope wherever the business logic allows it.
- **Mistake 5 — Choosing optimistic locking by default without checking the actual contention level (§38).** Optimistic locking's "usually better" reputation only holds under low-to-moderate contention — applying it unexamined to a genuinely hot, flash-sale-shaped row can perform worse than the pessimistic alternative it was chosen to avoid.
- **Mistake 6 — Forgetting the `version` column update in a hand-written `UPDATE`.** An `UPDATE` that changes business columns but forgets `version = version + 1` silently defeats optimistic locking for every subsequent writer — the version stops advancing, so future conflicts go undetected instead of correctly failing.
- **Mistake 7 — No maximum retry limit.** An optimistic retry loop with no cap (§28's `maxAttempts`) can, under sustained extreme contention, retry indefinitely rather than surfacing a clear, bounded failure back to the caller.

---

# 48. Testing Strategy: Proving the Fix With Concurrent Load

A single-threaded unit test **cannot** catch §9's bug, or prove §16/§24 fixed it — only genuine concurrent load can, matching the exact testing philosophy the [ConcurrentHashMap guide's §59](<Build Your Own ConcurrentHashMap From Scratch — Bucket-Level Locking and Multithreading Step-by-Step Guide.md>) already establishes:

```java
@Test
void naiveServiceOversellsUnderConcurrentLoad() throws Exception {
    setupProductWithStock(1); // exactly one unit available
    int concurrentBuyers = 50;
    ExecutorService pool = Executors.newFixedThreadPool(concurrentBuyers);
    CountDownLatch startGate = new CountDownLatch(1); // forces every thread to fire at the SAME instant
    AtomicInteger successCount = new AtomicInteger();

    for (int i = 0; i < concurrentBuyers; i++) {
        pool.submit(() -> {
            try {
                startGate.await();
                if (naivePurchaseService.purchase("sku-42", 1)) successCount.incrementAndGet();
            } catch (Exception ignored) { }
        });
    }
    startGate.countDown(); // release all 50 threads simultaneously
    pool.shutdown();
    pool.awaitTermination(10, TimeUnit.SECONDS);

    // With NaivePurchaseService: this FAILS, repeatedly, with successCount > 1 — proving the bug from §9-10
    // With PessimisticPurchaseService or OptimisticPurchaseService (with retry): this reliably PASSES
    assertEquals(1, successCount.get());
}
```

A `CountDownLatch` used as a **starting gate** — every thread waits at `startGate.await()` and is released simultaneously by one `countDown()` call — is the standard technique for maximizing the odds of exercising the exact race window §9 traced, rather than hoping thread scheduling happens to create the collision on its own.

---

# 49. Benchmarking Pessimistic vs Optimistic Under Different Contention Levels

Running §48's harness at increasing `concurrentBuyers` counts, against both `PessimisticPurchaseService` and `OptimisticPurchaseService`, and measuring total wall-clock time to complete all attempts, produces the concrete curve behind §37–§38's comparison table:

| Concurrent buyers | Pessimistic (queue-and-succeed) | Optimistic (retry-on-conflict) |
|---|---|---|
| Low (e.g. 5) | Slight overhead from lock acquisition on every request | Fastest — near-zero conflicts, near-zero retries |
| Moderate (e.g. 50) | Predictable, linear queueing delay | Still generally fast — occasional conflicts, occasional cheap retries |
| Very high (e.g. 5,000, all targeting one row) | Long queue, but still monotonic progress — every request eventually resolves | Retry storms possible without proper backoff (§28) — total work done can exceed the pessimistic case |

Measuring this yourself, rather than trusting the table, is the point — the actual crossover point where optimistic locking's retry cost starts to exceed pessimistic locking's queueing cost depends on your specific database, network latency, and transaction duration, and is exactly the kind of number §39's hybrid-strategy decision should be based on, not guessed at.

---

# 50. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Template Method** | `OptimisticLockTemplate.executeWithRetry` — a fixed retry/backoff skeleton, with the actual business operation supplied as a callback | §33 |
| **Strategy** | Choosing pessimistic vs optimistic (or a hybrid, §39) per resource/operation, behind a common `purchase(...)`-shaped interface | §37-§39 |
| **Optimistic Concurrency Control** (a named pattern in its own right) | The version-column compare-and-swap mechanism itself | §23-§24, §32 |
| **Compare-And-Swap / Atomic Reference** | `InMemoryOptimisticLock`, structurally identical to `AtomicStampedReference` | §34-§35 |
| **Sharding/Partitioning** | Splitting one hot row into several less-contended shard rows | §41 |

---

# 51. Full Worked Example End to End

```java
FlashSalePurchaseService service = new FlashSalePurchaseService(
    new OptimisticLockDao<>(dataSource, "product", Product::fromRow, Product::bindStock),
    new OptimisticLockTemplate(5, 10, 500)
);

// 500 concurrent requests, only 500 units of stock — the exact scenario from §4
List<Future<PurchaseResult>> results = IntStream.range(0, 5000)
    .mapToObj(i -> executor.submit(() -> service.purchase("sku-42", 1)))
    .toList();

long successCount = results.stream().map(Future::get).filter(r -> r == PurchaseResult.SUCCESS).count();
assert successCount == 500; // exactly the business constraint from §4 — never more, never fewer than available stock allows
```

5,000 requests, 500 units of stock, zero overselling — the version-column mechanism from §23, the generic DAO from §32, and the retry template from §33 combine to enforce exactly the constraint this entire guide set out to satisfy in §4, under real concurrent load, without a single explicit database lock ever being taken.

---

# 52. Final Architecture

```text
                       Purchase Request
                              |
                              v
                  FlashSalePurchaseService (§36)
                              |
              +---------------+---------------+
              v                               v
    Pessimistic path (§14-21)        Optimistic path (§22-29, §30-36)
    SELECT ... FOR UPDATE             OptimisticLockTemplate (§33)
    -> blocks conflicting readers       -> executeWithRetry(...)
    -> commit releases the lock              |
                                              v
                                    OptimisticLockDao (§32)
                                    compareAndUpdate(entity, ...)
                                    UPDATE ... WHERE version = ?
                                              |
                                    0 rows updated? -> conflict -> backoff (§28) -> retry
                                    1 row updated?  -> success, version incremented
              |                               |
              +---------------+---------------+
                              v
                    Database row (§7, §23)
                    id | stock | version

    Future optimizations layered on top, as contention grows (Part 6):
    sharding (§41) -> single-writer queue (§42) -> read caching (§43)
    -> advisory locks (§44) -> distributed locks (§45) -> batching (§46)
```

---

# 53. Progressive Interview Question Set

**Level 1 — The core bug**
1. Walk through, step by step, exactly how two concurrent purchases can both succeed against a stock of 1, with no code throwing an exception anywhere.
2. Why does this bug not show up in a single-threaded test, or even most manual testing?

**Level 2 — Pessimistic locking**
3. What exactly does `SELECT ... FOR UPDATE` do, and what does it *not* block?
4. How can pessimistic locking introduce a deadlock, and how do you prevent it?

**Level 3 — Optimistic locking**
5. How does a version column detect a conflict without ever taking a lock?
6. Why is "catch the conflict exception and fail the request" usually the wrong way to handle an optimistic-locking conflict?

**Level 4 — Choosing between them**
7. Give a concrete scenario where pessimistic locking would outperform optimistic locking, and explain why in terms of contention and wasted work.
8. Why might exponential backoff with jitter matter more under optimistic locking than under pessimistic locking?

**Level 5 — Building it yourself**
9. Explain why an optimistic retry loop must re-read the entity on every attempt, not reuse the object from the first attempt.
10. How is a version-based database CAS the same underlying idea as `AtomicStampedReference`, and what specific bug (ABA) does the version number prevent that a bare value comparison wouldn't?

**Final challenge:** A flash sale's traffic is 100x higher than this guide's worked example, targeting a single product row, for a ten-second window. Design the actual production system you'd deploy for that ten-second window specifically — which of Part 6's optimizations would you combine, in what order would you reach for them, and what would you monitor in real time to know if your chosen combination is actually working?

---

# 54. Final Takeaway

Both locking strategies exist to answer the exact same question — "did anything else touch this data between when I read it and when I write it?" — but they answer it at opposite ends of the timeline: pessimistic locking makes the question **unaskable** by preventing anyone else from touching the data in the first place (§14); optimistic locking lets the question get asked, honestly, at write time, and simply refuses to commit a write built on a wrong answer (§22). Neither is a strictly better default — the flash-sale example this guide traced from a real bug (§9) through two independent, provably correct fixes (§17, §25) and finally into a reusable, hand-built implementation (§30–§36) is exactly the kind of concrete scenario that should drive the choice, not an abstract preference for one paradigm. And neither is the end of the story: real systems at real scale layer sharding, queues, caching, and distributed coordination on top of whichever base strategy they start with (Part 6) — locking is the correctness foundation these optimizations get to safely build on, not a competitor to them.

