---
description: Architectural reference for BuilderSensorNot
---

# BuilderSensorNot

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorNot extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorNot class is a configuration-time component within the server-side NPC AI asset pipeline. It embodies the Builder and Decorator patterns to construct a runtime sensor object that inverts the logic of a nested sensor.

Its primary architectural function is to facilitate compositional AI behaviors. Rather than defining monolithic, complex sensors, designers can define simple, reusable sensor components and then combine or modify them using logical builders like BuilderSensorNot. This is a declarative approach to AI design, where behavior is defined in data (JSON configuration files) rather than imperative code.

This class is exclusively a *configuration-time* object. It reads a JSON definition, validates it, and acts as a factory for the corresponding *runtime* object, SensorNot. Once the runtime object is built, the builder instance is discarded.

## Lifecycle & Ownership
- **Creation:** Instantiated by the central BuilderManager during the deserialization of an NPC's behavior asset file. A new instance is created for each "not" sensor definition encountered in the configuration.
- **Scope:** The object's lifecycle is exceptionally brief. It exists only during the asset loading, validation, and building phase for a single NPC configuration.
- **Destruction:** It becomes eligible for garbage collection immediately after its build method is invoked and the resulting SensorNot object is passed to the parent component. It holds no state that persists beyond this initial loading phase.

## Internal State & Concurrency
- **State:** The internal state is **mutable** during a single, well-defined window: the call to readConfig. During this method call, the object is populated from a JSON source. After this point, its state is treated as immutable for the remainder of its lifecycle. It holds configuration data, not dynamic runtime state.

- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The entire NPC asset loading pipeline is a single-threaded process. All method calls on a BuilderSensorNot instance, from creation to build, must be performed sequentially on the server's main asset loading thread.

    **WARNING:** Accessing or modifying a builder instance from multiple threads will lead to race conditions and a corrupted AI configuration.

## API Surface
The public API is designed for use by the internal NPC asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorNot | O(N) | Constructs the runtime SensorNot instance. Complexity is dependent on the nested sensor's build process. Returns null if the nested sensor fails to build. |
| readConfig(JsonElement) | BuilderSensorNot | O(1) | Deserializes the JSON configuration into the builder's internal fields. Throws exceptions on malformed data. |
| validate(...) | boolean | O(N) | Validates the builder's configuration and recursively validates the nested sensor. Complexity is dependent on the nested sensor's validation. |
| getSensor(BuilderSupport) | Sensor | O(N) | A helper method to build the nested sensor that this instance will invert. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, it is used declaratively within an NPC's JSON configuration file. The system's BuilderManager locates and uses this class automatically based on the type name.

A typical definition in an NPC behavior file would look like this:
```json
{
  "type": "not",
  "sensor": {
    "type": "hasLineOfSight",
    "target": "currentTarget"
  }
}
```
During server startup, the asset pipeline parses this JSON, instantiates a BuilderSensorNot, calls `readConfig` with this data, and finally calls `build` to produce the runtime sensor used by the NPC.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderSensorNot()`. The NPC asset system relies on the BuilderManager to manage the creation and registration of all builder types. Direct instantiation bypasses this system and will fail.
- **State Mutation After Deserialization:** Do not modify the internal state of the builder (e.g., its sensor or targetSlot fields) after the readConfig method has completed. The object's state is considered final post-deserialization.
- **Reusing Instances:** Builder instances are single-use and tied to a specific configuration block. Do not attempt to cache or reuse them.

## Data Pipeline
BuilderSensorNot functions as a step in the configuration-to-runtime data pipeline for NPC AI. It transforms a declarative data structure into an executable object.

> Flow:
> NPC Behavior JSON File -> JSON Parser -> BuilderManager -> **BuilderSensorNot.readConfig()** -> **BuilderSensorNot.validate()** -> **BuilderSensorNot.build()** -> SensorNot (Runtime Object) -> NPC Behavior Tree

