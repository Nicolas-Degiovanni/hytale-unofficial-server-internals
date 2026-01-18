---
description: Architectural reference for BuilderSensorNav
---

# BuilderSensorNav

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderSensorNav extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorNav class is a key component of the server-side NPC Behavior Asset system. It functions as a concrete implementation of the Builder pattern, specifically designed to construct SensorNav instances from a data-driven configuration, typically a JSON file.

In the Hytale NPC AI framework, a Sensor is a conditional node within a behavior tree that evaluates the state of the world or the NPC itself. The SensorNav specifically queries the status of the NPC's navigation component.

This builder acts as the translation layer between a game designer's declarative JSON configuration and a live, executable SensorNav object. It decouples the data format from the runtime logic, enabling designers to define complex AI conditions without writing Java code. The builder is responsible for parsing specific keys from the JSON data—such as NavStates, ThrottleDuration, and TargetDelta—and holding them in a temporary, structured format before the final Sensor object is assembled.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorNav is created by the NPC asset loading framework when it encounters a corresponding sensor definition in an NPC behavior file. The framework then immediately calls the readConfig method to populate the builder's internal state from the provided JSON data.
- **Scope:** This object is ephemeral and has a very short lifecycle. It exists only during the asset parsing and object construction phase. Its sole purpose is to hold configuration data temporarily.
- **Destruction:** Once the build method is invoked and the resulting SensorNav object is returned and integrated into the NPC's behavior tree, the BuilderSensorNav instance is no longer referenced and becomes eligible for garbage collection. It does not persist in the game state.

## Internal State & Concurrency
- **State:** The internal state is mutable exclusively during the configuration phase via the readConfig method. It uses specialized container objects like EnumSetHolder and DoubleHolder to store the parsed values. After this initial population, the state should be considered effectively immutable.

- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. The entire NPC asset loading pipeline is expected to operate on a single thread, preventing concurrent modification of a builder instance. Unsynchronized, multi-threaded access will lead to unpredictable behavior and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns the final SensorNav instance using the configured state. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the input JSON, populating the builder's internal state. N is the number of keys. |
| getNavStates(BuilderSupport) | EnumSet<NavState> | O(1) | Retrieves the configured set of navigation states to check against. |
| getThrottleDuration(BuilderSupport) | double | O(1) | Retrieves the configured minimum time the pathfinder must be unable to reach its target. |
| getTargetDelta(BuilderSupport) | double | O(1) | Retrieves the configured minimum distance a target must move to trigger the sensor. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability level of this component, e.g., Stable. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked internally by the asset loading system. The conceptual flow is as follows.

```java
// Conceptual example of internal asset loader usage
JsonElement sensorConfig = parseNpcBehaviorFile("...");
BuilderSensorNav builder = new BuilderSensorNav();

// 1. Configure the builder from data
builder.readConfig(sensorConfig);

// 2. Provide runtime context and build the final object
BuilderSupport support = new BuilderSupport(/* ... */);
Sensor sensor = builder.build(support);

// 3. The builder is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of BuilderSensorNav in gameplay code. NPC behaviors must be defined in data files and processed by the asset system.
- **State Re-Mutation:** Do not call readConfig more than once on a single instance. A builder is meant to process a single JSON definition.
- **Instance Re-Use:** Do not hold onto a builder instance to create multiple Sensor objects. Each Sensor definition from the configuration file must be processed by a new, dedicated builder instance.

## Data Pipeline
The BuilderSensorNav is a critical step in the pipeline that transforms a designer's configuration into a runtime AI component.

> Flow:
> NPC_Behavior.json -> Server Asset Parser -> **BuilderSensorNav.readConfig()** -> **BuilderSensorNav.build()** -> SensorNav Instance -> NPC Behavior Tree -> Live AI Evaluation

