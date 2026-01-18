---
description: Architectural reference for StatsConditionInteraction
---

# StatsConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Data-Driven Component

## Definition
```java
// Signature
public class StatsConditionInteraction extends StatsConditionBaseInteraction {
```

## Architecture & Concepts
The StatsConditionInteraction is a server-side component that functions as a conditional gate within the broader Interaction System. Its primary architectural role is to determine if an entity can "afford" to perform an action by validating its current statistics against a predefined set of costs.

This class is not a standalone service but rather a modular, reusable piece of logic. It is designed to be defined within data files (e.g., JSON or HOCON) and loaded at runtime. This pattern decouples the game mechanics of "cost" from the implementation of the "effect", allowing designers to create complex interactions by chaining together simple, single-purpose components like this one.

For example, a "Fireball" spell might be an interaction chain composed of:
1.  A **StatsConditionInteraction** to check for sufficient mana.
2.  An `ApplyEffectInteraction` to consume the mana.
3.  A `LaunchProjectileInteraction` to cast the spell.

If the StatsConditionInteraction fails, the entire chain is aborted, preventing subsequent interactions from running.

### Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically using the `new` keyword. They are deserialized from game asset configuration files by the server's core asset management system. The static `CODEC` field is the entry point for this data-driven instantiation.
-   **Scope:** An instance of StatsConditionInteraction is effectively a singleton for its specific configuration. It is loaded once on server startup and persists in memory for the entire server session as a shared, immutable template.
-   **Destruction:** The object is garbage collected only when the server shuts down or performs a full reload of its game configuration assets.

## Internal State & Concurrency
-   **State:** **Immutable**. All configurable fields (costs, lessThan, valueType) are set once during deserialization from configuration files. At runtime, the object is a stateless processor; it reads external state from the `InteractionContext` but never modifies its own internal state.

-   **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature, a single instance can be safely referenced and executed by multiple threads simultaneously without any need for locks or synchronization primitives. This is critical in a server environment where many entities may be performing actions at the same time.

## API Surface
The public contract is focused on executing the condition check within the server's interaction pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Entry point for the interaction system. Executes the `canAfford` check and sets the context state to `Failed` if the conditions are not met. N is the number of stats in the `costs` map. |
| canAfford(ref, componentAccessor) | boolean | O(N) | The core validation logic. Fetches the entity's `EntityStatMap` and compares each required stat against its configured cost. |
| generatePacket() | Interaction | O(1) | Creates a network packet representing this interaction's configuration, used for client-server state synchronization. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by typical game logic code. It is designed to be configured by a game designer and executed automatically by the server's Interaction Module. The following example illustrates how the *system* would invoke it, not a typical developer.

```java
// This logic resides deep within the server's Interaction Module.
// A developer would not write this code.

InteractionContext context = createInteractionContextForEntity(playerEntity);
StatsConditionInteraction manaCheck = interactionRegistry.get("spell.fireball.manacheck");

// The system invokes firstRun to execute the check
manaCheck.firstRun(InteractionType.PRIMARY, context, cooldownHandler);

if (context.getState().state != InteractionState.Failed) {
    // Proceed with the rest of the interaction chain
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new StatsConditionInteraction()`. This bypasses the critical data-driven initialization from game configuration files, resulting in a non-functional object. All instances must be loaded via the server's asset system.
-   **Stateful Logic:** Do not extend this class to add mutable state. These objects are shared across the entire server; any instance-specific state will cause severe concurrency issues and unpredictable behavior for all entities using the interaction.

## Data Pipeline
The primary flow for this component involves reading entity state to make a decision that influences the control flow of a larger system.

> Flow:
> Player Action Event -> Server Interaction Module -> **StatsConditionInteraction.firstRun()** -> Reads `EntityStatMap` Component -> Writes `InteractionState` to `InteractionContext` -> Interaction Module Halts or Continues Chain

