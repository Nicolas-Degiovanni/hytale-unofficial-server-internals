---
description: Architectural reference for SearchCircular
---

# SearchCircular

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.generators
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class SearchCircular implements SpatialQuery {
```

## Architecture & Concepts
SearchCircular is a concrete implementation of the SpatialQuery strategy interface. Its role within the engine is to act as a candidate position generator for world-space searches. It is a foundational component used by higher-level systems, such as portal or structure placement services, to find suitable locations without performing exhaustive, computationally expensive scans of large world volumes.

The core architectural concept is **probabilistic sampling**. Instead of iterating through every block in a region, SearchCircular generates a finite stream of random points within a specified circular ring (or a simple circle if min and max radius are equal). This approach trades the guarantee of finding a location for a significant performance increase, which is critical for real-time server operations. The effectiveness of the search is controlled by the number of attempts, allowing developers to balance performance against the probability of success.

It operates purely on mathematical principles and does not interact with world data itself; it only generates potential coordinates. The consumer of the generated stream is responsible for validating each candidate position against the actual world state.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by a service or system that needs to perform a spatial search. It is configured with its search parameters (radius, attempts) at the time of construction. It is not managed by a dependency injection container or service registry.
-   **Scope:** The object's lifetime is ephemeral and typically confined to the scope of a single method call. It is created, used to generate a stream, and then immediately becomes eligible for garbage collection.
-   **Destruction:** Managed automatically by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
-   **State:** This object is **effectively immutable**. Its internal fields for minimum radius, maximum radius, and attempts are final and are set exclusively during construction. It holds no mutable state, caches no data, and its behavior is entirely deterministic based on its initial configuration and the random number generator.
-   **Thread Safety:** The class is inherently thread-safe. Its immutable nature prevents data races. The `createCandidates` method leverages ThreadLocalRandom to ensure that concurrent calls from different threads do not suffer from lock contention on a shared random number generator. A single SearchCircular instance can be safely shared and used by multiple threads, though the standard usage pattern is to create a new instance for each distinct search operation.

## API Surface
The public contract is minimal, focused entirely on fulfilling the SpatialQuery interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createCandidates(world, origin, debug) | Stream<Vector3d> | O(1) | Returns a lazy-evaluated stream of candidate positions. The creation of the stream object itself is a constant-time operation. The actual work is performed as the stream is consumed. |

## Integration Patterns

### Standard Usage
SearchCircular is designed to be the starting point of a stream-based processing pipeline. A service creates an instance, generates the candidate stream, and then chains subsequent operations like filtering and validation.

```java
// A higher-level service uses SearchCircular to find a valid spawn location.
// The query is configured to search in a ring between 20 and 40 blocks away, with 150 attempts.
SpatialQuery query = new SearchCircular(20.0, 40.0, 150);

// The stream is consumed to find the first candidate that passes a suitability check.
Optional<Vector3d> validLocation = query.createCandidates(world, origin, null)
    .filter(candidate -> world.isBlockSolid(candidate.floor()))
    .findFirst();

if (validLocation.isPresent()) {
    // Proceed with placement...
}
```

### Anti-Patterns (Do NOT do this)
-   **Insufficient Attempts:** Configuring the query with a very low number of attempts for a large search radius drastically reduces the probability of finding a valid location, even if many exist. This can lead to frequent, difficult-to-debug placement failures.
-   **Zero Radius Search:** Creating a SearchCircular with a maxRadius of zero will always generate points at the exact origin, which is rarely the desired behavior.
-   **Ignoring the Stream:** The `createCandidates` method is lazy. Calling it without attaching a terminal stream operation (like `findFirst`, `collect`, or `forEach`) performs no work and has no effect.

## Data Pipeline
SearchCircular acts as a data source or generator. It does not process incoming data; it creates it. The typical flow of data originating from this class is as follows:

> Flow:
> Configuration (Radius, Attempts) -> **SearchCircular** -> Stream of `Vector3d` Candidates -> Stream Filtering (e.g., `isLocationSuitable`) -> Terminal Operation (e.g., `findFirst`) -> Optional `Vector3d` Result

