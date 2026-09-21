# Build Your Own Compiler From Scratch in Java

A step-by-step, implementation-oriented guide for building a small but extensible programming language and compiler entirely in Java.

The compiler developed in this guide will have:

- Its own language syntax.
- A lexer/tokenizer.
- A recursive-descent parser.
- An Abstract Syntax Tree (AST).
- Semantic analysis.
- A symbol table.
- Type checking.
- A compiler intermediate representation (IR).
- Constant folding and other optimizations.
- A bytecode/VM backend.
- A command-line compiler.
- Error reporting with source locations.
- Tests for every compiler phase.
- A scalable package/module structure.
- A design that can later target JVM bytecode, native code, WebAssembly, or another backend.

The examples use Java 21-style code and emphasize SOLID principles, separation of concerns, extensibility, and interview/system-design-style follow-up questions.

---

# 1. Compiler Problem Statement

## 1.1 System-design-style question

> Design and implement a programming language and compiler from scratch in Java.
>
> The language should have its own syntax and support variables, primitive types, arithmetic expressions, conditions, loops, functions, return values, and a standard library.
>
> The compiler should transform source code into executable instructions.
>
> The architecture must be modular, testable, scalable, and extensible so that new language features, types, optimization passes, and target platforms can be added without rewriting existing components.

## 1.2 Initial language

We will call the language:

```text
Nova
```

Example:

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}

let x: int = 10;
let y: int = 20;

if (x < y) {
    print(add(x, y));
}
```

The implementation will evolve in stages rather than attempting the entire compiler at once.

---

# 2. What We Are Building

A traditional compiler pipeline looks like:

```text
Source Code
    |
    v
+----------------+
| Character Input|
+----------------+
    |
    v
+----------------+
| Lexer          |
| Source -> Token|
+----------------+
    |
    v
+----------------+
| Parser         |
| Token -> AST   |
+----------------+
    |
    v
+----------------+
| Semantic        |
| Analysis       |
+----------------+
    |
    v
+----------------+
| Typed AST      |
+----------------+
    |
    v
+----------------+
| IR Generation  |
+----------------+
    |
    v
+----------------+
| Optimization   |
+----------------+
    |
    v
+----------------+
| Backend        |
+----------------+
    |
    v
+----------------+
| Bytecode       |
+----------------+
    |
    v
+----------------+
| Nova VM        |
+----------------+
    |
    v
Program Output
```

Later:

```text
                       +--> Nova VM
                       |
Nova IR -> Backend ----+--> JVM
                       |
                       +--> Native
                       |
                       +--> WebAssembly
```

The critical architectural decision is to keep the frontend independent from the backend.

---

# 3. Why Compiler Design Should Be Layered

Do not write:

```java
Compiler.compile(source);
```

with thousands of lines in one class.

Instead:

```text
frontend/
    lexer/
    parser/
    ast/
    semantic/

ir/
optimizer/
backend/
runtime/
cli/
diagnostics/
```

Each phase should have a well-defined responsibility.

For example:

```java
Lexer
    -> tokens

Parser
    -> AST

SemanticAnalyzer
    -> validated/typed AST

IrGenerator
    -> IR

Optimizer
    -> optimized IR

Backend
    -> executable representation
```

This allows us to replace:

```text
NovaVMBackend
```

with:

```text
JvmBackend
```

without changing the lexer or parser.

---

# 4. Project Structure

Create a Maven project:

```text
nova-compiler/
├── pom.xml
├── README.md
├── docs/
│   ├── language.md
│   ├── grammar.md
│   ├── architecture.md
│   └── bytecode.md
│
├── src/
│   ├── main/
│   │   └── java/
│   │       └── com/
│   │           └── nova/
│   │               └── compiler/
│   │
│   │                   ├── Main.java
│   │
│   │                   ├── lexer/
│   │                   │   ├── Lexer.java
│   │                   │   ├── Token.java
│   │                   │   ├── TokenType.java
│   │                   │   └── KeywordTable.java
│   │
│   │                   ├── parser/
│   │                   │   ├── Parser.java
│   │                   │   └── ParseException.java
│   │
│   │                   ├── ast/
│   │                   │   ├── AstNode.java
│   │                   │   ├── Program.java
│   │                   │   ├── Statement.java
│   │                   │   ├── Expression.java
│   │                   │   ├── BinaryExpression.java
│   │                   │   ├── LiteralExpression.java
│   │                   │   ├── VariableExpression.java
│   │                   │   ├── VariableDeclaration.java
│   │                   │   ├── IfStatement.java
│   │                   │   ├── WhileStatement.java
│   │                   │   ├── FunctionDeclaration.java
│   │                   │   ├── ReturnStatement.java
│   │                   │   └── FunctionCallExpression.java
│   │
│   │                   ├── type/
│   │                   │   ├── Type.java
│   │                   │   ├── PrimitiveType.java
│   │                   │   ├── FunctionType.java
│   │                   │   └── TypeChecker.java
│   │
│   │                   ├── semantic/
│   │                   │   ├── SemanticAnalyzer.java
│   │                   │   ├── Symbol.java
│   │                   │   ├── SymbolTable.java
│   │                   │   └── Scope.java
│   │
│   │                   ├── ir/
│   │                   │   ├── IrInstruction.java
│   │                   │   ├── IrFunction.java
│   │                   │   ├── IrProgram.java
│   │                   │   ├── IrBuilder.java
│   │                   │   └── Opcode.java
│   │
│   │                   ├── optimizer/
│   │                   │   ├── OptimizationPass.java
│   │                   │   ├── ConstantFoldingPass.java
│   │                   │   ├── DeadCodeEliminationPass.java
│   │                   │   └── Optimizer.java
│   │
│   │                   ├── backend/
│   │                   │   ├── Backend.java
│   │                   │   ├── bytecode/
│   │                   │   └── vm/
│   │
│   │                   ├── runtime/
│   │                   │   ├── VirtualMachine.java
│   │                   │   ├── StackFrame.java
│   │                   │   └── RuntimeValue.java
│   │
│   │                   └── diagnostics/
│   │                       ├── SourceLocation.java
│   │                       ├── Diagnostic.java
│   │                       └── DiagnosticReporter.java
│   │
│   └── test/
│       └── java/
│           └── com/
│               └── nova/
│                   └── compiler/
│                       ├── lexer/
│                       ├── parser/
│                       ├── semantic/
│                       ├── ir/
│                       ├── optimizer/
│                       └── runtime/
```

---

# 5. Maven Configuration

`pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
           http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.nova</groupId>
    <artifactId>nova-compiler</artifactId>
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

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

Run:

```bash
mvn test
```

---

# 6. Design the Nova Language

Before implementing the compiler, define the language.

A compiler should not invent syntax accidentally while implementation is underway.

Start with:

## 6.1 Primitive types

```text
int
bool
string
void
```

Later:

```text
float
char
array<T>
map<K,V>
struct
enum
```

## 6.2 Variables

```nova
let age: int = 32;
let active: bool = true;
let name: string = "Arpan";
```

## 6.3 Assignment

```nova
age = 33;
```

## 6.4 Arithmetic

```nova
let total = 10 + 20 * 3;
```

## 6.5 Comparison

```nova
age > 18
age >= 18
age == 18
age != 18
```

## 6.6 Boolean operators

```nova
active && verified
active || verified
!active
```

## 6.7 If

```nova
if (age >= 18) {
    print("adult");
} else {
    print("minor");
}
```

## 6.8 While

```nova
let i: int = 0;

while (i < 10) {
    print(i);
    i = i + 1;
}
```

## 6.9 Functions

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}
```

## 6.10 Function calls

```nova
let result = add(10, 20);
```

---

# 7. Language Grammar

Use EBNF-like grammar.

The grammar is the contract between the parser and the language.

```text
program
    ::= declaration* EOF

declaration
    ::= functionDeclaration
     | statement

functionDeclaration
    ::= "fn" IDENTIFIER "(" parameters? ")" returnType? block

parameters
    ::= parameter ("," parameter)*

parameter
    ::= IDENTIFIER ":" type

returnType
    ::= "->" type

statement
    ::= variableDeclaration
     | assignment
     | ifStatement
     | whileStatement
     | returnStatement
     | expressionStatement
     | block

variableDeclaration
    ::= "let" IDENTIFIER (":" type)? "=" expression ";"

assignment
    ::= IDENTIFIER "=" expression ";"

ifStatement
    ::= "if" "(" expression ")" block ("else" block)?

whileStatement
    ::= "while" "(" expression ")" block

returnStatement
    ::= "return" expression? ";"

expressionStatement
    ::= expression ";"

block
    ::= "{" statement* "}"

expression
    ::= logicalOr

logicalOr
    ::= logicalAnd ("||" logicalAnd)*

logicalAnd
    ::= equality ("&&" equality)*

equality
    ::= comparison (("==" | "!=") comparison)*

comparison
    ::= term ((">" | ">=" | "<" | "<=") term)*

term
    ::= factor (("+" | "-") factor)*

factor
    ::= unary (("*" | "/" | "%") unary)*

unary
    ::= ("!" | "-") unary
     | primary

primary
    ::= INTEGER
     | STRING
     | "true"
     | "false"
     | IDENTIFIER
     | functionCall
     | "(" expression ")"

functionCall
    ::= IDENTIFIER "(" arguments? ")"

arguments
    ::= expression ("," expression)*

type
    ::= "int"
     | "bool"
     | "string"
     | "void"
```

---

# 8. Operator Precedence

The grammar intentionally separates operators into levels.

```text
lowest precedence
        ||
        &&
        == !=
        > >= < <=
        + -
        * / %
        ! -
        function calls / primary
highest precedence
```

Therefore:

```nova
10 + 20 * 3
```

becomes:

```text
10 + (20 * 3)
```

not:

```text
(10 + 20) * 3
```

This is one of the most important parser concepts.

---

# 9. Compiler Phase 1 — Character Input

The compiler first receives:

```java
String source
```

A better production design uses a source abstraction:

```java
public interface Source {
    char charAt(int index);

    int length();

    String text();
}
```

Implementation:

```java
public final class StringSource implements Source {

    private final String text;

    public StringSource(String text) {
        this.text = text;
    }

    @Override
    public char charAt(int index) {
        return text.charAt(index);
    }

    @Override
    public int length() {
        return text.length();
    }

    @Override
    public String text() {
        return text;
    }
}
```

Later this can support:

```text
StringSource
FileSource
MemoryMappedSource
IDE incremental source
```

---

# 10. Source Locations

Every token and AST node should eventually know where it came from.

```java
public record SourceLocation(
        int offset,
        int line,
        int column
) {}
```

Why?

Because this:

```text
Type error
```

is poor.

Instead:

```text
example.nova:7:14:
error: cannot add int and string
```

Source locations become extremely important once the compiler becomes usable.

---

# 11. Compiler Phase 2 — Token Model

Create:

```java
public enum TokenType {

    // Keywords
    FN,
    LET,
    IF,
    ELSE,
    WHILE,
    RETURN,
    TRUE,
    FALSE,

    // Types
    INT,
    BOOL,
    STRING,
    VOID,

    // Literals
    INTEGER_LITERAL,
    STRING_LITERAL,

    // Identifier
    IDENTIFIER,

    // Operators
    PLUS,
    MINUS,
    STAR,
    SLASH,
    PERCENT,

    EQUAL,
    EQUAL_EQUAL,
    BANG,
    BANG_EQUAL,

    LESS,
    LESS_EQUAL,
    GREATER,
    GREATER_EQUAL,

    AND_AND,
    OR_OR,

    // Symbols
    LEFT_PAREN,
    RIGHT_PAREN,
    LEFT_BRACE,
    RIGHT_BRACE,
    COMMA,
    COLON,
    SEMICOLON,
    ARROW,

    EOF
}
```

Token:

```java
public record Token(
        TokenType type,
        String lexeme,
        Object literal,
        SourceLocation location
) {}
```

This design separates:

```text
what token is it?
what text created it?
what value does it represent?
where did it occur?
```

---

# 12. Compiler Phase 3 — Lexer

The lexer converts:

```text
let x: int = 10 + 20;
```

into:

```text
LET
IDENTIFIER(x)
COLON
INT
EQUAL
INTEGER_LITERAL(10)
PLUS
INTEGER_LITERAL(20)
SEMICOLON
EOF
```

---

# 13. Lexer Architecture

Create:

```java
public interface Lexer {
    List<Token> tokenize();
}
```

Implementation:

```java
public final class NovaLexer implements Lexer {

    private final String source;

    private int start;
    private int current;
    private int line = 1;
    private int column = 1;

    private final List<Token> tokens = new ArrayList<>();

    public NovaLexer(String source) {
        this.source = source;
    }

    @Override
    public List<Token> tokenize() {
        while (!isAtEnd()) {
            start = current;
            scanToken();
        }

        tokens.add(new Token(
                TokenType.EOF,
                "",
                null,
                new SourceLocation(current, line, column)
        ));

        return List.copyOf(tokens);
    }

    private void scanToken() {
        char c = advance();

        switch (c) {
            case '(' -> add(TokenType.LEFT_PAREN);
            case ')' -> add(TokenType.RIGHT_PAREN);
            case '{' -> add(TokenType.LEFT_BRACE);
            case '}' -> add(TokenType.RIGHT_BRACE);
            case ',' -> add(TokenType.COMMA);
            case ':' -> add(TokenType.COLON);
            case ';' -> add(TokenType.SEMICOLON);

            case '+' -> add(TokenType.PLUS);
            case '-' -> {
                if (match('>')) {
                    add(TokenType.ARROW);
                } else {
                    add(TokenType.MINUS);
                }
            }

            case '*' -> add(TokenType.STAR);
            case '/' -> scanSlashOrComment();
            case '%' -> add(TokenType.PERCENT);

            case '=' -> add(match('=') ?
                    TokenType.EQUAL_EQUAL :
                    TokenType.EQUAL);

            case '!' -> add(match('=') ?
                    TokenType.BANG_EQUAL :
                    TokenType.BANG);

            case '<' -> add(match('=') ?
                    TokenType.LESS_EQUAL :
                    TokenType.LESS);

            case '>' -> add(match('=') ?
                    TokenType.GREATER_EQUAL :
                    TokenType.GREATER);

            case '&' -> {
                if (match('&')) {
                    add(TokenType.AND_AND);
                } else {
                    throw error("Expected '&' after '&'");
                }
            }

            case '|' -> {
                if (match('|')) {
                    add(TokenType.OR_OR);
                } else {
                    throw error("Expected '|' after '|'");
                }
            }

            case ' ', '\r', '\t' -> {}

            case '\n' -> {
                line++;
                column = 1;
            }

            case '"' -> string();

            default -> {
                if (isDigit(c)) {
                    number();
                } else if (isAlpha(c)) {
                    identifier();
                } else {
                    throw error("Unexpected character: " + c);
                }
            }
        }
    }

    // helper methods...
}
```

---

# 14. Lexer Helper Methods

```java
private char advance() {
    char c = source.charAt(current++);
    column++;
    return c;
}

private boolean match(char expected) {
    if (isAtEnd()) {
        return false;
    }

    if (source.charAt(current) != expected) {
        return false;
    }

    current++;
    column++;
    return true;
}

private boolean isAtEnd() {
    return current >= source.length();
}
```

Identifier detection:

```java
private void identifier() {

    while (!isAtEnd() && isAlphaNumeric(peek())) {
        advance();
    }

    String text = source.substring(start, current);

    TokenType type = keywords().getOrDefault(
            text,
            TokenType.IDENTIFIER
    );

    add(type);
}
```

Keyword table:

```java
private static Map<String, TokenType> keywords() {
    return Map.of(
            "fn", TokenType.FN,
            "let", TokenType.LET,
            "if", TokenType.IF,
            "else", TokenType.ELSE,
            "while", TokenType.WHILE,
            "return", TokenType.RETURN,
            "true", TokenType.TRUE,
            "false", TokenType.FALSE,
            "int", TokenType.INT,
            "bool", TokenType.BOOL,
            "string", TokenType.STRING,
            "void", TokenType.VOID
    );
}
```

Number:

```java
private void number() {

    while (!isAtEnd() && isDigit(peek())) {
        advance();
    }

    String text = source.substring(start, current);

    add(
        TokenType.INTEGER_LITERAL,
        Integer.parseInt(text)
    );
}
```

String:

```java
private void string() {

    while (!isAtEnd() && peek() != '"') {
        if (peek() == '\n') {
            line++;
            column = 1;
        }

        advance();
    }

    if (isAtEnd()) {
        throw error("Unterminated string");
    }

    advance();

    String value =
            source.substring(start + 1, current - 1);

    add(TokenType.STRING_LITERAL, value);
}
```

---

# 15. Comments

Add:

```nova
// this is a comment
```

The lexer should recognize:

```text
/
/
```

and ignore everything until newline.

```java
private void scanSlashOrComment() {

    if (match('/')) {
        while (!isAtEnd() && peek() != '\n') {
            advance();
        }
        return;
    }

    add(TokenType.SLASH);
}
```

Later add:

```nova
/*
   multi-line
   comment
*/
```

---

# 16. Lexer Test

Input:

```nova
let x: int = 10 + 20;
```

Test:

```java
@Test
void shouldTokenizeVariableDeclaration() {

    Lexer lexer =
            new NovaLexer("let x: int = 10 + 20;");

    List<Token> tokens = lexer.tokenize();

    assertEquals(TokenType.LET, tokens.get(0).type());
    assertEquals(TokenType.IDENTIFIER, tokens.get(1).type());
    assertEquals(TokenType.COLON, tokens.get(2).type());
    assertEquals(TokenType.INT, tokens.get(3).type());
    assertEquals(TokenType.EQUAL, tokens.get(4).type());
    assertEquals(TokenType.INTEGER_LITERAL, tokens.get(5).type());
    assertEquals(TokenType.PLUS, tokens.get(6).type());
    assertEquals(TokenType.INTEGER_LITERAL, tokens.get(7).type());
}
```

---

# 17. Compiler Phase 4 — AST

Tokens describe syntax at a low level.

AST describes program structure.

For:

```nova
10 + 20 * 3
```

the AST should be:

```text
       +
      / \
    10   *
        / \
       20  3
```

This is much easier for later compiler phases.

---

# 18. AST Base Interfaces

```java
public interface AstNode {

    SourceLocation location();

}
```

Expression:

```java
public interface Expression extends AstNode {
}
```

Statement:

```java
public interface Statement extends AstNode {
}
```

Program:

```java
public record Program(
        List<Statement> statements
) implements AstNode {

    @Override
    public SourceLocation location() {
        return statements.isEmpty()
                ? new SourceLocation(0, 1, 1)
                : statements.get(0).location();
    }
}
```

---

# 19. Literal Expression

```java
public record LiteralExpression(
        Object value,
        SourceLocation location
) implements Expression {
}
```

Examples:

```text
10
true
"hello"
```

---

# 20. Variable Expression

```java
public record VariableExpression(
        String name,
        SourceLocation location
) implements Expression {
}
```

---

# 21. Binary Expression

```java
public record BinaryExpression(
        Expression left,
        TokenType operator,
        Expression right,
        SourceLocation location
) implements Expression {
}
```

For:

```nova
a + b
```

we get:

```text
BinaryExpression(
    Variable(a),
    PLUS,
    Variable(b)
)
```

---

# 22. Unary Expression

```java
public record UnaryExpression(
        TokenType operator,
        Expression operand,
        SourceLocation location
) implements Expression {
}
```

---

# 23. Variable Declaration

```java
public record VariableDeclaration(
        String name,
        TypeNode declaredType,
        Expression initializer,
        SourceLocation location
) implements Statement {
}
```

Type syntax:

```java
public record TypeNode(
        String name,
        SourceLocation location
) implements AstNode {
}
```

Later this should be replaced by a richer type model.

---

# 24. Assignment

```java
public record AssignmentStatement(
        String name,
        Expression value,
        SourceLocation location
) implements Statement {
}
```

---

# 25. Block

```java
public record BlockStatement(
        List<Statement> statements,
        SourceLocation location
) implements Statement {
}
```

---

# 26. If Statement

```java
public record IfStatement(
        Expression condition,
        BlockStatement thenBranch,
        BlockStatement elseBranch,
        SourceLocation location
) implements Statement {
}
```

---

# 27. While Statement

```java
public record WhileStatement(
        Expression condition,
        BlockStatement body,
        SourceLocation location
) implements Statement {
}
```

---

# 28. Return Statement

```java
public record ReturnStatement(
        Expression value,
        SourceLocation location
) implements Statement {
}
```

---

# 29. Function Declaration

```java
public record FunctionDeclaration(
        String name,
        List<Parameter> parameters,
        TypeNode returnType,
        BlockStatement body,
        SourceLocation location
) implements Statement {
}
```

Parameter:

```java
public record Parameter(
        String name,
        TypeNode type,
        SourceLocation location
) implements AstNode {
}
```

---

# 30. Function Call

```java
public record FunctionCallExpression(
        String name,
        List<Expression> arguments,
        SourceLocation location
) implements Expression {
}
```

---

# 31. Compiler Phase 5 — Parser

We will implement a recursive-descent parser.

The parser reads:

```text
Token -> Token -> Token -> ...
```

and produces:

```text
AST
```

The parser methods directly correspond to grammar rules.

For example:

```text
expression
    ::= logicalOr
```

becomes:

```java
private Expression expression() {
    return logicalOr();
}
```

---

# 32. Parser Skeleton

```java
public final class NovaParser {

    private final List<Token> tokens;
    private int current;

    public NovaParser(List<Token> tokens) {
        this.tokens = tokens;
    }

    public Program parse() {

        List<Statement> statements =
                new ArrayList<>();

        while (!isAtEnd()) {
            statements.add(declaration());
        }

        return new Program(statements);
    }

    private Statement declaration() {

        if (match(TokenType.FN)) {
            return functionDeclaration();
        }

        return statement();
    }
}
```

---

# 33. Parsing Variable Declaration

Grammar:

```text
variableDeclaration
    ::= "let" IDENTIFIER (":" type)? "=" expression ";"
```

Implementation:

```java
private Statement variableDeclaration() {

    Token name = consume(
            TokenType.IDENTIFIER,
            "Expected variable name"
    );

    TypeNode type = null;

    if (match(TokenType.COLON)) {
        type = type();
    }

    consume(
            TokenType.EQUAL,
            "Expected '=' after variable name"
    );

    Expression initializer = expression();

    consume(
            TokenType.SEMICOLON,
            "Expected ';' after declaration"
    );

    return new VariableDeclaration(
            name.lexeme(),
            type,
            initializer,
            name.location()
    );
}
```

---

# 34. Parsing Statements

```java
private Statement statement() {

    if (match(TokenType.LET)) {
        return variableDeclaration();
    }

    if (match(TokenType.IF)) {
        return ifStatement();
    }

    if (match(TokenType.WHILE)) {
        return whileStatement();
    }

    if (match(TokenType.RETURN)) {
        return returnStatement();
    }

    if (check(TokenType.LEFT_BRACE)) {
        return block();
    }

    return expressionStatement();
}
```

---

# 35. Parsing Expressions

The recursive-descent parser mirrors precedence.

```java
private Expression expression() {
    return logicalOr();
}
```

```java
private Expression logicalOr() {

    Expression expression = logicalAnd();

    while (match(TokenType.OR_OR)) {

        Token operator = previous();

        Expression right = logicalAnd();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

Logical AND:

```java
private Expression logicalAnd() {

    Expression expression = equality();

    while (match(TokenType.AND_AND)) {

        Token operator = previous();

        Expression right = equality();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

---

# 36. Equality

```java
private Expression equality() {

    Expression expression = comparison();

    while (match(
            TokenType.EQUAL_EQUAL,
            TokenType.BANG_EQUAL
    )) {

        Token operator = previous();

        Expression right = comparison();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

---

# 37. Comparison

```java
private Expression comparison() {

    Expression expression = term();

    while (match(
            TokenType.LESS,
            TokenType.LESS_EQUAL,
            TokenType.GREATER,
            TokenType.GREATER_EQUAL
    )) {

        Token operator = previous();

        Expression right = term();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

---

# 38. Arithmetic

```java
private Expression term() {

    Expression expression = factor();

    while (match(
            TokenType.PLUS,
            TokenType.MINUS
    )) {

        Token operator = previous();

        Expression right = factor();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

```java
private Expression factor() {

    Expression expression = unary();

    while (match(
            TokenType.STAR,
            TokenType.SLASH,
            TokenType.PERCENT
    )) {

        Token operator = previous();

        Expression right = unary();

        expression = new BinaryExpression(
                expression,
                operator.type(),
                right,
                operator.location()
        );
    }

    return expression;
}
```

---

# 39. Unary Expressions

```java
private Expression unary() {

    if (match(
            TokenType.BANG,
            TokenType.MINUS
    )) {

        Token operator = previous();

        Expression operand = unary();

        return new UnaryExpression(
                operator.type(),
                operand,
                operator.location()
        );
    }

    return primary();
}
```

---

# 40. Primary Expressions

```java
private Expression primary() {

    if (match(TokenType.INTEGER_LITERAL)) {
        Token token = previous();

        return new LiteralExpression(
                token.literal(),
                token.location()
        );
    }

    if (match(TokenType.STRING_LITERAL)) {
        Token token = previous();

        return new LiteralExpression(
                token.literal(),
                token.location()
        );
    }

    if (match(TokenType.TRUE)) {
        return new LiteralExpression(
                true,
                previous().location()
        );
    }

    if (match(TokenType.FALSE)) {
        return new LiteralExpression(
                false,
                previous().location()
        );
    }

    if (match(TokenType.IDENTIFIER)) {
        Token token = previous();

        if (match(TokenType.LEFT_PAREN)) {
            return functionCall(token);
        }

        return new VariableExpression(
                token.lexeme(),
                token.location()
        );
    }

    if (match(TokenType.LEFT_PAREN)) {

        Expression expression = expression();

        consume(
                TokenType.RIGHT_PAREN,
                "Expected ')'"
        );

        return expression;
    }

    throw error(peek(), "Expected expression");
}
```

---

# 41. Parser Helper Methods

```java
private boolean match(TokenType... types) {

    for (TokenType type : types) {

        if (check(type)) {
            advance();
            return true;
        }
    }

    return false;
}
```

```java
private Token consume(
        TokenType type,
        String message
) {

    if (check(type)) {
        return advance();
    }

    throw error(peek(), message);
}
```

This is enough to build the first working parser.

---

# 42. Error Recovery

A compiler should not stop at the first syntax error.

Bad:

```text
line 3 error
compiler stopped
```

Better:

```text
line 3: expected ';'
line 7: expected ')'
line 12: unknown statement
```

Implement synchronization:

```java
private void synchronize() {

    advance();

    while (!isAtEnd()) {

        if (previous().type() ==
                TokenType.SEMICOLON) {
            return;
        }

        switch (peek().type()) {

            case FN:
            case LET:
            case IF:
            case WHILE:
            case RETURN:
                return;

            default:
                advance();
        }
    }
}
```

Later implement structured diagnostics rather than throwing generic exceptions.

---

# 43. Compiler Phase 6 — Semantic Analysis

Parsing answers:

> Is this syntactically valid?

Semantic analysis answers:

> Does this program make sense?

Example:

```nova
let x: int = "hello";
```

Syntax is valid.

Semantics are invalid.

Another:

```nova
foo(10);
```

if `foo` does not exist:

```text
undefined function 'foo'
```

---

# 44. Semantic Responsibilities

Semantic analysis should perform:

```text
name resolution
scope validation
duplicate declaration detection
type checking
function validation
return validation
assignment validation
control-flow validation
```

Later:

```text
generic type checking
overload resolution
constant propagation
borrow/lifetime analysis
nullability
```

---

# 45. Symbol Table

Define:

```java
public record Symbol(
        String name,
        Type type,
        SymbolKind kind,
        SourceLocation location
) {}
```

```java
public enum SymbolKind {
    VARIABLE,
    PARAMETER,
    FUNCTION
}
```

Symbol table:

```java
public interface SymbolTable {

    void define(Symbol symbol);

    Optional<Symbol> resolve(String name);
}
```

---

# 46. Nested Scopes

We need:

```nova
let x: int = 10;

if (true) {
    let y: int = 20;
    print(x);
    print(y);
}

print(y); // invalid
```

Use chained scopes:

```java
public final class Scope {

    private final Scope parent;

    private final Map<String, Symbol> symbols =
            new HashMap<>();

    public Scope(Scope parent) {
        this.parent = parent;
    }

    public void define(Symbol symbol) {

        if (symbols.containsKey(symbol.name())) {
            throw new SemanticException(
                    "Duplicate declaration: "
                            + symbol.name()
            );
        }

        symbols.put(symbol.name(), symbol);
    }

    public Optional<Symbol> resolve(String name) {

        Symbol local = symbols.get(name);

        if (local != null) {
            return Optional.of(local);
        }

        if (parent != null) {
            return parent.resolve(name);
        }

        return Optional.empty();
    }
}
```

---

# 47. Type System

Start with:

```java
public sealed interface Type
        permits IntType, BoolType, StringType,
                VoidType, FunctionType {
}
```

Implement:

```java
public record IntType() implements Type {}
public record BoolType() implements Type {}
public record StringType() implements Type {}
public record VoidType() implements Type {}
```

Function:

```java
public record FunctionType(
        List<Type> parameters,
        Type returnType
) implements Type {}
```

Singleton instances can later reduce allocations:

```java
public final class Types {

    public static final Type INT =
            new IntType();

    public static final Type BOOL =
            new BoolType();

    public static final Type STRING =
            new StringType();

    public static final Type VOID =
            new VoidType();

    private Types() {}
}
```

---

# 48. Type Checking Rules

Define operator rules centrally.

For example:

```text
int + int -> int
int - int -> int
int * int -> int
int / int -> int

int < int -> bool
int <= int -> bool
int > int -> bool
int >= int -> bool

int == int -> bool
bool == bool -> bool
string == string -> bool

bool && bool -> bool
bool || bool -> bool

!bool -> bool
-int -> int
```

Do not scatter these rules throughout the compiler.

Create:

```java
public interface OperatorTyping {

    Type binary(
            TokenType operator,
            Type left,
            Type right
    );

    Type unary(
            TokenType operator,
            Type operand
    );
}
```

This makes the type system extensible.

---

# 49. Type Checking Example

For:

```nova
10 + "hello"
```

semantic analyzer obtains:

```text
left  = int
right = string
operator = +
```

Typing engine rejects it:

```text
error: operator '+' cannot be applied to
       int and string
```

---

# 50. Semantic Analyzer

Use the visitor pattern.

```java
public interface AstVisitor<R> {

    R visitProgram(Program node);

    R visitLiteral(LiteralExpression node);

    R visitBinary(BinaryExpression node);

    R visitUnary(UnaryExpression node);

    R visitVariable(VariableExpression node);

    R visitVariableDeclaration(
            VariableDeclaration node);

    R visitIf(IfStatement node);

    R visitWhile(WhileStatement node);

    R visitReturn(ReturnStatement node);

    R visitFunctionDeclaration(
            FunctionDeclaration node);

    R visitFunctionCall(
            FunctionCallExpression node);
}
```

The semantic analyzer implements:

```java
public final class SemanticAnalyzer
        implements AstVisitor<Type> {
}
```

---

# 51. Typed AST

A useful architecture is to retain semantic information.

For example:

```java
public record TypedExpression(
        Expression expression,
        Type type
) {}
```

Eventually:

```text
AST
 |
 v
Typed AST
```

Example:

```text
Binary(+)
    type = int

  Literal(10)
      type = int

  Literal(20)
      type = int
```

This reduces repeated type inference during code generation.

---

# 52. Function Validation

For:

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}
```

register:

```text
add -> FunctionType(
    [int, int],
    int
)
```

When compiling:

```nova
add(10, 20)
```

verify:

```text
function exists
argument count = 2
argument 1 = int
argument 2 = int
```

---

# 53. Return Validation

Invalid:

```nova
fn add(a: int, b: int) -> int {
    return "hello";
}
```

Error:

```text
cannot return string from function returning int
```

Invalid:

```nova
fn add(a: int) -> int {
}
```

Error:

```text
missing return value
```

For `void`:

```nova
fn log(value: string) -> void {
    print(value);
}
```

return value should be forbidden unless language rules explicitly allow it.

---

# 54. Compiler Phase 7 — Intermediate Representation

Do not generate machine code directly from the AST.

Introduce IR.

Why?

Because:

```text
AST -> JVM
AST -> Native
AST -> VM
AST -> WASM
```

would create many frontend/backend dependencies.

Instead:

```text
AST -> IR -> Backend
```

---

# 55. IR Design

Example source:

```nova
let x: int = 10 + 20;
print(x);
```

IR:

```text
CONST 10
CONST 20
ADD
STORE_LOCAL x
LOAD_LOCAL x
CALL print 1
```

---

# 56. IR Opcodes

```java
public enum Opcode {

    CONST_INT,
    CONST_BOOL,
    CONST_STRING,

    LOAD_LOCAL,
    STORE_LOCAL,

    ADD,
    SUB,
    MUL,
    DIV,
    MOD,

    NEGATE,
    NOT,

    EQ,
    NE,
    LT,
    LE,
    GT,
    GE,

    JUMP,
    JUMP_IF_FALSE,

    CALL,
    RETURN,

    POP
}
```

---

# 57. IR Instruction

```java
public record IrInstruction(
        Opcode opcode,
        Object operand
) {}
```

However, a scalable compiler should eventually avoid `Object` everywhere.

Better:

```java
public sealed interface IrOperand
        permits IntegerOperand,
                StringOperand,
                LocalOperand,
                LabelOperand,
                FunctionOperand {
}
```

This gives the IR stronger type safety.

---

# 58. Labels

Conditional:

```nova
if (x > 10) {
    print("large");
} else {
    print("small");
}
```

IR:

```text
LOAD_LOCAL x
CONST_INT 10
GT

JUMP_IF_FALSE L1

CONST_STRING "large"
CALL print 1
JUMP L2

L1:
CONST_STRING "small"
CALL print 1

L2:
```

---

# 59. IR Builder

Create:

```java
public final class IrBuilder {

    private final List<IrInstruction> instructions =
            new ArrayList<>();

    private int nextLabel = 0;

    public void emit(
            Opcode opcode,
            Object operand
    ) {
        instructions.add(
                new IrInstruction(opcode, operand)
        );
    }

    public Label newLabel() {
        return new Label("L" + nextLabel++);
    }

    public void mark(Label label) {
        // emit label metadata
    }
}
```

Later replace this with a basic-block based CFG.

---

# 60. Control Flow Graph

For scalable optimization, represent IR as:

```text
Function
   |
   +-- BasicBlock 0
   |      |
   |      +-- BasicBlock 1
   |      |
   |      +-- BasicBlock 2
   |
   +-- BasicBlock 3
```

Each block has:

```text
instructions
predecessors
successors
```

This enables:

```text
dead code elimination
constant propagation
control-flow analysis
reachability
jump optimization
```

---

# 61. Compiler Phase 8 — Bytecode

Create a simple Nova bytecode format.

Example:

```text
CONST_INT 10
CONST_INT 20
ADD
PRINT
HALT
```

A bytecode instruction can be:

```java
public record BytecodeInstruction(
        BytecodeOpcode opcode,
        int operand
) {}
```

Opcode:

```java
public enum BytecodeOpcode {

    ICONST,
    BCONST,
    SCONST,

    LOAD,
    STORE,

    IADD,
    ISUB,
    IMUL,
    IDIV,

    IEQ,
    ILT,
    IGT,

    JMP,
    JMP_FALSE,

    CALL,
    RETURN,

    PRINT,
    HALT
}
```

---

# 62. Why Build Our Own VM?

It teaches:

```text
stack machine design
instruction decoding
call stack
stack frames
local variables
function invocation
return values
runtime errors
```

The compiler becomes:

```text
Nova Source
    |
Lexer
    |
Parser
    |
AST
    |
Semantic Analysis
    |
IR
    |
Optimization
    |
Bytecode
    |
Nova VM
```

---

# 63. Stack-Based Virtual Machine

For:

```nova
10 + 20
```

VM stack:

```text
PUSH 10

stack:
[10]

PUSH 20

stack:
[10, 20]

ADD

stack:
[30]
```

---

# 64. VM Architecture

```java
public final class VirtualMachine {

    private final Deque<Object> stack =
            new ArrayDeque<>();

    private final Object[] locals =
            new Object[256];

    public void execute(
            List<BytecodeInstruction> code
    ) {

        int pc = 0;

        while (pc < code.size()) {

            BytecodeInstruction instruction =
                    code.get(pc);

            switch (instruction.opcode()) {

                case ICONST ->
                    stack.push(instruction.operand());

                case IADD -> {
                    int right = (int) stack.pop();
                    int left = (int) stack.pop();

                    stack.push(left + right);
                }

                default ->
                    throw new UnsupportedOperationException(
                            "Opcode not implemented: "
                            + instruction.opcode()
                    );
            }

            pc++;
        }
    }
}
```

This is deliberately simple.

---

# 65. Stack Frames

Functions require separate execution state.

For:

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}
```

create:

```java
public final class StackFrame {

    private final Object[] locals;

    private int programCounter;

    private final StackFrame caller;

    public StackFrame(
            int localCount,
            StackFrame caller
    ) {
        this.locals =
                new Object[localCount];

        this.caller = caller;
    }

    public Object[] locals() {
        return locals;
    }

    public StackFrame caller() {
        return caller;
    }
}
```

The VM maintains:

```text
currentFrame
```

and creates a new frame on every function call.

---

# 66. Function Calls

Bytecode:

```text
CALL add 2
```

VM:

```text
1. pop argument values
2. find function
3. create StackFrame
4. initialize parameters
5. execute function
6. return value
7. restore caller frame
```

---

# 67. Runtime Values

Do not permanently use raw `Object`.

Create:

```java
public sealed interface RuntimeValue
        permits IntValue,
                BoolValue,
                StringValue,
                NullValue {
}
```

```java
public record IntValue(int value)
        implements RuntimeValue {
}
```

```java
public record BoolValue(boolean value)
        implements RuntimeValue {
}
```

This gives the runtime explicit semantics.

---

# 68. Runtime Errors

Examples:

```text
division by zero
undefined function
invalid opcode
stack underflow
invalid local index
invalid jump
```

Create:

```java
public final class RuntimeException
        extends RuntimeException {

    public RuntimeException(String message) {
        super(message);
    }
}
```

Prefer a domain-specific name such as:

```java
NovaRuntimeException
```

to avoid confusion with `java.lang.RuntimeException`.

---

# 69. Compiler Phase 9 — Optimization

Once the compiler works, add optimization passes.

Design:

```java
public interface OptimizationPass {

    String name();

    IrProgram apply(IrProgram program);
}
```

Optimizer:

```java
public final class Optimizer {

    private final List<OptimizationPass> passes;

    public Optimizer(
            List<OptimizationPass> passes
    ) {
        this.passes =
                List.copyOf(passes);
    }

    public IrProgram optimize(
            IrProgram program
    ) {

        IrProgram current = program;

        for (OptimizationPass pass : passes) {
            current = pass.apply(current);
        }

        return current;
    }
}
```

This follows Open/Closed Principle.

---

# 70. Constant Folding

Input:

```nova
let x = 10 + 20 * 3;
```

AST:

```text
10 + (20 * 3)
```

Optimizer:

```text
10 + 60
```

then:

```text
70
```

Instead of runtime computation.

---

# 71. Constant Folding Algorithm

For binary expression:

```text
if left is constant
and right is constant
and operator is pure
then calculate result
```

Example:

```text
ADD(10, 20)
```

becomes:

```text
CONST(30)
```

Do not fold unsafe operations blindly.

For example:

```text
10 / 0
```

requires the language specification to define whether this is:

```text
compile-time error
runtime error
undefined
```

---

# 72. Dead Code Elimination

Input:

```nova
if (false) {
    print("never");
}
```

Can become:

```text
nothing
```

But be careful if expressions can have side effects.

For example:

```nova
if (false) {
    dangerousOperation();
}
```

The language's semantics must determine whether removing the call is valid.

---

# 73. Other Optimization Passes

Implement incrementally:

```text
Constant Folding
Constant Propagation
Dead Code Elimination
Copy Propagation
Peephole Optimization
Jump Threading
Common Subexpression Elimination
Inlining
Loop Invariant Code Motion
Strength Reduction
```

Do not implement all of them initially.

---

# 74. Compiler Pipeline Object

Create a central orchestration class.

```java
public final class Compiler {

    private final LexerFactory lexerFactory;
    private final ParserFactory parserFactory;
    private final SemanticAnalyzer semanticAnalyzer;
    private final IrGenerator irGenerator;
    private final Optimizer optimizer;
    private final Backend backend;

    public Compiler(
            LexerFactory lexerFactory,
            ParserFactory parserFactory,
            SemanticAnalyzer semanticAnalyzer,
            IrGenerator irGenerator,
            Optimizer optimizer,
            Backend backend
    ) {
        this.lexerFactory = lexerFactory;
        this.parserFactory = parserFactory;
        this.semanticAnalyzer = semanticAnalyzer;
        this.irGenerator = irGenerator;
        this.optimizer = optimizer;
        this.backend = backend;
    }

    public CompilationResult compile(
            String source
    ) {

        Lexer lexer =
                lexerFactory.create(source);

        List<Token> tokens =
                lexer.tokenize();

        Parser parser =
                parserFactory.create(tokens);

        Program ast =
                parser.parse();

        semanticAnalyzer.analyze(ast);

        IrProgram ir =
                irGenerator.generate(ast);

        IrProgram optimized =
                optimizer.optimize(ir);

        return backend.compile(optimized);
    }
}
```

The compiler itself should be orchestration, not implementation.

---

# 75. Backend Abstraction

Define:

```java
public interface Backend {

    CompilationResult compile(
            IrProgram program
    );
}
```

Implement:

```text
NovaBytecodeBackend
JvmBackend
NativeBackend
WasmBackend
```

Later:

```text
LLVMBackend
```

The frontend should not know which backend is selected.

---

# 76. CLI

Create:

```bash
nova compile hello.nova
nova run hello.nova
nova check hello.nova
nova dump-tokens hello.nova
nova dump-ast hello.nova
nova dump-ir hello.nova
nova disassemble hello.nova
```

This is extremely useful while developing the compiler.

---

# 77. Compiler Debug Modes

Implement:

```text
--tokens
--ast
--typed-ast
--ir
--optimized-ir
--bytecode
--trace
```

Example:

```bash
nova compile example.nova --dump-ir
```

Output:

```text
function main:
    CONST_INT 10
    CONST_INT 20
    ADD
    CALL print 1
    RETURN
```

Compiler developers rely heavily on these intermediate representations.

---

# 78. Complete Example

Source:

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}

fn main() -> void {

    let x: int = 10;
    let y: int = 20;

    let result: int = add(x, y);

    if (result > 20) {
        print("greater");
    } else {
        print("small");
    }
}
```

---

# 79. Lexical Representation

The lexer produces:

```text
FN
IDENTIFIER(add)
LEFT_PAREN
IDENTIFIER(a)
COLON
INT
COMMA
IDENTIFIER(b)
COLON
INT
RIGHT_PAREN
ARROW
INT
LEFT_BRACE
RETURN
IDENTIFIER(a)
PLUS
IDENTIFIER(b)
SEMICOLON
RIGHT_BRACE
...
```

---

# 80. AST Representation

```text
Program
 |
 +-- FunctionDeclaration add
 |      |
 |      +-- Parameter a:int
 |      +-- Parameter b:int
 |      +-- Return:int
 |      |
 |      +-- Return
 |             |
 |             +-- Binary +
 |                   |
 |                   +-- Variable a
 |                   +-- Variable b
 |
 +-- FunctionDeclaration main
        |
        +-- Variable x
        +-- Variable y
        +-- Variable result
        |      |
        |      +-- Call add
        |
        +-- If
               |
               +-- Binary >
               |
               +-- Then
               |
               +-- Else
```

---

# 81. Semantic Representation

The analyzer creates symbols:

```text
Global Scope

add -> Function(int,int -> int)
main -> Function(() -> void)
```

Inside `add`:

```text
a -> int
b -> int
```

Inside `main`:

```text
x      -> int
y      -> int
result -> int
```

---

# 82. IR

Conceptually:

```text
function add:

LOAD_LOCAL 0
LOAD_LOCAL 1
ADD
RETURN
```

Main:

```text
CONST_INT 10
STORE_LOCAL 0

CONST_INT 20
STORE_LOCAL 1

LOAD_LOCAL 0
LOAD_LOCAL 1
CALL add 2
STORE_LOCAL 2

LOAD_LOCAL 2
CONST_INT 20
GT
JUMP_IF_FALSE L1

CONST_STRING "greater"
CALL print 1
JUMP L2

L1:
CONST_STRING "small"
CALL print 1

L2:
RETURN
```

---

# 83. Testing Strategy

Do not test only the final program.

Test every compiler phase independently.

```text
Lexer Tests
    |
Parser Tests
    |
Semantic Tests
    |
IR Tests
    |
Optimizer Tests
    |
VM Tests
    |
End-to-End Tests
```

---

# 84. Lexer Tests

Test:

```text
keywords
identifiers
numbers
strings
operators
comments
whitespace
newlines
invalid characters
unterminated strings
```

---

# 85. Parser Tests

Test:

```text
precedence
associativity
nested expressions
if/else
while
functions
function calls
return
blocks
invalid syntax
```

Example:

```java
@Test
void shouldRespectOperatorPrecedence() {

    Program program =
            parse("let x = 10 + 20 * 3;");

    VariableDeclaration declaration =
            (VariableDeclaration)
                    program.statements().get(0);

    assertInstanceOf(
            BinaryExpression.class,
            declaration.initializer()
    );
}
```

---

# 86. Semantic Tests

Test:

```text
undefined variable
duplicate variable
undefined function
wrong argument count
wrong argument type
wrong return type
invalid operator
invalid assignment
missing return
scope visibility
```

---

# 87. VM Tests

Example:

```java
@Test
void shouldAddIntegers() {

    List<BytecodeInstruction> code =
            List.of(
                    new BytecodeInstruction(
                            BytecodeOpcode.ICONST,
                            10
                    ),
                    new BytecodeInstruction(
                            BytecodeOpcode.ICONST,
                            20
                    ),
                    new BytecodeInstruction(
                            BytecodeOpcode.IADD,
                            0
                    )
            );

    VirtualMachine vm =
            new VirtualMachine();

    vm.execute(code);

    assertEquals(30, vm.peek());
}
```

---

# 88. End-to-End Test

Input:

```nova
fn main() -> void {
    let x: int = 10;
    let y: int = 20;
    print(x + y);
}
```

Expected:

```text
30
```

Test:

```java
@Test
void shouldCompileAndRunProgram() {

    String source = """
        fn main() -> void {
            let x: int = 10;
            let y: int = 20;
            print(x + y);
        }
        """;

    CompilationResult result =
            compiler.compile(source);

    vm.execute(result.bytecode());

    assertEquals(
            "30",
            output.toString()
    );
}
```

---

# 89. Error Reporting Architecture

Do not throw raw strings from every component.

Define:

```java
public enum DiagnosticSeverity {
    INFO,
    WARNING,
    ERROR
}
```

```java
public record Diagnostic(
        DiagnosticSeverity severity,
        String message,
        SourceLocation location
) {}
```

Reporter:

```java
public interface DiagnosticReporter {

    void report(Diagnostic diagnostic);

    boolean hasErrors();

    List<Diagnostic> diagnostics();
}
```

Then:

```text
Lexer
Parser
Semantic Analyzer
Optimizer
Backend
```

can all report diagnostics consistently.

---

# 90. Better Compiler API

Instead of:

```java
throw new RuntimeException(...)
```

use:

```java
CompilationResult result =
        compiler.compile(source);

if (result.hasErrors()) {
    diagnostics.print(result);
}
```

Result:

```java
public record CompilationResult(
        Optional<BytecodeProgram> bytecode,
        List<Diagnostic> diagnostics
) {

    public boolean hasErrors() {
        return diagnostics.stream()
                .anyMatch(d ->
                        d.severity()
                        == DiagnosticSeverity.ERROR);
    }
}
```

---

# 91. Error Example

Source:

```nova
fn main() -> void {
    let x: int = "hello";
}
```

Output:

```text
example.nova:2:19

let x: int = "hello";
                  ^^^^^^^

error: cannot assign string to int
```

The diagnostic subsystem should eventually print source snippets and carets.

---

# 92. Visitor Pattern vs Pattern Matching

Two good choices exist.

Visitor:

```java
expression.accept(visitor);
```

Pattern matching:

```java
switch (expression) {
    case LiteralExpression literal -> ...
    case BinaryExpression binary -> ...
    case VariableExpression variable -> ...
}
```

For a small Java 21 compiler, sealed interfaces + pattern matching are attractive.

For a compiler expected to have many operations over a stable AST, Visitor can also work well.

Choose one consistently.

---

# 93. SOLID Architecture

## Single Responsibility

Lexer:

```text
characters -> tokens
```

Parser:

```text
tokens -> AST
```

Semantic analyzer:

```text
AST -> semantic validation
```

IR generator:

```text
AST -> IR
```

Optimizer:

```text
IR -> optimized IR
```

Backend:

```text
IR -> target representation
```

---

# 94. Open/Closed Principle

Adding:

```text
WasmBackend
```

should not require modifying:

```text
Lexer
Parser
AST
SemanticAnalyzer
```

Similarly:

```text
DeadCodeEliminationPass
```

should be added as:

```java
new DeadCodeEliminationPass()
```

rather than modifying the optimizer's core algorithm.

---

# 95. Dependency Inversion

Bad:

```java
class Compiler {

    private final NovaVmBackend backend =
            new NovaVmBackend();
}
```

Better:

```java
class Compiler {

    private final Backend backend;

    Compiler(Backend backend) {
        this.backend = backend;
    }
}
```

Now:

```java
Backend backend =
        new NovaVmBackend();
```

or:

```java
Backend backend =
        new JvmBackend();
```

---

# 96. Factory Design

Use factories when compiler configuration becomes complicated.

```java
public interface CompilerFactory {

    Compiler create(
            CompilerConfiguration configuration
    );
}
```

Configuration:

```java
public record CompilerConfiguration(
        OptimizationLevel optimizationLevel,
        Target target,
        boolean debug
) {}
```

Target:

```java
public enum Target {
    NOVA_VM,
    JVM,
    WASM
}
```

---

# 97. Dependency Injection

The compiler eventually has:

```text
Compiler
  |
  +-- LexerFactory
  +-- ParserFactory
  +-- SemanticAnalyzer
  +-- TypeChecker
  +-- IrGenerator
  +-- Optimizer
  +-- Backend
  +-- DiagnosticReporter
```

Use constructor injection.

This also makes unit testing much easier.

---

# 98. Configuration

Example:

```java
CompilerConfiguration configuration =
        new CompilerConfiguration(
                OptimizationLevel.O2,
                Target.NOVA_VM,
                true
        );
```

Later:

```text
O0
O1
O2
O3
debug
release
target=jvm
target=wasm
```

---

# 99. Language Evolution

Do not immediately add:

```text
classes
interfaces
generics
async
threads
exceptions
modules
reflection
macros
```

Implement in stages.

Recommended roadmap:

```text
Stage 1
    integers
    arithmetic

Stage 2
    variables
    boolean
    conditions

Stage 3
    loops

Stage 4
    functions

Stage 5
    strings

Stage 6
    type checking

Stage 7
    IR

Stage 8
    VM

Stage 9
    optimizer

Stage 10
    arrays

Stage 11
    structs

Stage 12
    modules

Stage 13
    generics

Stage 14
    classes/interfaces

Stage 15
    JVM backend
```

---

# 100. Arrays

Syntax:

```nova
let numbers: int[] = [10, 20, 30];

print(numbers[0]);
```

Grammar additions:

```text
type
    ::= primitiveType
     | type "[" "]"

primary
    ::= ...
     | "[" arguments? "]"
```

AST:

```java
public record ArrayLiteralExpression(
        List<Expression> elements,
        SourceLocation location
) implements Expression {}
```

Index:

```java
public record IndexExpression(
        Expression target,
        Expression index,
        SourceLocation location
) implements Expression {}
```

---

# 101. Structs

Syntax:

```nova
struct User {
    id: int;
    name: string;
}
```

Usage:

```nova
let user = User {
    id: 10,
    name: "Arpan"
};
```

This requires:

```text
StructDeclaration
FieldDeclaration
StructType
FieldAccessExpression
StructLiteralExpression
```

---

# 102. Modules

Eventually:

```text
src/
    main.nova
    math.nova
    user.nova
```

Syntax:

```nova
import math;
```

Then:

```text
ModuleLoader
ModuleResolver
DependencyGraph
```

The compiler must detect:

```text
cyclic imports
missing modules
duplicate exports
visibility violations
```

---

# 103. Generics

Eventually:

```nova
fn identity<T>(value: T) -> T {
    return value;
}
```

This introduces:

```text
TypeParameter
GenericFunctionType
TypeInference
GenericInstantiation
```

Do not add generics until the basic type system is stable.

---

# 104. Classes

Possible syntax:

```nova
class User {

    name: string;

    fn getName() -> string {
        return self.name;
    }
}
```

This requires:

```text
class type
object layout
method lookup
self/this
constructor
visibility
inheritance or interfaces
```

This is a major compiler/runtime feature.

---

# 105. Exceptions

Possible syntax:

```nova
try {
    risky();
} catch (e) {
    print(e);
}
```

This affects:

```text
grammar
AST
semantic analysis
IR
bytecode
VM
runtime
stack unwinding
```

It should therefore be added only after the runtime architecture is mature.

---

# 106. Garbage Collection

Once Nova supports heap objects:

```text
string
array
struct
object
closure
```

memory management becomes necessary.

Possible stages:

```text
Stage 1:
manual heap ownership

Stage 2:
reference counting

Stage 3:
mark-and-sweep

Stage 4:
generational GC
```

For a learning compiler, implement mark-and-sweep first.

---

# 107. Heap Design

Conceptually:

```text
VM
 |
 +-- Stack
 |
 +-- Heap
       |
       +-- Object
       +-- Array
       +-- String
       +-- Struct
```

Heap object:

```java
public interface HeapObject {

    ObjectType type();

    boolean marked();

    void marked(boolean value);

    List<HeapObject> references();
}
```

GC:

```text
1. Find roots
2. Mark reachable objects
3. Sweep unreachable objects
```

---

# 108. Closures

Eventually support:

```nova
fn createCounter() -> fn() -> int {

    let count: int = 0;

    fn increment() -> int {
        count = count + 1;
        return count;
    }

    return increment;
}
```

This requires:

```text
closures
captured variables
environment objects
heap allocation
function values
```

This is a major milestone.

---

# 109. JVM Backend

Once Nova IR is stable, add:

```java
public final class JvmBackend
        implements Backend {
}
```

Pipeline:

```text
Nova
 |
AST
 |
Semantic Analysis
 |
Nova IR
 |
JVM Backend
 |
JVM bytecode
 |
.class
 |
JVM
```

This is preferable to compiling directly from AST to JVM bytecode because the IR becomes the common contract.

---

# 110. Multiple Backends

Final architecture:

```text
                    +--> Nova VM
                    |
Nova Source        +--> JVM
    |              |
    v              +--> WebAssembly
Frontend            |
    |              +--> Native
    v
Nova IR
    |
Optimizer
```

This is the architecture that makes the project scalable.

---

# 111. Intermediate Representation as the Most Important Boundary

A strong compiler architecture has:

```text
Frontend
    |
    | language-specific
    v
AST / Typed AST
    |
    v
IR
    |
    | language-independent
    v
Backend
```

The backend should not care whether the source contained:

```nova
x + y
```

or some other surface syntax.

It receives normalized IR.

---

# 112. Separate Frontend, Middle-End, Backend

Use terminology:

## Frontend

```text
Lexer
Parser
AST
Semantic Analysis
Type Checking
```

## Middle-End

```text
IR
CFG
Optimization
Data-flow analysis
```

## Backend

```text
Instruction selection
Register allocation
Bytecode generation
Machine code generation
```

This separation mirrors real compiler architecture.

---

# 113. Intermediate Representation Levels

For a larger compiler, introduce:

```text
AST
 |
Typed AST
 |
High-Level IR
 |
Control Flow IR
 |
Low-Level IR
 |
Machine-specific IR
 |
Machine code
```

Do not build all levels on day one.

Start with:

```text
AST -> IR -> Bytecode
```

Then evolve.

---

# 114. Control Flow Graph

Represent:

```java
public final class BasicBlock {

    private final int id;

    private final List<IrInstruction> instructions;

    private final List<BasicBlock> successors;

    private final List<BasicBlock> predecessors;
}
```

A function:

```java
public final class IrFunction {

    private final String name;

    private final List<BasicBlock> blocks;

    private final FunctionType type;
}
```

Program:

```java
public final class IrProgram {

    private final List<IrFunction> functions;
}
```

This structure is much better than one giant instruction list for advanced optimization.

---

# 115. Data-Flow Analysis

Once CFG exists, implement:

```text
reaching definitions
live variable analysis
available expressions
constant propagation
```

For live variables:

```text
IN[B]  = USE[B] ∪ (OUT[B] - DEF[B])

OUT[B] =
    union(IN[S])
    for every successor S
```

This teaches the mathematical foundation behind many compiler optimizations.

---

# 116. Register Allocation

For native code:

```text
virtual register
       |
       v
live ranges
       |
       v
interference graph
       |
       v
graph coloring
       |
       v
physical registers
```

Start with:

```text
stack-based bytecode
```

Then move toward:

```text
three-address IR
```

and finally:

```text
register allocation
```

---

# 117. Three-Address Code

Instead of:

```text
ADD
```

use:

```text
t1 = a + b
t2 = t1 * c
```

Example:

```text
t1 = 20 * 3
t2 = 10 + t1
store x, t2
```

This representation makes many optimization passes easier.

---

# 118. SSA

A more advanced stage is Static Single Assignment.

Example:

```text
if condition:
    x1 = 10
else:
    x2 = 20

x3 = phi(x1, x2)
```

SSA simplifies:

```text
constant propagation
dead code elimination
value numbering
data-flow analysis
```

Do not implement SSA until your CFG and IR are stable.

---

# 119. Compiler Pass Manager

Eventually:

```java
public interface CompilerPass {
    void run(CompilationUnit unit);
}
```

Pass manager:

```java
public final class PassManager {

    private final List<CompilerPass> passes;

    public void run(CompilationUnit unit) {

        for (CompilerPass pass : passes) {
            pass.run(unit);
        }
    }
}
```

Configuration:

```text
O0:
    no optimization

O1:
    constant folding
    dead code elimination

O2:
    + constant propagation
    + copy propagation

O3:
    + inlining
    + loop optimization
```

---

# 120. Source-to-Source Debugging

A very useful feature:

```bash
nova dump-tokens program.nova
nova dump-ast program.nova
nova dump-types program.nova
nova dump-ir program.nova
nova dump-bytecode program.nova
```

Every phase should have a human-readable printer.

For AST:

```java
public interface AstPrinter {

    String print(Program program);
}
```

IR:

```java
public interface IrPrinter {

    String print(IrProgram program);
}
```

---

# 121. Pretty Printer

Eventually build:

```text
Nova source
   |
AST
   |
Pretty Printer
   |
Normalized Nova source
```

This allows:

```bash
nova fmt program.nova
```

Example:

```nova
fn add(a:int,b:int)->int{return a+b;}
```

becomes:

```nova
fn add(a: int, b: int) -> int {
    return a + b;
}
```

---

# 122. IDE Support

Once the compiler has good source locations and diagnostics, it can power an IDE.

Expose:

```text
parse()
diagnostics()
autocomplete()
symbol lookup
go to definition
find references
type information
```

This can later become:

```text
Language Server Protocol (LSP)
```

The compiler frontend becomes reusable by:

```text
CLI
IDE
formatter
linter
REPL
language server
```

---

# 123. REPL

Create:

```bash
nova repl
```

Input:

```text
> let x: int = 10;
> x + 20
30
```

The REPL pipeline is:

```text
input
 |
lexer
 |
parser
 |
semantic
 |
IR
 |
VM
 |
result
```

The environment must persist across commands.

---

# 124. Standard Library

Eventually define:

```text
print
println
readLine
length
substring
math
collections
```

Separate:

```text
compiler built-ins
```

from:

```text
standard library
```

For example:

```nova
print("hello");
```

could compile to:

```text
CALL builtin.print 1
```

while more advanced libraries can be implemented as Nova modules.

---

# 125. Native Interoperability

Later:

```text
Nova
 |
FFI
 |
C ABI
 |
Native libraries
```

This requires:

```text
calling convention
type mapping
memory ownership
ABI compatibility
```

Do not attempt this until the VM runtime is stable.

---

# 126. Security Considerations

If executing untrusted Nova programs, the VM must enforce:

```text
maximum stack depth
maximum execution steps
memory limits
file access permissions
network permissions
native-call restrictions
```

Never assume bytecode is trusted simply because the compiler generated it.

---

# 127. Performance Considerations

Initial implementation:

```text
String
ArrayList<Token>
AST objects
Object-based VM
```

is acceptable.

Later optimize:

```text
character scanning
token allocation
AST allocation
symbol lookup
IR representation
bytecode decoding
VM dispatch
```

Potential VM strategies:

```text
switch dispatch
computed dispatch
threaded interpretation
JIT compilation
```

Java implementation can eventually explore:

```text
JIT
invokedynamic
MethodHandle
JVM bytecode generation
```

---

# 128. Incremental Compilation

A production compiler should eventually avoid rebuilding everything.

For:

```text
file A
file B
file C
```

if only `B` changes:

```text
parse B
analyze B
recompile affected dependencies
reuse A/C artifacts
```

Requires:

```text
module dependency graph
content hashes
incremental AST cache
incremental semantic cache
compiled artifacts
```

---

# 129. Build Cache

Use:

```text
source hash
compiler version
language version
compiler flags
dependency hashes
```

to calculate:

```text
CompilationCacheKey
```

Example:

```java
public record CompilationCacheKey(
        String sourceHash,
        String compilerVersion,
        String languageVersion,
        String optionsHash
) {}
```

---

# 130. Deterministic Builds

For the same:

```text
source
compiler version
dependencies
configuration
```

the compiler should produce the same output.

Avoid:

```text
HashMap iteration order
current time
random identifiers
machine-specific paths
```

in generated artifacts unless deliberately controlled.

---

# 131. Compiler Versioning

Define:

```text
Nova language version
Nova compiler version
Nova bytecode version
```

For example:

```text
language = 1.0
compiler = 0.5.0
bytecode = 3
```

This becomes important when bytecode survives across compiler upgrades.

---

# 132. Bytecode Verification

Before VM execution:

```text
verify stack balance
verify jump targets
verify local indexes
verify opcode operands
verify function signatures
```

Example:

```text
PUSH
PUSH
ADD
RETURN
```

is valid.

But:

```text
ADD
```

with an empty stack is invalid.

A verifier prevents corrupted bytecode from crashing the VM unpredictably.

---

# 133. VM Instruction Verification

Create:

```java
public interface BytecodeVerifier {

    void verify(BytecodeProgram program);
}
```

Checks:

```text
stack height
operand types
jump targets
function references
local variable indexes
return consistency
```

---

# 134. Compiler Architecture After Growth

Final architecture:

```text
                         +------------------+
                         | CLI / IDE / REPL |
                         +---------+--------+
                                   |
                                   v
                         +------------------+
                         | Compiler API     |
                         +---------+--------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
                 v                                   v
          +-------------+                    +---------------+
          | Diagnostics |                    | Configuration |
          +-------------+                    +---------------+
                 |
                 v
          +-------------+
          |   Lexer     |
          +-------------+
                 |
                 v
          +-------------+
          |   Parser    |
          +-------------+
                 |
                 v
          +-------------+
          | AST         |
          +-------------+
                 |
                 v
          +-------------+
          | Semantic    |
          | Analysis    |
          +-------------+
                 |
                 v
          +-------------+
          | Typed AST   |
          +-------------+
                 |
                 v
          +-------------+
          | IR          |
          +-------------+
                 |
                 v
          +-------------+
          | CFG / SSA   |
          +-------------+
                 |
                 v
          +-------------+
          | Optimizer   |
          +-------------+
                 |
        +--------+---------+
        |        |         |
        v        v         v
     Nova VM    JVM       WASM
```

---

# 135. Recommended Implementation Order

Do not implement everything simultaneously.

## Milestone 1 — Calculator

Support:

```nova
10 + 20 * 3
```

Implement:

```text
Lexer
Parser
AST
Evaluator
```

---

## Milestone 2 — Variables

Support:

```nova
let x: int = 10;
x + 20;
```

Implement:

```text
VariableDeclaration
VariableExpression
Scope
```

---

## Milestone 3 — Types

Support:

```text
int
bool
string
void
```

Implement:

```text
Type
TypeChecker
```

---

## Milestone 4 — Conditions

Support:

```nova
if (x > 10) {
    print(x);
}
```

Implement:

```text
IfStatement
conditional IR
jumps
```

---

## Milestone 5 — Loops

Support:

```nova
while (x < 10) {
    x = x + 1;
}
```

Implement:

```text
WhileStatement
CFG
backward jumps
```

---

## Milestone 6 — Functions

Implement:

```text
FunctionDeclaration
FunctionCallExpression
ReturnStatement
FunctionType
StackFrame
CALL
RETURN
```

---

## Milestone 7 — Bytecode VM

Replace direct AST interpretation with:

```text
AST
 ->
IR
 ->
Bytecode
 ->
VM
```

---

## Milestone 8 — Diagnostics

Add:

```text
source location
error codes
source snippets
multiple diagnostics
warnings
```

---

## Milestone 9 — Optimizer

Implement:

```text
constant folding
constant propagation
dead code elimination
```

---

## Milestone 10 — Arrays

Implement:

```text
heap
array runtime object
indexing
bounds checking
```

---

## Milestone 11 — Structs

Implement:

```text
StructType
field access
heap objects
```

---

## Milestone 12 — Modules

Implement:

```text
module resolver
imports
exports
dependency graph
```

---

## Milestone 13 — Garbage Collector

Implement:

```text
heap
roots
mark
sweep
```

---

## Milestone 14 — JVM Backend

Generate JVM bytecode.

---

## Milestone 15 — Advanced Optimizations

Implement:

```text
CFG
SSA
inlining
CSE
LICM
register allocation
```

---

# 136. Recommended Git Commit Structure

Build incrementally.

```text
01-initialize-project
02-add-source-model
03-add-token-model
04-implement-lexer
05-add-lexer-tests
06-add-expression-ast
07-implement-expression-parser
08-add-statement-parser
09-add-parser-tests
10-add-symbol-table
11-add-type-system
12-add-semantic-analysis
13-add-ir
14-add-ir-generator
15-add-bytecode
16-add-vm
17-add-function-calls
18-add-control-flow
19-add-diagnostics
20-add-constant-folding
21-add-dead-code-elimination
22-add-arrays
23-add-structs
24-add-modules
25-add-garbage-collector
26-add-jvm-backend
```

This makes debugging much easier.

---

# 137. Interview Follow-Up Questions

After implementing the basic compiler, ask:

## Lexer

1. Why use a lexer instead of parsing characters directly?
2. How would you support Unicode identifiers?
3. How do you handle escaped strings?
4. How do you tokenize nested comments?
5. How can lexer performance be improved?

## Parser

6. Why recursive descent?
7. What is operator precedence?
8. What is left recursion?
9. How can left recursion be removed?
10. When would Pratt parsing be better?
11. How do you recover from parser errors?

## Semantic Analysis

12. What is a symbol table?
13. How do nested scopes work?
14. How do you detect duplicate variables?
15. How do you resolve functions?
16. Where should type checking happen?

## IR

17. Why introduce IR?
18. Why not compile AST directly to JVM?
19. What is three-address code?
20. What is SSA?
21. What is a basic block?
22. What is a control-flow graph?

## Optimization

23. What is constant folding?
24. What is constant propagation?
25. What is dead code elimination?
26. What is common subexpression elimination?
27. What is function inlining?
28. Why can optimization change debugging behavior?

## Runtime

29. Stack VM vs register VM?
30. What is a stack frame?
31. How are function arguments passed?
32. How are closures implemented?
33. How does garbage collection find roots?

## Backend

34. What is instruction selection?
35. What is register allocation?
36. What is the interference graph?
37. Why is SSA useful?
38. How would you add a JVM backend?
39. How would you add WebAssembly?

---

# 138. Advanced Compiler Follow-Ups

Once the project works, explore:

```text
1. Pratt Parser
2. Error-tolerant parsing
3. Typed AST
4. Three-address IR
5. CFG
6. SSA
7. Data-flow analysis
8. Constant propagation
9. CSE
10. Dead code elimination
11. Inlining
12. Loop optimization
13. Closure conversion
14. Escape analysis
15. Garbage collection
16. JIT
17. JVM backend
18. Native backend
19. WASM backend
20. Incremental compilation
21. Parallel compilation
22. LSP
23. Debug information
24. Source maps
25. Package manager
```

---

# 139. Important Design Principle

Do not let language syntax leak into the VM.

Bad:

```java
if (token.type() == TokenType.PLUS) {
    vm.add();
}
```

The VM should never know what a `TokenType` is.

Correct:

```text
Lexer
  knows syntax

Parser
  knows grammar

Semantic Analyzer
  knows language rules

IR
  knows executable operations

Backend
  knows target architecture

VM
  knows runtime execution
```

This separation is one of the most important lessons in the project.

---

# 140. Final Compiler Dependency Graph

```text
                    Source Code
                        |
                        v
                  +-----------+
                  |   Lexer   |
                  +-----------+
                        |
                        v
                     Tokens
                        |
                        v
                  +-----------+
                  |  Parser   |
                  +-----------+
                        |
                        v
                       AST
                        |
                        v
             +---------------------+
             | Semantic Analyzer   |
             +---------------------+
                 |             |
                 v             v
             Symbols        Types
                 \             /
                  \           /
                   v         v
                    Typed AST
                        |
                        v
                  +-----------+
                  | IR Builder|
                  +-----------+
                        |
                        v
                       IR
                        |
                        v
                  +-----------+
                  | Optimizer |
                  +-----------+
                        |
                        v
                 Optimized IR
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
      Bytecode        JVM           WASM
          |
          v
       Verifier
          |
          v
       Nova VM
          |
          v
       Program
```

---

# 141. Final Recommended Package Architecture

Use this as the long-term package structure:

```text
com.nova.compiler

├── api
│   ├── Compiler.java
│   ├── CompilationResult.java
│   └── CompilerConfiguration.java
│
├── source
│   ├── Source.java
│   ├── SourceFile.java
│   └── SourceLocation.java
│
├── diagnostics
│   ├── Diagnostic.java
│   ├── DiagnosticSeverity.java
│   └── DiagnosticReporter.java
│
├── lexer
│   ├── Lexer.java
│   ├── NovaLexer.java
│   ├── Token.java
│   └── TokenType.java
│
├── parser
│   ├── Parser.java
│   ├── NovaParser.java
│   └── ParseException.java
│
├── ast
│   ├── AstNode.java
│   ├── Expression.java
│   ├── Statement.java
│   └── ...
│
├── semantic
│   ├── SemanticAnalyzer.java
│   ├── Scope.java
│   ├── Symbol.java
│   └── SymbolTable.java
│
├── types
│   ├── Type.java
│   ├── IntType.java
│   ├── BoolType.java
│   ├── StringType.java
│   └── FunctionType.java
│
├── ir
│   ├── IrProgram.java
│   ├── IrFunction.java
│   ├── BasicBlock.java
│   ├── IrInstruction.java
│   └── Opcode.java
│
├── optimizer
│   ├── OptimizationPass.java
│   ├── PassManager.java
│   ├── ConstantFoldingPass.java
│   └── DeadCodeEliminationPass.java
│
├── backend
│   ├── Backend.java
│   ├── nova
│   ├── jvm
│   └── wasm
│
├── bytecode
│   ├── BytecodeProgram.java
│   ├── BytecodeInstruction.java
│   └── BytecodeOpcode.java
│
├── runtime
│   ├── VirtualMachine.java
│   ├── StackFrame.java
│   ├── RuntimeValue.java
│   ├── Heap.java
│   └── GarbageCollector.java
│
├── cli
│   └── NovaCli.java
│
└── tooling
    ├── AstPrinter.java
    ├── IrPrinter.java
    └── Formatter.java
```

---

# 142. The Most Important Implementation Rule

Build vertically, not horizontally.

Do not spend two weeks implementing the entire AST before running anything.

Instead:

```text
Version 1

10 + 20
   |
Lexer
   |
Parser
   |
AST
   |
Evaluator
   |
30
```

Then:

```text
Version 2

let x = 10;
x + 20;
```

Then:

```text
Version 3

if (...)
```

Then:

```text
Version 4

while (...)
```

Then:

```text
Version 5

functions
```

Then:

```text
Version 6

IR
```

Then:

```text
Version 7

bytecode
```

Then:

```text
Version 8

VM
```

Then:

```text
Version 9

optimization
```

This keeps every stage executable and testable.

---

# 143. Final Learning Path

If the objective is not merely to make a toy compiler but to understand how a real compiler is designed, follow this sequence:

```text
                    ┌──────────────────────┐
                    │ Programming Language │
                    │       Design         │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Lexer / Tokenization │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Parsing / Grammar    │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ AST                  │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Semantic Analysis    │
                    │ Scope + Types        │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Intermediate         │
                    │ Representation       │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ CFG + Optimization   │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Bytecode Generation  │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ Virtual Machine      │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ GC + Runtime         │
                    └──────────┬───────────┘
                               |
                               v
                    ┌──────────────────────┐
                    │ JVM / WASM / Native  │
                    │ Backends             │
                    └──────────────────────┘
```

The core architecture to remember is:

```text
Language
   ↓
Lexer
   ↓
Parser
   ↓
AST
   ↓
Semantic Analysis
   ↓
Typed AST
   ↓
IR
   ↓
Optimization
   ↓
Backend
   ↓
Executable
```

And the key scalability boundary is:

```text
                 FRONTEND
                    |
                    v
                  AST
                    |
                    v
              SEMANTIC MODEL
                    |
                    v
                   IR
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
      VM          JVM         WASM
```

If these boundaries remain clean, Nova can evolve from a small educational compiler into a genuinely sophisticated compiler project without requiring the entire codebase to be rewritten.
