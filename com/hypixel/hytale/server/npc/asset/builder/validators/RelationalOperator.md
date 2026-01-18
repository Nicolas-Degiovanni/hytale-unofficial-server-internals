---
description: Architectural reference for RelationalOperator
---

# RelationalOperator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Value Object

## Definition
```java
// Signature
public enum RelationalOperator implements Supplier<String> {
```

## Architecture & Concepts
The RelationalOperator enum provides a type-safe, compile-time constant representation for mathematical and logical comparisons. It is a fundamental building block within the server's NPC asset validation system, designed to enforce rules and conditions defined in game data.

Its primary architectural purpose is to eliminate "stringly-typed" programming. By replacing raw strings like "less than" or ">=" with strongly-typed enum constants (e.g., RelationalOperator.Less, RelationalOperator.GreaterEqual), it prevents common runtime errors caused by typos and inconsistencies.

The implementation of the `Supplier<String>` interface provides a standardized contract for retrieving a human-readable representation of the operator. This is leveraged by higher-level systems for generating descriptive validation error messages, logs, or debugging UIs without needing to know the internal details of the enum.

## Lifecycle & Ownership
- **Creation:** All instances of this enum are created and initialized by the JVM during class loading. There is no manual instantiation path; the set of operators is fixed at compile time.
- **Scope:** Application-level singleton. The enum constants exist for the entire lifetime of the server application.
- **Destruction:** The instances are reclaimed by the garbage collector only when the application's class loader is unloaded, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a single, final `asText` field that is initialized at creation and never changes. The object's state is static for the application's lifetime.
- **Thread Safety:** Inherently thread-safe. As immutable, globally accessible singletons, instances of RelationalOperator can be safely read and passed between any number of threads without requiring locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| asText() | String | O(1) | Returns the human-readable text representation of the operator (e.g., "less than or equal to"). |
| get() | String | O(1) | Implements the Supplier interface. Functionally identical to asText(). |
| values() | RelationalOperator[] | O(N) | *Static method inherited from Enum.* Returns an array of all defined enum constants. **Warning:** Creates a new array on every call. |
| valueOf(String) | RelationalOperator | O(N) | *Static method inherited from Enum.* Returns the enum constant corresponding to the given name (e.g., "LessEqual"). Throws IllegalArgumentException if no match is found. |

## Integration Patterns

### Standard Usage
The primary use case is within conditional logic, often inside a `switch` statement, to perform validation checks. The `asText` method is then used for generating feedback.

```java
// How a developer should normally use this
public boolean validate(int value, int threshold, RelationalOperator op) {
    boolean result = false;
    switch (op) {
        case Equal:
            result = (value == threshold);
            break;
        case Greater:
            result = (value > threshold);
            break;
        // ... other cases
    }

    if (!result) {
        logger.warn("Validation failed: {} is not {} {}", value, op.asText(), threshold);
    }
    return result;
}
```

### Anti-Patterns (Do NOT do this)
- **String-based Comparison:** Never compare operators by their text value. This is inefficient and defeats the purpose of a type-safe enum.

```java
// INCORRECT
if (operator.asText().equals("equal to")) {
    // ...
}

// CORRECT
if (operator == RelationalOperator.Equal) {
    // ...
}
```

- **Unsafe `valueOf`:** Do not pass untrusted user or file input directly to `RelationalOperator.valueOf()`. It will throw an unhandled `IllegalArgumentException` if the input string does not exactly match an enum constant name, potentially crashing the thread. Always sanitize or validate such inputs first.

## Data Pipeline
RelationalOperator is not a data processing component itself, but rather a static data definition used *by* processing components. It typically enters a pipeline when game assets are parsed and validated.

> Flow:
> NPC Asset File (JSON) -> Asset Deserializer -> Validator Component -> **RelationalOperator** (used for rule evaluation) -> Validation Result (Success/Failure)

