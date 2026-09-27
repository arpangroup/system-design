# Build a Python-Like Dynamically Typed Compiler From Scratch in Java

## Goal

Build a scalable Python-like programming language in Java 21 with:

- No variable type declarations
- Dynamic typing
- Indentation-based blocks
- Python-like syntax
- Variables and expressions
- `if / elif / else`
- `while`
- `for`
- Functions and closures
- Lists and dictionaries
- Classes and objects
- Exceptions
- Modules/imports
- Bytecode
- A stack-based virtual machine
- Runtime objects
- Built-in functions
- REPL
- Diagnostics
- Testing and optimization

Example:

```python
name = "Arpan"
age = 32

if age >= 18:
    print("Adult")
else:
    print("Minor")
```

There is no:

```text
int age
String name
boolean active
```

The runtime determines the value's type.

---

# 1. Architecture

```text
Source Code
    |
    v
  Lexer
    |
  Tokens
    |
    v
  Parser
    |
   AST
    |
    v
Semantic Analysis
    |
Validated AST
    |
    v
Bytecode Compiler
    |
 Bytecode
    |
    v
Optimizer
    |
 Bytecode
    |
    v
Virtual Machine
    |
    v
Runtime Object Model
```

Recommended implementation path:

```text
AST Interpreter
      ↓
Bytecode Compiler
      ↓
Bytecode VM
      ↓
Optimizer
      ↓
Optional JIT
```

Starting with an AST interpreter gives you a simple correctness reference before introducing bytecode.

---

# 2. Project Structure

```text
pyra/
├── pom.xml
├── README.md
├── docs/
│   ├── language-spec.md
│   ├── grammar.md
│   ├── bytecode.md
│   └── runtime.md
└── src/
    ├── main/java/com/pyra/
    │   ├── api/
    │   ├── source/
    │   ├── diagnostics/
    │   ├── lexer/
    │   ├── parser/
    │   ├── ast/
    │   ├── semantic/
    │   ├── compiler/
    │   ├── bytecode/
    │   ├── vm/
    │   ├── runtime/
    │   ├── objects/
    │   ├── builtins/
    │   ├── modules/
    │   ├── debugger/
    │   └── cli/
    └── test/java/com/pyra/
        ├── lexer/
        ├── parser/
        ├── compiler/
        ├── vm/
        └── runtime/
```

Keep the existing typed compiler document unchanged. This is a separate architecture.

---

# 3. Maven

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.pyra</groupId>
    <artifactId>pyra-language</artifactId>
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

# 4. Language Philosophy

The language is dynamically typed.

This is valid:

```python
x = 10
x = "hello"
x = [1, 2, 3]
```

Conceptually:

```text
x -> IntegerObject(10)

x -> StringObject("hello")

x -> ListObject(...)
```

The variable itself does not have a permanent type.

The runtime value does.

---

# 5. Runtime Types

Initial runtime types:

```text
None
Boolean
Integer
Float
String
List
Dictionary
Function
NativeFunction
Class
Instance
Module
Exception
```

Future:

```text
Tuple
Set
Range
Iterator
Generator
File
```

---

# 6. Basic Syntax

Variables:

```python
name = "Arpan"
age = 32
active = true
```

Expressions:

```python
total = price * quantity
```

Conditional:

```python
if age >= 18:
    print("Adult")
```

Loop:

```python
while x < 10:
    x = x + 1
```

Function:

```python
def add(a, b):
    return a + b
```

---

# 7. Truthiness

Use Python-like truthiness:

```text
None  -> false
false -> false
0     -> false
0.0   -> false
""    -> false
[]    -> false
{}    -> false
other -> true
```

Centralize it:

```java
public interface Truthiness {
    boolean isTruthy(PyObject value);
}
```

Do not duplicate truthiness rules in the parser and VM.

---

# 8. Lexer

The lexer converts characters into tokens.

Input:

```python
x = 10
```

Output:

```text
IDENTIFIER(x)
EQUAL
INTEGER(10)
NEWLINE
EOF
```

---

# 9. Token Types

```java
public enum TokenType {

    NEWLINE,
    INDENT,
    DEDENT,
    EOF,

    IDENTIFIER,
    INTEGER,
    FLOAT,
    STRING,

    IF,
    ELIF,
    ELSE,

    WHILE,
    FOR,
    IN,

    DEF,
    RETURN,

    CLASS,

    IMPORT,
    FROM,
    AS,

    TRY,
    EXCEPT,
    FINALLY,
    RAISE,

    PASS,
    BREAK,
    CONTINUE,

    TRUE,
    FALSE,
    NONE,

    AND,
    OR,
    NOT,
    IS,

    PLUS,
    MINUS,
    STAR,
    SLASH,
    DOUBLE_SLASH,
    PERCENT,
    POWER,

    EQUAL,
    EQUAL_EQUAL,
    NOT_EQUAL,
    LESS,
    LESS_EQUAL,
    GREATER,
    GREATER_EQUAL,

    PLUS_EQUAL,
    MINUS_EQUAL,
    STAR_EQUAL,
    SLASH_EQUAL,

    LEFT_PAREN,
    RIGHT_PAREN,
    LEFT_BRACKET,
    RIGHT_BRACKET,
    LEFT_BRACE,
    RIGHT_BRACE,

    COMMA,
    DOT,
    COLON,

    ARROW
}
```

---

# 10. INDENT and DEDENT

Indentation is part of the language grammar.

Input:

```python
if x > 10:
    print(x)
    print("large")

print("done")
```

Tokens conceptually become:

```text
IF
IDENTIFIER
GREATER
INTEGER
COLON
NEWLINE

INDENT

IDENTIFIER(print)
...
NEWLINE

IDENTIFIER(print)
...
NEWLINE

DEDENT

IDENTIFIER(print)
...
```

Maintain:

```java
Deque<Integer> indentationStack =
        new ArrayDeque<>();
```

Initialize:

```text
[0]
```

For indentation:

```text
0
4
8
4
0
```

produce:

```text
0  -> nothing
4  -> INDENT
8  -> INDENT
4  -> DEDENT
0  -> DEDENT
```

Recommended initial rule:

```text
Only spaces are allowed for indentation.
```

---

# 11. Comments

Support:

```python
# comment
```

The lexer ignores everything after `#` until the newline.

---

# 12. Source Location

Every token should contain:

```java
public record SourceLocation(
        int offset,
        int line,
        int column
) {}
```

Error:

```text
program.pyra:4:9

if x >:
        ^
SyntaxError: expected expression
```

---

# 13. AST

Use immutable AST nodes.

```java
public sealed interface AstNode
        permits Program,
                Statement,
                Expression {

    SourceLocation location();
}
```

Program:

```java
public record Program(
        List<Statement> statements,
        SourceLocation location
) implements AstNode {}
```

---

# 14. Statement Hierarchy

```java
public sealed interface Statement
        extends AstNode
        permits AssignmentStatement,
                ExpressionStatement,
                IfStatement,
                WhileStatement,
                ForStatement,
                FunctionDeclaration,
                ReturnStatement,
                ClassDeclaration,
                ImportStatement,
                TryStatement,
                PassStatement,
                BreakStatement,
                ContinueStatement {
}
```

---

# 15. Expression Hierarchy

```java
public sealed interface Expression
        extends AstNode
        permits LiteralExpression,
                VariableExpression,
                BinaryExpression,
                UnaryExpression,
                CallExpression,
                AttributeExpression,
                IndexExpression,
                ListExpression,
                DictionaryExpression,
                LambdaExpression {
}
```

---

# 16. Literals

```java
public record LiteralExpression(
        Object value,
        SourceLocation location
) implements Expression {}
```

Supports:

```python
10
10.5
"hello"
true
false
None
```

---

# 17. Variables

```java
public record VariableExpression(
        String name,
        SourceLocation location
) implements Expression {}
```

---

# 18. Assignment

```java
public record AssignmentStatement(
        String name,
        Expression value,
        SourceLocation location
) implements Statement {}
```

For:

```python
x = 10
```

AST:

```text
Assignment
  x
  Integer(10)
```

---

# 19. Binary Expressions

```java
public record BinaryExpression(
        Expression left,
        BinaryOperator operator,
        Expression right,
        SourceLocation location
) implements Expression {}
```

```java
public enum BinaryOperator {

    ADD,
    SUBTRACT,
    MULTIPLY,
    DIVIDE,
    MODULO,
    POWER,

    EQUAL,
    NOT_EQUAL,
    LESS,
    LESS_EQUAL,
    GREATER,
    GREATER_EQUAL,

    AND,
    OR,

    IN,
    NOT_IN,

    IS,
    IS_NOT
}
```

---

# 20. Operator Precedence

Use:

```text
or
and
not
comparison
+
-
*
/
%
**
unary
primary
```

Therefore:

```python
a + b * c
```

becomes:

```text
      +
     /     a   *
       /       b   c
```

not:

```text
      *
     /     +   c
   /   a   b
```

---

# 21. Parser

Start with recursive descent.

```java
public interface Parser {
    Program parse();
}
```

Main flow:

```text
parse()
  |
  +-- statement()
       |
       +-- if
       +-- while
       +-- for
       +-- def
       +-- class
       +-- return
       +-- assignment
       +-- expression
```

---

# 22. Grammar

```text
program
    ::= statement* EOF

statement
    ::= assignment
     | expressionStatement
     | ifStatement
     | whileStatement
     | forStatement
     | functionDeclaration
     | returnStatement
     | classDeclaration
     | tryStatement
     | importStatement
     | passStatement
     | breakStatement
     | continueStatement
```

---

# 23. IF Grammar

```text
ifStatement
    ::= IF expression ":" NEWLINE
        INDENT statement*
        DEDENT
        elifClause*
        elseClause?

elifClause
    ::= ELIF expression ":" NEWLINE
        INDENT statement*
        DEDENT

elseClause
    ::= ELSE ":" NEWLINE
        INDENT statement*
        DEDENT
```

Example:

```python
if age >= 18:
    print("adult")
elif age >= 13:
    print("teen")
else:
    print("child")
```

---

# 24. IF AST

```java
public record IfStatement(
        Expression condition,
        List<Statement> thenBranch,
        List<ElifBranch> elifBranches,
        List<Statement> elseBranch,
        SourceLocation location
) implements Statement {}
```

---

# 25. WHILE

```python
while x < 10:
    print(x)
    x = x + 1
```

AST:

```text
While
 |
 +-- condition
 |
 +-- body
```

---

# 26. FOR

Syntax:

```python
for item in items:
    print(item)
```

Grammar:

```text
forStatement
    ::= FOR IDENTIFIER IN expression
        ":" NEWLINE
        INDENT statement*
        DEDENT
```

AST:

```java
public record ForStatement(
        String variable,
        Expression iterable,
        List<Statement> body,
        SourceLocation location
) implements Statement {}
```

---

# 27. Functions

Syntax:

```python
def add(a, b):
    return a + b
```

No parameter types.

No return type.

AST:

```java
public record FunctionDeclaration(
        String name,
        List<String> parameters,
        List<Statement> body,
        SourceLocation location
) implements Statement {}
```

---

# 28. Function Calls

```python
add(10, 20)
```

AST:

```java
public record CallExpression(
        Expression callee,
        List<Expression> arguments,
        SourceLocation location
) implements Expression {}
```

This also supports:

```python
obj.method()
factory()()
```

---

# 29. Return

```python
return a + b
```

AST:

```java
public record ReturnStatement(
        Expression value,
        SourceLocation location
) implements Statement {}
```

A bare:

```python
return
```

returns `None`.

---

# 30. Lists

```python
numbers = [1, 2, 3]
```

AST:

```java
public record ListExpression(
        List<Expression> elements,
        SourceLocation location
) implements Expression {}
```

---

# 31. Dictionaries

```python
user = {
    "name": "Arpan",
    "age": 32
}
```

AST:

```java
public record DictionaryExpression(
        List<Entry> entries,
        SourceLocation location
) implements Expression {}
```

---

# 32. Indexing

```python
numbers[0]
```

AST:

```java
public record IndexExpression(
        Expression target,
        Expression index,
        SourceLocation location
) implements Expression {}
```

---

# 33. Attribute Access

```python
user.name
```

AST:

```java
public record AttributeExpression(
        Expression target,
        String attribute,
        SourceLocation location
) implements Expression {}
```

This later enables:

```python
user.address.city
```

---

# 34. Runtime Object Model

Do not use raw Java `Object` everywhere.

Create:

```java
public interface PyObject {

    PyType type();

    boolean isTruthy();

    PyObject getAttribute(String name);

    void setAttribute(
            String name,
            PyObject value
    );
}
```

---

# 35. Runtime Types

```java
public enum PyType {

    NONE,
    BOOLEAN,
    INTEGER,
    FLOAT,
    STRING,
    LIST,
    DICTIONARY,
    FUNCTION,
    NATIVE_FUNCTION,
    CLASS,
    INSTANCE,
    MODULE,
    EXCEPTION
}
```

---

# 36. Integer Runtime Object

```java
public final class PyInteger
        implements PyObject {

    private final long value;

    public PyInteger(long value) {
        this.value = value;
    }

    public long value() {
        return value;
    }

    @Override
    public PyType type() {
        return PyType.INTEGER;
    }

    @Override
    public boolean isTruthy() {
        return value != 0;
    }
}
```

---

# 37. String Runtime Object

```java
public final class PyString
        implements PyObject {

    private final String value;

    public PyString(String value) {
        this.value = value;
    }

    public String value() {
        return value;
    }

    @Override
    public PyType type() {
        return PyType.STRING;
    }

    @Override
    public boolean isTruthy() {
        return !value.isEmpty();
    }
}
```

---

# 38. Dynamic Operations

For:

```python
x + y
```

the compiler should not assume that `x` and `y` are integers.

Generate a generic:

```text
ADD
```

instruction.

The VM delegates to:

```java
public interface RuntimeOperations {

    PyObject add(
            PyObject left,
            PyObject right
    );

    PyObject subtract(
            PyObject left,
            PyObject right
    );

    PyObject multiply(
            PyObject left,
            PyObject right
    );

    PyObject divide(
            PyObject left,
            PyObject right
    );
}
```

This supports:

```text
Integer + Integer
Float + Integer
String + String
List + List
```

without putting all type logic inside the VM.

---

# 39. Avoid Giant instanceof Chains

Avoid spreading code like this throughout the VM:

```java
if (left instanceof PyInteger &&
    right instanceof PyInteger) {
    ...
} else if (left instanceof PyString &&
           right instanceof PyString) {
    ...
}
```

Instead centralize dynamic dispatch in:

```text
RuntimeOperations
```

The VM should primarily execute instructions.

---

# 40. Environments

Variables require scopes.

```java
public final class Environment {

    private final Environment parent;

    private final Map<String, PyObject> values =
            new HashMap<>();

    public Environment(Environment parent) {
        this.parent = parent;
    }

    public void define(
            String name,
            PyObject value
    ) {
        values.put(name, value);
    }

    public PyObject get(String name) {

        if (values.containsKey(name)) {
            return values.get(name);
        }

        if (parent != null) {
            return parent.get(name);
        }

        throw new PyNameError(
                "name '" + name + "' is not defined"
        );
    }
}
```

---

# 41. Scope Example

```python
x = 10

def test():
    x = 20
    print(x)

test()
print(x)
```

Scopes:

```text
Global
  x = 10

Function
  x = 20
```

Output:

```text
20
10
```

---

# 42. Closures

Support:

```python
def outer():
    x = 10

    def inner():
        return x

    return inner
```

A function must retain its defining environment.

```java
public final class PyFunction
        implements PyObject {

    private final FunctionCode code;
    private final Environment closure;

    public PyFunction(
            FunctionCode code,
            Environment closure
    ) {
        this.code = code;
        this.closure = closure;
    }
}
```

---

# 43. Closure Cells

For captured variables:

```java
public final class Cell {

    private PyObject value;

    public Cell(PyObject value) {
        this.value = value;
    }

    public PyObject get() {
        return value;
    }

    public void set(PyObject value) {
        this.value = value;
    }
}
```

Conceptually:

```text
outer frame
    |
    x
    |
    v
 Cell
    |
    v
 inner function
```

---

# 44. AST Interpreter

Before bytecode, implement:

```java
public interface AstInterpreter {

    PyObject execute(
            Program program
    );
}
```

This gives you:

```text
Source
 ↓
AST
 ↓
Interpreter
```

It becomes a reference implementation for the bytecode VM.

---

# 45. Bytecode

Example:

```python
x = 10
y = 20
print(x + y)
```

Possible bytecode:

```text
LOAD_CONST 10
STORE_NAME x

LOAD_CONST 20
STORE_NAME y

LOAD_NAME print
LOAD_NAME x
LOAD_NAME y
ADD
CALL 1
POP
```

---

# 46. Bytecode Opcodes

```java
public enum OpCode {

    LOAD_CONST,
    LOAD_NAME,
    STORE_NAME,

    LOAD_LOCAL,
    STORE_LOCAL,

    LOAD_GLOBAL,
    STORE_GLOBAL,

    LOAD_FREE,
    STORE_FREE,

    POP,

    ADD,
    SUBTRACT,
    MULTIPLY,
    DIVIDE,
    MODULO,
    POWER,

    EQUAL,
    NOT_EQUAL,
    LESS,
    LESS_EQUAL,
    GREATER,
    GREATER_EQUAL,

    AND,
    OR,
    NOT,

    JUMP,
    JUMP_IF_FALSE,

    CALL,
    RETURN,

    BUILD_LIST,
    BUILD_MAP,

    GET_INDEX,
    SET_INDEX,

    GET_ATTRIBUTE,
    SET_ATTRIBUTE,

    MAKE_FUNCTION,

    ITER_START,
    ITER_NEXT,

    HALT
}
```

---

# 47. Instruction

```java
public record Instruction(
        OpCode opcode,
        Object operand
) {}
```

Example:

```text
LOAD_CONST 10
```

can be represented by:

```java
new Instruction(
    OpCode.LOAD_CONST,
    10
);
```

---

# 48. Constant Pool

Use a constant pool:

```java
public final class Chunk {

    private final List<Instruction> code =
            new ArrayList<>();

    private final List<PyObject> constants =
            new ArrayList<>();

    public int addConstant(
            PyObject value
    ) {
        constants.add(value);
        return constants.size() - 1;
    }
}
```

Then:

```text
LOAD_CONST 0
```

means:

```text
constant[0]
```

---

# 49. Bytecode Compiler

```java
public interface BytecodeCompiler {

    BytecodeModule compile(
            Program program
    );
}
```

Literal:

```python
10
```

becomes:

```text
LOAD_CONST 10
```

Assignment:

```python
x = 10
```

becomes:

```text
LOAD_CONST 10
STORE_NAME x
```

Expression:

```python
x + y
```

becomes:

```text
LOAD_NAME x
LOAD_NAME y
ADD
```

---

# 50. Stack-Based VM

Maintain:

```java
Deque<PyObject> stack =
        new ArrayDeque<>();
```

For:

```python
10 + 20
```

execution:

```text
LOAD_CONST 10

stack:
[10]

LOAD_CONST 20

stack:
[10, 20]

ADD

stack:
[30]
```

---

# 51. VM Loop

```java
public void run() {

    while (!halted()) {

        Instruction instruction =
                fetch();

        execute(instruction);
    }
}
```

Initial dispatcher:

```java
private void execute(
        Instruction instruction
) {

    switch (instruction.opcode()) {

        case LOAD_CONST ->
            loadConstant(instruction);

        case LOAD_NAME ->
            loadName(instruction);

        case STORE_NAME ->
            storeName(instruction);

        case ADD ->
            add();

        case CALL ->
            call(instruction);

        case RETURN ->
            returnFromFunction();

        case JUMP ->
            jump(instruction);

        case JUMP_IF_FALSE ->
            jumpIfFalse(instruction);

        case HALT ->
            halted = true;
    }
}
```

Later, opcode handlers can be split if the switch becomes too large.

---

# 52. Function Call Frames

A function needs:

```text
instruction pointer
local environment
closure
function code
```

```java
public final class CallFrame {

    private final BytecodeFunction function;
    private final Environment environment;

    private int instructionPointer;

    public CallFrame(
            BytecodeFunction function,
            Environment environment
    ) {
        this.function = function;
        this.environment = environment;
    }
}
```

VM:

```java
Deque<CallFrame> callStack =
        new ArrayDeque<>();
```

---

# 53. CALL

Source:

```python
add(10, 20)
```

Bytecode:

```text
LOAD_NAME add
LOAD_CONST 10
LOAD_CONST 20
CALL 2
```

Conceptually:

```text
stack:
[add, 10, 20]

CALL 2

pop arguments
pop callable
create frame
execute function
push return value
```

---

# 54. RETURN

Source:

```python
return x
```

Bytecode:

```text
LOAD_NAME x
RETURN
```

VM:

```text
value = stack.pop()
frame = callStack.pop()
caller.stack.push(value)
```

---

# 55. IF Compilation

Source:

```python
if x > 10:
    print("large")
else:
    print("small")
```

Conceptual bytecode:

```text
LOAD_NAME x
LOAD_CONST 10
GREATER

JUMP_IF_FALSE elseLabel

LOAD_NAME print
LOAD_CONST "large"
CALL 1
JUMP endLabel

elseLabel:

LOAD_NAME print
LOAD_CONST "small"
CALL 1

endLabel:
```

Use jump patching because labels are not always known while the compiler is generating instructions.

---

# 56. Jump Patching

```java
int jump =
        emit(
            OpCode.JUMP_IF_FALSE,
            -1
        );
```

Compile the branch.

Then:

```java
patchJump(
        jump,
        currentInstruction()
);
```

This technique is fundamental for control-flow bytecode.

---

# 57. WHILE Compilation

Source:

```python
while x < 10:
    print(x)
    x = x + 1
```

Bytecode conceptually:

```text
loopStart:

LOAD_NAME x
LOAD_CONST 10
LESS

JUMP_IF_FALSE loopEnd

LOAD_NAME print
LOAD_NAME x
CALL 1

LOAD_NAME x
LOAD_CONST 1
ADD
STORE_NAME x

JUMP loopStart

loopEnd:
```

---

# 58. FOR Compilation

Source:

```python
for item in items:
    print(item)
```

Compile through iterator operations:

```text
LOAD_NAME items
ITER_START

loopStart:

ITER_NEXT
JUMP_IF_FALSE loopEnd

STORE_NAME item

LOAD_NAME print
LOAD_NAME item
CALL 1

JUMP loopStart

loopEnd:
```

This allows lists, strings, ranges, generators, and custom iterables to share an iteration protocol.

---

# 59. Local Variable Optimization

Instead of map-based:

```text
LOAD_NAME x
```

functions can use:

```text
LOAD_LOCAL 0
```

For:

```python
def add(a, b):
    c = a + b
    return c
```

slots:

```text
a -> 0
b -> 1
c -> 2
```

Bytecode:

```text
LOAD_LOCAL 0
LOAD_LOCAL 1
ADD
STORE_LOCAL 2
LOAD_LOCAL 2
RETURN
```

This is a major VM optimization.

---

# 60. Global / Local / Closure Names

Use separate operations:

```text
LOAD_GLOBAL
STORE_GLOBAL

LOAD_LOCAL
STORE_LOCAL

LOAD_FREE
STORE_FREE
```

The semantic/compiler pass can determine whether a name is:

```text
global
local
captured
```

without imposing static types.

---

# 61. Built-ins

Initial built-ins:

```text
print()
len()
type()
range()
str()
int()
float()
bool()
```

Interface:

```java
public interface CallableObject
        extends PyObject {

    PyObject call(
            List<PyObject> arguments
    );
}
```

Native function:

```java
public final class NativeFunction
        implements CallableObject {

    private final Function<
            List<PyObject>,
            PyObject
        > implementation;

    public NativeFunction(
            Function<List<PyObject>, PyObject>
                    implementation
    ) {
        this.implementation =
                implementation;
    }

    @Override
    public PyObject call(
            List<PyObject> arguments
    ) {
        return implementation.apply(arguments);
    }
}
```

---

# 62. Global Environment

Initialize:

```java
Environment globals =
        new Environment(null);
```

Register:

```text
print
len
range
str
int
float
bool
```

Then execute user code inside this environment.

---

# 63. Lists

Source:

```python
numbers = [1, 2, 3]
```

Bytecode:

```text
LOAD_CONST 1
LOAD_CONST 2
LOAD_CONST 3
BUILD_LIST 3
STORE_NAME numbers
```

Runtime:

```java
public final class PyList
        implements PyObject {

    private final List<PyObject> values;

    public PyList(
            List<PyObject> values
    ) {
        this.values = new ArrayList<>(values);
    }
}
```

---

# 64. Dictionaries

Source:

```python
user = {
    "name": "Arpan",
    "age": 32
}
```

Bytecode:

```text
LOAD_CONST "name"
LOAD_CONST "Arpan"

LOAD_CONST "age"
LOAD_CONST 32

BUILD_MAP 2
STORE_NAME user
```

Runtime can initially use:

```java
Map<PyObject, PyObject>
```

with proper equality/hash semantics.

---

# 65. Classes

Python-like syntax:

```python
class User:

    def __init__(self, name):
        self.name = name

    def greet(self):
        print(self.name)
```

Initial class model:

```text
ClassDeclaration
      ↓
PyClass
      ↓
PyInstance
```

---

# 66. Class AST

```java
public record ClassDeclaration(
        String name,
        List<FunctionDeclaration> methods,
        SourceLocation location
) implements Statement {}
```

---

# 67. Instance

```java
public final class PyInstance
        implements PyObject {

    private final PyClass klass;

    private final Map<String, PyObject> fields =
            new HashMap<>();

    public PyInstance(PyClass klass) {
        this.klass = klass;
    }
}
```

Attribute lookup:

```text
instance field
    ↓
class attribute
    ↓
method
    ↓
error
```

---

# 68. Method Binding

For:

```python
user.greet()
```

the runtime should produce a bound method:

```text
user.greet
    ↓
BoundMethod
    ↓
greet(user)
```

`self` is therefore a runtime binding concern rather than a special compiler type.

---

# 69. Exceptions

Syntax:

```python
try:
    risky()
except:
    print("failed")
finally:
    cleanup()
```

Runtime exception hierarchy:

```text
PyException
 ├── PyRuntimeError
 ├── PyValueError
 ├── PyTypeError
 ├── PyNameError
 └── PyZeroDivisionError
```

Do not expose raw Java exceptions as the language's user-facing errors.

---

# 70. Exception Unwinding

If:

```text
c()
 ↓
b()
 ↓
a()
 ↓
main
```

and `c()` raises an exception, the VM should unwind:

```text
c
 ↓
b
 ↓
a
 ↓
main
```

until it finds an exception handler.

This is a core VM feature.

---

# 71. Modules

Support:

```python
import math
```

and:

```python
from math import sqrt
```

Interface:

```java
public interface ModuleLoader {

    PyModule load(String name);
}
```

Initial loading flow:

```text
module name
    ↓
file
    ↓
lexer
    ↓
parser
    ↓
compiler
    ↓
VM
    ↓
module object
```

Cache loaded modules.

---

# 72. REPL

Command:

```bash
pyra
```

Example:

```text
Pyra 0.1

>>> x = 10
>>> x + 20
30
>>> print("hello")
hello
```

The global environment must survive between commands.

---

# 73. Compiler API

```java
public final class PyraCompiler {

    private final Lexer lexer;
    private final Parser parser;
    private final BytecodeCompiler compiler;

    public CompiledModule compile(
            String source
    ) {

        List<Token> tokens =
                lexer.tokenize(source);

        Program program =
                parser.parse(tokens);

        return compiler.compile(program);
    }
}
```

Execution:

```java
CompiledModule module =
        compiler.compile(source);

vm.execute(module);
```

---

# 74. Semantic Analysis

Dynamic typing does not mean "no semantic analysis".

The semantic phase should detect things such as:

```text
break outside loop
continue outside loop
return outside function
invalid assignment target
duplicate declarations where forbidden
invalid syntax/structure
```

It does not need to require:

```text
x is always an integer
```

---

# 75. Scope Analysis

Track:

```text
global
function
class
closure
```

For each scope track:

```text
defined names
used names
captured names
```

This allows the compiler to choose:

```text
GLOBAL
LOCAL
FREE
```

bytecode operations.

---

# 76. Diagnostics

```java
public record Diagnostic(
        Severity severity,
        String code,
        String message,
        SourceLocation location
) {}
```

Examples:

```text
PY001 unexpected token
PY002 expected expression
PY003 expected ':'
PY004 indentation error
PY005 invalid assignment
PY006 name not found
PY007 return outside function
PY008 break outside loop
PY009 continue outside loop
```

---

# 77. Error Recovery

Input:

```python
x = 10

if x >:
    print(x)

y = 20
```

The parser should ideally report:

```text
line 3:
expected expression after '>'
```

and recover sufficiently to continue parsing later statements.

For a compiler CLI, fail-fast mode can also be supported.

---

# 78. AST Debugging

Add:

```bash
pyra --dump-ast program.pyra
```

Output:

```text
Program
  Assignment(x)
    Literal(10)

  Expression
    Binary(ADD)
      Variable(x)
      Literal(20)
```

This is extremely useful while developing the parser.

---

# 79. Bytecode Debugging

Add:

```bash
pyra --dump-bytecode program.pyra
```

Example:

```text
0 LOAD_CONST 0
1 STORE_GLOBAL x
2 LOAD_GLOBAL x
3 LOAD_CONST 1
4 ADD
5 RETURN
```

---

# 80. Debug Information

Each bytecode instruction can optionally map to source:

```java
public record DebugInfo(
        int instructionOffset,
        SourceLocation location
) {}
```

Runtime errors can then show:

```text
program.pyra:14

result = 10 / value
                 ^

ZeroDivisionError
```

instead of merely reporting a VM instruction number.

---

# 81. Testing Strategy

Test each stage independently:

```text
Lexer
Parser
AST
Semantic Analyzer
Compiler
Bytecode
VM
Runtime
Built-ins
Modules
Exceptions
```

Also create end-to-end tests.

---

# 82. Lexer Tests

Test:

```python
x = 10
```

Expected:

```text
IDENTIFIER
EQUAL
INTEGER
NEWLINE
EOF
```

Indentation:

```python
if x:
    print(x)
```

Expected:

```text
IF
IDENTIFIER
COLON
NEWLINE
INDENT
...
DEDENT
```

---

# 83. Parser Tests

Test:

```python
x = 10
```

```python
if x:
    print(x)
```

```python
while x < 10:
    x = x + 1
```

```python
for item in items:
    print(item)
```

```python
def add(a, b):
    return a + b
```

---

# 84. Runtime Tests

```python
print(10 + 20)
```

Expected:

```text
30
```

Dynamic reassignment:

```python
x = 10
x = "hello"
print(x)
```

Expected:

```text
hello
```

---

# 85. Closure Test

```python
def outer():
    x = 10

    def inner():
        return x

    return inner

f = outer()

print(f())
```

Expected:

```text
10
```

---

# 86. Recursion Test

```python
def factorial(n):
    if n <= 1:
        return 1

    return n * factorial(n - 1)

print(factorial(5))
```

Expected:

```text
120
```

This tests:

```text
functions
frames
recursion
return
arithmetic
conditionals
```

---

# 87. Collection Test

```python
numbers = [1, 2, 3, 4]

sum = 0

for n in numbers:
    sum = sum + n

print(sum)
```

Expected:

```text
10
```

---

# 88. Object Test

```python
class Counter:

    def __init__(self, value):
        self.value = value

    def increment(self):
        self.value = self.value + 1

counter = Counter(10)

counter.increment()

print(counter.value)
```

Expected:

```text
11
```

---

# 89. Differential Testing

Run every program through two implementations:

```text
Source
  |
  +----> AST Interpreter
  |
  +----> Bytecode Compiler -> VM
```

Compare:

```text
return value
stdout
exception type
exception message
```

This is one of the best ways to validate the bytecode VM.

---

# 90. Constant Folding

Source:

```python
x = 10 + 20 * 2
```

The compiler can safely transform the pure constant expression into:

```text
LOAD_CONST 50
STORE_NAME x
```

instead of:

```text
LOAD_CONST 10
LOAD_CONST 20
LOAD_CONST 2
MULTIPLY
ADD
STORE_NAME x
```

Keep optimization in a dedicated phase.

---

# 91. Peephole Optimization

Potential transformations:

```text
JUMP label
label:
```

to:

```text
nothing
```

and other safe instruction patterns.

Do not mix these optimizations into parsing.

---

# 92. Bytecode Verification

Before execution, verify:

```text
opcode is valid
constant index is valid
jump target is valid
local slot is valid
function metadata is valid
```

This makes the VM more robust.

---

# 93. Memory Management

Initially use the JVM garbage collector.

Architecture:

```text
Pyra runtime objects
        ↓
Java heap
        ↓
JVM GC
```

Do not implement a custom garbage collector while simultaneously building the lexer, parser, compiler, and VM.

Later options:

```text
reference counting
mark-and-sweep
generational GC
```

---

# 94. Performance Optimization

Start with correctness.

Then optimize:

```text
1. local slots
2. constant folding
3. dead code
4. jump simplification
5. specialized integer operations
6. inline caches
7. bytecode dispatch optimization
8. profiling
9. optional JIT
```

---

# 95. Inline Caching

For:

```python
x + y
```

the VM initially performs dynamic dispatch.

After repeatedly observing:

```text
Integer + Integer
```

the runtime can cache:

```text
Integer + Integer -> IntegerAdd
```

Future executions use the fast path.

If the types change:

```python
x = "hello"
```

the runtime falls back to generic dispatch.

---

# 96. Optional Type Hints

Although the core language is dynamic, later you can support:

```python
def add(a: int, b: int):
    return a + b
```

Do not make type hints mandatory.

Possible modes:

```text
dynamic
optional checking
strict checking
```

Type hints can initially be metadata for tooling rather than runtime requirements.

---

# 97. Optional JIT

Future architecture:

```text
Source
  ↓
Lexer
  ↓
Parser
  ↓
AST
  ↓
Bytecode
  ↓
Profiler
  ↓
Hot Function Detection
  ↓
Specialization
  ↓
Machine Code
```

For example:

```text
function executed 100,000 times
        ↓
mark hot
        ↓
compile optimized version
```

This should be a very late project phase.

---

# 98. Debugger

Add:

```bash
pyra debug program.pyra
```

Commands:

```text
break
run
continue
step
next
locals
stack
print
```

The existing VM already maintains:

```text
instruction pointer
call frames
locals
```

so debugger support fits naturally.

---

# 99. Profiling

Add:

```bash
pyra --profile program.pyra
```

Track:

```text
instructions executed
function calls
opcode frequency
allocations
exceptions
execution time
```

Example:

```text
Instructions: 1,204,332
Function calls: 52,100
Allocations: 18,422
```

Use this information to guide optimization.

---

# 100. Benchmark Suite

Create:

```text
benchmarks/
├── arithmetic.pyra
├── fibonacci.pyra
├── factorial.pyra
├── loops.pyra
├── lists.pyra
├── dictionaries.pyra
├── functions.pyra
└── objects.pyra
```

Compare:

```text
AST interpreter
Bytecode VM
Optimized VM
```

---

# 101. SOLID Architecture

## Single Responsibility

```text
Lexer
    characters -> tokens

Parser
    tokens -> AST

SemanticAnalyzer
    AST -> validated AST

BytecodeCompiler
    AST -> bytecode

VM
    bytecode -> execution

RuntimeOperations
    dynamic language behavior
```

## Open/Closed

Adding:

```text
new builtin
```

should not require rewriting the parser.

Adding:

```text
new optimization rule
```

should not require rewriting the lexer.

## Dependency Inversion

Inject:

```text
RuntimeOperations
ModuleLoader
BuiltinRegistry
DiagnosticReporter
```

rather than constructing everything inside the VM.

---

# 102. Runtime Interfaces

Prefer small interfaces:

```java
public interface CallableObject {

    PyObject call(
            List<PyObject> arguments
    );
}
```

```java
public interface IterableObject {

    PyIterator iterator();
}
```

```java
public interface IndexableObject {

    PyObject getItem(PyObject index);

    void setItem(
            PyObject index,
            PyObject value
    );
}
```

```java
public interface AttributeObject {

    PyObject getAttribute(String name);

    void setAttribute(
            String name,
            PyObject value
    );
}
```

This keeps the runtime extensible.

---

# 103. Complete Example

Source:

```python
def add(a, b):
    return a + b

x = add(10, 20)

if x > 20:
    print("large")
else:
    print("small")
```

Lexer produces tokens.

Parser produces:

```text
Program
 |
 +-- Function(add)
 |     |
 |     +-- parameters
 |     +-- Return
 |           |
 |           +-- Binary(ADD)
 |
 +-- Assignment(x)
 |     |
 |     +-- Call(add)
 |
 +-- If
       |
       +-- Binary(GREATER)
       +-- then
       +-- else
```

Compiler produces conceptually:

```text
MAKE_FUNCTION add
STORE_GLOBAL add

LOAD_GLOBAL add
LOAD_CONST 10
LOAD_CONST 20
CALL 2
STORE_GLOBAL x

LOAD_GLOBAL x
LOAD_CONST 20
GREATER
JUMP_IF_FALSE elseLabel

LOAD_GLOBAL print
LOAD_CONST "large"
CALL 1
JUMP endLabel

elseLabel:

LOAD_GLOBAL print
LOAD_CONST "small"
CALL 1

endLabel
```

The VM executes these instructions and the runtime handles dynamic values.

---

# 104. Milestone Roadmap

Implement in this order.

## Milestone 1 — Project

```text
Java 21
Maven
JUnit
CLI
```

## Milestone 2 — Lexer

```text
identifiers
numbers
strings
operators
NEWLINE
```

## Milestone 3 — Indentation

```text
INDENT
DEDENT
```

## Milestone 4 — Parser

```text
expressions
assignments
```

## Milestone 5 — AST Interpreter

```text
variables
arithmetic
print
```

## Milestone 6 — Control Flow

```text
if
elif
else
while
for
```

## Milestone 7 — Functions

```text
def
return
arguments
recursion
```

## Milestone 8 — Collections

```text
list
dictionary
indexing
attributes
```

## Milestone 9 — Closures

```text
nested functions
captured variables
```

## Milestone 10 — Classes

```text
class
instance
methods
self
```

## Milestone 11 — Exceptions

```text
try
except
finally
raise
```

## Milestone 12 — Modules

```text
import
from
module cache
```

## Milestone 13 — Bytecode

```text
instruction model
constant pool
compiler
```

## Milestone 14 — VM

```text
operand stack
call frames
jumps
function calls
```

## Milestone 15 — Optimization

```text
local slots
constant folding
dead code
jump optimization
```

## Milestone 16 — Tooling

```text
REPL
AST dump
bytecode dump
debugger
profiler
```

## Milestone 17 — Advanced Runtime

```text
inline caches
specialized operations
optional type hints
optional JIT
```

---

# 105. Final Package Architecture

```text
com.pyra

├── api
│   ├── PyraEngine.java
│   └── CompilerConfiguration.java
│
├── source
│   ├── SourceFile.java
│   └── SourceLocation.java
│
├── diagnostics
│   ├── Diagnostic.java
│   └── DiagnosticReporter.java
│
├── lexer
│   ├── Lexer.java
│   ├── Token.java
│   ├── TokenType.java
│   └── IndentationProcessor.java
│
├── parser
│   ├── Parser.java
│   ├── ParserImpl.java
│   ├── ExpressionParser.java
│   └── StatementParser.java
│
├── ast
│   ├── AstNode.java
│   ├── Program.java
│   ├── Statement.java
│   ├── Expression.java
│   └── ...
│
├── semantic
│   ├── SemanticAnalyzer.java
│   ├── Scope.java
│   ├── ScopeResolver.java
│   └── NameResolver.java
│
├── compiler
│   ├── BytecodeCompiler.java
│   ├── ExpressionCompiler.java
│   ├── StatementCompiler.java
│   └── ControlFlowCompiler.java
│
├── bytecode
│   ├── OpCode.java
│   ├── Instruction.java
│   ├── Chunk.java
│   ├── BytecodeFunction.java
│   └── BytecodeModule.java
│
├── vm
│   ├── VirtualMachine.java
│   ├── CallFrame.java
│   └── Stack.java
│
├── runtime
│   ├── RuntimeOperations.java
│   ├── Truthiness.java
│   ├── Environment.java
│   └── Cell.java
│
├── objects
│   ├── PyObject.java
│   ├── PyInteger.java
│   ├── PyFloat.java
│   ├── PyString.java
│   ├── PyList.java
│   ├── PyDictionary.java
│   ├── PyFunction.java
│   ├── PyClass.java
│   ├── PyInstance.java
│   └── ...
│
├── builtins
│   ├── BuiltinRegistry.java
│   ├── PrintFunction.java
│   ├── LenFunction.java
│   └── RangeFunction.java
│
├── modules
│   ├── ModuleLoader.java
│   ├── FileModuleLoader.java
│   └── ModuleCache.java
│
├── debugger
│   ├── Debugger.java
│   ├── Breakpoint.java
│   └── DebugSession.java
│
└── cli
    └── PyraCli.java
```

---

# 106. Final Compiler Flow

The completed project should follow:

```text
                 +----------------+
                 | CLI / REPL     |
                 +-------+--------+
                         |
                         v
                 +---------------+
                 | Source        |
                 +-------+-------+
                         |
                         v
                 +---------------+
                 | Lexer         |
                 | INDENT/DEDENT |
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
                 +---------------+
                 | Semantic      |
                 | Analysis      |
                 +-------+-------+
                         |
                         v
                 Validated AST
                         |
                         v
                 +---------------+
                 | Bytecode      |
                 | Compiler      |
                 +-------+-------+
                         |
                         v
                     Bytecode
                         |
                         v
                 +---------------+
                 | Optimizer     |
                 +-------+-------+
                         |
                         v
                     Bytecode
                         |
                         v
                 +---------------+
                 | Virtual       |
                 | Machine       |
                 +-------+-------+
                         |
              +----------+----------+
              |                     |
              v                     v
        Call Frames             Runtime
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
              Objects          Builtins         Modules
```

---

# 107. Key Design Rule

The most important architectural difference from the original typed compiler is:

```text
Typed compiler:

variable
   ↓
compile-time type
   ↓
typed operation
```

Pyra:

```text
variable
   ↓
runtime value
   ↓
runtime type
   ↓
dynamic operation
```

For:

```python
x = 10
x = "hello"
```

the compiler does not reject the reassignment.

For:

```python
x + y
```

the compiler generates:

```text
LOAD_NAME x
LOAD_NAME y
ADD
```

The VM delegates `ADD` to runtime dispatch.

This cleanly separates:

```text
language syntax
    ↓
compiler
    ↓
bytecode
    ↓
virtual machine
    ↓
dynamic runtime
```

and gives the project a strong foundation for:

```text
closures
classes
exceptions
modules
optimizations
inline caching
optional type hints
JIT compilation
```

---

# 108. Recommended Learning Sequence

Follow this sequence rather than implementing the entire language at once:

```text
1. Design syntax
        ↓
2. Implement lexer
        ↓
3. Implement INDENT/DEDENT
        ↓
4. Implement parser
        ↓
5. Build AST
        ↓
6. Build AST interpreter
        ↓
7. Implement dynamic runtime objects
        ↓
8. Add functions and scopes
        ↓
9. Add closures
        ↓
10. Add collections
        ↓
11. Add classes
        ↓
12. Add exceptions
        ↓
13. Add modules
        ↓
14. Design bytecode
        ↓
15. Build bytecode compiler
        ↓
16. Build stack VM
        ↓
17. Differential-test VM vs interpreter
        ↓
18. Optimize locals and constants
        ↓
19. Add debugger/profiler
        ↓
20. Add inline caches
        ↓
21. Optional type hints
        ↓
22. Optional JIT
```

The resulting project is not merely a parser. It is a complete learning-oriented language implementation:

```text
Python-like language
        +
Dynamic runtime
        +
Compiler
        +
Bytecode
        +
Virtual Machine
        +
Standard Library
        +
Tooling
```
