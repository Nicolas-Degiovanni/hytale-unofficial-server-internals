---
description: Architectural reference for BuilderSensorBeacon
---

# BuilderSensorBeacon

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorBeacon extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorBeacon is a transient, data-driven factory class within the server-side NPC asset pipeline. It adheres to the **Builder Pattern** and is responsible for translating a JSON configuration block into a fully realized SensorBeacon runtime object.

This class acts as a critical bridge between static data assets and live game objects. NPC behaviors in Hytale are defined in external JSON files, allowing designers to create complex AI without modifying engine code. When the server loads an NPC definition, it encounters various component configurations. If a component is a "SensorBeacon", the asset system instantiates this builder to parse the corresponding JSON data.

The resulting SensorBeacon object is then attached to an NPC's AI controller. During the game loop, the sensor will actively check for broadcasted messages from other entities, forming a core part of the NPC's environmental awareness and interaction system.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's asset loading framework. A central factory or registry identifies the "SensorBeacon" type in an NPC's JSON definition and creates a new BuilderSensorBeacon instance to handle its configuration.
- **Scope:** The lifecycle of a BuilderSensorBeacon instance is extremely short. It exists only for the duration of parsing a single JSON object and building one SensorBeacon instance. It does not persist into the active game state.
- **Destruction:** The object is immediately eligible for garbage collection after the `build` method returns the configured SensorBeacon. Its memory footprint is reclaimed as part of the asset loading process, not during the main game loop.

## Internal State & Concurrency
- **State:** The BuilderSensorBeacon is fundamentally stateful and mutable. Its fields, such as `message`, `range`, and `targetSlot`, are populated sequentially as the `readConfig` method processes a JSON element. This internal state is temporary and serves only to accumulate the necessary parameters for the final SensorBeacon constructor call.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within the asset loading pipeline. Accessing an instance from multiple threads would lead to race conditions and unpredictable state corruption. This design is intentional and performant for its designated role.

## API Surface
The public API is designed for a two-phase process: configuration followed by construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Sensor> | O(N) | Parses the provided JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| build(BuilderSupport support) | SensorBeacon | O(1) | Constructs and returns the final SensorBeacon runtime object using the previously configured state. |
| getMessageSlot(BuilderSupport support) | int | O(1) | Resolves the string-based message name into a more efficient integer slot ID via the BuilderSupport context. |
| getTargetSlot(BuilderSupport support) | int | O(1) | Resolves the string-based target slot name into its corresponding integer ID. Returns MIN_VALUE if no slot is defined. |

## Integration Patterns

### Standard Usage
This class is intended to be used exclusively by the NPC asset loading system. The typical flow involves instantiation, configuration from JSON, and a single build operation.

```java
// Pseudo-code for engine-level asset loading
JsonElement sensorConfig = parseNpcDefinitionFile(".../my_npc.json");

// 1. The factory creates the appropriate builder
BuilderSensorBeacon builder = new BuilderSensorBeacon();

// 2. The builder is configured from the data source
builder.readConfig(sensorConfig);

// 3. The final runtime object is constructed
// BuilderSupport provides context like symbol-to-ID mappings
SensorBeacon runtimeSensor = builder.build(engine.getBuilderSupport());

// 4. The sensor is attached to the NPC
npc.getAIController().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not attempt to re-use a single BuilderSensorBeacon instance to build multiple SensorBeacon objects. The internal state is not reset between `build` calls, which will result in subsequent objects being created with corrupted or blended data. Always create a new builder for each distinct component configuration.
- **Build Before Configure:** Calling `build` before `readConfig` will produce a SensorBeacon with default or null values, leading to severe and hard-to-diagnose runtime errors or non-functional AI behavior.
- **Direct Instantiation in Game Logic:** This is a configuration-time class. Never instantiate a BuilderSensorBeacon within the main game loop or in response to game events. Its purpose is strictly limited to the initial asset loading phase.

## Data Pipeline
The BuilderSensorBeacon is a specific step in the data transformation pipeline that turns a static asset file into a live, interactive game component.

> Flow:
> NPC Definition JSON File -> JSON Parser -> `JsonElement` -> **BuilderSensorBeacon.readConfig()** -> Internal Builder State -> **BuilderSensorBeacon.build()** -> `SensorBeacon` Object -> NPC AI Controller

