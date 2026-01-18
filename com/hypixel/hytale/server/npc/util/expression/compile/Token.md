---
description: Architectural reference for Token
---

# Token

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Utility / Type-Safe Enum

## Definition
```java
// Signature
public enum Token implements Supplier<String> {
```

## Architecture & Concepts

The Token enum is the fundamental atomic unit of the server-side NPC expression language. It represents the classified output of the lexical analysis phase (lexing) and serves as the direct input for the parsing phase. Each member of this enum is not merely a label, but a rich, self-describing object that encapsulates all syntactic and semantic metadata required by the parser.

This design is critical for creating a clean, maintainable, and efficient parser. Instead of the parser containing complex conditional logic to determine the properties of a given token (e.g., `if (char == '+')`), it can directly query the Token instance itself (e.g., `token.getPrecedence()`, `token.isOperator()`).

Key architectural features include:

*   **Operator Precedence:** The `precedence` field is central to implementing standard parsing algorithms like Shunting-yard or Pratt parsing, correctly handling the order of operations (e.g., multiplication before addition).
*   **Flag-Based Properties:** The system uses an `EnumSet<TokenFlags>` to efficiently store a collection of boolean properties (e.g., `OPERATOR`, `LITERAL`, `RIGHT_TO_LEFT`). This is a modern, type-safe, and highly performant alternative to traditional integer bitmasks.
*   **Context-Sensitive Disambiguation:** The `unaryVariant` field is a sophisticated mechanism to resolve ambiguity for operators like `+` and `-`. The lexer can tokenize them as their binary forms, and the parser can then transition them to their unary counterparts (`UNARY_PLUS`, `UNARY_MINUS`) based on their position within the expression, simplifying the parser's state machine.

This enum is the cornerstone of the expression compiler, providing a robust and immutable contract between the lexer and the parser.

### Lifecycle & Ownership

-   **Creation:** All Token instances are constants created by the Java Virtual Machine during the static initialization of the Token class when it is first loaded by the ClassLoader. They are not instantiated at runtime.
-   **Scope:** As enum constants, each Token is a singleton that persists for the entire lifetime of the application.
-   **Destruction:** Instances are destroyed only when the application's ClassLoader is garbage collected, which typically occurs at server shutdown.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields (`text`, `precedence`, `flags`, etc.) are final and are initialized at compile time. Once created, a Token instance's state can never change.
-   **Thread Safety:** **Inherently Thread-Safe**. As immutable singletons, Token constants can be safely accessed, passed, and compared from any thread without requiring any form of synchronization or locking. This is critical for any future multi-threaded compilation or evaluation pipelines.

## API Surface

The public API is designed for high-performance querying by the parser. Convenience methods provide a clean, readable interface over the internal flag set.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrecedence() | int | O(1) | Returns the operator precedence value. Essential for order-of-operations parsing. |
| getFlags() | EnumSet | O(1) | Returns the raw set of flags for advanced introspection. |
| containsAnyFlag(flags) | boolean | O(1) | Checks if the token possesses any of the specified flags. |
| isOperator() | boolean | O(1) | Returns true if the token is a binary or unary operator. |
| isRightToLeft() | boolean | O(1) | Returns true if the operator has right-to-left associativity (e.g., exponentiation). |
| getUnaryVariant() | Token | O(1) | Returns the unary version of this token (e.g., MINUS -> UNARY_MINUS), or null if none exists. |
| getMatchingBracket() | Token | O(1) | For a closing bracket, returns its opening counterpart. Used for syntax validation. |

## Integration Patterns

### Standard Usage

The Token enum is intended to be used by the Lexer and Parser components. The Lexer produces a stream of Tokens, and the Parser consumes them to build an Abstract Syntax Tree (AST) or evaluate the expression directly.

```java
// Hypothetical parser logic using Token properties
public void parse(TokenStream stream) {
    Token current = stream.next();

    // Example: Using properties for control flow
    if (current.isOperator()) {
        while (!operatorStack.isEmpty() && 
               operatorStack.peek().getPrecedence() >= current.getPrecedence()) {
            // Pop operators from the stack based on precedence
            popOperator();
        }
        operatorStack.push(current);
    } else if (current.isOperand()) {
        outputQueue.add(current);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **String Comparison:** Never compare a token's identity using its text representation. Enum constants guarantee a single instance, so direct object comparison is both faster and safer.
    -   **BAD:** `if (token.get().equals("+")) { ... }`
    -   **GOOD:** `if (token == Token.PLUS) { ... }`
-   **Reflection or Instantiation:** Do not attempt to create new instances of Token via reflection. The system relies on the singleton nature of enum constants.
-   **Misusing `name()` or `valueOf()`:** Avoid using `token.name()` or `Token.valueOf(String)` in performance-critical parsing loops. These methods involve string operations and memory allocation that are significantly slower than direct `==` comparison.

## Data Pipeline

The Token enum is a critical stage in the data transformation pipeline that turns a raw script string into an executable action or value.

> Flow:
> Raw Script String -> **Lexer** -> Stream of **Token** instances -> **Parser** -> Abstract Syntax Tree (AST) -> **Evaluator** -> Final Value


