---
description: Architectural reference for BuilderSensorReadPosition
---

# BuilderSensorReadPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorReadPosition extends BuilderSensorBase {
```

## Architecture & Concepts

The BuilderSensorReadPosition class is a factory component within the server-side NPC AI framework. It operates as a deserializer and builder for a specific type of AI perception module: the SensorReadPosition. Its primary function is to translate a JSON configuration block from an NPC behavior asset file into a live, executable Sensor object.

This class embodies the Builder pattern. It is not the runtime sensor itself, but rather the blueprint used to construct it. The core architectural concept is a two-stage process:

1.  **Configuration Time:** During server startup or asset loading, an instance of BuilderSensorReadPosition is created for each corresponding definition in an NPC's JSON file. The `readConfig` method is invoked to parse the JSON and populate the builder's internal state.
2.  **Runtime:** The constructed `SensorReadPosition` object maintains a direct reference to its originating builder. During an AI tick, the sensor calls back into this builder (e.g., `getSlot`, `getRange`) to resolve its operational parameters.

This deferred resolution, facilitated by `Holder` objects (e.g., StringHolder, DoubleHolder) and the `BuilderSupport` context, allows for highly dynamic AI. Parameters like target slots or ranges can be resolved just-in-time based on the NPC's current state, rather than being statically baked into the sensor at creation.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the NPC asset loading system when it encounters a sensor of this type within an NPC's JSON behavior definition. A new, distinct instance is created for every unique sensor definition in the asset files.
-   **Scope:** The lifecycle of a BuilderSensorReadPosition instance is tightly coupled to the `SensorReadPosition` instance it creates. It is effectively owned by the sensor and must persist in memory as long as the sensor is part of an active NPC behavior tree.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC asset is unloaded from memory, which in turn releases the reference to the `SensorReadPosition` object that holds it.

## Internal State & Concurrency

-   **State:** The internal state is mutable during the initial configuration phase when `readConfig` is called. After this phase, the builder's configuration is considered final. The `Holder` fields (`slot`, `range`, etc.) do not change, but the values they resolve to can vary at runtime depending on the `ExecutionContext` provided.
-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** The asset loading and AI evaluation pipelines are designed to operate within a single-threaded context (per world or per AI agent). Concurrent calls to its methods, especially `readConfig`, will lead to corrupted state. Runtime access via the `get...` methods from the sensor is assumed to happen synchronously within the main server game loop.

## API Surface

The public API is primarily for internal use by the asset loader and the sensor it builds.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Factory method. Instantiates the runtime SensorReadPosition object. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes a JSON object to configure the builder's internal state. N is the number of keys. |
| getSlot(BuilderSupport) | int | O(1) | Runtime resolution of the memory slot index to read the position from. |
| getRange(BuilderSupport) | double | O(1) | Runtime resolution of the maximum distance for the sensor to be considered "in range". |
| getMinRange(BuilderSupport) | double | O(1) | Runtime resolution of the minimum distance for the sensor to be considered "in range". |
| isUseMarkedTarget(BuilderSupport) | boolean | O(1) | Runtime resolution of whether to use a target slot or a standard position slot. |

## Integration Patterns

### Standard Usage

Developers do not interact with this Java class directly. Instead, they define its behavior declaratively in an NPC's JSON asset file. The system handles the instantiation and configuration based on this definition.

```json
// Example NPC Behavior Snippet (conceptual)
{
  "component": "Sensor",
  "type": "ReadPosition",
  "params": {
    "Slot": "lastKnownEnemyLocation",
    "UseMarkedTarget": false,
    "Range": 32.0,
    "MinRange": 2.0
  }
}
```

The game's asset loader parses this JSON block, instantiates a BuilderSensorReadPosition, and calls `readConfig` with the `params` object. The resulting builder is then used to create the sensor for the NPC's behavior tree.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BuilderSensorReadPosition()` in game logic. The class is useless without being configured from a JSON source by the asset pipeline.
-   **State Mutation After Build:** Do not attempt to get a reference to a builder and modify its state after the `build` method has been called. This leads to undefined behavior, as the sensor it already created will not reflect these changes.
-   **Instance Caching or Re-use:** Each builder instance is tied to a single, specific sensor definition. Do not attempt to cache and re-use builders for multiple sensors, as their internal state is unique to their original JSON configuration.

## Data Pipeline

The flow of data from configuration to runtime execution is a key aspect of the NPC AI system.

> Flow:
> NPC Behavior JSON File -> Asset Loading Service -> **BuilderSensorReadPosition.readConfig()** -> **BuilderSensorReadPosition.build()** -> SensorReadPosition Instance -> NPC Behavior Tree -> AI Tick Evaluation

