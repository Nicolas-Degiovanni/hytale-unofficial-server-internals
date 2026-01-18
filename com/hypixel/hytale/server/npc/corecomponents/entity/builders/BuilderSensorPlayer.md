---
description: Architectural reference for BuilderSensorPlayer
---

# BuilderSensorPlayer

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorPlayer extends BuilderSensorEntityBase {
```

## Architecture & Concepts
The BuilderSensorPlayer class is a factory component within the server-side NPC Behavior System. Its sole responsibility is to deserialize a JSON configuration object and construct a corresponding SensorPlayer instance.

This class embodies a critical design pattern in the Hytale engine: **data-driven design**. Instead of hard-coding NPC sensory logic, game designers define it in JSON asset files. A central asset loading system, upon encountering a sensor of type "Player", delegates the instantiation logic to this specific builder.

This approach decouples game logic (the SensorPlayer object) from its configuration (the JSON file), allowing for rapid iteration on NPC behaviors without requiring new engine code or server deployments. BuilderSensorPlayer acts as the translation layer between the declarative JSON data and the imperative Java object graph of a running NPC.

## Lifecycle & Ownership
- **Creation:** BuilderSensorPlayer instances are created on-demand by a higher-level factory or registry system, such as a BehaviorTreeLoader or an AssetManager. This process is typically triggered when an NPC's behavior asset is loaded from disk. The system identifies the sensor type from the JSON data and resolves it to this specific builder class.
- **Scope:** The lifecycle of a BuilderSensorPlayer instance is extremely short and confined to a single operation. It is created, configured via readConfig, used once to call build, and then immediately becomes eligible for garbage collection. It does not persist.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. Its transient nature ensures a minimal memory footprint.

## Internal State & Concurrency
- **State:** This class is stateful during its brief lifecycle. The readConfig method populates internal fields inherited from BuilderSensorEntityBase with values from the JSON data (e.g., sensor range, filters). This state is then used by the build method to construct the final SensorPlayer object. After the build method is called, the state is no longer relevant.
- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be instantiated and used within the scope of a single asset-loading task. The sequence of `readConfig` followed by `build` is not an atomic operation. Concurrent access would lead to unpredictable behavior and corrupted SensorPlayer objects. The parent asset loading system is responsible for ensuring that each thread processes assets with its own private builder instances.

## API Surface
The public API is minimal, exposing only the necessary methods for the builder pattern and metadata retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorPlayer | O(1) | Constructs and returns a new SensorPlayer instance using the previously loaded configuration. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the provided JSON, populating the builder's internal state. N is the number of keys. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the production readiness status of this component, e.g., Stable. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. It is invoked transparently by the NPC asset pipeline. The conceptual flow within the engine is as follows.

```java
// Conceptual example of how the engine uses this builder
JsonElement sensorConfig = loadNpcBehaviorAsset(".../behavior.json");
String sensorType = sensorConfig.get("type").getAsString(); // e.g., "SensorPlayer"

// The registry finds the correct builder for the type
Builder<Sensor> builder = BuilderRegistry.getBuilderForType(sensorType);

// The builder is configured and used to create the final object
builder.readConfig(sensorConfig);
Sensor sensor = builder.build(builderSupport);

// The resulting sensor is attached to the NPC
npc.getBehaviorTree().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderSensorPlayer()`. The asset system is responsible for managing the lifecycle of builders. Manually creating one bypasses the registry and can lead to uninitialized or misconfigured components.
- **Instance Re-use:** Do not cache and re-use a BuilderSensorPlayer instance. They are designed to be single-use and are not guaranteed to correctly reset their internal state if `readConfig` is called multiple times.

## Data Pipeline
BuilderSensorPlayer is a single, critical step in the data pipeline that transforms a static asset file into a live game component.

> Flow:
> NPC_Behavior.json -> JSON Parser -> **BuilderSensorPlayer.readConfig()** -> **BuilderSensorPlayer.build()** -> SensorPlayer Instance -> NPC Behavior Tree

