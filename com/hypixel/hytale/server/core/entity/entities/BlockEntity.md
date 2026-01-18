---
description: Architectural reference for BlockEntity
---

# BlockEntity

**Package:** com.hypixel.hytale.server.core.entity.entities
**Type:** Component

## Definition
```java
// Signature
public class BlockEntity implements Component<EntityStore> {
```

## Architecture & Concepts

The BlockEntity is a fundamental server-side **Component** within Hytale's Entity Component System (ECS). It does not represent an entity by itself; rather, it imparts the behavior of a dynamic, physics-enabled block to an entity it is attached to. Its primary architectural role is to serve as the data-driven link between a generic entity and a specific BlockType asset.

This component is the cornerstone for features where world blocks become temporary, movable objects, such as a sand block falling due to gravity or a log rolling down a hill after being chopped.

Key architectural responsibilities include:
*   **Asset Association:** It holds a `blockTypeKey`, a string identifier that references a `BlockType` asset. This asset defines the block's visual appearance, hitbox properties, and other game-specific data.
*   **Physics Encapsulation:** It contains an instance of `SimplePhysicsProvider`. This design co-locates simple physics state (like velocity) and logic directly with the component, allowing the entity to be processed by the server's physics simulation systems.
*   **State Synchronization:** It manages a dirty flag, `isBlockIdNetworkOutdated`, to signal the networking layer when its core state has changed, ensuring clients receive updates.

Interaction with the ECS is managed through the `CommandBuffer`, a standard pattern to safely defer state mutations to a synchronized point in the server tick.

### Lifecycle & Ownership

The lifecycle of a BlockEntity is intrinsically tied to its parent entity. It is designed to be ephemeral.

*   **Creation:** A BlockEntity is never instantiated directly. The sole sanctioned method of creation is via the static factory method `assembleDefaultBlockEntity`. This method constructs a complete entity `Holder`, populating it with the BlockEntity component and other essential components like `TransformComponent`, `DespawnComponent`, and `Velocity`. This ensures that any entity with BlockEntity behavior is fully and correctly configured from inception.
*   **Scope:** The component persists for the lifetime of its parent entity. The default configuration applied by `assembleDefaultBlockEntity` includes a `DespawnComponent` with a 120-second timer. This indicates that BlockEntity instances are intended for short-lived, dynamic effects.
*   **Destruction:** The component is destroyed and garbage collected when its parent entity is removed from the `EntityStore`. This is typically triggered by the `DespawnComponent` timer expiring, or by game logic that converts the dynamic block entity back into a static world block.

## Internal State & Concurrency

*   **State:** The internal state of BlockEntity is highly **mutable**. The `blockTypeKey` can be changed post-creation, and the embedded `SimplePhysicsProvider` constantly updates its internal velocity and collision state during physics ticks. The `simplePhysicsProvider` field is marked as `transient`, which is a critical detail. It is excluded from serialization and must be re-initialized after the component is deserialized from storage or network streams.

*   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All interactions with a BlockEntity instance must be performed exclusively on the main server thread that processes the game loop. The use of `CommandBuffer` for mutations like `setBlockTypeKey` is the required pattern to prevent race conditions and ensure state changes are applied deterministically.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assembleDefaultBlockEntity(time, key, pos) | Holder<EntityStore> | O(1) | **Static Factory.** The only supported method for creating a new, fully-formed block entity. |
| initPhysics(boundingBox) | SimplePhysicsProvider | O(1) | Initializes the transient physics provider. **Warning:** This must be called after creation or deserialization. |
| updateHitbox(ref, commandBuffer) | BoundingBox | O(1) | Recalculates and updates the entity's BoundingBox component based on the current blockTypeKey. |
| setBlockTypeKey(key, ref, buffer) | void | O(1) | Mutates the block type, flags it for network update, and queues a hitbox update. |
| addForce(x, y, z) | void | O(1) | Applies an instantaneous velocity change to the entity. |
| consumeBlockIdNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. Intended for use by the network synchronization system. |

## Integration Patterns

### Standard Usage

The correct pattern for spawning a dynamic falling block is to use the static factory method and add the resulting entity holder to the world.

```java
// How a developer should normally use this
TimeResource time = context.getResource(TimeResource.class);
Vector3d spawnPosition = new Vector3d(100, 64, 100);

// Create the complete entity with all necessary components
Holder<EntityStore> blockEntityHolder = BlockEntity.assembleDefaultBlockEntity(time, "hytale:sand", spawnPosition);

// The entity must be initialized by the ECS to be active
// (Specific world/entity manager API call would go here)
world.addEntity(blockEntityHolder);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Do not use `new BlockEntity()`. The component is inert and useless without being part of a correctly assembled entity `Holder` containing a `TransformComponent`, `Velocity`, and other required components. This will lead to runtime exceptions and undefined behavior.

*   **Asynchronous State Mutation:** Do not modify the BlockEntity from other threads. Calling `addForce` or `setBlockTypeKey` from a network thread or worker pool will corrupt physics state and crash the server. All mutations must be dispatched to the main game loop.

*   **Ignoring Physics Initialization:** The `simplePhysicsProvider` is transient. If a BlockEntity is ever loaded from a persistent store, its physics will be uninitialized. Failure to call `initPhysics` will result in a `NullPointerException` when physics systems attempt to process the entity.

## Data Pipeline

The BlockEntity acts as a stateful component within the server's main processing loop, primarily driven by game events and the physics engine.

> Flow:
> Game Event (e.g., Block Destruction) -> `BlockEntity.assembleDefaultBlockEntity` -> **Entity Creation** -> Physics System reads `TransformComponent` & `Velocity` -> Physics System updates `SimplePhysicsProvider` -> Physics System writes new `TransformComponent` -> Network System reads `consumeBlockIdNetworkOutdated` -> **Network Packet Sent to Client**

