---
description: Architectural reference for BuilderSensorTarget
---

# BuilderSensorTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorTarget extends BuilderSensorWithEntityFilters {
```

## Architecture & Concepts
The **BuilderSensorTarget** class is a configuration-driven factory within the server-side NPC asset pipeline. It is not a live game component but rather a blueprint used during the server's asset loading phase. Its primary responsibility is to deserialize a JSON configuration block and construct a **SensorTarget** instance, which is the actual runtime object used by an NPC's AI to evaluate potential targets.

This class acts as a bridge between static data assets (JSON files) and live, executable AI logic. It encapsulates the logic for parsing specific properties like *Range*, *TargetSlot*, and *AutoUnlockTarget*. By extending **BuilderSensorWithEntityFilters**, it inherits and integrates a standardized mechanism for applying complex filtering criteria, allowing designers to specify that a target is only valid if it meets conditions defined by other sensors (e.g., IsHostile, IsWithinHealthThreshold).

The use of Holder objects (e.g., **DoubleHolder**, **StringHolder**) is a key architectural pattern. This defers the final resolution of configuration values until runtime, enabling dynamic behaviors where parameters can change based on the NPC's current state or execution context.

## Lifecycle & Ownership
-   **Creation:** An instance of **BuilderSensorTarget** is created by the NPC asset loading system when it encounters a corresponding component definition within an NPC's JSON asset file. This process is typically managed by a central factory or registry that maps JSON type names to builder classes.
-   **Scope:** The object is extremely short-lived. It exists only for the duration of parsing its specific JSON block and executing its **build** method. It does not persist into the active game simulation.
-   **Destruction:** The instance is eligible for garbage collection immediately after the **build** method returns the configured **SensorTarget** object. Its memory footprint is transient and tied strictly to the server's startup or asset-reloading sequence.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** during the configuration phase. The **readConfig** method populates the internal Holder fields directly from the parsed JSON data. After this phase, the state is treated as effectively immutable and is only read to construct the final **SensorTarget** object.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be used in a single-threaded context. The NPC asset loading pipeline is expected to process assets sequentially. Concurrent access to a single **BuilderSensorTarget** instance, especially during the **readConfig** phase, will result in a corrupted and unpredictable state.

## API Surface
The public API is designed for consumption by the asset loading system, not for general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorTarget | O(1) | Constructs and returns the runtime **SensorTarget** instance. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Populates the builder's internal state from a JSON object. N is the number of keys. |
| getRange(BuilderSupport) | double | O(1) | Resolves the configured maximum range for the target check. |
| getAutoUnlockTarget(BuilderSupport) | boolean | O(1) | Resolves the flag determining if the target should be unlocked on failure. |
| getTargetSlot(BuilderSupport) | int | O(1) | Resolves the integer ID of the target slot to be evaluated. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay code. It is invoked transparently by the asset system. A game designer or developer interacts with it by defining its properties in an NPC's JSON file.

The *system-level* interaction during asset loading follows this pattern:
```java
// Pseudocode for asset loading pipeline
BuilderSensorTarget builder = new BuilderSensorTarget();
builder.readConfig(jsonElementFromAsset);

// The BuilderSupport provides runtime context and services
Sensor sensor = builder.build(builderSupport);
npcBehaviorTree.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually instantiate with **new BuilderSensorTarget()** in game logic. NPC behaviors should be defined entirely within data assets.
-   **State Mutation:** Do not attempt to call setters or modify public fields after **readConfig** has completed. The builder's state is considered sealed post-configuration.
-   **Instance Re-use:** Do not re-use a single builder instance to parse multiple, distinct JSON configurations. This will lead to state corruption. A new builder must be created for each component definition.

## Data Pipeline
The **BuilderSensorTarget** is a critical step in the transformation of static configuration data into an executable AI component.

> Flow:
> NPC Asset JSON File -> Server Asset Parser -> **BuilderSensorTarget.readConfig()** -> **BuilderSensorTarget.build()** -> SensorTarget Instance -> NPC Behavior Engine Execution

