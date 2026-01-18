---
description: Architectural reference for BuilderSensorSelf
---

# BuilderSensorSelf

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorSelf extends BuilderSensorWithEntityFilters {
```

## Architecture & Concepts
The BuilderSensorSelf class is a specialized factory within the server-side NPC asset pipeline. It embodies the Builder design pattern, serving the sole purpose of translating a JSON configuration block into a fully realized, runtime SensorSelf object.

This component acts as a bridge between static data assets (defined by designers in JSON files) and the live NPC AI engine. The SensorSelf object it produces is a fundamental building block for an NPC's self-awareness, allowing the AI to pose questions about its own state. For example, an NPC can use a sensor built by this class to determine "Is my health below 25%?" or "Am I currently on fire?".

Architecturally, it inherits common filter-parsing logic from its parent, BuilderSensorWithEntityFilters. This design promotes code reuse, allowing BuilderSensorSelf to focus exclusively on the context of applying these filters to the NPC entity itself, rather than to external entities.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorSelf is created dynamically by the server's asset loading system when it encounters the corresponding component type within an NPC's JSON definition file. It is never instantiated directly in game logic code.

- **Scope:** The lifecycle of a BuilderSensorSelf instance is extremely brief and ephemeral. It exists only during the asset parsing and loading phase. Its scope is confined to the execution of the `readConfig` and `build` methods.

- **Destruction:** The object is dereferenced and becomes eligible for garbage collection immediately after the `build` method returns the final SensorSelf product. It holds no persistent state and is not retained by any system post-load.

## Internal State & Concurrency
- **State:** This class is fundamentally **Mutable**. Its primary role is to accumulate state from a JSON source via the `readConfig` method, storing it internally (specifically in the `filters` collection inherited from its parent) before this state is consumed by the `build` method.

- **Thread Safety:** BuilderSensorSelf is **Not Thread-Safe** and must be treated as single-threaded. The asset loading pipeline is responsible for ensuring that each builder instance is created, configured, and used within a single, synchronized context. Concurrent access would lead to a corrupted internal state and result in unpredictable runtime AI behavior.

## API Surface
The public API is minimal, reflecting its focused role as a configuration-driven factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorSelf | O(1) | Constructs the final SensorSelf object from the builder's internal state. This is the terminal operation. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses a JSON object to configure the builder's internal filters. N is the number of filters in the JSON array. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is reserved for the asset pipeline. A designer or developer triggers its use by defining the component in an NPC's JSON file. The system then handles the lifecycle internally.

A conceptual representation of the *system's* interaction:

```java
// This code is conceptual and executed by the AssetManager, not a game developer.

// 1. A JSON block is parsed from a file.
JsonElement sensorConfig = gson.fromJson(jsonString, JsonElement.class);

// 2. The appropriate builder is instantiated.
BuilderSensorSelf builder = new BuilderSensorSelf();

// 3. The builder is configured from the JSON data.
builder.readConfig(sensorConfig);

// 4. The final runtime object is constructed.
// The builder instance is now discarded.
SensorSelf runtimeSensor = builder.build(builderSupportContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Under no circumstances should game logic instantiate this class using `new BuilderSensorSelf()`. The NPC's behavior should be defined entirely within data assets.

- **State Re-use:** Do not attempt to cache and re-use a BuilderSensorSelf instance. They are designed as one-shot factories and are not safe for reconfiguration or repeated builds.

- **Multi-threading:** Never share an instance of this builder across multiple threads. The asset loading system must enforce a single-threaded processing model for each builder.

## Data Pipeline
The BuilderSensorSelf is a critical transformation step in the data flow from static configuration to a live AI component.

> Flow:
> NPC Definition (*.json file) -> GSON Parser -> Asset Loading Service -> **BuilderSensorSelf** -> SensorSelf (Runtime Instance) -> NPC Behavior Tree

