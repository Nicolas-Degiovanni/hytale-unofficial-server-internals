---
description: Architectural reference for SimplePathWaypoint
---

# SimplePathWaypoint

**Package:** com.hypixel.hytale.server.core.universe.world.path
**Type:** Transient

## Definition
```java
// Signature
public class SimplePathWaypoint implements IPathWaypoint {
```

## Architecture & Concepts
The SimplePathWaypoint is a concrete implementation of the IPathWaypoint interface, representing a static, immutable point in a path sequence. It serves as a fundamental data structure for the server-side entity pathfinding and movement systems.

Its primary architectural role is to provide a fixed coordinate and orientation in world space. The "Simple" designation distinguishes it from more complex waypoint implementations that might calculate their position dynamically based on an entity's state or other world factors.

A critical design characteristic is its context-free nature. The methods getWaypointPosition and getWaypointRotation intentionally ignore the ComponentAccessor argument, signaling that this waypoint's data is absolute and not relative to any entity or component state. This makes it a lightweight and predictable building block for defining fixed patrol routes, scripted entity movements, and other non-dynamic paths.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by systems responsible for defining paths, such as a world generation script, a quest manager, or a tool that places NPCs. Its entire state, consisting of its order and transform, is provided at construction and is not intended to be changed thereafter.
-   **Scope:** The lifetime of a SimplePathWaypoint instance is bound to the lifetime of the path collection that contains it. It is a value object, not a managed service, and persists only as long as it is referenced by a path.
-   **Destruction:** The object is eligible for garbage collection as soon as the containing path is dereferenced. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
-   **State:** The internal state is **effectively immutable**. The class provides no public methods to alter its order or transform fields post-construction. The state is defined entirely at instantiation.
-   **Thread Safety:** This class is **conditionally thread-safe**. It contains no internal locks or synchronization primitives. Its safety in a multithreaded environment is contingent on the thread safety of the Transform object it holds. If the provided Transform is not modified externally after construction, the SimplePathWaypoint object can be safely read by multiple threads concurrently.

## API Surface
The public API provides read-only access to the waypoint's properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrder() | int | O(1) | Returns the sequence index of this waypoint within a larger path. |
| getWaypointPosition(accessor) | Vector3d | O(1) | Returns the static 3D world coordinate of the waypoint. **Warning:** The ComponentAccessor argument is ignored. |
| getWaypointRotation(accessor) | Vector3f | O(1) | Returns the static 3D orientation of the waypoint. **Warning:** The ComponentAccessor argument is ignored. |
| getPauseTime() | double | O(1) | Returns the time an entity should pause at this waypoint. This implementation always returns 0.0. |
| getObservationAngle() | float | O(1) | Returns a specific angle for an entity to face. This implementation always returns 0.0. |

## Integration Patterns

### Standard Usage
A SimplePathWaypoint is almost always created as part of a larger collection that defines a complete path. This collection is then consumed by a pathfinding or movement component.

```java
// A path is typically a collection of IPathWaypoint objects.
List<IPathWaypoint> patrolPath = new ArrayList<>();

// Define the position and rotation for the first point in the path.
Transform startPoint = new Transform(new Vector3d(100, 64, 250), Vector3f.ZERO);
patrolPath.add(new SimplePathWaypoint(0, startPoint));

// Define the second point.
Transform endPoint = new Transform(new Vector3d(120, 64, 250), Vector3f.ZERO);
patrolPath.add(new SimplePathWaypoint(1, endPoint));

// This path can now be assigned to an entity's pathfinding component to execute.
```

### Anti-Patterns (Do NOT do this)
-   **Post-Construction State Mutation:** Do not modify the Transform object after it has been passed to the SimplePathWaypoint constructor. The pathfinding system caches path data and assumes waypoints are immutable. Modifying the underlying Transform will lead to desynchronization and unpredictable entity behavior. If a change is needed, create a new path with new waypoints.
-   **Reliance on Accessor:** Do not expect the ComponentAccessor argument in getWaypointPosition or getWaypointRotation to have any effect. This implementation is for static points only. If entity-relative positioning is required, a different, more dynamic IPathWaypoint implementation must be used.

## Data Pipeline
SimplePathWaypoint acts as a data source, not a data processor. It holds static information that is requested by higher-level systems.

> Flow:
> Entity Movement System -> Requests next waypoint data from Path -> **SimplePathWaypoint** -> Returns its static Transform -> Movement System calculates velocity and orientation changes.

