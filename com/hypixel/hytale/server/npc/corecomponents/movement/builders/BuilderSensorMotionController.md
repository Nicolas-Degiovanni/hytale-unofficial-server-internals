---
description: Architectural reference for BuilderSensorMotionController
---

# BuilderSensorMotionController

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorMotionController extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorMotionController is a transient factory class within the server-side NPC asset pipeline. Its sole responsibility is to deserialize a specific JSON configuration block and construct a corresponding SensorMotionController instance.

This class is a concrete implementation of the Builder pattern, specialized for creating AI sensors. The NPC's behavior, defined in external JSON files, is composed of various components like sensors, instructions, and controllers. The asset loading system maintains a registry of builder classes, mapping a JSON type identifier to a specific builder implementation. When the system parses a sensor definition of this type, it instantiates BuilderSensorMotionController, passes it the relevant JSON data, and invokes its build method to produce the final runtime object.

The resulting SensorMotionController is then integrated into an NPC's behavior tree or state machine, where it is evaluated by the AI decision-making loop to check if a specific motion controller is currently active.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's NPC asset loading system during the deserialization of an NPC behavior definition file. It is never created manually by game logic developers.
- **Scope:** Ephemeral. The object's lifetime is strictly limited to the parsing and construction of a single SensorMotionController instance. It does not persist after the build method is called.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method returns. Ownership of the created SensorMotionController is transferred to the parent NPC behavior asset.

## Internal State & Concurrency
- **State:** The class holds mutable, transient state in the form of the `motionControllerName` field. This state is populated by the `readConfig` method and is read once during the construction of the SensorMotionController.
- **Thread Safety:** This class is **not thread-safe**. It is designed to operate exclusively within a single-threaded asset loading context. Concurrent access, particularly multiple calls to `readConfig` on the same instance, will result in a race condition and undefined behavior. The engine's asset pipeline must enforce serialized access.

## API Surface
The public API is minimal, designed for use only by the asset deserialization framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorMotionController instance using the internal state populated by `readConfig`. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the input JSON, validates the "MotionController" field, and stores it internally. Throws a configuration exception on invalid or missing data. |
| getMotionControllerName() | String | O(1) | Public accessor for the configured name. This is called by the SensorMotionController's constructor. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. The following example illustrates the conceptual interaction pattern managed by the NPC asset system.

```java
// Conceptual example of the asset loader's logic
JsonElement sensorJson = parseNpcDefinitionFile(...);

// The loader would instantiate this builder via reflection or a factory map
BuilderSensorMotionController builder = new BuilderSensorMotionController();

// 1. Configure the builder from the data source
builder.readConfig(sensorJson);

// 2. Build the final runtime object
Sensor sensor = builder.build(builderSupport);

// 3. The builder is now discarded and the sensor is registered with the NPC
npc.getBehaviorTree().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually instantiate this class with `new`. The NPC asset system is responsible for its creation and lifecycle.
- **State Re-use:** Do not attempt to re-use a builder instance to configure multiple sensors. Each instance is single-use and its internal state is not designed to be reset.
- **Build Before Configure:** Calling `build` before `readConfig` will result in a `SensorMotionController` with a null internal name, which will cause a `NullPointerException` during AI evaluation ticks.

## Data Pipeline
The class acts as a transformation step in the data flow from static configuration to live game object.

> Flow:
> NPC Behavior JSON File -> Server JSON Parser -> **BuilderSensorMotionController** (`readConfig`) -> **BuilderSensorMotionController** (`build`) -> `SensorMotionController` Instance -> NPC Behavior Tree

