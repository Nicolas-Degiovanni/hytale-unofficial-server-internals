---
description: Architectural reference for DeployablesUtils
---

# DeployablesUtils

**Package:** com.hypixel.hytale.builtin.deployables
**Type:** Utility

## Definition
```java
// Signature
public class DeployablesUtils {
```

## Architecture & Concepts
DeployablesUtils is a static utility class that serves as the primary factory and controller for the deployable entity system. It provides a high-level, declarative API for creating and managing temporary game objects like turrets, mines, or other placeable items.

This class acts as a crucial abstraction layer over the raw Entity Component System (ECS). Instead of manually constructing an entity and attaching dozens of required components, game logic can call a single function, `spawnDeployable`, passing in a data-driven `DeployableConfig` asset. The utility then orchestrates the entire entity creation process, ensuring all necessary components such as `TransformComponent`, `ModelComponent`, `EntityStatMap`, and `DespawnComponent` are correctly configured and added to the world.

The use of a `CommandBuffer` in the `spawnDeployable` method is a key architectural choice. It signifies that entity creation is not immediate but is a deferred, transactional operation. This guarantees that the world state remains consistent, as all entity additions are processed in a controlled manner at the end of the current game tick.

## Lifecycle & Ownership
As a static utility class, DeployablesUtils itself is stateless and has no lifecycle. However, it is solely responsible for defining the lifecycle of the deployable entities it creates.

-   **Creation:** Deployable entities are brought into existence exclusively through the `spawnDeployable` method. This is the designated and only supported entry point for creating a valid deployable.
-   **Scope:** The lifetime of a created entity is dictated by the `liveDurationInMillis` property within its `DeployableConfig`. If this value is positive, a `DespawnComponent` is attached, scheduling the entity for automatic removal. If the duration is zero or negative, the entity will persist until destroyed by other game mechanics, such as taking damage.
-   **Destruction:** This class does not directly handle entity destruction. Destruction is managed by the core `DespawnComponent` system or other combat-related systems that remove entities from the `EntityStore`.

## Internal State & Concurrency
-   **State:** DeployablesUtils is entirely stateless. It holds no member variables and all operations are performed on data passed in as method arguments. It is a pure function library operating on the world state provided via the `Store` and `CommandBuffer`.

-   **Thread Safety:** This class is **not thread-safe** and is fundamentally tied to the main server game loop. Its methods perform direct reads and deferred writes on core ECS structures like `Store` and `CommandBuffer`. Calling any method from this class on an asynchronous thread will lead to severe concurrency issues, including `ConcurrentModificationException`, world state corruption, and server crashes. All interactions with DeployablesUtils must be synchronized with the main game tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spawnDeployable(...) | Ref<EntityStore> | O(k) | Factory method to create and schedule a new deployable entity. Complexity is relative to the number of components (k) defined in the config. |
| playAnimation(...) | void | O(n) | Broadcasts an animation packet to all clients (n) that can see the specified entity. |
| stopAnimation(...) | void | O(n) | Broadcasts a packet to stop an animation on all relevant clients (n). |
| playSoundEventsAtEntity(...) | void | O(n) | Plays 2D and 3D sound events. 3D sounds are broadcast to nearby clients (n). |

## Integration Patterns

### Standard Usage
The primary integration pattern involves retrieving the necessary ECS context (`CommandBuffer`, `Store`) and invoking `spawnDeployable` with a configuration asset. This is typically done within a system that handles player actions, such as item usage.

```java
// Example: Spawning a deployable when a player uses an item
DeployableConfig config = getDeployableConfigForItem(item);
Ref<EntityStore> playerRef = getPlayerEntityRef();
Vector3f position = getTargetSpawnPosition();
Vector3f rotation = getTargetSpawnRotation();

// The CommandBuffer and Store are provided by the ECS scheduler to the system
DeployablesUtils.spawnDeployable(
    commandBuffer,
    store,
    config,
    playerRef,
    position,
    rotation,
    "ground"
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The class is a static utility. Attempting to create an instance via `new DeployablesUtils()` is impossible and indicates a misunderstanding of its purpose.
-   **Asynchronous Spawning:** Never call `spawnDeployable` or other methods from a separate thread or an asynchronous task. This will break the server's tick-based world state management and cause unpredictable crashes.
-   **Manual Component Population:** Do not attempt to replicate the logic of `spawnDeployable` by manually creating an entity and adding components. This bypasses critical setup steps, such as stat initialization and owner linking, resulting in a broken or incomplete entity.

## Data Pipeline
The data flow for the most critical operation, spawning, is a clear transformation from configuration to world state.

> Flow:
> `DeployableConfig` Asset -> `spawnDeployable` Method -> **`CommandBuffer` (Deferred Operation)** -> `EntityStore` Update -> Entity appears in world

For client-facing effects like animations and sounds, the pipeline terminates at the network layer for broadcast.

> Flow:
> Game Logic Trigger -> **`DeployablesUtils.playAnimation`** -> `PlayAnimation` Packet -> Network Subsystem -> Visible to Players

