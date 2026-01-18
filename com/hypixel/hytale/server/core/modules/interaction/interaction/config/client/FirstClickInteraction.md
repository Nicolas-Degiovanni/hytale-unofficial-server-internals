---
description: Architectural reference for FirstClickInteraction
---

# FirstClickInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Object

## Definition
```java
// Signature
public class FirstClickInteraction extends Interaction {
```

## Architecture & Concepts
The FirstClickInteraction class is a conditional branching node within the server-side Interaction system. Its primary function is to execute one of two different subsequent interaction chains based on client-side input context: a brief *click* versus a sustained *key hold*.

Architecturally, it acts as a server-side state machine gate that is controlled by client-provided data. It explicitly declares its dependency on the client via the `getWaitForDataFrom` method, which returns `WaitForDataFrom.Client`. This forces the server's InteractionManager to pause execution of this interaction chain until it receives an `InteractionSyncData` packet from the corresponding client, which contains the result of the client-side input evaluation.

The core of its design is the `compile` method. This method translates the declarative configuration (the `click` and `held` interaction asset strings) into a low-level, imperative sequence of operations within an `OperationsBuilder`. This pre-compilation step builds a jump table, allowing the interaction engine to efficiently branch to the correct subsequent interaction at runtime without needing to re-evaluate the logic.

## Lifecycle & Ownership
-   **Creation:** Instances of FirstClickInteraction are not created directly using the `new` keyword. They are instantiated by the engine's asset loading system via the static `CODEC` field. This process involves deserializing data from configuration files (e.g., JSON definitions for items or abilities).
-   **Scope:** Once loaded from an asset file, an instance of FirstClickInteraction is typically cached in a central registry managed by the `InteractionManager`. It persists for the entire server session as an immutable definition. The runtime state for a specific entity's use of this interaction is stored in a transient `InteractionContext` object.
-   **Destruction:** The object is managed by the asset system's lifecycle. It is eligible for garbage collection only when the server shuts down or performs a full asset reload that removes all references to it.

## Internal State & Concurrency
-   **State:** The object's state consists of the `click` and `held` fields, which are string references to other Interaction assets. After deserialization, this state is **effectively immutable**. The class itself is stateless regarding runtime execution; all execution-specific data is passed in via the `InteractionContext` parameter.
-   **Thread Safety:** The object is inherently thread-safe for read operations due to its immutable state post-creation. All methods that perform logic (`tick0`, `compile`) operate on a non-shared `InteractionContext` or `OperationsBuilder` instance. The responsibility for ensuring that a single interaction execution is handled on a single thread lies with the calling `InteractionManager`, not this class.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compile(OperationsBuilder builder) | void | O(N) | Translates the `click` and `held` paths into a sequence of low-level operations and jump labels. Complexity is recursive, proportional to the size of child interaction trees. |
| tick0(...) | void | O(1) | Executes one tick of server-side logic. Checks the client-provided state in the `InteractionContext` to determine if the interaction should finish (proceed to `click`) or fail (proceed to `held`). |
| simulateTick0(...) | void | O(1) | Executes one tick of client-side prediction logic. Determines if the interaction is still in a "charging" or "held" state. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.Client`, signaling to the engine that this interaction cannot proceed without data from the client. |
| walk(Collector collector, ...) | boolean | O(N) | Recursively traverses the child interaction trees (`click` and `held`) to gather data, used for validation or asset dependency analysis. |
| needsRemoteSync() | boolean | O(1) | Returns `true`, indicating that the state and definition of this interaction must be synchronized with the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, imperative use by developers. It is configured declaratively within game asset files. The `InteractionManager` is responsible for loading, compiling, and executing it.

The primary integration point is the `compile` method, which is invoked by the system to build an executable operation sequence.

```java
// Conceptual example of how the system uses the compiled operations.
// A developer would NOT write this code.

// 1. During asset loading, the interaction is compiled.
OperationsBuilder builder = new OperationsBuilder();
FirstClickInteraction interactionDef = Interaction.getInteractionOrUnknown("my_weapon_primary_fire");
interactionDef.compile(builder);
List<Operation> compiledSequence = builder.build();

// 2. During gameplay, the InteractionManager executes the sequence.
InteractionContext context = ...; // Create context for a player
// The manager runs the compiled operations, which will internally call
// tick0 on the FirstClickInteraction instance and then jump based on the result.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new FirstClickInteraction()`. The object will be unconfigured and will cause NullPointerExceptions. It must be loaded from assets via the `CODEC`.
-   **State Mutation:** Do not attempt to modify the `click` or `held` fields at runtime. These are intended to be immutable definitions. Modifying them would affect all subsequent uses of this interaction globally and is not thread-safe.
-   **Ignoring Client State:** Calling `tick0` with an `InteractionContext` that does not contain valid `InteractionSyncData` from the client will lead to an assertion error and incorrect behavior. The `InteractionManager` is responsible for enforcing this data dependency.

## Data Pipeline
The flow of data and control for this component is a round-trip between the client and server, orchestrated by the server's interaction engine.

> Flow:
> Client Input (Key Press) -> Client-side Simulation (`simulateTick0`) -> Network Packet (`InteractionSyncData`) -> Server `InteractionManager` -> **`FirstClickInteraction.tick0`** -> Reads `InteractionState` from Packet -> Sets Context to `Finished` or `Failed` -> `OperationsBuilder` directs execution to `click` or `held` interaction chain.

