---
description: Architectural reference for ZonePatternGeneratorCache
---

# ZonePatternGeneratorCache

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Transient

## Definition
```java
// Signature
public class ZonePatternGeneratorCache {
```

## Architecture & Concepts
The ZonePatternGeneratorCache is a critical performance component within the server-side world generation pipeline. Its primary function is to act as a memoization layer for ZonePatternGenerator instances, which are computationally expensive to create.

This class implements a classic cache-aside pattern. It decouples the consumers of zone patterns (e.g., biome or terrain generators) from the underlying provider system. By ensuring that a generator for a given integer seed is created only once per session, it dramatically reduces redundant computation, especially in a highly concurrent environment where multiple worker threads may request the same generator simultaneously.

The core of its design relies on a ConcurrentHashMap, making it an essential building block for scalable and parallelized world generation.

### Lifecycle & Ownership
-   **Creation:** An instance of ZonePatternGeneratorCache is typically created by a higher-level world generation coordinator or manager at the beginning of a world generation session. It is injected with a ZonePatternProvider, which acts as a factory for creating generators on a cache miss.
-   **Scope:** The cache is session-scoped. Its lifetime is bound to the specific world or dimension being generated. It is not a global singleton and should not be treated as such. A separate instance would exist for each independent world generation process.
-   **Destruction:** The object has no explicit destruction or cleanup logic. It is subject to standard Java garbage collection once the world generation session it belongs to is complete and all references to it are dropped.

## Internal State & Concurrency
-   **State:** The internal state is mutable, consisting of a map that grows on-demand as new seeds are requested. The cache is write-heavy during the initial phases of world generation and becomes read-heavy as the world expands and most common generators have been cached.
-   **Thread Safety:** This class is fully thread-safe and designed for high-contention access. All internal state management is handled by a ConcurrentHashMap and its atomic **computeIfAbsent** method. This guarantees that even if multiple threads request the same uncached seed simultaneously, the underlying compute function (provider::createGenerator) will be invoked exactly once for that seed. No external locking is required by the caller.

## API Surface
The public contract is minimal, exposing a single method for retrieving generators.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed) | ZonePatternGenerator | O(1) amortized | Retrieves a generator for the given seed. On a cache miss, it atomically computes, stores, and returns a new generator. Throws a fatal Error if the underlying provider fails. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve the cache from a shared context within a world generation worker thread and request generators as needed.

```java
// Within a world generation worker or task
ZonePatternGeneratorCache cache = worldGenContext.getGeneratorCache();
ZonePatternGenerator generator = cache.get(zoneSeed);

// ... use the generator to build terrain or place features
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use **new ZonePatternGeneratorCache()** within worker threads. This defeats the purpose of a shared cache, leading to severe performance degradation as each worker builds its own redundant cache.
-   **Per-Request Caching:** Do not create a new cache for each chunk or region being generated. The cache is designed to be long-lived for the duration of a world generation session.
-   **Ignoring Errors:** The **get** method throws a Java Error on failure, not an Exception. This indicates a catastrophic, unrecoverable configuration or state issue. While it cannot be caught by a standard catch block for Exception, failure to log or handle this condition at the top level of the thread will result in a silent thread death.

## Data Pipeline
The component acts as a high-speed intermediary between a generator request and the generator provider system.

> Flow:
> World Gen Worker -> Request for seed -> **ZonePatternGeneratorCache.get(seed)** -> (Cache Miss) -> ZonePatternProvider.createGenerator -> (Cache Write) -> Return ZonePatternGenerator instance

