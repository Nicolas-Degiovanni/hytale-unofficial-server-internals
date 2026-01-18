---
description: Architectural reference for NotNearPointXZ
---

# NotNearPointXZ

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.predicates
**Type:** Transient

## Definition
```java
// Signature
public final class NotNearPointXZ implements PositionPredicate {
```

## Architecture & Concepts
NotNearPointXZ is a concrete implementation of the **Strategy Pattern**, where the PositionPredicate interface defines the contract for a spatial condition. This class encapsulates a single, highly optimized rule: determining if a given point is outside a specific cylindrical area of exclusion.

Its primary role is to act as a filter within a larger position-finding or validation system, such as one used for procedural world generation or dynamic object placement (e.g., portals, structures).

The key architectural decision is the explicit focus on a 2D, XZ-plane comparison. By ignoring the Y-axis, it provides a significant performance advantage for scenarios where only horizontal proximity is relevant. This is common in world generation logic where vertical distance is often unconstrained or handled by separate predicates. The pre-calculation and storage of the squared radius in the constructor is a critical micro-optimization, avoiding expensive square root operations within the test method, which is expected to be called in high-frequency loops.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level service responsible for executing spatial queries. It is created directly using its constructor, for example: `new NotNearPointXZ(center, radius)`.
- **Scope:** Transient and short-lived. An instance typically exists only for the duration of a single, composite query operation. It is effectively a value object representing a query parameter.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the query operation that created it completes and no further references to it are held.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal fields for the center point and squared radius are private, final, and set only once during construction. The test method operates on this state but never modifies it, creating a temporary clone of the point vector to perform its calculation.
- **Thread Safety:** This class is unconditionally thread-safe. Due to its immutable nature, a single instance can be safely shared and used by multiple threads concurrently without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NotNearPointXZ(point, radius) | constructor | O(1) | Constructs a predicate that defines a 2D exclusion zone. |
| test(world, origin) | boolean | O(1) | Returns true if the origin point is outside the defined radius on the XZ plane. |

## Integration Patterns

### Standard Usage
This class is not intended to be used in isolation. It should be instantiated and passed as a behavioral parameter to a more complex system that iterates through candidate positions and uses the predicate to filter them.

```java
// A hypothetical service finds a valid spawn point using the predicate
PositionQueryService queryService = context.getService(PositionQueryService.class);

Vector3d existingPortalLocation = new Vector3d(1204.5, 75.0, -300.0);
double minSeparationDistance = 64.0;

// Create the predicate to enforce that a new position is not horizontally
// close to the existing portal.
PositionPredicate predicate = new NotNearPointXZ(existingPortalLocation, minSeparationDistance);

// The service uses the predicate to validate potential locations
Optional<Vector3d> newLocation = queryService.findFirstValidLocation(searchArea, predicate);
```

### Anti-Patterns (Do NOT do this)
- **Assuming 3D Check:** Do not use this predicate when true 3D spherical distance is required. This class will incorrectly return true for a point directly above or below the center point, even if it is only one block away vertically.
- **State Modification:** Do not attempt to modify an instance after creation. The design is immutable; if the exclusion zone needs to change, create a new NotNearPointXZ instance.

## Data Pipeline
NotNearPointXZ functions as a boolean filter within a data-processing stream. It does not transform data but instead gates its flow.

> Flow:
> Position Candidate Generator -> **NotNearPointXZ.test(candidate)** -> Boolean Result -> Query Aggregator (Accept/Reject Candidate)

