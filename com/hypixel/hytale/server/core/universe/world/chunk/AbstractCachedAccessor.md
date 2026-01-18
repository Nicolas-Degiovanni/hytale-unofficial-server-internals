---
description: Architectural reference for AbstractCachedAccessor
---

# AbstractCachedAccessor

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Base Class (Transient State Object)

## Definition
```java
// Signature
public abstract class AbstractCachedAccessor {
```

## Architecture & Concepts
The AbstractCachedAccessor is a high-performance, localized cache for world data. Its primary architectural role is to serve as a performance acceleration layer for systems that perform numerous, spatially-coherent data lookups within a constrained 3D volume of the world. Systems like world generation, block physics simulations, or AI environmental queries are typical consumers.

It implements a **cache-aside** strategy. When a request for a chunk, section, or component is made, the accessor first checks its internal arrays. On a cache hit, the data is returned immediately, avoiding expensive lookups to the canonical data source. On a cache miss, the accessor delegates the request to the authoritative `ChunkStore` via a `ComponentAccessor`, populates its internal cache with the result, and then returns the data.

This component is designed as a "hot" cache, intended for a single, focused operation. It is not a long-term or persistent caching solution. Its state represents a temporary working set of world data for a specific task. Subclasses are expected to extend this class to implement specific logic that benefits from this accelerated local data access.

## Lifecycle & Ownership
- **Creation:** Subclasses are instantiated by server systems performing localized world queries. The constructor only performs minimal setup for component arrays. The object is considered uninitialized and non-operational until the `init` method is invoked.

- **Scope:** The lifecycle is intentionally brief and bound to a single, discrete operation. The `init` method configures the accessor for a specific center point and radius. Once the operation is complete, the object should be discarded. It is not designed to be held as a long-lived service or reused for different world locations without re-initialization.

- **Destruction:** The class has no explicit destruction or cleanup method. It relies entirely on the Java Garbage Collector for memory reclamation once all external references are released.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The `chunks`, `sections`, and `sectionComponents` arrays are lazily populated on-demand as data is requested. The state is entirely transient and should be considered invalid after the owning operation completes. The size of the internal arrays is determined at runtime by the `init` method and is proportional to the specified radius.

- **Thread Safety:** **CRITICAL WARNING:** This class is fundamentally **not thread-safe**. It contains no internal locks, volatile fields, or other synchronization primitives. All method calls on a given instance **must** be confined to a single thread. Concurrent access, especially a mix of reads and writes (lazy initialization), will result in race conditions, data corruption, and non-deterministic engine behavior. The design strongly implies its use within a single, synchronous game loop or world update tick.

## API Surface
The public and protected API provides access to the cached world data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(accessor, cx, cy, cz, radius) | protected void | O(R³) | Configures the cache for a cubic region. Resets all internal arrays. **Must be called before any get operations.** |
| getChunk(cx, cz) | public Ref<ChunkStore> | O(1) | Retrieves a chunk reference. On cache miss, fetches from the ComponentAccessor and populates the cache. |
| getSection(cx, cy, cz) | public Ref<ChunkStore> | O(1) | Retrieves a chunk section reference. On cache miss, fetches and caches. Returns null for out-of-bounds Y coordinates. |
| getComponentSection(cx, cy, cz, typeIndex, type) | protected T | O(1) | Retrieves a specific Component from a chunk section. Implements the same cache-aside logic as other get methods. |

## Integration Patterns

### Standard Usage
Subclasses extend AbstractCachedAccessor to create specialized tools, such as a world generator. The `init` method is called to define the area of interest, followed by intensive data access via the `get` methods.

```java
// A hypothetical subclass for world generation
public class BiomeGenerator extends AbstractCachedAccessor {

    public BiomeGenerator() {
        super(1); // Caching 1 component type
    }

    public void generate(ComponentAccessor<ChunkStore> commands, int cx, int cy, int cz) {
        // Initialize the cache for a 5-chunk radius
        super.init(commands, cx, cy, cz, 5);

        // Perform intensive, localized reads
        for (int x = -5; x <= 5; x++) {
            for (int z = -5; z <= 5; z++) {
                // Access is fast due to the cache
                Ref<ChunkStore> section = getSection(cx + x, cy, cz + z);
                // ... generation logic ...
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not hold a reference to an accessor and use it for a new operation in a different part of the world without calling `init` again. The internal state will be stale and incorrect.
- **Multi-threaded Access:** Do not share an instance of this class or its subclasses across multiple threads. This will lead to severe data corruption. Each thread performing a world operation requires its own dedicated accessor instance.
- **Ignoring Initialization:** Calling any `get` method before `init` has been successfully invoked will result in a NullPointerException or incorrect behavior.

## Data Pipeline
This class does not process a stream of data but rather acts as an intermediary in a request-response data access pattern.

> Flow:
> World System -> `getSection(x,y,z)` -> **AbstractCachedAccessor Cache Check** -> (On Cache Miss) -> `ComponentAccessor` -> `ChunkStore` -> Return Data & Populate Cache -> World System

