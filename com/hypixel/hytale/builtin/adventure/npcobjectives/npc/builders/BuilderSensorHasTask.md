---
description: Architectural reference for BuilderSensorHasTask
---

# BuilderSensorHasTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorHasTask extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorHasTask class is a factory component within the server-side NPC (Non-Player Character) asset pipeline. It serves as a crucial bridge between declarative JSON configuration and executable server logic. Its primary responsibility is to parse a specific JSON definition for an NPC sensor and construct a runtime instance of SensorHasTask.

In Hytale's NPC architecture, behaviors are defined through a composition of components. A **Sensor** is a component that evaluates a condition about the game world, returning a boolean result. This specific builder creates a sensor that checks if a player currently interacting with the NPC has one or more specified tasks (quests) active.

This class embodies the Builder pattern, isolating the complex construction logic of a SensorHasTask object from its representation. Game designers define the desired behavior in a simple JSON format, and the engine's asset loader uses this class to translate that definition into a functional game object without needing to know the implementation details of the sensor itself.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorHasTask is created by the Hytale NPC asset loading system via reflection. This occurs when the system parses an NPC's JSON definition and encounters a sensor component with a type that maps to this builder. Direct instantiation by developers is not a supported or intended use case.
- **Scope:** The lifecycle of a BuilderSensorHasTask instance is extremely short and confined to the asset loading process. It exists only for the duration required to parse its corresponding JSON block and execute its build method.
- **Destruction:** Once the `build` method has been called and the resulting SensorHasTask object has been returned to the asset loader, the builder instance is no longer referenced. It becomes eligible for, and is promptly reclaimed by, the Java garbage collector. It does not persist with the NPC instance.

## Internal State & Concurrency
- **State:** The internal state of this class is **Mutable**. The `readConfig` method populates the internal `tasksById` field, which holds the list of task names parsed from the JSON configuration. This state is transient and serves only as a temporary container for configuration data before the final Sensor object is constructed.

- **Thread Safety:** This class is **Not Thread-Safe** and must not be accessed from multiple threads. It is designed to be used exclusively within the single-threaded context of the NPC asset loading sequence. The methods mutate internal state without any synchronization mechanisms, as concurrent access is not an expected operational condition.

## API Surface
The public API is designed for consumption by the automated asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getShortDescription() | String | O(1) | Provides a brief, human-readable summary for use in development tools. |
| build(BuilderSupport) | Sensor | O(1) | The core factory method. Constructs and returns a new SensorHasTask instance. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component, e.g., Stable. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the JSON configuration, populating the builder's internal state. N is the number of task IDs. |
| getTasksById(BuilderSupport) | String[] | O(1) | Accessor for the configured task IDs. Primarily called by the SensorHasTask constructor during the build process. |

## Integration Patterns

### Standard Usage
This builder is not used directly in code. Instead, it is invoked by the engine when a game designer defines the corresponding sensor in an NPC's JSON asset file.

```json
// Example NPC asset JSON snippet
{
  "type": "Sensor",
  "id": "hytale:sensor_has_task",
  "config": {
    "TasksById": [
      "hytale:main_quest_01",
      "hytale:side_quest_gather_wood"
    ]
  }
}
```
The asset loader identifies "hytale:sensor_has_task", instantiates BuilderSensorHasTask, and passes the `config` object to its `readConfig` method.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderSensorHasTask()`. The NPC asset system is responsible for the lifecycle of builder objects. Manually creating one circumvents the asset pipeline and will result in a non-functional component.
- **State Re-use:** This builder is not designed for re-use. Do not call `readConfig` multiple times on the same instance or attempt to modify its state after the initial parse. Each component definition in JSON results in a new, single-use builder instance.
- **Manual Building:** Do not call the `build` method manually. The `BuilderSupport` parameter is a complex context object provided by the asset system, and attempting to mock or create it will lead to runtime exceptions or unpredictable behavior.

## Data Pipeline
This class operates within a configuration-time data pipeline, transforming declarative data into an object.

> Flow:
> NPC JSON Asset File -> Hytale Asset Loader -> **BuilderSensorHasTask.readConfig()** -> Internal State Populated -> **BuilderSensorHasTask.build()** -> SensorHasTask Instance -> NPC Behavior Tree

