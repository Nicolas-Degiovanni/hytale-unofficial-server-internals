---
description: Architectural reference for NoDefaultValue
---

# NoDefaultValue

**Package:** com.hypixel.hytale.codec.schema.metadata
**Type:** Singleton

## Definition
```java
// Signature
public class NoDefaultValue implements Metadata {
```

## Architecture & Concepts
The NoDefaultValue class is a stateless policy object within the Hytale Schema and Codec framework. It embodies the Strategy design pattern, providing a concrete implementation of the Metadata interface. Its sole responsibility is to modify a Schema definition to ensure that a particular field has no default value.

This component is critical for data validation and serialization integrity. By explicitly removing a default value, the system can enforce that a field must be present in the source data, preventing silent data corruption or unexpected behavior from implicitly assigned defaults. It acts as a declarative instruction during the schema construction phase, influencing how data is later parsed and validated.

As a singleton, it is designed to be a globally accessible, reusable utility, eliminating the overhead of repeated instantiation for a stateless operation.

### Lifecycle & Ownership
- **Creation:** The single instance is created by the Java Class Loader when the NoDefaultValue class is first referenced, via the static final INSTANCE field. It is not instantiated by any application-level factory or dependency injection container.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the JVM.
- **Destruction:** The instance is eligible for garbage collection only when the application's Class Loader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** NoDefaultValue is completely stateless and therefore immutable. It contains no instance fields and its behavior is determined solely by the arguments passed to its methods.
- **Thread Safety:** This class is inherently thread-safe. Multiple threads can safely call the modify method on the shared INSTANCE without any risk of race conditions or data corruption, as the method operates only on its local parameters.

**WARNING:** While NoDefaultValue itself is thread-safe, the Schema object passed to the modify method may not be. It is the caller's responsibility to ensure that the Schema instance is not being mutated by other threads concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Sets the default value of the provided Schema to null. This operation only affects supported subtypes (StringSchema, IntegerSchema, NumberSchema, BooleanSchema). |

## Integration Patterns

### Standard Usage
NoDefaultValue is not intended to be invoked imperatively. Instead, it should be passed as a configuration parameter to a schema builder or definition factory. The framework then uses it to apply the policy during schema construction.

```java
// Hypothetical schema builder demonstrating declarative usage
Schema stringField = SchemaBuilder.string()
    .withName("username")
    .withMetadata(NoDefaultValue.INSTANCE) // Correct usage
    .build();

// The framework internally calls instance.modify(schema)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create new instances using reflection. This violates the singleton pattern and provides no benefit. Always use NoDefaultValue.INSTANCE.
- **Unnecessary Null Checks:** Do not check if NoDefaultValue.INSTANCE is null. As a static final field, it is guaranteed to be initialized before it can be accessed.
- **Misapplication:** Applying this metadata to a schema type that does not support default values (e.g., a complex object schema) will have no effect and may indicate a misunderstanding of the schema's capabilities.

## Data Pipeline
NoDefaultValue is not part of a runtime data processing pipeline. It operates at the meta-level, configuring the components that define the pipeline.

> **Configuration Flow:**
> Schema Definition (Code) -> Schema Builder -> **NoDefaultValue.modify()** -> Finalized Schema Object -> Data Validator / Codec

---

