---
description: Architectural reference for SpawnMarkerEntity
---

# SpawnMarkerEntity

**Package:** com.hypixel.hytale.server.spawning.spawnmarkers
**Type:** Component

## Definition
```java
// Signature
public class SpawnMarkerEntity implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnMarkerEntity is a data-driven component that represents the state and behavior of a single, persistent spawn point in the world. It is a fundamental part of the server's Entity-Component-System (ECS) architecture, specifically within the spawning subsystem.

This component does not actively perform logic on its own. Instead, it acts as a state machine that is managed and ticked by a higher-level spawning system (part of the SpawningPlugin). It encapsulates all data necessary for a spawn point to function, including what to spawn, when to respawn, references to its spawned entities, and logic for persisting these entities across chunk loads.

Its primary responsibilities include:
- Maintaining the cooldown timer for respawns.
- Tracking the number of currently active NPCs it has spawned.
- Storing references to its spawned NPCs, enabling re-association on world load.
- Handling the complex logic of entity deactivation and reactivation, where spawned NPCs are unloaded when no players are nearby and restored from a persisted state (StoredFlock) when players return.
- Implementing a failure-handling mechanism to automatically remove itself after repeated, unsuccessful spawn attempts.

The static CODEC field is central to its design, indicating that this component is fully serializable and intended to be persisted in the world database as part of its parent entity's data.

## Lifecycle & Ownership
- **Creation:** A SpawnMarkerEntity is typically created during world generation when a prefab containing a spawn marker is placed. It is also instantiated by the serialization system when a chunk is loaded from the database and an entity within it has this component. It is **never** intended to be instantiated directly via its constructor in game logic.
- **Scope:** The component's lifetime is strictly tied to the entity it is attached to. It persists for the entire duration that its host entity exists in the world, whether that entity is active or unloaded.
- **Destruction:** The component is destroyed under two conditions:
    1.  When its host entity is removed from the EntityStore for any reason (e.g., destroyed by a player, script, or game event).
    2.  Through self-destruction via the `fail` method, which removes the host entity after a series of `MAX_FAILED_SPAWNS` consecutive failures, preventing corrupted or badly placed markers from polluting the game world indefinitely.

## Internal State & Concurrency
- **State:** The SpawnMarkerEntity is highly mutable. Its internal state, including `respawnCounter`, `spawnCount`, `npcReferences`, and `storedFlock`, is constantly updated by the spawning system during the game loop. It caches the `SpawnMarker` asset definition for performance but primarily serves as a live state container. The `storedFlock` field is a critical piece of state used for persisting NPC data when the marker is deactivated.
- **Thread Safety:** This component is **not thread-safe** and must only be accessed from the main server thread responsible for its world zone. Its design relies on the single-threaded nature of the game loop. The use of `ThreadLocalRandom` and thread-local spatial query lists are strong indicators of this design assumption. Unsynchronized access from other threads will lead to state corruption, race conditions, and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spawnNPC(ref, marker, store) | boolean | O(N) | The primary logical entry point. Attempts to spawn an NPC based on its configuration. Complexity depends on the spatial query for player exclusion. |
| tickRespawnTimer(dt) | boolean | O(1) | Decrements the respawn timer. Returns true if the timer has expired. Designed to be called every tick by the managing system. |
| tickTimeToDeactivation(dt) | boolean | O(1) | Decrements the deactivation timer. Returns true if the timer has expired, signaling that spawned entities should be unloaded. |
| trigger(markerRef, store) | boolean | O(N) | Attempts to force a spawn if the marker is configured for manual triggering and is not on cooldown. |
| suppress(suppressor) | void | O(1) | Adds a UUID to a suppression set, preventing this marker from spawning until the suppression is released. |
| releaseSuppression(suppressor) | void | O(1) | Removes a UUID from the suppression set. |
| setSpawnMarker(marker) | void | O(1) | Initializes the component with its asset definition. Critically, this also initializes the persistence strategy (e.g., creating a StoredFlock). |

## Integration Patterns

### Standard Usage
The SpawnMarkerEntity is managed by a global or world-specific spawning system. The system iterates over all entities with this component each tick, checks their state, and drives their lifecycle.

```java
// Executed within a server system that iterates entities
Store<EntityStore> store = world.getStore();
Ref<EntityStore> entityRef = ... // Reference to the entity with the marker

SpawnMarkerEntity markerComponent = store.getComponent(entityRef, SpawnMarkerEntity.getComponentType());
if (markerComponent == null) {
    return;
}

// Check if spawn conditions are met (e.g., spawn count is zero)
if (markerComponent.getSpawnCount() <= 0) {
    // Tick the respawn timer and attempt to spawn if ready
    if (markerComponent.tickRespawnTimer(deltaTime)) {
        SpawnMarker markerAsset = markerComponent.getCachedMarker();
        markerComponent.spawnNPC(entityRef, markerAsset, store);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SpawnMarkerEntity()`. The component must be added to an entity via the `Store` so it is correctly managed by the ECS framework and serialization systems.
- **Manual Timer Manipulation:** Do not call `setRespawnCounter` directly to alter spawn times from arbitrary game logic. This bypasses the intended configuration from the `SpawnMarker` asset and can lead to unpredictable behavior. Use the `trigger` method for manual activation.
- **Cross-Thread Access:** Never read or write to a SpawnMarkerEntity from an asynchronous task or a different thread without explicit, engine-provided synchronization mechanisms. This will corrupt its state.
- **State Assumption:** Do not assume `getNpcReferences` contains valid, loaded entities. The references can point to unloaded entities. Always check if a reference is valid before dereferencing it.

## Data Pipeline

> **Spawning Flow:**
> Spawning System Tick -> **SpawnMarkerEntity** (Check `spawnCount` <= 0) -> **SpawnMarkerEntity** (Check `tickRespawnTimer`) -> `spawnNPC` -> Spatial Query (Check for nearby players) -> `SpawningContext` (Check for physical space) -> `NPCPlugin.spawnEntity` -> New NPC entity created in `EntityStore` -> `npcReferences` updated.

> **Deactivation & Persistence Flow:**
> Player moves far away -> Spawning System calls `tickTimeToDeactivation` -> Timer expires -> System iterates `npcReferences` -> NPC entity data is serialized into `storedFlock` -> NPC entities are removed from the world -> **SpawnMarkerEntity** persists with `storedFlock` data.

> **Reactivation Flow:**
> Player moves back into range -> Spawning System detects proximity -> System reads `storedFlock` -> `NPCPlugin` is used to re-spawn NPCs from the stored data -> New NPC entities are created and linked via `npcReferences`.

