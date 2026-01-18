---
description: Architectural reference for BuilderStateTransitionController
---

# BuilderStateTransitionController

**Package:** com.hypixel.hytale.server.npc.statetransition.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderStateTransitionController extends BuilderBase<StateTransitionController> {
```

## Architecture & Concepts
The BuilderStateTransitionController is a configuration parser and factory component within the server's NPC asset loading pipeline. Its sole responsibility is to interpret a specific JSON array structure from an NPC definition file and construct a fully configured, runtime-ready StateTransitionController object.

This class embodies the Builder pattern. It decouples the complex, multi-step process of parsing and validating state transition rules from the final, immutable representation of the StateTransitionController used by the NPC's AI at runtime.

Architecturally, it acts as a deserializer that is highly specialized for NPC state transitions. It does not contain any game logic itself; rather, it builds the object that *does*. It achieves this by composing a BuilderObjectListHelper, which in turn delegates the parsing of individual transition entries to the BuilderStateTransition class. This creates a hierarchical and maintainable parsing structure.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's core asset loading system, specifically the BuilderManager, when it encounters a JSON block designated for state transitions. It is never created directly by game logic code.

- **Scope:** The lifecycle of a BuilderStateTransitionController instance is extremely short and confined to the asset parsing phase. It exists only for the duration required to process one specific JSON array.

- **Destruction:** The object is eligible for garbage collection immediately after the `build` method has been called and the resulting StateTransitionController has been returned to the asset loader. It holds no persistent references and is not intended to be reused.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The primary state is held within the `stateTransitionEntries` field, which is populated incrementally as the `readConfig` method processes the input JSON. The object is stateful by design, accumulating configuration before the final build step.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to operate exclusively within the single-threaded context of the asset loading pipeline. Concurrent calls to `readConfig` or `build` would result in a corrupted object and unpredictable server behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | StateTransitionController | O(N) | Finalizes the construction process. Consumes the internal state to create and return a new StateTransitionController instance. N is the number of transitions. |
| readConfig(JsonElement) | Builder | O(N) | The primary entry point for parsing. Populates the builder's internal state from the provided JSON data. Throws exceptions on malformed input. |
| validate(...) | boolean | O(N) | A critical step in the asset pipeline. Recursively validates all parsed transition entries to ensure they are logically sound before the build phase. |
| getStateTransitionEntries(BuilderSupport) | List | O(N) | Builds and returns only the list of configured state transitions. This is primarily an internal helper for the StateTransitionController constructor. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay systems. It is invoked exclusively by the engine's asset loading framework. The conceptual flow is as follows:

```java
// Conceptual engine code for loading an NPC asset
BuilderStateTransitionController builder = builderManager.createBuilder(BuilderStateTransitionController.class);

// Engine feeds the relevant JSON section to the builder
builder.readConfig(npcJson.get("stateTransitions"));

// Engine validates the parsed data
boolean isValid = builder.validate(assetName, validationHelper, context, scope, errorList);
if (!isValid) {
    throw new AssetLoadException("Invalid state transitions for " + assetName);
}

// Engine builds the final runtime object
StateTransitionController controller = builder.build(builderSupport);
npc.setStateController(controller);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderStateTransitionController()`. The asset system's BuilderManager is responsible for its creation and lifecycle. Direct instantiation bypasses critical engine integration.

- **State Re-use:** Do not attempt to reuse a builder instance to parse a second JSON configuration. The internal state is not cleared between calls and will result in a corrupted, merged configuration. Each builder instance is single-use.

- **Calling Build Before Read:** Invoking `build` before `readConfig` has been successfully called will produce an empty or incomplete StateTransitionController, leading to runtime errors or inert NPCs.

## Data Pipeline
The flow of data through this component is linear and unidirectional, transforming declarative configuration into an executable object.

> Flow:
> NPC JSON File -> Asset Loader -> **BuilderStateTransitionController.readConfig()** -> Internal List of Builders -> **BuilderStateTransitionController.build()** -> StateTransitionController (Runtime Object) -> NPC Behavior System

