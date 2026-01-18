---
description: Architectural reference for SimpleInteraction
---

# SimpleInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Component / Configuration Object

## Definition
```java
// Signature
public class SimpleInteraction extends Interaction {
```

## Architecture & Concepts

The SimpleInteraction class is a fundamental control-flow node within the server-side Interaction System. It does not represent a tangible action like attacking or using an item, but rather serves as a logical junction or router within a sequence of interactions. Its primary role is to direct the execution flow based on the success or failure of preceding operations, functioning as a conditional branch in a state machine.

Interactions are defined declaratively in game asset files and are deserialized at runtime into a graph of Interaction objects. SimpleInteraction is one of the most common nodes in this graph.

The core architectural pattern here is the **Compiler Pattern**. The `compile` method transforms the declarative, graph-like structure of an interaction sequence (linked by *next* and *failed* properties) into a linear, imperative list of operations. This "bytecode" is then executed by the InteractionManager. SimpleInteraction's contribution to this compiled list is to insert itself as an operation and manage the jump labels for its success and failure branches.

This class is intentionally minimal; it performs no game logic itself. Its purpose is to connect other, more complex Interaction nodes, introduce delays, or trigger animations between state transitions without adding computational overhead.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the `new` keyword. They are deserialized from game asset files (e.g., JSON) by the Hytale asset loading pipeline using the provided `BuilderCodec`. Each instance represents a specific interaction defined in the game's data.
-   **Scope:** An instance is effectively a stateless template. It is loaded once when the server starts and persists in memory for the entire server session. It is shared and reused for all players executing that specific interaction.
-   **Destruction:** Instances are garbage collected when the server shuts down or during a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** **Immutable**. After being deserialized from an asset file, its internal fields, such as *next* and *failed*, are never modified. All runtime state (e.g., the current player's progress, success/failure status) is stored externally in the `InteractionContext` object passed into its methods.
-   **Thread Safety:** **Inherently thread-safe**. As a stateless configuration object, its methods can be safely called from multiple threads simultaneously. It does not contain any locks or synchronization primitives. The responsibility for thread-safe manipulation of the `InteractionContext` lies with the calling system, typically the per-entity game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compile(OperationsBuilder builder) | void | O(N) | **CRITICAL**. Recursively traverses the interaction graph from this node, adding operations and jump labels for the *next* and *failed* branches to the builder. N is the number of nodes in the subgraph. |
| tick0(...) | void | O(1) | Executes the runtime logic for this node. Checks the context for a failure state and instructs the context to jump to the failure label if necessary. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.None`, indicating this interaction does not pause execution to wait for client or server data. |
| walk(Collector collector, ...) | boolean | O(N) | Traverses the interaction graph to gather data, typically for validation or debugging purposes. N is the number of nodes in the subgraph. |
| needsRemoteSync() | boolean | O(N) | Determines if this interaction or any of its children need to be synchronized with the client. |

## Integration Patterns

### Standard Usage

A developer does not interact with this class directly in Java code. Instead, it is defined within a data file as part of a larger interaction graph. The system then loads and compiles this definition.

A conceptual asset definition might look like this:

```json
// in some_interaction_sequence.json
{
  "id": "my_simple_branch",
  "type": "SimpleInteraction",
  "next": "asset:interaction.success_path",
  "failed": "asset:interaction.failure_path"
}
```

The system uses this definition to build a state machine. The `SimpleInteraction` node acts as the branch point.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SimpleInteraction()`. The object will be unconfigured and will cause NullPointerExceptions when used by the InteractionManager. Interactions must always be defined as assets and loaded by the engine.
-   **Stateful Implementation:** Do not extend this class to add mutable, per-player state. All runtime data must be stored in the `InteractionContext`. This class must remain a stateless template.
-   **Blocking Operations:** The `tick0` method must remain non-blocking and execute instantly. Adding long-running or blocking logic will stall the server's interaction processing tick.

## Data Pipeline

The SimpleInteraction class participates in two distinct data pipelines: a one-time compilation pipeline and a recurring runtime execution pipeline.

> **Compilation Pipeline (Server Start):**
> Game Asset File (JSON) -> Asset Loader -> **SimpleInteraction.CODEC** -> In-Memory Graph of Interaction Objects -> `InteractionManager.compile()` -> Finalized, Linear Operation List

> **Runtime Pipeline (Per-Player Action):**
> Player Action Trigger -> InteractionManager Execution -> `SimpleInteraction.tick0()` is called -> `InteractionContext` state is read -> Execution pointer jumps to `next` or `failed` label in the Operation List

