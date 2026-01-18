---
description: Architectural reference for BuilderSensorMany
---

# BuilderSensorMany

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient Configuration Builder

## Definition
```java
// Signature
public abstract class BuilderSensorMany extends BuilderSensorBase {
```

## Architecture & Concepts
BuilderSensorMany is an abstract base class that serves as a composite builder within the NPC AI asset pipeline. Its primary function is not to define a singular sensor's logic, but to act as a container and orchestrator for a list of child Sensor configurations defined in a JSON asset.

This class embodies the **Composite design pattern**. It allows the asset loading system to parse a JSON array of disparate sensor definitions and treat them as a single, manageable unit. During server initialization or asset hot-reloading, this builder is responsible for deserializing, validating, and preparing a collection of runtime Sensor objects from a configuration file.

A key architectural feature is the `AutoUnlockTargetSlot` property. This provides a declarative mechanism for linking the collective state of the managed sensors to the NPC's targeting memory. When the conditions of the composite sensor are no longer met, the system can automatically clear a specified target slot, simplifying state management logic within the AI's behavior tree.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the engine's central `BuilderManager` during the deserialization of an NPC's JSON configuration file. The specific concrete subclass to be created is determined by a type identifier within the JSON data.

-   **Scope:** The lifecycle of a BuilderSensorMany instance is ephemeral and strictly confined to the asset loading and validation phase. It exists only to hold the intermediate, parsed representation of the configuration data.

-   **Destruction:** After the `validate` method is successfully called and the final, immutable runtime Sensor objects are constructed, the builder instance has fulfilled its purpose. It holds no further references within the active game state and becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** Highly mutable. The object is created in a default state and is populated by the `readConfig` method. Its internal fields, particularly the `objectListHelper` which contains the list of child builders, are modified during this process.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access during the server's asset loading sequence. Concurrent invocation of `readConfig` or `validate` from multiple threads will result in a race condition and lead to corrupted state or unpredictable validation errors. Any multi-threaded asset processing must ensure that individual builder instances are confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Sensor> | O(N) | Deserializes the JSON configuration. Populates the internal list of child sensors. N is the number of sensors in the JSON array. |
| validate(...) | boolean | O(N) | Recursively invokes the validation logic for itself and all child sensor builders. N is the number of child sensors. |
| getAutoUnlockedTargetSlot(BuilderSupport support) | int | O(1) | Resolves the configured target slot name into its runtime integer ID. Requires a valid support context. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java code. Instead, they define its behavior declaratively in an NPC's JSON configuration file. The engine handles the lifecycle automatically.

A concrete implementation, such as `SensorAllOf`, would be used in JSON like this:
```json
// Example NPC configuration snippet
{
  "type": "SensorAllOf", // This identifier maps to a concrete subclass of BuilderSensorMany
  "sensors": [
    {
      "type": "SensorHealth",
      "below": 0.5
    },
    {
      "type": "SensorTargetDistance",
      "max": 10
    }
  ],
  "autoUnlockTargetSlot": "primary_threat"
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ConcreteBuilderSensorMany()`. The asset pipeline relies on the `BuilderManager` to control instantiation and dependency injection. Direct creation will result in an uninitialized and non-functional object.

-   **State Mutation After Deserialization:** Do not attempt to modify the state of the builder after the `readConfig` method has completed. The validation and object construction phases assume the builder's state is a faithful representation of the source asset file.

-   **Ignoring Validation Result:** The boolean return value from the `validate` method is critical. If it returns false, the configuration is invalid and must not be used to construct runtime objects, as this will lead to server instability or severe AI logic errors.

## Data Pipeline
The BuilderSensorMany class is a key stage in the transformation of configuration data into executable game logic.

> Flow:
> NPC JSON Asset File -> GSON Deserializer -> `BuilderManager` Factory -> **BuilderSensorMany.readConfig()** -> **BuilderSensorMany.validate()** -> Finalized Runtime `Sensor` Object Graph -> NPC AI Component

