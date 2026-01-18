---
description: Architectural reference for Documentation
---

# Documentation

**Package:** com.hypixel.hytale.codec.util
**Type:** Utility

## Definition
```java
// Signature
public class Documentation {
```

## Architecture & Concepts
The Documentation class is a stateless, special-purpose utility designed for text sanitization. Its sole function is to transform a string containing simple Markdown emphasis syntax into a plain text representation by stripping the formatting characters.

This component resides within the `codec.util` package, indicating its role as a helper in a broader data processing or serialization context. It is typically invoked at the boundary where formatted text, potentially from content management systems, configuration files, or user input, must be rendered in a context that does not support rich text, such as a log file, a debug console, or a simple UI label. It acts as a final-stage filter in a data presentation pipeline.

The implementation is notable for its correctness in handling nested and balanced formatting markers, throwing an exception for malformed input. This makes it a robust tool for ensuring data integrity before rendering.

## Lifecycle & Ownership
- **Creation:** As a utility class composed exclusively of static methods, Documentation is never instantiated. The class is loaded into the JVM by its ClassLoader upon the first invocation of any of its methods.
- **Scope:** The class and its static methods have an application-wide scope. They are globally accessible throughout the application's lifetime.
- **Destruction:** The class is unloaded from the JVM when its defining ClassLoader is garbage collected, which typically occurs during application shutdown.

## Internal State & Concurrency
- **State:** The Documentation class is entirely stateless. The `stripMarkdown` method is a pure function; its output is determined exclusively by its input arguments. All internal variables, such as the `StringBuilder` and `IntArrayList`, are confined to the method's local scope and are discarded upon its completion.
- **Thread Safety:** This class is unconditionally thread-safe. The `stripMarkdown` method operates on an immutable input string and its own local variables. Multiple threads can invoke this method concurrently with different inputs without any risk of race conditions or data corruption. No synchronization mechanisms are necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| stripMarkdown(@Nullable String markdown) | String | O(N) | Parses the input string, removing Markdown emphasis characters (`*`, `_`). Returns null if the input is null. Throws `IllegalArgumentException` if formatting markers are unbalanced. |

## Integration Patterns

### Standard Usage
This utility should be invoked directly via its static method wherever Markdown-formatted text needs to be converted to plain text.

```java
// How a developer should normally use this
String formattedDescription = "This item is **critically important** for the quest.";
try {
    String plainText = Documentation.stripMarkdown(formattedDescription);
    // Use plainText for display in a simple tooltip or log
    System.out.println(plainText);
} catch (IllegalArgumentException e) {
    // Handle cases where content may be malformed
    System.err.println("Failed to parse description: " + e.getMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new Documentation()`. This provides no value, wastes memory, and subverts the intended static-utility design pattern.
- **Ignoring Exceptions:** The `IllegalArgumentException` is a critical signal of malformed input data. Failing to catch this exception when processing external or user-provided content can lead to unhandled exceptions and application instability. Always wrap calls in a try-catch block in such scenarios.
- **Misuse for Complex Markdown:** This utility is designed only for simple emphasis (`*` and `_`). Do not use it to parse complex Markdown containing headers, links, or code blocks, as it will produce incorrect output.

## Data Pipeline
The Documentation class functions as a transformation step within a larger data flow. It takes formatted text as input and outputs sanitized, plain text.

> Flow:
> Content Source (e.g., JSON API) -> Deserializer -> **Documentation.stripMarkdown** -> UI Text Component -> Rendered Output

