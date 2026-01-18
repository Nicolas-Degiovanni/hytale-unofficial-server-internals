---
description: Architectural reference for InstanceDataResource
---

# InstanceDataResource

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** Transient Data Model

## Definition
```java
// Signature
public class InstanceDataResource implements Resource<ChunkStore> {
```

## Architecture & Concepts
InstanceDataResource is a data-holding component, not an active service. It functions as a state container that attaches metadata to a game instance, specifically for managing its automatic removal and cleanup. By implementing the Resource interface with a generic type of ChunkStore, it signifies that this data is directly associated with the persistence layer of a world or dimension.

The primary consumer of this resource is the InstancesPlugin, which is responsible for the server-side lifecycle of game instances (e.g., dungeons, minigames, or temporary worlds). This resource stores critical state, such as various timeout timers and a flag indicating if a player has ever been present. The instance management system periodically inspects this data to determine if an instance is idle, expired, or otherwise eligible for unloading.

A critical feature is the static CODEC field. This enables the engine's persistence layer to automatically serialize and deserialize the resource's state to and from disk. This ensures that the removal timers and flags for an instance survive server restarts, preventing premature cleanup or orphaned instances.

## Lifecycle & Ownership
- **Creation:** An InstanceDataResource is typically instantiated by the InstancesPlugin when a new game instance is created. It is then attached to the instance's primary ChunkStore. Alternatively, it is created by the codec during deserialization when a world is loaded from disk.
- **Scope:** The lifecycle of this object is strictly bound to the lifecycle of the ChunkStore it is attached to. It exists for the entire duration of the game instance.
- **Destruction:** The object is marked for garbage collection when the parent game instance is fully unloaded and its associated ChunkStore is closed and de-referenced.

## Internal State & Concurrency
- **State:** This object is highly mutable. Its core purpose is to hold dynamic state that is frequently updated by the instance management system, such as advancing timers or flipping boolean flags. The state is non-transient and is persisted to disk.
- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal synchronization mechanisms. All reads and writes must be performed on the main server thread or within a context that guarantees exclusive access to the parent ChunkStore, such as a world's dedicated update loop. Unsynchronized access from other threads will result in data corruption and unpredictable instance removal behavior.

## API Surface
The public API consists primarily of simple accessors and mutators for its internal state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | ResourceType | O(1) | Statically retrieves the unique type identifier for this resource. |
| isRemoving() | boolean | O(1) | Returns a transient flag indicating if removal is in progress. |
| setRemoving(boolean) | void | O(1) | Sets the transient flag for the removal process. |
| getTimeoutTimer() | Instant | O(1) | Gets the primary expiration timestamp for the instance. |
| setTimeoutTimer(Instant) | void | O(1) | Sets the primary expiration timestamp. |
| getIdleTimeoutTimer() | Instant | O(1) | Gets the timestamp for when an empty instance should be removed. |
| setIdleTimeoutTimer(Instant) | void | O(1) | Sets the idle timeout timestamp. |
| hadPlayer() | boolean | O(1) | Returns true if a player has ever joined this instance. |
| setHadPlayer(boolean) | void | O(1) | Sets the flag indicating player presence. |
| clone() | InstanceDataResource | O(1) | **Warning:** Creates a new, empty resource, not a copy of the existing state. |

## Integration Patterns

### Standard Usage
This resource is not meant to be used directly by most game logic. It is managed by core engine systems. A system would retrieve it from a ChunkStore to check removal conditions.

```java
// Hypothetical usage within an instance management system
ChunkStore instanceStore = world.getChunkStore();
InstanceDataResource data = instanceStore.getResource(InstanceDataResource.getResourceType());

if (data != null && !data.hadPlayer() && Instant.now().isAfter(data.getIdleTimeoutTimer())) {
    // This instance has been idle and never used, schedule it for removal.
    instanceManager.scheduleForRemoval(world.getId());
    data.setRemoving(true);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InstanceDataResource()` to get the state of an existing instance. This creates a new, blank data object. Always retrieve the authoritative version from the ChunkStore via `chunkStore.getResource()`.
- **Manual Persistence:** Never attempt to serialize or deserialize this object manually. The engine's persistence layer handles this automatically using the provided static CODEC. Manual handling will lead to state desynchronization.
- **Unsynchronized Modification:** Do not modify this resource from an asynchronous task or a different thread without synchronizing on the parent world object. Doing so is a guaranteed race condition.

## Data Pipeline
InstanceDataResource acts as a stateful node in the instance management data flow rather than a processing stage.

> **State Update Flow:**
> Game Event (e.g., Player Leaves Instance) -> Instance Management System -> Retrieves **InstanceDataResource** from ChunkStore -> Updates Timers/Flags -> World Save Process -> Serializes **InstanceDataResource** -> Disk Storage
>
> **State Read Flow:**
> Server Tick -> Instance Management System -> Retrieves **InstanceDataResource** from ChunkStore -> Compares Timers with Current Time -> Triggers Instance Removal Logic

