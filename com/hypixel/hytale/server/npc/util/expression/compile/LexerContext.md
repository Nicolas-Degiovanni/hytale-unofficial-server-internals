---
description: Architectural reference for LexerContext
---

# LexerContext

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Transient State Object

## Definition
```java
// Signature
public class LexerContext<Token> {
```

## Architecture & Concepts
The LexerContext is a foundational state machine for the lexical analysis phase of the server-side NPC expression compiler. It is not a lexer itself, but rather the stateful cursor that a lexer algorithm operates upon. Its core responsibility is to manage the traversal of an input expression string, providing low-level primitives for character-by-character scanning and token assembly.

This class acts as a "scanner" or "pointer" into the raw source string. It maintains the current position, accumulates characters into potential tokens, and provides helper routines for parsing common language constructs like numbers, identifiers, and string literals.

The use of a generic parameter, Token, is a critical design choice. It decouples the low-level mechanics of string traversal from the high-level definition of the language's tokens. This allows the same context machinery to be used by a lexer that produces a specific, strongly-typed set of tokens, typically defined in an enum.

## Lifecycle & Ownership
- **Creation:** A LexerContext is instantiated by a higher-level lexer or parser at the beginning of a compilation task. It is a short-lived object, created on-demand for a single operation.
- **Scope:** The lifetime of a LexerContext is strictly bound to the tokenization of a single expression string. After the `init` method is called, it holds state for that expression until the process is complete.
- **Destruction:** The object is eligible for garbage collection as soon as the lexer has finished generating its token stream. The presence of the `init` method suggests it could be pooled and reused to reduce object allocation overhead, but its primary lifecycle is transient.

## Internal State & Concurrency
- **State:** The LexerContext is fundamentally a mutable object. Its entire purpose is to track and modify its internal state, most notably the `position` cursor and the `tokenString` builder, as it scans the input expression. It is a classic stateful component.

- **Thread Safety:** **WARNING:** This class is not thread-safe and must not be shared across threads. It is designed for single-threaded use within the context of a single, sequential parsing operation. Concurrent access to its methods will lead to state corruption, race conditions, and unpredictable parsing failures. Any multi-threaded compilation system must ensure each thread uses its own dedicated LexerContext instance.

## API Surface
The public and protected API provides the essential toolkit for a lexer implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(expression) | void | O(1) | Initializes or re-initializes the context for a new expression. CRITICAL: Must be called before any parsing. |
| resetToken() | void | O(1) | Clears the internal token buffer, preparing the context to scan the next token. |
| setToken(token) | Token | O(1) | Assigns the successfully parsed token to the context's state. |
| peekChar() | char | O(1) | Looks at the current character in the stream without advancing the position. |
| nextChar(error) | char | O(1) | Consumes the current character and advances the position. Throws ParseException at end-of-stream. |
| eatWhiteSpace() | boolean | O(N) | Advances the position past any contiguous block of whitespace. N is the number of whitespace characters. |
| addTokenCharacter(ch) | char | O(1) | Appends a character to the current token string and advances the position. |
| parseNumber(ch) | void | O(N) | Parses a numeric literal from the stream. N is the length of the number string. Throws on invalid format. |
| parseIdent(ch) | void | O(N) | Parses an identifier from the stream. N is the length of the identifier. |
| parseString(delimiter) | void | O(N) | Parses a string literal from the stream. N is the length of the string. Throws on unterminated string. |

## Integration Patterns

### Standard Usage
A lexer implementation is the sole intended consumer of this class. The lexer drives the context in a loop, consuming one token at a time until the end of the expression is reached.

```java
// A simplified lexer algorithm using the context
public Token nextToken(LexerContext<Token> context) throws ParseException {
    context.eatWhiteSpace();
    context.resetToken();

    char ch = context.peekChar();
    if (isIdentifierStart(ch)) {
        context.parseIdent(ch);
        // ... logic to map string to a Token enum ...
        return context.setToken(IDENTIFIER_TOKEN);
    } else if (context.isNumber(ch)) {
        context.parseNumber(ch);
        return context.setToken(NUMBER_TOKEN);
    }
    // ... other token types ...
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a LexerContext instance for a new expression without calling `init`. Failure to re-initialize will carry over state from the previous run, causing immediate and incorrect parsing results.
- **Concurrent Access:** Never pass a single LexerContext instance to multiple threads. Each parsing task requires its own isolated context.
- **External Position Management:** Avoid manipulating the position cursor with `setPosition` externally unless you are implementing advanced backtracking logic. The standard `parse*` and `nextChar` methods should be preferred as they manage state correctly.

## Data Pipeline
The LexerContext is the first stage of the compilation pipeline, responsible for converting a raw string into a structured, but still linear, format.

> Flow:
> Raw String Expression -> **LexerContext** (driven by Lexer) -> Stream of Tokens -> Parser -> Abstract Syntax Tree (AST)

