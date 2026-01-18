---
description: Architectural reference for BuilderSensorAdjustPosition
---

# BuilderSensorAdjustPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorAdjustPosition extends BuilderSensorBase {
```

## Architecture & Concepts
The **BuilderSensorAdjustPosition** class is a configuration-driven factory responsible for deserializing and constructing a **SensorAdjustPosition** runtime object. It operates exclusively within the server's NPC asset loading pipeline.

Architecturally, this class implements the Builder pattern. Its primary function is to translate a declarative JSON configuration block into a functional Java object. It embodies the Decorator pattern at the configuration level; it does not create a sensor from scratch but rather wraps an existing sensor definition, augmenting its behavior by applying a positional offset.

This builder acts as an intermediary between the raw asset data and the live NPC sensory system. It parses references to other sensor definitions and vector data, validates them, and ultimately produces a component that modifies the output of another sensor.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's central **BuilderManager** when it encounters the corresponding type identifier within an NPC's JSON asset file. The `readConfig` method is invoked by the manager immediately following instantiation.
- **Scope:** Ephemeral and short-lived. An instance of this builder exists only for the duration of a single NPC asset's parsing and construction phase. It is a throwaway object used to process one specific configuration block.
- **Destruction:** The builder instance becomes eligible for garbage collection as soon as its `build` method is called and the resulting **SensorAdjustPosition** object is returned to the **BuilderManager**. It holds no persistent references and is not retained.

## Internal State & Concurrency
- **State:** This object is highly mutable. Its internal fields, **sensor** and **offset**, are populated by the `readConfig` method. The state is held in specialized container objects (**BuilderObjectReferenceHelper**, **NumberArrayHolder**) which manage unresolved references and expressions until the final `build` phase. The state is considered incomplete and invalid until the entire asset loading process is finished.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used within a single, synchronous asset-loading thread. Concurrent access would result in a corrupted internal state and unpredictable build outcomes. The entire NPC building pipeline is architected as a single-threaded process.

## API Surface
The public API is designed for internal framework use by the **BuilderManager**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorAdjustPosition | O(N) | Constructs the final runtime **SensorAdjustPosition** object. Complexity is dependent on the depth (N) of the wrapped sensor chain, as it triggers a recursive build. Returns null if the wrapped sensor fails to build. |
| readConfig(JsonElement) | BuilderSensorAdjustPosition | O(1) | Deserializes a JSON object to populate the builder's internal state. This is the primary entry point for configuration. |
| validate(...) | boolean | O(N) | Recursively validates the configuration of this builder and the wrapped sensor it references. |
| getSensor(BuilderSupport) | Sensor | O(N) | A protected-access helper that resolves and builds the wrapped sensor. |
| getOffset(BuilderSupport) | Vector3d | O(1) | A protected-access helper that resolves the offset expression and returns it as a concrete **Vector3d**. |

## Integration Patterns

### Standard Usage
Developers do not interact with this Java class directly. Instead, they declare its use within an NPC's JSON definition file. The engine's **BuilderManager** handles the instantiation and invocation based on the asset data.

**Conceptual JSON Configuration:**
```json
{
  "type": "SensorAdjustPosition",
  "sensor": {
    "type": "SensorSelf"
  },
  "offset": [0, 5.0, 0]
}
```

**Engine-Side Invocation (Simplified):**
```java
// This logic resides deep within the server's BuilderManager
// and is not exposed to game developers.
BuilderSensorBase builder = builderManager.createBuilderFor("SensorAdjustPosition");
builder.readConfig(jsonObject); // Pass the JSON block from the asset file

// Later, during the final linking phase...
Sensor runtimeSensor = builder.build(builderSupport);
npc.addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorAdjustPosition()`. The object is useless without the context (**BuilderSupport**) and lifecycle management provided by the framework's **BuilderManager**. It will fail to build any object.
- **State Reuse:** Do not attempt to call `build` more than once on a single builder instance. Builders are single-use and their internal state is not designed for reset or reuse.
- **Post-Config Modification:** Do not modify the builder's internal state after `readConfig` has been called. The validation and build process assumes the state is immutable after the initial read.

## Data Pipeline
This builder is a key stage in the transformation of static configuration data into a live game object.

> Flow:
> NPC JSON Asset -> Server Asset Parser -> **BuilderSensorAdjustPosition.readConfig()** -> Validation Pass -> **BuilderSensorAdjustPosition.build()** -> **SensorAdjustPosition** (Runtime Instance) -> NPC Behavior Engine

