---
description: Architectural reference for LocalSpawnState
---

# LocalSpawnState

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** State Object / Resource

## Definition
```java
// Signature
public class LocalSpawnState implements Resource<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnState class is not a service or a system; it is a pure data structure that encapsulates the spawning state for a specific, localized region of the world. In the Hytale engine, game state is often attached to primary objects using a Resource system. This class implements the Resource interface, indicating that it is designed to be attached to an EntityStore, which typically represents a world chunk or a similar regional container.

Its primary architectural role is to **decouple spawning state from spawning logic**. Systems that identify potential spawns or spawn controllers (like a monster spawner block) populate the lists within this object. A separate processing system then reads this state to execute the actual entity spawning logic.

This component is a critical piece of the SpawningPlugin, holding the transient data necessary for the local spawning loop to function within a given server tick.

### Lifecycle & Ownership
-   **Creation:** An instance of LocalSpawnState is not created directly. It is instantiated by the engine's Resource management system when a parent EntityStore is loaded or initialized. The static getResourceType method provides the key for the engine to look up and attach the correct resource type.
-   **Scope:** The lifecycle of a LocalSpawnState instance is strictly bound to its parent EntityStore. It persists in memory as long as the corresponding world chunk is active and loaded on the server.
-   **Destruction:** The object is marked for garbage collection when its parent EntityStore is unloaded from memory, for example, when no players are nearby. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** This object is highly **mutable**. Its entire purpose is to be modified by various parts of the spawning system during a game tick. It contains lists of pending spawns and active controllers, which are expected to change frequently.

-   **Thread Safety:** This class is **not thread-safe**. It uses standard, non-synchronized collections like ObjectArrayList and a primitive boolean flag. All interactions with an instance of LocalSpawnState must be externally synchronized. It is designed to be accessed exclusively from the main server thread or a dedicated thread responsible for the world region it belongs to.

    **WARNING:** Unsynchronized access from multiple threads will lead to race conditions, particularly with the pollForceTriggerControllers method, and will likely cause ConcurrentModificationExceptions when iterating its lists.

## API Surface
The public API is minimal, focusing on state access and manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLocalControllerList() | List<Ref<EntityStore>> | O(1) | Returns a direct, mutable reference to the list of spawn controllers in the local area. |
| getLocalPendingSpawns() | List<LegacySpawnBeaconEntity> | O(1) | Returns a direct, mutable reference to the list of entities queued for spawning. |
| pollForceTriggerControllers() | boolean | O(1) | Atomically reads and resets the trigger flag. This is a non-idempotent read used for one-shot events. |
| forceTriggerControllers() | void | O(1) | Sets a flag to force the spawning system to re-evaluate controllers on the next tick. |
| clone() | Resource<EntityStore> | O(N) | Creates a new LocalSpawnState with a shallow copy of the internal lists. The lists themselves are new, but the elements within them are not cloned. |

## Integration Patterns

### Standard Usage
The intended pattern is for a system to retrieve this resource from a broader context, like an EntityStore, and then operate on its state.

```java
// A system processing a specific world chunk (EntityStore)
EntityStore activeChunk = ...;

// Retrieve the attached spawning state for this chunk
LocalSpawnState spawnState = activeChunk.getResource(LocalSpawnState.getResourceType());

if (spawnState != null) {
    // Check if a manual trigger was requested
    if (spawnState.pollForceTriggerControllers()) {
        // ... execute forced spawning logic ...
    }

    // Process pending spawns
    for (LegacySpawnBeaconEntity beacon : spawnState.getLocalPendingSpawns()) {
        // ... create the entity in the world ...
    }
    spawnState.getLocalPendingSpawns().clear();
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new LocalSpawnState()`. This creates an orphaned object that is not managed by the engine's resource system and will not be visible to the spawning processors. Always retrieve it from its parent EntityStore.
-   **Retaining References:** Do not hold a reference to a LocalSpawnState object beyond the scope of a single game tick. Its parent EntityStore could be unloaded at any time, making the reference stale and leading to memory leaks or unpredictable behavior.
-   **Misunderstanding Clone:** Using the clone method with the expectation of a deep copy is incorrect. Modifying an object referenced within the cloned lists will also modify the object in the original lists. The clone is primarily for creating a separate snapshot of the list contents at a specific moment.

## Data Pipeline
LocalSpawnState acts as a temporary buffer or mailbox for the spawning system. Data flows into it from discovery systems and is drained from it by a processing system.

> **Flow:**
> 1.  **World Scanner / Discovery System:** Identifies a spawner block -> Adds a Ref to **LocalSpawnState.localControllerList**.
> 2.  **Game Logic / Trigger System:** An event occurs (e.g., player proximity) -> Adds a LegacySpawnBeaconEntity to **LocalSpawnState.localPendingSpawns**.
> 3.  **LocalSpawningSystem (Processor):** On its tick, retrieves **LocalSpawnState** -> Reads lists -> Spawns entities in the world -> Clears the lists.

