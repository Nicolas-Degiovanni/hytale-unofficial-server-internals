---
description: Architectural reference for BuilderSensorAlarm
---

# BuilderSensorAlarm

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient / Factory

## Definition
```java
// Signature
public class BuilderSensorAlarm extends BuilderSensorBase {
```

## Architecture & Concepts
The **BuilderSensorAlarm** class is a factory component within the server's Non-Player Character (NPC) asset pipeline. Its sole responsibility is to translate a specific JSON configuration block into a functional **SensorAlarm** object. It acts as a deserializer and constructor, bridging the gap between static data assets and live game-engine objects.

This builder is part of a larger system of "Sensors", which represent conditions an NPC's AI can evaluate (e.g., "is an enemy nearby?", "is my health low?"). This specific builder creates a sensor that checks the state of a named **Alarm**, a server-wide timer mechanism.

A key architectural pattern employed here is the use of **Holder** objects (e.g., **StringHolder**, **EnumHolder**). These wrappers encapsulate configuration values, deferring their final resolution until an **ExecutionContext** is available. This allows for dynamic value resolution and provides a consistent interface for validation and data retrieval.

## Lifecycle & Ownership
- **Creation:** An instance of **BuilderSensorAlarm** is created dynamically by the NPC asset loading system when it encounters a sensor definition of the corresponding type within an NPC's JSON file. It is never instantiated directly in game logic.
- **Scope:** The object's lifetime is extremely short and confined to the asset parsing process. It exists only to be configured via **readConfig** and then immediately used to produce a **SensorAlarm** instance via the **build** method.
- **Destruction:** Once the **build** method has been called and the resulting **SensorAlarm** object is integrated into the NPC's behavior definition, the builder instance is no longer referenced and becomes eligible for garbage collection. It does not persist in any form.

## Internal State & Concurrency
- **State:** The internal state is mutable. The **readConfig** method populates the internal **Holder** fields based on the input JSON. This state represents the complete configuration for the final **SensorAlarm** object.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated, configured, and used within the single-threaded context of the asset loading pipeline. Concurrent calls to **readConfig** or **build** will result in a corrupted and unpredictable state.

## API Surface
The public API is primarily for internal use by the asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorAlarm | O(1) | Constructs and returns a new **SensorAlarm** instance using the current configuration. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Populates the builder's internal state from a JSON data element. N is the number of properties. |
| getAlarm(BuilderSupport) | Alarm | O(1) | Resolves and retrieves the target **Alarm** instance from the provided context. |
| getState(BuilderSupport) | SensorAlarm.State | O(1) | Resolves and retrieves the configured alarm state to check for. |
| isClear(BuilderSupport) | boolean | O(1) | Resolves and retrieves the flag indicating if the alarm should be cleared on success. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class using Java code. Instead, its functionality is invoked by defining a corresponding sensor in an NPC's JSON asset file. The server's asset loader handles the instantiation and configuration behind the scenes.

```json
// Example NPC Behavior JSON
{
  "trigger": {
    "type": "Sensor",
    "sensor": {
      "type": "Alarm",
      "Name": "patrol_timer",
      "State": "Passed",
      "Clear": true
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new BuilderSensorAlarm()**. The object is useless without being managed and configured by the asset pipeline, which provides the necessary **BuilderSupport** context.
- **State Reuse:** Do not attempt to reuse a builder instance to configure multiple sensors. Each JSON block must correspond to a new, distinct builder instance to prevent state leakage.
- **Concurrent Access:** Accessing a builder instance from multiple threads is strictly forbidden and will lead to unpredictable behavior and data corruption.

## Data Pipeline
The builder is a critical step in the data transformation pipeline from static configuration to a runtime AI component.

> Flow:
> NPC JSON Asset -> Asset Loader -> **BuilderSensorAlarm** -> **SensorAlarm** Instance -> NPC Behavior Tree

