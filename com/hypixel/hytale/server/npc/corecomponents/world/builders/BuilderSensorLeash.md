---
description: Architectural reference for BuilderSensorLeash
---

# BuilderSensorLeash

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorLeash extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorLeash is a factory class within the server-side NPC asset system. It is not the runtime sensor itself, but rather the configuration-driven blueprint responsible for instantiating a SensorLeash component. Its primary role is to translate a JSON configuration block into a live, functional SensorLeash object that can be integrated into an NPC's behavior tree.

This class is a concrete implementation of the **Builder** pattern, a core design principle of the NPC component framework. Each builder is responsible for a single, specific type of component, providing a decoupled and extensible way to define complex NPC behaviors from data files.

The builder operates by consuming a JsonElement and populating its internal state. During the build phase, it uses a provided BuilderSupport context object to resolve dependencies, declare requirements, and construct the final SensorLeash instance.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorLeash is created by the central NPC asset loading service. This typically occurs via reflection when the service encounters a component definition of type "SensorLeash" within an NPC's JSON asset file.
- **Scope:** The object is extremely short-lived. Its lifecycle is confined to the parsing and construction of a single SensorLeash component for a single NPC definition. It does not persist into the game world.
- **Destruction:** The instance is eligible for garbage collection immediately after the `build` method completes and the resulting SensorLeash object has been returned to the asset loader.

## Internal State & Concurrency
- **State:** The class holds mutable state, specifically the `range` field. This state is uninitialized upon creation and is populated exclusively by the `readConfig` method. The use of a DoubleHolder for `range` indicates that the final value may be resolved dynamically at runtime via an ExecutionContext, allowing for variable-driven configurations.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. The entire NPC asset loading pipeline is designed to be a single-threaded, synchronous process. Invoking its methods from multiple threads will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorLeash | O(1) | Constructs and returns the runtime SensorLeash instance. Critically, this method registers a `leashPosition` requirement with the BuilderSupport context. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Parses the JSON data, validates the "Range" property, and populates the internal state. Throws a configuration error if the data is invalid. |
| getRange(BuilderSupport) | double | O(1) | Resolves and returns the configured leash range. Requires a BuilderSupport context to resolve the potential dynamic value from the DoubleHolder. |

## Integration Patterns

### Standard Usage
The BuilderSensorLeash is not intended to be used directly by developers. It is invoked automatically by the NPC asset framework during server startup or asset reloading. The conceptual flow is managed entirely by the framework.

```java
// Conceptual example of framework-level usage
// This code does not exist in this form; it illustrates the process.

// 1. Framework instantiates the builder for a "SensorLeash" JSON block
BuilderSensorLeash builder = new BuilderSensorLeash();

// 2. Framework provides the JSON data to configure the builder
builder.readConfig(sensorJsonData);

// 3. Framework provides a context and requests the final object
BuilderSupport support = new BuilderSupport(executionContext);
SensorLeash runtimeSensor = builder.build(support);

// 4. The resulting sensor is attached to an NPC's behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderSensorLeash()`. The NPC asset system manages the lifecycle of all builder objects. Manually creating one circumvents the asset loading pipeline.
- **Calling build Before readConfig:** The internal state is uninitialized before `readConfig` is called. Invoking `build` prematurely will result in an improperly configured sensor and likely throw a runtime exception.
- **State Reuse:** Do not attempt to reuse a builder instance to configure multiple sensors. Each builder is single-use and should be discarded after one `build` cycle.

## Data Pipeline
The class functions as a transformation step in the NPC asset loading pipeline, converting declarative data into an executable game object.

> Flow:
> NPC_Asset.json -> Asset Loading Service -> **BuilderSensorLeash.readConfig()** -> **BuilderSensorLeash.build()** -> SensorLeash Instance -> NPC Behavior Tree

