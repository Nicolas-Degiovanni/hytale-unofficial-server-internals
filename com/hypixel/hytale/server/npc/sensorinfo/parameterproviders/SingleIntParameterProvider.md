---
description: Architectural reference for SingleIntParameterProvider
---

# SingleIntParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Transient Data Model

## Definition
```java
// Signature
public class SingleIntParameterProvider extends SingleParameterProvider implements IntParameterProvider {
```

## Architecture & Concepts
The SingleIntParameterProvider is a specialized, concrete implementation within the server-side NPC AI framework. Its primary function is to serve as a mutable, type-safe container for a single integer value that influences AI decision-making.

This class embodies the **Strategy Pattern** for data provision. It decouples the AI logic (the consumer) from the source of the data (the producer). An AI behavior, such as one for targeting, can depend on the IntParameterProvider contract to retrieve a value, without needing to know if that value originates from a sensor, a configuration file, or another game system.

By extending SingleParameterProvider, it inherits the mechanism for being identified by a unique integer ID, allowing a collection of disparate parameter providers to be managed and queried generically by the core AI system. This class is the final link in that chain, responsible for the actual storage and retrieval of the integer data.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by higher-level AI components, typically during the initialization of an NPC's `Sensor` or `Behavior` instance. It is not a global or shared service.
-   **Scope:** The lifecycle of a SingleIntParameterProvider is tightly coupled to its owning AI component. It persists only as long as the sensor or behavior that requires it remains active for a specific NPC.
-   **Destruction:** The object is eligible for garbage collection as soon as its owning AI component is destroyed or reconfigured. The `clear` method provides a mechanism for resetting its state, allowing the object to be reused within its scope to reduce object churn.

## Internal State & Concurrency
-   **State:** Highly mutable. The class contains a single integer field, `value`, which represents its core state. The `clear` method resets this state to `Integer.MIN_VALUE`, which serves as a sentinel value indicating an uninitialized or invalid state.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within the single-threaded context of the server's main game loop, specifically during the NPC update tick. Any concurrent access from other threads without external synchronization will result in race conditions and undefined AI behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIntParameter() | int | O(1) | Retrieves the currently stored integer value. |
| clear() | void | O(1) | Resets the internal state to its default sentinel value (`Integer.MIN_VALUE`). |
| overrideInt(int value) | void | O(1) | Overwrites the internal state with the provided integer value. |

## Integration Patterns

### Standard Usage
This provider is typically instantiated and registered with an AI context. Other systems, like sensors, then update its value, which is later consumed by AI behaviors.

```java
// In an AI component's initialization phase
SingleIntParameterProvider targetIdProvider = new SingleIntParameterProvider(ParameterRegistry.TARGET_ENTITY_ID);
npc.getAIContext().registerProvider(targetIdProvider);

// During a sensor's update tick
int nearestEnemyId = findNearestEnemy();
targetIdProvider.overrideInt(nearestEnemyId);

// A behavior can then consume this value
int currentTarget = aIContext.getProvider(ParameterRegistry.TARGET_ENTITY_ID).getIntParameter();
```

### Anti-Patterns (Do NOT do this)
-   **State Sharing:** Do not share a single instance of SingleIntParameterProvider across multiple, independent NPCs. This will cause catastrophic state corruption, where one NPC's sensor data incorrectly drives another's behavior. Each NPC requires its own isolated set of providers.
-   **Ignoring Sentinel Value:** Logic that consumes this provider must be implemented to handle the `Integer.MIN_VALUE` case. Treating this sentinel value as a valid entity ID or data point will lead to severe bugs and server exceptions.
-   **Cross-Thread Modification:** Never call `overrideInt` from a separate thread (e.g., a networking or database thread) without synchronizing back to the main server thread.

## Data Pipeline
This component acts as a simple stateful node in the NPC's internal data flow, bridging the gap between sensory input and behavioral output.

> Flow:
> AI Sensor Logic -> **overrideInt(newValue)** -> SingleIntParameterProvider (State) -> **getIntParameter()** -> AI Behavior Logic

