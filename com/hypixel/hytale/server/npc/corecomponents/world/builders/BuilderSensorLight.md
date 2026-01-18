---
description: Architectural reference for BuilderSensorLight
---

# BuilderSensorLight

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorLight extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorLight class is a factory component within the server-side NPC asset pipeline. Its primary responsibility is to deserialize a JSON configuration object and construct a corresponding SensorLight instance. It acts as the bridge between static data defined by designers in asset files and the live, executable sensor objects used by the NPC AI system.

This class does not perform any light-level checks itself. Instead, it serves as a configuration holder and factory for the SensorLight object that contains the actual runtime logic.

A critical architectural pattern employed here is the use of Holder objects, such as NumberArrayHolder and StringHolder. These objects encapsulate configuration values, deferring their final resolution until runtime. The constructed SensorLight object retains a reference to its originating BuilderSensorLight instance to query these values via a BuilderSupport context during the game tick. This allows for dynamic value resolution based on the NPC's current state or other contextual factors.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderSensorLight is created by the NPC asset loading system when it encounters a sensor of type "Light" within an NPC behavior definition file (typically JSON). This process is automated and hidden from the typical game logic developer.
- **Scope:** The builder's lifecycle is intrinsically linked to the SensorLight object it creates. Because the SensorLight holds a direct reference to the builder to resolve its configuration at runtime, the builder persists in memory as long as the SensorLight is part of an active NPC behavior tree. Its scope is effectively tied to the loaded NPC asset.
- **Destruction:** The object is marked for garbage collection when the parent NPC asset is unloaded from memory. This typically occurs when a world region is unloaded or during a server shutdown sequence.

## Internal State & Concurrency
- **State:** The internal state is highly mutable during the initial configuration phase, specifically within the readConfig method. After this phase, the state, encapsulated within the various Holder fields, is treated as immutable for the remainder of its lifecycle.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and configured within a single-threaded asset loading context. Concurrent calls to readConfig will result in a corrupted state. Access via the public get methods from the owning SensorLight is only safe if performed within the single-threaded server game loop, which is the intended operational model.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorLight | O(1) | Constructs and returns a new SensorLight instance. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Deserializes the provided JSON, populating the internal state. N is the number of keys in the JSON object. |
| getUsedTargetSlot(BuilderSupport) | int | O(1) | Resolves the configured target slot name into its runtime integer ID. |
| getLightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured light intensity range. |
| getSkyLightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured sky light range. |
| getSunlightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured sunlight range. |
| getRedLightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured red light channel range. |
| getGreenLightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured green light channel range. |
| getBlueLightRange(BuilderSupport) | double[] | O(1) | Resolves and returns the configured blue light channel range. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. It is operated upon by the server's asset loading systems. The following example illustrates the conceptual flow.

```java
// This process is handled automatically by the asset loading system.

// 1. A JSON definition for an NPC sensor is parsed from a behavior file.
JsonElement sensorConfig = NpcAssetParser.getSensorJson("my_npc_behavior.json");

// 2. The system instantiates the appropriate builder based on the sensor type.
BuilderSensorLight builder = new BuilderSensorLight();

// 3. The configuration is read into the builder.
builder.readConfig(sensorConfig);

// 4. The final Sensor object is built and integrated into the NPC's behavior tree.
// The 'builderSupport' object provides necessary runtime context.
Sensor sensor = builder.build(builderSupport);
npcBehaviorTree.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Manual Configuration:** Do not manually instantiate this class with `new BuilderSensorLight()` in game logic. All sensor configuration should be defined in JSON asset files to ensure consistency and maintain the integrity of the content pipeline.
- **State Modification After Build:** Do not call readConfig on a builder instance after its corresponding SensorLight has been built and is in use by an active NPC. This will lead to unpredictable sensor behavior, as the live sensor will begin reading modified configuration data mid-tick.
- **Builder Reuse:** A single BuilderSensorLight instance should not be used to configure multiple, distinct SensorLight objects. Each sensor in an asset file must have its own dedicated builder instance to prevent state corruption.

## Data Pipeline
The flow of configuration data from asset file to runtime execution is as follows:

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderSensorLight.readConfig()** -> SensorLight Instance -> NPC Behavior Tree -> Game Tick -> **BuilderSensorLight.get...Range()** -> Sensor Result (boolean)

