---
description: Architectural reference for SpatialQuery
---

# SpatialQuery

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries
**Type:** Functional Interface / Strategy Pattern

## Definition
```java
// Signature
public interface SpatialQuery {
    // ... default methods
    Stream<Vector3d> createCandidates(World var1, Vector3d var2, @Nullable SpatialQueryDebug var3);
}
```

## Architecture & Concepts
The SpatialQuery interface defines a contract for procedural position-finding operations within the game world. It is a core component of systems that need to dynamically locate suitable coordinates, such as portal placement, structure generation, or AI spawning.

Architecturally, this interface embodies the **Strategy** and **Builder** patterns. It decouples the *definition* of a spatial search from its *execution*. Each implementation of SpatialQuery represents a specific search strategy (e.g., find points in a sphere, find points on a surface). The default methods, *then* and *filter*, allow these strategies to be chained together, forming a powerful and declarative query pipeline.

The system is built upon Java Streams, promoting a lazy-evaluation model. Candidate positions are generated and tested on-demand, which is critical for performance when searching large world volumes. The query is not executed until a terminal operation like *findFirst* (via the *execute* method) is called.

## Lifecycle & Ownership
- **Creation:** Implementations of SpatialQuery are typically instantiated on-demand by higher-level systems that require a position search. They are lightweight, transient objects representing a single, specific search configuration.
- **Scope:** The scope of a SpatialQuery object is ephemeral. It exists only for the duration of the query construction and execution. It does not persist and holds no long-term state related to the world.
- **Destruction:** Once the stream is consumed and the result is obtained, the query object and its composed chain are eligible for garbage collection. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The SpatialQuery interface itself is stateless. Concrete implementations, however, may be configured with parameters (e.g., a radius for a sphere search) that constitute their internal state. This state is generally immutable after creation.
- **Thread Safety:** **This interface is not inherently thread-safe.** The safety of a query execution is entirely dependent on the thread-safety of the World object passed to it. Executing a query that reads or modifies world state from an asynchronous thread without proper synchronization with the main game loop will lead to race conditions, chunk corruption, and engine instability.

**WARNING:** All SpatialQuery execution should be performed on the main server thread unless the underlying World implementation guarantees thread-safe region access.

## API Surface
The public contract is designed as a fluent API for building and executing search pipelines.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createCandidates(world, origin, debug) | Stream | Varies | **Core Method.** Generates a lazy stream of candidate positions. The complexity is determined by the specific implementation. |
| then(expand) | SpatialQuery | Compositional | Chains another query, effectively performing a flatMap operation. Each candidate from the current query becomes an origin for the next. |
| filter(predicate) | SpatialQuery | Compositional | Applies a predicate to the stream of candidates, discarding those that do not match. |
| execute(world, origin) | Optional | Varies | A terminal operation. Executes the full query pipeline and returns the first valid position found, or an empty Optional if none exists. |
| debug(world, origin) | Optional | Varies | Executes the query while logging each step of the pipeline to the console. Incurs a significant performance penalty and should not be used in production code. |

## Integration Patterns

### Standard Usage
A developer should chain query components together to define a precise search, then execute it to retrieve a result. This declarative style is the intended use case.

```java
// How a developer should normally use this
// Assume SphereQuery and IsValidSpawnPoint are concrete implementations.

SpatialQuery findSpawnPoint = new SphereQuery(32)
    .filter(new IsOnSolidGround())
    .filter(new HasClearance(3));

Optional<Vector3d> result = findSpawnPoint.execute(world, player.getPosition());

result.ifPresent(pos -> world.spawnEntity(pos, "MyEntity"));
```

### Anti-Patterns (Do NOT do this)
- **Expensive Filtering:** Do not perform computationally expensive checks in a *filter* operation early in the chain when the candidate set is large. Structure queries to reject large numbers of invalid positions with cheap checks first.
- **Premature Stream Collection:** Avoid materializing the entire stream of candidates with `collect(Collectors.toList())`. This defeats the core performance benefit of lazy evaluation and can cause massive memory allocation and server stalls if the initial candidate set is large.
- **Asynchronous Execution:** Do not execute a query from a separate thread without a deep understanding of the engine's threading model. Modifying or reading from the World object off-thread is extremely dangerous.

## Data Pipeline
The SpatialQuery interface facilitates a clear, linear data pipeline where a stream of 3D vectors is progressively refined until a suitable candidate is found.

> Flow:
> Initial Generator (e.g., SphereQuery) -> Stream of many Vector3d -> **filter(PredicateA)** -> Smaller Stream of Vector3d -> **then(SubQuery)** -> Transformed Stream of Vector3d -> **findFirst()** -> Optional Vector3d result

