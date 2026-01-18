---
description: Architectural reference for SelectInteraction
---

# SelectInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient

## Definition
```java
// Signature
public class SelectInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The SelectInteraction is a fundamental component of the server-side Interaction System, acting as a dynamic, stateful query engine rather than a direct action. Its primary responsibility is to identify targets—entities or blocks—within a spatially and temporally defined area. It achieves this by executing a configurable Selector object over its prescribed duration (RunTime).

This class embodies the "fork-join" pattern in a non-blocking manner. For every unique target identified by the Selector, SelectInteraction forks a new, independent InteractionChain based on its configuration (HitEntity, HitBlock, HitEntityRules). It does not wait for these forked chains to complete. This allows a single causal event, such as a weapon swing or an area-of-effect spell, to trigger multiple, parallel consequences on various targets simultaneously.

Architecturally, SelectInteraction instances are data-driven objects, defined declaratively in asset files and instantiated at runtime by the engine's Codec system. This design separates the logic of *what* to look for and *how* to react from the core engine code, enabling designers to create complex behaviors without modifying the server binary.

A critical aspect of its design is the management of execution state. To remain stateless and reusable, the SelectInteraction object itself does not store transient data like which entities have already been hit. Instead, it leverages the DynamicMetaStore within the active InteractionContext, using static MetaKey constants (HIT_ENTITIES, HIT_BLOCKS) to track state for the duration of its execution.

## Lifecycle & Ownership
- **Creation:** Instances are deserialized and instantiated by the Hytale Codec system from game asset files. A SelectInteraction is created when the Interaction System processor encounters a corresponding definition within an active InteractionChain. **WARNING:** Direct instantiation via the `new` keyword is an anti-pattern and will result in a non-functional object.

- **Scope:** The lifetime of a SelectInteraction instance is scoped to its execution within a single step of an InteractionChain. It persists only for the duration defined by its RunTime property.

- **Destruction:** The object is eligible for garbage collection as soon as its execution completes (i.e., its runtime expires and its state is marked as Finished or Failed). All associated transient state stored in the InteractionContext's meta store is discarded when the context is destroyed or recycled.

## Internal State & Concurrency
- **State:** A SelectInteraction object is **immutable** after its creation from configuration. Its fields like selector and hitEntity represent its static definition. However, it orchestrates the management of **mutable, transient state** during its execution. This state, such as the set of entities already processed, is stored externally in the InteractionContext's DynamicMetaStore. This pattern ensures the configuration object itself remains pure and reusable.

- **Thread Safety:** This class is **not thread-safe**. The entire Interaction System operates within the main server world tick, which is a single-threaded environment. All method calls and state manipulations related to a SelectInteraction instance must occur on the server's main thread. Unsynchronized access from other threads will lead to severe concurrency issues, including collection modification exceptions and inconsistent game state.

## API Surface
The primary contract is fulfilled by its implementation of the Interaction interface, which is invoked by the Interaction System. Direct calls by user code are not supported.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | protected void | O(N) | The core execution method called by the Interaction System each tick. Complexity is proportional to the number of entities and blocks evaluated by the internal Selector. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns **Client**, indicating that this server-side interaction may depend on synchronized state from the client to resolve its execution, especially for player-initiated actions. |
| needsRemoteSync() | boolean | O(1) | Returns **true**, signifying that its configuration and state must be serialized and sent to the game client for prediction and presentation. |
| mapForkChain(...) | InteractionChain | O(F) | Attempts to map a client-predicted forked chain to a server-authoritative one, crucial for reconciling client-side effects. F is the number of existing forked chains. |

## Integration Patterns

### Standard Usage
SelectInteraction is not intended to be used directly in Java code. It is configured declaratively within game asset files (e.g., a weapon's definition file). The engine's Interaction System is responsible for loading and executing it.

The following example illustrates the *conceptual* flow of how the system processes this interaction, not how a developer would write it.

```java
// CONCEPTUAL: How the engine invokes the interaction
// This code resides deep within the server's InteractionChain processor.

InteractionContext context = ...; // The current interaction context
SelectInteraction interaction = ...; // The currently active interaction step

// The engine calls tick0 on every server update
interaction.tick0(isFirstRun, deltaTime, type, context, cooldownHandler);

// If the interaction finds a target, it internally calls context.fork(...)
// to create a new, parallel interaction chain for that target.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SelectInteraction()`. The object must be created via the `CODEC` system from an asset file to be properly configured. Manually created instances will be inert and cause NullPointerExceptions.
- **Stateful Modification:** Do not attempt to modify the fields of a SelectInteraction instance after it has been created. It is a configuration blueprint, not a stateful object.
- **External Ticking:** Do not call the `tick0` method from outside the server's core Interaction System loop. Doing so will break state management and cause unpredictable behavior.

## Data Pipeline
The flow of data and control during the execution of a SelectInteraction is a multi-stage process orchestrated by the server tick.

> Flow:
> InteractionChain Processor -> **SelectInteraction.tick0()** -> Selector.selectTargets() -> World Query -> Target Found -> **SelectInteraction forks new chain** -> New InteractionContext created -> New InteractionChain begins execution

1.  The server's `InteractionChain` processor executes the `SelectInteraction` node.
2.  On the first tick, `tick0` initializes a `Selector` instance and stores it in the `InteractionContext`'s meta store.
3.  On each tick, the `Selector` performs a spatial query against the `World` to find entities and blocks.
4.  For each potential target, it checks the `HIT_ENTITIES` or `HIT_BLOCKS` set within the context to ensure it has not been processed before.
5.  If the target is new, `SelectInteraction` duplicates the current `InteractionContext`, sets the new target within it, and calls `context.fork()`.
6.  This `fork` operation creates and schedules a new `InteractionChain` based on the `hitEntity` or `hitBlock` configuration, which begins executing on the next server tick.
7.  The original `SelectInteraction` continues this process until its `RunTime` expires, completely independent of the forked chains.

