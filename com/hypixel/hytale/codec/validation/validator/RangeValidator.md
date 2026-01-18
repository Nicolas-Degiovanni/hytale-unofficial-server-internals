---
description: Architectural reference for RangeValidator
---

# RangeValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class RangeValidator<T extends Comparable<T>> implements Validator<T> {
```

## Architecture & Concepts
The RangeValidator is a concrete implementation of the Validator interface, designed to enforce boundary constraints on any data type that implements the Comparable interface. It is a fundamental building block within the Hytale Codec system, which governs data serialization, deserialization, and validation across the engine.

This component follows the **Strategy** design pattern, encapsulating the logic for range checking into a reusable, stateless object. Its role is twofold:

1.  **Runtime Data Validation:** At its core, the RangeValidator checks if a given value falls within a specified minimum and maximum boundary. This is its primary operational function during data processing.
2.  **Schema Introspection & Enrichment:** The validator can inspect and modify a Hytale Schema object. It injects its own range constraints (minimum, maximum, inclusive/exclusive) into the schema definition. This critical feature allows the system to auto-generate documentation, client-side validation rules, or network contract definitions that are guaranteed to be in sync with server-side validation logic.

The generic nature of the class allows it to operate on various types, such as integers, floats, or even custom comparable objects, though its schema enrichment capabilities are specialized for numeric types.

## Lifecycle & Ownership
-   **Creation:** A RangeValidator is instantiated directly via its constructor, typically by a higher-level service or a configuration builder that defines a set of validation rules. It is not a managed singleton.
-   **Scope:** The object is short-lived and its lifetime is tied to the configuration or schema definition that references it. It holds no external resources and is designed to be lightweight.
-   **Destruction:** The RangeValidator is managed by the Java garbage collector. It is eligible for collection as soon as the containing configuration or rule set is no longer in scope. No explicit cleanup is necessary.

## Internal State & Concurrency
-   **State:** The internal state of a RangeValidator instance, consisting of its minimum, maximum, and inclusivity flag, is **immutable**. These values are set once at construction via final fields.
-   **Thread Safety:** Due to its immutable state, the RangeValidator is inherently **thread-safe**. A single instance can be safely shared and used by multiple threads simultaneously to perform validation without requiring any external locking or synchronization.

## API Surface
The public contract of RangeValidator is focused on its dual responsibilities of validation and schema modification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t, ValidationResults results) | void | O(1) | Executes the validation logic. Compares the input value against the configured bounds and reports failures by mutating the provided ValidationResults object. |
| updateSchema(SchemaContext context, Schema target) | void | O(N) | Enriches the target Schema with this validator's range constraints. Complexity is O(N) where N is the number of sub-schemas in an *anyOf* clause. Logs warnings if the target schema is not a recognized numeric type. |

## Integration Patterns

### Standard Usage
The most common use case is to instantiate a RangeValidator as part of a larger validation rule set, which is then applied to incoming data.

```java
// Example: Validating a player's health value
ValidationResults results = new ValidationResults();
RangeValidator<Integer> healthValidator = new RangeValidator<>(0, 100, true);

// The validator is applied to the data
healthValidator.accept(player.getHealth(), results);

if (results.hasFailed()) {
    // Handle the validation failure, e.g., log an error or reject the data.
    System.out.println("Validation failed: " + results.getErrors().get(0));
}
```

### Anti-Patterns (Do NOT do this)
-   **Mismatched Schema Types:** Do not apply the updateSchema method to a Schema that is not a NumberSchema or IntegerSchema. The method contains specific logic for these types and will fail to apply constraints to other schema types, logging a warning.
-   **Redundant Validation:** Creating a validator with both min and max set to null (e.g., `new RangeValidator<>(null, null, true)`) is useless. It performs no validation and adds unnecessary overhead. The system will not prevent this, but it indicates a logical error in the configuration.

## Data Pipeline
The RangeValidator acts as a gate within a larger data validation pipeline. It does not transform data but rather asserts its state.

> **Validation Flow:**
> Raw Data (e.g., from Network or File) -> Deserializer -> In-Memory Object (`T`) -> Validation Service -> **RangeValidator.accept(T)** -> Mutated ValidationResults

> **Schema Generation Flow:**
> Configuration Builder -> **RangeValidator.updateSchema(Schema)** -> Enriched Schema Definition -> Schema Serializer -> Published Contract (e.g., JSON Schema)

