---
description: Architectural reference for BuilderWeightedAction
---

# BuilderWeightedAction

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderWeightedAction extends BuilderBase<WeightedAction> {
```

## Architecture & Concepts
The BuilderWeightedAction class is a configuration-time factory within the server's NPC asset pipeline. It is not a runtime component. Its sole responsibility is to parse a specific JSON structure and act as a blueprint for creating a runtime WeightedAction object.

This class embodies a deferred construction pattern. Upon parsing a configuration file, it does not immediately instantiate the underlying Action object. Instead, it holds a reference via a BuilderObjectReferenceHelper. This architectural choice is critical for the asset system, as it allows for complex, forward-referenced, and modular NPC behavior definitions. The complete object graph is resolved only during the final build phase, which enables robust, load-time validation and dependency resolution before any runtime objects are created.

This builder is a foundational piece for any system that requires probabilistic selection of NPC behaviors, such as a random action selector.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderWeightedAction is created by the central BuilderManager service during the server's asset loading phase. Each unique weighted action definition within an NPC's JSON configuration file results in a new, distinct instance of this class.

-   **Scope:** The object is short-lived. Its scope is strictly confined to the asset loading and validation process. It exists only as an intermediate representation between the raw JSON configuration and the final, usable runtime object.

-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection immediately after its build method is invoked and the resulting WeightedAction object is integrated into its parent component (e.g., a list of actions). It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable during the configuration parsing phase, where the readConfig method populates its internal holders for the action reference and weight. After parsing is complete, its state should be considered effectively immutable. It serves as a read-only blueprint for the final build call.

-   **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed exclusively for use within the server's single-threaded asset loading pipeline. Any attempt to access or modify an instance from multiple threads will result in unpredictable behavior, race conditions, and a corrupted object state.

## API Surface
The public API is designed for consumption by the asset loading framework, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | WeightedAction | O(N) | Constructs the final runtime WeightedAction. Complexity is dependent on the referenced Action's build process. |
| readConfig(JsonElement) | Builder<WeightedAction> | O(1) | Populates the builder's internal state from a JSON object. Throws exceptions on malformed data. |
| validate(...) | boolean | O(N) | Performs load-time validation of the configuration. Delegates validation to the referenced Action builder. |
| getAction(BuilderSupport) | Action | O(N) | Resolves the object reference and builds the underlying Action. This is a potentially expensive operation. |
| getWeight(BuilderSupport) | double | O(1) | Resolves and returns the configured weight value. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its usage is managed entirely by the NPC asset loading system. A game designer defines the component in a JSON file, and the framework handles the rest.

*Example NPC JSON configuration snippet:*
```json
{
  "type": "RandomActionList",
  "actions": [
    {
      "type": "WeightedAction",
      "Action": {
        "type": "PlayAnimation",
        "animation": "idle_variant_1"
      },
      "Weight": 50.0
    },
    {
      "type": "WeightedAction",
      "Action": {
        "type": "PlaySound",
        "sound": "npc.grunt"
      },
      "Weight": 25.0
    }
  ]
}
```
The framework uses BuilderWeightedAction to parse each element in the actions array.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BuilderWeightedAction()`. The class is not designed for manual construction and requires initialization by the BuilderManager to function correctly.

-   **State Mutation After Read:** Do not attempt to modify the builder's state after the readConfig method has been called. The object's lifecycle is one-shot: parse, validate, build.

-   **Premature Build:** Calling build before readConfig has been successfully executed by the framework will result in validation errors or a NullPointerException.

## Data Pipeline
The BuilderWeightedAction serves as a critical transformation step in the data flow from configuration files to live game objects.

> Flow:
> NPC Behavior JSON File -> Gson Parser -> BuilderManager -> **BuilderWeightedAction.readConfig()** -> Validation Phase -> **BuilderWeightedAction.build()** -> WeightedAction Runtime Object -> NPC Behavior Tree

