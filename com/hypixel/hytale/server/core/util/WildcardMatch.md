---
description: Architectural reference for WildcardMatch
---

# WildcardMatch

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public final class WildcardMatch {
```

## Architecture & Concepts
WildcardMatch is a stateless, low-level utility class designed to provide a high-performance, non-regular-expression-based string matching capability. It supports two standard wildcards: the question mark (?) for a single character match and the asterisk (*) for a zero-or-more character match.

This component is a foundational utility, intended to be used across the server architecture wherever simple pattern matching is required. Common use cases include command argument validation, player name filtering, permission node checking, and configuration file parsing. It is intentionally isolated from the core game loop and engine services, functioning as a pure, dependency-free computational tool. Its design avoids the overhead and complexity of the standard Java Regex engine for scenarios where only basic wildcard support is necessary.

## Lifecycle & Ownership
As a final class with a private constructor and exclusively static methods, WildcardMatch has no object lifecycle.

- **Creation:** The class is never instantiated. The JVM class loader loads it on first use.
- **Scope:** Its static methods are available globally for the entire duration of the server's runtime.
- **Destruction:** The class is unloaded when the JVM shuts down. No manual cleanup is required.

## Internal State & Concurrency
- **State:** WildcardMatch is completely stateless. Each call to the test method operates independently on the provided arguments. All internal variables are method-local and exist only on the stack for the duration of the call.
- **Thread Safety:** This class is inherently thread-safe. Because it is stateless, multiple threads can call the test method concurrently with different inputs without any risk of interference or race conditions. No external locking or synchronization is required.

## API Surface
The public contract consists of two static methods providing the core matching logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(text, pattern) | boolean | O(T\*P) | Performs a case-sensitive wildcard match of a string against a pattern. |
| test(text, pattern, ignoreCase) | boolean | O(T\*P) | Performs a wildcard match, with an option to ignore character casing. |

**Warning:** The matching algorithm is highly optimized but its complexity is proportional to the product of the text length (T) and pattern length (P). Avoid using this utility on extremely large strings within performance-critical, high-frequency loops without careful profiling.

## Integration Patterns

### Standard Usage
This is a static utility. It should be invoked directly without instantiation or retrieval from a service context.

```java
// Example: Checking if a player's name matches a ban pattern.
String playerName = "TestPlayer123";
String banPattern = "test*";

boolean isMatch = WildcardMatch.test(playerName, banPattern, true); // ignore case

if (isMatch) {
    // Take action...
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The class has a private constructor and cannot be instantiated. `new WildcardMatch()` will result in a compile-time error.
- **Complex Regex:** Do not attempt to use standard regular expression syntax beyond `?` and `*`. Patterns like `[a-z]+` or `(test|prod)` are not supported and will be treated as literal characters.
- **Null Inputs:** Passing null for either the text or pattern argument will result in a NullPointerException. Always perform null checks on inputs before calling this utility.

## Data Pipeline
WildcardMatch acts as a pure function, transforming input strings into a boolean result. It is not part of a larger, asynchronous data flow.

> Flow:
> String `text`, String `pattern` -> **WildcardMatch.test()** -> Boolean `result`

