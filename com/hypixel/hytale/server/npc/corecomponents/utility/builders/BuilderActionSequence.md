---
description: Architectural reference for BuilderActionSequence
---

# BuilderActionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSequence extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSequence is a configuration-time component that translates a JSON definition into a runtime ActionSequence object. It serves as a critical bridge between the declarative NPC behavior system, defined in asset files, and the server's imperative execution logic.

This class is not intended for direct use by developers. Instead, it is discovered and managed by a central BuilderManager during the server's asset loading phase. When the loader encounters a JSON object representing a sequence of actions, it instantiates this builder and uses it to parse the data.

The core responsibility of this class is to configure an ActionList, which represents the series of commands an NPC will execute. It also configures two critical execution modifiers:
-   **Blocking:** If true, the sequence will wait for each action to complete before starting the next. This enforces a strict, sequential execution order.
-   **Atomic:** If true, the entire sequence is treated as a single transaction. The system validates that *all* actions in the list can be executed before initiating the first one. If any action is invalid or its preconditions are not met, the entire sequence is aborted.

A key architectural feature is its use of the BuilderObjectReferenceHelper for its list of actions. This allows designers to define an action list inline within the JSON or, more powerfully, reference a globally defined and reusable ActionList. This promotes configuration reuse and reduces duplication.

## Lifecycle & Ownership
-   **Creation:** Instantiated reflectively by the BuilderManager when it parses an NPC configuration file and identifies a JSON object corresponding to this builder type.
-   **Scope:** Extremely short-lived. An instance of BuilderActionSequence exists only for the duration of parsing its specific JSON block and building the resulting ActionSequence object.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the `build` method is called and its product, the ActionSequence, is integrated into the parent NPC configuration. There is no manual cleanup.

## Internal State & Concurrency
-   **State:** Highly mutable. Its primary purpose is to accumulate state from a JSON object via the `readConfig` method. The fields `actions`, `blocking`, and `atomic` are all populated during this process.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used exclusively by the single thread responsible for loading a specific NPC asset. Concurrent calls to `readConfig` or `build` will result in a corrupted and unpredictable state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionSequence | O(N) | The terminal operation. Constructs the final, configured ActionSequence object. N is the number of sub-actions. |
| readConfig(JsonElement) | BuilderActionSequence | O(K) | Populates the builder's internal state from a JSON element. K is the number of keys in the JSON object. |
| validate(...) | boolean | O(N) | Recursively validates the configuration, including all sub-actions, to detect errors at load-time. N is the number of sub-actions. |
| getActionList(BuilderSupport) | ActionList | O(N) | Resolves the action list reference and applies the `atomic` and `blocking` flags. Used internally by the ActionSequence constructor. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's asset loading framework. A developer would never interact with it directly. The conceptual flow within the engine is as follows:

```java
// Conceptual engine code - DO NOT WRITE THIS
JsonElement config = parseFile("npc_behavior.json");
BuilderActionSequence builder = builderManager.createBuilderFor(config);

// The engine populates the builder from the JSON data
builder.readConfig(config);

// The engine validates the configuration before use
builder.validate(...);

// The final runtime object is created
ActionSequence runtimeSequence = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderActionSequence()`. The asset loading framework is responsible for its creation and lifecycle.
-   **State Re-assignment:** Do not manually modify the public fields of this class after `readConfig` has been called. Doing so creates a discrepancy between the source asset and the runtime object, making debugging impossible.
-   **Instance Re-use:** A builder instance is single-use. Do not attempt to call `readConfig` on the same instance with a different JSON object.

## Data Pipeline
The BuilderActionSequence is a specific step in the data transformation pipeline that turns static configuration files into live game objects.

> Flow:
> NPC Behavior JSON File -> Gson Parser -> BuilderManager -> **BuilderActionSequence.readConfig()** -> **BuilderActionSequence.build()** -> Runtime ActionSequence -> NPC Behavior Engine

---

