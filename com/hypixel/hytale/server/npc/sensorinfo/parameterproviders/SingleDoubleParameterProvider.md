---
description: Architectural reference for SingleDoubleParameterProvider
---

# SingleDoubleParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Transient Data Model

## Definition
```java
// Signature
public class SingleDoubleParameterProvider extends SingleParameterProvider implements DoubleParameterProvider {
```

## Architecture & Concepts
The SingleDoubleParameterProvider is a specialized, high-performance data container within the server-side NPC Artificial Intelligence framework. Its sole responsibility is to hold a single, floating-point value representing the output of an NPC *Sensor*.

This class acts as a standardized data contract between systems that *produce* world-state information (Sensors) and systems that *consume* it to make decisions (Behaviors, Goal Arbitrators). By implementing the DoubleParameterProvider interface, it allows consumer systems to query sensor data in a type-safe, abstract manner. For example, a behavior that needs to know the distance to a target can request the DoubleParameterProvider for the TARGET_DISTANCE parameter, without needing to know the implementation details of the sensor that calculated it.

Its existence is a direct consequence of a performance-oriented design that favors object reuse and primitive data types to minimize garbage collection pressure and overhead within the main server AI tick loop.

## Lifecycle & Ownership
-   **Creation:** These objects are not intended for direct instantiation by developers. They are created and managed by a higher-level factory or manager, such as the SensorInfo component of an NPC. This system pre-allocates or dynamically creates providers as an NPC's sensor suite is configured.

-   **Scope:** The lifetime of the *data* within a SingleDoubleParameterProvider is extremely short, typically lasting for only a single AI processing tick. The object *instance* itself, however, is long-lived and is aggressively reused.

-   **Destruction:** Instances are not typically destroyed via garbage collection. Instead, the **clear** method is invoked at the beginning of each AI tick to reset its internal state. This allows the same object instance to be reused indefinitely for the lifetime of the parent NPC, which is a critical optimization pattern to avoid memory churn.

## Internal State & Concurrency
-   **State:** The internal state is mutable, consisting of a single double field. This value is volatile and is expected to be overwritten on every AI tick by its corresponding Sensor. The default, cleared state is -Double.MAX_VALUE, which serves as a sentinel value indicating that the sensor has not yet provided data for the current tick.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively owned and operated on by the single thread responsible for ticking a specific NPC's AI. Any concurrent read or write operations from other threads will result in race conditions, leading to corrupted state and unpredictable AI behavior. All access must be externally synchronized by the server's entity update scheduler.

## API Surface
The public contract is minimal, focusing entirely on state management for a single AI tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDoubleParameter() | double | O(1) | Retrieves the cached sensor value from the last evaluation. |
| clear() | void | O(1) | Resets the internal state to its default sentinel value. This is a critical part of the object reuse pattern. |
| overrideDouble(double value) | void | O(1) | Overwrites the internal state with a new value. This is the primary entry point, called by a Sensor. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Instead, a Sensor implementation will receive a provider instance from the NPC's state manager and use it to publish its calculated result.

```java
// Example from within a hypothetical "DistanceToTargetSensor"

// The provider is retrieved from the NPC's central state cache
DoubleParameterProvider provider = npc.getSensorInfo().getParameter(Parameter.TARGET_DISTANCE);

// The system guarantees the correct provider type is returned
if (provider instanceof SingleDoubleParameterProvider) {
   double calculatedDistance = findDistanceToCurrentTarget();
   
   // The sensor's only job is to update the provider's state
   ((SingleDoubleParameterProvider) provider).overrideDouble(calculatedDistance);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SingleDoubleParameterProvider()`. The AI framework manages the lifecycle and pooling of these objects. Bypassing the framework will break state management and lead to memory leaks.
-   **Cross-Tick Caching:** Do not store a reference to this object or its value across multiple AI ticks. The value is considered invalid as soon as the tick ends and will be reset or overwritten by the next one. Query the provider for its value each time it is needed.
-   **External Modification:** Only the designated Sensor responsible for a parameter should call `overrideDouble`. Other systems, such as Behaviors, should only read from the provider via `getDoubleParameter`.

## Data Pipeline
This class serves as a temporary storage node in the NPC AI data flow, decoupling data production from data consumption.

> Flow:
> World State Query -> NPC Sensor Logic -> **SingleDoubleParameterProvider.overrideDouble()** -> AI Behavior Tree reads with **getDoubleParameter()** -> NPC Action Execution

