---
description: Architectural reference for BuilderStateTransitionEdges
---

# BuilderStateTransitionEdges

**Package:** com.hypixel.hytale.server.npc.statetransition.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderStateTransitionEdges extends BuilderBase<BuilderStateTransitionEdges.StateTransitionEdges> {
```

## Architecture & Concepts
The BuilderStateTransitionEdges class is a configuration parser responsible for deserializing NPC state machine transition rules from a JSON definition. It acts as a critical bridge between human-readable asset files and the server's optimized, integer-indexed state machine runtime.

Its primary architectural function is to translate an array of string-based state names (e.g., "Idle", "Patrolling") into arrays of integer indices. This conversion is fundamental for performance, as the runtime state machine can perform fast integer comparisons instead of expensive string operations.

This translation is achieved via a deferred resolution mechanism. During the `readConfig` phase, the builder does not immediately know the integer index for a given state name. Instead, it registers a dependency with the asset loading system using `registerStateRequirer`. The asset system later resolves all state names to indices and invokes a callback to populate the `fromStateIndices` and `toStateIndices` fields. This decouples the builder from the global state registry, allowing for parallel and out-of-order asset loading.

The final product of this builder is an immutable `StateTransitionEdges` object, which serves as a lightweight data container for the resolved transition rules.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderStateTransitionEdges is created by the server's asset loading framework for each "edges" JSON object it encounters within an NPC's state machine definition. It is not a singleton; a new builder is instantiated for each configuration block.
-   **Scope:** The builder object is short-lived and exists only for the duration of the asset parsing and linking phase. Its purpose is complete once the `build` method has been called and the resulting `StateTransitionEdges` object has been retrieved.
-   **Destruction:** The builder instance becomes eligible for garbage collection as soon as the asset loader stores the final `StateTransitionEdges` product. The immutable `StateTransitionEdges` object, however, persists for the lifetime of the parent NPC asset.

## Internal State & Concurrency
-   **State:** The builder's internal state is highly mutable during configuration parsing. It temporarily stores state names as strings (`fromStates`, `toStates`) before they are resolved into integer arrays and nulled out. It also caches the final built product in `builtStateTransitionEdges` to ensure that subsequent calls to `build` are idempotent and do not trigger redundant work.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be confined to a single thread within the asset loading pipeline. All state mutations occur within the `readConfig` method. Concurrent calls to `readConfig` or modification of its internal state from multiple threads will result in a corrupted object and undefined behavior. The final `StateTransitionEdges` object it produces is, however, fully immutable and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder | O(N\*M) | Parses the JSON definition. Populates internal fields and registers state name dependencies for later resolution. |
| build(BuilderSupport support) | StateTransitionEdges | O(1) | Constructs and returns the immutable StateTransitionEdges object. Subsequent calls return a cached instance. |
| isEnabled(ExecutionContext context) | boolean | O(1) | Evaluates at runtime whether this set of transitions is currently active based on the provided context. |

## Integration Patterns

### Standard Usage
This class is intended for internal use by the asset loading system. A developer would define the transitions in a JSON file, and the engine would use this builder under the hood to process it.

```java
// Engine-level code (conceptual)
BuilderStateTransitionEdges builder = new BuilderStateTransitionEdges();

// The asset loader provides the relevant JSON snippet
builder.readConfig(npcJson.get("stateTransitions").get("edges").get(0));

// The asset loader later triggers the build phase after all dependencies are resolved
StateTransitionEdges transitionRules = builder.build(assetBuilderSupport);

// The resulting object is stored in the final NPC runtime asset
npcStateMachine.addTransitionRules(transitionRules);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class in game logic. It is exclusively part of the asset initialization pipeline.
-   **State Re-use:** Do not call `readConfig` on an instance that has already been used. Each builder is designed for a single, one-shot parsing operation.
-   **Pre-emptive Access:** Do not attempt to read the `fromStateIndices` or `toStateIndices` fields directly. These fields are populated asynchronously by the asset system and are not guaranteed to be valid until after the entire asset linking phase is complete.

## Data Pipeline
The flow of data through this component is linear, transforming a declarative configuration into an optimized runtime object.

> Flow:
> NPC JSON File -> GSON Parser -> **BuilderStateTransitionEdges.readConfig()** -> Deferred State Index Resolution -> **BuilderStateTransitionEdges.build()** -> StateTransitionEdges (Immutable Object) -> NPC State Machine Runtime

