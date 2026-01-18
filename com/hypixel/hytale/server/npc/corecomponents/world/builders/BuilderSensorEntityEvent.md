---
description: Architectural reference for BuilderSensorEntityEvent
---

# BuilderSensorEntityEvent

**Package:** `com.hypixel.hytale.server.npc.corecomponents.world.builders`
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorEntityEvent extends BuilderSensorEvent {
```

## Architecture & Concepts
The BuilderSensorEntityEvent is a factory class that operates within the server-side NPC asset pipeline. Its primary role is to translate a declarative JSON configuration into a concrete, runtime `SensorEntityEvent` object. This class is a fundamental component of Hytale's data-driven AI design, allowing game designers to define complex NPC sensory behaviors without writing code.

This builder acts as a deserializer and constructor. When the server loads an NPC's behavior definition from a JSON file, it identifies the sensor type and instantiates the corresponding builder. This builder then consumes the relevant JSON block, populates its internal state, and finally, produces a fully configured `Sensor` instance. The resulting `SensorEntityEvent` is then attached to an NPC's behavior tree, where it actively monitors the game world for specific entity events (like damage or death) and feeds that information into the AI's decision-making process.

The use of intermediate `Holder` objects (e.g., `AssetHolder`, `EnumHolder`) provides a layer of abstraction for parsing and validating configuration values before the final `Sensor` is constructed.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the NPC asset loading system, typically via reflection, when it encounters a JSON configuration for a sensor of this type. It is never created manually during gameplay logic.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single JSON sensor definition and building one `Sensor` object.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and its resulting `SensorEntityEvent` has been returned. It holds no persistent state and is not retained by any system.

## Internal State & Concurrency
- **State:** Highly mutable. The class's fields (`npcGroup`, `entityEventType`, `flockOnly`) are designed to be populated by the `readConfig` method. This state is transient and serves only as a temporary container for configuration data before it is passed to the `SensorEntityEvent` constructor.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed to operate within a single-threaded asset loading context. Accessing an instance from multiple threads will lead to race conditions and unpredictable state. The NPC asset pipeline guarantees serialized, single-threaded access for each builder instance.

## API Surface
The public API is designed for internal use by the asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new `SensorEntityEvent` instance using the state populated by `readConfig`. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the provided JSON, populating the internal holder fields. N is the number of keys in the JSON object. |
| getNPCGroup(BuilderSupport) | int | O(log M) | Resolves the NPC group name from the config into a performance-optimized integer index. Throws `IllegalArgumentException` if the group name is not found. |
| getEventType(BuilderSupport) | EntityEventType | O(1) | Retrieves the configured event type to listen for. |
| isFlockOnly(BuilderSupport) | boolean | O(1) | Retrieves the configured flag indicating if the sensor should only react to events from the NPC's own flock. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal NPC asset loading system. A developer would not interact with it directly but would instead define its behavior in a JSON file.

```java
// This is an internal engine process, not typical user code.

// 1. A JSON configuration for an NPC is loaded.
JsonElement sensorConfig = loadNpcBehaviorFile("...").get("sensor");

// 2. The engine instantiates the correct builder based on the config type.
BuilderSensorEntityEvent builder = new BuilderSensorEntityEvent();

// 3. The builder reads the configuration.
builder.readConfig(sensorConfig);

// 4. The builder constructs the final runtime object.
BuilderSupport support = engine.getBuilderSupport();
Sensor sensor = builder.build(support);

// 5. The sensor is attached to the NPC's behavior tree.
npc.getBehaviorTree().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to reuse a builder instance to create multiple `Sensor` objects. The internal state is not reset after a `build` call. A new builder must be instantiated for each unique JSON configuration block.
- **Calling `build` before `readConfig`:** Invoking `build` on a fresh instance without first calling `readConfig` will result in a `Sensor` configured with default or null values, leading to runtime exceptions or silent AI failures.
- **Manual Instantiation in Game Logic:** Do not use `new BuilderSensorEntityEvent()` in gameplay systems. NPC behaviors should be defined declaratively in JSON assets. Bypassing this pattern breaks the data-driven design of the AI system.

## Data Pipeline
This builder is a critical transformation step, converting static configuration data into a live, functional game object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderSensorEntityEvent** (`readConfig`) -> `SensorEntityEvent` Instance (`build`) -> NPC Behavior Tree

