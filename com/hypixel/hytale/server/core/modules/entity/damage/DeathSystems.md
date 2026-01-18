---
description: Architectural reference for DeathSystems
---

# DeathSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Utility / System Namespace

## Definition
```java
// Signature
public class DeathSystems {
    // Contains only static nested System classes.
    // Not designed for instantiation.
}
```

## Architecture & Concepts

The DeathSystems class is not a singular, stateful object but rather a static namespace that groups a collection of highly-specialized systems within the Entity Component System (ECS) framework. These systems collectively orchestrate the entire lifecycle of an entity's death, from the initial fatal blow to final cleanup.

The central architectural pattern is event-driven, with the **DeathComponent** acting as the primary trigger. When an entity's health is reduced to zero or it is otherwise killed, a DeathComponent is added to it. This component addition is the "event" that activates the various systems defined within DeathSystems.

These systems subscribe to this event through the `OnDeathSystem` base class, which is a specialized `RefChangeSystem`. This ensures they execute precisely when the DeathComponent is added. Each system is responsible for a distinct aspect of the death process:

*   **State Reset:** Clearing health, effects, and active interactions.
*   **Visuals & Feedback:** Playing death animations and broadcasting kill feed messages.
*   **Gameplay Logic:** Handling item drops, creating death markers for players, and displaying the respawn screen.
*   **Cleanup:** Managing the entity's corpse and eventual removal from the world.

The execution order of these systems is not arbitrary; it is strictly controlled by the ECS scheduler using explicit **System Dependencies**. For example, `PlayerDropItemsConfig` is guaranteed to run *before* `DropPlayerDeathItems` to ensure the drop behavior is configured correctly before being executed. This declarative dependency management is critical for maintaining a predictable and robust death sequence.

## Lifecycle & Ownership

-   **Creation:** The `DeathSystems` class itself is never instantiated. The nested system classes (e.g., `ClearHealth`, `CorpseRemoval`) are discovered and instantiated by the server's ECS System Registry during the server bootstrap sequence.
-   **Scope:** Once instantiated by the framework, these systems persist for the entire lifetime of the server session. They are stateless singletons within the context of the ECS world.
-   **Destruction:** The systems are destroyed and cleaned up when the server shuts down and the ECS world is torn down. There is no manual destruction logic required.

## Internal State & Concurrency

-   **State:** The DeathSystems class is stateless. The individual systems it contains are also fundamentally stateless. They operate exclusively on data stored in components (like DeathComponent, Player, TransformComponent) and ECS resources (like WorldTimeResource). All state changes are managed through a `CommandBuffer`, which safely queues operations to be applied at the end of the current tick.

-   **Thread Safety:** These systems are designed to be executed by the main server thread as part of the world tick. The ECS scheduler guarantees that systems are executed in a deterministic, single-threaded sequence for a given world. Therefore, manual locking or other concurrency controls are not required within the systems themselves. All interactions with entity state are inherently thread-safe within the context of the game loop.

## API Surface

The DeathSystems class does not expose a traditional public API. Its "surface" consists of the system classes it provides to the ECS framework. A developer does not call methods on DeathSystems; instead, they trigger its functionality by adding a DeathComponent to an entity.

| System Class | Base Type | Responsibility |
| :--- | :--- | :--- |
| ClearEntityEffects | OnDeathSystem | Removes all active status effects from the dying entity. |
| ClearHealth | OnDeathSystem | Sets the entity's Health stat to zero. |
| ClearInteractions | OnDeathSystem | Cancels any ongoing interactions (e.g., attacks, blocking). |
| CorpseRemoval | EntityTickingSystem | After death animations complete, manages corpse lifetime and eventual entity deletion. |
| DeathAnimation | OnDeathSystem | Plays the appropriate death animation based on entity type and cause of death. |
| DropPlayerDeathItems | OnDeathSystem | Handles the logic for dropping a player's inventory items upon death. |
| KillFeed | OnDeathSystem | Generates and broadcasts a kill feed message to relevant players. |
| PlayerDeathMarker | OnDeathSystem | Creates a persistent map marker at the player's location of death. |
| PlayerDeathScreen | OnDeathSystem | Opens the "You Died" UI screen for the affected player client. |
| RunDeathInteractions | OnDeathSystem | Executes any configured death interactions, such as special effects or logic scripts. |

## Integration Patterns

### Standard Usage

The primary and **only** correct way to initiate the death process is to add a `DeathComponent` to an entity using a `CommandBuffer`. This delegates the entire complex death sequence to the registered DeathSystems.

```java
// Correctly initiating the death of an entity.
// This typically happens after a damage calculation.

// Assume 'entityRef' is the reference to the entity that should die.
// Assume 'commandBuffer' is available from the current system's context.
// Assume 'finalDamage' contains details about the killing blow.

DeathComponent deathComponent = new DeathComponent(finalDamage);
commandBuffer.addComponent(entityRef, deathComponent);

// The ECS framework will now automatically invoke the appropriate
// systems from DeathSystems in the correct order.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create instances of the nested system classes (e.g., `new CorpseRemoval()`). The ECS framework is solely responsible for their lifecycle. Direct instantiation will lead to unmanaged, non-functional system objects.

-   **Manual Execution:** Do not attempt to call system methods like `onComponentAdded` directly. These methods are callbacks intended for the ECS scheduler. Bypassing the scheduler breaks dependencies, transactionality, and thread safety guarantees.

-   **Premature Component Modification:** Avoid directly modifying components on a dying entity from an unrelated system. The DeathSystems are ordered to prevent race conditions. For example, do not try to read inventory *after* `DropPlayerDeathItems` has run, as it may already be cleared.

## Data Pipeline

The death process represents a clear, sequential data pipeline orchestrated by the ECS framework. The flow begins with a gameplay event and ends with the entity's removal and corresponding client-side effects.

> Flow:
> Damage Event -> DamageSystem calculates fatal damage -> `CommandBuffer.addComponent(entity, DeathComponent)` -> **[ECS Tick Boundary]** -> `DeathSystems.ClearHealth` -> `DeathSystems.ClearEntityEffects` -> `DeathSystems.DeathAnimation` -> `DeathSystems.RunDeathInteractions` -> `DeathSystems.DropPlayerDeathItems` -> `DeathSystems.KillFeed` -> `DeathSystems.PlayerDeathScreen` -> `DeathSystems.CorpseRemoval` (over time) -> `CommandBuffer.removeEntity(entity)`

