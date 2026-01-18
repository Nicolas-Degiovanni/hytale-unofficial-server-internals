---
description: Architectural reference for Lexer
---

# Lexer

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Utility

## Definition
```java
// Signature
public class Lexer<Token extends Supplier<String>> {
```

## Architecture & Concepts
The Lexer is the foundational component of the expression compilation pipeline. Its primary responsibility is to perform lexical analysis: converting a raw stream of characters from an input string into a structured stream of discrete symbols, known as tokens. It serves as the bridge between raw text and the higher-level Parser, which understands grammatical structure.

This implementation is designed for reusability and efficiency. The core architectural decisions include:

*   **Context-Driven State:** The Lexer itself is stateless with respect to any single parsing operation. All mutable state, such as the current position in the input string and the text of the current token, is encapsulated within a `LexerContext` object passed into the `nextToken` method. This design allows a single configured Lexer instance to be safely used by multiple parsers concurrently, provided each uses a distinct `LexerContext`.

*   **Trie-based Operator Matching:** For recognizing operators (e.g., `+`, `-`, `==`, `!=`), the Lexer constructs a prefix tree, or Trie, via the internal `CharacterSequenceMatcher` class. This data structure allows for efficient matching of multi-character operators. As the Lexer consumes characters, it traverses the Trie. This avoids costly string comparisons and correctly handles overlapping operators (e.g., distinguishing `==` from `=`).

*   **Generic Token System:** The class is generic over a `Token` type, which must extend `Supplier<String>`. This provides flexibility, allowing the compiler to use any token representation (like an enum) as long as it can supply its string value. This string value is crucial for building the operator matching Trie during initialization.

## Lifecycle & Ownership
*   **Creation:** A Lexer is instantiated by a higher-level system responsible for defining a language's grammar, such as an `ExpressionCompiler` or a dedicated factory. During construction, it is configured with the specific token types for the language (identifiers, numbers, strings) and a complete stream of all valid operator tokens.
*   **Scope:** An instance of Lexer is long-lived. It is configured once with a specific language definition and persists as long as that language needs to be parsed. It is a reusable component, not a single-use object.
*   **Destruction:** The object has no explicit destruction logic. It is garbage collected when the owning compiler or factory that created it is no longer referenced.

## Internal State & Concurrency
*   **State:** The internal state of the Lexer is established during construction and is **immutable** thereafter. This state consists of the token type references and the `characterSequenceMatcher` Trie. The Trie is built once and is only used for read operations during tokenization.
*   **Thread Safety:** The Lexer class is **thread-safe**. The `nextToken` method's logic operates on the immutable internal configuration and the externally provided `LexerContext`. Because all mutable parsing state is confined to the `LexerContext`, multiple threads can invoke `nextToken` on the same Lexer instance without interference, as long as each thread supplies its own unique `LexerContext`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nextToken(LexerContext context) | Token | O(L) | Scans the input stream via the context and returns the next recognized token. L is the length of the token. Advances the context's internal cursor. Throws ParseException for unrecognized characters. |

## Integration Patterns

### Standard Usage
The Lexer is typically used in a loop by a Parser. The loop repeatedly calls `nextToken` to build a stream of tokens, which the Parser then consumes to build an Abstract Syntax Tree (AST).

```java
// Assume 'context' is a LexerContext initialized with source code
// Assume 'lexer' is a fully configured Lexer instance

List<Token> tokens = new ArrayList<>();
Token currentToken;

do {
    currentToken = lexer.nextToken(context);
    tokens.add(currentToken);
} while (currentToken != endToken);

// The 'tokens' list can now be processed by a parser.
```

### Anti-Patterns (Do NOT do this)
*   **Stateful Misconception:** Do not assume the Lexer instance stores the parsing position. All state is in the `LexerContext`. Attempting to use a single Lexer instance for one parse operation across multiple threads without separate contexts will lead to unpredictable behavior.
*   **Incomplete Operator Set:** Failing to provide the complete set of valid language operators to the constructor will result in a Lexer that cannot recognize them, causing `ParseException` errors for valid syntax.

## Data Pipeline
The Lexer is the first active stage in the expression processing pipeline, transforming raw text into a format suitable for syntactic analysis.

> Flow:
> Raw String -> `LexerContext` -> **`Lexer`** -> Stream of `Token` objects -> Parser -> Abstract Syntax Tree

