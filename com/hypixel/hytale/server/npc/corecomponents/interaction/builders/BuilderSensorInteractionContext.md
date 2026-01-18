---
description: Architectural reference for BuilderSensorInteractionContext
---

# BuilderSensorInteractionContext

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderSensorInteractionContext extends BuilderSensorBase {
```

## Architecture & Concepts
The BuilderSensorInteractionContext is a transient factory class within the NPC asset definition pipeline. Its sole responsibility is to deserialize a specific JSON configuration block and construct a runtime Sensor object, specifically a SensorInteractionContext.

In the Hytale NPC behavior system, a Sensor acts as a predicate or a conditional check. This particular builder creates a Sensor that evaluates to true if a player has previously interacted with the NPC in a specified manner, identified by a string *context*.

This class acts as a bridge between the static data representation of an NPC (JSON files) and the live, in-game behavior objects (Sensors). It is a key component for defining stateful, context-aware NPC interactions based on player history. The use of a StringHolder for the context suggests that the context name can be dynamically resolved at runtime, allowing for more flexible and reusable NPC behaviors.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by the server's asset loading system when it encounters a sensor of this type within an NPC's JSON definition file. It is not intended for manual instantiation by developers.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing its corresponding JSON block and building the final Sensor object.
- **Destruction:** The builder object is eligible for garbage collection immediately after the `build` method is called and the resulting SensorInteractionContext is returned to the asset loader. It holds no persistent state and is not retained.

## Internal State & Concurrency
- **State:** Highly mutable. The primary internal state is the `interactionContext` field, which is populated during the `readConfig` call. This state is transient and serves only to configure the Sensor object during its construction.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used in a single-threaded context during the asset deserialization phase. Concurrent calls to `readConfig` on the same instance will result in a corrupted or indeterminate state. This is an intentional design for a short-lived factory.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Sensor | O(1) | Constructs and returns a new SensorInteractionContext instance using the state configured via readConfig. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the input JSON, validates the presence and type of the "Context" field, and populates the internal state. Throws if required fields are missing. |
| getInteractionContext(BuilderSupport) | String | O(1) | Resolves and returns the configured interaction context string. Primarily used by the resulting Sensor object. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal NPC asset pipeline. A developer defines the sensor in a JSON file, and the system handles the builder's lifecycle automatically.

*NPC Definition File (example.json)*
```json
{
  "sensor": {
    "type": "SensorInteractionContext",
    "Context": "gave_quest_item"
  }
}
```

*Internal Asset Loader (conceptual)*
```java
// The system identifies the type and creates the correct builder
BuilderSensorInteractionContext builder = new BuilderSensorInteractionContext();

// The system passes the relevant JSON data to the builder
builder.readConfig(sensorJsonData);

// The builder creates the final runtime object
Sensor runtimeSensor = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Manual Configuration:** Do not manually instantiate this class with `new` and attempt to configure it. It is designed to be driven by the `readConfig` method from a JSON source.
- **State Reuse:** Do not retain an instance of this builder after calling `build`. It is not designed for reuse and its internal state should be considered consumed after construction. Create a new builder for each sensor you need to deserialize.

## Data Pipeline
The flow of data is strictly one-way, from static configuration to a runtime object.

> Flow:
> NPC JSON File -> Server Asset Parser -> **BuilderSensorInteractionContext**.readConfig() -> **BuilderSensorInteractionContext**.build() -> SensorInteractionContext (Live Object in Behavior Tree)

