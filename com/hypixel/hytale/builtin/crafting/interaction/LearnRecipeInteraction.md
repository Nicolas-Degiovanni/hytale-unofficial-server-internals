---
description: Architectural reference for LearnRecipeInteraction
---

# LearnRecipeInteraction

**Package:** com.hypixel.hytale.builtin.crafting.interaction
**Type:** Data-Driven Object

## Definition
```java
// Signature
public class LearnRecipeInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The LearnRecipeInteraction is a server-authoritative component within the Interaction System responsible for a single, specific action: teaching a player a crafting recipe. It is a concrete implementation of the **Command Pattern**, where an entire operation is encapsulated into a standalone object.

This class extends SimpleInstantInteraction, which signifies that the action it performs is immediate, atomic, and completes within a single server tick. It does not have a duration, charge-up time, or complex state machine.

Architecturally, it acts as a bridge between a player-initiated event and the server's core crafting system. It is not intended to be invoked procedurally from game logic. Instead, instances are created by the engine's deserialization layer (the Codec system) based on game data files, typically attached to an item's definition. When a player interacts with that item, the engine invokes this object's logic, passing in the full world state via an InteractionContext.

Its primary mechanism for world mutation is the CommandBuffer, a key element of the engine's Entity Component System (ECS) architecture. By queueing state changes, it ensures that all modifications are applied deterministically and safely at the end of the tick, preventing race conditions.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the new keyword. They are instantiated by the engine's asset loader via the public static CODEC field. This process typically occurs at server startup when parsing game assets, such as item definition files.
- **Scope:** An instance of LearnRecipeInteraction has a scope tied to the asset that defines it (e.g., an Item). It persists in memory as a stateless, reusable configuration object for as long as the parent asset is loaded.
- **Destruction:** The object is eligible for garbage collection when the asset registry unloads the corresponding game assets, for example, during a server shutdown or when a game module is disabled.

## Internal State & Concurrency
- **State:** The class holds a single configuration field, itemId, which is set upon deserialization. This state is considered immutable after initialization. During execution of its logic, the class is effectively stateless; all transactional data is provided through the InteractionContext parameter.
- **Thread Safety:** This class is **not thread-safe** and must only be invoked by the server's main game thread. The surrounding Interaction System guarantees this contract. All world-state modifications are funneled through the CommandBuffer, which defers the actual mutation to a controlled, synchronized phase of the server tick, thus ensuring data integrity.

## API Surface
The primary contract is defined by the parent SimpleInstantInteraction class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldown) | void | O(1) | Executes the recipe learning logic. This is the main entry point called by the Interaction System. Throws NullPointerException if the context is invalid. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, indicating this is a server-authoritative interaction that resolves instantly without client input. |

## Integration Patterns

### Standard Usage
This class is not used directly in Java code by developers. It is designed to be configured within data files (e.g., JSON) and attached to an item. The engine handles its lifecycle and invocation.

Below is a conceptual example of how this interaction would be defined for a "Recipe Scroll" item.

```json
// Example: my_scroll.json (Conceptual Item Definition)
{
  "id": "my_mod:recipe_scroll_iron_sword",
  "components": {
    "hytale:interaction": {
      "type": "hytale:learn_recipe",
      "ItemId": "hytale:iron_sword"
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LearnRecipeInteraction()`. The object is not designed for manual creation and will lack its configured itemId. Always define it in data files to be loaded by the engine.
- **Manual Invocation:** Do not call the firstRun method directly. Doing so bypasses the Interaction System's state management, providing an incomplete or null InteractionContext, which will lead to runtime exceptions and unpredictable world state.

## Data Pipeline
The flow of data for this interaction begins with player input and ends with a change in the player's persistent data. The LearnRecipeInteraction is a single, critical step in this server-side process.

> Flow:
> Player Input -> Network Packet -> Server Interaction System -> **LearnRecipeInteraction.firstRun()** -> CraftingPlugin.learnRecipe() -> CommandBuffer Write -> PlayerRef Component Update -> Player Data Persistence

