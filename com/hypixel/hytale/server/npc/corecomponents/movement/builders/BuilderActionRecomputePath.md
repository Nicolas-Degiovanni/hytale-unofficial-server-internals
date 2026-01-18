---
description: Architectural reference for BuilderActionRecomputePath
---

# BuilderActionRecomputePath

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionRecomputePath extends BuilderActionBase implements Builder<Action> {
```

## Architecture & Concepts
The BuilderActionRecomputePath class is a factory component within the server's NPC Behavior Asset Pipeline. Its sole responsibility is to translate a declarative configuration entry from an NPC behavior asset file into a concrete, executable `ActionRecomputePath` object.

This class is part of a broader architectural pattern where NPC behaviors are defined in external JSON files rather than being hard-coded. The server's asset loading system dynamically discovers and instantiates registered Builder classes like this one based on type identifiers in the JSON data. This decouples the game logic for NPC actions from the data that defines when and how those actions are used, enabling designers to create complex behaviors without modifying server code.

In essence, this builder acts as a deserialization endpoint, transforming a data-driven instruction into a live object that the NPC's internal state machine or behavior tree can execute.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's asset loading framework (e.g., an `NPCAssetManager`) when it parses an NPC behavior JSON file containing a corresponding action type. It is never created manually by game logic code.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of the asset parsing for a single action definition. Once its `build` method is called and the resulting `ActionRecomputePath` is returned, the builder instance is no longer referenced by the asset loader and becomes eligible for garbage collection.
- **Destruction:** Handled by the Java Garbage Collector. There are no explicit cleanup or `close` methods, as the object is stateless.

## Internal State & Concurrency
- **State:** Stateless and effectively immutable. The `readConfig` method is a no-op, indicating that this specific action requires no external parameters from the JSON configuration. The builder itself does not cache data or maintain any state between method calls.
- **Thread Safety:** Inherently thread-safe. Due to its stateless nature, a single instance could be safely used across multiple threads. However, the asset pipeline's typical operational model is to create a new builder instance for each asset it processes, making concurrent access to a shared instance a non-issue in practice.

## API Surface
The public API is designed for consumption by the automated asset pipeline, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionRecomputePath | O(1) | Constructs and returns a new `ActionRecomputePath` instance. This is the primary factory method. |
| readConfig(JsonElement) | BuilderActionRecomputePath | O(1) | Fluent method for configuration. For this class, it performs no operation and simply returns itself. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns an enum indicating the stability of this builder's API, used by asset validation tools. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is invoked transparently by the server's asset loading systems. The conceptual flow within the engine is as follows.

```java
// Conceptual example of engine-level usage
// A developer would NOT write this code.

// 1. Asset loader parses a JSON object for an action
JsonElement actionJson = ...; 
String actionType = "recomputePath"; // read from JSON

// 2. Loader looks up the correct builder in a registry
Builder<Action> builder = assetBuilderRegistry.get(actionType); // Returns a BuilderActionRecomputePath

// 3. Loader configures and builds the action
builder.readConfig(actionJson);
Action recomputeAction = builder.build(builderSupport);

// 4. The resulting action is added to an NPC's behavior
npcBehavior.addAction(recomputeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionRecomputePath()` in game logic. NPC behaviors must be defined in JSON asset files to ensure they are correctly managed by the asset pipeline. Direct creation bypasses this system entirely.
- **Extending for Custom Logic:** Do not extend this class to add custom behavior. To create a new NPC action, you must create a new `Action` class and a corresponding `Builder` class for it, then register the builder with the asset system.

## Data Pipeline
This builder is a single, critical step in the pipeline that transforms declarative NPC asset data into executable server objects.

> Flow:
> NPC_Behavior.json -> Server Asset Parser -> **BuilderActionRecomputePath** (Instantiation) -> `build()` call -> `ActionRecomputePath` (Instance) -> NPC Behavior Tree/State Machine

