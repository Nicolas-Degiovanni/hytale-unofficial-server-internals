---
description: Architectural reference for ParallelInteraction
---

# ParallelInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Object

## Definition
```java
// Signature
public class ParallelInteraction extends Interaction {
```

## Architecture & Concepts

The ParallelInteraction is a control-flow node within the server-side Interaction System. Its primary function is to fork an entity's interaction state, enabling multiple interaction chains to execute concurrently. It acts as a "fire-and-forget" dispatcher.

When the InteractionManager processes a ParallelInteraction node, it performs two distinct operations:
1.  **Continuation:** The first interaction defined in its list is executed within the *current* InteractionContext, effectively continuing the existing chain.
2.  **Forking:** All subsequent interactions in its list are initiated in new, independent InteractionContexts. These forked contexts are duplicates of the parent context at the moment of forking but evolve independently thereafter.

This mechanism is critical for creating complex behaviors where an entity needs to perform a primary action while simultaneously triggering secondary, non-blocking effects like playing an animation, starting a sound, or beginning a background cooldown timer.

This class is not intended for direct instantiation in gameplay code. It is defined declaratively in game asset files and deserialized at server startup via its associated codec.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the Hytale `BuilderCodec` system during the asset loading phase. It is deserialized from a corresponding definition in a game asset file, typically JSON.
-   **Scope:** Application-scoped and immutable. Once loaded from assets, an instance of ParallelInteraction persists for the entire server lifetime. It represents a static definition of a behavior, not a runtime state.
-   **Destruction:** Deallocated when the server shuts down and all game assets are unloaded from memory.

## Internal State & Concurrency
-   **State:** Immutable. The core state is the `interactions` array, which contains string identifiers for other RootInteractions. This array is populated once during deserialization and is never modified at runtime.
-   **Thread Safety:** The ParallelInteraction object itself is inherently thread-safe due to its immutability. It can be safely referenced and read from any thread.

    **Warning:** While the object is thread-safe, its primary execution method, `tick0`, operates on a mutable InteractionContext. The Interaction System guarantees that `tick0` is called from a context-safe thread, typically the main server tick loop for a given entity. All forked interactions are also scheduled and managed by this system, preventing direct concurrency violations on the entity's state.

## API Surface
The public contract is primarily consumed by the internal InteractionManager, not by gameplay developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(N) | Executes the parallel logic. Continues the first interaction and forks N-1 new interaction chains. Marks the current node as Finished. |
| simulateTick0(...) | void | O(1) | A simplified execution path for simulation purposes. Only continues the first interaction; does not fork. |
| walk(...) | boolean | O(Graph) | Traverses the interaction graph starting from this node. Used by engine tools for validation and data collection. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `None`, indicating this interaction does not pause execution to wait for client or external data. |

## Integration Patterns

### Standard Usage
ParallelInteraction is not used procedurally in code. It is defined declaratively within a RootInteraction asset file. The engine's InteractionManager is responsible for interpreting this data.

A typical asset definition would look like this:

```json
// Example in a hypothetical interaction asset file
{
  "id": "my_parallel_action",
  "type": "ParallelInteraction",
  "interactions": [
    "continue_main_quest_line", // This chain continues in the current context
    "play_success_sound",       // This chain is forked
    "trigger_celebration_vfx"   // This chain is also forked
  ]
}
```

The server engine processes this definition, and when an entity's interaction state reaches `my_parallel_action`, the `tick0` method is invoked automatically.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ParallelInteraction()`. The object is not designed for manual construction. Its internal state, particularly the resolution of interaction string names to asset IDs, is handled by the `CODEC` during asset loading. Manual creation will result in a non-functional object.
-   **Invalid Configuration:** The `CODEC` enforces a minimum of two interactions. Defining a ParallelInteraction with one or zero interactions will cause an asset validation error during server startup. The purpose of the node is to create at least one parallel fork.
-   **Stateful Logic:** Do not attempt to extend this class to add mutable runtime state. The instances are shared across all entities and are expected to be stateless definitions. Runtime state belongs in the InteractionContext or an entity's components.

## Data Pipeline
The primary flow for this component is one of control, not data transformation. It directs the flow of execution within the Interaction System.

> Flow:
> Game Asset (JSON) -> Server Asset Loader -> **ParallelInteraction.CODEC** -> In-Memory ParallelInteraction Object -> InteractionManager (evaluating an entity) -> `tick0` -> `InteractionContext.execute()` (Continuation) + `InteractionContext.fork()` (New Chains)

