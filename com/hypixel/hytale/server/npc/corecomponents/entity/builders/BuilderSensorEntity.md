---
description: Architectural reference for BuilderSensorEntity
---

# BuilderSensorEntity

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderSensorEntity extends BuilderSensorEntityBase {
```

## Architecture & Concepts
The BuilderSensorEntity is a factory component within the server-side NPC asset system. Its primary function is to translate a declarative JSON configuration block into a concrete, executable SensorEntity object. It acts as a deserializer and constructor, bridging the gap between static asset definitions and live game logic.

This builder is invoked by the asset pipeline when it encounters a sensor of this type within an NPC's behavior definition. It is responsible for parsing specific filter criteria, such as whether the sensor should detect players, other NPCs, or exclude entities of its own type.

A key architectural feature is its use of Holder objects, like BooleanHolder, instead of primitive types. This pattern defers the final resolution of a configuration value until runtime, allowing for dynamic behaviors. For example, whether to detect players might depend on the NPC's current state or world variables, which are provided at runtime via the BuilderSupport context.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's NPC asset loading system during the initial parsing of NPC behavior files. It is never created directly by game logic developers.
- **Scope:** Ephemeral and extremely short-lived. An instance of BuilderSensorEntity exists only for the duration of parsing a single JSON object and the subsequent call to its build method.
- **Destruction:** The object is immediately eligible for garbage collection after the `build` method returns the configured SensorEntity. It holds no persistent state and is not registered with any service locator.

## Internal State & Concurrency
- **State:** The builder's state is mutable and is populated during the `readConfig` call. It maintains three BooleanHolder fields: getPlayers, getNPCs, and excludeOwnType. These fields store the configuration data read from the JSON asset. This state is intended to be written once and read once.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively within the single-threaded context of the asset loading pipeline. Concurrent access, especially to the `readConfig` method, will result in a corrupted or unpredictable internal state.

## API Surface
The public API is designed for a two-step process: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Sensor> | O(N) | Parses the provided JSON, populating the internal state. N is the number of keys in the JSON object. Throws exceptions on malformed data. |
| build(BuilderSupport support) | SensorEntity | O(1) | Constructs and returns a new SensorEntity instance using the previously configured state. |
| isGetPlayers(BuilderSupport support) | boolean | O(1) | Resolves the runtime value for the getPlayers flag. Intended for internal use by the created SensorEntity. |
| isGetNPCs(BuilderSupport support) | boolean | O(1) | Resolves the runtime value for the getNPCs flag. Intended for internal use by the created SensorEntity. |
| isExcludeOwnType(BuilderSupport support) | boolean | O(1) | Resolves the runtime value for the excludeOwnType flag. Intended for internal use by the created SensorEntity. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal asset system. The pattern is to instantiate, configure from JSON, and immediately build the final sensor object.

```java
// Pseudocode for asset loading system
JsonElement sensorConfig = parseNpcBehaviorFile(".../behavior.json");
BuilderSensorEntity builder = new BuilderSensorEntity();

// 1. Configure the builder from the asset data
builder.readConfig(sensorConfig);

// 2. Build the final, immutable sensor object
// The builder instance is now discarded
SensorEntity sensor = builder.build(assetSupportContext);

// 3. The resulting sensor is integrated into the NPC's behavior tree
npc.getBehaviorTree().addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not attempt to reuse a BuilderSensorEntity instance to build multiple SensorEntity objects. Its state is not designed to be reset and will lead to configuration bleed-through.
- **Direct Instantiation:** Game logic developers must not instantiate this class using `new`. NPC behaviors should be defined entirely within JSON asset files.
- **Premature Access:** Calling `build` before `readConfig` will result in a SensorEntity with default, likely incorrect, behavior. The `is...` methods are not intended to be called on the builder itself, but are public for the SensorEntity it creates.

## Data Pipeline
The BuilderSensorEntity is a critical transformation step in the NPC asset pipeline, converting static data into a functional game object.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> **BuilderSensorEntity.readConfig()** -> **BuilderSensorEntity.build()** -> SensorEntity Instance -> NPC Behavior Tree -> Runtime Sensor Evaluation

