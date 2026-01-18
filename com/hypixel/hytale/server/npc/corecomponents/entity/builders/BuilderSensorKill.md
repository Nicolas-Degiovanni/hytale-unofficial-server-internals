---
description: Architectural reference for BuilderSensorKill
---

# BuilderSensorKill

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorKill extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorKill class is a factory component within the server-side NPC asset pipeline. Its sole responsibility is to deserialize a JSON configuration object and construct a runtime instance of a SensorKill component. It acts as a translator between declarative NPC behavior definitions stored in asset files and the concrete, executable sensor objects used by the NPC's AI at runtime.

This class follows the **Builder Pattern**, a common design in Hytale's asset system. Each builder is responsible for a specific, small piece of an NPC's configuration. A higher-level asset manager discovers and instantiates these builders based on type identifiers in the JSON data, orchestrates the configuration process by calling readConfig, and finally invokes the build method to produce the final runtime object.

The resulting SensorKill object is a **Sensor**, a type of instruction that allows an NPC's Behavior Tree or State Machine to query the game state. Specifically, this sensor answers the question: "Did I recently kill an entity, and if so, does it match a specific target slot?"

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorKill is created by the NPC asset loading system when it parses an NPC definition file and encounters a sensor of this type. It is never instantiated directly by game logic code.
- **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single sensor definition within a single NPC asset.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and the resulting SensorKill object has been integrated into the parent NPC's component collection. It holds no persistent references and is not intended to outlive the asset loading phase.

## Internal State & Concurrency
- **State:** Mutable and transient. The primary state is the `targetSlot` field, a StringHolder that stores the name of the target slot to be checked. This state is populated exclusively by the `readConfig` method and is only considered valid during the build process.
- **Thread Safety:** This class is **not thread-safe** and must only be used within a single-threaded context. The asset loading pipeline, where this class operates, is designed as a serialized, synchronous process. Concurrent access to an instance of BuilderSensorKill would result in a race condition when populating the `targetSlot` field, leading to unpredictable and corrupt NPC behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorKill | O(1) | Constructs and returns the final SensorKill runtime object. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Populates the builder's internal state from a JSON object. N is the number of keys. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its runtime integer ID via the BuilderSupport context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the server's asset loading framework. The conceptual flow is as follows:

```java
// Conceptual example of the asset loading system's interaction
// This code does not exist literally but illustrates the pattern.

JsonElement sensorConfig = parseNpcAssetFile(".../my_npc.json");
BuilderSensorKill builder = builderRegistry.createBuilderFor("SensorKill");

// The framework populates the builder from the config
builder.readConfig(sensorConfig);

// The framework provides runtime context and builds the final object
SensorKill runtimeSensor = builder.build(assetLoadingSupport);
npc.addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorKill()`. NPC behavior should be defined declaratively in JSON asset files, not programmatically constructed.
- **State Reuse:** Do not reuse a single builder instance to configure and build multiple SensorKill objects. Each builder is single-use and should be discarded after one build cycle.
- **Runtime Configuration:** Do not invoke `readConfig` outside of the initial server boot or asset loading phase. It is a high-cost operation that depends on a static configuration context.

## Data Pipeline
The BuilderSensorKill is a key stage in the transformation of static data into a runtime game object.

> Flow:
> NPC JSON Asset File -> Gson Parser -> **BuilderSensorKill.readConfig()** -> **BuilderSensorKill.build()** -> SensorKill Instance -> NPC Behavior Component List

