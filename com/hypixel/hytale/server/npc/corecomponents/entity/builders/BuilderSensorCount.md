---
description: Architectural reference for BuilderSensorCount
---

# BuilderSensorCount

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient / Builder

## Definition
```java
// Signature
public class BuilderSensorCount extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorCount class is a foundational component within the server's NPC behavior system, operating as part of the asset-to-runtime pipeline. It embodies the Builder design pattern, serving as a deserializer and factory for the concrete SensorCount object.

Its primary responsibility is to translate a static JSON configuration from an NPC behavior asset file into a memory-resident representation. This decouples the declarative, data-driven design of NPC behaviors from their imperative, in-game execution logic. During the server's NPC initialization phase, this builder is instantiated, configured via its readConfig method, and then used to construct the final SensorCount instance which is subsequently integrated into an NPC's behavior tree.

This architecture allows game designers to define complex NPC sensory conditions in simple JSON files without modifying core engine code.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorCount is created dynamically by the NPC asset loading system when it encounters a sensor of type "SensorCount" within a behavior definition file. It is not intended for manual instantiation by developers.
- **Scope:** The object's lifecycle is ephemeral and strictly confined to the asset loading and NPC initialization process. It exists only to parse a JSON block and produce a single SensorCount object.
- **Destruction:** The builder becomes eligible for garbage collection immediately after the build method is invoked and its resulting SensorCount product is passed to the NPC's behavior tree. It holds no persistent state and is not retained by any long-lived system.

## Internal State & Concurrency
- **State:** The BuilderSensorCount is inherently stateful and mutable. Its internal fields, such as count, range, includeGroups, and excludeGroups, are populated sequentially during the invocation of the readConfig method. This state is a direct, intermediate representation of the source JSON asset.

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access during the server's initialization or asset-loading phase. Concurrent calls to readConfig or build will result in a corrupted internal state and unpredictable behavior. All operations on a BuilderSensorCount instance must be externally synchronized or confined to a single thread.

## API Surface
The public API is focused on the two primary operations of a builder: configuration and construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorCount | O(1) | Constructs the final, runtime SensorCount object using the builder's configured state. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the provided JSON data into the builder's internal fields. N is the number of keys in the JSON object. Throws exceptions on validation failure. |
| getCount(BuilderSupport) | int[] | O(1) | Retrieves the configured entity count range. **Warning:** Only valid after readConfig has been called. |
| getRange(BuilderSupport) | double[] | O(1) | Retrieves the configured detection range. **Warning:** Only valid after readConfig has been called. |
| getIncludeGroups() | int[] | O(M) | Resolves and returns the integer tag indices for included groups. M is the number of groups. |
| getExcludeGroups() | int[] | O(M) | Resolves and returns the integer tag indices for excluded groups. M is the number of groups. |

## Integration Patterns

### Standard Usage
The builder is used exclusively by the internal NPC asset pipeline. A developer will never interact with this class directly, but will instead define its properties in a JSON file. The following pseudocode illustrates the engine's internal process.

```java
// Pseudocode representing the asset loading pipeline

// 1. A new builder is instantiated for a specific sensor definition
BuilderSensorCount builder = new BuilderSensorCount();
JsonElement sensorConfig = loadNpcBehaviorAsset(".../my_npc.json").get("sensor");

// 2. The engine configures the builder from the asset data
builder.readConfig(sensorConfig);

// 3. The engine provides runtime context and builds the final sensor
BuilderSupport support = npc.getBuilderSupport();
SensorCount runtimeSensor = builder.build(support);

// 4. The resulting sensor is integrated into the NPC's live behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Configuration:** Do not manually instantiate with `new BuilderSensorCount()` and attempt to set its fields. The class relies on the validation and parsing logic within the readConfig method.
- **Builder Reuse:** A builder instance is single-use. Do not attempt to call readConfig multiple times on the same instance or use it to build more than one SensorCount object. This will lead to state corruption.
- **Premature State Access:** Accessing getter methods like getCount or getRange before readConfig has been successfully called will result in uninitialized data and likely cause a NullPointerException or other runtime error.

## Data Pipeline
The BuilderSensorCount is a critical transformation step that converts static data into a live game object.

> Flow:
> NPC Behavior JSON Asset -> Server Asset Loader -> **BuilderSensorCount.readConfig()** -> In-Memory Builder State -> **BuilderSensorCount.build()** -> SensorCount Runtime Instance -> NPC Behavior Tree

