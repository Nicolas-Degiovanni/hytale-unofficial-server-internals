---
description: Architectural reference for AmbientEmitterSystems
---

# AmbientEmitterSystems

**Package:** com.hypixel.hytale.builtin.ambience.systems
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class AmbientEmitterSystems {
    public static class EntityAdded extends HolderSystem<EntityStore> { ... }
    public static class EntityRefAdded extends RefSystem<EntityStore> { ... }
    public static class Ticking extends EntityTickingSystem<EntityStore> { ... }
}
```

## Architecture & Concepts
AmbientEmitterSystems is not a concrete object but a static container for a suite of three distinct, cooperative Entity-Component-System (ECS) classes. These systems collectively manage the lifecycle and behavior of ambient sound sources in the world.

The core architectural pattern employed is the **Proxy Audio Entity**. An entity with an AmbientEmitterComponent does not produce sound directly. Instead, the systems create a separate, transient, and non-serialized entity whose sole purpose is to host an AudioComponent. This decouples the logical concept of a sound emitter from the physical, networked audio source, providing significant flexibility and preventing pollution of the primary entity's archetype.

The three systems orchestrate this pattern:
1.  **EntityAdded:** A preparatory system that ensures entities with an AmbientEmitterComponent are correctly configured for networking and persistence.
2.  **EntityRefAdded:** The primary lifecycle manager. It reacts to the creation of an emitter entity by spawning the corresponding proxy audio entity. It also handles cleanup, destroying the proxy when the original is removed.
3.  **Ticking:** A per-frame maintenance system. It synchronizes the position of the proxy audio entity with its parent emitter and performs health checks to clean up orphaned emitters.

These systems operate exclusively on the server and are driven by the main game loop's ECS processing stages.

### Lifecycle & Ownership
-   **Creation:** Instances of the nested system classes are created by the server's central System Registry during the engine bootstrap phase. They are discovered via reflection and added to the appropriate processing groups.
-   **Scope:** The systems are singletons within the context of a running server. They persist for the entire duration of the server session.
-   **Destruction:** The systems are destroyed when the server shuts down and the System Registry is cleared.

## Internal State & Concurrency
-   **State:** The systems themselves are entirely stateless. All state is stored within the components of the entities they operate on, primarily the AmbientEmitterComponent and TransformComponent. The only internal fields are cached Query objects, which are immutable and thread-safe.
-   **Thread Safety:** These systems are designed to be executed by a single-threaded ECS scheduler within a given world (EntityStore). Concurrency is managed at the engine level. All structural modifications to the ECS world (adding/removing entities) are deferred via a CommandBuffer. This ensures that changes are applied at a safe synchronization point at the end of the tick, preventing race conditions and iterator invalidation.

---

## System Breakdown

### EntityAdded
This system performs initial setup on an entity as soon as it is created with an AmbientEmitterComponent. It is a HolderSystem, meaning it operates on the raw, mutable entity definition (Holder) before it is fully committed to the world.

**Responsibilities:**
-   Assigns a NetworkId if one does not already exist, making the emitter entity network-aware.
-   Ensures the presence of the Intangible component, preventing physical collisions.
-   Ensures the presence of the PrefabCopyableComponent, allowing the entity to be correctly saved and loaded as part of a prefab.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback invoked by the ECS engine when a matching entity is created. |
| getQuery() | Query | O(1) | Returns a query for entities with both AmbientEmitterComponent and TransformComponent. |

### EntityRefAdded
This is the core lifecycle management system. As a RefSystem, it operates on stable references (Ref) to entities that have been fully added to the world. Its primary role is to manage the creation and destruction of the proxy audio entity.

**Responsibilities:**
-   **Creation:** On entity addition, it creates a new, separate entity. This new entity is given a TransformComponent (cloned from the parent), an AudioComponent configured with the sound event from the parent, a NetworkId, and the Intangible and NonSerialized components. The reference to this new proxy entity is stored back into the parent's AmbientEmitterComponent.
-   **Destruction:** On entity removal, it retrieves the proxy entity's reference from the parent's AmbientEmitterComponent and issues a command to remove it from the world.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(1) | Spawns the proxy audio entity via the CommandBuffer. |
| onEntityRemove(ref, reason, store, cmd) | void | O(1) | Despawns the proxy audio entity via the CommandBuffer. |
| getQuery() | Query | O(1) | Returns a query for entities with both AmbientEmitterComponent and TransformComponent. |

### Ticking
This system runs every game tick to perform continuous maintenance. As an EntityTickingSystem, it is highly optimized for iterating over large numbers of entities.

**Responsibilities:**
-   **Position Synchronization:** It checks the distance between the parent emitter and its proxy audio entity. If the distance exceeds a small threshold (1.0), it updates the proxy's position to match the parent's. This is a crucial step to ensure the sound emanates from the correct location as the parent moves.
-   **Health Check:** It validates that the proxy entity referenced by the AmbientEmitterComponent still exists and is valid. If the proxy has been lost or destroyed through other means, this system logs a warning and removes the parent emitter entity to prevent orphaned, non-functional emitters from accumulating in the world.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(1) per entity | Executes the synchronization and health check logic for a single entity. |
| getQuery() | Query | O(1) | Returns a query for entities with both AmbientEmitterComponent and TransformComponent. |

## Integration Patterns

### Standard Usage
These systems are not intended to be used directly by game logic developers. They are automatically registered and managed by the engine. The standard pattern for a developer is to simply add an AmbientEmitterComponent to an entity prefab or at runtime. The systems will then automatically detect this and perform all necessary logic.

```java
// A developer would typically do this in another system or setup code.
// The AmbientEmitterSystems will react to this automatically.

Holder<EntityStore> myEntity = ...;

AmbientEmitterComponent emitter = new AmbientEmitterComponent();
emitter.setSoundEventId("my.ambient.sound.event");

myEntity.addComponent(AmbientEmitterComponent.getComponentType(), emitter);
myEntity.ensureComponent(TransformComponent.getComponentType());

// The entity is then added to the world.
commandBuffer.addEntity(myEntity, AddReason.SPAWN);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AmbientEmitterSystems.Ticking()`. The engine's System Registry is solely responsible for the lifecycle of these objects.
-   **Manual Proxy Management:** Do not attempt to manually create or manipulate the proxy audio entity. The `spawnedEmitter` field within AmbientEmitterComponent is internal to this suite of systems. Modifying it will lead to desynchronization and orphaned entities.
-   **Component Misconfiguration:** Adding an AmbientEmitterComponent without a TransformComponent will cause the entity to be ignored by these systems, resulting in no sound being played.

## Data Pipeline

The data flow is cyclical and reactive, driven by the ECS engine's state changes and tick loop.

> **Creation Flow:**
> Entity with (AmbientEmitterComponent, TransformComponent) Added to World -> **EntityRefAdded.onEntityAdded** -> CommandBuffer.addEntity(Proxy) -> New Proxy Entity with (AudioComponent, TransformComponent) created at end of tick.
>
> ---
>
> **Per-Tick Update Flow:**
> Game Tick Begins -> **Ticking.tick** -> Read Parent TransformComponent.position -> Compare to Proxy TransformComponent.position -> Write updated position to Proxy TransformComponent if needed.
>
> ---
>
> **Destruction Flow:**
> Original Entity Removed from World -> **EntityRefAdded.onEntityRemove** -> CommandBuffer.removeEntity(Proxy) -> Proxy Entity destroyed at end of tick.

