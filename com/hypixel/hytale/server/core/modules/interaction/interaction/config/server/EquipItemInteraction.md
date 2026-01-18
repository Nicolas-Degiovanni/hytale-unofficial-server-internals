---
description: Architectural reference for EquipItemInteraction
---

# EquipItemInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient / Configuration Object

## Definition
```java
// Signature
public class EquipItemInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The EquipItemInteraction class is a server-authoritative, data-driven command that represents the action of equipping an item from an entity's hand to a corresponding armor slot. It is a concrete implementation of the *Strategy Pattern* within the server's core interaction system. This class is not a long-lived service but rather a stateless, single-purpose object that encapsulates a specific game rule.

Architecturally, it acts as a bridge between a high-level player action ("use item") and the low-level, transactional inventory system. Its primary responsibility is to validate the context of the interaction—ensuring the entity is a LivingEntity and the item is a piece of armor—and then to orchestrate an inventory transaction to perform the move.

By extending SimpleInstantInteraction, it signals to the engine that this action resolves within a single server tick and does not require a multi-stage, timed process (like eating or drawing a bow).

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via code using the *new* keyword. Instead, they are deserialized from asset configuration files by the server's asset loading pipeline using the provided static CODEC. The core Interaction Module instantiates it on-demand when an entity performs an action with an item configured to use this interaction.
- **Scope:** The object's lifetime is extremely short, scoped to a single interaction event. It is created, its `firstRun` method is executed once, and then it is discarded.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the interaction logic completes within the server tick. It holds no persistent references.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields to store data. All information required for its operation is passed into the `firstRun` method via the InteractionContext parameter. This design makes the class highly predictable and reusable across any number of interactions.

- **Thread Safety:** The class is inherently thread-safe. As it has no internal state, there are no race conditions to consider within the class itself. All stateful operations are performed on thread-safe or thread-confined objects provided by the InteractionContext, such as the CommandBuffer. The CommandBuffer pattern defers all entity mutations, ensuring they are applied sequentially at a safe point in the main server loop, thus preventing concurrency issues.

## API Surface
The public contract is minimal, designed to be invoked exclusively by the server's interaction processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, indicating this is a server-authoritative action. The client must wait for the server's response and cannot predict the outcome. |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the core logic. Attempts to move the held item to an armor slot. Mutates the InteractionContext state to Failed if the transaction cannot be completed. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly from imperative game logic code. It is designed to be declared within an item's configuration file (e.g., a JSON asset). The engine's interaction system reads this configuration and executes the interaction automatically.

*Conceptual Item Configuration:*
```json
{
  "id": "my_game:iron_helmet",
  "itemType": "ARMOR",
  "interactions": [
    {
      "type": "EquipItemInteraction"
    }
  ]
}
```
When a player holding this item performs the "use" action, the server automatically finds and executes the EquipItemInteraction logic.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new EquipItemInteraction()`. The class is not designed for manual creation and relies on being part of the configuration-driven interaction system.
- **Manual Invocation:** Do not call the `firstRun` method from other systems. It requires a fully-formed InteractionContext that is only correctly assembled by the core Interaction Module during event processing. Bypassing this will lead to inconsistent state and likely throw NullPointerExceptions.

## Data Pipeline
The flow of data for this interaction is strictly server-side and follows a clear, transactional path from player input to world state change.

> Flow:
> Client Input Packet -> Server Interaction Module -> **EquipItemInteraction.firstRun** -> Inventory MoveTransaction -> CommandBuffer Write -> EntityStore Update -> State Synchronization to Client

