---
description: Architectural reference for BuilderSensorHasInteracted
---

# BuilderSensorHasInteracted

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorHasInteracted extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorHasInteracted class is a specialized factory component within the server-side NPC Behavior Asset Pipeline. Its sole responsibility is to parse a specific JSON configuration block and construct an executable `SensorHasInteracted` object.

This builder acts as a deserialization and validation bridge between declarative NPC behavior definitions (stored in JSON files) and the runtime NPC AI system. It ensures that the `SensorHasInteracted` component is only used within its intended context—an **Interaction** instruction. This is enforced by the `requireInstructionType` check, which acts as a schema validator at the asset loading stage, preventing invalid AI configurations from ever reaching the runtime.

It is one of many builders managed by a higher-level asset loading service, which selects the correct builder based on a type identifier in the source JSON data.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading system when it encounters the corresponding sensor type identifier within an NPC's JSON definition file.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single sensor definition. It is a transient object used during the asset deserialization phase.
- **Destruction:** The object is eligible for garbage collection immediately after the `build` method returns the final `SensorHasInteracted` instance. It holds no references and is not retained by any system.

## Internal State & Concurrency
- **State:** This class is effectively stateless. While the `readConfig` method processes input, this state is immediately consumed by the `build` method. It does not cache data or maintain any state across multiple invocations.

- **Thread Safety:** **Not Thread-Safe.** Instances of this builder are designed to be confined to a single thread during the asset loading process. Do not share instances of this builder across threads. The parent asset loading system is responsible for managing the threading model for NPC compilation.

## API Surface
The public API is designed for consumption by the asset loading framework, not for general-purpose game logic development.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorHasInteracted instance. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses JSON data and validates that the sensor is within an Interaction instruction. Throws an exception on validation failure. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling and editors. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component, indicating it is production-ready. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the NPC asset loading system. The conceptual flow within the engine is as follows:

```java
// Conceptual example of engine-level usage
// Engine code identifies the builder for "hytale:sensor_has_interacted"
Builder<Sensor> builder = assetLoader.getBuilderFor("hytale:sensor_has_interacted");

// Engine provides the JSON data for this specific sensor
builder.readConfig(sensorJsonData);

// The builder creates the final runtime object
Sensor sensor = builder.build(builderSupport);

// The sensor is then attached to the NPC's behavior tree
npcBehaviorTree.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderSensorHasInteracted()`. The NPC asset system manages the lifecycle of builders. Manually creating one bypasses the entire asset pipeline.
- **State Re-use:** Do not hold a reference to this builder and attempt to call `build` multiple times. Each builder instance is meant for a single, one-shot creation process.

## Data Pipeline
This builder is a key transformation step in the NPC behavior data pipeline. It converts declarative configuration data into an executable object.

> Flow:
> NPC Definition (*.json) -> JSON Parser -> **BuilderSensorHasInteracted** -> SensorHasInteracted (Object) -> NPC Behavior Tree

