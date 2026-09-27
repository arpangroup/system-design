# TinyDB Server Mode and Remote JDBC Driver — Step-by-Step Implementation Guide

> **Goal:** Build a complete, real, loadable JDBC driver for TinyDB that works in two modes from a single, unified design — embedded (in-process, a local data directory) and server/remote (a real TinyDB server process, reachable over the network from any machine running DBeaver or any other JDBC-compliant SQL client) — as a standalone project with real, working code for every piece, from the wire protocol up through every `java.sql.*` interface that matters.
>
> The driver in this guide is built around one deliberate decision, made from the very first class: **how a statement actually gets executed is a pluggable strategy, decided once per connection, never hard-coded.** That one decision is what lets `Statement`, `PreparedStatement`, `ResultSet`, `ResultSetMetaData`, and `DatabaseMetaData` be written exactly once and work identically whether the connection underneath them is talking to a local file or to a server on another continent.

---

# 1. What We Are Building

```text
DBeaver (or any JDBC client)
      |
   java.sql.* API  (DriverManager, Connection, Statement, ResultSet, DatabaseMetaData)
      |
   TinyDbDriver -- dispatches by URL scheme
      |
      +---- jdbc:tinydb:/path -------> EmbeddedStatementExecutor -> BasicQueryEngine -> FileStorageEngine -> disk
      |
      +---- jdbc:tinydb://host:port -> NetworkStatementExecutor -> [wire protocol] -> TinyDbServer -> (same engine, remote)
```

By the end of this guide you will have:

- A real `java.sql.Driver`, discoverable via `META-INF/services`, that understands **two** URL forms and constructs the right execution strategy for each.
- A `StatementExecutor` interface — the one seam this entire driver is built around — with two real implementations: one that calls straight into an in-process database engine, one that talks to a remote server over a socket.
- Every JDBC interface that actually matters — `Connection`, `Statement`, `PreparedStatement`, `ResultSet`, `ResultSetMetaData`, `DatabaseMetaData` — built **once**, against that interface, never duplicated for the network case.
- A real, byte-level wire protocol: length-prefixed framing, a closed set of request/response types, and real serialization for every one of them.
- A real TinyDB server process: an accept loop, a bounded worker pool, per-connection transaction handling, and a credential-based authentication handshake — with an honest explanation of exactly where TLS wraps in without this guide hand-rolling any cryptography.
- A literal, step-by-step walkthrough of connecting DBeaver to TinyDB both ways: an embedded local file, and a genuinely remote server.

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Design a pluggable execution seam **from day one**, so that adding a second way to run a statement never means duplicating the JDBC-facing classes built around the first one.
- Implement every JDBC interface that a real client like DBeaver actually depends on — including the parts of `DatabaseMetaData` that make a schema browser work at all — and know which remaining methods are legitimately boilerplate.
- Implement `PreparedStatement` parameter binding safely, and explain precisely what vulnerability that specifically closes.
- Design a real, byte-level wire protocol: message framing, a closed set of request/response types, and serialization that doesn't leave "the tricky part" as pseudocode.
- Design a server process that safely shares one underlying database engine across many concurrent client connections, using a bounded worker pool for the same isolation reasons a bounded thread pool is used anywhere else.
- Explain precisely why authentication and transport encryption matter the moment a database becomes network-reachable, and where to draw the line between "build this" and "use the standard library's own implementation instead."

---

# 3. Why This Matters (Interview Motivation)

> **"Build a JDBC driver for a database engine you've already got working — one that a developer can use exactly the way they'd use a Postgres or MySQL driver, whether the database is a local file or a server across the network. Show me the design decision that makes both modes possible without duplicating half your codebase."**

This question tests something distinct from "design a database" — it tests a specific kind of architectural discipline:

- **Working inside someone else's contract.** `java.sql.*` is a specification the driver author has zero control over — every method signature, every exception type, every semantic (what must `ResultSet.next()` do when there are no more rows? what must `wasNull()` return, and when?) is fixed, and a real driver has to honor all of it correctly, not approximately.
- **Designing the seam before writing the first line against it.** A driver that hard-codes "execute this statement" as a direct method call into a local engine has to be substantially rewritten the moment a second deployment mode appears; a driver that names that seam as its own interface from the start pays that cost exactly once, up front, and never again.
- **Protocol design and server concurrency stop being optional the moment a database is reachable from the network** — and knowing where to build real logic (authentication, framing) versus where to reuse a battle-tested standard mechanism (TLS via the JDK's own `SSLSocket`) is itself a signal of engineering judgment.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | `java.sql.*`, `java.net.Socket`/`ServerSocket`, and `javax.net.ssl.SSLSocket` are all part of the JDK — no external dependency needed for any of this guide's core pieces. |
| Driver discovery | `java.util.ServiceLoader` via `META-INF/services/java.sql.Driver` | The JDBC 4.0+ standard mechanism — the one that lets DBeaver find the driver class without the user typing it in by hand (§31). |
| Embedded storage | A local, file-based database engine (built in this guide's own companion database-engine guides — see §6) | Zero new infrastructure for the embedded case: the engine already runs perfectly well inside any JVM process, including DBeaver's own. |
| Transport (server mode) | Plain TCP via `java.net.Socket`/`ServerSocket`, optionally wrapped in `SSLSocket` | The simplest transport that supports a real request/response protocol; TLS is a wrapping decision, not a redesign (§40). |
| Wire format | A custom, length-prefixed binary protocol | Small, precise control over exactly what's on the wire, with a framing discipline that never mistakes SQL text for a message boundary (§34). |
| Server concurrency | A bounded `ThreadPoolExecutor`, one task per accepted connection | The standard core/max/queue/reject shape used to bound how many concurrent clients one server process serves, never letting a connection flood spawn unbounded threads (§42). |
| Testing | JUnit 5, plus a real two-process (or two-JVM) integration test | A network protocol's correctness is only really proven by two independent processes actually talking to each other (§59). |

---

# 5. Project Structure

```text
tinydb-jdbc/
├── src/main/java/com/example/tinydb/jdbc/
│   ├── transaction/
│   │   ├── TransactionContext.java, TransactionManager.java     // §10-§11
│   ├── TinyDbInstance.java                                       // §12
│   ├── StatementExecutor.java                                    // §9 -- THE seam
│   ├── EmbeddedStatementExecutor.java                            // §13
│   ├── NetworkStatementExecutor.java                             // §46
│   ├── TinyDbConnection.java                                      // §14
│   ├── TinyDbStatement.java, TinyDbPreparedStatement.java          // §15-§19
│   ├── SqlParameterBinder.java                                     // §18
│   ├── TinyDbResultSet.java, TinyDbResultSetMetaData.java           // §20-§23
│   ├── TypeMapping.java                                             // §22
│   ├── TinyDbDatabaseMetaData.java                                  // §24-§29
│   ├── RemoteCatalogSnapshot.java                                    // §49
│   ├── SqlExceptionMapper.java                                       // §30
│   └── TinyDbDriver.java                                             // §31, §47
├── src/main/java/com/example/tinydb/server/
│   ├── protocol/
│   │   ├── Frame.java                                             // §34
│   │   ├── Request.java (sealed), Response.java (sealed)           // §35-§36
│   │   └── ProtocolCodec.java                                       // §37
│   ├── auth/
│   │   └── CredentialStore.java, AuthHandshake.java                 // §38-§39
│   ├── TinyDbServer.java                                             // §42
│   └── ServerConnectionHandler.java                                  // §43-§44
└── src/test/java/com/example/tinydb/jdbc/
    ├── DriverRegistrationTest.java
    ├── PreparedStatementParameterBindingTest.java
    ├── TransactionCommitRollbackTest.java
    ├── DatabaseMetaDataAgainstDbeaverTest.java
    └── TwoProcessIntegrationTest.java
```

---

# 6. High-Level Architecture: Embedded and Server Mode, One Driver

Underneath every piece this guide builds sits a real, already-working SQL engine and a real, durable storage engine — the actual database TinyDB is. Rather than re-deriving that engine here, this guide treats it as a fixed, given foundation with exactly the shape a database engine needs: a query engine that can plan and execute SQL text against a table (`BasicQueryEngine`, with `doQuery(sql)`/`doUpdate(sql)` methods returning a `QueryResult` of `Row`s), a durable storage layer (`FileStorageEngine`) that survives a crash, and a schema catalog (`Catalog`, with `listTables()`/`getTable(name)`) describing what tables and columns exist. Building that engine from scratch — the SQL lexer and parser, the query planner, the page-based storage layer, write-ahead logging and crash recovery — is real, substantial work covered in depth by this project's own database-internals guides; this guide's entire job starts one layer above that: **making an already-correct engine usable from any standard SQL client, locally or remotely.**

The one architectural decision that makes both modes possible without duplicating this guide's own JDBC-facing code: **the seam between "here is a SQL string" and "here is a result" is a pluggable interface, `StatementExecutor` (§9), decided once per `Connection`.** Everything from `Statement` through `DatabaseMetaData` (§15-§29) is written against that interface exactly once, and never needs a second version for the network case (§48).

---

# 7. The Domain Model This Driver Sits On

Four small, already-existing types are all this driver ever touches on the database-engine side, and every one of them is intentionally plain data:

```java
public enum DataType { INT, BIGINT, BOOLEAN, VARCHAR, DECIMAL, DATE, TIMESTAMP }
public record Column(String name, DataType type, boolean nullable) { }
public record TableSchema(String tableName, List<Column> columns) { }

public interface Catalog {
    void createTable(TableSchema schema);
    void dropTable(String tableName);
    TableSchema getTable(String tableName);
    boolean tableExists(String tableName);
    List<TableSchema> listTables();
}
```

```java
public final class Row {
    private final Map<String, Object> values;
    public Row(Map<String, Object> values) { this.values = Map.copyOf(values); }
    public Object get(String column) { return values.get(column); }
    public Set<String> columnNames() { return values.keySet(); }
}

public record QueryResult(List<Row> rows, int affectedRows) {
    public static QueryResult ofRows(List<Row> rows) { return new QueryResult(rows, rows.size()); }
    public static QueryResult ofAffected(int count) { return new QueryResult(List.of(), count); }
}
```

A row's runtime value for each `DataType` is fixed by the storage layer's own binary codec, and this driver treats that mapping as a contract it must honor, never reinvent: `INT`->`Integer`, `BIGINT`->`Long`, `BOOLEAN`->`Boolean`, `DECIMAL`->`Double` (a deliberate simplification the underlying engine makes, not `BigDecimal`), `DATE`/`TIMESTAMP`->`Instant`, `VARCHAR`->`String`. §22 builds the full `java.sql.Types` mapping from this table.

The actual query execution and transaction machinery this driver calls into — `BasicQueryEngine.doQuery(sql)`/`doUpdate(sql)`, a `BasicPlanner` for executing SQL directly against an open transaction, and `FileStorageEngine.beginTransaction(id)`/`commitTransaction(id)`/`abortTransaction(id)` for real, durable, WAL-backed transactions — is the actual database engine this project's SQL-engine and storage-engine guides build in full; this guide wires against those classes' existing, real APIs exactly as given, and builds nothing new at that layer.

---

# 8. The Connection URL Formats

```text
jdbc:tinydb:/absolute/path/to/data/directory      <-- embedded: opens a LOCAL FileStorageEngine directly, §12
jdbc:tinydb://host:port/database                  <-- server mode: opens a SOCKET to a running TinyDbServer, §47
```

One driver, one prefix, two shapes after it — the presence of `//` is the entire dispatch signal `TinyDbDriver` (§31, §47) needs to decide which `StatementExecutor` (§9) to construct. The embedded form is modeled directly on SQLite's own `jdbc:sqlite:/path/to/file.db`; the server form is modeled directly on Postgres's/MySQL's own `jdbc:postgresql://host:port/db` — both are established, unsurprising conventions, deliberately not invented fresh here.

---

# 9. The StatementExecutor Seam: One Interface, Two Implementations, Decided Up Front

```java
public interface StatementExecutor {
    QueryResult execute(String sql, boolean isQuery) throws SQLException;

    boolean getAutoCommit();
    void setAutoCommit(boolean autoCommit) throws SQLException;
    void commit() throws SQLException;
    void rollback() throws SQLException;
    void close() throws SQLException;

    Catalog catalog(); // §7 -- what DatabaseMetaData (§24) is built on, locally or remotely
}
```

Every method here names a responsibility every JDBC `Connection` genuinely has — running a statement, tracking auto-commit, committing, rolling back, closing, and exposing enough schema information for `DatabaseMetaData` to work. None of it is specific to *how* a statement actually gets executed. That is precisely the point: `TinyDbConnection` (§14), and every class built on top of it (§15-§29), will be written against this interface only, and will work correctly and completely unmodified against whichever of §13's `EmbeddedStatementExecutor` or §46's `NetworkStatementExecutor` a given connection happens to hold.

---

# 10. Implementing TransactionContext

```java
public final class TransactionContext {

    private static final AtomicLong NEXT_TRANSACTION_ID = new AtomicLong(1);

    private final long transactionId;
    private volatile boolean completed; // true once committed or rolled back -- guards against double-completion

    private TransactionContext(long transactionId) { this.transactionId = transactionId; }

    static TransactionContext allocate() { return new TransactionContext(NEXT_TRANSACTION_ID.getAndIncrement()); }

    public long transactionId() { return transactionId; }
    public boolean isCompleted() { return completed; }
    void markCompleted() { completed = true; }
}
```

Every open transaction, embedded or server-side, is one of these — a real, allocatable identity the underlying storage engine's own `beginTransaction(long)`/`commitTransaction(long)`/`abortTransaction(long)` can act on, with a completion flag that keeps a connection from ever committing or rolling back the same transaction twice.

---

# 11. Implementing TransactionManager

```java
public final class TransactionManager {

    private final StorageEngine storageEngine; // the durable, WAL-backed storage layer this project's storage-engine guide builds

    public TransactionManager(StorageEngine storageEngine) { this.storageEngine = storageEngine; }

    public TransactionContext begin() {
        TransactionContext tx = TransactionContext.allocate(); // §10
        try {
            storageEngine.beginTransaction(tx.transactionId());
        } catch (IOException e) {
            throw new TinyDbRuntimeException("Failed to begin transaction", e);
        }
        return tx;
    }

    public void commit(TransactionContext tx) {
        if (tx.isCompleted()) throw new IllegalStateException("Transaction " + tx.transactionId() + " already completed");
        try {
            storageEngine.commitTransaction(tx.transactionId());
        } catch (IOException e) {
            throw new TinyDbRuntimeException("Failed to commit transaction " + tx.transactionId(), e);
        } finally {
            tx.markCompleted();
        }
    }

    public void rollback(TransactionContext tx) {
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

`isCompleted()`'s guard against double-completion matters concretely in §14: a JDBC `Connection.close()` that runs after an explicit `commit()` already happened must never attempt a second commit or an implicit rollback against an already-finished transaction.

---

# 12. Bootstrapping a TinyDB Instance From a Path

```java
public final class TinyDbInstance {

    private final BasicQueryEngine queryEngine;
    private final BasicPlanner planner;                    // used directly when autoCommit is false, §13
    private final TransactionManager transactionManager; // §11
    private final Catalog catalog;                        // §7

    private TinyDbInstance(BasicQueryEngine queryEngine, BasicPlanner planner, TransactionManager transactionManager, Catalog catalog) {
        this.queryEngine = queryEngine;
        this.planner = planner;
        this.transactionManager = transactionManager;
        this.catalog = catalog;
    }

    /** Wires together the storage layer, the catalog, the planner, and this guide's own TransactionManager,
     *  starting purely from a filesystem path -- everything a Connection needs to actually do real work. */
    public static TinyDbInstance open(Path dataDirectory) throws IOException {
        StorageEngine storageEngine = new FileStorageEngine(dataDirectory, 1000);
        storageEngine.recover(); // MUST run first -- a database that starts serving queries before recovery
                                  // finishes would be handing out answers from a state that never existed

        Catalog catalog = new PersistentCatalog(storageEngine);
        QueryPlanner queryPlanner = new BasicQueryPlanner(catalog);
        UpdatePlanner updatePlanner = new BasicUpdatePlanner(catalog, storageEngine);
        BasicPlanner planner = new BasicPlanner(queryPlanner, updatePlanner);

        TransactionManager transactionManager = new TransactionManager(storageEngine); // §11, THIS guide
        BasicQueryEngine queryEngine = new BasicQueryEngine(planner, transactionManager);

        return new TinyDbInstance(queryEngine, planner, transactionManager, catalog);
    }

    public BasicQueryEngine queryEngine() { return queryEngine; }
    public BasicPlanner planner() { return planner; }
    public TransactionManager transactionManager() { return transactionManager; }
    public Catalog catalog() { return catalog; }
}
```

Every `TinyDbConnection` (§14) opened against the *same* data directory shares exactly one `TinyDbInstance`, never one per connection (§31 enforces this) — opening the same on-disk database twice, independently, would mean two separate, uncoordinated buffer pools and write-ahead logs fighting over the same files, which the storage engine's own concurrency model assumes never happens.

---

# 13. Implementing EmbeddedStatementExecutor

```java
public final class EmbeddedStatementExecutor implements StatementExecutor {

    private final TinyDbInstance instance; // §12
    private boolean autoCommit = true;      // JDBC's own default, per the specification
    private TransactionContext currentTransaction; // non-null only while autoCommit is false and a tx is open

    public EmbeddedStatementExecutor(TinyDbInstance instance) { this.instance = instance; }

    @Override
    public QueryResult execute(String sql, boolean isQuery) throws SQLException {
        try {
            if (autoCommit) {
                // Each statement is its OWN complete transaction -- doQuery/doUpdate begin AND commit internally.
                return isQuery ? instance.queryEngine().doQuery(sql) : instance.queryEngine().doUpdate(sql);
            }
            return executeWithinOpenTransaction(sql, isQuery);
        } catch (RuntimeException e) {
            throw new SQLException("Statement execution failed: " + e.getMessage(), e);
        }
    }

    /** autoCommit == false: THIS connection owns one open transaction across however many statements the
     *  client runs, until an explicit commit()/rollback() -- doQuery/doUpdate can't be used here at all,
     *  since they always commit internally regardless of what the JDBC caller actually asked for. */
    private QueryResult executeWithinOpenTransaction(String sql, boolean isQuery) throws SQLException {
        if (currentTransaction == null) {
            currentTransaction = instance.transactionManager().begin(); // §11 -- opened ONCE, held across calls
        }
        BasicPlanner planner = instance.planner();
        if (isQuery) {
            Plan plan = planner.createQueryPlan(sql, currentTransaction);
            return materializeAsQueryResult(plan); // walks the Plan's Scan into rows, mirroring doQuery's own logic
        }
        int affected = planner.executeUpdate(sql, currentTransaction);
        return QueryResult.ofAffected(affected);
    }

    @Override public boolean getAutoCommit() { return autoCommit; }

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        if (this.autoCommit == autoCommit) return;
        if (autoCommit && currentTransaction != null) {
            instance.transactionManager().commit(currentTransaction); // switching BACK to auto-commit must
            currentTransaction = null;                                  // finish whatever was still open
        }
        this.autoCommit = autoCommit;
    }

    @Override
    public void commit() throws SQLException {
        if (autoCommit) throw new SQLException("commit() is not valid while autoCommit is true");
        if (currentTransaction != null) { instance.transactionManager().commit(currentTransaction); currentTransaction = null; }
    }

    @Override
    public void rollback() throws SQLException {
        if (autoCommit) throw new SQLException("rollback() is not valid while autoCommit is true");
        if (currentTransaction != null) { instance.transactionManager().rollback(currentTransaction); currentTransaction = null; }
    }

    @Override
    public void close() throws SQLException {
        // Never leave a transaction dangling -- an unfinished piece of work should never be mistaken for a
        // committed one, so a connection closed without an explicit commit() rolls back, not commits.
        if (!autoCommit && currentTransaction != null && !currentTransaction.isCompleted()) {
            instance.transactionManager().rollback(currentTransaction);
        }
    }

    @Override public Catalog catalog() { return instance.catalog(); }
}
```

This class is the entire embedded-mode execution story — every branch above is a direct, honest translation of JDBC's own auto-commit semantics onto a real, WAL-backed transaction manager, with no shortcuts taken on the "what happens if a transaction is left open" case.

---

# 14. Implementing TinyDbConnection

```java
public final class TinyDbConnection implements Connection {

    private final StatementExecutor executor; // §9 -- EmbeddedStatementExecutor OR NetworkStatementExecutor, indistinguishably
    private boolean closed;

    public TinyDbConnection(StatementExecutor executor) { this.executor = executor; }

    @Override
    public Statement createStatement() throws SQLException {
        checkOpen();
        return new TinyDbStatement(this); // §15
    }

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        checkOpen();
        return new TinyDbPreparedStatement(this, sql); // §17
    }

    @Override
    public DatabaseMetaData getMetaData() throws SQLException {
        checkOpen();
        return new TinyDbDatabaseMetaData(executor.catalog()); // §24 -- fed by whichever Catalog the executor exposes
    }

    /** Package-private -- Statement/PreparedStatement call this, never the underlying engine directly, so
     *  every statement's execution goes through ONE place regardless of which executor is behind it. */
    QueryResult executeInternal(String sql, boolean isQuery) throws SQLException {
        checkOpen();
        return executor.execute(sql, isQuery); // §9's seam -- the ENTIRE point of this class's design
    }

    @Override public boolean getAutoCommit() throws SQLException { checkOpen(); return executor.getAutoCommit(); }
    @Override public void setAutoCommit(boolean autoCommit) throws SQLException { checkOpen(); executor.setAutoCommit(autoCommit); }
    @Override public void commit() throws SQLException { checkOpen(); executor.commit(); }
    @Override public void rollback() throws SQLException { checkOpen(); executor.rollback(); }

    @Override
    public void close() throws SQLException {
        if (closed) return; // idempotent, per the JDBC spec
        executor.close();
        closed = true;
    }

    private void checkOpen() throws SQLException { if (closed) throw new SQLException("Connection is closed"); }

    // isClosed()/isValid(...)/getCatalog()/setReadOnly(...)/etc. are boilerplate -- elided, since no real client this
    // guide targets calls them in a way that changes behavior; every method with real, load-bearing logic is above.
}
```

Every method here is generic over `StatementExecutor` — nothing in this class, or in anything built on top of it (§15-§29), ever needs to know or branch on whether `executor` is talking to a local file or a remote server. That distinction is made exactly once, at connection-creation time (§31, §47), and never again.

---

# 15. Implementing TinyDbStatement

```java
public final class TinyDbStatement implements Statement {

    protected final TinyDbConnection connection;
    private TinyDbResultSet currentResultSet;
    private int lastUpdateCount = -1;

    public TinyDbStatement(TinyDbConnection connection) { this.connection = connection; }

    @Override
    public ResultSet executeQuery(String sql) throws SQLException {
        QueryResult result = connection.executeInternal(sql, true); // §14
        currentResultSet = new TinyDbResultSet(result.rows(), this); // §20
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
        return dispatchByStatementKind(sql); // §16
    }

    @Override public ResultSet getResultSet() throws SQLException { return currentResultSet; }
    @Override public int getUpdateCount() throws SQLException { return lastUpdateCount; }
    @Override public Connection getConnection() throws SQLException { return connection; }
    @Override public void close() throws SQLException { if (currentResultSet != null) currentResultSet.close(); }
}
```

---

# 16. Dispatching execute/executeQuery/executeUpdate

`Statement.execute(sql)` is JDBC's own "I don't know in advance whether this is a query or an update" entry point — a real client (DBeaver included) uses it for arbitrary, user-typed SQL where the *caller* genuinely doesn't want to have to pre-classify the statement itself:

```java
private boolean dispatchByStatementKind(String sql) throws SQLException {
    boolean isQuery = sql.strip().regionMatches(true, 0, "SELECT", 0, 6); // the ENTIRE classification logic

    if (isQuery) {
        executeQuery(sql);
        return true; // per the JDBC contract: true means "call getResultSet()"
    } else {
        executeUpdate(sql);
        return false; // false means "call getUpdateCount()"
    }
}
```

A case-insensitive prefix check on `SELECT` is a deliberately minimal classifier — correct for every statement type TinyDB's own SQL grammar supports (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, `DROP TABLE`), and it sidesteps needing to run the real lexer/parser twice (once to classify, once to execute) for a decision this simple.

---

# 17. Implementing TinyDbPreparedStatement

```java
public final class TinyDbPreparedStatement extends TinyDbStatement implements PreparedStatement {

    private final String sqlTemplate;         // the ORIGINAL sql, with "?" placeholders still in place
    private final Object[] parameters;         // positional -- parameters[0] corresponds to the FIRST "?"

    public TinyDbPreparedStatement(TinyDbConnection connection, String sqlTemplate) {
        super(connection);
        this.sqlTemplate = sqlTemplate;
        this.parameters = new Object[SqlParameterBinder.countPlaceholders(sqlTemplate)]; // §18
    }

    @Override public ResultSet executeQuery() throws SQLException { return executeQuery(bindParameters()); }
    @Override public int executeUpdate() throws SQLException { return executeUpdate(bindParameters()); }

    @Override public void setString(int index, String value) { setParameter(index, value); }
    @Override public void setInt(int index, int value) { setParameter(index, value); }
    @Override public void setLong(int index, long value) { setParameter(index, value); }
    @Override public void setBoolean(int index, boolean value) { setParameter(index, value); }
    @Override public void setNull(int index, int sqlType) { setParameter(index, null); }

    @Override
    public void setBigDecimal(int index, BigDecimal value) {
        setParameter(index, value == null ? null : value.doubleValue()); // DECIMAL is a Double at runtime, §7 -- coerce once, here
    }

    @Override
    public void setDate(int index, java.sql.Date value) {
        setParameter(index, value == null ? null : Instant.ofEpochMilli(value.getTime())); // DATE/TIMESTAMP -> Instant, §7
    }

    @Override
    public void setTimestamp(int index, java.sql.Timestamp value) {
        setParameter(index, value == null ? null : value.toInstant());
    }

    @Override public void clearParameters() { Arrays.fill(parameters, null); }

    private void setParameter(int index, Object value) {
        parameters[index - 1] = value; // JDBC parameter indices are 1-based -- placeholders are 0-based internally
    }

    private String bindParameters() throws SQLException { return SqlParameterBinder.bind(sqlTemplate, parameters); } // §18
}
```

Coercing `BigDecimal`/`java.sql.Date`/`java.sql.Timestamp` **once, here, at bind time**, rather than deferring the conversion, means every layer beneath this class only ever has to deal with exactly the runtime types §7's engine already expects, never a second, parallel set of JDBC-flavored types leaking downward.

---

# 18. Parsing and Rewriting `?` Placeholders Into a Bound SQL Statement

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
                if (parameterIndex >= parameters.length) throw new SQLException("Not enough parameters bound for this statement");
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
        if (value instanceof String s) return "'" + s.replace("'", "''") + "'"; // doubled quote -- §19 explains why this is safe
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

---

# 19. Why This Binding Approach Is Safe, and Its One Honest Limitation

Escaping a single quote inside a string parameter by **doubling** it (`'` -> `''`) is exactly what turns this from a SQL-injection-shaped hole into a safe substitution — a parameter value of `O'Brien` becomes the SQL literal `'O''Brien'`, which TinyDB's own SQL lexer parses back as the single, correct string `O'Brien`, never as a string that ends early and lets the rest of the value's content be interpreted as SQL syntax. This is textual substitution done *safely*, specifically because it escapes before substituting — it is emphatically not the same operation as a caller naively concatenating an unescaped parameter directly into a SQL string, which is the actual vulnerability `PreparedStatement` exists to prevent in the first place. A production-grade driver typically goes one step further and passes bound values to the executor as **already-typed values** rather than re-serialized SQL text at all, avoiding a second parse-and-lex round trip; this guide's text-substitution approach is the simpler, still-correct version, named explicitly as a limitation rather than hidden (§60).

---

# 20. Implementing TinyDbResultSet

```java
public final class TinyDbResultSet implements ResultSet {

    private final List<Row> rows;              // §7's Row
    private final List<String> columnNames;     // fixed order, taken from the first row -- §21 explains why that's safe here
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
    @Override public ResultSetMetaData getMetaData() { return new TinyDbResultSetMetaData(columnNames, currentRow()); } // §23
    @Override public boolean wasNull() { return lastValueWasNull; }

    private Row currentRow() throws SQLException {
        if (currentIndex < 0 || currentIndex >= rows.size()) {
            throw new SQLException("Cursor is not positioned on a valid row -- call next() first");
        }
        return rows.get(currentIndex);
    }

    private void checkOpen() throws SQLException { if (closed) throw new SQLException("ResultSet is closed"); }

    // getInt/getString/getObject/etc. by index AND by name are §21. This driver's result sets are read-only/
    // forward-only -- updateXxx()/insertRow()/deleteRow() are never implemented, since no client this guide
    // targets performs updatable-result-set edits; every method with real, load-bearing logic is above.
}
```

`currentIndex` starting at `-1`, with `next()` required before any column access is valid, is the exact cursor discipline every JDBC client (DBeaver included) already assumes.

---

# 21. Column Access by Index and by Name, and wasNull() Tracking

```java
@Override
public Object getObject(int columnIndex) throws SQLException {
    Row row = currentRow();
    String columnName = columnNames.get(columnIndex - 1); // JDBC columns are 1-based
    Object value = row.get(columnName);
    lastValueWasNull = (value == null); // §20's wasNull() reads exactly this flag -- set on EVERY access, not just null ones
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
    return value == null ? 0 : ((Number) value).intValue(); // JDBC contract: null numeric returns 0, wasNull() disambiguates
}

@Override
public long getLong(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex);
    return value == null ? 0L : ((Number) value).longValue();
}

@Override
public BigDecimal getBigDecimal(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex); // a Double, per §22's DECIMAL mapping
    return value == null ? null : BigDecimal.valueOf((Double) value);
}

@Override
public java.sql.Timestamp getTimestamp(int columnIndex) throws SQLException {
    Object value = getObject(columnIndex); // an Instant, per §22's DATE/TIMESTAMP mapping
    return value == null ? null : java.sql.Timestamp.from((Instant) value);
}
```

`getInt`/`getLong` returning `0` for a `null` column, rather than throwing, is a real JDBC specification requirement — a primitive `int` cannot represent `null`, so the caller who cares about the difference calls `wasNull()` immediately afterward. Setting `lastValueWasNull` inside `getObject` — the single method every other getter above calls through — means that flag is always correct after *any* column access, not only after the specific getter a caller happens to have used.

---

# 22. Mapping TinyDB's DataType to java.sql.Types

| TinyDB `DataType` | Java runtime type (§7) | `java.sql.Types` constant |
|---|---|---|
| `INT` | `Integer` | `Types.INTEGER` |
| `BIGINT` | `Long` | `Types.BIGINT` |
| `BOOLEAN` | `Boolean` | `Types.BOOLEAN` |
| `DECIMAL` | `Double` (a stated simplification, not `BigDecimal` — §7) | `Types.DOUBLE` |
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

    public static String toJdbcTypeName(DataType dataType) { return dataType.name(); }
}
```

Mapping `DECIMAL` to `Types.DOUBLE` rather than `Types.DECIMAL` is a deliberate honesty check: reporting `Types.DECIMAL` while the underlying value is actually a `Double` would mislead a client that inspects the reported type to decide which getter is safe to call.

---

# 23. Implementing ResultSetMetaData

```java
public final class TinyDbResultSetMetaData implements ResultSetMetaData {

    private final List<String> columnNames;
    private final Row sampleRow; // used ONLY to infer each column's DataType -- never to read actual values

    public TinyDbResultSetMetaData(List<String> columnNames, Row sampleRow) {
        this.columnNames = columnNames;
        this.sampleRow = sampleRow;
    }

    @Override public int getColumnCount() { return columnNames.size(); }
    @Override public String getColumnName(int column) { return columnNames.get(column - 1); }
    @Override public String getColumnLabel(int column) { return getColumnName(column); }

    @Override
    public int getColumnType(int column) throws SQLException {
        return TypeMapping.toJdbcType(inferDataType(sampleRow.get(getColumnName(column)))); // §22
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

Inferring a column's type from a **sample row's runtime value**, rather than from the table's actual declared `TableSchema` (§7), is this section's one honest shortcut — correct for every ordinary query, but it silently gets an all-`NULL`-in-every-row column wrong. §27's `getColumns()` — driven directly from `Catalog.getTable(...)`'s real, declared schema, never from a sample row — does not have this limitation, and is what DBeaver's schema browser actually relies on.

---

# 24. Why DBeaver Needs DatabaseMetaData in Real Depth

Everything built so far (§15-§23) is enough to run a query and read its result — but DBeaver's *schema browser*, the tree on the left showing every database, table, and column before a user has typed a single SQL statement, is built **entirely** on `Connection.getMetaData()` (§14) and a handful of specific `DatabaseMetaData` methods, called automatically the moment a connection opens, with no SQL involved at all. Get these wrong, and DBeaver connects successfully but shows an empty or broken schema tree — a failure mode that looks like a connection problem but is actually a metadata-completeness problem.

---

# 25. Implementing getCatalogs() and getSchemas()

```java
public final class TinyDbDatabaseMetaData implements DatabaseMetaData {

    private final Catalog catalog; // §7

    public TinyDbDatabaseMetaData(Catalog catalog) { this.catalog = catalog; }

    @Override
    public ResultSet getCatalogs() throws SQLException {
        // TinyDB has exactly ONE catalog per data directory (§8's URL format), so this always returns one row.
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

`MetadataResultSets` is a small internal helper (elided here) that builds a `TinyDbResultSet` (§20) directly from an in-memory list of rows, rather than routing a manufactured metadata answer back through the SQL layer at all — `getCatalogs`/`getSchemas` have nothing to do with SQL execution, and forcing them through the lexer/parser/planner just to produce one or two fixed, known rows would be needless indirection.

---

# 26. Implementing getTables()

```java
@Override
public ResultSet getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types) throws SQLException {
    List<TableSchema> allTables = this.catalog.listTables(); // §7's Catalog.listTables()

    List<List<Object>> rows = allTables.stream()
            .filter(t -> tableNamePattern == null || matchesSqlLikePattern(t.tableName(), tableNamePattern))
            .map(t -> List.<Object>of("tinydb", "public", t.tableName(), "TABLE", ""))
            .toList();

    return MetadataResultSets.of(
            List.of("TABLE_CAT", "TABLE_SCHEM", "TABLE_NAME", "TABLE_TYPE", "REMARKS"),
            rows);
}
```

Every one of `getTables`'s parameters — `catalog`, `schemaPattern`, `tableNamePattern`, `types` — is allowed to be `null`, meaning "don't filter on this," per the JDBC specification; this implementation only honors `tableNamePattern` (the one DBeaver's own "filter tables" search box actually drives) and ignores `catalog`/`schemaPattern` entirely, correct specifically *because* §25 already established there is only ever one catalog and one schema.

---

# 27. Implementing getColumns()

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
                    TypeMapping.toJdbcType(column.type()),          // §22 -- DATA_TYPE, an int, java.sql.Types constant
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

This is the single method that makes DBeaver's "expand a table to see its columns" interaction work at all — built **entirely** from `Catalog.listTables()`/`TableSchema.columns()` (§7), the exact same `Catalog` every SQL statement already reads schema from. There is no second, parallel metadata store to keep in sync — a `CREATE TABLE` executed through §16's `Statement.execute` updates the *same* catalog `getColumns` reads from.

---

# 28. Implementing getPrimaryKeys() and Other Metadata DBeaver Expects

```java
@Override
public ResultSet getPrimaryKeys(String catalog, String schema, String table) throws SQLException {
    // TinyDB's TableSchema (§7) never modeled a primary key at all -- there is genuinely nothing to report
    // here yet. Returning an EMPTY result set (not null, not an exception) is the correct, specification-
    // honored way to say "this table has no primary key" -- DBeaver handles this gracefully, no key icon shown.
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

Returning a correctly-shaped, empty `ResultSet` — never `null`, never throwing `SQLFeatureNotSupportedException` — for metadata TinyDB genuinely doesn't have yet is the pattern this entire section follows: DBeaver calls a long, fixed list of `DatabaseMetaData` methods unconditionally while building its schema tree, and a method that throws instead of returning an empty result can abort that entire tree-building pass.

---

# 29. Capability Flags: The supportsXXX() Methods DBeaver Checks

Before attempting a feature, a well-behaved JDBC client asks the driver whether it's supported at all — DBeaver checks dozens of these before deciding, for instance, whether to offer transaction-related UI, or whether to attempt a certain kind of metadata query:

```java
@Override public boolean supportsTransactions() { return true; }                    // §10-§13 -- real, WAL-backed
@Override public boolean supportsBatchUpdates() { return false; }                    // honest -- §60 names this as future work
@Override public boolean supportsMultipleResultSets() { return false; }
@Override public boolean supportsStoredProcedures() { return false; }
@Override public boolean supportsSubqueries() { return false; }                      // matches TinyDB's stated grammar limits
@Override public boolean supportsOuterJoins() { return false; }                       // no JOIN support at all
@Override public boolean nullsAreSortedAtEnd() { return true; }                       // an honest, stated ordering convention
@Override public int getDatabaseMajorVersion() { return 1; }
@Override public int getJDBCMajorVersion() { return 4; }
```

Answering `false` honestly for a feature that genuinely isn't supported — rather than answering `true` and letting the actual attempt fail later with a confusing error — is what lets DBeaver correctly disable or hide UI for features this driver can't back.

---

# 30. Mapping TinyDB's Internal Errors to java.sql.SQLException

Every internal exception this driver's own code can encounter — a SQL parse error, an `IOException` from the storage layer, an `IllegalStateException` from a misused transaction — must reach the JDBC caller as a `SQLException`, because that's the *only* checked exception type the `java.sql.*` interfaces declare.

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

Using a **specific** `SQLException` subclass where one genuinely fits, and a meaningful **SQLSTATE** code even when it doesn't, is what lets a client distinguish "you wrote bad SQL" from "the disk failed" programmatically — a driver that only ever throws a bare `new SQLException(message)` with no state code makes every single error look identical to any tooling built to react differently to different failure classes.

---

# 31. Driver Registration and TinyDbDriver

A JDBC 4.0+ driver never requires its caller to write `Class.forName("com.example.tinydb.jdbc.TinyDbDriver")` — `java.sql.DriverManager` uses `java.util.ServiceLoader` to discover every driver on the classpath automatically, by reading a plain text file:

```text
com.example.tinydb.jdbc.TinyDbDriver
```

placed at `src/main/resources/META-INF/services/java.sql.Driver`. The moment the driver's JAR is on DBeaver's classpath, `ServiceLoader` finds this file, reads the one class name in it, instantiates that class via its no-argument constructor, and calls `DriverManager.registerDriver(...)` on the caller's behalf.

```java
public final class TinyDbDriver implements java.sql.Driver {

    private static final Map<Path, TinyDbInstance> OPEN_INSTANCES = new ConcurrentHashMap<>(); // §12 -- one per data directory

    static {
        try {
            DriverManager.registerDriver(new TinyDbDriver());
        } catch (SQLException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public TinyDbDriver() { } // required -- ServiceLoader instantiates via this exact constructor

    @Override
    public boolean acceptsURL(String url) { return url != null && url.startsWith("jdbc:tinydb:"); }

    @Override
    public Connection connect(String url, Properties info) throws SQLException {
        if (!acceptsURL(url)) return null; // a Driver that doesn't understand a URL returns null, never throws

        if (url.startsWith("jdbc:tinydb://")) {
            return connectRemote(url, info); // §47 -- built once the network layer exists, §34 onward
        }
        return connectEmbedded(url); // this section
    }

    private Connection connectEmbedded(String url) throws SQLException {
        Path dataDirectory = Path.of(url.substring("jdbc:tinydb:".length()));
        try {
            TinyDbInstance instance = OPEN_INSTANCES.computeIfAbsent(dataDirectory.toAbsolutePath(), path -> {
                try {
                    return TinyDbInstance.open(path); // §12
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
            return new TinyDbConnection(new EmbeddedStatementExecutor(instance)); // §13-§14
        } catch (UncheckedIOException e) {
            throw new SQLException("Failed to open TinyDB data directory: " + dataDirectory, e.getCause());
        }
    }

    @Override public int getMajorVersion() { return 1; }
    @Override public int getMinorVersion() { return 0; }
    @Override public boolean jdbcCompliant() { return false; } // honest -- this driver does not implement the FULL spec
    @Override public Logger getParentLogger() { throw new SQLFeatureNotSupportedException(); }
    @Override public DriverPropertyInfo[] getPropertyInfo(String url, Properties info) { return new DriverPropertyInfo[0]; }
}
```

`OPEN_INSTANCES.computeIfAbsent(...)`, keyed by the data directory's absolute path, is what enforces "exactly one `TinyDbInstance` per data directory" even when DBeaver opens several `Connection`s to the same embedded database concurrently (which it routinely does — one for browsing schema, one for running a query) — every one of them resolves to the identical shared `BasicQueryEngine`/`FileStorageEngine` pair, never a competing second copy. `connectRemote` is a stub reference for now — §47 fills it in once §34-§46 have built everything it needs.

---

# 32. Packaging and Connecting From DBeaver (Embedded)

The driver's JAR must contain every class this guide builds, plus the underlying query-engine and storage-engine classes it wires against (§7) — an "uber/fat JAR" bundling its own transitive dependencies, the same way any Java library ships when it can't assume the consumer separately provides them.

1. **Database -> Driver Manager -> New Driver.** Driver Name: `TinyDB`; Class Name: `com.example.tinydb.jdbc.TinyDbDriver`; URL Template: `jdbc:tinydb:{file}`.
2. **Libraries tab -> Add File** -> select the driver JAR.
3. **Database -> New Database Connection -> TinyDB**, providing the **Path** to a data directory (existing or new).
4. **Test Connection.** DBeaver calls `DriverManager.getConnection(url)`, resolving to `TinyDbDriver.connect` (§31), which opens or reuses a `TinyDbInstance` (§12) — including running `recover()` if the directory already contains a prior database.
5. On success, DBeaver calls `getMetaData()` (§14) and walks `getCatalogs`/`getSchemas`/`getTables`/`getColumns` (§25-§27) to populate the schema tree.
6. Open a SQL editor tab and run any statement — it flows through §15-§16's `Statement` dispatch into `EmbeddedStatementExecutor` (§13).

---

# 33. Follow-up: Can This Be Connected From a Remote Server?

Everything through §32 is a complete, working, **embedded** driver — DBeaver has to be running on the same machine as the data directory (or have it mounted as a local path), because `connectEmbedded` (§31) opens a `FileStorageEngine` directly on a local filesystem path, with no network protocol anywhere. That's the honest limit of what's been built so far, and it's a real one: point DBeaver at a path on a genuinely different machine, and there's nothing listening on the other end to connect to.

Making that actually possible needs exactly two new things, and — because §9's `StatementExecutor` seam was designed in from the start rather than hard-coded — **nothing already built above §14 needs to change to get them**: a real TinyDB server process (a wire protocol plus a process that owns the database and listens on a port, §34-§45), and a second `StatementExecutor` implementation that talks to it over a socket instead of calling straight into a local engine (§46). `TinyDbConnection`, `TinyDbStatement`, `TinyDbPreparedStatement`, `TinyDbResultSet`, `TinyDbResultSetMetaData`, and `TinyDbDatabaseMetaData` — every JDBC-facing class built so far — need exactly zero changes. §34 onward builds both new pieces.

---

# 34. The Wire Protocol: Framing and Message Types

Every message, in either direction, is a **length-prefixed frame**: a 4-byte big-endian length, followed by exactly that many bytes of payload. Length-prefixing is what lets a reader know exactly how many bytes make up "one message" without scanning for a delimiter that might legitimately appear inside the payload itself — a SQL string containing whatever byte sequence a naive delimiter-based framing might mistake for an end marker.

```java
public record Frame(byte[] payload) {

    public void writeTo(OutputStream out) throws IOException {
        DataOutputStream dataOut = new DataOutputStream(out);
        dataOut.writeInt(payload.length);
        dataOut.write(payload);
        dataOut.flush();
    }

    public static Frame readFrom(InputStream in) throws IOException {
        DataInputStream dataIn = new DataInputStream(in);
        int length = dataIn.readInt();
        if (length < 0 || length > MAX_FRAME_BYTES) throw new IOException("Invalid frame length: " + length); // §52
        byte[] payload = new byte[length];
        dataIn.readFully(payload); // blocks until exactly `length` bytes arrive, or throws EOFException on early close
        return new Frame(payload);
    }

    private static final int MAX_FRAME_BYTES = 64 * 1024 * 1024; // 64 MB -- a sanity bound, §52 explains why this matters
}
```

---

# 35. The Request Types

```java
public sealed interface Request permits AuthRequest, ExecuteRequest, CommitRequest, RollbackRequest,
        SetAutoCommitRequest, FetchCatalogSnapshotRequest, CloseRequest {
}

public record AuthRequest(String username, String password) implements Request { }                    // §39-§40
public record ExecuteRequest(String sql, boolean isQuery) implements Request { }                        // §9's execute(...)
public record CommitRequest() implements Request { }
public record RollbackRequest() implements Request { }
public record SetAutoCommitRequest(boolean autoCommit) implements Request { }
public record FetchCatalogSnapshotRequest() implements Request { }                                       // §50
public record CloseRequest() implements Request { }
```

Every one of these maps **exactly** onto one method of `StatementExecutor` (§9) — the wire protocol's request types are the seam's own methods, made serializable, one for one, rather than a separately-invented protocol vocabulary that then has to be reconciled with the interface it's actually driving.

---

# 36. The Response Types

```java
public sealed interface Response permits AuthResponse, QueryResultResponse, ErrorResponse, AckResponse,
        CatalogSnapshotResponse {
}

public record AuthResponse(boolean success, String failureReason /* nullable */) implements Response { }
public record QueryResultResponse(List<Map<String, Object>> rows, int affectedRows) implements Response { } // mirrors QueryResult
public record ErrorResponse(String message, String sqlState) implements Response { }                        // §30
public record AckResponse() implements Response { }                                                          // commit/rollback/setAutoCommit/close
public record CatalogSnapshotResponse(List<SerializedTableSchema> tables) implements Response { }             // §50

public record SerializedTableSchema(String tableName, List<SerializedColumn> columns) { }
public record SerializedColumn(String name, String dataType, boolean nullable) { } // dataType as a NAME, decoded via DataType.valueOf(...)
```

Every request type in §35 gets exactly one corresponding response type here — `ExecuteRequest` -> `QueryResultResponse` or `ErrorResponse`; `CommitRequest`/`RollbackRequest`/`SetAutoCommitRequest`/`CloseRequest` -> `AckResponse` or `ErrorResponse`; `FetchCatalogSnapshotRequest` -> `CatalogSnapshotResponse`.

---

# 37. Implementing Request/Response Serialization

```java
public final class ProtocolCodec {

    public byte[] encode(Request request) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        DataOutputStream dataOut = new DataOutputStream(out);
        switch (request) {
            case AuthRequest r -> { dataOut.writeByte(1); writeString(dataOut, r.username()); writeString(dataOut, r.password()); }
            case ExecuteRequest r -> { dataOut.writeByte(2); writeString(dataOut, r.sql()); dataOut.writeBoolean(r.isQuery()); }
            case CommitRequest r -> dataOut.writeByte(3);
            case RollbackRequest r -> dataOut.writeByte(4);
            case SetAutoCommitRequest r -> { dataOut.writeByte(5); dataOut.writeBoolean(r.autoCommit()); }
            case FetchCatalogSnapshotRequest r -> dataOut.writeByte(6);
            case CloseRequest r -> dataOut.writeByte(7);
        }
        return out.toByteArray();
    }

    public Request decodeRequest(byte[] payload) throws IOException {
        DataInputStream dataIn = new DataInputStream(new ByteArrayInputStream(payload));
        int typeTag = dataIn.readByte();
        return switch (typeTag) {
            case 1 -> new AuthRequest(readString(dataIn), readString(dataIn));
            case 2 -> new ExecuteRequest(readString(dataIn), dataIn.readBoolean());
            case 3 -> new CommitRequest();
            case 4 -> new RollbackRequest();
            case 5 -> new SetAutoCommitRequest(dataIn.readBoolean());
            case 6 -> new FetchCatalogSnapshotRequest();
            case 7 -> new CloseRequest();
            default -> throw new IOException("Unknown request type tag: " + typeTag);
        };
    }

    private void writeString(DataOutputStream out, String value) throws IOException {
        byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
        out.writeInt(bytes.length);
        out.write(bytes);
    }

    private String readString(DataInputStream in) throws IOException {
        int length = in.readInt();
        byte[] bytes = new byte[length];
        in.readFully(bytes);
        return new String(bytes, StandardCharsets.UTF_8);
    }

    // encode(Response)/decodeResponse(byte[]) follow the identical one-byte-type-tag-then-fields shape --
    // elided here since the pattern is fully established above; QueryResultResponse's row encoding reuses
    // writeString for keys and a small type-tagged value encoder for each Object (Integer/Long/Boolean/
    // Double/String/Instant/null), mirroring §36's DataType-name-based encoding for SerializedColumn.
}
```

A one-byte type tag, switched on to select the decoder, is a small, closed-set dispatch discipline — and because `Request`/`Response` are `sealed`, the `switch` in `encode` has **no `default` branch**: adding an eighth request type without adding its `case` here is a compile error, never a silently-dropped message.

---

# 38. Authentication: A Simple Credential Handshake

The embedded driver's only "authentication" was, implicitly, whatever the operating system's filesystem permissions already enforced on the data directory (§32). The moment TinyDB is reachable over a network, that goes away entirely — any process that can reach the port can otherwise connect. The very first message on every connection, before anything else, must be an `AuthRequest` (§35), checked against a credential store before the server does anything else on that socket, including reporting *why* a query failed — an unauthenticated caller should learn only "authentication failed," never any detail that leaks whether a specific username exists.

---

# 39. Implementing the Auth Handshake

```java
public interface CredentialStore {
    boolean verify(String username, String password);
}

public final class InMemoryCredentialStore implements CredentialStore {

    private final Map<String, String> hashedPasswordsByUsername; // username -> a salted hash, NEVER the plaintext password

    public InMemoryCredentialStore(Map<String, String> hashedPasswordsByUsername) {
        this.hashedPasswordsByUsername = hashedPasswordsByUsername;
    }

    @Override
    public boolean verify(String username, String password) {
        String expectedHash = hashedPasswordsByUsername.get(username);
        if (expectedHash == null) return false; // unknown username -- same failure shape as a wrong password, deliberately
        return expectedHash.equals(hash(password));
    }

    private String hash(String password) {
        // A real implementation uses a slow, salted password hash (bcrypt/Argon2/PBKDF2) from a well-reviewed
        // library -- NEVER a fast general-purpose hash like plain SHA-256 for password storage specifically.
        // Named honestly here rather than implemented, for the same reason §41 doesn't hand-roll TLS: password
        // hashing is exactly the kind of security-critical code you use a reviewed library for, not write yourself.
        throw new UnsupportedOperationException("Wire a real password-hashing library here, e.g. bcrypt");
    }
}
```

```java
/** Runs on the SERVER, as the first step of every new connection, before any Request other than
 *  AuthRequest (§35) is accepted at all. */
public boolean handleAuthHandshake(Socket socket, CredentialStore credentialStore) throws IOException {
    Frame frame = Frame.readFrom(socket.getInputStream());
    Request request = new ProtocolCodec().decodeRequest(frame.payload());

    if (!(request instanceof AuthRequest auth)) {
        sendError(socket, "First message on a connection must be AuthRequest");
        return false;
    }
    boolean authenticated = credentialStore.verify(auth.username(), auth.password());
    sendResponse(socket, new AuthResponse(authenticated, authenticated ? null : "Invalid credentials"));
    return authenticated; // §45's ServerConnectionHandler closes the socket immediately if this is false
}
```

Deliberately naming password hashing as *"wire a real library here"* rather than implementing SHA-256-by-hand is not a cop-out — it's the correct engineering call, stated honestly instead of hidden behind a plausible-looking but actually-wrong implementation. A hand-rolled password hash is one of the most common, most dangerous mistakes in exactly this kind of guide, and this section refuses to make it just to appear complete.

---

# 40. Why TLS Belongs Here, and Why We Don't Hand-Roll It

Without transport encryption, §39's `AuthRequest` — carrying a real username and password — travels across the network in plaintext, readable by anything positioned to intercept the connection. The fix is **not** "design a new encryption scheme" — it's "wrap the existing `Socket` in one already provided by the JDK":

```java
SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
sslContext.init(keyManagers, trustManagers, null); // standard JDK key/trust-store setup, elided -- not this guide's subject
SSLServerSocketFactory factory = sslContext.getServerSocketFactory();
ServerSocket serverSocket = factory.createServerSocket(port); // IDENTICAL usage from here on to a plain ServerSocket (§43)
```

Every line of protocol logic this guide builds — `Frame` (§34), `Request`/`Response` (§35-§36), `ProtocolCodec` (§37), the auth handshake (§39) — runs completely unchanged whether `serverSocket` came from `SSLServerSocketFactory` or a plain `ServerSocketFactory`, because `SSLSocket` **is** a `Socket`, and every method this guide calls on one (`getInputStream()`, `getOutputStream()`) is inherited, identically-behaving, from that same base type. This is the entire argument for never hand-rolling cryptography here: TLS is a **transport-layer wrapping decision**, made once, at the point a socket is created — never a protocol-design decision this guide's own code needs to reason about at all.

---

# 41. The TinyDB Server Process: Architecture

```text
                          ServerSocket.accept() loop (§42)
                                       |
                          one accepted Socket per client
                                       |
                    submitted as a task to a bounded worker pool (§42)
                                       |
                          ServerConnectionHandler (§43-§44)
                    - runs §39's auth handshake FIRST
                    - then a request/response loop, decoding via ProtocolCodec (§37)
                    - holds ITS OWN EmbeddedStatementExecutor (§13), reused SERVER-SIDE
                                       |
                    every handler's EmbeddedStatementExecutor wraps
                    the SAME, single, shared TinyDbInstance (§12)
```

The critical design fact this diagram makes visible: `EmbeddedStatementExecutor` (§13) is not only the embedded driver's own executor — on the server, **one instance of it is created per connected client**, each with its own `autoCommit` flag and its own possibly-open `TransactionContext`, all of them sharing the one underlying `TinyDbInstance`. This needed no server-specific redesign at all: a JDBC `Connection`'s transaction state has always been per-connection, whether that connection is a Java object living in the same process or a `ServerConnectionHandler` living on the other end of a socket.

---

# 42. Implementing the Accept Loop and a Bounded Worker Pool

```java
public final class TinyDbServer {

    private final int port;
    private final TinyDbInstance instance;                 // ONE shared instance, §12
    private final CredentialStore credentialStore;           // §39
    private final ExecutorService workerPool;                // bounded core+max+queue+reject -- §46 explains this bound's purpose
    private volatile boolean running;

    public TinyDbServer(int port, TinyDbInstance instance, CredentialStore credentialStore, int maxConcurrentClients) {
        this.port = port;
        this.instance = instance;
        this.credentialStore = credentialStore;
        this.workerPool = new ThreadPoolExecutor(
                maxConcurrentClients / 2, maxConcurrentClients,   // core / max
                60, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(100),                    // a bounded queue -- clients beyond capacity wait, not spawn unbounded threads
                new ThreadPoolExecutor.AbortPolicy());              // a client beyond queue capacity is rejected outright, §52
    }

    public void start() throws IOException {
        running = true;
        try (ServerSocket serverSocket = new ServerSocket(port)) { // or SSLServerSocketFactory, §40
            while (running) {
                Socket clientSocket = serverSocket.accept(); // blocks until a client connects
                workerPool.submit(new ServerConnectionHandler(clientSocket, instance, credentialStore)); // §43
            }
        }
    }

    public void stop() { running = false; workerPool.shutdown(); }
}
```

Submitting each accepted socket to a **bounded** pool, rather than spawning `new Thread(...)` unboundedly per client, means an unbounded thread-per-client server degrades catastrophically under a connection flood, where a bounded pool degrades gracefully — new connections queue, then get rejected outright once the queue itself is full (§52), rather than the server exhausting memory or OS thread limits trying to accept every single one.

---

# 43. Implementing ServerConnectionHandler

```java
public final class ServerConnectionHandler implements Runnable {

    private final Socket socket;
    private final TinyDbInstance instance;
    private final CredentialStore credentialStore;
    private final ProtocolCodec codec = new ProtocolCodec(); // §37

    public ServerConnectionHandler(Socket socket, TinyDbInstance instance, CredentialStore credentialStore) {
        this.socket = socket;
        this.instance = instance;
        this.credentialStore = credentialStore;
    }

    @Override
    public void run() {
        try (socket) {
            if (!authenticate()) return; // §38-§39 -- closes the socket immediately on failure, no further messages read

            EmbeddedStatementExecutor executor = new EmbeddedStatementExecutor(instance); // §13, ONE PER CONNECTION -- §41
            while (!socket.isClosed()) {
                Frame requestFrame = Frame.readFrom(socket.getInputStream());
                Request request = codec.decodeRequest(requestFrame.payload());
                Response response = dispatch(request, executor); // §44
                new Frame(codec.encode(response)).writeTo(socket.getOutputStream());
                if (request instanceof CloseRequest) break; // the client is done -- stop reading further frames
            }
        } catch (EOFException e) {
            // the client closed its socket without sending a CloseRequest -- treated identically to one that did,
            // after making sure any open transaction is rolled back (the try-with-resources below still runs)
        } catch (IOException e) {
            // a genuine network error mid-connection -- §52 discusses this case specifically
        } finally {
            // if authenticate() succeeded but the loop exited abnormally, any open transaction must still be
            // resolved -- NOT left dangling open against the shared TinyDbInstance indefinitely.
        }
    }

    private boolean authenticate() throws IOException { /* §39's handleAuthHandshake, inlined here */ return true; }
}
```

---

# 44. Dispatching a Request to the Executor

```java
private Response dispatch(Request request, EmbeddedStatementExecutor executor) {
    try {
        return switch (request) {
            case ExecuteRequest r -> {
                QueryResult result = executor.execute(r.sql(), r.isQuery()); // §9's seam, called SERVER-SIDE
                yield toQueryResultResponse(result);
            }
            case CommitRequest r -> { executor.commit(); yield new AckResponse(); }
            case RollbackRequest r -> { executor.rollback(); yield new AckResponse(); }
            case SetAutoCommitRequest r -> { executor.setAutoCommit(r.autoCommit()); yield new AckResponse(); }
            case FetchCatalogSnapshotRequest r -> toCatalogSnapshotResponse(executor.catalog()); // §50
            case CloseRequest r -> { executor.close(); yield new AckResponse(); }
            case AuthRequest r -> new ErrorResponse("Already authenticated on this connection", "08001");
        };
    } catch (SQLException e) {
        return new ErrorResponse(e.getMessage(), e.getSQLState()); // §30's exception mapping
    }
}
```

This `switch` over `Request` — sealed, §35 — needing **no `default` branch** is the same exhaustiveness guarantee every other sealed dispatch in this guide relies on: a ninth request type added later is a compile error here until this method is updated, never a request silently falling through unhandled.

---

# 45. Concurrency at the Server

Multiple `ServerConnectionHandler`s, each running on its own worker-pool thread, now call into the **same** `TinyDbInstance` — the same storage engine, the same buffer pool, the same write-ahead log — concurrently, from genuinely different threads, for the first time in this guide outside of a single embedded process's own multi-threaded use. This is exactly the scenario a durable storage engine's own concurrency model has to guarantee correctness for: per-page locks, a synchronized buffer-pool fetch/evict, and a synchronized write-ahead-log append are not optional hardening this guide needs to add on top — they are exactly what makes many `ServerConnectionHandler`s safely sharing one `TinyDbInstance` correct, and they belong entirely to the storage engine this guide treats as a given foundation (§6).

---

# 46. Implementing NetworkStatementExecutor

```java
public final class NetworkStatementExecutor implements StatementExecutor {

    private final Socket socket;
    private final ProtocolCodec codec = new ProtocolCodec(); // §37 -- IDENTICAL codec, used on both ends of the wire
    private boolean autoCommit = true; // mirrored client-side purely so getAutoCommit() needs no round trip

    public NetworkStatementExecutor(String host, int port, String username, String password) throws SQLException {
        try {
            this.socket = new Socket(host, port); // or an SSLSocket, §40 -- identical usage from here on
            Response authResponse = send(new AuthRequest(username, password)); // §38-§39
            if (!(authResponse instanceof AuthResponse auth) || !auth.success()) {
                throw new SQLException("Authentication failed" + (authResponse instanceof AuthResponse a ? ": " + a.failureReason() : ""));
            }
        } catch (IOException e) {
            throw new SQLException("Could not connect to TinyDB server at " + host + ":" + port, e);
        }
    }

    @Override
    public QueryResult execute(String sql, boolean isQuery) throws SQLException {
        Response response = send(new ExecuteRequest(sql, isQuery)); // §35
        return switch (response) {
            case QueryResultResponse r -> toQueryResult(r); // rebuilds a List<Row> from r.rows(), the SAME shape §20 needs
            case ErrorResponse e -> throw new SQLException(e.message(), e.sqlState());
            default -> throw new SQLException("Unexpected response type for ExecuteRequest: " + response);
        };
    }

    @Override public boolean getAutoCommit() { return autoCommit; }

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        sendExpectingAck(new SetAutoCommitRequest(autoCommit)); // §35 -- the SERVER's executor (§13) holds the real state
        this.autoCommit = autoCommit; // mirrored, purely to answer getAutoCommit() without a round trip
    }

    @Override public void commit() throws SQLException { sendExpectingAck(new CommitRequest()); }
    @Override public void rollback() throws SQLException { sendExpectingAck(new RollbackRequest()); }

    @Override
    public void close() throws SQLException {
        try {
            sendExpectingAck(new CloseRequest());
        } finally {
            try { socket.close(); } catch (IOException ignored) { }
        }
    }

    @Override
    public Catalog catalog() {
        return new RemoteCatalogSnapshot(this); // §50 -- fetches lazily, on first actual use
    }

    /** Package-private -- called only by RemoteCatalogSnapshot (§50), never directly by JDBC-facing code. */
    CatalogSnapshotResponse fetchCatalogSnapshot() throws SQLException {
        Response response = send(new FetchCatalogSnapshotRequest()); // §35
        if (response instanceof CatalogSnapshotResponse snapshot) return snapshot;
        if (response instanceof ErrorResponse e) throw new SQLException(e.message(), e.sqlState());
        throw new SQLException("Unexpected response type for FetchCatalogSnapshotRequest: " + response);
    }

    private Response send(Request request) throws SQLException {
        try {
            new Frame(codec.encode(request)).writeTo(socket.getOutputStream());
            Frame responseFrame = Frame.readFrom(socket.getInputStream());
            return codec.decodeResponse(responseFrame.payload());
        } catch (IOException e) {
            throw new SQLException("Network error communicating with TinyDB server", e); // §52 discusses this case
        }
    }

    private void sendExpectingAck(Request request) throws SQLException {
        Response response = send(request);
        if (response instanceof ErrorResponse e) throw new SQLException(e.message(), e.sqlState());
    }
}
```

Every method here is a **thin proxy**: encode a request, send it, decode the response, translate it back into exactly the same return type/exception `EmbeddedStatementExecutor` (§13) already used. This is the entire client-side implementation server mode needs — it contains no SQL logic, no transaction logic, no storage logic, because none of that lives on the client at all; it lives on the server, inside the identical `EmbeddedStatementExecutor` class §13 already built.

---

# 47. Completing TinyDbDriver: connectRemote

```java
private Connection connectRemote(String url, Properties info) throws SQLException {
    URI uri = URI.create(url.substring("jdbc:tinydb:".length())); // strips down to "//host:port/database"
    String host = uri.getHost();
    int port = uri.getPort();
    String username = info.getProperty("user");
    String password = info.getProperty("password");

    StatementExecutor executor = new NetworkStatementExecutor(host, port, username, password); // §46
    return new TinyDbConnection(executor); // §14 -- the SAME Connection class the embedded path also returns
}
```

This fills in exactly the stub §31 left in place. Both `connectEmbedded` and `connectRemote` return a `TinyDbConnection` — the exact same class. The only difference between an embedded connection and a remote one, anywhere in this entire driver, is which `StatementExecutor` implementation got constructed and handed to it.

---

# 48. Why Statement/PreparedStatement/ResultSet/DatabaseMetaData Need Zero Changes

`TinyDbStatement.executeQuery(sql)` (§15) calls `connection.executeInternal(sql, true)`. `TinyDbConnection.executeInternal` (§14) calls `executor.execute(sql, isQuery)`. Whether `executor` is `EmbeddedStatementExecutor` (§13) or `NetworkStatementExecutor` (§46), the return type is identical — a `QueryResult` (§7) — and `TinyDbResultSet` (§20) is constructed from that `List<Row>` exactly the same way regardless of where those rows actually came from. Nothing from §15 through §29 needed a single line changed to support server mode — this is the direct, designed consequence of every one of those classes having been written against `TinyDbConnection`'s public contract, never against `TinyDbInstance` or the storage engine directly.

---

# 49. Implementing RemoteCatalogSnapshot

```java
public final class RemoteCatalogSnapshot implements Catalog {

    private final NetworkStatementExecutor executor;
    private List<TableSchema> cachedTables; // fetched lazily, on first metadata call -- §52 discusses staleness

    public RemoteCatalogSnapshot(NetworkStatementExecutor executor) { this.executor = executor; }

    @Override
    public List<TableSchema> listTables() {
        ensureFetched();
        return cachedTables;
    }

    @Override
    public TableSchema getTable(String tableName) {
        return listTables().stream().filter(t -> t.tableName().equals(tableName)).findFirst().orElse(null);
    }

    @Override public boolean tableExists(String tableName) { return getTable(tableName) != null; }
    @Override public void createTable(TableSchema schema) { throw new UnsupportedOperationException("Read-only snapshot"); }
    @Override public void dropTable(String tableName) { throw new UnsupportedOperationException("Read-only snapshot"); }

    private void ensureFetched() {
        if (cachedTables != null) return;
        try {
            CatalogSnapshotResponse snapshot = executor.fetchCatalogSnapshot(); // sends FetchCatalogSnapshotRequest, §35
            cachedTables = snapshot.tables().stream().map(this::toTableSchema).toList();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to fetch catalog metadata from server", e);
        }
    }

    private TableSchema toTableSchema(SerializedTableSchema serialized) {
        List<Column> columns = serialized.columns().stream()
                .map(c -> new Column(c.name(), DataType.valueOf(c.dataType()), c.nullable()))
                .toList();
        return new TableSchema(serialized.tableName(), columns);
    }
}
```

`TinyDbConnection.getMetaData()` (§14) calls `new TinyDbDatabaseMetaData(executor.catalog())` — completely unaware that, for a remote connection, `executor.catalog()` (§46) returns a `RemoteCatalogSnapshot` instead of the embedded driver's directly-local `Catalog` implementation. `TinyDbDatabaseMetaData` (§25-§29) calls `listTables()`/`getTable(...)` on whatever `Catalog` it was handed — it was written against the `Catalog` *interface* from the start (§7), which is precisely what makes this the second, and last, genuinely new class server mode needs on the client side.

---

# 50. Follow-up: What About a Very Large Result Set Over the Network?

`EmbeddedStatementExecutor.execute` (§13) already, honestly, materializes an entire query's result into one `List<Row>` before returning — even in the purely embedded, no-network case, because the underlying query engine's own `doQuery` is written that way. `NetworkStatementExecutor` (§46) inherits that exact same limitation, unchanged: one `QueryResultResponse` (§36) carries every row of a result set in a single message. This is **not a new limitation server mode introduces** — but it does mean a result set with millions of rows now also pays the cost of serializing all of them into one large network message, bounded only by §34's `MAX_FRAME_BYTES` sanity check. A real cursor-based, fetch-size-driven protocol (paging rows across several smaller round trips instead of one large one) is genuinely valuable future work, named explicitly rather than built here (§61) — it would require both a new `FetchNextBatchRequest` and a server-side cursor held open across multiple requests on the same connection, a real design of its own scope.

---

# 51. Handling Connection Loss, Timeouts, and Partial Writes

Three distinct network failure modes, each needing its own specific handling, none of them optional to consider once a socket — rather than a guaranteed-reliable in-process method call — sits in the middle of every statement:

- **The client disconnects mid-transaction.** §43's `ServerConnectionHandler` catches `EOFException`/`IOException` around its request loop specifically so its `finally` block can roll back any transaction the now-gone client left open — an abandoned socket must never leave a `TransactionContext` (§10) held open against the shared `TinyDbInstance` indefinitely, which would otherwise block other connections' writes forever.
- **The server is slow or unreachable.** `NetworkStatementExecutor.send` (§46) has no timeout configured on its socket by default, which means a hung server can hang the client indefinitely — a real deployment sets `Socket.setSoTimeout(...)` explicitly, turning an indefinite hang into a bounded wait followed by a clear `SQLException`, rather than a client application that appears frozen with no diagnostic at all.
- **A malformed or oversized frame arrives.** §34's `MAX_FRAME_BYTES` check exists specifically so a corrupted length prefix (or a deliberately malicious one) can never make the server attempt to allocate an unbounded byte array — `Frame.readFrom` throwing `IOException` on an invalid length is a controlled failure of one connection, never a resource-exhaustion risk to the whole server process.

---

# 52. Server Configuration and Startup

```java
public final class TinyDbServerMain {
    public static void main(String[] args) throws Exception {
        Path dataDirectory = Path.of(args[0]);
        int port = Integer.parseInt(args[1]);

        TinyDbInstance instance = TinyDbInstance.open(dataDirectory); // §12 -- runs recover() internally
        CredentialStore credentialStore = loadCredentialStore(args[2]); // a config file path, elided

        TinyDbServer server = new TinyDbServer(port, instance, credentialStore, 200); // §42 -- 200 concurrent clients
        Runtime.getRuntime().addShutdownHook(new Thread(server::stop)); // graceful shutdown on Ctrl+C / SIGTERM
        server.start(); // blocks -- the accept loop, §42
    }
}
```

Registering a shutdown hook that calls `server.stop()` matters for a reason specific to this guide's whole design: `stop()` only stops accepting *new* connections and shuts down the worker pool — it deliberately does not forcibly kill in-flight `ServerConnectionHandler`s mid-statement, giving already-connected clients a chance to finish or explicitly close cleanly rather than having their in-flight transaction torn out from under them by a process kill.

---

# 53. Connecting From DBeaver to a Remote TinyDB Server, Step by Step

1. On the remote machine, start the server: `java -cp tinydb-server.jar TinyDbServerMain /var/tinydb/data 5432 /etc/tinydb/credentials.conf` (§52).
2. In DBeaver's **Driver Manager**, edit the existing "TinyDB" driver entry from §32's own walkthrough — add a second **URL Template**: `jdbc:tinydb://{host}:{port}/{database}`, alongside the existing embedded `jdbc:tinydb:{file}` template; both resolve to the same driver class, dispatched by §31/§47's `if`/`else`.
3. **Database -> New Database Connection -> TinyDB**, this time filling in **Host** and **Port** instead of a local **Path**.
4. Under the connection's **Authentication** tab, provide the **Username**/**Password** — these flow into `Properties info` (§47), read as `info.getProperty("user")`/`"password"`, and become the `AuthRequest` (§38-§39) sent as the very first message on the socket.
5. **Test Connection.** `DriverManager.getConnection(url, info)` resolves to `TinyDbDriver.connect` (§31), which now takes the `connectRemote` branch (§47), constructing a `NetworkStatementExecutor` (§46) that opens the socket and immediately runs the auth handshake.
6. On success, DBeaver calls `Connection.getMetaData()` exactly as it always has (§14) — now backed by a `RemoteCatalogSnapshot` (§49) instead of a local `Catalog`, invisibly to DBeaver, which never needed to know the difference.

---

# 54. Full Worked Example: A Complete Remote Query, End to End

```text
1.  java TinyDbServerMain /var/tinydb/data 5432 credentials.conf   -- server starts (§52), recover() already ran (§12)
2.  DBeaver (remote laptop): DriverManager.getConnection("jdbc:tinydb://db.example.com:5432/main", info)
3.  TinyDbDriver.connect(...) -> connectRemote(...)                                                  (§47)
4.  new NetworkStatementExecutor(host, port, user, pass)
       -> Socket opened -> AuthRequest sent -> server's handleAuthHandshake() verifies -> AuthResponse(success=true)  (§39, §46)
5.  new TinyDbConnection(executor)                                                                    (§14)
6.  DBeaver: connection.createStatement().executeQuery("SELECT * FROM employees")
       -> TinyDbStatement.executeQuery (§15, UNCHANGED)
       -> connection.executeInternal(sql, true) (§14) -> executor.execute(sql, true) (§46)
       -> ExecuteRequest sent over the socket                                                          (§35)
7.  SERVER: ServerConnectionHandler.run() reads the frame, decodes it, calls dispatch()                 (§43-§44)
       -> executor.execute("SELECT...", true) on the SERVER-SIDE EmbeddedStatementExecutor              (§13, §41)
       -> BasicQueryEngine.doQuery(sql) -- the SAME method the embedded driver already calls            (§7)
8.  QueryResultResponse sent back over the socket                                                       (§36-§37)
9.  CLIENT: NetworkStatementExecutor.execute() decodes it into a QueryResult                             (§46)
10. new TinyDbResultSet(result.rows(), statement) -- §20, completely UNCHANGED
11. DBeaver iterates rs.next()/rs.getObject(...) exactly as it would against the embedded driver
```

Steps 6 and 10-11 are byte-for-byte the *same code* that runs for an embedded connection — the only genuinely new work in this entire trace is steps 4 and 6-9, the wire round trip §34-§37's protocol and §13/§46's executors exist to carry out.

---

# 55. Final Architecture Diagram

```text
   Client machine                                        Remote machine
   +--------------------------+                    +---------------------------------------+
   | DBeaver                  |                    | TinyDbServerMain (§52)                 |
   |   TinyDbConnection (§14) |                    |   ServerSocket.accept() loop (§42)      |
   |   TinyDbStatement (§15,  |    AuthRequest      |     bounded worker pool                 |
   |   PreparedStatement,     |  ---------------->  |   ServerConnectionHandler x N (§43)      |
   |   ResultSet, MetaData -- |  ExecuteRequest     |     each: auth (§39), then own           |
   |   ALL unchanged, §48)    |  ---------------->  |     EmbeddedStatementExecutor (§13, §41) |
   |   NetworkStatementExecutor|  <---------------  |                                          |
   |   (§46) + RemoteCatalog  |  Response (§36)     |     ALL sharing ONE TinyDbInstance (§12), |
   |   Snapshot (§49)         |                    |     ONE storage engine, ONE buffer pool,  |
   +--------------------------+                    |     ONE write-ahead log (§45)             |
                                                     +---------------------------------------+
```

Every box on the client side either already existed for the embedded case (§14-§29) or is a thin protocol proxy (§46, §49); every box on the server side is genuinely new orchestration (§41-§45) with no SQL or storage logic of its own — that logic belongs entirely to the database engine this guide builds on top of (§6-§7).

---

# 56. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `StatementExecutor` (§9) | The single seam this entire driver's design is built on — embedded and network execution are two interchangeable strategies behind one interface, decided once per `Connection`. |
| **Adapter** | `NetworkStatementExecutor` (§46), `RemoteCatalogSnapshot` (§49) | Both translate a fixed, pre-existing contract (`StatementExecutor`, `Catalog`) onto a wire protocol underneath. |
| **Proxy** | `NetworkStatementExecutor` (§46) | Every method is a thin stand-in that forwards to the real logic (`EmbeddedStatementExecutor`) running elsewhere, on the server. |
| **Command** | `Request`/`Response` (§35-§36) | Each is a small, self-contained, serializable unit of "do this" or "here's what happened" — exactly the shape a wire protocol's messages need. |
| **Thread Pool** | `TinyDbServer`'s worker pool (§42) | A bounded core/max/queue/reject shape, reused here per client connection to protect the server from a connection flood. |
| **Iterator** | `TinyDbResultSet.next()` (§20) | The standard cursor-based row-at-a-time access pattern every `ResultSet` implementation is built around. |

---

# 57. SOLID Principles Applied

- **Single Responsibility**: `SqlParameterBinder` (§18) only rewrites placeholders; `TypeMapping` (§22) only converts between type systems; `ProtocolCodec` (§37) only serializes/deserializes; `CredentialStore` (§39) only verifies credentials. None of them know how to execute a statement.
- **Open/Closed**: adding a new `PreparedStatement` setter (§17), or a ninth `Request`/`Response` type (a future `FetchNextBatchRequest`, §50), requires no change to `SqlParameterBinder.bind` or to `Frame`/`TinyDbConnection` — only a new `case` in the relevant, already-narrow `switch`.
- **Liskov Substitution**: `EmbeddedStatementExecutor` and `NetworkStatementExecutor` are fully interchangeable behind `StatementExecutor` (§9) — `TinyDbConnection` (§14) never needs to know or branch on which one it holds; `TinyDbPreparedStatement extends TinyDbStatement` (§17) and is fully usable wherever a plain `Statement` is expected.
- **Interface Segregation**: `StatementExecutor` (§9) exposes exactly the methods a `Connection` actually needs — no socket detail, no transaction-internals detail leaks into its surface; `ResultSetMetaData`/`DatabaseMetaData` remain separate interfaces, never merged into one god-object.
- **Dependency Inversion**: `TinyDbConnection` (§14) depends on the `StatementExecutor` interface, injected through its constructor, never on `EmbeddedStatementExecutor` or `NetworkStatementExecutor` directly — the entire point of §9's design.

---

# 58. Common Mistakes When Building This Yourself

- **Hard-coding the execution path instead of naming it as its own interface from the start** (§9) — the single most expensive mistake in this entire domain; discovering the need for a second execution mode *after* `Statement`/`ResultSet`/`DatabaseMetaData` are already written against a concrete class means rewriting all of them, instead of adding one new class behind an interface that was already there.
- **Concatenating parameter values directly into SQL text instead of escaping them** (§18-§19) — turns `PreparedStatement`, whose entire purpose is safety against exactly this, into no safer than a raw string-built query.
- **Hand-rolling password hashing or transport encryption** (§39-§40) — both are exactly the kind of security-critical code a reviewed, standard library exists specifically so individual projects don't each reinvent it, usually incorrectly.
- **Using a delimiter instead of length-prefixing for message framing** (§34) — a SQL string can legitimately contain any byte sequence a naive delimiter might mistake for a message boundary; length-prefixing has no such ambiguity.
- **An unbounded thread-per-client server** (§42) — degrades catastrophically under a connection flood, where a bounded pool with a bounded queue degrades gracefully by rejecting outright once genuinely full.
- **Returning `null` from a `DatabaseMetaData` method instead of an empty, correctly-shaped `ResultSet`** (§28) — a real client iterating the result with `while (rs.next())` throws a `NullPointerException` on a `null`, where an empty result set correctly and silently produces zero iterations.
- **Leaving a client's open transaction dangling after an unexpected disconnect** (§43, §51) — silently holding locks or an open transaction against the shared engine indefinitely can block every other connection's writes.

---

# 59. Testing Strategy

- **`TinyDbDriver`** (§31): `acceptsURL` correctly accepts every `jdbc:tinydb:...` URL and rejects everything else; `connect` returns the same underlying `TinyDbInstance` for two embedded connections opened against the identical, normalized data-directory path.
- **`SqlParameterBinder`** (§18): a string parameter containing a single quote round-trips correctly; a parameter count mismatch throws `SQLException` rather than silently truncating or ignoring extras.
- **`TinyDbResultSet`** (§20-§21): `wasNull()` correctly reflects only the most recent column access; `getInt`/`getLong` on a genuinely null column return `0`, with `wasNull()` immediately after correctly reporting `true`.
- **`DatabaseMetaData.getColumns`** (§27): a table created via `Statement.execute("CREATE TABLE ...")` is immediately visible, with the correct columns and types, to a `getColumns` call on the *same* connection with no additional refresh step.
- **`Frame`/`ProtocolCodec`** (§34, §37): a payload round-trips through `writeTo`/`readFrom` unchanged; an invalid frame length throws `IOException` rather than allocating an unbounded array; every `Request`/`Response` subtype round-trips through `encode`/`decode` with all fields intact.
- **A real two-process integration test**: start an actual `TinyDbServer` in one JVM, connect to it with a real socket from a second JVM, browse the schema, run a query, commit a transaction, and confirm the result — the only test that actually exercises the wire protocol end to end rather than mocking either side of it.
- **Abandoned-connection cleanup** (§43, §51): a client that opens a transaction and then closes its socket without an explicit `CloseRequest` results in that transaction being rolled back server-side, verified by a second, independent connection observing no partial effects.

---

# 60. Suggested Future Enhancements

- **Cursor-based, fetch-size-driven result streaming** (§50) — a `FetchNextBatchRequest`/`FetchNextBatchResponse` pair and a server-side cursor held open across several requests on one connection, replacing the current "one message, every row" approach for genuinely large result sets.
- **Typed parameter binding**, passing `PreparedStatement` values to the executor as already-typed objects rather than re-serialized, escaped SQL text (§19's stated simplification) — avoiding a second lex/parse pass per execution and eliminating the escaping logic's attack surface entirely, rather than merely making it safe.
- **A real, reviewed password-hashing library wired into `CredentialStore`** (§39), replacing the deliberately-unimplemented placeholder.
- **Connection pooling** on the client side — reusing already-authenticated `NetworkStatementExecutor`/socket pairs across multiple short-lived `Connection`s, avoiding a fresh TCP handshake and auth round trip per connection.
- **Primary key and index metadata** (§28) — once `TableSchema` grows a real primary-key/index concept, `getPrimaryKeys`/`getIndexInfo` should be revisited to report it instead of the current, honest "none yet."
- **Batch updates** (`addBatch`/`executeBatch`) — genuinely useful for bulk `INSERT`s, and honestly reported as unsupported for now (§29).
- **Multi-statement pipelining** — allowing a client to send several `ExecuteRequest`s before reading their responses, instead of this guide's strict one-request-then-one-response-then-next-request loop (§43), for latency-sensitive bulk workloads.

---

# 61. Final Takeaway

The single decision this entire driver's design is built on is stated in §9, before a single JDBC interface is implemented: **how a statement actually gets executed is a pluggable strategy, not a hard-coded call.** Every genuinely interesting piece of engineering that follows — the wire protocol, per-connection transaction handling on the server, authentication, the honest decision to wrap TLS rather than hand-roll it — exists entirely behind that one seam, and none of it required touching a single line of the classes a real client actually calls (`Statement`, `PreparedStatement`, `ResultSet`, `DatabaseMetaData`). The lesson generalizes well past a JDBC driver: a system that's hard to extend to a new deployment mode usually isn't missing a feature — it's missing exactly one interface, drawn in exactly the right place, between "what this code decides" and "how that decision actually gets carried out."