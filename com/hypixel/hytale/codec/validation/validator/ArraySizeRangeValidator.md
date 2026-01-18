---
description: Architectural reference for ArraySizeRangeValidator
---

# ArraySizeRangeValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class ArraySizeRangeValidator<T> implements Validator<T[]> {
```

## Architecture & Concepts
The ArraySizeRangeValidator is a specialized, single-responsibility component within the Hytale Codec validation framework. It embodies a specific validation rule: ensuring an array's element count falls within a defined minimum and maximum boundary.

Architecturally, it functions as a *Strategy* in a classic Strategy Pattern, where the `Validator` interface defines the contract for all validation rules. This allows a collection of different Validator implementations to be composed and applied to a data structure during deserialization or processing.

A key design aspect of this class is its dual role:
1.  **Runtime Validation:** The `accept` method performs the actual validation against a live array instance.
2.  **Schema Introspection:** The `updateSchema` method allows the validator to contribute its own constraints back into a static `Schema` definition. This is critical for generating documentation, network contracts, or client-side validation hints without duplicating the validation logic. It bridges the gap between runtime behavior and static data definition.

## Lifecycle & Ownership
-   **Creation:** An ArraySizeRangeValidator is not intended for direct instantiation by application code. It is typically created by a higher-level schema builder or configuration factory when a size constraint is declared for an array field. The `min` and `max` values are injected during construction.
-   **Scope:** The object's lifetime is tied to the lifecycle of the schema definition or validation chain it belongs to. As a lightweight, immutable configuration object, it persists as long as its parent schema is in memory.
-   **Destruction:** Managed by the Java Garbage Collector. It is eligible for cleanup once the schema it is associated with is no longer referenced.

## Internal State & Concurrency
-   **State:** Immutable. The internal state consists of the `min` and `max` integer fields. These are set exclusively via the constructor and cannot be modified thereafter.
-   **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that multiple threads can call its methods concurrently without risk of data corruption or race conditions. No external locking is required when using this validator.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ArraySizeRangeValidator(min, max) | constructor | O(1) | Creates a new validator instance with the specified size boundaries. |
| accept(array, results) | void | O(1) | Executes the validation logic. Fails the provided ValidationResults if the array length is outside the configured range. |
| updateSchema(context, target) | void | O(1) | Modifies the target ArraySchema to reflect the minItems and maxItems constraints. Throws ClassCastException if the target is not an ArraySchema. |

## Integration Patterns

### Standard Usage
This validator is intended to be used as part of a larger schema definition process, where it is attached to a specific array field.

```java
// A schema builder or factory would construct the validator
// and associate it with a field's validation pipeline.
SchemaBuilder.forArray("tags", String.class)
    .withValidator(new ArraySizeRangeValidator(1, 10))
    .build();

// During data processing, the validator is invoked automatically.
// Manual invocation is not a standard use case.
```

### Anti-Patterns (Do NOT do this)
-   **Mis-typed Schema Target:** Applying this validator to a schema that does not represent an array is a critical error. The `updateSchema` method will fail at runtime with a `ClassCastException` if the target `Schema` is not an `ArraySchema`.
-   **Invalid Range:** Constructing the validator with `min > max` is a logical error that the class does not prevent. This will result in a validator that always fails for any array size. The responsibility for providing a valid range lies with the calling configuration system.

## Data Pipeline
The ArraySizeRangeValidator operates as a single step within a larger data validation or deserialization pipeline.

> Flow:
> Configuration (e.g., a schema file) -> Schema Builder instantiates **ArraySizeRangeValidator** -> Deserializer receives raw data -> An array is created -> The array is passed to the **ArraySizeRangeValidator**.accept() -> ValidationResults object is populated with success or failure -> Downstream logic inspects ValidationResults to halt or continue processing.

