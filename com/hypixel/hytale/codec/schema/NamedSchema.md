---
description: Architectural reference for NamedSchema
---

# NamedSchema

**Package:** com.hypixel.hytale.codec.schema
**Type:** Contract

## Definition
```java
// Signature
public interface NamedSchema {
```

## Architecture & Concepts
The NamedSchema interface establishes a fundamental contract within the Hytale codec and serialization framework. It is not a concrete implementation but a behavioral blueprint. Its sole purpose is to enforce that any data schema class implementing it can be uniquely identified by a stable, machine-readable string.

This interface is the cornerstone of a **Registry-based Polymorphic Deserialization** pattern. During data transmission or persistence, serialized objects are often tagged with a schema name. Upon deserialization, the codec system uses this name to query a central registry for the corresponding NamedSchema implementation. This allows the system to dynamically select the correct schema to interpret a raw byte stream, converting it back into a strongly-typed game object.

Without this contract, the system would have no reliable mechanism to map incoming data to its correct structural definition, leading to deserialization failures or data corruption.

## Lifecycle & Ownership
As an interface, NamedSchema itself has no runtime lifecycle. The following lifecycle concerns apply to **all concrete classes that implement this interface**.

-   **Creation:** Implementations are expected to be instantiated once at application startup. They are typically registered with a central SchemaRegistry or a similar service locator during a bootstrap or static initialization phase.
-   **Scope:** An instance of a NamedSchema implementation is a global singleton. It must persist for the entire application session to be available for any serialization or deserialization task.
-   **Destruction:** All registered schemas are cleared when the application shuts down or the associated registry is destroyed.

## Internal State & Concurrency
-   **State:** The NamedSchema contract implies that its implementers should be **stateless and immutable**. The schema's structure and its name must not change at runtime. Any stateful or dynamic behavior within a schema implementation is a severe anti-pattern.
-   **Thread Safety:** All implementations of this interface **must be unconditionally thread-safe**. The codec system will access registered schemas from multiple threads concurrently, especially during world loading and network packet processing. Implementations should achieve this by being immutable, typically by having only final fields and returning a compile-time constant for getSchemaName.

## API Surface
The public contract is minimal, consisting of a single method to provide the schema's identity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSchemaName() | String | O(1) | Returns the unique, non-null identifier for this schema. This value is used as a lookup key in the global schema registry. |

## Integration Patterns

### Standard Usage
A developer's primary interaction with this interface is to implement it on a class that defines a data structure. The system then handles the registration and lookup.

```java
// 1. Implement the interface on a schema definition class.
public class PlayerStateSchema implements NamedSchema {
    public static final String NAME = "hytale:player_state";

    // Schema field definitions...

    @Nonnull
    @Override
    public String getSchemaName() {
        return NAME; // Return a public, static, final constant.
    }
}

// 2. The system registers the schema at startup.
// (This code is typically in a central bootstrap location)
SchemaRegistry registry = context.getService(SchemaRegistry.class);
registry.register(new PlayerStateSchema());

// 3. The system later uses the name to look up the schema.
NamedSchema schema = registry.lookup("hytale:player_state");
```

### Anti-Patterns (Do NOT do this)
Violating these rules will lead to critical, hard-to-debug failures in the data serialization layer.

-   **Dynamic Name Generation:** Do not compute the schema name at runtime. It must be a stable, compile-time constant. Returning a value based on instance state or external factors will break the lookup mechanism.
-   **Name Collisions:** Never reuse a schema name. Every implementation of NamedSchema across the entire codebase must return a globally unique string. Collisions will cause one schema to overwrite another in the registry, leading to unpredictable deserialization errors.
-   **Null Return Value:** The method is annotated with Nonnull. Returning null will cause an immediate NullPointerException within the codec framework.

## Data Pipeline
NamedSchema is a critical component for routing data from its serialized form to the correct deserializer.

> Flow:
> Network Byte Buffer -> Packet Header (contains schema name) -> Codec Engine -> SchemaRegistry.lookup(name) -> **NamedSchema** instance -> Deserializer -> Game Object

