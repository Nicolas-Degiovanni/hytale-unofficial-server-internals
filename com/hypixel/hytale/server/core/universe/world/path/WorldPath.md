---
description: Architectural reference for WorldPath
---

# WorldPath

**Package:** com.hypixel.hytale.server.core.universe.world.path
**Type:** Data Model

## Definition
```java
// Signature
public class WorldPath implements IPath<SimplePathWaypoint> {
```

## Architecture & Concepts

The WorldPath class is a fundamental data model that represents a named, ordered sequence of spatial points (waypoints) within the game world. It is not an active system or manager; rather, it is a passive data container designed for persistence and network transfer.

Its primary architectural role is to serve as the canonical representation for paths used by various game systems, such as AI navigation, moving platforms, camera sequences, or scripted events. The inclusion of a static **CODEC** field is a critical design feature, indicating that WorldPath instances are intended to be serialized into and deserialized from the world data files.

A key internal concept is the distinction between the raw `waypoints` (a list of Transform objects) and the derived `simpleWaypoints` (a list of SimplePathWaypoint objects). The latter is a lazily-computed, cached representation that enriches the raw data with an index, which is often required by path-following algorithms. This lazy computation is a performance optimization to avoid unnecessary processing for paths that are loaded but not immediately used.

## Lifecycle & Ownership

-   **Creation:** A WorldPath instance is created in one of two ways:
    1.  **Deserialization:** The most common path. The engine's persistence layer reads world data from disk, using the static `WorldPath.CODEC` to decode the binary data and construct a WorldPath instance. This process uses the protected no-argument constructor.
    2.  **Programmatic Instantiation:** Created at runtime via `new WorldPath(name, waypoints)`. This is typically done by in-game editing tools, procedural generation systems, or for creating temporary, dynamic paths that are not meant to be saved.

-   **Scope:** A WorldPath is a transient object. Its lifetime is bound to the component that holds a reference to it, such as a world entity, a quest manager, or an AI behavior tree. It does not have a global scope and persists only as long as it is actively referenced in memory.

-   **Destruction:** The object is eligible for garbage collection once all references to it are released. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency

-   **State:** The class is **effectively immutable** after construction. While its internal fields are not declared as final, the public API provides no methods for mutation. The `simpleWaypoints` list is populated on first access, a form of internal state change, but this does not alter the logical data of the path itself.

-   **Thread Safety:** This class is **not thread-safe**. The lazy initialization logic within the `getPathWaypoints` method presents a clear race condition. If multiple threads attempt to access the waypoints of a newly created instance simultaneously, the initialization block may be entered more than once.

    **WARNING:** Any concurrent access to a WorldPath instance, especially calls to `getPathWaypoints` or `get(index)`, must be externally synchronized. Failure to do so can lead to unpredictable behavior and redundant computation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | UUID | O(1) | Returns the unique, persistent identifier for the path. |
| getName() | String | O(1) | Returns the human-readable name assigned to the path. |
| getPathWaypoints() | List | O(N) first call, O(1) after | Returns the list of indexed waypoints. Triggers a one-time computation and caching operation on its first invocation. |
| length() | int | O(1) | Returns the total number of waypoints in the path. |
| get(int index) | SimplePathWaypoint | O(N) first call, O(1) after | Retrieves a single waypoint by its index. Defers to `getPathWaypoints`. |
| getWaypoints() | List | O(1) | Returns the raw, underlying list of Transform objects. |

## Integration Patterns

### Standard Usage

WorldPath is designed to be consumed by other systems. A common pattern is for an AI entity to retrieve a path from a world manager or its own configuration and then iterate through the waypoints to guide its movement.

```java
// Example: An AI entity follows a pre-defined path
WorldPath patrolRoute = world.getPathManager().findByName("GuardPatrol_A");

if (patrolRoute != null && patrolRoute.length() > 0) {
    for (SimplePathWaypoint waypoint : patrolRoute.getPathWaypoints()) {
        aiController.moveTo(waypoint.getTransform().getPosition());
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **State Modification via Reflection:** Do not attempt to modify the internal `waypoints` list after construction. The class assumes its initial state is final, and any external modification will lead to desynchronization with the cached `simpleWaypoints` list.
-   **Unsynchronized Concurrent Access:** As detailed in the concurrency section, never share a WorldPath instance across threads without implementing external locking around calls to `getPathWaypoints` or `get`.

## Data Pipeline

The primary data flow for WorldPath involves serialization and deserialization from world storage. It acts as a data container that is hydrated from a binary source and then consumed by game logic.

> Flow:
> World File on Disk -> Engine I/O Service -> `WorldPath.CODEC` Deserializer -> **WorldPath Instance** -> AI Navigation System

