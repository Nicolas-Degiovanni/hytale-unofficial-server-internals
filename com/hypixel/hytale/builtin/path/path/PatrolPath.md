---
description: Architectural reference for PatrolPath
---

# PatrolPath

**Package:** com.hypixel.hytale.builtin.path.path
**Type:** Transient Data Model

## Definition
```java
// Signature
public class PatrolPath implements IPrefabPath {
```

## Architecture & Concepts

The PatrolPath class is a server-side, in-memory representation of an ordered sequence of waypoints. It serves as a foundational data structure for the AI navigation system, specifically for entities that follow predefined routes within the game world.

Its core architectural purpose is to manage a path that may be only partially loaded into memory at any given time. This is critical for performance and scalability in large, persistent worlds. The system distinguishes between the total number of waypoints that constitute the path (*length*) and the number of waypoints whose corresponding entities are currently loaded and active in the world (*loadedCount*).

This design allows high-level systems to reason about a complete path (e.g., its total length and identity) while only paying the memory and processing cost for the segments of the path that are in relevant, active regions of the world.

The class employs a read-optimized caching strategy for its primary data-access method, getPathWaypoints. It maintains a canonical but unordered concurrent map of waypoints and a lazily-generated, cached, ordered list. This ensures that frequent read operations by multiple AI agents are extremely fast, while modifications to the path structure correctly invalidate the cache.

## Lifecycle & Ownership

-   **Creation:** A PatrolPath instance is created by the PathPlugin service when a new path is defined by a user or loaded from world storage. The constructor requires a world generation ID, a unique UUID, and a name, indicating that its identity is tied to persistent world data.

-   **Scope:** The object persists as long as the path is relevant to the server's current state. Its lifecycle is managed by the PathPlugin and is typically tied to the lifetime of the world zone or region in which the path resides.

-   **Destruction:** The object is eligible for garbage collection when the PathPlugin releases all references to it. This typically occurs when a world chunk or zone containing the path is unloaded from server memory.

## Internal State & Concurrency

-   **State:** The internal state of a PatrolPath is highly mutable. It is composed of a collection of waypoints that can be added, removed, or re-ordered dynamically during runtime. The key state variables include:
    -   An Int2ObjectConcurrentHashMap storing waypoints by their order index.
    -   Atomic counters for the total path length and the number of currently loaded waypoints.
    -   A cached List of waypoints for fast, ordered iteration.
    -   An AtomicBoolean dirty flag, pathChanged, to manage cache validity.

-   **Thread Safety:** This class is explicitly designed to be thread-safe and is safe for concurrent access from multiple threads, such as different AI agent threads.
    -   **Atomic Operations:** Counters (length, loadedCount) and the dirty flag (pathChanged) use atomic classes to ensure safe, lock-free updates.
    -   **Concurrent Map:** The primary waypoint storage uses Int2ObjectConcurrentHashMap, which provides thread-safe, high-performance concurrent reads and point-wise writes.
    -   **Read-Write Lock:** A ReentrantReadWriteLock protects the cached waypointList. This pattern allows for an unlimited number of concurrent readers, maximizing performance for AI agents querying the path. A write lock is acquired only when the path's structure changes, invalidating the cache and rebuilding the list. This is a critical optimization for read-heavy workloads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPathWaypoints() | List | O(N) / O(1) | Returns an unmodifiable, ordered list of waypoints. Complexity is O(N) on the first call after a modification (cache rebuild) and O(1) for all subsequent reads. |
| addLoadedWaypoint(...) | void | O(1) | Registers a waypoint that has been loaded from storage. This is the primary method for populating the path as world chunks are loaded. |
| unloadWaypoint(index) | void | O(1) | Deregisters a waypoint, marking it as unloaded. The total path length remains unchanged. |
| registerNewWaypoint(...) | short | O(N) | Appends a new waypoint to the end of the path. Complexity is driven by the need to iterate and mark existing waypoints for persistence. |
| removeWaypoint(...) | void | O(N) | Removes a waypoint and re-indexes subsequent waypoints to fill the gap. This is a costly operation. |
| isFullyLoaded() | boolean | O(1) | Checks if the number of loaded waypoints equals the total path length. |
| mergeInto(...) | void | O(N) | Moves all waypoints from this path into a target path, effectively consuming this instance. |
| compact() | void | O(N) | Re-indexes all waypoints to remove any gaps in their ordering. This is a maintenance operation. |

## Integration Patterns

### Standard Usage

A server-side system, such as an AI behavior controller, should retrieve a PatrolPath instance from the central PathPlugin and use it to guide an entity. The primary interaction is to retrieve the ordered list of waypoints.

```java
// A system retrieves the path via the managing plugin
PatrolPath path = PathPlugin.get().getPath(pathId);

// Check if the path is ready for use
if (path != null && path.isFullyLoaded()) {
    List<IPrefabPathWaypoint> waypoints = path.getPathWaypoints();
    for (IPrefabPathWaypoint waypoint : waypoints) {
        // ... direct an AI agent to the waypoint's position
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new PatrolPath()`. The lifecycle and identity of a path are managed by the PathPlugin. Creating an instance directly will result in a disconnected, untracked path that will not be persisted or recognized by other game systems.

-   **Cache Holding:** Do not retrieve the waypoint list via getPathWaypoints and hold the reference for a long duration while other threads may be modifying the path. While the returned list is unmodifiable, a long-held reference may become stale if the path is changed. It is safer to re-request the list when needed.

-   **External Modification:** Do not attempt to modify the internal waypoint map through reflection or other means. All modifications must go through the public API to ensure the pathChanged flag is set and the cache is correctly invalidated. Failure to do so will lead to a permanently stale cache.

## Data Pipeline

The PatrolPath acts as a stateful container in the flow of path data from persistence to consumption by AI.

> Flow:
> World Storage -> PathPlugin Loader -> **PatrolPath.addLoadedWaypoint** -> In-Memory State (**PatrolPath**) -> **PatrolPath.getPathWaypoints** -> AI Behavior System

