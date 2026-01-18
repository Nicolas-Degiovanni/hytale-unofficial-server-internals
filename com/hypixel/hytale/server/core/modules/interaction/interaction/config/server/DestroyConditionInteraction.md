---
description: Architectural reference for DestroyConditionInteraction
---

# DestroyConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration-Driven Component

## Definition
```java
// Signature
@Deprecated
public class DestroyConditionInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The DestroyConditionInteraction is a server-authoritative component within the Interaction System. Its primary function is not to perform an action, but to act as a **conditional gate** in a larger interaction sequence. It validates whether a target block can be destroyed by a given entity, effectively enforcing game rules defined within the block's state itself.

This class is a concrete example of a data-driven design pattern. The static CODEC field indicates that instances are not created programmatically but are deserialized from configuration assets. This allows designers to build complex interaction chains by composing components like this one without writing new Java code.

Its most critical architectural characteristic is its server-side exclusivity. By returning `WaitForDataFrom.Server` and providing an empty client-side simulation method, it forces the client to pause and wait for the server's authoritative validation. This prevents exploits where a client might incorrectly predict that a block can be destroyed.

**WARNING:** This class is marked as **Deprecated**. It should not be used for new development and is maintained for legacy compatibility only. Future systems may use a more generalized condition-checking mechanism.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the `BuilderCodec` deserialization system when the server loads interaction graph assets from configuration files. It is never instantiated directly with the `new` keyword.
- **Scope:** An instance is a stateless template that persists as long as the game's configuration assets are loaded in memory. The same instance can be reused across countless interactions.
- **Destruction:** Instances are eligible for garbage collection when the server unloads the asset bundles containing the interaction configurations, typically during a server shutdown or a hot-reload of game data.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to its methods during an interaction event.
- **Thread Safety:** The class is inherently **thread-safe**. As it has no internal state to mutate, a single instance can be safely invoked by multiple threads simultaneously. However, the caller is responsible for ensuring that the `World` and `CommandBuffer` arguments are accessed in a thread-safe manner according to the engine's concurrency model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.Server`, signaling that this interaction logic is server-authoritative and has no client-side prediction. |
| interactWithBlock(...) | void | O(1) | Executes the core validation logic. Checks if the target `BlockState` implements `BreakValidatedBlockState` and calls its `canDestroy` method. Mutates the `InteractionContext` state to `Failed` or `Finished`. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. Intentionally left empty to prevent any client-side simulation for this validation step. |

## Integration Patterns

### Standard Usage
This component is not used directly in code. Instead, it is specified within a data file (e.g., JSON) that defines an interaction sequence. The server's Interaction Module deserializes this configuration and invokes the `interactWithBlock` method when a player attempts the corresponding action.

A conceptual configuration might look like this:
```json
"onBlockPrimaryAction": [
  {
    "type": "DestroyConditionInteraction"
  },
  {
    "type": "ApplyDamageToBlock"
  },
  {
    "type": "PlaySoundEffect"
  }
]
```
In this sequence, the `DestroyConditionInteraction` runs first. If it sets the context state to `Failed`, the subsequent `ApplyDamageToBlock` and `PlaySoundEffect` interactions are aborted.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DestroyConditionInteraction()`. The system relies on the `CODEC` for instantiation from data files.
- **Client-Side Invocation:** Attempting to execute this logic on the client will have no effect and violates the server-authoritative design.
- **Usage in New Systems:** Do not add this component to new interaction graphs. Its `@Deprecated` status indicates it is being phased out.

## Data Pipeline
The flow of data during this interaction step is strictly one-way, from the world state to the interaction context.

> Flow:
> Server Interaction Event -> **DestroyConditionInteraction.interactWithBlock** -> Read `World` for `BlockState` -> Call `BlockState.canDestroy` -> Write `InteractionState.Failed` or `InteractionState.Finished` to `InteractionContext` -> Interaction System reads context to proceed or halt.

