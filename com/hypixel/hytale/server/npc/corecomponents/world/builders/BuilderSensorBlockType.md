---
description: Architectural reference for BuilderSensorBlockType
---

# BuilderSensorBlockType

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorBlockType extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorBlockType is a specialized factory component within the server-side NPC (Non-Player Character) AI framework. Its primary role is to translate a declarative JSON configuration into a live, executable Sensor object. Specifically, it constructs a SensorBlockType, which is a runtime condition used by an NPC to detect if a block at a specific location belongs to a predefined BlockSet.

This class embodies the **Builder Pattern** as used throughout the Hytale asset system. It acts as a bridge between the static data definitions that describe an NPC's behavior and the dynamic, in-game AI engine. By externalizing logic into JSON, this builder enables designers to create complex NPC behaviors without modifying engine code.

It is a crucial component in the data-driven design of the NPC system, responsible for deserializing and validating one specific type of sensor configuration.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorBlockType is created by a higher-level asset loading mechanism, typically a GSON-based deserializer, when it encounters the corresponding type identifier in an NPC behavior JSON file.
- **Scope:** The object's lifetime is extremely short and confined to the asset loading phase. It exists only to parse a single JSON object, configure its internal state, and produce one SensorBlockType instance via its build method.
- **Destruction:** After the build method is called and the resulting Sensor is integrated into the NPC's behavior tree, the BuilderSensorBlockType instance is no longer referenced and becomes eligible for standard garbage collection. It holds no persistent resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** This class is fundamentally **mutable**. Its internal fields, such as sensor and blockSet, are populated by the readConfig method. This state is then consumed by the build method to construct the final Sensor object. The builder is designed for a single, linear sequence of operations: create, configure, build.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within a single-threaded asset loading pipeline. The internal state is unprotected, and concurrent calls to readConfig or build would result in a corrupted, unpredictable object state.

**WARNING:** Do not share instances of this builder across multiple threads. The asset loading system must guarantee that each builder instance is confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorBlockType instance using the state configured by readConfig. Returns null if the wrapped sensor fails to build. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the input JSON, populating internal fields like the target BlockSet and the wrapped Sensor. Performs validation. |
| getSensor(BuilderSupport) | Sensor | O(1) | Builds and returns the nested Sensor object that this sensor wraps. |
| getBlockSet(BuilderSupport) | int | O(1) | Resolves the configured BlockSet asset name into its runtime integer index. **Throws IllegalArgumentException** if the BlockSet name is not found in the global asset map. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by game logic developers. It is invoked automatically by the NPC asset deserialization process. The conceptual flow is as follows.

```java
// Conceptual example of how the asset system uses this builder.
// A higher-level parser would instantiate and invoke the builder.

JsonElement sensorConfig = parseNpcBehaviorFile(".../my_npc.json");
BuilderSensorBlockType builder = new BuilderSensorBlockType();

// 1. Configure the builder from the data file
builder.readConfig(sensorConfig);

// 2. Build the final runtime object
BuilderSupport support = new BuilderSupport(/* ... */);
Sensor runtimeSensor = builder.build(support);

// 3. Integrate the sensor into the NPC's AI
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance with `new BuilderSensorBlockType()` and expect it to function. The object is useless until its state is populated via the `readConfig` method.
- **State Re-use:** Do not attempt to re-use a single builder instance to build multiple Sensor objects. The internal state is not reset between `build` calls, which will lead to incorrect or corrupt AI behavior.
- **Premature Building:** Calling `build` before `readConfig` will result in an improperly configured Sensor, likely causing NullPointerExceptions or other runtime failures.

## Data Pipeline
The BuilderSensorBlockType is a key transformation step in the NPC behavior pipeline, converting static data into an executable object.

> Flow:
> NPC Behavior JSON File -> GSON Deserializer -> **BuilderSensorBlockType.readConfig()** -> **BuilderSensorBlockType.build()** -> SensorBlockType Instance -> NPC Behavior Tree -> Live AI Decision Making

