---
description: Architectural reference for BuilderBodyMotionLeave
---

# BuilderBodyMotionLeave

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionLeave extends BuilderBodyMotionFindBase {
```

## Architecture & Concepts
The BuilderBodyMotionLeave class is a factory component within the server-side NPC asset system. Its primary function is to deserialize a JSON configuration object into a concrete, executable movement behavior, specifically the BodyMotionLeave instance.

This class operates as a transient intermediary during the NPC asset loading phase. The engine's asset pipeline identifies a JSON block for a "leave" motion, instantiates this builder, and uses it to parse the configuration. The builder validates the input data—such as the required *Distance*—and holds it in a pre-processed format. The final BodyMotionLeave object, which contains the runtime logic for an NPC to move away from a location, is then produced by the build method.

The use of a DoubleHolder for the distance parameter is significant. It decouples the static configuration (the JSON file) from the dynamic runtime context, allowing for values that might be resolved later using an ExecutionContext provided via the BuilderSupport object.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the NPC asset loading framework when it encounters a corresponding type identifier in an NPC's behavior definition JSON. It is never created directly by game logic code.
- **Scope:** Ephemeral. The lifecycle of a BuilderBodyMotionLeave instance is strictly confined to the duration of the asset parsing and compilation process for a single NPC behavior.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and the resulting BodyMotionLeave object has been integrated into the parent behavior tree. It holds no persistent references and is not intended to outlive the asset loading transaction.

## Internal State & Concurrency
- **State:** Mutable. The core purpose of this class is to accumulate state from a JSON source via the `readConfig` method. The primary state is the `distance` field, which is modified during configuration.
- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the asset loading pipeline. Concurrent calls to `readConfig` will result in a race condition and corrupt the builder's internal state. The asset system must guarantee that each builder instance is configured and used in a single, isolated thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionLeave | O(1) | Constructs the final BodyMotionLeave runtime object using the configured state. |
| readConfig(JsonElement) | BuilderBodyMotionFindBase | O(N) | Populates the builder's internal state from a JSON object. Throws a configuration error if the required "Distance" key is missing or invalid. N is the number of keys in the JSON. |
| getDistance(BuilderSupport) | double | O(1) | Resolves and returns the configured distance value, potentially using the runtime context from the BuilderSupport. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal NPC asset system. A developer defining NPC behavior in JSON is an indirect user, but they will never interact with the Java class itself. The conceptual usage by the engine is as follows.

```java
// Engine-level pseudo-code for asset loading
JsonElement motionConfig = parseNpcBehaviorFile(".../behavior.json");

// The engine instantiates the correct builder based on a "type" field in the JSON
BuilderBodyMotionLeave builder = new BuilderBodyMotionLeave();

// The builder is configured from the specific JSON block
builder.readConfig(motionConfig);

// The final, immutable behavior object is created
BodyMotionLeave runtimeMotion = builder.build(engine.getBuilderSupport());

// The builder is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class with `new BuilderBodyMotionLeave()` in game logic. The NPC asset system is solely responsible for its creation and lifecycle.
- **State Re-use:** Do not attempt to cache and re-use a builder instance. It is stateful and must be re-created for each distinct behavior configuration being parsed.
- **Build Before Configure:** Calling `build` before `readConfig` will produce a behavior object with uninitialized or default state, leading to undefined runtime behavior or validation failures.

## Data Pipeline
This component functions within a configuration data pipeline, transforming declarative JSON into an executable server object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> `JsonElement` -> **BuilderBodyMotionLeave.readConfig()** -> Internal State (distance) -> **BuilderBodyMotionLeave.build()** -> BodyMotionLeave Instance

