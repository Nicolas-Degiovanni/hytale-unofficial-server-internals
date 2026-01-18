---
description: Architectural reference for BuilderSensorBase
---

# BuilderSensorBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Transient Factory Component

## Definition
```java
// Signature
public abstract class BuilderSensorBase extends BuilderBase<Sensor> {
```

## Architecture & Concepts
BuilderSensorBase is an abstract foundational class within the server's NPC asset loading pipeline. It is not used directly but serves as the parent for all concrete sensor builder implementations, such as a proximity sensor or a health-check sensor.

Its primary architectural role is to standardize the deserialization and validation of common properties shared by all NPC *Sensors*. Sensors are the perceptual components of an NPC's AI, acting as triggers for behaviors. This class provides the logic for two fundamental sensor configurations:
1.  **Once:** A boolean flag determining if the sensor should deactivate permanently after its first successful trigger.
2.  **Enabled:** A dynamic boolean state, often driven by an expression, that controls whether the sensor is active.

This class acts as a translation layer, converting declarative JSON configuration from an asset file into the state of a builder object, which will ultimately produce a runtime `Sensor` instance for an NPC's behavior tree.

## Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass (e.g., `BuilderSensorProximity`) is instantiated by the NPC asset loading system when it encounters a corresponding sensor definition within an NPC's JSON configuration file. This process is typically managed by a central asset factory or registry.
-   **Scope:** The object's lifetime is exceptionally brief and confined to the asset loading phase. It is a transient, single-use object created to process one specific sensor definition from one NPC file.
-   **Destruction:** The builder is eligible for garbage collection as soon as the `build` method is called and the final `Sensor` object is constructed and integrated into the NPC's runtime component set. It holds no state that persists beyond this process.

## Internal State & Concurrency
-   **State:** The state of BuilderSensorBase is mutable. Its fields, `once` and `enabled`, are populated directly by the `readCommonConfig` method during JSON parsing. This state is local and specific to the single sensor being constructed.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to operate exclusively on the thread responsible for asset loading. The server's asset pipeline is assumed to be a single-threaded process to ensure deterministic and safe object construction. Concurrent access to a builder instance would result in a corrupted or unpredictable `Sensor` object.

## API Surface
The public contract is designed for consumption by the asset loading framework, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement data) | Builder<Sensor> | O(k) | Deserializes common properties ("Once", "Enabled") from the provided JSON. `k` is the number of keys. |
| isEnabled(ExecutionContext context) | boolean | O(E) | Evaluates if the sensor is currently enabled. Complexity depends on the expression `E` in the BooleanHolder. |
| validate(...) | boolean | O(1) | Performs load-time validation and updates the parent validation context with its `once` status. |
| category() | Class<Sensor> | O(1) | Returns the `Sensor.class` token, identifying the type of object this builder family produces. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they create new sensor types by extending it. The framework then discovers and uses these concrete implementations.

```java
// A concrete builder extending the base functionality
public class BuilderSensorOnFire extends BuilderSensorBase {

    // Custom properties for this specific sensor
    private float minFireTicks;

    @Override
    public Builder<Sensor> readCommonConfig(@Nonnull JsonElement data) {
        super.readCommonConfig(data); // CRITICAL: Must call super
        // ... read custom properties like "minFireTicks" from JSON
        return this;
    }

    @Override
    public Sensor build(ExecutionContext context) {
        // Use the deserialized state to create the final runtime object
        return new OnFireSensor(this.once, this.enabled, this.minFireTicks);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The class is abstract and cannot be instantiated. Concrete builders must only be created by the server's asset loading system to ensure proper initialization and context.
-   **State Reuse:** Never attempt to reuse a builder instance to parse a second JSON object. Each builder is designed to be single-use and its internal state is not reset between operations.
-   **Omitting Super Call:** When overriding `readCommonConfig` or `validate`, failing to call `super.method()` will prevent common properties from being processed, resulting in a misconfigured and non-functional sensor.

## Data Pipeline
This class is a key component in the data transformation pipeline that turns static configuration into live AI components.

> Flow:
> NPC_Asset.json -> Server Asset Loader -> **BuilderSensorBase** (via concrete subclass) -> `build()` method -> Live `Sensor` Object -> NPC Behavior Tree

