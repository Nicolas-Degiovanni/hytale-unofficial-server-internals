---
description: Architectural reference for BuilderActionRemove
---

# BuilderActionRemove

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionRemove extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionRemove class is a factory component within the server's data-driven NPC behavior system. It serves as a bridge between static configuration data (JSON assets) and executable runtime objects. Its sole responsibility is to parse a specific JSON definition for a "remove entity" action and construct an instance of the corresponding runtime instruction, ActionRemove.

This class is a critical element of the asset pipeline. It allows game designers to specify that an NPC should be removed from the world within a behavior tree or script, without writing any Java code. The builder pattern here separates the complex logic of configuration parsing from the lean, focused execution logic of the final ActionRemove object.

The use of a BuilderSupport context object in its methods indicates that the build process is not simple instantiation. It is a context-aware operation that may resolve values or perform validation against the current execution environment.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderActionRemove is created by the server's asset loading framework. When the framework parses an NPC behavior asset and encounters a JSON object with a type identifier corresponding to "ActionRemove", it instantiates this builder to handle that specific block of data.

- **Scope:** The lifecycle of a BuilderActionRemove instance is extremely short and confined to the asset parsing phase. It is created, configured via the readConfig method, used once to produce an ActionRemove object via the build method, and is then immediately eligible for garbage collection.

- **Destruction:** Managed by the Java garbage collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The primary internal state is the useTarget field, a BooleanHolder. This object encapsulates whether the action should target the NPC's current sensor-acquired target or the NPC itself. The state is mutable and is populated exclusively by the readConfig method. The use of BooleanHolder instead of a primitive boolean suggests that the final value may be resolved dynamically at runtime using an ExecutionContext.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within the confines of a single-threaded asset loading process. Modifying its state via readConfig from multiple threads will lead to a corrupted configuration and unpredictable server behavior. All synchronization must be handled externally by the calling asset management system.

## API Surface
The public API is minimal, focusing entirely on the factory pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new ActionRemove instance based on the builder's configured state. |
| readConfig(JsonElement) | Builder<Action> | O(N) | Parses the input JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| getUseTarget(BuilderSupport) | boolean | O(1) | Resolves and returns the configured useTarget flag within a given runtime context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the server's asset pipeline. The conceptual flow within that system is as follows.

```java
// Pseudocode for engine's asset loader
JsonElement actionJson = parseNpcBehaviorFile(".../behavior.json");
BuilderActionRemove builder = new BuilderActionRemove();

// 1. Configure the builder from the data asset
builder.readConfig(actionJson);

// 2. Build the final, executable action object
// The builderSupport object is provided by the engine
ActionRemove runtimeAction = (ActionRemove) builder.build(engine.getBuilderSupport());

// 3. Add the action to a behavior tree or other runtime structure
behaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class manually in game logic. NPC behaviors must be defined in JSON asset files, which are then processed by the engine. Manually creating builders bypasses the entire data-driven framework.
- **State Re-use:** Do not call build, modify state, and call build again. A builder instance is meant to configure and build a single action. For each action defined in a data file, a new builder instance must be created.
- **Configuration Omission:** Calling build before readConfig has been successfully invoked will result in an ActionRemove object with an uninitialized, default configuration. This will almost certainly lead to incorrect runtime behavior.

## Data Pipeline
BuilderActionRemove is a transformation step in the pipeline that converts declarative data into an executable object.

> Flow:
> NPC Behavior JSON Asset -> Server Asset Loader -> Gson Parser -> JsonElement -> **BuilderActionRemove.readConfig()** -> Configured Builder Instance -> **BuilderActionRemove.build()** -> ActionRemove Object -> NPC Behavior Tree -> Action Execution Queue

