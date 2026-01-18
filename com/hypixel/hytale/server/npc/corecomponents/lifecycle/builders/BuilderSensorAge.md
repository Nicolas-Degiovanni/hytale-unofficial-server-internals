---
description: Architectural reference for BuilderSensorAge
---

# BuilderSensorAge

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorAge extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorAge class is a factory component within the server-side Non-Player Character (NPC) asset pipeline. Its primary responsibility is to parse a JSON configuration block and construct a runtime SensorAge object. It acts as a bridge between the static data definition of an NPC (the JSON asset) and its live, in-world behavior.

This class embodies the separation of configuration from execution. It handles the concerns of data parsing, validation, and translation, ensuring that the resulting SensorAge object receives a well-formed, absolute time range to check against.

During server startup or a hot-reload of NPC assets, a master parser will identify a sensor of this type and delegate the corresponding JSON element to an instance of BuilderSensorAge. The builder then transforms a human-readable temporal range (e.g., "between 1 and 5 game days") into a pair of absolute Instants, calculated relative to the specific NPC's spawn time. This pre-calculation is critical for performance, as the runtime SensorAge can then perform simple, efficient timestamp comparisons during each game tick.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset loading service (e.g., an NpcBehaviorFactory) when it encounters an "age" sensor definition within an NPC's JSON file. Developers should never instantiate this class directly.
- **Scope:** Short-lived and transient. An instance of BuilderSensorAge exists only for the duration of parsing a single sensor definition. Its lifecycle is tied to the asset loading process, not the game loop.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method has been called and the resulting SensorAge instance has been integrated into the NPC's runtime behavior tree. It holds no persistent references and is not managed by any service container.

## Internal State & Concurrency
- **State:** This class is **mutable**. Its primary internal state is the `ageRange` field, which is populated by the `readConfig` method. This state is essential for the subsequent `build` and `getAgeRange` calls.

- **Thread Safety:** BuilderSensorAge is **not thread-safe** and must not be shared across threads. The class is designed to be instantiated, configured via `readConfig`, and used to `build` a sensor in a single, sequential operation. Concurrent access would lead to race conditions and unpredictable behavior, as one thread could be reading the `ageRange` state while another is writing to it.

## API Surface
The public API is designed for a single, linear workflow: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Sensor> | O(1) | Parses the JSON definition, validates the temporal range, and populates internal state. Throws exceptions on invalid format. |
| build(BuilderSupport support) | Sensor | O(1) | Constructs and returns a new SensorAge instance using the previously configured state. |
| getAgeRange(BuilderSupport support) | Instant[] | O(1) | Calculates and returns the absolute start and end Instants for the age check, relative to the NPC's spawn time. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling and editors. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling and editors. |

## Integration Patterns

### Standard Usage
This class is intended for internal use by the NPC asset loading system. The pattern is to instantiate, configure, and build in immediate succession.

```java
// Hypothetical usage within an asset loader
JsonElement sensorConfig = ... // "AgeRange": ["P1D", "P5D"]
BuilderSupport support = ... // Context for the specific NPC entity

BuilderSensorAge builder = new BuilderSensorAge();
builder.readConfig(sensorConfig);
SensorAge runtimeSensor = (SensorAge) builder.build(support);

// The runtimeSensor is now attached to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not reuse a BuilderSensorAge instance to build multiple sensors. Its internal state is tied to a single `readConfig` call. Create a new instance for each sensor definition.
- **Premature Build:** Do not call `build` before `readConfig` has been successfully invoked. This will result in an improperly configured sensor that will likely throw a NullPointerException or fail validation.
- **Direct State Manipulation:** Do not attempt to access or modify the internal `ageRange` field directly. Rely exclusively on the `readConfig` method to ensure proper validation logic is applied.

## Data Pipeline
The BuilderSensorAge is a key transformation step in the NPC data pipeline, converting declarative configuration into an executable object.

> Flow:
> NPC Asset (JSON file) -> Asset Loading Service -> **BuilderSensorAge.readConfig()** -> **BuilderSensorAge.build()** -> SensorAge (Runtime Object) -> NPC Behavior Tree Tick Evaluation

