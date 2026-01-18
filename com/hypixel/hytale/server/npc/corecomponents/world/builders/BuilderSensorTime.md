---
description: Architectural reference for BuilderSensorTime
---

# BuilderSensorTime

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorTime extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorTime class is a configuration and construction component within the server's NPC asset loading pipeline. It embodies the **Builder Pattern**, serving as a transient intermediary that translates a declarative JSON configuration into a concrete, executable `SensorTime` object.

Its primary architectural function is to decouple the complex and error-prone process of data parsing, validation, and default value assignment from the runtime logic of the sensor itself. This class exists only during the asset deserialization phase. Once the `build` method is called and a `SensorTime` instance is produced, the builder's purpose is complete.

This pattern ensures that runtime components like `SensorTime` are instantiated in a valid and complete state, without being burdened by configuration-parsing logic.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderSensorTime is created dynamically by the NPC asset factory when it encounters a sensor of type "Time" within an NPC's JSON behavior definition. It is never instantiated directly by game logic code.
-   **Scope:** The object's lifetime is exceptionally brief. It exists only for the duration of parsing a single JSON object and constructing one `SensorTime` instance.
-   **Destruction:** The instance becomes eligible for garbage collection immediately after the `build` method returns. There are no external references maintained to the builder post-construction.

## Internal State & Concurrency
-   **State:** The internal state is highly **mutable**. Fields such as `period`, `checkDay`, and `checkYear` are populated and modified sequentially during the `readConfig` call. The object acts as a temporary data container for configuration values.

-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. It must only be used within the server's asset loading thread. Concurrent calls to `readConfig` would result in a corrupted and unpredictable configuration state. This is an intentional design choice, as the asset pipeline operates sequentially.

## API Surface
The public API is focused on the two-stage process of configuration and construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Sensor> | O(N) | Parses the provided JSON, validates fields, and populates the builder's internal state. N is the number of keys in the JSON object. |
| build(BuilderSupport support) | Sensor | O(1) | Constructs and returns a new `SensorTime` instance using the currently configured state. |
| getPeriod(BuilderSupport support) | double[] | O(1) | Resolves the time period. This may involve evaluating dynamic expressions via the `ExecutionContext` in the provided `BuilderSupport`. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked by the underlying asset loading system. The conceptual flow is as follows.

```java
// Hypothetical asset loading system code
JsonElement sensorConfig = parseNpcDefinitionFile("my_npc.json");

// The system instantiates the correct builder
BuilderSensorTime builder = new BuilderSensorTime();

// 1. Configure the builder from the JSON data
builder.readConfig(sensorConfig);

// 2. Construct the final runtime object
// BuilderSupport provides necessary runtime context
Sensor sensor = builder.build(builderSupport);

// The builder is now discarded and the sensor is added to the NPC
npc.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not retain an instance of BuilderSensorTime for reuse. Each sensor definition from JSON must be processed by a new builder instance to prevent state leakage and incorrect configurations.
-   **Premature Build:** Do not call `build` before `readConfig` has been successfully invoked. This will produce a `SensorTime` object with default or uninitialized values, leading to behavior that is difficult to debug.
-   **Direct Modification:** Do not modify the public fields of this class after `readConfig` has been called. The internal state is considered sealed after parsing.

## Data Pipeline
BuilderSensorTime is a critical step in the data transformation pipeline that turns a static asset file into a live game object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> `JsonElement` -> **BuilderSensorTime.readConfig()** -> **BuilderSensorTime.build()** -> `SensorTime` Instance -> NPC Behavior Tree

