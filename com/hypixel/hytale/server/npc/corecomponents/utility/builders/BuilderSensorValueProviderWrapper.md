---
description: Architectural reference for BuilderSensorValueProviderWrapper
---

# BuilderSensorValueProviderWrapper

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorValueProviderWrapper extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorValueProviderWrapper is a configuration-time factory component within the server's NPC asset loading pipeline. Its primary responsibility is to parse a JSON configuration block and construct a runtime instance of a SensorValueProviderWrapper.

This class embodies the **Decorator** pattern at the configuration level. It does not create a new sensor from scratch; instead, it "wraps" an existing, predefined Sensor definition. Its key architectural function is to create a dynamic bridge between an NPC's internal state (the "value store") and a sensor's parameters.

This allows a single, generic sensor implementation (e.g., a proximity check) to be reused in multiple contexts with different parameters that can change at runtime. For example, a sensor's detection radius could be dynamically supplied from an NPC's "aggression level" value, rather than being a static, hard-coded number. This component is the configuration mechanism that enables such dynamic behavior.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderSensorValueProviderWrapper is created by the central BuilderManager during the deserialization of an NPC's behavior asset file. It is never instantiated directly in game logic code.
-   **Scope:** The object's lifetime is extremely short and confined to the asset loading and validation phase. It exists only to process its corresponding JSON block.
-   **Destruction:** Once the `build` method is called and the final runtime SensorValueProviderWrapper object is returned, the builder instance is no longer referenced by the asset loader and becomes eligible for garbage collection. It does not persist into the active game state.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable during its lifecycle. The `readConfig` method populates the internal fields (`sensor`, `passValues`, `parameterMappings`) from the source JSON. This state represents an intermediate, unresolved configuration.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The entire NPC asset building pipeline is designed as a single-threaded, sequential process. Concurrent calls to `readConfig` or `build` would lead to unpredictable behavior and state corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorValueProviderWrapper | O(N) | Constructs the final, runtime SensorValueProviderWrapper instance. Resolves all internal object references. N is the number of parameter mappings. Returns null if the underlying sensor cannot be built. |
| readConfig(JsonElement) | BuilderSensorValueProviderWrapper | O(K) | Deserializes a JSON object into the builder's internal state. K is the number of properties in the JSON object. This is the primary entry point for populating the builder. |
| validate(...) | boolean | O(N) | Performs load-time validation to ensure configuration integrity. Critically, it checks for duplicate parameter slot names to prevent ambiguous overrides. N is the number of parameter mappings. |

## Integration Patterns

### Standard Usage
This class is used declaratively within NPC JSON asset files and processed by the engine's asset loading system. A developer would not interact with the Java class directly, but rather with the JSON structure it parses. The engine's internal `BuilderManager` orchestrates its lifecycle.

```java
// Conceptual engine code during asset loading
BuilderSensorValueProviderWrapper builder = new BuilderSensorValueProviderWrapper();

// 1. Engine deserializes a JSON block and passes it to the builder
builder.readConfig(npcBehaviorJson.get("myWrappedSensor"));

// 2. Engine validates the configuration
boolean isValid = builder.validate(configName, validationHelper, context, scope, errors);
if (!isValid) {
    // Halt loading and report errors
}

// 3. Engine builds the final runtime object
SensorValueProviderWrapper runtimeSensor = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BuilderSensorValueProviderWrapper()` in game logic. The class is designed to be managed exclusively by the asset loading framework.
-   **State Re-use:** Do not attempt to reuse a builder instance to parse multiple JSON configurations. Each instance is single-use and its state is specific to one configuration block.
-   **Calling Build Before Read:** Invoking `build` before `readConfig` has been successfully called will result in an improperly configured object and will likely throw a NullPointerException.

## Data Pipeline
The class acts as a transformation step in the pipeline that converts static configuration files into live game objects.

> Flow:
> NPC Behavior JSON File -> Engine JSON Parser -> **BuilderSensorValueProviderWrapper** instance -> `build()` call -> Runtime `SensorValueProviderWrapper` Object -> Attached to NPC Behavior Tree

