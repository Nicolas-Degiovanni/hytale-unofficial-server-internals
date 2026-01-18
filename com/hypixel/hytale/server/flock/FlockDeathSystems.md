---
description: Architectural reference for FlockDeathSystems
---

# FlockDeathSystems

**Package:** com.hypixel.hytale.server.flock
**Type:** System Container

## Definition
```java
// Signature
public class FlockDeathSystems {
   public static class EntityDeath extends DeathSystems.OnDeathSystem {
      // ...
   }

   public static class PlayerDeath extends DeathSystems.OnDeathSystem {
      // ...
   }
}
```

## Architecture & Concepts

FlockDeathSystems is a container for two distinct, highly specialized systems within the server's Entity Component System (ECS) framework. It is not a traditional object but rather a logical grouping of reactive handlers that manage the intersection of the **Flocking AI** and **Entity Death** mechanics.

Its primary responsibility is to ensure the integrity of flock data when a member entity dies. This is achieved by listening for the addition of a **DeathComponent** to entities and executing logic to clean up flock-related components and notify other flock members of the event.

The class is divided into two inner systems, each targeting a different entity type:

*   **EntityDeath:** Manages the death of all non-player living entities. This system contains the core logic for AI interaction, such as removing the dead NPC from its flock and notifying the attacker's flock that a kill has been secured. This notification is a critical trigger for subsequent AI behaviors, like changes in aggression or state.
*   **PlayerDeath:** A simpler system that handles player death. Its sole responsibility is to remove a player from any flock they might be a part of. This ensures that temporary alliances or charm effects involving flocks are correctly terminated upon player death.

These systems act as a critical bridge, decoupling the generic death processing pipeline from the specific business logic of the flocking AI.

### Lifecycle & Ownership

-   **Creation:** The inner system classes, EntityDeath and PlayerDeath, are not instantiated directly by developers. They are discovered and instantiated by the server's core System Registry during the server bootstrap sequence.
-   **Scope:** Instances of EntityDeath and PlayerDeath are singletons within the ECS world. They persist for the entire lifetime of a server session.
-   **Destruction:** The systems are destroyed and cleaned up when the server world is shut down and the ECS framework is torn down.

## Internal State & Concurrency

-   **State:** These systems are fundamentally **stateless**. They hold no mutable instance data. The Query field within EntityDeath is an immutable definition of the entities it targets. All stateful operations are performed on components retrieved from the shared ECS Store and changes are queued via the CommandBuffer.

-   **Thread Safety:** The systems are **not thread-safe** and must not be invoked from outside the main server game loop. They are designed to be executed exclusively by the single-threaded ECS scheduler. The use of a CommandBuffer for all component mutations is a key part of this design, ensuring that all state changes are deferred and applied atomically at the end of the current tick, preventing race conditions and partial state updates.

## API Surface

The public contract of these systems is not intended for direct invocation. The methods below are callbacks executed by the ECS framework.

### FlockDeathSystems.EntityDeath

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(ref, component, store, commandBuffer) | void | O(1) | **Framework Callback.** Triggered when a non-player entity receives a DeathComponent. Removes the entity from its flock and notifies the attacker's flock. |

### FlockDeathSystems.PlayerDeath

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(ref, component, store, commandBuffer) | void | O(1) | **Framework Callback.** Triggered when a Player entity receives a DeathComponent. Unconditionally removes the player's FlockMembership. |

## Integration Patterns

### Standard Usage

A developer does not interact with these systems directly. Interaction is implicit and declarative. To trigger the EntityDeath system, add a DeathComponent to an NPC entity.

```java
// Example: Killing an NPC in another system
// This action will implicitly trigger FlockDeathSystems.EntityDeath

void applyFatalDamage(CommandBuffer<EntityStore> commands, Ref<EntityStore> targetRef, Damage deathInfo) {
    DeathComponent death = new DeathComponent(deathInfo);
    commands.addComponent(targetRef, death);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance of these systems using `new`. The ECS framework is solely responsible for their lifecycle. `new FlockDeathSystems.EntityDeath()` will result in a non-functional object that is not registered with the engine.
-   **Manual Invocation:** Do not call the `onComponentAdded` method directly. This bypasses the ECS scheduler and its transactional guarantees, leading to severe state corruption and unpredictable behavior.
-   **Assuming Immediate State Change:** Component removals queued via the CommandBuffer are not immediate. They are processed at the end of the tick. Code in the same tick should not assume that a call to `tryRemoveComponent` has already taken effect.

## Data Pipeline

The systems operate as reactive processors in the entity death data pipeline. They are triggered by a state change (component addition) and produce further state changes as a result.

> Flow for NPC Death:
> Game Event (e.g., final damage dealt) -> `DeathSystems` adds **DeathComponent** to NPC -> ECS Scheduler detects component addition -> Invokes **FlockDeathSystems.EntityDeath** -> System reads attacker info from Damage component -> System queues removal of **FlockMembership** in CommandBuffer -> System calls `Flock.onTargetKilled` on attacker's flock -> End of Tick: CommandBuffer flushes all changes to the world state.

