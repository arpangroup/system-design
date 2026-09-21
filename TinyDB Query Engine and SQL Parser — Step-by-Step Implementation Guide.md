# TinyDB Query Engine and SQL Parser — Step-by-Step Implementation Guide

> **Goal:** Build the entire SQL processing stack of a database from scratch, as a standalone, self-contained project: a lexer that turns raw SQL text into tokens, a recursive-descent parser that turns tokens into an immutable AST, a set of statement executors that run directly against that AST, and — once that's working — a real query planner (`Plan`/`Scan` pipeline) and a persistent catalog manager that supersede direct execution the way every production database's query engine actually works.
>
> This guide is extracted and substantially expanded from the `TinyDB_From_Scratch_Java_Guide_FINAL.md` companion document's SQL-processing sections, reorganized to stand on its own — you should be able to implement everything in this guide without needing that document open. The storage layer this guide's `StorageEngine` interface talks to is built, in full, in the companion [TinyDB Storage Engine — Step-by-Step Implementation Guide.md](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>).

---

# 1. What We Are Building

A complete pipeline from raw SQL text to an executed result:

```text
"SELECT name FROM employee WHERE salary > 50000"
        |
        v
   SqlLexer            -- text -> tokens
        |
        v
   SqlParser            -- tokens -> SqlStatement (an immutable AST)
        |
        v
   QueryEngine           -- Level 1: execute the AST directly
        |                    Level 2: hand it to a Planner instead (Part 4 onward)
        v
   StorageEngine         -- built in the companion Storage Engine guide
```

By the end of this guide you will have:

- A **lexer** and a **hand-written recursive-descent parser** producing a sealed `SqlStatement` AST — no external grammar tool required.
- A small **`Expression`** AST and evaluator for `WHERE` clauses.
- **Six real statement executors** (`SelectExecutor`, `InsertExecutor`, `UpdateExecutor`, `DeleteExecutor`, `CreateTableExecutor`, `DropTableExecutor`) that run directly against a `StorageEngine`.
- A **`QueryEngine` dispatcher** using an exhaustive, compiler-checked `switch` over the sealed AST — the actual mechanism connecting lexer, parser, and executors.
- A full **query planner**: `Plan`/`Scan` abstractions, `TablePlan`/`SelectPlan`/`ProjectPlan`, `BasicQueryPlanner`/`BasicUpdatePlanner`, and a `BasicPlanner` facade — the same architecture real teaching databases (and, in spirit, production ones) use once "just execute the AST directly" stops being enough.
- A **persistent catalog manager** that stores table schema as rows in the database itself, not a throwaway in-memory map.
- A look at the **ANTLR + Visitor pattern** alternative real multi-dialect SQL engines (like Apache ShardingSphere) use instead of a hand-written parser.

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Implement a lexer and a recursive-descent parser for a small SQL grammar, and explain why a sealed AST interface makes adding a new statement type a compile-time-checked change.
- Explain the exact architectural seam between "text processing" (lexing/parsing) and "data processing" (execution) — and point to the one class where they meet.
- Explain the three parts of a query engine (Optimizer, Execution Engine, Catalog Manager) and build a working version of each.
- Explain the difference between Rule-Based and Cost-Based query optimization, with a concrete example.
- Implement a `Plan`/`Scan` pipeline that streams rows instead of materializing full result sets in memory, and explain precisely when that matters.
- Justify, with real tradeoffs, when a hand-written parser is the right choice and when ANTLR (or a similar parser generator) earns its keep instead.

---

# 3. Why This Matters (Interview Motivation)

> **"Implement a small SQL engine: parse a `SELECT`/`INSERT`/`UPDATE`/`DELETE` statement and execute it against an in-memory table. Then explain how you'd evolve this into something with a real query planner, and why a planner is worth the added complexity."**

This is a strong **backend/database-internals interview question** because it has a natural, escalating structure that separates candidates at every level:

- **Can you build a parser at all?** — tokenizing and recursive descent are fundamental, teachable-in-an-hour skills that a surprising number of engineers have never actually implemented.
- **Do you understand the AST as a real architectural boundary?** — not just "the parser's output," but the specific thing that lets lexing/parsing and execution evolve independently.
- **Can you name and build the three parts of a query engine**, rather than treating "the query engine" as one undifferentiated blob?
- **Do you know why a planner exists at all**, beyond "it's what real databases have" — the actual, concrete reason direct AST execution stops being sufficient?

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / JDK | Java 21 | Sealed interfaces, records, and exhaustive pattern-matching `switch` are used throughout — they are what make the AST-to-executor dispatch compiler-checked rather than a runtime `instanceof` chain. |
| Parsing technique | Hand-written lexer + recursive-descent parser | Small grammar, full control, zero build-tool dependency — see Part 8 for when this stops being the right choice. |
| Storage | The `StorageEngine` interface built in the companion [Storage Engine guide](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) | This guide's executors and planner depend only on that interface — swapping the in-memory version for the real page-based one requires no change here. |
| Testing | JUnit 5 | Parser and planner correctness is best proven with concrete input/output pairs, not manual inspection. |

---

# 5. Project Structure

```text
tinydb-query-engine/
├── src/main/java/com/tinydb/sql/
│   ├── lexer/
│   │   ├── Token.java, TokenType.java
│   │   └── SqlLexer.java
│   ├── ast/
│   │   ├── SqlStatement.java              // sealed interface
│   │   ├── SelectStatement.java, InsertStatement.java, UpdateStatement.java,
│   │   ├── DeleteStatement.java, CreateTableStatement.java, DropTableStatement.java
│   │   ├── Expression.java, Comparison.java, And.java, Or.java
│   │   └── ExpressionEvaluator.java
│   ├── parser/
│   │   └── SqlParser.java
│   ├── executor/
│   │   ├── StatementExecutor.java
│   │   ├── SelectExecutor.java, InsertExecutor.java, UpdateExecutor.java,
│   │   ├── DeleteExecutor.java, CreateTableExecutor.java, DropTableExecutor.java
│   │   └── QueryEngine.java
│   ├── plan/
│   │   ├── Plan.java, TablePlan.java, SelectPlan.java, ProjectPlan.java
│   │   ├── Scan.java, TableScan.java, SelectScan.java, ProjectScan.java
│   │   └── QueryPlanner.java, BasicQueryPlanner.java, UpdatePlanner.java, BasicUpdatePlanner.java
│   ├── engine/
│   │   ├── BasicPlanner.java
│   │   └── BasicQueryEngine.java
│   └── catalog/
│       ├── Catalog.java, TableSchema.java, Column.java, DataType.java
│       ├── TablePhysicalLayout.java
│       └── PersistentCatalog.java
└── src/test/java/com/tinydb/sql/
    ├── SqlLexerTest.java
    ├── SqlParserTest.java
    ├── ExpressionEvaluatorTest.java
    ├── QueryEngineTest.java
    └── BasicQueryPlannerTest.java
```

---

# 6. High-Level Architecture: Lexer → Parser → AST → Executors/Planner → Scan Pipeline

Two execution strategies are built in this guide, deliberately in sequence, not as alternatives to choose between arbitrarily:

```text
Level 1 (Part 3): SqlStatement -> Executor -> StorageEngine directly
                   Simple, correct, materializes full result sets in memory.

Level 2 (Part 4-5): SqlStatement -> Planner -> Plan -> Scan -> StorageEngine
                   More moving parts, streams rows one at a time, and is the
                   shape a Cost-Based Optimizer (Part 6) can eventually plug into.
```

Building Level 1 first and Level 2 second is deliberate pedagogy, not an accident of this guide's structure — Part 4 opens by naming exactly what Level 1 cannot do, which is what actually motivates every piece of Level 2's added complexity.

---

# 7. Phase 1 — Tokens and the SQL Lexer

A **lexer** turns raw text into a flat sequence of **tokens** — the smallest meaningful pieces of the language (keywords, identifiers, literals, operators, punctuation) — with no notion yet of how they combine into a statement. That structural question is entirely the parser's job (Part 2).

```java
// lexer/TokenType.java
public enum TokenType {
    // Keywords
    SELECT, FROM, WHERE, INSERT, INTO, VALUES, UPDATE, SET, DELETE,
    CREATE, TABLE, DROP, AND, OR, PRIMARY, KEY,
    // Literals
    IDENTIFIER, NUMBER, STRING,
    // Punctuation and operators
    STAR, COMMA, LPAREN, RPAREN, EQ, NEQ, GT, GTE, LT, LTE,
    // End of input
    EOF
}
```

```java
// lexer/Token.java
public record Token(TokenType type, String text) { }
```

```java
// lexer/SqlLexer.java
public final class SqlLexer {

    private final String sql;
    private int position = 0;

    public SqlLexer(String sql) {
        this.sql = sql;
    }

    public List<Token> tokenize() {
        List<Token> tokens = new ArrayList<>();
        Token token;
        while ((token = nextToken()).type() != TokenType.EOF) {
            tokens.add(token);
        }
        tokens.add(token); // include the EOF sentinel — the parser uses it to know when to stop
        return tokens;
    }

    private Token nextToken() {
        skipWhitespace();
        if (position >= sql.length()) return new Token(TokenType.EOF, "");

        char c = sql.charAt(position);

        if (Character.isLetter(c) || c == '_') return readIdentifierOrKeyword();
        if (Character.isDigit(c)) return readNumber();
        if (c == '\'') return readStringLiteral();

        return readOperatorOrPunctuation();
    }

    private void skipWhitespace() {
        while (position < sql.length() && Character.isWhitespace(sql.charAt(position))) position++;
    }

    private Token readIdentifierOrKeyword() {
        int start = position;
        while (position < sql.length() && (Character.isLetterOrDigit(sql.charAt(position)) || sql.charAt(position) == '_')) {
            position++;
        }
        String text = sql.substring(start, position);
        TokenType keyword = keywordOrNull(text.toUpperCase());
        return new Token(keyword != null ? keyword : TokenType.IDENTIFIER, text);
    }

    private TokenType keywordOrNull(String upper) {
        return switch (upper) {
            case "SELECT" -> TokenType.SELECT;
            case "FROM" -> TokenType.FROM;
            case "WHERE" -> TokenType.WHERE;
            case "INSERT" -> TokenType.INSERT;
            case "INTO" -> TokenType.INTO;
            case "VALUES" -> TokenType.VALUES;
            case "UPDATE" -> TokenType.UPDATE;
            case "SET" -> TokenType.SET;
            case "DELETE" -> TokenType.DELETE;
            case "CREATE" -> TokenType.CREATE;
            case "TABLE" -> TokenType.TABLE;
            case "DROP" -> TokenType.DROP;
            case "AND" -> TokenType.AND;
            case "OR" -> TokenType.OR;
            case "PRIMARY" -> TokenType.PRIMARY;
            case "KEY" -> TokenType.KEY;
            default -> null; // not a keyword -- caller falls back to IDENTIFIER
        };
    }

    private Token readNumber() {
        int start = position;
        while (position < sql.length() && Character.isDigit(sql.charAt(position))) position++;
        return new Token(TokenType.NUMBER, sql.substring(start, position));
    }

    private Token readStringLiteral() {
        position++; // consume the opening quote
        int start = position;
        while (position < sql.length() && sql.charAt(position) != '\'') position++;
        String text = sql.substring(start, position);
        position++; // consume the closing quote
        return new Token(TokenType.STRING, text);
    }

    private Token readOperatorOrPunctuation() {
        char c = sql.charAt(position);
        char next = position + 1 < sql.length() ? sql.charAt(position + 1) : '\0';

        return switch (c) {
            case '*' -> single(TokenType.STAR);
            case ',' -> single(TokenType.COMMA);
            case '(' -> single(TokenType.LPAREN);
            case ')' -> single(TokenType.RPAREN);
            case '=' -> single(TokenType.EQ);
            case '!' when next == '=' -> doubleChar(TokenType.NEQ);
            case '>' when next == '=' -> doubleChar(TokenType.GTE);
            case '>' -> single(TokenType.GT);
            case '<' when next == '=' -> doubleChar(TokenType.LTE);
            case '<' -> single(TokenType.LT);
            default -> throw new IllegalArgumentException("Unexpected character: " + c);
        };
    }

    private Token single(TokenType type) {
        String text = sql.substring(position, position + 1);
        position++;
        return new Token(type, text);
    }

    private Token doubleChar(TokenType type) {
        String text = sql.substring(position, position + 2);
        position += 2;
        return new Token(type, text);
    }
}
```

Every branch above resolves in a small, fixed number of character comparisons — a lexer's entire job is turning `O(text length)` characters into `O(text length)` tokens, with no lookahead beyond a single character (the `when next == '='` guards above are the only place this lexer looks more than one character ahead, and only by exactly one).

---

# 8. The SQL Grammar We Support (Start Small)

```text
statement
    := select | insert | update | delete | createTable | dropTable

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
    := '=' | '!=' | '>' | '>=' | '<' | '<='
```

No `JOIN`, no `ORDER BY`, no aggregate functions, no subqueries — Part 9 names these explicitly as future work, not silent omissions. A grammar this small is deliberately proportionate to a hand-written recursive-descent parser (Part 2) — earning a parser generator like ANTLR (Part 3's alternative) only becomes worthwhile once the grammar grows well past this.

---

# 9. The Domain Model: Row, TableSchema, Column, and DataType

Before the AST can reference a table's shape (§12's `CREATE TABLE` parsing needs `Column`/`DataType`/`TableSchema` immediately), the small set of plain data types every later section builds on needs to exist:

```java
// catalog/DataType.java
public enum DataType {
    INT, BIGINT, BOOLEAN, VARCHAR, DECIMAL, DATE, TIMESTAMP
}
```

```java
// catalog/Column.java
public record Column(String name, DataType type, boolean nullable) { }
```

```java
// catalog/TableSchema.java
public record TableSchema(String tableName, List<Column> columns) {

    public List<String> fieldNames() {
        return columns.stream().map(Column::name).toList();
    }

    public TableSchema project(List<String> fieldNames) {
        List<Column> projected = columns.stream().filter(c -> fieldNames.contains(c.name())).toList();
        return new TableSchema(tableName, projected);
    }
}
```

```java
// A row is deliberately a thin wrapper over a Map for this guide's purposes — see the
// companion Storage Engine guide for the REAL binary, on-disk row representation this
// in-memory shape stands in for at execution time.
public final class Row {

    private final Map<String, Object> values;

    public Row(Map<String, Object> values) {
        this.values = Map.copyOf(values);
    }

    public Object get(String column) { return values.get(column); }

    public Set<String> columnNames() { return values.keySet(); }
}
```

```java
// catalog/Catalog.java
public interface Catalog {

    void createTable(TableSchema schema);

    void dropTable(String tableName);

    TableSchema getTable(String tableName);

    boolean tableExists(String tableName);

    List<TableSchema> listTables();
}
```

`Catalog` is deliberately an interface, not a concrete class, from the very first line it's introduced — Part 6 builds two implementations (an in-memory one for Part 3's Level 1 execution, and `PersistentCatalog` for Part 6's real, durable version), and no code anywhere in this guide is ever written against a specific one of them.

---

# 10. Phase 2 — The SqlStatement AST

The parser's entire output is one value: a `SqlStatement`. Making it a `sealed interface` is the single most consequential design decision in this guide — every later dispatch (executors in Part 3, the planner in Part 4) gets compiler-enforced exhaustiveness from this one choice:

```java
// ast/SqlStatement.java
public sealed interface SqlStatement
        permits SelectStatement, InsertStatement, UpdateStatement,
                DeleteStatement, CreateTableStatement, DropTableStatement {
}
```

```java
public record SelectStatement(
        List<String> columns,
        String table,
        Expression where   // null if there was no WHERE clause
) implements SqlStatement { }

public record InsertStatement(
        String table,
        List<String> columns,
        List<Object> values
) implements SqlStatement { }

public record UpdateStatement(
        String table,
        Map<String, Object> assignments,
        Expression where
) implements SqlStatement { }

public record DeleteStatement(
        String table,
        Expression where
) implements SqlStatement { }

public record CreateTableStatement(
        TableSchema schema
) implements SqlStatement { }

public record DropTableStatement(
        String table
) implements SqlStatement { }
```

Every one of these records is a plain, immutable fact about what the user asked for — no behavior, no reference back to the tokens it was parsed from. That immutability is what lets the exact same `SqlStatement` object be handed to an executor (Part 3), a planner (Part 4), or a test assertion, with no risk of one consumer's use of it affecting another's.

---

# 11. Phase 3 — Recursive-Descent Parsing: From Tokens to SqlStatement

**Recursive descent** means: one method per grammar rule (§8), each method consuming tokens and calling other rule-methods for sub-rules, mirroring the grammar's own structure almost line for line.

```java
// parser/SqlParser.java
public final class SqlParser {

    private final List<Token> tokens;
    private int position = 0;

    public SqlParser(List<Token> tokens) {
        this.tokens = tokens;
    }

    public SqlStatement parseStatement() {
        return switch (peek().type()) {
            case SELECT -> parseSelect();
            case INSERT -> parseInsert();
            case UPDATE -> parseUpdate();
            case DELETE -> parseDelete();
            case CREATE -> parseCreateTable();
            case DROP -> parseDropTable();
            default -> throw new IllegalArgumentException("Unexpected token: " + peek());
        };
    }

    // ---- Token stream helpers ----

    private Token peek() { return tokens.get(position); }

    private Token advance() { return tokens.get(position++); }

    private Token expect(TokenType type) {
        Token token = advance();
        if (token.type() != type) {
            throw new IllegalArgumentException("Expected " + type + " but found " + token);
        }
        return token;
    }

    private boolean check(TokenType type) { return peek().type() == type; }
}
```

`peek`/`advance`/`expect` are the entire vocabulary a recursive-descent parser needs — look at the next token without consuming it, consume the next token, or consume it while asserting it's the type you expected. Every grammar rule below is built from exactly these three operations.

---

# 12. Representing INSERT, UPDATE, DELETE, CREATE TABLE, DROP TABLE

```java
// SqlParser.java, continued

private SelectStatement parseSelect() {
    expect(TokenType.SELECT);
    List<String> columns = parseSelectList();
    expect(TokenType.FROM);
    String table = expect(TokenType.IDENTIFIER).text();
    Expression where = check(TokenType.WHERE) ? parseWhereClause() : null;
    return new SelectStatement(columns, table, where);
}

private List<String> parseSelectList() {
    if (check(TokenType.STAR)) {
        advance();
        return List.of("*");
    }
    List<String> columns = new ArrayList<>();
    columns.add(expect(TokenType.IDENTIFIER).text());
    while (check(TokenType.COMMA)) {
        advance();
        columns.add(expect(TokenType.IDENTIFIER).text());
    }
    return columns;
}

private InsertStatement parseInsert() {
    expect(TokenType.INSERT);
    expect(TokenType.INTO);
    String table = expect(TokenType.IDENTIFIER).text();

    expect(TokenType.LPAREN);
    List<String> columns = parseIdentifierList();
    expect(TokenType.RPAREN);

    expect(TokenType.VALUES);
    expect(TokenType.LPAREN);
    List<Object> values = parseValueList();
    expect(TokenType.RPAREN);

    return new InsertStatement(table, columns, values);
}

private List<String> parseIdentifierList() {
    List<String> identifiers = new ArrayList<>();
    identifiers.add(expect(TokenType.IDENTIFIER).text());
    while (check(TokenType.COMMA)) {
        advance();
        identifiers.add(expect(TokenType.IDENTIFIER).text());
    }
    return identifiers;
}

private List<Object> parseValueList() {
    List<Object> values = new ArrayList<>();
    values.add(parseLiteral());
    while (check(TokenType.COMMA)) {
        advance();
        values.add(parseLiteral());
    }
    return values;
}

private Object parseLiteral() {
    Token token = advance();
    return switch (token.type()) {
        case NUMBER -> Long.parseLong(token.text());
        case STRING -> token.text();
        default -> throw new IllegalArgumentException("Expected a literal but found " + token);
    };
}

private UpdateStatement parseUpdate() {
    expect(TokenType.UPDATE);
    String table = expect(TokenType.IDENTIFIER).text();
    expect(TokenType.SET);

    Map<String, Object> assignments = new LinkedHashMap<>();
    assignments.putAll(parseAssignment());
    while (check(TokenType.COMMA)) {
        advance();
        assignments.putAll(parseAssignment());
    }

    Expression where = check(TokenType.WHERE) ? parseWhereClause() : null;
    return new UpdateStatement(table, assignments, where);
}

private Map<String, Object> parseAssignment() {
    String column = expect(TokenType.IDENTIFIER).text();
    expect(TokenType.EQ);
    Object value = parseLiteral();
    return Map.of(column, value);
}

private DeleteStatement parseDelete() {
    expect(TokenType.DELETE);
    expect(TokenType.FROM);
    String table = expect(TokenType.IDENTIFIER).text();
    Expression where = check(TokenType.WHERE) ? parseWhereClause() : null;
    return new DeleteStatement(table, where);
}

private CreateTableStatement parseCreateTable() {
    expect(TokenType.CREATE);
    expect(TokenType.TABLE);
    String table = expect(TokenType.IDENTIFIER).text();

    expect(TokenType.LPAREN);
    List<Column> columns = new ArrayList<>();
    columns.add(parseColumnDefinition());
    while (check(TokenType.COMMA)) {
        advance();
        columns.add(parseColumnDefinition());
    }
    expect(TokenType.RPAREN);

    return new CreateTableStatement(new TableSchema(table, columns));
}

private Column parseColumnDefinition() {
    String name = expect(TokenType.IDENTIFIER).text();
    DataType type = DataType.valueOf(expect(TokenType.IDENTIFIER).text().toUpperCase());
    boolean isPrimaryKey = false;
    if (check(TokenType.PRIMARY)) {
        advance();
        expect(TokenType.KEY);
        isPrimaryKey = true;
    }
    return new Column(name, type, !isPrimaryKey); // a primary key column is implicitly NOT NULL
}

private DropTableStatement parseDropTable() {
    expect(TokenType.DROP);
    expect(TokenType.TABLE);
    String table = expect(TokenType.IDENTIFIER).text();
    return new DropTableStatement(table);
}
```

Every one of these methods reads, top to bottom, almost exactly like the grammar rule it implements (§8) — that correspondence is recursive descent's biggest practical advantage: a grammar change usually maps to a small, localized, easy-to-locate change in exactly one method.

---

# 13. Phase 4 — The WHERE Clause: A Small Expression AST

```java
// ast/Expression.java
public sealed interface Expression permits Comparison, And, Or { }

public record Comparison(String column, String operator, Object value) implements Expression { }
public record And(Expression left, Expression right) implements Expression { }
public record Or(Expression left, Expression right) implements Expression { }
```

```java
// SqlParser.java, continued — implements expression := comparison | expression AND expression | expression OR expression
private Expression parseWhereClause() {
    expect(TokenType.WHERE);
    return parseExpression();
}

private Expression parseExpression() {
    Expression left = parseComparison();
    while (check(TokenType.AND) || check(TokenType.OR)) {
        boolean isAnd = check(TokenType.AND);
        advance();
        Expression right = parseComparison();
        left = isAnd ? new And(left, right) : new Or(left, right);
    }
    return left;
}

private Comparison parseComparison() {
    String column = expect(TokenType.IDENTIFIER).text();
    String operator = parseOperator();
    Object value = parseLiteral();
    return new Comparison(column, operator, value);
}

private String parseOperator() {
    Token token = advance();
    return switch (token.type()) {
        case EQ -> "="; case NEQ -> "!="; case GT -> ">";
        case GTE -> ">="; case LT -> "<"; case LTE -> "<=";
        default -> throw new IllegalArgumentException("Expected a comparison operator but found " + token);
    };
}
```

`salary > 50000 AND department = 'Engineering'` parses into `And(Comparison("salary", ">", 50000), Comparison("department", "=", "Engineering"))` — a tree shape §13's evaluator walks recursively, matching the sealed interface's own shape exactly.

---

# 14. Implementing the Expression Evaluator

```java
// ast/ExpressionEvaluator.java
public final class ExpressionEvaluator {

    public static boolean matches(Row row, Expression expression) {
        return switch (expression) {
            case Comparison c -> evaluateComparison(row.get(c.column()), c);
            case And a -> matches(row, a.left()) && matches(row, a.right());
            case Or o -> matches(row, o.left()) || matches(row, o.right());
        };
    }

    static boolean evaluateComparison(Object actual, Comparison c) {
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

`Expression` being a sealed interface means `matches`'s `switch` needs **no `default` branch** — and the compiler refuses to build this file if a fourth expression type (say, a future `Not`) is ever added to the `permits` clause without a matching `case` here. That guarantee is what stops a new expression type from silently falling through to "matches nothing" instead of a compile error.

---

# 15. Phase 5 — QueryResult and TransactionContext

Two small supporting types every executor (§17) and the dispatcher (§18) need:

```java
public record QueryResult(List<Row> rows, int affectedRows) {

    public static QueryResult ofRows(List<Row> rows) {
        return new QueryResult(rows, rows.size());
    }

    public static QueryResult ofAffected(int count) {
        return new QueryResult(List.of(), count);
    }
}
```

```java
public final class TransactionContext {

    // Deliberately empty for now. This guide focuses on parsing/planning/execution;
    // real isolation, locking, and rollback belong to a transaction manager, covered
    // in depth by the companion Storage Engine guide's crash-recovery and WAL sections.
    // Until a real TransactionContext is wired in, every statement is its own
    // implicit, auto-committed transaction.
    public static final TransactionContext AUTOCOMMIT = new TransactionContext();
}
```

---

# 16. Phase 6 — A Minimal StorageEngine to Execute Against

The executors need something real underneath them. This in-memory version is the Level 1 stand-in — the companion [Storage Engine guide](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) builds the real, page-based, durable implementation of this exact same interface.

```java
public interface StorageEngine {

    void createTable(TableSchema schema);

    void dropTable(String tableName);

    void insert(String tableName, Row row);

    List<Row> scan(String tableName);

    void replaceRows(String tableName, List<Row> rows);

    Scan openScan(String tableName); // streaming access — used from Part 5 onward
}
```

```java
public final class MemoryStorageEngine implements StorageEngine {

    private final Map<String, List<Row>> tables = new ConcurrentHashMap<>();

    @Override
    public void createTable(TableSchema schema) {
        tables.putIfAbsent(schema.tableName(), new CopyOnWriteArrayList<>());
    }

    @Override
    public void dropTable(String tableName) { tables.remove(tableName); }

    @Override
    public void insert(String tableName, Row row) { tables.get(tableName).add(row); }

    @Override
    public List<Row> scan(String tableName) { return List.copyOf(tables.get(tableName)); }

    @Override
    public void replaceRows(String tableName, List<Row> rows) {
        tables.put(tableName, new CopyOnWriteArrayList<>(rows));
    }

    @Override
    public Scan openScan(String tableName) {
        // Implemented in full in Part 5 (§21), once the Scan interface itself is introduced.
        throw new UnsupportedOperationException("see Part 5");
    }
}
```

`replaceRows` is a deliberately blunt instrument: `UpdateExecutor`/`DeleteExecutor` (§17) compute a new row list and swap the whole thing in, rather than mutating individual rows — correct and simple here, and honestly wasteful once a table is large, which is exactly the problem the real page-based storage engine (the companion guide) solves properly.

---

# 17. Phase 7 — The Six Statement Executors

Each executor implements one small, typed contract:

```java
public interface StatementExecutor<S extends SqlStatement> {
    QueryResult execute(S statement, TransactionContext transaction);
}
```

**SelectExecutor**

```java
public final class SelectExecutor implements StatementExecutor<SelectStatement> {

    private final StorageEngine storageEngine;

    public SelectExecutor(StorageEngine storageEngine) { this.storageEngine = storageEngine; }

    @Override
    public QueryResult execute(SelectStatement statement, TransactionContext transaction) {
        List<Row> matching = storageEngine.scan(statement.table()).stream()
                .filter(row -> statement.where() == null || ExpressionEvaluator.matches(row, statement.where()))
                .toList();

        if (statement.columns().size() == 1 && statement.columns().get(0).equals("*")) {
            return QueryResult.ofRows(matching);
        }
        List<Row> projected = matching.stream().map(row -> projectColumns(row, statement.columns())).toList();
        return QueryResult.ofRows(projected);
    }

    private Row projectColumns(Row row, List<String> columns) {
        Map<String, Object> projected = new LinkedHashMap<>();
        for (String column : columns) projected.put(column, row.get(column));
        return new Row(projected);
    }
}
```

**InsertExecutor**

```java
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
        if (schema == null) throw new IllegalStateException("No such table: " + statement.table());
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

```java
public final class UpdateExecutor implements StatementExecutor<UpdateStatement> {

    private final StorageEngine storageEngine;

    public UpdateExecutor(StorageEngine storageEngine) { this.storageEngine = storageEngine; }

    @Override
    public QueryResult execute(UpdateStatement statement, TransactionContext transaction) {
        List<Row> current = storageEngine.scan(statement.table());
        List<Row> updated = new ArrayList<>();
        int affected = 0;

        for (Row row : current) {
            if (statement.where() == null || ExpressionEvaluator.matches(row, statement.where())) {
                Map<String, Object> newValues = new LinkedHashMap<>();
                for (String column : row.columnNames()) newValues.put(column, row.get(column));
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

```java
public final class DeleteExecutor implements StatementExecutor<DeleteStatement> {

    private final StorageEngine storageEngine;

    public DeleteExecutor(StorageEngine storageEngine) { this.storageEngine = storageEngine; }

    @Override
    public QueryResult execute(DeleteStatement statement, TransactionContext transaction) {
        List<Row> current = storageEngine.scan(statement.table());
        List<Row> kept = new ArrayList<>();
        int affected = 0;

        for (Row row : current) {
            boolean shouldDelete = statement.where() == null || ExpressionEvaluator.matches(row, statement.where());
            if (shouldDelete) affected++; else kept.add(row);
        }
        storageEngine.replaceRows(statement.table(), kept);
        return QueryResult.ofAffected(affected);
    }
}
```

**CreateTableExecutor**

```java
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

```java
public final class DropTableExecutor implements StatementExecutor<DropTableStatement> {

    private final Catalog catalog;
    private final StorageEngine storageEngine;

    public DropTableExecutor(Catalog catalog, StorageEngine storageEngine) {
        this.catalog = catalog;
        this.storageEngine = storageEngine;
    }

    @Override
    public QueryResult execute(DropTableStatement statement, TransactionContext transaction) {
        if (!catalog.tableExists(statement.table())) throw new IllegalStateException("No such table: " + statement.table());
        catalog.dropTable(statement.table());
        storageEngine.dropTable(statement.table());
        return QueryResult.ofAffected(0);
    }
}
```

Every executor above depends only on `Catalog` and `StorageEngine` — **interfaces**, never `MemoryStorageEngine` directly. Swapping in the companion guide's real, page-based `FileStorageEngine` later means constructing these executors with a different object; not one line of executor logic changes.

---

# 18. Phase 8 — The QueryEngine Dispatcher: Wiring Lexer, Parser, and Executors Together

This is the answer to "how do lexer, parser, and executors actually connect":

```java
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
        List<Token> tokens = new SqlLexer(sql).tokenize();            // §7
        SqlStatement statement = new SqlParser(tokens).parseStatement(); // §11
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

Two details make this dispatcher the actual seam this whole guide has been building toward:

1. `SqlStatement` is `sealed ... permits Select, Insert, Update, Delete, CreateTable, DropTable`. The `switch` in `execute` needs **no `default` branch**, and the compiler refuses to compile if a seventh statement type is ever added without a matching `case` here — adding `CREATE INDEX` later is a compile error until this switch is updated, not a runtime surprise discovered months later.
2. `run(String sql)` is the **only** method that touches the lexer or parser at all. Every executor, and `execute(SqlStatement, ...)` itself, only ever sees the already-parsed AST. Lexing/parsing is a text-processing concern; execution is a data-processing concern; `QueryEngine.run` is the one seam where they meet.

---

# 19. Alternative Architecture: ANTLR-Generated Parsing + the Visitor Pattern

A hand-written lexer and recursive-descent parser (§7, §11) is not the only legitimate way to build this — **ANTLR** (a parser generator: write a `.g4` grammar file, get a lexer, a parser, and a base `Visitor` class generated for you) is a real, production-proven alternative. Apache ShardingSphere's MySQL frontend, for one, is built exactly this way:

```java
// Uses an ANTLR-generated MySQLLexer/MySQLStatementParser instead of a hand-written one.
public class MySqlParser implements IParser {
    MySqlStatementVisitor sqlStatementVisitor;

    public MySqlParser(String sql) {
        MySQLLexer lexer = new MySQLLexer(CharStreams.fromString(sql));
        MySQLStatementParser parser = new MySQLStatementParser(new CommonTokenStream(lexer));

        sqlStatementVisitor = new MySqlStatementVisitor(parser);
        sqlStatementVisitor.visit(parser.execute());
    }

    @Override
    public QueryData queryCmd() { return (QueryData) sqlStatementVisitor.getValue(); }

    @Override
    public Object updateCmd() { return sqlStatementVisitor.getValue(); }
}
```

```java
// The Visitor walks ANTLR's generated parse tree and extracts exactly the fields needed
// into a domain object -- the ANTLR equivalent of this guide's SqlStatement records.
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

```text
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
 (an exhaustive switch)          (structurally the same idea, without a
                                  sealed-type-checked switch, since ANTLR's
                                  generated node types are not a sealed
                                  hierarchy this codebase controls)
```

**Why this guide does not start there:** a hand-rolled recursive-descent parser producing a sealed `SqlStatement` gets exhaustiveness checking from the Java compiler itself (§18's point 1) with zero extra tooling. ANTLR earns its keep once the grammar gets large enough that hand-writing and hand-maintaining a parser for it becomes the bigger cost — full dialect compatibility (MySQL, Postgres, and their many edge cases) is exactly that situation, which is why real multi-dialect tools reach for a grammar file and a generated parser instead. Know both; pick the hand-written one for as long as §8's grammar stays small, and reach for ANTLR when it stops being small.

---

# 20. Why Direct Execution Doesn't Scale: Introducing the Planner

§17's executors work correctly, and they share one real limitation: `SelectExecutor.execute` always calls `storageEngine.scan(table)` — a **full table scan**, every single time, even for `WHERE id = 100` against a table with a primary-key index that could answer the same question in `O(log n)` instead of `O(n)`. Nothing in §17's code has anywhere to *record* "use the index here" as a decision — the scan-then-filter logic is hard-coded inline. Fixing this requires separating "what to do" (a `Plan`, an object you can inspect, compare, and choose between) from "just doing it" — which is the entire reason a **query planner** exists.

---

# 21. The Three Parts of a Query Engine

A query engine is not one component. It is three, each with a distinct job:

```text
Query Optimizer      -- decides HOW to answer a query (which plan, which indexes)
Execution Engine      -- actually reads rows/columns according to that plan
Catalog / Metadata     -- knows what tables and indexes exist, and their shape
Manager
```

Part 4 (this part) builds the Optimizer's mechanism and the seam it plugs into. Part 5 builds the Execution Engine. Part 6 builds the Catalog Manager for real. Keeping these three separate is the same Interface Segregation discipline (a `QueryPlanner`, a `QueryExecutor`, and a `Catalog` as three different interfaces) that motivates never merging them into one `DatabaseEverything` class.

---

# 22. Rule-Based vs Cost-Based Optimization

"Use the index if the `WHERE` clause matches an indexed column, otherwise scan the table" is **Rule-Based Optimization (RBO)** — a fixed rule, applied without comparing alternatives. A **Cost-Based Optimizer (CBO)** generalizes this: instead of one hard-coded rule, it generates several candidate plans and picks whichever has the lowest *estimated* cost, using statistics the Catalog Manager (Part 6) collects — row counts, blocks accessed, selectivity. RBO is simpler to build and reason about; CBO scales better as the number of indexes and join orderings grows, at the cost of needing real statistics to estimate against. This guide builds RBO's actual mechanism — §27's `Plan.blocksAccessed()`/`recordsOutput()` are exactly the hooks a later CBO would compare across candidate plans.

---

# 23. Phase 9 — The Plan Interface

A `Plan` describes *how* a piece of relational algebra will be executed, and can report an estimated cost for it — exactly what a Cost-Based Optimizer (§22) needs to compare alternatives:

```java
public interface Plan {

    Scan open();          // §29 -- calling this does not read any data yet, only prepares to

    int blocksAccessed();  // cost estimate, for a future CBO to compare across candidate plans

    int recordsOutput();

    TableSchema schema();
}
```

**TablePlan** — the only `Plan` that talks to `StorageEngine` directly:

```java
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
    public Scan open() { return storageEngine.openScan(tableName); } // the ONE seam into physical storage

    @Override
    public int blocksAccessed() { return catalog.getStatistics(tableName).blockCount(); }

    @Override
    public int recordsOutput() { return catalog.getStatistics(tableName).recordCount(); }

    @Override
    public TableSchema schema() { return catalog.getTable(tableName); }
}
```

**SelectPlan** and **ProjectPlan** — each wraps another `Plan`, the way a Decorator wraps another object, rather than talking to storage itself:

```java
public final class SelectPlan implements Plan {

    private final Plan input;
    private final Expression predicate;

    public SelectPlan(Plan input, Expression predicate) {
        this.input = input;
        this.predicate = predicate;
    }

    @Override
    public Scan open() { return new SelectScan(input.open(), predicate); } // §30

    @Override
    public int blocksAccessed() { return input.blocksAccessed(); } // filtering reads the SAME blocks -- no extra I/O

    @Override
    public int recordsOutput() { return input.recordsOutput() / 2; } // a naive fixed selectivity -- a CBO refines this

    @Override
    public TableSchema schema() { return input.schema(); } // filtering ROWS never changes which COLUMNS exist
}
```

```java
public final class ProjectPlan implements Plan {

    private final Plan input;
    private final List<String> fields;

    public ProjectPlan(Plan input, List<String> fields) {
        this.input = input;
        this.fields = fields;
    }

    @Override
    public Scan open() { return new ProjectScan(input.open(), fields); } // §30

    @Override
    public int blocksAccessed() { return input.blocksAccessed(); }

    @Override
    public int recordsOutput() { return input.recordsOutput(); } // projecting columns doesn't change row COUNT

    @Override
    public TableSchema schema() { return input.schema().project(fields); } // a NARROWER schema -- fewer columns
}
```

`TablePlan`, `SelectPlan`, and `ProjectPlan` compose exactly the way `SelectStatement`'s three parts (table, where, columns) already suggest: read, then filter, then narrow columns — each step wrapping the previous `Plan`, never reaching around it to touch storage directly.

---

# 24. Phase 10 — BasicQueryPlanner and UpdatePlanner

```java
public interface QueryPlanner {
    Plan createPlan(SelectStatement statement, TransactionContext tx);
}
```

```java
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

        if (statement.where() != null) plan = new SelectPlan(plan, statement.where()); // Step 2: filter rows

        boolean selectStar = statement.columns().size() == 1 && statement.columns().get(0).equals("*");
        if (!selectStar) plan = new ProjectPlan(plan, statement.columns()); // Step 3: filter columns

        return plan;
    }
}
```

Compare this three-step body to §17's `SelectExecutor` — the *logic* (read, then filter, then project) is identical. What changed is that each step is now a composed, reusable `Plan` object instead of an inline `.stream().filter(...)` call — the exact shape a Cost-Based Optimizer needs later, since it can only compare plans it can hold as objects, not inline expressions buried in an executor's method body.

```java
public interface UpdatePlanner {
    int executeInsert(InsertStatement statement, TransactionContext tx);
    int executeUpdate(UpdateStatement statement, TransactionContext tx);
    int executeDelete(DeleteStatement statement, TransactionContext tx);
    int executeCreateTable(CreateTableStatement statement, TransactionContext tx);
    int executeDropTable(DropTableStatement statement, TransactionContext tx);
}
```

```java
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
        Scan scan = plan.open();
        scan.insert(); // position at a fresh slot
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
            for (var entry : statement.assignments().entrySet()) scan.setVal(entry.getKey(), entry.getValue());
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
        while (scan.next()) { scan.delete(); affected++; }
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

`executeUpdate`/`executeDelete` reuse `SelectPlan` for their `WHERE` clause — filtering which rows to mutate is the *identical* operation as filtering which rows to read, so it is composed from the same `Plan`, not reimplemented.

---

# 25. Phase 11 — BasicPlanner: A Facade Over Parsing and Planning

`BasicPlanner` is a **Facade** sitting between the parser and the two planners above — the same seam idea as §18's `QueryEngine`, one layer deeper now that statements produce `Plan`s instead of being executed immediately:

```java
public final class BasicPlanner {

    private final QueryPlanner queryPlanner;
    private final UpdatePlanner updatePlanner;

    public BasicPlanner(QueryPlanner queryPlanner, UpdatePlanner updatePlanner) {
        this.queryPlanner = queryPlanner;
        this.updatePlanner = updatePlanner;
    }

    public Plan createQueryPlan(String sql, TransactionContext tx) {
        SqlStatement statement = parse(sql);
        if (!(statement instanceof SelectStatement select)) throw new IllegalArgumentException("Not a query: " + sql);
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
        List<Token> tokens = new SqlLexer(sql).tokenize(); // §7
        return new SqlParser(tokens).parseStatement();      // §11
    }
}
```

This is the exact same exhaustive-`switch`-over-a-sealed-interface trick §18's `QueryEngine.execute` already used — adding a `CREATE INDEX` statement later means the compiler refuses to build this method until a matching `case` is added here too.

---

# 26. Phase 12 — BasicQueryEngine: doQuery and doUpdate

The externally-visible database facade — what a CLI or a JDBC driver actually calls:

```java
public final class BasicQueryEngine {

    private final BasicPlanner planner;
    private final TransactionManager transactionManager; // see the companion Storage Engine guide

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
            for (String field : plan.schema().fieldNames()) values.put(field, scan.getVal(field));
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

`doQuery` is the only place left that walks a `Scan` into a `List<Row>` — and it does that only because the caller above it wants a materialized result set to print or return. The engine itself, all the way down through `Plan`/`Scan`, never materializes more than one row at a time (§32 explains exactly why that matters).

---

# 27. Revisiting the Six Executors to Use the Planner

Here is what §17 looks like once the planner (§23–§26) exists — for one executor; the rest follow the identical shape:

```java
// SelectExecutor, REVISITED: same StatementExecutor<SelectStatement> contract as §17,
// but the BODY now delegates to the planner instead of scanning+filtering a materialized List<Row> by hand.
public final class SelectExecutor implements StatementExecutor<SelectStatement> {

    private final QueryPlanner queryPlanner;

    public SelectExecutor(QueryPlanner queryPlanner) { this.queryPlanner = queryPlanner; }

    @Override
    public QueryResult execute(SelectStatement statement, TransactionContext transaction) {
        Plan plan = queryPlanner.createPlan(statement, transaction);
        Scan scan = plan.open();

        List<Row> rows = new ArrayList<>();
        while (scan.next()) {
            Map<String, Object> values = new LinkedHashMap<>();
            for (String field : plan.schema().fieldNames()) values.put(field, scan.getVal(field));
            rows.add(new Row(values));
        }
        scan.close();
        return QueryResult.ofRows(rows);
    }
}
```

The public contract did not change at all. Only what happens *inside* `execute` changed — from "load everything, then filter in Java" to "compose a plan, then stream through it." This is precisely why §17 was told to depend on `Catalog`/`StorageEngine` interfaces: the same Dependency Inversion that let `MemoryStorageEngine` be swapped for a real one also lets an executor's *implementation strategy* change without its callers noticing.

---

# 28. Phase 13 — The Scan Interface: Streaming Instead of Materializing

§16's `StorageEngine.scan(tableName)` returns a `List<Row>` — every row, loaded into memory, before anything even looks at the first one. That is fine for a table that fits comfortably in memory; it is a real problem the moment a table doesn't. A `Scan` fixes this by being an **iterator**, not a collection — it produces rows one at a time, on demand:

```java
public interface Scan {

    boolean next();

    Object getVal(String fieldName);

    boolean hasField(String fieldName);

    void close();

    // Writable operations -- a stricter design (see the note below) splits these
    // into a separate interface so a read-only SELECT path can never call them.
    void insert();
    void setVal(String fieldName, Object value);
    void delete();
}
```

Putting both read and write methods on one interface is a pragmatic simplification, not the final answer — Interface Segregation argues against exactly this shape in the abstract. A stricter design splits this into `ReadOnlyScan` (`next`/`getVal`/`hasField`/`close`) and `UpdatableScan extends ReadOnlyScan` (adding `insert`/`setVal`/`delete`), so a `SELECT` path is never even *able* to call `delete()` by accident — the compiler enforces it instead of a runtime check. §29's implementations work identically either way; only which interface each one is declared to return changes.

---

# 29. Phase 14 — TableScan

`TableScan` is the only `Scan` that talks to `StorageEngine` directly — everything else wraps another `Scan`:

```java
// StorageEngine (§16) gains one method beyond insert()/scan()/replaceRows():
public interface StorageEngine {
    // ... createTable, dropTable, insert, scan, replaceRows from §16 ...
    Scan openScan(String tableName);
}
```

```java
// MemoryStorageEngine.openScan -- still backed by the in-memory List<Row> for now.
// Once real page-based storage exists (the companion Storage Engine guide), this walks
// Pages via RowId instead of a Java Iterator -- the Scan CONTRACT above does not change.
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
        public Object getVal(String fieldName) { return rows.get(index).get(fieldName); }

        @Override
        public boolean hasField(String fieldName) { return rows.get(index).columnNames().contains(fieldName); }

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

---

# 30. Phase 15 — SelectScan and ProjectScan

**SelectScan** wraps a child `Scan` and a predicate — it filters rows as they stream past, one at a time, never holding more than the current row:

```java
public final class SelectScan implements Scan {

    private final Scan input;
    private final Expression predicate;

    public SelectScan(Scan input, Expression predicate) {
        this.input = input;
        this.predicate = predicate;
    }

    @Override
    public boolean next() {
        while (input.next()) {
            if (ExpressionEvaluator.matchesScan(this, predicate)) return true; // §31
        }
        return false;
    }

    @Override public Object getVal(String fieldName) { return input.getVal(fieldName); }
    @Override public boolean hasField(String fieldName) { return input.hasField(fieldName); }
    @Override public void setVal(String fieldName, Object value) { input.setVal(fieldName, value); }
    @Override public void insert() { input.insert(); }
    @Override public void delete() { input.delete(); }
    @Override public void close() { input.close(); }
}
```

**ProjectScan** wraps a child `Scan` and a field list — it filters *columns*, and explicitly rejects access to any column outside the projection, matching the "narrower schema" guard `ProjectPlan.schema()` (§23) already imposes:

```java
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

`ProjectScan` throwing on every write method is worth noticing — projection is fundamentally a read-shaping operation, and §28's "split into `ReadOnlyScan`/`UpdatableScan`" refinement would let this class simply not implement the write methods at all, rather than implementing them just to reject every call.

---

# 31. Extending the Expression Evaluator to Work Against a Scan

`SelectScan` (§30) needs to evaluate the same `Expression` tree §14 already built — but against a `Scan`'s current position, not a fully-formed `Row`. Extend `ExpressionEvaluator` with a second entry point that mirrors `matches(Row, Expression)` field for field, reading through `Scan.getVal(...)` instead of `Row.get(...)`:

```java
// Added to ExpressionEvaluator (§14):
public static boolean matchesScan(Scan scan, Expression expression) {
    return switch (expression) {
        case Comparison c -> evaluateComparison(scan.getVal(c.column()), c);
        case And a -> matchesScan(scan, a.left()) && matchesScan(scan, a.right());
        case Or o -> matchesScan(scan, o.left()) || matchesScan(scan, o.right());
    };
}
```

§14's `evaluateComparison` was already written to take the actual value directly (not a `Row`), specifically so both `matches(Row, ...)` and `matchesScan(Scan, ...)` can share that one comparison method instead of duplicating it — one comparison implementation, two entry points, which is what keeps `Row`-based execution (Part 3) and `Scan`-based execution (Part 5) from silently drifting into two different definitions of "matches."

---

# 32. Why Streaming Beats Materializing: A Concrete Cost Trace

Trace what happens for:

```sql
SELECT name FROM employee WHERE salary > 50000;
```

against a table of one million rows where ten thousand match:

```text
Part 3's original SelectExecutor (§17):
  storageEngine.scan("employee")                 -- loads ALL 1,000,000 rows into a List<Row>
    .stream().filter(...)                          -- then filters down to 10,000, still in memory
    .map(...)                                       -- then projects, producing a THIRD list

Part 4-5's Plan/Scan pipeline (§27-30):
  TableScan.next()  -> SelectScan.next()  -> ProjectScan.next()
  -- pulls ONE row at a time through all three stages, filters it, projects it, and only
     the caller (BasicQueryEngine.doQuery, §26) decides whether to keep it in a List --
     the pipeline itself never holds more than one row per stage at once.
```

Both eventually produce the same 10,000-row result. The difference is **peak memory**: one materializes three full intermediate collections along the way; the other never holds more than a handful of rows at any given instant, regardless of whether the table has a thousand rows or a billion. This is the concrete, measurable payoff of the Iterator-shaped `Scan` interface (§28) over §17's original "load a list, then `.stream()` it" approach — and it is also exactly why `Sort` and `Aggregate` (Part 9's future work) are structurally different from `Filter`/`Project`: sorting the whole result requires seeing every row before producing the first one, so a `SortScan` cannot stream the way `SelectScan`/`ProjectScan` do — a real, unavoidable limit on how far the streaming idea extends, worth stating honestly rather than implying every operator can be lazy for free.

---

# 33. Phase 16 — The Catalog Interface, Revisited

§9's `Catalog` interface has been used throughout this guide without a real, durable implementation — every example so far implicitly assumes something like a `Map<String, TableSchema>` in memory, which forgets every table the instant the process restarts. Part 6 fixes that properly.

```java
public interface Catalog {
    void createTable(TableSchema schema);
    void dropTable(String tableName);
    TableSchema getTable(String tableName);
    boolean tableExists(String tableName);
    List<TableSchema> listTables();
    TableStatistics getStatistics(String tableName); // consumed by TablePlan.blocksAccessed()/recordsOutput(), §23
}

public record TableStatistics(int blockCount, int recordCount) { }
```

---

# 34. Phase 17 — TablePhysicalLayout: From Logical Schema to Byte Offsets

`TableSchema` (§9) is a table's *logical* shape — names, types, nullability. Computing a row's exact physical layout needs one more thing: the byte **offset** of each column within a row, and the row's total **slot size**:

```java
public final class TablePhysicalLayout {

    private final Map<String, Integer> offsets = new HashMap<>();
    private final int slotSize;

    public TablePhysicalLayout(TableSchema schema) {
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
            case VARCHAR -> 50; // fixed-length slots for simplicity -- even VARCHAR gets a fixed budget
            case DECIMAL -> 12;
            case DATE, TIMESTAMP -> 8;
        };
    }

    public int offset(String fieldName) { return offsets.get(fieldName); }
    public int slotSize() { return slotSize; }
}
```

Fixed-length slots (even for `VARCHAR`) is a deliberate simplifying decision — worth restating because it's precisely what makes `offset(fieldName)` a single, cheap arithmetic lookup instead of "scan every earlier column to find where this one starts." The companion [Storage Engine guide](<TinyDB Storage Engine — Step-by-Step Implementation Guide.md>) uses exactly this layout to encode/decode rows at the byte level.

---

# 35. Phase 18 — PersistentCatalog: Storing Schema as Data

A real database stores its **own schema as ordinary rows**, in special tables the database itself manages, so table definitions get the exact same durability (WAL, covered in the companion Storage Engine guide) as user data, instead of a separate, ad hoc persistence mechanism:

```text
tinydb_tables               tinydb_columns
+-----------+-----------+   +-----------+-----------+--------+--------+
| tblname   | slotsize  |   | tblname   | fldname   | type   | length |
+-----------+-----------+   +-----------+-----------+--------+--------+
| employee  | 48        |   | employee  | id        | BIGINT | 0      |
|           |           |   | employee  | name      | VARCHAR| 50     |
+-----------+-----------+   +-----------+-----------+--------+--------+
```

```java
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
        int slotSize = new TablePhysicalLayout(schema).slotSize(); // §34
        storageEngine.insert(TABLES_SCHEMA.tableName(),
                new Row(Map.of("tblname", schema.tableName(), "slotsize", slotSize)));
        for (Column column : schema.columns()) {
            storageEngine.insert(COLUMNS_SCHEMA.tableName(), new Row(Map.of(
                    "tblname", schema.tableName(), "fldname", column.name(),
                    "type", column.type().name(), "length", 0)));
        }
    }

    @Override
    public TableSchema getTable(String tableName) {
        boolean exists = storageEngine.scan(TABLES_SCHEMA.tableName()).stream()
                .anyMatch(row -> row.get("tblname").equals(tableName));
        if (!exists) return null;

        List<Column> columns = storageEngine.scan(COLUMNS_SCHEMA.tableName()).stream()
                .filter(row -> row.get("tblname").equals(tableName))
                .map(row -> new Column((String) row.get("fldname"), DataType.valueOf((String) row.get("type")), true))
                .toList();
        return new TableSchema(tableName, columns);
    }

    // dropTable/tableExists/listTables/getStatistics omitted for brevity -- same
    // "read from tinydb_tables / tinydb_columns" pattern as getTable() above.
}
```

---

# 36. Bootstrapping: How the Catalog Describes Itself

The bootstrap step in `PersistentCatalog`'s constructor — creating `tinydb_tables` and `tinydb_columns` *using the same `storageEngine.createTable` every other table uses* — is the interesting part: the catalog describes every table, **including its own two tables**, through one uniform mechanism. This is the same self-describing idea real databases use (Postgres's `pg_catalog`, MySQL's `information_schema`) — worth pausing on, because it means `getTable("tinydb_tables")` is itself a valid, answerable call, using the exact same code path as `getTable("employee")`.

---

# 37. Follow-up: How Does the Planner Actually Choose an IndexScan?

§22 named Rule-Based Optimization's canonical example — "use the index if the `WHERE` clause matches an indexed column" — without yet showing the mechanism. With `Plan` (§23) as a real abstraction, the answer is now concrete: `BasicQueryPlanner.createPlan` (§24) needs one more decision point, made **before** defaulting to `TablePlan`:

```java
@Override
public Plan createPlan(SelectStatement statement, TransactionContext tx) {
    Plan plan = selectAccessPath(statement); // NEW: choose TablePlan OR IndexSelectPlan, per §38

    if (statement.where() != null) plan = new SelectPlan(plan, statement.where());
    boolean selectStar = statement.columns().size() == 1 && statement.columns().get(0).equals("*");
    if (!selectStar) plan = new ProjectPlan(plan, statement.columns());
    return plan;
}

private Plan selectAccessPath(SelectStatement statement) {
    if (statement.where() instanceof Comparison c
            && c.operator().equals("=")
            && catalog.hasIndex(statement.table(), c.column())) {
        return new IndexSelectPlan(statement.table(), c.column(), c.value(), catalog); // §38
    }
    return new TablePlan(statement.table(), storageEngine, catalog); // the RBO "default" rule
}
```

`catalog.hasIndex(table, column)` is the one new `Catalog` method this decision needs — a Cost-Based Optimizer would replace this single `if` with a comparison of `blocksAccessed()` across *every* viable `Plan`, but the RBO version above is a real, working decision, not a placeholder.

---

# 38. Sketch: An IndexSelectPlan

```java
public final class IndexSelectPlan implements Plan {

    private final String tableName;
    private final String indexedColumn;
    private final Object searchKey;
    private final Catalog catalog;

    public IndexSelectPlan(String tableName, String indexedColumn, Object searchKey, Catalog catalog) {
        this.tableName = tableName;
        this.indexedColumn = indexedColumn;
        this.searchKey = searchKey;
        this.catalog = catalog;
    }

    @Override
    public Scan open() {
        RowId rowId = catalog.lookupIndex(tableName, indexedColumn, searchKey); // O(log n), not O(n)
        return new SingleRowScan(tableName, rowId); // a Scan over exactly one row, or zero if not found
    }

    @Override
    public int blocksAccessed() { return 3; } // a B+Tree lookup touches a small, roughly-constant number of pages

    @Override
    public int recordsOutput() { return 1; } // an equality lookup on a unique index returns at most one row

    @Override
    public TableSchema schema() { return catalog.getTable(tableName); }
}
```

`blocksAccessed()` returning a small constant instead of `TablePlan`'s (§23) full-table-scan estimate is the entire point made numerically: this is exactly the number a Cost-Based Optimizer would compare `TablePlan.blocksAccessed()` against to justify preferring the index. Building `SingleRowScan` and the actual B+Tree-backed index this sketch calls into is out of scope for this guide — see the future enhancements in §46.

---

# 39. Full Worked Example: CREATE TABLE, INSERT, SELECT End to End

```text
$ CREATE TABLE employee (id BIGINT PRIMARY KEY, name VARCHAR, salary DECIMAL);
Query OK, 0 rows affected

$ INSERT INTO employee (id, name, salary) VALUES (1, 'Alice', 95000);
Query OK, 1 row affected

$ SELECT name FROM employee WHERE salary > 50000;
+-------+
| name  |
+-------+
| Alice |
+-------+
1 row in set
```

Tracing the third statement through every layer this guide built:

```text
"SELECT name FROM employee WHERE salary > 50000"
   -> SqlLexer.tokenize()                                   (§7)
   -> SqlParser.parseStatement() -> SelectStatement          (§10-11, §13)
   -> BasicQueryEngine.doQuery(sql)                          (§26)
        -> BasicPlanner.createQueryPlan(sql, tx)             (§25)
             -> BasicQueryPlanner.createPlan(statement, tx)  (§24, extended by §37)
                  -> selectAccessPath -> TablePlan (no index on "salary")
                  -> wrapped in SelectPlan(where: salary > 50000)
                  -> wrapped in ProjectPlan(columns: ["name"])
        -> plan.open() -> ProjectScan(SelectScan(TableScan))  (§29-30)
        -> while (scan.next()) { ... }                         (§26)
   -> QueryResult.ofRows([{name: "Alice"}])
```

Every mechanism this guide built — lexing, parsing, the AST, the planner, and the streaming scan pipeline — appears somewhere in this one trace.

---

# 40. Final Architecture

```text
                     SQL text
                         |
                    SqlLexer (§7)
                         |
                    SqlParser (§10-14)
                         |
                    SqlStatement (sealed AST)
                         |
            +------------+------------+
            v                         v
   Level 1: QueryEngine       Level 2: BasicPlanner (Facade)
   direct dispatch (§18)      -> QueryPlanner / UpdatePlanner (§24)
            |                         |
            v                         v
     Six Executors (§17)         Plan (TablePlan/SelectPlan/ProjectPlan, §23)
            |                    or IndexSelectPlan (§38)
            |                         |
            |                         v
            |                    Scan (TableScan/SelectScan/ProjectScan, §29-30)
            |                         |
            +------------+------------+
                         v
                  StorageEngine (§16)
                  Catalog / PersistentCatalog (§33-36)
                         |
                         v
        Companion Storage Engine guide's real, page-based implementation
```

---

# 41. Tracing a Query Through Every Layer

§39's trace already showed one `SELECT`; the same shape holds for a write — `INSERT INTO employee ... VALUES (...)` goes `SqlLexer -> SqlParser -> BasicPlanner.executeUpdate -> BasicUpdatePlanner.executeInsert -> TablePlan.open() -> Scan.insert()/setVal(...)` — the mirror image of the read path, sharing `TablePlan` and the same `Scan` contract rather than a separate write-only pipeline.

---

# 42. Design Patterns Used

| Pattern | Applied to | Where |
|---|---|---|
| **Command** | Each `SqlStatement` is a self-contained, immutable description of an action | §10, §12 |
| **Facade** | `BasicPlanner`/`BasicQueryEngine` hiding the parser, planner, and executor layers behind `doQuery`/`doUpdate` | §25-§26 |
| **Decorator** | `SelectPlan`/`ProjectPlan` and `SelectScan`/`ProjectScan` each wrap another `Plan`/`Scan` | §23, §30 |
| **Iterator** | `Scan` streams rows one at a time, exactly like `java.util.Iterator` | §28-§30 |
| **Strategy** | Choosing `TablePlan` vs `IndexSelectPlan` per query, without the caller knowing which was chosen | §37-§38 |
| **Visitor** (alternative architecture) | ANTLR's generated parse-tree traversal | §19 |

---

# 43. SOLID Principles Applied

| Principle | Where it holds |
|---|---|
| **S**ingle Responsibility | `SqlLexer` only tokenizes; `SqlParser` only parses; each executor only executes one statement type |
| **O**pen/Closed | Adding a seventh `SqlStatement` type is additive everywhere the sealed interface is switched over — the compiler enforces it (§18, §25) |
| **L**iskov Substitution | Any `Plan`/`Scan` implementation is fully interchangeable from its caller's point of view |
| **I**nterface Segregation | `QueryPlanner`/`UpdatePlanner`/`Catalog` are separate, narrow interfaces; §28 names the further split `Scan` itself deserves |
| **D**ependency Inversion | Every executor and planner depends on `Catalog`/`StorageEngine` interfaces, never a concrete implementation |

---

# 44. Common Mistakes

- **Mistake 1 — Not making `SqlStatement`/`Expression` sealed interfaces.** Loses compiler-enforced exhaustiveness at every dispatch point (§18, §14) — a new statement or expression type can silently fall through instead of failing to compile.
- **Mistake 2 — Confusing the lexer's job with the parser's.** The lexer should never make a decision that depends on grammar structure (e.g. "is this the start of a `WHERE` clause") — that's the parser's job, using tokens the lexer already produced.
- **Mistake 3 — Materializing a full result set inside `Plan`/`Scan` code.** Defeats the entire purpose of Part 5's streaming design — only the outermost caller (§26's `doQuery`) should ever build a `List<Row>`.
- **Mistake 4 — Letting `Scan` implementations hold onto stale state after `close()`.** A `Scan` that's been closed should never be usable again — not enforced by the interface in this guide, worth adding an explicit check in a production version.
- **Mistake 5 — A `Catalog` that isn't durable.** An in-memory-only catalog forgets every table on restart — §35's `PersistentCatalog` exists specifically to close this gap.
- **Mistake 6 — Choosing `IndexSelectPlan` (§38) without checking the operator is `=`.** A range predicate (`WHERE id > 100`) against an equality-only index needs a different index-scan strategy (a range scan, not a point lookup) — silently using a point-lookup index for a range query returns wrong results, not just slow ones.

---

# 45. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| `SqlLexer` (§7) | Every token type, including edge cases (`!=`, string literals with no content, numbers) | Unit tests feeding hand-written SQL strings, asserting the exact token sequence |
| `SqlParser` (§11-13) | Each statement type parses to the exact expected AST; malformed input throws | Unit tests comparing `parseStatement()`'s output to a hand-built `SqlStatement` record |
| `ExpressionEvaluator` (§14, §31) | `AND`/`OR` short-circuiting is correct; both `matches(Row, ...)` and `matchesScan(Scan, ...)` agree on the same input | Property-based: generate random rows and expressions, assert both entry points return the same boolean |
| `QueryEngine`/`BasicQueryEngine` (§18, §26) | End-to-end: `CREATE TABLE` → `INSERT` → `SELECT` returns the inserted row | Integration test running real SQL strings through the full pipeline |
| `BasicQueryPlanner` (§24, §37) | The correct `Plan` type is chosen for a given statement (`TablePlan` vs `IndexSelectPlan`) | Unit test asserting `createPlan(...)` returns an instance of the expected class |

---

# 46. Suggested Future Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| `JOIN` support | Combining rows from two tables | A `JoinPlan`/`NestedLoopJoinScan` pair, composed the same way `SelectPlan`/`SelectScan` are |
| `ORDER BY` | Sorted results | A `SortPlan`/`SortScan` — the one operator §32 already flags as unable to stream |
| `GROUP BY` / aggregate functions | `COUNT`, `SUM`, `AVG`, etc. | An `AggregatePlan` consuming a full (possibly sorted) input before producing output rows |
| Real B+Tree-backed indexes | Making §38's `IndexSelectPlan` genuinely O(log n), not a sketch | See the companion Storage Engine guide's indexing sections |
| A real Cost-Based Optimizer | Comparing multiple candidate `Plan`s by estimated cost, not one fixed rule | Extending §37's `selectAccessPath` to build several plans and pick the cheapest via `blocksAccessed()` |
| Prepared statements | Reusing a parsed `SqlStatement`/`Plan` across many executions with different literal values | Parameterizing `Comparison.value()` instead of baking a literal into the AST at parse time |

---

# 47. Progressive Interview Question Set

**Level 1 — Lexing and parsing**
1. Why does a lexer need no knowledge of grammar structure, only character-level rules?
2. Walk through, by hand, how `SELECT id FROM t WHERE x = 1` tokenizes and then parses into a `SelectStatement`.

**Level 2 — The AST as a boundary**
3. Why does making `SqlStatement` a sealed interface matter beyond style — what does the compiler actually enforce because of it?

**Level 3 — Execution**
4. Explain the seam between lexing/parsing and execution — name the one class where they meet, and explain why nothing on either side of it needs to know about the other.

**Level 4 — The planner**
5. What can a `Plan`-based design do that direct AST execution (§17) cannot?
6. Explain Rule-Based vs Cost-Based optimization with a concrete example from this guide.

**Level 5 — Streaming**
7. Why does composing `Scan`s use less peak memory than composing `List<Row>`s, concretely?
8. Name one query operator that fundamentally cannot stream, and explain why.

**Final challenge:** Extend this guide's planner to support `SELECT ... ORDER BY column LIMIT n` efficiently — without first materializing and sorting the entire input. What data structure would you use, and where does it sit in the `Plan`/`Scan` pipeline?

---

# 48. Final Takeaway

Every layer in this guide exists to answer one question at a progressively more sophisticated level: **given SQL text, what should actually happen?** The lexer and parser turn that question from "text" into "a typed fact" (the `SqlStatement` AST). The executors (Part 3) answer it directly and simply. The planner (Part 4-6) answers the identical question with room for optimization — a `Plan` an optimizer can inspect and compare, and a `Scan` pipeline that streams instead of materializing. Nothing in the second answer contradicts the first; §27 shows the exact, mechanical adaptation from one to the other, because both were built from the very beginning against the same `SqlStatement`/`Expression`/`Catalog`/`StorageEngine` vocabulary. That vocabulary — not any single class — is the real architecture this guide teaches.

