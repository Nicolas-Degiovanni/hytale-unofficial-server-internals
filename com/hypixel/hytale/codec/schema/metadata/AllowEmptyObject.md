---
description: Architectural reference for AllowEmptyObject
---

# AllowEmptyObject

**Package:** com.hypixel.hytale.codec.schema.metadata
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class AllowEmptyObject implements Metadata {
```

## Architecture & Concepts
The AllowEmptyObject class is a concrete implementation of the Metadata interface, designed to apply a specific configuration policy to a Schema object. It embodies the **Strategy Pattern**, encapsulating a single, immutable rule: whether an object structure is permitted to be empty during data serialization or validation.

Architecturally, this component functions as a declarative instruction within the Hytale codec's schema-building framework. Instead of imperatively calling setters on a schema, developers compose a list of Metadata objects, including AllowEmptyObject, which are then applied to a Schema instance. This decouples the schema definition from the schema modification logic, allowing for more flexible and maintainable configuration.

Its primary role is to control a specific validation constraint, making it a critical piece for defining lenient or strict data contracts within the engine.

### Lifecycle & Ownership
-   **Creation:** Instantiated in two primary ways:
    1.  Accessed statically via the pre-configured `AllowEmptyObject.INSTANCE` for the common use case of *enabling* the policy.
    2.  Constructed directly via `new AllowEmptyObject(false)` for the less-common case of explicitly *disabling* the policy.
    Ownership is typically held by a higher-level schema builder or configuration factory during the schema construction phase.

-   **Scope:** **Transient**. These objects are short-lived and intended to exist only for the duration of the schema configuration process. They are not designed to be held as long-term state.

-   **Destruction:** Becomes eligible for garbage collection immediately after its `modify` method has been invoked and the parent configuration builder releases its reference. There are no manual cleanup requirements.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal `allowEmptyObject` boolean is a private final field, set exclusively at construction time. This design guarantees that an instance's behavior is consistent and predictable throughout its lifecycle.

-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a single AllowEmptyObject instance can be safely shared and used across multiple threads without locks or other synchronization primitives. The static `INSTANCE` field is also safe for concurrent access.

## API Surface
The public contract is minimal, focusing solely on configuration and application.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | static AllowEmptyObject | O(1) | A shared, pre-configured instance that enables the "allow empty object" behavior. Use this to avoid object allocation. |
| AllowEmptyObject(boolean) | Constructor | O(1) | Creates a new metadata policy object. |
| modify(Schema) | void | O(1) | Applies the encapsulated boolean policy to the target Schema object. Throws NullPointerException if the provided schema is null. |

## Integration Patterns

### Standard Usage
This class is intended to be passed to a schema configuration system, which will then invoke the `modify` method at the appropriate time.

```java
// Hypothetical schema builder
SchemaBuilder builder = new SchemaBuilder();

// Apply the standard policy to allow empty objects
builder.withMetadata(AllowEmptyObject.INSTANCE);

// Apply a custom policy to disallow empty objects
builder.withMetadata(new AllowEmptyObject(false));

// The builder will later iterate and apply these policies
Schema finalSchema = builder.build();
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Do not use `new AllowEmptyObject(true)`. The static `AllowEmptyObject.INSTANCE` field provides a shared, reusable object for this purpose, preventing unnecessary memory allocation.
-   **Stateful Misuse:** Do not retain a reference to an AllowEmptyObject instance after the schema has been built. It is a transient configuration object, not a component for storing runtime state.
-   **Manual Invocation:** Avoid calling the `modify` method directly. This bypasses the intended schema-building framework and can lead to an improperly configured or inconsistent Schema object.

## Data Pipeline
AllowEmptyObject is not part of a runtime data-processing pipeline. Instead, it operates within the **configuration pipeline** for schema construction.

> Flow:
> Schema Definition Code -> **AllowEmptyObject** -> Schema Builder -> Finalized Schema -> Codec Validator/Serializer

