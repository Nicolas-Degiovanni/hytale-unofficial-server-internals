---
description: Architectural reference for WrappedBiCoordinateDoubleCache
---

# WrappedBiCoordinateDoubleCache

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.bicoordinatecache
**Type:** Transient

## Definition
```java
// Signature
public class WrappedBiCoordinateDoubleCache implements BiCoordinateCache<Double> {
```

## Architecture & Concepts
The WrappedBiCoordinateDoubleCache is a specialized, fixed-size 2D grid cache designed for high-performance, localized data storage, primarily within procedural generation contexts. Its key architectural feature is its "wrapping" or "toroidal" behavior; world coordinates provided to the cache are mapped via a modulo operation into a local, bounded grid. This makes it an ideal data structure for caching the results of expensive, stateless calculations (like noise functions) that are repeatedly accessed within a finite region.

Internally, the cache is implemented using two parallel 2D arrays:
1.  **values**: A `double[][]` array that stores the actual cached numerical data.
2.  **populated**: A `boolean[][]` array that acts as a presence map, tracking whether a given coordinate in the `values` array contains valid, cached data.

This dual-array design is critical. It disambiguates between a coordinate that has not been cached and a coordinate that has been cached with a default value (e.g., 0.0). A request for an unpopulated coordinate will fail fast, preventing the use of invalid data.

This component is not a general-purpose cache. It is a low-level, performance-oriented data structure intended to be instantiated and discarded on a per-task basis, such as during the generation of a single world chunk or region.

### Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new WrappedBiCoordinateDoubleCache(sizeX, sizeZ)`. The creator, typically a world generator or a noise evaluation system, is responsible for defining the cache's dimensions.
- **Scope:** The object's lifetime is intentionally short and is scoped to the specific, bounded task that requires it. It is designed to be created, populated, heavily read from, and then released for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the system that created it releases its reference. The `flush()` method allows for the cache to be cleared and reused within its scope, but it does not release the underlying memory of the arrays.

## Internal State & Concurrency
- **State:** This class is highly mutable. The internal `values`, `populated`, and `size` fields are modified by `save` and `flush` operations. Its state is entirely dependent on the sequence of API calls made after its creation.
- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. Concurrent calls to `save` or `flush` from multiple threads will result in race conditions, leading to incorrect size tracking, inconsistent state between the `values` and `populated` arrays, and potential data corruption. Any multi-threaded access **must** be managed by external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int x, int z) | Double | O(1) | Retrieves the cached value for the given world coordinates. Throws IllegalStateException if the coordinate is not populated. |
| save(int x, int z, Double value) | Double | O(1) | Caches the given value at the specified world coordinates, marking it as populated. |
| isCached(int x, int z) | boolean | O(1) | Checks if a value exists for the given world coordinates without attempting to retrieve it. |
| flush() | void | O(N*M) | Invalidates the entire cache by resetting the populated flags. N and M are the cache dimensions. |
| flush(int x, int z) | void | O(1) | Invalidates a single entry at the specified world coordinates. |
| localXFrom(int x) | int | O(1) | Public utility to compute the wrapped local X coordinate from a world X coordinate. |
| localZFrom(int z) | int | O(1) | Public utility to compute the wrapped local Z coordinate from a world Z coordinate. |

## Integration Patterns

### Standard Usage
The canonical use case is to wrap an expensive calculation. The caller first checks if a value is cached. If not, it performs the calculation, saves the result to the cache, and then proceeds. Subsequent requests for the same coordinate will hit the cache.

```java
// A generator creates a cache for its immediate task
WrappedBiCoordinateDoubleCache noiseCache = new WrappedBiCoordinateDoubleCache(16, 16);
int worldX = 1024;
int worldZ = 512;

double value;
if (noiseCache.isCached(worldX, worldZ)) {
    value = noiseCache.get(worldX, worldZ);
} else {
    value = SomeExpensiveNoise.calculate(worldX, worldZ);
    noiseCache.save(worldX, worldZ, value);
}
// ... use the value
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not share a single instance across multiple threads without external locking. The internal state will become corrupted.
- **Get-Before-Check:** Do not call `get(x, z)` without first confirming the entry exists with `isCached(x, z)`. This will result in an `IllegalStateException` and halt execution.
- **Long-Term Storage:** Do not use this class as a long-lived, application-scoped cache. It is designed for transient, task-scoped memoization and does not have mechanisms for eviction or memory management beyond a full flush.

## Data Pipeline
This component typically acts as a memoization layer within a larger data generation process, creating a short feedback loop to prevent redundant computation.

> Flow:
> 1. Generator requests data for coordinate (X, Z).
> 2. Generator queries **WrappedBiCoordinateDoubleCache** with `isCached(X, Z)`.
> 3. **Cache Miss:**
>    a. Generator executes an expensive calculation (e.g., noise function).
>    b. Generator stores the result in the **WrappedBiCoordinateDoubleCache** via `save(X, Z, result)`.
> 4. **Cache Hit:**
>    a. Generator retrieves the pre-computed value from the **WrappedBiCoordinateDoubleCache** via `get(X, Z)`.
> 5. Generator uses the value for further processing.

