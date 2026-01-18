---
description: Architectural reference for NPCDamageSystems
---

# NPCDamageSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility

## Definition
```java
// Signature
public class NPCDamageSystems {
    // Contains nested static system classes
}
```

## Architecture & Concepts
The NPCDamageSystems class is not a system itself, but a static container or namespace for a collection of highly specialized, inner classes that implement the server-side damage and death logic for Non-Player Characters (NPCs). These systems are fundamental to the server's Entity-Component-System (ECS) architecture and integrate directly with the core DamageModule.

Each nested class represents a distinct stage in the damage processing pipeline, from initial validation to final consequences like death and item drops. This modular design allows for a clear separation of concerns and hooks into specific, well-defined phases of the engine's damage event lifecycle.

The primary roles of this collection of systems are:
1.  **Filtering:** To determine if an NPC can be damaged by a specific source.
2.  **State Tracking:** To record damage taken and dealt in the NPCEntity component for AI and gameplay purposes.
3.  **AI Notification:** To translate low-level damage events into high-level AI stimuli within the Blackboard system.
4.  **Death Consequence:** To handle the effects of an NPC's death, primarily by spawning loot drops.

These systems operate on entities matching specific queries, typically those possessing an NPCEntity component, and are executed automatically by the server's main game loop as part of dedicated SystemGroups.

### Lifecycle & Ownership
-   **Creation:** The nested system classes are instantiated by the server's ECS framework during the world initialization or system registration phase. They are not intended for manual creation.
-   **Scope:** Instances of these systems persist for the entire lifetime of the server world. They are stateless processors that are re-used every game tick.
-   **Destruction:** The systems are destroyed and cleaned up when the server world is shut down.

## Internal State & Concurrency
-   **State:** The systems within NPCDamageSystems are designed to be **stateless**. They do not maintain any mutable state across ticks or method invocations. All required data is passed in as arguments to their `handle` or `onComponentAdded` methods, or accessed via the provided Store and CommandBuffer. State is stored exclusively within ECS components (e.g., NPCEntity) and resources (e.g., Blackboard).
-   **Thread Safety:** These systems are designed to be run by a single-threaded game loop or within a system graph that guarantees exclusive access to the components they operate on during execution. The use of a CommandBuffer for all entity modifications (e.g., adding entities, changing components) is a critical concurrency pattern. It defers all structural changes to a synchronization point at the end of the tick, preventing race conditions and data corruption. Direct modification of the Store is not permitted for structural changes.

## API Surface
The public contract consists of the handler methods within each nested system, which are invoked by the ECS engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FilterDamageSystem.handle(...) | void | O(1) | Intercepts a damage event before it is applied. May cancel the event based on NPC logic. |
| DamageDealtSystem.handle(...) | void | O(1) | Updates the attacker's NPCEntity state when it successfully inflicts damage. |
| DamageReceivedSystem.handle(...) | void | O(1) | Updates the victim's NPCEntity state when it suffers damage. |
| DamageReceivedEventViewSystem.handle(...) | void | O(k) | Publishes a high-level "attacked" event to the AI Blackboard for nearby NPCs to sense. |
| DropDeathItems.onComponentAdded(...) | void | O(N) | Triggered when a DeathComponent is added to an NPC. Generates and spawns item drop entities. |

## Integration Patterns

### Standard Usage
Developers do not interact with these systems directly. They are automatically registered and executed by the server's `DamageModule`. To influence their behavior, a developer would modify the data in the components these systems read from, such as NPCEntity or an entity's Inventory.

For example, to define what an NPC drops on death, you would configure its `dropListId` via its Role component, which is then read by the DropDeathItems system.

```java
// PSEUDOCODE: Configuring an NPC that will be processed by these systems.
// This is done during entity creation, not by calling the systems.

NPCEntity npc = ...;
Role npcRole = npc.getRole();

// The DropDeathItems system will read this value upon the NPC's death.
npcRole.setDropListId("goblin_warrior_loot_table");

// The FilterDamageSystem might use this flag to prevent friendly fire.
npc.setFaction(Faction.MONSTERS);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of these systems using `new`. The ECS framework manages their lifecycle. Direct instantiation will result in a non-functional object that is not registered with the engine.
-   **Manual Invocation:** Do not call the `handle` or `onComponentAdded` methods directly. These methods require a complex and valid state (ArchetypeChunk, Store, CommandBuffer) that is only provided by the ECS scheduler during the game tick. Manual calls will lead to unpredictable behavior and likely throw NullPointerExceptions or IllegalStateExceptions.
-   **Stateful Implementation:** Do not add member variables to these system classes to store state between ticks. This violates the stateless design principle and will cause severe bugs in a multi-threaded or re-entrant environment.

## Data Pipeline
The systems in NPCDamageSystems form a sequential and conditional pipeline for processing a single damage instance against an NPC.

> Flow:
> 1.  **Damage Event Creation**: An external system (e.g., combat logic) creates a `Damage` object and submits it to the `DamageModule`.
> 2.  **Filtering Phase (`FilterDamageGroup`)**: The `DamageModule` executes systems in this group first.
>     -   **FilterDamageSystem**: Intercepts the event. It checks game logic (e.g., factions, invulnerability) via `npcComponent.getCanCauseDamage`. If it returns false, the system calls `damage.setCancelled(true)`. The pipeline terminates for this event.
> 3.  **Inspection Phase (`InspectDamageGroup`)**: If the damage was not cancelled, the `DamageModule` executes systems in this group.
>     -   **DamageDealtSystem**: If the attacker is an NPC, its `DamageData` is updated.
>     -   **DamageReceivedSystem**: The victim NPC's `DamageData` is updated with the incoming damage.
>     -   **DamageReceivedEventViewSystem**: The system accesses the `Blackboard` resource and publishes an `EntityEventType.DAMAGE` event, making the attack perceivable to the AI of other nearby entities.
> 4.  **Death Processing**: If the damage applied in the previous phase reduces the NPC's health to zero, the core health system adds a `DeathComponent` to the entity.
> 5.  **On-Death Trigger (`OnDeathSystem`)**: The addition of the `DeathComponent` triggers systems that listen for this change.
>     -   **DropDeathItems**: This system activates. It reads the NPC's `Role` and `Inventory` to determine which items to drop, creates new item entities via `ItemComponent.generateItemDrops`, and adds them to the world using the `CommandBuffer`.

