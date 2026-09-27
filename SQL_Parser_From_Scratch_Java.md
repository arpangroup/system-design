# SQL Parser From Scratch in Java

## Goal

Build a scalable SQL parser in Java 21 and evolve it into a complete SQL frontend:

```text
SQL
 ↓
Lexer
 ↓
Tokens
 ↓
Parser
 ↓
AST
 ↓
Semantic Analyzer
 ↓
Validated AST
 ↓
Logical Plan
 ↓
Optimizer
 ↓
Physical Plan
 ↓
Executor
```

This is a **new, separate document**. It does not modify the original compiler guide.

---

## 1. Project Structure

```text
sql-parser/
├── pom.xml
├── README.md
├── docs/
│   ├── grammar.md
│   ├── architecture.md
│   └── dialects.md
└── src/
    ├── main/java/com/sqlparser/
    │   ├── api/
    │   ├── source/
    │   ├── diagnostics/
    │   ├── lexer/
    │   ├── parser/
    │   ├── ast/
    │   ├── type/
    │   ├── semantic/
    │   ├── normalize/
    │   ├── plan/
    │   ├── optimizer/
    │   ├── physical/
    │   ├── executor/
    │   ├── dialect/
    │   ├── formatter/
    │   └── cli/
    └── test/java/com/sqlparser/
        ├── lexer/
        ├── parser/
        ├── semantic/
        ├── optimizer/
        └── formatter/
```

---

## 2. Build Incrementally

Implement in this order:

```text
1. Lexer
2. SELECT + FROM
3. Expressions + precedence
4. WHERE
5. JOIN
6. GROUP BY + HAVING
7. ORDER BY + LIMIT + OFFSET
8. Functions
9. INSERT / UPDATE / DELETE
10. CREATE TABLE
11. Subqueries
12. CTEs
13. UNION / INTERSECT / EXCEPT
14. CASE
15. Window functions
16. Semantic analysis
17. Catalog + type system
18. Logical planner
19. Optimizer
20. Dialects
21. Optional execution engine
```

Do not start by implementing every SQL feature.

---

## 3. Maven

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
           http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.sqlparser</groupId>
    <artifactId>sql-parser</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.11.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

# 4. Source Location

Every token and AST node should carry source information.

```java
public record SourceLocation(
        int offset,
        int line,
        int column
) {}
```

This enables:

```text
query.sql:4:18

SELECT *
FROM users
WHERE age >=
             ^

error: expected expression
```

---

# 5. Token Types

```java
public enum TokenType {

    SELECT, FROM, WHERE,

    INSERT, INTO, VALUES,
    UPDATE, SET, DELETE,

    CREATE, ALTER, DROP, TABLE,

    AS, DISTINCT,
    GROUP, BY, HAVING,
    ORDER, ASC, DESC,
    LIMIT, OFFSET,

    JOIN, INNER, LEFT, RIGHT,
    FULL, OUTER, CROSS, ON,

    UNION, ALL, INTERSECT, EXCEPT,

    WITH, RECURSIVE,

    AND, OR, NOT,
    IN, IS, NULL, BETWEEN, LIKE,
    EXISTS,

    CASE, WHEN, THEN, ELSE, END,

    INT, BIGINT, DECIMAL,
    VARCHAR, CHAR, BOOLEAN,
    DATE, TIMESTAMP,

    INTEGER_LITERAL,
    DECIMAL_LITERAL,
    STRING_LITERAL,
    BOOLEAN_LITERAL,

    IDENTIFIER,
    QUOTED_IDENTIFIER,

    PLUS, MINUS, STAR,
    SLASH, PERCENT,

    EQUAL,
    NOT_EQUAL,
    LESS,
    LESS_EQUAL,
    GREATER,
    GREATER_EQUAL,

    LEFT_PAREN,
    RIGHT_PAREN,
    COMMA,
    DOT,
    SEMICOLON,

    PARAMETER,

    EOF
}
```

---

# 6. Token

```java
public record Token(
        TokenType type,
        String lexeme,
        Object literal,
        SourceLocation location
) {}
```

For:

```sql
123
```

store:

```text
type    = INTEGER_LITERAL
lexeme  = "123"
literal = 123
```

For:

```sql
'hello'
```

store:

```text
type    = STRING_LITERAL
lexeme  = "'hello'"
literal = "hello"
```

Keeping the original lexeme and decoded value is useful for diagnostics and formatting.

---

# 7. Keyword Table

Keep keyword handling outside the lexer implementation.

```java
public interface KeywordTable {

    Optional<TokenType> lookup(String text);
}
```

Example:

```java
public final class StandardKeywordTable
        implements KeywordTable {

    private static final Map<String, TokenType> KEYWORDS =
            Map.ofEntries(
                Map.entry("SELECT", TokenType.SELECT),
                Map.entry("FROM", TokenType.FROM),
                Map.entry("WHERE", TokenType.WHERE),
                Map.entry("INSERT", TokenType.INSERT),
                Map.entry("UPDATE", TokenType.UPDATE),
                Map.entry("DELETE", TokenType.DELETE),
                Map.entry("GROUP", TokenType.GROUP),
                Map.entry("BY", TokenType.BY),
                Map.entry("ORDER", TokenType.ORDER),
                Map.entry("LIMIT", TokenType.LIMIT),
                Map.entry("OFFSET", TokenType.OFFSET),
                Map.entry("JOIN", TokenType.JOIN),
                Map.entry("LEFT", TokenType.LEFT),
                Map.entry("RIGHT", TokenType.RIGHT),
                Map.entry("INNER", TokenType.INNER),
                Map.entry("ON", TokenType.ON),
                Map.entry("AS", TokenType.AS),
                Map.entry("AND", TokenType.AND),
                Map.entry("OR", TokenType.OR),
                Map.entry("NOT", TokenType.NOT),
                Map.entry("NULL", TokenType.NULL)
            );

    @Override
    public Optional<TokenType> lookup(String text) {
        return Optional.ofNullable(
                KEYWORDS.get(text.toUpperCase())
        );
    }
}
```

Later this becomes dialect-specific.

---

# 8. Lexer

Input:

```sql
SELECT id, name
FROM users
WHERE age >= 18;
```

Output:

```text
SELECT
IDENTIFIER(id)
COMMA
IDENTIFIER(name)
FROM
IDENTIFIER(users)
WHERE
IDENTIFIER(age)
GREATER_EQUAL
INTEGER_LITERAL(18)
SEMICOLON
EOF
```

Interface:

```java
public interface SqlLexer {

    List<Token> tokenize();
}
```

Implementation skeleton:

```java
public final class SqlLexerImpl
        implements SqlLexer {

    private final String source;
    private final KeywordTable keywords;

    private int current;
    private int line = 1;
    private int column = 1;

    private final List<Token> tokens =
            new ArrayList<>();

    public SqlLexerImpl(
            String source,
            KeywordTable keywords
    ) {
        this.source = source;
        this.keywords = keywords;
    }

    @Override
    public List<Token> tokenize() {

        while (!isAtEnd()) {
            scanToken();
        }

        tokens.add(new Token(
                TokenType.EOF,
                "",
                null,
                new SourceLocation(
                        current,
                        line,
                        column
                )
        ));

        return List.copyOf(tokens);
    }

    private void scanToken() {
        char c = advance();

        switch (c) {
            case '(' -> add(TokenType.LEFT_PAREN);
            case ')' -> add(TokenType.RIGHT_PAREN);
            case ',' -> add(TokenType.COMMA);
            case '.' -> add(TokenType.DOT);
            case ';' -> add(TokenType.SEMICOLON);

            case '+' -> add(TokenType.PLUS);
            case '-' -> scanMinusOrComment();
            case '*' -> add(TokenType.STAR);
            case '/' -> scanSlashOrComment();
            case '%' -> add(TokenType.PERCENT);

            case '=' -> add(TokenType.EQUAL);

            case '<' -> add(
                    match('=') ?
                    TokenType.LESS_EQUAL :
                    TokenType.LESS
            );

            case '>' -> add(
                    match('=') ?
                    TokenType.GREATER_EQUAL :
                    TokenType.GREATER
            );

            case ''' -> string();

            case ' ', '\r', '\t', '\n' -> whitespace();

            default -> {
                if (isDigit(c)) {
                    number();
                } else if (isIdentifierStart(c)) {
                    identifier();
                } else {
                    throw new IllegalArgumentException(
                            "Unexpected character: " + c
                    );
                }
            }
        }
    }
}
```

Implement helper methods:

```text
advance()
peek()
match()
add()
string()
number()
identifier()
whitespace()
```

---

# 9. Comments

Support:

```sql
-- single-line comment
```

and:

```sql
/*
   multi-line
   comment
*/
```

Example:

```java
private void scanMinusOrComment() {

    if (match('-')) {

        while (!isAtEnd() && peek() != '\n') {
            advance();
        }

        return;
    }

    add(TokenType.MINUS);
}
```

---

# 10. Strings

Support:

```sql
'hello'
```

and:

```sql
'John''s house'
```

The lexer should produce:

```text
John's house
```

It should also produce a diagnostic for:

```sql
SELECT 'hello
```

instead of silently consuming the rest of the query.

---

# 11. Quoted Identifiers

Depending on dialect, support:

```sql
SELECT "order"
FROM "user";
```

Keep:

```text
STRING_LITERAL
QUOTED_IDENTIFIER
```

as separate token types.

---

# 12. Initial Grammar

```text
sql
    ::= statement* EOF

statement
    ::= selectStatement
     | insertStatement
     | updateStatement
     | deleteStatement
     | createTableStatement
```

Store the grammar separately in:

```text
docs/grammar.md
```

---

# 13. SELECT Grammar

```text
selectStatement
    ::= SELECT
        selectList
        fromClause?
        whereClause?
        groupByClause?
        havingClause?
        orderByClause?
        limitClause?
        offsetClause?
        ";"

selectList
    ::= "*"
     | selectItem ("," selectItem)*

selectItem
    ::= expression alias?

alias
    ::= AS IDENTIFIER
     | IDENTIFIER
```

---

# 14. FROM Grammar

```text
fromClause
    ::= FROM tableReference

tableReference
    ::= tableFactor joinClause*

tableFactor
    ::= tableName alias?
     | "(" selectStatement ")" alias?

tableName
    ::= qualifiedName

qualifiedName
    ::= IDENTIFIER ("." IDENTIFIER)*
```

This supports:

```sql
users
app.users
catalog.app.users
```

---

# 15. JOIN Grammar

```text
joinClause
    ::= joinType? JOIN tableFactor joinCondition?

joinType
    ::= INNER
     | LEFT OUTER?
     | RIGHT OUTER?
     | FULL OUTER?
     | CROSS

joinCondition
    ::= ON expression
```

Example:

```sql
SELECT u.id, o.id
FROM users u
LEFT JOIN orders o
    ON u.id = o.user_id;
```

---

# 16. WHERE / GROUP / HAVING

```text
whereClause
    ::= WHERE expression

groupByClause
    ::= GROUP BY expressionList

havingClause
    ::= HAVING expression
```

Example:

```sql
SELECT department_id, COUNT(*)
FROM employees
WHERE active = true
GROUP BY department_id
HAVING COUNT(*) > 10;
```

---

# 17. ORDER / LIMIT / OFFSET

```text
orderByClause
    ::= ORDER BY orderItem ("," orderItem)*

orderItem
    ::= expression (ASC | DESC)?

limitClause
    ::= LIMIT INTEGER_LITERAL

offsetClause
    ::= OFFSET INTEGER_LITERAL
```

Dialect-specific forms should remain inside dialect-specific parsing.

---

# 18. Expression Grammar

Use precedence:

```text
OR
AND
NOT
comparison
additive
multiplicative
unary
primary
```

Grammar:

```text
expression
    ::= logicalOr

logicalOr
    ::= logicalAnd ("OR" logicalAnd)*

logicalAnd
    ::= logicalNot ("AND" logicalNot)*

logicalNot
    ::= NOT logicalNot
     | comparison

comparison
    ::= additive
        (
            comparisonOperator additive
          | IS NOT? NULL
          | IN "(" inBody ")"
          | BETWEEN additive AND additive
          | LIKE additive
        )*

comparisonOperator
    ::= "=" | "<>" | "!="
     | "<" | "<=" | ">" | ">="

additive
    ::= multiplicative
        (("+" | "-") multiplicative)*

multiplicative
    ::= unary
        (("*" | "/" | "%") unary)*

unary
    ::= ("+" | "-") unary
     | primary

primary
    ::= literal
     | qualifiedColumn
     | functionCall
     | "(" expression ")"
     | caseExpression
     | subqueryExpression
     | parameter
```

---

# 19. Operator Precedence

Input:

```sql
a + b * c
```

must become:

```text
    +
   /   a   *
     /     b   c
```

Therefore:

```text
a + (b * c)
```

is represented in the AST.

---

# 20. AST

Use sealed interfaces.

```java
public sealed interface SqlNode
        permits SqlStatement,
                Expression,
                TableReference,
                OrderByItem {

    SourceLocation location();
}
```

```java
public sealed interface SqlStatement
        extends SqlNode
        permits SelectStatement,
                InsertStatement,
                UpdateStatement,
                DeleteStatement,
                CreateTableStatement {
}
```

---

# 21. SELECT AST

```java
public record SelectStatement(
        boolean distinct,
        List<SelectItem> selectItems,
        TableReference from,
        Expression where,
        List<Expression> groupBy,
        Expression having,
        List<OrderByItem> orderBy,
        Integer limit,
        Integer offset,
        SourceLocation location
) implements SqlStatement {
}
```

Prefer immutable lists.

---

# 22. SELECT Item

```java
public record SelectItem(
        Expression expression,
        String alias,
        SourceLocation location
) implements SqlNode {
}
```

Supports:

```sql
SELECT id
SELECT id AS user_id
SELECT COUNT(*) AS total
```

---

# 23. Table AST

```java
public sealed interface TableReference
        extends SqlNode
        permits TableName,
                Join,
                SubqueryTable {
}
```

```java
public record TableName(
        QualifiedName name,
        String alias,
        SourceLocation location
) implements TableReference {
}
```

```java
public record QualifiedName(
        List<String> parts
) {}
```

---

# 24. JOIN AST

```java
public record Join(
        TableReference left,
        JoinType type,
        TableReference right,
        Expression condition,
        SourceLocation location
) implements TableReference {
}
```

```java
public enum JoinType {
    INNER,
    LEFT,
    RIGHT,
    FULL,
    CROSS
}
```

---

# 25. Column AST

```java
public record ColumnReference(
        String qualifier,
        String name,
        SourceLocation location
) implements Expression {
}
```

For:

```sql
id
```

use:

```text
qualifier = null
name = id
```

For:

```sql
u.id
```

use:

```text
qualifier = u
name = id
```

For arbitrary qualification, use `QualifiedName`.

---

# 26. Literals

```java
public sealed interface LiteralExpression
        extends Expression
        permits IntegerLiteral,
                DecimalLiteral,
                StringLiteral,
                BooleanLiteral,
                NullLiteral {
}
```

Example:

```java
public record IntegerLiteral(
        long value,
        SourceLocation location
) implements LiteralExpression {
}
```

---

# 27. Binary Expression

```java
public record BinaryExpression(
        Expression left,
        BinaryOperator operator,
        Expression right,
        SourceLocation location
) implements Expression {
}
```

```java
public enum BinaryOperator {

    ADD, SUBTRACT,
    MULTIPLY, DIVIDE, MODULO,

    EQUAL, NOT_EQUAL,
    LESS, LESS_EQUAL,
    GREATER, GREATER_EQUAL,

    AND, OR,
    LIKE
}
```

---

# 28. Unary Expression

```java
public record UnaryExpression(
        UnaryOperator operator,
        Expression expression,
        SourceLocation location
) implements Expression {
}
```

```java
public enum UnaryOperator {
    PLUS,
    MINUS,
    NOT
}
```

---

# 29. Function Calls

```java
public record FunctionCallExpression(
        String name,
        List<Expression> arguments,
        boolean distinct,
        SourceLocation location
) implements Expression {
}
```

Examples:

```sql
COUNT(*)
COUNT(DISTINCT user_id)
SUM(amount)
AVG(salary)
LOWER(name)
COALESCE(name, 'Unknown')
```

Semantic analysis can later classify functions as:

```text
SCALAR
AGGREGATE
WINDOW
```

---

# 30. CASE

```sql
CASE
    WHEN age >= 18 THEN 'adult'
    ELSE 'minor'
END
```

AST:

```java
public record CaseExpression(
        List<WhenClause> clauses,
        Expression elseExpression,
        SourceLocation location
) implements Expression {
}
```

---

# 31. NULL / IN / BETWEEN

NULL:

```sql
deleted_at IS NULL
```

```java
public record NullCheckExpression(
        Expression expression,
        boolean negated,
        SourceLocation location
) implements Expression {
}
```

IN:

```sql
id IN (1, 2, 3)
```

```java
public record InExpression(
        Expression value,
        List<Expression> values,
        boolean negated,
        SourceLocation location
) implements Expression {
}
```

BETWEEN:

```sql
age BETWEEN 18 AND 60
```

```java
public record BetweenExpression(
        Expression value,
        Expression lower,
        Expression upper,
        boolean negated,
        SourceLocation location
) implements Expression {
}
```

---

# 32. Parser API

```java
public interface SqlParser {

    List<SqlStatement> parse();
}
```

---

# 33. TokenStream

```java
public interface TokenStream {

    Token peek();

    Token peek(int distance);

    Token previous();

    Token advance();

    boolean check(TokenType type);

    boolean match(TokenType... types);

    Token consume(
            TokenType type,
            String message
    );

    boolean isAtEnd();
}
```

Keep token navigation in one place.

---

# 34. Statement Dispatcher

```java
private SqlStatement statement() {

    return switch (tokens.peek().type()) {

        case SELECT -> selectStatement();
        case INSERT -> insertStatement();
        case UPDATE -> updateStatement();
        case DELETE -> deleteStatement();
        case CREATE -> createStatement();

        default -> throw error(
                tokens.peek(),
                "Expected SQL statement"
        );
    };
}
```

As the parser grows, split:

```text
SelectParser
ExpressionParser
DmlParser
DdlParser
```

---

# 35. SELECT Parser

```java
private SelectStatement selectStatement() {

    Token start = tokens.consume(
            TokenType.SELECT,
            "Expected SELECT"
    );

    boolean distinct =
            tokens.match(TokenType.DISTINCT);

    List<SelectItem> items =
            selectList();

    TableReference from = null;

    if (tokens.match(TokenType.FROM)) {
        from = fromClause();
    }

    Expression where = null;

    if (tokens.match(TokenType.WHERE)) {
        where = expression();
    }

    List<Expression> groupBy = List.of();

    if (tokens.match(TokenType.GROUP)) {

        tokens.consume(
                TokenType.BY,
                "Expected BY after GROUP"
        );

        groupBy = expressionList();
    }

    Expression having = null;

    if (tokens.match(TokenType.HAVING)) {
        having = expression();
    }

    List<OrderByItem> orderBy = List.of();

    if (tokens.match(TokenType.ORDER)) {

        tokens.consume(
                TokenType.BY,
                "Expected BY after ORDER"
        );

        orderBy = orderByList();
    }

    Integer limit = null;

    if (tokens.match(TokenType.LIMIT)) {
        limit = parseInteger();
    }

    Integer offset = null;

    if (tokens.match(TokenType.OFFSET)) {
        offset = parseInteger();
    }

    return new SelectStatement(
            distinct,
            items,
            from,
            where,
            groupBy,
            having,
            orderBy,
            limit,
            offset,
            start.location()
    );
}
```

---

# 36. Expression Parser

Start with recursive descent:

```java
private Expression expression() {
    return logicalOr();
}
```

```java
private Expression logicalOr() {

    Expression left = logicalAnd();

    while (tokens.match(TokenType.OR)) {

        Expression right = logicalAnd();

        left = new BinaryExpression(
                left,
                BinaryOperator.OR,
                right,
                left.location()
        );
    }

    return left;
}
```

AND:

```java
private Expression logicalAnd() {

    Expression left = logicalNot();

    while (tokens.match(TokenType.AND)) {

        Expression right = logicalNot();

        left = new BinaryExpression(
                left,
                BinaryOperator.AND,
                right,
                left.location()
        );
    }

    return left;
}
```

---

# 37. Comparison Parser

Handle:

```text
=
<>
!=
<
<=
>
>=
IS NULL
IS NOT NULL
IN
NOT IN
BETWEEN
NOT BETWEEN
LIKE
NOT LIKE
```

Example:

```java
private Expression comparison() {

    Expression left = additive();

    if (tokens.match(
            TokenType.EQUAL,
            TokenType.NOT_EQUAL,
            TokenType.LESS,
            TokenType.LESS_EQUAL,
            TokenType.GREATER,
            TokenType.GREATER_EQUAL
    )) {

        Token operator = tokens.previous();

        Expression right = additive();

        return new BinaryExpression(
                left,
                mapOperator(operator),
                right,
                operator.location()
        );
    }

    if (tokens.match(TokenType.IS)) {

        boolean negated =
                tokens.match(TokenType.NOT);

        tokens.consume(
                TokenType.NULL,
                "Expected NULL"
        );

        return new NullCheckExpression(
                left,
                negated,
                left.location()
        );
    }

    // IN / BETWEEN / LIKE...

    return left;
}
```

---

# 38. Pratt Parser Option

As expression syntax becomes large, a Pratt parser can replace the precedence methods.

Example binding powers:

```text
OR        10
AND       20
=         30
<         30
>         30
+         40
-         40
*         50
/         50
%         50
```

Recommended evolution:

```text
Initial implementation:
    Recursive descent

Advanced implementation:
    Pratt parser
```

---

# 39. INSERT

Grammar:

```text
insertStatement
    ::= INSERT INTO tableName
        columnList?
        VALUES valueRows

valueRows
    ::= "(" expressionList ")"
        ("," "(" expressionList ")")*
```

Example:

```sql
INSERT INTO users (id, name)
VALUES
    (1, 'A'),
    (2, 'B');
```

AST:

```java
public record InsertStatement(
        TableName table,
        List<String> columns,
        List<List<Expression>> values,
        SourceLocation location
) implements SqlStatement {
}
```

Later support:

```sql
INSERT INTO users
SELECT ...
```

---

# 40. UPDATE

```sql
UPDATE users
SET name = 'John',
    age = age + 1
WHERE id = 10;
```

```java
public record UpdateStatement(
        TableName table,
        List<Assignment> assignments,
        Expression where,
        SourceLocation location
) implements SqlStatement {
}
```

---

# 41. DELETE

```sql
DELETE FROM users
WHERE id = 10;
```

```java
public record DeleteStatement(
        TableName table,
        Expression where,
        SourceLocation location
) implements SqlStatement {
}
```

---

# 42. CREATE TABLE

Grammar:

```text
createTableStatement
    ::= CREATE TABLE tableName
        "(" columnDefinition ("," columnDefinition)* ")"

columnDefinition
    ::= IDENTIFIER dataType columnConstraint*

dataType
    ::= INT
     | BIGINT
     | VARCHAR "(" INTEGER_LITERAL ")"
     | DECIMAL "(" INTEGER_LITERAL "," INTEGER_LITERAL ")"
     | BOOLEAN
     | DATE
     | TIMESTAMP
```

Example:

```sql
CREATE TABLE users (
    id INT,
    name VARCHAR(100),
    active BOOLEAN
);
```

AST:

```java
public record CreateTableStatement(
        TableName table,
        List<ColumnDefinition> columns,
        SourceLocation location
) implements SqlStatement {
}
```

---

# 43. SQL Type System

```java
public sealed interface SqlType
        permits IntType,
                BigIntType,
                DecimalType,
                VarcharType,
                BooleanType,
                DateType,
                TimestampType,
                NullType {
}
```

Example:

```java
public record DecimalType(
        int precision,
        int scale
) implements SqlType {
}
```

Do not represent every SQL type as a string.

The type model is required for:

```text
type checking
function resolution
implicit casts
NULL handling
expression validation
planner decisions
```

---

# 44. Semantic Analysis

Parsing answers:

> Is this syntactically valid?

Semantic analysis answers:

> Does this SQL make sense against a schema?

Example:

```sql
SELECT unknown_column
FROM users;
```

can parse correctly but fail semantic validation.

---

# 45. Catalog

```java
public interface Catalog {

    Optional<TableMetadata> findTable(
            QualifiedName name
    );

    Optional<FunctionDefinition> findFunction(
            String name
    );
}
```

```java
public record TableMetadata(
        QualifiedName name,
        List<ColumnMetadata> columns
) {}
```

```java
public record ColumnMetadata(
        String name,
        SqlType type,
        boolean nullable
) {}
```

Possible implementations:

```text
InMemoryCatalog
JdbcCatalog
PostgresCatalog
MySqlCatalog
MockCatalog
```

---

# 46. Scope Resolution

For:

```sql
SELECT u.id, o.total
FROM users u
JOIN orders o
    ON u.id = o.user_id;
```

build:

```text
Query Scope

u -> users
o -> orders
```

Then resolve:

```text
u.id -> users.id
o.total -> orders.total
```

---

# 47. Ambiguous Columns

If both tables contain `id`:

```sql
SELECT id
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

should produce:

```text
error: column 'id' is ambiguous
```

while:

```sql
SELECT u.id
```

is valid.

This is semantic analysis, not parsing.

---

# 48. SQL NULL Semantics

SQL has three-valued logic:

```text
TRUE
FALSE
UNKNOWN
```

For example:

```sql
NULL = 10
```

produces:

```text
UNKNOWN
```

Model explicitly:

```java
public enum SqlBoolean {
    TRUE,
    FALSE,
    UNKNOWN
}
```

This becomes important for an execution engine.

---

# 49. Function Registry

Do not hard-code every function into the parser.

```java
public interface FunctionRegistry {

    Optional<FunctionDefinition> find(
            String name
    );
}
```

```java
public record FunctionDefinition(
        String name,
        List<SqlType> parameterTypes,
        SqlType returnType,
        FunctionKind kind
) {}
```

```java
public enum FunctionKind {
    SCALAR,
    AGGREGATE,
    WINDOW
}
```

Examples:

```text
COUNT
SUM
AVG
MIN
MAX
LOWER
UPPER
COALESCE
SUBSTRING
```

---

# 50. Subqueries

Support:

```sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
```

Represent:

```text
Select
  Filter
    InExpression
      Subquery
        Select
```

The parser should only build structure.

---

# 51. CTEs

Support:

```sql
WITH active_users AS (
    SELECT *
    FROM users
    WHERE active = true
)
SELECT *
FROM active_users;
```

AST:

```java
public record CommonTableExpression(
        String name,
        SelectStatement query,
        SourceLocation location
) implements SqlNode {
}
```

---

# 52. Set Operations

Support:

```sql
SELECT id FROM users
UNION
SELECT id FROM admins;
```

AST:

```java
public record SetOperation(
        SqlStatement left,
        SetOperator operator,
        boolean all,
        SqlStatement right,
        SourceLocation location
) implements SqlStatement {
}
```

```java
public enum SetOperator {
    UNION,
    INTERSECT,
    EXCEPT
}
```

---

# 53. Window Functions

Later support:

```sql
SELECT
    name,
    department_id,
    ROW_NUMBER() OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS rank
FROM employees;
```

AST:

```java
public record WindowExpression(
        Expression function,
        List<Expression> partitionBy,
        List<OrderByItem> orderBy,
        WindowFrame frame,
        SourceLocation location
) implements Expression {
}
```

---

# 54. SQL Dialects

SQL dialects differ:

```text
PostgreSQL
MySQL
SQLite
Oracle
SQL Server
```

Avoid scattering:

```java
if (postgres) ...
if (mysql) ...
if (oracle) ...
```

Use:

```java
public interface SqlDialect {

    KeywordTable keywords();

    IdentifierRules identifiers();

    GrammarFeatures features();
}
```

---

# 55. Dialect Features

```java
public record GrammarFeatures(
        boolean supportsLimit,
        boolean supportsOffset,
        boolean supportsReturning,
        boolean supportsRecursiveCte,
        boolean supportsWindowFunctions
) {}
```

Implement:

```text
StandardSqlDialect
PostgresDialect
MySqlDialect
SQLiteDialect
```

---

# 56. Logical Query Processing

SQL text is written approximately as:

```text
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

A planner can model the relational operations approximately as:

```text
FROM
  ↓
WHERE
  ↓
GROUP BY
  ↓
HAVING
  ↓
SELECT
  ↓
ORDER BY
  ↓
LIMIT
```

This is why a logical plan is useful.

---

# 57. Logical Plan

```java
public sealed interface LogicalPlan
        permits LogicalScan,
                LogicalFilter,
                LogicalProject,
                LogicalJoin,
                LogicalAggregate,
                LogicalSort,
                LogicalLimit {
}
```

```java
public record LogicalScan(
        TableMetadata table
) implements LogicalPlan {
}
```

```java
public record LogicalFilter(
        LogicalPlan input,
        Expression predicate
) implements LogicalPlan {
}
```

```java
public record LogicalProject(
        LogicalPlan input,
        List<Expression> expressions
) implements LogicalPlan {
}
```

---

# 58. Example Plan

SQL:

```sql
SELECT name
FROM users
WHERE age >= 18;
```

Logical plan:

```text
Project(name)
    |
Filter(age >= 18)
    |
Scan(users)
```

The parser creates the AST; the planner creates this representation.

---

# 59. JOIN Plan

SQL:

```sql
SELECT u.name, o.total
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

Logical plan:

```text
Project(u.name, o.total)
          |
        Join
       /     Scan(users) Scan(orders)

condition:
u.id = o.user_id
```

---

# 60. Optimizer

```java
public interface OptimizationRule {

    boolean matches(LogicalPlan plan);

    LogicalPlan apply(LogicalPlan plan);
}
```

```java
public final class PlanOptimizer {

    private final List<OptimizationRule> rules;

    public PlanOptimizer(
            List<OptimizationRule> rules
    ) {
        this.rules = List.copyOf(rules);
    }

    public LogicalPlan optimize(
            LogicalPlan plan
    ) {

        LogicalPlan current = plan;
        boolean changed;

        do {
            changed = false;

            for (OptimizationRule rule : rules) {

                if (rule.matches(current)) {

                    LogicalPlan next =
                            rule.apply(current);

                    if (next != current) {
                        current = next;
                        changed = true;
                    }
                }
            }

        } while (changed);

        return current;
    }
}
```

---

# 61. Predicate Pushdown

Input:

```text
Filter(u.age > 18)
        |
      Join
      /    users   orders
```

Potential optimized form:

```text
      Join
      /    Filter   orders
   |
 users

condition:
u.id = o.user_id
```

The optimizer must verify that moving the predicate preserves semantics.

---

# 62. Projection Pruning

For:

```sql
SELECT u.name
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

users may only need:

```text
id
name
```

and orders may only need:

```text
user_id
```

rather than every column.

---

# 63. Physical Plan

A logical `Join` does not specify implementation.

Possible physical operators:

```text
HashJoin
NestedLoopJoin
MergeJoin
```

Pipeline:

```text
AST
 ↓
Logical Plan
 ↓
Optimizer
 ↓
Physical Plan
 ↓
Executor
```

---

# 64. Execution Engine

Optional future phase:

```java
public interface PhysicalOperator {

    void open();

    Row next();

    void close();
}
```

Operators:

```text
TableScan
IndexScan
Filter
Project
HashJoin
NestedLoopJoin
Aggregate
Sort
Limit
```

---

# 65. Row Model

Start simple:

```java
public record Row(
        List<Object> values
) {}
```

Later consider:

```text
columnar execution
vectorized execution
primitive arrays
off-heap memory
```

---

# 66. Storage Boundary

The parser should know nothing about:

```text
pages
B+ trees
buffer pools
WAL
disk offsets
MVCC
```

Future architecture:

```text
SQL Parser
    ↓
Planner
    ↓
Executor
    ↓
Storage Engine
        |
        +-- Buffer Pool
        +-- Heap Files
        +-- B+ Tree
        +-- WAL
        +-- Transactions
```

---

# 67. Formatter

```java
public interface SqlFormatter {

    String format(SqlNode node);
}
```

Input:

```sql
select id,name from users where age>18;
```

Output:

```sql
SELECT
    id,
    name
FROM users
WHERE age > 18;
```

---

# 68. AST Printer

Add:

```bash
sqltool ast query.sql
```

Example:

```text
SelectStatement
  SelectItems
    Column(id)
    Column(name)

  From
    Table(users)

  Where
    Binary(GREATER)
      Column(age)
      Integer(18)
```

Also add:

```bash
sqltool tokens query.sql
sqltool format query.sql
sqltool plan query.sql
```

---

# 69. EXPLAIN

Support:

```sql
EXPLAIN
SELECT *
FROM users
WHERE age >= 18;
```

Output:

```text
Project(*)
  Filter(age >= 18)
    Scan(users)
```

Later:

```sql
EXPLAIN ANALYZE ...
```

can show runtime information.

---

# 70. Diagnostics

```java
public record Diagnostic(
        Severity severity,
        String code,
        String message,
        SourceLocation location
) {}
```

Example codes:

```text
SQL001 unexpected token
SQL002 expected expression
SQL003 invalid identifier
SQL004 invalid JOIN
SQL005 unknown table
SQL006 unknown column
SQL007 ambiguous column
SQL008 incompatible types
```

---

# 71. Error Recovery

Useful synchronization tokens:

```text
FROM
WHERE
GROUP
HAVING
ORDER
LIMIT
OFFSET
JOIN
ON
)
;
EOF
```

For IDE/editor usage, collect diagnostics and continue parsing where possible.

For command-line execution, fail-fast mode can remain available.

---

# 72. Prepared Statements

Support:

```sql
SELECT *
FROM users
WHERE id = ?;
```

AST:

```java
public record ParameterExpression(
        int index,
        SourceLocation location
) implements Expression {
}
```

Semantic analysis can infer:

```text
parameter 1 -> expected INT
```

from the expression and catalog.

---

# 73. Query Fingerprinting

Normalize:

```sql
SELECT *
FROM users
WHERE id = 10;
```

and:

```sql
SELECT *
FROM users
WHERE id = 20;
```

to:

```text
SELECT * FROM users WHERE id = ?
```

Useful for:

```text
query statistics
plan caching
query grouping
monitoring
```

---

# 74. Testing Strategy

Test independently:

```text
Lexer
 ↓
Parser
 ↓
AST
 ↓
Semantic Analyzer
 ↓
Planner
 ↓
Optimizer
 ↓
Formatter
 ↓
End-to-End
```

Lexer tests:

```text
keywords
identifiers
numbers
strings
escaped strings
operators
comments
quoted identifiers
```

Parser tests:

```text
SELECT
WHERE
JOIN
GROUP BY
HAVING
ORDER BY
LIMIT
subqueries
CTEs
CASE
functions
```

Semantic tests:

```text
unknown table
unknown column
ambiguous column
invalid function
invalid types
invalid GROUP BY
```

---

# 75. Golden AST Tests

For:

```sql
SELECT id, name
FROM users
WHERE age >= 18;
```

expected AST:

```text
Select
  items:
    Column(id)
    Column(name)

  from:
    Table(users)

  where:
    Binary(GREATER_EQUAL)
      Column(age)
      Integer(18)
```

Compare generated AST with the expected representation.

---

# 76. Round-Trip Testing

Use:

```text
SQL
 ↓
parse
 ↓
AST
 ↓
format
 ↓
SQL
 ↓
parse
 ↓
AST
```

The two ASTs should be structurally or semantically equivalent.

Example:

```sql
select id,name from users where age>18;
```

formats as:

```sql
SELECT
    id,
    name
FROM users
WHERE age > 18;
```

and should parse back to an equivalent AST.

---

# 77. Fuzz Testing

Generate random:

```text
identifiers
parentheses
operators
nested expressions
JOINs
subqueries
```

Required invariant:

```text
valid SQL
    -> AST

invalid SQL
    -> Diagnostic
```

The parser should not produce:

```text
NullPointerException
unexpected JVM crash
unbounded recursion
StackOverflowError
```

For deeply nested input, configurable parser-depth limits are useful.

---

# 78. Thread Safety

Avoid:

```java
static List<Token> tokens;
```

Prefer independent parser instances:

```java
SqlParser parser =
        parserFactory.create(configuration);

ParseResult result =
        parser.parse(source);
```

This allows concurrent parsing.

---

# 79. Parser Configuration

```java
public record ParserConfiguration(
        SqlDialect dialect,
        boolean collectDiagnostics,
        boolean allowMultipleStatements,
        boolean allowExtensions
) {}
```

This is preferable to passing many boolean arguments.

---

# 80. Top-Level Compilation API

A useful API is:

```java
public final class SqlCompiler {

    public QueryCompilationResult compile(
            String sql
    ) {

        List<Token> tokens =
                lexer.tokenize(sql);

        List<SqlStatement> ast =
                parser.parse(tokens);

        ValidatedSql validated =
                semanticAnalyzer.analyze(ast);

        LogicalPlan logical =
                planner.plan(validated);

        LogicalPlan optimized =
                optimizer.optimize(logical);

        PhysicalPlan physical =
                physicalPlanner.plan(optimized);

        return new QueryCompilationResult(
                ast,
                validated,
                logical,
                optimized,
                physical
        );
    }
}
```

Conceptually:

```text
SQL
 ↓
Tokens
 ↓
AST
 ↓
Validated AST
 ↓
Logical Plan
 ↓
Optimized Logical Plan
 ↓
Physical Plan
```

---

# 81. SOLID Design

## Single Responsibility

```text
Lexer
    characters -> tokens

Parser
    tokens -> AST

SemanticAnalyzer
    AST -> validated AST

Planner
    AST -> logical plan

Optimizer
    logical plan -> optimized plan

Executor
    physical plan -> rows
```

## Open/Closed

Adding:

```text
PostgresDialect
```

should not require rewriting the generic parser.

Adding:

```text
HashJoinRule
```

should not require rewriting the optimizer.

## Dependency Inversion

Use interfaces for:

```text
Catalog
FunctionRegistry
SqlDialect
OptimizationRule
PhysicalOperator
```

---

# 82. Dependency Injection

Use constructor injection:

```java
public final class SqlCompiler {

    private final SqlLexer lexer;
    private final SqlParser parser;
    private final SemanticAnalyzer semanticAnalyzer;
    private final QueryPlanner planner;
    private final PlanOptimizer optimizer;

    public SqlCompiler(
            SqlLexer lexer,
            SqlParser parser,
            SemanticAnalyzer semanticAnalyzer,
            QueryPlanner planner,
            PlanOptimizer optimizer
    ) {
        this.lexer = lexer;
        this.parser = parser;
        this.semanticAnalyzer = semanticAnalyzer;
        this.planner = planner;
        this.optimizer = optimizer;
    }
}
```

The parser must not create database connections, catalogs, optimizers, or executors itself.

---

# 83. Git Milestones

```text
01-initialize-project
02-source-and-location
03-token-model
04-keyword-table
05-sql-lexer
06-lexer-tests
07-sql-ast
08-select-parser
09-expression-parser
10-where-order-limit
11-join-parser
12-group-having
13-functions
14-insert-update-delete
15-create-table
16-subqueries
17-cte
18-set-operations
19-semantic-analysis
20-catalog
21-type-system
22-null-semantics
23-logical-planner
24-optimizer
25-formatter
26-dialect-support
27-prepared-parameters
28-explain
29-executor
```

Keep the project buildable and tested after every milestone.

---

# 84. Learning / Interview Follow-Ups

## Lexer

1. Why use a lexer?
2. Should SQL keywords be case-sensitive?
3. How do quoted identifiers work?
4. How are SQL strings escaped?
5. How do dialects affect lexical analysis?

## Parser

6. Why recursive descent?
7. What is operator precedence?
8. What is a Pratt parser?
9. How do you parse nested queries?
10. How do you recover from syntax errors?

## AST

11. Why not execute directly from tokens?
12. Why immutable AST nodes?
13. How would you represent qualified names?
14. How would you model dialect-specific syntax?

## Semantic Analysis

15. How do you resolve columns?
16. How do you detect ambiguous columns?
17. How do aliases create scopes?
18. How do correlated subqueries work?
19. How does NULL affect type checking?

## Query Planning

20. Why separate logical and physical plans?
21. What is predicate pushdown?
22. What is projection pruning?
23. How do you choose a join algorithm?
24. How do table statistics affect optimization?

## Database Engine

25. How does a hash join work?
26. How does a merge join work?
27. What is a Volcano iterator?
28. What is vectorized execution?
29. How would you implement indexes?
30. How would you implement MVCC?
31. How would you implement WAL and recovery?

---

# 85. Advanced Roadmap: Parser to Database

Once the parser is stable:

```text
SQL Parser
     ↓
Semantic Analyzer
     ↓
Logical Planner
     ↓
Optimizer
     ↓
Physical Planner
     ↓
Executor
     ↓
Storage Engine
```

Then add:

```text
Page Storage
Buffer Pool
Heap Files
B+ Tree Index
Hash Index
Transactions
Lock Manager
MVCC
WAL
Recovery
Checkpointing
Statistics
Cost-Based Optimizer
```

The parser architecture can remain independent from these components.

---

# 86. Final Architecture

```text
                         +----------------+
                         | CLI / IDE / API|
                         +-------+--------+
                                 |
                                 v
                         +---------------+
                         | Parser API    |
                         +-------+-------+
                                 |
                         +-------v-------+
                         | SQL Dialect   |
                         +-------+-------+
                                 |
                                 v
                         +---------------+
                         | Lexer         |
                         +-------+-------+
                                 |
                                 v
                              Tokens
                                 |
                                 v
                         +---------------+
                         | Parser        |
                         +-------+-------+
                                 |
                                 v
                               AST
                                 |
                                 v
                      +--------------------+
                      | Semantic Analyzer  |
                      +---------+----------+
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
                 Catalog                Type System
                    |                       |
                    +-----------+-----------+
                                |
                                v
                         Validated AST
                                |
                                v
                         Normalizer
                                |
                                v
                         Logical Plan
                                |
                                v
                          Optimizer
                                |
                                v
                         Physical Plan
                                |
                                v
                           Executor
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
               Storage Engine         External DB
```

---

# 87. Final Package Architecture

```text
com.sqlparser

├── api
│   ├── SqlParser.java
│   ├── ParseResult.java
│   └── ParserConfiguration.java
│
├── source
│   ├── SqlSource.java
│   └── SourceLocation.java
│
├── diagnostics
│   ├── Diagnostic.java
│   └── DiagnosticReporter.java
│
├── lexer
│   ├── SqlLexer.java
│   ├── Token.java
│   ├── TokenType.java
│   └── KeywordTable.java
│
├── parser
│   ├── SqlParserImpl.java
│   ├── SelectParser.java
│   ├── ExpressionParser.java
│   ├── DmlParser.java
│   ├── DdlParser.java
│   └── TokenStream.java
│
├── ast
│   ├── SqlNode.java
│   ├── SqlStatement.java
│   ├── SelectStatement.java
│   ├── InsertStatement.java
│   ├── UpdateStatement.java
│   ├── DeleteStatement.java
│   ├── CreateTableStatement.java
│   ├── Expression.java
│   ├── BinaryExpression.java
│   ├── FunctionCallExpression.java
│   ├── SubqueryExpression.java
│   ├── Join.java
│   └── ...
│
├── type
│   ├── SqlType.java
│   ├── IntType.java
│   ├── DecimalType.java
│   ├── VarcharType.java
│   └── ...
│
├── semantic
│   ├── SemanticAnalyzer.java
│   ├── Scope.java
│   ├── Catalog.java
│   ├── TableMetadata.java
│   ├── ColumnMetadata.java
│   └── FunctionRegistry.java
│
├── normalize
│   ├── SqlAstTransformer.java
│   └── QueryNormalizer.java
│
├── plan
│   ├── LogicalPlan.java
│   ├── LogicalScan.java
│   ├── LogicalFilter.java
│   ├── LogicalProject.java
│   ├── LogicalJoin.java
│   ├── LogicalAggregate.java
│   └── LogicalSort.java
│
├── optimizer
│   ├── OptimizationRule.java
│   ├── PlanOptimizer.java
│   ├── PredicatePushdown.java
│   └── ProjectionPruning.java
│
├── physical
│   ├── PhysicalPlan.java
│   ├── PhysicalScan.java
│   ├── HashJoin.java
│   ├── NestedLoopJoin.java
│   └── SortOperator.java
│
├── executor
│   ├── Executor.java
│   ├── PhysicalOperator.java
│   └── Row.java
│
├── dialect
│   ├── SqlDialect.java
│   ├── StandardSqlDialect.java
│   ├── PostgresDialect.java
│   └── MySqlDialect.java
│
├── formatter
│   └── SqlFormatter.java
│
└── cli
    └── SqlCli.java
```

---

# 88. Core Design Rule

Do **not** build:

```text
SQL
  ↓
Parser
  ↓
JDBC.executeQuery()
```

Build:

```text
SQL
  ↓
Lexer
  ↓
Parser
  ↓
AST
  ↓
Semantic Analysis
  ↓
Logical Plan
  ↓
Optimizer
  ↓
Physical Plan
  ↓
Executor
```

This allows the same SQL frontend to support:

```text
CLI
IDE
Formatter
Linter
Query Analyzer
Query Optimizer
In-memory Database
JDBC Backend
PostgreSQL-compatible Engine
```

The architecture is essentially a compiler pipeline whose intermediate representation is a relational query plan instead of machine or VM instructions.
