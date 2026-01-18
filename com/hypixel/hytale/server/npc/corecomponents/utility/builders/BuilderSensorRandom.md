---
description: Architectural reference for BuilderSensorRandom
---

# BuilderSensorRandom

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorRandom extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorRandom class is a factory component within the server-side NPC asset pipeline. Its sole responsibility is to parse a declarative JSON configuration and construct a runtime SensorRandom instance. It serves as the bridge between static asset definitions on disk and live, executable sensor logic used by an NPC's AI.

This builder encapsulates the validation and configuration logic for the SensorRandom component. By processing the JSON configuration, it populates its internal state with the specified time ranges. The builder instance is then passed directly to the SensorRandom constructor, where it acts as a long-lived data source for the sensor it creates. This pattern decouples the runtime sensor logic from the complexities of asset parsing and validation.

## Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the NPC asset loading system when an NPC definition file contains a sensor of type *Random*. This class is never intended for manual instantiation by developers.
-   **Scope:** The builder's lifecycle is unexpectedly long. While it is created during the asset loading phase, the SensorRandom object it builds maintains a direct reference to it. Therefore, the builder persists in memory for the entire lifetime of the NPC that uses the resulting sensor. It effectively becomes a read-only configuration object for its product.
-   **Destruction:** The builder becomes eligible for garbage collection only when the SensorRandom instance, and by extension the parent NPC, is destroyed.

## Internal State & Concurrency
-   **State:** The internal state is mutable during the configuration phase. The fields falseRange and trueRange are populated by the readConfig method. After the build method is called, the state should be considered effectively immutable, as it is only read by the created SensorRandom object.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, configured, and used within the single-threaded context of the asset loading pipeline. Concurrent calls to readConfig would corrupt its internal state.

## API Surface
The public API is primarily for internal use by the asset system and the SensorRandom object it constructs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorRandom instance. This is the final step in the builder's primary lifecycle. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the input JSON, validates, and populates the internal duration range fields. Throws exceptions on invalid or missing data. |
| getFalseRange(BuilderSupport) | double[] | O(1) | Accessor for the configured false-state duration range. Intended to be called by the created SensorRandom object. |
| getTrueRange(BuilderSupport) | double[] | O(1) | Accessor for the configured true-state duration range. Intended to be called by the created SensorRandom object. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is not a standard development pattern. Instead, developers define the sensor declaratively within an NPC's JSON asset file. The engine's asset loader handles the instantiation and configuration of the builder behind the scenes.

A conceptual example of how the *engine* uses this class:

```java
// This code is conceptual and executed by the asset loader.
// Do not replicate this pattern in game logic.
JsonElement sensorConfig = parseNpcAssetFile(".../my_npc.json");

BuilderSensorRandom builder = new BuilderSensorRandom();
builder.readConfig(sensorConfig);
Sensor sensor = builder.build(builderSupport);

// The 'sensor' is now a fully configured SensorRandom instance.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new BuilderSensorRandom() in game or system logic. The asset pipeline is the sole owner of this class's lifecycle.
-   **Calling build Before readConfig:** Invoking the build method on a non-configured builder will produce a SensorRandom instance with default (likely zero) duration ranges, leading to invalid AI behavior.
-   **Modifying After Build:** Although not prevented by the API, modifying a builder's state after it has been used to construct a sensor is an anti-pattern. The created sensor holds a reference to the builder, and subsequent state changes would unpredictably alter the sensor's runtime behavior.

## Data Pipeline
The BuilderSensorRandom acts as a transformation stage in the data pipeline that converts static data into a runtime AI component.

> Flow:
> NPC Asset JSON File -> Engine JSON Parser -> Asset Loading Service -> **BuilderSensorRandom.readConfig()** -> **BuilderSensorRandom.build()** -> Live SensorRandom Instance -> NPC Behavior Tree Execution

