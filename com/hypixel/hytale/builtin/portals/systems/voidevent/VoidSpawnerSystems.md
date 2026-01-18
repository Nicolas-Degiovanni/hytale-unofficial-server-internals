---
description: Architectural reference for VoidSpawnerSystems
---

# VoidSpawnerSystems

**Package:** com.hypixel.hytale.builtin.portals.systems.voidevent
**Type:** ECS System Container

## Definition
```java
// Signature
public final class VoidSpawnerSystems {
    // ... static query definition ...

    public static class Instantiate extends RefSystem<EntityStore> {
        // ... system implementation ...
    }
}
```

## Architecture & Concepts

The VoidSpawnerSystems class is a container for a critical Entity Component System (ECS) that manages the lifecycle of dynamic mob spawners associated with a "Void Event". The primary logic resides within the nested **Instantiate** system.

This system operates reactively, observing the creation and destruction of entities that possess both a VoidSpawner and a TransformComponent. Its core responsibility is to translate a single, high-level *spawner entity* into a cluster of functional *spawn beacon entities* based on external gameplay configuration.

Upon the creation of a VoidSpawner entity, the **Instantiate** system:
1.  Retrieves the world-specific PortalGameplayConfig.
2.  Reads a list of spawn beacon asset names from the configuration.
3.  For each configured beacon, it creates a new LegacySpawnBeaconEntity at a calculated offset from the parent VoidSpawner entity's position.
4.  Crucially, it records the UUIDs of these newly created beacons into the parent's VoidSpawner component. This establishes an explicit ownership link.
5.  All entity modifications are deferred through a **CommandBuffer**, ensuring transactional integrity within the game tick.

Conversely, upon the destruction of the VoidSpawner entity, the system performs a cascading delete. It reads the stored beacon UUIDs from the component and issues commands to remove each associated beacon entity, ensuring no orphaned entities remain.

## Lifecycle & Ownership

-   **Creation:** The VoidSpawnerSystems.Instantiate system is instantiated and managed by the server's core ECS engine during world initialization. It is not created directly by game logic. Its operational methods are invoked by the engine as callbacks.
-   **Scope:** The system instance persists for the entire lifetime of the server world. Its logic is active in every game tick, processing entities that match its query.
-   **Destruction:** The system is destroyed when the server world is unloaded. The critical cleanup logic within `onEntityRemove` is tied to the lifecycle of individual *entities*, not the system itself.

**WARNING:** The integrity of the void event relies on the proper cleanup performed by this system. If the parent VoidSpawner entity is removed without this system's `onEntityRemove` callback executing, the associated spawn beacon entities will be orphaned, potentially causing persistent mob spawning.

## Internal State & Concurrency

-   **State:** The system is entirely **stateless**. It holds no instance fields and does not cache data between invocations. All necessary information is read directly from entity components or external configuration sources like PortalGameplayConfig during execution. State is owned by the components, not the system.
-   **Thread Safety:** This system is **not thread-safe** and must only be executed by the main server game loop. It relies on the single-threaded execution model of the ECS engine. The use of a CommandBuffer is a key pattern here; it queues all world modifications (entity creation/destruction) to be executed atomically at a safe point in the tick, preventing concurrent modification exceptions and race conditions.

## API Surface

The public contract is defined by the RefSystem interface. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(N) | Callback invoked when a matching entity is created. Creates N spawn beacons based on config. |
| onEntityRemove(ref, reason, store, cmd) | void | O(N) | Callback invoked when a matching entity is removed. Deletes N owned spawn beacons. |
| getQuery() | Query | O(1) | Returns the static query for entities with VoidSpawner and TransformComponent. |

*N = Number of spawn beacons defined in the InvasionPortalConfig.*

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this system directly. To trigger its logic, one must create an entity and attach the required components. The system will automatically discover and process it.

```java
// Example: Creating a Void Spawner entity that the system will act upon.
// This code would exist in some other game logic system.

// 1. Create a new entity holder.
Holder<EntityStore> spawnerEntity = world.getEntityStore().createHolder();

// 2. Add the necessary components.
TransformComponent transform = new TransformComponent();
transform.setPosition(new Vector3d(100, 64, 100));

VoidSpawner spawnerComponent = new VoidSpawner();

// 3. Add components to the entity via a CommandBuffer.
commandBuffer.addComponent(spawnerEntity.getRef(), transform);
commandBuffer.addComponent(spawnerEntity.getRef(), spawnerComponent);

// 4. Add the entity to the world.
// The VoidSpawnerSystems.Instantiate system will now automatically run its
// onEntityAdded logic for this entity in the same or a subsequent tick.
commandBuffer.addEntity(spawnerEntity, AddReason.SPAWN);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new VoidSpawnerSystems.Instantiate()`. The ECS engine is responsible for the lifecycle of all systems.
-   **Manual Invocation:** Never call `onEntityAdded` or `onEntityRemove` directly. Doing so bypasses the engine's state management and CommandBuffer, which will lead to world corruption or server crashes.
-   **State Tampering:** Do not manually clear or modify the list of `spawnBeaconUuids` within a VoidSpawner component after it has been populated by this system. This will break the cascading delete logic and lead to orphaned beacon entities.

## Data Pipeline

The system acts as a processor in a reactive data flow. It does not initiate actions but responds to state changes in the world.

> **Creation Flow:**
> Game Logic creates Entity -> ECS Engine adds **VoidSpawner** + **TransformComponent** -> System's Query matches -> `onEntityAdded` is invoked -> System reads **PortalGameplayConfig** -> System writes `addEntity` commands to **CommandBuffer** -> New **LegacySpawnBeaconEntity** instances are created.

> **Destruction Flow:**
> Game Logic removes Entity -> ECS Engine detects removal -> `onEntityRemove` is invoked -> System reads UUIDs from **VoidSpawner** component -> System writes `removeEntity` commands to **CommandBuffer** -> Owned **LegacySpawnBeaconEntity** instances are destroyed.

