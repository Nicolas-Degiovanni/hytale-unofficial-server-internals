---
description: Architectural reference for ConfigurableInstanceBlock
---

# ConfigurableInstanceBlock

**Package:** com.hypixel.hytale.builtin.instances.blocks
**Type:** Data Component

## Definition
```java
// Signature
public class ConfigurableInstanceBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The ConfigurableInstanceBlock is a server-side data component that attaches configuration to a block entity, effectively turning that block into a gateway for a world instance. It does not contain any logic itself; instead, it serves as a data source for other systems that manage world instances, player teleportation, and entity lifecycle.

Architecturally, this component is a fundamental part of Hytale's data-driven world design. Its properties are defined in external data files and deserialized at runtime via its static **CODEC**. This allows designers and creators to configure complex instancing behavior without modifying engine code.

The component's behavior upon destruction is managed by the nested **OnRemove** class, a **RefSystem**. This system subscribes to the removal event for any entity possessing a ConfigurableInstanceBlock. This is a critical design pattern within the engine's Entity Component System (ECS), ensuring that component-specific cleanup logic is automatically executed when an entity is removed from the world. In this case, it guarantees that the associated world instance can be safely torn down when its entry-point block is destroyed.

## Lifecycle & Ownership
- **Creation:** A ConfigurableInstanceBlock is not created directly. It is instantiated by the engine's serialization layer when a chunk containing the associated block entity is loaded from storage. The static **CODEC** field governs this deserialization process. Alternatively, it can be added programmatically to an entity via a **CommandBuffer** by a higher-level game logic system.
- **Scope:** The component's lifetime is strictly bound to the entity to which it is attached. It persists as long as the block entity exists within a loaded chunk in the world.
- **Destruction:** The component is destroyed when its parent entity is removed from the **Store**. This action triggers the **OnRemove** system, which executes cleanup logic, such as closing the linked world instance if **closeOnRemove** is true.

## Internal State & Concurrency
- **State:** The component's state is entirely **mutable**. It is a Plain Old Java Object (POJO) designed to hold configuration data. It contains a **CompletableFuture** field, **worldFuture**, which introduces an asynchronous element to its state, representing a world that may still be loading.

- **Thread Safety:** This component is **not thread-safe** and must not be considered as such. All interactions, including reads and writes, must be performed on the primary world thread where the owning entity is processed. Accessing this component from worker threads or asynchronous callbacks without proper synchronization or scheduling will lead to race conditions, data corruption, and server instability.

    **WARNING:** The **worldFuture** field may complete on a separate thread. Any logic chained to it via methods like **thenAccept** must be scheduled back to the main game thread before modifying any game state, including other components.

## API Surface
The public API consists primarily of accessors for its configuration properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the unique type identifier for this component within the ECS. |
| getWorldUUID() | UUID | O(1) | Returns the unique identifier of the target world instance. |
| getInstanceKey() | String | O(1) | Returns a key used for identifying or grouping specific instances. |
| isCloseOnRemove() | boolean | O(1) | Determines if the associated world instance should be shut down when this block is removed. |
| clone() | Component | O(N) | Creates a deep copy of the component's configuration data. |

## Integration Patterns

### Standard Usage
Systems should query for entities with this component to implement logic such as player teleportation. The component's data is read-only from the perspective of most systems, with the **OnRemove** system handling its primary reactive logic.

```java
// Example of a system that processes player interaction with an instance block
void onPlayerInteract(Ref<ChunkStore> blockEntityRef, CommandBuffer<ChunkStore> commands) {
    // Retrieve the component from the entity. Do not create it directly.
    ConfigurableInstanceBlock instanceData = commands.getComponent(blockEntityRef, ConfigurableInstanceBlock.getComponentType());

    if (instanceData != null) {
        UUID targetWorld = instanceData.getWorldUUID();
        // Trigger player teleportation logic using the retrieved UUID...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use **new ConfigurableInstanceBlock()** in game logic. Components must be managed by the ECS. To add one, use a **CommandBuffer** to attach it to an entity.
- **Asynchronous State Modification:** Do not modify the component's state from a callback on the **worldFuture** without scheduling the operation back to the main thread. This will break thread-safety guarantees of the game loop.
- **Ignoring Nullability:** Fields like **positionOffset** and **rotation** are nullable. Always perform null checks before dereferencing them to prevent NullPointerExceptions.

## Data Pipeline
The component acts as a data container whose lifecycle is managed by the ECS and whose state change (destruction) triggers a command pipeline.

> Flow:
> World Data File -> Engine **CODEC** Deserialization -> **ConfigurableInstanceBlock** attached to Entity -> Entity Removal Event -> **OnRemove** System -> **InstancesPlugin.safeRemoveInstance** Command -> World Instance Shutdown

