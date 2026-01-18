---
description: Architectural reference for FlatMapQuery
---

# FlatMapQuery

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.predicates.generic
**Type:** Utility

## Definition
```java
// Signature
public class FlatMapQuery implements SpatialQuery {
```

## Architecture & Concepts
The FlatMapQuery is a higher-order, compositional operator within the Spatial Query framework. It does not generate candidate positions on its own; instead, it acts as a powerful combinator that links two separate SpatialQuery instances into a sequential, dependent search operation.

Its core architectural purpose is to enable multi-stage spatial searches. It functions analogously to the *flatMap* operation found in functional programming paradigms, but applied to streams of 3D world coordinates.

The system operates in two phases:
1.  **Generation:** An initial query, the *generator*, is executed to produce a stream of primary candidate positions.
2.  **Expansion:** For each primary candidate produced by the generator, a second query, the *expand* query, is executed using that candidate's position as its new origin.

All resulting streams from the expansion phase are then flattened into a single, unified stream of final candidates. This pattern is essential for complex procedural generation and location-finding tasks, such as "find a 5x5 flat area within a 100-block radius of the player."

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor (`new FlatMapQuery(...)`) by a system that is dynamically composing a complex spatial search. It is not a managed service and is not retrieved from a central registry.
-   **Scope:** Transient and short-lived. A FlatMapQuery instance typically exists only for the duration of a single, composite query execution. Once the resulting stream is consumed, the object is eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** Immutable. The internal references to the *generator* and *expand* queries are final and are set at construction time. The class holds no mutable state, making its behavior deterministic based on its inputs.
-   **Thread Safety:** The FlatMapQuery class itself is inherently thread-safe. However, the overall thread safety of an operation involving FlatMapQuery is contingent upon the thread safety of the composed *generator* and *expand* queries. If those underlying queries are safe, the resulting stream can be processed in parallel without issue.

**WARNING:** Passing stateful or non-reentrant SpatialQuery implementations to the constructor can lead to unpredictable and erroneous behavior, especially if the stream is processed in parallel.

## API Surface
The public contract consists of a single method inherited from the SpatialQuery interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createCandidates(world, origin, debug) | Stream<Vector3d> | O(N \* M) | Executes the composite query. Complexity is the product of candidates from the generator (N) and the average candidates from the expander (M). |

## Integration Patterns

### Standard Usage
The primary use case is to chain two queries where the second depends on the output of the first. This is the canonical way to build sophisticated, multi-step location searches.

```java
// Example: Find a suitable location for a structure.
// 1. First, find any solid ground block in a large radius.
SpatialQuery generator = new FindSurfaceQuery(SearchVolume.sphere(100));

// 2. For each ground block found, check for a 3x3 clear area above it.
SpatialQuery expander = new CheckClearanceQuery(new Vector3i(3, 3, 3), Vector3i.UNIT_Y);

// 3. Combine them into a single, powerful query.
SpatialQuery findStructureSpotQuery = new FlatMapQuery(generator, expander);

// 4. Execute the query relative to the player's position.
Stream<Vector3d> validSpots = findStructureSpotQuery.createCandidates(world, player.getPosition(), null);
Vector3d firstValidSpot = validSpots.findFirst().orElse(null);
```

### Anti-Patterns (Do NOT do this)
-   **Deep Nesting:** Avoid chaining multiple FlatMapQuery instances within each other (e.g., `new FlatMapQuery(new FlatMapQuery(...), ...)`). While functionally possible, this can create highly complex and inefficient queries that are difficult to debug and may lead to performance bottlenecks. Prefer to design simpler, more direct query components.
-   **Infinite Generators:** Never use a generator query that can produce an infinite or excessively large stream of candidates. This will result in the expander being called an unbounded number of times, likely causing the server to hang or crash. Always use generators with well-defined, finite search volumes.

## Data Pipeline
FlatMapQuery acts as a transformation stage within a data stream. It consumes a stream of positions and produces a new, more refined stream of positions.

> Flow:
> Initial Origin -> **Generator Query** -> Stream of Primary Candidates -> **FlatMapQuery** -> (For each Primary Candidate) -> **Expand Query** -> Stream of Streams -> Flattening Operation -> Final Stream of Candidates

