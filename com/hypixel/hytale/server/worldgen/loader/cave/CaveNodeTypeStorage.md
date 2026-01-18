---
description: Architectural reference for CaveNodeTypeStorage
---

# CaveNodeTypeStorage

**Package:** com.hypixel.hytale.server.worldgen.loader.cave
**Type:** Transient Service

## Definition
```java
// Signature
public class CaveNodeTypeStorage {
```

## Architecture & Concepts
The **CaveNodeTypeStorage** class is a specialized, stateful service responsible for the management, caching, and on-demand loading of **CaveNodeType** definitions during server-side world generation. It acts as a central registry for all cave types within a specific generation context, abstracting the underlying file system and JSON parsing logic from the core procedural generation algorithms.

Its primary architectural pattern is a **lazy-loading cache**. Cave definitions, which are stored as JSON files on disk, are only read, parsed, and instantiated when first requested by the world generator. Subsequent requests for the same cave type are served directly from an in-memory map, providing a significant performance optimization by avoiding redundant disk I/O and deserialization.

This class is tightly scoped to a single world generation task, defined by the **SeedString** and **ZoneFileContext** provided during its construction. It is not a global singleton but rather a contextual service.

## Lifecycle & Ownership
- **Creation:** An instance of **CaveNodeTypeStorage** is created by a higher-level world generation orchestrator. It is not intended to be instantiated directly by feature-level generation code. The constructor requires a complete context, including the world seed, data paths, and the current zone context, which are supplied by the parent process.
- **Scope:** The object's lifetime is bound to the generation of a specific world zone or region. It persists only as long as that single, focused generation task is active.
- **Destruction:** The object holds no native resources and does not require explicit cleanup. It is eligible for garbage collection as soon as the world generation orchestrator that created it completes its task and releases its reference.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The core of the class is the `caveNodeTypes` HashMap, which is initially empty and is populated over time as new **CaveNodeType** definitions are requested and loaded from disk. This makes the object inherently stateful.

- **Thread Safety:** This class is **not thread-safe**. The internal `HashMap` is not synchronized. If multiple threads attempt to call `getOrLoadCaveNodeType` concurrently for the same, not-yet-loaded cave type, a race condition will occur. This can lead to a node being loaded multiple times or corruption of the internal map.

    **WARNING:** All access to a shared **CaveNodeTypeStorage** instance from a multi-threaded world generation pipeline **must** be externally synchronized. Failure to do so will result in unpredictable behavior and potential crashes.

## API Surface
The public API is designed around the single responsibility of providing **CaveNodeType** instances by name.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrLoadCaveNodeType(name) | CaveNodeType | O(1) to O(N) | The primary entry point. Retrieves a node type from the cache. If not found, triggers a blocking load from disk. Complexity is O(1) on a cache hit and O(N) on a miss, where N is proportional to file size and JSON complexity. |
| add(name, caveNodeType) | void | O(1) | Manually inserts a pre-loaded node type into the cache. Throws a fatal **Error** if a node with the same name already exists. Intended for initialization or special cases. |
| getCaveNodeType(name) | CaveNodeType | O(1) | Performs a direct, non-loading lookup in the cache. Returns null if the node is not present. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the context-specific storage instance from the world generation system and use it as a factory for cave node types.

```java
// Obtain the storage service from the current generation context
CaveNodeTypeStorage storage = worldGenContext.getCaveNodeTypeStorage();

// Request a cave node type by its qualified name.
// The storage handles loading from disk if it's the first time.
CaveNodeType deepCaverns = storage.getOrLoadCaveNodeType("overworld.caves.deep");

// Use the loaded node type in the generator
proceduralGenerator.apply(deepCaverns);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CaveNodeTypeStorage()`. The required context (seed, paths, zone context) is managed by the world generation framework. Incorrectly constructing this object will lead to a disconnected and non-functional state.

- **Concurrent Loading:** Do not call `getOrLoadCaveNodeType` from multiple threads without an external lock. This will cause a race condition.
    ```java
    // BAD - Race condition if two threads call this for "overworld.caves.deep"
    // at the same time.
    executor.submit(() -> storage.getOrLoadCaveNodeType("overworld.caves.deep"));
    executor.submit(() -> storage.getOrLoadCaveNodeType("overworld.caves.deep"));
    ```

- **Forceful Re-addition:** Calling `add` for a key that may have already been loaded via `getOrLoadCaveNodeType` will crash the server with an **Error**. The `add` method is not idempotent.

## Data Pipeline
The class transforms a string identifier into a fully realized Java object by orchestrating I/O and deserialization.

> Flow:
> String Name -> **CaveNodeTypeStorage**.getOrLoadCaveNodeType -> Internal HashMap (Cache) Check
>
> **On Cache Miss:**
> String Name -> Path Resolution (`overworld.caves` -> `overworld/caves.node.json`) -> File System Read -> JSON Parser -> **CaveNodeTypeJsonLoader** -> New **CaveNodeType** Instance -> Update Internal HashMap -> Return Instance
>
> **On Cache Hit:**
> Return Instance from Internal HashMap

