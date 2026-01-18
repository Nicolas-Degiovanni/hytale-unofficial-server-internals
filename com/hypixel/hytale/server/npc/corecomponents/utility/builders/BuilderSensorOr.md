---
description: Architectural reference for BuilderSensorOr
---

# BuilderSensorOr

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorOr extends BuilderSensorMany {
```

## Architecture & Concepts
The BuilderSensorOr class is a factory component within the server-side NPC AI asset pipeline. It embodies the Builder pattern to construct a `SensorOr` instance from declarative asset definitions, likely specified in JSON or a similar format.

Its primary role is to translate a data-driven AI configuration into a concrete, runtime game logic object. The resulting `SensorOr` is a composite sensor that aggregates multiple child sensors. It signals true if *any* of its children signal true, implementing a logical OR operation. This allows AI designers to create complex, multi-faceted conditions for NPC behavior, such as reacting when a player is nearby OR when the NPC's health is low.

This builder is not intended for direct use by game logic developers. Instead, it is discovered and invoked by a higher-level NPC asset loading system that parses an NPC's definition and instantiates the necessary components.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading framework. The framework reads a type identifier (e.g., "SensorOr") from the asset file and uses a registry or reflection to create an instance of this specific builder.
- **Scope:** Extremely short-lived. A BuilderSensorOr instance exists only for the duration of a single `build` operation during the NPC's initial construction. It is a single-use, temporary object.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method returns its `SensorOr` product. It holds no references that would extend its lifetime beyond the asset instantiation phase.

## Internal State & Concurrency
- **State:** The builder is stateful, inheriting an `objectListHelper` from its parent `BuilderSensorMany`. This helper accumulates the configuration for the child sensors that will be composed into the final `SensorOr` object. This internal state is populated by the asset deserializer before the `build` method is called.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used by a single thread during the construction of a single NPC. Attempting to share a builder instance across threads or reuse it for multiple `build` operations will result in unpredictable behavior and state corruption. Asset loading systems must ensure that each builder instance is confined to a single, synchronous build task.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorOr | O(N) | Constructs a SensorOr object. N is the number of child sensors defined in the asset. Returns null if no child sensors are configured, effectively pruning this logical branch. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling and editors. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling and editors. |

## Integration Patterns

### Standard Usage
This class is used internally by the engine's asset loading systems. A developer defines the sensor in a data file, and the engine handles the construction.

```java
// Conceptual representation of engine-level usage
// This code is NOT written by a typical developer.

// 1. Engine deserializes asset data onto the builder instance
BuilderSensorOr builder = assetLoader.createBuilder("SensorOr");
assetLoader.configureBuilder(builder, sensorOrJsonData);

// 2. Engine invokes the build method to create the runtime component
BuilderSupport support = new BuilderSupport(...);
SensorOr runtimeSensor = builder.build(support);

// 3. Engine attaches the component to the NPC
npc.getAIComponent().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new BuilderSensorOr()`. NPC behavior should be defined in data assets, not hard-coded. The system relies on declarative configuration to function correctly.
- **Instance Reuse:** Do not call `build` more than once on a single builder instance. The internal state is not designed to be reset, and subsequent calls will produce invalid or unexpected results.
- **Manual Configuration:** Do not attempt to access or modify the internal `objectListHelper`. The builder's state should be managed exclusively by the asset deserialization process.

## Data Pipeline
This component operates within a configuration and instantiation pipeline, not a real-time game loop data pipeline.

> Flow:
> NPC Definition Asset (JSON) -> Asset Deserialization -> **BuilderSensorOr** Instantiation & Configuration -> `build()` Invocation -> `SensorOr` Object -> NPC AI Component Graph

