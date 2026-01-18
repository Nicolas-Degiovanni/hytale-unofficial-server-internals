---
description: Architectural reference for BuilderSensorCanPlace
---

# BuilderSensorCanPlace

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorCanPlace extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorCanPlace class is a configuration-driven factory within the server's NPC Behavior asset pipeline. Its sole responsibility is to deserialize a JSON configuration block and construct an instance of a SensorCanPlace, which is the runtime component that performs the actual in-world checks.

This class acts as a bridge between the static data representation of an NPC's behavior (defined in JSON files) and the live, executable object graph that constitutes an NPC's brain. It is a manifestation of the **Builder Pattern**, ensuring that the complex SensorCanPlace object is always constructed in a valid state.

A critical architectural concept employed here is the use of `Holder` objects (e.g., DoubleHolder, EnumHolder). These objects encapsulate configuration values, deferring their final resolution until runtime. This allows a single static asset definition to produce behaviors that can be dynamically influenced by an `ExecutionContext` (e.g., world state, NPC parameters) when the sensor is evaluated.

This builder also declares its required capabilities by calling `provideFeature(Feature.Position)`, integrating with a dependency system to ensure the NPC has the necessary components for this sensor to function correctly.

## Lifecycle & Ownership
- **Creation:** Instances of BuilderSensorCanPlace are created reflectively by the NPC asset loading system when it parses a behavior definition file. A developer will never instantiate this class directly.
- **Scope:** The lifecycle of a builder instance is exceptionally short and confined to the asset loading phase. It exists only to parse its corresponding JSON block and execute its `build` method once.
- **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method returns the configured SensorCanPlace instance. The resulting `Sensor` is then owned by the NPC's behavior tree and lives as long as the NPC itself.

## Internal State & Concurrency
- **State:** The internal state consists of several `Holder` fields that store the deserialized configuration. This state is mutable exclusively during the call to `readConfig`. After configuration, the object should be treated as immutable. It holds no runtime state; all state pertains to the configuration of the object it will create.
- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The asset loading pipeline, where this class is used, is designed to be a single-threaded process. Concurrent calls to `readConfig` would result in a corrupted and unpredictable configuration state.

## API Surface
The public API is designed for two distinct phases: configuration (`readConfig`) and construction (`build`). The accessor methods are intended for use by the `SensorCanPlace` instance that this builder creates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | **Factory Method.** Constructs and returns a fully configured SensorCanPlace instance. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | **Configuration Method.** Populates the builder's internal state from a JSON object. N is the number of keys. |
| getDirection(BuilderSupport) | SensorCanPlace.Direction | O(1) | Resolves the configured placement direction from the internal holder using the runtime context. |
| getOffset(BuilderSupport) | SensorCanPlace.Offset | O(1) | Resolves the configured placement offset from the internal holder using the runtime context. |
| getRetryDelay(BuilderSupport) | double | O(1) | Resolves the configured retry delay from the internal holder using the runtime context. |
| isAllowEmptyMaterials(BuilderSupport) | boolean | O(1) | Resolves the configured empty materials flag from the internal holder using the runtime context. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the server's internal asset loading systems. A content designer or developer defines the sensor's properties in a JSON file, which the system then uses to instantiate and configure this builder.

The following example demonstrates the *conceptual* flow, not code a developer would write.

```java
// System-level code during NPC asset loading
JsonElement sensorConfig = parseNpcDefinitionFile("my_npc.json");

// The system identifies the correct builder and instantiates it
BuilderSensorCanPlace builder = new BuilderSensorCanPlace();

// The system configures the builder from the JSON data
builder.readConfig(sensorConfig);

// The system builds the final runtime sensor object
BuilderSupport support = createBuilderSupportForNpc();
Sensor runtimeSensor = builder.build(support);

// The sensor is now ready to be added to the NPC's behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorCanPlace()` in game logic. The NPC asset pipeline manages the entire lifecycle of builders.
- **State Re-Configuration:** Do not call `readConfig` more than once on a single builder instance. This can lead to unpredictable behavior and corrupted state. Builders are single-use.
- **Instance Caching:** Do not cache or reuse builder instances. They are lightweight, transient objects designed to be created and discarded during the loading process.

## Data Pipeline
The BuilderSensorCanPlace serves as a specific, typed step in the data transformation pipeline that turns static asset files into live game objects.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderSensorCanPlace.readConfig()** -> **BuilderSensorCanPlace.build()** -> SensorCanPlace Instance -> NPC Behavior Tree

