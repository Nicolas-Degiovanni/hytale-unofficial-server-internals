---
description: Architectural reference for PersistentMetaKey
---

# PersistentMetaKey

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Value Object

## Definition
```java
// Signature
public class PersistentMetaKey<T> extends MetaKey<T> {
```

## Architecture & Concepts
The PersistentMetaKey is a foundational component of the server's metadata and persistence system. It functions as a type-safe, composite key that bridges the gap between high-performance in-memory data access and stable, long-term storage.

Unlike its parent, MetaKey, which is designed for transient, session-only data using an integer ID, PersistentMetaKey adds two critical pieces of information:
1.  A stable **String key**: This provides a human-readable, serialization-friendly identifier used when writing data to disk or a database.
2.  A **Codec**: This encapsulates the logic for encoding and decoding the associated value of type T, ensuring data integrity between storage and runtime.

In essence, a PersistentMetaKey is not just a key; it is a complete schema definition for a single piece of metadata. It guarantees that any data associated with it can be correctly serialized, stored, and deserialized back into its original type-safe form. These keys are registered with a central authority, which uses the integer ID for fast lookups in hot paths, while the string key and codec are used for persistence operations.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by a central registry system within the `com.hypixel.hytale.server.core.meta` package during server initialization. The package-private constructor strictly prohibits direct instantiation by consumers. This centralized creation process guarantees that each key has a unique integer ID and string identifier.
-   **Scope:** A PersistentMetaKey is a static singleton for its definition. Once created and registered, it exists for the entire lifetime of the server process. They are intended to be defined as constants and referenced throughout the codebase.
-   **Destruction:** Instances are garbage collected only upon final server shutdown when their defining class loader is unloaded. There is no mechanism for manual destruction or de-registration.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (the string key, the codec, and the inherited integer ID) are declared `final` and are set only once within the constructor. An instance of this class cannot be modified after creation.
-   **Thread Safety:** **Inherently thread-safe**. Due to its complete immutability, a PersistentMetaKey instance can be safely shared, cached, and accessed by any number of threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract is minimal, focusing on providing the necessary components for persistence systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getKey() | String | O(1) | Returns the stable string identifier used for serialization and database keys. |
| getCodec() | Codec<T> | O(1) | Returns the codec responsible for serializing and deserializing the associated value. |

## Integration Patterns

### Standard Usage
Developers should never create instances of this class. Instead, they should reference statically defined keys provided by the engine or a plugin's API to interact with metadata containers.

```java
// Assume a metadata-aware object, like a Player or World
PersistentMetadataContainer container = player.getPersistentMetadata();

// 1. Retrieve a pre-defined, static key from a central registry or constants class
PersistentMetaKey<Integer> pvpDeathsKey = HytaleMetaKeys.PLAYER_PVP_DEATHS;

// 2. Use the key to set a value. The container will internally use the key's
//    codec to prepare the data for persistence.
container.set(pvpDeathsKey, 10);

// 3. Retrieve the value. The container uses the key to ensure type safety.
Integer currentDeaths = container.get(pvpDeathsKey).orElse(0);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to use reflection to bypass the package-private constructor. Doing so will create an unregistered key, leading to ID collisions, runtime exceptions, and data corruption, as the persistence layer will not recognize it.
-   **Dynamic Key Creation:** Do not generate PersistentMetaKey definitions at runtime based on dynamic game state. These keys represent a fixed data schema and must be defined at compile-time and registered at server startup.

## Data Pipeline
PersistentMetaKey does not process data itself; rather, it *defines the pipeline* for serializing and deserializing metadata associated with it.

> **Serialization Flow:**
> Game Logic -> `MetadataContainer.set(key, value)` -> **key.getCodec().encode(value)** -> Serialized Payload -> Storage Engine (using **key.getKey()** as the field name)

> **Deserialization Flow:**
> Storage Engine -> Raw Data Read -> Lookup **PersistentMetaKey** by its string name -> **key.getCodec().decode(data)** -> `MetadataContainer.set(key, decodedValue)` -> Game Logic

