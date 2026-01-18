---
description: Architectural reference for DamageBlockEvent
---

# DamageBlockEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Object

## Definition
```java
// Signature
public class DamageBlockEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The DamageBlockEvent is a fundamental message object within the server-side Entity Component System (ECS). It represents the discrete action of a block in the world receiving damage, but *before* that damage is finalized and applied to the world state.

This class serves as a data carrier that decouples the cause of damage (e.g., a player action, an explosion) from its numerous potential effects and modifications. By extending CancellableEcsEvent, it creates a critical interception point in the game logic. Various systems can listen for this event, inspect its payload, and subsequently alter the outcome. This pattern allows for complex, emergent gameplay mechanics such as tool enchantments, area protection, and custom block behaviors without creating tight coupling between systems.

This event is a mutable container. Its primary purpose is to be passed through a chain of listeners, each of which may read or write its properties, before a final system consumes the result and updates the world.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a high-level game logic system when an action that could damage a block occurs. For example, the PlayerInteractionSystem would create and dispatch this event in response to a player attempting to break a block.
-   **Scope:** Extremely short-lived. An instance of DamageBlockEvent exists only for the duration of its dispatch on the ECS event bus, which occurs synchronously within a single game tick.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the event bus has finished notifying all listeners. It holds no persistent state or external resources and requires no explicit cleanup. **Warning:** Holding a reference to this event beyond the scope of an event handler is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** Highly mutable. The fields for damage and targetBlock are intentionally exposed via setters. This mutability is a core design feature, enabling event listeners to dynamically modify the damage amount (e.g., applying a bonus from a tool) or even redirect the damage to a different block. The initial state represents the raw, unmodified action.
-   **Thread Safety:** **This class is not thread-safe.** All ECS events are designed to be created, dispatched, and handled within the single main server thread. Any attempt to access or modify an event instance from a worker thread will result in race conditions, data corruption, and undefined server behavior. All modifications must occur synchronously within the event listener's callback.

## API Surface
The public contract includes getters for initial state and setters for state modification. Inherited methods from CancellableEcsEvent, such as setCancelled(true), are also a critical part of the API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemInHand() | ItemStack | O(1) | Retrieves the item, if any, used to inflict the damage. May be null. |
| getTargetBlock() | Vector3i | O(1) | Retrieves the world-space coordinate of the block being damaged. |
| setTargetBlock(target) | void | O(1) | Modifies the target block. **Warning:** Use with extreme caution as this can have significant gameplay side-effects. |
| getBlockType() | BlockType | O(1) | Retrieves the static type definition of the block being damaged. |
| getCurrentDamage() | float | O(1) | Retrieves the accumulated damage on the block *before* this event is processed. |
| getDamage() | float | O(1) | Retrieves the amount of damage to be applied by this specific event. |
| setDamage(damage) | void | O(1) | Modifies the damage to be applied. This is the primary hook for systems that alter damage values. |

## Integration Patterns

### Standard Usage
The canonical use case is for a system to subscribe to this event via the ECS event bus. The listener then inspects the event's context and applies its own logic, potentially modifying the outcome.

```java
// Example: A system that doubles damage from diamond pickaxes
@Subscribe
public void onBlockDamaged(DamageBlockEvent event) {
    ItemStack tool = event.getItemInHand();
    if (tool != null && tool.isA(DIAMOND_PICKAXE)) {
        float originalDamage = event.getDamage();
        event.setDamage(originalDamage * 2.0f);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not store a reference to a DamageBlockEvent instance in a field or long-lived collection. Its data is only relevant for the instant it is being processed.
-   **Asynchronous Handling:** Do not dispatch this event to a separate thread for processing. The server's game loop expects the event's modifications to be complete before the end of the current tick.
-   **Manual Instantiation:** Creating an instance with `new DamageBlockEvent()` has no effect on the game world. Events must be dispatched through the server's central event bus to be processed by listeners.

## Data Pipeline
The DamageBlockEvent acts as a transient data packet in a larger processing pipeline. Its flow represents a chain of responsibility where data is enriched and validated at each step.

> Flow:
> Player Input Packet -> Server Network Layer -> PlayerInteractionSystem -> **DamageBlockEvent (Creation & Dispatch)** -> Event Bus -> ProtectionSystem (Listener) -> ToolBehaviorSystem (Listener) -> BlockHealthSystem (Final Consumer) -> World State Change

