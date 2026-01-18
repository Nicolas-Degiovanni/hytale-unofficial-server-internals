---
description: Architectural reference for DeprecatedValidator
---

# DeprecatedValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class DeprecatedValidator<T> implements LegacyValidator<T> {
```

## Architecture & Concepts
The DeprecatedValidator is a specialized component within the data codec and validation framework. Its sole responsibility is to mark a data field as deprecated, serving as a forward-compatibility and schema evolution mechanism.

Architecturally, it functions as a stateless, non-blocking "marker" validator. Unlike other validators that might inspect data for correctness, this class completely ignores the input data. Its only action is to unconditionally append a standardized warning message to the validation results. This design allows developers to signal that a field is being phased out without breaking older clients or data formats.

The use of a generic type parameter `T` and a static wildcard instance `<?>` makes this validator universally applicable to any field of any type, maximizing reusability and minimizing the need for type-specific deprecation logic.

### Lifecycle & Ownership
-   **Creation:** The singleton `INSTANCE` is instantiated once by the JVM upon class loading. This is a static initialization pattern.
-   **Scope:** Application-wide. The single instance persists for the entire lifetime of the application.
-   **Destruction:** The instance is destroyed only when the JVM shuts down. There is no manual cleanup required or possible.

## Internal State & Concurrency
-   **State:** This component is **stateless and immutable**. It contains no instance fields and its behavior is constant.
-   **Thread Safety:** The DeprecatedValidator is inherently thread-safe. As a stateless singleton, it can be safely invoked by multiple threads concurrently without any risk of race conditions or data corruption. No synchronization mechanisms are required.

## API Surface
The public contract consists of a single method inherited from the LegacyValidator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t, ValidationResults results) | void | O(1) | Appends a deprecation warning to the results. The input `t` is ignored. Throws NullPointerException if `results` is null. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly. Instead, it should be registered with a field within a higher-level schema definition or data-binding framework. The framework is then responsible for invoking the validator during the deserialization or validation process.

```java
// Hypothetical usage within a schema definition system
// The framework will automatically use DeprecatedValidator.INSTANCE
// during the validation phase for the "playerScore" field.

Schema.builder()
    .field("playerScore", Integer.class)
        .withValidator(DeprecatedValidator.INSTANCE)
    .build();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to create a new instance. The constructor is private to enforce the singleton pattern. Always use the static `DeprecatedValidator.INSTANCE` field.
-   **Conditional Logic:** Do not wrap calls to this validator in conditional logic. Its purpose is to unconditionally mark a field as deprecated. If conditional validation is needed, a different, more complex validator should be used.
-   **Expecting Data Inspection:** Do not assume this validator inspects or uses the data passed to it. It is designed to ignore the data entirely.

## Data Pipeline
The DeprecatedValidator operates as a single, simple step within a larger data validation pipeline.

> Flow:
> Serialized Data Stream -> Codec Deserializer -> Validation Engine -> **DeprecatedValidator** -> Aggregated ValidationResults -> Application Logic

