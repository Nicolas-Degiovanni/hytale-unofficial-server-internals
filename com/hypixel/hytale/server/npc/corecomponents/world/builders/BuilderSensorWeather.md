---
description: Architectural reference for BuilderSensorWeather
---

# BuilderSensorWeather

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderSensorWeather extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorWeather class is a factory component within the server-side NPC Behavior System. Its sole responsibility is to parse a specific block of a JSON configuration file and construct a runtime SensorWeather object. This class acts as a bridge between declarative game data (assets) and imperative game logic (the sensor object).

It is part of a larger, reflection-based system where different builder classes are responsible for constructing pieces of an NPC's behavior tree. This specific builder translates a list of weather asset names or glob patterns into a functional sensor that can be evaluated by the NPC's decision-making logic during gameplay. The parent class, BuilderSensorBase, likely provides foundational hooks into the NPC sensor system.

## Lifecycle & Ownership
The lifecycle of a BuilderSensorWeather instance is extremely brief and tied directly to the server's asset loading phase.

-   **Creation:** An instance is created by the NPC asset loading service when it encounters the corresponding key for this sensor type within an NPC's behavior definition file (JSON). It is never instantiated directly by game logic developers.
-   **Scope:** The object exists only for the duration of parsing its specific JSON block and building the corresponding SensorWeather object. It is a short-lived, single-purpose object.
-   **Destruction:** The instance is abandoned and becomes eligible for garbage collection immediately after the `build` method is called and its result (the SensorWeather object) is integrated into the larger NPC behavior asset. It holds no persistent state and is not retained.

## Internal State & Concurrency
-   **State:** The class is stateful but effectively write-once. Its primary internal state is the `weathers` field, an AssetArrayHolder. This field is populated exactly once by the `readConfig` method. After this initialization, the state should be considered immutable for the remainder of its short lifecycle.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and used within a single-threaded asset loading context. Concurrent calls to `readConfig` on the same instance would result in a race condition and corrupt the internal `weathers` collection. This is a safe design assumption given its intended lifecycle.

## API Surface
The public API is designed for use by the asset loading system, not for general-purpose invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns the runtime SensorWeather object. This is the final step in the builder's lifecycle. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the input JSON, validates the weather asset references, and populates the internal state. N is the number of weather patterns. |
| getWeathers(BuilderSupport) | String[] | O(N) | Resolves and returns the configured weather patterns from the internal asset holder. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling or debugging. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling or debugging. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the engine's asset pipeline. A game designer defines the sensor in a JSON file, and the engine uses this builder to realize it. A developer would never interact with this class directly.

The conceptual flow within the engine is as follows:

```java
// PSEUDO-CODE: Engine's internal asset loading logic
JsonElement sensorConfig = parseNpcBehaviorFile(".../behavior.json");

// The engine identifies the correct builder and instantiates it
BuilderSensorWeather builder = new BuilderSensorWeather();

// The engine configures the builder from the JSON data
builder.readConfig(sensorConfig);

// The engine builds the final runtime object
Sensor runtimeSensor = builder.build(engine.getBuilderSupport());

// The builder instance is now discarded
// The runtimeSensor is attached to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance via `new BuilderSensorWeather()` in game logic. The asset system manages the lifecycle.
-   **State Re-use:** Do not call `readConfig` more than once on a single instance. The internal state is not designed to be cleared or re-initialized.
-   **Post-Build Access:** Do not hold a reference to the builder after the `build` method has been called. Its purpose is complete, and it should be considered a dead object.

## Data Pipeline
The primary function of this class is to process data from a configuration file into a live game object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderSensorWeather.readConfig** -> Internal AssetArrayHolder State -> **BuilderSensorWeather.build** -> Live SensorWeather Object -> NPC Behavior Tree

