---
description: Architectural reference for NotNearPoint
---

# NotNearPoint

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.predicates
**Type:** Transient Predicate

## Definition
```java
// Signature
public final class NotNearPoint implements PositionPredicate {
```

## Architecture & Concepts
The NotNearPoint class is a concrete implementation of the PositionPredicate strategy interface. Its primary architectural role is to serve as a composable spatial rule within a larger position querying framework, most notably for procedural generation and structure placement systems.

This predicate defines a spherical exclusion zone. When evaluated, it returns true for any position *outside* this sphere and false for any position *inside*. This is a fundamental building block for ensuring that generated features, such as portals or dungeons, do not spawn too close to predefined points of interest like player bases, world landmarks, or other critical locations.

The design favors performance by pre-calculating the squared radius during construction. This allows the core test method to operate using squared distance checks, a common optimization that avoids computationally expensive square root operations inside tight generation loops.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by a higher-level system responsible for constructing a spatial query. It is not a managed service and should not be treated as one. For example, a PortalPlacementService would create an instance of NotNearPoint to define a "keep-out" zone around an existing portal.
-   **Scope:** Short-lived. The object's lifetime is typically bound to the execution of a single position-finding operation. It is created, used by the query executor, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
-   **State:** Immutable. The internal state, consisting of the center point and the squared radius, is established in the constructor and cannot be modified thereafter. All fields are declared final to enforce this guarantee.
-   **Thread Safety:** Inherently thread-safe. Due to its immutable nature, a single instance of NotNearPoint can be safely passed to and executed by multiple threads simultaneously without any risk of data corruption or race conditions. This makes it highly suitable for use in parallelized world generation or analysis tasks.

## API Surface
The public contract is minimal, consisting only of the constructor and the interface method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NotNearPoint(point, radius) | constructor | O(1) | Constructs a new predicate with a center point and exclusion radius. |
| test(world, origin) | boolean | O(1) | Returns true if the origin is outside or on the boundary of the exclusion sphere. |

**Warning:** The `world` parameter in the `test` method is currently unused by this specific predicate. Do not rely on it for any side effects.

## Integration Patterns

### Standard Usage
NotNearPoint is not intended to be used in isolation. It should be instantiated and passed as a rule to a position querying system, which will then invoke its `test` method on candidate positions.

```java
// A world generation service needs to find a location for a new structure
// that is at least 500 blocks away from the world spawn point.

Vector3d worldSpawn = new Vector3d(0, 128, 0);
double minDistance = 500.0;

// Create the predicate to define the exclusion zone.
PositionPredicate exclusionRule = new NotNearPoint(worldSpawn, minDistance);

// Pass the rule to a query executor.
// The executor will iterate through candidate positions, calling
// exclusionRule.test(world, candidate) until it finds a valid spot.
Optional<Vector3d> validPosition = positionQueryEngine.findPosition(exclusionRule);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Avoid creating new instances of NotNearPoint inside a tight loop that checks against the same point and radius. If the exclusion zone is constant for the duration of the search, instantiate it once before the loop begins.
-   **Logic Inversion:** Do not misinterpret the class name. It confirms a position is **NOT** near the point. Using it to find a position *inside* a radius will produce incorrect behavior. For that, a corresponding `IsNearPoint` predicate should be used.

## Data Pipeline
As a predicate, NotNearPoint acts as a filter or gate in a data flow. It does not transform data; it either permits or rejects a candidate position based on its internal logic.

> Flow:
> Position Query Engine -> Supplies `Vector3d` candidate -> **NotNearPoint.test(candidate)** -> Returns `boolean` result -> Query Engine accepts or rejects candidate

