---
description: Architectural reference for StartObjectiveInteraction
---

# StartObjectiveInteraction

**Package:** com.hypixel.hytale.builtin.adventure.objectives.interactions
**Type:** Configuration-Driven Object

## Definition
```java
// Signature
public class StartObjectiveInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The StartObjectiveInteraction class is a concrete implementation of an interaction handler within the server's core interaction module. Its primary role is to act as a bridge between a player's action (typically using an item) and the server's adventure and questing system.

This class is not intended for direct programmatic use. Instead, it is designed to be instantiated and configured declaratively through Hytale's data-driven systems, identified by its static CODEC field. Game designers associate this interaction with a specific item or entity in configuration files. When a player interacts with that item, the server's interaction module invokes this handler to execute its logic.

The core logic follows two distinct paths:
1.  **New Objective Creation:** If the item used in the interaction does not have an objective UUID in its metadata, this class will instantiate a new Objective based on its internal `objectiveTypeSetup` configuration. It then "stamps" the item by writing the new Objective's UUID back into the item's metadata, effectively linking that specific item instance to the newly created quest instance.
2.  **Existing Objective Participation:** If the item already contains an objective UUID, the class will not create a new objective. Instead, it will add the interacting player to the existing objective instance identified by the UUID.

This mechanism allows for items to act as starters for both personal quests and shared, persistent world quests.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during the loading of game assets and configurations. The static `CODEC` field defines how to deserialize configuration data into a valid StartObjectiveInteraction object. Direct instantiation via the `new` keyword is an anti-pattern and will result in a non-functional object.
-   **Scope:** An instance of this class is a stateless template. It persists for as long as the game configuration that defines it is loaded, typically for the entire server session. The execution of its logic via the `firstRun` method is transient and occurs only at the moment of a player interaction.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup when the server unloads the associated game assets or during server shutdown. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The primary internal state is the `objectiveTypeSetup` field. This field is populated once during deserialization and is treated as **immutable** thereafter. The class itself is effectively stateless during method execution, as all required contextual data is provided via the `InteractionContext` parameter.
-   **Thread Safety:** This class is **not thread-safe** for arbitrary concurrent access. It is designed to be called exclusively from the server's main game loop thread. The `firstRun` method operates on a `CommandBuffer`, a critical pattern for deferring world state mutations to a safe point in the server tick. This ensures that all modifications to entities and components are synchronized with the main game loop, preventing race conditions.

## API Surface
The public contract is defined by its parent class, `SimpleInstantInteraction`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(C) | The primary entry point. Executes the objective start/join logic. Complexity is dependent on the configuration of the objective being started. Throws NullPointerException if `objectiveTypeSetup` was not initialized. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. A game designer configures it within an item's definition file. The server's interaction system automatically discovers and executes it.

*Example configuration snippet (conceptual):*
```json
{
  "itemId": "hytale:quest_scroll",
  "interaction": {
    "type": "StartObjectiveInteraction",
    "Setup": {
      "objectiveIdToStart": "hytale:main_quest_01",
      // ... other objective setup data
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new StartObjectiveInteraction()`. The object will be invalid as its internal `objectiveTypeSetup` field will be null, causing a NullPointerException when `firstRun` is invoked.
-   **Manual Invocation:** Do not call the `firstRun` method directly. This bypasses the server's interaction pipeline, including critical systems like cooldowns, permissions, and command buffer synchronization, which can lead to world state corruption.

## Data Pipeline
The flow of data and control for this component is managed entirely by the server's core systems.

> Flow:
> Player Input (Use Item) -> Server Network Layer -> Interaction Module -> **StartObjectiveInteraction.firstRun()** -> ObjectivePlugin -> Writes to CommandBuffer -> CommandBuffer applies changes to EntityStore (World State)

