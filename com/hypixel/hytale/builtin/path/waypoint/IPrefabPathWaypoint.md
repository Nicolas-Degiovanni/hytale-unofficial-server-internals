---
description: Architectural reference for IPrefabPathWaypoint
---

# IPrefabPathWaypoint

**Package:** com.hypixel.hytale.builtin.path.waypoint
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IPrefabPathWaypoint extends IPathWaypoint {
```

## Architecture & Concepts

The IPrefabPathWaypoint interface defines a specialized contract for waypoints that are directly associated with a server-managed entity, typically one spawned from a prefab. It extends the generic IPathWaypoint contract, adding functionality necessary to bind an abstract point in a path to a concrete, stateful object in the game world.

This interface serves as a critical bridge between the server's abstract pathfinding system and its concrete entity management system (EntityStore). While a standard waypoint might only represent a coordinate, an IPrefabPathWaypoint represents a location *defined by an entity*. This is essential for creating complex AI behaviors, such as patrol routes where an NPC must interact with specific objects, guard designated posts, or follow a path of spawned markers.

Implementations of this interface are expected to hold direct or indirect references to an entity's unique identifier and its source prefab, allowing the pathing system and AI agents to query for entity-specific properties.

### Lifecycle & Ownership
As an interface, IPrefabPathWaypoint does not have its own lifecycle. The following describes the lifecycle of a typical object that **implements** this contract.

-   **Creation:** Instances are created by an IPath object, which acts as a factory and owner. This typically occurs when a path is loaded from world data or constructed dynamically by a game system. Direct instantiation by client code is strongly discouraged.
-   **Scope:** The lifetime of an IPrefabPathWaypoint object is strictly bound to its parent IPath. It persists as long as it remains a node within that path's sequence.
-   **Destruction:** An instance is eligible for garbage collection when it is removed from its parent IPath or when the path itself is unloaded. The onReplaced method is a critical lifecycle hook called immediately before the waypoint is removed or replaced, providing a final opportunity for implementations to release resources or deregister from other systems.

## Internal State & Concurrency

-   **State:** Implementations are inherently stateful. They must maintain internal state linking them to a specific entity UUID, prefab name, and various pathing parameters (e.g., travel speed, delay) provided during initialization. This state is established via the initialise method and is considered mutable only during that phase.
-   **Thread Safety:** **Not thread-safe.** All interactions with implementations of this interface must be performed on the main server tick thread. The pathing and entity systems are not designed for concurrent access. Modifying or accessing a waypoint from an asynchronous task without proper synchronization will lead to severe state corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onReplaced() | void | O(1) | A lifecycle callback invoked by the parent path just before this waypoint is removed or replaced. Used for resource cleanup. |
| initialise(...) | void | O(1) | Configures the waypoint's state, binding it to a world entity and setting its behavioral parameters. Designed to be called once after instantiation. |
| getParentPath() | IPath<IPrefabPathWaypoint> | O(1) | Returns a reference to the IPath instance that owns and manages this waypoint. |

## Integration Patterns

### Standard Usage
Developers should not interact with this interface directly, but rather with objects that implement it. The common pattern is to retrieve a path from a world service or entity, iterate its waypoints, and access their data.

```java
// Assume 'entity' has a path component
IPath<IPrefabPathWaypoint> patrolPath = entity.getPathComponent().getPath();

for (IPrefabPathWaypoint waypoint : patrolPath.getWaypoints()) {
    // Access waypoint-specific data to influence AI behavior
    // For example, get the entity associated with this waypoint
    UUID targetEntityId = waypoint.getEntityId(); // Assumes method exists on implementation
    System.out.println("AI moving towards entity: " + targetEntityId);
}
```

### Anti-Patterns (Do NOT do this)
-   **Custom Implementation:** Avoid creating new implementations of IPrefabPathWaypoint. The engine provides canonical implementations that are tightly integrated with the entity and world persistence systems. A custom class may fail to serialize correctly or handle lifecycle events properly.
-   **Re-initialization:** Calling initialise on an already configured waypoint is undefined behavior. This will corrupt the internal state of the waypoint and its parent path, leading to unpredictable pathfinding results.
-   **Ignoring Ownership:** Do not maintain long-lived external references to a waypoint object. Its lifecycle is managed by its parent IPath. If the path is modified, your reference may become stale, pointing to a waypoint that is no longer part of the valid path.

## Data Pipeline
The IPrefabPathWaypoint is a key component in the transformation of static world data into dynamic AI behavior.

> Flow:
> World Data (e.g., Zone File) -> Server Path Loader -> IPath Factory -> **IPrefabPathWaypoint Instance** -> AI Behavior System -> Entity Movement Component

