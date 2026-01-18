---
description: Architectural reference for SearchCone
---

# SearchCone

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.generators
**Type:** Utility

## Definition
```java
// Signature
public class SearchCone implements SpatialQuery {
```

## Architecture & Concepts
The SearchCone class is a concrete implementation of the **Strategy** pattern, conforming to the SpatialQuery interface. Its sole responsibility is to act as a candidate generator for spatial search operations that require finding a point within a three-dimensional conical volume.

Architecturally, it is a self-contained, immutable configuration object. It does not perform any world lookups or validation of the positions it generates. Instead, it produces a lazy stream of potential coordinates based on its configured parameters: direction, radius, angle, and number of attempts. This decouples the *generation* of candidate points from the *validation* of those points, allowing consuming systems like portal placement or procedural spawning services to apply their own complex validation logic to the stream.

The generation algorithm is probabilistic, not exhaustive. It randomly samples a finite number of points within the cone's volume. This design is optimized for performance in scenarios where finding *any* suitable point is sufficient, rather than finding the *optimal* or *all possible* points.

## Lifecycle & Ownership
-   **Creation:** A SearchCone is instantiated on-demand by a higher-level service that needs to execute a spatial search. It is a transient object, created with specific parameters for a single, discrete query operation.
-   **Scope:** The object's lifetime is typically method-scoped. It exists only for the duration of the search algorithm that uses it.
-   **Destruction:** As a simple data object with no external resources, it is automatically garbage collected once it falls out of scope. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** The SearchCone is **immutable**. All its internal fields are private and final, set exclusively at construction time. It does not cache data or change its state after being created.
-   **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that multiple threads can share and operate on a single instance without risk of data corruption or race conditions. The use of ThreadLocalRandom for point generation ensures high-performance, contention-free random number generation in multi-threaded server environments.

## API Surface
The public contract is minimal, consisting of constructors and the single interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SearchCone(direction, radius, maxDegrees, attempts) | constructor | O(1) | Creates a cone where the minimum and maximum search radii are identical. |
| SearchCone(direction, minRadius, maxRadius, maxDegrees, attempts) | constructor | O(1) | Creates a cone with a distinct minimum and maximum search radius. |
| createCandidates(world, origin, debug) | Stream<Vector3d> | O(N) | Returns a lazy, finite stream of N candidate points, where N is the number of attempts. The stream generates points on demand. |

## Integration Patterns

### Standard Usage
A SearchCone should be instantiated to define the search space, then its `createCandidates` method should be used to generate a stream of points. This stream is then typically processed by a consumer that applies validation logic and uses a short-circuiting terminal operation to find the first suitable candidate.

```java
// Define the search parameters for a portal
Vector3d searchDirection = player.getLookVector();
int searchAttempts = 50;
double searchRadius = 20.0;
double searchAngle = 30.0;

// Create the query strategy
SpatialQuery query = new SearchCone(searchDirection, searchRadius, searchAngle, searchAttempts);

// Execute the query and process the stream of candidates
Stream<Vector3d> candidates = query.createCandidates(world, player.getPosition(), null);

// Find the first valid, non-solid location
Optional<Vector3d> validPosition = candidates
    .filter(pos -> world.isBlockSolid(pos) == false)
    .findFirst();

if (validPosition.isPresent()) {
    // A suitable location was found
    world.spawnPortalAt(validPosition.get());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation for Generic Use:** Do not create a single, static SearchCone instance to be reused for different searches. Each instance is a specific configuration for a specific query. Always create a new instance with the correct parameters for the operation at hand.
-   **Expecting Exhaustive Search:** Do not assume this class will find a valid position if one exists. It is a probabilistic sampler. If a search fails, the correct response is often to retry the operation or broaden the search parameters, not to increase the attempt count to an extreme value.
-   **Materializing the Entire Stream:** Avoid collecting all candidates into a list (e.g., using `collect(Collectors.toList())`) unless you genuinely need to process every single generated point. This defeats the performance benefit of the lazy stream and can generate significant garbage. Use short-circuiting operations like `findFirst()` or `anyMatch()` whenever possible.

## Data Pipeline
The SearchCone acts as the origin point in a data processing pipeline. It generates raw data (candidate points) that is then refined by subsequent steps.

> Flow:
> **SearchCone** (Configuration) -> `createCandidates()` -> Lazy Stream of `Vector3d` -> Consumer Filtering (e.g., `world.isBlockSolid`) -> Terminal Operation (e.g., `findFirst`) -> Optional `Vector3d` (Final Position)

