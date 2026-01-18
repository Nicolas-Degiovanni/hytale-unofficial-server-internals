---
description: Architectural reference for PrefabBufferDeserializer
---

# PrefabBufferDeserializer

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface PrefabBufferDeserializer<T> {
```

## Architecture & Concepts
The PrefabBufferDeserializer interface defines a contract for a critical engine component: the deserialization of a prefab from a persistent storage format into a runtime, in-memory representation. This interface embodies the Strategy design pattern, allowing the core prefab loading system to remain agnostic to the underlying file format (e.g., NBT, JSON, XML).

Each implementation of this interface is responsible for a single format. A central registry or manager, likely keyed by file extension or a format identifier, selects the appropriate deserializer at runtime. The generic parameter, T, represents a context or options object that can be passed to the deserializer, allowing for flexible and extensible loading behavior without modifying the interface contract. This design is fundamental to the engine's ability to support custom prefab formats and future-proof the asset pipeline.

## Lifecycle & Ownership
As an interface, PrefabBufferDeserializer does not have a concrete lifecycle. The following applies to its **implementations**.

- **Creation:** Implementations are expected to be instantiated once at application startup. They are typically registered with a central service, such as a PrefabFormatRegistry or AssetManager, during a bootstrap or plugin-loading phase.
- **Scope:** An implementation's instance is a long-lived singleton. It persists for the entire server session. Because they are stateless strategies, a single instance can serve all deserialization requests for its designated format.
- **Destruction:** The instances are destroyed when the server shuts down and the central registry is cleared. There is no manual cleanup required during normal operation.

## Internal State & Concurrency
- **State:** The interface contract implies that all implementations **must be stateless**. They should not hold any data related to a specific deserialization operation between calls. Any required state must be passed in via the path or the generic context object T.
- **Thread Safety:** Implementations **must be thread-safe**. The asset loading system may invoke deserializers from multiple worker threads concurrently to improve performance, especially during world generation or chunk loading. All implementations must be re-entrant and free of side effects.

**WARNING:** Failure to ensure an implementation is stateless and thread-safe will lead to severe data corruption, race conditions, and unpredictable server crashes. Avoid instance variables that are modified during the `deserialize` call.

## API Surface
The public contract consists of a single method for performing the deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(Path, T) | PrefabBuffer | O(N) | Reads and parses the file at the given Path, converting it into a PrefabBuffer. N is the size of the file. Throws IOException on file access errors or a format-specific exception on parsing failure. |

## Integration Patterns

### Standard Usage
A developer should never invoke a deserializer directly. Interaction should occur through a higher-level service that manages the selection and execution of the correct deserializer.

```java
// Example of how the engine's PrefabService might use a deserializer
// This code is conceptual and for illustration only.

// 1. The service retrieves the correct deserializer from a registry
PrefabBufferDeserializer deserializer = formatRegistry.getDeserializerFor("nbt");

// 2. The service invokes the deserializer with the path and context
Path prefabPath = Paths.get("prefabs/structures/house.nbt");
PrefabLoadOptions options = new PrefabLoadOptions(); // The 'T' context object
PrefabBuffer buffer = deserializer.deserialize(prefabPath, options);

// 3. The buffer is now ready for use in the game world
world.placePrefab(buffer);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Do not look up and call a specific deserializer. Always use the central prefab loading service, which handles format detection, caching, and error handling.
- **Stateful Implementations:** Do not store file handles, partial results, or any other per-request data as instance fields within an implementation class. This immediately violates thread-safety requirements.
- **Incorrect Registration:** Registering a deserializer for the wrong file type or overriding a default engine deserializer without careful consideration can destabilize the entire asset system.

## Data Pipeline
This interface is a key transformation step in the data pipeline that brings prefabs from disk into the live game world.

> Flow:
> File Path & Load Options -> Prefab Service -> **PrefabBufferDeserializer** -> In-Memory PrefabBuffer -> World Engine

