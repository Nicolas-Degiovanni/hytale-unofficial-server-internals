---
description: Architectural reference for RespawnSystems
---

# RespawnSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Utility / Container

## Definition
```java
// Signature
public class RespawnSystems {
    // Contains multiple static inner System classes
}
```

## Architecture & Concepts
The RespawnSystems class is not a traditional object but a static container, acting as a namespace for a collection of related systems within the Entity Component System (ECS) framework. Its sole purpose is to group the various logic modules that execute sequentially when an entity respawns.

Architecturally, this class embodies the "System" part of ECS. Each inner class is a distinct, single-responsibility system that operates on entities possessing specific components. The central design pattern is event-driven logic triggered by a component state change. Specifically, all systems within this container are subclasses of OnRespawnSystem, which itself is a RefChangeSystem that monitors the `DeathComponent`.

The core trigger for all respawn logic is the **removal** of the `DeathComponent` from an entity. This signifies the end of the "dead" state and the beginning of the "respawning" process. The engine's System Manager invokes each of these systems in a deterministic order, forming a well-defined pipeline for resetting an entity's state.

## Lifecycle & Ownership
- **Creation:** The RespawnSystems container class itself is never instantiated. Its inner system classes (e.g., ResetStatsRespawnSystem, ClearEntityEffectsRespawnSystem) are instantiated once by the server's core ECS engine during the server bootstrap and registration phase.
- **Scope:** Instances of the inner systems are singletons managed by the ECS engine. They persist for the entire lifetime of the server. They do not hold per-entity state and are designed to be reused for all respawn events.
- **Destruction:** The system instances are destroyed and garbage collected only when the server shuts down and the ECS engine is dismantled.

## Internal State & Concurrency
- **State:** The RespawnSystems class and all its inner systems are fundamentally **stateless**. They do not maintain any internal fields that track entity-specific data between invocations. All necessary state is passed into their `onComponentRemoved` methods via parameters like the entity Ref, the Store, and the CommandBuffer.
- **Thread Safety:** These systems are designed to be thread-safe under the execution model of the Hytale ECS engine. The engine guarantees that system logic for a given entity is not executed concurrently. The use of a CommandBuffer for all entity modifications is a critical concurrency pattern. Instead of modifying entity state directly, systems queue up operations (e.g., `commandBuffer.getComponent`, `effectControllerComponent.clearEffects`). The CommandBuffer then executes these commands in a safe, synchronized manner at the end of the current engine tick, preventing race conditions and data corruption.

## API Surface
The public API is defined by the abstract parent class `OnRespawnSystem`, which all concrete systems implement. The primary contract is the `onComponentRemoved` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(ref, component, store, commandBuffer) | void | O(N) | **Core Hook.** Invoked by the ECS engine when a `DeathComponent` is removed from an entity matching the system's query. Complexity is dependent on the specific system's logic (e.g., O(N) for iterating stats). |
| getQuery() | Query | O(1) | Defines which entities this system will operate on. For example, some systems only target entities with a `Player` component. |

## Integration Patterns

### Standard Usage
Developers do not interact with these systems directly. They are automatically registered and managed by the server's ECS framework. The engine identifies entities that have had their `DeathComponent` removed and passes them to the `onComponentRemoved` method of each relevant system.

A developer would typically add a new respawn behavior by creating a new inner class that follows the established pattern:

```java
// Example of adding a new system to the RespawnSystems container
public static class MyNewRespawnSystem extends RespawnSystems.OnRespawnSystem {
    @Nonnull
    @Override
    public Query<EntityStore> getQuery() {
        // Target entities that have a CustomComponent
        return CustomComponent.getComponentType();
    }

    public void onComponentRemoved(
        @Nonnull Ref<EntityStore> ref,
        @Nonnull DeathComponent component,
        @Nonnull Store<EntityStore> store,
        @Nonnull CommandBuffer<EntityStore> commandBuffer
    ) {
        // Add custom logic here, e.g., reset a score
        CustomComponent custom = commandBuffer.getComponent(ref, CustomComponent.getComponentType());
        if (custom != null) {
            custom.resetScore();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of a system using `new ResetStatsRespawnSystem()`. The ECS engine is responsible for the lifecycle of all systems. Manual instantiation will result in a non-functional object that is not registered to receive engine events.
- **Direct Invocation:** Do not call the `onComponentRemoved` method directly. This bypasses the engine's state management and CommandBuffer synchronization, which can lead to severe concurrency issues, data corruption, or game crashes.
- **Stateful Systems:** Do not add member variables to a system class to store temporary data. Systems must be stateless to ensure they can be safely reused for any entity at any time.

## Data Pipeline
The respawn process is a clear data transformation pipeline where the "event" is the removal of a component. Each system acts as a stage in this pipeline.

> Flow:
> Entity takes fatal damage -> `DeathComponent` is **added** -> Entity is in a "dead" state -> Timer or game logic completes -> `DeathComponent` is **removed** -> ECS Engine detects removal -> **RespawnSystems Pipeline Executes**:
> 1. **ClearInteractionsRespawnSystem**: Clears any active interactions.
> 2. **ClearEntityEffectsRespawnSystem**: Removes all status effects.
> 3. **ResetStatsRespawnSystem**: Resets health and other stats to their default values.
> 4. **CheckBrokenItemsRespawnSystem**: Checks inventory and sends a message if items are broken.
> 5. **ResetPlayerRespawnSystem**: Updates player-specific metadata like the last spawn time.
> 6. **RespawnControllerRespawnSystem**: The final stage, which invokes the world's `RespawnController` to handle the logic of physically placing the player back into the world.

