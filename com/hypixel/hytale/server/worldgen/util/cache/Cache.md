---
description: Architectural reference for the Cache interface, the core contract for world generation caching.
---

# Cache

**Package:** com.hypixel.hytale.server.worldgen.util.cache
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Cache<K, V> {
```

## Architecture & Concepts
The Cache interface defines the essential contract for all key-value storage systems within the server-side world generation pipeline. It serves as a critical performance abstraction, allowing different caching strategies (e.g., in-memory, disk-backed) to be used interchangeably by generation tasks.

Its primary role is to reduce redundant, computationally expensive operations such as noise sampling, biome calculation, or structure placement. By providing a simple get-based retrieval mechanism, it allows world generation workers to check for pre-computed results before initiating a costly task. This component is fundamental to achieving acceptable world generation speeds, especially in highly concurrent environments.

Implementations of this interface are expected to be specialized for the high-throughput, low-latency demands of procedural generation.

## Lifecycle & Ownership
As an interface, Cache does not have its own lifecycle. The lifecycle described here pertains to the concrete classes that implement this contract.

- **Creation:** Cache implementations are typically instantiated by a central service, such as the WorldGenerator or a dedicated CacheManager, during the initialization of a world. They are configured and provisioned based on the world's specific needs.
- **Scope:** The lifetime of a Cache instance is tightly coupled to the world or session it serves. It persists as long as the world is loaded and active, serving requests from multiple generation threads.
- **Destruction:** The `shutdown` method is the explicit signal for termination. It **must** be called when the associated world is unloaded or the server is shutting down. Failure to call `shutdown` on implementations that manage off-heap memory or file handles will result in resource leaks. The `cleanup` method is intended for periodic, non-terminal maintenance, such as evicting stale entries.

## Internal State & Concurrency
- **State:** The contract implies a mutable internal state, as the core purpose of a cache is to store and manage a collection of key-value pairs that changes over time.
- **Thread Safety:** **CRITICAL:** All implementations of the Cache interface **must be thread-safe**. The world generation system is heavily multi-threaded, and multiple worker threads will call `get` concurrently for different keys. Implementers are responsible for providing the necessary synchronization mechanisms (e.g., using concurrent collections, locks, or lock-free algorithms) to guarantee data consistency and prevent race conditions.

## API Surface
The public contract is minimal, focusing on the essential operations for a cache's lifecycle and data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shutdown() | void | O(N) | Terminates the cache and releases all associated resources. Must be called. |
| cleanup() | void | O(N) | Performs periodic maintenance, such as entry eviction. |
| get(K key) | V | O(1) to O(log N) | Retrieves the value associated with the key. Returns null on a cache miss. |

## Integration Patterns

### Standard Usage
The canonical use case involves checking the cache before performing a generation task. The caller is responsible for handling a cache miss and populating the cache with the newly generated result.

```java
// A world generation service retrieves a specialized cache
ChunkDataCache chunkCache = world.getCache(ChunkDataCache.class);
ChunkCoordinate targetCoord = new ChunkCoordinate(0, 0, 0);

// Attempt to retrieve from cache first
ChunkData data = chunkCache.get(targetCoord);

if (data == null) {
    // Cache miss: perform expensive generation
    data = generateChunkData(targetCoord);
    // Populate the cache for subsequent requests
    chunkCache.put(targetCoord, data);
}

// Use the data
processChunk(data);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Nulls:** The `get` method returns null for a cache miss. Code that does not check for null will inevitably cause NullPointerExceptions.
- **Forgetting Shutdown:** Failing to call `shutdown` on a world unload is a guaranteed resource leak for any non-trivial cache implementation.
- **Incorrect Key Implementation:** The key object K must have a correct and stable implementation of `hashCode` and `equals`. Using mutable objects or objects without proper equality checks as keys will lead to unpredictable cache behavior.

## Data Pipeline
The Cache interface acts as a conditional branch in the data flow of a world generation task. It either short-circuits the pipeline by providing a stored result or allows it to proceed, capturing the output for future requests.

> Flow:
> Generation Request (Key) -> **Cache.get(Key)** -> [Cache Hit] -> Return Cached Value (V)
>
> **Cache.get(Key)** -> [Cache Miss] -> Generation System -> Compute Value (V) -> Cache.put(Key, V) -> Return Computed Value (V)

