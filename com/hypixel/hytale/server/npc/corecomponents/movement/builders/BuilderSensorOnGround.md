---
description: Architectural reference for BuilderSensorOnGround
---

# BuilderSensorOnGround

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorOnGround extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorOnGround class is a factory component within the server-side NPC Asset system. Its sole responsibility is to construct an instance of SensorOnGround during the deserialization of an NPC's behavior definition from a configuration source, typically a JSON file.

This class adheres to the **Builder pattern**, but in the context of a data-driven architecture. It acts as a concrete implementation for a specific type of Sensor, translating a configuration block into a live, executable game object. It is discovered and invoked by a higher-level asset management service, which maps type identifiers in the data files to their corresponding builder classes.

Its existence allows the core NPC system to remain decoupled from the specific implementations of sensors. The system interacts with the generic Builder interface, while this class provides the specific logic for the "OnGround" condition. The `readConfig` method, despite being a no-op in this implementation, is a critical part of the contract, allowing other, more complex builders to parse parameters from the JSON data.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset loading pipeline when it encounters a sensor definition of type "OnGround" in an NPC configuration file. It is never created directly by game logic code.
- **Scope:** Extremely short-lived. An instance of BuilderSensorOnGround exists only for the duration of the parsing and building of a single SensorOnGround object.
- **Destruction:** The builder instance is eligible for garbage collection immediately after the `build` method has been called and its result (the SensorOnGround object) has been integrated into the parent NPC asset. It holds no persistent references and is not managed or cached.

## Internal State & Concurrency
- **State:** This object is **stateless and immutable**. It contains no fields and its behavior is fixed. The `readConfig` method does not modify any internal state, reinforcing its role as a simple, configuration-less factory.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, multiple threads can safely use separate instances of this builder to construct SensorOnGround objects concurrently during a multi-threaded asset loading process without any risk of interference or race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorOnGround | O(1) | Constructs and returns a new SensorOnGround instance. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description for tooling. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the development status of this component, indicating it is stable. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Fulfills the configuration contract. This implementation is a no-op. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked by the NPC asset system. The following example illustrates a conceptual representation of how the system would utilize this builder.

```java
// Hypothetical asset loading service
// The service maps a string "sensor.on_ground" to the BuilderSensorOnGround.class
Builder<Sensor> builder = assetService.getBuilderFor("sensor.on_ground");

// The service would pass the relevant JSON data, which is ignored by this specific builder
builder.readConfig(sensorJsonData);

// The final sensor object is built and attached to an NPC
SensorOnGround sensor = (SensorOnGround) builder.build(builderSupport);
npc.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorOnGround()` in game logic. The construction of NPC components must be driven by the asset configuration pipeline to ensure the NPC's behavior matches its definition file.
- **Extending for Game Logic:** Do not extend this class to add custom game logic. Custom sensors should have their own dedicated builder classes and be registered with the asset system.

## Data Pipeline
The data flow for this component is part of the larger NPC asset deserialization process.

> Flow:
> NPC Definition (JSON File) -> Asset Deserializer -> **BuilderSensorOnGround** -> `build()` -> SensorOnGround (Runtime Object) -> NPC Behavior Tree

