---
description: Architectural reference for BuilderSensorInAir
---

# BuilderSensorInAir

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSensorInAir extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorInAir class is a factory component within the server-side NPC asset pipeline. Its sole responsibility is to translate a declarative JSON configuration into a concrete, executable `SensorInAir` object. This class embodies the Builder pattern, providing a level of abstraction between the raw asset data and the live game-engine components.

It functions as a stateless translator. When the server's asset loader parses an NPC's behavior definition and encounters a sensor of this type, it instantiates this builder. The builder is then used to construct the final `SensorInAir` object, which is subsequently injected into the NPC's runtime behavior tree or state machine.

Notably, the `readConfig` method is a no-operation. This signifies that the `SensorInAir` component is a simple, parameter-less check; it requires no external configuration from the JSON asset beyond its type declaration. The resulting `SensorInAir` object will hold a reference to its originating builder, allowing the runtime to potentially access metadata like descriptions if needed.

### Lifecycle & Ownership
- **Creation:** Instantiated dynamically by a higher-level asset factory, such as `BuilderSupport`, during the deserialization of an NPC's JSON definition file. A new builder is created for each corresponding sensor entry in the asset.
- **Scope:** Extremely short-lived. The builder's lifecycle is confined to the asset construction phase. It exists only to produce a `SensorInAir` instance.
- **Destruction:** The builder object becomes eligible for garbage collection immediately after the `build` method is called and its product, the `SensorInAir` object, has been integrated into the parent NPC asset. It does not persist into the active game simulation.

## Internal State & Concurrency
- **State:** This class is stateless and effectively immutable. It contains no member fields and its behavior is fixed. The `readConfig` method does not alter its state.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the entire NPC asset building pipeline is designed to be a single-threaded process. Concurrent invocation of this builder is not a supported or expected scenario.

## API Surface
The public API is designed for consumption by the asset loading framework, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorInAir | O(1) | Constructs and returns a new `SensorInAir` instance. |
| readConfig(JsonElement) | Builder<Sensor> | O(1) | A no-op for this implementation. Returns a reference to itself. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable description for tooling or debug UIs. |
| getLongDescription() | String | O(1) | Returns a detailed, human-readable description for tooling or debug UIs. |

## Integration Patterns

### Standard Usage
A developer defining NPC behavior in JSON would trigger this builder's usage implicitly. The framework handles the instantiation and invocation. Direct interaction with this class is not part of the standard workflow.

*Conceptual framework-level invocation:*
```java
// This code is representative of the asset loading system's internal logic.
// Do not replicate this pattern in game logic.

// 1. A registry maps the JSON type "inAir" to this builder class.
Builder builder = builderRegistry.get("inAir"); // Returns a new BuilderSensorInAir

// 2. The framework passes the relevant JSON data.
builder.readConfig(sensorJsonData); // This is a no-op for this specific builder

// 3. The final sensor object is constructed.
Sensor sensor = builder.build(builderSupport);

// 4. The sensor is added to the NPC's behavior tree.
npcBehavior.addSensor(sensor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderSensorInAir()`. The asset pipeline manages the lifecycle of builders. Manually creating one bypasses the configuration and registration system, leading to uninitialized or disconnected components.
- **Retaining References:** Do not hold a reference to a builder instance after the asset loading process is complete. They are transient and should be allowed to be garbage collected.

## Data Pipeline
This builder acts as a transformation step in the data flow from static assets to live game objects.

> Flow:
> NPC Definition (JSON file) -> Asset Deserializer -> **BuilderSensorInAir** -> `SensorInAir` Object -> NPC Behavior Tree -> Game Loop Evaluation

