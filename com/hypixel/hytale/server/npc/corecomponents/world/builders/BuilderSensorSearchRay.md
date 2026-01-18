---
description: Architectural reference for BuilderSensorSearchRay
---

# BuilderSensorSearchRay

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorSearchRay extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorSearchRay is a factory class responsible for translating a data-driven JSON configuration into a functional SensorSearchRay instance. It serves as a critical bridge between the server's asset loading pipeline and the runtime NPC AI system.

Its primary architectural role is to enforce the principle of **configuration over code**. By defining a sensor's properties—such as its angle, range, and target block types—in external JSON files, this builder allows designers to create and tune complex NPC behaviors without modifying core engine code.

The builder pattern is used here to encapsulate the complex logic of parsing, validating, and preparing sensor parameters. It consumes a raw JsonElement, applies a series of validators (e.g., DoubleRangeValidator, BlockSetExistsValidator), and populates its internal state. The final, validated configuration is then used to construct the immutable, runtime SensorSearchRay object.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorSearchRay is created by the NPC asset loading system whenever it encounters a sensor of type "SearchRay" within an NPC's behavior definition file. It is never instantiated directly by game logic.
- **Scope:** Ephemeral. The object's lifetime is strictly limited to the parsing and instantiation phase of a single NPC type's AI definition. It does not persist into the game state.
- **Destruction:** The builder becomes eligible for garbage collection immediately after its `build` method is invoked and the resulting SensorSearchRay is integrated into the NPC's behavior tree. It holds no persistent references and is not intended to be reused.

## Internal State & Concurrency
- **State:** The internal state is composed of various `Holder` objects (e.g., StringHolder, FloatHolder) that store the parsed configuration. This state is **mutable** exclusively during the invocation of the `readConfig` method. Once parsing is complete, the state should be considered immutable.

- **Thread Safety:** This class is **not thread-safe** and is fundamentally tied to the server's main thread during asset loading. All operations, from creation through the `build` call, must be performed sequentially within a single-threaded context. Concurrent access will lead to unpredictable behavior and state corruption.

## API Surface
The public API is designed for two distinct phases: configuration by the asset system and construction of the final sensor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | Builder | O(N) | Parses and validates the JSON definition. Throws exceptions on validation failure. N is the number of properties in the JSON object. |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorSearchRay instance using the previously parsed configuration. |
| getAngle(BuilderSupport) | float | O(1) | Retrieves the configured angle, converted to radians. Called by the SensorSearchRay during its operation. |
| getRange(BuilderSupport) | double | O(1) | Retrieves the configured search range. |
| getBlockSet(BuilderSupport) | int | O(1) | Resolves the configured block set name to its internal integer ID. Throws IllegalArgumentException if the asset key is unknown. |
| getId(BuilderSupport) | int | O(1) | Resolves the sensor's string ID to a unique integer slot for caching purposes, managed by the BuilderSupport context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked internally by the NPC asset pipeline. The conceptual flow is as follows:

```java
// Conceptual example of the asset pipeline's usage
JsonElement sensorDefinition = loadNpcBehaviorFile("sentry_bot.json");
BuilderSensorSearchRay builder = new BuilderSensorSearchRay();

// 1. Configure the builder from the data source
builder.readConfig(sensorDefinition);

// 2. Provide a runtime context and build the final object
// The BuilderSupport provides access to runtime services.
Sensor sensor = builder.build(npc.getBuilderSupport());

// 3. The builder is now discarded, and the sensor is used by the NPC.
npc.getBehaviorTree().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderSensorSearchRay()` in game logic. NPC behaviors must be defined in JSON asset files.
- **State Mutation:** Do not attempt to modify the builder's internal state after `readConfig` has been called. The object is not designed for reconfiguration.
- **Instance Caching:** Do not cache or reuse builder instances. A new builder must be created for each distinct sensor definition to ensure state isolation.
- **Cross-Thread Access:** Never pass a builder instance to another thread. It must be created, configured, and used on the server's main asset-loading thread.

## Data Pipeline
The BuilderSensorSearchRay is a specific step in the data pipeline that transforms declarative NPC configuration into executable server-side logic.

> Flow:
> NPC JSON Asset -> GSON Parser -> **BuilderSensorSearchRay.readConfig()** -> **BuilderSensorSearchRay.build()** -> SensorSearchRay Instance -> NPC Behavior Tree

