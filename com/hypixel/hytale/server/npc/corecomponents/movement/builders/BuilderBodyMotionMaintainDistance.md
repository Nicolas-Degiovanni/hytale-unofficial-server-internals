---
description: Architectural reference for BuilderBodyMotionMaintainDistance
---

# BuilderBodyMotionMaintainDistance

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderBodyMotionMaintainDistance extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionMaintainDistance class is a configuration factory within the server-side NPC asset pipeline. It is not a runtime component that directly controls an NPC. Instead, its sole responsibility is to parse a specific JSON configuration block from an NPC asset file and construct a fully configured, runtime-ready BodyMotionMaintainDistance instance.

This class is a concrete implementation of the abstract factory pattern used throughout the NPC behavior system. The engine maintains a registry of these builder classes, dispatching asset configuration data to the appropriate builder based on a type identifier in the JSON.

A critical architectural concept is the use of Holder objects, such as NumberArrayHolder and DoubleHolder, for its internal fields. This pattern decouples the raw configuration data from its resolved value. It allows NPC designers to use dynamic expressions or references within the JSON assets, which are then resolved at load-time via an ExecutionContext. This provides significant flexibility without complicating the runtime behavior code.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the NPC asset loading system when it encounters a corresponding behavior definition in a JSON file. It is never manually instantiated by game logic developers.
- **Scope:** The object's lifetime is extremely brief and confined to the asset parsing and validation phase. It exists only to process a single JSON object.
- **Destruction:** It becomes eligible for garbage collection immediately after its `build` method is invoked and the resulting BodyMotionMaintainDistance object is passed to the parent NPC definition. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The `readConfig` method populates all fields from the provided JSON data. This state is a temporary, intermediate representation of the final runtime configuration. Once the `build` method is called, the state is effectively transferred to the new BodyMotionMaintainDistance object and this builder is discarded.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be used in a single-threaded context during the asset loading sequence. The asset pipeline is responsible for ensuring that each builder instance is created, configured, and used without concurrent access.

## API Surface
The public API is designed for internal use by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionMaintainDistance | O(1) | Constructs the final runtime object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderBodyMotionMaintainDistance | O(N) | Parses the JSON configuration, populating the builder's internal state. N is the number of properties in the JSON object. |
| validate(...) | boolean | O(1) | Performs load-time validation, ensuring system dependencies like MotionControllerWalk are met. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's asset loading framework. A developer will never interact with it directly. The conceptual flow within the engine is as follows.

```java
// Conceptual engine code
// Engine identifies the correct builder for the JSON block
BuilderBodyMotionMaintainDistance builder = new BuilderBodyMotionMaintainDistance();

// Engine populates the builder from the asset file
builder.readConfig(npcBehaviorJson);

// Engine validates the configuration in the context of the NPC
builder.validate(assetName, validationHelper, context, scope, errors);

// Engine constructs the final runtime component
BodyMotionMaintainDistance runtimeComponent = builder.build(builderSupport);

// The 'builder' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderBodyMotionMaintainDistance()` in game logic. The resulting object would be unconfigured and useless without the asset pipeline to call `readConfig`.
- **State Reuse:** Do not retain a reference to this builder after the `build` method has been called. It is a one-shot factory and its state should not be reused or modified post-build.
- **Premature Build:** Calling `build` before `readConfig` will produce a component with default, and likely invalid, parameters. The asset pipeline guarantees the correct order of operations.

## Data Pipeline
This builder acts as a transformation stage in the NPC asset loading data pipeline. It converts declarative configuration data into an executable runtime object.

> Flow:
> NPC Asset JSON File -> JSON Parser -> **BuilderBodyMotionMaintainDistance.readConfig()** -> Validation & Expression Resolution -> **BuilderBodyMotionMaintainDistance.build()** -> BodyMotionMaintainDistance (Runtime Object) -> NPC Behavior System

