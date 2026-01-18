---
description: Architectural reference for IntArraySizeValidator
---

# IntArraySizeValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient / Configured Validator

## Definition
```java
// Signature
public class IntArraySizeValidator implements Validator<int[]> {
```

## Architecture & Concepts
The IntArraySizeValidator is a specialized component within the Hytale Codec and Schema Validation framework. Its sole responsibility is to enforce a fixed-size constraint on primitive integer arrays. It embodies the Single Responsibility Principle, acting as a small, composable rule within a larger validation chain.

This class serves a dual purpose, bridging runtime validation with schema definition:
1.  **Runtime Validation:** The `accept` method is executed during data deserialization or processing to check if a given `int[]` conforms to the required size.
2.  **Schema Generation:** The `updateSchema` method contributes to the static definition of a data structure. It programmatically configures an `ArraySchema` object, setting its `minItems` and `maxItems` properties to the same value. This allows for the generation of precise data schemas (e.g., for API documentation, client-side validation, or tooling) that reflect the runtime validation rules.

It is designed to be instantiated and held by a higher-level entity, such as a `ValidatorRegistry` or a specific `Codec`, rather than being used directly in application business logic.

### Lifecycle & Ownership
-   **Creation:** An instance is created with a specific, fixed size during the construction of a validation ruleset. This is typically done by a factory or builder class that interprets a higher-level data model definition. Example: `new IntArraySizeValidator(16)`.
-   **Scope:** The object's lifetime is bound to the schema or codec it helps validate. It is not a global or session-scoped service. It persists as long as the parent validation ruleset is in memory.
-   **Destruction:** The object is eligible for garbage collection once the parent schema or codec is dereferenced. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The IntArraySizeValidator is stateful, containing a single private integer field, `size`. This state is configured at construction and is **immutable** thereafter.
-   **Thread Safety:** This class is inherently **thread-safe**. Its internal state is immutable, and its methods operate only on that immutable state and the parameters passed to them. A single instance can be safely shared across multiple threads to validate different arrays concurrently without requiring locks or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| IntArraySizeValidator(int size) | constructor | O(1) | Creates a validator that enforces the specified array size. |
| accept(int[] array, ValidationResults results) | void | O(1) | Validates the array's length. If it fails, an error is added to the `results` object. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Configures the target `ArraySchema` to require a fixed number of items. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in application logic. It is composed by the framework when defining a data structure's validation rules.

```java
// Example: Within a hypothetical schema builder
SchemaBuilder builder = new SchemaBuilder();

// The framework would instantiate and attach the validator internally
builder.forField("transformMatrix")
       .asIntArray()
       .withValidator(new IntArraySizeValidator(16));

// The validator is now part of the validation chain for "transformMatrix"
```

### Anti-Patterns (Do NOT do this)
-   **Misapplication to Incorrect Schema Type:** The `updateSchema` method will throw a `ClassCastException` if the target `Schema` is not an `ArraySchema`. The framework must ensure this validator is only applied to array-type fields.
-   **Ignoring ValidationResults:** Calling the `accept` method without subsequently checking the state of the `ValidationResults` object is a critical error. A failed validation will not throw an exception; it will only report the failure to the results collector.

## Data Pipeline
The IntArraySizeValidator acts as a gate within two distinct data flows: the runtime validation pipeline and the schema generation pipeline.

**Runtime Validation Flow:**
> Raw Data (e.g., Binary Packet) -> Deserializer -> `int[]` instance -> **IntArraySizeValidator.accept()** -> ValidationResults -> Application Logic

**Schema Generation Flow:**
> High-Level Model Definition -> Schema Generator -> **IntArraySizeValidator.updateSchema()** -> Finalized `ArraySchema` -> Serialized Schema (e.g., JSON)

