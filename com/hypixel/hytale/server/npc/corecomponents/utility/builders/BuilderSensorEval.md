---
description: Architectural reference for BuilderSensorEval
---

# BuilderSensorEval

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorEval extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorEval class is a factory component within the server's NPC asset loading framework. It embodies the Builder design pattern, serving the specific purpose of deserializing a JSON configuration block into a concrete SensorEval instance.

Its primary role is to act as a bridge between the data-driven NPC definitions (stored in JSON files) and the live, in-game NPC behavior system. When the server loads an NPC's behavior asset, a master parser delegates the construction of individual components, like sensors, to specialized builders. BuilderSensorEval is responsible for handling sensor definitions that rely on JavaScript expression evaluation. It extracts the expression string from the JSON data and uses it to configure and construct a fully operational SensorEval object, which is then integrated into the NPC's behavior tree.

This class is fundamental to enabling dynamic and data-driven NPC logic, allowing designers to specify complex conditional behaviors in external files without modifying core engine code.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset factory or parser during the NPC asset loading sequence. The specific builder to use is typically determined by a "type" field within the JSON definition. Manual instantiation is not a supported use case.
- **Scope:** Extremely short-lived and transactional. An instance of BuilderSensorEval exists only for the duration of parsing a single JSON object. Its lifecycle is tied directly to the `readConfig` and `build` method calls.
- **Destruction:** The builder object is intended to be discarded and becomes eligible for garbage collection immediately after the `build` method returns the final SensorEval product. It holds no persistent state and is not retained by any system post-construction.

## Internal State & Concurrency
- **State:** Mutable and transient. The class maintains a single significant state field, `expression`, which is populated by the `readConfig` method. This state is not intended to be observed or modified outside of the configure-then-build sequence.
- **Thread Safety:** **This class is not thread-safe.** The `readConfig` method mutates the internal state of the object. If a single instance were shared across multiple threads, it would create a severe race condition. The asset loading pipeline is expected to process assets in a single-threaded or thread-contained manner, ensuring that each builder instance is only ever accessed by one thread.

## API Surface
The public API is designed for use by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorEval | O(1) | Constructs and returns the final SensorEval object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | Deserializes the provided JSON, populating the builder's internal state. Requires a string field named "Expression". |
| getExpression() | String | O(1) | Returns the configured JavaScript expression string. |
| getShortDescription() | String | O(1) | Provides metadata for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides detailed metadata for tooling and debugging. |

## Integration Patterns

### Standard Usage
The BuilderSensorEval is used exclusively by the server's asset loading machinery. The pattern is to instantiate, configure, build, and discard.

```java
// Hypothetical usage within an asset parser
JsonElement sensorConfig = getSensorJsonFromAsset();
BuilderSensorEval builder = new BuilderSensorEval();

// Configure the builder from the data source
builder.readConfig(sensorConfig);

// Construct the final runtime object
BuilderSupport support = ...; // Obtain from context
SensorEval runtimeSensor = builder.build(support);

// The 'builder' instance is no longer needed and can be garbage collected.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a BuilderSensorEval instance to build multiple SensorEval objects. Each builder is designed for a single, one-shot construction process. Re-using an instance can lead to state bleeding from a previous configuration.
- **State Mutation After Read:** Do not attempt to modify the builder's state after calling `readConfig`. The internal fields are not designed for public manipulation.
- **Calling build Before readConfig:** Invoking `build` on a non-configured builder will result in a SensorEval object with a null expression, leading to a NullPointerException or other undefined behavior at runtime.

## Data Pipeline
The class functions as a specific step in the data transformation pipeline that converts static asset files into live game engine objects.

> Flow:
> NPC_Asset.json -> Server AssetLoader -> JsonElement -> **BuilderSensorEval.readConfig()** -> Internal `expression` state -> **BuilderSensorEval.build()** -> `SensorEval` Instance -> NPC Behavior Tree

