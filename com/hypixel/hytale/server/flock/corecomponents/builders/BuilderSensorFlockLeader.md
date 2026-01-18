---
description: Architectural reference for BuilderSensorFlockLeader
---

# BuilderSensorFlockLeader

**Package:** com.hypixel.hytale.server.flock.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorFlockLeader extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorFlockLeader is a factory class that operates within the server-side NPC asset pipeline. Its sole responsibility is to deserialize a JSON configuration snippet and construct a concrete SensorFlockLeader instance. This class embodies the Builder pattern, providing a clear separation between the complex process of object construction and its final representation.

In Hytale's AI architecture, a Sensor is a component that allows an NPC to perceive and react to its environment. This specific builder creates a sensor that enables a flocking entity (e.g., a bird, a fish) to identify and track its designated leader. The builder acts as the translation layer between the declarative data format (JSON) used by designers and the imperative Java code that runs on the server.

The call to `provideFeature(Feature.LiveEntity)` during configuration is critical. It signals to the parent asset loading system that any NPC using this sensor has a hard dependency on being a live, in-world entity, allowing the system to perform validation and resource allocation accordingly.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's NPC asset loading system when it encounters a sensor of this type within an NPC's JSON definition file. It is never created manually by game logic.
- **Scope:** Ephemeral and short-lived. An instance of this builder exists only for the duration of parsing a single sensor definition and is discarded immediately after the `build` method is invoked.
- **Destruction:** The object becomes eligible for garbage collection as soon as the fully configured SensorFlockLeader is returned. It holds no persistent references and requires no manual cleanup.

## Internal State & Concurrency
- **State:** Mutable. The builder's internal state is configured by the `readConfig` method. This state, primarily managed by the parent BuilderSensorBase, is used to parameterize the SensorFlockLeader object during construction.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be used in a single-threaded context during asset loading. Sharing a builder instance across multiple threads will result in unpredictable behavior and corrupted Sensor components. The asset pipeline must guarantee that each builder is confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorFlockLeader | O(1) | Constructs the final SensorFlockLeader instance using the state configured by `readConfig`. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the JSON data, configures the builder's internal state, and registers required engine features. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay code. It is exclusively invoked by the internal asset loading framework. The following pseudocode illustrates its intended lifecycle.

```java
// Hypothetical usage within an NPC Asset Loader
BuilderSensorFlockLeader builder = new BuilderSensorFlockLeader();

// Configure the builder from the NPC's JSON file
builder.readConfig(sensorJsonDefinition);

// Construct the final sensor object
SensorFlockLeader sensor = builder.build(builderSupport);

// Attach the sensor to the NPC's AI component set
npc.getAI().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to cache and re-use a BuilderSensorFlockLeader instance to build multiple sensors. Each sensor definition from a configuration file must be processed by a new, clean builder instance.
- **Premature Building:** Calling `build` before `readConfig` will result in a default, unconfigured sensor. This will likely lead to severe runtime errors or undetectable logic flaws in NPC behavior.
- **Manual Instantiation:** Game logic should never instantiate this class directly using `new`. The NPC's behavior should be defined entirely within data files, which are then processed by the asset system.

## Data Pipeline
The builder is a key transformation step in the data flow from disk to a live game object.

> Flow:
> NPC Definition (JSON file) -> Server Asset Deserializer -> **BuilderSensorFlockLeader** -> SensorFlockLeader Instance -> Live NPC AI Component

