---
description: Architectural reference for BuilderSensorBlock
---

# BuilderSensorBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorBlock extends BuilderSensorBase {
```

## Architecture & Concepts

The BuilderSensorBlock is a configuration-driven factory responsible for instantiating a SensorBlock component for an NPC. It serves as a critical bridge between static asset definitions (typically JSON files) and live, in-world NPC sensor objects. Its primary role is within the NPC behavior asset loading pipeline.

This class does not perform any world queries itself. Instead, it acts as a blueprint, holding the configuration that the resulting SensorBlock will use at runtime to perform its environmental scans.

A key architectural pattern employed here is the use of Holder objects, such as DoubleHolder and AssetHolder. These objects decouple the static configuration from the dynamic runtime values. This allows NPC designers to define sensor parameters not just as fixed values, but as references to variables within an NPC's execution context. For example, the search *range* can be linked to a variable that changes based on the NPC's level or current state, enabling more dynamic and complex AI behaviors without altering the core asset.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderSensorBlock is created by the server's NPC asset parsing system when it encounters a "block" sensor definition within an NPC's behavior configuration file. It is not intended for manual instantiation by game logic developers.
-   **Scope:** The builder's lifecycle is ephemeral and confined to the asset loading and NPC initialization phase. It exists only to parse a JSON configuration block and produce a corresponding Sensor instance.
-   **Destruction:** Once the `build` method is called and the resulting SensorBlock is attached to the NPC's behavior component, the BuilderSensorBlock instance is no longer referenced by the asset loader and becomes eligible for garbage collection. However, the newly created SensorBlock maintains a reference to it for runtime configuration retrieval.

## Internal State & Concurrency

-   **State:** The internal state is **mutable** during the configuration phase, where the `readConfig` method populates the various Holder fields. After this phase, the state should be considered effectively immutable. The state itself does not contain world data but rather the *parameters* for querying world data.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the single-threaded context of the NPC asset loading pipeline. Concurrent calls to `readConfig` or modification of its fields after the build phase will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorBlock instance configured by this builder. |
| readConfig(JsonElement) | Builder | O(N) | Parses the provided JSON data to configure the builder's internal state. N is the number of properties in the JSON. |
| getRange(BuilderSupport) | double | O(1) | Resolves the search range at runtime. Called by the SensorBlock during its update tick. |
| getBlockSet(BuilderSupport) | int | O(1) | Resolves the target block set asset index at runtime. Throws IllegalArgumentException if the asset key is unknown. |
| isReserveBlock(BuilderSupport) | boolean | O(1) | Resolves the reservation behavior flag at runtime. |

## Integration Patterns

### Standard Usage

The BuilderSensorBlock is used exclusively by the internal NPC behavior system. A developer would define its properties in a JSON file, which the engine then uses to create and configure the builder automatically.

```java
// Engine-level code (conceptual)
// A service parses an NPC's JSON behavior file.

JsonElement sensorDefinition = parseNpcBehaviorFile(".../creeper.json");
BuilderSensorBlock builder = new BuilderSensorBlock();

// The engine populates the builder from the JSON data.
builder.readConfig(sensorDefinition);

// The engine builds the runtime sensor and attaches it to the NPC.
Sensor runtimeSensor = builder.build(npc.getBuilderSupport());
npc.addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)

-   **Manual Lifecycle Management:** Do not instantiate and manage a BuilderSensorBlock directly in game logic. All sensor configuration should be handled through asset files to ensure consistency and maintain the data-driven design of the AI system.
-   **State Mutation After Build:** Do not modify the builder's state after the `build` method has been called. The created SensorBlock holds a direct reference to its builder. Modifying the builder post-creation will unpredictably alter the sensor's behavior at runtime, creating bugs that are difficult to trace.
-   **Builder Reuse:** A single builder instance should not be used to configure and build multiple, distinct SensorBlock instances. Each sensor in a behavior tree must have its own unique builder instance.

## Data Pipeline

The flow of configuration from a static file to a runtime query is a multi-stage process orchestrated by the engine.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> **BuilderSensorBlock.readConfig** -> In-Memory Builder State -> **BuilderSensorBlock.build()** -> SensorBlock Instance -> Game Tick -> SensorBlock queries Builder for runtime values -> World Query -> NPC Blackboard Update

