---
description: Architectural reference for NPCDeathSystems
---

# NPCDeathSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility

## Definition
```java
// Signature
public class NPCDeathSystems {
    public static class EntityViewSystem extends DeathSystems.OnDeathSystem {
        // ...
    }

    public static class NPCKillsEntitySystem extends DeathSystems.OnDeathSystem {
        // ...
    }
}
```

## Architecture & Concepts

The NPCDeathSystems class is not an executable system itself, but rather a static container for two distinct, highly specialized systems that integrate the engine's core damage and death mechanics with the Non-Player Character (NPC) AI framework. These inner systems act as event listeners, responding specifically to the moment an entity is marked for death.

Their primary architectural function is to translate a low-level engine event—the addition of a DeathComponent to an entity—into high-level, semantically meaningful information for the AI.

1.  **EntityViewSystem:** This system serves as a perception bridge. When an NPC or Player dies, this system's responsibility is to publish a formal DEATH event into the AI's **Blackboard**. The Blackboard is the central nervous system for AI, and this event allows other NPCs in the vicinity to perceive the death and react accordingly, for example, by becoming hostile or fleeing.

2.  **NPCKillsEntitySystem:** This system handles the internal bookkeeping for the aggressor. When an NPC successfully kills another entity, this system is triggered to update the killer NPC's internal state. This is critical for tracking kill counts, satisfying quest objectives, or modifying behavior based on combat success.

These systems are fundamental to creating a reactive and stateful AI world. They ensure that death is not merely an entity's removal from the game, but a tangible event that influences the behavior of other actors.

### Lifecycle & Ownership

-   **Creation:** The outer NPCDeathSystems class is never instantiated. The inner static classes, EntityViewSystem and NPCKillsEntitySystem, are discovered via reflection and instantiated by the server's central SystemRegistry during world initialization.
-   **Scope:** Instances of the inner systems persist for the entire lifetime of the server world. They are singletons within the context of a given world instance.
-   **Destruction:** The systems are destroyed and eligible for garbage collection only when the server world is shut down and the SystemRegistry is cleared.

## Internal State & Concurrency

-   **State:** The NPCDeathSystems container is stateless. The inner systems are also fundamentally stateless. They hold references to ComponentType and ResourceType definitions, which are immutable, but they do not store any per-entity or per-tick data themselves. All state they operate on is passed into the onComponentAdded method via the Store and CommandBuffer parameters.
-   **Thread Safety:** These systems are designed to be executed by the engine's single-threaded or managed-thread scheduler. They are not thread-safe for arbitrary concurrent access. All interactions with entity components occur through the provided Store (for reads) and CommandBuffer (for writes), which guarantees transactional integrity within a single engine tick. Direct, multi-threaded invocation of these systems would lead to race conditions and world state corruption.

## API Surface

The public API of the inner systems consists of framework-level callbacks, not intended for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityViewSystem.onComponentAdded(...) | void | O(1) | **Framework Hook.** Triggered when a watched entity dies. Publishes a death event to the chunk-local Blackboard. |
| NPCKillsEntitySystem.onComponentAdded(...) | void | O(1) | **Framework Hook.** Triggered when any living entity dies. Updates the internal state of the attacker if it was an NPC. |

## Integration Patterns

### Standard Usage

A developer does not interact with these systems directly. They are automatically registered and executed by the server's Entity Component System (ECS) engine. The engine identifies all classes extending DeathSystems.OnDeathSystem and wires them into the appropriate execution phase.

The only "interaction" is indirect: by designing NPC behaviors that listen for the DEATH event on the Blackboard, or by creating NPC components that use the kill data tracked by the DamageData component.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new NPCDeathSystems.EntityViewSystem()`. The systems require management by the ECS engine to receive the correct Store and CommandBuffer context. Manual instantiation will result in a non-functional object.
-   **Manual Invocation:** Do not call the onComponentAdded method directly. This bypasses the ECS scheduler and its state management, and will almost certainly cause world state corruption or NullPointerExceptions, as the system will lack the necessary context.
-   **Modifying Queries at Runtime:** The queries returned by getQuery are expected to be static. Attempting to change the query after initialization is unsupported and will lead to undefined behavior.

## Data Pipeline

The flow of data through these systems is event-driven, originating from the core damage simulation and terminating in the AI's state and perception layers.

> **Flow 1: AI Perception of Death**
>
> Entity takes lethal damage -> Core `DeathSystems` adds `DeathComponent` -> **`EntityViewSystem`** is triggered -> System accesses global `Blackboard` resource -> System retrieves chunk-local `EntityEventView` -> System calls `processAttackedEvent(DEATH)` -> Event is now visible to AI behaviors in that chunk.

> **Flow 2: NPC Kill Tracking**
>
> Entity takes lethal damage -> Core `DeathSystems` adds `DeathComponent` -> **`NPCKillsEntitySystem`** is triggered -> System identifies the attacker as an `NPCEntity` -> System calls `NPCEntity.getDamageData().onKill()` -> The killer NPC's internal state is updated.

