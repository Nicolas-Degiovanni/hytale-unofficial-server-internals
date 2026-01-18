---
description: Architectural reference for ExtendedCoordinateCache
---

# ExtendedCoordinateCache

**Package:** com.hypixel.hytale.server.worldgen.cache
**Type:** Abstract Cache Component

## Definition
```java
// Signature
public abstract class ExtendedCoordinateCache<K, T> {
```

## Architecture & Concepts

The ExtendedCoordinateCache is an abstract, high-performance caching layer designed specifically for the demands of server-side world generation. It serves as a foundational component for caching computed data that is dependent on a combination of a generic key, a world seed, and a 2D spatial coordinate.

Its architecture is built upon three core principles to ensure minimal performance overhead and high throughput in a multi-threaded environment:

1.  **Composite Keying:** The cache is indexed by a composite key comprising a generic object K, an integer seed, and an (x, z) coordinate pair. This structure is ideal for memoizing the results of deterministic world generation functions, such as biome calculation or structure placement, which depend on these specific inputs. The (x, z) coordinates are packed into a single long for efficiency.

2.  **Aggressive Object Pooling:** To combat garbage collector pressure, a critical concern in long-running server processes, this class internally manages an ObjectPool of its key objects, ExtendedCoordinateKey. When a cache lookup is performed, a key object is borrowed from the pool, its state is mutated for the current lookup, and it is returned to the pool after the operation. This significantly reduces object churn.

3.  **Load-Through with Dual Eviction Policy:** The cache implements a "load-through" pattern. Callers simply request data via the get method; if the data is not present (a cache miss), the cache transparently invokes a developer-provided `loader` function to compute the value, stores the result, and returns it. The underlying storage, SizedTimeoutCache, employs a dual eviction strategy, removing entries based on both a maximum size and a time-to-live (TTL). This ensures the cache remains bounded in memory and does not hold stale data.

This class is abstract and requires a concrete implementation to provide a thread-safe mechanism for acquiring a pooled key object via the `localKey` method.

### Lifecycle & Ownership

-   **Creation:** An instance of a concrete subclass is created and configured by a higher-level manager, typically a service responsible for a specific aspect of world generation (e.g., a BiomeManager). It is not a globally managed singleton.
-   **Scope:** The lifecycle of an ExtendedCoordinateCache instance is tightly coupled to its owner. For example, a cache associated with a specific world or dimension will persist as long as that world is loaded and active on the server.
-   **Destruction:** The object is eligible for garbage collection once its owning manager is destroyed and all references to it are released. There is no explicit `destroy` method. Cached values can be cleaned up via the optional ExtendedCoordinateRemovalListener, which is invoked upon eviction.

## Internal State & Concurrency

-   **State:** This component is highly stateful and mutable. Its primary state is the collection of key-value pairs managed by the internal SizedTimeoutCache. The contents of this collection are volatile, changing continuously as items are added, accessed, and evicted.
-   **Thread Safety:** The component is designed to be thread-safe, assuming the underlying SizedTimeoutCache is also thread-safe. The primary public method, `get`, can be safely invoked by multiple concurrent world generation threads. The abstract `localKey` method is the critical point for ensuring thread safety in concrete implementations; it is typically implemented using a ThreadLocal to provide each thread with its own reusable key object, thus preventing race conditions during key mutation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(K k, int seed, int x, int y) | T | O(1) / O(loader) | Retrieves the cached value. On a cache hit, this is an O(1) average time operation. On a cache miss, complexity is determined by the provided loader function. |

## Integration Patterns

### Standard Usage

A concrete implementation must be created, typically using a ThreadLocal to manage the key object pool for thread safety. This implementation is then used by a world generation service.

```java
// 1. Create a concrete implementation
public class BiomeCache extends ExtendedCoordinateCache<Dimension, Biome> {
    private final ThreadLocal<ExtendedCoordinateKey<Dimension>> localKey =
        ThreadLocal.withInitial(ExtendedCoordinateKey::new);

    public BiomeCache(int maxSize, long expireAfterSeconds, ExtendedCoordinateObjectFunction<Dimension, Biome> loader) {
        super(loader, null, maxSize, expireAfterSeconds);
    }

    @Override
    protected ExtendedCoordinateKey<Dimension> localKey() {
        return this.localKey.get();
    }
}

// 2. Instantiate and use within a service
BiomeCache biomeCache = new BiomeCache(8192, 300, (dimension, seed, x, z) -> {
    // Complex biome calculation logic here...
    return new Biome("Plains");
});

// 3. Use in a world generation thread
Biome biome = biomeCache.get(currentDimension, worldSeed, chunkX, chunkZ);
```

### Anti-Patterns (Do NOT do this)

-   **Expensive Loader Functions:** The `loader` function is executed synchronously and will block the calling thread on a cache miss. A long-running loader will become a severe performance bottleneck, as other threads waiting for the same or different keys may be stalled. Loaders must be highly optimized and non-blocking.
-   **Caching Mutable Values:** The generic type T should be an immutable object. If a mutable object is cached, two different threads could retrieve a reference to the same object and modify its state concurrently, leading to unpredictable behavior and severe race conditions.
-   **Incorrect `localKey` Implementation:** Failing to implement `localKey` in a thread-safe manner (e.g., by returning a shared, non-synchronized instance) will corrupt the cache and cause catastrophic failures under concurrent load. Using a ThreadLocal is the standard and expected pattern.

## Data Pipeline

The data flow for a `get` operation follows a standard load-through cache pattern.

> Flow:
> World Generation Thread -> `get(k, seed, x, z)` -> **ExtendedCoordinateCache** -> Check internal SizedTimeoutCache
>
> *   **On Cache Hit:** SizedTimeoutCache -> Return existing value `T` -> World Generation Thread
> *   **On Cache Miss:** SizedTimeoutCache -> Invoke `loader` function -> Compute new value `T` -> Store `T` in SizedTimeoutCache -> Return new value `T` -> World Generation Thread

