# TinyDB JDBC Driver — Step-by-Step Implementation Guide

> **Goal:** Build a real, loadable JDBC driver for TinyDB — one that DBeaver (or IntelliJ's database tool, or any other JDBC-compliant SQL client) can connect to directly, browse tables and columns through, and run queries against — as a standalone project with real, working code for every JDBC interface that actually matters.
>
> This guide is the third in the TinyDB series, and it is deliberately the seam where the other two meet: the [TinyDB Query Engine and SQL Parser guide](<TinyDB Query Engine and SQL Parser — Step-by-Step Implementation Guide.md>) built `BasicQueryEngine`, explicitly commenting that it is *"the externally-visible database facade — what a CLI or a JDBC driver actually calls"* — but never built that driver. It also referenced a `TransactionManager` by name, for the exact same reason a JDBC `Connection`'s commit/rollback semantics need one, without ever implementing it. The [TinyDB Storage Engine guide](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) built the real, durable `FileStorageEngine` those transactions actually run against. This guide closes both gaps and wires every piece together into one JAR a SQL editor can load.

---

# 1. What We Are Building

```text
DBeaver (or any JDBC client)
      |
   java.sql.* API  (DriverManager, Connection, Statement, ResultSet, DatabaseMetaData)
      |
   TinyDbDriver / TinyDbConnection / TinyDbStatement / TinyDbResultSet   <-- THIS GUIDE
      |
   BasicQueryEngine / BasicPlanner / PersistentCatalog   (companion Query Engine guide)
      |
   FileStorageEngine / TableHeap / WalManager             (companion Storage Engine guide)
      |
   Disk
```

By the end of this guide you will have:

- A real `java.sql.Driver`, auto-discoverable by `DriverManager` via `META-INF/services`, accepting a `jdbc:tinydb:` connection URL that points at a local, embedded, file-based database — the same architecture SQLite's and H2's own embedded JDBC drivers use.
- A real `Connection`, `Statement`, `PreparedStatement`, and `ResultSet`, each backed directly by the already-built `BasicQueryEngine`/`BasicPlanner`/`Row` classes — no new query engine, no new storage engine, only the adapter layer between them and the JDBC SPI.
- A real, previously-missing `TransactionManager`, filling the exact gap the Query Engine guide named but never built, making `Connection.setAutoCommit(false)`/`commit()`/`rollback()` behave correctly against the Storage Engine's real WAL-backed transactions.
- A real `DatabaseMetaData` implementation covering exactly the methods DBeaver actually calls to populate its schema browser — `getCatalogs`, `getSchemas`, `getTables`, `getColumns`, `getPrimaryKeys` — built from the already-existing `Catalog` interface, not hand-waved.
- A concrete, literal walkthrough of loading the driver JAR into DBeaver and connecting to a real TinyDB data directory.

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain exactly which `java.sql.*` interfaces a minimal, real JDBC driver must implement, and which methods on each interface actually matter for a tool like DBeaver versus which are boilerplate.
- Implement `java.sql.Driver` registration via `META-INF/services/java.sql.Driver`, and explain why that mechanism exists instead of requiring an explicit `Class.forName(...)` call.
- Bridge a `Connection`'s auto-commit/commit/rollback semantics onto a real transaction manager, including the specific case where auto-commit is `false` and a connection must hold one transaction open across many statements.
- Implement `PreparedStatement` parameter binding by rewriting `?` placeholders, and explain the SQL-injection risk this specifically avoids compared to naive string concatenation.
- Implement `DatabaseMetaData.getTables`/`getColumns` directly from an existing `Catalog` implementation, and explain exactly why DBeaver cannot browse a database's schema without them.
- Package the driver as a JAR and connect to it from DBeaver, end to end, with a real, working example.

---

# 3. Why This Matters (Interview Motivation)

> **"You've built a SQL engine and a storage engine. Now make it usable from a real tool a developer already has installed — no custom CLI, no bespoke protocol. Implement the actual JDBC driver."**

This is a genuinely different kind of question from "design the query engine" or "design the storage engine" — it's an **integration** question, and it tests a specific, often-overlooked skill:

- **Working inside someone else's contract** — `java.sql.*` is a specification TinyDB has zero control over; every method signature, every exception type, every semantic (what does `ResultSet.next()` do when there are no more rows? what must `wasNull()` return, and when?) is fixed, and a real driver has to honor all of it correctly, not approximately.
- **Knowing which 10% of a huge interface actually matters** — `Connection` alone declares around 70 methods; a real, working driver implements a handful of them for real and stubs the rest deliberately and explicitly, and knowing *which* handful is itself the skill being tested.
- **Bridging two already-built systems without changing either one** — this guide adds exactly one new class (`TransactionManager`) to make the seam work, and otherwise only ever calls existing, already-built APIs from both companion guides — a realistic simulation of the kind of integration work that dominates real backend engineering far more than greenfield design does.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | Matches both companion guides; `java.sql` is part of the JDK itself, no external dependency needed. |
| Driver discovery | `java.util.ServiceLoader` via `META-INF/services/java.sql.Driver` | The JDBC 4.0+ standard mechanism — the one that lets DBeaver find the driver class without the user typing it in by hand (§14). |
| Packaging | A plain JAR, built with the same build tool as the companion guides | DBeaver loads a driver exactly like any other JDBC driver — a JAR on its classpath, nothing TinyDB-specific about the packaging step itself (§35). |
| Testing | JUnit 5, plus a literal DBeaver connection test | A JDBC driver's correctness is only really proven by a real client successfully browsing schema and running a query — §41 treats that as a first-class test, not an afterthought. |

---

# 5. Project Structure

```text
tinydb-jdbc/
├── src/main/java/com/example/tinydb/jdbc/
│   ├── TinyDbDriver.java                                    // §15
│   ├── TinyDbConnection.java                                // §16-§18
│   ├── TinyDbStatement.java                                  // §19-§20
│   ├── TinyDbPreparedStatement.java                          // §21-§23
│   ├── TinyDbResultSet.java                                  // §24-§26
│   ├── TinyDbResultSetMetaData.java                          // §27
│   ├── TinyDbDatabaseMetaData.java                            // §28-§33
│   ├── TypeMapping.java                                       // §26
│   └── transaction/
│       ├── TransactionManager.java                            // §9
│       └── RealTransactionContext.java                        // §8
├── src/main/resources/
│   └── META-INF/services/java.sql.Driver                      // §14 -- contains one line, the driver's class name
└── src/test/java/com/example/tinydb/jdbc/
    ├── DriverRegistrationTest.java
    ├── PreparedStatementParameterBindingTest.java
    ├── TransactionCommitRollbackTest.java
    └── DatabaseMetaDataAgainstDbeaverTest.java
```

---

# 6. High-Level Architecture: Where the Driver Sits

The single most important architectural decision this guide makes, stated up front rather than discovered halfway through: **this is an embedded driver, running in the same JVM process as the client tool, with no network protocol anywhere.** DBeaver, when it loads `tinydb-jdbc.jar` and calls `DriverManager.getConnection("jdbc:tinydb:/path/to/data")`, is not opening a socket to a running TinyDB server — it is directly, in-process, constructing a `FileStorageEngine` (companion Storage Engine guide, §45) pointed at that path, and every subsequent `Statement.execute(...)` call is an ordinary Java method call down through `BasicQueryEngine` into that same in-process storage engine. This is exactly the architecture SQLite's and H2's (in embedded mode) own JDBC drivers use, and it's what makes "connect from any SQL editor" achievable without building a wire protocol first — §11 justifies this choice explicitly, and §42 names the server-mode alternative as real, but separate, future work.

---

# 7. Bridging Two Gaps the Companion Guides Left Open

Before any JDBC-specific code, two small but load-bearing gaps between the two companion guides need closing, because `Connection`'s transaction methods (§17-§18) depend on both:

- The Query Engine guide's `TransactionContext` (its own §15) is, verbatim, *"deliberately empty for now"* — a placeholder singleton, `TransactionContext.AUTOCOMMIT`, with no actual transaction identity. A JDBC `Connection` needs a **real** one: something that carries an actual transaction ID a `commit()`/`rollback()` call can act on.
- `BasicQueryEngine`'s constructor (that guide's §26) takes a `TransactionManager`, referenced by name — *"see the companion Storage Engine guide"* — but no such class exists in either guide. The Storage Engine guide's real `StorageEngine` interface (§45 there) exposes `beginTransaction(long id)`/`commitTransaction(long id)`/`abortTransaction(long id)` directly, keyed by a raw `long`, with nothing that allocates that ID or wraps it in a `TransactionContext` for a caller above it.

§8-§9 build both, as the one new piece of infrastructure this entire guide adds — everything else from here on is adapter code calling classes that already exist.

---

# 8. Implementing a Real TransactionContext

```java
public final class RealTransactionContext {

    private static final AtomicLong NEXT_TRANSACTION_ID = new AtomicLong(1);

    private final long transactionId;
    private volatile boolean completed; // true once committed or rolled back -- guards against double-completion

    private RealTransactionContext(long transactionId) { this.transactionId = transactionId; }

    static RealTransactionContext allocate() {
        return new RealTransactionContext(NEXT_TRANSACTION_ID.getAndIncrement());
    }

    public long transactionId() { return transactionId; }
    public boolean isCompleted() { return completed; }
    void markCompleted() { completed = true; }
}
```

This replaces the Query Engine guide's placeholder `TransactionContext.AUTOCOMMIT` for every code path this guide touches — `BasicPlanner.createQueryPlan(sql, tx)`/`executeUpdate(sql, tx)` (that guide's §25) take whatever the second parameter's actual type is; wiring this guide's driver against them means `RealTransactionContext` is the concrete type flowing through every one of those calls from here on, carrying a real, allocatable transaction ID the Storage Engine's `beginTransaction(long)`/`commitTransaction(long)`/`abortTransaction(long)` (companion guide, §45) can actually act on.

---

# 9. Implementing TransactionManager

```java
public final class TransactionManager {

    private final StorageEngine storageEngine; // companion Storage Engine guide, §45

    public TransactionManager(StorageEngine storageEngine) { this.storageEngine = storageEngine; }

    public RealTransactionContext begin() {
        RealTransactionContext tx = RealTransactionContext.allocate(); // §8
        try {
            storageEngine.beginTransaction(tx.transactionId());
        } catch (IOException e) {
            throw new TinyDbRuntimeException("Failed to begin transaction", e);
        }
        return tx;
    }

    public void commit(RealTransactionContext tx) {
        if (tx.isCompleted()) throw new IllegalStateException("Transaction " + tx.transactionId() + " already completed");
        try {
            storageEngine.commitTransaction(tx.transactionId());
        } catch (IOException e) {
            throw new TinyDbRuntimeException("Failed to commit transaction " + tx.transactionId(), e);
        } finally {
            tx.markCompleted();
        }
    }

    public void rollback(RealTransactionContext tx) {
        if (tx.isCompleted()) throw new IllegalStateException("Transaction " + tx.transactionId() + " already completed");
        try {
            storageEngine.abortTransaction(tx.transactionId());
        } catch (IOException e) {
            throw new TinyDbRuntimeException("Failed to abort transaction " + tx.transactionId(), e);
        } finally {
            tx.markCompleted();
        }
    }
}
```

This is the exact class `BasicQueryEngine`'s constructor (companion Query Engine guide, §26) was already written expecting — nothing about `BasicQueryEngine.doQuery`/`doUpdate` needs to change to accept it; the gap named in §7 was purely that nothing had implemented it yet. `isCompleted()`'s guard against double-completion matters specifically for §17-§18: a JDBC `Connection.close()` that runs after an explicit `commit()` already happened must never attempt a second commit or an implicit rollback against an already-finished transaction.

---

# 10. The JDBC SPI Surface: Which Interfaces We Actually Need to Implement

| Interface | Why it's needed | Built in |
|---|---|---|
| `java.sql.Driver` | The entry point `DriverManager` (and DBeaver) discovers and calls to obtain a `Connection` | §15 |
| `java.sql.Connection` | Owns the transaction boundary, and factories `Statement`/`PreparedStatement`/`DatabaseMetaData` | §16-§18 |
| `java.sql.Statement` | Executes a fixed SQL string, once, with no parameters | §19-§20 |
| `java.sql.PreparedStatement` | Executes a parameterized SQL string, safely, potentially many times | §21-§23 |
| `java.sql.ResultSet` | Iterates rows and columns of a query result | §24-§26 |
| `java.sql.ResultSetMetaData` | Describes a result set's columns — names, types — without materializing rows | §27 |
| `java.sql.DatabaseMetaData` | Describes the database itself — catalogs, schemas, tables, columns — what DBeaver's schema browser is built entirely on | §28-§33 |

Every one of these is a genuinely large interface in the real JDBC specification (`ResultSet` alone declares roughly 200 methods). This guide implements every method that has real, correct behavior to provide, and is explicit, every time, about which remaining methods are deliberately left as boilerplate (`throw new SQLFeatureNotSupportedException(...)`) because no real client this guide targets ever calls them — never silently omitted, always a stated, visible decision.

---

# 11. Choosing Embedded Over Client-Server for This Driver

Three concrete reasons this guide commits to the embedded model (§6) rather than a network protocol talking to a separately-running TinyDB server process:

- **Zero new infrastructure**: the companion Storage Engine guide's `FileStorageEngine` already runs perfectly well inside any JVM process, including DBeaver's own — there is no server to write, deploy, or keep running.
- **Matches real precedent**: SQLite's JDBC driver, and H2 in embedded mode, work exactly this way, and both are routinely used from DBeaver — this is not a toy simplification, it's a legitimate, common JDBC driver architecture.
- **The hard parts are identical either way**: everything genuinely difficult in this guide — transaction semantics (§8-§9), parameter binding (§21-§23), metadata (§28-§33) — is exactly as necessary and exactly as hard whether the `BasicQueryEngine` call at the bottom of the stack happens to be a local method call or the result of a network round trip. Choosing embedded defers the *networking* problem, not the *JDBC* problem, which is the one this guide is actually about.

A server-mode variant, connecting to a separately-running TinyDB process over a socket, is named honestly as real, valuable future work (§42) rather than pretended away — the two [companion Storage Engine guide's embedded-vs-server-mode section](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) already establishes that `StorageEngine` itself is agnostic to which mode wraps it; only this driver's `Connection` layer would need a second implementation.

---

# 12. The Connection URL Format

```text
jdbc:tinydb:/absolute/path/to/data/directory
jdbc:tinydb:./relative/path/to/data
```

Modeled directly on SQLite's own `jdbc:sqlite:/path/to/file.db` — everything after the second colon is a filesystem path, passed straight through to `FileStorageEngine`'s constructor (companion Storage Engine guide, §45) as its `dataRoot`. No host, no port, no username or password segment — an embedded, single-user, file-based database has none of those concepts, and a URL format that pretended otherwise would only mislead a user filling in DBeaver's connection dialog (§36).

---

# 13. Bootstrapping a TinyDB Instance From a URL

```java
public final class TinyDbInstance {

    private final BasicQueryEngine queryEngine;
    private final BasicPlanner planner;                    // companion Query Engine guide, §25 -- used directly, §17, when autoCommit is false
    private final TransactionManager transactionManager; // §9
    private final Catalog catalog;                        // companion Query Engine guide, §33

    private TinyDbInstance(BasicQueryEngine queryEngine, BasicPlanner planner, TransactionManager transactionManager, Catalog catalog) {
        this.queryEngine = queryEngine;
        this.planner = planner;
        this.transactionManager = transactionManager;
        this.catalog = catalog;
    }

    /** Wires together every layer of BOTH companion guides, starting purely from a filesystem path. */
    public static TinyDbInstance open(Path dataDirectory) throws IOException {
        StorageEngine storageEngine = new FileStorageEngine(dataDirectory, 1000); // Storage Engine guide, §45
        storageEngine.recover();                                                    // §40/§43 there -- MUST run first

        Catalog catalog = new PersistentCatalog(storageEngine);                     // Query Engine guide, §36
        QueryPlanner queryPlanner = new BasicQueryPlanner(catalog);                 // §24 there
        UpdatePlanner updatePlanner = new BasicUpdatePlanner(catalog, storageEngine); // §24 there
        BasicPlanner planner = new BasicPlanner(queryPlanner, updatePlanner);        // §25 there

        TransactionManager transactionManager = new TransactionManager(storageEngine); // §9, THIS guide
        BasicQueryEngine queryEngine = new BasicQueryEngine(planner, transactionManager); // §26 there

        return new TinyDbInstance(queryEngine, planner, transactionManager, catalog);
    }

    public BasicQueryEngine queryEngine() { return queryEngine; }
    public BasicPlanner planner() { return planner; }
    public TransactionManager transactionManager() { return transactionManager; }
    public Catalog catalog() { return catalog; }
}
```

`storageEngine.recover()` running **before** anything else touches the database is not optional ordering — the [companion Storage Engine guide's RecoveryManager section](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) states this precisely: a database that started serving queries against pages recovery hasn't finished fixing up yet would be handing out answers from a state that never actually existed. Every `TinyDbConnection` (§16) this driver ever creates for the *same* data directory shares exactly one `TinyDbInstance`, never one per connection — opening the same on-disk database twice, independently, from two different `TinyDbInstance`s would mean two separate, uncoordinated buffer pools and WAL managers fighting over the same files, which the Storage Engine guide's entire concurrency model (its own Part 10) assumes never happens.

---

# 14. Driver Registration: java.sql.Driver and META-INF/services

A JDBC 4.0+ driver never requires its caller to write `Class.forName("com.example.tinydb.jdbc.TinyDbDriver")` — `java.sql.DriverManager` uses `java.util.ServiceLoader` to discover every driver on the classpath automatically, by reading a plain text file:

```text
# src/main/resources/META-INF/services/java.sql.Driver
com.example.tinydb.jdbc.TinyDbDriver
```

The moment `tinydb-jdbc.jar` is on DBeaver's classpath, `ServiceLoader` finds this file, reads the one class name in it, instantiates that class via its no-argument constructor, and calls `DriverManager.registerDriver(...)` on the caller's behalf — this is the entire mechanism behind "DBeaver just finds the driver," and it's why §15's `TinyDbDriver` needs a public no-argument constructor and nothing more to be discoverable.

---

# 15. Implementing TinyDbDriver

```java
public final class TinyDbDriver implements java.sql.Driver {

    private static final String URL_PREFIX = "jdbc:tinydb:";
    private static final Map<Path, TinyDbInstance> OPEN_INSTANCES = new ConcurrentHashMap<>(); // §13 -- one per data directory

    static {
        try {
            DriverManager.registerDriver(new TinyDbDriver());
        } catch (SQLException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public TinyDbDriver() { } // required -- ServiceLoader (§14) instantiates via this exact constructor

    @Override
    public boolean acceptsURL(String url) { return url != null && url.startsWith(URL_PREFIX); }

    @Override
    public Connection connect(String url, Properties info) throws SQLException {
        if (!acceptsURL(url)) return null; // §16's contract: a Driver that doesn't understand a URL returns null, never throws

        Path dataDirectory = Path.of(url.substring(URL_PREFIX.length()));
        try {
            TinyDbInstance instance = OPEN_INSTANCES.computeIfAbsent(dataDirectory.toAbsolutePath(), path -> {
                try {
                    return TinyDbInstance.open(path); // §13
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
            return new TinyDbConnection(instance); // §16
        } catch (UncheckedIOException e) {
            throw new SQLException("Failed to open TinyDB data directory: " + dataDirectory, e.getCause());
        }
    }

    @Override public int getMajorVersion() { return 1; }
    @Override public int getMinorVersion() { return 0; }
    @Override public boolean jdbcCompliant() { return false; } // honest -- this driver does not implement the FULL spec (§10)
    @Override public Logger getParentLogger() { throw new SQLFeatureNotSupportedException(); }
    @Override public DriverPropertyInfo[] getPropertyInfo(String url, Properties info) { return new DriverPropertyInfo[0]; }
}
```

`OPEN_INSTANCES.computeIfAbsent(...)`, keyed by the data directory's absolute path, is what enforces §13's "exactly one `TinyDbInstance` per data directory" rule even when DBeaver opens several `Connection`s to the same database concurrently (which it routinely does — one for browsing schema, one for running a query, one for an auto-refresh) — every one of them resolves to the identical shared `BasicQueryEngine`/`FileStorageEngine` pair, never a competing second copy.

---

# 16. Implementing TinyDbConnection

```java
public final class TinyDbConnection implements Connection {

    private final TinyDbInstance instance;
    private boolean autoCommit = true;                    // JDBC's own default, per the specification
    private RealTransactionContext currentTransaction;     // non-null only while autoCommit is false and a tx is open
    private boolean closed;

    public TinyDbConnection(TinyDbInstance instance) { this.instance = instance; }

    @Override
    public Statement createStatement() throws SQLException {
        checkOpen();
        return new TinyDbStatement(this); // §19
    }

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        checkOpen();
        return new TinyDbPreparedStatement(this, sql); // §21
    }

    @Override
    public DatabaseMetaData getMetaData() throws SQLException {
        checkOpen();
        return new TinyDbDatabaseMetaData(instance.catalog()); // §28
    }

    /** Package-private -- Statement/PreparedStatement call this, never the query engine directly, so every
     *  statement's transaction handling goes through ONE place. §17 explains the two branches precisely. */
    QueryResult executeInternal(String sql, boolean isQuery) throws SQLException {
        checkOpen();
        try {
            if (autoCommit) {
                return isQuery ? instance.queryEngine().doQuery(sql) : instance.queryEngine().doUpdate(sql); // §26/§26 there
            }
            return executeWithinOpenTransaction(sql, isQuery); // §17
        } catch (RuntimeException e) {
            throw new SQLException("Statement execution failed: " + e.getMessage(), e);
        }
    }

    private void checkOpen() throws SQLException {
        if (closed) throw new SQLException("Connection is closed");
    }

    // getAutoCommit()/setAutoCommit(...)/commit()/rollback()/close() are §17-§18.
    // isClosed()/isValid(...)/getCatalog()/setReadOnly(...)/etc. are boilerplate, elided per §10's stated policy.
}
```

---

# 17. Auto-Commit and Transaction Semantics in the Connection

Two genuinely different execution paths, and getting this distinction right is the entire point of this section:

- **`autoCommit == true`** (JDBC's default): every single `Statement`/`PreparedStatement` call is its own, complete, implicit transaction. This is *exactly* what `BasicQueryEngine.doQuery`/`doUpdate` (companion Query Engine guide, §26) already do internally — each one calls `transactionManager.begin()`, runs the plan, calls `transactionManager.commit(tx)`, all in one method. §16's `executeInternal` calls them directly for this case, for good reason: no reason to duplicate logic those methods already implement correctly.
- **`autoCommit == false`**: the *connection itself* now owns one open transaction, spanning however many statements the client executes, until an explicit `commit()` or `rollback()`. `BasicQueryEngine.doQuery`/`doUpdate` cannot be used here at all — they always begin AND commit internally, which would silently commit after every single statement regardless of what the JDBC caller asked for. This case has to drop down one layer, to `BasicPlanner.createQueryPlan(sql, tx)`/`executeUpdate(sql, tx)` (companion guide, §25) directly, passing the connection's own held-open `RealTransactionContext` (§8) instead of letting anything begin or commit on its own.

```java
private QueryResult executeWithinOpenTransaction(String sql, boolean isQuery) throws SQLException {
    if (currentTransaction == null) {
        currentTransaction = instance.transactionManager().begin(); // §9 -- opens ONE transaction, held across calls
    }
    try {
        BasicPlanner planner = instance.planner(); // exposed alongside queryEngine()/catalog(), §13
        if (isQuery) {
            Plan plan = planner.createQueryPlan(sql, currentTransaction);
            return materializeAsQueryResult(plan); // walks the Plan's Scan into rows, mirroring doQuery's own logic
        } else {
            int affected = planner.executeUpdate(sql, currentTransaction);
            return QueryResult.ofAffected(affected);
        }
    } catch (RuntimeException e) {
        throw new SQLException("Statement failed inside an open transaction", e);
    }
}
```

---

# 18. Implementing commit()/rollback()/setAutoCommit()/close()

```java
@Override
public void setAutoCommit(boolean autoCommit) throws SQLException {
    checkOpen();
    if (this.autoCommit == autoCommit) return; // no-op -- nothing to reconcile
    if (!autoCommit) {
        this.autoCommit = false; // switching INTO manual mode -- no transaction opened yet, § 17 opens one lazily
        return;
    }
    // Switching FROM manual mode back to auto-commit -- the JDBC spec requires committing whatever's still open.
    if (currentTransaction != null) {
        instance.transactionManager().commit(currentTransaction);
        currentTransaction = null;
    }
    this.autoCommit = true;
}

@Override
public void commit() throws SQLException {
    checkOpen();
    if (autoCommit) throw new SQLException("commit() is not valid while autoCommit is true");
    if (currentTransaction != null) {
        instance.transactionManager().commit(currentTransaction); // §9
        currentTransaction = null; // the NEXT statement (§17) lazily opens a fresh transaction
    }
}

@Override
public void rollback() throws SQLException {
    checkOpen();
    if (autoCommit) throw new SQLException("rollback() is not valid while autoCommit is true");
    if (currentTransaction != null) {
        instance.transactionManager().rollback(currentTransaction); // §9
        currentTransaction = null;
    }
}

@Override
public void close() throws SQLException {
    if (closed) return; // idempotent, per the JDBC spec
    if (!autoCommit && currentTransaction != null && !currentTransaction.isCompleted()) {
        instance.transactionManager().rollback(currentTransaction); // never leave a transaction dangling open
    }
    closed = true;
}
```

`close()` rolling back an unfinished transaction, rather than silently committing it or silently leaving it open, matches the real JDBC specification's own stated behavior and is the safer default for exactly the reason [the TinyDB Storage Engine guide's own crash-recovery discipline](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) insists on elsewhere: an incomplete piece of work should never be mistaken for a committed one — a client that forgot to call `commit()` before disconnecting almost certainly did not intend for that half-finished work to become permanent.

---

# 19. Implementing TinyDbStatement

```java
public final class TinyDbStatement implements Statement {

    protected final TinyDbConnection connection;
    private TinyDbResultSet currentResultSet;
    private int lastUpdateCount = -1;

    public TinyDbStatement(TinyDbConnection connection) { this.connection = connection; }

    @Override
    public ResultSet executeQuery(String sql) throws SQLException {
        QueryResult result = connection.executeInternal(sql, true); // §16-§17
        currentResultSet = new TinyDbResultSet(result.rows(), this); // §24
        return currentResultSet;
    }

    @Override
    public int executeUpdate(String sql) throws SQLException {
        QueryResult result = connection.executeInternal(sql, false);
        lastUpdateCount = result.affectedRows();
        return lastUpdateCount;
    }

    @Override
    public boolean execute(String sql) throws SQLException {
        return dispatchByStatementKind(sql); // §20
    }

    @Override public ResultSet getResultSet() throws SQLException { return currentResultSet; }
    @Override public int getUpdateCount() throws SQLException { return lastUpdateCount; }
    @Override public Connection getConnection() throws SQLException { return connection; }
    @Override public void close() throws SQLException { if (currentResultSet != null) currentResultSet.close(); }
}
```

---

# 20. Dispatching execute/executeQuery/executeUpdate

`Statement.execute(sql)` is JDBC's own "I don't know in advance whether this is a query or an update" entry point — a real client (DBeaver included) uses it for arbitrary, user-typed SQL where the *caller* genuinely doesn't want to have to pre-classify the statement itself:

```java
private boolean dispatchByStatementKind(String sql) throws SQLException {
    boolean isQuery = sql.strip().regionMatches(true, 0, "SELECT", 0, 6); // §20's ENTIRE classification logic

    if (isQuery) {
        executeQuery(sql);
        return true; // per the JDBC contract: true means "call getResultSet()"
    } else {
        executeUpdate(sql);
        return false; // false means "call getUpdateCount()"
    }
}
```

A case-insensitive prefix check on `SELECT` is a deliberately minimal classifier — it is correct for every statement type the companion Query Engine guide's grammar actually supports (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, `DROP TABLE`, that guide's §8), and it sidesteps needing to run the real lexer/parser twice (once to classify, once to execute) for a decision this simple. A grammar that later grows statement types where "does this return rows" isn't determinable from the first keyword alone (a `WITH` CTE prefixing a `SELECT`, say) would need a real classification step here instead — named honestly as a limitation of this simplification, not hidden.

---

# 21. Implementing TinyDbPreparedStatement: Parameter Placeholders

```java
public final class TinyDbPreparedStatement extends TinyDbStatement implements PreparedStatement {

    private final String sqlTemplate;         // the ORIGINAL sql, with "?" placeholders still in place
    private final Object[] parameters;         // positional -- parameters[0] corresponds to the FIRST "?"
    private final int parameterCount;

    public TinyDbPreparedStatement(TinyDbConnection connection, String sqlTemplate) {
        super(connection);
        this.sqlTemplate = sqlTemplate;
        this.parameterCount = countPlaceholders(sqlTemplate); // §22
        this.parameters = new Object[parameterCount];
    }

    @Override
    public ResultSet executeQuery() throws SQLException { return executeQuery(bindParameters()); } // §22, then §19's method

    @Override
    public int executeUpdate() throws SQLException { return executeUpdate(bindParameters()); }

    @Override
    public void setString(int index, String value) { setParameter(index, value); }     // §23
    @Override
    public void setInt(int index, int value) { setParameter(index, value); }
    @Override
    public void setNull(int index, int sqlType) { setParameter(index, null); }

    private void setParameter(int index, Object value) {
        parameters[index - 1] = value; // JDBC parameter indices are 1-based -- §22's placeholders are 0-based internally
    }

    private String bindParameters() throws SQLException { return SqlParameterBinder.bind(sqlTemplate, parameters); } // §22
}
```

---

# 22. Parsing and Rewriting `?` Placeholders Into a Bound SQL Statement

```java
public final class SqlParameterBinder {

    public static String bind(String sqlTemplate, Object[] parameters) throws SQLException {
        StringBuilder bound = new StringBuilder();
        int parameterIndex = 0;
        boolean insideStringLiteral = false;

        for (int i = 0; i < sqlTemplate.length(); i++) {
            char c = sqlTemplate.charAt(i);
            if (c == '\'') insideStringLiteral = !insideStringLiteral; // a "?" inside a string literal is NOT a placeholder

            if (c == '?' && !insideStringLiteral) {
                if (parameterIndex >= parameters.length) {
                    throw new SQLException("Not enough parameters bound for this statement");
                }
                bound.append(literalFor(parameters[parameterIndex++]));
            } else {
                bound.append(c);
            }
        }
        if (parameterIndex != parameters.length) {
            throw new SQLException("Parameter count mismatch: statement declares " + parameterIndex
                    + " placeholders, " + parameters.length + " were set");
        }
        return bound.toString();
    }

    private static String literalFor(Object value) {
        if (value == null) return "NULL";
        if (value instanceof String s) return "'" + s.replace("'", "''") + "'"; // §22's escaping -- see the note below
        if (value instanceof Boolean || value instanceof Number) return value.toString();
        if (value instanceof Instant instant) return "'" + instant.toString() + "'";
        throw new IllegalArgumentException("Unsupported parameter type: " + value.getClass());
    }

    public static int countPlaceholders(String sqlTemplate) {
        int count = 0;
        boolean insideStringLiteral = false;
        for (char c : sqlTemplate.toCharArray()) {
            if (c == '\'') insideStringLiteral = !insideStringLiteral;
            if (c == '?' && !insideStringLiteral) count++;
        }
        return count;
    }
}
```

Escaping a single quote inside a string parameter by **doubling** it (`'` -> `''`) is exactly what turns this from a SQL-injection-shaped hole into a safe substitution — a parameter value of `O'Brien` becomes the SQL literal `'O''Brien'`, which the companion Query Engine guide's `SqlLexer` (that guide's §7) parses back as the single, correct string `O'Brien`, never as a string that ends early and lets the rest of the value's content be interpreted as SQL syntax. This is textual substitution done *safely*, specifically because it escapes before substituting — it is emphatically not the same operation as a caller naively concatenating an unescaped parameter directly into a SQL string, which is the actual vulnerability `PreparedStatement` exists to prevent in the first place. A production-grade driver typically goes one step further and passes bound values to the executor as **already-typed values** rather than re-serialized SQL text at all (avoiding a second parse-and-lex round trip); this guide's text-substitution approach is the simpler, still-correct version, named explicitly as a simplification (§42).

---

# 23. The Remaining Setters, and Type Coercion Against TinyDB's Real Storage Types

The companion Storage Engine guide's `BinaryRowCodec` (that guide's §16) fixes exactly which Java runtime type each `DataType` decodes to — `DECIMAL` as a `Double`, `DATE`/`TIMESTAMP` as an `Instant`, never a `BigDecimal` or a `java.sql.Date`. Every remaining `PreparedStatement` setter has to coerce *into* one of those exact types, not into whatever type feels most natural for the setter's own name:

```java
@Override public void setLong(int index, long value) { setParameter(index, value); }         // BIGINT -> Long, matches exactly
@Override public void setBoolean(int index, boolean value) { setParameter(index, value); }    // BOOLEAN -> Boolean, matches exactly

@Override
public void setBigDecimal(int index, BigDecimal value) {
    setParameter(index, value == null ? null : value.doubleValue()); // DECIMAL is stored as a Double (§26) -- coerce here, once
}

@Override
public void setDate(int index, java.sql.Date value) {
    setParameter(index, value == null ? null : Instant.ofEpochMilli(value.getTime())); // DATE/TIMESTAMP -> Instant (§26)
}

@Override
public void setTimestamp(int index, java.sql.Timestamp value) {
    setParameter(index, value == null ? null : value.toInstant());
}

@Override public void clearParameters() { Arrays.fill(parameters, null); }
```

Coercing `BigDecimal`/`java.sql.Date`/`java.sql.Timestamp` **once, here, at bind time**, rather than deferring the conversion to whatever eventually reads the value back out, means every layer beneath `PreparedStatement` — the SQL text substitution (§22), the lexer, the storage codec — only ever has to deal with exactly the types it was already built to handle, never a second, parallel set of JDBC-flavored types leaking down into code that was never written to expect them.

---

# 24. Implementing TinyDbResultSet

```java
public final class TinyDbResultSet implements ResultSet {

    private final List<Row> rows;              // companion Query Engine guide's Row, §9
    private final List<String> columnNames;     // fixed order, taken from the first row -- §25 explains why that's safe here
    private final Statement statement;
    private int currentIndex = -1;              // -1 means "before the first row" -- next() must be called at least once
    private boolean lastValueWasNull;
    private boolean closed;

    public TinyDbResultSet(List<Row> rows, Statement statement) {
        this.rows = rows;
        this.statement = statement;
        this.columnNames = rows.isEmpty() ? List.of() : new ArrayList<>(rows.get(0).columnNames());
    }

    @Override
    public boolean next() throws SQLException {
        checkOpen();
        if (currentIndex + 1 >= rows.size()) return false;
        currentIndex++;
        return true;
    }

    @Override public boolean isClosed() { return closed; }
    @Override public void close() { closed = true; }
    @Override public Statement getStatement() { return statement; }
    @Override public ResultSetMetaData getMetaData() { return new TinyDbResultSetMetaData(columnNames, currentRow()); } // §27
    @Override public boolean wasNull() { return lastValueWasNull; }

    private Row currentRow() throws SQLException {
        if (currentIndex < 0 || currentIndex >= rows.size()) {
            throw new SQLException("Cursor is not positioned on a valid row -- call next() first");
        }
        return rows.get(currentIndex);
    }

    private void checkOpen() throws SQLException { if (closed) throw new SQLException("ResultSet is closed"); }

    // getInt/getString/getObject/etc. by index AND by name are §25. getBigDecimal/getDate/getTimestamp
    // coercion mirrors §23's setters, in reverse. updateXxx()/insertRow()/deleteRow() (an UPDATABLE result
    // set) are never implemented -- this driver's result sets are read-only/forward-only, §28's stated scope.
}
```

`currentIndex` starting at `-1`, with `next()` required before any column access is valid, is the exact cursor discipline every JDBC client (DBeaver included) already assumes — a result set that let a caller read column 1 *before* the first `next()` call would violate the specification's own stated contract, and a client relying on that contract (correctly) would get subtly wrong behavior nowhere in its own code to blame.

---

# 25. Column Access by Index and by Name, and wasNull() Tracking

```java
@Override
public Object getObject(int columnIndex) throws SQLException {
    Row row = currentRow();
    String columnName = columnNames.get(columnIndex - 1); // JDBC columns are 1-based
    Object value = row.get(columnName);
    lastValueWasNull = (value == null); // §24's wasNull() reads exactly this flag -- set on EVERY access, not just null ones
    return value;
}

@Override
public Object getObject(String columnLabel) throws SQLException {
    Row row = currentRow();
    Object value = row.get(columnLabel);
    lastValueWasNull = (value == null);
    return value;
}

@Override public String getString(int columnIndex) throws SQLException { return (String) getObject(columnIndex); }
@Override public String getString(String columnLabel) throws SQLException { return (String) getObject(columnLabel); }

@Override
public int getInt(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex);
    return value == null ? 0 : ((Number) value).intValue(); // JDBC contract: a null numeric column returns 0, use wasNull() to disambiguate
}

@Override
public long getLong(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex);
    return value == null ? 0L : ((Number) value).longValue();
}

@Override
public BigDecimal getBigDecimal(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex); // a Double, per §26's DECIMAL mapping
    return value == null ? null : BigDecimal.valueOf((Double) value);
}

@Override
public java.sql.Timestamp getTimestamp(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex); // an Instant, per §26's DATE/TIMESTAMP mapping
    return value == null ? null : java.sql.Timestamp.from((Instant) value);
}
```

`getInt`/`getLong` returning `0` for a `null` column, rather than throwing or returning some sentinel, is a real, sometimes-surprising **JDBC specification requirement** — a primitive `int` cannot represent `null` at all, so the specification's answer is "return zero, and the caller who cares about the difference calls `wasNull()` immediately afterward." Setting `lastValueWasNull` inside `getObject` — the single method every other getter above ultimately calls through — means that flag is always correct after *any* column access, never only after the specific getter a caller happens to have used.

---

# 26. Mapping TinyDB's DataType to java.sql.Types and Back

| TinyDB `DataType` | Java runtime type (§16 in the Storage Engine guide's codec) | `java.sql.Types` constant |
|---|---|---|
| `INT` | `Integer` | `Types.INTEGER` |
| `BIGINT` | `Long` | `Types.BIGINT` |
| `BOOLEAN` | `Boolean` | `Types.BOOLEAN` |
| `DECIMAL` | `Double` (a stated simplification, not `BigDecimal` — that guide's §16) | `Types.DOUBLE` |
| `DATE` | `Instant` | `Types.DATE` |
| `TIMESTAMP` | `Instant` | `Types.TIMESTAMP` |
| `VARCHAR` | `String` | `Types.VARCHAR` |

```java
public final class TypeMapping {

    public static int toJdbcType(DataType dataType) {
        return switch (dataType) {
            case INT -> Types.INTEGER;
            case BIGINT -> Types.BIGINT;
            case BOOLEAN -> Types.BOOLEAN;
            case DECIMAL -> Types.DOUBLE;      // NOT Types.DECIMAL -- honest about the underlying Double representation
            case DATE -> Types.DATE;
            case TIMESTAMP -> Types.TIMESTAMP;
            case VARCHAR -> Types.VARCHAR;
        };
    }

    public static String toJdbcTypeName(DataType dataType) { return dataType.name(); } // TinyDB's own names double as JDBC type names
}
```

Mapping `DECIMAL` to `Types.DOUBLE` rather than `Types.DECIMAL` is a deliberate, stated honesty check, not an oversight: reporting `Types.DECIMAL` while the underlying value is actually a `Double` would mislead a client (DBeaver included) that inspects the reported type to decide *which getter is safe to call* — `Types.DOUBLE` accurately describes what `getObject`/`getBigDecimal` (§25) actually hand back.

---

# 27. Implementing ResultSetMetaData

```java
public final class TinyDbResultSetMetaData implements ResultSetMetaData {

    private final List<String> columnNames;
    private final Row sampleRow; // used ONLY to infer each column's DataType, §26 -- never to read actual values

    public TinyDbResultSetMetaData(List<String> columnNames, Row sampleRow) {
        this.columnNames = columnNames;
        this.sampleRow = sampleRow;
    }

    @Override public int getColumnCount() { return columnNames.size(); }
    @Override public String getColumnName(int column) { return columnNames.get(column - 1); }
    @Override public String getColumnLabel(int column) { return getColumnName(column); }

    @Override
    public int getColumnType(int column) throws SQLException {
        Object sampleValue = sampleRow.get(getColumnName(column));
        return TypeMapping.toJdbcType(inferDataType(sampleValue)); // §26
    }

    @Override
    public String getColumnTypeName(int column) throws SQLException {
        return TypeMapping.toJdbcTypeName(inferDataType(sampleRow.get(getColumnName(column))));
    }

    private DataType inferDataType(Object value) {
        return switch (value) {
            case Integer i -> DataType.INT;
            case Long l -> DataType.BIGINT;
            case Boolean b -> DataType.BOOLEAN;
            case Double d -> DataType.DECIMAL;
            case Instant i -> DataType.TIMESTAMP;
            case String s -> DataType.VARCHAR;
            case null, default -> DataType.VARCHAR; // an all-null column has no sample to infer from -- a safe, stated fallback
        };
    }
}
```

Inferring a column's type from a **sample row's runtime value**, rather than from the table's actual declared `TableSchema` (companion Query Engine guide, §9), is this section's one honest shortcut: it's simpler, and correct for every ordinary query, but it silently gets an all-`NULL`-in-every-row column wrong (falling back to `VARCHAR`) and can't distinguish two integer columns' differing nullability. §33's `getColumns()` — driven directly from `Catalog.getTable(...)`'s real, declared schema, never from a sample row — does not have this limitation, and DBeaver's schema browser (§28 onward) uses exactly that path, not this one; this section's approach is deliberately scoped only to describing an *already-executed query's* result shape, where no other source of truth is available.

---

# 28. Why DBeaver Needs DatabaseMetaData in Real Depth

Everything built so far (§15-§27) is enough to run a query and read its result — but DBeaver's *schema browser*, the tree on the left showing every database, table, and column before a user has typed a single SQL statement, is built **entirely** on `Connection.getMetaData()` (§16) and a handful of specific `DatabaseMetaData` methods, called automatically the moment a connection opens, with no SQL involved at all. Get these wrong, and DBeaver connects successfully but shows an empty or broken schema tree — a failure mode that looks like a connection problem but is actually a metadata-completeness problem, which is exactly why this section treats `DatabaseMetaData` as a first-class, non-optional part of the driver rather than an afterthought bolted on once "the real work" is done.

---

# 29. Implementing getCatalogs() and getSchemas()

```java
public final class TinyDbDatabaseMetaData implements DatabaseMetaData {

    private final Catalog catalog; // companion Query Engine guide, §33

    public TinyDbDatabaseMetaData(Catalog catalog) { this.catalog = catalog; }

    @Override
    public ResultSet getCatalogs() throws SQLException {
        // TinyDB has exactly ONE catalog per data directory (§12's URL format -- no multi-catalog concept
        // exists anywhere in the companion guides), so this always returns a single-row result.
        return MetadataResultSets.singleColumn("TABLE_CAT", List.of("tinydb"));
    }

    @Override
    public ResultSet getSchemas() throws SQLException {
        // Likewise, exactly one schema -- DBeaver's tree still expects a non-empty schemas result to
        // render a schema node at all, so returning an empty result here would hide every table beneath it.
        return MetadataResultSets.twoColumn("TABLE_SCHEM", "TABLE_CATALOG", List.of(List.of("public", "tinydb")));
    }

    @Override public String getDatabaseProductName() { return "TinyDB"; }
    @Override public String getDatabaseProductVersion() { return "1.0"; }
    @Override public String getDriverName() { return "TinyDB JDBC Driver"; }
    @Override public String getDriverVersion() { return "1.0"; }
}
```

`MetadataResultSets` is a small internal helper (elided here) that builds a `TinyDbResultSet` (§24) directly from an in-memory list of rows, rather than routing a manufactured metadata answer back through the SQL layer at all — `getCatalogs`/`getSchemas` have nothing to do with SQL execution, and forcing them through the lexer/parser/planner just to produce one or two fixed, known rows would be needless indirection for a fixed, known answer.

---

# 30. Implementing getTables()

```java
@Override
public ResultSet getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types) throws SQLException {
    List<TableSchema> allTables = this.catalog.listTables(); // companion Query Engine guide, §33's Catalog.listTables()

    List<List<Object>> rows = allTables.stream()
            .filter(t -> tableNamePattern == null || matchesSqlLikePattern(t.tableName(), tableNamePattern))
            .map(t -> List.<Object>of("tinydb", "public", t.tableName(), "TABLE", ""))
            .toList();

    return MetadataResultSets.of(
            List.of("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "TABLE_TYPE", "REMARKS"),
            rows);
}
```

Every one of `getTables`'s parameters — `catalog`, `schemaPattern`, `tableNamePattern`, `types` — is allowed to be `null`, meaning "don't filter on this," per the JDBC specification; this implementation only honors `tableNamePattern` (the one DBeaver's own "filter tables" search box actually drives) and ignores `catalog`/`schemaPattern` entirely, which is correct specifically *because* §29 already established there is only ever one catalog and one schema — filtering on a dimension with exactly one possible value can never change the result, so implementing that filter would be dead code with extra steps, not a missing feature.

---

# 31. Implementing getColumns()

```java
@Override
public ResultSet getColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern) throws SQLException {
    List<List<Object>> rows = new ArrayList<>();
    for (TableSchema table : this.catalog.listTables()) {
        if (tableNamePattern != null && !matchesSqlLikePattern(table.tableName(), tableNamePattern)) continue;

        int ordinalPosition = 1;
        for (Column column : table.columns()) {
            if (columnNamePattern != null && !matchesSqlLikePattern(column.name(), columnNamePattern)) continue;
            rows.add(List.of(
                    "tinydb", "public", table.tableName(), column.name(),
                    TypeMapping.toJdbcType(column.type()),          // §26 -- DATA_TYPE, an int, java.sql.Types constant
                    TypeMapping.toJdbcTypeName(column.type()),      // TYPE_NAME
                    column.nullable() ? "YES" : "NO",                // IS_NULLABLE
                    ordinalPosition++));                              // ORDINAL_POSITION
        }
    }
    return MetadataResultSets.of(
            List.of("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "COLUMN_NAME", "DATA_TYPE", "TYPE_NAME", "IS_NULLABLE", "ORDINAL_POSITION"),
            rows);
}
```

This is the single method that makes DBeaver's "expand a table to see its columns" interaction work at all — and notice it is built **entirely** from `Catalog.listTables()`/`TableSchema.columns()` (companion Query Engine guide, §9 and §33), the exact same `Catalog` every SQL statement already reads schema from. There is no second, parallel metadata store to keep in sync — a `CREATE TABLE` executed through §20's `Statement.execute` updates the *same* catalog `getColumns` reads from, so a newly-created table's columns are correctly visible to DBeaver the next time its schema tree refreshes, with zero additional wiring.

---

# 32. Implementing getPrimaryKeys() and Other Metadata DBeaver Expects

```java
@Override
public ResultSet getPrimaryKeys(String catalog, String schema, String table) throws SQLException {
    // The companion Query Engine guide's TableSchema (§9) never modeled a primary key at all -- there is
    // genuinely nothing to report here yet. Returning an EMPTY result set (not null, not an exception) is
    // the correct, specification-honored way to say "this table has no primary key," and is exactly what a
    // real table without one would also report -- DBeaver handles this gracefully, showing no key icon.
    return MetadataResultSets.of(
            List.of("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "COLUMN_NAME", "KEY_SEQ", "PK_NAME"),
            List.of());
}

@Override
public ResultSet getIndexInfo(String catalog, String schema, String table, boolean unique, boolean approximate) throws SQLException {
    return MetadataResultSets.of(
            List.of("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "INDEX_NAME", "COLUMN_NAME"), List.of()); // same honest "none yet"
}
```

Returning a correctly-shaped, empty `ResultSet` — never `null`, never throwing `SQLFeatureNotSupportedException` — for metadata TinyDB genuinely doesn't have yet is the pattern this entire Part follows: DBeaver calls a long, fixed list of `DatabaseMetaData` methods unconditionally while building its schema tree, and a method that throws instead of returning an empty result can abort that entire tree-building pass, turning "this table has no primary key" into "this table failed to load at all."

---

# 33. Capability Flags: The supportsXXX() Methods DBeaver Checks Before Using a Feature

Before attempting a feature, a well-behaved JDBC client asks the driver whether it's supported at all — DBeaver checks dozens of these before deciding, for instance, whether to offer transaction-related UI at all, or whether to attempt a certain kind of metadata query:

```java
@Override public boolean supportsTransactions() { return true; }                    // §17-§18 -- real, WAL-backed
@Override public boolean supportsBatchUpdates() { return false; }                    // honest -- §42 names this as future work
@Override public boolean supportsMultipleResultSets() { return false; }
@Override public boolean supportsStoredProcedures() { return false; }
@Override public boolean supportsSubqueries() { return false; }                      // matches the companion guide's stated grammar limits, §8 there
@Override public boolean supportsOuterJoins() { return false; }                       // no JOIN support at all, same guide
@Override public boolean nullsAreSortedAtEnd() { return true; }                       // an honest, stated ordering convention
@Override public int getDatabaseMajorVersion() { return 1; }
@Override public int getJDBCMajorVersion() { return 4; }
```

Answering `false` honestly for a feature that genuinely isn't supported — rather than answering `true` and letting the actual attempt fail later with a confusing error — is what lets DBeaver correctly disable or hide UI for features this driver can't back, instead of presenting an option that predictably breaks the moment a user clicks it.

---

# 34. Mapping TinyDB's Internal Errors to java.sql.SQLException

Every internal exception this driver's own code can encounter — a parse error from the companion Query Engine guide's `SqlLexer`/`SqlParser`, an `IOException` from the companion Storage Engine guide's `FileStorageEngine`, an `IllegalStateException` from a misused transaction — must reach the JDBC caller as a `SQLException`, because that's the *only* checked exception type the `java.sql.*` interfaces declare. Silently swallowing an internal error, or letting a `RuntimeException` propagate unwrapped, both violate the contract every JDBC client is written against.

```java
public final class SqlExceptionMapper {

    public static SQLException map(Exception internal) {
        if (internal instanceof SqlParseException e) {
            return new SQLSyntaxErrorException(e.getMessage(), e);      // a real, specific SQLException subclass
        }
        if (internal instanceof IOException e) {
            return new SQLException("Storage I/O error: " + e.getMessage(), "58030", e); // SQLSTATE class 58 -- system error
        }
        if (internal instanceof IllegalArgumentException e) {
            return new SQLException(e.getMessage(), "42000", e);         // SQLSTATE class 42 -- syntax/access rule violation
        }
        return new SQLException("Unexpected error: " + internal.getMessage(), internal);
    }
}
```

Using a **specific** `SQLException` subclass (`SQLSyntaxErrorException`) where one genuinely fits, and a meaningful **SQLSTATE** code (the second constructor argument) even when it doesn't, is what lets a client distinguish "you wrote bad SQL" from "the disk failed" programmatically — DBeaver surfaces both the message and, in its error details view, the SQLSTATE, and a driver that only ever throws a bare `new SQLException(message)` with no state code makes every single error look identical to any tooling built to react differently to different failure classes.

---

# 35. Packaging the Driver as a JAR DBeaver Can Load

```text
tinydb-jdbc-1.0.jar
├── com/example/tinydb/jdbc/*.class          (this guide)
├── com/example/tinydb/query/*.class          (companion Query Engine guide, bundled in)
├── com/example/tinydb/storage/*.class        (companion Storage Engine guide, bundled in)
└── META-INF/services/java.sql.Driver         (§14 -- one line: com.example.tinydb.jdbc.TinyDbDriver)
```

The JAR must contain **every** class from all three guides, not just this one's driver classes — `TinyDbDriver.connect(...)` (§15) constructs a real `FileStorageEngine`/`BasicQueryEngine` at runtime, so those classes must be present on whatever classpath DBeaver loads the driver from, exactly the same as any Java library shipping with its own transitive dependencies bundled in ("an uber/fat JAR," in the common tooling term) rather than expecting the consumer to separately provide them.

---

# 36. Connecting From DBeaver, Step by Step

1. **Database -> Driver Manager -> New Driver.** Set Driver Name to `TinyDB`; Class Name to `com.example.tinydb.jdbc.TinyDbDriver` (DBeaver's own driver-manager dialog does not strictly need `META-INF/services` — it lets a user specify the class directly, though §14's mechanism means it isn't required either way); URL Template to `jdbc:tinydb:{file}`.
2. **Libraries tab -> Add File** -> select `tinydb-jdbc-1.0.jar` (§35).
3. **Database -> New Database Connection -> select the "TinyDB" driver** just registered.
4. In the connection settings, provide the **Path** — a filesystem directory where TinyDB's data files should live (or already do) — which DBeaver substitutes into the URL template from step 1, producing exactly the `jdbc:tinydb:/path/to/data` (§12) `TinyDbDriver.connect(...)` (§15) expects.
5. **Test Connection.** DBeaver calls `DriverManager.getConnection(url)`, which resolves to `TinyDbDriver` (§14's `ServiceLoader` discovery, or the explicit class name from step 1), which calls `TinyDbInstance.open(...)` (§13) — including running `recover()` if the directory already contains a prior database.
6. On success, DBeaver immediately calls `Connection.getMetaData()` (§16) and walks `getCatalogs`/`getSchemas`/`getTables`/`getColumns` (§29-§31) to populate the schema tree on the left — this is the exact moment §28-§33's work either pays off or doesn't.
7. Open a SQL editor tab against the new connection and run any statement the companion Query Engine guide's grammar supports — it flows through §19-§20's `Statement` dispatch exactly as traced in §37.

---

# 37. Full Worked Example: A Complete Round Trip From DBeaver to TinyDB and Back

```text
1.  User runs, in DBeaver's SQL editor: SELECT * FROM employees WHERE age > 30
2.  DBeaver calls Statement.execute(sql) on its TinyDbStatement                          (§19-§20)
3.  dispatchByStatementKind() classifies it as a query (starts with SELECT)              (§20)
4.  executeQuery(sql) -> connection.executeInternal(sql, true)                            (§16)
5.  autoCommit is true (DBeaver's default for an ad-hoc query) -> queryEngine.doQuery(sql) (§17, companion guide §26)
6.  BasicQueryEngine.doQuery: begins a transaction, plans, opens a Scan, walks every row   (companion guide §26)
7.  QueryResult(rows, ...) returned up to TinyDbStatement
8.  new TinyDbResultSet(result.rows(), this)                                              (§24)
9.  DBeaver calls rs.getMetaData() to learn column names/types before rendering the grid    (§27, §26)
10. DBeaver loops rs.next() / rs.getObject(columnIndex) to populate the results grid         (§24-§25)
11. User expands the "employees" table in the schema tree (independently, at any time)
       -> DatabaseMetaData.getColumns("tinydb", "public", "employees", null)                (§31)
       -> reads Catalog.getTable("employees").columns() -- the SAME catalog step 6's plan used
```

Step 11 happening **independently of, and consistently with,** steps 1-10 is the concrete payoff of §31's design choice: the schema tree and a running query both read the identical `Catalog`, so nothing about the driver can ever show DBeaver's schema browser and a query's actual behavior disagreeing about what columns a table has.

---

# 38. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Adapter** | Every class in this guide (§15-§33) | The entire driver is an adapter — translating the fixed `java.sql.*` contract on one side into calls against the companion guides' own, differently-shaped APIs on the other. |
| **Facade** | `TinyDbInstance` (§13) | One class hides the wiring of `FileStorageEngine`, `PersistentCatalog`, `BasicPlanner`, `TransactionManager`, and `BasicQueryEngine` behind a single `open(path)` call. |
| **Singleton (scoped)** | `TinyDbDriver.OPEN_INSTANCES` (§15) | Exactly one `TinyDbInstance` per data directory, shared across every `Connection` opened against it — a per-key singleton, not a single global one. |
| **Iterator** | `TinyDbResultSet.next()` (§24) | The standard cursor-based row-at-a-time access pattern every `ResultSet` implementation is built around. |
| **Strategy (reused)** | `SqlExceptionMapper` (§34) | A dedicated mapping step, kept separate from every call site, exactly like the companion guides' own pluggable-algorithm sections keep a decision isolated from its use. |

---

# 39. SOLID Principles Applied

- **Single Responsibility**: `SqlParameterBinder` (§22) only rewrites placeholders; `TypeMapping` (§26) only converts between type systems; `SqlExceptionMapper` (§34) only translates exceptions. None of them know how to execute a statement.
- **Open/Closed**: adding a new `PreparedStatement` setter (§23) for a type this guide didn't cover requires no change to `SqlParameterBinder.bind` itself — only a new `case` in `literalFor`, and a new coercing setter method.
- **Liskov Substitution**: `TinyDbPreparedStatement extends TinyDbStatement` (§21) and is fully usable wherever a plain `Statement` is expected — DBeaver's own internal code that only knows about `Statement` never needs special-casing for the prepared variant.
- **Interface Segregation**: this guide never introduces one giant "TinyDbJdbcHelper" god-class — `TransactionManager` (§9), `SqlParameterBinder` (§22), and `TypeMapping` (§26) each expose exactly the narrow surface their one job needs.
- **Dependency Inversion**: `TinyDbConnection` (§16) depends on `TinyDbInstance`'s exposed `BasicQueryEngine`/`BasicPlanner`/`TransactionManager`/`Catalog` — all types already defined by the companion guides' own public APIs — never on any storage or query-execution detail beneath those seams.

---

# 40. Common Mistakes When Building This Yourself

- **Concatenating parameter values directly into SQL text instead of escaping them** (§22) — turns `PreparedStatement`, whose entire purpose is safety against exactly this, into no safer than a raw string-built query.
- **Forgetting `wasNull()` must reflect the MOST RECENT column access**, not the most recent non-null one (§25) — a caller reading column 1 (null) then column 2 (not null) and then calling `wasNull()` must get `false`, reflecting column 2, not a stale `true` left over from column 1.
- **Opening a second, independent `TinyDbInstance` for the same data directory** (§13, §15) — two uncoordinated buffer pools and WAL managers writing to the same files is exactly the concurrency hazard the companion Storage Engine guide's own locking discipline assumes never happens.
- **Returning `null` from a `DatabaseMetaData` method instead of an empty, correctly-shaped `ResultSet`** (§32) — a real client iterating the result with `while (rs.next())` throws a `NullPointerException` on a `null`, where an empty result set correctly and silently produces zero iterations.
- **Letting `Connection.close()` silently commit an open, unfinished transaction** (§18) — the JDBC specification's own stated behavior is to roll back, and a driver that commits instead can make a client's forgotten, half-finished work permanent.
- **Answering `true` from a `supportsXxx()` method for a feature that isn't actually implemented** (§33) — the failure this causes surfaces later, and in a much more confusing form, than an honest `false` that lets the client avoid the unsupported path entirely.

---

# 41. Testing Strategy

- **`TinyDbDriver`** (§15): `acceptsURL` correctly accepts every `jdbc:tinydb:...` URL and rejects everything else; `connect` returns the same underlying `TinyDbInstance` (verified via the shared `BasicQueryEngine` instance) for two connections opened against the identical, normalized data-directory path.
- **`TinyDbConnection`** transaction semantics (§17-§18): with `autoCommit=true`, each statement's effects are visible immediately; with `autoCommit=false`, effects from an uncommitted statement are invisible to a *second*, independent connection until `commit()`, and vanish entirely after `rollback()`.
- **`SqlParameterBinder`** (§22): a string parameter containing a single quote round-trips correctly; a parameter count mismatch (too few or too many `?` bound) throws `SQLException` rather than silently truncating or ignoring extras.
- **`TinyDbResultSet`** (§24-§25): `wasNull()` correctly reflects only the most recent column access; `getInt`/`getLong` on a genuinely null column return `0`, with `wasNull()` immediately after correctly reporting `true`.
- **`DatabaseMetaData.getColumns`** (§31): a table created via `Statement.execute("CREATE TABLE ...")` is immediately visible, with the correct columns and types, to a `getColumns` call on the *same* connection with no additional refresh step.
- **The literal DBeaver test** (§36): connect, browse the schema tree, run a query, edit a row via a `PreparedStatement` update, and confirm every step succeeds against a real DBeaver install, not only against this guide's own unit tests — the single most convincing proof this driver actually satisfies its own stated goal.

---

# 42. Suggested Future Enhancements

- **A server/network mode** — a second `Connection` implementation talking over a socket to a separately-running TinyDB server process (the companion Storage Engine guide's own "server mode," that guide's §47), for the case where the database genuinely needs to be shared by multiple, independent client machines rather than embedded in one.
- **Typed parameter binding**, passing `PreparedStatement` values to the planner as already-typed objects rather than re-serialized, escaped SQL text (§22's stated simplification) — avoiding a second lex/parse pass per execution and eliminating the escaping logic's attack surface entirely, rather than merely making it safe.
- **Batch updates** (`addBatch`/`executeBatch`) — genuinely useful for bulk `INSERT`s, and honestly reported as unsupported for now (§33).
- **Primary key and index metadata** (§32) — once the companion Query Engine guide's `TableSchema` grows a real primary-key/index concept, `getPrimaryKeys`/`getIndexInfo` should be revisited to report it instead of the current, honest "none yet."
- **Streaming result sets** for a `SELECT` returning far more rows than comfortably fit in memory — `BasicQueryEngine.doQuery` (companion guide, §26) already materializes every row before returning, and a truly large result would want `ResultSet.next()` (§24) pulling directly from an open `Scan` instead.
- **Savepoints**, extending `TransactionManager` (§9) with `Connection.setSavepoint()`/`rollback(Savepoint)` support, for partial rollback within one still-open transaction.

---

# 43. Final Takeaway

A JDBC driver is not a new database feature — it's a **translation layer**, and its entire job is faithfully honoring a specification it has zero say in, using pieces that were built for entirely different purposes than "satisfy `java.sql.*`." Every genuinely interesting decision in this guide was about *fidelity to that contract*: `wasNull()` reflecting the truly most recent access, `close()` rolling back rather than committing, an empty metadata result instead of `null`, a `supportsXxx()` flag answered honestly rather than optimistically. None of that required new database engineering — the query engine and the storage engine were already real and already correct. What this guide adds is the discipline of making an existing, correct system speak a protocol a tool like DBeaver already knows how to use, without quietly breaking either side's own guarantees in the process.
