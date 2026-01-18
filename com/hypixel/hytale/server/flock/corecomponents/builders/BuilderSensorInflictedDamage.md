---
description: Architectural reference for BuilderSensorInflictedDamage
---

# BuilderSensorInflictedDamage

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorInflictedDamage extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorInflictedDamage class is a key component in the server-side NPC asset pipeline. It functions as a concrete implementation of the Builder pattern, responsible for translating a declarative JSON configuration into a runtime AI object.

Its specific role is to construct a SensorInflictedDamage instance. In Hytale's AI engine, a Sensor is a component within an NPC's behavior tree that evaluates a world-state condition to return true or false, guiding decision-making. This particular sensor answers the question: "Have I, or my flock, recently dealt combat damage?".

This builder acts as the bridge between the static data defined in JSON asset files and the live, executable AI system. During server startup or asset hot-reloading, a central asset manager parses NPC behavior files. When it encounters a sensor of type *InflictedDamage*, it instantiates this builder, populates it using the `readConfig` method, and then calls `build` to produce the final, configured Sensor object which is then integrated into the NPC's behavior tree.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level NPC asset factory or parser during the deserialization of an NPC's behavior definition from a JSON file. It is never created directly by game logic.
-   **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single sensor definition within a JSON file.
-   **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and the resulting SensorInflictedDamage object has been returned to the calling factory. It holds no persistent state and is not retained.

## Internal State & Concurrency
-   **State:** The internal state is **Mutable**. The `target` and `friendlyFire` fields are directly modified by the `readConfig` method based on the input JSON. This state is a temporary container for configuration parameters that are ultimately passed to the SensorInflictedDamage constructor.

-   **Thread Safety:** This class is **Not Thread-Safe** and must not be shared across threads. It is designed to be instantiated, configured, and used by a single thread within the asset loading subsystem. Concurrent access to an instance, especially calls to `readConfig`, would result in a race condition and unpredictable sensor configuration.

## API Surface
The public contract is focused on configuration and construction. Metadata methods like `getShortDescription` are for tooling and editor support.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorInflictedDamage | O(1) | Constructs and returns the final SensorInflictedDamage object using the internal state. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the provided JSON, populates the internal configuration state, and returns itself for chaining. |

## Integration Patterns

### Standard Usage
This builder is used exclusively by the internal asset loading system. A developer defining NPC behavior in JSON is an indirect user. The typical internal flow is sequential: instantiate, configure, build.

```java
// Hypothetical usage within an NPC Asset Factory
JsonElement sensorConfig = parseNpcJsonFile(".../my_npc.json");

BuilderSensorInflictedDamage builder = new BuilderSensorInflictedDamage();
builder.readConfig(sensorConfig);
SensorInflictedDamage sensor = builder.build(builderSupport);

// The 'sensor' object is now added to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse a builder instance to configure multiple sensors. The internal state will persist from the first configuration, leading to incorrect behavior for subsequent builds. Always create a new builder for each sensor definition.
-   **Build Before Configure:** Calling `build` on a newly instantiated builder without first calling `readConfig` will produce a sensor with default values (e.g., Target.Self, friendlyFire=false), which will likely not match the intended behavior from the asset file.
-   **Direct Instantiation in Game Logic:** This is a configuration-time class. It should never be instantiated or used as part of the real-time game loop.

## Data Pipeline
This class operates within a configuration and asset ingestion pipeline, not a real-time game data pipeline.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> JSON Parser -> **BuilderSensorInflictedDamage.readConfig()** -> **BuilderSensorInflictedDamage.build()** -> Live SensorInflictedDamage Object -> NPC Behavior Tree

