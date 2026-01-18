---
description: Architectural reference for FileContext and its nested Registry, core components for loading and managing world generation data.
---

# FileContext

**Package:** com.hypixel.hytale.server.worldgen.loader.context
**Type:** Data Structure

## Definition
```java
// Signature
public class FileContext<T> {
    // ... fields and methods ...

    public static class Registry<T> implements Iterable<Entry<String, T>> {
        // ... fields and methods ...
    }
}
```

## Architecture & Concepts

The FileContext class and its nested Registry are fundamental components of the server's world generation data loading pipeline. They serve distinct but related purposes:

*   **FileContext:** Acts as an immutable data carrier or metadata wrapper. An instance of FileContext represents a single configuration file that has been discovered by the loader. It contains essential metadata such as a unique ID, a canonical name, the file's path on disk, and a typed reference to its parent context. This design allows for the creation of a hierarchical, in-memory representation of the nested configuration file structure used for world generation (e.g., a Biome file existing within a Zone).

*   **FileContext.Registry:** Serves as a strict, high-performance, and order-preserving repository for objects deserialized from configuration files. It is not a general-purpose map; it is a purpose-built system designed to enforce configuration integrity during the server's boot sequence. Its key architectural characteristics are:
    *   **Strictness:** The Registry fails fast and hard by throwing an Error on attempts to register a duplicate key or retrieve a non-existent key. This is an intentional design choice to halt the server startup on invalid or corrupt world generation data, preventing runtime errors in a live environment.
    *   **Performance:** It is backed by the `fastutil` library's Object2ObjectLinkedOpenHashMap, which provides superior performance and lower memory overhead compared to the standard JDK HashMap, critical for processing potentially thousands of configuration entries.
    *   **Order Preservation:** The use of a linked map guarantees that the iteration order of registered items matches their registration order. This is crucial for systems where processing or dependency order matters.

Together, these classes form a pattern where a loading system iterates through files, creates a `FileContext` for each, deserializes the content, and then populates a central `Registry` with the resulting game objects (like Biomes, Prefabs, or Loot Tables).

## Lifecycle & Ownership

-   **Creation:** Instances of FileContext and its Registry are created exclusively by higher-level data loaders during the server's world generation initialization phase. A loader will typically instantiate one Registry per data type (e.g., a "Biome Registry", a "Prefab Registry"). It will then create a new FileContext for each corresponding file it finds on disk.
-   **Scope:** These objects are transient and scoped to the data loading process. They exist only as long as needed to parse files, build the in-memory data graph, and make it available to the primary world generation services.
-   **Destruction:** Once the loading phase is complete and the registered objects have been consumed by the core engine systems, the FileContext and Registry instances are no longer referenced and become eligible for garbage collection. They do not persist into the main game loop.

## Internal State & Concurrency

-   **State:**
    *   **FileContext:** Strictly immutable. All its fields are final and are assigned only once at construction.
    *   **Registry:** Mutable. Its internal state is the backing map, which is modified via the `register` method.

-   **Thread Safety:**
    *   **FileContext:** Inherently thread-safe due to its immutability. It can be safely passed between threads.
    *   **Registry:** **Not thread-safe.** The backing map is not synchronized. All operations, particularly `register`, must be performed from a single thread. This aligns with its intended use case within a single-threaded, sequential data loading stage.

    **WARNING:** Concurrent modification of a Registry instance from multiple threads without external locking will lead to data corruption, race conditions, and non-deterministic behavior. The system is designed to operate sequentially.

## API Surface

The primary API surface is on the nested Registry class. The parent FileContext class only provides simple getters for its metadata.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(name, value) | void | O(1) avg | Adds a new object to the registry. Throws Error if the key `name` already exists. |
| get(name) | T | O(1) avg | Retrieves an object by its key. Throws Error if the key `name` does not exist. |
| contains(name) | boolean | O(1) avg | Checks for the existence of a key without throwing an error. |
| size() | int | O(1) | Returns the total number of registered entries. |
| iterator() | Iterator | O(N) | Provides an iterator over all registered entries, preserving insertion order. |

## Integration Patterns

### Standard Usage

The intended pattern involves creating a registry, iterating through discovered files, and populating it. Other systems then consume the fully populated registry.

```java
// 1. A loader creates a registry for a specific data type.
FileContext.Registry<Biome> biomeRegistry = new FileContext.Registry<>("Biomes");

// 2. During file processing, the loader deserializes an object and registers it.
for (Path biomeFile : findBiomeFiles()) {
    Biome loadedBiome = parseBiome(biomeFile);
    // The name is typically derived from the file or its contents.
    biomeRegistry.register(loadedBiome.getName(), loadedBiome);
}

// 3. A downstream system, like the WorldGenerator, retrieves data from the now-immutable registry.
WorldGenerator generator = ...;
Biome plains = biomeRegistry.get("hytale:plains");
generator.useBiome(plains);
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not call `register` from multiple threads on the same Registry instance. The loading process must be single-threaded or use external synchronization mechanisms.
-   **Lazy Retrieval:** Do not attempt to `get` an entry from a registry that may not have been fully populated yet. The design assumes all registration is complete before any retrieval occurs. The thrown Error on a missing key is a signal of a configuration or load-order defect.
-   **Error Swallowing:** Do not wrap calls to `get` or `register` in a try-catch block that silently ignores the thrown Error. These errors indicate a critical, unrecoverable problem with the game's data assets that must be fixed.

## Data Pipeline

FileContext and its Registry are central components in the data ingestion pipeline for world generation.

> Flow:
> Filesystem Scan -> **FileContext** (Created per file) -> Data Deserializer -> Game Object (e.g., Biome) -> **Registry.register()** -> World Generation Service (Consumer)

