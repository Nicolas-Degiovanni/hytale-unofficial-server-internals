---
description: Architectural reference for BuilderSensorCanInteract
---

# BuilderSensorCanInteract

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorCanInteract extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorCanInteract class is a configuration factory within the server-side NPC Behavior system. It serves a single, critical purpose: to deserialize a JSON definition into a concrete `SensorCanInteract` instance. This class acts as a bridge between the declarative world of asset configuration files and the imperative, executable world of the server's AI engine.

It is part of a larger **Builder Pattern** implementation used throughout the NPC asset pipeline. Each `Builder` class corresponds to a specific, configurable component of an NPC's behavior, such as a sensor, action, or condition.

A key architectural concept demonstrated here is the use of `Holder` objects (e.g., `FloatHolder`, `EnumSetHolder`). These wrappers decouple the static configuration from dynamic, runtime evaluation. This allows content designers to specify values that can be resolved later from the game's `ExecutionContext`, enabling behaviors that adapt to variables, player stats, or world state without requiring new code.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorCanInteract is created by the server's NPC asset loading system whenever it encounters the corresponding component type in an NPC's JSON definition. It is never instantiated directly by game logic code.
- **Scope:** The object's lifecycle is extremely short and confined to the asset loading process. It is a transient object that exists only to parse its configuration block and produce a `Sensor` object.
- **Destruction:** The instance is immediately eligible for garbage collection after the `build` method has been called and its product, the `SensorCanInteract` object, has been integrated into the parent NPC behavior asset. It holds no persistent state or external references.

## Internal State & Concurrency
- **State:** The internal state is mutable and is populated exclusively by the `readConfig` method. The fields `viewSector` and `attitudes` store the deserialized configuration data. Once `readConfig` completes, the object's state is effectively immutable for the remainder of its brief lifecycle.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated, configured, and used by a single thread within the asset loading pipeline. Concurrent calls to `readConfig` or other methods would result in a corrupted internal state and unpredictable behavior. This design is intentional, as asset loading is a controlled, sequential process.

## API Surface
The public API is designed for use by the asset loading system, not for general-purpose game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns the final `SensorCanInteract` instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the provided JSON, populating the builder's internal state. N is the number of keys in the JSON object. |
| getViewSectorRadians(BuilderSupport) | float | O(1) | Resolves and returns the configured view sector in radians. Requires a valid `BuilderSupport` context. |
| getAttitudes(BuilderSupport) | EnumSet<Attitude> | O(1) | Resolves and returns the configured set of attitudes. Requires a valid `BuilderSupport` context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the NPC asset factory. The conceptual flow within the engine is as follows:

```java
// Conceptual engine code - DO NOT REPLICATE
JsonElement sensorConfig = parseNpcDefinitionFile(".../my_npc.json");
BuilderSensorCanInteract builder = new BuilderSensorCanInteract();

// 1. Configure the builder from the data source
builder.readConfig(sensorConfig);

// 2. Build the final, immutable sensor object
// The BuilderSupport provides runtime context if needed
Sensor sensor = builder.build(engine.getBuilderSupport());

// The builder instance is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new BuilderSensorCanInteract()` in game logic. The NPC asset system is the sole owner of this class's lifecycle.
- **State Manipulation:** Do not call accessor methods like `getViewSectorRadians` before `readConfig` has been invoked. This will result in uninitialized state and likely throw a NullPointerException or return invalid default values.
- **Builder Reuse:** An instance of this builder is single-use. Do not attempt to call `readConfig` multiple times or use it to build more than one `Sensor` object.

## Data Pipeline
The BuilderSensorCanInteract class is a specific step in the data transformation pipeline that turns a static asset file into a live NPC agent.

> Flow:
> NPC Definition (*.json file*) -> Server Asset Loader -> `JsonElement` -> **BuilderSensorCanInteract.readConfig()** -> **BuilderSensorCanInteract.build()** -> `SensorCanInteract` Instance -> NPC Behavior Tree -> Live NPC Agent Evaluation

