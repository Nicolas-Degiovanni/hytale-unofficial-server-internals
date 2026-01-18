---
description: Architectural reference for BuilderActionRandom
---

# BuilderActionRandom

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionRandom extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionRandom class is a key component in the server-side NPC asset deserialization pipeline. It functions as a specialized factory responsible for translating a declarative JSON configuration into a runtime game object, specifically the ActionRandom.

Its role is to act as an intermediate representation during asset loading. When the server parses an NPC definition file, it encounters various "action" blocks. If an action is of type "Random", the asset loading system instantiates a BuilderActionRandom and uses its `readConfig` method to populate it with the data from the JSON.

This class encapsulates the logic for parsing, validating, and ultimately constructing a list of weighted actions. It leverages the `BuilderObjectListHelper` utility to manage the nested complexity of parsing a list of `WeightedAction` objects, each of which may have its own complex configuration. This pattern isolates the parsing logic for a specific game concept from the core asset loading machinery, promoting modularity.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the central `BuilderManager` service during the server's asset loading phase. This occurs when the manager encounters a JSON object with a "type" field corresponding to this builder.
-   **Scope:** Ephemeral. A BuilderActionRandom instance exists only for the duration of parsing a single JSON block. It is a short-lived, single-use object.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the `build` method has been called and the resulting `ActionRandom` object has been integrated into the parent NPC asset. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** Highly **Mutable**. The primary state is the `actions` field, which is a `BuilderObjectListHelper`. The `readConfig` method directly mutates this field, populating it with data deserialized from the input JSON. The state represents a pre-validated, pre-compiled blueprint of the final runtime object.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be used exclusively within the server's single-threaded asset loading pipeline. Concurrent calls to `readConfig` or `build` would lead to race conditions and unpredictable state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionRandom | O(N) | Constructs the final, immutable `ActionRandom` runtime object from the parsed state. N is the number of actions. |
| readConfig(JsonElement) | BuilderActionRandom | O(N) | Populates the builder's internal state from a JSON array. This is the primary entry point for configuration. |
| validate(...) | boolean | O(N) | Recursively validates the configuration, ensuring all child actions and weights are valid. Critical for preventing runtime errors. |
| getActions(BuilderSupport) | List<WeightedAction> | O(N) | Builds and returns only the list of `WeightedAction` runtime objects. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the asset loading system. The conceptual flow within the engine is as follows.

```java
// Conceptual engine code
// The BuilderManager finds the right builder for a given JSON object
BuilderActionBase builder = builderManager.getBuilderFor(jsonObject);

if (builder instanceof BuilderActionRandom) {
    // 1. Configure the builder from the JSON data
    ((BuilderActionRandom) builder).readConfig(jsonObject);

    // 2. Validate the configuration against game rules
    boolean isValid = builder.validate(assetName, validationHelper, context, scope, errors);
    if (!isValid) {
        throw new AssetLoadException("Validation failed for ActionRandom");
    }

    // 3. Build the final runtime object
    ActionRandom runtimeAction = ((BuilderActionRandom) builder).build(builderSupport);
    
    // The runtimeAction is now ready to be used by the NPC's behavior system.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BuilderActionRandom()`. The lifecycle of this object is managed exclusively by the server's `BuilderManager` to ensure proper dependency injection and context.
-   **State Reuse:** Do not attempt to reuse a builder instance. It is designed as a one-shot factory. Calling `readConfig` multiple times on the same instance will result in corrupt or unpredictable state.
-   **Skipping Validation:** The `validate` method is a critical step that prevents malformed or invalid NPC assets from loading, which could cause server instability. Bypassing this check is highly discouraged.

## Data Pipeline
The BuilderActionRandom serves as a transformation stage in the NPC asset data pipeline, converting structured text data into an executable game object.

> Flow:
> NPC Asset JSON File -> Server Asset Loader -> `BuilderManager` -> **`BuilderActionRandom.readConfig()`** -> Internal State (Validated) -> **`BuilderActionRandom.build()`** -> `ActionRandom` (Runtime Object) -> NPC Behavior Engine

