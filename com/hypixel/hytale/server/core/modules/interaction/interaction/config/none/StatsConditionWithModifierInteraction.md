---
description: Architectural reference for StatsConditionWithModifierInteraction
---

# StatsConditionWithModifierInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient / Data-Driven Component

## Definition
```java
// Signature
public class StatsConditionWithModifierInteraction extends StatsConditionBaseInteraction {
```

## Architecture & Concepts
The StatsConditionWithModifierInteraction is a server-side, data-driven component responsible for validating whether an entity can afford the resource cost of an interaction, after applying modifiers from equipped armor. It represents a specific rule within the broader Interaction System.

This class extends StatsConditionBaseInteraction, inheriting the fundamental logic for checking an entity's stats. Its unique responsibility is to calculate a *dynamic* cost at runtime. Before comparing the entity's current stat value against the required cost, it inspects the entity's equipped armor for any `StaticModifier` definitions that match the interaction's configured `InteractionModifierId`. This allows game designers to create items that reduce the cost of specific actions, such as a "Wizard Hat" that lowers the mana cost of spells.

Instances of this class are not manually created in code. Instead, they are deserialized from asset files using the provided Hytale `CODEC`. This design pattern decouples the game logic from the game content, allowing designers to create complex interactions without modifying server code. It acts as a stateless, reusable rule object that is invoked by the core interaction processing loop.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the Hytale `Codec` system during server startup when game assets (e.g., item definitions in JSON files) are loaded into memory. It is never instantiated directly with the `new` keyword in game logic.
-   **Scope:** An instance of this class is effectively a singleton for a given interaction definition. It is shared by all game objects that reference it. Its lifetime is tied to the server's asset registry, persisting for the entire server session.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down and the asset registry is cleared.

## Internal State & Concurrency
-   **State:** The object is effectively immutable after its creation by the codec. Its fields, such as `interactionModifierId` and the inherited `costs` map, are configured once during asset loading and are not mutated during gameplay. The class is stateless; all runtime data is passed in via the `InteractionContext` parameter.
-   **Thread Safety:** This class is inherently thread-safe for read operations. Its methods operate on data provided by a `ComponentAccessor`, which is part of the broader Entity Component System (ECS). The responsibility for ensuring safe access to and modification of entity components (like `EntityStatMap` and `Inventory`) lies with the calling system, typically the main server thread or a dedicated world-processing thread, which guarantees a consistent view of the game state. The class itself contains no locks or synchronization primitives.

## API Surface
The primary contract is fulfilled by overriding methods from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(C * A) | Entry point for the interaction check. Fails the interaction if `canAfford` returns false. C is the number of stat costs; A is the number of armor slots. |
| canAfford(ref, componentAccessor) | boolean | O(C * A) | Core logic. Checks if the entity's stats meet the required costs after applying all relevant armor modifiers. |

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class directly in Java code. Instead, it is configured within an asset file. The server's interaction system automatically locates and executes this logic when a corresponding event occurs.

The following is a conceptual example of how this class might be defined in a game asset file:

```json
// Example: my_spell_item.json
{
  "id": "my_spell",
  "interaction": {
    "type": "StatsConditionWithModifierInteraction",
    "costs": {
      "mana": 20
    },
    "interactionModifierId": "SPELL_COST"
  }
}
```

When a player uses this item, the server's `InteractionModule` will find the `StatsConditionWithModifierInteraction` instance configured for it and invoke its `firstRun` method.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new StatsConditionWithModifierInteraction()`. The object will be unconfigured and will not function. All instances must be created via the asset loading system and its `CODEC`.
-   **State Mutation:** Do not attempt to modify the `interactionModifierId` or `costs` fields at runtime. As this instance is shared, such a change would affect every item using this interaction definition, leading to unpredictable global side effects.

## Data Pipeline
This class acts as a conditional gate within the server-side interaction handling pipeline.

> Flow:
> Player Input Packet -> Server Network Layer -> InteractionModule -> **StatsConditionWithModifierInteraction.firstRun()** -> Read EntityStatMap & Inventory Components -> Return Success/Failure -> InteractionModule proceeds or aborts the action.

