---
description: Architectural reference for NonEmptyArrayValidator
---

# NonEmptyArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class NonEmptyArrayValidator<T> extends NonNullValidator<T[]> {
```

## Architecture & Concepts
The NonEmptyArrayValidator is a specialized, stateless component within the Hytale codec and schema validation framework. Its primary function is to enforce a strict contract: an array data structure must not only exist but must also contain at least one element.

This validator serves as a critical link between declarative data schemas and imperative runtime validation. It operates in two distinct modes:

1.  **Runtime Validation:** Through its `accept` method, it performs a direct check on a materialized array object, failing immediately if the array is null or empty. This is typically invoked during data deserialization or configuration loading to ensure data integrity before it is consumed by game systems.

2.  **Schema Definition:** The `updateSchema` method allows this validator to declaratively enrich a schema definition. When applied to an `ArraySchema`, it programmatically sets the minimum item count to one. This enables tools to pre-validate data, generate compliant UI, or provide clear error messages without executing the validation logic itself.

Architecturally, it is a leaf-node component designed for composition. It is intended to be aggregated into validator chains or attached to specific fields within a larger schema, promoting reusability and a single source of truth for this common validation rule.

### Lifecycle & Ownership
-   **Creation:** The validator is a static singleton, exposed via the public `INSTANCE` field. It is instantiated by the Java ClassLoader when the `NonEmptyArrayValidator` class is first referenced. There is no manual instantiation.
-   **Scope:** Application-wide. The single instance persists for the entire lifetime of the JVM process.
-   **Destruction:** The instance is garbage collected by the JVM during application shutdown. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is exclusively determined by the arguments passed to its methods.
-   **Thread Safety:** The NonEmptyArrayValidator is **inherently thread-safe**. As a stateless singleton, the shared `INSTANCE` can be safely used by any number of threads concurrently without synchronization or risk of race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | NonEmptyArrayValidator | O(1) | The canonical, globally accessible singleton instance. |
| accept(T[], ValidationResults) | void | O(1) | Executes the validation logic. Fails if the input array is null or its length is zero. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies the target `ArraySchema` to require a minimum of one item. Throws `ClassCastException` if the target is not an `ArraySchema`. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly in application code. Instead, it should be registered with a schema builder or a validation service, which will then apply it during the appropriate phase of the data lifecycle.

```java
// Hypothetical usage within a schema definition context
SchemaBuilder.arrayField("player_inventory")
    .withValidator(NonEmptyArrayValidator.INSTANCE)
    .withItems(ITEM_SCHEMA);

// The framework later uses this to validate loaded data
ValidationResults results = new ValidationResults();
validator.accept(loadedConfig.getPlayerInventory(), results);
```

### Anti-Patterns (Do NOT do this)
-   **Reflection-based Instantiation:** The constructor is private for a reason. Do not attempt to create new instances of this class via reflection. Always use the provided `NonEmptyArrayValidator.INSTANCE`.
-   **Incorrect Schema Target:** Applying this validator to a schema that does not represent an array (e.g., a `StringSchema` or `ObjectSchema`) will result in a `ClassCastException` when `updateSchema` is called. The framework must ensure type-correctness when pairing validators with schemas.

## Data Pipeline
The NonEmptyArrayValidator acts as a gate within a larger data processing pipeline, typically during deserialization or configuration loading.

> Flow:
> Raw Data (JSON, Binary) -> Deserializer -> Object Graph -> Validation Service -> **NonEmptyArrayValidator.accept()** -> [Fail or Pass] -> Validated Game State

