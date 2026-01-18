---
description: Architectural reference for BuilderSensorState
---

# BuilderSensorState

**Package:** com.hypixel.hytale.server.npc.corecomponents.statememachine.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorState extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorState is a configuration-time component within the server's NPC asset loading framework. It acts as a specialized parser and factory for creating a runtime Sensor object. Its sole responsibility is to translate a JSON definition that describes a state-based condition into a functional SensorState instance.

In the Hytale NPC AI system, a Sensor is a node in a behavior tree that evaluates a condition and signals true or false, influencing the NPC's actions. The SensorState object specifically checks if an NPC is currently in a designated state, such as *attacking* or *fleeing*.

BuilderSensorState is the bridge between the static JSON asset file and the live, in-memory SensorState object. It is invoked by the asset loading system when it encounters a sensor of this type, populates its internal fields from the JSON data, and then constructs the final runtime object via its build method.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the core NPC asset loading system (BuilderSupport) when parsing an NPC's behavior definition from a JSON file. It is never created directly by game logic.
-   **Scope:** Extremely short-lived. An instance of BuilderSensorState exists only for the duration of parsing a single sensor definition.
-   **Destruction:** The object is eligible for garbage collection immediately after the build method is called and the resulting SensorState is integrated into the NPC's behavior asset. It holds no persistent references.

## Internal State & Concurrency
-   **State:** Highly mutable. The object's fields (state, subState, ignoreMissingSetState) serve as temporary storage for configuration values parsed from JSON. This state is transient and is consumed during the creation of the SensorState object.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. The asset loading pipeline, where this builder operates, is a sequential, single-threaded process. Concurrent calls to readConfig or build would result in a corrupted and unpredictable configuration.

## API Surface
The primary contract is for use by the asset building system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorState | O(1) | Constructs the final SensorState runtime object using the configuration parsed by readConfig. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the JSON configuration, populating the builder's internal state. This is the main entry point. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is used exclusively by the asset loading framework. The pattern is declarative, defined in a JSON asset.

A developer would define the sensor in JSON:
```json
{
  "type": "State",
  "State": "combat.attacking",
  "IgnoreMissingSetState": false
}
```

The system would then use the builder like this internally:
```java
// Conceptual example of internal system usage
BuilderSensorState builder = new BuilderSensorState();
builder.readConfig(jsonElementFromAsset);
SensorState sensor = builder.build(builderSupport);
// 'sensor' is now added to the NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BuilderSensorState()`. The class is useless without being driven by the asset loading system, which provides the JSON data and the BuilderSupport context.
-   **State Re-use:** Do not attempt to reuse a builder instance to create multiple SensorState objects. Each builder is a one-shot factory for a single configuration block.
-   **Manual Population:** Do not manually call setter methods on the builder. All state should be derived from the JSON configuration via the readConfig method to ensure consistency.

## Data Pipeline
BuilderSensorState is a component in the asset *configuration* pipeline, not a runtime data pipeline.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> Asset Loader -> **BuilderSensorState.readConfig()** -> **BuilderSensorState.build()** -> Live SensorState Object

