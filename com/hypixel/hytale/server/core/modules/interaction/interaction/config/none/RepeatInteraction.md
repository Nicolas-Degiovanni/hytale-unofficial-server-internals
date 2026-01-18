---
description: Architectural reference for RepeatInteraction
---

# RepeatInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Stateful Component

## Definition
```java
// Signature
public class RepeatInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The RepeatInteraction is a fundamental control-flow node within the server-side Interaction System. Architecturally, it functions as a stateful `for` loop, enabling a sub-graph of interactions to be executed a specified number of times. It is not a standalone service but a component that exists as part of a larger, declarative `InteractionChain`.

Its primary role is to fork the main execution path, manage the lifecycle of a temporary, nested `InteractionChain`, and then repeat this process based on its configuration. This allows for complex, looping behaviors without requiring imperative script code.

The component's design sharply distinguishes between its static configuration and its dynamic runtime state:
*   **Configuration:** Defined by the `CODEC`, which deserializes properties like `ForkInteractions` (the asset reference to the sub-chain) and `Repeat` (the loop count) from game data files. This configuration is immutable once loaded.
*   **Runtime State:** Managed externally via a `DynamicMetaStore` supplied by the `InteractionContext`. State such as the currently active forked chain (`FORKED_CHAIN`) and the remaining loop count (`REMAINING_REPEATS`) is stored here. This pattern keeps the RepeatInteraction object itself stateless and reusable, while its execution state is tied to a specific context.

The internal logic operates as a finite state machine, driven by the `tick0` method. It transitions between forking a new chain, waiting for that chain to complete, and deciding whether to repeat or to transition to its `Next` or `Failed` interaction path.

## Lifecycle & Ownership

-   **Creation:** RepeatInteraction instances are not created directly via their constructor. They are instantiated by the engine's `BuilderCodec` system during the deserialization of game assets (e.g., entity definitions, quest data). The `InteractionManager` is responsible for loading and preparing the interaction graphs that contain these nodes.
-   **Scope:** The configuration object persists as long as the parent asset is loaded in memory. However, its *runtime state* is ephemeral and is scoped strictly to its execution within a specific `InteractionContext`. This state is created when the node becomes active in an `InteractionChain` and is discarded once it transitions to a terminal state (Finished or Failed).
-   **Destruction:** The configuration object is eligible for garbage collection when its parent asset is unloaded. The runtime state, stored in the `DynamicMetaStore`, is implicitly destroyed when the `InteractionContext` and its parent `InteractionChain` are terminated and cleaned up by the `InteractionManager`.

## Internal State & Concurrency

-   **State:** The component is highly stateful during execution, but this state is managed externally.
    -   **Immutable Configuration:** The `forkInteractions` and `repeat` fields are set upon deserialization and are treated as immutable for the object's lifetime.
    -   **Mutable Runtime State:** The active forked chain and remaining repeat count are stored in the `InteractionContext`'s `DynamicMetaStore`. The `tick0` method reads and writes this state on every execution frame, making the operation inherently mutable.

-   **Thread Safety:** **This class is not thread-safe and must not be considered as such.** The entire Interaction System, including this component, is designed to be executed serially on the main server game thread. The `tick0` method performs unchecked mutations on the shared `InteractionContext`. Concurrent access would corrupt the interaction's state machine, leading to undefined behavior, race conditions, and server instability.

## API Surface

The primary contract is with the Interaction System's ticking mechanism.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(firstRun, time, type, context, cooldownHandler) | void | O(1) | Executes a single logic step. Manages the state of the forked chain, checks for completion, and updates the context state. This method drives the entire lifecycle of the interaction. |
| walk(collector, context) | boolean | O(N) | Traverses the interaction graph for analysis or asset collection, where N is the number of nodes in all reachable sub-graphs (forked, next, failed). |
| needsRemoteSync() | boolean | O(1) | Returns true, indicating that the state and configuration of this interaction must be synchronized with the client for correct prediction and presentation. |

## Integration Patterns

### Standard Usage

RepeatInteraction is not used imperatively. It is defined declaratively within game data files (e.g., JSON) and executed by the `InteractionManager`. A developer would define its behavior as part of an entity or world object's interaction graph.

*Hypothetical JSON Asset Definition:*
```json
{
  "id": "my_repeating_task",
  "type": "RepeatInteraction",
  "ForkInteractions": "asset:hytale:interactions/chains/spawn_and_attack",
  "Repeat": 5,
  "Next": "my_success_interaction",
  "Failed": "my_failure_interaction"
}
```

The engine then executes this as part of a larger chain. The developer does not call `tick0` directly.

```java
// Engine-level code (conceptual)
// An entity's InteractionChain is ticked by the InteractionManager.
InteractionChain chain = entity.getActiveInteractionChain();
chain.tick(context); // This will eventually call RepeatInteraction.tick0 if it's the active node.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new RepeatInteraction()`. This bypasses the `CODEC` deserialization process, resulting in an unconfigured and non-functional object. Interactions must always be loaded from assets.
-   **External State Mutation:** Do not manually modify the `FORKED_CHAIN` or `REMAINING_REPEATS` values in the `DynamicMetaStore` from outside this class. Doing so will break the internal state machine and cause unpredictable behavior.
-   **Asynchronous Execution:** Do not attempt to call `tick0` from a separate thread. All interactions within a chain must be ticked sequentially on the main server thread.

## Data Pipeline

RepeatInteraction acts as a control-flow router, not a data transformer. Its "pipeline" concerns the flow of execution control within the Interaction System.

> Flow:
> `InteractionManager` ticks parent chain -> **`RepeatInteraction.tick0()`** is invoked -> A new `InteractionChain` is created via `context.fork()` -> The new chain is passed to the `InteractionManager` for execution -> The forked chain completes, setting its state -> **`RepeatInteraction.tick0()`** detects the state change -> Execution is passed to the `Next` or `Failed` interaction.

