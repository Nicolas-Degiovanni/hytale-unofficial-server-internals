---
description: Architectural reference for BuilderActionResetBlockSensors
---

# BuilderActionResetBlockSensors

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionResetBlockSensors extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionResetBlockSensors class is a key component in the server-side NPC behavior system. It functions as a data-driven factory, responsible for deserializing a JSON configuration block and constructing a runtime-executable `Action` object.

Its primary architectural role is to bridge the gap between static data definitions (NPC behavior files) and the live game engine. It translates human-readable asset names, specifically BlockSet identifiers, from the configuration into optimized, integer-based indices used by the game for fast lookups. This separation of configuration (the Builder) from execution (the Action) is a fundamental pattern in Hytale's NPC AI, allowing designers to define complex behaviors in data without modifying engine code.

This class is not intended for direct use in gameplay logic; it is an internal component of the asset loading and NPC initialization pipeline.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderActionResetBlockSensors is created by the NPC behavior parsing system whenever it encounters the corresponding action type within an NPC's JSON definition file.
- **Scope:** The object's lifetime is extremely short and confined to the parsing process. It exists only to read a specific JSON fragment and produce a single `ActionResetBlockSensors` instance.
- **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method has been called and its resulting `Action` has been integrated into the NPC's behavior tree. It does not persist.

## Internal State & Concurrency
- **State:** This class is **Mutable**. Its primary internal state is the `blockSets` field, an AssetArrayHolder. This field is populated by the `readConfig` method and holds the list of BlockSet names specified in the JSON configuration. This state is transient and exists only to facilitate the creation of the final `Action` object.

- **Thread Safety:** This class is **Not Thread-Safe**. It is designed to be instantiated and used within a single-threaded context during asset deserialization. The sequence of calling `readConfig` followed by `build` is not atomic and must not be performed concurrently on the same instance.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new `ActionResetBlockSensors` instance. |
| readConfig(JsonElement) | BuilderActionResetBlockSensors | O(N) | Deserializes the JSON data, populating the internal list of BlockSets. N is the number of BlockSets in the config. |
| getBlockSets(BuilderSupport) | int[] | O(N) | Resolves the stored BlockSet names into their integer asset indices. Throws IllegalArgumentException if a name is not found. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the engine's behavior parsing systems. The typical lifecycle is programmatic and follows a strict sequence.

```java
// Pseudo-code for engine's behavior parser
JsonElement actionConfig = parseActionFromJson("...");
BuilderActionResetBlockSensors builder = new BuilderActionResetBlockSensors();

// 1. Configure the builder from data
builder.readConfig(actionConfig);

// 2. Build the final, executable action
Action runtimeAction = builder.build(supportContext);

// The builder is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not manually instantiate this class in gameplay code. NPC behaviors should be defined entirely within JSON asset files.
- **State Tampering:** Do not call `getBlockSets` before `readConfig` has successfully populated the builder. This will result in an empty array or unpredictable behavior.
- **Instance Re-use:** Do not attempt to re-use a single builder instance to parse multiple, different JSON configurations. The internal state is not cleared between calls to `readConfig`, leading to incorrect or merged configurations.

## Data Pipeline
This builder acts as a transformation step in the NPC asset loading pipeline, converting declarative JSON data into an optimized, executable game object.

> Flow:
> NPC Behavior JSON File -> Engine JSON Parser -> **BuilderActionResetBlockSensors.readConfig()** -> **BuilderActionResetBlockSensors.build()** -> `ActionResetBlockSensors` Object -> NPC Behavior Tree

