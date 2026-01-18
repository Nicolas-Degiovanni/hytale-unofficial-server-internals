---
description: Architectural reference for UniquePrefabCache
---

# UniquePrefabCache

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Stateful Service

## Definition
```java
// Signature
public class UniquePrefabCache {
```

## Architecture & Concepts

The UniquePrefabCache is a specialized, in-memory caching layer designed to optimize the server's world generation pipeline. Its primary function is to memoize the computationally expensive process of determining unique prefab placements for a given world seed.

This class acts as a facade over a more generic, time-and-size-limited cache implementation, SizedTimeoutCache. By wrapping this generic cache, it provides a type-safe and domain-specific API for handling UniquePrefabEntry arrays. It implements a **Read-Through** caching strategy. When a request for prefab data is made via the `get` method, the cache is checked first. On a cache miss, it transparently invokes a "loader" function (the UniquePrefabFunction provided at construction) to generate the data, stores it, and then returns it to the caller.

This component is critical for performance, preventing the server from re-calculating deterministic prefab data for the same seed repeatedly, which is common during concurrent or sequential chunk generation for a single world.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation coordinator, such as a WorldGenerator or WorldGenContext, during the initialization of a new world generation session. The cache's behavior (max size, expiration) is configured at this time, likely based on server settings.
-   **Scope:** The lifecycle of a UniquePrefabCache instance is tightly coupled to the world generation session it serves. It is not a global, server-wide singleton. It persists as long as its owning generator context is active.
-   **Destruction:** The object is eligible for garbage collection once the world generation session concludes and all references to its parent context are released. There is no explicit `destroy` or `close` method.

## Internal State & Concurrency
-   **State:** This class is **Mutable** and stateful by design. Its internal SizedTimeoutCache continuously adds, evicts, and expires entries based on usage patterns and time.
-   **Thread Safety:** This class is designed to be **thread-safe**. The responsibility for managing concurrent access is delegated to the underlying SizedTimeoutCache implementation. This is essential in a server environment where multiple worker threads may be generating different chunks for the same world simultaneously and require access to the same seed-based prefab data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed) | UniquePrefabContainer.UniquePrefabEntry[] | Amortized O(1) | Retrieves prefab entries for a seed. A cache miss triggers the loader function, making the worst-case complexity dependent on that function. Throws a fatal Error if the loader fails. |

## Integration Patterns

### Standard Usage
The cache is intended to be created once per world generation context, providing a loader function that performs the actual prefab calculation. Generation threads then query the cache instance.

```java
// In a WorldGenerator or similar context
// 1. Define the expensive calculation logic
UniquePrefabCache.UniquePrefabFunction prefabLoader = (seed) -> {
    // ... perform complex, deterministic prefab calculations based on seed
    return calculatePrefabsForSeed(seed);
};

// 2. Instantiate the cache with specific policies
int maxCachedSeeds = 10;
long expireAfterMinutes = 60;
UniquePrefabCache cache = new UniquePrefabCache(prefabLoader, maxCachedSeeds, expireAfterMinutes * 60);

// 3. Use the cache during chunk generation
UniquePrefabContainer.UniquePrefabEntry[] prefabs = cache.get(world.getSeed());
// ... use prefabs to populate the world
```

### Anti-Patterns (Do NOT do this)
-   **Non-Deterministic Loader:** Providing a UniquePrefabFunction that does not produce the exact same output for the same input seed is a critical error. This will lead to inconsistent and corrupt world generation.
-   **Ignoring Errors:** The `get` method throws an Error, not an Exception. This signals a catastrophic, unrecoverable failure in the world generation pipeline. Attempting to catch and suppress this Error will mask fundamental problems and likely lead to server instability. The process should be allowed to fail.
-   **Incorrect Scoping:** Creating a new UniquePrefabCache for every chunk generation request completely defeats its purpose and will severely degrade performance. An instance should be shared across all generation tasks for a given context.

## Data Pipeline
The flow of data is managed internally, abstracting the caching logic from the world generator.

> Flow:
> World Generator requests data for seed -> **UniquePrefabCache.get(seed)** -> Cache Hit?
> - **Yes:** Return cached UniquePrefabEntry array
> - **No:** Invoke UniquePrefabFunction -> Calculate Prefab Data -> Store result in cache -> Return new UniquePrefabEntry array

