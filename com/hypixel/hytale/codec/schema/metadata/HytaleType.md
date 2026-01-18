---
description: Architectural reference for HytaleType
---

# HytaleType

**Package:** com.hypixel.hytale.codec.schema.metadata
**Type:** Transient

## Definition
```java
// Signature
public class HytaleType implements Metadata {
```

## Architecture & Concepts
The HytaleType class is a specific implementation of the Metadata interface, designed to encapsulate a single, deferred configuration action. It embodies the **Command Pattern**, where an object represents an operation itself, rather than just data. In this case, an instance of HytaleType represents the command "Set the Hytale type property on a given Schema".

This component is a fundamental part of the schema definition pipeline. It allows for the decoupling of schema configuration logic. Instead of a monolithic builder with dozens of methods, the system can assemble a list of Metadata commands, including HytaleType, and apply them sequentially to a target Schema object. This makes the configuration process more modular, extensible, and easier to manage.

Its primary role is to carry the Hytale-specific type identifier (e.g., "block", "item", "particle_effect") from a configuration source into the canonical Schema object that governs serialization and deserialization.

### Lifecycle & Ownership
-   **Creation:** HytaleType instances are created on-demand by higher-level schema builders or configuration parsers. They are not managed by a dependency injection container and should be considered cheap, disposable objects. A typical creator would be a system that reads a definition file and translates a string value into this command object.
-   **Scope:** Extremely short-lived. An instance typically exists only for the duration of a single method call, where it is created, passed to a schema modifier, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the JVM garbage collector. There are no external resources or cleanup procedures required.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal *type* string is a final field, initialized exclusively at construction time. The object's state cannot be altered after creation, making its behavior perfectly predictable.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a HytaleType instance can be safely shared across multiple threads without any risk of data corruption or race conditions. No external locking or synchronization is necessary when interacting with this class.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HytaleType(String type) | constructor | O(1) | Constructs a new metadata command with the specified type identifier. |
| modify(Schema schema) | void | O(1) | Applies the stored type to the target Schema object. Throws NullPointerException if the schema is null. |

## Integration Patterns

### Standard Usage
The intended use is to instantiate HytaleType as part of a larger schema construction process. It is typically passed to a builder or a collection of Metadata objects that are applied in sequence.

```java
// A SchemaBuilder or similar factory would perform this logic
Schema targetSchema = new Schema();
List<Metadata> metadataCommands = new ArrayList<>();

// Create the command from a configuration value
metadataCommands.add(new HytaleType("hytale:block"));

// Apply all commands to the schema
for (Metadata command : metadataCommands) {
    command.modify(targetSchema);
}
```

### Anti-Patterns (Do NOT do this)
-   **Storing Instances:** Do not maintain long-term references to HytaleType instances. They are commands, not data models. Storing them in caches or as member variables of services is a misuse of their intended short-lived, transient nature.
-   **Null Schema:** Passing a null Schema object to the *modify* method will result in a NullPointerException. The caller is responsible for ensuring the target schema is properly initialized before modification.

## Data Pipeline
HytaleType acts as a carrier in the configuration pipeline. It transforms a primitive string from a source into a structured modification command that is applied to an in-memory object graph.

> Flow:
> Configuration Source (e.g., JSON) -> Parser -> **new HytaleType("value")** -> SchemaBuilder.apply() -> Modified Schema Object

