---
description: Architectural reference for LegacySpawnBeaconEntity
---

# LegacySpawnBeaconEntity

**Package:** com.hypixel.hytale.server.spawning.beacons
**Type:** Transient Entity Component

## Definition
```java
// Signature
public class LegacySpawnBeaconEntity extends Entity {
```

## Architecture & Concepts
The LegacySpawnBeaconEntity is a server-side, non-physical entity that functions as a configurable NPC spawn point. It is the runtime representation of a spawn location defined in a `BeaconNPCSpawn` asset. This class is not a manager or a system; rather, it is a discrete, stateful object that embodies the rules and cooldowns for a single spawn location within the world.

Architecturally, it serves as a data-driven anchor in the Entity Component System (ECS). It holds the state necessary for the overarching spawning systems to make decisions. Key responsibilities include:

*   **State Management:** Tracking spawn cooldowns (both game-time and real-time), spawn attempt counters, and player proximity data.
*   **Configuration Linking:** Maintaining a reference to its configuration via a `BeaconSpawnWrapper`, which dictates what to spawn, when, and how.
*   **Lifecycle Control:** Managing its own existence through a self-despawn timer, allowing for temporary or event-based spawn points.
*   **Visibility Logic:** Implementing logic to remain invisible to players in non-creative game modes, ensuring it is purely a server-side mechanism.

It integrates deeply with `BeaconSpawnController` for managing the population of spawned NPCs and `FloodFillPositionSelector` for determining valid spawn locations in the nearby terrain. The term *Legacy* suggests this implementation may coexist with or be superseded by newer spawning paradigms, but it remains a core component for scripted and world-based NPC generation.

### Lifecycle & Ownership
-   **Creation:** Instances are never created with a direct `new` call in game logic. They are instantiated and configured exclusively through the static factory methods `create` or `createHolder`. These methods are typically invoked by higher-level world generation or spawning plugins when populating a zone with spawn points from asset definitions. The `create` method immediately registers the entity with the world's `EntityStore`.
-   **Scope:** An instance persists as long as its parent chunk is loaded. Due to its `CODEC` definition, its state is serialized and saved with the world, allowing it to persist across server restarts. It can also have a transient lifecycle if a `despawnSelfAfter` timer is set, causing it to be removed from the world after a specified duration.
-   **Destruction:** The entity is destroyed when the server explicitly removes it, its self-despawn timer elapses, or the chunk it resides in is unloaded and pruned. The ECS framework manages the final memory reclamation.

## Internal State & Concurrency
-   **State:** The state is highly mutable. It contains timers (`nextSpawnAfter`, `despawnSelfAfter`), counters (`spawnAttempts`), and references to its configuration and controller. This state is critical for the spawning loop and is persisted to disk, making it a fundamental part of the world's saved data.
-   **Thread Safety:** This class is **not thread-safe**. All interactions with a LegacySpawnBeaconEntity instance must be performed on the main server thread for the world it belongs to. The engine's use of `ComponentAccessor` and `CommandBuffer` implies a single-threaded, sequential update model for game logic. Unsynchronized access from other threads will lead to data corruption, race conditions, and server instability.

## API Surface
The public API is designed for interaction with the server's spawning systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create(spawnWrapper, pos, rot, accessor) | Pair | O(N) | **Critical:** Static factory for creating, configuring, and registering a new beacon entity in the world. |
| notifySpawn(target, spawnedEntity, store) | void | O(M) | Informs the beacon of a successful spawn. Links the new NPC and resets spawn timers. M is flock size. |
| prepareSpawnContext(pos, count, index, context, buffer) | boolean | O(K) | Prepares the spawning context for a potential spawn. Delegates to FloodFillPositionSelector. |
| prepareNextSpawnTimer(ref, accessor) | void | O(1) | **Critical:** Static method to calculate and set the next spawn time based on asset configuration. |
| setToDespawnAfter(ref, duration, accessor) | void | O(1) | Static method to schedule the entity for automatic removal from the world. |
| isHiddenFromLivingEntity(ref, target, accessor) | boolean | O(1) | Core visibility logic. Hides the beacon from non-creative mode players. |

## Integration Patterns

### Standard Usage
The beacon is designed to be managed by an external system, not to act on its own. A typical interaction involves a global spawning system that queries for all beacon entities, checks their timers, and initiates the spawn process.

```java
// In a server system that runs every tick...
for (Ref<EntityStore> beaconRef : world.getEntitiesWith(LegacySpawnBeaconEntity.getComponentType())) {
    LegacySpawnBeaconEntity beacon = store.getComponent(beaconRef, LegacySpawnBeaconEntity.getComponentType());

    // Check if the spawn cooldown has elapsed
    if (isReadyToSpawn(beacon)) {
        // Attempt to find a player and a valid position
        SpawningContext context = new SpawningContext();
        boolean canSpawn = beacon.prepareSpawnContext(player.getPosition(), ..., context, commandBuffer);

        if (canSpawn) {
            // Create the NPC entity using the context
            Ref<EntityStore> spawnedNpc = createNpcFromContext(context);
            
            // Notify the beacon so it can reset its timer and track the NPC
            beacon.notifySpawn(player, spawnedNpc, store);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LegacySpawnBeaconEntity()`. This bypasses the essential component setup (`TransformComponent`, `ModelComponent`, `UUIDComponent`, etc.) performed by the `createHolder` factory method, resulting in a malformed and non-functional entity.
-   **Manual Timer Management:** Do not set the `nextSpawnAfter` field directly. Always use the static `prepareNextSpawnTimer` method. This ensures the correct time source (real-time vs. game-time) and duration range are used as defined in the spawn configuration asset.
-   **State Modification Without Notification:** Modifying the beacon's position via its `TransformComponent` without calling `moveTo` will cause inconsistencies. The `moveTo` method correctly triggers a rebuild of the `FloodFillPositionSelector` cache, which is essential for finding valid spawn locations.

## Data Pipeline
The LegacySpawnBeaconEntity acts as a stateful node in the server's NPC spawning pipeline. Data flows into it from configuration assets and out of it in the form of spawn commands and state updates.

> Flow:
> 1.  **Asset Loader:** Loads `BeaconNPCSpawn` configuration from game assets.
> 2.  **World Generator:** Calls `LegacySpawnBeaconEntity.create()` with the configuration to place a beacon in the world.
> 3.  **Spawning System (Tick):** Queries for ready beacons by checking `nextSpawnAfter`.
> 4.  **LegacySpawnBeaconEntity:** The system calls `prepareSpawnContext`. The beacon uses its internal `FloodFillPositionSelector` to find a valid spawn point near a player.
> 5.  **Spawning System (Entity Creation):** Creates the NPC entity based on the context provided by the beacon.
> 6.  **LegacySpawnBeaconEntity (Feedback Loop):** The system calls `notifySpawn`. The beacon attaches a `SpawnBeaconReference` to the new NPC, updates its internal tracking, and calls `prepareNextSpawnTimer` to begin the next cooldown cycle.

