---
description: Architectural reference for BuilderSensorEvent
---

# BuilderSensorEvent

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public abstract class BuilderSensorEvent extends BuilderSensorBase {
```

## Architecture & Concepts
BuilderSensorEvent is an abstract base class within the NPC asset definition pipeline. It serves as a configuration blueprint for creating `Sensor` instances that react to discrete world events, such as player interactions or other NPC actions. This class is not a runtime component; its sole purpose is to translate a declarative JSON configuration block into a concrete, executable `Sensor` object that will be embedded within an NPC's behavior tree.

The core architectural pattern employed is the use of `Holder` objects (e.g., DoubleHolder, EnumHolder). This design decouples the raw configuration data from its final, resolved value. It allows for dynamic value resolution at build time via the `BuilderSupport` context, enabling features like variable substitution or context-aware defaults.

By inheriting from this class, concrete builders can define sensors for specific world events (e.g., `BuilderSensorDamage`, `BuilderSensorInteraction`) while reusing the common logic for defining range, target filtering, and memory slot assignment. The call to `provideFeature(Feature.LiveEntity)` signals to the NPC asset system that any NPC using a sensor derived from this builder requires access to the live entity system, ensuring necessary dependencies are available at runtime.

### Lifecycle & Ownership
- **Creation:** A concrete subclass of BuilderSensorEvent is instantiated by the server's asset loading system when it parses the `sensors` array within an NPC's JSON definition file. The system uses a registry to map a JSON `type` field to the corresponding builder class.
- **Scope:** Extremely short-lived. An instance exists only for the duration of parsing a single sensor definition block. It is a transient, stateful object.
- **Destruction:** The instance becomes eligible for garbage collection immediately after its `build` method is invoked and the resulting `Sensor` object is constructed and attached to the NPC's behavior profile. It does not persist into the main game loop.

## Internal State & Concurrency
- **State:** Highly mutable. The object is initialized with empty `Holder` fields. The `readConfig` method populates these fields based on the input JSON, making the instance stateful and specific to a single sensor configuration.
- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded use within the asset loading pipeline. Concurrent calls to `readConfig` or modification of its state from multiple threads will result in corrupted configuration and unpredictable server behavior.

## API Surface
The primary interaction is through the `readConfig` method, which populates the builder's internal state. The `get*` methods are subsequently used by the final `build` process to construct the `Sensor` object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(data) | Builder<Sensor> | O(N) | Parses the provided JsonElement, populates internal state, and validates inputs. N is the number of keys in the JSON object. |
| getRange(support) | double | O(1) | Resolves and returns the configured detection range using the provided build context. |
| getEventSearchType(support) | SensorEvent.EventSearchType | O(1) | Resolves and returns the entity search filter (e.g., PlayerOnly, NpcOnly). |
| getLockOnTargetSlot(support) | int | O(1) | Resolves the string-based target slot name into its integer ID via the BuilderSupport context. Returns Integer.MIN_VALUE if no slot is specified. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the asset loading framework. A concrete implementation would be used as follows by the framework:

```java
// Pseudo-code for the asset loading system
JsonElement sensorJson = parseNpcDefinitionFile(".../mob.json");
String sensorType = sensorJson.get("type").getAsString();

// The framework finds the correct concrete builder (e.g., BuilderSensorDamage)
BuilderSensorEvent builder = assetRegistry.getSensorBuilder(sensorType);

// The framework populates the builder and then builds the final Sensor
builder.readConfig(sensorJson);
Sensor runtimeSensor = builder.build(buildSupportContext);
npc.addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Never reuse a builder instance to configure more than one sensor. Each instance is stateful and must be discarded after a single `build` operation.
- **Runtime Invocation:** Do not attempt to instantiate or call methods on this class during the main game loop. It is strictly a pre-runtime, configuration-time component.
- **Manual Instantiation:** Avoid `new MyConcreteSensorBuilder()`. The asset system is responsible for creating builder instances to ensure proper registration and lifecycle management.

## Data Pipeline
BuilderSensorEvent is a key stage in the configuration data pipeline that transforms a static asset file into a live game object.

> Flow:
> NPC JSON File -> Gson Parser -> Asset Loader -> **BuilderSensorEvent.readConfig()** -> `build()` method -> Live `Sensor` Instance -> NPC Behavior Tree

