---
description: Architectural reference for PrefabSelectionInteraction
---

# PrefabSelectionInteraction

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Transient Handler

## Definition
```java
// Signature
public class PrefabSelectionInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The PrefabSelectionInteraction class is a concrete implementation of the `SimpleInstantInteraction` contract. It serves as a command object that encapsulates the logic for selecting a prefab within an active `PrefabEditSession`.

This class is a critical component of the server-side Builder Tools plugin. It is not a long-lived service but a data-driven, stateless handler. The Hytale interaction system invokes this class when a player, holding a configured "prefab selection tool", performs a primary or secondary action (e.g., left or right-click).

Its primary architectural role is to act as a translator between low-level player input and high-level state changes within the prefab editing system. It queries the game world for the player's target and uses this information to update the player's `PrefabEditSession`, which is managed by the `PrefabEditSessionManager`. The class itself holds no state, ensuring that each interaction is an atomic and isolated operation.

## Lifecycle & Ownership
- **Creation:** This class is not instantiated directly using its constructor in gameplay code. Instead, it is instantiated by the engine's `BuilderCodec` system when loading game assets. An interaction configuration, likely defined in a JSON asset file, references this class via its `CODEC`. The engine then registers this interaction handler, associating it with a specific item or action.
- **Scope:** An instance of this class is effectively a singleton managed by the interaction system's registry. However, its *execution* is scoped to a single, transient player action. The logic within the `firstRun` method is invoked and completes within a single server tick.
- **Destruction:** The object is destroyed when the server unloads the assets that define it, typically during a server shutdown or a hot-reload of game data.

## Internal State & Concurrency
- **State:** PrefabSelectionInteraction is **stateless**. It contains no mutable instance fields. All state it reads from or writes to, such as the player's current session or the list of loaded prefabs, is owned and managed by external systems like `PrefabEditSessionManager` and `PrefabEditSession`. This design makes the interaction predictable and idempotent.

- **Thread Safety:** This class is **not thread-safe** and is designed to be operated exclusively by the server's main game thread. The `firstRun` method receives a `CommandBuffer` from its `InteractionContext`, which is an inherently single-threaded mechanism for queueing world modifications. Any attempt to invoke this logic from an asynchronous task or worker thread will lead to race conditions and world state corruption.

## API Surface
The public API is defined by its parent class, `SimpleInstantInteraction`. The primary entry point is the `firstRun` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the prefab selection logic. Complexity is O(N) where N is the number of loaded prefabs in the player's session. This method is the sole entry point for the interaction's logic. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is designed to be configured in game data assets and invoked automatically by the server's interaction system. The system identifies the interaction to trigger based on the item a player is holding and the action they perform.

A conceptual representation of its invocation by the engine:

```java
// ENGINE-LEVEL CODE (Conceptual)
// This code is handled by the server's interaction module.
// Do not replicate this pattern.

// 1. Player performs an action.
InteractionType type = InteractionType.Primary;
InteractionContext context = createInteractionContextForPlayer(player);

// 2. Engine looks up the interaction configured for the player's held item.
SimpleInstantInteraction interaction = getInteractionForItem(player.getHeldItem());

// 3. If the interaction is a PrefabSelectionInteraction, its logic is run.
if (interaction instanceof PrefabSelectionInteraction) {
    interaction.firstRun(type, context, new CooldownHandler());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PrefabSelectionInteraction()`. The class relies on its `CODEC` for proper initialization and registration within the engine's data-driven systems.
- **Stateful Modification:** Do not modify this class to include instance fields or any form of internal state. It must remain a stateless command object to ensure predictable behavior. All state must be stored in dedicated session or component objects.
- **External Invocation:** Do not call the `firstRun` method from other systems. It is strictly designed to be called by the interaction module, which provides a valid and correctly-scoped `InteractionContext`. Calling it manually will result in a null or invalid context, leading to unpredictable exceptions.

## Data Pipeline
The flow of data for a prefab selection event begins with player input and results in a state change within a server-side session manager.

> Flow:
> Player Input (Click) -> Server Network Listener -> Interaction Module -> **PrefabSelectionInteraction.firstRun()** -> TargetUtil Raycast -> PrefabEditSessionManager -> PrefabEditSession State Update

