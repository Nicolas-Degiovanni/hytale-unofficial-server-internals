---
description: Architectural reference for BuilderSensorBlockChange
---

# BuilderSensorBlockChange

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorBlockChange extends BuilderSensorEvent {
```

## Architecture & Concepts
The BuilderSensorBlockChange class is a factory component within the NPC behavior definition system. Its sole responsibility is to translate a declarative JSON configuration into a live, executable Sensor object. It acts as a bridge between static game assets (NPC definition files) and the runtime NPC entity.

This builder is part of a larger ecosystem of builders, each responsible for constructing a specific piece of an NPC's logic, such as sensors, instructions, or goals. It encapsulates the complex logic of parsing configuration, validating asset references, and instantiating the final runtime component, in this case, a SensorBlockChange. This pattern decouples the data representation (JSON) from the runtime implementation (the Sensor object), allowing for flexible and data-driven NPC design.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorBlockChange is created by a higher-level asset parsing system when it encounters the corresponding component type within an NPC's JSON definition file. It is not intended for manual instantiation.
- **Scope:** The object's lifetime is ephemeral. It exists only during the NPC's asset loading and behavior tree construction phase.
- **Destruction:** Once the build method is called and the resulting Sensor object is integrated into the NPC's runtime components, the builder instance has served its purpose and is eligible for garbage collection. It does not persist in the game state.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The readConfig method populates the internal AssetHolder and EnumHolder fields based on the provided JSON data. This state is temporary and serves as a blueprint for the final Sensor object.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used in a single-threaded context during the server's asset loading phase. Accessing a single instance from multiple threads, especially calls to readConfig, will result in race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorBlockChange instance using the previously loaded configuration. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the provided JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| getBlockSet(BuilderSupport) | int | O(1) | Resolves the configured BlockSet asset key into its runtime integer index. Throws IllegalArgumentException if the asset key is not found. |
| getEventType(BuilderSupport) | BlockEventType | O(1) | Retrieves the configured BlockEventType enum. |

## Integration Patterns

### Standard Usage
The builder is used by the engine's asset loading system. The typical sequence is to instantiate the builder, configure it from a data source, and then build the final runtime object.

```java
// Conceptual example within the asset parsing system
BuilderSensorBlockChange builder = new BuilderSensorBlockChange();
JsonElement configData = parseNpcDefinitionFile("my_npc.json");

// The builder is configured and then immediately used
builder.readConfig(configData.getAsJsonObject("sensor"));
Sensor runtimeSensor = builder.build(builderSupportContext);

// The resulting sensor is then attached to the NPC
npc.getBehaviorSystem().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not reuse a single builder instance to configure and build multiple, different sensors. Each builder should be treated as a single-use factory for one specific component configuration.
- **Incomplete Initialization:** Calling build before readConfig has been successfully invoked will result in a misconfigured or non-functional Sensor, likely causing runtime exceptions.
- **Direct State Manipulation:** The internal AssetHolder and EnumHolder fields should not be accessed or modified directly. Only interact with the builder through its public API (readConfig and build).

## Data Pipeline
The BuilderSensorBlockChange is a key transformation step in the NPC data pipeline, converting static data into a runtime object.

> Flow:
> NPC Definition (JSON File) -> Engine Asset Parser -> **BuilderSensorBlockChange**.readConfig() -> **BuilderSensorBlockChange**.build() -> SensorBlockChange (Runtime Object) -> NPC Behavior System

