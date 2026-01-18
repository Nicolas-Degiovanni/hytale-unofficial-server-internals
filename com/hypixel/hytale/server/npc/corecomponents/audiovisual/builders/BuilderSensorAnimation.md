---
description: Architectural reference for BuilderSensorAnimation
---

# BuilderSensorAnimation

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorAnimation extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorAnimation class is a factory component within the server's NPC asset pipeline. It adheres to the Builder design pattern and is responsible for translating a declarative JSON configuration into a concrete, executable `Sensor` object.

Its specific role is to construct a `SensorAnimation` instance. This resulting sensor is integrated into an NPC's behavior tree or state machine, where it functions as a conditional check. The AI uses this sensor to query whether the NPC is currently playing a specific animation in a given animation slot. This allows for the creation of complex, animation-driven behaviors, such as triggering an attack only after a wind-up animation completes, or transitioning to an idle state after a death animation finishes.

This class is a critical link in the data-driven design of NPC behavior, enabling game designers to define sophisticated logic in simple JSON files without modifying core engine code.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's asset loading system. When parsing an NPC's JSON definition, if a sensor of type "Animation" is encountered, the system creates a new BuilderSensorAnimation instance to process that specific JSON block.
- **Scope:** Ephemeral and short-lived. An instance exists only for the duration of parsing one sensor definition and building one `SensorAnimation` object.
- **Destruction:** The object is immediately eligible for garbage collection after the `build` method is called and its product, the `SensorAnimation`, is returned to the asset loader. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** The class is stateful but its state is transient. It internally stores the `animationSlot` and `animationId` parsed from the configuration source. These are held in `EnumHolder` and `StringHolder` wrappers, which are specialized containers that can defer value resolution until the `build` phase, using a provided execution context. The state is populated by `readConfig` and consumed by `build`.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated, configured, and used by a single thread within the asset loading pipeline. Concurrent access would result in a race condition and corrupt the internal state, leading to an incorrectly configured `Sensor` object. The asset system must enforce serialized, single-threaded access to each builder instance.

## API Surface
The public API is focused on the configuration and construction lifecycle of the `Sensor` object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new `SensorAnimation` instance based on the internal state. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the provided JSON, populating the internal `animationSlot` and `animationId` holders. |
| getAnimationSlot(BuilderSupport) | NPCAnimationSlot | O(1) | Resolves and returns the configured animation slot. |
| getAnimationId(BuilderSupport) | String | O(1) | Resolves and returns the configured animation ID. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked exclusively by the NPC asset loading framework. The typical flow involves the framework identifying the sensor type from JSON and delegating the construction process to this builder.

```java
// Conceptual example within an asset loading system
JsonElement sensorConfig = parseNpcDefinitionFile().get("sensor");
BuilderSensorAnimation builder = new BuilderSensorAnimation();

// Configure the builder with the specific JSON data
builder.readConfig(sensorConfig);

// Create the runtime sensor object
// The builderSupport object provides necessary context (e.g., ExecutionContext)
Sensor runtimeSensor = builder.build(builderSupport);

// Add the newly created sensor to the NPC's behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to reuse a BuilderSensorAnimation instance to build multiple sensors. Each instance is designed for a single `readConfig` -> `build` lifecycle. Re-calling `readConfig` on an existing instance can lead to unpredictable state.
- **Instantiation without Configuration:** Creating an instance via `new BuilderSensorAnimation()` and calling `build` without first calling `readConfig` will result in an invalid `Sensor` that will throw exceptions at runtime.

## Data Pipeline
The BuilderSensorAnimation acts as a transformation step in the data pipeline that converts static NPC definition files into live, in-memory game objects.

> Flow:
> NPC Definition (*.json file*) -> JSON Parser -> **BuilderSensorAnimation.readConfig()** -> **BuilderSensorAnimation.build()** -> `SensorAnimation` Instance -> NPC Behavior Tree Evaluation

---

