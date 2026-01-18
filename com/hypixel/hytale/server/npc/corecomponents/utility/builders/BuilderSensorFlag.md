---
description: Architectural reference for BuilderSensorFlag
---

# BuilderSensorFlag

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderSensorFlag extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorFlag is a factory class that operates within the server's NPC asset loading pipeline. Its primary responsibility is to translate a declarative JSON configuration into a concrete, executable SensorFlag object. This class embodies the Builder pattern, which is a core design principle for NPC component instantiation throughout the engine.

In the context of an NPC's AI, a Sensor is a condition-checking node, effectively the "if" statement in a behavior tree. The resulting SensorFlag object is a highly optimized runtime component that checks if a specific boolean flag on an NPC's state is true or false.

A critical architectural function of this builder is the translation of a human-readable string flag name (e.g., "isAlerted") into a high-performance integer slot via the getFlagSlot method. This indirection, managed by the BuilderSupport context, ensures that the game loop avoids expensive string comparisons, instead performing a fast array or bitmask lookup. The use of Holder objects like StringHolder and BooleanHolder allows for deferred value resolution, a pattern that enables dynamic data to be injected at build time via an ExecutionContext.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the NPC asset deserialization system when it encounters a sensor definition of the corresponding type in a JSON file. It is never created directly by game logic developers.
- **Scope:** Ephemeral and extremely short-lived. An instance of BuilderSensorFlag exists only for the duration of parsing a single JSON object and the subsequent call to its build method.
- **Destruction:** The object is immediately eligible for garbage collection after the build method returns the final SensorFlag instance. It holds no persistent state and is not retained by any system.

## Internal State & Concurrency
- **State:** The internal state is mutable, consisting of StringHolder and BooleanHolder fields. This state is populated exclusively by the readConfig method during the asset loading phase. Once configured, the state should be considered immutable for the remainder of its brief lifecycle.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used in a single-threaded context during asset loading. Sharing a BuilderSensorFlag instance across multiple threads for configuration or building will result in unpredictable behavior and race conditions. The asset pipeline is responsible for ensuring this constraint.

## API Surface
The public API is designed for a single, sequential flow: readConfig, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs the runtime SensorFlag object. Throws if called before configuration. |
| readConfig(JsonElement) | Builder<Sensor> | O(k) | Populates the builder's internal state from a JSON definition. k is the number of keys. |
| getFlagSlot(BuilderSupport) | int | O(log N) | Resolves the configured flag name into a unique integer ID for fast runtime access. |
| getValue(BuilderSupport) | boolean | O(1) | Retrieves the configured boolean value the sensor will test against. |

## Integration Patterns

### Standard Usage
The BuilderSensorFlag is used exclusively by the internal asset loading framework. A developer will never interact with this class directly, but rather with the JSON that configures it.

```java
// Conceptual representation of internal engine usage
BuilderSensorFlag builder = new BuilderSensorFlag();
builder.readConfig(sensorJsonData);
Sensor runtimeSensor = builder.build(builderSupportContext);
npcBehaviorTree.addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new BuilderSensorFlag() in game logic. NPC behaviors are defined in data (JSON), not code.
- **State Reuse:** Do not attempt to reuse a builder instance to create a second, different sensor. Each builder is a one-shot factory for a single component definition.
- **Premature Building:** Calling build before readConfig has been successfully invoked will result in an improperly configured sensor and likely throw a runtime exception.

## Data Pipeline
This builder acts as a transformation step, converting declarative data into an executable object.

> Flow:
> NPC Asset (JSON) -> Asset Deserializer -> **BuilderSensorFlag.readConfig()** -> **BuilderSensorFlag.build()** -> SensorFlag (Runtime Object) -> NPC Behavior Tree

