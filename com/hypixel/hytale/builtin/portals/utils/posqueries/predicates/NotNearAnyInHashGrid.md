---
description: Architectural reference for NotNearAnyInHashGrid
---

# NotNearAnyInHashGrid

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.predicates
**Type:** Predicate / Data Object

## Definition
```java
// Signature
public record NotNearAnyInHashGrid(SpatialHashGrid<?> hashGrid, double radius) implements PositionPredicate {
```

## Architecture & Concepts
NotNearAnyInHashGrid is a specialized, stateless implementation of the PositionPredicate interface. It functions as a behavioral component that encapsulates a specific spatial query rule: "Is a given point sufficiently far away from all other known points?"

Architecturally, this class acts as a bridge between high-level position-finding algorithms and the low-level SpatialHashGrid data structure. It decouples the *intent* of a spatial query (ensuring separation) from the *mechanism* of the query (the hash grid's internal search logic).

Its primary role is within procedural generation and object placement systems. For example, when placing a new portal, structure, or resource node, the system can use this predicate to filter out candidate positions that are too close to existing objects, preventing visual clipping and maintaining desired game design spacing. It is a fundamental building block for enforcing spatial constraints.

## Lifecycle & Ownership
- **Creation:** Instantiated on-the-fly by a higher-level service, such as a PortalPlacementManager or a WorldGenerator. It is constructed with a reference to a pre-populated SpatialHashGrid and a distance value. It does not create or own the hash grid.
- **Scope:** Short-lived and transient. The lifetime of a NotNearAnyInHashGrid instance is typically confined to the scope of a single method call or a single, discrete placement operation. It is a value object, not a persistent service.
- **Destruction:** Eligible for garbage collection as soon as the operation that created it completes. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. As a Java record, its fields, hashGrid and radius, are final and cannot be changed after construction. The object itself holds no other state and performs no caching. Each invocation of the test method is an independent, fresh query against the provided hashGrid.
- **Thread Safety:** Conditionally thread-safe. The NotNearAnyInHashGrid object is immutable and inherently safe to share across threads. However, its operational thread safety is entirely dependent on the thread safety of the SpatialHashGrid instance it references.

    **WARNING:** If the referenced SpatialHashGrid is modified by another thread concurrently with a call to the test method, the behavior is undefined and may result in inconsistent results or runtime exceptions. The caller is responsible for ensuring that the hashGrid is in a stable state during the query or is itself a thread-safe implementation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(world, point) | boolean | O(1) | Evaluates the predicate. Returns true if the point is not within the specified radius of any entry in the hashGrid. The O(1) complexity is contingent on a well-distributed hash grid; worst-case performance degrades with poor hash distribution. |

## Integration Patterns

### Standard Usage
This predicate is designed to be used as a filter within a larger position-searching algorithm. It is instantiated with the set of existing objects and then used to validate new candidate positions.

```java
// Assume 'existingPortalsGrid' is a SpatialHashGrid already populated with portal locations.
SpatialHashGrid<Portal> existingPortalsGrid = world.getPortals().getSpatialGrid();
double minimumSeparation = 100.0;

// Create the predicate for this specific placement operation.
PositionPredicate separationRule = new NotNearAnyInHashGrid(existingPortalsGrid, minimumSeparation);

// Now, test a candidate position.
Vector3d candidatePosition = new Vector3d(150, 64, 230);
if (separationRule.test(world, candidatePosition)) {
    // This position is valid; proceed with portal placement.
    placePortalAt(candidatePosition);
}
```

### Anti-Patterns (Do NOT do this)
- **Grid Mismanagement:** Passing a SpatialHashGrid that is actively being modified on another thread without synchronization. This will lead to race conditions and non-deterministic behavior. The predicate assumes a stable snapshot of the grid for the duration of the test call.
- **Ignoring the Result:** The boolean result of the test method is the sole purpose of this object. Calling it and not acting upon the result is a logical error.
- **Long-Lived Instances:** Holding onto an instance of NotNearAnyInHashGrid after the placement operation is complete. While not harmful due to its small memory footprint, it is semantically incorrect as the state of the underlying hashGrid may have changed, rendering the predicate's reference stale.

## Data Pipeline
This component does not transform data; it acts as a conditional gate in a data query flow.

> Flow:
> Candidate Position (Vector3d) -> **NotNearAnyInHashGrid.test()** -> [Delegation] -> SpatialHashGrid.hasAnyWithin() -> Boolean Result

