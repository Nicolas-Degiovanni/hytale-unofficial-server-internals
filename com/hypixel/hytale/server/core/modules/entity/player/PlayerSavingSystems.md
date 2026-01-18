---
description: Architectural reference for PlayerSavingSystems
---

# PlayerSavingSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class PlayerSavingSystems {
    // Contains nested static system classes:
    // public static class SaveDataResource implements Resource<EntityStore>
    // public static class TickingSystem extends EntityTickingSystem<EntityStore>
    // public static class WorldRemovedSystem extends StoreSystem<EntityStore>
}
```

## Architecture & Concepts

The PlayerSavingSystems class is not a traditional object but a static namespace that groups related ECS (Entity Component System) systems responsible for player data persistence. It provides the core logic for both periodic "autosaving" during gameplay and a final, guaranteed save upon world shutdown. This separation of concerns into distinct systems is a fundamental pattern in the server architecture.

The class defines two primary systems:

1.  **TickingSystem:** A periodic system that runs continuously with the game loop. It is responsible for automatically saving player data at a fixed interval (10 seconds). To optimize performance, it only performs a save operation if it detects a change in the player's position, head rotation, inventory, or other configuration data. This prevents redundant disk I/O for idle players.

2.  **WorldRemovedSystem:** A lifecycle-aware system that acts as a shutdown hook. Its logic is executed only once when its parent ECS Store (representing a game world) is being destroyed. This system guarantees that all player data is flushed to storage before the world unloads, preventing data loss from server stops or world transitions. After saving, it disconnects all associated players.

These systems operate on entities possessing a specific `Archetype`: a combination of `Player`, `TransformComponent`, and `HeadRotation` components.

### Lifecycle & Ownership

The systems defined within PlayerSavingSystems are not instantiated directly. They are managed by the ECS framework and their lifecycle is bound to a `Store` instance, which typically corresponds to a `World`.

-   **Creation:** Instances of `TickingSystem` and `WorldRemovedSystem` are created during the server's world initialization sequence. A higher-level bootstrap process, likely part of the world or module loader, is responsible for instantiating these systems and registering them with the world's primary `Store`.
-   **Scope:** An instance of each system persists for the entire lifetime of the `Store` it is registered with. State, such as the autosave timer, is managed within a `Resource` that is also owned by the `Store`.
-   **Destruction:** The systems are garbage collected when the `Store` is destroyed. The framework guarantees that `onSystemRemovedFromStore` on `WorldRemovedSystem` is invoked just before the `Store` becomes unreachable, enabling the final save operation.

## Internal State & Concurrency

-   **State:** The outer PlayerSavingSystems class is a stateless namespace.
    -   **TickingSystem** is stateful via a `Resource` object, `SaveDataResource`, which is owned by the `Store`. This resource contains a mutable `delay` field to track the time until the next save check. This design ensures that state is tied to the world instance, not the system class itself.
    -   **WorldRemovedSystem** is stateless. It operates on the components present in the `Store` at the moment of its execution.

-   **Thread Safety:** These systems are designed for concurrent execution and rely on the ECS scheduler for thread safety.
    -   `TickingSystem` implements `isParallel`, allowing the ECS scheduler to process different entity chunks on multiple threads simultaneously.
    -   `WorldRemovedSystem` uses `store.forEachEntityParallel` to explicitly parallelize the final save and disconnect operations across all player entities.
    -   All cross-thread operations and structural changes to the ECS world (e.g., entity deletion) are funneled through a `CommandBuffer`, a standard pattern for ensuring thread-safe modifications from within a parallel system.

    **WARNING:** Direct, unsynchronized access to components managed by these systems from outside the ECS scheduler will lead to race conditions and data corruption.

## API Surface

The public contract of these systems is intended for the ECS framework, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TickingSystem(playerComponentType) | constructor | O(1) | Creates the periodic saving system. Requires the `Player` component type for its query. |
| WorldRemovedSystem(playerComponentType) | constructor | O(1) | Creates the shutdown saving system. Requires the `Player` component type for its query. |
| TickingSystem.tick(...) | void | O(N) | Framework callback. Executes one tick of the save-check logic for N entities. |
| WorldRemovedSystem.onSystemRemovedFromStore(...) | void | O(N) | Framework callback. Triggers final save and disconnect for all N players in the store. |

## Integration Patterns

### Standard Usage

These systems are not used by calling their methods directly. They should be instantiated and registered with a `Store` as part of a world's system configuration.

```java
// Example of registering the systems during world setup
Store<EntityStore> worldStore = world.getStore();
ComponentRegistry registry = worldStore.getComponentRegistry();

// Obtain the required component type
ComponentType<EntityStore, Player> playerType = registry.getComponentType(Player.class);

// Instantiate and add systems to the store's scheduler
worldStore.addSystem(new PlayerSavingSystems.TickingSystem(playerType));
worldStore.addSystem(new PlayerSavingSystems.WorldRemovedSystem(playerType));
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation and Invocation:** Do not create an instance of `TickingSystem` and call its `tick` method manually. The ECS scheduler is responsible for managing its lifecycle, state, and execution context.
-   **Manual Timer Management:** Do not attempt to manage the save timer from outside the `TickingSystem`. The internal `SaveDataResource` is managed by the system and its `Store`.
-   **Assuming Immediate Save:** Do not assume that a change to a player's inventory or position will be saved instantly. The save only occurs when the `TickingSystem`'s internal timer elapses or when the world is shut down.

## Data Pipeline

The class facilitates two distinct data pipelines for player persistence.

**1. Periodic Autosave Pipeline (TickingSystem)**

> Flow:
> Game Loop Tick -> ECS Scheduler -> **TickingSystem.tick()** -> Check Save Timer & Component Dirty Flags -> `Player.saveConfig()` -> World Storage (Disk)

**2. World Shutdown Pipeline (WorldRemovedSystem)**

> Flow:
> World Shutdown Signal -> `Store.destroy()` -> ECS Scheduler -> **WorldRemovedSystem.onSystemRemovedFromStore()** -> `Player.saveConfig()` -> World Storage (Disk) -> `PacketHandler.disconnect()` -> Network Layer

