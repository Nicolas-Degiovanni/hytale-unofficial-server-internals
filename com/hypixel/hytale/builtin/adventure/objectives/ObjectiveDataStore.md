---
description: Architectural reference for ObjectiveDataStore
---

# ObjectiveDataStore

**Package:** com.hypixel.hytale.builtin.adventure.objectives
**Type:** Singleton Service

## Definition
```java
// Signature
public class ObjectiveDataStore {
```

## Architecture & Concepts
The ObjectiveDataStore is the authoritative repository for all active and persisted adventure mode objectives. It functions as a high-performance service layer that decouples the in-game objective logic from the underlying persistence mechanism.

Architecturally, this class implements a **facade** and a **repository pattern** over a generic DataStore. Its primary responsibilities are:

1.  **In-Memory Caching:** It maintains a live, in-memory cache of all currently active Objective instances. This provides low-latency access for frequent operations during gameplay, such as checking task progress.
2.  **Persistence Management:** It orchestrates the serialization (saving) and deserialization (loading) of Objective state to and from disk via the injected DataStore dependency.
3.  **Query Acceleration:** It builds and maintains several secondary indexes, such as task references by type and entity tasks by player. These indexes are critical for performance, allowing other systems to efficiently query for relevant objectives without iterating through the entire collection. For example, an entity death event can quickly find all objectives that are listening for that specific type of event.

This class is central to the objective system's state management, ensuring data integrity and availability throughout a world session.

### Lifecycle & Ownership
-   **Creation:** An instance of ObjectiveDataStore is created once per world session, typically during the initialization phase of the ObjectivePlugin. It is constructed with a concrete DataStore implementation, which dictates how and where objective data is physically stored.
-   **Scope:** The instance is a long-lived singleton that persists for the entire lifecycle of a game world. It accumulates the state of all objectives for all players within that world.
-   **Destruction:** The instance is destroyed during world unload or server shutdown. It is expected that a system-level process will invoke `saveToDiskAllObjectives` before destruction to ensure all in-memory changes are flushed to disk.

## Internal State & Concurrency
-   **State:** The ObjectiveDataStore is highly stateful and mutable. Its core state consists of three concurrent maps that serve as the primary cache and secondary indexes. This state is continuously modified as players start, progress, and complete objectives.

-   **Thread Safety:** This class is designed to be thread-safe for concurrent access. All internal collections are implemented using thread-safe structures like ConcurrentHashMap. This is crucial in a server environment where player actions, entity updates, and system tasks may execute on different threads and need to query or update objective state simultaneously.

    **Warning:** While individual method calls are thread-safe, sequences of calls are not atomic. For example, checking for an objective's existence and then modifying it in a separate call is not a single atomic operation and could be subject to race conditions if not properly managed by the calling code.

## API Surface
The public API provides a contract for managing the lifecycle and state of Objective instances.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadObjective(uuid, store) | Objective | High | Loads an objective from disk or retrieves it from the cache if already active. Involves I/O. |
| unloadObjective(uuid) | void | Medium | Removes an objective from the in-memory cache and triggers its cleanup logic. |
| saveToDisk(id, objective) | void | High | Persists a single objective's state to disk if it has been modified. Involves I/O. |
| saveToDiskAllObjectives() | void | Very High | Iterates and persists all active, modified objectives. A costly operation for world-save events. |
| getObjective(uuid) | Objective | O(1) | Retrieves an active objective from the in-memory cache. Returns null if not loaded. |
| getTaskRefsForType(class) | Set | O(1) | Returns a set of references to all active tasks of a specific type for efficient event processing. |
| addObjective(uuid, objective) | boolean | O(1) | Adds a new objective to the in-memory cache. |
| removeObjective(uuid) | void | O(1) | Removes an objective from the in-memory cache. Does not remove from disk. |

## Integration Patterns

### Standard Usage
The ObjectiveDataStore should be retrieved from a central service registry or plugin context. It is used to load objectives for a player and to query for tasks during event handling.

```java
// Retrieving the store and loading a player's objective
ObjectiveDataStore objectiveStore = ObjectivePlugin.get().getObjectiveDataStore();
Objective playerObjective = objectiveStore.loadObjective(playerObjectiveUUID, entityStoreProvider);

if (playerObjective != null) {
    // ... process objective logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ObjectiveDataStore()`. The class depends on a correctly configured DataStore provided via dependency injection at startup. Direct instantiation will result in a non-functional service that is disconnected from the persistence layer.
-   **Holding Stale References:** An Objective object retrieved from the store is a live object. However, if `unloadObjective` is called, any external systems holding a reference to that object will have a stale, non-functional reference. Always re-fetch from the store if there is doubt.
-   **Forgetting to Save:** Modifying an Objective's state in memory does not guarantee persistence. The `saveToDisk` method must be explicitly called, typically by a higher-level manager, to write changes to disk. The `consumeDirty` flag pattern is used to prevent redundant writes.

## Data Pipeline
The ObjectiveDataStore sits between high-level game logic and the low-level storage system. It primarily processes data in response to load requests or save triggers.

**Save Pipeline:**
> Flow:
> Game Event (e.g., Player kills mob) -> Task Processor updates Objective state -> Objective marked as dirty -> World Save Trigger -> **ObjectiveDataStore**.saveToDisk() -> DataStore serialization -> Disk Storage

**Load Pipeline:**
> Flow:
> Player Join Event -> Request to load Objective -> **ObjectiveDataStore**.loadObjective() -> Cache Check (Miss) -> DataStore deserialization -> Object Hydration & Validation -> **ObjectiveDataStore** populates cache -> Objective returned to caller

