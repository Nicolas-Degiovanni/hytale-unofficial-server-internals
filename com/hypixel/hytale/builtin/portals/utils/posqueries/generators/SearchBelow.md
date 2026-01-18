---
description: Architectural reference for SearchBelow
---

# SearchBelow

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.generators
**Type:** Transient

## Definition
```java
// Signature
public class SearchBelow implements SpatialQuery {
```

## Architecture & Concepts
The SearchBelow class is a concrete implementation of the **Strategy** pattern, conforming to the SpatialQuery interface. Its sole responsibility is to generate a finite stream of candidate 3D coordinates for a spatial search operation.

Architecturally, it serves as a *candidate generator* within the portal placement and validation system. It does not perform any checks or world lookups itself. Instead, it provides the set of locations *to be checked* by a higher-level query executor. This decouples the definition of a search area (the *what*) from the logic that validates positions within that area (the *how*).

This component is designed to produce a simple, linear search path: a vertical column extending downwards from a given origin point. It is a fundamental building block often combined with other SpatialQuery implementations to define more complex search volumes.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by systems that define portal search logic. It is not a managed service and is created via a direct call to its constructor: `new SearchBelow(height)`.
- **Scope:** The object's lifetime is exceptionally brief. It is typically created, its `createCandidates` method is called once to produce a stream, and it is then immediately eligible for garbage collection. It holds no state that needs to persist beyond a single query operation.
- **Destruction:** Handled automatically by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** This class is **immutable**. Its only state, the `height` field, is final and is set exclusively during construction. It does not cache data or modify its internal state after creation.
- **Thread Safety:** SearchBelow is inherently **thread-safe**. Due to its immutable nature, a single instance can be safely shared and used across multiple threads without locks or synchronization. The `createCandidates` method is fully re-entrant.

## API Surface
The public API consists only of the constructor and the interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SearchBelow(int height) | constructor | O(1) | Constructs a new search strategy with a maximum downward search distance. |
| createCandidates(World, Vector3d, SpatialQueryDebug) | Stream<Vector3d> | O(N) | Generates a lazy stream of N candidate positions, where N is `height + 1`. |

## Integration Patterns

### Standard Usage
SearchBelow is intended to be used as a component in a larger query system. An instance is created to define the search volume and is then passed to an executor which consumes the generated stream of candidate positions.

```java
// Define a search strategy to check 10 blocks below the origin
SpatialQuery verticalSearch = new SearchBelow(10);
Vector3d origin = new Vector3d(100, 64, 250);

// A hypothetical consumer processes the candidates
// This consumer would perform the actual block checks at each position
Stream<Vector3d> candidates = verticalSearch.createCandidates(world, origin, null);
Optional<Vector3d> validPosition = candidates
    .filter(pos -> world.isBlockSolid(pos))
    .findFirst();
```

### Anti-Patterns (Do NOT do this)
- **Excessive Height:** Do not instantiate this class with an extremely large `height` value. While the stream generation is lazy, a large value can lead to a downstream consumer processing an enormous number of positions. This can cause significant performance degradation and server tick lag, especially if the subsequent validation logic is complex.

    **WARNING:** A search height in the thousands can easily stall the thread performing the query. Always cap search parameters to reasonable, gameplay-defined limits.

- **Unnecessary Instantiation:** While cheap to create, avoid creating new SearchBelow instances inside a tight loop if the height parameter is constant. An instance can be created once and reused for multiple queries at different origins.

## Data Pipeline
SearchBelow acts as a producer at the beginning of a data processing pipeline. It transforms a single origin point into a stream of many candidate points for further processing.

> Flow:
> Portal Placement Logic -> **new SearchBelow(height)** -> `createCandidates(origin)` -> Stream of `Vector3d` -> Position Validator (e.g., block checker) -> Validated Position or Failure

---

