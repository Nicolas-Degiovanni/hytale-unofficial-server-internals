---
description: Architectural reference for BuilderSensorPath
---

# BuilderSensorPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorPath extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorPath class is a factory component within the server's declarative NPC AI framework. It is not a sensor itself, but rather a **blueprint** responsible for translating a static JSON configuration into a live, executable SensorPath instance. Its primary role is to parse, validate, and hold the parameters required for an NPC to perform a world query for pathing information.

This class acts as an intermediary between the asset loading system and the NPC's runtime sensor system. It embodies the configuration for a specific path-finding query, such as finding the nearest waypoint of a named world path within a certain range.

A critical architectural pattern employed is the use of *Holder* objects (e.g., StringHolder, DoubleHolder). These wrappers decouple the static configuration values from the dynamic game state. This allows parameters like the path name or search range to be resolved at runtime from an ExecutionContext, enabling more complex and context-aware NPC behaviors.

The builder also declares the *Features* it provides (Position, Path), which informs the broader AI system about the types of data the resulting sensor can inject into the NPC's memory or blackboard.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorPath is created by the asset pipeline when it encounters a corresponding "SensorPath" definition within an NPC's JSON behavior file. This is an automated process managed by the server's configuration loaders.
- **Scope:** The object is short-lived and transient. Its lifecycle is confined to the asset loading and NPC instantiation phase. It exists only to parse a configuration block and be passed to the SensorPath constructor via the build method.
- **Destruction:** The BuilderSensorPath instance becomes eligible for garbage collection immediately after the `build` method is invoked and the resulting SensorPath object is registered with the NPC's sensor suite. It holds no persistent references and is not intended to outlive the initial setup process.

## Internal State & Concurrency
- **State:** The internal state is **mutable** during the configuration phase. The readConfig method populates the internal Holder fields from a JsonElement. Once readConfig is complete, the state should be considered effectively immutable. The created SensorPath object holds a direct reference to its builder to retrieve these configured values during execution.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. It is designed for synchronous, single-threaded use during the server's asset loading sequence. Any attempt to call readConfig or build from multiple threads will result in a corrupted state and undefined behavior.

## API Surface
The public API is focused on the factory pattern: configuring the object from data and then building the final product.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorPath instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the provided JSON, populating the internal state. N is the number of keys in the JSON object. |
| getPath(BuilderSupport) | String | O(1) | Resolves and returns the configured path name from the ExecutionContext. |
| getRange(BuilderSupport) | double | O(1) | Resolves and returns the configured search range from the ExecutionContext. |
| getPathType(BuilderSupport) | SensorPath.PathType | O(1) | Resolves and returns the configured path type from the ExecutionContext. |

## Integration Patterns

### Standard Usage
The BuilderSensorPath is not intended for direct use by game logic developers. It is invoked by the server's asset loading systems. The internal flow follows a clear pattern.

```java
// Conceptual example of how the system uses this builder
BuilderSensorPath builder = new BuilderSensorPath();

// 1. The builder is configured from a JSON source
builder.readConfig(npcBehaviorJson.get("sensor"));

// 2. The final Sensor object is constructed and registered
Sensor sensor = builder.build(builderSupport);
npc.getSensorSystem().register(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create instances with `new BuilderSensorPath()`. The asset system manages the lifecycle. A manually created instance without a subsequent call to readConfig is an unconfigured, useless object.
- **State Mutation Post-Build:** Do not modify the builder's state after the `build` method has been called. The created SensorPath maintains a reference to its builder. Modifying the builder after the fact will lead to unpredictable behavior in the live sensor.
- **Instance Re-use:** A single BuilderSensorPath instance must not be used to process multiple, different JSON configurations. Each configuration requires a new, dedicated builder instance to avoid state contamination.

## Data Pipeline
The BuilderSensorPath is a critical transformation step in the data pipeline that converts static NPC definition files into active, in-game AI components.

> Flow:
> NPC_Behavior.json -> Server Asset Loader -> JsonElement -> **BuilderSensorPath.readConfig()** -> In-Memory Builder State -> **BuilderSensorPath.build()** -> SensorPath Instance -> NPC Sensor System Execution

