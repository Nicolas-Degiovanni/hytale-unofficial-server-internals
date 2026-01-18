---
description: Architectural reference for UniqueInArrayValidator
---

# UniqueInArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class UniqueInArrayValidator<T> implements Validator<T[]> {
```

## Architecture & Concepts
The UniqueInArrayValidator is a specific, stateless implementation of the Validator interface, designed to enforce a uniqueness constraint on array elements. It operates within the Hytale data codec and schema validation framework.

Its architectural role is twofold:
1.  **Runtime Validation:** During data deserialization or processing, a parent validation engine invokes this class to check if a given array contains duplicate elements. It performs a direct, brute-force comparison of all elements.
2.  **Schema Configuration:** It participates in the schema definition lifecycle via the `updateSchema` method. When this validator is associated with an array property, it programmatically updates the corresponding ArraySchema to flag that its items must be unique. This allows the schema itself to accurately represent the validation rules that will be applied to it.

This component is designed as a reusable, thread-safe singleton, ensuring minimal memory overhead as it can be shared across the entire application for any array that requires this constraint.

### Lifecycle & Ownership
-   **Creation:** The single instance is created statically during class loading via the `public static final UniqueInArrayValidator<?> INSTANCE` field. It is not managed by a dependency injection container or created on-demand.
-   **Scope:** Application-wide. The singleton instance persists for the lifetime of the application's ClassLoader.
-   **Destruction:** The instance is eligible for garbage collection only when its ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no instance fields and its behavior depends solely on the arguments passed to its methods. It is therefore immutable from the perspective of a consumer.
-   **Thread Safety:** The UniqueInArrayValidator is **inherently thread-safe**. As a stateless singleton, the same instance can be safely used by multiple threads concurrently without risk of race conditions or data corruption. The responsibility for thread-safe management of the `ValidationResults` object passed into the `accept` method lies with the caller.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T[] arr, ValidationResults results) | void | O(n^2) | Validates that all elements in the input array are unique using `Objects.equals`. If a duplicate is found, a failure is registered in the `results` object. **Warning:** The quadratic complexity can cause significant performance degradation on large arrays. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Modifies the provided schema object by setting its unique items property to true. **Warning:** Throws a `ClassCastException` at runtime if the target `Schema` is not an `ArraySchema`. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct invocation. It should be registered within a schema definition, which is then used by a validation service to process data.

```java
// Hypothetical schema builder usage
Schema arraySchema = new ArraySchema()
    .setElementType(String.class)
    .addValidator(UniqueInArrayValidator.INSTANCE);

// The validation framework would then internally call:
// UniqueInArrayValidator.INSTANCE.accept(myStringArray, results);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to create a new instance of this class. The constructor is private for a reason. Always use the static `UniqueInArrayValidator.INSTANCE`.
-   **Manual Invocation on Large Datasets:** Avoid calling the `accept` method directly on arrays with thousands of elements or more. The O(n^2) performance will become a bottleneck. For large-scale uniqueness checks, a `HashSet` based approach would be more performant. This validator is intended for configuration and small-to-medium sized data structures.

## Data Pipeline
The UniqueInArrayValidator acts as a processing step within a larger data validation pipeline. It does not produce or consume data from I/O sources itself.

> Flow:
> Raw Data (e.g., JSON) -> Deserializer -> `T[]` object -> Validation Engine -> **UniqueInArrayValidator** -> ValidationResults -> Application Logic

