# TinyDB --- Build a SQL Database from Scratch in Java

> A step-by-step learning project for building a small relational
> database engine, SQL server, terminal client, JDBC driver, MiniSpring
> integration, DBeaver connectivity, transactions, ACID, rollback, WAL,
> indexes, TTL, replication, and DynamoDB-style change streams.

## 0. What We Are Building

We will build **TinyDB**, a relational database server written from
scratch in Java.

The target architecture is:

``` text
                     +----------------------+
                     |      Applications    |
                     | MiniSpring / Java    |
                     | Python / Go / etc.   |
                     +----------+-----------+
                                |
                              JDBC
                                |
                     +----------v-----------+
                     |     TinyDB Driver    |
                     +----------+-----------+
                                |
                         TCP / TinyDB Wire
                                |
+-------------------------------v-------------------------------+
|                         TinyDB Server                          |
|                                                               |
|  SQL Parser -> Planner -> Executor -> Transaction Manager     |
|                         |                                     |
|                  Storage Engine                                |
|             +-----------+-----------+                          |
|             |           |           |                          |
|           WAL         Buffer       Lock/MVCC                   |
|                         |                                     |
|                    B+Tree / Hash Index                         |
|                         |                                     |
|                   Data Pages / Files                           |
|                                                               |
|   TTL Scheduler | Change Stream | Replication | Catalog       |
+-------------------------------+-------------------------------+
                                |
                         Files / Disk
```

The final goal is not merely a toy key-value store. The database should
understand SQL such as:

``` sql
CREATE DATABASE company;

CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(200) UNIQUE,
    salary DECIMAL(12,2),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP
);

INSERT INTO employee
(id, name, email, salary)
VALUES
(1, 'Arpan', 'arpan@example.com', 100000.00);

SELECT *
FROM employee
WHERE salary > 50000
ORDER BY salary DESC;

UPDATE employee
SET salary = 110000
WHERE id = 1;

DELETE FROM employee
WHERE id = 1;
```

Later, the same database should be reachable through:

``` text
1. TinyDB terminal client
2. JDBC
3. MiniSpring
4. DBeaver
5. Other database tools that can use JDBC
6. Eventually PostgreSQL-wire or MySQL-wire compatibility
```

------------------------------------------------------------------------

# 1. Important Architecture Decision

There are two different meanings of "support DBeaver":

### Option A --- Recommended for the first version

Create:

``` text
TinyDB JDBC Driver
```

DBeaver connects through the JDBC driver.

``` text
DBeaver
   |
 JDBC
   |
TinyDB JDBC Driver
   |
TCP
   |
TinyDB Server
```

This is much easier than implementing the PostgreSQL/MySQL wire
protocol.

### Option B --- Native DBeaver compatibility

Implement a standard database protocol such as:

``` text
PostgreSQL wire protocol
```

or

``` text
MySQL client/server protocol
```

Then DBeaver can connect as if TinyDB were that database.

This is a later milestone.

**Recommendation: build JDBC first, protocol compatibility later.**

------------------------------------------------------------------------

# 2. Technology Stack

Use:

``` text
Java 21
Maven
JUnit 5
SLF4J + Logback
Jackson
JDBC API
Java NIO
JMH
```

Suggested modules:

``` text
tinydb-parent
|
+-- tinydb-common
+-- tinydb-catalog
+-- tinydb-sql
+-- tinydb-storage
+-- tinydb-index
+-- tinydb-transaction
+-- tinydb-execution
+-- tinydb-server
+-- tinydb-jdbc
+-- tinydb-client
+-- tinydb-cli
+-- tinydb-stream
+-- tinydb-replication
+-- tinydb-minispring
+-- tinydb-tests
```

Keep modules separated so that storage, SQL, networking, and
transactions do not become one large package.

------------------------------------------------------------------------

# 3. Project Structure

Create:

``` text
tinydb/
├── pom.xml
├── README.md
│
├── tinydb-common/
├── tinydb-catalog/
├── tinydb-sql/
├── tinydb-storage/
├── tinydb-index/
├── tinydb-transaction/
├── tinydb-execution/
├── tinydb-server/
├── tinydb-jdbc/
├── tinydb-client/
├── tinydb-cli/
├── tinydb-stream/
├── tinydb-replication/
├── tinydb-minispring/
└── tinydb-tests/
```

Dependency direction should be approximately:

``` text
common
  ^
catalog
  ^
storage
  ^
index
  ^
transaction
  ^
execution
  ^
server
```

The SQL parser should not depend on networking.

The storage engine should not know about HTTP.

The transaction manager should not know about DBeaver.

This separation is essential for SOLID design.

------------------------------------------------------------------------

# 4. Phase 1 --- Build the In-Memory Database

Do not start with disk, networking, replication, and SQL simultaneously.

Start with:

``` text
Database
 -> Table
    -> Schema
    -> Rows
```

Basic domain model:

``` java
public record Column(
        String name,
        DataType type,
        boolean nullable
) {}
```

``` java
public enum DataType {
    INT,
    BIGINT,
    BOOLEAN,
    VARCHAR,
    DECIMAL,
    DATE,
    TIMESTAMP
}
```

``` java
public record TableSchema(
        String tableName,
        List<Column> columns
) {}
```

Represent a row initially as:

``` java
public final class Row {

    private final Map<String, Object> values;

    public Row(Map<String, Object> values) {
        this.values = Map.copyOf(values);
    }

    public Object get(String column) {
        return values.get(column);
    }

    public Set<String> columnNames() {
        return values.keySet();
    }
}
```

`columnNames()` is added here, ahead of where it is first needed, because the
executors in Phase 5 (section 10) and the scans in section 20 both need to
ask a `Row` what it actually contains -- for example, `UPDATE` (section
10.3) merges new values into a row's *existing* columns, which requires
knowing what those existing columns are.

This is deliberately simple.

Later replace generic maps with encoded records.

------------------------------------------------------------------------

# 5. Phase 2 --- Database Catalog

Create metadata structures:

``` text
Database
 ├── DatabaseName
 ├── Tables
 ├── Indexes
 └── Metadata
```

Catalog responsibilities:

``` java
public interface Catalog {

    void createDatabase(String name);

    void createTable(TableSchema schema);

    void dropTable(String tableName);

    TableSchema getTable(String tableName);

    boolean tableExists(String tableName);

    List<TableSchema> listTables();
}
```

`dropTable` is added here, ahead of where it was originally introduced, because
`DropTableExecutor` (Phase 5, below) needs somewhere real to remove a table's
metadata from. The catalog and the executors are designed together, not in
strict isolation.

The catalog is the database's metadata system.

It must eventually persist:

``` text
database
table
column
data type
primary key
unique constraint
index
foreign key
default value
```

------------------------------------------------------------------------

# 6. Phase 3 --- Implement SQL Lexer

Before parsing SQL, tokenize it.

Input:

``` sql
SELECT id, name
FROM employee
WHERE salary > 50000;
```

Tokens:

``` text
SELECT
IDENTIFIER(id)
COMMA
IDENTIFIER(name)
FROM
IDENTIFIER(employee)
WHERE
IDENTIFIER(salary)
GREATER_THAN
NUMBER(50000)
SEMICOLON
```

Create:

``` java
public enum TokenType {
    SELECT,
    INSERT,
    UPDATE,
    DELETE,
    CREATE,
    DROP,
    ALTER,

    FROM,
    WHERE,
    VALUES,
    INTO,
    SET,
    TABLE,
    DATABASE,

    AND,
    OR,
    NOT,

    IDENTIFIER,
    STRING,
    NUMBER,

    EQ,
    NE,
    GT,
    GTE,
    LT,
    LTE,

    COMMA,
    DOT,
    LPAREN,
    RPAREN,
    STAR,

    EOF
}
```

Implement:

``` java
Lexer
Token
TokenType
```

Test the lexer independently.

------------------------------------------------------------------------

# 7. Phase 4 --- Build the SQL Parser

Build an Abstract Syntax Tree.

For:

``` sql
SELECT id, name
FROM employee
WHERE salary > 50000;
```

Create:

``` text
SelectStatement
 |
 +-- columns: id, name
 +-- table: employee
 +-- where:
       salary > 50000
```

Define:

``` java
public sealed interface SqlStatement
        permits SelectStatement,
                InsertStatement,
                UpdateStatement,
                DeleteStatement,
                CreateTableStatement,
                DropTableStatement {
}
```

Example:

``` java
// SQL: SELECT AGE,NAME FROM STUDENTS WHERE AGE=18;
// AGE,NAME -> Fields/ columns
// STUDENTS -> Table
// AGE=18   -> Predicate
public record SelectStatement(
        List<String> columns,
        String table,
        Expression where
) implements SqlStatement {
}
```

``` java
// SQL: SELECT AGE,NAME FROM STUDENTS WHERE AGE=18;
// tblName -> STUDENTS
// flds	-> NAME, AGE
// vals	-> "Bob", 18 
// AGE=18   -> Predicate
public record InsertStatement(
        List<String> columns,
        String table,
        List<D_Constant> vals
) implements SqlStatement {
}

```

------------------------------------------------------------------------

# 8. SQL Grammar --- Start Small

Initial grammar:

``` text
statement
    := select
     | insert
     | update
     | delete
     | createTable
     | dropTable

select
    := SELECT selectList FROM identifier whereClause?

selectList
    := STAR
     | identifier (COMMA identifier)*

whereClause
    := WHERE expression

expression
    := comparison
     | expression AND expression
     | expression OR expression

comparison
    := identifier operator value

operator
    := =
     | !=
     | >
     | >=
     | <
     | <=
```

Then expand gradually.

------------------------------------------------------------------------

# 9. SQL Features Roadmap

Implement SQL in stages.

## Level 1

``` sql
CREATE TABLE
INSERT
SELECT
UPDATE
DELETE
DROP TABLE
```

## Level 2

``` sql
WHERE
AND
OR
NOT
ORDER BY
LIMIT
OFFSET
```

## Level 3

``` sql
PRIMARY KEY
UNIQUE
NOT NULL
DEFAULT
```

## Level 4

``` sql
COUNT
SUM
AVG
MIN
MAX
GROUP BY
HAVING
```

## Level 5

``` sql
INNER JOIN
LEFT JOIN
RIGHT JOIN
```

## Level 6

``` sql
CREATE INDEX
DROP INDEX
ALTER TABLE
```

## Level 7

``` sql
BEGIN
COMMIT
ROLLBACK
SAVEPOINT
ROLLBACK TO SAVEPOINT
```

## Level 8

``` sql
EXPLAIN
```

## Level 9

``` sql
CREATE DATABASE
DROP DATABASE
```

Do not claim "standard SQL" after implementing only
SELECT/INSERT/UPDATE/DELETE.

SQL is a large language with dialect differences.

The project should explicitly document its supported SQL subset and
dialect.

------------------------------------------------------------------------

# 10. Phase 5 --- Execution Engine

Separate SQL parsing from execution.

``` text
SQL
 |
Lexer
 |
Parser
 |
AST (SqlStatement)
 |
Planner              <- introduced in section 19, NOT yet
 |
Execution Plan        <- introduced in section 19, NOT yet
 |
Executor
 |
Storage Engine
```

The `Executor` interface shown further down already anticipates the
`ExecutionPlan` that section 19's Query Planner will eventually produce.
Do not build the planner yet. For the first working version, execute
directly against the `SqlStatement` the parser already hands you --
one executor per concrete statement type. Once the planner and query
operators (sections 19-20) exist, these same executors are adapted to
consume `ExecutionPlan` operator trees instead of raw statements. The
public contract barely changes; only what feeds it does.

------------------------------------------------------------------------

## 10.1 How the Lexer, Parser, and Executors Actually Connect

This is the part most write-ups skip. Concretely, end to end:

``` text
String sql = "INSERT INTO employee (id, name) VALUES (1, 'Alice')";

1. Lexer   : sql          -> List<Token>
2. Parser  : List<Token>  -> SqlStatement        (a concrete InsertStatement)
3. Engine  : SqlStatement -> pick the matching Executor
4. Executor: SqlStatement -> mutate/read StorageEngine -> QueryResult
```

The Lexer and Parser never know executors exist. The executors never know
how the statement was parsed. The only object that crosses that boundary
is `SqlStatement` -- a plain, immutable, parser-produced fact about what
the user asked for. That is the entire point of putting `SqlStatement`
between them: it lets the parsing side and the execution side change
independently, as long as both agree on the shape of the AST.

Define the remaining statement types (only `SelectStatement` was shown in
section 7):

``` java
public record InsertStatement(
        String table,
        List<String> columns,
        List<Object> values
) implements SqlStatement {}
```

``` java
public record UpdateStatement(
        String table,
        Map<String, Object> assignments,
        Expression where
) implements SqlStatement {}
```

``` java
public record DeleteStatement(
        String table,
        Expression where
) implements SqlStatement {}
```

``` java
public record CreateTableStatement(
        TableSchema schema
) implements SqlStatement {}
```

``` java
public record DropTableStatement(
        String table
) implements SqlStatement {}
```

`Expression` (referenced by `SelectStatement.where()` in section 7, and by
`UpdateStatement`/`DeleteStatement` above) also needs a real shape, since
"real implementation" means the `WHERE` clause has to actually evaluate
against a row, not just exist as an unused field:

``` java
public sealed interface Expression
        permits Comparison, And, Or {
}
```

``` java
public record Comparison(String column, String operator, Object value)
        implements Expression {}

public record And(Expression left, Expression right)
        implements Expression {}

public record Or(Expression left, Expression right)
        implements Expression {}
```

``` java
public final class ExpressionEvaluator {

    public static boolean matches(Row row, Expression expression) {
        return switch (expression) {
            case Comparison c -> evaluateComparison(row, c);
            case And a -> matches(row, a.left()) && matches(row, a.right());
            case Or o -> matches(row, o.left()) || matches(row, o.right());
        };
    }

    private static boolean evaluateComparison(Row row, Comparison c) {
        Object actual = row.get(c.column());
        int cmp = compare(actual, c.value());
        return switch (c.operator()) {
            case "=" -> Objects.equals(actual, c.value());
            case "!=" -> !Objects.equals(actual, c.value());
            case ">" -> cmp > 0;
            case ">=" -> cmp >= 0;
            case "<" -> cmp < 0;
            case "<=" -> cmp <= 0;
            default -> throw new IllegalStateException("Unsupported operator: " + c.operator());
        };
    }

    @SuppressWarnings("unchecked")
    private static int compare(Object actual, Object expected) {
        if (actual instanceof Comparable<?> && expected != null) {
            return ((Comparable<Object>) actual).compareTo(expected);
        }
        return 0;
    }
}
```

`Expression` is a sealed interface for the exact same reason `SqlStatement`
is: the compiler refuses to let `ExpressionEvaluator.matches` compile if a
new expression type is ever added without a matching `case` here. That
guarantee is worth far more than it looks like on first read -- it is what
stops a future `LIKE` or `IN` expression from silently falling through to
nothing.

Finally, `QueryResult` and `TransactionContext`, both referenced by the
`Executor` interface above but never defined until now:

``` java
public record QueryResult(List<Row> rows, int affectedRows) {

    public static QueryResult ofRows(List<Row> rows) {
        return new QueryResult(rows, rows.size());
    }

    public static QueryResult ofAffected(int count) {
        return new QueryResult(List.of(), count);
    }
}
```

``` java
public final class TransactionContext {

    // Deliberately empty for now. Real isolation, locking, and undo
    // arrive with the Transaction Manager (section 22). Until then,
    // every statement is its own implicit, auto-committed transaction.
    public static final TransactionContext AUTOCOMMIT = new TransactionContext();
}
```

------------------------------------------------------------------------

## 10.2 A Minimal StorageEngine to Execute Against

The executors need something real underneath them. Section 11 replaces
this with page-based disk storage; this is the "Level 1" in-memory version
`StorageEngine` -> `MemoryStorageEngine` pairing that section 94 (Liskov
Substitution) already assumes exists.

``` java
public interface StorageEngine {

    void createTable(TableSchema schema);

    void dropTable(String tableName);

    void insert(String tableName, Row row);

    List<Row> scan(String tableName);

    void replaceRows(String tableName, List<Row> rows);
}
```

``` java
public final class MemoryStorageEngine implements StorageEngine {

    private final Map<String, List<Row>> tables = new ConcurrentHashMap<>();

    @Override
    public void createTable(TableSchema schema) {
        tables.putIfAbsent(schema.tableName(), new CopyOnWriteArrayList<>());
    }

    @Override
    public void dropTable(String tableName) {
        tables.remove(tableName);
    }

    @Override
    public void insert(String tableName, Row row) {
        tables.get(tableName).add(row);
    }

    @Override
    public List<Row> scan(String tableName) {
        return List.copyOf(tables.get(tableName));
    }

    @Override
    public void replaceRows(String tableName, List<Row> rows) {
        tables.put(tableName, new CopyOnWriteArrayList<>(rows));
    }
}
```

`replaceRows` is a deliberately blunt instrument -- `UpdateExecutor` and
`DeleteExecutor` (below) compute a new row list in memory and swap the
whole thing in, rather than mutating individual rows. That is correct and
simple for this phase, and honestly wasteful once tables are large. Once
page-based storage and the buffer pool exist (sections 11-16), `UPDATE`
and `DELETE` rewrite individual pages/slots instead -- the executor's own
code barely changes; only what `StorageEngine` does underneath it does.

------------------------------------------------------------------------

## 10.3 The Six Executors, For Real

Each executor implements one, small, typed contract:

``` java
public interface StatementExecutor<S extends SqlStatement> {

    QueryResult execute(S statement, TransactionContext transaction);
}
```

**SelectExecutor**

``` java
public final class SelectExecutor implements StatementExecutor<SelectStatement> {

    private final StorageEngine storageEngine;

    public SelectExecutor(StorageEngine storageEngine) {
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(SelectStatement statement, TransactionContext transaction) {
        List<Row> matching = storageEngine.scan(statement.table()).stream()
                .filter(row -> statement.where() == null
                        || ExpressionEvaluator.matches(row, statement.where()))
                .toList();

        if (statement.columns().size() == 1 && statement.columns().get(0).equals("*")) {
            return QueryResult.ofRows(matching);
        }

        List<Row> projected = matching.stream()
                .map(row -> projectColumns(row, statement.columns()))
                .toList();
        return QueryResult.ofRows(projected);
    }

    private Row projectColumns(Row row, List<String> columns) {
        Map<String, Object> projected = new LinkedHashMap<>();
        for (String column : columns) {
            projected.put(column, row.get(column));
        }
        return new Row(projected);
    }
}
```

**InsertExecutor**

``` java
public final class InsertExecutor implements StatementExecutor<InsertStatement> {

    private final StorageEngine storageEngine;
    private final Catalog catalog;

    public InsertExecutor(StorageEngine storageEngine, Catalog catalog) {
        this.storageEngine = storageEngine;
        this.catalog = catalog;
    }

    @Override
    public QueryResult execute(InsertStatement statement, TransactionContext transaction) {
        TableSchema schema = catalog.getTable(statement.table());
        if (schema == null) {
            throw new IllegalStateException("No such table: " + statement.table());
        }
        if (statement.columns().size() != statement.values().size()) {
            throw new IllegalArgumentException("Column count does not match value count");
        }

        Map<String, Object> values = new LinkedHashMap<>();
        for (int i = 0; i < statement.columns().size(); i++) {
            values.put(statement.columns().get(i), statement.values().get(i));
        }
        validateAgainstSchema(schema, values);

        storageEngine.insert(statement.table(), new Row(values));
        return QueryResult.ofAffected(1);
    }

    private void validateAgainstSchema(TableSchema schema, Map<String, Object> values) {
        for (Column column : schema.columns()) {
            if (!column.nullable() && values.get(column.name()) == null) {
                throw new IllegalStateException(column.name() + " cannot be null");
            }
        }
    }
}
```

**UpdateExecutor**

``` java
public final class UpdateExecutor implements StatementExecutor<UpdateStatement> {

    private final StorageEngine storageEngine;

    public UpdateExecutor(StorageEngine storageEngine) {
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(UpdateStatement statement, TransactionContext transaction) {
        List<Row> current = storageEngine.scan(statement.table());
        List<Row> updated = new ArrayList<>();
        int affected = 0;

        for (Row row : current) {
            if (statement.where() == null || ExpressionEvaluator.matches(row, statement.where())) {
                Map<String, Object> newValues = new LinkedHashMap<>();
                for (String column : row.columnNames()) {
                    newValues.put(column, row.get(column));
                }
                newValues.putAll(statement.assignments());
                updated.add(new Row(newValues));
                affected++;
            } else {
                updated.add(row);
            }
        }

        storageEngine.replaceRows(statement.table(), updated);
        return QueryResult.ofAffected(affected);
    }
}
```

**DeleteExecutor**

``` java
public final class DeleteExecutor implements StatementExecutor<DeleteStatement> {

    private final StorageEngine storageEngine;

    public DeleteExecutor(StorageEngine storageEngine) {
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(DeleteStatement statement, TransactionContext transaction) {
        List<Row> current = storageEngine.scan(statement.table());
        List<Row> kept = new ArrayList<>();
        int affected = 0;

        for (Row row : current) {
            boolean shouldDelete = statement.where() == null
                    || ExpressionEvaluator.matches(row, statement.where());
            if (shouldDelete) {
                affected++;
            } else {
                kept.add(row);
            }
        }

        storageEngine.replaceRows(statement.table(), kept);
        return QueryResult.ofAffected(affected);
    }
}
```

**CreateTableExecutor**

``` java
public final class CreateTableExecutor implements StatementExecutor<CreateTableStatement> {

    private final Catalog catalog;
    private final StorageEngine storageEngine;

    public CreateTableExecutor(Catalog catalog, StorageEngine storageEngine) {
        this.catalog = catalog;
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(CreateTableStatement statement, TransactionContext transaction) {
        if (catalog.tableExists(statement.schema().tableName())) {
            throw new IllegalStateException("Table already exists: " + statement.schema().tableName());
        }
        catalog.createTable(statement.schema());
        storageEngine.createTable(statement.schema());
        return QueryResult.ofAffected(0);
    }
}
```

**DropTableExecutor**

``` java
public final class DropTableExecutor implements StatementExecutor<DropTableStatement> {

    private final Catalog catalog;
    private final StorageEngine storageEngine;

    public DropTableExecutor(Catalog catalog, StorageEngine storageEngine) {
        this.catalog = catalog;
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(DropTableStatement statement, TransactionContext transaction) {
        if (!catalog.tableExists(statement.table())) {
            throw new IllegalStateException("No such table: " + statement.table());
        }
        catalog.dropTable(statement.table());
        storageEngine.dropTable(statement.table());
        return QueryResult.ofAffected(0);
    }
}
```

Every executor above depends only on `Catalog` and `StorageEngine` --
interfaces, never `MemoryStorageEngine` directly -- which is exactly the
Dependency Inversion example section 96 already shows for `InsertExecutor`.
Swapping `MemoryStorageEngine` for a page-based `FileStorageEngine` later
means constructing the executors with a different object; not one line of
executor logic changes.

------------------------------------------------------------------------

## 10.4 Wiring It Together: The QueryEngine Dispatcher

This is the piece that turns six independent executor classes into one
callable database. It is also the direct answer to "how do lexer, parser,
and executors connect":

``` java
public final class QueryEngine {

    private final SelectExecutor selectExecutor;
    private final InsertExecutor insertExecutor;
    private final UpdateExecutor updateExecutor;
    private final DeleteExecutor deleteExecutor;
    private final CreateTableExecutor createTableExecutor;
    private final DropTableExecutor dropTableExecutor;

    public QueryEngine(Catalog catalog, StorageEngine storageEngine) {
        this.selectExecutor = new SelectExecutor(storageEngine);
        this.insertExecutor = new InsertExecutor(storageEngine, catalog);
        this.updateExecutor = new UpdateExecutor(storageEngine);
        this.deleteExecutor = new DeleteExecutor(storageEngine);
        this.createTableExecutor = new CreateTableExecutor(catalog, storageEngine);
        this.dropTableExecutor = new DropTableExecutor(catalog, storageEngine);
    }

    public QueryResult run(String sql) {
        List<Token> tokens = new SqlLexer(sql).tokenize();          // section 6
        SqlStatement statement = new SqlParser(tokens).parseStatement(); // section 7
        return execute(statement, TransactionContext.AUTOCOMMIT);
    }

    public QueryResult execute(SqlStatement statement, TransactionContext transaction) {
        return switch (statement) {
            case SelectStatement s -> selectExecutor.execute(s, transaction);
            case InsertStatement s -> insertExecutor.execute(s, transaction);
            case UpdateStatement s -> updateExecutor.execute(s, transaction);
            case DeleteStatement s -> deleteExecutor.execute(s, transaction);
            case CreateTableStatement s -> createTableExecutor.execute(s, transaction);
            case DropTableStatement s -> dropTableExecutor.execute(s, transaction);
        };
    }
}
```

Two details make this dispatcher worth pointing at directly:

``` text
1. SqlStatement is `sealed ... permits Select, Insert, Update, Delete,
   CreateTable, DropTable`. The switch above therefore needs NO default
   branch, and the compiler REFUSES TO COMPILE if a seventh statement
   type is ever added to the sealed interface without a matching case
   here. Adding CREATE INDEX later is a compile error until this switch
   is updated -- not a runtime surprise months after shipping it.

2. run(String sql) is the only method that touches Lexer/Parser at all.
   Every executor, and execute(SqlStatement, ...) itself, only ever sees
   the already-parsed AST. This is the actual architectural boundary:
   lexing and parsing are a text-processing concern; execution is a
   data-processing concern; QueryEngine.run is the one seam where they
   meet.
```

------------------------------------------------------------------------

## 10.5 Alternative: ANTLR-Generated Parser + the Visitor Pattern

Section 97 already lists "AST traversal" under the Visitor pattern.
This is what that looks like in a real system, and it is a legitimate,
production-proven alternative to sections 6-7's hand-written lexer and
parser -- Apache ShardingSphere's MySQL frontend, for one, is built
exactly this way:

``` java
// Uses an ANTLR-generated MySQLLexer/MySQLStatementParser instead of a
// hand-written one. ANTLR turns a .g4 grammar file into the lexer,
// parser, and a base Visitor class automatically.
public class MySqlParser implements IParser {
    MySqlStatementVisitor sqlStatementVisitor;

    public MySqlParser(String sql) {
        MySQLLexer lexer = new MySQLLexer(CharStreams.fromString(sql));
        MySQLStatementParser parser = new MySQLStatementParser(new CommonTokenStream(lexer));

        sqlStatementVisitor = new MySqlStatementVisitor(parser);
        sqlStatementVisitor.visit(parser.execute());
    }

    @Override
    public QueryData queryCmd() {
        return (QueryData) sqlStatementVisitor.getValue();
    }

    @Override
    public Object updateCmd() {
        return sqlStatementVisitor.getValue();
    }
}
```

``` java
// The Visitor walks ANTLR's generated parse tree and extracts exactly
// the fields needed into a domain object -- the ANTLR equivalent of
// this guide's SqlStatement records.
public class MySqlStatementVisitor extends MySQLStatementBaseVisitor {
    private final MySQLStatementParser parser;
    private COMMAND_TYPE commandType;
    private String tableName;

    @Override
    public Object visitCreateTable(MySQLStatementParser.CreateTableContext ctx) {
        commandType = COMMAND_TYPE.CREATE_TABLE;
        return super.visitCreateTable(ctx);
    }

    @Override
    public Object visitTableName(MySQLStatementParser.TableNameContext ctx) {
        this.tableName = ctx.name().getText();
        return super.visitTableName(ctx);
    }
}
```

The pipeline is structurally identical to section 10.1's, just built from
generated pieces instead of hand-written ones:

``` text
This guide (hand-written):      ANTLR-based (e.g. ShardingSphere):
SQL text                        SQL text
  |                               |
SqlLexer (hand-written)         MySQLLexer (generated from a grammar)
  |                               |
SqlParser (hand-written)        MySQLStatementParser (generated) + a
  |                             Visitor subclass that walks its parse tree
SqlStatement (a sealed          QueryData / InsertData / ModifyData
 interface of records)          (plain domain objects the Visitor builds)
  |                               |
QueryEngine.execute(...)         dispatch by commandType / object type
 (an exhaustive switch)          (structurally the same idea, without
                                  a sealed-type-checked switch, since
                                  ANTLR's generated node types are not a
                                  sealed hierarchy this codebase controls)
```

Why this guide does not start there: a hand-rolled recursive-descent
parser producing a sealed `SqlStatement` gets exhaustiveness checking
from the Java compiler itself (section 10.4's point 1) with zero extra
tooling. ANTLR earns its keep once the grammar gets large enough that
hand-writing and hand-maintaining a parser for it becomes the bigger
cost -- full dialect compatibility (MySQL, Postgres, and their many
edge cases) is exactly that situation, which is why real multi-dialect
tools reach for a grammar file and a generated parser instead. Know both;
pick the hand-written one for as long as the grammar in section 8 stays
small, and reach for ANTLR when it stops being small.

------------------------------------------------------------------------

# 11. Phase 6 --- Storage Engine

The first version can use:

``` text
Map<tableName, List<Row>>
```

Do not stop there.

The real database begins when rows are stored on disk.

Use Java NIO:

``` text
java.nio.file.Path
FileChannel
ByteBuffer
```

Store data as pages.

Recommended initial page size:

``` text
4096 bytes
```

or

``` text
8192 bytes
```

A table becomes:

``` text
Table
 |
 +-- Data File
      |
      +-- Page 0
      +-- Page 1
      +-- Page 2
      +-- ...
```

------------------------------------------------------------------------

# 12. Page-Based Storage

Create:

``` java
public final class Page {

    public static final int PAGE_SIZE = 8192;

    private final long pageId;
    private final ByteBuffer buffer;
}
```

A page should eventually contain:

``` text
Page Header
-------------------
pageId
pageType
LSN
freeSpaceOffset
recordCount
checksum

Records
-------------------
Record 1
Record 2
Record 3
...
```

This makes updates and recovery manageable.

------------------------------------------------------------------------

# 13. Record Format

Do not simply serialize Java objects.

Define your own stable binary format.

Example:

``` text
Record
-----------------
recordLength
transactionId
flags
columnCount

column1
column2
column3
...
```

For VARCHAR:

``` text
length
bytes
```

For BIGINT:

``` text
8 bytes
```

For BOOLEAN:

``` text
1 byte
```

The binary format becomes part of your database storage contract.

Version it early.

------------------------------------------------------------------------

# 14. Row Identifier

Every physical record should have a stable internal identifier.

For example:

``` java
public record RowId(
        long pageId,
        int slotId
) {}
```

Then:

``` text
RowId -> Page -> Slot -> Record
```

Indexes can point to RowId instead of copying complete rows.

------------------------------------------------------------------------

# 15. Buffer Pool

Do not read the same page from disk repeatedly.

Implement:

``` text
BufferPool
```

Responsibilities:

``` text
load page
cache page
mark dirty
flush page
evict page
```

Interface:

``` java
public interface BufferPool {

    Page get(PageId pageId);

    void markDirty(PageId pageId);

    void flush(PageId pageId);

    void flushAll();
}
```

Use an eviction strategy.

Initially:

``` text
LRU
```

Later:

``` text
Clock
2Q
LFU
adaptive policies
```

------------------------------------------------------------------------

# 16. Free Space Management

The database must know where a new record can be inserted.

Implement:

``` text
FreePageList
```

or:

``` text
Free Space Map
```

Example:

``` text
Page 10 -> 30% free
Page 11 -> 5% free
Page 12 -> 70% free
```

An insert can select Page 12.

------------------------------------------------------------------------

# 17. Indexing

Without indexes:

``` sql
SELECT *
FROM employee
WHERE id = 100;
```

requires:

``` text
scan every row
```

This is:

``` text
O(N)
```

For primary-key lookup we want approximately:

``` text
O(log N)
```

Implement:

``` text
B+Tree
```

first.

Later:

``` text
Hash Index
Bitmap Index
Full Text Index
```

------------------------------------------------------------------------

# 18. B+Tree

Structure:

``` text
                  Root
                 /    \
              Node    Node
             /  \      / \
           Leaf Leaf  Leaf Leaf
```

Leaf nodes contain:

``` text
key -> RowId
```

Example:

``` text
100 -> (page=20, slot=3)
101 -> (page=20, slot=4)
102 -> (page=21, slot=1)
```

B+Tree is especially useful for:

``` sql
WHERE id = 100

WHERE id > 100

WHERE id BETWEEN 100 AND 200

ORDER BY id
```

------------------------------------------------------------------------

# 19. Query Planner

Do not execute every query as a table scan.

Given:

``` sql
SELECT *
FROM employee
WHERE id = 100;
```

The planner should inspect indexes.

Plan:

``` text
IndexScan(employee_pk, 100)
```

instead of:

``` text
TableScan(employee)
Filter(id = 100)
```

Introduce:

``` java
public interface QueryPlanner {

    ExecutionPlan plan(SqlStatement statement);
}
```

Later add:

``` text
cost estimation
statistics
selectivity
join ordering
index selection
```

Section 10 executed every statement directly against `StorageEngine`,
scanning and filtering a fully-materialized `List<Row>` in one pass. That
was explicitly a Level 1 shortcut -- section 10's own text promised that
"once the planner and query operators (sections 19-20) exist, these same
executors are adapted to consume a plan instead of raw statements." The
rest of this section, and section 20, deliver on that promise for real.

------------------------------------------------------------------------

## 19.1 The Three Parts of a Query Engine

A query engine is not one component. It is three, each with a distinct
job:

``` text
Query Optimizer      -- decides HOW to answer a query (which plan, which indexes)
Execution Engine      -- actually reads rows/columns according to that plan
Catalog / Metadata     -- knows what tables and indexes exist, and their shape
Manager
```

This section (19) builds the Optimizer and the Catalog Manager. Section
20 builds the Execution Engine. The three are separate concerns for the
same reason section 95 (Interface Segregation) already argues for:
`QueryPlanner`, `QueryExecutor`, and `Catalog` are three different
interfaces, not one `DatabaseEverything` God interface.

The IndexScan-vs-TableScan-plus-Filter comparison already shown above IS
an optimizer decision -- specifically, **Rule-Based Optimization (RBO)**:
a fixed rule ("if the `WHERE` clause matches an indexed column, use the
index") applied without comparing alternatives. A **Cost-Based Optimizer
(CBO)** generalizes this: instead of one hard-coded rule, it generates
several candidate plans and picks whichever has the lowest *estimated*
cost, using statistics the Catalog Manager collects (row counts, blocks
accessed, selectivity). RBO is simpler to build and reason about; CBO
scales better as the number of indexes and join orderings grows, at the
cost of needing real statistics to estimate against. Start with RBO --
this section builds RBO's actual mechanism.

------------------------------------------------------------------------

## 19.2 The Plan Interface: Relational Algebra Plus Cost

A `Plan` describes *how* a piece of relational algebra will be executed,
and can report an estimated cost for it -- exactly what a Cost-Based
Optimizer (19.1) needs to compare alternatives:

``` java
public interface Plan {

    Scan open();

    int blocksAccessed();

    int recordsOutput();

    TableSchema schema();
}
```

`open()` returns a `Scan` (section 20) -- calling `open()` does not read
any data yet, it only prepares to. `blocksAccessed()`/`recordsOutput()`
are cost estimates a CBO would compare across candidate plans; for now,
with only one strategy per statement, they exist but nothing yet chooses
between competing values.

**TablePlan** -- the only `Plan` that talks to `StorageEngine` directly:

``` java
public final class TablePlan implements Plan {

    private final String tableName;
    private final StorageEngine storageEngine;
    private final Catalog catalog;

    public TablePlan(String tableName, StorageEngine storageEngine, Catalog catalog) {
        this.tableName = tableName;
        this.storageEngine = storageEngine;
        this.catalog = catalog;
    }

    @Override
    public Scan open() {
        return storageEngine.openScan(tableName); // section 20.2 -- the ONE seam into physical storage
    }

    @Override
    public int blocksAccessed() {
        return catalog.getStatistics(tableName).blockCount(); // fed by whatever tracks physical page counts, sections 12-16
    }

    @Override
    public int recordsOutput() {
        return catalog.getStatistics(tableName).recordCount();
    }

    @Override
    public TableSchema schema() {
        return catalog.getTable(tableName);
    }
}
```

**SelectPlan** and **ProjectPlan** -- each wraps another `Plan`, the way a
Decorator wraps another object, rather than talking to storage itself:

``` java
public final class SelectPlan implements Plan {

    private final Plan input;
    private final Expression predicate;

    public SelectPlan(Plan input, Expression predicate) {
        this.input = input;
        this.predicate = predicate;
    }

    @Override
    public Scan open() {
        return new SelectScan(input.open(), predicate); // section 20.2
    }

    @Override
    public int blocksAccessed() {
        return input.blocksAccessed(); // filtering reads the SAME blocks as its input -- no extra I/O
    }

    @Override
    public int recordsOutput() {
        return input.recordsOutput() / 2; // a naive fixed selectivity guess -- a CBO replaces this with real statistics
    }

    @Override
    public TableSchema schema() {
        return input.schema(); // filtering ROWS never changes which COLUMNS exist
    }
}
```

``` java
public final class ProjectPlan implements Plan {

    private final Plan input;
    private final List<String> fields;

    public ProjectPlan(Plan input, List<String> fields) {
        this.input = input;
        this.fields = fields;
    }

    @Override
    public Scan open() {
        return new ProjectScan(input.open(), fields); // section 20.2
    }

    @Override
    public int blocksAccessed() { return input.blocksAccessed(); }

    @Override
    public int recordsOutput() { return input.recordsOutput(); } // projecting columns doesn't change row COUNT

    @Override
    public TableSchema schema() {
        return input.schema().project(fields); // a NARROWER schema -- fewer columns
    }
}
```

`TableSchema.project(fields)` is one small addition needed on the record
from section 4:

``` java
public record TableSchema(String tableName, List<Column> columns) {

    public TableSchema project(List<String> fieldNames) {
        List<Column> projected = columns.stream()
                .filter(c -> fieldNames.contains(c.name()))
                .toList();
        return new TableSchema(tableName, projected);
    }

    public List<String> fieldNames() {
        return columns.stream().map(Column::name).toList();
    }
}
```

`TablePlan`, `SelectPlan`, and `ProjectPlan` compose exactly the way
`SelectStatement`'s three parts (table, where, columns) already suggest:
read, then filter, then narrow columns -- each step wrapping the previous
`Plan`, never reaching around it to touch storage directly.

------------------------------------------------------------------------

## 19.3 BasicQueryPlanner and UpdatePlanner: From SqlStatement to a Plan

``` java
public interface QueryPlanner {

    Plan createPlan(SelectStatement statement, TransactionContext tx);
}
```

``` java
public final class BasicQueryPlanner implements QueryPlanner {

    private final Catalog catalog;
    private final StorageEngine storageEngine;

    public BasicQueryPlanner(Catalog catalog, StorageEngine storageEngine) {
        this.catalog = catalog;
        this.storageEngine = storageEngine;
    }

    @Override
    public Plan createPlan(SelectStatement statement, TransactionContext tx) {
        Plan plan = new TablePlan(statement.table(), storageEngine, catalog); // Step 1: read

        if (statement.where() != null) {
            plan = new SelectPlan(plan, statement.where()); // Step 2: filter rows
        }
        boolean selectStar = statement.columns().size() == 1 && statement.columns().get(0).equals("*");
        if (!selectStar) {
            plan = new ProjectPlan(plan, statement.columns()); // Step 3: filter columns
        }
        return plan;
    }
}
```

Compare this three-step body to `SelectExecutor` in section 10.3 -- the
*logic* (read, then filter, then project) is identical. What changed is
that each step is now a composed, reusable `Plan` object instead of an
inline `.stream().filter(...)` call -- the exact shape a Cost-Based
Optimizer needs later, since it can only compare plans it can hold as
objects, not inline expressions buried in an executor's method body.

``` java
public interface UpdatePlanner {

    int executeInsert(InsertStatement statement, TransactionContext tx);
    int executeUpdate(UpdateStatement statement, TransactionContext tx);
    int executeDelete(DeleteStatement statement, TransactionContext tx);
    int executeCreateTable(CreateTableStatement statement, TransactionContext tx);
    int executeDropTable(DropTableStatement statement, TransactionContext tx);
}
```

``` java
public final class BasicUpdatePlanner implements UpdatePlanner {

    private final Catalog catalog;
    private final StorageEngine storageEngine;

    public BasicUpdatePlanner(Catalog catalog, StorageEngine storageEngine) {
        this.catalog = catalog;
        this.storageEngine = storageEngine;
    }

    @Override
    public int executeInsert(InsertStatement statement, TransactionContext tx) {
        Plan plan = new TablePlan(statement.table(), storageEngine, catalog);
        Scan scan = plan.open();                 // section 20 -- a WRITABLE scan
        scan.insert();                            // position at a fresh slot
        for (int i = 0; i < statement.columns().size(); i++) {
            scan.setVal(statement.columns().get(i), statement.values().get(i));
        }
        scan.close();
        return 1;
    }

    @Override
    public int executeUpdate(UpdateStatement statement, TransactionContext tx) {
        Plan plan = new TablePlan(statement.table(), storageEngine, catalog);
        if (statement.where() != null) plan = new SelectPlan(plan, statement.where());
        Scan scan = plan.open();
        int affected = 0;
        while (scan.next()) {
            for (var entry : statement.assignments().entrySet()) {
                scan.setVal(entry.getKey(), entry.getValue());
            }
            affected++;
        }
        scan.close();
        return affected;
    }

    @Override
    public int executeDelete(DeleteStatement statement, TransactionContext tx) {
        Plan plan = new TablePlan(statement.table(), storageEngine, catalog);
        if (statement.where() != null) plan = new SelectPlan(plan, statement.where());
        Scan scan = plan.open();
        int affected = 0;
        while (scan.next()) {
            scan.delete();
            affected++;
        }
        scan.close();
        return affected;
    }

    @Override
    public int executeCreateTable(CreateTableStatement statement, TransactionContext tx) {
        catalog.createTable(statement.schema());
        storageEngine.createTable(statement.schema());
        return 0;
    }

    @Override
    public int executeDropTable(DropTableStatement statement, TransactionContext tx) {
        catalog.dropTable(statement.table());
        storageEngine.dropTable(statement.table());
        return 0;
    }
}
```

Notice `executeUpdate`/`executeDelete` reuse `SelectPlan` for their
`WHERE` clause -- filtering which rows to mutate is the *identical*
operation as filtering which rows to read, so it is composed from the
same `Plan`, not reimplemented.

------------------------------------------------------------------------

## 19.4 BasicPlanner and QueryEngine: Wiring Parsing, Planning, and Transactions

`BasicPlanner` is a **Facade** (section 97) sitting between the parser and
the two planners above -- exactly the same seam idea as section 10.4's
`QueryEngine`, one layer deeper now that statements produce `Plan`s
instead of being executed immediately:

``` java
public final class BasicPlanner {

    private final QueryPlanner queryPlanner;
    private final UpdatePlanner updatePlanner;

    public BasicPlanner(QueryPlanner queryPlanner, UpdatePlanner updatePlanner) {
        this.queryPlanner = queryPlanner;
        this.updatePlanner = updatePlanner;
    }

    public Plan createQueryPlan(String sql, TransactionContext tx) {
        SqlStatement statement = parse(sql);
        if (!(statement instanceof SelectStatement select)) {
            throw new IllegalArgumentException("Not a query: " + sql);
        }
        return queryPlanner.createPlan(select, tx);
    }

    public int executeUpdate(String sql, TransactionContext tx) {
        SqlStatement statement = parse(sql);
        return switch (statement) {
            case InsertStatement s -> updatePlanner.executeInsert(s, tx);
            case UpdateStatement s -> updatePlanner.executeUpdate(s, tx);
            case DeleteStatement s -> updatePlanner.executeDelete(s, tx);
            case CreateTableStatement s -> updatePlanner.executeCreateTable(s, tx);
            case DropTableStatement s -> updatePlanner.executeDropTable(s, tx);
            case SelectStatement s -> throw new IllegalArgumentException("Use createQueryPlan for a SELECT");
        };
    }

    private SqlStatement parse(String sql) {
        List<Token> tokens = new SqlLexer(sql).tokenize();   // section 6
        return new SqlParser(tokens).parseStatement();        // section 7
    }
}
```

This is the exact same exhaustive-`switch`-over-a-sealed-interface trick
section 10.4 already used for `QueryEngine.execute` -- adding a `CREATE
INDEX` statement later means the compiler refuses to build this method
until a matching `case` is added here too, same as before.

The externally-visible database facade -- what a CLI (section 41-42) or a
JDBC driver (section 37) actually calls:

``` java
public final class BasicQueryEngine {

    private final BasicPlanner planner;
    private final TransactionManager transactionManager; // section 22

    public BasicQueryEngine(BasicPlanner planner, TransactionManager transactionManager) {
        this.planner = planner;
        this.transactionManager = transactionManager;
    }

    public QueryResult doQuery(String sql) {
        TransactionContext tx = transactionManager.begin();
        Plan plan = planner.createQueryPlan(sql, tx);
        Scan scan = plan.open();

        List<Row> rows = new ArrayList<>();
        while (scan.next()) {
            Map<String, Object> values = new LinkedHashMap<>();
            for (String field : plan.schema().fieldNames()) {
                values.put(field, scan.getVal(field));
            }
            rows.add(new Row(values));
        }
        scan.close();
        transactionManager.commit(tx);
        return QueryResult.ofRows(rows);
    }

    public QueryResult doUpdate(String sql) {
        TransactionContext tx = transactionManager.begin();
        int affected = planner.executeUpdate(sql, tx);
        transactionManager.commit(tx);
        return QueryResult.ofAffected(affected);
    }
}
```

`doQuery` is the only place left that walks a `Scan` into a `List<Row>` --
and it does that only because the CLI/JDBC layer above it wants a
materialized result set to print or return. The engine itself, all the
way down through `Plan`/`Scan`, never materializes more than one row at a
time (section 20.1 explains exactly why that matters).

------------------------------------------------------------------------

## 19.5 Adapting Section 10's Executors to Use the Planner

Section 10 promised this adaptation explicitly. Here it is, for one
executor -- the rest follow the identical shape:

``` java
// SelectExecutor, REVISITED: same StatementExecutor<SelectStatement> contract as section 10.3,
// but the BODY now delegates to the planner instead of scanning+filtering a materialized List<Row> by hand.
public final class SelectExecutor implements StatementExecutor<SelectStatement> {

    private final QueryPlanner queryPlanner;

    public SelectExecutor(QueryPlanner queryPlanner) {
        this.queryPlanner = queryPlanner;
    }

    @Override
    public QueryResult execute(SelectStatement statement, TransactionContext transaction) {
        Plan plan = queryPlanner.createPlan(statement, transaction);
        Scan scan = plan.open();

        List<Row> rows = new ArrayList<>();
        while (scan.next()) {
            Map<String, Object> values = new LinkedHashMap<>();
            for (String field : plan.schema().fieldNames()) {
                values.put(field, scan.getVal(field));
            }
            rows.add(new Row(values));
        }
        scan.close();
        return QueryResult.ofRows(rows);
    }
}
```

The public contract did not change at all. Only what happens *inside*
`execute` changed -- from "load everything, then filter in Java" to
"compose a plan, then stream through it." This is precisely why section
10.3 was told to depend on `Catalog`/`StorageEngine` *interfaces*: the
same Dependency Inversion (section 96) that let `MemoryStorageEngine` be
swapped for a `FileStorageEngine` later also lets an executor's
*implementation strategy* change without its callers noticing.

------------------------------------------------------------------------

## 19.6 The Catalog Manager, Properly: Persisting Schema as Data

Section 5's `Catalog` interface is metadata-complete but storage-naive --
nothing says *where* a `TableSchema` actually lives once created. A toy
answer is a `Map<String, TableSchema>` in memory, which forgets every
table the instant the process restarts. A real database does something
more interesting: it stores its **own schema as ordinary rows**, in
special tables the database itself manages, so table definitions get the
exact same durability (section 24's WAL) as user data, instead of a
separate, ad hoc persistence mechanism.

``` text
tinydb_tables               tinydb_columns
+-----------+-----------+   +-----------+-----------+--------+--------+
| tblname   | slotsize  |   | tblname   | fldname   | type   | length |
+-----------+-----------+   +-----------+-----------+--------+--------+
| employee  | 48        |   | employee  | id        | BIGINT | 0      |
|           |           |   | employee  | name      | VARCHAR| 50     |
+-----------+-----------+   +-----------+-----------+--------+--------+
```

``` java
public final class PersistentCatalog implements Catalog {

    private static final TableSchema TABLES_SCHEMA = new TableSchema("tinydb_tables", List.of(
            new Column("tblname", DataType.VARCHAR, false),
            new Column("slotsize", DataType.INT, false)));

    private static final TableSchema COLUMNS_SCHEMA = new TableSchema("tinydb_columns", List.of(
            new Column("tblname", DataType.VARCHAR, false),
            new Column("fldname", DataType.VARCHAR, false),
            new Column("type", DataType.VARCHAR, false),
            new Column("length", DataType.INT, false)));

    private final StorageEngine storageEngine;

    public PersistentCatalog(StorageEngine storageEngine) {
        this.storageEngine = storageEngine;
        storageEngine.createTable(TABLES_SCHEMA);   // bootstrap: the catalog's own tables must exist first
        storageEngine.createTable(COLUMNS_SCHEMA);
    }

    @Override
    public void createTable(TableSchema schema) {
        storageEngine.insert(TABLES_SCHEMA.tableName(),
                new Row(Map.of("tblname", schema.tableName(), "slotsize", estimateSlotSize(schema))));
        for (Column column : schema.columns()) {
            storageEngine.insert(COLUMNS_SCHEMA.tableName(), new Row(Map.of(
                    "tblname", schema.tableName(),
                    "fldname", column.name(),
                    "type", column.type().name(),
                    "length", 0)));
        }
    }

    @Override
    public TableSchema getTable(String tableName) {
        boolean exists = storageEngine.scan(TABLES_SCHEMA.tableName()).stream()
                .anyMatch(row -> row.get("tblname").equals(tableName));
        if (!exists) return null;

        List<Column> columns = storageEngine.scan(COLUMNS_SCHEMA.tableName()).stream()
                .filter(row -> row.get("tblname").equals(tableName))
                .map(row -> new Column((String) row.get("fldname"),
                        DataType.valueOf((String) row.get("type")), true))
                .toList();
        return new TableSchema(tableName, columns);
    }

    // dropTable/tableExists/listTables/createDatabase omitted for brevity -- same
    // "read from tinydb_tables / tinydb_columns" pattern as getTable() above.
    // estimateSlotSize(...) is covered next, in 19.7.
}
```

The bootstrap step in the constructor -- creating `tinydb_tables` and
`tinydb_columns` *using the same `storageEngine.createTable` every other
table uses* -- is the interesting part: the catalog describes every
table, **including its own two tables**, through one uniform mechanism.
This is the same self-describing idea real databases use (Postgres's
`pg_catalog`, MySQL's `information_schema`).

------------------------------------------------------------------------

## 19.7 TablePhysicalLayout: From Logical Schema to Byte Offsets

`TableSchema` (section 4) is a table's *logical* shape -- names, types,
nullability. Section 13's binary record format needs one more thing: the
exact byte **offset** of each column within a row, and the row's total
**slot size** -- its physical layout.

``` java
public final class TablePhysicalLayout {

    private final TableSchema schema;
    private final Map<String, Integer> offsets;
    private final int slotSize;

    public TablePhysicalLayout(TableSchema schema) {
        this.schema = schema;
        this.offsets = new HashMap<>();
        int position = 0;
        for (Column column : schema.columns()) {
            offsets.put(column.name(), position);
            position += sizeInBytes(column);
        }
        this.slotSize = position;
    }

    private int sizeInBytes(Column column) {
        return switch (column.type()) {
            case INT -> 4;
            case BIGINT -> 8;
            case BOOLEAN -> 1;
            case VARCHAR -> 50; // fixed-length slots for simplicity, per section 13 -- even VARCHAR gets a fixed budget
            case DECIMAL -> 12;
            case DATE, TIMESTAMP -> 8;
        };
    }

    public int offset(String fieldName) { return offsets.get(fieldName); }
    public int slotSize() { return slotSize; }
}
```

`estimateSlotSize(schema)` in 19.6's `PersistentCatalog.createTable` is
exactly `new TablePhysicalLayout(schema).slotSize()` -- the catalog
records this number once, at table-creation time, so `TableScan` (section
20.2) never has to recompute it on every single row read.

Fixed-length slots (even for `VARCHAR`) is the same simplifying decision
section 13 already made -- worth restating here because it's precisely
what makes `offset(fieldName)` a single, cheap arithmetic lookup instead
of "scan every earlier column to find where this one starts."

------------------------------------------------------------------------

# 20. Query Operators

Model execution as operators:

``` text
TableScan
IndexScan
Filter
Project
Sort
Limit
Aggregate
HashJoin
NestedLoopJoin
```

Example:

``` text
SELECT name
FROM employee
WHERE salary > 50000
ORDER BY salary DESC
LIMIT 10
```

Plan:

``` text
Limit
  |
Sort
  |
Filter salary > 50000
  |
TableScan employee
```

This architecture makes the query engine extensible.

Section 19 built the `Plan` side of this diagram (`TablePlan`, `SelectPlan`,
`ProjectPlan`). This is the Execution Engine part of the query engine
(section 19.1) -- the `Scan` objects each `Plan.open()` actually returns,
and the ones that make `Filter`/`Project` above real, running code
instead of just boxes in a diagram.

------------------------------------------------------------------------

## 20.1 The Scan Interface: Streaming Rows Instead of Materializing Lists

Section 10.2's `StorageEngine.scan(tableName)` returns a `List<Row>` --
every row, loaded into memory, before the executor even looks at the
first one. That is fine for a table that fits comfortably in memory; it
is a real problem the moment a table doesn't. A `Scan` fixes this by
being an **iterator**, not a collection -- it produces rows one at a
time, on demand:

``` java
public interface Scan {

    boolean next();

    Object getVal(String fieldName);

    boolean hasField(String fieldName);

    void close();

    // Writable operations -- see the note below on Interface Segregation
    void insert();
    void setVal(String fieldName, Object value);
    void delete();
}
```

Putting both read and write methods on one `Scan` interface is a
pragmatic simplification, not the final answer -- section 95 (Interface
Segregation) already argues against exactly this shape in the abstract.
A stricter design splits this into `ReadOnlyScan` (`next`/`getVal`/
`hasField`/`close`) and `UpdatableScan extends ReadOnlyScan` (adding
`insert`/`setVal`/`delete`), so a read-only `SELECT` path is never even
*able* to call `delete()` by accident -- the compiler enforces it instead
of a runtime check. Section 20.2's implementations work identically
either way; only which interface each one is declared to return changes.

------------------------------------------------------------------------

## 20.2 TableScan, SelectScan, ProjectScan: Real Implementations

**TableScan** is the only `Scan` that talks to `StorageEngine` directly --
everything else wraps another `Scan`:

``` java
// StorageEngine (section 10.2) gains one new method alongside insert()/scan()/replaceRows():
public interface StorageEngine {
    // ... createTable, dropTable, insert, scan, replaceRows from section 10.2 ...
    Scan openScan(String tableName);
}
```

``` java
// MemoryStorageEngine.openScan -- still backed by the in-memory List<Row> for now.
// Once page-based storage (sections 11-16) exists, this walks Pages via RowId (section 14)
// instead of a Java Iterator -- the Scan CONTRACT above does not change either way.
@Override
public Scan openScan(String tableName) {
    List<Row> rows = tables.get(tableName);
    return new Scan() {
        private int index = -1;

        @Override
        public boolean next() {
            index++;
            return index < rows.size();
        }

        @Override
        public Object getVal(String fieldName) {
            return rows.get(index).get(fieldName);
        }

        @Override
        public boolean hasField(String fieldName) {
            return rows.get(index).columnNames().contains(fieldName);
        }

        @Override
        public void insert() {
            rows.add(new Row(new HashMap<>()));
            index = rows.size() - 1;
        }

        @Override
        public void setVal(String fieldName, Object value) {
            Map<String, Object> updated = new LinkedHashMap<>();
            for (String column : rows.get(index).columnNames()) updated.put(column, rows.get(index).get(column));
            updated.put(fieldName, value);
            rows.set(index, new Row(updated));
        }

        @Override
        public void delete() {
            rows.remove(index);
            index--; // the next next() call must not skip the row that just slid into this slot
        }

        @Override
        public void close() { }
    };
}
```

**SelectScan** wraps a child `Scan` and a predicate -- it filters rows as
they stream past, one at a time, never holding more than the current row:

``` java
public final class SelectScan implements Scan {

    private final Scan input;
    private final Expression predicate; // section 10.1

    public SelectScan(Scan input, Expression predicate) {
        this.input = input;
        this.predicate = predicate;
    }

    @Override
    public boolean next() {
        while (input.next()) {
            if (matchesCurrentRow()) return true;
        }
        return false;
    }

    private boolean matchesCurrentRow() {
        return ExpressionEvaluator.matchesScan(this, predicate); // see the ExpressionEvaluator extension just below
    }

    @Override public Object getVal(String fieldName) { return input.getVal(fieldName); }
    @Override public boolean hasField(String fieldName) { return input.hasField(fieldName); }
    @Override public void setVal(String fieldName, Object value) { input.setVal(fieldName, value); }
    @Override public void insert() { input.insert(); }
    @Override public void delete() { input.delete(); }
    @Override public void close() { input.close(); }
}
```

`SelectScan` needs to evaluate the same `Expression` tree section 10.1
already built -- but against a `Scan`'s current position, not a
fully-formed `Row`. Extend `ExpressionEvaluator` with a second entry
point that mirrors `matches(Row, Expression)` field for field, reading
through `Scan.getVal(...)` instead of `Row.get(...)`:

``` java
// Added to ExpressionEvaluator (section 10.1) once Scan exists to evaluate against:
public static boolean matchesScan(Scan scan, Expression expression) {
    return switch (expression) {
        case Comparison c -> evaluateComparison(scan.getVal(c.column()), c);
        case And a -> matchesScan(scan, a.left()) && matchesScan(scan, a.right());
        case Or o -> matchesScan(scan, o.left()) || matchesScan(scan, o.right());
    };
}

// evaluateComparison is refactored to take the ACTUAL value directly, so both matches(Row, ...)
// and matchesScan(Scan, ...) share this one comparison method instead of duplicating it:
private static boolean evaluateComparison(Object actual, Comparison c) {
    int cmp = compare(actual, c.value());
    return switch (c.operator()) {
        case "=" -> Objects.equals(actual, c.value());
        case "!=" -> !Objects.equals(actual, c.value());
        case ">" -> cmp > 0;
        case ">=" -> cmp >= 0;
        case "<" -> cmp < 0;
        case "<=" -> cmp <= 0;
        default -> throw new IllegalStateException("Unsupported operator: " + c.operator());
    };
}
```

`matches(Row, Expression)` (section 10.1) is updated to call this same
refactored `evaluateComparison(row.get(c.column()), c)` -- one comparison
implementation, two entry points, exactly the kind of small refactor that
keeps `Row`-based execution (section 10) and `Scan`-based execution
(section 19-20) from silently drifting into two different definitions of
"matches."

**ProjectScan** wraps a child `Scan` and a field list -- it filters
*columns*, and explicitly rejects access to any column outside the
projection, exactly the same "not part of this schema" guard
`ProjectPlan.schema()` (section 19.2) already narrows to:

``` java
public final class ProjectScan implements Scan {

    private final Scan input;
    private final List<String> fields;

    public ProjectScan(Scan input, List<String> fields) {
        this.input = input;
        this.fields = fields;
    }

    @Override
    public boolean next() { return input.next(); }

    @Override
    public Object getVal(String fieldName) {
        if (!hasField(fieldName)) throw new RuntimeException("field " + fieldName + " not found.");
        return input.getVal(fieldName);
    }

    @Override
    public boolean hasField(String fieldName) { return fields.contains(fieldName); }

    @Override public void setVal(String fieldName, Object value) { throw new UnsupportedOperationException("read-only projection"); }
    @Override public void insert() { throw new UnsupportedOperationException("read-only projection"); }
    @Override public void delete() { throw new UnsupportedOperationException("read-only projection"); }
    @Override public void close() { input.close(); }
}
```

`ProjectScan` throwing on every write method is worth noticing --
projection is fundamentally a read-shaping operation, and section 20.1's
"split into ReadOnlyScan/UpdatableScan" refinement would let this class
simply not implement the write methods at all, rather than implementing
them just to reject every call.

------------------------------------------------------------------------

## 20.3 Why Composing Scans Is Cheaper Than Composing Materialized Lists

Trace what happens for:

``` sql
SELECT name FROM employee WHERE salary > 50000;
```

against a table of one million rows where ten thousand match:

``` text
Section 10.3's original SelectExecutor:
  storageEngine.scan("employee")                 -- loads ALL 1,000,000 rows into a List<Row>
    .stream().filter(...)                          -- then filters down to 10,000, still in memory
    .map(...)                                       -- then projects, producing a THIRD list

Section 19-20's Plan/Scan pipeline:
  TableScan.next()  -> SelectScan.next()  -> ProjectScan.next()
  -- pulls ONE row at a time through all three stages, filters it, projects it, and only
     the caller (BasicQueryEngine.doQuery, section 19.4) decides whether to keep it in a
     List -- the pipeline itself never holds more than one row per stage at once.
```

Both eventually produce the same 10,000-row result. The difference is
peak memory: one materializes three full intermediate collections along
the way; the other never holds more than a handful of rows at any given
instant, regardless of whether the table has a thousand rows or a
billion. This is the concrete, measurable payoff of the Iterator-shaped
`Scan` interface (section 20.1) over section 10's original "load a list,
then `.stream()` it" approach -- and it is also exactly why `Sort` and
`Aggregate` (the two operators in this section's very first list that
still need building) are structurally different from `Filter`/`Project`:
sorting the whole result requires seeing every row before producing the
first one, so a `SortScan` cannot stream the way `SelectScan`/
`ProjectScan` do -- a real, unavoidable limit on how far the streaming
idea extends, worth stating honestly rather than implying every operator
can be lazy for free.

------------------------------------------------------------------------

# 21. Constraints

Implement:

``` text
PRIMARY KEY
UNIQUE
NOT NULL
DEFAULT
CHECK
FOREIGN KEY
```

Example:

``` sql
CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    email VARCHAR(200) UNIQUE,
    salary DECIMAL(12,2) CHECK (salary >= 0)
);
```

Constraint validation should happen before committing a transaction.

------------------------------------------------------------------------

# 22. Transaction Manager

Now introduce transactions.

API:

``` java
public interface TransactionManager {

    Transaction begin();

    void commit(Transaction transaction);

    void rollback(Transaction transaction);
}
```

SQL:

``` sql
BEGIN;

UPDATE employee
SET salary = 120000
WHERE id = 1;

ROLLBACK;
```

The salary must return to its previous value.

------------------------------------------------------------------------

# 23. ACID

TinyDB should explicitly implement:

``` text
A — Atomicity
C — Consistency
I — Isolation
D — Durability
```

These should not be treated as a single feature.

They are implemented through several subsystems.

``` text
Atomicity  -> WAL + transaction state + undo/MVCC
Consistency -> constraints + transaction rules
Isolation  -> locks and/or MVCC
Durability -> WAL + fsync + recovery
```

------------------------------------------------------------------------

# 24. Write-Ahead Logging

WAL is one of the most important components.

Rule:

> The database must persist the log record before the corresponding
> dirty data page is considered durable.

Example:

``` text
UPDATE employee
SET salary = 120000
WHERE id = 1;
```

WAL:

``` text
BEGIN TX 10

UPDATE
table=employee
row=1
oldSalary=100000
newSalary=120000

COMMIT TX 10
```

Disk ordering:

``` text
1. Write WAL
2. Flush WAL
3. Write data page
4. Flush data page later
```

If the server crashes after step 2:

``` text
WAL exists
data page may not
```

Recovery can reconstruct the intended state.

------------------------------------------------------------------------

# 25. Log Sequence Number

Give every WAL record an LSN.

``` java
public record Lsn(long value) {}
```

Every page stores its latest applied LSN.

During recovery:

``` text
if WAL LSN > page LSN
    replay WAL
```

This is the foundation for crash recovery.

------------------------------------------------------------------------

# 26. WAL Record Types

Start with:

``` text
BEGIN
INSERT
UPDATE
DELETE
COMMIT
ABORT
CHECKPOINT
```

Later:

``` text
CREATE_TABLE
DROP_TABLE
CREATE_INDEX
DROP_INDEX
ALTER_TABLE
```

------------------------------------------------------------------------

# 27. Rollback Strategy

There are two major approaches.

## Approach A --- Undo logging

Store:

``` text
old value
```

For:

``` sql
UPDATE employee
SET salary = 120000
WHERE id = 1;
```

record:

``` text
old = 100000
new = 120000
```

Rollback writes:

``` text
salary = 100000
```

## Approach B --- MVCC

Keep multiple row versions.

``` text
employee id=1

Version 1:
salary=100000
xmin=10
xmax=20

Version 2:
salary=120000
xmin=20
xmax=null
```

Transactions see the version appropriate to their snapshot.

**Recommendation:** implement a simple undo/WAL system first, then
evolve toward MVCC.

------------------------------------------------------------------------

# 28. MVCC

MVCC means Multi-Version Concurrency Control.

Instead of modifying one row in place:

``` text
old row
   |
new row version
```

Track:

``` text
xmin
xmax
transaction status
```

Example:

``` text
Row Version A
xmin = 10
xmax = 20

Row Version B
xmin = 20
xmax = null
```

Transaction 10 can see A.

Transaction 20 can see B after the appropriate visibility rules.

------------------------------------------------------------------------

# 29. Isolation Levels

Support:

``` text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

Do not implement them merely as enum values.

Each isolation level requires actual visibility/locking behavior.

Possible implementation:

``` text
READ COMMITTED
    -> statement-level snapshot

REPEATABLE READ
    -> transaction-level snapshot

SERIALIZABLE
    -> strict locking or serializable MVCC
```

------------------------------------------------------------------------

# 30. Lock Manager

Create:

``` java
public interface LockManager {

    void acquire(
            TransactionId transactionId,
            ResourceId resource,
            LockMode mode
    );

    void releaseAll(TransactionId transactionId);
}
```

Lock modes:

``` text
SHARED
EXCLUSIVE
```

Compatibility:

``` text
S + S -> allowed
S + X -> blocked
X + S -> blocked
X + X -> blocked
```

Later implement:

``` text
row locks
page locks
table locks
lock escalation
deadlock detection
deadlock timeout
```

------------------------------------------------------------------------

# 31. Deadlock Detection

Example:

``` text
TX1 holds Row A
TX2 holds Row B

TX1 waits for Row B
TX2 waits for Row A
```

Graph:

``` text
TX1 -> TX2
TX2 -> TX1
```

Cycle means deadlock.

Create a:

``` text
Wait-for Graph
```

Detect cycles periodically.

Abort one transaction.

------------------------------------------------------------------------

# 32. Checkpoints

Without checkpoints, recovery may have to replay a huge WAL.

Create:

``` text
CHECKPOINT
```

Procedure:

``` text
1. Flush dirty pages
2. Flush WAL
3. Write checkpoint record
4. Persist checkpoint metadata
```

Recovery starts near the latest checkpoint.

------------------------------------------------------------------------

# 33. Crash Recovery

On startup:

``` text
TinyDB
 |
RecoveryManager
 |
Read checkpoint
 |
Read WAL
 |
Redo committed operations
 |
Undo incomplete operations
 |
Rebuild temporary state
 |
Open database
```

Test it by deliberately killing the process:

``` text
insert
update
kill -9
restart
```

Verify the database remains consistent.

------------------------------------------------------------------------

# 34. Durability Tests

Create tests for:

``` text
commit + crash
rollback + crash
partial WAL write
partial page write
multiple transactions
crash during checkpoint
crash during index update
```

A database is not durable because it has a WAL class.

Durability requires actual crash testing.

------------------------------------------------------------------------

# 35. SQL Transactions

Support:

``` sql
BEGIN;

INSERT INTO employee
(id, name)
VALUES
(10, 'John');

UPDATE employee
SET salary = 50000
WHERE id = 10;

COMMIT;
```

Rollback:

``` sql
BEGIN;

UPDATE employee
SET salary = 1
WHERE id = 10;

ROLLBACK;
```

Savepoint:

``` sql
BEGIN;

UPDATE employee
SET salary = 100000
WHERE id = 10;

SAVEPOINT before_bonus;

UPDATE employee
SET salary = 200000
WHERE id = 10;

ROLLBACK TO SAVEPOINT before_bonus;

COMMIT;
```

------------------------------------------------------------------------

# 36. MiniSpring Integration

MiniSpring should not directly access TinyDB's internal storage classes.

Use:

``` text
MiniSpring
   |
TinyDB DataSource
   |
JDBC
   |
TinyDB JDBC Driver
   |
TinyDB Server
```

Example:

``` java
@Configuration
public class DatabaseConfig {

    @Bean
    public DataSource dataSource() {
        return new TinyDataSource(
                "jdbc:tinydb://localhost:9090/company"
        );
    }
}
```

Repository:

``` java
@Repository
public class EmployeeRepository {

    private final JdbcTemplate jdbc;

    public EmployeeRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<Employee> findActiveEmployees() {
        return jdbc.query(
            "SELECT * FROM employee WHERE active = true"
        );
    }
}
```

This is the correct architectural boundary.

MiniSpring should know JDBC.

It should not know:

``` text
Page
WAL
B+Tree
MVCC
BufferPool
```

------------------------------------------------------------------------

# 37. Build a JDBC Driver

Implement the JDBC interfaces needed by your supported feature set.

Start with:

``` text
Driver
Connection
Statement
PreparedStatement
ResultSet
DatabaseMetaData
ResultSetMetaData
SQLException
```

Example:

``` java
public final class TinyDbDriver
        implements java.sql.Driver {

}
```

JDBC URL:

``` text
jdbc:tinydb://localhost:9090/company
```

Connection:

``` java
Connection connection =
        DriverManager.getConnection(
            "jdbc:tinydb://localhost:9090/company"
        );
```

------------------------------------------------------------------------

# 38. Prepared Statements

Support:

``` java
PreparedStatement ps =
    connection.prepareStatement(
        "SELECT * FROM employee WHERE id = ?"
    );

ps.setLong(1, 100);

ResultSet rs = ps.executeQuery();
```

Do not implement prepared statements by simple string replacement.

Represent parameters in the AST or execution plan.

This prevents SQL injection in clients and enables plan reuse later.

------------------------------------------------------------------------

# 39. TinyDB Wire Protocol

Create a small binary or framed protocol.

Example:

``` text
+------------+----------+----------+----------------+
| length     | version  | type     | payload        |
+------------+----------+----------+----------------+
| 4 bytes    | 1 byte   | 1 byte   | N bytes        |
+------------+----------+----------+----------------+
```

Request:

``` text
EXECUTE_SQL
```

Response:

``` text
RESULT_SET
```

Error:

``` text
ERROR
```

Transaction commands:

``` text
BEGIN
COMMIT
ROLLBACK
```

------------------------------------------------------------------------

# 40. Why Not HTTP?

You can expose an HTTP API:

``` text
POST /query
```

but HTTP should not be the core database protocol.

For a database client/server architecture use:

``` text
TCP + framed protocol
```

HTTP can be a separate administration/API layer.

------------------------------------------------------------------------

# 41. TinyDB Terminal

Build:

``` text
tinydb-cli
```

Run:

``` bash
tinydb --host localhost --port 9090
```

Display:

``` text
tinydb>
```

Then:

``` sql
tinydb> CREATE DATABASE company;
tinydb> USE company;

tinydb> CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100)
);

tinydb> INSERT INTO employee
VALUES (1, 'Arpan');

tinydb> SELECT * FROM employee;
```

Output:

``` text
+----+-------+
| id | name  |
+----+-------+
| 1  | Arpan |
+----+-------+

1 row
```

Add commands:

``` text
.help
.tables
.databases
.schema employee
.describe employee
.indexes
.transactions
.version
.exit
```

------------------------------------------------------------------------

# 42. Interactive SQL Shell

The CLI should support multiline SQL:

``` text
tinydb> SELECT *
     -> FROM employee
     -> WHERE id = 10;
```

Detect:

``` text
;
```

as the statement terminator.

Add:

``` text
history
arrow-key navigation
tab completion
command history
formatted result sets
```

------------------------------------------------------------------------

# 43. DBeaver Integration --- JDBC

Create:

``` text
tinydb-jdbc.jar
```

DBeaver needs:

``` text
Driver class
JDBC URL
username
password
```

For example:

``` text
Driver Class:
com.tinydb.jdbc.TinyDbDriver

URL:
jdbc:tinydb://localhost:9090/company
```

DBeaver's generic JDBC connection can then communicate with TinyDB.

For richer DBeaver support, implement:

``` java
DatabaseMetaData
```

so that DBeaver can discover:

``` text
databases
schemas
tables
columns
indexes
primary keys
foreign keys
```

------------------------------------------------------------------------

# 44. Native DBeaver Protocol Support

If you want TinyDB to appear as a native PostgreSQL/MySQL-style
database:

``` text
DBeaver
 |
PostgreSQL/MySQL driver
 |
TinyDB PostgreSQL/MySQL-compatible protocol
 |
TinyDB server
```

This is a separate project phase.

Do this only after:

``` text
SQL engine
storage
transactions
JDBC
CLI
```

are stable.

------------------------------------------------------------------------

# 45. Database Authentication

Implement:

``` text
CREATE USER
DROP USER
ALTER USER
GRANT
REVOKE
```

Example:

``` sql
CREATE USER app_user
WITH PASSWORD 'secret';

GRANT SELECT, INSERT, UPDATE
ON employee
TO app_user;
```

Never store plaintext passwords.

Use:

``` text
salted password hash
```

with a suitable password hashing algorithm.

------------------------------------------------------------------------

# 46. TLS

Eventually support:

``` text
TinyDB client
   |
 TLS
   |
TinyDB server
```

Configuration:

``` properties
tinydb.tls.enabled=true
tinydb.tls.keystore=...
```

This is necessary before treating TinyDB as a remotely accessible
production database.

------------------------------------------------------------------------

# 47. TTL

TTL is useful for:

``` text
sessions
temporary records
caches
event retention
OTP records
temporary application data
```

SQL example:

``` sql
CREATE TABLE session (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    expires_at TIMESTAMP
);
```

Then:

``` sql
SELECT *
FROM session
WHERE expires_at > CURRENT_TIMESTAMP;
```

For automatic expiration, create:

``` text
TTLManager
```

Store expiration metadata in the row:

``` text
expiresAt
```

Do not rely solely on a background thread deleting rows.

Queries should also treat expired rows as invisible.

------------------------------------------------------------------------

# 48. TTL Index

A better implementation uses a time-ordered structure:

``` text
expiration timestamp -> RowId
```

For example:

``` text
12:00 -> row A
12:01 -> row B
12:04 -> row C
```

The TTL worker scans only expired entries.

Do not scan the entire database every second.

------------------------------------------------------------------------

# 49. TTL Semantics

Define clearly:

``` text
TTL starts at INSERT
TTL starts at UPDATE
TTL is absolute timestamp
TTL is duration
expired rows are immediately invisible
expired rows are physically deleted asynchronously
```

Recommended:

``` text
expires_at = absolute timestamp
```

Visibility:

``` text
if now >= expires_at
    row is logically deleted
```

Physical cleanup happens asynchronously.

------------------------------------------------------------------------

# 50. Change Data Capture / Streams

Now implement DynamoDB-style change streams.

Every committed mutation generates an event:

``` text
INSERT
UPDATE
DELETE
```

Example:

``` json
{
  "eventId": "evt-100",
  "transactionId": "tx-20",
  "table": "employee",
  "operation": "UPDATE",
  "key": {
    "id": 1
  },
  "oldImage": {
    "salary": 100000
  },
  "newImage": {
    "salary": 120000
  },
  "timestamp": "..."
}
```

------------------------------------------------------------------------

# 51. Stream Architecture

Use:

``` text
SQL
 |
Transaction
 |
WAL
 |
Commit
 |
ChangeEvent
 |
StreamLog
 |
+--------+--------+--------+
|        |        |        |
Consumer Consumer Consumer
```

Important:

**Publish database change events only after the transaction reaches the
required commit point.**

Otherwise consumers may receive events for changes that later roll back.

------------------------------------------------------------------------

# 52. Stream Log

Implement an append-only log:

``` text
stream-000001.log
stream-000002.log
...
```

Each event gets:

``` text
sequenceNumber
```

Consumers maintain:

``` text
consumer -> last processed sequence
```

Example:

``` text
consumerA -> 100
consumerB -> 95
consumerC -> 70
```

------------------------------------------------------------------------

# 53. Stream APIs

SQL:

``` sql
CREATE STREAM employee_changes
ON employee;
```

Client API:

``` java
ChangeStream stream =
        tinyDb.stream("employee_changes");

stream.subscribe(event -> {
    System.out.println(event);
});
```

Also support polling:

``` java
List<ChangeEvent> events =
        stream.readFrom(sequenceNumber);
```

Polling is important because a consumer can recover after a restart.

------------------------------------------------------------------------

# 54. Stream Retention

Configure:

``` properties
tinydb.stream.retention=24h
```

or:

``` properties
tinydb.stream.retention.bytes=10GB
```

Deletion policy:

``` text
keep events until:
    time retention OR
    size retention
```

Do not delete an event that a consumer still needs unless the documented
retention policy permits it.

------------------------------------------------------------------------

# 55. Replication

Replication should not be added before WAL.

The natural architecture is:

``` text
             +----------------+
             | TinyDB Primary |
             +-------+--------+
                     |
                    WAL
                     |
          +----------+----------+
          |                     |
      Replica 1             Replica 2
```

The primary produces WAL.

Replicas consume WAL records.

------------------------------------------------------------------------

# 56. Replication Types

Implement in stages.

## Stage 1

Asynchronous primary -\> replica.

``` text
Primary
  |
  | WAL
  v
Replica
```

## Stage 2

Replica acknowledgements.

``` text
Primary
 |
 +---- Replica A
 |
 +---- Replica B
```

## Stage 3

Synchronous replication.

Commit may require:

``` text
local WAL flush
+
replica acknowledgement
```

## Stage 4

Leader election / failover.

This is significantly harder.

------------------------------------------------------------------------

# 57. Replication Protocol

Replica requests:

``` text
GET_WAL_FROM LSN 1000
```

Primary responds:

``` text
WAL 1000
WAL 1001
WAL 1002
...
```

Replica:

``` text
append WAL
flush WAL
apply WAL
ack LSN
```

Track:

``` text
primary LSN
replica LSN
replication lag
```

------------------------------------------------------------------------

# 58. Replication Metadata

Expose:

``` sql
SHOW REPLICATION STATUS;
```

Output:

``` text
ROLE       PRIMARY
LSN        100000
REPLICA 1  99980
REPLICA 2  100000
```

Metrics:

``` text
replication_lag
wal_bytes
wal_lsn
checkpoint_lsn
```

------------------------------------------------------------------------

# 59. Read Replicas

Eventually allow:

``` text
Application
   |
   +---- Write -> Primary
   |
   +---- Read  -> Replica
```

But document consistency:

``` text
read-after-write
eventual consistency
stale reads
```

A replica cannot automatically provide the same consistency guarantees
as the primary.

------------------------------------------------------------------------

# 60. Backup and Restore

A real database needs backup.

Implement:

``` text
full backup
incremental backup
WAL archive
restore
point-in-time recovery
```

Commands:

``` bash
tinydb backup company ./backup
tinydb restore ./backup company
```

Point-in-time recovery:

``` text
restore backup
+
replay WAL
until timestamp
```

------------------------------------------------------------------------

# 61. Schema Versioning

Persist:

``` text
formatVersion
catalogVersion
```

Example:

``` text
TinyDB storage version: 3
```

When storage format changes:

``` text
migration
```

must be possible.

Never assume Java class serialization is your long-term storage format.

------------------------------------------------------------------------

# 62. Checksum and Corruption Detection

Pages should have checksums.

On read:

``` text
read page
 |
calculate checksum
 |
compare stored checksum
 |
valid?
```

If invalid:

``` text
report corruption
```

Later integrate:

``` text
backup recovery
replica recovery
```

------------------------------------------------------------------------

# 63. Threading Model

TinyDB will need multiple thread pools.

Separate:

``` text
Network threads
Query execution threads
WAL writer
Checkpoint thread
TTL worker
Replication worker
Stream worker
Background maintenance
```

Do not create one thread per database operation.

Use bounded executors.

------------------------------------------------------------------------

# 64. Backpressure

Streams and replication can overwhelm consumers.

Implement:

``` text
bounded queues
rate limits
batching
timeouts
consumer lag monitoring
```

Avoid:

``` java
new Thread(...).start();
```

for every event.

------------------------------------------------------------------------

# 65. Connection Pool

The JDBC client should support connection pooling.

MiniSpring can later provide:

``` text
TinyDataSource
```

with:

``` text
maxPoolSize
minIdle
connectionTimeout
idleTimeout
maxLifetime
```

Do not confuse a connection pool with the database's internal worker
pool.

------------------------------------------------------------------------

# 66. SQL Result Streaming

Do not load millions of rows into memory.

Bad:

``` java
List<Row> allRows = readEverything();
```

Better:

``` text
ResultSet
   |
iterator
   |
network batches
```

Support:

``` text
fetchSize
```

Example:

``` java
statement.setFetchSize(1000);
```

------------------------------------------------------------------------

# 67. Pagination

Support:

``` sql
SELECT *
FROM employee
ORDER BY id
LIMIT 100
OFFSET 1000;
```

But large OFFSET can be expensive.

Later support keyset pagination:

``` sql
SELECT *
FROM employee
WHERE id > 1000
ORDER BY id
LIMIT 100;
```

------------------------------------------------------------------------

# 68. Statistics

Query planner needs statistics.

Maintain:

``` text
row count
distinct values
min
max
null count
histogram
```

Example:

``` text
employee.salary

rows = 1,000,000
min = 20,000
max = 1,000,000
distinct = 50,000
```

This enables cost-based optimization.

------------------------------------------------------------------------

# 69. EXPLAIN

Support:

``` sql
EXPLAIN
SELECT *
FROM employee
WHERE id = 100;
```

Output:

``` text
IndexScan
  index = employee_pk
  estimatedRows = 1
```

Later:

``` sql
EXPLAIN ANALYZE
```

to compare estimated versus actual execution.

------------------------------------------------------------------------

# 70. Join Engine

Start with:

``` text
Nested Loop Join
```

Then:

``` text
Hash Join
```

Then:

``` text
Merge Join
```

Example:

``` sql
SELECT e.name, d.name
FROM employee e
JOIN department d
ON e.department_id = d.id;
```

Execution:

``` text
Join
 /  \
Scan Scan
```

------------------------------------------------------------------------

# 71. Aggregation

Implement:

``` sql
SELECT department_id, COUNT(*)
FROM employee
GROUP BY department_id;
```

Operators:

``` text
Aggregate
HashAggregate
SortAggregate
```

Later optimize with indexes and statistics.

------------------------------------------------------------------------

# 72. Sorting

Implement external sorting when data does not fit memory.

Instead of:

``` text
load all rows
sort in RAM
```

use:

``` text
read chunk
sort
write temporary file

read next chunk
sort
write temporary file

merge sorted runs
```

This is essential for large datasets.

------------------------------------------------------------------------

# 73. Memory Manager

Define limits:

``` properties
tinydb.memory.bufferPool=1GB
tinydb.memory.query=256MB
tinydb.memory.sort=128MB
```

Queries must not consume unlimited memory.

Introduce:

``` text
MemoryManager
Quota
QueryMemoryContext
```

------------------------------------------------------------------------

# 74. Configuration

Use a central configuration system.

Example:

``` properties
tinydb.server.port=9090

tinydb.storage.path=./data

tinydb.storage.page-size=8192

tinydb.buffer-pool.pages=10000

tinydb.wal.path=./wal

tinydb.wal.sync=true

tinydb.transaction.isolation=READ_COMMITTED

tinydb.ttl.enabled=true

tinydb.stream.enabled=true

tinydb.replication.enabled=false
```

Avoid hard-coded configuration.

------------------------------------------------------------------------

# 75. Observability

Expose:

``` text
metrics
logs
health
diagnostics
```

Metrics:

``` text
queries_total
query_latency
transactions_total
transactions_committed
transactions_rolled_back
wal_bytes
buffer_hits
buffer_misses
cache_hit_ratio
deadlocks
active_connections
stream_lag
replication_lag
ttl_deleted_rows
```

------------------------------------------------------------------------

# 76. Admin Commands

CLI:

``` text
.status
.metrics
.tables
.schema employee
.indexes
.connections
.transactions
.replication
.streams
.checkpoint
.backup
```

Eventually SQL equivalents:

``` sql
SHOW DATABASES;
SHOW TABLES;
SHOW INDEXES;
SHOW TRANSACTIONS;
SHOW REPLICATION STATUS;
SHOW STREAMS;
```

------------------------------------------------------------------------

# 77. Error Handling

Create typed errors.

Examples:

``` text
SyntaxError
TableNotFoundException
ColumnNotFoundException
DuplicateKeyException
ConstraintViolationException
TransactionException
DeadlockException
SerializationException
StorageCorruptionException
```

Convert them into:

``` text
SQLSTATE-like codes
```

This will make JDBC integration much easier.

------------------------------------------------------------------------

# 78. MiniSpring Transaction Integration

Eventually MiniSpring can expose:

``` java
@Transactional
public void transfer() {

    accountRepository.debit(...);

    accountRepository.credit(...);
}
```

MiniSpring transaction interceptor:

``` text
@Transactional
      |
TransactionInterceptor
      |
BEGIN
      |
method execution
      |
COMMIT
```

On exception:

``` text
ROLLBACK
```

This is an excellent project for understanding Spring transaction
management.

------------------------------------------------------------------------

# 79. TinyDB Transaction API

Expose:

``` java
public interface TinyTransaction {

    void begin();

    void commit();

    void rollback();

    Savepoint savepoint(String name);

    void rollbackTo(Savepoint savepoint);
}
```

JDBC maps these to:

``` java
Connection.setAutoCommit(false);
Connection.commit();
Connection.rollback();
Connection.setSavepoint();
```

------------------------------------------------------------------------

# 80. Connection Lifecycle

Implement:

``` text
connect
 |
authenticate
 |
create session
 |
execute
 |
transaction
 |
commit/rollback
 |
close
```

Each session contains:

``` text
sessionId
user
database
transaction
settings
lastActivity
```

------------------------------------------------------------------------

# 81. Session Variables

Support:

``` sql
SET isolation_level = 'READ_COMMITTED';
SET timezone = 'UTC';
```

Session state should not leak between connections.

------------------------------------------------------------------------

# 82. SQL Parser Improvement

Do not create a giant parser class.

Use:

``` text
Lexer
Parser
ExpressionParser
StatementParser
DDLParser
DMLParser
```

For complex SQL, use precedence-aware parsing such as Pratt parsing or
another structured expression parser.

------------------------------------------------------------------------

# 83. Security Boundaries

Separate:

``` text
Authentication
Authorization
SQL execution
Storage
```

Do not let SQL statements bypass authorization.

Flow:

``` text
SQL
 |
Parser
 |
Authorization
 |
Planner
 |
Executor
```

Check:

``` text
user
database
table
operation
column
```

------------------------------------------------------------------------

# 84. SQL Injection Protection

The database parser should parse values as values.

Applications should use:

``` text
PreparedStatement
```

not:

``` java
"SELECT * FROM user WHERE id = " + id
```

TinyDB itself should still correctly tokenize and parse quoted strings.

------------------------------------------------------------------------

# 85. Testing Strategy

Use several levels.

## Unit tests

``` text
LexerTest
ParserTest
BTreeTest
PageTest
WalTest
LockManagerTest
MvccTest
```

## Integration tests

``` text
CREATE TABLE
INSERT
SELECT
UPDATE
DELETE
transactions
rollback
```

## Crash tests

``` text
kill process
restart
verify state
```

## Concurrency tests

``` text
100 threads
1000 transactions
```

## JDBC tests

``` text
Driver
Connection
PreparedStatement
ResultSet
Transaction
```

## Compatibility tests

Run the same SQL test suite against:

``` text
TinyDB
reference database
```

where the syntax overlaps.

------------------------------------------------------------------------

# 86. Property-Based Testing

Database engines have many edge cases.

Useful properties:

``` text
insert then select -> row exists

insert then rollback -> row absent

update then rollback -> old value

commit then restart -> committed value exists

delete then rollback -> row exists
```

Generate random sequences:

``` text
INSERT
UPDATE
DELETE
BEGIN
ROLLBACK
COMMIT
```

and compare TinyDB against an oracle for supported behavior.

------------------------------------------------------------------------

# 87. Benchmarking

Use JMH.

Benchmark:

``` text
insert throughput
point lookup
range lookup
update
delete
transaction commit
rollback
WAL throughput
B+Tree lookup
buffer hit
buffer miss
```

Example targets should be measured, not assumed.

------------------------------------------------------------------------

# 88. Concurrency Benchmark

Run:

``` text
1 thread
2
4
8
16
32
64
```

Measure:

``` text
throughput
p50
p95
p99
CPU
memory
lock contention
```

------------------------------------------------------------------------

# 89. Build the CLI First

Milestone:

``` text
tinydb
```

must be usable without MiniSpring.

Example:

``` bash
./tinydb-server
```

Then:

``` bash
./tinydb
```

and:

``` sql
CREATE DATABASE test;
USE test;

CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO users VALUES (1, 'Arpan');

SELECT * FROM users;
```

This milestone proves the database itself works.

------------------------------------------------------------------------

# 90. End-to-End Architecture

Final architecture:

``` text
                         +-------------------+
                         |       DBeaver     |
                         +---------+---------+
                                   |
                                  JDBC
                                   |
                         +---------v---------+
                         | TinyDB JDBC Driver|
                         +---------+---------+
                                   |
                              TCP Protocol
                                   |
+----------------------------------v----------------------------------+
|                           TinyDB Server                              |
|                                                                      |
| Session / Auth                                                       |
|       |                                                              |
| SQL Lexer                                                            |
|       |                                                              |
| SQL Parser                                                           |
|       |                                                              |
| AST                                                                  |
|       |                                                              |
| Authorization                                                        |
|       |                                                              |
| Query Planner                                                        |
|       |                                                              |
| Execution Engine                                                     |
|       |                                                              |
| Transaction Manager <----> Lock Manager / MVCC                        |
|       |                                                              |
| Storage Engine                                                       |
|       |                                                              |
| Buffer Pool                                                          |
|       |                                                              |
| B+Tree Indexes                                                       |
|       |                                                              |
| Page Manager                                                         |
|       |                                                              |
| WAL <-------- Recovery Manager                                       |
|       |                                                              |
| Disk                                                                 |
|                                                                      |
| Change Stream -> Consumers                                           |
|                                                                      |
| Replication -> Replica Nodes                                         |
|                                                                      |
| TTL Manager                                                          |
+----------------------------------------------------------------------+
```

------------------------------------------------------------------------

# 91. Recommended Implementation Order

Follow this order.

## Milestone 1 --- In-memory database

``` text
Database
Table
Schema
Row
Insert
Select
Update
Delete
```

## Milestone 2 --- SQL

``` text
Lexer
Parser
AST
Executor
WHERE
ORDER BY
LIMIT
```

## Milestone 3 --- Persistent storage

``` text
Page
Record
File
BufferPool
```

## Milestone 4 --- Index

``` text
B+Tree
Primary key
Secondary index
```

## Milestone 5 --- Query planner

``` text
TableScan
IndexScan
Filter
Project
Sort
Limit
```

## Milestone 6 --- Transactions

``` text
BEGIN
COMMIT
ROLLBACK
```

## Milestone 7 --- WAL and recovery

``` text
WAL
LSN
Checkpoint
Redo
Undo
Crash recovery
```

## Milestone 8 --- Concurrency

``` text
LockManager
Deadlock detection
MVCC
Isolation levels
```

## Milestone 9 --- Server

``` text
TCP
Sessions
Authentication
Protocol
```

## Milestone 10 --- CLI

``` text
tinydb terminal
```

## Milestone 11 --- JDBC

``` text
Driver
Connection
Statement
PreparedStatement
ResultSet
```

## Milestone 12 --- DBeaver

``` text
JDBC metadata
Generic JDBC connection
```

## Milestone 13 --- MiniSpring

``` text
DataSource
JdbcTemplate
@Transactional
Repository
```

## Milestone 14 --- Advanced SQL

``` text
JOIN
GROUP BY
HAVING
aggregates
subqueries
EXPLAIN
```

## Milestone 15 --- TTL

``` text
expires_at
TTL index
background cleanup
```

## Milestone 16 --- Streams

``` text
ChangeEvent
StreamLog
Consumer
Offsets
Retention
```

## Milestone 17 --- Replication

``` text
WAL shipping
Replica
Acknowledgement
Read replica
```

## Milestone 18 --- Native protocol

``` text
PostgreSQL/MySQL-compatible protocol
```

## Milestone 19 --- Production hardening

``` text
TLS
backup
restore
monitoring
checksums
rate limiting
resource limits
```

------------------------------------------------------------------------

# 92. SOLID Design

## Single Responsibility

Bad:

``` java
DatabaseManager
```

doing:

``` text
SQL parsing
storage
transactions
networking
logging
```

Good:

``` text
SqlParser
QueryPlanner
QueryExecutor
StorageEngine
TransactionManager
WalManager
BufferPool
```

Each has a clear responsibility.

------------------------------------------------------------------------

# 93. Open/Closed Principle

New index types should not require modifying query execution everywhere.

Use:

``` java
public interface Index {

    void insert(Key key, RowId rowId);

    void delete(Key key, RowId rowId);

    List<RowId> search(IndexPredicate predicate);
}
```

Implement:

``` text
BTreeIndex
HashIndex
BitmapIndex
```

------------------------------------------------------------------------

# 94. Liskov Substitution

Executors should be interchangeable where their contracts permit it.

For example:

``` text
StorageEngine
   |
   +-- MemoryStorageEngine
   +-- FileStorageEngine
```

Tests can use:

``` text
MemoryStorageEngine
```

without changing the query layer.

------------------------------------------------------------------------

# 95. Interface Segregation

Avoid:

``` java
interface DatabaseEverything {
    parse();
    execute();
    writeDisk();
    replicate();
    authenticate();
    stream();
}
```

Instead:

``` text
StorageEngine
QueryEngine
TransactionManager
ReplicationManager
StreamManager
AuthenticationService
```

------------------------------------------------------------------------

# 96. Dependency Inversion

High-level components should depend on interfaces.

Example:

``` java
public final class InsertExecutor {

    private final StorageEngine storageEngine;

    public InsertExecutor(StorageEngine storageEngine) {
        this.storageEngine = storageEngine;
    }
}
```

Not:

``` java
new FileStorageEngine()
```

inside the executor.

This makes testing and future storage implementations easier.

------------------------------------------------------------------------

# 97. Design Patterns

Use patterns where they solve real problems.

## Strategy

Use for:

``` text
EvictionPolicy
IsolationStrategy
QueryOptimizationStrategy
IndexStrategy
ReplicationStrategy
```

## Factory

Use for:

``` text
ExecutorFactory
DataTypeFactory
StorageEngineFactory
```

## Builder

Use for:

``` text
QueryPlan
TableSchema
DatabaseConfig
```

## Visitor

Useful for:

``` text
AST traversal
SQL expression analysis
query optimization
```

## Command

Useful for:

``` text
SQL statements
CLI commands
transaction operations
```

## Observer / Publisher-Subscriber

Use for:

``` text
change streams
metrics
events
```

## Chain of Responsibility

Useful for:

``` text
authentication
authorization
query processing pipeline
```

## Template Method

Useful for:

``` text
common executor lifecycle
```

## Adapter

Very useful for:

``` text
JDBC
DBeaver integration
storage compatibility
```

## Facade

Expose:

``` java
TinyDatabase
```

while hiding:

``` text
WAL
buffer pool
catalog
transaction internals
```

## Proxy

Useful for:

``` text
transaction boundaries
metrics
authorization
logging
MiniSpring @Transactional
```

## State

Useful for:

``` text
transaction state
connection state
replication state
```

------------------------------------------------------------------------

# 98. Avoid Overusing Design Patterns

Do not create a factory for every class.

Good architecture is:

``` text
clear responsibilities
small interfaces
low coupling
high cohesion
testability
```

not:

``` text
maximum number of design patterns
```

Use a pattern only when it makes the design easier to evolve.

------------------------------------------------------------------------

# 99. Recommended Core Interfaces

A mature TinyDB could contain interfaces like:

``` java
interface Catalog {}

interface SqlParser {}

interface QueryPlanner {}

interface QueryExecutor {}

interface StorageEngine {}

interface PageManager {}

interface BufferPool {}

interface Index {}

interface IndexManager {}

interface TransactionManager {}

interface LockManager {}

interface WalManager {}

interface RecoveryManager {}

interface CheckpointManager {}

interface ReplicationManager {}

interface ChangeStreamManager {}

interface TtlManager {}

interface AuthenticationService {}

interface AuthorizationService {}

interface ProtocolServer {}
```

Keep implementations replaceable.

------------------------------------------------------------------------

# 100. Plugin Architecture for Future Features

Eventually support:

``` text
StoragePlugin
IndexPlugin
ReplicationPlugin
StreamPlugin
AuthenticationPlugin
OptimizerPlugin
```

Configuration:

``` properties
tinydb.storage=page-file
tinydb.index=btree
tinydb.replication=async
tinydb.authentication=password
tinydb.stream=wal-cdc
```

This makes experimentation easier.

------------------------------------------------------------------------

# 101. Important Improvement --- Separate Logical and Physical Layers

A critical architectural rule:

``` text
SQL
 |
Logical Plan
 |
Physical Plan
 |
Execution
 |
Storage
```

Do not allow SQL classes to directly manipulate pages.

For example:

``` text
SELECT
```

should never call:

``` java
pageManager.readPage(...)
```

directly.

It should go through:

``` text
Planner -> Executor -> Storage
```

This separation is one of the most important design decisions in the
project.

------------------------------------------------------------------------

# 102. Important Improvement --- WAL as the Source for Replication and CDC

Avoid implementing separate mutation pipelines:

``` text
SQL -> database
SQL -> replication
SQL -> stream
```

Prefer:

``` text
SQL
 |
Transaction
 |
WAL
 |
Commit
 |
+------------+-------------+
|            |             |
Recovery  Replication   CDC Stream
```

This dramatically reduces duplicated logic.

------------------------------------------------------------------------

# 103. Important Improvement --- One Transactional Commit Point

All of these should be coordinated:

``` text
data
index
WAL
CDC
replication
```

The transaction manager should define the commit boundary.

Example:

``` text
BEGIN
 |
mutations
 |
write WAL
 |
flush WAL
 |
commit record
 |
publish committed change events
 |
COMMIT
```

The exact ordering must be designed carefully once your durability and
replication guarantees are defined.

------------------------------------------------------------------------

# 104. Important Improvement --- Logical vs Physical Replication

Physical replication:

``` text
ship WAL/page changes
```

Advantages:

``` text
fast
simple for replicas
preserves storage-level state
```

Logical replication:

``` text
INSERT
UPDATE
DELETE
```

Advantages:

``` text
cross-version compatibility
selective replication
CDC integration
```

Eventually TinyDB can support both.

------------------------------------------------------------------------

# 105. Important Improvement --- Event Idempotency

Consumers can receive:

``` text
event 100
event 100
```

after a retry.

Therefore events need:

``` text
eventId
transactionId
sequenceNumber
```

Consumers should process idempotently.

------------------------------------------------------------------------

# 106. Important Improvement --- Exactly Once

Do not casually claim:

``` text
exactly once
```

across arbitrary distributed systems.

Implement:

``` text
at-least-once delivery
+
idempotent consumers
```

first.

Then define exactly what TinyDB guarantees.

------------------------------------------------------------------------

# 107. Important Improvement --- Backpressure Everywhere

Apply backpressure to:

``` text
network
query execution
WAL
replication
CDC
TTL
backup
```

Example:

``` text
Primary WAL
    |
bounded replication queue
    |
Replica
```

If the queue is full:

``` text
block
slow down
or fail according to policy
```

Do not allow unlimited memory growth.

------------------------------------------------------------------------

# 108. Important Improvement --- Configuration Profiles

Support:

``` text
development
test
production
```

Example:

``` text
tinydb-dev.properties
tinydb-test.properties
tinydb-prod.properties
```

Production should enforce safer defaults:

``` text
WAL sync enabled
TLS enabled
authentication enabled
bounded memory
backup enabled
```

------------------------------------------------------------------------

# 109. Important Improvement --- Database Observability

Eventually expose an admin endpoint:

``` text
/admin/health
/admin/metrics
/admin/status
/admin/replication
/admin/wal
```

But keep it separate from the SQL protocol.

------------------------------------------------------------------------

# 110. Important Improvement --- Graceful Shutdown

Shutdown sequence:

``` text
stop accepting connections
 |
stop new transactions
 |
wait for active transactions
 |
commit/rollback according to policy
 |
flush WAL
 |
flush dirty pages
 |
checkpoint
 |
close files
 |
stop workers
```

Never simply call:

``` java
System.exit(0);
```

and assume the database is safe.

------------------------------------------------------------------------

# 111. Important Improvement --- Database State Machine

Represent lifecycle:

``` text
CREATED
STARTING
RECOVERING
RUNNING
CHECKPOINTING
STOPPING
STOPPED
FAILED
```

This avoids invalid operations during startup/recovery.

------------------------------------------------------------------------

# 112. Important Improvement --- Versioned Protocol

Wire protocol:

``` text
version
requestType
requestId
payload
```

Future versions can coexist.

Example:

``` text
protocol v1
protocol v2
```

Do not make protocol compatibility dependent on Java class
serialization.

------------------------------------------------------------------------

# 113. Suggested Final Package Structure

``` text
com.tinydb
│
├── common
│   ├── error
│   ├── config
│   ├── types
│   └── util
│
├── catalog
│   ├── Catalog
│   ├── DatabaseMetadata
│   ├── TableMetadata
│   └── SchemaManager
│
├── sql
│   ├── lexer
│   ├── parser
│   ├── ast
│   └── formatter
│
├── planner
│   ├── logical
│   ├── physical
│   ├── optimizer
│   └── statistics
│
├── execution
│   ├── executor
│   └── operator
│
├── storage
│   ├── page
│   ├── record
│   ├── file
│   ├── buffer
│   └── recovery
│
├── index
│   ├── btree
│   └── hash
│
├── transaction
│   ├── tx
│   ├── lock
│   ├── mvcc
│   └── isolation
│
├── wal
│
├── stream
│
├── replication
│
├── ttl
│
├── server
│
├── protocol
│
├── jdbc
│
└── cli
```

------------------------------------------------------------------------

# 114. Definition of Done --- Basic Database

TinyDB v1 should be considered complete only when this works:

``` sql
CREATE DATABASE company;

USE company;

CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(200) UNIQUE,
    salary DECIMAL(12,2)
);

INSERT INTO employee
VALUES
(1, 'Arpan', 'arpan@example.com', 100000);

INSERT INTO employee
VALUES
(2, 'John', 'john@example.com', 80000);

SELECT *
FROM employee;

SELECT name, salary
FROM employee
WHERE salary > 90000;

UPDATE employee
SET salary = 110000
WHERE id = 1;

DELETE FROM employee
WHERE id = 2;
```

Then:

``` text
restart TinyDB
```

and verify data still exists.

------------------------------------------------------------------------

# 115. Definition of Done --- Transactions

This must work:

``` sql
BEGIN;

INSERT INTO employee
VALUES (3, 'Test', 'test@example.com', 50000);

ROLLBACK;
```

Then:

``` sql
SELECT *
FROM employee
WHERE id = 3;
```

Expected:

``` text
0 rows
```

Commit test:

``` sql
BEGIN;

INSERT INTO employee
VALUES (3, 'Test', 'test@example.com', 50000);

COMMIT;
```

Restart server.

Then:

``` sql
SELECT *
FROM employee
WHERE id = 3;
```

The committed row must still exist.

------------------------------------------------------------------------

# 116. Definition of Done --- DBeaver

DBeaver should be able to:

``` text
connect
authenticate
list databases
list tables
inspect columns
execute SELECT
execute INSERT
execute UPDATE
execute DELETE
commit
rollback
```

Initially through:

``` text
JDBC
```

Later through:

``` text
native PostgreSQL/MySQL-compatible protocol
```

------------------------------------------------------------------------

# 117. Definition of Done --- MiniSpring

MiniSpring application:

``` java
@Repository
class EmployeeRepository {

    private final JdbcTemplate jdbcTemplate;

    EmployeeRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}
```

should be able to execute:

``` java
jdbcTemplate.update(
    "INSERT INTO employee VALUES (?, ?, ?, ?)",
    10,
    "Arpan",
    "arpan@example.com",
    100000
);
```

And:

``` java
@Transactional
public void updateEmployee() {
    repository.updateSalary(...);
    repository.updateSomethingElse(...);
}
```

should commit or rollback atomically.

------------------------------------------------------------------------

# 118. Definition of Done --- Streams

After:

``` sql
INSERT INTO employee
VALUES (100, 'Stream Test', 'stream@example.com', 50000);
```

a consumer should receive:

``` text
INSERT employee id=100
```

After:

``` sql
UPDATE employee
SET salary = 60000
WHERE id = 100;
```

receive:

``` text
UPDATE employee id=100
old salary=50000
new salary=60000
```

After:

``` sql
DELETE FROM employee
WHERE id = 100;
```

receive:

``` text
DELETE employee id=100
```

------------------------------------------------------------------------

# 119. Definition of Done --- Replication

Start:

``` text
TinyDB Primary
TinyDB Replica
```

Insert:

``` sql
INSERT INTO employee ...
```

Verify:

``` text
Primary -> WAL -> Replica
```

and eventually:

``` sql
SELECT ...
```

on the replica returns the replicated data.

Also expose:

``` text
replication lag
last applied LSN
last received LSN
```

------------------------------------------------------------------------

# 120. What Not to Implement Too Early

Avoid starting with:

``` text
distributed consensus
sharding
full SQL compatibility
parallel query execution
query optimizer
columnar storage
distributed transactions
```

until these are working:

``` text
single-node storage
SQL
WAL
recovery
transactions
indexes
JDBC
CLI
```

Otherwise debugging becomes extremely difficult.

------------------------------------------------------------------------

# 121. Suggested Long-Term Architecture

The mature TinyDB architecture should look like:

``` text
                        Clients
                           |
          +----------------+----------------+
          |                |                |
        CLI             JDBC            Native Protocol
          |                |                |
          +----------------+----------------+
                           |
                       Session
                           |
                  Auth / Authorization
                           |
                       SQL Parser
                           |
                    Logical Planner
                           |
                    Query Optimizer
                           |
                    Physical Planner
                           |
                    Execution Engine
                           |
                  Transaction Manager
                    /            \
                 MVCC          Locks
                    \            /
                     Storage Engine
                           |
                    +------+------+
                    |             |
                  Buffer         WAL
                   Pool           |
                    |       +-----+------+
                    |       |            |
                  Pages  Recovery    Replication
                    |
                 Indexes
                    |
                  Disk

WAL
 |
 +---- Recovery
 |
 +---- Replication
 |
 +---- CDC / Streams
```

------------------------------------------------------------------------

# 122. Final Feature Matrix

  Feature                   Target
  ------------------------- ----------------
  CREATE DATABASE           Yes
  CREATE TABLE              Yes
  INSERT                    Yes
  SELECT                    Yes
  SELECT \*                 Yes
  WHERE                     Yes
  AND / OR                  Yes
  UPDATE                    Yes
  DELETE                    Yes
  ORDER BY                  Yes
  LIMIT                     Yes
  OFFSET                    Yes
  Primary Key               Yes
  Unique                    Yes
  NOT NULL                  Yes
  DEFAULT                   Yes
  CHECK                     Yes
  Foreign Key               Later
  JOIN                      Later
  GROUP BY                  Later
  HAVING                    Later
  Aggregation               Later
  Subqueries                Later
  Indexes                   Yes
  B+Tree                    Yes
  Transactions              Yes
  COMMIT                    Yes
  ROLLBACK                  Yes
  SAVEPOINT                 Later
  WAL                       Yes
  Crash Recovery            Yes
  MVCC                      Later/advanced
  Isolation Levels          Yes
  Deadlock Detection        Yes
  TTL                       Yes
  Change Streams            Yes
  Replication               Yes
  Read Replicas             Later
  Backup                    Yes
  Point-in-Time Recovery    Later
  CLI                       Yes
  JDBC                      Yes
  MiniSpring                Yes
  DBeaver via JDBC          Yes
  Native DBeaver protocol   Later
  TLS                       Yes
  Authentication            Yes
  Authorization             Yes
  Metrics                   Yes
  EXPLAIN                   Yes
  EXPLAIN ANALYZE           Later

------------------------------------------------------------------------

# 123. Final Recommended Development Sequence

If this is a learning project, implement the following exact sequence:

``` text
1. Java project structure
2. DataType
3. Column
4. TableSchema
5. Row
6. In-memory Table
7. Database Catalog
8. INSERT
9. SELECT
10. UPDATE
11. DELETE

12. Lexer
13. Parser
14. AST
15. SQL Executor

16. CREATE TABLE
17. DROP TABLE
18. WHERE
19. ORDER BY
20. LIMIT

21. Page
22. Record
23. FileManager
24. BufferPool
25. Persistent Table

26. B+Tree
27. Primary Key
28. Secondary Index

29. Query Planner
30. TableScan
31. IndexScan
32. Filter
33. Projection
34. Sort

35. TransactionManager
36. BEGIN
37. COMMIT
38. ROLLBACK

39. WAL
40. LSN
41. Checkpoint
42. Recovery
43. Crash testing

44. LockManager
45. Deadlock detection
46. Isolation levels
47. MVCC

48. TCP protocol
49. Session
50. Authentication

51. TinyDB CLI

52. JDBC Driver
53. PreparedStatement
54. ResultSet
55. DatabaseMetaData

56. DBeaver through JDBC

57. MiniSpring DataSource
58. MiniSpring JdbcTemplate
59. MiniSpring @Transactional

60. JOIN
61. GROUP BY
62. HAVING
63. Aggregates
64. EXPLAIN

65. TTL
66. TTL index
67. TTL cleanup

68. CDC
69. Stream log
70. Consumer offsets
71. Stream retention

72. WAL replication
73. Replica
74. Replication acknowledgement
75. Read replica

76. Backup
77. Restore
78. PITR

79. TLS
80. Metrics
81. Monitoring
82. Security hardening

83. Native PostgreSQL/MySQL protocol compatibility
84. Distributed replication / failover
85. Sharding
```

------------------------------------------------------------------------

# 124. The Most Important Design Principle

Do not think of TinyDB as:

``` text
SQL parser + file
```

Think of it as several independent engines:

``` text
SQL Engine
+
Query Engine
+
Transaction Engine
+
Storage Engine
+
Index Engine
+
Recovery Engine
+
Replication Engine
+
Stream Engine
+
Server Engine
+
Client/JDBC Engine
```

Each engine should communicate through well-defined interfaces.

That architecture will let you evolve TinyDB without rewriting the
entire database.

------------------------------------------------------------------------

# 125. Suggested Future Enhancements

After the core system works, the most valuable improvements are:

### Storage

``` text
compressed pages
page checksums
free-space maps
compaction
LSM-tree storage
columnar storage
tiered storage
```

### Query Engine

``` text
cost-based optimizer
statistics
histograms
parallel query execution
vectorized execution
prepared-plan cache
query result cache
```

### Transactions

``` text
full MVCC
serializable isolation
predicate locks
SSI
distributed transactions
```

### Reliability

``` text
WAL archiving
point-in-time recovery
automatic repair
replica-based recovery
checksums
corruption detection
```

### Distributed Systems

``` text
leader election
consensus
automatic failover
read replicas
multi-region replication
quorum writes
quorum reads
sharding
partition rebalancing
```

### Streams

``` text
consumer groups
partitioning
parallel consumers
event filtering
event replay
dead-letter stream
stream compaction
exactly-once processing semantics where explicitly achievable
```

### SQL

``` text
CTEs
window functions
subqueries
views
materialized views
triggers
stored procedures
generated columns
JSON data
full-text search
```

### Developer Experience

``` text
CLI autocompletion
migration tool
database explorer
schema diff
backup CLI
admin dashboard
JDBC metadata
ORM integration
Flyway/Liquibase-style migrations
```

------------------------------------------------------------------------

# 126. Final Project Goal

The completed project should allow this:

``` text
                  DBeaver
                     |
                   JDBC
                     |
              +------+------+
              |             |
         MiniSpring      CLI
              |             |
              +------+------+
                     |
                TinyDB Server
                     |
       +-------------+-------------+
       |             |             |
      SQL         Transaction    Session
       |             |             |
     Planner       MVCC/Lock       |
       |             |             |
       +-------------+-------------+
                     |
               Storage Engine
                     |
          +----------+----------+
          |          |          |
        Buffer      Index       WAL
         Pool        |           |
          |          |      +----+----+
        Pages      B+Tree   |         |
          |                 CDC    Replication
          |
         Disk
```

And a user should be able to run:

``` bash
tinydb-server --data ./data
```

then:

``` bash
tinydb --host localhost --port 9090
```

and execute:

``` sql
CREATE DATABASE shop;

USE shop;

CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(12,2),
    stock INT
);

INSERT INTO product
VALUES (1, 'Laptop', 75000.00, 10);

SELECT *
FROM product
WHERE price > 50000;

BEGIN;

UPDATE product
SET stock = stock - 1
WHERE id = 1;

COMMIT;
```

The same database should then be accessible from:

``` text
TinyDB CLI
      |
JDBC
      |
MiniSpring
      |
DBeaver
```

while internally supporting:

``` text
SQL
ACID
Transactions
Rollback
WAL
Crash Recovery
Indexes
TTL
Change Streams
Replication
Backup
Observability
```

That is the practical roadmap for turning TinyDB from a Java learning
project into a small but architecturally serious relational database
engine.

------------------------------------------------------------------------

# Physical File Storage Implementation Guide

This section is the concrete implementation guide for the part of TinyDB
that actually persists data on disk. It should be implemented before
advanced SQL features.

## 1. What TinyDB Stores on Disk

Configure one root directory:

``` text
tinydb-data/
├── databases/
│   └── company/
│       ├── database.meta
│       ├── tables/
│       │   ├── employees.tbl
│       │   └── departments.tbl
│       ├── indexes/
│       │   ├── employees_pk.idx
│       │   └── employees_email.idx
│       └── fsm/
│           └── employees.fsm
├── catalog/
│   └── catalog.db
├── wal/
│   ├── wal-000001.log
│   └── wal-000002.log
├── checkpoints/
│   └── checkpoint.meta
└── server.lock
```

Meaning:

-   `.tbl` = table heap pages
-   `.idx` = persisted index pages
-   `.fsm` = free-space metadata
-   `catalog.db` = database metadata
-   `wal/*.log` = write-ahead log
-   `checkpoint.meta` = recovery starting point
-   `server.lock` = prevents two server processes from opening the same
    database

A row is **not** stored as a Java object. It is encoded into bytes
inside a page.

------------------------------------------------------------------------

## 2. Implement the Storage Module First

Recommended packages:

``` text
tinydb-storage/
└── src/main/java/com/tinydb/storage/
    ├── file/
    │   ├── DatabaseDirectory.java
    │   ├── TableFile.java
    │   └── FileManager.java
    ├── page/
    │   ├── Page.java
    │   ├── PageId.java
    │   ├── PageHeader.java
    │   ├── Slot.java
    │   └── RowId.java
    ├── row/
    │   ├── Row.java
    │   ├── RowCodec.java
    │   └── BinaryRowCodec.java
    ├── buffer/
    │   ├── BufferPool.java
    │   └── BufferFrame.java
    ├── table/
    │   ├── TableHeap.java
    │   └── HeapFile.java
    ├── fsm/
    │   └── FreeSpaceMap.java
    └── wal/
        ├── WalManager.java
        ├── WalRecord.java
        └── WalWriter.java
```

The dependency direction should be:

``` text
SQL / Execution
      ↓
Transaction
      ↓
Storage API
      ↓
Page / Buffer / WAL
      ↓
FileChannel
      ↓
Operating System / Disk
```

The SQL layer should never call `FileChannel` directly.

------------------------------------------------------------------------

## 3. Define the Page Size

Start with 8 KiB:

``` java
public final class StorageConstants {

    public static final int PAGE_SIZE = 8192;

    private StorageConstants() {
    }
}
```

The physical offset of page `N` is:

``` text
offset = N × PAGE_SIZE
```

Example:

``` text
page 0 -> offset 0
page 1 -> offset 8192
page 2 -> offset 16384
page 10 -> offset 81920
```

------------------------------------------------------------------------

## 4. DatabaseDirectory

``` java
public final class DatabaseDirectory {

    private final Path root;

    public DatabaseDirectory(Path root) {
        this.root = root;
    }

    public Path database(String databaseName) {
        return root
                .resolve("databases")
                .resolve(databaseName);
    }

    public void createDatabase(String databaseName)
            throws IOException {

        Path db = database(databaseName);

        Files.createDirectories(db.resolve("tables"));
        Files.createDirectories(db.resolve("indexes"));
        Files.createDirectories(db.resolve("fsm"));
    }
}
```

Usage:

``` java
DatabaseDirectory directory =
        new DatabaseDirectory(Path.of("tinydb-data"));

directory.createDatabase("company");
```

------------------------------------------------------------------------

## 5. Table File

Each table gets one heap file initially:

``` text
employees.tbl
```

``` java
public final class TableFile {

    private final Path path;

    public TableFile(Path path) {
        this.path = path;
    }

    public void create() throws IOException {
        Files.createDirectories(path.getParent());

        if (Files.notExists(path)) {
            Files.createFile(path);
        }
    }

    public Path path() {
        return path;
    }
}
```

------------------------------------------------------------------------

## 6. Open the File with FileChannel

``` java
FileChannel channel = FileChannel.open(
        path,
        StandardOpenOption.CREATE,
        StandardOpenOption.READ,
        StandardOpenOption.WRITE
);
```

Use a single file manager abstraction so the rest of TinyDB does not
know how files are opened.

``` java
public interface FileManager {

    FileChannel openTable(Path path) throws IOException;

    void close() throws IOException;
}
```

------------------------------------------------------------------------

## 7. PageId and RowId

``` java
public record PageId(
        String database,
        String table,
        long pageNumber
) {
}
```

A physical row is identified by:

``` java
public record RowId(
        long pageNumber,
        int slotNumber
) {
}
```

Therefore:

``` text
employees.tbl
page = 12
slot = 4
```

becomes:

``` java
new RowId(12, 4);
```

Indexes can store:

``` text
employee_id -> RowId
```

rather than copying the whole row.

------------------------------------------------------------------------

## 8. Physical Page Layout

Use a slotted page.

``` text
+--------------------------------------------------+
| PAGE HEADER                                      |
+--------------------------------------------------+
| SLOT 0                                           |
| SLOT 1                                           |
| SLOT 2                                           |
+--------------------------------------------------+
|                                                  |
|                  FREE SPACE                      |
|                                                  |
+--------------------------------------------------+
| ROW DATA                                         |
| ROW DATA                                         |
| ROW DATA                                         |
+--------------------------------------------------+
```

The slot directory points to the physical row bytes.

Example:

``` text
Slot 0 -> offset 8100, length 92
Slot 1 -> offset 7980, length 120
Slot 2 -> offset 7860, length 120
```

This allows row bytes to move without changing the logical `RowId`.

------------------------------------------------------------------------

## 9. Page Header

A useful first format:

``` text
Offset  Size  Field
------  ----  ----------------
0       8     page number
8       8     page LSN
16      4     page type
20      4     slot count
24      4     free-space start
28      4     free-space end
32      4     checksum
```

This means the header is 36 bytes in this example.

Keep the binary format documented because old database files must remain
readable after the Java classes evolve.

------------------------------------------------------------------------

## 10. Page Class

``` java
public final class Page {

    private final PageId pageId;
    private final ByteBuffer buffer;

    private boolean dirty;

    public Page(PageId pageId) {
        this.pageId = pageId;
        this.buffer =
                ByteBuffer.allocate(StorageConstants.PAGE_SIZE);
    }

    public PageId pageId() {
        return pageId;
    }

    public ByteBuffer buffer() {
        return buffer;
    }

    public boolean dirty() {
        return dirty;
    }

    public void markDirty() {
        dirty = true;
    }

    public void clearDirty() {
        dirty = false;
    }
}
```

Later add:

``` text
pinCount
pageLSN
checksum
pageType
```

------------------------------------------------------------------------

## 11. Reading a Page

``` java
public Page readPage(
        FileChannel channel,
        PageId pageId
) throws IOException {

    Page page = new Page(pageId);

    ByteBuffer buffer = page.buffer();

    long offset =
            pageId.pageNumber()
            * StorageConstants.PAGE_SIZE;

    int totalRead = 0;

    while (buffer.hasRemaining()) {

        int read =
                channel.read(buffer, offset + totalRead);

        if (read < 0) {
            break;
        }

        totalRead += read;
    }

    if (totalRead == 0) {
        throw new EOFException(
                "Page does not exist: " + pageId
        );
    }

    buffer.flip();

    return page;
}
```

Important:

``` text
pageNumber × PAGE_SIZE
```

is the physical file offset.

------------------------------------------------------------------------

## 12. Writing a Page

``` java
public void writePage(
        FileChannel channel,
        Page page
) throws IOException {

    ByteBuffer buffer =
            page.buffer().duplicate();

    buffer.rewind();

    long offset =
            page.pageId().pageNumber()
            * StorageConstants.PAGE_SIZE;

    long position = offset;

    while (buffer.hasRemaining()) {
        int written =
                channel.write(buffer, position);

        if (written <= 0) {
            throw new IOException(
                    "Unable to write page"
            );
        }

        position += written;
    }
}
```

Do not assume one `FileChannel.write()` writes the complete buffer.

------------------------------------------------------------------------

## 13. Force Data to Storage

``` java
channel.force(true);
```

But do not use this as a substitute for WAL.

The durability rule is:

``` text
WAL durable first
       ↓
data page may become durable
```

------------------------------------------------------------------------

## 14. Row Binary Format

For a row:

``` java
record Employee(
        long id,
        String name,
        String email,
        int age
) {}
```

encode:

``` text
id
name length
name bytes
email length
email bytes
age
```

For example:

``` text
+---------+------------+-------------+---------+
| id 8 B  | name      | email       | age 4 B |
+---------+------------+-------------+---------+
```

Variable strings use:

``` text
4-byte length
N bytes UTF-8
```

------------------------------------------------------------------------

## 15. RowCodec

``` java
public interface RowCodec<T> {

    byte[] encode(T value);

    T decode(byte[] bytes);
}
```

Do not make the storage layer dependent on Java object serialization.

The production version should build codecs from `TableDefinition`.

------------------------------------------------------------------------

## 16. Null Values

A generic row needs null information.

One approach:

``` text
NULL BITMAP
COLUMN 1
COLUMN 2
COLUMN 3
...
```

For example:

``` text
null bitmap = 00101000
```

means selected columns are null.

This is more compact than storing a separate boolean object for every
column.

------------------------------------------------------------------------

## 17. Slot Directory

``` java
public record Slot(
        int offset,
        int length
) {
}
```

A page maintains:

``` text
slotCount
freeSpaceStart
freeSpaceEnd
```

Insert algorithm:

``` text
1. Encode row
2. Calculate required space
3. Check page capacity
4. Move free-space end backward
5. Write row bytes
6. Add slot
7. Increment slot count
8. Update header
9. Mark page dirty
```

------------------------------------------------------------------------

## 18. Page Insert

Conceptually:

``` java
public RowId insert(byte[] rowBytes) {

    if (!hasSpace(rowBytes.length)) {
        throw new PageFullException();
    }

    int slotNumber = slotCount;

    int rowOffset =
            freeSpaceEnd - rowBytes.length;

    buffer.position(rowOffset);
    buffer.put(rowBytes);

    writeSlot(
            slotNumber,
            rowOffset,
            rowBytes.length
    );

    slotCount++;

    freeSpaceEnd = rowOffset;

    markDirty();

    return new RowId(
            pageNumber,
            slotNumber
    );
}
```

The real implementation must reserve space for the slot itself.

------------------------------------------------------------------------

## 19. Reading a Row from a Page

``` text
RowId
 ↓
PageManager.read(page)
 ↓
read slot
 ↓
slot.offset
 ↓
slot.length
 ↓
copy bytes
 ↓
RowCodec.decode()
```

This is the fundamental point lookup path.

------------------------------------------------------------------------

## 20. TableHeap

``` java
public interface TableHeap {

    RowId insert(byte[] row);

    byte[] read(RowId rowId);

    void update(RowId rowId, byte[] row);

    void delete(RowId rowId);
}
```

Implementation:

``` text
TableHeap
   |
   +-- FreeSpaceMap
   +-- BufferPool
   +-- PageManager
   +-- RowCodec
```

------------------------------------------------------------------------

## 21. FreeSpaceMap

When inserting a row, TinyDB needs to find a page with enough space.

``` java
public interface FreeSpaceMap {

    Optional<Long> findPage(int requiredBytes);

    void update(long pageNumber, int freeBytes);
}
```

Initial implementation can simply scan pages.

Later:

``` text
FSM
 ↓
bucket by free-space percentage
 ↓
find candidate page quickly
```

------------------------------------------------------------------------

## 22. Buffer Pool

A buffer pool prevents repeated disk reads.

``` text
BufferPool
+--------------------------------+
| PageId -> BufferFrame          |
+--------------------------------+
| employees/0 -> page            |
| employees/1 -> page            |
| employees/2 -> page            |
+--------------------------------+
```

Interface:

``` java
public interface BufferPool {

    Page fetch(PageId pageId);

    void unpin(PageId pageId);

    void flush(PageId pageId);

    void flushAll();
}
```

A `BufferFrame` should track:

``` text
Page
pin count
dirty flag
last access
page LSN
```

------------------------------------------------------------------------

## 23. Page Fetch Algorithm

``` text
fetch(PageId)
    |
    +-- cache hit?
    |       |
    |      yes -> return page
    |
    +-- no
          |
          v
      choose victim
          |
          v
      if dirty:
          flush
          |
          v
      read page using FileChannel
          |
          v
      put into buffer pool
          |
          v
      return page
```

Pinned pages must never be evicted.

------------------------------------------------------------------------

## 24. LRU Eviction

Start with LRU.

``` text
capacity = 3

A B C
access A

order:
B C A

insert D

evict B
```

But:

``` text
pinned page
```

cannot be evicted.

A production implementation can later use Clock or another policy.

------------------------------------------------------------------------

## 25. Table File Manager

``` java
public final class TableFileManager {

    private final Path path;
    private final FileChannel channel;

    public TableFileManager(Path path)
            throws IOException {

        this.path = path;

        this.channel = FileChannel.open(
                path,
                StandardOpenOption.CREATE,
                StandardOpenOption.READ,
                StandardOpenOption.WRITE
        );
    }

    public FileChannel channel() {
        return channel;
    }

    public void close() throws IOException {
        channel.close();
    }
}
```

------------------------------------------------------------------------

## 26. Physical INSERT Implementation

Suppose:

``` sql
INSERT INTO employees
VALUES (1, 'Arpan', 'arpan@example.com', 32);
```

Actual storage sequence:

``` text
SQL parser
   ↓
InsertExecutor
   ↓
TransactionManager
   ↓
RowCodec.encode()
   ↓
FreeSpaceMap.findPage()
   ↓
BufferPool.fetch()
   ↓
Page.insert()
   ↓
WAL INSERT record
   ↓
page.markDirty()
   ↓
COMMIT WAL
   ↓
WAL force()
   ↓
success returned
   ↓
background flush
   ↓
FileChannel.write()
```

This is the concrete answer to:

> How does TinyDB write an INSERT to a file?

------------------------------------------------------------------------

## 27. WAL Must Be Before Data Page Flush

Suppose page 10 changes.

The page has:

``` text
pageLSN = 100
```

The WAL record has:

``` text
LSN = 100
```

Before page 10 is flushed:

``` text
durable WAL LSN >= 100
```

must be true.

Otherwise a crash could leave:

``` text
data page contains change
WAL does not contain change
```

and recovery cannot safely reason about the page.

------------------------------------------------------------------------

## 28. WAL Record Format

A useful first format:

``` text
+----------------------------+
| record length              |
+----------------------------+
| LSN                        |
+----------------------------+
| transaction id             |
+----------------------------+
| record type                |
+----------------------------+
| page id                    |
+----------------------------+
| row id                     |
+----------------------------+
| payload length             |
+----------------------------+
| payload                    |
+----------------------------+
| checksum                   |
+----------------------------+
```

Types:

``` java
public enum WalType {
    BEGIN,
    INSERT,
    UPDATE,
    DELETE,
    COMMIT,
    ABORT,
    CHECKPOINT
}
```

------------------------------------------------------------------------

## 29. WAL Writer

``` java
public interface WalManager {

    long append(WalRecord record)
            throws IOException;

    void flush(long lsn)
            throws IOException;

    long durableLsn();
}
```

The writer should serialize access to the WAL file so LSN ordering is
deterministic.

------------------------------------------------------------------------

## 30. WAL Append

``` text
transaction
   ↓
create WAL record
   ↓
serialize
   ↓
append to wal segment
   ↓
assign LSN
   ↓
return LSN
```

For commit:

``` text
BEGIN
INSERT
COMMIT
```

and the commit record must be durable before the client is told that the
transaction committed.

------------------------------------------------------------------------

## 31. Update

A first non-MVCC implementation:

``` text
read old row
   ↓
encode old image
   ↓
WAL UPDATE with old/new information
   ↓
modify page
   ↓
mark dirty
```

For MVCC:

``` text
old version remains
        ↓
new version created
        ↓
transaction visibility decides which version is visible
```

------------------------------------------------------------------------

## 32. Delete

Do not necessarily physically erase bytes immediately.

Initially:

``` text
row header:
deleted = true
```

or:

``` text
delete transaction id = X
```

Later garbage collection can reclaim the space.

This becomes important once MVCC is implemented.

------------------------------------------------------------------------

## 33. Crash Recovery

On startup:

``` text
open data directory
      ↓
read checkpoint
      ↓
scan WAL
      ↓
identify committed transactions
      ↓
redo required changes
      ↓
undo incomplete transactions
      ↓
rebuild/validate in-memory structures
      ↓
open for clients
```

Example:

``` text
T1 BEGIN
T1 INSERT
T1 COMMIT

T2 BEGIN
T2 INSERT

CRASH
```

Recovery:

``` text
T1 -> committed -> keep/replay
T2 -> incomplete -> undo
```

------------------------------------------------------------------------

## 34. Checkpoint

Without checkpoints:

``` text
startup
 ↓
scan entire WAL history
```

With checkpoints:

``` text
checkpoint
 ↓
remember recovery position
 ↓
startup
 ↓
scan WAL from checkpoint
```

Checkpoint metadata can contain:

``` text
checkpoint LSN
active transaction IDs
dirty page table
catalog version
```

------------------------------------------------------------------------

## 35. Page Checksums

Before writing:

``` text
calculate checksum
store checksum in header
```

After reading:

``` text
calculate checksum
compare with stored checksum
```

Mismatch:

``` java
throw new CorruptPageException(pageId);
```

Never silently continue with corrupted pages.

------------------------------------------------------------------------

## 36. Complete Physical Storage API

A useful storage-engine boundary:

``` java
public interface StorageEngine {

    DatabaseHandle createDatabase(
            String name
    );

    TableHandle createTable(
            TableDefinition definition
    );

    RowId insert(
            TransactionId txId,
            String table,
            byte[] row
    );

    byte[] read(
            TransactionId txId,
            String table,
            RowId rowId
    );

    void update(
            TransactionId txId,
            String table,
            RowId rowId,
            byte[] row
    );

    void delete(
            TransactionId txId,
            String table,
            RowId rowId
    );

    void flush();

    void checkpoint();
}
```

The execution engine uses this API rather than manipulating files.

------------------------------------------------------------------------

## 37. Integration With SQL

For:

``` sql
INSERT INTO employees
(id, name, email, age)
VALUES (1, 'Arpan', 'a@example.com', 32);
```

the SQL layer should produce an internal row:

``` text
{id=1, name="Arpan", email="a@example.com", age=32}
```

Then:

``` text
InsertExecutor
    ↓
TableMetadata
    ↓
RowCodec
    ↓
StorageEngine.insert()
```

Storage should not know that the original input came from SQL.

It only knows:

``` text
table
schema
encoded row
transaction
```

------------------------------------------------------------------------

## 38. Integration With MiniSpring

MiniSpring should never directly access:

``` text
employees.tbl
wal-000001.log
```

Instead:

``` text
MiniSpring
   ↓
TinyDB JDBC Driver
   ↓
TinyDB Server
   ↓
SQL / Execution
   ↓
Transaction
   ↓
StorageEngine
```

For embedded mode, MiniSpring can inject the storage interfaces
directly, but the boundary should remain the same.

------------------------------------------------------------------------

## 39. Embedded vs Server Mode

Support two modes.

### Embedded

``` text
Application
   ↓
TinyDB
   ↓
FileChannel
```

Useful for tests and local applications.

### Server

``` text
Application
   ↓
JDBC
   ↓
TCP
   ↓
TinyDB Server
   ↓
Storage
```

DBeaver normally uses server mode.

------------------------------------------------------------------------

## 40. Storage Test That Must Work

Write this test before building advanced SQL:

``` java
@Test
void shouldPersistRowAcrossRestart() throws Exception {

    Path root =
            Files.createTempDirectory("tinydb-test");

    // process 1
    TinyDatabase db =
            TinyDatabase.open(root);

    TableHandle employees =
            db.createTable(employeeSchema());

    RowId id =
            employees.insert(
                    employee(1, "Arpan")
            );

    db.close();

    // process 2
    TinyDatabase reopened =
            TinyDatabase.open(root);

    Employee employee =
            reopened
                    .table("employees")
                    .read(id);

    assertEquals(1, employee.id());
    assertEquals("Arpan", employee.name());

    reopened.close();
}
```

If this test fails, do not proceed to query optimization.

------------------------------------------------------------------------

## 41. Crash Test

Create:

``` text
BEGIN
INSERT
WAL flush
COMMIT
process crash
```

Restart.

Expected:

``` text
row exists
```

Then test:

``` text
BEGIN
INSERT
WAL INSERT
NO COMMIT
process crash
```

Expected:

``` text
row is not visible as committed
```

This test is the foundation of transaction correctness.

------------------------------------------------------------------------

## 42. Storage Implementation Checklist

### Files

-   [ ] root directory
-   [ ] database directories
-   [ ] table files
-   [ ] index files
-   [ ] WAL segments
-   [ ] checkpoint metadata
-   [ ] lock file

### Pages

-   [ ] fixed page size
-   [ ] page header
-   [ ] page LSN
-   [ ] checksum
-   [ ] slot directory
-   [ ] free-space tracking

### Rows

-   [ ] binary encoding
-   [ ] binary decoding
-   [ ] null bitmap
-   [ ] variable-length values
-   [ ] RowId

### Buffer Pool

-   [ ] page cache
-   [ ] pin/unpin
-   [ ] dirty pages
-   [ ] eviction
-   [ ] flush

### WAL

-   [ ] LSN
-   [ ] append
-   [ ] flush
-   [ ] commit record
-   [ ] checksums
-   [ ] segment rotation

### Recovery

-   [ ] checkpoint
-   [ ] WAL scan
-   [ ] redo
-   [ ] undo
-   [ ] incomplete transaction detection

### Concurrency

-   [ ] page synchronization
-   [ ] buffer pool synchronization
-   [ ] WAL writer serialization
-   [ ] transaction locking
-   [ ] deadlock detection

------------------------------------------------------------------------

## 43. Storage Layer Completion Criteria

The physical storage layer is complete enough for the next phase when
all of these work:

``` text
CREATE database directory
        ↓
CREATE table file
        ↓
CREATE page
        ↓
encode row
        ↓
insert row
        ↓
read row
        ↓
restart process
        ↓
read row again
        ↓
update row
        ↓
delete row
        ↓
crash process
        ↓
recover from WAL
```

Only after this should TinyDB rely on SQL to drive the storage engine.

------------------------------------------------------------------------

# Physical Storage: Final Mental Model

The complete physical path is:

``` text
SQL
 ↓
Parser
 ↓
AST
 ↓
Planner
 ↓
Executor
 ↓
Transaction Manager
 ↓
StorageEngine
 ↓
TableHeap
 ↓
FreeSpaceMap
 ↓
BufferPool
 ↓
Page
 ↓
RowCodec
 ↓
WAL
 ↓
FileChannel
 ↓
employees.tbl
```

And the recovery path is:

``` text
employees.tbl
       +
wal/*.log
       +
checkpoint.meta
       ↓
Recovery Manager
       ↓
redo / undo
       ↓
BufferPool
       ↓
consistent database state
```

The central design principle is:

> **SQL describes what the user wants. The execution engine decides how
> to do it. The transaction layer decides whether it is safe. The
> storage engine decides how bytes are persisted. WAL makes those
> changes recoverable.**
