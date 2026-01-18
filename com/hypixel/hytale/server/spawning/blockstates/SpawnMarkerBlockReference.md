---
description: Architectural reference for SpawnMarkerBlockReference
---

# SpawnMarkerBlockReference

**Package:** com.hypixel.hytale.server.spawning.blockstates
**Type:** Transient

## Definition
```java
// Signature
public class SpawnMarkerBlockReference implements Component<EntityStore> {
```

## Architecture & Concepts
The SpawnMarkerBlockReference is a data component within the server-side Entity Component System (ECS). Its sole purpose is to create a persistent link between a spawned entity and the specific world block that served as its origin point, often referred to as a "spawn marker".

This component acts as a piece of state attached to an entity, not as a system that performs logic. It enables other systems, particularly the SpawningPlugin, to manage the lifecycle of spawned entities in relation to their source. For instance, if a spawner block is destroyed, a management system can query for all entities with a SpawnMarkerBlockReference pointing to that block's position and trigger a despawn.

The component includes a timeout mechanism, `originLostTimeout`, which provides resilience against temporary state inconsistencies, such as when the chunk containing the origin block is unloaded. If the origin block cannot be verified for a continuous period (30 seconds), the entity is considered orphaned and can be safely removed by a cleanup system.

### Lifecycle & Ownership
- **Creation:** An instance is created by a spawning system at the moment an entity is spawned from a specific block. The component is then immediately attached to the new entity.
- **Scope:** The component's lifetime is strictly bound to the lifetime of the entity it is attached to. It persists as long as the host entity exists in the world's EntityStore.
- **Destruction:** The component is destroyed and garbage collected when its host entity is removed from the EntityStore. This can be triggered by the entity's death, a manual despawn, or by the `tickOriginLostTimeout` method returning true, which signals to a managing system that the entity's origin is permanently lost.

## Internal State & Concurrency
- **State:** The component holds mutable state. The `blockPosition` is set upon creation and is conceptually immutable, but the `originLostTimeout` field is decremented on each applicable server tick, making the object's state volatile during gameplay.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be exclusively accessed and modified by systems operating on the main server game loop thread. Unsynchronized access from other threads will lead to race conditions and unpredictable behavior, especially with the `originLostTimeout` value. All interactions must be marshaled to the main thread.

## API Surface
The public API provides the necessary hooks for entity management systems to track an entity's link to its origin.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SpawnMarkerBlockReference(Vector3i) | constructor | O(1) | Creates a new component, linking it to a specific block position. |
| getBlockPosition() | Vector3i | O(1) | Returns the world-space coordinate of the origin block. |
| refreshOriginLostTimeout() | void | O(1) | Resets the 30-second countdown. This is called when the origin block is confirmed to exist. |
| tickOriginLostTimeout(float dt) | boolean | O(1) | Decrements the countdown by the delta time. Returns true if the timer has expired. |

## Integration Patterns

### Standard Usage
A managing system queries for entities with this component and updates their timeout status based on the validity of the origin block.

```java
// In a server-side entity management system
for (Entity entity : world.getEntitiesWith(SpawnMarkerBlockReference.class)) {
    SpawnMarkerBlockReference ref = entity.getComponent(SpawnMarkerBlockReference.class);
    Vector3i originPos = ref.getBlockPosition();

    // Assumes world.isOriginBlockValid() checks if the chunk is loaded
    // and the block at the position is still a valid spawner.
    if (world.isOriginBlockValid(originPos)) {
        ref.refreshOriginLostTimeout();
    } else {
        boolean isOrphaned = ref.tickOriginLostTimeout(deltaTime);
        if (isOrphaned) {
            world.removeEntity(entity); // Despawn the orphaned entity
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual State Management:** Do not manually set the `originLostTimeout` field. Always use the `refreshOriginLostTimeout` and `tickOriginLostTimeout` methods to ensure the logic is handled correctly.
- **Detached Instances:** Creating a SpawnMarkerBlockReference with `new` without immediately attaching it to an entity serves no purpose. The component is meaningless outside the context of a host entity within the EntityStore.
- **Cross-Thread Access:** Never read `getBlockPosition` or call `tickOriginLostTimeout` from an asynchronous task or different thread without proper synchronization with the main server thread.

## Data Pipeline
This component is a data container, not a pipeline processor. It participates in a larger entity lifecycle management flow.

> Flow:
> Spawner Block Logic -> Entity Creation -> **SpawnMarkerBlockReference attached** -> Entity Management System (tick) -> Check Origin Validity -> **Update Timeout** -> Despawn Entity (if timeout expires)

