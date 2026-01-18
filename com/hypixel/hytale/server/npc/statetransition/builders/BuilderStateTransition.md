---
description: Architectural reference for BuilderStateTransition
---

# BuilderStateTransition

**Package:** com.hypixel.hytale.server.npc.statetransition.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderStateTransition extends BuilderBase<BuilderStateTransition.StateTransition> {
```

## Architecture & Concepts
The BuilderStateTransition class is a configuration-to-object transformer within the server's NPC asset loading pipeline. Its primary responsibility is to parse a specific `JsonElement` from an NPC's definition file—the section defining state transitions—and construct a valid, runtime-usable `StateTransition` object.

This builder acts as the authoritative parser for the logic that connects different NPC states (e.g., *Idle* to *Attacking*). It defines not just the valid pathways between states but also the sequence of `Actions` that must execute when a transition occurs.

Architecturally, it is a leaf component in a larger "Builder" design pattern orchestrated by a central `BuilderManager`. It relies on specialized helper classes like `BuilderObjectStaticListHelper` and `BuilderObjectReferenceHelper` to recursively parse nested JSON structures for state edges and action lists, respectively. This composition allows the core builder to focus solely on the validation and assembly logic specific to state transitions.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly by developers. The server's `BuilderManager` service instantiates this class, likely via reflection, when it encounters the corresponding key (e.g., "stateTransitions") during the deserialization of an NPC asset file.
-   **Scope:** The lifecycle of a BuilderStateTransition instance is extremely short and confined to the parsing of a single NPC configuration. It is created, used to process one JSON block, and then becomes eligible for garbage collection.
-   **Destruction:** Managed by the Java garbage collector. There is no manual cleanup required. The resulting `StateTransition` object, however, is retained as part of the compiled NPC definition for the server's lifetime.

## Internal State & Concurrency
-   **State:** Highly mutable. The instance's fields (`stateTransitionEdges`, `actions`, `enabled`) are populated by the `readConfig` method. This internal state is transient and serves as a temporary container for data between parsing, validation, and the final build step.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be used in a single-threaded context during the asset loading phase. Concurrent calls to `readConfig` or `build` on the same instance will lead to corrupt state and unpredictable behavior. Asset loading operations must be synchronized externally.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder | O(N) | Parses the input JSON, populating internal fields. N is the number of tokens in the JSON structure. |
| validate(...) | boolean | O(S^2) | Performs deep validation, most notably checking for duplicate state transition edges. Complexity is roughly quadratic to the number of states in the worst case. |
| build(BuilderSupport support) | StateTransition | O(1) | Constructs the final, immutable StateTransition object from the parsed and validated internal state. |
| getActionList(BuilderSupport support) | ActionList | O(1) | Builds the associated ActionList. **Warning:** This method has a side effect of modifying the resulting ActionList by setting its atomic and blocking flags. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is invoked automatically by the engine's asset loading systems. The conceptual flow within the `BuilderManager` is as follows.

```java
// Conceptual example of engine-level usage
JsonElement configSection = npcJson.get("stateTransitions");
BuilderStateTransition builder = builderManager.getBuilderFor(BuilderStateTransition.class);

// The engine populates, validates, and builds the final object
builder.readConfig(configSection);
builder.validate(npcName, validationHelper, context, scope, errors);
StateTransition runtimeObject = builder.build(builderSupport);

// The runtimeObject is then registered with the StateTransitionController
stateTransitionController.add(runtimeObject);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderStateTransition()`. The `BuilderManager` is responsible for its creation and dependency injection (e.g., `stateHelper`, `builderManager` fields from `BuilderBase`).
-   **Instance Re-use:** Do not re-use a builder instance to parse a second configuration. Its internal state is not reset between calls and will result in corrupted, merged configuration data.
-   **Out-of-Order Calls:** The operational sequence must be `readConfig`, then `validate`, then `build`. Calling `build` before `readConfig` will produce an empty or invalid `StateTransition` object, leading to runtime `NullPointerException`s.

## Data Pipeline
The flow of configuration data is strictly linear, moving from raw text to a structured runtime object.

> Flow:
> NPC JSON File -> GSON Parser -> `JsonElement` -> `BuilderManager` -> **BuilderStateTransition.readConfig()** -> **BuilderStateTransition.validate()** -> **BuilderStateTransition.build()** -> `StateTransition` Object -> `StateTransitionController`

