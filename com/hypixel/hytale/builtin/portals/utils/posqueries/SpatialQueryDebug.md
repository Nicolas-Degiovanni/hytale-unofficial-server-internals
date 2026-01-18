---
description: Architectural reference for SpatialQueryDebug
---

# SpatialQueryDebug

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries
**Type:** Transient Utility

## Definition
```java
// Signature
public class SpatialQueryDebug {
```

## Architecture & Concepts
The SpatialQueryDebug class is a diagnostic tool designed to produce structured, hierarchical log traces for complex, multi-step geometric or spatial algorithms. It is not a core engine system but rather a stateful helper used to provide deep insight into operations like portal placement validation, procedural generation queries, or pathfinding searches.

Its primary function is to act as a contextual logger. By maintaining an indentation level and a stack of named scopes, it transforms a flat sequence of log messages into a readable, tree-like structure that mirrors the algorithm's call stack or logical flow.

This utility integrates directly with the engine's central HytaleLogger, streaming its output in real-time. This design choice prevents the need to buffer potentially large volumes of debug information in memory, making it efficient for tracing very long or complex operations.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via the `new SpatialQueryDebug()` constructor. It is typically instantiated at the beginning of a method that performs a complex spatial query and requires detailed tracing. Ownership is confined to the method scope in which it is created.
- **Scope:** The object's lifetime is intentionally brief, lasting only for the duration of the single, high-level operation it is tracing.
- **Destruction:** The instance is marked for garbage collection as soon as it falls out of scope (e.g., when the method it was created in returns). No explicit cleanup or destruction method is required.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It internally manages:
    - A `String` representing the current indentation level.
    - A `Stack` of scope descriptions, which must be manually managed via `indent` and `unindent` calls.
    - A `StringBuilder` field named `builder` is present but is never mutated outside of the `toString` method. All live logging is routed directly to HytaleLogger. This suggests the `toString` functionality may be vestigial or intended for a different, non-streaming use case.

- **Thread Safety:** This class is **not thread-safe**. Its internal state, including the indent string and scope stack, is mutated without any synchronization mechanisms. Using a single instance from multiple threads will result in a corrupted, interleaved log output and potential runtime exceptions. It is strictly intended for use by a single thread within a sequential operation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| appendLine(String) | SpatialQueryDebug | O(1) | Appends a formatted log message at the current indentation level. |
| indent(String) | SpatialQueryDebug | O(1) | Increases the indentation level and logs the reason for entering the new scope. |
| unindent() | SpatialQueryDebug | O(1) | Decreases the indentation level and logs the completion of the current scope. |
| fmt(Vector3d) | String | O(1) | A static utility method to format a Vector3d for readable log output. |

## Integration Patterns

### Standard Usage
The class is designed to be used in a lexically-scoped manner, often wrapping blocks of logic to provide context in the debug output.

```java
// Example from a hypothetical portal placement algorithm
private void findValidPlacement() {
    SpatialQueryDebug debug = new SpatialQueryDebug();
    debug.appendLine("Starting portal placement search...");

    for (Region candidateRegion : getCandidateRegions()) {
        debug.indent("Analyzing region: " + candidateRegion.getId());

        if (isRegionValid(candidateRegion)) {
            debug.appendLine("Region is valid. Finalizing placement.");
            // ...
        } else {
            debug.appendLine("Region is invalid. Skipping.");
        }

        debug.unindent();
    }
    // The 'debug' instance is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instance:** Do not store an instance of SpatialQueryDebug as a field in a service or share it across different operations. Each distinct operation requiring a trace should create its own instance.
- **Unbalanced Indentation:** Every call to `indent` must have a corresponding call to `unindent`. Failure to unindent will leave the logger in a permanently indented state for its remaining lifetime and cause a logical leak on the internal scope stack. In complex code paths with early returns, use a `try...finally` block to guarantee `unindent` is called.
- **Multi-threaded Access:** Never pass an instance to another thread or use it in a parallel stream. The resulting log output will be nondeterministic and unreadable.

## Data Pipeline
The flow of information is a direct path from the client code to the engine's logging system. The class acts as a formatting stage in this pipeline.

> Flow:
> Method Call (`appendLine`, `indent`) -> **SpatialQueryDebug** (State Update & String Formatting) -> HytaleLogger Facade -> Log Output (Console, File, etc.)

