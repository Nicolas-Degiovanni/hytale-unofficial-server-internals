---
description: Architectural reference for BuilderSensorTimer
---

# BuilderSensorTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient Builder

## Definition
```java
// Signature
public class BuilderSensorTimer extends BuilderSensorBase {
```

## Architecture & Concepts

The BuilderSensorTimer is a key component within the server's NPC asset deserialization pipeline. It functions as a specialized factory, responsible for translating a JSON configuration block into a concrete, runtime instance of a SensorTimer.

In the Hytale NPC AI system, a Sensor is a conditional node that evaluates to true or false, guiding the decision-making process of an NPC's behavior tree. The SensorTimer specifically checks the status of a named Timer object associated with the NPC.

This builder's primary role is to define and enforce the data contract for a timer sensor configuration. It uses helper classes like StringHolder and NumberArrayHolder to encapsulate the configured values, which may be resolved later using a dynamic ExecutionContext. The builder pattern here decouples the complex process of JSON parsing, validation, and object construction from the final, immutable SensorTimer object.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderSensorTimer is created by the server's central asset loading system. When the system parses an NPC behavior file and encounters a JSON object with a type field corresponding to this sensor, it instantiates this builder to handle the configuration.
-   **Scope:** The lifecycle of a BuilderSensorTimer instance is extremely short and ephemeral. It exists only for the duration of parsing a single JSON object and building one SensorTimer instance.
-   **Destruction:** The builder is immediately eligible for garbage collection after the `build` method is called and the resulting SensorTimer is passed to the parent system. The asset loader does not retain references to builder objects.

## Internal State & Concurrency

-   **State:** The internal state is highly **mutable**. The `readConfig` method populates instance fields such as `name`, `timeRemainingRange`, and `timerState`. This state is transient and directly reflects the JSON data being processed.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The asset loading pipeline is designed to process configurations in a single-threaded manner or to create a new builder instance for each configuration. Concurrent calls to `readConfig` on a shared instance will result in a corrupted and unpredictable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | SensorTimer | O(1) | Constructs and returns the final SensorTimer runtime object. |
| readConfig(JsonElement) | Builder<Sensor> | O(N) | Parses the provided JSON, validates fields, and populates the builder's internal state. N is the number of keys in the JSON object. |
| getRemainingTimeRange(BuilderSupport) | double[] | O(1) | Retrieves the configured time range, resolving it against the execution context if necessary. |
| getTimer(BuilderSupport) | Timer | O(K) | Retrieves the named Timer instance from the provided support context. K is the complexity of the context's name lookup. |
| getTimerState() | Timer.TimerState | O(1) | Returns the configured timer state to check against. |

## Integration Patterns

### Standard Usage

Direct interaction with this class is an anti-pattern for game logic developers. Its usage is internal to the asset loading system. A developer defines the sensor in a JSON file, and the engine handles the builder's lifecycle automatically.

The conceptual internal flow resembles the following:

```java
// This code is a conceptual representation of the asset loader's work.
// Do not write code like this.

JsonElement sensorConfig = parseNpcBehaviorFile(".../my_npc.json");
BuilderSensorTimer builder = new BuilderSensorTimer();

// The engine populates the builder from JSON
builder.readConfig(sensorConfig);

// The engine provides a context and builds the final object
BuilderSupport support = engine.createBuilderSupportForNpc(npc);
SensorTimer runtimeSensor = builder.build(support);

// The sensor is then added to the NPC's behavior tree
npc.getBehaviorTree().addSensor(runtimeSensor);
```

### Anti-Patterns (Do NOT do this)

-   **Reusing Instances:** Never reuse a BuilderSensorTimer instance to build multiple SensorTimer objects. The internal state is not reset between builds and will lead to incorrect configurations.
-   **Manual Configuration:** Do not instantiate the builder and attempt to set its internal fields (e.g., `name`, `timerState`) directly. The class is designed to be configured exclusively through the `readConfig` method to ensure proper validation.
-   **Cross-Thread Sharing:** Do not access a builder instance from multiple threads. This will cause race conditions and data corruption.

## Data Pipeline

The flow of data from a static asset file to a runtime game object is a critical transformation pipeline where this builder operates.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> Gson Parser -> `JsonElement` -> **BuilderSensorTimer.readConfig()** -> Configured Builder State -> **BuilderSensorTimer.build()** -> `SensorTimer` Instance -> NPC Behavior Tree

