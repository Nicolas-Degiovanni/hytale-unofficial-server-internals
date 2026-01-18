---
description: Architectural reference for StateMappingHelper
---

# StateMappingHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient State Builder

## Definition
```java
// Signature
public class StateMappingHelper {
```

## Architecture & Concepts
The StateMappingHelper is a critical component within the server-side NPC asset compilation pipeline. It is not a runtime system. Its sole purpose is to act as a stateful builder during the parsing of an NPC's JSON configuration file.

This class serves as the authoritative translator between human-readable string identifiers for states (e.g., "attack", "idle.looking") and their optimized, integer-based representations required by the runtime state machine engine. It operates by building a bidirectional map of state names to integer indices.

Crucially, it also tracks the *usage context* of each state. It differentiates between:
*   **Sensors:** States that are defined or can be transitioned *from* (e.g., in a state evaluator).
*   **Setters:** States that are transitioned *to* (e.g., in an action or motion).
*   **Requirers:** States that are checked by a parameter but not necessarily part of the core state machine graph.

This contextual tracking enables a powerful validation step that catches logical errors in the state machine design at compile time, such as defining an action that transitions to a non-existent state.

The helper is designed to be driven by a recursive-descent parser, using its depth-tracking methods (`increaseDepth`, `decreaseDepth`) to manage nested state definitions within the asset file.

### Lifecycle & Ownership
The lifecycle of a StateMappingHelper is ephemeral and confined to the parsing of a single asset.

- **Creation:** A new instance is created by a high-level builder, such as an NpcAssetBuilder, at the beginning of processing a single NPC JSON definition. It is created with a clean slate for each new asset.
- **Scope:** The object lives only for the duration of the asset parsing process. It accumulates state mappings and usage data as the parser traverses the JSON tree.
- **Destruction:** After the asset file is fully parsed, the `validate` and `optimise` methods are called. The resulting integer maps are extracted and stored in the runtime asset object. The StateMappingHelper instance is then discarded and becomes eligible for garbage collection. **WARNING:** StateMappingHelper instances must NOT be reused across different asset builds; doing so will result in corrupted and merged state definitions.

## Internal State & Concurrency
- **State:** The StateMappingHelper is fundamentally a mutable object. Its primary function is to build internal collections, including maps from state names to integer indices and BitSets for usage tracking. The `optimise` method performs a final, destructive mutation to reduce memory footprint by trimming collections and nullifying temporary data structures.

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within an asset parsing job. Its internal data structures are not protected by locks or other concurrency primitives. Concurrent access from multiple threads will lead to race conditions, data corruption, and unpredictable state indices. Each parsing thread must maintain its own private instance of StateMappingHelper.

## API Surface
The public API is designed to be called by a driving parser system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAndPutSensorIndex(state, subState, setter) | void | O(1) | Registers a state as a "sensor". Creates a new index if the state is not yet known. |
| getAndPutSetterIndex(state, subState, setter) | void | O(1) | Registers a state as a "setter". Creates a new index if the state is not yet known. |
| validate(configName, errors) | void | O(N) | Performs logical validation on the collected states. Verifies that all setters have corresponding sensors. |
| optimise() | void | O(N) | Finalizes the internal maps. Trims collections to minimum size and nulls out temporary build data. |
| increaseDepth() / decreaseDepth() | void | O(1) | Manages the internal parsing depth counter, used for contextual state resolution. |
| getStateIndex(state) | int | O(1) | Retrieves the integer index for a given main state name. Returns MIN_VALUE if not found. |
| getSubStateIndex(index, subState) | int | O(1) | Retrieves the integer index for a given sub-state name within a main state. |

## Integration Patterns

### Standard Usage
The StateMappingHelper is intended to be used as a transient tool by a higher-level builder class responsible for parsing a JSON file. The typical sequence of operations is: create, populate, validate, optimize, discard.

```java
// In a hypothetical NpcAssetBuilder...
StateMappingHelper stateHelper = new StateMappingHelper();

// While parsing the JSON recursively...
// A state definition is encountered
stateHelper.getAndPutSensorIndex("attack", "swing", (main, sub) -> { /* ... */ });
// An action that triggers a state change is encountered
stateHelper.getAndPutSetterIndex("idle", null, (main, sub) -> { /* ... */ });

// After the entire file is parsed...
List<String> validationErrors = new ArrayList<>();
stateHelper.validate("my_npc_asset.json", validationErrors);
if (!validationErrors.isEmpty()) {
    throw new AssetParseException(validationErrors);
}

// Finalize the helper to prepare data for runtime
stateHelper.optimise();

// Extract the final data and discard the helper
int[] runtimeStates = stateHelper.getAllMainStates();
// ... store runtimeStates in the final asset object
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and re-use a StateMappingHelper instance for parsing multiple, distinct asset files. This will cause state definitions to leak between assets, leading to catastrophic runtime errors.
- **Concurrent Modification:** Do not share a single StateMappingHelper instance across multiple threads. This will corrupt its internal state.
- **Skipping Validation:** Failure to call the `validate` method before `optimise` can allow logically inconsistent state machines (e.g., transitions to undefined states) to be built, causing unpredictable runtime behavior.
- **Premature Optimization:** Do not call `optimise` before parsing is complete. This method is destructive and will prevent further states from being added correctly.

## Data Pipeline
The StateMappingHelper functions exclusively at asset compile-time. It is a key transformation step that converts declarative, string-based configuration into an efficient, integer-based format for the game server's runtime.

> Flow:
> NPC JSON File -> GSON Parser -> High-Level Asset Builder -> **StateMappingHelper** (Builds & Validates Mappings) -> Optimized Integer Arrays & Maps -> Runtime NPC Asset -> NPC State Machine Engine

