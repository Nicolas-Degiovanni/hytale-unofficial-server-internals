---
description: Architectural reference for IPrefabPath
---

# IPrefabPath

**Package:** com.hypixel.hytale.builtin.path.path
**Type:** Interface

## Definition
```java
// Signature
public interface IPrefabPath extends IPath<IPrefabPathWaypoint> {
```

## Architecture & Concepts
The IPrefabPath interface defines the contract for a specialized, dynamic path structure associated with a server-side "prefab". In Hytale's architecture, a prefab is a pre-authored collection of entities and blocks that can be instantiated within the world, such as a dungeon room, a bandit camp, or a puzzle structure.

This interface extends the generic IPath, specializing it to manage a collection of IPrefabPathWaypoint objects. Its primary role is to serve as a data structure for systems that require spatial pathing information, such as NPC patrol routes, minecart tracks, or scripted event sequences.

The key architectural distinction of IPrefabPath is its direct integration with the world's streaming and chunk-loading system. Methods like `addLoadedWaypoint` and `unloadWaypoint` reveal that a path's components are not always held in memory. The path can exist as a partially loaded entity, with its waypoints being hydrated or dehydrated as the relevant world chunks enter or leave a player's view distance. This design is critical for performance in a large, dynamic world, preventing the server from holding thousands of complete paths for inactive prefabs in memory.

## Lifecycle & Ownership
As an interface, IPrefabPath does not have a concrete lifecycle. The lifecycle described here applies to any class that implements this contract.

-   **Creation:** An IPrefabPath implementation is instantiated by the server's prefab management system when a prefab containing a path is placed into the world. It is typically created during world generation or when a game master dynamically spawns a prefab.
-   **Scope:** The object's lifetime is strictly tied to the lifetime of its parent prefab instance within the world. It persists as long as the prefab exists.
-   **Destruction:** The path object is marked for garbage collection when its parent prefab is removed from the world, either through gameplay actions (e.g., a dungeon is cleared and despawned) or world unloading.

**WARNING:** Systems should not hold long-term references to an IPrefabPath. Always re-acquire the path from the parent prefab or entity to avoid operating on a stale or destroyed object.

## Internal State & Concurrency
The IPrefabPath contract implies a highly mutable internal state. The collection of waypoints, and their loaded status, can change frequently in response to world streaming events.

-   **State:** The state is mutable, representing the current set of waypoints and their connectivity. A key piece of internal state is the distinction between the *complete* set of waypoints defined in the prefab data and the *currently loaded* set of waypoints available for immediate use.
-   **Thread Safety:** Implementations of this interface are **not guaranteed to be thread-safe**. All modifications and most read operations, especially those involving a ComponentAccessor, must be performed on the main server thread during the world tick. Accessing the path from an asynchronous task without proper synchronization will lead to severe concurrency issues, such as ConcurrentModificationExceptions or inconsistent path data.

## API Surface
The public API is designed for management by the world streaming system and consumption by path-following agents.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerNewWaypoint(waypoint, connection) | short | O(log N) | Adds a new waypoint to the path, typically during initial creation. Returns a local ID. |
| addLoadedWaypoint(waypoint, id, chunkX, chunkZ) | void | O(log N) | Hydrates a waypoint from storage when its world chunk is loaded. Critical for world streaming. |
| unloadWaypoint(id) | void | O(log N) | Dehydrates a waypoint when its world chunk is unloaded to conserve memory. |
| removeWaypoint(id, connection) | void | O(log N) | Permanently removes a waypoint from the path. |
| hasLoadedWaypoints() | boolean | O(1) | Returns true if at least one waypoint is currently loaded in memory. |
| isFullyLoaded() | boolean | O(1) | Returns true only if all waypoints comprising the path are loaded in memory. |
| getNearestWaypointPosition(pos, accessor) | Vector3d | O(N) | Scans loaded waypoints to find the one closest to a given world position. |
| mergeInto(otherPath, connection, accessor) | void | O(M+N) | A complex operation to combine this path with another, stitching them together. |
| compact(connection) | void | O(N) | An optimization routine to clean up the path data, potentially removing redundant nodes. |

## Integration Patterns

### Standard Usage
The IPrefabPath is not typically manipulated directly by gameplay logic. Instead, it is queried by higher-level systems, such as an AI behavior tree or a movement component, to guide an entity.

```java
// Example: An AI system finding the start of a patrol route
// This code would execute within a server-side system update (e.g., AI tick)

Entity prefabRoot = ... // Get the entity representing the prefab
IPrefabPath patrolPath = prefabRoot.getComponent(PrefabPathComponent.class).getPath();

if (patrolPath.hasLoadedWaypoints()) {
    Vector3d startPosition = patrolPath.getNearestWaypointPosition(self.getPosition(), entityStoreAccessor);
    // The AI can now move towards startPosition
    movementComponent.setTarget(startPosition);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Assumption:** Do not assume a path is fully loaded. Always check `hasLoadedWaypoints` or `isFullyLoaded` before attempting to traverse the entire path. Iterating over a partially loaded path can result in incomplete or incorrect behavior.
-   **External Modification:** Do not call modification methods like `registerNewWaypoint` or `removeWaypoint` from general gameplay code. These methods are intended for use by the core prefab and world generation systems. Unauthorized modification can corrupt the prefab's state.
-   **Asynchronous Access:** Never read or write to an IPrefabPath from an external thread. All interactions must be marshaled back to the main server thread to prevent data corruption.

## Data Pipeline
The IPrefabPath acts as a dynamic data container whose state is governed by world events. It does not process a continuous stream of data but rather reacts to discrete events.

> **Flow (Waypoint Loading):**
> World Streamer detects player proximity -> Chunk Load Event -> Prefab System receives event -> **IPrefabPath**.addLoadedWaypoint() -> Waypoint becomes available for AI queries.

> **Flow (AI Path Query):**
> AI Behavior Tree Tick -> AI System requests path -> Queries **IPrefabPath**.getNearestWaypointPosition() -> Receives Vector3d -> Entity Movement Component is updated.

