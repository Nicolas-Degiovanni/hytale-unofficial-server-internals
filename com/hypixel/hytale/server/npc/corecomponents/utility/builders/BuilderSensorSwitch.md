---
description: Architectural reference for BuilderSensorSwitch
---

# BuilderSensorSwitch

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorSwitch extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorSwitch is a component within the server-side NPC asset pipeline. It functions as a specialized factory, responsible for translating a declarative JSON configuration into a concrete runtime object, specifically the SensorSwitch.

In the Hytale NPC AI framework, a Sensor is a node in the AI's logic graph that perceives the game state. It answers a true or false question, which then influences the NPC's behavior. This particular builder creates a Sensor that checks the value of a pre-computed boolean, effectively acting as a simple flag or switch in the AI's decision-making process.

Its primary role is to bridge the gap between static data (JSON assets) and live game logic (Sensor objects). It is instantiated by a higher-level asset management system during the NPC loading phase, consumes a specific JSON block, and produces a fully configured SensorSwitch instance ready for integration into the NPC's runtime behavior controller.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC asset loading system when it encounters a "SensorSwitch" type definition within an NPC's JSON configuration file. It is never created directly by game logic code.
-   **Scope:** The lifecycle of a BuilderSensorSwitch instance is extremely short. It exists only for the duration of parsing a single JSON object and building one SensorSwitch.
-   **Destruction:** The builder becomes eligible for garbage collection immediately after the `build` method is called and the resulting SensorSwitch object is returned to the asset loader. It holds no persistent references and is not managed by any registry.

## Internal State & Concurrency
-   **State:** This class is stateful and mutable. Its internal field, `switchHolder`, stores the configuration parsed from the `readConfig` method. This state is essential for the subsequent `build` operation but is specific to a single build process.

-   **Thread Safety:** BuilderSensorSwitch is **not thread-safe**. It is designed to be used exclusively within the single-threaded context of the asset loading pipeline. Concurrent calls to `readConfig` or `build` on the same instance will lead to race conditions and unpredictable behavior. Each configuration block must be processed by a new, distinct builder instance.

## API Surface
The public API is designed for a sequential, single-pass build process orchestrated by the asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorSwitch | O(1) | Terminal operation. Constructs and returns the final SensorSwitch object using the internal state. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Populates the builder's internal state from a JSON data source. Must be called before `build`. |
| getSwitch(BuilderSupport) | boolean | O(1) | Retrieves the resolved boolean value. This is typically called by the SensorSwitch object it creates, not externally. |
| getShortDescription() | String | O(1) | Provides metadata for tooling and debugging. |
| registerTags(Set<String>) | void | O(1) | Registers metadata tags (e.g., "logic") with the asset system for categorization. |

## Integration Patterns

### Standard Usage
This class is intended to be used exclusively by the server's asset loading framework. The framework identifies the correct builder for a given JSON block, instantiates it, configures it, and builds the final product.

```java
// Conceptual example within an asset loader
JsonElement sensorConfig = getSensorJsonFromAsset();
BuilderSensorSwitch builder = new BuilderSensorSwitch();

// 1. Configure the builder from the data asset
builder.readConfig(sensorConfig);

// 2. Build the final runtime object
BuilderSupport support = getBuilderSupportFromContext();
SensorSwitch runtimeSensor = builder.build(support);

// 3. Integrate the sensor into the NPC's AI
npc.getAIController().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not reuse a BuilderSensorSwitch instance to build multiple sensors. Its internal state is specific to the first `readConfig` call. A new builder must be created for each distinct sensor configuration.
-   **Calling `build` Before `readConfig`:** Invoking `build` on a fresh instance without first calling `readConfig` will produce a misconfigured SensorSwitch that relies on default values, leading to incorrect AI behavior.
-   **Direct State Manipulation:** Do not attempt to access or modify the internal `switchHolder` field directly. The public API contract must be respected.

## Data Pipeline
The BuilderSensorSwitch is a key transformation step in the NPC asset-to-runtime pipeline. It converts a static data definition into an executable logic component.

> Flow:
> NPC JSON Asset File -> JSON Parser -> **BuilderSensorSwitch.readConfig()** -> **BuilderSensorSwitch.build()** -> SensorSwitch Instance -> NPC Behavior Controller

