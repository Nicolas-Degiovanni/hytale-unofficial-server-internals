---
description: Architectural reference for BuilderSensorAny
---

# BuilderSensorAny

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorAny extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorAny class is a specialized factory component within the server-side NPC (Non-Player Character) asset pipeline. Its sole responsibility is to construct an instance of SensorAny, a logical sensor that provides a constant *true* signal to the NPC's behavior system.

This component acts as a bridge between declarative NPC configuration (typically defined in JSON files) and the runtime behavior objects. In the context of a Behavior Tree or a Finite State Machine, the sensor produced by this builder serves as an unconditional trigger. It allows designers to create logic that always executes a particular branch, acting as a default case or an entry point for a sequence of actions without requiring any environmental perception.

It is a leaf node in the builder hierarchy, representing one of the simplest possible sensor configurations. Its behavior is static and not influenced by JSON configuration data, as evidenced by its implementation of readConfig.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset management system, such as an NPC behavior deserializer, when it encounters the corresponding type identifier in an NPC definition file.
- **Scope:** Ephemeral and short-lived. An instance of BuilderSensorAny exists only for the duration of the NPC asset parsing and construction phase.
- **Destruction:** The builder object is eligible for garbage collection immediately after its build method has been invoked and the resulting Sensor object has been integrated into the NPC's runtime component graph. It holds no persistent references and is not managed by any registry.

## Internal State & Concurrency
- **State:** The class inherits state from its parent, BuilderSensorBase, which includes fields like the boolean *once*. However, BuilderSensorAny introduces no new state of its own. Its readConfig method is a no-op, meaning its internal state cannot be mutated by external configuration data. The state is determined at instantiation and is effectively immutable from a configuration perspective.

- **Thread Safety:** **This class is not thread-safe.** Builder objects are designed to be used exclusively within a single-threaded asset loading context. Do not share instances of this builder across threads, as the state inherited from its parent class is not protected against concurrent access.

## API Surface
The public API is minimal, focusing entirely on the factory pattern and metadata retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs a new SensorAny instance or returns Sensor.NULL based on internal state. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | A no-op method that ignores the provided JSON data and returns the current instance. This signifies that the component is not configurable. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for tooling and editors. |
| registerTags(Set<String>) | void | O(1) | Registers metadata tags, such as "logic", used for categorization and filtering in development tools. |

## Integration Patterns

### Standard Usage
This builder is not intended for direct use by game logic developers. It is invoked transparently by the NPC asset loading system. A simplified conceptual example of its internal usage is shown below.

```java
// Hypothetical asset loader context
// The loader identifies the type and creates the correct builder.
BuilderSensorBase builder = assetRegistry.createBuilderFor("SensorAny");

// The configuration step is called, though it does nothing for this type.
builder.readConfig(sensorJsonData);

// The final runtime object is built and attached to the NPC.
Sensor runtimeSensor = builder.build(builderSupport);
npc.getBehaviorController().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderSensorAny()`. The NPC asset system is responsible for the lifecycle of builder objects. Direct creation bypasses the asset pipeline and can lead to uninitialized or disconnected components.
- **Reusing Instances:** Do not retain and reuse builder instances. They are intended to be single-use, transient objects.
- **Providing Configuration:** Attempting to configure a BuilderSensorAny via JSON is futile. The readConfig method intentionally ignores all input, and relying on it to have an effect will lead to bugs.

## Data Pipeline
BuilderSensorAny is a processing step in the transformation of static data into live game objects.

> Flow:
> NPC Definition (*.json*) -> JSON Deserializer -> **BuilderSensorAny** -> SensorAny Instance -> NPC Behavior Controller

