---
description: Architectural reference for SpawnMarkerSystems.Ticking
---

# SpawnMarkerSystems.Ticking

**Package:** com.hypixel.hytale.server.spawning.spawnmarkers
**Type:** Transient System

## Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The Ticking system is the primary runtime engine for all Spawn Marker entities on the server. As an EntityTickingSystem, it is invoked by the server's main scheduler once per game tick for every entity that possesses both a SpawnMarkerEntity and a TransformComponent.

Its core responsibility is to manage the lifecycle of NPCs spawned by markers, implementing complex logic for activation, deactivation, and respawning. This system acts as the state machine driver for a spawn marker, transitioning it between states like *dormant*, *spawning*, *active*, and *deactivating*.

A key architectural feature is its handling of entity hibernation. To conserve server resources, this system actively despawns and stores NPC data when no players are within a configurable deactivation radius. When a player re-enters the area, it restores the NPCs from this stored state, providing the illusion of persistence without the constant overhead. This makes it a critical component for server performance in a world with many spawn points.

## Lifecycle & Ownership
- **Creation:** Instantiated by the core SystemRegistry during the server's bootstrap sequence when the EntityModule is initialized. It is not created or managed by user code.
- **Scope:** A single instance of this system persists for the entire server session. The system itself is stateless; all state is maintained within the components of the entities it processes.
- **Destruction:** The instance is discarded during server shutdown when the SystemRegistry is torn down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It reads and modifies component data (e.g., SpawnMarkerEntity, TransformComponent) but holds no internal state between ticks or across entities. All data required for its logic, such as timers and NPC references, is stored on the SpawnMarkerEntity component itself.
- **Thread Safety:** This system is designed for parallel execution. The `isParallel` method allows the ECS scheduler to process different batches of entities (ArchetypeChunks) on separate worker threads. Thread safety is achieved through two primary mechanisms:
    1. The ECS framework guarantees that different threads operate on distinct ArchetypeChunks, preventing data races on component data.
    2. All structural mutations to the world (spawning/despawning entities, adding/removing components) are deferred via a CommandBuffer. These commands are queued during the parallel phase and executed sequentially at a safe synchronization point later in the tick.

## API Surface
The public contract is with the ECS scheduler, not end-users.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N) | The main entry point called by the scheduler. Executes the spawn marker logic for a single entity. Complexity is O(N) where N is the number of players found in the proximity query. |

## Integration Patterns

### Standard Usage
This system is automatically registered and executed by the engine. A developer's interaction is indirect: creating an entity with a SpawnMarkerEntity component will cause this system to begin processing it on the next tick.

```java
// A developer does not call this system directly.
// Instead, they create an entity that this system will find.

// Example of creating a spawn marker entity
commandBuffer.createEntity(
    new TransformComponent(position),
    new SpawnMarkerEntity("some_marker_asset_id")
);

// The Ticking system will automatically pick up and manage this entity.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this system using `new`. The ECS framework manages its lifecycle.
- **Direct Invocation:** Do not call the `tick` method directly. This bypasses the scheduler, CommandBuffer synchronization, and dependency ordering, which will lead to world corruption, crashes, and concurrency violations.
- **Stateful Logic:** Do not modify the system to hold state in its own fields. This will break parallel execution and cause unpredictable behavior, as a single instance processes many entities across multiple threads.

## Data Pipeline
The system orchestrates a complex data flow involving spatial queries, component state changes, and deferred commands.

> **Activation/Deactivation Flow:**
> Game Tick -> **Ticking.tick()** -> Read TransformComponent -> Query SpatialResource for nearby players -> (Branch)
>
> **Branch A: No Players In Range**
> Start deactivation timer on SpawnMarkerEntity -> If timer expires -> Play despawn animation on NPCs -> **CommandBuffer.run()** -> Serialize NPC data into StoredFlock component -> Destroy NPC entities.
>
> **Branch B: Players In Range**
> Check for stored NPCs in StoredFlock -> If present -> **CommandBuffer.run()** -> Restore NPCs from StoredFlock -> Re-create NPC entities -> Clear StoredFlock.
>
> **Spawning Flow:**
> Game Tick -> **Ticking.tick()** -> Check if spawn conditions are met (timers, triggers) -> If true -> **CommandBuffer.run()** -> Call `SpawnMarkerEntity.spawnNPC` -> Create new NPC entities and link them to the marker.

---
---
description: Architectural reference for SpawnMarkerSystems.CacheMarker
---

# SpawnMarkerSystems.CacheMarker

**Package:** com.hypixel.hytale.server.spawning.spawnmarkers
**Type:** Transient System

## Definition
```java
// Signature
public static class CacheMarker extends RefSystem<EntityStore> {
```

## Architecture & Concepts
CacheMarker is an initialization and validation system that operates on newly created Spawn Marker entities. As a RefSystem, it hooks into the `onEntityAdded` event for any entity possessing a SpawnMarkerEntity component.

Its primary function is to act as a bridge between asset identifiers and live asset data. A SpawnMarkerEntity is typically created with only a string ID for its corresponding SpawnMarker asset. This system performs the crucial one-time task of looking up the full SpawnMarker asset from the AssetMap and caching a direct reference to it on the SpawnMarkerEntity component.

This pattern is a critical performance optimization. By caching the asset reference upon creation, the main Ticking system avoids expensive string-based hash map lookups every single frame for every active spawn marker. It also serves as a data integrity check: if the SpawnMarker asset ID is invalid and the lookup fails, this system immediately destroys the entity, preventing invalid entities from propagating through the server.

## Lifecycle & Ownership
- **Creation:** Instantiated by the SystemRegistry during server bootstrap.
- **Scope:** A single instance persists for the entire server session. It is stateless.
- **Destruction:** Discarded during server shutdown.

## Internal State & Concurrency
- **State:** This system is **stateless**.
- **Thread Safety:** The `onEntityAdded` callback can be invoked from any thread that creates entities. All structural changes, such as destroying the entity on validation failure, are safely queued in the provided CommandBuffer, ensuring thread safety.

## API Surface
The public contract is with the ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(1) | Callback executed when a matching entity is created. Performs an asset lookup and either caches the result or destroys the entity. |

## Integration Patterns

### Standard Usage
This system's execution is implicit and managed by dependencies. Other systems that need to access the full SpawnMarker asset data declare a dependency to ensure they run *after* CacheMarker has completed.

```java
// In another system, e.g., SpawnMarkerSystems.EntityAdded
public Set<Dependency<EntityStore>> getDependencies() {
    // This dependency guarantees that CacheMarker runs first.
    // By the time this system's onEntityAdded runs, the
    // cached marker is guaranteed to be populated or the entity is gone.
    return Set.of(new SystemDependency<>(Order.AFTER, SpawnMarkerSystems.CacheMarker.class));
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Dependencies:** Attempting to access the `getCachedMarker` field from another `onEntityAdded` system without declaring an explicit `Order.AFTER` dependency on CacheMarker. This creates a race condition where the field may be null.
- **Re-implementing Caching:** Do not perform the asset lookup manually in other systems. Rely on this system to be the single source of truth for populating the asset cache.

## Data Pipeline
The data flow is a simple, linear initialization step.

> Flow:
> Entity Creation with SpawnMarkerEntity -> ECS dispatches `onEntityAdded` event -> **CacheMarker.onEntityAdded()** -> Read `spawnMarkerId` from component -> Lookup in `SpawnMarker.getAssetMap()` -> (Branch)
>
> **Branch A: Asset Found**
> Call `entity.setCachedMarker(asset)` -> System completes.
>
> **Branch B: Asset Not Found**
> Log a severe error -> **commandBuffer.removeEntity()** -> Entity is queued for destruction.

