---
description: Architectural reference for SensorTime
---

# SensorTime

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class SensorTime extends SensorBase {
```

## Architecture & Concepts
The SensorTime class is a specialized component within the server-side Non-Player Character (NPC) AI framework. It functions as a world-state predicate, designed to answer a single, specific question: "Does the current in-game time fall within a predefined period?".

Architecturally, it embodies the **Sensor** pattern. It decouples high-level AI decision-making logic (e.g., Behavior Trees, Goal-Oriented Planners) from the low-level details of the world's time-keeping system. An NPC's behavioral definition, or Role, aggregates multiple sensors to build a comprehensive understanding of its environment. SensorTime is the primary mechanism for creating time-sensitive behaviors, such as nocturnal creature activity or villager daily schedules.

Its configuration is derived entirely from asset definitions via a corresponding builder, BuilderSensorTime. This ensures that AI behavior is data-driven and can be modified without changing core engine code.

## Lifecycle & Ownership
-   **Creation:** SensorTime instances are created exclusively by the server's NPC asset loading pipeline. During server startup or asset hot-reloading, a BuilderSensorTime definition from an NPC asset file is processed, resulting in the instantiation of a SensorTime object. **Warning:** Direct instantiation by gameplay code is a critical design violation.

-   **Scope:** The object's lifetime is bound to the NPC Role or AI definition that contains it. It is a stateless, configuration-holding object that persists as long as its parent AI definition is loaded in memory.

-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection when the parent NPC Role is unloaded, typically when a zone unloads or the server shuts down. There is no manual destruction or cleanup method.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (minTime, maxTime, checkDay, etc.) are final and are set only once within the constructor. The object's purpose is to hold static, pre-configured time parameters. It does not cache or modify any state during its lifetime.

-   **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature, a SensorTime instance can be safely accessed and used by multiple threads simultaneously without any need for external locking or synchronization. The core matches method is a pure function with respect to the object's internal state.

## API Surface
The public contract is minimal, focused entirely on its predicate function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the current world time against the sensor's configured range. Retrieves WorldTimeResource from the provided Store. Returns true if conditions are met. |
| getSensorInfo() | InfoProvider | O(1) | Returns metadata for debugging tools. In this implementation, it returns null, indicating no extended debug information is available. |

## Integration Patterns

### Standard Usage
SensorTime is not used directly. Instead, it is invoked polymorphically by the NPC's core AI loop, which iterates through all sensors attached to the current Role.

```java
// Example from within a hypothetical NPC AI update system

// The 'currentRole' object holds a collection of configured sensors
for (SensorBase sensor : currentRole.getAllSensors()) {
    // The AI system does not need to know the concrete type is SensorTime
    boolean isConditionMet = sensor.matches(npcEntityRef, currentRole, deltaTime, worldStore);

    if (isConditionMet) {
        // The sensor's condition is true, potentially activating a new behavior
        // or contributing to a decision-making score.
        aiBlackboard.setCondition(sensor.getConditionKey(), true);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Under no circumstances should this class be instantiated using the new keyword in gameplay logic. All configuration must originate from NPC asset files to maintain the data-driven design.
    ```java
    // ANTI-PATTERN: Do not do this.
    SensorTime manualSensor = new SensorTime(someBuilder, someSupport);
    ```
-   **Stateful Misuse:** Do not attempt to modify the SensorTime object after creation via reflection or other means. It is designed and expected to be immutable.

## Data Pipeline
The flow of data for SensorTime begins with asset definition and ends with a boolean result during the game loop.

> Flow:
> NPC Asset File (JSON/HOCON) -> Asset Loader -> **BuilderSensorTime** -> **SensorTime Instance** (Stored in NPC Role)
>
> ---
>
> Game Tick -> NPC AI Update -> `matches()` call on SensorTime -> Access **WorldTimeResource** -> **Boolean Result** -> AI Behavior Tree

---

