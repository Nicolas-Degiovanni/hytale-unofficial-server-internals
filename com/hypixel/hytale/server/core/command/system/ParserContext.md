---
description: Architectural reference for ParserContext
---

# ParserContext

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class ParserContext {
```

## Architecture & Concepts
The ParserContext is a stateful, short-lived container responsible for transforming a linear sequence of command tokens into a structured, queryable format. It serves as a critical intermediate stage in the command processing pipeline, sitting between the initial Tokenizer and the final command execution logic.

Its primary architectural function is to parse and categorize arguments into three distinct types:
1.  **Positional Arguments:** Standard, ordered parameters that appear before any named arguments.
2.  **List Arguments:** A special type of positional argument, denoted by brackets `[` `]`, that groups multiple values. These lists can contain tuples of values, separated by a comma `,`.
3.  **Optional Arguments:** Key-value pairs or flags prefixed with `--`, such as `--duration=300` or `--force`.

The ParserContext operates as a single-pass parser. During its construction, it iterates through the provided token list exactly once, building its internal data structures. Parsing failures do not throw exceptions; instead, they are registered in the provided ParseResult object, which serves as an output parameter. This design allows the command system to gracefully handle malformed input and provide precise error feedback to the user.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the command system's central dispatcher for each individual command that is submitted for processing. It is created via the static factory method `ParserContext.of(tokens, parseResult)`.
-   **Scope:** The object's lifetime is strictly bound to the parsing and execution of a single command. It holds the state for one invocation only and is immediately eligible for garbage collection after the command logic has extracted the necessary arguments.
-   **Destruction:** Managed by the Java garbage collector. There are no manual cleanup or resource release methods.

## Internal State & Concurrency
-   **State:** The ParserContext is highly mutable during its construction phase. The `contextualizeTokens` method populates several internal maps and lists that represent the parsed command structure. After construction, its state is primarily read-only, with the notable exception of the `subCommandIndex` which is mutated by `convertToSubCommand` to support nested command parsing. The object effectively acts as a read-only data structure post-initialization.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the scope of a command processing task. Concurrent access to its internal collections or state variables will result in undefined behavior and data corruption.

## API Surface
The public API is designed for querying the parsed command structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(tokens, parseResult) | ParserContext | O(N) | Static factory. Creates and fully initializes a new context by parsing N tokens. |
| isListToken(index) | boolean | O(1) | Checks if the positional argument at the given index is a list argument. |
| getNumPreOptionalTokens() | int | O(L) | Returns the total count of positional arguments, where L is the number of list contexts. |
| getPreOptionalSingleValueToken(index) | String | O(1) | Retrieves a single-value positional argument by its index. |
| getPreOptionalListToken(index) | PreOptionalListContext | O(1) | Retrieves a structured list argument context by its index. |
| getFirstToken() | String | O(1) | Gets the next available token, respecting the current sub-command parsing depth. |
| getOptionalArgs() | ObjectSortedSet | O(1) | Returns the complete set of all parsed optional (named) arguments. |
| isHelpSpecified() | boolean | O(1) | A convenience method to check for the presence of a `--help` or `--?` flag. |
| convertToSubCommand() | void | O(1) | Advances the internal token cursor, effectively consuming a token as a sub-command. |

## Integration Patterns

### Standard Usage
The ParserContext is created by the command system, which then checks the associated ParseResult for errors before proceeding. If parsing is successful, getter methods are used to retrieve arguments for the command's business logic.

```java
// A command dispatcher receives raw tokens
List<String> tokens = ...;
ParseResult result = new ParseResult();

// Create the context, which performs the parsing
ParserContext context = ParserContext.of(tokens, result);

// CRITICAL: Always check the result before using the context
if (result.failed()) {
    // Send result.getFailureMessage() to the command sender
    return;
}

// Proceed to extract arguments and execute the command
String playerName = context.getPreOptionalSingleValueToken(0);
if (context.isHelpSpecified()) {
    // show help text
}
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** A ParserContext instance is stateful and tied to a single input. Never reuse an instance to parse a different command string. Always create a new one.
-   **Ignoring ParseResult:** Do not assume the context is valid after creation. The internal state may be incomplete or inconsistent if parsing failed. The ParseResult is the single source of truth for a successful parse.
-   **Concurrent Access:** Do not pass a ParserContext instance to another thread. All interaction with the object must occur on the thread that created it.

## Data Pipeline
The ParserContext is a key transformation step in the server's command processing data flow. It converts an unstructured list of strings into a semantically aware data structure.

> Flow:
> Raw User Input String -> Tokenizer -> `List<String>` -> **ParserContext** -> Structured Argument Data -> Command Executor

